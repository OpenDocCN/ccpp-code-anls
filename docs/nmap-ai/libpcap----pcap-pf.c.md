# `nmap\libpcap\pcap-pf.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下重新分发和使用，无论是否修改
 * 在源代码分发中，保留以上版权声明和本段文字
 * 在包含二进制代码的分发中，包括以上版权声明和本段文字在文档或其他提供的材料中
 * 所有提及此软件特性或使用的广告材料，需显示以下声明：
 * “本产品包括由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <sys/socket.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <net/pfilt.h>

struct mbuf;
struct rtentry;
#include <net/if.h>

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
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
/*
 * Make "pcap.h" not include "pcap/bpf.h"; we are going to include the native OS version, as we need various BPF ioctls from it.
 */
#define PCAP_DONT_INCLUDE_PCAP_BPF_H
#include <net/bpf.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * FDDI packets are padded to make everything line up on a nice boundary.
 */
#define       PCAP_FDDIPAD 3

/*
 * Private data for capturing on Ultrix and DEC OSF/1^WDigital UNIX^W^W Tru64 UNIX packetfilter devices.
 */
struct pcap_pf {
    int    filtering_in_kernel; /* using kernel filter */
    u_long    TotPkts;    /* can't oflow for 79 hrs on ether */
    u_long    TotAccepted;    /* count accepted by filter */
    u_long    TotDrops;    /* count of dropped packets */
    long    TotMissed;    /* missed by i/f during this run */
    long    OrigMissed;    /* missed by i/f before this run */
};

static int pcap_setfilter_pf(pcap_t *, struct bpf_program *);

/*
 * BUFSPACE is the size in bytes of the packet read buffer.  Most tcpdump applications aren't going to need more than 200 bytes of packet header and the read shouldn't return more packets than packetfilter's internal queue limit (bounded at 256).
 */
#define BUFSPACE (200 * 256)

static int
pcap_read_pf(pcap_t *pc, int cnt, pcap_handler callback, u_char *user)
{
    struct pcap_pf *pf = pc->priv;
    register u_char *p, *bp;
    register int cc, n, buflen, inc;
    register struct enstamp *sp;
    struct enstamp stamp;
    register u_int pad;

 again:
    cc = pc->cc;
    # 如果 cc 等于 0，则表示需要读取数据
    if (cc == 0) {
        # 从文件描述符中读取数据到缓冲区中
        cc = read(pc->fd, (char *)pc->buffer + pc->offset, pc->bufsize);
        # 如果读取失败
        if (cc < 0) {
            # 如果是因为非阻塞操作而导致的读取失败，则返回 0
            if (errno == EWOULDBLOCK)
                return (0);
            # 如果是因为文件偏移量超出范围导致的读取失败
            if (errno == EINVAL &&
                lseek(pc->fd, 0L, SEEK_CUR) + pc->bufsize < 0) {
                '''
                 * 由于内核 bug，在读取 2^31 字节后，
                 * 内核文件偏移量会溢出，导致 read 函数返回 EINVAL 错误。
                 * 通过 lseek() 函数将文件偏移量设置为 0 可以解决这个问题。
                 '''
                (void)lseek(pc->fd, 0L, SEEK_SET);
                # 重新尝试读取数据
                goto again;
            }
            # 格式化错误消息并返回 -1
            pcap_fmt_errmsg_for_errno(pc->errbuf,
                sizeof(pc->errbuf), errno, "pf read");
            return (-1);
        }
        # 设置数据指针指向缓冲区中的数据
        bp = (u_char *)pc->buffer + pc->offset;
    } else
        # 如果 cc 不等于 0，则表示已经有数据可读，直接使用已有数据指针
        bp = pc->bp;
    '''
     * 循环处理每个数据包。
     *
     * 这里假设单个数据包缓冲区中的数据包数量不会超过 INT_MAX，以避免数据包计数溢出。
     '''
    n = 0;
    # 设置填充值为 pc->fddipad
    pad = pc->fddipad;
    # 重置 cc 为 0
    pc->cc = 0;
    # 返回数据包数量
    return (n);
# 用于向网络接口发送数据包
static int
pcap_inject_pf(pcap_t *p, const void *buf, int size)
{
    int ret;

    # 调用系统调用 write() 向文件描述符 p->fd 写入数据包
    ret = write(p->fd, buf, size);
    # 如果返回值为 -1，表示出错，将错误信息格式化到 p->errbuf 中
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
    # 返回写入的字节数
    return (ret);
}

# 获取网络接口的统计信息
static int
pcap_stats_pf(pcap_t *p, struct pcap_stat *ps)
{
    # 获取 pcap_t 结构体中的私有数据结构 pcap_pf
    struct pcap_pf *pf = p->priv;

    """
    根据是否在内核中进行数据包过滤，设置不同的统计信息：
    - 如果在内核中进行数据包过滤，ps_recv 只计算通过过滤器的数据包数量，ps_drop 计算因输入队列满而被丢弃的数据包数量
    - 如果不在内核中进行数据包过滤，ps_recv 只计算通过过滤器的数据包数量，ps_drop 计算因输入队列满而被丢弃的数据包数量
    """
    ps->ps_recv = pf->TotAccepted;
    ps->ps_drop = pf->TotDrops;
    # 将pf->TotMissed减去pf->OrigMissed的结果赋值给ps->ps_ifdrop
    ps->ps_ifdrop = pf->TotMissed - pf->OrigMissed;
    # 返回0，表示函数执行成功
    return (0);
static int
pcap_activate_pf(pcap_t *p)
{
    // 获取 pcap_t 结构体中的私有数据结构 pcap_pf
    struct pcap_pf *pf = p->priv;
    // 定义 enmode 变量
    short enmode;
    // backlog 初始化为 -1，表示请求最大值
    int backlog = -1;
    // 定义过滤器 Filter
    struct enfilter Filter;
    // 定义设备参数 devparams
    struct endevp devparams;
    // 定义错误码变量 err
    int err;

    /*
     * 首先尝试读写打开设备（以允许注入数据包）。如果由于权限问题而失败，则回退到只读模式。
     * 这允许通过文件权限向非root用户授予对 pcap 功能的特定访问权限。
     *
     * XXX - 我们应该有一个 API，其中有一个标志，控制是打开只读还是读写，
     * 以便在打开时指示拒绝发送权限（或者如果不支持在该设备上发送数据包，则无法发送）。
     *
     * XXX - 我们在这里假设 "pfopen()" 实际上不会修改其参数，即使它以 "char *" 而不是 "const char *" 作为其第一个参数。
     * 至少在 Digital UNIX 4.0 上似乎是这样。
     *
     * XXX - 是否有一种错误意味着 "没有这样的设备"？是否有一种意味着 "该设备不支持 pf"？
     */
    // 尝试以读写模式打开设备
    p->fd = pfopen(p->opt.device, O_RDWR);
    // 如果以读写模式打开失败且错误码为 EACCES（权限不足），则以只读模式打开
    if (p->fd == -1 && errno == EACCES)
        p->fd = pfopen(p->opt.device, O_RDONLY);
    // 如果打开设备失败
    if (p->fd < 0) {
        // 如果错误码为 EACCES（权限不足）
        if (errno == EACCES) {
            // 格式化错误消息
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "pf open: %s: Permission denied\n"
"your system may not be properly configured; see the packetfilter(4) man page",
                p->opt.device);
            // 设置错误码为权限被拒绝
            err = PCAP_ERROR_PERM_DENIED;
        } else {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "pf open: %s", p->opt.device);
            // 设置错误码为一般错误
            err = PCAP_ERROR;
        }
        // 跳转到错误处理标签
        goto bad;
    }
    /*
     * 将负的快照值（无效值）、快照值为0（未指定值）或大于正常最大值的值，转换为最大允许值。
     *
     * 如果某些应用程序真的 *需要* 更大的快照长度，我们应该只需增加 MAXIMUM_SNAPLEN。
     */
    如果快照值小于等于0或快照值大于MAXIMUM_SNAPLEN
        将快照值设置为MAXIMUM_SNAPLEN

    pf->OrigMissed = -1;
    enmode = ENTSTAMP|ENNONEXCL;
    如果不是立即模式
        enmode |= ENBATCH;
    如果是混杂模式
        enmode |= ENPROMISC;
    如果ioctl调用失败
        设置错误消息并跳转到bad
#ifdef    ENCOPYALL
    /* 尝试设置COPYALL模式，以便我们可以看到发送给自己的数据包 */
    enmode = ENCOPYALL;
    (void)ioctl(p->fd, EIOCMBIS, (caddr_t)&enmode);/* 如果失败也没关系 */
#endif
    /* 设置backlog */
    if (ioctl(p->fd, EIOCSETW, (caddr_t)&backlog) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "EIOCSETW");
        err = PCAP_ERROR;
        goto bad;
    }
    /* 发现接口类型 */
    if (ioctl(p->fd, EIOCDEVP, (caddr_t)&devparams) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "EIOCDEVP");
        err = PCAP_ERROR;
        goto bad;
    }
    /* HACK: 为了在Ultrix 4.2之前编译 */
#ifndef    ENDT_FDDI
#define    ENDT_FDDI    4
#endif
    switch (devparams.end_dev_type) {

    case ENDT_10MB:
        p->linktype = DLT_EN10MB;
        p->offset = 2;
        /*
         * 这是（大概）一个真正的以太网捕获；给它一个链路层类型列表，包括DLT_EN10MB和DLT_DOCSIS，
         * 这样应用程序可以选择它，以防你捕获了DOCSIS流量，而Cisco Cable Modem Termination System
         * 将其放在以太网上（它不会在以太网上放置以太网头，而是在低级以太网封装中放置原始的DOCSIS帧）。
         */
        p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
        /*
         * 如果失败，就让列表保持空白。
         */
        if (p->dlt_list != NULL) {
            p->dlt_list[0] = DLT_EN10MB;
            p->dlt_list[1] = DLT_DOCSIS;
            p->dlt_count = 2;
        }
        break;

    case ENDT_FDDI:
        p->linktype = DLT_FDDI;
        break;

#ifdef ENDT_SLIP
    case ENDT_SLIP:
        p->linktype = DLT_SLIP;
        break;
#endif

#ifdef ENDT_PPP
    case ENDT_PPP:
        p->linktype = DLT_PPP;
        break;
#endif

#ifdef ENDT_LOOPBACK
    # 如果是回环结束帧
    case ENDT_LOOPBACK:
        # 设置链路类型为以太网帧
        p->linktype = DLT_EN10MB;
        # 设置偏移量为2
        p->offset = 2;
        break;
#ifdef ENDT_TRN
    // 如果数据链路类型是 ENDT_TRN
    case ENDT_TRN:
        // 设置链路类型为 DLT_IEEE802
        p->linktype = DLT_IEEE802;
        // 跳出 switch 语句
        break;
#endif

    default:
        /*
         * XXX - what about ENDT_IEEE802?  The pfilt.h header file calls this "IEEE 802 networks (non-Ethernet)",
         * but that doesn't specify a specific link layer type;
         * it could be 802.4, or 802.5 (except that 802.5 is ENDT_TRN), or 802.6, or 802.11, or....  That's why
         * DLT_IEEE802 was hijacked to mean Token Ring in various BSDs, and why we went along with that hijacking.
         *
         * XXX - what about ENDT_HDLC and ENDT_NULL?
         * Presumably, as ENDT_OTHER is just "Miscellaneous framing", there's not much we can do, as that
         * doesn't specify a particular type of header.
         */
        // 如果是其他未知的数据链路类型
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "unknown data-link type %u", devparams.end_dev_type);
        // 设置错误标志
        err = PCAP_ERROR;
        // 跳转到错误处理部分
        goto bad;
    }
    /* set truncation */
    // 如果链路类型是 DLT_FDDI
    if (p->linktype == DLT_FDDI) {
        // 设置 FDDI 填充长度
        p->fddipad = PCAP_FDDIPAD;

        /* packetfilter includes the padding in the snapshot */
        // 快照长度加上 FDDI 填充长度
        p->snapshot += PCAP_FDDIPAD;
    } else
        // 否则 FDDI 填充长度为 0
        p->fddipad = 0;
    // 设置截断
    if (ioctl(p->fd, EIOCTRUNCATE, (caddr_t)&p->snapshot) < 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE, errno, "EIOCTRUNCATE");
        // 设置错误标志
        err = PCAP_ERROR;
        // 跳转到错误处理部分
        goto bad;
    }
    /* accept all packets */
    // 清空过滤器
    memset(&Filter, 0, sizeof(Filter));
    // 设置过滤器优先级
    Filter.enf_Priority = 37;    /* anything > 2 */
    // 设置过滤器长度
    Filter.enf_FilterLen = 0;    /* means "always true" */
    // 设置过滤器
    if (ioctl(p->fd, EIOCSETF, (caddr_t)&Filter) < 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE, errno, "EIOCSETF");
        // 设置错误标志
        err = PCAP_ERROR;
        // 跳转到错误处理部分
        goto bad;
    }
    # 如果设置了超时时间
    if (p->opt.timeout != 0) {
        # 创建 timeval 结构体，设置秒和微秒
        struct timeval timeout;
        timeout.tv_sec = p->opt.timeout / 1000;
        timeout.tv_usec = (p->opt.timeout * 1000) % 1000000;
        # 设置超时时间
        if (ioctl(p->fd, EIOCSRTIMEOUT, (caddr_t)&timeout) < 0) {
            # 格式化错误消息并返回错误码
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "EIOCSRTIMEOUT");
            err = PCAP_ERROR;
            # 跳转到错误处理
            goto bad;
        }
    }

    # 设置缓冲区大小并分配内存
    p->bufsize = BUFSPACE;
    p->buffer = malloc(p->bufsize + p->offset);
    # 检查内存分配是否成功
    if (p->buffer == NULL) {
        # 格式化错误消息并返回错误码
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        err = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }

    # 设置可选择的文件描述符为当前文件描述符
    p->selectable_fd = p->fd;

    # 设置读取操作为 pcap_read_pf
    p->read_op = pcap_read_pf;
    # 设置注入操作为 pcap_inject_pf
    p->inject_op = pcap_inject_pf;
    # 设置过滤器操作为 pcap_setfilter_pf
    p->setfilter_op = pcap_setfilter_pf;
    # 设置方向操作为 NULL，未实现
    p->setdirection_op = NULL;    /* Not implemented. */
    # 设置数据链路类型操作为 NULL，无法更改数据链路类型
    p->set_datalink_op = NULL;    /* can't change data link type */
    # 设置获取非阻塞操作为 pcap_getnonblock_fd
    p->getnonblock_op = pcap_getnonblock_fd;
    # 设置设置非阻塞操作为 pcap_setnonblock_fd
    p->setnonblock_op = pcap_setnonblock_fd;
    # 设置统计操作为 pcap_stats_pf
    p->stats_op = pcap_stats_pf;

    # 返回 0 表示成功
    return (0);
 bad:
    # 清理资源并返回错误码
    pcap_cleanup_live_common(p);
    return (err);
}

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    // 创建一个 pcap_t 结构体指针
    pcap_t *p;

    // 调用 PCAP_CREATE_COMMON 宏创建一个 pcap_t 结构体指针
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_pf);
    // 如果创建失败，返回 NULL
    if (p == NULL)
        return (NULL);

    // 设置 pcap_t 结构体的 activate_op 指向 pcap_activate_pf 函数
    p->activate_op = pcap_activate_pf;
    // 返回创建的 pcap_t 结构体指针
    return (p);
}

/*
 * XXX - is there an error from pfopen() that means "no such device"?
 * Is there one that means "that device doesn't support pf"?
 */
static int
can_be_bound(const char *name _U_)
{
    // 返回 1，表示设备可以绑定
    return (1);
}

static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
    /*
     * Nothing we can do other than mark loopback devices as "the
     * connected/disconnected status doesn't apply".
     *
     * XXX - is there a way to find out whether an adapter has
     * something plugged into it?
     */
    // 如果是回环设备，设置连接状态不适用
    if (*flags & PCAP_IF_LOOPBACK) {
        /*
         * Loopback devices aren't wireless, and "connected"/
         * "disconnected" doesn't apply to them.
         */
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        return (0);
    }
    return (0);
}

int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    // 调用 pcap_findalldevs_interfaces 函数查找设备
    return (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
        get_if_flags));
}

static int
pcap_setfilter_pf(pcap_t *p, struct bpf_program *fp)
{
    // 获取 pcap_t 结构体的私有数据
    struct pcap_pf *pf = p->priv;
    // 定义一个 bpf_version 结构体
    struct bpf_version bv;

    /*
     * See if BIOCVERSION works.  If not, we assume the kernel doesn't
     * support BPF-style filters (it's not documented in the bpf(7)
     * or packetfiler(7) man pages, but the code used to fail if
     * BIOCSETF worked but BIOCVERSION didn't, and I've seen it do
     * kernel filtering in DU 4.0, so presumably BIOCVERSION works
     * there, at least).
     */
    }

    /*
     * We couldn't do filtering in the kernel; do it in userland.
     */
    // 如果在内核中无法进行过滤，则在用户空间中进行过滤
    if (install_bpf_program(p, fp) < 0)
        return (-1);

    /*
     * XXX - this message should be supplied by the application as
     * a warning of some sort.
     */
}
    # 在标准错误流中输出提示信息
    fprintf(stderr, "tcpdump: Filtering in user process\n");
    # 将过滤器设置为在用户进程中进行过滤
    pf->filtering_in_kernel = 0;
    # 返回0，表示成功
    return (0);
// 返回 libpcap 版本字符串
const char *
pcap_lib_version(void)
{
    // 返回 libpcap 版本字符串常量
    return (PCAP_VERSION_STRING);
}
```