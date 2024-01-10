# `nmap\libdnet-stripped\src\intf.c`

```
/*
 * intf.c
 *
 * 版权所有 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: intf.c 616 2006-01-09 07:09:49Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <sys/param.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#ifdef HAVE_SYS_SOCKIO_H
# include <sys/sockio.h>
#endif
/* XXX - AIX */
#ifdef HAVE_GETKERNINFO
#include <sys/ndd_var.h>
#include <sys/kinfo.h>
#endif
#ifndef IP_MULTICAST
# define IP_MULTICAST
#endif
#include <net/if.h>
#ifdef HAVE_NET_IF_VAR_H
# include <net/if_var.h>
#endif
#undef IP_MULTICAST
/* XXX - IPv6 ioctls */
#ifdef HAVE_NETINET_IN_VAR_H
# include <netinet/in.h>
# include <netinet/in_var.h>
#endif
#ifdef HAVE_NETINET_IN6_VAR_H
# include <sys/protosw.h>
# include <netinet/in6_var.h>
#endif

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

/* XXX - Tru64 */
#if defined(SIOCRIPMTU) && defined(SIOCSIPMTU)
# define SIOCGIFMTU    SIOCRIPMTU
# define SIOCSIFMTU    SIOCSIPMTU
#endif

/* XXX - HP-UX */
#if defined(SIOCADDIFADDR) && defined(SIOCDELIFADDR)
# define SIOCAIFADDR    SIOCADDIFADDR
# define SIOCDIFADDR    SIOCDELIFADDR
#endif

/* XXX - HP-UX, Solaris */
#if !defined(ifr_mtu) && defined(ifr_metric)
# define ifr_mtu    ifr_metric
#endif

#ifdef HAVE_SOCKADDR_SA_LEN
# define max(a, b) ((a) > (b) ? (a) : (b))
# define NEXTIFR(i)    ((struct ifreq *) \
                max((u_char *)i + sizeof(struct ifreq), \
                (u_char *)&i->ifr_addr + i->ifr_addr.sa_len))
#else
# define NEXTIFR(i)    (i + 1)
#endif

#define NEXTLIFR(i)    (i + 1)

/* XXX - superset of ifreq, for portable SIOC{A,D}IFADDR */
struct dnet_ifaliasreq {
    char        ifra_name[IFNAMSIZ];
    union {
        struct sockaddr ifrau_addr;
        int             ifrau_align;
    } ifra_ifrau;
#ifndef ifra_addr
#define ifra_addr      ifra_ifrau.ifrau_addr
#endif
    struct sockaddr ifra_brdaddr;
    struct sockaddr ifra_mask;
    int        ifra_cookie;    /* 定义一个整型变量 ifra_cookie，用于存储某种 cookie 值 */
};

// 定义一个结构体，用于存储接口的文件描述符和相关信息
struct intf_handle {
    int        fd;
    int        fd6;
    struct ifconf    ifc;
#ifdef SIOCGLIFCONF
    struct lifconf    lifc;
#endif
    u_char        ifcbuf[4192];
};

// 将接口标志转换为接口标志位
static int
intf_flags_to_iff(u_short flags, int iff)
{
    if (flags & INTF_FLAG_UP)
        iff |= IFF_UP;
    else
        iff &= ~IFF_UP;
    if (flags & INTF_FLAG_NOARP)
        iff |= IFF_NOARP;
    else
        iff &= ~IFF_NOARP;
    
    return (iff);
}

// 将接口标志位转换为接口标志
static u_int
intf_iff_to_flags(uint64_t iff)
{
    u_int n = 0;

    if (iff & IFF_UP)
        n |= INTF_FLAG_UP;    
    if (iff & IFF_LOOPBACK)
        n |= INTF_FLAG_LOOPBACK;
    if (iff & IFF_POINTOPOINT)
        n |= INTF_FLAG_POINTOPOINT;
    if (iff & IFF_NOARP)
        n |= INTF_FLAG_NOARP;
    if (iff & IFF_BROADCAST)
        n |= INTF_FLAG_BROADCAST;
    if (iff & IFF_MULTICAST)
        n |= INTF_FLAG_MULTICAST;
#ifdef IFF_IPMP
    /* Unset the BROADCAST and MULTICAST flags from Solaris IPMP interfaces,
     * otherwise _intf_set_type will think they are INTF_TYPE_ETH. */
    if (iff & IFF_IPMP)
        n &= ~(INTF_FLAG_BROADCAST | INTF_FLAG_MULTICAST);
#endif

    return (n);
}

// 打开接口
intf_t *
intf_open(void)
{
    intf_t *intf;
    int one = 1;
    
    // 分配内存给接口结构体
    if ((intf = calloc(1, sizeof(*intf))) != NULL) {
        intf->fd = intf->fd6 = -1;
        
        // 创建 IPv4 套接字
        if ((intf->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
            return (intf_close(intf));

        // 设置套接字选项，允许广播
        setsockopt(intf->fd, SOL_SOCKET, SO_BROADCAST,
            (const char *) &one, sizeof(one));

#if defined(SIOCGLIFCONF) || defined(SIOCGIFNETMASK_IN6) || defined(SIOCGIFNETMASK6)
        // 创建 IPv6 套接字
        if ((intf->fd6 = socket(AF_INET6, SOCK_DGRAM, 0)) < 0) {
#  ifdef EPROTONOSUPPORT
            if (errno != EPROTONOSUPPORT)
#  endif
                return (intf_close(intf));
        }
#endif
    }
    return (intf);
}

// 删除地址
static int
_intf_delete_addrs(intf_t *intf, struct intf_entry *entry)
{
#if defined(SIOCDIFADDR)
    struct dnet_ifaliasreq ifra;
    
    memset(&ifra, 0, sizeof(ifra));
    # 将 entry->intf_name 的内容复制到 ifra.ifra_name 中，长度为 sizeof(ifra.ifra_name)
    strlcpy(ifra.ifra_name, entry->intf_name, sizeof(ifra.ifra_name));
    # 如果 entry->intf_addr 的地址类型为 IP
    if (entry->intf_addr.addr_type == ADDR_TYPE_IP) {
        # 将 entry->intf_addr 转换为网络字节序的 IP 地址，存入 ifra.ifra_addr
        addr_ntos(&entry->intf_addr, &ifra.ifra_addr);
        # 调用 ioctl 函数，删除指定接口的 IP 地址
        ioctl(intf->fd, SIOCDIFADDR, &ifra);
    }
    # 如果 entry->intf_dst_addr 的地址类型为 IP
    if (entry->intf_dst_addr.addr_type == ADDR_TYPE_IP) {
        # 将 entry->intf_dst_addr 转换为网络字节序的 IP 地址，存入 ifra.ifra_addr
        addr_ntos(&entry->intf_dst_addr, &ifra.ifra_addr);
        # 调用 ioctl 函数，删除指定接口的 IP 地址
        ioctl(intf->fd, SIOCDIFADDR, &ifra);
    }
#elif defined(SIOCLIFREMOVEIF)
    // 如果定义了SIOCLIFREMOVEIF，则执行以下代码
    struct ifreq ifr;
    // 定义一个ifreq结构体变量ifr
    memset(&ifr, 0, sizeof(ifr));
    // 将ifr结构体变量的内存清零
    strlcpy(ifr.ifr_name, entry->intf_name, sizeof(ifr.ifr_name));
    // 将entry->intf_name的值复制到ifr.ifr_name中，确保不超过ifr.ifr_name的大小
    /* XXX - overloading Solaris lifreq with ifreq */
    // 注释：在Solaris中重载lifreq结构体与ifreq结构体
    ioctl(intf->fd, SIOCLIFREMOVEIF, &ifr);
    // 使用ioctl系统调用执行SIOCLIFREMOVEIF命令，传入ifr结构体的地址
#endif
    return (0);
    // 返回0
}

static int
_intf_delete_aliases(intf_t *intf, struct intf_entry *entry)
{
    // 删除接口的别名
    int i;
#if defined(SIOCDIFADDR) && !defined(__linux__)    /* XXX - see Linux below */
    // 如果定义了SIOCDIFADDR并且未定义__linux__，则执行以下代码
    struct dnet_ifaliasreq ifra;
    // 定义一个dnet_ifaliasreq结构体变量ifra
    memset(&ifra, 0, sizeof(ifra));
    // 将ifra结构体变量的内存清零
    strlcpy(ifra.ifra_name, entry->intf_name, sizeof(ifra.ifra_name));
    // 将entry->intf_name的值复制到ifra.ifra_name中，确保不超过ifra.ifra_name的大小
    for (i = 0; i < (int)entry->intf_alias_num; i++) {
        // 遍历接口的别名数量
        addr_ntos(&entry->intf_alias_addrs[i], &ifra.ifra_addr);
        // 将entry->intf_alias_addrs[i]的值转换为网络地址格式，并赋值给ifra.ifra_addr
        ioctl(intf->fd, SIOCDIFADDR, &ifra);
        // 使用ioctl系统调用执行SIOCDIFADDR命令，传入ifra结构体的地址
    }
#else
    // 如果不满足上述条件，则执行以下代码
    struct ifreq ifr;
    // 定义一个ifreq结构体变量ifr
    for (i = 0; i < entry->intf_alias_num; i++) {
        // 遍历接口的别名数量
        snprintf(ifr.ifr_name, sizeof(ifr.ifr_name), "%s:%d",
            entry->intf_name, i + 1);
        // 将"%s:%d"格式化为字符串，赋值给ifr.ifr_name
# ifdef SIOCLIFREMOVEIF
        // 如果定义了SIOCLIFREMOVEIF，则执行以下代码
        /* XXX - overloading Solaris lifreq with ifreq */
        // 注释：在Solaris中重载lifreq结构体与ifreq结构体
        ioctl(intf->fd, SIOCLIFREMOVEIF, &ifr);
        // 使用ioctl系统调用执行SIOCLIFREMOVEIF命令，传入ifr结构体的地址
# else
        // 如果不满足上述条件，则执行以下代码
        /* XXX - only need to set interface down on Linux */
        // 注释：只需要在Linux上将接口设置为关闭状态
        ifr.ifr_flags = 0;
        // 将ifr.ifr_flags的值设置为0
        ioctl(intf->fd, SIOCSIFFLAGS, &ifr);
        // 使用ioctl系统调用执行SIOCSIFFLAGS命令，传入ifr结构体的地址
# endif
    }
#endif
    return (0);
    // 返回0
}

static int
_intf_add_aliases(intf_t *intf, const struct intf_entry *entry)
{
    // 添加接口的别名
    int i;
#ifdef SIOCAIFADDR
    // 如果定义了SIOCAIFADDR，则执行以下代码
    struct dnet_ifaliasreq ifra;
    // 定义一个dnet_ifaliasreq结构体变量ifra
    struct addr bcast;
    // 定义一个addr结构体变量bcast
    memset(&ifra, 0, sizeof(ifra));
    // 将ifra结构体变量的内存清零
    strlcpy(ifra.ifra_name, entry->intf_name, sizeof(ifra.ifra_name));
    // 将entry->intf_name的值复制到ifra.ifra_name中，确保不超过ifra.ifra_name的大小
    # 遍历接口的所有别名
    for (i = 0; i < (int)entry->intf_alias_num; i++) {
        # 如果别名地址类型不是 IP 地址，则跳过当前循环，继续下一个别名
        if (entry->intf_alias_addrs[i].addr_type != ADDR_TYPE_IP)
            continue;
        
        # 将别名地址转换为网络字节序，并存储到 ifra.ifra_addr 中
        if (addr_ntos(&entry->intf_alias_addrs[i],
            &ifra.ifra_addr) < 0)
            return (-1);
        
        # 计算广播地址，并存储到 ifra.ifra_brdaddr 中
        addr_bcast(&entry->intf_alias_addrs[i], &bcast);
        addr_ntos(&bcast, &ifra.ifra_brdaddr);
        
        # 将别名地址的子网掩码转换为二进制形式，并存储到 ifra.ifra_mask 中
        addr_btos(entry->intf_alias_addrs[i].addr_bits,
            &ifra.ifra_mask);
        
        # 使用 SIOCAIFADDR 命令向接口添加别名地址
        if (ioctl(intf->fd, SIOCAIFADDR, &ifra) < 0)
            return (-1);
    }
#else
    # 定义一个 ifreq 结构体
    struct ifreq ifr;
    # 初始化变量 n 为 1
    int n = 1;
    
    # 遍历接口别名的数量
    for (i = 0; i < entry->intf_alias_num; i++) {
        # 如果别名地址类型不是 IP 地址，则跳过本次循环
        if (entry->intf_alias_addrs[i].addr_type != ADDR_TYPE_IP)
            continue;
        
        # 格式化接口名称和别名序号，存入 ifr.ifr_name
        snprintf(ifr.ifr_name, sizeof(ifr.ifr_name), "%s:%d",
            entry->intf_name, n++);
# ifdef SIOCLIFADDIF
        # 如果定义了 SIOCLIFADDIF，则调用 ioctl 添加接口别名
        if (ioctl(intf->fd, SIOCLIFADDIF, &ifr) < 0)
            return (-1);
# endif
        # 将别名地址转换为网络字节序，存入 ifr.ifr_addr，并调用 ioctl 设置接口地址
        if (addr_ntos(&entry->intf_alias_addrs[i], &ifr.ifr_addr) < 0)
            return (-1);
        if (ioctl(intf->fd, SIOCSIFADDR, &ifr) < 0)
            return (-1);
    }
    # 将 entry->intf_name 复制到 ifr.ifr_name
    strlcpy(ifr.ifr_name, entry->intf_name, sizeof(ifr.ifr_name));
#endif
    # 返回 0 表示设置接口成功
    return (0);
}

# 设置接口信息
int
intf_set(intf_t *intf, const struct intf_entry *entry)
{
    # 定义 ifreq 结构体、intf_entry 结构体、addr 结构体和缓冲区
    struct ifreq ifr;
    struct intf_entry *orig;
    struct addr bcast;
    u_char buf[BUFSIZ];
    
    # 将 entry 复制到 buf 中的 intf_entry 结构体 orig
    orig = (struct intf_entry *)buf;
    orig->intf_len = sizeof(buf);
    strcpy(orig->intf_name, entry->intf_name);
    
    # 获取原始接口信息
    if (intf_get(intf, orig) < 0)
        return (-1);
    
    # 删除任何现有的别名
    if (_intf_delete_aliases(intf, orig) < 0)
        return (-1);

    # 删除任何现有的地址
    if (_intf_delete_addrs(intf, orig) < 0)
        return (-1);
    
    # 清空 ifr 结构体
    memset(&ifr, 0, sizeof(ifr));
    # 将 entry->intf_name 复制到 ifr.ifr_name
    strlcpy(ifr.ifr_name, entry->intf_name, sizeof(ifr.ifr_name));
    
    # 设置接口 MTU
    if (entry->intf_mtu != 0) {
        ifr.ifr_mtu = entry->intf_mtu;
# ifdef SIOCSIFMTU
        # 如果定义了 SIOCSIFMTU，则调用 ioctl 设置接口 MTU
        if (ioctl(intf->fd, SIOCSIFMTU, &ifr) < 0)
#endif
            return (-1);
    }
    # 设置接口地址
    if (entry->intf_addr.addr_type == ADDR_TYPE_IP) {
# if defined(BSD) && !defined(__OPENBSD__)
        # 在调用 SIOCSIFADDR 之前，将地址转换为网络字节序，并设置接口子网掩码
        if (addr_btos(entry->intf_addr.addr_bits,
            &ifr.ifr_addr) == 0) {
            if (ioctl(intf->fd, SIOCSIFNETMASK, &ifr) < 0)
                return (-1);
        }
    # 如果地址转换失败，返回-1
    if (addr_ntos(&entry->intf_addr, &ifr.ifr_addr) < 0)
        return (-1);
    # 如果设置接口地址失败并且错误码不是EEXIST，返回-1
    if (ioctl(intf->fd, SIOCSIFADDR, &ifr) < 0 && errno != EEXIST)
        return (-1);
    
    # 如果地址转换成功并且是Linux系统并且接口地址不为0
    if (addr_btos(entry->intf_addr.addr_bits, &ifr.ifr_addr) == 0
#ifdef __linux__
        && entry->intf_addr.addr_ip != 0
#endif
        ) {
        # 如果设置接口子网掩码失败，返回-1
        if (ioctl(intf->fd, SIOCSIFNETMASK, &ifr) < 0)
            return (-1);
    }
    # 如果广播地址转换成功
    if (addr_bcast(&entry->intf_addr, &bcast) == 0) {
        # 如果广播地址转换成功
        if (addr_ntos(&bcast, &ifr.ifr_broadaddr) == 0) {
            # 忽略非广播接口的错误
            ioctl(intf->fd, SIOCSIFBRDADDR, &ifr);
        }
    }
    # 如果接口类型为以太网并且接口地址与原始地址不同
    if (entry->intf_link_addr.addr_type == ADDR_TYPE_ETH &&
        addr_cmp(&entry->intf_link_addr, &orig->intf_link_addr) != 0) {
        # 如果支持SIOCSIFHWADDR
        # 如果地址转换失败，返回-1
        if (addr_ntos(&entry->intf_link_addr, &ifr.ifr_hwaddr) < 0)
            return (-1);
        # 如果设置接口硬件地址失败，返回-1
        if (ioctl(intf->fd, SIOCSIFHWADDR, &ifr) < 0)
            return (-1);
    # 如果支持SIOCSIFLLADDR
    # 复制以太网地址到ifr.ifr_addr.sa_data
    # 设置以太网地址长度
    # 如果设置接口链路地址失败，返回-1
    # 否则
    # 打开以太网接口
    # 如果设置以太网地址失败
    # 关闭以太网接口
    # 关闭以太网接口
    # 如果接口目的地址类型为IP
    # 如果地址转换失败，返回-1
    # 如果设置接口目的地址失败并且错误码不是EEXIST，返回-1
    # 添加别名
    # 如果添加接口别名失败，则返回-1
    if (_intf_add_aliases(intf, entry) < 0)
        return (-1);
    
    # 获取接口的标志位
    if (ioctl(intf->fd, SIOCGIFFLAGS, &ifr) < 0)
        return (-1);
    
    # 根据给定的接口标志位设置接口的标志位
    ifr.ifr_flags = intf_flags_to_iff(entry->intf_flags, ifr.ifr_flags);
    
    # 设置接口的标志位
    if (ioctl(intf->fd, SIOCSIFFLAGS, &ifr) < 0)
        return (-1);
    
    # 返回成功
    return (0);
/* XXX - this is total crap. how to do this without walking ifnet? */
// 这段代码很糟糕。如何在不遍历 ifnet 的情况下完成这个任务？

static void
_intf_set_type(struct intf_entry *entry)
{
    // 如果接口标志中包含 INTF_FLAG_LOOPBACK，则设置接口类型为 INTF_TYPE_LOOPBACK
    if ((entry->intf_flags & INTF_FLAG_LOOPBACK) != 0)
        entry->intf_type = INTF_TYPE_LOOPBACK;
    // 如果接口标志中包含 INTF_FLAG_BROADCAST，则设置接口类型为 INTF_TYPE_ETH
    else if ((entry->intf_flags & INTF_FLAG_BROADCAST) != 0)
        entry->intf_type = INTF_TYPE_ETH;
    // 如果接口标志中包含 INTF_FLAG_POINTOPOINT，则设置接口类型为 INTF_TYPE_TUN
    else if ((entry->intf_flags & INTF_FLAG_POINTOPOINT) != 0)
        entry->intf_type = INTF_TYPE_TUN;
    // 否则，设置接口类型为 INTF_TYPE_OTHER
    else
        entry->intf_type = INTF_TYPE_OTHER;
}

#ifdef SIOCGLIFCONF
int
_intf_get_noalias(intf_t *intf, struct intf_entry *entry)
{
    struct lifreq lifr;
    int fd;

    /* Get interface index. */
    // 获取接口索引
    entry->intf_index = if_nametoindex(entry->intf_name);
    // 如果接口索引为 0，则返回 -1
    if (entry->intf_index == 0)
        return (-1);

    // 将接口名拷贝到 lifreq 结构体中
    strlcpy(lifr.lifr_name, entry->intf_name, sizeof(lifr.lifr_name));

    /* Get interface flags. Here he also check whether we need to use fd or
     * fd6 in the rest of the function. Using the wrong address family in
     * the ioctls gives ENXIO on Solaris. */
    // 获取接口标志。在这里还要检查在函数的其余部分是否需要使用 fd 还是 fd6。在 Solaris 上，使用错误的地址族在 ioctls 中会导致 ENXIO 错误。
    if (ioctl(intf->fd, SIOCGLIFFLAGS, &lifr) >= 0)
        fd = intf->fd;
    else if (intf->fd6 != -1 && ioctl(intf->fd6, SIOCGLIFFLAGS, &lifr) >= 0)
        fd = intf->fd6;
    else
        return (-1);
    
    // 将 lifreq 结构体中的标志转换为接口标志，并设置接口类型
    entry->intf_flags = intf_iff_to_flags(lifr.lifr_flags);
    _intf_set_type(entry);
    
    /* Get interface MTU. */
#ifdef SIOCGLIFMTU
    // 获取接口 MTU
    if (ioctl(fd, SIOCGLIFMTU, &lifr) < 0)
#endif
        return (-1);
    entry->intf_mtu = lifr.lifr_mtu;

    // 将接口地址类型设置为 ADDR_TYPE_NONE
    entry->intf_addr.addr_type = entry->intf_dst_addr.addr_type =
        entry->intf_link_addr.addr_type = ADDR_TYPE_NONE;
    
    /* Get primary interface address. */
    // 获取主要接口地址
    if (ioctl(fd, SIOCGLIFADDR, &lifr) == 0) {
        // 将 lifr 结构体中的地址转换为接口地址
        addr_ston((struct sockaddr *)&lifr.lifr_addr, &entry->intf_addr);
        // 如果获取网络掩码失败，则返回 -1
        if (ioctl(fd, SIOCGLIFNETMASK, &lifr) < 0)
            return (-1);
        // 将 lifr 结构体中的地址转换为接口地址的位数
        addr_stob((struct sockaddr *)&lifr.lifr_addr, &entry->intf_addr.addr_bits);
    }
    // 获取其他地址。
}
    # 如果接口类型为TUN
    if (entry->intf_type == INTF_TYPE_TUN) {
        # 如果成功获取目的地址
        if (ioctl(fd, SIOCGLIFDSTADDR, &lifr) == 0) {
            # 如果地址转换失败，返回-1
            if (addr_ston((struct sockaddr *)&lifr.lifr_addr,
                &entry->intf_dst_addr) < 0)
                return (-1);
        }
    } 
    # 如果接口类型为ETH
    else if (entry->intf_type == INTF_TYPE_ETH) {
        eth_t *eth;
        # 打开以太网接口
        if ((eth = eth_open(entry->intf_name)) != NULL) {
            # 如果成功获取以太网地址
            if (!eth_get(eth, &entry->intf_link_addr.addr_eth)) {
                entry->intf_link_addr.addr_type =
                    ADDR_TYPE_ETH;
                entry->intf_link_addr.addr_bits =
                    ETH_ADDR_BITS;
            }
            # 关闭以太网接口
            eth_close(eth);
        }
    }
    # 返回0表示成功
    return (0);
# 如果不是 C 语言，返回获取不支持别名的接口信息
#else
static int
_intf_get_noalias(intf_t *intf, struct intf_entry *entry)
{
    struct ifreq ifr;
#ifdef HAVE_GETKERNINFO
  int size;
  struct kinfo_ndd *nddp;
  void *end;
#endif

    # 获取接口索引
    entry->intf_index = if_nametoindex(entry->intf_name);
    如果接口索引为 0，返回错误
    if (entry->intf_index == 0)
        return (-1);

    将接口名拷贝到 ifr.ifr_name 中
    strlcpy(ifr.ifr_name, entry->intf_name, sizeof(ifr.ifr_name));

    # 获取接口标志
    如果 ioctl 调用失败，返回错误
    if (ioctl(intf->fd, SIOCGIFFLAGS, &ifr) < 0)
        return (-1);
    
    将接口标志转换为标志位
    entry->intf_flags = intf_iff_to_flags(ifr.ifr_flags);
    设置接口类型
    _intf_set_type(entry);
    
    # 获取接口 MTU
    #ifdef SIOCGIFMTU
    如果 ioctl 调用失败，返回错误
    if (ioctl(intf->fd, SIOCGIFMTU, &ifr) < 0)
    返回 (-1);
    entry->intf_mtu = ifr.ifr_mtu;

    将接口地址类型设置为 ADDR_TYPE_NONE
    entry->intf_addr.addr_type = entry->intf_dst_addr.addr_type =
        entry->intf_link_addr.addr_type = ADDR_TYPE_NONE;
    
    # 获取主要接口地址
    如果 ioctl 调用成功
    if (ioctl(intf->fd, SIOCGIFADDR, &ifr) == 0) {
        将地址转换为字符串形式
        addr_ston(&ifr.ifr_addr, &entry->intf_addr);
        如果获取子网掩码失败，返回错误
        if (ioctl(intf->fd, SIOCGIFNETMASK, &ifr) < 0)
            return (-1);
        将子网掩码转换为位数形式
        addr_stob(&ifr.ifr_addr, &entry->intf_addr.addr_bits);
    }
    # 获取其他地址
    如果接口类型为 INTF_TYPE_TUN
    if (entry->intf_type == INTF_TYPE_TUN) {
        如果 ioctl 调用成功
        if (ioctl(intf->fd, SIOCGIFDSTADDR, &ifr) == 0) {
            如果地址转换失败，返回错误
            if (addr_ston(&ifr.ifr_addr,
                &entry->intf_dst_addr) < 0)
                return (-1);
        }
    } else 如果接口类型为 INTF_TYPE_ETH
    else if (entry->intf_type == INTF_TYPE_ETH) {
    #if defined(HAVE_GETKERNINFO)
      # AIX 也定义了 SIOCGIFHWADDR，但它会悄悄失败？
      # 这是 IBM 在这里推荐的方法：
      # http://www-01.ibm.com/support/knowledgecenter/ssw_aix_53/com.ibm.aix.progcomm/doc/progcomc/skt_sndother_ex.htm%23ssqinc2joyc?lang=en
      # 返回多少字节？
    size = getkerninfo(KINFO_NDD, 0, 0, 0);
    如果返回值小于等于 0，返回错误
    if (size <= 0) {
      return -1;
    }
    分配内存空间
    nddp = (struct kinfo_ndd *)malloc(size);

    如果分配失败，返回错误
    if (!nddp) {
      return -1;
    }
    /* 获取所有网络设备驱动程序（NDD）的信息 */
    if (getkerninfo(KINFO_NDD, nddp, &size, 0) < 0) {
      free(nddp);
      return -1;
    }
    /* 循环遍历返回的数值，直到找到匹配项 */
    end = (void *)nddp + size;
    while ((void *)nddp < end) {
      if (!strcmp(nddp->ndd_alias, entry->intf_name) ||
          !strcmp(nddp->ndd_name, entry->intf_name)) {
        addr_pack(&entry->intf_link_addr, ADDR_TYPE_ETH, ETH_ADDR_BITS,
            nddp->ndd_addr, ETH_ADDR_LEN);
        break;
      } else
        nddp++;
    }
    free(nddp);
#ifdef SIOCGIFHWADDR
        // 如果定义了SIOCGIFHWADDR，则执行以下代码块
        if (ioctl(intf->fd, SIOCGIFHWADDR, &ifr) < 0)
            // 如果获取接口的硬件地址失败，返回-1
            return (-1);
        if (addr_ston(&ifr.ifr_addr, &entry->intf_link_addr) < 0) {
          /* Likely we got an unsupported address type. Just use NONE for now. */
          // 如果转换地址失败，将地址类型设置为NONE，地址位数设置为0
          entry->intf_link_addr.addr_type = ADDR_TYPE_NONE;
          entry->intf_link_addr.addr_bits = 0;
    }
#elif defined(SIOCRPHYSADDR)
        // 如果定义了SIOCRPHYSADDR，则执行以下代码块
        /* Tru64 */
        // 定义一个ifdevea结构体指针，并将其指向ifr的地址
        struct ifdevea *ifd = (struct ifdevea *)&ifr; /* XXX */
        
        if (ioctl(intf->fd, SIOCRPHYSADDR, ifd) < 0)
            // 如果获取接口的物理地址失败，返回-1
            return (-1);
        // 将获取到的物理地址打包到entry的intf_link_addr中
        addr_pack(&entry->intf_link_addr, ADDR_TYPE_ETH, ETH_ADDR_BITS,
            ifd->current_pa, ETH_ADDR_LEN);
#else
        // 如果都未定义，则执行以下代码块
        eth_t *eth;
        // 打开一个以太网接口
        if ((eth = eth_open(entry->intf_name)) != NULL) {
            // 获取以太网接口的地址，并存储到entry的intf_link_addr中
            if (!eth_get(eth, &entry->intf_link_addr.addr_eth)) {
                entry->intf_link_addr.addr_type =
                    ADDR_TYPE_ETH;
                entry->intf_link_addr.addr_bits =
                    ETH_ADDR_BITS;
            }
            // 关闭以太网接口
            eth_close(eth);
        }
#endif
    }
    // 返回0表示成功
    return (0);
}
#endif

#ifdef SIOCLIFADDR
/* XXX - aliases on IRIX don't show up in SIOCGIFCONF */
static int
_intf_get_aliases(intf_t *intf, struct intf_entry *entry)
{
    // 定义一个ifaliasreq结构体
    struct dnet_ifaliasreq ifra;
    struct addr *ap, *lap;
    
    // 将entry的接口名拷贝到ifra的ifra_name中
    strlcpy(ifra.ifra_name, entry->intf_name, sizeof(ifra.ifra_name));
    // 将entry的接口地址转换为网络字节序，并存储到ifra的ifra_addr中
    addr_ntos(&entry->intf_addr, &ifra.ifra_addr);
    // 将entry的接口地址位数转换为网络字节序，并存储到ifra的ifra_mask中
    addr_btos(entry->intf_addr.addr_bits, &ifra.ifra_mask);
    // 将ifra的ifra_brdaddr清零
    memset(&ifra.ifra_brdaddr, 0, sizeof(ifra.ifra_brdaddr));
    // 设置ifra的ifra_cookie为1
    ifra.ifra_cookie = 1;

    // 定义ap指向entry的intf_alias_addrs，lap指向entry的intf_len后面的地址
    ap = entry->intf_alias_addrs;
    lap = (struct addr *)((u_char *)entry + entry->intf_len);
    
    // 循环获取接口的别名地址，并存储到entry的intf_alias_addrs中
    while (ioctl(intf->fd, SIOCLIFADDR, &ifra) == 0 &&
        ifra.ifra_cookie > 0 && (ap + 1) < lap) {
        if (addr_ston(&ifra.ifra_addr, ap) < 0)
            break;
        ap++, entry->intf_alias_num++;
    }
    // 更新entry的intf_len
    entry->intf_len = (u_char *)ap - (u_char *)entry;
    
    // 返回0表示成功
    return (0);
}
    # 如果定义了 SIOCGLIFCONF，则定义 _intf_get_aliases 函数
static int
_intf_get_aliases(intf_t *intf, struct intf_entry *entry)
{
    # 定义 lifreq 结构体指针
    struct lifreq *lifr, *llifr;
    # 定义 tmplifr 结构体
    struct lifreq tmplifr;
    # 定义地址指针
    struct addr *ap, *lap;
    # 定义字符指针
    char *p;
    
    # 如果接口的 lifc_len 小于 lifreq 结构体的大小，则返回错误
    if (intf->lifc.lifc_len < (int)sizeof(*lifr)) {
        errno = EINVAL;
        return (-1);
    }
    # 将接口的别名数量设置为 0
    entry->intf_alias_num = 0;
    # 将接口的别名地址指针指向 entry 结构体中的 intf_alias_addrs
    ap = entry->intf_alias_addrs;
    # 计算 llifr 指针的位置
    llifr = (struct lifreq *)intf->lifc.lifc_buf + 
        (intf->lifc.lifc_len / sizeof(*llifr));
    # 计算 lap 指针的位置
    lap = (struct addr *)((u_char *)entry + entry->intf_len);
    
    /* 获取该接口的地址 */
    for (lifr = intf->lifc.lifc_req; lifr < llifr && (ap + 1) < lap;
        lifr = NEXTLIFR(lifr)) {
        /* 遍历接口请求列表，直到达到列表末尾或者达到最大别名数 */
        /* XXX - Linux, Solaris ifaliases */
        /* 如果接口名中包含冒号，将冒号替换为字符串结束符 */
        if ((p = strchr(lifr->lifr_name, ':')) != NULL)
            *p = '\0';
        
        /* 如果当前接口名不等于给定接口名，且存在冒号，则将冒号替换回去并继续下一次循环 */
        if (strcmp(lifr->lifr_name, entry->intf_name) != 0) {
            if (p) *p = ':';
            continue;
        }
        
        /* 如果存在冒号，将冒号替换回去 */
        if (p) *p = ':';

        /* 将接口地址转换为字符串形式，并存储到 entry 结构中 */
        if (addr_ston((struct sockaddr *)&lifr->lifr_addr, ap) < 0)
            continue;
        
        /* 如果地址类型为以太网类型，将地址拷贝到 entry 结构中，并继续下一次循环 */
        /* 否则，如果地址类型为 IP 类型，进行一系列判断和操作 */
        /* 否则，如果地址类型为 IPv6 类型，进行一系列判断和操作 */
        ap++, entry->intf_alias_num++;
    }
    /* 计算接口长度，并存储到 entry 结构中 */
    entry->intf_len = (u_char *)ap - (u_char *)entry;
    
    /* 返回 0 表示成功 */
    return (0);
}  // 结束 else 分支的代码块

#else  // 如果没有定义宏，则执行以下代码

static int  // 定义静态整型函数_intf_get_aliases，接受 intf_t 类型指针和 struct intf_entry 类型指针参数
_intf_get_aliases(intf_t *intf, struct intf_entry *entry)
{
    struct ifreq *ifr, *lifr;  // 定义 ifreq 结构体指针 ifr 和 lifr
    struct ifreq tmpifr;  // 定义 ifreq 结构体 tmpifr
    struct addr *ap, *lap;  // 定义 addr 结构体指针 ap 和 lap
    char *p;  // 定义字符指针 p

    // 如果接口缓冲区长度小于 ifreq 结构体的大小，并且长度不为 0，则设置错误码并返回 -1
    if (intf->ifc.ifc_len < (int)sizeof(*ifr) && intf->ifc.ifc_len != 0) {
        errno = EINVAL;
        return (-1);
    }
    // 将 intf_alias_num 设置为 0
    entry->intf_alias_num = 0;
    // 将 intf_alias_addrs 赋值给 ap
    ap = entry->intf_alias_addrs;
    // 将 intf 缓冲区中的 ifreq 结构体数量赋值给 lifr
    lifr = (struct ifreq *)intf->ifc.ifc_buf + 
        (intf->ifc.ifc_len / sizeof(*lifr));
    // 将 entry 的 intf_len 字节偏移后的地址赋值给 lap
    
    /* 获取此接口的地址。*/
    for (ifr = intf->ifc.ifc_req; ifr < lifr && (ap + 1) < lap;
        ifr = NEXTIFR(ifr)) {
        /* XXX - Linux, Solaris ifaliases */
        if ((p = strchr(ifr->ifr_name, ':')) != NULL)
            *p = '\0';
        
        if (strcmp(ifr->ifr_name, entry->intf_name) != 0) {
            if (p) *p = ':';
            continue;
        }
        
        /* 修复名称 */
        if (p) *p = ':';

        if (addr_ston(&ifr->ifr_addr, ap) < 0)
            continue;
        
        /* XXX */
        if (ap->addr_type == ADDR_TYPE_ETH) {
            memcpy(&entry->intf_link_addr, ap, sizeof(*ap));
            continue;
        } else if (ap->addr_type == ADDR_TYPE_IP) {
            if (ap->addr_ip == entry->intf_addr.addr_ip ||
                ap->addr_ip == entry->intf_dst_addr.addr_ip)
                continue;
            strlcpy(tmpifr.ifr_name, ifr->ifr_name, sizeof(tmpifr.ifr_name));
            if (ioctl(intf->fd, SIOCGIFNETMASK, &tmpifr) == 0)
                addr_stob(&tmpifr.ifr_addr, &ap->addr_bits);
        }
#ifdef SIOCGIFNETMASK_IN6
        // 如果支持获取 IPv6 掩码信息，并且接口文件描述符有效
        else if (ap->addr_type == ADDR_TYPE_IP6 && intf->fd6 != -1) {
            // 定义 IPv6 掩码信息结构体
            struct in6_ifreq ifr6;

            /* XXX - sizeof(ifr) < sizeof(ifr6) */
            // 将 ifr 结构体的内容拷贝到 ifr6 结构体中
            memcpy(&ifr6, ifr, sizeof(ifr6));
            
            // 获取接口的 IPv6 掩码信息
            if (ioctl(intf->fd6, SIOCGIFNETMASK_IN6, &ifr6) == 0) {
                // 将获取到的地址信息转换为二进制格式
                addr_stob((struct sockaddr *)&ifr6.ifr_addr,
                    &ap->addr_bits);
            }
            else perror("SIOCGIFNETMASK_IN6");
        }
#else
#ifdef SIOCGIFNETMASK6
        // 如果支持获取 IPv6 掩码信息，并且接口文件描述符有效
        else if (ap->addr_type == ADDR_TYPE_IP6 && intf->fd6 != -1) {
            // 定义 IPv6 掩码信息结构体
            struct in6_ifreq ifr6;

            /* XXX - sizeof(ifr) < sizeof(ifr6) */
            // 将 ifr 结构体的内容拷贝到 ifr6 结构体中
            memcpy(&ifr6, ifr, sizeof(ifr6));
            
            // 获取接口的 IPv6 掩码信息
            if (ioctl(intf->fd6, SIOCGIFNETMASK6, &ifr6) == 0) {
                // 设置地址族为 AF_INET6
                ifr6.ifr_Addr.sin6_family = AF_INET6;
                // 将获取到的地址信息转换为二进制格式
                addr_stob((struct sockaddr *)&ifr6.ifr_Addr,
                    &ap->addr_bits);
            }
            else perror("SIOCGIFNETMASK6");
        }
#endif
#endif
        // 移动指针并增加接口别名数
        ap++, entry->intf_alias_num++;
    }
#ifdef HAVE_LINUX_PROCFS
#define PROC_INET6_FILE    "/proc/net/if_inet6"
    {
        // 声明文件指针
        FILE *f;
        // 声明字符数组和变量
        char buf[256], s[8][5], name[INTF_NAME_LEN];
        u_int idx, bits, scope, flags;
        
        // 如果成功打开文件
        if ((f = fopen(PROC_INET6_FILE, "r")) != NULL) {
            // 当指针 ap 小于 lap 并且成功读取一行内容
            while (ap < lap &&
                   fgets(buf, sizeof(buf), f) != NULL) {
                /* scan up to INTF_NAME_LEN-1 bytes to reserve space for null terminator */
                // 从 buf 中按指定格式读取数据到变量中
                sscanf(buf, "%04s%04s%04s%04s%04s%04s%04s%04s %x %02x %02x %02x %15s\n",
                    s[0], s[1], s[2], s[3], s[4], s[5], s[6], s[7],
                    &idx, &bits, &scope, &flags, name);
                // 如果读取的接口名与 entry->intf_name 相同
                if (strcmp(name, entry->intf_name) == 0) {
                    // 格式化 IP 地址字符串并转换为二进制形式
                    snprintf(buf, sizeof(buf), "%s:%s:%s:%s:%s:%s:%s:%s/%d",
                        s[0], s[1], s[2], s[3], s[4], s[5], s[6], s[7], bits);
                    addr_aton(buf, ap);
                    // 指针 ap 自增，entry->intf_alias_num 自增
                    ap++, entry->intf_alias_num++;
                }
            }
            // 关闭文件
            fclose(f);
        }
    }
#endif
    # 计算 entry 结构体中 intf_len 字段的值
    entry->intf_len = (u_char *)ap - (u_char *)entry;
    
    # 返回 0
    return (0);
}
#endif /* SIOCLIFADDR */

# 从 intf 结构体中获取接口信息
int
intf_get(intf_t *intf, struct intf_entry *entry):
    # 如果 _intf_get_noalias 函数返回值小于 0，则返回 -1
    if (_intf_get_noalias(intf, entry) < 0)
        return (-1);
    # 如果未定义 SIOCLIFADDR，则执行以下代码
#ifndef SIOCLIFADDR
    # 设置 intf->ifc.ifc_buf 的值为 intf->ifcbuf 的地址
    intf->ifc.ifc_buf = (caddr_t)intf->ifcbuf;
    # 设置 intf->ifc.ifc_len 的值为 intf->ifcbuf 的大小
    intf->ifc.ifc_len = sizeof(intf->ifcbuf);
    
    # 如果 ioctl 函数返回值小于 0，则返回 -1
    if (ioctl(intf->fd, SIOCGIFCONF, &intf->ifc) < 0)
        return (-1);
#endif
    # 返回 _intf_get_aliases 函数的返回值
    return (_intf_get_aliases(intf, entry));
}

# 通过索引查找接口信息
int
intf_get_index(intf_t *intf, struct intf_entry *entry, int af, unsigned int index):
    char namebuf[IFNAMSIZ];
    char *devname;

    # 忽略 af 参数，仅在 intf-win32.c 中使用
    devname = if_indextoname(index, namebuf);
    # 如果 devname 为 NULL，则返回 -1
    if (devname == NULL)
        return (-1);
    # 将 devname 复制到 entry->intf_name 中
    strlcpy(entry->intf_name, devname, sizeof(entry->intf_name));
    # 返回 intf_get 函数的返回值
    return intf_get(intf, entry);
}

# 匹配接口源地址
static int
_match_intf_src(const struct intf_entry *entry, void *arg):
    struct intf_entry *save = (struct intf_entry *)arg;
    int matched = 0, cnt;
    
    # 如果 entry 中 intf_addr 的地址类型为 ADDR_TYPE_IP，并且 intf_addr 的 IP 地址与 save 中 intf_addr 的 IP 地址相同，则将 matched 设置为 1
    if (entry->intf_addr.addr_type == ADDR_TYPE_IP &&
        entry->intf_addr.addr_ip == save->intf_addr.addr_ip)
        matched = 1;

    # 遍历 entry 中的 intf_alias_addrs 数组，如果找到与 save 中 intf_addr 的 IP 地址相同的地址，则将 matched 设置为 1
    for (cnt = 0; !matched && cnt < (int) entry->intf_alias_num; cnt++):
        if (entry->intf_alias_addrs[cnt].addr_type != ADDR_TYPE_IP)
            continue
        if (entry->intf_alias_addrs[cnt].addr_ip == save->intf_addr.addr_ip)
            matched = 1

    # 如果匹配成功，则将 entry 复制到 save 中，并返回 1
    if (matched):
        # 如果 save 的 intf_len 小于 entry 的 intf_len，则将 entry 复制到 save 中
        if (save->intf_len < entry->intf_len)
            memcpy(save, entry, save->intf_len)
        else
            memcpy(save, entry, entry->intf_len)
        return (1)
    # 如果匹配失败，则返回 0
    return (0)

# 通过源地址获取接口信息
int
intf_get_src(intf_t *intf, struct intf_entry *entry, struct addr *src):
    # 将 src 的内容复制到 entry->intf_addr 中
    memcpy(&entry->intf_addr, src, sizeof(*src));
    # 如果调用intf_loop函数返回值不等于1，则执行下面的代码块
    if (intf_loop(intf, _match_intf_src, entry) != 1) {
        # 设置错误码为ENXIO
        errno = ENXIO;
        # 返回-1
        return (-1);
    }
    # 如果调用intf_loop函数返回值等于1，则执行下面的代码块
    # 返回0
    return (0);
}
// 结束函数定义

int
intf_get_dst(intf_t *intf, struct intf_entry *entry, struct addr *dst)
{
    // 定义变量
    struct sockaddr_in sin;
    socklen_t n;
    
    // 检查目标地址类型是否为IP，如果不是则返回错误
    if (dst->addr_type != ADDR_TYPE_IP) {
        errno = EINVAL;
        return (-1);
    }
    // 将目标地址转换为网络字节序的IP地址
    addr_ntos(dst, (struct sockaddr *)&sin);
    // 设置目标地址的端口号为666
    sin.sin_port = htons(666);
    
    // 连接到目标地址
    if (connect(intf->fd, (struct sockaddr *)&sin, sizeof(sin)) < 0)
        return (-1);
    
    // 获取本地套接字地址
    n = sizeof(sin);
    if (getsockname(intf->fd, (struct sockaddr *)&sin, &n) < 0)
        return (-1);
    
    // 将本地套接字地址转换为地址结构
    addr_ston((struct sockaddr *)&sin, &entry->intf_addr);
    
    // 在接口列表中查找与给定条件匹配的接口
    if (intf_loop(intf, _match_intf_src, entry) != 1)
        return (-1);
    
    return (0);
}

#ifdef HAVE_LINUX_PROCFS
#define PROC_DEV_FILE    "/proc/net/dev"

int
intf_loop(intf_t *intf, intf_handler callback, void *arg)
{
    // 定义变量
    FILE *fp;
    struct intf_entry *entry;
    char *p, buf[BUFSIZ], ebuf[BUFSIZ];
    int ret;

    // 将ebuf强制转换为struct intf_entry类型
    entry = (struct intf_entry *)ebuf;
    
    // 打开/proc/net/dev文件
    if ((fp = fopen(PROC_DEV_FILE, "r")) == NULL)
        return (-1);
    
    // 设置接口缓冲区
    intf->ifc.ifc_buf = (caddr_t)intf->ifcbuf;
    intf->ifc.ifc_len = sizeof(intf->ifcbuf);
    
    // 获取接口配置信息
    if (ioctl(intf->fd, SIOCGIFCONF, &intf->ifc) < 0) {
        fclose(fp);
        return (-1);
    }

    // 初始化返回值
    ret = 0;
    // 逐行读取/proc/net/dev文件
    while (fgets(buf, sizeof(buf), fp) != NULL) {
        // 查找冒号位置
        if ((p = strchr(buf, ':')) == NULL)
            continue;
        *p = '\0';
        // 跳过空格
        for (p = buf; *p == ' '; p++)
            ;

        // 清空ebuf，将接口名拷贝到entry中
        memset(ebuf, 0, sizeof(ebuf));
        strlcpy(entry->intf_name, p, sizeof(entry->intf_name));
        entry->intf_len = sizeof(ebuf);
        
        // 获取接口的非别名地址
        if (_intf_get_noalias(intf, entry) < 0) {
            ret = -1;
            break;
        }
        // 获取接口的别名地址
        if (_intf_get_aliases(intf, entry) < 0) {
            ret = -1;
            break;
        }
        // 调用回调函数处理接口信息
        if ((ret = (*callback)(entry, arg)) != 0)
            break;
    }
    // 检查文件读取错误
    if (ferror(fp))
        ret = -1;
    
    // 关闭文件
    fclose(fp);
    
    return (ret);
}
#elif defined(SIOCGLIFCONF)
int
intf_loop(intf_t *intf, intf_handler callback, void *arg)
{
    struct intf_entry *entry;
    struct lifreq *lifr, *llifr, *plifr;
    char *p, ebuf[BUFSIZ];
    int ret;
    struct lifreq lifrflags;
    memset(&lifrflags, 0, sizeof(struct lifreq));

    entry = (struct intf_entry *)ebuf;

    /* 设置接口配置结构体的地址族为未指定 */
    intf->lifc.lifc_family = AF_UNSPEC;
    /* 设置接口配置结构体的标志为0 */
    intf->lifc.lifc_flags = 0;
    /* 如果定义了LIFC_UNDER_IPMP宏，则将接口配置结构体的标志设置为LIFC_UNDER_IPMP */
#ifdef LIFC_UNDER_IPMP
    intf->lifc.lifc_flags |= LIFC_UNDER_IPMP;
#endif
    /* 设置接口配置结构体的缓冲区为接口缓冲区 */
    intf->lifc.lifc_buf = (caddr_t)intf->ifcbuf;
    /* 设置接口配置结构体的长度为接口缓冲区的大小 */
    intf->lifc.lifc_len = sizeof(intf->ifcbuf);
    
    /* 使用SIOCGLIFCONF命令获取接口列表 */
    if (ioctl(intf->fd, SIOCGLIFCONF, &intf->lifc) < 0)
        return (-1);

    /* 将接口配置结构体的缓冲区的末尾地址赋值给llifr指针 */
    llifr = (struct lifreq *)&intf->lifc.lifc_buf[intf->lifc.lifc_len];
}
    # 遍历接口请求列表中的每个接口请求结构体
    for (lifr = intf->lifc.lifc_req; lifr < llifr; lifr = NEXTLIFR(lifr)) {
        # 如果接口名中包含冒号，则将冒号后的内容截断
        if ((p = strchr(lifr->lifr_name, ':')) != NULL)
            *p = '\0';
        
        # 遍历接口请求列表中的每个接口请求结构体，查找是否存在相同的接口名
        for (plifr = intf->lifc.lifc_req; plifr < lifr; plifr = NEXTLIFR(lifr)) {
            if (strcmp(lifr->lifr_name, plifr->lifr_name) == 0)
                break;
        }
        # 如果存在相同的接口名，则跳过当前循环
        if (lifr > intf->lifc.lifc_req && plifr < lifr)
            continue;

        # 将 entry 结构体的 intf_name 成员赋值为当前接口名
        memset(ebuf, 0, sizeof(ebuf));
        strlcpy(entry->intf_name, lifr->lifr_name,
            sizeof(entry->intf_name));

        # 如果接口名中存在冒号，则将冒号还原
        if (p) *p = ':';

        # 复制接口名到 lifrflags 结构体的 lifr_name 成员
        strlcpy(lifrflags.lifr_name, lifr->lifr_name, sizeof(lifrflags.lifr_name));
        # 获取接口的标志信息
        if (ioctl(intf->fd, SIOCGLIFFLAGS, &lifrflags) >= 0)
            ;
        # 如果获取失败，尝试获取 IPv6 的接口标志信息
        else if (intf->fd6 != -1 && ioctl(intf->fd6, SIOCGLIFFLAGS, &lifrflags) >= 0)
            ;
        # 如果获取失败，则返回 -1
        else
            return (-1);
#ifdef IFF_IPMP
        # 如果接口标志包含 IFF_IPMP，则跳过当前循环
        if (lifrflags.lifr_flags & IFF_IPMP) {
            continue;
        }
#endif
        
        # 获取接口的非别名信息，如果失败则返回-1
        if (_intf_get_noalias(intf, entry) < 0)
            return (-1);
        # 获取接口的别名信息，如果失败则返回-1
        if (_intf_get_aliases(intf, entry) < 0)
            return (-1);
        
        # 调用回调函数处理接口信息，如果返回值不为0则返回该值
        if ((ret = (*callback)(entry, arg)) != 0)
            return (ret);
    }
    # 循环结束，返回0
    return (0);
}
#else
int
intf_loop(intf_t *intf, intf_handler callback, void *arg)
{
    struct intf_entry *entry;
    struct ifreq *ifr, *lifr, *pifr;
    char *p, ebuf[BUFSIZ];
    int ret;

    entry = (struct intf_entry *)ebuf;

    intf->ifc.ifc_buf = (caddr_t)intf->ifcbuf;
    intf->ifc.ifc_len = sizeof(intf->ifcbuf);
    
    # 获取接口信息，如果失败则返回-1
    if (ioctl(intf->fd, SIOCGIFCONF, &intf->ifc) < 0)
        return (-1);

    pifr = NULL;
    lifr = (struct ifreq *)&intf->ifc.ifc_buf[intf->ifc.ifc_len];
    
    # 遍历接口信息
    for (ifr = intf->ifc.ifc_req; ifr < lifr; ifr = NEXTIFR(ifr)) {
        /* XXX - Linux, Solaris ifaliases */
        # 如果接口名中包含':'，则将其替换为'\0'
        if ((p = strchr(ifr->ifr_name, ':')) != NULL)
            *p = '\0';
        
        # 如果上一个接口名不为空且与当前接口名相同，则跳过当前循环
        if (pifr != NULL && strcmp(ifr->ifr_name, pifr->ifr_name) == 0) {
            if (p) *p = ':';
            continue;
        }

        # 清空ebuf
        memset(ebuf, 0, sizeof(ebuf));
        # 将接口名拷贝到entry中
        strlcpy(entry->intf_name, ifr->ifr_name,
            sizeof(entry->intf_name));
        entry->intf_len = sizeof(ebuf);

        # 如果获取接口的非别名信息失败，则返回-1
        if (_intf_get_noalias(intf, entry) < 0)
            return (-1);
        # 如果获取接口的别名信息失败，则返回-1
        if (_intf_get_aliases(intf, entry) < 0)
            return (-1);
        
        # 调用回调函数处理接口信息，如果返回值不为0则返回该值
        if ((ret = (*callback)(entry, arg)) != 0)
            return (ret);

        pifr = ifr;
    }
    # 循环结束，返回0
    return (0);
}
#endif /* !HAVE_LINUX_PROCFS */

# 关闭接口
intf_t *
intf_close(intf_t *intf)
{
    if (intf != NULL) {
        if (intf->fd >= 0)
            close(intf->fd);
        if (intf->fd6 >= 0)
            close(intf->fd6);
        free(intf);
    }
    # 返回NULL
    return (NULL);
}
```