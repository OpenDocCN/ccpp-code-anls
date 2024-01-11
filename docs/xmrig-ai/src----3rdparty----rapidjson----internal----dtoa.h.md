# `xmrig\src\3rdparty\rapidjson\internal\dtoa.h`

```
// 定义了一个头文件保护宏，避免重复包含
#ifndef RAPIDJSON_DTOA_
#define RAPIDJSON_DTOA_

// 包含其他头文件
#include "itoa.h" // GetDigitsLut()
#include "diyfp.h"
#include "ieee754.h"

// 命名空间的开始
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

#ifdef __GNUC__
// 关闭特定的编译器警告
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(effc++)
RAPIDJSON_DIAG_OFF(array-bounds) // some gcc versions generate wrong warnings https://gcc.gnu.org/bugzilla/show_bug.cgi?id=59124
#endif

// 定义了一个函数，用于对数字进行舍入
inline void GrisuRound(char* buffer, int len, uint64_t delta, uint64_t rest, uint64_t ten_kappa, uint64_t wp_w) {
    while (rest < wp_w && delta - rest >= ten_kappa &&
           (rest + ten_kappa < wp_w ||  /// closer
            wp_w - rest > rest + ten_kappa - wp_w)) {
        buffer[len - 1]--;
        rest += ten_kappa;
    }
}

// 定义了一个函数，用于计算32位无符号整数的十进制位数
inline int CountDecimalDigit32(uint32_t n) {
    // 简单的纯C++实现在这种情况下比__builtin_clz版本更快
    if (n < 10) return 1;
    if (n < 100) return 2;
    if (n < 1000) return 3;
    if (n < 10000) return 4;
    if (n < 100000) return 5;
    if (n < 1000000) return 6;
    if (n < 10000000) return 7;
    # 如果 n 小于 100000000，则返回 8
    if (n < 100000000) return 8;
    # 在 DigitGen() 中不会达到 10 位数
    # 如果 n 小于 1000000000，则返回 9
    #return 10;
    # 返回 9
    return 9;
// 生成数字的函数，接受两个 DiyFp 对象和一个 delta 值作为参数，同时返回一个字符数组、一个长度和一个整数 K
inline void DigitGen(const DiyFp& W, const DiyFp& Mp, uint64_t delta, char* buffer, int* len, int* K) {
    // 保存 10 的幂的数组
    static const uint64_t kPow10[] = { 1U, 10U, 100U, 1000U, 10000U, 100000U, 1000000U, 10000000U, 100000000U,
                                       1000000000U, 10000000000U, 100000000000U,
                                       1000000000000U, 10000000000000U, 100000000000000U, 1000000000000000U,
                                       10000000000000000U, 100000000000000000U, 1000000000000000000U,
                                       10000000000000000000U };
    // 创建一个 DiyFp 对象，表示 1
    const DiyFp one(uint64_t(1) << -Mp.e, Mp.e);
    // 计算 Mp - W
    const DiyFp wp_w = Mp - W;
    // 将 Mp.f 右移 -one.e 位，得到 p1
    uint32_t p1 = static_cast<uint32_t>(Mp.f >> -one.e);
    // 将 Mp.f 与 (one.f - 1) 按位与，得到 p2
    uint64_t p2 = Mp.f & (one.f - 1);
    // 计算 p1 的十进制位数，保存在 kappa 中
    int kappa = CountDecimalDigit32(p1); // kappa in [0, 9]
    // 初始化长度为 0
    *len = 0;

    // 循环直到 kappa 为 0
    while (kappa > 0) {
        // 初始化 d 为 0
        uint32_t d = 0;
        // 根据 kappa 的值进行不同的处理
        switch (kappa) {
            // ...
        }
        // 如果 d 不为 0 或者长度不为 0，则将 d 转换为字符并保存到 buffer 中
        if (d || *len)
            buffer[(*len)++] = static_cast<char>('0' + static_cast<char>(d));
        // kappa 减一
        kappa--;
        // 计算 tmp 的值
        uint64_t tmp = (static_cast<uint64_t>(p1) << -one.e) + p2;
        // 如果 tmp 小于等于 delta，则更新 K 的值并调用 GrisuRound 函数，然后返回
        if (tmp <= delta) {
            *K += kappa;
            GrisuRound(buffer, *len, delta, tmp, kPow10[kappa] << -one.e, wp_w.f);
            return;
        }
    }

    // kappa = 0
}
    # 无限循环，直到条件不满足
    for (;;) {
        # p2 增大十倍
        p2 *= 10;
        # delta 增大十倍
        delta *= 10;
        # 将 p2 右移 -one.e 位，并转换为 char 类型
        char d = static_cast<char>(p2 >> -one.e);
        # 如果 d 不为 0 或者 len 指向的值不为 0
        if (d || *len)
            # 将 '0' + d 转换为 char 类型后存入 buffer 中，同时 len 指向的值加一
            buffer[(*len)++] = static_cast<char>('0' + d);
        # p2 与 one.f - 1 按位与
        p2 &= one.f - 1;
        # kappa 减一
        kappa--;
        # 如果 p2 小于 delta
        if (p2 < delta) {
            # K 指向的值加上 kappa
            *K += kappa;
            # 计算 index
            int index = -kappa;
            # 调用 GrisuRound 函数，然后返回
            GrisuRound(buffer, *len, delta, p2, one.f, wp_w.f * (index < 20 ? kPow10[index] : 0));
            return;
        }
    }
// 计算 Grisu2 算法的实现，用于将浮点数转换为字符串表示
inline void Grisu2(double value, char* buffer, int* length, int* K) {
    // 将浮点数转换为 DiyFp 类型
    const DiyFp v(value);
    // 计算浮点数的规范化边界
    DiyFp w_m, w_p;
    v.NormalizedBoundaries(&w_m, &w_p);

    // 获取缓存的幂次方
    const DiyFp c_mk = GetCachedPower(w_p.e, K);
    // 计算 W 值
    const DiyFp W = v.Normalize() * c_mk;
    DiyFp Wp = w_p * c_mk;
    DiyFp Wm = w_m * c_mk;
    Wm.f++;
    Wp.f--;
    // 生成数字
    DigitGen(W, Wp, Wp.f - Wm.f, buffer, length, K);
}

// 写指数
inline char* WriteExponent(int K, char* buffer) {
    if (K < 0) {
        *buffer++ = '-';
        K = -K;
    }

    if (K >= 100) {
        *buffer++ = static_cast<char>('0' + static_cast<char>(K / 100));
        K %= 100;
        const char* d = GetDigitsLut() + K * 2;
        *buffer++ = d[0];
        *buffer++ = d[1];
    }
    else if (K >= 10) {
        const char* d = GetDigitsLut() + K * 2;
        *buffer++ = d[0];
        *buffer++ = d[1];
    }
    else
        *buffer++ = static_cast<char>('0' + static_cast<char>(K));

    return buffer;
}

// 美化字符串表示
inline char* Prettify(char* buffer, int length, int k, int maxDecimalPlaces) {
    const int kk = length + k;  // 10^(kk-1) <= v < 10^kk

    if (0 <= k && kk <= 21) {
        // 1234e7 -> 12340000000
        for (int i = length; i < kk; i++)
            buffer[i] = '0';
        buffer[kk] = '.';
        buffer[kk + 1] = '0';
        return &buffer[kk + 2];
    }
    else if (0 < kk && kk <= 21) {
        // 1234e-2 -> 12.34
        std::memmove(&buffer[kk + 1], &buffer[kk], static_cast<size_t>(length - kk));
        buffer[kk] = '.';
        if (0 > k + maxDecimalPlaces) {
            // 当 maxDecimalPlaces = 2 时，1.2345 -> 1.23, 1.102 -> 1.1
            // 截断后移除额外的尾随零（至少一个）
            for (int i = kk + maxDecimalPlaces; i > kk + 1; i--)
                if (buffer[i] != '0')
                    return &buffer[i + 1];
            return &buffer[kk + 2]; // 保留一个零
        }
        else
            return &buffer[length + 1];
    }
    else if (-6 < kk && kk <= 0) {
        // 如果指数在-6到0之间，执行以下操作
        const int offset = 2 - kk;
        // 计算偏移量
        std::memmove(&buffer[offset], &buffer[0], static_cast<size_t>(length));
        // 将buffer中的内容向后移动offset个位置
        buffer[0] = '0';
        buffer[1] = '.';
        // 设置小数点位置
        for (int i = 2; i < offset; i++)
            buffer[i] = '0';
        // 补充0
        if (length - kk > maxDecimalPlaces) {
            // 当长度减去指数大于最大小数位数时，执行以下操作
            for (int i = maxDecimalPlaces + 1; i > 2; i--)
                if (buffer[i] != '0')
                    return &buffer[i + 1];
            // 移除截断后多余的尾随零（至少一个）
            return &buffer[3]; // 保留一个零
        }
        else
            return &buffer[length + offset];
    }
    else if (kk < -maxDecimalPlaces) {
        // 如果指数小于最大小数位数的负数，执行以下操作
        buffer[0] = '0';
        buffer[1] = '.';
        buffer[2] = '0';
        // 截断为零
        return &buffer[3];
    }
    else if (length == 1) {
        // 如果长度为1，执行以下操作
        buffer[1] = 'e';
        return WriteExponent(kk - 1, &buffer[2]);
    }
    else {
        // 其他情况，执行以下操作
        std::memmove(&buffer[2], &buffer[1], static_cast<size_t>(length - 1));
        // 将buffer中的内容向后移动1个位置
        buffer[1] = '.';
        buffer[length + 1] = 'e';
        // 设置小数点和指数
        return WriteExponent(kk - 1, &buffer[0 + length + 2]);
    }
} // 结束 namespace internal

RAPIDJSON_NAMESPACE_END // 结束 rapidjson 命名空间

#endif // 结束 RAPIDJSON_DTOA_
```