# `nmap\libdnet-stripped\src\tun-bsd.c`

```cpp
/*
 * tun-bsd.c
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: tun-bsd.c 573 2005-02-10 23:50:04Z dugsong $
 */

#include "config.h"

#include <sys/socket.h>
#include <sys/uio.h>

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct tun {
    int               fd;
    intf_t           *intf;
    struct intf_entry save;
};

#define MAX_DEVS    16    /* XXX - max number of tunnel devices */

// 打开一个隧道设备
tun_t *
tun_open(struct addr *src, struct addr *dst, int mtu)
{
    struct intf_entry ifent;
    tun_t *tun;
    char dev[128];
    int i;

    // 检查源地址和目标地址是否为 IP 地址
    if (src->addr_type != ADDR_TYPE_IP || dst->addr_type != ADDR_TYPE_IP ||
        src->addr_bits != IP_ADDR_BITS || dst->addr_bits != IP_ADDR_BITS) {
        errno = EINVAL;
        return (NULL);
    }
    // 分配内存给 tun 结构体
    if ((tun = calloc(1, sizeof(*tun))) == NULL)
        return (NULL);

    // 打开一个接口
    if ((tun->intf = intf_open()) == NULL)
        return (tun_close(tun));

    // 初始化 ifent 结构体
    memset(&ifent, 0, sizeof(ifent));
    ifent.intf_len = sizeof(ifent);
    # 遍历设备编号，从0到MAX_DEVS-1
    for (i = 0; i < MAX_DEVS; i++) {
        # 格式化设备名称，将"/dev/tun"和i拼接成完整的设备名称
        snprintf(dev, sizeof(dev), "/dev/tun%d", i);
        # 将设备名称中的"tun"后面的部分复制到ifent.intf_name中
        strlcpy(ifent.intf_name, dev + 5, sizeof(ifent.intf_name));
        # 保存当前接口信息
        tun->save = ifent;
        
        # 尝试打开设备文件，以读写方式打开
        if ((tun->fd = open(dev, O_RDWR, 0)) != -1 &&
            # 获取接口信息，如果成功则继续
            intf_get(tun->intf, &tun->save) == 0) {
            # 定义路由和路由项
            route_t *r;
            struct route_entry entry;
            
            # 设置接口标志为UP和POINTOPOINT
            ifent.intf_flags = INTF_FLAG_UP|INTF_FLAG_POINTOPOINT;
            # 设置接口地址和目的地址
            ifent.intf_addr = *src;
            ifent.intf_dst_addr = *dst;    
            # 设置接口最大传输单元
            ifent.intf_mtu = mtu;
            
            # 如果设置接口信息失败，则关闭tun并赋值为NULL
            if (intf_set(tun->intf, &ifent) < 0)
                tun = tun_close(tun);

            # 尝试确保我们的路由设置成功
            if ((r = route_open()) != NULL) {
                # 设置路由项的目的地址和网关地址
                entry.route_dst = *dst;
                entry.route_gw = *src;
                # 添加路由项
                route_add(r, &entry);
                # 关闭路由
                route_close(r);
            }
            # 跳出循环
            break;
        }
    }
    # 如果i等于MAX_DEVS，则关闭tun并赋值为NULL
    if (i == MAX_DEVS)
        tun = tun_close(tun);
    # 返回tun
    return (tun);
}

const char *
tun_name(tun_t *tun)
{
    // 返回保存在 tun 结构体中的接口名称
    return (tun->save.intf_name);
}

int
tun_fileno(tun_t *tun)
{
    // 返回保存在 tun 结构体中的文件描述符
    return (tun->fd);
}

ssize_t
tun_send(tun_t *tun, const void *buf, size_t size)
{
#ifdef __OpenBSD__
    // 如果是 OpenBSD 系统，使用 writev 函数发送数据
    struct iovec iov[2];
    uint32_t af = htonl(AF_INET);

    iov[0].iov_base = &af;
    iov[0].iov_len = sizeof(af);
    iov[1].iov_base = (void *)buf;
    iov[1].iov_len = size;
    
    return (writev(tun->fd, iov, 2));
#else
    // 如果不是 OpenBSD 系统，使用 write 函数发送数据
    return (write(tun->fd, buf, size));
#endif
}

ssize_t
tun_recv(tun_t *tun, void *buf, size_t size)
{
#ifdef __OpenBSD__
    // 如果是 OpenBSD 系统，使用 readv 函数接收数据
    struct iovec iov[2];
    uint32_t af;
    
    iov[0].iov_base = &af;
    iov[0].iov_len = sizeof(af);
    iov[1].iov_base = (void *)buf;
    iov[1].iov_len = size;
    
    return (readv(tun->fd, iov, 2) - sizeof(af));
#else
    // 如果不是 OpenBSD 系统，使用 read 函数接收数据
    return (read(tun->fd, buf, size));
#endif
}

tun_t *
tun_close(tun_t *tun)
{
    if (tun->fd > 0)
        // 如果文件描述符大于 0，关闭文件描述符
        close(tun->fd);
    if (tun->intf != NULL) {
        /* 在关闭时恢复接口配置 */
        // 如果接口不为空，恢复接口配置并关闭接口
        intf_set(tun->intf, &tun->save);
        intf_close(tun->intf);
    }
    // 释放 tun 结构体内存并返回 NULL
    free(tun);
    return (NULL);
}
```