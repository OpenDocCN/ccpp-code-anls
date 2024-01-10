# `nmap\libssh2\os400\os400sys.c`

```
/*
 * 版权声明，版权所有
 * 允许在源代码和二进制形式下重新分发和使用，无论是否经过修改，前提是满足以下条件：
 *   1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。无论在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、附带、特殊、惩罚性或后果性损害，包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，版权所有者或贡献者均不承担任何责任。
 */

/* OS/400的额外支持 */

// 禁用QADRT扩展
#define LIBSSH2_DISABLE_QADRT_EXT

// 引入libssh2_priv.h头文件
#include "libssh2_priv.h"

// 引入系统类型头文件
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/un.h>

// 引入标准头文件
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>
#include <stdarg.h>
#include <string.h>
#include <alloca.h>
#include <netdb.h>
#include <qadrt.h>
#include <errno.h>
/* 包含网络地址转换和压缩解压相关的头文件 */
#include <netinet/in.h>
#include <arpa/inet.h>

/* 如果支持 ZLIB，则包含 ZLIB 头文件 */
#ifdef LIBSSH2_HAVE_ZLIB
# include <zlib.h>
#endif

/**
***     QADRT OS/400 ASCII runtime defines only the most used procedures, but
***             a lot of them are not supported. This module implements
***             ASCII wrappers for those that are used by libssh2, but not
***             defined by QADRT.
**/

/* 恢复 EBCDIC 编码 */
#pragma convert(37)

/* 定义将套接字地址转换为指定格式的函数 */
static int
convert_sockaddr(struct sockaddr_storage * dstaddr,
                                const struct sockaddr * srcaddr, int srclen)

{
  const struct sockaddr_un * srcu;
  struct sockaddr_un * dstu;
  unsigned int i;
  unsigned int dstsize;

  /* 如果源地址为空，或者长度不符合要求，则返回错误 */
  if(!srcaddr || srclen < offsetof(struct sockaddr, sa_family) +
     sizeof srcaddr->sa_family || srclen > sizeof *dstaddr) {
    errno = EINVAL;
    return -1;
    }

  /* 将源地址的内容复制到目标地址 */
  memcpy((char *) dstaddr, (char *) srcaddr, srclen);

  /* 根据不同的地址类型进行处理 */
  switch (srcaddr->sa_family) {

  case AF_UNIX:
    srcu = (const struct sockaddr_un *) srcaddr;
    dstu = (struct sockaddr_un *) dstaddr;
    dstsize = sizeof *dstaddr - offsetof(struct sockaddr_un, sun_path);
    srclen -= offsetof(struct sockaddr_un, sun_path);
    i = QadrtConvertA2E(dstu->sun_path, srcu->sun_path, dstsize - 1, srclen);
    dstu->sun_path[i] = '\0';
    i += offsetof(struct sockaddr_un, sun_path);
    srclen = i;
    }

  return srclen;
}

/* 定义连接函数，将地址转换后再连接 */
int
_libssh2_os400_connect(int sd, struct sockaddr * destaddr, int addrlen)

{
  int i;
  struct sockaddr_storage laddr;

  i = convert_sockaddr(&laddr, destaddr, addrlen);

  if(i < 0)
    return -1;

  return connect(sd, (struct sockaddr *) &laddr, i);
}

/* 定义格式化输出函数，使用指定格式和参数列表 */
int
_libssh2_os400_vsnprintf(char *dst, size_t len, const char *fmt, va_list args)
{
    size_t l = 4096;
    int i;
    char *buf;

    /* 如果目标地址为空或长度为0，则返回错误 */
    if (!dst || !len) {
        errno = EINVAL;
        return -1;
    }

    /* 如果缓冲区长度小于目标长度，则使用目标长度 */
    if (l < len)
        l = len;

    /* 分配临时缓冲区 */
    buf = alloca(l);
}
    # 如果缓冲区为空，则设置错误码为内存不足，并返回-1
    if (!buf) {
        errno = ENOMEM;
        return -1;
    }

    # 将格式化的字符串写入缓冲区
    i = vsprintf(buf, fmt, args);

    # 如果写入失败，则返回错误码
    if (i < 0)
        return i;

    # 如果剩余长度大于写入长度，则将剩余长度设置为写入长度
    if (--len > i)
        len = i;

    # 如果剩余长度大于0，则将缓冲区的内容复制到目标地址
    if (len)
        memcpy(dst, buf, len);

    # 在目标地址的末尾添加字符串结束符
    dst[len] = '\0';
    # 返回写入的长度
    return len;
/* VARARGS3 */
// 定义一个可变参数的函数_libssh2_os400_snprintf，用于格式化字符串
int
_libssh2_os400_snprintf(char *dst, size_t len, const char *fmt, ...)
{
    va_list args; // 定义一个可变参数列表
    int ret; // 定义返回值变量

    va_start(args, fmt); // 初始化可变参数列表
    ret = _libssh2_os400_vsnprintf(dst, len, fmt, args); // 调用_vsnprintf函数进行格式化字符串
    va_end(args); // 结束可变参数列表
    return ret; // 返回格式化后的字符串长度
}

#ifdef LIBSSH2_HAVE_ZLIB
// 如果支持zlib，则定义以下两个函数

// 初始化zlib的解压缩流
int
_libssh2_os400_inflateInit_(z_streamp strm,
                            const char *version, int stream_size)
{
    char *ebcversion; // 定义一个字符指针
    int i; // 定义一个整型变量

    if (!version) // 如果版本号为空
        return Z_VERSION_ERROR; // 返回版本错误
    i = strlen(version); // 获取版本号的长度
    ebcversion = alloca(i + 1); // 分配内存空间
    if (!ebcversion) // 如果内存分配失败
        return Z_VERSION_ERROR; // 返回版本错误
    i = QadrtConvertA2E(ebcversion, version, i, i - 1); // 转换版本号的编码格式
    ebcversion[i] = '\0'; // 在转换后的版本号末尾添加结束符
    return inflateInit_(strm, ebcversion, stream_size); // 调用zlib的inflateInit_函数
}

// 初始化zlib的压缩流
int
_libssh2_os400_deflateInit_(z_streamp strm, int level,
                            const char *version, int stream_size)
{
    char *ebcversion; // 定义一个字符指针
    int i; // 定义一个整型变量

    if (!version) // 如果版本号为空
        return Z_VERSION_ERROR; // 返回版本错误
    i = strlen(version); // 获取版本号的长度
    ebcversion = alloca(i + 1); // 分配内存空间
    if (!ebcversion) // 如果内存分配失败
        return Z_VERSION_ERROR; // 返回版本错误
    i = QadrtConvertA2E(ebcversion, version, i, i - 1); // 转换版本号的编码格式
    ebcversion[i] = '\0'; // 在转换后的版本号末尾添加结束符
    return deflateInit_(strm, level, ebcversion, stream_size); // 调用zlib的deflateInit_函数
}
#endif
```