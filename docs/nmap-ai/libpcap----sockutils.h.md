# `nmap\libpcap\sockutils.h`

```
/*
 * 版权声明，版权所有
 * NetGroup, 意大利都灵理工大学，2002-2003年
 * 禁止在未经许可的情况下以源代码或二进制形式重新分发和使用本软件
 *
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 不得使用都灵理工大学或其贡献者的名称来认可或推广从本软件派生的产品，除非事先得到特定的书面许可
 *
 * 本软件由版权所有者和贡献者提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他原因），版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）负责，即使已被告知可能发生此类损害。
 */
#ifndef __SOCKUTILS_H__
#define __SOCKUTILS_H__

#ifdef _MSC_VER
#pragma once
#endif

#include <stdarg.h>    /* 声明可变参数函数 */

#include "pcap/funcattrs.h"

#include "pcap/socket.h"
#ifndef _WIN32
  /* UN*X */
  #include <unistd.h>    /* close() */

  /*!
   * \brief In Winsock, the close() call cannot be used on a socket;
   * closesocket() must be used.
   * We define closesocket() to be a wrapper around close() on UN*X,
   * so that it can be used on both platforms.
   */
  #define closesocket(a) close(a)
#endif

#include "sslutils.h"  // for SSL type, whatever that turns out to be

/*
 * MingW headers include this definition, but only for Windows XP and above.
 * MSDN states that this function is available for most versions on Windows.
 */
#if ((defined(__MINGW32__)) && (_WIN32_WINNT < 0x0501))
int WSAAPI getnameinfo(const struct sockaddr*,socklen_t,char*,DWORD,
    char*,DWORD,int);
#endif

/*
 * \defgroup SockUtils Cross-platform socket utilities (IPv4-IPv6)
 */

/*
 * \addtogroup SockUtils
 * \{
 */

/*
 * \defgroup ExportedStruct Exported Structures and Definitions
 */

/*
 * \addtogroup ExportedStruct
 * \{
 */

/****************************************************
 *                                                  *
 * Exported functions / definitions                 *
 *                                                  *
 ****************************************************/

/* 'checkonly' flag, into the rpsock_bufferize() */
#define SOCKBUF_CHECKONLY 1
/* no 'checkonly' flag, into the rpsock_bufferize() */
#define SOCKBUF_BUFFERIZE 0

/* no 'server' flag; it opens a client socket */
#define SOCKOPEN_CLIENT 0
/* 'server' flag; it opens a server socket */
#define SOCKOPEN_SERVER 1

/*
 * Flags for sock_recv().
 */
#define SOCK_RECEIVEALL_NO    0x00000000    /* Don't wait to receive all data */
#define SOCK_RECEIVEALL_YES    0x00000001    /* Wait to receive all data */

#define SOCK_EOF_ISNT_ERROR    0x00000000    /* Return 0 on EOF */
#define SOCK_EOF_IS_ERROR    0x00000002    /* Return an error on EOF */

#define SOCK_MSG_PEEK        0x00000004    /* Return data but leave it in the socket queue */

/*
 * \}
 */

#ifdef __cplusplus
#ifdef

/*
 * \defgroup ExportedFunc Exported Functions
 */

/*
 * \addtogroup ExportedFunc
 * \{
 */

// 初始化套接字库，设置错误缓冲区和长度
int sock_init(char *errbuf, int errbuflen);
// 清理套接字库
void sock_cleanup(void);
// 获取错误码
int sock_geterrcode(void);
// 格式化错误消息
void sock_vfmterrmsg(char *errbuf, size_t errbuflen, int errcode, PCAP_FORMAT_STRING(const char *fmt), va_list ap) PCAP_PRINTFLIKE(4, 0);
// 格式化错误消息
void sock_fmterrmsg(char *errbuf, size_t errbuflen, int errcode, PCAP_FORMAT_STRING(const char *fmt), ...) PCAP_PRINTFLIKE(4, 5);
// 获取错误消息
void sock_geterrmsg(char *errbuf, size_t errbuflen, PCAP_FORMAT_STRING(const char *fmt), ...)  PCAP_PRINTFLIKE(3, 4);
// 初始化地址信息
int sock_initaddress(const char *address, const char *port, struct addrinfo *hints, struct addrinfo **addrinfo, char *errbuf, int errbuflen);
// 接收数据
int sock_recv(SOCKET sock, SSL *, void *buffer, size_t size, int receiveall, char *errbuf, int errbuflen);
// 接收数据报
int sock_recv_dgram(SOCKET sock, SSL *, void *buffer, size_t size, char *errbuf, int errbuflen);
// 打开套接字
SOCKET sock_open(const char *host, struct addrinfo *addrinfo, int server, int nconn, char *errbuf, int errbuflen);
// 关闭套接字
int sock_close(SOCKET sock, char *errbuf, int errbuflen);

// 发送数据
int sock_send(SOCKET sock, SSL *, const char *buffer, size_t size, char *errbuf, int errbuflen);
// 将数据缓冲化
int sock_bufferize(const void *data, int size, char *outbuf, int *offset, int totsize, int checkonly, char *errbuf, int errbuflen);
// 丢弃数据
int sock_discard(SOCKET sock, SSL *, int size, char *errbuf, int errbuflen);
// 检查主机列表
int    sock_check_hostlist(char *hostlist, const char *sep, struct sockaddr_storage *from, char *errbuf, int errbuflen);
// 比较地址
int sock_cmpaddr(struct sockaddr_storage *first, struct sockaddr_storage *second);

// 获取本地信息
int sock_getmyinfo(SOCKET sock, char *address, int addrlen, char *port, int portlen, int flags, char *errbuf, int errbuflen);
// 获取ASCII格式的地址和端口
int sock_getascii_addrport(const struct sockaddr_storage *sockaddr, char *address, int addrlen, char *port, int portlen, int flags, char *errbuf, size_t errbuflen);
# 将套接字绑定到网络地址
int sock_present2network(const char *address, struct sockaddr_storage *sockaddr, int addr_family, char *errbuf, int errbuflen);

#ifdef __cplusplus
}
#endif

/*
 * \}
 */

/*
 * \}
 */

#endif
```