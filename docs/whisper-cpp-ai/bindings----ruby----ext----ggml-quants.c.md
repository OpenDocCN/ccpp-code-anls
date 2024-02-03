# `whisper.cpp\bindings\ruby\ext\ggml-quants.c`

```cpp
// 包含头文件 ggml-quants.h 和 ggml-impl.h
#include "ggml-quants.h"
#include "ggml-impl.h"

// 包含数学、字符串、断言和浮点数处理的标准库头文件
#include <math.h>
#include <string.h>
#include <assert.h>
#include <float.h>

#ifdef __ARM_NEON

// 如果 YCM 找不到 <arm_neon.h>，需要创建符号链接到该文件
// 例如：$ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
#include <arm_neon.h>

#if !defined(__aarch64__)
// 定义一个函数，将 int16x8_t 向量中的 16 位整数相加并返回 32 位整数结果
inline static int32_t vaddvq_s16(int16x8_t v) {
    return
        (int32_t)vgetq_lane_s16(v, 0) + (int32_t)vgetq_lane_s16(v, 1) +
        (int32_t)vgetq_lane_s16(v, 2) + (int32_t)vgetq_lane_s16(v, 3) +
        (int32_t)vgetq_lane_s16(v, 4) + (int32_t)vgetq_lane_s16(v, 5) +
        (int32_t)vgetq_lane_s16(v, 6) + (int32_t)vgetq_lane_s16(v, 7);
}

// 定义一个函数，将两个 int16x8_t 向量中的 16 位整数对应位置相加并返回结果向量
inline static int16x8_t vpaddq_s16(int16x8_t a, int16x8_t b) {
    int16x4_t a0 = vpadd_s16(vget_low_s16(a), vget_high_s16(a));
    int16x4_t b0 = vpadd_s16(vget_low_s16(b), vget_high_s16(b));
    return vcombine_s16(a0, b0);
}

// 定义一个函数，将 int32x4_t 向量中的 32 位整数相加并返回结果
inline static int32_t vaddvq_s32(int32x4_t v) {
    return vgetq_lane_s32(v, 0) + vgetq_lane_s32(v, 1) + vgetq_lane_s32(v, 2) + vgetq_lane_s32(v, 3);
}
#endif

#else

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
#if !defined(__riscv) && !defined(__s390__)
#include <immintrin.h>
#endif
#endif
#endif
#endif
#endif

#ifdef __riscv_v_intrinsic
#include <riscv_vector.h>
#endif

// 取消宏定义 MIN 和 MAX，重新定义 MIN 和 MAX 宏
#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义宏 MM256_SET_M128I，用于将两个 __m128i 向量合并成一个 __m256i 向量
#define MM256_SET_M128I(a, b) _mm256_insertf128_si256(_mm256_castsi128_si256(b), (a), 1)

#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)
// 定义一个函数，将两个 __m128i 向量中的 8 位整数相乘并返回结果向量，然后将结果向量中的元素两两相加
static inline __m128i mul_sum_i8_pairs(const __m128i x, const __m128i y) {
    // 获取 x 向量中元素的绝对值
    const __m128i ax = _mm_sign_epi8(x, x);
    // 对 y 向量的值进行符号化处理
    const __m128i sy = _mm_sign_epi8(y, x);
    // 执行乘法并创建16位值
    const __m128i dot = _mm_maddubs_epi16(ax, sy);
    // 创建全为1的16位整数向量
    const __m128i ones = _mm_set1_epi16(1);
    // 返回 ones 和 dot 的乘积结果
    return _mm_madd_epi16(ones, dot);
// 关闭大括号
}

// 如果支持 AVX、AVX2 或 AVX512F，则定义以下函数

// 水平加和 8 个浮点数
static inline float hsum_float_8(const __m256 x) {
    // 从 x 中提取第二个 128 位的数据
    __m128 res = _mm256_extractf128_ps(x, 1);
    // 将 x 和 res 相加
    res = _mm_add_ps(res, _mm256_castps256_ps128(x));
    // 将 res 和 res 的高低位相加
    res = _mm_add_ps(res, _mm_movehl_ps(res, res));
    // 将 res 和 res 的高位复制到低位，再相加
    res = _mm_add_ss(res, _mm_movehdup_ps(res));
    // 将结果转换为 float 类型并返回
    return _mm_cvtss_f32(res);
}

// 水平加和 8 个 int32_t
static inline int hsum_i32_8(const __m256i a) {
    // 将 a 拆分为两个 128 位的数据，然后相加
    const __m128i sum128 = _mm_add_epi32(_mm256_castsi256_si128(a), _mm256_extractf128_si256(a, 1));
    // 将 sum128 的高低位相加
    const __m128i hi64 = _mm_unpackhi_epi64(sum128, sum128);
    // 将 hi64 和 sum128 相加
    const __m128i sum64 = _mm_add_epi32(hi64, sum128);
    // 将 sum64 的高低位相加
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    // 将结果转换为 int 类型并返回
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

// 水平加和 4 个 int32_t
static inline int hsum_i32_4(const __m128i a) {
    // 将 a 的高低位相加
    const __m128i hi64 = _mm_unpackhi_epi64(a, a);
    // 将 hi64 和 a 相加
    const __m128i sum64 = _mm_add_epi32(hi64, a);
    // 将 sum64 的高低位相加
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    // 将结果转换为 int 类型并返回
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

// 如果支持 AVX2 或 AVX512F，则定义以下函数

// 将 32 位数据扩展为 32 字节 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    uint32_t x32;
    // 将 x 的内容复制到 x32
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于重排的掩码
    const __m256i shuf_mask = _mm256_set_epi64x(
            0x0303030303030303, 0x0202020202020202,
            0x0101010101010101, 0x0000000000000000);
    // 使用掩码将 x32 重排为 bytes
    __m256i bytes = _mm256_shuffle_epi8(_mm256_set1_epi32(x32), shuf_mask);
    // 创建用于位操作的掩码
    const __m256i bit_mask = _mm256_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 对 bytes 进行位或操作
    bytes = _mm256_or_si256(bytes, bit_mask);
    // 比较 bytes 和 -1，返回结果
    return _mm256_cmpeq_epi8(bytes, _mm256_set1_epi64x(-1));
}

// 将 32 个 4 位字段解包为 32 字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从 rsi 加载 128 位数据
    const __m128i tmp = _mm_loadu_si128((const __m128i *)rsi);
    // 将两个 128 位整数拼接成一个 256 位整数
    const __m256i bytes = MM256_SET_M128I(_mm_srli_epi16(tmp, 4), tmp);
    // 创建一个 256 位整数，每个字节都是 0xF
    const __m256i lowMask = _mm256_set1_epi8( 0xF );
    // 对两个 256 位整数进行按位与操作，返回结果
    return _mm256_and_si256(lowMask, bytes);
}

// add int16_t pairwise and return as float vector
static inline __m256 sum_i16_pairs_float(const __m256i x) {
    // 创建一个包含全1的__m256i对象
    const __m256i ones = _mm256_set1_epi16(1);
    // 将输入向量x中的每两个16位整数相加，并返回结果向量
    const __m256i summed_pairs = _mm256_madd_epi16(ones, x);
    // 将结果向量转换为浮点数向量并返回
    return _mm256_cvtepi32_ps(summed_pairs);
}

static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
#if __AVXVNNI__
    // 创建一个全0的__m256i对象
    const __m256i zero = _mm256_setzero_si256();
    // 使用无符号8位整数乘法求和指令计算两个输入向量的点积，并返回结果向量
    const __m256i summed_pairs = _mm256_dpbusd_epi32(zero, ax, sy);
    // 将结果向量转换为浮点数向量并返回
    return _mm256_cvtepi32_ps(summed_pairs);
#else
    // 执行乘法并创建16位值
    const __m256i dot = _mm256_maddubs_epi16(ax, sy);
    // 调用sum_i16_pairs_float函数对结果向量进行处理并返回
    return sum_i16_pairs_float(dot);
#endif
}

// multiply int8_t, add results pairwise twice and return as float vector
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
#if __AVXVNNIINT8__
    // 创建一个全0的__m256i对象
    const __m256i zero = _mm256_setzero_si256();
    // 使用有符号8位整数乘法求和指令计算两个输入向量的点积，并返回结果向量
    const __m256i summed_pairs = _mm256_dpbssd_epi32(zero, x, y);
    // 将结果向量转换为浮点数向量并返回
    return _mm256_cvtepi32_ps(summed_pairs);
#else
    // 获取x向量的绝对值
    const __m256i ax = _mm256_sign_epi8(x, x);
    // 对y向量的值进行符号化
    const __m256i sy = _mm256_sign_epi8(y, x);
    // 调用mul_sum_us8_pairs_float函数对结果向量进行处理并返回
    return mul_sum_us8_pairs_float(ax, sy);
#endif
}

static inline __m128i packNibbles( __m256i bytes )
{
    // 将16位通道内的位移动，从0000_abcd_0000_efgh移动到0000_0000_abcd_efgh
#if __AVX512F__
    // 将输入向量中的每个16位整数向右逻辑移位4位
    const __m256i bytes_srli_4 = _mm256_srli_epi16(bytes, 4);   // 0000_0000_abcd_0000
    // 将两个向量按位或操作
    bytes = _mm256_or_si256(bytes, bytes_srli_4);               // 0000_abcd_abcd_efgh
    // 将结果向量转换为128位整数向量并返回
    return _mm256_cvtepi16_epi8(bytes);                         // abcd_efgh
#else
    // 创建一个全1的16位整数向量
    const __m256i lowByte = _mm256_set1_epi16( 0xFF );
    // 对输入向量进行按位非操作
    __m256i high = _mm256_andnot_si256( lowByte, bytes );
    // 对输入向量进行按位与操作
    __m256i low = _mm256_and_si256( lowByte, bytes );
    // 将高位向右逻辑移位4位
    high = _mm256_srli_epi16( high, 4 );
    // 将低位和高位进行按位或操作
    bytes = _mm256_or_si256( low, high );

    // 压缩uint16_t通道为字节
    # 将 AVX 寄存器中的 256 位整数转换为两个 128 位整数
    __m128i r0 = _mm256_castsi256_si128( bytes );
    # 从 AVX 寄存器中提取第二个 128 位整数
    __m128i r1 = _mm256_extracti128_si256( bytes, 1 );
    # 将两个 128 位整数打包成一个 128 位整数，无符号饱和转换
    return _mm_packus_epi16( r0, r1 );
#endif
}
#elif defined(__AVX__)
// 如果定义了 AVX，则执行以下代码块

// 将 32 位扩展为 32 字节 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    // 从 x 中复制 32 位到 x32
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于位移的掩码
    const __m128i shuf_maskl = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
    const __m128i shuf_maskh = _mm_set_epi64x(0x0303030303030303, 0x0202020202020202);
    // 使用掩码对 x32 进行位移
    __m128i bytesl = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskl);
    __m128i bytesh = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskh);
    // 创建位掩码
    const __m128i bit_mask = _mm_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 对 bytesl 和 bytesh 进行位或运算
    bytesl = _mm_or_si128(bytesl, bit_mask);
    bytesh = _mm_or_si128(bytesh, bit_mask);
    // 比较 bytesl 和 bytesh 是否等于 -1
    bytesl = _mm_cmpeq_epi8(bytesl, _mm_set1_epi64x(-1));
    bytesh = _mm_cmpeq_epi8(bytesh, _mm_set1_epi64x(-1));
    // 返回结果
    return MM256_SET_M128I(bytesh, bytesl);
}

// 将 32 个 4 位字段解包成 32 字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从内存加载 16 个字节
    __m128i tmpl = _mm_loadu_si128((const __m128i *)rsi);
    __m128i tmph = _mm_srli_epi16(tmpl, 4);
    const __m128i lowMask = _mm_set1_epi8(0xF);
    tmpl = _mm_and_si128(lowMask, tmpl);
    tmph = _mm_and_si128(lowMask, tmph);
    // 返回结果
    return MM256_SET_M128I(tmph, tmpl);
}

// 将 int16_t 两两相加并作为浮点向量返回
static inline __m256 sum_i16_pairs_float(const __m128i xh, const __m128i xl) {
    const __m128i ones = _mm_set1_epi16(1);
    const __m128i summed_pairsl = _mm_madd_epi16(ones, xl);
    const __m128i summed_pairsh = _mm_madd_epi16(ones, xh);
    const __m256i summed_pairs = MM256_SET_M128I(summed_pairsh, summed_pairsl);
    return _mm256_cvtepi32_ps(summed_pairs);
}

// 将 us8 两两相乘并相加，作为浮点向量返回
static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
    const __m128i axl = _mm256_castsi256_si128(ax);
    const __m128i axh = _mm256_extractf128_si256(ax, 1);
    const __m128i syl = _mm256_castsi256_si128(sy);
    # 从 AVX 寄存器 sy 中提取高 128 位作为 __m128i 类型的变量 syh
    const __m128i syh = _mm256_extractf128_si256(sy, 1);
    # 执行乘法操作并创建 16 位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    # 调用函数 sum_i16_pairs_float 处理 doth 和 dotl，并返回结果
    return sum_i16_pairs_float(doth, dotl);
// multiply int8_t, add results pairwise twice and return as float vector
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
    // 将 x 向量拆分为低 128 位和高 128 位
    const __m128i xl = _mm256_castsi256_si128(x);
    const __m128i xh = _mm256_extractf128_si256(x, 1);
    // 将 y 向量拆分为低 128 位和高 128 位
    const __m128i yl = _mm256_castsi256_si128(y);
    const __m128i yh = _mm256_extractf128_si256(y, 1);
    // 获取 x 向量的绝对值
    const __m128i axl = _mm_sign_epi8(xl, xl);
    const __m128i axh = _mm_sign_epi8(xh, xh);
    // 对 y 向量的值进行符号化
    const __m128i syl = _mm_sign_epi8(yl, xl);
    const __m128i syh = _mm_sign_epi8(yh, xh);
    // 执行乘法并创建 16 位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    return sum_i16_pairs_float(doth, dotl);
}

static inline __m128i packNibbles( __m128i bytes1, __m128i bytes2 )
{
    // 将 16 位 lanes 中的位从 0000_abcd_0000_efgh 移动到 0000_0000_abcd_efgh
    const __m128i lowByte = _mm_set1_epi16( 0xFF );
    __m128i high = _mm_andnot_si128( lowByte, bytes1 );
    __m128i low = _mm_and_si128( lowByte, bytes1 );
    high = _mm_srli_epi16( high, 4 );
    bytes1 = _mm_or_si128( low, high );
    high = _mm_andnot_si128( lowByte, bytes2 );
    low = _mm_and_si128( lowByte, bytes2 );
    high = _mm_srli_epi16( high, 4 );
    bytes2 = _mm_or_si128( low, high );

    return _mm_packus_epi16( bytes1, bytes2);
}
#endif
#elif defined(__SSSE3__)
// horizontally add 4x4 floats
static inline float hsum_float_4x4(const __m128 a, const __m128 b, const __m128 c, const __m128 d) {
    // 水平相加 4x4 浮点数
    __m128 res_0 =_mm_hadd_ps(a, b);
    __m128 res_1 =_mm_hadd_ps(c, d);
    __m128 res =_mm_hadd_ps(res_0, res_1);
    res =_mm_hadd_ps(res, res);
    res =_mm_hadd_ps(res, res);

    return _mm_cvtss_f32(res);
}
#endif // __AVX__ || __AVX2__ || __AVX512F__
#endif // defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)
#if defined(__ARM_NEON)  // 如果定义了 ARM NEON

#if !defined(__aarch64__)  // 如果未定义 ARM 64 位

// 计算 int32x4_t 类型向量的和
inline static int32_t vaddvq_s32(int32x4_t v) {
    return vgetq_lane_s32(v, 0) + vgetq_lane_s32(v, 1) + vgetq_lane_s32(v, 2) + vgetq_lane_s32(v, 3);
}

// 计算 float32x4_t 类型向量的和
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}

// 计算 float32x4_t 类型向量的最大值
inline static float vmaxvq_f32(float32x4_t v) {
    return
        MAX(MAX(vgetq_lane_f32(v, 0), vgetq_lane_f32(v, 1)),
            MAX(vgetq_lane_f32(v, 2), vgetq_lane_f32(v, 3)));
}

// 将 float32x4_t 类型向量转换为 int32x4_t 类型向量
inline static int32x4_t vcvtnq_s32_f32(float32x4_t v) {
    int32x4_t res;

    res[0] = roundf(vgetq_lane_f32(v, 0));
    res[1] = roundf(vgetq_lane_f32(v, 1));
    res[2] = roundf(vgetq_lane_f32(v, 2));
    res[3] = roundf(vgetq_lane_f32(v, 3));

    return res;
}

#endif
#endif

#if defined(__ARM_NEON) || defined(__wasm_simd128__)  // 如果定义了 ARM NEON 或者 wasm SIMD128

// 定义宏用于生成扩展 8 位到 8 字节的预计算表
#define B1(c,s,n)  0x ## n ## c ,  0x ## n ## s
#define B2(c,s,n) B1(c,s,n ## c), B1(c,s,n ## s)
#define B3(c,s,n) B2(c,s,n ## c), B2(c,s,n ## s)
#define B4(c,s,n) B3(c,s,n ## c), B3(c,s,n ## s)
#define B5(c,s,n) B4(c,s,n ## c), B4(c,s,n ## s)
#define B6(c,s,n) B5(c,s,n ## c), B5(c,s,n ## s)
#define B7(c,s,n) B6(c,s,n ## c), B6(c,s,n ## s)
#define B8(c,s  ) B7(c,s,     c), B7(c,s,     s)

// 预计算表，用于将 8 位扩展为 8 字节
static const uint64_t table_b2b_0[1 << 8] = { B8(00, 10) }; // ( b) << 4
static const uint64_t table_b2b_1[1 << 8] = { B8(10, 00) }; // (!b) << 4
#endif

// 参考实现，用于确定性地创建模型文件
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k) {
    static const int qk = QK4_0;

    assert(k % qk == 0);

    const int nb = k / qk;
    // 遍历 nb 次，处理每个输入向量
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值
        float max  = 0.0f;

        // 遍历 qk 次，处理每个输入向量的元素
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];
            // 更新绝对值最大值和最大值
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算缩放因子
        const float d  = max / -8;
        const float id = d ? 1.0f/d : 0.0f;

        // 将缩放因子转换为 FP16 格式，并存储到输出向量中
        y[i].d = GGML_FP32_TO_FP16(d);

        // 遍历 qk/2 次，处理每个输出向量的元素
        for (int j = 0; j < qk/2; ++j) {
            // 计算归一化后的输入值
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            // 将归一化后的输入值转换为 uint8_t 类型，并存储到输出向量中
            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 8.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 8.5f));

            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k) {
    // 调用参考实现 quantize_row_q4_0_reference 进行量化
    quantize_row_q4_0_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k) {
    // 定义量化因子 qk
    const int qk = QK4_1;

    // 断言输入数组长度 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算结构体数组 y 的长度 nb
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

        // 计算间隔 d 和其倒数 id
        const float d  = (max - min) / ((1 << 4) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 和 min 转换为 FP16 存储在结构体中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 遍历每个结构体中的元素，进行量化
        for (int j = 0; j < qk/2; ++j) {
            // 计算归一化后的值 x0 和 x1
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            // 将归一化后的值转换为 uint8_t 类型并存储在结构体中
            const uint8_t xi0 = MIN(15, (int8_t)(x0 + 0.5f));
            const uint8_t xi1 = MIN(15, (int8_t)(x1 + 0.5f));

            y[i].qs[j]  = xi0;
            y[i].qs[j] |= xi1 << 4;
        }
    }
}

// 将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k) {
    // 调用参考实现 quantize_row_q4_1_reference 进行量化
    quantize_row_q4_1_reference(x, y, k);
}

// 参考实现，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k) {
    // 定义静态量化因子 qk
    static const int qk = QK5_0;

    // 断言输入数组长度 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算结构体数组 y 的长度 nb
    const int nb = k / qk;
    // 遍历 nb 次，处理每个输入向量
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值
        float max  = 0.0f;

        // 遍历 qk 次，处理每个输入向量的元素
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];
            // 更新绝对值最大值和最大值
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算 d 和 id
        const float d  = max / -16;
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 格式并存储在 y[i].d 中
        y[i].d = GGML_FP32_TO_FP16(d);

        uint32_t qh = 0;

        // 遍历 qk/2 次，处理每个输入向量的一半元素
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1，并归一化
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            // 将 x0 和 x1 量化为 5 位整数
            const uint8_t xi0 = MIN(31, (int8_t)(x0 + 16.5f));
            const uint8_t xi1 = MIN(31, (int8_t)(x1 + 16.5f));

            // 将 xi0 和 xi1 合并为一个字节并存储在 y[i].qs[j] 中
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第 5 位并将其存储在 qh 的正确位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到 y[i].qh 中
        memcpy(&y[i].qh, &qh, sizeof(qh));
    }
// 将输入的浮点数组 x 进行量化处理，结果存储在 y 中，每个块的大小为 k
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k) {
    // 调用参考实现 quantize_row_q5_0_reference 进行量化处理
    quantize_row_q5_0_reference(x, y, k);
}

// 参考实现，将输入的浮点数组 x 进行量化处理，结果存储在 y 中，每个块的大小为 k
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k) {
    // 定义量化因子 qk 为 QK5_1
    const int qk = QK5_1;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算块的数量 nb
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        // 遍历每个块中的元素
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];

            // 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 计算间隔 d 和其倒数 id
        const float d  = (max - min) / ((1 << 5) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 和 min 转换为半精度浮点数存储在 y 中
        y[i].d = GGML_FP32_TO_FP16(d);
        y[i].m = GGML_FP32_TO_FP16(min);

        // 初始化 qh 为 0
        uint32_t qh = 0;

        // 遍历每个块的一半
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1 的量化值
            const float x0 = (x[i*qk + 0    + j] - min)*id;
            const float x1 = (x[i*qk + qk/2 + j] - min)*id;

            const uint8_t xi0 = (uint8_t)(x0 + 0.5f);
            const uint8_t xi1 = (uint8_t)(x1 + 0.5f);

            // 将 x0 和 x1 的量化值存储在 y 中
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取第 5 位并将其存储在 qh 的正确位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到 y[i].qh 中
        memcpy(&y[i].qh, &qh, sizeof(y[i].qh));
    }
}

// 将输入的浮点数组 x 进行量化处理，结果存储在 y 中，每个块的大小为 k
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k) {
    // 调用参考实现 quantize_row_q5_1_reference 进行量化处理
    quantize_row_q5_1_reference(x, y, k);
}

// 参考实现，用于确定性地创建模型文件
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k) {
    // 断言 k 能够被 QK8_0 整除
    assert(k % QK8_0 == 0);
    // 计算块的数量 nb
    const int nb = k / QK8_0;
    // 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 初始化绝对值最大值为 0
        float amax = 0.0f; // absolute max

        // 遍历 QK8_0 次，j 从 0 到 QK8_0-1
        for (int j = 0; j < QK8_0; j++) {
            // 获取 x[i*QK8_0 + j] 的值
            const float v = x[i*QK8_0 + j];
            // 更新绝对值最大值为 v 和当前绝对值最大值的较大值
            amax = MAX(amax, fabsf(v));
        }

        // 计算 d 为绝对值最大值除以 (1 << 7) - 1
        const float d = amax / ((1 << 7) - 1);
        // 计算 id 为 d 不为 0 时为 1/d，否则为 0
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 格式后存入 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 再次遍历 QK8_0 次，j 从 0 到 QK8_0-1
        for (int j = 0; j < QK8_0; ++j) {
            // 计算 x0 为 x[i*QK8_0 + j] 乘以 id
            const float x0 = x[i*QK8_0 + j]*id;
            // 将 x0 四舍五入后存入 y[i].qs[j]
            y[i].qs[j] = roundf(x0);
        }
    }
// 量化一行数据为 QK8_0 类型，其中 QK8_0 为 32
void quantize_row_q8_0(const float * restrict x, void * restrict vy, int k) {
    // 断言 QK8_0 的值为 32
    assert(QK8_0 == 32);
    // 断言 k 能够整除 QK8_0
    assert(k % QK8_0 == 0);
    // 计算分块数量
    const int nb = k / QK8_0;

    // 将 vy 强制转换为 block_q8_0 类型的指针
    block_q8_0 * restrict y = vy;

#if defined(__ARM_NEON)
    // 使用 NEON 指令集进行量化计算
    for (int i = 0; i < nb; i++) {
        // 定义存储数据的向量数组
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 从输入数据 x 中加载数据到向量数组 srcv
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算 srcv 中数据的绝对值并存储到 asrcv
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算每两个元素的最大值并存储到 amaxv
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算最大值并存储到 amax
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算量化因子 d 和其倒数 id
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换为 FP16 类型并存储到 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 对每个元素进行量化计算并存储到 y[i].qs
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
    // 循环处理每个数据块
    for (int i = 0; i < nb; i++) {
        // 定义并初始化源向量数组、绝对值向量数组和最大值向量数组
        v128_t srcv [8];
        v128_t asrcv[8];
        v128_t amaxv[8];

        // 加载源向量数据到源向量数组
        for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
        // 计算源向量数组的绝对值并存储到绝对值向量数组
        for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

        // 计算绝对值向量数组中每两个元素的最大值并存储到最大值向量数组
        for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
        // 计算最大值向量数组中每两个元素的最大值并存储到最大值向量数组
        for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
        // 计算最大值向量数组中每两个元素的最大值并存储到最大值向量数组
        for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

        // 计算最大值向量数组中的最大值
        const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                                   wasm_f32x4_extract_lane(amaxv[0], 1)),
                               MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                                   wasm_f32x4_extract_lane(amaxv[0], 3)));

        // 计算比例因子和倒数
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将比例因子转换为 FP16 格式并存储到输出数组
        y[i].d = GGML_FP32_TO_FP16(d);

        // 对每个源向量进行量化并存储到输出数组
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
    // 如果支持 AVX2 或 AVX 指令集
    for (int i = 0; i < nb; i++) {
        // 将元素加载到4个 AVX 向量中
        __m256 v0 = _mm256_loadu_ps( x );
        __m256 v1 = _mm256_loadu_ps( x + 8 );
        __m256 v2 = _mm256_loadu_ps( x + 16 );
        __m256 v3 = _mm256_loadu_ps( x + 24 );
        x += 32;

        // 计算块的 max(abs(e))
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
}

// 对输入的量化数据进行反量化操作，将结果存储在浮点数数组中
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK8_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算块数 nb
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化数据转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个量化值，进行反量化操作
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
static float make_qx_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, int rmse_type) {
    float max = 0;
    float amax = 0;
    // 找到输入数组中的最大值
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果最大值接近零，则所有量化值设为零
    if (amax < 1e-30f) {
        for (int i = 0; i < n; ++i) {
            L[i] = 0;
        }
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    if (rmse_type == 0) {
        // 根据缩放因子将输入数据量化，并存储在 L 中
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
    // 根据权重类型计算量化值，并计算误差
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
    if (return_early) return suml2 > 0 ? 0.5f*(scale + 1/iscale) : 1/iscale;
    float best = scale * sumlx;
}
    // 循环遍历 is 变量从 -9 到 9
    for (int is = -9; is <= 9; ++is) {
        // 如果 is 等于 0，则跳过当前循环
        if (is == 0) {
            continue;
        }
        // 计算 iscale 值
        iscale = -(nmax + 0.1f*is) / max;
        // 初始化 sumlx 和 suml2 变量
        sumlx = suml2 = 0;
        // 循环遍历 i 变量从 0 到 n
        for (int i = 0; i < n; ++i) {
            // 计算最接近 iscale * x[i] 的整数值
            int l = nearest_int(iscale * x[i]);
            // 将 l 限制在 -nmax 和 nmax-1 之间
            l = MAX(-nmax, MIN(nmax-1, l));
            // 根据 weight_type 的值选择权重计算方式
            float w = weight_type == 1 ? x[i] * x[i] : 1;
            // 更新 sumlx 和 suml2 变量
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        // 如果 suml2 大于 0 且 sumlx*sumlx 大于 best*suml2
        if (suml2 > 0 && sumlx*sumlx > best*suml2) {
            // 更新 L 数组的值
            for (int i = 0; i < n; ++i) {
                int l = nearest_int(iscale * x[i]);
                L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
            }
            // 更新 scale 和 best 变量的值
            scale = sumlx/suml2; best = scale*sumlx;
        }
    }
    // 返回 scale 变量的值
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
    // 如果最大值等于最小值，将所有元素置为0，更新最小值为0，返回0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = 0;
        return 0.f;
    }
    // 如果最小值大于0，将最小值置为0
    if (min > 0) min = 0;
    // 计算缩放比例
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    // 迭代ntry次
    for (int itry = 0; itry < ntry; ++itry) {
        float sumlx = 0; int suml2 = 0;
        bool did_change = false;
        // 遍历数组，更新L数组，计算sumlx和suml2
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
        // 更新scale
        scale = sumlx/suml2;
        float sum = 0;
        // 计算sum
        for (int i = 0; i < n; ++i) {
            sum += x[i] - scale*L[i];
        }
        // 更新最小值
        min = alpha*min + (1 - alpha)*sum/n;
        if (min > 0) min = 0;
        iscale = 1/scale;
        // 如果没有改变，跳出循环
        if (!did_change) break;
    }
    // 更新the_min，返回scale
    *the_min = -min;
    return scale;
}

static float make_qkx2_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    // 初始化最小值和最大值为数组第一个元素，初始化sum_w和sum_x
    float min = x[0];
    float max = x[0];
    float sum_w = weights[0];
    float sum_x = sum_w * x[0];
    // 遍历数组，更新最小值、最大值、sum_w和sum_x
    for (int i = 1; i < n; ++i) {
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
        float w = weights[i];
        sum_w += w;
        sum_x += w * x[i];
    }
    // 如果最小值大于0，将最小值置为0
    if (min > 0) min = 0;
    // 如果最大值等于最小值，将所有元素置为0，更新the_min，返回0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = -min;
        return 0.f;
    }
    // 计算缩放比例
    float iscale = nmax/(max - min);
    # 计算缩放比例
    float scale = 1/iscale;
    # 初始化最佳平均绝对偏差
    float best_mad = 0;
    # 遍历数据点
    for (int i = 0; i < n; ++i) {
        # 计算最近整数值
        int l = nearest_int(iscale*(x[i] - min));
        # 将 l 限制在 0 和 nmax 之间
        L[i] = MAX(0, MIN(nmax, l));
        # 计算差值
        float diff = scale * L[i] + min - x[i];
        # 根据 use_mad 决定使用绝对值还是平方
        diff = use_mad ? fabsf(diff) : diff * diff;
        # 获取权重
        float w = weights[i];
        # 更新最佳平均绝对偏差
        best_mad += w * diff;
    }
    # 如果步数小于 1，则返回最小值并返回缩放比例
    if (nstep < 1) {
        *the_min = -min;
        return scale;
    }
    # 遍历步数
    for (int is = 0; is <= nstep; ++is) {
        # 更新缩放比例
        iscale = (rmin + rdelta*is + nmax)/(max - min);
        # 初始化变量
        float sum_l = 0, sum_l2 = 0, sum_xl = 0;
        # 遍历数据点
        for (int i = 0; i < n; ++i) {
            # 计算最近整数值
            int l = nearest_int(iscale*(x[i] - min));
            # 将 l 限制在 0 和 nmax 之间
            l = MAX(0, MIN(nmax, l));
            # 更新 Laux 数组
            Laux[i] = l;
            # 获取权重
            float w = weights[i];
            # 更新变量
            sum_l += w*l;
            sum_l2 += w*l*l;
            sum_xl += w*l*x[i];
        }
        # 计算矩阵 D
        float D = sum_w * sum_l2 - sum_l * sum_l;
        # 如果 D 大于 0
        if (D > 0) {
            # 计算新的缩放比例和最小值
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;
            # 如果最小值大于 0，则更新为 0
            if (this_min > 0) {
                this_min = 0;
                this_scale = sum_xl / sum_l2;
            }
            # 初始化平均绝对偏差
            float mad = 0;
            # 遍历数据点
            for (int i = 0; i < n; ++i) {
                # 计算差值
                float diff = this_scale * Laux[i] + this_min - x[i];
                # 根据 use_mad 决定使用绝对值还是平方
                diff = use_mad ? fabsf(diff) : diff * diff;
                # 获取权重
                float w = weights[i];
                # 更新平均绝对偏差
                mad += w * diff;
            }
            # 如果当前平均绝对偏差小于最佳平均绝对偏差
            if (mad < best_mad) {
                # 更新 L 数组、最佳平均绝对偏差、缩放比例和最小值
                for (int i = 0; i < n; ++i) {
                    L[i] = Laux[i];
                }
                best_mad = mad;
                scale = this_scale;
                min = this_min;
            }
        }
    }
    # 返回最小值并返回缩放比例
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
        // 计算每个通道的量化参数
        float dl1 = d * (x[i].scales[0] & 0xF), ml1 = min * (x[i].scales[0] >> 4);
        float dl2 = d * (x[i].scales[1] & 0xF), ml2 = min * (x[i].scales[1] >> 4);
        float dl3 = d * (x[i].scales[2] & 0xF), ml3 = min * (x[i].scales[2] >> 4);
        float dl4 = d * (x[i].scales[3] & 0xF), ml4 = min * (x[i].scales[3] >> 4);
        // 对每个通道进行量化
        for (int l = 0; l < 16; ++l) {
            y[l+ 0] = dl1 * ((int8_t)((q[l] >> 0) & 3)) - ml1;
            y[l+16] = dl2 * ((int8_t)((q[l] >> 2) & 3)) - ml2;
            y[l+32] = dl3 * ((int8_t)((q[l] >> 4) & 3)) - ml3;
            y[l+48] = dl4 * ((int8_t)((q[l] >> 6) & 3)) - ml4;
        }
        // 更新指针位置
        y += QK_K;
#endif
    }
}

void quantize_row_q2_K(const float * restrict x, void * restrict vy, int k) {
    // 调用参考函数进行量化
    quantize_row_q2_K_reference(x, vy, k);
}

size_t ggml_quantize_q2_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对每个块进行量化
    for (int j = 0; j < n; j += k) {
        block_q2_K * restrict y = (block_q2_K *)dst + j/QK_K;
        quantize_row_q2_K_reference(src + j, y, k);
    }
    return (n/QK_K*sizeof(block_q2_K));
}

//========================= 3-bit (de)-quantization

void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k) {
    // 确保块大小是 QK_K 的整数倍
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    int8_t L[QK_K];
    float scales[QK_K / 16];

    for (int i = 0; i < nb; i++) {

        float max_scale = 0;
        float amax = 0;
        // 计算每个通道的量化参数
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
            // 将L数��中的值存入y[i].qs数组
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
// 结束条件判断，检查是否到达文件末尾
}
// 结束条件判断，检查是否到达文件末尾
#endif

// 对输入数组进行量化处理，调用参考函数 quantize_row_q3_K_reference
void quantize_row_q3_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q3_K_reference(x, vy, k);
}

// 对输入数组进行量化处理，返回处理后的数据大小
size_t ggml_quantize_q3_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 遍历输入数组，每次处理 k 个元素
    for (int j = 0; j < n; j += k) {
        // 将处理后的数据存储到目标数组中
        block_q3_K * restrict y = (block_q3_K *)dst + j/QK_K;
        quantize_row_q3_K_reference(src + j, y, k);
    }
    // 返回处理后的数据大小
    return (n/QK_K*sizeof(block_q3_K));
}

// ====================== 4-bit (de)-quantization

// 对输入数组进行 4 位量化处理的参考函数
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k) {
    // 断言，确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    uint8_t L[QK_K];
    uint8_t Laux[32];
    float   weights[32];
    float mins[QK_K/32];
    float scales[QK_K/32];

    // 遍历处理每个块
    for (int i = 0; i < nb; i++) {

        float max_scale = 0; // as we are deducting the min, scales are always positive
        float max_min = 0;
        // 遍历每个块中的子块
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算子块的平均值和权重
            float sum_x2 = 0;
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l];
            float av_x = sqrtf(sum_x2/32);
            for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]);
            // 进行 4 位量化处理
            scales[j] = make_qkx2_quants(32, 15, x + 32*j, weights, L + 32*j, &mins[j], Laux, -1.f, 0.1f, 20, false);
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
    // 初始化变量 is, sc, m
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
        // 循环计算并赋值
        for (int l = 0; l < 32; ++l) *y++ = d2 * (q[l]  >> 4) - m2;
        // 循环计算并赋值
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
        // 计算并赋值
        y[l+32] = d2 * (q[l] >>  4) - m2;
        // 计算并赋值
    }
    y += QK_K;
    // y 增加 QK_K
#endif
    // 结束条件编译指令块

}
    # 定义一个包含32个浮点数的数组weights
    float weights[32];
    # 定义一个包含32个8位无符号整数的数组Laux
    uint8_t Laux[32];
#else
    // 定义长度为 QK_K 的 int8_t 数组 L
    int8_t L[QK_K];
    // 定义长度为 QK_K/16 的 float 数组 scales
    float scales[QK_K/16];
#endif

    // 循环处理每个块
    for (int i = 0; i < nb; i++) {

#else
        // 初始化最大比例尺和最大值
        float max_scale = 0, amax = 0;
        // 遍历每个块的 16 个子块
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算每个子块的量化比例尺
            scales[j] = make_qx_quants(16, 16, x + 16*j, L + 16*j, 1);
            // 计算绝对值的比例尺
            float abs_scale = fabsf(scales[j]);
            // 更新最大值和最大比例尺
            if (abs_scale > amax) {
                amax = abs_scale;
                max_scale = scales[j];
            }
        }

        // 计算缩放因子
        float iscale = -128.f/max_scale;
        // 更新每个子块的量化比例尺
        for (int j = 0; j < QK_K/16; ++j) {
            int l = nearest_int(iscale*scales[j]);
            y[i].scales[j] = MAX(-128, MIN(127, l));
        }
        // 更新块的 d 值
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        // 更新每个子块的量化值
        for (int j = 0; j < QK_K/16; ++j) {
            const float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            if (!d) continue;
            for (int ii = 0; ii < 16; ++ii) {
                int l = nearest_int(x[16*j + ii]/d);
                l = MAX(-16, MIN(15, l));
                L[16*j + ii] = l + 16;
            }
        }

        // 初始化 qh 和 ql 数组
        uint8_t * restrict qh = y[i].qh;
        uint8_t * restrict ql = y[i].qs;
        memset(qh, 0, QK_K/8);

        // 更新 ql 和 qh 数组的值
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

        // 更新 x 指针
        x += QK_K;

    }
}

// 解量化函数
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k) {
    // 断言 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    // 循环处理每个块
    for (int i = 0; i < nb; i++) {

        // 获取 ql 和 qh 数组的值
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
            // 计���结果并存入 y
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
// ====================== 6-bit (de)-quantization

// 对输入的一行数据进行 6 位量化，结果存储在 block_q6_K 结构体中
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    int8_t L[QK_K]; // 存储量化后的数据
    float   scales[QK_K/16]; // 存储每个块的缩放因子

    for (int i = 0; i < nb; i++) {

        float max_scale = 0; // 最大缩放因子
        float max_abs_scale = 0; // 最大绝对值的缩放因子

        for (int ib = 0; ib < QK_K/16; ++ib) {

            // 对每个块进行 16 位量化，返回缩放因子
            const float scale = make_qx_quants(16, 32, x + 16*ib, L + 16*ib, 1);
            scales[ib] = scale;

            const float abs_scale = fabsf(scale); // 计算缩放因子的绝对值
            if (abs_scale > max_abs_scale) {
                max_abs_scale = abs_scale;
                max_scale = scale;
            }

        }

        // 如果最大绝对值的缩放因子为 0，则将 y[i] 清零并跳过当前循环
        if (!max_abs_scale) {
            memset(&y[i], 0, sizeof(block_q6_K));
            y[i].d = GGML_FP32_TO_FP16(0.f);
            x += QK_K;
            continue;
        }

        // 计算缩放因子的倒数
        float iscale = -128.f/max_scale;
        y[i].d = GGML_FP32_TO_FP16(1/iscale);
        // 对每个块的缩放因子进行量化
        for (int ib = 0; ib < QK_K/16; ++ib) {
            y[i].scales[ib] = MIN(127, nearest_int(iscale*scales[ib]));
        }

        // 对每个块进行反量化
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

        uint8_t * restrict ql = y[i].ql; // 低位量化结果
        uint8_t * restrict qh = y[i].qh; // 高位量化结果
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
//===================================== Q8_K ==============================================

// 对输入的一维数组进行量化，结果存储在 block_q8_K 结构体数组中，每个结构体包含 QK_K 个量化值
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    for (int i = 0; i < nb; i++) {

        float max = 0;
        float amax = 0;
        // 找到当前块中绝对值最大的值
        for (int j = 0; j < QK_K; ++j) {
            float ax = fabsf(x[j]);
            if (ax > amax) {
                amax = ax; max = x[j];
            }
        }
        // 如果最大值为0，则将当前块的量化值设为0，并清空量化值数组
        if (!amax) {
            y[i].d = 0;
            memset(y[i].qs, 0, QK_K);
            x += QK_K;
            continue;
        }
        // 计算缩放因子
        const float iscale = -128.f/max;
        // 对当前块进行量化
        for (int j = 0; j < QK_K; ++j) {
            int v = nearest_int(iscale*x[j]);
            y[i].qs[j] = MIN(127, v);
        }
        // 计算每16个量化值的和，存储在 bsums 数组中
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

// 将 block_q8_K 结构体数组中的量化值反量化，结果存储在一维数组中
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    const int nb = k / QK_K;

    for (int i = 0; i < nb; i++) {
        for (int j = 0; j < QK_K; ++j) {
            // 反量化
            *y++ = x[i].d * x[i].qs[j];
        }
    }
}

// 对输入的一维数组进行量化，结果存储在 block_q8_K 结构体数组中，每个结构体包含 QK_K 个量化值
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k) {
    quantize_row_q8_K_reference(x, y, k);
}

//===================================== Dot ptoducts =================================

//
// Helper functions
//
#if __AVX__ || __AVX2__ || __AVX512F__

// 用于在点积中选择所需的缩放值的洗牌操作
static inline __m256i get_scale_shuffle_q3k(int i) {
    // 定义一个包含128个元素的常量数组，用于对数据进行重新排列
    static const uint8_t k_shuffle[128] = {
         0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,     2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
         4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5, 4, 5,     6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7, 6, 7,
         8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9, 8, 9,    10,11,10,11,10,11,10,11,10,11,10,11,10,11,10,11,
        12,13,12,13,12,13,12,13,12,13,12,13,12,13,12,13,    14,15,14,15,14,15,14,15,14,15,14,15,14,15,14,15,
    };
    // 根据索引 i 加载对应位置的数据并返回
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
// 定义一个静态内联函数，返回一个包含256个元素的__m256i类型的向量，根据参数i返回不同的值
static inline __m256i get_scale_shuffle_k4(int i) {
    // 定义一个包含256个元素的uint8_t类型的数组，用于存储特定的值
    static const uint8_t k_shuffle[256] = {
         // 数组内容省略，包含256个元素
    };
    // 返回根据参数i计算得到的__m256i类型的向量
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
}

// 定义一个静态内联函数，返回一个包含128个元素的__m128i类型的向量，根据参数i返回不同的值
static inline __m128i get_scale_shuffle(int i) {
    // 定义一个包含128个元素的uint8_t类型的数组，用于存储特定的值
    static const uint8_t k_shuffle[128] = {
         // 数组内容省略，包含128个元素
    };
    // 返回根据参数i计算得到的__m128i类型的向量
    return _mm_loadu_si128((const __m128i*)k_shuffle + i);
}

// 定义一个函数，计算两个向量的点积
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量qk为QK8_0
    const int qk = QK8_0;
    // 计算n除以qk的商，得到nb
    const int nb = n / qk;

    // 断言n对qk取模为0
    assert(n % qk == 0);

    // 定义指向vx和vy的指针，分别转换为block_q4_0和block_q8_0类型
    const block_q4_0 * restrict x = vx;
    const block_q8_0 * restrict y = vy;

    // 如果定义了__ARM_NEON，则使用NEON指令集进行计算
    #if defined(__ARM_NEON)
    // 定义一个包含4个float32x4_t类型元素的向量，每个元素值为0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    // 创建一个包含四个浮点数的向量，每个值都是0.0f
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 断言nb是偶数，如果不是偶数则需要处理
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环处理每两个元素
    for (int i = 0; i < nb; i += 2) {
        // 定义指向x和y数组中当前元素的指针
        const block_q4_0 * restrict x0 = &x[i + 0];
        const block_q4_0 * restrict x1 = &x[i + 1];
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个包含16个0x0F的无符号整数向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
        // 创建一个包含16个0x8的有符号整数向量
        const int8x16_t  s8b = vdupq_n_s8(0x8);

        // 从x数组中加载两个16字节的无符号整数向量
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

        // 从y数组中加载数据
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);
#if defined(__ARM_FEATURE_DOTPROD)
        // 如果支持 ARM dot product 指令集
        // 将两个 int32x4_t 向量进行点积运算
        const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0ls, v1_0l), v0_0hs, v1_0h);
        const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1ls, v1_1l), v0_1hs, v1_1h);

        // 使用点积结果更新累加器 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#else
        // 如果不支持 ARM dot product 指令集
        // 将两个 int16x8_t 向量进行乘法运算
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0ls), vget_low_s8 (v1_0l));
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0ls), vget_high_s8(v1_0l));
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hs), vget_low_s8 (v1_0h));
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hs), vget_high_s8(v1_0h));

        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1ls), vget_low_s8 (v1_1l));
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1ls), vget_high_s8(v1_1l));
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hs), vget_low_s8 (v1_1h));
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hs), vget_high_s8(v1_1h));

        // 将乘法结果累加为 int32x4_t 向量
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 使用累加结果更新累加器 sumv0 和 sumv1
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif
    }

    // 计算最终结果并存储在指针 s 所指向的位置
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__)
    // 如果支持 AVX2 指令集
    // 初始化累加器为全零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    // 遍历数组 x 和 y，计算每个元素的乘积并累加到 acc 中
    for (int i = 0; i < nb; ++i) {
        // 计算 x[i].d 和 y[i].d 的乘积，并将结果复制到向量 d 中
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 将 x[i].qs 中的 nibble 转换为字节，存储到向量 bx 中
        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 将 bx 中的字节值从 [0, 15] 范围偏移为 [-8, 7] 范围
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );

        // 从 y[i].qs 中加载字节到向量 by 中
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算 bx 和 by 的乘积，并将结果存储到向量 q 中
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将向量 d 乘以向量 q，并将结果累加到 acc 中
        acc = _mm256_fmadd_ps( d, q, acc );
    }

    // 将向量 acc 中的元素水平求和，并将结果存储到指针 s 指向的位置
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 如果定义了 AVX，则执行以下代码块

    // 初始化累加器为零
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
    // 如果定义了 SSSE3，则执行以下代码块

    // 设置常量
    const __m128i lowMask = _mm_set1_epi8(0xF);
    const __m128i off = _mm_set1_epi8(8);

    // 初始化累加器为零
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
    // 如果定义了 riscv_v_intrinsic，则执行以下代码块

    // 初始化浮点数和为 0.0
    float sumf = 0.0;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 加载元素到向量 tx
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        // 加载 y[i].qs 和 y[i].qs+16 的元素到向量 y0 和 y1
        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 对 tx 进行位与运算，得到 x_a 和 x_l
        vuint8mf2_t x_a = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        vuint8mf2_t x_l = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        // 将 x_a 和 x_l 转换为有符号整数向量 x_ai 和 x_li
        vint8mf2_t x_ai = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        vint8mf2_t x_li = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        // 对 x_ai 和 x_li 分别减去 8
        vint8mf2_t v0 = __riscv_vsub_vx_i8mf2(x_ai, 8, vl);
        vint8mf2_t v1 = __riscv_vsub_vx_i8mf2(x_li, 8, vl);

        // 对 v0 和 y0 进行有符号整数乘法，得到 vec_mul1
        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        // 对 v1 和 y1 进行有符号整数乘法，得到 vec_mul2
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        // 创建一个全为 0 的有符号整数向量 vec_zero
        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        // 对 vec_mul1 进行累加求和，结果存储在 vs1 中
        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        // 对 vec_mul2 进行累加求和，结果存储在 vs2 中
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        // 将 vs2 转换为标量整数 sumi
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        // 计算 x[i].d 和 y[i].d 的乘积，并加到 sumf 上
        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#else
    // 如果不支持 SIMD，使用标量计算
    float sumf = 0.0; // 初始化浮点数求和变量

    for (int i = 0; i < nb; i++) { // 遍历每个块
        int sumi = 0; // 初始化整数求和变量

        for (int j = 0; j < qk/2; ++j) { // 遍历每个块的一半
            // 计算每个块的值并求和
            const int v0 = (x[i].qs[j] & 0x0F) - 8; // 计算第一个值
            const int v1 = (x[i].qs[j] >>   4) - 8; // 计算第二个值

            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]); // 更新求和值
        }

        // 计算最终结果并加到总和中
        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    *s = sumf; // 将最终结果赋值给输出变量
#endif
}

void ggml_vec_dot_q4_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_1; // 定义块大小
    const int nb = n / qk; // 计算块的数量

    assert(n % qk == 0); // 断言确保 n 能被 qk 整除

    const block_q4_1 * restrict x = vx; // 强制转换输入为 block_q4_1 类型
    const block_q8_1 * restrict y = vy; // 强制转换输入为 block_q8_1 类型

    // TODO: add WASM SIMD
#if defined(__ARM_NEON)
    // 如果支持 NEON SIMD 指令集
    float32x4_t sumv0 = vdupq_n_f32(0.0f); // 初始化 NEON 浮点数求和变量
    float32x4_t sumv1 = vdupq_n_f32(0.0f); // 初始化 NEON 浮点数求和变量

    float summs = 0; // 初始化浮点数求和变量

    assert(nb % 2 == 0); // 断言确保块数量为偶数

    for (int i = 0; i < nb; i += 2) { // 每次处理两个块
        const block_q4_1 * restrict x0 = &x[i + 0]; // 获取第一个块
        const block_q4_1 * restrict x1 = &x[i + 1]; // 获取第二个块
        const block_q8_1 * restrict y0 = &y[i + 0]; // 获取第一个块
        const block_q8_1 * restrict y1 = &y[i + 1]; // 获取第二个块

        // 计算 NEON 浮点数求和
        summs += GGML_FP16_TO_FP32(x0->m) * y0->s + GGML_FP16_TO_FP32(x1->m) * y1->s;

        const uint8x16_t m4b = vdupq_n_u8(0x0F); // 创建一个 16 个 8 位整数的常量向量

        const uint8x16_t v0_0 = vld1q_u8(x0->qs); // 加载第一个块的数据
        const uint8x16_t v0_1 = vld1q_u8(x1->qs); // 加载第二个块的数据

        // 4位 -> 8位
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b)); // 将第一个块的数据转换为 8 位整数
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4)); // 将第一个块的数据转换为 8 位整数
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b)); // 将第二个块的数据转换为 8 位整数
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4)); // 将第二个块的数据转换为 8 位整数

        // 加载 y
        const int8x16_t v1_0l = vld1q_s8(y0->qs); // 加载第一个块的数据
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16); // 加载第一个块的数据
        const int8x16_t v1_1l = vld1q_s8(y1->qs); // 加载第二个块的数据
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16); // 加载第二个块的数据
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM dot product 特性，则执行以下代码块
    // 计算两个 int32x4_t 向量的点积
    const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0l, v1_0l), v0_0h, v1_0h);
    const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1l, v1_1l), v0_1h, v1_1h);

    // 使用点积结果更新 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*y0->d);
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
    // 如果不支持 ARM dot product 特性，则执行以下代码块
    // 计算 int16x8_t 向量的乘法
    const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0l), vget_low_s8 (v1_0l));
    const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0l), vget_high_s8(v1_0l));
    const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0h), vget_low_s8 (v1_0h));
    const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0h), vget_high_s8(v1_0h));

    const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1l), vget_low_s8 (v1_1l));
    const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1l), vget_high_s8(v1_1l));
    const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1h), vget_low_s8 (v1_1h));
    const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1h), vget_high_s8(v1_1h));

    // 将 int16x8_t 向量相加得到 int32x4_t 向量
    const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
    const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
    const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
    const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

    // 使用相加结果更新 sumv0 和 sumv1
    sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
}

// 将 sumv0 和 sumv1 的结果相加，再加上 summs，得到最终结果
*s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs;
#elif defined(__AVX2__) || defined(__AVX__)
// 如果支持 AVX2 或 AVX 特性，则执行以下代码块
// 初始化一个全为零的 __m256 向量
__m256 acc = _mm256_setzero_ps();

// 初始化 summs 为 0
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
    // 如果支持 ARM NEON 指令集

    // 初始化两个包含四个浮点数的向量，每个元素都是 0.0f
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 定义两个无符号整数变量
    uint32_t qh0;
    uint32_t qh1;

    // 定义两个包含四个 64 位整数的数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言 nb 为偶数，如果不是偶数则抛出异常
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 如果支持 ARM Dot Product 指令集
#if defined(__ARM_FEATURE_DOTPROD)
        // 计算 sumv0 的值
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        // 计算 sumv1 的值
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#else
        // 计算低位和高位的乘积
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));

        // 计算低位和高位的和
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 计算最终结果
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif
    }

    // 计算最终结果并存储在指针 s 中
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__wasm_simd128__)
    // 初始化一个包含 0.0f 的 SIMD 寄存器
    v128_t sumv = wasm_f32x4_splat(0.0f);

    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }

    // 提取 SIMD 寄存器中的每个元素并相加，存储在指针 s 中
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3);
#elif defined(__AVX2__)
    // 初始化一个包含 0.0f 的 AVX 寄存器
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    // 遍历循环，从0到nb-1
    for (int i = 0; i < nb; i++) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将x[i].qs转换为字节并存储在bx中
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将x[i].qh转换为字节并存储在bxhi中
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 对bxhi取反后与0xF0进行与操作，并将结果存储在bxhi中
        bxhi = _mm256_andnot_si256(bxhi, _mm256_set1_epi8((char)0xF0));
        // 将bx和bxhi进行或操作，并将结果存储在bx中
        bx = _mm256_or_si256(bx, bxhi);

        // 从y[i].qs加载数据到by中
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积和并将结果存储在q中
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将q乘以比例d并累加到acc中
        acc = _mm256_fmadd_ps(d, q, acc);
    }

    // 将acc中的8个浮点数求和并存储在*s中
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 如果定义了 AVX，则执行以下代码块

    // 使用零初始化累加器
    __m256 acc = _mm256_setzero_ps();
    // 创建一个掩码，用于屏蔽高位字节
    __m128i mask = _mm_set1_epi8((char)0xF0);

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 x[i].qs 转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为字节
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

        // 加载 y[i].qs 到 by
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并累加
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加到 acc
        acc = _mm256_add_ps(_mm256_mul_ps(d, q), acc);
    }

    // 将 acc 中的值求和并存储到 s
    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 如果定义了 riscv_v_intrinsic，则执行以下代码块

    // 初始化 sumf
    float sumf = 0.0;

    uint32_t qh;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 以下临时寄存器用于掩码和移位操作
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vsll_vv_u32m2(__riscv_vmv_v_x_u32m2(1, vl), vt_1, vl);

    vuint32m2_t vt_3 = __riscv_vsll_vx_u32m2(vt_2, 16, vl);
    vuint32m2_t vt_4 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    // 将 sumf 存储到 s
    *s = sumf;
#else
    // 如果以上条件都不满足，则执行以下代码块

    // 初始化 sumf
    float sumf = 0.0;
    // 遍历数组 x 中的元素，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        // 从结构体 x 中复制 qh 到变量 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 初始化 sumi 为 0
        int sumi = 0;

        // 遍历 qk/2 次，j 从 0 到 qk/2-1
        for (int j = 0; j < qk/2; ++j) {
            // 计算 xh_0，取出 qh 中第 j+0 位的值，左移 4 位
            const uint8_t xh_0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
            // 计算 xh_1，取出 qh 中第 j+16 位的值，右移 12 位
            const uint8_t xh_1 = ((qh & (1u << (j + 16))) >> (j + 12));

            // 计算 x0，取出 x[i].qs[j] 的低 4 位，与 xh_0 相或，减去 16
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            // 计算 x1，取出 x[i].qs[j] 的高 4 位，与 xh_1 相或，减去 16
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            // 计算 sumi，累加 x0 与 y[i].qs[j] 以及 x1 与 y[i].qs[j + qk/2] 的乘积
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }

        // 计算 sumf，累加 x[i].d 与 y[i].d 的乘积再乘以 sumi
        sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;
    }

    // 将最终结果赋值给指针 s 指向的变量
    *s = sumf;
#endif
}

// 计算两个向量的点积，使用 ARM NEON 指令集
void ggml_vec_dot_q5_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义每个块的大小
    const int qk = QK8_1;
    // 计算块的数量
    const int nb = n / qk;

    // 断言确保 n 能够被 qk 整除
    assert(n % qk == 0);
    // 断言确保 qk 的值为 QK5_1

    const block_q5_1 * restrict x = vx;
    const block_q8_1 * restrict y = vy;

    // 如果定义了 ARM NEON，则执行以下代码块
#if defined(__ARM_NEON)
    // 初始化两个包含四个浮点数的向量，值为 0.0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 初始化两个浮点数，值为 0.0
    float summs0 = 0.0f;
    float summs1 = 0.0f;

    // 定义两个无符号整数变量
    uint32_t qh0;
    uint32_t qh1;

    // 定义两个包含四个 64 位整数的数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言确保 nb 是偶数，TODO: 处理奇数的情况

    // 如果定义了 ARM DOTPROD，则执行以下代码块
#if defined(__ARM_FEATURE_DOTPROD)
        // 计算点积并更新 sumv0
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*y0->d);
        // 计算点积并更新 sumv1
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
        // 计算低位和高位的乘积
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));

        // 计算低位和高位的和
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 计算最终结果
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
    }

    // 计算最终结果并赋值给指针 s
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs0 + summs1;
#elif defined(__wasm_simd128__)
    // 初始化 SIMD 寄存器为 0.0f
    v128_t sumv = wasm_f32x4_splat(0.0f);

    // 初始化浮点数为 0.0f
    float summs = 0.0f;

    uint32_t qh;
    uint64_t tmp[4];

    // TODO: check if unrolling this is better
    }

    // 计算最终结果并赋值给指针 s
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
         wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3) + summs;
#elif defined(__AVX2__)
    // 初始化 AVX 寄存器为 0.0f
    __m256 acc = _mm256_setzero_ps();

    // 初始化浮点数为 0.0f
    float summs = 0.0f;

    // 主循环
    // 遍历长度为nb的循环
    for (int i = 0; i < nb; i++) {
        // 将x[i].d转换为__m256类型的向量dx
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算summs，累加x[i].m和y[i].s的乘积
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将x[i].qs转换为__m256i类型的向量bx
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将x[i].qh转换为__m256i类型的向量bxhi
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 将bxhi中大于0x10的位清零
        bxhi = _mm256_and_si256(bxhi, _mm256_set1_epi8(0x10));
        // 将bx和bxhi按位或操作
        bx = _mm256_or_si256(bx, bxhi);

        // 将y[i].d转换为__m256类型的向量dy
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从y[i].qs加载__m256i类型的向量by
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算bx和by的乘积，结果存储在q中
        const __m256 q = mul_sum_us8_pairs_float(bx, by);

        // 使用FMA指令计算累加器acc
        acc = _mm256_fmadd_ps(q, _mm256_mul_ps(dx, dy), acc);
    }

    // 将acc中的8个浮点数求和，并加上summs，结果存储在*s中
    *s = hsum_float_8(acc) + summs;
#elif defined(__AVX__)
    // 如果定义了 AVX，则执行以下代码块

    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含特定值的 128 位整数
    __m128i mask = _mm_set1_epi8(0x10);

    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 创建一个包含特定值的 256 位浮点数
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将 32 位整数转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从 32 位整数中提取高位字节
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

        // 创建一个包含特定值的 256 位浮点数
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 从内存中加载 256 位整数
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并求和
        const __m256 q = mul_sum_us8_pairs_float(bx, by);

        // 更新累加器
        acc = _mm256_add_ps(_mm256_mul_ps(q, _mm256_mul_ps(dx, dy)), acc);
    }

    // 计算最终结果并赋值给 s
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 如果定义了 riscv_v_intrinsic，则执行以下代码块

    float sumf = 0.0;

    uint32_t qh;

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 用于移位操作的临时寄存器
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
    // 如果不是 ARM_NEON，使用标量计算
    float sumf = 0.0; // 初始化浮点数求和变量为0

    for (int i = 0; i < nb; i++) { // 遍历每个块
        uint32_t qh; // 定义无符号整型变量
        memcpy(&qh, x[i].qh, sizeof(qh)); // 从 x[i].qh 复制数据到 qh

        int sumi = 0; // 初始化整型求和变量为0

        for (int j = 0; j < qk/2; ++j) { // 遍历每个块的一半
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10; // 计算 xh_0
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10; // 计算 xh_1

            const int32_t x0 = (x[i].qs[j] & 0xF) | xh_0; // 计算 x0
            const int32_t x1 = (x[i].qs[j] >>  4) | xh_1; // 计算 x1

            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]); // 更新 sumi
        }

        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s; // 更新 sumf
    }

    *s = sumf; // 将最终结果赋值给 s
#endif
}

void ggml_vec_dot_q8_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_0; // 定义 qk 为 QK8_0
    const int nb = n / qk; // 计算 nb

    assert(n % qk == 0); // 断言 n 能被 qk 整除

    const block_q8_0 * restrict x = vx; // 定义 x 指向 vx
    const block_q8_0 * restrict y = vy; // 定义 y 指向 vy

#if defined(__ARM_NEON)
    // 如果是 ARM_NEON，使用 SIMD 计算
    float32x4_t sumv0 = vdupq_n_f32(0.0f); // 初始化 SIMD 浮点数求和变量为0
    float32x4_t sumv1 = vdupq_n_f32(0.0f); // 初始化 SIMD 浮点数求和变量为0

    assert(nb % 2 == 0); // 断言 nb 是偶数，处理奇数情况

    for (int i = 0; i < nb; i += 2) { // 每次处理两个块
        const block_q8_0 * restrict x0 = &x[i + 0]; // 指向第一个块
        const block_q8_0 * restrict x1 = &x[i + 1]; // 指向第二个块
        const block_q8_0 * restrict y0 = &y[i + 0]; // 指向第一个块
        const block_q8_0 * restrict y1 = &y[i + 1]; // 指向第二个块

        const int8x16_t x0_0 = vld1q_s8(x0->qs); // 加载 x0 的第一个部分
        const int8x16_t x0_1 = vld1q_s8(x0->qs + 16); // 加载 x0 的第二个部分
        const int8x16_t x1_0 = vld1q_s8(x1->qs); // 加载 x1 的第一个部分
        const int8x16_t x1_1 = vld1q_s8(x1->qs + 16); // 加载 x1 的第二个部分

        // 加载 y
        const int8x16_t y0_0 = vld1q_s8(y0->qs); // 加载 y0 的第一个部分
        const int8x16_t y0_1 = vld1q_s8(y0->qs + 16); // 加载 y0 的第二个部分
        const int8x16_t y1_0 = vld1q_s8(y1->qs); // 加载 y1 的第一个部分
        const int8x16_t y1_1 = vld1q_s8(y1->qs + 16); // 加载 y1 的第二个部分
#if defined(__ARM_FEATURE_DOTPROD)
        // 如果支持 ARM Dot Product 指令集，则使用 vdotq_s32 计算点积
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), x0_0, y0_0),
                        vdotq_s32(vdupq_n_s32(0), x0_1, y0_1))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));

        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), x1_0, y1_0),
                        vdotq_s32(vdupq_n_s32(0), x1_1, y1_1))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));

#else
        // 如果不支持 ARM Dot Product 指令集，则使用 vmull_s8 和 vpaddlq_s16 计算点积
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

        // 使用 vmull_s8 和 vpaddlq_s16 计算点积，然后进行累加
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(p0, p1)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(p2, p3)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif
    }

    // 计算最终结果
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__) || defined(__AVX__)
    // 如果支持 AVX2 或 AVX 指令集，则初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    // 循环遍历每个块，i 从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));
        // 从 x[i].qs 加载 256 位整数到 bx
        __m256i bx = _mm256_loadu_si256((const __m256i *)x[i].qs);
        // 从 y[i].qs 加载 256 位整数到 by
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 调用 mul_sum_i8_pairs_float 函数，计算 bx 和 by 的乘积
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加
#if defined(__AVX2__)
        // 如果支持 AVX2 指令集，则使用 AVX2 指令进行计算
        acc = _mm256_fmadd_ps( d, q, acc );
#else
        // 如果不支持 AVX2 指令集，则使用普通的乘法和加法指令进行计算
        acc = _mm256_add_ps( _mm256_mul_ps( d, q ), acc );
#endif
    }

    // 计算累加和并存储到指定地址
    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 如果支持 RISC-V 指令集，则使用 RISC-V 指令进行计算
    float sumf = 0.0;
    // 获取当前向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk);

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vint8m1_t bx = __riscv_vle8_v_i8m1(x[i].qs, vl);
        vint8m1_t by = __riscv_vle8_v_i8m1(y[i].qs, vl);

        // 向量乘法
        vint16m2_t vw_mul = __riscv_vwmul_vv_i16m2(bx, by, vl);

        // 初始化零向量
        vint32m1_t v_zero = __riscv_vmv_v_x_i32m1(0, vl);
        // 向量求和
        vint32m1_t v_sum = __riscv_vwredsum_vs_i16m2_i32m1(vw_mul, v_zero, vl);

        // 将求和结果转换为整数
        int sumi = __riscv_vmv_x_s_i32m1_i32(v_sum);

        // 计算最终结果
        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    // 存储最终结果到指定地址
    *s = sumf;
#else
    // 如果不支持特定指令集，则使用标量计算
    float sumf = 0.0;

    for (int i = 0; i < nb; i++) {
        int sumi = 0;

        for (int j = 0; j < qk; j++) {
            sumi += x[i].qs[j]*y[i].qs[j];
        }

        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    // 存储最终结果到指定地址
    *s = sumf;
#endif
}

#if QK_K == 256
void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    const block_q2_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 定义常量向量
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    const uint8x16_t m4 = vdupq_n_u8(0xF);
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 定义变量
    int8x16x2_t q2bytes;
    uint8_t aux[16];

    // 初始化和值
    float sum = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d 值，y[i].d 乘以 x[i].d 的浮点数结果
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 dmin 值，-y[i].d 乘以 x[i].dmin 的浮点数结果
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 声明并初始化指向 x[i].qs 的 uint8_t 指针 q2
        const uint8_t * restrict q2 = x[i].qs;
        // 声明并初始化指向 y[i].qs 的 int8_t 指针 q8
        const int8_t  * restrict q8 = y[i].qs;
        // 声明并初始化指向 x[i].scales 的 uint8_t 指针 sc
        const uint8_t * restrict sc = x[i].scales;

        // 从 sc 中加载 16 个字节到 mins_and_scales 中
        const uint8x16_t mins_and_scales = vld1q_u8(sc);
        // 通过位与操作获取 scales，与 m4 进行位与操作
        const uint8x16_t scales = vandq_u8(mins_and_scales, m4);
        // 将 scales 存储到 aux 中
        vst1q_u8(aux, scales);

        // 右移 4 位得到 mins
        const uint8x16_t mins = vshrq_n_u8(mins_and_scales, 4);
        // 从 y[i].bsums 中加载 16 个 int16_t 到 q8sums 中
        const int16x8x2_t q8sums = vld1q_s16_x2(y[i].bsums);
        // 将 mins 转换为 int16x8x2_t 类型的 mins16
        const int16x8x2_t mins16 = {vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(mins))), vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(mins)))};
        // 计算 s0，s1
        const int32x4_t s0 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[0]), vget_low_s16 (q8sums.val[0])),
                                       vmull_s16(vget_high_s16(mins16.val[0]), vget_high_s16(q8sums.val[0]));
        const int32x4_t s1 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[1]), vget_low_s16 (q8sums.val[1])),
                                       vmull_s16(vget_high_s16(mins16.val[1]), vget_high_s16(q8sums.val[1]));
        // 计算 sum
        sum += dmin * vaddvq_s32(vaddq_s32(s0, s1));

        // 初始化 isum 和 is
        int isum = 0;
        int is = 0;
// 如果支持 ARM 的 dot product 特性，则使用宏定义 MULTIPLY_ACCUM_WITH_SCALE
#if defined(__ARM_FEATURE_DOTPROD)
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        isum += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * aux[is+(index)];\
        isum += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * aux[is+1+(index)];
#else
// 否则使用 AVX2 指令集，定义 MULTIPLY_ACCUM_WITH_SCALE 宏
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        {\
    const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0])));\
    const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[1]), vget_low_s8 (q8bytes.val[1])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[1]), vget_high_s8(q8bytes.val[1])));\
    isum += vaddvq_s16(p1) * aux[is+(index)] + vaddvq_s16(p2) * aux[is+1+(index)];\
        }
#endif

// 定义 SHIFT_MULTIPLY_ACCUM_WITH_SCALE 宏
#define SHIFT_MULTIPLY_ACCUM_WITH_SCALE(shift, index)\
        q8bytes = vld1q_s8_x2(q8); q8 += 32;\
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[0], (shift)), m3));\
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[1], (shift)), m3));\
        MULTIPLY_ACCUM_WITH_SCALE((index));

// 循环处理 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 加载两个 128 位的无符号整数向量到 q2bits
            const uint8x16x2_t q2bits = vld1q_u8_x2(q2); q2 += 32;

            // 加载两个 128 位的有符号整数向量到 q8bytes
            int8x16x2_t q8bytes = vld1q_s8_x2(q8); q8 += 32;
            // 将 q2bits 的值按位与 m3，转换为有符号整数向量并赋值给 q2bytes
            q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[0], m3));
            q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[1], m3));
            // 调用 MULTIPLY_ACCUM_WITH_SCALE 处理索引为 0 的情况
            MULTIPLY_ACCUM_WITH_SCALE(0);

            // 调用 SHIFT_MULTIPLY_ACCUM_WITH_SCALE 处理索引为 2 的情况
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(2, 2);

            // 调用 SHIFT_MULTIPLY_ACCUM_WITH_SCALE 处理索引为 4 的情况
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(4, 4);

            // 调用 SHIFT_MULTIPLY_ACCUM_WITH_SCALE 处理索引为 6 的情况
            SHIFT_MULTIPLY_ACCUM_WITH_SCALE(6, 6);

            // 更新索引值
            is += 8;
        }
        // 更新 sum 值
        sum += d * isum;

    }

    *s = sum;

#elif defined __AVX2__
    // 创建一个包含8个元素，每个元素都是8位整数3的 AVX 寄存器
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含16个元素，每个元素都是8位整数15的 SSE 寄存器
    const __m128i m4 = _mm_set1_epi8(0xF);

    // 创建一个包含8个单精度浮点数的 AVX 寄存器，并初始化为0
    __m256 acc = _mm256_setzero_ps();

    // 计算 AVX 寄存器 acc 中所有元素的和，并将结果存储在 s 指向的地址中
    *s = hsum_float_8(acc);
#elif defined __AVX__

    # 如果定义了 AVX，则定义一些常量
    const __m128i m3 = _mm_set1_epi8(0x3);
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(0x2);

    # 初始化一个 AVX 寄存器为全零
    __m256 acc = _mm256_setzero_ps();

    }

    # 调用 hsum_float_8 函数计算 AVX 寄存器中的和，并将结果存入指针 s 指向的位置

#elif defined __riscv_v_intrinsic

    # 如果定义了 riscv_v_intrinsic，则进行一些操作
    float sumf = 0;
    uint8_t temp_01[32] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                            1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1};

    }

    # 将 sumf 的值存入指针 s 指向的位置

#else

    # 如果未定义 AVX 和 riscv_v_intrinsic，则进行一些操作
    float sumf = 0;

    # 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        # 获取 x[i] 和 y[i] 的指针
        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;

        int summs = 0;
        # 遍历 16 次
        for (int j = 0; j < 16; ++j) {
            # 计算 summs 的值
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        # 计算 dall 和 dmin 的值
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        int isum = 0;
        int is = 0;
        int d;
        # 遍历 QK_K/128 次
        for (int k = 0; k < QK_K/128; ++k) {
            int shift = 0;
            # 遍历 4 次
            for (int j = 0; j < 4; ++j) {
                d = sc[is++] & 0xF;
                int isuml = 0;
                # 遍历 16 次
                for (int l =  0; l < 16; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3);
                isum += d * isuml;
                d = sc[is++] & 0xF;
                isuml = 0;
                # 遍历 16 次
                for (int l = 16; l < 32; ++l) isuml += q8[l] * ((q2[l] >> shift) & 3);
                isum += d * isuml;
                shift += 2;
                q8 += 32;
            }
            q2 += 32;
        }
        # 计算 sumf 的值
        sumf += dall * isum - dmin * summs;
    }
    # 将 sumf 的值存入指针 s 指向的位置
    *s = sumf;
#endif
}

#else

void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    # 定义指向 x 和 y 的指针
    const block_q2_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    # 计算 nb 的值
    const int nb = n / QK_K;

    #ifdef __ARM_NEON

    # 如果定义了 ARM_NEON，则定义一些常量
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    #if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
    #endif
    // 定义一个包含4个int8x16类型的向量的变量q2bytes
    int8x16x4_t q2bytes;

    // 定义一个包含2个uint32_t类型的数组aux32，并将其转换为const uint8_t类型指针scales
    uint32_t aux32[2];
    const uint8_t * scales = (const uint8_t *)aux32;

    // 定义一个float类型变量sum，并初始化为0
    float sum = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 计算d和dmin的值
        const float d = y[i].d * (float)x[i].d;
        const float dmin = -y[i].d * (float)x[i].dmin;

        // 定义const uint8_t类型指针q2和const int8_t类型指针q8，并初始化为x[i].qs和y[i].qs
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;

        // 将sc数组中的值按位与0x0f0f0f0f后存入aux32数组的第一个元素，将右移4位后按位与0x0f0f0f0f后存入aux32数组的第二个元素
        aux32[0] = sc[0] & 0x0f0f0f0f;
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算sum的值
        sum += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);

        // 定义isum1和isum2为0

        // 从q2中加载16个uint8_t类型的值到q2bits
        const uint8x16_t q2bits = vld1q_u8(q2);

        // 从q8中加载4个int8x16类型的值到q8bytes
        const int8x16x4_t q8bytes = vld1q_s8_x4(q8);

        // 将q2bits中的值按位与m3后转换为int8x16类型存入q2bytes的val[0]中
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits, m3));
        // 将q2bits右移2位后按位与m3后转换为int8x16类型存入q2bytes的val[1]中
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 2), m3));
        // 将q2bits右移4位后按位与m3后转换为int8x16类型存入q2bytes的val[2]中
        q2bytes.val[2] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 4), m3));
        // 将q2bits右移6位后按位与m3后转换为int8x16类型存入q2bytes的val[3]中
        q2bytes.val[3] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 6), m3));
#if defined(__ARM_FEATURE_DOTPROD)
        // 如果支持 ARM Dot Product 指令集
        // 计算第一个和第二个向量的点积，并乘以对应的比例因子，累加到 isum1 和 isum2 中
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * scales[0];
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * scales[1];
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[2], q8bytes.val[2])) * scales[2];
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[3], q8bytes.val[3])) * scales[3];
#else
        // 如果不支持 ARM Dot Product 指令集
        // 计算第一个和第二个向量的点积，并乘以对应的比例因子，累加到 isum1 和 isum2 中
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q2bytes.val[1]), vget_high_s8(q8bytes.val[1])));
        isum1 += vaddvq_s16(p1) * scales[0];
        isum2 += vaddvq_s16(p2) * scales[1];

        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q2bytes.val[2]), vget_high_s8(q8bytes.val[2])));
        const int16x8_t p4 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q2bytes.val[3]), vget_high_s8(q8bytes.val[3])));
        isum1 += vaddvq_s16(p3) * scales[2];
        isum2 += vaddvq_s16(p4) * scales[3];
#endif
        // 将 isum1 和 isum2 的加权和乘以 d，累加到 sum 中
        sum += d * (isum1 + isum2);

    }

    *s = sum;

#elif defined __AVX2__

    // 如果支持 AVX2 指令集
    // 设置一个包含 3 的 256 位整数向量
    const __m256i m3 = _mm256_set1_epi8(3);

    // 初始化一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    uint32_t ud, um;
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;

    float summs = 0;

    // TODO: optimize this
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

    // 定义一个包含全部元素为3的__m128i类型变量m3
    const __m128i m3 = _mm_set1_epi8(3);

    // 初始化一个全为0的__m256类型变量acc
    __m256 acc = _mm256_setzero_ps();

    // 定义两个uint32_t类型变量ud和um
    uint32_t ud, um;
    // 将ud和um的地址转换为const uint8_t类型指针，并限定为只读
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;

    // 初始化一个float类型变量summs为0
    float summs = 0;

    // TODO: optimize this

    }

    // 将acc中的8个float数相加，并将结果与summs相加，赋值给*s
    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic

    // 定义一个包含2个uint32_t类型元素的数组aux32
    uint32_t aux32[2];
    // 将aux32的地址转换为const uint8_t类型指针，并赋值给scales
    const uint8_t * scales = (const uint8_t *)aux32;

    // 初始化一个float类型变量sumf为0

    }

    // 将sumf赋值给*s

#else

    // 初始化一个float类型变量sumf为0
    float sumf = 0;

    // 初始化一个int类型数组isum，包含4个元素
    int isum[4];

    // 遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 获取x[i]和y[i]的qs、scales等属性
        const uint8_t * q2 = x[i].qs;
        const  int8_t * q8 = y[i].qs;
        const uint8_t * sc = x[i].scales;

        // 初始化一个int类型变量summs为0
        int summs = 0;
        // 遍历QK_K/16次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算summs
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        // 计算dall和dmin
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 初始化isum数组为0
        isum[0] = isum[1] = isum[2] = isum[3] = 0;
        // 遍历16次
        for (int l =  0; l < 16; ++l) {
            // 计算isum数组的值
            isum[0] += q8[l+ 0] * ((q2[l] >> 0) & 3);
            isum[1] += q8[l+16] * ((q2[l] >> 2) & 3);
            isum[2] += q8[l+32] * ((q2[l] >> 4) & 3);
            isum[3] += q8[l+48] * ((q2[l] >> 6) & 3);
        }
        // 根据sc数组的值调整isum数组的值
        for (int l = 0; l < 4; ++l) {
            isum[l] *= (sc[l] & 0xF);
        }
        // 计算sumf
        sumf += dall * (isum[0] + isum[1] + isum[2] + isum[3]) - dmin * summs;
    }
    // 将sumf赋值给*s
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言n能被QK_K整除
    assert(n % QK_K == 0);

    // 定义两个掩码常量kmask1和kmask2
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 将vx和vy转换为block_q3_K和block_q8_K类型指针
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算nb
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 定义两个数组aux和utmp
    uint32_t aux[3];
    uint32_t utmp[4];

    // 定义一个包含16个元素为3的uint8x16_t类型变量m3b
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 定义一个全为0的int32x4_t类型变量vzero
// 结束条件宏定义
#endif

// 创建一个包含16个元素，每个元素都是值为1的uint8类型的向量
const uint8x16_t m0 = vdupq_n_u8(1);
// 将m0中的每个元素左移1位，得到新的向量m1
const uint8x16_t m1 = vshlq_n_u8(m0, 1);
// 将m0中的每个元素左移2位，得到新的向量m2
const uint8x16_t m2 = vshlq_n_u8(m0, 2);
// 将m0中的每个元素左移3位，得到新的向量m3
const uint8x16_t m3 = vshlq_n_u8(m0, 3);
// 创建一个值为32的int8类型常量m32
const int8_t m32 = 32;

// 创建一个包含4个int8x16_t类型向量的结构体q3bytes
int8x16x4_t q3bytes;

// 初始化一个浮点数变量sum，初始值为0
float sum = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 GGML_FP16_TO_FP32(x[i].d) 的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 获取 x[i] 的 qs、hmask 和 y[i] 的 qs
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict qh = x[i].hmask;
        const int8_t  * restrict q8 = y[i].qs;

        // 从 qh 中加载 16 个字节到 qhbits
        uint8x16x2_t qhbits = vld1q_u8_x2(qh);

        // 定义 q3h 和 isum
        uint8x16x4_t q3h;
        int32_t isum = 0;

        // 设置比例
        memcpy(aux, x[i].scales, 12);
        utmp[3] = ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4);
        utmp[2] = ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4);
        utmp[1] = (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4);
        utmp[0] = (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4);

        int8_t * scale = (int8_t *)utmp;
        for (int j = 0; j < 16; ++j) scale[j] -= m32;

        // 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从 q3 中加载 32 个字节到 q3bits，并更新 q3 指针
            const uint8x16x2_t q3bits = vld1q_u8_x2(q3); q3 += 32;
            // 从 q8 中加载 64 个字节到 q8bytes_1，并更新 q8 指针
            const int8x16x4_t q8bytes_1 = vld1q_s8_x4(q8); q8 += 64;
            // 从 q8 中加载 64 个字节到 q8bytes_2，并更新 q8 指针
            const int8x16x4_t q8bytes_2 = vld1q_s8_x4(q8); q8 += 64;

            // 计算 q3h 的值
            q3h.val[0] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[0]), 2);
            q3h.val[1] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[1]), 2);
            q3h.val[2] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[0]), 1);
            q3h.val[3] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[1]), 1);

            // 计算 q3bytes 的值
            q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[0], m3b)), vreinterpretq_s8_u8(q3h.val[0]);
            q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[1], m3b)), vreinterpretq_s8_u8(q3h.val[1]);
            q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 2), m3b)), vreinterpretq_s8_u8(q3h.val[2]);
            q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 2), m3b)), vreinterpretq_s8_u8(q3h.val[3]);
#if defined(__ARM_FEATURE_DOTPROD)
            # 如果支持 ARM Dot Product 指令集
            # 计算第一个向量和第二个向量的点积，再乘以对应的比例，累加到结果中
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes_1.val[0])) * scale[0];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes_1.val[1])) * scale[1];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes_1.val[2])) * scale[2];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes_1.val[3])) * scale[3];
#else
            # 如果不支持 ARM Dot Product 指令集
            # 计算第一个向量和第二个向量的点积，再乘以对应的比例，累加到结果中
            int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes_1.val[0])),
                                     vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes_1.val[0]));
            int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes_1.val[1])),
                                     vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes_1.val[1]));
            int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_1.val[2])),
                                     vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_1.val[2]));
            int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_1.val[3])),
                                     vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_1.val[3]));
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
#endif
            // 增加缩放值
            scale += 4;

            // 计算 q3h 的值
            q3h.val[0] = vbicq_u8(m2, qhbits.val[0]);
            q3h.val[1] = vbicq_u8(m2, qhbits.val[1]);
            q3h.val[2] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[0]), 1);
            q3h.val[3] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[1]), 1);

            // 计算 q3bytes 的值
            q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 4), m3b)), vreinterpretq_s8_u8(q3h.val[0]));
            q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 4), m3b)), vreinterpretq_s8_u8(q3h.val[1]));
            q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 6), m3b)), vreinterpretq_s8_u8(q3h.val[2]));
            q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 6), m3b)), vreinterpretq_s8_u8(q3h.val[3]));

#if defined(__ARM_FEATURE_DOTPROD)
            // 计算 isum 的值
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes_2.val[0])) * scale[0];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes_2.val[1])) * scale[1];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes_2.val[2])) * scale[2];
            isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes_2.val[3])) * scale[3];
#else
            // 如果不满足条件，执行以下代码块
            p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes_2.val[0])),
                           vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes_2.val[0]));
            // 计算两个有符号 8 位整数向量的乘积并相加
            p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes_2.val[1])),
                           vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes_2.val[1]));
            // 计算两个有符号 8 位整数向量的乘积并相加
            p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_2.val[2])),
                           vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_2.val[2]));
            // 计算两个有符号 8 位整数向量的乘积并相加
            p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_2.val[3])),
                           vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_2.val[3]));
            // 计算两个有符号 8 位整数向量的乘积并相加
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
            // 将四个有符号 16 位整数向量相加并乘以对应的比例因子，累加到 isum 中
#endif
            scale += 4;
            // 指针 scale 向后移动 4 个位置

            if (j == 0) {
                // 如果 j 等于 0
                qhbits.val[0] = vshrq_n_u8(qhbits.val[0], 4);
                // 将第一个 128 位寄存器中的每个字节右移 4 位
                qhbits.val[1] = vshrq_n_u8(qhbits.val[1], 4);
                // 将第二个 128 位寄存器中的每个字节右移 4 位
            }

        }
        // 结束内层循环

        sum += d * isum;
        // 将 d 乘以 isum 的结果累加到 sum 中

    }
    // 结束外层循环

    *s = sum;
    // 将 sum 的值赋给指针 s

#elif defined __AVX2__

    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含 32 个 3 的 256 位整数向量
    const __m256i mone = _mm256_set1_epi8(1);
    // 创建一个包含 32 个 1 的 256 位整数向量
    const __m128i m32 = _mm_set1_epi8(32);
    // 创建一个包含 16 个 32 的 128 位整数向量

    __m256 acc = _mm256_setzero_ps();
    // 创建一个全零的 256 位浮点数向量

    uint32_t aux[3];
    // 创建一个包含 3 个 32 位无符号整数的数组

    }

    *s = hsum_float_8(acc);
    // 将 acc 中的 8 个浮点数相加并将结果赋给指针 s

#elif defined __AVX__

    const __m128i m3 = _mm_set1_epi8(3);
    // 创建一个包含 16 个 3 的 128 位整数向量
    const __m128i mone = _mm_set1_epi8(1);
    // 创建一个包含 16 个 1 的 128 位整数向量
    const __m128i m32 = _mm_set1_epi8(32);
    // 创建一个包含 16 个 32 的 128 位整数向量
    const __m128i m2 = _mm_set1_epi8(2);
    // 创建一个包含 16 个 2 的 128 位整数向量

    __m256 acc = _mm256_setzero_ps();
    // 创建一个全零的 256 位浮点数向量

    const uint32_t *aux;
    // 创建一个指向无符号整数的指针

    }

    *s = hsum_float_8(acc);
    // 将 acc 中的 8 个浮点数相加并将结果赋给指针 s

#elif defined __riscv_v_intrinsic

    uint32_t aux[3];
    // 创建一个包含 3 个 32 位无符号整数的数组
    uint32_t utmp[4];
    // 创建一个包含 4 个 32 位无符号整数的数组

    float sumf = 0;
    // 创建一个浮点数 sumf 并初始化为 0

    }

    *s = sumf;
    // 将 sumf 的值赋给指针 s

#else
    // scalar version
    // This function is written like this so the compiler can manage to vectorize most of it
    // 标量版本
    // 此函数的编写方式是为了让编译器能够对大部分代码进行矢量化处理
    // 使用 -Ofast 选项，GCC 和 clang 能够生成的代码与手动矢量化版本相差不到 2 倍。
    // 我尝试的其他版本至少运行速度慢了 4 倍。
    // 理想情况是，我们只需编写一次代码，编译器就会自动产生最佳的机器指令集，而不是我们手动为 AVX、ARM_NEON 等编写矢量化版本。

    // 定义长度为 QK_K 的 int8_t 类型数组 aux8
    int8_t  aux8[QK_K];
    // 定义长度为 8 的 int16_t 类型数组 aux16
    int16_t aux16[8];
    // 定义长度为 8 的 float 类型数组 sums
    float   sums [8];
    // 定义长度为 8 的 int32_t 类型数组 aux32
    int32_t aux32[8];
    // 将 sums 数组的内容全部初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 定义长度为 4 的 uint32_t 类型数组 auxs
    uint32_t auxs[4];
    // 将 auxs 数组强制转换为 const int8_t* 类型，并赋值给 scales
    const int8_t * scales = (const int8_t*)auxs;

    // 定义一个 float 类型变量 sumf，并初始化为 0
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
#else
// 如果不是 ARM 平台，执行以下代码块
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q3_K 和 block_q8_K 类型
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算 nb，即 n/QK_K
    const int nb = n / QK_K;

#ifdef __ARM_NEON
// 如果是 ARM 平台，执行以下代码块

#ifdef __ARM_FEATURE_DOTPROD
    // 如果支持 ARM Dot Product 指令集，定义 vzero 为全零的 int32x4_t 类型变量
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 定义 m3b 和 mh 为全 3 和 4 的 uint8x16_t 类型变量
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    const uint8x16_t mh  = vdupq_n_u8(4);

    // 定义 q3bytes 为 int8x16x4_t 类型变量
    int8x16x4_t q3bytes;

    // 定义 aux16 和 scales 变量
    uint16_t aux16[2];
    int8_t * scales = (int8_t *)aux16;

    // 初始化 sum 为 0
    float sum = 0;

    // 循环处理每个 block
    for (int i = 0; i < nb; ++i) {

        // 定义 q3h 为 uint8x16x4_t 类型变量
        uint8x16x4_t q3h;

        // 读取 x[i] 的 hmask 和 qs，y[i] 的 qs
        const uint8x8_t  hbits    = vld1_u8(x[i].hmask);
        const uint8x16_t q3bits   = vld1q_u8(x[i].qs);
        const int8x16x4_t q8bytes = vld1q_s8_x4(y[i].qs);

        // 读取 x[i] 的 scales，并处理为 int8_t 类型
        const uint16_t a = *(const uint16_t *)x[i].scales;
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        // 调整 scales 数组的值
        for (int j = 0; j < 4; ++j) scales[j] -= 8;

        // 计算 isum
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        // 计算 d
        const float d = y[i].d * (float)x[i].d;

        // 处理 hbits，生成 q3h
        const uint8x16_t htmp = vcombine_u8(hbits, vshr_n_u8(hbits, 1));
        q3h.val[0] = vandq_u8(mh, vshlq_n_u8(htmp, 2));
        q3h.val[1] = vandq_u8(mh, htmp);
        q3h.val[2] = vandq_u8(mh, vshrq_n_u8(htmp, 2));
        q3h.val[3] = vandq_u8(mh, vshrq_n_u8(htmp, 4));

        // 生成 q3bytes
        q3bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q3bits, m3b),                q3h.val[0]));
        q3bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 2), m3b), q3h.val[1]));
        q3bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 4), m3b), q3h.val[2]));
        q3bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q3bits, 6),                q3h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
        // 如果支持 ARM Dot Product 指令集，则使用 vdotq_s32 计算乘积并累加到 isum 中
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes.val[0])) * scales[0];
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes.val[1])) * scales[2];
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes.val[2])) * scales[1];
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes.val[3])) * scales[3];
#else
        // 如果不支持 ARM Dot Product 指令集，则使用 vmull_s8 计算乘积并累加到 isum 中
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes.val[1])));
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes.val[2])));
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes.val[3])));
        // 将计算结果乘以对应的 scales 值并累加到 isum 中
        isum += vaddvq_s16(p0) * scales[0] + vaddvq_s16(p1) * scales[2] + vaddvq_s16(p2) * scales[1] + vaddvq_s16(p3) * scales[3];
#endif

        // 将 d 乘以 isum 的结果累加到 sum 中
        sum += d * isum;

    }

    *s = sum;

#elif defined __AVX2__

    // 设置常量 m3 和 m1
    const __m256i m3 = _mm256_set1_epi8(3);
    const __m256i m1 = _mm256_set1_epi8(1);

    // 初始化 acc 为全零
    __m256 acc = _mm256_setzero_ps();

    uint64_t aux64;

    uint16_t aux16[2];
    const int8_t * aux8 = (const int8_t *)aux16;

    }

    // 将 acc 中的值求和并存储到 *s 中
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 设置常量 m3 和 m1
    const __m128i m3 = _mm_set1_epi8(3);
    const __m128i m1 = _mm_set1_epi8(1);

    // 初始化 acc 为全零
    __m256 acc = _mm256_setzero_ps();

    uint64_t aux64;

    uint16_t aux16[2];
    const int8_t * aux8 = (const int8_t *)aux16;

    }
    # 计算浮点数数组acc的和，将结果赋值给变量s
    s = hsum_float_8(acc);
#elif defined __riscv_v_intrinsic
    // 如果定义了 __riscv_v_intrinsic，则执行以下代码块

    uint16_t aux16[2];
    // 声明一个长度为2的 uint16_t 数组 aux16
    int8_t * scales = (int8_t *)aux16;
    // 将 aux16 强制转换为 int8_t 指针，并赋值给 scales

    float sumf = 0;
    // 声明一个浮点数 sumf，并初始化为0

    }

    *s = sumf;
    // 将 sumf 的值赋给指针 s 指向的变量

#else
    // 如果未定义 __riscv_v_intrinsic，则执行以下代码块

    int8_t  aux8[QK_K];
    // 声明一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[8];
    // 声明一个长度为 8 的 int16_t 数组 aux16
    float   sums [8];
    // 声明一个长度为 8 的浮点数数组 sums
    int32_t aux32[8];
    // 声明一个长度为 8 的 int32_t 数组 aux32
    int32_t scales[4];
    // 声明一个长度为 4 的 int32_t 数组 scales
    memset(sums, 0, 8*sizeof(float));
    // 将 sums 数组的内容初始化为0

    float sumf = 0;
    // 声明一个浮点数 sumf，并初始化为0
    for (int i = 0; i < nb; ++i) {
        // 循环遍历 i 从0到 nb-1
        const uint8_t * restrict q3 = x[i].qs;
        // 声明一个指向 x[i].qs 的常量 uint8_t 指针 q3
        const uint8_t * restrict hm = x[i].hmask;
        // 声明一个指向 x[i].hmask 的常量 uint8_t 指针 hm
        const  int8_t * restrict q8 = y[i].qs;
        // 声明一个指向 y[i].qs 的常量 int8_t 指针 q8
        int8_t * restrict a = aux8;
        // 声明一个指向 aux8 的 int8_t 指针 a
        for (int l = 0; l < 8; ++l) {
            // 循环遍历 l 从0到7
            a[l+ 0] = (int8_t)((q3[l+0] >> 0) & 3) - (hm[l] & 0x01 ? 0 : 4);
            // 计算并赋值给 a[l+0]
            a[l+ 8] = (int8_t)((q3[l+8] >> 0) & 3) - (hm[l] & 0x02 ? 0 : 4);
            // 计算并赋值给 a[l+8]
            // 其余类似，计算并赋值给 a[l+16] 到 a[l+56]
        }

        scales[0] = (x[i].scales[0] & 0xF) - 8;
        // 计算并赋值给 scales[0]
        // 其余类似，计算并赋值给 scales[1] 到 scales[3]

        memset(aux32, 0, 8*sizeof(int32_t));
        // 将 aux32 数组的内容初始化为0
        for (int j = 0; j < QK_K/16; ++j) {
            // 循环遍历 j 从0到 QK_K/16-1
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 计算并赋值给 aux16
            // 其余类似，计算并赋值给 aux16，然后计算 aux32
        }
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 计算并赋值给浮点数 d
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
        // 计算 sums 数组的值
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 计算 sumf 的值
    *s = sumf;
    // 将 sumf 的值赋给指针 s 指向的变量

#endif
}
#endif

#if QK_K == 256
    // 如果 QK_K 等于 256，则执行以下代码块
// 计算两个向量的点积，其中 n 是向量长度，s 是结果数组，vx 和 vy 是输入向量
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将输入向量转换为 block_q4_K 和 block_q8_K 类型
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以 QK_K 的商
    const int nb = n / QK_K;

    // 定义用于位运算的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    // 定义临时数组
    uint32_t utmp[4];

#ifdef __ARM_NEON

    // 定义 NEON 指令需要使用的常量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
#ifdef __ARM_FEATURE_DOTPROD
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 定义用于存储结果的变量
    int8x16x2_t q4bytes;
    int8x16x2_t q8bytes;

    float sumf = 0;

    // 遍历每个 block
    for (int i = 0; i < nb; ++i) {

        // 计算 d 和 dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 计算 q8sums
        const int16x8_t q8sums = vpaddq_s16(vld1q_s16(y[i].bsums), vld1q_s16(y[i].bsums + 8));

        // 复制 x[i].scales 到 utmp
        memcpy(utmp, x[i].scales, 12);

        // 提取 utmp 中的数据并进行位运算
        uint32x2_t mins8 = { 0 };
        mins8 = vset_lane_u32(utmp[1] & kmask1, mins8, 0);
        mins8 = vset_lane_u32(((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4), mins8, 1);

        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[0] &= kmask1;

        // 转换 mins8 为 int16x8_t 类型
        const int16x8_t mins = vreinterpretq_s16_u16(vmovl_u8(vreinterpret_u8_u32(mins8)));
        // 计算 prod
        const int32x4_t prod = vaddq_s32(vmull_s16(vget_low_s16 (q8sums), vget_low_s16 (mins)),
                                         vmull_s16(vget_high_s16(q8sums), vget_high_s16(mins)));
        sumf -= dmin * vaddvq_s32(prod);

        // 将 utmp 转换为 uint8_t 类型
        const uint8_t * scales = (const uint8_t *)utmp;

        // 获取 q4 和 q8 指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        int32_t sumi1 = 0;
        int32_t sumi2 = 0;

        // 遍历每个 QK_K/64
        for (int j = 0; j < QK_K/64; ++j) {

            // 加载 q4bits
            const uint8x16x2_t q4bits = vld1q_u8_x2(q4); q4 += 32;
#ifdef __ARM_FEATURE_DOTPROD
            // 如果支持 ARM Dot Product 指令集

            // 从内存中加载 32 个有符号 8 位整数到寄存器中
            q8bytes = vld1q_s8_x2(q8); q8 += 32;
            // 将 4 位整数与掩码进行按位与操作，并转换为有符号 8 位整数
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

            // 计算两个 4 位整数向量的点积，并将结果存储在 p1 中
            const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
            // 将 p1 中的元素相加，并乘以对应的缩放因子，加到 sumi1 中
            sumi1 += vaddvq_s32(p1) * scales[2*j+0];

            // 从内存中加载 32 个有符号 8 位整数到寄存器中
            q8bytes = vld1q_s8_x2(q8); q8 += 32;
            // 将 4 位整数右移 4 位，并转换为有符号 8 位整数
            q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
            q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

            // 计算两个 4 位整数向量的点积，并将结果存储在 p2 中
            const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);

            // 将 p2 中的元素相加，并乘以对应的缩放因子，加到 sumi2 中
            sumi2 += vaddvq_s32(p2) * scales[2*j+1];
#else
            // 从内存中加载 32 个有符号 8 位整数到两个寄存器中
            q8bytes = vld1q_s8_x2(q8); q8 += 32;
            // 将 4 位整数与掩码进行按位与操作，转换为有符号 8 位整数
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
            // 计算两个 8 位整数向量的乘积和，并将结果存储在两个 16 位整数向量中
            const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1])));
            // 计算两个 16 位整数向量的和，并乘以指定的比例因子，累加到 sumi1 中
            sumi1 += vaddvq_s16(vaddq_s16(p0, p1)) * scales[2*j+0];

            // 从内存中加载 32 个有符号 8 位整数到两个寄存器中
            q8bytes = vld1q_s8_x2(q8); q8 += 32;
            // 将 4 位整数右移 4 位，并转换为有符号 8 位整数
            q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
            q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
            // 计算两个 8 位整数向量的乘积和，并将结果存储在两个 16 位整数向量中
            const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0]));
            const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1]));
            // 计算两个 16 位整数向量的和，并乘以指定的比例因子，累加到 sumi2 中
            sumi2 += vaddvq_s16(vaddq_s16(p2, p3)) * scales[2*j+1];

#endif
        }

        // 将 d 乘以 sumi1 和 sumi2 的和，累加到 sumf 中
        sumf += d * (sumi1 + sumi2);

    }

    // 将 sumf 的值赋给指针 s 指向的变量

#elif defined __AVX2__

    // 创建一个包含 16 个相同值的有符号 8 位整数向量
    const __m256i m4 = _mm256_set1_epi8(0xF);

    // 创建一个全为零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();
    }

    // 将 acc_m 的高低 128 位进行交换，并将结果与原值相加
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    // 将 acc_m 的高低 64 位进行交换，并将结果与原值相加
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    // 将 acc 向量中的 8 个浮点数求和，并将结果与 acc_m 向量中的单精度浮点数相加，将结果赋给指针 s 指向的变量

#elif defined __AVX__

    // 创建一个包含 16 个相同值的有符号 8 位整数向量
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个包含 16 个相同值的有符号 8 位整数向量
    const __m128i m2 = _mm_set1_epi8(0x2);

    // 创建一个全为零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();
    }
    # 使用指令对两个寄存器中的四个单精度浮点数进行相加，并将结果存储在第一个寄存器中
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    # 使用指令将寄存器中的两个单精度浮点数相加，并将结果存储在第一个单精度浮点数中
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    # 将寄存器中的四个单精度浮点数求和，并将结果与另一个单精度浮点数相加，最终存储在指针所指向的位置
    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);
#elif defined __riscv_v_intrinsic
    # 如果定义了 __riscv_v_intrinsic，则执行以下代码块
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 将 utmp 数组中的第一个元素的地址转换为 uint8_t 类型指针，并赋给 scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 将 utmp 数组中的第三个元素的地址转换为 uint8_t 类型指针，并赋给 mins

    float sumf = 0;
    # 初始化一个浮点数 sumf，并赋值为 0

    }

    *s = sumf;
    # 将 sumf 的值赋给指针 s 所指向的变量

#else
    # 如果未定义 __riscv_v_intrinsic，则执行以下代码块

    const uint8_t * scales = (const uint8_t*)&utmp[0];
    # 将 utmp 数组中的第一个元素的地址转换为 uint8_t 类型指针，并赋给 scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    # 将 utmp 数组中的第三个元素的地址转换为 uint8_t 类型指针，并赋给 mins

    int8_t  aux8[QK_K];
    # 声明一个长度为 QK_K 的 int8_t 类型数组 aux8
    int16_t aux16[8];
    # 声明一个长度为 8 的 int16_t 类型数组 aux16
    float   sums [8];
    # 声明一个长度为 8 的 float 类型数组 sums
    int32_t aux32[8];
    # 声明一个长度为 8 的 int32_t 类型数组 aux32
    memset(sums, 0, 8*sizeof(float));
    # 将 sums 数组的前 8 个元素初始化为 0

    float sumf = 0;
    # 初始化一个浮点数 sumf，并赋值为 0
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
// 如果不支持 ARM NEON 指令集，则使用标量计算向量点积
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将输入指针转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算块数
    const int nb = n / QK_K;

    // 如果支持 ARM NEON 指令集
#ifdef __ARM_NEON
    // 创建一个包含 16 个相同值的 uint8x16_t 类型的向量
    const uint8x16_t m4b = vdupq_n_u8(0xf);

    // 如果支持 ARM Dot Product 指令集
#ifdef __ARM_FEATURE_DOTPROD
    // 创建一个包含 4 个相同值的 int32x4_t 类型的向量
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 初始化浮点数 sumf 为 0
    float sumf = 0;

    // 创建 int8x16x2_t 和 int8x16x4_t 类型的变量
    int8x16x2_t q4bytes;
    int8x16x4_t q8bytes;

    // 初始化浮点数 sum_mins 为 0
    float sum_mins = 0.f;

    // 创建一个包含 2 个 uint16_t 类型元素的数组
    uint16_t aux16[2];
    // 将数组转换为 uint8_t 类型指针
    const uint8_t * restrict scales = (const uint8_t *)aux16;

    // 遍历每个块
    for (int i = 0; i < nb; ++i) {

        // 获取 q4 和 q8 指针
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 获取 scales 数组的值
        const uint16_t * restrict a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算 sum_mins
        const int32_t summi = scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]);
        sum_mins += y[i].d * (float)x[i].d[1] * summi;

        // 计算 d
        const float d = y[i].d * (float)x[i].d[0];

        // 加载 q4 的数据到 q4bits
        const uint8x16x2_t q4bits = vld1q_u8_x2(q4);

        // 如果支持 ARM Dot Product 指令集
#ifdef __ARM_FEATURE_DOTPROD
        // 加载 q8 的数据到 q8bytes
        q8bytes = vld1q_s8_x4(q8);
        // 将 q4bits 和 m4b 进行按位与操作
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

        // 计算点积 p1
        const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
        const int32_t sumi1 = vaddvq_s32(p1) * scales[0];

        // 将 q4bits 右移 4 位并加载到 q4bytes
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

        // 计算点积 p2
        const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[2]), q4bytes.val[1], q8bytes.val[3]);
        const int32_t sumi2 = vaddvq_s32(p2) * scales[1];
#else
        // 如果不是 AVX2 架构，则执行以下代码块

        // 从输入向量中加载 8 个有符号字节
        q8bytes = vld1q_s8_x4(q8);
        // 将输入向量中的每个字节与掩码 m4b 进行按位与操作，并转换为有符号字节
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
        // 计算两个 8 位有符号整数向量的乘积，并将结果相加
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1])));
        // 将两个 16 位有符号整数向量相加，并乘以 scales 数组中的值
        int32_t sumi1 = vaddvq_s16(vaddq_s16(p0, p1)) * scales[0];

        // 将输入向量中的每个字节右移 4 位，并转换为有符号字节
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
        // 计算两个 8 位有符号整数向量的乘积，并将结果相加
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[2]));
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[3]));
        // 将两个 16 位有符号整数向量相加，并乘以 scales 数组中的值
        int32_t sumi2 = vaddvq_s16(vaddq_s16(p2, p3)) * scales[1];

#endif
        // 将 d 乘以 sumi1 和 sumi2 的和，加到 sumf 中
        sumf += d * (sumi1 + sumi2);

    }

    // 计算最终结果并减去 sum_mins
    *s = sumf - sum_mins;

#elif defined __AVX2__

    // 如果是 AVX2 架构，则执行以下代码块

    // 创建一个包含 16 个相同值的 256 位整数向量
    const __m256i m4 = _mm256_set1_epi8(0xF);

    // 创建一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 初始化 summs 为 0
    float summs = 0;

    // 创建一个包含 2 个 uint16_t 元素的数组
    uint16_t aux16[2];
    // 将 aux16 强制转换为指向常量 uint8_t 的指针，并赋值给 scales
    const uint8_t * scales = (const uint8_t *)aux16;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算浮点数 d 和 m
        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        const __m256 vd = _mm256_set1_ps(d);

        // 从 x[i].scales 中提取数据并存储到 aux16 数组中
        const uint16_t * a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算 summs 的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 从 x[i].qs 和 y[i].qs 中提取数据
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 加载和处理 q4bits 数据
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        const __m256i q4l = _mm256_and_si256(q4bits, m4);
        const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

        // 加载和处理 q8l 和 q8h 数据
        const __m256i q8l = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8h = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算 p16l 和 p16h
        const __m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
        const __m256i p16h = _mm256_maddubs_epi16(q4h, q8h);

        // 计算 p32l 和 p32h，并更新 acc
        const __m256i p32l = _mm256_madd_epi16(_mm256_set1_epi16(scales[0]), p16l);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32l), acc);

        const __m256i p32h = _mm256_madd_epi16(_mm256_set1_epi16(scales[1]), p16h);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32h), acc);

    }

    // 计算最终结果并存储到 s 中
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
    // 创建长���为 8 的 float 数组 sums，并将其所有元素初始化为 0
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
    // 确保向量长度是 QK_K 的整数倍
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

    // 临时存储变量数组
    uint32_t utmp[4];

#ifdef __ARM_NEON

    // 定义用于 NEON 指令的常量
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    const uint8x16_t mone = vdupq_n_u8(1);
    const uint8x16_t mtwo = vdupq_n_u8(2);
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 定义用于存储向量数据的变量
    int8x16x4_t q5bytes;

    // 初始化浮点数求和变量
    float sumf = 0;

#if defined(__ARM_FEATURE_DOTPROD)
    // 计算点积并累加到 sumi 中
    sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]), q5bytes.val[1], q8bytes.val[1])) * *scales++;
    sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]), q5bytes.val[3], q8bytes.val[3])) * *scales++;
#else

            // 计算两组8位整数的乘积并相加，得到16位整数
            const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                           vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0]));
            const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                           vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1]));
            // 将两组16位整数相加并求和
            sumi += vaddvq_s16(vaddq_s16(p0, p1)) * *scales++;

            // 计算另外两组8位整数的乘积并相加，得到16位整数
            const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                           vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2]));
            const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                           vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3]));
            // 将另外两组16位整数相加并求和
            sumi += vaddvq_s16(vaddq_s16(p2, p3)) * *scales++;
#endif
        }

        // 计算最终结果
        sumf += d * sumi - dmin * sumi_mins;

    }

    *s = sumf;

#elif defined __AVX2__

    // 设置常量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m128i mzero = _mm_setzero_si128();
    const __m256i mone  = _mm256_set1_epi8(1);

    // 初始化累加器
    __m256 acc = _mm256_setzero_ps();

    float summs = 0.f;

   for (int i = 0; i < nb; ++i) {

        const uint8_t * restrict q5 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

#if QK_K == 256
        // 计算d和dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 复制缩放因子
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
    # 计算浮点数数组 acc 的和，并将结果与 summs 相加，然后赋值给变量 s
    *s = hsum_float_8(acc) + summs;
#elif defined __AVX__

    # 如果定义了 AVX，则使用 AVX 指令集

    # 创建常量 __m128i 类型的变量，每个元素都是 0xF
    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建常量 __m128i 类型的变量，所有元素都是 0
    const __m128i mzero = _mm_setzero_si128();
    # 创建常量 __m128i 类型的变量，所有元素都是 1
    const __m128i mone  = _mm_set1_epi8(1);
    # 创建常量 __m128i 类型的变量，每个元素都是 2
    const __m128i m2 = _mm_set1_epi8(2);

    # 创建 __m256 类型的变量，所有元素都是 0
    __m256 acc = _mm256_setzero_ps();

    # 创建 float 类型的变量，初始化为 0
    float summs = 0.f;

    }

    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic

    # 如果定义了 __riscv_v_intrinsic，则使用 RISC-V 指令集

    # 将 utmp 数组中的元素转换为 uint8_t 类型指针
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    # 创建 float 类型的变量，初始化为 0
    float sumf = 0;
    # 创建 float 类型的变量，初始化为 0
    float sums = 0.0;

    # 创建 size_t 类型的变量
    size_t vl;

    }

    *s = sumf+sums;

#else

    # 如果未定义 AVX 和 RISC-V 指令集，则执行以下代码块

    # 将 utmp 数组中的元素转换为 uint8_t 类型指针
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    # 创建 int8_t 类型数组，长度为 QK_K
    int8_t  aux8[QK_K];
    # 创建 int16_t 类型数组，长度为 8
    int16_t aux16[8];
    # 创建 float 类型数组，长度为 8，初始化为 0
    float   sums [8];
    # 创建 int32_t 类型数组，长度为 8
    int32_t aux32[8];
    # 将 sums 数组中的元素全部设置为 0
    memset(sums, 0, 8*sizeof(float));

    # 创建 float 类型的变量，初始化为 0
    float sumf = 0;
    }
    # 计算 sums 数组中所有元素的和
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    # 将计算结果赋值给 s
    *s = sumf;
#endif
}

#else

void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 断言 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    # 将 vx 和 vy 强制转换为 block_q5_K 和 block_q8_K 类型指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    # 计算块的数量
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    # 如果定义了 ARM NEON，则使用 ARM NEON 指令集

    # 创建 uint8x16_t 类型的变量，每个元素都是 0xf
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    # 创建 uint8x16_t 类型的变量，每个元素都是 16
    const uint8x16_t mh = vdupq_n_u8(16);
    # 如果支持 ARM Dot Product 指令，则创建 int32x4_t 类型的变量，每个元素都是 0
    # 否则不创建该变量
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    # 创建 int8x16x4_t 和 uint8x16x4_t 类型的变量
    int8x16x4_t q5bytes;
    uint8x16x4_t q5h;

    # 创建 float 类型的变量，初始化为 0
    float sumf = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * (float)x[i].d;
        // 获取 x[i] 的 scales 数组
        const int8_t * sc = x[i].scales;

        // 获取 x[i] 的 qs 数组
        const uint8_t * restrict q5 = x[i].qs;
        // 获取 x[i] 的 qh 数组
        const uint8_t * restrict qh = x[i].qh;
        // 获取 y[i] 的 qs 数组
        const int8_t  * restrict q8 = y[i].qs;

        // 从 qh 中加载一个 uint8x8_t 类型的值
        const uint8x8_t qhbits = vld1_u8(qh);

        // 从 q5 中加载两个 uint8x16_t 类型的值
        const uint8x16x2_t q5bits = vld1q_u8_x2(q5);
        // 从 q8 中加载四个 int8x16_t 类型的值
        const int8x16x4_t q8bytes = vld1q_s8_x4(q8);

        // 将 qhbits 的值组合成一个 uint8x16_t 类型的值
        const uint8x16_t htmp = vcombine_u8(qhbits, vshr_n_u8(qhbits, 1));
        // 计算 q5h.val[0] 到 q5h.val[3] 的值
        q5h.val[0] = vbicq_u8(mh, vshlq_n_u8(htmp, 4));
        q5h.val[1] = vbicq_u8(mh, vshlq_n_u8(htmp, 2));
        q5h.val[2] = vbicq_u8(mh, htmp);
        q5h.val[3] = vbicq_u8(mh, vshrq_n_u8(htmp, 2));

        // 计算 q5bytes.val[0] 到 q5bytes.val[3] 的值
        q5bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[0], m4b)), vreinterpretq_s8_u8(q5h.val[0]));
        q5bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[1], m4b)), vreinterpretq_s8_u8(q5h.val[1]);
        q5bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[0], 4)), vreinterpretq_s8_u8(q5h.val[2]));
        q5bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[1], 4)), vreinterpretq_s8_u8(q5h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM Dot Product 指令集

        // 计算第一个部分的加权和
        int32_t sumi1 = sc[0] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]));
        int32_t sumi2 = sc[1] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[1], q8bytes.val[1]));
        int32_t sumi3 = sc[2] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]));
        int32_t sumi4 = sc[3] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[3], q8bytes.val[3]));

        // 更新总和
        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);

#else
    // 如果不支持 ARM Dot Product 指令集

        // 计算第一个部分的乘积和
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0]));
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1]));
        int32_t sumi = sc[0] * vaddvq_s16(p0) + sc[1] * vaddvq_s16(p1);

        // 计算第二个部分的乘积和
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2]));
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        sumi += sc[2] * vaddvq_s16(p2) + sc[3] * vaddvq_s16(p3);

        // 更新总和
        sumf += d * sumi;
#endif

    }

    *s = sumf;

#elif defined __AVX2__
    // 如果支持 AVX2 指令集

    // 创建常量向量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i mone  = _mm256_set1_epi8(1);

    // 初始化累加器
    __m256 acc = _mm256_setzero_ps();
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 的 qs 指针
        const uint8_t * restrict q5 = x[i].qs;
        // 获取 y[i] 的 qs 指针
        const int8_t  * restrict q8 = y[i].qs;

        // 计算 d 值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 加载 q5 到 256 位寄存器
        const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);

        // 设置 scale_l 和 scale_h
        const __m256i scale_l = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[1]), _mm_set1_epi16(x[i].scales[0]));
        const __m256i scale_h = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[3]), _mm_set1_epi16(x[i].scales[2]));

        // 复制 x[i].qh 到 aux64，然后转换为 128 位寄存器
        int64_t aux64;
        memcpy(&aux64, x[i].qh, 8);
        const __m128i haux128 = _mm_set_epi64x(aux64 >> 1, aux64);
        const __m256i haux256 = MM256_SET_M128I(_mm_srli_epi16(haux128, 2), haux128);

        // 计算 q5h_0 和 q5h_1
        const __m256i q5h_0 = _mm256_slli_epi16(_mm256_andnot_si256(haux256, mone), 4);
        const __m256i q5h_1 = _mm256_slli_epi16(_mm256_andnot_si256(_mm256_srli_epi16(haux256, 4), mone), 4);

        // 计算 q5l_0 和 q5l_1
        const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);
        const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);

        // 加载 q8_0 和 q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算 p16_0、p16_1、s16_0、s16_1
        const __m256i p16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5l_0, q8_0));
        const __m256i p16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5l_1, q8_1));
        const __m256i s16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5h_0, q8_0));
        const __m256i s16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5h_1, q8_1));

        // 计算 dot
        const __m256i dot = _mm256_sub_epi32(_mm256_add_epi32(p16_0, p16_1), _mm256_add_epi32(s16_0, s16_1));

        // 计算累加结果
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(dot), acc);

    }

    // 将累加结果转换为 float 类型，存储到 s 指针指向的位置
    *s = hsum_float_8(acc);
#elif defined __AVX__

    #ifdef __AVX__ 宏定义条件编译，表示代码将使用 AVX 指令集
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个包含 0xF 的 128 位整数向量
    const __m128i mone  = _mm_set1_epi8(1);
    // 创建一个包含 1 的 128 位整数向量

    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含全 0 的 256 位浮点数向量

    }

    *s = hsum_float_8(acc);
    // 调用 hsum_float_8 函数，将 256 位浮点数向量求和，并将结果存储到 s 中

#elif defined __riscv_v_intrinsic

    #ifdef __riscv_v_intrinsic 宏定义条件编译，表示代码将使用 RISC-V 指令集
    float sumf = 0;
    // 初始化一个浮点数 sumf 为 0

    }

    *s = sumf;
    // 将 sumf 的值赋给 s

#else

    #ifndef __AVX__ 和 __riscv_v_intrinsic 宏未定义时执行以下代码块
    int8_t aux8[QK_K];
    // 创建一个长度为 QK_K 的 int8_t 数组 aux8
    int16_t aux16[16];
    // 创建一个长度为 16 的 int16_t 数组 aux16
    float   sums [8];
    // 创建一个长度为 8 的 float 数组 sums，并初始化为 0
    memset(sums, 0, 8*sizeof(float));
    // 将 sums 数组的所有元素初始化为 0

    float sumf = 0;
    // 初始化一个浮点数 sumf 为 0
    for (int i = 0; i < nb; ++i) {
        // 循环遍历 nb 次
        const uint8_t * restrict q4 = x[i].qs;
        // 定义一个指向 x[i].qs 的 uint8_t 指针 q4
        const uint8_t * restrict hm = x[i].qh;
        // 定义一个指向 x[i].qh 的 uint8_t 指针 hm
        const  int8_t * restrict q8 = y[i].qs;
        // 定义一个指向 y[i].qs 的 int8_t 指针 q8
        int8_t * restrict a = aux8;
        // 定义一个指向 aux8 的 int8_t 指针 a
        for (int l = 0; l < 32; ++l) {
            // 循环遍历 32 次
            a[l+ 0] = q4[l] & 0xF;
            // 将 q4[l] 与 0xF 进行按位与操作，并赋值给 a[l+0]
            a[l+32] = q4[l]  >> 4;
            // 将 q4[l] 右移 4 位后的结果赋值给 a[l+32]
        }
        for (int is = 0; is < 8; ++is) {
            // 循环遍历 8 次
            uint8_t m = 1 << is;
            // 计算 1 左移 is 位的结果
            for (int l = 0; l < 8; ++l) a[8*is + l] -= (hm[l] & m ? 0 : 16);
            // 根据条件判断，对 a[8*is + l] 进行减法操作
        }

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 d 的值
        const int8_t * restrict sc = x[i].scales;
        // 定义一个指向 x[i].scales 的 int8_t 指针 sc

        for (int j = 0; j < QK_K/16; ++j) {
            // 循环遍历 QK_K/16 次
            const float dl = d * sc[j];
            // 计算 dl 的值
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            // 计算 aux16 数组的值
            for (int l = 0; l <  8; ++l) sums[l] += dl * (aux16[l] + aux16[8+l]);
            // 更新 sums 数组的值
            q8 += 16; a += 16;
            // 更新 q8 和 a 的指针位置
        }
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sums 数组的值累加到 sumf 中
    *s = sumf;
    // 将 sumf 的值赋给 s
#endif
}
#endif


#if QK_K == 256
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义一个函数 ggml_vec_dot_q6_K_q8_K，参数为 n, s, vx, vy
    assert(n % QK_K == 0);
    // 断言 n 能被 QK_K 整除

    const block_q6_K * restrict x = vx;
    // 定义一个指向 vx 的 block_q6_K 指针 x
    const block_q8_K * restrict y = vy;
    // 定义一个指向 vy 的 block_q8_K 指针 y

    const int nb = n / QK_K;
    // 计算 nb 的值为 n 除以 QK_K 的结果

#ifdef __ARM_NEON

    #ifdef __ARM_NEON 宏定义条件编译，表示代码将使用 ARM NEON 指令集
    float sum = 0;
    // 初始化一个浮点数 sum 为 0

    const uint8x16_t m4b = vdupq_n_u8(0xF);
    // 创建一个包含 0xF 的 16 位无符号整数向量
    #if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
    #endif
    // 创建一个包含 0 的 32 位有符号整数向量
    //const int8x16_t  m32s = vdupq_n_s8(32);

    const uint8x16_t mone = vdupq_n_u8(3);
    // 创建一个包含 3 的 16 位无符号整数向量

    int8x16x4_t q6bytes;
    // 定义一个 int8x16x4_t 类型的变量 q6bytes
    uint8x16x4_t q6h;
    // 定义一个 uint8x16x4_t 类型的变量 q6h
#if defined(__ARM_FEATURE_DOTPROD)
    #ifdef __ARM_FEATURE_DOTPROD 宏定义判断是否支持 ARM Dot Product 指令集

            // 计算四个向量的点积并乘以对应的缩放因子，累加到 isum 中
            isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
            // 更新缩放因子指针
            scale += 4;

    #else
    #ifdef __ARM_FEATURE_DOTPROD 宏定义不成立时执行以下代码块

            // 计算第一个向量的点积并乘以对应的缩放因子
            int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                     vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0]));
            // 计算第二个向量的点积并乘以对应的缩放因子
            int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                     vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1]));
            // 将点积结果乘以缩放因子后累加到 isum 中
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
            // 更新缩放因子指针
            scale += 2;

            // 计算第三个向量的点积并乘以对应的缩放因子
            int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                     vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2]));
            // 计算第四个向量的点积并乘以对应的缩放因子
            int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                     vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3]));
            // 将点积结果乘以缩放因子后累加到 isum 中
            isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
            // 更新缩放因子指针
            scale += 2;

    #endif
#endif
#endif

            // 从指针 q8 处加载 64 个有符号 8 位整数到寄存器 q8bytes
            q8bytes = vld1q_s8_x4(q8); q8 += 64;

            // 将 qhbits.val[0] 右移 4 位，存储到变量 shifted
            shifted = vshrq_n_u8(qhbits.val[0], 4);
            // 将 shifted 中的低 4 位左移 4 位，存储到 q6h.val[0]
            q6h.val[0] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
            // 重复上述过程，分别存储到 q6h.val[1]、q6h.val[2]、q6h.val[3]
            shifted = vshrq_n_u8(qhbits.val[1], 4);
            q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
            shifted = vshrq_n_u8(qhbits.val[0], 6);
            q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
            shifted = vshrq_n_u8(qhbits.val[1], 6);
            q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

            // 将 q6bits 右移 4 位，与 q6h 进行或运算，存储到 q6bytes
            q6bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[0]));
            q6bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[1]));
            q6bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[2], 4), q6h.val[2]));
            q6bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[3], 4), q6h.val[3]));
#if defined(__ARM_FEATURE_DOTPROD)
// 如果支持 ARM Dot Product 指令集，则执行以下代码块
            // 计算四个向量的点积并乘以对应的标量，累加到 isum 中
            isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
            // 更新 scale 指针位置
            scale += 4;

            // 循环计算四个向量的点积并乘以对应的标量，累加到 isum 中
            //for (int l = 0; l < 4; ++l) {
            //    const int32x4_t p = vdotq_s32(vzero, q6bytes.val[l], q8bytes.val[l]);
            //    isum += vaddvq_s32(p) * *scale++;
            //}
#else
// 如果不支持 ARM Dot Product 指令集，则执行以下代码块
            // 计算两个向量的点积并乘以对应的标量，累加到 isum 中
            p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                    vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                    vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1])));
            isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
            // 更新 scale 指针位置
            scale += 2;

            // 计算两个向量的点积并乘以对应的标量，累加到 isum 中
            p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                    vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2]));
            p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                    vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3]));
            isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
            // 更新 scale 指针位置
            scale += 2;
#endif

        }
        // 计算最终的 isum 乘以 d_all 乘以 y[i].d，累加到 sum 中
        sum += d_all * y[i].d * (isum - 32 * isum_mins);

    }
    *s = sum;

#elif defined __AVX2__
// 如果支持 AVX2 指令集，则执行以下代码块

    // 初始化常量向量
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i m2 = _mm256_set1_epi8(3);
    const __m256i m32s = _mm256_set1_epi8(32);

    // 初始化累加器向量
    __m256 acc = _mm256_setzero_ps();
    }
    # 将累加器中的值转换为浮点数，并赋值给指针 s 所指向的变量
    *s = hsum_float_8(acc);
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
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 确保 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q6_K 和 block_q8_K 类型的指针
    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算块数
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 初始化求和变量 sum
    float sum = 0;

    // 创建常量向量
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    const int8x16_t  m32s = vdupq_n_s8(32);
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    const uint8x16_t mone = vdupq_n_u8(3);

    // 初始化变量
    int8x16x4_t q6bytes;
    uint8x16x4_t q6h;

    // 遍历每个块
    for (int i = 0; i < nb; ++i) {

        // 获取当前块的 d 值
        const float d_all = (float)x[i].d;

        // 获取当前块的 ql, qh, qs 和 scales
        const uint8_t * restrict q6 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const int8_t  * restrict q8 = y[i].qs;

        const int8_t * restrict scale = x[i].scales;

        // 初始化求和变量 isum
        int32_t isum = 0;

        // 加载 qhbits, q6bits 和 q8bytes
        uint8x16_t   qhbits = vld1q_u8(qh);
        uint8x16x2_t q6bits = vld1q_u8_x2(q6);
        int8x16x4_t q8bytes = vld1q_s8_x4(q8);

        // 计算 q6h 和 q6bytes
        q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits), 4);
        uint8x16_t shifted = vshrq_n_u8(qhbits, 2);
        q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 4);
        q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
        shifted = vshrq_n_u8(qhbits, 6);
        q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

        q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);
        q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);
        q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[2])), m32s);
        q6bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[3])), m32s);
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持 ARM Dot Product 指令集，则使用 vdotq_s32 计算乘积并累加
    isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
            vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
#else
    // 否则使用 vmull_s8 和 vaddq_s16 计算乘积并累加
    int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                             vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
    int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                             vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1]));
    // 计算乘积并累加
    isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];

    int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                             vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2])));
    int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                             vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3]));
    // 计算乘积并累加
    isum += vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
#endif

    // 计算最终结果并乘以相关系数
    sum += isum * d_all * y[i].d;

}

// 根据不同的编译选项选择不同的处理方式
#elif defined __AVX2__
    // 使用 AVX2 指令集进行计算
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i m2 = _mm256_set1_epi8(3);
    const __m256i m32s = _mm256_set1_epi8(32);

    __m256 acc = _mm256_setzero_ps();

    // 计算累加和
    *s = hsum_float_8(acc);

#elif defined __AVX__
    // 使用 AVX 指令集进行计算
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(3);
    const __m128i m32s = _mm_set1_epi8(32);

    __m256 acc = _mm256_setzero_ps();

    // 计算累加和
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic
    // 使用 RISC-V 指令集进行计算
    float sumf = 0;

    // 计算最终结果
    *s = sumf;

#else
    // 其他情况下的处理方式
    // 定义一个长度为 QK_K 的 int8_t 数组
    int8_t  aux8[QK_K];
    // 定义一个长度为 8 的 int16_t 数组
    int16_t aux16[8];
    // 定义一个长度为 8 的 float 数组
    float   sums [8];
    // 定义一个长度为 8 的 int32_t 数组
    int32_t aux32[8];
    // 将 sums 数组中的所有元素初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 初始化一个 float 类型的变量 sumf 为 0
    float sumf = 0;
    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 获取 x[i] 中的 ql、qh、qs 指针
        const uint8_t * restrict q4 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const  int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组中的所有元素初始化为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        // 获取指向 aux8 数组的指针 a
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
            // 获取 scale 值
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
    // 计算 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 指向的变量
    *s = sumf;
#ifdef
#endif



#ifdef
#endif



#ifdef
#endif
```