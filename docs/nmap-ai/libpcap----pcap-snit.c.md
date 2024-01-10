# `nmap\libpcap\pcap-snit.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，需要保留以上版权声明和本段文字
 * 在包含二进制代码的发布中，需要在文档或其他提供的材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都需要显示以下声明：
 * “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 为适应新的SunOS4.0 NIT设施而进行的修改，由哥伦比亚大学的Micky Liu在1989年5月完成
 * 该模块现在处理基于STREAMS的NIT
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <sys/dir.h>
#include <sys/fcntlcom.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/stropts.h>

#include <net/if.h>
#include <net/nit.h>
#include <net/nit_if.h>
#include <net/nit_pf.h>
#include <net/nit_buf.h>

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
#include <string.h>
#include <unistd.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * The chunk size for NIT.  This is the amount of buffering
 * done for read calls.
 */
#define CHUNKSIZE (2*1024)

/*
 * The total buffer space used by NIT.
 */
#define BUFSPACE (4*CHUNKSIZE)

/* Forwards */
static int nit_setflags(int, int, int, char *);

/*
 * Private data for capturing on STREAMS NIT devices.
 */
struct pcap_snit {
    struct pcap_stat stat;
};

static int
pcap_stats_snit(pcap_t *p, struct pcap_stat *ps)
{
    struct pcap_snit *psn = p->priv;

    /*
     * "ps_recv" counts packets handed to the filter, not packets
     * that passed the filter.  As filtering is done in userland,
     * this does not include packets dropped because we ran out
     * of buffer space.
     *
     * "ps_drop" counts packets dropped inside the "/dev/nit"
     * device because of flow control requirements or resource
     * exhaustion; it doesn't count packets dropped by the
     * interface driver, or packets dropped upstream.  As filtering
     * is done in userland, it counts packets regardless of whether
     * they would've passed the filter.
     *
     * These statistics don't include packets not yet read from the
     * kernel by libpcap or packets not yet read from libpcap by the
     * application.
     */
    *ps = psn->stat;
    return (0);
}

static int
pcap_read_snit(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    struct pcap_snit *psn = p->priv;
    register int cc, n;
    register u_char *bp, *cp, *ep;
    register struct nit_bufhdr *hdrp;
    register struct nit_iftime *ntp;
    register struct nit_iflen *nlp;
    register struct nit_ifdrops *ndp;
    register int caplen;

    cc = p->cc;
    # 如果当前缓冲区中没有数据
    if (cc == 0) {
        # 从文件描述符中读取数据到缓冲区中
        cc = read(p->fd, (char *)p->buffer, p->bufsize);
        # 如果读取出错
        if (cc < 0) {
            # 如果是非阻塞操作导致的读取阻塞，则返回0
            if (errno == EWOULDBLOCK)
                return (0);
            # 格式化错误消息，返回-1
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "pcap_read");
            return (-1);
        }
        # 将缓冲区的指针指向数据的起始位置
        bp = (u_char *)p->buffer;
    } 
    # 如果当前缓冲区中有数据，则直接使用当前指针
    else
        bp = p->bp;

    '''
     * 循环遍历每个快照中的数据
     *
     * 这假设一个数据包的单个缓冲区将有
     * <= INT_MAX 个数据包，因此数据包计数不会溢出。
     '''
    # 初始化数据包计数
    n = 0;
    # 计算当前快照的结束位置
    ep = bp + cc;
    while (bp < ep) {
        /*
         * 如果调用了 "pcap_breakloop()" 函数？
         * 如果是，立即返回 - 如果我们还没有读取任何数据包，清除标志并返回 -2，表示我们被告知跳出循环，否则保留标志，这样下一次调用将在没有读取任何数据包的情况下跳出循环，并返回到目前为止我们处理的数据包数量。
         */
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (-2);
            } else {
                p->bp = bp;
                p->cc = ep - bp;
                return (n);
            }
        }

        ++psn->stat.ps_recv;
        cp = bp;

        /* 跳过 NIT 缓冲区 */
        hdrp = (struct nit_bufhdr *)cp;
        cp += sizeof(*hdrp);

        /* 跳过 NIT 计时器 */
        ntp = (struct nit_iftime *)cp;
        cp += sizeof(*ntp);

        ndp = (struct nit_ifdrops *)cp;
        psn->stat.ps_drop = ndp->nh_drops;
        cp += sizeof *ndp;

        /* 跳过数据包长度 */
        nlp = (struct nit_iflen *)cp;
        cp += sizeof(*nlp);

        /* 下一个快照 */
        bp += hdrp->nhb_totlen;

        caplen = nlp->nh_pktlen;
        if (caplen > p->snapshot)
            caplen = p->snapshot;

        if (pcap_filter(p->fcode.bf_insns, cp, nlp->nh_pktlen, caplen)) {
            struct pcap_pkthdr h;
            h.ts = ntp->nh_timestamp;
            h.len = nlp->nh_pktlen;
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
}



static int
pcap_inject_snit(pcap_t *p, const void *buf, int size)
{
    struct strbuf ctl, data;

    /*
     * XXX - can we just do
     *
    ret = write(pd->f, buf, size);
     */
    // 设置控制消息的长度为 sa 结构体的大小
    ctl.len = sizeof(*sa);    /* XXX - what was this? */
    // 将 sa 结构体的地址赋给控制消息的缓冲区
    ctl.buf = (char *)sa;
    // 设置数据消息的缓冲区为传入的 buf，长度为传入的 size
    data.buf = buf;
    data.len = size;
    // 发送消息
    ret = putmsg(p->fd, &ctl, &data);
    // 如果发送失败，设置错误消息并返回 -1
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
    // 返回发送的字节数
    return (ret);
}

static int
nit_setflags(pcap_t *p)
{
    bpf_u_int32 flags;
    struct strioctl si;
    u_int zero = 0;
    struct timeval timeout;

    if (p->opt.immediate) {
        /*
         * Set the chunk size to zero, so that chunks get sent
         * up immediately.
         */
        // 如果设置了立即模式，将数据块大小设置为零，以便立即发送
        si.ic_cmd = NIOCSCHUNK;
        si.ic_len = sizeof(zero);
        si.ic_dp = (char *)&zero;
        // 执行设置数据块大小的操作
        if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
            // 设置错误消息并返回 -1
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "NIOCSCHUNK");
            return (-1);
        }
    }
    // 设置超时时间为无限
    si.ic_timout = INFTIM;
    // 如果设置了超时时间，将超时时间设置为指定的值
    if (p->opt.timeout != 0) {
        timeout.tv_sec = p->opt.timeout / 1000;
        timeout.tv_usec = (p->opt.timeout * 1000) % 1000000;
        si.ic_cmd = NIOCSTIME;
        si.ic_len = sizeof(timeout);
        si.ic_dp = (char *)&timeout;
        // 执行设置超时时间的操作
        if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
            // 设置错误消息并返回 -1
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "NIOCSTIME");
            return (-1);
        }
    }
    // 设置标志位，包括时间戳、数据包长度、丢弃计数
    flags = NI_TIMESTAMP | NI_LEN | NI_DROPS;
    // 如果设置了混杂模式，设置混杂模式标志位
    if (p->opt.promisc)
        flags |= NI_PROMISC;
    // 执行设置标志位的操作
    si.ic_cmd = NIOCSFLAGS;
    si.ic_len = sizeof(flags);
    si.ic_dp = (char *)&flags;
    // 执行设置标志位的操作
    if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
        // 设置错误消息并返回 -1
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "NIOCSFLAGS");
        return (-1);
    }
    // 设置成功，返回 0
    return (0);
}

static int
pcap_activate_snit(pcap_t *p)
{
    # 用于 ioctl() 的结构体
    struct strioctl si;        
    # 用于接口请求的结构体
    struct ifreq ifr;        
    # 定义块大小为 CHUNKSIZE
    int chunksize = CHUNKSIZE;
    # 文件描述符
    int fd;
    # 设备路径
    static const char dev[] = "/dev/nit";
    # 错误码
    int err;

    # 如果启用了混杂模式，则不支持混杂模式，返回错误码
    if (p->opt.rfmon) {
        return (PCAP_ERROR_RFMON_NOTSUP);
    }

    # 将快照长度为负数（无效）、为0（未指定）或大于最大允许值的值，设置为最大允许值
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    # 如果快照长度小于96，则设置为96
    if (p->snapshot < 96)
        p->snapshot = 96;

    # 首先尝试读写打开（以允许注入方法工作）。如果由于权限问题而失败，则回退到只读。这允许通过文件权限向非root用户授予对pcap功能的特定访问。
    # XXX - 我们应该有一个API，其中有一个标志，控制是打开只读还是读写，以便在打开时指示发送的权限被拒绝（或者如果不支持在问题设备上发送数据包，则无法发送）。
    p->fd = fd = open(dev, O_RDWR);
    if (fd < 0 && errno == EACCES)
        p->fd = fd = open(dev, O_RDONLY);
    # 如果文件描述符小于0
    if (fd < 0) {
        # 如果错误号是EACCES
        if (errno == EACCES) {
            # 设置错误码为权限被拒绝
            err = PCAP_ERROR_PERM_DENIED;
            # 格式化错误消息，指示尝试打开设备失败，并可能需要root权限
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open %s failed with EACCES - root privileges may be required",
                dev);
        } else {
            # 设置错误码为一般错误
            err = PCAP_ERROR;
            # 格式化错误消息，根据错误号生成错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "%s", dev);
        }
        # 跳转到错误处理
        goto bad;
    }

    # 设置从流中获取离散消息并使用NIT_BUF
    if (ioctl(fd, I_SRDOPT, (char *)RMSGD) < 0) {
        # 格式化错误消息，根据错误号生成错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "I_SRDOPT");
        # 设置错误码为一般错误
        err = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }
    # 推送nbuf
    if (ioctl(fd, I_PUSH, "nbuf") < 0) {
        # 格式化错误消息，根据错误号生成错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "push nbuf");
        # 设置错误码为一般错误
        err = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }
    # 设置块大小
    si.ic_cmd = NIOCSCHUNK;
    si.ic_timout = INFTIM;
    si.ic_len = sizeof(chunksize);
    si.ic_dp = (char *)&chunksize;
    if (ioctl(fd, I_STR, (char *)&si) < 0) {
        # 格式化错误消息，根据错误号生成错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "NIOCSCHUNK");
        # 设置错误码为一般错误
        err = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }

    # 请求接口
    strncpy(ifr.ifr_name, p->opt.device, sizeof(ifr.ifr_name));
    ifr.ifr_name[sizeof(ifr.ifr_name) - 1] = '\0';
    si.ic_cmd = NIOCBIND;
    si.ic_len = sizeof(ifr);
    si.ic_dp = (char *)&ifr;
    if (ioctl(fd, I_STR, (char *)&si) < 0) {
        # 格式化错误消息，根据错误号生成错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "NIOCBIND: %s", ifr.ifr_name);
        # 设置错误码为一般错误
        err = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }

    # 设置快照长度
    si.ic_cmd = NIOCSSNAP;
    si.ic_len = sizeof(p->snapshot);
    si.ic_dp = (char *)&p->snapshot;
    # 如果调用ioctl函数失败，则格式化错误消息并返回错误
    if (ioctl(fd, I_STR, (char *)&si) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "NIOCSSNAP");
        err = PCAP_ERROR;
        goto bad;
    }
    # 如果nit_setflags函数返回值小于0，则返回错误
    if (nit_setflags(p) < 0) {
        err = PCAP_ERROR;
        goto bad;
    }

    # 调用ioctl函数，刷新输入和输出队列
    (void)ioctl(fd, I_FLUSH, (char *)FLUSHR);
    /*
     * NIT仅支持以太网。
     */
    p->linktype = DLT_EN10MB;

    # 设置缓冲区大小为BUFSPACE，并分配内存
    p->bufsize = BUFSPACE;
    p->buffer = malloc(p->bufsize);
    # 如果分配内存失败，则格式化错误消息并返回错误
    if (p->buffer == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        err = PCAP_ERROR;
        goto bad;
    }

    /*
     * "p->fd"是一个STREAMS设备的FD，因此"select()"和"poll()"应该在其上工作。
     */
    p->selectable_fd = p->fd;

    /*
     * 这是（大概）一个真正的以太网捕获；给它一个链路层类型列表，包括DLT_EN10MB和DLT_DOCSIS，以便应用程序可以选择它，以防你捕获了由Cisco Cable Modem Termination System放在以太网上的DOCSIS流量（它不会在电线上放置以太网头，而是在低级以太网框架内放置原始的DOCSIS帧）。
     */
    p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
    /*
     * 如果失败，就让列表为空。
     */
    if (p->dlt_list != NULL) {
        p->dlt_list[0] = DLT_EN10MB;
        p->dlt_list[1] = DLT_DOCSIS;
        p->dlt_count = 2;
    }

    # 设置读操作为pcap_read_snit
    p->read_op = pcap_read_snit;
    # 设置注入操作为pcap_inject_snit
    p->inject_op = pcap_inject_snit;
    # 设置过滤器操作为install_bpf_program（无内核过滤）
    p->setfilter_op = install_bpf_program;
    # 设置方向操作为NULL（未实现）
    p->setdirection_op = NULL;
    # 设置数据链路类型操作为NULL（无法更改数据链路类型）
    p->set_datalink_op = NULL;
    # 设置获取非阻塞操作为pcap_getnonblock_fd
    p->getnonblock_op = pcap_getnonblock_fd;
    # 设置设置非阻塞操作为pcap_setnonblock_fd
    p->setnonblock_op = pcap_setnonblock_fd;
    # 设置统计操作为pcap_stats_snit
    p->stats_op = pcap_stats_snit;

    # 返回0
    return (0);
 bad:
    # 清理资源并返回错误
    pcap_cleanup_live_common(p);
    return (err);
}

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    pcap_t *p;

    // 使用通用的创建函数创建一个 pcap_t 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_snit);
    if (p == NULL)
        return (NULL);

    // 设置激活操作为 pcap_activate_snit
    p->activate_op = pcap_activate_snit;
    return (p);
}

/*
 * XXX - there's probably a NIOCBIND error that means "that device
 * doesn't support NIT"; if so, we should try an NIOCBIND and use that.
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
    // 返回 0，表示没有任何操作
    return (0);
}

// 查找所有的网络接口
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