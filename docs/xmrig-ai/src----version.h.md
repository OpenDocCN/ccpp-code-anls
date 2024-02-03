# `xmrig\src\version.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   许可证的版本为 3 或者（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_VERSION_H
#define XMRIG_VERSION_H

#define APP_ID        "xmrig"
#define APP_NAME      "XMRig"
#define APP_DESC      "XMRig miner"
#define APP_VERSION   "6.21.0"
#define APP_DOMAIN    "xmrig.com"
#define APP_SITE      "www.xmrig.com"
#define APP_COPYRIGHT "版权所有 (C) 2016-2023 xmrig.com"
#define APP_KIND      "miner"

#define APP_VER_MAJOR  6
#define APP_VER_MINOR  21
#define APP_VER_PATCH  0

#ifdef _MSC_VER
#   if (_MSC_VER >= 1930)
#       define MSVC_VERSION 2022
#   elif (_MSC_VER >= 1920 && _MSC_VER < 1930)
#       define MSVC_VERSION 2019
#   elif (_MSC_VER >= 1910 && _MSC_VER < 1920)
#       define MSVC_VERSION 2017
#   elif _MSC_VER == 1900
#       define MSVC_VERSION 2015
#   elif _MSC_VER == 1800
#       define MSVC_VERSION 2013
#   elif _MSC_VER == 1700
#       define MSVC_VERSION 2012
#   elif _MSC_VER == 1600
#       define MSVC_VERSION 2010
#   else
#       define MSVC_VERSION 0
#   endif
#endif

#ifdef XMRIG_OS_WIN
#    define APP_OS "Windows"
#elif defined XMRIG_OS_IOS
#    define APP_OS "iOS"
#elif defined XMRIG_OS_MACOS
#    define APP_OS "macOS"
#elif defined XMRIG_OS_ANDROID
#    define APP_OS "Android"
// 如果定义了 XMRIG_OS_LINUX，则定义 APP_OS 为 "Linux"
#elif defined XMRIG_OS_LINUX
#    define APP_OS "Linux"
// 如果定义了 XMRIG_OS_FREEBSD，则定义 APP_OS 为 "FreeBSD"
#elif defined XMRIG_OS_FREEBSD
#    define APP_OS "FreeBSD"
// 如果都不符合，则定义 APP_OS 为 "Unknown OS"
#else
#    define APP_OS "Unknown OS"
#endif

// 定义一个宏函数 STR，用于将参数 X 转换为字符串
#define STR(X) #X
// 定义一个宏函数 STR2，用于将参数 X 转换为字符串
#define STR2(X) STR(X)

// 如果定义了 XMRIG_ARM，则定义 APP_ARCH 为 "ARMv" 加上 XMRIG_ARM 的字符串值
#ifdef XMRIG_ARM
#   define APP_ARCH "ARMv" STR2(XMRIG_ARM)
// 如果没有定义 XMRIG_ARM，则根据不同的条件定义 APP_ARCH
#else
#   if defined(__x86_64__) || defined(__amd64__) || defined(_M_X64) || defined(_M_AMD64)
#       define APP_ARCH "x86-64"
#   else
#       define APP_ARCH "x86"
#   endif
#endif

// 如果定义了 XMRIG_64_BIT，则定义 APP_BITS 为 "64 bit"
#ifdef XMRIG_64_BIT
#   define APP_BITS "64 bit"
// 如果没有定义 XMRIG_64_BIT，则定义 APP_BITS 为 "32 bit"
#else
#   define APP_BITS "32 bit"
#endif

// 结束 XMRIG_VERSION_H 的定义
#endif // XMRIG_VERSION_H
```