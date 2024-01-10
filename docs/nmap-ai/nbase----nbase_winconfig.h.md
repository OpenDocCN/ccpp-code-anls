# `nmap\nbase\nbase_winconfig.h`

```
/* $Id$ */

#ifndef NBASE_WINCONFIG_H
#define NBASE_WINCONFIG_H

/* 定义我们支持的最早版本的 Windows。这些控制了 Windows API 的可用部分。可用的常量在 <sdkddkver.h> 中。
http://msdn.microsoft.com/en-us/library/aa383745.aspx
http://blogs.msdn.com/oldnewthing/archive/2007/04/11/2079137.aspx */
#undef _WIN32_WINNT
#define _WIN32_WINNT _WIN32_WINNT_WIN7
#undef NTDDI_VERSION
#define NTDDI_VERSION NTDDI_WIN7

// 这禁用了警告 4800 http://msdn.microsoft.com/en-us/library/b6801kcy(v=vs.71).aspx
#pragma warning(disable : 4800)
/* 它实际上没有 struct IP，但我们使用一个不同的来代替 Nmap 自带的那个 */
#define HAVE_STRUCT_IP 1
/* #define HAVE_STRUCT_ICMP 1 */
#define HAVE_STRNCASECMP 1
#define HAVE_IP_IP_SUM 1
#define STDC_HEADERS 1
#define HAVE_STRING_H 1
#define HAVE_MEMORY_H 1
#define HAVE_FCNTL_H 1
#define HAVE_ERRNO_H 1
#define HAVE_SYS_STAT_H 1
#define HAVE_MEMCPY 1
#define HAVE_STRERROR 1
/* #define HAVE_SYS_SOCKIO_H 1 */
/* #undef HAVE_TERMIOS_H */
#define HAVE_ERRNO_H 1
#define HAVE_GAI_STRERROR 1
/* #define HAVE_STRCASESTR 1 */
#define HAVE_STRCASECMP 1
#define HAVE_NETINET_IF_ETHER_H 1
#define HAVE_SYS_STAT_H 1
/* #define HAVE_INTTYPES_H */

/* 这些函数在 Vista 及更高版本可用 */
#if defined(_WIN32_WINNT) && _WIN32_WINNT >= _WIN32_WINNT_WIN6
#define HAVE_INET_PTON 1
#define HAVE_INET_NTOP 1
#endif

#ifdef _MSC_VER
/* <wspiapi.h> 只有 Visual Studio 才有。 */
#define HAVE_WSPIAPI_H 1
#else
#undef HAVE_WSPIAPI_H
#endif

#define HAVE_GETADDRINFO 1
#define HAVE_GETNAMEINFO 1

#define HAVE_SNPRINTF 1
// #undef HAVE_VASPRINTF
#define HAVE_VSNPRINTF 1

typedef unsigned __int8 uint8_t;
typedef unsigned __int16 uint16_t;
typedef unsigned __int32 uint32_t;
typedef unsigned __int64 uint64_t;
typedef signed __int8 int8_t;
typedef signed __int16 int16_t;
typedef signed __int32 int32_t;
typedef signed __int64 int64_t;

#define HAVE_IPV6 1
#define HAVE_AF_INET6 1
#define HAVE_SOCKADDR_STORAGE 1

/* 定义宏，表示系统支持 AF_INET6 地址族 */
#define _CRT_SECURE_NO_DEPRECATE 1
/* 如果未定义 _CRT_SECURE_NO_WARNINGS 宏，则定义它为 1 */
#ifndef _CRT_SECURE_NO_WARNINGS
#define _CRT_SECURE_NO_WARNINGS 1
#endif
/* 禁止特定警告，例如禁止警告 4996 */
#pragma warning(disable: 4996)

#ifdef __GNUC__
/* 如果是 GCC 编译器，则定义 bzero 宏为 __builtin_memset 函数 */
#define bzero(addr, num) __builtin_memset (addr, '\0', num)
#else
/* 否则定义 __attribute__ 为空 */
#define __attribute__(x)
#endif

/* 定义 __func__ 为 __FUNCTION__ */
#define __func__ __FUNCTION__

#endif /* NBASE_WINCONFIG_H */
```