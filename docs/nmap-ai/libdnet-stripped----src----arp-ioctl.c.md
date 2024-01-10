# `nmap\libdnet-stripped\src\arp-ioctl.c`

```
/*
 * arp-ioctl.c
 *
 * 版权所有 (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp-ioctl.c 554 2005-02-09 22:31:00Z dugsong $
 */

#include "config.h"

#include <sys/param.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#ifdef HAVE_STREAMS_MIB2
# include <sys/sockio.h>
# include <sys/stream.h>
# include <sys/tihdr.h>
# include <sys/tiuser.h>
# include <inet/common.h>
# include <inet/mib2.h>
# include <inet/ip.h>
# undef IP_ADDR_LEN
#elif defined(HAVE_SYS_MIB_H)
# include <sys/mib.h>
#endif

#include <net/if.h>
#include <net/if_arp.h>
#ifdef HAVE_STREAMS_MIB2
# include <netinet/in.h>
# include <stropts.h>
#endif
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

#ifdef HAVE_LINUX_PROCFS
#define PROC_ARP_FILE    "/proc/net/arp"
#endif

struct arp_handle {
    int     fd;
#ifdef HAVE_ARPREQ_ARP_DEV
    intf_t    *intf;
#endif
};

// 打开 ARP 处理器
arp_t *
arp_open(void)
{
    arp_t *a;
    
    // 分配内存给 ARP 处理器
    if ((a = calloc(1, sizeof(*a))) != NULL) {
#ifdef HAVE_STREAMS_MIB2
        // 如果支持 MIB2，则打开 IP 设备
        if ((a->fd = open(IP_DEV_NAME, O_RDWR)) < 0)
#elif defined(HAVE_STREAMS_ROUTE)
        // 如果支持路由，则打开路由设备
        if ((a->fd = open("/dev/route", O_WRONLY, 0)) < 0)
#else
        // 否则，创建一个数据报套接字
        if ((a->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
#endif
            return (arp_close(a));
#ifdef HAVE_ARPREQ_ARP_DEV
        // 如果支持 ARP 设备，则打开接口
        if ((a->intf = intf_open()) == NULL)
            return (arp_close(a));
#endif
    }
    return (a);
}

#ifdef HAVE_ARPREQ_ARP_DEV
static int
_arp_set_dev(const struct intf_entry *entry, void *arg)
{
    struct arpreq *ar = (struct arpreq *)arg;
    struct addr dst;
    uint32_t mask;
    # 如果接口类型为以太网且地址类型为IP
    if (entry->intf_type == INTF_TYPE_ETH &&
        entry->intf_addr.addr_type == ADDR_TYPE_IP) {
        # 将地址位转换为掩码
        addr_btom(entry->intf_addr.addr_bits, &mask, IP_ADDR_LEN);
        # 将地址结构体转换为网络字节序
        addr_ston((struct sockaddr *)&ar->arp_pa, &dst);
    
        # 如果接口地址与目标地址按位与操作后相等
        if ((entry->intf_addr.addr_ip & mask) ==
            (dst.addr_ip & mask)) {
            # 将接口名拷贝到ARP设备字段中
            strlcpy(ar->arp_dev, entry->intf_name,
                sizeof(ar->arp_dev));
            # 返回1表示匹配成功
            return (1);
        }
    }
    # 返回0表示匹配失败
    return (0);
}
#endif

int
arp_add(arp_t *a, const struct arp_entry *entry)
{
    struct arpreq ar;

    memset(&ar, 0, sizeof(ar));  // 初始化 arpreq 结构体为 0

    if (addr_ntos(&entry->arp_pa, &ar.arp_pa) < 0)  // 将 entry 中的 arp_pa 转换为网络字节序，存入 ar.arp_pa
        return (-1);

    /* XXX - see arp(7) for details... */
#ifdef __linux__
    if (addr_ntos(&entry->arp_ha, &ar.arp_ha) < 0)  // 将 entry 中的 arp_ha 转换为网络字节序，存入 ar.arp_ha
        return (-1);
    ar.arp_ha.sa_family = ARP_HRD_ETH;  // 设置 ar.arp_ha 的地址族为以太网
#else
    /* XXX - Solaris, HP-UX, IRIX, other Mentat stacks? */
    ar.arp_ha.sa_family = AF_UNSPEC;  // 设置 ar.arp_ha 的地址族为未指定
    memcpy(ar.arp_ha.sa_data, &entry->arp_ha.addr_eth, ETH_ADDR_LEN);  // 将 entry 中的 arp_ha 的以太网地址拷贝到 ar.arp_ha 的地址数据中
#endif

#ifdef HAVE_ARPREQ_ARP_DEV
    if (intf_loop(a->intf, _arp_set_dev, &ar) != 1) {  // 如果设置 ar.arp_dev 失败
        errno = ESRCH;  // 设置错误码为 ESRCH
        return (-1);
    }
#endif
    ar.arp_flags = ATF_PERM | ATF_COM;  // 设置 ar.arp_flags 的标志位为永久和完成
#ifdef hpux
    /* XXX - screwy extended arpreq struct */
    {
        struct sockaddr_in *sin;

        ar.arp_hw_addr_len = ETH_ADDR_LEN;  // 设置 ar.arp_hw_addr_len 为以太网地址长度
        sin = (struct sockaddr_in *)&ar.arp_pa_mask;  // 将 ar.arp_pa_mask 强制转换为 sockaddr_in 结构体
        sin->sin_family = AF_INET;  // 设置 sin 的地址族为 IPv4
        sin->sin_addr.s_addr = IP_ADDR_BROADCAST;  // 设置 sin 的 IP 地址为广播地址
    }
#endif
    if (ioctl(a->fd, SIOCSARP, &ar) < 0)  // 发送 SIOCSARP 命令，添加 ARP 表项
        return (-1);

#ifdef HAVE_STREAMS_MIB2
    /* XXX - force entry into ipNetToMediaTable. */
    {
        struct sockaddr_in sin;
        int fd;
        
        addr_ntos(&entry->arp_pa, (struct sockaddr *)&sin);  // 将 entry 中的 arp_pa 转换为网络字节序，存入 sin
        sin.sin_port = htons(666);  // 设置 sin 的端口号为 666
        
        if ((fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)  // 创建一个 IPv4 的数据报套接字
            return (-1);
        
        if (connect(fd, (struct sockaddr *)&sin, sizeof(sin)) < 0) {  // 连接到指定地址
            close(fd);
            return (-1);
        }
        write(fd, NULL, 0);  // 向套接字写入数据
        close(fd);  // 关闭套接字
    }
#endif
    return (0);  // 返回成功
}

int
arp_delete(arp_t *a, const struct arp_entry *entry)
{
    struct arpreq ar;

    memset(&ar, 0, sizeof(ar));  // 初始化 arpreq 结构体为 0
    
    if (addr_ntos(&entry->arp_pa, &ar.arp_pa) < 0)  // 将 entry 中的 arp_pa 转换为网络字节序，存入 ar.arp_pa
        return (-1);
    
    if (ioctl(a->fd, SIOCDARP, &ar) < 0)  // 发送 SIOCDARP 命令，删除 ARP 表项
        return (-1);

    return (0);  // 返回成功
}

int
arp_get(arp_t *a, struct arp_entry *entry)
{
    struct arpreq ar;
    # 使用 0 填充 ar 变量的内存，大小为 ar 变量的大小
    memset(&ar, 0, sizeof(ar));
    
    # 如果将网络地址转换为字符串形式失败，则返回 -1
    if (addr_ntos(&entry->arp_pa, &ar.arp_pa) < 0)
        return (-1);
#ifdef HAVE_ARPREQ_ARP_DEV
    # 如果系统支持获取 ARP 设备信息
    if (intf_loop(a->intf, _arp_set_dev, &ar) != 1) {
        # 如果获取失败，设置错误码并返回-1
        errno = ESRCH;
        return (-1);
    }
#endif
    # 获取指定文件描述符的 ARP 地址信息
    if (ioctl(a->fd, SIOCGARP, &ar) < 0)
        return (-1);

    # 如果 ARP 地址标志不包含 ATF_COM
    if ((ar.arp_flags & ATF_COM) == 0) {
        # 设置错误码并返回-1
        errno = ESRCH;
        return (-1);
    }
    # 将 ARP 地址转换为网络地址结构
    return (addr_ston(&ar.arp_ha, &entry->arp_ha));
}

#ifdef HAVE_LINUX_PROCFS
int
arp_loop(arp_t *a, arp_handler callback, void *arg)
{
    FILE *fp;
    struct arp_entry entry;
    char buf[BUFSIZ], ipbuf[100], macbuf[100], maskbuf[100], devbuf[100];
    int i, type, flags, ret;

    # 打开 /proc/net/arp 文件
    if ((fp = fopen(PROC_ARP_FILE, "r")) == NULL)
        return (-1);

    ret = 0;
    # 逐行读取 /proc/net/arp 文件内容
    while (fgets(buf, sizeof(buf), fp) != NULL) {
        # 从行中解析出 IP 地址、类型、标志、MAC 地址、掩码和设备信息
        i = sscanf(buf, "%s 0x%x 0x%x %99s %99s %99s\n",
            ipbuf, &type, &flags, macbuf, maskbuf, devbuf);
        
        # 如果解析出的字段不足4个或者标志不包含ATF_COM，则继续下一次循环
        if (i < 4 || (flags & ATF_COM) == 0)
            continue;
        
        # 将 IP 地址和 MAC 地址转换为网络地址结构
        if (addr_aton(ipbuf, &entry.arp_pa) == 0 &&
            addr_aton(macbuf, &entry.arp_ha) == 0) {
            # 调用回调函数处理 ARP 地址信息
            if ((ret = callback(&entry, arg)) != 0)
                break;
        }
    }
    # 如果发生文件读取错误，关闭文件并返回-1
    if (ferror(fp)) {
        fclose(fp);
        return (-1);
    }
    # 关闭文件
    fclose(fp);
    
    # 返回处理结果
    return (ret);
}
#elif defined (HAVE_STREAMS_MIB2)
int
arp_loop(arp_t *r, arp_handler callback, void *arg)
{
    struct arp_entry entry;
    struct strbuf msg;
    struct T_optmgmt_req *tor;
    struct T_optmgmt_ack *toa;
    struct T_error_ack *tea;
    struct opthdr *opt;
    mib2_ipNetToMediaEntry_t *arp, *arpend;
    u_char buf[8192];
    int flags, rc, atable, ret;

    tor = (struct T_optmgmt_req *)buf;
    toa = (struct T_optmgmt_ack *)buf;
    tea = (struct T_error_ack *)buf;

    # 设置 TPI 操作类型为 T_OPTMGMT_REQ
    tor->PRIM_type = T_OPTMGMT_REQ;
    tor->OPT_offset = sizeof(*tor);
    tor->OPT_length = sizeof(*opt);
    tor->MGMT_flags = T_CURRENT;
    
    opt = (struct opthdr *)(tor + 1);
    opt->level = MIB2_IP;
    opt->name = opt->len = 0;
    
    msg.maxlen = sizeof(buf);
    # 计算消息长度，包括tor和opt的大小
    msg.len = sizeof(*tor) + sizeof(*opt);
    # 将消息缓冲区设置为给定的buf
    msg.buf = buf;
    
    # 如果向文件描述符r->fd发送消息失败，则返回-1
    if (putmsg(r->fd, &msg, NULL, 0) < 0)
        return (-1);
    
    # 将opt指针设置为指向toa后面的位置
    opt = (struct opthdr *)(toa + 1);
    # 设置消息的最大长度为buf的大小
    msg.maxlen = sizeof(buf);
    # 无限循环，直到条件满足跳出循环
    for (;;) {
        # 初始化标志位
        flags = 0;
        # 从套接字中读取消息
        if ((rc = getmsg(r->fd, &msg, NULL, &flags)) < 0)
            return (-1);

        # 检查是否完成
        if (rc == 0 &&
            msg.len >= sizeof(*toa) &&
            toa->PRIM_type == T_OPTMGMT_ACK &&
            toa->MGMT_flags == T_SUCCESS && opt->len == 0)
            break;

        # 检查是否收到错误消息
        if (msg.len >= sizeof(*tea) && tea->PRIM_type == T_ERROR_ACK)
            return (-1);
        
        # 检查是否收到预期的消息
        if (rc != MOREDATA || msg.len < (int)sizeof(*toa) ||
            toa->PRIM_type != T_OPTMGMT_ACK ||
            toa->MGMT_flags != T_SUCCESS)
            return (-1);
        
        # 判断是否为特定的表
        atable = (opt->level == MIB2_IP && opt->name == MIB2_IP_22);
        
        # 设置消息的最大长度和长度
        msg.maxlen = sizeof(buf) - (sizeof(buf) % sizeof(*arp));
        msg.len = 0;
        flags = 0;
        
        # 循环读取消息
        do {
            rc = getmsg(r->fd, NULL, &msg, &flags);
            
            # 检查是否出错
            if (rc != 0 && rc != MOREDATA)
                return (-1);
            
            # 如果不是特定表，继续循环
            if (!atable)
                continue;
            
            # 将消息缓冲区转换为arp结构
            arp = (mib2_ipNetToMediaEntry_t *)msg.buf;
            arpend = (mib2_ipNetToMediaEntry_t *)
                (msg.buf + msg.len);

            # 设置arp表的地址类型和位数
            entry.arp_pa.addr_type = ADDR_TYPE_IP;
            entry.arp_pa.addr_bits = IP_ADDR_BITS;
            
            entry.arp_ha.addr_type = ADDR_TYPE_ETH;
            entry.arp_ha.addr_bits = ETH_ADDR_BITS;

            # 遍历arp表，处理每一项
            for ( ; arp < arpend; arp++) {
                entry.arp_pa.addr_ip =
                    arp->ipNetToMediaNetAddress;
                
                memcpy(&entry.arp_ha.addr_eth,
                    arp->ipNetToMediaPhysAddress.o_bytes,
                    ETH_ADDR_LEN);
                
                # 调用回调函数处理每一项
                if ((ret = callback(&entry, arg)) != 0)
                    return (ret);
            }
        } while (rc == MOREDATA);
    }
    # 循环结束，返回0
    return (0);
}  // 结束 if-else 块

#elif defined(HAVE_SYS_MIB_H)  // 如果定义了 HAVE_SYS_MIB_H

#define MAX_ARPENTRIES    512    /* XXX */  // 定义最大 ARP 条目数为 512

int  // 定义返回类型为整数的函数 arp_loop
arp_loop(arp_t *r, arp_handler callback, void *arg)  // 函数参数包括 arp_t 结构体指针 r, arp_handler 类型的回调函数指针 callback, 以及 void 类型的指针 arg
{
    struct nmparms nm;  // 定义 nmparms 结构体变量 nm
    struct arp_entry entry;  // 定义 arp_entry 结构体变量 entry
    mib_ipNetToMediaEnt arpentries[MAX_ARPENTRIES];  // 定义 mib_ipNetToMediaEnt 类型的数组 arpentries，长度为 MAX_ARPENTRIES
    int fd, i, n, ret;  // 定义整型变量 fd, i, n, ret
    
    if ((fd = open_mib("/dev/ip", O_RDWR, 0 /* XXX */, 0)) < 0)  // 如果打开 MIB 失败
        return (-1);  // 返回 -1
    
    nm.objid = ID_ipNetToMediaTable;  // 设置 nm 的 objid 为 ID_ipNetToMediaTable
    nm.buffer = arpentries;  // 设置 nm 的 buffer 为 arpentries
    n = sizeof(arpentries);  // 设置 n 为 arpentries 的大小
    nm.len = &n;  // 设置 nm 的 len 为 n 的地址
    
    if (get_mib_info(fd, &nm) < 0) {  // 如果获取 MIB 信息失败
        close_mib(fd);  // 关闭 MIB
        return (-1);  // 返回 -1
    }
    close_mib(fd);  // 关闭 MIB

    entry.arp_pa.addr_type = ADDR_TYPE_IP;  // 设置 entry 的 arp_pa 的 addr_type 为 ADDR_TYPE_IP
    entry.arp_pa.addr_bits = IP_ADDR_BITS;  // 设置 entry 的 arp_pa 的 addr_bits 为 IP_ADDR_BITS

    entry.arp_ha.addr_type = ADDR_TYPE_ETH;  // 设置 entry 的 arp_ha 的 addr_type 为 ADDR_TYPE_ETH
    entry.arp_ha.addr_bits = ETH_ADDR_BITS;  // 设置 entry 的 arp_ha 的 addr_bits 为 ETH_ADDR_BITS
    
    n /= sizeof(*arpentries);  // 计算 n 除以 arpentries 中每个元素的大小
    ret = 0;  // 设置 ret 为 0
    
    for (i = 0; i < n; i++) {  // 循环遍历 arpentries 数组
        if (arpentries[i].Type == INTM_INVALID ||  // 如果 arpentries[i] 的 Type 为 INTM_INVALID
            arpentries[i].PhysAddr.o_length != ETH_ADDR_LEN)  // 或者 arpentries[i] 的 PhysAddr.o_length 不等于 ETH_ADDR_LEN
            continue;  // 继续下一次循环
        
        entry.arp_pa.addr_ip = arpentries[i].NetAddr;  // 设置 entry 的 arp_pa 的 addr_ip 为 arpentries[i] 的 NetAddr
        memcpy(&entry.arp_ha.addr_eth, arpentries[i].PhysAddr.o_bytes,  // 使用 memcpy 将 arpentries[i] 的 PhysAddr.o_bytes 复制到 entry 的 arp_ha 的 addr_eth
            ETH_ADDR_LEN);
        
        if ((ret = callback(&entry, arg)) != 0)  // 调用回调函数 callback，如果返回值不为 0
            break;  // 跳出循环
    }
    return (ret);  // 返回 ret
}
#elif defined(HAVE_NET_RADIX_H) && !defined(_AIX)  // 如果定义了 HAVE_NET_RADIX_H 并且未定义 _AIX

/* XXX - Tru64, others? */  // 注释：Tru64，其他？
#include <netinet/if_ether.h>  // 包含头文件 <netinet/if_ether.h>
#include <nlist.h>  // 包含头文件 <nlist.h>

static int  // 定义静态函数 _kread，返回类型为整数
_kread(int fd, void *addr, void *buf, int len)  // 函数参数包括整型变量 fd, 两个 void 类型指针 addr, buf，以及整型变量 len
{
    if (lseek(fd, (off_t)addr, SEEK_SET) == (off_t)-1L)  // 如果 lseek 失败
        return (-1);  // 返回 -1
    return (read(fd, buf, len) == len ? 0 : -1);  // 如果读取成功，返回 0，否则返回 -1
}

static int  // 定义静态函数 _radix_walk，返回类型为整数
_radix_walk(int fd, struct radix_node *rn, arp_handler callback, void *arg)  // 函数参数包括整型变量 fd, 结构体指针 rn, arp_handler 类型的回调函数指针 callback, 以及 void 类型的指针 arg
{
    struct radix_node rnode;  // 定义结构体变量 rnode
    struct rtentry rt;  // 定义结构体变量 rt
    struct sockaddr_in sin;  // 定义结构体变量 sin
    struct arptab at;  // 定义结构体变量 at
    struct arp_entry entry;  // 定义结构体变量 entry
    int ret = 0;  // 定义整型变量 ret，并初始化为 0
 again:  // 标签：again

    _kread(fd, rn, &rnode, sizeof(rnode));  // 调用 _kread 函数
    # 如果路由节点的 rn_b 小于 0
    if (rnode.rn_b < 0) {
        # 如果路由节点的标志不是 RNF_ROOT
        if (!(rnode.rn_flags & RNF_ROOT)) {
            # 从文件描述符 fd 中读取 rt 结构体的内容，大小为 sizeof(rt)，并存储在 rn 中
            _kread(fd, rn, &rt, sizeof(rt));
            # 从文件描述符 fd 中读取 rt 结构体中的键值，大小为 sizeof(sin)，并存储在 rt_key(&rt) 中
            _kread(fd, rt_key(&rt), &sin, sizeof(sin));
            # 将 sin 转换为地址结构体，存储在 entry.arp_pa 中
            addr_ston((struct sockaddr *)&sin, &entry.arp_pa);
            # 从文件描述符 fd 中读取 rt 结构体中的 rt_llinfo，大小为 sizeof(at)，并存储在 rt.rt_llinfo 中
            _kread(fd, rt.rt_llinfo, &at, sizeof(at));
            # 如果 at_flags 中包含 ATF_COM
            if (at.at_flags & ATF_COM) {
                # 将 at 中的硬件地址转换为地址结构体中的硬件地址，存储在 entry.arp_ha 中
                addr_pack(&entry.arp_ha, ADDR_TYPE_ETH, ETH_ADDR_BITS, at.at_hwaddr, ETH_ADDR_LEN);
                # 如果回调函数返回值不为 0，则返回该值
                if ((ret = callback(&entry, arg)) != 0)
                    return (ret);
            }
        }
        # 如果路由节点的 rn_dupedkey 不为空，则跳转到 again 标签处
        if ((rn = rnode.rn_dupedkey))
            goto again;
    } 
    # 否则
    else {
        # 将 rn 设置为路由节点的右子节点
        rn = rnode.rn_r;
        # 如果 _radix_walk 函数返回值不为 0，则返回该值
        if ((ret = _radix_walk(fd, rnode.rn_l, callback, arg)) != 0)
            return (ret);
        # 如果 _radix_walk 函数返回值不为 0，则返回该值
        if ((ret = _radix_walk(fd, rn, callback, arg)) != 0)
            return (ret);
    }
    # 返回 ret 变量的值
    return (ret);
# 定义一个名为 arp_loop 的函数，接受 arp_t 结构体指针、arp_handler 回调函数和 void 指针作为参数
int
arp_loop(arp_t *r, arp_handler callback, void *arg)
{
    # 定义 ifnet 结构体指针 ifp 和 ifnet 结构体
    struct ifnet *ifp, ifnet;
    # 定义 ifarp 结构体和 radix_node_head 结构体指针 head
    struct ifnet_arp_cache_head ifarp;
    struct radix_node_head *head;
    
    # 定义 nlist 结构体数组 nl，整型变量 fd 和 ret
    struct nlist nl[2];
    int fd, ret = 0;

    # 将 nl 数组清零
    memset(nl, 0, sizeof(nl));
    # 设置 nl[0] 的 n_name 字段为 "ifnet"
    nl[0].n_name = "ifnet";
    
    # 如果 knlist(nl) 小于 0 或者 nl[0].n_type 等于 0，或者打开 /dev/kmem 文件失败，则返回 -1
    if (knlist(nl) < 0 || nl[0].n_type == 0 ||
        (fd = open("/dev/kmem", O_RDONLY, 0)) < 0)
        return (-1);

    # 遍历 ifnet 链表
    for (ifp = (struct ifnet *)nl[0].n_value;
        ifp != NULL; ifp = ifnet.if_next) {
        # 从内核内存中读取 ifnet 结构体的内容
        _kread(fd, ifp, &ifnet, sizeof(ifnet));
        # 如果 ifnet.if_arp_cache_head 不为空
        if (ifnet.if_arp_cache_head != NULL) {
            # 从内核内存中读取 ifarp 结构体的内容
            _kread(fd, ifnet.if_arp_cache_head,
                &ifarp, sizeof(ifarp));
            # XXX - 只有一个 rnh，只有 AF_INET。
            # 调用 _radix_walk 函数遍历 arp 缓存
            if ((ret = _radix_walk(fd, ifarp.arp_cache_head.rnh_treetop,
                 callback, arg)) != 0)
                break;
        }
    }
    # 关闭 /dev/kmem 文件
    close(fd);
    # 返回 ret 变量的值
    return (ret);
}
#else
# 如果没有定义，则定义一个名为 arp_loop 的函数，接受 arp_t 结构体指针、arp_handler 回调函数和 void 指针作为参数
int
arp_loop(arp_t *a, arp_handler callback, void *arg)
{
    # 设置 errno 为 ENOSYS
    errno = ENOSYS;
    # 返回 -1
    return (-1);
}
#endif

# 定义一个名为 arp_close 的函数，接受 arp_t 结构体指针作为参数
arp_t *
arp_close(arp_t *a)
{
    # 如果 a 不为空
    if (a != NULL) {
        # 如果 a 的文件描述符大于等于 0，则关闭文件描述符
        if (a->fd >= 0)
            close(a->fd);
        # 如果定义了 HAVE_ARPREQ_ARP_DEV，并且 a 的 intf 字段不为空，则关闭 intf
#ifdef HAVE_ARPREQ_ARP_DEV
        if (a->intf != NULL)
            intf_close(a->intf);
#endif
        # 释放 a 指针指向的内存
        free(a);
    }
    # 返回空指针
    return (NULL);
}
```