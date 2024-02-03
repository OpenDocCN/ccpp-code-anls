# `nmap\libssh2\os400\libssh2_config.h`

```cpp
/*
 * 版权声明，版权所有
 * 2015年 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 * 禁止在源代码和二进制形式中重新分发和使用，无论是否经过修改，只要满足以下条件：
 *   1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害。
 */

#ifndef LIBSSH2_CONFIG_H
#define LIBSSH2_CONFIG_H

/* 如果正在构建通用（内部辅助宏），则定义 */
#undef AC_APPLE_UNIVERSAL_BUILD

/* 为Cray-2和Cray-YMP系统之一定义`_getb67'、`GETB67'、`getb67'。这个函数在这些系统上对于`alloca.c'的支持是必需的。 */
#undef CRAY_STACKSEG_END
/* Define to 1 if using `alloca.c'. */
#undef C_ALLOCA
/* 如果使用`alloca.c'，则定义为1 */

/* Define to 1 if you have `alloca', as a function or macro. */
#define HAVE_ALLOCA 1
/* 如果有`alloca`函数或宏，则定义为1 */

/* Define to 1 if you have <alloca.h> and it should be used (not on Ultrix). */
#define HAVE_ALLOCA_H 1
/* 如果有<alloca.h>头文件并且应该使用它（不在Ultrix上），则定义为1 */

/* Define to 1 if you have the <arpa/inet.h> header file. */
#define HAVE_ARPA_INET_H 1
/* 如果有<arpa/inet.h>头文件，则定义为1 */

/* Define to 1 if you have the declaration of `SecureZeroMemory', and to 0 if you don't. */
#undef HAVE_DECL_SECUREZEROMEMORY
/* 如果有`SecureZeroMemory`的声明，则定义为1，否则定义为0 */

/* disabled non-blocking sockets */
#undef HAVE_DISABLED_NONBLOCKING
/* 禁用非阻塞套接字 */

/* Define to 1 if you have the <dlfcn.h> header file. */
#undef HAVE_DLFCN_H
/* 如果有<dlfcn.h>头文件，则定义为1 */

/* Define to 1 if you have the <errno.h> header file. */
#define HAVE_ERRNO_H 1
/* 如果有<errno.h>头文件，则定义为1 */

/* Define to 1 if you have the `EVP_aes_128_ctr' function. */
#undef HAVE_EVP_AES_128_CTR
/* 如果有`EVP_aes_128_ctr`函数，则定义为1 */

/* Define to 1 if you have the <fcntl.h> header file. */
#define HAVE_FCNTL_H 1
/* 如果有<fcntl.h>头文件，则定义为1 */

/* use FIONBIO for non-blocking sockets */
#undef HAVE_FIONBIO
/* 使用FIONBIO来处理非阻塞套接字 */

/* Define to 1 if you have the `gettimeofday' function. */
#define HAVE_GETTIMEOFDAY 1
/* 如果有`gettimeofday`函数，则定义为1 */

/* Define to 1 if you have the <inttypes.h> header file. */
#define HAVE_INTTYPES_H 1
/* 如果有<inttypes.h>头文件，则定义为1 */

/* use ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET
/* 使用ioctlsocket()来处理非阻塞套接字 */

/* use Ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET_CASE
/* 使用Ioctlsocket()来处理非阻塞套接字 */

/* Define if you have the bcrypt library. */
#undef HAVE_LIBBCRYPT
/* 如果有bcrypt库，则定义为1 */

/* Define if you have the crypt32 library. */
#undef HAVE_LIBCRYPT32
/* 如果有crypt32库，则定义为1 */

/* Define if you have the gcrypt library. */
#undef HAVE_LIBGCRYPT
/* 如果有gcrypt库，则定义为1 */

/* Define if you have the ssl library. */
#undef HAVE_LIBSSL
/* 如果有ssl库，则定义为1 */

/* Define if you have the z library. */
/* #undef HAVE_LIBZ */
/* 如果有z库，则定义为1 */

/* Define to 1 if the compiler supports the 'long long' data type. */
#define HAVE_LONGLONG 1
/* 如果编译器支持'long long'数据类型，则定义为1 */

/* Define to 1 if you have the <memory.h> header file. */
#undef HAVE_MEMORY_H
/* 如果有<memory.h>头文件，则定义为1 */

/* Define to 1 if you have the <netinet/in.h> header file. */
#define HAVE_NETINET_IN_H 1
/* 如果有<netinet/in.h>头文件，则定义为1 */

/* Define to 1 if you have the <ntdef.h> header file. */
#undef HAVE_NTDEF_H
/* 如果有<ntdef.h>头文件，则定义为1 */

/* Define to 1 if you have the <ntstatus.h> header file. */
/* 如果有<ntstatus.h>头文件，则定义为1 */
/* 定义未定义的宏，表示没有 NTSTATUS_H */
#undef HAVE_NTSTATUS_H

/* 使用 O_NONBLOCK 用于非阻塞套接字 */
#define HAVE_O_NONBLOCK 1

/* 如果有 poll 函数，则定义为 1 */
#undef HAVE_POLL

/* 如果有 select 函数，则定义为 1 */
#define HAVE_SELECT 1

/* 使用 SO_NONBLOCK 用于非阻塞套接字 */
#undef HAVE_SO_NONBLOCK

/* 如果有 stdint.h 头文件，则定义为 1 */
#define HAVE_STDINT_H 1

/* 如果有 stdio.h 头文件，则定义为 1 */
#define HAVE_STDIO_H 1

/* 如果有 stdlib.h 头文件，则定义为 1 */
#define HAVE_STDLIB_H 1

/* 如果有 strings.h 头文件，则定义为 1 */
#define HAVE_STRINGS_H 1

/* 如果有 string.h 头文件，则定义为 1 */
#define HAVE_STRING_H 1

/* 如果有 strtoll 函数，则定义为 1 */
#define HAVE_STRTOLL 1

/* 如果有 sys/ioctl.h 头文件，则定义为 1 */
#define HAVE_SYS_IOCTL_H 1

/* 如果有 sys/select.h 头文件，则定义为 1 */
#undef HAVE_SYS_SELECT_H

/* 如果有 sys/socket.h 头文件，则定义为 1 */
#define HAVE_SYS_SOCKET_H 1

/* 如果有 sys/stat.h 头文件，则定义为 1 */
#define HAVE_SYS_STAT_H 1

/* 如果有 sys/time.h 头文件，则定义为 1 */
#define HAVE_SYS_TIME_H 1

/* 如果有 sys/types.h 头文件，则定义为 1 */
#define HAVE_SYS_TYPES_H 1

/* 如果有 sys/uio.h 头文件，则定义为 1 */
#define HAVE_SYS_UIO_H 1

/* 如果有 sys/un.h 头文件，则定义为 1 */
#define HAVE_SYS_UN_H 1

/* 如果有 unistd.h 头文件，则定义为 1 */
#define HAVE_UNISTD_H 1

/* 如果有 windows.h 头文件，则定义为 1 */
#undef HAVE_WINDOWS_H

/* 如果有 winsock2.h 头文件，则定义为 1 */
#undef HAVE_WINSOCK2_H

/* 如果有 ws2tcpip.h 头文件，则定义为 1 */
#undef HAVE_WS2TCPIP_H

/* 使符号可见 */
#undef LIBSSH2_API

/* 启用在释放之前清除内存 */
#define LIBSSH2_CLEAR_MEMORY 1
/* Enable "none" cipher -- NOT RECOMMENDED */
#undef LIBSSH2_CRYPT_NONE

/* Enable newer diffie-hellman-group-exchange-sha1 syntax */
#define LIBSSH2_DH_GEX_NEW 1

/* Compile in zlib support */
/* #undef LIBSSH2_HAVE_ZLIB */

/* Use libgcrypt */
#undef LIBSSH2_LIBGCRYPT

/* Enable "none" MAC -- NOT RECOMMENDED */
#undef LIBSSH2_MAC_NONE

/* Use OpenSSL */
#undef LIBSSH2_OPENSSL

/* Use Windows CNG */
#undef LIBSSH2_WINCNG

/* Use OS/400 Qc3 */
#define LIBSSH2_OS400QC3

/* Define to the sub-directory in which libtool stores uninstalled libraries.
*/
#define LT_OBJDIR ".libs/"

/* Define to 1 if _REENTRANT preprocessor symbol must be defined. */
#undef NEED_REENTRANT

/* Name of package */
#define PACKAGE "libssh2"

/* Define to the address where bug reports for this package should be sent. */
#define PACKAGE_BUGREPORT "libssh2-devel@cool.haxx.se"

/* Define to the full name of this package. */
#define PACKAGE_NAME "libssh2"

/* Define to the full name and version of this package. */
#define PACKAGE_STRING "libssh2 -"

/* Define to the one symbol short name of this package. */
#define PACKAGE_TARNAME "libssh2"

/* Define to the home page for this package. */
#define PACKAGE_URL ""

/* Define to the version of this package. */
#define PACKAGE_VERSION "-"

/* If using the C implementation of alloca, define if you know the
   direction of stack growth for your system; otherwise it will be
   automatically deduced at runtime.
    STACK_DIRECTION > 0 => grows toward higher addresses
    STACK_DIRECTION < 0 => grows toward lower addresses
    STACK_DIRECTION = 0 => direction of growth unknown */
#undef STACK_DIRECTION

/* Define to 1 if you have the ANSI C header files. */
#define STDC_HEADERS 1

/* Version number of package */
#define VERSION "-"

/* Define WORDS_BIGENDIAN to 1 if your processor stores words with the most
   significant byte first (like Motorola and SPARC, unlike Intel). */
#define WORDS_BIGENDIAN 1

/* Enable large inode numbers on Mac OS X 10.5.  */
#ifndef _DARWIN_USE_64_BIT_INODE
# define _DARWIN_USE_64_BIT_INODE 1
#endif

/* 如果宏 _DARWIN_USE_64_BIT_INODE 未定义，则定义为 1 */

#undef _FILE_OFFSET_BITS
/* 取消对宏 _FILE_OFFSET_BITS 的定义 */

#undef _LARGE_FILES
/* 取消对宏 _LARGE_FILES 的定义 */

#undef const
/* 取消对关键字 const 的定义 */

#ifndef __cplusplus
#define inline
#endif
/* 如果不是 C++ 环境，则将关键字 inline 定义为空 */

#undef size_t
/* 取消对类型 size_t 的定义 */

#ifndef LIBSSH2_DISABLE_QADRT_EXT
/* 如果未定义宏 LIBSSH2_DISABLE_QADRT_EXT，则进行以下操作 */

#pragma map(inflateInit_, "_libssh2_os400_inflateInit_")
#pragma map(deflateInit_, "_libssh2_os400_deflateInit_")
/* 使用 pragma 指令将 zlib 的函数映射到 ASCII 版本的函数名上 */

#endif

#endif
/* 结束条件编译指令块 */

/* vim: set expandtab ts=4 sw=4: */
/* 设置 vim 编辑器的选项，包括自动缩进和制表符宽度 */
```