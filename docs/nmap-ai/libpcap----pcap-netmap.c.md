# `nmap\libpcap\pcap-netmap.c`

```
/*
 * 版权声明
 * 版权所有（C）2014 Luigi Rizzo。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *
 *   1. 源代码的重新分发必须保留上述版权声明、此条件列表和以下免责声明。
 *   2. 以二进制形式重新分发时，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
 *
 * 本软件由作者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。无论在任何情况下，作者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <poll.h>           // 包含 poll 函数相关的头文件
#include <errno.h>          // 包含错误码相关的头文件
#include <netdb.h>          // 包含网络数据库操作相关的头文件
#include <stdio.h>          // 包含标准输入输出相关的头文件
#include <stdlib.h>         // 包含标准库相关的头文件
#include <string.h>         // 包含字符串操作相关的头文件
#include <unistd.h>         // 包含 POSIX 系统服务相关的头文件

#define NETMAP_WITH_LIBS     // 定义 NETMAP_WITH_LIBS 宏
#include <net/netmap_user.h> // 包含 netmap 用户态库相关的头文件

#include "pcap-int.h"        // 包含 pcap 库内部实现相关的头文件
#include "pcap-netmap.h"     // 包含 pcap-netmap 相关的头文件

#ifndef __FreeBSD__
  /*
   * 在 FreeBSD 上使用 IFF_PPROMISC，它在 ifr_flagshigh 中。
   * 在其他平台上重新映射为 IFF_PROMISC。
   *
   * XXX - DragonFly BSD？
   */
  #define IFF_PPROMISC    IFF_PROMISC
#endif /* __FreeBSD__ */

struct pcap_netmap {
    struct nm_desc *d;    /* nm_open() 返回的指针 */
    pcap_handler cb;    /* 定义一个名为cb的pcap_handler类型变量，用于回调函数和参数 */
    u_char *cb_arg;    /* 定义一个名为cb_arg的u_char类型指针变量 */
    int must_clear_promisc;    /* 定义一个名为must_clear_promisc的整型变量，用于标记是否需要清除混杂模式 */
    uint64_t rx_pkts;    /* 定义一个名为rx_pkts的uint64_t类型变量，用于记录接收到的数据包数量 */
};

static int
pcap_netmap_stats(pcap_t *p, struct pcap_stat *ps)
{
    // 获取 pcap_netmap 结构体指针
    struct pcap_netmap *pn = p->priv;

    // 设置接收的数据包数量
    ps->ps_recv = (u_int)pn->rx_pkts;
    // 设置丢弃的数据包数量为 0
    ps->ps_drop = 0;
    // 设置接口丢弃的数据包数量为 0
    ps->ps_ifdrop = 0;
    return 0;
}


static void
pcap_netmap_filter(u_char *arg, struct pcap_pkthdr *h, const u_char *buf)
{
    // 获取 pcap_t 结构体指针
    pcap_t *p = (pcap_t *)arg;
    // 获取 pcap_netmap 结构体指针
    struct pcap_netmap *pn = p->priv;
    // 获取过滤器指令
    const struct bpf_insn *pc = p->fcode.bf_insns;

    // 接收的数据包数量加一
    ++pn->rx_pkts;
    // 如果过滤器指令为空或者过滤通过，则调用回调函数
    if (pc == NULL || pcap_filter(pc, buf, h->len, h->caplen))
        pn->cb(pn->cb_arg, h, buf);
}


static int
pcap_netmap_dispatch(pcap_t *p, int cnt, pcap_handler cb, u_char *user)
{
    int ret;
    // 获取 pcap_netmap 结构体指针
    struct pcap_netmap *pn = p->priv;
    // 获取 nm_desc 结构体指针
    struct nm_desc *d = pn->d;
    // 设置 pollfd 结构体
    struct pollfd pfd = { .fd = p->fd, .events = POLLIN, .revents = 0 };

    // 设置回调函数和回调参数
    pn->cb = cb;
    pn->cb_arg = user;

    for (;;) {
        if (p->break_loop) {
            p->break_loop = 0;
            return PCAP_ERROR_BREAK;
        }
        /* nm_dispatch won't run forever */

        // 调用 nm_dispatch 函数
        ret = nm_dispatch((void *)d, cnt, (void *)pcap_netmap_filter, (void *)p);
        if (ret != 0)
            break;
        errno = 0;
        // 调用 poll 函数
        ret = poll(&pfd, 1, p->opt.timeout);
    }
    return ret;
}


/* XXX need to check the NIOCTXSYNC/poll */
static int
pcap_netmap_inject(pcap_t *p, const void *buf, int size)
{
    // 获取 pcap_netmap 结构体指针
    struct pcap_netmap *pn = p->priv;
    // 获取 nm_desc 结构体指针
    struct nm_desc *d = pn->d;

    // 调用 nm_inject 函数
    return nm_inject(d, buf, size);
}


static int
pcap_netmap_ioctl(pcap_t *p, u_long what, uint32_t *if_flags)
{
    // 获取 pcap_netmap 结构体指针
    struct pcap_netmap *pn = p->priv;
    // 获取 nm_desc 结构体指针
    struct nm_desc *d = pn->d;
    // 定义 ifreq 结构体
    struct ifreq ifr;
    int error, fd = d->fd;

#ifdef linux
    // 在 Linux 下创建 socket
    fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (fd < 0) {
        fprintf(stderr, "Error: cannot get device control socket.\n");
        return -1;
    }
#endif /* linux */
    // 清空 ifreq 结构体
    bzero(&ifr, sizeof(ifr));
    // 将设备名称复制到 ifreq 结构体
    strncpy(ifr.ifr_name, d->req.nr_name, sizeof(ifr.ifr_name));
    switch (what) {
    # 设置接口标志位
    case SIOCSIFFLAGS:
        '''
         * 我们传递的标志位是32位无符号整数。
         *
         * 在大多数甚至所有的UNIX系统上，ifr_flags是16位有符号整数，
         * 将较长的无符号值赋给较短的有符号值的结果是实现定义的
         * （即使在实践中，它将在我们支持的所有平台上执行预期的操作）。
         * 因此，我们屏蔽掉高16位。
         '''
        # 将传入的标志位与0xffff进行按位与操作，屏蔽掉高16位
        ifr.ifr_flags = *if_flags & 0xffff;
#ifdef __FreeBSD__
        /*
         * 在 FreeBSD 中，我们需要设置高阶标志，
         * 因为我们使用的是 IFF_PPROMISC，它在这些位中。
         *
         * XXX - DragonFly BSD?
         */
        ifr.ifr_flagshigh = *if_flags >> 16;
#endif /* __FreeBSD__ */
        break;
    }
    // 使用 ioctl 函数执行设备特定命令
    error = ioctl(fd, what, &ifr);
    if (!error) {
        switch (what) {
        case SIOCGIFFLAGS:
            /*
             * 我们返回的标志是 32 位的。
             *
             * 在大多数 UN*X 系统中，ifr_flags 是 16 位有符号数，并且会被符号扩展，
             * 因此这些标志的高 16 位将被强制设置。因此我们屏蔽了符号扩展值的高 16 位。
             */
            *if_flags = ifr.ifr_flags & 0xffff;
#ifdef __FreeBSD__
            /*
             * 在 FreeBSD 中，我们需要返回高阶标志，
             * 因为我们使用的是 IFF_PPROMISC，它在这些位中。
             *
             * XXX - DragonFly BSD?
             */
            *if_flags |= (ifr.ifr_flagshigh << 16);
#endif /* __FreeBSD__ */
        }
    }
#ifdef linux
    close(fd);
#endif /* linux */
    // 如果出现错误则返回 -1，否则返回 0
    return error ? -1 : 0;
}


static void
pcap_netmap_close(pcap_t *p)
{
    struct pcap_netmap *pn = p->priv;
    struct nm_desc *d = pn->d;
    uint32_t if_flags = 0;

    if (pn->must_clear_promisc) {
        // 获取接口标志
        pcap_netmap_ioctl(p, SIOCGIFFLAGS, &if_flags); /* fetch flags */
        if (if_flags & IFF_PPROMISC) {
            // 清除混杂模式
            if_flags &= ~IFF_PPROMISC;
            pcap_netmap_ioctl(p, SIOCSIFFLAGS, &if_flags);
        }
    }
    // 关闭 netmap 描述符
    nm_close(d);
    // 清理 pcap 公共资源
    pcap_cleanup_live_common(p);
}


static int
pcap_netmap_activate(pcap_t *p)
{
    struct pcap_netmap *pn = p->priv;
    struct nm_desc *d;
    uint32_t if_flags = 0;

    // 打开 netmap 描述符
    d = nm_open(p->opt.device, NULL, 0, NULL);
    # 如果指针 d 为空
    if (d == NULL) {
        # 格式化错误消息，将错误号和设备名称添加到错误缓冲区中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "netmap open: cannot access %s",
            p->opt.device);
        # 清理并释放 pcap 对象资源
        pcap_cleanup_live_common(p);
        # 返回 pcap 错误代码
        return (PCAP_ERROR);
    }
#if 0
    # 如果条件为假，则向标准错误流输出设备名称、特权、文件描述符和端口范围
    fprintf(stderr, "%s device %s priv %p fd %d ports %d..%d\n",
        __FUNCTION__, p->opt.device, d, d->fd,
        d->first_rx_ring, d->last_rx_ring);
#endif
    # 将指针 d 赋值给结构体成员 pn->d
    pn->d = d;
    # 将文件描述符 d->fd 赋值给结构体成员 p->fd
    p->fd = d->fd;

    """
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，
     * 转换为允许的最大值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     """
    # 如果快照值小于等于 0 或大于 MAXIMUM_SNAPLEN，则将其设置为 MAXIMUM_SNAPLEN
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    # 如果启用混杂模式并且请求的环不是软件环
    if (p->opt.promisc && !(d->req.nr_ringid & NETMAP_SW_RING)) {
        # 获取接口标志
        pcap_netmap_ioctl(p, SIOCGIFFLAGS, &if_flags); /* fetch flags */
        # 如果接口标志不包含 IFF_PPROMISC
        if (!(if_flags & IFF_PPROMISC)) {
            # 设置必须清除混杂模式标志
            pn->must_clear_promisc = 1;
            # 将接口标志设置为包含 IFF_PPROMISC
            if_flags |= IFF_PPROMISC;
            pcap_netmap_ioctl(p, SIOCSIFFLAGS, &if_flags);
        }
    }
    # 设置数据链路类型为 DLT_EN10MB
    p->linktype = DLT_EN10MB;
    # 设置可选择的文件描述符为 p->fd
    p->selectable_fd = p->fd;
    # 设置读操作为 pcap_netmap_dispatch
    p->read_op = pcap_netmap_dispatch;
    # 设置注入操作为 pcap_netmap_inject
    p->inject_op = pcap_netmap_inject;
    # 设置过滤器安装操作为 install_bpf_program
    p->setfilter_op = install_bpf_program;
    # 设置方向设置操作为 NULL
    p->setdirection_op = NULL;
    # 设置数据链路设置操作为 NULL
    p->set_datalink_op = NULL;
    # 设置获取非阻塞操作为 pcap_getnonblock_fd
    p->getnonblock_op = pcap_getnonblock_fd;
    # 设置设置非阻塞操作为 pcap_setnonblock_fd
    p->setnonblock_op = pcap_setnonblock_fd;
    # 设置统计操作为 pcap_netmap_stats
    p->stats_op = pcap_netmap_stats;
    # 设置清理操作为 pcap_netmap_close
    p->cleanup_op = pcap_netmap_close;

    # 返回 0
    return (0);
}


pcap_t *
pcap_netmap_create(const char *device, char *ebuf, int *is_ours)
{
    pcap_t *p;

    # 检查设备名称是否以 "netmap:" 或 "vale" 开头，如果是则将 is_ours 设置为真
    *is_ours = (!strncmp(device, "netmap:", 7) || !strncmp(device, "vale", 4));
    # 如果不是我们的设备，则返回空指针
    if (! *is_ours)
        return NULL;
    # 创建一个 pcap_t 结构体，并将其赋值给 p
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_netmap);
    # 如果创建失败，则返回空指针
    if (p == NULL)
        return (NULL);
    # 设置激活操作为 pcap_netmap_activate
    p->activate_op = pcap_netmap_activate;
    # 返回 p
    return (p);
}

"""
 * netmap 设备的“设备名称”不是设备的名称，而是指示设备应该如何设置的表达式，
 * 因此无法枚举它们。
 """
# 返回 0
int
# 定义一个函数名为pcap_netmap_findalldevs，接受两个参数：devlistp和err_str
# 该函数返回值为0
pcap_netmap_findalldevs(pcap_if_list_t *devlistp _U_, char *err_str _U_)
{
    return 0;
}
```