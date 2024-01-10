# `nmap\nbase\getaddrinfo.c`

```
/* $Id$ */

#include "nbase.h"

#include <stdio.h>
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>
#endif
#if HAVE_ARPA_INET_H
#include <arpa/inet.h>
#endif
#if HAVE_NETDB_H
#include <netdb.h>
#endif
#include <assert.h>


#if !defined(HAVE_GAI_STRERROR) || defined(__MINGW32__)
#ifdef __MINGW32__
#undef gai_strerror
#endif

// 自定义 gai_strerror 函数，根据错误码返回对应的错误信息
const char *gai_strerror(int errcode) {
  static char customerr[64];
  switch (errcode) {
  case EAI_FAMILY:
    return "ai_family not supported";
  case EAI_NODATA:
    return "no address associated with hostname";
  case EAI_NONAME:
    return "hostname nor servname provided, or not known";
  default:
    Snprintf(customerr, sizeof(customerr), "unknown error (%d)", errcode);
    return "unknown error.";
  }
  return NULL; /* unreached */
}
#endif

#ifdef __MINGW32__
// 在 Windows 下定义 gai_strerrorA 函数，调用 gai_strerror 函数
char* WSAAPI gai_strerrorA (int errcode)
{
  return gai_strerror(errcode);
}
#endif

#ifndef HAVE_GETADDRINFO
// 自定义 freeaddrinfo 函数，释放 addrinfo 结构
void freeaddrinfo(struct addrinfo *res) {
  struct addrinfo *next;

  do {
    next = res->ai_next;
    free(res);
  } while ((res = next) != NULL);
}

/* Allocates and initializes a new AI structure with the port and IPv4
   address specified in network byte order */
// 分配并初始化一个新的 addrinfo 结构，使用指定的端口和 IPv4 地址（网络字节顺序）
static struct addrinfo *new_ai(unsigned short portno, u32 addr)
{
        struct addrinfo *ai;

        ai = (struct addrinfo *) safe_malloc(sizeof(struct addrinfo) + sizeof(struct sockaddr_in));

        memset(ai, 0, sizeof(struct addrinfo) + sizeof(struct sockaddr_in));

        ai->ai_family = AF_INET;
        ai->ai_addrlen = sizeof(struct sockaddr_in);
        ai->ai_addr = (struct sockaddr *)(ai + 1);
        ai->ai_addr->sa_family = AF_INET;
#if HAVE_SOCKADDR_SA_LEN
        ai->ai_addr->sa_len = ai->ai_addrlen;
#endif
        ((struct sockaddr_in *)(ai)->ai_addr)->sin_port = portno;
        ((struct sockaddr_in *)(ai)->ai_addr)->sin_addr.s_addr = addr;

        return(ai);
}
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints, struct addrinfo **res) {

  struct addrinfo *cur, *prev = NULL;  // 声明 addrinfo 结构体指针 cur 和 prev，初始化 prev 为 NULL
  struct hostent *he;  // 声明 hostent 结构体指针 he
  struct in_addr ip;  // 声明 in_addr 结构体变量 ip
  unsigned short portno;  // 声明无符号短整型变量 portno
  int i;  // 声明整型变量 i

  if (service)  // 如果 service 不为空
    portno = htons(atoi(service));  // 将 service 转换为整数并转换为网络字节顺序，赋值给 portno
  else
    portno = 0;  // 否则将 portno 置为 0

  if (hints && hints->ai_flags & AI_PASSIVE) {  // 如果 hints 不为空且 ai_flags 包含 AI_PASSIVE 标志
    *res = new_ai(portno, htonl(0x00000000));  // 调用 new_ai 函数创建 addrinfo 结构体，赋值给 res
    return 0;  // 返回 0
  }

  if (!node) {  // 如果 node 为空
    *res = new_ai(portno, htonl(0x7f000001));  // 调用 new_ai 函数创建 addrinfo 结构体，赋值给 res
    return 0;  // 返回 0
  }

  if (inet_pton(AF_INET, node, &ip)) {  // 如果将 node 转换为 AF_INET 地址成功
    *res = new_ai(portno, ip.s_addr);  // 调用 new_ai 函数创建 addrinfo 结构体，赋值给 res
    return 0;  // 返回 0
  }

  he = gethostbyname(node);  // 获取 node 对应的主机信息，赋值给 he
  if (he && he->h_addr_list[0]) {  // 如果获取主机信息成功且主机地址列表不为空
    for (i = 0; he->h_addr_list[i]; i++) {  // 遍历主机地址列表
      cur = new_ai(portno, ((struct in_addr *)he->h_addr_list[i])->s_addr);  // 调用 new_ai 函数创建 addrinfo 结构体，赋值给 cur

      if (prev)  // 如果 prev 不为空
        prev->ai_next = cur;  // 将 cur 赋值给 prev 的下一个指针
      else
        *res = cur;  // 否则将 cur 赋值给 res

      prev = cur;  // 将 cur 赋值给 prev
    }
    return 0;  // 返回 0
  }

  return EAI_NODATA;  // 返回 EAI_NODATA
}
#endif /* HAVE_GETADDRINFO */
```