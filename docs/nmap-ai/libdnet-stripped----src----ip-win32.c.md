# `nmap\libdnet-stripped\src\ip-win32.c`

```cpp
/*
 * ip-win32.c
 *
 * 版权所有 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-win32.c 547 2005-01-25 21:30:40Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <ws2tcpip.h>

#include <errno.h>
#include <stdlib.h>

#include "dnet.h"

// 定义 IP 句柄结构
struct ip_handle {
    WSADATA            wsdata;  // Windows 套接字信息
    SOCKET            fd;       // 套接字文件描述符
    struct sockaddr_in    sin;  // IPv4 地址结构
};

// 打开 IP 句柄
ip_t *
ip_open(void)
{
    BOOL on;  // 布尔类型
    ip_t *ip;  // IP 句柄指针

    // 分配内存给 IP 句柄
    if ((ip = calloc(1, sizeof(*ip))) != NULL) {
        // 初始化 Windows 套接字库
        if (WSAStartup(MAKEWORD(2, 2), &ip->wsdata) != 0) {
            free(ip);
            return (NULL);
        }
        // 创建原始套接字
        if ((ip->fd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW)) ==
            INVALID_SOCKET)
            return (ip_close(ip));
        
        on = TRUE;
        // 设置套接字选项，包括 IP 头部信息
        if (setsockopt(ip->fd, IPPROTO_IP, IP_HDRINCL,
            (const char *)&on, sizeof(on)) == SOCKET_ERROR) {
            SetLastError(ERROR_NETWORK_ACCESS_DENIED);
            return (ip_close(ip));
        }
        // 设置 IPv4 地址结构
        ip->sin.sin_family = AF_INET;
        ip->sin.sin_port = htons(666);
    }
    return (ip);
}

// 发送 IP 数据包
ssize_t
ip_send(ip_t *ip, const void *buf, size_t len)
{
    struct ip_hdr *hdr = (struct ip_hdr *)buf;  // IP 头部结构体指针
    
    // 设置目标 IP 地址
    ip->sin.sin_addr.s_addr = hdr->ip_src;
    
    // 发送数据包
    if ((len = sendto(ip->fd, (const char *)buf, (int)len, 0,
        (struct sockaddr *)&ip->sin, sizeof(ip->sin))) != SOCKET_ERROR)
        return (ssize_t)(len);
    
    return (-1);
}

// 关闭 IP 句柄
ip_t *
ip_close(ip_t *ip)
{
    if (ip != NULL) {
        WSACleanup();  // 清理 Windows 套接字库
        if (ip->fd != INVALID_SOCKET)
            closesocket(ip->fd);  // 关闭套接字
        free(ip);  // 释放内存
    }
    return (NULL);
}
```