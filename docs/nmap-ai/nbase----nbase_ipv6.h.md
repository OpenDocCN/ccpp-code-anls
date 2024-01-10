# `nmap\nbase\nbase_ipv6.h`

```
#ifndef NBASE_IPV6_H
#define NBASE_IPV6_H

#ifdef __amigaos__
#ifndef _NMAP_AMIGAOS_H_
#include "../nmap_amigaos.h"
#endif
#endif

#if HAVE_SYS_TYPES_H
#include <sys/types.h>
#endif
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>
#endif
#if HAVE_ARPA_INET_H
#include <arpa/inet.h>
#endif

#ifndef HAVE_AF_INET6
#define AF_INET6 10
#define PF_INET6 10
#endif /* HAVE_AF_INET6 */
#ifndef HAVE_INET_PTON
/* int
 * inet_pton(af, src, dst)
 *      convert from presentation format (which usually means ASCII printable)
 *      to network format (which is usually some kind of binary format).
 * return:
 *      1 if the address was valid for the specified address family
 *      0 if the address wasn't valid (`dst' is untouched in this case)
 *      -1 if some other error occurred (`dst' is untouched in this case, too)
 * author:
 *      Paul Vixie, 1996.
 */
int inet_pton(int af, const char *src, void *dst);
#endif /* HAVE_INET_PTON */

#ifndef HAVE_INET_NTOP
/* char *
 * inet_ntop(af, src, dst, size)
 *    convert a network format address to presentation format.
 * return:
 *    pointer to presentation format address (`dst'), or NULL (see errno).
 * author:
 *    Paul Vixie, 1996.
 */
const char *inet_ntop(int af, const void *src, char *dst, size_t size);
#endif /* HAVE_INET_NTOP */

#ifndef HAVE_SOCKADDR_STORAGE
struct sockaddr_storage {
  u16 ss_family;
  u16 __align_to_64[3];
  u64 __padding[16];
};
#endif /* SOCKADDR_STORAGE */

/* Compares two sockaddr_storage structures with a return value like strcmp.
   First the address families are compared, then the addresses if the families
   are equal. The structures must be real full-length sockaddr_storage
   structures, not something shorter like sockaddr_in. */
int sockaddr_storage_cmp(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b);

/* Does sockaddr_storage_cmp(a, b) == 0 for you. */
/* 比较两个 sockaddr_storage 结构体是否相等，返回整型值 */
int sockaddr_storage_equal(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b);

/* 这个函数是 inet_ntop 的简化版本，因为你不需要传递目标缓冲区。
   相反，它返回一个静态缓冲区，你可以在函数再次被调用之前（由同一线程或进程中的另一个线程）一直使用。
   如果出现奇怪的错误（比如 sslen 太短），则返回 NULL。 */
const char *inet_ntop_ez(const struct sockaddr_storage *ss, size_t sslen);

#if !HAVE_GETNAMEINFO || !HAVE_GETADDRINFO
#if !defined(EAI_MEMORY)
/* 定义一系列错误码，用于处理 getnameinfo 和 getaddrinfo 的错误情况 */
#define EAI_ADDRFAMILY   1      /* 主机名的地址族不受支持 */
#define EAI_AGAIN        2      /* 名称解析临时失败 */
#define EAI_BADFLAGS     3      /* ai_flags 的值无效 */
#define EAI_FAIL         4      /* 名称解析不可恢复的失败 */
#define EAI_FAMILY       5      /* ai_family 不受支持 */
#define EAI_MEMORY       6      /* 内存分配失败 */
#define EAI_NODATA       7      /* 主机名没有关联的地址 */
#define EAI_NONAME       8      /* 未提供主机名或服务名，或者未知 */
#define EAI_SERVICE      9      /* ai_socktype 不受支持的服务名 */
#define EAI_SOCKTYPE    10      /* ai_socktype 不受支持 */
#define EAI_SYSTEM      11      /* 在 errno 中返回系统错误 */
#define EAI_BADHINTS    12      /* 无效的提示 */
#define EAI_PROTOCOL    13      /* 协议不受支持 */
#define EAI_MAX         14      /* 错误码的最大值 */
#endif /* EAI_MEMORY */
#endif /* !HAVE_GETNAMEINFO || !HAVE_GETADDRINFO */

#if !HAVE_GETNAMEINFO
/* 这个替代版本 *不是*一个完整的实现，无论如何 */
/* getnameinfo 的替代版本，定义了一些标志 */
#if !defined(NI_NAMEREQD)
#define NI_NOFQDN 8
#define NI_NUMERICHOST 16
#define NI_NAMEREQD 32
#define NI_NUMERICSERV 64
#define NI_DGRAM 128
#endif

/* 定义了 getnameinfo 函数的替代版本，参数和原函数一致 */
struct sockaddr;
int getnameinfo(const struct sockaddr *sa, size_t salen,
                char *host, size_t hostlen,
                char *serv, size_t servlen, int flags);
#endif /* !HAVE_GETNAMEINFO */

#if !HAVE_GETADDRINFO
/* 如果系统不支持 getaddrinfo 函数，则使用以下结构体和函数作为替代，这不是一个完整的实现 */
struct addrinfo {
  int ai_flags;      /* AI_PASSIVE, AI_CANONNAME, AI_NUMERICHOST */
  int ai_family;    /* PF_xxx */
  int ai_socktype;  /* SOCK_xxx */
  int ai_protocol;  /* 0 or IPPROTO_xxx for IPv4 and IPv6 */
  size_t ai_addrlen;   /* ai_addr 的长度 */
  char *ai_canonname; /* nodename 的规范名称 */
  struct sockaddr  *ai_addr; /* 二进制地址 */
  struct  addrinfo  *ai_next; /* 链表中的下一个结构体 */
};

/* getaddrinfo 的标志 */
#if !defined(AI_PASSIVE) || !defined(AI_CANONNAME) || !defined(AI_NUMERICHOST)
#define AI_PASSIVE 1
#define AI_CANONNAME 2
#define AI_NUMERICHOST 4
#endif

void freeaddrinfo(struct addrinfo *res);
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints, struct addrinfo **res);

#endif /* !HAVE_GETADDRINFO */

#ifndef HAVE_GAI_STRERROR
const char *gai_strerror(int errcode);
#endif

int sockaddr_storage_inet_pton(const char * ip_str, struct sockaddr_storage * addr);
const char *sockaddr_storage_iptop(const struct sockaddr_storage * addr, char * dst);

#endif /* NBASE_IPV6_H */
```