# `nmap\libdnet-stripped\src\arp-bsd.c`

```cpp
/*
 * arp-bsd.c
 * 
 * 版权所有 (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp-bsd.c 539 2005-01-23 07:36:54Z dugsong $
 */

#include "config.h"

#include <sys/param.h>
#include <sys/types.h>
#include <sys/socket.h>
#ifdef HAVE_SYS_SYSCTL_H
#include <sys/sysctl.h>
#endif
#ifdef HAVE_STREAMS_ROUTE
#include <sys/stream.h>
#include <sys/stropts.h>
#endif

#include <net/if.h>
#include <net/if_dl.h>
#include <net/route.h>
#include <netinet/in.h>
#include <netinet/if_ether.h>

#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct arp_handle {
    int    fd;
    int    seq;
};

struct arpmsg {
    struct rt_msghdr    rtm;
    u_char            addrs[256];
};

// 打开 ARP 处理器
arp_t *
arp_open(void)
{
    arp_t *arp;

    if ((arp = calloc(1, sizeof(*arp))) != NULL) {
#ifdef HAVE_STREAMS_ROUTE
        if ((arp->fd = open("/dev/route", O_RDWR, 0)) < 0)
#else
        if ((arp->fd = socket(PF_ROUTE, SOCK_RAW, 0)) < 0)
#endif
            return (arp_close(arp));
    }
    return (arp);
}

// 发送 ARP 消息
static int
arp_msg(arp_t *arp, struct arpmsg *msg)
{
    struct arpmsg smsg;
    int len, i = 0;
    pid_t pid;
    
    // 设置消息版本和序列号
    msg->rtm.rtm_version = RTM_VERSION;
    msg->rtm.rtm_seq = ++arp->seq; 
    memcpy(&smsg, msg, sizeof(smsg));
    
#ifdef HAVE_STREAMS_ROUTE
    return (ioctl(arp->fd, RTSTR_SEND, &msg->rtm));
#else
    // 写入消息到文件描述符
    if (write(arp->fd, &smsg, smsg.rtm.rtm_msglen) < 0) {
        if (errno != ESRCH || msg->rtm.rtm_type != RTM_DELETE)
            return (-1);
    }
    pid = getpid();
    
    /* XXX - should we only read RTM_GET responses here? */
    # 当从文件描述符中读取数据到消息结构体中，直到读取长度为0时结束循环
    while ((len = read(arp->fd, msg, sizeof(*msg))) > 0) {
        # 如果读取的长度小于消息结构体的长度，返回-1
        if (len < (int)sizeof(msg->rtm))
            return (-1);

        # 如果消息中的进程ID等于指定的PID
        if (msg->rtm.rtm_pid == pid) {
            # 如果消息中的序列号等于ARP缓存中的序列号，跳出循环
            if (msg->rtm.rtm_seq == arp->seq)
                break;
            # 否则继续下一次循环
            continue;
        } else if ((i++ % 2) == 0)
            # 如果不是指定的PID，且i的值除以2的余数为0，继续下一次循环
            continue;
        
        # 重复请求
        # 如果向文件描述符中写入消息失败
        if (write(arp->fd, &smsg, smsg.rtm.rtm_msglen) < 0) {
            # 如果错误码不是ESRCH，或者消息类型不是RTM_DELETE，返回-1
            if (errno != ESRCH || msg->rtm.rtm_type != RTM_DELETE)
                return (-1);
        }
    }
    # 如果读取长度小于0，返回-1
    if (len < 0)
        return (-1);
    
    # 返回0表示成功
    return (0);
#endif
}

// 添加 ARP 表项
int
arp_add(arp_t *arp, const struct arp_entry *entry)
{
    struct arpmsg msg;  // 定义 ARP 消息结构体
    struct sockaddr_in *sin;  // 定义指向 sockaddr_in 结构体的指针
    struct sockaddr *sa;  // 定义指向 sockaddr 结构体的指针
    int index, type;  // 定义整型变量 index 和 type

    // 检查地址类型是否为 IP 地址和以太网地址，如果不是则设置错误码并返回
    if (entry->arp_pa.addr_type != ADDR_TYPE_IP ||
        entry->arp_ha.addr_type != ADDR_TYPE_ETH) {
        errno = EAFNOSUPPORT;
        return (-1);
    }
    sin = (struct sockaddr_in *)msg.addrs;  // 将 msg.addrs 转换为 sockaddr_in 结构体指针
    sa = (struct sockaddr *)(sin + 1);  // 将 sin 后面的内存地址转换为 sockaddr 结构体指针

    // 将 entry->arp_pa 转换为网络地址字符串形式，如果失败则返回
    if (addr_ntos(&entry->arp_pa, (struct sockaddr *)sin) < 0)
        return (-1);

    // 初始化 ARP 消息结构体
    memset(&msg.rtm, 0, sizeof(msg.rtm));
    msg.rtm.rtm_type = RTM_GET;
    msg.rtm.rtm_addrs = RTA_DST;
    msg.rtm.rtm_msglen = sizeof(msg.rtm) + sizeof(*sin);

    // 发送 ARP 消息，如果失败则返回
    if (arp_msg(arp, &msg) < 0)
        return (-1);

    // 检查返回的消息长度，如果不符合要求则设置错误码并返回
    if (msg.rtm.rtm_msglen < (int)sizeof(msg.rtm) +
        sizeof(*sin) + sizeof(*sa)) {
        errno = EADDRNOTAVAIL;
        return (-1);
    }

    // 检查返回的 IP 地址是否与要添加的 IP 地址相同，如果不同则设置错误码并返回
    if (sin->sin_addr.s_addr == entry->arp_pa.addr_ip) {
        if ((msg.rtm.rtm_flags & RTF_LLINFO) == 0 ||
            (msg.rtm.rtm_flags & RTF_GATEWAY) != 0) {
            errno = EADDRINUSE;
            return (-1);
        }
    }

    // 检查 sa 的地址族是否为 AF_LINK，如果不是则设置错误码并返回；否则获取索引和类型
    if (sa->sa_family != AF_LINK) {
        errno = EADDRNOTAVAIL;
        return (-1);
    } else {
        index = ((struct sockaddr_dl *)sa)->sdl_index;
        type = ((struct sockaddr_dl *)sa)->sdl_type;
    }

    // 将 entry->arp_pa 和 entry->arp_ha 转换为网络地址字符串形式，如果失败则返回
    if (addr_ntos(&entry->arp_pa, (struct sockaddr *)sin) < 0 ||
        addr_ntos(&entry->arp_ha, sa) < 0)
        return (-1);

    // 设置 sa 的索引和类型
    ((struct sockaddr_dl *)sa)->sdl_index = index;
    ((struct sockaddr_dl *)sa)->sdl_type = type;

    // 初始化 ARP 消息结构体
    memset(&msg.rtm, 0, sizeof(msg.rtm));
    msg.rtm.rtm_type = RTM_ADD;
    msg.rtm.rtm_addrs = RTA_DST | RTA_GATEWAY;
    msg.rtm.rtm_inits = RTV_EXPIRE;
    msg.rtm.rtm_flags = RTF_HOST | RTF_STATIC;
#ifdef HAVE_SOCKADDR_SA_LEN
    msg.rtm.rtm_msglen = sizeof(msg.rtm) + sin->sin_len + sa->sa_len;
#else
    msg.rtm.rtm_msglen = sizeof(msg.rtm) + sizeof(*sin) + sizeof(*sa);
#endif
    return (arp_msg(arp, &msg));  // 发送 ARP 消息并返回结果
}

int
# 从 ARP 表中删除指定的条目
arp_delete(arp_t *arp, const struct arp_entry *entry)
{
    # 定义用于发送 ARP 消息的结构体
    struct arpmsg msg;
    # 定义用于存储 IP 地址的结构体指针
    struct sockaddr_in *sin;
    # 定义通用的套接字地址结构体指针
    struct sockaddr *sa;

    # 如果条目的地址类型不是 IP 地址，则设置错误码并返回 -1
    if (entry->arp_pa.addr_type != ADDR_TYPE_IP) {
        errno = EAFNOSUPPORT;
        return (-1);
    }
    # 将 sin 指向 msg.addrs 后面的地址
    sin = (struct sockaddr_in *)msg.addrs;
    sa = (struct sockaddr *)(sin + 1);

    # 将条目的 IP 地址转换为套接字地址结构体，如果失败则返回 -1
    if (addr_ntos(&entry->arp_pa, (struct sockaddr *)sin) < 0)
        return (-1);

    # 将 msg.rtm 清零，设置消息类型为获取路由表项，设置消息地址类型为目标地址，设置消息长度
    memset(&msg.rtm, 0, sizeof(msg.rtm));
    msg.rtm.rtm_type = RTM_GET;
    msg.rtm.rtm_addrs = RTA_DST;
    msg.rtm.rtm_msglen = sizeof(msg.rtm) + sizeof(*sin);
    
    # 如果发送 ARP 消息失败，则返回 -1
    if (arp_msg(arp, &msg) < 0)
        return (-1);
    
    # 如果消息长度小于指定长度，则设置错误码并返回 -1
    if (msg.rtm.rtm_msglen < (int)sizeof(msg.rtm) +
        sizeof(*sin) + sizeof(*sa)) {
        errno = ESRCH;
        return (-1);
    }
    # 如果目标地址与条目的 IP 地址相同，并且消息标志不包含 LLINFO 或者包含 GATEWAY，则设置错误码并返回 -1
    if (sin->sin_addr.s_addr == entry->arp_pa.addr_ip) {
        if ((msg.rtm.rtm_flags & RTF_LLINFO) == 0 ||
            (msg.rtm.rtm_flags & RTF_GATEWAY) != 0) {
            errno = EADDRINUSE;
            return (-1);
        }
    }
    # 如果 sa 的地址族不是 LINK 类型，则设置错误码并返回 -1
    if (sa->sa_family != AF_LINK) {
        errno = ESRCH;
        return (-1);
    }
    # 设置消息类型为删除路由表项
    msg.rtm.rtm_type = RTM_DELETE;
    
    # 返回发送 ARP 消息的结果
    return (arp_msg(arp, &msg));
}

# 从 ARP 表中获取指定条目的信息
int
arp_get(arp_t *arp, struct arp_entry *entry)
{
    # 定义用于发送 ARP 消息的结构体
    struct arpmsg msg;
    # 定义用于存储 IP 地址的结构体指针
    struct sockaddr_in *sin;
    # 定义通用的套接字地址结构体指针
    struct sockaddr *sa;
    
    # 如果条目的地址类型不是 IP 地址，则设置错误码并返回 -1
    if (entry->arp_pa.addr_type != ADDR_TYPE_IP) {
        errno = EAFNOSUPPORT;
        return (-1);
    }
    # 将 sin 指向 msg.addrs 后面的地址
    sin = (struct sockaddr_in *)msg.addrs;
    sa = (struct sockaddr *)(sin + 1);
    
    # 将条目的 IP 地址转换为套接字地址结构体，如果失败则返回 -1
    if (addr_ntos(&entry->arp_pa, (struct sockaddr *)sin) < 0)
        return (-1);
    
    # 将 msg.rtm 清零，设置消息类型为获取路由表项，设置消息地址类型为目标地址，设置消息标志为 LLINFO，设置消息长度
    memset(&msg.rtm, 0, sizeof(msg.rtm));
    msg.rtm.rtm_type = RTM_GET;
    msg.rtm.rtm_addrs = RTA_DST;
    msg.rtm.rtm_flags = RTF_LLINFO;
    msg.rtm.rtm_msglen = sizeof(msg.rtm) + sizeof(*sin);
    
    # 如果发送 ARP 消息失败，则返回 -1
    if (arp_msg(arp, &msg) < 0)
        return (-1);
    # 如果消息长度小于消息结构体的大小加上 sin 指针的大小和 sa 指针的大小，或者 sin 指向的地址不等于 entry 中的 arp_pa 地址，或者 sa 的地址族不等于 AF_LINK
    if (msg.rtm.rtm_msglen < (int)sizeof(msg.rtm) +
        sizeof(*sin) + sizeof(*sa) ||
        sin->sin_addr.s_addr != entry->arp_pa.addr_ip ||
        sa->sa_family != AF_LINK) {
        # 设置错误码为 ESRCH
        errno = ESRCH;
        # 返回 -1
        return (-1);
    }
    # 如果将地址结构体转换为字符串失败，返回 -1
    if (addr_ston(sa, &entry->arp_ha) < 0)
        return (-1);
    
    # 返回 0
    return (0);
#ifdef HAVE_SYS_SYSCTL_H
// 如果系统支持 sys/sysctl.h 头文件，则定义 arp_loop 函数
int
arp_loop(arp_t *arp, arp_handler callback, void *arg)
{
    // 定义需要使用的变量
    struct arp_entry entry;
    struct rt_msghdr *rtm;
    struct sockaddr_in *sin;
    struct sockaddr *sa;
    char *buf, *lim, *next;
    size_t len;
    int ret, mib[6] = { CTL_NET, PF_ROUTE, 0, AF_INET,
                NET_RT_FLAGS, RTF_LLINFO };

    // 获取 ARP 表项的数量
    if (sysctl(mib, 6, NULL, &len, NULL, 0) < 0)
        return (-1);

    // 如果 ARP 表项数量为 0，则返回 0
    if (len == 0)
        return (0);

    // 分配内存空间用于存储 ARP 表项数据
    if ((buf = malloc(len)) == NULL)
        return (-1);
    
    // 获取 ARP 表项数据
    if (sysctl(mib, 6, buf, &len, NULL, 0) < 0) {
        free(buf);
        return (-1);
    }
    lim = buf + len;
    ret = 0;
    
    // 遍历 ARP 表项数据
    for (next = buf; next < lim; next += rtm->rtm_msglen) {
        rtm = (struct rt_msghdr *)next;
        sin = (struct sockaddr_in *)(rtm + 1);
        sa = (struct sockaddr *)(sin + 1);
        
        // 将地址转换为网络字节顺序
        if (addr_ston((struct sockaddr *)sin, &entry.arp_pa) < 0 ||
            addr_ston(sa, &entry.arp_ha) < 0)
            continue;
        
        // 调用回调函数处理 ARP 表项数据
        if ((ret = callback(&entry, arg)) != 0)
            break;
    }
    // 释放内存空间
    free(buf);
    
    return (ret);
}
#else
// 如果系统不支持 sys/sysctl.h 头文件，则返回错误
int
arp_loop(arp_t *arp, arp_handler callback, void *arg)
{
    errno = ENOSYS;
    return (-1);
}
#endif

// 关闭 ARP 对象
arp_t *
arp_close(arp_t *arp)
{
    if (arp != NULL) {
        if (arp->fd >= 0)
            close(arp->fd);
        free(arp);
    }
    return (NULL);
}
```