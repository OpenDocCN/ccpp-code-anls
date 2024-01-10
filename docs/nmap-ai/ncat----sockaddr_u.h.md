# `nmap\ncat\sockaddr_u.h`

```
/* $Id:$ */

#include "ncat_config.h"

#ifndef SOCKADDR_U_H_
#define SOCKADDR_U_H_

#ifdef WIN32
# include <ws2def.h>
#endif
#if HAVE_NETINET_IN_H
# include <netinet/in.h>
#endif
#if HAVE_SYS_SOCKET_H
# include <sys/socket.h>
#endif
#if HAVE_SYS_UN_H
# include <sys/un.h>
#endif
#if HAVE_LINUX_VM_SOCKETS_H
#include <linux/vm_sockets.h>
#endif

// 定义一个联合体，用于存储不同类型的套接字地址
union sockaddr_u {
    struct sockaddr_storage storage;  // 通用的套接字地址结构
#ifdef HAVE_SYS_UN_H
    struct sockaddr_un un;  // UNIX 域套接字地址结构
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
    struct sockaddr_vm vm;  // Linux 虚拟套接字地址结构
#endif
    struct sockaddr_in in;  // IPv4 套接字地址结构
    struct sockaddr_in6 in6;  // IPv6 套接字地址结构
    struct sockaddr sockaddr;  // 通用的套接字地址结构
};

// 获取套接字地址的长度
static inline socklen_t get_socklen(const union sockaddr_u *s)
{
    switch(s->storage.ss_family) {
#ifdef HAVE_SYS_UN_H
      case AF_UNIX:
        return SUN_LEN(&s->un);  // 返回 UNIX 域套接字地址的长度
        break;
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
      case AF_VSOCK:
        return sizeof(struct sockaddr_vm);  // 返回 Linux 虚拟套接字地址的长度
        break;
#endif
#ifdef HAVE_SOCKADDR_SA_LEN
      default:
        return s->sockaddr.sa_len;  // 返回通用套接字地址的长度
        break;
#else
      case AF_INET:
        return sizeof(struct sockaddr_in);  // 返回 IPv4 套接字地址的长度
        break;
#ifdef AF_INET6
      case AF_INET6:
        return sizeof(struct sockaddr_in6);  // 返回 IPv6 套接字地址的长度
        break;
#endif
      default:
        return sizeof(union sockaddr_u);  // 返回联合体的长度
        break;
#endif
    }
    return 0;  // 默认返回 0
}
#endif
```