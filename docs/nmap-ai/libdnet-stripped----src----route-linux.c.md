# `nmap\libdnet-stripped\src\route-linux.c`

```
/*
 * route-linux.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-linux.c 619 2006-01-15 07:33:29Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/uio.h>

#include <asm/types.h>
#include <net/if.h>
#include <netinet/in.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

#include <net/route.h>

#include <ctype.h>
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

#define PROC_ROUTE_FILE        "/proc/net/route"
#define PROC_IPV6_ROUTE_FILE    "/proc/net/ipv6_route"

struct route_handle {
    int     fd;
    int     nlfd;
};

route_t *
route_open(void)
{
    struct sockaddr_nl snl;
    route_t *r;

    if ((r = calloc(1, sizeof(*r))) != NULL) {
        r->fd = r->nlfd = -1;
        
        if ((r->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
            return (route_close(r));
        
        if ((r->nlfd = socket(AF_NETLINK, SOCK_RAW,
             NETLINK_ROUTE)) < 0)
            return (route_close(r));
        
        memset(&snl, 0, sizeof(snl));
        snl.nl_family = AF_NETLINK;
        
        if (bind(r->nlfd, (struct sockaddr *)&snl, sizeof(snl)) < 0)
            return (route_close(r));
    }
    return (r);
}

int
route_add(route_t *r, const struct route_entry *entry)
{
    struct rtentry rt;
    struct addr dst;

    memset(&rt, 0, sizeof(rt));
    rt.rt_flags = RTF_UP | RTF_GATEWAY;

    if (ADDR_ISHOST(&entry->route_dst)) {
        rt.rt_flags |= RTF_HOST;
        memcpy(&dst, &entry->route_dst, sizeof(dst));
    } else
        addr_net(&entry->route_dst, &dst);
    # 如果将目的地址转换为字符串失败，或者将路由网关地址转换为字符串失败，或者将路由目的地址的子网掩码转换为字符串失败
    if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
        addr_ntos(&entry->route_gw, &rt.rt_gateway) < 0 ||
        addr_btos(entry->route_dst.addr_bits, &rt.rt_genmask) < 0)
        # 返回 -1，表示失败
        return (-1);
    
    # 调用 ioctl 函数，添加路由表项
    return (ioctl(r->fd, SIOCADDRT, &rt));
}

int
route_delete(route_t *r, const struct route_entry *entry)
{
    struct rtentry rt; // 定义路由表项结构体
    struct addr dst; // 定义目的地址结构体
    
    memset(&rt, 0, sizeof(rt)); // 将路由表项结构体清零
    rt.rt_flags = RTF_UP; // 设置路由表项标志为 UP

    if (ADDR_ISHOST(&entry->route_dst)) { // 如果目的地址是主机地址
        rt.rt_flags |= RTF_HOST; // 设置路由表项标志为主机
        memcpy(&dst, &entry->route_dst, sizeof(dst)); // 复制目的地址
    } else
        addr_net(&entry->route_dst, &dst); // 计算网络地址
    
    if (addr_ntos(&dst, &rt.rt_dst) < 0 || // 如果转换目的地址到字符串失败
        addr_btos(entry->route_dst.addr_bits, &rt.rt_genmask) < 0) // 或者转换地址位数到字符串失败
        return (-1); // 返回错误
    
    return (ioctl(r->fd, SIOCDELRT, &rt)); // 调用系统调用删除路由表项
}

int
route_get(route_t *r, struct route_entry *entry)
{
    static int seq; // 静态变量，用于记录序列号
    struct nlmsghdr *nmsg; // 定义 Netlink 消息头
    struct rtmsg *rmsg; // 定义路由消息
    struct rtattr *rta; // 定义路由属性
    struct sockaddr_nl snl; // 定义 Netlink 地址结构
    struct iovec iov; // 定义 I/O 向量
    struct msghdr msg; // 定义消息头
    u_char buf[512]; // 定义缓冲区
    int i, af, alen; // 定义整型变量和地址长度

    switch (entry->route_dst.addr_type) { // 根据目的地址类型进行切换
    case ADDR_TYPE_IP: // 如果是 IPv4 地址
        af = AF_INET; // 设置地址族为 IPv4
        alen = IP_ADDR_LEN; // 设置地址长度为 IPv4 地址长度
        break;
    case ADDR_TYPE_IP6: // 如果是 IPv6 地址
        af = AF_INET6; // 设置地址族为 IPv6
        alen = IP6_ADDR_LEN; // 设置地址长度为 IPv6 地址长度
        break;
    default: // 其他情况
        errno = EINVAL; // 设置错误码为无效参数
        return (-1); // 返回错误
    }
    memset(buf, 0, sizeof(buf)); // 将缓冲区清零

    nmsg = (struct nlmsghdr *)buf; // 将缓冲区转换为 Netlink 消息头
    nmsg->nlmsg_len = NLMSG_LENGTH(sizeof(*nmsg)) + RTA_LENGTH(alen); // 设置消息长度
    nmsg->nlmsg_flags = NLM_F_REQUEST; // 设置消息标志为请求
    nmsg->nlmsg_type = RTM_GETROUTE; // 设置消息类型为获取路由
    nmsg->nlmsg_seq = ++seq; // 设置消息序列号

    rmsg = (struct rtmsg *)(nmsg + 1); // 将消息头后的部分转换为路由消息
    rmsg->rtm_family = af; // 设置路由消息的地址族
    rmsg->rtm_dst_len = entry->route_dst.addr_bits; // 设置路由消息的目的地址长度
    
    rta = RTM_RTA(rmsg); // 获取路由消息的路由属性
    rta->rta_type = RTA_DST; // 设置路由属性类型为目的地址
    rta->rta_len = RTA_LENGTH(alen); // 设置路由属性长度

    /* XXX - gross hack for default route */
    if (af == AF_INET && entry->route_dst.addr_ip == IP_ADDR_ANY) { // 如果是 IPv4 地址且目的地址为任意地址
        i = htonl(0x60060606); // 设置一个特殊的地址
        memcpy(RTA_DATA(rta), &i, alen); // 将特殊地址复制到路由属性数据中
    } else
        memcpy(RTA_DATA(rta), entry->route_dst.addr_data8, alen); // 否则将目的地址复制到路由属性数据中
    
    memset(&snl, 0, sizeof(snl)); // 将 Netlink 地址结构清零
    snl.nl_family = AF_NETLINK; // 设置地址族为 Netlink

    iov.iov_base = nmsg; // 设置 I/O 向量的基地址为消息头
    iov.iov_len = nmsg->nlmsg_len; // 设置 I/O 向量的长度为消息长度
    
    memset(&msg, 0, sizeof(msg)); // 将消息头清零
    # 设置消息的目标地址为 snl，并指定目标地址的长度
    msg.msg_name = &snl;
    msg.msg_namelen = sizeof(snl);
    # 设置消息的数据缓冲区和缓冲区长度
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    
    # 发送消息到指定的套接字文件描述符，如果发送失败则返回-1
    if (sendmsg(r->nlfd, &msg, 0) < 0)
        return (-1);

    # 设置数据缓冲区的基地址和长度
    iov.iov_base = buf;
    iov.iov_len = sizeof(buf);
    
    # 从指定的套接字文件描述符接收消息，如果接收失败或者接收到的消息长度小于等于0则返回-1
    if ((i = recvmsg(r->nlfd, &msg, 0)) <= 0)
        return (-1);

    # 检查接收到的消息是否有效，如果无效则返回-1
    if (nmsg->nlmsg_len < (int)sizeof(*nmsg) || nmsg->nlmsg_len > i ||
        nmsg->nlmsg_seq != seq) {
        errno = EINVAL;
        return (-1);
    }
    # 检查接收到的消息类型是否为错误消息，如果是则返回-1
    if (nmsg->nlmsg_type == NLMSG_ERROR)
        return (-1);
    
    # 计算剩余消息的长度
    i -= NLMSG_LENGTH(sizeof(*nmsg));
    
    # 设置路由网关地址类型为无，清空接口名称
    entry->route_gw.addr_type = ADDR_TYPE_NONE;
    entry->intf_name[0] = '\0';
    # 遍历消息中的路由属性
    for (rta = RTM_RTA(rmsg); RTA_OK(rta, i); rta = RTA_NEXT(rta, i)) {
        # 如果路由属性类型为网关，则设置路由网关地址类型和地址数据
        if (rta->rta_type == RTA_GATEWAY) {
            entry->route_gw.addr_type = entry->route_dst.addr_type;
            memcpy(entry->route_gw.addr_data8, RTA_DATA(rta), alen);
            entry->route_gw.addr_bits = alen * 8;
        } 
        # 如果路由属性类型为输出接口，则获取接口名称并设置到路由表项中
        else if (rta->rta_type == RTA_OIF) {
            char ifbuf[IFNAMSIZ];
            char *p;
            int intf_index;

            intf_index = *(int *) RTA_DATA(rta);
            p = if_indextoname(intf_index, ifbuf);
            if (p == NULL)
                return (-1);
            strlcpy(entry->intf_name, ifbuf, sizeof(entry->intf_name));
        }
    }
    # 如果路由网关地址类型为无，则设置错误码并返回-1
    if (entry->route_gw.addr_type == ADDR_TYPE_NONE) {
        errno = ESRCH;
        return (-1);
    }
    
    # 返回成功
    return (0);
# 定义一个函数，用于遍历路由表并调用回调函数处理每一条路由
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    FILE *fp;  # 定义文件指针
    struct route_entry entry;  # 定义路由表条目结构体
    char buf[BUFSIZ];  # 定义缓冲区
    char ifbuf[16];  # 定义接口缓冲区
    int ret = 0;  # 定义返回值，默认为0

    if ((fp = fopen(PROC_ROUTE_FILE, "r")) != NULL) {  # 打开路由文件，如果成功
        int i, iflags, refcnt, use, metric, mss, win, irtt;  # 定义各种参数
        uint32_t mask;  # 定义掩码

        while (fgets(buf, sizeof(buf), fp) != NULL) {  # 逐行读取路由文件内容
            i = sscanf(buf, "%15s %X %X %X %d %d %d %X %d %d %d\n",  # 从缓冲区中解析出各个参数
                ifbuf, &entry.route_dst.addr_ip,
                &entry.route_gw.addr_ip, &iflags, &refcnt, &use,
                &metric, &mask, &mss, &win, &irtt);
            
            if (i < 11 || !(iflags & RTF_UP))  # 如果解析的参数不足11个或者接口标志不是RTF_UP，则跳过
                continue;
        
            strlcpy(entry.intf_name, ifbuf, sizeof(entry.intf_name));  # 复制接口名到路由表条目结构体中

            entry.route_dst.addr_type = entry.route_gw.addr_type =  # 设置目的地址和网关地址的类型为IP地址
                ADDR_TYPE_IP;
        
            if (addr_mtob(&mask, IP_ADDR_LEN,  # 将掩码转换为位表示，并存入路由表条目结构体中
                &entry.route_dst.addr_bits) < 0)
                continue;
            
            entry.route_gw.addr_bits = IP_ADDR_BITS;  # 设置网关地址的位数为IP地址位数
            entry.metric = metric;  # 设置路由表条目的度量值为解析得到的度量值
            
            if ((ret = callback(&entry, arg)) != 0)  # 调用回调函数处理路由表条目，如果返回值不为0，则跳出循环
                break;
        }
        fclose(fp);  # 关闭文件
    }
    # 如果 ret 等于 0 并且成功打开 PROC_IPV6_ROUTE_FILE 文件，则执行以下操作
    if (ret == 0 && (fp = fopen(PROC_IPV6_ROUTE_FILE, "r")) != NULL) {
        # 定义字符数组和整型变量
        char s[33], d[8][5], n[8][5];
        int i, iflags, metric;
        u_int slen, dlen;
        
        # 逐行读取文件内容
        while (fgets(buf, sizeof(buf), fp) != NULL) {
            # 从字符串中按指定格式读取数据
            i = sscanf(buf, "%04s%04s%04s%04s%04s%04s%04s%04s %02x "
                "%32s %02x %04s%04s%04s%04s%04s%04s%04s%04s "
                "%x %*x %*x %x %15s",
                d[0], d[1], d[2], d[3], d[4], d[5], d[6], d[7],
                &dlen, s, &slen,
                n[0], n[1], n[2], n[3], n[4], n[5], n[6], n[7],
                &metric, &iflags, ifbuf);
            
            # 如果读取的数据不足 21 个或者 iflags 不包含 RTF_UP 标志，则跳过本次循环
            if (i < 21 || !(iflags & RTF_UP))
                continue;

            # 将 ifbuf 复制到 entry.intf_name
            strlcpy(entry.intf_name, ifbuf, sizeof(entry.intf_name));

            # 格式化目的地址字符串并转换为网络地址结构
            snprintf(buf, sizeof(buf), "%s:%s:%s:%s:%s:%s:%s:%s/%d",
                d[0], d[1], d[2], d[3], d[4], d[5], d[6], d[7],
                dlen);
            addr_aton(buf, &entry.route_dst);
            
            # 格式化网关地址字符串并转换为网络地址结构
            snprintf(buf, sizeof(buf), "%s:%s:%s:%s:%s:%s:%s:%s/%d",
                n[0], n[1], n[2], n[3], n[4], n[5], n[6], n[7],
                IP6_ADDR_BITS);
            addr_aton(buf, &entry.route_gw);
            
            # 设置 metric 值
            entry.metric = metric;
            
            # 调用回调函数，如果返回值不为 0 则跳出循环
            if ((ret = callback(&entry, arg)) != 0)
                break;
        }
        # 关闭文件
        fclose(fp);
    }
    # 返回 ret 的值
    return (ret);
# 释放路由对象占用的资源，并返回空指针
route_t *
route_close(route_t *r)
{
    # 如果路由对象不为空
    if (r != NULL) {
        # 如果路由对象的文件描述符大于等于0，关闭文件描述符
        if (r->fd >= 0)
            close(r->fd);
        # 如果路由对象的nlfd大于等于0，关闭nlfd
        if (r->nlfd >= 0)
            close(r->nlfd);
        # 释放路由对象占用的内存
        free(r);
    }
    # 返回空指针
    return (NULL);
}
```