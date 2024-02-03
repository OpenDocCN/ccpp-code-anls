# `xmrig\src\3rdparty\rapidjson\internal\strtod.h`

```cpp
// 定义了一个宏，用于防止重复包含头文件
#ifndef RAPIDJSON_STRTOD_
#define RAPIDJSON_STRTOD_

// 包含必要的头文件
#include "ieee754.h"
#include "biginteger.h"
#include "diyfp.h"
#include "pow10.h"
#include <climits>
#include <limits>

// 命名空间的开始
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

// 定义了一个内联函数，用于快速计算浮点数的值
inline double FastPath(double significand, int exp) {
    if (exp < -308)
        return 0.0;
    else if (exp >= 0)
        return significand * internal::Pow10(exp);
    else
        return significand / internal::Pow10(-exp);
}

// 定义了一个内联函数，用于在正常精度下进行字符串转换为浮点数的计算
inline double StrtodNormalPrecision(double d, int p) {
    if (p < -308) {
        // 防止 expSum < -308，使得 Pow10(p) = 0
        d = FastPath(d, -308);
        d = FastPath(d, p + 308);
    }
    else
        d = FastPath(d, p);
    return d;
}

// 定义了一个内联函数，用于返回三个值中的最小值
template <typename T>
inline T Min3(T a, T b, T c) {
    T m = a;
    if (m > b) m = b;
    if (m > c) m = c;
    return m;
}

// 定义了一个内联函数，用于检查浮点数是否在半个最小单位精度范围内
inline int CheckWithinHalfULP(double b, const BigInteger& d, int dExp) {
    const Double db(b);
    const uint64_t bInt = db.IntegerSignificand();
    const int bExp = db.IntegerExponent();
    const int hExp = bExp - 1;

    int dS_Exp2 = 0, dS_Exp5 = 0, bS_Exp2 = 0, bS_Exp5 = 0, hS_Exp2 = 0, hS_Exp5 = 0;

    // 根据十进制指数进行调整
    if (dExp >= 0) {
        dS_Exp2 += dExp;
        dS_Exp5 += dExp;
    }

... (以下省略)
    else {
        // 如果指数差为负数，需要调整二进制和十进制的指数
        bS_Exp2 -= dExp;
        bS_Exp5 -= dExp;
        hS_Exp2 -= dExp;
        hS_Exp5 -= dExp;
    }

    // Adjust for binary exponent
    // 根据二进制指数进行调整
    if (bExp >= 0)
        bS_Exp2 += bExp;
    else {
        // 如果二进制指数为负数，需要调整十进制和十六进制的指数
        dS_Exp2 -= bExp;
        hS_Exp2 -= bExp;
    }

    // Adjust for half ulp exponent
    // 根据十六进制指数进行调整
    if (hExp >= 0)
        hS_Exp2 += hExp;
    else {
        // 如果十六进制指数为负数，需要调整十进制和二进制的指数
        dS_Exp2 -= hExp;
        bS_Exp2 -= hExp;
    }

    // Remove common power of two factor from all three scaled values
    // 从所有三个缩放值中移除公共的二次幂因子
    int common_Exp2 = Min3(dS_Exp2, bS_Exp2, hS_Exp2);
    dS_Exp2 -= common_Exp2;
    bS_Exp2 -= common_Exp2;
    hS_Exp2 -= common_Exp2;

    // 对十进制数进行缩放和乘以5的幂次方
    BigInteger dS = d;
    dS.MultiplyPow5(static_cast<unsigned>(dS_Exp5)) <<= static_cast<unsigned>(dS_Exp2);

    // 对二进制数进行缩放和乘以5的幂次方
    BigInteger bS(bInt);
    bS.MultiplyPow5(static_cast<unsigned>(bS_Exp5)) <<= static_cast<unsigned>(bS_Exp2);

    // 对十六进制数进行缩放和乘以5的幂次方
    BigInteger hS(1);
    hS.MultiplyPow5(static_cast<unsigned>(hS_Exp5)) <<= static_cast<unsigned>(hS_Exp2);

    // 计算差值
    BigInteger delta(0);
    dS.Difference(bS, &delta);

    // 返回差值与十六进制数的比较结果
    return delta.Compare(hS);
}

// 使用快速路径进行字符串到双精度浮点数的转换，如果可能的话
// 参考 http://www.exploringbinary.com/fast-path-decimal-to-floating-point-conversion/
inline bool StrtodFast(double d, int p, double* result) {
    if (p > 22  && p < 22 + 16) {
        // 如果指数在 22 到 22+16 之间，使用快速路径进行转换
        d *= internal::Pow10(p - 22);
        p = 22;
    }

    if (p >= -22 && p <= 22 && d <= 9007199254740991.0) { // 2^53 - 1
        // 如果指数在 -22 到 22 之间，且值小于等于 9007199254740991.0，使用快速路径进行转换
        *result = FastPath(d, p);
        return true;
    }
    else
        return false;
}

// 计算一个近似值，并检查是否在 1/2 ULP 范围内
template<typename Ch>
inline bool StrtodDiyFp(const Ch* decimals, int dLen, int dExp, double* result) {
    uint64_t significand = 0;
    int i = 0;   // 2^64 - 1 = 18446744073709551615, 1844674407370955161 = 0x1999999999999999
    for (; i < dLen; i++) {
        if (significand  >  RAPIDJSON_UINT64_C2(0x19999999, 0x99999999) ||
            (significand == RAPIDJSON_UINT64_C2(0x19999999, 0x99999999) && decimals[i] > Ch('5')))
            break;
        significand = significand * 10u + static_cast<unsigned>(decimals[i] - Ch('0'));
    }

    if (i < dLen && decimals[i] >= Ch('5')) // 四舍五入
        significand++;

    int remaining = dLen - i;
    const int kUlpShift = 3;
    const int kUlp = 1 << kUlpShift;
    int64_t error = (remaining == 0) ? 0 : kUlp / 2;

    DiyFp v(significand, 0);
    v = v.Normalize();
    error <<= -v.e;

    dExp += remaining;

    int actualExp;
    // 获取缓存的 10 的幂次方
    DiyFp cachedPower = GetCachedPower10(dExp, &actualExp);
    // 如果实际指数不等于期望指数
    if (actualExp != dExp) {
        // 定义10的幂的数组
        static const DiyFp kPow10[] = {
            DiyFp(RAPIDJSON_UINT64_C2(0xa0000000, 0x00000000), -60),  // 10^1
            DiyFp(RAPIDJSON_UINT64_C2(0xc8000000, 0x00000000), -57),  // 10^2
            DiyFp(RAPIDJSON_UINT64_C2(0xfa000000, 0x00000000), -54),  // 10^3
            DiyFp(RAPIDJSON_UINT64_C2(0x9c400000, 0x00000000), -50),  // 10^4
            DiyFp(RAPIDJSON_UINT64_C2(0xc3500000, 0x00000000), -47),  // 10^5
            DiyFp(RAPIDJSON_UINT64_C2(0xf4240000, 0x00000000), -44),  // 10^6
            DiyFp(RAPIDJSON_UINT64_C2(0x98968000, 0x00000000), -40)   // 10^7
        };
        // 计算调整值
        int adjustment = dExp - actualExp;
        // 断言调整值在1到7之间
        RAPIDJSON_ASSERT(adjustment >= 1 && adjustment < 8);
        // 乘以10的幂
        v = v * kPow10[adjustment - 1];
        // 如果小数部分长度加上调整值大于19，则错误加上半个最小单位
        if (dLen + adjustment > 19) // has more digits than decimal digits in 64-bit
            error += kUlp / 2;
    }

    // 乘以缓存的幂
    v = v * cachedPower;

    // 错误加上最小单位，并且如果错误为0，则加上1
    error += kUlp + (error == 0 ? 0 : 1);

    // 保存旧的指数
    const int oldExp = v.e;
    // 规范化
    v = v.Normalize();
    // 错误左移旧的指数减去新的指数
    error <<= oldExp - v.e;

    // 计算有效尾数大小
    const int effectiveSignificandSize = Double::EffectiveSignificandSize(64 + v.e);
    // 计算精度大小
    int precisionSize = 64 - effectiveSignificandSize;
    // 如果精度大小加上最小单位的位移大于等于64
    if (precisionSize + kUlpShift >= 64) {
        // 计算缩放指数
        int scaleExp = (precisionSize + kUlpShift) - 63;
        v.f >>= scaleExp;
        v.e += scaleExp;
        error = (error >> scaleExp) + 1 + kUlp;
        precisionSize -= scaleExp;
    }

    // 四舍五入
    DiyFp rounded(v.f >> precisionSize, v.e + precisionSize);
    const uint64_t precisionBits = (v.f & ((uint64_t(1) << precisionSize) - 1)) * kUlp;
    const uint64_t halfWay = (uint64_t(1) << (precisionSize - 1)) * kUlp;
    if (precisionBits >= halfWay + static_cast<unsigned>(error)) {
        rounded.f++;
        if (rounded.f & (DiyFp::kDpHiddenBit << 1)) { // rounding overflows mantissa (issue #340)
            rounded.f >>= 1;
            rounded.e++;
        }
    }

    // 将结果保存到指针指向的位置
    *result = rounded.ToDouble();
    # 返回一个布尔值，判断是否满足条件：halfWay - error >= precisionBits 或 precisionBits >= halfWay + error
    return halfWay - static_cast<unsigned>(error) >= precisionBits || precisionBits >= halfWay + static_cast<unsigned>(error);
}
// 模板函数，将字符串转换为大整数，返回双精度浮点数
template<typename Ch>
inline double StrtodBigInteger(double approx, const Ch* decimals, int dLen, int dExp) {
    // 断言，确保小数长度大于等于0
    RAPIDJSON_ASSERT(dLen >= 0);
    // 根据小数部分创建大整数对象
    const BigInteger dInt(decimals, static_cast<unsigned>(dLen));
    // 创建双精度浮点数对象
    Double a(approx);
    // 检查是否在半个最小单位内
    int cmp = CheckWithinHalfULP(a.Value(), dInt, dExp);
    if (cmp < 0)
        return a.Value();  // 在半个最小单位内
    else if (cmp == 0) {
        // 向偶数舍入
        if (a.Significand() & 1)
            return a.NextPositiveDouble();
        else
            return a.Value();
    }
    else // 调整
        return a.NextPositiveDouble();
}

// 模板函数，将字符串转换为完整精度的双精度浮点数
template<typename Ch>
inline double StrtodFullPrecision(double d, int p, const Ch* decimals, size_t length, size_t decimalPosition, int exp) {
    // 断言，确保d大于等于0
    RAPIDJSON_ASSERT(d >= 0.0);
    // 断言，确保长度大于等于1
    RAPIDJSON_ASSERT(length >= 1);

    double result = 0.0;
    // 如果可以快速转换，则直接返回结果
    if (StrtodFast(d, p, &result))
        return result;

    // 断言，确保长度不超过INT_MAX
    RAPIDJSON_ASSERT(length <= INT_MAX);
    int dLen = static_cast<int>(length);

    // 断言，确保长度大于等于小数点位置
    RAPIDJSON_ASSERT(length >= decimalPosition);
    // 断言，确保长度减去小数点位置不超过INT_MAX
    RAPIDJSON_ASSERT(length - decimalPosition <= INT_MAX);
    int dExpAdjust = static_cast<int>(length - decimalPosition);

    // 断言，确保exp减去dExpAdjust不小于INT_MIN
    RAPIDJSON_ASSERT(exp >= INT_MIN + dExpAdjust);
    int dExp = exp - dExpAdjust;

    // 确保长度+dExp不会溢出
    RAPIDJSON_ASSERT(dExp <= INT_MAX - dLen);

    // 去除前导零
    while (dLen > 0 && *decimals == '0') {
        dLen--;
        decimals++;
    }

    // 去除尾随零
    while (dLen > 0 && decimals[dLen - 1] == '0') {
        dLen--;
        dExp++;
    }

    if (dLen == 0) { // 缓冲区只包含零
        return 0.0;
    }

    // 去除最右边的数字
    const int kMaxDecimalDigit = 767 + 1;
    if (dLen > kMaxDecimalDigit) {
        dExp += dLen - kMaxDecimalDigit;
        dLen = kMaxDecimalDigit;
    }

    // 如果太小，下溢为零
    // 任何x <= 10^-324都被解释为零
    if (dLen + dExp <= -324)
        return 0.0;
}
    // 如果太大，溢出为无穷大。
    // 任何 x >= 10^309 都被解释为 +无穷大。
    if (dLen + dExp > 309)
        // 返回 C++ 标准库中 double 类型的正无穷大
        return std::numeric_limits<double>::infinity();

    // 调用 StrtodDiyFp 函数，进行字符串转换为 double 类型的操作
    if (StrtodDiyFp(decimals, dLen, dExp, &result))
        // 返回转换后的结果
        return result;

    // 使用 StrtodDiyFp 的近似值，并通过 BigInteger 比较进行调整
    return StrtodBigInteger(result, decimals, dLen, dExp);
}

} // namespace internal
RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_STRTOD_
```