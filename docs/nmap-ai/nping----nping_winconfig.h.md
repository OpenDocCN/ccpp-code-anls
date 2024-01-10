# `nmap\nping\nping_winconfig.h`

```
/* $Id: nmap_winconfig.h 12955 2009-04-15 00:37:03Z fyodor $ */
// 定义了 nmap_winconfig.h 文件的版本信息

#ifndef NPING_WINCONFIG_H
#define NPING_WINCONFIG_H
// 如果 NPING_WINCONFIG_H 未定义，则定义 NPING_WINCONFIG_H

/* Without this, Windows will give us all sorts of crap about using functions
   like strcpy() even if they are done safely */
#define _CRT_SECURE_NO_DEPRECATE 1
// 定义 _CRT_SECURE_NO_DEPRECATE 为 1，避免在使用一些函数时出现警告

//#ifndef NPING_NAME
//    #define NPING_NAME "Nmap"
//    #define NPING_URL "https://nmap.org"
//#endif
// 如果 NPING_NAME 未定义，则定义 NPING_NAME 为 "Nmap"，NPING_URL 为 "https://nmap.org"

#ifdef NPING_PLATFORM
    #undef NPING_PLATFORM 
#endif
// 如果 NPING_PLATFORM 已定义，则取消定义 NPING_PLATFORM

#define NPING_PLATFORM "i686-pc-windows-windows"
// 定义 NPING_PLATFORM 为 "i686-pc-windows-windows"

#define HAVE_OPENSSL 1
#define HAVE_SSL_SET_TLSEXT_HOST_NAME 1
// 定义 HAVE_OPENSSL 和 HAVE_SSL_SET_TLSEXT_HOST_NAME 为 1

/* Apparently __func__ isn't yet supported */
#define __func__ __FUNCTION__
// 定义 __func__ 为 __FUNCTION__

typedef unsigned __int32 u_int32_t;
typedef unsigned __int16 u_int16_t;
typedef unsigned __int8 u_int8_t;
// 定义无符号整型别名

#endif /* NPING_WINCONFIG_H */
// 结束条件编译指令，定义结束
```