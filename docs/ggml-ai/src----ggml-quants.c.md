# `ggml\src\ggml-quants.c`

```
#include "ggml-quants.h"  // 引入自定义的头文件 ggml-quants.h
#include "ggml-impl.h"  // 引入自定义的头文件 ggml-impl.h

#include <math.h>  // 引入数学函数库
#include <string.h>  // 引入字符串处理函数库
#include <assert.h>  // 引入断言函数库
#include <float.h>  // 引入浮点数处理函数库

#ifdef __ARM_NEON  // 如果是 ARM 平台并且支持 NEON 指令集

// 如果 YCM 无法找到 <arm_neon.h>，可以创建一个符号链接指向它，例如：
//
//   $ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
//
#include <arm_neon.h>  // 引入 ARM NEON 指令集的头文件

#else  // 如果不是 ARM 平台或不支持 NEON 指令集

#ifdef __wasm_simd128__  // 如果是 WebAssembly 平台并且支持 SIMD 指令集

#include <wasm_simd128.h>  // 引入 WebAssembly SIMD 指令集的头文件

#else  // 如果不是 WebAssembly 平台或不支持 SIMD 指令集
#if defined(__POWER9_VECTOR__) || defined(__powerpc64__)  // 如果是 PowerPC 平台并且支持向量指令集

#include <altivec.h>  // 引入 PowerPC 平台的向量指令集头文件
#undef bool  // 取消定义 bool
#define bool _Bool  // 重新定义 bool 为 _Bool
#else  // 如果不是 PowerPC 平台

#if defined(_MSC_VER) || defined(__MINGW32__)  // 如果是 Windows 平台并且使用 MSVC 或 MinGW 编译器

#include <intrin.h>  // 引入 MSVC 或 MinGW 编译器的内置函数头文件

#else  // 如果不是 Windows 平台

#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__) || defined(__SSE3__)  // 如果支持 AVX、AVX2、AVX512F、SSSE3 或 SSE3 指令集

#if !defined(__riscv)  // 如果不是 RISC-V 平台

#include <immintrin.h>  // 引入 AVX、AVX2、AVX512F、SSSE3 或 SSE3 指令集的头文件

#endif
#endif
#endif
#endif
#endif
#endif

#ifdef __riscv_v_intrinsic  // 如果是 RISC-V 平台并且支持向量指令集

#include <riscv_vector.h>  // 引入 RISC-V 平台的向量指令集头文件

#endif

#undef MIN  // 取消定义 MIN
#undef MAX  // 取消定义 MAX

#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义宏 MIN，返回 a 和 b 中的最小值
#define MAX(a, b) ((a) > (b) ? (a) : (b))  // 定义宏 MAX，返回 a 和 b 中的最大值

#define MM256_SET_M128I(a, b) _mm256_insertf128_si256(_mm256_castsi128_si256(b), (a), 1)  // 定义宏 MM256_SET_M128I，将两个 __m128i 合并成一个 __m256i

#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)  // 如果支持 AVX、AVX2、AVX512F 或 SSSE3 指令集

// 乘法 int8_t，两两相加结果
static inline __m128i mul_sum_i8_pairs(const __m128i x, const __m128i y) {
    // 获取 x 向量的绝对值
    const __m128i ax = _mm_sign_epi8(x, x);
    // 对 y 向量的值进行符号化
    const __m128i sy = _mm_sign_epi8(y, x);
    // 执行乘法并创建 16 位值
    const __m128i dot = _mm_maddubs_epi16(ax, sy);
    const __m128i ones = _mm_set1_epi16(1);
    return _mm_madd_epi16(ones, dot);
}

#if __AVX__ || __AVX2__ || __AVX512F__  // 如果支持 AVX、AVX2 或 AVX512F 指令集

// 水平加法 8 个浮点数
static inline float hsum_float_8(const __m256 x) {
    __m128 res = _mm256_extractf128_ps(x, 1);
    res = _mm_add_ps(res, _mm256_castps256_ps128(x));
    res = _mm_add_ps(res, _mm_movehl_ps(res, res));
    res = _mm_add_ss(res, _mm_movehdup_ps(res));
    return _mm_cvtss_f32(res);
}
// 水平添加 8 个 int32_t
static inline int hsum_i32_8(const __m256i a) {
    // 将 a 的低 128 位和高 128 位相加
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

// 水平添加 4 个 int32_t
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
// 将 32 位扩展为 32 字节 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于按位重排的掩码
    const __m256i shuf_mask = _mm256_set_epi64x(
            0x0303030303030303, 0x0202020202020202,
            0x0101010101010101, 0x0000000000000000);
    // 使用掩码按位重排 x32，并存储到 bytes 中
    __m256i bytes = _mm256_shuffle_epi8(_mm256_set1_epi32(x32), shuf_mask);
    // 创建用于按位或的掩码
    const __m256i bit_mask = _mm256_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 将 bytes 和 bit_mask 按位或，并返回结果
    bytes = _mm256_or_si256(bytes, bit_mask);
    return _mm256_cmpeq_epi8(bytes, _mm256_set1_epi64x(-1));
}

// 将 32 个 4 位字段解包为 32 字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 加载 rsi 指向的 16 字节数据到 tmp 中
    const __m128i tmp = _mm_loadu_si128((const __m128i *)rsi);
    // 将 tmp 右移 4 位，并与 tmp 相加，存储到 bytes 中
    const __m256i bytes = MM256_SET_M128I(_mm_srli_epi16(tmp, 4), tmp);
    // 创建用于按位与的掩码
    const __m256i lowMask = _mm256_set1_epi8( 0xF );
    // 将 bytes 和 lowMask 按位与，并返回结果
    return _mm256_and_si256(lowMask, bytes);
}

// 逐对添加 int16_t 并作为浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m256i x) {
    // 创建所有元素为 1 的向量
    const __m256i ones = _mm256_set1_epi16(1);
    # 使用 AVX2 指令集中的 _mm256_madd_epi16 函数，对两个 256 位整数寄存器中的元素进行乘法和加法操作
    const __m256i summed_pairs = _mm256_madd_epi16(ones, x);
    # 使用 AVX2 指令集中的 _mm256_cvtepi32_ps 函数，将 256 位整数寄存器中的元素转换为 256 位浮点数寄存器
    return _mm256_cvtepi32_ps(summed_pairs);
// 用于将 32 位数据扩展为 32 个字节数据 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    // 将输入的字节数组 x 的前 4 个字节拷贝到 x32 中
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于低位字节的掩码
    const __m128i shuf_maskl = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
    // 创建用于高位字节的掩码
    const __m128i shuf_maskh = _mm_set_epi64x(0x0303030303030303, 0x0202020202020202);
    // 使用掩码对输入的 x32 进行字节重排，得到低位字节
    __m128i bytesl = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskl);
    // 使用掩码对输入的 x32 进行字节重排，得到高位字节
    __m128i bytesh = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskh);
    // 创建用于设置特定位的掩码
    const __m128i bit_mask = _mm_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 对低位字节进行按位或操作
    bytesl = _mm_or_si128(bytesl, bit_mask);
    // 对高位字节进行按位或操作
    bytesh = _mm_or_si128(bytesh, bit_mask);
    // 对低位字节进行比较操作，判断是否全部为 1
    bytesl = _mm_cmpeq_epi8(bytesl, _mm_set1_epi64x(-1));
    // 对高位字节进行比较操作，判断是否全部为 1
    bytesh = _mm_cmpeq_epi8(bytesh, _mm_set1_epi64x(-1));
    // 将低位字节和高位字节合并成一个 256 位的结果并返回
    return MM256_SET_M128I(bytesh, bytesl);
// 将 32 个 4 位字段解压缩为 32 个字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从内存中加载 16 个字节
    __m128i tmpl = _mm_loadu_si128((const __m128i *)rsi);
    // 将 tmpl 向右移动 4 位
    __m128i tmph = _mm_srli_epi16(tmpl, 4);
    // 创建一个低位掩码
    const __m128i lowMask = _mm_set1_epi8(0xF);
    // 对 tmpl 和 tmph 进行按位与操作
    tmpl = _mm_and_si128(lowMask, tmpl);
    tmph = _mm_and_si128(lowMask, tmph);
    return MM256_SET_M128I(tmph, tmpl);
}

// 将 int16_t 两两相加，并作为浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m128i xh, const __m128i xl) {
    // 创建一个全为 1 的向量
    const __m128i ones = _mm_set1_epi16(1);
    // 将 xl 中的每对值相加
    const __m128i summed_pairsl = _mm_madd_epi16(ones, xl);
    // 将 xh 中的每对值相加
    const __m128i summed_pairsh = _mm_madd_epi16(ones, xh);
    // 将结果合并为一个 256 位整数向量
    const __m256i summed_pairs = MM256_SET_M128I(summed_pairsh, summed_pairsl);
    // 将结果转换为浮点向量并返回
    return _mm256_cvtepi32_ps(summed_pairs);
}

static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
    // 将 ax 转换为两个 128 位整数向量
    const __m128i axl = _mm256_castsi256_si128(ax);
    const __m128i axh = _mm256_extractf128_si256(ax, 1);
    // 将 sy 转换为两个 128 位整数向量
    const __m128i syl = _mm256_castsi256_si128(sy);
    const __m128i syh = _mm256_extractf128_si256(sy, 1);
    // 执行乘法并创建 16 位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    return sum_i16_pairs_float(doth, dotl);
}

// 乘以 int8_t，两两相加两次并作为浮点向量返回
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
    // 将 x 转换为两个 128 位整数向量
    const __m128i xl = _mm256_castsi256_si128(x);
    const __m128i xh = _mm256_extractf128_si256(x, 1);
    // 将 y 转换为两个 128 位整数向量
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
    # 从给定的向量中获取指定索引位置的32位整数，并将它们相加
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

// 定义一个内联函数，用于加载 int8_t 类型的数据到 ggml_int8x16x2_t 结构体中
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
// 定义一个内联函数，用于加载指针指向的内存中的数据到一个包含四个 int8x16_t 类型元素的结构体中
inline static ggml_int8x16x4_t ggml_vld1q_s8_x4(const int8_t * ptr) {
    ggml_int8x16x4_t res;

    // 从 ptr + 0 处加载 16 个 int8 类型的数据到 res.val[0] 中
    res.val[0] = vld1q_s8(ptr + 0);
    // 从 ptr + 16 处加载 16 个 int8 类型的数据到 res.val[1] 中
    res.val[1] = vld1q_s8(ptr + 16);
    // 从 ptr + 32 处加载 16 个 int8 类型的数据到 res.val[2] 中
    res.val[2] = vld1q_s8(ptr + 32);
    // 从 ptr + 48 处加载 16 个 int8 类型的数据到 res.val[3] 中
    res.val[3] = vld1q_s8(ptr + 48);

    return res;
}

#else

// 定义一系列宏，将一些类型和函数重命名为另一些类型和函数
#define ggml_int16x8x2_t  int16x8x2_t
#define ggml_uint8x16x2_t uint8x16x2_t
#define ggml_uint8x16x4_t uint8x16x4_t
#define ggml_int8x16x2_t  int8x16x2_t
#define ggml_int8x16x4_t  int8x16x4_t

#define ggml_vld1q_s16_x2 vld1q_s16_x2
#define ggml_vld1q_u8_x2  vld1q_u8_x2
#define ggml_vld1q_u8_x4  vld1q_u8_x4
#define ggml_vld1q_s8_x2  vld1q_s8_x2
#define ggml_vld1q_s8_x4  vld1q_s8_x4

#endif

#if !defined(__ARM_FEATURE_DOTPROD)

// 定义一个内联函数，用于计算两个 int8x16_t 类型向量的点积
inline static int32x4_t ggml_vdotq_s32(int32x4_t acc, int8x16_t a, int8x16_t b) {
    // 计算 a 和 b 的低 8 位的乘积，并将结果扩展为 int16x8_t 类型
    const int16x8_t p0 = vmull_s8(vget_low_s8 (a), vget_low_s8 (b));
    // 计算 a 和 b 的高 8 位的乘积，并将结果扩展为 int16x8_t 类型
    const int16x8_t p1 = vmull_s8(vget_high_s8(a), vget_high_s8(b));

    // 将 p0 和 p1 的每个元素相加，并将结果扩展为 int32x4_t 类型，然后与 acc 相加
    return vaddq_s32(acc, vaddq_s32(vpaddlq_s16(p0), vpaddlq_s16(p1)));
}

#else

// 将 ggml_vdotq_s32 宏重命名为 vdotq_s32
#define ggml_vdotq_s32(a, b, c) vdotq_s32(a, b, c)

#endif

#endif

#if defined(__ARM_NEON) || defined(__wasm_simd128__)
// 定义一系列宏，用于生成预先计算的表格，将 8 位数据扩展为 8 字节数据
#define B1(c,s,n)  0x ## n ## c ,  0x ## n ## s
#define B2(c,s,n) B1(c,s,n ## c), B1(c,s,n ## s)
#define B3(c,s,n) B2(c,s,n ## c), B2(c,s,n ## s)
#define B4(c,s,n) B3(c,s,n ## c), B3(c,s,n ## s)
#define B5(c,s,n) B4(c,s,n ## c), B4(c,s,n ## s)
#define B6(c,s,n) B5(c,s,n ## c), B5(c,s,n ## s)
#define B7(c,s,n) B6(c,s,n ## c), B6(c,s,n ## s)
#define B8(c,s  ) B7(c,s,     c), B7(c,s,     s)

// 预先计算的表格，将 8 位数据扩展为 8 字节数据
static const uint64_t table_b2b_0[1 << 8] = { B8(00, 10) }; // ( b) << 4
static const uint64_t table_b2b_1[1 << 8] = { B8(10, 00) }; // (!b) << 4
#endif

// 参考实现，用于确定性地创建模型文件
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k) {
    static const int qk = QK4_0;

    // 断言 k 是 qk 的倍数
    assert(k % qk == 0);

    // 计算 nb，即 k 除以 qk 的商
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
}

// 将输入的一维数组 x 进行量化，结果存储在 y 中，每个块的大小为 k
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_0_reference 函数进行量化
    quantize_row_q4_0_reference(x, y, k);
}

// 对输入的一维数组 x 进行量化，结果存储在 y 中，每个块的大小为 k
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k) {
    // 定义每个块的大小为 QK4_1
    const int qk = QK4_1;

    // 断言 k 能够整除 qk
    assert(k % qk == 0);

    // 计算块的数量
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历当前块内的元素，找到最小值和最大值
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];

            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算量化参数
        const float d  = (max - min) / ((1 << 4) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将量化参数和最小值存储在 y 中
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

// 将输入的一维数组 x 进行量化，结果存储在 y 中，每个块的大小为 k
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_1_reference 函数进行量化
    quantize_row_q4_1_reference(x, y, k);
}

// 对输入的一维数组 x 进行量化，结果存储在 y 中，每个块的大小为 k
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k) {
    // 定义每个块的大小为 QK5_0
    static const int qk = QK5_0;

    // 断言 k 能够整除 qk
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
        // 定义并初始化 srcv 数组，用于存储加载的数据
        v128_t srcv [8];
        // 定义并初始化 asrcv 数组，用于存储 srcv 数组中数据的绝对值
        v128_t asrcv[8];
        // 定义并初始化 amaxv 数组，用于存储 asrcv 数组中数据的最大值

        // 遍历加载数据到 srcv 数组中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        // 计算 srcv 数组中数据的绝对值并存储到 asrcv 数组中
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        // 计算 asrcv 数组中数据的最大值并存储到 amaxv 数组中
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv 数组中数据的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        // 计算 d 和 id 的值
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 格式并存储到 y[i].d 中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 遍历计算并存储最终结果到 y[i].qs 数组中
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
    // 如果定义了 AVX2 或 AVX，执行以下代码块
    for (int i = 0; i < nb; i++) {
        // 对 x 中的元素加载到 4 个 AVX 向量中
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
        // 如果我们没有 AVX2 中一些必要的函数，
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
    // 根据指定的配置参数创建一个向量长度对象
    size_t vl = __riscv_vsetvl_e32m4(QK8_0);

    // 遍历处理每个元素
    for (int i = 0; i < nb; i++) {
        // 加载元素到向量中
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_0, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 初始化临时变量
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0f, vl);
        // 计算向量中的最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        // 将最大值转换为标量
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        // 计算常量值
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将浮点数转换为半精度浮点数
        y[i].d = GGML_FP32_TO_FP16(d);

        // 对向量进行除法运算
        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        // 将浮点数向量转换为整数向量
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        // 将整数向量转换为字节向量
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        // 存储结果
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);
    }
// 如果不满足条件，则使用 GGML_UNUSED 宏来标记未使用的变量 nb
#else
    GGML_UNUSED(nb);
    // 标记注释，说明下面的操作是对标量进行量化
    // 调用 quantize_row_q8_0_reference 函数对 x, y, k 进行量化
    quantize_row_q8_0_reference(x, y, k);
#endif
}

// 用于确定性创建模型文件的参考实现
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k) {
    // 断言，确保 QK8_1 的值为 32
    assert(QK8_1 == 32);
    // 断言，确保 k 能够被 QK8_1 整除
    assert(k % QK8_1 == 0);
    // 计算 nb 的值
    const int nb = k / QK8_1;

    // 循环处理每个块
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对最大值

        // 计算每个块中的绝对最大值
        for (int j = 0; j < QK8_1; j++) {
            const float v = x[i*QK8_1 + j];
            amax = MAX(amax, fabsf(v));
        }

        // 计算量化因子 d
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 存储到 y[i].d 中
        y[i].d = d;

        int sum = 0;

        // 对每个块进行量化
        for (int j = 0; j < QK8_1/2; ++j) {
            const float v0 = x[i*QK8_1           + j]*id;
            const float v1 = x[i*QK8_1 + QK8_1/2 + j]*id;

            y[i].qs[          j] = roundf(v0);
            y[i].qs[QK8_1/2 + j] = roundf(v1);

            sum += y[i].qs[          j];
            sum += y[i].qs[QK8_1/2 + j];
        }

        // 计算 y[i].s
        y[i].s = sum*d;
    }
}

// 对输入进行量化
void quantize_row_q8_1(const float * restrict x, void * restrict vy, int k) {
    // 断言，确保 k 能够被 QK8_1 整除
    assert(k % QK8_1 == 0);
    // 计算 nb 的值
    const int nb = k / QK8_1;

    // 将 vy 强制转换为 block_q8_1 类型的指针
    block_q8_1 * restrict y = vy;

    // 如果定义了 __ARM_NEON，则执行以下操作
#if defined(__ARM_NEON)
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 声明并初始化长度为 8 的 float32x4_t 数组
        float32x4_t srcv [8];
        // 声明并初始化长度为 8 的 float32x4_t 数组
        float32x4_t asrcv[8];
        // 声明并初始化长度为 8 的 float32x4_t 数组
        float32x4_t amaxv[8];

        // 循环遍历 8 次，从数组 x 中加载数据到 srcv 数组
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 循环遍历 8 次，计算 srcv 数组中元素的绝对值并存储到 asrcv 数组
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 循环遍历 4 次，计算 asrcv 数组中相邻两个元素的最大值并存储到 amaxv 数组
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 循环遍历 2 次，计算 amaxv 数组中相邻两个元素的最大值并存储到 amaxv 数组
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 计算 amaxv 数组中相邻两个元素的最大值并存储到 amaxv 数组
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv[0] 中的最大值并存储到 amax 变量
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算 d 变量的值
        const float d = amax / ((1 << 7) - 1);
        // 计算 id 变量的值
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 变量的值存储到 y[i].d 中
        y[i].d = d;

        // 初始化长度为 4 的 int32x4_t 数组
        int32x4_t accv = vdupq_n_s32(0);

        // 循环遍历 8 次，进行一系列计算并将结果存储到 y[i].qs 和 accv 中
        for (int j = 0; j < 8; j++) {
            // 计算 v 变量的值
            const float32x4_t v  = vmulq_n_f32(srcv[j], id);
            // 将 v 转换为整型并存储到 vi 中
            const int32x4_t   vi = vcvtnq_s32_f32(v);

            // 将 vi 中的值存储到 y[i].qs 中
            y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
            y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
            y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
            y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);

            // 将 vi 中的值加到 accv 中
            accv = vaddq_s32(accv, vi);
        }

        // 计算 y[i].s 的值并存储到 y[i].s 中
        y[i].s = d * vaddvq_s32(accv);
    }
#elif defined(__wasm_simd128__)
    # 对于每个块
    for (int i = 0; i < nb; i++) {
        # 定义8个128位向量
        v128_t srcv [8];
        v128_t asrcv[8];
        v128_t amaxv[8];

        # 从内存中加载32位浮点数到向量中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        # 计算每个向量的绝对值
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        # 计算每两个相邻向量的最大值
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        # 计算每两个相邻最大值的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        # 计算所有最大值的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        # 计算最大值的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        # 计算比例因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        # 将比例因子存储到y[i].d中
        y[i].d = d;

        # 初始化累加器向量
        v128_t accv = wasm_i32x4_splat(0);

        # 对于每个向量
        for (int j = 0; j < 8; j++) {
            # 计算缩放后的向量
            const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
            # 将浮点数向下取整到整数
            const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

            # 将整数存储到y[i].qs中
            y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
            y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
            y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
            y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);

            # 更新累加器
            accv = wasm_i32x4_add(accv, vi);
        }

        # 计算y[i].s
        y[i].s = d * (wasm_i32x4_extract_lane(accv, 0) +
                      wasm_i32x4_extract_lane(accv, 1) +
                      wasm_i32x4_extract_lane(accv, 2) +
                      wasm_i32x4_extract_lane(accv, 3));
    }
#elif defined(__AVX2__) || defined(__AVX__)
    for (int i = 0; i < nb; i++) {
        // 遍历每个块中的元素，每次加载4个 AVX 向量
        __m256 v0 = _mm256_loadu_ps( x );
        __m256 v1 = _mm256_loadu_ps( x + 8 );
        __m256 v2 = _mm256_loadu_ps( x + 16 );
        __m256 v3 = _mm256_loadu_ps( x + 24 );
        x += 32;

        // 计算块中元素的最大绝对值
        const __m256 signBit = _mm256_set1_ps( -0.0f );
        __m256 maxAbs = _mm256_andnot_ps( signBit, v0 );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v1 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v2 ) );
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v3 ) );

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

        // 我们得到了有价值的有符号字节，但是顺序现在是错误的
        // 这些 AVX2 pack 指令独立处理 16 字节的数据
        // 以下指令用于修正顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );

        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 如果不支持 AVX，将寄存器分成两半，并从 SSE 调用 AVX2 的模拟函数
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

    size_t vl = __riscv_vsetvl_e32m4(QK8_1);
    for (int i = 0; i < nb; i++) {
        // 遍历处理每个元素
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_1, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 初始化临时变量
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

// 对行进行反量化，使用 Q4_0 量化
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK4_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb
    const int nb = k / qk;

    // 遍历 nb
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历 qk/2
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

// 对行进行反量化，使用 Q4_1 量化
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK4_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb
    const int nb = k / qk;

    // 遍历 nb
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 和 x[i].m 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        // 遍历 qk/2
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

// 对行进行反量化，使用 Q5_0 量化
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK5_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb
    const int nb = k / qk;

    // 遍历 nb
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 从 x[i].qh 复制 4 个字节到 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 遍历 qk/2
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

// 对行进行反量化，使用 Q5_1 量化
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK5_1;
    # 确保 k 能够整除 qk，否则触发断言错误
    assert(k % qk == 0);

    # 计算出数组 x 的长度
    const int nb = k / qk;

    # 遍历数组 x 的每个元素
    for (int i = 0; i < nb; i++) {
        # 将数组 x 中的元素从 FP16 格式转换为 FP32 格式
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        # 从数组 x 中的元素中提取出 qh，并转换为 uint32_t 类型
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 遍历每个元素的一半长度
        for (int j = 0; j < qk/2; ++j) {
            # 从 qh 中提取出两个字节的数据
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            # 从数组 x 中的元素中提取出两个字节的数据
            const int x0 = (x[i].qs[j] & 0x0F) | xh_0;
            const int x1 = (x[i].qs[j] >>   4) | xh_1;

            # 计算并存储结果到数组 y 中
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
// 对输入的行进行反量化，将量化后的数据转换为浮点数
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK8_0;

    // 断言 k 能够整除 qk
    assert(k % qk == 0);

    // 计算块数
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化后的数据转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个块中的量化数据
        for (int j = 0; j < qk; ++j) {
            // 反量化
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
    // 断言 fval 不大于 4194303.f
    assert(fval <= 4194303.f);
    float val = fval + 12582912.f;
    int i; memcpy(&i, &val, sizeof(int));
    return (i & 0x007fffff) - 0x00400000;
}

// 生成量化后的数据
static float make_qx_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, int rmse_type) {
    float max = 0;
    float amax = 0;
    // 遍历输入数据，找到最大值
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果最大值接近于零，将输出数据全部置为零
    if (amax < 1e-30f) {
        for (int i = 0; i < n; ++i) {
            L[i] = 0;
        }
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    // 根据 rmse_type 的不同进行不同的量化处理
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
    // 根据不同的权重类型计算量化后的数据
    for (int i = 0; i < n; ++i) {
        int l = nearest_int(iscale * x[i]);
        l = MAX(-nmax, MIN(nmax-1, l));
        L[i] = l + nmax;
        float w = weight_type == 1 ? x[i] * x[i] : 1;
        sumlx += w*x[i]*l;
        suml2 += w*l*l;
    }
    // 计算缩放因子
    float scale = sumlx/suml2;
    // 如果需要提前返回结果，则根据条件返回不同的值
    if (return_early) return suml2 > 0 ? 0.5f*(scale + 1/iscale) : 1/iscale;
    float best = scale * sumlx;
    # 遍历 is 变量，范围为 -9 到 9
    for (int is = -9; is <= 9; ++is) {
        # 如果 is 等于 0，则跳过本次循环
        if (is == 0) {
            continue;
        }
        # 计算 iscale 值
        iscale = -(nmax + 0.1f*is) / max;
        # 初始化 sumlx 和 suml2
        sumlx = suml2 = 0;
        # 遍历 n 个元素
        for (int i = 0; i < n; ++i) {
            # 计算最接近 iscale * x[i] 的整数值
            int l = nearest_int(iscale * x[i]);
            # 将 l 限制在 -nmax 和 nmax-1 之间
            l = MAX(-nmax, MIN(nmax-1, l));
            # 根据权重类型计算权重值
            float w = weight_type == 1 ? x[i] * x[i] : 1;
            # 更新 sumlx 和 suml2
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        # 如果 suml2 大于 0 并且 sumlx*sumlx 大于 best*suml2
        if (suml2 > 0 && sumlx*sumlx > best*suml2) {
            # 更新 L 数组的值
            for (int i = 0; i < n; ++i) {
                int l = nearest_int(iscale * x[i]);
                L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
            }
            # 更新 scale 和 best 的值
            scale = sumlx/suml2; best = scale*sumlx;
        }
    }
    # 返回 scale 的值
    return scale;
// 计算第三个四分位数的分位数，返回分位数和量化后的数据
static float make_q3_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, bool do_rmse) {
    // 初始化最大值和绝对值最大值
    float max = 0;
    float amax = 0;
    // 遍历数组 x，找到最大值和绝对值最大值
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果绝对值最大值为 0，即所有值都为 0
    if (!amax) { 
        // 将 L 数组全部置为 0
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
            // 计算量化值
            int l = nearest_int(iscale * x[i]);
            // 限制量化值的范围
            l = MAX(-nmax, MIN(nmax-1, l));
            // 将量化值保存到 L 数组中
            L[i] = l;
            // 计算中间变量
            float w = x[i]*x[i];
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        // 迭代计算，最多 5 次
        for (int itry = 0; itry < 5; ++itry) {
            int n_changed = 0;
            // 遍历数组 x
            for (int i = 0; i < n; ++i) {
                // 计算中间变量
                float w = x[i]*x[i];
                float slx = sumlx - w*x[i]*L[i];
                // 如果满足条件
                if (slx > 0) {
                    float sl2 = suml2 - w*L[i]*L[i];
                    // 计算新的量化值
                    int new_l = nearest_int(x[i] * sl2 / slx);
                    new_l = MAX(-nmax, MIN(nmax-1, new_l));
                    // 如果新的量化值不等于原来的值
                    if (new_l != L[i]) {
                        // 更新中间变量
                        slx += w*x[i]*new_l;
                        sl2 += w*new_l*new_l;
                        // 如果满足条件
                        if (sl2 > 0 && slx*slx*suml2 > sumlx*sumlx*sl2) {
                            // 更新量化值和中间变量
                            L[i] = new_l; sumlx = slx; suml2 = sl2;
                            ++n_changed;
                        }
                    }
                }
            }
            // 如果没有变化，跳出循环
            if (!n_changed) {
                break;
            }
        }
        // 将量化值转换为非负值
        for (int i = 0; i < n; ++i) {
            L[i] += nmax;
        }
        // 返回均方根误差
        return sumlx / suml2;
    }
    // 如果不需要计算均方根误差
    for (int i = 0; i < n; ++i) {
        // 计算量化值
        int l = nearest_int(iscale * x[i]);
        // 限制量化值的范围
        l = MAX(-nmax, MIN(nmax-1, l));
        // 将量化值保存到 L 数组中
        L[i] = l + nmax;
    }
    // 返回缩放因子的倒数
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
    # 如果最大值等于最小值，将量化数组全部置为0，返回0
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
        # 遍历数组，更新量化数组，并计算相关值
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
    # 返回最小值的相反数
    *the_min = -min;
    return scale;
}

# 计算并返回二维量化的比例因子
static float make_qkx2_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    # 初始化最小值和最大值为数组第一个元素的值
    float min = x[0];
    float max = x[0];
    float sum_w = weights[0];
    float sum_x = sum_w * x[0];
    # 遍历数组，更新最小值、最大值、权重和
#ifdef HAVE_BUGGY_APPLE_LINKER
    # 使用'volatile'修饰符来解决苹果ld64 1015.7版本的bug
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
    # 如果最大值和最小值相等
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
        # 如果使用平均绝对偏差，则计算绝对值，否则计算平方
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
            # 计算和与乘积
            sum_l += w*l;
            sum_l2 += w*l*l;
            sum_xl += w*l*x[i];
        }
        # 计算 D 值
        float D = sum_w * sum_l2 - sum_l * sum_l;
        # 如果 D 大于 0
        if (D > 0) {
            # 计算 this_scale 和 this_min
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;
            # 如果 this_min 大于 0，则重新计算 this_min 和 this_scale
            if (this_min > 0) {
                this_min = 0;
                this_scale = sum_xl / sum_l2;
            }
            # 初始化平均绝对偏差
            float mad = 0;
            # 遍历数组 x
            for (int i = 0; i < n; ++i) {
                # 计算差值
                float diff = this_scale * Laux[i] + this_min - x[i];
                # 如果使用平均绝对偏差，则计算绝对值，否则计算平方
                diff = use_mad ? fabsf(diff) : diff * diff;
                # 获取权重
                float w = weights[i];
                # 计算平均绝对偏差
                mad += w * diff;
            }
            # 如果平均绝对偏差小于最佳平均绝对偏差
            if (mad < best_mad) {
                # 更新 L、best_mad、scale 和 min
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
        // 将 q[j] 和 q[j+4] 分别与 63 进行按位与操作，将结果赋值给 *d 和 *m
        *d = q[j] & 63; *m = q[j + 4] & 63;
    } else {
        // 否则执行以下操作
        // 将 q[j+4] 与 0xF 进行按位与操作，将 q[j-4] 右移 6 位后的结果左移 4 位，然后进行按位或操作，将结果赋值给 *d
        *d = (q[j+4] & 0xF) | ((q[j-4] >> 6) << 4);
        // 将 q[j+4] 右移 4 位，将 q[j-0] 右移 6 位后的结果左移 4 位，然后进行按位或操作，将结果赋值给 *m
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

    // 定义长度为 QK_K 的 uint8_t 数组 L
    uint8_t L[QK_K];
    // 定义长度为 16 的 uint8_t 数组 Laux
    uint8_t Laux[16];
    // 定义长度为 16 的 float 数组 weights
    float   weights[16];
    // 定义长度为 QK_K/16 的 float 数组 mins
    float mins[QK_K/16];
    // 定义长度为 QK_K/16 的 float 数组 scales
    float scales[QK_K/16];

    // 定义一个常量 q4scale，值为 15.0
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
            float scale = scales[j];
            // 更新最大比例
            if (scale > max_scale) {
                max_scale = scale;
            }
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
                y[i].scales[j] = l;
            }
            // 计算并存储 d 值
            y[i].d = GGML_FP32_TO_FP16(max_scale/q4scale);
        } else {
            // 如果最大比例小于等于 0，则将所有比例值设为 0
            for (int j = 0; j < QK_K/16; ++j) y[i].scales[j] = 0;
            // 将 d 值设为 0
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
                y[i].scales[j] |= (l << 4);
            }
            // 计算并存储 dmin 值
            y[i].dmin = GGML_FP32_TO_FP16(max_min/q4scale);
        } else {
            // 如果最大最小值小于等于 0，则将 dmin 值设为 0
            y[i].dmin = GGML_FP32_TO_FP16(0.f);
        }
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 d 和 dm 的值，并更新 L 数组
            const float d = GGML_FP16_TO_FP32(y[i].d) * (y[i].scales[j] & 0xF);
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * (y[i].scales[j] >> 4);
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int((x[16*j + ii] + dm)/d);
                l = MAX(0, MIN(3, l));
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
                // 将 L 数组中的值按位组合后赋给 y[i].qs 数组
            }
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 循环遍历，每次增加 1
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
            // 将 L 数组中的值按位组合后赋给 y[i].qs 数组
        }
#endif

        x += QK_K;
        // x 增加 QK_K 的值

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
        // 将 x[i].d 转换为 float 类型赋给 d
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].dmin 转换为 float 类型赋给 min

        const uint8_t * q = x[i].qs;
        // 将 x[i].qs 的地址赋给 q

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
                // 从 x[i].scales 数组中取值赋给 sc
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 计算 dl 和 ml 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l] >> shift) & 3)) - ml;
                // 计算并赋值给 y 数组
                sc = x[i].scales[is++];
                // 从 x[i].scales 数组中取值赋给 sc
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 计算 dl 和 ml 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3)) - ml;
                // 计算并赋值给 y 数组
                shift += 2;
            }
            q += 32;
            // q 的地址增加 32
        }
#else
        // 计算每个元素的 dl 和 ml 值
        float dl1 = d * (x[i].scales[0] & 0xF), ml1 = min * (x[i].scales[0] >> 4);
        float dl2 = d * (x[i].scales[1] & 0xF), ml2 = min * (x[i].scales[1] >> 4);
        float dl3 = d * (x[i].scales[2] & 0xF), ml3 = min * (x[i].scales[2] >> 4);
        float dl4 = d * (x[i].scales[3] & 0xF), ml4 = min * (x[i].scales[3] >> 4);
        // 遍历每个元素，进行量化计算
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

// 将 float 数组 x 进行量化，结果存储在 vy 中，每次处理 k 个元素
void quantize_row_q2_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q2_K_reference(x, vy, k);
}

// 对输入的 float 数组进行量化，并将结果存储在 dst 中，同时计算直方图
size_t ggml_quantize_q2_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 每次处理 k 个元素，调用 quantize_row_q2_K_reference 进行量化
    for (int j = 0; j < n; j += k) {
        block_q2_K * restrict y = (block_q2_K *)dst + j/QK_K;
        quantize_row_q2_K_reference(src + j, y, k);
    }
    return (n/QK_K*sizeof(block_q2_K));
}

//========================= 3-bit (de)-quantization

// 对输入的 float 数组进行 3-bit 量化，结果存储在 y 中，每次处理 k 个元素
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k) {
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    int8_t L[QK_K];
    float scales[QK_K / 16];

    // 遍历每个块，计算量化的 scales
    for (int i = 0; i < nb; i++) {
        float max_scale = 0;
        float amax = 0;
        // 遍历每个子块，计算 scales
        for (int j = 0; j < QK_K/16; ++j) {
            scales[j] = make_q3_quants(16, 4, x + 16*j, L + 16*j, true);
            float scale = fabsf(scales[j]);
            if (scale > amax) {
                amax = scale; max_scale = scales[j];
            }
        }
#if QK_K == 256
        // 如果 QK_K 等于 256，则将 y[i].scales 数组的前 12 个元素全部置为 0
        memset(y[i].scales, 0, 12);
        // 如果 max_scale 不为 0
        if (max_scale) {
            // 计算 iscale 的值
            float iscale = -32.f/max_scale;
            // 遍历 scales 数组
            for (int j = 0; j < QK_K/16; ++j) {
                // 将 iscale 乘以 scales[j] 的最近整数值赋给 l
                int8_t l = nearest_int(iscale*scales[j]);
                // 将 l 限制在 -32 和 31 之间，并加上 32
                l = MAX(-32, MIN(31, l)) + 32;
                // 根据 j 的值将 l 存入 y[i].scales 数组中
                if (j < 8) {
                    y[i].scales[j] = l & 0xF;
                } else {
                    y[i].scales[j-8] |= ((l & 0xF) << 4);
                }
                // 将 l 右移 4 位
                l >>= 4;
                // 将 l 存入 y[i].scales 数组中
                y[i].scales[j%4 + 8] |= (l << (2*(j/4)));
            }
            // 将 GGML_FP32_TO_FP16(1/iscale) 的结果存入 y[i].d 中
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
        } else {
            // 如果 max_scale 为 0，则将 GGML_FP32_TO_FP16(0.f) 的结果存入 y[i].d 中
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }

        int8_t sc;
        // 遍历 scales 数组
        for (int j = 0; j < QK_K/16; ++j) {
            // 根据 j 的值将 y[i].scales 数组中的值赋给 sc
            sc = j < 8 ? y[i].scales[j] & 0xF : y[i].scales[j-8] >> 4;
            // 对 sc 进行位运算和数值调整，并将结果存入 sc 中
            sc = (sc | (((y[i].scales[8 + j%4] >> (2*(j/4))) & 3) << 4)) - 32;
            // 计算 d 的值
            float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            // 如果 d 为 0，则跳过当前循环
            if (!d) {
                continue;
            }
            // 遍历 x 数组
            for (int ii = 0; ii < 16; ++ii) {
                // 将 x[16*j + ii] 除以 d 的最近整数值赋给 l
                int l = nearest_int(x[16*j + ii]/d);
                // 将 l 限制在 -4 和 3 之间，并加上 4
                l = MAX(-4, MIN(3, l));
                // 将 l + 4 的结果存入 L 数组中
                L[16*j + ii] = l + 4;
            }
        }
#else
        # 如果 max_scale 为真，则执行以下代码块
        if (max_scale) {
            # 计算缩放比例
            float iscale = -8.f/max_scale;
            # 遍历 QK_K/16 次，每次处理两个元素
            for (int j = 0; j < QK_K/16; j+=2) {
                # 计算缩放后的值，并限制在 -8 到 7 之间
                int l1 = nearest_int(iscale*scales[j]);
                l1 = 8 + MAX(-8, MIN(7, l1));
                int l2 = nearest_int(iscale*scales[j+1]);
                l2 = 8 + MAX(-8, MIN(7, l2));
                # 将 l1 和 l2 组合成一个 8 位的值，存入 y[i].scales 数组
                y[i].scales[j/2] = l1 | (l2 << 4);
            }
            # 将 1/iscale 转换成 FP16 格式，存入 y[i].d
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
        } else {
            # 如果 max_scale 为假，则执行以下代码块
            # 将 y[i].scales 数组的值全部置为 0
            for (int j = 0; j < QK_K/16; j+=2) {
                y[i].scales[j/2] = 0;
            }
            # 将 0.f 转换成 FP16 格式，存入 y[i].d
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        # 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            # 根据条件选择 y[i].scales[j/2] 的低 4 位或高 4 位，计算 d 的值
            int s = j%2 == 0 ? y[i].scales[j/2] & 0xF : y[i].scales[j/2] >> 4;
            float d = GGML_FP16_TO_FP32(y[i].d) * (s - 8);
            # 如果 d 为 0，则跳过当前循环
            if (!d) {
                continue;
            }
            # 遍历 16 次，计算 l 的值并存入 L 数组
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-4, MIN(3, l));
                L[16*j + ii] = l + 4;
            }
        }
#endif

        # 将 y[i].hmask 数组的值全部置为 0
        memset(y[i].hmask, 0, QK_K/8);
        # 将 L 数组中的值转换成对应的高位掩码，存入 y[i].hmask 数组
        int m = 0;
        uint8_t hm = 1;
        for (int j = 0; j < QK_K; ++j) {
            if (L[j] > 3) {
                y[i].hmask[m] |= hm;
                L[j] -= 4;
            }
            if (++m == QK_K/8) {
                m = 0; hm <<= 1;
            }
        }
#if QK_K == 256
        # 如果 QK_K 等于 256，则执行以下代码块
        # 将 L 数组中的值重新组合成 8 位的值，存入 y[i].qs 数组
        for (int j = 0; j < QK_K; j += 128) {
            for (int l = 0; l < 32; ++l) {
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
            }
        }
#else
        # 如果 QK_K 不等于 256，则执行以下代码块
        # 将 L 数组中的值重新组合成 8 位的值，存入 y[i].qs 数组
        for (int l = 0; l < 16; ++l) {
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
        }
#endif

        # 更新 x 的值
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
    // 遍历 nb 次，nb 为某个变量的值
    for (int i = 0; i < nb; i++) {

        // 将 x[i].d 从 FP16 格式转换为 FP32 格式
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 限定指针 q 指向的内存只能用于读取，指针 hm 指向的内存只能用于读取
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;

        // 根据不同的位运算和位移操作，计算出 d1、d2、d3、d4 的值
        const float d1 = d_all * ((x[i].scales[0] & 0xF) - 8);
        const float d2 = d_all * ((x[i].scales[0] >>  4) - 8);
        const float d3 = d_all * ((x[i].scales[1] & 0xF) - 8);
        const float d4 = d_all * ((x[i].scales[1] >>  4) - 8);

        // 遍历 8 次，计算出 y 数组中的值
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

// 对输入数组进行分块量化，调用参考实现函数
void quantize_row_q3_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q3_K_reference(x, vy, k);
}

// 对输入数组进行整体量化，返回量化后的结果和直方图
size_t ggml_quantize_q3_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对输入数组进行分块量化
    for (int j = 0; j < n; j += k) {
        block_q3_K * restrict y = (block_q3_K *)dst + j/QK_K;
        quantize_row_q3_K_reference(src + j, y, k);
    }
    // 返回量化后的结果字节数
    return (n/QK_K*sizeof(block_q3_K));
}

// ====================== 4-bit (de)-quantization

// 对输入数组进行分块量化的参考实现函数
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k) {
    // 断言，检查 k 是否能被 QK_K 整除
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    // 定义变量
    uint8_t L[QK_K];
    uint8_t Laux[32];
    float   weights[32];
    float mins[QK_K/32];
    float scales[QK_K/32];

    // 循环处理每个分块
    for (int i = 0; i < nb; i++) {

        // 初始化最大比例和最大最小值
        float max_scale = 0; // as we are deducting the min, scales are always positive
        float max_min = 0;
        // 循环处理每个子块
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算权重和比例
            //scales[j] = make_qkx1_quants(32, 15, x + 32*j, L + 32*j, &mins[j], 9, 0.5f);
            float sum_x2 = 0;
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l];
            float av_x = sqrtf(sum_x2/32);
            for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]);
            scales[j] = make_qkx2_quants(32, 15, x + 32*j, weights, L + 32*j, &mins[j], Laux, -1.f, 0.1f, 20, false);
            // 更新最大比例和最大最小值
            float scale = scales[j];
            if (scale > max_scale) {
                max_scale = scale;
            }
            float min = mins[j];
            if (min > max_min) {
                max_min = min;
            }
        }
#if QK_K == 256
        // 如果 QK_K 等于 256，则计算最大比例和最小比例的倒数，否则为 0
        float inv_scale = max_scale > 0 ? 63.f/max_scale : 0.f;
        float inv_min   = max_min   > 0 ? 63.f/max_min   : 0.f;
        // 遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算最接近的整数值，并限制在 0 到 63 之间
            uint8_t ls = nearest_int(inv_scale*scales[j]);
            uint8_t lm = nearest_int(inv_min*mins[j]);
            ls = MIN(63, ls);
            lm = MIN(63, lm);
            // 根据索引 j 的值进行条件判断和赋值
            if (j < 4) {
                y[i].scales[j] = ls;
                y[i].scales[j+4] = lm;
            } else {
                y[i].scales[j+4] = (ls & 0xF) | ((lm & 0xF) << 4);
                y[i].scales[j-4] |= ((ls >> 4) << 6);
                y[i].scales[j-0] |= ((lm >> 4) << 6);
            }
        }
        // 将最大比例和最小比例转换为 FP16 格式并赋值给 y[i].d 和 y[i].dmin
        y[i].d = GGML_FP32_TO_FP16(max_scale/63.f);
        y[i].dmin = GGML_FP32_TO_FP16(max_min/63.f);

        uint8_t sc, m;
        // 再次遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 调用函数获取 scale 和 min 的值
            get_scale_min_k4(j, y[i].scales, &sc, &m);
            // 计算 d 和 dm 的值，并根据条件进行处理
            const float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * m;
            // 遍历 32 次，计算 l 的值并限制在 0 到 15 之间
            for (int ii = 0; ii < 32; ++ii) {
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
        // 将第一个维度的值和最小值存入y[i]的scales数组中
        y[i].scales[0] = d1 | (m1 << 4);
        // 将第二个维度的值和最小值存入y[i]的scales数组中
        y[i].scales[1] = d2 | (m2 << 4);
        // 将最大缩放值转换为FP16格式存入y[i]的d数组中
        y[i].d[0] = GGML_FP32_TO_FP16(max_scale/s_factor);
        // 将最大最小值转换为FP16格式存入y[i]的d数组中
        y[i].d[1] = GGML_FP32_TO_FP16(max_min/s_factor);

        // 初始化sumlx和suml2
        float sumlx = 0;
        int   suml2 = 0;
        // 遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取当前scales数组中的值和最小值
            const uint8_t sd = y[i].scales[j] & 0xF;
            const uint8_t sm = y[i].scales[j] >>  4;
            // 计算d的值
            const float d = GGML_FP16_TO_FP32(y[i].d[0]) * sd;
            // 如果d为0，则跳过当前循环
            if (!d) continue;
            // 计算m的值
            const float m = GGML_FP16_TO_FP32(y[i].d[1]) * sm;
            // 遍历32次
            for (int ii = 0; ii < 32; ++ii) {
                // 计算l的值
                int l = nearest_int((x[32*j + ii] + m)/d);
                // 将l限制在0到15之间
                l = MAX(0, MIN(15, l));
                // 将l存入L数组中
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

        // 更新x的指针位置
        x += QK_K;
    }
}

// 解量化函数，将量化后的数据转换为浮点数
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k) {
    // 断言k能被QK_K整除
    assert(k % QK_K == 0);
    // 计算nb的值
    const int nb = k / QK_K;

    // 遍历nb次
    for (int i = 0; i < nb; i++) {
        // 获取当前x[i]的qs数组的指针
        const uint8_t * q = x[i].qs;
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块

        const float d   = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 32 位浮点数，存储在变量 d 中
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].dmin 转换为 32 位浮点数，存储在变量 min 中

        int is = 0;
        // 初始化变量 is 为 0
        uint8_t sc, m;
        // 声明 uint8_t 类型的变量 sc 和 m
        for (int j = 0; j < QK_K; j += 64) {
            // 循环，j 从 0 开始，每次增加 64，直到 j 小于 QK_K
            get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
            // 调用 get_scale_min_k4 函数，传入参数 is + 0, x[i].scales, &sc, &m
            const float d1 = d * sc; const float m1 = min * m;
            // 计算 d1 和 m1 的值
            get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
            // 调用 get_scale_min_k4 函数，传入参数 is + 1, x[i].scales, &sc, &m
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
        // 将 x[i].d[0] 转换为 32 位浮点数，存储在变量 dall 中
        const float mall = GGML_FP16_TO_FP32(x[i].d[1]);
        // 将 x[i].d[1] 转换为 32 位浮点数，存储在变量 mall 中
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
}

void quantize_row_q4_K(const float * restrict x, void * restrict vy, int k) {
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    block_q4_K * restrict y = vy;
    // 声明并初始化 block_q4_K 类型的指针 y，指向 vy
    quantize_row_q4_K_reference(x, y, k);
    // 调用 quantize_row_q4_K_reference 函数，传入参数 x, y, k
}

size_t ggml_quantize_q4_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    (void)hist; // TODO: collect histograms
    // 忽略 hist 参数，待实现：收集直方图

    for (int j = 0; j < n; j += k) {
        // 循环，j 从 0 开始，每次增加 k，直到 j 小于 n
        block_q4_K * restrict y = (block_q4_K *)dst + j/QK_K;
        // 声明并初始化 block_q4_K 类型的指针 y，指向 dst + j/QK_K
        quantize_row_q4_K_reference(src + j, y, k);
        // 调用 quantize_row_q4_K_reference 函数，传入参数 src + j, y, k
    }
    return (n/QK_K*sizeof(block_q4_K));
    // 返回结果
}

// ====================== 5-bit (de)-quantization

void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k) {
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
    # 定义一个包含32个浮点数的数组，用于存储权重数据
    float weights[32];
    # 定义一个包含32个8位无符号整数的数组，用于存储Laux数据
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
            y[i].scales[j] = MAX(-128, MIN(127, l));
        }
        // 计算 y[i].d 的值
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 d 的值
            const float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            // 如果 d 为 0，则继续下一次循环
            if (!d) continue;
            // 循环遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算 l 的值
                int l = nearest_int(x[16*j + ii]/d);
                // 将 l 限制在 -16 到 15 之间，并赋值给 L[16*j + ii]
                l = MAX(-16, MIN(15, l));
                L[16*j + ii] = l + 16;
            }
        }

        // 定义指针 qh 和 ql，分别指向 y[i].qh 和 y[i].qs
        uint8_t * restrict qh = y[i].qh;
        uint8_t * restrict ql = y[i].qs;
        // 将 qh 和 ql 的值初始化为 0
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
            // 将 l1 和 l2 的值合并，并赋值给 ql[j]
            ql[j] = l1 | (l2 << 4);
        }
#endif

        // 更新 x 的值
        x += QK_K;

    }
}

// 定义函数 dequantize_row_q5_K，参数为 restrict 类型的指针 x 和 y，以及整数 k
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k) {
    // 断言 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 定义指针 ql 和 qh，分别指向 x[i].qs 和 x[i].qh
        const uint8_t * ql = x[i].qs;
        const uint8_t * qh = x[i].qh;
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块
        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 32 位浮点数，存储在变量 d 中
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].dmin 转换为 32 位浮点数，存储在变量 min 中

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
            // 循环，计算并赋值给 y
            for (int l = 0; l < 32; ++l) *y++ = d2 * ((ql[l]  >> 4) + (qh[l] & u2 ? 16 : 0)) - m2;
            // 循环，计算并赋值给 y
            ql += 32; is += 2;
            // ql 增加 32，is 增加 2
            u1 <<= 2; u2 <<= 2;
            // u1 左移 2 位，u2 左移 2 位
        }
#else
// 如果 QK_K 不等于 256，则执行以下代码块
        float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 32 位浮点数，存储在变量 d 中
        const int8_t * restrict s = x[i].scales;
        // 声明指向常量 int8_t 类型的指针 s，指向 x[i].scales
        for (int l = 0; l < 8; ++l) {
            // 循环，l 从 0 到 7
            y[l+ 0] = d * s[0] * ((ql[l+ 0] & 0xF) - (qh[l] & 0x01 ? 0 : 16));
            // 计算并赋值给 y
            y[l+ 8] = d * s[0] * ((ql[l+ 8] & 0xF) - (qh[l] & 0x02 ? 0 : 16));
            // 计算并赋值给 y
            y[l+16] = d * s[1] * ((ql[l+16] & 0xF) - (qh[l] & 0x04 ? 0 : 16));
            // 计算并赋值给 y
            y[l+24] = d * s[1] * ((ql[l+24] & 0xF) - (qh[l] & 0x08 ? 0 : 16));
            // 计算并赋值给 y
            y[l+32] = d * s[2] * ((ql[l+ 0] >>  4) - (qh[l] & 0x10 ? 0 : 16));
            // 计算并赋值给 y
            y[l+40] = d * s[2] * ((ql[l+ 8] >>  4) - (qh[l] & 0x20 ? 0 : 16));
            // 计算并赋值给 y
            y[l+48] = d * s[3] * ((ql[l+16] >>  4) - (qh[l] & 0x40 ? 0 : 16));
            // 计算并赋值给 y
            y[l+56] = d * s[3] * ((ql[l+24] >>  4) - (qh[l] & 0x80 ? 0 : 16));
            // 计算并赋值给 y
        }
        y += QK_K;
        // y 增加 QK_K
#endif
    }
}

void quantize_row_q5_K(const float * restrict x, void * restrict vy, int k) {
    // 定义函数 quantize_row_q5_K，传入参数 x, vy, k
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    block_q5_K * restrict y = vy;
    // 声明指向 block_q5_K 类型的指针 y，指向 vy
    quantize_row_q5_K_reference(x, y, k);
    // 调用函数 quantize_row_q5_K_reference，传入参数 x, y, k
}

size_t ggml_quantize_q5_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 定义函数 ggml_quantize_q5_K，传入参数 src, dst, n, k, hist
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    (void)hist; // TODO: collect histograms
    // 忽略 hist 参数，待完成：收集直方图
    # 循环，每次增加 k 的步长，直到 j 大于等于 n
    for (int j = 0; j < n; j += k) {
        # 将目标地址转换为 block_q5_K 类型的指针，并偏移 j/QK_K 个单位
        block_q5_K * restrict y = (block_q5_K *)dst + j/QK_K;
        # 调用 quantize_row_q5_K_reference 函数，对源地址为 src + j 的数据进行量化，结果存储在 y 中
        quantize_row_q5_K_reference(src + j, y, k);
    }
    # 返回结果，计算公式为 n/QK_K 乘以 sizeof(block_q5_K)
    return (n/QK_K*sizeof(block_q5_K));
// 定义一个函数，用于将输入的浮点数数组进行6位量化，并存储到输出结构体中
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k) {
    // 断言输入的 k 值是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算输入数组 x 的长度并存储到 nb 中
    const int nb = k / QK_K;

    // 定义一个长度为 QK_K 的 int8_t 数组 L 和长度为 QK_K/16 的 float 数组 scales
    int8_t L[QK_K];
    float   scales[QK_K/16];

    // 循环处理每个块
    for (int i = 0; i < nb; i++) {

        // 初始化最大比例尺和最大绝对值比例尺
        float max_scale = 0;
        float max_abs_scale = 0;

        // 循环处理每个子块
        for (int ib = 0; ib < QK_K/16; ++ib) {

            // 调用 make_qx_quants 函数进行量化，并将结果存储到 scales 数组中
            const float scale = make_qx_quants(16, 32, x + 16*ib, L + 16*ib, 1);
            scales[ib] = scale;

            // 计算绝对值比例尺，并更新最大比例尺和最大绝对值比例尺
            const float abs_scale = fabsf(scale);
            if (abs_scale > max_abs_scale) {
                max_abs_scale = abs_scale;
                max_scale = scale;
            }

        }

        // 如果最大绝对值比例尺为 0，则将输出结构体 y[i] 清零并跳过当前循环
        if (!max_abs_scale) {
            memset(&y[i], 0, sizeof(block_q6_K));
            y[i].d = GGML_FP32_TO_FP16(0.f);
            x += QK_K;
            continue;
        }

        // 计算比例尺的倒数，并存储到输出结构体 y[i] 中
        float iscale = -128.f/max_scale;
        y[i].d = GGML_FP32_TO_FP16(1/iscale);
        // 根据比例尺的倒数和子块的量化比例尺，更新输出结构体 y[i] 中的量化比例尺
        for (int ib = 0; ib < QK_K/16; ++ib) {
            y[i].scales[ib] = MIN(127, nearest_int(iscale*scales[ib]));
        }

        // 根据输出结构体 y[i] 中的比例尺进行量化，并存储到数组 L 中
        for (int j = 0; j < QK_K/16; ++j) {
            float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            if (!d) {
                continue;
            }
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-32, MIN(31, l));
                L[16*j + ii] = l + 32;
            }
        }

        // 定义指向输出结构体 y[i] 中 ql 和 qh 数组的指针
        uint8_t * restrict ql = y[i].ql;
        uint8_t * restrict qh = y[i].qh;
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
                // 将指定位置的值右移 4 位，并与后续三个位置的值进行按位或操作，并赋值给指定位置
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
            // 将指定位置的值右移 4 位，并与后续三个位置的值进行按位或操作，并赋值给指定位置
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
        // 定义指针变量 ql，指向 x[i].ql
        const uint8_t * restrict qh = x[i].qh;
        // 定义指针变量 qh，指向 x[i].qh
        const int8_t  * restrict sc = x[i].scales;
        // 定义指针变量 sc，指向 x[i].scales
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
// 结束条件编译指令
    }
}

void quantize_row_q6_K(const float * restrict x, void * restrict vy, int k) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 限制 y 必须是 block_q6_K 类型的指针
    block_q6_K * restrict y = vy;
    // 调用 quantize_row_q6_K_reference 函数
    quantize_row_q6_K_reference(x, y, k);
}

size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 忽略 hist 参数，待实现：收集直方图
    (void)hist; // TODO: collect histograms

    for (int j = 0; j < n; j += k) {
        // 限制 y 必须是 block_q6_K 类型的指针
        block_q6_K * restrict y = (block_q6_K *)dst + j/QK_K;
        // 调用 quantize_row_q6_K_reference 函数
        quantize_row_q6_K_reference(src + j, y, k);
    }
    // 返回压缩后的数据大小
    return (n/QK_K*sizeof(block_q6_K));
}
//===================================== Q8_K ==============================================

// 对长度为 k 的输入数组 x 进行量化，结果存储在 block_q8_K 结构体数组 y 中
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
            // 如果绝对值大于绝对值最大值，则更新最大值和绝对值最大值
            if (ax > amax) {
                amax = ax; max = x[j];
            }
        }
        // 如果绝对值最大值为 0，则将当前 block_q8_K 结构体的数据置为 0，并继续下一个循环
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

// 对 block_q8_K 结构体数组 x 进行反量化，结果存储在长度为 k 的输出数组 y 中
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

// 对长度为 k 的输入数组 x 进行量化，结果存储在 block_q8_K 结构体数组 y 中
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k) {
    quantize_row_q8_K_reference(x, y, k);
}

//===================================== Dot ptoducts =================================

//
// Helper functions
//
#if __AVX__ || __AVX2__ || __AVX512F__

// shuffles to pick the required scales in dot products
static inline __m256i get_scale_shuffle_q3k(int i) {
    // 定义一个包含128个元素的常量数组，用于乱序加载数据
    static const uint8_t k_shuffle[128] = {
         0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,     2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
         4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5,     6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7,
         8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9,    10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,
        12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,    14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,
    };
    // 使用 AVX2 指令集的函数加载常量数组中的数据
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
    // 定义一个静态内联函数，返回一个__m256i类型的值，参数为整数i
    static inline __m256i get_scale_shuffle_k4(int i) {
        // 定义一个静态的uint8_t类型数组k_shuffle，包含256个元素
        static const uint8_t k_shuffle[256] = {
             // 数组元素的具体数值
        };
        // 返回一个__m256i类型的值，加载k_shuffle数组中第i个元素及其后续的256位数据
        return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
    }
    // 定义一个静态内联函数，返回一个__m128i类型的值，参数为整数i
    static inline __m128i get_scale_shuffle(int i) {
        // 定义一个静态的uint8_t类型数组k_shuffle，包含128个元素
        static const uint8_t k_shuffle[128] = {
             // 数组元素的具体数值
        };
        // 返回一个__m128i类型的值，加载k_shuffle数组中第i个元素及其后续的128位数据
        return _mm_loadu_si128((const __m128i*)k_shuffle + i);
    }
#endif

// 定义一个函数，参数为整数n、指向float类型的指针s、指向常量vx和vy的指针
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义一个整数常量qk，赋值为QK8_0
    const int qk = QK8_0;
    // 定义一个整数常量nb，赋值为n除以qk的结果
    const int nb = n / qk;

    // 断言n除以qk的余数为0，如果不成立则终止程序
    assert(n % qk == 0);

    // 定义一个指向block_q4_0类型的常量指针x，指向vx
    const block_q4_0 * restrict x = vx;
    // 定义一个指向block_q8_0类型的常量指针y，指向vy

#if defined(__ARM_NEON)
    // 定义一个float32x4_t类型的变量sumv0，赋值为4个0.0f组成的向量
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    // 创建一个包含四个浮点数的向量，并初始化为0.0
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 断言，确保nb是偶数，如果不是，需要处理奇数情况
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 定义并初始化指向x和y的指针
        const block_q4_0 * restrict x0 = &x[i + 0];
        const block_q4_0 * restrict x1 = &x[i + 1];
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个包含16个0x0F的无符号8位整数向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
        // 创建一个包含16个0x8的有符号8位整数向量
        const int8x16_t  s8b = vdupq_n_s8(0x8);

        // 从x0和x1中加载16个无符号8位整数到向量v0_0和v0_1
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        // 将4位转换为8位
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 减去8
        const int8x16_t v0_0ls = vsubq_s8(v0_0l, s8b);
        const int8x16_t v0_0hs = vsubq_s8(v0_0h, s8b);
        const int8x16_t v0_1ls = vsubq_s8(v0_1l, s8b);
        const int8x16_t v0_1hs = vsubq_s8(v0_1h, s8b);

        // 从y中加载数据到向量v1_0l, v1_0h, v1_1l, v1_1h
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

        // 计算点积并存储到int32x4_t类型的变量p_0和p_1中
        const int32x4_t p_0 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_0ls, v1_0l), v0_0hs, v1_0h);
        const int32x4_t p_1 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_1ls, v1_1l), v0_1hs, v1_1h);

        // 使用点积结果计算sumv0和sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
    }

    // 将sumv0和sumv1中的所有元素相加，并将结果存储到指针s指向的位置
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
    // 如果定义了 AVX2，初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 从 x[i].qs 中加载字节并转换为 nibbles
        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 现在我们有一个字节向量，范围在 [0 .. 15] 区间。将它们偏移为 [-8 .. +7] 区间。
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );

        // 从 y[i].qs 中加载字节
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并累加
        const __m256 q = mul_sum_i8_pairs_float(bx, by);
        acc = _mm256_fmadd_ps( d, q, acc );
    }

    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 如果定义了 AVX，初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 从 x[i].qs 中加载字节并进行位运算
        const __m128i lowMask = _mm_set1_epi8(0xF);
        const __m128i off = _mm_set1_epi8(8);
        const __m128i tmp = _mm_loadu_si128((const __m128i *)x[i].qs);
        __m128i bx = _mm_and_si128(lowMask, tmp);
        __m128i by = _mm_loadu_si128((const __m128i *)y[i].qs);
        bx = _mm_sub_epi8(bx, off);
        const __m128i i32_0 = mul_sum_i8_pairs(bx, by);

        // 从 x[i].qs 中加载字节并进行位运算
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
    // 设置常量
    const __m128i lowMask = _mm_set1_epi8(0xF);
    const __m128i off = _mm_set1_epi8(8);

    // 用零初始化累加器
    __m128 acc_0 = _mm_setzero_ps();
    __m128 acc_1 = _mm_setzero_ps();
    __m128 acc_2 = _mm_setzero_ps();
    __m128 acc_3 = _mm_setzero_ps();

    // 第一轮不累加
    }

    // 断言，确保 nb 是偶数，处理奇数的情况
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 主循环
    }

    // 计算累加器的和并存储到指定的地址
    *s = hsum_float_4x4(acc_0, acc_1, acc_2, acc_3);
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    // 根据 qk/2 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 遍历 nb 次
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
    // scalar
    // 定义一个浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 定义一个整数变量 sumi，并初始化为 0
        int sumi = 0;

        // 遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 计算 v0 和 v1
            const int v0 = (x[i].qs[j] & 0x0F) - 8;
            const int v1 = (x[i].qs[j] >>   4) - 8;

            // 计算 sumi
            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]);
        }

        // 更新 sumf
        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#endif
}

// 定义一个函数 ggml_vec_dot_q4_1_q8_1，接受参数 n, s, vx, vy
void ggml_vec_dot_q4_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk，并初始化为 QK8_1
    const int qk = QK8_1;
    // 定义常量 nb，并初始化为 n 除以 qk
    const int nb = n / qk;

    // 断言 n 除以 qk 的余数为 0
    assert(n % qk == 0);

    // 定义指向 block_q4_1 类型的指针 x，指向 vx
    const block_q4_1 * restrict x = vx;
    // 声明一个指向常量 block_q8_1 结构体的指针 y，并将其指向 vy
    const block_q8_1 * restrict y = vy;

    // 待办事项：添加 WASM SIMD（WebAssembly SIMD）支持
#if defined(__ARM_NEON)
    // 使用 NEON 指令集进行优化

    // 创建四个浮点数的向量，并初始化为0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 初始化一个浮点数用于累加
    float summs = 0;

    // 断言，确保 nb 是偶数，处理奇数的情况
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 限定指针 x0 指向 x[i+0] 的地址
        const block_q4_1 * restrict x0 = &x[i + 0];
        // 限定指针 x1 指向 x[i+1] 的地址
        const block_q4_1 * restrict x1 = &x[i + 1];
        // 限定指针 y0 指向 y[i+0] 的地址
        const block_q8_1 * restrict y0 = &y[i + 0];
        // 限定指针 y1 指向 y[i+1] 的地址
        const block_q8_1 * restrict y1 = &y[i + 1];

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x0->m) * y0->s + GGML_FP16_TO_FP32(x1->m) * y1->s;

        // 创建一个包含16个8位无符号整数的向量，每个元素都是0x0F
        const uint8x16_t m4b = vdupq_n_u8(0x0F);

        // 从地址 x0->qs 处加载16个8位无符号整数到向量 v0_0
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        // 从地址 x1->qs 处加载16个8位无符号整数到向量 v0_1
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        // 将16个8位无符号整数的向量 v0_0 中的每个元素与 m4b 中的对应元素进行按位与操作
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        // 将16个8位无符号整数的向量 v0_0 中的每个元素右移4位
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        // 将16个8位无符号整数的向量 v0_1 中的每个元素与 m4b 中的对应元素进行按位与操作
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        // 将16个8位无符号整数的向量 v0_1 中的每个元素右移4位
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 从地址 y0->qs 处加载16个8位有符号整数到向量 v1_0l
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        // 从地址 y0->qs+16 处加载16个8位有符号整数到向量 v1_0h
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        // 从地址 y1->qs 处加载16个8位有符号整数到向量 v1_1l
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        // 从地址 y1->qs+16 处加载16个8位有符号整数到向量 v1_1h
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

        // 计算点积并存储到 int32x4_t 类型的变量 p_0
        const int32x4_t p_0 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_0l, v1_0l), v0_0h, v1_0h);
        // 计算点积并存储到 int32x4_t 类型的变量 p_1
        const int32x4_t p_1 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_1l, v1_1l), v0_1h, v1_1h);

        // 使用点积结果更新 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*y1->d);
    }

    // 将 sumv0 和 sumv1 中的所有元素相加，并将结果存储到 s 中
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs;
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果支持 AVX2 或 AVX 指令集，则使用 AVX 进行优化

    // 使用 AVX 指令集进行优化
    // 初始化一个包含8个单精度浮点数的向量，并初始化为0
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数用于累加
    float summs = 0;

    // 主循环
    // 循环遍历nb次，i从0到nb-1
    for (int i = 0; i < nb; ++i) {
        // 将x[i]的d值从FP16转换为FP32
        const float d0 = GGML_FP16_TO_FP32(x[i].d);
        // 获取y[i]的d值
        const float d1 = y[i].d;

        // 计算summs的累加值，将x[i]的m值从FP16转换为FP32，然后与y[i]的s值相乘
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将d0值转换为256位的向量
        const __m256 d0v = _mm256_set1_ps( d0 );
        // 将d1值转换为256位的向量
        const __m256 d1v = _mm256_set1_ps( d1 );

        // 计算d0v和d1v的乘积
        const __m256 d0d1 = _mm256_mul_ps( d0v, d1v );

        // 从x[i]的qs中加载16个字节，并将4个位字段解压缩成字节，生成32个字节
        const __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从y[i]的qs中加载16个字节
        const __m256i by = _mm256_loadu_si256( (const __m256i *)y[i].qs );

        // 计算bx和by的乘积，并将结果相加
        const __m256 xy = mul_sum_us8_pairs_float(bx, by);

        // 累加d0*d1*x*y的结果
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
    // 如果支持 RISC-V 向量指令集，则使用向量指令进行计算
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
    // 如果不支持向量指令集，则使用标量方式进行计算
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
    
    # 声明并初始化指向 vx 的指针 x，限定其只能指向 block_q5_0 类型的数据
    const block_q5_0 * restrict x = vx;
    # 声明并初始化指向 vy 的指针 y，限定其只能指向 block_q8_0 类型的数据
    const block_q8_0 * restrict y = vy;
#if defined(__ARM_NEON)
    // 使用 NEON 指令集，初始化两个包含四个单精度浮点数的向量，每个元素都是 0.0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 用于存储结果的临时变量
    uint32_t qh0;
    uint32_t qh1;

    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言，确保 nb 是偶数
    assert(nb % 2 == 0); // TODO: handle odd nb

    }

    // 将两个向量的元素相加，并将结果存储到指针 s 指向的位置
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__wasm_simd128__)
    // 使用 SIMD 指令集，初始化包含四个单精度浮点数的向量，每个元素都是 0.0
    v128_t sumv = wasm_f32x4_splat(0.0f);

    // 用于存储结果的临时变量
    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }

    // 将向量的元素相加，并将结果存储到指针 s 指向的位置
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3);
#elif defined(__AVX2__)
    // 使用 AVX2 指令集，初始化包含八个单精度浮点数的向量，每个元素都是 0.0
    __m256 acc = _mm256_setzero_ps();

    // 主循环，遍历 nb 次
    for (int i = 0; i < nb; i++) {
        /* 计算块的组合比例 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        bxhi = _mm256_andnot_si256(bxhi, _mm256_set1_epi8((char)0xF0));
        bx = _mm256_or_si256(bx, bxhi);

        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 将 q 乘以比例并累加 */
        acc = _mm256_fmadd_ps(d, q, acc);
    }

    // 将向量的元素水平相加，并将结果存储到指针 s 指向的位置
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 使用 AVX 指令集，初始化包含八个单精度浮点数的向量，每个元素都是 0.0
    __m256 acc = _mm256_setzero_ps();
    __m128i mask = _mm_set1_epi8((char)0xF0);

    // 主循环
    for (int i = 0; i < nb; i++) {
        /* 对块计算组合比例 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        __m256i bx = bytes_from_nibbles_32(x[i].qs);
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

        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 用比例乘以 q 并累加 */
        acc = _mm256_add_ps(_mm256_mul_ps(d, q), acc);
    }

    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    # 定义浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    # 定义无符号整型变量 qh
    uint32_t qh;

    # 调用 __riscv_vsetvl_e8m1 函数，计算向量长度并赋值给变量 vl
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    # 定义临时寄存器 vt_1 和 vt_2，用于掩码和移位操作
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vsll_vv_u32m2(__riscv_vmv_v_x_u32m2(1, vl), vt_1, vl);

    # 定义临时寄存器 vt_3 和 vt_4，进行移位和加法操作
    vuint32m2_t vt_3 = __riscv_vsll_vx_u32m2(vt_2, 16, vl);
    vuint32m2_t vt_4 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    }

    *s = sumf;
#else
    # scalar
    # 定义浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    # 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        # 定义无符号整型变量 qh，并从 x[i].qh 复制数据
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 定义整型变量 sumi，并初始化为 0
        int sumi = 0;

        # 循环遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            # 计算 xh_0 和 xh_1
            const uint8_t xh_0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
            const uint8_t xh_1 = ((qh & (1u << (j + 16))) >> (j + 12));

            # 计算 x0 和 x1
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            # 计算 sumi
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }

        # 计算 sumf
        sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;
    }

    # 将 sumf 赋值给 *s
    *s = sumf;
#endif
}

# 定义 ggml_vec_dot_q5_1_q8_1 函数
void ggml_vec_dot_q5_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 定义常量 qk，并赋值为 QK8_1
    const int qk = QK8_1;
    # 定义常量 nb，并赋值为 n / qk
    const int nb = n / qk;

    # 断言 n 除以 qk 的余数为 0
    assert(n % qk == 0);
    # 断言 qk 等于 QK5_1
    assert(qk == QK5_1);

    # 定义指向 block_q5_1 结构体的指针 x，并赋值为 vx
    const block_q5_1 * restrict x = vx;
    # 定义指向 block_q8_1 结构体的指针 y，并赋值为 vy

#if defined(__ARM_NEON)
    # 定义四个浮点数向量变量 sumv0 和 sumv1，并初始化为 0.0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    # 定义两个浮点数变量 summs0 和 summs1，并初始化为 0.0
    float summs0 = 0.0f;
    float summs1 = 0.0f;

    # 定义两个无符号整型变量 qh0 和 qh1

    # 定义两个长度为 4 的无符号长整型数组 tmp0 和 tmp1

    # 断言 nb 除以 2 的余数为 0

    # 将 sumv0 和 sumv1 的累加结果加上 summs0 和 summs1，赋值给 *s
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs0 + summs1;
#elif defined(__wasm_simd128__)
    # 定义浮点数向量变量 sumv，并初始化为 0.0
    v128_t sumv = wasm_f32x4_splat(0.0f);

    # 定义浮点数变量 summs，并初始化为 0.0

    # 定义无符号整型变量 qh
    uint32_t qh;
    # 定义长度为 4 的无符号长整型数组 tmp

    # TODO: check if unrolling this is better
    # 计算 sumv 中的四个元素的和，加上 summs 的值，赋给指针 s
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3) + summs;
#elif defined(__AVX2__)
    // 使用AVX2指令集进行计算

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();

    // 初始化浮点数变量summs
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将x[i].d转换为单精度浮点数，并使用AVX2指令集进行操作
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算GGML_FP16_TO_FP32(x[i].m) * y[i].s，并累加到summs中
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将x[i].qs转换为字节流，并进行位操作
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        bxhi = _mm256_and_si256(bxhi, _mm256_set1_epi8(0x10));
        bx = _mm256_or_si256(bx, bxhi);

        // 将y[i].d转换为单精度浮点数，并使用AVX2指令集进行操作
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从y[i].qs加载数据到by中
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算mul_sum_us8_pairs_float的结果，并使用FMA指令进行累加
        const __m256 q = mul_sum_us8_pairs_float(bx, by);
        acc = _mm256_fmadd_ps(q, _mm256_mul_ps(dx, dy), acc);
    }

    // 将acc的8个单精度浮点数求和，并加上summs，结果赋值给*s
    *s = hsum_float_8(acc) + summs;
#elif defined(__AVX__)
    // 使用AVX指令集进行计算

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();
    // 设置掩码
    __m128i mask = _mm_set1_epi8(0x10);

    // 初始化浮点数变量summs
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将x[i].d转换为单精度浮点数，并使用AVX指令集进行操作
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算GGML_FP16_TO_FP32(x[i].m) * y[i].s，并累加到summs中
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将x[i].qs转换为字节流，并进行位操作
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        bxhil = _mm_and_si128(bxhil, mask);
        bxhih = _mm_and_si128(bxhih, mask);
        __m128i bxl = _mm256_castsi256_si128(bx);
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        bxl = _mm_or_si128(bxl, bxhil);
        bxh = _mm_or_si128(bxh, bxhih);
        bx = MM256_SET_M128I(bxh, bxl);

        // 将y[i].d转换为单精度浮点数，并使用AVX指令集进行操作
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从y[i].qs加载数据到by中
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算mul_sum_us8_pairs_float的结果，并使用FMA指令进行累加
        acc = _mm256_add_ps(_mm256_mul_ps(mul_sum_us8_pairs_float(bx, by), _mm256_mul_ps(dx, dy)), acc);
    }

    // 将acc的8个单精度浮点数求和，并加上summs，结果赋值给*s
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数变量 sumf，初始化为 0.0
    float sumf = 0.0;

    // 定义一个无符号整型变量 qh
    uint32_t qh;

    // 调用 __riscv_vsetvl_e8m1 函数计算 vl 的值
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 为移位操作定义临时寄存器
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);
    // 循环遍历从 0 到 nb-1 的整数
    for (int i = 0; i < nb; i++) {
        // 将 x[i].qh 的内容拷贝到 qh 中，拷贝的长度为 uint32_t 的大小
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

        // 将 xha_0 向量转换为 16 位有符号整数
        vuint16m1_t xhc_0 = __riscv_vncvt_x_x_w_u16m1(xha_0, vl);
        // 将 xhc_0 向量转换为 8 位有符号整数
        vuint8mf2_t xh_0 = __riscv_vncvt_x_x_w_u8mf2(xhc_0, vl);

        // 将 xha_1 向量转换为 16 位有符号整数
        vuint16m1_t xhc_1 = __riscv_vncvt_x_x_w_u16m1(xha_1, vl);
        // 将 xhc_1 向量转换为 8 位有符号整数
        vuint8mf2_t xh_1 = __riscv_vncvt_x_x_w_u8mf2(xhc_1, vl);

        // 加载 x[i].qs 的内容到向量寄存器 tx 中
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        // 加载 y[i].qs 的内容到向量寄存器 y0 和 y1 中
        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 计算 x[i].qs & 0x0F
        vuint8mf2_t x_at = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        // 计算 x[i].qs >> 0x04
        vuint8mf2_t x_lt = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        // 计算 x_a = x_at | xh_0
        vuint8mf2_t x_a = __riscv_vor_vv_u8mf2(x_at, xh_0, vl);
        // 计算 x_l = x_lt | xh_1
        vuint8mf2_t x_l = __riscv_vor_vv_u8mf2(x_lt, xh_1, vl);

        // 将 x_a 向量转换为 8 位有符号整数
        vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        // 将 x_l 向量转换为 8 位有符号整数
        vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        // 计算 v0 和 y0 的乘积
        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        // 计算 v1 和 y1 的乘积
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        // 将 0 转换为 32 位有符号整数
        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        // 对 vec_mul1 进行累加求和
        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        // 对 vec_mul2 进行累加求和
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        // 将 vs2 转换为标量整数
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        // 计算 sumf 的累加值
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
#else
    // 如果不是 ARM 平台，则使用标量计算
    float sumf = 0.0;  // 初始化浮点数求和变量

    for (int i = 0; i < nb; i++) {  // 遍历 nb 次
        uint32_t qh;  // 定义无符号 32 位整数变量 qh
        memcpy(&qh, x[i].qh, sizeof(qh));  // 将 x[i].qh 的内容复制到 qh

        int sumi = 0;  // 初始化整数求和变量

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
    float32x4_t sumv0 = vdupq_n_f32(0.0f);  // 使用 NEON 指令初始化 sumv0
    float32x4_t sumv1 = vdupq_n_f32(0.0f);  // 使用 NEON 指令初始化 sumv1

    assert(nb % 2 == 0); // 断言 nb 是偶数，处理奇数情况
    // 循环遍历数组，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 定义指向 x 和 y 数组中元素的指针
        const block_q8_0 * restrict x0 = &x[i + 0];
        const block_q8_0 * restrict x1 = &x[i + 1];
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 从 x0 和 x1 中加载数据到 int8x16_t 类型的变量
        const int8x16_t x0_0 = vld1q_s8(x0->qs);
        const int8x16_t x0_1 = vld1q_s8(x0->qs + 16);
        const int8x16_t x1_0 = vld1q_s8(x1->qs);
        const int8x16_t x1_1 = vld1q_s8(x1->qs + 16);

        // 从 y0 和 y1 中加载数据到 int8x16_t 类型的变量
        const int8x16_t y0_0 = vld1q_s8(y0->qs);
        const int8x16_t y0_1 = vld1q_s8(y0->qs + 16);
        const int8x16_t y1_0 = vld1q_s8(y1->qs);
        const int8x16_t y1_1 = vld1q_s8(y1->qs + 16);

        // 计算 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        ggml_vdotq_s32(vdupq_n_s32(0), x0_0, y0_0),
                        ggml_vdotq_s32(vdupq_n_s32(0), x0_1, y0_1))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));

        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        ggml_vdotq_s32(vdupq_n_s32(0), x1_0, y1_0),
                        ggml_vdotq_s32(vdupq_n_s32(0), x1_1, y1_1))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
    }

    // 将 sumv0 和 sumv1 中的所有元素相加，然后将两个结果相加并存储到指针 s 指向的位置
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果编译器支持 AVX2 或 AVX 指令集，则初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));
        __m256i bx = _mm256_loadu_si256((const __m256i *)x[i].qs);
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加
#if defined(__AVX2__)
        acc = _mm256_fmadd_ps( d, q, acc );
#else
        acc = _mm256_add_ps( _mm256_mul_ps( d, q ), acc );
#endif
    }

    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 如果编译器支持 RISC-V 指令集，则初始化浮点数和向量长度
    float sumf = 0.0;
    size_t vl = __riscv_vsetvl_e8m1(qk);

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vint8m1_t bx = __riscv_vle8_v_i8m1(x[i].qs, vl);
        vint8m1_t by = __riscv_vle8_v_i8m1(y[i].qs, vl);

        vint16m2_t vw_mul = __riscv_vwmul_vv_i16m2(bx, by, vl);

        vint32m1_t v_zero = __riscv_vmv_v_x_i32m1(0, vl);
        vint32m1_t v_sum = __riscv_vwredsum_vs_i16m2_i32m1(vw_mul, v_zero, vl);

        int sumi = __riscv_vmv_x_s_i32m1_i32(v_sum);

        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
#else
    // 标量
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
    // 如果目标平台支持 ARM NEON 指令集，则创建一个包含 16 个相同值的向量
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    // 创建一个包含16个相同值的无符号8位整数向量
    const uint8x16_t m4 = vdupq_n_u8(0xF);

    // 创建一个包含4个相同值的有符号32位整数向量
    const int32x4_t vzero = vdupq_n_s32(0);

    // 创建一个包含两个16位整数向量的自定义数据类型
    ggml_int8x16x2_t q2bytes;
    // 创建一个长度为16的无符号8位整数数组
    uint8_t aux[16];

    // 初始化浮点数变量 sum
    float sum = 0;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 计算 d 和 dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 获取指向 x[i].qs, y[i].qs, x[i].scales 的指针
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;
        const uint8_t * restrict sc = x[i].scales;

        // 从指定地址加载16个无符号8位整数到向量 mins_and_scales
        const uint8x16_t mins_and_scales = vld1q_u8(sc);
        // 从 mins_and_scales 中提取最低4位并存储到向量 scales 和数组 aux 中
        const uint8x16_t scales = vandq_u8(mins_and_scales, m4);
        vst1q_u8(aux, scales);

        // 从 mins_and_scales 中右移4位并存储到向量 mins
        const uint8x16_t mins = vshrq_n_u8(mins_and_scales, 4);
        // 从 y[i].bsums 加载16个16位整数到两个自定义数据类型 ggml_int16x8x2_t 中
        const ggml_int16x8x2_t q8sums = ggml_vld1q_s16_x2(y[i].bsums);
        // 将 mins 转换为16位整数并存储到两个16位整数向量中
        const ggml_int16x8x2_t mins16 = {{vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(mins))), vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(mins)))}};
        // 计算 s0 和 s1
        const int32x4_t s0 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[0]), vget_low_s16 (q8sums.val[0])),
                                       vmull_s16(vget_high_s16(mins16.val[0]), vget_high_s16(q8sums.val[0])));
        const int32x4_t s1 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[1]), vget_low_s16 (q8sums.val[1])),
                                       vmull_s16(vget_high_s16(mins16.val[1]), vget_high_s16(q8sums.val[1])));
        // 更新 sum
        sum += dmin * vaddvq_s32(vaddq_s32(s0, s1));

        // 初始化 isum 和 is
        int isum = 0;
        int is = 0;
// 定义宏，用于将乘积累加到isum中，同时乘以aux数组中对应位置的值
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * aux[is+(index)];\
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * aux[is+1+(index)];

// 定义宏，用于对q8bytes进行位移、乘积累加和乘以aux数组中对应位置的值
#define SHIFT_MULTIPLY_ACCUM_WITH_SCALE(shift, index)\
        q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;\
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[0], (shift)), m3));\
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[1], (shift)), m3));\
        MULTIPLY_ACCUM_WITH_SCALE((index));

// 循环遍历QK_K/128次
for (int j = 0; j < QK_K/128; ++j) {
    // 从q2中加载数据到q2bits
    const ggml_uint8x16x2_t q2bits = ggml_vld1q_u8_x2(q2); q2 += 32;

    // 从q8中加载数据到q8bytes
    ggml_int8x16x2_t q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
    // 对q2bits进行位与运算，并将结果转换为int8类型，存入q2bytes
    q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[0], m3));
    q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[1], m3));

    // 对index为0的位置进行乘积累加
    MULTIPLY_ACCUM_WITH_SCALE(0);

    // 对index为2、4、6的位置进行位移、乘积累加和乘以aux数组中对应位置的值
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(2, 2);
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(4, 4);
    SHIFT_MULTIPLY_ACCUM_WITH_SCALE(6, 6);

    // is增加8
    is += 8;
}

// 将d乘以isum并累加到sum中
sum += d * isum;

// 如果定义了__AVX2__，则执行以下代码
#elif defined __AVX2__

    // 定义常量m3和m4
    const __m256i m3 = _mm256_set1_epi8(3);
    const __m128i m4 = _mm_set1_epi8(0xF);

    // 初始化acc为全0
    __m256 acc = _mm256_setzero_ps();

    // 计算acc的和并存入*s中
    *s = hsum_float_8(acc);

// 如果定义了__AVX__，则执行以下代码
#elif defined __AVX__

    // 定义常量m3、m4和m2
    const __m128i m3 = _mm_set1_epi8(0x3);
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(0x2);

    // 初始化acc为全0
    __m256 acc = _mm256_setzero_ps();

    // 计算acc的和并存入*s中
    *s = hsum_float_8(acc);

// 如果定义了__riscv_v_intrinsic，则执行以下代码
#elif defined __riscv_v_intrinsic

    // 初始化sumf为0
    float sumf = 0;
    // 初始化temp_01数组
    uint8_t temp_01[32] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                            1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1};

    // ...
    # 将指针 *s 指向函数 sumf
#else
// 如果不是 ARM_NEON 架构，则执行以下代码

    float sumf = 0; // 初始化浮点数 sumf 为 0

    for (int i = 0; i < nb; ++i) { // 遍历 nb 次，nb 为 n/QK_K 的整数部分

        const uint8_t * q2 = x[i].qs; // 获取 x[i] 的 qs 属性，赋值给指针 q2
        const  int8_t * q8 = y[i].qs; // 获取 y[i] 的 qs 属性，赋值给指针 q8
        const uint8_t * sc = x[i].scales; // 获取 x[i] 的 scales 属性，赋值给指针 sc

        int summs = 0; // 初始化整数 summs 为 0
        for (int j = 0; j < 16; ++j) { // 遍历 16 次
            summs += y[i].bsums[j] * (sc[j] >> 4); // 计算 summs 的值
        }

        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d); // 计算 dall 的值
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin); // 计算 dmin 的值

        int isum = 0; // 初始化整数 isum 为 0
        int is = 0; // 初始化整数 is 为 0
        int d; // 声明整数变量 d
        for (int k = 0; k < QK_K/128; ++k) { // 遍历 QK_K/128 次
            int shift = 0; // 初始化整数 shift 为 0
            for (int j = 0; j < 4; ++j) { // 遍历 4 次
                d = sc[is++] & 0xF; // 计算 d 的值
                int isuml = 0; // 初始化整数 isuml 为 0
                for (int l =  0; l < 16; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3); // 计算 isuml 的值
                isum += d * isuml; // 更新 isum 的值
                d = sc[is++] & 0xF; // 计算 d 的值
                isuml = 0; // 重置 isuml 为 0
                for (int l = 16; l < 32; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3); // 计算 isuml 的值
                isum += d * isuml; // 更新 isum 的值
                shift += 2; // shift 增加 2
                q8 += 32; // q8 指针向后移动 32 个位置
            }
            q2 += 32; // q2 指针向后移动 32 个位置
        }
        sumf += dall * isum - dmin * summs; // 更新 sumf 的值
    }
    *s = sumf; // 将 sumf 的值赋给指针 s
#endif
}

#else
// 如果不是 ARM_NEON 架构，则执行以下代码

void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 函数定义
    const block_q2_K * restrict x = vx; // 获取 vx 的值，赋给指针 x
    const block_q8_K * restrict y = vy; // 获取 vy 的值，赋给指针 y

    const int nb = n / QK_K; // 计算 nb 的值，n 除以 QK_K

#ifdef __ARM_NEON
    // 如果是 ARM_NEON 架构，则执行以下代码
    const uint8x16_t m3 = vdupq_n_u8(0x3); // 初始化 m3 为 16 个 0x3 组成的向量

    const int32x4_t vzero = vdupq_n_s32(0); // 初始化 vzero 为 4 个 0 组成的向量

    ggml_int8x16x4_t q2bytes; // 声明 ggml_int8x16x4_t 类型的变量 q2bytes

    uint32_t aux32[2]; // 声明长度为 2 的 uint32_t 类型数组 aux32
    const uint8_t * scales = (const uint8_t *)aux32; // 将 aux32 强制转换为 uint8_t 类型的指针，赋给 scales

    float sum = 0; // 初始化浮点数 sum 为 0
    // 遍历循环，从 i=0 开始，直到 i<nb 为止
    for (int i = 0; i < nb; ++i) {

        // 计算 d 值，y[i].d 乘以 x[i].d
        const float d = y[i].d * (float)x[i].d;
        // 计算 dmin 值，-y[i].d 乘以 x[i].dmin
        const float dmin = -y[i].d * (float)x[i].dmin;

        // 限定指针 q2 指向的内存区域，只能通过 q2 访问
        const uint8_t * restrict q2 = x[i].qs;
        // 限定指针 q8 指向的内存区域，只能通过 q8 访问
        const int8_t  * restrict q8 = y[i].qs;
        // 将 x[i].scales 强制转换为 const uint32_t 指针，并限定只能通过 sc 访问
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;

        // 将 sc[0] 的低 4 位与 0x0f0f0f0f 进行按位与操作，存入 aux32[0]
        aux32[0] = sc[0] & 0x0f0f0f0f;
        // 将 sc[0] 右移 4 位后的低 4 位与 0x0f0f0f0f 进行按位与操作，存入 aux32[1]
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算 sum 值，加上 dmin 乘以一系列值的和
        sum += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);

        // 初始化 isum1 和 isum2 为 0
        int isum1 = 0, isum2 = 0;

        // 从 q2 指向的内存区域加载 16 个 uint8_t 类型的值，存入 q2bits
        const uint8x16_t q2bits = vld1q_u8(q2);

        // 从 q8 指向的内存区域加载 4 组 int8_t 类型的值，存入 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 q2bits 进行位与操作，并存入 q2bytes 的不同 val 中
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits, m3));
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 2), m3));
        q2bytes.val[2] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 4), m3));
        q2bytes.val[3] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 6), m3));

        // 计算 isum1 和 isum2 的值
        isum1 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * scales[0];
        isum2 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * scales[1];
        isum1 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[2], q8bytes.val[2])) * scales[2];
        isum2 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[3], q8bytes.val[3])) * scales[3];

        // 计算 sum 的值，加上 d 乘以 isum1 和 isum2 的和
        sum += d * (isum1 + isum2);
    }

    // 将 sum 的值赋给指针 s 指向的内存区域
    *s = sum;
#elif defined __AVX2__
    # 如果定义了 AVX2，执行以下代码

    const __m256i m3 = _mm256_set1_epi8(3);
    # 创建一个包含 8 个 3 的 AVX 寄存器

    __m256 acc = _mm256_setzero_ps();
    # 创建一个全零的 AVX 寄存器

    uint32_t ud, um;
    # 声明两个 32 位无符号整数变量

    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;
    # 声明两个指向 ud 和 um 的无符号 8 位整数指针

    float summs = 0;
    # 声明一个浮点数变量并初始化为 0

    // TODO: optimize this
    # 待优化的部分，暂时没有实现
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
        # 从 sc 中提取 ud 和 um 值
        ud = (sc[0] >> 0) & 0x0f0f0f0f;
        um = (sc[0] >> 4) & 0x0f0f0f0f;

        # 计算 smin 值
        int32_t smin = mb[0] * y[i].bsums[0] + mb[1] * y[i].bsums[1] + mb[2] * y[i].bsums[2] + mb[3] * y[i].bsums[3];
        # 计算 summs 值
        summs += dmin * smin;

        # 加载 q2 中的数据到 __m128i 类型的寄存器 q2bits
        const __m128i q2bits = _mm_loadu_si128((const __m128i*)q2);
        # 对 q2bits 进行位运算，得到 q2_0 和 q2_1
        const __m256i q2_0 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 2), q2bits), m3);
        const __m256i q2_1 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 6), _mm_srli_epi16(q2bits, 4)), m3);

        # 加载 q8 中的数据到 __m256i 类型的寄存器 q8_0 和 q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 对 q2_0 和 q8_0 进行乘法运算，得到 p0
        const __m256i p0 = _mm256_maddubs_epi16(q2_0, q8_0);
        # 对 q2_1 和 q8_1 进行乘法运算，得到 p1
        const __m256i p1 = _mm256_maddubs_epi16(q2_1, q8_1);

        # 将 p0 和 p1 转换为 32 位整数类型
        const __m256i p_0 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 0));
        const __m256i p_1 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 1));
        const __m256i p_2 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 0));
        const __m256i p_3 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 1));

        # 使用乘法和加法运算，更新 acc 寄存器的值
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[0]), _mm256_cvtepi32_ps(p_0), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[1]), _mm256_cvtepi32_ps(p_1), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[2]), _mm256_cvtepi32_ps(p_2), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[3]), _mm256_cvtepi32_ps(p_3), acc);
    }

    # 计算最终结果并赋值给 s
    *s = hsum_float_8(acc) + summs;
#elif defined __AVX__
    # 如果定义了 AVX，执行以下代码

    const __m128i m3 = _mm_set1_epi8(3);
    # 创建一个包含全部元素为3的128位整数向量

    __m256 acc = _mm256_setzero_ps();
    # 创建一个包含全部元素为0的256位单精度浮点数向量

    uint32_t ud, um;
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;
    # 定义两个32位无符号整数和对应的指针

    float summs = 0;
    # 初始化一个浮点数变量summs为0

    // TODO: optimize this
    # 待优化的部分

    }
    # 结束当前代码块

    *s = hsum_float_8(acc) + summs;
    # 将acc向量的元素求和并加上summs，结果赋值给指针s指向的变量

#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，执行以下代码

    uint32_t aux32[2];
    const uint8_t * scales = (const uint8_t *)aux32;
    # 定义一个包含两个32位无符号整数的数组aux32，并定义一个指向该数组的指针scales

    float sumf = 0;
    # 初始化一个浮点数变量sumf为0

    }
    # 结束当前代码块

    *s = sumf;
    # 将sumf的值赋给指针s指向的变量

#else
    # 如果以上条件都不满足，执行以下代码

    float sumf = 0;
    # 初始化一个浮点数变量sumf为0

    int isum[4];
    # 定义一个包含4个整数的数组isum

    for (int i = 0; i < nb; ++i) {
        # 循环，i从0到nb-1

        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;
        # 定义三个指针分别指向x[i].qs, y[i].qs, x[i].scales

        int summs = 0;
        # 初始化一个整数变量summs为0

        for (int j = 0; j < QK_K/16; ++j) {
            # 循环，j从0到QK_K/16-1
            summs += y[i].bsums[j] * (sc[j] >> 4);
            # 计算summs的值
        }

        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
        # 计算dall和dmin的值

        isum[0] = isum[1] = isum[2] = isum[3] = 0;
        # 将isum数组的所有元素初始化为0

        for (int l =  0; l < 16; ++l) {
            # 循环，l从0到15
            isum[0] += q8[l+ 0] * ((q2[l] >> 0) & 3);
            isum[1] += q8[l+16] * ((q2[l] >> 2) & 3);
            isum[2] += q8[l+32] * ((q2[l] >> 4) & 3);
            isum[3] += q8[l+48] * ((q2[l] >> 6) & 3);
            # 计算isum数组的值
        }
        for (int l = 0; l < 4; ++l) {
            # 循环，l从0到3
            isum[l] *= (sc[l] & 0xF);
            # 计算isum数组的值
        }
        sumf += dall * (isum[0] + isum[1] + isum[2] + isum[3]) - dmin * summs;
        # 计算sumf的值
    }
    *s = sumf;
    # 将sumf的值赋给指针s指向的变量
#endif
}
#endif

#if QK_K == 256
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 如果QK_K等于256，执行以下代码
    assert(n % QK_K == 0);
    # 断言n能被QK_K整除

    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;
    # 定义两个32位无符号整数kmask1和kmask2

    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;
    # 定义两个指针分别指向vx和vy

    const int nb = n / QK_K;
    # 计算nb的值为n除以QK_K的结果

#ifdef __ARM_NEON
    # 如果定义了ARM_NEON，执行以下代码

    uint32_t aux[3];
    uint32_t utmp[4];
    # 定义两个数组aux和utmp

    const uint8x16_t m3b = vdupq_n_u8(0x3);
    const int32x4_t  vzero = vdupq_n_s32(0);
    # 定义两个NEON向量m3b和vzero
    # 创建一个包含16个8位无符号整数1的向量
    const uint8x16_t m0 = vdupq_n_u8(1);
    # 将向量m0中的每个元素左移1位，得到新的向量m1
    const uint8x16_t m1 = vshlq_n_u8(m0, 1);
    # 将向量m0中的每个元素左移2位，得到新的向量m2
    const uint8x16_t m2 = vshlq_n_u8(m0, 2);
    # 将向量m0中的每个元素左移3位，得到新的向量m3
    const uint8x16_t m3 = vshlq_n_u8(m0, 3);
    # 创建一个包含32的8位有符号整数
    const int8_t m32 = 32;

    # 创建一个包含4个16字节的整数向量的结构体
    ggml_int8x16x4_t q3bytes;

    # 初始化一个浮点数sum为0
    float sum = 0;

    # 结束了一个大括号，可能是代码块的结束

    # 将sum的值赋给指针s所指向的位置
    *s = sum;
#elif defined __AVX2__
    // 设置一个包含8个元素的常量向量，每个元素都是3
    const __m256i m3 = _mm256_set1_epi8(3);
    // 设置一个包含8个元素的常量向量，每个元素都是1
    const __m256i mone = _mm256_set1_epi8(1);
    // 设置一个包含8个元素的常量向量，每个元素都是32
    const __m128i m32 = _mm_set1_epi8(32);
    // 设置一个包含8个元素的浮点型常量向量，每个元素都是0
    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含3个元素的无符号整型数组
    uint32_t aux[3];
    // 计算8个浮点数的和，并将结果存储到指定的地址
    *s = hsum_float_8(acc);
#elif defined __AVX__
    // 设置一个包含4个元素的常量向量，每个元素都是3
    const __m128i m3 = _mm_set1_epi8(3);
    // 设置一个包含4个元素的常量向量，每个元素都是1
    const __m128i mone = _mm_set1_epi8(1);
    // 设置一个包含4个元素的常量向量，每个元素都是32
    const __m128i m32 = _mm_set1_epi8(32);
    // 设置一个包含4个元素的常量向量，每个元素都是2
    const __m128i m2 = _mm_set1_epi8(2);
    // 设置一个包含8个元素的浮点型常量向量，每个元素都是0
    __m256 acc = _mm256_setzero_ps();
    // 创建一个指向无符号整型数组的指针
    const uint32_t *aux;
    // 计算8个浮点数的和，并将结果存储到指定的地址
    *s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    // 创建一个包含3个元素的无符号整型数组
    uint32_t aux[3];
    // 创建一个包含4个元素的无符号整型数组
    uint32_t utmp[4];
    // 初始化一个浮点数的和为0
    float sumf = 0;
    // 将计算的和存储到指定的地址
    *s = sumf;
#else
    // scalar version
    // This function is written like this so the compiler can manage to vectorize most of it
    // Using -Ofast, GCC and clang manage to produce code that is within a factor of 2 or so from the
    // manually vectorized version above. Every other version I tried would run at least 4 times slower.
    // The ideal situation would be if we could just write the code once, and the compiler would
    // automatically produce the best possible set of machine instructions, instead of us having to manually
    // write vectorized versions for AVX, ARM_NEON, etc.
    // 初始化一个包含QK_K个元素的有符号8位整型数组
    int8_t  aux8[QK_K];
    // 初始化一个包含8个元素的有符号16位整型数组
    int16_t aux16[8];
    // 初始化一个包含8个元素的浮点数数组，每个元素都是0
    float   sums [8];
    // 初始化一个包含8个元素的有符号32位整型数组，每个元素都是0
    int32_t aux32[8];
    // 将指定地址开始的8个浮点数的值都设置为0
    memset(sums, 0, 8*sizeof(float));
    // 初始化一个包含4个元素的无符号整型数组
    uint32_t auxs[4];
    // 创建一个指向有符号8位整型数组的指针，指向auxs数组
    const int8_t * scales = (const int8_t*)auxs;
    // 初始化一个浮点数的和为0
    float sumf = 0;
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
        // 循环，j 从 0 到 QK_K-1，每次增加 128
        for (int j = 0; j < QK_K; j += 128) {
            // 将 q3 中的每个元素与 3 进行按位与操作，赋值给 a 中的前 32 个元素
            for (int l = 0; l < 32; ++l) a[l] = q3[l] & 3;
            // 根据 hm 中的每个元素是否与 m 位相与，决定是否对 a 中的前 32 个元素进行减法操作
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            // 指针 a 后移 32 位，m 左移 1 位
            a += 32; m <<= 1;
            // 重复上述操作，直到 j 达到 QK_K
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            // ...
            //
#endif
// 结束条件编译指令

}
// 结束 else 语句块

#else
// 如果不满足条件编译指令，则执行以下代码

void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    const int nb = n / QK_K;
    // 定义并初始化条件编译指令

#ifdef __ARM_NEON
    // 如果是 ARM 平台，则执行以下代码
    const int32x4_t vzero = vdupq_n_s32(0);

    const uint8x16_t m3b = vdupq_n_u8(0x3);
    const uint8x16_t mh  = vdupq_n_u8(4);

    ggml_int8x16x4_t q3bytes;

    uint16_t aux16[2];
    int8_t * scales = (int8_t *)aux16;

    float sum = 0;
    # 遍历循环，从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {

        # 定义一个包含 4 个 uint8x16_t 类型的结构体变量 q3h
        ggml_uint8x16x4_t q3h;

        # 从 x[i].hmask 中加载 8 个 uint8_t 类型的数据到 hbits
        const uint8x8_t  hbits    = vld1_u8(x[i].hmask);
        # 从 x[i].qs 中加载 16 个 uint8_t 类型的数据到 q3bits
        const uint8x16_t q3bits   = vld1q_u8(x[i].qs);
        # 从 y[i].qs 中加载 64 个 int8_t 类型的数据到 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(y[i].qs);

        # 从 x[i].scales 中加载 16 位数据到 a
        const uint16_t a = *(const uint16_t *)x[i].scales;
        # 将 a 的低 8 位和高 8 位分别存储到 aux16 数组中
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        # 对 scales 数组中的每个元素减去 8
        for (int j = 0; j < 4; ++j) scales[j] -= 8;

        # 计算 isum 的值
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        # 计算 d 的值
        const float d = y[i].d * (float)x[i].d;

        # 对 hbits 进行位移和逻辑运算，得到 htmp
        const uint8x16_t htmp = vcombine_u8(hbits, vshr_n_u8(hbits, 1));
        # 根据 htmp 计算 q3h 的 4 个元素值
        q3h.val[0] = vandq_u8(mh, vshlq_n_u8(htmp, 2));
        q3h.val[1] = vandq_u8(mh, htmp);
        q3h.val[2] = vandq_u8(mh, vshrq_n_u8(htmp, 2));
        q3h.val[3] = vandq_u8(mh, vshrq_n_u8(htmp, 4));

        # 根据 q3bits 和 htmp 计算 q3bytes 的 4 个元素值
        q3bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q3bits, m3b),                q3h.val[0]));
        q3bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 2), m3b), q3h.val[1]));
        q3bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 4), m3b), q3h.val[2]));
        q3bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q3bits, 6),                q3h.val[3]));

        # 根据 q3bytes、q8bytes 和 scales 计算 isum 的值
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[0], q8bytes.val[0])) * scales[0];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[1], q8bytes.val[1])) * scales[2];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[2], q8bytes.val[2])) * scales[1];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[3], q8bytes.val[3])) * scales[3];

        # 计算 sum 的值
        sum += d * isum;

    }

    # 将 sum 的值赋给指针变量 s 所指向的地址
    *s = sum;
#elif defined __AVX2__
    # 如果定义了 AVX2，则创建一个包含 8 个 3 的向量
    const __m256i m3 = _mm256_set1_epi8(3);
    # 如果定义了 AVX2，则创建一个包含 8 个 1 的向量
    const __m256i m1 = _mm256_set1_epi8(1);
    # 如果定义了 AVX2，则创建一个全零的 8 个单精度浮点数的向量
    __m256 acc = _mm256_setzero_ps();
    # 定义一个 64 位无符号整数
    uint64_t aux64;
    # 定义一个包含 2 个 16 位整数的数组
    uint16_t aux16[2];
    # 将 aux16 强制转换为 int8_t 类型的指针
    const int8_t * aux8 = (const int8_t *)aux16;
    }
    # 调用 hsum_float_8 函数，将结果赋值给 s
    *s = hsum_float_8(acc);
#elif defined __AVX__
    # 如果定义了 AVX，则创建一个包含 16 个 3 的向量
    const __m128i m3 = _mm_set1_epi8(3);
    # 如果定义了 AVX，则创建一个包含 16 个 1 的向量
    const __m128i m1 = _mm_set1_epi8(1);
    # 如果定义了 AVX，则创建一个全零的 8 个单精度浮点数的向量
    __m256 acc = _mm256_setzero_ps();
    # 定义一个 64 位无符号整数
    uint64_t aux64;
    # 定义一个包含 2 个 16 位整数的数组
    uint16_t aux16[2];
    # 将 aux16 强制转换为 int8_t 类型的指针
    const int8_t * aux8 = (const int8_t *)aux16;
    }
    # 调用 hsum_float_8 函数，将结果赋值给 s
    *s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    # 定义一个包含 2 个 16 位整数的数组
    uint16_t aux16[2];
    # 将 aux16 强制转换为 int8_t 类型的指针
    int8_t * scales = (int8_t *)aux16;
    # 定义一个单精度浮点数 sumf，并初始化为 0
    float sumf = 0;
    }
    # 将 sumf 的值赋给 s
    *s = sumf;
#else
    # 如果以上条件都不满足，则执行以下代码
    # 定义一个包含 QK_K 个 8 位整数的数组
    int8_t  aux8[QK_K];
    # 定义一个包含 8 个 16 位整数的数组
    int16_t aux16[8];
    # 定义一个包含 8 个单精度浮点数的数组
    float   sums [8];
    # 定义一个包含 8 个 32 位整数的数组
    int32_t aux32[8];
    # 定义一个包含 4 个 32 位整数的数组
    int32_t scales[4];
    # 将 sums 数组的值全部初始化为 0
    memset(sums, 0, 8*sizeof(float));
    # 定义一个单精度浮点数 sumf，并初始化为 0
    float sumf = 0;
    // 遍历 nb 次，处理 x 和 y 数组中的数据
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 中的 qs 和 hmask，以及 y[i] 中的 qs
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        const  int8_t * restrict q8 = y[i].qs;
        int8_t * restrict a = aux8;
        // 遍历 8 次，处理每个元素
        for (int l = 0; l < 8; ++l) {
            // 计算 a 数组中的值
            a[l+ 0] = (int8_t)((q3[l+0] >> 0) & 3) - (hm[l] & 0x01 ? 0 : 4);
            a[l+ 8] = (int8_t)((q3[l+8] >> 0) & 3) - (hm[l] & 0x02 ? 0 : 4);
            a[l+16] = (int8_t)((q3[l+0] >> 2) & 3) - (hm[l] & 0x04 ? 0 : 4);
            a[l+24] = (int8_t)((q3[l+8] >> 2) & 3) - (hm[l] & 0x08 ? 0 : 4);
            a[l+32] = (int8_t)((q3[l+0] >> 4) & 3) - (hm[l] & 0x10 ? 0 : 4);
            a[l+40] = (int8_t)((q3[l+8] >> 4) & 3) - (hm[l] & 0x20 ? 0 : 4);
            a[l+48] = (int8_t)((q3[l+0] >> 6) & 3) - (hm[l] & 0x40 ? 0 : 4);
            a[l+56] = (int8_t)((q3[l+8] >> 6) & 3) - (hm[l] & 0x80 ? 0 : 4);
        }

        // 计算 scales 数组中的值
        scales[0] = (x[i].scales[0] & 0xF) - 8;
        scales[1] = (x[i].scales[0] >>  4) - 8;
        scales[2] = (x[i].scales[1] & 0xF) - 8;
        scales[3] = (x[i].scales[1] >>  4) - 8;

        // 将 aux32 数组清零
        memset(aux32, 0, 8*sizeof(int32_t));
        // 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 aux16 数组中的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] += q8[l] * a[l];
            q8 += 8; a += 8;
            // 计算 aux32 数组中的值
            for (int l = 0; l < 8; ++l) aux32[l] += scales[j] * aux16[l];
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 计算 sums 数组中的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    // 计算 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 定义函数 ggml_vec_dot_q4_K_q8_K，计算两个向量的点积
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算块的数量
    const int nb = n / QK_K;

    // 定义静态常量掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    // 如果定义了 __ARM_NEON
#ifdef __ARM_NEON
    // 定义 NEON 寄存器和变量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const int32x4_t mzero = vdupq_n_s32(0);

    ggml_int8x16x2_t q4bytes;
    ggml_int8x16x2_t q8bytes;

    float sumf = 0;

    // 计算点积

    *s = sumf;

// 如果定义了 __AVX2__
#elif defined __AVX2__
    // 定义 AVX2 寄存器和变量
    const __m256i m4 = _mm256_set1_epi8(0xF);

    __m256 acc = _mm256_setzero_ps();
    // 计算点积

    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

// 如果定义了 __AVX__
#elif defined __AVX__
    // 定义 AVX 寄存器和变量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(0x2);

    __m256 acc = _mm256_setzero_ps();
    // 计算点积

    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

// 如果定义了 __riscv_v_intrinsic
#elif defined __riscv_v_intrinsic
    // 定义 RISC-V 寄存器和变量
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    float sumf = 0;

    // 计算点积

    *s = sumf;

// 如果以上条件都不满足
#else
    // 定义变量和数组
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    memset(sums, 0, 8*sizeof(float));

    float sumf = 0;
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
            // a 指针后移 32 位
            a += 32;
            // 将 q4 数组中的前 32 个元素右移 4 位后的值赋值给 a 数组
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l]  >> 4);
            // a 指针后移 32 位，q4 指针后移 32 位
            a += 32; q4 += 32;
        }
        // 将 x[i] 的 scales 数组中的前 12 个元素拷贝到 utmp 数组
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
        // 重置 a 指针为 aux8 数组的指针，初始化 is 为 0
        a = aux8;
        int is = 0;
        // 遍历 QK_K/32 次，j 从 0 到 QK_K/32-1
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取 scales 数组中的值，计算并更新 aux32 数组的值
            int32_t scale = scales[is++];
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            // 重复上述操作
            // ...
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 更新 sums 数组的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        // 计算 dmin 的值，更新 sumf 的值
        const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;
        sumf -= dmin * sumi;
    }
    // 遍历结束后，将 sums 数组的值累加到 sumf
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#endif
}
#else
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 nb，即 n / QK_K
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 定义常量 m4b 和 mzero
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const int32x4_t mzero = vdupq_n_s32(0);

    // 初始化 sumf 和 sum_mins
    float sumf = 0;
    float sum_mins = 0.f;

    // 定义辅助变量和指针 scales
    uint16_t aux16[2];
    const uint8_t * restrict scales = (const uint8_t *)aux16;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 获取 q4 和 q8 的指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 获取 x[i].scales 的值，并将其存储到 scales 数组中
        const uint16_t * restrict a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算 sum_mins
        const int32_t summi = scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]);
        sum_mins += y[i].d * (float)x[i].d[1] * summi;

        // 计算 d
        const float d = y[i].d * (float)x[i].d[0];

        // 加载 q4 和 q8 的值到寄存器
        const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4);
        ggml_int8x16x4_t q8bytes;
        q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 q4 和 q8 进行位运算
        ggml_int8x16x2_t q4bytes;
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

        // 计算 p1 和 sumi1
        const int32x4_t p1 = ggml_vdotq_s32(ggml_vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
        const int32_t sumi1 = vaddvq_s32(p1) * scales[0];

        // 对 q4 进行位运算
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

        // 计算 p2 和 sumi2
        const int32x4_t p2 = ggml_vdotq_s32(ggml_vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[2]), q4bytes.val[1], q8bytes.val[3]);
        const int32_t sumi2 = vaddvq_s32(p2) * scales[1];

        // 计算 sumf
        sumf += d * (sumi1 + sumi2);
    }

    // 将 sumf - sum_mins 的值赋给 s
    *s = sumf - sum_mins;

#elif defined __AVX2__
    // 创建一个包含8个相同字节的256位整数
    const __m256i m4 = _mm256_set1_epi8(0xF);

    // 创建一个全零的256位浮点数
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数变量
    float summs = 0;

    // 初始化一个包含2个16位无符号整数的数组
    uint16_t aux16[2];
    // 将aux16数组转换为uint8_t类型的指针
    const uint8_t * scales = (const uint8_t *)aux16;

    // 循环处理nb次
    for (int i = 0; i < nb; ++i) {

        // 计算d和m的值
        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        // 创建一个包含8个相同浮点数的256位浮点数
        const __m256 vd = _mm256_set1_ps(d);

        // 将x[i].scales转换为uint16_t类型的指针，并提取其中的值
        const uint16_t * a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算summs的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 将x[i].qs和y[i].qs转换为相应类型的指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 加载q4的位，并进行位运算
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        const __m256i q4l = _mm256_and_si256(q4bits, m4);
        const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

        // 加载q8的位
        const __m256i q8l = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8h = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算p16l和p16h的值
        const __m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
        const __m256i p16h = _mm256_maddubs_epi16(q4h, q8h);

        // 计算p32l的值，并更新acc
        const __m256i p32l = _mm256_madd_epi16(_mm256_set1_epi16(scales[0]), p16l);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32l), acc);

        // 计算p32h的值，并更新acc
        const __m256i p32h = _mm256_madd_epi16(_mm256_set1_epi16(scales[1]), p16h);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32h), acc);

    }

    // 计算最终结果并赋值给s
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
// 计算两个向量的点积，其中向量长度为 QK_K 的倍数
void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度为 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将输入指针转换为特定类型的指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度为 QK_K 的倍数的块数
    const int nb = n / QK_K;

    // 定义用于位运算的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    uint32_t utmp[4];

#ifdef __ARM_NEON
    // 定义 ARM NEON 指令集下的变量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const uint8x16_t mone = vdupq_n_u8(1);
    const uint8x16_t mtwo = vdupq_n_u8(2);
    const int32x4_t mzero = vdupq_n_s32(0);

    ggml_int8x16x4_t q5bytes;

    float sumf = 0;

    }

    *s = sumf;

#elif defined __AVX2__
    // 定义 AVX2 指令集下的变量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m128i mzero = _mm_setzero_si128();
    const __m256i mone  = _mm256_set1_epi8(1);

    __m256 acc = _mm256_setzero_ps();

    float summs = 0.f;

   for (int i = 0; i < nb; ++i) {

        const uint8_t * restrict q5 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

#if QK_K == 256
        // 计算 d 和 dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 复制数据到 utmp 数组
        memcpy(utmp, x[i].scales, 12);
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;
#else
        // TODO
        const float d = 0, dmin = 0;
    }

    *s = hsum_float_8(acc) + summs;

#elif defined __AVX__
    // 定义 AVX 指令集下的变量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i mzero = _mm_setzero_si128();
    const __m128i mone  = _mm_set1_epi8(1);
    const __m128i m2 = _mm_set1_epi8(2);

    __m256 acc = _mm256_setzero_ps();

    float summs = 0.f;

    }

    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic
    // 将 utmp 数组的地址转换为 uint8_t 类型的指针，并赋值给 scales
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    // 将 utmp 数组的地址偏移2个字节后转换为 uint8_t 类型的指针，并赋值给 mins
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    // 初始化 sumf 和 sums 为 0
    float sumf = 0;
    float sums = 0.0;

    // 声明一个变量 vl，但未初始化

    }

    // 将 sumf 和 sums 的和赋值给指针变量 s 所指向的位置
    *s = sumf+sums;
#else
// 如果不是 ARM 平台，则执行以下代码

    // 从 utmp 数组中获取 scales 和 mins 的地址
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    // 定义长度为 QK_K 的 int8_t 数组和长度为 8 的 int16_t 数组、float 数组、int32_t 数组
    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    // 将 sums 数组的内容初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 定义一个名为 sumf 的 float 变量，并初始化为 0
    float sumf = 0;
    }
    // 遍历 sums 数组，计算其元素的和
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将计算结果赋值给指针 s 所指向的变量
    *s = sumf;
#endif
}

#else
// 如果不是 ARM 平台，则执行以下代码

void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q5_K* 和 block_q8_K* 类型的指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 n 除以 QK_K 的商，赋值给 nb
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 如果是 ARM 平台，则执行以下代码
    // 定义一些 NEON 指令需要用到的常量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const uint8x16_t mh = vdupq_n_u8(16);
    const int32x4_t mzero = vdupq_n_s32(0);

    // 定义一些 NEON 指令需要用到的变量
    ggml_int8x16x4_t q5bytes;
    ggml_uint8x16x4_t q5h;

    // 定义一个名为 sumf 的 float 变量，并初始化为 0
    float sumf = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * (float)x[i].d;
        // 获取 x[i].scales 的值
        const int8_t * sc = x[i].scales;

        // 获取 x[i].qs 和 x[i].qh 的值
        const uint8_t * restrict q5 = x[i].qs;
        const uint8_t * restrict qh = x[i].qh;
        const int8_t  * restrict q8 = y[i].qs;

        // 从 qh 中加载 8 个 uint8_t 类型的值到 qhbits
        const uint8x8_t qhbits = vld1_u8(qh);

        // 从 q5 中加载 16 个 uint8_t 类型的值到 q5bits
        const ggml_uint8x16x2_t q5bits = ggml_vld1q_u8_x2(q5);
        // 从 q8 中加载 16 个 int8_t 类型的值到 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 将 qhbits 的值进行位操作和移位操作，然后存储到 q5h 中
        const uint8x16_t htmp = vcombine_u8(qhbits, vshr_n_u8(qhbits, 1));
        q5h.val[0] = vbicq_u8(mh, vshlq_n_u8(htmp, 4));
        q5h.val[1] = vbicq_u8(mh, vshlq_n_u8(htmp, 2));
        q5h.val[2] = vbicq_u8(mh, htmp);
        q5h.val[3] = vbicq_u8(mh, vshrq_n_u8(htmp, 2));

        // 对 q5bits 和 q5h 进行位操作和减法操作，然后存储到 q5bytes 中
        q5bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[0], m4b)), vreinterpretq_s8_u8(q5h.val[0]));
        q5bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[1], m4b)), vreinterpretq_s8_u8(q5h.val[1]));
        q5bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[0], 4)), vreinterpretq_s8_u8(q5h.val[2]));
        q5bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[1], 4)), vreinterpretq_s8_u8(q5h.val[3]));

        // 计算四个部分的点积，并乘以对应的 sc 值，然后相加
        int32_t sumi1 = sc[0] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]));
        int32_t sumi2 = sc[1] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[1], q8bytes.val[1]));
        int32_t sumi3 = sc[2] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]));
        int32_t sumi4 = sc[3] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[3], q8bytes.val[3]));

        // 将四个部分的结果相加，并乘以 d，然后累加到 sumf 中
        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);
    }

    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
    // 如果支持 AVX2 指令集
    const __m256i m4 = _mm256_set1_epi8(0xF);  // 创建一个包含 32 个 0xF 的 AVX 寄存器
    const __m256i mone  = _mm256_set1_epi8(1);  // 创建一个包含 32 个 1 的 AVX 寄存器

    __m256 acc = _mm256_setzero_ps();  // 创建一个包含 8 个 0.0 的 AVX 寄存器

    for (int i = 0; i < nb; ++i) {  // 循环处理 nb 次

        const uint8_t * restrict q5 = x[i].qs;  // 获取 x[i] 的 qs 指针
        const int8_t  * restrict q8 = y[i].qs;  // 获取 y[i] 的 qs 指针

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);  // 计算 d 值

        const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);  // 从内存中加载 q5 的值到 AVX 寄存器

        const __m256i scale_l = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[1]), _mm_set1_epi16(x[i].scales[0]));  // 创建包含 x[i].scales[1] 和 x[i].scales[0] 的 AVX 寄存器
        const __m256i scale_h = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[3]), _mm_set1_epi16(x[i].scales[2]));  // 创建包含 x[i].scales[3] 和 x[i].scales[2] 的 AVX 寄存器

        int64_t aux64;  // 定义一个 64 位整型变量
        memcpy(&aux64, x[i].qh, 8);  // 从 x[i].qh 复制 8 个字节到 aux64
        const __m128i haux128 = _mm_set_epi64x(aux64 >> 1, aux64);  // 创建一个包含 aux64 值的 AVX 寄存器
        const __m256i haux256 = MM256_SET_M128I(_mm_srli_epi16(haux128, 2), haux128);  // 创建一个包含 haux128 值的 AVX 寄存器

        const __m256i q5h_0 = _mm256_slli_epi16(_mm256_andnot_si256(haux256, mone), 4);  // 创建一个 AVX 寄存器进行位移和逻辑运算
        const __m256i q5h_1 = _mm256_slli_epi16(_mm256_andnot_si256(_mm256_srli_epi16(haux256, 4), mone), 4);  // 创建一个 AVX 寄存器进行位移和逻辑运算

        const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);  // 创建一个 AVX 寄存器进行位与运算
        const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);  // 创建一个 AVX 寄存器进行位移和位与运算

        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));  // 从内存中加载 q8 的值到 AVX 寄存器
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));  // 从内存中加载 q8 的值到 AVX 寄存器

        const __m256i p16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5l_0, q8_0));  // 创建一个 AVX 寄存器进行乘法和加法运算
        const __m256i p16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5l_1, q8_1));  // 创建一个 AVX 寄存器进行乘法和加法运算
        const __m256i s16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5h_0, q8_0));  // 创建一个 AVX 寄存器进行乘法和加法运算
        const __m256i s16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5h_1, q8_1));  // 创建一个 AVX 寄存器进行乘法和加法运算

        const __m256i dot = _mm256_sub_epi32(_mm256_add_epi32(p16_0, p16_1), _mm256_add_epi32(s16_0, s16_1));  // 创建一个 AVX 寄存器进行加法和减法运算

        acc = _mm256_fmadd_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(dot), acc);  // 创建一个 AVX 寄存器进行乘法和加法运算

    }

    *s = hsum_float_8(acc);  // 将 acc 寄存器中的值求和并赋给 *s

#elif defined __AVX__
    # 创建一个包含8个相同值的128位整数向量，每个元素都是0xF
    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个包含8个相同值的128位整数向量，每个元素都是1
    const __m128i mone  = _mm_set1_epi8(1);
    # 创建一个包含8个0的256位单精度浮点数向量
    __m256 acc = _mm256_setzero_ps();
    # 计算256位单精度浮点数向量acc的和，并将结果存储到指针s指向的位置
    *s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    # 如果定义了 __riscv_v_intrinsic，则执行以下代码块

    float sumf = 0;
    # 初始化一个浮点数变量 sumf 为 0

    }

    *s = sumf;
    # 将 sumf 的值赋给指针 s 指向的变量

#else
    # 如果没有定义 __riscv_v_intrinsic，则执行以下代码块

    int8_t aux8[QK_K];
    # 声明一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[16];
    # 声明一个长度为 16 的 int16_t 数组 aux16
    float   sums [8];
    # 声明一个长度为 8 的 float 数组 sums
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组的内容全部初始化为 0

    float sumf = 0;
    # 初始化一个浮点数变量 sumf 为 0
    for (int i = 0; i < nb; ++i) {
        # 循环，i 从 0 到 nb-1
        const uint8_t * restrict q4 = x[i].qs;
        # 声明一个指向 x[i].qs 的 uint8_t 指针 q4
        const uint8_t * restrict hm = x[i].qh;
        # 声明一个指向 x[i].qh 的 uint8_t 指针 hm
        const  int8_t * restrict q8 = y[i].qs;
        # 声明一个指向 y[i].qs 的 int8_t 指针 q8
        int8_t * restrict a = aux8;
        # 声明一个指向 aux8 的 int8_t 指针 a
        for (int l = 0; l < 32; ++l) {
            # 循环，l 从 0 到 31
            a[l+ 0] = q4[l] & 0xF;
            # 将 q4[l] 和 0xF 按位与的结果赋给 a[l+0]
            a[l+32] = q4[l]  >> 4;
            # 将 q4[l] 右移 4 位的结果赋给 a[l+32]
        }
        for (int is = 0; is < 8; ++is) {
            # 循环，is 从 0 到 7
            uint8_t m = 1 << is;
            # 计算 1 左移 is 位的结果赋给 m
            for (int l = 0; l < 8; ++l) a[8*is + l] -= (hm[l] & m ? 0 : 16);
            # 循环，l 从 0 到 7，根据条件减去相应的值
        }

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        # 计算 d 的值
        const int8_t * restrict sc = x[i].scales;
        # 声明一个指向 x[i].scales 的 int8_t 指针 sc

        for (int j = 0; j < QK_K/16; ++j) {
            # 循环，j 从 0 到 QK_K/16-1
            const float dl = d * sc[j];
            # 计算 dl 的值
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            # 循环，l 从 0 到 15，计算 aux16 数组的值
            for (int l = 0; l <  8; ++l) sums[l] += dl * (aux16[l] + aux16[8+l]);
            # 循环，l 从 0 到 7，计算 sums 数组的值
            q8 += 16; a += 16;
            # 指针 q8 和 a 分别向后移动 16 个位置
        }
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 循环，l 从 0 到 7，累加 sums 数组的值到 sumf
    *s = sumf;
    # 将 sumf 的值赋给指针 s 指向的变量
#endif
}
#endif


#if QK_K == 256
# 如果 QK_K 等于 256，则执行以下代码块
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 定义一个函数 ggml_vec_dot_q6_K_q8_K，参数为 n, s, vx, vy

    assert(n % QK_K == 0);
    # 断言 n 能被 QK_K 整除

    const block_q6_K * restrict x = vx;
    # 声明一个指向 vx 的 block_q6_K 指针 x
    const block_q8_K * restrict y = vy;
    # 声明一个指向 vy 的 block_q8_K 指针 y

    const int nb = n / QK_K;
    # 计算 nb 的值为 n 除以 QK_K 的结果

#ifdef __ARM_NEON
    # 如果定义了 __ARM_NEON，则执行以下代码块

    float sum = 0;
    # 初始化一个浮点数变量 sum 为 0

    const uint8x16_t m4b = vdupq_n_u8(0xF);
    # 声明一个 uint8x16_t 类型的变量 m4b，值为 0xF
    const int32x4_t  vzero = vdupq_n_s32(0);
    # 声明一个 int32x4_t 类型的变量 vzero，值为 0
    //const int8x16_t  m32s = vdupq_n_s8(32);
    # 注释掉的代码

    const uint8x16_t mone = vdupq_n_u8(3);
    # 声明一个 uint8x16_t 类型的变量 mone，值为 3

    ggml_int8x16x4_t q6bytes;
    # 声明一个 ggml_int8x16x4_t 类型的变量 q6bytes
    ggml_uint8x16x4_t q6h;
    # 声明一个 ggml_uint8x16x4_t 类型的变量 q6h

    }
    *s = sum;
    # 将 sum 的值赋给指针 s 指向的变量

#elif defined __AVX2__
    # 如果定义了 __AVX2__，则执行以下代码块

    const __m256i m4 = _mm256_set1_epi8(0xF);
    # 声明一个 __m256i 类型的变量 m4，值为 0xF
    const __m256i m2 = _mm256_set1_epi8(3);
    # 声明一个 __m256i 类型的变量 m2，值为 3
    const __m256i m32s = _mm256_set1_epi8(32);
    # 声明一个 __m256i 类型的变量 m32s，值为 32

    __m256 acc = _mm256_setzero_ps();
    # 声明一个 __m256 类型的变量 acc，值为 0

    }
    *s = hsum_float_8(acc);
    # 将 acc 的值赋给指针 s 指向的变量
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
// 计算两个向量的点积，其中 n 为向量长度，s 为结果数组，vx 和 vy 分别为输入向量
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将输入向量转换为 block_q6_K 和 block_q8_K 类型的指针
    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以 QK_K 的商
    const int nb = n / QK_K;

    // 如果支持 ARM NEON 指令集
#ifdef __ARM_NEON
    // 初始化累加和为 0
    float sum = 0;

    // 初始化常量向量
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    const int8x16_t  m32s = vdupq_n_s8(32);
    const int32x4_t  vzero = vdupq_n_s32(0);

    // 初始化常量向量
    const uint8x16_t mone = vdupq_n_u8(3);

    // 定义 ggml_int8x16x4_t 和 ggml_uint8x16x4_t 类型的变量
    ggml_int8x16x4_t q6bytes;
    ggml_uint8x16x4_t q6h;
    // 遍历循环，从 i=0 开始，直到 i<nb 为止
    for (int i = 0; i < nb; ++i) {

        // 将 x[i] 的 d 属性转换为 float 类型
        const float d_all = (float)x[i].d;

        // 限定指针 q6 指向的内存区域，避免指针别名
        const uint8_t * restrict q6 = x[i].ql;
        // 限定指针 qh 指向的内存区域，避免指针别名
        const uint8_t * restrict qh = x[i].qh;
        // 限定指针 q8 指向的内存区域，避免指针别名
        const int8_t  * restrict q8 = y[i].qs;

        // 限定指针 scale 指向的内存区域，避免指针别名
        const int8_t * restrict scale = x[i].scales;

        // 初始化变量 isum 为 0
        int32_t isum = 0;

        // 从内存中加载 qh 指向的 16 个字节到寄存器中
        uint8x16_t qhbits = vld1q_u8(qh);
        // 从内存中加载 q6 指向的 32 个字节到寄存器中
        ggml_uint8x16x2_t q6bits = ggml_vld1q_u8_x2(q6);
        // 从内存中加载 q8 指向的 64 个字节到寄存器中
        ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 qhbits 寄存器中的数据进行位运算和移位操作，得到 q6h.val[0] 到 q6h.val[3]
        // 并存储到 q6h 寄存器中
        q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits), 4);
        uint8x16_t shifted = vshrq_n_u8(qhbits, 2);
        q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 4);
        q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 6);
        q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

        // 对 q6bits 和 q6h 寄存器中的数据进行位运算和移位操作，得到 q6bytes.val[0] 到 q6bytes.val[3]
        // 并存储到 q6bytes 寄存器中
        q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);
        q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);
        q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[2])), m32s);
        q6bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[3])), m32s);

        // 对 q6bytes 和 q8bytes 寄存器中的数据进行点乘操作，并将结果累加到 isum 中
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];

        // 将 isum 乘以 d_all 和 y[i].d，然后累加到 sum 中
        sum += isum * d_all * y[i].d;

    }
    // 将 sum 的值赋给指针 s 指向的内存区域
    *s = sum;
#elif defined __AVX2__
    # 如果定义了 AVX2，则执行以下代码
    const __m256i m4 = _mm256_set1_epi8(0xF);
    # 创建一个包含 32 个 0xF 的 256 位整数向量
    const __m256i m2 = _mm256_set1_epi8(3);
    # 创建一个包含 32 个 3 的 256 位整数向量
    const __m256i m32s = _mm256_set1_epi8(32);
    # 创建一个包含 32 个 32 的 256 位整数向量
    __m256 acc = _mm256_setzero_ps();
    # 创建一个包含 8 个 0 的 256 位单精度浮点数向量
    }
    *s = hsum_float_8(acc);
    # 将 acc 中的 8 个单精度浮点数相加，并将结果存储到 s 中
#elif defined __AVX__
    # 如果定义了 AVX，则执行以下代码
    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个包含 16 个 0xF 的 128 位整数向量
    const __m128i m2 = _mm_set1_epi8(3);
    # 创建一个包含 16 个 3 的 128 位整数向量
    const __m128i m32s = _mm_set1_epi8(32);
    # 创建一个包含 16 个 32 的 128 位整数向量
    __m256 acc = _mm256_setzero_ps();
    # 创建一个包含 8 个 0 的 256 位单精度浮点数向量
    }
    *s = hsum_float_8(acc);
    # 将 acc 中的 8 个单精度浮点数相加，并将结果存储到 s 中
#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则执行以下代码
    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    }
    *s = sumf;
    # 将 sumf 的值存储到 s 中
#else
    # 如果以上条件都不满足，则执行以下代码
    int8_t  aux8[QK_K];
    # 创建一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[8];
    # 创建一个长度为 8 的 int16_t 数组 aux16
    float   sums [8];
    # 创建一个长度为 8 的 float 数组 sums
    int32_t aux32[8];
    # 创建一个长度为 8 的 int32_t 数组 aux32
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组的内容全部初始化为 0
    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    for (int i = 0; i < nb; ++i) {
        # 循环遍历 i 从 0 到 nb-1
        const uint8_t * restrict q4 = x[i].ql;
        # 创建一个指向 x[i].ql 的 uint8_t 指针 q4
        const uint8_t * restrict qh = x[i].qh;
        # 创建一个指向 x[i].qh 的 uint8_t 指针 qh
        const  int8_t * restrict q8 = y[i].qs;
        # 创建一个指向 y[i].qs 的 int8_t 指针 q8
        memset(aux32, 0, 8*sizeof(int32_t));
        # 将 aux32 数组的内容全部初始化为 0
        int8_t * restrict a = aux8;
        # 创建一个指向 aux8 的 int8_t 指针 a
        for (int l = 0; l < 16; ++l) {
            # 循环遍历 l 从 0 到 15
            a[l+ 0] = (int8_t)((q4[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            # 计算并赋值给 a[l+0]
            a[l+16] = (int8_t)((q4[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            # 计算并赋值给 a[l+16]
            a[l+32] = (int8_t)((q4[l+ 0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            # 计算并赋值给 a[l+32]
            a[l+48] = (int8_t)((q4[l+16] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
            # 计算并赋值给 a[l+48]
        }
        # 循环结束
        int is = 0;
        # 初始化一个整数 is 为 0
        for (int j = 0; j < QK_K/16; ++j) {
            # 循环遍历 j 从 0 到 QK_K/16-1
            int scale = x[i].scales[is++];
            # 从 x[i].scales 数组中取出一个值赋给 scale，并 is 自增
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算并赋值给 aux16 数组
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 计算并更新 aux32 数组
            q8 += 8; a += 8;
            # q8 和 a 分别向后移动 8 个位置
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算并赋值给 aux16 数组
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 计算并更新 aux32 数组
            q8 += 8; a += 8;
            # q8 和 a 分别向后移动 8 个位置
        }
        # 循环结束
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        # 计算并赋值给浮点数 d
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        # 计算并更新 sums 数组
    }
    # 循环结束
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 计算并更新 sumf
    *s = sumf;
    # 将 sumf 的值存储到 s 中
#endif
}
#endif
```