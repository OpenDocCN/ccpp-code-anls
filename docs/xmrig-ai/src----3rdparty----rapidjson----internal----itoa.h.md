# `xmrig\src\3rdparty\rapidjson\internal\itoa.h`

```
// 定义了一个宏，用于防止头文件被重复包含
#ifndef RAPIDJSON_ITOA_

// 开始定义命名空间 RAPIDJSON_NAMESPACE_BEGIN
#define RAPIDJSON_ITOA_

// 包含 rapidjson.h 头文件
#include "../rapidjson.h"

// 命名空间开始
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

// 定义了一个内联函数，返回一个包含数字字符的静态常量数组
inline const char* GetDigitsLut() {
    // 静态常量数组，包含了数字字符
    static const char cDigitsLut[200] = {
        // 数字字符的映射表
        '0','0','0','1','0','2','0','3','0','4','0','5','0','6','0','7','0','8','0','9',
        '1','0','1','1','1','2','1','3','1','4','1','5','1','6','1','7','1','8','1','9',
        '2','0','2','1','2','2','2','3','2','4','2','5','2','6','2','7','2','8','2','9',
        '3','0','3','1','3','2','3','3','3','4','3','5','3','6','3','7','3','8','3','9',
        '4','0','4','1','4','2','4','3','4','4','4','5','4','6','4','7','4','8','4','9',
        '5','0','5','1','5','2','5','3','5','4','5','5','5','6','5','7','5','8','5','9',
        '6','0','6','1','6','2','6','3','6','4','6','5','6','6','6','7','6','8','6','9',
        '7','0','7','1','7','2','7','3','7','4','7','5','7','6','7','7','7','8','7','9',
        '8','0','8','1','8','2','8','3','8','4','8','5','8','6','8','7','8','8','8','9',
        '9','0','9','1','9','2','9','3','9','4','9','5','9','6','9','7','9','8','9','9'
    };
    // 返回数字字符的映射表
    return cDigitsLut;
}

// 定义了一个内联函数，将无符号 32 位整数转换为字符串
inline char* u32toa(uint32_t value, char* buffer) {
    // 断言，确保 buffer 不为空
    RAPIDJSON_ASSERT(buffer != 0);

    // 获取数字字符的映射表
    const char* cDigitsLut = GetDigitsLut();
    # 如果数值小于10000
    if (value < 10000) {
        # 将数值除以100并左移1位，得到d1
        const uint32_t d1 = (value / 100) << 1;
        # 将数值取模100并左移1位，得到d2
        const uint32_t d2 = (value % 100) << 1;

        # 如果数值大于等于1000，将cDigitsLut[d1]写入buffer
        if (value >= 1000)
            *buffer++ = cDigitsLut[d1];
        # 如果数值大于等于100，将cDigitsLut[d1 + 1]写入buffer
        if (value >= 100)
            *buffer++ = cDigitsLut[d1 + 1];
        # 如果数值大于等于10，将cDigitsLut[d2]写入buffer
        if (value >= 10)
            *buffer++ = cDigitsLut[d2];
        # 将cDigitsLut[d2 + 1]写入buffer
        *buffer++ = cDigitsLut[d2 + 1];
    }
    # 如果数值大于等于10000且小于100000000
    else if (value < 100000000) {
        # 将数值除以10000得到b，取模10000得到c
        const uint32_t b = value / 10000;
        const uint32_t c = value % 10000;

        # 将b除以100并左移1位，得到d1
        const uint32_t d1 = (b / 100) << 1;
        # 将b取模100并左移1位，得到d2
        const uint32_t d2 = (b % 100) << 1;

        # 将c除以100并左移1位，得到d3
        const uint32_t d3 = (c / 100) << 1;
        # 将c取模100并左移1位，得到d4
        const uint32_t d4 = (c % 100) << 1;

        # 如果数值大于等于10000000，将cDigitsLut[d1]写入buffer
        if (value >= 10000000)
            *buffer++ = cDigitsLut[d1];
        # 如果数值大于等于1000000，将cDigitsLut[d1 + 1]写入buffer
        if (value >= 1000000)
            *buffer++ = cDigitsLut[d1 + 1];
        # 如果数值大于等于100000，将cDigitsLut[d2]写入buffer
        if (value >= 100000)
            *buffer++ = cDigitsLut[d2];
        # 将cDigitsLut[d2 + 1]写入buffer
        *buffer++ = cDigitsLut[d2 + 1];

        # 将cDigitsLut[d3]写入buffer
        *buffer++ = cDigitsLut[d3];
        # 将cDigitsLut[d3 + 1]写入buffer
        *buffer++ = cDigitsLut[d3 + 1];
        # 将cDigitsLut[d4]写入buffer
        *buffer++ = cDigitsLut[d4];
        # 将cDigitsLut[d4 + 1]写入buffer
        *buffer++ = cDigitsLut[d4 + 1];
    }
    else {
        // value = aabbbbcccc in decimal

        // 将 value 除以 100000000，得到商 a 和余数 value
        const uint32_t a = value / 100000000; // 1 to 42
        value %= 100000000;

        // 如果 a 大于等于 10，则将 a 左移一位，然后将对应的字符添加到 buffer 中
        if (a >= 10) {
            const unsigned i = a << 1;
            *buffer++ = cDigitsLut[i];
            *buffer++ = cDigitsLut[i + 1];
        }
        // 否则，将 a 转换为字符添加到 buffer 中
        else
            *buffer++ = static_cast<char>('0' + static_cast<char>(a));

        // 将 value 除以 10000，得到商 b 和余数 c
        const uint32_t b = value / 10000; // 0 to 9999
        const uint32_t c = value % 10000; // 0 to 9999

        // 计算 b 的百位和十位对应的字符索引，然后将对应的字符添加到 buffer 中
        const uint32_t d1 = (b / 100) << 1;
        const uint32_t d2 = (b % 100) << 1;

        // 计算 c 的百位和十位对应的字符索引，然后将对应的字符添加到 buffer 中
        const uint32_t d3 = (c / 100) << 1;
        const uint32_t d4 = (c % 100) << 1;

        *buffer++ = cDigitsLut[d1];
        *buffer++ = cDigitsLut[d1 + 1];
        *buffer++ = cDigitsLut[d2];
        *buffer++ = cDigitsLut[d2 + 1];
        *buffer++ = cDigitsLut[d3];
        *buffer++ = cDigitsLut[d3 + 1];
        *buffer++ = cDigitsLut[d4];
        *buffer++ = cDigitsLut[d4 + 1];
    }
    // 返回 buffer
    return buffer;
// 将 int32_t 类型的整数转换为字符串，并存储到指定的 buffer 中
inline char* i32toa(int32_t value, char* buffer) {
    // 断言 buffer 不为空
    RAPIDJSON_ASSERT(buffer != 0);
    // 将 value 转换为无符号整数
    uint32_t u = static_cast<uint32_t>(value);
    // 如果 value 小于 0，则在 buffer 中添加负号，并将 u 取反加一
    if (value < 0) {
        *buffer++ = '-';
        u = ~u + 1;
    }

    // 调用 u32toa 函数将 u 转换为字符串并存储到 buffer 中
    return u32toa(u, buffer);
}

// 将 uint64_t 类型的整数转换为字符串，并存储到指定的 buffer 中
inline char* u64toa(uint64_t value, char* buffer) {
    // 断言 buffer 不为空
    RAPIDJSON_ASSERT(buffer != 0);
    // 获取数字字符的查找表
    const char* cDigitsLut = GetDigitsLut();
    // 定义常量
    const uint64_t  kTen8 = 100000000;
    const uint64_t  kTen9 = kTen8 * 10;
    const uint64_t kTen10 = kTen8 * 100;
    const uint64_t kTen11 = kTen8 * 1000;
    const uint64_t kTen12 = kTen8 * 10000;
    const uint64_t kTen13 = kTen8 * 100000;
    const uint64_t kTen14 = kTen8 * 1000000;
    const uint64_t kTen15 = kTen8 * 10000000;
    const uint64_t kTen16 = kTen8 * kTen8;
    # 如果数值小于 100000000，执行以下操作
    if (value < kTen8) {
        # 将 value 转换为无符号 32 位整数
        uint32_t v = static_cast<uint32_t>(value);
        # 如果 v 小于 10000，执行以下操作
        if (v < 10000) {
            # 计算百位和十位数字在 cDigitsLut 中的索引
            const uint32_t d1 = (v / 100) << 1;
            const uint32_t d2 = (v % 100) << 1;

            # 根据 v 的值判断是否需要将对应的数字添加到 buffer 中
            if (v >= 1000)
                *buffer++ = cDigitsLut[d1];
            if (v >= 100)
                *buffer++ = cDigitsLut[d1 + 1];
            if (v >= 10)
                *buffer++ = cDigitsLut[d2];
            *buffer++ = cDigitsLut[d2 + 1];
        }
        # 如果 v 大于等于 10000，执行以下操作
        else {
            # 将 v 拆分为 b 和 c
            const uint32_t b = v / 10000;
            const uint32_t c = v % 10000;

            # 计算 b 的百位和十位数字在 cDigitsLut 中的索引
            const uint32_t d1 = (b / 100) << 1;
            const uint32_t d2 = (b % 100) << 1;

            # 计算 c 的百位和十位数字在 cDigitsLut 中的索引
            const uint32_t d3 = (c / 100) << 1;
            const uint32_t d4 = (c % 100) << 1;

            # 根据 value 的值判断是否需要将对应的数字添加到 buffer 中
            if (value >= 10000000)
                *buffer++ = cDigitsLut[d1];
            if (value >= 1000000)
                *buffer++ = cDigitsLut[d1 + 1];
            if (value >= 100000)
                *buffer++ = cDigitsLut[d2];
            *buffer++ = cDigitsLut[d2 + 1];

            # 将 c 的百位和十位数字添加到 buffer 中
            *buffer++ = cDigitsLut[d3];
            *buffer++ = cDigitsLut[d3 + 1];
            *buffer++ = cDigitsLut[d4];
            *buffer++ = cDigitsLut[d4 + 1];
        }
    }
    else if (value < kTen16) {
        // 将 value 分解为两个 uint32_t 类型的数 v0 和 v1
        const uint32_t v0 = static_cast<uint32_t>(value / kTen8);
        const uint32_t v1 = static_cast<uint32_t>(value % kTen8);

        // 分别计算 v0 的高四位和低四位
        const uint32_t b0 = v0 / 10000;
        const uint32_t c0 = v0 % 10000;

        // 计算 d1 和 d2
        const uint32_t d1 = (b0 / 100) << 1;
        const uint32_t d2 = (b0 % 100) << 1;

        // 计算 d3 和 d4
        const uint32_t d3 = (c0 / 100) << 1;
        const uint32_t d4 = (c0 % 100) << 1;

        // 分别计算 v1 的高四位和低四位
        const uint32_t b1 = v1 / 10000;
        const uint32_t c1 = v1 % 10000;

        // 计算 d5 和 d6
        const uint32_t d5 = (b1 / 100) << 1;
        const uint32_t d6 = (b1 % 100) << 1;

        // 计算 d7 和 d8
        const uint32_t d7 = (c1 / 100) << 1;
        const uint32_t d8 = (c1 % 100) << 1;

        // 根据 value 的大小依次将对应的字符添加到 buffer 中
        if (value >= kTen15)
            *buffer++ = cDigitsLut[d1];
        if (value >= kTen14)
            *buffer++ = cDigitsLut[d1 + 1];
        if (value >= kTen13)
            *buffer++ = cDigitsLut[d2];
        if (value >= kTen12)
            *buffer++ = cDigitsLut[d2 + 1];
        if (value >= kTen11)
            *buffer++ = cDigitsLut[d3];
        if (value >= kTen10)
            *buffer++ = cDigitsLut[d3 + 1];
        if (value >= kTen9)
            *buffer++ = cDigitsLut[d4];

        // 将剩余的字符添加到 buffer 中
        *buffer++ = cDigitsLut[d4 + 1];
        *buffer++ = cDigitsLut[d5];
        *buffer++ = cDigitsLut[d5 + 1];
        *buffer++ = cDigitsLut[d6];
        *buffer++ = cDigitsLut[d6 + 1];
        *buffer++ = cDigitsLut[d7];
        *buffer++ = cDigitsLut[d7 + 1];
        *buffer++ = cDigitsLut[d8];
        *buffer++ = cDigitsLut[d8 + 1];
    }
    }

    // 返回 buffer
    return buffer;
} // 结束 i64toa 函数的定义

// 将 int64_t 类型的值转换为字符串，并存储在 buffer 中
inline char* i64toa(int64_t value, char* buffer) {
    // 断言 buffer 不为空
    RAPIDJSON_ASSERT(buffer != 0);
    // 将 value 转换为无符号整数
    uint64_t u = static_cast<uint64_t>(value);
    // 如果 value 小于 0
    if (value < 0) {
        // 在 buffer 中添加负号
        *buffer++ = '-';
        // 取反并加一，得到绝对值
        u = ~u + 1;
    }

    // 调用 u64toa 函数将无符号整数 u 转换为字符串，并存储在 buffer 中
    return u64toa(u, buffer);
}

} // 结束 namespace internal
RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_ITOA_
```