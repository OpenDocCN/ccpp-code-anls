# `nmap\libdnet-stripped\src\ip.c`

```cpp
/*
 * ip.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <netinet/in.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

// 定义 IP 处理器结构体
struct ip_handle {
    int    fd;
};

// 打开一个 IP 处理器
ip_t *
ip_open(void)
{
    ip_t *i;
    int n;
    socklen_t len;

    // 分配内存给 IP 处理器
    if ((i = calloc(1, sizeof(*i))) == NULL)
        return (NULL);

    // 创建原始套接字
    if ((i->fd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW)) < 0)
        return (ip_close(i));
#ifdef IP_HDRINCL
    n = 1;
    // 设置套接字选项，包括 IP 头部
    if (setsockopt(i->fd, IPPROTO_IP, IP_HDRINCL, &n, sizeof(n)) < 0)
        return (ip_close(i));
#endif
#ifdef SO_SNDBUF
    len = sizeof(n);
    // 获取套接字发送缓冲区大小
    if (getsockopt(i->fd, SOL_SOCKET, SO_SNDBUF, &n, &len) < 0)
        return (ip_close(i));

    // 调整套接字发送缓冲区大小
    for (n += 128; n < 1048576; n += 128) {
        if (setsockopt(i->fd, SOL_SOCKET, SO_SNDBUF, &n, len) < 0) {
            if (errno == ENOBUFS)
                break;
            return (ip_close(i));
        }
    }
#endif
#ifdef SO_BROADCAST
    n = 1;
    // 设置套接字选项，允许广播
    if (setsockopt(i->fd, SOL_SOCKET, SO_BROADCAST, &n, sizeof(n)) < 0)
        return (ip_close(i));
#endif
    return (i);
}

// 发送 IP 数据包
ssize_t
ip_send(ip_t *i, const void *buf, size_t len)
{
    struct ip_hdr *ip;
    struct sockaddr_in sin;

    ip = (struct ip_hdr *)buf;

    memset(&sin, 0, sizeof(sin));
#ifdef HAVE_SOCKADDR_SA_LEN       
    sin.sin_len = sizeof(sin);
#endif
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = ip->ip_dst;
    
#ifdef HAVE_RAWIP_HOST_OFFLEN
    ip->ip_len = ntohs(ip->ip_len);
    ip->ip_off = ntohs(ip->ip_off);

    len = sendto(i->fd, buf, len, 0,
        (struct sockaddr *)&sin, sizeof(sin));
    
    ip->ip_len = htons(ip->ip_len);
    ip->ip_off = htons(ip->ip_off);

    return (len);
#else
    return (sendto(i->fd, buf, len, 0,
        (struct sockaddr *)&sin, sizeof(sin)));
#endif
}

// 关闭 IP 处理器
ip_t *
ip_close(ip_t *i)
{
    # 检查指针 i 是否为空
    if (i != NULL) {
        # 如果 i 的文件描述符大于等于 0，则关闭文件
        if (i->fd >= 0)
            close(i->fd);
        # 释放指针 i 指向的内存空间
        free(i);
    }
    # 返回空指针
    return (NULL);
# 闭合前面的函数定义
```