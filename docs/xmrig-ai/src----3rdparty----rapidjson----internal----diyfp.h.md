# `xmrig\src\3rdparty\rapidjson\internal\diyfp.h`

```cpp
// 包含版权声明和许可信息
// 定义了RapidJSON命名空间内部的DiyFp结构体
#ifndef RAPIDJSON_DIYFP_H_
#define RAPIDJSON_DIYFP_H_

// 包含rapidjson.h头文件和clzll.h头文件
#include "../rapidjson.h"
#include "clzll.h"
#include <limits>

// 如果是MSC编译器且是AMD64架构并且不是Intel编译器，则包含intrin.h头文件并使用intrinsic函数_umul128
#if defined(_MSC_VER) && defined(_M_AMD64) && !defined(__INTEL_COMPILER)
#include <intrin.h>
#pragma intrinsic(_umul128)
#endif

// 进入RapidJSON命名空间内部
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

// 关闭GCC编译器的effc++警告
#ifdef __GNUC__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(effc++)
#endif

// 关闭Clang编译器的padded警告
#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
#endif

// 定义DiyFp结构体
struct DiyFp {
    DiyFp() : f(), e() {} // 默认构造函数，f和e初始化为0

    DiyFp(uint64_t fp, int exp) : f(fp), e(exp) {} // 带参数的构造函数，初始化f和e

    explicit DiyFp(double d) { // 显式构造函数，接受一个double类型的参数
        union { // 使用联合体进行类型转换
            double d;
            uint64_t u64;
        } u = { d };

        int biased_e = static_cast<int>((u.u64 & kDpExponentMask) >> kDpSignificandSize); // 计算偏置指数
        uint64_t significand = (u.u64 & kDpSignificandMask); // 计算尾数
        if (biased_e != 0) { // 如果偏置指数不为0
            f = significand + kDpHiddenBit; // 计算尾数+隐藏位
            e = biased_e - kDpExponentBias; // 计算指数
        }
        else { // 如果偏置指数为0
            f = significand; // 尾数为原始尾数
            e = kDpMinExponent + 1; // 指数为最小指数+1
        }
    }
    # 重载减法运算符，返回两个 DiyFp 对象相减的结果
    DiyFp operator-(const DiyFp& rhs) const {
        # 返回一个新的 DiyFp 对象，其 f 值为当前对象的 f 值减去 rhs 对象的 f 值，e 值不变
        return DiyFp(f - rhs.f, e);
    }

    # 重载乘法运算符，返回两个 DiyFp 对象相乘的结果
    DiyFp operator*(const DiyFp& rhs) const {
#if defined(_MSC_VER) && defined(_M_AMD64)
        // 如果编译器是 MSVC 并且目标平台是 AMD64
        uint64_t h;
        uint64_t l = _umul128(f, rhs.f, &h);
        // 使用 _umul128 函数计算两个64位整数的乘积，结果存储在 h 和 l 中
        if (l & (uint64_t(1) << 63)) // rounding
            // 如果 l 的第63位为1，进行四舍五入
            h++;
        return DiyFp(h, e + rhs.e + 64);
#elif (__GNUC__ > 4 || (__GNUC__ == 4 && __GNUC_MINOR__ >= 6)) && defined(__x86_64__)
        // 如果编译器是 GCC 版本大于4.6，并且目标平台是 x86_64
        __extension__ typedef unsigned __int128 uint128;
        // 定义一个128位无符号整数类型 uint128
        uint128 p = static_cast<uint128>(f) * static_cast<uint128>(rhs.f);
        // 使用 static_cast 进行类型转换，计算两个128位整数的乘积
        uint64_t h = static_cast<uint64_t>(p >> 64);
        uint64_t l = static_cast<uint64_t>(p);
        // 将128位整数的乘积拆分成高64位和低64位
        if (l & (uint64_t(1) << 63)) // rounding
            // 如果 l 的第63位为1，进行四舍五入
            h++;
        return DiyFp(h, e + rhs.e + 64);
#else
        // 如果不满足上述条件
        const uint64_t M32 = 0xFFFFFFFF;
        const uint64_t a = f >> 32;
        const uint64_t b = f & M32;
        const uint64_t c = rhs.f >> 32;
        const uint64_t d = rhs.f & M32;
        // 将64位整数拆分成高32位和低32位
        const uint64_t ac = a * c;
        const uint64_t bc = b * c;
        const uint64_t ad = a * d;
        const uint64_t bd = b * d;
        // 计算乘积的中间结果
        uint64_t tmp = (bd >> 32) + (ad & M32) + (bc & M32);
        tmp += 1U << 31;  /// mult_round
        // 进行四舍五入
        return DiyFp(ac + (ad >> 32) + (bc >> 32) + (tmp >> 32), e + rhs.e + 64);
#endif
    }

    DiyFp Normalize() const {
        // 对当前 DiyFp 对象进行规范化处理
        int s = static_cast<int>(clzll(f));
        // 计算 f 的前导零位数
        return DiyFp(f << s, e - s);
    }

    DiyFp NormalizeBoundary() const {
        // 对当前 DiyFp 对象进行边界规范化处理
        DiyFp res = *this;
        while (!(res.f & (kDpHiddenBit << 1))) {
            res.f <<= 1;
            res.e--;
        }
        // 循环左移 f 直到 f 的最高位为1
        res.f <<= (kDiySignificandSize - kDpSignificandSize - 2);
        res.e = res.e - (kDiySignificandSize - kDpSignificandSize - 2);
        // 左移 f 使得其最高位为1，并更新 e 的值
        return res;
    }

    void NormalizedBoundaries(DiyFp* minus, DiyFp* plus) const {
        // 计算当前 DiyFp 对象的规范化边界值
        DiyFp pl = DiyFp((f << 1) + 1, e - 1).NormalizeBoundary();
        DiyFp mi = (f == kDpHiddenBit) ? DiyFp((f << 2) - 1, e - 2) : DiyFp((f << 1) - 1, e - 1);
        mi.f <<= mi.e - pl.e;
        mi.e = pl.e;
        *plus = pl;
        *minus = mi;
    }
    // 将当前对象转换为双精度浮点数
    double ToDouble() const {
        // 定义一个联合体，用于转换双精度浮点数和64位无符号整数
        union {
            double d;
            uint64_t u64;
        }u;
        // 断言浮点数小于等于隐藏位和尾数掩码之和
        RAPIDJSON_ASSERT(f <= kDpHiddenBit + kDpSignificandMask);
        // 如果指数小于规格化数的指数
        if (e < kDpDenormalExponent) {
            // 下溢，返回0.0
            return 0.0;
        }
        // 如果指数大于等于最大指数
        if (e >= kDpMaxExponent) {
            // 上溢，返回正无穷
            return std::numeric_limits<double>::infinity();
        }
        // 计算二进制表示的指数
        const uint64_t be = (e == kDpDenormalExponent && (f & kDpHiddenBit) == 0) ? 0 :
            static_cast<uint64_t>(e + kDpExponentBias);
        // 将尾数和指数组合成64位浮点数
        u.u64 = (f & kDpSignificandMask) | (be << kDpSignificandSize);
        // 返回双精度浮点数
        return u.d;
    }

    // 定义常量
    static const int kDiySignificandSize = 64;
    static const int kDpSignificandSize = 52;
    static const int kDpExponentBias = 0x3FF + kDpSignificandSize;
    static const int kDpMaxExponent = 0x7FF - kDpExponentBias;
    static const int kDpMinExponent = -kDpExponentBias;
    static const int kDpDenormalExponent = -kDpExponentBias + 1;
    static const uint64_t kDpExponentMask = RAPIDJSON_UINT64_C2(0x7FF00000, 0x00000000);
    static const uint64_t kDpSignificandMask = RAPIDJSON_UINT64_C2(0x000FFFFF, 0xFFFFFFFF);
    static const uint64_t kDpHiddenBit = RAPIDJSON_UINT64_C2(0x00100000, 0x00000000);

    // 定义成员变量
    uint64_t f;
    int e;
};

inline DiyFp GetCachedPowerByIndex(size_t index) {
    // 通过索引获取缓存的指数值，范围为10^-348, 10^-340, ..., 10^340
    };
    static const int16_t kCachedPowers_E[] = {
        // 缓存的指数值数组
        -1220, -1193, -1166, -1140, -1113, -1087, -1060, -1034, -1007,  -980,
        -954,  -927,  -901,  -874,  -847,  -821,  -794,  -768,  -741,  -715,
        -688,  -661,  -635,  -608,  -582,  -555,  -529,  -502,  -475,  -449,
        -422,  -396,  -369,  -343,  -316,  -289,  -263,  -236,  -210,  -183,
        -157,  -130,  -103,   -77,   -50,   -24,     3,    30,    56,    83,
        109,   136,   162,   189,   216,   242,   269,   295,   322,   348,
        375,   402,   428,   455,   481,   508,   534,   561,   588,   614,
        641,   667,   694,   720,   747,   774,   800,   827,   853,   880,
        907,   933,   960,   986,  1013,  1039,  1066
    };
    RAPIDJSON_ASSERT(index < 87);
    return DiyFp(kCachedPowers_F[index], kCachedPowers_E[index]);
}

inline DiyFp GetCachedPower(int e, int* K) {

    // 计算缓存的指数值
    double dk = (-61 - e) * 0.30102999566398114 + 347;  // dk 必须为正数，因此可以对正数进行向上取整
    int k = static_cast<int>(dk);
    if (dk - k > 0.0)
        k++;

    unsigned index = static_cast<unsigned>((k >> 3) + 1);
    *K = -(-348 + static_cast<int>(index << 3));    // 十进制指数不需要查找表
    // 返回缓存的指数值
    return GetCachedPowerByIndex(index);
}

inline DiyFp GetCachedPower10(int exp, int *outExp) {
    RAPIDJSON_ASSERT(exp >= -348);
    unsigned index = static_cast<unsigned>(exp + 348) / 8u;
    *outExp = -348 + static_cast<int>(index) * 8;
    return GetCachedPowerByIndex(index);
}

#ifdef __GNUC__
RAPIDJSON_DIAG_POP
#endif

#ifdef __clang__
RAPIDJSON_DIAG_POP
RAPIDJSON_DIAG_OFF(padded)
#endif

} // namespace internal
RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_DIYFP_H_
```