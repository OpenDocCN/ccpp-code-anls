# `PowerInfer\ggml-quants.c`

```
// 包含自定义的头文件 ggml-quants.h 和 ggml-impl.h
#include "ggml-quants.h"
#include "ggml-impl.h"

// 包含数学函数库、字符串处理库、断言库和浮点数处理库的头文件
#include <math.h>
#include <string.h>
#include <assert.h>
#include <float.h>

// 如果是 ARM 平台，包含 ARM NEON 指令集的头文件
#ifdef __ARM_NEON

// 如果 YCM 无法找到 <arm_neon.h>，可以创建一个符号链接指向它
// 例如：$ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
#include <arm_neon.h>

// 如果不是 ARM 平台，但支持 SIMD 指令集，包含 SIMD 指令集的头文件
#else
#ifdef __wasm_simd128__
#include <wasm_simd128.h>
// 如果不是__POWER9_VECTOR__，则包含altivec.h头文件，并重新定义bool为_Bool
#else
#ifdef __POWER9_VECTOR__
#include <altivec.h>
#undef bool
#define bool _Bool
// 如果不是__POWER9_VECTOR__，且是_MSC_VER或__MINGW32__，则包含intrin.h头文件
#else
#if defined(_MSC_VER) || defined(__MINGW32__)
#include <intrin.h>
// 如果不是_MSC_VER或__MINGW32__，且定义了__AVX__、__AVX2__、__AVX512F__、__SSSE3__、__SSE3__，则包含immintrin.h头文件
#else
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__) || defined(__SSE3__)
// 如果不是__riscv，则包含immintrin.h头文件
#if !defined(__riscv)
#include <immintrin.h>
#endif
#endif
#endif
#endif
#endif
#endif
#endif

// 如果定义了__riscv_v_intrinsic
#ifdef __riscv_v_intrinsic
// 包含 RISC-V 向量指令集的头文件
#include <riscv_vector.h>
#endif

// 取消定义 MIN 和 MAX 宏
#undef MIN
#undef MAX

// 定义 MIN 宏，返回 a 和 b 中较小的值
#define MIN(a, b) ((a) < (b) ? (a) : (b))
// 定义 MAX 宏，返回 a 和 b 中较大的值
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义 MM256_SET_M128I 宏，将两个 __m128i 向量合并成一个 __m256i 向量
#define MM256_SET_M128I(a, b) _mm256_insertf128_si256(_mm256_castsi128_si256(b), (a), 1)

// 如果定义了 AVX、AVX2、AVX512F 或 SSSE3 指令集，则执行以下代码
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)
// 乘以 int8_t 类型的向量，将结果成对相加两次
static inline __m128i mul_sum_i8_pairs(const __m128i x, const __m128i y) {
    // 获取 x 向量的绝对值
    const __m128i ax = _mm_sign_epi8(x, x);
    // 对 y 向量的值进行符号扩展
    const __m128i sy = _mm_sign_epi8(y, x);
    // 执行乘法并创建 16 位值
    const __m128i dot = _mm_maddubs_epi16(ax, sy);
    // 创建一个包含全部元素为1的__m128i类型的变量
    const __m128i ones = _mm_set1_epi16(1);
    // 对两个__m128i类型的变量进行逐元素相乘并相加
    return _mm_madd_epi16(ones, dot);
}

#if __AVX__ || __AVX2__ || __AVX512F__
// 水平加和8个浮点数
static inline float hsum_float_8(const __m256 x) {
    // 从__m256类型的变量中提取出第二个__m128类型的变量
    __m128 res = _mm256_extractf128_ps(x, 1);
    // 两个__m128类型的变量逐元素相加
    res = _mm_add_ps(res, _mm256_castps256_ps128(x));
    // 继续相加
    res = _mm_add_ps(res, _mm_movehl_ps(res, res));
    // 继续相加
    res = _mm_add_ss(res, _mm_movehdup_ps(res));
    // 将__m128类型的变量转换为float类型并返回
    return _mm_cvtss_f32(res);
}

// 水平加和8个int32_t类型的整数
static inline int hsum_i32_8(const __m256i a) {
    // 将__m256i类型的变量拆分为两个__m128i类型的变量并逐元素相加
    const __m128i sum128 = _mm_add_epi32(_mm256_castsi256_si128(a), _mm256_extractf128_si256(a, 1));
    // 将得到的__m128i类型的变量进行unpack和相加
    const __m128i hi64 = _mm_unpackhi_epi64(sum128, sum128);
    const __m128i sum64 = _mm_add_epi32(hi64, sum128);
    // 对得到的__m128i类型的变量进行shuffle和相加
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
// 返回两个__m128i类型的参数相加后的低32位整数
static inline int hsum_i32_4(const __m128i a) {
    // 将参数a的高64位和低64位交叉组合成一个新的__m128i类型
    const __m128i hi64 = _mm_unpackhi_epi64(a, a);
    // 将参数a和hi64相加得到一个新的__m128i类型
    const __m128i sum64 = _mm_add_epi32(hi64, a);
    // 将sum64按照指定顺序重新排列得到一个新的__m128i类型
    const __m128i hi32  = _mm_shuffle_epi32(sum64, _MM_SHUFFLE(2, 3, 0, 1));
    // 返回两个__m128i类型的参数相加后的低32位整数
    return _mm_cvtsi128_si32(_mm_add_epi32(sum64, hi32));
}

// 如果AVX2或AVX512F被定义，则执行以下代码
// 将32位整数扩展为32个字节{ 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    // 从指针x处复制一个32位整数到x32
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建一个__m256i类型的变量，用于指定按照特定顺序排列的字节
    const __m256i shuf_mask = _mm256_set_epi64x(
            0x0303030303030303, 0x0202020202020202,
            0x0101010101010101, 0x0000000000000000);
    // 使用指定顺序排列的字节对x32进行重排列得到一个新的__m256i类型
    __m256i bytes = _mm256_shuffle_epi8(_mm256_set1_epi32(x32), shuf_mask);
// 创建一个包含特定值的 256 位整数向量
const __m256i bit_mask = _mm256_set1_epi64x(0x7fbfdfeff7fbfdfe);
// 将两个 256 位整数向量进行按位或操作
bytes = _mm256_or_si256(bytes, bit_mask);
// 比较两个 256 位整数向量的相等性，返回结果向量
return _mm256_cmpeq_epi8(bytes, _mm256_set1_epi64x(-1));
}

// 将 32 个 4 位字段解包成 32 个字节
// 输出向量包含 32 个字节，每个字节在 [ 0 .. 15 ] 区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从内存中加载 128 位整数向量
    const __m128i tmp = _mm_loadu_si128((const __m128i *)rsi);
    // 创建一个 256 位整数向量，包含两个 128 位整数向量
    const __m256i bytes = MM256_SET_M128I(_mm_srli_epi16(tmp, 4), tmp);
    // 创建一个包含特定值的 256 位整数向量
    const __m256i lowMask = _mm256_set1_epi8( 0xF );
    // 将两个 256 位整数向量进行按位与操作
    return _mm256_and_si256(lowMask, bytes);
}

// 将 int16_t 类型的向量成对相加，并以浮点数向量返回
static inline __m256 sum_i16_pairs_float(const __m256i x) {
    // 创建一个包含特定值的 256 位整数向量
    const __m256i ones = _mm256_set1_epi16(1);
    // 将两个 256 位整数向量进行成对相加，并返回结果向量
    const __m256i summed_pairs = _mm256_madd_epi16(ones, x);
    // 将整数向量转换为浮点数向量
    return _mm256_cvtepi32_ps(summed_pairs);
}

// 使用 AVXVNNI 指令集计算两个 __m256i 向量的乘积和，并将结果转换为 __m256 浮点向量返回
static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
#if __AVXVNNI__
    // 创建一个全零的 __m256i 向量
    const __m256i zero = _mm256_setzero_si256();
    // 使用 AVXVNNI 指令计算两个向量的乘积和
    const __m256i summed_pairs = _mm256_dpbusd_epi32(zero, ax, sy);
    // 将结果转换为浮点数向量返回
    return _mm256_cvtepi32_ps(summed_pairs);
#else
    // 如果不支持 AVXVNNI 指令集，则执行下面的代码
    // 执行乘法并创建 16 位值
    const __m256i dot = _mm256_maddubs_epi16(ax, sy);
    // 调用 sum_i16_pairs_float 函数返回结果
    return sum_i16_pairs_float(dot);
#endif
}

// 以整型方式乘法，对结果进行两次成对相加，并以浮点向量返回
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
#if __AVXVNNIINT8__
    // 创建一个全零的 __m256i 向量
    const __m256i zero = _mm256_setzero_si256();
    // 使用 AVXVNNIINT8 指令计算两个向量的乘积和
    const __m256i summed_pairs = _mm256_dpbssd_epi32(zero, x, y);
    // 将结果转换为浮点数向量返回
    return _mm256_cvtepi32_ps(summed_pairs);
// 如果不是 AVX512F 架构，则执行以下代码
// 将 x 向量中的每个元素取绝对值
const __m256i ax = _mm256_sign_epi8(x, x);
// 将 y 向量中的每个元素与 x 向量中的每个元素进行符号相乘
const __m256i sy = _mm256_sign_epi8(y, x);
// 调用 mul_sum_us8_pairs_float 函数，返回结果
return mul_sum_us8_pairs_float(ax, sy);
#endif
}

// 将 256 位的字节向量转换为 128 位的字节向量
static inline __m128i packNibbles( __m256i bytes )
{
    // 将每个 16 位的字节向量中的位移动，将高位字节移到低位字节位置
#if __AVX512F__
    // 将字节向量中的每个元素右移 4 位
    const __m256i bytes_srli_4 = _mm256_srli_epi16(bytes, 4);   // 0000_0000_abcd_0000
    // 将原始字节向量与右移后的字节向量进行按位或操作
    bytes = _mm256_or_si256(bytes, bytes_srli_4);               // 0000_abcd_abcd_efgh
    // 将 256 位的字节向量转换为 128 位的字节向量
    return _mm256_cvtepi16_epi8(bytes);                         // abcd_efgh
#else
    // 如果不是 AVX512F 架构，则执行以下代码
    // 创建一个低位字节向量，每个元素都是 0xFF
    const __m256i lowByte = _mm256_set1_epi16( 0xFF );
    // 对字节向量进行按位非操作，然后与低位字节向量进行按位与操作
    __m256i high = _mm256_andnot_si256( lowByte, bytes );
    __m256i low = _mm256_and_si256( lowByte, bytes );
    // 将高位的每个16位整数右移4位
    high = _mm256_srli_epi16( high, 4 );
    // 将低位和右移后的高位进行按位或操作
    bytes = _mm256_or_si256( low, high );

    // 将uint16_t的数据压缩成字节
    // 将256位的数据转换成128位的数据
    __m128i r0 = _mm256_castsi256_si128( bytes );
    // 从256位数据中提取第二个128位的数据
    __m128i r1 = _mm256_extracti128_si256( bytes, 1 );
    // 将两个128位的数据打包成一个128位的数据
    return _mm_packus_epi16( r0, r1 );
#endif
}
#elif defined(__AVX__)
// 将32位的数据扩展成32个字节 { 0x00, 0xFF }
static inline __m256i bytes_from_bits_32(const uint8_t * x) {
    // 从指针x处复制32位的数据到x32
    uint32_t x32;
    memcpy(&x32, x, sizeof(uint32_t));
    // 创建用于按位重排的掩码
    const __m128i shuf_maskl = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
    const __m128i shuf_maskh = _mm_set_epi64x(0x0303030303030303, 0x0202020202020202);
    // 使用掩码对32位数据进行按位重排
    __m128i bytesl = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskl);
    __m128i bytesh = _mm_shuffle_epi8(_mm_set1_epi32(x32), shuf_maskh);
    // 创建用于按位或操作的掩码
    const __m128i bit_mask = _mm_set1_epi64x(0x7fbfdfeff7fbfdfe);
    // 对按位重排后的数据进行按位或操作
    bytesl = _mm_or_si128(bytesl, bit_mask);
// 将两个__m128i类型的向量按位或操作，并将结果存储在bytesh中
bytesh = _mm_or_si128(bytesh, bit_mask);
// 将bytesl中的每个字节与-1进行比较，结果存储在bytesl中
bytesl = _mm_cmpeq_epi8(bytesl, _mm_set1_epi64x(-1));
// 将bytesh中的每个字节与-1进行比较，结果存储在bytesh中
bytesh = _mm_cmpeq_epi8(bytesh, _mm_set1_epi64x(-1));
// 返回一个__m256i类型的向量，其中bytesh和bytesl分别作为高128位和低128位
return MM256_SET_M128I(bytesh, bytesl);
}

// 将32个4位字段解包成32个字节
// 输出向量包含32个字节，每个字节在[0..15]区间内
static inline __m256i bytes_from_nibbles_32(const uint8_t * rsi)
{
    // 从内存中加载16个字节
    __m128i tmpl = _mm_loadu_si128((const __m128i *)rsi);
    // 将tmpl中的每个字节逻辑右移4位，结果存储在tmph中
    __m128i tmph = _mm_srli_epi16(tmpl, 4);
    // 创建一个低位掩码，所有位都是0xF
    const __m128i lowMask = _mm_set1_epi8(0xF);
    // 将tmpl中的每个字节与低位掩码进行与操作，结果存储在tmpl中
    tmpl = _mm_and_si128(lowMask, tmpl);
    // 将tmph中的每个字节与低位掩码进行与操作，结果存储在tmph中
    tmph = _mm_and_si128(lowMask, tmph);
    // 返回一个__m256i类型的向量，其中tmph和tmpl分别作为高128位和低128位
    return MM256_SET_M128I(tmph, tmpl);
}

// 将int16_t类型的向量成对相加，并以浮点向量的形式返回
// 对两个__m128i类型的参数进行16位整数相乘，并将结果相加，返回一个__m256类型的浮点数向量
static inline __m256 sum_i16_pairs_float(const __m128i xh, const __m128i xl) {
    // 创建一个所有元素都为1的__m128i类型向量
    const __m128i ones = _mm_set1_epi16(1);
    // 将xl中的每对16位整数相加，并返回一个__m128i类型的向量
    const __m128i summed_pairsl = _mm_madd_epi16(ones, xl);
    // 将xh中的每对16位整数相加，并返回一个__m128i类型的向量
    const __m128i summed_pairsh = _mm_madd_epi16(ones, xh);
    // 将summed_pairsh和summed_pairsl合并成一个__m256i类型的向量
    const __m256i summed_pairs = MM256_SET_M128I(summed_pairsh, summed_pairsl);
    // 将summed_pairs中的整数转换成浮点数，并返回一个__m256类型的浮点数向量
    return _mm256_cvtepi32_ps(summed_pairs);
}

// 对两个__m256i类型的参数进行无符号8位整数相乘，将结果相加，并返回一个__m256类型的浮点数向量
static inline __m256 mul_sum_us8_pairs_float(const __m256i ax, const __m256i sy) {
    // 将ax的低128位转换成__m128i类型的向量
    const __m128i axl = _mm256_castsi256_si128(ax);
    // 提取ax的高128位，并转换成__m128i类型的向量
    const __m128i axh = _mm256_extractf128_si256(ax, 1);
    // 将sy的低128位转换成__m128i类型的向量
    const __m128i syl = _mm256_castsi256_si128(sy);
    // 提取sy的高128位，并转换成__m128i类型的向量
    const __m128i syh = _mm256_extractf128_si256(sy, 1);
    // 对axl和syl进行无符号8位整数相乘，并将结果相加，返回一个__m128i类型的向量
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    // 对axh和syh进行无符号8位整数相乘，并将结果相加，返回一个__m128i类型的向量
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    // 调用sum_i16_pairs_float函数，对doth和dotl进行处理，并返回一个__m256类型的浮点数向量
    return sum_i16_pairs_float(doth, dotl);
}

// 对int8_t类型的数进行相乘，将结果两两相加，并返回一个浮点数向量
static inline __m256 mul_sum_i8_pairs_float(const __m256i x, const __m256i y) {
    // 将输入的两个__m256i类型的参数x和y分别转换为两个__m128i类型的变量xl和xh
    const __m128i xl = _mm256_castsi256_si128(x);
    const __m128i xh = _mm256_extractf128_si256(x, 1);
    // 将输入的两个__m256i类型的参数x和y分别转换为两个__m128i类型的变量yl和yh
    const __m128i yl = _mm256_castsi256_si128(y);
    const __m128i yh = _mm256_extractf128_si256(y, 1);
    // 获取x向量的绝对值
    const __m128i axl = _mm_sign_epi8(xl, xl);
    const __m128i axh = _mm_sign_epi8(xh, xh);
    // 对y向量的值进行符号处理
    const __m128i syl = _mm_sign_epi8(yl, xl);
    const __m128i syh = _mm_sign_epi8(yh, xh);
    // 执行乘法并创建16位值
    const __m128i dotl = _mm_maddubs_epi16(axl, syl);
    const __m128i doth = _mm_maddubs_epi16(axh, syh);
    return sum_i16_pairs_float(doth, dotl);
}

static inline __m128i packNibbles( __m128i bytes1, __m128i bytes2 )
{
    // 将16位字节中的位移动，从0000_abcd_0000_efgh移动到0000_0000_abcd_efgh
// 创建一个包含全1字节的__m128i类型变量，用于与其他变量进行按位与运算
const __m128i lowByte = _mm_set1_epi16( 0xFF );
// 对bytes1进行按位非运算，并将结果存储到high变量中
__m128i high = _mm_andnot_si128( lowByte, bytes1 );
// 对bytes1进行按位与运算，并将结果存储到low变量中
__m128i low = _mm_and_si128( lowByte, bytes1 );
// 对high变量进行逻辑右移16位操作
high = _mm_srli_epi16( high, 4 );
// 将low和high进行按位或运算，并将结果存储到bytes1中
bytes1 = _mm_or_si128( low, high );
// 对bytes2进行按位非运算，并将结果存储到high变量中
high = _mm_andnot_si128( lowByte, bytes2 );
// 对bytes2进行按位与运算，并将结果存储到low变量中
low = _mm_and_si128( lowByte, bytes2 );
// 对high变量进行逻辑右移16位操作
high = _mm_srli_epi16( high, 4 );
// 将low和high进行按位或运算，并将结果存储到bytes2中
bytes2 = _mm_or_si128( low, high );

// 将bytes1和bytes2进行16位整数打包成8位无符号整数
return _mm_packus_epi16( bytes1, bytes2);
}
#endif
#elif defined(__SSSE3__)
// 水平加法，将4个__m128类型的变量中的浮点数进行水平相加
static inline float hsum_float_4x4(const __m128 a, const __m128 b, const __m128 c, const __m128 d) {
// 将a和b中的浮点数进行水平相加，并将结果存储到res_0中
__m128 res_0 =_mm_hadd_ps(a, b);
// 将c和d中的浮点数进行水平相加，并将结果存储到res_1中
__m128 res_1 =_mm_hadd_ps(c, d);
// 将res_0和res_1中的浮点数进行水平相加，并将结果存储到res中
__m128 res =_mm_hadd_ps(res_0, res_1);
// 将res中的浮点数进行水平相加，并将结果存储到res中
res =_mm_hadd_ps(res, res);
    # 对两个寄存器中的数据进行水平加法操作，并将结果存储在一个寄存器中
    res =_mm_hadd_ps(res, res);
    
    # 将结果寄存器中的单精度浮点数转换为标量值并返回
    return _mm_cvtss_f32(res);
}
#endif // __AVX__ || __AVX2__ || __AVX512F__
#endif // defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__)

#if defined(__ARM_NEON)
#if !defined(__aarch64__)

// 64-bit compatibility

// 下面是一系列的内联函数，用于执行不同的 NEON 指令操作，包括整数加法、整数乘法、浮点数加法、浮点数最大值比较、浮点数转换等
// 每个函数的作用在函数名后的注释中有说明
inline static int32_t vaddvq_s16(int16x8_t v) {
// 返回一个包含向量中所有元素相加的结果的整数
return
    (int32_t)vgetq_lane_s16(v, 0) + (int32_t)vgetq_lane_s16(v, 1) +
    (int32_t)vgetq_lane_s16(v, 2) + (int32_t)vgetq_lane_s16(v, 3) +
    (int32_t)vgetq_lane_s16(v, 4) + (int32_t)vgetq_lane_s16(v, 5) +
    (int32_t)vgetq_lane_s16(v, 6) + (int32_t)vgetq_lane_s16(v, 7);
}

// 将两个 int16x8_t 类型的向量按元素相加
inline static int16x8_t vpaddq_s16(int16x8_t a, int16x8_t b) {
    int16x4_t a0 = vpadd_s16(vget_low_s16(a), vget_high_s16(a));
    int16x4_t b0 = vpadd_s16(vget_low_s16(b), vget_high_s16(b));
    return vcombine_s16(a0, b0);
}

// 返回一个包含向量中所有元素相加的结果的整数
inline static int32_t vaddvq_s32(int32x4_t v) {
    return vgetq_lane_s32(v, 0) + vgetq_lane_s32(v, 1) + vgetq_lane_s32(v, 2) + vgetq_lane_s32(v, 3);
}

// 返回一个包含向量中所有元素相加的结果的浮点数
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}
// 返回四个浮点数中的最大值
inline static float vmaxvq_f32(float32x4_t v) {
    return
        MAX(MAX(vgetq_lane_f32(v, 0), vgetq_lane_f32(v, 1)),
            MAX(vgetq_lane_f32(v, 2), vgetq_lane_f32(v, 3)));
}

// 将四个浮点数向下取整转换为四个整数
inline static int32x4_t vcvtnq_s32_f32(float32x4_t v) {
    int32x4_t res;

    res[0] = roundf(vgetq_lane_f32(v, 0));
    res[1] = roundf(vgetq_lane_f32(v, 1));
    res[2] = roundf(vgetq_lane_f32(v, 2));
    res[3] = roundf(vgetq_lane_f32(v, 3));

    return res;
}

// 加载两个寄存器的有符号16位整数或无符号8位整数
// vld1q_s16_x2
// vld1q_u8_x2
// vld1q_u8_x4
// vld1q_s8_x2
// vld1q_s8_x4
// TODO: double-check these work correctly
// 上面三行是函数声明，用于加载不同类型和数量的数据到寄存器中，需要进一步确认其正确性

// 定义一个包含两个int16x8_t类型的结构体
typedef struct ggml_int16x8x2_t {
    int16x8_t val[2];
} ggml_int16x8x2_t;

// 定义一个静态内联函数，用于加载指针指向的int16_t类型数据到寄存器中
inline static ggml_int16x8x2_t ggml_vld1q_s16_x2(const int16_t * ptr) {
    ggml_int16x8x2_t res;

    // 加载ptr指向的数据到val[0]中
    res.val[0] = vld1q_s16(ptr + 0);
    // 加载ptr+8指向的数据到val[1]中
    res.val[1] = vld1q_s16(ptr + 8);

    return res;
}

// 定义一个包含两个uint8x16_t类型的结构体
typedef struct ggml_uint8x16x2_t {
    uint8x16_t val[2];
// 定义一个包含两个 16 个 8 位无符号整数的结构体
} ggml_uint8x16x2_t;

// 定义一个函数，用于加载两个 16 个 8 位无符号整数的值到结构体中
inline static ggml_uint8x16x2_t ggml_vld1q_u8_x2(const uint8_t * ptr) {
    ggml_uint8x16x2_t res; // 定义一个结构体变量

    res.val[0] = vld1q_u8(ptr + 0); // 加载第一个 16 个 8 位无符号整数的值
    res.val[1] = vld1q_u8(ptr + 16); // 加载第二个 16 个 8 位无符号整数的值

    return res; // 返回加载后的结构体
}

// 定义一个包含四个 16 个 8 位无符号整数的结构体
typedef struct ggml_uint8x16x4_t {
    uint8x16_t val[4];
} ggml_uint8x16x4_t;

// 定义一个函数，用于加载四个 16 个 8 位无符号整数的值到结构体中
inline static ggml_uint8x16x4_t ggml_vld1q_u8_x4(const uint8_t * ptr) {
    ggml_uint8x16x4_t res; // 定义一个结构体变量

    res.val[0] = vld1q_u8(ptr + 0); // 加载第一个 16 个 8 位无符号整数的值
    res.val[1] = vld1q_u8(ptr + 16); // 加载第二个 16 个 8 位无符号整数的值
    # 将指针指向的内存中的数据加载到val[2]中
    res.val[2] = vld1q_u8(ptr + 32);
    # 将指针指向的内存中的数据加载到val[3]中
    res.val[3] = vld1q_u8(ptr + 48);

    return res;
}

typedef struct ggml_int8x16x2_t {
    int8x16_t val[2];
} ggml_int8x16x2_t;

# 从指针指向的内存中加载数据到val[0]和val[1]中
inline static ggml_int8x16x2_t ggml_vld1q_s8_x2(const int8_t * ptr) {
    ggml_int8x16x2_t res;

    res.val[0] = vld1q_s8(ptr + 0);
    res.val[1] = vld1q_s8(ptr + 16);

    return res;
}

typedef struct ggml_int8x16x4_t {
// 定义一个包含4个int8x16_t类型元素的结构体ggml_int8x16x4_t
} ggml_int8x16x4_t;

// 定义一个静态内联函数ggml_vld1q_s8_x4，参数为指向int8_t类型的指针ptr
inline static ggml_int8x16x4_t ggml_vld1q_s8_x4(const int8_t * ptr) {
    // 定义一个ggml_int8x16x4_t类型的变量res
    ggml_int8x16x4_t res;

    // 从ptr指向的地址开始，分别加载4个int8x16_t类型的值到res的val数组中
    res.val[0] = vld1q_s8(ptr + 0);
    res.val[1] = vld1q_s8(ptr + 16);
    res.val[2] = vld1q_s8(ptr + 32);
    res.val[3] = vld1q_s8(ptr + 48);

    // 返回res
    return res;
}

#else

// 定义宏，将ggml_int16x8x2_t替换为int16x8x2_t
#define ggml_int16x8x2_t  int16x8x2_t
// 定义宏，将ggml_uint8x16x2_t替换为uint8x16x2_t
#define ggml_uint8x16x2_t uint8x16x2_t
// 定义宏，将ggml_uint8x16x4_t替换为uint8x16x4_t
#define ggml_uint8x16x4_t uint8x16x4_t
// 定义宏，将ggml_int8x16x2_t替换为int8x16x2_t
#define ggml_int8x16x2_t  int8x16x2_t
// 定义宏 ggml_int8x16x4_t 为 int8x16x4_t

// 定义宏 ggml_vld1q_s16_x2 为 vld1q_s16_x2
// 定义宏 ggml_vld1q_u8_x2 为 vld1q_u8_x2
// 定义宏 ggml_vld1q_u8_x4 为 vld1q_u8_x4
// 定义宏 ggml_vld1q_s8_x2 为 vld1q_s8_x2
// 定义宏 ggml_vld1q_s8_x4 为 vld1q_s8_x4

// 结束宏定义

// 如果定义了 __ARM_NEON 或者 __wasm_simd128__，则进行以下宏定义
// 定义宏 B1(c,s,n) 为 0xnc, 0xns
// 定义宏 B2(c,s,n) 为 B1(c,s,nc), B1(c,s,ns)
// 定义宏 B3(c,s,n) 为 B2(c,s,nc), B2(c,s,ns)
// 定义宏 B4(c,s,n) 为 B3(c,s,nc), B3(c,s,ns)
// 定义宏 B5(c,s,n) 为 B4(c,s,nc), B4(c,s,ns)
// 定义宏 B6(c,s,n) 为 B5(c,s,nc), B5(c,s,ns)
// 定义宏 B7(c,s,n) 为 B6(c,s,nc), B6(c,s,ns)
// 定义宏 B8(c,s) 为 B7(c,s,c), B7(c,s,s)
// 用于将8位扩展为8字节的预计算表：
static const uint64_t table_b2b_0[1 << 8] = { B8(00, 10) }; // ( b) << 4
static const uint64_t table_b2b_1[1 << 8] = { B8(10, 00) }; // (!b) << 4
#endif

// 用于确定性创建模型文件的参考实现
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k) {
    static const int qk = QK4_0;

    assert(k % qk == 0); // 断言k能被qk整除

    const int nb = k / qk; // 计算块的数量

    for (int i = 0; i < nb; i++) { // 遍历每个块
        float amax = 0.0f; // 绝对最大值
        float max  = 0.0f; // 最大值

        for (int j = 0; j < qk; j++) { // 遍历每个元素
            const float v = x[i*qk + j]; // 获取输入数组中的值
        // 寻找数组中绝对值最大的元素
        if (amax < fabsf(v)) {
            amax = fabsf(v);  // 更新最大绝对值
            max  = v;         // 更新最大值
        }
    }

    const float d  = max / -8;  // 计算最大值的负数除以8
    const float id = d ? 1.0f/d : 0.0f;  // 如果d不为0，则计算其倒数，否则为0

    y[i].d = GGML_FP32_TO_FP16(d);  // 将d转换为16位浮点数表示

    for (int j = 0; j < qk/2; ++j) {
        const float x0 = x[i*qk + 0    + j]*id;  // 计算x0
        const float x1 = x[i*qk + qk/2 + j]*id;  // 计算x1

        const uint8_t xi0 = MIN(15, (int8_t)(x0 + 8.5f));  // 将x0转换为8位有符号整数并限制在0-15之间
        const uint8_t xi1 = MIN(15, (int8_t)(x1 + 8.5f));  // 将x1转换为8位有符号整数并限制在0-15之间

        y[i].qs[j]  = xi0;        // 将xi0存入y[i].qs[j]的低4位
        y[i].qs[j] |= xi1 << 4;   // 将xi1存入y[i].qs[j]的高4位
// 定义一个函数，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_0_reference 函数进行量化
    quantize_row_q4_0_reference(x, y, k);
}

// 定义一个函数，将输入的一维数组 x 进行量化，结果存储在结构体数组 y 中，每个结构体包含 qk 个元素
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k) {
    // 定义量化的步长为 QK4_1
    const int qk = QK4_1;

    // 断言 k 能够被 qk 整除
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
        // 从输入数组中获取特定位置的值
        const float v = x[i*qk + j];

        // 如果当前值小于最小值，则更新最小值
        if (v < min) min = v;
        // 如果当前值大于最大值，则更新最大值
        if (v > max) max = v;
    }

    // 计算最大值和最小值之间的间隔
    const float d  = (max - min) / ((1 << 4) - 1);
    // 计算间隔的倒数，如果间隔为0，则设为0
    const float id = d ? 1.0f/d : 0.0f;

    // 将间隔和最小值转换为半精度浮点数，并存储在输出数组中
    y[i].d = GGML_FP32_TO_FP16(d);
    y[i].m = GGML_FP32_TO_FP16(min);

    // 对输入数组进行量化和缩放，并存储在输出数组中
    for (int j = 0; j < qk/2; ++j) {
        const float x0 = (x[i*qk + 0    + j] - min)*id;
        const float x1 = (x[i*qk + qk/2 + j] - min)*id;

        const uint8_t xi0 = MIN(15, (int8_t)(x0 + 0.5f));
        const uint8_t xi1 = MIN(15, (int8_t)(x1 + 0.5f));

        y[i].qs[j]  = xi0;
// 将 x 数组中的值进行量化，并存储到 y 数组中
void quantize_row_q4_1_reference(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q4_1_reference 函数对 x 数组进行量化
    quantize_row_q4_1_reference(x, y, k);
}

// 将 x 数组中的值进行量化，并存储到 y 结构体数组中
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k) {
    // 定义量化因子 qk
    static const int qk = QK5_0;

    // 断言 k 能被 qk 整除
    assert(k % qk == 0);

    // 计算块数
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 初始化绝对值最大值和最大值
        float amax = 0.0f; // absolute max
        float max  = 0.0f;
        // 遍历数组 x 中的元素，找出绝对值最大的元素及其值
        for (int j = 0; j < qk; j++) {
            const float v = x[i*qk + j];
            if (amax < fabsf(v)) {
                amax = fabsf(v);
                max  = v;
            }
        }

        // 计算最大值的绝对值除以 -16，并计算其倒数
        const float d  = max / -16;
        const float id = d ? 1.0f/d : 0.0f;

        // 将 d 转换成 16 位浮点数并赋值给 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);

        // 初始化 qh 为 0
        uint32_t qh = 0;

        // 遍历数组 x 中的一半元素，进行一系列计算
        for (int j = 0; j < qk/2; ++j) {
            // 计算 x0 和 x1，并将其除以 d
            const float x0 = x[i*qk + 0    + j]*id;
            const float x1 = x[i*qk + qk/2 + j]*id;

            // 将 x0 加上 16.5 后取整并限制在 0 到 31 之间
            const uint8_t xi0 = MIN(31, (int8_t)(x0 + 16.5f));
            // 计算 xi1 的值，取 x1 + 16.5f 和 31 中的较小值
            const uint8_t xi1 = MIN(31, (int8_t)(x1 + 16.5f));

            // 将 xi0 和 xi1 的低 4 位分别存储到 y[i].qs[j] 中
            y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

            // 获取 xi0 和 xi1 的第 5 位，并将其存储到 qh 中的相应位置
            qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
            qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
        }

        // 将 qh 的值拷贝到 y[i].qh 中
        memcpy(&y[i].qh, &qh, sizeof(qh));
    }
}

// 调用 quantize_row_q5_0_reference 函数
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k) {
    quantize_row_q5_0_reference(x, y, k);
}

// quantize_row_q5_1_reference 函数的实现
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k) {
    // 定义 qk 的值为 QK5_1
    const int qk = QK5_1;
    # 确保 k 能被 qk 整除
    assert(k % qk == 0);

    # 计算 nb，即 k 除以 qk 的商
    const int nb = k / qk;

    # 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        # 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        # 遍历 qk 次
        for (int j = 0; j < qk; j++) {
            # 获取 x 数组中的值
            const float v = x[i*qk + j];

            # 更新最小值和最大值
            if (v < min) min = v;
            if (v > max) max = v;
        }

        # 计算 d，即最大值和最小值的差除以 2^5-1
        const float d  = (max - min) / ((1 << 5) - 1);
        # 计算 id，即 d 的倒数，如果 d 不为 0 则为 1/d，否则为 0
        const float id = d ? 1.0f/d : 0.0f;

        # 将 d 转换成 FP16 格式并赋值给 y[i].d
        y[i].d = GGML_FP32_TO_FP16(d);
        # 将 min 转换成 FP16 格式并赋值给 y[i].m
        y[i].m = GGML_FP32_TO_FP16(min);
// 初始化一个无符号32位整数变量 qh
uint32_t qh = 0;

// 遍历 qk 的一半次数
for (int j = 0; j < qk/2; ++j) {
    // 计算 x0 和 x1 的值
    const float x0 = (x[i*qk + 0    + j] - min)*id;
    const float x1 = (x[i*qk + qk/2 + j] - min)*id;

    // 将 x0 和 x1 转换为无符号8位整数
    const uint8_t xi0 = (uint8_t)(x0 + 0.5f);
    const uint8_t xi1 = (uint8_t)(x1 + 0.5f);

    // 将 xi0 和 xi1 的低4位存储到 y[i].qs[j] 中
    y[i].qs[j] = (xi0 & 0x0F) | ((xi1 & 0x0F) << 4);

    // 获取 xi0 和 xi1 的第5位，并将其存储到 qh 的相应位置
    qh |= ((xi0 & 0x10u) >> 4) << (j + 0);
    qh |= ((xi1 & 0x10u) >> 4) << (j + qk/2);
}

// 将 qh 的值拷贝到 y[i].qh 中
memcpy(&y[i].qh, &qh, sizeof(y[i].qh));
// 将输入数组 x 中的数据量化为 q5_1 格式，存储到 y 中，k 为数据量
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k) {
    // 调用 quantize_row_q5_1_reference 函数进行数据量化
    quantize_row_q5_1_reference(x, y, k);
}

// 用于确定性创建模型文件的参考实现
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k) {
    // 确保 k 是 QK8_0 的倍数
    assert(k % QK8_0 == 0);
    // 计算块的数量
    const int nb = k / QK8_0;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对最大值

        // 遍历每个元素
        for (int j = 0; j < QK8_0; j++) {
            // 获取元素值
            const float v = x[i*QK8_0 + j];
            // 更新绝对最大值
            amax = MAX(amax, fabsf(v));
        }

        // 计算量化因子
        const float d = amax / ((1 << 7) - 1);
        // 计算倒数的量化因子
        const float id = d ? 1.0f/d : 0.0f;
        # 将浮点数转换为半精度浮点数
        y[i].d = GGML_FP32_TO_FP16(d);

        # 遍历每个元素，将其乘以id，并存储到y[i].qs[j]中
        for (int j = 0; j < QK8_0; ++j) {
            const float x0 = x[i*QK8_0 + j]*id;

            y[i].qs[j] = roundf(x0);
        }
    }
}

# 将输入的浮点数数组x进行量化，并存储到vy中
void quantize_row_q8_0(const float * restrict x, void * restrict vy, int k) {
    # 确保QK8_0的值为32
    assert(QK8_0 == 32);
    # 确保k能够被QK8_0整除
    assert(k % QK8_0 == 0);
    # 计算数组x的长度
    const int nb = k / QK8_0;

    # 将vy强制转换为block_q8_0类型的指针，并赋值给y
    block_q8_0 * restrict y = vy;

    # 如果支持ARM NEON指令集，则执行以下代码
    # 遍历数组x，将其进行量化，并存储到vy中
    for (int i = 0; i < nb; i++) {
# 定义存储 8 个 float32x4_t 类型数据的数组
float32x4_t srcv [8];
# 定义存储 8 个 float32x4_t 类型数据的数组
float32x4_t asrcv[8];
# 定义存储 8 个 float32x4_t 类型数据的数组
float32x4_t amaxv[8];

# 从数组 x 中加载数据到 srcv 数组中
for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
# 计算 srcv 数组中每个元素的绝对值，存储到 asrcv 数组中
for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

# 计算 asrcv 数组中相邻两个元素的最大值，存储到 amaxv 数组中
for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
# 计算 amaxv 数组中相邻两个元素的最大值，存储到 amaxv 数组中
for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
# 计算 amaxv 数组中相邻两个元素的最大值，存储到 amaxv 数组中
for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

# 计算 amaxv 数组中所有元素的最大值
const float amax = vmaxvq_f32(amaxv[0]);

# 计算 d 的值
const float d = amax / ((1 << 7) - 1);
# 计算 id 的值
const float id = d ? 1.0f/d : 0.0f;

# 将 d 转换为 GGML_FP16 类型，并存储到 y[i].d 中
y[i].d = GGML_FP32_TO_FP16(d);

# 将 srcv 数组中的每个元素与 id 相乘，存储到 v 中
for (int j = 0; j < 8; j++) {
    const float32x4_t v  = vmulq_n_f32(srcv[j], id);
// 如果编译器支持 NEON SIMD 指令集
const int32x4_t vi = vcvtnq_s32_f32(v);
// 将浮点数向下取整转换为整型，并存储到 int32x4_t 类型的变量 vi 中

y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);
// 将 vi 中的整型数据按照索引存储到 y[i].qs 数组中

// 如果编译器支持 WebAssembly SIMD 指令集
for (int i = 0; i < nb; i++) {
    v128_t srcv [8];
    v128_t asrcv[8];
    v128_t amaxv[8];

    for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
    // 从内存中加载数据到 srcv 数组中

    for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);
    // 计算 srcv 数组中每个元素的绝对值，存储到 asrcv 数组中

    for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
    // 计算 asrcv 数组中相邻两个元素的最大值，存储到 amaxv 数组中

    for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
    // 计算 amaxv 数组中相邻两个元素的最大值，存储到 amaxv 数组中

    for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);
    // 计算 amaxv 数组中相邻两个元素的最大值，存储到 amaxv 数组中
// 计算四个向量中的最大值，并将结果赋给 amax
const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                           wasm_f32x4_extract_lane(amaxv[0], 1)),
                       MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                           wasm_f32x4_extract_lane(amaxv[0], 3)));

// 计算 d 的值，即 amax 除以 (1 << 7) - 1
const float d = amax / ((1 << 7) - 1);

// 计算 id 的值，如果 d 不为 0，则 id 为 1/d，否则为 0
const float id = d ? 1.0f/d : 0.0f;

// 将 d 转换为 FP16 格式，并赋值给 y[i].d
y[i].d = GGML_FP32_TO_FP16(d);

// 对每个 j 进行循环
for (int j = 0; j < 8; j++) {
    // 将 srcv[j] 与 id 相乘，并将结果赋给 v
    const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
    // 将 v 转换为整数类型，并饱和转换为 32 位整数
    const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

    // 将 vi 中的每个元素分别赋值给 y[i].qs 数组的对应位置
    y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
    y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
    y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
    y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);
}
    }
#elif defined(__AVX2__) || defined(__AVX__)
    // 遍历每个块
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

        // 提取 maxAbs 的高低位
        __m128 max4 = _mm_max_ps( _mm256_extractf128_ps( maxAbs, 1 ), _mm256_castps256_ps128( maxAbs ) );
        max4 = _mm_max_ps( max4, _mm_movehl_ps( max4, max4 ) );
        max4 = _mm_max_ss( max4, _mm_movehdup_ps( max4 ) );
        // 计算最大值并转换为标量
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

        // 四舍五入到最接近的整数
        v0 = _mm256_round_ps( v0, _MM_ROUND_NEAREST );
        v1 = _mm256_round_ps( v1, _MM_ROUND_NEAREST );
        v2 = _mm256_round_ps( v2, _MM_ROUND_NEAREST );
        v3 = _mm256_round_ps( v3, _MM_ROUND_NEAREST );
        // 将浮点数转换为整数
        __m256i i0 = _mm256_cvtps_epi32( v0 ); // 将v0中的浮点数转换为整数并存储在i0中
        __m256i i1 = _mm256_cvtps_epi32( v1 ); // 将v1中的浮点数转换为整数并存储在i1中
        __m256i i2 = _mm256_cvtps_epi32( v2 ); // 将v2中的浮点数转换为整数并存储在i2中
        __m256i i3 = _mm256_cvtps_epi32( v3 ); // 将v3中的浮点数转换为整数并存储在i3中

#if defined(__AVX2__)
        // 将int32转换为int16
        i0 = _mm256_packs_epi32( i0, i1 ); // 将i0和i1中的int32转换为int16
        i2 = _mm256_packs_epi32( i2, i3 ); // 将i2和i3中的int32转换为int16
                                            // 将int16转换为int8
        i0 = _mm256_packs_epi16( i0, i2 ); // 将i0和i2中的int16转换为int8

        // 我们得到了我们宝贵的有符号字节，但是顺序现在是错误的
        // 这些AVX2 pack指令独立处理16字节块
        // 以下指令正在修复顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 ); // 创建一个用于重新排列的常量
        i0 = _mm256_permutevar8x32_epi32( i0, perm ); // 使用perm对i0进行重新排列

        _mm256_storeu_si256((__m256i *)y[i].qs, i0); // 将i0中的值存储到y[i].qs中
// 如果不支持 AVX，将寄存器分成两半，并从 SSE 中调用 AVX2 的类似函数
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

    size_t vl = __riscv_vsetvl_e32m4(QK8_0);  // 设置向量长度为 QK8_0 对应的值

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_0, vl);  // 从地址 x+i*QK8_0 处加载长度为 vl 的向量到 v_x

        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);  // 计算 v_x 中每个元素的绝对值
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0f, vl);  // 创建一个长度为 vl 的向量，每个元素值为 0.0f
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);  // 计算 vfabs 中的最大值，并将结果存储在长度为 vl 的向量 vmax 中
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);  // 将 vmax 中的值提取出来存储在 amax 中

        const float d = amax / ((1 << 7) - 1);  // 计算 d 的值
        const float id = d ? 1.0f/d : 0.0f;  // 如果 d 不为 0，则计算 id 的值为 1.0f/d，否则为 0.0f
// 将浮点数转换为半精度浮点数
y[i].d = GGML_FP32_TO_FP16(d);

// 使用向量指令将浮点数乘以标量
vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

// 将浮点数向下取整转换为整数
vint16m2_t vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
// 将整数向下取整转换为字节大小的整数
vint8m1_t vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

// 将结果存储到数组中
__riscv_vse8_v_i8m1(y[i].qs , vs, vl);
// 定义一个函数，用于将输入的浮点数数组进行量化，存储到输出的固定点数数组中
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k) {
    // 断言，确保 QK8_1 的值为 32
    assert(QK8_1 == 32);
    // 断言，确保 k 是 QK8_1 的倍数
    assert(k % QK8_1 == 0);
    // 计算需要处理的块数
    const int nb = k / QK8_1;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值

        // 遍历每个元素
        for (int j = 0; j < QK8_1; j++) {
            // 获取输入数组中的值
            const float v = x[i*QK8_1 + j];
            // 计算绝对值最大值
            amax = MAX(amax, fabsf(v));
        }

        // 计算量化因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将量化因子存储到输出数组中
        y[i].d = d;

        // 初始化求和变量
        int sum = 0;
        // 循环遍历每个块
        for (int j = 0; j < QK8_1/2; ++j) {
            // 计算每个块中两个元素的乘积
            const float v0 = x[i*QK8_1           + j]*id;
            const float v1 = x[i*QK8_1 + QK8_1/2 + j]*id;

            // 对乘积进行四舍五入，并存入输出数组中
            y[i].qs[          j] = roundf(v0);
            y[i].qs[QK8_1/2 + j] = roundf(v1);

            // 计算每个块中元素的和
            sum += y[i].qs[          j];
            sum += y[i].qs[QK8_1/2 + j];
        }

        // 将每个块的和乘以缩放因子，并存入输出数组中
        y[i].s = sum*d;
    }
}

// 对输入数组进行量化，并存入输出数组中
void quantize_row_q8_1(const float * restrict x, void * restrict vy, int k) {
    // 确保输入数组的长度是块大小的整数倍
    assert(k % QK8_1 == 0);
    // 计算块的个数
    const int nb = k / QK8_1;

    // 将输出数组转换为块结构体类型
    block_q8_1 * restrict y = vy;
#if defined(__ARM_NEON)
    // 如果定义了 ARM NEON，则执行以下代码
    for (int i = 0; i < nb; i++) {
        // 对于每个 i，执行以下操作
        float32x4_t srcv [8];
        float32x4_t asrcv[8];
        float32x4_t amaxv[8];

        // 从 x 数组中加载 32 个 float 值到 srcv 数组中
        for (int j = 0; j < 8; j++) srcv[j]  = vld1q_f32(x + i*32 + 4*j);
        // 计算 srcv 数组中每个元素的绝对值，存储到 asrcv 数组中
        for (int j = 0; j < 8; j++) asrcv[j] = vabsq_f32(srcv[j]);

        // 计算 asrcv 数组中每两个元素的最大值，存储到 amaxv 数组中
        for (int j = 0; j < 4; j++) amaxv[2*j] = vmaxq_f32(asrcv[2*j], asrcv[2*j+1]);
        // 继续计算 amaxv 数组中每两个元素的最大值
        for (int j = 0; j < 2; j++) amaxv[4*j] = vmaxq_f32(amaxv[4*j], amaxv[4*j+2]);
        // 继续计算 amaxv 数组中每两个元素的最大值
        for (int j = 0; j < 1; j++) amaxv[8*j] = vmaxq_f32(amaxv[8*j], amaxv[8*j+4]);

        // 计算 amaxv 数组中的最大值
        const float amax = vmaxvq_f32(amaxv[0]);

        // 计算 d 和 id
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将计算得到的 d 存储到 y 数组中
        y[i].d = d;
        # 创建一个包含4个32位整数的向量，每个整数都初始化为0
        int32x4_t accv = vdupq_n_s32(0);

        # 循环8次，对srcv中的每个元素与id进行乘法运算，并将结果存储在v中
        for (int j = 0; j < 8; j++) {
            const float32x4_t v  = vmulq_n_f32(srcv[j], id);
            # 将浮点数向量v转换为整数向量vi
            const int32x4_t   vi = vcvtnq_s32_f32(v);

            # 将vi中的每个元素存储到y[i].qs数组中
            y[i].qs[4*j + 0] = vgetq_lane_s32(vi, 0);
            y[i].qs[4*j + 1] = vgetq_lane_s32(vi, 1);
            y[i].qs[4*j + 2] = vgetq_lane_s32(vi, 2);
            y[i].qs[4*j + 3] = vgetq_lane_s32(vi, 3);

            # 将vi中的每个元素与accv中对应位置的元素相加，并将结果存储在accv中
            accv = vaddq_s32(accv, vi);
        }

        # 将accv中的所有元素相加，并将结果乘以d，存储在y[i].s中
        y[i].s = d * vaddvq_s32(accv);
    }
#elif defined(__wasm_simd128__)
    # 循环nb次，每次创建一个包含8个浮点数的向量
    for (int i = 0; i < nb; i++) {
        v128_t srcv [8];
# 定义两个长度为8的128位向量数组
v128_t asrcv[8];
v128_t amaxv[8];

# 通过循环将x中的数据加载到srcv数组中
for (int j = 0; j < 8; j++) srcv[j]  = wasm_v128_load(x + i*32 + 4*j);
# 将srcv数组中的数据取绝对值，存入asrcv数组中
for (int j = 0; j < 8; j++) asrcv[j] = wasm_f32x4_abs(srcv[j]);

# 通过循环计算asrcv数组中相邻两个元素的最大值，存入amaxv数组中
for (int j = 0; j < 4; j++) amaxv[2*j] = wasm_f32x4_max(asrcv[2*j], asrcv[2*j+1]);
# 通过循环计算amaxv数组中相邻两个元素的最大值，存入amaxv数组中
for (int j = 0; j < 2; j++) amaxv[4*j] = wasm_f32x4_max(amaxv[4*j], amaxv[4*j+2]);
# 通过循环计算amaxv数组中相邻两个元素的最大值，存入amaxv数组中
for (int j = 0; j < 1; j++) amaxv[8*j] = wasm_f32x4_max(amaxv[8*j], amaxv[8*j+4]);

# 计算amaxv数组中的最大值，并存入常量amax中
const float amax = MAX(MAX(wasm_f32x4_extract_lane(amaxv[0], 0),
                           wasm_f32x4_extract_lane(amaxv[0], 1)),
                       MAX(wasm_f32x4_extract_lane(amaxv[0], 2),
                           wasm_f32x4_extract_lane(amaxv[0], 3)));

# 计算d的值
const float d = amax / ((1 << 7) - 1);
# 计算id的值
const float id = d ? 1.0f/d : 0.0f;

# 将d的值存入y[i].d中
y[i].d = d;
# 创建一个包含四个相同值的 128 位向量
v128_t accv = wasm_i32x4_splat(0);

# 循环遍历 8 次
for (int j = 0; j < 8; j++) {
    # 将 srcv[j] 中的每个元素与 id 相乘，得到一个新的向量 v
    const v128_t v  = wasm_f32x4_mul(srcv[j], wasm_f32x4_splat(id));
    # 将 v 中的浮点数转换为整数，并进行饱和转换，得到一个新的整数向量 vi
    const v128_t vi = wasm_i32x4_trunc_sat_f32x4(v);

    # 将 vi 中的每个元素提取出来，分别赋值给 y[i].qs 中的不同位置
    y[i].qs[4*j + 0] = wasm_i32x4_extract_lane(vi, 0);
    y[i].qs[4*j + 1] = wasm_i32x4_extract_lane(vi, 1);
    y[i].qs[4*j + 2] = wasm_i32x4_extract_lane(vi, 2);
    y[i].qs[4*j + 3] = wasm_i32x4_extract_lane(vi, 3);

    # 将 vi 中的每个元素与 accv 中对应位置的元素相加，更新 accv
    accv = wasm_i32x4_add(accv, vi);
}

# 计算 accv 中所有元素的和，并乘以 d，赋值给 y[i].s
y[i].s = d * (wasm_i32x4_extract_lane(accv, 0) +
              wasm_i32x4_extract_lane(accv, 1) +
              wasm_i32x4_extract_lane(accv, 2) +
              wasm_i32x4_extract_lane(accv, 3));
    for (int i = 0; i < nb; i++) {
        // 遍历一个块中的元素，每次加载4个 AVX 向量
        __m256 v0 = _mm256_loadu_ps( x ); // 加载 x 指向的 8 个单精度浮点数到 AVX 向量 v0
        __m256 v1 = _mm256_loadu_ps( x + 8 ); // 加载 x 指向的下一个 8 个单精度浮点数到 AVX 向量 v1
        __m256 v2 = _mm256_loadu_ps( x + 16 ); // 加载 x 指向的下一个 8 个单精度浮点数到 AVX 向量 v2
        __m256 v3 = _mm256_loadu_ps( x + 24 ); // 加载 x 指向的下一个 8 个单精度浮点数到 AVX 向量 v3
        x += 32; // 更新 x 的指向位置，移动到下一个块的起始位置

        // 计算块中元素的最大绝对值
        const __m256 signBit = _mm256_set1_ps( -0.0f ); // 创建一个全为负零的 AVX 向量
        __m256 maxAbs = _mm256_andnot_ps( signBit, v0 ); // 计算 v0 中绝对值最大的元素
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v1 ) ); // 计算 v1 中绝对值最大的元素
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v2 ) ); // 计算 v2 中绝对值最大的元素
        maxAbs = _mm256_max_ps( maxAbs, _mm256_andnot_ps( signBit, v3 ) ); // 计算 v3 中绝对值最大的元素

        __m128 max4 = _mm_max_ps( _mm256_extractf128_ps( maxAbs, 1 ), _mm256_castps256_ps128( maxAbs ) ); // 从 AVX 向量中提取两个 128 位的值，然后计算它们的最大值
        max4 = _mm_max_ps( max4, _mm_movehl_ps( max4, max4 ) ); // 计算 max4 的高低位的最大值
        max4 = _mm_max_ss( max4, _mm_movehdup_ps( max4 ) ); // 计算 max4 的高位复制的最大值
        const float maxScalar = _mm_cvtss_f32( max4 ); // 将 max4 转换为标量值
// 将这些浮点数进行量化
const float d = maxScalar / 127.f; // 计算量化因子
y[i].d = d; // 将量化因子存储到数组中
const float id = ( maxScalar != 0.0f ) ? 127.f / maxScalar : 0.0f; // 计算乘法因子
const __m256 mul = _mm256_set1_ps( id ); // 创建一个包含乘法因子的 AVX 寄存器

// 应用乘法因子
v0 = _mm256_mul_ps( v0, mul ); // 将 v0 中的每个元素乘以乘法因子
v1 = _mm256_mul_ps( v1, mul ); // 将 v1 中的每个元素乘以乘法因子
v2 = _mm256_mul_ps( v2, mul ); // 将 v2 中的每个元素乘以乘法因子
v3 = _mm256_mul_ps( v3, mul ); // 将 v3 中的每个元素乘以乘法因子

// 四舍五入到最近的整数
v0 = _mm256_round_ps( v0, _MM_ROUND_NEAREST ); // 将 v0 中的每个元素四舍五入到最近的整数
v1 = _mm256_round_ps( v1, _MM_ROUND_NEAREST ); // 将 v1 中的每个元素四舍五入到最近的整数
v2 = _mm256_round_ps( v2, _MM_ROUND_NEAREST ); // 将 v2 中的每个元素四舍五入到最近的整数
v3 = _mm256_round_ps( v3, _MM_ROUND_NEAREST ); // 将 v3 中的每个元素四舍五入到最近的整数

// 将浮点数转换为整数
__m256i i0 = _mm256_cvtps_epi32( v0 ); // 将 v0 中的每个元素转换为整数
        // 将 8 个单精度浮点数转换为整数
        __m256i i1 = _mm256_cvtps_epi32( v1 );
        __m256i i2 = _mm256_cvtps_epi32( v2 );
        __m256i i3 = _mm256_cvtps_epi32( v3 );

#if defined(__AVX2__)
        // 计算四个整数的和，并将结果存入 y[i].s
        y[i].s = d * hsum_i32_8(_mm256_add_epi32(_mm256_add_epi32(i0, i1), _mm256_add_epi32(i2, i3)));

        // 将 int32 转换为 int16
        i0 = _mm256_packs_epi32( i0, i1 );	// 0, 1, 2, 3,  8, 9, 10, 11,  4, 5, 6, 7, 12, 13, 14, 15
        i2 = _mm256_packs_epi32( i2, i3 );	// 16, 17, 18, 19,  24, 25, 26, 27,  20, 21, 22, 23, 28, 29, 30, 31
                                            // 将 int16 转换为 int8
        i0 = _mm256_packs_epi16( i0, i2 );	// 0, 1, 2, 3,  8, 9, 10, 11,  16, 17, 18, 19,  24, 25, 26, 27,  4, 5, 6, 7, 12, 13, 14, 15, 20, 21, 22, 23, 28, 29, 30, 31

        // 我们得到了宝贵的有符号字节，但是顺序现在是错误的
        // 这些 AVX2 pack 指令独立处理 16 字节的数据块
        // 以下指令用于修正顺序
        const __m256i perm = _mm256_setr_epi32( 0, 4, 1, 5, 2, 6, 3, 7 );
        i0 = _mm256_permutevar8x32_epi32( i0, perm );
        _mm256_storeu_si256((__m256i *)y[i].qs, i0);
#else
        // 由于在 AVX 中缺少一些必要的函数，我们将寄存器分成两半，并从 SSE 中调用 AVX2 的类似函数
        __m128i ni0 = _mm256_castsi256_si128( i0 );
        __m128i ni1 = _mm256_extractf128_si256( i0, 1);
        __m128i ni2 = _mm256_castsi256_si128( i1 );
        __m128i ni3 = _mm256_extractf128_si256( i1, 1);
        __m128i ni4 = _mm256_castsi256_si128( i2 );
        __m128i ni5 = _mm256_extractf128_si256( i2, 1);
        __m128i ni6 = _mm256_castsi256_si128( i3 );
        __m128i ni7 = _mm256_extractf128_si256( i3, 1);

        // 计算量化值的和并设置 y[i].s
        const __m128i s0 = _mm_add_epi32(_mm_add_epi32(ni0, ni1), _mm_add_epi32(ni2, ni3));
        const __m128i s1 = _mm_add_epi32(_mm_add_epi32(ni4, ni5), _mm_add_epi32(ni6, ni7));
        y[i].s = d * hsum_i32_4(_mm_add_epi32(s0, s1));

        // 将 int32 转换为 int16
        ni0 = _mm_packs_epi32( ni0, ni1 );
        // 将两个 128 位整数打包成 64 位整数
        ni2 = _mm_packs_epi32( ni2, ni3 );
        ni4 = _mm_packs_epi32( ni4, ni5 );
        ni6 = _mm_packs_epi32( ni6, ni7 );
        // 将 32 位整数打包成 16 位整数
        ni0 = _mm_packs_epi16( ni0, ni2 );
        ni4 = _mm_packs_epi16( ni4, ni6 );

        // 将结果存储到内存中
        _mm_storeu_si128((__m128i *)(y[i].qs +  0), ni0);
        _mm_storeu_si128((__m128i *)(y[i].qs + 16), ni4);
#endif
    }
#elif defined(__riscv_v_intrinsic)

    // 设置向量长度
    size_t vl = __riscv_vsetvl_e32m4(QK8_1);

    for (int i = 0; i < nb; i++) {
        // 加载元素
        vfloat32m4_t v_x   = __riscv_vle32_v_f32m4(x+i*QK8_1, vl);

        // 计算绝对值
        vfloat32m4_t vfabs = __riscv_vfabs_v_f32m4(v_x, vl);
        # 创建一个浮点数向量，初始化为0.0
        vfloat32m1_t tmp   = __riscv_vfmv_v_f_f32m1(0.0, vl);
        # 计算浮点数向量中的最大值
        vfloat32m1_t vmax  = __riscv_vfredmax_vs_f32m4_f32m1(vfabs, tmp, vl);
        # 将最大值转换为标量
        float amax = __riscv_vfmv_f_s_f32m1_f32(vmax);

        # 计算常量d
        const float d  = amax / ((1 << 7) - 1);
        # 计算常量id
        const float id = d ? 1.0f/d : 0.0f;

        # 将d存储到y[i].d中

        vfloat32m4_t x0 = __riscv_vfmul_vf_f32m4(v_x, id, vl);

        # 转换为整数
        vint16m2_t   vi = __riscv_vfncvt_x_f_w_i16m2(x0, vl);
        vint8m1_t    vs = __riscv_vncvt_x_x_w_i8m1(vi, vl);

        # 存储结果
        __riscv_vse8_v_i8m1(y[i].qs , vs, vl);

        # 计算y[i].s的和
        vint16m1_t tmp2 = __riscv_vmv_v_x_i16m1(0, vl);
        vint16m1_t vwrs = __riscv_vwredsum_vs_i8m1_i16m1(vs, tmp2, vl);

        // 使用 RISC-V 汇编指令对一组有符号 8 位整数进行累加求和，并将结果存储在 16 位整数向量寄存器中
        int sum = __riscv_vmv_x_s_i16m1_i16(vwrs);
        // 将累加求和的结果乘以 d，并存储在 y[i].s 中
        y[i].s = sum*d;
    }
#else
    GGML_UNUSED(nb);
    // 如果不满足条件，则使用标量方式进行量化
    quantize_row_q8_1_reference(x, y, k);
#endif
}

void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k) {
    static const int qk = QK4_0;

    // 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    // 计算 nb 的值
    const int nb = k / qk;
    # 遍历从 0 到 nb-1 的整数
    for (int i = 0; i < nb; i++) {
        # 将 x[i].d 转换为 32 位浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        # 遍历从 0 到 qk/2-1 的整数
        for (int j = 0; j < qk/2; ++j) {
            # 计算 x[i].qs[j] & 0x0F，并将结果减去 8
            const int x0 = (x[i].qs[j] & 0x0F) - 8;
            # 计算 x[i].qs[j] >> 4，并将结果减去 8
            const int x1 = (x[i].qs[j] >>   4) - 8;

            # 计算 y[i*qk + j + 0] 的值并赋给它
            y[i*qk + j + 0   ] = x0*d;
            # 计算 y[i*qk + j + qk/2] 的值并赋给它
            y[i*qk + j + qk/2] = x1*d;
        }
    }
}

# 对输入的量化数据进行反量化，将结果存储在 y 中
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k) {
    # 定义常量 qk，并赋值为 QK4_1
    static const int qk = QK4_1;

    # 断言 k 能被 qk 整除
    assert(k % qk == 0);

    # 计算 nb 的值，并赋给它
    const int nb = k / qk;
    # 遍历 nb 次，nb 为输入数据长度除以 qk 的商
    for (int i = 0; i < nb; i++) {
        # 将输入数据 x[i] 的 d 和 m 从 FP16 格式转换为 FP32 格式
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        # 遍历 qk/2 次，qk 为常量，表示每个输入数据的一半长度
        for (int j = 0; j < qk/2; ++j) {
            # 将输入数据 x[i] 的 qs[j] 拆分为两部分，分别赋值给 x0 和 x1
            const int x0 = (x[i].qs[j] & 0x0F);
            const int x1 = (x[i].qs[j] >>   4);

            # 根据公式计算得到输出数据 y[i*qk + j + 0] 和 y[i*qk + j + qk/2]
            y[i*qk + j + 0   ] = x0*d + m;
            y[i*qk + j + qk/2] = x1*d + m;
        }
    }
}

# 定义函数 dequantize_row_q5_0，输入为 block_q5_0 类型的指针 x 和 float 类型的指针 y，以及整数 k
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k) {
    # 定义常量 qk，并赋值为 QK5_0
    static const int qk = QK5_0;

    # 断言 k 能够被 qk 整除
    assert(k % qk == 0);

    # 计算 nb，即 k 除以 qk 的商
    const int nb = k / qk;
# 循环遍历nb次，nb为某个变量
for (int i = 0; i < nb; i++) {
    # 将x[i].d的值从FP16格式转换为FP32格式
    const float d = GGML_FP16_TO_FP32(x[i].d);

    # 定义一个uint32_t类型的变量qh，并将x[i].qh的值拷贝给qh
    uint32_t qh;
    memcpy(&qh, x[i].qh, sizeof(qh));

    # 再次循环遍历qk/2次，qk为某个变量的一半
    for (int j = 0; j < qk/2; ++j) {
        # 计算xh_0和xh_1的值
        const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
        const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

        # 计算x0和x1的值
        const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
        const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

        # 将计算得到的x0和x1乘以d，并存入y数组中
        y[i*qk + j + 0   ] = x0*d;
        y[i*qk + j + qk/2] = x1*d;
    }
}
// 对输入的量化后的数据进行反量化，将结果存储在输出数组中
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k) {
    // 定义量化因子
    static const int qk = QK5_1;

    // 确保输入的 k 是 qk 的整数倍
    assert(k % qk == 0);

    // 计算块的数量
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化后的数据转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float m = GGML_FP16_TO_FP32(x[i].m);

        // 从量化后的数据中提取 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 遍历每个块中的子块
        for (int j = 0; j < qk/2; ++j) {
            // 从 qh 中提取 xh_0 和 xh_1
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            // 将 xh_0 和 xh_1 与子块中的数据进行组合得到 x0 和 x1
            const int x0 = (x[i].qs[j] & 0x0F) | xh_0;
            const int x1 = (x[i].qs[j] >>   4) | xh_1;
// 对输入的量化数据进行反量化，将结果存储在浮点数数组中
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k) {
    // 定义每个量化块的大小
    static const int qk = QK8_0;

    // 确保输入的数据长度是量化块大小的整数倍
    assert(k % qk == 0);

    // 计算输入数据的块数
    const int nb = k / qk;

    // 遍历每个块
    for (int i = 0; i < nb; i++) {
        // 将量化因子转换为浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 遍历每个块中的量化数据，进行反量化操作
        for (int j = 0; j < qk; ++j) {
            y[i*qk + j] = x[i].qs[j]*d;
        }
    }
}
    }
}

//
// 2-6 bit quantization in super-blocks
//

//
// ===================== Helper functions
//

// 定义一个内联函数，用于将浮点数四舍五入为最接近的整数
static inline int nearest_int(float fval) {
    // 断言输入的浮点数不大于 4194303
    assert(fval <= 4194303.f);
    // 将输入浮点数加上一个偏移量，然后转换为整数
    float val = fval + 12582912.f;
    int i; memcpy(&i, &val, sizeof(int));
    return (i & 0x007fffff) - 0x00400000;
}

// 定义一个函数，用于生成量化后的数据
static float make_qx_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, int rmse_type) {
    // 初始化最大值和绝对值最大值
    float max = 0;
    float amax = 0;
    // 遍历数组 x 中的元素，计算绝对值并找出最大值
    for (int i = 0; i < n; ++i) {
        float ax = fabsf(x[i]);
        // 如果绝对值大于最大值，则更新最大值和对应的元素值
        if (ax > amax) { amax = ax; max = x[i]; }
    }
    // 如果最大值小于 1e-30f，则所有元素都为零
    if (amax < 1e-30f) { 
        // 将数组 L 中的所有元素设为 0
        for (int i = 0; i < n; ++i) {
            L[i] = 0;
        }
        // 返回 0
        return 0.f;
    }
    // 计算缩放因子
    float iscale = -nmax / max;
    // 如果 rmse_type 为 0
    if (rmse_type == 0) {
        // 根据缩放因子将 x 中的元素映射到指定范围，并存储到数组 L 中
        for (int i = 0; i < n; ++i) {
            int l = nearest_int(iscale * x[i]);
            L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
        }
        // 返回缩放因子的倒数
        return 1/iscale;
    }
    // 如果 rmse_type 小于 0
    bool return_early = false;
    if (rmse_type < 0) {
    # 将 rmse_type 取反
    rmse_type = -rmse_type;
    # 设置 return_early 为 true
    return_early = true;
    # 计算权重类型
    int weight_type = rmse_type%2;
    # 初始化 sumlx 和 suml2
    float sumlx = 0;
    float suml2 = 0;
    # 遍历 n 个数据点
    for (int i = 0; i < n; ++i) {
        # 计算最近整数
        int l = nearest_int(iscale * x[i]);
        # 将 l 限制在 -nmax 和 nmax-1 之间
        l = MAX(-nmax, MIN(nmax-1, l));
        # 将 l 存入 L 数组
        L[i] = l + nmax;
        # 根据权重类型计算 w
        float w = weight_type == 1 ? x[i] * x[i] : 1;
        # 更新 sumlx 和 suml2
        sumlx += w*x[i]*l;
        suml2 += w*l*l;
    }
    # 计算比例尺
    float scale = sumlx/suml2;
    # 如果 return_early 为 true，则根据条件返回结果
    if (return_early) return suml2 > 0 ? 0.5f*(scale + 1/iscale) : 1/iscale;
    # 计算最佳值
    float best = scale * sumlx;
    # 遍历 -9 到 9 的范围
    for (int is = -9; is <= 9; ++is) {
        # 如果 is 为 0，则跳过本次循环
        if (is == 0) {
            continue;
        }
        // 计算 iscale 的值
        iscale = -(nmax + 0.1f*is) / max;
        // 初始化 sumlx 和 suml2
        sumlx = suml2 = 0;
        // 遍历数组 x
        for (int i = 0; i < n; ++i) {
            // 计算最近整数 l
            int l = nearest_int(iscale * x[i]);
            // 限制 l 的取值范围在 -nmax 和 nmax-1 之间
            l = MAX(-nmax, MIN(nmax-1, l));
            // 根据权重类型计算 w
            float w = weight_type == 1 ? x[i] * x[i] : 1;
            // 更新 sumlx 和 suml2
            sumlx += w*x[i]*l;
            suml2 += w*l*l;
        }
        // 判断条件，更新 L 和 scale 的值
        if (suml2 > 0 && sumlx*sumlx > best*suml2) {
            for (int i = 0; i < n; ++i) {
                // 计算最近整数 l
                int l = nearest_int(iscale * x[i]);
                // 限制 l 的取值范围在 -nmax 和 nmax-1 之间，并更新 L[i] 的值
                L[i] = nmax + MAX(-nmax, MIN(nmax-1, l));
            }
            // 更新 scale 和 best 的值
            scale = sumlx/suml2; best = scale*sumlx;
        }
    }
    // 返回最终的 scale 值
    return scale;
}
# 计算第三个四分位数的分位数，并返回分位数的值
static float make_q3_quants(int n, int nmax, const float * restrict x, int8_t * restrict L, bool do_rmse) {
    float max = 0;  # 初始化最大值
    float amax = 0;  # 初始化绝对值最大值
    for (int i = 0; i < n; ++i) {  # 遍历数组 x
        float ax = fabsf(x[i]);  # 计算 x[i] 的绝对值
        if (ax > amax) { amax = ax; max = x[i]; }  # 更新绝对值最大值和最大值
    }
    if (!amax) {  # 如果绝对值最大值为 0，即所有值都为 0
        for (int i = 0; i < n; ++i) { L[i] = 0; }  # 将 L 数组全部置为 0
        return 0.f;  # 返回 0
    }
    float iscale = -nmax / max;  # 计算缩放因子
    if (do_rmse) {  # 如果需要计算均方根误差
        float sumlx = 0;  # 初始化 sumlx
        float suml2 = 0;  # 初始化 suml2
        for (int i = 0; i < n; ++i) {  # 遍历数组 x
            int l = nearest_int(iscale * x[i]);  # 计算缩放后的值并取最近整数
            l = MAX(-nmax, MIN(nmax-1, l));  # 将 l 限制在 -nmax 和 nmax-1 之间
            L[i] = l;  # 将 l 赋值给 L 数组
            // 计算 x[i] 的平方
            float w = x[i]*x[i];
            // 更新 sumlx
            sumlx += w*x[i]*l;
            // 更新 suml2
            suml2 += w*l*l;
        }
        // 迭代 5 次
        for (int itry = 0; itry < 5; ++itry) {
            // 记录改变的数量
            int n_changed = 0;
            // 遍历数组
            for (int i = 0; i < n; ++i) {
                // 计算 x[i] 的平方
                float w = x[i]*x[i];
                // 计算 slx
                float slx = sumlx - w*x[i]*L[i];
                // 如果 slx 大于 0
                if (slx > 0) {
                    // 计算 sl2
                    float sl2 = suml2 - w*L[i]*L[i];
                    // 计算新的 l
                    int new_l = nearest_int(x[i] * sl2 / slx);
                    // 限制 new_l 的范围
                    new_l = MAX(-nmax, MIN(nmax-1, new_l));
                    // 如果 new_l 不等于 L[i]
                    if (new_l != L[i]) {
                        // 更新 slx 和 sl2
                        slx += w*x[i]*new_l;
                        sl2 += w*new_l*new_l;
                        // 如果满足条件
                        if (sl2 > 0 && slx*slx*suml2 > sumlx*sumlx*sl2) {
                            // 更新 L[i], sumlx, suml2，并增加改变的数量
                            L[i] = new_l; sumlx = slx; suml2 = sl2;
                            ++n_changed;
                        }
    }
        } // 结束内层循环
    } // 结束外层循环
    // 如果没有发生改变，跳出循环
    if (!n_changed) {
        break;
    }
}
// 对数组 L 进行操作
for (int i = 0; i < n; ++i) {
    // 对数组 L 进行操作
    L[i] += nmax;
}
// 返回 sumlx 除以 suml2 的结果
return sumlx / suml2;
}
// 对数组进行操作
for (int i = 0; i < n; ++i) {
    // 将 iscale 乘以 x[i] 并取最近的整数
    int l = nearest_int(iscale * x[i]);
    // 将 l 限制在 -nmax 和 nmax-1 之间
    l = MAX(-nmax, MIN(nmax-1, l));
    // 对数组 L 进行操作
    L[i] = l + nmax;
}
// 返回 1 除以 iscale 的结果
return 1/iscale;
}
# 计算并返回一维数组的分位数压缩值
static float make_qkx1_quants(int n, int nmax, const float * restrict x, uint8_t * restrict L, float * restrict the_min,
        int ntry, float alpha) {
    # 初始化最小值和最大值为数组第一个元素
    float min = x[0];
    float max = x[0];
    # 遍历数组，更新最小值和最大值
    for (int i = 1; i < n; ++i) {
        if (x[i] < min) min = x[i];
        if (x[i] > max) max = x[i];
    }
    # 如果最大值等于最小值，将压缩值设为0
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = 0;
        return 0.f;
    }
    # 如果最小值大于0，将最小值设为0
    if (min > 0) min = 0;
    # 计算缩放比例
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    # 迭代计算压缩值
    for (int itry = 0; itry < ntry; ++itry) {
        float sumlx = 0; int suml2 = 0;
        bool did_change = false;
        # 遍历数组，计算压缩值
        for (int i = 0; i < n; ++i) {
        // 计算最接近的整数，根据缩放比例和数据范围
        int l = nearest_int(iscale*(x[i] - min));
        // 将 l 限制在 0 和 nmax 之间
        l = MAX(0, MIN(nmax, l));
        // 如果 l 与 L[i] 不相等，则更新 L[i]，并标记 did_change 为 true
        if (l != L[i]) {
            L[i] = l;
            did_change = true;
        }
        // 计算 sumlx 和 suml2
        sumlx += (x[i] - min)*l;
        suml2 += l*l;
    }
    // 计算新的缩放比例 scale
    scale = sumlx/suml2;
    // 计算 sum
    float sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += x[i] - scale*L[i];
    }
    // 更新 min
    min = alpha*min + (1 - alpha)*sum/n;
    if (min > 0) min = 0;
    // 更新 iscale
    iscale = 1/scale;
    // 如果没有发生改变，则跳出循环
    if (!did_change) break;
    // 返回 -min 作为结果
    *the_min = -min;
    // 返回比例
    return scale;
}

// 创建 QKX2 量化
static float make_qkx2_quants(int n, int nmax, const float * restrict x, const float * restrict weights,
        uint8_t * restrict L, float * restrict the_min, uint8_t * restrict Laux,
        float rmin, float rdelta, int nstep, bool use_mad) {
    // 初始化最小值为 x[0]
    float min = x[0];
    // 初始化最大值为 x[0]
    float max = x[0];
    // 初始化权重和为 weights[0]
    float sum_w = weights[0];
    // 初始化加权和为 sum_w * x[0]
    float sum_x = sum_w * x[0];
#ifdef HAVE_BUGGY_APPLE_LINKER
    // 使用 'volatile' 防止展开循环，并解决 Apple ld64 1015.7 中的一个 bug
    for (volatile int i = 1; i < n; ++i) {
#else
    // 遍历数组 x
    for (int i = 1; i < n; ++i) {
#endif
        // 如果 x[i] 小于最小值，则更新最小值
        if (x[i] < min) min = x[i];
        // 如果 x[i] 大于最大值，则更新最大值
        if (x[i] > max) max = x[i];
        // 获取权重值
        float w = weights[i];
        // 更新加权和
        sum_w += w;
    // 计算加权和
    sum_x += w * x[i];
    // 如果最小值大于0，则将最小值设为0
    if (min > 0) min = 0;
    // 如果最大值等于最小值，则将 L 数组全部设为0，并返回结果
    if (max == min) {
        for (int i = 0; i < n; ++i) L[i] = 0;
        *the_min = -min;
        return 0.f;
    }
    // 计算缩放比例
    float iscale = nmax/(max - min);
    float scale = 1/iscale;
    float best_mad = 0;
    // 遍历数组，计算最小绝对偏差或者最小平方差
    for (int i = 0; i < n; ++i) {
        int l = nearest_int(iscale*(x[i] - min));
        L[i] = MAX(0, MIN(nmax, l));
        float diff = scale * L[i] + min - x[i];
        diff = use_mad ? fabsf(diff) : diff * diff;
        float w = weights[i];
        best_mad += w * diff;
    }
    // 如果步长小于1，则执行以下操作
    if (nstep < 1) {
        *the_min = -min;  // 将最小值取反赋值给 the_min
        return scale;  // 返回 scale 变量的值
    }
    for (int is = 0; is <= nstep; ++is) {  // 循环，is 从 0 到 nstep
        iscale = (rmin + rdelta*is + nmax)/(max - min);  // 计算 iscale 的值
        float sum_l = 0, sum_l2 = 0, sum_xl = 0;  // 初始化三个浮点数变量
        for (int i = 0; i < n; ++i) {  // 循环，i 从 0 到 n
            int l = nearest_int(iscale*(x[i] - min));  // 计算 l 的值
            l = MAX(0, MIN(nmax, l));  // 将 l 的值限制在 0 和 nmax 之间
            Laux[i] = l;  // 将 l 的值赋给 Laux 数组的第 i 个元素
            float w = weights[i];  // 获取 weights 数组的第 i 个元素的值
            sum_l += w*l;  // 计算 sum_l 的值
            sum_l2 += w*l*l;  // 计算 sum_l2 的值
            sum_xl += w*l*x[i];  // 计算 sum_xl 的值
        }
        float D = sum_w * sum_l2 - sum_l * sum_l;  // 计算 D 的值
        if (D > 0) {  // 如果 D 大于 0
            float this_scale = (sum_w * sum_xl - sum_x * sum_l)/D;  // 计算 this_scale 的值
            float this_min   = (sum_l2 * sum_x - sum_l * sum_xl)/D;  // 计算 this_min 的值
            if (this_min > 0) {  // 如果 this_min 大于 0
// 初始化 this_min 为 0
this_min = 0;
// 计算 this_scale 为 sum_xl 除以 sum_l2
this_scale = sum_xl / sum_l2;
// 如果条件成立，执行以下代码块
if (condition) {
    // 初始化 mad 为 0
    float mad = 0;
    // 遍历 n 次
    for (int i = 0; i < n; ++i) {
        // 计算 diff 为 this_scale 乘以 Laux[i] 加上 this_min 减去 x[i]
        float diff = this_scale * Laux[i] + this_min - x[i];
        // 如果 use_mad 为真，计算 diff 的绝对值，否则计算 diff 的平方
        diff = use_mad ? fabsf(diff) : diff * diff;
        // 获取权重值
        float w = weights[i];
        // 计算 mad 为 mad 加上 w 乘以 diff
        mad += w * diff;
    }
    // 如果 mad 小于 best_mad，执行以下代码块
    if (mad < best_mad) {
        // 遍历 n 次
        for (int i = 0; i < n; ++i) {
            // 将 Laux[i] 赋值给 L[i]
            L[i] = Laux[i];
        }
        // 更新 best_mad 为 mad
        best_mad = mad;
        // 更新 scale 为 this_scale
        scale = this_scale;
        // 更新 min 为 this_min
        min = this_min;
    }
}
    *the_min = -min;
    // 将变量 min 取反后赋值给 the_min
    return scale;
    // 返回变量 scale 的值
}

#if QK_K == 256
static inline void get_scale_min_k4(int j, const uint8_t * restrict q, uint8_t * restrict d, uint8_t * restrict m) {
    // 如果 j 小于 4，则执行以下操作
    if (j < 4) {
        *d = q[j] & 63; *m = q[j + 4] & 63;
    } else {
        *d = (q[j+4] & 0xF) | ((q[j-4] >> 6) << 4);
        *m = (q[j+4] >>  4) | ((q[j-0] >> 6) << 4);
    }
}
#endif
// 如果 QK_K 等于 256，则定义一个内联函数 get_scale_min_k4，根据参数 j、q 和 m 的值进行一系列操作

//========================- 2-bit (de)-quantization

void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k) {
    // 断言 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值为 k 除以 QK_K
    const int nb = k / QK_K;
```

    // 定义长度为 QK_K 的无符号 8 位整数数组
    uint8_t L[QK_K];
    // 定义长度为 16 的无符号 8 位整数数组
    uint8_t Laux[16];
    // 定义长度为 16 的浮点数数组
    float   weights[16];
    // 定义长度为 QK_K/16 的浮点数数组
    float mins[QK_K/16];
    // 定义长度为 QK_K/16 的浮点数数组
    float scales[QK_K/16];

    // 定义常量 q4scale 并赋值为 15.0
    const float q4scale = 15.f;

    // 循环，i 从 0 到 nb
    for (int i = 0; i < nb; i++) {
        // 初始化最大比例为 0
        float max_scale = 0; // as we are deducting the min, scales are always positive
        // 初始化最大最小值为 0
        float max_min = 0;
        // 循环，j 从 0 到 QK_K/16
        for (int j = 0; j < QK_K/16; ++j) {
            // 循环，l 从 0 到 16
            for (int l = 0; l < 16; ++l) 
                // 计算权重数组中每个元素的绝对值并赋值给 weights 数组
                weights[l] = fabsf(x[16*j + l]);
            // 调用 make_qkx2_quants 函数计算 scales[j] 的值
            scales[j] = make_qkx2_quants(16, 3, x + 16*j, weights, L + 16*j, &mins[j], Laux, -0.5f, 0.1f, 15, true);
            // 将 scales[j] 的值赋给 scale
            float scale = scales[j];
            // 如果 scale 大于 max_scale，则将 scale 赋给 max_scale
            if (scale > max_scale) {
                max_scale = scale;
            }
            // 将 mins[j] 的值赋给 min
            float min = mins[j];
        // 如果最小值大于当前最大最小值，则更新最大最小值
        if (min > max_min) {
            max_min = min;
        }
    }

    // 如果最大比例大于0
    if (max_scale > 0) {
        // 计算比例
        float iscale = q4scale/max_scale;
        // 遍历并计算每个元素的值
        for (int j = 0; j < QK_K/16; ++j) {
            int l = nearest_int(iscale*scales[j]);
            y[i].scales[j] = l;
        }
        // 将结果转换为FP16格式
        y[i].d = GGML_FP32_TO_FP16(max_scale/q4scale);
    } else {
        // 如果最大比例小于等于0，则将所有元素的值设为0
        for (int j = 0; j < QK_K/16; ++j) y[i].scales[j] = 0;
        // 将结果转换为FP16格式
        y[i].d = GGML_FP32_TO_FP16(0.f);
    }
    // 如果最大最小值大于0
    if (max_min > 0) {
        // 计算比例
        float iscale = q4scale/max_min;
        // 遍历并计算每个元素的值
        for (int j = 0; j < QK_K/16; ++j) {
            int l = nearest_int(iscale*mins[j]);
            // 将l左移4位后与y[i].scales[j]进行按位或操作
            y[i].scales[j] |= (l << 4);
        }
        // 计算dmin的值
        y[i].dmin = GGML_FP32_TO_FP16(max_min/q4scale);
    } else {
        // 如果条件不满足，将dmin的值设为0
        y[i].dmin = GGML_FP32_TO_FP16(0.f);
    }
    // 循环遍历QK_K/16次
    for (int j = 0; j < QK_K/16; ++j) {
        // 计算d的值
        const float d = GGML_FP16_TO_FP32(y[i].d) * (y[i].scales[j] & 0xF);
        // 如果d为0，则跳过本次循环
        if (!d) continue;
        // 计算dm的值
        const float dm = GGML_FP16_TO_FP32(y[i].dmin) * (y[i].scales[j] >> 4);
        // 循环遍历16次
        for (int ii = 0; ii < 16; ++ii) {
            // 计算l的值
            int l = nearest_int((x[16*j + ii] + dm)/d);
            // 将l的值限制在0到3之间
            l = MAX(0, MIN(3, l));
            // 将l的值赋给L[16*j + ii]
            L[16*j + ii] = l;
        }
    }

    // 如果QK_K等于256，则执行以下代码
#if QK_K == 256
        // 循环遍历QK_K次，每次增加128
        for (int j = 0; j < QK_K; j += 128) {
            // 循环遍历32次
            for (int l = 0; l < 32; ++l) {
        // 对数组进行解量化操作，将结果存储到 y[i].qs[j/4 + l] 中
        y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
        // 对数组进行解量化操作，将结果存储到 y[i].qs[l] 中
        y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
        
        // 对 x 进行增加 QK_K 的操作
        x += QK_K;
    }
}

// 对 block_q2_K 数组进行解量化操作，将结果存储到 float 数组 y 中
void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k) {
    // 断言 k 能够被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;

    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 转换为 32 位浮点数
        const float d = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].dmin 转换为 32 位浮点数
        const float min = GGML_FP16_TO_FP32(x[i].dmin);

        // 获取 x[i].qs 的指针
        const uint8_t * q = x[i].qs;

        // 如果 QK_K 等于 256
        int is = 0;
        float dl, ml;
        for (int n = 0; n < QK_K; n += 128) {
            int shift = 0;
            for (int j = 0; j < 4; ++j) {

                // 获取 x[i].scales 中的值
                uint8_t sc = x[i].scales[is++];
                // 计算 dl 和 ml
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 对每个 l，计算 y 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l] >> shift) & 3)) - ml;

                // 获取 x[i].scales 中的值
                sc = x[i].scales[is++];
                // 计算 dl 和 ml
                dl = d * (sc & 0xF); ml = min * (sc >> 4);
                // 对每个 l，计算 y 的值
                for (int l = 0; l < 16; ++l) *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3)) - ml;
#if
        // 如果条件满足，则执行以下代码块
        float dl1 = d * (x[i].scales[0] & 0xF), ml1 = min * (x[i].scales[0] >> 4);
        float dl2 = d * (x[i].scales[1] & 0xF), ml2 = min * (x[i].scales[1] >> 4);
        float dl3 = d * (x[i].scales[2] & 0xF), ml3 = min * (x[i].scales[2] >> 4);
        float dl4 = d * (x[i].scales[3] & 0xF), ml4 = min * (x[i].scales[3] >> 4);
        // 循环遍历16次
        for (int l = 0; l < 16; ++l) {
            // 计算并赋值y数组的值
            y[l+ 0] = dl1 * ((int8_t)((q[l] >> 0) & 3)) - ml1;
            y[l+16] = dl2 * ((int8_t)((q[l] >> 2) & 3)) - ml2;
            y[l+32] = dl3 * ((int8_t)((q[l] >> 4) & 3)) - ml3;
            y[l+48] = dl4 * ((int8_t)((q[l] >> 6) & 3)) - ml4;
        }
        // y指针移动QK_K个位置
        y += QK_K;
#endif
    }
}
// 对输入的一行数据进行 Q2_K 量化
void quantize_row_q2_K(const float * restrict x, void * restrict vy, int k) {
    // 调用参考实现的 Q2_K 量化函数
    quantize_row_q2_K_reference(x, vy, k);
}

// 对输入的数据进行 Q2_K 量化，并返回结果
size_t ggml_quantize_q2_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms

    // 对输入数据进行分块处理
    for (int j = 0; j < n; j += k) {
        // 将输出数据转换为 block_q2_K 类型
        block_q2_K * restrict y = (block_q2_K *)dst + j/QK_K;
        // 调用参考实现的 Q2_K 量化函数
        quantize_row_q2_K_reference(src + j, y, k);
    }
    // 返回结果数据的大小
    return (n/QK_K*sizeof(block_q2_K));
}

//========================= 3-bit (de)-quantization

// 对输入的一行数据进行 Q3_K 量化
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    const int nb = k / QK_K;
    # 定义长度为 QK_K 的 int8_t 类型数组
    int8_t L[QK_K];
    # 定义长度为 QK_K/16 的 float 类型数组
    float scales[QK_K / 16];

    # 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        # 初始化最大比例和最大绝对值
        float max_scale = 0;
        float amax = 0;
        # 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            # 调用 make_q3_quants 函数，计算 scales[j] 的值
            scales[j] = make_q3_quants(16, 4, x + 16*j, L + 16*j, true);
            # 计算 scales[j] 的绝对值
            float scale = fabsf(scales[j]);
            # 如果 scale 大于 amax，则更新 amax 和 max_scale 的值
            if (scale > amax) {
                amax = scale; max_scale = scales[j];
            }
        }

        # 如果 QK_K 等于 256
        # 将 y[i].scales 数组的前 12 个元素置为 0
        memset(y[i].scales, 0, 12);
        # 如果 max_scale 不为 0
        # 计算 iscale 的值
        float iscale = -32.f/max_scale;
        # 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
        // 将iscale乘以scales[j]并取最近的整数值
        int8_t l = nearest_int(iscale*scales[j]);
        // 将l限制在-32和31之间，并加上32
        l = MAX(-32, MIN(31, l)) + 32;
        // 如果j小于8，将l的低4位存入y[i].scales[j]中
        if (j < 8) {
            y[i].scales[j] = l & 0xF;
        } else {
            // 如果j大于等于8，将l的低4位左移4位后存入y[i].scales[j-8]中
            y[i].scales[j-8] |= ((l & 0xF) << 4);
        }
        // l右移4位
        l >>= 4;
        // 将l的值存入y[i].scales[j%4 + 8]中，根据j的值确定存入的位置和位移
        y[i].scales[j%4 + 8] |= (l << (2*(j/4)));
    }
    // 如果iscale不为0，将1/iscale转换为FP16格式后存入y[i].d中
    y[i].d = GGML_FP32_TO_FP16(1/iscale);
} else {
    // 如果iscale为0，将0.f转换为FP16格式后存入y[i].d中
    y[i].d = GGML_FP32_TO_FP16(0.f);
}

int8_t sc;
// 遍历y[i].scales数组，根据j的值确定取低4位或高4位，并进行相应的位移和操作
for (int j = 0; j < QK_K/16; ++j) {
    sc = j < 8 ? y[i].scales[j] & 0xF : y[i].scales[j-8] >> 4;
    sc = (sc | (((y[i].scales[8 + j%4] >> (2*(j/4))) & 3) << 4)) - 32;
    // 将y[i].d转换为FP32格式后乘以sc，并存入d中
    float d = GGML_FP16_TO_FP32(y[i].d) * sc;
// 如果 d 为假值，则跳过当前循环，继续下一次循环
if (!d) {
    continue;
}
// 遍历 16 次，计算并存储每个元素的值
for (int ii = 0; ii < 16; ++ii) {
    // 计算 x[16*j + ii] 除以 d 的最接近整数值
    int l = nearest_int(x[16*j + ii]/d);
    // 将 l 限制在 -4 到 3 之间
    l = MAX(-4, MIN(3, l));
    // 将 l 加上 4 后存储到 L 数组中
    L[16*j + ii] = l + 4;
}
// 如果未定义 max_scale，则执行以下代码
#else
// 如果 max_scale 存在，则执行以下代码
if (max_scale) {
    // 计算 iscale 的值
    float iscale = -8.f/max_scale;
    // 遍历 QK_K/16 次，计算并存储每个元素的值
    for (int j = 0; j < QK_K/16; j+=2) {
        // 计算 iscale 乘以 scales[j] 的最接近整数值
        int l1 = nearest_int(iscale*scales[j]);
        // 将 l1 限制在 -8 到 7 之间，并加上 8 后存储到 y[i].scales[j/2] 中
        l1 = 8 + MAX(-8, MIN(7, l1));
        // 计算 iscale 乘以 scales[j+1] 的最接近整数值
        int l2 = nearest_int(iscale*scales[j+1]);
        // 将 l2 限制在 -8 到 7 之间，并加上 8 后左移 4 位，然后与 l1 相或后存储到 y[i].scales[j/2] 中
        y[i].scales[j/2] = l1 | (l2 << 4);
    }
    // 将 GGML_FP32_TO_FP16(1/iscale) 的值存储到 y[i].d 中
    y[i].d = GGML_FP32_TO_FP16(1/iscale);
}
        } else {
            // 如果条件不满足，执行以下操作
            for (int j = 0; j < QK_K/16; j+=2) {
                // 将y[i].scales[j/2]的值设为0
                y[i].scales[j/2] = 0;
            }
            // 将y[i].d的值设为0.f的32位浮点数转换成16位浮点数的值
            y[i].d = GGML_FP32_TO_FP16(0.f);
        }
        // 循环遍历QK_K/16次
        for (int j = 0; j < QK_K/16; ++j) {
            // 计算s的值
            int s = j%2 == 0 ? y[i].scales[j/2] & 0xF : y[i].scales[j/2] >> 4;
            // 计算d的值
            float d = GGML_FP16_TO_FP32(y[i].d) * (s - 8);
            // 如果d为0，则跳过当前循环
            if (!d) {
                continue;
            }
            // 循环遍历16次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算l的值
                int l = nearest_int(x[16*j + ii]/d);
                // 将l的值限制在-4和3之间
                l = MAX(-4, MIN(3, l));
                // 将L[16*j + ii]的值设为l + 4
                L[16*j + ii] = l + 4;
            }
        }
#endif
        // 使用memset函数将y[i].hmask数组的前QK_K/8个元素初始化为0
        memset(y[i].hmask, 0, QK_K/8);
        
        // 将每个quant的高位存储到y[i].hmask数组中
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
        
        // 如果QK_K等于256，则将L数组中的数据按照一定规则存储到y[i].qs数组中
        #if QK_K == 256
        for (int j = 0; j < QK_K; j += 128) {
            for (int l = 0; l < 32; ++l) {
                y[i].qs[j/4 + l] = L[j + l] | (L[j + l + 32] << 2) | (L[j + l + 64] << 4) | (L[j + l + 96] << 6);
            }
        }
        #else
        // 对数组进行循环，将每个元素按位组合成一个32位整数
        for (int l = 0; l < 16; ++l) {
            y[i].qs[l] = L[l] | (L[l + 16] << 2) | (L[l + 32] << 4) | (L[l + 48] << 6);
        }
#endif

        // 对变量 x 进行增加 QK_K 的操作
        x += QK_K;
    }
}

#if QK_K == 256
// 对输入的 block_q3_K 数组进行反量化，结果存储在 float 数组中
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 断言 k 能够被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;

    // 定义两个 32 位整数的掩码
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    // 定义一个长度为 4 的无符号整数数组
    uint32_t aux[4];
    // 将 aux 强制转换为 int8_t 类型的指针，并赋值给 scales
    const int8_t * scales = (const int8_t*)aux;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 将 x[i].d 转换为 32 位浮点数
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 定义指向 x[i].qs 和 x[i].hmask 的指针
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;
        // 初始化 m 为 1
        uint8_t m = 1;

        // 将 x[i].scales 的内容复制到 aux 数组
        memcpy(aux, x[i].scales, 12);
        // 对 aux 数组进行位操作
        uint32_t tmp = aux[2];
        aux[2] = ((aux[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        aux[3] = ((aux[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        aux[0] = (aux[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        aux[1] = (aux[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);

        // 初始化 is 和 dl
        int is = 0;
        float dl;
        // 循环遍历 QK_K 次，每次增加 128
        for (int n = 0; n < QK_K; n += 128) {
            // 初始化 shift 为 0
            int shift = 0;
            // 循环遍历 4 次
            for (int j = 0; j < 4; ++j) {
// 使用当前比例因子和温度值计算出偏移量
dl = d_all * (scales[is++] - 32);
// 遍历16个元素，根据偏移量和位移值计算出结果并存入y中
for (int l = 0; l < 16; ++l) {
    *y++ = dl * ((int8_t)((q[l+ 0] >> shift) & 3) - ((hm[l+ 0] & m) ? 0 : 4));
}

// 使用当前比例因子和温度值计算出偏移量
dl = d_all * (scales[is++] - 32);
// 遍历16个元素，根据偏移量和位移值计算出结果并存入y中
for (int l = 0; l < 16; ++l) {
    *y++ = dl * ((int8_t)((q[l+16] >> shift) & 3) - ((hm[l+16] & m) ? 0 : 4));
}

// 位移值增加2
shift += 2;
// m左移1位
m <<= 1;
// q指针向后移动32个元素
q += 32;
// 对输入的量化后的数据进行反量化处理，将结果存储在输出数组中
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 确保 QK_K 的值为 64
    assert(QK_K == 64);
    // 计算块的数量
    const int nb = k / QK_K;

    for (int i = 0; i < nb; i++) {
        // 将所有数据转换为单精度浮点数
        const float d_all = GGML_FP16_TO_FP32(x[i].d);

        // 获取量化参数和掩码
        const uint8_t * restrict q = x[i].qs;
        const uint8_t * restrict hm = x[i].hmask;

        // 计算四个反量化后的值
        const float d1 = d_all * ((x[i].scales[0] & 0xF) - 8);
        const float d2 = d_all * ((x[i].scales[0] >>  4) - 8);
        const float d3 = d_all * ((x[i].scales[1] & 0xF) - 8);
        const float d4 = d_all * ((x[i].scales[1] >>  4) - 8);

        // 对每个块中的8个元素进行反量化处理
        for (int l=0; l<8; ++l) {
            // 获取掩码值
            uint8_t h = hm[l];
            // 计算反量化后的值并存储在输出数组中
            y[l+ 0] = d1 * ((int8_t)((q[l+0] >> 0) & 3) - ((h & 0x01) ? 0 : 4));
            // 对输入数据进行量化处理，将结果存储到数组 y 中
            y[l+ 8] = d1 * ((int8_t)((q[l+8] >> 0) & 3) - ((h & 0x02) ? 0 : 4));
            y[l+16] = d2 * ((int8_t)((q[l+0] >> 2) & 3) - ((h & 0x04) ? 0 : 4));
            y[l+24] = d2 * ((int8_t)((q[l+8] >> 2) & 3) - ((h & 0x08) ? 0 : 4));
            y[l+32] = d3 * ((int8_t)((q[l+0] >> 4) & 3) - ((h & 0x10) ? 0 : 4));
            y[l+40] = d3 * ((int8_t)((q[l+8] >> 4) & 3) - ((h & 0x20) ? 0 : 4));
            y[l+48] = d4 * ((int8_t)((q[l+0] >> 6) & 3) - ((h & 0x40) ? 0 : 4));
            y[l+56] = d4 * ((int8_t)((q[l+8] >> 6) & 3) - ((h & 0x80) ? 0 : 4));
        }
        // 移动 y 的指针到下一个块的起始位置
        y += QK_K;
    }
}
#endif

// 调用 quantize_row_q3_K_reference 函数对输入数据进行量化处理
void quantize_row_q3_K(const float * restrict x, void * restrict vy, int k) {
    quantize_row_q3_K_reference(x, vy, k);
}

// 对输入数据进行量化处理，并返回结果的大小
size_t ggml_quantize_q3_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    (void)hist; // TODO: collect histograms
    // 对输入数组进行分块量化，每次处理 k 个元素
    for (int j = 0; j < n; j += k) {
        // 将 dst 强制转换为 block_q3_K 类型的指针，并偏移 j/QK_K 个元素
        block_q3_K * restrict y = (block_q3_K *)dst + j/QK_K;
        // 调用 quantize_row_q3_K_reference 函数，对 src + j 的数据进行量化，结果存储在 y 中
        quantize_row_q3_K_reference(src + j, y, k);
    }
    // 返回处理的数据量
    return (n/QK_K*sizeof(block_q3_K));
}

// ====================== 4-bit (de)-quantization

// 对输入数组进行 4-bit 分块量化
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k) {
    // 断言 k 可以被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算分块数量
    const int nb = k / QK_K;

    // 定义一些临时数组和变量
    uint8_t L[QK_K];
    uint8_t Laux[32];
    float   weights[32];
    float mins[QK_K/32];
    float scales[QK_K/32];

    // 对每个分块进行处理
    for (int i = 0; i < nb; i++) {
        float max_scale = 0; // 初始化最大比例为0
        float max_min = 0; // 初始化最大最小值为0
        for (int j = 0; j < QK_K/32; ++j) { // 循环遍历QK_K/32次
            //scales[j] = make_qkx1_quants(32, 15, x + 32*j, L + 32*j, &mins[j], 9, 0.5f); 
            // 使用make_qkx1_quants函数计算比例，并存储在scales数组中
            float sum_x2 = 0; // 初始化sum_x2为0
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l]; // 计算32个元素的平方和
            float av_x = sqrtf(sum_x2/32); // 计算平均值
            for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]); // 计算权重
            scales[j] = make_qkx2_quants(32, 15, x + 32*j, weights, L + 32*j, &mins[j], Laux, -1.f, 0.1f, 20, false); // 使用make_qkx2_quants函数计算比例，并存储在scales数组中
            float scale = scales[j]; // 获取当前比例
            if (scale > max_scale) { // 如果当前比例大于最大比例
                max_scale = scale; // 更新最大比例
            }
            float min = mins[j]; // 获取当前最小值
            if (min > max_min) { // 如果当前最小值大于最大最小值
                max_min = min; // 更新最大最小值
            }
        }
# 如果 QK_K 等于 256，则执行以下操作
# 计算最大比例的倒数，如果最大比例大于0，则为 63 除以最大比例，否则为 0
float inv_scale = max_scale > 0 ? 63.f/max_scale : 0.f;
# 计算最小比例的倒数，如果最小比例大于0，则为 63 除以最小比例，否则为 0
float inv_min   = max_min   > 0 ? 63.f/max_min   : 0.f;
# 遍历 QK_K/32 次
for (int j = 0; j < QK_K/32; ++j) {
    # 将比例值转换为最接近的整数，并限制在 0 到 63 之间
    uint8_t ls = nearest_int(inv_scale*scales[j]);
    uint8_t lm = nearest_int(inv_min*mins[j]);
    ls = MIN(63, ls);
    lm = MIN(63, lm);
    # 根据索引 j 的值进行条件判断和赋值操作
    if (j < 4) {
        y[i].scales[j] = ls;
        y[i].scales[j+4] = lm;
    } else {
        y[i].scales[j+4] = (ls & 0xF) | ((lm & 0xF) << 4);
        y[i].scales[j-4] |= ((ls >> 4) << 6);
        y[i].scales[j-0] |= ((lm >> 4) << 6);
    }
}
# 将最大比例值转换为 FP16 格式并赋值给 y[i].d
y[i].d = GGML_FP32_TO_FP16(max_scale/63.f);
# 将最小比例值转换为 FP16 格式并赋值给 y[i].dmin
y[i].dmin = GGML_FP32_TO_FP16(max_min/63.f);
        // 声明两个无符号8位整数变量
        uint8_t sc, m;
        // 循环遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            // 调用函数获取比例和最小值
            get_scale_min_k4(j, y[i].scales, &sc, &m);
            // 计算d的值
            const float d = GGML_FP16_TO_FP32(y[i].d) * sc;
            // 如果d为0，则跳过本次循环
            if (!d) continue;
            // 计算dm的值
            const float dm = GGML_FP16_TO_FP32(y[i].dmin) * m;
            // 再次循环遍历32次
            for (int ii = 0; ii < 32; ++ii) {
                // 计算l的值
                int l = nearest_int((x[32*j + ii] + dm)/d);
                // 将l的值限制在0到15之间
                l = MAX(0, MIN(15, l));
                // 将l的值赋给L数组对应位置
                L[32*j + ii] = l;
            }
        }
#else
        // 声明并初始化浮点数变量s_factor
        const float s_factor = 15.f;
        // 计算inv_scale的值
        float inv_scale = max_scale > 0 ? s_factor/max_scale : 0.f;
        // 计算inv_min的值
        float inv_min   = max_min   > 0 ? s_factor/max_min   : 0.f;
        // 计算d1的值
        int d1 = nearest_int(inv_scale*scales[0]);
        // 计算m1的值
        int m1 = nearest_int(inv_min*mins[0]);
        // 计算d2的值
        int d2 = nearest_int(inv_scale*scales[1]);
        // 计算m2的值
        int m2 = nearest_int(inv_min*mins[1]);
        # 设置y[i]的第一个scales值为d1 | (m1 << 4)
        y[i].scales[0] = d1 | (m1 << 4);
        # 设置y[i]的第二个scales值为d2 | (m2 << 4)
        y[i].scales[1] = d2 | (m2 << 4);
        # 设置y[i]的第一个d值为将max_scale/s_factor转换为FP16格式
        y[i].d[0] = GGML_FP32_TO_FP16(max_scale/s_factor);
        # 设置y[i]的第二个d值为将max_min/s_factor转换为FP16格式
        y[i].d[1] = GGML_FP32_TO_FP16(max_min/s_factor);

        # 初始化sumlx为0
        float sumlx = 0;
        # 初始化suml2为0
        int   suml2 = 0;
        # 循环遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            # 获取y[i]的第j个scales值的低4位
            const uint8_t sd = y[i].scales[j] & 0xF;
            # 获取y[i]的第j个scales值的高4位
            const uint8_t sm = y[i].scales[j] >>  4;
            # 计算d值，将y[i]的第一个d值转换为FP32格式后乘以sd
            const float d = GGML_FP16_TO_FP32(y[i].d[0]) * sd;
            # 如果d为0，则跳过本次循环
            if (!d) continue;
            # 计算m值，将y[i]的第二个d值转换为FP32格式后乘以sm
            const float m = GGML_FP16_TO_FP32(y[i].d[1]) * sm;
            # 循环遍历32次
            for (int ii = 0; ii < 32; ++ii) {
                # 计算l值，将(x[32*j + ii] + m)/d取最近的整数
                int l = nearest_int((x[32*j + ii] + m)/d);
                # 将l限制在0到15之间
                l = MAX(0, MIN(15, l));
                # 将l值赋给L数组的对应位置
                L[32*j + ii] = l;
                # 计算sumlx的累加值
                sumlx += (x[32*j + ii] + m)*l*sd;
                # 计算suml2的累加值
                suml2 += l*l*sd*sd;
            }
        }
        // 如果 suml2 不为 0，则将 sumlx/suml2 转换为 FP16 存入 y[i].d[0] 中
        if (suml2) {
            y[i].d[0] = GGML_FP32_TO_FP16(sumlx/suml2);
        }
#endif
        // 定义 uint8_t 类型指针 q，指向 y[i].qs
        uint8_t * q = y[i].qs;
        // 循环遍历 QK_K，每次增加 64
        for (int j = 0; j < QK_K; j += 64) {
            // 将 L[j + l] 和 L[j + l + 32] 的值合并为一个字节存入 q[l] 中
            for (int l = 0; l < 32; ++l) q[l] = L[j + l] | (L[j + l + 32] << 4);
            // q 指针向后移动 32 个字节
            q += 32;
        }
        // x 指针向后移动 QK_K 个元素
        x += QK_K;
    }
}

// 对于给定的 block_q4_K 类型指针 x，将其量化为 float 类型存入 y 中，k 为长度
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k) {
    // 断言 k 能被 QK_K 整除
    assert(k % QK_K == 0);
    // 计算 nb 的值
    const int nb = k / QK_K;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {

        // 获取指向 x[i].qs 的指针
        const uint8_t * q = x[i].qs;

        // 如果 QK_K 等于 256
#if QK_K == 256

        // 将 x[i].d 转换为 float 类型的数据
        const float d   = GGML_FP16_TO_FP32(x[i].d);
        // 将 x[i].dmin 转换为 float 类型的数据
        const float min = GGML_FP16_TO_FP32(x[i].dmin);

        // 初始化 is 为 0，声明 sc 和 m 为 uint8_t 类型
        int is = 0;
        uint8_t sc, m;
        // 循环遍历 QK_K 次，每次增加 64
        for (int j = 0; j < QK_K; j += 64) {
            // 获取 x[i].scales 中的数据，存入 sc 和 m 中
            get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
            // 计算 d1 和 m1 的值
            const float d1 = d * sc; const float m1 = min * m;
            // 获取 x[i].scales 中的数据，存入 sc 和 m 中
            get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
            // 计算 d2 和 m2 的值
            const float d2 = d * sc; const float m2 = min * m;
            // 循环遍历 32 次，每次将 d1 乘以 (q[l] & 0xF) 减去 m1 的结果存入 y 中
            for (int l = 0; l < 32; ++l) *y++ = d1 * (q[l] & 0xF) - m1;
            // 循环遍历 32 次，每次将 d2 乘以 (q[l] >> 4) 减去 m2 的结果存入 y 中
            for (int l = 0; l < 32; ++l) *y++ = d2 * (q[l]  >> 4) - m2;
            // 指针 q 向后移动 32 位，is 增加 2
            q += 32; is += 2;
        }
// 如果条件不成立，则执行以下代码
#else
        // 将x[i].d[0]转换为32位浮点数
        const float dall = GGML_FP16_TO_FP32(x[i].d[0]);
        // 将x[i].d[1]转换为32位浮点数
        const float mall = GGML_FP16_TO_FP32(x[i].d[1]);
        // 计算dall与mall乘以x[i].scales[0]的低4位和高4位的结果
        const float d1 = dall * (x[i].scales[0] & 0xF), m1 = mall * (x[i].scales[0] >> 4);
        // 计算dall与mall乘以x[i].scales[1]的低4位和高4位的结果
        const float d2 = dall * (x[i].scales[1] & 0xF), m2 = mall * (x[i].scales[1] >> 4);
        // 循环遍历32次
        for (int l = 0; l < 32; ++l) {
            // 计算y[l+ 0]的值
            y[l+ 0] = d1 * (q[l] & 0xF) - m1;
            // 计算y[l+32]的值
            y[l+32] = d2 * (q[l] >>  4) - m2;
        }
        // y指针向后移动QK_K个位置
        y += QK_K;
#endif
    }
}

// 将输入的浮点数数组x进行量化，结果存储在vy中，k为数组长度
void quantize_row_q4_K(const float * restrict x, void * restrict vy, int k) {
    // 断言k能被QK_K整除
    assert(k % QK_K == 0);
    // 将vy强制转换为block_q4_K类型的指针
    block_q4_K * restrict y = vy;
    // 调用quantize_row_q4_K_reference函数进行量化
    quantize_row_q4_K_reference(x, y, k);
}
// 用于将输入的浮点数组 src 进行量化，并将结果存储到目标数组 dst 中，同时计算直方图
size_t ggml_quantize_q4_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 忽略 hist 参数，待以后收集直方图
    (void)hist; // TODO: collect histograms

    // 对输入数组进行分块量化
    for (int j = 0; j < n; j += k) {
        // 将目标数组转换为 block_q4_K 类型，并进行量化
        block_q4_K * restrict y = (block_q4_K *)dst + j/QK_K;
        quantize_row_q4_K_reference(src + j, y, k);
    }
    // 返回量化后的数据占用的字节数
    return (n/QK_K*sizeof(block_q4_K));
}

// ====================== 5-bit (de)-quantization

// 将输入的浮点数组 x 进行量化，并将结果存储到目标数组 y 中
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    // 如果 QK_K 等于 256，则创建一个长度为 256 的无符号字符数组 L
    #if QK_K == 256
    uint8_t L[QK_K];
    // 定义存储最小值的数组，数组大小为QK_K/32
    float mins[QK_K/32];
    // 定义存储比例的数组，数组大小为QK_K/32
    float scales[QK_K/32];
    // 定义存储权重的数组，数组大小为32
    float weights[32];
    // 定义存储Laux的数组，数组大小为32
    uint8_t Laux[32];
#else
    // 如果QK_K不等于256，则定义存储L的数组，数组大小为QK_K
    int8_t L[QK_K];
    // 定义存储比例的数组，数组大小为QK_K/16
    float scales[QK_K/16];
#endif

    // 循环遍历nb次
    for (int i = 0; i < nb; i++) {

#if QK_K == 256

        // 定义最大比例为0
        float max_scale = 0; // as we are deducting the min, scales are always positive
        // 定义最大最小值为0
        float max_min = 0;
        // 循环遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算x的平方和
            float sum_x2 = 0;
            // 循环遍历32次
            for (int l = 0; l < 32; ++l) sum_x2 += x[32*j + l] * x[32*j + l];
            // 计算x的平均值
            float av_x = sqrtf(sum_x2/32);
        // 遍历循环，计算权重数组中的值
        for (int l = 0; l < 32; ++l) weights[l] = av_x + fabsf(x[32*j + l]);
        // 使用权重数组和其他参数计算得到scales[j]的值
        scales[j] = make_qkx2_quants(32, 31, x + 32*j, weights, L + 32*j, &mins[j], Laux, -0.5f, 0.1f, 15, false);
        // 获取scales[j]的值
        float scale = scales[j];
        // 如果scales[j]的值大于max_scale，则更新max_scale
        if (scale > max_scale) {
            max_scale = scale;
        }
        // 获取mins[j]的值
        float min = mins[j];
        // 如果mins[j]的值大于max_min，则更新max_min
        if (min > max_min) {
            max_min = min;
        }

        // 计算inv_scale和inv_min的值
        float inv_scale = max_scale > 0 ? 63.f/max_scale : 0.f;
        float inv_min   = max_min   > 0 ? 63.f/max_min   : 0.f;
        
        // 遍历循环，计算ls和lm的值
        for (int j = 0; j < QK_K/32; ++j) {
            uint8_t ls = nearest_int(inv_scale*scales[j]);
            uint8_t lm = nearest_int(inv_min*mins[j]);
            // 将ls和lm的值限制在0到63之间
            ls = MIN(63, ls);
            lm = MIN(63, lm);
            // 如果j小于4，则执行以下操作
            if (j < 4) {
        // 如果条件成立，将 ls 赋值给 y[i].scales[j]，将 lm 赋值给 y[i].scales[j+4]
        y[i].scales[j] = ls;
        y[i].scales[j+4] = lm;
        // 如果条件不成立
        } else {
            // 将 ls 和 lm 的低 4 位分别赋值给 y[i].scales[j+4]
            y[i].scales[j+4] = (ls & 0xF) | ((lm & 0xF) << 4);
            // 将 ls 右移 4 位后的值左移 6 位并赋值给 y[i].scales[j-4]
            y[i].scales[j-4] |= ((ls >> 4) << 6);
            // 将 lm 右移 4 位后的值左移 6 位并赋值给 y[i].scales[j-0]
            y[i].scales[j-0] |= ((lm >> 4) << 6);
        }
    }
    // 将 max_scale/63.f 转换为 FP16 格式并赋值给 y[i].d
    y[i].d = GGML_FP32_TO_FP16(max_scale/63.f);
    // 将 max_min/63.f 转换为 FP16 格式并赋值给 y[i].dmin
    y[i].dmin = GGML_FP32_TO_FP16(max_min/63.f);

    // 声明 uint8_t 类型的变量 sc 和 m
    uint8_t sc, m;
    // 循环遍历 QK_K/32 次
    for (int j = 0; j < QK_K/32; ++j) {
        // 调用 get_scale_min_k4 函数，传入参数 j, y[i].scales, &sc, &m
        get_scale_min_k4(j, y[i].scales, &sc, &m);
        // 将 GGML_FP16_TO_FP32(y[i].d) 乘以 sc 的结果赋值给 d
        const float d = GGML_FP16_TO_FP32(y[i].d) * sc;
        // 如果 d 为 0，则跳过本次循环
        if (!d) continue;
        // 将 GGML_FP16_TO_FP32(y[i].dmin) 乘以 m 的结果赋值给 dm
        const float dm = GGML_FP16_TO_FP32(y[i].dmin) * m;
        // 循环遍历 32 次
        for (int ii = 0; ii < 32; ++ii) {
            // 将 (x[32*j + ii] + dm)/d 最接近的整数赋值给 l
            int l = nearest_int((x[32*j + ii] + dm)/d);
            // 将 l 限制在 0 到 31 之间的范围内并赋值给 l
            l = MAX(0, MIN(31, l));
        // 将计算得到的 L 数组中的值存入对应的位置
        L[32*j + ii] = l;
    }
}

// 为当前处理的 y[i] 分配内存空间
uint8_t * restrict qh = y[i].qh;
uint8_t * restrict ql = y[i].qs;
// 将 qh 数组清零
memset(qh, 0, QK_K/8);

// 初始化 m1 和 m2 的值
uint8_t m1 = 1, m2 = 2;
// 遍历 QK_K，每次处理 64 个元素
for (int n = 0; n < QK_K; n += 64) {
    for (int j = 0; j < 32; ++j) {
        // 获取 L 数组中的值
        int l1 = L[n + j];
        // 如果值大于 15，则减去 16 并将对应的 qh[j] 置位
        if (l1 > 15) {
            l1 -= 16; qh[j] |= m1;
        }
        // 获取 L 数组中的值
        int l2 = L[n + j + 32];
        // 如果值大于 15，则减去 16 并将对应的 qh[j] 置位
        if (l2 > 15) {
            l2 -= 16; qh[j] |= m2;
        }
        // 将 l1 和 l2 组合成 ql[j] 的值
        ql[j] = l1 | (l2 << 4);
        }
        // 左移 m1 和 m2 两位
        m1 <<= 2; m2 <<= 2;
        // ql 增加 32
        ql += 32;
    }
#else
    // 定义最大比例尺和最大绝对值比例尺
    float max_scale = 0, amax = 0;
    // 遍历 QK_K/16 次
    for (int j = 0; j < QK_K/16; ++j) {
        // 计算比例尺并存储到 scales 数组中
        scales[j] = make_qx_quants(16, 16, x + 16*j, L + 16*j, 1);
        // 计算绝对值比例尺
        float abs_scale = fabsf(scales[j]);
        // 如果绝对值比例尺大于最大绝对值比例尺，则更新最大绝对值比例尺和最大比例尺
        if (abs_scale > amax) {
            amax = abs_scale;
            max_scale = scales[j];
        }
    }

    // 计算输入比例尺
    float iscale = -128.f/max_scale;
    // 遍历 QK_K/16 次
    for (int j = 0; j < QK_K/16; ++j) {
        // 计算最接近的整数比例尺，并存储到结果数组中
        int l = nearest_int(iscale*scales[j]);
        y[i].scales[j] = MAX(-128, MIN(127, l));
    }
        # 将 y[i].d 转换为 FP16 格式
        y[i].d = GGML_FP32_TO_FP16(1/iscale);

        # 遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            # 计算 d 的值
            const float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            # 如果 d 为 0，则跳过当前循环
            if (!d) continue;
            # 遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                # 计算 l 的值
                int l = nearest_int(x[16*j + ii]/d);
                # 将 l 限制在 -16 和 15 之间
                l = MAX(-16, MIN(15, l));
                # 将 l+16 存入 L 数组中
                L[16*j + ii] = l + 16;
            }
        }

        # 限制指针 qh 和 ql，将其初始化为 0
        uint8_t * restrict qh = y[i].qh;
        uint8_t * restrict ql = y[i].qs;
        memset(qh, 0, QK_K/8);

        # 遍历 32 次
        for (int j = 0; j < 32; ++j) {
            # 计算 jm 和 is 的值
            int jm = j%8;
            int is = j/8;
            # 获取 L[j] 的值
            int l1 = L[j];
            // 如果l1大于15，减去16，并将qh[jm]的第is位设为1
            if (l1 > 15) {
                l1 -= 16; qh[jm] |= (1 << is);
            }
            // 获取L[j + 32]的值
            int l2 = L[j + 32];
            // 如果l2大于15，减去16，并将qh[jm]的第(4 + is)位设为1
            if (l2 > 15) {
                l2 -= 16; qh[jm] |= (1 << (4 + is));
            }
            // 将l1和l2合并成一个8位的值，存储在ql[j]中
            ql[j] = l1 | (l2 << 4);
        }
#endif

        // x增加QK_K
        x += QK_K;

    }
}

// 对输入的x进行反量化，存储在y中，k表示x的长度
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k) {
    // 断言k是QK_K的倍数
    assert(k % QK_K == 0);
    // 计算nb的值，表示x的长度除以QK_K的结果
    const int nb = k / QK_K;
    # 遍历 nb 次，nb 为某个变量的值
    for (int i = 0; i < nb; i++) {

        # 获取 x[i] 结构体中的 qs 和 qh 字节流
        const uint8_t * ql = x[i].qs;
        const uint8_t * qh = x[i].qh;

        # 如果 QK_K 等于 256
        # 将 x[i].d 和 x[i].dmin 从 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);
        const float min = GGML_FP16_TO_FP32(x[i].dmin);

        # 初始化一些变量
        int is = 0;
        uint8_t sc, m;
        uint8_t u1 = 1, u2 = 2;

        # 遍历 QK_K 次，QK_K 为某个变量的值，每次增加 64
        for (int j = 0; j < QK_K; j += 64) {
            # 获取 x[i].scales 中的数据，并计算得到 sc 和 m
            get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
            const float d1 = d * sc; const float m1 = min * m;
            get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
            const float d2 = d * sc; const float m2 = min * m;
            # 遍历 32 次，每次将计算结果存入 y 中
            for (int l = 0; l < 32; ++l) *y++ = d1 * ((ql[l] & 0xF) + (qh[l] & u1 ? 16 : 0)) - m1;
            for (int l = 0; l < 32; ++l) *y++ = d2 * ((ql[l]  >> 4) + (qh[l] & u2 ? 16 : 0)) - m2;
// 如果定义了宏，则执行以下代码块
#if defined(XXX)
        // ql 和 is 分别增加 32 和 2
        ql += 32; is += 2;
        // u1 和 u2 左移 2 位
        u1 <<= 2; u2 <<= 2;
    }
// 如果没有定义宏，则执行以下代码块
#else
        // 将 x[i].d 从 FP16 转换为 FP32 存储在 d 中
        float d = GGML_FP16_TO_FP32(x[i].d);
        // 定义指向 x[i].scales 的指针 s
        const int8_t * restrict s = x[i].scales;
        // 循环遍历 l 从 0 到 7
        for (int l = 0; l < 8; ++l) {
            // 根据一系列计算公式计算 y 的值
            y[l+ 0] = d * s[0] * ((ql[l+ 0] & 0xF) - (qh[l] & 0x01 ? 0 : 16));
            // ...
            // 重复上述计算过程，直到 l+56
        }
        // 将 y 指针向后移动 QK_K 位
        y += QK_K;
#endif
    }
}
// 以 QK_K 为步长对行进行量化
void quantize_row_q5_K(const float * restrict x, void * restrict vy, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 将 vy 强制转换为 block_q5_K 类型的指针
    block_q5_K * restrict y = vy;
    // 调用 quantize_row_q5_K_reference 函数对行进行量化
    quantize_row_q5_K_reference(x, y, k);
}

// 对输入的浮点数数组进行量化，并将结果存储到目标数组中
size_t ggml_quantize_q5_K(const float * restrict src, void * restrict dst, int n, int k, int64_t * restrict hist) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 忽略 hist 参数，待实现：收集直方图
    (void)hist; // TODO: collect histograms

    // 对每个长度为 k 的子数组进行量化
    for (int j = 0; j < n; j += k) {
        // 将目标数组 dst 强制转换为 block_q5_K 类型的指针，并偏移 j/QK_K 个单位
        block_q5_K * restrict y = (block_q5_K *)dst + j/QK_K;
        // 调用 quantize_row_q5_K_reference 函数对行进行量化
        quantize_row_q5_K_reference(src + j, y, k);
    }
    // 返回量化后的目标数组的大小
    return (n/QK_K*sizeof(block_q5_K));
}

// ====================== 6-bit (de)-quantization
// 对输入的一维数组进行量化，结果存储到输出的结构体数组中，每个结构体包含 QK_K 个元素
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k) {
    // 确保 k 是 QK_K 的整数倍
    assert(k % QK_K == 0);
    // 计算结构体数组的个数
    const int nb = k / QK_K;

    // 定义长度为 QK_K 的 int8_t 数组和长度为 QK_K/16 的 float 数组
    int8_t L[QK_K];
    float   scales[QK_K/16];

    // 遍历结构体数组
    for (int i = 0; i < nb; i++) {

        // 初始化最大比例和最大绝对值比例
        float max_scale = 0;
        float max_abs_scale = 0;

        // 遍历每个结构体中的 16 个元素
        for (int ib = 0; ib < QK_K/16; ++ib) {

            // 对输入数组的一部分进行量化，结果存储到 L 数组中，并返回比例
            const float scale = make_qx_quants(16, 32, x + 16*ib, L + 16*ib, 1);
            scales[ib] = scale;

            // 计算比例的绝对值
            const float abs_scale = fabsf(scale);
            // 更新最大绝对值比例
            if (abs_scale > max_abs_scale) {
                max_abs_scale = abs_scale;
        // 如果最大绝对值缩放因子为0，则将y[i]的内容全部置为0，并跳过当前循环
        if (!max_abs_scale) {
            memset(&y[i], 0, sizeof(block_q6_K));
            y[i].d = GGML_FP32_TO_FP16(0.f);
            x += QK_K;
            continue;
        }

        // 计算缩放因子iscale
        float iscale = -128.f/max_scale;
        // 将y[i]的d成员设置为1/iscale
        y[i].d = GGML_FP32_TO_FP16(1/iscale);
        // 遍历scales数组，计算y[i]的scales数组
        for (int ib = 0; ib < QK_K/16; ++ib) {
            y[i].scales[ib] = MIN(127, nearest_int(iscale*scales[ib]));
        }

        // 遍历QK_K/16次，计算d的值
        for (int j = 0; j < QK_K/16; ++j) {
            float d = GGML_FP16_TO_FP32(y[i].d) * y[i].scales[j];
            // 如果 d 为假值，则跳过当前循环
            if (!d) {
                continue;
            }
            // 遍历 16 次
            for (int ii = 0; ii < 16; ++ii) {
                // 计算 x[16*j + ii] 除以 d 的最接近整数值
                int l = nearest_int(x[16*j + ii]/d);
                // 将 l 限制在 -32 到 31 之间
                l = MAX(-32, MIN(31, l));
                // 将 l 加上 32 后存入 L 数组中
                L[16*j + ii] = l + 32;
            }
        }

        // 限制 ql 和 qh 指针指向的内存区域
        uint8_t * restrict ql = y[i].ql;
        uint8_t * restrict qh = y[i].qh;
        // 当 QK_K 等于 256 时执行以下循环
#if QK_K == 256
        // 每次循环增加 128
        for (int j = 0; j < QK_K; j += 128) {
            // 遍历 32 次
            for (int l = 0; l < 32; ++l) {
                // 从 L 数组中取出特定位置的值，并进行位运算后存入 ql 数组
                const uint8_t q1 = L[j + l +  0] & 0xF;
                const uint8_t q2 = L[j + l + 32] & 0xF;
                const uint8_t q3 = L[j + l + 64] & 0xF;
                const uint8_t q4 = L[j + l + 96] & 0xF;
                ql[l+ 0] = q1 | (q3 << 4);
#if defined(QUARTERROUND)
        // 如果定义了QUARTERROUND，则执行以下代码块
        for (int l = 0; l < 32; ++l) {
            // 从L数组中取出对应位置的值，并进行位运算和赋值操作
            const uint8_t q1 = L[l +  0] & 0xF;
            const uint8_t q2 = L[l + 32] & 0xF;
            ql[l] = q1 | (q2 << 4);
        }
        // 遍历L数组的前16个元素，进行位运算和赋值操作
        for (int l = 0; l < 16; ++l) {
            qh[l] = (L[l] >> 4) | ((L[l + 16] >> 4) << 2) | ((L[l + 32] >> 4) << 4) | ((L[l + 48] >> 4) << 6);
        }
#else
        // 如果没有定义QUARTERROUND，则执行以下代码块
        for (int l = 0; l < 32; ++l) {
            // 对L数组中的值进行位运算和赋值操作
            ql[l+32] = q2 | (q4 << 4);
            // 对L数组中的值进行位运算和赋值操作
            qh[l] = (L[j + l] >> 4) | ((L[j + l + 32] >> 4) << 2) | ((L[j + l + 64] >> 4) << 4) | ((L[j + l + 96] >> 4) << 6);
        }
#endif
        // 对x进行加上QK_K的操作
        x += QK_K;
    }
}

void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k) {
    // 确保 k 是 QK_K 的倍数
    assert(k % QK_K == 0);
    // 计算块的数量
    const int nb = k / QK_K;

    for (int i = 0; i < nb; i++) {

        // 将 FP16 转换为 FP32
        const float d = GGML_FP16_TO_FP32(x[i].d);

        // 限定指针的类型和访问权限
        const uint8_t * restrict ql = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const int8_t  * restrict sc = x[i].scales;

#if QK_K == 256
        // 循环处理每个块
        for (int n = 0; n < QK_K; n += 128) {
            // 循环处理每个元素
            for (int l = 0; l < 32; ++l) {
                // 计算索引
                int is = l/16;
                // 计算量化值
                const int8_t q1 = (int8_t)((ql[l +  0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
                const int8_t q2 = (int8_t)((ql[l + 32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
// 对于每个 l，计算 q1, q2, q3, q4，并根据它们计算 y[l]
const int8_t q1 = (int8_t)((ql[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
const int8_t q2 = (int8_t)((ql[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
const int8_t q3 = (int8_t)((ql[l+ 0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32;
const int8_t q4 = (int8_t)((ql[l+16]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32;
y[l+ 0] = d * sc[0] * q1;
y[l+16] = d * sc[1] * q2;
// 对 l 进行迭代
            // 将计算结果赋值给数组中的元素
            y[l+32] = d * sc[2] * q3;
            y[l+48] = d * sc[3] * q4;
        }
        // 指针移动到下一个64个元素的位置
        y  += 64;
#endif

    }
}

// 对输入数组进行量化处理，结果存储在vy中
void quantize_row_q6_K(const float * restrict x, void * restrict vy, int k) {
    // 确保k是QK_K的倍数
    assert(k % QK_K == 0);
    // 将vy转换为block_q6_K类型的指针
    block_q6_K * restrict y = vy;
    // 调用quantize_row_q6_K_reference函数进行量化处理
    quantize_row_q6_K_reference(x, y, k);
}

// 对输入数组进行量化处理，并将结果存储在dst中，同时返回处理后的数据大小
size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保k是QK_K的倍数
    assert(k % QK_K == 0);
    // 忽略hist参数，待实现直方图收集功能
    (void)hist; // TODO: collect histograms

    // 对输入数组进行循环处理
    for (int j = 0; j < n; j += k) {
// 定义指向目标内存块的指针，每次移动一个块的大小
block_q6_K * restrict y = (block_q6_K *)dst + j/QK_K;
// 对输入数据进行量化，将结果存储到目标内存块中
quantize_row_q6_K_reference(src + j, y, k);
// 返回量化后的数据大小
}
return (n/QK_K*sizeof(block_q6_K));
}

//===================================== Q8_K ==============================================

// 对输入数据进行量化，将结果存储到目标内存块中
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k) {
// 确保 k 是 QK_K 的倍数
assert(k % QK_K == 0);
// 计算块的数量
const int nb = k / QK_K;

for (int i = 0; i < nb; i++) {

    float max = 0;
    float amax = 0;
    for (int j = 0; j < QK_K; ++j) {
        // 计算输入数据的绝对值
        float ax = fabsf(x[j]);
        // 如果绝对值大于最大值，则更新最大值和对应的原始值
        if (ax > amax) {
            amax = ax; max = x[j];
        }
        }
        // 如果最大值为0，则将y[i].d设为0，将y[i].qs数组清零，然后继续下一轮循环
        if (!amax) {
            y[i].d = 0;
            memset(y[i].qs, 0, QK_K);
            x += QK_K;
            continue;
        }
        // 计算缩放因子iscale
        const float iscale = -128.f/max;
        // 对x中的每个元素进行量化，并存储到y[i].qs数组中
        for (int j = 0; j < QK_K; ++j) {
            int v = nearest_int(iscale*x[j]);
            y[i].qs[j] = MIN(127, v);
        }
        // 计算y[i].qs数组的每16个元素的和，并存储到y[i].bsums数组中
        for (int j = 0; j < QK_K/16; ++j) {
            int sum = 0;
            for (int ii = 0; ii < 16; ++ii) {
                sum += y[i].qs[j*16 + ii];
            }
            y[i].bsums[j] = sum;
        }
// 将y[i].d的值设置为1/iscale
// 将x的值增加QK_K
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k) {
    // 确保k能被QK_K整除
    assert(k % QK_K == 0);
    // 计算nb的值，即k除以QK_K的结果
    const int nb = k / QK_K;

    // 遍历nb次
    for (int i = 0; i < nb; i++) {
        // 遍历QK_K次
        for (int j = 0; j < QK_K; ++j) {
            // 将y的值设置为x[i].d乘以x[i].qs[j]
            *y++ = x[i].d * x[i].qs[j];
        }
    }
}

// 将float类型的数组x量化为void类型的数组y，长度为k
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k) {
    // 调用quantize_row_q8_K_reference函数进行量化
    quantize_row_q8_K_reference(x, y, k);
}
//===================================== Dot ptoducts =================================
// 这部分代码是关于点积计算的

//
// Helper functions
//
// 辅助函数

#if __AVX__ || __AVX2__ || __AVX512F__
// 如果支持 AVX、AVX2 或 AVX512F 指令集

// shuffles to pick the required scales in dot products
// 用于在点积中选择所需的缩放值的洗牌操作
static inline __m256i get_scale_shuffle_q3k(int i) {
    // 从预定义的数组中加载洗牌索引
    static const uint8_t k_shuffle[128] = {
         // 洗牌索引数组
    };
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
}
static inline __m256i get_scale_shuffle_k4(int i) {
    // 从预定义的数组中加载洗牌索引
    static const uint8_t k_shuffle[256] = {
         // 洗牌索引数组
    };
// 创建一个名为 get_scale_shuffle 的函数，参数为整数 i，返回类型为 __m128i
static inline __m128i get_scale_shuffle(int i) {
    // 创建一个名为 k_shuffle 的静态常量数组，包含128个元素
    static const uint8_t k_shuffle[128] = {
         // 数组元素的值
         // ...
    };
    // 返回 k_shuffle 数组中索引为 i 的元素，作为 __m128i 类型返回
    return _mm256_loadu_si256((const __m256i*)k_shuffle + i);
}
    };
    return _mm_loadu_si128((const __m128i*)k_shuffle + i);
}
#endif

void ggml_axpy_q4_0_q8_0(const int n, const void * restrict vx, const void * restrict vy, const void * restrict vz, int8_t alpha, ggml_fp16_t scale) {
    const int qk = QK8_0;  // 定义常量 qk 为 QK8_0
    const int nb = n / qk;  // 计算 nb 为 n 除以 qk
    assert(n % qk == 0);  // 断言 n 能被 qk 整除
    assert(nb % 2 == 0);  // 断言 nb 是偶数

    const block_q4_0 * restrict x = vx;  // 定义指针 x 指向 vx

#if defined(__AVX2__)
    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    __m256i alpha_v = _mm256_set1_epi16((short)alpha);  // 将 alpha 转换为 16 位整数并复制到 alpha_v
    // 主循环
    for (int i = 0; i < nb; ++i) {
        /* 计算块的组合比例 */
        // 使用 x[i].d 和 scale 计算得到一个包含 8 个单精度浮点数的 AVX 寄存器
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(scale) );
        // 将 x[i].qs 中的每个字节转换为整数，存储到 bx 中
        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 将 bx 中的每个字节的值减去 8
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );
        // 从 bx 中提取出低 128 位的数据
        __m128i m_a = _mm256_extracti128_si256(bx, 0);
        // 将 m_a 中的每个字节转换为短整数
        __m256i m_x = _mm256_cvtepi8_epi16(m_a); //16 elements
        // 将 m_x 中的每个短整数乘以 alpha_v 中的对应元素
        m_x = _mm256_mullo_epi16(m_x, alpha_v);
        // 从 m_x 中提取出低 128 位的数据
        __m128i x_0 = _mm256_extracti128_si256(m_x, 0);
        // 将 x_0 中的每个短整数转换为整数
        __m256i x0_32 = _mm256_cvtepi16_epi32(x_0);
        // 将 x0_32 中的每个整数转换为单精度浮点数
        __m256 fx0 = _mm256_cvtepi32_ps(x0_32);
        // 将 fx0 中的每个单精度浮点数乘以 d 中的对应元素
        fx0 = _mm256_mul_ps(fx0, d);

        // 从 vy 中加载 8 个单精度浮点数到 by 中
        __m256 by = _mm256_loadu_ps((const __m256 *)(vy+i*128));

        // 将 by 中的每个单精度浮点数加上 fx0 中的对应元素
        by = _mm256_add_ps(by, fx0);
        // 将 by 中的 8 个单精度浮点数存储到 vz 中
        _mm256_storeu_ps((__m256*)(vz + i*128), by);
// 第二阶段
// 从 m_x 中提取高 128 位的数据
x_0 = _mm256_extracti128_si256(m_x, 1);
// 将 x_0 中的 16 位整数扩展为 32 位整数
x0_32 = _mm256_cvtepi16_epi32(x_0);
// 将 x0_32 中的 32 位整数转换为单精度浮点数
fx0 = _mm256_cvtepi32_ps(x0_32);
// 将 fx0 乘以 d
fx0 = _mm256_mul_ps(fx0, d);
// 从 vy 数组中加载 32 个单精度浮点数到 by 中
by = _mm256_loadu_ps((const __m256 *)(vy+i*128+32));
// 将 fx0 和 by 相加，并将结果存储到 vz 数组中
by = _mm256_add_ps(by, fx0);
_mm256_storeu_ps((__m256*)(vz + i*128+32), by);

// 第三阶段
// 从 bx 中提取高 128 位的数据
m_a = _mm256_extracti128_si256(bx, 1);
// 将 m_a 中的 8 位整数扩展为 16 位整数
m_x = _mm256_cvtepi8_epi16(m_a);
// 将 m_x 中的 16 位整数乘以 alpha_v
m_x = _mm256_mullo_epi16(m_x, alpha_v);
// 从 m_x 中提取低 128 位的数据
x_0 = _mm256_extracti128_si256(m_x, 0);
// 将 x_0 中的 16 位整数扩展为 32 位整数
x0_32 = _mm256_cvtepi16_epi32(x_0);
// 将 x0_32 中的 32 位整数转换为单精度浮点数
fx0 = _mm256_cvtepi32_ps(x0_32);
// 将 fx0 乘以 d
fx0 = _mm256_mul_ps(fx0, d);
// 从 vy 数组中加载 64 个单精度浮点数到 by 中
by = _mm256_loadu_ps((const __m256 *)(vy+i*128+64));
        // 将两个__m256类型的寄存器中的元素逐个相加，并将结果存储到by中
        by = _mm256_add_ps(by, fx0);
        // 将by中的数据存储到vz数组中
        _mm256_storeu_ps((__m256*)(vz + i*128+64), by);

        // 第四阶段
        // 从m_x中提取第二个128位的数据
        x_0 = _mm256_extracti128_si256(m_x, 1);
        // 将x_0中的16位整数转换成32位整数
        x0_32 = _mm256_cvtepi16_epi32(x_0);
        // 将x0_32中的32位整数转换成单精度浮点数
        fx0 = _mm256_cvtepi32_ps(x0_32);
        // 将fx0中的数据乘以d
        fx0 = _mm256_mul_ps(fx0, d);
        // 从vy数组中加载数据到by中
        by = _mm256_loadu_ps((const __m256 *)(vy+i*128+96));
        // 将by中的数据存储到vz数组中
        by = _mm256_add_ps(by, fx0);
        _mm256_storeu_ps((__m256*)(vz + i*128+96), by);

    }
#else
    // 将vz强制转换为float指针
    float *res = (float *)vz;
    // 将scale从半精度浮点数转换为单精度浮点数
    float scale_fp32 = GGML_FP16_TO_FP32(scale);
    // 遍历nb次
    for (int i = 0; i < nb; i++) {
        // 将x[i].d从半精度浮点数转换为单精度浮点数，并乘以scale_fp32
        float result_scale = GGML_FP16_TO_FP32(x[i].d) * scale_fp32;
        // 计算偏移量
        int offset = i * QK4_0;
    // 循环遍历数组，每次处理两个元素
    for (int j = 0; j < qk/2; ++j) {
        // 从x[i].qs[j]中取出低4位，减去8，得到v0
        const int v0 = (x[i].qs[j] & 0x0F) - 8;
        // 从x[i].qs[j]中取出高4位，减去8，得到v1
        const int v1 = (x[i].qs[j] >>   4) - 8;
        // 将v0乘以alpha的整数部分，再乘以result_scale，加到res[offset + j]上
        res[offset + j] = res[offset + j] + ((float)(v0 * (int)alpha) * result_scale);
        // 将v1乘以alpha的整数部分，再乘以result_scale，加到res[offset + j + qk/2]上
        res[offset + j + qk/2] = res[offset + j + qk/2] + ((float)(v1 * (int)alpha) * result_scale);
    }
    // 结束for循环
    }
// 结束#ifdef
}

// 定义一个函数ggml_vec_dot_q4_0_q8_0，参数为n, s, vx, vy
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量qk为QK8_0
    const int qk = QK8_0;
    // 定义常量nb为n除以qk的商
    const int nb = n / qk;

    // 断言n能被qk整除
    assert(n % qk == 0);

    // 定义指向block_q4_0类型的指针x，指向vx
    const block_q4_0 * restrict x = vx;
    // 定义指向block_q8_0类型的指针y，指向vy
    const block_q8_0 * restrict y = vy;
#if defined(__ARM_NEON)
    // 创建一个包含4个32位浮点数的向量，并初始化为0.0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 断言nb是偶数，如果不是偶数则需要处理
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环遍历nb，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 定义指向x和y数组的指针，每次增加1
        const block_q4_0 * restrict x0 = &x[i + 0];
        const block_q4_0 * restrict x1 = &x[i + 1];
        const block_q8_0 * restrict y0 = &y[i + 0];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个包含16个8位无符号整数的向量，并初始化为0x0F
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
        // 创建一个包含16个8位有符号整数的向量，并初始化为0x8
        const int8x16_t  s8b = vdupq_n_s8(0x8);

        // 从x0和x1指向的数组中加载16个8位无符号整数到向量v0_0和v0_1中
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        // 将v0_0中的每个4位数据扩展为8位数据
        const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        // 将v0_0的每个字节右移4位，然后重新解释为有符号8位整数
        const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        // 将v0_1和m4b按位与，然后重新解释为有符号8位整数
        const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        // 将v0_1的每个字节右移4位，然后重新解释为有符号8位整数
        const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        // 减去8
        const int8x16_t v0_0ls = vsubq_s8(v0_0l, s8b);
        const int8x16_t v0_0hs = vsubq_s8(v0_0h, s8b);
        const int8x16_t v0_1ls = vsubq_s8(v0_1l, s8b);
        const int8x16_t v0_1hs = vsubq_s8(v0_1h, s8b);

        // 加载y
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

#if defined(__ARM_FEATURE_DOTPROD)
        // 将v0_0ls和v1_0l的每个元素相乘，然后相加得到int32x4_t类型的结果
        const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0ls, v1_0l), v0_0hs, v1_0h);
        // 将v0_1ls和v1_1l的每个元素相乘，然后相加得到int32x4_t类型的结果
        const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1ls, v1_1l), v0_1hs, v1_1h);
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d);
        # 使用 NEON 指令 vmlaq_n_f32 对 sumv0 进行乘加操作，将结果存储到 sumv0 中

        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
        # 使用 NEON 指令 vmlaq_n_f32 对 sumv1 进行乘加操作，将结果存储到 sumv1 中

#else
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0ls), vget_low_s8 (v1_0l));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行低位元素相乘，将结果存储到 pl0l 中
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0ls), vget_high_s8(v1_0l));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行高位元素相乘，将结果存储到 pl0h 中
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hs), vget_low_s8 (v1_0h));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行低位元素相乘，将结果存储到 ph0l 中
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hs), vget_high_s8(v1_0h));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行高位元素相乘，将结果存储到 ph0h 中

        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1ls), vget_low_s8 (v1_1l));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行低位元素相乘，将结果存储到 pl1l 中
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1ls), vget_high_s8(v1_1l));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行高位元素相乘，将结果存储到 pl1h 中
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hs), vget_low_s8 (v1_1h));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行低位元素相乘，将结果存储到 ph1l 中
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hs), vget_high_s8(v1_1h));
        # 使用 NEON 指令 vmull_s8 对两个 int8x8_t 类型的向量进行高位元素相乘，将结果存储到 ph1h 中

        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        # 使用 NEON 指令 vaddq_s32 对两个 int16x8_t 类型的向量进行低位元素相加，并将结果存储到 pl0 中
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        # 使用 NEON 指令 vaddq_s32 对两个 int16x8_t 类型的向量进行低位元素相加，并将结果存储到 ph0 中
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        # 使用 NEON 指令 vaddq_s32 对两个 int16x8_t 类型的向量进行低位元素相加，并将结果存储到 pl1 中
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));
        # 使用 NEON 指令 vaddq_s32 对两个 int16x8_t 类型的向量进行低位元素相加，并将结果存储到 ph1 中

        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        # 使用 NEON 指令 vmlaq_n_f32 对 sumv0 进行乘加操作，将结果存储到 sumv0 中
    // 使用 NEON 指令计算两个向量的乘积，并将结果与常数相加
    sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d);
#endif
    }

    // 计算两个向量的和并返回结果
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__)
    // 使用 AVX2 指令集初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 将 nibbles 转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);

        // 现在我们有一个字节向量，范围在 [ 0 .. 15 ] 区间。将它们偏移为 [ -8 .. +7 ] 区间
        const __m256i off = _mm256_set1_epi8( 8 );
        bx = _mm256_sub_epi8( bx, off );
    // 从数组y[i].qs中加载256位整数到寄存器by中
    __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

    // 计算bx和by的乘积，并将结果存储在寄存器q中
    const __m256 q = mul_sum_i8_pairs_float(bx, by);

    /* 将q乘以比例并累加 */
    // 使用d乘以q，并将结果累加到acc中
    acc = _mm256_fmadd_ps( d, q, acc );
    }

    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 用零初始化累加器
    // 使用_mm256_setzero_ps()函数将acc初始化为全零的256位浮点数
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        // 将x[i].d和y[i].d转换为32位浮点数，然后相乘，并将结果存储在寄存器d中
        const __m256 d = _mm256_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        // 创建一个低位掩码
        // 使用_mm_set1_epi8(0xF)函数创建一个低位掩码，所有位都是0xF
        const __m128i lowMask = _mm_set1_epi8(0xF);
        // 创建一个偏移量
        // 使用_mm_set1_epi8(8)函数创建一个偏移量，所有位都是8
        const __m128i off = _mm_set1_epi8(8);
        // 从数组 x 中加载 128 位数据到临时变量 tmp
        const __m128i tmp = _mm_loadu_si128((const __m128i *)x[i].qs);

        // 对 tmp 和 lowMask 进行按位与操作
        __m128i bx = _mm_and_si128(lowMask, tmp);
        // 从数组 y 中加载 128 位数据到临时变量 by
        __m128i by = _mm_loadu_si128((const __m128i *)y[i].qs);
        // 对 bx 减去 off
        bx = _mm_sub_epi8(bx, off);
        // 对 bx 和 by 进行乘法和求和操作，得到 i32_0
        const __m128i i32_0 = mul_sum_i8_pairs(bx, by);

        // 对 tmp 右移 4 位，并与 lowMask 进行按位与操作
        bx = _mm_and_si128(lowMask, _mm_srli_epi64(tmp, 4));
        // 从数组 y 中加载 128 位数据到临时变量 by
        by = _mm_loadu_si128((const __m128i *)(y[i].qs + 16));
        // 对 bx 减去 off
        bx = _mm_sub_epi8(bx, off);
        // 对 bx 和 by 进行乘法和求和操作，得到 i32_1
        const __m128i i32_1 = mul_sum_i8_pairs(bx, by);

        // 将 int32_t 转换为 float
        __m256 p = _mm256_cvtepi32_ps(MM256_SET_M128I(i32_0, i32_1));

        // 应用比例，并累加
        acc = _mm256_add_ps(_mm256_mul_ps( d, p ), acc);
    }
    *s = hsum_float_8(acc);
#elif defined(__SSSE3__)
    // 如果定义了__SSSE3__，则执行以下代码

    // 设置常量
    const __m128i lowMask = _mm_set1_epi8(0xF);
    const __m128i off = _mm_set1_epi8(8);

    // 用零初始化累加器
    __m128 acc_0 = _mm_setzero_ps();
    __m128 acc_1 = _mm_setzero_ps();
    __m128 acc_2 = _mm_setzero_ps();
    __m128 acc_3 = _mm_setzero_ps();

    // 第一轮不累加
    {
        _mm_prefetch(&x[0] + sizeof(block_q4_0), _MM_HINT_T0);
        _mm_prefetch(&y[0] + sizeof(block_q8_0), _MM_HINT_T0);

        // 计算块0和1的组合比例
        const __m128 d_0_1 = _mm_set1_ps( GGML_FP16_TO_FP32(x[0].d) * GGML_FP16_TO_FP32(y[0].d) );
// 从数组x[0]中加载128位数据到临时寄存器tmp_0_1
const __m128i tmp_0_1 = _mm_loadu_si128((const __m128i *)x[0].qs);

// 对tmp_0_1和lowMask进行按位与操作，存储结果到bx_0
__m128i bx_0 = _mm_and_si128(lowMask, tmp_0_1);

// 从数组y[0]中加载128位数据到临时寄存器by_0
__m128i by_0 = _mm_loadu_si128((const __m128i *)y[0].qs);

// 对bx_0减去off，存储结果到bx_0
bx_0 = _mm_sub_epi8(bx_0, off);

// 调用mul_sum_i8_pairs函数，传入bx_0和by_0，存储结果到i32_0
const __m128i i32_0 = mul_sum_i8_pairs(bx_0, by_0);

// 对tmp_0_1右移4位，然后与lowMask进行按位与操作，存储结果到bx_1
__m128i bx_1 = _mm_and_si128(lowMask, _mm_srli_epi64(tmp_0_1, 4));

// 从数组y[0]中加载128位数据到临时寄存器by_1
__m128i by_1 = _mm_loadu_si128((const __m128i *)(y[0].qs + 16));

// 对bx_1减去off，存储结果到bx_1
bx_1 = _mm_sub_epi8(bx_1, off);

// 调用mul_sum_i8_pairs函数，传入bx_1和by_1，存储结果到i32_1
const __m128i i32_1 = mul_sum_i8_pairs(bx_1, by_1);

// 预取下一个数据块x[1]和y[1]到缓存中
_mm_prefetch(&x[1] + sizeof(block_q4_0), _MM_HINT_T0);
_mm_prefetch(&y[1] + sizeof(block_q8_0), _MM_HINT_T0);

// 计算数据块2和3的组合比例，存储结果到d_2_3
const __m128 d_2_3 = _mm_set1_ps( GGML_FP16_TO_FP32(x[1].d) * GGML_FP16_TO_FP32(y[1].d) );

// 从数组x[1]中加载128位数据到临时寄存器tmp_2_3
const __m128i tmp_2_3 = _mm_loadu_si128((const __m128i *)x[1].qs);
        # 使用低位掩码和临时变量进行位与操作
        __m128i bx_2 = _mm_and_si128(lowMask, tmp_2_3);
        # 加载 y[1].qs 中的数据到 bx_2
        __m128i by_2 = _mm_loadu_si128((const __m128i *)y[1].qs);
        # 从 bx_2 中减去 off
        bx_2 = _mm_sub_epi8(bx_2, off);
        # 计算 bx_2 和 by_2 的乘积和
        const __m128i i32_2 = mul_sum_i8_pairs(bx_2, by_2);

        # 使用低位掩码和右移 4 位的临时变量进行位与操作
        __m128i bx_3 = _mm_and_si128(lowMask, _mm_srli_epi64(tmp_2_3, 4));
        # 加载 y[1].qs + 16 中的数据到 bx_3
        __m128i by_3 = _mm_loadu_si128((const __m128i *)(y[1].qs + 16));
        # 从 bx_3 中减去 off
        bx_3 = _mm_sub_epi8(bx_3, off);
        # 计算 bx_3 和 by_3 的乘积和
        const __m128i i32_3 = mul_sum_i8_pairs(bx_3, by_3);

        // 将 int32_t 转换为 float
        __m128 p0 = _mm_cvtepi32_ps(i32_0);
        __m128 p1 = _mm_cvtepi32_ps(i32_1);
        __m128 p2 = _mm_cvtepi32_ps(i32_2);
        __m128 p3 = _mm_cvtepi32_ps(i32_3);

        // 应用比例
        acc_0 = _mm_mul_ps( d_0_1, p0 );
        acc_1 = _mm_mul_ps( d_0_1, p1 );
        acc_2 = _mm_mul_ps( d_2_3, p2 );
    // 计算acc_3
    acc_3 = _mm_mul_ps( d_2_3, p3 );
    }

    // 断言nb是偶数
    assert(nb % 2 == 0); // TODO: 处理奇数nb

    // 主循环
    for (int i = 2; i < nb; i+=2) {
        // 预取x[i]和y[i]的下一个块
        _mm_prefetch(&x[i] + sizeof(block_q4_0), _MM_HINT_T0);
        _mm_prefetch(&y[i] + sizeof(block_q8_0), _MM_HINT_T0);

        // 计算块0和1的组合比例
        const __m128 d_0_1 = _mm_set1_ps( GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d) );

        const __m128i tmp_0_1 = _mm_loadu_si128((const __m128i *)x[i].qs);

        __m128i bx_0 = _mm_and_si128(lowMask, tmp_0_1);
        __m128i by_0 = _mm_loadu_si128((const __m128i *)y[i].qs);
        bx_0 = _mm_sub_epi8(bx_0, off);
        const __m128i i32_0 = mul_sum_i8_pairs(bx_0, by_0);
# 使用低位掩码和右移操作，将tmp_0_1的值与lowMask进行按位与运算
__m128i bx_1 = _mm_and_si128(lowMask, _mm_srli_epi64(tmp_0_1, 4));
# 从y[i].qs + 16地址加载一个128位的值到by_1
__m128i by_1 = _mm_loadu_si128((const __m128i *)(y[i].qs + 16));
# 将bx_1减去off
bx_1 = _mm_sub_epi8(bx_1, off);
# 调用mul_sum_i8_pairs函数，传入bx_1和by_1，将结果存储在i32_1中
const __m128i i32_1 = mul_sum_i8_pairs(bx_1, by_1);

# 预取x[i] + 2 * sizeof(block_q4_0)的数据到缓存中
_mm_prefetch(&x[i] + 2 * sizeof(block_q4_0), _MM_HINT_T0);
# 预取y[i] + 2 * sizeof(block_q8_0)的数据到缓存中
_mm_prefetch(&y[i] + 2 * sizeof(block_q8_0), _MM_HINT_T0);

# 计算块2和3的组合比例
const __m128 d_2_3 = _mm_set1_ps( GGML_FP16_TO_FP32(x[i + 1].d) * GGML_FP16_TO_FP32(y[i + 1].d) );

# 从x[i + 1].qs地址加载一个128位的值到tmp_2_3
const __m128i tmp_2_3 = _mm_loadu_si128((const __m128i *)x[i + 1].qs);
# 使用低位掩码和tmp_2_3的值进行按位与运算
__m128i bx_2 = _mm_and_si128(lowMask, tmp_2_3);
# 从y[i + 1].qs地址加载一个128位的值到by_2
__m128i by_2 = _mm_loadu_si128((const __m128i *)y[i + 1].qs);
# 将bx_2减去off
bx_2 = _mm_sub_epi8(bx_2, off);
# 调用mul_sum_i8_pairs函数，传入bx_2和by_2，将结果存储在i32_2中
const __m128i i32_2 = mul_sum_i8_pairs(bx_2, by_2);

# 使用低位掩码和tmp_2_3的右移4位的值进行按位与运算
__m128i bx_3 = _mm_and_si128(lowMask, _mm_srli_epi64(tmp_2_3, 4));
# 从y[i + 1].qs + 16地址加载一个128位的值到by_3
__m128i by_3 = _mm_loadu_si128((const __m128i *)(y[i + 1].qs + 16));
        bx_3 = _mm_sub_epi8(bx_3, off); // 从 bx_3 中减去 off
        const __m128i i32_3 = mul_sum_i8_pairs(bx_3, by_3); // 使用 bx_3 和 by_3 计算乘积和

        // 将 int32_t 转换为 float
        __m128 p0 = _mm_cvtepi32_ps(i32_0);
        __m128 p1 = _mm_cvtepi32_ps(i32_1);
        __m128 p2 = _mm_cvtepi32_ps(i32_2);
        __m128 p3 = _mm_cvtepi32_ps(i32_3);

        // 应用比例
        __m128 p0_d = _mm_mul_ps( d_0_1, p0 );
        __m128 p1_d = _mm_mul_ps( d_0_1, p1 );
        __m128 p2_d = _mm_mul_ps( d_2_3, p2 );
        __m128 p3_d = _mm_mul_ps( d_2_3, p3 );

        // 累加
        acc_0 = _mm_add_ps(p0_d, acc_0);
        acc_1 = _mm_add_ps(p1_d, acc_1);
        acc_2 = _mm_add_ps(p2_d, acc_2);
        acc_3 = _mm_add_ps(p3_d, acc_3);
    }

    *s = hsum_float_4x4(acc_0, acc_1, acc_2, acc_3);
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    // 使用 __riscv_vsetvl_e8m1 函数设置向量长度 vl
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 加载元素
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 对 x 进行掩码操作，并存储低位部分和高位部分
        vuint8mf2_t x_a = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        vuint8mf2_t x_l = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        // 将 x_a 转换为有符号整数类型
        vint8mf2_t x_ai = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        // 将vint8mf2_t类型的向量x_l重新解释为vint8mf2_t类型的向量x_li
        vint8mf2_t x_li = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        // 从x_ai中减去偏移量8，存储结果到v0
        vint8mf2_t v0 = __riscv_vsub_vx_i8mf2(x_ai, 8, vl);
        // 从x_li中减去偏移量8，存储结果到v1
        vint8mf2_t v1 = __riscv_vsub_vx_i8mf2(x_li, 8, vl);

        // 将v0和y0的元素逐个相乘，存储结果到vec_mul1
        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        // 将v1和y1的元素逐个相乘，存储结果到vec_mul2
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        // 创建一个全为0的vint32m1_t类型的向量vec_zero
        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        // 将vec_mul1中的元素累加求和，存储结果到vs1
        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        // 将vec_mul2中的元素累加求和，存储结果到vs2
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        // 将vs2中的元素转换为标量int类型，存储结果到sumi
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        // 将sumi乘以x[i].d和y[i].d的浮点数值，加到sumf上
        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d);
    }

    // 将sumf的值存储到指针s指向的位置
    *s = sumf;
#else
    // 如果不是标量，则执行以下代码
    float sumf = 0.0; // 初始化浮点数 sumf 为 0.0

    for (int i = 0; i < nb; i++) { // 循环遍历 nb 次
        int sumi = 0; // 初始化整数 sumi 为 0

        for (int j = 0; j < qk/2; ++j) { // 循环遍历 qk/2 次
            const int v0 = (x[i].qs[j] & 0x0F) - 8; // 计算 v0
            const int v1 = (x[i].qs[j] >>   4) - 8; // 计算 v1

            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]); // 更新 sumi
        }

        sumf += sumi*GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d); // 更新 sumf
    }

    *s = sumf; // 将 sumf 赋值给指针 s 所指向的变量
#endif
}
// 计算两个向量的点积，其中n为向量长度，s为结果数组，vx和vy分别为输入向量
void ggml_vec_dot_q4_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义每个向量块的大小
    const int qk = QK8_1;
    // 计算向量长度对应的块数
    const int nb = n / qk;

    // 断言向量长度能够整除块大小
    assert(n % qk == 0);

    // 将输入向量转换为特定类型的指针
    const block_q4_1 * restrict x = vx;
    const block_q8_1 * restrict y = vy;

    // TODO: add WASM SIMD
    // 如果是ARM平台，使用NEON指令集进行SIMD加速
#if defined(__ARM_NEON)
    // 定义两个用于累加的向量
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 定义一个用于累加的标量
    float summs = 0;

    // 断言块数为偶数，暂不处理奇数块
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环遍历每个块，进行SIMD加速的点积计算
    for (int i = 0; i < nb; i += 2) {
// 定义指向 x 数组中第 i 和 i+1 位置的指针
const block_q4_1 * restrict x0 = &x[i + 0];
const block_q4_1 * restrict x1 = &x[i + 1;
// 定义指向 y 数组中第 i 和 i+1 位置的指针
const block_q8_1 * restrict y0 = &y[i + 0];
const block_q8_1 * restrict y1 = &y[i + 1];

// 计算 summs，将 x0 和 x1 的值转换为 32 位浮点数后与 y0 和 y1 的值相乘并相加
summs += GGML_FP16_TO_FP32(x0->m) * y0->s + GGML_FP16_TO_FP32(x1->m) * y1->s;

// 创建一个包含 0x0F 的 16 个元素的向量
const uint8x16_t m4b = vdupq_n_u8(0x0F);

// 从 x0 和 x1 中加载 16 个 8 位无符号整数到向量中
const uint8x16_t v0_0 = vld1q_u8(x0->qs);
const uint8x16_t v0_1 = vld1q_u8(x1->qs);

// 将 4 位整数扩展为 8 位整数
const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

// 从 y0 中加载 16 个 8 位有符号整数到向量中
const int8x16_t v1_0l = vld1q_s8(y0->qs);
        // 从指针 y0->qs + 16 处加载 16 个有符号 8 位整数到 int8x16_t 类型的寄存器 v1_0h
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        // 从指针 y1->qs 处加载 16 个有符号 8 位整数到 int8x16_t 类型的寄存器 v1_1l
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        // 从指针 y1->qs + 16 处加载 16 个有符号 8 位整数到 int8x16_t 类型的寄存器 v1_1h
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

#if defined(__ARM_FEATURE_DOTPROD)
        // 将 v0_0l 和 v1_0l 的每个对应元素相乘，然后将结果相加，存储到 int32x4_t 类型的寄存器 p_0
        const int32x4_t p_0 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_0l, v1_0l), v0_0h, v1_0h);
        // 将 v0_1l 和 v1_1l 的每个对应元素相乘，然后将结果相加，存储到 int32x4_t 类型的寄存器 p_1
        const int32x4_t p_1 = vdotq_s32(vdotq_s32(vdupq_n_s32(0), v0_1l, v1_1l), v0_1h, v1_1h);

        // 将 p_0 转换为 float32x4_t 类型，然后与 x0->d 和 y0->d 相乘，再与 sumv0 相加
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(p_0), GGML_FP16_TO_FP32(x0->d)*y0->d);
        // 将 p_1 转换为 float32x4_t 类型，然后与 x1->d 和 y1->d 相乘，再与 sumv1 相加
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(p_1), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
        // 将 v0_0l 和 v1_0l 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 pl0l
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0l), vget_low_s8 (v1_0l));
        // 将 v0_0l 和 v1_0l 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 pl0h
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0l), vget_high_s8(v1_0l));
        // 将 v0_0h 和 v1_0h 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 ph0l
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0h), vget_low_s8 (v1_0h));
        // 将 v0_0h 和 v1_0h 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 ph0h
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0h), vget_high_s8(v1_0h));

        // 将 v0_1l 和 v1_1l 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 pl1l
        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1l), vget_low_s8 (v1_1l));
        // 将 v0_1l 和 v1_1l 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 pl1h
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1l), vget_high_s8(v1_1l));
        // 将 v0_1h 和 v1_1h 的每个对应元素相乘，结果存储到 int16x8_t 类型的寄存器 ph1l
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1h), vget_low_s8 (v1_1h));
        // 使用 vget_high_s8 从 int8x16_t 向量中获取高位的 8 个元素，然后使用 vmull_s8 进行有符号 8 位整数乘法
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1h), vget_high_s8(v1_1h));

        // 使用 vpaddlq_s16 进行有符号 16 位整数相加和向上取整，然后使用 vaddq_s32 进行有符号 32 位整数相加
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 使用 vcvtq_f32_s32 进行有符号 32 位整数转换为单精度浮点数，然后使用 vmlaq_n_f32 进行单精度浮点数乘加
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
    }

    // 使用 vaddvq_f32 对四个单精度浮点数向量进行垂直加法，然后进行累加
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs;
#elif defined(__AVX2__) || defined(__AVX__)
    // 使用 _mm256_setzero_ps 初始化 256 位 AVX 寄存器为零
    __m256 acc = _mm256_setzero_ps();

    // 初始化 summs 为 0
    float summs = 0;

    // 主循环
    // 循环遍历，计算每个元素的值
    for (int i = 0; i < nb; ++i) {
        // 将x[i]的值从FP16转换为FP32
        const float d0 = GGML_FP16_TO_FP32(x[i].d);
        // 获取y[i]的值
        const float d1 = y[i].d;

        // 计算summs的累加值
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将d0的值转换为__m256类型
        const __m256 d0v = _mm256_set1_ps( d0 );
        // 将d1的值转换为__m256类型
        const __m256 d1v = _mm256_set1_ps( d1 );

        // 计算d0v和d1v的乘积
        const __m256 d0d1 = _mm256_mul_ps( d0v, d1v );

        // 从x[i].qs中加载16个字节，并将4个位字段解压缩成字节，生成32个字节
        const __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从y[i].qs中加载16个字节
        const __m256i by = _mm256_loadu_si256( (const __m256i *)y[i].qs );

        // 计算bx和by的乘积之和
        const __m256 xy = mul_sum_us8_pairs_float(bx, by);

        // 累加d0*d1*x*y的值
#if defined(__AVX2__)
        // 如果定义了__AVX2__宏，则使用AVX2指令集中的_mm256_fmadd_ps函数进行乘加运算，并将结果累加到acc中
        acc = _mm256_fmadd_ps( d0d1, xy, acc );
#else
        // 如果未定义__AVX2__宏，则使用AVX指令集中的_mm256_mul_ps和_mm256_add_ps函数进行乘法和加法运算，并将结果累加到acc中
        acc = _mm256_add_ps( _mm256_mul_ps( d0d1, xy ), acc );
#endif
    }

    // 计算累加和并加上summs
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    // 定义一个浮点数sumf，并初始化为0.0
    float sumf = 0.0;

    // 使用__riscv_vsetvl_e8m1函数设置向量长度vl为qk/2
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 遍历nb次循环
    for (int i = 0; i < nb; i++) {
        // 加载元素到向量tx
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        // 加载y[i].qs中的前16个元素到向量y0
        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        // 加载y[i].qs中的后16个元素到向量y1
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 掩码和存储x的低部分，然后是上部分
        # 使用位与操作将 tx 中的每个字节与 0x0F 进行按位与操作
        vuint8mf2_t x_a = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        # 使用逻辑右移操作将 tx 中的每个字节向右移动 4 位
        vuint8mf2_t x_l = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        # 将 x_a 转换为有符号 8 位整数
        vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
        # 将 x_l 转换为有符号 8 位整数
        vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

        # 对 v0 和 y0 中的每个元素进行有符号 16 位整数乘法
        vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
        # 对 v1 和 y1 中的每个元素进行有符号 16 位整数乘法
        vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

        # 创建一个全为 0 的有符号 32 位整数向量
        vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

        # 对 vec_mul1 中的元素进行有符号 16 位整数累加，并将结果存储在 vs1 中
        vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
        # 对 vec_mul2 中的元素进行有符号 16 位整数累加，并将结果存储在 vs2 中
        vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

        # 将 vs2 中的元素转换为标量有符号 32 位整数
        int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

        # 计算最终结果并将其加到 sumf 中
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    # 将最终结果存储在 s 中
    *s = sumf;
#else
    // 如果不是标量，则执行以下代码

    // 定义一个浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 定义一个整数变量 sumi，并初始化为 0
        int sumi = 0;

        // 内层循环遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 从 x[i].qs[j] 中取出低 4 位的值
            const int v0 = (x[i].qs[j] & 0x0F);
            // 从 x[i].qs[j] 中取出高 4 位的值
            const int v1 = (x[i].qs[j] >>   4);

            // 计算 sumi 的值
            sumi += (v0 * y[i].qs[j]) + (v1 * y[i].qs[j + qk/2]);
        }

        // 计算 sumf 的值
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }

    // 将 sumf 的值赋给指针 s 所指向的变量
    *s = sumf;
#endif
}
// 计算两个向量的点积，其中 n 为向量长度，s 为结果数组，vx 和 vy 分别为输入向量
void ggml_vec_dot_q5_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义向量的长度
    const int qk = QK8_0;
    // 计算向量长度的块数
    const int nb = n / qk;

    // 断言向量长度能够整除块大小
    assert(n % qk == 0);
    // 断言块大小与预定义的大小相等
    assert(qk == QK5_0);

    // 将输入向量转换为特定类型的指针
    const block_q5_0 * restrict x = vx;
    const block_q8_0 * restrict y = vy;

    // 如果支持 ARM NEON 指令集
#if defined(__ARM_NEON)
    // 创建两个初始值为 0 的 4 个单精度浮点数向量
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);

    // 定义两个无符号整数变量
    uint32_t qh0;
    uint32_t qh1;

    // 定义两个 64 位无符号整数数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];
    // 确保 nb 是偶数，如果不是，需要处理奇数情况
    assert(nb % 2 == 0);

    // 遍历数组 x，每次增加 2
    for (int i = 0; i < nb; i += 2) {
        // 定义指向 x 和 y 数组中特定位置的指针
        const block_q5_0 * restrict x0 = &x[i];
        const block_q5_0 * restrict x1 = &x[i + 1];
        const block_q8_0 * restrict y0 = &y[i];
        const block_q8_0 * restrict y1 = &y[i + 1];

        // 创建一个包含 0x0F 的 16 个元素的向量
        const uint8x16_t m4b = vdupq_n_u8(0x0F);

        // 通过查找表提取第 5 位的值
        memcpy(&qh0, x0->qh, sizeof(qh0));
        memcpy(&qh1, x1->qh, sizeof(qh1));

        tmp0[0] = table_b2b_1[(qh0 >>  0) & 0xFF];
        tmp0[1] = table_b2b_1[(qh0 >>  8) & 0xFF];
        tmp0[2] = table_b2b_1[(qh0 >> 16) & 0xFF];
        tmp0[3] = table_b2b_1[(qh0 >> 24)       ];
        # 将 qh1 按位右移 0、8、16、24 位，并根据结果在 table_b2b_1 中查找对应的值，存入 tmp1 数组
        tmp1[0] = table_b2b_1[(qh1 >>  0) & 0xFF];
        tmp1[1] = table_b2b_1[(qh1 >>  8) & 0xFF];
        tmp1[2] = table_b2b_1[(qh1 >> 16) & 0xFF];
        tmp1[3] = table_b2b_1[(qh1 >> 24)       ];

        # 从 tmp0 和 tmp1 数组中加载数据，分别存入 qhl0、qhh0、qhl1、qhh1 四个变量中
        const int8x16_t qhl0 = vld1q_s8((const int8_t *)(tmp0 + 0));
        const int8x16_t qhh0 = vld1q_s8((const int8_t *)(tmp0 + 2));
        const int8x16_t qhl1 = vld1q_s8((const int8_t *)(tmp1 + 0));
        const int8x16_t qhh1 = vld1q_s8((const int8_t *)(tmp1 + 2));

        # 从 x0 和 x1 的 qs 数组中加载数据，分别存入 v0_0 和 v0_1 两个变量中
        const uint8x16_t v0_0 = vld1q_u8(x0->qs);
        const uint8x16_t v0_1 = vld1q_u8(x1->qs);

        # 将 v0_0 和 v0_1 中的每个 4 位数据扩展为 8 位，并存入对应的 v0_0l、v0_0h、v0_1l、v0_1h 变量中
        int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
        int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
        int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
        int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

        # 将高位和低位数据合并，并减去 16，结果存入对应的变量中
        // 计算 v0_0lf，v0_0hf，v0_1lf，v0_1hf 分别为 v0_0l，v0_0h，v0_1l，v0_1h 与 qhl0，qhh0，qhl1，qhh1 的差值
        const int8x16_t v0_0lf = vsubq_s8(v0_0l, qhl0);
        const int8x16_t v0_0hf = vsubq_s8(v0_0h, qhh0);
        const int8x16_t v0_1lf = vsubq_s8(v0_1l, qhl1);
        const int8x16_t v0_1hf = vsubq_s8(v0_1h, qhh1);

        // 加载 y 的值
        const int8x16_t v1_0l = vld1q_s8(y0->qs);
        const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

#if defined(__ARM_FEATURE_DOTPROD)
        // 使用 ARM dot product 指令计算 sumv0 和 sumv1 的值
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#else
        // 如果不支持 ARM dot product 指令，则使用传统方法计算 pl0l 的值
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        // 使用 vmull_s8 函数将两个 int8x8_t 类型的向量的对应元素相乘，并将结果扩展为 int16x8_t 类型
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));

        // 将两个 int16x8_t 类型的向量的对应元素相加，并将结果扩展为 int32x4_t 类型
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        // 将两个 int16x8_t 类型的向量的对应元素相加，并将结果扩展为 int32x4_t 类型
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        // 将两个 int16x8_t 类型的向量的对应元素相加，并将结果扩展为 int32x4_t 类型
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        // 将两个 int16x8_t 类型的向量的对应元素相加，并将结果扩展为 int32x4_t 类型
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 使用 vaddq_s32 函数将两个 int32x4_t 类型的向量的对应元素相加
        // 使用 vcvtq_f32_s32 函数将 int32x4_t 类型的向量转换为 float32x4_t 类型的向量
        // 使用 vmlaq_n_f32 函数将两个 float32x4_t 类型的向量相乘，并加上一个 float32x4_t 类型的向量
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        // 使用 vaddvq_f32 函数将 float32x4_t 类型的向量的所有元素相加
        *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
# 如果定义了 __wasm_simd128__，则执行以下代码
v128_t sumv = wasm_f32x4_splat(0.0f);  # 创建一个包含四个浮点数的 SIMD 向量，并将其所有元素初始化为 0.0

uint32_t qh;  # 定义一个 32 位无符号整数变量
uint64_t tmp[4];  # 定义一个包含 4 个 64 位无符号整数的数组

# 循环处理 nb 次
for (int i = 0; i < nb; ++i) {
    const block_q5_0 * restrict x0 = &x[i];  # 定义一个指向 block_q5_0 结构体的指针，并指向 x[i] 的地址
    const block_q8_0 * restrict y0 = &y[i];  # 定义一个指向 block_q8_0 结构体的指针，并指向 y[i] 的地址

    const v128_t m4b  = wasm_i8x16_splat(0x0F);  # 创建一个包含 16 个 8 位整数的 SIMD 向量，并将其所有元素初始化为 0x0F

    memcpy(&qh, x0->qh, sizeof(qh));  # 将 x0->qh 的内容复制到 qh 中

    # 从 qh 中提取第 5 位的数据，并根据提取的数据查找 table_b2b_1 中对应的值，存储到 tmp 数组中
    tmp[0] = table_b2b_1[(qh >>  0) & 0xFF];
    tmp[1] = table_b2b_1[(qh >>  8) & 0xFF];
    tmp[2] = table_b2b_1[(qh >> 16) & 0xFF];
    tmp[3] = table_b2b_1[(qh >> 24)       ];
// 从内存中加载 128 位数据到寄存器中
const v128_t qhl = wasm_v128_load(tmp + 0);
const v128_t qhh = wasm_v128_load(tmp + 2);

// 从内存中加载 128 位数据到寄存器中
const v128_t v0 = wasm_v128_load(x0->qs);

// 将 4 位数据扩展为 8 位数据
const v128_t v0l = wasm_v128_and (v0, m4b);
const v128_t v0h = wasm_u8x16_shr(v0, 4);

// 对高位进行加法和低位进行减法
const v128_t v0lf = wasm_i8x16_sub(v0l, qhl);
const v128_t v0hf = wasm_i8x16_sub(v0h, qhh);

// 从内存中加载 128 位数据到寄存器中
const v128_t v1l = wasm_v128_load(y0->qs);
const v128_t v1h = wasm_v128_load(y0->qs + 16);

// 将 8 位数据扩展为 16 位数据
const v128_t v0lfl = wasm_i16x8_extend_low_i8x16 (v0lf);
        // 将v0lf的高8位扩展为16位整数
        const v128_t v0lfh = wasm_i16x8_extend_high_i8x16(v0lf);
        // 将v0hf的低8位扩展为16位整数
        const v128_t v0hfl = wasm_i16x8_extend_low_i8x16 (v0hf);
        // 将v0hf的高8位扩展为16位整数
        const v128_t v0hfh = wasm_i16x8_extend_high_i8x16(v0hf);

        // 将v1l的低8位扩展为16位整数
        const v128_t v1ll = wasm_i16x8_extend_low_i8x16 (v1l);
        // 将v1l的高8位扩展为16位整数
        const v128_t v1lh = wasm_i16x8_extend_high_i8x16(v1l);
        // 将v1h的低8位扩展为16位整数
        const v128_t v1hl = wasm_i16x8_extend_low_i8x16 (v1h);
        // 将v1h的高8位扩展为16位整数
        const v128_t v1hh = wasm_i16x8_extend_high_i8x16(v1h);

        // 计算点积
        sumv = wasm_f32x4_add(sumv, wasm_f32x4_mul(wasm_f32x4_convert_i32x4(
                        wasm_i32x4_add(
                            wasm_i32x4_add(wasm_i32x4_dot_i16x8(v0lfl, v1ll),
                                           wasm_i32x4_dot_i16x8(v0lfh, v1lh)),
                            wasm_i32x4_add(wasm_i32x4_dot_i16x8(v0hfl, v1hl),
                                           wasm_i32x4_dot_i16x8(v0hfh, v1hh)))),
                    wasm_f32x4_splat(GGML_FP16_TO_FP32(x0->d) * GGML_FP16_TO_FP32(y0->d))));
    }

    // 将sumv的第0和第1个元素提取出来相加
    *s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
    // 如果编译器支持 AVX2 指令集，则执行以下代码块
    // 使用 _mm256_setzero_ps() 初始化一个256位的浮点数向量为全零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 计算块的组合比例
        // 将 x[i].d 和 y[i].d 转换为单精度浮点数，然后相乘
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 x[i].qs 转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为字节，并将高位清零
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        bxhi = _mm256_andnot_si256(bxhi, _mm256_set1_epi8((char)0xF0));
        bx = _mm256_or_si256(bx, bxhi);

        // 从 y[i].qs 加载 256 位整数向量
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算乘积并求和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 将 q 乘以比例并累加到 acc 中
    // 使用 FMA 指令计算乘加操作，并将结果累加到累加器中
    acc = _mm256_fmadd_ps(d, q, acc);
    }

    // 计算累加器中所有元素的和并存储到指定地址
    *s = hsum_float_8(acc);
#elif defined(__AVX__)
    // 使用 AVX 指令集初始化一个全为零的累加器
    __m256 acc = _mm256_setzero_ps();
    // 使用 AVX 指令集创建一个全为0xF0的掩码
    __m128i mask = _mm_set1_epi8((char)0xF0);

    // 主循环
    for (int i = 0; i < nb; i++) {
        /* 计算块的组合比例 */
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));

        // 将 32 个四位数转换为字节
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 从 bx 中提取高 128 位
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        // 对 bxhil 和 bxhih 进行按位取反并与掩码进行与操作
        bxhil = _mm_andnot_si128(bxhil, mask);
        bxhih = _mm_andnot_si128(bxhih, mask);
        # 将 bx 的低128位存储到 bxl 中
        __m128i bxl = _mm256_castsi256_si128(bx);
        # 将 bx 的高128位存储到 bxh 中
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        # 将 bxhil 中的值与 bxl 进行按位或操作
        bxl = _mm_or_si128(bxl, bxhil);
        # 将 bxhih 中的值与 bxh 进行按位或操作
        bxh = _mm_or_si128(bxh, bxhih);
        # 将 bxl 和 bxh 合并成一个新的 bx
        bx = MM256_SET_M128I(bxh, bxl);

        # 从内存中加载 y[i].qs 到 by 中
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        # 调用 mul_sum_i8_pairs_float 函数，将 bx 和 by 进行乘法运算并求和，结果存储在 q 中
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        # 将 d 与 q 相乘，然后与 acc 相加，结果存储在 acc 中
        acc = _mm256_add_ps(_mm256_mul_ps(d, q), acc);
    }

    # 计算 acc 中所有元素的和，存储在 s 中
    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    # 初始化 sumf 为 0.0
    float sumf = 0.0;

    # 初始化 qh 为 0
    uint32_t qh;
    // 根据输入参数计算出适合当前指令集的向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 创建临时寄存器用于掩码和移位操作
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vsll_vv_u32m2(__riscv_vmv_v_x_u32m2(1, vl), vt_1, vl);

    vuint32m2_t vt_3 = __riscv_vsll_vx_u32m2(vt_2, 16, vl);
    vuint32m2_t vt_4 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    for (int i = 0; i < nb; i++) {
        // 从结构体 x 中复制 qh 的值
        memcpy(&qh, x[i].qh, sizeof(uint32_t));

        // 计算 ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4
        vuint32m2_t xha_0 = __riscv_vand_vx_u32m2(vt_2, qh, vl);
        vuint32m2_t xhr_0 = __riscv_vsrl_vv_u32m2(xha_0, vt_1, vl);
        vuint32m2_t xhl_0 = __riscv_vsll_vx_u32m2(xhr_0, 4, vl);

        // 计算 ((qh & (1u << (j + 16))) >> (j + 12))
        vuint32m2_t xha_1 = __riscv_vand_vx_u32m2(vt_3, qh, vl);
        vuint32m2_t xhl_1 = __riscv_vsrl_vv_u32m2(xha_1, vt_4, vl);
// 使用指定的宽度将 xhl_0 转换为 uint16 类型
vuint16m1_t xhc_0 = __riscv_vncvt_x_x_w_u16m1(xhl_0, vl);
// 使用指定的宽度将 xhc_0 转换为 uint8 类型
vuint8mf2_t xh_0 = __riscv_vncvt_x_x_w_u8mf2(xhc_0, vl);

// 使用指定的宽度将 xhl_1 转换为 uint16 类型
vuint16m1_t xhc_1 = __riscv_vncvt_x_x_w_u16m1(xhl_1, vl);
// 使用指定的宽度将 xhc_1 转换为 uint8 类型
vuint8mf2_t xh_1 = __riscv_vncvt_x_x_w_u8mf2(xhc_1, vl);

// 从内存中加载 uint8 类型的数据到寄存器中
vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

// 从内存中加载 int8 类型的数据到寄存器中
vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

// 对 tx 中的数据进行位与运算
vuint8mf2_t x_at = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
// 对 tx 中的数据进行逻辑右移运算
vuint8mf2_t x_lt = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

// 对 x_at 和 xh_0 中的数据进行逻辑或运算
vuint8mf2_t x_a = __riscv_vor_vv_u8mf2(x_at, xh_0, vl);
// 对 x_lt 和 xh_1 中的数据进行逻辑或运算
vuint8mf2_t x_l = __riscv_vor_vv_u8mf2(x_lt, xh_1, vl);
# 将x_a转换为vint8mf2_t类型
vint8mf2_t x_ai = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
# 将x_l转换为vint8mf2_t类型
vint8mf2_t x_li = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

# 对x_ai中的每个元素减去16
vint8mf2_t v0 = __riscv_vsub_vx_i8mf2(x_ai, 16, vl);
# 对x_li中的每个元素减去16
vint8mf2_t v1 = __riscv_vsub_vx_i8mf2(x_li, 16, vl);

# 对v0和y0中的每个元素进行有符号16位乘法
vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
# 对v1和y1中的每个元素进行有符号16位乘法
vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

# 创建一个全为0的vint32m1_t类型向量
vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

# 对vec_mul1中的元素进行有符号16位累加求和，结果为vint32m1_t类型
vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
# 对vec_mul2中的元素进行有符号16位累加求和，结果与vs1相加，结果为vint32m1_t类型
vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

# 将vint32m1_t类型的vs2转换为int类型
int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

# 将x[i].d和y[i].d转换为32位浮点数，相乘后再乘以sumi，加到sumf上
sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;

# 将sumf的值赋给指针s所指向的变量
*s = sumf;
#else
    // 如果不是上述条件，执行以下操作
    // 定义一个浮点数变量 sumf，并初始化为 0.0
    float sumf = 0.0;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 定义一个 32 位无符号整数变量 qh，并从 x[i].qh 复制数据到 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        // 定义一个整数变量 sumi，并初始化为 0
        int sumi = 0;

        // 再次循环遍历 qk/2 次
        for (int j = 0; j < qk/2; ++j) {
            // 定义一个 8 位无符号整数变量 xh_0，并根据位运算从 qh 中提取数据
            const uint8_t xh_0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
            // 定义一个 8 位无符号整数变量 xh_1，并根据位运算从 qh 中提取数据
            const uint8_t xh_1 = ((qh & (1u << (j + 16))) >> (j + 12));

            // 定义两个 32 位整数变量 x0 和 x1，并根据位运算和数据处理计算出它们的值
            const int32_t x0 = ((x[i].qs[j] & 0x0F) | xh_0) - 16;
            const int32_t x1 = ((x[i].qs[j] >>   4) | xh_1) - 16;

            // 将 x0 和 x1 与 y[i].qs[j] 和 y[i].qs[j + qk/2] 相乘，并加到 sumi 中
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }
// 计算两个向量的点积，并将结果累加到 sumf 中
sumf += (GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d)) * sumi;

}

// 计算两个向量的点积，其中 n 为向量的长度，s 为结果指针，vx 和 vy 分别为两个向量的指针
void ggml_vec_dot_q5_1_q8_1(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义常量 qk 为 QK8_1，nb 为 n 除以 qk 的商
    const int qk = QK8_1;
    const int nb = n / qk;

    // 断言 n 能被 qk 整除
    assert(n % qk == 0);
    // 断言 qk 等于 QK5_1
    assert(qk == QK5_1);

    // 定义指向向量的指针 x 和 y
    const block_q5_1 * restrict x = vx;
    const block_q8_1 * restrict y = vy;

#if defined(__ARM_NEON)
    // 创建两个长度为 4 的浮点数向量，并初始化为 0
    float32x4_t sumv0 = vdupq_n_f32(0.0f);
    float32x4_t sumv1 = vdupq_n_f32(0.0f);
    // 初始化两个浮点数变量为0
    float summs0 = 0.0f;
    float summs1 = 0.0f;

    // 初始化两个32位无符号整数变量
    uint32_t qh0;
    uint32_t qh1;

    // 初始化两个64位无符号整数数组
    uint64_t tmp0[4];
    uint64_t tmp1[4];

    // 断言nb为偶数，如果不是偶数则抛出异常
    assert(nb % 2 == 0); // TODO: handle odd nb

    // 循环遍历，每次增加2
    for (int i = 0; i < nb; i += 2) {
        // 定义指向x和y数组中特定位置的指针
        const block_q5_1 * restrict x0 = &x[i];
        const block_q5_1 * restrict x1 = &x[i + 1];
        const block_q8_1 * restrict y0 = &y[i];
        const block_q8_1 * restrict y1 = &y[i + 1];

        // 创建一个包含16个8位无符号整数的向量，每个元素都是0x0F
        const uint8x16_t m4b = vdupq_n_u8(0x0F);
// 计算两个变量的乘积并累加到summs0和summs1中
summs0 += GGML_FP16_TO_FP32(x0->m) * y0->s;
summs1 += GGML_FP16_TO_FP32(x1->m) * y1->s;

// 从x0和x1的qh中复制数据到qh0和qh1
memcpy(&qh0, x0->qh, sizeof(qh0));
memcpy(&qh1, x1->qh, sizeof(qh1));

// 通过查找表提取qh0的第5位 ((b) << 4)
tmp0[0] = table_b2b_0[(qh0 >>  0) & 0xFF];
tmp0[1] = table_b2b_0[(qh0 >>  8) & 0xFF];
tmp0[2] = table_b2b_0[(qh0 >> 16) & 0xFF];
tmp0[3] = table_b2b_0[(qh0 >> 24)       ];

// 通过查找表提取qh1的第5位 ((b) << 4)
tmp1[0] = table_b2b_0[(qh1 >>  0) & 0xFF];
tmp1[1] = table_b2b_0[(qh1 >>  8) & 0xFF];
tmp1[2] = table_b2b_0[(qh1 >> 16) & 0xFF];
tmp1[3] = table_b2b_0[(qh1 >> 24)       ];

// 从tmp0和tmp1中加载数据到qhl0、qhh0和qhl1中
const int8x16_t qhl0 = vld1q_s8((const int8_t *)(tmp0 + 0));
const int8x16_t qhh0 = vld1q_s8((const int8_t *)(tmp0 + 2));
const int8x16_t qhl1 = vld1q_s8((const int8_t *)(tmp1 + 0));
// 从指向tmp1 + 2的地址加载16个有符号8位整数，存储到qhh1中
const int8x16_t qhh1 = vld1q_s8((const int8_t *)(tmp1 + 2));

// 从x0->qs指向的地址加载16个无符号8位整数，存储到v0_0中
const uint8x16_t v0_0 = vld1q_u8(x0->qs);
// 从x1->qs指向的地址加载16个无符号8位整数，存储到v0_1中
const uint8x16_t v0_1 = vld1q_u8(x1->qs);

// 将4位整数转换为8位整数
const int8x16_t v0_0l = vreinterpretq_s8_u8(vandq_u8  (v0_0, m4b));
const int8x16_t v0_0h = vreinterpretq_s8_u8(vshrq_n_u8(v0_0, 4));
const int8x16_t v0_1l = vreinterpretq_s8_u8(vandq_u8  (v0_1, m4b));
const int8x16_t v0_1h = vreinterpretq_s8_u8(vshrq_n_u8(v0_1, 4));

// 添加高位
const int8x16_t v0_0lf = vorrq_s8(v0_0l, qhl0);
const int8x16_t v0_0hf = vorrq_s8(v0_0h, qhh0);
const int8x16_t v0_1lf = vorrq_s8(v0_1l, qhl1);
const int8x16_t v0_1hf = vorrq_s8(v0_1h, qhh1);

// 加载y
const int8x16_t v1_0l = vld1q_s8(y0->qs);
const int8x16_t v1_0h = vld1q_s8(y0->qs + 16);
        // 从指针 y1->qs 加载 16 个有符号 8 位整数到寄存器 v1_1l
        const int8x16_t v1_1l = vld1q_s8(y1->qs);
        // 从指针 y1->qs + 16 加载 16 个有符号 8 位整数到寄存器 v1_1h
        const int8x16_t v1_1h = vld1q_s8(y1->qs + 16);

#if defined(__ARM_FEATURE_DOTPROD)
        // 使用 SIMD 指令计算两个向量的点积，并将结果转换为浮点数，然后进行一系列乘法和加法操作
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_0lf, v1_0l),
                        vdotq_s32(vdupq_n_s32(0), v0_0hf, v1_0h))), GGML_FP16_TO_FP32(x0->d)*y0->d);
        // 使用 SIMD 指令计算两个向量的点积，并将结果转换为浮点数，然后进行一系列乘法和加法操作
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), v0_1lf, v1_1l),
                        vdotq_s32(vdupq_n_s32(0), v0_1hf, v1_1h))), GGML_FP16_TO_FP32(x1->d)*y1->d);
#else
        // 使用 SIMD 指令计算两个向量的点积，并将结果转换为 16 位整数
        const int16x8_t pl0l = vmull_s8(vget_low_s8 (v0_0lf), vget_low_s8 (v1_0l));
        const int16x8_t pl0h = vmull_s8(vget_high_s8(v0_0lf), vget_high_s8(v1_0l));
        const int16x8_t ph0l = vmull_s8(vget_low_s8 (v0_0hf), vget_low_s8 (v1_0h));
        const int16x8_t ph0h = vmull_s8(vget_high_s8(v0_0hf), vget_high_s8(v1_0h));

        const int16x8_t pl1l = vmull_s8(vget_low_s8 (v0_1lf), vget_low_s8 (v1_1l));
        const int16x8_t pl1h = vmull_s8(vget_high_s8(v0_1lf), vget_high_s8(v1_1l));
        const int16x8_t ph1l = vmull_s8(vget_low_s8 (v0_1hf), vget_low_s8 (v1_1h));
        const int16x8_t ph1h = vmull_s8(vget_high_s8(v0_1hf), vget_high_s8(v1_1h));
#endif
        // 计算四个 int16x8_t 向量的低位和高位的和，并将结果转换为 int32x4_t 向量
        const int32x4_t pl0 = vaddq_s32(vpaddlq_s16(pl0l), vpaddlq_s16(pl0h));
        const int32x4_t ph0 = vaddq_s32(vpaddlq_s16(ph0l), vpaddlq_s16(ph0h));
        const int32x4_t pl1 = vaddq_s32(vpaddlq_s16(pl1l), vpaddlq_s16(pl1h));
        const int32x4_t ph1 = vaddq_s32(vpaddlq_s16(ph1l), vpaddlq_s16(ph1h));

        // 使用 vaddq_s32 计算两个 int32x4_t 向量的和，并将结果转换为 float32x4_t 向量，然后使用 vmlaq_n_f32 计算乘法和加法
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(pl0, ph0)), GGML_FP16_TO_FP32(x0->d)*y0->d);
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(pl1, ph1)), GGML_FP16_TO_FP32(x1->d)*y1->d);
#endif
    }

    // 将两个 float32x4_t 向量的和相加，并将结果存储在 *s 中
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1) + summs0 + summs1;
#elif defined(__wasm_simd128__)
    // 创建一个所有元素为 0.0f 的 float32x4_t 向量
    v128_t sumv = wasm_f32x4_splat(0.0f);

    // 初始化一个浮点数变量 summs 为 0.0f
    float summs = 0.0f;

    // 声明变量 qh 和 tmp
    uint32_t qh;
    uint64_t tmp[4];
    // 循环遍历nb次，对x和y数组中的元素进行操作
    for (int i = 0; i < nb; ++i) {
        // 获取指向x[i]的指针
        const block_q5_1 * restrict x0 = &x[i];
        // 获取指向y[i]的指针
        const block_q8_1 * restrict y0 = &y[i];

        // 计算GGML_FP16_TO_FP32(x0->m) * y0->s的和
        summs += GGML_FP16_TO_FP32(x0->m) * y0->s;

        // 创建一个包含0x0F的v128_t类型的向量
        const v128_t m4b = wasm_i8x16_splat(0x0F);

        // 从x0->qh中提取第5位
        memcpy(&qh, x0->qh, sizeof(qh));

        // 使用位运算从qh中提取4个字节的数据，并查表得到结果存入tmp数组
        tmp[0] = table_b2b_0[(qh >>  0) & 0xFF];
        tmp[1] = table_b2b_0[(qh >>  8) & 0xFF];
        tmp[2] = table_b2b_0[(qh >> 16) & 0xFF];
        tmp[3] = table_b2b_0[(qh >> 24)       ];

        // 从tmp数组中加载数据到qhl和qhh向量中
        const v128_t qhl = wasm_v128_load(tmp + 0);
        const v128_t qhh = wasm_v128_load(tmp + 2);
    }
        // 从内存中加载 x0->qs 中的数据到 v128_t 类型的变量 v0
        const v128_t v0 = wasm_v128_load(x0->qs);

        // 将 v0 中的每个 4 位数据扩展为 8 位数据
        const v128_t v0l = wasm_v128_and (v0, m4b);
        const v128_t v0h = wasm_u8x16_shr(v0, 4);

        // 将 v0l 和 v0h 中的数据的最高位设置为 1
        const v128_t v0lf = wasm_v128_or(v0l, qhl);
        const v128_t v0hf = wasm_v128_or(v0h, qhh);

        // 从内存中加载 y0->qs 和 y0->qs + 16 中的数据到 v128_t 类型的变量 v1l 和 v1h
        const v128_t v1l = wasm_v128_load(y0->qs);
        const v128_t v1h = wasm_v128_load(y0->qs + 16);

        // 将 v0lf 和 v0hf 中的每个 8 位数据扩展为 16 位数据
        const v128_t v0lfl = wasm_i16x8_extend_low_i8x16 (v0lf);
        const v128_t v0lfh = wasm_i16x8_extend_high_i8x16(v0lf);
        const v128_t v0hfl = wasm_i16x8_extend_low_i8x16 (v0hf);
        const v128_t v0hfh = wasm_i16x8_extend_high_i8x16(v0hf);
// 将v1l的低位字节扩展为16位整数
const v128_t v1ll = wasm_i16x8_extend_low_i8x16 (v1l);
// 将v1l的高位字节扩展为16位整数
const v128_t v1lh = wasm_i16x8_extend_high_i8x16(v1l);
// 将v1h的低位字节扩展为16位整数
const v128_t v1hl = wasm_i16x8_extend_low_i8x16 (v1h);
// 将v1h的高位字节扩展为16位整数
const v128_t v1hh = wasm_i16x8_extend_high_i8x16(v1h);

// 计算点积
sumv = wasm_f32x4_add(sumv,
        wasm_f32x4_mul(wasm_f32x4_convert_i32x4(wasm_i32x4_add(
                    wasm_i32x4_add(wasm_i32x4_dot_i16x8(v0lfl, v1ll),
                                   wasm_i32x4_dot_i16x8(v0lfh, v1lh)),
                    wasm_i32x4_add(wasm_i32x4_dot_i16x8(v0hfl, v1hl),
                                   wasm_i32x4_dot_i16x8(v0hfh, v1hh)))),
            wasm_f32x4_splat(GGML_FP16_TO_FP32(x0->d) * y0->d)));
}

// 将sumv的每个元素提取出来相加，并加上summs
*s = wasm_f32x4_extract_lane(sumv, 0) + wasm_f32x4_extract_lane(sumv, 1) +
     wasm_f32x4_extract_lane(sumv, 2) + wasm_f32x4_extract_lane(sumv, 3) + summs;
#elif defined(__AVX2__)
// 用零初始化累加器
__m256 acc = _mm256_setzero_ps();
    // 初始化浮点数求和变量
    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将x[i]的数据转换为__m256类型的向量
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算x[i].m和y[i].s的乘积，并加到summs上
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将x[i].qs转换为字节流
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将x[i].qh转换为字节流，并进行位与运算
        __m256i bxhi = bytes_from_bits_32(x[i].qh);
        bxhi = _mm256_and_si256(bxhi, _mm256_set1_epi8(0x10));
        bx = _mm256_or_si256(bx, bxhi);

        // 将y[i].d转换为__m256类型的向量
        const __m256 dy = _mm256_set1_ps(y[i].d);
        // 将y[i].qs转换为__m256i类型的向量
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        // 计算bx和by的乘积，并求和
        const __m256 q = mul_sum_us8_pairs_float(bx, by);

        // 将q和(dx * dy)的乘积加到acc上
        acc = _mm256_fmadd_ps(q, _mm256_mul_ps(dx, dy), acc);
    }

    *s = hsum_float_8(acc) + summs;
#elif defined(__AVX__)
    // 使用 AVX 指令集进行计算

    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();
    __m128i mask = _mm_set1_epi8(0x10);

    float summs = 0.0f;

    // 主循环
    for (int i = 0; i < nb; i++) {
        // 将 GGML_FP16_TO_FP32(x[i].d) 转换为 __m256 类型
        const __m256 dx = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d));

        // 计算 summs
        summs += GGML_FP16_TO_FP32(x[i].m) * y[i].s;

        // 将 x[i].qs 转换为 __m256i 类型
        __m256i bx = bytes_from_nibbles_32(x[i].qs);
        // 将 x[i].qh 转换为 __m256i 类型
        const __m256i bxhi = bytes_from_bits_32(x[i].qh);
        // 将 bxhi 的低 128 位转换为 __m128i 类型
        __m128i bxhil = _mm256_castsi256_si128(bxhi);
        // 将 bxhi 的高 128 位转换为 __m128i 类型
        __m128i bxhih = _mm256_extractf128_si256(bxhi, 1);
        # 使用掩码对 bxhil 进行按位与操作
        bxhil = _mm_and_si128(bxhil, mask);
        # 使用掩码对 bxhih 进行按位与操作
        bxhih = _mm_and_si128(bxhih, mask);
        # 将 bx 转换为 128 位的整型
        __m128i bxl = _mm256_castsi256_si128(bx);
        # 从 bx 中提取高 128 位的整型
        __m128i bxh = _mm256_extractf128_si256(bx, 1);
        # 对 bxl 和 bxhil 进行按位或操作
        bxl = _mm_or_si128(bxl, bxhil);
        # 对 bxh 和 bxhih 进行按位或操作
        bxh = _mm_or_si128(bxh, bxhih);
        # 将 bxl 和 bxh 合并成一个 256 位整型
        bx = MM256_SET_M128I(bxh, bxl);

        # 创建一个包含 y[i].d 的 256 位浮点数
        const __m256 dy = _mm256_set1_ps(y[i].d);
        # 从 y[i].qs 加载一个 256 位整型
        const __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);

        # 对 bx 和 by 进行乘法和求和操作
        const __m256 q = mul_sum_us8_pairs_float(bx, by);

        # 将 q 乘以 dx 和 dy，然后加到 acc 中
        acc = _mm256_add_ps(_mm256_mul_ps(q, _mm256_mul_ps(dx, dy)), acc);
    }

    # 计算 acc 中所有元素的和，并加上 summs
    *s = hsum_float_8(acc) + summs;
#elif defined(__riscv_v_intrinsic)
    # 初始化 sumf 为 0.0
    float sumf = 0.0;
    // 声明一个32位无符号整数变量 qh
    uint32_t qh;

    // 计算向量长度 vl，使用__riscv_vsetvl_e8m1函数
    size_t vl = __riscv_vsetvl_e8m1(qk/2);

    // 为移位操作声明临时寄存器
    vuint32m2_t vt_1 = __riscv_vid_v_u32m2(vl);
    vuint32m2_t vt_2 = __riscv_vadd_vx_u32m2(vt_1, 12, vl);

    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 从结构体 x[i] 中复制 qh 的值到变量 qh
        memcpy(&qh, x[i].qh, sizeof(uint32_t));

        // 将 qh 加载到向量寄存器 vqh 中
        vuint32m2_t vqh = __riscv_vmv_v_x_u32m2(qh, vl);

        // 计算 ((qh >> (j +  0)) << 4) & 0x10
        vuint32m2_t xhr_0 = __riscv_vsrl_vv_u32m2(vqh, vt_1, vl);
        vuint32m2_t xhl_0 = __riscv_vsll_vx_u32m2(xhr_0, 4, vl);
        vuint32m2_t xha_0 = __riscv_vand_vx_u32m2(xhl_0, 0x10, vl);

        // 计算 ((qh >> (j + 12))     ) & 0x10
        // 使用无符号右移操作符将 vqh 和 vt_2 中的每个元素进行逻辑右移，结果存储在 xhr_1 中
        vuint32m2_t xhr_1 = __riscv_vsrl_vv_u32m2(vqh, vt_2, vl);
        // 使用按位与操作符将 xhr_1 和 0x10 中的每个元素进行按位与操作，结果存储在 xha_1 中
        vuint32m2_t xha_1 = __riscv_vand_vx_u32m2(xhr_1, 0x10, vl);

        // 将 xha_0 中的每个元素转换为 16 位有符号整数，结果存储在 xhc_0 中
        vuint16m1_t xhc_0 = __riscv_vncvt_x_x_w_u16m1(xha_0, vl);
        // 将 xhc_0 中的每个元素转换为 8 位有符号整数，结果存储在 xh_0 中
        vuint8mf2_t xh_0 = __riscv_vncvt_x_x_w_u8mf2(xhc_0, vl);

        // 将 xha_1 中的每个元素转换为 16 位有符号整数，结果存储在 xhc_1 中
        vuint16m1_t xhc_1 = __riscv_vncvt_x_x_w_u16m1(xha_1, vl);
        // 将 xhc_1 中的每个元素转换为 8 位有符号整数，结果存储在 xh_1 中
        vuint8mf2_t xh_1 = __riscv_vncvt_x_x_w_u8mf2(xhc_1, vl);

        // 从 x[i].qs 中加载 8 位无符号整数，结果存储在 tx 中
        vuint8mf2_t tx = __riscv_vle8_v_u8mf2(x[i].qs, vl);

        // 从 y[i].qs 中加载 8 位有符号整数，结果存储在 y0 中
        vint8mf2_t y0 = __riscv_vle8_v_i8mf2(y[i].qs, vl);
        // 从 y[i].qs+16 中加载 8 位有符号整数，结果存储在 y1 中
        vint8mf2_t y1 = __riscv_vle8_v_i8mf2(y[i].qs+16, vl);

        // 使用按位与操作符将 tx 和 0x0F 中的每个元素进行按位与操作，结果存储在 x_at 中
        vuint8mf2_t x_at = __riscv_vand_vx_u8mf2(tx, 0x0F, vl);
        // 使用无符号右移操作符将 tx 中的每个元素进行逻辑右移，结果存储在 x_lt 中
        vuint8mf2_t x_lt = __riscv_vsrl_vx_u8mf2(tx, 0x04, vl);

        // 使用按位或操作符将 x_at 和 xh_0 中的每个元素进行按位或操作，结果存储在 x_a 中
        vuint8mf2_t x_a = __riscv_vor_vv_u8mf2(x_at, xh_0, vl);
# 使用逻辑或操作符对两个向量进行按位或操作，结果存储在 x_l 中
vuint8mf2_t x_l = __riscv_vor_vv_u8mf2(x_lt, xh_1, vl);

# 将 x_a 中的无符号 8 位整数向量重新解释为有符号 8 位整数向量，结果存储在 v0 中
vint8mf2_t v0 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_a);
# 将 x_l 中的无符号 8 位整数向量重新解释为有符号 8 位整数向量，结果存储在 v1 中
vint8mf2_t v1 = __riscv_vreinterpret_v_u8mf2_i8mf2(x_l);

# 对 v0 和 y0 进行有符号 16 位整数向量乘法，结果存储在 vec_mul1 中
vint16m1_t vec_mul1 = __riscv_vwmul_vv_i16m1(v0, y0, vl);
# 对 v1 和 y1 进行有符号 16 位整数向量乘法，结果存储在 vec_mul2 中
vint16m1_t vec_mul2 = __riscv_vwmul_vv_i16m1(v1, y1, vl);

# 创建一个全为 0 的有符号 32 位整数向量，结果存储在 vec_zero 中
vint32m1_t vec_zero = __riscv_vmv_v_x_i32m1(0, vl);

# 对 vec_mul1 进行有符号 16 位整数向量累加求和，结果存储在 vs1 中
vint32m1_t vs1 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul1, vec_zero, vl);
# 对 vec_mul2 进行有符号 16 位整数向量累加求和，结果存储在 vs2 中
vint32m1_t vs2 = __riscv_vwredsum_vs_i16m1_i32m1(vec_mul2, vs1, vl);

# 将 vs2 中的有符号 32 位整数向量转换为标量整数，结果存储在 sumi 中
int sumi = __riscv_vmv_x_s_i32m1_i32(vs2);

# 计算最终结果并将其加到 sumf 中
sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    # 定义一个浮点数变量 sumf，用于存储累加结果
    float sumf = 0.0;

    # 遍历 nb 次循环
    for (int i = 0; i < nb; i++) {
        # 定义一个 32 位无符号整数变量 qh，并从 x[i].qh 复制数据到 qh
        uint32_t qh;
        memcpy(&qh, x[i].qh, sizeof(qh));

        # 定义一个整数变量 sumi，用于存储内部累加结果
        int sumi = 0;

        # 遍历 qk/2 次循环
        for (int j = 0; j < qk/2; ++j) {
            # 计算 xh_0 和 xh_1 的值
            const uint8_t xh_0 = ((qh >> (j +  0)) << 4) & 0x10;
            const uint8_t xh_1 = ((qh >> (j + 12))     ) & 0x10;

            # 计算 x0 和 x1 的值
            const int32_t x0 = (x[i].qs[j] & 0xF) | xh_0;
            const int32_t x1 = (x[i].qs[j] >>  4) | xh_1;

            # 对 x0 和 x1 与 y[i].qs[j] 和 y[i].qs[j + qk/2] 的乘积进行累加
            sumi += (x0 * y[i].qs[j]) + (x1 * y[i].qs[j + qk/2]);
        }

        # 对 sumf 进行累加操作
        sumf += (GGML_FP16_TO_FP32(x[i].d)*y[i].d)*sumi + GGML_FP16_TO_FP32(x[i].m)*y[i].s;
    }
    }

    *s = sumf;
#endif
}

// 计算两个长度为n的向量的点积，结果存储在指针s所指向的位置
void ggml_vec_dot_q8_0_q8_0(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    const int qk = QK8_0; // 定义qk为QK8_0
    const int nb = n / qk; // 计算nb为n除以qk的结果

    assert(n % qk == 0); // 断言n能被qk整除

    const block_q8_0 * restrict x = vx; // 定义指针x指向vx
    const block_q8_0 * restrict y = vy; // 定义指针y指向vy

#if defined(__ARM_NEON)
    float32x4_t sumv0 = vdupq_n_f32(0.0f); // 使用0.0f初始化sumv0
    float32x4_t sumv1 = vdupq_n_f32(0.0f); // 使用0.0f初始化sumv1

    assert(nb % 2 == 0); // 断言nb是偶数，处理奇数nb的情况
// 循环遍历数组，每次增加2
for (int i = 0; i < nb; i += 2) {
    // 定义指向数组元素的指针
    const block_q8_0 * restrict x0 = &x[i + 0];
    const block_q8_0 * restrict x1 = &x[i + 1];
    const block_q8_0 * restrict y0 = &y[i + 0];
    const block_q8_0 * restrict y1 = &y[i + 1];

    // 从指针指向的位置加载数据到寄存器
    const int8x16_t x0_0 = vld1q_s8(x0->qs);
    const int8x16_t x0_1 = vld1q_s8(x0->qs + 16);
    const int8x16_t x1_0 = vld1q_s8(x1->qs);
    const int8x16_t x1_1 = vld1q_s8(x1->qs + 16);

    // 加载 y 数据到寄存器
    const int8x16_t y0_0 = vld1q_s8(y0->qs);
    const int8x16_t y0_1 = vld1q_s8(y0->qs + 16);
    const int8x16_t y1_0 = vld1q_s8(y1->qs);
    const int8x16_t y1_1 = vld1q_s8(y1->qs + 16);

    // 如果支持 ARM dot product 指令集
    #if defined(__ARM_FEATURE_DOTPROD)
        // 执行一些特定的操作
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
# 计算两个向量的点积，并将结果乘以一个常数，然后加到一个累加器中
sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(
                        vdotq_s32(vdupq_n_s32(0), x0_0, y0_0),
                        vdotq_s32(vdupq_n_s32(0), x0_1, y0_1))), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));

# 如果不支持 NEON 指令集，则进行下面的计算
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
        // 计算 p1_0 和 p1_1 的元素相加，并将结果转换为 32 位有符号整数
        const int32x4_t p2 = vaddq_s32(vpaddlq_s16(p1_0), vpaddlq_s16(p1_1));
        // 计算 p1_2 和 p1_3 的元素相加，并将结果转换为 32 位有符号整数
        const int32x4_t p3 = vaddq_s32(vpaddlq_s16(p1_2), vpaddlq_s16(p1_3));

        // 使用 p0 和 p1 的元素相加的结果，乘以 x0 和 y0 的浮点数值，累加到 sumv0 中
        sumv0 = vmlaq_n_f32(sumv0, vcvtq_f32_s32(vaddq_s32(p0, p1)), GGML_FP16_TO_FP32(x0->d)*GGML_FP16_TO_FP32(y0->d));
        // 使用 p2 和 p3 的元素相加的结果，乘以 x1 和 y1 的浮点数值，累加到 sumv1 中
        sumv1 = vmlaq_n_f32(sumv1, vcvtq_f32_s32(vaddq_s32(p2, p3)), GGML_FP16_TO_FP32(x1->d)*GGML_FP16_TO_FP32(y1->d));
#endif
    }

    // 将 sumv0 和 sumv1 的元素相加，并存储到指针 s 指向的位置
    *s = vaddvq_f32(sumv0) + vaddvq_f32(sumv1);
#elif defined(__AVX2__) || defined(__AVX__)
    // 初始化累加器为零
    __m256 acc = _mm256_setzero_ps();

    // 主循环
    for (int i = 0; i < nb; ++i) {
        // 计算块的组合比例
        const __m256 d = _mm256_set1_ps(GGML_FP16_TO_FP32(x[i].d) * GGML_FP16_TO_FP32(y[i].d));
        // 加载 x[i].qs 到 bx
        __m256i bx = _mm256_loadu_si256((const __m256i *)x[i].qs);
        // 加载 y[i].qs 到 by
        __m256i by = _mm256_loadu_si256((const __m256i *)y[i].qs);
        // 计算 bx 和 by 的乘积并求和
        const __m256 q = mul_sum_i8_pairs_float(bx, by);

        // 用 scale 值乘以 q 并累加到 acc 中
#if defined(__AVX2__)
        // 如果支持 AVX2 指令集，则使用 FMA 指令进行乘加运算
        acc = _mm256_fmadd_ps( d, q, acc );
#else
        // 如果不支持 AVX2 指令集，则使用乘法和加法指令进行乘加运算
        acc = _mm256_add_ps( _mm256_mul_ps( d, q ), acc );
#endif
    }

    // 将累加结果求和并存储到 s 中
    *s = hsum_float_8(acc);
#elif defined(__riscv_v_intrinsic)
    // 初始化浮点数求和结果为 0
    float sumf = 0.0;
    // 获取当前向量长度
    size_t vl = __riscv_vsetvl_e8m1(qk);

    for (int i = 0; i < nb; i++) {
        // 加载向量元素
        vint8m1_t bx = __riscv_vle8_v_i8m1(x[i].qs, vl);
        vint8m1_t by = __riscv_vle8_v_i8m1(y[i].qs, vl);
        // 使用 RISC-V 指令进行 int16 类型的向量乘法
        vint16m2_t vw_mul = __riscv_vwmul_vv_i16m2(bx, by, vl);

        // 创建一个全零的 int32 类型向量
        vint32m1_t v_zero = __riscv_vmv_v_x_i32m1(0, vl);
        // 对 int16 类型向量进行求和并转换为 int32 类型
        vint32m1_t v_sum = __riscv_vwredsum_vs_i16m2_i32m1(vw_mul, v_zero, vl);

        // 将 int32 类型向量转换为标量 int 类型
        int sumi = __riscv_vmv_x_s_i32m1_i32(v_sum);

        // 计算浮点数的累加乘积
        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    // 如果不支持向量化指令，则使用标量计算
    *s = sumf;
#else
    // scalar
    // 初始化浮点数累加和
    float sumf = 0.0;

    // 遍历数组进行标量计算
    for (int i = 0; i < nb; i++) {
        // 初始化整数累加和
        int sumi = 0;

        // 遍历数组进行标量乘法和累加
        for (int j = 0; j < qk; j++) {
            sumi += x[i].qs[j]*y[i].qs[j];
    }

    // 计算向量 x 和 y 的点积，结果存储在 s 中
    sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
#endif
}

#if QK_K == 256
// 计算两个向量的点积，其中向量 x 是 q2_K 类型，向量 y 是 q8_K 类型
void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {

    // 将输入的向量转换为特定类型的指针
    const block_q2_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度的块数
    const int nb = n / QK_K;

#ifdef __ARM_NEON

    // 创建一个包含 16 个相同值的向量
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    // 创建一个包含16个8位无符号整数的向量，每个元素都是0xF
    const uint8x16_t m4 = vdupq_n_u8(0xF);
    
    // 如果支持ARM dot product指令集，则创建一个包含4个32位有符号整数的向量，每个元素都是0
    #if defined(__ARM_FEATURE_DOTPROD)
        const int32x4_t  vzero = vdupq_n_s32(0);
    #endif

    // 创建一个包含两个16位有符号整数的向量结构体
    ggml_int8x16x2_t q2bytes;
    // 创建一个包含16个8位无符号整数的数组
    uint8_t aux[16];

    // 创建一个浮点数变量，用于存储累加和
    float sum = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 计算d的值，y[i].d乘以x[i].d的GGML_FP16_TO_FP32转换结果
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算dmin的值，-y[i].d乘以x[i].dmin的GGML_FP16_TO_FP32转换结果
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 创建指向x[i].qs的无别名限定符的指针
        const uint8_t * restrict q2 = x[i].qs;
        // 创建指向y[i].qs的有别名限定符的指针
        const int8_t  * restrict q8 = y[i].qs;
        // 创建指向x[i].scales的无别名限定符的指针
        const uint8_t * restrict sc = x[i].scales;

        // 从sc指向的地址加载16个8位无符号整数，存储到mins_and_scales向量中
        const uint8x16_t mins_and_scales = vld1q_u8(sc);
        // 使用vandq_u8函数将mins_and_scales和m4进行按位与操作，并将结果存储到scales中
        const uint8x16_t scales = vandq_u8(mins_and_scales, m4);
        // 将scales中的值存储到aux数组中
        vst1q_u8(aux, scales);

        // 使用vshrq_n_u8函数将mins_and_scales右移4位，并将结果存储到mins中
        const uint8x16_t mins = vshrq_n_u8(mins_and_scales, 4);
        // 从y[i].bsums中加载数据到q8sums中
        const ggml_int16x8x2_t q8sums = ggml_vld1q_s16_x2(y[i].bsums);
        // 将mins转换为16位有符号整数，并存储到mins16中
        const ggml_int16x8x2_t mins16 = {vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(mins))), vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(mins)))};
        // 计算s0和s1的值
        const int32x4_t s0 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[0]), vget_low_s16 (q8sums.val[0])),
                                       vmull_s16(vget_high_s16(mins16.val[0]), vget_high_s16(q8sums.val[0])));
        const int32x4_t s1 = vaddq_s32(vmull_s16(vget_low_s16 (mins16.val[1]), vget_low_s16 (q8sums.val[1])),
                                       vmull_s16(vget_high_s16(mins16.val[1]), vget_high_s16(q8sums.val[1])));
        // 将s0和s1的值相加，并乘以dmin，然后将结果累加到sum中
        sum += dmin * vaddvq_s32(vaddq_s32(s0, s1));

        // 初始化isum和is的值
        int isum = 0;
        int is = 0;

        // 我们使用这个宏而不是函数调用，因为由于某种原因，即使函数声明为内联，代码运行速度也会慢2-3%
#if defined(__ARM_FEATURE_DOTPROD)
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        // 将vzero、q2bytes.val[0]和q8bytes.val[0]进行点积运算，并将结果乘以aux[is+(index)]，然后累加到isum中
        isum += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * aux[is+(index)];\
// 定义一个宏，用于将两个向量的乘积累加到累加器中，并乘以一个比例因子
#define MULTIPLY_ACCUM_WITH_SCALE(index)\
        {\
    // 计算两个向量的乘积并加到累加器中
    const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0])));\
    const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[1]), vget_low_s8 (q8bytes.val[1])),\
                                   vmull_s8(vget_high_s8(q2bytes.val[1]), vget_high_s8(q8bytes.val[1])));\
    // 将乘积累加到累加器中，并乘以一个比例因子
    isum += vaddvq_s16(p1) * aux[is+(index)] + vaddvq_s16(p2) * aux[is+1+(index)];\
        }
// 定义一个宏，用于将两个向量的乘积累加到累加器中，并乘以一个比例因子，并进行位移操作
#define SHIFT_MULTIPLY_ACCUM_WITH_SCALE(shift, index)\
        q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;\
        // 对两个向量进行位移和按位与操作
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[0], (shift)), m3));\
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits.val[1], (shift)), m3));\
        // 调用前面定义的宏，将乘积累加到累加器中，并乘以一个比例因子
        MULTIPLY_ACCUM_WITH_SCALE((index));

// 循环遍历QK_K/128次
for (int j = 0; j < QK_K/128; ++j) {
// 从内存中加载两个 128 位的无符号 8 位整数向量到 q 寄存器中，并且每次加载后移动指针 32 个字节
const ggml_uint8x16x2_t q2bits = ggml_vld1q_u8_x2(q2); q2 += 32;

// 从内存中加载两个 128 位的有符号 8 位整数向量到 q 寄存器中，并且每次加载后移动指针 32 个字节
ggml_int8x16x2_t q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;

// 将 q2bits 中的数据与 m3 进行按位与操作，并将结果转换为有符号 8 位整数向量存储到 q2bytes 中
q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[0], m3));
q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(q2bits.val[1], m3));

// 使用比例因子进行乘法累加操作
MULTIPLY_ACCUM_WITH_SCALE(0);

// 对数据进行移位后使用比例因子进行乘法累加操作
SHIFT_MULTIPLY_ACCUM_WITH_SCALE(2, 2);

// 对数据进行移位后使用比例因子进行乘法累加操作
SHIFT_MULTIPLY_ACCUM_WITH_SCALE(4, 4);

// 对数据进行移位后使用比例因子进行乘法累加操作
SHIFT_MULTIPLY_ACCUM_WITH_SCALE(6, 6);

// 增加指针的偏移量
is += 8;
}
// 将 d 乘以 isum 后累加到 sum 中
sum += d * isum;
    *s = sum;
    // 如果定义了 __AVX2__，则执行以下代码
#elif defined __AVX2__
    // 创建一个包含 3 的 256 位整数向量
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含 0xF 的 128 位整数向量
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();
    // 循环处理每个元素
    for (int i = 0; i < nb; ++i) {
        // 计算 d 和 dmin
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
        // 获取指向 x[i].qs 和 y[i].qs 的指针
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;
        // 加载 x[i].scales 中的数据到整数向量 mins_and_scales
        const __m128i mins_and_scales = _mm_loadu_si128((const __m128i*)x[i].scales);
        // 将 mins_and_scales 中的数据按位与 m4，得到 scales8
        const __m128i scales8 = _mm_and_si128(mins_and_scales, m4);
        // 将 mins_and_scales 中的数据右移 4 位，再按位与 m4，得到 mins8
        const __m128i mins8 = _mm_and_si128(_mm_srli_epi16(mins_and_scales, 4), m4);
        # 将8位整数mins8转换为16位整数mins
        const __m256i mins = _mm256_cvtepi8_epi16(mins8);
        # 计算mins和y[i].bsums的乘积
        const __m256i prod = _mm256_madd_epi16(mins, _mm256_loadu_si256((const __m256i*)y[i].bsums));

        # 使用dmin对prod进行加权累加到acc中
        acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&dmin), _mm256_cvtepi32_ps(prod), acc);

        # 将8位整数scales8转换为16位整数all_scales
        const __m256i all_scales = _mm256_cvtepi8_epi16(scales8);
        # 从all_scales中提取低128位整数到l_scales
        const __m128i l_scales = _mm256_extracti128_si256(all_scales, 0);
        # 从all_scales中提取高128位整数到h_scales
        const __m128i h_scales = _mm256_extracti128_si256(all_scales, 1);
        # 将l_scales和h_scales组成两个__m128i类型的数组scales
        const __m256i scales[2] = {MM256_SET_M128I(l_scales, l_scales), MM256_SET_M128I(h_scales, h_scales)};

        # 初始化sumi为全0的256位整数
        __m256i sumi = _mm256_setzero_si256();

        # 循环遍历QK_K/128次
        for (int j = 0; j < QK_K/128; ++j) {
            # 从内存中加载32字节数据到q2bits
            const __m256i q2bits = _mm256_loadu_si256((const __m256i*)q2); q2 += 32;

            # 从内存中加载32字节数据到q8_0, q8_1, q8_2, q8_3
            const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
            const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
            const __m256i q8_2 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
            const __m256i q8_3 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
// 使用位与运算将q2bits中的低2位提取出来
const __m256i q2_0 = _mm256_and_si256(q2bits, m3);
const __m256i q2_1 = _mm256_and_si256(_mm256_srli_epi16(q2bits, 2), m3);
const __m256i q2_2 = _mm256_and_si256(_mm256_srli_epi16(q2bits, 4), m3);
const __m256i q2_3 = _mm256_and_si256(_mm256_srli_epi16(q2bits, 6), m3);

// 使用乘法和加法操作计算p0、p1、p2、p3
__m256i p0 = _mm256_maddubs_epi16(q2_0, q8_0);
__m256i p1 = _mm256_maddubs_epi16(q2_1, q8_1);
__m256i p2 = _mm256_maddubs_epi16(q2_2, q8_2);
__m256i p3 = _mm256_maddubs_epi16(q2_3, q8_3);

// 使用乘法和加法操作计算p0、p1、p2、p3
p0 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(0)), p0);
p1 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(1)), p1);
p2 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(2)), p2);
p3 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(3)), p3);

// 将p0和p1、p2和p3分别相加
p0 = _mm256_add_epi32(p0, p1);
p2 = _mm256_add_epi32(p2, p3);

// 将p0和p2相加，并加到sumi中
sumi = _mm256_add_epi32(sumi, _mm256_add_epi32(p0, p2));
    }

    // 使用 AVX 指令集进行计算
    acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi), acc);

}

// 如果定义了 AVX 指令集
#elif defined __AVX__

// 创建一些常量
const __m128i m3 = _mm_set1_epi8(0x3);
const __m128i m4 = _mm_set1_epi8(0xF);
const __m128i m2 = _mm_set1_epi8(0x2);

// 初始化一个 256 位的浮点数向量为 0
__m256 acc = _mm256_setzero_ps();

// 遍历数组进行计算
for (int i = 0; i < nb; ++i) {

    // 计算一些浮点数
    const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
    const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
// 从数组 x[i].qs 中加载数据到指针 q2
const uint8_t * restrict q2 = x[i].qs;
// 从数组 y[i].qs 中加载数据到指针 q8
const int8_t  * restrict q8 = y[i].qs;

// 从 x[i].scales 中加载 mins 和 scales 到 __m128i 类型的变量 mins_and_scales
const __m128i mins_and_scales = _mm_loadu_si128((const __m128i*)x[i].scales;
// 从 mins_and_scales 中提取 scales 的低 16 位到 scales16
const __m128i scales16 = _mm_and_si128(mins_and_scales, m4);
// 从 mins_and_scales 中提取 mins 的低 16 位到 mins16
const __m128i mins16 = _mm_and_si128(_mm_srli_epi16(mins_and_scales, 4), m4);
// 将 mins16 转换成 16 位有符号整数 mins_0 和 mins_1
const __m128i mins_0 = _mm_cvtepi8_epi16(mins16);
const __m128i mins_1 = _mm_cvtepi8_epi16(_mm_unpackhi_epi64(mins16, mins16));

// 计算 summs_0 和 summs_1，将 y[i].bsums 与 (x[i].scales >> 4) 相乘
const __m128i summs_0 = _mm_madd_epi16(mins_0, _mm_loadu_si128((const __m128i*)&y[i].bsums[0]));
const __m128i summs_1 = _mm_madd_epi16(mins_1, _mm_loadu_si128((const __m128i*)&y[i].bsums[8]));

// 将 acc 与 -dmin * summs 相加
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&dmin), _mm256_cvtepi32_ps(MM256_SET_M128I(summs_1, summs_0))), acc);

// 将 scales16 转换成 16 位有符号整数 scales_0 和 scales_1
const __m128i scales_0 = _mm_cvtepi8_epi16(scales16);
const __m128i scales_1 = _mm_cvtepi8_epi16(_mm_unpackhi_epi64(scales16, scales16));
        // 定义两个__m128i类型的变量scales，用于存储scales_0和scales_1
        const __m128i scales[2] = { scales_0, scales_1 };

        // 初始化两个__m128i类型的变量sumi_0和sumi_1，值为0
        __m128i sumi_0 = _mm_setzero_si128();
        __m128i sumi_1 = _mm_setzero_si128();

        // 循环遍历QK_K/128次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从block_q8_K.qs[QK_K]中加载8个int8*16*8到q8中
            const __m128i q8_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_2 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_3 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_4 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_5 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_6 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
            const __m128i q8_7 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;

            // 从block_q2_K.qs[QK_K/4]中加载16个2bits*16*8到q2bits中
            __m128i q2bits = _mm_loadu_si128((const __m128i*)q2); q2 += 16;
            // 将q2bits与m3进行按位与操作，结果存储到q2_0中
            const __m128i q2_0 = _mm_and_si128(q2bits, m3);
// 将 q2bits 右移 2 位并与 m3 进行按位与操作，得到 q2_2
const __m128i q2_2 = _mm_and_si128(_mm_srli_epi16(q2bits, 2), m3);
// 将 q2bits 右移 4 位并与 m3 进行按位与操作，得到 q2_4
const __m128i q2_4 = _mm_and_si128(_mm_srli_epi16(q2bits, 4), m3);
// 将 q2bits 右移 6 位并与 m3 进行按位与操作，得到 q2_6
const __m128i q2_6 = _mm_and_si128(_mm_srli_epi16(q2bits, 6), m3);
// 从内存中加载 q2bits 的值，然后移动指针
q2bits = _mm_loadu_si128((const __m128i*)q2); q2 += 16;
// 将 q2bits 与 m3 进行按位与操作，得到 q2_1
const __m128i q2_1 = _mm_and_si128(q2bits, m3);
// 将 q2bits 右移 2 位并与 m3 进行按位与操作，得到 q2_3
const __m128i q2_3 = _mm_and_si128(_mm_srli_epi16(q2bits, 2), m3);
// 将 q2bits 右移 4 位并与 m3 进行按位与操作，得到 q2_5
const __m128i q2_5 = _mm_and_si128(_mm_srli_epi16(q2bits, 4), m3);
// 将 q2bits 右移 6 位并与 m3 进行按位与操作，得到 q2_7
const __m128i q2_7 = _mm_and_si128(_mm_srli_epi16(q2bits, 6), m3);

// 计算 q8 与 q2 的乘积，并将结果存储在 p0 到 p7 中
__m128i p0 = _mm_maddubs_epi16(q2_0, q8_0);
__m128i p1 = _mm_maddubs_epi16(q2_1, q8_1);
__m128i p2 = _mm_maddubs_epi16(q2_2, q8_2);
__m128i p3 = _mm_maddubs_epi16(q2_3, q8_3);
__m128i p4 = _mm_maddubs_epi16(q2_4, q8_4);
__m128i p5 = _mm_maddubs_epi16(q2_5, q8_5);
__m128i p6 = _mm_maddubs_epi16(q2_6, q8_6);
__m128i p7 = _mm_maddubs_epi16(q2_7, q8_7);

// 计算 x[i].scales[is++] & 0xF 与 isuml 的乘积，并将结果累加到 isum 中
# 创建一个用于按位移动的常量向量
__m128i shuffle = _mm_set1_epi16(0x0100);
# 使用常量向量对scales[j]进行按位移动和乘法累加，结果存储到p0中
p0 = _mm_madd_epi16(_mm_shuffle_epi8(scales[j], shuffle), p0);
# 更新常量向量
shuffle = _mm_add_epi16(shuffle, m2);
# 重复上述过程，对p1-p7进行相同的操作
...
# 将p0和p1的结果相加
p0 = _mm_add_epi32(p0, p1);
# 重复上述过程，对p2-p5进行相同的操作
...
// 使用 SIMD 指令对 p6 和 p7 进行 32 位整数相加
p6 = _mm_add_epi32(p6, p7);

// 计算 isum，将 p0 和 p2 相加，将 p4 和 p6 相加，结果存储在 sumi_0 和 sumi_1 中
sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p0, p2));
sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p4, p6));

// 计算 sumf，使用 SIMD 指令进行一系列乘法和加法操作
__m256i sumi = MM256_SET_M128I(sumi_1, sumi_0);
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&dall), _mm256_cvtepi32_ps(sumi)), acc);

// 如果定义了 __riscv_v_intrinsic，则执行以下代码
float sumf = 0;
uint8_t temp_01[32] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1};
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 获取 x[i] 的 qs 值
        const uint8_t * q2 = x[i].qs;
        // 获取 y[i] 的 qs 值
        const  int8_t * q8 = y[i].qs;
        // 获取 x[i] 的 scales 值
        const uint8_t * sc = x[i].scales;

        // 计算 dall 值
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 dmin 值
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 设置向量长度为 16
        size_t vl = 16;

        // 从内存中加载 scales 到向量寄存器
        vuint8m1_t scales = __riscv_vle8_v_u8m1(sc, vl);
        // 对 scales 向量进行按位与操作
        vuint8m1_t aux = __riscv_vand_vx_u8m1(scales, 0x0F, vl);

        // 从内存中加载 y[i].bsums 到向量寄存器
        vint16m1_t q8sums = __riscv_vle16_v_i16m1(y[i].bsums, vl);

        // 从内存中加载 scales 到向量寄存器
        vuint8mf2_t scales_2 = __riscv_vle8_v_u8mf2(sc, vl);
        // 对 scales_2 向量进行逻辑右移操作
        vuint8mf2_t mins8 = __riscv_vsrl_vx_u8mf2(scales_2, 0x4, vl);
        // 将 mins8 向量进行零扩展和类型转换
        vint16m1_t mins = __riscv_vreinterpret_v_u16m1_i16m1(__riscv_vzext_vf2_u16m1(mins8, vl));
        // 对 q8sums 和 mins 向量进行乘法操作
        vint32m2_t prod = __riscv_vwmul_vv_i32m2(q8sums, mins, vl);
        # 使用向量指令对一组整数进行求和，并将结果存储在向量寄存器中
        vint32m1_t vsums = __riscv_vredsum_vs_i32m2_i32m1(prod, __riscv_vmv_v_x_i32m1(0, 1), vl);

        # 将求和结果乘以 dmin，并加到 sumf 中
        sumf  += dmin * __riscv_vmv_x_s_i32m1_i32(vsums);

        # 设置向量长度为 32
        vl = 32;

        # 创建一个值为 0 的向量
        vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);
        # 从内存中加载一组无符号 8 位整数到向量寄存器中
        vuint8m1_t v_b = __riscv_vle8_v_u8m1(temp_01, vl);

        # 初始化变量 is 和 isum
        uint8_t is=0;
        int isum=0;

        # 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {
            # 从内存中加载一组无符号 8 位整数到向量寄存器中
            vuint8m1_t q2_x = __riscv_vle8_v_u8m1(q2, vl);

            # 对加载的向量进行位与运算，并将结果存储在新的向量中
            vuint8m1_t q2_0 = __riscv_vand_vx_u8m1(q2_x, 0x03, vl);
            vuint8m1_t q2_1 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q2_x, 0x2, vl), 0x03 , vl);
            vuint8m1_t q2_2 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q2_x, 0x4, vl), 0x03 , vl);
            vuint8m1_t q2_3 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q2_x, 0x6, vl), 0x03 , vl);
// 为产品复制比例元素
vuint8m1_t sc0 = __riscv_vrgather_vv_u8m1(aux, __riscv_vadd_vx_u8m1(v_b, 0+is, vl), vl);
vuint8m1_t sc1 = __riscv_vrgather_vv_u8m1(aux, __riscv_vadd_vx_u8m1(v_b, 2+is, vl), vl);
vuint8m1_t sc2 = __riscv_vrgather_vv_u8m1(aux, __riscv_vadd_vx_u8m1(v_b, 4+is, vl), vl);
vuint8m1_t sc3 = __riscv_vrgather_vv_u8m1(aux, __riscv_vadd_vx_u8m1(v_b, 6+is, vl), vl);

vint16m2_t p0 = __riscv_vreinterpret_v_u16m2_i16m2(__riscv_vwmulu_vv_u16m2(q2_0, sc0, vl));
vint16m2_t p1 = __riscv_vreinterpret_v_u16m2_i16m2(__riscv_vwmulu_vv_u16m2(q2_1, sc1, vl));
vint16m2_t p2 = __riscv_vreinterpret_v_u16m2_i16m2(__riscv_vwmulu_vv_u16m2(q2_2, sc2, vl));
vint16m2_t p3 = __riscv_vreinterpret_v_u16m2_i16m2(__riscv_vwmulu_vv_u16m2(q2_3, sc3, vl);

// 加载 Q8
vint8m1_t q8_0 = __riscv_vle8_v_i8m1(q8, vl);
vint8m1_t q8_1 = __riscv_vle8_v_i8m1(q8+32, vl);
vint8m1_t q8_2 = __riscv_vle8_v_i8m1(q8+64, vl);
vint8m1_t q8_3 = __riscv_vle8_v_i8m1(q8+96, vl);

vint32m4_t s0 = __riscv_vwmul_vv_i32m4(p0, __riscv_vwcvt_x_x_v_i16m2(q8_0, vl), vl);
vint32m4_t s1 = __riscv_vwmul_vv_i32m4(p1, __riscv_vwcvt_x_x_v_i16m2(q8_1, vl), vl);
# 使用向量乘法计算 p2 和 q8_2 的结果，并存储在 s2 中
vint32m4_t s2 = __riscv_vwmul_vv_i32m4(p2, __riscv_vwcvt_x_x_v_i16m2(q8_2, vl), vl);
# 使用向量乘法计算 p3 和 q8_3 的结果，并存储在 s3 中
vint32m4_t s3 = __riscv_vwmul_vv_i32m4(p3, __riscv_vwcvt_x_x_v_i16m2(q8_3, vl), vl);

# 对 s0 和 s1 的结果进行向量加法，并进行向量求和，存储在 isum0 中
vint32m1_t isum0 = __riscv_vredsum_vs_i32m4_i32m1(__riscv_vadd_vv_i32m4(s0, s1, vl), vzero, vl);
# 对 s2 和 s3 的结果进行向量加法，并进行向量求和，加上上一步的结果 isum0，存储在 isum1 中
vint32m1_t isum1 = __riscv_vredsum_vs_i32m4_i32m1(__riscv_vadd_vv_i32m4(s2, s3, vl), isum0, vl);

# 将 isum1 的值加到 isum 上
isum += __riscv_vmv_x_s_i32m1_i32(isum1);

# 更新 q2、q8 和 is 的值
q2+=32;  q8+=128;  is=8;

# 计算 sumf 的值，将其加到原来的 sumf 上
sumf += dall * isum;

# 将最终的 sumf 存储在指针 s 指向的位置
*s = sumf;
    # 初始化浮点数求和变量
    float sumf = 0;

    # 遍历 nb 次，nb 为一个整数变量
    for (int i = 0; i < nb; ++i) {

        # 获取 x[i] 的 qs 属性，存入 q2 变量
        const uint8_t * q2 = x[i].qs;
        # 获取 y[i] 的 qs 属性，存入 q8 变量
        const  int8_t * q8 = y[i].qs;
        # 获取 x[i] 的 scales 属性，存入 sc 变量
        const uint8_t * sc = x[i].scales;

        # 初始化整数求和变量
        int summs = 0;
        # 遍历 16 次，计算 y[i].bsums[j] 与 sc[j] >> 4 的乘积并累加到 summs 变量
        for (int j = 0; j < 16; ++j) {
            summs += y[i].bsums[j] * (sc[j] >> 4);
        }

        # 计算 dall 变量的值
        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        # 计算 dmin 变量的值
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        # 初始化整数变量
        int isum = 0;
        int is = 0;
        int d;
        # 遍历 QK_K/128 次，QK_K 为一个常量，计算一些值
        for (int k = 0; k < QK_K/128; ++k) {
// 初始化一个整型变量 shift 为 0
int shift = 0;
// 循环4次，每次增加1
for (int j = 0; j < 4; ++j) {
    // 从数组 sc 中取出一个元素，并与 0xF 进行按位与操作
    d = sc[is++] & 0xF;
    // 初始化一个整型变量 isuml 为 0
    int isuml = 0;
    // 循环16次，每次增加1
    for (int l =  0; l < 16; ++l) 
        // 计算 isuml 的值
        isuml += q8[l] * ((q2[l] >> shift) & 3);
    // 计算 isum 的值
    isum += d * isuml;
    // 从数组 sc 中取出一个元素，并与 0xF 进行按位与操作
    d = sc[is++] & 0xF;
    // 重新初始化 isuml 为 0
    isuml = 0;
    // 循环16次，每次增加1
    for (int l = 16; l < 32; ++l) 
        // 计算 isuml 的值
        isuml += q8[l] * ((q2[l] >> shift) & 3);
    // 计算 isum 的值
    isum += d * isuml;
    // shift 增加2
    shift += 2;
    // q8 指针向后移动32个位置
    q8 += 32;
}
// q2 指针向后移动32个位置
q2 += 32;
// 循环结束

// 计算 sumf 的值
sumf += dall * isum - dmin * summs;
// 结束 if 语句块
}

// 将 sumf 的值赋给指针 s 所指向的变量
*s = sumf;
// 结束条件编译指令块
#endif
}
// 定义一个函数，计算两个向量的点积
void ggml_vec_dot_q2_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 将输入的指针转换为特定类型的指针
    const block_q2_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以 QK_K 的商，用于后续循环
    const int nb = n / QK_K;

    // 如果是 ARM 平台，使用 NEON 指令集
#ifdef __ARM_NEON
    // 创建一个包含 16 个相同值的向量
    const uint8x16_t m3 = vdupq_n_u8(0x3);
    // 如果支持 ARM 的 dot product 指令集，创建一个包含 4 个相同值的向量
#if defined(__ARM_FEATURE_DOTPROD)
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 定义一个包含 4 个 int8x16_t 类型的结构体
    ggml_int8x16x4_t q2bytes;

    // 定义一个包含 2 个 32 位无符号整数的数组
    uint32_t aux32[2];
    // 将 aux32 强制转换为 uint8_t 类型的指针
    const uint8_t * scales = (const uint8_t *)aux32;

    // 初始化 sum 变量
    float sum = 0;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d 和 dmin
        const float d = y[i].d * (float)x[i].d;
        const float dmin = -y[i].d * (float)x[i].dmin;

        // 定义并初始化指向 x[i].qs 和 y[i].qs 的指针
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;
        // 将 x[i].scales 强制转换为 uint32_t 类型的指针
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;

        // 将 sc[0] 的值按位与 0x0f0f0f0f，然后存储到 aux32[0] 和 aux32[1] 中
        aux32[0] = sc[0] & 0x0f0f0f0f;
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算 sum 的值
        sum += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);

        // 初始化 isum1 和 isum2 变量
        int isum1 = 0, isum2 = 0;
        // 加载一个包含8个8位整数的寄存器
        const uint8x16_t q2bits = vld1q_u8(q2);

        // 加载4个包含16位有符号整数的寄存器
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 将q2bits寄存器中的数据按位与m3，然后转换为有符号8位整数，存入q2bytes.val[0]
        q2bytes.val[0] = vreinterpretq_s8_u8(vandq_u8(q2bits, m3));
        // 将q2bits寄存器中的数据右移2位，按位与m3，然后转换为有符号8位整数，存入q2bytes.val[1]
        q2bytes.val[1] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 2), m3));
        // 将q2bits寄存器中的数据右移4位，按位与m3，然后转换为有符号8位整数，存入q2bytes.val[2]
        q2bytes.val[2] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 4), m3));
        // 将q2bits寄存器中的数据右移6位，按位与m3，然后转换为有符号8位整数，存入q2bytes.val[3]
        q2bytes.val[3] = vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q2bits, 6), m3));

#if defined(__ARM_FEATURE_DOTPROD)
        // 使用向量点乘计算两个向量的内积，然后将结果乘以scales[0]并加到isum1上
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[0], q8bytes.val[0])) * scales[0];
        // 使用向量点乘计算两个向量的内积，然后将结果乘以scales[1]并加到isum2上
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[1], q8bytes.val[1])) * scales[1];
        // 使用向量点乘计算两个向量的内积，然后将结果乘以scales[2]并加到isum1上
        isum1 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[2], q8bytes.val[2])) * scales[2];
        // 使用向量点乘计算两个向量的内积，然后将结果乘以scales[3]并加到isum2上
        isum2 += vaddvq_s32(vdotq_s32(vzero, q2bytes.val[3], q8bytes.val[3])) * scales[3];
#else
        // 如果不支持向量点乘，则使用低位和高位分别进行乘法和加法，然后将结果乘以scales[0]并加到isum1上
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q2bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        isum1 += vaddvq_s16(p1) * scales[0];
        // 计算isum2的值，将p2的值乘以scales[1]并加到isum2上
        isum2 += vaddvq_s16(p2) * scales[1];

        // 计算p3的值，将q2bytes.val[2]和q8bytes.val[2]的低位相乘，将q2bytes.val[2]和q8bytes.val[2]的高位相乘，然后相加
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q2bytes.val[2]), vget_high_s8(q8bytes.val[2])));
        // 计算p4的值，将q2bytes.val[3]和q8bytes.val[3]的低位相乘，将q2bytes.val[3]和q8bytes.val[3]的高位相乘，然后相加
        const int16x8_t p4 = vaddq_s16(vmull_s8(vget_low_s8 (q2bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q2bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        // 将p3的值乘以scales[2]并加到isum1上
        isum1 += vaddvq_s16(p3) * scales[2];
        // 将p4的值乘以scales[3]并加到isum2上
        isum2 += vaddvq_s16(p4) * scales[3];
#endif
        // 将d乘以(isum1 + isum2)的和加到sum上
        sum += d * (isum1 + isum2);

    }

    *s = sum;

#elif defined __AVX2__

    // 将m3的值设置为全1的8位整数
    const __m256i m3 = _mm256_set1_epi8(3);

    // 将acc的值设置为全0的8个单精度浮点数
    __m256 acc = _mm256_setzero_ps();
    // 定义两个32位无符号整数变量ud和um
    uint32_t ud, um;
    // 将ud和um的地址强制转换为指向无符号8位整数的指针，并赋给db和mb
    const uint8_t * restrict db = (const uint8_t *)&ud;
    const uint8_t * restrict mb = (const uint8_t *)&um;

    // 定义一个浮点数变量summs，并初始化为0
    float summs = 0;

    // 循环遍历nb次，执行下面的操作
    // TODO: 优化这部分代码

    for (int i = 0; i < nb; ++i) {

        // 计算d和dmin的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 定义指向x[i].qs和y[i].qs的指针q2和q8
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 将x[i].scales的地址强制转换为指向32位无符号整数的指针，并赋给sc
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        // 将sc[0]右移0位并与0x0f0f0f0f进行按位与运算，结果赋给ud
        ud = (sc[0] >> 0) & 0x0f0f0f0f;
        // 将sc[0]右移4位并与0x0f0f0f0f进行按位与运算，结果赋给um
        um = (sc[0] >> 4) & 0x0f0f0f0f;
// 计算 smin 的值，使用 mb 和 y[i].bsums 的加权和
int32_t smin = mb[0] * y[i].bsums[0] + mb[1] * y[i].bsums[1] + mb[2] * y[i].bsums[2] + mb[3] * y[i].bsums[3];
// 计算 summs 的值，使用 dmin 和 smin 的乘积
summs += dmin * smin;

// 加载 q2 的数据到 __m128i 类型的变量 q2bits
const __m128i q2bits = _mm_loadu_si128((const __m128i*)q2);
// 对 q2bits 进行位运算，得到 q2_0 和 q2_1
const __m256i q2_0 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 2), q2bits), m3);
const __m256i q2_1 = _mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q2bits, 6), _mm_srli_epi16(q2bits, 4)), m3);

// 加载 q8 的数据到 __m256i 类型的变量 q8_0 和 q8_1
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

// 使用 q2_0 和 q8_0 计算 p0，使用 q2_1 和 q8_1 计算 p1
const __m256i p0 = _mm256_maddubs_epi16(q2_0, q8_0);
const __m256i p1 = _mm256_maddubs_epi16(q2_1, q8_1);

// 将 p0 和 p1 转换为 32 位整数类型
const __m256i p_0 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 0));
const __m256i p_1 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p0, 1));
const __m256i p_2 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 0));
const __m256i p_3 = _mm256_cvtepi16_epi32(_mm256_extracti128_si256(p1, 1));

// 使用 d * db[0] 和 p_0 计算 acc 的值
acc = _mm256_fmadd_ps(_mm256_set1_ps(d * db[0]), _mm256_cvtepi32_ps(p_0), acc);
        使用AVX指令集中的_mm256_fmadd_ps函数，将两个__m256类型的参数相乘，然后将结果与第三个__m256类型的参数相加，并将结果存储在acc中。
        使用AVX指令集中的_mm256_fmadd_ps函数，将两个__m256类型的参数相乘，然后将结果与第三个__m256类型的参数相加，并将结果存储在acc中。
        使用AVX指令集中的_mm256_fmadd_ps函数，将两个__m256类型的参数相乘，然后将结果与第三个__m256类型的参数相加，并将结果存储在acc中。
    }

    *s = hsum_float_8(acc) + summs;  // 将acc中的8个单精度浮点数相加，并加上summs，然后将结果存储在指针s所指向的位置

#elif defined __AVX__

    const __m128i m3 = _mm_set1_epi8(3);  // 创建一个__m128i类型的常量m3，其所有元素都是3

    __m256 acc = _mm256_setzero_ps();  // 创建一个__m256类型的变量acc，并将其所有元素初始化为0

    uint32_t ud, um;  // 声明两个32位无符号整数变量ud和um
    const uint8_t * restrict db = (const uint8_t *)&ud;  // 声明一个指向ud的指针db，并将其转换为const uint8_t类型
    const uint8_t * restrict mb = (const uint8_t *)&um;  // 声明一个指向um的指针mb，并将其转换为const uint8_t类型

    float summs = 0;  // 声明一个单精度浮点数变量summs，并初始化为0

    // TODO: optimize this  // 待优化的部分，可能需要进一步优化这部分代码
    // 遍历数组 y 的元素，计算 d 和 dmin
    for (int i = 0; i < nb; ++i) {
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 定义指向 x[i].qs 和 y[i].qs 的指针
        const uint8_t * restrict q2 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 定义指向 x[i].scales 的指针，并提取其中的值
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        ud = (sc[0] >> 0) & 0x0f0f0f0f;
        um = (sc[0] >> 4) & 0x0f0f0f0f;

        // 计算 smin，并累加到 summs 中
        int32_t smin = mb[0] * y[i].bsums[0] + mb[1] * y[i].bsums[1] + mb[2] * y[i].bsums[2] + mb[3] * y[i].bsums[3];
        summs += dmin * smin;

        // 加载 q2 的值到 __m128i 类型的变量中，并进行位运算
        const __m128i q2bits = _mm_loadu_si128((const __m128i*)q2);
        const __m128i q2_0 = _mm_and_si128(q2bits, m3);
        const __m128i q2_1 = _mm_and_si128(_mm_srli_epi16(q2bits, 2), m3);
        const __m128i q2_2 = _mm_and_si128(_mm_srli_epi16(q2bits, 4), m3);
    }
# 使用位移和与操作，将q2bits右移6位并与m3进行与操作，得到__m128i类型的结果q2_3
const __m128i q2_3 = _mm_and_si128(_mm_srli_epi16(q2bits, 6), m3);

# 从内存中加载两个__m256i类型的数据，分别为q8+0和q8+32
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

# 使用_mm_maddubs_epi16函数，将q2_0和q2_1与q8_0的低128位和高128位进行乘积累加，得到__m128i类型的结果p0和p1
const __m128i p0 = _mm_maddubs_epi16(q2_0, _mm256_extractf128_si256(q8_0, 0));
const __m128i p1 = _mm_maddubs_epi16(q2_1, _mm256_extractf128_si256(q8_0, 1));

# 使用_mm_maddubs_epi16函数，将q2_2和q2_3与q8_1的低128位和高128位进行乘积累加，得到__m128i类型的结果p2和p3
const __m128i p2 = _mm_maddubs_epi16(q2_2, _mm256_extractf128_si256(q8_1, 0));
const __m128i p3 = _mm_maddubs_epi16(q2_3, _mm256_extractf128_si256(q8_1, 1));

# 将p0、p1、p2、p3进行零拓展和转换成__m256i类型的结果p_0、p_1、p_2、p_3
const __m256i p_0 = MM256_SET_M128I(_mm_cvtepi16_epi32(_mm_unpackhi_epi64(p0, p0)), _mm_cvtepi16_epi32(p0));
const __m256i p_1 = MM256_SET_M128I(_mm_cvtepi16_epi32(_mm_unpackhi_epi64(p1, p1)), _mm_cvtepi16_epi32(p1));
const __m256i p_2 = MM256_SET_M128I(_mm_cvtepi16_epi32(_mm_unpackhi_epi64(p2, p2)), _mm_cvtepi16_epi32(p2));
const __m256i p_3 = MM256_SET_M128I(_mm_cvtepi16_epi32(_mm_unpackhi_epi64(p3, p3)), _mm_cvtepi16_epi32(p3));

# 使用_mm256_mul_ps和_mm256_add_ps函数，将p_0、p_1、p_2、p_3与d*db[i]进行乘法和累加操作，得到累加结果acc
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_set1_ps(d * db[0]), _mm256_cvtepi32_ps(p_0)), acc);
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_set1_ps(d * db[1]), _mm256_cvtepi32_ps(p_1)), acc);
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_set1_ps(d * db[2]), _mm256_cvtepi32_ps(p_2)), acc);
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_set1_ps(d * db[3]), _mm256_cvtepi32_ps(p_3)), acc);
    *s = hsum_float_8(acc) + summs;
    // 将acc中的8个浮点数相加，并加上summs，结果存入*s中

#elif defined __riscv_v_intrinsic
    // 如果定义了__riscv_v_intrinsic
    uint32_t aux32[2];
    // 定义一个32位无符号整数数组aux32，包含2个元素
    const uint8_t * scales = (const uint8_t *)aux32;
    // 定义一个指向aux32的无符号8位整数指针scales

    float sumf = 0;
    // 定义一个浮点数sumf，并初始化为0

    for (int i = 0; i < nb; ++i) {
        // 循环，i从0到nb-1
        const float d = y[i].d * (float)x[i].d;
        // 计算y[i].d和x[i].d的乘积，存入d
        const float dmin = -y[i].d * (float)x[i].dmin;
        // 计算-y[i].d和x[i].dmin的乘积，存入dmin

        const uint8_t * restrict q2 = x[i].qs;
        // 定义一个指向x[i].qs的限制指针q2
        const int8_t  * restrict q8 = y[i].qs;
        // 定义一个指向y[i].qs的限制指针q8
        const uint32_t * restrict sc = (const uint32_t *)x[i].scales;
        // 定义一个指向x[i].scales的限制指针sc

        aux32[0] = sc[0] & 0x0f0f0f0f;
        // 将sc[0]与0x0f0f0f0f进行按位与操作，结果存入aux32[0]
        # 将sc[0]右移4位，并与0x0f0f0f0f进行按位与操作，结果存入aux32[1]
        aux32[1] = (sc[0] >> 4) & 0x0f0f0f0f;

        # 计算sumf的值，包括dmin与后面一系列计算的乘积之和
        sumf += dmin * (scales[4] * y[i].bsums[0] + scales[5] * y[i].bsums[1] + scales[6] * y[i].bsums[2] + scales[7] * y[i].bsums[3]);

        # 初始化isum1和isum2为0
        int isum1 = 0;
        int isum2 = 0;

        # 初始化vl为16
        size_t vl = 16;

        # 将0赋值给vzero
        vint16m1_t vzero = __riscv_vmv_v_x_i16m1(0, 1);

        # 加载Q2
        vuint8mf2_t q2_x = __riscv_vle8_v_u8mf2(q2, vl);

        # 对Q2进行位操作，将结果存入q2_0、q2_1、q2_2、q2_3
        vint8mf2_t q2_0 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(q2_x, 0x03, vl));
        vint8mf2_t q2_1 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(q2_x, 0x2, vl), 0x03 , vl));
        vint8mf2_t q2_2 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(q2_x, 0x4, vl), 0x03 , vl));
        vint8mf2_t q2_3 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(q2_x, 0x6, vl), 0x03 , vl));

        # 加载Q8，并与Q2进行乘积运算
# 使用向量乘法计算 q2_0 和 q8 之间的乘积
vint16m1_t p0 = __riscv_vwmul_vv_i16m1(q2_0, __riscv_vle8_v_i8mf2(q8, vl), vl);
# 使用向量乘法计算 q2_1 和 q8+16 之间的乘积
vint16m1_t p1 = __riscv_vwmul_vv_i16m1(q2_1, __riscv_vle8_v_i8mf2(q8+16, vl), vl);
# 使用向量乘法计算 q2_2 和 q8+32 之间的乘积
vint16m1_t p2 = __riscv_vwmul_vv_i16m1(q2_2, __riscv_vle8_v_i8mf2(q8+32, vl), vl);
# 使用向量乘法计算 q2_3 和 q8+48 之间的乘积
vint16m1_t p3 = __riscv_vwmul_vv_i16m1(q2_3, __riscv_vle8_v_i8mf2(q8+48, vl), vl);

# 对 p0 进行向量求和
vint16m1_t vs_0 = __riscv_vredsum_vs_i16m1_i16m1(p0, vzero, vl);
# 对 p1 进行向量求和
vint16m1_t vs_1 = __riscv_vredsum_vs_i16m1_i16m1(p1, vzero, vl);
# 对 p2 进行向量求和
vint16m1_t vs_2 = __riscv_vredsum_vs_i16m1_i16m1(p2, vzero, vl);
# 对 p3 进行向量求和
vint16m1_t vs_3 = __riscv_vredsum_vs_i16m1_i16m1(p3, vzero, vl);

# 将 vs_0 乘以 scales[0] 并加到 isum1 上
isum1 += __riscv_vmv_x_s_i16m1_i16(vs_0) * scales[0];
# 将 vs_1 乘以 scales[1] 并加到 isum2 上
isum2 += __riscv_vmv_x_s_i16m1_i16(vs_1) * scales[1];
# 将 vs_2 乘以 scales[2] 并加到 isum1 上
isum1 += __riscv_vmv_x_s_i16m1_i16(vs_2) * scales[2];
# 将 vs_3 乘以 scales[3] 并加到 isum2 上
isum2 += __riscv_vmv_x_s_i16m1_i16(vs_3) * scales[3];

# 将 d 乘以 (isum1 + isum2) 并加到 sumf 上
sumf += d * (isum1 + isum2);

# 将 sumf 的值赋给指针 s 所指向的位置
*s = sumf;
#else
// 如果条件不满足，则执行以下代码

    float sumf = 0;
    // 定义一个浮点数变量 sumf，并初始化为 0

    int isum[4];
    // 定义一个包含 4 个整数的数组 isum

    for (int i = 0; i < nb; ++i) {
    // 循环，i 从 0 开始，小于 nb 时执行，每次循环后 i 自增 1

        const uint8_t * q2 = x[i].qs;
        // 定义一个指向 x[i].qs 的常量无符号 8 位整数指针 q2

        const  int8_t * q8 = y[i].qs;
        // 定义一个指向 y[i].qs 的常量有符号 8 位整数指针 q8

        const uint8_t * sc = x[i].scales;
        // 定义一个指向 x[i].scales 的常量无符号 8 位整数指针 sc

        int summs = 0;
        // 定义一个整数变量 summs，并初始化为 0
        for (int j = 0; j < QK_K/16; ++j) {
        // 循环，j 从 0 开始，小于 QK_K/16 时执行，每次循环后 j 自增 1
            summs += y[i].bsums[j] * (sc[j] >> 4);
            // 计算 summs 的值
        }

        const float dall = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 dall 的值

        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
        // 计算 dmin 的值
        // 初始化四个整型数组元素为0
        isum[0] = isum[1] = isum[2] = isum[3] = 0;
        // 遍历16次
        for (int l =  0; l < 16; ++l) {
            // 根据位运算和数组元素相乘，累加到isum数组对应位置
            isum[0] += q8[l+ 0] * ((q2[l] >> 0) & 3);
            isum[1] += q8[l+16] * ((q2[l] >> 2) & 3);
            isum[2] += q8[l+32] * ((q2[l] >> 4) & 3);
            isum[3] += q8[l+48] * ((q2[l] >> 6) & 3);
        }
        // 再次遍历4次
        for (int l = 0; l < 4; ++l) {
            // 将isum数组对应位置乘以sc数组对应位置的低4位
            isum[l] *= (sc[l] & 0xF);
        }
        // 计算sumf的值
        sumf += dall * (isum[0] + isum[1] + isum[2] + isum[3]) - dmin * summs;
    }
    // 将sumf的值赋给指针所指向的变量
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 定义一个函数，计算两个向量的点积
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    # 确保 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    # 定义两个掩码常量
    const uint32_t kmask1 = 0x03030303;
    const uint32_t kmask2 = 0x0f0f0f0f;

    # 定义指向输入数组的指针，并使用 restrict 限定符
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    # 计算输入数组的块数
    const int nb = n / QK_K;

    # 如果是 ARM 平台，执行以下代码
#ifdef __ARM_NEON

    # 定义辅助数组和临时数组
    uint32_t aux[3];
    uint32_t utmp[4];

    # 创建一个包含 16 个相同值的向量
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    # 如果支持 ARM Dot Product 指令集，定义一个全零的 32 位整数向量
#ifdef __ARM_FEATURE_DOTPROD
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif
    // 创建一个包含16个8位无符号整数1的向量
    const uint8x16_t m0 = vdupq_n_u8(1);
    // 将向量m0中的每个元素左移1位
    const uint8x16_t m1 = vshlq_n_u8(m0, 1);
    // 将向量m0中的每个元素左移2位
    const uint8x16_t m2 = vshlq_n_u8(m0, 2);
    // 将向量m0中的每个元素左移3位
    const uint8x16_t m3 = vshlq_n_u8(m0, 3);
    // 创建一个包含32的8位有符号整数
    const int8_t m32 = 32;

    // 创建一个包含4个16位8字节整数的结构体
    ggml_int8x16x4_t q3bytes;

    // 初始化一个浮点数sum为0
    float sum = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {
        // 计算y[i].d和x[i].d的乘积，并赋值给d
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 声明并初始化指向x[i].qs、x[i].hmask和y[i].qs的指针
        const uint8_t * restrict q3 = x[i].qs;
        const uint8_t * restrict qh = x[i].hmask;
        const int8_t  * restrict q8 = y[i].qs;

        // 从qh中加载两个16字节的8位无符号整数，并赋值给qhbits
        ggml_uint8x16x2_t qhbits = ggml_vld1q_u8_x2(qh);
    }
        // 定义一个包含四个 16 位无符号整数的向量
        ggml_uint8x16x4_t q3h;

        // 初始化一个整型变量 isum 为 0
        int32_t isum = 0;

        // 设置比例
        // 从 x[i].scales 复制 12 个字节到 aux
        memcpy(aux, x[i].scales, 12);
        // 根据 aux 中的数据设置 utmp 数组
        utmp[3] = ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4);
        utmp[2] = ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4);
        utmp[1] = (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4);
        utmp[0] = (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4);

        // 将 utmp 转换为 int8_t 类型的指针，并对每个元素减去 m32
        int8_t * scale = (int8_t *)utmp;
        for (int j = 0; j < 16; ++j) scale[j] -= m32;

        // 遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {
            // 从 q3 中加载两个 16 位无符号整数向量到 q3bits，然后 q3 指针向后移动 32 个字节
            const ggml_uint8x16x2_t q3bits = ggml_vld1q_u8_x2(q3); q3 += 32;
            // 从 q8 中加载四个 16 位有符号整数向量到 q8bytes_1，然后 q8 指针向后移动 64 个字节
            const ggml_int8x16x4_t q8bytes_1 = ggml_vld1q_s8_x4(q8); q8 += 64;
            // 从 q8 中加载四个 16 位有符号整数向量到 q8bytes_2，然后 q8 指针向后移动 64 个字节
            const ggml_int8x16x4_t q8bytes_2 = ggml_vld1q_s8_x4(q8); q8 += 64;
# 将 qhbits.val[0] 中的位从 m0 中清除，然后左移 2 位，存入 q3h.val[0]
q3h.val[0] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[0]), 2);
# 将 qhbits.val[1] 中的位从 m0 中清除，然后左移 2 位，存入 q3h.val[1]
q3h.val[1] = vshlq_n_u8(vbicq_u8(m0, qhbits.val[1]), 2);
# 将 qhbits.val[0] 中的位从 m1 中清除，然后左移 1 位，存入 q3h.val[2]
q3h.val[2] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[0]), 1);
# 将 qhbits.val[1] 中的位从 m1 中清除，然后左移 1 位，存入 q3h.val[3]
q3h.val[3] = vshlq_n_u8(vbicq_u8(m1, qhbits.val[1]), 1);

# 从 q3bits.val[0] 和 m3b 中取出位与运算的结果，然后减去 q3h.val[0]，存入 q3bytes.val[0]
q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[0], m3b)), vreinterpretq_s8_u8(q3h.val[0]));
# 从 q3bits.val[1] 和 m3b 中取出位与运算的结果，然后减去 q3h.val[1]，存入 q3bytes.val[1]
q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q3bits.val[1], m3b)), vreinterpretq_s8_u8(q3h.val[1]));
# 从 q3bits.val[0] 右移 2 位和 m3b 中取出位与运算的结果，然后减去 q3h.val[2]，存入 q3bytes.val[2]
q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 2), m3b)), vreinterpretq_s8_u8(q3h.val[2]));
# 从 q3bits.val[1] 右移 2 位和 m3b 中取出位与运算的结果，然后减去 q3h.val[3]，存入 q3bytes.val[3]
q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 2), m3b)), vreinterpretq_s8_u8(q3h.val[3));

# 如果支持 ARM 的 dot product 指令集
# 将 q3bytes.val[0] 和 q8bytes_1.val[0] 进行 dot product 运算，然后乘以 scale[0]，并累加到 isum 中
# 将 q3bytes.val[1] 和 q8bytes_1.val[1] 进行 dot product 运算，然后乘以 scale[1]，并累加到 isum 中
# 将 q3bytes.val[2] 和 q8bytes_1.val[2] 进行 dot product 运算，然后乘以 scale[2]，并累加到 isum 中
# 将 q3bytes.val[3] 和 q8bytes_1.val[3] 进行 dot product 运算，然后乘以 scale[3]，并累加到 isum 中
# 否则
# 将 q3bytes.val[0] 和 q8bytes_1.val[0] 进行乘法运算，然后累加到 p0 中
# 将 q3bytes.val[1] 和 q8bytes_1.val[1] 进行乘法运算，然后累加到 p1 中
// 计算两个 int16x8_t 类型的变量 p2 和 p3，分别为两组字节的乘积之和
int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_1.val[2])),
                         vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_1.val[2]));
int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_1.val[3])),
                         vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_1.val[3]));

// 将 p0 和 p1 的和乘以 scale[0]，p2 和 p3 的和乘以 scale[2]，然后将它们相加并加到 isum 上
isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
scale += 4;

// 对 q3h.val 数组进行一系列位运算操作
q3h.val[0] = vbicq_u8(m2, qhbits.val[0]);
q3h.val[1] = vbicq_u8(m2, qhbits.val[1]);
q3h.val[2] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[0]), 1);
q3h.val[3] = vshrq_n_u8(vbicq_u8(m3, qhbits.val[1]), 1);

// 对 q3bytes.val 数组进行一系列位运算操作
q3bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 4), m3b)), vreinterpretq_s8_u8(q3h.val[0]));
q3bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 4), m3b)), vreinterpretq_s8_u8(q3h.val[1]));
q3bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[0], 6), m3b)), vreinterpretq_s8_u8(q3h.val[2]));
q3bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(vshrq_n_u8(q3bits.val[1], 6), m3b)), vreinterpretq_s8_u8(q3h.val[3));

// 如果支持 ARM dot product 指令集，则进行一系列乘法和加法操作，并将结果加到 isum 上
isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes_2.val[0])) * scale[0];
# 对isum进行累加，使用vdotq_s32计算两个向量的点积，再使用vaddvq_s32将结果向量的所有元素相加，最后乘以scale[1]
isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes_2.val[1])) * scale[1];
# 对isum进行累加，使用vdotq_s32计算两个向量的点积，再使用vaddvq_s32将结果向量的所有元素相加，最后乘以scale[2]
isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes_2.val[2])) * scale[2];
# 对isum进行累加，使用vdotq_s32计算两个向量的点积，再使用vaddvq_s32将结果向量的所有元素相加，最后乘以scale[3]
isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes_2.val[3])) * scale[3];
# 如果不满足条件，则执行以下代码
#else
    # 使用vmull_s8将两个int8x8_t类型的向量进行逐元素相乘，再使用vaddq_s16将结果向量的对应元素相加
    p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes_2.val[0])),
                   vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes_2.val[0])));
    p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes_2.val[1])),
                   vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes_2.val[1])));
    p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes_2.val[2])),
                   vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes_2.val[2])));
    p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes_2.val[3])),
                   vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes_2.val[3])));
    # 对p0、p1、p2、p3进行累加，乘以对应的scale值，最后累加到isum上
    isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1] + vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];
#endif
# scale指针向后移动4个位置
scale += 4;
# 如果j等于0，则执行以下代码
if (j == 0) {
    # 对qhbits.val[0]和qhbits.val[1]中的每个元素进行逻辑右移4位操作
    qhbits.val[0] = vshrq_n_u8(qhbits.val[0], 4);
    qhbits.val[1] = vshrq_n_u8(qhbits.val[1], 4);
}
        }
        sum += d * isum;

    }

    *s = sum;  // 将计算得到的总和赋值给指针所指向的变量

#elif defined __AVX2__  // 如果定义了 AVX2

    const __m256i m3 = _mm256_set1_epi8(3);  // 创建一个包含 3 的 AVX2 寄存器
    const __m256i mone = _mm256_set1_epi8(1);  // 创建一个包含 1 的 AVX2 寄存器
    const __m128i m32 = _mm_set1_epi8(32);  // 创建一个包含 32 的 AVX 寄存器

    __m256 acc = _mm256_setzero_ps();  // 初始化一个全零的 AVX2 寄存器

    uint32_t aux[3];  // 创建一个包含 3 个元素的无符号整数数组

    for (int i = 0; i < nb; ++i) {  // 循环遍历 nb 次

        // 计算两个浮点数的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 定义指向 x[i].qs 的限定指针
        const uint8_t * restrict q3 = x[i].qs;
        // 定义指向 y[i].qs 的限定指针
        const int8_t  * restrict q8 = y[i].qs;

        // 设置比例
        // 将 x[i].scales 的前 12 个字节复制到 aux 数组
        memcpy(aux, x[i].scales, 12);
        // 将 aux 数组中的数据按位组合成 128 位整数
        __m128i scales128 = _mm_set_epi32(
                ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4),
                ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4),
                (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4),
                (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4));
        // 用 m32 减去 scales128 中的每个元素
        scales128 = _mm_sub_epi8(scales128, m32);
        // 将 scales128 转换为 256 位整数
        const __m256i all_scales = _mm256_cvtepi8_epi16(scales128);
        // 从 all_scales 中提取低 128 位整数
        const __m128i l_scales = _mm256_extracti128_si256(all_scales, 0);
        // 从 all_scales 中提取高 128 位整数
        const __m128i h_scales = _mm256_extracti128_si256(all_scales, 1);
        // 创建包含 l_scales 和 h_scales 的 256 位整数数组
        const __m256i scales[2] = {MM256_SET_M128I(l_scales, l_scales), MM256_SET_M128I(h_scales, h_scales)};

        // 加载 x[i].hmask 中的数据到 hbits 中
        const __m256i hbits = _mm256_loadu_si256((const __m256i*)x[i].hmask);
        // 创建一个全零的 256 位整数累加器
        __m256i sumi = _mm256_setzero_si256();

        // 初始化位偏移和位掩码
        int bit = 0;
        int is  = 0;

        // 遍历 QK_K/128 次，每次处理 128 位数据
        for (int j = 0; j < QK_K/128; ++j) {
            // 加载低 2 位数据
            const __m256i q3bits = _mm256_loadu_si256((const __m256i*)q3); q3 += 32;

            // 准备低位和高位数据
            const __m256i q3l_0 = _mm256_and_si256(q3bits, m3);
            const __m256i q3h_0 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_andnot_si256(hbits, _mm256_slli_epi16(mone, bit)), bit), 2);
            ++bit;

            const __m256i q3l_1 = _mm256_and_si256(_mm256_srli_epi16(q3bits, 2), m3);
            const __m256i q3h_1 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_andnot_si256(hbits, _mm256_slli_epi16(mone, bit)), bit), 2);
            ++bit;
// 使用位移和与操作，提取q3bits的低4位
const __m256i q3l_2 = _mm256_and_si256(_mm256_srli_epi16(q3bits, 4), m3);
// 使用位移、与非和位移操作，提取q3bits的高4位
const __m256i q3h_2 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_andnot_si256(hbits, _mm256_slli_epi16(mone, bit)), bit), 2);
// 递增bit变量
++bit;

// 使用位移和与操作，提取q3bits的低6位
const __m256i q3l_3 = _mm256_and_si256(_mm256_srli_epi16(q3bits, 6), m3);
// 使用位移、与非和位移操作，提取q3bits的高6位
const __m256i q3h_3 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_andnot_si256(hbits, _mm256_slli_epi16(mone, bit)), bit), 2);
// 递增bit变量
++bit;

// 加载Q8 quants
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
const __m256i q8_2 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
const __m256i q8_3 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;

// 点乘：分别对低位和高位进行乘法累加，然后相减
__m256i q8s_0 = _mm256_maddubs_epi16(q3h_0, q8_0);
__m256i q8s_1 = _mm256_maddubs_epi16(q3h_1, q8_1);
__m256i q8s_2 = _mm256_maddubs_epi16(q3h_2, q8_2);
// 使用 AVX2 指令集计算两个 256 位整数的乘积和
__m256i q8s_3 = _mm256_maddubs_epi16(q3h_3, q8_3);

// 使用 AVX2 指令集计算四组 256 位整数的乘积和
__m256i p16_0 = _mm256_maddubs_epi16(q3l_0, q8_0);
__m256i p16_1 = _mm256_maddubs_epi16(q3l_1, q8_1);
__m256i p16_2 = _mm256_maddubs_epi16(q3l_2, q8_2);
__m256i p16_3 = _mm256_maddubs_epi16(q3l_3, q8_3);

// 使用 AVX2 指令集计算四组 256 位整数的差值
p16_0 = _mm256_sub_epi16(p16_0, q8s_0);
p16_1 = _mm256_sub_epi16(p16_1, q8s_1);
p16_2 = _mm256_sub_epi16(p16_2, q8s_2);
p16_3 = _mm256_sub_epi16(p16_3, q8s_3);

// 使用 AVX2 指令集对四组 256 位整数进行乘法和加法操作
p16_0 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(is + 0)), p16_0);
p16_1 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(is + 1)), p16_1);
p16_2 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(is + 2)), p16_2);
p16_3 = _mm256_madd_epi16(_mm256_shuffle_epi8(scales[j], get_scale_shuffle_q3k(is + 3)), p16_3);

// 使用 AVX2 指令集对四组 256 位整数进行累加操作
p16_0 = _mm256_add_epi32(p16_0, p16_1);
        // 将两个__m256类型的寄存器p16_2和p16_3中的对应元素相加，并将结果存储在p16_2中
        p16_2 = _mm256_add_epi32(p16_2, p16_3);
        // 将sumi和p16_0、p16_2对应元素相加的结果再与sumi相加，并将结果存储在sumi中
        sumi  = _mm256_add_epi32(sumi, _mm256_add_epi32(p16_0, p16_2));

        }

        // 用块缩放乘以并累加
        // 将d的值广播到__m256类型的寄存器中，然后将其与sumi转换为单精度浮点数后的值相乘，再与acc相加，并将结果存储在acc中
        acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi), acc);

    }

    // 将acc中的8个单精度浮点数求和，并返回结果
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 创建一些常量__m128i类型的寄存器，并初始化它们的值
    const __m128i m3 = _mm_set1_epi8(3);
    const __m128i mone = _mm_set1_epi8(1);
    const __m128i m32 = _mm_set1_epi8(32);
    const __m128i m2 = _mm_set1_epi8(2);

    // 将__m256类型的寄存器acc的值初始化为0
    __m256 acc = _mm256_setzero_ps();
// 定义一个指向常量无符号32位整数的指针
const uint32_t *aux;

// 遍历nb次循环
for (int i = 0; i < nb; ++i) {

    // 计算y[i].d和x[i].d的乘积，并转换为32位浮点数
    const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

    // 定义指向常量无符号8位整数的指针q3和指向常量有符号8位整数的指针q8
    const uint8_t * restrict q3 = x[i].qs;
    const int8_t  * restrict q8 = y[i].qs;

    // 设置比例
    aux = (const uint32_t *)x[i].scales;
    // 使用_mm_set_epi32函数设置128位整数，进行位运算和移位操作
    __m128i scales128 = _mm_set_epi32(
            ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4),
            ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4),
            (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4),
            (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4));
    // 对scales128进行减法操作
    scales128 = _mm_sub_epi8(scales128, m32);
    // 将scales128转换为16位整数
    const __m128i scales_0 = _mm_cvtepi8_epi16(scales128);
    const __m128i scales_1 = _mm_cvtepi8_epi16(_mm_unpackhi_epi64(scales128, scales128));
        // 定义一个包含两个__m128i类型的数组，用于存储两个缩放值
        const __m128i scales[2] = { scales_0, scales_1 };

        // 从x[i].hmask数组中加载128位数据到__m128i类型的变量hbits_0和hbits_1中
        const __m128i hbits_0 = _mm_loadu_si128((const __m128i*)&x[i].hmask[0]);
        const __m128i hbits_1 = _mm_loadu_si128((const __m128i*)&x[i].hmask[16]);

        // 初始化两个128位整型累加器为0
        __m128i sumi_0 = _mm_setzero_si128();
        __m128i sumi_1 = _mm_setzero_si128();

        // 循环遍历QK_K/128次
        for (int j = 0; j < QK_K/128; ++j) {
            // 从q3数组中加载两个128位数据到__m128i类型的变量q3bits_0和q3bits_1中，并更新q3指针
            const __m128i q3bits_0 = _mm_loadu_si128((const __m128i*)q3); q3 += 16;
            const __m128i q3bits_1 = _mm_loadu_si128((const __m128i*)q3); q3 += 16;

            // 准备低位和高位数据
            const int bit = j << 2;

            // 对q3bits_0和q3bits_1进行按位与操作，结果存储到q3l_0和q3l_1中
            const __m128i q3l_0 = _mm_and_si128(q3bits_0, m3);
            const __m128i q3l_1 = _mm_and_si128(q3bits_1, m3);
// 使用位移和逻辑运算对输入的数据进行处理，生成新的数据
const __m128i q3h_0 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_0, _mm_slli_epi16(mone, bit)), bit), 2);
const __m128i q3h_1 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_1, _mm_slli_epi16(mone, bit)), bit), 2);

const __m128i q3l_2 = _mm_and_si128(_mm_srli_epi16(q3bits_0, 2), m3);
const __m128i q3l_3 = _mm_and_si128(_mm_srli_epi16(q3bits_1, 2), m3);
const __m128i q3h_2 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_0, _mm_slli_epi16(mone, bit+1)), bit+1), 2);
const __m128i q3h_3 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_1, _mm_slli_epi16(mone, bit+1)), bit+1), 2);

const __m128i q3l_4 = _mm_and_si128(_mm_srli_epi16(q3bits_0, 4), m3);
const __m128i q3l_5 = _mm_and_si128(_mm_srli_epi16(q3bits_1, 4), m3);
const __m128i q3h_4 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_0, _mm_slli_epi16(mone, bit+2)), bit+2), 2);
const __m128i q3h_5 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_1, _mm_slli_epi16(mone, bit+2)), bit+2), 2);

const __m128i q3l_6 = _mm_and_si128(_mm_srli_epi16(q3bits_0, 6), m3);
const __m128i q3l_7 = _mm_and_si128(_mm_srli_epi16(q3bits_1, 6), m3);
const __m128i q3h_6 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_0, _mm_slli_epi16(mone, bit+3)), bit+3), 2);
const __m128i q3h_7 = _mm_slli_epi16(_mm_srli_epi16(_mm_andnot_si128(hbits_1, _mm_slli_epi16(mone, bit+3)), bit+3), 2);

// 从内存中加载 Q8 quants 数据到寄存器
const __m128i q8_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 从内存中加载16字节的数据到__m128i类型的寄存器中，并将指针向后移动16个字节
const __m128i q8_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_2 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_3 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_4 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_5 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_6 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 重复上述操作，加载下一个16字节的数据
const __m128i q8_7 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;

// 计算点积：将两个低位部分和一个高位部分分别相乘，然后相加，最后相减。高位部分已经减去了2（如果高位被设置了的话），所以如果高位未被设置，则为0，如果高位被设置，则为2
__m128i q8s_0 = _mm_maddubs_epi16(q3h_0, q8_0);
__m128i q8s_1 = _mm_maddubs_epi16(q3h_1, q8_1);
__m128i q8s_2 = _mm_maddubs_epi16(q3h_2, q8_2);
__m128i q8s_3 = _mm_maddubs_epi16(q3h_3, q8_3);
__m128i q8s_4 = _mm_maddubs_epi16(q3h_4, q8_4);
__m128i q8s_5 = _mm_maddubs_epi16(q3h_5, q8_5);
__m128i q8s_6 = _mm_maddubs_epi16(q3h_6, q8_6);
__m128i q8s_7 = _mm_maddubs_epi16(q3h_7, q8_7);
# 使用 MMX 指令对两个 8 位有符号整数进行乘法运算，并将结果相加
__m128i p16_0 = _mm_maddubs_epi16(q3l_0, q8_0);
__m128i p16_1 = _mm_maddubs_epi16(q3l_1, q8_1);
__m128i p16_2 = _mm_maddubs_epi16(q3l_2, q8_2);
__m128i p16_3 = _mm_maddubs_epi16(q3l_3, q8_3);
__m128i p16_4 = _mm_maddubs_epi16(q3l_4, q8_4);
__m128i p16_5 = _mm_maddubs_epi16(q3l_5, q8_5);
__m128i p16_6 = _mm_maddubs_epi16(q3l_6, q8_6);
__m128i p16_7 = _mm_maddubs_epi16(q3l_7, q8_7);

# 从每个结果中减去相应的 16 位有符号整数
p16_0 = _mm_sub_epi16(p16_0, q8s_0);
p16_1 = _mm_sub_epi16(p16_1, q8s_1);
p16_2 = _mm_sub_epi16(p16_2, q8s_2);
p16_3 = _mm_sub_epi16(p16_3, q8s_3);
p16_4 = _mm_sub_epi16(p16_4, q8s_4);
p16_5 = _mm_sub_epi16(p16_5, q8s_5);
p16_6 = _mm_sub_epi16(p16_6, q8s_6);
p16_7 = _mm_sub_epi16(p16_7, q8s_7);

# 创建一个包含 16 位整数值的常量向量
__m128i shuffle = _mm_set1_epi16(0x0100);
# 使用_mm_shuffle_epi8函数对scales[j]进行重新排列，并与p16_0进行16位整数乘法累加
p16_0 = _mm_madd_epi16(_mm_shuffle_epi8(scales[j], shuffle), p16_0);
# 将shuffle与m2相加，用于下一次_shuffle_epi8函数的重新排列
shuffle = _mm_add_epi16(shuffle, m2);
# 重复上述过程，对p16_1到p16_7进行相同的操作
...
# 将p16_0和p16_1相加，用于下一步累加操作
p16_0 = _mm_add_epi32(p16_0, p16_1);
# 重复上述过程，对p16_2和p16_3，p16_4和p16_5进行相同的操作
...
// 使用 SIMD 指令对两个 128 位整数寄存器进行相加
p16_6 = _mm_add_epi32(p16_6, p16_7);

// 使用 SIMD 指令对两个 128 位整数寄存器进行相加，并将结果与 sumi_0 相加
sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p16_0, p16_2));

// 使用 SIMD 指令对两个 128 位整数寄存器进行相加，并将结果与 sumi_1 相加
sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p16_4, p16_6));

// 将 sumi_1 和 sumi_0 合并成一个 256 位整数寄存器
__m256i sumi = MM256_SET_M128I(sumi_1, sumi_0);

// 将 sumi 转换成浮点数寄存器，与常数 d 相乘，再与 acc 相加
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi)), acc);

// 如果定义了 __riscv_v_intrinsic，则执行以下代码
uint32_t aux[3];
uint32_t utmp[4];
float sumf = 0;
    # 遍历循环，从0到nb-1
    for (int i = 0; i < nb; ++i) {

        # 从结构体x中获取qs指针，限制其对应的内存区域
        const uint8_t * restrict q3 = x[i].qs;
        # 从结构体x中获取hmask指针，限制其对应的内存区域
        const uint8_t * restrict qh = x[i].hmask;
        # 从结构体y中获取qs指针，限制其对应的内存区域
        const  int8_t * restrict q8 = y[i].qs;

        # 将x[i].scales的内容拷贝到aux数组中
        memcpy(aux, x[i].scales, 12);
        # 根据aux数组的内容计算utmp数组的值
        utmp[3] = ((aux[1] >> 4) & kmask2) | (((aux[2] >> 6) & kmask1) << 4);
        utmp[2] = ((aux[0] >> 4) & kmask2) | (((aux[2] >> 4) & kmask1) << 4);
        utmp[1] = (aux[1] & kmask2) | (((aux[2] >> 2) & kmask1) << 4);
        utmp[0] = (aux[0] & kmask2) | (((aux[2] >> 0) & kmask1) << 4);

        # 将utmp数组的内容转换为int8_t类型的scale数组
        int8_t * scale = (int8_t *)utmp;
        # 遍历循环，从0到15
        for (int j = 0; j < 16; ++j) scale[j] -= 32;

        # 设置vl的值为32
        size_t vl = 32;
        # 设置m的值为1
        uint8_t m =  1;

        # 创建一个vint32m1_t类型的变量vzero，并将其初始化为0
        vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);
// 从内存中加载一个 8 位无符号整数向量到寄存器 vqh 中
vuint8m1_t vqh = __riscv_vle8_v_u8m1(qh, vl);

// 初始化一个整数变量 sum_t 为 0
int sum_t = 0;

// 循环遍历 QK_K，每次增加 128
for (int j = 0; j < QK_K; j += 128) {

    // 设置向量长度为 32
    vl = 32;

    // 从内存中加载一个 8 位无符号整数向量到寄存器 q3_x 中
    vuint8m1_t q3_x = __riscv_vle8_v_u8m1(q3, vl);

    // 对 q3_x 进行位运算和类型转换，得到 q3_0, q3_1, q3_2, q3_3 四个向量
    vint8m1_t q3_0 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(q3_x, 0x03, vl));
    vint8m1_t q3_1 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q3_x, 0x2, vl), 0x03 , vl));
    vint8m1_t q3_2 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q3_x, 0x4, vl), 0x03 , vl));
    vint8m1_t q3_3 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(q3_x, 0x6, vl), 0x03 , vl));

    // 计算用于减法的掩码
    vuint8m1_t qh_m0 = __riscv_vand_vx_u8m1(vqh, m, vl);
    vbool8_t vmask_0 = __riscv_vmseq_vx_u8m1_b8(qh_m0, 0, vl);
    vint8m1_t q3_m0 = __riscv_vsub_vx_i8m1_m(vmask_0, q3_0, 0x4, vl);
            // 将 m 左移 1 位
            m <<= 1;

            // 使用位与操作符将 vqh 和 m 进行按位与操作，得到结果存储在 qh_m1 中
            vuint8m1_t qh_m1 = __riscv_vand_vx_u8m1(vqh, m, vl);
            // 使用相等比较操作符判断 qh_m1 中的值是否等于 0，结果存储在 vmask_1 中
            vbool8_t vmask_1 = __riscv_vmseq_vx_u8m1_b8(qh_m1, 0, vl);
            // 使用条件减法操作符，根据 vmask_1 中的值判断是否执行减法操作，结果存储在 q3_m1 中
            vint8m1_t q3_m1 = __riscv_vsub_vx_i8m1_m(vmask_1, q3_1, 0x4, vl);
            // 再次将 m 左移 1 位
            m <<= 1;

            // 重复上述操作，分别计算 qh_m2, vmask_2, q3_m2
            vuint8m1_t qh_m2 = __riscv_vand_vx_u8m1(vqh, m, vl);
            vbool8_t vmask_2 = __riscv_vmseq_vx_u8m1_b8(qh_m2, 0, vl);
            vint8m1_t q3_m2 = __riscv_vsub_vx_i8m1_m(vmask_2, q3_2, 0x4, vl);
            m <<= 1;

            // 重复上述操作，分别计算 qh_m3, vmask_3, q3_m3
            vuint8m1_t qh_m3 = __riscv_vand_vx_u8m1(vqh, m, vl);
            vbool8_t vmask_3 = __riscv_vmseq_vx_u8m1_b8(qh_m3, 0, vl);
            vint8m1_t q3_m3 = __riscv_vsub_vx_i8m1_m(vmask_3, q3_3, 0x4, vl);
            m <<= 1;

            // 加载 Q8 并与 Q3 求积
            // 使用乘法操作符将 q3_m0 和 q8 中的值进行乘法操作，结果存储在 a0 中
            vint16m2_t a0 = __riscv_vwmul_vv_i16m2(q3_m0, __riscv_vle8_v_i8m1(q8, vl), vl);
            // 使用乘法操作符将 q3_m1 和 q8+32 中的值进行乘法操作，结果存储在 a1 中
            vint16m2_t a1 = __riscv_vwmul_vv_i16m2(q3_m1, __riscv_vle8_v_i8m1(q8+32, vl), vl);
// 使用 q3_m2 和 q8+64 位置的数据进行 16 位整数乘法
vint16m2_t a2 = __riscv_vwmul_vv_i16m2(q3_m2, __riscv_vle8_v_i8m1(q8+64, vl), vl);
// 使用 q3_m3 和 q8+96 位置的数据进行 16 位整数乘法
vint16m2_t a3 = __riscv_vwmul_vv_i16m2(q3_m3, __riscv_vle8_v_i8m1(q8+96, vl), vl);

// 设置向量长度为 16
vl = 16;

// 从 a0 中获取第 0 个元素，与 scale[0] 相乘
vint32m2_t aux0_0 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a0, 0), (scale[0]), vl);
// 从 a0 中获取第 1 个元素，与 scale[1] 相乘
vint32m2_t aux0_1 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a0, 1), (scale[1]), vl);
// 从 a1 中获取第 0 个元素，与 scale[2] 相乘
vint32m2_t aux1_0 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a1, 0), (scale[2]), vl);
// 从 a1 中获取第 1 个元素，与 scale[3] 相乘
vint32m2_t aux1_1 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a1, 1), (scale[3]), vl);
// 从 a2 中获取第 0 个元素，与 scale[4] 相乘
vint32m2_t aux2_0 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a2, 0), (scale[4]), vl);
// 从 a2 中获取第 1 个元素，与 scale[5] 相乘
vint32m2_t aux2_1 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a2, 1), (scale[5]), vl);
// 从 a3 中获取第 0 个元素，与 scale[6] 相乘
vint32m2_t aux3_0 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a3, 0), (scale[6]), vl);
// 从 a3 中获取第 1 个元素，与 scale[7] 相乘
vint32m2_t aux3_1 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(a3, 1), (scale[7]), vl);

// 对 aux0_0 和 aux0_1 相加，然后进行向量求和
vint32m1_t isum0 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(aux0_0, aux0_1, vl), vzero, vl);
// 对 aux1_0 和 aux1_1 相加，然后进行向量求和
vint32m1_t isum1 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(aux1_0, aux1_1, vl), isum0, vl);
// 对 aux2_0 和 aux2_1 相加，然后进行向量求和
vint32m1_t isum2 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(aux2_0, aux2_1, vl), isum1, vl);
// 对 aux3_0 和 aux3_1 相加，然后进行向量求和
vint32m1_t isum3 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(aux3_0, aux3_1, vl), isum2, vl);
            // 将__riscv_vmv_x_s_i32m1_i32(isum3)的结果累加到sum_t中
            sum_t +=  __riscv_vmv_x_s_i32m1_i32(isum3);

            // 更新q3、q8、scale的值
            q3 += 32;    q8 += 128;   scale += 8;

        }

        // 计算x[i].d和y[i].d的乘积，并赋值给d
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;

        // 将d乘以sum_t的结果累加到sumf中
        sumf += d*sum_t;

    }

    // 将sumf的值赋给*s
    *s = sumf;

#else
    // scalar version
    // This function is written like this so the compiler can manage to vectorize most of it
    // Using -Ofast, GCC and clang manage to produce code that is within a factor of 2 or so from the
    // manually vectorized version above. Every other version I tried would run at least 4 times slower.
    // The ideal situation would be if we could just write the code once, and the compiler would
    // 自动产生最佳的机器指令集，而不是我们手动为 AVX、ARM_NEON 等编写矢量化版本

    int8_t  aux8[QK_K]; // 声明一个长度为 QK_K 的 int8_t 类型数组
    int16_t aux16[8];   // 声明一个长度为 8 的 int16_t 类型数组
    float   sums [8];    // 声明一个长度为 8 的 float 类型数组
    int32_t aux32[8];    // 声明一个长度为 8 的 int32_t 类型数组
    memset(sums, 0, 8*sizeof(float)); // 将 sums 数组的内容初始化为 0

    uint32_t auxs[4];    // 声明一个长度为 4 的 uint32_t 类型数组
    const int8_t * scales = (const int8_t*)auxs; // 将 auxs 强制转换为 int8_t 类型指针，并赋值给 scales

    float sumf = 0;      // 声明一个 float 类型变量 sumf，并初始化为 0
    for (int i = 0; i < nb; ++i) { // 循环，i 从 0 到 nb-1
        const uint8_t * restrict q3 = x[i].qs; // 声明一个指向 x[i].qs 的 uint8_t 类型指针，并使用 restrict 限定
        const uint8_t * restrict hm = x[i].hmask; // 声明一个指向 x[i].hmask 的 uint8_t 类型指针，并使用 restrict 限定
        const  int8_t * restrict q8 = y[i].qs; // 声明一个指向 y[i].qs 的 int8_t 类型指针，并使用 restrict 限定
        memset(aux32, 0, 8*sizeof(int32_t)); // 将 aux32 数组的内容初始化为 0
        int8_t * restrict a = aux8; // 声明一个指向 aux8 的 int8_t 类型指针，并使用 restrict 限定
        uint8_t m = 1; // 声明一个 uint8_t 类型变量 m，并初始化为 1
# 循环处理QK_K次，每次处理128个元素
for (int j = 0; j < QK_K; j += 128) {
    # 将q3中的前32个元素的低2位存入a数组
    for (int l = 0; l < 32; ++l) a[l] = q3[l] & 3;
    # 将a数组中的元素减去hm数组中对应位置的值（如果hm数组中对应位置的值为0，则减去4）
    for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
    # a指针向后移动32个位置，m左移1位
    a += 32; m <<= 1;
    # 将q3中的前32个元素的2-3位存入a数组
    for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 2) & 3;
    # 将a数组中的元素减去hm数组中对应位置的值（如果hm数组中对应位置的值为0，则减去4）
    for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
    # a指针向后移动32个位置，m左移1位
    a += 32; m <<= 1;
    # 将q3中的前32个元素的4-5位存入a数组
    for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 4) & 3;
    # 将a数组中的元素减去hm数组中对应位置的值（如果hm数组中对应位置的值为0，则减去4）
    for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
    # a指针向后移动32个位置，m左移1位
    a += 32; m <<= 1;
    # 将q3中的前32个元素的6-7位存入a数组
    for (int l = 0; l < 32; ++l) a[l] = (q3[l] >> 6) & 3;
    # 将a数组中的元素减去hm数组中对应位置的值（如果hm数组中对应位置的值为0，则减去4）
    for (int l = 0; l < 32; ++l) a[l] -= (hm[l] & m ? 0 : 4);
    # a指针向后移动32个位置，m左移1位
    a += 32; m <<= 1;
    # q3指针向后移动32个位置
    q3 += 32;
}
# 将aux8的内容复制到a中
a = aux8;
# 将x[i].scales的内容复制到auxs中
memcpy(auxs, x[i].scales, 12);
# 将auxs[2]的值存入tmp
uint32_t tmp = auxs[2];
# 将auxs[2]的值更新为((auxs[0]的右移4位与kmask2的按位与结果)与((tmp的右移4位与kmask1的按位与结果)左移4位)的按位或结果
auxs[2] = ((auxs[0] >> 4) & kmask2) | (((tmp >> 4) & kmask1) << 4);
        # 对数组进行位运算和赋值操作
        auxs[3] = ((auxs[1] >> 4) & kmask2) | (((tmp >> 6) & kmask1) << 4);
        auxs[0] = (auxs[0] & kmask2) | (((tmp >> 0) & kmask1) << 4);
        auxs[1] = (auxs[1] & kmask2) | (((tmp >> 2) & kmask1) << 4);
        # 循环计算
        for (int j = 0; j < QK_K/16; ++j) {
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += (scales[j] - 32) * aux16[l];
            q8 += 8; a += 8;
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            for (int l = 0; l < 8; ++l) aux32[l] += (scales[j] - 32) * aux16[l];
            q8 += 8; a += 8;
        }
        # 计算浮点数并赋值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    }
    # 循环计算
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    *s = sumf;

#endif

}
// 如果条件不成立，则执行以下代码
#else

// 计算两个向量的点积，其中n为向量长度，s为结果数组，vx和vy为输入向量
void ggml_vec_dot_q3_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言n能被QK_K整除
    assert(n % QK_K == 0);

    // 将输入向量转换为特定类型的指针
    const block_q3_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以QK_K的商
    const int nb = n / QK_K;

    // 如果支持ARM NEON指令集
#ifdef __ARM_NEON

    // 如果支持ARM Dot Product指令
#ifdef __ARM_FEATURE_DOTPROD
    // 创建一个全为0的int32x4_t类型的向量
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 创建一个全为0x3的uint8x16_t类型的向量
    const uint8x16_t m3b = vdupq_n_u8(0x3);
    // 创建一个全为4的uint8x16_t类型的向量
    const uint8x16_t mh  = vdupq_n_u8(4);
    # 定义一个包含4个8位整数的向量类型变量
    ggml_int8x16x4_t q3bytes;

    # 定义一个包含2个16位无符号整数的数组
    uint16_t aux16[2];
    # 将aux16数组强制转换为int8_t类型的指针，并赋值给scales变量
    int8_t * scales = (int8_t *)aux16;

    # 定义一个浮点数变量sum，并初始化为0
    float sum = 0;

    # 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        # 定义一个包含4个16位无符号整数的向量类型变量
        ggml_uint8x16x4_t q3h;

        # 从x[i].hmask中加载8个8位无符号整数，赋值给hbits
        const uint8x8_t  hbits    = vld1_u8(x[i].hmask);
        # 从x[i].qs中加载16个8位无符号整数，赋值给q3bits
        const uint8x16_t q3bits   = vld1q_u8(x[i].qs);
        # 从y[i].qs中加载4个16位有符号整数，赋值给q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(y[i].qs);

        # 从x[i].scales中加载一个16位无符号整数，赋值给a
        const uint16_t a = *(const uint16_t *)x[i].scales;
        # 将a的低8位和高8位分别存入aux16数组的两个元素中
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        # 循环遍历4次，将scales数组中的每个元素减去8
        for (int j = 0; j < 4; ++j) scales[j] -= 8;
        # 计算isum的值，使用了一系列的乘法和加法操作
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        # 计算d的值，使用了y[i].d和x[i].d的乘法操作
        const float d = y[i].d * (float)x[i].d;

        # 对hbits进行一系列的位运算操作，得到htmp
        const uint8x16_t htmp = vcombine_u8(hbits, vshr_n_u8(hbits, 1));
        q3h.val[0] = vandq_u8(mh, vshlq_n_u8(htmp, 2));
        q3h.val[1] = vandq_u8(mh, htmp);
        q3h.val[2] = vandq_u8(mh, vshrq_n_u8(htmp, 2));
        q3h.val[3] = vandq_u8(mh, vshrq_n_u8(htmp, 4));

        # 对q3bits和m3b进行位运算操作，得到q3bytes
        q3bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q3bits, m3b),                q3h.val[0]));
        q3bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 2), m3b), q3h.val[1]));
        q3bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(vshrq_n_u8(q3bits, 4), m3b), q3h.val[2]));
        q3bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q3bits, 6),                q3h.val[3]));

        # 如果支持ARM的dot product特性，则使用dot product计算isum的值
        # 并且使用一系列的乘法和加法操作
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[0], q8bytes.val[0])) * scales[0];
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[1], q8bytes.val[1])) * scales[2];
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[2], q8bytes.val[2])) * scales[1];
# 如果条件满足，则执行以下代码块
        # 计算第一个元素的乘积和
        isum += vaddvq_s32(vdotq_s32(vzero, q3bytes.val[3], q8bytes.val[3])) * scales[3];
# 否则
        # 计算每个元素的乘积和
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q3bytes.val[0]), vget_high_s8(q8bytes.val[0]));
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q3bytes.val[1]), vget_high_s8(q8bytes.val[1]));
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q3bytes.val[2]), vget_high_s8(q8bytes.val[2]));
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q3bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q3bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        # 计算每个元素的乘积和，并乘以相应的比例因子，然后累加到isum中
        isum += vaddvq_s16(p0) * scales[0] + vaddvq_s16(p1) * scales[2] + vaddvq_s16(p2) * scales[1] + vaddvq_s16(p3) * scales[3];
# 结束条件判断

        # 计算最终结果并累加到sum中
        sum += d * isum;

    }

    *s = sum;

#elif defined __AVX2__
    // 创建一个包含8个元素，每个元素都是3的__m256i类型的向量
    const __m256i m3 = _mm256_set1_epi8(3);
    // 创建一个包含8个元素，每个元素都是1的__m256i类型的向量
    const __m256i m1 = _mm256_set1_epi8(1);

    // 创建一个包含8个元素，每个元素都是0的__m256类型的向量
    __m256 acc = _mm256_setzero_ps();

    // 64位辅助变量
    uint64_t aux64;

    // 16位辅助数组
    uint16_t aux16[2];
    // 将16位辅助数组转换为8位辅助指针
    const int8_t * aux8 = (const int8_t *)aux16;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 计算y[i].d和x[i].d的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 限制指针q3指向x[i].qs
        const uint8_t * restrict q3 = x[i].qs;
        // 限制指针q8指向y[i].qs
        const int8_t  * restrict q8 = y[i].qs;

        // 从x[i].scales中读取一个16位的无符号整数，并将其存储到a中
        const uint16_t a = *(const uint16_t *)x[i].scales;
        // 将a的低8位存储到aux16[0]中
        aux16[0] = a & 0x0f0f;
        # 将 a 右移 4 位，并且与 0x0f0f 进行按位与操作，结果存入 aux16[1]
        aux16[1] = (a >> 4) & 0x0f0f;

        # 创建两个 256 位整数变量 scale_0 和 scale_1，分别存储 aux8[2] - 8 和 aux8[0] - 8 的值
        const __m256i scale_0 = MM256_SET_M128I(_mm_set1_epi16(aux8[2] - 8), _mm_set1_epi16(aux8[0] - 8));
        const __m256i scale_1 = MM256_SET_M128I(_mm_set1_epi16(aux8[3] - 8), _mm_set1_epi16(aux8[1] - 8));

        # 将 x[i].hmask 的前 8 个字节拷贝到 aux64 中
        memcpy(&aux64, x[i].hmask, 8);

        # 创建一个 128 位整数变量 haux，存储 aux64 右移 1 位和右移 0 位的值
        const __m128i haux = _mm_set_epi64x(aux64 >> 1, aux64 >> 0);
        # 创建两个 256 位整数变量 q3h_0 和 q3h_1，分别存储 haux 右移 2 位和右移 4 位的值
        __m256i q3h_0 = MM256_SET_M128I(_mm_srli_epi16(haux, 2), haux);
        __m256i q3h_1 = _mm256_srli_epi16(q3h_0, 4);
        # 对 q3h_0 和 q3h_1 进行位操作，并左移 2 位，结果存入 q3h_0 和 q3h_1
        q3h_0 = _mm256_slli_epi16(_mm256_andnot_si256(q3h_0, m1), 2);
        q3h_1 = _mm256_slli_epi16(_mm256_andnot_si256(q3h_1, m1), 2);

        # 加载 q3 的低 2 位到 128 位整数变量 q3bits
        const __m128i q3bits = _mm_loadu_si128((const __m128i*)q3);

        # 准备低位和高位的位操作结果，存入 q3l_0 和 q3l_1
        const __m256i q3aux  = MM256_SET_M128I(_mm_srli_epi16(q3bits, 2), q3bits);
        const __m256i q3l_0 = _mm256_and_si256(q3aux, m3);
        const __m256i q3l_1 = _mm256_and_si256(_mm256_srli_epi16(q3aux, 4), m3);
// 加载 Q8 quants
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

// 点积：我们分别对低位和高位进行乘法运算，这样我们可以使用 _mm256_maddubs_epi16，
// 然后进行减法运算。高位部分已经减去了2（如果高位未设置，则为零，如果高位设置，则为2）
const __m256i q8s_0 = _mm256_maddubs_epi16(q3h_0, q8_0);
const __m256i q8s_1 = _mm256_maddubs_epi16(q3h_1, q8_1);

__m256i p16_0 = _mm256_maddubs_epi16(q3l_0, q8_0);
__m256i p16_1 = _mm256_maddubs_epi16(q3l_1, q8_1);

p16_0 = _mm256_sub_epi16(p16_0, q8s_0);
p16_1 = _mm256_sub_epi16(p16_1, q8s_1);

// 乘以比例
p16_0 = _mm256_madd_epi16(scale_0, p16_0);
p16_1 = _mm256_madd_epi16(scale_1, p16_1);
    // 将两个__m256类型的寄存器中的对应元素相加
    p16_0 = _mm256_add_epi32(p16_0, p16_1);

    // 将__m256类型的寄存器中的元素与标量d相乘，并将结果与acc寄存器中的元素相加
    acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(p16_0), acc);

    // 如果定义了__AVX__，则执行以下代码
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 创建一个所有元素都为3的__m128i类型的寄存器
    const __m128i m3 = _mm_set1_epi8(3);
    // 创建一个所有元素都为1的__m128i类型的寄存器
    const __m128i m1 = _mm_set1_epi8(1);

    // 创建一个所有元素都为0的__m256类型的寄存器
    __m256 acc = _mm256_setzero_ps();

    // 64位辅助变量
    uint64_t aux64;

    // 16位辅助数组
    uint16_t aux16[2];
    // 将 aux16 转换为 int8_t 类型的指针
    const int8_t * aux8 = (const int8_t *)aux16;

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 y[i].d 与 x[i].d 的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 将 x[i].qs 转换为 uint8_t 类型的指针，将 y[i].qs 转换为 int8_t 类型的指针
        const uint8_t * restrict q3 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 从 x[i].scales 中读取 uint16_t 类型的值，并将其存储到 aux16 中
        const uint16_t a = *(const uint16_t *)x[i].scales;
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        // 使用 aux8 中的值减去 8，分别创建四个 __m128i 类型的变量
        const __m128i scale_0 = _mm_set1_epi16(aux8[0] - 8);
        const __m128i scale_1 = _mm_set1_epi16(aux8[2] - 8);
        const __m128i scale_2 = _mm_set1_epi16(aux8[1] - 8);
        const __m128i scale_3 = _mm_set1_epi16(aux8[3] - 8);

        // 将 x[i].hmask 的内容复制到 aux64 中
        memcpy(&aux64, x[i].hmask, 8);
// 使用 aux64 的值创建一个包含两个 64 位整数的 __m128i 类型变量 q3h_0
__m128i q3h_0 = _mm_set_epi64x(aux64 >> 1, aux64 >> 0);
// 对 q3h_0 中的每个 16 位整数进行逻辑右移 2 位，存储到 q3h_1
__m128i q3h_1 = _mm_srli_epi16(q3h_0, 2);
// 对 q3h_0 中的每个 16 位整数进行逻辑右移 4 位，存储到 q3h_2
__m128i q3h_2 = _mm_srli_epi16(q3h_0, 4);
// 对 q3h_0 中的每个 16 位整数进行逻辑右移 6 位，存储到 q3h_3
__m128i q3h_3 = _mm_srli_epi16(q3h_0, 6);
// 对 q3h_0 中的每个 16 位整数进行逻辑非操作，然后与 m1 进行按位与操作，再左移 2 位，存储回 q3h_0
q3h_0 = _mm_slli_epi16(_mm_andnot_si128(q3h_0, m1), 2);
// 对 q3h_1 中的每个 16 位整数进行逻辑非操作，然后与 m1 进行按位与操作，再左移 2 位，存储回 q3h_1
q3h_1 = _mm_slli_epi16(_mm_andnot_si128(q3h_1, m1), 2);
// 对 q3h_2 中的每个 16 位整数进行逻辑非操作，然后与 m1 进行按位与操作，再左移 2 位，存储回 q3h_2
q3h_2 = _mm_slli_epi16(_mm_andnot_si128(q3h_2, m1), 2);
// 对 q3h_3 中的每个 16 位整数进行逻辑非操作，然后与 m1 进行按位与操作，再左移 2 位，存储回 q3h_3
q3h_3 = _mm_slli_epi16(_mm_andnot_si128(q3h_3, m1), 2);

// 从内存中加载 q3 的值，存储到 q3bits
const __m128i q3bits = _mm_loadu_si128((const __m128i*)q3);

// 对 q3bits 中的每个 16 位整数与 m3 进行按位与操作，存储到 q3l_0
const __m128i q3l_0 = _mm_and_si128(q3bits, m3);
// 对 q3bits 中的每个 16 位整数右移 2 位，再与 m3 进行按位与操作，存储到 q3l_1
const __m128i q3l_1 = _mm_and_si128(_mm_srli_epi16(q3bits, 2), m3);
// 对 q3bits 中的每个 16 位整数右移 4 位，再与 m3 进行按位与操作，存储到 q3l_2
const __m128i q3l_2 = _mm_and_si128(_mm_srli_epi16(q3bits, 4), m3);
// 对 q3bits 中的每个 16 位整数右移 6 位，再与 m3 进行按位与操作，存储到 q3l_3
const __m128i q3l_3 = _mm_and_si128(_mm_srli_epi16(q3bits, 6), m3);

// 从内存中加载 q8 的值，存储到 q8_0
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
// 从内存中加载256位的数据到寄存器中
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

// 计算点积：我们分别对低位和高位进行乘法运算，这样我们可以使用_mm_maddubs_epi16函数，然后进行减法运算。
// 高位部分已经减去了2（所以，如果高位未设置，则为零，如果高位设置了，则为2）
const __m128i q8s_0 = _mm_maddubs_epi16(q3h_0, _mm256_extractf128_si256(q8_0, 0));
const __m128i q8s_1 = _mm_maddubs_epi16(q3h_1, _mm256_extractf128_si256(q8_0, 1));
const __m128i q8s_2 = _mm_maddubs_epi16(q3h_2, _mm256_extractf128_si256(q8_1, 0));
const __m128i q8s_3 = _mm_maddubs_epi16(q3h_3, _mm256_extractf128_si256(q8_1, 1));

__m128i p16_0 = _mm_maddubs_epi16(q3l_0, _mm256_extractf128_si256(q8_0, 0));
__m128i p16_1 = _mm_maddubs_epi16(q3l_1, _mm256_extractf128_si256(q8_0, 1));
__m128i p16_2 = _mm_maddubs_epi16(q3l_2, _mm256_extractf128_si256(q8_1, 0));
__m128i p16_3 = _mm_maddubs_epi16(q3l_3, _mm256_extractf128_si256(q8_1, 1));

p16_0 = _mm_sub_epi16(p16_0, q8s_0);
p16_1 = _mm_sub_epi16(p16_1, q8s_1);
p16_2 = _mm_sub_epi16(p16_2, q8s_2);
p16_3 = _mm_sub_epi16(p16_3, q8s_3);
        // 使用规模进行乘法运算
        p16_0 = _mm_madd_epi16(scale_0, p16_0);
        p16_1 = _mm_madd_epi16(scale_1, p16_1);
        p16_2 = _mm_madd_epi16(scale_2, p16_2);
        p16_3 = _mm_madd_epi16(scale_3, p16_3);

        // 将两个 32 位整数相加
        p16_0 = _mm_add_epi32(p16_0, p16_2);
        p16_1 = _mm_add_epi32(p16_1, p16_3);
        // 将两个 128 位寄存器合并成一个 256 位寄存器
        __m256i p16 = MM256_SET_M128I(p16_1, p16_0);

        // 使用块规模进行乘法运算并累加
        acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(p16)), acc);

    }

    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic

    uint16_t aux16[2];
```

    // 将aux16强制转换为int8_t类型的指针
    int8_t * scales = (int8_t *)aux16;

    // 初始化sumf为0
    float sumf = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 限制指针q3和q8的访问范围，分别指向x[i].qs和y[i].qs
        const uint8_t * restrict q3 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 从x[i].scales中读取一个uint16_t类型的值，并将其存储到a中
        const uint16_t a = *(const uint16_t *)x[i].scales;
        // 将a的低8位和高8位分别存储到aux16的两个元素中
        aux16[0] = a & 0x0f0f;
        aux16[1] = (a >> 4) & 0x0f0f;

        // 循环遍历4次，将scales数组中的每个元素减去8
        for (int j = 0; j < 4; ++j) scales[j] -= 8;

        // 计算isum的值
        int32_t isum = -4*(scales[0] * y[i].bsums[0] + scales[2] * y[i].bsums[1] + scales[1] * y[i].bsums[2] + scales[3] * y[i].bsums[3]);

        // 计算d的值
        const float d = y[i].d * (float)x[i].d;

        // 初始化vzero为一个全0的向量
        vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);
// 加载 qh
vuint8mf4_t qh_x1   = __riscv_vle8_v_u8mf4(x[i].hmask, 8); // 从数组 x[i].hmask 中加载数据到向量寄存器 qh_x1
vuint8mf2_t qh_x2   = __riscv_vlmul_ext_v_u8mf4_u8mf2(__riscv_vsrl_vx_u8mf4(qh_x1, 1, 8)); // 对 qh_x1 进行位移和扩展操作，得到向量寄存器 qh_x2

size_t vl = 16; // 定义变量 vl 为 16

// 扩展和合并 qh_x1 和 qh_x2
vuint8mf2_t qh_x = __riscv_vslideup_vx_u8mf2(__riscv_vlmul_ext_v_u8mf4_u8mf2(qh_x1), qh_x2, vl/2, vl); // 对 qh_x1 和 qh_x2 进行扩展和合并操作，得到向量寄存器 qh_x

vuint8mf2_t qh_0 = __riscv_vand_vx_u8mf2(__riscv_vsll_vx_u8mf2(qh_x, 0x2, vl), 0x4, vl); // 对 qh_x 进行位移和按位与操作，得到向量寄存器 qh_0
vuint8mf2_t qh_1 = __riscv_vand_vx_u8mf2(qh_x, 0x4, vl); // 对 qh_x 进行按位与操作，得到向量寄存器 qh_1
vuint8mf2_t qh_2 = __riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(qh_x, 0x2, vl), 0x4, vl); // 对 qh_x 进行位移和按位与操作，得到向量寄存器 qh_2
vuint8mf2_t qh_3 = __riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(qh_x, 0x4, vl), 0x4, vl); // 对 qh_x 进行位移和按位与操作，得到向量寄存器 qh_3

// 加载 Q3
vuint8mf2_t q3_x  = __riscv_vle8_v_u8mf2(q3, vl); // 从数组 q3 中加载数据到向量寄存器 q3_x

vuint8mf2_t q3h_0 = __riscv_vor_vv_u8mf2(__riscv_vand_vx_u8mf2(q3_x, 0x3, vl), qh_0, vl); // 对 q3_x 和 qh_0 进行按位与和按位或操作，得到向量寄存器 q3h_0
vuint8mf2_t q3h_1 = __riscv_vor_vv_u8mf2(__riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(q3_x, 2, vl), 0x3, vl), qh_1, vl); // 对 q3_x 进行位移和按位与操作，再与 qh_1 进行按位或操作，得到向量寄存器 q3h_1
# 计算 q3h_2，将 q3_x 右移 4 位，与 0x3 进行按位与操作，再与 qh_2 进行按位或操作
vuint8mf2_t q3h_2 = __riscv_vor_vv_u8mf2(__riscv_vand_vx_u8mf2(__riscv_vsrl_vx_u8mf2(q3_x, 4, vl), 0x3, vl), qh_2, vl);
# 计算 q3h_3，将 q3_x 右移 6 位，与 qh_3 进行按位或操作
vuint8mf2_t q3h_3 = __riscv_vor_vv_u8mf2(__riscv_vsrl_vx_u8mf2(q3_x, 0x6, vl), qh_3, vl);

# 将 q3h_0 转换为有符号整数类型
vint8mf2_t q3_0 = __riscv_vreinterpret_v_u8mf2_i8mf2(q3h_0);
# 将 q3h_1 转换为有符号整数类型
vint8mf2_t q3_1 = __riscv_vreinterpret_v_u8mf2_i8mf2(q3h_1);
# 将 q3h_2 转换为有符号整数类型
vint8mf2_t q3_2 = __riscv_vreinterpret_v_u8mf2_i8mf2(q3h_2);
# 将 q3h_3 转换为有符号整数类型
vint8mf2_t q3_3 = __riscv_vreinterpret_v_u8mf2_i8mf2(q3h_3);

# 加载 Q8 并与 Q3 求积
vint16m1_t p0 = __riscv_vwmul_vv_i16m1(q3_0, __riscv_vle8_v_i8mf2(q8, vl), vl);
vint16m1_t p1 = __riscv_vwmul_vv_i16m1(q3_1, __riscv_vle8_v_i8mf2(q8+16, vl), vl);
vint16m1_t p2 = __riscv_vwmul_vv_i16m1(q3_2, __riscv_vle8_v_i8mf2(q8+32, vl), vl);
vint16m1_t p3 = __riscv_vwmul_vv_i16m1(q3_3, __riscv_vle8_v_i8mf2(q8+48, vl), vl);

# 对 p0 求和
vint32m1_t vs_0 = __riscv_vwredsum_vs_i16m1_i32m1(p0, vzero, vl);
# 对 p1 求和
vint32m1_t vs_1 = __riscv_vwredsum_vs_i16m1_i32m1(p1, vzero, vl);
# 对 p2 求和
vint32m1_t vs_2 = __riscv_vwredsum_vs_i16m1_i32m1(p2, vzero, vl);
# 对 p3 求和
vint32m1_t vs_3 = __riscv_vwredsum_vs_i16m1_i32m1(p3, vzero, vl);

# 将 vs_0 转换为标量整数类型，并乘以 scales[0]，加到 isum 上
isum += __riscv_vmv_x_s_i32m1_i32(vs_0) * scales[0];
        # 计算isum的值，使用__riscv_vmv_x_s_i32m1_i32函数返回的值与scales数组中的值相乘，并累加到isum中
        isum += __riscv_vmv_x_s_i32m1_i32(vs_1) * scales[2];
        isum += __riscv_vmv_x_s_i32m1_i32(vs_2) * scales[1];
        isum += __riscv_vmv_x_s_i32m1_i32(vs_3) * scales[3];

        # 将d与isum的乘积累加到sumf中
        sumf += d * isum;

    }

    # 如果条件不满足，则执行以下代码
    *s = sumf;

#else

    # 定义一些辅助数组和变量
    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    int32_t scales[4];
    # 将sums数组的值全部初始化为0
    memset(sums, 0, 8*sizeof(float));

    # 定义并初始化sumf变量
    float sumf = 0;
# 遍历 nb 次，nb 为某个变量的值
for (int i = 0; i < nb; ++i) {
    # 为了提高性能，使用 restrict 修饰指针，表示该指针是唯一访问其所指向对象的指针
    const uint8_t * restrict q3 = x[i].qs;  # 从 x[i] 中获取 qs，并赋值给 q3
    const uint8_t * restrict hm = x[i].hmask;  # 从 x[i] 中获取 hmask，并赋值给 hm
    const  int8_t * restrict q8 = y[i].qs;  # 从 y[i] 中获取 qs，并赋值给 q8
    int8_t * restrict a = aux8;  # 将 aux8 的地址赋值给 a
    # 遍历 8 次
    for (int l = 0; l < 8; ++l) {
        # 根据位运算获取 q3 中的数据，并根据 hm 中的数据进行调整，然后赋值给 a
        a[l+ 0] = (int8_t)((q3[l+0] >> 0) & 3) - (hm[l] & 0x01 ? 0 : 4);
        a[l+ 8] = (int8_t)((q3[l+8] >> 0) & 3) - (hm[l] & 0x02 ? 0 : 4);
        a[l+16] = (int8_t)((q3[l+0] >> 2) & 3) - (hm[l] & 0x04 ? 0 : 4);
        a[l+24] = (int8_t)((q3[l+8] >> 2) & 3) - (hm[l] & 0x08 ? 0 : 4);
        a[l+32] = (int8_t)((q3[l+0] >> 4) & 3) - (hm[l] & 0x10 ? 0 : 4);
        a[l+40] = (int8_t)((q3[l+8] >> 4) & 3) - (hm[l] & 0x20 ? 0 : 4);
        a[l+48] = (int8_t)((q3[l+0] >> 6) & 3) - (hm[l] & 0x40 ? 0 : 4);
        a[l+56] = (int8_t)((q3[l+8] >> 6) & 3) - (hm[l] & 0x80 ? 0 : 4);
    }

    # 对 scales 数组进行赋值
    scales[0] = (x[i].scales[0] & 0xF) - 8;
    scales[1] = (x[i].scales[0] >>  4) - 8;
    scales[2] = (x[i].scales[1] & 0xF) - 8;
    scales[3] = (x[i].scales[1] >>  4) - 8;
}
# 将aux32数组的前8个元素初始化为0
memset(aux32, 0, 8*sizeof(int32_t));

# 遍历QK_K/16次
for (int j = 0; j < QK_K/16; ++j) {
    # 计算aux16数组的值
    for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
    # 更新q8和a的指针位置
    q8 += 8; a += 8;
    # 继续计算aux16数组的值
    for (int l = 0; l < 8; ++l) aux16[l] += q8[l] * a[l];
    # 更新q8和a的指针位置
    q8 += 8; a += 8;
    # 计算aux32数组的值
    for (int l = 0; l < 8; ++l) aux32[l] += scales[j] * aux16[l];
}

# 计算d的值
const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
# 更新sums数组的值
for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
}

# 计算sumf的值
for (int l = 0; l < 8; ++l) sumf += sums[l];
# 将sumf的值赋给指针s所指向的位置
*s = sumf;
#if QK_K == 256
// 如果 QK_K 等于 256，则执行以下代码
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算块的数量
    const int nb = n / QK_K;

    // 定义用于位运算的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;

    // 定义用于 NEON 指令的临时变量
    uint32_t utmp[4];

#ifdef __ARM_NEON
    // 定义用于 NEON 指令的掩码
    const uint8x16_t m4b = vdupq_n_u8(0xf);
#ifdef __ARM_FEATURE_DOTPROD
    // 定义用于 NEON 指令的零向量
    const int32x4_t mzero = vdupq_n_s32(0);
#endif
// 结束条件判断，根据条件结束代码块

ggml_int8x16x2_t q4bytes;
ggml_int8x16x2_t q8bytes;
// 声明两个自定义的数据类型

float sumf = 0;
// 声明一个浮点数变量并初始化为0

for (int i = 0; i < nb; ++i) {
    // 循环，从0到nb-1，每次递增1

    const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
    const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
    // 计算两个浮点数变量的值

    const int16x8_t q8sums = vpaddq_s16(vld1q_s16(y[i].bsums), vld1q_s16(y[i].bsums + 8));
    // 使用SIMD指令进行向量运算

    memcpy(utmp, x[i].scales, 12);
    // 将x[i].scales的内容复制到utmp中

    uint32x2_t mins8 = { 0 };
    mins8 = vset_lane_u32(utmp[1] & kmask1, mins8, 0);
    mins8 = vset_lane_u32(((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4), mins8, 1);
    // 使用SIMD指令进行位运算
        # 将 utmp[2] 和 kmask2 进行位与运算，并将结果存入 utmp[1]
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        # 将 utmp[0] 和 kmask1 进行位与运算，并将结果存入 utmp[0]
        utmp[0] &= kmask1;

        # 将 mins8 转换为 uint8x16_t 类型，然后转换为 int16x8_t 类型
        const int16x8_t mins = vreinterpretq_s16_u16(vmovl_u8(vreinterpret_u8_u32(mins8)));
        # 计算两个 int16x8_t 类型的向量的乘积，并将结果存入 int32x4_t 类型的向量 prod
        const int32x4_t prod = vaddq_s32(vmull_s16(vget_low_s16 (q8sums), vget_low_s16 (mins)),
                                         vmull_s16(vget_high_s16(q8sums), vget_high_s16(mins)));
        # 计算 sumf 减去 dmin 乘以 prod 的和，并将结果存入 sumf
        sumf -= dmin * vaddvq_s32(prod);

        # 将 utmp 强制转换为 uint8_t 类型指针，并赋值给 scales
        const uint8_t * scales = (const uint8_t *)utmp;

        # 将 x[i].qs 强制转换为 uint8_t 类型指针，并赋值给 q4
        const uint8_t * restrict q4 = x[i].qs;
        # 将 y[i].qs 强制转换为 int8_t 类型指针，并赋值给 q8
        const int8_t  * restrict q8 = y[i].qs;

        # 初始化 sumi1 和 sumi2 为 0
        int32_t sumi1 = 0;
        int32_t sumi2 = 0;

        # 循环遍历 QK_K/64 次
        for (int j = 0; j < QK_K/64; ++j) {
            # 从 q4 中加载 32 个字节的数据到 q4bits，并将 q4 指针向后移动 32 个字节
            const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4); q4 += 32;
#ifdef __ARM_FEATURE_DOTPROD
            // 如果支持 ARM Dot Product 指令集
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 从地址 q8 处加载两个 128 位的有符号整数向量
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            // 将两个 128 位的有符号整数向量转换为无符号整数向量，并进行按位与操作
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

            const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
            // 计算两个 128 位的有符号整数向量的点积，并将结果相加后乘以 scales[2*j+0]，并加到 sumi1 上

            sumi1 += vaddvq_s32(p1) * scales[2*j+0];

            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 从地址 q8 处加载两个 128 位的有符号整数向量
            q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
            // 将两个 128 位的有符号整数向量右移 4 位，并转换为无符号整数向量
            q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

            const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
            // 计算两个 128 位的有符号整数向量的点积，并将结果相加后乘以 scales[2*j+1]，并加到 sumi2 上

            sumi2 += vaddvq_s32(p2) * scales[2*j+1];
#else
            // 如果不支持 ARM Dot Product 指令集
            q8bytes = ggml_vld1q_s8_x2(q8); q8 += 32;
            // 从地址 q8 处加载两个 128 位的有符号整数向量
            q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
            // 将两个 128 位的有符号整数向量转换为无符号整数向量，并进行按位与操作
            q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
            const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
            // 将两个 128 位的有符号整数向量的低 64 位进行有符号整数乘法，并将结果相加

```

// 以下代码是使用 ARM NEON 指令集进行优化的计算过程，主要是对两个向量进行乘法和加法操作
// 通过 vmull_s8 函数对两个 int8x8_t 类型的向量进行乘法操作，并将结果存储在 int16x8_t 类型的变量中
// 通过 vaddq_s16 函数对两个 int16x8_t 类型的向量进行加法操作，并将结果存储在 int16x8_t 类型的变量中
// 通过 vaddvq_s16 函数对 int16x8_t 类型的向量进行求和操作，并将结果存储在一个标量中
// 代码中的注释应该根据具体的上下文和变量含义进行解释，这里只是提供了一般性的解释
    *s = sumf;  // 将指针 s 指向的值设置为 sumf

#elif defined __AVX2__  // 如果定义了 __AVX2__ 宏，则执行以下代码

    const __m256i m4 = _mm256_set1_epi8(0xF);  // 创建一个包含 32 个 0xF 的 AVX2 寄存器

    __m256 acc = _mm256_setzero_ps();  // 创建一个包含 8 个 0 的 AVX2 浮点寄存器
    __m128 acc_m = _mm_setzero_ps();  // 创建一个包含 4 个 0 的 SSE 浮点寄存器

   for (int i = 0; i < nb; ++i) {  // 循环，i 从 0 到 nb-1

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);  // 计算 y[i].d 与 x[i].d 乘积
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);  // 计算 -y[i].d 与 x[i].dmin 乘积

        memcpy(utmp, x[i].scales, 12);  // 将 x[i].scales 的前 12 个字节复制到 utmp
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);  // 对 utmp 数组进行位运算
        const uint32_t uaux = utmp[1] & kmask1;  // 计算 utmp[1] 与 kmask1 的按位与
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);  // 对 utmp 数组进行位运算
        utmp[2] = uaux;  // 将 uaux 的值赋给 utmp[2]
        # 将utmp[0]与kmask1进行按位与操作
        utmp[0] &= kmask1;

        # 定义指向x[i].qs的常量指针q4和指向y[i].qs的常量指针q8
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        # 将utmp数组中的值转换成__m256i类型的数据，并存储到mins_and_scales中
        const __m256i mins_and_scales = _mm256_cvtepu8_epi16(_mm_set_epi32(utmp[3], utmp[2], utmp[1], utmp[0]));

        # 从y[i].bsums中加载__m256i类型的数据到q8sums中
        const __m256i q8sums = _mm256_loadu_si256((const __m256i*)y[i].bsums);
        # 将q8sums中的数据进行水平加法，并存储到q8s中
        const __m128i q8s = _mm_hadd_epi16(_mm256_extracti128_si256(q8sums, 0), _mm256_extracti128_si256(q8sums, 1));
        # 将mins_and_scales中的数据与q8s进行乘法累加，并将结果存储到acc_m中
        const __m128i prod = _mm_madd_epi16(_mm256_extracti128_si256(mins_and_scales, 1), q8s);
        acc_m = _mm_fmadd_ps(_mm_set1_ps(dmin), _mm_cvtepi32_ps(prod), acc_m);

        # 从mins_and_scales中提取__m128i类型的数据到sc128中
        const __m128i sc128  = _mm256_extracti128_si256(mins_and_scales, 0);
        # 将sc128中的数据复制到scales中，形成__m256i类型的数据
        const __m256i scales = MM256_SET_M128I(sc128, sc128);

        # 初始化sumi为全0的__m256i类型数据
        __m256i sumi = _mm256_setzero_si256();

        # 循环遍历QK_K/64次
        for (int j = 0; j < QK_K/64; ++j) {
            # 从scales中根据索引获取数据，并进行重新排列，存储到scale_l中
            const __m256i scale_l = _mm256_shuffle_epi8(scales, get_scale_shuffle_k4(2*j+0));
// 使用 scales 中的值对第 2*j+1 个元素进行重新排列
const __m256i scale_h = _mm256_shuffle_epi8(scales, get_scale_shuffle_k4(2*j+1));

// 从内存中加载 32 个字节的数据到 q4bits 中，并将 q4 指针向后移动 32 个字节
const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4); q4 += 32;
// 将 q4bits 中的值与 m4 进行按位与操作，得到 q4l
const __m256i q4l = _mm256_and_si256(q4bits, m4);
// 将 q4bits 中的值右移 4 位，并与 m4 进行按位与操作，得到 q4h
const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

// 从内存中加载 32 个字节的数据到 q8l 中，并将 q8 指针向后移动 32 个字节
const __m256i q8l = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
// 将 q4l 和 q8l 中的值进行有符号 8 位乘法并累加到 p16l 中
__m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
// 将 p16l 中的值与 scale_l 中的值进行有符号 16 位乘法并累加
p16l = _mm256_madd_epi16(scale_l, p16l);

// 从内存中加载 32 个字节的数据到 q8h 中，并将 q8 指针向后移动 32 个字节
const __m256i q8h = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
// 将 q4h 和 q8h 中的值进行有符号 8 位乘法并累加到 p16h 中
__m256i p16h = _mm256_maddubs_epi16(q4h, q8h);
// 将 p16h 中的值与 scale_h 中的值进行有符号 16 位乘法并累加
p16h = _mm256_madd_epi16(scale_h, p16h);
// 将 p16l 和 p16h 中的值进行 32 位整数相加
const __m256i sumj = _mm256_add_epi32(p16l, p16h);

// 将 sumi 和 sumj 中的值进行 32 位整数相加
sumi = _mm256_add_epi32(sumi, sumj);

// 创建一个包含 d 值的 256 位浮点数向量
__m256 vd = _mm256_set1_ps(d);
// 将 vd 与 sumi 中的值进行浮点数乘法并累加到 acc 中
acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(sumi), acc);
    }

    # 使用水平加法指令对两个__m128类型的寄存器进行相加
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    # 使用单精度浮点数水平复制指令对__m128类型的寄存器进行相加
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    # 将累加器acc中的8个单精度浮点数相加，并将结果转换为单精度浮点数
    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

#elif defined __AVX__

    # 创建一个所有元素都为0xF的__m128i类型的寄存器
    const __m128i m4 = _mm_set1_epi8(0xF);
    # 创建一个所有元素都为0x2的__m128i类型的寄存器
    const __m128i m2 = _mm_set1_epi8(0x2);

    # 创建一个所有元素都为0的__m256类型的寄存器
    __m256 acc = _mm256_setzero_ps();
    # 创建一个所有元素都为0的__m128类型的寄存器
    __m128 acc_m = _mm_setzero_ps();

   for (int i = 0; i < nb; ++i) {

        # 计算y[i].d和x[i].d的乘积，并转换为单精度浮点数
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        # 计算-y[i].d和x[i].dmin的乘积，并转换为单精度浮点数
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);
// 从结构体数组 x 中取出第 i 个元素的 qs 字段，赋值给指针 q4
const uint8_t * restrict q4 = x[i].qs;
// 从结构体数组 y 中取出第 i 个元素的 qs 字段，赋值给指针 q8
const int8_t  * restrict q8 = y[i].qs;

// 将结构体数组 x 中第 i 个元素的 scales 字段的内容拷贝到 utmp 数组中
memcpy(utmp, x[i].scales, 12);
// 对 utmp 数组进行位操作和赋值
utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
const uint32_t uaux = utmp[1] & kmask1;
utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
utmp[2] = uaux;
utmp[0] &= kmask1;

// 将 utmp 数组的内容转换成 __m128i 类型的数据
const __m128i utmps = _mm_set_epi32(utmp[3], utmp[2], utmp[1], utmp[0]);
// 将 utmps 转换成 __m128i 类型的数据，并进行零扩展
const __m128i scales = _mm_cvtepu8_epi16(utmps);
// 将 utmps 进行 unpack 操作，并转换成 __m128i 类型的数据
const __m128i mins = _mm_cvtepu8_epi16(_mm_unpackhi_epi64(utmps, utmps));

// 从结构体数组 y 中取出第 i 个元素的 bsums 字段的内容，加载到 __m128i 类型的数据中
const __m128i q8sums_0 = _mm_loadu_si128((const __m128i*)&y[i].bsums[0]);
const __m128i q8sums_1 = _mm_loadu_si128((const __m128i*)&y[i].bsums[8]);
// 对 q8sums_0 和 q8sums_1 进行水平加法
const __m128i q8s = _mm_hadd_epi16(q8sums_0, q8sums_1);
// 对 mins 和 q8s 进行乘法累加
const __m128i prod = _mm_madd_epi16(mins, q8s);
// 将 prod 转换成单精度浮点数类型的数据，并与 dmin 相乘，再与 acc_m 相加
acc_m = _mm_add_ps(_mm_mul_ps(_mm_set1_ps(dmin), _mm_cvtepi32_ps(prod)), acc_m);
# 创建一个全零的 128 位整数寄存器
__m128i sumi_0 = _mm_setzero_si128();
# 创建一个全零的 128 位整数寄存器
__m128i sumi_1 = _mm_setzero_si128();

# 创建一个 16 位整数寄存器，所有元素都是 0x0100
__m128i shuffle = _mm_set1_epi16(0x0100);
# 循环处理每个 64 位块
for (int j = 0; j < QK_K/64; ++j) {

    # 从 scales 寄存器中按照 shuffle 寄存器的索引顺序加载数据
    const __m128i scale_l = _mm_shuffle_epi8(scales, shuffle);
    # 更新 shuffle 寄存器的值
    shuffle = _mm_add_epi16(shuffle, m2);
    # 从 scales 寄存器中按照 shuffle 寄存器的索引顺序加载数据
    const __m128i scale_h = _mm_shuffle_epi8(scales, shuffle);
    # 更新 shuffle 寄存器的值
    shuffle = _mm_add_epi16(shuffle, m2);

    # 从内存中加载 128 位数据到 q4bits 寄存器
    __m128i q4bits = _mm_loadu_si128((const __m128i*)q4); q4 += 16;
    # 对 q4bits 寄存器中的数据进行位与运算
    const __m128i q4l_0 = _mm_and_si128(q4bits, m4);
    # 对 q4bits 寄存器中的数据进行逻辑右移 4 位，然后进行位与运算
    const __m128i q4h_0 = _mm_and_si128(_mm_srli_epi16(q4bits, 4), m4);
    # 从内存中加载 128 位数据到 q4bits 寄存器
    q4bits = _mm_loadu_si128((const __m128i*)q4); q4 += 16;
    # 对 q4bits 寄存器中的数据进行位与运算
    const __m128i q4l_1 = _mm_and_si128(q4bits, m4);
    # 对 q4bits 寄存器中的数据进行逻辑右移 4 位，然后进行位与运算
    const __m128i q4h_1 = _mm_and_si128(_mm_srli_epi16(q4bits, 4), m4);

    # 从内存中加载 128 位数据到 q8l_0 寄存器
    const __m128i q8l_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 SIMD 指令对两个 128 位整数寄存器中的数据进行有符号乘法并相加，结果存储在 p16l 中
__m128i p16l = _mm_maddubs_epi16(q4l_0, q8l_0);
# 使用 SIMD 指令对 p16l 中的数据进行有符号乘法并相加，并乘以 scale_l 中的值
p16l = _mm_madd_epi16(scale_l, p16l);
# 使用 SIMD 指令将 sumi_0 和 p16l 中的数据进行相加
sumi_0 = _mm_add_epi32(sumi_0, p16l);
# 从内存中加载 128 位数据到 q8l_1 中，并将 q8 指针向后移动 16 个字节
const __m128i q8l_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 SIMD 指令对两个 128 位整数寄存器中的数据进行有符号乘法并相加，结果存储在 p16l 中
p16l = _mm_maddubs_epi16(q4l_1, q8l_1);
# 使用 SIMD 指令对 p16l 中的数据进行有符号乘法并相加，并乘以 scale_l 中的值
p16l = _mm_madd_epi16(scale_l, p16l);
# 使用 SIMD 指令将 sumi_1 和 p16l 中的数据进行相加
sumi_1 = _mm_add_epi32(sumi_1, p16l);

# 从内存中加载 128 位数据到 q8h_0 中，并将 q8 指针向后移动 16 个字节
const __m128i q8h_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 SIMD 指令对两个 128 位整数寄存器中的数据进行有符号乘法并相加，结果存储在 p16h 中
__m128i p16h = _mm_maddubs_epi16(q4h_0, q8h_0);
# 使用 SIMD 指令对 p16h 中的数据进行有符号乘法并相加，并乘以 scale_h 中的值
p16h = _mm_madd_epi16(scale_h, p16h);
# 使用 SIMD 指令将 sumi_0 和 p16h 中的数据进行相加
sumi_0 = _mm_add_epi32(sumi_0, p16h);
# 从内存中加载 128 位数据到 q8h_1 中，并将 q8 指针向后移动 16 个字节
const __m128i q8h_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 SIMD 指令对两个 128 位整数寄存器中的数据进行有符号乘法并相加，结果存储在 p16h 中
p16h = _mm_maddubs_epi16(q4h_1, q8h_1);
# 使用 SIMD 指令对 p16h 中的数据进行有符号乘法并相加，并乘以 scale_h 中的值
p16h = _mm_madd_epi16(scale_h, p16h);
# 使用 SIMD 指令将 sumi_1 和 p16h 中的数据进行相加
sumi_1 = _mm_add_epi32(sumi_1, p16h);

# 使用 SIMD 指令将单精度浮点数 d 复制到所有位置，存储在 vd 中
__m256 vd = _mm256_set1_ps(d);
    # 使用两个 128 位整数构建一个 256 位整数
    __m256i sumi = MM256_SET_M128I(sumi_1, sumi_0);
    # 使用单精度浮点数乘法和加法计算结果
    acc = _mm256_add_ps(_mm256_mul_ps(vd, _mm256_cvtepi32_ps(sumi)), acc);

    # 对 acc_m 进行一系列的单精度浮点数加法操作
    acc_m = _mm_add_ps(acc_m, _mm_movehl_ps(acc_m, acc_m));
    acc_m = _mm_add_ss(acc_m, _mm_movehdup_ps(acc_m));

    # 计算最终结果并存储
    *s = hsum_float_8(acc) + _mm_cvtss_f32(acc_m);

# 如果定义了 __riscv_v_intrinsic
#elif defined __riscv_v_intrinsic
    # 将 utmp 数组中的数据转换为指向 uint8_t 类型的指针
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    # 初始化 sumf 变量
    float sumf = 0;

    # 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        # 设置 vl 变量的值为 8
        size_t vl = 8;
        // 计算浮点数乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算浮点数乘积的最小值
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 从内存中加载一组16位整数到向量寄存器q8sums_0和q8sums_1中
        vint16mf2_t q8sums_0 = __riscv_vlse16_v_i16mf2(y[i].bsums, 4, vl);
        vint16mf2_t q8sums_1 = __riscv_vlse16_v_i16mf2(y[i].bsums+1, 4, vl);
        // 对两个向量寄存器中的数据进行相加
        vint16mf2_t q8sums   = __riscv_vadd_vv_i16mf2(q8sums_0, q8sums_1, vl);

        // 将x[i].scales中的数据复制到utmp中
        memcpy(utmp, x[i].scales, 12);
        // 对utmp中的数据进行位操作
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;

        // 从内存中加载一组8位无符号整数到向量寄存器mins8中
        vuint8mf4_t mins8  = __riscv_vle8_v_u8mf4(mins, vl);
        // 将mins8中的数据进行零扩展和类型转换，存储到v_mins中
        vint16mf2_t v_mins = __riscv_vreinterpret_v_u16mf2_i16mf2(__riscv_vzext_vf2_u16mf2(mins8, vl));
        // 对两个向量寄存器中的数据进行乘法
        vint32m1_t  prod   = __riscv_vwmul_vv_i32m1(q8sums, v_mins, vl);

        // 对向量寄存器中的数据进行累加求和
        vint32m1_t sumi = __riscv_vredsum_vs_i32m1_i32m1(prod, __riscv_vmv_v_x_i32m1(0, 1), vl);
        // 用最小值乘以sumi，然后从sumf中减去
        sumf -= dmin * __riscv_vmv_x_s_i32m1_i32(sumi);

        // 定义指向x[i].qs的指针q4和指向y[i].qs的指针q8
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 设置向量长度为32
        vl = 32;

        // 初始化两个用于存储求和结果的变量
        int32_t sum_1 = 0;
        int32_t sum_2 = 0;

        // 初始化一个全零的向量vzero
        vint16m1_t vzero = __riscv_vmv_v_x_i16m1(0, 1);

        // 循环遍历QK_K/64次
        for (int j = 0; j < QK_K/64; ++j) {
            // 加载Q4
            vuint8m1_t q4_x = __riscv_vle8_v_u8m1(q4, vl);

            // 加载Q8并将其与低Q4半字节相乘
            vint8m1_t  q8_0 = __riscv_vle8_v_i8m1(q8, vl);
            vint8m1_t  q4_0 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(q4_x, 0x0F, vl));
            vint16m2_t qv_0 = __riscv_vwmul_vv_i16m2(q4_0, q8_0, vl);
// 使用指定的二进制位宽和长度对两个向量进行有符号整数乘法和求和操作
vint16m1_t vs_0 = __riscv_vredsum_vs_i16m2_i16m1(qv_0, vzero, vl);

// 将求和结果乘以预先定义的比例因子，并加到总和中
sum_1 += __riscv_vmv_x_s_i16m1_i16(vs_0) * scales[2*j+0];

// 加载 Q8 并将其与上半部分的 Q4 进行乘法
vint8m1_t  q8_1 = __riscv_vle8_v_i8m1(q8+32, vl);
vint8m1_t  q4_1 = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vsrl_vx_u8m1(q4_x, 0x04, vl));
vint16m2_t qv_1 = __riscv_vwmul_vv_i16m2(q4_1, q8_1, vl);
vint16m1_t vs_1 = __riscv_vredsum_vs_i16m2_i16m1(qv_1, vzero, vl);

// 将求和结果乘以预先定义的比例因子，并加到总和中
sum_2 += __riscv_vmv_x_s_i16m1_i16(vs_1) * scales[2*j+1];

// 更新 Q4 和 Q8 的指针位置
q4 += 32;    q8 += 64;

// 将两个求和结果乘以权重因子并加到总和中
sumf += d*(sum_1 + sum_2);
    *s = sumf;
    // 将指针 s 指向的值设置为 sumf 的值

#else
    // 如果不满足条件，则执行以下代码

    const uint8_t * scales = (const uint8_t*)&utmp[0];
    // 创建指向 utmp 数组第一个元素的 uint8_t 类型指针，并命名为 scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];
    // 创建指向 utmp 数组第三个元素的 uint8_t 类型指针，并命名为 mins

    int8_t  aux8[QK_K];
    // 创建长度为 QK_K 的 int8_t 类型数组 aux8
    int16_t aux16[8];
    // 创建长度为 8 的 int16_t 类型数组 aux16
    float   sums [8];
    // 创建长度为 8 的 float 类型数组 sums
    int32_t aux32[8];
    // 创建长度为 8 的 int32_t 类型数组 aux32
    memset(sums, 0, 8*sizeof(float));
    // 将 sums 数组的前 8 个元素设置为 0

    float sumf = 0;
    // 创建 float 类型变量 sumf，并初始化为 0
    for (int i = 0; i < nb; ++i) {
        // 循环，i 从 0 到 nb-1
        const uint8_t * restrict q4 = x[i].qs;
        // 创建指向 x[i].qs 的 uint8_t 类型指针，并命名为 q4
        const  int8_t * restrict q8 = y[i].qs;
        // 创建指向 y[i].qs 的 int8_t 类型指针，并命名为 q8
        memset(aux32, 0, 8*sizeof(int32_t));
        // 将 aux32 数组的前 8 个元素设置为 0
        int8_t * restrict a = aux8;
        // 创建指向 aux8 数组的 int8_t 类型指针，并命名为 a
        // 循环遍历QK_K/64次
        for (int j = 0; j < QK_K/64; ++j) {
            // 将q4数组中的每个元素的低4位赋值给a数组
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l] & 0xF);
            // a指针向后移动32位
            a += 32;
            // 将q4数组中的每个元素的高4位赋值给a数组
            for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l]  >> 4);
            // a指针再向后移动32位，q4指针也向后移动32位
            a += 32; q4 += 32;
        }
        // 将x[i].scales数组的前12个字节拷贝到utmp数组
        memcpy(utmp, x[i].scales, 12);
        // 对utmp数组进行位运算操作
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;

        // 初始化sumi为0
        int sumi = 0;
        // 循环遍历QK_K/16次，累加y[i].bsums[j] * mins[j/2]的值
        for (int j = 0; j < QK_K/16; ++j) sumi += y[i].bsums[j] * mins[j/2];
        // 将a指针重新指向aux8数组
        a = aux8;
        // 初始化is为0
        int is = 0;
        // 循环遍历QK_K/32次
        for (int j = 0; j < QK_K/32; ++j) {
            // 将scales数组中的元素赋值给scale
            int32_t scale = scales[is++];
            // 循环遍历8次，计算q8[l] * a[l]的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
# 对辅助数组 aux32 进行更新，加上辅助数组 aux16 乘以 scale 的结果
for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
# 更新 q8 和 a 的指针位置，移动到下一个位置
q8 += 8; a += 8;
# 对辅助数组 aux16 进行更新，乘以 q8 和 a 的对应元素
for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
# 对辅助数组 aux32 进行更新，加上辅助数组 aux16 乘以 scale 的结果
for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
# 更新 q8 和 a 的指针位置，移动到下一个位置
q8 += 8; a += 8;
# 对辅助数组 aux16 进行更新，乘以 q8 和 a 的对应元素
for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
# 对辅助数组 aux32 进行更新，加上辅助数组 aux16 乘以 scale 的结果
for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
# 更新 q8 和 a 的指针位置，移动到下一个位置
q8 += 8; a += 8;
# 计算 d，将 x[i].d 转换为单精度浮点数后乘以 y[i].d
const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
# 对 sums 数组进行更新，加上 d 乘以 aux32 的对应元素
for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
# 计算 dmin，将 x[i].dmin 转换为单精度浮点数后乘以 y[i].d
const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;
# 更新 sumf，减去 dmin 乘以 sumi
sumf -= dmin * sumi;
# 对 sumf 进行更新，加上 sums 的所有元素之和
for (int l = 0; l < 8; ++l) sumf += sums[l];
# 将最终结果赋值给指针 s 指向的位置
*s = sumf;
}
#else
// 定义一个函数，计算两个向量的点积
void ggml_vec_dot_q4_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言，确保 n 是 QK_K 的倍数
    assert(n % QK_K == 0);

    // 将 vx 和 vy 强制转换为 block_q4_K 和 block_q8_K 类型的指针
    const block_q4_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 创建一个包含 16 个相同值的向量，值为 0xf
    const uint8x16_t m4b = vdupq_n_u8(0xf);

#ifdef __ARM_FEATURE_DOTPROD
    // 创建一个包含 4 个相同值的向量，值为 0
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 初始化一个浮点数变量 sumf 为 0
    float sumf = 0;
    # 定义一个包含两个 16 位整数的向量
    ggml_int8x16x2_t q4bytes;
    # 定义一个包含四个 16 位整数的向量
    ggml_int8x16x4_t q8bytes;

    # 定义一个浮点数变量 sum_mins，并初始化为 0
    float sum_mins = 0.f;

    # 定义一个包含两个 16 位无符号整数的数组
    uint16_t aux16[2];
    # 将 aux16 强制转换为 uint8_t 类型的指针，并赋值给 scales
    const uint8_t * restrict scales = (const uint8_t *)aux16;

    # 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        # 定义一个指向 x[i].qs 的限定指针，并赋值给 q4
        const uint8_t * restrict q4 = x[i].qs;
        # 定义一个指向 y[i].qs 的限定指针，并赋值给 q8
        const int8_t  * restrict q8 = y[i].qs;

        # 将 x[i].scales 强制转换为 uint16_t 类型的指针，并赋值给 a
        const uint16_t * restrict a = (const uint16_t *)x[i].scales;
        # 将 a[0] 的低 8 位与 0x0f0f 进行按位与操作，并赋值给 aux16[0]
        aux16[0] = a[0] & 0x0f0f;
        # 将 a[0] 右移 4 位后的低 8 位与 0x0f0f 进行按位与操作，并赋值给 aux16[1]

        # 计算 summi 的值
        const int32_t summi = scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]);
        # 计算 sum_mins 的值
        sum_mins += y[i].d * (float)x[i].d[1] * summi;
        // 计算浮点数 d，等于 y[i] 的浮点数乘以 x[i] 的第一个元素
        const float d = y[i].d * (float)x[i].d[0];

        // 从内存中加载两个 128 位的 8 位无符号整数向量，存储在 q4bits 中
        const ggml_uint8x16x2_t q4bits = ggml_vld1q_u8_x2(q4);

#ifdef __ARM_FEATURE_DOTPROD
        // 从内存中加载四个 128 位的 8 位有符号整数向量，存储在 q8bytes 中
        q8bytes = ggml_vld1q_s8_x4(q8);
        // 将 q4bits 中的值与 m4b 进行按位与操作，并将结果转换为有符号整数向量，存储在 q4bytes 中
        q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
        q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));

        // 计算两个 128 位有符号整数向量的点积，并将结果乘以 scales[0]，存储在 sumi1 中
        const int32x4_t p1 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[0]), q4bytes.val[1], q8bytes.val[1]);
        const int32_t sumi1 = vaddvq_s32(p1) * scales[0];

        // 将 q4bits 中的值右移 4 位，并将结果转换为有符号整数向量，存储在 q4bytes 中
        q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
        q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));

        // 计算两个 128 位有符号整数向量的点积，并将结果乘以 scales[1]，存储在 sumi2 中
        const int32x4_t p2 = vdotq_s32(vdotq_s32(mzero, q4bytes.val[0], q8bytes.val[2]), q4bytes.val[1], q8bytes.val[3]);
        const int32_t sumi2 = vaddvq_s32(p2) * scales[1];

#else
        // 从内存中加载四个 128 位的 8 位有符号整数向量，存储在 q8bytes 中
        q8bytes = ggml_vld1q_s8_x4(q8);
# 将q4bits.val[0]和m4b进行按位与操作，并将结果转换为有符号8位整数，存储在q4bytes.val[0]中
q4bytes.val[0] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[0], m4b));
# 将q4bits.val[1]和m4b进行按位与操作，并将结果转换为有符号8位整数，存储在q4bytes.val[1]中
q4bytes.val[1] = vreinterpretq_s8_u8(vandq_u8  (q4bits.val[1], m4b));
# 计算p0，将q4bytes.val[0]和q8bytes.val[0]的低位和高位分别进行有符号8位整数乘法，并将结果相加
const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                               vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[0])));
# 计算p1，将q4bytes.val[1]和q8bytes.val[1]的低位和高位分别进行有符号8位整数乘法，并将结果相加
const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                               vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[1])));
# 计算sumi1，将p0和p1相加后乘以scales[0]的值
int32_t sumi1 = vaddvq_s16(vaddq_s16(p0, p1)) * scales[0];

# 将q4bits.val[0]右移4位，并将结果转换为有符号8位整数，存储在q4bytes.val[0]中
q4bytes.val[0] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[0], 4));
# 将q4bits.val[1]右移4位，并将结果转换为有符号8位整数，存储在q4bytes.val[1]中
q4bytes.val[1] = vreinterpretq_s8_u8(vshrq_n_u8(q4bits.val[1], 4));
# 计算p2，将q4bytes.val[0]和q8bytes.val[2]的低位和高位分别进行有符号8位整数乘法，并将结果相加
const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[0]), vget_low_s8 (q8bytes.val[2])),
                               vmull_s8(vget_high_s8(q4bytes.val[0]), vget_high_s8(q8bytes.val[2]));
# 计算p3，将q4bytes.val[1]和q8bytes.val[3]的低位和高位分别进行有符号8位整数乘法，并将结果相加
const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q4bytes.val[1]), vget_low_s8 (q8bytes.val[3])),
                               vmull_s8(vget_high_s8(q4bytes.val[1]), vget_high_s8(q8bytes.val[3]));
# 计算sumi2，将p2和p3相加后乘以scales[1]的值
int32_t sumi2 = vaddvq_s16(vaddq_s16(p2, p3)) * scales[1];
# 将d乘以(sumi1 + sumi2)的值，并加到sumf上
sumf += d * (sumi1 + sumi2);
    *s = sumf - sum_mins;
    // 计算 sumf 减去 sum_mins 的结果，并赋值给指针 s

#elif defined __AVX2__
    // 如果定义了 __AVX2__ 宏，则执行以下代码

    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 创建一个包含 32 个 0xF 的 256 位整数寄存器

    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含 8 个 0.0 的 256 位单精度浮点数寄存器

    float summs = 0;
    // 初始化 summs 为 0

    uint16_t aux16[2];
    // 创建一个包含 2 个 16 位无符号整数的数组
    const uint8_t * scales = (const uint8_t *)aux16;
    // 将 aux16 强制转换为指向 8 位无符号整数的指针，并赋值给 scales

    for (int i = 0; i < nb; ++i) {
        // 循环，i 从 0 到 nb-1

        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        // 计算 x[i].d[0] 转换为单精度浮点数后乘以 y[i].d 的结果，并赋值给 d
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        // 计算 x[i].d[1] 转换为单精度浮点数后乘以 y[i].d 的结果，并赋值给 m
        const __m256 vd = _mm256_set1_ps(d);
        // 创建一个包含 8 个 d 的 256 位单精度浮点数寄存器
        // 将x[i].scales强制转换为const uint16_t指针类型，并赋值给a
        const uint16_t * a = (const uint16_t *)x[i].scales;
        // 将a[0]的低8位与0x0f0f进行按位与操作，并赋值给aux16[0]
        aux16[0] = a[0] & 0x0f0f;
        // 将a[0]右移4位后的低8位与0x0f0f进行按位与操作，并赋值给aux16[1]
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算summs的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 将x[i].qs强制转换为const uint8_t指针类型，并赋值给q4
        const uint8_t * restrict q4 = x[i].qs;
        // 将y[i].qs强制转换为const int8_t指针类型，并赋值给q8
        const int8_t  * restrict q8 = y[i].qs;

        // 加载q4指向的数据到__m256i类型的寄存器q4bits
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        // 将q4bits中的数据与m4进行按位与操作，并赋值给q4l
        const __m256i q4l = _mm256_and_si256(q4bits, m4);
        // 将q4bits中的数据右移4位后与m4进行按位与操作，并赋值给q4h
        const __m256i q4h = _mm256_and_si256(_mm256_srli_epi16(q4bits, 4), m4);

        // 加载q8指向的数据到__m256i类型的寄存器q8l
        const __m256i q8l = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        // 加载q8指向的数据到__m256i类型的寄存器q8h
        const __m256i q8h = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 计算p16l的值
        const __m256i p16l = _mm256_maddubs_epi16(q4l, q8l);
        // 计算p16h的值
        const __m256i p16h = _mm256_maddubs_epi16(q4h, q8h);

        // 计算p32l的值
        const __m256i p32l = _mm256_madd_epi16(_mm256_set1_epi16(scales[0]), p16l);
        # 使用 AVX 指令集计算两个 256 位浮点数向量的乘积并加上第三个 256 位浮点数向量
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32l), acc);

        # 使用 AVX 指令集计算两个 256 位整数向量的乘积并加上第三个 256 位整数向量
        const __m256i p32h = _mm256_madd_epi16(_mm256_set1_epi16(scales[1]), p16h);
        acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(p32h), acc);

    }

    # 计算累加器中 8 个单精度浮点数的和，并减去 summs
    *s = hsum_float_8(acc) - summs;

#elif defined __AVX__

    # 创建一个 128 位整数向量，所有元素都是 0xF
    const __m128i m4 = _mm_set1_epi8(0xF);

    # 创建一个 256 位浮点数向量，所有元素都是 0
    __m256 acc = _mm256_setzero_ps();

    # 初始化一个浮点数变量 summs 为 0
    float summs = 0;

    # 创建一个包含两个 16 位无符号整数的数组 aux16，并将其转换为 uint8_t 类型指针
    uint16_t aux16[2];
    const uint8_t * scales = (const uint8_t *)aux16;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算浮点数 d 和 m
        const float d = GGML_FP16_TO_FP32(x[i].d[0]) * y[i].d;
        const float m = GGML_FP16_TO_FP32(x[i].d[1]) * y[i].d;
        // 创建包含 d 的 __m256 向量
        const __m256 vd = _mm256_set1_ps(d);

        // 从 x[i].scales 中提取数据并存储到 aux16 数组中
        const uint16_t * a = (const uint16_t *)x[i].scales;
        aux16[0] = a[0] & 0x0f0f;
        aux16[1] = (a[0] >> 4) & 0x0f0f;

        // 计算 summs 的值
        summs += m * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 限制指针 q4 和 q8 的访问范围
        const uint8_t * restrict q4 = x[i].qs;
        const int8_t  * restrict q8 = y[i].qs;

        // 加载 q4 中的数据到 __m256i 向量 q4bits
        const __m256i q4bits = _mm256_loadu_si256((const __m256i*)q4);
        // 从 q4bits 中提取数据到两个 __m128i 向量
        const __m128i q4bits_0 = _mm256_extractf128_si256(q4bits, 0);
        const __m128i q4bits_1 = _mm256_extractf128_si256(q4bits, 1);
        // 对两个 __m128i 向量进行按位与操作
        const __m128i q4_0 = _mm_and_si128(q4bits_0, m4);
        const __m128i q4_1 = _mm_and_si128(q4bits_1, m4);
        # 将 q4bits_0 右移 4 位并与 m4 进行按位与操作，结果存储在 q4_2 中
        const __m128i q4_2 = _mm_and_si128(_mm_srli_epi16(q4bits_0, 4), m4);
        # 将 q4bits_1 右移 4 位并与 m4 进行按位与操作，结果存储在 q4_3 中
        const __m128i q4_3 = _mm_and_si128(_mm_srli_epi16(q4bits_1, 4), m4);

        # 从内存中加载 q8 的值到 q8_0 中
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        # 从内存中加载 q8 的值到 q8_1 中
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 将 q4_0 与 q8_0 的低 128 位进行乘积累加，结果存储在 p16_0 中
        const __m128i p16_0 = _mm_maddubs_epi16(q4_0, _mm256_extractf128_si256(q8_0, 0));
        # 将 q4_1 与 q8_0 的高 128 位进行乘积累加，结果存储在 p16_1 中
        const __m128i p16_1 = _mm_maddubs_epi16(q4_1, _mm256_extractf128_si256(q8_0, 1));
        # 将 q4_2 与 q8_1 的低 128 位进行乘积累加，结果存储在 p16_2 中
        const __m128i p16_2 = _mm_maddubs_epi16(q4_2, _mm256_extractf128_si256(q8_1, 0));
        # 将 q4_3 与 q8_1 的高 128 位进行乘积累加，结果存储在 p16_3 中
        const __m128i p16_3 = _mm_maddubs_epi16(q4_3, _mm256_extractf128_si256(q8_1, 1));

        # 将 p16_0 乘以 scales[0] 并累加，结果存储在 p32_0 中
        const __m128i p32_0 = _mm_madd_epi16(_mm_set1_epi16(scales[0]), p16_0);
        # 将 p16_1 乘以 scales[0] 并累加，结果存储在 p32_1 中
        const __m128i p32_1 = _mm_madd_epi16(_mm_set1_epi16(scales[0]), p16_1);
        # 将 p32_1 和 p32_0 转换成单精度浮点数并与 vd 进行乘法累加，结果存储在 acc 中
        acc = _mm256_add_ps(_mm256_mul_ps(vd, _mm256_cvtepi32_ps(MM256_SET_M128I(p32_1, p32_0))), acc);

        # 将 p16_2 乘以 scales[1] 并累加，结果存储在 p32_2 中
        const __m128i p32_2 = _mm_madd_epi16(_mm_set1_epi16(scales[1]), p16_2);
        # 将 p16_3 乘以 scales[1] 并累加，结果存储在 p32_3 中
        const __m128i p32_3 = _mm_madd_epi16(_mm_set1_epi16(scales[1]), p16_3);
        # 将 p32_3 和 p32_2 转换成单精度浮点数并与 vd 进行乘法累加，结果存储在 acc 中
        acc = _mm256_add_ps(_mm256_mul_ps(vd, _mm256_cvtepi32_ps(MM256_SET_M128I(p32_3, p32_2))), acc);
    *s = hsum_float_8(acc) - summs; 
    // 计算指针s指向的值，等于acc的8位浮点数和summs的差值

#elif defined __riscv_v_intrinsic
    // 如果定义了__riscv_v_intrinsic
    uint16_t s16[2];
    // 声明一个长度为2的uint16_t数组s16
    const uint8_t * restrict scales = (const uint8_t *)s16;
    // 声明一个指向s16的const uint8_t指针scales

    float sumf = 0;
    // 声明一个浮点数sumf并初始化为0

    for (int i = 0; i < nb; ++i) {
        // 循环，i从0到nb-1
        const uint8_t * restrict q4 = x[i].qs;
        // 声明一个指向x[i].qs的const uint8_t指针q4
        const  int8_t * restrict q8 = y[i].qs;
        // 声明一个指向y[i].qs的const int8_t指针q8

        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        // 声明一个指向x[i].scales的const uint16_t指针b，并将其转换为const uint16_t类型
        s16[0] = b[0] & 0x0f0f;
        // 将b[0]与0x0f0f进行按位与操作，并将结果赋值给s16[0]
        s16[1] = (b[0] >> 4) & 0x0f0f;
        // 将b[0]右移4位，然后与0x0f0f进行按位与操作，并将结果赋值给s16[1]

        sumf -= y[i].d * GGML_FP16_TO_FP32(x[i].d[1]) * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));
        // 计算sumf的值，等于sumf减去y[i].d乘以GGML_FP16_TO_FP32(x[i].d[1])乘以(scales[2]乘以(y[i].bsums[0]加上y[i].bsums[1])加上scales[3]乘以(y[i].bsums[2]加上y[i].bsums[3]))
        // 计算乘积并存储在浮点数变量d中
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d[0]);

        // 设置向量长度为32
        size_t vl = 32;

        // 创建一个全零的向量vzero
        vint16m1_t vzero = __riscv_vmv_v_x_i16m1(0, 1);

        // 加载Q4向量
        vuint8m1_t q4_x = __riscv_vle8_v_u8m1(q4, vl); // 加载Q4向量

        // 加载Q8向量并将其与低Q4半字节相乘
        vint8m1_t  q4_a = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(q4_x, 0x0F, vl)); // 将Q4向量与0x0F进行与操作
        vint16m2_t va_0 = __riscv_vwmul_vv_i16m2(q4_a, __riscv_vle8_v_i8m1(q8, vl), vl); // 将Q4_a与Q8相乘
        vint16m1_t aux1 = __riscv_vredsum_vs_i16m2_i16m1(va_0, vzero, vl); // 对va_0进行求和

        // 将d乘以scales[0]和aux1的值加到sumf中
        sumf += d*scales[0]*__riscv_vmv_x_s_i16m1_i16(aux1);

        // 加载Q8向量并将其与高Q4半字节相乘
        vint8m1_t  q4_s = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vsrl_vx_u8m1(q4_x, 0x04, vl)); // 将Q4向量右移4位
        vint16m2_t va_1 = __riscv_vwmul_vv_i16m2(q4_s, __riscv_vle8_v_i8m1(q8+32, vl), vl); // 将Q4_s与Q8+32相乘
        vint16m1_t aux2 = __riscv_vredsum_vs_i16m2_i16m1(va_1, vzero, vl); // 对va_1进行求和
    // 如果定义了__riscv_vector变量，则执行以下代码
    sumf += d * scales[1] * __riscv_vmv_x_s_i16m1_i16(aux2);

    // 将结果存储到指针s所指向的位置
    *s = sumf;

#else

    // 定义辅助数组和变量
    uint8_t aux8[QK_K];
    int16_t aux16[16];
    float sums[8];
    // 将sums数组清零
    memset(sums, 0, 8 * sizeof(float));

    // 定义辅助数组和变量
    uint16_t s16[2];
    const uint8_t *restrict scales = (const uint8_t *)s16;

    // 定义浮点数变量并初始化为0
    float sumf = 0;
    // 遍历nb次循环
    for (int i = 0; i < nb; ++i) {
        // 定义指向x[i].qs的指针
        const uint8_t *restrict q4 = x[i].qs;
        // 声明一个指向 y[i].qs 的常量指针 q8
        const int8_t * restrict q8 = y[i].qs;
        // 声明一个指向 aux8 的指针 a
        uint8_t * restrict a = aux8;
        // 将 q4 中的前 32 位数据的低 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+ 0] = q4[l] & 0xF;
        // 将 q4 中的前 32 位数据的高 4 位存入 a 中
        for (int l = 0; l < 32; ++l) a[l+32] = q4[l] >> 4;

        // 声明一个指向 x[i].scales 的常量指针 b
        const uint16_t * restrict b = (const uint16_t *)x[i].scales;
        // 将 b[0] 中的低 8 位数据存入 s16[0]
        s16[0] = b[0] & 0x0f0f;
        // 将 b[0] 中的高 8 位数据存入 s16[1]
        s16[1] = (b[0] >> 4) & 0x0f0f;

        // 计算 sumf 的值
        sumf -= y[i].d * GGML_FP16_TO_FP32(x[i].d[1]) * (scales[2] * (y[i].bsums[0] + y[i].bsums[1]) + scales[3] * (y[i].bsums[2] + y[i].bsums[3]));

        // 声明一个常量 d，存储 y[i].d 与 GGML_FP16_TO_FP32(x[i].d[0]) 的乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d[0]);

        // 循环计算 sums 数组的值
        for (int j = 0; j < QK_K/32; ++j) {
            // 计算 aux16 数组的值
            for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
            q8 += 16; a += 16;
            for (int l = 0; l < 16; ++l) aux16[l] += q8[l] * a[l];
            q8 += 16; a += 16;
            // 计算 dl 的值
            const float dl = d * scales[j];
            // 计算 sums 数组的值
            for (int l = 0; l < 8; ++l) sums[l] += dl * (aux16[l] + aux16[l+8]);
    }
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    *s = sumf;
#endif
}
#endif

#if QK_K == 256
// 计算两个向量的点积，其中向量长度为 n，结果存储在 s 中
void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言向量长度是 QK_K 的整数倍
    assert(n % QK_K == 0);

    // 将输入指针转换为特定类型的指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度为 QK_K 时的块数
    const int nb = n / QK_K;

    // 定义用于按位与操作的掩码
    static const uint32_t kmask1 = 0x3f3f3f3f;
    static const uint32_t kmask2 = 0x0f0f0f0f;
    static const uint32_t kmask3 = 0x03030303;
    // 定义一个包含4个32位无符号整数的数组
    uint32_t utmp[4];

    // 如果支持 ARM NEON 指令集
#ifdef __ARM_NEON
    // 创建一个包含16个8位无符号整数的向量，每个元素都是0xf
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    // 创建一个包含16个8位无符号整数的向量，每个元素都是1
    const uint8x16_t mone = vdupq_n_u8(1);
    // 创建一个包含16个8位无符号整数的向量，每个元素都是2
    const uint8x16_t mtwo = vdupq_n_u8(2);
    // 如果支持 ARM Dot Product 指令集
#if defined(__ARM_FEATURE_DOTPROD)
    // 创建一个包含4个32位有符号整数的向量，每个元素都是0
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 定义一个包含4个8位整数向量的结构体
    ggml_int8x16x4_t q5bytes;

    // 定义一个浮点数变量，用于存储累加和
    float sumf = 0;

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 计算 y[i].d 与 x[i].d 的乘积，并存储在 d 中
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算dmin，y[i].d乘以x[i].dmin的FP16到FP32的转换结果
        const float dmin = y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 使用SIMD指令计算q8sums，将y[i].bsums加载到寄存器中并进行累加
        const int16x8_t q8sums = vpaddq_s16(vld1q_s16(y[i].bsums), vld1q_s16(y[i].bsums + 8));

        // 复制x[i].scales的前12个字节到utmp数组
        memcpy(utmp, x[i].scales, 12);
        // 对utmp数组进行位操作和移位运算
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;

        // 使用SIMD指令加载utmp数组的后8个字节到mins8，并将其转换为int16x8_t类型的mins
        const uint8x8_t mins8 = vld1_u8((const uint8_t*)utmp + 8);
        const int16x8_t mins = vreinterpretq_s16_u16(vmovl_u8(mins8));
        // 使用SIMD指令计算prod，q8sums和mins的逐元素乘积并累加
        const int32x4_t prod = vaddq_s32(vmull_s16(vget_low_s16 (q8sums), vget_low_s16 (mins)),
                                         vmull_s16(vget_high_s16(q8sums), vget_high_s16(mins)));
        // 使用SIMD指令将prod的元素累加得到sumi_mins
        int32_t sumi_mins = vaddvq_s32(prod);

        // 将utmp数组转换为uint8_t类型的scales指针
        const uint8_t * scales = (const uint8_t *)utmp;

        // 将x[i].qs转换为const uint8_t类型的指针q5
        const uint8_t * restrict q5 = x[i].qs;
// 从数组 x 中取出第 i 个元素的地址，并限定其为只读
const uint8_t * restrict qh = x[i].qh;
// 从数组 y 中取出第 i 个元素的地址，并限定其为只读
const int8_t  * restrict q8 = y[i].qs;

// 从地址 qh 加载两个 16 字节的无符号整数向量
ggml_uint8x16x2_t qhbits = ggml_vld1q_u8_x2(qh);

// 声明一个包含四个 16 字节的无符号整数向量的变量
ggml_uint8x16x4_t q5h;

// 声明一个 32 位整数变量 sumi，并初始化为 0
int32_t sumi = 0;

// 循环遍历 QK_K/64 次
for (int j = 0; j < QK_K/64; ++j) {
    // 从地址 q5 加载两个 16 字节的无符号整数向量，并将地址后移 32 字节
    const ggml_uint8x16x2_t q5bits = ggml_vld1q_u8_x2(q5); q5 += 32;
    // 从地址 q8 加载四个 16 字节的有符号整数向量，并将地址后移 64 字节
    const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8); q8 += 64;

    // 对 qhbits 中的数据进行位运算和移位操作，并存储到 q5h 中
    q5h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits.val[0]), 4);
    q5h.val[1] = vshlq_n_u8(vandq_u8(mone, qhbits.val[1]), 4);
    q5h.val[2] = vshlq_n_u8(vandq_u8(mtwo, qhbits.val[0]), 3);
    q5h.val[3] = vshlq_n_u8(vandq_u8(mtwo, qhbits.val[1]), 3);
    // 对 qhbits 中的数据进行右移位操作
    qhbits.val[0] = vshrq_n_u8(qhbits.val[0], 2);
    qhbits.val[1] = vshrq_n_u8(qhbits.val[1], 2);
}
// 将 q5bits.val[0] 和 m4b 进行按位与操作，然后与 q5h.val[0] 进行按位或操作，最后将结果转换为字节类型
q5bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q5bits.val[0], m4b), q5h.val[0]));

// 将 q5bits.val[1] 和 m4b 进行按位与操作，然后与 q5h.val[1] 进行按位或操作，最后将结果转换为字节类型
q5bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q5bits.val[1], m4b), q5h.val[1]));

// 将 q5bits.val[0] 右移 4 位，然后与 q5h.val[2] 进行按位或操作，最后将结果转换为字节类型
q5bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q5bits.val[0], 4), q5h.val[2]));

// 将 q5bits.val[1] 右移 4 位，然后与 q5h.val[3] 进行按位或操作，最后将结果转换为字节类型
q5bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q5bits.val[1], 4), q5h.val[3]));

// 如果支持 ARM dot product 指令集
sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]), q5bytes.val[1], q8bytes.val[1])) * *scales++;
sumi += vaddvq_s32(vdotq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]), q5bytes.val[3], q8bytes.val[3])) * *scales++;
// 否则
const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                               vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0])));
const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                               vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1])));
sumi += vaddvq_s16(vaddq_s16(p0, p1)) * *scales++;

const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                               vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2]));
    // 定义一个常量 int16x8_t 类型的变量 p3，其值为两个 int8x8_t 类型向量的乘积之和
    const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                   vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3])));
    // 将 p2 和 p3 的和相加，并乘以指针所指的值，然后累加到 sumi 上
    sumi += vaddvq_s16(vaddq_s16(p2, p3)) * *scales++;
#endif
}

// 计算 sumf 的值
sumf += d * sumi - dmin * sumi_mins;

}

// 将 sumf 的值赋给指针所指的位置
*s = sumf;

#elif defined __AVX2__

// 定义一些 AVX2 指令集需要使用的常量和变量
const __m256i m4 = _mm256_set1_epi8(0xF);
const __m128i mzero = _mm_setzero_si128();
const __m256i mone  = _mm256_set1_epi8(1);
// 初始化一个 256 位的浮点型累加器，所有元素的值都为 0
__m256 acc = _mm256_setzero_ps();
    # 初始化浮点数变量 summs
    float summs = 0.f;

    # 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        # 声明并初始化指向 x[i].qs 的指针 q5
        const uint8_t * restrict q5 = x[i].qs;
        # 声明并初始化指向 y[i].qs 的指针 q8
        const int8_t  * restrict q8 = y[i].qs;

        # 如果 QK_K 等于 256，则执行以下代码块
        # 计算 d 和 dmin 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        # 复制 x[i].scales 的内容到 utmp
        memcpy(utmp, x[i].scales, 12);
        # 对 utmp 数组进行位操作
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        const uint32_t uaux = utmp[1] & kmask1;
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        utmp[2] = uaux;
        utmp[0] &= kmask1;
    # 如果 QK_K 不等于 256，则执行以下代码块
    # 设置 d 和 dmin 的值为 0
    else
        const float d = 0, dmin = 0;
// 定义一个常量__m256i类型的变量mins_and_scales，用于存储转换后的无符号8位整数
const __m256i mins_and_scales = _mm256_cvtepu8_epi16(_mm_set_epi32(utmp[3], utmp[2], utmp[1], utmp[0]));

// 从y[i].bsums中加载数据到常量__m256i类型的变量q8sums
const __m256i q8sums = _mm256_loadu_si256((const __m256i*)y[i].bsums);

// 对q8sums中的数据进行水平加法，存储到常量__m128i类型的变量q8s
const __m128i q8s = _mm_hadd_epi16(_mm256_extracti128_si256(q8sums, 0), _mm256_extracti128_si256(q8sums, 1));

// 对mins_and_scales和q8s中的数据进行乘法和加法操作，存储到常量__m128i类型的变量prod
const __m128i prod = _mm_madd_epi16(_mm256_extracti128_si256(mins_and_scales, 1), q8s);

// 对prod中的数据进行水平加法，存储到常量__m128i类型的变量hsum
const __m128i hsum = _mm_hadd_epi32(_mm_hadd_epi32(prod, mzero), mzero);

// 将hsum中的数据提取出来，并与dmin相乘，然后加到summs上
summs += dmin * _mm_extract_epi32(hsum, 0);

// 从mins_and_scales中提取数据，存储到常量__m128i类型的变量sc128
const __m128i sc128  = _mm256_extracti128_si256(mins_and_scales, 0);

// 将sc128中的数据扩展到__m256i类型的变量scales
const __m256i scales = MM256_SET_M128I(sc128, sc128);

// 从x[i].qh中加载数据到常量__m256i类型的变量hbits
const __m256i hbits = _mm256_loadu_si256((const __m256i*)x[i].qh);

// 初始化一个__m256i类型的变量hmask为全1
__m256i hmask = mone;

// 初始化一个__m256i类型的变量sumi为全0
__m256i sumi = _mm256_setzero_si256();

// 初始化一个整型变量bit为0
int bit = 0;
# 循环遍历QK_K/64次，每次处理64个元素
for (int j = 0; j < QK_K/64; ++j) {

    # 从scales中根据索引获取对应的8个元素，用于后续计算
    const __m256i scale_0 = _mm256_shuffle_epi8(scales, get_scale_shuffle_k4(2*j+0));
    const __m256i scale_1 = _mm256_shuffle_epi8(scales, get_scale_shuffle_k4(2*j+1));

    # 从q5中加载32个元素，每次加载32个元素
    const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5); q5 += 32;

    # 对q5bits进行位与运算，获取低16位的值
    const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);
    # 对hbits进行位与和位移运算，获取高16位的值
    const __m256i q5h_0 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_and_si256(hbits, hmask), bit++), 4);
    # 将低16位和高16位相加得到结果
    const __m256i q5_0  = _mm256_add_epi8(q5l_0, q5h_0);
    # 对hmask进行位左移运算
    hmask = _mm256_slli_epi16(hmask, 1);

    # 对q5bits进行位移和位与运算，获取低16位的值
    const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);
    # 对hbits进行位与和位移运算，获取高16位的值
    const __m256i q5h_1 = _mm256_slli_epi16(_mm256_srli_epi16(_mm256_and_si256(hbits, hmask), bit++), 4);
    # 将低16位和高16位相加得到结果
    const __m256i q5_1  = _mm256_add_epi8(q5l_1, q5h_1);
    # 对hmask进行位左移运算
    hmask = _mm256_slli_epi16(hmask, 1);

    # 从q8中加载32个元素，每次加载32个元素
    const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
    const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
}
# 使用 AVX 指令集进行向量化计算，计算两个 256 位整数向量的乘积并相加
__m256i p16_0 = _mm256_maddubs_epi16(q5_0, q8_0);
__m256i p16_1 = _mm256_maddubs_epi16(q5_1, q8_1);

# 使用 AVX 指令集进行向量化计算，将两个 256 位整数向量的乘积与另外两个 256 位整数向量相加
p16_0 = _mm256_madd_epi16(scale_0, p16_0);
p16_1 = _mm256_madd_epi16(scale_1, p16_1);

# 使用 AVX 指令集进行向量化计算，将两个 256 位整数向量相加并与另一个 256 位整数向量相加
sumi = _mm256_add_epi32(sumi, _mm256_add_epi32(p16_0, p16_1));

# 使用 AVX 指令集进行向量化计算，将一个 256 位浮点数向量与一个 256 位整数向量相乘并与另一个 256 位浮点数向量相加
__m256 vd = _mm256_set1_ps(d);
acc = _mm256_fmadd_ps(vd, _mm256_cvtepi32_ps(sumi), acc);

# 如果定义了 __AVX__，则执行以下代码
const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个全零的 128 位整数向量
    const __m128i mzero = _mm_setzero_si128();
    // 创建一个所有元素都为 1 的 128 位整数向量
    const __m128i mone  = _mm_set1_epi8(1);
    // 创建一个所有元素都为 2 的 128 位整数向量
    const __m128i m2 = _mm_set1_epi8(2);

    // 创建一个全零的 256 位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 初始化一个浮点数变量 summs 为 0
    float summs = 0.f;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 dmin 的值
        const float dmin = -y[i].d * GGML_FP16_TO_FP32(x[i].dmin);

        // 定义指向 x[i].qs 的 uint8_t 类型指针 q5
        const uint8_t * restrict q5 = x[i].qs;
        // 定义指向 y[i].qs 的 int8_t 类型指针 q8
        const int8_t  * restrict q8 = y[i].qs;

        // 将 x[i].scales 的内容复制到 utmp
        memcpy(utmp, x[i].scales, 12);
        // 对 utmp 进行位操作
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        // 对 utmp 进行位操作
        const uint32_t uaux = utmp[1] & kmask1;
        // 对 utmp 进行位操作
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        # 将第二个元素赋值为uaux
        utmp[2] = uaux;
        # 将第一个元素与kmask1进行按位与操作
        utmp[0] &= kmask1;

        # 将utmp数组中的元素构造成一个__m128i类型的变量utmps
        const __m128i utmps = _mm_set_epi32(utmp[3], utmp[2], utmp[1], utmp[0]);
        # 将utmps中的每个8位整数扩展为16位整数
        const __m128i scales = _mm_cvtepu8_epi16(utmps);
        # 将utmps中的每个8位整数扩展为16位整数，并将高低位拼接
        const __m128i mins = _mm_cvtepu8_epi16(_mm_unpackhi_epi64(utmps, utmps));

        # 从y[i].bsums数组中加载16字节数据到__m128i类型的变量q8sums_0和q8sums_1
        const __m128i q8sums_0 = _mm_loadu_si128((const __m128i*)&y[i].bsums[0]);
        const __m128i q8sums_1 = _mm_loadu_si128((const __m128i*)&y[i].bsums[8]);
        # 对q8sums_0和q8sums_1中的元素进行16位整数的水平加法
        const __m128i q8s = _mm_hadd_epi16(q8sums_0, q8sums_1);
        # 对mins和q8s中的元素进行16位整数的乘法累加
        const __m128i prod = _mm_madd_epi16(mins, q8s);
        # 对prod中的元素进行32位整数的水平加法
        const __m128i hsum = _mm_hadd_epi32(_mm_hadd_epi32(prod, mzero), mzero);
        # 将hsum中的第一个32位整数提取出来，并与dmin相乘后累加到summs中
        summs += dmin * _mm_extract_epi32(hsum, 0);

        # 从x[i].qh数组中加载16字节数据到__m128i类型的变量hbits_0和hbits_1
        const __m128i hbits_0 = _mm_loadu_si128((const __m128i*)&x[i].qh[0]);
        const __m128i hbits_1 = _mm_loadu_si128((const __m128i*)&x[i].qh[16]);
        # 初始化一个全1的__m128i类型变量hmask
        __m128i hmask = mone;

        # 初始化两个全0的__m128i类型变量sumi_0和sumi_1
        __m128i sumi_0 = _mm_setzero_si128();
        __m128i sumi_1 = _mm_setzero_si128();
# 初始化一个整型变量bit
int bit = 0;

# 创建一个用于数据重排的128位整数，所有元素都是0x0100
__m128i shuffle = _mm_set1_epi16(0x0100);
for (int j = 0; j < QK_K/64; ++j) {
    # 从scales中按照shuffle的规则提取数据
    const __m128i scale_0 = _mm_shuffle_epi8(scales, shuffle);
    # 更新shuffle的值，用于下一次数据重排
    shuffle = _mm_add_epi16(shuffle, m2);
    # 从scales中按照更新后的shuffle的规则提取数据
    const __m128i scale_1 = _mm_shuffle_epi8(scales, shuffle);
    # 更新shuffle的值，用于下一次数据重排
    shuffle = _mm_add_epi16(shuffle, m2);

    # 从q5中加载16字节数据到q5bits_0和q5bits_1中，然后更新q5的指针位置
    const __m128i q5bits_0 = _mm_loadu_si128((const __m128i*)q5); q5 += 16;
    const __m128i q5bits_1 = _mm_loadu_si128((const __m128i*)q5); q5 += 16;

    # 对q5bits_0和q5bits_1进行按位与操作，结果存储到q5l_0和q5l_1中
    __m128i q5l_0 = _mm_and_si128(q5bits_0, m4);
    __m128i q5l_1 = _mm_and_si128(q5bits_1, m4);
    # 对hbits_0和hbits_1进行按位与和位移操作，结果存储到q5h_0和q5h_1中
    __m128i q5h_0 = _mm_slli_epi16(_mm_srli_epi16(_mm_and_si128(hbits_0, hmask), bit), 4);
    __m128i q5h_1 = _mm_slli_epi16(_mm_srli_epi16(_mm_and_si128(hbits_1, hmask), bit++), 4);
    # 对q5l_0和q5h_0进行字节级加法，结果存储到q5_0中
    __m128i q5_0  = _mm_add_epi8(q5l_0, q5h_0);
    # 对q5l_1和q5h_1进行字节级加法，结果存储到q5_1中
    __m128i q5_1  = _mm_add_epi8(q5l_1, q5h_1);
# 将 hmask 左移 1 位
hmask = _mm_slli_epi16(hmask, 1);

# 从内存中加载 q8 的值到 __m128i 类型的变量 q8_0，然后将 q8 指针向后移动 16 个字节
__m128i q8_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 从内存中加载 q8 的值到 __m128i 类型的变量 q8_1，然后将 q8 指针向后移动 16 个字节
__m128i q8_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 q5_0 和 q8_0 计算 p16_0
__m128i p16_0 = _mm_maddubs_epi16(q5_0, q8_0);
# 使用 q5_1 和 q8_1 计算 p16_1
__m128i p16_1 = _mm_maddubs_epi16(q5_1, q8_1);
# 使用 scale_0 和 p16_0 计算 p16_0
p16_0 = _mm_madd_epi16(scale_0, p16_0);
# 使用 scale_0 和 p16_1 计算 p16_1
p16_1 = _mm_madd_epi16(scale_0, p16_1);

# 对 q5bits_0 右移 4 位并与 m4 进行按位与操作，结果存入 q5l_0
q5l_0 = _mm_and_si128(_mm_srli_epi16(q5bits_0, 4), m4);
# 对 q5bits_1 右移 4 位并与 m4 进行按位与操作，结果存入 q5l_1
q5l_1 = _mm_and_si128(_mm_srli_epi16(q5bits_1, 4), m4);
# 对 hbits_0 与 hmask 进行按位与操作，然后右移 bit 位并左移 4 位，结果存入 q5h_0
q5h_0 = _mm_slli_epi16(_mm_srli_epi16(_mm_and_si128(hbits_0, hmask), bit), 4);
# 对 hbits_1 与 hmask 进行按位与操作，然后右移 bit 位并左移 4 位，结果存入 q5h_1
q5h_1 = _mm_slli_epi16(_mm_srli_epi16(_mm_and_si128(hbits_1, hmask), bit++), 4);
# 将 q5l_0 和 q5h_0 进行相加，结果存入 q5_0
q5_0  = _mm_add_epi8(q5l_0, q5h_0);
# 将 q5l_1 和 q5h_1 进行相加，结果存入 q5_1
q5_1  = _mm_add_epi8(q5l_1, q5h_1);
# 将 hmask 左移 1 位
hmask = _mm_slli_epi16(hmask, 1);

# 从内存中加载 q8 的值到 __m128i 类型的变量 q8_0，然后将 q8 指针向后移动 16 个字节
q8_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 从内存中加载 q8 的值到 __m128i 类型的变量 q8_1，然后将 q8 指针向后移动 16 个字节
q8_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
# 使用 q5_0 和 q8_0 计算 p16_2
__m128i p16_2 = _mm_maddubs_epi16(q5_0, q8_0);
            # 使用 q5_1 和 q8_1 进行有符号字节乘法并累加到 p16_3
            __m128i p16_3 = _mm_maddubs_epi16(q5_1, q8_1);
            # 使用 scale_1 进行有符号字节乘法并累加到 p16_2
            p16_2 = _mm_madd_epi16(scale_1, p16_2);
            # 使用 scale_1 进行有符号字节乘法并累加到 p16_3
            p16_3 = _mm_madd_epi16(scale_1, p16_3);

            # 将 p16_0 和 p16_2 相加，并累加到 sumi_0
            sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p16_0, p16_2));
            # 将 p16_1 和 p16_3 相加，并累加到 sumi_1
            sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p16_1, p16_3));

        }

        # 创建一个包含 8 个 d 值的 __m256 向量
        __m256 vd = _mm256_set1_ps(d);
        # 将 sumi_1 和 sumi_0 合并成一个 __m256i 向量
        __m256i sumi = MM256_SET_M128I(sumi_1, sumi_0);
        # 将 vd 乘以 sumi，并加到 acc
        acc = _mm256_add_ps(_mm256_mul_ps(vd, _mm256_cvtepi32_ps(sumi)), acc);

    }

    # 将 acc 中的 8 个单精度浮点数求和，并加到 summs，结果赋值给 *s
    *s = hsum_float_8(acc) + summs;

#elif defined __riscv_v_intrinsic

    # 创建一个指向 utmp[0] 的 uint8_t 指针
    const uint8_t * scales = (const uint8_t*)&utmp[0];
    // 从 utmp 数组的第三个元素开始，将其转换为 uint8_t 类型的指针 mins
    const uint8_t * mins   = (const uint8_t*)&utmp[2];

    // 初始化两个浮点数变量 sumf 和 sums
    float sumf = 0;
    float sums = 0.0;

    // 声明一个 size_t 类型的变量 vl
    size_t vl;

    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 初始化 vl 为 8
        vl = 8;

        // 声明并初始化指向 x[i].qs 的 uint8_t 类型指针 q5
        const uint8_t * restrict q5 = x[i].qs;
        // 声明并初始化指向 x[i].qh 的 uint8_t 类型指针 hm
        const uint8_t * restrict hm = x[i].qh;
        // 声明并初始化指向 y[i].qs 的 int8_t 类型指针 q8
        const  int8_t * restrict q8 = y[i].qs;

        // 计算 d 和 dmin 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;

        // 从 y[i].bsums 中加载数据到 vint16mf2_t 类型的变量 q8sums_0 和 q8sums_1
        vint16mf2_t q8sums_0 = __riscv_vlse16_v_i16mf2(y[i].bsums, 4, vl);
        vint16mf2_t q8sums_1 = __riscv_vlse16_v_i16mf2(y[i].bsums+1, 4, vl);
        # 使用 vadd 指令对两个 vint16mf2_t 类型的寄存器进行相加操作，结果存储在 q8sums 中

        # 将 x[i].scales 中的数据复制到 utmp 中，长度为 12 个字节
        memcpy(utmp, x[i].scales, 12);
        # 对 utmp 中的数据进行位操作，将结果存储在 utmp 中
        utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
        # 将 utmp[1] 中的数据进行位操作，将结果存储在 uaux 中
        const uint32_t uaux = utmp[1] & kmask1;
        # 对 utmp 中的数据进行位操作，将结果存储在 utmp 中
        utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
        # 将 uaux 中的数据存储回 utmp[2] 中
        utmp[2] = uaux;
        # 对 utmp[0] 中的数据进行位操作，将结果存储在 utmp 中
        utmp[0] &= kmask1;

        # 使用 vle8 指令从 mins 中加载数据到 mins8 中
        vuint8mf4_t mins8 = __riscv_vle8_v_u8mf4(mins, vl);
        # 将 mins8 中的数据进行零扩展和类型转换，结果存储在 v_mins 中
        vint16mf2_t v_mins = __riscv_vreinterpret_v_u16mf2_i16mf2(__riscv_vzext_vf2_u16mf2(mins8, vl));
        # 使用 vwmul 指令对两个 vint16mf2_t 类型的寄存器进行乘法操作，结果存储在 prod 中
        vint32m1_t prod = __riscv_vwmul_vv_i32m1(q8sums, v_mins, vl);

        # 使用 vredsum 指令对 vint32m1_t 类型的寄存器进行求和操作，结果存储在 sumi 中
        vint32m1_t sumi = __riscv_vredsum_vs_i32m1_i32m1(prod, __riscv_vmv_v_x_i32m1(0, 1), vl);
        # 对 sumf 进行减法操作，结果存储在 sumf 中
        sumf -= dmin * __riscv_vmv_x_s_i32m1_i32(sumi);

        # 将 vl 设置为 32
        vl = 32;
        # 将 aux32 设置为 0
        int32_t aux32 = 0;
        # 将 is 设置为 0
        int is = 0;
        // 定义一个无符号8位整数变量m，并初始化为1
        uint8_t m = 1;
        // 使用指令加载一个32位整数向量，其中元素为0和1
        vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);
        // 使用指令加载一个无符号8位整数向量，从内存hm地址处加载数据
        vuint8m1_t vqh = __riscv_vle8_v_u8m1(hm, vl);

        for (int j = 0; j < QK_K/64; ++j) {
            // 加载Q5和Q8
            vuint8m1_t q5_x = __riscv_vle8_v_u8m1(q5, vl);
            vint8m1_t  q8_y1 = __riscv_vle8_v_i8m1(q8, vl);
            vint8m1_t  q8_y2 = __riscv_vle8_v_i8m1(q8+32, vl);

            // 计算加法的掩码
            vint8m1_t q5_a = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vand_vx_u8m1(q5_x, 0x0F, vl));
            vuint8m1_t qh_m1 = __riscv_vand_vx_u8m1(vqh, m, vl);
            vbool8_t vmask_1 = __riscv_vmsne_vx_u8m1_b8(qh_m1, 0, vl);
            vint8m1_t q5_m1 = __riscv_vadd_vx_i8m1_m(vmask_1, q5_a, 16, vl);
            m <<= 1;

            vint8m1_t q5_l = __riscv_vreinterpret_v_u8m1_i8m1(__riscv_vsrl_vx_u8m1(q5_x, 0x04, vl));
            vuint8m1_t qh_m2 = __riscv_vand_vx_u8m1(vqh, m, vl);
            vbool8_t vmask_2 = __riscv_vmsne_vx_u8m1_b8(qh_m2, 0, vl);
// 使用指定的掩码和常量值对两个向量进行 8 位整数加法
vint8m1_t q5_m2 = __riscv_vadd_vx_i8m1_m(vmask_2, q5_l, 16, vl);
// 左移操作
m <<= 1;

// 两个向量的 16 位整数乘法
vint16m2_t v0 = __riscv_vwmul_vv_i16m2(q5_m1, q8_y1, vl);
vint16m2_t v1 = __riscv_vwmul_vv_i16m2(q5_m2, q8_y2, vl);

// 向量与标量的 32 位整数乘法
vint32m4_t vs1 = __riscv_vwmul_vx_i32m4(v0, scales[is++], vl);
vint32m4_t vs2 = __riscv_vwmul_vx_i32m4(v1, scales[is++], vl);

// 向量的 32 位整数求和
vint32m1_t vacc1 = __riscv_vredsum_vs_i32m4_i32m1(vs1, vzero, vl);
vint32m1_t vacc2 = __riscv_vredsum_vs_i32m4_i32m1(vs2, vzero, vl);

// 将两个向量的结果相加
aux32 += __riscv_vmv_x_s_i32m1_i32(vacc1) + __riscv_vmv_x_s_i32m1_i32(vacc2);
q5 += 32;    q8 += 64;

// 将 32 位整数转换为单精度浮点数，并进行单精度浮点数乘法
vfloat32m1_t vaux = __riscv_vfmul_vf_f32m1(__riscv_vfmv_v_f_f32m1(aux32, 1), d, 1);
// 将单精度浮点数结果累加到总和中
sums += __riscv_vfmv_f_s_f32m1_f32(vaux);
    }

    *s = sumf+sums;  // 将sumf和sums的和赋值给指针s

#else

    const uint8_t * scales = (const uint8_t*)&utmp[0];  // 定义并初始化指向utmp数组第一个元素的指针scales
    const uint8_t * mins   = (const uint8_t*)&utmp[2];  // 定义并初始化指向utmp数组第三个元素的指针mins

    int8_t  aux8[QK_K];  // 定义长度为QK_K的int8_t类型数组aux8
    int16_t aux16[8];  // 定义长度为8的int16_t类型数组aux16
    float   sums [8];  // 定义长度为8的float类型数组sums
    int32_t aux32[8];  // 定义长度为8的int32_t类型数组aux32
    memset(sums, 0, 8*sizeof(float));  // 将sums数组的前8个元素初始化为0

    float sumf = 0;  // 定义并初始化sumf为0
    for (int i = 0; i < nb; ++i) {  // 循环，i从0到nb-1
        const uint8_t * restrict q4 = x[i].qs;  // 定义并初始化指向x[i].qs的指针q4
        const uint8_t * restrict hm = x[i].qh;  // 定义并初始化指向x[i].qh的指针hm
        const  int8_t * restrict q8 = y[i].qs;  // 定义并初始化指向y[i].qs的指针q8
# 将 aux32 数组清零，长度为 8*sizeof(int32_t)
memset(aux32, 0, 8*sizeof(int32_t));
# 声明指向 aux8 的 int8_t 类型指针 a
int8_t * restrict a = aux8;
# 声明 uint8_t 类型变量 m，并赋值为 1
uint8_t m = 1;
# 循环 QK_K/64 次
for (int j = 0; j < QK_K/64; ++j) {
    # 将 q4 数组的低 4 位赋值给 a 数组
    for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l] & 0xF);
    # 根据 hm 数组和 m 的值，给 a 数组的每个元素加上 16
    for (int l = 0; l < 32; ++l) a[l] += (hm[l] & m ? 16 : 0);
    # a 指针向后移动 32 个位置，m 左移 1 位
    a += 32; m <<= 1;
    # 将 q4 数组的高 4 位赋值给 a 数组
    for (int l = 0; l < 32; ++l) a[l] = (int8_t)(q4[l]  >> 4);
    # 根据 hm 数组和 m 的值，给 a 数组的每个元素加上 16
    for (int l = 0; l < 32; ++l) a[l] += (hm[l] & m ? 16 : 0);
    # a 指针向后移动 32 个位置，m 左移 1 位
    a += 32; m <<= 1;
    # q4 指针向后移动 32 个位置
    q4 += 32;
}
# 将 x[i].scales 的前 12 个字节拷贝到 utmp 数组
memcpy(utmp, x[i].scales, 12);
# 对 utmp 数组进行位运算和赋值操作
utmp[3] = ((utmp[2] >> 4) & kmask2) | (((utmp[1] >> 6) & kmask3) << 4);
# 将 utmp[1] 的部分值赋给 uaux，然后对 utmp[1] 进行位运算和赋值操作
const uint32_t uaux = utmp[1] & kmask1;
utmp[1] = (utmp[2] & kmask2) | (((utmp[0] >> 6) & kmask3) << 4);
utmp[2] = uaux;
utmp[0] &= kmask1;
# 声明并初始化 sumi 变量为 0
int sumi = 0;
        // 遍历循环，计算y[i].bsums[j] * mins[j/2]的和
        for (int j = 0; j < QK_K/16; ++j) sumi += y[i].bsums[j] * mins[j/2];
        // 初始化变量a为aux8，is为0
        a = aux8;
        int is = 0;
        // 遍历循环，计算scale * aux16[l]的和
        for (int j = 0; j < QK_K/32; ++j) {
            int32_t scale = scales[is++];
            // 计算q8[l] * a[l]的值，存入aux16[l]
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 计算scale * aux16[l]的值，存入aux32[l]
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            // 更新q8和a的值
            q8 += 8; a += 8;
            // 重复上述操作
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
        // 计算GGML_FP16_TO_FP32(x[i].d) * y[i].d的值，存入d
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 计算d * aux32[l]的和，存入sums[l]
        for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
// 计算浮点数的最小值，并乘以另一个浮点数
const float dmin = GGML_FP16_TO_FP32(x[i].dmin) * y[i].d;
// 用最小值乘以整数和，减去结果
sumf -= dmin * sumi;
// 循环遍历8次，将sums数组中的值加到sumf上
for (int l = 0; l < 8; ++l) sumf += sums[l];
// 将sumf的值赋给指针s所指向的变量
*s = sumf;
#endif
}

#else

// 计算两个向量的点积，其中n是向量的长度，s是结果指针，vx和vy分别是两个向量的指针
void ggml_vec_dot_q5_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言n能被QK_K整除
    assert(n % QK_K == 0);

    // 将vx和vy分别转换为block_q5_K和block_q8_K类型的指针
    const block_q5_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    // 计算向量长度除以QK_K的商
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 创建一个包含16个8位无符号整数的向量，每个元素都是0xf
    const uint8x16_t m4b = vdupq_n_u8(0xf);
    // 创建一个包含16个8位无符号整数的向量，每个元素都是16
    const uint8x16_t mh = vdupq_n_u8(16);
#if defined(__ARM_FEATURE_DOTPROD)
    // 如果支持ARM的dot product特性，则创建一个包含4个32位有符号整数的向量，每个元素都是0
    const int32x4_t mzero = vdupq_n_s32(0);
#endif

    // 创建一个包含4个8位整数向量的结构体
    ggml_int8x16x4_t q5bytes;
    // 创建一个包含4个8位无符号整数向量的结构体
    ggml_uint8x16x4_t q5h;

    // 初始化一个浮点数变量sumf为0
    float sumf = 0;

    // 循环遍历nb次
    for (int i = 0; i < nb; ++i) {

        // 计算y[i].d和x[i].d的乘积，并转换为浮点数赋值给d
        const float d = y[i].d * (float)x[i].d;
        // 获取x[i].scales的指针赋值给sc
        const int8_t * sc = x[i].scales;

        // 获取x[i].qs的指针赋值给q5
        const uint8_t * restrict q5 = x[i].qs;
        // 获取x[i].qh的指针赋值给qh
        const uint8_t * restrict qh = x[i].qh;
        // 获取y[i].qs的指针赋值给q8
        const int8_t  * restrict q8 = y[i].qs;
        // 从指针 qh 加载 8 个字节的数据到寄存器 qhbits
        const uint8x8_t qhbits = vld1_u8(qh);

        // 从指针 q5 加载 32 个字节的数据到寄存器 q5bits
        const ggml_uint8x16x2_t q5bits = ggml_vld1q_u8_x2(q5);
        // 从指针 q8 加载 64 个字节的数据到寄存器 q8bytes
        const ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

        // 将 qhbits 的数据进行位移和组合，存储到寄存器 htmp
        const uint8x16_t htmp = vcombine_u8(qhbits, vshr_n_u8(qhbits, 1));
        // 使用位操作和移位操作，计算 q5h 的值
        q5h.val[0] = vbicq_u8(mh, vshlq_n_u8(htmp, 4));
        q5h.val[1] = vbicq_u8(mh, vshlq_n_u8(htmp, 2));
        q5h.val[2] = vbicq_u8(mh, htmp);
        q5h.val[3] = vbicq_u8(mh, vshrq_n_u8(htmp, 2));

        // 使用位操作和移位操作，计算 q5bytes 的值
        q5bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[0], m4b)), vreinterpretq_s8_u8(q5h.val[0]));
        q5bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vandq_u8(q5bits.val[1], m4b)), vreinterpretq_s8_u8(q5h.val[1]));
        q5bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[0], 4)), vreinterpretq_s8_u8(q5h.val[2]));
        q5bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vshrq_n_u8(q5bits.val[1], 4)), vreinterpretq_s8_u8(q5h.val[3]));

        // 如果支持 ARM Dot Product 指令集
        #if defined(__ARM_FEATURE_DOTPROD)
        // 计算 sumi1 和 sumi2 的值
        int32_t sumi1 = sc[0] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[0], q8bytes.val[0]));
        int32_t sumi2 = sc[1] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[1], q8bytes.val[1]));
        // 计算第2和第3个元素的乘积，并将结果与mzero进行点积运算，再与sc[2]相乘
        int32_t sumi3 = sc[2] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[2], q8bytes.val[2]));
        // 计算第3和第4个元素的乘积，并将结果与mzero进行点积运算，再与sc[3]相乘
        int32_t sumi4 = sc[3] * vaddvq_s32(vdotq_s32(mzero, q5bytes.val[3], q8bytes.val[3]));

        // 将上述计算结果相加，并乘以d，加到sumf上
        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);

#else

        // 计算第0个元素的乘积，并将结果相加到p0上
        const int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                       vmull_s8(vget_high_s8(q5bytes.val[0]), vget_high_s8(q8bytes.val[0])));
        // 计算第1个元素的乘积，并将结果相加到p1上
        const int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                       vmull_s8(vget_high_s8(q5bytes.val[1]), vget_high_s8(q8bytes.val[1]));
        // 计算p0和p1的加权和，并乘以sc[0]和sc[1]，加到sumf上
        int32_t sumi = sc[0] * vaddvq_s16(p0) + sc[1] * vaddvq_s16(p1);

        // 计算第2个元素的乘积，并将结果相加到p2上
        const int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                                       vmull_s8(vget_high_s8(q5bytes.val[2]), vget_high_s8(q8bytes.val[2]));
        // 计算第3个元素的乘积，并将结果相加到p3上
        const int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q5bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                                       vmull_s8(vget_high_s8(q5bytes.val[3]), vget_high_s8(q8bytes.val[3]));
        // 计算p2和p3的加权和，并乘以sc[2]和sc[3]，加到sumf上
        sumi += sc[2] * vaddvq_s16(p2) + sc[3] * vaddvq_s16(p3);

        // 将上述计算结果乘以d，加到sumf上
        sumf += d*sumi;
#endif
    // 如果定义了条件编译符号，则执行以下代码

    }

    *s = sumf;
    // 将指针 s 指向的值设置为 sumf

#elif defined __AVX2__
    // 如果定义了条件编译符号 __AVX2__，则执行以下代码

    const __m256i m4 = _mm256_set1_epi8(0xF);
    // 创建一个包含 32 个 0xF 的 256 位整数向量

    const __m256i mone  = _mm256_set1_epi8(1);
    // 创建一个包含 32 个 1 的 256 位整数向量

    __m256 acc = _mm256_setzero_ps();
    // 创建一个包含 8 个 0 的 256 位单精度浮点数向量

    for (int i = 0; i < nb; ++i) {
        // 循环，i 从 0 到 nb-1

        const uint8_t * restrict q5 = x[i].qs;
        // 创建一个指向 x[i].qs 的 uint8_t 类型指针，并限制其它指针对该指针的访问

        const int8_t  * restrict q8 = y[i].qs;
        // 创建一个指向 y[i].qs 的 int8_t 类型指针，并限制其它指针对该指针的访问

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
        // 计算 y[i].d 乘以 GGML_FP16_TO_FP32(x[i].d) 的结果，并赋值给变量 d
        // 从内存中加载 q5 数组的值到 256 位寄存器
        const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);

        // 将 x[i] 结构体中的 scales 数组的值分别扩展成 16 位整数，存储到两个 128 位寄存器中
        const __m256i scale_l = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[1]), _mm_set1_epi16(x[i].scales[0]));
        const __m256i scale_h = MM256_SET_M128I(_mm_set1_epi16(x[i].scales[3]), _mm_set1_epi16(x[i].scales[2]));

        // 将 x[i] 结构体中的 qh 数组的值拷贝到 64 位整数中，然后分别存储到两个 128 位寄存器中
        int64_t aux64;
        memcpy(&aux64, x[i].qh, 8);
        const __m128i haux128 = _mm_set_epi64x(aux64 >> 1, aux64);
        const __m256i haux256 = MM256_SET_M128I(_mm_srli_epi16(haux128, 2), haux128);

        // 对 q5bits 中的值进行位运算和移位操作，得到 q5h_0 和 q5h_1
        const __m256i q5h_0 = _mm256_slli_epi16(_mm256_andnot_si256(haux256, mone), 4);
        const __m256i q5h_1 = _mm256_slli_epi16(_mm256_andnot_si256(_mm256_srli_epi16(haux256, 4), mone), 4);

        // 对 q5bits 中的值进行位运算和移位操作，得到 q5l_0 和 q5l_1
        const __m256i q5l_0 = _mm256_and_si256(q5bits, m4);
        const __m256i q5l_1 = _mm256_and_si256(_mm256_srli_epi16(q5bits, 4), m4);

        // 从内存中加载 q8 数组的值到 256 位寄存器中的两个变量
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        // 对 scale_l、q5l_0 和 q8_0 中的值进行乘法和加法操作，得到 p16_0
        const __m256i p16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5l_0, q8_0));
        // 使用乘法和加法操作对两个 16 位整数向量进行处理
        const __m256i p16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5l_1, q8_1));
        // 使用乘法和加法操作对两个 16 位整数向量进行处理
        const __m256i s16_0 = _mm256_madd_epi16(scale_l, _mm256_maddubs_epi16(q5h_0, q8_0));
        // 使用乘法和加法操作对两个 16 位整数向量进行处理
        const __m256i s16_1 = _mm256_madd_epi16(scale_h, _mm256_maddubs_epi16(q5h_1, q8_1));

        // 对两个 32 位整数向量进行减法和加法操作
        const __m256i dot = _mm256_sub_epi32(_mm256_add_epi32(p16_0, p16_1), _mm256_add_epi32(s16_0, s16_1));

        // 使用乘法和加法操作对两个 32 位整数向量进行处理，并将结果累加到 acc 中
        acc = _mm256_fmadd_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(dot), acc);

    }

    // 对累加器中的 8 个单精度浮点数进行水平求和
    *s = hsum_float_8(acc);

#elif defined __AVX__

    // 创建一个所有元素都为 0xF 的 128 位整数向量
    const __m128i m4 = _mm_set1_epi8(0xF);
    // 创建一个所有元素都为 1 的 128 位整数向量
    const __m128i mone  = _mm_set1_epi8(1);

    // 创建一个所有元素都为 0 的 256 位单精度浮点数向量
    __m256 acc = _mm256_setzero_ps();

    // 循环处理每个元素
    for (int i = 0; i < nb; ++i) {
// 从结构体数组 x 中获取指针 q5，限制指针的操作范围
const uint8_t * restrict q5 = x[i].qs;
// 从结构体数组 y 中获取指针 q8，限制指针的操作范围
const int8_t  * restrict q8 = y[i].qs;

// 计算浮点数 d，使用结构体数组 y 中的 d 乘以结构体数组 x 中的 d 转换为单精度浮点数
const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

// 从指针 q5 中加载 256 位数据到寄存器中
const __m256i q5bits = _mm256_loadu_si256((const __m256i*)q5);

// 创建四个 128 位的常量寄存器，用于后续的计算
const __m128i scale_0 = _mm_set1_epi16(x[i].scales[0]);
const __m128i scale_1 = _mm_set1_epi16(x[i].scales[1]);
const __m128i scale_2 = _mm_set1_epi16(x[i].scales[2]);
const __m128i scale_3 = _mm_set1_epi16(x[i].scales[3]);

// 定义一个 64 位整型变量 aux64，将结构体数组 x 中的 qh 拷贝到 aux64 中
int64_t aux64;
memcpy(&aux64, x[i].qh, 8);
// 将 aux64 的值右移 1 位，然后将结果和 aux64 组合成一个 128 位的寄存器 haux128_0
const __m128i haux128_0 = _mm_set_epi64x(aux64 >> 1, aux64);
// 将 haux128_0 的值逻辑右移 2 位，得到 haux128_1
const __m128i haux128_1 = _mm_srli_epi16(haux128_0, 2);

// 对 q5h_0 和 q5h_1 进行位运算和移位操作，得到最终的结果
const __m128i q5h_0 = _mm_slli_epi16(_mm_andnot_si128(haux128_0, mone), 4);
const __m128i q5h_1 = _mm_slli_epi16(_mm_andnot_si128(haux128_1, mone), 4);
// 使用位移和逻辑运算对输入的__m128i类型的数据进行处理，得到q5h_2和q5h_3
const __m128i q5h_2 = _mm_slli_epi16(_mm_andnot_si128(_mm_srli_epi16(haux128_0, 4), mone), 4);
const __m128i q5h_3 = _mm_slli_epi16(_mm_andnot_si128(_mm_srli_epi16(haux128_1, 4), mone), 4);

// 使用位移和逻辑运算对输入的__m128i类型的数据进行处理，得到q5l_0、q5l_1、q5l_2和q5l_3
const __m128i q5l_0 = _mm_and_si128(_mm256_extractf128_si256(q5bits, 0), m4);
const __m128i q5l_1 = _mm_and_si128(_mm256_extractf128_si256(q5bits, 1), m4);
const __m128i q5l_2 = _mm_and_si128(_mm_srli_epi16(_mm256_extractf128_si256(q5bits, 0), 4), m4);
const __m128i q5l_3 = _mm_and_si128(_mm_srli_epi16(_mm256_extractf128_si256(q5bits, 1), 4), m4);

// 加载__m256i类型的数据到q8_0和q8_1
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

// 对输入的__m128i类型的数据进行处理，得到p16_0、p16_1、p16_2、p16_3、s16_0、s16_1、s16_2和s16_3
const __m128i p16_0 = _mm_madd_epi16(scale_0, _mm_maddubs_epi16(q5l_0, _mm256_extractf128_si256(q8_0, 0)));
const __m128i p16_1 = _mm_madd_epi16(scale_1, _mm_maddubs_epi16(q5l_1, _mm256_extractf128_si256(q8_0, 1)));
const __m128i p16_2 = _mm_madd_epi16(scale_2, _mm_maddubs_epi16(q5l_2, _mm256_extractf128_si256(q8_1, 0)));
const __m128i p16_3 = _mm_madd_epi16(scale_3, _mm_maddubs_epi16(q5l_3, _mm256_extractf128_si256(q8_1, 1)));
const __m128i s16_0 = _mm_madd_epi16(scale_0, _mm_maddubs_epi16(q5h_0, _mm256_extractf128_si256(q8_0, 0)));
const __m128i s16_1 = _mm_madd_epi16(scale_1, _mm_maddubs_epi16(q5h_1, _mm256_extractf128_si256(q8_0, 1)));
const __m128i s16_2 = _mm_madd_epi16(scale_2, _mm_maddubs_epi16(q5h_2, _mm256_extractf128_si256(q8_1, 0)));
const __m128i s16_3 = _mm_madd_epi16(scale_3, _mm_maddubs_epi16(q5h_3, _mm256_extractf128_si256(q8_1, 1)));
        // 计算 dot_0，即 p16_0 + p16_2 - s16_0 - s16_2
        const __m128i dot_0 = _mm_sub_epi32(_mm_add_epi32(p16_0, p16_2), _mm_add_epi32(s16_0, s16_2));
        // 计算 dot_1，即 p16_1 + p16_3 - s16_1 - s16_3
        const __m128i dot_1 = _mm_sub_epi32(_mm_add_epi32(p16_1, p16_3), _mm_add_epi32(s16_1, s16_3));

        // 计算 acc = acc + d * (dot_1, dot_0)
        acc = _mm256_add_ps(_mm256_mul_ps(_mm256_set1_ps(d), _mm256_cvtepi32_ps(MM256_SET_M128I(dot_1, dot_0))), acc);

    }

    // 将 acc 中的 8 个 float 求和
    *s = hsum_float_8(acc);

#elif defined __riscv_v_intrinsic

    // 初始化 sumf 为 0
    float sumf = 0;

    // 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        // 计算 d = y[i].d * (float)x[i].d
        const float d = y[i].d * (float)x[i].d;
        // 获取 x[i].scales 的指针
        const int8_t * sc = x[i].scales;

        // 获取 x[i].qs 的指针
        const uint8_t * restrict q5 = x[i].qs;
        // 获取 x[i].qh 的指针
        const uint8_t * restrict qh = x[i].qh;
// 声明一个指向 y[i].qs 的 int8_t 类型的指针，并使用 restrict 限定符
const int8_t  * restrict q8 = y[i].qs;

// 声明一个 vint32m1_t 类型的变量 vzero，并初始化为全零
vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);

// 从内存中加载 qh 的值到 vuint8mf4_t 类型的变量 qh_x1 和 qh_x2
vuint8mf4_t qh_x1   = __riscv_vle8_v_u8mf4(qh, 8);
vuint8mf2_t qh_x2   = __riscv_vlmul_ext_v_u8mf4_u8mf2(__riscv_vsrl_vx_u8mf4(qh_x1, 1, 8));

// 初始化变量 vl 为 16
size_t vl = 16;

// 将 qh_x1 和 qh_x2 合并成一个新的 vuint8mf2_t 类型的变量 qh_x
vuint8mf2_t qh_x = __riscv_vslideup_vx_u8mf2(__riscv_vlmul_ext_v_u8mf4_u8mf2(qh_x1), qh_x2, vl/2, vl);

// 对 qh_x 进行位运算，得到 qh_h0、qh_h1、qh_h2、qh_h3 四个新的 vuint8mf2_t 类型的变量
vuint8mf2_t qh_h0 = __riscv_vand_vx_u8mf2(__riscv_vnot_v_u8mf2(__riscv_vsll_vx_u8mf2(qh_x, 0x4, vl), vl), 16, vl);
vuint8mf2_t qh_h1 = __riscv_vand_vx_u8mf2(__riscv_vnot_v_u8mf2(__riscv_vsll_vx_u8mf2(qh_x, 0x2, vl), vl), 16, vl);
vuint8mf2_t qh_h2 = __riscv_vand_vx_u8mf2(__riscv_vnot_v_u8mf2(qh_x, vl), 16, vl);
vuint8mf2_t qh_h3 = __riscv_vand_vx_u8mf2(__riscv_vnot_v_u8mf2(__riscv_vsrl_vx_u8mf2(qh_x, 0x4, vl), vl), 16, vl);

// 将 qh_h0 和 qh_h1 转换为 vint8mf2_t 类型的变量 qh_0 和 qh_1
vint8mf2_t qh_0 = __riscv_vreinterpret_v_u8mf2_i8mf2(qh_h0);
vint8mf2_t qh_1 = __riscv_vreinterpret_v_u8mf2_i8mf2(qh_h1);
        // 将 uint8 类型的寄存器转换为 int8 类型的寄存器
        vint8mf2_t qh_2 = __riscv_vreinterpret_v_u8mf2_i8mf2(qh_h2);
        vint8mf2_t qh_3 = __riscv_vreinterpret_v_u8mf2_i8mf2(qh_h3);

        // 加载 q5 寄存器的值
        vuint8mf2_t q5_x1  = __riscv_vle8_v_u8mf2(q5, vl);
        vuint8mf2_t q5_x2  = __riscv_vle8_v_u8mf2(q5+16, vl);

        // 将 q5 寄存器的值进行位与运算和右移操作，得到 q5s 寄存器的值
        vint8mf2_t q5s_0 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(q5_x1, 0xF, vl));
        vint8mf2_t q5s_1 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vand_vx_u8mf2(q5_x2, 0xF, vl));
        vint8mf2_t q5s_2 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vsrl_vx_u8mf2(q5_x1, 0x4, vl));
        vint8mf2_t q5s_3 = __riscv_vreinterpret_v_u8mf2_i8mf2(__riscv_vsrl_vx_u8mf2(q5_x2, 0x4, vl));

        // 将 q5s 寄存器的值减去 qh 寄存器的值，得到 q5 寄存器的值
        vint8mf2_t q5_0 = __riscv_vsub_vv_i8mf2(q5s_0, qh_0, vl);
        vint8mf2_t q5_1 = __riscv_vsub_vv_i8mf2(q5s_1, qh_1, vl);
        vint8mf2_t q5_2 = __riscv_vsub_vv_i8mf2(q5s_2, qh_2, vl);
        vint8mf2_t q5_3 = __riscv_vsub_vv_i8mf2(q5s_3, qh_3, vl);

        // 加载 Q8 寄存器的值，并将其与 Q5 寄存器的值进行乘法运算
        vint16m1_t p0 = __riscv_vwmul_vv_i16m1(q5_0, __riscv_vle8_v_i8mf2(q8, vl), vl);
        vint16m1_t p1 = __riscv_vwmul_vv_i16m1(q5_1, __riscv_vle8_v_i8mf2(q8+16, vl), vl);
        # 使用 q5_2 和 q8+32 的值进行有符号 16 位整数乘法
        vint16m1_t p2 = __riscv_vwmul_vv_i16m1(q5_2, __riscv_vle8_v_i8mf2(q8+32, vl), vl);
        # 使用 q5_3 和 q8+48 的值进行有符号 16 位整数乘法
        vint16m1_t p3 = __riscv_vwmul_vv_i16m1(q5_3, __riscv_vle8_v_i8mf2(q8+48, vl), vl);

        # 对 p0 进行有符号 16 位整数累加求和
        vint32m1_t vs_0 = __riscv_vwredsum_vs_i16m1_i32m1(p0, vzero, vl);
        # 对 p1 进行有符号 16 位整数累加求和
        vint32m1_t vs_1 = __riscv_vwredsum_vs_i16m1_i32m1(p1, vzero, vl);
        # 对 p2 进行有符号 16 位整数累加求和
        vint32m1_t vs_2 = __riscv_vwredsum_vs_i16m1_i32m1(p2, vzero, vl);
        # 对 p3 进行有符号 16 位整数累加求和
        vint32m1_t vs_3 = __riscv_vwredsum_vs_i16m1_i32m1(p3, vzero, vl);

        # 计算 sumi1
        int32_t sumi1 = sc[0] * __riscv_vmv_x_s_i32m1_i32(vs_0);
        # 计算 sumi2
        int32_t sumi2 = sc[1] * __riscv_vmv_x_s_i32m1_i32(vs_1);
        # 计算 sumi3
        int32_t sumi3 = sc[2] * __riscv_vmv_x_s_i32m1_i32(vs_2);
        # 计算 sumi4
        int32_t sumi4 = sc[3] * __riscv_vmv_x_s_i32m1_i32(vs_3);

        # 计算 sumf
        sumf += d * (sumi1 + sumi2 + sumi3 + sumi4);

    }

    # 将 sumf 的值赋给 s
    *s = sumf;

#else
    // 定义一个长度为 QK_K 的 int8_t 数组
    int8_t aux8[QK_K];
    // 定义一个长度为 16 的 int16_t 数组
    int16_t aux16[16];
    // 定义一个长度为 8 的 float 数组
    float   sums [8];
    // 将 sums 数组的内容全部初始化为 0
    memset(sums, 0, 8*sizeof(float));

    // 定义一个 float 类型的变量 sumf，并初始化为 0
    float sumf = 0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; ++i) {
        // 定义指向 x[i].qs 的指针 q4
        const uint8_t * restrict q4 = x[i].qs;
        // 定义指向 x[i].qh 的指针 hm
        const uint8_t * restrict hm = x[i].qh;
        // 定义指向 y[i].qs 的指针 q8
        const  int8_t * restrict q8 = y[i].qs;
        // 定义指向 aux8 的指针 a
        int8_t * restrict a = aux8;
        // 循环遍历 32 次
        for (int l = 0; l < 32; ++l) {
            // 将 q4[l] 的低 4 位存入 a 数组
            a[l+ 0] = q4[l] & 0xF;
            // 将 q4[l] 的高 4 位存入 a 数组
            a[l+32] = q4[l]  >> 4;
        }
        // 循环遍历 8 次
        for (int is = 0; is < 8; ++is) {
            // 计算掩码 m
            uint8_t m = 1 << is;
            // 循环遍历 8 次，根据掩码 m 对 a 数组进行调整
            for (int l = 0; l < 8; ++l) a[8*is + l] -= (hm[l] & m ? 0 : 16);
        }
// 计算两个向量的点积，结果保存在 s 中
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 定义一个常量 d，表示 y[i].d 乘以 x[i].d 的结果
    const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);
    // 定义一个指向 x[i].scales 的指针 sc
    const int8_t * restrict sc = x[i].scales;

    // 循环遍历 QK_K/16 次
    for (int j = 0; j < QK_K/16; ++j) {
        // 计算 dl，表示 d 乘以 sc[j]
        const float dl = d * sc[j];
        // 循环遍历 16 次，计算 aux16[l] = q8[l] * a[l]
        for (int l = 0; l < 16; ++l) aux16[l] = q8[l] * a[l];
        // 循环遍历 8 次，计算 sums[l] += dl * (aux16[l] + aux16[8+l])
        for (int l = 0; l <  8; ++l) sums[l] += dl * (aux16[l] + aux16[8+l]);
        // 更新 q8 和 a 的指针位置
        q8 += 16; a += 16;
    }
    // 循环遍历 8 次，计算 sumf += sums[l]
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值保存在指针 s 指向的位置
    *s = sumf;
}
    # 确保 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    # 定义指向 vx 和 vy 的指针，并限制其只能指向特定类型的数据
    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;

    # 计算 nb 的值，即 n 除以 QK_K 的商
    const int nb = n / QK_K;

    # 如果是 ARM 平台，执行以下代码
#ifdef __ARM_NEON

    # 初始化 sum 为 0
    float sum = 0;

    # 创建一个所有元素都为 0xF 的 16 位无符号整数向量
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    
    # 如果支持 ARM 的 dot product 指令，则创建一个所有元素都为 0 的 32 位有符号整数向量
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    # 创建一个所有元素都为 3 的 16 位无符号整数向量
    const uint8x16_t mone = vdupq_n_u8(3);

    # 定义一个名为 q6bytes 的自定义数据类型 ggml_int8x16x4_t
    ggml_uint8x16x4_t q6h; // 定义一个包含4个16字节的无符号8位整数向量的结构体

    for (int i = 0; i < nb; ++i) { // 循环开始，遍历nb次

        const float d_all = GGML_FP16_TO_FP32(x[i].d); // 将x[i].d的值从半精度浮点数转换为单精度浮点数，并赋给d_all

        const uint8_t * restrict q6 = x[i].ql; // 定义一个指向x[i].ql的无符号8位整数指针
        const uint8_t * restrict qh = x[i].qh; // 定义一个指向x[i].qh的无符号8位整数指针
        const int8_t  * restrict q8 = y[i].qs; // 定义一个指向y[i].qs的有符号8位整数指针

        const int8_t * restrict scale = x[i].scales; // 定义一个指向x[i].scales的有符号8位整数指针

        const ggml_int16x8x2_t q8sums = ggml_vld1q_s16_x2(y[i].bsums); // 从y[i].bsums加载16位有符号整数，存储到ggml_int16x8x2_t结构体中
        const int8x16_t scales = vld1q_s8(scale); // 从scale加载16个有符号8位整数，存储到int8x16_t类型的变量scales中
        const ggml_int16x8x2_t q6scales = {vmovl_s8(vget_low_s8(scales)), vmovl_s8(vget_high_s8(scales))}; // 将scales中的8位整数扩展为16位整数，存储到ggml_int16x8x2_t结构体中

        const int32x4_t prod = vaddq_s32(vaddq_s32(vmull_s16(vget_low_s16 (q8sums.val[0]), vget_low_s16 (q6scales.val[0])), // 计算两个16位整数向量的乘积，并将结果相加
                                                   vmull_s16(vget_high_s16(q8sums.val[0]), vget_high_s16(q6scales.val[0]))),
                                         vaddq_s32(vmull_s16(vget_low_s16 (q8sums.val[1]), vget_low_s16 (q6scales.val[1])), // 计算两个16位整数向量的乘积，并将结果相加
                                                   vmull_s16(vget_high_s16(q8sums.val[1]), vget_high_s16(q6scales.val[1])))); // 计算两个16位整数向量的乘积，并将结果相加
        // 计算prod中所有元素的和
        int32_t isum_mins = vaddvq_s32(prod);

        // 初始化isum为0
        int32_t isum = 0;

        // 遍历QK_K/128次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从qh中加载两个uint8x16向量
            ggml_uint8x16x2_t qhbits = ggml_vld1q_u8_x2(qh); qh += 32;
            // 从q6中加载四个uint8x16向量
            ggml_uint8x16x4_t q6bits = ggml_vld1q_u8_x4(q6); q6 += 64;
            // 从q8中加载四个int8x16向量
            ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8); q8 += 64;

            // 对qhbits中的值进行位运算和移位操作，得到q6h的四个uint8x16向量
            q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits.val[0]), 4);
            q6h.val[1] = vshlq_n_u8(vandq_u8(mone, qhbits.val[1]), 4);
            uint8x16_t shifted = vshrq_n_u8(qhbits.val[0], 2);
            q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
            shifted = vshrq_n_u8(qhbits.val[1], 2);
            q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

            // 对q6bits和q6h进行位运算和减法操作，得到q6bytes的四个int8x16向量
            //q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);
            //q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);
            //q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[2], m4b), q6h.val[2])), m32s);
            // 将 q6bits 和 q6h 进行位运算，并将结果转换为有符号字节类型
            q6bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0]));
            q6bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1]));
            q6bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[2], m4b), q6h.val[2]));
            q6bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[3], m4b), q6h.val[3]));

#if defined(__ARM_FEATURE_DOTPROD)
            // 如果支持 ARM dot product 指令集，则使用 dot product 进行计算
            // 计算 q6bytes 和 q8bytes 的点积，并乘以对应的 scale 值，然后累加到 isum 中
            isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
                    vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
            scale += 4;
#else
            // 如果不支持 ARM dot product 指令集，则使用传统的乘法和累加进行计算
            // 计算 q6bytes 和 q8bytes 的乘积，并将结果累加到 p0 和 p1 中
            int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                                     vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
            int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                                     vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1])));
// 计算isum的值，使用vaddvq_s16函数将p0和p1的值乘以scale数组中的值相加
isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
// 将scale指针向后移动两个位置
scale += 2;

// 使用vmull_s8函数计算q6bytes.val[2]和q8bytes.val[2]的乘积，并使用vaddq_s16函数将结果相加
int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                         vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2])));
// 同上，计算q6bytes.val[3]和q8bytes.val[3]的乘积，并使用vaddq_s16函数将结果相加
int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                         vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3])));
// 将p2和p3的值乘以scale数组中的值相加，并加到isum上
isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
// 将scale指针向后移动两个位置
scale += 2;

// 从q8指针处加载64个int8_t类型的值到q8bytes中
q8bytes = ggml_vld1q_s8_x4(q8); q8 += 64;

// 对qhbits.val[0]中的值进行右移4位，将结果存储到shifted中
shifted = vshrq_n_u8(qhbits.val[0], 4);
// 将mone和shifted进行按位与运算，并将结果左移4位，存储到q6h.val[0]中
q6h.val[0] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
// 同上，对qhbits.val[1]中的值进行右移4位，并将结果存储到shifted中
shifted = vshrq_n_u8(qhbits.val[1], 4);
// 将mone和shifted进行按位与运算，并将结果左移4位，存储到q6h.val[1]中
q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
// 同上，对qhbits.val[0]中的值进行右移6位，并将结果存储到shifted中
shifted = vshrq_n_u8(qhbits.val[0], 6);
// 将mone和shifted进行按位与运算，并将结果左移4位，存储到q6h.val[2]中
q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);
// 对qhbits.val[1]中的值进行右移6位，并将结果存储到shifted中
shifted = vshrq_n_u8(qhbits.val[1], 6);
// 将 shifted 中的值逻辑左移 4 位，并与 mone 进行按位与操作，结果存入 q6h.val[3]
q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

// 将 q6bits.val 中的值逻辑右移 4 位，与 q6h.val 中的值进行按位或操作，结果转换为有符号 8 位整数，存入 q6bytes.val 中
q6bytes.val[0] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[0]));
q6bytes.val[1] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[1]));
q6bytes.val[2] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[2], 4), q6h.val[2]));
q6bytes.val[3] = vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[3], 4), q6h.val[3]));

// 如果支持 ARM Dot Product 指令集
isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];
scale += 4;
// 如果定义了某个条件编译宏
#ifdef
    // 使用 SIMD 指令计算两个向量的点积，并将结果存储在 p 中
    const int32x4_t p = vdotq_s32(vzero, q6bytes.val[l], q8bytes.val[l]);
    // 将 p 中的四个元素相加，并乘以 scale 指向的值，然后加到 isum 上
    isum += vaddvq_s32(p) * *scale++;
    // 结束 if 块
    //}
#else
    // 计算两个向量的乘积，并将结果存储在 p0 和 p1 中
    p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                            vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
    p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                            vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1])));
    // 将 p0 和 p1 中的元素相加，并乘以 scale 指向的值，然后加到 isum 上
    isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
    // scale 指针向后移动两个位置
    scale += 2;

    // 计算两个向量的乘积，并将结果存储在 p2 和 p3 中
    p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                            vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2])));
    p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                            vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3])));
    // 将 p2 和 p3 中的元素相加，并乘以 scale 指向的值，然后加到 isum 上
    isum += vaddvq_s16(p2) * scale[0] + vaddvq_s16(p3) * scale[1];
    // scale 指针向后移动两个位置
    scale += 2;
#endif
    // 如果定义了 __AVX2__，则执行以下代码块
    const __m256i m4 = _mm256_set1_epi8(0xF); // 创建一个包含 0xF 的 32 位整数向量
    const __m256i m2 = _mm256_set1_epi8(3); // 创建一个包含 3 的 32 位整数向量
    const __m256i m32s = _mm256_set1_epi8(32); // 创建一个包含 32 的 32 位整数向量

    __m256 acc = _mm256_setzero_ps(); // 创建一个包含 0 的 32 位浮点数向量

    for (int i = 0; i < nb; ++i) { // 遍历 nb 次循环

        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d); // 计算 y[i].d 与 GGML_FP16_TO_FP32(x[i].d) 的乘积并赋值给 d

        const uint8_t * restrict q4 = x[i].ql; // 将 x[i].ql 赋值给指针 q4
        const uint8_t * restrict qh = x[i].qh; // 将 x[i].qh 赋值给指针 qh
        // 声明一个指向 y[i].qs 的 int8_t 类型的指针，并使用 restrict 限定符
        const int8_t  * restrict q8 = y[i].qs;

        // 从 x[i].scales 中加载 128 位数据到 __m128i 类型的变量 scales
        const __m128i scales = _mm_loadu_si128((const __m128i*)x[i].scales);

        // 初始化一个全零的 256 位整数变量 sumi
        __m256i sumi = _mm256_setzero_si256();

        // 初始化一个整数变量 is 为 0
        int is = 0;

        // 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从 scales 中根据 is + 0 获取对应的 128 位数据到 scale_0
            const __m128i scale_0 = _mm_shuffle_epi8(scales, get_scale_shuffle(is + 0));
            // 从 scales 中根据 is + 1 获取对应的 128 位数据到 scale_1
            const __m128i scale_1 = _mm_shuffle_epi8(scales, get_scale_shuffle(is + 1));
            // 从 scales 中根据 is + 2 获取对应的 128 位数据到 scale_2
            const __m128i scale_2 = _mm_shuffle_epi8(scales, get_scale_shuffle(is + 2));
            // 从 scales 中根据 is + 3 获取对应的 128 位数据到 scale_3
            const __m128i scale_3 = _mm_shuffle_epi8(scales, get_scale_shuffle(is + 3));
            // is 增加 4
            is += 4;

            // 从 q4 中加载 256 位数据到 q4bits1，并将 q4 指针向后移动 32 个字节
            const __m256i q4bits1 = _mm256_loadu_si256((const __m256i*)q4); q4 += 32;
            // 从 q4 中加载 256 位数据到 q4bits2，并将 q4 指针向后移动 32 个字节
            const __m256i q4bits2 = _mm256_loadu_si256((const __m256i*)q4); q4 += 32;
            // 从 qh 中加载 256 位数据到 q4bitsH，并将 qh 指针向后移动 32 个字节
            const __m256i q4bitsH = _mm256_loadu_si256((const __m256i*)qh); qh += 32;
# 使用位移和逻辑与操作将 q4bitsH 的每个字节左移 4 位，并保存到 q4h_0 中
const __m256i q4h_0 = _mm256_slli_epi16(_mm256_and_si256(q4bitsH, m2), 4);
# 使用位移和逻辑与操作将 q4bitsH 的每个字节右移 2 位，再进行位与操作和左移 4 位，并保存到 q4h_1 中
const __m256i q4h_1 = _mm256_slli_epi16(_mm256_and_si256(_mm256_srli_epi16(q4bitsH, 2), m2), 4);
# 使用位移和逻辑与操作将 q4bitsH 的每个字节右移 4 位，再进行位与操作和左移 4 位，并保存到 q4h_2 中
const __m256i q4h_2 = _mm256_slli_epi16(_mm256_and_si256(_mm256_srli_epi16(q4bitsH, 4), m2), 4);
# 使用位移和逻辑与操作将 q4bitsH 的每个字节右移 6 位，再进行位与操作和左移 4 位，并保存到 q4h_3 中
const __m256i q4h_3 = _mm256_slli_epi16(_mm256_and_si256(_mm256_srli_epi16(q4bitsH, 6), m2), 4);

# 使用位与和位或操作将 q4bits1 和 m4 进行组合，并保存到 q4_0 中
const __m256i q4_0 = _mm256_or_si256(_mm256_and_si256(q4bits1, m4), q4h_0);
# 使用位与和位或操作将 q4bits2 和 m4 进行组合，并保存到 q4_1 中
const __m256i q4_1 = _mm256_or_si256(_mm256_and_si256(q4bits2, m4), q4h_1);
# 使用位移、位与和位或操作将 q4bits1 右移 4 位和 m4 进行组合，并保存到 q4_2 中
const __m256i q4_2 = _mm256_or_si256(_mm256_and_si256(_mm256_srli_epi16(q4bits1, 4), m4), q4h_2);
# 使用位移、位与和位或操作将 q4bits2 右移 4 位和 m4 进行组合，并保存到 q4_3 中
const __m256i q4_3 = _mm256_or_si256(_mm256_and_si256(_mm256_srli_epi16(q4bits2, 4), m4), q4h_3);

# 从内存中加载 32 个字节到 q8_0 中，并将指针 q8 向后移动 32 个字节
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
# 从内存中加载 32 个字节到 q8_1 中，并将指针 q8 向后移动 32 个字节
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
# 从内存中加载 32 个字节到 q8_2 中，并将指针 q8 向后移动 32 个字节
const __m256i q8_2 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;
# 从内存中加载 32 个字节到 q8_3 中，并将指针 q8 向后移动 32 个字节
const __m256i q8_3 = _mm256_loadu_si256((const __m256i*)q8); q8 += 32;

# 使用 m32s 对 q8_0 进行乘法累加，并保存到 q8s_0 中
__m256i q8s_0 = _mm256_maddubs_epi16(m32s, q8_0);
# 使用 m32s 对 q8_1 进行乘法累加，并保存到 q8s_1 中
__m256i q8s_1 = _mm256_maddubs_epi16(m32s, q8_1);
# 使用 m32s 对 q8_2 进行乘法累加，并保存到 q8s_2 中
__m256i q8s_2 = _mm256_maddubs_epi16(m32s, q8_2);
# 使用 m32s 对 q8_3 进行乘法累加，并保存到 q8s_3 中
__m256i q8s_3 = _mm256_maddubs_epi16(m32s, q8_3);
# 使用 AVX2 指令集进行向量化计算，将 q4_0 和 q8_0 的每个字节相乘并相加，结果存储在 p16_0 中
__m256i p16_0 = _mm256_maddubs_epi16(q4_0, q8_0);

# 类似地，将 q4_1 和 q8_1 的每个字节相乘并相加，结果存储在 p16_1 中
__m256i p16_1 = _mm256_maddubs_epi16(q4_1, q8_1);

# 继续进行相乘相加操作，将结果存储在 p16_2 和 p16_3 中
__m256i p16_2 = _mm256_maddubs_epi16(q4_2, q8_2);
__m256i p16_3 = _mm256_maddubs_epi16(q4_3, q8_3);

# 对 p16_0 到 p16_3 分别进行减法操作
p16_0 = _mm256_sub_epi16(p16_0, q8s_0);
p16_1 = _mm256_sub_epi16(p16_1, q8s_1);
p16_2 = _mm256_sub_epi16(p16_2, q8s_2);
p16_3 = _mm256_sub_epi16(p16_3, q8s_3);

# 将 p16_0 到 p16_3 分别与 scale_0 到 scale_3 进行乘法操作，并将结果存储在相应的变量中
p16_0 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_0), p16_0);
p16_1 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_1), p16_1);
p16_2 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_2), p16_2);
p16_3 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_3), p16_3);

# 将 p16_0 到 p16_3 的结果分别与 sumi 进行加法操作，并将结果存储在 sumi 中
sumi = _mm256_add_epi32(sumi, _mm256_add_epi32(p16_0, p16_1));
sumi = _mm256_add_epi32(sumi, _mm256_add_epi32(p16_2, p16_3);
    # 使用 AVX 指令集进行计算
    acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi), acc);
    }

    # 如果编译器支持 AVX 指令集
    *s = hsum_float_8(acc);

#elif defined __AVX__

    # 定义一些常量
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m3 = _mm_set1_epi8(3);
    const __m128i m32s = _mm_set1_epi8(32);
    const __m128i m2 = _mm_set1_epi8(2);

    # 初始化一个 256 位的浮点数向量为 0
    __m256 acc = _mm256_setzero_ps();

    # 遍历 nb 次
    for (int i = 0; i < nb; ++i) {

        # 计算 d 的值
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        # 定义指向 x[i].ql 和 x[i].qh 的指针
        const uint8_t * restrict q4 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        // 声明并初始化指向数组 y[i].qs 的指针
        const int8_t  * restrict q8 = y[i].qs;

        // 从数组 x[i].scales 中加载 128 位数据到 __m128i 类型的变量 scales 中
        const __m128i scales = _mm_loadu_si128((const __m128i*)x[i].scales);

        // 初始化两个 128 位的变量 sumi_0 和 sumi_1 为全零
        __m128i sumi_0 = _mm_setzero_si128();
        __m128i sumi_1 = _mm_setzero_si128();

        // 初始化一个 128 位的变量 shuffle，其中包含两个 64 位的值
        __m128i shuffle = _mm_set_epi64x(0x0101010101010101, 0x0000000000000000);
        // 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 从指针 qh 指向的地址加载 128 位数据到 __m128i 类型的变量 q4bitsH_0，然后移动指针 qh
            const __m128i q4bitsH_0 = _mm_loadu_si128((const __m128i*)qh); qh += 16;
            // 从指针 qh 指向的地址加载 128 位数据到 __m128i 类型的变量 q4bitsH_1，然后移动指针 qh
            const __m128i q4bitsH_1 = _mm_loadu_si128((const __m128i*)qh); qh += 16;

            // 对 q4bitsH_0 和 q4bitsH_1 进行位与运算，并将结果左移 4 位，然后存入 q4h_0 和 q4h_1 中
            const __m128i q4h_0 = _mm_slli_epi16(_mm_and_si128(q4bitsH_0, m3), 4);
            const __m128i q4h_1 = _mm_slli_epi16(_mm_and_si128(q4bitsH_1, m3), 4);
            // 对 q4bitsH_0 和 q4bitsH_1 进行右移 2 位，再进行位与运算，并将结果左移 4 位，然后存入 q4h_2 和 q4h_3 中
            const __m128i q4h_2 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_0, 2), m3), 4);
            const __m128i q4h_3 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_1, 2), m3), 4);
            // 对 q4bitsH_0 和 q4bitsH_1 进行右移 4 位，再进行位与运算，并将结果左移 4 位，然后存入 q4h_4 和 q4h_5 中
            const __m128i q4h_4 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_0, 4), m3), 4);
            const __m128i q4h_5 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_1, 4), m3), 4);
            // 对 q4bitsH_0 和 q4bitsH_1 进行右移 6 位，再进行位与运算，并将结果左移 4 位，然后存入 q4h_6 中
            const __m128i q4h_6 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_0, 6), m3), 4);
// 使用 _mm_slli_epi16 函数将 q4bitsH_1 右移 6 位并与 m3 进行与操作，然后左移 4 位，得到 q4h_7
const __m128i q4h_7 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH_1, 6), m3), 4);

// 从 q4 指针指向的地址加载 16 字节数据到 q4bits1_0，然后将指针向后移动 16 个字节
const __m128i q4bits1_0 = _mm_loadu_si128((const __m128i*)q4); q4 += 16;
// 依此类推，加载 q4bits1_1、q4bits2_0、q4bits2_1
const __m128i q4bits1_1 = _mm_loadu_si128((const __m128i*)q4); q4 += 16;
const __m128i q4bits2_0 = _mm_loadu_si128((const __m128i*)q4); q4 += 16;
const __m128i q4bits2_1 = _mm_loadu_si128((const __m128i*)q4); q4 += 16;

// 对加载的数据进行位运算和或运算，得到 q4_0 到 q4_7
const __m128i q4_0 = _mm_or_si128(_mm_and_si128(q4bits1_0, m4), q4h_0);
const __m128i q4_1 = _mm_or_si128(_mm_and_si128(q4bits1_1, m4), q4h_1);
const __m128i q4_2 = _mm_or_si128(_mm_and_si128(q4bits2_0, m4), q4h_2);
const __m128i q4_3 = _mm_or_si128(_mm_and_si128(q4bits2_1, m4), q4h_3);
const __m128i q4_4 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(q4bits1_0, 4), m4), q4h_4);
const __m128i q4_5 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(q4bits1_1, 4), m4), q4h_5);
const __m128i q4_6 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(q4bits2_0, 4), m4), q4h_6);
const __m128i q4_7 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(q4bits2_1, 4), m4), q4h_7);

// 依次加载 q8 指针指向的数据到 q8_0 到 q8_3
const __m128i q8_0 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_1 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_2 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_3 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
// 从内存中加载16字节的数据到__m128i类型的寄存器中，并将q8指针向后移动16个字节
const __m128i q8_4 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_5 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_6 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;
const __m128i q8_7 = _mm_loadu_si128((const __m128i*)q8); q8 += 16;

// 使用m32s和q8_x进行有符号字节乘法并累加，得到__m128i类型的结果
__m128i q8s_0 = _mm_maddubs_epi16(m32s, q8_0);
__m128i q8s_1 = _mm_maddubs_epi16(m32s, q8_1);
__m128i q8s_2 = _mm_maddubs_epi16(m32s, q8_2);
__m128i q8s_3 = _mm_maddubs_epi16(m32s, q8_3);
__m128i q8s_4 = _mm_maddubs_epi16(m32s, q8_4);
__m128i q8s_5 = _mm_maddubs_epi16(m32s, q8_5);
__m128i q8s_6 = _mm_maddubs_epi16(m32s, q8_6);
__m128i q8s_7 = _mm_maddubs_epi16(m32s, q8_7);

// 使用q4_x和q8_x进行有符号字节乘法并累加，得到__m128i类型的结果
__m128i p16_0 = _mm_maddubs_epi16(q4_0, q8_0);
__m128i p16_1 = _mm_maddubs_epi16(q4_1, q8_1);
__m128i p16_2 = _mm_maddubs_epi16(q4_2, q8_2);
__m128i p16_3 = _mm_maddubs_epi16(q4_3, q8_3);
__m128i p16_4 = _mm_maddubs_epi16(q4_4, q8_4);
__m128i p16_5 = _mm_maddubs_epi16(q4_5, q8_5);
# 使用 MMX 指令对两个 128 位寄存器中的无符号字节进行乘法，并将结果相加
__m128i p16_6 = _mm_maddubs_epi16(q4_6, q8_6);
__m128i p16_7 = _mm_maddubs_epi16(q4_7, q8_7);

# 对两个 128 位寄存器中的有符号 16 位整数进行减法
p16_0 = _mm_sub_epi16(p16_0, q8s_0);
p16_1 = _mm_sub_epi16(p16_1, q8s_1);
p16_2 = _mm_sub_epi16(p16_2, q8s_2);
p16_3 = _mm_sub_epi16(p16_3, q8s_3);
p16_4 = _mm_sub_epi16(p16_4, q8s_4);
p16_5 = _mm_sub_epi16(p16_5, q8s_5);
p16_6 = _mm_sub_epi16(p16_6, q8s_6);
p16_7 = _mm_sub_epi16(p16_7, q8s_7);

# 使用 MMX 指令对两个 128 位寄存器中的字节进行按位混洗，并将结果存储在新的寄存器中
const __m128i scale_0 = _mm_shuffle_epi8(scales, shuffle);
shuffle = _mm_add_epi8(shuffle, m2);
const __m128i scale_1 = _mm_shuffle_epi8(scales, shuffle);
shuffle = _mm_add_epi8(shuffle, m2);
const __m128i scale_2 = _mm_shuffle_epi8(scales, shuffle);
shuffle = _mm_add_epi8(shuffle, m2);
const __m128i scale_3 = _mm_shuffle_epi8(scales, shuffle);
shuffle = _mm_add_epi8(shuffle, m2);
# 使用_mm_madd_epi16函数将两个16位整数的低位乘积相加，并将结果存储在p16_0中
p16_0 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_0), p16_0);
# 使用_mm_unpackhi_epi64函数将64位整数的高位和低位交替组合，并将结果存储在p16_1中
p16_1 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_0, scale_0)), p16_1);
# 使用_mm_madd_epi16函数将两个16位整数的低位乘积相加，并将结果存储在p16_2中
p16_2 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_1), p16_2);
# 使用_mm_unpackhi_epi64函数将64位整数的高位和低位交替组合，并将结果存储在p16_3中
p16_3 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_1, scale_1)), p16_3);
# 使用_mm_madd_epi16函数将两个16位整数的低位乘积相加，并将结果存储在p16_4中
p16_4 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_2), p16_4);
# 使用_mm_unpackhi_epi64函数将64位整数的高位和低位交替组合，并将结果存储在p16_5中
p16_5 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_2, scale_2)), p16_5);
# 使用_mm_madd_epi16函数将两个16位整数的低位乘积相加，并将结果存储在p16_6中
p16_6 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_3), p16_6);
# 使用_mm_unpackhi_epi64函数将64位整数的高位和低位交替组合，并将结果存储在p16_7中
p16_7 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_3, scale_3)), p16_7);

# 使用_mm_add_epi32函数将两个32位整数相加，并将结果存储在sumi_0中
sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p16_0, p16_2));
# 使用_mm_add_epi32函数将两个32位整数相加，并将结果存储在sumi_1中
sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p16_1, p16_3));
# 使用_mm_add_epi32函数将两个32位整数相加，并将结果存储在sumi_0中
sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p16_4, p16_6));
# 使用_mm_add_epi32函数将两个32位整数相加，并将结果存储在sumi_1中
sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p16_5, p16_7);

# 使用MM256_SET_M128I函数将两个128位整数拼接成256位整数，并将结果存储在sumi中
__m256i sumi = MM256_SET_M128I(sumi_1, sumi_0);
# 使用_mm256_broadcast_ss函数将单精度浮点数值扩展为256位浮点数，并将结果存储在acc中
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi)), acc);
    *s = hsum_float_8(acc);
    // 将累加器中的8个浮点数相加，并将结果存储在指针s所指向的位置

#elif defined __riscv_v_intrinsic
    // 如果定义了__riscv_v_intrinsic，则执行以下代码
    float sumf = 0;
    // 初始化一个浮点数sumf为0
    for (int i = 0; i < nb; ++i) {
        // 循环遍历nb次
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
        // 将x[i].d转换为32位浮点数，与y[i].d相乘，结果存储在d中

        const uint8_t * restrict q6 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        const  int8_t * restrict q8 = y[i].qs;
        // 定义并初始化指向x[i].ql, x[i].qh, y[i].qs的指针

        const int8_t * restrict scale = x[i].scales;
        // 定义并初始化指向x[i].scales的指针

        size_t vl;
        // 定义一个size_t类型的变量vl

        vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);
        // 使用__riscv_vmv_v_x_i32m1函数初始化一个vint32m1_t类型的变量vzero
        // 初始化变量 sum_t 和 is
        int sum_t = 0;
        int is = 0;

        // 循环遍历 QK_K/128 次
        for (int j = 0; j < QK_K/128; ++j) {

            // 设置向量长度为 32
            vl = 32;

            // 加载 qh
            vuint8m1_t qh_x = __riscv_vle8_v_u8m1(qh, vl);

            // 加载 Q6 的前 32 个元素和后 32 个元素
            vuint8m1_t q6_0 = __riscv_vle8_v_u8m1(q6, vl);
            vuint8m1_t q6_1 = __riscv_vle8_v_u8m1(q6+32, vl);

            // 对 Q6 的前 32 个元素和后 32 个元素进行位与和右移操作
            vuint8m1_t q6a_0 = __riscv_vand_vx_u8m1(q6_0, 0x0F, vl);
            vuint8m1_t q6a_1 = __riscv_vand_vx_u8m1(q6_1, 0x0F, vl);
            vuint8m1_t q6s_0 = __riscv_vsrl_vx_u8m1(q6_0, 0x04, vl);
            vuint8m1_t q6s_1 = __riscv_vsrl_vx_u8m1(q6_1, 0x04, vl);

            // 对 qh_x 进行位与操作
            vuint8m1_t qh_0 = __riscv_vand_vx_u8m1(qh_x, 0x03, vl);
// 从 qh_x 中右移 2 位并与 0x03 进行按位与操作，得到 qh_1
vuint8m1_t qh_1 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(qh_x, 0x2, vl), 0x03 , vl);
// 从 qh_x 中右移 4 位并与 0x03 进行按位与操作，得到 qh_2
vuint8m1_t qh_2 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(qh_x, 0x4, vl), 0x03 , vl);
// 从 qh_x 中右移 6 位并与 0x03 进行按位与操作，得到 qh_3
vuint8m1_t qh_3 = __riscv_vand_vx_u8m1(__riscv_vsrl_vx_u8m1(qh_x, 0x6, vl), 0x03 , vl);

// 将 q6a_0 和 qh_0 左移 4 位并进行按位或操作，得到 qhi_0
vuint8m1_t qhi_0 = __riscv_vor_vv_u8m1(q6a_0, __riscv_vsll_vx_u8m1(qh_0, 0x04, vl), vl);
// 将 q6a_1 和 qh_1 左移 4 位并进行按位或操作，得到 qhi_1
vuint8m1_t qhi_1 = __riscv_vor_vv_u8m1(q6a_1, __riscv_vsll_vx_u8m1(qh_1, 0x04, vl), vl);
// 将 q6s_0 和 qh_2 左移 4 位并进行按位或操作，得到 qhi_2
vuint8m1_t qhi_2 = __riscv_vor_vv_u8m1(q6s_0, __riscv_vsll_vx_u8m1(qh_2, 0x04, vl), vl);
// 将 q6s_1 和 qh_3 左移 4 位并进行按位或操作，得到 qhi_3
vuint8m1_t qhi_3 = __riscv_vor_vv_u8m1(q6s_1, __riscv_vsll_vx_u8m1(qh_3, 0x04, vl), vl);

// 将 qhi_0 转换为有符号整数并减去 32，得到 a_0
vint8m1_t a_0 = __riscv_vsub_vx_i8m1(__riscv_vreinterpret_v_u8m1_i8m1(qhi_0), 32, vl);
// 将 qhi_1 转换为有符号整数并减去 32，得到 a_1
vint8m1_t a_1 = __riscv_vsub_vx_i8m1(__riscv_vreinterpret_v_u8m1_i8m1(qhi_1), 32, vl);
// 将 qhi_2 转换为有符号整数并减去 32，得到 a_2
vint8m1_t a_2 = __riscv_vsub_vx_i8m1(__riscv_vreinterpret_v_u8m1_i8m1(qhi_2), 32, vl);
// 将 qhi_3 转换为有符号整数并减去 32，得到 a_3
vint8m1_t a_3 = __riscv_vsub_vx_i8m1(__riscv_vreinterpret_v_u8m1_i8m1(qhi_3), 32, vl);

// 加载 Q8 并将 a_0 与之相乘，得到 va_q_0
vint16m2_t va_q_0 = __riscv_vwmul_vv_i16m2(a_0, __riscv_vle8_v_i8m1(q8, vl), vl);
// 加载 Q8 并将 a_1 与之相乘，得到 va_q_1
vint16m2_t va_q_1 = __riscv_vwmul_vv_i16m2(a_1, __riscv_vle8_v_i8m1(q8+32, vl), vl);
// 加载 Q8 并将 a_2 与之相乘，得到 va_q_2
vint16m2_t va_q_2 = __riscv_vwmul_vv_i16m2(a_2, __riscv_vle8_v_i8m1(q8+64, vl), vl);
// 加载 Q8 并将 a_3 与之相乘，得到 va_q_3
vint16m2_t va_q_3 = __riscv_vwmul_vv_i16m2(a_3, __riscv_vle8_v_i8m1(q8+96, vl), vl);
# 设置变量 vl 的值为 16
vl = 16;

# 使用 SIMD 指令对两个 16 位整数向量进行乘法操作，并将结果存储在 32 位整数向量中
vaux_0 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_0, 0), scale[is+0], vl);
vaux_1 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_0, 1), scale[is+1], vl);
vaux_2 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_1, 0), scale[is+2], vl);
vaux_3 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_1, 1), scale[is+3], vl);
vaux_4 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_2, 0), scale[is+4], vl);
vaux_5 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_2, 1), scale[is+5], vl);
vaux_6 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_3, 0), scale[is+6], vl);
vaux_7 = __riscv_vwmul_vx_i32m2(__riscv_vget_v_i16m2_i16m1(va_q_3, 1), scale[is+7], vl);

# 使用 SIMD 指令对两个 32 位整数向量进行加法操作，并将结果存储在 32 位整数向量中
isum0 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(vaux_0, vaux_1, vl), vzero, vl);
isum1 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(vaux_2, vaux_3, vl), isum0, vl);
isum2 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(vaux_4, vaux_5, vl), isum1, vl);
isum3 = __riscv_vredsum_vs_i32m2_i32m1(__riscv_vadd_vv_i32m2(vaux_6, vaux_7, vl), isum2, vl);

# 将 isum3 中的值加到 sum_t 中
sum_t += __riscv_vmv_x_s_i32m1_i32(isum3);

# 对变量 q6、qh、q8 和 is 进行增加操作
q6 += 64;   qh += 32;   q8 += 128;   is=8;
    }

    sumf += d * sum_t;

}

*s = sumf;

#else

int8_t  aux8[QK_K];
int16_t aux16[8];
float   sums [8];
int32_t aux32[8];
// 初始化 sums 数组，将其所有元素置为 0
memset(sums, 0, 8*sizeof(float));

float sumf = 0;
// 遍历 nb 次循环
for (int i = 0; i < nb; ++i) {
    // 获取指向 x[i].ql 的指针
    const uint8_t * restrict q4 = x[i].ql;
    // 获取指向 x[i].qh 的指针
    const uint8_t * restrict qh = x[i].qh;
        // 声明一个指向 y[i].qs 的 int8_t 类型的指针 q8
        const int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组的前 8 个元素全部置为 0
        memset(aux32, 0, 8*sizeof(int32_t));
        // 声明一个指向 aux8 的 int8_t 类型的指针 a
        int8_t * restrict a = aux8;
        // 循环遍历 QK_K，每次增加 128
        for (int j = 0; j < QK_K; j += 128) {
            // 内层循环，遍历 32 次
            for (int l = 0; l < 32; ++l) {
                // 对 a 数组进行赋值操作
                a[l +  0] = (int8_t)((q4[l +  0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
                a[l + 32] = (int8_t)((q4[l + 32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
                a[l + 64] = (int8_t)((q4[l +  0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
                a[l + 96] = (int8_t)((q4[l + 32] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
            }
            // a 指针向后移动 128 个位置
            a  += 128;
            // q4 指针向后移动 64 个位置
            q4 += 64;
            // qh 指针向后移动 32 个位置
            qh += 32;
        }
        // 将 a 指针重新指向 aux8
        a = aux8;
        // 初始化 is 为 0
        int is = 0;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 获取 x[i].scales[is] 的值，赋给 scale
            int scale = x[i].scales[is++];
            // 对 aux16 数组进行赋值操作
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 对 aux32 数组进行累加操作
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
    // 对 q8 和 a 分别加 8
    q8 += 8; a += 8;
    // 对于 l 从 0 到 7，计算 aux16 数组的值
    for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
    // 对于 l 从 0 到 7，计算 aux32 数组的值
    for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
    // 对 q8 和 a 分别加 8
    q8 += 8; a += 8;
    // 计算 d 的值
    const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
    // 对于 l 从 0 到 7，计算 sums 数组的值
    for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    // 对于 l 从 0 到 7，计算 sumf 的值
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将 sumf 的值赋给指针 s 所指向的位置
    *s = sumf;
#endif
}

#else

// 定义函数 ggml_vec_dot_q6_K_q8_K，参数为 n, s, vx, vy
void ggml_vec_dot_q6_K_q8_K(const int n, float * restrict s, const void * restrict vx, const void * restrict vy) {
    // 断言 n 能被 QK_K 整除
    assert(n % QK_K == 0);

    // 定义指针 x 指向 vx，指针 y 指向 vy
    const block_q6_K * restrict x = vx;
    const block_q8_K * restrict y = vy;
    // 将 n 除以 QK_K 的结果赋值给常量 nb
    const int nb = n / QK_K;

#ifdef __ARM_NEON
    // 初始化浮点数变量 sum 为 0
    float sum = 0;

    // 初始化 16 个 8 位无符号整数的向量 m4b，每个元素都是 0xF
    const uint8x16_t m4b = vdupq_n_u8(0xF);
    // 初始化 16 个 8 位有符号整数的向量 m32s，每个元素都是 32
    const int8x16_t  m32s = vdupq_n_s8(32);
#if defined(__ARM_FEATURE_DOTPROD)
    // 初始化 4 个 32 位有符号整数的向量 vzero，每个元素都是 0
    const int32x4_t  vzero = vdupq_n_s32(0);
#endif

    // 初始化 16 个 8 位无符号整数的向量 mone，每个元素都是 3
    const uint8x16_t mone = vdupq_n_u8(3);

    // 定义 ggml_int8x16x4_t 和 ggml_uint8x16x4_t 类型的变量 q6bytes 和 q6h
    ggml_int8x16x4_t q6bytes;
    ggml_uint8x16x4_t q6h;

    // 循环，从 0 到 nb-1
    for (int i = 0; i < nb; ++i) {
// 将x[i].d转换为浮点数类型d_all
const float d_all = (float)x[i].d;

// 从结构体x[i]中获取指向uint8_t类型的指针q6
const uint8_t * restrict q6 = x[i].ql;

// 从结构体x[i]中获取指向uint8_t类型的指针qh
const uint8_t * restrict qh = x[i].qh;

// 从结构体y[i]中获取指向int8_t类型的指针q8
const int8_t  * restrict q8 = y[i].qs;

// 从结构体x[i]中获取指向int8_t类型的指针scale
const int8_t * restrict scale = x[i].scales;

// 初始化isum为0
int32_t isum = 0;

// 从qh中加载16个uint8_t类型的数据到qhbits
uint8x16_t qhbits = vld1q_u8(qh);

// 从q6中加载32个uint8_t类型的数据到q6bits
ggml_uint8x16x2_t q6bits = ggml_vld1q_u8_x2(q6);

// 从q8中加载64个int8_t类型的数据到q8bytes
ggml_int8x16x4_t q8bytes = ggml_vld1q_s8_x4(q8);

// 将qhbits中的数据进行位运算和移位操作，存储到q6h.val[0]中
q6h.val[0] = vshlq_n_u8(vandq_u8(mone, qhbits), 4);

// 将qhbits中的数据右移2位，存储到shifted中
uint8x16_t shifted = vshrq_n_u8(qhbits, 2);

// 将shifted中的数据进行位运算和移位操作，存储到q6h.val[1]中
q6h.val[1] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

// 将shifted中的数据右移4位，存储到shifted中
shifted = vshrq_n_u8(qhbits, 4);

// 将shifted中的数据进行位运算和移位操作，存储到q6h.val[2]中
q6h.val[2] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

// 将shifted中的数据右移6位，存储到shifted中
shifted = vshrq_n_u8(qhbits, 6);
# 将 vandq_u8(mone, shifted) 的结果左移 4 位，存入 q6h.val[3]
q6h.val[3] = vshlq_n_u8(vandq_u8(mone, shifted), 4);

# 将 q6bits.val[0] 和 m4b 进行按位与操作，然后与 q6h.val[0] 进行按位或操作，再取反，最后减去 m32s，存入 q6bytes.val[0]
q6bytes.val[0] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[0], m4b), q6h.val[0])), m32s);

# 类似上一条语句，处理 q6bits.val[1]，存入 q6bytes.val[1]
q6bytes.val[1] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vandq_u8(q6bits.val[1], m4b), q6h.val[1])), m32s);

# 类似上一条语句，处理 q6bits.val[0]，存入 q6bytes.val[2]
q6bytes.val[2] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[0], 4), q6h.val[2])), m32s);

# 类似上一条语句，处理 q6bits.val[1]，存入 q6bytes.val[3]
q6bytes.val[3] = vsubq_s8(vreinterpretq_s8_u8(vorrq_u8(vshrq_n_u8(q6bits.val[1], 4), q6h.val[3])), m32s);

# 如果支持 ARM dot product 指令集
# 计算 q6bytes 和 q8bytes 的点积，然后分别乘以对应的 scale 值，最后将结果累加到 isum 中
isum += vaddvq_s32(vdotq_s32(vzero, q6bytes.val[0], q8bytes.val[0])) * scale[0] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[1], q8bytes.val[1])) * scale[1] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[2], q8bytes.val[2])) * scale[2] +
        vaddvq_s32(vdotq_s32(vzero, q6bytes.val[3], q8bytes.val[3])) * scale[3];

# 如果不支持 ARM dot product 指令集
# 计算 q6bytes 和 q8bytes 的点积，然后分别乘以对应的 scale 值，最后将结果累加到 isum 中
int16x8_t p0 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[0]), vget_low_s8 (q8bytes.val[0])),
                         vmull_s8(vget_high_s8(q6bytes.val[0]), vget_high_s8(q8bytes.val[0])));
int16x8_t p1 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[1]), vget_low_s8 (q8bytes.val[1])),
                         vmull_s8(vget_high_s8(q6bytes.val[1]), vget_high_s8(q8bytes.val[1]));
isum += vaddvq_s16(p0) * scale[0] + vaddvq_s16(p1) * scale[1];
    // 使用 SIMD 指令进行并行计算，将两个 int8x16_t 类型的寄存器中的元素逐个相乘，然后相加
    int16x8_t p2 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[2]), vget_low_s8 (q8bytes.val[2])),
                             vmull_s8(vget_high_s8(q6bytes.val[2]), vget_high_s8(q8bytes.val[2])));
    // 同上，计算另外一个 int8x16_t 类型的寄存器
    int16x8_t p3 = vaddq_s16(vmull_s8(vget_low_s8 (q6bytes.val[3]), vget_low_s8 (q8bytes.val[3])),
                             vmull_s8(vget_high_s8(q6bytes.val[3]), vget_high_s8(q8bytes.val[3])));
    // 将 p2 和 p3 寄存器中的元素相加，并乘以对应的 scale 值，然后累加到 isum 中
    isum += vaddvq_s16(p2) * scale[2] + vaddvq_s16(p3) * scale[3];

    // 将 isum 乘以 d_all 和 y[i].d，然后累加到 sum 中
    sum += isum * d_all * y[i].d;

    // 使用 AVX2 指令集进行并行计算，设置一些常量值和初始值
    const __m256i m4 = _mm256_set1_epi8(0xF);
    const __m256i m2 = _mm256_set1_epi8(3);
    const __m256i m32s = _mm256_set1_epi8(32);
    __m256 acc = _mm256_setzero_ps();
# 循环遍历 nb 次，nb 为某个变量的值
for (int i = 0; i < nb; ++i) {
    # 计算 y[i].d 与 x[i].d 的乘积，并赋值给 d
    const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

    # 定义指向 x[i].ql, x[i].qh, y[i].qs 的指针
    const uint8_t * restrict q4 = x[i].ql;
    const uint8_t * restrict qh = x[i].qh;
    const int8_t  * restrict q8 = y[i].qs;

    # 使用 x[i].scales 中的值创建 __m64 类型的变量
    const __m64 scales_1 = _mm_set1_pi8(x[i].scales[0]);
    const __m64 scales_2 = _mm_set1_pi8(x[i].scales[1]);
    const __m64 scales_3 = _mm_set1_pi8(x[i].scales[2]);
    const __m64 scales_4 = _mm_set1_pi8(x[i].scales[3]);

    # 创建一个全零的 __m256i 类型的变量
    __m256i sumi = _mm256_setzero_si256();

    # 使用 x[i].scales 中的值创建 __m128i 类型的变量
    const __m128i scale_0 = _mm_set_epi64(scales_2, scales_1);
    const __m128i scale_1 = _mm_set_epi64(scales_4, scales_3);

    # 从内存中加载 q4 的值到 __m256i 类型的变量 q4bits1
    const __m256i q4bits1 = _mm256_loadu_si256((const __m256i*)q4);
// 从内存中加载128位整数到寄存器中
const __m128i q4bitsH = _mm_loadu_si128((const __m128i*)qh);

// 对128位整数进行位运算和移位操作，得到新的128位整数
const __m256i q4h_0 = _mm256_slli_epi16(_mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q4bitsH, 2), q4bitsH), m2), 4);
const __m256i q4h_1 = _mm256_slli_epi16(_mm256_and_si256(MM256_SET_M128I(_mm_srli_epi16(q4bitsH, 6), _mm_srli_epi16(q4bitsH, 4)), m2), 4);

// 对256位整数进行位运算和移位操作，得到新的256位整数
const __m256i q4_0 = _mm256_or_si256(_mm256_and_si256(q4bits1, m4), q4h_0);
const __m256i q4_1 = _mm256_or_si256(_mm256_and_si256(_mm256_srli_epi16(q4bits1, 4), m4), q4h_1);

// 从内存中加载256位整数到寄存器中
const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

// 对256位整数进行有符号乘法和累加操作，得到新的256位整数
__m256i q8s_0 = _mm256_maddubs_epi16(m32s, q8_0);
__m256i q8s_1 = _mm256_maddubs_epi16(m32s, q8_1);

// 对256位整数进行有符号乘法和累加操作，得到新的256位整数
__m256i p16_0 = _mm256_maddubs_epi16(q4_0, q8_0);
__m256i p16_1 = _mm256_maddubs_epi16(q4_1, q8_1);

// 对256位整数进行减法操作，得到新的256位整数
p16_0 = _mm256_sub_epi16(p16_0, q8s_0);
p16_1 = _mm256_sub_epi16(p16_1, q8s_1);
        # 使用_mm256_cvtepi8_epi16将scale_0转换为16位整数，然后与p16_0进行逐元素乘法和累加
        p16_0 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_0), p16_0);
        # 使用_mm256_cvtepi8_epi16将scale_1转换为16位整数，然后与p16_1进行逐元素乘法和累加
        p16_1 = _mm256_madd_epi16(_mm256_cvtepi8_epi16(scale_1), p16_1);

        # 将p16_0和p16_1的结果进行逐元素相加，然后与sumi进行逐元素相加
        sumi = _mm256_add_epi32(sumi, _mm256_add_epi32(p16_0, p16_1));

        # 使用_mm256_broadcast_ss将d的值广播到所有元素，然后与sumi进行逐元素乘法和累加，结果累加到acc中
        acc = _mm256_fmadd_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(sumi), acc);
    }

    # 计算acc中所有元素的和并返回结果
    *s = hsum_float_8(acc);

# 如果定义了__AVX__，则执行以下代码
#elif defined __AVX__

    # 创建常量m4、m2、m32s，分别用于后续的计算
    const __m128i m4 = _mm_set1_epi8(0xF);
    const __m128i m2 = _mm_set1_epi8(3);
    const __m128i m32s = _mm_set1_epi8(32);

    # 初始化acc为全0的256位浮点数向量
    __m256 acc = _mm256_setzero_ps();

    # 循环遍历nb次
    for (int i = 0; i < nb; ++i) {
        // 计算浮点数乘积
        const float d = y[i].d * GGML_FP16_TO_FP32(x[i].d);

        // 定义指向 x[i].ql 的指针
        const uint8_t * restrict q4 = x[i].ql;
        // 定义指向 x[i].qh 的指针
        const uint8_t * restrict qh = x[i].qh;
        // 定义指向 y[i].qs 的指针
        const int8_t  * restrict q8 = y[i].qs;

        // 创建包含 x[i].scales[0] 的 __m64 类型变量
        const __m64 scales_1 = _mm_set1_pi8(x[i].scales[0]);
        // 创建包含 x[i].scales[1] 的 __m64 类型变量
        const __m64 scales_2 = _mm_set1_pi8(x[i].scales[1]);
        // 创建包含 x[i].scales[2] 的 __m64 类型变量
        const __m64 scales_3 = _mm_set1_pi8(x[i].scales[2]);
        // 创建包含 x[i].scales[3] 的 __m64 类型变量
        const __m64 scales_4 = _mm_set1_pi8(x[i].scales[3]);

        // 初始化两个 __m128i 类型变量为零
        __m128i sumi_0 = _mm_setzero_si128();
        __m128i sumi_1 = _mm_setzero_si128();

        // 创建包含 scales_1 和 scales_2 的 __m128i 类型变量
        const __m128i scale_0 = _mm_set_epi64(scales_2, scales_1);
        // 创建包含 scales_3 和 scales_4 的 __m128i 类型变量
        const __m128i scale_1 = _mm_set_epi64(scales_4, scales_3);

        // 从内存中加载 q4 的数据到 __m256i 类型变量 q4bits1
        const __m256i q4bits1 = _mm256_loadu_si256((const __m256i*)q4);
        // 从内存中加载 qh 的数据到 __m128i 类型变量 q4bitsH
        const __m128i q4bitsH = _mm_loadu_si128((const __m128i*)qh);
        # 将 q4bitsH 与 m2 进行按位与操作，然后将结果左移 4 位，得到 q4h_0
        const __m128i q4h_0 = _mm_slli_epi16(_mm_and_si128(q4bitsH, m2), 4);
        # 将 q4bitsH 右移 2 位，再与 m2 进行按位与操作，然后将结果左移 4 位，得到 q4h_1
        const __m128i q4h_1 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH, 2), m2), 4);
        # 类似地，得到 q4h_2 和 q4h_3
        const __m128i q4h_2 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH, 4), m2), 4);
        const __m128i q4h_3 = _mm_slli_epi16(_mm_and_si128(_mm_srli_epi16(q4bitsH, 6), m2), 4);

        # 从 q4bits1 中提取数据，与 m4 进行按位与操作，然后与对应的 q4h_x 进行按位或操作，得到 q4_x
        const __m128i q4_0 = _mm_or_si128(_mm_and_si128(_mm256_extractf128_si256(q4bits1, 0), m4), q4h_0);
        const __m128i q4_1 = _mm_or_si128(_mm_and_si128(_mm256_extractf128_si256(q4bits1, 1), m4), q4h_1);
        const __m128i q4_2 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(_mm256_extractf128_si256(q4bits1, 0), 4), m4), q4h_2);
        const __m128i q4_3 = _mm_or_si128(_mm_and_si128(_mm_srli_epi16(_mm256_extractf128_si256(q4bits1, 1), 4), m4), q4h_3);

        # 从内存中加载 q8 的数据到 __m256i 类型的变量 q8_0 和 q8_1
        const __m256i q8_0 = _mm256_loadu_si256((const __m256i*)(q8+ 0));
        const __m256i q8_1 = _mm256_loadu_si256((const __m256i*)(q8+32));

        # 将 m32s 与 q8_0 和 q8_1 进行逐位相乘并相加，得到 q8s_x
        __m128i q8s_0 = _mm_maddubs_epi16(m32s, _mm256_extractf128_si256(q8_0, 0));
        __m128i q8s_1 = _mm_maddubs_epi16(m32s, _mm256_extractf128_si256(q8_0, 1));
        __m128i q8s_2 = _mm_maddubs_epi16(m32s, _mm256_extractf128_si256(q8_1, 0));
        __m128i q8s_3 = _mm_maddubs_epi16(m32s, _mm256_extractf128_si256(q8_1, 1));

        # 将 q4_x 与对应的 q8_x 进行逐位相乘并相加，得到 p16_x
        __m128i p16_0 = _mm_maddubs_epi16(q4_0, _mm256_extractf128_si256(q8_0, 0));
        __m128i p16_1 = _mm_maddubs_epi16(q4_1, _mm256_extractf128_si256(q8_0, 1));
# 使用SSE指令集进行SIMD（单指令多数据）运算，对两个128位寄存器中的无符号字节进行有符号16位整数乘法并相加
__m128i p16_2 = _mm_maddubs_epi16(q4_2, _mm256_extractf128_si256(q8_1, 0));
__m128i p16_3 = _mm_maddubs_epi16(q4_3, _mm256_extractf128_si256(q8_1, 1));

# 对两个128位寄存器中的有符号16位整数进行减法
p16_0 = _mm_sub_epi16(p16_0, q8s_0);
p16_1 = _mm_sub_epi16(p16_1, q8s_1);
p16_2 = _mm_sub_epi16(p16_2, q8s_2);
p16_3 = _mm_sub_epi16(p16_3, q8s_3);

# 将有符号16位整数乘以有符号16位整数并相加
p16_0 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_0), p16_0);
p16_1 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_0, scale_0)), p16_1);
p16_2 = _mm_madd_epi16(_mm_cvtepi8_epi16(scale_1), p16_2);
p16_3 = _mm_madd_epi16(_mm_cvtepi8_epi16(_mm_unpackhi_epi64(scale_1, scale_1)), p16_3);

# 对两个32位整数寄存器进行相加
sumi_0 = _mm_add_epi32(sumi_0, _mm_add_epi32(p16_0, p16_2));
sumi_1 = _mm_add_epi32(sumi_1, _mm_add_epi32(p16_1, p16_3));

# 使用AVX指令集进行SIMD运算，将32位整数寄存器转换为单精度浮点数寄存器，并进行乘法和加法
acc = _mm256_add_ps(_mm256_mul_ps(_mm256_broadcast_ss(&d), _mm256_cvtepi32_ps(MM256_SET_M128I(sumi_1, sumi_0))), acc);

# 将累加器中的8个单精度浮点数求和，并将结果存储到指定的内存位置
*s = hsum_float_8(acc);
# 如果定义了 __riscv_v_intrinsic，则执行以下代码
float sumf = 0;  # 初始化浮点数 sumf 为 0

for (int i = 0; i < nb; ++i) {  # 循环遍历 nb 次
    const float d_all = (float)x[i].d;  # 将 x[i].d 转换为浮点数并赋值给 d_all

    const uint8_t * restrict q6 = x[i].ql;  # 将 x[i].ql 赋值给指针 q6
    const uint8_t * restrict qh = x[i].qh;  # 将 x[i].qh 赋值给指针 qh
    const int8_t  * restrict q8 = y[i].qs;  # 将 y[i].qs 赋值给指针 q8

    const int8_t * restrict scale = x[i].scales;  # 将 x[i].scales 赋值给指针 scale

    int32_t isum = 0;  # 初始化整数 isum 为 0

    size_t vl = 16;  # 初始化大小为 16 的 vl

    vint32m1_t vzero = __riscv_vmv_v_x_i32m1(0, 1);  # 使用 RISC-V 指令创建一个大小为 32 位的向量 vzero，并将其初始化为 0
// 读取 Q6 寄存器的值
vuint8mf2_t q6_0 = __riscv_vle8_v_u8mf2(q6, vl);
vuint8mf2_t q6_1 = __riscv_vle8_v_u8mf2(q6+16, vl);

// 读取 qh 寄存器的值
vuint8mf2_t qh_x = __riscv_vle8_v_u8mf2(qh, vl);

// 对 qh 寄存器的值进行位运算和移位操作，得到 qh0、qh1、qh2、qh3 四个值
vuint8mf2_t qh0 = __riscv_vsll_vx_u8mf2(__riscv_vand_vx_u8mf2(qh_x, 0x3, vl), 0x4, vl);
qh_x = __riscv_vsrl_vx_u8mf2(qh_x, 0x2, vl);
vuint8mf2_t qh1 = __riscv_vsll_vx_u8mf2(__riscv_vand_vx_u8mf2(qh_x, 0x3, vl), 0x4, vl);
qh_x = __riscv_vsrl_vx_u8mf2(qh_x, 0x2, vl);
vuint8mf2_t qh2 = __riscv_vsll_vx_u8mf2(__riscv_vand_vx_u8mf2(qh_x, 0x3, vl), 0x4, vl);
qh_x = __riscv_vsrl_vx_u8mf2(qh_x, 0x2, vl);
vuint8mf2_t qh3 = __riscv_vsll_vx_u8mf2(__riscv_vand_vx_u8mf2(qh_x, 0x3, vl), 0x4, vl);

// 对 Q6 寄存器和 qh 寄存器的值进行位运算和或运算，得到 q6h_0、q6h_1、q6h_2、q6h_3 四个值
vuint8mf2_t q6h_0 = __riscv_vor_vv_u8mf2(__riscv_vand_vx_u8mf2(q6_0, 0xF, vl), qh0, vl);
vuint8mf2_t q6h_1 = __riscv_vor_vv_u8mf2(__riscv_vand_vx_u8mf2(q6_1, 0xF, vl), qh1, vl);
vuint8mf2_t q6h_2 = __riscv_vor_vv_u8mf2(__riscv_vsrl_vx_u8mf2(q6_0, 0x4, vl), qh2, vl);
vuint8mf2_t q6h_3 = __riscv_vor_vv_u8mf2(__riscv_vsrl_vx_u8mf2(q6_1, 0x4, vl), qh3, vl);
// 从q6h_0中减去32，结果存储在q6v_0中
vint8mf2_t q6v_0 = __riscv_vsub_vx_i8mf2(__riscv_vreinterpret_v_u8mf2_i8mf2(q6h_0), 32, vl);
// 从q6h_1中减去32，结果存储在q6v_1中
vint8mf2_t q6v_1 = __riscv_vsub_vx_i8mf2(__riscv_vreinterpret_v_u8mf2_i8mf2(q6h_1), 32, vl);
// 从q6h_2中减去32，结果存储在q6v_2中
vint8mf2_t q6v_2 = __riscv_vsub_vx_i8mf2(__riscv_vreinterpret_v_u8mf2_i8mf2(q6h_2), 32, vl);
// 从q6h_3中减去32，结果存储在q6v_3中
vint8mf2_t q6v_3 = __riscv_vsub_vx_i8mf2(__riscv_vreinterpret_v_u8mf2_i8mf2(q6h_3), 32, vl);

// 从q8中加载16位整数，与q6v_0相乘，结果存储在p0中
vint16m1_t p0 = __riscv_vwmul_vv_i16m1(q6v_0, __riscv_vle8_v_i8mf2(q8, vl), vl);
// 从q8+16中加载16位整数，与q6v_1相乘，结果存储在p1中
vint16m1_t p1 = __riscv_vwmul_vv_i16m1(q6v_1, __riscv_vle8_v_i8mf2(q8+16, vl), vl);
// 从q8+32中加载16位整数，与q6v_2相乘，结果存储在p2中
vint16m1_t p2 = __riscv_vwmul_vv_i16m1(q6v_2, __riscv_vle8_v_i8mf2(q8+32, vl), vl);
// 从q8+48中加载16位整数，与q6v_3相乘，结果存储在p3中
vint16m1_t p3 = __riscv_vwmul_vv_i16m1(q6v_3, __riscv_vle8_v_i8mf2(q8+48, vl), vl);

// 对p0中的元素进行累加求和，结果存储在vs_0中
vint32m1_t vs_0 = __riscv_vwredsum_vs_i16m1_i32m1(p0, vzero, vl);
// 对p1中的元素进行累加求和，结果存储在vs_1中
vint32m1_t vs_1 = __riscv_vwredsum_vs_i16m1_i32m1(p1, vzero, vl);
// 对p2中的元素进行累加求和，结果存储在vs_2中
vint32m1_t vs_2 = __riscv_vwredsum_vs_i16m1_i32m1(p2, vzero, vl);
// 对p3中的元素进行累加求和，结果存储在vs_3中
vint32m1_t vs_3 = __riscv_vwredsum_vs_i16m1_i32m1(p3, vzero, vl);

// 将vs_0乘以scale[0]的值加到isum中
isum += __riscv_vmv_x_s_i32m1_i32(vs_0) * scale[0];
// 将vs_1乘以scale[1]的值加到isum中
isum += __riscv_vmv_x_s_i32m1_i32(vs_1) * scale[1];
// 将vs_2乘以scale[2]的值加到isum中
isum += __riscv_vmv_x_s_i32m1_i32(vs_2) * scale[2];
        # 将 vs_3 中的值乘以 scale[3]，然后加到 isum 上
        isum += __riscv_vmv_x_s_i32m1_i32(vs_3) * scale[3];

        # 将 isum 乘以 d_all 和 y[i].d 的乘积加到 sumf 上
        sumf += isum * d_all * y[i].d;

    }

    *s = sumf;

#else

    # 定义一些辅助数组和变量
    int8_t  aux8[QK_K];
    int16_t aux16[8];
    float   sums [8];
    int32_t aux32[8];
    # 将 sums 数组清零
    memset(sums, 0, 8*sizeof(float));

    # 初始化 sumf 为 0
    float sumf = 0;
    # 遍历 nb 次循环
    for (int i = 0; i < nb; ++i) {
        # 定义指向 x[i].ql 和 x[i].qh 的指针
        const uint8_t * restrict q4 = x[i].ql;
        const uint8_t * restrict qh = x[i].qh;
        // 声明一个指向 y[i].qs 的常量 int8_t 指针 q8
        const int8_t * restrict q8 = y[i].qs;
        // 将 aux32 数组清零，长度为 8 个 int32_t 类型的元素
        memset(aux32, 0, 8*sizeof(int32_t));
        // 声明一个指向 aux8 的 int8_t 指针 a
        int8_t * restrict a = aux8;
        // 循环遍历 16 次
        for (int l = 0; l < 16; ++l) {
            // 根据一定规则将数据赋值给 a 数组
            a[l+ 0] = (int8_t)((q4[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32;
            a[l+16] = (int8_t)((q4[l+16] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32;
            a[l+32] = (int8_t)((q4[l+ 0] >>  4) | (((qh[l] >> 4) & 3) << 4)) - 32;
            a[l+48] = (int8_t)((q4[l+16] >>  4) | (((qh[l] >> 6) & 3) << 4)) - 32;
        }
        // 初始化 is 为 0
        int is = 0;
        // 循环遍历 QK_K/16 次
        for (int j = 0; j < QK_K/16; ++j) {
            // 获取 x[i].scales[is] 的值赋给 scale
            int scale = x[i].scales[is++];
            // 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 将 scale 乘以 aux16 数组的值加到 aux32 数组中
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            // 移动 q8 和 a 指针的位置
            q8 += 8; a += 8;
            // 计算 aux16 数组的值
            for (int l = 0; l < 8; ++l) aux16[l] = q8[l] * a[l];
            // 将 scale 乘以 aux16 数组的值加到 aux32 数组中
            for (int l = 0; l < 8; ++l) aux32[l] += scale * aux16[l];
            // 移动 q8 和 a 指针的位置
            q8 += 8; a += 8;
        }
        // 计算 d 的值
        const float d = GGML_FP16_TO_FP32(x[i].d) * y[i].d;
    // 循环计算数组 sums 中每个元素的值
    for (int l = 0; l < 8; ++l) sums[l] += d * aux32[l];
    // 循环计算数组 sums 中所有元素的和
    }
    for (int l = 0; l < 8; ++l) sumf += sums[l];
    // 将计算得到的 sumf 赋值给指针 s 所指向的位置
    *s = sumf;
#endif
}

#endif
```