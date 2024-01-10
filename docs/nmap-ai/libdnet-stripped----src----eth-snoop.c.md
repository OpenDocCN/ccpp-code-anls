# `nmap\libdnet-stripped\src\eth-snoop.c`

```
/*
 * eth-snoop.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-snoop.c 548 2005-01-30 06:01:57Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>

#include <net/if.h>
#include <net/raw.h>

#include <assert.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 定义一个结构体用于存储以太网句柄
struct eth_handle {
    int        fd;
    struct ifreq    ifr;
};

// 打开以太网设备
eth_t *
eth_open(const char *device)
{
    struct sockaddr_raw sr;
    eth_t *e;
    int n;
    
    // 分配以太网句柄内存
    if ((e = calloc(1, sizeof(*e))) == NULL)
        return (NULL);

    // 创建原始套接字
    if ((e->fd = socket(PF_RAW, SOCK_RAW, RAWPROTO_SNOOP)) < 0)
        return (eth_close(e));
    
    // 绑定原始套接字到指定设备
    memset(&sr, 0, sizeof(sr));
    sr.sr_family = AF_RAW;
    strlcpy(sr.sr_ifname, device, sizeof(sr.sr_ifname));

    if (bind(e->fd, (struct sockaddr *)&sr, sizeof(sr)) < 0)
        return (eth_close(e));
    
    // 设置发送缓冲区大小
    n = 60000;
    if (setsockopt(e->fd, SOL_SOCKET, SO_SNDBUF, &n, sizeof(n)) < 0)
        return (eth_close(e));
    
    // 将设备名称拷贝到 ifreq 结构体中
    strlcpy(e->ifr.ifr_name, device, sizeof(e->ifr.ifr_name));
    
    return (e);
}

// 获取以太网地址
int
eth_get(eth_t *e, eth_addr_t *ea)
{
    struct addr ha;
    
    // 获取设备地址
    if (ioctl(e->fd, SIOCGIFADDR, &e->ifr) < 0)
        return (-1);

    // 将地址转换为以太网地址
    if (addr_ston(&e->ifr.ifr_addr, &ha) < 0)
        return (-1);

    // 检查地址类型是否为以太网地址
    if (ha.addr_type != ADDR_TYPE_ETH) {
        errno = EINVAL;
        return (-1);
    }
    memcpy(ea, &ha.addr_eth, sizeof(*ea));
    
    return (0);
}

// 设置以太网地址
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    struct addr ha;

    ha.addr_type = ADDR_TYPE_ETH;
    ha.addr_bits = ETH_ADDR_BITS;
    memcpy(&ha.addr_eth, ea, ETH_ADDR_LEN);
        
    if (addr_ntos(&ha, &e->ifr.ifr_addr) < 0)
        return (-1);
    
    return (ioctl(e->fd, SIOCSIFADDR, &e->ifr));
}

// 发送数据
ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    return (write(e->fd, buf, len));
}

// 关闭以太网句柄
eth_t *
eth_close(eth_t *e)
{
    # 如果指针 e 不为空
    if (e != NULL) {
        # 如果指针 e 的文件描述符大于等于 0
        if (e->fd >= 0)
            # 关闭指针 e 的文件描述符
            close(e->fd);
        # 释放指针 e 指向的内存
        free(e);
    }
    # 返回空指针
    return (NULL);
# 闭合前面的函数定义
```