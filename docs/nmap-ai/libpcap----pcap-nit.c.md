# `nmap\libpcap\pcap-nit.c`

```cpp
/*
 * 版权声明
 * 版权所有（c）1990年，1991年，1992年，1993年，1994年，1995年，1996年
 * 加利福尼亚大学董事会。保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，前提是：（1）源代码分发
 * 保留上述版权声明和本段文字的完整性，（2）包含二进制代码的分发包括上述版权声明和
 * 本段文字的完整性在文档或其他提供的材料中，（3）所有提到此软件特性或使用的广告材料
 * 显示以下声明：
 * “本产品包括由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从此软件派生的产品。
 * 本软件按“原样”提供，没有任何明示或暗示的保证，包括但不限于
 * 商业适销性和特定用途的暗示保证。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>

#include <net/if.h>
#include <net/nit.h>

#include <netinet/in.h>
#include <netinet/in_systm.h>
#include <netinet/ip.h>
#include <netinet/if_ether.h>
#include <netinet/ip_var.h>
#include <netinet/udp.h>
#include <netinet/udp_var.h>
#include <netinet/tcp.h>
#include <netinet/tcpip.h>

#include <errno.h>
#include <stdio.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * NIT的块大小。这是读取调用的缓冲区大小。
 */
#define CHUNKSIZE (2*1024)

/*
 * NIT使用的总缓冲空间。
 */
#define BUFSPACE (4*CHUNKSIZE)

/* 前向声明 */
static int nit_setflags(int, int, int, char *);

/*
 * Private data for capturing on NIT devices.
 */
struct pcap_nit {
    struct pcap_stat stat;
};

static int
pcap_stats_nit(pcap_t *p, struct pcap_stat *ps)
{
    struct pcap_nit *pn = p->priv;

    /*
     * "ps_recv" counts packets handed to the filter, not packets
     * that passed the filter.  As filtering is done in userland,
     * this does not include packets dropped because we ran out
     * of buffer space.
     *
     * "ps_drop" presumably counts packets dropped by the socket
     * because of flow control requirements or resource exhaustion;
     * it doesn't count packets dropped by the interface driver.
     * As filtering is done in userland, it counts packets regardless
     * of whether they would've passed the filter.
     *
     * These statistics don't include packets not yet read from the
     * kernel by libpcap or packets not yet read from libpcap by the
     * application.
     */
    *ps = pn->stat;
    return (0);
}

static int
pcap_read_nit(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    struct pcap_nit *pn = p->priv;
    register int cc, n;
    register u_char *bp, *cp, *ep;
    register struct nit_hdr *nh;
    register int caplen;

    cc = p->cc;
    if (cc == 0) {
        cc = read(p->fd, (char *)p->buffer, p->bufsize);
        if (cc < 0) {
            if (errno == EWOULDBLOCK)
                return (0);
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "pcap_read");
            return (-1);
        }
        bp = (u_char *)p->buffer;
    } else
        bp = p->bp;

    /*
     * Loop through each packet.  The increment expression
     * rounds up to the next int boundary past the end of
     * the previous packet.
     *
     * This assumes that a single buffer of packets will have
     * <= INT_MAX packets, so the packet count doesn't overflow.
     */
    n = 0;
    ep = bp + cc;
    while (bp < ep) {
        /*
         * 检查是否调用了 "pcap_breakloop()" 函数
         * 如果是，立即返回 - 如果我们还没有读取任何数据包，则清除标志并返回 -2，表示我们被告知退出循环，否则保持标志设置，这样下一次调用将在没有读取任何数据包的情况下退出循环，并返回到目前为止我们处理的数据包数量。
         */
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (-2);
            } else {
                p->cc = ep - bp;
                p->bp = bp;
                return (n);
            }
        }

        nh = (struct nit_hdr *)bp;
        cp = bp + sizeof(*nh);

        switch (nh->nh_state) {

        case NIT_CATCH:
            break;

        case NIT_NOMBUF:
        case NIT_NOCLUSTER:
        case NIT_NOSPACE:
            pn->stat.ps_drop = nh->nh_dropped;
            continue;

        case NIT_SEQNO:
            continue;

        default:
            snprintf(p->errbuf, sizeof(p->errbuf),
                "bad nit state %d", nh->nh_state);
            return (-1);
        }
        ++pn->stat.ps_recv;
        bp += ((sizeof(struct nit_hdr) + nh->nh_datalen +
            sizeof(int) - 1) & ~(sizeof(int) - 1));

        caplen = nh->nh_wirelen;
        if (caplen > p->snapshot)
            caplen = p->snapshot;
        if (pcap_filter(p->fcode.bf_insns, cp, nh->nh_wirelen, caplen)) {
            struct pcap_pkthdr h;
            h.ts = nh->nh_timestamp;
            h.len = nh->nh_wirelen;
            h.caplen = caplen;
            (*callback)(user, &h, cp);
            if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
                p->cc = ep - bp;
                p->bp = bp;
                return (n);
            }
        }
    }
    p->cc = 0;
    return (n);
# 设置网络接口的发送数据包
static int
pcap_inject_nit(pcap_t *p, const void *buf, int size)
{
    # 定义一个套接字地址结构
    struct sockaddr sa;
    # 定义一个返回值
    int ret;

    # 将套接字地址结构清零
    memset(&sa, 0, sizeof(sa));
    # 将设备名称复制到套接字地址结构中
    strncpy(sa.sa_data, device, sizeof(sa.sa_data));
    # 发送数据包
    ret = sendto(p->fd, buf, size, 0, &sa, sizeof(sa));
    # 如果发送失败，则设置错误信息并返回-1
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
    # 返回发送的字节数
    return (ret);
}

# 设置网络接口的标志
static int
nit_setflags(pcap_t *p)
{
    # 定义一个网络接口控制结构
    struct nit_ioc nioc;

    # 将网络接口控制结构清零
    memset(&nioc, 0, sizeof(nioc));
    # 设置网络接口控制结构的参数
    nioc.nioc_typetomatch = NT_ALLTYPES;
    nioc.nioc_snaplen = p->snapshot;
    nioc.nioc_bufalign = sizeof(int);
    nioc.nioc_bufoffset = 0;

    # 如果设置了缓冲区大小，则使用设置的大小，否则使用默认大小
    if (p->opt.buffer_size != 0)
        nioc.nioc_bufspace = p->opt.buffer_size;
    else {
        /* 默认缓冲区大小 */
        nioc.nioc_bufspace = BUFSPACE;
    }

    # 如果设置了立即模式，则设置数据块大小为0
    if (p->opt.immediate) {
        nioc.nioc_chunksize = 0;
    } else
        nioc.nioc_chunksize = CHUNKSIZE;
    # 如果设置了超时时间，则设置超时标志和超时时间
    if (p->opt.timeout != 0) {
        nioc.nioc_flags |= NF_TIMEOUT;
        nioc.nioc_timeout.tv_sec = p->opt.timeout / 1000;
        nioc.nioc_timeout.tv_usec = (p->opt.timeout * 1000) % 1000000;
    }
    # 如果设置了混杂模式，则设置混杂标志
    if (p->opt.promisc)
        nioc.nioc_flags |= NF_PROMISC;

    # 使用ioctl设置网络接口参数
    if (ioctl(p->fd, SIOCSNIT, &nioc) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCSNIT");
        return (-1);
    }
    # 返回0表示成功
    return (0);
}

# 激活网络接口
static int
pcap_activate_nit(pcap_t *p)
{
    # 定义一个文件描述符和一个网络接口地址结构
    int fd;
    struct sockaddr_nit snit;

    # 如果设置了监控模式，则返回不支持监控模式的错误
    if (p->opt.rfmon) {
        /*
         * 在SunOS 3.x或更早版本上不支持监控模式（不支持支持Wi-Fi设备的硬件！）。
         */
        return (PCAP_ERROR_RFMON_NOTSUP);
    }
    /*
     * 将负的快照值（无效值）、快照值为0（未指定值）或大于正常最大值的值，转换为允许的最大值。
     *
     * 如果某些应用程序真的*需要*更大的快照长度，我们应该增加MAXIMUM_SNAPLEN。
     */
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    if (p->snapshot < 96)
        /*
         * NIT要求快照长度至少为96。
         */
        p->snapshot = 96;

    memset(p, 0, sizeof(*p));
    p->fd = fd = socket(AF_NIT, SOCK_RAW, NITPROTO_RAW);
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "socket");
        goto bad;
    }
    snit.snit_family = AF_NIT;
    (void)strncpy(snit.snit_ifname, p->opt.device, NITIFSIZ);

    if (bind(fd, (struct sockaddr *)&snit, sizeof(snit))) {
        /*
         * XXX - 可能有一个特定的绑定错误意味着“没有这样的设备”，
         * 以及一个特定的绑定错误意味着“该设备不支持NIT”；
         * 如果它们最终都意味着“NIT不知道该设备”，它们可能是相同的错误。
         */
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "bind: %s", snit.snit_ifname);
        goto bad;
    }
    if (nit_setflags(p) < 0)
        goto bad;

    /*
     * NIT仅支持以太网。
     */
    p->linktype = DLT_EN10MB;

    p->bufsize = BUFSPACE;
    p->buffer = malloc(p->bufsize);
    if (p->buffer == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        goto bad;
    }

    /*
     * “p->fd”是一个套接字，因此“select()”应该在其上工作。
     */
    p->selectable_fd = p->fd;
    /*
     * 分配一个包含 DLT_EN10MB 和 DLT_DOCSIS 的链路层类型列表，以便应用程序可以选择它，
     * 以防你正在捕获由 Cisco Cable Modem Termination System 放在以太网上的 DOCSIS 流量
     * （它不会在以太网上放置以太网头，而是在低级以太网帧内放置原始的 DOCSIS 帧）。
     */
    p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
    /*
     * 如果分配失败，就让列表保持为空。
     */
    if (p->dlt_list != NULL) {
        p->dlt_list[0] = DLT_EN10MB;
        p->dlt_list[1] = DLT_DOCSIS;
        p->dlt_count = 2;
    }

    p->read_op = pcap_read_nit;
    p->inject_op = pcap_inject_nit;
    p->setfilter_op = install_bpf_program;    /* 没有内核过滤 */
    p->setdirection_op = NULL;    /* 未实现 */
    p->set_datalink_op = NULL;    /* 无法更改数据链路类型 */
    p->getnonblock_op = pcap_getnonblock_fd;
    p->setnonblock_op = pcap_setnonblock_fd;
    p->stats_op = pcap_stats_nit;

    return (0);
 bad:
    pcap_cleanup_live_common(p);
    return (PCAP_ERROR);
}

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    pcap_t *p;

    // 使用通用的创建函数创建一个 pcap_t 结构体，传入错误缓冲区和 struct pcap_nit 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_nit);
    if (p == NULL)
        return (NULL);

    // 设置 pcap_t 结构体的激活操作为 pcap_activate_nit
    p->activate_op = pcap_activate_nit;
    return (p);
}

/*
 * XXX - there's probably a particular bind error that means "that device
 * doesn't support NIT"; if so, we should try a bind and use that.
 */
static int
can_be_bound(const char *name _U_)
{
    // 返回 1，表示可以绑定
    return (1);
}

static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
    /*
     * Nothing we can do.
     * XXX - is there a way to find out whether an adapter has
     * something plugged into it?
     */
    // 返回 0，表示无法获取接口标志
    return (0);
}

// 查找平台特定的网络接口
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    // 调用 pcap_findalldevs_interfaces 函数查找网络接口
    return (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
        get_if_flags));
}

/*
 * Libpcap version string.
 */
// 返回 Libpcap 版本字符串
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING);
}
```