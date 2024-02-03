# `PowerInfer\ggml-quants.c`

```cpp
// 包含自定义的头文件 ggml-quants.h 和 ggml-impl.h
#include "ggml-quants.h"
#include "ggml-impl.h"

// 包含数学函数库、字符串处理库、断言库、浮点数处理库的头文件
#include <math.h>
#include <string.h>
#include <assert.h>
#include <float.h>

// 如果是 ARM 平台，包含 ARM NEON 指令集的头文件
#ifdef __ARM_NEON
#include <arm_neon.h>
#else
// 如果不是 ARM 平台，根据不同的平台包含对应的 SIMD 指令集的头文件
#ifdef __wasm_simd128__
#include <wasm_simd128.h>
#else
#ifdef __POWER9_VECTOR__
#include <altivec.h>
#undef bool
#define bool _Bool
#else
#if defined(_MSC_VER) || defined(__MINGW32__)
#include <intrin.h>
#else
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__) || defined(__SSE3__)
#if !defined(__riscv)
#include <immintrin.h>
#endif
#endif
#endif
#endif
#endif
#endif

// 如果是 RISC-V 平台，包含 RISC-V 向量指令集的头文件
#ifdef __riscv_v_intrinsic
#include <riscv_vector.h>
#endif

// 重新定义 MIN 和 MAX 宏，确保不会与其他库中的定义冲突
#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义一个宏，用于将两个 __m128i 类型的变量合并成一个 __m256i 类型的变量
#define MM256_SET_M128I(a, b) _mm256_insertf128_si256(_mm256_castsi128_si256(b), (a), 1)

// 如果支持 AVX、AVX2、AVX512F 或 SSSE3 指令集，定义一个内联函数，用于对 int8_t 类型的变量进行乘法和求和
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)
static inline __m128i mul_sum_i8_pairs(const __m128i x, const __m128i y) {
    const __m128i ax = _mm_sign_epi8(x, x);
    const __m128i sy = _mm_sign_epi8(y, x);
    const __m128i dot = _mm_maddubs_epi16(ax, sy);
    const __m128i ones = _mm_set1_epi16(1);
    return _mm_madd_epi16(ones, dot);
}

// 如果支持 AVX、AVX2 或 AVX512F 指令集，定义一个内联函数，用于对 8 个 float 类型的变量进行水平求和
#if __AVX__ || __AVX2__ || __AVX512F__
static inline float hsum_float_8(const __m256 x) {
    __m128 res = _mm256_extractf128_ps(x, 1);
    res = _mm_add_ps(res, _mm256_castps256_ps128(x));
    res = _mm_add_ps(res, _mm_movehl_ps(res, res));
    res = _mm_add_ss(res, _mm_movehdup_ps(res));
    return _mm_cvtss_f32(res);
}
// 水平相加 8 个 int32_t，返回结果
static inline int hsum_i32_8(const __m256i a) {
    // 将 a 转换为两个 128 位的寄存器，然后相加
    const __m128i sum128 = _mm_add_epi32(_mm256_castsi256_si128(a), _mm256_extractf128_si256(a, 1));
    // 将 sum128 的高 64 位和低 64 位相加
    const __m128i hi64 = _mm_unpackhi_epi64(sum128, sum128);
    // 将 hi64 和 sum128 相加
    const __m128i sum64 = _mm_add_epi32(hi64, sum128);
    // 将 sum64 的高 32 位和低 32 位交换位置后相加
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    // 将 sum64 和 hi32 相加后返回结果
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

// 水平相加 4 个 int32_t，返回结果
static inline int hsum_i32_4(const __m128i a) {
    // 将 a 的高 64 位和低 64 位相加
    const __m128i hi64 = _mm_unpackhi_epi64(a, a);
    // 将 hi64 和 a 相加
    const __m128i sum64 = _mm_add_epi32(hi64, a);
    // 将 sum64 的高 32 位和低 32 位交换位置后相加
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    // 将 sum64 和 hi32 相加后返回结果
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

#if defined(__AVX2__) || defined(__AVX512F__)
// 将 32 位数据扩展为 32 字节数据 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于扩展的掩码
    const __m256i shuf_mask = _mm256_set_epi64x(
            0x0303030303030303, 0x0202020202020202,
            0x0101010101010101, 0x0000000000000000);
    // 使用掩码将 x32 扩展为 32 字节数据
    __m256i bytes = _mm256_shuffle_epi8(_mm256_set1_epi32(x32), shuf_mask);
    // 创建用于设置特定位的掩码
    const __m256i bit_mask = _mm256_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 将特定位设置为 1
    bytes = _mm256_or_si256(bytes, bit_mask);
    // 返回结果
    return _mm256_cmpeq_epi8(bytes, _mm256_set1_epi64x(-1));
}

// 将 32 个 4 位数据解包为 32 字节数据
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 加载 16 字节数据到 tmp 寄存器
    const __m128i tmp = _mm_loadu_si128((const __m128i *)rsi);
    // 将 tmp 中的数据右移 4 位，与原始数据组合成 32 字节数据
    const __m256i bytes = MM256_SET_M128I(_mm_srli_epi16(tmp, 4), tmp);
    // 创建用于提取低 4 位的掩码
    const __m256i lowMask = _mm256_set1_epi8( 0xF );
    // 提取低 4 位数据
    return _mm256_and_si256(lowMask, bytes);
}

// 将 int16_t 两两相加，并以浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m256i x) {
    // 创建全为 1 的 16 位整数向量
    const __m256i ones = _mm256_set1_epi16(1);
    // 将 x 中的 int16_t 两两相加
    const __m256i summed_pairs = _mm256_madd_epi16(ones, x);
    # 将__m256i类型的寄存器中的整数值转换为__m256类型的寄存器中的单精度浮点数值
    return _mm256_cvtepi32_ps(summed_pairs);
// 将 32 位数据扩展为 32 个字节，其中每个字节的值为 0x00 或 0xFF
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    # 将输入的字节流 x 转换为 32 位整数 x32
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    # 创建用于字节重排的掩码常量 shuf_maskl 和 shuf_maskh
    const __m128i shuf_maskl = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
    const __m128i shuf_maskh = _mm_set_epi64x(0x0303030303030303, 0x0202020202020202);
    # 使用 shuf_maskl 和 shuf_maskh 对 x32 进行字节重排，得到 bytesl 和 bytesh
    __m128i bytesl = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskl);
    __m128i bytesh = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskh);
    # 创建用于位操作的掩码常量 bit_mask
    const __m128i bit_mask = _mm_set1_epi64x(0x7fbfdfeff7fbfdfe);
    # 对 bytesl 和 bytesh 进行位或操作，将特定位设置为 1
    bytesl = _mm_or_si128(bytesl, bit_mask);
    bytesh = _mm_or_si128(bytesh, bit_mask);
    # 对 bytesl 和 bytesh 进行比较操作，将结果存储在 bytesl 和 bytesh 中
    bytesl = _mm_cmpeq_epi8(bytesl, _mm_set1_epi64x(-1));
    bytesh = _mm_cmpeq_epi8(bytesh, _mm_set1_epi64x(-1));
    # 将 bytesh 和 bytesl 合并为一个 256 位的结果并返回
    return MM256_SET_M128I(bytesh, bytesl);
// 将 32 个 4 位字段解压缩为 32 个字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从内存中加载 16 个字节
    __m128i tmpl = _mm_loadu_si128((const __m128i *)rsi);
    // 将 tmpl 中的每个字节向右移动 4 位
    __m128i tmph = _mm_srli_epi16(tmpl, 4);
    // 创建一个低位掩码
    const __m128i lowMask = _mm_set1_epi8(0xF);
    // 使用低位掩码和 tmpl 进行按位与操作
    tmpl = _mm_and_si128(lowMask, tmpl);
    // 使用低位掩码和 tmph 进行按位与操作
    tmph = _mm_and_si128(lowMask, tmph);
    // 返回由 tmph 和 tmpl 组成的 __m256i 类型的向量
    return MM256_SET_M128I(tmph, tmpl);
}

// 将 int16_t 两两相加，并作为浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m128i xh, const __m128i xl) {
    // 创建一个全为 1 的 __m128i 类型的向量
    const __m128i ones = _mm_set1_epi16(1);
    // 将 xl 中的每对值相加，并将结果存储在 summed_pairsl 中
    const __m128i summed_pairsl = _mm_madd_epi16(ones, xl);
    // 将 xh 中的每对值相加，并将结果存储在 summed_pairsh 中
    const __m128i summed_pairsh = _mm_madd_epi16(ones, xh);
    // 将 summed_pairsh 和 summed_pairsl 组成的 __m256i 类型的向量转换为浮点向量并返回
    const __m256i summed_pairs = MM256_SET_M128I(summed_pairsh, summed_pairsl);
    return _mm256_cvtepi32_ps(summed_pairs);
}

static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
    // 将 ax 转换为两个 __m128i 类型的向量
    const __m128i axl = _mm256_castsi256_si128(ax);
    const __m128i axh = _mm256_extractf128_si256(ax, 1);
    // 将 sy 转换为两个 __m128i 类型的向量
    const __m128i syl = _mm256_castsi256_si128(sy);
    const __m128i syh = _mm256_extractf128_si256(sy, 1);
    // 执行乘法并创建 16 位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    // 将 doth 和 dotl 组成的 __m256i 类型的向量传递给 sum_i16_pairs_float 函数，并返回结果
    return sum_i16_pairs_float(doth, dotl);
}

// 乘以 int8_t，两两相加两次，并作为浮点向量返回
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
    // 将 x 转换为两个 __m128i 类型的向量
    const __m128i xl = _mm256_castsi256_si128(x);
    const __m128i xh = _mm256_extractf128_si256(x, 1);
    // 将 y 转换为两个 __m128i 类型的向量
    const __m128i yl = _mm256_castsi256_si128(y);
    const __m128i yh = _mm256_extractf128_si256(y, 1);
    // 获取 x 向量的绝对值
    const __m128i axl = _mm_sign_epi8(xl, xl);
    const __m128i axh = _mm_sign_epi8(xh, xh);
    // 对 y 向量的值进行符号化
    # 使用低位字节的符号来对 yl 和 xl 进行符号扩展
    const __m128i syl = _mm_sign_epi8(yl, xl);
    # 使用高位字节的符号来对 yh 和 xh 进行符号扩展
    const __m128i syh = _mm_sign_epi8(yh, xh);
    # 执行乘法并创建16位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    # 返回16位整数对的和的浮点值
    return sum_i16_pairs_float(doth, dotl);
// 将两个输入的字节流按照每个字节的高低位进行重新排列，组成新的字节流
static inline __m128i packNibbles( __m128i bytes1, __m128i bytes2 )
{
    // 用于提取低位字节的掩码
    const __m128i lowByte = _mm_set1_epi16( 0xFF );
    // 提取 bytes1 中的高位字节
    __m128i high = _mm_andnot_si128( lowByte, bytes1 );
    // 提取 bytes1 中的低位字节
    __m128i low = _mm_and_si128( lowByte, bytes1 );
    // 将高位字节右移4位
    high = _mm_srli_epi16( high, 4 );
    // 将低位字节和右移后的高位字节进行或运算，得到重新排列后的 bytes1
    bytes1 = _mm_or_si128( low, high );
    // 重复上述步骤，对 bytes2 进行重新排列
    high = _mm_andnot_si128( lowByte, bytes2 );
    low = _mm_and_si128( lowByte, bytes2 );
    high = _mm_srli_epi16( high, 4 );
    bytes2 = _mm_or_si128( low, high );

    // 将重新排列后的 bytes1 和 bytes2 合并成一个 8 位整数的字节流
    return _mm_packus_epi16( bytes1, bytes2);
}
#endif
#elif defined(__SSSE3__)
// 水平加法，对4个4字节的浮点数进行水平相加
static inline float hsum_float_4x4(const __m128 a, const __m128 b, const __m128 c, const __m128 d) {
    // 分别对 a 和 b，c 和 d 进行水平相加
    __m128 res_0 =_mm_hadd_ps(a, b);
    __m128 res_1 =_mm_hadd_ps(c, d);
    // 对 res_0 和 res_1 进行水平相加
    __m128 res =_mm_hadd_ps(res_0, res_1);
    // 进行额外的水平相加操作，得到最终结果
    res =_mm_hadd_ps(res, res);
    res =_mm_hadd_ps(res, res);

    return _mm_cvtss_f32(res);
}
#endif // __AVX__ || __AVX2__ || __AVX512F__
#endif // defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)

#if defined(__ARM_NEON)
#if !defined(__aarch64__)

// 64-bit compatibility

// vaddvq_s16
// vpaddq_s16
// vaddvq_s32
// vaddvq_f32
// vmaxvq_f32
// vcvtnq_s32_f32

// 对输入的 8 个 16 位整数进行求和
inline static int32_t vaddvq_s16(int16x8_t v) {
    return
        (int32_t)vgetq_lane_s16(v, 0) + (int32_t)vgetq_lane_s16(v, 1) +
        (int32_t)vgetq_lane_s16(v, 2) + (int32_t)vgetq_lane_s16(v, 3) +
        (int32_t)vgetq_lane_s16(v, 4) + (int32_t)vgetq_lane_s16(v, 5) +
        (int32_t)vgetq_lane_s16(v, 6) + (int32_t)vgetq_lane_s16(v, 7);
}

// 对两个输入的 8 个 16 位整数进行两两相加
inline static int16x8_t vpaddq_s16(int16x8_t a, int16x8_t b) {
    // 分别对 a 和 b 的低位和高位进行相加
    int16x4_t a0 = vpadd_s16(vget_low_s16(a), vget_high_s16(a));
    int16x4_t b0 = vpadd_s16(vget_low_s16(b), vget_high_s16(b));
    // 将相加后的结果合并成一个 8 位整数的字节流
    return vcombine_s16(a0, b0);
}

// 对输入的 4 个 32 位整数进行求和
inline static int32_t vaddvq_s32(int32x4_t v) {
    # 返回向量 v 中第 0、1、2、3 个元素的和
    return vgetq_lane_s32(v, 0) + vgetq_lane_s32(v, 1) + vgetq_lane_s32(v, 2) + vgetq_lane_s32(v, 3);
// 定义一个内联函数，用于将 float32x4_t 类型的向量中的四个元素相加并返回结果
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}

// 定义一个内联函数，用于找到 float32x4_t 类型的向量中的最大值并返回结果
inline static float vmaxvq_f32(float32x4_t v) {
    return
        MAX(MAX(vgetq_lane_f32(v, 0), vgetq_lane_f32(v, 1)),
            MAX(vgetq_lane_f32(v, 2), vgetq_lane_f32(v, 3)));
}

// 定义一个内联函数，用于将 float32x4_t 类型的向量中的浮点数元素转换为整数并返回结果
inline static int32x4_t vcvtnq_s32_f32(float32x4_t v) {
    int32x4_t res;

    res[0] = roundf(vgetq_lane_f32(v, 0));
    res[1] = roundf(vgetq_lane_f32(v, 1));
    res[2] = roundf(vgetq_lane_f32(v, 2));
    res[3] = roundf(vgetq_lane_f32(v, 3));

    return res;
}

// 定义一个结构体 ggml_int16x8x2_t，包含两个 int16x8_t 类型的向量
typedef struct ggml_int16x8x2_t {
    int16x8_t val[2];
} ggml_int16x8x2_t;

// 定义一个内联函数，用于加载 int16_t 类型的数据到 ggml_int16x8x2_t 结构体中
inline static ggml_int16x8x2_t ggml_vld1q_s16_x2(const int16_t * ptr) {
    ggml_int16x8x2_t res;

    res.val[0] = vld1q_s16(ptr + 0);
    res.val[1] = vld1q_s16(ptr + 8);

    return res;
}

// 定义一个结构体 ggml_uint8x16x2_t，包含两个 uint8x16_t 类型的向量
typedef struct ggml_uint8x16x2_t {
    uint8x16_t val[2];
} ggml_uint8x16x2_t;

// 定义一个内联函数，用于加载 uint8_t 类型的数据到 ggml_uint8x16x2_t 结构体中
inline static ggml_uint8x16x2_t ggml_vld1q_u8_x2(const uint8_t * ptr) {
    ggml_uint8x16x2_t res;

    res.val[0] = vld1q_u8(ptr + 0);
    res.val[1] = vld1q_u8(ptr + 16);

    return res;
}

// 定义一个结构体 ggml_uint8x16x4_t，包含四个 uint8x16_t 类型的向量
typedef struct ggml_uint8x16x4_t {
    uint8x16_t val[4];
} ggml_uint8x16x4_t;

// 定义一个内联函数，用于加载 uint8_t 类型的数据到 ggml_uint8x16x4_t 结构体中
inline static ggml_uint8x16x4_t ggml_vld1q_u8_x4(const uint8_t * ptr) {
    ggml_uint8x16x4_t res;

    res.val[0] = vld1q_u8(ptr + 0);
    res.val[1] = vld1q_u8(ptr + 16);
    res.val[2] = vld1q_u8(ptr + 32);
    res.val[3] = vld1q_u8(ptr + 48);

    return res;
}

// 定义一个结构体 ggml_int8x16x2_t，包含两个 int8x16_t 类型的向量
typedef struct ggml_int8x16x2_t {
    int8x16_t val[2];
} ggml_int8x16x2_t;

// 定义一个内联函数，用于加载 int8_t 类��的数据到 ggml_int8x16x2_t 结构体中
inline static ggml_int8x16x2_t ggml_vld1q_s8_x2(const int8_t * ptr) {
    ggml_int8x16x2_t res;

    res.val[0] = vld1q_s8(ptr + 0);
    res.val[1] = vld1q_s8(ptr + 16);

    return res;
}

// 定义一个结构体 ggml_int8x16x4_t，包含四个 int8x16_t 类型的向量
typedef struct ggml_int8x16x4_t {
    int8x16_t val[4];
} ggml_int8x16x4_t;
// 定义一个内联函数，用于加载指针指向的内存中的数据到一个包含四个 int8x16 类型的结构体中
inline static ggml_int8x16x4_t ggml_vld1q_s8_x4(const int8_t * ptr) {
    ggml_int8x16x4_t res; // 定义一个包含四个 int8x16 类型的结构体变量 res

    res.val[0] = vld1q_s8(ptr + 0); // 将 ptr + 0 处的数据加载到 res 结构体的第一个元素中
    res.val[1] = vld1q_s8(ptr + 16); // 将 ptr + 16 处的数据加载到 res 结构体的第二个元素中
    res.val[2] = vld1q_s8(ptr + 32); // 将 ptr + 32 处的数据加载到 res 结构体的第三个元素中
    res.val[3] = vld1q_s8(ptr + 48); // 将 ptr + 48 处的数据加载到 res 结构体的第四个元素中

    return res; // 返回包含加载数据的结构体
}

#else

// 定义一系列宏，用于将特定类型重命名为另一种类型
#define ggml_int16x8x2_t  int16x8x2_t
#define ggml_uint8x16x2_t uint8x16x2_t
#define ggml_uint8x16x4_t uint8x16x4_t
#define ggml_int8x16x2_t  int8x16x2_t
#define ggml_int8x16x4_t  int8x16x4_t

// 定义一系列宏，用于将特定函数重命名为另一种函数
#define ggml_vld1q_s16_x2 vld1q_s16_x2
#define ggml_vld1q_u8_x2  vld1q_u8_x2
#define ggml_vld1q_u8_x4  vld1q_u8_x4
#define ggml_vld1q_s8_x2  vld1q_s8_x2
#define ggml_vld1q_s8_x4  vld1q_s8_x4

#endif
#endif

#if defined(__ARM_NEON) || defined(__wasm_simd128__)
// 定义一系列宏，用于生成预先计算的表格，用于将 8 位数据扩展为 8 个字节
#define B1(c,s,n)  0x ## n ## c ,  0x ## n ## s
#define B2(c,s,n) B1(c,s,n ## c), B1(c,s,n ## s)
#define B3(c,s,n) B2(c,s,n ## c), B2(c,s,n ## s)
#define B4(c,s,n) B3(c,s,n ## c), B3(c,s,n ## s)
#define B5(c,s,n) B4(c,s,n ## c), B4(c,s,n ## s)
#define B6(c,s,n) B5(c,s,n ## c), B5(c,s,n ## s)
#define B7(c,s,n) B6(c,s,n ## c), B6(c,s,n ## s)
#define B8(c,s  ) B7(c,s,     c), B7(c,s,     s)

// 预先计算的表格，用于将 8 位数据扩展为 8 个字节
static const uint64_t table_b2b_0[1 << 8] = { B8(00, 10) }; // ( b) << 4
static const uint64_t table_b2b_1[1 << 8] = { B8(10, 00) }; // (!b) << 4
#endif

// 参考实现，用于确定性地创建模型文件
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k) {
    static const int qk = QK4_0; // 定义一个静态常量 qk，值为 QK4_0

    assert(k % qk == 0); // 断言 k 能被 qk 整除

    const int nb = k / qk; // 计算 nb，值为 k 除以 qk 的结果
    // 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 初始化绝对值最大值和最大值
        float amax = 0.0f; // absolute max
        float max  = 0.0f;

        // 遍历 qk 次，j 从 0 到 qk-1
        for (int j = 0; j < qk; j++) {
            // 获取 x[i*qk + j] 的值
            const float v = x[i*qk + j];
            // 如果 v 的绝对值大于 amax，则更新 amax 和 max
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算 d 和 id
        const float d  = max / -8;
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换成 FP16 格式，并赋值给 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 遍历 qk/2 次，j 从 0 到 qk/2-1
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            // 将 x0 和 x1 转换成 uint8_t 类型，并限制在 0 到 15 之间
            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 8.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 8.5f));

            // 将 xi0 和 xi1 组合成一个字节，并赋值给 y[i].qs[j]
            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
// 将输入数组 x 进行量化，结果存储到 y 中，每个块的大小为 qk
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_0_reference 函数进行量化
    quantize_row_q4_0_reference(x, y, k);
}

// 对输入数组 x 进行量化，结果存储到 y 中，每个块的大小为 qk
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k) {
    // 定义每个块的大小为 QK4_1
    const int qk = QK4_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算块的数量
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历当前块内的元素
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];

            // 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算量化因子和其倒数
        const float d  = (max - min) / ((1 << 4) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将量化因子和最小值存储到 y 中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 对当前块内的元素进行量化
        for (int j = 0; j < qk/2; ++j) {
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 0.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 0.5f));

            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
}

// 将输入数组 x 进行量化，结果存储到 y 中，每个块的大小为 qk
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_1_reference 函数进行量化
    quantize_row_q4_1_reference(x, y, k);
}

// 对输入数组 x 进行量化，结果存储到 y 中，每个块的大小为 qk
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k) {
    // 定义每个块的大小为 QK5_0
    static const int qk = QK5_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算块的数量
    const int nb = k / qk;
    // 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 初始化绝对值最大值和最大值
        float amax = 0.0f; // absolute max
        float max  = 0.0f;

        // 遍历 qk 次，j 从 0 到 qk-1
        for (int j = 0; j < qk; j++) {
            // 获取 x[i*qk + j] 的值
            const float v = x[i*qk + j];
            // 如果 v 的绝对值大于 amax，则更新 amax 和 max
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算 d 和 id
        const float d  = max / -16;
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换成 FP16 格式，并赋值给 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 初始化 qh 为 0
        uint32_t qh = 0;

        // 遍历 qk/2 次，j 从 0 到 qk/2-1
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            // 将 x0 和 x1 转换成 5 位无符号整数，并存储在 y[i].qs[j] 中
            const uint8_t xi0 = MIN(31, (int8_t)(x0 + 16.5f));
            const uint8_t xi1 = MIN(31, (int8_t)(x1 + 16.5f));
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第 5 位，并根据位置存储在 qh 中
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到 y[i].qh 中
        memcpy(&y[i].qh, &qh, sizeof(qh));
    }
// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q5_0_reference 函数进行量化
    quantize_row_q5_0_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k) {
    // 定义量化的步长 qk
    const int qk = QK5_1;

    // 断言输入数组 x 的长度是 qk 的整数倍
    assert(k % qk == 0);

    // 计算结构体数组 y 的长度
    const int nb = k / qk;

    // 遍历结构体数组 y
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历每个结构体中的元素
        for (int j = 0; j < qk; j++) {
            // 获取当前元素的值
            const float v = x[i*qk + j];

            // 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算量化的步长和其倒数
        const float d  = (max - min) / ((1 << 5) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将步长和最小值存储到结构体中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 初始化用于存储量化结果的变量
        uint32_t qh = 0;

        // 遍历每个结构体中的元素
        for (int j = 0; j < qk/2; ++j) {
            // 计算量化结果并存储到结构体中
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            const uint8_t xi0 = (uint8_t)(x0 + 0.5f);
            const uint8_t xi1 = (uint8_t)(x1 + 0.5f);

            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第五位并存储到 qh 变量中
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 变量的值拷贝到结构体中
        memcpy(&y[i].qh, &qh, sizeof(y[i].qh));
    }
}

// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q5_1_reference 函数进行量化
    quantize_row_q5_1_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 QK8_0 个元素
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k) {
    // 断言输入数组 x 的长度是 QK8_0 的整数倍
    assert(k % QK8_0 == 0);
    // 计算结构体数组 y 的长度
    const int nb = k / QK8_0;
    # 遍历 nb 次，nb 为循环次数
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值，初始化为 0.0

        # 遍历 QK8_0 次，QK8_0 为循环次数
        for (int j = 0; j < QK8_0; j++) {
            # 获取 x 数组中的值
            const float v = x[i*QK8_0 + j];
            # 更新绝对值最大值
            amax = MAX(amax, fabsf(v));
        }

        # 计算 d 值，为绝对值最大值除以 ((1 << 7) - 1)
        const float d = amax / ((1 << 7) - 1);
        # 计算 id 值，如果 d 不为 0，则为 1/d，否则为 0
        const float id = d ? 1.0f/d : 0.0f;

        # 将 d 转换为 FP16 格式，并赋值给 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        # 遍历 QK8_0 次，QK8_0 为循环次数
        for (int j = 0; j < QK8_0; ++j) {
            # 计算 x0 值，为 x[i*QK8_0 + j] 乘以 id
            const float x0 = x[i*QK8_0 + j]*id;
            # 将 x0 四舍五入后赋值给 y[i].qs[j]
            y[i].qs[j] = roundf(x0);
        }
    }
}
// 以 QK8_0 为单位量化输入向量 x，并将结果存储在 vy 中，k 为输入向量的长度
void quantize_row_q8_0(const float * restrict x, void * restrict vy, int k) {
    // 断言 QK8_0 的值为 32
    assert(QK8_0 == 32);
    // 断言 k 能够被 QK8_0 整除
    assert(k % QK8_0 == 0);
    // 计算需要处理的块数
    const int nb = k / QK8_0;

    // 将 vy 强制类型转换为 block_q8_0 指针，并命名为 y
    block_q8_0 * restrict y = vy;

#if defined(__ARM_NEON)
    // 使用 NEON 指令集进行量化计算
    for (int i = 0; i < nb; i++) {
        // 定义存储 8 个 float32x4_t 类型数据的数组
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 从输入向量 x 中加载 8 个 float32x4_t 类型数据到 srcv 数组
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算 srcv 数组中每个元素的绝对值，存储到 asrcv 数组
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算 asrcv 数组中每两个元素的最大值，存储到 amaxv 数组
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 计算 amaxv 数组中每两个元素的最大值，存储到 amaxv 数组
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 计算 amaxv 数组中每两个元素的最大值，存储到 amaxv 数组
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv[0] 中的最大值，存储到 amax 中
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算量化因子 d 和其倒数 id
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 格式，并存储到 y[i].d 中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 使用量化因子 id 对 srcv 数组中的数据进行量化，并存储到 y[i].qs 数组中
        for (int j = 0; j < 8; j++) {
            const float32x4_t v  = vmulq_n_f32(srcv[j], id);
            const int32x4_t   vi = vcvtnq_s32_f32(v);

            y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
            y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
            y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
            y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);
        }
    }
#elif defined(__wasm_simd128__)
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 定义并初始化 srcv 数组，存储 8 个 v128_t 类型的数据
        v128_t srcv [8];
        // 定义并初始化 asrcv 数组，存储 8 个 v128_t 类型的数据
        v128_t asrcv[8];
        // 定义并初始化 amaxv 数组，存储 8 个 v128_t 类型的数据
        v128_t amaxv[8];

        // 遍历 srcv 数组，从 x 数组中加载数据到 srcv 数组中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        // 遍历 asrcv 数组，对 srcv 数组中的数据取绝对值并存储到 asrcv 数组中
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        // 计算每两个相邻元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        // 计算每四个相邻元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        // 计算每八个相邻元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv 数组中的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        // 计算 d 的值
        const float d = amax / ((1 << 7) - 1);
        // 计算 id 的值
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 类型并存储到 y[i].d 中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 遍历 srcv 数组，对每个元素进行一系列计算并存储到 y[i].qs 数组中
        for (int j = 0; j < 8; j++) {
            const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
            const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

            y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
            y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
            y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
            y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);
        }
    }
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果编译器支持 AVX2 或 AVX 指令集，则执行以下代码
    for (int i = 0; i < nb; i++) {
        // 将 32 个元素加载到 4 个 AVX 向量中
        __m256 v0 = _mm256_loadu_ps( x );
        __m256 v1 = _mm256_loadu_ps( x + 8 );
        __m256 v2 = _mm256_loadu_ps( x + 16 );
        __m256 v3 = _mm256_loadu_ps( x + 24 );
        x += 32;

        // 计算该块的 max(abs(e))
        const __m256 signBit = _mm256_set1_ps( -0.0f );
        __m256 maxAbs = _mm256_andnot_ps( signBit, v0 );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v1 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v2 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v3 ) );

        // 提取 maxAbs 的高低 128 位，进行 max 操作
        __m128 max4 = _mm_max_ps( _mm256_extractf128_ps( maxAbs, 1 ), _mm256_castps256_ps128( maxAbs ) );
        max4 = _mm_max_ps( max4, _mm_movehl_ps( max4, max4 ) );
        max4 = _mm_max_ss( max4, _mm_movehdup_ps( max4 ) );
        const float maxScalar = _mm_cvtss_f32( max4 );

        // 对这些浮点数进行量化
        const float d = maxScalar / 127.f;
        y[i].d = GGML_FP32_TO_FP16(d);
        const float id = ( maxScalar != 0.0f ) ? 127.f / maxScalar : 0.0f;
        const __m256 mul = _mm256_set1_ps( id );

        // 应用乘法器
        v0 = _mm256_mul_ps( v0, mul );
        v1 = _mm256_mul_ps( v1, mul );
        v2 = _mm256_mul_ps( v2, mul );
        v3 = _mm256_mul_ps( v3, mul );

        // 四舍五入到最近的整数
        v0 = _mm256_round_ps( v0, _MM_ROUND_NEAREST );
        v1 = _mm256_round_ps( v1, _MM_ROUND_NEAREST );
        v2 = _mm256_round_ps( v2, _MM_ROUND_NEAREST );
        v3 = _mm256_round_ps( v3, _MM_ROUND_NEAREST );

        // 将浮点数转换为整数
        __m256i i0 = _mm256_cvtps_epi32( v0 );
        __m256i i1 = _mm256_cvtps_epi32( v1 );
        __m256i i2 = _mm256_cvtps_epi32( v2 );
        __m256i i3 = _mm256_cvtps_epi32( v3 );
#if defined(__AVX2__)
        // 如果定义了 AVX2 指令集

        // 将 int32 转换为 int16
        i0 = _mm256_packs_epi32( i0, i1 );    // 0, 1, 2, 3,  8, 9, 10, 11,  4, 5, 6, 7, 12, 13, 14, 15
        i2 = _mm256_packs_epi32( i2, i3 );    // 16, 17, 18, 19,  24, 25, 26, 27,  20, 21, 22, 23, 28, 29, 30, 31
                                            // 将 int16 转换为 int8
        i0 = _mm256_packs_epi16( i0, i2 );    // 0, 1, 2, 3,  8, 9, 10, 11,  16, 17, 18, 19,  24, 25, 26, 27,  4, 5, 6, 7, 12, 13, 14, 15, 20, 21, 22, 23, 28, 29, 30, 31

        // 我们得到了宝贵的有符号字节，但是顺序现在是错误的
        // 这些 AVX2 pack 指令独立处理 16 字节块
        // 以下指令用于修复顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );

        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 因为我们在 AVX 中没有一些必要的函数，
        // 我们将寄存器分成两半，并从 SSE 调用 AVX2 的模拟函数
        __m128i ni0 = _mm256_castsi256_si128( i0 );
        __m128i ni1 = _mm256_extractf128_si256( i0, 1);
        __m128i ni2 = _mm256_castsi256_si128( i1 );
        __m128i ni3 = _mm256_extractf128_si256( i1, 1);
        __m128i ni4 = _mm256_castsi256_si128( i2 );
        __m128i ni5 = _mm256_extractf128_si256( i2, 1);
        __m128i ni6 = _mm256_castsi256_si128( i3 );
        __m128i ni7 = _mm256_extractf128_si256( i3, 1);

        // 将 int32 转换为 int16
        ni0 = _mm_packs_epi32( ni0, ni1 );
        ni2 = _mm_packs_epi32( ni2, ni3 );
        ni4 = _mm_packs_epi32( ni4, ni5 );
        ni6 = _mm_packs_epi32( ni6, ni7 );
        // 将 int16 转换为 int8
        ni0 = _mm_packs_epi16( ni0, ni2 );
        ni4 = _mm_packs_epi16( ni4, ni6 );

        _mm_storeu_si128((__m128i *)(y[i].qs +  0), ni0);
        _mm_storeu_si128((__m128i *)(y[i].qs + 16), ni4);
#endif
    }
#elif defined(__riscv_v_intrinsic)
    // 设置向量长度为 e32m4
    size_t vl = __riscv_vsetvl_e32m4(QK8_0);

    // 遍历每个元素
    for (int i = 0; i < nb; i++) {
        // 加载元素
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_0, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 初始化临时变量
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0f, vl);
        // 计算最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        // 将最大值转换为标量
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        // 计算 d 和 id
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 格式并存储到 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 计算 x0
        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        // 转换为整数
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        // 存储结果
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);
    }
#else
    GGML_UNUSED(nb);
    // 如果不是 ARM 平台，则忽略 nb 变量
    quantize_row_q8_0_reference(x, y, k);
#endif
}

// reference implementation for deterministic creation of model files
// 用于确定性创建模型文件的参考实现
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k) {
    assert(QK8_1 == 32);
    // 断言 QK8_1 的值为 32
    assert(k % QK8_1 == 0);
    // 断言 k 能被 QK8_1 整除
    const int nb = k / QK8_1;

    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // absolute max
        // 初始化绝对值最大值为 0

        for (int j = 0; j < QK8_1; j++) {
            const float v = x[i*QK8_1 + j];
            // 获取 x 数组中的值
            amax = MAX(amax, fabsf(v));
            // 计算绝对值最大值
        }

        const float d = amax / ((1 << 7) - 1);
        // 计算比例因子 d
        const float id = d ? 1.0f/d : 0.0f;
        // 计算倒数 id

        y[i].d = d;
        // 将比例因子 d 存入 y 结构体中

        int sum = 0;

        for (int j = 0; j < QK8_1/2; ++j) {
            const float v0 = x[i*QK8_1           + j]*id;
            const float v1 = x[i*QK8_1 + QK8_1/2 + j]*id;

            y[i].qs[          j] = roundf(v0);
            y[i].qs[QK8_1/2 + j] = roundf(v1);
            // 对 x 数组中的值进行量化，并存入 y 结构体中

            sum += y[i].qs[          j];
            sum += y[i].qs[QK8_1/2 + j];
            // 计算量化后的值的和
        }

        y[i].s = sum*d;
        // 将量化后的值的和乘以比例因子存入 y 结构体中
    }
}

void quantize_row_q8_1(const float * restrict x, void * restrict vy, int k) {
    assert(k % QK8_1 == 0);
    // 断言 k 能被 QK8_1 整除
    const int nb = k / QK8_1;

    block_q8_1 * restrict y = vy;
    // 将 vy 强制转换为 block_q8_1 结构体指针

#if defined(__ARM_NEON)
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 声明并初始化存储 8 个 float32x4_t 类型的变量
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 通过 NEON 指令加载 x 数组中的数据到 srcv 数组中
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算 srcv 数组中每个元素的绝对值并存储到 asrcv 数组中
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算 asrcv 数组中相邻两个元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 计算 amaxv 数组中相邻两个元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 计算 amaxv 数组中相邻两个元素的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv[0] 中的最大值并存储到 amax 变量中
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算 d 和 id 变量的值
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 的值存储到 y[i].d 中
        y[i].d = d;

        // 初始化 accv 变量为全 0
        int32x4_t accv = vdupq_n_s32(0);

        // 循环遍历 8 次
        for (int j = 0; j < 8; j++) {
            // 计算 v 和 vi 变量的值
            const float32x4_t v  = vmulq_n_f32(srcv[j], id);
            const int32x4_t   vi = vcvtnq_s32_f32(v);

            // 将 vi 中的值存储到 y[i].qs 数组中
            y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
            y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
            y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
            y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);

            // 将 vi 中的值加到 accv 变量中
            accv = vaddq_s32(accv, vi);
        }

        // 计算 y[i].s 的值并存储到 y[i].s 中
        y[i].s = d * vaddvq_s32(accv);
    }
# 如果编译器定义了 __wasm_simd128__，则执行以下代码块
elif defined(__wasm_simd128__):
    # 遍历 nb 次
    for (int i = 0; i < nb; i++):
        # 定义 8 个 v128_t 类型的变量
        v128_t srcv [8];
        v128_t asrcv[8];
        v128_t amaxv[8];

        # 遍历 8 次，从内存中加载数据到 srcv 数组中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        # 遍历 8 次，计算 srcv 数组中元素的绝对值，存入 asrcv 数组中
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        # 遍历 4 次，计算 asrcv 数组中相邻两个元素的最大值，存入 amaxv 数组中
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        # 遍历 2 次，计算 amaxv 数组中相邻两个元素的最大值，存入 amaxv 数组中
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        # 遍历 1 次，计算 amaxv 数组中相邻两个元素的最大值，存入 amaxv 数组中
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        # 计算 amaxv 数组中所有元素的最大值，存入 amax 变量中
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        # 计算 d 变量的值
        const float d = amax / ((1 << 7) - 1);
        # 计算 id 变量的值
        const float id = d ? 1.0f/d : 0.0f;

        # 将 d 变量的值赋给 y[i].d
        y[i].d = d;

        # 定义一个 v128_t 类型的变量 accv，并初始化为 0
        v128_t accv = wasm_i32x4_splat(0);

        # 遍历 8 次
        for (int j = 0; j < 8; j++):
            # 计算 v 变量的值
            const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
            # 将 v 变量的值转换为整型，并存入 vi 变量中
            const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

            # 将 vi 变量中的值分别存入 y[i].qs 数组中
            y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
            y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
            y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
            y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);

            # 将 vi 变量的值加到 accv 变量中
            accv = wasm_i32x4_add(accv, vi);

        # 计算 y[i].s 的值
        y[i].s = d * (wasm_i32x4_extract_lane(accv, 0) +
                      wasm_i32x4_extract_lane(accv, 1) +
                      wasm_i32x4_extract_lane(accv, 2) +
                      wasm_i32x4_extract_lane(accv, 3));
    // 循环遍历，每次处理4个元素
    for (int i = 0; i < nb; i++) {
        // 将4个元素加载到4个 AVX 向量中
        __m256 v0 = _mm256_loadu_ps( x );
        __m256 v1 = _mm256_loadu_ps( x + 8 );
        __m256 v2 = _mm256_loadu_ps( x + 16 );
        __m256 v3 = _mm256_loadu_ps( x + 24 );
        x += 32;

        // 计算该块中的 max(abs(e))
        const __m256 signBit = _mm256_set1_ps( -0.0f );
        __m256 maxAbs = _mm256_andnot_ps( signBit, v0 );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v1 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v2 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v3 ) );

        // 提取最大值并转换为标量
        __m128 max4 = _mm_max_ps( _mm256_extractf128_ps( maxAbs, 1 ), _mm256_castps256_ps128( maxAbs ) );
        max4 = _mm_max_ps( max4, _mm_movehl_ps( max4, max4 ) );
        max4 = _mm_max_ss( max4, _mm_movehdup_ps( max4 ) );
        const float maxScalar = _mm_cvtss_f32( max4 );

        // 对这些浮点数进行量化
        const float d = maxScalar / 127.f;
        y[i].d = d;
        const float id = ( maxScalar != 0.0f ) ? 127.f / maxScalar : 0.0f;
        const __m256 mul = _mm256_set1_ps( id );

        // 应用乘法器
        v0 = _mm256_mul_ps( v0, mul );
        v1 = _mm256_mul_ps( v1, mul );
        v2 = _mm256_mul_ps( v2, mul );
        v3 = _mm256_mul_ps( v3, mul );

        // 四舍五入到最近的整数
        v0 = _mm256_round_ps( v0, _MM_ROUND_NEAREST );
        v1 = _mm256_round_ps( v1, _MM_ROUND_NEAREST );
        v2 = _mm256_round_ps( v2, _MM_ROUND_NEAREST );
        v3 = _mm256_round_ps( v3, _MM_ROUND_NEAREST );

        // 将浮点数转换为整数
        __m256i i0 = _mm256_cvtps_epi32( v0 );
        __m256i i1 = _mm256_cvtps_epi32( v1 );
        __m256i i2 = _mm256_cvtps_epi32( v2 );
        __m256i i3 = _mm256_cvtps_epi32( v3 );
#if defined(__AVX2__)
        // 如果定义了 AVX2 指令集

        // 计算四个整数向量的和，并将结果存储到 y[i].s
        y[i].s = d * hsum_i32_8(_mm256_add_epi32(_mm256_add_epi32(i0, i1), _mm256_add_epi32(i2, i3)));

        // 将 int32 转换为 int16
        i0 = _mm256_packs_epi32( i0, i1 );    // 0, 1, 2, 3,  8, 9, 10, 11,  4, 5, 6, 7, 12, 13, 14, 15
        i2 = _mm256_packs_epi32( i2, i3 );    // 16, 17, 18, 19,  24, 25, 26, 27,  20, 21, 22, 23, 28, 29, 30, 31
                                              // 将 int16 转换为 int8
        i0 = _mm256_packs_epi16( i0, i2 );    // 0, 1, 2, 3,  8, 9, 10, 11,  16, 17, 18, 19,  24, 25, 26, 27,  4, 5, 6, 7, 12, 13, 14, 15, 20, 21, 22, 23, 28, 29, 30, 31

        // 我们得到了宝贵的有符号字节，但是顺序现在是错误的
        // 这些 AVX2 pack 指令独立处理 16 字节的数据
        // 以下指令用于修正顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );

        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 如果不支持 AVX，将寄存器拆分成两半，并从 SSE 调用 AVX2 的模拟函数
        __m128i ni0 = _mm256_castsi256_si128( i0 );
        __m128i ni1 = _mm256_extractf128_si256( i0, 1);
        __m128i ni2 = _mm256_castsi256_si128( i1 );
        __m128i ni3 = _mm256_extractf128_si256( i1, 1);
        __m128i ni4 = _mm256_castsi256_si128( i2 );
        __m128i ni5 = _mm256_extractf128_si256( i2, 1);
        __m128i ni6 = _mm256_castsi256_si128( i3 );
        __m128i ni7 = _mm256_extractf128_si256( i3, 1);

        // 计算量化值的和，并设置 y[i].s
        const __m128i s0 = _mm_add_epi32(_mm_add_epi32(ni0, ni1), _mm_add_epi32(ni2, ni3));
        const __m128i s1 = _mm_add_epi32(_mm_add_epi32(ni4, ni5), _mm_add_epi32(ni6, ni7));
        y[i].s = d * hsum_i32_4(_mm_add_epi32(s0, s1));

        // 将 int32 转换为 int16
        ni0 = _mm_packs_epi32( ni0, ni1 );
        ni2 = _mm_packs_epi32( ni2, ni3 );
        ni4 = _mm_packs_epi32( ni4, ni5 );
        ni6 = _mm_packs_epi32( ni6, ni7 );
        // 将 int16 转换为 int8
        ni0 = _mm_packs_epi16( ni0, ni2 );
        ni4 = _mm_packs_epi16( ni4, ni6 );

        _mm_storeu_si128((__m128i *)(y[i].qs +  0), ni0);
        _mm_storeu_si128((__m128i *)(y[i].qs + 16), ni4);
#endif
    }
#elif defined(__riscv_v_intrinsic)
    // 设置 VL 寄存器的值
    size_t vl = __riscv_vsetvl_e32m4(QK8_1);
    for (int i = 0; i < nb; i++) {
        // 遍历处理每个元素
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_1, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 初始化临时变量为0
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0, vl);
        // 计算最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        // 将最大值转换为标量
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        // 计算比例因子
        const float d  = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 存储比例因子
        y[i].d = d;

        // 对输入数据进行缩放
        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        // 转换为整数
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        // 存储结果
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);

        // 计算 y[i].s 的和
        vint16m1_t tmp2 = __riscv_vmv_v_x_i16m1(0, vl);
        vint16m1_t vwrs = __riscv_vwredsum_vs_i8m1_i16m1(vs, tmp2, vl);

        // 设置 y[i].s
        int sum = __riscv_vmv_x_s_i16m1_i16(vwrs);
        y[i].s = sum*d;
    }
#else
    // 如果不满足条件，则使用 GGML_UNUSED 宏来标记 nb 未使用
    GGML_UNUSED(nb);
    // scalar
    // 对行进行量化
    quantize_row_q8_1_reference(x, y, k);
#endif
}

void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k) {
    static const int qk = QK4_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1
            const int x0 = (x[i].qs[j] & 0x0F) - 8;
            const int x1 = (x[i].qs[j] >>   4) - 8;

            // 计算 y[i*qk + j + 0] 和 y[i*qk + j + qk/2]
            y[i*qk + j + 0   ] = x0*d;
            y[i*qk + j + qk/2] = x1*d;
        }
    }
}

void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k) {
    static const int qk = QK4_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 和 x[i].m 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        // 遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1
            const int x0 = (x[i].qs[j] & 0x0F);
            const int x1 = (x[i].qs[j] >>   4);

            // 计算 y[i*qk + j + 0] 和 y[i*qk + j + qk/2]
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
}

void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k) {
    static const int qk = QK5_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 从 x[i].qh 中复制 4 个字节到 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 计算 xh_0 和 xh_1
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            // 计算 x0 和 x1
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            // 计算 y[i*qk + j + 0] 和 y[i*qk + j + qk/2]
            y[i*qk + j + 0   ] = x0*d;
            y[i*qk + j + qk/2] = x1*d;
        }
    }
}

void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k) {
    static const int qk = QK5_1;
    # 确保 k 能够整除 qk，否则触发断言错误
    assert(k % qk == 0);

    # 计算出数组 x 的长度
    const int nb = k / qk;

    # 遍历数组 x 的每个元素
    for (int i = 0; i < nb; i++) {
        # 将 x[i].d 从 FP16 格式转换为 FP32 格式
        const float d = GGML_FP16_TO_FP32(x[i].d);
        # 将 x[i].m 从 FP16 格式转换为 FP32 格式
        const float m = GGML_FP16_TO_FP32(x[i].m);

        # 从 x[i].qh 中复制出一个 uint32_t 类型的值
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 遍历每个 qk/2 的值
        for (int j = 0; j < qk/2; ++j) {
            # 从 qh 中提取出 xh_0 和 xh_1 的值
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            # 计算出 x0 和 x1 的值
            const int x0 = (x[i].qs[j] & 0x0F) | xh_0;
            const int x1 = (x[i].qs[j] >>   4) | xh_1;

            # 计算出 y[i*qk + j + 0] 和 y[i*qk + j + qk/2] 的值
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
}

// 对输入的一维量化数据进行反量化，将结果存储到输出数组中
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK8_0;

    // 断言 k 能够整除 qk
    assert(k % qk == 0);

    // 计算块数
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化数据转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个量化数据，进行反量化
        for (int j = 0; j < qk; ++j) {
            y[i*qk + j] = x[i].qs[j]*d;
        }
    }
}

//
// 2-6 bit quantization in super-blocks
//

//
// ===================== Helper functions
//

// 寻找最接近的整数
static inline int nearest_int(float fval) {
    // 断言 fval 不大于 4194303
    assert(fval <= 4194303.f);
    float val = fval + 12582912.f;
    int i; memcpy(&i, &val, sizeof(int));
    return (i & 0x007fffff) - 0x00400000;
}

// 生成量化数据
static float make_qx_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, int rmse_type) {
    float max = 0;
    float amax = 0;
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    if (amax < 1e-30f) { // 如果所有值都为零
        for (int i = 0; i < n; ++i) {
            L[i] = 0;
        }
        return 0.f;
    }
    float iscale = -nmax / max;
    if (rmse_type == 0) {
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale * x[i]);
            L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
        }
        return 1/iscale;
    }
    bool return_early = false;
    if (rmse_type < 0) {
        rmse_type = -rmse_type;
        return_early = true;
    }
    int weight_type = rmse_type%2;
    float sumlx = 0;
    float suml2 = 0;
    for (int i = 0; i < n; ++i) {
        int l = nearest_int(iscale * x[i]);
        l = MAX(-nmax, MIN(nmax-1, l));
        L[i] = l + nmax;
        float w = weight_type == 1 ? x[i] * x[i] : 1;
        sumlx += w*x[i]*l;
        suml2 += w*l*l;
    }
    float scale = sumlx/suml2;
    if (return_early) return suml2 > 0 ? 0.5f*(scale + 1/iscale) : 1/iscale;
    float best = scale * sumlx;
    # 遍历 is 取值范围为 -9 到 9
    for (int is = -9; is <= 9; ++is) {
        # 如果 is 等于 0，则跳过本次循环
        if (is == 0) {
            continue;
        }
        # 计算 iscale 的值
        iscale = -(nmax + 0.1f*is) / max;
        # 初始化 sumlx 和 suml2
        sumlx = suml2 = 0;
        # 遍历 n 个元素
        for (int i = 0; i < n; ++i) {
            # 计算最近整数 l
            int l = nearest_int(iscale * x[i]);
            # 将 l 限制在 -nmax 和 nmax-1 之间
            l = MAX(-nmax, MIN(nmax-1, l));
            # 根据权重类型计算 w
            float w = weight_type == 1 ? x[i] * x[i] : 1;
            # 更新 sumlx 和 suml2
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        # 如果 suml2 大于 0 并且 sumlx*sumlx 大于 best*suml2
        if (suml2 > 0 && sumlx*sumlx > best*suml2) {
            # 更新 L 数组
            for (int i = 0; i < n; ++i) {
                int l = nearest_int(iscale * x[i]);
                L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
            }
            # 更新 scale 和 best
            scale = sumlx/suml2; best = scale*sumlx;
        }
    }
    # 返回 scale 的值
    return scale;
// 计算第三个四分位数的分位数，返回分位数
static float make_q3_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, bool do_rmse) {
    // 初始化最大值和绝对值最大值
    float max = 0;
    float amax = 0;
    // 遍历数组 x
    for (int i = 0; i < n; ++i) {
        // 计算绝对值
        float ax = fabsf(x[i]);
        // 更新最大值和绝对值最大值
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果绝对值最大值为 0，即所有值都为 0
    if (!amax) { // all zero
        // 将 L 数组中的所有元素设为 0
        for (int i = 0; i < n; ++i) { L[i] = 0; }
        // 返回 0
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    // 如果需要计算均方根误差
    if (do_rmse) {
        // 初始化变量
        float sumlx = 0;
        float suml2 = 0;
        // 遍历数组 x
        for (int i = 0; i < n; ++i) {
            // 计算最近整数
            int l = nearest_int(iscale * x[i]);
            // 将 l 限制在 -nmax 和 nmax-1 之间
            l = MAX(-nmax, MIN(nmax-1, l));
            // 更新 L 数组中的值
            L[i] = l;
            // 计算权重
            float w = x[i]*x[i];
            // 更新 sumlx 和 suml2
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        // 迭代计算
        for (int itry = 0; itry < 5; ++itry) {
            int n_changed = 0;
            // 遍历数组 x
            for (int i = 0; i < n; ++i) {
                // 计算权重
                float w = x[i]*x[i];
                // 更新 slx
                float slx = sumlx - w*x[i]*L[i];
                // 如果 slx 大于 0
                if (slx > 0) {
                    // 更新 sl2
                    float sl2 = suml2 - w*L[i]*L[i];
                    // 计算新的 l
                    int new_l = nearest_int(x[i] * sl2 / slx);
                    // 将 new_l 限制在 -nmax 和 nmax-1 之间
                    new_l = MAX(-nmax, MIN(nmax-1, new_l));
                    // 如果 new_l 不等于 L[i]
                    if (new_l != L[i]) {
                        // 更新 slx 和 sl2
                        slx += w*x[i]*new_l;
                        sl2 += w*new_l*new_l;
                        // 如果满足条件
                        if (sl2 > 0 && slx*slx*suml2 > sumlx*sumlx*sl2) {
                            // 更新 L[i]，sumlx，suml2，n_changed
                            L[i] = new_l; sumlx = slx; suml2 = sl2;
                            ++n_changed;
                        }
                    }
                }
            }
            // 如果没有改变
            if (!n_changed) {
                break;
            }
        }
        // 将 L 数组中的值加上 nmax
        for (int i = 0; i < n; ++i) {
            L[i] += nmax;
        }
        // 返回 sumlx / suml2
        return sumlx / suml2;
    }
    // 如果不需要计算均方根误差
    for (int i = 0; i < n; ++i) {
        // 计算最近整数
        int l = nearest_int(iscale * x[i]);
        // 将 l 限制在 -nmax 和 nmax-1 之间
        l = MAX(-nmax, MIN(nmax-1, l));
        // 更新 L 数组中的值
        L[i] = l + nmax;
    }
    // 返回 1/iscale
    return 1/iscale;
}
# 计算并返回一维量化的比例因子
static float make_qkx1_quants(int n, int nmax, const float * restrict x, uint8_t * restrict L, float * restrict the_min,
        int ntry, float alpha) {
    # 初始化最小值和最大值为数组第一个元素的值
    float min = x[0];
    float max = x[0];
    # 遍历数组，更新最小值和最大值
    for (int i = 1; i < n; ++i) {
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
    }
    # 如果最大值等于最小值，将量化数组全部置为0，更新最小值为0，返回0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = 0;
        return 0.f;
    }
    # 如果最小值大于0，将最小值置为0
    if (min > 0) min = 0;
    # 计算比例因子
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    # 迭代ntry次
    for (int itry = 0; itry < ntry; ++itry) {
        float sumlx = 0; int suml2 = 0;
        bool did_change = false;
        # 遍历数组，更新量化数组，计算sumlx和suml2
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale*(x[i] - min));
            l = MAX(0, MIN(nmax, l));
            if (l != L[i]) {
                L[i] = l;
                did_change = true;
            }
            sumlx += (x[i] - min)*l;
            suml2 += l*l;
        }
        # 更新比例因子
        scale = sumlx/suml2;
        float sum = 0;
        # 计算和值
        for (int i = 0; i < n; ++i) {
            sum += x[i] - scale*L[i];
        }
        # 更新最小值
        min = alpha*min + (1 - alpha)*sum/n;
        if (min > 0) min = 0;
        iscale = 1/scale;
        if (!did_change) break;
    }
    # 更新最小值，返回比例因子
    *the_min = -min;
    return scale;
}

# 计算并返回二维量化的比例因子
static float make_qkx2_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    # 初始化最小值和最大值为数组第一个元素的值，初始化权重和为第一个元素的权重，初始化加权和为第一个元素的值乘以权重
    float min = x[0];
    float max = x[0];
    float sum_w = weights[0];
    float sum_x = sum_w * x[0];
    # 遍历数组，更新最小值、最大值、权重和、加权和
#ifdef HAVE_BUGGY_APPLE_LINKER
    # 使用'volatile'修饰符来防止循环展开，并解决苹果ld64 1015.7版本的bug
    for (volatile int i = 1; i < n; ++i) {
#else
    for (int i = 1; i < n; ++i) {
#endif
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
        float w = weights[i];
        sum_w += w;
        sum_x += w * x[i];
    }
    # 如果最小值大于0，将最小值置为0
    if (min > 0) min = 0;
    # 如果最大值等于最小值
    if (max == min) {
        # 将 L 数组中的所有元素设置为 0
        for (int i = 0; i < n; ++i) L[i] = 0;
        # 将 the_min 指针指向的值设置为 -min
        *the_min = -min;
        # 返回 0.0
        return 0.f;
    }
    # 计算缩放比例
    float iscale = nmax/(max - min);
    # 计算反向缩放比例
    float scale = 1/iscale;
    # 初始化最佳平均绝对偏差
    float best_mad = 0;
    # 遍历数组 x
    for (int i = 0; i < n; ++i) {
        # 计算最近的整数值
        int l = nearest_int(iscale*(x[i] - min));
        # 将 L 数组中的元素限制在 0 和 nmax 之间
        L[i] = MAX(0, MIN(nmax, l));
        # 计算差值
        float diff = scale * L[i] + min - x[i];
        # 如果使用平均绝对偏差，则取绝对值，否则取平方
        diff = use_mad ? fabsf(diff) : diff * diff;
        # 获取权重
        float w = weights[i];
        # 计算最佳平均绝对偏差
        best_mad += w * diff;
    }
    # 如果步数小于 1
    if (nstep < 1) {
        # 将 the_min 指针指向的值设置为 -min
        *the_min = -min;
        # 返回缩放比例
        return scale;
    }
    # 遍历步数
    for (int is = 0; is <= nstep; ++is) {
        # 重新计算缩放比例
        iscale = (rmin + rdelta*is + nmax)/(max - min);
        # 初始化变量
        float sum_l = 0, sum_l2 = 0, sum_xl = 0;
        # 遍历数组 x
        for (int i = 0; i < n; ++i) {
            # 计算最近的整数值
            int l = nearest_int(iscale*(x[i] - min));
            # 将 l 限制在 0 和 nmax 之间
            l = MAX(0, MIN(nmax, l));
            # 将 Laux 数组中的元素设置为 l
            Laux[i] = l;
            # 获取权重
            float w = weights[i];
            # 计算和
            sum_l += w*l;
            sum_l2 += w*l*l;
            sum_xl += w*l*x[i];
        }
        # 计算行列式
        float D = sum_w * sum_l2 - sum_l * sum_l;
        # 如果行列式大于 0
        if (D > 0) {
            # 计算新的缩放比例和最小值
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;
            # 如果最小值大于 0
            if (this_min > 0) {
                # 将最小值设置为 0
                this_min = 0;
                # 重新计算缩放比例
                this_scale = sum_xl / sum_l2;
            }
            # 初始化平均绝对偏差
            float mad = 0;
            # 遍历数组 x
            for (int i = 0; i < n; ++i) {
                # 计算差值
                float diff = this_scale * Laux[i] + this_min - x[i];
                # 如果使用平均绝对偏差，则取绝对值，否则取平方
                diff = use_mad ? fabsf(diff) : diff * diff;
                # 获取权重
                float w = weights[i];
                # 计算平均绝对偏差
                mad += w * diff;
            }
            # 如果平均绝对偏差小于最佳平均绝对偏差
            if (mad < best_mad) {
                # 更新 L、scale 和 min
                for (int i = 0; i < n; ++i) {
                    L[i] = Laux[i];
                }
                best_mad = mad;
                scale = this_scale;
                min = this_min;
            }
        }
    }
    # 将 the_min 指针指向的值设置为 -min
    *the_min = -min;
    # 返回缩放比例
    return scale;
// 如果 QK_K 等于 256，则定义一个内联函数，用于获取最小比例
static inline void get_scale_min_k4(int j, const uint8_t * restrict q, uint8_t * restrict d, uint8_t * restrict m) {
    // 如果 j 小于 4，则执行以下操作
    if (j < 4) {
        // 将 q[j] 和 q[j+4] 与 63 进行按位与操作，将结果存储到 d 和 m 中
        *d = q[j] & 63; *m = q[j + 4] & 63;
    } else {
        // 否则，执行以下操作
        // 将 q[j+4] 与 0xF 进行按位与操作，将结果存储到 d 中
        // 将 q[j-4] 右移 6 位，再左移 4 位，与 q[j+4] 右移 4 位进行按位或操作，将结果存储到 m 中
        *d = (q[j+4] & 0xF) | ((q[j-4] >> 6) << 4);
        *m = (q[j+4] >>  4) | ((q[j-0] >> 6) << 4);
    }
}
#endif

// 定义一个函数，用于对输入的浮点数数组进行 2-bit 量化
void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k) {
    // 断言 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值，即 k 除以 QK_K 的结果
    const int nb = k / QK_K;

    // 定义一些变量
    uint8_t L[QK_K];
    uint8_t Laux[16];
    float   weights[16];
    float mins[QK_K/16];
    float scales[QK_K/16];

    // 定义一个常量 q4scale，并赋值为 15.0
    const float q4scale = 15.f;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 初始化最大比例和最大最小值
        float max_scale = 0; // 因为我们在减去最小值，所以比例始终为正数
        float max_min = 0;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 循环遍历 16 次
            for (int l = 0; l < 16; ++l) weights[l] = fabsf(x[16*j + l]);
            // 调用 make_qkx2_quants 函数计算比例和最小值
            scales[j] = make_qkx2_quants(16, 3, x + 16*j, weights, L + 16*j, &mins[j], Laux, -0.5f, 0.1f, 15, true);
            // 获取当前比例
            float scale = scales[j];
            // 更新最大比例
            if (scale > max_scale) {
                max_scale = scale;
            }
            // 获取当前最小值
            float min = mins[j];
            // 更新最大最小值
            if (min > max_min) {
                max_min = min;
            }
        }

        // 如果最大比例大于 0
        if (max_scale > 0) {
            // 计算缩放比例
            float iscale = q4scale/max_scale;
            // 循环遍历 QK_K/16 次
            for (int j = 0; j < QK_K/16; ++j) {
                // 计算最接近的整数值
                int l = nearest_int(iscale*scales[j]);
                // 更新 y[i].scales[j]
                y[i].scales[j] = l;
            }
            // 更新 y[i].d
            y[i].d = GGML_FP32_TO_FP16(max_scale/q4scale);
        } else {
            // 如果最大比例小于等于 0，则将 y[i].scales[j] 设置为 0
            for (int j = 0; j < QK_K/16; ++j) y[i].scales[j] = 0;
            // 将 y[i].d 设置为 0
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        // 如果最大最小值大于 0
        if (max_min > 0) {
            // 计算缩放比例
            float iscale = q4scale/max_min;
            // 循环遍历 QK_K/16 次
            for (int j = 0; j < QK_K/16; ++j) {
                // 计算最接近的整数值
                int l = nearest_int(iscale*mins[j]);
                // 更新 y[i].scales[j]
                y[i].scales[j] |= (l << 4);
            }
            // 更新 y[i].dmin
            y[i].dmin = GGML_FP32_TO_FP16(max_min/q4scale);
        } else {
            // 如果最大最小值小于等于 0，则将 y[i].dmin 设置为 0
            y[i].dmin = GGML_FP32_TO_FP16(0.f);
        }
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 d 和 dm 的值
            const float d = GGML_FP16_TO_FP32(y[i].d) * (y[i].scales[j] & 0xF);
            // 如果 d 为 0，则继续下一次循环
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * (y[i].scales[j] >> 4);
            // 循环遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算 l 的值
                int l = nearest_int((x[16*j + ii] + dm)/d);
                // 将 l 的值限制在 0 到 3 之间
                l = MAX(0, MIN(3, l));
                // 更新 L[16*j + ii]
                L[16*j + ii] = l;
            }
        }
    }
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        for (int j = 0; j < QK_K; j += 128) {
            // 循环遍历，每次增加 128
            for (int l = 0; l < 32; ++l) {
                // 内部循环，每次增加 1
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
                // 将计算结果赋值给数组元素
            }
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 循环遍历，每次增加 1
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
            // 将计算结果赋值给数组元素
        }
#endif

        x += QK_K;
        // 将 x 增加 QK_K 的值
    }
}

void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k) {
    // 对 k 取模，确保能整除 QK_K
    assert(k % QK_K == 0);
    const int nb = k / QK_K;
    // 计算 nb 的值

    for (int i = 0; i < nb; i++) {
        // 循环遍历 nb 次

        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].d 和 x[i].dmin 转换为 float 类型

        const uint8_t * q = x[i].qs;
        // 获取 x[i].qs 的地址

#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        int is = 0;
        float dl, ml;
        for (int n = 0; n < QK_K; n += 128) {
            // 循环遍历，每次增加 128
            int shift = 0;
            for (int j = 0; j < 4; ++j) {
                // 内部循环，每次增加 1
                uint8_t sc = x[i].scales[is++];
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l] >> shift) & 3)) - ml;
                // 计算结果赋值给 y 数组

                sc = x[i].scales[is++];
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3)) - ml;
                // 计算结果赋值给 y 数组

                shift += 2;
            }
            q += 32;
        }
#else
        // 计算每个元素的 dl 和 ml 值
        float dl1 = d * (x[i].scales[0] & 0xF), ml1 = min * (x[i].scales[0] >> 4);
        float dl2 = d * (x[i].scales[1] & 0xF), ml2 = min * (x[i].scales[1] >> 4);
        float dl3 = d * (x[i].scales[2] & 0xF), ml3 = min * (x[i].scales[2] >> 4);
        float dl4 = d * (x[i].scales[3] & 0xF), ml4 = min * (x[i].scales[3] >> 4);
        // 遍历每个元素，进行量化和反量化操作
        for (int l = 0; l < 16; ++l) {
            y[l+ 0] = dl1 * ((int8_t)((q[l] >> 0) & 3)) - ml1;
            y[l+16] = dl2 * ((int8_t)((q[l] >> 2) & 3)) - ml2;
            y[l+32] = dl3 * ((int8_t)((q[l] >> 4) & 3)) - ml3;
            y[l+48] = dl4 * ((int8_t)((q[l] >> 6) & 3)) - ml4;
        }
        y += QK_K;
#endif
    }
}

void quantize_row_q2_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q2_K_reference(x, vy, k);
}

size_t ggml_quantize_q2_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对输入数据进行分块量化
    for (int j = 0; j < n; j += k) {
        block_q2_K * restrict y = (block_q2_K *)dst + j/QK_K;
        quantize_row_q2_K_reference(src + j, y, k);
    }
    return (n/QK_K*sizeof(block_q2_K));
}

//========================= 3-bit (de)-quantization

void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k) {
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    int8_t L[QK_K];
    float scales[QK_K / 16];

    for (int i = 0; i < nb; i++) {

        float max_scale = 0;
        float amax = 0;
        // 计算每个块的量化因子和量化值
        for (int j = 0; j < QK_K/16; ++j) {
            scales[j] = make_q3_quants(16, 4, x + 16*j, L + 16*j, true);
            float scale = fabsf(scales[j]);
            if (scale > amax) {
                amax = scale; max_scale = scales[j];
            }
        }
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下操作
        memset(y[i].scales, 0, 12);
        // 将 y[i].scales 数组的前 12 个字节设置为 0
        if (max_scale) {
            // 如果 max_scale 存在
            float iscale = -32.f/max_scale;
            // 计算 iscale 的值
            for (int j = 0; j < QK_K/16; ++j) {
                // 遍历循环，j 从 0 到 QK_K/16
                int8_t l = nearest_int(iscale*scales[j]);
                // 计算 l 的值
                l = MAX(-32, MIN(31, l)) + 32;
                // 将 l 的值限制在 -32 到 31 之间，并加上 32
                if (j < 8) {
                    y[i].scales[j] = l & 0xF;
                } else {
                    y[i].scales[j-8] |= ((l & 0xF) << 4);
                }
                // 根据条件将 l 的值存储到 y[i].scales 数组中
                l >>= 4;
                y[i].scales[j%4 + 8] |= (l << (2*(j/4)));
                // 将 l 的值存储到 y[i].scales 数组中
            }
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
            // 计算 y[i].d 的值
        } else {
            y[i].d = GGML_FP32_TO_FP16(0.f);
            // 如果 max_scale 不存在，则将 y[i].d 的值设置为 0
        }

        int8_t sc;
        // 声明 int8_t 类型的变量 sc
        for (int j = 0; j < QK_K/16; ++j) {
            // 遍历循环，j 从 0 到 QK_K/16
            sc = j < 8 ? y[i].scales[j] & 0xF : y[i].scales[j-8] >> 4;
            // 根据条件给 sc 赋值
            sc = (sc | (((y[i].scales[8 + j%4] >> (2*(j/4))) & 3) << 4)) - 32;
            // 根据条件计算 sc 的值
            float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            // 计算 d 的值
            if (!d) {
                // 如果 d 为 0，则跳过当前循环
                continue;
            }
            for (int ii = 0; ii < 16; ++ii) {
                // 遍历循环，ii 从 0 到 15
                int l = nearest_int(x[16*j + ii]/d);
                // 计算 l 的值
                l = MAX(-4, MIN(3, l));
                // 将 l 的值限制在 -4 到 3 之间
                L[16*j + ii] = l + 4;
                // 将 l 的值加上 4 存储到 L 数组中
            }
        }
#else
        // 如果 max_scale 为真，则执行以下代码块
        if (max_scale) {
            // 计算缩放比例
            float iscale = -8.f/max_scale;
            // 遍历 QK_K/16 次，每次增加2
            for (int j = 0; j < QK_K/16; j+=2) {
                // 计算缩放值并限制在 -8 到 7 之间
                int l1 = nearest_int(iscale*scales[j]);
                l1 = 8 + MAX(-8, MIN(7, l1));
                int l2 = nearest_int(iscale*scales[j+1]);
                l2 = 8 + MAX(-8, MIN(7, l2));
                // 将 l1 和 l2 组合成一个 8 位的值，存入 y[i].scales 数组
                y[i].scales[j/2] = l1 | (l2 << 4);
            }
            // 将 1/iscale 转换成 FP16 格式，存入 y[i].d
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
        } else {
            // 如果 max_scale 为假，则执行以下代码块
            for (int j = 0; j < QK_K/16; j+=2) {
                // 将 y[i].scales 数组中的值全部设为 0
                y[i].scales[j/2] = 0;
            }
            // 将 0.f 转换成 FP16 格式，存入 y[i].d
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        // 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 根据条件选择 y[i].scales[j/2] 中的低 4 位或高 4 位，并进行计算
            int s = j%2 == 0 ? y[i].scales[j/2] & 0xF : y[i].scales[j/2] >> 4;
            float d = GGML_FP16_TO_FP32(y[i].d) * (s - 8);
            // 如果 d 为 0，则跳过当前循环
            if (!d) {
                continue;
            }
            // 遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                // 根据计算结果将 x 中的值除以 d，并限制在 -4 到 3 之间，存入 L 数组
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-4, MIN(3, l));
                L[16*j + ii] = l + 4;
            }
        }
#endif

        // 将 y[i].hmask 数组中的值全部设为 0
        memset(y[i].hmask, 0, QK_K/8);
        // 设置高位掩码
        int m = 0;
        uint8_t hm = 1;
        // 遍历 QK_K 次
        for (int j = 0; j < QK_K; ++j) {
            // 如果 L[j] 大于 3，则将对应的 y[i].hmask[m] 的位设置为 1，并将 L[j] 减去 4
            if (L[j] > 3) {
                y[i].hmask[m] |= hm;
                L[j] -= 4;
            }
            // 更新 m 和 hm 的值
            if (++m == QK_K/8) {
                m = 0; hm <<= 1;
            }
        }
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        for (int j = 0; j < QK_K; j += 128) {
            // 将 L 中的值按照规则组合成 32 个 8 位的值，存入 y[i].qs 数组
            for (int l = 0; l < 32; ++l) {
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
            }
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 将 L 中的值按照规则组合成 16 个 8 位的值，存入 y[i].qs 数组
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
        }
#endif

        // 更新 x 的指针位置
        x += QK_K;
    }
}

#if QK_K == 256
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    // 定义掩码常量
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 定义辅助数组
    uint32_t aux[4];
    // 将辅助数组转换为 int8_t 类型的指针
    const int8_t * scales = (const int8_t*)aux;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {

        // 将 FP16 类型的数据转换为 FP32 类型
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 获取指向量化后数据和掩码的指针
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        uint8_t m = 1;

        // 将量化因子转换为 int8_t 类型的数组
        memcpy(aux, x[i].scales, 12);
        uint32_t tmp = aux[2];
        // 重新排列量化因子的字节顺序
        aux[2] = ((aux[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        aux[3] = ((aux[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        aux[0] = (aux[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        aux[1] = (aux[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);

        int is = 0;
        float dl;
        // 遍历每个块中的量化数据
        for (int n = 0; n < QK_K; n += 128) {
            int shift = 0;
            // 遍历每个量化因子
            for (int j = 0; j < 4; ++j) {

                // 计算量化后的数据
                dl = d_all * (scales[is++] - 32);
                for (int l = 0; l < 16; ++l) {
                    // 计算量化后的数据并存储到输出数组中
                    *y++ = dl * ((int8_t)((q[l+ 0] >> shift) & 3) - ((hm[l+ 0] & m) ? 0 : 4));
                }

                // 计算量化后的数据
                dl = d_all * (scales[is++] - 32);
                for (int l = 0; l < 16; ++l) {
                    // 计算量化后的数据并存储到输出数组中
                    *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3) - ((hm[l+16] & m) ? 0 : 4));
                }

                shift += 2;
                m <<= 1;
            }
            q += 32;
        }

    }
}
#else
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 确保 QK_K 的值为 64
    assert(QK_K == 64);
    // 计算块的数量
    const int nb = k / QK_K;
    // 遍历 nb 次，nb 为循环次数
    for (int i = 0; i < nb; i++) {

        // 将 x[i] 的 d 属性从 FP16 转换为 FP32，并赋值给 d_all
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 限定指针 q 指向 x[i] 的 qs 属性，限定指针 hm 指向 x[i] 的 hmask 属性
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;

        // 根据 x[i] 的 scales 属性计算 d1、d2、d3、d4
        const float d1 = d_all * ((x[i].scales[0] & 0xF) - 8);
        const float d2 = d_all * ((x[i].scales[0] >>  4) - 8);
        const float d3 = d_all * ((x[i].scales[1] & 0xF) - 8);
        const float d4 = d_all * ((x[i].scales[1] >>  4) - 8);

        // 遍历 8 次，计算 y 数组的值
        for (int l=0; l<8; ++l) {
            uint8_t h = hm[l];
            y[l+ 0] = d1 * ((int8_t)((q[l+0] >> 0) & 3) - ((h & 0x01) ? 0 : 4));
            y[l+ 8] = d1 * ((int8_t)((q[l+8] >> 0) & 3) - ((h & 0x02) ? 0 : 4));
            y[l+16] = d2 * ((int8_t)((q[l+0] >> 2) & 3) - ((h & 0x04) ? 0 : 4));
            y[l+24] = d2 * ((int8_t)((q[l+8] >> 2) & 3) - ((h & 0x08) ? 0 : 4));
            y[l+32] = d3 * ((int8_t)((q[l+0] >> 4) & 3) - ((h & 0x10) ? 0 : 4));
            y[l+40] = d3 * ((int8_t)((q[l+8] >> 4) & 3) - ((h & 0x20) ? 0 : 4));
            y[l+48] = d4 * ((int8_t)((q[l+0] >> 6) & 3) - ((h & 0x40) ? 0 : 4));
            y[l+56] = d4 * ((int8_t)((q[l+8] >> 6) & 3) - ((h & 0x80) ? 0 : 4));
        }
        // 将 y 指针向后移动 QK_K 个位置
        y += QK_K;
    }
// 结束条件判断，检查是否定义了宏
}
#endif

// 定义函数 quantize_row_q3_K，接受一个浮点型数组和一个指针作为参数
void quantize_row_q3_K(const float * restrict x, void * restrict vy, int k) {
    // 调用 quantize_row_q3_K_reference 函数
    quantize_row_q3_K_reference(x, vy, k);
}

// 定义函数 ggml_quantize_q3_K，接受一个浮点型数组、一个指针、一个整数和三个整型指针作为参数
size_t ggml_quantize_q3_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 忽略 hist 参数，待实现：收集直方图
    (void)hist;

    // 循环，每次增加 k 的步长
    for (int j = 0; j < n; j += k) {
        // 将 dst 转换为 block_q3_K 指针，并加上 j/QK_K 的偏移量
        block_q3_K * restrict y = (block_q3_K *)dst + j/QK_K;
        // 调用 quantize_row_q3_K_reference 函数
        quantize_row_q3_K_reference(src + j, y, k);
    }
    // 返回值为 n/QK_K 乘以 block_q3_K 的大小
    return (n/QK_K*sizeof(block_q3_K));
}

// 定义函数 quantize_row_q4_K_reference，接受一个浮点型数组、一个 block_q4_K 指针和一个整数作为参数
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k) {
    // 断言，确保 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;

    // 声明并初始化一些数组和变量
    uint8_t L[QK_K];
    uint8_t Laux[32];
    float   weights[32];
    float mins[QK_K/32];
    float scales[QK_K/32];

    // 循环，每次增加 1 的步长
    for (int i = 0; i < nb; i++) {
        // 声明并初始化一些变量
        float max_scale = 0;
        float max_min = 0;
        // 循环，每次增加 1 的步长
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算 sum_x2 的值
            float sum_x2 = 0;
            // 循环，每次增加 1 的步长
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l];
            // 计算 av_x 的值
            float av_x = sqrtf(sum_x2/32);
            // 循环，每次增加 1 的步长
            for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]);
            // 调用 make_qkx2_quants 函数，计算 scales[j] 的值
            scales[j] = make_qkx2_quants(32, 15, x + 32*j, weights, L + 32*j, &mins[j], Laux, -1.f, 0.1f, 20, false);
            // 声明并初始化 scale 的值
            float scale = scales[j];
            // 判断 scale 是否大于 max_scale，更新 max_scale 的值
            if (scale > max_scale) {
                max_scale = scale;
            }
            // 声明并初始化 min 的值
            float min = mins[j];
            // 判断 min 是否大于 max_min，更新 max_min 的值
            if (min > max_min) {
                max_min = min;
            }
        }
#if QK_K == 256
        // 如果 QK_K 等于 256，则计算最大比例和最小比例的倒数
        float inv_scale = max_scale > 0 ? 63.f/max_scale : 0.f;
        float inv_min   = max_min   > 0 ? 63.f/max_min   : 0.f;
        // 遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算最接近的整数值，并限制在 0 到 63 之间
            uint8_t ls = nearest_int(inv_scale*scales[j]);
            uint8_t lm = nearest_int(inv_min*mins[j]);
            ls = MIN(63, ls);
            lm = MIN(63, lm);
            // 根据索引 j 的值进行条件判断
            if (j < 4) {
                y[i].scales[j] = ls;
                y[i].scales[j+4] = lm;
            } else {
                y[i].scales[j+4] = (ls & 0xF) | ((lm & 0xF) << 4);
                y[i].scales[j-4] |= ((ls >> 4) << 6);
                y[i].scales[j-0] |= ((lm >> 4) << 6);
            }
        }
        // 将最大比例和最小比例转换为 FP16 格式
        y[i].d = GGML_FP32_TO_FP16(max_scale/63.f);
        y[i].dmin = GGML_FP32_TO_FP16(max_min/63.f);

        uint8_t sc, m;
        // 再次遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 调用函数获取比例和最小值
            get_scale_min_k4(j, y[i].scales, &sc, &m);
            const float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            // 如果 d 为 0，则跳过当前循环
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * m;
            // 再次遍历 32 次
            for (int ii = 0; ii < 32; ++ii) {
                // 计算最接近的整数值，并限制在 0 到 15 之间
                int l = nearest_int((x[32*j + ii] + dm)/d);
                l = MAX(0, MIN(15, l));
                L[32*j + ii] = l;
            }
        }
        // 计算缩放因子
        const float s_factor = 15.f;
        // 计算逆缩放比例
        float inv_scale = max_scale > 0 ? s_factor/max_scale : 0.f;
        // 计算逆最小值比例
        float inv_min   = max_min   > 0 ? s_factor/max_min   : 0.f;
        // 计算第一个维度的值
        int d1 = nearest_int(inv_scale*scales[0]);
        // 计算第一个维度的最小值
        int m1 = nearest_int(inv_min*mins[0]);
        // 计算第二个维度的值
        int d2 = nearest_int(inv_scale*scales[1]);
        // 计算第二个维度的最小值
        int m2 = nearest_int(inv_min*mins[1]);
        // 将计算得到的值和最小值存入y[i]的scales数组中
        y[i].scales[0] = d1 | (m1 << 4);
        y[i].scales[1] = d2 | (m2 << 4);
        // 将最大值和缩放因子存入y[i]的d数组中
        y[i].d[0] = GGML_FP32_TO_FP16(max_scale/s_factor);
        y[i].d[1] = GGML_FP32_TO_FP16(max_min/s_factor);

        // 初始化sumlx和suml2
        float sumlx = 0;
        int   suml2 = 0;
        // 遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取scales数组中的值和最小值
            const uint8_t sd = y[i].scales[j] & 0xF;
            const uint8_t sm = y[i].scales[j] >>  4;
            // 计算d和m的值
            const float d = GGML_FP16_TO_FP32(y[i].d[0]) * sd;
            if (!d) continue;
            const float m = GGML_FP16_TO_FP32(y[i].d[1]) * sm;
            // 遍历32次
            for (int ii = 0; ii < 32; ++ii) {
                // 计算l的值
                int l = nearest_int((x[32*j + ii] + m)/d);
                l = MAX(0, MIN(15, l));
                // 将l的值存入L数组中
                L[32*j + ii] = l;
                // 更新sumlx和suml2的值
                sumlx += (x[32*j + ii] + m)*l*sd;
                suml2 += l*l*sd*sd;
            }
        }
        // 如果suml2不为0，则更新y[i]的d[0]值
        if (suml2) {
            y[i].d[0] = GGML_FP32_TO_FP16(sumlx/suml2);
        }

        // 将L数组中的值存入y[i]的qs数组中
        uint8_t * q = y[i].qs;
        for (int j = 0; j < QK_K; j += 64) {
            for (int l = 0; l < 32; ++l) q[l] = L[j + l] | (L[j + l + 32] << 4);
            q += 32;
        }

        // 更新x的值
        x += QK_K;
    }
}

// 定义dequantize_row_q4_K函数
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k) {
    // 断言k能被QK_K整除
    assert(k % QK_K == 0);
    // 计算nb的值
    const int nb = k / QK_K;

    // 遍历nb次
    for (int i = 0; i < nb; i++) {
        // 获取x[i]的qs数组的值
        const uint8_t * q = x[i].qs;
#if QK_K == 256
    // 如果 QK_K 等于 256，则执行以下代码块

    const float d   = GGML_FP16_TO_FP32(x[i].d);
    // 将 x[i].d 转换为 32 位浮点数赋值给 d
    const float min = GGML_FP16_TO_FP32(x[i].dmin);
    // 将 x[i].dmin 转换为 32 位浮点数赋值给 min

    int is = 0;
    // 初始化 is 为 0
    uint8_t sc, m;
    // 声明 uint8_t 类型的变量 sc 和 m
    for (int j = 0; j < QK_K; j += 64) {
        // 循环，j 从 0 开始，每次增加 64，直到 j 小于 QK_K
        get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
        // 调用 get_scale_min_k4 函数，将结果赋值给 sc 和 m
        const float d1 = d * sc; const float m1 = min * m;
        // 计算 d1 和 m1 的值
        get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
        // 调用 get_scale_min_k4 函数，将结果赋值给 sc 和 m
        const float d2 = d * sc; const float m2 = min * m;
        // 计算 d2 和 m2 的值
        for (int l = 0; l < 32; ++l) *y++ = d1 * (q[l] & 0xF) - m1;
        // 循环，计算并赋值给 y
        for (int l = 0; l < 32; ++l) *y++ = d2 * (q[l]  >> 4) - m2;
        // 循环，计算并赋值给 y
        q += 32; is += 2;
        // q 增加 32，is 增加 2
    }
#else
    // 如果 QK_K 不等于 256，则执行以下代码块

    const float dall = GGML_FP16_TO_FP32(x[i].d[0]);
    // 将 x[i].d[0] 转换为 32 位浮点数赋值给 dall
    const float mall = GGML_FP16_TO_FP32(x[i].d[1]);
    // 将 x[i].d[1] 转换为 32 位浮点数赋值给 mall
    const float d1 = dall * (x[i].scales[0] & 0xF), m1 = mall * (x[i].scales[0] >> 4);
    // 计算 d1 和 m1 的值
    const float d2 = dall * (x[i].scales[1] & 0xF), m2 = mall * (x[i].scales[1] >> 4);
    // 计算 d2 和 m2 的值
    for (int l = 0; l < 32; ++l) {
        y[l+ 0] = d1 * (q[l] & 0xF) - m1;
        // 计算并赋值给 y
        y[l+32] = d2 * (q[l] >>  4) - m2;
        // 计算并赋值给 y
    }
    y += QK_K;
    // y 增加 QK_K
#endif
    // 结束条件编译指令块

}

void quantize_row_q4_K(const float * restrict x, void * restrict vy, int k) {
    // 定义函数 quantize_row_q4_K，参数为 x、vy 和 k
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    block_q4_K * restrict y = vy;
    // 声明并初始化 block_q4_K 类型的指针 y，指向 vy
    quantize_row_q4_K_reference(x, y, k);
    // 调用 quantize_row_q4_K_reference 函数
}

size_t ggml_quantize_q4_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 定义函数 ggml_quantize_q4_K，参数为 src、dst、n、k 和 hist
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    (void)hist; // TODO: collect histograms
    // 忽略 hist 参数，待实现：收集直方图

    for (int j = 0; j < n; j += k) {
        // 循环，j 从 0 开始，每次增加 k，直到 j 小于 n
        block_q4_K * restrict y = (block_q4_K *)dst + j/QK_K;
        // 声明并初始化 block_q4_K 类型的指针 y，指向 dst + j/QK_K
        quantize_row_q4_K_reference(src + j, y, k);
        // 调用 quantize_row_q4_K_reference 函数
    }
    return (n/QK_K*sizeof(block_q4_K));
    // 返回结果
}

// ====================== 5-bit (de)-quantization

void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k) {
    // 定义函数 quantize_row_q5_K_reference，参数为 x、y 和 k
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    const int nb = k / QK_K;
    // 计算 nb 的值

#if QK_K == 256
    // 如果 QK_K 等于 256，则执行以下代码块
    uint8_t L[QK_K];
    // 声明长度为 QK_K 的 uint8_t 类型数组 L
    float mins[QK_K/32];
    // 声明长度为 QK_K/32 的 float 类型数组 mins
    float scales[QK_K/32];
    // 声明长度为 QK_K/32 的 float 类型数组 scales
    # 定义一个包含32个浮点数的数组
    float weights[32];
    # 定义一个包含32个8位无符号整数的数组
    uint8_t Laux[32];
#else
    // 定义长度为 QK_K 的 int8_t 数组 L
    int8_t L[QK_K];
    // 定义长度为 QK_K/16 的 float 数组 scales
    float scales[QK_K/16];
#endif

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

#else
        // 定义变量 max_scale 和 amax，并初始化为 0
        float max_scale = 0, amax = 0;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 scales[j] 的值
            scales[j] = make_qx_quants(16, 16, x + 16*j, L + 16*j, 1);
            // 计算 scales[j] 的绝对值
            float abs_scale = fabsf(scales[j]);
            // 如果 abs_scale 大于 amax，则更新 amax 和 max_scale 的值
            if (abs_scale > amax) {
                amax = abs_scale;
                max_scale = scales[j];
            }
        }

        // 计算 iscale 的值
        float iscale = -128.f/max_scale;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 y[i].scales[j] 的值
            int l = nearest_int(iscale*scales[j]);
            // 将计算结果限制在 -128 和 127 之间，并赋值给 y[i].scales[j]
            y[i].scales[j] = MAX(-128, MIN(127, l));
        }
        // 计算 y[i].d 的值
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 d 的值
            const float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            // 如果 d 为 0，则跳过当前循环
            if (!d) continue;
            // 循环遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算 l 的值
                int l = nearest_int(x[16*j + ii]/d);
                // 将 l 限制在 -16 和 15 之间，并赋值给 L[16*j + ii]
                l = MAX(-16, MIN(15, l));
                L[16*j + ii] = l + 16;
            }
        }

        // 定义指向 y[i].qh 和 y[i].qs 的指针，并将其初始化为 0
        uint8_t * restrict qh = y[i].qh;
        uint8_t * restrict ql = y[i].qs;
        memset(qh, 0, QK_K/8);

        // 循环遍历 32 次
        for (int j = 0; j < 32; ++j) {
            // 计算 jm 和 is 的值
            int jm = j%8;
            int is = j/8;
            // 获取 L[j] 的值，并根据条件进行处理
            int l1 = L[j];
            if (l1 > 15) {
                l1 -= 16; qh[jm] |= (1 << is);
            }
            // 获取 L[j + 32] 的值，并根据条件进行处理
            int l2 = L[j + 32];
            if (l2 > 15) {
                l2 -= 16; qh[jm] |= (1 << (4 + is));
            }
            // 将 l1 和 l2 组合成 ql[j] 的值
            ql[j] = l1 | (l2 << 4);
        }
#endif

        // 更新 x 的值
        x += QK_K;

    }
}

// 定义函数 dequantize_row_q5_K，参数为 block_q5_K 类型的指针 x，float 类型的指针 y，int 类型的 k
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k) {
    // 断言 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 定义指向 x[i].qs 和 x[i].qh 的指针 ql 和 qh
        const uint8_t * ql = x[i].qs;
        const uint8_t * qh = x[i].qh;
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块

        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 32 位浮点数，并赋给变量 d
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].dmin 转换为 32 位浮点数，并赋给变量 min

        int is = 0;
        // 初始化变量 is 为 0
        uint8_t sc, m;
        // 声明无符号 8 位整数变量 sc 和 m
        uint8_t u1 = 1, u2 = 2;
        // 声明无符号 8 位整数变量 u1 和 u2，并初始化为 1 和 2
        for (int j = 0; j < QK_K; j += 64) {
            // 循环，j 从 0 开始，每次增加 64，直到 j 小于 QK_K
            get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
            // 调用函数 get_scale_min_k4，传入参数 is + 0, x[i].scales, &sc, &m
            const float d1 = d * sc; const float m1 = min * m;
            // 计算 d1 和 m1 的值
            get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
            // 调用函数 get_scale_min_k4，传入参数 is + 1, x[i].scales, &sc, &m
            const float d2 = d * sc; const float m2 = min * m;
            // 计算 d2 和 m2 的值
            for (int l = 0; l < 32; ++l) *y++ = d1 * ((ql[l] & 0xF) + (qh[l] & u1 ? 16 : 0)) - m1;
            // 循环，l 从 0 到 31，对 y 进行赋值操作
            for (int l = 0; l < 32; ++l) *y++ = d2 * ((ql[l]  >> 4) + (qh[l] & u2 ? 16 : 0)) - m2;
            // 循环，l 从 0 到 31，对 y 进行赋值操作
            ql += 32; is += 2;
            // ql 增加 32，is 增加 2
            u1 <<= 2; u2 <<= 2;
            // u1 左移 2 位，u2 左移 2 位
        }
#else
// 如果 QK_K 不等于 256，则执行以下代码块
        float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 32 位浮点数，并赋给变量 d
        const int8_t * restrict s = x[i].scales;
        // 声明指向常量 int8_t 类型的指针 s，指向 x[i].scales
        for (int l = 0; l < 8; ++l) {
            // 循环，l 从 0 到 7
            y[l+ 0] = d * s[0] * ((ql[l+ 0] & 0xF) - (qh[l] & 0x01 ? 0 : 16));
            // 对 y[l+ 0] 进行赋值操作
            y[l+ 8] = d * s[0] * ((ql[l+ 8] & 0xF) - (qh[l] & 0x02 ? 0 : 16));
            // 对 y[l+ 8] 进行赋值操作
            y[l+16] = d * s[1] * ((ql[l+16] & 0xF) - (qh[l] & 0x04 ? 0 : 16));
            // 对 y[l+16] 进行赋值操作
            y[l+24] = d * s[1] * ((ql[l+24] & 0xF) - (qh[l] & 0x08 ? 0 : 16));
            // 对 y[l+24] 进行赋值操作
            y[l+32] = d * s[2] * ((ql[l+ 0] >>  4) - (qh[l] & 0x10 ? 0 : 16));
            // 对 y[l+32] 进行赋值操作
            y[l+40] = d * s[2] * ((ql[l+ 8] >>  4) - (qh[l] & 0x20 ? 0 : 16));
            // 对 y[l+40] 进行赋值操作
            y[l+48] = d * s[3] * ((ql[l+16] >>  4) - (qh[l] & 0x40 ? 0 : 16));
            // 对 y[l+48] 进行赋值操作
            y[l+56] = d * s[3] * ((ql[l+24] >>  4) - (qh[l] & 0x80 ? 0 : 16));
            // 对 y[l+56] 进行赋值操作
        }
        y += QK_K;
        // y 增加 QK_K
#endif
    }
}

void quantize_row_q5_K(const float * restrict x, void * restrict vy, int k) {
    // 定义函数 quantize_row_q5_K，传入参数 x, vy, k
    assert(k % QK_K == 0);
    // 断言 k 能被 QK_K 整除
    block_q5_K * restrict y = vy;
    // 声明指向 block_q5_K 类型的指针 y，指向 vy
    quantize_row_q5_K_reference(x, y, k);
    // 调用函数 quantize_row_q5_K_reference，传入参数 x, y, k
}

size_t ggml_quantize_q5_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 定义函数 ggml_quantize_q5_K，传入参数 src, dst, n, k, hist
    assert(k % QK_K == 0);
    // 断言 k 能被 QK_K 整除
    (void)hist; // TODO: collect histograms
    // 忽略 hist 参数，待实现：收集直方图
    # 循环，每次增加 k 的步长，直到 j 大于等于 n
    for (int j = 0; j < n; j += k) {
        # 将目标地址转换为 block_q5_K 类型的指针，并偏移 j/QK_K 个单位
        block_q5_K * restrict y = (block_q5_K *)dst + j/QK_K;
        # 调用 quantize_row_q5_K_reference 函数，对源地址偏移 j 位置开始的数据进行量化，结果存储在 y 中
        quantize_row_q5_K_reference(src + j, y, k);
    }
    # 返回结果，计算公式为 n/QK_K 乘以 block_q5_K 类型的大小
    return (n/QK_K*sizeof(block_q5_K));
// ====================== 6-bit (de)-quantization

// 对输入的一行数据进行 6 位量化，结果存储在 block_q6_K 结构体中
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    int8_t L[QK_K];  // 存储量化后的数据
    float   scales[QK_K/16];  // 存储每个块的缩放因子

    for (int i = 0; i < nb; i++) {  // 遍历每个块

        float max_scale = 0;  // 最大缩放因子
        float max_abs_scale = 0;  // 最大绝对值的缩放因子

        for (int ib = 0; ib < QK_K/16; ++ib) {  // 遍历每个块中的子块

            // 对每个子块进行 16 位量化，并返回缩放因子
            const float scale = make_qx_quants(16, 32, x + 16*ib, L + 16*ib, 1);
            scales[ib] = scale;  // 存储缩放因子

            const float abs_scale = fabsf(scale);  // 计算缩放因子的绝对值
            if (abs_scale > max_abs_scale) {  // 更新最大绝对值的缩放因子和对应的缩放因子
                max_abs_scale = abs_scale;
                max_scale = scale;
            }

        }

        if (!max_abs_scale) {  // 如果最大绝对值的缩放因子为 0
            memset(&y[i], 0, sizeof(block_q6_K));  // 将 y[i] 清零
            y[i].d = GGML_FP32_TO_FP16(0.f);  // 设置 y[i].d 为 0
            x += QK_K;  // 移动输入数据指针
            continue;  // 继续下一次循环
        }

        float iscale = -128.f/max_scale;  // 计算缩放因子的倒数
        y[i].d = GGML_FP32_TO_FP16(1/iscale);  // 存储缩放因子的倒数
        for (int ib = 0; ib < QK_K/16; ++ib) {  // 遍历每个子块
            y[i].scales[ib] = MIN(127, nearest_int(iscale*scales[ib]));  // 计算并存储量化后的缩放因子
        }

        for (int j = 0; j < QK_K/16; ++j) {  // 遍历每个子块
            float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];  // 计算缩放后的值
            if (!d) {  // 如果 d 为 0
                continue;  // 继续下一次循环
            }
            for (int ii = 0; ii < 16; ++ii) {  // 遍历每个元素
                int l = nearest_int(x[16*j + ii]/d);  // 计算并存储量化后的值
                l = MAX(-32, MIN(31, l));  // 确保 l 在 -32 到 31 之间
                L[16*j + ii] = l + 32;  // 存储量化后的值
            }
        }

        uint8_t * restrict ql = y[i].ql;  // 低位量化后的值
        uint8_t * restrict qh = y[i].qh;  // 高位量化后的值
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块
        for (int j = 0; j < QK_K; j += 128) {
            // 循环遍历，每次增加 128
            for (int l = 0; l < 32; ++l) {
                // 循环遍历，每次增加 1
                const uint8_t q1 = L[j + l +  0] & 0xF;
                // 获取指定位置的值，并与 0xF 进行按位与操作
                const uint8_t q2 = L[j + l + 32] & 0xF;
                // 获取指定位置的值，并与 0xF 进行按位与操作
                const uint8_t q3 = L[j + l + 64] & 0xF;
                // 获取指定位置的值，并与 0xF 进行按位与操作
                const uint8_t q4 = L[j + l + 96] & 0xF;
                // 获取指定位置的值，并与 0xF 进行按位与操作
                ql[l+ 0] = q1 | (q3 << 4);
                // 将 q1 和 (q3 << 4) 进行按位或操作，并赋值给指定位置
                ql[l+32] = q2 | (q4 << 4);
                // 将 q2 和 (q4 << 4) 进行按位或操作，并赋值给指定位置
                qh[l] = (L[j + l] >> 4) | ((L[j + l + 32] >> 4) << 2) | ((L[j + l + 64] >> 4) << 4) | ((L[j + l + 96] >> 4) << 6);
                // 将指定位置的值右移 4 位，并与后续值进行按位或操作，并赋值给指定位置
            }
            ql += 64;
            // ql 指针向后移动 64 位
            qh += 32;
            // qh 指针向后移动 32 位
        }
#else
// 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 32; ++l) {
            // 循环遍历，每次增加 1
            const uint8_t q1 = L[l +  0] & 0xF;
            // 获取指定位置的值，并与 0xF 进行按位与操作
            const uint8_t q2 = L[l + 32] & 0xF;
            // 获取指定位置的值，并与 0xF 进行按位与操作
            ql[l] = q1 | (q2 << 4);
            // 将 q1 和 (q2 << 4) 进行按位或操作，并赋值给指定位置
        }
        for (int l = 0; l < 16; ++l) {
            // 循环遍历，每次增加 1
            qh[l] = (L[l] >> 4) | ((L[l + 16] >> 4) << 2) | ((L[l + 32] >> 4) << 4) | ((L[l + 48] >> 4) << 6);
            // 将指定位置的值右移 4 位，并与后续值进行按位或操作，并赋值给指定位置
        }
#endif
// 结束条件编译指令
        x += QK_K;
        // x 指针向后移动 QK_K 位
    }
}

void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k) {
    // 定义函数 dequantize_row_q6_K，传入参数 x, y, k
    assert(k % QK_K == 0);
    // 断言 k 除以 QK_K 的余数为 0
    const int nb = k / QK_K;
    // 定义变量 nb，赋值为 k 除以 QK_K 的结果

    for (int i = 0; i < nb; i++) {
        // 循环遍历，每次增加 1

        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 float 类型，并赋值给变量 d

        const uint8_t * restrict ql = x[i].ql;
        // 定义指针 ql，指向 x[i].ql
        const uint8_t * restrict qh = x[i].qh;
        // 定义指针 qh，指向 x[i].qh
        const int8_t  * restrict sc = x[i].scales;
        // 定义指针 sc，指向 x[i].scales
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块
        for (int n = 0; n < QK_K; n += 128) {
            // 循环，每次增加 128
            for (int l = 0; l < 32; ++l) {
                // 循环，每次增加 1
                int is = l/16;
                // 计算 is 的值
                const int8_t q1 = (int8_t)((ql[l +  0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
                // 计算 q1 的值
                const int8_t q2 = (int8_t)((ql[l + 32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
                // 计算 q2 的值
                const int8_t q3 = (int8_t)((ql[l +  0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32;
                // 计算 q3 的值
                const int8_t q4 = (int8_t)((ql[l + 32]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32;
                // 计算 q4 的值
                y[l +  0] = d * sc[is + 0] * q1;
                // 计算 y[l +  0] 的值
                y[l + 32] = d * sc[is + 2] * q2;
                // 计算 y[l + 32] 的值
                y[l + 64] = d * sc[is + 4] * q3;
                // 计算 y[l + 64] 的值
                y[l + 96] = d * sc[is + 6] * q4;
                // 计算 y[l + 96] 的值
            }
            y  += 128;
            // y 增加 128
            ql += 64;
            // ql 增加 64
            qh += 32;
            // qh 增加 32
            sc += 8;
            // sc 增加 8
        }
#else
// 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 循环，每次增加 1
            const int8_t q1 = (int8_t)((ql[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            // 计算 q1 的值
            const int8_t q2 = (int8_t)((ql[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            // 计算 q2 的值
            const int8_t q3 = (int8_t)((ql[l+ 0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            // 计算 q3 的值
            const int8_t q4 = (int8_t)((ql[l+16]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32;
            // 计算 q4 的值
            y[l+ 0] = d * sc[0] * q1;
            // 计算 y[l+ 0] 的值
            y[l+16] = d * sc[1] * q2;
            // 计算 y[l+16] 的值
            y[l+32] = d * sc[2] * q3;
            // 计算 y[l+32] 的值
            y[l+48] = d * sc[3] * q4;
            // 计算 y[l+48] 的值
        }
        y  += 64;
        // y 增加 64
#endif
// 结束条件判断
    }
}

void quantize_row_q6_K(const float * restrict x, void * restrict vy, int k) {
    // 限制 k 必须是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 将 vy 强制转换为 block_q6_K 类型，并赋值给 y
    block_q6_K * restrict y = vy;
    // 调用 quantize_row_q6_K_reference 函数
    quantize_row_q6_K_reference(x, y, k);
}

size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 限制 k 必须是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // TODO: 收集直方图
    (void)hist; // TODO: collect histograms

    for (int j = 0; j < n; j += k) {
        // 将 dst 强制转换为 block_q6_K 类型，并赋值给 y
        block_q6_K * restrict y = (block_q6_K *)dst + j/QK_K;
        // 调用 quantize_row_q6_K_reference 函数
        quantize_row_q6_K_reference(src + j, y, k);
    }
    // 返回值为 n/QK_K 乘以 block_q6_K 的大小
    return (n/QK_K*sizeof(block_q6_K));
}
//===================================== Q8_K ==============================================

// 对长度为 k 的一维数组进行量化，结果存储在 block_q8_K 结构体数组中
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算 block_q8_K 结构体数组的长度
    const int nb = k / QK_K;

    // 遍历 block_q8_K 结构体数组
    for (int i = 0; i < nb; i++) {

        // 初始化最大值和绝对值最大值
        float max = 0;
        float amax = 0;
        // 遍历每个 block_q8_K 结构体中的元素
        for (int j = 0; j < QK_K; ++j) {
            // 计算当前元素的绝对值
            float ax = fabsf(x[j]);
            // 如果当前元素的绝对值大于绝对值最大值，则更新最大值和绝对值最大值
            if (ax > amax) {
                amax = ax; max = x[j];
            }
        }
        // 如果绝对值最大值为 0，则将当前 block_q8_K 结构体的数据清零，并继续下一次循环
        if (!amax) {
            y[i].d = 0;
            memset(y[i].qs, 0, QK_K);
            x += QK_K;
            continue;
        }
        // 计算缩放因子
        const float iscale = -128.f/max;
        // 对每个元素进行量化
        for (int j = 0; j < QK_K; ++j) {
            int v = nearest_int(iscale*x[j]);
            y[i].qs[j] = MIN(127, v);
        }
        // 计算每 16 个元素的和
        for (int j = 0; j < QK_K/16; ++j) {
            int sum = 0;
            for (int ii = 0; ii < 16; ++ii) {
                sum += y[i].qs[j*16 + ii];
            }
            y[i].bsums[j] = sum;
        }
        // 存储缩放因子的倒数
        y[i].d = 1/iscale;
        x += QK_K;
    }
}

// 对 block_q8_K 结构体数组进行反量化，结果存储在长度为 k 的一维数组中
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算 block_q8_K 结构体数组的长度
    const int nb = k / QK_K;

    // 遍历 block_q8_K 结构体数组
    for (int i = 0; i < nb; i++) {
        // 对每个元素进行反量化
        for (int j = 0; j < QK_K; ++j) {
            *y++ = x[i].d * x[i].qs[j];
        }
    }
}

// 对长度为 k 的一维数组进行量化，结果存储在 block_q8_K 结构体数组中
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q8_K_reference 函数进行量化
    quantize_row_q8_K_reference(x, y, k);
}

//===================================== Dot ptoducts =================================

//
// Helper functions
//
#if __AVX__ || __AVX2__ || __AVX512F__

// shuffles to pick the required scales in dot products
// 用于在点积中选择所需的缩放比例的洗牌操作
static inline __m256i get_scale_shuffle_q3k(int i) {
    // 定义一个包含128个元素的常量数组，用于乱序加载数据
    static const uint8_t k_shuffle[128] = {
         0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,     2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
         4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5,     6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7,
         8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9,    10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,
        12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,    14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,
    };
    // 从常量数组中加载128位数据到寄存器中并返回
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
    // 定义静态内联函数，返回一个__m256i类型的值，参数为整数i
    static inline __m256i get_scale_shuffle_k4(int i) {
        // 定义静态常量数组k_shuffle，包含256个uint8_t类型的元素
        static const uint8_t k_shuffle[256] = {
             // 数组元素的数值
        };
        // 返回加载自k_shuffle数组中第i个元素的__m256i类型值
        return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
    }
    // 定义静态内联函数，返回一个__m128i类型的值，参数为整数i
    static inline __m128i get_scale_shuffle(int i) {
        // 定义静态常量数组k_shuffle，包含128个uint8_t类型的元素
        static const uint8_t k_shuffle[128] = {
             // 数组元素的数值
        };
        // 返回加载自k_shuffle数组中第i个元素的__m128i类型值
        return _mm_loadu_si128((const __m128i*)k_shuffle + i);
    }
#endif

// 定义函数ggml_axpy_q4_0_q8_0，参数为整数n，指向常量的指针vx、vy、vz，int8_t类型的alpha，ggml_fp16_t类型的scale
void ggml_axpy_q4_0_q8_0(const int n, const void * restrict vx, const void * restrict vy, const void * restrict vz, int8_t alpha, ggml_fp16_t scale) {
    // 定义整数qk，赋值为QK8_0
    const int qk = QK8_0;
    // 定义整数nb，赋值为n除以qk的商
    const int nb = n / qk;
    // 断言n除以qk的余数为0
    assert(n % qk == 0);
    // 断言nb除以2的余数为0
    assert(nb % 2 == 0);

    // 定义指向block_q4_0类型的指针x，指向vx
    const block_q4_0 * restrict x = vx;
#if defined(__AVX2__)
    // 初始化累加器为零
    # 初始化一个256位全零的寄存器acc
    __m256 acc = _mm256_setzero_ps();
    
    # 使用alpha值初始化一个256位整数寄存器alpha_v
    __m256i alpha_v = _mm256_set1_epi16((short)alpha;
    # 主循环
#else
    // 定义一个指向浮点数的指针 res，指向 vx
    float *res = (float *)vz;
    // 将 scale 转换为单精度浮点数
    float scale_fp32 = GGML_FP16_TO_FP32(scale);
    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 转换为单精度浮点数，乘以 scale_fp32
        float result_scale = GGML_FP16_TO_FP32(x[i].d) * scale_fp32;
        // 计算偏移量
        int offset = i * QK4_0;

        // 遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 获取 x[i].qs[j] 的低 4 位，减去 8
            const int v0 = (x[i].qs[j] & 0x0F) - 8;
            // 获取 x[i].qs[j] 的高 4 位，减去 8
            const int v1 = (x[i].qs[j] >>   4) - 8;
            // 更新 res[offset + j] 的值
            res[offset + j] = res[offset + j] + ((float)(v0 * (int)alpha) * result_scale);
            // 更新 res[offset + j + qk/2] 的值
            res[offset + j + qk/2] = res[offset + j + qk/2] + ((float)(v1 * (int)alpha) * result_scale);
        }
    }
#endif
}

// 定义函数 ggml_vec_dot_q4_0_q8_0，参数为 n, s, vx, vy
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk，值为 QK8_0
    const int qk = QK8_0;
    // 定义常量 nb，值为 n 除以 qk
    const int nb = n / qk;

    // 断言 n 除以 qk 的余数为 0
    assert(n % qk == 0);

    // 定义指向 block_q4_0 类型的指针 x，指向 vx
    const block_q4_0 * restrict x = vx;
    // 定义指向 block_q8_0 类型的指针 y，指向 vy

#if defined(__ARM_NEON)
    // 定义四个单精度浮点数向量，初始值为 0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 断言 nb 除以 2 的余数为 0
    assert(nb % 2 == 0); // TODO: handle odd nb
    // 循环遍历，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 定义并初始化指向 x 数组中元素的指针
        const block_q4_0 * restrict x0 = &x[i + 0];
        const block_q4_0 * restrict x1 = &x[i + 1];
        // 定义并初始化指向 y 数组中元素的指针
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个包含 0x0F 的 16 个元素的向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
        // 创建一个包含 0x8 的 16 个元素的向量
        const int8x16_t  s8b = vdupq_n_s8(0x8);

        // 从 x0 和 x1 中加载 16 个 8 位无符号整数到向量中
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        // 将 4 位整数转换为 8 位整数
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 减去 8
        const int8x16_t v0_0ls = vsubq_s8(v0_0l, s8b);
        const int8x16_t v0_0hs = vsubq_s8(v0_0h, s8b);
        const int8x16_t v0_1ls = vsubq_s8(v0_1l, s8b);
        const int8x16_t v0_1hs = vsubq_s8(v0_1h, s8b);

        // 从 y0 和 y1 中加载 16 个 8 位有符号整数到向量中
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM dot product 特性，则执行以下代码
    // 将两个 int32x4_t 向量进行点乘
    const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0ls, v1_0l), v0_0hs, v1_0h);
    const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1ls, v1_1l), v0_1hs, v1_1h);

    // 使用点乘结果更新 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d);
#else
    // 如果不支持 ARM dot product 特性，则执行以下代码
    // 将两个 int16x8_t 向量的低位和高位分别进行乘法运算
    const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0ls), vget_low_s8 (v1_0l));
    const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0ls), vget_high_s8(v1_0l));
    const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hs), vget_low_s8 (v1_0h));
    const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hs), vget_high_s8(v1_0h));

    const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1ls), vget_low_s8 (v1_1l));
    const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1ls), vget_high_s8(v1_1l));
    const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hs), vget_low_s8 (v1_1h));
    const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hs), vget_high_s8(v1_1h));

    // 将乘法运算结果进行累加得到 int32x4_t 向量
    const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
    const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
    const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
    const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

    // 使用累加结果更新 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d);
#endif
    }

    // 将 sumv0 和 sumv1 的结果相加，并存储到指定地址
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__)
    // 如果支持 AVX2 特性，则执行以下代码
    // 将 256 位的浮点向量初始化为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        /* 计算块的组合比例 */
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 现在我们有一个包含在 [ 0 .. 15 ] 区间的字节向量。将它们偏移至 [ -8 .. +7 ] 区间。
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );

        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 将 q 乘以比例并累加 */
        acc = _mm256_fmadd_ps( d, q, acc );
    }

    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 如果编译器定义了 AVX，则执行以下代码块

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 设置低位掩码
        const __m128i lowMask = _mm_set1_epi8(0xF);
        // 设置偏移量
        const __m128i off = _mm_set1_epi8(8);

        // 加载 x[i].qs 的值
        const __m128i tmp = _mm_loadu_si128((const __m128i *)x[i].qs);

        // 对 bx 和 by 进行位与运算
        __m128i bx = _mm_and_si128(lowMask, tmp);
        __m128i by = _mm_loadu_si128((const __m128i *)y[i].qs);
        bx = _mm_sub_epi8(bx, off);
        const __m128i i32_0 = mul_sum_i8_pairs(bx, by);

        // 对 bx 和 by 进行位与运算
        bx = _mm_and_si128(lowMask, _mm_srli_epi64(tmp, 4));
        by = _mm_loadu_si128((const __m128i *)(y[i].qs + 16));
        bx = _mm_sub_epi8(bx, off);
        const __m128i i32_1 = mul_sum_i8_pairs(bx, by);

        // 将 int32_t 转换为 float
        __m256 p = _mm256_cvtepi32_ps(MM256_SET_M128I(i32_0, i32_1));

        // 应用比例，并累加
        acc = _mm256_add_ps(_mm256_mul_ps( d, p ), acc);
    }

    *s = hsum_float_8(acc);
#elif defined(__SSSE3__)
    // 如果编译器定义了 SSSE3，则执行以下代码块

    // 设置常量
    const __m128i lowMask = _mm_set1_epi8(0xF);
    const __m128i off = _mm_set1_epi8(8);

    // 使用零初始化累加器
    __m128 acc_0 = _mm_setzero_ps();
    __m128 acc_1 = _mm_setzero_ps();
    __m128 acc_2 = _mm_setzero_ps();
    __m128 acc_3 = _mm_setzero_ps();

    // 第一轮不累加
    }

    assert(nb % 2 == 0); // TODO: 处理奇数 nb

    // 主循环
    }

    *s = hsum_float_4x4(acc_0, acc_1, acc_2, acc_3);
#elif defined(__riscv_v_intrinsic)
    // 如果编译器定义了 riscv_v_intrinsic，则执行以下代码块

    // 初始化浮点数和为 0.0
    float sumf = 0.0;

    // 设置 vl 的大小
    size_t vl = __riscv_vsetvl_e8m1(qk/2);
    for (int i = 0; i < nb; i++) {
        // 加载元素
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 掩码和存储 x 的低部分，然后是高部分
        vuint8mf2_t x_a = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        vuint8mf2_t x_l = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        vint8mf2_t x_ai = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        vint8mf2_t x_li = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        // 减去偏移量
        vint8mf2_t v0 = __riscv_vsub_vx_i8mf2(x_ai, 8, vl);
        vint8mf2_t v1 = __riscv_vsub_vx_i8mf2(x_li, 8, vl);

        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    *s = sumf;
#else
    // 如果不是向量化运算，则使用标量计算
    float sumf = 0.0; // 初始化浮点数求和变量

    for (int i = 0; i < nb; i++) { // 遍历每个块
        int sumi = 0; // 初始化整数求和变量

        for (int j = 0; j < qk/2; ++j) { // 遍历每个块中的元素
            const int v0 = (x[i].qs[j] & 0x0F) - 8; // 计算第一个元素的值
            const int v1 = (x[i].qs[j] >>   4) - 8; // 计算第二个元素的值

            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]); // 更新求和变量
        }

        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d); // 更新最终的浮点数求和变量
    }

    *s = sumf; // 将最终的求和结果赋值给输出变量
#endif
}

void ggml_vec_dot_q4_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_1; // 定义常量 qk
    const int nb = n / qk; // 计算块的数量

    assert(n % qk == 0); // 断言 n 能够整除 qk

    const block_q4_1 * restrict x = vx; // 定义指向输入向量 x 的指针
    const block_q8_1 * restrict y = vy; // 定义指向输入向量 y 的指针

    // TODO: add WASM SIMD
#if defined(__ARM_NEON)
    float32x4_t sumv0 = vdupq_n_f32(0.0f); // 初始化 NEON 浮点数求和变量
    float32x4_t sumv1 = vdupq_n_f32(0.0f); // 初始化 NEON 浮点数求和变量

    float summs = 0; // 初始化浮点数求和变量

    assert(nb % 2 == 0); // 断言块的数量是偶数

    for (int i = 0; i < nb; i += 2) { // 每次处理两个块
        const block_q4_1 * restrict x0 = &x[i + 0]; // 定义指向第一个块的指针
        const block_q4_1 * restrict x1 = &x[i + 1]; // 定义指向第二个块的指针
        const block_q8_1 * restrict y0 = &y[i + 0]; // 定义指向第一个块的指针
        const block_q8_1 * restrict y1 = &y[i + 1]; // 定义指向第二个块的指针

        summs += GGML_FP16_TO_FP32(x0->m) * y0->s + GGML_FP16_TO_FP32(x1->m) * y1->s; // 更新浮点数求和变量

        const uint8x16_t m4b = vdupq_n_u8(0x0F); // 初始化 NEON 8 位整数向量

        const uint8x16_t v0_0 = vld1q_u8(x0->qs); // 加载第一个块的数据
        const uint8x16_t v0_1 = vld1q_u8(x1->qs); // 加载第二个块的数据

        // 4-bit -> 8-bit
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b)); // 转换为 8 位整数向量
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4)); // 转换为 8 位整数向量
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b)); // 转换为 8 位整数向量
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4)); // 转换为 8 位整数向量

        // load y
        const int8x16_t v1_0l = vld1q_s8(y0->qs); // 加载第一个块的数据
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16); // 加载第一个块的数据
        const int8x16_t v1_1l = vld1q_s8(y1->qs); // 加载第二个块的数据
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16); // 加载第二个块的数据
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM dot product 特性，则执行以下代码
    // 将两个 int32x4_t 向量进行点积运算
    const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0l, v1_0l), v0_0h, v1_0h);
    const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1l, v1_1l), v0_1h, v1_1h);

    // 使用点积结果和浮点数进行乘法和加法运算
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*y0->d);
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
    // 如果不支持 ARM dot product 特性，则执行以下代码
    // 将两个 int16x8_t 向量进行乘法运算
    const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0l), vget_low_s8 (v1_0l));
    const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0l), vget_high_s8(v1_0l));
    const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0h), vget_low_s8 (v1_0h));
    const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0h), vget_high_s8(v1_0h));

    const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1l), vget_low_s8 (v1_1l));
    const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1l), vget_high_s8(v1_1l));
    const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1h), vget_low_s8 (v1_1h));
    const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1h), vget_high_s8(v1_1h));

    // 将 int16x8_t 向量转换为 int32x4_t 向量，并进行加法运算
    const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
    const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
    const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
    const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

    // 使用转换后的 int32x4_t 向量和浮点数进行乘法和加法运算
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
    }

    // 将两个 float 向量的元素相加，并加上 summs 的值
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs;
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果支持 AVX2 或 AVX 特性，则执行以下代码
    // 将 256 位的浮点数向量初始化为零
    __m256 acc = _mm256_setzero_ps();

    // 初始化 summs 为 0
    float summs = 0;

    // 主循环
    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {
        // 将x[i]的d值从FP16转换为FP32
        const float d0 = GGML_FP16_TO_FP32(x[i].d);
        // 获取y[i]的d值
        const float d1 = y[i].d;

        // 计算summs的累加值
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将d0的值转换为256位的向量
        const __m256 d0v = _mm256_set1_ps( d0 );
        // 将d1的值转换为256位的向量
        const __m256 d1v = _mm256_set1_ps( d1 );

        // 计算d0v和d1v的乘积
        const __m256 d0d1 = _mm256_mul_ps( d0v, d1v );

        // 从x[i]的qs中加载16个字节，并将4位字段解压缩成字节，生成32个字节
        const __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从y[i]的qs中加载16个字节
        const __m256i by = _mm256_loadu_si256( (const __m256i *)y[i].qs );

        // 计算bx和by的乘积，并累加
        const __m256 xy = mul_sum_us8_pairs_float(bx, by);

        // 累加d0*d1*x*y
#if defined(__AVX2__)
        // 如果支持 AVX2 指令集，则使用 FMA 指令进行乘加运算
        acc = _mm256_fmadd_ps( d0d1, xy, acc );
#else
        // 如果不支持 AVX2 指令集，则使用乘法和加法指令进行运算
        acc = _mm256_add_ps( _mm256_mul_ps( d0d1, xy ), acc );
#endif
    }

    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 如果支持 RISC-V 向量指令集
    float sumf = 0.0;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 掩码和存储 x 的低部分，然后是高部分
        vuint8mf2_t x_a = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        vuint8mf2_t x_l = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    *s = sumf;
#else
    // 如果不支持向量指令集，则使用标量运算
    float sumf = 0.0;

    for (int i = 0; i < nb; i++) {
        int sumi = 0;

        for (int j = 0; j < qk/2; ++j) {
            const int v0 = (x[i].qs[j] & 0x0F);
            const int v1 = (x[i].qs[j] >>   4);

            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]);
        }

        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    *s = sumf;
#endif
}

void ggml_vec_dot_q5_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk 和 nb
    const int qk = QK8_0;
    const int nb = n / qk;
    # 断言 n 能被 qk 整除
    assert(n % qk == 0);
    # 断言 qk 的值等于 QK5_0
    assert(qk == QK5_0);
    
    # 声明并初始化指向 vx 的指针 x，指向的数据类型为 block_q5_0
    const block_q5_0 * restrict x = vx;
    # 声明并初始化指向 vy 的指针 y，指向的数据类型为 block_q8_0
    const block_q8_0 * restrict y = vy;
# 如果定义了 ARM NEON，则执行以下操作
    # 创建一个四个元素都是 0.0f 的 float32x4_t 类型的向量 sumv0
    float32x4_t sumv0 = vdupq_n_f32(0.0f)
    # 创建一个四个元素都是 0.0f 的 float32x4_t 类型的向量 sumv1
    float32x4_t sumv1 = vdupq_n_f32(0.0f)

    # 创建两个 32 位无符号整数变量 qh0 和 qh1
    uint32_t qh0
    uint32_t qh1

    # 创建两个长度为 4 的 64 位无符号整数数组 tmp0 和 tmp1
    uint64_t tmp0[4]
    uint64_t tmp1[4]

    # 断言 nb 为偶数，如果不是偶数则抛出异常
    assert(nb % 2 == 0) # TODO: 处理奇数 nb

    # 如果定义了 ARM DOTPROD 特性，则执行以下操作
        # 计算 sumv0 的值
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d))
        # 计算 sumv1 的值
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d))
#else
        // 计算两个 int8x16_t 向量的低位元素的乘积
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        // 计算两个 int8x16_t 向量的高位元素的乘积
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        // 计算两个 int8x16_t 向量的低位元素的乘积
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        // 计算两个 int8x16_t 向量的高位元素的乘积
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        // 计算两个 int8x16_t 向量的低位元素的乗积
        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        // 计算两个 int8x16_t 向量的高位元素的乗积
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        // 计算两个 int8x16_t 向量的低位元素的乗积
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        // 计算两个 int8x16_t 向量的高位元素的乗积
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));

        // 将两个 int16x8_t 向量的元素相加，得到 int32x4_t 向量
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 将 int32x4_t 向量转换为 float32x4_t 向量，然后进行乘法和加法运算
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif
    }

    // 将两个 float32x4_t 向量的元素相加，得到一个标量值
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__wasm_simd128__)
    // 创建一个所有元素为 0.0f 的 float32x4_t 向量
    v128_t sumv = wasm_f32x4_splat(0.0f);

    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }

    // 提取 float32x4_t 向量的每个元素，并相加得到一个标量值
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3);
#elif defined(__AVX2__)
    // 用零初始化一个 __m256 向量
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; i++) {
        /* 对块进行组合比例计算 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 从四位半字节中获取字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从四位比特中获取字节
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 对 bxhi 进行位求反并与 0xF0 进行与操作
        bxhi = _mm256_andnot_si256(bxhi, _mm256_set1_epi8((char)0xF0));
        // 对 bx 和 bxhi 进行或操作
        bx = _mm256_or_si256(bx, bxhi);

        // 从内存中加载 32 位整数
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 将 bx 和 by 进行乘法求和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 将 q 与比例相乘并累加 */
        acc = _mm256_fmadd_ps(d, q, acc);
    }

    // 对 8 个单精度浮点数进行水平求和
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 如果 AVX 被定义，则执行以下代码块

    // 用零初始化累加器
    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含指定值的 128 位整数
    __m128i mask = _mm_set1_epi8((char)0xF0);

    // 主循环
    for (int i = 0; i < nb; i++) {
        /* 计算块的组合比例 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 32 位整数转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从 32 位整数中提取高 128 位
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        bxhil = _mm_andnot_si128(bxhil, mask);
        bxhih = _mm_andnot_si128(bxhih, mask);
        __m128i bxl = _mm256_castsi256_si128(bx);
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        bxl = _mm_or_si128(bxl, bxhil);
        bxh = _mm_or_si128(bxh, bxhih);
        bx = MM256_SET_M128I(bxh, bxl);

        // 从内存中加载 256 位整数
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 将 q 乘以比例并累加 */
        acc = _mm256_add_ps(_mm256_mul_ps(d, q), acc);
    }

    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 如果 __riscv_v_intrinsic 被定义，则执行以下代码块
    float sumf = 0.0;

    uint32_t qh;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 这些临时寄存器用于掩码和移位操作
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vsll_vv_u32m2(__riscv_vmv_v_x_u32m2(1, vl), vt_1, vl);

    vuint32m2_t vt_3 = __riscv_vsll_vx_u32m2(vt_2, 16, vl);
    vuint32m2_t vt_4 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    }

    *s = sumf;
#else
    // 如果以上条件都不满足，则执行以下代码块
    float sumf = 0.0;
    # 遍历 nb 次循环
    for (int i = 0; i < nb; i++) {
        # 定义一个 32 位无符号整数变量 qh，并将 x[i].qh 的内容拷贝到 qh 中
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 初始化 sumi 为 0
        int sumi = 0;

        # 遍历 qk/2 次循环
        for (int j = 0; j < qk/2; ++j) {
            # 计算 xh_0 和 xh_1 的值
            const uint8_t xh_0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
            const uint8_t xh_1 = ((qh & (1u << (j + 16))) >> (j + 12));

            # 计算 x0 和 x1 的值
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            # 计算 sumi 的累加值
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }

        # 计算 sumf 的累加值
        sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;
    }

    # 将 sumf 的值赋给指针变量 s
    *s = sumf;
#endif
}

void ggml_vec_dot_q5_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_1;
    const int nb = n / qk;

    // 断言 n 能被 qk 整除
    assert(n % qk == 0);
    // 断言 qk 等于 QK5_1
    assert(qk == QK5_1);

    // 限定 x 指向 block_q5_1 类型的指针
    const block_q5_1 * restrict x = vx;
    // 限定 y 指向 block_q8_1 类型的指针
    const block_q8_1 * restrict y = vy;

#if defined(__ARM_NEON)
    // 创建 4 个 32 位浮点数的向量，每个元素的值都是 0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 创建 2 个 32 位浮点数，值都是 0.0f
    float summs0 = 0.0f;
    float summs1 = 0.0f;

    // 无符号整数 qh0 和 qh1
    uint32_t qh0;
    uint32_t qh1;

    // 两个 64 位无符号整数数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言 nb 是偶数
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 如果支持 ARM Dot Product 指令集
    #if defined(__ARM_FEATURE_DOTPROD)
        // 计算 sumv0
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*y0->d);
        // 计算 sumv1
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
        // 计算两个 int8x8_t 向量的低位元素乘积
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        // 计算两个 int8x8_t 向量的高位元素乘积
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        // 计算两个 int8x8_t 向量的低位元素乘积
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        // 计算两个 int8x8_t 向量的高位元素乘积
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        // 计算两个 int8x8_t 向量的低位元素乘积
        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        // 计算两个 int8x8_t 向量的高位元素乘积
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        // 计算两个 int8x8_t 向量的低位元素乘积
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        // 计算两个 int8x8_t 向量的高位元素乘积
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));

        // 将两个 int16x8_t 向量的元素相加，得到 int32x4_t 向量
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 将两个 int32x4_t 向量的元素相加，再转换为 float32x4_t 向量，与另一 float32x4_t 向量相乘，再累加到 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
    }

    // 将两个 float32x4_t 向量的元素相加，再将结果的每个元素相加，再加上 summs0 和 summs1，赋值给 s
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs0 + summs1;
#elif defined(__wasm_simd128__)
    // 创建一个所有元素为 0.0f 的 float32x4_t 向量
    v128_t sumv = wasm_f32x4_splat(0.0f);

    // 初始化 summs 为 0.0f
    float summs = 0.0f;

    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }

    // 将 sumv 的每个元素提取出来相加，再加上 summs，赋值给 s
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3) + summs;
#elif defined(__AVX2__)
    // 初始化一个所有元素为 0 的 __m256 向量
    __m256 acc = _mm256_setzero_ps();

    // 初始化 summs 为 0.0f
    float summs = 0.0f;

    // 主循环
    # 遍历循环，从 i=0 开始，直到 i<nb 为止
    for (int i = 0; i < nb; i++) {
        # 将 x[i].d 转换成 256 位的浮点数向量
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        # 计算 summs，累加 x[i].m 乘以 y[i].s 的结果
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        # 将 x[i].qs 转换成 256 位的整数向量
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        # 将 x[i].qh 转换成 256 位的整数向量
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        # 将 bxhi 中的每个元素与 0x10 进行按位与操作
        bxhi = _mm256_and_si256(bxhi, _mm256_set1_epi8(0x10));
        # 将 bx 和 bxhi 中的每个元素进行按位或操作
        bx = _mm256_or_si256(bx, bxhi);

        # 将 y[i].d 转换成 256 位的浮点数向量
        const __m256 dy = _mm256_set1_ps(y[i].d);
        # 从内存中加载 256 位的整数向量到 by
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        # 计算乘积和的无符号 8 位整数对的浮点数乘积
        const __m256 q = mul_sum_us8_pairs_float(bx, by);

        # 使用 FMA 指令计算累加器的新值
        acc = _mm256_fmadd_ps(q, _mm256_mul_ps(dx, dy), acc);
    }

    # 将 acc 中的 8 个浮点数求和，并加上 summs 的值，存储到 s 指针指向的位置
    *s = hsum_float_8(acc) + summs;
#elif defined(__AVX__)
    // 使用 AVX 指令集，初始化累加器为零
    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含 0x10 的 128 位整数向量
    __m128i mask = _mm_set1_epi8(0x10);

    // 初始化浮点数 summs 为 0.0
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将 GGML_FP16_TO_FP32(x[i].d) 转换为 256 位浮点数向量
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将 x[i].qs 转换为 256 位整数向量
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从 bx 中提取高 128 位和低 128 位
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        // 对 bxhil 和 bxhih 进行按位与操作
        bxhil = _mm_and_si128(bxhil, mask);
        bxhih = _mm_and_si128(bxhih, mask);
        // 对 bxl 和 bxh 进行按位或操作
        __m128i bxl = _mm256_castsi256_si128(bx);
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        bxl = _mm_or_si128(bxl, bxhil);
        bxh = _mm_or_si128(bxh, bxhih);
        bx = MM256_SET_M128I(bxh, bxl);

        // 将 y[i].d 转换为 256 位浮点数向量
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从 y[i].qs 中加载 256 位整数向量
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并累加到 acc 中
        const __m256 q = mul_sum_us8_pairs_float(bx, by);
        acc = _mm256_add_ps(_mm256_mul_ps(q, _mm256_mul_ps(dx, dy)), acc);
    }

    // 计算最终结果并赋值给 s
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 使用 RISC-V 指令集，初始化 sumf 为 0.0
    float sumf = 0.0;

    // 初始化 qh
    uint32_t qh;

    // 设置向量长度为 qk/2
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 为移位操作创建临时寄存器
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);
    // 循环遍历从 0 到 nb-1 的整数
    for (int i = 0; i < nb; i++) {
        // 将 x[i].qh 的 sizeof(uint32_t) 大小的数据拷贝到 qh 中
        memcpy(&qh, x[i].qh, sizeof(uint32_t));

        // 加载 qh 到向量寄存器 vqh 中
        vuint32m2_t vqh = __riscv_vmv_v_x_u32m2(qh, vl);

        // 计算 ((qh >> (j +  0)) << 4) & 0x10
        vuint32m2_t xhr_0 = __riscv_vsrl_vv_u32m2(vqh, vt_1, vl);
        vuint32m2_t xhl_0 = __riscv_vsll_vx_u32m2(xhr_0, 4, vl);
        vuint32m2_t xha_0 = __riscv_vand_vx_u32m2(xhl_0, 0x10, vl);

        // 计算 ((qh >> (j + 12))     ) & 0x10
        vuint32m2_t xhr_1 = __riscv_vsrl_vv_u32m2(vqh, vt_2, vl);
        vuint32m2_t xha_1 = __riscv_vand_vx_u32m2(xhr_1, 0x10, vl);

        // 将 xha_0 向量寄存器中的数据转换为 16 位有符号整数
        vuint16m1_t xhc_0 = __riscv_vncvt_x_x_w_u16m1(xha_0, vl);
        // 将 xhc_0 向量寄存器中的数据转换为 8 位有符号整数
        vuint8mf2_t xh_0 = __riscv_vncvt_x_x_w_u8mf2(xhc_0, vl);

        // 将 xha_1 向量寄存器中的数据转换为 16 位有符号整数
        vuint16m1_t xhc_1 = __riscv_vncvt_x_x_w_u16m1(xha_1, vl);
        // 将 xhc_1 向量寄存器中的数据转换为 8 位有符号整数
        vuint8mf2_t xh_1 = __riscv_vncvt_x_x_w_u8mf2(xhc_1, vl);

        // 加载 x[i].qs 向量数据到 tx 中
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        // 加载 y[i].qs 和 y[i].qs+16 向量数据到 y0 和 y1 中
        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 计算 x[i].qs 和 0x0F 的按位与
        vuint8mf2_t x_at = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        // 计算 x[i].qs 右移 4 位和 0x10 的按位与
        vuint8mf2_t x_lt = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        // 计算 x_at 和 xh_0 的按位或
        vuint8mf2_t x_a = __riscv_vor_vv_u8mf2(x_at, xh_0, vl);
        // 计算 x_lt 和 xh_1 的按位或
        vuint8mf2_t x_l = __riscv_vor_vv_u8mf2(x_lt, xh_1, vl);

        // 将 x_a 向量寄存器中的数据重新解释为 8 位有符号整数
        vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        // 将 x_l 向量寄存器中的数据重新解释为 8 位有符号整数
        vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        // 计算 v0 和 y0 的宽乘法
        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        // 计算 v1 和 y1 的宽乘法
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        // 创建一个值全为 0 的 32 位有符号整数向量
        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        // 对 vec_mul1 进行宽加和操作
        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        // 对 vec_mul2 进行宽加和操作
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        // 将 vs2 中的数据移动到标量寄存器 sumi 中
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        // 计算 sumf 的值
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
#else
    // 如果不是 ARM_NEON，使用标量计算
    float sumf = 0.0;  // 初始化浮点数 sumf 为 0.0

    for (int i = 0; i < nb; i++) {  // 遍历 nb 次
        uint32_t qh;  // 定义无符号整型变量 qh
        memcpy(&qh, x[i].qh, sizeof(qh));  // 将 x[i].qh 的内容复制到 qh

        int sumi = 0;  // 初始化整型变量 sumi 为 0

        for (int j = 0; j < qk/2; ++j) {  // 遍历 qk/2 次
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;  // 计算 xh_0
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;  // 计算 xh_1

            const int32_t x0 = (x[i].qs[j] & 0xF) | xh_0;  // 计算 x0
            const int32_t x1 = (x[i].qs[j] >>  4) | xh_1;  // 计算 x1

            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);  // 更新 sumi
        }

        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;  // 更新 sumf
    }

    *s = sumf;  // 将 sumf 赋值给指针 s
#endif
}

void ggml_vec_dot_q8_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_0;  // 定义常量 qk 为 QK8_0
    const int nb = n / qk;  // 计算 nb

    assert(n % qk == 0);  // 断言 n 能被 qk 整除

    const block_q8_0 * restrict x = vx;  // 定义指向 vx 的指针 x
    const block_q8_0 * restrict y = vy;  // 定义指向 vy 的指针 y

#if defined(__ARM_NEON)
    float32x4_t sumv0 = vdupq_n_f32(0.0f);  // 初始化 sumv0 为 0.0
    float32x4_t sumv1 = vdupq_n_f32(0.0f);  // 初始化 sumv1 为 0.0

    assert(nb % 2 == 0); // 断言 nb 是偶数，处理奇数 nb 的情况

    for (int i = 0; i < nb; i += 2) {  // 每次增加 2
        const block_q8_0 * restrict x0 = &x[i + 0];  // 定义指向 x[i+0] 的指针 x0
        const block_q8_0 * restrict x1 = &x[i + 1];  // 定义指向 x[i+1] 的指针 x1
        const block_q8_0 * restrict y0 = &y[i + 0];  // 定义指向 y[i+0] 的指针 y0
        const block_q8_0 * restrict y1 = &y[i + 1];  // 定义指向 y[i+1] 的指针 y1

        const int8x16_t x0_0 = vld1q_s8(x0->qs);  // 加载 x0->qs 到 x0_0
        const int8x16_t x0_1 = vld1q_s8(x0->qs + 16);  // 加载 x0->qs+16 到 x0_1
        const int8x16_t x1_0 = vld1q_s8(x1->qs);  // 加载 x1->qs 到 x1_0
        const int8x16_t x1_1 = vld1q_s8(x1->qs + 16);  // 加载 x1->qs+16 到 x1_1

        // 加载 y
        const int8x16_t y0_0 = vld1q_s8(y0->qs);  // 加载 y0->qs 到 y0_0
        const int8x16_t y0_1 = vld1q_s8(y0->qs + 16);  // 加载 y0->qs+16 到 y0_1
        const int8x16_t y1_0 = vld1q_s8(y1->qs);  // 加载 y1->qs 到 y1_0
        const int8x16_t y1_1 = vld1q_s8(y1->qs + 16);  // 加载 y1->qs+16 到 y1_1
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM Dot Product 指令集，则使用 vdotq_s32 和 vmlaq_n_f32 计算 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                    vdotq_s32(vdupq_n_s32(0), x0_0, y0_0),
                    vdotq_s32(vdupq_n_s32(0), x0_1, y0_1))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));

    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                    vdotq_s32(vdupq_n_s32(0), x1_0, y1_0),
                    vdotq_s32(vdupq_n_s32(0), x1_1, y1_1))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));

#else
    // 如果不支持 ARM Dot Product 指令集，则使用 vmull_s8 和 vpaddlq_s16 计算 p0_0, p0_1, p0_2, p0_3, p1_0, p1_1, p1_2, p1_3
    const int16x8_t p0_0 = vmull_s8(vget_low_s8 (x0_0), vget_low_s8 (y0_0));
    const int16x8_t p0_1 = vmull_s8(vget_high_s8(x0_0), vget_high_s8(y0_0));
    const int16x8_t p0_2 = vmull_s8(vget_low_s8 (x0_1), vget_low_s8 (y0_1));
    const int16x8_t p0_3 = vmull_s8(vget_high_s8(x0_1), vget_high_s8(y0_1));

    const int16x8_t p1_0 = vmull_s8(vget_low_s8 (x1_0), vget_low_s8 (y1_0));
    const int16x8_t p1_1 = vmull_s8(vget_high_s8(x1_0), vget_high_s8(y1_0));
    const int16x8_t p1_2 = vmull_s8(vget_low_s8 (x1_1), vget_low_s8 (y1_1));
    const int16x8_t p1_3 = vmull_s8(vget_high_s8(x1_1), vget_high_s8(y1_1));

    const int32x4_t p0 = vaddq_s32(vpaddlq_s16(p0_0), vpaddlq_s16(p0_1));
    const int32x4_t p1 = vaddq_s32(vpaddlq_s16(p0_2), vpaddlq_s16(p0_3));
    const int32x4_t p2 = vaddq_s32(vpaddlq_s16(p1_0), vpaddlq_s16(p1_1));
    const int32x4_t p3 = vaddq_s32(vpaddlq_s16(p1_2), vpaddlq_s16(p1_3));

    // 使用 vaddq_s32 和 vmlaq_n_f32 计算 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(p0, p1)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(p2, p3)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif

// 计算 sumv0 和 sumv1 的和，并将结果存储在指针 s 指向的位置
*s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果支持 AVX2 或 AVX 指令集，则初始化一个全为 0 的 256 位浮点数寄存器
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));
        // 从 x[i].qs 加载 256 位整数到 bx
        __m256i bx = _mm256_loadu_si256((const __m256i *)x[i].qs);
        // 从 y[i].qs 加载 256 位整数到 by
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算 bx 和 by 的乘积之和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加
#if defined(__AVX2__)
        // 如果定义了 AVX2，则使用 AVX2 指令集进行计算
        acc = _mm256_fmadd_ps( d, q, acc );
#else
        // 如果未定义 AVX2，则使用普通的乘加操作
        acc = _mm256_add_ps( _mm256_mul_ps( d, q ), acc );
#endif
    }

    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 如果定义了 __riscv_v_intrinsic，则使用 RISC-V 指令集进行计算
    float sumf = 0.0;
    size_t vl = __riscv_vsetvl_e8m1(qk);

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vint8m1_t bx = __riscv_vle8_v_i8m1(x[i].qs, vl);
        vint8m1_t by = __riscv_vle8_v_i8m1(y[i].qs, vl);

        // 两个向量的乘法
        vint16m2_t vw_mul = __riscv_vwmul_vv_i16m2(bx, by, vl);

        // 初始化一个全为 0 的向量
        vint32m1_t v_zero = __riscv_vmv_v_x_i32m1(0, vl);
        // 对 16 位整数向量进行求和
        vint32m1_t v_sum = __riscv_vwredsum_vs_i16m2_i32m1(vw_mul, v_zero, vl);

        // 将求和结果转换为整数
        int sumi = __riscv_vmv_x_s_i32m1_i32(v_sum);

        // 计算浮点数的和
        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
#else
    // 如果未定义 __AVX2__ 和 __riscv_v_intrinsic，则使用标量计算
    float sumf = 0.0;

    for (int i = 0; i < nb; i++) {
        int sumi = 0;

        for (int j = 0; j < qk; j++) {
            sumi += x[i].qs[j]*y[i].qs[j];
        }

        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
#endif
}

#if QK_K == 256
void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    const block_q2_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 如果定义了 __ARM_NEON，则使用 ARM NEON 指令集进行计算
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    const uint8x16_t m4 = vdupq_n_u8(0xF);
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    ggml_int8x16x2_t q2bytes;
    uint8_t aux[16];

    float sum = 0;
    # 遍历循环，从 i=0 开始，直到 i<nb 为止
    for (int i = 0; i < nb; ++i) {

        # 计算 d 值，y[i].d 乘以 x[i].d 的浮点数结果
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        # 计算 dmin 值，-y[i].d 乘以 x[i].dmin 的浮点数结果
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        # 定义指向 x[i].qs 的 uint8_t 类型指针 q2
        const uint8_t * restrict q2 = x[i].qs;
        # 定义指向 y[i].qs 的 int8_t 类型指针 q8
        const int8_t  * restrict q8 = y[i].qs;
        # 定义指向 x[i].scales 的 uint8_t 类型指针 sc
        const uint8_t * restrict sc = x[i].scales;

        # 从 sc 中加载 16 个 uint8_t 类型的值到 mins_and_scales 中
        const uint8x16_t mins_and_scales = vld1q_u8(sc);
        # 使用 m4 对 mins_and_scales 进行按位与操作，结果存储到 scales 中
        const uint8x16_t scales = vandq_u8(mins_and_scales, m4);
        # 将 scales 中的值存储到 aux 中
        vst1q_u8(aux, scales);

        # 将 mins_and_scales 中的值右移 4 位，存储到 mins 中
        const uint8x16_t mins = vshrq_n_u8(mins_and_scales, 4);
        # 从 y[i].bsums 中加载 16 个 int16_t 类型的值到 q8sums 中
        const ggml_int16x8x2_t q8sums = ggml_vld1q_s16_x2(y[i].bsums);
        # 将 mins 转换为 int16 类型，存储到 mins16 中
        const ggml_int16x8x2_t mins16 = {vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(mins))), vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(mins)))};
        # 计算 s0 值
        const int32x4_t s0 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[0]), vget_low_s16 (q8sums.val[0])),
                                       vmull_s16(vget_high_s16(mins16.val[0]), vget_high_s16(q8sums.val[0])));
        # 计算 s1 值
        const int32x4_t s1 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[1]), vget_low_s16 (q8sums.val[1])),
                                       vmull_s16(vget_high_s16(mins16.val[1]), vget_high_s16(q8sums.val[1])));
        # 计算 sum 值，dmin 乘以 s0 和 s1 的和
        sum += dmin * vaddvq_s32(vaddq_s32(s0, s1));

        # 初始化 isum 和 is 变量
        int isum = 0;
        int is = 0;
// 定义宏，根据 ARM 特性选择不同的计算方式
#if defined(__ARM_FEATURE_DOTPROD)
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        isum += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * aux[is+(index)];\
        isum += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * aux[is+1+(index)];
#else
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        {\
    // 计算两个向量的点积并乘以辅助数组的值
    const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0])));\
    const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[1]), vget_low_s8 (q8bytes.val[1])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[1]), vget_high_s8(q8bytes.val[1])));\
    isum += vaddvq_s16(p1) * aux[is+(index)] + vaddvq_s16(p2) * aux[is+1+(index)];\
        }
#endif

// 定义宏，根据位移和索引计算乘积并累加
#define SHIFT_MULTIPLY_ACCUM_WITH_SCALE(shift, index)\
        q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;\
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[0], (shift)), m3));\
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[1], (shift)), m3));\
        MULTIPLY_ACCUM_WITH_SCALE((index));

// 循环处理每个 128 位的数据
for (int j = 0; j < QK_K/128; ++j) {
    // 从内存加载两个 128 位的无符号字符型数据
    const ggml_uint8x16x2_t q2bits = ggml_vld1q_u8_x2(q2); q2 += 32;

    // 从内存加载两个 128 位的有符号字符型数据
    ggml_int8x16x2_t q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
    // 对两个 128 位数据进行位与运算
    q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[0], m3));
    q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[1], m3));
    // 根据索引调用宏进行计算
    MULTIPLY_ACCUM_WITH_SCALE(0);

    // 根据位移和索引调用宏进行计算
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(2, 2);

    // 根据位移和索引调用宏进行计算
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(4, 4);

    // 根据位移和索引调用宏进行计算
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(6, 6);

    // 更新索引
    is += 8;
}
// 计算最终结果并累加
sum += d * isum;

// 将结果赋值给指针所指向的变量
*s = sum;
#elif defined __AVX2__

    // 创建一个包含 8 个相同值的 8 位整数的 AVX 寄存器
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含 16 个相同值的 8 位整数的 AVX 寄存器
    const __m128i m4 = _mm_set1_epi8(0xF);

    // 创建一个全零的 256 位浮点数 AVX 寄存器
    __m256 acc = _mm256_setzero_ps();

    }

    // 对 8 个单精度浮点数进行水平求和
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 创建一个包含 16 个相同值的 8 位整数的 AVX 寄存器
    const __m128i m3 = _mm_set1_epi8(0x3);
    // 创建一个包含 16 个相同值的 8 位整数的 AVX 寄存器
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个包含 16 个相同值的 8 位整数的 AVX 寄存器
    const __m128i m2 = _mm_set1_epi8(0x2);

    // 创建一个全零的 256 位浮点数 AVX 寄存器
    __m256 acc = _mm256_setzero_ps();

    }

    // 对 8 个单精度浮点数进行水平求和
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic

    // 初始化浮点数 sumf
    float sumf = 0;
    // 初始化长度为 32 的无符号 8 位整数数组 temp_01
    uint8_t temp_01[32] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                            1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1};

    }

    // 将 sumf 赋值给指针 s 指向的变量
    *s = sumf;

#else

    // 初始化浮点数 sumf
    float sumf = 0;

    // 遍历数组 x 中的元素
    for (int i = 0; i < nb; ++i) {

        // 获取指针 q2 和 q8 指向的值
        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;

        // 初始化整数 summs
        int summs = 0;
        // 遍历数组 y[i].bsums 中的元素
        for (int j = 0; j < 16; ++j) {
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        // 计算浮点数 dall 和 dmin
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 初始化整数 isum 和 is
        int isum = 0;
        int is = 0;
        int d;
        // 遍历数组 x[i].scales 中的元素
        for (int k = 0; k < QK_K/128; ++k) {
            int shift = 0;
            // 遍历数组 q2 和 q8 中的元素
            for (int j = 0; j < 4; ++j) {
                d = sc[is++] & 0xF;
                int isuml = 0;
                // 遍历数组 q8 中的元素
                for (int l =  0; l < 16; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3);
                isum += d * isuml;
                d = sc[is++] & 0xF;
                isuml = 0;
                // 遍历数组 q8 中的元素
                for (int l = 16; l < 32; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3);
                isum += d * isuml;
                shift += 2;
                q8 += 32;
            }
            q2 += 32;
        }
        // 更新 sumf
        sumf += dall * isum - dmin * summs;
    }
    // 将 sumf 赋值给指针 s 指向的变量
    *s = sumf;
#endif
}

#else

void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    // 获取指针 x 指向的值
    const block_q2_K * restrict x = vx;
    # 声明一个指向常量 block_q8_K 类型的指针 y，并将其指向 vy
    const block_q8_K * restrict y = vy;

    # 计算 n 除以 QK_K 的结果，并赋值给变量 nb
    const int nb = n / QK_K;
#ifdef __ARM_NEON
    # 如果支持 ARM NEON 指令集

    const uint8x16_t m3 = vdupq_n_u8(0x3);
    # 创建一个包含 16 个 0x3 的向量

#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
    # 如果支持 ARM Dot Product 指令集，创建一个包含 4 个 0 的向量
#endif

    ggml_int8x16x4_t q2bytes;
    # 创建一个包含 4 个 16 位整数向量的结构体

    uint32_t aux32[2];
    # 创建一个包含 2 个 32 位整数的数组
    const uint8_t * scales = (const uint8_t *)aux32;
    # 将 aux32 强制转换为 uint8_t 类型的指针，并赋值给 scales

    float sum = 0;
    # 创建一个浮点数 sum，初始化为 0

    for (int i = 0; i < nb; ++i) {
        # 循环遍历 nb 次，i 从 0 到 nb-1

        const float d = y[i].d * (float)x[i].d;
        # 计算 d，y[i].d 乘以 x[i].d 的浮点数结果
        const float dmin = -y[i].d * (float)x[i].dmin;
        # 计算 dmin，-y[i].d 乘以 x[i].dmin 的浮点数结果

        const uint8_t * restrict q2 = x[i].qs;
        # 创建一个指向 x[i].qs 的 uint8_t 类型指针，并限制其它指针不能指向同一内存位置
        const int8_t  * restrict q8 = y[i].qs;
        # 创建一个指向 y[i].qs 的 int8_t 类型指针，并限制其它指针不能指向同一内存位置
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        # 创建一个指向 x[i].scales 的 uint32_t 类型指针，并限制其它指针不能指向同一内存位置

        aux32[0] = sc[0] & 0x0f0f0f0f;
        # 将 sc[0] 与 0x0f0f0f0f 按位与的结果存入 aux32[0]
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;
        # 将 sc[0] 右移 4 位后再与 0x0f0f0f0f 按位与的结果存入 aux32[1]

        sum += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);
        # 计算 sum，dmin 乘以一系列值的和，并加到 sum 上

        int isum1 = 0, isum2 = 0;
        # 创建两个整数 isum1 和 isum2，初始化为 0

        const uint8x16_t q2bits = vld1q_u8(q2);
        # 从 q2 加载 16 个 uint8_t 类型的值到 q2bits

        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);
        # 从 q8 加载 4 个 16 位整数向量到 q8bytes

        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits, m3));
        # 将 q2bits 与 m3 按位与的结果转换为 int8x16_t 类型后存入 q2bytes.val[0]
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 2), m3));
        # 将 q2bits 右移 2 位后与 m3 按位与的结果转换为 int8x16_t 类型后存入 q2bytes.val[1]
        q2bytes.val[2] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 4), m3));
        # 将 q2bits 右移 4 位后与 m3 按位与的结果转换为 int8x16_t 类型后存入 q2bytes.val[2]
        q2bytes.val[3] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 6), m3));
        # 将 q2bits 右移 6 位后与 m3 按位与的结果转换为 int8x16_t 类型后存入 q2bytes.val[3]

#if defined(__ARM_FEATURE_DOTPROD)
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * scales[0];
        # 如果支持 ARM Dot Product 指令集，计算 isum1
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * scales[1];
        # 如果支持 ARM Dot Product 指令集，计算 isum2
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[2], q8bytes.val[2])) * scales[2];
        # 如果支持 ARM Dot Product 指令集，计算 isum1
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[3], q8bytes.val[3])) * scales[3];
        # 如果支持 ARM Dot Product 指令集，计算 isum2
#else
        // 定义并计算第一组乘积和
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0]));
        // 定义并计算第二组乘积和
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q2bytes.val[1]), vget_high_s8(q8bytes.val[1]));
        // 将第一组乘积和乘以对应的比例因子，并累加到isum1
        isum1 += vaddvq_s16(p1) * scales[0];
        // 将第二组乘积和乘以对应的比例因子，并累加到isum2
        isum2 += vaddvq_s16(p2) * scales[1];

        // 定义并计算第三组乘积和
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q2bytes.val[2]), vget_high_s8(q8bytes.val[2]));
        // 定义并计算第四组乘积和
        const int16x8_t p4 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q2bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        // 将第三组乘积和乘以对应的比例因子，并累加到isum1
        isum1 += vaddvq_s16(p3) * scales[2];
        // 将第四组乘积和乘以对应的比例因子，并累加到isum2
        isum2 += vaddvq_s16(p4) * scales[3];
#endif
        // 将d乘以(isum1 + isum2)的和，并累加到sum
        sum += d * (isum1 + isum2);

    }

    *s = sum;

#elif defined __AVX2__

    // 设置一个256位整数，每个字节都是3
    const __m256i m3 = _mm256_set1_epi8(3);

    // 设置一个256位浮点数，所有元素都是0
    __m256 acc = _mm256_setzero_ps();

    uint32_t ud, um;
    // 定义并初始化指向ud的字节指针
    const uint8_t * restrict db = (const uint8_t *)&ud;
    // 定义并初始化指向um的字节指针
    const uint8_t * restrict mb = (const uint8_t *)&um;

    // 初始化一个浮点数变量
    float summs = 0;

    // 待优化的部分，需要进一步优化
    // TODO: optimize this
    # 遍历循环，从 i=0 到 nb-1
    for (int i = 0; i < nb; ++i) {

        # 计算 d 值，y[i].d 乘以 x[i].d 的浮点数结果
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        # 计算 dmin 值，-y[i].d 乘以 x[i].dmin 的浮点数结果
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        # 定义指向 x[i].qs 的 uint8_t 类型指针 q2
        const uint8_t * restrict q2 = x[i].qs;
        # 定义指向 y[i].qs 的 int8_t 类型指针 q8
        const int8_t  * restrict q8 = y[i].qs;

        # 定义指向 x[i].scales 的 uint32_t 类型指针 sc
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        # 从 sc[0] 中提取 ud 和 um 的值
        ud = (sc[0] >> 0) & 0x0f0f0f0f;
        um = (sc[0] >> 4) & 0x0f0f0f0f;

        # 计算 smin 值，mb 数组元素与 y[i].bsums 数组元素的乘积之和
        int32_t smin = mb[0] * y[i].bsums[0] + mb[1] * y[i].bsums[1] + mb[2] * y[i].bsums[2] + mb[3] * y[i].bsums[3];
        # 计算 summs 值，dmin 乘以 smin 的结果之和
        summs += dmin * smin;

        # 加载 q2 指向的数据到 __m128i 类型的寄存器 q2bits
        const __m128i q2bits = _mm_loadu_si128((const __m128i*)q2);
        # 对 q2bits 进行位运算，得到 q2_0 和 q2_1
        const __m256i q2_0 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 2), q2bits), m3);
        const __m256i q2_1 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 6), _mm_srli_epi16(q2bits, 4)), m3);

        # 加载 q8 指向的数据到 __m256i 类型的寄存器 q8_0 和 q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 对 q2_0 和 q8_0 进行乘法累加，得到 p0
        const __m256i p0 = _mm256_maddubs_epi16(q2_0, q8_0);
        # 对 q2_1 和 q8_1 进行乘法累加，得到 p1
        const __m256i p1 = _mm256_maddubs_epi16(q2_1, q8_1);

        # 将 p0 和 p1 转换为 32 位整数类型
        const __m256i p_0 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 0));
        const __m256i p_1 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 1));
        const __m256i p_2 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 0));
        const __m256i p_3 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 1));

        # 使用 d 乘以 db 数组元素，与 p0、p1、p2、p3 进行浮点数累加，得到 acc
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[0]), _mm256_cvtepi32_ps(p_0), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[1]), _mm256_cvtepi32_ps(p_1), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[2]), _mm256_cvtepi32_ps(p_2), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[3]), _mm256_cvtepi32_ps(p_3), acc);
    }

    # 将 acc 的值进行横向求和，加上 summs 的值，赋给 *s
    *s = hsum_float_8(acc) + summs;
#elif defined __AVX__

    // 定义一个包含全部元素为3的__m128i类型的变量m3
    const __m128i m3 = _mm_set1_epi8(3);

    // 初始化一个全部元素为0的__m256类型的变量acc
    __m256 acc = _mm256_setzero_ps();

    // 定义两个32位整数变量ud和um
    uint32_t ud, um;
    // 将ud和um的地址转换为指向uint8_t类型的指针，并使用restrict关键字进行限定
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;

    // 初始化一个浮点数变量summs为0
    float summs = 0;

    // TODO: optimize this
    // 待优化的部分

    }

    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic

    // 定义一个包含两个32位整数的数组aux32
    uint32_t aux32[2];
    // 将aux32的地址转换为指向uint8_t类型的指针scales
    const uint8_t * scales = (const uint8_t *)aux32;

    // 初始化一个浮点数变量sumf为0
    float sumf = 0;

    }

    *s = sumf;

#else

    // 初始化一个浮点数变量sumf为0
    float sumf = 0;

    // 定义一个包含4个整数的数组isum
    int isum[4];

    // 遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 获取x[i]和y[i]的qs和scales
        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;

        // 初始化一个整数变量summs为0
        int summs = 0;
        // 遍历QK_K/16次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算summs的值
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        // 计算dall和dmin的值
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 将isum数组的所有元素初始化为0
        isum[0] = isum[1] = isum[2] = isum[3] = 0;
        // 遍历16次
        for (int l =  0; l < 16; ++l) {
            // 计算isum数组的值
            isum[0] += q8[l+ 0] * ((q2[l] >> 0) & 3);
            isum[1] += q8[l+16] * ((q2[l] >> 2) & 3);
            isum[2] += q8[l+32] * ((q2[l] >> 4) & 3);
            isum[3] += q8[l+48] * ((q2[l] >> 6) & 3);
        }
        // 遍历4次
        for (int l = 0; l < 4; ++l) {
            // 计算isum数组的值
            isum[l] *= (sc[l] & 0xF);
        }
        // 计算sumf的值
        sumf += dall * (isum[0] + isum[1] + isum[2] + isum[3]) - dmin * summs;
    }
    // 将sumf的值赋给*s
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言n能被QK_K整除
    assert(n % QK_K == 0);

    // 定义两个32位整数变量kmask1和kmask2
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 将vx和vy转换为block_q3_K和block_q8_K类型的指针x和y
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算nb的值
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 定义两个32位整数数组aux和utmp
    uint32_t aux[3];
    uint32_t utmp[4];

    // 初始化一个包含16个元素，每个元素为3的uint8x16_t类型的变量m3b
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 如果支持ARM的dot product指令，则初始化一个包含4个元素，每个元素为0的int32x4_t类型的变量vzero
#ifdef __ARM_FEATURE_DOTPROD
    const int32x4_t  vzero = vdupq_n_s32(0);
// 定义一个包含16个8位无符号整数的向量，每个元素都是1
const uint8x16_t m0 = vdupq_n_u8(1);
// 将m0中的每个元素左移1位，得到新的向量m1
const uint8x16_t m1 = vshlq_n_u8(m0, 1);
// 将m0中的每个元素左移2位，得到新的向量m2
const uint8x16_t m2 = vshlq_n_u8(m0, 2);
// 将m0中的每个元素左移3位，得到新的向量m3
const uint8x16_t m3 = vshlq_n_u8(m0, 3);
// 定义一个包含16个8位有符号整数的向量，每个元素都是32
const int8_t m32 = 32;

// 定义一个包含4个16字节的整数向量的结构体
ggml_int8x16x4_t q3bytes;

// 定义一个浮点数变量sum，初始值为0
float sum = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 获取指向 x[i].qs 的指针
        const uint8_t * restrict q3 = x[i].qs;
        // 获取指向 x[i].hmask 的指针
        const uint8_t * restrict qh = x[i].hmask;
        // 获取指向 y[i].qs 的指针
        const int8_t  * restrict q8 = y[i].qs;

        // 从 qh 中加载 16 个字节到 qhbits
        ggml_uint8x16x2_t qhbits = ggml_vld1q_u8_x2(qh);

        // 定义 q3h 变量
        ggml_uint8x16x4_t q3h;

        // 初始化 isum 为 0
        int32_t isum = 0;

        // 设置比例
        memcpy(aux, x[i].scales, 12);
        utmp[3] = ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4);
        utmp[2] = ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4);
        utmp[1] = (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4);
        utmp[0] = (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4);

        // 将 utmp 转换为 int8_t 类型的 scale
        int8_t * scale = (int8_t *)utmp;
        // 对 scale 中的每个元素减去 m32
        for (int j = 0; j < 16; ++j) scale[j] -= m32;

        // 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从 q3 中加载 32 个字节到 q3bits，并更新 q3 指针
            const ggml_uint8x16x2_t q3bits = ggml_vld1q_u8_x2(q3); q3 += 32;
            // 从 q8 中加载 64 个字节到 q8bytes_1，并更新 q8 指针
            const ggml_int8x16x4_t q8bytes_1 = ggml_vld1q_s8_x4(q8); q8 += 64;
            // 从 q8 中加载 64 个字节到 q8bytes_2，并更新 q8 指针
            const ggml_int8x16x4_t q8bytes_2 = ggml_vld1q_s8_x4(q8); q8 += 64;

            // 计算 q3h.val 中的值
            q3h.val[0] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[0]), 2);
            q3h.val[1] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[1]), 2);
            q3h.val[2] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[0]), 1);
            q3h.val[3] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[1]), 1);

            // 计算 q3bytes.val 中的值
            q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[0], m3b)), vreinterpretq_s8_u8(q3h.val[0]));
            q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[1], m3b)), vreinterpretq_s8_u8(q3h.val[1]));
            q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 2), m3b)), vreinterpretq_s8_u8(q3h.val[2]));
            q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 2), m3b)), vreinterpretq_s8_u8(q3h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
            // 如果支持 ARM Dot Product 指令集，则使用 vdotq_s32 函数计算乘积并累加到 isum 中
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes_1.val[0])) * scale[0];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes_1.val[1])) * scale[1];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes_1.val[2])) * scale[2];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes_1.val[3])) * scale[3];
#else
            // 如果不支持 ARM Dot Product 指令集，则使用 vmull_s8 和 vaddq_s16 函数计算乘积并累加到 isum 中
            int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes_1.val[0])),
                                     vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes_1.val[0])));
            int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes_1.val[1])),
                                     vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes_1.val[1])));
            int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_1.val[2])),
                                     vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_1.val[2])));
            int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_1.val[3])),
                                     vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_1.val[3])));
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
#endif
#endif
            // 增加缩放值4
            scale += 4;

            // 计算 q3h.val[0]，使用 vbicq_u8 函数
            q3h.val[0] = vbicq_u8(m2, qhbits.val[0]);
            // 计算 q3h.val[1]，使用 vbicq_u8 函数
            q3h.val[1] = vbicq_u8(m2, qhbits.val[1]);
            // 计算 q3h.val[2]，使用 vbicq_u8 和 vshrq_n_u8 函数
            q3h.val[2] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[0]), 1);
            // 计算 q3h.val[3]，使用 vbicq_u8 和 vshrq_n_u8 函数
            q3h.val[3] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[1]), 1);

            // 计算 q3bytes.val[0]，使用 vsubq_s8、vreinterpretq_s8_u8 和 vandq_u8 函数
            q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 4), m3b)), vreinterpretq_s8_u8(q3h.val[0]));
            // 计算 q3bytes.val[1]，使用 vsubq_s8、vreinterpretq_s8_u8 和 vandq_u8 函数
            q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 4), m3b)), vreinterpretq_s8_u8(q3h.val[1]));
            // 计算 q3bytes.val[2]，使用 vsubq_s8、vreinterpretq_s8_u8 和 vandq_u8 函数
            q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 6), m3b)), vreinterpretq_s8_u8(q3h.val[2]));
            // 计算 q3bytes.val[3]，使用 vsubq_s8、vreinterpretq_s8_u8 和 vandq_u8 函数
            q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 6), m3b)), vreinterpretq_s8_u8(q3h.val[3]));

#if defined(__ARM_FEATURE_DOTPROD)
            // 计算 isum，使用 vaddvq_s32 和 vdotq_s32 函数
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes_2.val[0])) * scale[0];
            // 计算 isum，使用 vaddvq_s32 和 vdotq_s32 函数
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes_2.val[1])) * scale[1];
            // 计算 isum，使用 vaddvq_s32 和 vdotq_s32 函数
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes_2.val[2])) * scale[2];
            // 计算 isum，使用 vaddvq_s32 和 vdotq_s32 函数
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes_2.val[3])) * scale[3];
#else
            // 计算 p0，将两个 int8x8_t 类型的寄存器中的每个元素分别相乘，然后相加
            p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes_2.val[0])),
                           vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes_2.val[0]));
            // 计算 p1，将两个 int8x8_t 类型的寄存器中的每个元素分别相乘，然后相加
            p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes_2.val[1])),
                           vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes_2.val[1]));
            // 计算 p2，将两个 int8x8_t 类型的寄存器中的每个元素分别相乘，然后相加
            p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_2.val[2])),
                           vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_2.val[2]));
            // 计算 p3，将两个 int8x8_t 类型的寄存器中的每个元素分别相乘，然后相加
            p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_2.val[3])),
                           vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_2.val[3]));
            // 将 p0、p1、p2、p3 的每个元素相加，并乘以对应的 scale 值，然后累加到 isum 中
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
#endif
            // 将 scale 指针向后移动 4 个元素
            scale += 4;

            // 如果 j 等于 0，则将 qhbits 寄存器中的每个元素右移 4 位
            if (j == 0) {
                qhbits.val[0] = vshrq_n_u8(qhbits.val[0], 4);
                qhbits.val[1] = vshrq_n_u8(qhbits.val[1], 4);
            }

        }
        // 将 d 乘以 isum 的结果累加到 sum 中
        sum += d * isum;

    }

    // 将 sum 赋值给 s
    *s = sum;

#elif defined __AVX2__

    // 创建一个所有元素都为 3 的 __m256i 类型寄存器
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个所有元素都为 1 的 __m256i 类型寄存器
    const __m256i mone = _mm256_set1_epi8(1);
    // 创建一个所有元素都为 32 的 __m128i 类型寄存器
    const __m128i m32 = _mm_set1_epi8(32);

    // 创建一个所有元素都为 0 的 __m256 类型寄存器
    __m256 acc = _mm256_setzero_ps();

    // 创建一个包含 3 个元素的 uint32_t 数组

    // 将 acc 的值赋给 s

#elif defined __AVX__

    // 创建一个所有元素都为 3 的 __m128i 类型寄存器
    const __m128i m3 = _mm_set1_epi8(3);
    // 创建一个所有元素都为 1 的 __m128i 类型寄存器
    const __m128i mone = _mm_set1_epi8(1);
    // 创建一个所有元素都为 32 的 __m128i 类型寄存器
    const __m128i m32 = _mm_set1_epi8(32);
    // 创建一个所有元素都为 2 的 __m128i 类型寄存器
    const __m128i m2 = _mm_set1_epi8(2);

    // 创建一个所有元素都为 0 的 __m256 类型寄存器
    __m256 acc = _mm256_setzero_ps();

    // 创建一个指向 uint32_t 类型的指针

    // 将 acc 的值赋给 s

#elif defined __riscv_v_intrinsic

    // 创建一个包含 3 个元素的 uint32_t 数组
    // 创建一个包含 4 个元素的 uint32_t 数组

    // 将 sumf 的值赋给 s

#else
    // scalar version
    // This function is written like this so the compiler can manage to vectorize most of it
    // 使用 -Ofast，GCC 和 clang 能够生成的代码与手动矢量化版本相差不到 2 倍。
    // 我尝试的每个其他版本都至少运行 4 倍慢。
    // 理想的情况是，我们只需编写一次代码，编译器就会自动产生最佳的机器指令集，而不是我们必须手动为 AVX、ARM_NEON 等编写矢量化版本。
    
    int8_t  aux8[QK_K];  // 声明一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[8];     // 声明一个长度为 8 的 int16_t 数组 aux16
    float   sums [8];     // 声明一个长度为 8 的 float 数组 sums
    int32_t aux32[8];     // 声明一个长度为 8 的 int32_t 数组 aux32
    memset(sums, 0, 8*sizeof(float));  // 将 sums 数组的前 8 个元素初始化为 0
    
    uint32_t auxs[4];     // 声明一个长度为 4 的 uint32_t 数组 auxs
    const int8_t * scales = (const int8_t*)auxs;  // 声明一个指向 auxs 的 int8_t 指针 scales
    
    float sumf = 0;       // 声明一个 float 类型的变量 sumf，并初始化为 0
    // 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 的 qs 和 hmask，赋值给 q3 和 hm
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        // 获取 y[i] 的 qs，赋值给 q8
        const  int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组的前 8 个元素置为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        // 定义指向 aux8 数组的指针 a，并初始化 m 为 1
        int8_t * restrict a = aux8;
        uint8_t m = 1;
        // 循环，每次增加 128，j 从 0 到 QK_K-1
        for (int j = 0; j < QK_K; j += 128) {
            // 将 q3 中的每个元素与 3 进行按位与操作，赋值给 a 中的前 32 个元素
            for (int l = 0; l < 32; ++l) a[l] = q3[l] & 3;
            // 将 a 中的前 32 个元素减去（hm 中的元素与 m 进行按位与操作的结果是否为 0，如果是则减去 0，否则减去 4）
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            // a 指针向后移动 32 个位置，m 左移 1 位
            a += 32; m <<= 1;
            // ... 重复上述操作，直到 j 达到 QK_K
            // q3 指针向后移动 32 个位置
            q3 += 32;
        }
        // 将 a 指针重新指向 aux8
        a = aux8;

        // 将 x[i] 的 scales 数组的前 12 个元素拷贝到 auxs 数组
        memcpy(auxs, x[i].scales, 12);
        // 对 auxs 数组进行位操作
        uint32_t tmp = auxs[2];
        auxs[2] = ((auxs[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        auxs[3] = ((auxs[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        auxs[0] = (auxs[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        auxs[1] = (auxs[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);
        // 循环，j 从 0 到 QK_K/16-1
        for (int j = 0; j < QK_K/16; ++j) {
            // ... 重复一系列操作
            // q8 指针向后移动 8 个位置，a 指针向后移动 8 个位置
            q8 += 8; a += 8;
            // ... 重复一系列操作
            // q8 指针向后移动 8 个位置，a 指针向后移动 8 个位置
            q8 += 8; a += 8;
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 循环，l 从 0 到 7
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    // 循环，l 从 0 到 7
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#endif
}

#else

void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q3_K 和 block_q8_K 类型的指针
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 nb，即 n / QK_K
    const int nb = n / QK_K;

#ifdef __ARM_NEON

#ifdef __ARM_FEATURE_DOTPROD
    // 定义一个全为 0 的 int32x4_t 类型的变量 vzero
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 定义一个全为 0x3 的 uint8x16_t 类型的变量 m3b
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 定义一个全为 4 的 uint8x16_t 类型的变量 mh
    const uint8x16_t mh  = vdupq_n_u8(4);

    // 定义一个 ggml_int8x16x4_t 类型的变量 q3bytes
    ggml_int8x16x4_t q3bytes;

    // 定义一个长度为 2 的 uint16_t 类型的数组 aux16
    uint16_t aux16[2];
    // 将 aux16 强制转换为 int8_t 类型的指针，赋值给 scales
    int8_t * scales = (int8_t *)aux16;

    // 定义一个 float 类型的变量 sum，初始化为 0
    float sum = 0;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 定义一个 ggml_uint8x16x4_t 类型的变量 q3h
        ggml_uint8x16x4_t q3h;

        // 从 x[i].hmask 中加载 8 个 uint8_t 类型的数据，赋值给 hbits
        const uint8x8_t  hbits    = vld1_u8(x[i].hmask);
        // 从 x[i].qs 中加载 16 个 uint8_t 类型的数据，赋值给 q3bits
        const uint8x16_t q3bits   = vld1q_u8(x[i].qs);
        // 从 y[i].qs 中加载 16 个 int8_t 类型的数据，赋值给 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(y[i].qs);

        // 从 x[i].scales 中加载 16 位数据，分别存储到 aux16[0] 和 aux16[1] 中
        const uint16_t a = *(const uint16_t *)x[i].scales;
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        // 将 scales 数组中的每个元素减去 8
        for (int j = 0; j < 4; ++j) scales[j] -= 8;

        // 计算 isum
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        // 计算 d
        const float d = y[i].d * (float)x[i].d;

        // 对 hbits 进行位移和逻辑运算，得到 q3h
        const uint8x16_t htmp = vcombine_u8(hbits, vshr_n_u8(hbits, 1));
        q3h.val[0] = vandq_u8(mh, vshlq_n_u8(htmp, 2));
        q3h.val[1] = vandq_u8(mh, htmp);
        q3h.val[2] = vandq_u8(mh, vshrq_n_u8(htmp, 2));
        q3h.val[3] = vandq_u8(mh, vshrq_n_u8(htmp, 4));

        // 对 q3bits 和 q3h 进行位移和逻辑运算，得到 q3bytes
        q3bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q3bits, m3b),                q3h.val[0]));
        q3bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 2), m3b), q3h.val[1]));
        q3bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 4), m3b), q3h.val[2]));
        q3bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q3bits, 6),                q3h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM dot product 指令集
    isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes.val[0])) * scales[0];
    // 使用 ARM dot product 指令计算两个向量的点积，并将结果累加到 isum 中
    isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes.val[1])) * scales[2];
    // 使用 ARM dot product 指令计算两个向量的点积，并将结果累加到 isum 中
    isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes.val[2])) * scales[1];
    // 使用 ARM dot product 指令计算两个向量的点积，并将结果累加到 isum 中
    isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes.val[3])) * scales[3];
    // 使用 ARM dot product 指令计算两个向量的点积，并将结果累加到 isum 中
#else
    // 如果不支持 ARM dot product 指令集
    const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                   vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes.val[0])));
    // 使用 NEON 指令计算两个向量的乘积，并将结果累加到 p0 中
    const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                   vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes.val[1])));
    // 使用 NEON 指令计算两个向量的乘积，并将结果累加到 p1 中
    const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                   vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes.val[2])));
    // 使用 NEON 指令计算两个向量的乘积，并将结果累加到 p2 中
    const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                   vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes.val[3])));
    // 使用 NEON 指令计算两个向量的乘积，并将结果累加到 p3 中
    isum += vaddvq_s16(p0) * scales[0] + vaddvq_s16(p1) * scales[2] + vaddvq_s16(p2) * scales[1] + vaddvq_s16(p3) * scales[3];
    // 将 p0、p1、p2、p3 的累加结果乘以对应的比例因子后累加到 isum 中
#endif

    sum += d * isum;
    // 将 isum 乘以 d 后累加到 sum 中

}

*s = sum;
// 将 sum 的值赋给指针 s 指向的变量

#elif defined __AVX2__

const __m256i m3 = _mm256_set1_epi8(3);
const __m256i m1 = _mm256_set1_epi8(1);
// 设置 AVX2 寄存器 m3 和 m1 的值为 3 和 1

__m256 acc = _mm256_setzero_ps();
// 初始化 AVX2 寄存器 acc 为全零

uint64_t aux64;
// 定义一个 uint64_t 类型的变量 aux64

uint16_t aux16[2];
const int8_t * aux8 = (const int8_t *)aux16;
// 定义一个长度为 2 的 uint16_t 数组 aux16 和一个指向 aux16 的 int8_t 类型指针 aux8

}

*s = hsum_float_8(acc);
// 将 AVX2 寄存器 acc 中的值求和后赋给指针 s 指向的变量

#elif defined __AVX__

const __m128i m3 = _mm_set1_epi8(3);
const __m128i m1 = _mm_set1_epi8(1);
// 设置 AVX 寄存器 m3 和 m1 的值为 3 和 1

__m256 acc = _mm256_setzero_ps();
// 初始化 AVX 寄存器 acc 为全零

uint64_t aux64;
// 定义一个 uint64_t 类型的变量 aux64

uint16_t aux16[2];
const int8_t * aux8 = (const int8_t *)aux16;
// 定义一个长度为 2 的 uint16_t 数组 aux16 和一个指向 aux16 的 int8_t 类型指针 aux8

}
    # 计算浮点数数组的和
    s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    # 如果定义了 __riscv_v_intrinsic，则执行以下代码块

    uint16_t aux16[2];
    # 声明一个包含2个uint16_t类型元素的数组aux16
    int8_t * scales = (int8_t *)aux16;
    # 声明一个指向aux16的int8_t类型指针scales

    float sumf = 0;
    # 声明一个浮点数sumf并初始化为0

    }

    *s = sumf;
    # 将sumf的值赋给指针s所指向的变量

#else
    # 如果未定义 __riscv_v_intrinsic，则执行以下代码块

    int8_t  aux8[QK_K];
    # 声明一个包含QK_K个int8_t类型元素的数组aux8
    int16_t aux16[8];
    # 声明一个包含8个int16_t类型元素的数组aux16
    float   sums [8];
    # 声明一个包含8个float类型元素的数组sums
    int32_t aux32[8];
    # 声明一个包含8个int32_t类型元素的数组aux32
    int32_t scales[4];
    # 声明一个包含4个int32_t类型元素的数组scales
    memset(sums, 0, 8*sizeof(float));
    # 将sums数组的内容全部初始化为0

    float sumf = 0;
    # 声明一个浮点数sumf并初始化为0
    for (int i = 0; i < nb; ++i) {
        # 循环，i从0到nb-1，每次递增1
        const uint8_t * restrict q3 = x[i].qs;
        # 声明一个指向x[i].qs的常量uint8_t类型指针q3
        const uint8_t * restrict hm = x[i].hmask;
        # 声明一个指向x[i].hmask的常量uint8_t类型指针hm
        const  int8_t * restrict q8 = y[i].qs;
        # 声明一个指向y[i].qs的常量int8_t类型指针q8
        int8_t * restrict a = aux8;
        # 声明一个指向aux8的int8_t类型指针a
        for (int l = 0; l < 8; ++l) {
            # 循环，l从0到7，每次递增1
            a[l+ 0] = (int8_t)((q3[l+0] >> 0) & 3) - (hm[l] & 0x01 ? 0 : 4);
            # 计算a[l+0]的值
            a[l+ 8] = (int8_t)((q3[l+8] >> 0) & 3) - (hm[l] & 0x02 ? 0 : 4);
            # 计算a[l+8]的值
            # ...（后续类似）
        }

        scales[0] = (x[i].scales[0] & 0xF) - 8;
        # 计算scales[0]的值
        scales[1] = (x[i].scales[0] >>  4) - 8;
        # 计算scales[1]的值
        scales[2] = (x[i].scales[1] & 0xF) - 8;
        # 计算scales[2]的值
        scales[3] = (x[i].scales[1] >>  4) - 8;
        # 计算scales[3]的值

        memset(aux32, 0, 8*sizeof(int32_t));
        # 将aux32数组的内容全部初始化为0
        for (int j = 0; j < QK_K/16; ++j) {
            # 循环，j从0到QK_K/16-1，每次递增1
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算aux16数组的值
            # ...（后续类似）
        }
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        # 计算浮点数d的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        # 循环，将d乘以aux32数组的每个元素并加到sums数组的对应元素上
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 循环，将sums数组的每个元素加到sumf上
    *s = sumf;
    # 将sumf的值赋给指针s所指向的变量

#endif
}
#endif

#if QK_K == 256
    # 如果QK_K等于256，则执行以下代码块
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 确保 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    const block_q4_K * restrict x = vx;  // 将 vx 强制转换为 block_q4_K 类型的指针
    const block_q8_K * restrict y = vy;  // 将 vy 强制转换为 block_q8_K 类型的指针

    const int nb = n / QK_K;  // 计算 nb，即 n 除以 QK_K 的结果

    static const uint32_t kmask1 = 0x3f3f3f3f;  // 定义静态的掩码常量
    static const uint32_t kmask2 = 0x0f0f0f0f;  // 定义静态的掩码常量
    static const uint32_t kmask3 = 0x03030303;  // 定义静态的掩码常量

    uint32_t utmp[4];  // 定义一个长度为 4 的无符号整数数组

#ifdef __ARM_NEON

    const uint8x16_t m4b = vdupq_n_u8(0xf);  // 创建一个包含 16 个 0xf 的向量
#ifdef __ARM_FEATURE_DOTPROD
    const int32x4_t mzero = vdupq_n_s32(0);  // 创建一个包含 4 个 0 的向量
#endif

    ggml_int8x16x2_t q4bytes;  // 定义一个自定义类型的变量
    ggml_int8x16x2_t q8bytes;  // 定义一个自定义类型的变量

    float sumf = 0;  // 定义一个浮点数变量并初始化为 0

    for (int i = 0; i < nb; ++i) {  // 循环 nb 次

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);  // 计算 d 的值
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);  // 计算 dmin 的值

        const int16x8_t q8sums = vpaddq_s16(vld1q_s16(y[i].bsums), vld1q_s16(y[i].bsums + 8));  // 计算 q8sums 的值

        memcpy(utmp, x[i].scales, 12);  // 将 x[i].scales 的内容复制到 utmp 中

        uint32x2_t mins8 = { 0 };  // 创建一个长度为 2 的无符号整数向量并初始化为 0
        mins8 = vset_lane_u32(utmp[1] & kmask1, mins8, 0);  // 设置 mins8 的第一个元素
        mins8 = vset_lane_u32(((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4), mins8, 1);  // 设置 mins8 的第二个元素

        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);  // 更新 utmp[1] 的值
        utmp[0] &= kmask1;  // 更新 utmp[0] 的值

        const int16x8_t mins = vreinterpretq_s16_u16(vmovl_u8(vreinterpret_u8_u32(mins8)));  // 计算 mins 的值
        const int32x4_t prod = vaddq_s32(vmull_s16(vget_low_s16 (q8sums), vget_low_s16 (mins)),
                                         vmull_s16(vget_high_s16(q8sums), vget_high_s16(mins)));  // 计算 prod 的值
        sumf -= dmin * vaddvq_s32(prod);  // 更新 sumf 的值

        const uint8_t * scales = (const uint8_t *)utmp;  // 将 utmp 强制转换为 uint8_t 类型的指针

        const uint8_t * restrict q4 = x[i].qs;  // 将 x[i].qs 强制转换为 uint8_t 类型的指针
        const int8_t  * restrict q8 = y[i].qs;  // 将 y[i].qs 强制转换为 int8_t 类型的指针

        int32_t sumi1 = 0;  // 定义一个整数变量并初始化为 0
        int32_t sumi2 = 0;  // 定义一个整数变量并初始化为 0

        for (int j = 0; j < QK_K/64; ++j) {  // 循环 QK_K/64 次

            const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4); q4 += 32;  // 从 q4 中加载数据到 q4bits 中
#ifdef __ARM_FEATURE_DOTPROD
            // 从内存中加载 32 个有符号 8 位整数到两个寄存器中
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 将 32 个有符号 8 位整数按位与操作，并存储到两个寄存器中
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

            // 计算两个寄存器中的数据的点积，并将结果存储到 p1 中
            const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
            // 将 p1 中的数据求和，并乘以对应的比例因子，加到 sumi1 中
            sumi1 += vaddvq_s32(p1) * scales[2*j+0];

            // 从内存中加载 32 个有符号 8 位整数到两个寄存器中
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 将 32 个有符号 8 位整数右移 4 位，并存储到两个寄存器中
            q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
            q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

            // 计算两个寄存器中的数据的点积，并将结果存储到 p2 中
            const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);

            // 将 p2 中的数据求和，并乘以对应的比例因子，加到 sumi2 中
            sumi2 += vaddvq_s32(p2) * scales[2*j+1];
#endif
        # 如果不满足条件，执行以下代码块
        else
        {
            # 从指针 q8 指向的内存中加载 32 个有符号 8 位整数，存储到 q8bytes 中，并将 q8 指针向后移动 32 个字节
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            # 将 q4bits.val[0] 和 m4b 按位与，结果存储到 q4bytes.val[0] 中
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            # 将 q4bits.val[1] 和 m4b 按位与，结果存储到 q4bytes.val[1] 中
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
            # 计算 p0，将结果存储到 p0 中
            const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            # 计算 p1，将结果存储到 p1 中
            const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1])));
            # 计算 sumi1，将结果存储到 sumi1 中
            sumi1 += vaddvq_s16(vaddq_s16(p0, p1)) * scales[2*j+0];

            # 从指针 q8 指向的内存中加载 32 个有符号 8 位整数，存储到 q8bytes 中，并将 q8 指针向后移动 32 个字节
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            # 将 q4bits.val[0] 右移 4 位，结果存储到 q4bytes.val[0] 中
            q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
            # 将 q4bits.val[1] 右移 4 位，结果存储到 q4bytes.val[1] 中
            q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
            # 计算 p2，将结果存储到 p2 中
            const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            # 计算 p3，将结果存储到 p3 中
            const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1]));
            # 计算 sumi2，将结果存储到 sumi2 中
            sumi2 += vaddvq_s16(vaddq_s16(p2, p3)) * scales[2*j+1];
        }
        # 结束 if-else 语句

        # 计算 sumf，将结果存储到 sumf 中
        sumf += d * (sumi1 + sumi2);
    }
    # 结束 for 循环

    # 将 sumf 存储到指针 s 指向的内存中
    *s = sumf;

#elif defined __AVX2__

    # 创建一个 256 位全 1 的有符号 8 位整数向量 m4
    const __m256i m4 = _mm256_set1_epi8(0xF);

    # 创建一个全 0 的 256 位浮点数向量 acc
    __m256 acc = _mm256_setzero_ps();
    # 结束 if-else 语句

    # 将 acc_m 的高 128 位和低 128 位相加，结果存储到 acc_m 中
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    # 将 acc_m 的高 64 位和低 64 位相加，结果存储到 acc_m 中
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    # 将 acc_m 的 8 个单精度浮点数相加，返回结果并存储到指针 s 指向的内存中
    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

#elif defined __AVX__

    # 创建一个 128 位全 1 的有符号 8 位整数向量 m4
    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个 128 位全 1 的有符号 8 位整数向量 m2
    const __m128i m2 = _mm_set1_epi8(0x2);

    # 创建一个全 0 的 256 位浮点数向量 acc
    __m256 acc = _mm256_setzero_ps();
    # 将两个寄存器中的四个单精度浮点数相加，并将结果存储在第一个寄存器中
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    # 将寄存器中的两个单精度浮点数相加，并将结果存储在寄存器中
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));
    # 将寄存器中的单精度浮点数求和，并将结果存储在指针所指向的位置
    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);
#elif defined __riscv_v_intrinsic
    # 如果定义了 __riscv_v_intrinsic，则执行以下代码块
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 将 utmp 数组的第一个元素的地址转换为 uint8_t 类型指针，并赋值给 scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 将 utmp 数组的第三个元素的地址转换为 uint8_t 类型指针，并赋值给 mins
    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    }
    *s = sumf;
    # 将 sumf 的值赋给指针 s 所指向的变量
#else
    # 如果未定义 __riscv_v_intrinsic，则执行以下代码块
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 将 utmp 数组的第一个元素的地址转换为 uint8_t 类型指针，并赋值给 scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 将 utmp 数组的第三个元素的地址转换为 uint8_t 类型指针，并赋值给 mins
    int8_t  aux8[QK_K];
    # 声明一个长度为 QK_K 的 int8_t 类型数组 aux8
    int16_t aux16[8];
    # 声明一个长度为 8 的 int16_t 类型数组 aux16
    float   sums [8];
    # 声明一个长度为 8 的 float 类型数组 sums
    int32_t aux32[8];
    # 声明一个长度为 8 的 int32_t 类型数组 aux32
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组的内容全部初始化为 0
    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    // 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 的 qs 指针
        const uint8_t * restrict q4 = x[i].qs;
        // 获取 y[i] 的 qs 指针
        const  int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组的前 8 个元素置为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        // 获取 aux8 数组的指针
        int8_t * restrict a = aux8;
        // 遍历 QK_K/64 次，j 从 0 到 QK_K/64-1
        for (int j = 0; j < QK_K/64; ++j) {
            // 将 q4 数组中的前 32 个元素的低 4 位赋值给 a 数组
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l] & 0xF);
            a += 32;
            // 将 q4 数组中的前 32 个元素的高 4 位赋值给 a 数组
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l]  >> 4);
            a += 32; q4 += 32;
        }
        // 将 x[i] 的 scales 数组的前 12 个元素拷贝到 utmp 数组
        memcpy(utmp, x[i].scales, 12);
        // 对 utmp 数组进行位运算
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;

        // 初始化 sumi 为 0
        int sumi = 0;
        // 遍历 QK_K/16 次，j 从 0 到 QK_K/16-1
        for (int j = 0; j < QK_K/16; ++j) sumi += y[i].bsums[j] * mins[j/2];
        // 重新获取 aux8 数组的指针
        a = aux8;
        // 初始化 is 为 0
        int is = 0;
        // 遍历 QK_K/32 次，j 从 0 到 QK_K/32-1
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取 scales 数组中的一个元素
            int32_t scale = scales[is++];
            // 计算并累加 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            // 计算并累加 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            // 计算并累加 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            // 计算并累加 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 更新 sums 数组的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        // 计算 dmin 的值
        const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;
        // 更新 sumf 的值
        sumf -= dmin * sumi;
    }
    // 遍历结束后，更新 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
#endif
}
#else
# 计算两个向量的点积
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    # 将 vx 和 vy 强制转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    # 计算 nb，即 n / QK_K
    const int nb = n / QK_K;

    #ifdef __ARM_NEON

    # 创建一个包含 16 个 0xf 的 uint8x16_t 类型的向量
    const uint8x16_t m4b = vdupq_n_u8(0xf);

    #ifdef __ARM_FEATURE_DOTPROD
    # 创建一个包含 0 的 int32x4_t 类型的向量
    const int32x4_t mzero = vdupq_n_s32(0);
    #endif

    # 初始化 sumf 为 0
    float sumf = 0;

    # 定义 ggml_int8x16x2_t 和 ggml_int8x16x4_t 类型的变量
    ggml_int8x16x2_t q4bytes;
    ggml_int8x16x4_t q8bytes;

    # 初始化 sum_mins 为 0
    float sum_mins = 0.f;

    # 定义一个包含 2 个 uint16_t 类型元素的数组
    uint16_t aux16[2];
    # 将 aux16 强制转换为 uint8_t 类型的指针，并赋值给 scales
    const uint8_t * restrict scales = (const uint8_t *)aux16;

    # 遍历 nb
    for (int i = 0; i < nb; ++i) {

        # 将 x[i].qs 和 y[i].qs 强制转换为 uint8_t 和 int8_t 类型的指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        # 将 x[i].scales 的值按位与 0x0f0f，分别赋值给 aux16[0] 和 aux16[1]
        const uint16_t * restrict a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        # 计算 summi
        const int32_t summi = scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]);
        sum_mins += y[i].d * (float)x[i].d[1] * summi;

        # 计算 d
        const float d = y[i].d * (float)x[i].d[0];

        # 从 q4 中加载 32 个 uint8_t 类型的数据，存储到 q4bits
        const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4);

        #ifdef __ARM_FEATURE_DOTPROD
        # 从 q8 中加载 64 个 int8_t 类型的数据，存储到 q8bytes
        q8bytes = ggml_vld1q_s8_x4(q8);
        # 将 q4bits.val[0] 和 q4bits.val[1] 按位与 m4b，存储到 q4bytes.val[0] 和 q4bytes.val[1]
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

        # 计算 p1
        const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
        const int32_t sumi1 = vaddvq_s32(p1) * scales[0];

        # 将 q4bits.val[0] 和 q4bits.val[1] 右移 4 位，存储到 q4bytes.val[0] 和 q4bytes.val[1]
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

        # 计算 p2
        const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[2]), q4bytes.val[1], q8bytes.val[3]);
        const int32_t sumi2 = vaddvq_s32(p2) * scales[1];
#else
        // 从输入的q8中加载8个有符号字节，存储到q8bytes中
        q8bytes = ggml_vld1q_s8_x4(q8);
        // 将q4bits中的每个字节与m4b进行按位与操作，结果存储到q4bytes中
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
        // 计算p0和p1，分别为q4bytes和q8bytes对应位置字节的乘积之和
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1])));
        // 计算sumi1，为p0和p1的和乘以scales[0]
        int32_t sumi1 = vaddvq_s16(vaddq_s16(p0, p1)) * scales[0];

        // 将q4bits中的每个字节右移4位，结果存储到q4bytes中
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
        // 计算p2和p3，分别为q4bytes和q8bytes对应位置字节的乘积之和
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[2])));
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[3])));
        // 计算sumi2，为p2和p3的和乘以scales[1]
        int32_t sumi2 = vaddvq_s16(vaddq_s16(p2, p3)) * scales[1];

#endif
        // 计算sumf，为d乘以(sumi1 + sumi2)的和
        sumf += d * (sumi1 + sumi2);

    }

    // 将sumf减去sum_mins，存储到s中
    *s = sumf - sum_mins;

#elif defined __AVX2__

    // 创建一个包含16个元素的__m256i类型的向量m4，每个元素的值都为0xF
    const __m256i m4 = _mm256_set1_epi8(0xF);

    // 创建一个初始值为0的__m256类型的向量acc
    __m256 acc = _mm256_setzero_ps();

    // 初始化summs为0
    float summs = 0;

    // 创建一个包含2个元素的uint16_t类型的数组aux16
    uint16_t aux16[2];
    // 将aux16强制转换为指向常量uint8_t类型的指针，存储到scales中
    const uint8_t * scales = (const uint8_t *)aux16;
    # 遍历循环，从 i=0 开始，直到 i<nb 为止
    for (int i = 0; i < nb; ++i) {

        # 计算 x[i].d[0] 转换为 float 后乘以 y[i].d 的结果，存储在 d 中
        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        # 计算 x[i].d[1] 转换为 float 后乘以 y[i].d 的结果，存储在 m 中
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        # 创建一个包含 d 的 __m256 向量
        const __m256 vd = _mm256_set1_ps(d);

        # 将 x[i].scales 强制转换为 uint16_t 指针，取出其中的值进行位运算，并存储在 aux16 数组中
        const uint16_t * a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        # 计算 summs 的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        # 将 x[i].qs 强制转换为 uint8_t 指针，存储在 q4 中；将 y[i].qs 强制转换为 int8_t 指针，存储在 q8 中
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        # 加载 q4 中的数据到 q4bits 中
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        # 对 q4bits 进行位与运算，存储在 q4l 中；对 q4bits 右移 4 位后进行位与运算，存储在 q4h 中
        const __m256i q4l = _mm256_and_si256(q4bits, m4);
        const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

        # 加载 q8 中的数据到 q8l 和 q8h 中
        const __m256i q8l = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8h = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 对 q4l 和 q8l 进行乘法累加运算，存储在 p16l 中；对 q4h 和 q8h 进行乘法累加运算，存储在 p16h 中
        const __m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
        const __m256i p16h = _mm256_maddubs_epi16(q4h, q8h);

        # 对 p16l 进行有符号 16 位整数乘法累加运算，再乘以 scales[0]，存储在 p32l 中；将结果转换为 float 后与 vd 进行 FMA 运算，存储在 acc 中
        const __m256i p32l = _mm256_madd_epi16(_mm256_set1_epi16(scales[0]), p16l);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32l), acc);

        # 对 p16h 进行有符号 16 位整数乘法累加运算，再乘以 scales[1]，存储在 p32h 中；将结果转换为 float 后与 vd 进行 FMA 运算，存储在 acc 中
        const __m256i p32h = _mm256_madd_epi16(_mm256_set1_epi16(scales[1]), p16h);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32h), acc);

    }

    # 计算最终结果并存储在 s 中
    *s = hsum_float_8(acc) - summs;
#elif defined __AVX__
    # 如果定义了 AVX，则执行以下代码

    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个包含 0xF 的 16 个字节的常量 __m128i 类型变量 m4

    __m256 acc = _mm256_setzero_ps();
    # 创建一个包含全零的 8 个单精度浮点数的 __m256 类型变量 acc

    float summs = 0;
    # 创建一个单精度浮点数变量 summs，并初始化为 0

    uint16_t aux16[2];
    # 创建一个包含 2 个 16 位无符号整数的数组 aux16

    const uint8_t * scales = (const uint8_t *)aux16;
    # 创建一个指向 aux16 数组的常量指针 scales

    }

    *s = hsum_float_8(acc) - summs;
    # 将 acc 中的 8 个单精度浮点数求和后减去 summs，并将结果赋值给指针变量 s

#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则执行以下代码

    uint16_t s16[2];
    # 创建一个包含 2 个 16 位无符号整数的数组 s16

    const uint8_t * restrict scales = (const uint8_t *)s16;
    # 创建一个指向 s16 数组的常量指针 restrict scales

    float sumf = 0;
    # 创建一个单精度浮点数变量 sumf，并初始化为 0

    for (int i = 0; i < nb; ++i) {
        # 循环遍历 i 从 0 到 nb-1

        const uint8_t * restrict q4 = x[i].qs;
        # 创建一个指向 x[i].qs 的常量指针 restrict q4

        const  int8_t * restrict q8 = y[i].qs;
        # 创建一个指向 y[i].qs 的常量指针 restrict q8

        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        # 创建一个指向 x[i].scales 的常量指针 restrict b，并将其转换为 uint16_t 类型指针

        s16[0] = b[0] & 0x0f0f;
        s16[1] = (b[0] >> 4) & 0x0f0f;
        # 将 b[0] 的低 8 位和高 8 位分别与 0x0f0f 进行按位与操作，并将结果分别赋值给 s16 数组的两个元素

        sumf -= y[i].d * GGML_FP16_TO_FP32(x[i].d[1]) * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));
        # 计算 sumf 的值

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d[0]);
        # 创建一个单精度浮点数变量 d，并赋值为 y[i].d 乘以 GGML_FP16_TO_FP32(x[i].d[0])

        size_t vl = 32;
        # 创建一个大小为 32 的 size_t 类型变量 vl

        vint16m1_t vzero = __riscv_vmv_v_x_i16m1(0, 1);
        # 创建一个包含 0 和 1 的 vint16m1_t 类型变量 vzero

        // load Q4
        # 加载 Q4

        // load Q8 and multiply it with lower Q4 nibble
        # 加载 Q8 并将其与较低的 Q4 字节进行乘法运算

        // load Q8 and multiply it with upper Q4 nibble
        # 加载 Q8 并将其与较高的 Q4 字节进行乘法运算

    }

    *s = sumf;
    # 将 sumf 的值赋值给指针变量 s

#else
    # 如果不满足以上条件，则执行以下代码

    uint8_t aux8[QK_K];
    # 创建一个包含 QK_K 个 8 位无符号整数的数组 aux8

    int16_t aux16[16];
    # 创建一个包含 16 个 16 位有符号整数的数组 aux16

    float   sums [8];
    # 创建一个包含 8 个单精度浮点数的数组 sums

    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组中的所有元素初始化为 0

    uint16_t s16[2];
    # 创建一个包含 2 个 16 位无符号整数的数组 s16
    // 将指针 s16 强制转换为 const uint8_t 类型的指针，并赋给 scales
    const uint8_t * restrict scales = (const uint8_t *)s16;

    // 初始化浮点数 sumf 为 0
    float sumf = 0;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 将指针 x[i].qs 强制转换为 const uint8_t 类型的指针，并赋给 q4
        const uint8_t * restrict q4 = x[i].qs;
        // 将指针 y[i].qs 强制转换为 const int8_t 类型的指针，并赋给 q8
        const int8_t * restrict q8 = y[i].qs;
        // 将指针 aux8 赋给 a
        uint8_t * restrict a = aux8;
        // 将 q4 中的每个字节的低 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+ 0] = q4[l] & 0xF;
        // 将 q4 中的每个字节的高 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+32] = q4[l]  >> 4;

        // 将指针 x[i].scales 强制转换为 const uint16_t 类型的指针，并赋给 b
        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        // 将 b[0] 的低 8 位存入 s16[0]，高 8 位存入 s16[1]
        s16[0] = b[0] & 0x0f0f;
        s16[1] = (b[0] >> 4) & 0x0f0f;

        // 计算 sumf 的值
        sumf -= y[i].d * GGML_FP16_TO_FP32(x[i].d[1]) * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 计算 d 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d[0]);

        // 遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算 aux16 中的值
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            q8 += 16; a += 16;
            for (int l = 0; l < 16; ++l) aux16[l] += q8[l] * a[l];
            q8 += 16; a += 16;
            // 计算 dl 的值
            const float dl = d * scales[j];
            // 更新 sums 中的值
            for (int l = 0; l < 8; ++l) sums[l] += dl * (aux16[l] + aux16[l+8]);
        }
    }
    // 更新 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 计算两个向量的点积，其中向量长度为 256
void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 256 的整数倍
    assert(n % QK_K == 0);

    // 将输入指针转换为指向 block_q5_K 类型的指针
    const block_q5_K * restrict x = vx;
    // 将输入指针转换为指向 block_q8_K 类型的指针
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以 256 的商
    const int nb = n / QK_K;

    // 定义 4 个 32 位无符号整数
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    // 定义 4 个 32 位无符号整数数组
    uint32_t utmp[4];


#ifdef __ARM_NEON

    // 定义 16 个 8 位无符号整数的向量，每个元素都是 0xf
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    // 定义 16 个 8 位无符号整数的向量，每个元素都是 1
    const uint8x16_t mone = vdupq_n_u8(1);
    // 定义 16 个 8 位无符号整数的向量，每个元素都是 2
    const uint8x16_t mtwo = vdupq_n_u8(2);
#if defined(__ARM_FEATURE_DOTPROD)
    // 定义 4 个 32 位有符号整数的向量，每个元素都是 0
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 定义包含 4 个 16 位有符号整数向量的结构体
    ggml_int8x16x4_t q5bytes;

    // 定义浮点数变量 sumf，用于存储累加结果
    float sumf = 0;

#if defined(__ARM_FEATURE_DOTPROD)
    // 计算两个向量的点积，并将结果累加到 sumi 中
    sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]), q5bytes.val[1], q8bytes.val[1])) * *scales++;
    // 计算两个向量的点积，并将结果累加到 sumi 中
    sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]), q5bytes.val[3], q8bytes.val[3])) * *scales++;
        // 如果条件不成立，则执行以下代码块
        else
        {
            // 计算两个 int8x8_t 向量的乘积，并将结果相加
            const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            // 计算两个 int8x8_t 向量的乘积，并将结果相加
            const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1]));
            // 将两个 int16x8_t 向量的结果相加，并将结果乘以指针所指的值，然后加到 sumi 上
            sumi += vaddvq_s16(vaddq_s16(p0, p1)) * *scales++;

            // 计算两个 int8x8_t 向量的乘积，并将结果相加
            const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                           vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2])));
            // 计算两个 int8x8_t 向量的乘积，并将结果相加
            const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                           vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3]));
            // 将两个 int16x8_t 向量的结果相加，并将结果乘以指针所指的值，然后加到 sumi 上
            sumi += vaddvq_s16(vaddq_s16(p2, p3)) * *scales++;
        }

        // 将 d 乘以 sumi 减去 dmin 乘以 sumi_mins 的结果加到 sumf 上
        sumf += d * sumi - dmin * sumi_mins;
    }

    // 将 sumf 的值赋给指针所指的位置
    *s = sumf;

#elif defined __AVX2__

    // 创建一个包含 16 个 0xF 的 __m256i 向量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 创建一个全为 0 的 __m128i 向量
    const __m128i mzero = _mm_setzero_si128();
    // 创建一个包含 32 个 1 的 __m256i 向量
    const __m256i mone  = _mm256_set1_epi8(1);

    // 创建一个全为 0 的 __m256 向量
    __m256 acc = _mm256_setzero_ps();

    // 创建一个浮点数 summs，并初始化为 0
    float summs = 0.f;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 限制指针 q5 指向的内存区域，使其只能被修改
        const uint8_t * restrict q5 = x[i].qs;
        // 限制指针 q8 指向的内存区域，使其只能被修改
        const int8_t  * restrict q8 = y[i].qs;

        // 如果 QK_K 等于 256，则执行以下代码块
#if QK_K == 256
        // 计算 d 和 dmin 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 将 x[i].scales 的前 12 个字节拷贝到 utmp 中
        memcpy(utmp, x[i].scales, 12);
        // 对 utmp 数组进行位运算
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        // TODO
        const float d = 0, dmin = 0;
    }
    # 计算浮点数数组 acc 的和，并加上 summs 的值，然后赋给变量 s
    *s = hsum_float_8(acc) + summs;
#elif defined __AVX__
    # 如果定义了 AVX，则使用 AVX 指令集

    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个包含 16 个 8 位整数的常量向量，每个元素都是 0xF
    const __m128i mzero = _mm_setzero_si128();
    # 创建一个全零的 128 位整数向量
    const __m128i mone  = _mm_set1_epi8(1);
    # 创建一个包含 16 个 8 位整数的常量向量，每个元素都是 1
    const __m128i m2 = _mm_set1_epi8(2);
    # 创建一个包含 16 个 8 位整数的常量向量，每个元素都是 2

    __m256 acc = _mm256_setzero_ps();
    # 创建一个全零的 256 位单精度浮点数向量
    float summs = 0.f;
    # 初始化一个浮点数 summs 为 0

    }

    *s = hsum_float_8(acc) + summs;
    # 将 acc 向量中的 8 个单精度浮点数相加，并加上 summs，结果赋值给 s

#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则使用 riscv_v_intrinsic 指令集

    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 创建一个指向 utmp[0] 的 uint8_t 类型指针
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 创建一个指向 utmp[2] 的 uint8_t 类型指针

    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    float sums = 0.0;
    # 初始化一个浮点数 sums 为 0.0

    size_t vl;
    # 声明一个 size_t 类型的变量 vl

    }

    *s = sumf+sums;
    # 将 sumf 和 sums 相加，结果赋值给 s

#else
    # 如果不满足以上条件

    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 创建一个指向 utmp[0] 的 uint8_t 类型指针
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 创建一个指向 utmp[2] 的 uint8_t 类型指针

    int8_t  aux8[QK_K];
    # 创建一个长度为 QK_K 的 int8_t 类型数组 aux8
    int16_t aux16[8];
    # 创建一个长度为 8 的 int16_t 类型数组 aux16
    float   sums [8];
    # 创建一个长度为 8 的 float 类型数组 sums
    int32_t aux32[8];
    # 创建一个长度为 8 的 int32_t 类型数组 aux32
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组中的元素全部初始化为 0

    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 遍历 sums 数组，将每个元素累加到 sumf
    *s = sumf;
    # 将 sumf 的值赋给 s
#endif
}
#else
# 如果不满足以上条件

void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 定义一个函数 ggml_vec_dot_q5_K_q8_K，接受参数 n, s, vx, vy

    assert(n % QK_K == 0);
    # 断言 n 能被 QK_K 整除

    const block_q5_K * restrict x = vx;
    # 创建一个指向 vx 的 block_q5_K 类型指针 x
    const block_q8_K * restrict y = vy;
    # 创建一个指向 vy 的 block_q8_K 类型指针 y

    const int nb = n / QK_K;
    # 计算 n 除以 QK_K 的商，赋值给 nb

#ifdef __ARM_NEON
    # 如果定义了 ARM_NEON，则使用 ARM_NEON 指令集

    const uint8x16_t m4b = vdupq_n_u8(0xf);
    # 创建一个包含 16 个 8 位无符号整数的常量向量，每个元素都是 0xf
    const uint8x16_t mh = vdupq_n_u8(16);
    # 创建一个包含 16 个 8 位无符号整数的常量向量，每个元素都是 16
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t mzero = vdupq_n_s32(0);
    # 创建一个包含 4 个 32 位有符号整数的常量向量，每个元素都是 0
#endif

    ggml_int8x16x4_t q5bytes;
    # 创建一个包含 4 个 16 位有符号整数的结构体
    ggml_uint8x16x4_t q5h;
    # 创建一个包含 4 个 16 位无符号整数的结构体

    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * (float)x[i].d;
        // 获取 x[i].scales 的指针
        const int8_t * sc = x[i].scales;

        // 获取 x[i].qs 的指针，并使用 restrict 限定指针的唯一性
        const uint8_t * restrict q5 = x[i].qs;
        // 获取 x[i].qh 的指针，并使用 restrict 限定指针的唯一性
        const uint8_t * restrict qh = x[i].qh;
        // 获取 y[i].qs 的指针，并使用 restrict 限定指针的唯一性
        const int8_t  * restrict q8 = y[i].qs;

        // 从 qh 中加载 8 个字节到 uint8x8_t 类型的变量 qhbits
        const uint8x8_t qhbits = vld1_u8(qh);

        // 从 q5 中加载 16 个字节到 ggml_uint8x16x2_t 类型的变量 q5bits
        const ggml_uint8x16x2_t q5bits = ggml_vld1q_u8_x2(q5);
        // 从 q8 中加载 16 个字节到 ggml_int8x16x4_t 类型的变量 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 将 qhbits 右移 1 位后与自身合并成 16 个字节的变量 htmp
        const uint8x16_t htmp = vcombine_u8(qhbits, vshr_n_u8(qhbits, 1));
        // 计算 q5h.val[0] 到 q5h.val[3] 的值
        q5h.val[0] = vbicq_u8(mh, vshlq_n_u8(htmp, 4));
        q5h.val[1] = vbicq_u8(mh, vshlq_n_u8(htmp, 2));
        q5h.val[2] = vbicq_u8(mh, htmp);
        q5h.val[3] = vbicq_u8(mh, vshrq_n_u8(htmp, 2));

        // 计算 q5bytes.val[0] 到 q5bytes.val[3] 的值
        q5bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[0], m4b)), vreinterpretq_s8_u8(q5h.val[0]));
        q5bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[1], m4b)), vreinterpretq_s8_u8(q5h.val[1]));
        q5bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[0], 4)), vreinterpretq_s8_u8(q5h.val[2]));
        q5bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[1], 4)), vreinterpretq_s8_u8(q5h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
// 如果定义了 ARM 特性 DOTPROD，则执行以下代码块

        int32_t sumi1 = sc[0] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]));
        // 计算第一个元素的乘积和加和
        int32_t sumi2 = sc[1] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[1], q8bytes.val[1]));
        // 计算第二个元素的乘积和加和
        int32_t sumi3 = sc[2] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]));
        // 计算第三个元素的乘积和加和
        int32_t sumi4 = sc[3] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[3], q8bytes.val[3]));
        // 计算第四个元素的乘积和加和

        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);
        // 将每个元素的乘积和加和相加，并乘以 d，加到 sumf 上

#else
// 如果未定义 ARM 特性 DOTPROD，则执行以下代码块

        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        // 计算第一个元素的乘积和加和
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1]));
        // 计算第二个元素的乘积和加和
        int32_t sumi = sc[0] * vaddvq_s16(p0) + sc[1] * vaddvq_s16(p1);
        // 将第一个和第二个元素的乘积和加和相加，并乘以 sc[0] 和 sc[1]，赋值给 sumi

        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2]));
        // 计算第三个元素的乘积和加和
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        // 计算第四个元素的乘积和加和
        sumi += sc[2] * vaddvq_s16(p2) + sc[3] * vaddvq_s16(p3);
        // 将第三个和第四个元素的乘积和加和相加，并乘以 sc[2] 和 sc[3]，加到 sumi 上

        sumf += d*sumi;
        // 将 sumi 乘以 d，加到 sumf 上
#endif

    }

    *s = sumf;
    // 将 sumf 的值赋给指针 s

#elif defined __AVX2__
// 如果定义了 AVX2 特性，则执行以下代码块

    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 创建一个包含 0xF 的 256 位整数常量
    const __m256i mone  = _mm256_set1_epi8(1);
    // 创建一个包含 1 的 256 位整数常量

    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含 0 的 256 位单精度浮点数常量
    # 遍历循环，从0到nb-1
    for (int i = 0; i < nb; ++i) {

        # 获取x[i]的qs指针，限制指针对应的内存区域
        const uint8_t * restrict q5 = x[i].qs;
        # 获取y[i]的qs指针，限制指针对应的内存区域
        const int8_t  * restrict q8 = y[i].qs;

        # 计算d值，y[i].d乘以x[i].d的GGML_FP16_TO_FP32值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        # 从q5中加载256位数据到q5bits
        const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);

        # 设置256位整数scale_l和scale_h
        const __m256i scale_l = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[1]), _mm_set1_epi16(x[i].scales[0]));
        const __m256i scale_h = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[3]), _mm_set1_epi16(x[i].scales[2]));

        # 定义int64_t类型的aux64，从x[i].qh中复制8个字节到aux64，然后转换为__m128i类型的haux128，再转换为__m256i类型的haux256
        int64_t aux64;
        memcpy(&aux64, x[i].qh, 8);
        const __m128i haux128 = _mm_set_epi64x(aux64 >> 1, aux64);
        const __m256i haux256 = MM256_SET_M128I(_mm_srli_epi16(haux128, 2), haux128);

        # 计算q5h_0和q5h_1
        const __m256i q5h_0 = _mm256_slli_epi16(_mm256_andnot_si256(haux256, mone), 4);
        const __m256i q5h_1 = _mm256_slli_epi16(_mm256_andnot_si256(_mm256_srli_epi16(haux256, 4), mone), 4);

        # 计算q5l_0和q5l_1
        const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);
        const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);

        # 从q8中加载256位数据到q8_0和q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 计算p16_0、p16_1、s16_0、s16_1
        const __m256i p16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5l_0, q8_0));
        const __m256i p16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5l_1, q8_1));
        const __m256i s16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5h_0, q8_0));
        const __m256i s16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5h_1, q8_1));

        # 计算dot
        const __m256i dot = _mm256_sub_epi32(_mm256_add_epi32(p16_0, p16_1), _mm256_add_epi32(s16_0, s16_1));

        # 使用_mm256_fmadd_ps计算acc
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(dot), acc);

    }

    # 将acc的值求和并存储到*s中
    *s = hsum_float_8(acc);
#elif defined __AVX__

    // 设置一个包含8个0xF的128位整数寄存器
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 设置一个包含8个1的128位整数寄存器
    const __m128i mone  = _mm_set1_epi8(1);
    // 设置一个全0的256位浮点数寄存器
    __m256 acc = _mm256_setzero_ps();

    }

    // 计算256位浮点数寄存器中所有元素的和
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic

    // 初始化一个浮点数变量为0
    float sumf = 0;

    }

    // 将计算结果赋值给指定的变量
    *s = sumf;

#else

    // 定义一些变量和数组
    int8_t aux8[QK_K];
    int16_t aux16[16];
    float   sums [8];
    // 将数组初始化为0
    memset(sums, 0, 8*sizeof(float));

    // 初始化一个浮点数变量为0
    float sumf = 0;
    // 循环计算
    for (int i = 0; i < nb; ++i) {
        // ...
    }
    // 将计算结果赋值给指定的变量
    *s = sumf;
#endif
}
#endif


#if QK_K == 256
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言，确保n是QK_K的整数倍
    assert(n % QK_K == 0);

    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 初始化一个浮点数变量为0
    float sum = 0;

    // ...
#if defined(__ARM_FEATURE_DOTPROD)
    # 如果定义了 ARM 的 DOTPROD 特性，则执行以下代码块
    isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
    // 使用 SIMD 指令计算四个向量的点积，并将结果与对应的比例相乘，然后累加到 isum 上
    scale += 4;
    // 指针 scale 向后移动 4 个位置
#else
    // 如果未定义 ARM 的 DOTPROD 特性，则执行以下代码块
    int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                             vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
    // 计算两个向量的低位和高位的乘积，并将结果相加
    int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                             vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1]));
    // 计算两个向量的低位和高位的乘积，并将结果相加
    isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
    // 将两个乘积的和与对应的比例相乘，然后累加到 isum 上
    scale += 2;
    // 指针 scale 向后移动 2 个位置

    int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                             vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2]));
    // 计算两个向量的低位和高位的乘积，并将结果相加
    int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                             vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3]));
    // 计算两个向量的低位和高位的乘积，并将结果相加
    isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
    // 将两个乘积的和与对应的比例相乘，然后累加到 isum 上
    scale += 2;
    // 指针 scale 向后移动 2 个位置
#endif
// 读取 q8 寄存器中的 64 个字节数据
q8bytes = ggml_vld1q_s8_x4(q8); q8 += 64;

// 对 qhbits 寄存器中的值进行位移和位与操作，得到 q6h 寄存器的值
shifted = vshrq_n_u8(qhbits.val[0], 4);
q6h.val[0] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
shifted = vshrq_n_u8(qhbits.val[1], 4);
q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
shifted = vshrq_n_u8(qhbits.val[0], 6);
q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
shifted = vshrq_n_u8(qhbits.val[1], 6);
q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

// 对 q6bits 寄存器和 q6h 寄存器进行位移、位或和类型转换操作，得到 q6bytes 寄存器的值
q6bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[0]));
q6bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[1]));
q6bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[2], 4), q6h.val[2]));
q6bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[3], 4), q6h.val[3]);
#if defined(__ARM_FEATURE_DOTPROD)
// 如果定义了 ARM 特性 DOTPROD，则执行以下代码块
            // 计算四个向量的点积，并分别乘以对应的比例因子，然后求和
            isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
            scale += 4;

            // 注释掉的循环代码块
            //for (int l = 0; l < 4; ++l) {
            //    const int32x4_t p = vdotq_s32(vzero, q6bytes.val[l], q8bytes.val[l]);
            //    isum += vaddvq_s32(p) * *scale++;
            //}
#else
// 如果未定义 ARM 特性 DOTPROD，则执行以下代码块
            // 计算两个向量的点积，并分别乘以对应的比例因子，然后求和
            p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                    vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                    vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1])));
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
            scale += 2;

            // 计算两个向量的点积，并分别乘以对应的比例因子，然后求和
            p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                    vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2]));
            p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                    vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3]));
            isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
            scale += 2;
#endif
        }
        //sum += isum * d_all * y[i].d;
        // 计算最终的 sum，并将结果存储到 s 指针指向的位置
        sum += d_all * y[i].d * (isum - 32 * isum_mins);

    }
    *s = sum;

#elif defined __AVX2__
// 如果定义了 AVX2 特性，则执行以下代码块
    // 定义一些 AVX2 所需的常量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i m2 = _mm256_set1_epi8(3);
    const __m256i m32s = _mm256_set1_epi8(32);

    // 初始化一个全零的 AVX 寄存器
    __m256 acc = _mm256_setzero_ps();
    # 结束当前的代码块
    }

    # 将浮点数累加和计算结果赋值给指针变量 s
    *s = hsum_float_8(acc);
#elif defined __AVX__
    # 如果定义了 AVX，则设置一些常量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m3 = _mm_set1_epi8(3);
    const __m128i m32s = _mm_set1_epi8(32);
    const __m128i m2 = _mm_set1_epi8(2);
    # 初始化一个 256 位的零向量
    __m256 acc = _mm256_setzero_ps();
    }
    # 调用 hsum_float_8 函数，将结果赋值给指针 s
    *s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则初始化一个浮点数 sumf 为 0
    float sumf = 0;
    }
    # 将 sumf 的值赋给指针 s
    *s = sumf;
#else
    # 如果以上条件都不满足，则进行以下操作
    # 初始化一些数组和变量
    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    # 将 sums 数组的值全部初始化为 0
    memset(sums, 0, 8*sizeof(float));
    # 初始化一个浮点数 sumf 为 0
    float sumf = 0;
    # 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        # 一系列数组的赋值和计算操作
        # ...
    }
    # 将 sums 数组中的值累加到 sumf 中
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 将 sumf 的值赋给指针 s
    *s = sumf;
#endif
}
#else
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 确保 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q6_K 和 block_q8_K 类型的指针
    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 nb，即 n / QK_K
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 初始化 sum 为 0
    float sum = 0;

    // 初始化 m4b 为 0xF 的 16 个副本
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    // 初始化 m32s 为 32 的 16 个副本
    const int8x16_t  m32s = vdupq_n_s8(32);
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM Dot Product 指令集，则初始化 vzero 为 0 的 4 个副本
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 初始化 mone 为 3 的 16 个副本
    const uint8x16_t mone = vdupq_n_u8(3);

    // 定义 ggml_int8x16x4_t 和 ggml_uint8x16x4_t 类型的变量
    ggml_int8x16x4_t q6bytes;
    ggml_uint8x16x4_t q6h;

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i].d 的值，并转换为 float 类型
        const float d_all = (float)x[i].d;

        // 获取 x[i].ql, x[i].qh, y[i].qs, x[i].scales 的指针
        const uint8_t * restrict q6 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const int8_t  * restrict q8 = y[i].qs;
        const int8_t * restrict scale = x[i].scales;

        // 初始化 isum 为 0
        int32_t isum = 0;

        // 从内存中加载 qhbits, q6bits, q8bytes 的值
        uint8x16_t qhbits = vld1q_u8(qh);
        ggml_uint8x16x2_t q6bits = ggml_vld1q_u8_x2(q6);
        ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 qhbits 进行位运算和移位操作，得到 q6h.val[0] 到 q6h.val[3] 的值
        q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits), 4);
        uint8x16_t shifted = vshrq_n_u8(qhbits, 2);
        q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 4);
        q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 6);
        q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

        // 对 q6bits 进行位运算和移位操作，得到 q6bytes.val[0] 到 q6bytes.val[3] 的值
        q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);
        q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);
        q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[2])), m32s);
        q6bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[3])), m32s);
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM Dot Product 指令集
    isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
#else
    // 如果不支持 ARM Dot Product 指令集
    int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                             vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0]));
    // ...
    // 其他类似的操作
#endif

sum += isum * d_all * y[i].d;

#elif defined __AVX2__
    // 如果支持 AVX2 指令集
    const __m256i m4 = _mm256_set1_epi8(0xF);
    // ...
    // 其他类似的操作
    *s = hsum_float_8(acc);

#elif defined __AVX__
    // 如果支持 AVX 指令集
    const __m128i m4 = _mm_set1_epi8(0xF);
    // ...
    // 其他类似的操作
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic
    // 如果支持 RISC-V 向量指令集
    float sumf = 0;
    // ...
    // 其他类似的操作
    *s = sumf;

#else
    // 如果不支持以上任何一种指令集
    // ...
#endif
    # 定义一个长度为 QK_K 的 int8_t 类型数组
    int8_t  aux8[QK_K];
    # 定义一个长度为 8 的 int16_t 类型数组
    int16_t aux16[8];
    # 定义一个长度为 8 的 float 类型数组
    float   sums [8];
    # 定义一个长度为 8 的 int32_t 类型数组
    int32_t aux32[8];
    # 将 sums 数组的内容全部初始化为 0
    memset(sums, 0, 8*sizeof(float));

    # 定义一个 float 类型的变量 sumf，并初始化为 0
    float sumf = 0;
    # 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        # 定义指向 x[i].ql 的 uint8_t 类型指针 q4
        const uint8_t * restrict q4 = x[i].ql;
        # 定义指向 x[i].qh 的 uint8_t 类型指针 qh
        const uint8_t * restrict qh = x[i].qh;
        # 定义指向 y[i].qs 的 int8_t 类型指针 q8
        const  int8_t * restrict q8 = y[i].qs;
        # 将 aux32 数组的内容全部初始化为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        # 定义指向 aux8 的 int8_t 类型指针 a
        int8_t * restrict a = aux8;
        # 循环遍历 16 次
        for (int l = 0; l < 16; ++l) {
            # 计算 a 数组的值
            a[l+ 0] = (int8_t)((q4[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            a[l+16] = (int8_t)((q4[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            a[l+32] = (int8_t)((q4[l+ 0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            a[l+48] = (int8_t)((q4[l+16] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
        }
        # 定义一个 int 类型的变量 is，并初始化为 0
        int is = 0;
        # 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            # 获取 x[i].scales[is] 的值
            int scale = x[i].scales[is++];
            # 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 更新 q8 和 a 的指针位置
            q8 += 8; a += 8;
            # 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算 aux32 数组的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 更新 q8 和 a 的指针位置
            q8 += 8; a += 8;
        }
        # 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        # 更新 sums 数组的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    # 循环遍历 8 次，更新 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 将 sumf 的值赋给指针 s 指向的位置
    *s = sumf;
#endif



}



#endif
```