# `xmrig\src\3rdparty\rapidjson\internal\biginteger.h`

```cpp
// 定义了一个名为 BigInteger 的类，用于处理大整数
#ifndef RAPIDJSON_BIGINTEGER_H_
#define RAPIDJSON_BIGINTEGER_H_

#include "../rapidjson.h"

#if defined(_MSC_VER) && !defined(__INTEL_COMPILER) && defined(_M_AMD64)
#include <intrin.h> // for _umul128
#pragma intrinsic(_umul128)
#endif

RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

class BigInteger {
public:
    typedef uint64_t Type; // 定义了一个类型别名 Type，表示 uint64_t 类型的整数

    // 拷贝构造函数，用于初始化一个 BigInteger 对象
    BigInteger(const BigInteger& rhs) : count_(rhs.count_) {
        std::memcpy(digits_, rhs.digits_, count_ * sizeof(Type)); // 使用 memcpy 复制 rhs 对象的数据到当前对象
    }

    // 显式构造函数，用于将 uint64_t 类型的整数转换为 BigInteger 对象
    explicit BigInteger(uint64_t u) : count_(1) {
        digits_[0] = u; // 将 u 存储到当前对象的 digits_ 数组中
    }

    // 模板构造函数，用于将十进制字符串转换为 BigInteger 对象
    template<typename Ch>
    BigInteger(const Ch* decimals, size_t length) : count_(1) {
        RAPIDJSON_ASSERT(length > 0); // 断言 length 大于 0
        digits_[0] = 0; // 初始化当前对象的 digits_ 数组
        size_t i = 0;
        const size_t kMaxDigitPerIteration = 19;  // 2^64 = 18446744073709551616 > 10^19
        while (length >= kMaxDigitPerIteration) {
            AppendDecimal64(decimals + i, decimals + i + kMaxDigitPerIteration); // 将指定范围内的十进制字符串转换为 BigInteger 对象
            length -= kMaxDigitPerIteration; // 更新剩余长度
            i += kMaxDigitPerIteration; // 更新起始位置
        }

        if (length > 0)
            AppendDecimal64(decimals + i, decimals + i + length); // 将剩余的十进制字符串转换为 BigInteger 对象
    }

    // 赋值运算符重载，用于将一个 BigInteger 对象赋值给另一个 BigInteger 对象
    BigInteger& operator=(const BigInteger &rhs)
    // 复制赋值运算符重载，用于将一个 BigInteger 对象的值赋给另一个对象
    {
        // 检查是否自我赋值，如果不是则执行赋值操作
        if (this != &rhs) {
            count_ = rhs.count_;
            // 使用内存拷贝函数将 rhs 对象的数据拷贝到当前对象中
            std::memcpy(digits_, rhs.digits_, count_ * sizeof(Type));
        }
        // 返回当前对象的引用
        return *this;
    }

    // 将无符号 64 位整数赋值给 BigInteger 对象的重载运算符
    BigInteger& operator=(uint64_t u) {
        // 将无符号整数赋值给当前对象的第一个数字
        digits_[0] = u;
        count_ = 1;
        // 返回当前对象的引用
        return *this;
    }

    // 将无符号 64 位整数加到 BigInteger 对象上的重载运算符
    BigInteger& operator+=(uint64_t u) {
        // 备份当前对象的第一个数字
        Type backup = digits_[0];
        // 将无符号整数加到当前对象的第一个数字上
        digits_[0] += u;
        // 遍历对象的数字数组，处理进位
        for (size_t i = 0; i < count_ - 1; i++) {
            if (digits_[i] >= backup)
                return *this; // 没有进位
            backup = digits_[i + 1];
            digits_[i + 1] += 1;
        }

        // 处理最后的进位
        if (digits_[count_ - 1] < backup)
            PushBack(1);

        // 返回当前对象的引用
        return *this;
    }

    // 将无符号 64 位整数乘到 BigInteger 对象上的重载运算符
    BigInteger& operator*=(uint64_t u) {
        // 处理特殊情况
        if (u == 0) return *this = 0;
        if (u == 1) return *this;
        if (*this == 1) return *this = u;

        uint64_t k = 0;
        // 遍历对象的数字数组，执行乘法运算
        for (size_t i = 0; i < count_; i++) {
            uint64_t hi;
            digits_[i] = MulAdd64(digits_[i], u, k, &hi);
            k = hi;
        }

        // 处理最后的进位
        if (k > 0)
            PushBack(k);

        // 返回当前对象的引用
        return *this;
    }

    // 将无符号 32 位整数乘到 BigInteger 对象上的重载运算符
    BigInteger& operator*=(uint32_t u) {
        // 处理特殊情况
        if (u == 0) return *this = 0;
        if (u == 1) return *this;
        if (*this == 1) return *this = u;

        uint64_t k = 0;
        // 遍历对象的数字数组，执行乘法运算
        for (size_t i = 0; i < count_; i++) {
            const uint64_t c = digits_[i] >> 32;
            const uint64_t d = digits_[i] & 0xFFFFFFFF;
            const uint64_t uc = u * c;
            const uint64_t ud = u * d;
            const uint64_t p0 = ud + k;
            const uint64_t p1 = uc + (p0 >> 32);
            digits_[i] = (p0 & 0xFFFFFFFF) | (p1 << 32);
            k = p1 >> 32;
        }

        // 处理最后的进位
        if (k > 0)
            PushBack(k);

        // 返回当前对象的引用
        return *this;
    }
    # 左移操作符重载函数，将大整数左移指定位数
    BigInteger& operator<<=(size_t shift) {
        # 如果大整数为0或者左移位数为0，则直接返回
        if (IsZero() || shift == 0) return *this;

        # 计算左移的偏移量和位数
        size_t offset = shift / kTypeBit;
        size_t interShift = shift % kTypeBit;
        # 断言大整数的位数加上偏移量不超过容量
        RAPIDJSON_ASSERT(count_ + offset <= kCapacity);

        # 如果位移为整数倍的情况
        if (interShift == 0) {
            # 使用内存移动函数将数据整体左移
            std::memmove(digits_ + offset, digits_, count_ * sizeof(Type));
            count_ += offset;
        }
        # 如果位移不是整数倍
        else {
            # 在末尾添加一个0，为了后续位移操作
            digits_[count_] = 0;
            # 循环进行位移操作
            for (size_t i = count_; i > 0; i--)
                digits_[i + offset] = (digits_[i] << interShift) | (digits_[i - 1] >> (kTypeBit - interShift));
            digits_[offset] = digits_[0] << interShift;
            count_ += offset;
            # 如果最高位不为0，则位数加1
            if (digits_[count_])
                count_++;
        }

        # 将偏移量部分的数据置0
        std::memset(digits_, 0, offset * sizeof(Type));

        # 返回左移后的大整数
        return *this;
    }

    # 判断两个大整数是否相等
    bool operator==(const BigInteger& rhs) const {
        return count_ == rhs.count_ && std::memcmp(digits_, rhs.digits_, count_ * sizeof(Type)) == 0;
    }

    # 判断大整数是否与指定整数相等
    bool operator==(const Type rhs) const {
        return count_ == 1 && digits_[0] == rhs;
    }

    # 对大整数进行5的幂次方乘法
    BigInteger& MultiplyPow5(unsigned exp) {
        # 预定义5的幂次方数组
        static const uint32_t kPow5[12] = {
            5,
            5 * 5,
            5 * 5 * 5,
            5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5,
            5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5 * 5
        };
        # 如果指数为0，则直接返回
        if (exp == 0) return *this;
        # 对指数进行拆解，分别进行5的幂次方乘法
        for (; exp >= 27; exp -= 27) *this *= RAPIDJSON_UINT64_C2(0X6765C793, 0XFA10079D); // 5^27
        for (; exp >= 13; exp -= 13) *this *= static_cast<uint32_t>(1220703125u); // 5^13
        if (exp > 0)                 *this *= kPow5[exp - 1];
        # 返回5的幂次方乘法后的结果
        return *this;
    }
    // 计算该大整数与rhs的绝对差值。
    // 假设this != rhs
    bool Difference(const BigInteger& rhs, BigInteger* out) const {
        // 比较this和rhs的大小关系
        int cmp = Compare(rhs);
        RAPIDJSON_ASSERT(cmp != 0);
        const BigInteger *a, *b;  // 使a > b
        bool ret;
        // 如果this < rhs，则a指向rhs，b指向this，ret为true；否则a指向this，b指向rhs，ret为false
        if (cmp < 0) { a = &rhs; b = this; ret = true; }
        else         { a = this; b = &rhs; ret = false; }

        Type borrow = 0;
        // 遍历a的每一位
        for (size_t i = 0; i < a->count_; i++) {
            // 计算差值，并考虑上一位的借位
            Type d = a->digits_[i] - borrow;
            if (i < b->count_)
                d -= b->digits_[i];
            borrow = (d > a->digits_[i]) ? 1 : 0;
            out->digits_[i] = d;
            // 更新out的有效位数
            if (d != 0)
                out->count_ = i + 1;
        }

        return ret;
    }

    // 比较该大整数与rhs的大小关系
    int Compare(const BigInteger& rhs) const {
        // 如果位数不同，直接返回比较结果
        if (count_ != rhs.count_)
            return count_ < rhs.count_ ? -1 : 1;

        // 从高位到低位逐个比较
        for (size_t i = count_; i-- > 0;)
            if (digits_[i] != rhs.digits_[i])
                return digits_[i] < rhs.digits_[i] ? -1 : 1;

        return 0;
    }

    // 返回该大整数的位数
    size_t GetCount() const { return count_; }
    // 返回指定位置的数字
    Type GetDigit(size_t index) const { RAPIDJSON_ASSERT(index < count_); return digits_[index]; }
    // 判断该大整数是否为0
    bool IsZero() const { return count_ == 1 && digits_[0] == 0; }
    // 使用模板函数，将十进制数字追加到当前数字中
    template<typename Ch>
    void AppendDecimal64(const Ch* begin, const Ch* end) {
        // 将字符数组解析为无符号64位整数
        uint64_t u = ParseUint64(begin, end);
        // 如果当前数字为0，则将其替换为解析出的无符号整数
        if (IsZero())
            *this = u;
        // 否则，计算10的指数次幂乘以解析出的整数，并加到当前数字中
        else {
            unsigned exp = static_cast<unsigned>(end - begin);
            (MultiplyPow5(exp) <<= exp) += u;   // *this = *this * 10^exp + u
        }
    }

    // 将数字追加到当前数字的末尾
    void PushBack(Type digit) {
        RAPIDJSON_ASSERT(count_ < kCapacity);
        digits_[count_++] = digit;
    }

    // 使用模板函数，将字符数组解析为无符号64位整数
    template<typename Ch>
    static uint64_t ParseUint64(const Ch* begin, const Ch* end) {
        uint64_t r = 0;
        for (const Ch* p = begin; p != end; ++p) {
            RAPIDJSON_ASSERT(*p >= Ch('0') && *p <= Ch('9'));
            r = r * 10u + static_cast<unsigned>(*p - Ch('0'));
        }
        return r;
    }

    // 假设 a * b + k < 2^128
    // 计算 a * b + k，并将结果的高64位存储到 outHigh 中
    static uint64_t MulAdd64(uint64_t a, uint64_t b, uint64_t k, uint64_t* outHigh) {
        // 根据不同的编译器和架构，选择不同的计算方式
        #if defined(_MSC_VER) && defined(_M_AMD64)
            uint64_t low = _umul128(a, b, outHigh) + k;
            if (low < k)
                (*outHigh)++;
            return low;
        #elif (__GNUC__ > 4 || (__GNUC__ == 4 && __GNUC_MINOR__ >= 6)) && defined(__x86_64__)
            __extension__ typedef unsigned __int128 uint128;
            uint128 p = static_cast<uint128>(a) * static_cast<uint128>(b);
            p += k;
            *outHigh = static_cast<uint64_t>(p >> 64);
            return static_cast<uint64_t>(p);
        #else
            const uint64_t a0 = a & 0xFFFFFFFF, a1 = a >> 32, b0 = b & 0xFFFFFFFF, b1 = b >> 32;
            uint64_t x0 = a0 * b0, x1 = a0 * b1, x2 = a1 * b0, x3 = a1 * b1;
            x1 += (x0 >> 32); // can't give carry
            x1 += x2;
            if (x1 < x2)
                x3 += (static_cast<uint64_t>(1) << 32);
            uint64_t lo = (x1 << 32) + (x0 & 0xFFFFFFFF);
            uint64_t hi = x3 + (x1 >> 32);

            lo += k;
            if (lo < k)
                hi++;
            *outHigh = hi;
            return lo;
        #endif
    }

    // 定义常量 kBitCount，表示位数为3328，用于存储大整数
    static const size_t kBitCount = 3328;  // 64bit * 54 > 10^1000
    # 定义常量 kCapacity，表示数组容量，即位数除以 Type 类型的大小
    static const size_t kCapacity = kBitCount / sizeof(Type);
    # 定义常量 kTypeBit，表示 Type 类型的位数
    static const size_t kTypeBit = sizeof(Type) * 8;

    # 声明一个名为 digits_ 的数组，用于存储数字
    Type digits_[kCapacity];
    # 声明一个名为 count_ 的变量，用于存储数字的个数
    size_t count_;
}; // 结束 internal 命名空间

} // 结束命名空间 rapidjson

RAPIDJSON_NAMESPACE_END // 结束 rapidjson 命名空间

#endif // 结束 RAPIDJSON_BIGINTEGER_H_ 的条件编译
```