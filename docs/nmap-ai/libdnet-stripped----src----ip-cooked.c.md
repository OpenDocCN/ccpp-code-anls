# `nmap\libdnet-stripped\src\ip-cooked.c`

```cpp
/*
 * ip-cooked.c
 *
 * 版权所有 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-cooked.c 547 2005-01-25 21:30:40Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#ifndef _WIN32
#include <netinet/in.h>
#include <unistd.h>
#endif
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"
#include "queue.h"

// 定义 IP 接口结构体
struct ip_intf {
    eth_t            *eth;
    char             name[INTF_NAME_LEN];
    struct addr         ha;
    struct addr         pa;
    int             mtu;
    LIST_ENTRY(ip_intf)     next;
};

// 定义 IP 句柄结构体
struct ip_handle {
    arp_t            *arp;
    intf_t            *intf;
    route_t            *route;
    int             fd;
    struct sockaddr_in     sin;
    
    LIST_HEAD(, ip_intf)     ip_intf_list;
};

// 添加 IP 接口函数
static int
_add_ip_intf(const struct intf_entry *entry, void *arg)
{
    ip_t *ip = (ip_t *)arg;
    struct ip_intf *ipi;

    // 检查接口类型、状态、MTU、地址类型等条件
    if (entry->intf_type == INTF_TYPE_ETH &&
        (entry->intf_flags & INTF_FLAG_UP) != 0 &&
        entry->intf_mtu >= ETH_LEN_MIN &&
        entry->intf_addr.addr_type == ADDR_TYPE_IP &&
        entry->intf_link_addr.addr_type == ADDR_TYPE_ETH) {
        
        // 分配内存并初始化 IP 接口结构体
        if ((ipi = calloc(1, sizeof(*ipi))) == NULL)
            return (-1);
        
        // 复制接口名称和地址信息到 IP 接口结构体
        strlcpy(ipi->name, entry->intf_name, sizeof(ipi->name));
        memcpy(&ipi->ha, &entry->intf_link_addr, sizeof(ipi->ha));
        memcpy(&ipi->pa, &entry->intf_addr, sizeof(ipi->pa));
        ipi->mtu = entry->intf_mtu;

        // 将 IP 接口结构体插入 IP 接口链表
        LIST_INSERT_HEAD(&ip->ip_intf_list, ipi, next);
    }
    return (0);
}

// 打开 IP 函数
ip_t *
ip_open(void)
{
    ip_t *ip;
    # 分配内存给 ip 指针，如果成功则继续执行
    if ((ip = calloc(1, sizeof(*ip))) != NULL) {
        # 初始化文件描述符为 -1
        ip->fd = -1;
        
        # 打开 ARP、接口和路由表，如果有任何一个打开失败则关闭 ip 并返回
        if ((ip->arp = arp_open()) == NULL ||
            (ip->intf = intf_open()) == NULL ||
            (ip->route = route_open()) == NULL)
            return (ip_close(ip));
        
        # 创建一个 AF_INET 类型的套接字
        if ((ip->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
            return (ip_close(ip));

        # 将 ip->sin 的内存清零，并设置 sin_family 为 AF_INET，sin_port 为 htons(666)
        memset(&ip->sin, 0, sizeof(ip->sin));
        ip->sin.sin_family = AF_INET;
        ip->sin.sin_port = htons(666);
        
        # 初始化 ip_intf_list
        LIST_INIT(&ip->ip_intf_list);

        # 对接口列表进行循环，调用 _add_ip_intf 函数，如果返回值不为 0 则关闭 ip 并返回
        if (intf_loop(ip->intf, _add_ip_intf, ip) != 0)
            return (ip_close(ip));
    }
    # 返回 ip 指针
    return (ip);
# 查找指定 IP 地址对应的网络接口
static struct ip_intf *
_lookup_ip_intf(ip_t *ip, ip_addr_t dst)
{
    struct ip_intf *ipi;  # 定义一个指向 ip_intf 结构体的指针
    int n;  # 定义一个整型变量 n

    ip->sin.sin_addr.s_addr = dst;  # 将目标 IP 地址赋值给 ip 结构体中的 sin_addr 字段
    n = sizeof(ip->sin);  # 获取 ip 结构体中 sin 字段的大小
    
    if (connect(ip->fd, (struct sockaddr *)&ip->sin, n) < 0)  # 如果连接失败
        return (NULL);  # 返回空指针

    if (getsockname(ip->fd, (struct sockaddr *)&ip->sin, &n) < 0)  # 如果获取套接字名称失败
        return (NULL);  # 返回空指针

    LIST_FOREACH(ipi, &ip->ip_intf_list, next) {  # 遍历 ip_intf_list 链表
        if (ipi->pa.addr_ip == ip->sin.sin_addr.s_addr) {  # 如果 ipi 结构体中的 addr_ip 字段等于目标 IP 地址
            if (ipi->eth == NULL) {  # 如果 ipi 结构体中的 eth 字段为空
                if ((ipi->eth = eth_open(ipi->name)) == NULL)  # 尝试打开网络接口
                    return (NULL);  # 如果失败，返回空指针
            }
            if (ipi != LIST_FIRST(&ip->ip_intf_list)) {  # 如果 ipi 不是 ip_intf_list 链表的第一个元素
                LIST_REMOVE(ipi, next);  # 从链表中移除 ipi
                LIST_INSERT_HEAD(&ip->ip_intf_list, ipi, next);  # 将 ipi 插入到链表头部
            }
            return (ipi);  # 返回找到的 ip_intf 结构体指针
        }
    }
    return (NULL);  # 如果未找到匹配的 ip_intf 结构体，返回空指针
}

# 发送 ARP 请求
static void
_request_arp(struct ip_intf *ipi, struct addr *dst)
{
    u_char frame[ETH_HDR_LEN + ARP_HDR_LEN + ARP_ETHIP_LEN];  # 定义一个以太网帧

    eth_pack_hdr(frame, ETH_ADDR_BROADCAST, ipi->ha.addr_eth,  # 封装以太网帧头部
        ETH_TYPE_ARP);
    arp_pack_hdr_ethip(frame + ETH_HDR_LEN, ARP_OP_REQUEST,  # 封装 ARP 头部
        ipi->ha.addr_eth, ipi->pa.addr_ip, ETH_ADDR_BROADCAST,
        dst->addr_ip);

    eth_send(ipi->eth, frame, sizeof(frame));  # 发送以太网帧
}

# 发送 IP 数据包
ssize_t
ip_send(ip_t *ip, const void *buf, size_t len)
{
    struct ip_hdr *iph;  # 定义一个指向 ip_hdr 结构体的指针
    struct ip_intf *ipi;  # 定义一个指向 ip_intf 结构体的指针
    struct arp_entry arpent;  # 定义一个 arp_entry 结构体
    struct route_entry rtent;  # 定义一个 route_entry 结构体
    u_char frame[ETH_LEN_MAX];  # 定义一个以太网帧
    int i, usec;  # 定义整型变量 i 和 usec

    iph = (struct ip_hdr *)buf;  # 将 buf 转换为 ip_hdr 结构体指针
    
    if ((ipi = _lookup_ip_intf(ip, iph->ip_dst)) == NULL) {  # 查找目标 IP 地址对应的网络接口
        errno = EHOSTUNREACH;  # 设置错误码为主机不可达
        return (-1);  # 返回 -1
    }
    arpent.arp_pa.addr_type = ADDR_TYPE_IP;  # 设置 arp_entry 结构体中的 addr_type 字段
    arpent.arp_pa.addr_bits = IP_ADDR_BITS;  # 设置 arp_entry 结构体中的 addr_bits 字段
    arpent.arp_pa.addr_ip = iph->ip_dst;  # 设置 arp_entry 结构体中的 addr_ip 字段为目标 IP 地址
    memcpy(&rtent.route_dst, &arpent.arp_pa, sizeof(rtent.route_dst));  # 复制 arp_entry 结构体到 route_entry 结构体
}
    # 循环3次，每次延迟时间递增
    for (i = 0, usec = 10; i < 3; i++, usec *= 100) {
        # 如果能够获取到目标 IP 的 ARP 地址，则跳出循环
        if (arp_get(ip->arp, &arpent) == 0)
            break;
        
        # 如果能够获取到目标 IP 的路由信息，并且路由的下一跳地址不是本机地址，则更新 ARP 地址
        if (route_get(ip->route, &rtent) == 0 &&
            rtent.route_gw.addr_ip != ipi->pa.addr_ip) {
            memcpy(&arpent.arp_pa, &rtent.route_gw,
                sizeof(arpent.arp_pa));
            # 如果能够获取到更新后的 ARP 地址，则跳出循环
            if (arp_get(ip->arp, &arpent) == 0)
                break;
        }
        # 发送 ARP 请求
        _request_arp(ipi, &arpent.arp_pa);

        # 延迟一段时间
        usleep(usec);
    }
    # 如果循环结束仍未获取到 ARP 地址，则将目标 MAC 地址设置为广播地址
    if (i == 3)
        memset(&arpent.arp_ha.addr_eth, 0xff, ETH_ADDR_LEN);
    
    # 封装以太网帧头部
    eth_pack_hdr(frame, arpent.arp_ha.addr_eth,
        ipi->ha.addr_eth, ETH_TYPE_IP);

    # 如果数据包长度大于 MTU，则进行分片处理
    if (len > ipi->mtu) {
        u_char *p, *start, *end, *ip_data;
        int ip_hl, fraglen;
        
        # 计算 IP 头部长度和分片长度
        ip_hl = iph->ip_hl << 2;
        fraglen = ipi->mtu - ip_hl;

        # 拷贝 IP 头部
        iph = (struct ip_hdr *)(frame + ETH_HDR_LEN);
        memcpy(iph, buf, ip_hl);
        ip_data = (u_char *)iph + ip_hl;

        start = (u_char *)buf + ip_hl;
        end = (u_char *)buf + len;
        
        # 对数据进行分片处理
        for (p = start; p < end; ) {
            memcpy(ip_data, p, fraglen);
            
            # 更新 IP 头部长度和偏移量
            iph->ip_len = htons(ip_hl + fraglen);
            iph->ip_off = htons(((p + fraglen < end) ? IP_MF : 0) |
                ((p - start) >> 3));

            # 计算 IP 头部校验和
            ip_checksum(iph, ip_hl + fraglen);

            i = ETH_HDR_LEN + ip_hl + fraglen;
            # 发送分片后的数据包
            if (eth_send(ipi->eth, frame, i) != i)
                return (-1);
            p += fraglen;
            if (end - p < fraglen)
                fraglen = end - p;
        }
        return (len);
    }
    # 如果数据包长度不大于 MTU，则直接拷贝数据并发送
    memcpy(frame + ETH_HDR_LEN, buf, len);
    i = ETH_HDR_LEN + len;
    if (eth_send(ipi->eth, frame, i) != i)
        return (-1);
    
    return (len);
# 释放 IP 对象占用的资源，并返回空指针
ip_t *
ip_close(ip_t *ip)
{
    struct ip_intf *ipi, *nxt;

    # 如果 IP 对象不为空
    if (ip != NULL) {
        # 遍历 IP 对象的接口列表
        for (ipi = LIST_FIRST(&ip->ip_intf_list);
            ipi != LIST_END(&ip->ip_intf_list); ipi = nxt) {
            nxt = LIST_NEXT(ipi, next);
            # 如果接口的以太网对象不为空，则关闭以太网对象
            if (ipi->eth != NULL)
                eth_close(ipi->eth);
            # 释放接口对象占用的资源
            free(ipi);
        }
        # 如果 IP 对象的文件描述符大于等于 0，则关闭文件描述符
        if (ip->fd >= 0)
            close(ip->fd);
        # 如果 IP 对象的路由表不为空，则关闭路由表
        if (ip->route != NULL)
            route_close(ip->route);
        # 如果 IP 对象的接口不为空，则关闭接口
        if (ip->intf != NULL)
            intf_close(ip->intf);
        # 如果 IP 对象的 ARP 对象不为空，则关闭 ARP 对象
        if (ip->arp != NULL)
            arp_close(ip->arp);
        # 释放 IP 对象占用的资源
        free(ip);
    }
    # 返回空指针
    return (NULL);
}
```