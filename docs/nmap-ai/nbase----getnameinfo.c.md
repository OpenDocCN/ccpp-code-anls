# `nmap\nbase\getnameinfo.c`

```cpp
/* $Id$ */
#include "nbase.h"

#if HAVE_NETDB_H
#include <netdb.h>
#endif
#include <assert.h>
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

// 获取地址信息的函数
int getnameinfo(const struct sockaddr *sa, size_t salen,
                char *host, size_t hostlen,
                char *serv, size_t servlen, int flags) {

  // 将传入的地址结构转换为 sockaddr_in 结构
  struct sockaddr_in *sin = (struct sockaddr_in *)sa;
  struct hostent *he;

  // 检查地址族和地址长度是否符合要求
  if (sin->sin_family != AF_INET || salen != sizeof(struct sockaddr_in))
    return EAI_FAMILY;

  // 如果传入的服务名不为空，则将端口号转换为字符串
  if (serv != NULL) {
    Snprintf(serv, servlen, "%d", ntohs(sin->sin_port));
    return 0;
  }

  // 如果传入的主机名不为空
  if (host) {
    // 如果设置了 NI_NUMERICHOST 标志，则将 IP 地址转换为字符串
    if (flags & NI_NUMERICHOST) {
      Strncpy(host, inet_ntoa(sin->sin_addr), hostlen);
      return 0;
    } else {
      // 否则根据 IP 地址获取主机名
      he = gethostbyaddr((char *)&sin->sin_addr, sizeof(struct in_addr),
                         AF_INET);
      // 如果获取不到主机名
      if (he == NULL) {
        // 如果设置了 NI_NAMEREQD 标志，则返回无法解析主机名的错误
        if (flags & NI_NAMEREQD)
          return EAI_NONAME;
        // 否则将 IP 地址转换为字符串
        Strncpy(host, inet_ntoa(sin->sin_addr), hostlen);
        return 0;
      }
      // 获取到主机名，则将主机名复制到 host 中
      assert(he->h_name);
      Strncpy(host, he->h_name, hostlen);
      return 0;
    }
  }
  return 0;
}
```