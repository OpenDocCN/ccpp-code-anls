# `nmap\libpcap\dlpisubs.c`

```cpp
/*
 * This code is derived from code formerly in pcap-dlpi.c, originally
 * contributed by Atanu Ghosh (atanu@cs.ucl.ac.uk), University College
 * London, and subsequently modified by Guy Harris (guy@alum.mit.edu),
 * Mark Pizzolato <List-tcpdump-workers@subscriptions.pizzolato.net>,
 * Mark C. Brown (mbrown@hp.com), and Sagun Shakya <Sagun.Shakya@Sun.COM>.
 */

/*
 * This file contains dlpi/libdlpi related common functions used
 * by pcap-[dlpi,libdlpi].c.
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifndef DL_IPATM
#define DL_IPATM    0x12    /* ATM Classical IP interface */
#endif

#ifdef HAVE_SYS_BUFMOD_H
    /*
     * Size of a bufmod chunk to pass upstream; that appears to be the
     * biggest value to which you can set it, and setting it to that value
     * (which is bigger than what appears to be the Solaris default of 8192)
     * reduces the number of packet drops.
     */
#define    CHUNKSIZE    65536

    /*
     * Size of the buffer to allocate for packet data we read; it must be
     * large enough to hold a chunk.
     */
#define    PKTBUFSIZE    CHUNKSIZE

#else /* HAVE_SYS_BUFMOD_H */

    /*
     * Size of the buffer to allocate for packet data we read; this is
     * what the value used to be - there's no particular reason why it
     * should be tied to MAXDLBUF, but we'll leave it as this for now.
     */
#define    MAXDLBUF    8192
#define    PKTBUFSIZE    (MAXDLBUF * sizeof(bpf_u_int32))

#endif

#include <sys/types.h>
#include <sys/time.h>
#ifdef HAVE_SYS_BUFMOD_H
#include <sys/bufmod.h>
#endif
#include <sys/dlpi.h>
#include <sys/stream.h>

#include <errno.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stropts.h>
#include <unistd.h>

#ifdef HAVE_LIBDLPI
#include <libdlpi.h>
#endif

#include "pcap-int.h"
#include "dlpisubs.h"

#ifdef HAVE_SYS_BUFMOD_H
static void pcap_stream_err(const char *, int, char *);
#endif

/*
 * Get the packet statistics.
 */
int
# 使用 DLPI 接口获取 pcap_t 结构体的统计信息
pcap_stats_dlpi(pcap_t *p, struct pcap_stat *ps)
{
    # 获取 pcap_t 结构体中的私有数据
    struct pcap_dlpi *pd = p->priv;

    /*
     * "ps_recv" counts packets handed to the filter, not packets
     * that passed the filter.  As filtering is done in userland,
     * this would not include packets dropped because we ran out
     * of buffer space; in order to make this more like other
     * platforms (Linux 2.4 and later, BSDs with BPF), where the
     * "packets received" count includes packets received but dropped
     * due to running out of buffer space, and to keep from confusing
     * applications that, for example, compute packet drop percentages,
     * we also make it count packets dropped by "bufmod" (otherwise we
     * might run the risk of the packet drop count being bigger than
     * the received-packet count).
     *
     * "ps_drop" counts packets dropped by "bufmod" because of
     * flow control requirements or resource exhaustion; it doesn't
     * count packets dropped by the interface driver, or packets
     * dropped upstream.  As filtering is done in userland, it counts
     * packets regardless of whether they would've passed the filter.
     *
     * These statistics don't include packets not yet read from
     * the kernel by libpcap, but they may include packets not
     * yet read from libpcap by the application.
     */
    # 将私有数据中的统计信息赋值给传入的结构体指针
    *ps = pd->stat;

    /*
     * Add in the drop count, as per the above comment.
     */
    # 将丢弃的数据包数量加到接收的数据包数量中
    ps->ps_recv += ps->ps_drop;
    # 返回 0 表示成功
    return (0);
}

/*
 * Does the processor for which we're compiling this support aligned loads?
 */
# 检查当前编译的处理器是否支持对齐加载
#if (defined(__i386__) || defined(_M_IX86) || defined(__X86__) || defined(__x86_64__) || defined(_M_X64)) || \
    (defined(__arm__) || defined(_M_ARM) || defined(__aarch64__)) || \
    (defined(__m68k__) && (!defined(__mc68000__) && !defined(__mc68010__))) || \
    (defined(__ppc__) || defined(__ppc64__) || defined(_M_PPC) || defined(_ARCH_PPC) || defined(_ARCH_PPC64)) || \
    # 如果定义了 __s390__ 或者 __s390x__ 或者 __zarch__，则执行下面的注释
    /* Yes, it does. */
#else
    /* 如果不满足条件，定义 REQUIRE_ALIGNMENT */
    #define REQUIRE_ALIGNMENT
#endif

/*
 * 循环遍历数据包，并对每个数据包调用回调函数。
 * 返回读取的数据包数量。
 */
int
pcap_process_pkts(pcap_t *p, pcap_handler callback, u_char *user,
    int count, u_char *bufp, int len)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_dlpi *pd = p->priv;
    int n, caplen, origlen;
    u_char *ep, *pk;
    struct pcap_pkthdr pkthdr;
#ifdef HAVE_SYS_BUFMOD_H
    struct sb_hdr *sbp;
#ifdef REQUIRE_ALIGNMENT
    struct sb_hdr sbhdr;
#endif
#endif

    /*
     * 循环遍历数据包。
     *
     * 这假设一个数据包的单个缓冲区将有 <= INT_MAX 个数据包，因此数据包计数不会溢出。
     */
    ep = bufp + len;
    n = 0;

#ifdef HAVE_SYS_BUFMOD_H
    while (bufp < ep) {
        /*
         * "pcap_breakloop()" 被调用了吗？
         * 如果是，立即返回 - 如果我们还没有读取任何数据包，清除标志并返回 -2 表示我们被告知中断循环，否则保持标志设置，这样 *下一次* 调用将在没有读取任何数据包的情况下中断循环，并返回到目前为止我们处理的数据包数量。
         */
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (-2);
            } else {
                p->bp = bufp;
                p->cc = ep - bufp;
                return (n);
            }
        }
#ifdef REQUIRE_ALIGNMENT
        if ((long)bufp & 3) {
            sbp = &sbhdr;
            memcpy(sbp, bufp, sizeof(*sbp));
        } else
#endif
            sbp = (struct sb_hdr *)bufp;
        pd->stat.ps_drop = sbp->sbh_drops;
        pk = bufp + sizeof(*sbp);
        bufp += sbp->sbh_totlen;
        origlen = sbp->sbh_origlen;
        caplen = sbp->sbh_msglen;
#else
        origlen = len;
        caplen = min(p->snapshot, len);
        pk = bufp;
        bufp += caplen;
#endif
        # 增加接收数据包的统计计数
        ++pd->stat.ps_recv;
        # 使用过滤器过滤数据包
        if (pcap_filter(p->fcode.bf_insns, pk, origlen, caplen)) {
#ifdef HAVE_SYS_BUFMOD_H
            # 如果系统支持 bufmod.h，使用 sbh_timestamp 作为时间戳
            pkthdr.ts.tv_sec = sbp->sbh_timestamp.tv_sec;
            pkthdr.ts.tv_usec = sbp->sbh_timestamp.tv_usec;
#else
            # 否则使用 gettimeofday 获取当前时间作为时间戳
            (void) gettimeofday(&pkthdr.ts, NULL);
#endif
            # 设置数据包的长度和捕获长度
            pkthdr.len = origlen;
            pkthdr.caplen = caplen;
            # 确保捕获长度不超过快照长度
            if (pkthdr.caplen > (bpf_u_int32)p->snapshot)
                pkthdr.caplen = (bpf_u_int32)p->snapshot;
            # 调用回调函数处理数据包
            (*callback)(user, &pkthdr, pk);
            # 如果达到指定的数据包数量，返回已捕获的数据包数量
            if (++n >= count && !PACKET_COUNT_IS_UNLIMITED(count)) {
                p->cc = ep - bufp;
                p->bp = bufp;
                return (n);
            }
        }
#ifdef HAVE_SYS_BUFMOD_H
    }
#endif
    # 重置捕获计数器
    p->cc = 0;
    # 返回已捕获的数据包数量
    return (n);
}

/*
 * 处理 MAC 类型。如果没有匹配的 MAC 类型，则返回 -1，否则返回 0。
 */
int
pcap_process_mactype(pcap_t *p, u_int mactype)
{
    int retv = 0;

    switch (mactype) {

    case DL_CSMACD:
    case DL_ETHER:
        # 设置链路类型为 DLT_EN10MB
        p->linktype = DLT_EN10MB;
        # 设置偏移量为 2
        p->offset = 2;
        /*
         * 这是（可能是）真正的以太网捕获；给它一个链路层类型列表，包括 DLT_EN10MB 和 DLT_DOCSIS，
         * 这样应用程序可以选择它，以防你捕获了由 Cisco Cable Modem Termination System 放在以太网上的 DOCSIS 流量
         * （它不会在以太网上放置以太网头，而是在低级以太网封装中放置原始的 DOCSIS 帧）。
         */
        # 分配链路类型列表的内存空间
        p->dlt_list = (u_int *)malloc(sizeof(u_int) * 2);
        /*
         * 如果分配失败，将列表保持为空。
         */
        if (p->dlt_list != NULL) {
            p->dlt_list[0] = DLT_EN10MB;
            p->dlt_list[1] = DLT_DOCSIS;
            p->dlt_count = 2;
        }
        break;
    # 如果是 FDDI 类型的数据链路层，则设置链路类型为 FDDI，偏移量为 3
    case DL_FDDI:
        p->linktype = DLT_FDDI;
        p->offset = 3;
        break;

    # 如果是 TPR 类型的数据链路层
    case DL_TPR:
        # XXX - 关于 DL_TPB 呢？那是 Token Bus 吗？
        # 设置链路类型为 IEEE802，偏移量为 2
        p->linktype = DLT_IEEE802;
        p->offset = 2;
        break;
#ifdef HAVE_SOLARIS
    // 如果系统是 Solaris
    case DL_IPATM:
        // 设置链路类型为 DLT_SUNATM
        p->linktype = DLT_SUNATM;
        // 偏移量为 0，适用于 LANE 和 LLC 封装
        p->offset = 0;  
        break;
#endif

#ifdef DL_IPV4
    // 如果支持 IPv4
    case DL_IPV4:
        // 设置链路类型为 DLT_IPV4
        p->linktype = DLT_IPV4;
        // 偏移量为 0
        p->offset = 0;
        break;
#endif

#ifdef DL_IPV6
    // 如果支持 IPv6
    case DL_IPV6:
        // 设置链路类型为 DLT_IPV6
        p->linktype = DLT_IPV6;
        // 偏移量为 0
        p->offset = 0;
        break;
#endif

#ifdef DL_IPNET
    // 如果支持 IPNET
    case DL_IPNET:
        /*
         * XXX - DL_IPNET devices default to "raw IP" rather than
         * "IPNET header"; see
         *
         *    https://seclists.org/tcpdump/2009/q1/202
         *
         * We'd have to do DL_IOC_IPNET_INFO to enable getting
         * the IPNET header.
         */
        // IPNET 设备默认为 "raw IP" 而不是 "IPNET header"
        // 需要执行 DL_IOC_IPNET_INFO 来获取 IPNET header
        p->linktype = DLT_RAW;
        // 偏移量为 0
        p->offset = 0;
        break;
#endif

    default:
        // 默认情况下，设置错误信息并返回 -1
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "unknown mactype 0x%x",
            mactype);
        retv = -1;
    }

    return (retv);
}

#ifdef HAVE_SYS_BUFMOD_H
/*
 * 推送和配置缓冲模块。发生错误返回 -1，否则返回 0。
 */
int
pcap_conf_bufmod(pcap_t *p, int snaplen)
{
    struct timeval to;
    bpf_u_int32 ss, chunksize;

    /* 非标准调用，使数据得到良好的缓冲。 */
    // 如果调用 I_PUSH 失败
    if (ioctl(p->fd, I_PUSH, "bufmod") != 0) {
        pcap_stream_err("I_PUSH bufmod", errno, p->errbuf);
        return (-1);
    }

    ss = snaplen;
    // 如果 snaplen 大于 0 并且设置 snaplen 失败
    if (ss > 0 &&
        strioctl(p->fd, SBIOCSSNAP, sizeof(ss), (char *)&ss) != 0) {
        pcap_stream_err("SBIOCSSNAP", errno, p->errbuf);
        return (-1);
    }

    if (p->opt.immediate) {
        // 设置超时为零，立即传递数据
        to.tv_sec = 0;
        to.tv_usec = 0;
        // 如果设置超时失败
        if (strioctl(p->fd, SBIOCSTIME, sizeof(to), (char *)&to) != 0) {
            pcap_stream_err("SBIOCSTIME", errno, p->errbuf);
            return (-1);
        }
    } else {
        /* 设置 bufmod 超时时间。*/
        if (p->opt.timeout != 0) {
            // 将毫秒转换为秒和微秒
            to.tv_sec = p->opt.timeout / 1000;
            to.tv_usec = (p->opt.timeout * 1000) % 1000000;
            // 如果设置超时时间失败，则返回错误
            if (strioctl(p->fd, SBIOCSTIME, sizeof(to), (char *)&to) != 0) {
                pcap_stream_err("SBIOCSTIME", errno, p->errbuf);
                return (-1);
            }
        }

        /* 设置数据块长度。*/
        chunksize = CHUNKSIZE;
        // 如果设置数据块长度失败，则返回错误
        if (strioctl(p->fd, SBIOCSCHUNK, sizeof(chunksize), (char *)&chunksize)
            != 0) {
            pcap_stream_err("SBIOCSCHUNKP", errno, p->errbuf);
            return (-1);
        }
    }

    // 返回成功
    return (0);
#endif /* HAVE_SYS_BUFMOD_H */

/*
 * 分配数据缓冲区。如果内存分配失败则返回-1，否则返回0。
 */
int
pcap_alloc_databuf(pcap_t *p)
{
    p->bufsize = PKTBUFSIZE; // 设置缓冲区大小为PKTBUFSIZE
    p->buffer = malloc(p->bufsize + p->offset); // 分配缓冲区内存空间
    if (p->buffer == NULL) { // 如果内存分配失败
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc"); // 格式化错误消息
        return (-1); // 返回-1
    }

    return (0); // 返回0
}

/*
 * 发出STREAMS I_STR ioctl。在错误时返回-1，否则在成功时返回返回数据的长度。
 */
int
strioctl(int fd, int cmd, int len, char *dp)
{
    struct strioctl str; // 定义strioctl结构体
    int retv; // 定义返回值变量

    str.ic_cmd = cmd; // 设置命令
    str.ic_timout = -1; // 设置超时时间
    str.ic_len = len; // 设置长度
    str.ic_dp = dp; // 设置数据指针
    if ((retv = ioctl(fd, I_STR, &str)) < 0) // 调用ioctl函数
        return (retv); // 返回错误值

    return (str.ic_len); // 返回数据长度
}

#ifdef HAVE_SYS_BUFMOD_H
/*
 * 将流错误消息写入errbuf。
 */
static void
pcap_stream_err(const char *func, int err, char *errbuf)
{
    pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, err, "%s", func); // 格式化错误消息
}
#endif
```