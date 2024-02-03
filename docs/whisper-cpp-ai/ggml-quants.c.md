# `whisper.cpp\ggml-quants.c`

```cpp
#include "ggml-quants.h"
#include "ggml-impl.h"

#include <math.h>
#include <string.h>
#include <assert.h>
#include <float.h>
#include <stdlib.h> // for qsort
#include <stdio.h>  // for GGML_ASSERT

#ifdef __ARM_NEON

// 如果 YCM 无法找到 <arm_neon.h>，请创建一个符号链接指向它，例如：
//
//   $ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
//
#include <arm_neon.h>

#else

#ifdef __wasm_simd128__
#include <wasm_simd128.h>
#else
#if defined(__POWER9_VECTOR__) || defined(__powerpc64__)
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

#ifdef __riscv_v_intrinsic
#include <riscv_vector.h>
#endif

#undef MIN
#undef MAX

#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义取最小值的宏
#define MAX(a, b) ((a) > (b) ? (a) : (b))  // 定义取最大值的宏

#define MM256_SET_M128I(a, b) _mm256_insertf128_si256(_mm256_castsi128_si256(b), (a), 1)  // 定义将两个 __m128i 合并成一个 __m256i 的宏

#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)
// 乘法 int8_t，两两相加结果两次
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

#if __AVX__ || __AVX2__ || __AVX512F__
// 水平相加 8 个浮点数
static inline float hsum_float_8(const __m256 x) {
    __m128 res = _mm256_extractf128_ps(x, 1);
    res = _mm_add_ps(res, _mm256_castps256_ps128(x));
    res = _mm_add_ps(res, _mm_movehl_ps(res, res));
    # 使用 SSE 指令对两个单精度浮点数寄存器的数据进行加法运算
    res = _mm_add_ss(res, _mm_movehdup_ps(res));
    # 将结果寄存器中的单精度浮点数转换为标量单精度浮点数并返回
    return _mm_cvtss_f32(res);
// 水平相加 8 个 int32_t
static inline int hsum_i32_8(const __m256i a) {
    // 将输入参数 a 的低 128 位和高 128 位相加得到 sum128
    const __m128i sum128 = _mm_add_epi32(_mm256_castsi256_si128(a), _mm256_extractf128_si256(a, 1));
    // 将 sum128 的高 64 位和低 64 位相加得到 sum64
    const __m128i hi64 = _mm_unpackhi_epi64(sum128, sum128);
    const __m128i sum64 = _mm_add_epi32(hi64, sum128);
    // 将 sum64 的高 32 位和低 32 位相加得到最终结果
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

// 水平相加 4 个 int32_t
static inline int hsum_i32_4(const __m128i a) {
    // 将输入参数 a 的高 64 位和低 64 位相加得到 sum64
    const __m128i hi64 = _mm_unpackhi_epi64(a, a);
    const __m128i sum64 = _mm_add_epi32(hi64, a);
    // 将 sum64 的高 32 位和低 32 位相加得到最终结果
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

#if defined(__AVX2__) || defined(__AVX512F__)
// 将 32 位扩展为 32 字节 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于按位扩展的掩码 shuf_mask
    const __m256i shuf_mask = _mm256_set_epi64x(
            0x0303030303030303, 0x0202020202020202,
            0x0101010101010101, 0x0000000000000000);
    // 使用 shuf_mask 对 x32 进行按位扩展
    __m256i bytes = _mm256_shuffle_epi8(_mm256_set1_epi32(x32), shuf_mask);
    // 创建用于按位或运算的掩码 bit_mask
    const __m256i bit_mask = _mm256_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 将 bytes 和 bit_mask 进行按位或运算
    bytes = _mm256_or_si256(bytes, bit_mask);
    // 返回 bytes 和 -1 进行比较的结果
    return _mm256_cmpeq_epi8(bytes, _mm256_set1_epi64x(-1));
}

// 将 32 个 4 位字段解包为 32 字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 加载 rsi 指向的 16 字节数据到 tmp
    const __m128i tmp = _mm_loadu_si128((const __m128i *)rsi);
    // 将 tmp 的高 64 位右移 4 位和低 64 位合并为 bytes
    const __m256i bytes = MM256_SET_M128I(_mm_srli_epi16(tmp, 4), tmp);
    // 创建用于按位与运算的掩码 lowMask
    const __m256i lowMask = _mm256_set1_epi8( 0xF );
    // 对 bytes 和 lowMask 进行按位与运算
    return _mm256_and_si256(lowMask, bytes);
}

// 将 int16_t 两两相加并作为浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m256i x) {
    // 创建全为 1 的向量 ones
    const __m256i ones = _mm256_set1_epi16(1);
    # 使用 AVX2 指令集中的 _mm256_madd_epi16 函数对两个 __m256i 类型的参数进行乘法和加法操作，得到结果
    const __m256i summed_pairs = _mm256_madd_epi16(ones, x);
    # 将 __m256i 类型的结果转换为 __m256 类型的浮点数向量
    return _mm256_cvtepi32_ps(summed_pairs);
// 返回两个__m256i类型向量的乘积和，并将结果转换为float向量
static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
#if __AVXVNNI__
    // 创建一个全零的__m256i向量
    const __m256i zero = _mm256_setzero_si256();
    // 使用_dpbusd_epi32函数计算两个向量的乘积和
    const __m256i summed_pairs = _mm256_dpbusd_epi32(zero, ax, sy);
    // 将结果转换为float向量
    return _mm256_cvtepi32_ps(summed_pairs);
#else
    // 执行乘法并创建16位值
    const __m256i dot = _mm256_maddubs_epi16(ax, sy);
    return sum_i16_pairs_float(dot);
#endif
}

// 将int8_t相乘，将结果成对相加两次，并作为float向量返回
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
#if __AVXVNNIINT8__
    // 创建一个全零的__m256i向量
    const __m256i zero = _mm256_setzero_si256();
    // 使用_dpbssd_epi32函数计算两个向量的乘积和
    const __m256i summed_pairs = _mm256_dpbssd_epi32(zero, x, y);
    // 将结果转换为float向量
    return _mm256_cvtepi32_ps(summed_pairs);
#else
    // 获取x向量的绝对值
    const __m256i ax = _mm256_sign_epi8(x, x);
    // 对y向量的值进行符号化
    const __m256i sy = _mm256_sign_epi8(y, x);
    return mul_sum_us8_pairs_float(ax, sy);
#endif
}

static inline __m128i packNibbles( __m256i bytes )
{
    // 将16位通道内的位移动，从0000_abcd_0000_efgh到0000_0000_abcd_efgh
#if __AVX512F__
    // 将向量中的每个16位元素右移4位
    const __m256i bytes_srli_4 = _mm256_srli_epi16(bytes, 4);   // 0000_0000_abcd_0000
    // 将原始向量和右移后的向量进行按位或操作
    bytes = _mm256_or_si256(bytes, bytes_srli_4);               // 0000_abcd_abcd_efgh
    // 将结果转换为8位整数向量
    return _mm256_cvtepi16_epi8(bytes);                         // abcd_efgh
#else
    // 创建一个全1的16位整数向量
    const __m256i lowByte = _mm256_set1_epi16( 0xFF );
    // 将bytes向量中的高位和低位分开
    __m256i high = _mm256_andnot_si256( lowByte, bytes );
    __m256i low = _mm256_and_si256( lowByte, bytes );
    // 将高位向右移4位
    high = _mm256_srli_epi16( high, 4 );
    // 将高位和低位重新组合
    bytes = _mm256_or_si256( low, high );

    // 将uint16_t通道压缩为字节
    __m128i r0 = _mm256_castsi256_si128( bytes );
    __m128i r1 = _mm256_extracti128_si256( bytes, 1 );
    return _mm_packus_epi16( r0, r1 );
#endif
}
#elif defined(__AVX__)
// 将32位扩展为32字节{ 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    # 定义一个32位整数变量x32，将x的内容拷贝到x32中
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    # 定义两个128位整数常量shuf_maskl和shuf_maskh，用于后续的数据处理
    const __m128i shuf_maskl = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
    const __m128i shuf_maskh = _mm_set_epi64x(0x0303030303030303, 0x0202020202020202);
    # 使用shuf_maskl和shuf_maskh对x32进行按位重排，得到bytesl和bytesh
    __m128i bytesl = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskl);
    __m128i bytesh = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskh);
    # 定义一个128位整数常量bit_mask，用于后续的位运算
    const __m128i bit_mask = _mm_set1_epi64x(0x7fbfdfeff7fbfdfe);
    # 对bytesl和bytesh进行位或运算，将bit_mask的值与其进行位或操作
    bytesl = _mm_or_si128(bytesl, bit_mask);
    bytesh = _mm_or_si128(bytesh, bit_mask);
    # 对bytesl和bytesh进行比较操作，判断是否等于-1
    bytesl = _mm_cmpeq_epi8(bytesl, _mm_set1_epi64x(-1));
    bytesh = _mm_cmpeq_epi8(bytesh, _mm_set1_epi64x(-1));
    # 返回一个256位整数，将bytesh和bytesl合并为一个256位整数
    return MM256_SET_M128I(bytesh, bytesl);
// Unpack 32 4-bit fields into 32 bytes
// The output vector contains 32 bytes, each one in [ 0 .. 15 ] interval
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // Load 16 bytes from memory
    __m128i tmpl = _mm_loadu_si128((const __m128i *)rsi);
    // Shift right 4 bits to get the high nibbles
    __m128i tmph = _mm_srli_epi16(tmpl, 4);
    // Create a mask to extract the low nibbles
    const __m128i lowMask = _mm_set1_epi8(0xF);
    // Extract the low nibbles
    tmpl = _mm_and_si128(lowMask, tmpl);
    // Extract the high nibbles
    tmph = _mm_and_si128(lowMask, tmph);
    // Combine the high and low nibbles into a 32-byte vector
    return MM256_SET_M128I(tmph, tmpl);
}

// add int16_t pairwise and return as float vector
static inline __m256 sum_i16_pairs_float(const __m128i xh, const __m128i xl) {
    // Create a vector of ones
    const __m128i ones = _mm_set1_epi16(1);
    // Multiply and add pairwise the low nibbles
    const __m128i summed_pairsl = _mm_madd_epi16(ones, xl);
    // Multiply and add pairwise the high nibbles
    const __m128i summed_pairsh = _mm_madd_epi16(ones, xh);
    // Combine the results into a 256-bit vector
    const __m256i summed_pairs = MM256_SET_M128I(summed_pairsh, summed_pairsl);
    // Convert the result to a float vector
    return _mm256_cvtepi32_ps(summed_pairs);
}

static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
    // Extract the low 128 bits of ax
    const __m128i axl = _mm256_castsi256_si128(ax);
    // Extract the high 128 bits of ax
    const __m128i axh = _mm256_extractf128_si256(ax, 1);
    // Extract the low 128 bits of sy
    const __m128i syl = _mm256_castsi256_si128(sy);
    // Extract the high 128 bits of sy
    const __m128i syh = _mm256_extractf128_si256(sy, 1);
    // Perform multiplication and create 16-bit values
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    // Sum the results of the multiplication and return as float vector
    return sum_i16_pairs_float(doth, dotl);
}

// multiply int8_t, add results pairwise twice and return as float vector
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
    // Extract the low 128 bits of x
    const __m128i xl = _mm256_castsi256_si128(x);
    // Extract the high 128 bits of x
    const __m128i xh = _mm256_extractf128_si256(x, 1);
    // Extract the low 128 bits of y
    const __m128i yl = _mm256_castsi256_si128(y);
    // Extract the high 128 bits of y
    const __m128i yh = _mm256_extractf128_si256(y, 1);
    // Get absolute values of x vectors
    const __m128i axl = _mm_sign_epi8(xl, xl);
    const __m128i axh = _mm_sign_epi8(xh, xh);
    // Sign the values of the y vectors
    # 使用 xl 和 yl 的符号位来对 yl 进行符号扩展
    const __m128i syl = _mm_sign_epi8(yl, xl);
    # 使用 xh 和 yh 的符号位来对 yh 进行符号扩展
    const __m128i syh = _mm_sign_epi8(yh, xh);
    # 执行乘法并创建16位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    # 调用函数对16位值进行求和并返回浮点数结果
    return sum_i16_pairs_float(doth, dotl);
// 定义一个静态内联函数，将两个__m128i类型的参数按照nibbles（半字节）打包成一个__m128i类型的结果
static inline __m128i packNibbles( __m128i bytes1, __m128i bytes2 )
{
    // 创建一个低字节掩码，用于提取低字节
    const __m128i lowByte = _mm_set1_epi16( 0xFF );
    // 提取bytes1中的高字节和低字节
    __m128i high = _mm_andnot_si128( lowByte, bytes1 );
    __m128i low = _mm_and_si128( lowByte, bytes1 );
    // 将高字节右移4位，与低字节合并
    high = _mm_srli_epi16( high, 4 );
    bytes1 = _mm_or_si128( low, high );
    // 提取bytes2中的高字节和低字节
    high = _mm_andnot_si128( lowByte, bytes2 );
    low = _mm_and_si128( lowByte, bytes2 );
    // 将高字节右移4位，与低字节合并
    high = _mm_srli_epi16( high, 4 );
    bytes2 = _mm_or_si128( low, high );

    // 将bytes1和bytes2打包成一个__m128i类型的结果
    return _mm_packus_epi16( bytes1, bytes2);
}
#endif
#elif defined(__SSSE3__)
// 水平相加4个4个浮点数
static inline float hsum_float_4x4(const __m128 a, const __m128 b, const __m128 c, const __m128 d) {
    // 分别对a和b、c和d进行水平相加
    __m128 res_0 =_mm_hadd_ps(a, b);
    __m128 res_1 =_mm_hadd_ps(c, d);
    // 继续对res_0和res_1进行水平相加
    __m128 res =_mm_hadd_ps(res_0, res_1);
    res =_mm_hadd_ps(res, res);
    res =_mm_hadd_ps(res, res);

    // 将结果转换为单精度浮点数并返回
    return _mm_cvtss_f32(res);
}
#endif // __AVX__ || __AVX2__ || __AVX512F__
#endif // defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)

#if defined(__ARM_NEON)
#if !defined(__aarch64__)

// 64位兼容性

// vaddvq_s16
// vpaddq_s16
// vpaddq_s32
// vaddvq_s32
// vaddvq_f32
// vmaxvq_f32
// vcvtnq_s32_f32
// vzip1_u8
// vzip2_u8

// 内联函数，对int16x8_t类型的向量进行求和
inline static int32_t vaddvq_s16(int16x8_t v) {
    return
        (int32_t)vgetq_lane_s16(v, 0) + (int32_t)vgetq_lane_s16(v, 1) +
        (int32_t)vgetq_lane_s16(v, 2) + (int32_t)vgetq_lane_s16(v, 3) +
        (int32_t)vgetq_lane_s16(v, 4) + (int32_t)vgetq_lane_s16(v, 5) +
        (int32_t)vgetq_lane_s16(v, 6) + (int32_t)vgetq_lane_s16(v, 7);
}

// 内联函数，对两个int16x8_t类型的向量进行逐元素相加
inline static int16x8_t vpaddq_s16(int16x8_t a, int16x8_t b) {
    int16x4_t a0 = vpadd_s16(vget_low_s16(a), vget_high_s16(a));
    int16x4_t b0 = vpadd_s16(vget_low_s16(b), vget_high_s16(b));
    return vcombine_s16(a0, b0);
}

// 内联函数，对两个int32x4_t类型的向量进行逐元素相加
inline static int32x4_t vpaddq_s32(int32x4_t a, int32x4_t b) {
    # 将输入向量 a 的低位和高位相加，得到一个包含两个 32 位整数的向量
    int32x2_t a0 = vpadd_s32(vget_low_s32(a), vget_high_s32(a));
    # 将输入向量 b 的低位和高位相加，得到一个包含两个 32 位整数的向量
    int32x2_t b0 = vpadd_s32(vget_low_s32(b), vget_high_s32(b));
    # 将两个 32 位整数的向量 a0 和 b0 合并成一个包含四个 32 位整数的向量
    return vcombine_s32(a0, b0);
// 定义一个内联函数，计算 int32x4_t 类型向量的四个元素之和
inline static int32_t vaddvq_s32(int32x4_t v) {
    return vgetq_lane_s32(v, 0) + vgetq_lane_s32(v, 1) + vgetq_lane_s32(v, 2) + vgetq_lane_s32(v, 3);
}

// 定义一个内联函数，计算 float32x4_t 类型向量的四个元素之和
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}

// 定义一个内联函数，计算 float32x4_t 类型向量的四个元素中的最大值
inline static float vmaxvq_f32(float32x4_t v) {
    return
        MAX(MAX(vgetq_lane_f32(v, 0), vgetq_lane_f32(v, 1)),
            MAX(vgetq_lane_f32(v, 2), vgetq_lane_f32(v, 3)));
}

// 定义一个内联函数，将 float32x4_t 类型向量转换为 int32x4_t 类型向量
inline static int32x4_t vcvtnq_s32_f32(float32x4_t v) {
    int32x4_t res;

    res[0] = roundf(vgetq_lane_f32(v, 0));
    res[1] = roundf(vgetq_lane_f32(v, 1));
    res[2] = roundf(vgetq_lane_f32(v, 2));
    res[3] = roundf(vgetq_lane_f32(v, 3));

    return res;
}

// 定义一个内联函数，将两个 uint8x8_t 类型向量按交错顺序合并成一个向量
inline static uint8x8_t vzip1_u8(uint8x8_t a, uint8x8_t b) {
    uint8x8_t res;

    res[0] = a[0]; res[1] = b[0];
    res[2] = a[1]; res[3] = b[1];
    res[4] = a[2]; res[5] = b[2];
    res[6] = a[3]; res[7] = b[3];

    return res;
}

// 定义一个内联函数，将两个 uint8x8_t 类型向量按交错顺序合并成一个向量
inline static uint8x8_t vzip2_u8(uint8x8_t a, uint8x8_t b) {
    uint8x8_t res;

    res[0] = a[4]; res[1] = b[4];
    res[2] = a[5]; res[3] = b[5];
    res[4] = a[6]; res[5] = b[6];
    res[6] = a[7]; res[7] = b[7];

    return res;
}

// 定义一个结构体 ggml_int16x8x2_t，包含两个 int16x8_t 类型的向量
// 用于加载两个 int16_t 类型数组的数据
typedef struct ggml_int16x8x2_t {
    int16x8_t val[2];
} ggml_int16x8x2_t;

// 定义一个内联函数，加载两个 int16_t 类型数组的数据到 ggml_int16x8x2_t 结构体中
inline static ggml_int16x8x2_t ggml_vld1q_s16_x2(const int16_t * ptr) {
    ggml_int16x8x2_t res;

    res.val[0] = vld1q_s16(ptr + 0);
    res.val[1] = vld1q_s16(ptr + 8);

    return res;
}

// 定义一个结构体 ggml_uint8x16x2_t，包含两个 uint8x16_t 类型的向量
// 用于加载两个 uint8_t 类型数组的数据
typedef struct ggml_uint8x16x2_t {
    uint8x16_t val[2];
} ggml_uint8x16x2_t;

// 定义一个内联函数，加载两个 uint8_t 类型数组的数据到 ggml_uint8x16x2_t 结构体中
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
// 定义一个内联函数，用于加载四个 uint8_t 类型的数据到一个 ggml_uint8x16x4_t 结构体中
inline static ggml_uint8x16x4_t ggml_vld1q_u8_x4(const uint8_t * ptr) {
    ggml_uint8x16x4_t res;

    // 加载第一个 uint8x16_t 类型的数据到 res 结构体的第一个元素中
    res.val[0] = vld1q_u8(ptr + 0);
    // 加载第二个 uint8x16_t 类型的数据到 res 结构体的第二个元素中
    res.val[1] = vld1q_u8(ptr + 16);
    // 加载第三个 uint8x16_t 类型的数据到 res 结构体的第三个元素中
    res.val[2] = vld1q_u8(ptr + 32);
    // 加载第四个 uint8x16_t 类型的数据到 res 结构体的第四个元素中
    res.val[3] = vld1q_u8(ptr + 48);

    return res;
}

// 定义一个结构体 ggml_int8x16x2_t，包含两个 int8x16_t 类型的元素
typedef struct ggml_int8x16x2_t {
    int8x16_t val[2];
} ggml_int8x16x2_t;

// 定义一个内联函数，用于加载两个 int8_t 类型的数据到一个 ggml_int8x16x2_t 结构体中
inline static ggml_int8x16x2_t ggml_vld1q_s8_x2(const int8_t * ptr) {
    ggml_int8x16x2_t res;

    // 加载第一个 int8x16_t 类型的数据到 res 结构体的第一个元素中
    res.val[0] = vld1q_s8(ptr + 0);
    // 加载第二个 int8x16_t 类型的数据到 res 结构体的第二个元素中
    res.val[1] = vld1q_s8(ptr + 16);

    return res;
}

// 定义一个结构体 ggml_int8x16x4_t，包含四个 int8x16_t 类型的元素
typedef struct ggml_int8x16x4_t {
    int8x16_t val[4];
} ggml_int8x16x4_t;

// 定义一个内联函数，用于加载四个 int8_t 类型的数据到一个 ggml_int8x16x4_t 结构体中
inline static ggml_int8x16x4_t ggml_vld1q_s8_x4(const int8_t * ptr) {
    ggml_int8x16x4_t res;

    // 加载第一个 int8x16_t 类型的数据到 res 结构体的第一个元素中
    res.val[0] = vld1q_s8(ptr + 0);
    // 加载第二个 int8x16_t 类型的数据到 res 结构体的第二个元素中
    res.val[1] = vld1q_s8(ptr + 16);
    // 加载第三个 int8x16_t 类型的数据到 res 结构体的第三个元素中
    res.val[2] = vld1q_s8(ptr + 32);
    // 加载第四个 int8x16_t 类型的数据到 res 结构体的第四个元素中
    res.val[3] = vld1q_s8(ptr + 48);

    return res;
}

#else

// 定义一系列宏，用于在不支持 ARM_FEATURE_DOTPROD 的情况下进行替换

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

// 定义一个内联函数，用于在不支持 ARM_FEATURE_DOTPROD 的情况下计算 int32x4_t 类型的点积
inline static int32x4_t ggml_vdotq_s32(int32x4_t acc, int8x16_t a, int8x16_t b) {
    // 计算两个 int8x16_t 类型数据的点积
    const int16x8_t p0 = vmull_s8(vget_low_s8 (a), vget_low_s8 (b));
    const int16x8_t p1 = vmull_s8(vget_high_s8(a), vget_high_s8(b));

    // 返回点积结果
    return vaddq_s32(acc, vaddq_s32(vpaddlq_s16(p0), vpaddlq_s16(p1)));
}

#else

// 在支持 ARM_FEATURE_DOTPROD 的情况下，使用 vdotq_s32 函数进行点积计算
#define ggml_vdotq_s32(a, b, c) vdotq_s32(a, b, c)

#endif

#endif

#if defined(__ARM_NEON) || defined(__wasm_simd128__)
// 定义一系列宏，用于在支持 ARM_NEON 或者 wasm_simd128 的情况下进行替换
#define B1(c,s,n)  0x ## n ## c ,  0x ## n ## s
#define B2(c,s,n) B1(c,s,n ## c), B1(c,s,n ## s)
#define B3(c,s,n) B2(c,s,n ## c), B2(c,s,n ## s)
#define B4(c,s,n) B3(c,s,n ## c), B3(c,s,n ## s)
#define B5(c,s,n) B4(c,s,n ## c), B4(c,s,n ## s)
// 定义宏，用于生成表格数据
#define B6(c,s,n) B5(c,s,n ## c), B5(c,s,n ## s)
#define B7(c,s,n) B6(c,s,n ## c), B6(c,s,n ## s)
#define B8(c,s  ) B7(c,s,     c), B7(c,s,     s)

// 预先计算的表格，用于将8位扩展为8字节
static const uint64_t table_b2b_0[1 << 8] = { B8(00, 10) }; // ( b) << 4
static const uint64_t table_b2b_1[1 << 8] = { B8(10, 00) }; // (!b) << 4

// 参考实现，用于确定性地创建模型文件
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k) {
    static const int qk = QK4_0;

    assert(k % qk == 0);

    const int nb = k / qk;

    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对最大值
        float max  = 0.0f;

        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        const float d  = max / -8;
        const float id = d ? 1.0f/d : 0.0f;

        y[i].d = GGML_FP32_TO_FP16(d);

        for (int j = 0; j < qk/2; ++j) {
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 8.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 8.5f));

            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
}

// 将浮点数数组转换为压缩数据数组
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k) {
    quantize_row_q4_0_reference(x, y, k);
}

// 参考实现，用于确定性地创建模型文件
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k) {
    const int qk = QK4_1;

    assert(k % qk == 0);

    const int nb = k / qk;
    // 遍历 nb 次，处理每个输入向量
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历 qk 次，处理每个输入向量的元素
        for (int j = 0; j < qk; j++) {
            // 获取当前元素的值
            const float v = x[i*qk + j];

            // 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算量化因子
        const float d  = (max - min) / ((1 << 4) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将浮点数转换为半精度浮点数，并存储到输出向量中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 遍历 qk/2 次，处理每个输入向量的一半元素
        for (int j = 0; j < qk/2; ++j) {
            // 计算归一化后的值，并量化为 8 位整数
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            // 将量化后的值存储到输出向量中
            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 0.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 0.5f));

            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含多个量化后的值
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_1_reference 函数进行量化
    quantize_row_q4_1_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含多个量化后的值
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK5_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算结构体数组 y 的长度
    const int nb = k / qk;

    // 遍历结构体数组 y
    for (int i = 0; i < nb; i++) {
        // 初始化绝对值最大值和最大值
        float amax = 0.0f; // absolute max
        float max  = 0.0f;

        // 遍历每个量化块
        for (int j = 0; j < qk; j++) {
            // 获取当前值
            const float v = x[i*qk + j];
            // 更新绝对值最大值和最大值
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算量化值和倒数
        const float d  = max / -16;
        const float id = d ? 1.0f/d : 0.0f;

        // 将量化值转换为 FP16 格式并存储在结构体数组 y 中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 初始化存储量化结果的变量
        uint32_t qh = 0;

        // 遍历每个量化块的一半
        for (int j = 0; j < qk/2; ++j) {
            // 计算量化值并转换为 uint8_t 类型
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            const uint8_t xi0 = MIN(31, (int8_t)(x0 + 16.5f));
            const uint8_t xi1 = MIN(31, (int8_t)(x1 + 16.5f));

            // 将量化结果存储在结构体数组 y 中
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第五位并存储在 qh 中
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到结构体数组 y 中
        memcpy(&y[i].qh, &qh, sizeof(qh));
    }
}

// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含多个量化后的值
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q5_0_reference 函数进行量化
    quantize_row_q5_0_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含多个量化后的值
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k) {
    // 定义量化因子 qk
    const int qk = QK5_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算结构体数组 y 的长度
    const int nb = k / qk;
    // 遍历 nb 次，处理每个输入向量
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历 qk 次，处理每个输入向量的元素
        for (int j = 0; j < qk; j++) {
            // 获取当前元素的值
            const float v = x[i*qk + j];

            // 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算量化因子
        const float d  = (max - min) / ((1 << 5) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将浮点数转换为半精度浮点数并存储到输出结构体中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 初始化量化结果
        uint32_t qh = 0;

        // 遍历输入向量的一半元素，进行量化处理
        for (int j = 0; j < qk/2; ++j) {
            // 计算归一化后的值并进行四舍五入
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            // 将归一化后的值转换为无符号8位整数
            const uint8_t xi0 = (uint8_t)(x0 + 0.5f);
            const uint8_t xi1 = (uint8_t)(x1 + 0.5f);

            // 将两个8位整数合并为一个4位整数
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第5位并将其存储在 qh 中的正确位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到输出结构体中
        memcpy(&y[i].qh, &qh, sizeof(y[i].qh));
    }
// 量化一行数据到 Q8_0 格式，调用参考实现函数 quantize_row_q5_1_reference
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k) {
    quantize_row_q5_1_reference(x, y, k);
}

// 用于确定性创建模型文件的参考实现
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k) {
    // 断言 k 是 QK8_0 的倍数
    assert(k % QK8_0 == 0);
    const int nb = k / QK8_0;

    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对最大值

        for (int j = 0; j < QK8_0; j++) {
            const float v = x[i*QK8_0 + j];
            amax = MAX(amax, fabsf(v));
        }

        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        y[i].d = GGML_FP32_TO_FP16(d);

        for (int j = 0; j < QK8_0; ++j) {
            const float x0 = x[i*QK8_0 + j]*id;

            y[i].qs[j] = roundf(x0);
        }
    }
}

// 量化一行数据到 Q8_0 格式
void quantize_row_q8_0(const float * restrict x, void * restrict vy, int k) {
    // 断言 QK8_0 等于 32
    assert(QK8_0 == 32);
    // 断言 k 是 QK8_0 的倍数
    assert(k % QK8_0 == 0);
    const int nb = k / QK8_0;

    block_q8_0 * restrict y = vy;

#if defined(__ARM_NEON)
    // 循环处理每个数据块
    for (int i = 0; i < nb; i++) {
        // 定义存储数据的向量数组
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 从输入数组中加载数据到向量数组中
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算向量数组中数据的绝对值
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算每两个数据中的最大值
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 计算每四个数据中的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 计算所有数据中的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算最大值中的最大值
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算缩放因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将缩放因子转换为 FP16 格式并存储到输出数组中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 对每个数据进行缩放和整数转换，并存储到输出数组中
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
    # 如果编译器支持 WebAssembly SIMD 128 位指令集，则执行以下代码块
    for (int i = 0; i < nb; i++) {
        # 对于每个块的循环，创建存储源向量、绝对值向量和最大值向量的数组
        v128_t srcv [8];
        v128_t asrcv[8];
        v128_t amaxv[8];

        # 从内存中加载源向量数据到数组中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        # 计算源向量的绝对值向量
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        # 计算绝对值向量中每两个元素的最大值
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        # 计算每两个最大值的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        # 计算所有最大值的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        # 计算最大值向量中的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        # 计算归一化因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        # 将归一化因子转换为 FP16 格式并存储到输出数组中
        y[i].d = GGML_FP32_TO_FP16(d);

        # 对每个源向量进行归一化处理，并将结果存储到输出数组中
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
    # 如果编译器支持 AVX2 或 AVX 指令集，则执行下一个条件分支
    // 循环遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将元素加载到4个 AVX 向量中
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

        // 提取最大值并转换为标量
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
        i0 = _mm256_packs_epi32( i0, i1 );    // 将 i0 和 i1 中的 int32 数据打包成 int16 数据
        i2 = _mm256_packs_epi32( i2, i3 );    // 将 i2 和 i3 中的 int32 数据打包成 int16 数据

        // 将 int16 转换为 int8
        i0 = _mm256_packs_epi16( i0, i2 );    // 将 i0 和 i2 中的 int16 数据打包成 int8 数据

        // 我们得到了宝贵的有符号字节，但顺序现在是错误的
        // 这些 AVX2 打包指令独立处理 16 字节块
        // 以下指令用于修正顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );

        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 如果没有 AVX2 指令集

        // 由于我们没有 AVX 中一些必要的函数，
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

    // 遍历处理每个元素
    for (int i = 0; i < nb; i++) {
        // 加载元素到向量 v_x
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_0, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 初始化临时变量 tmp
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0f, vl);
        // 计算最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        // 将最大值转换为 float 类型
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        // 计算 d 和 id
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 类型并存储到 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 计算 x0
        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        // 将结果转换为整数
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        // 存储结果
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);
    }
// 如果不是 ARM_NEON 架构，则执行以下代码
#else
    // 使用 GGML_UNUSED 宏来标记 nb 变量未使用
    GGML_UNUSED(nb);
    // 调用 quantize_row_q8_0_reference 函数，对输入数据进行量化处理
    quantize_row_q8_0_reference(x, y, k);
#endif
}

// 用于确定性创建模型文件的参考实现
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k) {
    // 断言 QK8_1 的值为 32
    assert(QK8_1 == 32);
    // 断言 k 可以被 QK8_1 整除
    assert(k % QK8_1 == 0);
    // 计算 nb 的值
    const int nb = k / QK8_1;

    // 遍历每个 block
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值

        // 计算每个 block 中的绝对值最大值
        for (int j = 0; j < QK8_1; j++) {
            const float v = x[i*QK8_1 + j];
            amax = MAX(amax, fabsf(v));
        }

        // 计算缩放因子 d
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 存储到 y[i].d 中
        y[i].d = d;

        int sum = 0;

        // 对每个 block 中的数据进行量化
        for (int j = 0; j < QK8_1/2; ++j) {
            const float v0 = x[i*QK8_1           + j]*id;
            const float v1 = x[i*QK8_1 + QK8_1/2 + j]*id;

            y[i].qs[          j] = roundf(v0);
            y[i].qs[QK8_1/2 + j] = roundf(v1);

            sum += y[i].qs[          j];
            sum += y[i].qs[QK8_1/2 + j];
        }

        // 计算 s 值
        y[i].s = sum*d;
    }
}

// 对输入数据进行量化处理
void quantize_row_q8_1(const float * restrict x, void * restrict vy, int k) {
    // 断言 k 可以被 QK8_1 整除
    assert(k % QK8_1 == 0);
    // 计算 nb 的值
    const int nb = k / QK8_1;

    // 将 vy 强制转换为 block_q8_1 指针
    block_q8_1 * restrict y = vy;

    // 如果是 ARM_NEON 架构
#if defined(__ARM_NEON)
    // 循环处理每个数据块
    for (int i = 0; i < nb; i++) {
        // 定义存储数据的向量数组
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 从输入数组中加载数据到向量数组中
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算向量数组中数据的绝对值
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算每两个相邻元素的最大值
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 计算每四个相邻元素的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 计算所有元素的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算最大值
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算比例因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将比例因子存储到输出结构体中
        y[i].d = d;

        // 初始化累加器向量
        int32x4_t accv = vdupq_n_s32(0);

        // 对每个向量进行处理
        for (int j = 0; j < 8; j++) {
            // 计算乘以比例因子后的向量
            const float32x4_t v  = vmulq_n_f32(srcv[j], id);
            // 将浮点向量转换为整型向量
            const int32x4_t   vi = vcvtnq_s32_f32(v);

            // 将整型向量的每个元素存储到输出结构体中
            y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
            y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
            y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
            y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);

            // 更新累加器向量
            accv = vaddq_s32(accv, vi);
        }

        // 计算输出结构体中的另一个字段
        y[i].s = d * vaddvq_s32(accv);
    }
#elif defined(__wasm_simd128__)
    # 如果编译器支持 WebAssembly SIMD 128 位指令集
    for (int i = 0; i < nb; i++) {
        # 对于每个块的处理
        v128_t srcv [8];
        v128_t asrcv[8];
        v128_t amaxv[8];

        # 加载每个块中的数据到 SIMD 寄存器中
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        # 计算每个块中数据的绝对值
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        # 计算每两个相邻数据的最大值
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        # 计算每四个数据的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        # 计算所有数据的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        # 计算最大值中的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        # 计算比例因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        # 将比例因子存储到输出数组中
        y[i].d = d;

        # 初始化累加器
        v128_t accv = wasm_i32x4_splat(0);

        # 对每个数据进行量化和反量化
        for (int j = 0; j < 8; j++) {
            const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
            const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

            # 存储量化后的数据到输出数组中
            y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
            y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
            y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
            y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);

            # 更新累加器
            accv = wasm_i32x4_add(accv, vi);
        }

        # 计算输出的标量值
        y[i].s = d * (wasm_i32x4_extract_lane(accv, 0) +
                      wasm_i32x4_extract_lane(accv, 1) +
                      wasm_i32x4_extract_lane(accv, 2) +
                      wasm_i32x4_extract_lane(accv, 3));
    }
#elif defined(__AVX2__) || defined(__AVX__)
    # 如果编译器支持 AVX2 或 AVX 指令集
    // 循环处理每个块中的元素
    for (int i = 0; i < nb; i++) {
        // 将元素加载到4个 AVX 向量中
        __m256 v0 = _mm256_loadu_ps( x );
        __m256 v1 = _mm256_loadu_ps( x + 8 );
        __m256 v2 = _mm256_loadu_ps( x + 16 );
        __m256 v3 = _mm256_loadu_ps( x + 24 );
        x += 32;

        // 计算块中的 max(abs(e))
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

        // 计算四个整数向量的和，并将结果存储到 y[i].s 中
        y[i].s = d * hsum_i32_8(_mm256_add_epi32(_mm256_add_epi32(i0, i1), _mm256_add_epi32(i2, i3)));

        // 将 int32 类型转换为 int16 类型
        i0 = _mm256_packs_epi32( i0, i1 );    // 0, 1, 2, 3,  8, 9, 10, 11,  4, 5, 6, 7, 12, 13, 14, 15
        i2 = _mm256_packs_epi32( i2, i3 );    // 16, 17, 18, 19,  24, 25, 26, 27,  20, 21, 22, 23, 28, 29, 30, 31
                                              // 将 int16 类型转换为 int8 类型
        i0 = _mm256_packs_epi16( i0, i2 );    // 0, 1, 2, 3,  8, 9, 10, 11,  16, 17, 18, 19,  24, 25, 26, 27,  4, 5, 6, 7, 12, 13, 14, 15, 20, 21, 22, 23, 28, 29, 30, 31

        // 我们得到了宝贵的有符号字节，但顺序现在是错误的
        // 这些 AVX2 打包指令独立处理 16 字节块
        // 以下指令用于修正顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );

        // 将结果存储到 y[i].qs 中
        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 如果不支持 AVX，将寄存器拆分成两半，并从 SSE 中调用 AVX2 的类似函数
        __m128i ni0 = _mm256_castsi256_si128( i0 );
        __m128i ni1 = _mm256_extractf128_si256( i0, 1);
        __m128i ni2 = _mm256_castsi256_si128( i1 );
        __m128i ni3 = _mm256_extractf128_si256( i1, 1);
        __m128i ni4 = _mm256_castsi256_si128( i2 );
        __m128i ni5 = _mm256_extractf128_si256( i2, 1);
        __m128i ni6 = _mm256_castsi256_si128( i3 );
        __m128i ni7 = _mm256_extractf128_si256( i3, 1);

        // 计算量化值的总和并设置 y[i].s
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

    // 设置 VL 为 QK8_1 对应的值
    size_t vl = __riscv_vsetvl_e32m4(QK8_1);
    // 遍历循环，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 加载元素到向量 v_x
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_1, vl);

        // 计算向量 v_x 中每个元素的绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        // 创建一个临时的单精度浮点数向量，值为 0.0
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0, vl);
        // 在向量 vfabs 中找到最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        // 将最大值转换为单精度浮点数
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        // 计算 d 的值
        const float d  = amax / ((1 << 7) - 1);
        // 计算 id 的值
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 存储到 y[i].d 中
        y[i].d = d;

        // 将向量 v_x 乘以 id，结果存储到 x0 中
        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        // 将 x0 转换为整数
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        // 将结果存储到 y[i].qs 中
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);

        // 计算 y[i].s 的和
        vint16m1_t tmp2 = __riscv_vmv_v_x_i16m1(0, vl);
        vint16m1_t vwrs = __riscv_vwredsum_vs_i8m1_i16m1(vs, tmp2, vl);

        // 将结果存储到 y[i].s 中
        int sum = __riscv_vmv_x_s_i16m1_i16(vwrs);
        y[i].s = sum*d;
    }
// 如果不是特定条件下的情况，执行以下代码
#else
    // 使用 GGML_UNUSED 宏来标记变量 nb 未使用
    GGML_UNUSED(nb);
    // 对行进行量化
    quantize_row_q8_1_reference(x, y, k);
#endif
}

// 对行进行反量化，使用 Q4_0 量化
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k) {
    // 定义 Q4_0 量化的常量 qk
    static const int qk = QK4_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历每个 nb
    for (int i = 0; i < nb; i++) {
        // 将 GGML_FP16_TO_FP32 转换后的值赋给 d
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个 qk/2
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1 的值
            const int x0 = (x[i].qs[j] & 0x0F) - 8;
            const int x1 = (x[i].qs[j] >>   4) - 8;

            // 计算 y 的值
            y[i*qk + j + 0   ] = x0*d;
            y[i*qk + j + qk/2] = x1*d;
        }
    }
}

// 对行进行反量化，使用 Q4_1 量化
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k) {
    // 定义 Q4_1 量化的常量 qk
    static const int qk = QK4_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历每个 nb
    for (int i = 0; i < nb; i++) {
        // 将 GGML_FP16_TO_FP32 转换后的值赋给 d 和 m
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        // 遍历每个 qk/2
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1 的值
            const int x0 = (x[i].qs[j] & 0x0F);
            const int x1 = (x[i].qs[j] >>   4);

            // 计算 y 的值
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
}

// 对行进行反量化，使用 Q5_0 量化
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k) {
    // 定义 Q5_0 量化的常量 qk
    static const int qk = QK5_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;

    // 遍历每个 nb
    for (int i = 0; i < nb; i++) {
        // 将 GGML_FP16_TO_FP32 转换后的值赋给 d
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 将 x[i].qh 的值拷贝到 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 遍历每个 qk/2
        for (int j = 0; j < qk/2; ++j) {
            // 计算 xh_0 和 xh_1 的值
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            // 计算 x0 和 x1 的值
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            // 计算 y 的值
            y[i*qk + j + 0   ] = x0*d;
            y[i*qk + j + qk/2] = x1*d;
        }
    }
}

// 对行进行反量化，使用 Q5_1 量化
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k) {
    // 定义 Q5_1 量化的常量 qk
    static const int qk = QK5_1;
    # 确保 k 能够整除 qk，否则会触发断言错误
    assert(k % qk == 0);

    # 计算分块数量 nb，即 k 除以 qk 的结果
    const int nb = k / qk;

    # 遍历每个分块
    for (int i = 0; i < nb; i++) {
        # 将 x[i] 中的 d 和 m 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        # 从 x[i] 中复制 qh 的值到 qh 变量
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 遍历每个分块的一半 qk/2
        for (int j = 0; j < qk/2; ++j) {
            # 计算 xh_0 和 xh_1 的值
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            # 计算 x0 和 x1 的值
            const int x0 = (x[i].qs[j] & 0x0F) | xh_0;
            const int x1 = (x[i].qs[j] >>   4) | xh_1;

            # 计算 y 中的值，根据公式 y = x*d + m
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
// 函数结束标记
}

// 对输入的量化数据进行反量化，得到浮点数数据
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k) {
    // 定义静态常量 qk 为 QK8_0
    static const int qk = QK8_0;

    // 断言 k 能被 qk 整除
    assert(k % qk == 0);

    // 计算块数 nb
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化数据转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个块中的量化数据，进行反量化
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

// 返回最接近输入浮点数的整数
static inline int nearest_int(float fval) {
    // 断言输入浮点数小于等于 4194303
    assert(fval <= 4194303.f);
    float val = fval + 12582912.f;
    int i; memcpy(&i, &val, sizeof(int));
    return (i & 0x007fffff) - 0x00400000;
}

// 生成量化后的数据
static float make_qx_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, int rmse_type,
        const float * restrict qw) {
    float max = 0;
    float amax = 0;
    // 找到输入数据中的最大值
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果最大值接近零，则所有数据都设为零
    if (amax < 1e-30f) {
        for (int i = 0; i < n; ++i) {
            L[i] = 0;
        }
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    // 根据 rmse_type 进行不同的量化
    if (rmse_type == 0) {
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale * x[i]);
            L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
        }
        return 1/iscale;
    }
    bool return_early = false;
    // 处理 rmse_type 为负数的情况
    if (rmse_type < 0) {
        rmse_type = -rmse_type;
        return_early = true;
    }
    float sumlx = 0;
    float suml2 = 0;
#ifdef HAVE_BUGGY_APPLE_LINKER
    // 在 Apple ld64 1015.7 中使用 'volatile' 避免展开循环，解决 bug
    for (volatile int i = 0; i < n; ++i) {
#else
    for (int i = 0; i < n; ++i) {
#endif
        int l = nearest_int(iscale * x[i]);
        l = MAX(-nmax, MIN(nmax-1, l));
        L[i] = l + nmax;
        float w = qw ? qw[i] : rmse_type == 1 ? x[i] * x[i] : rmse_type == 2 ? 1 : rmse_type == 3 ? fabsf(x[i]) : sqrtf(fabsf(x[i]));
        sumlx += w*x[i]*l;
        suml2 += w*l*l;
    }
    # 计算比例尺
    float scale = sumlx/suml2;
    # 如果需要提前返回，则根据条件返回相应值
    if (return_early) return suml2 > 0 ? 0.5f*(scale + 1/iscale) : 1/iscale;
    # 计算最佳比例尺
    float best = scale * sumlx;
    # 遍历一定范围内的值
    for (int is = -9; is <= 9; ++is) {
        # 如果值为0，则跳过
        if (is == 0) {
            continue;
        }
        # 计算新的比例尺
        iscale = -(nmax + 0.1f*is) / max;
        # 重置 sumlx 和 suml2
        sumlx = suml2 = 0;
        # 遍历数据集
        for (int i = 0; i < n; ++i) {
            # 计算最接近 iscale*x[i] 的整数值
            int l = nearest_int(iscale * x[i]);
            # 将 l 限制在一定范围内
            l = MAX(-nmax, MIN(nmax-1, l));
            # 计算权重
            float w = qw ? qw[i] : rmse_type == 1 ? x[i] * x[i] : rmse_type == 2 ? 1 : rmse_type == 3 ? fabsf(x[i]) : sqrtf(fabsf(x[i]));
            # 更新 sumlx 和 suml2
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        # 如果满足条件，则更新最佳比例尺和相关值
        if (suml2 > 0 && sumlx*sumlx > best*suml2) {
            # 更新数据集中的值
            for (int i = 0; i < n; ++i) {
                int l = nearest_int(iscale * x[i]);
                L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
            }
            # 更新比例尺和最佳比例尺
            scale = sumlx/suml2; best = scale*sumlx;
        }
    }
    # 返回最终的比例尺
    return scale;
// 计算第三个四分位数的分位数，返回 RMSE 值
static float make_q3_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, bool do_rmse) {
    // 初始化最大值和绝对值最大值
    float max = 0;
    float amax = 0;
    // 遍历输入数组 x
    for (int i = 0; i < n; ++i) {
        // 计算绝对值
        float ax = fabsf(x[i]);
        // 更新最大值和绝对值最大值
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果绝对值最大值为 0，即全部为零
    if (!amax) { // all zero
        // 将输出数组 L 全部置为 0
        for (int i = 0; i < n; ++i) { L[i] = 0; }
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    // 如果需要计算 RMSE
    if (do_rmse) {
        // 初始化变量
        float sumlx = 0;
        float suml2 = 0;
        // 遍历输入数组 x
        for (int i = 0; i < n; ++i) {
            // 计算最近整数值
            int l = nearest_int(iscale * x[i]);
            // 限制 l 的范围
            l = MAX(-nmax, MIN(nmax-1, l));
            L[i] = l;
            // 计算中间变量
            float w = x[i]*x[i];
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        // 迭代优化
        for (int itry = 0; itry < 5; ++itry) {
            int n_changed = 0;
            // 遍历输入数组 x
            for (int i = 0; i < n; ++i) {
                float w = x[i]*x[i];
                float slx = sumlx - w*x[i]*L[i];
                // 如果满足条件
                if (slx > 0) {
                    float sl2 = suml2 - w*L[i]*L[i];
                    int new_l = nearest_int(x[i] * sl2 / slx);
                    new_l = MAX(-nmax, MIN(nmax-1, new_l));
                    // 更新 L[i]
                    if (new_l != L[i]) {
                        slx += w*x[i]*new_l;
                        sl2 += w*new_l*new_l;
                        if (sl2 > 0 && slx*slx*suml2 > sumlx*sumlx*sl2) {
                            L[i] = new_l; sumlx = slx; suml2 = sl2;
                            ++n_changed;
                        }
                    }
                }
            }
            // 如果没有变化，结束迭代
            if (!n_changed) {
                break;
            }
        }
        // 将结果映射到 [0, 2*nmax] 区间
        for (int i = 0; i < n; ++i) {
            L[i] += nmax;
        }
        // 返回 RMSE 值
        return sumlx / suml2;
    }
    // 如果不需要计算 RMSE
    for (int i = 0; i < n; ++i) {
        // 计算最近整数值
        int l = nearest_int(iscale * x[i]);
        // 限制 l 的范围
        l = MAX(-nmax, MIN(nmax-1, l));
        L[i] = l + nmax;
    }
    // 返回缩放因子的倒数
    return 1/iscale;
}
// 计算并生成一维量化器的量化值，返回缩放因子
static float make_qkx1_quants(int n, int nmax, const float * restrict x, uint8_t * restrict L, float * restrict the_min,
        int ntry, float alpha) {
    // 初始化最小值和最大值为数组第一个元素
    float min = x[0];
    float max = x[0];
    // 遍历数组，更新最小值和最大值
    for (int i = 1; i < n; ++i) {
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
    }
    // 若最大值等于最小值，则将所有量化值设为0，返回0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = 0;
        return 0.f;
    }
    // 若最小值大于0，则将最小值设为0
    if (min > 0) min = 0;
    // 计算缩放因子
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    // 迭代ntry次
    for (int itry = 0; itry < ntry; ++itry) {
        float sumlx = 0; int suml2 = 0;
        bool did_change = false;
        // 遍历数组，更新量化值
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
        // 计算新的最小值
        for (int i = 0; i < n; ++i) {
            sum += x[i] - scale*L[i];
        }
        min = alpha*min + (1 - alpha)*sum/n;
        if (min > 0) min = 0;
        iscale = 1/scale;
        // 若没有改变则跳出循环
        if (!did_change) break;
    }
    *the_min = -min;
    return scale;
}

// 计算并生成二维量化器的量化值，返回缩放因子
static float make_qkx2_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    // 初始化最小值、最大值、权重总和和加权和
    float min = x[0];
    float max = x[0];
    float sum_w = weights[0];
    float sum_x = sum_w * x[0];
#ifdef HAVE_BUGGY_APPLE_LINKER
    // 使用 'volatile' 防止展开循环，并解决 Apple ld64 1015.7 中的一个 bug
    for (volatile int i = 1; i < n; ++i) {
#else
    for (int i = 1; i < n; ++i) {
#endif
        // 更新最小值、最大值、权重总和和加权和
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
        float w = weights[i];
        sum_w += w;
        sum_x += w * x[i];
    }
    // 若最小值大于0，则将最小值设为0
    if (min > 0) min = 0;
    // 如果最大值等于最小值，则将所有元素置为0，更新the_min为负的最小值，返回0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = -min;
        return 0.f;
    }
    // 计算缩放比例
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    float best_mad = 0;
    // 遍历数组元素
    for (int i = 0; i < n; ++i) {
        // 计算最近整数
        int l = nearest_int(iscale*(x[i] - min));
        L[i] = MAX(0, MIN(nmax, l));
        float diff = scale * L[i] + min - x[i];
        // 根据use_mad标志选择计算绝对值差值还是平方差值
        diff = use_mad ? fabsf(diff) : diff * diff;
        float w = weights[i];
        best_mad += w * diff;
    }
    // 如果nstep小于1，则更新the_min为负的最小值，返回缩放比例
    if (nstep < 1) {
        *the_min = -min;
        return scale;
    }
    // 遍历nstep次
    for (int is = 0; is <= nstep; ++is) {
        iscale = (rmin + rdelta*is + nmax)/(max - min);
        float sum_l = 0, sum_l2 = 0, sum_xl = 0;
        // 遍历数组元素
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale*(x[i] - min));
            l = MAX(0, MIN(nmax, l));
            Laux[i] = l;
            float w = weights[i];
            sum_l += w*l;
            sum_l2 += w*l*l;
            sum_xl += w*l*x[i];
        }
        float D = sum_w * sum_l2 - sum_l * sum_l;
        // 如果D大于0，则计算新的缩放比例和最小值
        if (D > 0) {
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;
            if (this_min > 0) {
                this_min = 0;
                this_scale = sum_xl / sum_l2;
            }
            float mad = 0;
            // 计算新的最小绝对偏差
            for (int i = 0; i < n; ++i) {
                float diff = this_scale * Laux[i] + this_min - x[i];
                diff = use_mad ? fabsf(diff) : diff * diff;
                float w = weights[i];
                mad += w * diff;
            }
            // 如果新的最小绝对偏差小于当前最小绝对偏差，则更新L、best_mad、scale和min
            if (mad < best_mad) {
                for (int i = 0; i < n; ++i) {
                    L[i] = Laux[i];
                }
                best_mad = mad;
                scale = this_scale;
                min = this_min;
            }
        }
    }
    // 更新the_min为负的最小值，返回缩放比例
    *the_min = -min;
    return scale;
// 如果 QK_K 等于 256，则定义一个内联函数，根据索引 j 从输入数组 q 中获取数据，并存储到输出数组 d 和 m 中
static inline void get_scale_min_k4(int j, const uint8_t * restrict q, uint8_t * restrict d, uint8_t * restrict m) {
    // 如果索引 j 小于 4，则从输入数组 q 中获取数据并存储到输出数组 d 和 m 中
    if (j < 4) {
        *d = q[j] & 63; *m = q[j + 4] & 63;
    } else {
        // 否则，根据索引 j 从输入数组 q 中获取数据，并根据特定规则存储到输出数组 d 和 m 中
        *d = (q[j+4] & 0xF) | ((q[j-4] >> 6) << 4);
        *m = (q[j+4] >>  4) | ((q[j-0] >> 6) << 4);
    }
}
#endif

// 2-bit (de)-quantization

// 根据输入数组 x，将其量化为 2-bit 的数据，并存储到输出结构体 y 中，其中 k 为数组长度
void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k) {
    // 断言 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算数组 x 的长度除以 QK_K 的商，得到 nb
    const int nb = k / QK_K;

    // 定义一些数组和变量用于存储中间结果
    uint8_t L[QK_K];
    uint8_t Laux[16];
    float   weights[16];
    float mins[QK_K/16];
    float scales[QK_K/16];

    // 定义一个常量 q4scale 为 15.0
    const float q4scale = 15.f;
}
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 初始化最大比例和最大最小值
        float max_scale = 0; // 由于我们减去了最小值，因此比例始终为正数
        float max_min = 0;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 循环遍历 16 次
            for (int l = 0; l < 16; ++l) weights[l] = fabsf(x[16*j + l]);
            // 计算权重和量化参数
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
            // 更新 y[i].scales 和 y[i].d
            for (int j = 0; j < QK_K/16; ++j) {
                int l = nearest_int(iscale*scales[j]);
                y[i].scales[j] = l;
            }
            y[i].d = GGML_FP32_TO_FP16(max_scale/q4scale);
        } else {
            // 如果最大比例小于等于 0，将 y[i].scales 和 y[i].d 设置为 0
            for (int j = 0; j < QK_K/16; ++j) y[i].scales[j] = 0;
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        
        // 如果最大最小值大于 0
        if (max_min > 0) {
            // 计算缩放比例
            float iscale = q4scale/max_min;
            // 更新 y[i].scales 和 y[i].dmin
            for (int j = 0; j < QK_K/16; ++j) {
                int l = nearest_int(iscale*mins[j]);
                y[i].scales[j] |= (l << 4);
            }
            y[i].dmin = GGML_FP32_TO_FP16(max_min/q4scale);
        } else {
            // 如果最大最小值小于等于 0，将 y[i].dmin 设置为 0
            y[i].dmin = GGML_FP32_TO_FP16(0.f);
        }
        
        // 更新 L 数组
        for (int j = 0; j < QK_K/16; ++j) {
            const float d = GGML_FP16_TO_FP32(y[i].d) * (y[i].scales[j] & 0xF);
            // 如果 d 为 0，则继续下一次循环
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * (y[i].scales[j] >> 4);
            // 循环遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int((x[16*j + ii] + dm)/d);
                // 将 l 限制在 0 到 3 之间
                l = MAX(0, MIN(3, l));
                L[16*j + ii] = l;
            }
        }
    }
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        for (int j = 0; j < QK_K; j += 128) {
            // 遍历 j，每次增加 128
            for (int l = 0; l < 32; ++l) {
                // 遍历 l，每次增加 1
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
                // 将 L 数组中的值按位组合后赋值给 y[i].qs 数组
            }
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 遍历 l，每次增加 1
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
            // 将 L 数组中的值按位组合后赋值给 y[i].qs 数组
        }
#endif

        x += QK_K;
        // x 增加 QK_K 的值

    }
}

void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k) {
    // 断言 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    const int nb = k / QK_K;
    // 计算 nb 的值为 k 除以 QK_K

    for (int i = 0; i < nb; i++) {

        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float min = GGML_FP16_TO_FP32(x[i].dmin);

        const uint8_t * q = x[i].qs;

#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        int is = 0;
        float dl, ml;
        for (int n = 0; n < QK_K; n += 128) {
            // 遍历 n，每次增加 128
            int shift = 0;
            for (int j = 0; j < 4; ++j) {
                // 遍历 j，每次增加 1

                uint8_t sc = x[i].scales[is++];
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 计算 dl 和 ml 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l] >> shift) & 3)) - ml;
                // 计算 y 的值

                sc = x[i].scales[is++];
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 计算 dl 和 ml 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3)) - ml;
                // 计算 y 的值

                shift += 2;
            }
            q += 32;
        }
#else
        // 计算四个不同的缩放系数对应的量化值
        float dl1 = d * (x[i].scales[0] & 0xF), ml1 = min * (x[i].scales[0] >> 4);
        float dl2 = d * (x[i].scales[1] & 0xF), ml2 = min * (x[i].scales[1] >> 4);
        float dl3 = d * (x[i].scales[2] & 0xF), ml3 = min * (x[i].scales[2] >> 4);
        float dl4 = d * (x[i].scales[3] & 0xF), ml4 = min * (x[i].scales[3] >> 4);
        // 对每个量化值进行计算
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
    // 调用参考函数 quantize_row_q2_K_reference 处理量化
    quantize_row_q2_K_reference(x, vy, k);
}

size_t ggml_quantize_q2_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对输入数据进行分块处理并量化
    for (int j = 0; j < n; j += k) {
        block_q2_K * restrict y = (block_q2_K *)dst + j/QK_K;
        quantize_row_q2_K_reference(src + j, y, k);
    }
    return (n/QK_K*sizeof(block_q2_K));
}

static float make_qkx3_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    float min = x[0];
    float max = x[0];
    float sum_w = weights ? weights[0] : x[0]*x[0];
    float sum_x = sum_w * x[0];
#ifdef HAVE_BUGGY_APPLE_LINKER
    // 在苹果 ld64 1015.7 中使用 'volatile' 避免展开循环以解决 bug
    for (volatile int i = 1; i < n; ++i) {
#else
    for (int i = 1; i < n; ++i) {
#endif
        // 更新最小值、最大值、权重和加权和
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
        float w = weights ? weights[i] : x[i]*x[i];
        sum_w += w;
        sum_x += w * x[i];
    }
    // 如果最小值大于0，则将最小值设为0
    if (min > 0) {
        min = 0;
    }
    // 如果最大值小于等于最小值，则将数组 L 全部置为 0，设置 the_min 为 -min，返回 0
    if (max <= min) {
        memset(L, 0, n);
        *the_min = -min;
        return 0.f;
    }
    // 计算缩放比例 iscale
    float iscale = nmax/(max - min);
    // 计算反向缩放比例 scale
    float scale = 1/iscale;
    // 初始化最佳平均绝对偏差 best_mad
    float best_mad = 0;
    // 遍历数组 x
    for (int i = 0; i < n; ++i) {
        // 计算最近整数 l
        int l = nearest_int(iscale*(x[i] - min));
        // 将 l 限制在 [0, nmax] 范围内，并赋值给数组 L
        L[i] = MAX(0, MIN(nmax, l));
        // 计算差值 diff
        float diff = scale * L[i] + min - x[i];
        // 根据 use_mad 判断是否使用绝对值
        diff = use_mad ? fabsf(diff) : diff*diff;
        // 获取权重值 w
        float w = weights ? weights[i] : x[i]*x[i];
        // 更新最佳平均绝对偏差 best_mad
        best_mad += w * diff;
    }
    // 如果 nstep 小于 1，则设置 the_min 为 -min，返回 scale
    if (nstep < 1) {
        *the_min = -min;
        return scale;
    }
    // 遍历 nstep
    for (int is = 0; is <= nstep; ++is) {
        // 计算 iscale
        iscale = (rmin + rdelta*is + nmax)/(max - min);
        // 初始化 sum_l, sum_l2, sum_xl
        float sum_l = 0, sum_l2 = 0, sum_xl = 0;
        // 遍历数组 x
        for (int i = 0; i < n; ++i) {
            // 计算最近整数 l
            int l = nearest_int(iscale*(x[i] - min));
            // 将 l 限制在 [0, nmax] 范围内，并赋值给数组 Laux
            l = MAX(0, MIN(nmax, l));
            Laux[i] = l;
            // 获取权重值 w
            float w = weights ? weights[i] : x[i]*x[i];
            // 更新 sum_l, sum_l2, sum_xl
            sum_l  += w*l;
            sum_l2 += w*l*l;
            sum_xl += w*l*x[i];
        }
        // 计算矩阵 D
        float D = sum_w * sum_l2 - sum_l * sum_l;
        // 如果 D 大于 0
        if (D > 0) {
            // 计算 this_scale 和 this_min
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;
            // 如果 this_min 大于 0，则重新计算 this_min 和 this_scale
            if (this_min > 0) {
                this_min = 0;
                this_scale = sum_xl / sum_l2;
            }
            // 初始化 mad
            float mad = 0;
            // 遍历数组 x
            for (int i = 0; i < n; ++i) {
                // 计算差值 diff
                float diff = this_scale * Laux[i] + this_min - x[i];
                // 根据 use_mad 判断是否使用绝对值
                diff = use_mad ? fabsf(diff) : diff*diff;
                // 获取权重值 w
                float w = weights ? weights[i] : x[i]*x[i];
                // 更新 mad
                mad += w * diff;
            }
            // 如果 mad 小于 best_mad，则更新 L, best_mad, scale, min
            if (mad < best_mad) {
                for (int i = 0; i < n; ++i) {
                    L[i] = Laux[i];
                }
                best_mad = mad;
                scale = this_scale;
                min = this_min;
            }
        }
    }
    // 设置 the_min 为 -min，返回 scale
    *the_min = -min;
    return scale;
// 计算并生成量化参数
static float make_qp_quants(int n, int nmax, const float * restrict x, uint8_t * restrict L, const float * quant_weights) {
    // 初始化最大值为0
    float max = 0;
    // 遍历数组找到最大值
    for (int i = 0; i < n; ++i) {
        max = MAX(max, x[i]);
    }
    // 如果最大值为0，说明全部为0
    if (!max) { // all zero
        // 将输出数组全部置为0
        for (int i = 0; i < n; ++i) { L[i] = 0; }
        return 0.f;
    }
    // 计算缩放因子
    float iscale = nmax / max;
    // 对输入数组进行量化
    for (int i = 0; i < n; ++i) {
        L[i] = nearest_int(iscale * x[i]);
    }
    // 计算反向缩放因子
    float scale = 1/iscale;
    // 初始化最小均方误差
    float best_mse = 0;
    // 计算初始均方误差
    for (int i = 0; i < n; ++i) {
        float diff = x[i] - scale*L[i];
        float w = quant_weights[i];
        best_mse += w*diff*diff;
    }
    // 遍历不同缩放因子
    for (int is = -4; is <= 4; ++is) {
        if (is == 0) continue;
        float iscale_is = (0.1f*is + nmax)/max;
        float scale_is = 1/iscale_is;
        float mse = 0;
        // 计算不同缩放因子下的均方误差
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale_is*x[i]);
            l = MIN(nmax, l);
            float diff = x[i] - scale_is*l;
            float w = quant_weights[i];
            mse += w*diff*diff;
        }
        // 更新最小均方误差和缩放因子
        if (mse < best_mse) {
            best_mse = mse;
            iscale = iscale_is;
        }
    }
    // 初始化变量
    float sumlx = 0;
    float suml2 = 0;
    // 计算最终量化结果
    for (int i = 0; i < n; ++i) {
        int l = nearest_int(iscale * x[i]);
        l = MIN(nmax, l);
        L[i] = l;
        float w = quant_weights[i];
        sumlx += w*x[i]*l;
        suml2 += w*l*l;
    }
    // 循环5次，尝试更新权重
    for (int itry = 0; itry < 5; ++itry) {
        // 记录每次循环中权重更新的次数
        int n_changed = 0;
        // 遍历所有数据点
        for (int i = 0; i < n; ++i) {
            // 获取当前数据点的权重
            float w = quant_weights[i];
            // 计算更新后的 slx 和 sl2
            float slx = sumlx - w*x[i]*L[i];
            float sl2 = suml2 - w*L[i]*L[i];
            // 如果 slx 和 sl2 大于0，则进行更新
            if (slx > 0 && sl2 > 0) {
                // 计算新的 L 值
                int new_l = nearest_int(x[i] * sl2 / slx);
                // 限制新的 L 值不超过 nmax
                new_l = MIN(nmax, new_l);
                // 如果新的 L 值与原来不同，则更新 slx 和 sl2
                if (new_l != L[i]) {
                    slx += w*x[i]*new_l;
                    sl2 += w*new_l*new_l;
                    // 如果更新后的 slx 和 sl2 比之前更好，则更新 L[i]，sumlx 和 suml2，并增加 n_changed
                    if (slx*slx*suml2 > sumlx*sumlx*sl2) {
                        L[i] = new_l; sumlx = slx; suml2 = sl2;
                        ++n_changed;
                    }
                }
            }
        }
        // 如果没有数据点的权重发生改变，则跳出循环
        if (!n_changed) {
            break;
        }
    }
    // 返回最终结果
    return sumlx / suml2;
static void quantize_row_q2_K_impl(const float * restrict x, block_q2_K * restrict y, int k, const float * restrict quant_weights) {
    // 断言量化权重不为空
    GGML_ASSERT(quant_weights);
    // 断言 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块数
    const int nb = k / QK_K;
    // 是否重新量化
    const bool requantize = true;

    // 定义一些变量
    uint8_t L[QK_K];
    uint8_t Laux[16];
    float mins[QK_K/16];
    float scales[QK_K/16];
    float sw[QK_K/16];
    float weight[QK_K/16];
    uint8_t Ls[QK_K/16], Lm[QK_K/16];

    // 循环处理每个块
    for (int i = 0; i < nb; i++) {
        // 初始化 sw 数组
        memset(sw, 0, QK_K/16*sizeof(float));
        float sumx2 = 0;
        // 计算 x 的平方和
        for (int j = 0; j < QK_K; ++j) sumx2 += x[j]*x[j];
        float sigma2 = sumx2/QK_K;
        // 计算每个子块的量化参数
        for (int j = 0; j < QK_K/16; ++j) {
            const float * restrict qw = quant_weights + QK_K * i + 16*j;
            for (int l = 0; l < 16; ++l) weight[l] = qw[l] * sqrtf(sigma2 + x[16*j + l]*x[16*j + l]);
            for (int l = 0; l < 16; ++l) sw[j] += weight[l];
            scales[j] = make_qkx3_quants(16, 3, x + 16*j, weight, L + 16*j, &mins[j], Laux, -0.9f, 0.05f, 36, false);
        }

        // 计算块的最大值和最小值
        float dm  = make_qp_quants(QK_K/16, 15, scales, Ls, sw);
        float mm  = make_qp_quants(QK_K/16, 15, mins,   Lm, sw);
        // 将浮点数转换为半精度浮点数
        y[i].d    = GGML_FP32_TO_FP16(dm);
        y[i].dmin = GGML_FP32_TO_FP16(mm);
        dm        = GGML_FP16_TO_FP32(y[i].d);
        mm        = GGML_FP16_TO_FP32(y[i].dmin);

        // 将量化参数存储到块中
        for (int j = 0; j < QK_K/16; ++j) {
            y[i].scales[j] = Ls[j] | (Lm[j] << 4);
        }

        // 如果需要重新量化
        if (requantize) {
            for (int j = 0; j < QK_K/16; ++j) {
                const float d = dm * (y[i].scales[j] & 0xF);
                if (!d) continue;
                const float m = mm * (y[i].scales[j] >> 4);
                for (int ii = 0; ii < 16; ++ii) {
                    int l = nearest_int((x[16*j + ii] + m)/d);
                    l = MAX(0, MIN(3, l));
                    L[16*j + ii] = l;
                }
            }
        }
    }
}
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        for (int j = 0; j < QK_K; j += 128) {
            // 遍历 j 从 0 到 QK_K，每次增加 128
            for (int l = 0; l < 32; ++l) {
                // 遍历 l 从 0 到 32
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
                // 将 L 数组中的元素按位组合后存入 y[i].qs 数组中
            }
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 遍历 l 从 0 到 16
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
            // 将 L 数组中的元素按位组合后存入 y[i].qs 数组中
        }
#endif

        // x 增加 QK_K
        x += QK_K;

    }
}

// 计算并返回量化后的数据大小
size_t quantize_q2_K(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 忽略 hist 参数
    (void)hist;
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q2_K, n_per_row);
    // 如果 quant_weights 为空，则使用参考方法量化行数据
    if (!quant_weights) {
        quantize_row_q2_K_reference(src, dst, nrow*n_per_row);
    }
    else {
        // 否则，使用 quantize_row_q2_K_impl 方法量化行数据
        char * qrow = (char *)dst;
        for (int row = 0; row < nrow; ++row) {
            quantize_row_q2_K_impl(src, (block_q2_K*)qrow, n_per_row, quant_weights);
            src += n_per_row;
            qrow += row_size;
        }
    }
    // 返回总数据大小
    return nrow * row_size;
}

// 3 位（解）量化
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k) {
    // 断言 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb
    const int nb = k / QK_K;

    int8_t L[QK_K];
    float scales[QK_K / 16];

    for (int i = 0; i < nb; i++) {

        float max_scale = 0;
        float amax = 0;
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
        // 将 y[i].scales 数组的前 12 个元素全部设置为 0
        if (max_scale) {
            // 如果 max_scale 存在
            float iscale = -32.f/max_scale;
            // 计算 iscale 的值
            for (int j = 0; j < QK_K/16; ++j) {
                // 遍历循环，每次处理 16 个元素
                int8_t l = nearest_int(iscale*scales[j]);
                // 计算 l 的值
                l = MAX(-32, MIN(31, l)) + 32;
                // 将 l 的值限制在 -32 到 31 之间，并加上 32
                if (j < 8) {
                    y[i].scales[j] = l & 0xF;
                } else {
                    y[i].scales[j-8] |= ((l & 0xF) << 4);
                }
                // 根据 j 的值将 l 的低 4 位或高 4 位存入 y[i].scales 数组
                l >>= 4;
                y[i].scales[j%4 + 8] |= (l << (2*(j/4)));
                // 将 l 的高 4 位存入 y[i].scales 数组
            }
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
            // 计算 y[i].d 的值
        } else {
            y[i].d = GGML_FP32_TO_FP16(0.f);
            // 如果 max_scale 不存在，则将 y[i].d 设置为 0
        }

        int8_t sc;
        // 定义变量 sc
        for (int j = 0; j < QK_K/16; ++j) {
            // 遍历循环，每次处理 16 个元素
            sc = j < 8 ? y[i].scales[j] & 0xF : y[i].scales[j-8] >> 4;
            // 根据 j 的值从 y[i].scales 数组中取出 sc 的值
            sc = (sc | (((y[i].scales[8 + j%4] >> (2*(j/4))) & 3) << 4) - 32;
            // 根据 j 的值计算 sc 的值
            float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            // 计算 d 的值
            if (!d) {
                // 如果 d 为 0，则跳过当前循环
                continue;
            }
            for (int ii = 0; ii < 16; ++ii) {
                // 遍历循环，每次处理 16 个元素
                int l = nearest_int(x[16*j + ii]/d);
                // 计算 l 的值
                l = MAX(-4, MIN(3, l));
                // 将 l 的值限制在 -4 到 3 之间
                L[16*j + ii] = l + 4;
                // 将 l 的值加上 4 存入 L 数组
            }
        }
#else
        // 如果存在最大缩放比例
        if (max_scale) {
            // 计算缩放比例
            float iscale = -8.f/max_scale;
            // 遍历 QK_K/16 次，每次处理两个元素
            for (int j = 0; j < QK_K/16; j+=2) {
                // 计算缩放后的值并限制在[-8, 7]范围内
                int l1 = nearest_int(iscale*scales[j]);
                l1 = 8 + MAX(-8, MIN(7, l1));
                int l2 = nearest_int(iscale*scales[j+1]);
                l2 = 8 + MAX(-8, MIN(7, l2));
                // 将缩放后的值存入y[i].scales数组
                y[i].scales[j/2] = l1 | (l2 << 4);
            }
            // 将缩放比例存入y[i].d
            y[i].d = GGML_FP32_TO_FP16(1/iscale);
        } else {
            // 如果不存在最大缩放比例，将y[i].scales数组置零
            for (int j = 0; j < QK_K/16; j+=2) {
                y[i].scales[j/2] = 0;
            }
            // 将y[i].d置零
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        // 遍历QK_K/16次
        for (int j = 0; j < QK_K/16; ++j) {
            // 根据j的奇偶性取出scales数组中的值
            int s = j%2 == 0 ? y[i].scales[j/2] & 0xF : y[i].scales[j/2] >> 4;
            // 计算d值
            float d = GGML_FP16_TO_FP32(y[i].d) * (s - 8);
            // 如果d为0，跳过当前循环
            if (!d) {
                continue;
            }
            // 遍历16次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算L数组中的值
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-4, MIN(3, l));
                // 将计算结果存入L数组
                L[16*j + ii] = l + 4;
            }
        }
#endif

        // 将y[i].hmask数组置零
        memset(y[i].hmask, 0, QK_K/8);
        // 将高位设置为1，每8个quants占用一个bit
        int m = 0;
        uint8_t hm = 1;
        // 遍历QK_K次
        for (int j = 0; j < QK_K; ++j) {
            // 如果L[j]大于3，设置对应的hmask
            if (L[j] > 3) {
                y[i].hmask[m] |= hm;
                L[j] -= 4;
            }
            // 更新m和hm的值
            if (++m == QK_K/8) {
                m = 0; hm <<= 1;
            }
        }
#if QK_K == 256
        // 如果QK_K等于256
        for (int j = 0; j < QK_K; j += 128) {
            // 将L数组中的值存入y[i].qs数组
            for (int l = 0; l < 32; ++l) {
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
            }
        }
#else
        // 如果QK_K不等于256
        for (int l = 0; l < 16; ++l) {
            // 将L数组中的值存入y[i].qs数组
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
        }
#endif

        // 更新x的值
        x += QK_K;
    }
}

#if QK_K == 256
// 对输入的量化数据进行反量化，将结果存储在浮点数数组中
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    // 定义掩码常量
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 辅助数组和量化比例数组
    uint32_t aux[4];
    const int8_t * scales = (const int8_t*)aux;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {

        // 将所有量化数据转换为浮点数
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 获取量化数据和掩码
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        uint8_t m = 1;

        // 复制量化比例到辅助数组，并进行位操作
        memcpy(aux, x[i].scales, 12);
        uint32_t tmp = aux[2];
        aux[2] = ((aux[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        aux[3] = ((aux[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        aux[0] = (aux[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        aux[1] = (aux[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);

        // 初始化变量
        int is = 0;
        float dl;
        // 遍历每个量化块
        for (int n = 0; n < QK_K; n += 128) {
            int shift = 0;
            // 遍历每个子块
            for (int j = 0; j < 4; ++j) {

                // 计算浮点数值
                dl = d_all * (scales[is++] - 32);
                for (int l = 0; l < 16; ++l) {
                    // 计算反量化值并存储到结果数组中
                    *y++ = dl * ((int8_t)((q[l+ 0] >> shift) & 3) - ((hm[l+ 0] & m) ? 0 : 4));
                }

                // 计算浮点数值
                dl = d_all * (scales[is++] - 32);
                for (int l = 0; l < 16; ++l) {
                    // 计算反量化值并存储到结果数组中
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
// 对输入的量化数据进行反量化，将结果存储在浮点数数组中
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 确保 QK_K 的值为 64
    assert(QK_K == 64);
    // 计算块的数量
    const int nb = k / QK_K;
    // 遍历 nb 次，nb 为循环次数
    for (int i = 0; i < nb; i++) {

        // 将 x[i] 的 d 属性从 FP16 转换为 FP32 存储在 d_all 中
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 获取 x[i] 的 qs 和 hmask 属性
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;

        // 根据不同的 scales 值计算 d1、d2、d3、d4
        const float d1 = d_all * ((x[i].scales[0] & 0xF) - 8);
        const float d2 = d_all * ((x[i].scales[0] >>  4) - 8);
        const float d3 = d_all * ((x[i].scales[1] & 0xF) - 8);
        const float d4 = d_all * ((x[i].scales[1] >>  4) - 8);

        // 遍历 8 次，计算 y 的值
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
        // 更新 y 的位置
        y += QK_K;
    }
}
#endif

// 将输入的一行数据进行量化处理，调用参考实现函数
void quantize_row_q3_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q3_K_reference(x, vy, k);
}

// 对输入数据进行 Q3_K 类型的量化处理
size_t ggml_quantize_q3_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对输入数据进行分块处理，每块大小为 k
    for (int j = 0; j < n; j += k) {
        // 将处理后的数据存储到目标内存中
        block_q3_K * restrict y = (block_q3_K *)dst + j/QK_K;
        quantize_row_q3_K_reference(src + j, y, k);
    }
    // 返回处理后的数据大小
    return (n/QK_K*sizeof(block_q3_K));
}

// 对一行数据进行 Q3_K 类型的量化处理的具体实现
static void quantize_row_q3_K_impl(const float * restrict x, block_q3_K * restrict y, int n_per_row, const float * restrict quant_weights) {
#if QK_K != 256
    (void)quant_weights;
    quantize_row_q3_K_reference(x, y, n_per_row);
#else
    assert(n_per_row % QK_K == 0);
    const int nb = n_per_row / QK_K;

    int8_t L[QK_K];
    float scales[QK_K / 16];
    float weight[16];
    float sw[QK_K / 16];
    int8_t Ls[QK_K / 16];

    // 这里是 QK_K 为 256 时的具体量化处理实现
#endif
}

// 对输入数据进行 Q3_K 类型的量化处理
size_t quantize_q3_K(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    (void)hist;
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q3_K, n_per_row);
    if (!quant_weights) {
        // 如果没有量化权重，则直接调用参考实现函数进行量化处理
        quantize_row_q3_K_reference(src, dst, nrow*n_per_row);
    }
    else {
        char * qrow = (char *)dst;
        // 对每行数据进行 Q3_K 类型的量化处理
        for (int row = 0; row < nrow; ++row) {
            quantize_row_q3_K_impl(src, (block_q3_K*)qrow, n_per_row, quant_weights);
            src += n_per_row;
            qrow += row_size;
        }
    }
    // 返回处理后的数据总大小
    return nrow * row_size;
}

// ====================== 4-bit (de)-quantization

// 将输入的一行数据进行 Q4_K 类型的量化处理，调用参考实现函数
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k) {
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    uint8_t L[QK_K];
    uint8_t Laux[32];
    float   weights[32];
    float mins[QK_K/32];
    float scales[QK_K/32];

    // Q4_K 类型的量化处理实现
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 初始化最大缩放比例和最大最小值
        float max_scale = 0; // 由于我们在减去最小值，因此缩放始终为正值
        float max_min = 0;

        // 循环遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算 x 的平方和
            float sum_x2 = 0;
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l];
            // 计算 x 的平均值
            float av_x = sqrtf(sum_x2/32);
            // 计算权重
            for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]);
            // 生成 QKX2 的量化值
            scales[j] = make_qkx2_quants(32, 15, x + 32*j, weights, L + 32*j, &mins[j], Laux, -1.f, 0.1f, 20, false);
            // 获取当前缩放比例
            float scale = scales[j];
            // 更新最大缩放比例
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
    }
#if QK_K == 256
        // 如果 QK_K 等于 256，则计算比例因子和最小值的倒数
        float inv_scale = max_scale > 0 ? 63.f/max_scale : 0.f;
        float inv_min   = max_min   > 0 ? 63.f/max_min   : 0.f;
        // 遍历每个 32 位块
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算最接近的比例因子和最小值的整数值
            uint8_t ls = nearest_int(inv_scale*scales[j]);
            uint8_t lm = nearest_int(inv_min*mins[j]);
            // 将值限制在 0 到 63 之间
            ls = MIN(63, ls);
            lm = MIN(63, lm);
            // 根据索引将比例因子和最小值存储到对应的位置
            if (j < 4) {
                y[i].scales[j] = ls;
                y[i].scales[j+4] = lm;
            } else {
                y[i].scales[j+4] = (ls & 0xF) | ((lm & 0xF) << 4);
                y[i].scales[j-4] |= ((ls >> 4) << 6);
                y[i].scales[j-0] |= ((lm >> 4) << 6);
            }
        }
        // 将最大比例因子和最大最小值转换为 FP16 格式
        y[i].d = GGML_FP32_TO_FP16(max_scale/63.f);
        y[i].dmin = GGML_FP32_TO_FP16(max_min/63.f);

        uint8_t sc, m;
        // 遍历每个 32 位块
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取比例因子和最小值的 4 位值
            get_scale_min_k4(j, y[i].scales, &sc, &m);
            // 计算新的值并更新到 L 数组中
            const float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            if (!d) continue;
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * m;
            for (int ii = 0; ii < 32; ++ii) {
                int l = nearest_int((x[32*j + ii] + dm)/d);
                // 将值限制在 0 到 15 之间
                l = MAX(0, MIN(15, l));
                L[32*j + ii] = l;
            }
        }
#else
        // 定义缩放因子
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
        // 将第一个维度的值和最小值存入数组
        y[i].scales[0] = d1 | (m1 << 4);
        // 将第二个维度的值和最小值存入数组
        y[i].scales[1] = d2 | (m2 << 4);
        // 将最大缩放值转换为 FP16 存入数组
        y[i].d[0] = GGML_FP32_TO_FP16(max_scale/s_factor);
        // 将最大最小值转换为 FP16 存入数组
        y[i].d[1] = GGML_FP32_TO_FP16(max_min/s_factor);

        // 初始化变量
        float sumlx = 0;
        int   suml2 = 0;
        // 遍历每个元素
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取当前元素的缩放因子和最小值
            const uint8_t sd = y[i].scales[j] & 0xF;
            const uint8_t sm = y[i].scales[j] >>  4;
            // 计算 d 值
            const float d = GGML_FP16_TO_FP32(y[i].d[0]) * sd;
            // 如果 d 为 0，则跳过
            if (!d) continue;
            // 计算 m 值
            const float m = GGML_FP16_TO_FP32(y[i].d[1]) * sm;
            // 遍历每个元素
            for (int ii = 0; ii < 32; ++ii) {
                // 计算 l 值
                int l = nearest_int((x[32*j + ii] + m)/d);
                // 将 l 值限制在 0 到 15 之间
                l = MAX(0, MIN(15, l));
                // 将 l 值存入数组
                L[32*j + ii] = l;
                // 更新 sumlx 和 suml2
                sumlx += (x[32*j + ii] + m)*l*sd;
                suml2 += l*l*sd*sd;
            }
        }
        // 如果 suml2 不为 0，则更新 y[i].d[0]
        if (suml2) {
            y[i].d[0] = GGML_FP32_TO_FP16(sumlx/suml2);
        }
#endif
        // 初始化指针
        uint8_t * q = y[i].qs;
        // 遍历每个元素
        for (int j = 0; j < QK_K; j += 64) {
            // 将每个元素的值存入数组
            for (int l = 0; l < 32; ++l) q[l] = L[j + l] | (L[j + l + 32] << 4);
            q += 32;
        }

        // 更新指针
        x += QK_K;

    }
}

// 解量化函数
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k) {
    // 断言 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块数
    const int nb = k / QK_K;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {

        // 初始化指针
        const uint8_t * q = x[i].qs;
#if QK_K == 256
    // 如果 QK_K 等于 256，则执行以下代码块
    const float d   = GGML_FP16_TO_FP32(x[i].d);
    // 将 x[i].d 转换为单精度浮点数
    const float min = GGML_FP16_TO_FP32(x[i].dmin);
    // 将 x[i].dmin 转换为单精度浮点数

    int is = 0;
    uint8_t sc, m;
    // 定义变量 is, sc, m
    for (int j = 0; j < QK_K; j += 64) {
        // 循环遍历，每次增加 64
        get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
        // 调用函数 get_scale_min_k4，获取 scale 和 min
        const float d1 = d * sc; const float m1 = min * m;
        // 计算 d1 和 m1
        get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
        // 调用函数 get_scale_min_k4，获取 scale 和 min
        const float d2 = d * sc; const float m2 = min * m;
        // 计算 d2 和 m2
        for (int l = 0; l < 32; ++l) *y++ = d1 * (q[l] & 0xF) - m1;
        // 循环遍历，计算结果并赋值给 y
        for (int l = 0; l < 32; ++l) *y++ = d2 * (q[l]  >> 4) - m2;
        // 循环遍历，计算结果并赋值给 y
        q += 32; is += 2;
        // q 增加 32，is 增加 2
    }
#else
    // 如果 QK_K 不等于 256，则执行以下代码块
    const float dall = GGML_FP16_TO_FP32(x[i].d[0]);
    // 将 x[i].d[0] 转换为单精度浮点数
    const float mall = GGML_FP16_TO_FP32(x[i].d[1]);
    // 将 x[i].d[1] 转换为单精度浮点数
    const float d1 = dall * (x[i].scales[0] & 0xF), m1 = mall * (x[i].scales[0] >> 4);
    // 计算 d1 和 m1
    const float d2 = dall * (x[i].scales[1] & 0xF), m2 = mall * (x[i].scales[1] >> 4);
    // 计算 d2 和 m2
    for (int l = 0; l < 32; ++l) {
        // 循环遍历
        y[l+ 0] = d1 * (q[l] & 0xF) - m1;
        // 计算结果并赋值给 y
        y[l+32] = d2 * (q[l] >>  4) - m2;
        // 计算结果并赋值给 y
    }
    y += QK_K;
    // y 增加 QK_K
#endif
    // 结束条件编译指令

}

void quantize_row_q4_K(const float * restrict x, void * restrict vy, int k) {
    // 定义函数 quantize_row_q4_K，传入参数 x, vy, k
    assert(k % QK_K == 0);
    // 断言 k 能被 QK_K 整除
    block_q4_K * restrict y = vy;
    // 定义 block_q4_K 类型的指针 y，指向 vy
    quantize_row_q4_K_reference(x, y, k);
    // 调用函数 quantize_row_q4_K_reference，传入参数 x, y, k
}

size_t ggml_quantize_q4_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 定义函数 ggml_quantize_q4_K，传入参数 src, dst, n, k, hist
    assert(k % QK_K == 0);
    // 断言 k 能被 QK_K 整除
    (void)hist; // TODO: collect histograms
    // 忽略 hist 参数，待实现：收集直方图

    for (int j = 0; j < n; j += k) {
        // 循环遍历
        block_q4_K * restrict y = (block_q4_K *)dst + j/QK_K;
        // 定义 block_q4_K 类型的指针 y，指向 dst 的偏移位置
        quantize_row_q4_K_reference(src + j, y, k);
        // 调用函数 quantize_row_q4_K_reference，传入参数 src + j, y, k
    }
    return (n/QK_K*sizeof(block_q4_K));
    // 返回结果
}

static void quantize_row_q4_K_impl(const float * restrict x, block_q4_K * restrict y, int n_per_row, const float * quant_weights) {
    // 定义静态函数 quantize_row_q4_K_impl，传入参数 x, y, n_per_row, quant_weights
#if QK_K != 256
    // 如果 QK_K 不等于 256，则执行以下代码块
    (void)quant_weights;
    // 忽略 quant_weights 参数
    quantize_row_q4_K_reference(x, y, n_per_row);
    // 调用函数 quantize_row_q4_K_reference，传入参数 x, y, n_per_row
#else
    // 如果 QK_K 等于 256，则执行以下代码块
    assert(n_per_row % QK_K == 0);
    // 断言 n_per_row 能被 QK_K 整除
    const int nb = n_per_row / QK_K;
    // 计算 nb
    // 声明一个长度为 QK_K 的 uint8_t 类型数组 L
    uint8_t L[QK_K];
    // 声明一个长度为 32 的 uint8_t 类型数组 Laux
    uint8_t Laux[32];
    // 声明一个长度为 32 的 float 类型数组 weights
    float   weights[32];
    // 声明一个长度为 QK_K/32 的 float 类型数组 mins
    float mins[QK_K/32];
    // 声明一个长度为 QK_K/32 的 float 类型数组 scales
    float scales[QK_K/32];
    
    // 代码块结束
    }
#endif
}

// 对输入数据进行 4 位量化，返回量化后的数据大小
size_t quantize_q4_K(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 忽略未使用的参数 hist
    (void)hist;
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q4_K, n_per_row);
    // 如果没有量化权重，则使用参考方法对数据进行量化
    if (!quant_weights) {
        quantize_row_q4_K_reference(src, dst, nrow*n_per_row);
    }
    else {
        // 将目标数据转换为字符指针
        char * qrow = (char *)dst;
        // 遍历每一行数据
        for (int row = 0; row < nrow; ++row) {
            // 使用实现方法对数据进行 4 位量化
            quantize_row_q4_K_impl(src, (block_q4_K*)qrow, n_per_row, quant_weights);
            // 更新源数据指针和目标数据指针
            src += n_per_row;
            qrow += row_size;
        }
    }
    // 返回总数据大小
    return nrow * row_size;
}

// ====================== 5-bit (de)-quantization

// 使用参考方法对一行数据进行 5 位量化
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块数
    const int nb = k / QK_K;

    // 根据 QK_K 的值选择不同的数据类型
#if QK_K == 256
    uint8_t L[QK_K];
    float mins[QK_K/32];
    float scales[QK_K/32];
    float weights[32];
    uint8_t Laux[32];
#else
    int8_t L[QK_K];
    float scales[QK_K/16];
#endif

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
#else
        // 定义最大缩放比例和绝对值最大缩放比例
        float max_scale = 0, amax = 0;
        // 遍历每个16个元素的块
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算每个块的缩放比例并存储在scales数组中
            scales[j] = make_qx_quants(16, 16, x + 16*j, L + 16*j, 1, NULL);
            // 计算绝对值缩放比例
            float abs_scale = fabsf(scales[j]);
            // 更新最大绝对值缩放比例和对应的缩放比例
            if (abs_scale > amax) {
                amax = abs_scale;
                max_scale = scales[j];
            }
        }

        // 计算缩放比例
        float iscale = -128.f/max_scale;
        // 更新y[i]的缩放比例
        for (int j = 0; j < QK_K/16; ++j) {
            int l = nearest_int(iscale*scales[j]);
            y[i].scales[j] = MAX(-128, MIN(127, l));
        }
        // 更新y[i]的d值
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        // 解量化每个块的数据
        for (int j = 0; j < QK_K/16; ++j) {
            const float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            if (!d) continue;
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-16, MIN(15, l));
                L[16*j + ii] = l + 16;
            }
        }

        // 初始化qh和ql数组
        uint8_t * restrict qh = y[i].qh;
        uint8_t * restrict ql = y[i].qs;
        memset(qh, 0, QK_K/8);

        // 将解量化后的数据存储到qh和ql数组中
        for (int j = 0; j < 32; ++j) {
            int jm = j%8;
            int is = j/8;
            int l1 = L[j];
            if (l1 > 15) {
                l1 -= 16; qh[jm] |= (1 << is);
            }
            int l2 = L[j + 32];
            if (l2 > 15) {
                l2 -= 16; qh[jm] |= (1 << (4 + is));
            }
            ql[j] = l1 | (l2 << 4);
        }
#endif

        // 更新x指针位置
        x += QK_K;

    }
}

// 解量化每行数据
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k) {
    // 确保k是QK_K的整数倍
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    for (int i = 0; i < nb; i++) {

        // 获取ql和qh数组
        const uint8_t * ql = x[i].qs;
        const uint8_t * qh = x[i].qh;
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码块
        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 float 类型
        const float min = GGML_FP16_TO_FP32(x[i].dmin);
        // 将 x[i].dmin 转换为 float 类型

        int is = 0;
        uint8_t sc, m;
        uint8_t u1 = 1, u2 = 2;
        // 初始化变量 is, sc, m, u1, u2
        for (int j = 0; j < QK_K; j += 64) {
            // 循环遍历，每次增加 64
            get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
            // 调用函数 get_scale_min_k4，获取 scale 和 min
            const float d1 = d * sc; const float m1 = min * m;
            // 计算 d1 和 m1
            get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
            // 调用函数 get_scale_min_k4，获取 scale 和 min
            const float d2 = d * sc; const float m2 = min * m;
            // 计算 d2 和 m2
            for (int l = 0; l < 32; ++l) *y++ = d1 * ((ql[l] & 0xF) + (qh[l] & u1 ? 16 : 0)) - m1;
            // 循环遍历，计算结果并存入 y
            for (int l = 0; l < 32; ++l) *y++ = d2 * ((ql[l]  >> 4) + (qh[l] & u2 ? 16 : 0)) - m2;
            // 循环遍历，计算结果并存入 y
            ql += 32; is += 2;
            // 更新 ql 和 is
            u1 <<= 2; u2 <<= 2;
            // 更新 u1 和 u2
        }
#else
// 如果 QK_K 不等于 256，则执行以下代码块
        float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 float 类型
        const int8_t * restrict s = x[i].scales;
        // 获取 x[i].scales，并限制只能在当前作用域内使用
        for (int l = 0; l < 8; ++l) {
            // 循环遍历 8 次
            y[l+ 0] = d * s[0] * ((ql[l+ 0] & 0xF) - (qh[l] & 0x01 ? 0 : 16));
            // 计算结果并存入 y
            y[l+ 8] = d * s[0] * ((ql[l+ 8] & 0xF) - (qh[l] & 0x02 ? 0 : 16));
            // 计算结果并存入 y
            y[l+16] = d * s[1] * ((ql[l+16] & 0xF) - (qh[l] & 0x04 ? 0 : 16));
            // 计算结果并存入 y
            y[l+24] = d * s[1] * ((ql[l+24] & 0xF) - (qh[l] & 0x08 ? 0 : 16));
            // 计算结果并存入 y
            y[l+32] = d * s[2] * ((ql[l+ 0] >>  4) - (qh[l] & 0x10 ? 0 : 16));
            // 计算结果并存入 y
            y[l+40] = d * s[2] * ((ql[l+ 8] >>  4) - (qh[l] & 0x20 ? 0 : 16));
            // 计算结果并存入 y
            y[l+48] = d * s[3] * ((ql[l+16] >>  4) - (qh[l] & 0x40 ? 0 : 16));
            // 计算结果并存入 y
            y[l+56] = d * s[3] * ((ql[l+24] >>  4) - (qh[l] & 0x80 ? 0 : 16));
            // 计算结果并存入 y
        }
        y += QK_K;
        // 更新 y
#endif
    }
}

void quantize_row_q5_K(const float * restrict x, void * restrict vy, int k) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 将 vy 强制转换为 block_q5_K 类型，并限制只能在当前作用域内使用
    block_q5_K * restrict y = vy;
    // 调用 quantize_row_q5_K_reference 函数
    quantize_row_q5_K_reference(x, y, k);
}

size_t ggml_quantize_q5_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 忽略 hist 参数，待以后收集直方图
    # 循环，每次增加 k，直到 j 大于等于 n
    for (int j = 0; j < n; j += k) {
        # 将目标地址转换为 block_q5_K 类型，并偏移 j/QK_K 个单位
        block_q5_K * restrict y = (block_q5_K *)dst + j/QK_K;
        # 调用 quantize_row_q5_K_reference 函数，对源地址为 src + j 的数据进行量化，结果存储在 y 中
        quantize_row_q5_K_reference(src + j, y, k);
    }
    # 返回结果，计算 n/QK_K 乘以 sizeof(block_q5_K) 的值
    return (n/QK_K*sizeof(block_q5_K));
// 实现对一行数据进行 Q5_K 量化的函数，输入为浮点数数组 x，输出为 Q5_K 类型的数据结构 y，每行包含 n_per_row 个元素，quant_weights 为量化权重数组
static void quantize_row_q5_K_impl(const float * restrict x, block_q5_K * restrict y, int n_per_row, const float * quant_weights) {
#if QK_K != 256
    // 如果 QK_K 不等于 256，则忽略 quant_weights 参数，调用 quantize_row_q5_K_reference 函数进行量化
    (void)quant_weights;
    quantize_row_q5_K_reference(x, y, n_per_row);
#else
    // 如果 QK_K 等于 256，则进行以下操作
    assert(n_per_row % QK_K == 0);
    // 计算每行包含的 QK_K 数量
    const int nb = n_per_row / QK_K;

    // 定义一些变量
    uint8_t L[QK_K];
    float mins[QK_K/32];
    float scales[QK_K/32];
    float weights[32];
    uint8_t Laux[32];

    // 结束 if 条件
#endif
}

// 对输入的浮点数数组进行 Q5_K 量化，将结果存储在 dst 中，nrow 表示行数，n_per_row 表示每行元素个数，hist 为直方图，quant_weights 为量化权重数组
size_t quantize_q5_K(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 忽略 hist 参数
    (void)hist;
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q5_K, n_per_row);
    // 如果没有量化权重数组，则调用 quantize_row_q5_K_reference 函数进行量化
    if (!quant_weights) {
        quantize_row_q5_K_reference(src, dst, nrow*n_per_row);
    }
    else {
        // 否则，对每行数据进行 Q5_K 量化
        char * qrow = (char *)dst;
        for (int row = 0; row < nrow; ++row) {
            quantize_row_q5_K_impl(src, (block_q5_K*)qrow, n_per_row, quant_weights);
            src += n_per_row;
            qrow += row_size;
        }
    }
    // 返回总的数据大小
    return nrow * row_size;
}

// 实现对一行数据进行 Q6_K 量化的函数，输入为浮点数数组 x，输出为 Q6_K 类型的数据结构 y，k 表示元素个数
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算每行包含的 QK_K 数量
    const int nb = k / QK_K;

    // 定义一些变量
    int8_t L[QK_K];
    float   scales[QK_K/16];
    // 循环处理每个块
    for (int i = 0; i < nb; i++) {

        // 初始化最大缩放值和最大绝对值缩放值
        float max_scale = 0;
        float max_abs_scale = 0;

        // 遍历每个块中的子块
        for (int ib = 0; ib < QK_K/16; ++ib) {

            // 调用函数生成量化值，并获取缩放值
            const float scale = make_qx_quants(16, 32, x + 16*ib, L + 16*ib, 1, NULL);
            scales[ib] = scale;

            // 计算绝对值缩放值，并更新最大绝对值缩放值和最大缩放值
            const float abs_scale = fabsf(scale);
            if (abs_scale > max_abs_scale) {
                max_abs_scale = abs_scale;
                max_scale = scale;
            }

        }

        // 如果最大绝对值缩放值为0，则将y[i]置零并跳过当前循环
        if (!max_abs_scale) {
            memset(&y[i], 0, sizeof(block_q6_K));
            y[i].d = GGML_FP32_TO_FP16(0.f);
            x += QK_K;
            continue;
        }

        // 计算缩放比例，并更新y[i]的d值
        float iscale = -128.f/max_scale;
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        // 更新y[i]的scales数组
        for (int ib = 0; ib < QK_K/16; ++ib) {
            y[i].scales[ib] = MIN(127, nearest_int(iscale*scales[ib]));
        }

        // 更新L数组中的值
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

        // 获取y[i]的ql和qh指针
        uint8_t * restrict ql = y[i].ql;
        uint8_t * restrict qh = y[i].qh;

    }
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
                // 将 q1 和左移 4 位后的 q3 进行按位或操作，并赋值给 ql[l+0]
                ql[l+32] = q2 | (q4 << 4);
                // 将 q2 和左移 4 位后的 q4 进行按位或操作，并赋值给 ql[l+32]
                qh[l] = (L[j + l] >> 4) | ((L[j + l + 32] >> 4) << 2) | ((L[j + l + 64] >> 4) << 4) | ((L[j + l + 96] >> 4) << 6);
                // 将指定位置的值右移 4 位后的结果进行按位或操作，并赋值给 qh[l]
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
            // 将 q1 和左移 4 位后的 q2 进行按位或操作，并赋值给 ql[l]
        }
        for (int l = 0; l < 16; ++l) {
            // 循环遍历，每次增加 1
            qh[l] = (L[l] >> 4) | ((L[l + 16] >> 4) << 2) | ((L[l + 32] >> 4) << 4) | ((L[l + 48] >> 4) << 6);
            // 将指定位置的值右移 4 位后的结果进行按位或操作，并赋值给 qh[l]
        }
#endif

        x += QK_K;
        // x 指针向后移动 QK_K 位
    }
}

void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k) {
    // 函数声明，将 block_q6_K 结构体指针 x 转换为 float 类型指针 y，k 为整数
    assert(k % QK_K == 0);
    // 断言，确保 k 能被 QK_K 整除
    const int nb = k / QK_K;
    // 计算 nb 的值为 k 除以 QK_K 的结果

    for (int i = 0; i < nb; i++) {
        // 循环遍历 nb 次
        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].d 转换为 float 类型，并赋值给 d

        const uint8_t * restrict ql = x[i].ql;
        // 将 x[i].ql 转换为 uint8_t 类型指针，并赋值给 ql
        const uint8_t * restrict qh = x[i].qh;
        // 将 x[i].qh 转换为 uint8_t 类型指针，并赋值给 qh
        const int8_t  * restrict sc = x[i].scales;
        // 将 x[i].scales 转换为 int8_t 类型指针，并赋值给 sc
#if QK_K == 256
        // 如果 QK_K 等于 256，则执行以下代码块
        for (int n = 0; n < QK_K; n += 128) {
            // 循环处理，每次处理128个元素
            for (int l = 0; l < 32; ++l) {
                // 循环处理，每次处理32个元素
                int is = l/16;
                // 计算索引 is
                const int8_t q1 = (int8_t)((ql[l +  0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
                // 计算 q1
                const int8_t q2 = (int8_t)((ql[l + 32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
                // 计算 q2
                const int8_t q3 = (int8_t)((ql[l +  0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32;
                // 计算 q3
                const int8_t q4 = (int8_t)((ql[l + 32]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32;
                // 计算 q4
                y[l +  0] = d * sc[is + 0] * q1;
                // 计算 y[l + 0]
                y[l + 32] = d * sc[is + 2] * q2;
                // 计算 y[l + 32]
                y[l + 64] = d * sc[is + 4] * q3;
                // 计算 y[l + 64]
                y[l + 96] = d * sc[is + 6] * q4;
                // 计算 y[l + 96]
            }
            y  += 128;
            // 更新 y 指针
            ql += 64;
            // 更新 ql 指针
            qh += 32;
            // 更新 qh 指针
            sc += 8;
            // 更新 sc 指针
        }
#else
        // 如果 QK_K 不等于 256，则执行以下代码块
        for (int l = 0; l < 16; ++l) {
            // 循环处理，每次处理16个元素
            const int8_t q1 = (int8_t)((ql[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            // 计算 q1
            const int8_t q2 = (int8_t)((ql[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            // 计算 q2
            const int8_t q3 = (int8_t)((ql[l+ 0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            // 计算 q3
            const int8_t q4 = (int8_t)((ql[l+16]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32;
            // 计算 q4
            y[l+ 0] = d * sc[0] * q1;
            // 计算 y[l + 0]
            y[l+16] = d * sc[1] * q2;
            // 计算 y[l + 16]
            y[l+32] = d * sc[2] * q3;
            // 计算 y[l + 32]
            y[l+48] = d * sc[3] * q4;
            // 计算 y[l + 48]
        }
        y  += 64;
        // 更新 y 指针
#endif

    }
}

void quantize_row_q6_K(const float * restrict x, void * restrict vy, int k) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 将 vy 强制转换为 block_q6_K 类型，并赋值给 y
    block_q6_K * restrict y = vy;
    // 调用 quantize_row_q6_K_reference 函数处理数据
    quantize_row_q6_K_reference(x, y, k);
}

size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 限制 k 必须是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 忽略 hist 参数，用于收集直方图
    (void)hist; // TODO: collect histograms

    for (int j = 0; j < n; j += k) {
        // 将 dst 强制转换为 block_q6_K 类型，并赋值给 y
        block_q6_K * restrict y = (block_q6_K *)dst + j/QK_K;
        // 调用 quantize_row_q6_K_reference 函数处理数据
        quantize_row_q6_K_reference(src + j, y, k);
    }
    // 返回处理后的数据大小
    return (n/QK_K*sizeof(block_q6_K));
}
static void quantize_row_q6_K_impl(const float * restrict x, block_q6_K * restrict y, int n_per_row, const float * quant_weights) {
    // 如果 QK_K 不等于 256，则调用参考实现函数 quantize_row_q6_K_reference
#if QK_K != 256
    (void)quant_weights;
    quantize_row_q6_K_reference(x, y, n_per_row);
#else
    // 断言 n_per_row 必须是 QK_K 的倍数
    assert(n_per_row % QK_K == 0);
    const int nb = n_per_row / QK_K;

    int8_t L[QK_K];
    float   scales[QK_K/16];
    //float   weights[16];

    }
#endif
}

size_t quantize_q6_K(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 忽略 hist 参数
    (void)hist;
    // 计算每行的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q6_K, n_per_row);
    // 如果 quant_weights 为空，则调用参考实现函数 quantize_row_q6_K_reference
    if (!quant_weights) {
        quantize_row_q6_K_reference(src, dst, nrow*n_per_row);
    }
    else {
        char * qrow = (char *)dst;
        // 遍历每一行
        for (int row = 0; row < nrow; ++row) {
            // 调用 quantize_row_q6_K_impl 函数进行量化
            quantize_row_q6_K_impl(src, (block_q6_K*)qrow, n_per_row, quant_weights);
            src += n_per_row;
            qrow += row_size;
        }
    }
    // 返回总大小
    return nrow * row_size;
}

static void quantize_row_q4_0_impl(const float * restrict x, block_q4_0 * restrict y, int n_per_row, const float * quant_weights) {
    // 静态断言，确保 QK4_0 等于 32
    static_assert(QK4_0 == 32, "QK4_0 must be 32");

    // 如果 quant_weights 为空，则调用参考实现函数 quantize_row_q4_0_reference
    if (!quant_weights) {
        quantize_row_q4_0_reference(x, y, n_per_row);
        return;
    }

    float weight[QK4_0];
    int8_t L[QK4_0];

    float sum_x2 = 0;
    // 计算 x 中每个元素的平方和
    for (int j = 0; j < n_per_row; ++j) sum_x2 += x[j]*x[j];
    float sigma2 = sum_x2/n_per_row;

    const int nb = n_per_row/QK4_0;
    // 对每个子块进行量化
    for (int ib = 0; ib < nb; ++ib) {
        const float * xb = x + QK4_0 * ib;
        const float * qw = quant_weights + QK4_0 * ib;
        // 计算权重
        for (int j = 0; j < QK4_0; ++j) weight[j] = qw[j] * sqrtf(sigma2 + xb[j]*xb[j]);
        // 调用 make_qx_quants 函数进行量化
        float d = make_qx_quants(QK4_0, 8, xb, L, 1, weight);
        // 将浮点数转换为半精度浮点数
        y[ib].d = GGML_FP32_TO_FP16(d);
        // 将量化结果存储到 block_q4_0 结构体中
        for (int j = 0; j < 16; ++j) {
            y[ib].qs[j] = L[j] | (L[j+16] << 4);
        }
    }
}
# 对输入的浮点数数组进行 Q4_0 格式的量化，并将结果存储到目标数组中
size_t quantize_q4_0(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    # 如果没有提供量化权重，则调用 ggml_quantize_q4_0 函数进行量化
    if (!quant_weights) {
        return ggml_quantize_q4_0(src, dst, nrow*n_per_row, n_per_row, hist);
    }
    # 计算每行的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q4_0, n_per_row);
    # 将目标数组转换为字符指针
    char * qrow = (char *)dst;
    # 遍历每一行进行量化
    for (int row = 0; row < nrow; ++row) {
        # 调用 quantize_row_q4_0_impl 函数对当前行进行量化
        quantize_row_q4_0_impl(src, (block_q4_0*)qrow, n_per_row, quant_weights);
        # 更新源数组和目标数组的指针位置
        src += n_per_row;
        qrow += row_size;
    }
    # 返回总共量化的字节数
    return nrow * row_size;
}

# 对输入的浮点数数组进行 Q4_1 格式的量化，并将结果存储到目标数组中
static void quantize_row_q4_1_impl(const float * restrict x, block_q4_1 * restrict y, int n_per_row, const float * quant_weights) {
    # 静态断言，确保 QK4_1 的值为 32
    static_assert(QK4_1 == 32, "QK4_1 must be 32");

    # 如果没有提供量化权重，则调用 quantize_row_q4_1_reference 函数进行量化
    if (!quant_weights) {
        quantize_row_q4_1_reference(x, y, n_per_row);
        return;
    }

    # 初始化权重、量化结果数组等
    float weight[QK4_1];
    uint8_t L[QK4_1], Laux[QK4_1];

    # 计算输入数组平方和
    float sum_x2 = 0;
    for (int j = 0; j < n_per_row; ++j) sum_x2 += x[j]*x[j];
    float sigma2 = sum_x2/n_per_row;

    # 将每个块进行量化
    const int nb = n_per_row/QK4_1;
    for (int ib = 0; ib < nb; ++ib) {
        const float * xb = x + QK4_1 * ib;
        const float * qw = quant_weights + QK4_1 * ib;
        for (int j = 0; j < QK4_1; ++j) weight[j] = qw[j] * sqrtf(sigma2 + xb[j]*xb[j]);
        float min;
        float d = make_qkx3_quants(QK4_1, 15, xb, weight, L, &min, Laux, -0.9f, 0.05f, 36, false);
        y[ib].d = GGML_FP32_TO_FP16(d);
        y[ib].m = GGML_FP32_TO_FP16(-min);
        for (int j = 0; j < 16; ++j) {
            y[ib].qs[j] = L[j] | (L[j+16] << 4);
        }
    }
}

# 对输入的浮点数数组进行 Q4_1 格式的量化，并将结果存储到目标数组中
size_t quantize_q4_1(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    # 如果没有提供量化权重，则调用 ggml_quantize_q4_1 函数进行量化
    if (!quant_weights) {
        return ggml_quantize_q4_1(src, dst, nrow*n_per_row, n_per_row, hist);
    }
    # 计算每行的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q4_1, n_per_row);
    # 将目标数组转换为字符指针
    char * qrow = (char *)dst;
    # 遍历每一行数据
    for (int row = 0; row < nrow; ++row) {
        # 对当前行数据进行量化处理，将结果存储在qrow中
        quantize_row_q4_1_impl(src, (block_q4_1*)qrow, n_per_row, quant_weights);
        # 移动源数据指针到下一行的起始位置
        src += n_per_row;
        # 移动量化后数据的指针到下一行的起始位置
        qrow += row_size;
    }
    # 返回处理后的数据总大小
    return nrow * row_size;
// 实现对一行数据进行 Q5_0 精度的量化，将结果存储在指定的数据结构中
static void quantize_row_q5_0_impl(const float * restrict x, block_q5_0 * restrict y, int n_per_row, const float * quant_weights) {
    // 确保 QK5_0 的值为 32
    static_assert(QK5_0 == 32, "QK5_0 must be 32");

    // 如果没有量化权重，则使用参考函数 quantize_row_q5_0_reference 进行量化并返回
    if (!quant_weights) {
        quantize_row_q5_0_reference(x, y, n_per_row);
        return;
    }

    // 初始化权重数组和量化结果数组
    float weight[QK5_0];
    int8_t L[QK5_0];

    // 计算输入数据平方和
    float sum_x2 = 0;
    for (int j = 0; j < n_per_row; ++j) sum_x2 += x[j]*x[j];
    float sigma2 = sum_x2/n_per_row;

    // 计算每个数据块的数量
    const int nb = n_per_row/QK5_0;
    for (int ib = 0; ib < nb; ++ib) {
        // 获取当前数据块的指针和对应的量化权重指针
        const float * xb = x + QK5_0 * ib;
        const float * qw = quant_weights + QK5_0 * ib;
        // 计算权重值
        for (int j = 0; j < QK5_0; ++j) weight[j] = qw[j] * sqrtf(sigma2 + xb[j]*xb[j]);
        // 进行 Q5_0 精度的量化
        float d = make_qx_quants(QK5_0, 16, xb, L, 1, weight);
        // 将结果转换为 FP16 存储在输出数据结构中
        y[ib].d = GGML_FP32_TO_FP16(d);

        // 初始化存储高位信息的变量
        uint32_t qh = 0;

        // 将量化结果存储到输出数据结构中
        for (int j = 0; j < 16; ++j) {
            const uint8_t xi0 = L[j];
            const uint8_t xi1 = L[j+16];
            y[ib].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第 5 位并将其存储在 qh 变量的正确位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + QK5_0/2);
        }

        // 将高位信息存储到输出数据结构中
        memcpy(&y[ib].qh, &qh, sizeof(qh));
    }
}

// 对输入数据进行 Q5_0 精度的量化，并将结果存储在输出缓冲区中
size_t quantize_q5_0(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 如果没有量化权重，则调用 ggml_quantize_q5_0 函数进行量化并返回结果
    if (!quant_weights) {
        return ggml_quantize_q5_0(src, dst, nrow*n_per_row, n_per_row, hist);
    }
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q5_0, n_per_row);
    char * qrow = (char *)dst;
    // 遍历每行数据进行量化
    for (int row = 0; row < nrow; ++row) {
        quantize_row_q5_0_impl(src, (block_q5_0*)qrow, n_per_row, quant_weights);
        src += n_per_row;
        qrow += row_size;
    }
    // 返回总共量化的字节数
    return nrow * row_size;
}

// 实现对一行数据进行 Q5_1 精度的量化，将结果存储在指定的数据结构中
static void quantize_row_q5_1_impl(const float * restrict x, block_q5_1 * restrict y, int n_per_row, const float * quant_weights) {
    // 确保 QK5_1 的值为 32，否则触发静态断言错误
    static_assert(QK5_1 == 32, "QK5_1 must be 32");

    // 如果量化权重为空，则使用参考方法 quantize_row_q5_1_reference 进行量化
    if (!quant_weights) {
        quantize_row_q5_1_reference(x, y, n_per_row);
        return;
    }

    // 定义权重数组和两个长度为 QK5_1 的无符号整型数组
    float weight[QK5_1];
    uint8_t L[QK5_1], Laux[QK5_1];

    // 计算输入向量 x 的平方和 sum_x2 和方差 sigma2
    float sum_x2 = 0;
    for (int j = 0; j < n_per_row; ++j) sum_x2 += x[j]*x[j];
    float sigma2 = sum_x2/n_per_row;

    // 计算每个子块的数量 nb
    const int nb = n_per_row/QK5_1;
    for (int ib = 0; ib < nb; ++ib) {
        // 指向当前子块的指针 xb 和对应的量化权重指针 qw
        const float * xb = x + QK5_1 * ib;
        const float * qw = quant_weights + QK5_1 * ib;
        
        // 计算权重数组 weight，并调用 make_qkx3_quants 函数进行量化
        for (int j = 0; j < QK5_1; ++j) weight[j] = qw[j] * sqrtf(sigma2 + xb[j]*xb[j]);
        float min;
        float d = make_qkx3_quants(QK5_1, 31, xb, weight, L, &min, Laux, -0.9f, 0.05f, 36, false);
        
        // 将量化结果存储到输出 y 中
        y[ib].d = GGML_FP32_TO_FP16(d);
        y[ib].m = GGML_FP32_TO_FP16(-min);

        // 处理量化结果的高位信息
        uint32_t qh = 0;
        for (int j = 0; j < 16; ++j) {
            const uint8_t xi0 = L[j];
            const uint8_t xi1 = L[j+16];
            y[ib].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);
            // 获取第 5 位并将其存储在 qh 中的正确位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + QK5_0/2);
        }
        // 将 qh 的值复制到输出 y 中
        memcpy(&y[ib].qh, &qh, sizeof(qh));
    }
// 定义一个函数，用于将浮点数组量化为 Q5.1 格式的数据，并返回量化后的数据大小
size_t quantize_q5_1(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    // 如果量化权重为空，则调用 ggml_quantize_q5_1 函数进行量化
    if (!quant_weights) {
        return ggml_quantize_q5_1(src, dst, nrow*n_per_row, n_per_row, hist);
    }
    // 计算每行数据的大小
    size_t row_size = ggml_row_size(GGML_TYPE_Q5_1, n_per_row);
    // 将目标数据转换为字符指针
    char * qrow = (char *)dst;
    // 遍历每一行数据
    for (int row = 0; row < nrow; ++row) {
        // 调用 quantize_row_q5_1_impl 函数进行 Q5.1 格式的行数据量化
        quantize_row_q5_1_impl(src, (block_q5_1*)qrow, n_per_row, quant_weights);
        // 更新源数据指针和目标数据指针
        src += n_per_row;
        qrow += row_size;
    }
    // 返回总数据大小
    return nrow * row_size;
}

// 定义一个静态的 256 个元素的 uint64_t 数组，用于存储 "True" 2-bit (de)-quantization 的网格数据
static const  uint64_t iq2xxs_grid[256] = {
    // 256 个元素的具体数值，用于 "True" 2-bit (de)-quantization
};
    # 定义一系列十六进制数值
    0x08082b0819081908, 0x08082b0819190808, 0x08082b081919082b, 0x08082b082b082b08,
    0x08082b1908081908, 0x08082b1919080808, 0x08082b2b0808082b, 0x08082b2b08191908,
    0x0819080808080819, 0x0819080808081908, 0x0819080808190808, 0x08190808082b0819,
    0x0819080819080808, 0x08190808192b0808, 0x081908082b081908, 0x081908082b190808,
    0x081908082b191919, 0x0819081908080808, 0x0819081908082b08, 0x08190819082b0808,
    0x0819081919190808, 0x0819081919192b2b, 0x081908192b080808, 0x0819082b082b1908,
    0x0819082b19081919, 0x0819190808080808, 0x0819190808082b08, 0x08191908082b0808,
    0x08191908082b1919, 0x0819190819082b19, 0x081919082b080808, 0x0819191908192b08,
    0x08191919192b082b, 0x0819192b08080808, 0x0819192b0819192b, 0x08192b0808080819,
    0x08192b0808081908, 0x08192b0808190808, 0x08192b0819080808, 0x08192b082b080819,
    0x08192b1908080808, 0x08192b1908081919, 0x08192b192b2b0808, 0x08192b2b19190819,
    0x082b080808080808, 0x082b08080808082b, 0x082b080808082b2b, 0x082b080819081908,
    0x082b0808192b0819, 0x082b08082b080808, 0x082b08082b08082b, 0x082b0819082b2b19,
    0x082b081919082b08, 0x082b082b08080808, 0x082b082b0808082b, 0x082b190808080819,
    0x082b190808081908, 0x082b190808190808, 0x082b190819080808, 0x082b19081919192b,
    0x082b191908080808, 0x082b191919080819, 0x082b1919192b1908, 0x082b192b2b190808,
    0x082b2b0808082b08, 0x082b2b08082b0808, 0x082b2b082b191908, 0x082b2b2b19081908,
    0x1908080808080819, 0x1908080808081908, 0x1908080808190808, 0x1908080808192b08,
    0x19080808082b0819, 0x19080808082b1908, 0x1908080819080808, 0x1908080819082b08,
    0x190808081919192b, 0x19080808192b0808, 0x190808082b080819, 0x190808082b081908,
    0x190808082b190808, 0x1908081908080808, 0x19080819082b0808, 0x19080819192b0819,
    0x190808192b080808, 0x190808192b081919, 0x1908082b08080819, 0x1908082b08190808,
    0x1908082b19082b08, 0x1908082b1919192b, 0x1908082b192b2b08, 0x1908190808080808
    # 定义一系列十六进制数值
    0x1908190808082b08, 0x19081908082b0808, 0x190819082b080808, 0x190819082b192b19,
    0x190819190819082b, 0x19081919082b1908, 0x1908192b08080808, 0x19082b0808080819,
    0x19082b0808081908, 0x19082b0808190808, 0x19082b0819080808, 0x19082b0819081919,
    0x19082b1908080808, 0x19082b1919192b08, 0x19082b19192b0819, 0x19082b192b08082b,
    0x19082b2b19081919, 0x19082b2b2b190808, 0x1919080808080808, 0x1919080808082b08,
    0x1919080808190819, 0x1919080808192b19, 0x19190808082b0808, 0x191908082b080808,
    0x191908082b082b08, 0x1919081908081908, 0x191908191908082b, 0x191908192b2b1908,
    0x1919082b2b190819, 0x191919082b190808, 0x191919082b19082b, 0x1919191908082b2b,
    0x1919192b08080819, 0x1919192b19191908, 0x19192b0808080808, 0x19192b0808190819,
    0x19192b0808192b19, 0x19192b08192b1908, 0x19192b1919080808, 0x19192b2b08082b08,
    0x192b080808081908, 0x192b080808190808, 0x192b080819080808, 0x192b0808192b2b08,
    0x192b081908080808, 0x192b081919191919, 0x192b082b08192b08, 0x192b082b192b0808,
    0x192b190808080808, 0x192b190808081919, 0x192b191908190808, 0x192b19190819082b,
    0x192b19192b081908, 0x192b2b081908082b, 0x2b08080808080808, 0x2b0808080808082b,
    0x2b08080808082b2b, 0x2b08080819080819, 0x2b0808082b08082b, 0x2b08081908081908,
    0x2b08081908192b08, 0x2b08081919080808, 0x2b08082b08190819, 0x2b08190808080819,
    0x2b08190808081908, 0x2b08190808190808, 0x2b08190808191919, 0x2b08190819080808,
    0x2b081908192b0808, 0x2b08191908080808, 0x2b0819191908192b, 0x2b0819192b191908,
    0x2b08192b08082b19, 0x2b08192b19080808, 0x2b08192b192b0808, 0x2b082b080808082b,
    0x2b082b1908081908, 0x2b082b2b08190819, 0x2b19080808081908, 0x2b19080808190808,
    0x2b190808082b1908, 0x2b19080819080808, 0x2b1908082b2b0819, 0x2b1908190819192b,
    0x2b1908192b080808, 0x2b19082b19081919, 0x2b19190808080808, 0x2b191908082b082b,
    0x2b19190819081908, 0x2b19191919190819, 0x2b192b082b080819, 0x2b192b19082b0808,
    # 这些是十六进制数值，表示一系列数据
    0x2b2b08080808082b, 0x2b2b080819190808, 0x2b2b08082b081919, 0x2b2b081908082b19,
    0x2b2b082b08080808, 0x2b2b190808192b08, 0x2b2b2b0819190808, 0x2b2b2b1908081908,
// 定义一个静态的 uint64_t 类型的数组，包含512个元素
static const uint64_t iq2xs_grid[512] = {
    // 以下为数组的初始化值
    0x0808080808080808, 0x080808080808082b, 0x0808080808081919, 0x0808080808082b08,
    0x0808080808082b2b, 0x0808080808190819, 0x0808080808191908, 0x080808080819192b,
    0x0808080808192b19, 0x08080808082b0808, 0x08080808082b082b, 0x08080808082b1919,
    0x08080808082b2b08, 0x0808080819080819, 0x0808080819081908, 0x080808081908192b,
    0x0808080819082b19, 0x0808080819190808, 0x080808081919082b, 0x0808080819191919,
    0x0808080819192b08, 0x08080808192b0819, 0x08080808192b1908, 0x080808082b080808,
    0x080808082b08082b, 0x080808082b081919, 0x080808082b082b08, 0x080808082b190819,
    0x080808082b191908, 0x080808082b192b19, 0x080808082b2b0808, 0x0808081908080819,
    0x0808081908081908, 0x080808190808192b, 0x0808081908082b19, 0x0808081908190808,
    0x080808190819082b, 0x0808081908191919, 0x0808081908192b08, 0x0808081908192b2b,
    0x08080819082b0819, 0x08080819082b1908, 0x0808081919080808, 0x080808191908082b,
    0x0808081919081919, 0x0808081919082b08, 0x0808081919190819, 0x0808081919191908,
    0x08080819192b0808, 0x08080819192b2b08, 0x080808192b080819, 0x080808192b081908,
    0x080808192b190808, 0x0808082b08080808, 0x0808082b0808082b, 0x0808082b08081919,
    0x0808082b08082b08, 0x0808082b08190819, 0x0808082b08191908, 0x0808082b082b0808,
    0x0808082b19080819, 0x0808082b19081908, 0x0808082b19190808, 0x0808082b19191919,
    0x0808082b2b080808, 0x0808082b2b082b2b, 0x0808190808080819, 0x0808190808081908,
    0x080819080808192b, 0x0808190808082b19, 0x0808190808190808, 0x080819080819082b,
    0x0808190808191919, 0x0808190808192b08, 0x08081908082b0819, 0x08081908082b1908,
    0x0808190819080808, 0x080819081908082b, 0x0808190819081919, 0x0808190819082b08,
    0x0808190819190819, 0x0808190819191908, 0x080819081919192b, 0x08081908192b0808,
    0x080819082b080819, 0x080819082b081908, 0x080819082b190808, 0x0808191908080808,
    0x080819190808082b, 0x0808191908081919, 0x0808191908082b08, 0x0808191908190819,
    // 以上为部分数组初始化值，省略了后续的初始化
    # 定义一系列十六进制数值
    0x0808191908191908, 0x08081919082b0808, 0x0808191919080819, 0x0808191919081908,
    0x0808191919190808, 0x08081919192b0819, 0x080819192b080808, 0x0808192b08080819,
    0x0808192b08081908, 0x0808192b08190808, 0x0808192b082b192b, 0x0808192b19080808,
    0x0808192b1908082b, 0x0808192b2b081908, 0x08082b0808080808, 0x08082b080808082b,
    0x08082b0808081919, 0x08082b0808082b08, 0x08082b0808082b2b, 0x08082b0808190819,
    0x08082b0808191908, 0x08082b08082b0808, 0x08082b08082b1919, 0x08082b0819080819,
    0x08082b0819081908, 0x08082b0819190808, 0x08082b0819192b08, 0x08082b082b080808,
    0x08082b082b2b0808, 0x08082b082b2b2b2b, 0x08082b1908080819, 0x08082b1908081908,
    0x08082b1908190808, 0x08082b1919080808, 0x08082b192b080819, 0x08082b192b082b19,
    0x08082b2b08080808, 0x08082b2b082b0808, 0x08082b2b082b2b08, 0x08082b2b2b19192b,
    0x08082b2b2b2b0808, 0x0819080808080819, 0x0819080808081908, 0x081908080808192b,
    0x0819080808082b19, 0x0819080808190808, 0x081908080819082b, 0x0819080808191919,
    0x0819080808192b08, 0x08190808082b0819, 0x08190808082b1908, 0x0819080819080808,
    0x081908081908082b, 0x0819080819081919, 0x0819080819082b08, 0x0819080819190819,
    0x0819080819191908, 0x08190808192b0808, 0x08190808192b2b2b, 0x081908082b080819,
    0x081908082b081908, 0x081908082b190808, 0x0819081908080808, 0x081908190808082b,
    0x0819081908081919, 0x0819081908082b08, 0x0819081908190819, 0x0819081908191908,
    0x08190819082b0808, 0x0819081919080819, 0x0819081919081908, 0x0819081919190808,
    0x081908192b080808, 0x081908192b191908, 0x081908192b19192b, 0x0819082b08080819,
    0x0819082b08081908, 0x0819082b0808192b, 0x0819082b08190808, 0x0819082b19080808,
    0x0819082b192b0808, 0x0819190808080808, 0x081919080808082b, 0x0819190808081919,
    0x0819190808082b08, 0x0819190808190819, 0x0819190808191908, 0x08191908082b0808,
    0x0819190819080819, 0x0819190819081908, 0x0819190819082b19, 0x0819190819190808,
    # 定义一系列十六进制数值
    0x08191908192b1908, 0x081919082b080808, 0x0819191908080819, 0x0819191908081908,
    0x0819191908190808, 0x0819191919080808, 0x0819192b08080808, 0x0819192b08191908,
    0x0819192b19082b19, 0x08192b0808080819, 0x08192b0808081908, 0x08192b0808190808,
    0x08192b080819082b, 0x08192b0819080808, 0x08192b0819191908, 0x08192b082b08192b,
    0x08192b1908080808, 0x08192b1908081919, 0x08192b19192b192b, 0x08192b2b19190819,
    0x08192b2b2b2b2b19, 0x082b080808080808, 0x082b08080808082b, 0x082b080808081919,
    0x082b080808082b08, 0x082b080808082b2b, 0x082b080808190819, 0x082b080808191908,
    0x082b0808082b0808, 0x082b080819080819, 0x082b080819081908, 0x082b080819190808,
    0x082b08082b080808, 0x082b08082b2b0808, 0x082b081908080819, 0x082b081908081908,
    0x082b081908190808, 0x082b081919080808, 0x082b081919082b08, 0x082b0819192b1919,
    0x082b082b08080808, 0x082b082b082b082b, 0x082b082b2b080808, 0x082b082b2b2b2b08,
    0x082b190808080819, 0x082b190808081908, 0x082b190808190808, 0x082b1908082b2b19,
    0x082b190819080808, 0x082b191908080808, 0x082b191919080819, 0x082b19191919082b,
    0x082b19192b192b19, 0x082b192b08080819, 0x082b192b08192b2b, 0x082b192b2b2b192b,
    0x082b2b0808080808, 0x082b2b0808082b08, 0x082b2b0808082b2b, 0x082b2b08082b0808,
    0x082b2b0819191919, 0x082b2b082b082b08, 0x082b2b082b2b082b, 0x082b2b19192b2b08,
    0x082b2b192b190808, 0x082b2b2b08082b08, 0x082b2b2b082b0808, 0x082b2b2b2b08082b,
    0x082b2b2b2b082b08, 0x082b2b2b2b082b2b, 0x1908080808080819, 0x1908080808081908,
    0x190808080808192b, 0x1908080808082b19, 0x1908080808190808, 0x190808080819082b,
    0x1908080808191919, 0x1908080808192b08, 0x19080808082b0819, 0x19080808082b1908,
    0x1908080819080808, 0x190808081908082b, 0x1908080819081919, 0x1908080819082b08,
    0x1908080819082b2b, 0x1908080819190819, 0x1908080819191908, 0x19080808192b0808,
    0x19080808192b1919, 0x190808082b080819, 0x190808082b081908, 0x190808082b190808,
    # 定义一系列十六进制数值
    0x1908081908080808, 0x190808190808082b, 0x1908081908081919, 0x1908081908082b08,
    0x1908081908190819, 0x1908081908191908, 0x19080819082b0808, 0x1908081919080819,
    0x1908081919081908, 0x1908081919190808, 0x190808192b080808, 0x190808192b081919,
    0x190808192b2b082b, 0x1908082b08080819, 0x1908082b08081908, 0x1908082b08190808,
    0x1908082b0819082b, 0x1908082b082b2b19, 0x1908082b19080808, 0x1908190808080808,
    0x190819080808082b, 0x1908190808081919, 0x1908190808082b08, 0x1908190808190819,
    0x1908190808191908, 0x1908190808192b19, 0x19081908082b0808, 0x1908190819080819,
    0x1908190819081908, 0x1908190819190808, 0x190819082b080808, 0x190819082b191908,
    0x1908191908080819, 0x1908191908081908, 0x1908191908190808, 0x19081919082b1908,
    0x1908191919080808, 0x190819192b192b2b, 0x1908192b08080808, 0x1908192b08082b2b,
    0x1908192b19081908, 0x1908192b19190808, 0x19082b0808080819, 0x19082b0808081908,
    0x19082b0808190808, 0x19082b0819080808, 0x19082b0819081919, 0x19082b0819191908,
    0x19082b08192b082b, 0x19082b1908080808, 0x19082b1908190819, 0x19082b1919081908,
    0x19082b1919190808, 0x19082b19192b2b19, 0x19082b2b08081908, 0x1919080808080808,
    0x191908080808082b, 0x1919080808081919, 0x1919080808082b08, 0x1919080808190819,
    0x1919080808191908, 0x19190808082b0808, 0x19190808082b2b08, 0x1919080819080819,
    0x1919080819081908, 0x1919080819190808, 0x191908082b080808, 0x1919081908080819,
    0x1919081908081908, 0x1919081908190808, 0x1919081908191919, 0x1919081919080808,
    0x191908191908082b, 0x1919082b08080808, 0x1919082b19081908, 0x1919082b2b2b2b2b,
    0x1919190808080819, 0x1919190808081908, 0x1919190808190808, 0x19191908082b0819,
    0x1919190819080808, 0x19191908192b0808, 0x191919082b080819, 0x191919082b2b0819,
    0x1919191908080808, 0x1919191908082b08, 0x191919192b080808, 0x191919192b082b08,
    0x1919192b082b0819, 0x1919192b192b2b08, 0x1919192b2b2b0819, 0x19192b0808080808,
    # 定义一系列十六进制数值
    0x19192b0808191908, 0x19192b0819080819, 0x19192b0819190808, 0x19192b082b192b19,
    0x19192b1908192b2b, 0x19192b1919080808, 0x19192b191908082b, 0x19192b2b2b081919,
    0x192b080808080819, 0x192b080808081908, 0x192b080808190808, 0x192b080819080808,
    0x192b080819191908, 0x192b0808192b082b, 0x192b08082b08192b, 0x192b08082b2b2b19,
    0x192b081908080808, 0x192b082b082b1908, 0x192b082b19082b2b, 0x192b082b2b19082b,
    0x192b190808080808, 0x192b19080819192b, 0x192b191908190808, 0x192b191919080808,
    0x192b191919081919, 0x192b19192b2b1908, 0x192b2b0808080819, 0x192b2b08192b2b2b,
    0x192b2b19082b1919, 0x192b2b2b0808192b, 0x192b2b2b19191908, 0x192b2b2b192b082b,
    0x2b08080808080808, 0x2b0808080808082b, 0x2b08080808081919, 0x2b08080808082b08,
    0x2b08080808190819, 0x2b08080808191908, 0x2b080808082b0808, 0x2b080808082b2b2b,
    0x2b08080819080819, 0x2b08080819081908, 0x2b08080819190808, 0x2b0808082b080808,
    0x2b0808082b08082b, 0x2b0808082b2b2b08, 0x2b0808082b2b2b2b, 0x2b08081908080819,
    0x2b08081908081908, 0x2b0808190808192b, 0x2b08081908190808, 0x2b08081919080808,
    0x2b08081919190819, 0x2b08081919192b19, 0x2b08082b08080808, 0x2b08082b082b0808,
    0x2b08082b2b080808, 0x2b08082b2b08082b, 0x2b08082b2b2b0808, 0x2b08082b2b2b2b08,
    0x2b08190808080819, 0x2b08190808081908, 0x2b08190808190808, 0x2b0819080819082b,
    0x2b08190808191919, 0x2b08190819080808, 0x2b081908192b0808, 0x2b0819082b082b19,
    0x2b08191908080808, 0x2b08191919081908, 0x2b0819192b2b1919, 0x2b08192b08192b08,
    0x2b08192b192b2b2b, 0x2b082b0808080808, 0x2b082b0808082b08, 0x2b082b08082b1919,
    0x2b082b0819192b2b, 0x2b082b082b080808, 0x2b082b082b08082b, 0x2b082b082b2b2b08,
    0x2b082b190808192b, 0x2b082b2b082b082b, 0x2b082b2b2b080808, 0x2b082b2b2b082b08,
    0x2b082b2b2b19192b, 0x2b082b2b2b2b2b08, 0x2b19080808080819, 0x2b19080808081908,
    0x2b19080808190808, 0x2b19080819080808, 0x2b1908081919192b, 0x2b1908082b081908,
    0x2b19081908080808, 0x2b190819082b082b, 0x2b190819192b1908, 0x2b19082b1919192b,
    0x2b19082b2b082b19, 0x2b19190808080808, 0x2b19190808081919, 0x2b19190819081908,
    0x2b19190819190808, 0x2b19190819192b08, 0x2b191919082b2b19, 0x2b1919192b190808,
    0x2b1919192b19082b, 0x2b19192b19080819, 0x2b192b0819190819, 0x2b192b082b2b192b,
    0x2b192b1919082b19, 0x2b192b2b08191919, 0x2b192b2b192b0808, 0x2b2b080808080808,
    0x2b2b08080808082b, 0x2b2b080808082b08, 0x2b2b080808082b2b, 0x2b2b0808082b0808,
    0x2b2b0808082b2b2b, 0x2b2b08082b2b0808, 0x2b2b081919190819, 0x2b2b081919192b19,
    0x2b2b08192b2b192b, 0x2b2b082b08080808, 0x2b2b082b0808082b, 0x2b2b082b08082b08,
    0x2b2b082b082b2b2b, 0x2b2b082b2b080808, 0x2b2b082b2b2b0808, 0x2b2b190819080808,
    0x2b2b19082b191919, 0x2b2b192b192b1919, 0x2b2b192b2b192b08, 0x2b2b2b0808082b2b,
    0x2b2b2b08082b0808, 0x2b2b2b08082b082b, 0x2b2b2b08082b2b08, 0x2b2b2b082b2b0808,
    0x2b2b2b082b2b2b08, 0x2b2b2b1908081908, 0x2b2b2b192b081908, 0x2b2b2b192b08192b,
    0x2b2b2b2b082b2b08, 0x2b2b2b2b082b2b2b, 0x2b2b2b2b2b190819, 0x2b2b2b2b2b2b2b2b,
// 定义一个静态的无符号32位整数数组，包含256个元素
static const uint32_t iq3xxs_grid[256] = {
    // 数组元素依次为16进制数
    0x04040404, 0x04040414, 0x04040424, 0x04040c0c, 0x04040c1c, 0x04040c3e, 0x04041404, 0x04041414,
    0x04041c0c, 0x04042414, 0x04043e1c, 0x04043e2c, 0x040c040c, 0x040c041c, 0x040c0c04, 0x040c0c14,
    0x040c140c, 0x040c142c, 0x040c1c04, 0x040c1c14, 0x040c240c, 0x040c2c24, 0x040c3e04, 0x04140404,
    0x04140414, 0x04140424, 0x04140c0c, 0x04141404, 0x04141414, 0x04141c0c, 0x04141c1c, 0x04141c3e,
    0x04142c0c, 0x04142c3e, 0x04143e2c, 0x041c040c, 0x041c043e, 0x041c0c04, 0x041c0c14, 0x041c142c,
    0x041c3e04, 0x04240c1c, 0x04241c3e, 0x04242424, 0x04242c3e, 0x04243e1c, 0x04243e2c, 0x042c040c,
    0x042c043e, 0x042c1c14, 0x042c2c14, 0x04341c2c, 0x04343424, 0x043e0c04, 0x043e0c24, 0x043e0c34,
    0x043e241c, 0x043e340c, 0x0c04040c, 0x0c04041c, 0x0c040c04, 0x0c040c14, 0x0c04140c, 0x0c04141c,
    0x0c041c04, 0x0c041c14, 0x0c041c24, 0x0c04243e, 0x0c042c04, 0x0c0c0404, 0x0c0c0414, 0x0c0c0c0c,
    0x0c0c1404, 0x0c0c1414, 0x0c14040c, 0x0c14041c, 0x0c140c04, 0x0c140c14, 0x0c14140c, 0x0c141c04,
    0x0c143e14, 0x0c1c0404, 0x0c1c0414, 0x0c1c1404, 0x0c1c1c0c, 0x0c1c2434, 0x0c1c3434, 0x0c24040c,
    0x0c24042c, 0x0c242c04, 0x0c2c1404, 0x0c2c1424, 0x0c2c2434, 0x0c2c3e0c, 0x0c34042c, 0x0c3e1414,
    0x0c3e2404, 0x14040404, 0x14040414, 0x14040c0c, 0x14040c1c, 0x14041404, 0x14041414, 0x14041434,
    0x14041c0c, 0x14042414, 0x140c040c, 0x140c041c, 0x140c042c, 0x140c0c04, 0x140c0c14, 0x140c140c,
    0x140c1c04, 0x140c341c, 0x140c343e, 0x140c3e04, 0x14140404, 0x14140414, 0x14140c0c, 0x14140c3e,
    0x14141404, 0x14141414, 0x14141c3e, 0x14142404, 0x14142c2c, 0x141c040c, 0x141c0c04, 0x141c0c24,
    0x141c3e04, 0x141c3e24, 0x14241c2c, 0x14242c1c, 0x142c041c, 0x142c143e, 0x142c240c, 0x142c3e24,
    0x143e040c, 0x143e041c, 0x143e0c34, 0x143e242c, 0x1c04040c, 0x1c040c04, 0x1c040c14, 0x1c04140c,
    0x1c04141c, 0x1c042c04, 0x1c04342c, 0x1c043e14, 0x1c0c0404, 0x1c0c0414, 0x1c0c1404, 0x1c0c1c0c,
};
    0x1c0c2424, 0x1c0c2434, 0x1c14040c, 0x1c14041c, 0x1c140c04, 0x1c14142c, 0x1c142c14, 0x1c143e14,
    0x1c1c0c0c, 0x1c1c1c1c, 0x1c241c04, 0x1c24243e, 0x1c243e14, 0x1c2c0404, 0x1c2c0434, 0x1c2c1414,
    0x1c2c2c2c, 0x1c340c24, 0x1c341c34, 0x1c34341c, 0x1c3e1c1c, 0x1c3e3404, 0x24040424, 0x24040c3e,
    0x24041c2c, 0x24041c3e, 0x24042c1c, 0x24042c3e, 0x240c3e24, 0x24141404, 0x24141c3e, 0x24142404,
    0x24143404, 0x24143434, 0x241c043e, 0x241c242c, 0x24240424, 0x24242c0c, 0x24243424, 0x242c142c,
    0x242c241c, 0x242c3e04, 0x243e042c, 0x243e0c04, 0x243e0c14, 0x243e1c04, 0x2c040c14, 0x2c04240c,
    0x2c043e04, 0x2c0c0404, 0x2c0c0434, 0x2c0c1434, 0x2c0c2c2c, 0x2c140c24, 0x2c141c14, 0x2c143e14,
    0x2c1c0414, 0x2c1c2c1c, 0x2c240c04, 0x2c24141c, 0x2c24143e, 0x2c243e14, 0x2c2c0414, 0x2c2c1c0c,
    0x2c342c04, 0x2c3e1424, 0x2c3e2414, 0x34041424, 0x34042424, 0x34042434, 0x34043424, 0x340c140c,
    0x340c340c, 0x34140c3e, 0x34143424, 0x341c1c04, 0x341c1c34, 0x34242424, 0x342c042c, 0x342c2c14,
    0x34341c1c, 0x343e041c, 0x343e140c, 0x3e04041c, 0x3e04042c, 0x3e04043e, 0x3e040c04, 0x3e041c14,
    0x3e042c14, 0x3e0c1434, 0x3e0c2404, 0x3e140c14, 0x3e14242c, 0x3e142c14, 0x3e1c0404, 0x3e1c0c2c,
    0x3e1c1c1c, 0x3e1c3404, 0x3e24140c, 0x3e24240c, 0x3e2c0404, 0x3e2c0414, 0x3e2c1424, 0x3e341c04,
// IQ2XS 码字的符号表，128个元素
static const uint8_t ksigns_iq2xs[128] = {
      0, 129, 130,   3, 132,   5,   6, 135, 136,   9,  10, 139,  12, 141, 142,  15,
    144,  17,  18, 147,  20, 149, 150,  23,  24, 153, 154,  27, 156,  29,  30, 159,
    160,  33,  34, 163,  36, 165, 166,  39,  40, 169, 170,  43, 172,  45,  46, 175,
     48, 177, 178,  51, 180,  53,  54, 183, 184,  57,  58, 187,  60, 189, 190,  63,
    192,  65,  66, 195,  68, 197, 198,  71,  72, 201, 202,  75, 204,  77,  78, 207,
     80, 209, 210,  83, 212,  85,  86, 215, 216,  89,  90, 219,  92, 221, 222,  95,
     96, 225, 226,  99, 228, 101, 102, 231, 232, 105, 106, 235, 108, 237, 238, 111,
    240, 113, 114, 243, 116, 245, 246, 119, 120, 249, 250, 123, 252, 125, 126, 255,
};

// IQ2XS 码字的掩码表，8个元素
static const uint8_t kmask_iq2xs[8] = {1, 2, 4, 8, 16, 32, 64, 128};

// 对 IQ2XXS 码字进行反量化
void dequantize_row_iq2_xxs(const block_iq2_xxs * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    // 辅助变量
    uint32_t aux32[2];
    const uint8_t * aux8 = (const uint8_t *)aux32;

    for (int i = 0; i < nb; i++) {
        // 将 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        for (int ib32 = 0; ib32 < QK_K/32; ++ib32) {
            // 从 x[i].qs 中复制 2 个 uint32_t 到 aux32
            memcpy(aux32, x[i].qs + 4*ib32, 2*sizeof(uint32_t));
            // 计算 db
            const float db = d * (0.5f + (aux32[1] >> 28)) * 0.25f;
            for (int l = 0; l < 4; ++l) {
                // 获取 IQ2XXS 码字格点
                const uint8_t * grid = (const uint8_t *)(iq2xxs_grid + aux8[l]);
                // 获取符号
                const uint8_t  signs = ksigns_iq2xs[(aux32[1] >> 7*l) & 127];
                for (int j = 0; j < 8; ++j) {
                    // 计算 y[j]
                    y[j] = db * grid[j] * (signs & kmask_iq2xs[j] ? -1.f : 1.f);
                }
                y += 8;
            }
        }
    }
}

// 对 IQ2XS 码字进行反量化
void dequantize_row_iq2_xs(const block_iq2_xs * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    // 辅助变量数组
    float db[2];
}
    // 遍历 nb 次，nb 为一个整数，表示循环次数
    for (int i = 0; i < nb; i++) {

        // 将 x[i].d 转换为 float 类型，并赋值给 d
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历 QK_K/32 次，QK_K 为一个整数，每次循环处理 32 个元素
        for (int ib32 = 0; ib32 < QK_K/32; ++ib32) {
            // 计算 db[0] 和 db[1] 的值
            db[0] = d * (0.5f + (x[i].scales[ib32] & 0xf)) * 0.25f;
            db[1] = d * (0.5f + (x[i].scales[ib32] >>  4)) * 0.25f;
            // 遍历 4 次，每次处理一个元素
            for (int l = 0; l < 4; ++l) {
                // 获取 iq2xs_grid 中的数据和 ksigns_iq2xs 中的数据
                const uint8_t * grid = (const uint8_t *)(iq2xs_grid + (x[i].qs[4*ib32 + l] & 511));
                const uint8_t  signs = ksigns_iq2xs[x[i].qs[4*ib32 + l] >> 9];
                // 遍历 8 次，每次处理一个元素
                for (int j = 0; j < 8; ++j) {
                    // 计算 y[j] 的值
                    y[j] = db[l/2] * grid[j] * (signs & kmask_iq2xs[j] ? -1.f : 1.f);
                }
                // y 指针向后移动 8 个位置
                y += 8;
            }
        }
    }
// ====================== 3.0625 bpw (de)-quantization

// 对输入的 IQ3 数据进行反量化操作，将结果存储到 float 类型的数组中
void dequantize_row_iq3_xxs(const block_iq3_xxs * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    uint32_t aux32;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {

        // 将 IQ3 数据转换为 float 类型
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const uint8_t * qs = x[i].qs;
        const uint8_t * scales_and_signs = qs + QK_K/4;

        // 遍历每个 32 位块
        for (int ib32 = 0; ib32 < QK_K/32; ++ib32) {
            // 从 scales_and_signs 中复制 32 位数据
            memcpy(&aux32, scales_and_signs + 4*ib32, sizeof(uint32_t));
            // 计算 db 值
            const float db = d * (0.5f + (aux32 >> 28)) * 0.5f;
            // 遍历每个 4 位块
            for (int l = 0; l < 4; ++l) {
                // 获取符号信息
                const uint8_t  signs = ksigns_iq2xs[(aux32 >> 7*l) & 127];
                const uint8_t * grid1 = (const uint8_t *)(iq3xxs_grid + qs[2*l+0]);
                const uint8_t * grid2 = (const uint8_t *)(iq3xxs_grid + qs[2*l+1]);
                // 遍历每个 4 位数据
                for (int j = 0; j < 4; ++j) {
                    // 计算 y 值并根据符号进行调整
                    y[j+0] = db * grid1[j] * (signs & kmask_iq2xs[j+0] ? -1.f : 1.f);
                    y[j+4] = db * grid2[j] * (signs & kmask_iq2xs[j+4] ? -1.f : 1.f);
                }
                y += 8;
            }
            qs += 8;
        }
    }
}

//===================================== Q8_K ==============================================

// 对输入的 float 数组进行量化操作，将结果存储到 Q8_K 结构体数组中
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 初始化最大值和绝对值最大值
        float max = 0;
        float amax = 0;
        // 内层循环遍历 QK_K 次
        for (int j = 0; j < QK_K; ++j) {
            // 计算当前元素的绝对值
            float ax = fabsf(x[j]);
            // 如果绝对值大于绝对值最大值，则更新最大值和绝对值最大值
            if (ax > amax) {
                amax = ax; 
                max = x[j];
            }
        }
        // 如果绝对值最大值为0，则将 y[i] 的值设为0，并将 y[i].qs 数组清零，然后跳过当前循环
        if (!amax) {
            y[i].d = 0;
            memset(y[i].qs, 0, QK_K);
            x += QK_K;
            continue;
        }
        // 计算 iscale 值，用于后续计算
        //const float iscale = -128.f/max;
        // We need this change for IQ2_XXS, else the AVX implementation becomes very awkward
        const float iscale = -127.f/max;
        // 根据 iscale 对 x[j] 进行量化，并存储到 y[i].qs 数组中
        for (int j = 0; j < QK_K; ++j) {
            int v = nearest_int(iscale*x[j]);
            y[i].qs[j] = MIN(127, v);
        }
        // 计算 y[i].bsums[j] 的值
        for (int j = 0; j < QK_K/16; ++j) {
            int sum = 0;
            for (int ii = 0; ii < 16; ++ii) {
                sum += y[i].qs[j*16 + ii];
            }
            y[i].bsums[j] = sum;
        }
        // 更新 y[i].d 的值
        y[i].d = 1/iscale;
        x += QK_K;
    }
// 结束 dequantize_row_q8_K 函数定义
}

// 对输入的 block_q8_K 结构体数组进行反量化操作，结果存储在 float 数组中，每个 block_q8_K 结构体包含 QK_K 个量化值
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k) {
    // 断言 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算 block_q8_K 数组的长度
    const int nb = k / QK_K;

    // 遍历 block_q8_K 数组
    for (int i = 0; i < nb; i++) {
        // 遍历每个 block_q8_K 结构体中的量化值
        for (int j = 0; j < QK_K; ++j) {
            // 对每个量化值进行反量化操作，结果存储在 float 数组中
            *y++ = x[i].d * x[i].qs[j];
        }
    }
}

// 对输入的 float 数组进行量化操作，调用 quantize_row_q8_K_reference 函数
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k) {
    quantize_row_q8_K_reference(x, y, k);
}

//===================================== Dot ptoducts =================================

//
// Helper functions
//

// 如果支持 AVX 指令集，则定义用于 dot product 的辅助函数
#if __AVX__ || __AVX2__ || __AVX512F__

// 用于 dot product 的辅助函数，根据索引 i 获取所需的缩放值的 shuffle
static inline __m256i get_scale_shuffle_q3k(int i) {
    // 预定义的 shuffle 数组，用于选择 dot product 中所需的缩放值
    static const uint8_t k_shuffle[128] = {
         0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,     2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
         4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5,     6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7,
         8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9,    10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,
        12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,    14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,
    };
    // 加载 shuffle 数组中的值到 __m256i 类型的变量中
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
}
// 用于 dot product 的辅助函数，根据索引 i 获取所需的缩放值的 shuffle
static inline __m256i get_scale_shuffle_k4(int i) {
    // 定义一个包含256个元素的常量数组，用于乱序加载数据
    static const uint8_t k_shuffle[256] = {
         0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,
         2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
         4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5,
         6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7,
         8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9,
        10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,
        12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,
        14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15
    };
    // 返回根据索引 i 加载的 256 位数据
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
// 定义一个静态内联函数，返回根据输入参数 i 得到的 __m128i 类型的数据
static inline __m128i get_scale_shuffle(int i) {
    // 预定义的大小为 128 的 uint8_t 类型数组 k_shuffle
    static const uint8_t k_shuffle[128] = {
         // 数组内容省略，包含一系列数字
    };
    // 返回根据数组 k_shuffle 的第 i 个元素构造的 __m128i 类型数据
    return _mm_loadu_si128((const __m128i*)k_shuffle + i);
}
#endif

// 定义一个函数，计算两个向量的点积
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk 为 QK8_0
    const int qk = QK8_0;
    // 计算向量长度 n 能被 qk 整除的块数
    const int nb = n / qk;

    // 断言 n 能被 qk 整除
    assert(n % qk == 0);

    // 将输入指针转换为 block_q4_0 和 block_q8_0 类型的指针
    const block_q4_0 * restrict x = vx;
    const block_q8_0 * restrict y = vy;

    // 如果支持 ARM NEON 指令集
#if defined(__ARM_NEON)
    // 定义两个 float32x4_t 类型的变量 sumv0 和 sumv1，初始化为 0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 断言 nb 是偶数，暂不处理奇数情况
    assert(nb % 2 == 0); // TODO: handle odd nb
    // 循环遍历，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 获取指向 x[i] 的指针
        const block_q4_0 * restrict x0 = &x[i + 0];
        // 获取指向 x[i+1] 的指针
        const block_q4_0 * restrict x1 = &x[i + 1];
        // 获取指向 y[i] 的指针
        const block_q8_0 * restrict y0 = &y[i + 0];
        // 获取指向 y[i+1] 的指针
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个所有元素为 0x0F 的 16 位无符号整数向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
        // 创建一个所有元素为 0x8 的 16 位有符号整数向量
        const int8x16_t  s8b = vdupq_n_s8(0x8);

        // 从 x0 中加载 16 个 8 位无符号整数到向量 v0_0
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        // 从 x1 中加载 16 个 8 位无符号整数到向量 v0_1

        // 将 4 位转换为 8 位
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 减去 8
        const int8x16_t v0_0ls = vsubq_s8(v0_0l, s8b);
        const int8x16_t v0_0hs = vsubq_s8(v0_0h, s8b);
        const int8x16_t v0_1ls = vsubq_s8(v0_1l, s8b);
        const int8x16_t v0_1hs = vsubq_s8(v0_1h, s8b);

        // 从 y 中加载数据
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

        // 计算点积到 int32x4_t
        const int32x4_t p_0 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_0ls, v1_0l), v0_0hs, v1_0h);
        const int32x4_t p_1 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_1ls, v1_1l), v0_1hs, v1_1h);

        // 计算最终结果并更新 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d);
    }

    // 将 sumv0 和 sumv1 的结果相加并存储到 s
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__)
    // 使用 AVX2 指令集

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 现在我们有一个字节向量，范围在 [0 .. 15] 区间。将它们偏移至 [-8 .. +7] 区间。
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );

        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加
        acc = _mm256_fmadd_ps( d, q, acc );
    }

    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 使用 AVX 指令集

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        const __m128i lowMask = _mm_set1_epi8(0xF);
        const __m128i off = _mm_set1_epi8(8);

        const __m128i tmp = _mm_loadu_si128((const __m128i *)x[i].qs);

        __m128i bx = _mm_and_si128(lowMask, tmp);
        __m128i by = _mm_loadu_si128((const __m128i *)y[i].qs);
        bx = _mm_sub_epi8(bx, off);
        const __m128i i32_0 = mul_sum_i8_pairs(bx, by);

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

    // 第一轮不进行累加
    }

    // 断言检查 nb 是否为偶数，TODO: 处理奇数 nb
    assert(nb % 2 == 0);

    // 主循环
    }

    // 将累加器中的值相加，并将结果存储在指针 s 指向的位置
    *s = hsum_float_4x4(acc_0, acc_1, acc_2, acc_3);
#elif defined(__riscv_v_intrinsic)
    // 如果定义了 __riscv_v_intrinsic，则执行以下代码块
    float sumf = 0.0;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 遍历每个块
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
    // 如果未定义 __riscv_v_intrinsic，则执行以下代码块
    // 标量计算
    float sumf = 0.0;

    for (int i = 0; i < nb; i++) {
        int sumi = 0;

        for (int j = 0; j < qk/2; ++j) {
            // 计算 v0 和 v1
            const int v0 = (x[i].qs[j] & 0x0F) - 8;
            const int v1 = (x[i].qs[j] >>   4) - 8;

            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]);
        }

        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    *s = sumf;
#endif
}

// 计算向量点积
void ggml_vec_dot_q4_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_1;
    const int nb = n / qk;

    // 断言 n 能被 qk 整除
    assert(n % qk == 0);

    // 将 vx 强制转换为 block_q4_1 类型
    const block_q4_1 * restrict x = vx;
    // 声明一个指向常量 block_q8_1 结构体的指针 y，并将其指向 vy
    const block_q8_1 * restrict y = vy;

    // 待办事项：添加 WASM SIMD
#if defined(__ARM_NEON)
    // 使用 NEON 指令集，定义两个包含四个单精度浮点数的向量，初始值为0.0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 初始化一个浮点数用于累加
    float summs = 0;

    // 断言 nb 为偶数，暂不处理奇数情况
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环处理每两个元素
    for (int i = 0; i < nb; i += 2) {
        // 获取指向 x 和 y 的指针
        const block_q4_1 * restrict x0 = &x[i + 0];
        const block_q4_1 * restrict x1 = &x[i + 1];
        const block_q8_1 * restrict y0 = &y[i + 0];
        const block_q8_1 * restrict y1 = &y[i + 1];

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x0->m) * y0->s + GGML_FP16_TO_FP32(x1->m) * y1->s;

        // 创建一个包含全为 0x0F 的 16 个无符号字符的向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);

        // 加载 x 的数据到向量中
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        // 将 4 位转换为 8 位
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 加载 y 的数据到向量中
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

        // 计算点积并存储到 int32x4_t 中
        const int32x4_t p_0 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_0l, v1_0l), v0_0h, v1_0h);
        const int32x4_t p_1 = ggml_vdotq_s32(ggml_vdotq_s32(vdupq_n_s32(0), v0_1l, v1_1l), v0_1h, v1_1h);

        // 使用点积结果更新 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*y1->d);
    }

    // 计算最终结果并存储到 s 中
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs;
#elif defined(__AVX2__) || defined(__AVX__)
    // 使用 AVX2 或 AVX 指令集，初始化一个包含 8 个单精度浮点数的向量，初始值为0
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数用于累加
    float summs = 0;

    // 主循环
    // 循环遍历从0到nb-1的整数
    for (int i = 0; i < nb; ++i) {
        // 将x[i]的d值从FP16转换为FP32
        const float d0 = GGML_FP16_TO_FP32(x[i].d);
        // 获取y[i]的d值
        const float d1 = y[i].d;

        // 计算summs，累加x[i]的m值乘以y[i]的s值
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 创建包含d0值的__m256向量
        const __m256 d0v = _mm256_set1_ps( d0 );
        // 创建包含d1值的__m256向量
        const __m256 d1v = _mm256_set1_ps( d1 );

        // 计算d0v和d1v的乘积
        const __m256 d0d1 = _mm256_mul_ps( d0v, d1v );

        // 从x[i].qs中加载16个字节，并将4位字段解压缩为字节，生成32个字节
        const __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从y[i].qs中加载16个字节
        const __m256i by = _mm256_loadu_si256( (const __m256i *)y[i].qs );

        // 计算bx和by的乘积，并将结果存储在xy中
        const __m256 xy = mul_sum_us8_pairs_float(bx, by);

        // 累加d0*d1*x*y
#if defined(__AVX2__)
        // 如果支持 AVX2 指令集，则使用 FMA 指令计算乘加操作
        acc = _mm256_fmadd_ps( d0d1, xy, acc );
#else
        // 如果不支持 AVX2 指令集，则使用乘法和加法指令计算乘加操作
        acc = _mm256_add_ps( _mm256_mul_ps( d0d1, xy ), acc );
#endif
    }

    // 计算累加和并加上 summs
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 使用 RISC-V 向量指令集进行计算
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

    // 将结果存入 s
    *s = sumf;
#else
    // 标量计算
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

    // 将结果存入 s
    *s = sumf;
#endif
}

// 计算向量点积
void ggml_vec_dot_q5_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk 和 nb
    const int qk = QK8_0;
    const int nb = n / qk;
    # 断言 n 能被 qk 整除
    assert(n % qk == 0);
    # 断言 qk 的值为 QK5_0

    # 声明并初始化指向 vx 的指针 x，指向 block_q5_0 类型的数据
    const block_q5_0 * restrict x = vx;
    # 声明并初始化指向 vy 的指针 y，指向 block_q8_0 类型的数据
    const block_q8_0 * restrict y = vy;
#if defined(__ARM_NEON)
    // 使用 NEON 指令集，初始化两个包含四个浮点数的向量，每个元素都是 0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 定义两个无符号整数变量
    uint32_t qh0;
    uint32_t qh1;

    // 定义两个包含四个 64 位整数的数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言 nb 为偶数，暂不处理奇数情况
    assert(nb % 2 == 0);

    // 计算两个向量的和并存储到指定地址
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__wasm_simd128__)
    // 使用 SIMD128 指令集，初始化包含四个浮点数的向量，每个元素都是 0.0f
    v128_t sumv = wasm_f32x4_splat(0.0f);

    // 定义一个无符号整数变量
    uint32_t qh;
    // 定义一个包含四个 64 位整数的数组
    uint64_t tmp[4];

    // 检查是否展开循环更好
    // 计算向量中所有元素的和并存储到指定地址
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3);
#elif defined(__AVX2__)
    // 使用 AVX2 指令集，初始化包含八个单精度浮点数的向量，所有元素为 0
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; i++) {
        /* 计算块的组合比例 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 x[i].qs 转换为字节并存储到 bx 中
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为字节并存储到 bxhi 中
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 对 bxhi 进行位与运算，并取反
        bxhi = _mm256_andnot_si256(bxhi, _mm256_set1_epi8((char)0xF0));
        // 将 bx 和 bxhi 进行或运算
        bx = _mm256_or_si256(bx, bxhi);

        // 从 y[i].qs 加载数据到 by 中
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并求和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        /* 将 q 乘以比例并累加 */
        acc = _mm256_fmadd_ps(d, q, acc);
    }

    // 将 acc 中的八个浮点数相加并存储到指定地址
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 使用 AVX 指令集，初始化包含八个单精度浮点数的向量，所有元素为 0
    __m256 acc = _mm256_setzero_ps();
    // 初始化一个包含八个字节的向量，所有元素为 0xF0
    __m128i mask = _mm_set1_epi8((char)0xF0);

    // 主循环
    // 循环遍历每个块
    for (int i = 0; i < nb; i++) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 nibbles 转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从 bits 转换为字节
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 提取 bxhi 的低位
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        // 提取 bxhi 的高位
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        // 对 bxhil 进行位非操作
        bxhil = _mm_andnot_si128(bxhil, mask);
        // 对 bxhih 进行位非操作
        bxhih = _mm_andnot_si128(bxhih, mask);
        // 提取 bx 的低位
        __m128i bxl = _mm256_castsi256_si128(bx);
        // 提取 bx 的高位
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        // 将 bxhil 与 bxl 进行或操作
        bxl = _mm_or_si128(bxl, bxhil);
        // 将 bxhih 与 bxh 进行或操作
        bxh = _mm_or_si128(bxh, bxhih);
        // 组合 bxh 和 bxl
        bx = MM256_SET_M128I(bxh, bxl);

        // 加载 y[i].qs 到 by
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并求和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加到 acc
        acc = _mm256_add_ps(_mm256_mul_ps(d, q), acc);
    }

    // 对 acc 进行横向求和并存储到 s
    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数变量 sumf，初始化为 0.0
    float sumf = 0.0;

    // 定义一个无符号整型变量 qh
    uint32_t qh;

    // 调用 __riscv_vsetvl_e8m1 函数获取向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 定义用于掩码和移位操作的临时寄存器
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vsll_vv_u32m2(__riscv_vmv_v_x_u32m2(1, vl), vt_1, vl);

    vuint32m2_t vt_3 = __riscv_vsll_vx_u32m2(vt_2, 16, vl);
    vuint32m2_t vt_4 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    }

    *s = sumf;
#else
    // scalar
    // 定义一个浮点数变量 sumf，初始化为 0.0
    float sumf = 0.0;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 定义一个无符号整型变量 qh，并从 x[i].qh 复制数据
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 初始化一个整型变量 sumi 为 0
        int sumi = 0;

        // 遍历每个块中的元素
        for (int j = 0; j < qk/2; ++j) {
            // 计算 xh_0 和 xh_1
            const uint8_t xh_0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
            const uint8_t xh_1 = ((qh & (1u << (j + 16))) >> (j + 12));

            // 计算 x0 和 x1
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            // 计算 sumi
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }

        // 计算 sumf
        sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;
    }

    // 将最终结果赋值给 s
    *s = sumf;
#endif
}

// 计算两个向量的点积
void ggml_vec_dot_q5_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk 和 nb
    const int qk = QK8_1;
    const int nb = n / qk;

    // 断言 n 能被 qk 整除
    assert(n % qk == 0);
    // 断言 qk 等于 QK5_1

    // 强制转换输入向量为特定类型
    const block_q5_1 * restrict x = vx;
    const block_q8_1 * restrict y = vy;

#if defined(__ARM_NEON)
    // 使用 NEON 指令集进行向量化计算
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    float summs0 = 0.0f;
    float summs1 = 0.0f;

    uint32_t qh0;
    uint32_t qh1;

    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言 nb 是偶数
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 将最终结果赋值给 s
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs0 + summs1;
#elif defined(__wasm_simd128__)
    // 使用 SIMD 指令集进行向量化计算
    v128_t sumv = wasm_f32x4_splat(0.0f);

    float summs = 0.0f;

    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }
    # 计算 sumv 向量中的四个元素的和，加上 summs 的值，赋给指针 s
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3) + summs;
#elif defined(__AVX2__)
    // 如果支持 AVX2 指令集

    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 初始化浮点数 summs 为 0.0
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 转换为 __m256 类型
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将 x[i].qs 转换为字节流
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为字节流，并进行位运算
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        bxhi = _mm256_and_si256(bxhi, _mm256_set1_epi8(0x10));
        bx = _mm256_or_si256(bx, bxhi);

        // 将 y[i].d 转换为 __m256 类型
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从 y[i].qs 加载数据到 __m256i 类型
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算 q，并更新累加器 acc
        const __m256 q = mul_sum_us8_pairs_float(bx, by);
        acc = _mm256_fmadd_ps(q, _mm256_mul_ps(dx, dy), acc);
    }

    // 计算最终结果并赋值给 s
    *s = hsum_float_8(acc) + summs;
#elif defined(__AVX__)
    // 如果支持 AVX 指令集

    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();
    // 初始化掩码为 0x10
    __m128i mask = _mm_set1_epi8(0x10);

    // 初始化浮点数 summs 为 0.0
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 转换为 __m256 类型
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将 x[i].qs 转换为字节流
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为字节流，并进行位运算
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

        // 将 y[i].d 转换为 __m256 类型
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从 y[i].qs 加载数据到 __m256i 类型
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算 q，并更新累加器 acc
        const __m256 q = mul_sum_us8_pairs_float(bx, by);
        acc = _mm256_add_ps(_mm256_mul_ps(q, _mm256_mul_ps(dx, dy)), acc);
    }

    // 计算最终结果并赋值给 s
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数变量 sumf，初始化为 0.0
    float sumf = 0.0;

    // 定义一个无符号整数变量 qh
    uint32_t qh;

    // 调用 __riscv_vsetvl_e8m1 函数计算向量长度 vl
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 定义两个临时向量寄存器 vt_1 和 vt_2，用于位移操作
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);
    // 遍历循环，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 将 x[i].qh 的内容复制到 qh 中，长度为 uint32_t 的大小
        memcpy(&qh, x[i].qh, sizeof(uint32_t));

        // 加载 qh 到向量 vqh 中
        vuint32m2_t vqh = __riscv_vmv_v_x_u32m2(qh, vl);

        // 计算 ((qh >> (j +  0)) << 4) & 0x10
        vuint32m2_t xhr_0 = __riscv_vsrl_vv_u32m2(vqh, vt_1, vl);
        vuint32m2_t xhl_0 = __riscv_vsll_vx_u32m2(xhr_0, 4, vl);
        vuint32m2_t xha_0 = __riscv_vand_vx_u32m2(xhl_0, 0x10, vl);

        // 计算 ((qh >> (j + 12))     ) & 0x10
        vuint32m2_t xhr_1 = __riscv_vsrl_vv_u32m2(vqh, vt_2, vl);
        vuint32m2_t xha_1 = __riscv_vand_vx_u32m2(xhr_1, 0x10, vl);

        // 缩小数据类型
        vuint16m1_t xhc_0 = __riscv_vncvt_x_x_w_u16m1(xha_0, vl);
        vuint8mf2_t xh_0 = __riscv_vncvt_x_x_w_u8mf2(xhc_0, vl);

        vuint16m1_t xhc_1 = __riscv_vncvt_x_x_w_u16m1(xha_1, vl);
        vuint8mf2_t xh_1 = __riscv_vncvt_x_x_w_u8mf2(xhc_1, vl);

        // 加载数据
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        vuint8mf2_t x_at = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        vuint8mf2_t x_lt = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        vuint8mf2_t x_a = __riscv_vor_vv_u8mf2(x_at, xh_0, vl);
        vuint8mf2_t x_l = __riscv_vor_vv_u8mf2(x_lt, xh_1, vl);

        vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        // 将 vs2 转换为 int 类型
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        // 计算 sumf
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    // 将 sumf 赋值给 s
    *s = sumf;
#else
    // 如果不是向量，则使用标量计算
    float sumf = 0.0; // 初始化浮点数求和变量为0

    for (int i = 0; i < nb; i++) { // 遍历每个块
        uint32_t qh; // 定义32位无符号整数变量
        memcpy(&qh, x[i].qh, sizeof(qh)); // 将x[i].qh的内容复制到qh变量中

        int sumi = 0; // 初始化整数求和变量为0

        for (int j = 0; j < qk/2; ++j) { // 遍历每个块的一半
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10; // 计算xh_0
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10; // 计算xh_1

            const int32_t x0 = (x[i].qs[j] & 0xF) | xh_0; // 计算x0
            const int32_t x1 = (x[i].qs[j] >>  4) | xh_1; // 计算x1

            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]); // 更新sumi
        }

        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s; // 更新sumf
    }

    *s = sumf; // 将最终结果赋值给输出指针
#endif
}

void ggml_vec_dot_q8_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_0; // 定义qk为QK8_0
    const int nb = n / qk; // 计算块数

    assert(n % qk == 0); // 断言n能被qk整除

    const block_q8_0 * restrict x = vx; // 定义输入向量x
    const block_q8_0 * restrict y = vy; // 定义输入向量y

#if defined(__ARM_NEON)
    // 使用NEON指令集，初始化两个浮点数向量为0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    assert(nb % 2 == 0); // 断言块数为偶数，处理奇数块的情况
    // 循环遍历数组，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 获取指向 x[i] 和 x[i+1] 的指针
        const block_q8_0 * restrict x0 = &x[i + 0];
        const block_q8_0 * restrict x1 = &x[i + 1];
        // 获取指向 y[i] 和 y[i+1] 的指针
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 加载 x0 的数据到寄存器
        const int8x16_t x0_0 = vld1q_s8(x0->qs);
        const int8x16_t x0_1 = vld1q_s8(x0->qs + 16);
        // 加载 x1 的数据到寄存器
        const int8x16_t x1_0 = vld1q_s8(x1->qs);
        const int8x16_t x1_1 = vld1q_s8(x1->qs + 16);

        // 加载 y0 的数据到寄存器
        const int8x16_t y0_0 = vld1q_s8(y0->qs);
        const int8x16_t y0_1 = vld1q_s8(y0->qs + 16);
        // 加载 y1 的数据到寄存器
        const int8x16_t y1_0 = vld1q_s8(y1->qs);
        const int8x16_t y1_1 = vld1q_s8(y1->qs + 16);

        // 计算 sumv0
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        ggml_vdotq_s32(vdupq_n_s32(0), x0_0, y0_0),
                        ggml_vdotq_s32(vdupq_n_s32(0), x0_1, y0_1))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));

        // 计算 sumv1
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        ggml_vdotq_s32(vdupq_n_s32(0), x1_0, y1_0),
                        ggml_vdotq_s32(vdupq_n_s32(0), x1_1, y1_1))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
    }

    // 将 sumv0 和 sumv1 的结果相加，存储到指定地址
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果支持 AVX2 或 AVX 指令集，则执行以下代码块

    // 初始化累加器为零
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
    // 如果支持 RISC-V 指令集，则执行以下代码块
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
    // 如果不支持 AVX2、AVX 或 RISC-V 指令集，则执行以下代码块
    // 标量计算
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
    // 如果是 ARM 平台且支持 NEON 指令集，则执行以下代码块
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    // 创建一个包含16个相同值0xF的向量
    const uint8x16_t m4 = vdupq_n_u8(0xF);

    // 创建一个包含4个相同值0的向量
    const int32x4_t vzero = vdupq_n_s32(0);

    // 定义一个包含两个int8x16_t类型的结构体变量
    ggml_int8x16x2_t q2bytes;
    // 创建一个长度为16的辅助数组
    uint8_t aux[16];

    // 初始化变量sum为0
    float sum = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {
        // 计算d为y[i].d与x[i].d的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算dmin为-y[i].d与x[i].dmin的乘积
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 获取指向x[i].qs的指针
        const uint8_t * restrict q2 = x[i].qs;
        // 获取指向y[i].qs的指针
        const int8_t  * restrict q8 = y[i].qs;
        // 获取指向x[i].scales的指针
        const uint8_t * restrict sc = x[i].scales;

        // 从指针sc处加载16个字节到mins_and_scales向量
        const uint8x16_t mins_and_scales = vld1q_u8(sc);
        // 通过与操作获取scales向量
        const uint8x16_t scales = vandq_u8(mins_and_scales, m4);
        // 将scales向量存储到辅助数组aux中
        vst1q_u8(aux, scales);

        // 右移4位获取mins向量
        const uint8x16_t mins = vshrq_n_u8(mins_and_scales, 4);
        // 从y[i].bsums加载16个int16_t到q8sums向量
        const ggml_int16x8x2_t q8sums = ggml_vld1q_s16_x2(y[i].bsums);
        // 将mins向量拆分成两个int16x8_t类型的向量mins16
        const ggml_int16x8x2_t mins16 = {{vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(mins))), vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(mins)))}};
        // 计算s0和s1
        const int32x4_t s0 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[0]), vget_low_s16 (q8sums.val[0])),
                                       vmull_s16(vget_high_s16(mins16.val[0]), vget_high_s16(q8sums.val[0])));
        const int32x4_t s1 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[1]), vget_low_s16 (q8sums.val[1])),
                                       vmull_s16(vget_high_s16(mins16.val[1]), vget_high_s16(q8sums.val[1])));
        // 更新sum的值
        sum += dmin * vaddvq_s32(vaddq_s32(s0, s1));

        // 初始化isum和is的值为0
        int isum = 0;
        int is = 0;
// 定义宏，用于将两个向量进行点乘，并将结果累加到isum中，同时乘以aux数组中对应位置的值
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * aux[is+(index)];\
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * aux[is+1+(index)];

// 定义宏，用于对向量进行位移、点乘和累加操作
#define SHIFT_MULTIPLY_ACCUM_WITH_SCALE(shift, index)\
        q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;\
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[0], (shift)), m3));\
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[1], (shift)), m3));\
        MULTIPLY_ACCUM_WITH_SCALE((index));

// 循环处理QK_K/128次
        for (int j = 0; j < QK_K/128; ++j) {
            // 从q2中加载两个128位的向量
            const ggml_uint8x16x2_t q2bits = ggml_vld1q_u8_x2(q2); q2 += 32;

            // 从q8中加载两个128位的向量
            ggml_int8x16x2_t q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 对q2bits中的值进行位与运算
            q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[0], m3));
            q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[1], m3));

            // 对第一个向量进行点乘和累加操作
            MULTIPLY_ACCUM_WITH_SCALE(0);

            // 对第二个向量进行位移、点乘和累加操作
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(2, 2);
            // 对第三个向量进行位移、点乘和累加操作
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(4, 4);
            // 对第四个向量进行位移、点乘和累加操作
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(6, 6);

            // 更新is的值
            is += 8;
        }

        // 将d乘以isum的结果加到sum中

        sum += d * isum;
    }

    // 将sum的值赋给*s

#elif defined __AVX2__

    // 定义常量向量m3和m4
    const __m256i m3 = _mm256_set1_epi8(3);
    const __m128i m4 = _mm_set1_epi8(0xF);

    // 初始化256位浮点累加器acc为0

    }

    // 将acc的值通过hsum_float_8函数求和后赋给*s

#elif defined __AVX__

    // 定义常量向量m3、m4和m2
    const __m128i m3 = _mm_set1_epi8(0x3);
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(0x2);

    // 初始化256位浮点累加器acc为0

    }

    // 将acc的值通过hsum_float_8函数求和后赋给*s

#elif defined __riscv_v_intrinsic

    // 初始化sumf为0
    float sumf = 0;
    // 初始化长度为32的uint8_t数组temp_01

    }
    # 将指针 *s 指向 sumf 函数
    *s = sumf;
#else
// 如果不是 ARM 平台，则执行以下代码块

    // 初始化浮点数 sumf 为 0
    float sumf = 0;

    // 遍历 n 个元素
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 的 qs 指针
        const uint8_t * q2 = x[i].qs;
        // 获取 y[i] 的 qs 指针
        const int8_t * q8 = y[i].qs;
        // 获取 x[i] 的 scales 指针
        const uint8_t * sc = x[i].scales;

        // 初始化 summs 为 0
        int summs = 0;
        // 遍历 16 个元素
        for (int j = 0; j < 16; ++j) {
            // 计算 summs
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        // 计算 dall 和 dmin
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 初始化 isum, is 和 d
        int isum = 0;
        int is = 0;
        int d;
        // 遍历 QK_K/128 次
        for (int k = 0; k < QK_K/128; ++k) {
            int shift = 0;
            // 遍历 4 次
            for (int j = 0; j < 4; ++j) {
                d = sc[is++] & 0xF;
                int isuml = 0;
                // 计算 isuml
                for (int l =  0; l < 16; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3);
                isum += d * isuml;
                d = sc[is++] & 0xF;
                isuml = 0;
                // 计算 isuml
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
    // 将 sumf 赋值给指针 s
    *s = sumf;
#endif
}

#else
// 如果不是 ARM 平台，则执行以下代码块

void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    // 将 vx 强制转换为 block_q2_K 指针 x
    const block_q2_K * restrict x = vx;
    // 将 vy 强制转换为 block_q8_K 指针 y
    const block_q8_K * restrict y = vy;

    // 计算 nb
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 初始化 m3 为 0x3 的 uint8x16_t 常量
    const uint8x16_t m3 = vdupq_n_u8(0x3);

    // 初始化 vzero 为 0 的 int32x4_t 常量
    const int32x4_t vzero = vdupq_n_s32(0);

    // 初始化 q2bytes 为 ggml_int8x16x4_t 结构体

    uint32_t aux32[2];
    const uint8_t * scales = (const uint8_t *)aux32;

    // 初始化浮点数 sum 为 0
    float sum = 0;
    // 遍历循环，i 从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {

        // 计算 d 值，y[i].d 乘以 x[i].d
        const float d = y[i].d * (float)x[i].d;
        // 计算 dmin 值，-y[i].d 乘以 x[i].dmin
        const float dmin = -y[i].d * (float)x[i].dmin;

        // 获取指向 x[i].qs 的指针
        const uint8_t * restrict q2 = x[i].qs;
        // 获取指向 y[i].qs 的指针
        const int8_t  * restrict q8 = y[i].qs;
        // 获取指向 x[i].scales 的指针，并转换为 uint32_t 类型
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;

        // 将 sc[0] 的低 4 位存入 aux32[0]，高 4 位存入 aux32[1]
        aux32[0] = sc[0] & 0x0f0f0f0f;
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算 sum 值，加上一系列乘积结果
        sum += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);

        // 初始化 isum1 和 isum2 为 0
        int isum1 = 0, isum2 = 0;

        // 加载 q2 指向的 16 个字节到 uint8x16_t 类型的变量 q2bits
        const uint8x16_t q2bits = vld1q_u8(q2);

        // 加载 q8 指向的 4 个 int8_t 类型数组到 ggml_int8x16x4_t 类型变量 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 q2bits 进行位运算，存入 q2bytes 的 4 个元素中
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits, m3));
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 2), m3));
        q2bytes.val[2] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 4), m3));
        q2bytes.val[3] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 6), m3));

        // 计算 isum1 和 isum2 的值，使用 vdotq_s32 函数和 scales 数组
        isum1 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * scales[0];
        isum2 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * scales[1];
        isum1 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[2], q8bytes.val[2])) * scales[2];
        isum2 += vaddvq_s32(ggml_vdotq_s32(vzero, q2bytes.val[3], q8bytes.val[3])) * scales[3];

        // 计算 sum 值，加上 d 乘以 isum1 和 isum2 的和
        sum += d * (isum1 + isum2);
    }

    // 将 sum 的值赋给指针 s 所指向的变量
    *s = sum;
#elif defined __AVX2__
    # 如果定义了 AVX2，执行以下代码块

    const __m256i m3 = _mm256_set1_epi8(3);
    # 创建一个包含全部元素为3的256位整数向量

    __m256 acc = _mm256_setzero_ps();
    # 创建一个256位浮点数向量，所有元素初始化为0

    uint32_t ud, um;
    # 声明两个32位无符号整数变量 ud 和 um
    const uint8_t * restrict db = (const uint8_t *)&ud;
    # 声明一个指向 ud 的无符号8位整数指针 db
    const uint8_t * restrict mb = (const uint8_t *)&um;
    # 声明一个指向 um 的无符号8位整数指针 mb

    float summs = 0;
    # 声明一个浮点数变量 summs，初始化为0

    // TODO: optimize this
    # 待优化的部分，暂时未实现具体代码
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d 和 dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 获取指向 x[i].qs 和 y[i].qs 的指针
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 获取指向 x[i].scales 的指针，并提取其中的值
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        ud = (sc[0] >> 0) & 0x0f0f0f0f;
        um = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算 smin，并更新 summs
        int32_t smin = mb[0] * y[i].bsums[0] + mb[1] * y[i].bsums[1] + mb[2] * y[i].bsums[2] + mb[3] * y[i].bsums[3];
        summs += dmin * smin;

        // 加载 q2 中的数据到寄存器
        const __m128i q2bits = _mm_loadu_si128((const __m128i*)q2);
        const __m256i q2_0 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 2), q2bits), m3);
        const __m256i q2_1 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 6), _mm_srli_epi16(q2bits, 4)), m3);

        // 加载 q8 中的数据到寄存器
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算 p0 和 p1
        const __m256i p0 = _mm256_maddubs_epi16(q2_0, q8_0);
        const __m256i p1 = _mm256_maddubs_epi16(q2_1, q8_1);

        // 转换 p0 和 p1 的数据类型
        const __m256i p_0 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 0));
        const __m256i p_1 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 1));
        const __m256i p_2 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 0));
        const __m256i p_3 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 1));

        // 计算累加器 acc
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[0]), _mm256_cvtepi32_ps(p_0), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[1]), _mm256_cvtepi32_ps(p_1), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[2]), _mm256_cvtepi32_ps(p_2), acc);
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[3]), _mm256_cvtepi32_ps(p_3), acc);
    }

    // 计算最终结果并存储到 s
    *s = hsum_float_8(acc) + summs;
#elif defined __AVX__
    // 如果定义了 AVX，使用 AVX 指令集

    // 创建一个包含 3 的 __m128i 类型的向量
    const __m128i m3 = _mm_set1_epi8(3);

    // 创建一个全零的 __m256 类型的向量
    __m256 acc = _mm256_setzero_ps();

    // 定义变量 ud 和 um，分别表示 uint32_t 类型的数据
    uint32_t ud, um;
    // 将 ud 和 um 强制转换为 uint8_t 类型的指针，并限制指针的访问范围
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;

    // 定义一个浮点数 summs，并初始化为 0
    float summs = 0;

    // TODO: optimize this
    // 待优化的部分

    }

    // 将累加和 acc 的每个元素相加，并加上 summs，赋值给指针 s 指向的变量
    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic
    // 如果定义了 riscv_v_intrinsic，使用 riscv_v_intrinsic 指令集

    // 定义一个包含 2 个 uint32_t 类型数据的数组 aux32
    uint32_t aux32[2];
    // 将 aux32 强制转换为 uint8_t 类型的指针，并赋值给 scales
    const uint8_t * scales = (const uint8_t *)aux32;

    // 定义一个浮点数 sumf，并初始化为 0
    float sumf = 0;

    }

    // 将 sumf 赋值给指针 s 指向的变量
    *s = sumf;

#else
    // 如果未定义 AVX 和 riscv_v_intrinsic，执行以下代码块

    // 定义一个浮点数 sumf，并初始化为 0
    float sumf = 0;

    // 定义一个包含 4 个 int 类型数据的数组 isum，并初始化为 0
    int isum[4];

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 和 y[i] 的指针
        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;

        // 定义一个 int 类型的 summs，并初始化为 0
        int summs = 0;
        // 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 summs
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        // 计算 dall 和 dmin
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 将 isum[0]、isum[1]、isum[2]、isum[3] 初始化为 0
        isum[0] = isum[1] = isum[2] = isum[3] = 0;
        // 遍历 16 次
        for (int l =  0; l < 16; ++l) {
            // 计算 isum
            isum[0] += q8[l+ 0] * ((q2[l] >> 0) & 3);
            isum[1] += q8[l+16] * ((q2[l] >> 2) & 3);
            isum[2] += q8[l+32] * ((q2[l] >> 4) & 3);
            isum[3] += q8[l+48] * ((q2[l] >> 6) & 3);
        }
        // 遍历 4 次
        for (int l = 0; l < 4; ++l) {
            // 计算 isum
            isum[l] *= (sc[l] & 0xF);
        }
        // 计算 sumf
        sumf += dall * (isum[0] + isum[1] + isum[2] + isum[3]) - dmin * summs;
    }
    // 将 sumf 赋值给指针 s 指向的变量
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 如果 QK_K 等于 256，执行以下代码块
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 定义两个掩码
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 获取 x 和 y 的指针
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 nb
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 如果定义了 ARM_NEON，使用 ARM_NEON 指令集

    // 定义两个数组
    uint32_t aux[3];
    uint32_t utmp[4];

    // 创建一个包含 16 个 3 的 uint8x16_t 类型的向量
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 创建一个包含 4 个 0 的 int32x4_t 类型的向量
    const int32x4_t  vzero = vdupq_n_s32(0);
    // 创建一个包含16个8位无符号整数1的向量
    const uint8x16_t m0 = vdupq_n_u8(1);
    // 将向量m0中的每个元素左移1位，得到新的向量m1
    const uint8x16_t m1 = vshlq_n_u8(m0, 1);
    // 将向量m0中的每个元素左移2位，得到新的向量m2
    const uint8x16_t m2 = vshlq_n_u8(m0, 2);
    // 将向量m0中的每个元素左移3位，得到新的向量m3
    const uint8x16_t m3 = vshlq_n_u8(m0, 3);
    // 创建一个包含32的8位有符号整数的常量
    const int8_t m32 = 32;

    // 创建一个包含4个8x16整数向量的结构体
    ggml_int8x16x4_t q3bytes;

    // 初始化一个浮点数sum为0
    float sum = 0;

    }

    // 将计算得到的sum赋值给指针s指向的变量
    *s = sum;
#elif defined __AVX2__
    // 如果定义了 AVX2，使用 AVX2 指令集

    // 创建一个包含 3 的 256 位整数向量
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含 1 的 256 位整数向量
    const __m256i mone = _mm256_set1_epi8(1);
    // 创建一个包含 32 的 128 位整数向量
    const __m128i m32 = _mm_set1_epi8(32);

    // 创建一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 创建一个包含 3 个 32 位无符号整数的数组
    uint32_t aux[3];

    }

    // 计算浮点数向量 acc 的和，存储到 s 中

#elif defined __AVX__
    // 如果定义了 AVX，使用 AVX 指令集

    // 创建一个包含 3 的 128 位整数向量
    const __m128i m3 = _mm_set1_epi8(3);
    // 创建一个包含 1 的 128 位整数向量
    const __m128i mone = _mm_set1_epi8(1);
    // 创建一个包含 32 的 128 位整数向量
    const __m128i m32 = _mm_set1_epi8(32);
    // 创建一个包含 2 的 128 位整数向量
    const __m128i m2 = _mm_set1_epi8(2);

    // 创建一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 指向无符号整数数组的指针
    const uint32_t *aux;

    }

    // 计算浮点数向量 acc 的和，存储到 s 中

#elif defined __riscv_v_intrinsic
    // 如果定义了 riscv_v_intrinsic，使用 riscv_v_intrinsic 指令集

    // 创建一个包含 3 个 32 位无符号整数的数组
    uint32_t aux[3];
    // 创建一个包含 4 个 32 位无符号整数的数组
    uint32_t utmp[4];

    // 初始化浮点数和为 0
    float sumf = 0;
    }

    // 将浮点数和存储到 s 中

#else
    // 如果未定义特定指令集，使用标量版本

    // 这个函数的编写方式使得编译器能够尝试对大部分代码进行向量化
    // 使用 -Ofast，GCC 和 clang 能够生成与手动向量化版本相差不大的代码
    // 我尝试的其他版本运行速度至少慢 4 倍
    // 理想情况下，我们希望只需编写一次代码，编译器就能自动产生最佳的机器指令集，而不是我们手动编写 AVX、ARM_NEON 等向量化版本

    // 创建一个包含 QK_K 个 8 位整数的数组
    int8_t  aux8[QK_K];
    // 创建一个包含 8 个 16 位整数的数组
    int16_t aux16[8];
    // 创建一个包含 8 个浮点数的数组
    float   sums [8];
    // 创建一个包含 8 个 32 位整数的数组
    int32_t aux32[8];
    // 将 sums 数组初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 创建一个包含 4 个 32 位无符号整数的数组
    uint32_t auxs[4];
    // 将 auxs 转换为 int8_t 类型的指针
    const int8_t * scales = (const int8_t*)auxs;

    // 初始化浮点数和为 0
    float sumf = 0;
    // 遍历 nb 次，处理每个 x[i] 和 y[i] 对应的数据
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 的 qs 和 hmask，以及 y[i] 的 qs
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        const int8_t * restrict q8 = y[i].qs;
        
        // 将 aux32 数组清零
        memset(aux32, 0, 8*sizeof(int32_t));
        int8_t * restrict a = aux8;
        uint8_t m = 1;
        
        // 处理 QK_K 次数据
        for (int j = 0; j < QK_K; j += 128) {
            // 处理每个 32 位数据
            for (int l = 0; l < 32; ++l) a[l] = q3[l] & 3;
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            a += 32; m <<= 1;
            // 处理每个 32 位数据
            for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 2) & 3;
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            a += 32; m <<= 1;
            // 处理每个 32 位数据
            for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 4) & 3;
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            a += 32; m <<= 1;
            // 处理每个 32 位数据
            for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 6) & 3;
            for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
            a += 32; m <<= 1;
            q3 += 32;
        }
        a = aux8;
        
        // 复制 x[i] 的 scales 到 auxs
        memcpy(auxs, x[i].scales, 12);
        uint32_t tmp = auxs[2];
        // 重新排列 auxs 数组中的数据
        auxs[2] = ((auxs[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        auxs[3] = ((auxs[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        auxs[0] = (auxs[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        auxs[1] = (auxs[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);
        
        // 处理 QK_K/16 次数据
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 aux16 和 aux32
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += (scales[j] - 32) * aux16[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += (scales[j] - 32) * aux16[l];
            q8 += 8; a += 8;
        }
        
        // 计算 d，并更新 sums 数组
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    
    // 计算 sumf，并更新 s 指向的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    *s = sumf;
#endif
}

#else

// 计算两个向量的点积，其中向量长度为 n，存储在 s 中，向量 vx 和 vy 分别为输入向量
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度为 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将输入向量转换为特定类型的指针
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度为 QK_K 时的块数
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 创建一个全为 0 的 int32x4_t 类型的向量
    const int32x4_t vzero = vdupq_n_s32(0);

    // 创建一个全为 0x3 的 uint8x16_t 类型的向量
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 创建一个全为 4 的 uint8x16_t 类型的向量
    const uint8x16_t mh  = vdupq_n_u8(4);

    // 定义一个包含 4 个 int8x16_t 类型的结构体
    ggml_int8x16x4_t q3bytes;

    // 创建一个辅助数组 aux16，用于存储 int8_t 类型的数据
    uint16_t aux16[2];
    int8_t * scales = (int8_t *)aux16;

    // 初始化点积的结果为 0
    float sum = 0;
    // 遍历 nb 次循环
    for (int i = 0; i < nb; ++i) {

        // 定义一个包含四个 uint8x16x4_t 结构的变量 q3h
        ggml_uint8x16x4_t q3h;

        // 从 x[i].hmask 中加载一个 uint8x8_t 类型的变量 hbits
        const uint8x8_t  hbits    = vld1_u8(x[i].hmask);
        // 从 x[i].qs 中加载一个 uint8x16_t 类型的变量 q3bits
        const uint8x16_t q3bits   = vld1q_u8(x[i].qs);
        // 从 y[i].qs 中加载一个 ggml_int8x16x4_t 类型的变量 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(y[i].qs);

        // 从 x[i].scales 中加载一个 uint16_t 类型的变量 a
        const uint16_t a = *(const uint16_t *)x[i].scales;
        // 将 a 的低 8 位和高 8 位分别存储到 aux16 数组中
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        // 遍历四次循环，对 scales 数组中的值减去 8
        for (int j = 0; j < 4; ++j) scales[j] -= 8;

        // 计算 isum 的值
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        // 计算 d 的值
        const float d = y[i].d * (float)x[i].d;

        // 对 hbits 进行位移和位与操作，存储到 q3h 中
        const uint8x16_t htmp = vcombine_u8(hbits, vshr_n_u8(hbits, 1));
        q3h.val[0] = vandq_u8(mh, vshlq_n_u8(htmp, 2));
        q3h.val[1] = vandq_u8(mh, htmp);
        q3h.val[2] = vandq_u8(mh, vshrq_n_u8(htmp, 2));
        q3h.val[3] = vandq_u8(mh, vshrq_n_u8(htmp, 4));

        // 对 q3bits 和 q3h 进行位与和位或操作，存储到 q3bytes 中
        q3bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q3bits, m3b),                q3h.val[0]));
        q3bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 2), m3b), q3h.val[1]));
        q3bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 4), m3b), q3h.val[2]));
        q3bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q3bits, 6),                q3h.val[3]));

        // 计算 isum 的值
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[0], q8bytes.val[0])) * scales[0];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[1], q8bytes.val[1])) * scales[2];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[2], q8bytes.val[2])) * scales[1];
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q3bytes.val[3], q8bytes.val[3])) * scales[3];

        // 计算 sum 的值
        sum += d * isum;

    }

    // 将 sum 的值存储到 s 中
    *s = sum;
#elif defined __AVX2__
    # 如果定义了 AVX2，则执行以下代码块

    const __m256i m3 = _mm256_set1_epi8(3);
    # 创建一个包含全部元素为3的256位整数向量
    const __m256i m1 = _mm256_set1_epi8(1);
    # 创建一个包含全部元素为1的256位整数向量

    __m256 acc = _mm256_setzero_ps();
    # 创建一个256位浮点数向量，所有元素初始化为0

    uint64_t aux64;
    # 声明一个64位无符号整数变量

    uint16_t aux16[2];
    # 声明一个包含2个元素的16位无符号整数数组
    const int8_t * aux8 = (const int8_t *)aux16;
    # 将16位无符号整数数组转换为8位有符号整数指针

    }

    *s = hsum_float_8(acc);
    # 调用函数hsum_float_8计算256位浮点数向量acc的和，并将结果存储在指针s指向的位置

#elif defined __AVX__
    # 如果定义了 AVX，则执行以下代码块

    const __m128i m3 = _mm_set1_epi8(3);
    # 创建一个包含全部元素为3的128位整数向量
    const __m128i m1 = _mm_set1_epi8(1);
    # 创建一个包含全部元素为1的128位整数向量

    __m256 acc = _mm256_setzero_ps();
    # 创建一个256位浮点数向量，所有元素初始化为0

    uint64_t aux64;
    # 声明一个64位无符号整数变量

    uint16_t aux16[2];
    # 声明一个包含2个元素的16位无符号整数数组
    const int8_t * aux8 = (const int8_t *)aux16;
    # 将16位无符号整数数组转换为8位有符号整数指针

    }

    *s = hsum_float_8(acc);
    # 调用函数hsum_float_8计算256位浮点数向量acc的和，并将结果存储在指针s指向的位置

#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则执行以下代码块

    uint16_t aux16[2];
    # 声明一个包含2个元素的16位无符号整数数组
    int8_t * scales = (int8_t *)aux16;
    # 将16位无符号整数数组转换为8位有符号整数指针

    float sumf = 0;
    # 声明一个浮点数变量sumf，并初始化为0

    }

    *s = sumf;
    # 将sumf的值存储在指针s指向的位置

#else
    # 如果以上条件都不满足，则执行以下代码块

    int8_t  aux8[QK_K];
    # 声明一个长度为QK_K的8位有符号整数数组
    int16_t aux16[8];
    # 声明一个包含8个元素的16位有符号整数数组
    float   sums [8];
    # 声明一个包含8个元素的浮点数数组
    int32_t aux32[8];
    # 声明一个包含8个元素的32位有符号整数数组
    int32_t scales[4];
    # 声明一个包含4个元素的32位有符号整数数组
    memset(sums, 0, 8*sizeof(float));
    # 将浮点数数组sums的所有元素初始化为0

    float sumf = 0;
    # 声明一个浮点数变量sumf，并初始化为0
    // 遍历输入数组 x 的元素
    for (int i = 0; i < nb; ++i) {
        // 获取当前元素的 qs、hmask 和 scales
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        const int8_t * restrict q8 = y[i].qs;
        int8_t * restrict a = aux8;
        
        // 遍历处理每个元素的 8 个子元素
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

        // 初始化 aux32 数组
        memset(aux32, 0, 8*sizeof(int32_t));
        
        // 遍历处理每个元素的 8 个子元素
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算 aux16 数组中的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] += q8[l] * a[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux32[l] += scales[j] * aux16[l];
        }
        
        // 计算 d 值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        
        // 更新 sums 数组中的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    
    // 计算 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    
    // 将 sumf 的值赋给 s
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 计算两个向量的点积，其中向量长度为 QK_K
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将输入指针转换为特定类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量中块的数量
    const int nb = n / QK_K;

    // 定义用于按位与操作的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    // 定义用于存储临时结果的数组
    uint32_t utmp[4];

#ifdef __ARM_NEON
    // 定义 ARM NEON 指令需要使用的变量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const int32x4_t mzero = vdupq_n_s32(0);

    ggml_int8x16x2_t q4bytes;
    ggml_int8x16x2_t q8bytes;

    float sumf = 0;

    // 计算向量点积

    *s = sumf;

#elif defined __AVX2__
    // 定义 AVX2 指令需要使用的变量
    const __m256i m4 = _mm256_set1_epi8(0xF);

    __m256 acc = _mm256_setzero_ps();
    // 计算向量点积

    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

#elif defined __AVX__
    // 定义 AVX 指令需要使用的变量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(0x2);

    __m256 acc = _mm256_setzero_ps();
    // 计算向量点积

    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

#elif defined __riscv_v_intrinsic
    // 定义 RISC-V 指令需要使用的变量
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    float sumf = 0;

    // 计算向量点积

    *s = sumf;

#else
    // 定义默认情况下需要使用的变量
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    memset(sums, 0, 8*sizeof(float));

    float sumf = 0;
    // 遍历 nb 次，处理每个元素
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 的 qs 指针
        const uint8_t * restrict q4 = x[i].qs;
        // 获取 y[i] 的 qs 指针
        const int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组清零
        memset(aux32, 0, 8*sizeof(int32_t));
        // 获取 aux8 数组指针
        int8_t * restrict a = aux8;
        
        // 遍历 QK_K/64 次
        for (int j = 0; j < QK_K/64; ++j) {
            // 将 q4 中的数据按位拆分到 a 中
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l] & 0xF);
            a += 32;
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l]  >> 4);
            a += 32; q4 += 32;
        }
        
        // 复制 x[i] 的 scales 到 utmp
        memcpy(utmp, x[i].scales, 12);
        // 对 utmp 进行位运算
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;

        // 初始化 sumi 为 0
        int sumi = 0;
        // 计算 sumi
        for (int j = 0; j < QK_K/16; ++j) sumi += y[i].bsums[j] * mins[j/2];
        a = aux8;
        int is = 0;
        
        // 遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 获取当前 scale
            int32_t scale = scales[is++];
            // 计算 aux32
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
        }
        
        // 计算 d
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 更新 sums
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        // 计算 dmin
        const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;
        // 更新 sumf
        sumf -= dmin * sumi;
    }
    
    // 计算 sums 的总和
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将最终结果赋值给 s
    *s = sumf;
#else
// 如果不是 ARM_NEON 架构，则执行以下代码块
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
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

    // 定义 q4bytes、q8bytes 和 scales
    ggml_int8x16x2_t q4bytes;
    ggml_int8x16x4_t q8bytes;
    uint16_t aux16[2];
    const uint8_t * restrict scales = (const uint8_t *)aux16;

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 获取 q4 和 q8 的指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 获取 a 和计算 summi
        const uint16_t * restrict a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;
        const int32_t summi = scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]);
        sum_mins += y[i].d * (float)x[i].d[1] * summi;

        // 计算 d
        const float d = y[i].d * (float)x[i].d[0];

        // 加载 q4bits 和 q8bytes
        const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4);
        q8bytes = ggml_vld1q_s8_x4(q8);

        // 处理 q4bits 和 q8bytes，计算 sumi1 和 sumi2
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
        const int32x4_t p1 = ggml_vdotq_s32(ggml_vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
        const int32_t sumi1 = vaddvq_s32(p1) * scales[0];

        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
        const int32x4_t p2 = ggml_vdotq_s32(ggml_vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[2]), q4bytes.val[1], q8bytes.val[3]);
        const int32_t sumi2 = vaddvq_s32(p2) * scales[1];

        // 更新 sumf
        sumf += d * (sumi1 + sumi2);
    }

    // 计算最终结果并赋值给 s
    *s = sumf - sum_mins;

#elif defined __AVX2__
    // 创建一个包含所有元素为0xF的256位整数寄存器
    const __m256i m4 = _mm256_set1_epi8(0xF);

    // 创建一个所有元素为0的256位单精度浮点数寄存器
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数变量 summs 为0
    float summs = 0;

    // 创建一个包含两个16位整数的数组 aux16
    uint16_t aux16[2];
    // 将 aux16 强制转换为 uint8_t 类型指针，并赋值给 scales
    const uint8_t * scales = (const uint8_t *)aux16;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d 和 m 的值
        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        // 创建一个所有元素为 d 的256位单精度浮点数寄存器
        const __m256 vd = _mm256_set1_ps(d);

        // 将 x[i].scales 转换为 uint16_t 类型指针 a
        const uint16_t * a = (const uint16_t *)x[i].scales;
        // 将 a[0] 的值按位与 0x0f0f 和右移4位后按位与 0x0f0f，存入 aux16 数组
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 更新 summs 的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 将 x[i].qs 和 y[i].qs 转换为 uint8_t 和 int8_t 类型指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 加载 q4 和 q8 的值到寄存器
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        const __m256i q4l = _mm256_and_si256(q4bits, m4);
        const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

        const __m256i q8l = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8h = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算 p16l 和 p16h 的值
        const __m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
        const __m256i p16h = _mm256_maddubs_epi16(q4h, q8h);

        // 计算 p32l 和 p32h 的值，并更新 acc 的值
        const __m256i p32l = _mm256_madd_epi16(_mm256_set1_epi16(scales[0]), p16l);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32l), acc);

        const __m256i p32h = _mm256_madd_epi16(_mm256_set1_epi16(scales[1]), p16h);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32h), acc);

    }

    // 计算最终结果并存入指针 s 指向的地址
    *s = hsum_float_8(acc) - summs;
#elif defined __AVX__
    // 如果定义了 AVX，则执行以下代码块

    // 创建一个包含相同值的 __m128i 类型变量 m4
    const __m128i m4 = _mm_set1_epi8(0xF);

    // 创建一个初始值为零的 __m256 类型变量 acc
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数 summs 为 0
    float summs = 0;

    // 创建一个长度为 2 的 uint16_t 数组 aux16
    uint16_t aux16[2];
    // 将 aux16 强制转换为 const uint8_t 指针类型，并赋值给 scales
    const uint8_t * scales = (const uint8_t *)aux16;

    }

    // 计算 acc 中所有元素的和，并减去 summs，将结果赋值给 *s
    *s = hsum_float_8(acc) - summs;

#elif defined __riscv_v_intrinsic
    // 如果定义了 riscv_v_intrinsic，则执行以下代码块

    // 创建一个长度为 2 的 uint16_t 数组 s16
    uint16_t s16[2];
    // 将 s16 强制转换为 const uint8_t 指针类型，并赋值给 scales
    const uint8_t * restrict scales = (const uint8_t *)s16;

    // 初始化一个浮点数 sumf 为 0
    float sumf = 0;

    // 遍历 nb 次循环
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 和 y[i] 的相关属性
        const uint8_t * restrict q4 = x[i].qs;
        const  int8_t * restrict q8 = y[i].qs;

        // 将 x[i].scales 转换为 uint16_t 类型数组 b，并取其中的低 8 位和高 8 位，分别赋值给 s16[0] 和 s16[1]
        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        s16[0] = b[0] & 0x0f0f;
        s16[1] = (b[0] >> 4) & 0x0f0f;

        // 根据一系列计算更新 sumf 的值

        // 加载 Q4
        vuint8m1_t q4_x = __riscv_vle8_v_u8m1(q4, vl);

        // 加载 Q8 并将其与低 Q4 的 nibble 相乘
        vint8m1_t  q4_a = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(q4_x, 0x0F, vl));
        vint16m2_t va_0 = __riscv_vwmul_vv_i16m2(q4_a, __riscv_vle8_v_i8m1(q8, vl), vl);
        vint16m1_t aux1 = __riscv_vredsum_vs_i16m2_i16m1(va_0, vzero, vl);

        sumf += d*scales[0]*__riscv_vmv_x_s_i16m1_i16(aux1);

        // 加载 Q8 并将其与高 Q4 的 nibble 相乘
        vint8m1_t  q4_s = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vsrl_vx_u8m1(q4_x, 0x04, vl));
        vint16m2_t va_1 = __riscv_vwmul_vv_i16m2(q4_s, __riscv_vle8_v_i8m1(q8+32, vl), vl);
        vint16m1_t aux2 = __riscv_vredsum_vs_i16m2_i16m1(va_1, vzero, vl);

        sumf += d*scales[1]*__riscv_vmv_x_s_i16m1_i16(aux2);

    }

    // 将 sumf 的值赋给 *s

#else
    // 如果不满足以上条件，则执行以下代码块

    // 创建长度为 QK_K 的 uint8_t 数组 aux8
    uint8_t aux8[QK_K];
    // 创建长度为 16 的 int16_t 数组 aux16
    int16_t aux16[16];
    // 创建长度为 8 的 float 数组 sums，并将其所有元素初始化为 0
    float   sums [8];
    memset(sums, 0, 8*sizeof(float));

    // 创建长度为 2 的 uint16_t 数组 s16
    uint16_t s16[2];
    // 将 s16 强制转换为 uint8_t 类型指针，并赋值给 scales
    const uint8_t * restrict scales = (const uint8_t *)s16;

    // 初始化 sumf 为 0
    float sumf = 0;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 将 x[i].qs 强制转换为 uint8_t 类型指针，并赋值给 q4
        const uint8_t * restrict q4 = x[i].qs;
        // 将 y[i].qs 强制转换为 int8_t 类型指针，并赋值给 q8
        const int8_t * restrict q8 = y[i].qs;
        // 初始化 uint8_t 类型指针 a 指向 aux8
        uint8_t * restrict a = aux8;
        // 将 q4 中的低 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+ 0] = q4[l] & 0xF;
        // 将 q4 中的高 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+32] = q4[l]  >> 4;

        // 将 x[i].scales 强制转换为 uint16_t 类型指针，并赋值给 b
        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        // 将 b[0] 的低 8 位存入 s16[0]，高 8 位存入 s16[1]
        s16[0] = b[0] & 0x0f0f;
        s16[1] = (b[0] >> 4) & 0x0f0f;

        // 更新 sumf 的值
        sumf -= y[i].d * GGML_FP16_TO_FP32(x[i].d[1]) * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 计算 d 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d[0]);

        // 遍历 QK_K/32 次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算 aux16[l] 的值
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            q8 += 16; a += 16;
            // 计算 aux16[l] 的值
            for (int l = 0; l < 16; ++l) aux16[l] += q8[l] * a[l];
            q8 += 16; a += 16;
            // 计算 dl 的值
            const float dl = d * scales[j];
            // 更新 sums 的值
            for (int l = 0; l < 8; ++l) sums[l] += dl * (aux16[l] + aux16[l+8]);
        }
    }
    // 更新 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给 s
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 计算两个向量的点积，其中向量长度为 QK_K
void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将输入指针转换为特定类型的指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量中块的数量
    const int nb = n / QK_K;

    // 定义用于位运算的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    uint32_t utmp[4];

#ifdef __ARM_NEON
    // 定义 ARM NEON 指令需要使用的常量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const uint8x16_t mone = vdupq_n_u8(1);
    const uint8x16_t mtwo = vdupq_n_u8(2);
    const int32x4_t mzero = vdupq_n_s32(0);

    ggml_int8x16x4_t q5bytes;

    // 初始化浮点数求和
    float sumf = 0;

    }

    *s = sumf;

#elif defined __AVX2__

    // 定义 AVX2 指令需要使用的常量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m128i mzero = _mm_setzero_si128();
    const __m256i mone  = _mm256_set1_epi8(1);

    __m256 acc = _mm256_setzero_ps();

    // 初始化浮点数求和
    float summs = 0.f;

   for (int i = 0; i < nb; ++i) {

        // 获取指向当前块的指针
        const uint8_t * restrict q5 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

#if QK_K == 256
        // 计算缩放系数
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 复制缩放系数到临时数组，并进行位操作
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

    // 定义 AVX 指令需要使用的常量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i mzero = _mm_setzero_si128();
    const __m128i mone  = _mm_set1_epi8(1);
    const __m128i m2 = _mm_set1_epi8(2);

    __m256 acc = _mm256_setzero_ps();

    // 初始化浮点数求和
    float summs = 0.f;

    }

    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic
    // 将 utmp 数组中的前两个元素的地址转换为 uint8_t 类型的指针，并分别赋值给 scales 和 mins
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    // 初始化 sumf 和 sums 为 0
    float sumf = 0;
    float sums = 0.0;

    // 声明一个变量 vl，用于存储后续计算的结果

    }

    // 将 sumf 和 sums 的和赋值给指针 s 所指向的位置
    *s = sumf+sums;
#else
// 如果不是 ARM 平台，则执行以下代码块

    // 将 utmp 数组中的数据转换为 uint8_t 类型的指针，并赋值给 scales
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    // 将 utmp 数组中的数据转换为 uint8_t 类型的指针，并赋值给 mins
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    // 定义长度为 QK_K 的 int8_t 数组 aux8
    int8_t  aux8[QK_K];
    // 定义长度为 8 的 int16_t 数组 aux16
    int16_t aux16[8];
    // 定义长度为 8 的 float 数组 sums，并初始化为 0
    float   sums [8];
    // 定义长度为 8 的 int32_t 数组 aux32，并初始化为 0
    int32_t aux32[8];
    // 将 sums 数组的内容全部设置为 0
    memset(sums, 0, 8*sizeof(float));

    // 定义变量 sumf，并初始化为 0
    float sumf = 0;
    }
    // 遍历 sums 数组，计算总和并赋值给 sumf
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 指向的位置
    *s = sumf;
#endif
}

#else
// 如果不是 ARM 平台，则执行以下代码块

void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 强制转换为 block_q5_K 类型的指针，并赋值给 x
    const block_q5_K * restrict x = vx;
    // 将 vy 强制转换为 block_q8_K 类型的指针，并赋值给 y
    const block_q8_K * restrict y = vy;

    // 计算块数 nb，每个块包含 QK_K 个元素
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 创建一个包含 16 个相同值的 uint8x16_t 类型变量 m4b
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    // 创建一个包含 16 个相同值的 uint8x16_t 类型变量 mh
    const uint8x16_t mh = vdupq_n_u8(16);
    // 创建一个包含 4 个相同值的 int32x4_t 类型变量 mzero，并初始化为 0
    const int32x4_t mzero = vdupq_n_s32(0);

    // 定义 ggml_int8x16x4_t 和 ggml_uint8x16x4_t 类型的变量 q5bytes 和 q5h
    ggml_int8x16x4_t q5bytes;
    ggml_uint8x16x4_t q5h;

    // 定义变量 sumf，并初始化为 0
    float sumf = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * (float)x[i].d;
        // 获取 x[i] 的 scales
        const int8_t * sc = x[i].scales;

        // 获取 x[i] 的 qs
        const uint8_t * restrict q5 = x[i].qs;
        // 获取 x[i] 的 qh
        const uint8_t * restrict qh = x[i].qh;
        // 获取 y[i] 的 qs
        const int8_t  * restrict q8 = y[i].qs;

        // 加载 qh 的数据到 uint8x8_t 类型
        const uint8x8_t qhbits = vld1_u8(qh);

        // 加载 q5 的数据到 ggml_uint8x16x2_t 类型
        const ggml_uint8x16x2_t q5bits = ggml_vld1q_u8_x2(q5);
        // 加载 q8 的数据到 ggml_int8x16x4_t 类型
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 将 qhbits 的数据组合成 uint8x16_t 类型
        const uint8x16_t htmp = vcombine_u8(qhbits, vshr_n_u8(qhbits, 1));
        // 计算 q5h.val[0] 到 q5h.val[3]
        q5h.val[0] = vbicq_u8(mh, vshlq_n_u8(htmp, 4));
        q5h.val[1] = vbicq_u8(mh, vshlq_n_u8(htmp, 2));
        q5h.val[2] = vbicq_u8(mh, htmp);
        q5h.val[3] = vbicq_u8(mh, vshrq_n_u8(htmp, 2));

        // 计算 q5bytes.val[0] 到 q5bytes.val[3]
        q5bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[0], m4b)), vreinterpretq_s8_u8(q5h.val[0]);
        q5bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[1], m4b)), vreinterpretq_s8_u8(q5h.val[1]);
        q5bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[0], 4)), vreinterpretq_s8_u8(q5h.val[2]);
        q5bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[1], 4)), vreinterpretq_s8_u8(q5h.val[3]);

        // 计算 sumi1 到 sumi4
        int32_t sumi1 = sc[0] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]));
        int32_t sumi2 = sc[1] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[1], q8bytes.val[1]));
        int32_t sumi3 = sc[2] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]));
        int32_t sumi4 = sc[3] * vaddvq_s32(ggml_vdotq_s32(mzero, q5bytes.val[3], q8bytes.val[3]));

        // 更新 sumf
        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);
    }

    // 将 sumf 的值赋给 *s
    *s = sumf;
#elif defined __AVX2__

    // 设置一个包含全1字节的常量向量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 设置一个包含全1字节的常量向量
    const __m256i mone  = _mm256_set1_epi8(1);

    // 设置一个全0的256位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 循环处理每个输入数据
    for (int i = 0; i < nb; ++i) {

        // 获取第i个输入数据的qs和d
        const uint8_t * restrict q5 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 计算d的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 加载q5的数据到256位整数向量
        const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);

        // 设置256位整数向量scale_l和scale_h
        const __m256i scale_l = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[1]), _mm_set1_epi16(x[i].scales[0]));
        const __m256i scale_h = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[3]), _mm_set1_epi16(x[i].scales[2]));

        // 复制x[i].qh的值到aux64，然后转换为128位整数向量haux128和256位整数向量haux256
        int64_t aux64;
        memcpy(&aux64, x[i].qh, 8);
        const __m128i haux128 = _mm_set_epi64x(aux64 >> 1, aux64);
        const __m256i haux256 = MM256_SET_M128I(_mm_srli_epi16(haux128, 2), haux128);

        // 计算q5的高位和低位
        const __m256i q5h_0 = _mm256_slli_epi16(_mm256_andnot_si256(haux256, mone), 4);
        const __m256i q5h_1 = _mm256_slli_epi16(_mm256_andnot_si256(_mm256_srli_epi16(haux256, 4), mone), 4);
        const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);
        const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);

        // 加载q8的数据到256位整数向量q8_0和q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算p16_0、p16_1、s16_0、s16_1
        const __m256i p16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5l_0, q8_0));
        const __m256i p16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5l_1, q8_1));
        const __m256i s16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5h_0, q8_0));
        const __m256i s16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5h_1, q8_1));

        // 计算dot
        const __m256i dot = _mm256_sub_epi32(_mm256_add_epi32(p16_0, p16_1), _mm256_add_epi32(s16_0, s16_1));

        // 更新acc
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(dot), acc);

    }

    // 计算acc的和并存储到s
    *s = hsum_float_8(acc);

#elif defined __AVX__
    // 创建一个包含8个相同字节的128位整数寄存器，每个字节都是0xF
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个包含8个相同字节的128位整数寄存器，每个字节都是1
    const __m128i mone  = _mm_set1_epi8(1);

    // 创建一个256位浮点寄存器，所有元素初始化为0
    __m256 acc = _mm256_setzero_ps();

    // 计算累加和并将结果存储在指针s指向的位置
    *s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
``` 
# 如果定义了 __riscv_v_intrinsic，则执行以下代码块


    float sumf = 0;
``` 
# 初始化一个浮点数 sumf 为 0


    }
``` 
# 结束当前代码块


    *s = sumf;
``` 
# 将 sumf 赋值给指针 s 指向的变量


#else
``` 
# 如果未定义 __riscv_v_intrinsic，则执行以下代码块


    int8_t aux8[QK_K];
    int16_t aux16[16];
    float   sums [8];
    memset(sums, 0, 8*sizeof(float));
``` 
# 声明一个长度为 QK_K 的 int8_t 数组 aux8，一个长度为 16 的 int16_t 数组 aux16，一个长度为 8 的 float 数组 sums，并将 sums 数组清零


    float sumf = 0;
``` 
# 初始化一个浮点数 sumf 为 0


    for (int i = 0; i < nb; ++i) {
``` 
# 循环遍历 nb 次，其中 nb 为 n 除以 QK_K 的商


        const uint8_t * restrict q4 = x[i].qs;
        const uint8_t * restrict hm = x[i].qh;
        const  int8_t * restrict q8 = y[i].qs;
        int8_t * restrict a = aux8;
``` 
# 声明并初始化指向不同数据类型的指针


        for (int l = 0; l < 32; ++l) {
            a[l+ 0] = q4[l] & 0xF;
            a[l+32] = q4[l]  >> 4;
        }
``` 
# 将 q4 数组中的数据按位与运算后存入 aux8 数组中


        for (int is = 0; is < 8; ++is) {
            uint8_t m = 1 << is;
            for (int l = 0; l < 8; ++l) a[8*is + l] -= (hm[l] & m ? 0 : 16);
        }
``` 
# 根据条件对 aux8 数组中的数据进行修改


        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const int8_t * restrict sc = x[i].scales;
``` 
# 声明并初始化浮点数 d 和指向 int8_t 类型的指针 sc


        for (int j = 0; j < QK_K/16; ++j) {
``` 
# 循环遍历 QK_K/16 次


            const float dl = d * sc[j];
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l <  8; ++l) sums[l] += dl * (aux16[l] + aux16[8+l]);
            q8 += 16; a += 16;
        }
``` 
# 计算 dl 值，更新 aux16 数组和 sums 数组的值


    }
``` 
# 结束当前循环


    for (int l = 0; l < 8; ++l) sumf += sums[l];
    *s = sumf;
``` 
# 计算 sumf 的值，将其赋值给指针 s 指向的变量


#endif
``` 
# 结束当前代码块


}
#endif
``` 
# 结束当前代码块


#if QK_K == 256
``` 
# 如果 QK_K 等于 256，则执行以下代码块


void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
``` 
# 定义一个函数 ggml_vec_dot_q6_K_q8_K，接受参数 n、s、vx、vy


    assert(n % QK_K == 0);
``` 
# 断言 n 除以 QK_K 的余数为 0


    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;
``` 
# 声明并初始化指向 block_q6_K 和 block_q8_K 类型的指针 x 和 y


    const int nb = n / QK_K;
``` 
# 计算 nb 的值为 n 除以 QK_K 的商


#ifdef __ARM_NEON
``` 
# 如果定义了 __ARM_NEON，则执行以下代码块


    float sum = 0;
``` 
# 初始化一个浮点数 sum 为 0


    const uint8x16_t m4b = vdupq_n_u8(0xF);
    const int32x4_t  vzero = vdupq_n_s32(0);
    const uint8x16_t mone = vdupq_n_u8(3);
``` 
# 初始化不同类型的 NEON 寄存器


    ggml_int8x16x4_t q6bytes;
    ggml_uint8x16x4_t q6h;
``` 
# 声明两个自定义类型的变量


    }
    *s = sum;
``` 
# 将 sum 赋值给指针 s 指向的变量


#elif defined __AVX2__
``` 
# 如果定义了 __AVX2__，则执行以下代码块


    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i m2 = _mm256_set1_epi8(3);
    const __m256i m32s = _mm256_set1_epi8(32);
``` 
# 初始化 AVX2 寄存器


    __m256 acc = _mm256_setzero_ps();
``` 
# 初始化一个 AVX2 浮点数寄存器 acc 为 0


    }
    *s = hsum_float_8(acc);
``` 
# 将 acc 寄存器中的值求和后赋值给指针 s 指向的变量
#elif defined __AVX__
    # 如果定义了 AVX，则执行以下代码块

    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个包含全部元素为 0xF 的 128 位整数向量
    const __m128i m3 = _mm_set1_epi8(3);
    # 创建一个包含全部元素为 3 的 128 位整数向量
    const __m128i m32s = _mm_set1_epi8(32);
    # 创建一个包含全部元素为 32 的 128 位整数向量
    const __m128i m2 = _mm_set1_epi8(2);
    # 创建一个包含全部元素为 2 的 128 位整数向量

    __m256 acc = _mm256_setzero_ps();
    # 创建一个全部元素为 0 的 256 位浮点数向量 acc

    }

    *s = hsum_float_8(acc);
    # 将 acc 中的 8 个浮点数相加并将结果存储到 s 中

#elif defined __riscv_v_intrinsic
    # 如果定义了 riscv_v_intrinsic，则执行以下代码块

    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0

    }

    *s = sumf;
    # 将 sumf 的值存储到 s 中

#else
    # 如果以上条件都不满足，则执行以下代码块

    int8_t  aux8[QK_K];
    # 创建一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[8];
    # 创建一个长度为 8 的 int16_t 数组 aux16
    float   sums [8];
    # 创建一个长度为 8 的 float 数组 sums
    int32_t aux32[8];
    # 创建一个长度为 8 的 int32_t 数组 aux32
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组中的所有元素初始化为 0

    float sumf = 0;
    # 初始化一个浮点数 sumf 为 0
    for (int i = 0; i < nb; ++i) {
        # 循环遍历 i 从 0 到 nb-1
        const uint8_t * restrict q4 = x[i].ql;
        # 定义一个指向 x[i].ql 的 uint8_t 指针 q4
        const uint8_t * restrict qh = x[i].qh;
        # 定义一个指向 x[i].qh 的 uint8_t 指针 qh
        const  int8_t * restrict q8 = y[i].qs;
        # 定义一个指向 y[i].qs 的 int8_t 指针 q8
        memset(aux32, 0, 8*sizeof(int32_t));
        # 将 aux32 数组中的所有元素初始化为 0
        int8_t * restrict a = aux8;
        # 定义一个指向 aux8 的 int8_t 指针 a
        for (int j = 0; j < QK_K; j += 128) {
            # 循环遍历 j 从 0 到 QK_K-1，每次增加 128
            for (int l = 0; l < 32; ++l) {
                # 循环遍历 l 从 0 到 31
                a[l +  0] = (int8_t)((q4[l +  0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
                # 计算并赋值给 a[l+0]
                a[l + 32] = (int8_t)((q4[l + 32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
                # 计算并赋值给 a[l+32]
                a[l + 64] = (int8_t)((q4[l +  0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
                # 计算并赋值给 a[l+64]
                a[l + 96] = (int8_t)((q4[l + 32] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
                # 计算并赋值给 a[l+96]
            }
            a  += 128;
            # 指针 a 向后移动 128 个位置
            q4 += 64;
            # 指针 q4 向后移动 64 个位置
            qh += 32;
            # 指针 qh 向后移动 32 个位置
        }
        a = aux8;
        # 将指针 a 指向 aux8
        int is = 0;
        # 初始化一个整数 is 为 0
        for (int j = 0; j < QK_K/16; ++j) {
            # 循环遍历 j 从 0 到 QK_K/16-1
            int scale = x[i].scales[is++];
            # 获取 x[i].scales 中的值赋给 scale，并 is 自增
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 计算 aux32 数组的值
            q8 += 8; a += 8;
            # 指针 q8 和 a 各向后移动 8 个位置
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            # 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            # 计算 aux32 数组的值
            q8 += 8; a += 8;
            # 指针 q8 和 a 各向后移动 8 个位置
        }
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        # 计算 d 的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        # 计算 sums 数组的值
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 计算 sumf 的值
    *s = sumf;
    # 将 sumf 的值存储到 s 中
#endif
}
# 结束函数定义
#else
# 如果以上条件都不满足，则执行以下代码块
# 计算两个向量的点积，其中 n 为向量长度
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 断言向量长度为 QK_K 的整数倍
    assert(n % QK_K == 0);

    # 将输入指针转换为 block_q6_K 类型的指针
    const block_q6_K * restrict x = vx;
    # 将输入指针转换为 block_q8_K 类型的指针
    const block_q8_K * restrict y = vy;

    # 计算向量长度除以 QK_K 的商
    const int nb = n / QK_K;

    # 如果支持 ARM NEON 指令集
#ifdef __ARM_NEON
    # 初始化变量 sum 为 0
    float sum = 0;

    # 创建常量向量，用于 NEON 指令
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    const int8x16_t  m32s = vdupq_n_s8(32);
    const int32x4_t  vzero = vdupq_n_s32(0);

    # 创建常量向量，用于 NEON 指令
    const uint8x16_t mone = vdupq_n_u8(3);

    # 定义 NEON 数据类型变量
    ggml_int8x16x4_t q6bytes;
    ggml_uint8x16x4_t q6h;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 的 d 值，并转换为 float 类型
        const float d_all = (float)x[i].d;

        // 获取 x[i] 的 ql、qh、qs、scales 值
        const uint8_t * restrict q6 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const int8_t  * restrict q8 = y[i].qs;
        const int8_t * restrict scale = x[i].scales;

        // 初始化 isum 为 0
        int32_t isum = 0;

        // 从 qh、q6、q8 中加载数据到寄存器
        uint8x16_t qhbits = vld1q_u8(qh);
        ggml_uint8x16x2_t q6bits = ggml_vld1q_u8_x2(q6);
        ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 对 qhbits 进行位运算，得到 q6h 的值
        q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits), 4);
        uint8x16_t shifted = vshrq_n_u8(qhbits, 2);
        q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 4);
        q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 6);
        q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

        // 对 q6bits 进行位运算，得到 q6bytes 的值
        q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);
        q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);
        q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[2])), m32s);
        q6bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[3])), m32s);

        // 计算 isum 的值
        isum += vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                vaddvq_s32(ggml_vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];

        // 更新 sum 的值
        sum += isum * d_all * y[i].d;

    }
    // 将最终结果赋值给 s
    *s = sum;
#elif defined __AVX2__

    // 定义 AVX2 指令集下的常量 m4，用于设置每个字节为 0xF
    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 定义 AVX2 指令集下的常量 m2，用于设置每个字节为 3
    const __m256i m2 = _mm256_set1_epi8(3);
    // 定义 AVX2 指令集下的常量 m32s，用于设置每个字节为 32
    const __m256i m32s = _mm256_set1_epi8(32);

    // 初始化一个全为零的 __m256 类型变量 acc
    __m256 acc = _mm256_setzero_ps();

    }

    // 调用 hsum_float_8 函数对 acc 进行求和，并将结果存储到 s 指向的地址
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 定义 AVX 指令集下的常量 m4，用于设置每个字节为 0xF
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 定义 AVX 指令集下的常量 m2，用于设置每个字节为 3
    const __m128i m2 = _mm_set1_epi8(3);
    // 定义 AVX 指令集下的常量 m32s，用于设置每个字节为 32
    const __m128i m32s = _mm_set1_epi8(32);

    // 初始化一个全为零的 __m256 类型变量 acc
    __m256 acc = _mm256_setzero_ps();

    }

    // 调用 hsum_float_8 函数对 acc 进行求和，并将结果存储到 s 指向的地址
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic

    // 初始化一个浮点数 sumf 为 0
    float sumf = 0;

    }

    // 将 sumf 的值存储到 s 指向的地址
    *s = sumf;

#else

    // 定义长度为 QK_K 的 int8_t 类型数组 aux8
    int8_t  aux8[QK_K];
    // 定义长度为 8 的 int16_t 类型数组 aux16
    int16_t aux16[8];
    // 定义长度为 8 的 float 类型数组 sums
    float   sums [8];
    // 定义长度为 8 的 int32_t 类型数组 aux32
    int32_t aux32[8];
    // 将 sums 数组中的所有元素初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 初始化一个浮点数 sumf 为 0
    float sumf = 0;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 的 ql 和 qh 指针
        const uint8_t * restrict q4 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        // 获取 y[i] 的 qs 指针
        const  int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组中的所有元素初始化为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        // 定义指向 aux8 数组的指针 a
        int8_t * restrict a = aux8;
        // 遍历 16 次
        for (int l = 0; l < 16; ++l) {
            // 计算 a 数组中的值
            a[l+ 0] = (int8_t)((q4[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            a[l+16] = (int8_t)((q4[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            a[l+32] = (int8_t)((q4[l+ 0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            a[l+48] = (int8_t)((q4[l+16] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
        }
        // 初始化 is 为 0
        int is = 0;
        // 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 获取 x[i] 的 scales 中的值
            int scale = x[i].scales[is++];
            // 计算 aux16 数组中的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 计算 aux32 数组中的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
            // 计算 aux16 数组中的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 计算 aux32 数组中的值
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            q8 += 8; a += 8;
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 更新 sums 数组中的值
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    // 将 sums 数组中的所有元素相加得到 sumf
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值存储到 s 指向的地址
    *s = sumf;
#endif
}

#endif

};
void ggml_vec_dot_iq2_xxs_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_iq2_xxs 和 block_q8_K 类型的指针
    const block_iq2_xxs * restrict x = vx;
    const block_q8_K    * restrict y = vy;

    // 计算 nb，即 n 除以 QK_K 的商
    const int nb = n / QK_K;

#if defined(__ARM_NEON)

    // 定义并初始化一些变量
    const uint64_t * signs64 = (const uint64_t *)keven_signs_q2xs;

    uint32_t aux32[4];
    const uint8_t * aux8 = (const uint8_t *)aux32;

    ggml_int8x16x4_t q2u;
    ggml_int8x16x4_t q2s;
    ggml_int8x16x4_t q8b;

    float sumf = 0;
    // 计算 sumf 的值

    *s = 0.25f * sumf; // 将 sumf 乘以 0.25 后赋值给 s

#elif defined(__AVX2__)

    // 定义并初始化一些变量
    const uint64_t * signs64 = (const uint64_t *)keven_signs_q2xs;

    uint32_t aux32[4];
    const uint8_t * aux8 = (const uint8_t *)aux32;

    __m256 accumf = _mm256_setzero_ps();
    // 计算 accumf 的值

    *s = 0.125f * hsum_float_8(accumf); // 将 accumf 的和乘以 0.125 后赋值给 s

#else

    // 定义一些变量
    uint32_t aux32[2];
    const uint8_t * aux8 = (const uint8_t *)aux32;

    float sumf = 0.f;
    // 计算 sumf 的值
    for (int i = 0; i < nb; ++i) {
        // 计算 sumf 的值
    }
    *s = 0.125f * sumf; // 将 sumf 乘以 0.125 后赋值给 s
#endif
}

void ggml_vec_dot_iq2_xs_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);
    // 定义指向vx的常量指针x，限制指针指向的数据块为block_iq2_xs类型
    const block_iq2_xs * restrict x = vx;
    // 定义指向vy的常量指针y，限制指针指向的数据块为block_q8_K类型
    const block_q8_K   * restrict y = vy;

    // 计算n除以QK_K的商，赋值给nb
    const int nb = n / QK_K;
#if defined(__ARM_NEON)
    // 如果定义了 ARM NEON，执行以下代码块

    const uint64_t * signs64 = (const uint64_t *)keven_signs_q2xs;
    // 定义一个指向 keven_signs_q2xs 的 uint64_t 指针 signs64

    ggml_int8x16x4_t q2u;
    // 定义一个 ggml_int8x16x4_t 类型的变量 q2u
    ggml_int8x16x4_t q2s;
    // 定义一个 ggml_int8x16x4_t 类型的变量 q2s
    ggml_int8x16x4_t q8b;
    // 定义一个 ggml_int8x16x4_t 类型的变量 q8b

    int32x4x4_t scales32;
    // 定义一个 int32x4x4_t 类型的变量 scales32

    float sumf = 0;
    // 定义一个浮点数变量 sumf，并初始化为 0

    *s = 0.125f * sumf;
    // 将 sumf 乘以 0.125f 的结果赋值给指针 s

#elif defined(__AVX2__)
    // 如果定义了 AVX2，执行以下代码块

    const __m128i m4 = _mm_set1_epi8(0xf);
    // 定义一个 __m128i 类型的常量 m4，值为 0xf
    const __m128i m1 = _mm_set1_epi8(1);
    // 定义一个 __m128i 类型的常量 m1，值为 1
    const __m256i m511 = _mm256_set1_epi16(511);
    // 定义一个 __m256i 类型的常量 m511，值为 511
    const __m256i mone = _mm256_set1_epi8(1);
    // 定义一个 __m256i 类型的常量 mone，值为 1

    static const uint8_t k_bit_helper[32] = {
        // 定义一个静态的 uint8_t 类型数组 k_bit_helper，包含 32 个元素
        0x00, 0x80, 0x80, 0x00, 0x80, 0x00, 0x00, 0x80, 0x80, 0x00, 0x00, 0x80, 0x00, 0x80, 0x80, 0x00,
        0x00, 0x80, 0x80, 0x00, 0x80, 0x00, 0x00, 0x80, 0x80, 0x00, 0x00, 0x80, 0x00, 0x80, 0x80, 0x00,
    };
    static const char block_sign_shuffle_mask_1[32] = {
        // 定义一个静态的 char 类型数组 block_sign_shuffle_mask_1，包含 32 个元素
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x02, 0x02, 0x02, 0x02, 0x02, 0x02, 0x02,
        0x04, 0x04, 0x04, 0x04, 0x04, 0x04, 0x04, 0x04, 0x06, 0x06, 0x06, 0x06, 0x06, 0x06, 0x06, 0x06,
    };
    static const char block_sign_shuffle_mask_2[32] = {
        // 定义一个静态的 char 类型数组 block_sign_shuffle_mask_2，包含 32 个元素
        0x08, 0x08, 0x08, 0x08, 0x08, 0x08, 0x08, 0x08, 0x0a, 0x0a, 0x0a, 0x0a, 0x0a, 0x0a, 0x0a, 0x0a,
        0x0c, 0x0c, 0x0c, 0x0c, 0x0c, 0x0c, 0x0c, 0x0c, 0x0e, 0x0e, 0x0e, 0x0e, 0x0e, 0x0e, 0x0e, 0x0e,
    };
    static const uint8_t bit_selector_mask_bytes[32] = {
        // 定义一个静态的 uint8_t 类型数组 bit_selector_mask_bytes，包含 32 个元素
        0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
        0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
    };

    const __m256i bit_helper = _mm256_loadu_si256((const __m256i*)k_bit_helper);
    // 将 k_bit_helper 载入到 __m256i 类型的变量 bit_helper
    const __m256i bit_selector_mask = _mm256_loadu_si256((const __m256i*)bit_selector_mask_bytes);
    // 将 bit_selector_mask_bytes 载入到 __m256i 类型的变量 bit_selector_mask
    const __m256i block_sign_shuffle_1 = _mm256_loadu_si256((const __m256i*)block_sign_shuffle_mask_1);
    // 将 block_sign_shuffle_mask_1 载入到 __m256i 类型的变量 block_sign_shuffle_1
    const __m256i block_sign_shuffle_2 = _mm256_loadu_si256((const __m256i*)block_sign_shuffle_mask_2);
    // 将 block_sign_shuffle_mask_2 载入到 __m256i 类型的变量 block_sign_shuffle_2

    uint64_t aux64;
    // 定义一个 uint64_t 类型的变量 aux64
    // 使用 SIMD 指令集中的 __m256i 类型来存储辅助的全局索引
    __m256i aux_gindex;
    // 将辅助全局索引转换为 uint16_t 类型的指针
    const uint16_t * gindex = (const uint16_t *)&aux_gindex;

    // 使用 SIMD 指令集中的 __m256 类型来初始化一个全零的累加器
    __m256 accumf = _mm256_setzero_ps();
    }

    // 将累加器中的值乘以 0.125，并将结果存储到指定的内存地址中
    *s = 0.125f * hsum_float_8(accumf);
#else
// 如果不是 ARM_NEON 或者 AVX2 架构，则执行以下代码

    // 初始化浮点数 sumf 为 0
    float sumf = 0.f;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 获取 x[i] 的 qs、scales 和 y[i] 的 qs
        const uint16_t * restrict q2 = x[i].qs;
        const uint8_t  * restrict sc = x[i].scales;
        const int8_t   * restrict q8 = y[i].qs;
        // 初始化 bsum 为 0
        int32_t bsum = 0;
        // 遍历 QK_K/32 次
        for (int ib32 = 0; ib32 < QK_K/32; ++ib32) {
            // 计算 ls1 和 ls2 的值
            const uint16_t ls1 = 2*(sc[ib32] & 0xf) + 1;
            const uint16_t ls2 = 2*(sc[ib32] >>  4) + 1;
            // 初始化 sumi 为 0
            int32_t sumi = 0;
            // 遍历两次
            for (int l = 0; l < 2; ++l) {
                // 获取 grid 和 signs 的值
                const uint8_t * grid = (const uint8_t *)(iq2xs_grid + (q2[l] & 511));
                const uint8_t  signs = ksigns_iq2xs[q2[l] >> 9];
                // 遍历 8 次
                for (int j = 0; j < 8; ++j) {
                    // 计算 sumi 的值
                    sumi += grid[j] * q8[j] * (signs & kmask_iq2xs[j] ? -1 : 1);
                }
                q8 += 8;
            }
            // 更新 bsum 的值
            bsum += sumi * ls1;
            sumi = 0;
            // 遍历两次
            for (int l = 2; l < 4; ++l) {
                // 获取 grid 和 signs 的值
                const uint8_t * grid = (const uint8_t *)(iq2xs_grid + (q2[l] & 511));
                const uint8_t  signs = ksigns_iq2xs[q2[l] >> 9];
                // 遍历 8 次
                for (int j = 0; j < 8; ++j) {
                    // 计算 sumi 的值
                    sumi += grid[j] * q8[j] * (signs & kmask_iq2xs[j] ? -1 : 1);
                }
                q8 += 8;
            }
            // 更新 bsum 的值
            bsum += sumi * ls2;
            q2 += 4;
        }
        // 更新 sumf 的值
        sumf += d * bsum;
    }
    // 更新 s 的值
    *s = 0.125f * sumf;
#endif
}

// TODO
// 计算向量的点积
void ggml_vec_dot_iq3_xxs_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 获取 vx 和 vy 的指针
    const block_iq3_xxs * restrict x = vx;
    const block_q8_K    * restrict y = vy;

    // 计算 nb 的值
    const int nb = n / QK_K;

#if defined(__ARM_NEON)

    // 获取 keven_signs_q2xs 的指针
    const uint64_t * signs64 = (const uint64_t *)keven_signs_q2xs;

    // 初始化 aux32 数组
    uint32_t aux32[2];

    // 初始化 q3s 和 q8b
    ggml_int8x16x4_t q3s;
    ggml_int8x16x4_t q8b;

    // 初始化浮点数 sumf 为 0
    float sumf = 0;
    // 更新 s 的值
    *s = 0.5f * sumf;

#elif defined(__AVX2__)
    // 将 keven_signs_q2xs 转换为 uint64_t 类型的指针，并赋值给 signs64
    const uint64_t * signs64 = (const uint64_t *)keven_signs_q2xs;

    // 创建一个包含两个 uint32_t 元素的数组 aux32
    uint32_t aux32[2];

    // 创建一个 __m256 类型的变量 accumf，并初始化为全零
    __m256 accumf = _mm256_setzero_ps();
    }

    // 将 accumf 中的 8 个 float 类型元素求和，并乘以 0.25，将结果赋值给 *s
    *s = 0.25f * hsum_float_8(accumf);
#else

    // 定义辅助变量 aux32
    uint32_t aux32;

    // 初始化浮点数 sumf 为 0
    float sumf = 0.f;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 计算浮点数 d
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 获取指向 x[i].qs 的指针
        const uint8_t * restrict q3 = x[i].qs;
        // 获取指向 x[i].qs + QK_K/4 的指针
        const uint8_t * restrict gas = x[i].qs + QK_K/4;
        // 获取指向 y[i].qs 的指针
        const int8_t  * restrict q8 = y[i].qs;
        // 初始化 bsum 为 0
        int32_t bsum = 0;
        // 遍历 QK_K/32 次
        for (int ib32 = 0; ib32 < QK_K/32; ++ib32) {
            // 从 gas 复制一个 uint32_t 到 aux32
            memcpy(&aux32, gas, sizeof(uint32_t)); gas += sizeof(uint32_t);
            // 计算 ls
            const uint32_t ls = 2*(aux32 >> 28) + 1;
            // 初始化 sumi 为 0
            int32_t sumi = 0;
            // 遍历 4 次
            for (int l = 0; l < 4; ++l) {
                // 获取 grid1 和 grid2 的指针
                const uint8_t * grid1 = (const uint8_t *)(iq3xxs_grid + q3[2*l+0]);
                const uint8_t * grid2 = (const uint8_t *)(iq3xxs_grid + q3[2*l+1]);
                // 获取 signs
                const uint8_t  signs = ksigns_iq2xs[(aux32 >> 7*l) & 127];
                // 遍历 4 次
                for (int j = 0; j < 4; ++j) {
                    // 计算 sumi
                    sumi += grid1[j] * q8[j+0] * (signs & kmask_iq2xs[j+0] ? -1 : 1);
                    sumi += grid2[j] * q8[j+4] * (signs & kmask_iq2xs[j+4] ? -1 : 1);
                }
                q8 += 8;
            }
            q3 += 8;
            bsum += sumi * ls;
        }
        sumf += d * bsum;
    }
    // 将 sumf 的值乘以 0.25 赋给 *s
    *s = 0.25f * sumf;
#endif
}

// ================================ IQ2 quantization =============================================

// 定义 IQ2 数据结构
typedef struct {
    uint64_t * grid;
    int      * map;
    uint16_t * neighbours;
} iq2_entry_t;

// 初始化 IQ2 数据
static iq2_entry_t iq2_data[2] = {
    {NULL, NULL, NULL},
    {NULL, NULL, NULL},
};

// 返回 IQ2 数据的索引
static inline int iq2_data_index(int grid_size) {
    GGML_ASSERT(grid_size == 256 || grid_size == 512);
    return grid_size == 256 ? 0 : 1;
}

// 比较函数，用于排序
static int iq2_compare_func(const void * left, const void * right) {
    const int * l = (const int *)left;
    const int * r = (const int *)right;
    return l[0] < r[0] ? -1 : l[0] > r[0] ? 1 : l[1] < r[1] ? -1 : l[1] > r[1] ? 1 : 0;
}

// 初始化 IQ2 量化
void iq2xs_init_impl(int grid_size) {
    // 获取 IQ2 数据的索引
    const int gindex = iq2_data_index(grid_size);
    # 如果 iq2_data[gindex].grid 存在（不为 null 或 undefined），则直接返回，不执行后续代码
    if (iq2_data[gindex].grid) {
        return;
    }
    // 定义一个静态的常量数组，包含256个uint16_t类型的元素
    static const uint16_t kgrid_256[256] = {
            // 数组元素的值依次为0, 2, 5, 8, ...，共256个元素
            0,     2,     5,     8,    10,    17,    20,    32,    34,    40,    42,    65,    68,    80,    88,    97,
            100,   128,   130,   138,   162,   257,   260,   272,   277,   320,   388,   408,   512,   514,   546,   642,
            1025,  1028,  1040,  1057,  1060,  1088,  1090,  1096,  1120,  1153,  1156,  1168,  1188,  1280,  1282,  1288,
            1312,  1350,  1385,  1408,  1425,  1545,  1552,  1600,  1668,  1700,  2048,  2053,  2056,  2068,  2088,  2113,
            2116,  2128,  2130,  2184,  2308,  2368,  2562,  2580,  4097,  4100,  4112,  4129,  4160,  4192,  4228,  4240,
            4245,  4352,  4360,  4384,  4432,  4442,  4480,  4644,  4677,  5120,  5128,  5152,  5157,  5193,  5248,  5400,
            5474,  5632,  5654,  6145,  6148,  6160,  6208,  6273,  6400,  6405,  6560,  6737,  8192,  8194,  8202,  8260,
            8289,  8320,  8322,  8489,  8520,  8704,  8706,  9217,  9220,  9232,  9280,  9302,  9472,  9537,  9572,  9872,
            10248, 10272, 10388, 10820, 16385, 16388, 16400, 16408, 16417, 16420, 16448, 16456, 16470, 16480, 16513, 16516,
            16528, 16640, 16672, 16737, 16768, 16773, 16897, 16912, 16968, 16982, 17000, 17408, 17416, 17440, 17536, 17561,
            17682, 17700, 17920, 18433, 18436, 18448, 18496, 18501, 18688, 18776, 18785, 18818, 19013, 19088, 20480, 20488,
            20497, 20505, 20512, 20608, 20616, 20740, 20802, 20900, 21137, 21648, 21650, 21770, 22017, 22100, 22528, 22545,
            22553, 22628, 22848, 23048, 24580, 24592, 24640, 24680, 24832, 24917, 25112, 25184, 25600, 25605, 25872, 25874,
            25988, 26690, 32768, 32770, 32778, 32833, 32898, 33028, 33048, 33088, 33297, 33793, 33796, 33808, 33813, 33856,
            33888, 34048, 34118, 34196, 34313, 34368, 34400, 34818, 35076, 35345, 36868, 36880, 36900, 36928, 37025, 37142,
            37248, 37445, 37888, 37922, 37956, 38225, 39041, 39200, 40962, 41040, 41093, 41225, 41472, 42008, 43088, 43268,
    };
    };
    // 定义常量 kmap_size 为 43692
    const int kmap_size = 43692;
    // 定义常量 nwant 为 2
    const int nwant = 2;
    // 根据条件判断，选择不同大小的 kgrid 数组
    const uint16_t * kgrid = grid_size == 256 ? kgrid_256 : kgrid_512;
    // 声明指向 uint64_t 类型的指针 kgrid_q2xs
    uint64_t * kgrid_q2xs;
    // 声明指向 int 类型的指针 kmap_q2xs
    int      * kmap_q2xs;
    // 声明指向 uint16_t 类型的指针 kneighbors_q2xs

    // 打印函数名和 grid_size 的值
    printf("================================================================= %s(grid_size = %d)\n", __func__, grid_size);
    // 分配内存给 the_grid，大小为 grid_size 乘以 uint64_t 的大小
    uint64_t * the_grid = (uint64_t *)malloc(grid_size*sizeof(uint64_t));
    // 遍历 the_grid 数组
    for (int k = 0; k < grid_size; ++k) {
        // 将 the_grid 数组中的元素转换为 int8_t 类型指针 pos
        int8_t * pos = (int8_t *)(the_grid + k);
        // 遍历 pos 指向的数组
        for (int i = 0; i < 8; ++i) {
            // 从 kgrid[k] 中提取数据，计算 l 的值
            int l = (kgrid[k] >> 2*i) & 0x3;
            // 根据 l 的值计算 pos[i] 的值
            pos[i] = 2*l + 1;
        }
    }
    // 将 the_grid 赋值给 kgrid_q2xs
    kgrid_q2xs = the_grid;
    // 将 the_grid 赋值给 iq2_data[gindex].grid
    iq2_data[gindex].grid = the_grid;
    // 分配内存给 kmap_q2xs，大小为 kmap_size 乘以 int 的大小
    kmap_q2xs = (int *)malloc(kmap_size*sizeof(int));
    // 将 kmap_q2xs 数组中的元素初始化为 -1
    for (int i = 0; i < kmap_size; ++i) kmap_q2xs[i] = -1;
    // 声明辅助变量 aux64 和 aux8
    uint64_t aux64;
    uint8_t * aux8 = (uint8_t *)&aux64;
    // 遍历 grid_size
    for (int i = 0; i < grid_size; ++i) {
        // 将 kgrid_q2xs[i] 赋值给 aux64
        aux64 = kgrid_q2xs[i];
        // 初始化 index 为 0
        uint16_t index = 0;
        // 遍历 aux8 数组
        for (int k=0; k<8; ++k) {
            // 计算 q 的值
            uint16_t q = (aux8[k] - 1)/2;
            // 更新 index 的值
            index |= (q << 2*k);
        }
        // 将 i 赋值给 kmap_q2xs[index]
        kmap_q2xs[index] = i;
    }
    // 声明 pos 数组
    int8_t pos[8];
    // 分配内存给 dist2，大小为 2 倍的 grid_size 乘以 int 的大小
    int * dist2 = (int *)malloc(2*grid_size*sizeof(int));
    // 初始化 num_neighbors 和 num_not_in_map 的值为 0
    int num_neighbors = 0, num_not_in_map = 0;
    // 遍历 kmap_q2xs 数组，查找未被映射的元素
    for (int i = 0; i < kmap_size; ++i) {
        // 如果当前元素已经映射，则跳过
        if (kmap_q2xs[i] >= 0) continue;
        // 未被映射的元素数量加一
        ++num_not_in_map;
        // 根据当前元素的索引计算出 pos 数组的值
        for (int k = 0; k < 8; ++k) {
            int l = (i >> 2*k) & 0x3;
            pos[k] = 2*l + 1;
        }
        // 遍历 grid_size，计算每个元素与 pos 的距离平方
        for (int j = 0; j < grid_size; ++j) {
            const int8_t * pg = (const int8_t *)(kgrid_q2xs + j);
            int d2 = 0;
            for (int k = 0; k < 8; ++k) d2 += (pg[k] - pos[k])*(pg[k] - pos[k]);
            dist2[2*j+0] = d2;
            dist2[2*j+1] = j;
        }
        // 对 dist2 数组进行排序
        qsort(dist2, grid_size, 2*sizeof(int), iq2_compare_func);
        int n = 0; int d2 = dist2[0];
        int nhave = 1;
        // 遍历 grid_size，计算邻居数量
        for (int j = 0; j < grid_size; ++j) {
            if (dist2[2*j] > d2) {
                if (nhave == nwant) break;
                d2 = dist2[2*j];
                ++nhave;
            }
            ++n;
        }
        // 更新邻居总数
        num_neighbors += n;
    }
    // 打印函数名和邻居总数
    printf("%s: %d neighbours in total\n", __func__, num_neighbors);
    // 分配内存给 kneighbors_q2xs 数组
    kneighbors_q2xs = (uint16_t *)malloc((num_neighbors + num_not_in_map)*sizeof(uint16_t));
    // 将 kneighbors_q2xs 数组赋值给 iq2_data[gindex].neighbours
    iq2_data[gindex].neighbours = kneighbors_q2xs;
    // 初始化计数器
    int counter = 0;
    // 遍历 kmap_q2xs 数组，如果当前元素大于等于0，则跳过
    for (int i = 0; i < kmap_size; ++i) {
        if (kmap_q2xs[i] >= 0) continue;
        // 根据当前索引 i 计算 pos 数组的值
        for (int k = 0; k < 8; ++k) {
            int l = (i >> 2*k) & 0x3;
            pos[k] = 2*l + 1;
        }
        // 遍历 grid_size，计算每个点到 pos 的距离平方，存储在 dist2 数组中
        for (int j = 0; j < grid_size; ++j) {
            const int8_t * pg = (const int8_t *)(kgrid_q2xs + j);
            int d2 = 0;
            for (int k = 0; k < 8; ++k) d2 += (pg[k] - pos[k])*(pg[k] - pos[k]);
            dist2[2*j+0] = d2;
            dist2[2*j+1] = j;
        }
        // 对 dist2 数组进行排序
        qsort(dist2, grid_size, 2*sizeof(int), iq2_compare_func);
        // 更新 kmap_q2xs[i] 的值
        kmap_q2xs[i] = -(counter + 1);
        int d2 = dist2[0];
        uint16_t * start = &kneighbors_q2xs[counter++];
        int n = 0, nhave = 1;
        // 遍历 grid_size，根据距离筛选出最近的 nwant 个点
        for (int j = 0; j < grid_size; ++j) {
            if (dist2[2*j] > d2) {
                if (nhave == nwant) break;
                d2 = dist2[2*j];
                ++nhave;
            }
            kneighbors_q2xs[counter++] = dist2[2*j+1];
            ++n;
        }
        // 将 n 存储到 start 指向的位置
        *start = n;
    }
    // 释放 dist2 数组的内存
    free(dist2);
}

// 释放 IQ2 数据结构中指定网格大小的内存
void iq2xs_free_impl(int grid_size) {
    // 确保网格大小为 256、512 或 1024
    GGML_ASSERT(grid_size == 256 || grid_size == 512 || grid_size == 1024);
    // 获取 IQ2 数据结构中指定网格大小的索引
    const int gindex = iq2_data_index(grid_size);
    // 如果 IQ2 数据结构中指定网格大小的网格存在，则释放内存并将指针置为 NULL
    if (iq2_data[gindex].grid) {
        free(iq2_data[gindex].grid);       iq2_data[gindex].grid = NULL;
        free(iq2_data[gindex].map);        iq2_data[gindex].map  = NULL;
        free(iq2_data[gindex].neighbours); iq2_data[gindex].neighbours = NULL;
    }
}

// 在指定邻居中找到最佳邻居
static int iq2_find_best_neighbour(const uint16_t * restrict neighbours, const uint64_t * restrict grid,
        const float * restrict xval, const float * restrict weight, float scale, int8_t * restrict L) {
    // 获取邻居数量
    int num_neighbors = neighbours[0];
    // 确保邻居数量大于 0
    GGML_ASSERT(num_neighbors > 0);
    // 初始化最佳距离为最大值
    float best_d2 = FLT_MAX;
    int grid_index = -1;
    // 遍历邻居
    for (int j = 1; j <= num_neighbors; ++j) {
        // 获取当前邻居的指针
        const int8_t * pg = (const int8_t *)(grid + neighbours[j]);
        float d2 = 0;
        // 计算距离
        for (int i = 0; i < 8; ++i) {
            float q = pg[i];
            float diff = scale*q - xval[i];
            d2 += weight[i]*diff*diff;
        }
        // 更新最佳距离和网格索引
        if (d2 < best_d2) {
            best_d2 = d2; grid_index = neighbours[j];
        }
    }
    // 确保找到的最佳邻居索引大于等于 0
    GGML_ASSERT(grid_index >= 0);
    // 获取最佳邻居的指针
    const int8_t * pg = (const int8_t *)(grid + grid_index);
    // 将最佳邻居的值进行量化处理并存储到 L 中
    for (int i = 0; i < 8; ++i) L[i] = (pg[i] - 1)/2;
    return grid_index;
}

// 对输入数据进行 IQ2_XXS 量化处理
static void quantize_row_iq2_xxs_impl(const float * restrict x, void * restrict vy, int n, const float * restrict quant_weights) {

    // 获取 IQ2 数据结构中网格大小为 256 的索引
    const int gindex = iq2_data_index(256);

    // 获取 IQ2 数据结构中网格、映射和邻居的指针
    const uint64_t * kgrid_q2xs      = iq2_data[gindex].grid;
    const int      * kmap_q2xs       = iq2_data[gindex].map;
    const uint16_t * kneighbors_q2xs = iq2_data[gindex].neighbours;

    // 确保量化权重、网格和映射存在
    GGML_ASSERT(quant_weights   && "missing quantization weights");
    GGML_ASSERT(kgrid_q2xs      && "forgot to call ggml_quantize_init()?");
    GGML_ASSERT(kmap_q2xs       && "forgot to call ggml_quantize_init()?");
    // 检查 kneighbors_q2xs 是否为空，如果为空则输出提示信息
    GGML_ASSERT(kneighbors_q2xs && "forgot to call ggml_quantize_init()?");
    // 检查 n 是否为 QK_K 的倍数，如果不是则输出提示信息
    GGML_ASSERT(n%QK_K == 0);

    // 定义最大的 Q 值为 3
    const int kMaxQ = 3;

    // 计算 n 的块数
    const int nbl = n/256;

    // 初始化指针 y 指向 vy
    block_iq2_xxs * y = vy;

    // 定义一些数组和变量
    float scales[QK_K/32];
    float weight[32];
    float xval[32];
    int8_t L[32];
    int8_t Laux[32];
    float  waux[32];
    bool   is_on_grid[4];
    bool   is_on_grid_aux[4];
    uint8_t block_signs[4];
    uint32_t q2[2*(QK_K/32)];
}

static void quantize_row_iq2_xs_impl(const float * restrict x, void * restrict vy, int n, const float * restrict quant_weights) {

    // 获取 IQ2 数据的索引
    const int gindex = iq2_data_index(512);

    // 获取 IQ2 数据中的网格、映射和邻居信息
    const uint64_t * kgrid_q2xs      = iq2_data[gindex].grid;
    const int      * kmap_q2xs       = iq2_data[gindex].map;
    const uint16_t * kneighbors_q2xs = iq2_data[gindex].neighbours;

    // 断言量化权重不为空
    GGML_ASSERT(quant_weights   && "missing quantization weights");
    // 断言映射不为空
    GGML_ASSERT(kmap_q2xs       && "forgot to call ggml_quantize_init()?");
    // 断言网格不为空
    GGML_ASSERT(kgrid_q2xs      && "forgot to call ggml_quantize_init()?");
    // 断言邻居信息不为空
    GGML_ASSERT(kneighbors_q2xs && "forgot to call ggml_quantize_init()?");
    // 断言 n 是 QK_K 的倍数
    GGML_ASSERT(n%QK_K == 0);

    // 定义最大量化值
    const int kMaxQ = 3;

    // 计算块数
    const int nbl = n/256;

    // 初始化输出指针
    block_iq2_xs * y = vy;

    // 定义一些数组和变量
    float scales[QK_K/16];
    float weight[16];
    float xval[16];
    int8_t L[16];
    int8_t Laux[16];
    float  waux[16];
    bool   is_on_grid[2];
    bool   is_on_grid_aux[2];
    uint8_t block_signs[2];
    uint16_t q2[2*(QK_K/16)];

    }
}

// 对输入数据进行 IQ2_XXS 量化
size_t quantize_iq2_xxs(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    (void)hist;
    // 断言每行数据的数量是 QK_K 的倍数
    GGML_ASSERT(n_per_row%QK_K == 0);
    // 计算每个块的数量
    int nblock = n_per_row/QK_K;
    // 将输出指针转换为字符指针
    char * qrow = (char *)dst;
    // 遍历每行数据
    for (int row = 0; row < nrow; ++row) {
        // 对每行数据进行 IQ2_XXS 量化
        quantize_row_iq2_xxs_impl(src, qrow, n_per_row, quant_weights);
        // 更新输入数据指针
        src += n_per_row;
        // 更新输出数据指针
        qrow += nblock*sizeof(block_iq2_xxs);
    }
    // 返回总共量化的字节数
    return nrow * nblock * sizeof(block_iq2_xxs);
}

// 对输入数据进行 IQ2_XS 量化
size_t quantize_iq2_xs(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    (void)hist;
    // 断言每行数据的数量是 QK_K 的倍数
    GGML_ASSERT(n_per_row%QK_K == 0);
    // 计算每个块的数量
    int nblock = n_per_row/QK_K;
    // 将输出指针转换为字符指针
    char * qrow = (char *)dst;
    // 遍历每行数据
    for (int row = 0; row < nrow; ++row) {
        // 对每行数据进行 IQ2_XS 量化
        quantize_row_iq2_xs_impl(src, qrow, n_per_row, quant_weights);
        // 更新输入数据指针
        src += n_per_row;
        // 更新输出数据指针
        qrow += nblock*sizeof(block_iq2_xs);
    }
    # 返回 nrow * nblock * sizeof(block_iq2_xs) 的结果
    return nrow * nblock * sizeof(block_iq2_xs);
// 结构体定义，包含一个指向 uint32_t 类型的 grid 数组的指针，一个指向 int 类型的 map 数组的指针，一个指向 uint16_t 类型的 neighbours 数组的指针
typedef struct {
    uint32_t * grid;
    int      * map;
    uint16_t * neighbours;
} iq3_entry_t;

// 静态变量，存储一个 iq3_entry_t 结构体的数组，数组大小为 1
static iq3_entry_t iq3_data[1] = {
    {NULL, NULL, NULL},
};

// 内联函数，根据 grid_size 返回 iq3_data 数组的索引
static inline int iq3_data_index(int grid_size) {
    // 忽略参数 grid_size
    (void)grid_size;
    // 断言 grid_size 等于 256
    GGML_ASSERT(grid_size == 256);
    // 返回索引值 0
    return 0;
}

// 比较函数，用于 qsort 排序，比较两个指针指向的 int 数组
static int iq3_compare_func(const void * left, const void * right) {
    // 将 left 和 right 转换为 int 指针
    const int * l = (const int *)left;
    const int * r = (const int *)right;
    // 比较两个数组的第一个元素，若不相等则返回比较结果，若相等则比较第二个元素
    return l[0] < r[0] ? -1 : l[0] > r[0] ? 1 : l[1] < r[1] ? -1 : l[1] > r[1] ? 1 : 0;
}

// 初始化函数，根据 grid_size 初始化 iq3_data 数组中的数据
void iq3xs_init_impl(int grid_size) {
    // 获取 iq3_data 数组的索引
    const int gindex = iq3_data_index(grid_size);
    // 如果 iq3_data[gindex].grid 不为空，则直接返回
    if (iq3_data[gindex].grid) {
        return;
    }
}
    // 定义一个包含256个元素的常量数组，数组元素为uint16_t类型
    static const uint16_t kgrid_256[256] = {
            // 数组元素的值依次为0, 2, 4, ..., 3992
            0,     2,     4,     9,    11,    15,    16,    18,    25,    34,    59,    61,    65,    67,    72,    74,
            81,    85,    88,    90,    97,   108,   120,   128,   130,   132,   137,   144,   146,   153,   155,   159,
            169,   175,   189,   193,   199,   200,   202,   213,   248,   267,   287,   292,   303,   315,   317,   321,
            327,   346,   362,   413,   436,   456,   460,   462,   483,   497,   513,   515,   520,   522,   529,   531,
            536,   538,   540,   551,   552,   576,   578,   585,   592,   594,   641,   643,   648,   650,   657,   664,
            698,   704,   706,   720,   729,   742,   758,   769,   773,   808,   848,   852,   870,   889,   901,   978,
            992,  1024,  1026,  1033,  1035,  1040,  1042,  1046,  1049,  1058,  1089,  1091,  1093,  1096,  1098,  1105,
            1112,  1139,  1143,  1144,  1152,  1154,  1161,  1167,  1168,  1170,  1183,  1184,  1197,  1217,  1224,  1228,
            1272,  1276,  1309,  1323,  1347,  1367,  1377,  1404,  1473,  1475,  1486,  1509,  1537,  1544,  1546,  1553,
            1555,  1576,  1589,  1594,  1600,  1602,  1616,  1625,  1636,  1638,  1665,  1667,  1672,  1685,  1706,  1722,
            1737,  1755,  1816,  1831,  1850,  1856,  1862,  1874,  1901,  1932,  1950,  1971,  2011,  2032,  2052,  2063,
            2077,  2079,  2091,  2095,  2172,  2192,  2207,  2208,  2224,  2230,  2247,  2277,  2308,  2345,  2356,  2389,
            2403,  2424,  2501,  2504,  2506,  2520,  2570,  2593,  2616,  
    // 定义常量 kmap_size 为 4096
    const int kmap_size = 4096;
    // 定义常量 nwant 为 2
    const int nwant = 2;
    // 将指针 kgrid 指向 kgrid_256
    const uint16_t * kgrid = kgrid_256;
    // 声明指针 kgrid_q3xs
    uint32_t * kgrid_q3xs;
    // 声明指针 kmap_q3xs
    int      * kmap_q3xs;
    // 声明指针 kneighbors_q3xs

    // 打印函数名和 grid_size 的值
    printf("================================================================= %s(grid_size = %d)\n", __func__, grid_size);
    // 分配内存给 the_grid，大小为 grid_size 乘以 uint32_t 的大小
    uint32_t * the_grid = (uint32_t *)malloc(grid_size*sizeof(uint32_t));
    // 遍历 the_grid
    for (int k = 0; k < grid_size; ++k) {
        // 将 pos 指向 the_grid 的第 k 个元素
        int8_t * pos = (int8_t *)(the_grid + k);
        // 遍历 4 次
        for (int i = 0; i < 4; ++i) {
            // 计算 l 的值
            int l = (kgrid[k] >> 3*i) & 0x7;
            // 将 pos 的第 i 个元素赋值为 2*l + 1
            pos[i] = 2*l + 1;
        }
    }
    // 将 kgrid_q3xs 指向 the_grid
    kgrid_q3xs = the_grid;
    // 将 iq3_data[gindex].grid 指向 the_grid
    iq3_data[gindex].grid = the_grid;
    // 分配内存给 kmap_q3xs，大小为 kmap_size 乘以 int 的大小
    kmap_q3xs = (int *)malloc(kmap_size*sizeof(int));
    // 将 iq3_data[gindex].map 指向 kmap_q3xs
    iq3_data[gindex].map = kmap_q3xs;
    // 遍历 kmap_q3xs，将每个元素赋值为 -1
    for (int i = 0; i < kmap_size; ++i) kmap_q3xs[i] = -1;
    // 声明辅助变量 aux32 和 aux8
    uint32_t aux32;
    uint8_t * aux8 = (uint8_t *)&aux32;
    // 遍历 grid_size
    for (int i = 0; i < grid_size; ++i) {
        // 将 aux32 赋值为 kgrid_q3xs[i]
        aux32 = kgrid_q3xs[i];
        // 声明 index 为 0
        uint16_t index = 0;
        // 遍历 4 次
        for (int k=0; k<4; ++k) {
            // 计算 q 的值
            uint16_t q = (aux8[k] - 1)/2;
            // 更新 index
            index |= (q << 3*k);
        }
        // 将 kmap_q3xs[index] 赋值为 i
        kmap_q3xs[index] = i;
    }
    // 声明 pos 数组
    int8_t pos[4];
    // 分配内存给 dist2，大小为 2 倍的 grid_size 乘以 int 的大小
    int * dist2 = (int *)malloc(2*grid_size*sizeof(int));
    // 声明 num_neighbors 和 num_not_in_map 的初始值为 0
    int num_neighbors = 0, num_not_in_map = 0;
    // 遍历 kmap_q3xs 数组，计算不在地图中的元素个数
    for (int i = 0; i < kmap_size; ++i) {
        // 如果当前元素大于等于 0，则跳过
        if (kmap_q3xs[i] >= 0) continue;
        // 增加不在地图中的元素计数
        ++num_not_in_map;
        // 根据当前索引计算位置坐标
        for (int k = 0; k < 4; ++k) {
            int l = (i >> 3*k) & 0x7;
            pos[k] = 2*l + 1;
        }
        // 计算当前位置与所有网格点的距离平方
        for (int j = 0; j < grid_size; ++j) {
            const int8_t * pg = (const int8_t *)(kgrid_q3xs + j);
            int d2 = 0;
            for (int k = 0; k < 4; ++k) d2 += (pg[k] - pos[k])*(pg[k] - pos[k]);
            dist2[2*j+0] = d2;
            dist2[2*j+1] = j;
        }
        // 对距离平方进行排序
        qsort(dist2, grid_size, 2*sizeof(int), iq3_compare_func);
        int n = 0; int d2 = dist2[0];
        int nhave = 1;
        // 统计邻居数量
        for (int j = 0; j < grid_size; ++j) {
            if (dist2[2*j] > d2) {
                if (nhave == nwant) break;
                d2 = dist2[2*j];
                ++nhave;
            }
            ++n;
        }
        // 累加邻居数量
        num_neighbors += n;
    }
    // 打印函数名和总邻居数量
    printf("%s: %d neighbours in total\n", __func__, num_neighbors);
    // 分配内存给 kneighbors_q3xs 数组
    kneighbors_q3xs = (uint16_t *)malloc((num_neighbors + num_not_in_map)*sizeof(uint16_t));
    // 将 kneighbors_q3xs 数组赋值给 iq3_data[gindex].neighbours
    iq3_data[gindex].neighbours = kneighbors_q3xs;
    // 初始化计数器
    int counter = 0;
    // 遍历 kmap_q3xs 数组，如果当前元素大于等于0，则跳过
    for (int i = 0; i < kmap_size; ++i) {
        if (kmap_q3xs[i] >= 0) continue;
        // 根据当前索引计算出 pos 数组的值
        for (int k = 0; k < 4; ++k) {
            int l = (i >> 3*k) & 0x7;
            pos[k] = 2*l + 1;
        }
        // 遍历 grid_size，计算每个点到 pos 的距离平方，并存储到 dist2 数组中
        for (int j = 0; j < grid_size; ++j) {
            const int8_t * pg = (const int8_t *)(kgrid_q3xs + j);
            int d2 = 0;
            for (int k = 0; k < 4; ++k) d2 += (pg[k] - pos[k])*(pg[k] - pos[k]);
            dist2[2*j+0] = d2;
            dist2[2*j+1] = j;
        }
        // 对 dist2 数组进行排序
        qsort(dist2, grid_size, 2*sizeof(int), iq3_compare_func);
        // 更新 kmap_q3xs 数组的值，并初始化一些变量
        kmap_q3xs[i] = -(counter + 1);
        int d2 = dist2[0];
        uint16_t * start = &kneighbors_q3xs[counter++];
        int n = 0, nhave = 1;
        // 遍历 grid_size，根据条件选择邻居点，并存储到 kneighbors_q3xs 数组中
        for (int j = 0; j < grid_size; ++j) {
            if (dist2[2*j] > d2) {
                if (nhave == nwant) break;
                d2 = dist2[2*j];
                ++nhave;
            }
            kneighbors_q3xs[counter++] = dist2[2*j+1];
            ++n;
        }
        // 更新 start 指向的值
        *start = n;
    }
    // 释放 dist2 数组的内存
    free(dist2);
// 释放 IQ3 数据结构中指定网格大小的内存，确保网格大小为 256
void iq3xs_free_impl(int grid_size) {
    GGML_ASSERT(grid_size == 256);
    // 获取 IQ3 数据结构中指定网格大小的索引
    const int gindex = iq3_data_index(grid_size);
    // 如果 IQ3 数据结构中指定网格大小的网格存在，则释放网格、地图和邻居数据，并将其指针置为空
    if (iq3_data[gindex].grid) {
        free(iq3_data[gindex].grid);       iq3_data[gindex].grid = NULL;
        free(iq3_data[gindex].map);        iq3_data[gindex].map  = NULL;
        free(iq3_data[gindex].neighbours); iq3_data[gindex].neighbours = NULL;
    }
}

// 在指定邻居中找到最佳邻居，根据权重计算距离
static int iq3_find_best_neighbour(const uint16_t * restrict neighbours, const uint32_t * restrict grid,
        const float * restrict xval, const float * restrict weight, float scale, int8_t * restrict L) {
    // 获取邻居数量
    int num_neighbors = neighbours[0];
    GGML_ASSERT(num_neighbors > 0);
    // 初始化最佳距离为最大值，网格索引为 -1
    float best_d2 = FLT_MAX;
    int grid_index = -1;
    // 遍历邻居
    for (int j = 1; j <= num_neighbors; ++j) {
        // 获取当前邻居的指针
        const int8_t * pg = (const int8_t *)(grid + neighbours[j]);
        float d2 = 0;
        // 计算距离
        for (int i = 0; i < 4; ++i) {
            float q = pg[i];
            float diff = scale*q - xval[i];
            d2 += weight[i]*diff*diff;
        }
        // 更新最佳距离和网格索引
        if (d2 < best_d2) {
            best_d2 = d2; grid_index = neighbours[j];
        }
    }
    GGML_ASSERT(grid_index >= 0);
    // 获取最佳邻居的指针，并根据计算结果更新 L 数组
    const int8_t * pg = (const int8_t *)(grid + grid_index);
    for (int i = 0; i < 4; ++i) L[i] = (pg[i] - 1)/2;
    return grid_index;
}

// 对输入数据进行 IQ3_XXS 算法的行量化
static void quantize_row_iq3_xxs_impl(const float * restrict x, void * restrict vy, int n, const float * restrict quant_weights) {

    // 获取 IQ3 数据结构中指定网格大小的索引
    const int gindex = iq3_data_index(256);

    // 获取 IQ3 数据结构中指定网格大小的网格、地图和邻居数据的指针
    const uint32_t * kgrid_q3xs      = iq3_data[gindex].grid;
    const int      * kmap_q3xs       = iq3_data[gindex].map;
    const uint16_t * kneighbors_q3xs = iq3_data[gindex].neighbours;

    // 确保量化权重、网格、地图和邻居数据存在，并且输入数据长度是 QK_K 的倍数
    //GGML_ASSERT(quant_weights   && "missing quantization weights");
    GGML_ASSERT(kgrid_q3xs      && "forgot to call ggml_quantize_init()?");
    GGML_ASSERT(kmap_q3xs       && "forgot to call ggml_quantize_init()?");
    GGML_ASSERT(kneighbors_q3xs && "forgot to call ggml_quantize_init()?");
    GGML_ASSERT(n%QK_K == 0);
}
    // 定义常量 kMaxQ，表示最大的 Q 值为 8
    const int kMaxQ = 8;

    // 计算 n 的 1/256 值，存储在 nbl 中
    const int nbl = n/256;

    // 定义指针 y，指向 block_iq3_xxs 结构体的数组 vy
    block_iq3_xxs * y = vy;

    // 定义存储 QK_K/32 个浮点数的数组 scales
    float scales[QK_K/32];
    
    // 定义存储 32 个浮点数的数组 weight
    float weight[32];
    
    // 定义存储 32 个浮点数的数组 xval
    float xval[32];
    
    // 定义存储 32 个 int8_t 类型的整数的数组 L
    int8_t L[32];
    
    // 定义存储 32 个 int8_t 类型的整数的数组 Laux
    int8_t Laux[32];
    
    // 定义存储 32 个浮点数的数组 waux
    float waux[32];
    
    // 定义存储 8 个布尔值的数组 is_on_grid
    bool is_on_grid[8];
    
    // 定义存储 8 个布尔值的数组 is_on_grid_aux
    bool is_on_grid_aux[8];
    
    // 定义存储 8 个 uint8_t 类型的整数的数组 block_signs
    uint8_t block_signs[8];
    
    // 定义存储 3*(QK_K/8) 个 uint8_t 类型的整数的数组 q3
    uint8_t q3[3*(QK_K/8)];
    
    // 定义指针 scales_and_signs，指向 q3 + QK_K/4 的位置
    uint32_t * scales_and_signs = (uint32_t *)(q3 + QK_K/4);
# 对输入的浮点数数组进行 IQ3 压缩，将结果存储到目标数组中，并返回压缩后的数据大小
size_t quantize_iq3_xxs(const float * src, void * dst, int nrow, int n_per_row, int64_t * hist, const float * quant_weights) {
    # 忽略未使用的参数 hist
    (void)hist;
    # 断言每行元素个数能够被 QK_K 整除
    GGML_ASSERT(n_per_row%QK_K == 0);
    # 计算每个块的数量
    int nblock = n_per_row/QK_K;
    # 将目标数组转换为字符指针
    char * qrow = (char *)dst;
    # 遍历每一行
    for (int row = 0; row < nrow; ++row) {
        # 对当前行进行 IQ3 压缩
        quantize_row_iq3_xxs_impl(src, qrow, n_per_row, quant_weights);
        # 更新源数组指针，指向下一行的起始位置
        src += n_per_row;
        # 更新目标数组指针，指向下一个块的起始位置
        qrow += nblock*sizeof(block_iq3_xxs);
    }
    # 返回压缩后的数据总大小
    return nrow * nblock * sizeof(block_iq3_xxs);
}

# 对单行数据进行 IQ3 压缩
void quantize_row_iq3_xxs(const float * restrict x, void * restrict vy, int k) {
    # 断言每行元素个数能够被 QK_K 整除
    assert(k % QK_K == 0);
    # 将目标数组转换为 block_iq3_xxs 指针
    block_iq3_xxs * restrict y = vy;
    # 调用参考实现函数对单行数据进行 IQ3 压缩
    quantize_row_iq3_xxs_reference(x, y, k);
}

# 参考实现函数，对单行数据进行 IQ3 压缩
void quantize_row_iq3_xxs_reference(const float * restrict x, block_iq3_xxs * restrict y, int k) {
    # 断言每行元素个数能够被 QK_K 整除
    assert(k % QK_K == 0);
    # 调用实现函数对单行数据进行 IQ3 压缩
    quantize_row_iq3_xxs_impl(x, y, k, NULL);
}
```