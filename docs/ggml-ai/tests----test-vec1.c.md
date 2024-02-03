# `ggml\tests\test-vec1.c`

```cpp
#include <stdint.h>  // 包含标准整数类型的头文件
#include <stdio.h>   // 包含标准输入输出的头文件
#include <assert.h>  // 包含断言的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <time.h>    // 包含时间处理函数的头文件

#include <sys/time.h>  // 包含系统时间处理函数的头文件

#include <immintrin.h>  // 包含 AVX 指令集的头文件

const int N = 1 << 14;  // 定义常量 N 为 2 的 14 次方
const int M = 768;      // 定义常量 M 为 768

//
// naive implementation
//

void mul_mat_vec_f32_0(
    const float * restrict src0,  // 源矩阵指针
    const float * restrict src1,  // 源向量指针
    float * dst,                   // 目标向量指针
    int nrows,                     // 矩阵行数
    int ncols) {                   // 矩阵列数
    for (int i = 0; i < nrows; i++) {  // 遍历矩阵行
        float sum = 0.0f;               // 初始化求和变量
        for (int j = 0; j < ncols; j++) {  // 遍历矩阵列
            sum += src0[i*ncols + j]*src1[j];  // 矩阵与向量相乘并累加
        }
        dst[i] = sum;  // 将结果存入目标向量
    }
}

//
// SIMD with 8 32-bit floats
//

float reduce_vector8_0(__m256 v) {  // 定义函数用于对 8 个 32 位浮点数向量求和
    __m128 v1 = _mm256_extractf128_ps(v, 0);  // 提取 v 的低 128 位
    __m128 v2 = _mm256_extractf128_ps(v, 1);  // 提取 v 的高 128 位
    __m128 v3 = _mm_add_ps(v1, v2);           // 低 128 位和高 128 位相加
    __m128 v4 = _mm_shuffle_ps(v3, v3, 0x4e);  // 按指定模式进行数据交换
    __m128 v5 = _mm_add_ps(v3, v4);            // 求和
    __m128 v6 = _mm_shuffle_ps(v5, v5, 0x11);  // 按指定模式进行数据交换
    __m128 v7 = _mm_add_ps(v5, v6);            // 求和
    return _mm_cvtss_f32(v7);                  // 将结果转换为 float 类型并返回
}

// vectorized implementation using AVX
void mul_mat_vec_f32_1(
    const float * restrict src0,  // 源矩阵指针
    const float * restrict src1,  // 源向量指针
    float * dst,                   // 目标向量指针
    int nrows,                     // 矩阵行数
    int ncols) {                   // 矩阵列数

    const int ncols8 = ncols & ~7;  // 计算 ncols 的最大的 8 的倍数

    for (int i = 0; i < nrows; i++) {  // 遍历矩阵行
        __m256 sum = _mm256_setzero_ps();  // 初始化求和向量
        for (int j = 0; j < ncols8; j += 8) {  // 遍历矩阵列，每次处理 8 个元素
            __m256 a = _mm256_loadu_ps(src0 + i*ncols + j);  // 加载矩阵中的数据到向量
            __m256 b = _mm256_loadu_ps(src1 + j);            // 加载向量中的数据到向量
            __m256 c = _mm256_mul_ps(a, b);                  // 向量相乘
            sum = _mm256_add_ps(sum, c);                     // 求和
        }
        dst[i] = reduce_vector8_0(sum);  // 对求和向量进行求和并存入目标向量

        for (int j = ncols8; j < ncols; j++) {  // 处理剩余的元素
            dst[i] += src0[i*ncols + j]*src1[j];  // 矩阵与向量相乘并累加
        }
    }
}

void mul_mat_vec_f32_2(
    const float * restrict src0,  // 源矩阵指针
    const float * restrict src1,  // 源向量指针
    float * dst,                   // 目标向量指针
    int nrows,                     // 矩阵行数
    int ncols) {                   // 矩阵列数

    const int ncols32 = ncols & ~31;  // 计算 ncols 的最大的 32 的倍数
    // 循环遍历每一行
    for (int i = 0; i < nrows; i++) {
        // 初始化四个 256 位浮点数变量，值为 0
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        // 获取源数据行的指针
        const float * restrict src0_row = src0 + i*ncols;
        // 循环遍历每 32 列
        for (int j = 0; j < ncols32; j += 32) {
            // 从源数据行中加载 32 个单精度浮点数到 256 位浮点数变量
            __m256 a0 = _mm256_loadu_ps(src0_row + j + 0);
            __m256 a1 = _mm256_loadu_ps(src0_row + j + 8);
            __m256 a2 = _mm256_loadu_ps(src0_row + j + 16);
            __m256 a3 = _mm256_loadu_ps(src0_row + j + 24);
            // 从第二个源数据行中加载 32 个单精度浮点数到 256 位浮点数变量
            __m256 b0 = _mm256_loadu_ps(src1 + j + 0);
            __m256 b1 = _mm256_loadu_ps(src1 + j + 8);
            __m256 b2 = _mm256_loadu_ps(src1 + j + 16);
            __m256 b3 = _mm256_loadu_ps(src1 + j + 24);
#if defined(__FMA__)
            // 如果定义了__FMA__，则使用 FMA 指令进行乘加运算
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            // 如果未定义__FMA__，则使用乘法和加法指令进行运算
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        // 将累加和进行向量规约操作，得到一个标量值，存入dst[i]
        dst[i] = reduce_vector8_0(_mm256_add_ps(_mm256_add_ps(sum0, sum1), _mm256_add_ps(sum2, sum3)));

        // 对于剩余的列，使用标量乘法和加法进行运算
        for (int j = ncols32; j < ncols; j++) {
            dst[i] += src0[i*ncols + j]*src1[j];
        }
    }
}

//
// SIMD with 8 16-bit floats
//

// 将 32 位无符号整数转换为单精度浮点数
static inline float fp32_from_bits(uint32_t w) {
    // 根据不同的编译器和平台，使用不同的方式进行转换
    #if defined(__OPENCL_VERSION__)
    return as_float(w);
    #elif defined(__CUDA_ARCH__)
    return __uint_as_float((unsigned int) w);
    #elif defined(__INTEL_COMPILER)
    return _castu32_f32(w);
    #elif defined(_MSC_VER) && (defined(_M_ARM) || defined(_M_ARM64))
    return _CopyFloatFromInt32((__int32) w);
    #else
    union {
        uint32_t as_bits;
        float as_value;
    } fp32 = { w };
    return fp32.as_value;
    #endif
}

// 将单精度浮点数转换为 32 位无符号整数
static inline uint32_t fp32_to_bits(float f) {
    // 根据不同的编译器和平台，使用不同的方式进行转换
    #if defined(__OPENCL_VERSION__)
    return as_uint(f);
    #elif defined(__CUDA_ARCH__)
    return (uint32_t) __float_as_uint(f);
    #elif defined(__INTEL_COMPILER)
    return _castf32_u32(f);
    #elif defined(_MSC_VER) && (defined(_M_ARM) || defined(_M_ARM64))
    return (uint32_t) _CopyInt32FromFloat(f);
    #else
    union {
        float as_value;
        uint32_t as_bits;
    } fp32 = { f };
    return fp32.as_bits;
    #endif
}
/*
 * 将以IEEE半精度格式表示的16位浮点数（位表示）转换为以IEEE单精度格式表示的32位浮点数。
 *
 * @note 该实现依赖于类似IEEE的（不假设舍入模式，不对非规格化数进行操作）浮点运算和整数与浮点变量之间的位转换。
 */
static inline float fp16_ieee_to_fp32_value(uint16_t h) {
    /*
     * 将半精度浮点数扩展为32位并移位到32位字的高位：
     *      +---+-----+------------+-------------------+
     *      | S |EEEEE|MM MMMM MMMM|0000 0000 0000 0000|
     *      +---+-----+------------+-------------------+
     * 位   31  26-30    16-25            0-15
     *
     * S - 符号位，E - 偏置指数的位，M - 尾数的位，0 - 零位。
     */
    const uint32_t w = (uint32_t) h << 16;
    /*
     * 将输入数的符号提取到32位字的高位：
     *
     *      +---+----------------------------------+
     *      | S |0000000 00000000 00000000 00000000|
     *      +---+----------------------------------+
     * 位   31                 0-31
     */
    const uint32_t sign = w & UINT32_C(0x80000000);
    /*
     * 将输入数的尾数和偏置指数提取到32位字的高位：
     *
     *      +-----+------------+---------------------+
     *      |EEEEE|MM MMMM MMMM|0 0000 0000 0000 0000|
     *      +-----+------------+---------------------+
     * 位   27-31    17-26            0-16
     */
    const uint32_t two_w = w + w;
    /*
     * 将尾数和指数移位到位23-28和位13-22，使它们成为单精度浮点数的尾数和指数：
     *
     *       S|Exponent |          Mantissa
     *      +-+---+-----+------------+----------------+
     *      |0|000|EEEEE|MM MMMM MMMM|0 0000 0000 0000|
     *      +-+---+-----+------------+----------------+
     * Bits   | 23-31   |           0-22
     *
     * 接下来，对指数进行一些调整：
     * - 指数需要根据单精度和半精度格式之间的指数偏差进行校正（0x7F - 0xF = 0x70）
     * - 输入中的Inf和NaN值在转换为单精度数后应保持为Inf和NaN值。
     *   因此，如果半精度输入的偏置指数为0x1F（最大可能值），则单精度输出的偏置指数必须为0xFF（最大可能值）。
     *   我们通过两个步骤进行此校正：
     *   - 首先，我们通过（0xFF - 0x1F）= 0xE0（参见exp_offset）来调整指数，而不是通过上面指数偏差的差异建议的0x70。
     *   - 然后，我们将指数调整的单精度结果乘以2 **（-112）以撤消0xE0减去0x70的必要指数调整的影响。
     *     浮点乘法硬件将确保Inf和NaN在至少部分符合IEEE754的实现上保持其值。
     *
     * 请注意，上述操作不处理非规格化输入（其中偏置指数== 0）。但是，它们也不对非规格化输入进行操作，也不产生非规格化结果。
     */
    const uint32_t exp_offset = UINT32_C(0xE0) << 23;
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    # 如果满足条件：C 标准版本大于等于 C99 或者是 GNU 编译器且不是严格 ANSI 模式
    const float exp_scale = 0x1.0p-112f;
#else
    # 否则
    const float exp_scale = fp32_from_bits(UINT32_C(0x7800000));
#endif
    # 根据条件设置 exp_scale 常量的值

    # 计算 normalized_value 常量的值
    const float normalized_value = fp32_from_bits((two_w >> 4) + exp_offset) * exp_scale;
    /*
     * 将非规格化的半精度输入转换为单精度结果（始终是规格化的）。
     * 零输入也在此处处理。
     *
     * 在非规格化数中，偏置指数为零，尾数具有非零位。
     * 首先，我们将尾数移位到32位字的位0-9。
     *
     *                  零               |  尾数
     *      +---------------------------+------------+
     *      |0000 0000 0000 0000 0000 00|MM MMMM MMMM|
     *      +---------------------------+------------+
     * 位               10-31                0-9
     *
     * 现在，记住非规格化的半精度数表示为：
     *    FP16 = 尾数 * 2**(-24)。
     * 技巧是构造一个具有相同尾数和半精度输入的规格化单精度数，并且具有将相应尾数位缩放到2**(-24)的指数。
     * 规格化的单精度浮点数表示为：
     *    FP32 = (1 + 尾数 * 2**(-23)) * 2**(指数 - 127)
     * 因此，当偏置指数为126时，输入非规格化半精度数的尾数的单位变化会导致构造的单精度数变化为2**(-24)，即相同的量。
     *
     * 最后一步是调整构造的单精度数的偏置。当输入半精度数为零时，构造的单精度数的值为
     *    FP32 = 1 * 2**(126 - 127) = 2**(-1) = 0.5
     * 因此，我们需要从构造的单精度数中减去0.5，以获得输入半精度数的数值等价物。
     */
    const uint32_t magic_mask = UINT32_C(126) << 23;
    const float magic_bias = 0.5f;
    const float denormalized_value = fp32_from_bits((two_w >> 17) | magic_mask) - magic_bias;
    /*
     * - 根据输入的指数，选择将输入转换为规格化数还是非规格化数。变量 two_w 包含输入指数的位 27-31，因此如果它小于 2**27，
     *   则输入要么是非规格化数，要么是零。
     * - 将指数和尾数的转换结果与输入数字的符号组合起来。
     */
    // 定义非规格化数的截断值
    const uint32_t denormalized_cutoff = UINT32_C(1) << 27;
    // 将结果组合为带符号的 32 位浮点数
    const uint32_t result = sign |
        (two_w < denormalized_cutoff ? fp32_to_bits(denormalized_value) : fp32_to_bits(normalized_value));
    // 返回 32 位浮点数
    return fp32_from_bits(result);
// 结尾的大括号，表示函数或代码块的结束
}

/*
 * 将 IEEE 单精度格式的 32 位浮点数转换为 IEEE 半精度格式的 16 位浮点数，以位表示
 *
 * @note 该实现依赖于 IEEE 类似的（不假设舍入模式，不对非规格化数进行操作）浮点运算和整数与浮点变量之间的位转换。
 */
static inline uint16_t fp16_ieee_from_fp32_value(float f) {
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    // 定义常量 scale_to_inf 和 scale_to_zero，用于转换浮点数
    const float scale_to_inf = 0x1.0p+112f;
    const float scale_to_zero = 0x1.0p-110f;
#else
    // 定义常量 scale_to_inf 和 scale_to_zero，用于转换浮点数
    const float scale_to_inf = fp32_from_bits(UINT32_C(0x77800000));
    const float scale_to_zero = fp32_from_bits(UINT32_C(0x08800000));
#endif
    // 计算 base 值
    float base = (fabsf(f) * scale_to_inf) * scale_to_zero;

    // 将浮点数 f 转换为 32 位整数
    const uint32_t w = fp32_to_bits(f);
    // 左移一位的 w 值
    const uint32_t shl1_w = w + w;
    // 提取符号位
    const uint32_t sign = w & UINT32_C(0x80000000);
    // 计算偏置值
    uint32_t bias = shl1_w & UINT32_C(0xFF000000);
    if (bias < UINT32_C(0x71000000)) {
        bias = UINT32_C(0x71000000);
    }

    // 计算 base 值
    base = fp32_from_bits((bias >> 1) + UINT32_C(0x07800000)) + base;
    // 将 base 转换为 32 位整数
    const uint32_t bits = fp32_to_bits(base);
    // 提取指数部分的位
    const uint32_t exp_bits = (bits >> 13) & UINT32_C(0x00007C00);
    // 提取尾数部分的位
    const uint32_t mantissa_bits = bits & UINT32_C(0x00000FFF);
    // 计算非符号位
    const uint32_t nonsign = exp_bits + mantissa_bits;
    // 返回最终的 16 位浮点数值
    return (sign >> 16) | (shl1_w > UINT32_C(0xFF000000) ? UINT16_C(0x7E00) : nonsign);
}

// 乘以矩阵和向量的函数，使用 IEEE 半精度格式
void mul_mat_vec_f16_0(
    const uint16_t * src0,  // 输入矩阵
    const uint16_t * src1,  // 输入向量
             float * dst,   // 输出向量
    int nrows,              // 矩阵的行数
    int ncols) {            // 矩阵的列数

    const int ncols8 = ncols & ~7;  // 计算 ncols 的最接近的 8 的倍数的值
    // 遍历行
    for (int i = 0; i < nrows; i++) {
        // 初始化 AVX 寄存器为全零
        __m256 sum = _mm256_setzero_ps();

        // 获取源数据行的指针
        const uint16_t * src0_row = src0 + i * ncols;
        // 遍历列，每次处理8个元素
        for (int j = 0; j < ncols8; j += 8) {
            // 从内存加载16位整数，转换成32位浮点数，存入 AVX 寄存器 a
            __m256 a = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j)));
            // 从内存加载16位整数，转换成32位浮点数，存入 AVX 寄存器 b
            __m256 b = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
// 如果定义了__FMA__，则使用FMA指令进行乘加操作，将结果累加到sum0和sum1中
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
// 否则，使用乘法和加法指令进行操作，将结果累加到sum0和sum1中
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
// 将累加结果存储到dst数组中
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1);

// 处理剩余的不足16的列
        for (int j = ncols16; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

// 对16位无符号整数数组src0和src1进行矩阵向量乘法，结果存储到dst数组中
void mul_mat_vec_f16_1(
    const uint16_t * src0,
    const uint16_t * src1,
             float * dst,
    int nrows,
    int ncols) {

    // 计算不足16的列数
    const int ncols16 = ncols & ~15;

    // 遍历矩阵的行
    for (int i = 0; i < nrows; i++) {
        // 初始化两个累加寄存器为0
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();

        // 获取src0的当前行
        const uint16_t * src0_row = src0 + i * ncols;
        // 遍历16列，每次处理16个元素
        for (int j = 0; j < ncols16; j += 16) {
            // 加载16个src0和src1的元素到寄存器中
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 b0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
            __m256 b1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 8)));
// 如果定义了__FMA__，则使用FMA指令进行乘加操作，将结果累加到sum0和sum1中
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
// 否则，使用乘法和加法指令进行操作，将结果累加到sum0和sum1中
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
        }
        // 将累加结果存储到dst数组中
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1);

        // 处理剩余的不足16的列
        for (int j = ncols16; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

// 对16位无符号整数数组src0和src1进行矩阵向量乘法，结果存储到dst数组中
void mul_mat_vec_f16_2(
    const uint16_t * src0,
    const uint16_t * src1,
             float * dst,
    int nrows,
    int ncols) {

    // 计算不足32的列数
    const int ncols32 = ncols & ~31;
    // 循环遍历每一行
    for (int i = 0; i < nrows; i++) {
        // 初始化四个256位浮点数变量，值为0
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        // 获取源数据行的指针
        const uint16_t * src0_row = src0 + i * ncols;
        // 循环遍历每32列
        for (int j = 0; j < ncols32; j += 32) {
            // 从源数据行中加载16个16位整数，转换成8个32位浮点数，存储到a0-a3
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 a2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 16)));
            __m256 a3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 24)));
            // 从第二个源数据行中加载16个16位整数，转换成8个32位浮点数，存储到b0-b3
            __m256 b0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
            __m256 b1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 8)));
            __m256 b2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 16)));
            __m256 b3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 24)));
#if defined(__FMA__)
            // 如果支持 FMA 指令集，则使用 FMA 指令进行乘加运算
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            // 如果不支持 FMA 指令集，则分别进行乘法和加法运算
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        // 将每个 sum 向量中的元素相加，得到最终结果并存储到 dst 数组中
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1) + reduce_vector8_0(sum2) + reduce_vector8_0(sum3);

        // 处理剩余的列数（不是32的倍数的列）
        for (int j = ncols32; j < ncols; j++) {
            // 将 src0 和 src1 对应位置的元素转换成 float 类型，然后相乘并累加到 dst 数组中
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

void mul_mat_vec_f16_3(
    const uint16_t * src0,
    const    float * src1,
             float * dst,
    int nrows,
    int ncols) {

    const int ncols32 = ncols & ~31;

    for (int i = 0; i < nrows; i++) {
        // 初始化四个用于累加的向量为零向量
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        const uint16_t * src0_row = src0 + i * ncols;
        for (int j = 0; j < ncols32; j += 32) {
            // 从 src0 和 src1 中加载数据到向量中，准备进行乘法和加法运算
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 a2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 16)));
            __m256 a3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 24)));
            __m256 b0 = _mm256_loadu_ps(src1 + j);
            __m256 b1 = _mm256_loadu_ps(src1 + j + 8);
            __m256 b2 = _mm256_loadu_ps(src1 + j + 16);
            __m256 b3 = _mm256_loadu_ps(src1 + j + 24);
#if defined(__FMA__)
            // 如果定义了__FMA__，则使用 FMA 指令计算乘加操作
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            // 如果未定义__FMA__，则使用乘法和加法指令分别计算乘加操作
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        // 将每个向量的结果进行规约求和，并存储到 dst 数组中
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1) + reduce_vector8_0(sum2) + reduce_vector8_0(sum3);

        // 处理剩余的列，使用 IEEE 半精度浮点数转换函数将 src0 和 src1 的值转换为单精度浮点数，并相乘后累加到 dst[i] 中
        for (int j = ncols32; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

// 获取当前时间的微秒数
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

// 主函数
int main(int argc, const char ** argv) {
    // 分配内存空间
    float * src0 = malloc(sizeof(float)*N*M);
    float * src1 = malloc(sizeof(float)*M);
    float * dst  = malloc(sizeof(float)*N);

    // 以下是使用对齐内存分配函数的注释，暂时未使用
    //float * src0 = (float *)(aligned_alloc(64, sizeof(float)*N*M));
    //float * src1 = (float *)(aligned_alloc(64, sizeof(float)*M));
    //float * dst  = (float *)(aligned_alloc(64, sizeof(float)*N));

    // 为 src0 数组赋随机值
    for (int i = 0; i < N*M; i++) {
        src0[i] = rand() / (float)RAND_MAX;
    }

    // 为 src1 数组赋随机值
    for (int i = 0; i < M; i++) {
        src1[i] = rand() / (float)RAND_MAX;
    }

    // 将 src0 和 src1 数组的值转换为 IEEE 半精度浮点数，并存储到对应的数组中
    uint16_t * src0_fp16 = (uint16_t *)(malloc(sizeof(uint16_t)*N*M));
    uint16_t * src1_fp16 = (uint16_t *)(malloc(sizeof(uint16_t)*M));
    //uint16_t * src0_fp16 = (uint16_t *)(aligned_alloc(64, sizeof(uint16_t)*N*M));
    //uint16_t * src1_fp16 = (uint16_t *)(aligned_alloc(64, sizeof(uint16_t)*M));
    // 获取当前时间的微秒数作为起始时间
    const uint64_t t_start = get_time_us();

    // 将src0中的每个元素从32位浮点数转换为16位浮点数
    for (int i = 0; i < N*M; i++) {
        src0_fp16[i] = fp16_ieee_from_fp32_value(src0[i]);
        // 打印转换前后的值并断言转换后的值不是NaN
        //printf("%f %f\n", src0[i], fp16_ieee_to_fp32_value(src0_fp16[i]));
        //assert(!isnan(fp16_ieee_to_fp32_value(src0_fp16[i]));
    }

    // 将src1中的每个元素从32位浮点数转换为16位浮点数
    for (int i = 0; i < M; i++) {
        src1_fp16[i] = fp16_ieee_from_fp32_value(src1[i]);
    }

    // 获取当前时间的微秒数作为结束时间
    const uint64_t t_end = get_time_us();
    // 打印转换时间
    printf("convert time: %f ms\n", (t_end - t_start) / 1000.0);

    // 打印转换后的前16个元素
    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], fp16_ieee_to_fp32_value(src0_fp16[i]));
    }

    // 从命令行参数中获取method值
    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    // 设置迭代次数
    const int nIter = 1000;

    // 获取开始时钟时间和微秒时间
    const clock_t start = clock();
    const uint64_t start_us = get_time_us();

    // 计算1/M和初始化sum
    double iM = 1.0/M;
    double sum = 0.0f;
    // 循环执行nIter次矩阵乘法和向量乘法
    for (int i = 0; i < nIter; i++) {
        // 根据method值选择不同的矩阵乘法和向量乘法函数
        if (method == 0) {
            mul_mat_vec_f32_0(src0, src1, dst, N, M);
        }

        // ... (重复上述过程，直到method == 6)

    }

    // 计算dst中元素的加权平均值
    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }

    // 获取结束时钟时间和微秒时间，并打印函数执行的时钟周期数和耗时
    {
        const clock_t end = clock();
        const uint64_t end_us = get_time_us();
        printf("%s: elapsed ticks: %ld\n", __func__, end - start);
        printf("%s: elapsed us: %ld\n", __func__, end_us - start_us);
    }

    // 打印加权平均值
    printf("%f\n", sum);

    // 释放动态分配的内存
    free(src0);
    free(src1);
    # 释放内存，释放 dst 指针指向的内存空间
    free(dst);
    
    # 释放内存，释放 src0_fp16 指针指向的内存空间
    free(src0_fp16);
    # 释放内存，释放 src1_fp16 指针指向的内存空间
    free(src1_fp16);
    
    # 返回 0，表示函数执行成功
    return 0;
# 闭合前面的函数定义
```