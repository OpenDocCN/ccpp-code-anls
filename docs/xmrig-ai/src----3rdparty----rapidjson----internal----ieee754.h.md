# `xmrig\src\3rdparty\rapidjson\internal\ieee754.h`

```
// 定义了一个宏，用于判断是否已经包含了 IEEE754 头文件
#ifndef RAPIDJSON_IEEE754_
#define RAPIDJSON_IEEE754_

// 包含 rapidjson.h 头文件
#include "../rapidjson.h"

// rapidjson 命名空间开始
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

// 定义了一个 Double 类
class Double {
public:
    // 默认构造函数
    Double() {}
    // 以 double 类型参数构造函数
    Double(double d) : d_(d) {}
    // 以 uint64_t 类型参数构造函数
    Double(uint64_t u) : u_(u) {}

    // 返回 double 类型值
    double Value() const { return d_; }
    // 返回 uint64_t 类型值
    uint64_t Uint64Value() const { return u_; }

    // 返回下一个正数的 double 类型值
    double NextPositiveDouble() const {
        RAPIDJSON_ASSERT(!Sign());
        return Double(u_ + 1).Value();
    }

    // 判断是否为负数
    bool Sign() const { return (u_ & kSignMask) != 0; }
    // 返回尾数
    uint64_t Significand() const { return u_ & kSignificandMask; }
    // 返回指数
    int Exponent() const { return static_cast<int>(((u_ & kExponentMask) >> kSignificandSize) - kExponentBias); }

    // 判断是否为 NaN
    bool IsNan() const { return (u_ & kExponentMask) == kExponentMask && Significand() != 0; }
    // 判断是否为无穷大
    bool IsInf() const { return (u_ & kExponentMask) == kExponentMask && Significand() == 0; }
    // 判断是否为 NaN 或无穷大
    bool IsNanOrInf() const { return (u_ & kExponentMask) == kExponentMask; }
    // 判断是否为正常数
    bool IsNormal() const { return (u_ & kExponentMask) != 0 || Significand() == 0; }
    // 判断是否为零
    bool IsZero() const { return (u_ & (kExponentMask | kSignificandMask)) == 0; }

    // 返回整数尾数
    uint64_t IntegerSignificand() const { return IsNormal() ? Significand() | kHiddenBit : Significand(); }
    # 返回浮点数的指数部分减去偏置值
    int IntegerExponent() const { return (IsNormal() ? Exponent() : kDenormalExponent) - kSignificandSize; }
    # 将浮点数转换为偏置值
    uint64_t ToBias() const { return (u_ & kSignMask) ? ~u_ + 1 : u_ | kSignMask; }

    # 计算有效尾数位数
    static int EffectiveSignificandSize(int order) {
        if (order >= -1021)
            return 53;
        else if (order <= -1074)
            return 0;
        else
            return order + 1074;
    }
private:
    // 双精度浮点数的尾数位数
    static const int kSignificandSize = 52;
    // 指数偏移值
    static const int kExponentBias = 0x3FF;
    // 非规格化数的指数值
    static const int kDenormalExponent = 1 - kExponentBias;
    // 符号位掩码
    static const uint64_t kSignMask = RAPIDJSON_UINT64_C2(0x80000000, 0x00000000);
    // 指数位掩码
    static const uint64_t kExponentMask = RAPIDJSON_UINT64_C2(0x7FF00000, 0x00000000);
    // 尾数位掩码
    static const uint64_t kSignificandMask = RAPIDJSON_UINT64_C2(0x000FFFFF, 0xFFFFFFFF);
    // 隐藏位
    static const uint64_t kHiddenBit = RAPIDJSON_UINT64_C2(0x00100000, 0x00000000);

    // 联合体，用于双精度浮点数和64位无符号整数的转换
    union {
        double d_;       // 双精度浮点数
        uint64_t u_;     // 64位无符号整数
    };
};

} // namespace internal
RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_IEEE754_
```