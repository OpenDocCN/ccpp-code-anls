# `nmap\libdnet-stripped\src\route-hpux.c`

```cpp
/*
 * route-hpux.c
 *
 * 版权所有 (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-hpux.c 483 2004-01-14 04:52:11Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/mib.h>
#include <sys/socket.h>

#include <net/route.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

#define ADDR_ISHOST(a)    (((a)->addr_type == ADDR_TYPE_IP &&    \
              (a)->addr_bits == IP_ADDR_BITS) ||    \
             ((a)->addr_type == ADDR_TYPE_IP6 &&    \
              (a)->addr_bits == IP6_ADDR_BITS))

struct route_handle {
    int    fd;
};

route_t *
route_open(void)
{
    route_t *r;

    if ((r = calloc(1, sizeof(*r))) != NULL) {
        if ((r->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
            return (route_close(r));
    }
    return (r);
}

int
route_add(route_t *r, const struct route_entry *entry)
{
    struct rtentry rt;
    struct addr dst;
    
    memset(&rt, 0, sizeof(rt)); // 初始化 rt 结构体为 0
    rt.rt_flags = RTF_UP | RTF_GATEWAY; // 设置路由标志为 UP 和 GATEWAY

    if (ADDR_ISHOST(&entry->route_dst)) { // 如果目的地址是主机地址
        rt.rt_flags |= RTF_HOST; // 设置路由标志为 HOST
        memcpy(&dst, &entry->route_dst, sizeof(dst)); // 复制目的地址
    } else
        addr_net(&entry->route_dst, &dst); // 否则计算网络地址

    if (addr_ntos(&dst, &rt.rt_dst) < 0 || // 将地址转换为字符串形式
        addr_ntos(&entry->route_gw, &rt.rt_gateway) < 0 || // 将网关地址转换为字符串形式
        addr_btom(entry->route_dst.addr_bits, &rt.rt_subnetmask, // 将地址位转换为子网掩码
        IP_ADDR_LEN) < 0)
        return (-1); // 如果转换失败，返回错误
    
    return (ioctl(r->fd, SIOCADDRT, &rt)); // 添加路由
}

int
route_delete(route_t *r, const struct route_entry *entry)
{
    struct rtentry rt;
    struct addr dst;

    memset(&rt, 0, sizeof(rt)); // 初始化 rt 结构体为 0
    rt.rt_flags = RTF_UP; // 设置路由标志为 UP
    
    if (ADDR_ISHOST(&entry->route_dst)) { // 如果目的地址是主机地址
        rt.rt_flags |= RTF_HOST; // 设置路由标志为 HOST
        memcpy(&dst, &entry->route_dst, sizeof(dst)); // 复制目的地址
    } else
        addr_net(&entry->route_dst, &dst); // 否则计算网络地址
    # 如果将目的地址转换为字符串失败，或者将路由目的地址转换为子网掩码失败，则返回-1
    if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
        addr_btom(entry->route_dst.addr_bits, &rt.rt_subnetmask,
        IP_ADDR_LEN) < 0)
        return (-1);
    
    # 使用ioctl系统调用删除指定的路由
    return (ioctl(r->fd, SIOCDELRT, &rt));
}

int
route_get(route_t *r, struct route_entry *entry)
{
    // 初始化路由请求结构体
    struct rtreq rtr;

    // 将路由请求结构体清零
    memset(&rtr, 0, sizeof(rtr));

    /* XXX - gross hack for default route */
    // 如果目的地址是任意地址，则设置默认路由的目的地址和子网掩码
    if (entry->route_dst.addr_ip == IP_ADDR_ANY) {
        rtr.rtr_destaddr = htonl(0x60060606);
        rtr.rtr_subnetmask = 0xffffffff;
    } else {
        // 否则，将目的地址和子网掩码拷贝到路由请求结构体中
        memcpy(&rtr.rtr_destaddr, &entry->route_dst.addr_ip,
            IP_ADDR_LEN);
        // 如果目的地址的位数小于 IP 地址位数，则计算子网掩码
        if (entry->route_dst.addr_bits < IP_ADDR_BITS)
            addr_btom(entry->route_dst.addr_bits,
                &rtr.rtr_subnetmask, IP_ADDR_LEN);
    }
    // 使用 ioctl 获取路由表项信息
    if (ioctl(r->fd, SIOCGRTENTRY, &rtr) < 0)
        return (-1);

    // 如果网关地址为 0，则设置错误码并返回
    if (rtr.rtr_gwayaddr == 0) {
        errno = ESRCH;
        return (-1);
    }
    // 清空接口名
    entry->intf_name[0] = '\0';
    // 设置路由网关地址类型和位数，并拷贝网关地址
    entry->route_gw.addr_type = ADDR_TYPE_IP;
    entry->route_gw.addr_bits = IP_ADDR_BITS;
    memcpy(&entry->route_gw.addr_ip, &rtr.rtr_gwayaddr, IP_ADDR_LEN);
    // 设置度量值为 0
    entry->metric = 0;
    
    return (0);
}

// 定义最大路由表项数
#define MAX_RTENTRIES    256    /* XXX */

int
route_loop(route_t *r, route_handler callback, void *arg)
{
    // 定义网络参数结构体和路由表项结构体
    struct nmparms nm;
    struct route_entry entry;
    mib_ipRouteEnt rtentries[MAX_RTENTRIES];
    int fd, i, n, ret;
    
    // 打开 MIB 设备
    if ((fd = open_mib("/dev/ip", O_RDWR, 0 /* XXX */, 0)) < 0)
        return (-1);
    
    // 设置网络参数结构体的对象 ID、缓冲区和长度
    nm.objid = ID_ipRouteTable;
    nm.buffer = rtentries;
    n = sizeof(rtentries);
    nm.len = &n;
    
    // 获取 MIB 信息
    if (get_mib_info(fd, &nm) < 0) {
        close_mib(fd);
        return (-1);
    }
    close_mib(fd);

    // 计算路由表项数
    n /= sizeof(*rtentries);
    ret = 0;
    # 遍历从 0 到 n-1 的索引
    for (i = 0; i < n; i++) {
        # 如果当前路由条目的类型不是 NMDIRECT 且不是 NMREMOTE，则跳过当前循环，继续下一次循环
        if (rtentries[i].Type != NMDIRECT &&
            rtentries[i].Type != NMREMOTE)
            continue;
        
        # 将接口名称的第一个字符设置为空字符
        entry.intf_name[0] = '\0';
        # 设置路由目的地址和网关地址的地址类型为 IP 地址
        entry.route_dst.addr_type = entry.route_gw.addr_type = ADDR_TYPE_IP;
        # 设置路由目的地址和网关地址的地址位数为 IP 地址位数
        entry.route_dst.addr_bits = entry.route_gw.addr_bits = IP_ADDR_BITS;
        # 设置路由目的地址为当前路由条目的目的地址
        entry.route_dst.addr_ip = rtentries[i].Dest;
        # 将当前路由条目的掩码转换为地址位数，并存储到路由目的地址的地址位数中
        addr_mtob(&rtentries[i].Mask, IP_ADDR_LEN,
            &entry.route_dst.addr_bits);
        # 设置路由网关地址为当前路由条目的下一跳地址
        entry.route_gw.addr_ip = rtentries[i].NextHop;
        # 设置度量值为 0
        entry.metric = 0;

        # 调用回调函数，将当前路由条目作为参数传入，如果返回值不为 0，则跳出循环
        if ((ret = callback(&entry, arg)) != 0)
            break;
    }
    # 返回回调函数的返回值
    return (ret);
# 释放路由对象占用的资源，并返回空指针
route_t *
route_close(route_t *r)
{
    # 检查路由对象是否为空
    if (r != NULL) {
        # 如果路由对象的文件描述符大于等于0，则关闭文件
        if (r->fd >= 0)
            close(r->fd);
        # 释放路由对象占用的内存
        free(r);
    }
    # 返回空指针
    return (NULL);
}
```