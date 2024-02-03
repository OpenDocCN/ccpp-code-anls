# `nmap\nsock\src\netutils.c`

```cpp
/* $Id$ */

#include "netutils.h"  // 包含自定义的网络工具头文件
#include "error.h"  // 包含自定义的错误处理头文件

#if WIN32
#include "Winsock2.h"  // 在 Windows 平台下包含 Winsock2 头文件
#endif

#if HAVE_SYS_TIME_H
#include <sys/time.h>  // 包含系统时间头文件
#endif
#if HAVE_SYS_RESOURCE_H
#include <sys/resource.h>  // 包含系统资源头文件
#endif
#if HAVE_UNISTD_H
#include <unistd.h>  // 包含系统标准头文件
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>  // 包含网络地址族头文件
#endif
#if HAVE_SYS_TYPES_H
#include <sys/types.h>  // 包含系统类型头文件
#endif
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>  // 包含套接字头文件
#endif
#if HAVE_FCNTL_H
#include <fcntl.h>  // 包含文件控制头文件
#endif
#if HAVE_SYS_UN_H
#include <sys/un.h>  // 包含 UNIX 域套接字头文件
#endif

static int netutils_debugging = 0;  // 定义并初始化网络工具调试标志

/* 最大化允许该进程的文件描述符数（包括套接字），并返回该最大值
 * 注意 -- 你最好不要实际打开这么多文件 -- 标准输入、标准输出、库使用的其他文件等都计入此限制。留一点余地 */
rlim_t maximize_fdlimit(void) {

#ifndef WIN32
  struct rlimit r;  // 定义系统资源限制结构体
  static int maxfds_set = 0;  // 定义并初始化最大文件描述符设置标志
  static rlim_t maxfds = 0;  // 定义并初始化最大文件描述符数

  if (maxfds_set)
    return maxfds;  // 如果最大文件描述符已设置，则直接返回

#ifndef RLIMIT_NOFILE
 #ifdef RLIMIT_OFILE
  #define RLIMIT_NOFILE RLIMIT_OFILE
 #else
  #error Neither RLIMIT_NOFILE nor RLIMIT_OFILE defined
 #endif
#endif
  if (!getrlimit(RLIMIT_NOFILE, &r)) {  // 获取文件描述符限制
    maxfds = r.rlim_cur;  // 获取当前文件描述符限制
    r.rlim_cur = r.rlim_max;  // 将当前文件描述符限制设置为最大限制
    if (!setrlimit(RLIMIT_NOFILE, &r))  // 设置文件描述符限制
      if (netutils_debugging)  // 如果网络工具调试标志为真
        perror("setrlimit RLIMIT_NOFILE failed");  // 输出错误信息

    if (!getrlimit(RLIMIT_NOFILE, &r)) {  // 再次获取文件描述符限制
      maxfds = r.rlim_cur;  // 获取当前文件描述符限制
    }
    maxfds_set = 1;  // 设置最大文件描述符已设置标志为真
    return maxfds;  // 返回最大文件描述符数
  }
#endif /* !WIN32 */
  return 0;  // 返回 0
}

#if HAVE_SYS_UN_H
  #define PEER_STR_LEN sizeof(((struct sockaddr_un *) 0)->sun_path)  // 定义 UNIX 域套接字路径长度
#else
  #define PEER_STR_LEN sizeof("[ffff:ffff:ffff:ffff:ffff:ffff:255.255.255.255]:xxxxx")  // 定义 IP 地址字符串长度
#endif

#if HAVE_SYS_UN_H
/* 获取 UNIX 域套接字路径，如果地址族不是 AF_UNIX，则返回空字符串 */
const char *get_unixsock_path(const struct sockaddr_storage *addr) {
  struct sockaddr_un *su = (struct sockaddr_un *)addr;  // 将地址结构转换为 UNIX 域套接字结构

  if (!addr || addr->ss_family != AF_UNIX)  // 如果地址为空或地址族不是 AF_UNIX
    # 返回一个空字符串
    return "";

  # 返回指向sun_path的常量字符指针
  return (const char *)su->sun_path;
/* 结束条件判断，检查是否定义了#endif */
}
#endif

/* 获取端口号 */
static int get_port(const struct sockaddr_storage *ss) {
  /* 如果地址族是 AF_INET，则返回端口号 */
  if (ss->ss_family == AF_INET)
    return ntohs(((struct sockaddr_in *) ss)->sin_port);
  /* 如果支持 IPv6 并且地址族是 AF_INET6，则返回端口号 */
#if HAVE_IPV6
  else if (ss->ss_family == AF_INET6)
    return ntohs(((struct sockaddr_in6 *) ss)->sin6_port);
#endif
  /* 其他情况返回 -1 */
  return -1;
}

/* 获取地址字符串 */
static char *get_addr_string(const struct sockaddr_storage *ss, size_t sslen) {
  static char buffer[PEER_STR_LEN];
  /* 如果支持 UNIX 域套接字并且地址族是 AF_UNIX，则返回 UNIX 套接字路径 */
#if HAVE_SYS_UN_H
  if (ss->ss_family == AF_UNIX) {
    sprintf(buffer, "%s", get_unixsock_path(ss));
    return buffer;
  }
#endif
  /* 否则返回 "<address>:<port>" 形式的字符串 */
  sprintf(buffer, "%s:%d", inet_ntop_ez(ss, sslen), get_port(ss));
  return buffer;
}

/* 获取对等方/主机地址字符串 */
/* 如果支持 UNIX 域套接字，函数返回包含 UNIX 套接字路径的字符串，如果地址族是 AF_UNIX，
   否则返回包含 "<address>:<port>" 的字符串。 */
char *get_peeraddr_string(const struct niod *iod) {
  /* 如果对等方地址长度大于 0，则返回地址字符串 */
  if (iod->peerlen > 0)
    return get_addr_string(&iod->peer, iod->peerlen);
  /* 否则返回 "peer unspecified" */
  else
    return "peer unspecified";
}

/* 获取本地绑定地址字符串 */
char *get_localaddr_string(const struct niod *iod) {
  /* 返回地址字符串 */
  return get_addr_string(&iod->local, iod->locallen);
}
```