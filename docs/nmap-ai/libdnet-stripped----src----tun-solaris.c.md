# `nmap\libdnet-stripped\src\tun-solaris.c`

```cpp
/*
 * tun-solaris.c
 *
 * Universal TUN/TAP driver
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: tun-solaris.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/sockio.h>

#include <net/if.h>
#include <net/if_tun.h>

#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stropts.h>
#include <unistd.h>

#include "dnet.h"

#define DEV_TUN        "/dev/tun"
#define DEV_IP        "/dev/ip"

struct tun {
    int         fd;
    int         ip_fd;
    int         if_fd;
    char         name[16];
};

// 打开 TUN 设备
tun_t *
tun_open(struct addr *src, struct addr *dst, int mtu)
{
    tun_t *tun;
    char cmd[512];
    int ppa;

    if ((tun = calloc(1, sizeof(*tun))) == NULL)
        return (NULL);

    tun->fd = tun->ip_fd = tun->if_fd = -1;
    
    // 打开 TUN 设备
    if ((tun->fd = open(DEV_TUN, O_RDWR, 0)) < 0)
        return (tun_close(tun));

    // 打开 IP 设备
    if ((tun->ip_fd = open(DEV_IP, O_RDWR, 0)) < 0)
        return (tun_close(tun));
    
    // 创建新的 PPA
    if ((ppa = ioctl(tun->fd, TUNNEWPPA, ppa)) < 0)
        return (tun_close(tun));

    // 打开 TUN 设备
    if ((tun->if_fd = open(DEV_TUN, O_RDWR, 0)) < 0)
        return (tun_close(tun));

    // 将 IP 协议推送到 TUN 设备
    if (ioctl(tun->if_fd, I_PUSH, "ip") < 0)
        return (tun_close(tun));
    
    // 选择指定的 PPA
    if (ioctl(tun->if_fd, IF_UNITSEL, (char *)&ppa) < 0)
        return (tun_close(tun));

    // 在 IP 设备上创建链接到 TUN 设备
    if (ioctl(tun->ip_fd, I_LINK, tun->if_fd) < 0)
        return (tun_close(tun));

    // 根据 PPA 创建 TUN 设备名称
    snprintf(tun->name, sizeof(tun->name), "tun%d", ppa);
    
    // 配置 TUN 设备的网络参数
    snprintf(cmd, sizeof(cmd), "ifconfig %s %s/32 %s mtu %d up",
        tun->name, addr_ntoa(src), addr_ntoa(dst), mtu);
    
    // 执行系统命令配置 TUN 设备
    if (system(cmd) < 0)
        return (tun_close(tun));
    
    return (tun);
}

// 获取 TUN 设备名称
const char *
tun_name(tun_t *tun)
{
    return (tun->name);
}

// 获取 TUN 设备文件描述符
int
tun_fileno(tun_t *tun)
{
    return (tun->fd);
}

// 发送数据到 TUN 设备
ssize_t
tun_send(tun_t *tun, const void *buf, size_t size)
{
    struct strbuf sbuf;

    sbuf.buf = buf;
    sbuf.len = size;
    # 如果通过tun->fd发送消息成功，则返回发送消息的长度，否则返回-1
    return (putmsg(tun->fd, NULL, &sbuf, 0) >= 0 ? sbuf.len : -1);
# 接收从 TUN 设备读取的数据，存储到指定的缓冲区中
ssize_t
tun_recv(tun_t *tun, void *buf, size_t size)
{
    # 定义用于接收消息的结构体
    struct strbuf sbuf;
    # 定义接收消息的标志
    int flags = 0;
    
    # 设置接收消息的缓冲区和最大长度
    sbuf.buf = buf;
    sbuf.maxlen = size;
    # 调用 getmsg 函数接收消息，并根据返回值判断是否接收成功
    return (getmsg(tun->fd, NULL, &sbuf, &flags) >= 0 ? sbuf.len : -1);
}

# 关闭 TUN 设备，并释放相关资源
tun_t *
tun_close(tun_t *tun)
{
    # 如果 TUN 设备文件描述符有效，则关闭文件
    if (tun->if_fd >= 0)
        close(tun->if_fd);
    # 如果 IP 设备文件描述符有效，则关闭文件
    if (tun->ip_fd >= 0)
        close(tun->ip_fd);
    # 如果 TUN 设备文件描述符有效，则关闭文件
    if (tun->fd >= 0)
        close(tun->fd);
    # 释放 TUN 设备结构体内存
    free(tun);
    # 返回空指针
    return (NULL);
}
```