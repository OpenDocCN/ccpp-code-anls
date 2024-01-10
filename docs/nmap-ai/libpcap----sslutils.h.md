# `nmap\libpcap\sslutils.h`

```
/*
 * 版权声明
 * 版权所有 (c) 2002 - 2003
 * NetGroup, 意大利都灵理工大学
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 *
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 不能使用意大利都灵理工大学的名称或其贡献者的名称，来认可或推广从本软件衍生的产品，除非事先得到特定的书面许可。
 *
 * 本软件由版权所有者和贡献者按原样提供，任何明示或暗示的保证，包括但不限于对适销性和特定用途的适用性的暗示保证，都是被拒绝的。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润的损失、业务中断），都不会由版权所有者或贡献者承担责任，即使已被告知可能发生这种损害的可能性。
 *
 */

#ifndef __SSLUTILS_H__
#define __SSLUTILS_H__

#ifdef HAVE_OPENSSL
#include "pcap/socket.h"  // 用于 SOCKET
#include <openssl/ssl.h>
#include <openssl/err.h>

/*
 * 实用函数
 */

// 设置证书文件路径
void ssl_set_certfile(const char *certfile);
// 设置密钥文件路径
void ssl_set_keyfile(const char *keyfile);
// 初始化 SSL，仅需执行一次
int ssl_init_once(int is_server, int enable_compression, char *errbuf, size_t errbuflen);
// 定义一个函数，用于创建或接受 SSL 连接
SSL *ssl_promotion(int is_server, SOCKET s, char *errbuf, size_t errbuflen);

// 定义一个函数，用于结束 SSL 连接
void ssl_finish(SSL *ssl);

// 定义一个函数，用于发送数据到 SSL 连接
int ssl_send(SSL *, char const *buffer, int size, char *errbuf, size_t errbuflen);

// 定义一个函数，用于从 SSL 连接接收数据
int ssl_recv(SSL *, char *buffer, int size, char *errbuf, size_t errbuflen);

// 如果没有使用 SSL，定义 _U_NOSSL_
#define _U_NOSSL_

#else   // HAVE_OPENSSL

// 如果使用了 OpenSSL，将 SSL 定义为 void const
#define SSL void const

// 如果没有使用 SSL，定义 _U_NOSSL_ 为 _U_
#define _U_NOSSL_    _U_

#endif  // HAVE_OPENSSL

#endif  // __SSLUTILS_H__
```