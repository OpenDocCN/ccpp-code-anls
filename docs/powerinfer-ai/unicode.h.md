# `PowerInfer\unicode.h`

```
// 防止头文件重复包含
#pragma once

// 包含必要的头文件
#include <cassert>
#include <stdexcept>
#include <vector>
#include <unordered_map>

// 定义数字范围的静态常量
static const std::vector<std::pair<uint32_t, uint32_t>> digit_ranges = {
    // 数字范围的起始和结束值
    // ...
};

// 定义字母范围的静态常量
static const std::vector<std::pair<uint32_t, uint32_t>> letter_ranges = {
    // 字母范围的起始和结束值
    // ...
};
# 以下是一系列十六进制数对，表示 Unicode 字符的范围
# 例如 {0x712, 0x72F} 表示 Unicode 字符范围从 0x712 到 0x72F
# 每个数对表示一个字符范围，用逗号分隔
# 这些数对可能用于定义字符集合或范围
这段代码是一系列的十六进制数对，每对数代表 Unicode 字符范围的起始和结束。
# 定义了一组十六进制范围，用于表示 Unicode 字符的范围
{0x115D8, 0x115DB}, {0x11600, 0x1162F}, {0x11644, 0x11644}, {0x11680, 0x116AA}, {0x116B8, 0x116B8}, {0x11700, 0x1171A}, {0x11800, 0x1182B}, {0x118A0, 0x118DF}, {0x118FF, 0x11906}, {0x11909, 0x11909},
{0x1190C, 0x11913}, {0x11915, 0x11916}, {0x11918, 0x1192F}, {0x1193F, 0x1193F}, {0x11941, 0x11941}, {0x119A0, 0x119A7}, {0x119AA, 0x119D0}, {0x119E1, 0x119E1}, {0x119E3, 0x119E3}, {0x11A00, 0x11A00},
{0x11A0B, 0x11A32}, {0x11A3A, 0x11A3A}, {0x11A50, 0x11A50}, {0x11A5C, 0x11A89}, {0x11A9D, 0x11A9D}, {0x11AC0, 0x11AF8}, {0x11C00, 0x11C08}, {0x11C0A, 0x11C2E}, {0x11C40, 0x11C40}, {0x11C72, 0x11C8F},
{0x11D00, 0x11D06}, {0x11D08, 0x11D09}, {0x11D0B, 0x11D30}, {0x11D46, 0x11D46}, {0x11D60, 0x11D65}, {0x11D67, 0x11D68}, {0x11D6A, 0x11D89}, {0x11D98, 0x11D98}, {0x11EE0, 0x11EF2}, {0x11FB0, 0x11FB0},
{0x12000, 0x12399}, {0x12480, 0x12543}, {0x13000, 0x1342E}, {0x14400, 0x14646}, {0x16800, 0x16A38}, {0x16A40, 0x16A5E}, {0x16AD0, 0x16AED}, {0x16B00, 0x16B2F}, {0x16B40, 0x16B43}, {0x16B63, 0x16B77},
{0x16B7D, 0x16B8F}, {0x16E40, 0x16E7F}, {0x16F00, 0x16F4A}, {0x16F50, 0x16F50}, {0x16F93, 0x16F9F}, {0x16FE0, 0x16FE1}, {0x16FE3, 0x16FE3}, {0x17000, 0x187F7}, {0x18800, 0x18CD5}, {0x18D00, 0x18D08},
{0x1B000, 0x1B11E}, {0x1B150, 0x1B152}, {0x1B164, 0x1B167}, {0x1B170, 0x1B2FB}, {0x1BC00, 0x1BC6A}, {0x1BC70, 0x1BC7C}, {0x1BC80, 0x1BC88}, {0x1BC90, 0x1BC99}, {0x1D400, 0x1D454}, {0x1D456, 0x1D49C},
{0x1D49E, 0x1D49F}, {0x1D4A2, 0x1D4A2}, {0x1D4A5, 0x1D4A6}, {0x1D4A9, 0x1D4AC}, {0x1D4AE, 0x1D4B9}, {0x1D4BB, 0x1D4BB}, {0x1D4BD, 0x1D4C3}, {0x1D4C5, 0x1D505}, {0x1D507, 0x1D50A}, {0x1D50D, 0x1D514},
{0x1D516, 0x1D51C}, {0x1D51E, 0x1D539}, {0x1D53B, 0x1D53E}, {0x1D540, 0x1D544}, {0x1D546, 0x1D546}, {0x1D54A, 0x1D550}, {0x1D552, 0x1D6A5}, {0x1D6A8, 0x1D6C0}, {0x1D6C2, 0x1D6DA}, {0x1D6DC, 0x1D6FA},
{0x1D6FC, 0x1D714}, {0x1D716, 0x1D734}, {0x1D736, 0x1D74E}, {0x1D750, 0x1D76E}, {0x1D770, 0x1D788}, {0x1D78A, 0x1D7A8}, {0x1D7AA, 0x1D7C2}, {0x1D7C4, 0x1D7CB}, {0x1E100, 0x1E12C}, {0x1E137, 0x1E13D},
{0x1E14E, 0x1E14E}, {0x1E2C0, 0x1E2EB}, {0x1E800, 0x1E8C4}, {0x1E900, 0x1E943}, {0x1E94B, 0x1E94B}, {0x1EE00, 0x1EE03}, {0x1EE05, 0x1EE1F}, {0x1EE21, 0x1EE22}, {0x1EE24, 0x1EE24}, {0x1EE27, 0x1EE27},
{0x1EE29, 0x1EE32}, {0x1EE34, 0x1EE37}, {0x1EE39, 0x1EE39}, {0x1EE3B, 0x1EE3B}, {0x1EE42, 0x1EE42}, {0x1EE47, 0x1EE47}, {0x1EE49, 0x1EE49}, {0x1EE4B, 0x1EE4B}, {0x1EE4D, 0x1EE4F}, {0x1EE51, 0x1EE52},
{0x1EE54, 0x1EE54}, {0x1EE57, 0x1EE57}, {0x1EE59, 0x1EE59}, {0x1EE5B, 0x1EE5B}, {0x1EE5D, 0x1EE5D}, {0x1EE5F, 0x1EE5F}, {0x1EE61, 0x1EE62}, {0x1EE64, 0x1EE64}, {0x1EE67, 0x1EE6A}, {0x1EE6C, 0x1EE72},
{0x1EE74, 0x1EE77}, {0x1EE79, 0x1EE7C}, {0x1EE7E, 0x1EE7E}, {0x1EE80, 0x1EE89}, {0x1EE8B, 0x1EE9B}, {0x1EEA1, 0x1EEA3}, {0x1EEA5, 0x1EEA9}, {0x1EEAB, 0x1EEBB}, {0x20000, 0x2A6DD}, {0x2A700, 0x2B734},
{0x2B740, 0x2B81D}, {0x2B820, 0x2CEA1}, {0x2CEB0, 0x2EBE0}, {0x2F800, 0x2FA1D}, {0x30000, 0x3134A},

# 定义了一组 Unicode 空白字符的范围
static const std::vector<std::pair<uint32_t, uint32_t>> whitespace_ranges = {
{0x9, 0xD}, {0x1C, 0x20}, {0x85, 0x85}, {0xA0, 0xA0}, {0x1680, 0x1680}, {0x2000, 0x200A}, {0x2028, 0x2029}, {0x202F, 0x202F}, {0x205F, 0x205F}, {0x3000, 0x3000},
};
// 定义一个静态常量，存储重音符号的 Unicode 范围
static const std::vector<std::pair<uint32_t, uint32_t>> accent_mark_ranges = {
    // 以 {开始，结束} 的形式存储每个重音符号的 Unicode 范围
    {0x300, 0x36F}, {0x483, 0x489}, {0x591, 0x5BD}, {0x5BF, 0x5BF}, {0x5C1, 0x5C2}, {0x5C4, 0x5C5}, {0x5C7, 0x5C7}, {0x610, 0x61A}, {0x64B, 0x65F}, {0x670, 0x670}, {0x6D6, 0x6DC}, {0x6DF, 0x6E4},
    // ... 还有很多范围
};
// 定义一组十六进制范围，每个范围包含两个值，表示起始和结束的十六进制数
{0x11145, 0x11146}, {0x11173, 0x11173}, {0x11180, 0x11182}, {0x111B3, 0x111C0}, {0x111C9, 0x111CC}, {0x111CE, 0x111CF}, {0x1122C, 0x11237}, {0x1123E, 0x1123E}, {0x112DF, 0x112EA}, {0x11300, 0x11303},
{0x1133B, 0x1133C}, {0x1133E, 0x11344}, {0x11347, 0x11348}, {0x1134B, 0x1134D}, {0x11357, 0x11357}, {0x11362, 0x11363}, {0x11366, 0x1136C}, {0x11370, 0x11374}, {0x11435, 0x11446}, {0x1145E, 0x1145E},
// ... （省略部分范围）
{0x3001, 0x3003}, {0x3008, 0x3011}, {0x3014, 0x301F},
```

```
// 定义一个静态的包含一组无符号32位整数对的向量，每对整数表示一个范围的起始和结束值
static const std::vector<std::pair<uint32_t, uint32_t>> punctuation_ranges = {
{0x21, 0x23}, {0x25, 0x2A}, {0x2C, 0x2F}, {0x3A, 0x3B}, {0x3F, 0x40}, {0x5B, 0x5D}, {0x5F, 0x5F}, {0x7B, 0x7B}, {0x7D, 0x7D}, {0xA1, 0xA1}, {0xA7, 0xA7}, {0xAB, 0xAB}, {0xB6, 0xB7}, {0xBB, 0xBB},
// ... （省略部分范围）
{0x2E00, 0x2E2E}, {0x2E30, 0x2E4F}, {0x2E52, 0x2E52}, {0x3001, 0x3003}, {0x3008, 0x3011}, {0x3014, 0x301F},
```

// 以上代码是一组用于表示标点符号范围的数据，每个范围由起始和结束的十六进制数表示。
// 定义一个静态的包含 Unicode 符号范围的向量
static const std::vector<std::pair<uint32_t, uint32_t>> symbol_ranges = {
    // 向量中包含了一系列的 Unicode 符号范围，每个范围由一对十六进制数表示
    // 每对数表示一个符号的起始和结束码点
    // 例如 {0x24, 0x24} 表示一个符号的码点范围
    // 该向量包含了多个这样的符号范围
    // ...
};
// 定义一个静态的控制字符范围数组
static const std::vector<std::pair<uint32_t, uint32_t>> control_ranges = {
    // 控制字符范围的起始和结束值
    {0x0, 0x8}, {0xE, 0x1B}, {0x7F, 0x84}, {0x86, 0x9F}, {0xAD, 0xAD}, {0x378, 0x379}, {0x380, 0x383}, {0x38B, 0x38B}, {0x38D, 0x38D}, {0x3A2, 0x3A2}, {0x530, 0x530}, {0x557, 0x558}, {0x58B, 0x58C},
    {0x590, 0x590}, {0x5C8, 0x5CF}, {0x5EB, 0x5EE}, {0x5F5, 0x605}, {0x61C, 0x61D}, {0x6DD, 0x6DD}, {0x70E, 0x70F}, {0x74B, 0x74C}, {0x7B2, 0x7BF}, {0x7FB, 0x7FC}, {0x82E, 0x82F}, {0x83F, 0x83F},
    {0x85C, 0x85D}, {0x85F, 0x85F}, {0x86B, 0x89F}, {0x8B5, 0x8B5}, {0x8C8, 0x8D2}, {0x8E2, 0x8E2}, {0x984, 0x984}, {0x98D, 0x98E}, {0x991, 0x992}, {0x9A9, 0x9A9}, {0x9B1, 0x9B1}, {0x9B3, 0x9B5},
    {0x9BA, 0x9BB}, {0x9C5, 0x9C6}, {0x9C9, 0x9CA}, {0x9CF, 0x9D6}, {0x9D8, 0x9DB}, {0x9DE, 0x9DE}, {0x9E4, 0x9E5}, {0x9FF, 0xA00}, {0xA04, 0xA04}, {0xA0B, 0xA0E}, {0xA11, 0xA12}, {0xA29, 0xA29},
    // ... 其他控制字符范围
};
# 一系列的十六进制数对，表示 Unicode 字符范围
这段代码是一系列十六进制数对，每对数代表 Unicode 字符范围的起始和结束。
# 以下是一系列十六进制数对，表示 Unicode 字符的范围
# 这些范围用于定义字符集合，用于文本处理和编码
# 例如，{0x11939, 0x1193A} 表示 Unicode 字符范围从 0x11939 到 0x1193A
# 该范围内的字符可以被用于文本处理和编码
# 代码中列出了许多这样的范围，用于定义不同的字符集合
// 将 Unicode 编码的 codepoint 转换为 UTF-8 编码的字符串
static std::string codepoint_to_utf8(uint32_t cp) {
    std::string result;
    // 如果 codepoint 在 0x00 到 0x7f 之间，直接将其转换为一个字节的 UTF-8 编码
    if (/* 0x00 <= cp && */ cp <= 0x7f) {
        result.push_back(cp);
    }
    // 如果 codepoint 在 0x80 到 0x7ff 之间，将其转换为两个字节的 UTF-8 编码
    else if (0x80 <= cp && cp <= 0x7ff) {
        result.push_back(0xc0 | ((cp >> 6) & 0x1f));
        result.push_back(0x80 | (cp & 0x3f));
    }
    // 如果 codepoint 在 0x800 到 0xffff 之间，将其转换为三个字节的 UTF-8 编码
    else if (0x800 <= cp && cp <= 0xffff) {
        result.push_back(0xe0 | ((cp >> 12) & 0x0f));
        result.push_back(0x80 | ((cp >> 6) & 0x3f));
        result.push_back(0x80 | (cp & 0x3f));
    }
    // 如果 codepoint 在 0x10000 到 0x10ffff 之间，将其转换为四个字节的 UTF-8 编码
    else if (0x10000 <= cp && cp <= 0x10ffff) {
        result.push_back(0xf0 | ((cp >> 18) & 0x07));
        result.push_back(0x80 | ((cp >> 12) & 0x3f));
        result.push_back(0x80 | ((cp >> 6) & 0x3f));
    // 将结果推入结果向量中，使用位运算将 codepoint 转换为 UTF-8 编码
    result.push_back(0x80 | (cp & 0x3f));
    // 如果 codepoint 无效，则抛出异常
    else {
        throw std::invalid_argument("invalid codepoint");
    }
    // 返回结果向量
    return result;
}

// 将一组 codepoints 转换为 UTF-8 编码的字符串
static std::string codepoints_to_utf8(const std::vector<uint32_t> & cps) {
    std::string result;
    // 遍历 codepoints，将每个 codepoint 转换为 UTF-8 编码并追加到结果字符串中
    for (size_t i = 0; i < cps.size(); ++i) {
        result.append(codepoint_to_utf8(cps[i]));
    }
    // 返回结果字符串
    return result;
}

// 从 UTF-8 编码的字符串中获取一个 codepoint
static uint32_t codepoint_from_utf8(const std::string & utf8, size_t & offset) {
    // 断言偏移量小于 UTF-8 字符串的长度
    assert(offset < utf8.size());
    // 如果第一个字节的最高位为 0，则直接返回该字节
    if (!(utf8[offset + 0] & 0x80)) {
        auto result = utf8[offset + 0];
    // 偏移量加一
    offset += 1;
    // 返回结果
    return result;
}
// 如果 utf8[offset + 0] 的第七位不为1，抛出异常
else if (!(utf8[offset + 0] & 0x40)) {
    throw std::invalid_argument("invalid character");
}
// 如果 utf8[offset + 0] 的第六位不为1
else if (!(utf8[offset + 0] & 0x20)) {
    // 如果偏移量加一大于等于 utf8 的大小，或者 utf8[offset + 1] 的前两位不为10，抛出异常
    if (offset + 1 >= utf8.size() || ! ((utf8[offset + 1] & 0xc0) == 0x80))
        throw std::invalid_argument("invalid character");
    // 计算结果并更新偏移量
    auto result = ((utf8[offset + 0] & 0x1f) << 6) | (utf8[offset + 1] & 0x3f);
    offset += 2;
    // 返回结果
    return result;
}
// 如果 utf8[offset + 0] 的第五位不为1
else if (!(utf8[offset + 0] & 0x10)) {
    // 如果偏移量加二大于等于 utf8 的大小，或者 utf8[offset + 1] 或 utf8[offset + 2] 的前两位不为10，抛出异常
    if (offset + 2 >= utf8.size() || ! ((utf8[offset + 1] & 0xc0) == 0x80) || ! ((utf8[offset + 2] & 0xc0) == 0x80))
        throw std::invalid_argument("invalid character");
    // 计算结果并更新偏移量
    auto result = ((utf8[offset + 0] & 0x0f) << 12) | ((utf8[offset + 1] & 0x3f) << 6) | (utf8[offset + 2] & 0x3f);
    offset += 3;
    // 返回结果
    return result;
}
    else if (!(utf8[offset + 0] & 0x08)) {
        // 如果 UTF-8 编码的第一个字节不是以 10 开头，表示这是一个单字节字符
        if (offset + 3 >= utf8.size() || ! ((utf8[offset + 1] & 0xc0) == 0x80) || ! ((utf8[offset + 2] & 0xc0) == 0x80) || !((utf8[offset + 3] & 0xc0) == 0x80))
            // 如果不满足单字节字符的条件，抛出异常
            throw std::invalid_argument("invalid character");
        // 计算四字节 UTF-8 编码对应的 Unicode 码点
        auto result = ((utf8[offset + 0] & 0x07) << 18) | ((utf8[offset + 1] & 0x3f) << 12) | ((utf8[offset + 2] & 0x3f) << 6) | (utf8[offset + 3] & 0x3f);
        // 偏移量增加 4
        offset += 4;
        // 返回计算结果
        return result;
    }
    // 如果不是单字节字符，抛出异常
    throw std::invalid_argument("invalid string");
}

static std::vector<uint32_t> codepoints_from_utf8(const std::string & utf8) {
    // 从 UTF-8 编码的字符串中提取 Unicode 码点
    std::vector<uint32_t> result;
    // 初始化偏移量
    size_t offset = 0;
    // 循环直到偏移量超出字符串长度
    while (offset < utf8.size()) {
        // 调用函数提取 Unicode 码点并添加到结果数组中
        result.push_back(codepoint_from_utf8(utf8, offset));
    }
    // 返回结果数组
    return result;
}

static std::vector<uint16_t> codepoint_to_utf16(uint32_t cp) {
// 创建一个空的16位无符号整数向量
std::vector<uint16_t> result;
// 如果 codepoint 在 0x0000 到 0xffff 之间
if (/* 0x0000 <= cp && */ cp <= 0xffff) {
    // 将 codepoint 添加到结果向量中
    result.emplace_back(cp);
}
// 如果 codepoint 在 0x10000 到 0x10ffff 之间
else if (0x10000 <= cp && cp <= 0x10ffff) {
    // 将 codepoint 转换为 UTF-16 编码，并添加到结果向量中
    result.emplace_back(0xd800 | ((cp - 0x10000) >> 10));
    result.emplace_back(0xdc00 | ((cp - 0x10000) & 0x03ff));
}
// 如果 codepoint 超出范围
else {
    // 抛出异常，表示无效的 codepoint
    throw std::invalid_argument("invalid codepoint");
}
// 返回结果向量
return result;
}

// 将一组 codepoints 转换为 UTF-16 编码
static std::vector<uint16_t> codepoints_to_utf16(const std::vector<uint32_t> & cps) {
    // 创建一个空的16位无符号整数向量
    std::vector<uint16_t> result;
    // 遍历每个 codepoint
    for (size_t i = 0; i < cps.size(); ++i) {
        // 将每个 codepoint 转换为 UTF-16 编码，并添加到结果向量中
        auto temp = codepoint_to_utf16(cps[i]);
        result.insert(result.end(), temp.begin(), temp.end());
    }
// 返回一个无符号32位整数，表示从给定的UTF-16编码中解析出的码点
static uint32_t codepoint_from_utf16(const std::vector<uint16_t> & utf16, size_t & offset) {
    // 确保偏移量在有效范围内
    assert(offset < utf16.size());
    // 如果第一个16位编码不是代理对的高位
    if (((utf16[0] >> 10) << 10) != 0xd800) {
        // 返回第一个16位编码，并将偏移量加1
        auto result = utf16[offset + 0];
        offset += 1;
        return result;
    }
    else {
        // 如果第一个16位编码是代理对的高位
        if (offset + 1 >= utf16.size() || !((utf16[1] & 0xdc00) == 0xdc00))
            // 如果下一个16位编码不是代理对的低位，抛出异常
            throw std::invalid_argument("invalid character");
        // 计算代理对的码点，并将偏移量加2
        auto result = 0x10000 + (((utf16[0] & 0x03ff) << 10) | (utf16[1] & 0x03ff));
        offset += 2;
        return result;
    }
    // 如果以上条件都不满足，抛出异常
    throw std::invalid_argument("invalid string");
}
// 从 UTF-16 编码的字符数组中获取 Unicode 码点数组
static std::vector<uint32_t> codepoints_from_utf16(const std::vector<uint16_t> & utf16) {
    std::vector<uint32_t> result; // 用于存储结果的数组
    size_t offset = 0; // 偏移量初始化为 0
    while (offset < utf16.size()) // 遍历整个 UTF-16 编码数组
        result.push_back(codepoint_from_utf16(utf16, offset)); // 将每个 UTF-16 编码转换为 Unicode 码点并存储到结果数组中
    return result; // 返回结果数组
}

// 定义不同类型的 Unicode 码点
#define CODEPOINT_TYPE_UNIDENTIFIED 0
#define CODEPOINT_TYPE_DIGIT 1
#define CODEPOINT_TYPE_LETTER 2
#define CODEPOINT_TYPE_WHITESPACE 3
#define CODEPOINT_TYPE_ACCENT_MARK 4
#define CODEPOINT_TYPE_PUNCTUATION 5
#define CODEPOINT_TYPE_SYMBOL 6
#define CODEPOINT_TYPE_CONTROL 7

// 创建 Unicode 码点到类型的映射
static std::unordered_map<uint32_t, int> codepoint_type_map() {
    std::unordered_map<uint32_t, int> codepoint_types; // 用于存储码点类型映射的哈希表
    for (auto p : digit_ranges) { // 遍历数字范围
    // 遍历数字范围，将对应的字符编码类型设置为数字
    for(auto i = p.first; i <= p.second; ++ i)
        codepoint_types[i] = CODEPOINT_TYPE_DIGIT;
    // 遍历字母范围，将对应的字符编码类型设置为字母
    for(auto p : letter_ranges) {
        for(auto i = p.first; i <= p.second; ++ i)
            codepoint_types[i] = CODEPOINT_TYPE_LETTER;
    }
    // 遍历空白字符范围，将对应的字符编码类型设置为空白字符
    for(auto p : whitespace_ranges) {
        for(auto i = p.first; i <= p.second; ++ i)
            codepoint_types[i] = CODEPOINT_TYPE_WHITESPACE;
    }
    // 遍历重音符范围，将对应的字符编码类型设置为重音符
    for(auto p : accent_mark_ranges) {
        for(auto i = p.first; i <= p.second; ++ i)
            codepoint_types[i] = CODEPOINT_TYPE_ACCENT_MARK;
    }
    // 遍历标点符号范围，将对应的字符编码类型设置为标点符号
    for(auto p : punctuation_ranges) {
        for(auto i = p.first; i <= p.second; ++ i)
            codepoint_types[i] = CODEPOINT_TYPE_PUNCTUATION;
    }
    // 遍历符号范围，进行相应的操作（缺少具体代码，无法添加注释）
    for (auto p : symbol_ranges) {
// 遍历控制字符范围，将对应的码点类型设置为符号类型
for (auto i = p.first; i <= p.second; ++i)
    codepoint_types[i] = CODEPOINT_TYPE_SYMBOL;
}
// 遍历控制字符范围，将对应的码点类型设置为控制类型
for(auto p : control_ranges) {
    for(auto i = p.first; i <= p.second; ++ i)
        codepoint_types[i] = CODEPOINT_TYPE_CONTROL;
}
// 返回码点类型数组
return codepoint_types;
}

// 获取指定码点的类型
static int codepoint_type(uint32_t cp) {
    // 创建静态的码点类型映射表
    static std::unordered_map<uint32_t, int> codepoint_types = codepoint_type_map();
    // 返回指定码点的类型
    return codepoint_types[cp];
}

// 获取指定 UTF-8 字符串的码点类型
static int codepoint_type(const std::string & utf8) {
    // 如果字符串长度为0，则返回未识别类型
    if (utf8.length() == 0)
        return CODEPOINT_TYPE_UNIDENTIFIED;
    // 初始化偏移量为0，获取第一个码点的类型
    size_t offset = 0;
    return codepoint_type(codepoint_from_utf8(utf8, offset));
// 创建一个静态函数，返回一个从字节到Unicode字符串的映射的无序映射表
static std::unordered_map<uint8_t, std::string> bytes_to_unicode_map_bpe() {
    // 创建一个空的无序映射表
    std::unordered_map<uint8_t, std::string> map;
    // 遍历ASCII字符集的可打印字符，将其转换为Unicode字符串并添加到映射表中
    for (int ch = u'!'; ch <= u'~'; ++ch) {
        assert(0 <= ch && ch < 256);
        map[ch] = codepoint_to_utf8(ch);
    }
    // 遍历其他特殊字符范围，将其转换为Unicode字符串并添加到映射表中
    for (int ch = u'¡'; ch <= u'¬'; ++ch) {
        assert(0 <= ch && ch < 256);
        map[ch] = codepoint_to_utf8(ch);
    }
    // 遍历剩余的字符范围，将其转换为Unicode字符串并添加到映射表中
    for (int ch = u'®'; ch <= u'ÿ'; ++ch) {
        assert(0 <= ch && ch < 256);
        map[ch] = codepoint_to_utf8(ch);
    }
    // 对于剩余的未映射的字符，将其转换为Unicode字符串并添加到映射表中
    auto n = 0;
    for (int ch = 0; ch < 256; ++ch) {
        if (map.find(ch) == map.end()) {
            map[ch] = codepoint_to_utf8(256 + n);
// 增加变量 n 的值
++n;
// 返回结果 map
}
// 将字节转换为 Unicode BPE
static std::string bytes_to_unicode_bpe(uint8_t byte) {
    // 创建字节到 Unicode 映射表
    static std::unordered_map<uint8_t, std::string> map = bytes_to_unicode_map_bpe();
    // 返回字节对应的 Unicode
    return map.at(byte);
}

// 创建 Unicode 到字节的映射表
static std::unordered_map<std::string, uint8_t> unicode_to_bytes_map_bpe() {
    // 创建映射表
    std::unordered_map<std::string, uint8_t> map;
    // 遍历字符集合
    for (int ch = u'!'; ch <= u'~'; ++ch) {
        // 断言字符在合法范围内
        assert(0 <= ch && ch < 256);
        // 将 Unicode 转换为 UTF-8，并映射到对应的字节
        map[codepoint_to_utf8(ch)] = ch;
    }
    // 遍历字符集合
    for (int ch = u'¡'; ch <= u'¬'; ++ch) {
        // 断言字符在合法范围内
        assert(0 <= ch && ch < 256);
        // 将 Unicode 转换为 UTF-8，并映射到对应的字节
        map[codepoint_to_utf8(ch)] = ch;
    }
    // 循环遍历从特定字符到特定字符的范围
    for (int ch = u'®'; ch <= u'ÿ'; ++ch) {
        // 断言特定字符的范围在0到255之间
        assert(0 <= ch && ch < 256);
        // 将特定字符转换为UTF-8编码，并将其映射到对应的字符
        map[codepoint_to_utf8(ch)] = ch;
    }
    // 初始化变量n为0
    auto n = 0;
    // 循环遍历0到255之间的字符
    for (int ch = 0; ch < 256; ++ch) {
        // 如果映射中不包含当前字符的UTF-8编码
        if (map.find(codepoint_to_utf8(ch)) == map.end()) {
            // 将当前字符的UTF-8编码映射到256+n的位置，并增加n的值
            map[codepoint_to_utf8(256 + n)] = ch;
            ++n;
        }
    }
    // 返回映射表
    return map;
}

// 将Unicode字符转换为字节的位数
static uint8_t unicode_to_bytes_bpe(const std::string & utf8) {
    // 创建静态的Unicode字符到字节位数的映射表
    static std::unordered_map<std::string, uint8_t> map = unicode_to_bytes_map_bpe();
    // 返回给定UTF-8编码对应的字节位数
    return map.at(utf8);
}
抱歉，我无法为您提供代码注释，因为您没有提供任何代码。如果您有任何需要帮助的代码，请随时告诉我。我会尽力帮助您。
```