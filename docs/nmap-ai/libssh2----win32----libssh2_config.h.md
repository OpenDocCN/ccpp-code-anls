# `nmap\libssh2\win32\libssh2_config.h`

```
#ifndef LIBSSH2_CONFIG_H
#define LIBSSH2_CONFIG_H
// 如果 LIBSSH2_CONFIG_H 未定义，则定义 LIBSSH2_CONFIG_H

#ifndef WIN32
#define WIN32
#endif
// 如果 WIN32 未定义，则定义 WIN32

#ifndef _CRT_SECURE_NO_DEPRECATE
#define _CRT_SECURE_NO_DEPRECATE 1
#endif /* _CRT_SECURE_NO_DEPRECATE */
// 如果 _CRT_SECURE_NO_DEPRECATE 未定义，则定义 _CRT_SECURE_NO_DEPRECATE 为 1

#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
// 引入 Windows 网络编程相关的头文件

#ifdef __MINGW32__
#define HAVE_UNISTD_H
#define HAVE_INTTYPES_H
#define HAVE_SYS_TIME_H
#define HAVE_GETTIMEOFDAY
#endif /* __MINGW32__ */
// 如果是在 MinGW32 环境下，则定义一些宏

#define HAVE_LIBCRYPT32
#define HAVE_WINSOCK2_H
#define HAVE_IOCTLSOCKET
#define HAVE_SELECT
// 定义一些宏

#ifdef _MSC_VER
#if _MSC_VER < 1900
#define snprintf _snprintf
#if _MSC_VER < 1500
#define vsnprintf _vsnprintf
#endif
#define strdup _strdup
#define strncasecmp _strnicmp
#define strcasecmp _stricmp
#endif
#else
#ifndef __MINGW32__
#define strncasecmp strnicmp
#define strcasecmp stricmp
#endif /* __MINGW32__ */
#endif /* _MSC_VER */
// 根据不同的编译器和环境，定义一些字符串处理相关的宏

/* Enable newer diffie-hellman-group-exchange-sha1 syntax */
#define LIBSSH2_DH_GEX_NEW 1
// 启用新的 diffie-hellman-group-exchange-sha1 语法

#endif /* LIBSSH2_CONFIG_H */
// 结束 LIBSSH2_CONFIG_H 的定义
```