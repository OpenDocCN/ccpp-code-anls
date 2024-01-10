# `nmap\nbase\nbase_winunix.h`

```
/* $Id$ */

#ifndef NBASE_WINUNIX_H
#define NBASE_WINUNIX_H

#include "nbase_winconfig.h"

/* Winsock defines its own error codes that are analogous to but
   different from those in <errno.h>. The error macros have similar
   names, for example
     EINTR -> WSAEINTR
     ECONNREFUSED -> WSAECONNREFUSED
   But the values are different. The errno codes are small integers,
   while the Winsock codes start at 10000 or so.
   http://msdn.microsoft.com/en-us/library/ms737828

   Later in this file there is a block of code that defines the errno
   names to their Winsock equivalents, so that you can write code using
   the errno names only, and have it still work on Windows. However this
   causes some problems that are worked around in the following few
   lines. First, we prohibit the inclusion of <errno.h>, so that the
   only error codes visible are those we explicitly define in this file.
   This will cause a compilation error if someone uses a code we're not
   yet aware of instead of using an incompatible value at runtime.
   Second, because <errno.h> is not defined, the C++0x header
   <system_error> doesn't compile, so we pretend not to have C++0x to
   avoid it. */
#if _MSC_VER < 1600 /* Breaks on VS2010 and later */
#define _INC_ERRNO  /* suppress errno.h */
#define _ERRNO_H_ /* Also for errno.h suppression */
#define _SYSTEM_ERROR_
#undef _HAS_CPP0X
#define _HAS_CPP0X 0
#else
/* VS2013: we include errno.h, then redefine the constants we want.
 * This may work in other versions, but haven't tested (since the other method
 * has been working just fine). */
#include <errno.h>
#endif

/* Suppress winsock.h */
#define _WINSOCKAPI_
#define WIN32_LEAN_AND_MEAN

#include <windows.h>
#include <winsock2.h>
#include <ws2tcpip.h> /* IPv6 stuff */
#if HAVE_WSPIAPI_H
/* <wspiapi.h> is necessary for getaddrinfo before Windows XP, but it isn't
   available on some platforms like MinGW. */
#include <wspiapi.h>
#endif
#include <time.h>
#include <iptypes.h>
#include <stdlib.h>  // 标准库函数声明
#include <malloc.h>  // 内存分配函数声明
#include <io.h>  // 文件 I/O 函数声明
#include <fcntl.h>  // 文件控制函数声明
#include <sys/stat.h>  // 系统状态函数声明
#include <sys/types.h>  // 系统类型函数声明
#include <process.h>  // 进程控制函数声明
#include <limits.h>  // 整数限制声明
#include <WINCRYPT.H>  // Windows 加密 API 声明
#include <math.h>  // 数学函数声明

#define SIOCGIFCONF     0x8912  // 获取接口列表的控制命令

#ifndef GLOBALS
#define GLOBALS 1  // 如果未定义 GLOBALS，则定义为 1
#endif

#define munmap(ptr, len) win32_munmap(ptr, len)  // 定义 munmap 宏，调用 win32_munmap 函数

/* Windows error message names */
#undef  ECONNABORTED
#define ECONNABORTED    WSAECONNABORTED  // 连接中止错误
#undef ECONNRESET
#define ECONNRESET      WSAECONNRESET  // 连接重置错误
#undef ECONNREFUSED
#define ECONNREFUSED    WSAECONNREFUSED  // 连接拒绝错误
#undef  EAGAIN
#define EAGAIN        WSAEWOULDBLOCK  // 再试一次错误
#undef EWOULDBLOCK
#define EWOULDBLOCK    WSAEWOULDBLOCK  // 再试一次错误
#undef EHOSTUNREACH
#define EHOSTUNREACH    WSAEHOSTUNREACH  // 主机不可达错误
#undef ENETDOWN
#define ENETDOWN    WSAENETDOWN  // 网络关闭错误
#undef ENETUNREACH
#define ENETUNREACH    WSAENETUNREACH  // 网络不可达错误
#undef ENETRESET
#define ENETRESET    WSAENETRESET  // 网络重置错误
#undef ETIMEDOUT
#define ETIMEDOUT    WSAETIMEDOUT  // 连接超时错误
#undef EHOSTDOWN
#define EHOSTDOWN    WSAEHOSTDOWN  // 主机关闭错误
#undef EINPROGRESS
#define EINPROGRESS    WSAEINPROGRESS  // 操作进行中错误
#undef  EINVAL
#define EINVAL          WSAEINVAL      // 无效参数错误
#undef  EPERM
#define EPERM           WSAEACCES      // 操作不允许错误
#undef  EACCES
#define EACCES          WSAEACCES     // 操作不允许错误
#undef  EINTR
#define EINTR           WSAEINTR      // 中断系统调用错误
#undef ENOBUFS
#define ENOBUFS         WSAENOBUFS     // 没有可用的缓冲区错误
#undef EMSGSIZE
#define EMSGSIZE        WSAEMSGSIZE    // 消息过长错误
#undef  ENOMEM
#define ENOMEM          WSAENOBUFS  // 没有可用的缓冲区错误
#undef  ENOTSOCK
#define ENOTSOCK        WSAENOTSOCK  // 不是套接字错误
#undef  EOPNOTSUPP
#define EOPNOTSUPP      WSAEOPNOTSUPP  // 操作不支持错误
#undef  EIO
#define EIO             WSASYSCALLFAILURE  // 系统调用失败错误

/*
This is not used by our network code, and causes problems in programs using
Nbase that legitimately use ENOENT for file operations.
#undef  ENOENT
#define ENOENT          WSAENOENT
*/

#define close(x) closesocket(x)  // 定义 close 宏，调用 closesocket 函数
// 定义一个无符号短整型的别名 u_short_t
typedef unsigned short u_short_t;

// 声明一个函数 win_stdin_start_thread，返回类型为整型
int win_stdin_start_thread(void);

// 声明一个函数 win_stdin_ready，返回类型为整型
int win_stdin_ready();

// 结束条件编译指令，关闭 NBASE_WINUNIX_H 头文件
#endif /* NBASE_WINUNIX_H */
```