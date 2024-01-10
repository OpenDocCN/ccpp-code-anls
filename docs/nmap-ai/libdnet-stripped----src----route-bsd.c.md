# `nmap\libdnet-stripped\src\route-bsd.c`

```
/*
 * route-bsd.c
 *
 * 版权所有 (c) 2001 Dug Song <dugsong@monkey.org>
 * 版权所有 (c) 1999 Masaki Hirabaru <masaki@merit.edu>
 * 
 * $Id: route-bsd.c 555 2005-02-10 05:18:38Z dugsong $
 */

#include "config.h"

#include <sys/param.h>
#include <sys/types.h>
#include <sys/socket.h>
#ifdef HAVE_SYS_SYSCTL_H
#include <sys/sysctl.h>
#endif
#ifdef HAVE_STREAMS_MIB2
#include <sys/stream.h>
#include <sys/tihdr.h>
#include <sys/tiuser.h>
#include <inet/common.h>
#include <inet/mib2.h>
#include <inet/ip.h>
#undef IP_ADDR_LEN
#include <stropts.h>
#elif defined(HAVE_STREAMS_ROUTE)
#include <sys/stream.h>
#include <sys/stropts.h>
#endif
#ifdef HAVE_GETKERNINFO
#include <sys/kinfo.h>
#endif

#define route_t    oroute_t    /* XXX - unixware */
#include <net/route.h>
#undef route_t
#include <net/if.h>
#include <netinet/in.h>

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

#if defined(RT_ROUNDUP) && defined(__NetBSD__)
/* NetBSD定义了这个宏，将其舍入到64位边界。
   http://fxr.watson.org/fxr/ident?v=NETBSD;i=RT_ROUNDUP */
#define ROUNDUP(a) RT_ROUNDUP(a)
#else
/* Unix网络编程第3版指出，rt_msghdr中的sockaddr结构应该填充，以便它们的地址从sizeof(u_long)的倍数开始。
   但至少在64位Mac OS X 10.6上，这是错误的。苹果的netstat代码使用4字节填充，而不是8字节。这对于IPv6地址是相关的，因为sa_len == 28。
   http://www.opensource.apple.com/source/network_cmds/network_cmds-329.2.2/netstat.tproj/route.c */
#ifdef __APPLE__
#define RT_MSGHDR_ALIGNMENT sizeof(uint32_t)
#else
#define RT_MSGHDR_ALIGNMENT sizeof(unsigned long)
#endif
#define ROUNDUP(a) \
    ((a) > 0 ? (1 + (((a) - 1) | (RT_MSGHDR_ALIGNMENT - 1))) : RT_MSGHDR_ALIGNMENT)
#endif

#ifdef HAVE_SOCKADDR_SA_LEN
#define NEXTSA(s) \
    ((struct sockaddr *)((u_char *)(s) + ROUNDUP((s)->sa_len)))
#else
#define NEXTSA(s) \
    # 将指针 s 强制转换为 u_char 类型的指针，然后加上结构体 s 的大小，再将结果强制转换为 sockaddr 结构体的指针
    ((struct sockaddr *)((u_char *)(s) + ROUNDUP(sizeof(*(s)))))
#endif

struct route_handle {
    int    fd;  // 文件描述符
    int    seq;  // 序列号
#ifdef HAVE_STREAMS_MIB2
    int    ip_fd;  // IP 文件描述符
#endif
};

#ifdef DEBUG
static void
route_msg_print(struct rt_msghdr *rtm)
{
    printf("v: %d type: 0x%x flags: 0x%x addrs: 0x%x pid: %d seq: %d\n",
        rtm->rtm_version, rtm->rtm_type, rtm->rtm_flags,
        rtm->rtm_addrs, rtm->rtm_pid, rtm->rtm_seq);
}
#endif

static int
route_msg(route_t *r, int type, char intf_name[INTF_NAME_LEN], struct addr *dst, struct addr *gw)
{
    struct addr net;  // 网络地址
    struct rt_msghdr *rtm;  // 路由消息头
    struct sockaddr *sa;  // 套接字地址
    u_char buf[BUFSIZ];  // 缓冲区
    pid_t pid;  // 进程ID
    int len;  // 长度

    memset(buf, 0, sizeof(buf));  // 初始化缓冲区

    rtm = (struct rt_msghdr *)buf;  // 将缓冲区转换为路由消息头
    rtm->rtm_version = RTM_VERSION;  // 设置路由消息版本号
    if ((rtm->rtm_type = type) != RTM_DELETE)  // 如果消息类型不是删除
        rtm->rtm_flags = RTF_UP;  // 设置消息标志为UP
    rtm->rtm_addrs = RTA_DST;  // 设置消息地址为目的地址
    rtm->rtm_seq = ++r->seq;  // 设置消息序列号

    /* Destination */
    sa = (struct sockaddr *)(rtm + 1);  // 设置套接字地址为路由消息头后的位置
    if (addr_net(dst, &net) < 0 || addr_ntos(&net, sa) < 0)  // 如果获取目的地址网络失败或者转换为字符串失败
        return (-1);  // 返回错误
    sa = NEXTSA(sa);  // 设置下一个套接字地址位置

    /* Gateway */
    if (gw != NULL && type != RTM_GET) {  // 如果网关不为空且消息类型不是获取
        rtm->rtm_flags |= RTF_GATEWAY;  // 设置消息标志为网关
        rtm->rtm_addrs |= RTA_GATEWAY;  // 设置消息地址为网关地址
        if (addr_ntos(gw, sa) < 0)  // 如果转换网关地址为字符串失败
            return (-1);  // 返回错误
        sa = NEXTSA(sa);  // 设置下一个套接字地址位置
    }
    /* Netmask */
    if (dst->addr_ip == IP_ADDR_ANY || dst->addr_bits < IP_ADDR_BITS) {  // 如果目的地址为任意IP或者位数小于IP位数
        rtm->rtm_addrs |= RTA_NETMASK;  // 设置消息地址为子网掩码
        if (addr_btos(dst->addr_bits, sa) < 0)  // 如果转换位数为字符串失败
            return (-1);  // 返回错误
        sa = NEXTSA(sa);  // 设置下一个套接字地址位置
    } else
        rtm->rtm_flags |= RTF_HOST;  // 否则设置消息标志为主机
    
    rtm->rtm_msglen = (u_char *)sa - buf;  // 设置消息长度为套接字地址位置减去缓冲区位置
#ifdef DEBUG
    route_msg_print(rtm);  // 打印路由消息
#endif
#ifdef HAVE_STREAMS_ROUTE
    if (ioctl(r->fd, RTSTR_SEND, rtm) < 0)  // 如果使用流式路由发送失败
        return (-1);  // 返回错误
#else
    if (write(r->fd, buf, rtm->rtm_msglen) < 0)  // 如果写入文件描述符失败
        return (-1);  // 返回错误

    pid = getpid();  // 获取进程ID
    # 当类型为 RTM_GET 且成功读取到数据时进入循环
    while (type == RTM_GET && (len = read(r->fd, buf, sizeof(buf))) > 0) {
        # 如果读取的数据长度小于 rtm 结构体的大小，则返回 -1
        if (len < (int)sizeof(*rtm)) {
            return (-1);
        }
        # 如果读取到的数据中的 rtm_type 等于指定类型，rtm_pid 等于指定进程ID，rtm_seq 等于指定序列号，则进入条件
        if (rtm->rtm_type == type && rtm->rtm_pid == pid &&
            rtm->rtm_seq == r->seq) {
            # 如果读取到的数据中 rtm_errno 不为 0，则将 errno 设置为 rtm_errno，并返回 -1
            if (rtm->rtm_errno) {
                errno = rtm->rtm_errno;
                return (-1);
            }
            # 跳出循环
            break;
        }
    }
#endif
    // 如果消息类型为 RTM_GET，并且包含目的地址和网关地址
    if (type == RTM_GET && (rtm->rtm_addrs & (RTA_DST|RTA_GATEWAY)) ==
        (RTA_DST|RTA_GATEWAY)) {
        // 获取指向地址结构的指针
        sa = (struct sockaddr *)(rtm + 1);
        // 移动指针到下一个地址结构
        sa = NEXTSA(sa);
        
        // 如果地址转换失败或者网关地址类型不是 IP 地址
        if (addr_ston(sa, gw) < 0 || gw->addr_type != ADDR_TYPE_IP) {
            // 设置错误码并返回 -1
            errno = ESRCH;
            return (-1);
        }

        // 如果接口名不为空
        if (intf_name != NULL) {
            char namebuf[IF_NAMESIZE];

            // 如果根据索引获取接口名失败
            if (if_indextoname(rtm->rtm_index, namebuf) == NULL) {
                // 设置错误码并返回 -1
                errno = ESRCH;
                return (-1);
            }
            // 将接口名拷贝到指定长度的缓冲区
            strlcpy(intf_name, namebuf, INTF_NAME_LEN);
        }
    }
    // 返回 0
    return (0);
}

// 打开路由
route_t *
route_open(void)
{
    route_t *r;
    
    // 分配内存并初始化为 0
    if ((r = calloc(1, sizeof(*r))) != NULL) {
        r->fd = -1;
#ifdef HAVE_STREAMS_MIB2
        // 如果支持 MIB2
        if ((r->ip_fd = open(IP_DEV_NAME, O_RDWR)) < 0)
            return (route_close(r));
#endif
#ifdef HAVE_STREAMS_ROUTE
        // 如果支持路由
        if ((r->fd = open("/dev/route", O_RDWR, 0)) < 0)
#else
        // 如果不支持路由
        if ((r->fd = socket(PF_ROUTE, SOCK_RAW, AF_INET)) < 0)
#endif
            return (route_close(r));
    }
    // 返回路由对象
    return (r);
}

// 添加路由
int
route_add(route_t *r, const struct route_entry *entry)
{
    struct route_entry rtent;
    
    // 拷贝路由条目
    memcpy(&rtent, entry, sizeof(rtent));
    
    // 发送添加路由消息
    if (route_msg(r, RTM_ADD, NULL, &rtent.route_dst, &rtent.route_gw) < 0)
        return (-1);
    
    // 返回 0
    return (0);
}

// 删除路由
int
route_delete(route_t *r, const struct route_entry *entry)
{
    struct route_entry rtent;
    
    // 拷贝路由条目
    memcpy(&rtent, entry, sizeof(rtent));
    
    // 获取路由信息
    if (route_get(r, &rtent) < 0)
        return (-1);
    
    // 发送删除路由消息
    if (route_msg(r, RTM_DELETE, NULL, &rtent.route_dst, &rtent.route_gw) < 0)
        return (-1);
    
    // 返回 0
    return (0);
}

// 获取路由信息
int
route_get(route_t *r, struct route_entry *entry)
{
    // 发送获取路由消息
    if (route_msg(r, RTM_GET, entry->intf_name, &entry->route_dst, &entry->route_gw) < 0)
        return (-1);
    // 清空接口名和度量值
    entry->intf_name[0] = '\0';
    entry->metric = 0;
    
    // 返回 0
    return (0);
}
# 如果定义了 HAVE_SYS_SYSCTL_H 或者 HAVE_STREAMS_ROUTE 或者 HAVE_GETKERNINFO
# 则定义一个 addr_ston_gateway 函数，用于将地址转换为网络地址结构
# 如果转换失败，则检查网关地址族是否为 AF_LINK，如果是，则存储与目标地址相同类型的全零地址
# 全零地址是同一子网路由表条目的约定
static int
addr_ston_gateway(const struct addr *dst,
    const struct sockaddr *sa, struct addr *a)
{
    int rc;

    # 调用 addr_ston 函数将地址转换为网络地址结构
    rc = addr_ston(sa, a);
    # 如果转换成功，则直接返回结果
    if (rc == 0)
        return rc;

    # 如果转换失败，则根据条件判断是否为 AF_LINK 地址族，如果是，则存储全零地址
#ifdef HAVE_NET_IF_DL_H
# ifdef AF_LINK
    if (sa->sa_family == AF_LINK) {
        # 将地址结构 a 清零
        memset(a, 0, sizeof(*a));
        # 设置地址类型为目标地址的类型
        a->addr_type = dst->addr_type;
        return (0);
    }
# endif
#endif

    # 如果不是 AF_LINK 地址族，则返回错误
    return (-1);
}

# 定义 route_loop 函数，用于循环处理路由表中的每一条路由
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    struct rt_msghdr *rtm;
    struct route_entry entry;
    struct sockaddr *sa;
    char *buf, *lim, *next;
    int ret;
# 如果定义了 HAVE_SYS_SYSCTL_H
    int mib[6] = { CTL_NET, PF_ROUTE, 0, 0 /* XXX */, NET_RT_DUMP, 0 };
    size_t len;
    
    # 调用 sysctl 函数获取路由表信息
    if (sysctl(mib, 6, NULL, &len, NULL, 0) < 0)
        return (-1);

    # 如果获取的信息长度为 0，则直接返回
    if (len == 0)
        return (0);
    
    # 分配内存空间用于存储路由表信息
    if ((buf = malloc(len)) == NULL)
        return (-1);
    
    # 再次调用 sysctl 函数获取路由表信息
    if (sysctl(mib, 6, buf, &len, NULL, 0) < 0) {
        free(buf);
        return (-1);
    }
    lim = buf + len;
    next = buf;
# 如果定义了 HAVE_GETKERNINFO
    int len = getkerninfo(KINFO_RT_DUMP,0,0,0);

    # 如果获取的信息长度为 0，则直接返回
    if (len == 0)
        return (0);

    # 分配内存空间用于存储路由表信息
    if ((buf = malloc(len)) == NULL)
        return (-1);

    # 再次调用 getkerninfo 函数获取路由表信息
    if (getkerninfo(KINFO_RT_DUMP,buf,&len,0) < 0) {
        free(buf);
        return (-1);
    }
    lim = buf + len;
    next = buf;
# 如果没有定义 HAVE_SYS_SYSCTL_H 也没有定义 HAVE_GETKERNINFO
    struct rt_giarg giarg, *gp;

    # 将 giarg 结构清零
    memset(&giarg, 0, sizeof(giarg));
    giarg.gi_op = KINFO_RT_DUMP;

    # 调用 ioctl 函数获取路由表信息
    if (ioctl(r->fd, RTSTR_GETROUTE, &giarg) < 0)
        return (-1);

    # 分配内存空间用于存储路由表信息
    if ((buf = malloc(giarg.gi_size)) == NULL)
        return (-1);

    gp = (struct rt_giarg *)buf;
    # 设置gp结构体的大小为giarg结构体的大小
    gp->gi_size = giarg.gi_size;
    # 设置gp结构体的操作类型为KINFO_RT_DUMP
    gp->gi_op = KINFO_RT_DUMP;
    # 设置gp结构体的数据存放位置为buf
    gp->gi_where = buf;
    # 设置gp结构体的参数为RTF_UP和RTF_GATEWAY的按位或结果
    gp->gi_arg = RTF_UP | RTF_GATEWAY;

    # 如果ioctl函数执行失败
    if (ioctl(r->fd, RTSTR_GETROUTE, buf) < 0) {
        # 释放buf内存
        free(buf);
        # 返回-1
        return (-1);
    }
    # 设置lim为buf加上gp结构体的大小
    lim = buf + gp->gi_size;
    # 设置next为buf加上giarg结构体的大小
    next = buf + sizeof(giarg);
#endif
    /* 这个循环假设 RTA_DST、RTA_GATEWAY 和 RTA_NETMASK 分别具有数值 1、2 和 4。
     * 参考 Unix Network Programming 第 494 页的 get_rtaddrs 函数。*/
    for (ret = 0; next < lim; next += rtm->rtm_msglen) {
        char namebuf[IF_NAMESIZE];
        sa_family_t sfam;
        rtm = (struct rt_msghdr *)next;
        sa = (struct sockaddr *)(rtm + 1);
        /* 查看地址族 */
        sfam = sa->sa_family;

        if (if_indextoname(rtm->rtm_index, namebuf) == NULL)
            continue;
        strlcpy(entry.intf_name, namebuf, sizeof(entry.intf_name));

        if ((rtm->rtm_addrs & RTA_DST) == 0)
            /* 需要目的地。*/
            continue;
        if (addr_ston(sa, &entry.route_dst) < 0)
            continue;

        if ((rtm->rtm_addrs & RTA_GATEWAY) == 0)
            /* 需要网关。*/
            continue;
        sa = NEXTSA(sa);
        if (addr_ston_gateway(&entry.route_dst, sa, &entry.route_gw) < 0)
            continue;
        
        if (entry.route_dst.addr_type != entry.route_gw.addr_type ||
            (entry.route_dst.addr_type != ADDR_TYPE_IP &&
            entry.route_dst.addr_type != ADDR_TYPE_IP6))
            continue;

        if (rtm->rtm_addrs & RTA_NETMASK) {
            sa = NEXTSA(sa);
            /* FreeBSD 对于 IPv6 使用不同的 AF 表示子网掩码。强制使用相同的 AF。*/
            sa->sa_family = sfam;
            if (addr_stob(sa, &entry.route_dst.addr_bits) < 0)
                continue;
        }

        entry.metric = 0;

        if ((ret = callback(&entry, arg)) != 0)
            break;
    }
    free(buf);
    
    return (ret);
}
#elif defined(HAVE_STREAMS_MIB2)

#ifdef IRE_DEFAULT        /* 这表示 Solaris 5.6 */
/* 虽然我不确定它们是否兼容 -- masaki */
#define IRE_ROUTE IRE_CACHE
#define IRE_ROUTE_REDIRECT IRE_HOST_REDIRECT
#endif /* IRE_DEFAULT */

int
route_loop(route_t *r, route_handler callback, void *arg)
{
    # 定义路由表条目结构
    struct route_entry entry;
    # 定义字符串缓冲区消息结构
    struct strbuf msg;
    # 定义 T_optmgmt_req 结构
    struct T_optmgmt_req *tor;
    # 定义 T_optmgmt_ack 结构
    struct T_optmgmt_ack *toa;
    # 定义 T_error_ack 结构
    struct T_error_ack *tea;
    # 定义选项头结构
    struct opthdr *opt;
    # 定义字节流缓冲区
    u_char buf[8192];
    # 定义标志、返回值、路由表、返回结果
    int flags, rc, rtable, ret;
    
    # 将 buf 强制转换为 T_optmgmt_req 结构
    tor = (struct T_optmgmt_req *)buf;
    # 将 buf 强制转换为 T_optmgmt_ack 结构
    toa = (struct T_optmgmt_ack *)buf;
    # 将 buf 强制转换为 T_error_ack 结构
    tea = (struct T_error_ack *)buf;
    
    # 设置 T_optmgmt_req 结构的各个字段值
    tor->PRIM_type = T_OPTMGMT_REQ;
    tor->OPT_offset = sizeof(*tor);
    tor->OPT_length = sizeof(*opt);
    tor->MGMT_flags = T_CURRENT;
    
    # 将 opt 指向 T_optmgmt_req 结构后面的位置
    opt = (struct opthdr *)(tor + 1);
    # 设置选项头结构的 level、name、len 字段值
    opt->level = MIB2_IP;
    opt->name = opt->len = 0;
    
    # 设置消息结构的最大长度和实际长度，以及消息缓冲区的位置
    msg.maxlen = sizeof(buf);
    msg.len = sizeof(*tor) + sizeof(*opt);
    msg.buf = buf;
    
    # 如果向 r->ip_fd 发送消息失败，则返回 -1
    if (putmsg(r->ip_fd, &msg, NULL, 0) < 0)
        return (-1);
    
    # 将 opt 指向 T_optmgmt_ack 结构后面的位置
    opt = (struct opthdr *)(toa + 1);
    # 重新设置消息结构的最大长度
    msg.maxlen = sizeof(buf);
    
    # 重复上述过程，设置 T_optmgmt_req 结构的各个字段值
    tor = (struct T_optmgmt_req *)buf;
    toa = (struct T_optmgmt_ack *)buf;
    tea = (struct T_error_ack *)buf;
    tor->PRIM_type = T_OPTMGMT_REQ;
    tor->OPT_offset = sizeof(*tor);
    tor->OPT_length = sizeof(*opt);
    tor->MGMT_flags = T_CURRENT;
    opt = (struct opthdr *)(tor + 1);
    opt->level = MIB2_IP6;
    opt->name = opt->len = 0;
    msg.maxlen = sizeof(buf);
    msg.len = sizeof(*tor) + sizeof(*opt);
    msg.buf = buf;
    if (putmsg(r->ip_fd, &msg, NULL, 0) < 0)
        return (-1);
    opt = (struct opthdr *)(toa + 1);
    msg.maxlen = sizeof(buf);
    
    # 返回 0
    return (0);
# 如果定义了 HAVE_NET_RADIX_H，则包含 nlist.h 头文件
#elif defined(HAVE_NET_RADIX_H)
/* XXX - Tru64, others? */
#include <nlist.h>

# 定义一个函数 _kread，用于从指定文件描述符和地址处读取指定长度的数据到缓冲区
static int
_kread(int fd, void *addr, void *buf, int len)
{
    if (lseek(fd, (off_t)addr, SEEK_SET) == (off_t)-1L)
        return (-1);
    return (read(fd, buf, len) == len ? 0 : -1);
}

# 定义一个函数 _radix_walk，用于遍历路由表
static int
_radix_walk(int fd, struct radix_node *rn, route_handler callback, void *arg)
{
    struct radix_node rnode;
    struct rtentry rt;
    struct sockaddr_in sin;
    struct route_entry entry;
    int ret = 0;
 again:
    _kread(fd, rn, &rnode, sizeof(rnode));
    if (rnode.rn_b < 0) {
        if (!(rnode.rn_flags & RNF_ROOT)) {
            entry.intf_name[0] = '\0';
            _kread(fd, rn, &rt, sizeof(rt));
            _kread(fd, rt_key(&rt), &sin, sizeof(sin));
            addr_ston((struct sockaddr *)&sin, &entry.route_dst);
            if (!(rt.rt_flags & RTF_HOST)) {
                _kread(fd, rt_mask(&rt), &sin, sizeof(sin));
                addr_stob((struct sockaddr *)&sin,
                    &entry.route_dst.addr_bits);
            }
            _kread(fd, rt.rt_gateway, &sin, sizeof(sin));
            addr_ston((struct sockaddr *)&sin, &entry.route_gw);
            entry.metric = 0;
            if ((ret = callback(&entry, arg)) != 0)
                return (ret);
        }
        if ((rn = rnode.rn_dupedkey))
            goto again;
    } else {
        rn = rnode.rn_r;
        if ((ret = _radix_walk(fd, rnode.rn_l, callback, arg)) != 0)
            return (ret);
        if ((ret = _radix_walk(fd, rn, callback, arg)) != 0)
            return (ret);
    }
    return (ret);
}

# 定义一个函数 route_loop，用于循环遍历路由表
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    struct radix_node_head *rnh, head;
    struct nlist nl[2];
    int fd, ret = 0;

    # 初始化 nl 数组
    memset(nl, 0, sizeof(nl));
    nl[0].n_name = "radix_node_head";
    
    # 获取内核符号表中的符号信息
    if (knlist(nl) < 0 || nl[0].n_type == 0 ||
        (fd = open("/dev/kmem", O_RDONLY, 0)) < 0)
        return (-1);
}
    # 从文件描述符中读取指定位置的数据到 rnh 变量中，大小为 rnh 结构体的大小
    for (_kread(fd, (void *)nl[0].n_value, &rnh, sizeof(rnh));
        rnh != NULL; rnh = head.rnh_next) {
        # 从文件描述符中读取 rnh 指向的内存块的数据到 head 变量中，大小为 head 结构体的大小
        _kread(fd, rnh, &head, sizeof(head));
        # 只处理 IPv4 地址的情况
        /* XXX - only IPv4 for now... */
        if (head.rnh_af == AF_INET) {
            # 调用 _radix_walk 函数遍历树结构，处理回调函数 callback，传入参数 arg
            if ((ret = _radix_walk(fd, head.rnh_treetop,
                 callback, arg)) != 0)
                break;
        }
    }
    # 关闭文件描述符
    close(fd);
    # 返回 ret 变量的值
    return (ret);
}
#else
// 如果没有定义宏，则定义一个返回-1并设置errno为ENOSYS的函数
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    errno = ENOSYS;
    return (-1);
}
#endif

// 关闭路由对象
route_t *
route_close(route_t *r)
{
    // 如果路由对象不为空
    if (r != NULL) {
#ifdef HAVE_STREAMS_MIB2
        // 如果定义了宏HAVE_STREAMS_MIB2，并且ip_fd大于等于0
        if (r->ip_fd >= 0)
            // 关闭ip_fd
            close(r->ip_fd);
#endif
        // 如果fd大于等于0
        if (r->fd >= 0)
            // 关闭fd
            close(r->fd);
        // 释放路由对象内存
        free(r);
    }
    // 返回空指针
    return (NULL);
}
```