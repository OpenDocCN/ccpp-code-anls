# `xmrig\src\base\tools\bswap_64.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BSWAP_64_H
#define XMRIG_BSWAP_64_H

#ifdef _MSC_VER

#include <stdlib.h>
#define bswap_64(x) _byteswap_uint64(x)  // 在 Windows 平台上使用内置函数 _byteswap_uint64 进行64位字节交换
#define bswap_32(x) _byteswap_ulong(x)    // 在 Windows 平台上使用内置函数 _byteswap_ulong 进行32位字节交换

#elif defined __GNUC__

#define bswap_64(x) __builtin_bswap64(x)  // 在 GNU 编译器上使用内置函数 __builtin_bswap64 进行64位字节交换
#define bswap_32(x) __builtin_bswap32(x)  // 在 GNU 编译器上使用内置函数 __builtin_bswap32 进行32位字节交换

#else

#include <byteswap.h>  // 在其他平台上包含 byteswap.h 头文件

#endif

#endif /* XMRIG_BSWAP_64_H */
```