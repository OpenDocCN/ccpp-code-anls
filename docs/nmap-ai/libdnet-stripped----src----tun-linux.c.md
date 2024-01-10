# `nmap\libdnet-stripped\src\tun-linux.c`

```
/*
 * tun-linux.c
 *
 * Universal TUN/TAP driver, in Linux 2.4+
 * /usr/src/linux/Documentation/networking/tuntap.txt
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: tun-linux.c 612 2005-09-12 02:18:06Z dugsong $
 */

#include "config.h"

#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/uio.h>

#include <linux/if.h>
#include <linux/if_tun.h>

#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct tun {
    int fd;             // 文件描述符
    intf_t *intf;       // 网络接口
    struct ifreq ifr;   // 网络接口请求
};

tun_t *
tun_open(struct addr *src, struct addr *dst, int mtu)
{
    tun_t *tun;
    struct intf_entry ifent;
    
    if ((tun = calloc(1, sizeof(*tun))) == NULL)  // 分配内存
        return (NULL);

    if ((tun->fd = open("/dev/net/tun", O_RDWR, 0)) < 0 ||  // 打开 TUN/TAP 设备
        (tun->intf = intf_open()) == NULL)  // 打开网络接口
        return (tun_close(tun));  // 关闭 TUN/TAP 设备和网络接口
    
    tun->ifr.ifr_flags = IFF_TUN;  // 设置网络接口标志为 TUN
    
    if (ioctl(tun->fd, TUNSETIFF, (void *) &tun->ifr) < 0)  // 设置 TUN/TAP 设备的接口
        return (tun_close(tun));  // 关闭 TUN/TAP 设备和网络接口

    memset(&ifent, 0, sizeof(ifent));  // 初始化网络接口信息
    strlcpy(ifent.intf_name, tun->ifr.ifr_name, sizeof(ifent.intf_name));  // 复制接口名称
    ifent.intf_flags = INTF_FLAG_UP|INTF_FLAG_POINTOPOINT;  // 设置接口标志
    ifent.intf_addr = *src;  // 设置接口地址
    ifent.intf_dst_addr = *dst;  // 设置目的地址
    ifent.intf_mtu = mtu;  // 设置最大传输单元
    
    if (intf_set(tun->intf, &ifent) < 0)  // 设置网络接口
        return (tun_close(tun));  // 关闭 TUN/TAP 设备和网络接口
    
    return (tun);  // 返回 TUN/TAP 设备
}

const char *
tun_name(tun_t *tun)
{
    return (tun->ifr.ifr_name);  // 返回 TUN/TAP 设备的接口名称
}

int
tun_fileno(tun_t *tun)
{
    return (tun->fd);  // 返回 TUN/TAP 设备的文件描述符
}

ssize_t
tun_send(tun_t *tun, const void *buf, size_t size)
{
    struct iovec iov[2];  // 定义数据块
    uint32_t type = ETH_TYPE_IP;  // 定义以太网类型
    
    iov[0].iov_base = &type;  // 设置数据块的基地址
    iov[0].iov_len = sizeof(type);  // 设置数据块的长度
    iov[1].iov_base = (void *)buf;  // 设置数据块的基地址
    iov[1].iov_len = size;  // 设置数据块的长度
    
    return (writev(tun->fd, iov, 2));  // 向 TUN/TAP 设备写入数据
}

ssize_t
tun_recv(tun_t *tun, void *buf, size_t size)
{
    struct iovec iov[2];  // 定义数据块
    uint32_t type;

    iov[0].iov_base = &type;  // 设置数据块的基地址
    iov[0].iov_len = sizeof(type);  // 设置数据块的长度
    # 设置第二个缓冲区的起始地址为buf
    iov[1].iov_base = (void *)buf;
    # 设置第二个缓冲区的长度为size
    iov[1].iov_len = size;
    
    # 从文件描述符tun->fd中读取iov数组中的数据，并返回读取的字节数减去type的大小
    return (readv(tun->fd, iov, 2) - sizeof(type));
# 关闭虚拟网络接口
tun_t *
tun_close(tun_t *tun)
{
    # 如果虚拟网络接口的文件描述符大于0，则关闭文件描述符
    if (tun->fd > 0)
        close(tun->fd);
    # 如果虚拟网络接口的接口不为空，则关闭接口
    if (tun->intf != NULL)
        intf_close(tun->intf);
    # 释放虚拟网络接口的内存空间
    free(tun);
    # 返回空指针
    return (NULL);
}
```