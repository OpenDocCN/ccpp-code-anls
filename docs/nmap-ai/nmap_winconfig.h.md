# `nmap\nmap_winconfig.h`

```
/* $Id$ */

#ifndef NMAP_WINCONFIG_H
#define NMAP_WINCONFIG_H
/* 如果不定义这个，Windows 将会在使用像 strcpy() 这样的函数时给出各种警告 */
#define _CRT_SECURE_NO_DEPRECATE 1
#define NMAP_PLATFORM "i686-pc-windows-windows"

#define HAVE_OPENSSL 1
#define HAVE_SSL_SET_TLSEXT_HOST_NAME 1
#define HAVE_LIBSSH2 1
#define HAVE_LIBZ 1
/* 自从 MSVC 2010，stdint.h 被包含作为 C99 兼容的一部分 */
#define HAVE_STDINT_H 1

#define LUA_INCLUDED 1
#undef PCAP_INCLUDED
#define DNET_INCLUDED 1
#define PCRE_INCLUDED 1
#define LIBSSH2_INCLUDED 1
#define ZLIB_INCLUDED 1

#endif /* NMAP_WINCONFIG_H */
```