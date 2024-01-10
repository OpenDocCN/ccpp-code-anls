# `nmap\ncat\ncat.h`

```
/* $Id$ */

#ifndef NCAT_H_
#define NCAT_H_

#include "ncat_config.h"

#include <nbase.h>

#include "nsock.h"
#include "util.h"
#include "sys_wrap.h"

#include "ncat_connect.h"
#include "ncat_core.h"
#include "ncat_exec.h"
#include "ncat_listen.h"
#include "ncat_proxy.h"
#include "ncat_ssl.h"

/* Ncat information for output, etc. */
#define NCAT_NAME "Ncat"  // 定义 Ncat 的名称
#define NCAT_URL "https://nmap.org/ncat"  // 定义 Ncat 的官方网址
#define NCAT_VERSION "7.94SVN"  // 定义 Ncat 的版本号

#ifndef __GNUC__
#ifndef __attribute__
#define __attribute__(x)  // 定义属性宏
#endif
#endif

#define SOCKS_BUFF_SIZE 512  // 定义 SOCKS 缓冲区大小

/* structs */

#ifdef WIN32
#pragma pack(1)
#endif
struct socks4_data {
    char version;  // SOCKS4 数据结构的版本号
    char type;  // SOCKS4 数据结构的类型
    unsigned short port;  // SOCKS4 数据结构的端口号
    uint32_t address;  // SOCKS4 数据结构的地址
    char data[SOCKS_BUFF_SIZE]; // this has to be able to hold FQDN and username  // SOCKS4 数据结构的数据
} __attribute__((packed));

struct socks5_connect {
    char ver;  // SOCKS5 连接结构的版本号
    unsigned char nmethods;  // SOCKS5 连接结构的方法数
    char methods[3];  // SOCKS5 连接结构的方法数组
} __attribute__((packed));

struct socks5_auth {
    char ver; // must be always 1  // SOCKS5 认证结构的版本号
    unsigned char data[SOCKS_BUFF_SIZE];  // SOCKS5 认证结构的数据
} __attribute__((packed));

struct socks5_request {
    char ver;  // SOCKS5 请求结构的版本号
    char cmd;  // SOCKS5 请求结构的命令
    char rsv;  // SOCKS5 请求结构的保留字段
    char atyp;  // SOCKS5 请求结构的地址类型
    unsigned char dst[SOCKS_BUFF_SIZE]; // addr/name and port info  // SOCKS5 请求结构的目标地址和端口信息
} __attribute__((packed));
#ifdef WIN32
#pragma pack()
#endif

/* defines */

/* Client-mode timeout for reads, infinite */
#define DEFAULT_READ_TIMEOUT -1  // 客户端模式读取超时，默认为无限

/* Client-mode timeout for writes, in msecs */
#define DEFAULT_WRITE_TIMEOUT 2000  // 客户端模式写入超时，默认为 2000 毫秒

/* Client-mode timeout for connection establishment, in msecs */
#define DEFAULT_CONNECT_TIMEOUT 10000  // 客户端模式连接建立超时，默认为 10000 毫秒

/* The default length of Ncat buffers */
#define DEFAULT_BUF_LEN      (1024)  // Ncat 缓冲区的默认长度
#define DEFAULT_TCP_BUF_LEN  (1024 * 8)  // TCP 缓冲区的默认长度
#define DEFAULT_UDP_BUF_LEN  (1024 * 128)  // UDP 缓冲区的默认长度

/* Default Ncat port */
#define DEFAULT_NCAT_PORT 31337  // 默认 Ncat 端口号

/* Default port for SOCKS4 */
#define DEFAULT_SOCKS4_PORT 1080  // 默认 SOCKS4 端口号

/* Default port for SOCKS5 */
#define DEFAULT_SOCKS5_PORT 1080  // 默认 SOCKS5 端口号
/* 默认端口 Ncat 将连接的 HTTP 代理服务器。当前设置是 squid 的默认值，可能也适用于其他 HTTP 代理。但也可能是 8080、8888 等。 */
#define DEFAULT_PROXY_PORT 3128

/* Listen() 的等待队列长度 */
#define BACKLOG 10

/* Ncat 将接受到一个监听端口的默认最大同时连接数。根据具体需求，可能需要增加或减少这个值。 */
#ifdef WIN32
/* Windows 通常限制为 64 个套接字，因此默认值要略低于这个值。http://www.tangentsoft.net/wskfaq/advanced.html#64sockets */
#define DEFAULT_MAX_CONNS 60
#else
#define DEFAULT_MAX_CONNS 100
#endif

/* SOCKS4 协议响应 */
#define SOCKS4_VERSION          4
#define SOCKS_CONNECT           1
#define SOCKS_BIND              2
#define SOCKS4_CONN_ACC         90 /* woot */
#define SOCKS4_CONN_REF         91
#define SOCKS4_CONN_IDENT       92
#define SOCKS4_CONN_IDENTDIFF   93

/* SOCKS5 协议 */
#define SOCKS5_VERSION          5
#define SOCKS5_AUTH_NONE        0
#define SOCKS5_AUTH_GSSAPI      1
#define SOCKS5_AUTH_USERPASS    2
#define SOCKS5_AUTH_FAILED      255
#define SOCKS5_ATYP_IPv4        1
#define SOCKS5_ATYP_NAME        3
#define SOCKS5_ATYP_IPv6        4

#define SOCKS5_USR_MAXLEN       255
#define SOCKS5_PWD_MAXLEN       255
#define SOCKS5_DST_MAXLEN       255

#if SOCKS_BUFF_SIZE < (1 + SOCKS5_USR_MAXLEN) + (1 + SOCKS5_PWD_MAXLEN)
#error SOCKS_BUFF_SIZE 定义过小，无法处理 SOCKS5 认证
#endif

#if SOCKS_BUFF_SIZE < (1 + SOCKS5_DST_MAXLEN) + 2
#error SOCKS_BUFF_SIZE 定义过小，无法处理 SOCKS5 目标
#endif

/* IPv6 地址的长度 */
#ifndef INET6_ADDRSTRLEN
#define INET6_ADDRSTRLEN 46
#endif

#ifndef IPPROTO_SCTP
#define IPPROTO_SCTP 132
#endif

/* Windows 的虚拟 WNOHANG */
#ifndef WNOHANG
#define WNOHANG 0
#endif

#endif
```