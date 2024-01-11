# `xmrig\src\3rdparty\rapidjson\internal\clzll.h`

```
// 定义了一个宏，用于获取64位整数的前导零的数量
#ifndef RAPIDJSON_CLZLL_H_
#define RAPIDJSON_CLZLL_H_

// 包含 rapidjson.h 文件
#include "../rapidjson.h"

// 如果编译器是 MSC 并且不是在 Windows CE 下
#if defined(_MSC_VER) && !defined(UNDER_CE)
// 包含内联汇编头文件
#include <intrin.h>
// 如果是 64 位 Windows
#if defined(_WIN64)
// 使用内联汇编获取 64 位整数的前导零数量
#pragma intrinsic(_BitScanReverse64)
#else
// 使用内联汇编获取 32 位整数的前导零数量
#pragma intrinsic(_BitScanReverse)
#endif
#endif

// 进入 rapidjson 命名空间
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

// 定义一个内联函数，用于获取 64 位整数的前导零数量
inline uint32_t clzll(uint64_t x) {
    // 断言 x 不等于 0
    RAPIDJSON_ASSERT(x != 0);

    // 如果编译器是 MSC 并且不是在 Windows CE 下
#if defined(_MSC_VER) && !defined(UNDER_CE)
    // 定义一个无符号长整型变量 r
    unsigned long r = 0;
    // 如果是 64 位 Windows
#if defined(_WIN64)
    // 使用内联汇编获取 64 位整数的前导零数量
    _BitScanReverse64(&r, x);
#else
    // 扫描高 32 位
    if (_BitScanReverse(&r, static_cast<uint32_t>(x >> 32)))
        return 63 - (r + 32);

    // 扫描低 32 位
    _BitScanReverse(&r, static_cast<uint32_t>(x & 0xFFFFFFFF));
#endif // _WIN64

    return 63 - r;
// 如果是 GCC 编译器并且版本大于等于 4，或者支持 __builtin_clzll
#elif (defined(__GNUC__) && __GNUC__ >= 4) || RAPIDJSON_HAS_BUILTIN(__builtin_clzll)
    // 使用内建函数获取 64 位整数的前导零数量
    return static_cast<uint32_t>(__builtin_clzll(x));
#else
    // 朴素版本
    uint32_t r = 0;
    // 当 x 的最高位不为 1 时
    while (!(x & (static_cast<uint64_t>(1) << 63))) {
        x <<= 1;
        ++r;
    }

    return r;
#endif // _MSC_VER
}

// 定义宏 RAPIDJSON_CLZLL，指向 internal 命名空间下的 clzll 函数
#define RAPIDJSON_CLZLL RAPIDJSON_NAMESPACE::internal::clzll

} // namespace internal
// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END
// 结束条件编译指令，关闭 CLZLL_H_ 宏定义
#endif // RAPIDJSON_CLZLL_H_
```