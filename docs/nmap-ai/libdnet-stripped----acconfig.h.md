# `nmap\libdnet-stripped\acconfig.h`

```cpp
@BOTTOM@
// 定义了一些系统类型

#include <sys/types.h>

#ifdef HAVE_WINSOCK2_H
# include <winsock2.h>
# include <windows.h>
#endif

#ifdef __svr4__
# define BSD_COMP    1
#endif

#if defined(__osf__) && !defined(_SOCKADDR_LEN)
# define _SOCKADDR_LEN    1
#endif

#ifndef HAVE_INET_PTON
// 如果没有定义 HAVE_INET_PTON，则定义 inet_pton 函数
int    inet_pton(int, const char *, void *);
#endif

#ifndef HAVE_STRLCPY
// 如果没有定义 HAVE_STRLCPY，则定义 strlcpy 函数
int    strlcpy(char *, const char *, int);
#endif

#ifndef HAVE_STRSEP
// 如果没有定义 HAVE_STRSEP，则定义 strsep 函数
char    *strsep(char **, const char *);
#endif

#ifndef HAVE_SOCKLEN_T
// 如果没有定义 HAVE_SOCKLEN_T，则定义 socklen_t 类型
typedef int socklen_t;
#endif
```