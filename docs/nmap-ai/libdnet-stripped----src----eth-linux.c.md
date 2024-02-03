# `nmap\libdnet-stripped\src\eth-linux.c`

```cpp
/*
 * eth-linux.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-linux.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>

#include <net/if.h>
#include <features.h>
#if __GLIBC__ >= 2 && __GLIBC_MINOR >= 1
#include <netpacket/packet.h>
#include <net/ethernet.h>
#else
#include <asm/types.h>
#include <linux/if_packet.h>
#include <linux/if_ether.h>
#endif /* __GLIBC__ */
#include <netinet/in.h>

#include <assert.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct eth_handle {
    int            fd;
    struct ifreq        ifr;
    struct sockaddr_ll    sll;
};

// 打开以太网设备
eth_t *
eth_open(const char *device)
{
    eth_t *e;
    int n;
    
    if ((e = calloc(1, sizeof(*e))) != NULL) {
        // 创建原始套接字
        if ((e->fd = socket(PF_PACKET, SOCK_RAW,
             htons(ETH_P_ALL))) < 0)
            return (eth_close(e));
#ifdef SO_BROADCAST
        n = 1;
        // 设置套接字选项，允许广播
        if (setsockopt(e->fd, SOL_SOCKET, SO_BROADCAST, &n,
            sizeof(n)) < 0)
            return (eth_close(e));
#endif
        // 将设备名称复制到 ifreq 结构体中
        strlcpy(e->ifr.ifr_name, device, sizeof(e->ifr.ifr_name));
        
        // 获取设备索引
        if (ioctl(e->fd, SIOCGIFINDEX, &e->ifr) < 0)
            return (eth_close(e));
        
        // 设置 sockaddr_ll 结构体
        e->sll.sll_family = AF_PACKET;
        e->sll.sll_ifindex = e->ifr.ifr_ifindex;
    }
    return (e);
}

// 发送以太网数据帧
ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    struct eth_hdr *eth = (struct eth_hdr *)buf;
    
    // 设置协议类型
    e->sll.sll_protocol = eth->eth_type;

    // 发送数据帧
    return (sendto(e->fd, buf, len, 0, (struct sockaddr *)&e->sll,
        sizeof(e->sll)));
}

// 关闭以太网设备
eth_t *
eth_close(eth_t *e)
{
    if (e != NULL) {
        if (e->fd >= 0)
            close(e->fd);
        free(e);
    }
    return (NULL);
}

// 获取以太网地址
int
eth_get(eth_t *e, eth_addr_t *ea)
{
    struct addr ha;
    
    if (ioctl(e->fd, SIOCGIFHWADDR, &e->ifr) < 0)
        return (-1);
    # 如果获取网络接口的硬件地址失败，则返回-1
    if (addr_ston(&e->ifr.ifr_hwaddr, &ha) < 0)
        return (-1);
    
    # 将硬件地址复制到指定的内存地址中
    memcpy(ea, &ha.addr_eth, sizeof(*ea));
    # 返回0表示成功
    return (0);
# 设置以太网地址
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    # 创建地址结构体
    struct addr ha;
    
    # 设置地址类型为以太网，地址位数为以太网地址位数
    ha.addr_type = ADDR_TYPE_ETH;
    ha.addr_bits = ETH_ADDR_BITS;
    # 复制以太网地址到地址结构体
    memcpy(&ha.addr_eth, ea, ETH_ADDR_LEN);
    
    # 将地址结构体转换为网络地址格式
    addr_ntos(&ha, &e->ifr.ifr_hwaddr);
    
    # 设置以太网接口的硬件地址
    return (ioctl(e->fd, SIOCSIFHWADDR, &e->ifr));
}
```