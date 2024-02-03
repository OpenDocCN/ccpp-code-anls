# `nmap\libpcap\pcap-snoop.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，需要保留以上版权声明和本段文字
 * 在包含二进制代码的发布中，需要在文档或其他提供的材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都需要显示以下声明：
 * “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/time.h>

#include <net/raw.h>
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
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * 用于捕获 snoop 设备的私有数据
 */
struct pcap_snoop {
    struct pcap_stat stat;
};

static int
// 从 Snoop 格式的数据包中读取数据
pcap_read_snoop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_snoop *psn = p->priv;
    int cc;
    register struct snoopheader *sh;
    register u_int datalen;
    register u_int caplen;
    register u_char *cp;

again:
    /*
     * "pcap_breakloop()" 被调用了吗？
     */
    if (p->break_loop) {
        /*
         * 是的 - 清除指示它已经被调用的标志，并返回 -2 表示我们被告知跳出循环。
         */
        p->break_loop = 0;
        return (-2);
    }
    // 从文件描述符中读取数据到缓冲区
    cc = read(p->fd, (char *)p->buffer, p->bufsize);
    if (cc < 0) {
        /* 当我们被 ptraced 时不要阻塞 */
        switch (errno) {

        case EINTR:
            goto again;

        case EWOULDBLOCK:
            return (0);            /* XXX */
        }
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "read");
        return (-1);
    }
    // 获取 Snoop 数据包头部
    sh = (struct snoopheader *)p->buffer;
    datalen = sh->snoop_packetlen;

    /*
     * XXX - 唉，snoop_packetlen 是一个 16 位的值。如果我们得到了一个短长度，但读取了一个完整大小的 snoop 数据包，
     * 假设我们溢出了并添加回 64K...
     */
    if (cc == (p->snapshot + sizeof(struct snoopheader)) &&
        (datalen < p->snapshot))
        datalen += (64 * 1024);

    // 获取数据包的捕获长度
    caplen = (datalen < p->snapshot) ? datalen : p->snapshot;
    cp = (u_char *)(sh + 1) + p->offset;        /* XXX */

    /*
     * XXX 不幸的是，snoop 回环不完全像 BSD 的回环。地址族编码在前 2 个字节中，而不是前 4 个字节！
     * 幸运的是，snoop 回环的最后两个字节被清零。
     */
    if (p->linktype == DLT_NULL && *((short *)(cp + 2)) == 0) {
        u_int *uip = (u_int *)cp;
        *uip >>= 16;
    }
}
    # 如果过滤器指令为空，或者数据包通过了过滤器检查
    if (p->fcode.bf_insns == NULL ||
        pcap_filter(p->fcode.bf_insns, cp, datalen, caplen)) {
        # 创建一个 pcap 数据包头结构体
        struct pcap_pkthdr h;
        # 接收数据包计数加一
        ++psn->stat.ps_recv;
        # 设置数据包头的时间戳为捕获时间戳
        h.ts.tv_sec = sh->snoop_timestamp.tv_sec;
        h.ts.tv_usec = sh->snoop_timestamp.tv_usec;
        # 设置数据包头的长度和捕获长度
        h.len = datalen;
        h.caplen = caplen;
        # 调用回调函数处理数据包
        (*callback)(user, &h, cp);
        # 返回 1，表示数据包通过了过滤器
        return (1);
    }
    # 返回 0，表示数据包未通过过滤器
    return (0);
}

static int
pcap_inject_snoop(pcap_t *p, const void *buf, int size)
{
    int ret;

    /*
     * XXX - libnet overwrites the source address with what I
     * presume is the interface's address; is that required?
     */
    // 使用 write 函数将数据包写入到文件描述符对应的套接字中
    ret = write(p->fd, buf, size);
    // 如果写入失败，设置错误信息并返回 -1
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
    // 返回写入的字节数
    return (ret);
}

static int
pcap_stats_snoop(pcap_t *p, struct pcap_stat *ps)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_snoop *psn = p->priv;
    register struct rawstats *rs;
    struct rawstats rawstats;

    // 将 rawstats 结构体清零
    rs = &rawstats;
    memset(rs, 0, sizeof(*rs));
    // 获取 Snoop 接口的统计信息
    if (ioctl(p->fd, SIOCRAWSTATS, (char *)rs) < 0) {
        // 如果获取失败，设置错误信息并返回 -1
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "SIOCRAWSTATS");
        return (-1);
    }

    /*
     * "ifdrops" are those dropped by the network interface
     * due to resource shortages or hardware errors.
     *
     * "sbdrops" are those dropped due to socket buffer limits.
     *
     * As filter is done in userland, "sbdrops" counts packets
     * regardless of whether they would've passed the filter.
     *
     * XXX - does this count *all* Snoop or Drain sockets,
     * rather than just this socket?  If not, why does it have
     * both Snoop and Drain statistics?
     */
    // 计算丢弃的数据包数量
    psn->stat.ps_drop =
        rs->rs_snoop.ss_ifdrops + rs->rs_snoop.ss_sbdrops +
        rs->rs_drain.ds_ifdrops + rs->rs_drain.ds_sbdrops;

    /*
     * "ps_recv" counts only packets that passed the filter.
     * As filtering is done in userland, this does not include
     * packets dropped because we ran out of buffer space.
     */
    // 设置接收的数据包数量
    *ps = psn->stat;
    return (0);
}

/* XXX can't disable promiscuous */
static int
pcap_activate_snoop(pcap_t *p)
{
    int fd;
    struct sockaddr_raw sr;
    struct snoopfilter sf;
    u_int v;
    int ll_hdrlen;
    int snooplen;
    struct ifreq ifr;

    // 创建原始套接字
    fd = socket(PF_RAW, SOCK_RAW, RAWPROTO_SNOOP);
    # 如果文件描述符小于0
    if (fd < 0) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "snoop socket");
        # 跳转到错误处理标签
        goto bad;
    }
    # 将文件描述符存储到结构体中
    p->fd = fd;
    # 将结构体 sr 清零
    memset(&sr, 0, sizeof(sr));
    # 设置 sr 的协议族为 AF_RAW
    sr.sr_family = AF_RAW;
    # 将设备名称拷贝到 sr 的接口名称中
    (void)strncpy(sr.sr_ifname, p->opt.device, sizeof(sr.sr_ifname));
    # 如果绑定失败
    if (bind(fd, (struct sockaddr *)&sr, sizeof(sr))) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "snoop bind");
        # 跳转到错误处理标签
        goto bad;
    }
    # 将结构体 sf 清零
    memset(&sf, 0, sizeof(sf));
    # 如果 ioctl 调用失败
    if (ioctl(fd, SIOCADDSNOOP, &sf) < 0) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCADDSNOOP");
        # 跳转到错误处理标签
        goto bad;
    }
    # 如果选项中的缓冲区大小不为0，则将其赋值给 v，否则默认为64K
    if (p->opt.buffer_size != 0)
        v = p->opt.buffer_size;
    else
        v = 64 * 1024;    /* default to 64K buffer size */
    # 设置套接字选项，接收缓冲区大小为 v
    (void)setsockopt(fd, SOL_SOCKET, SO_RCVBUF, (char *)&v, sizeof(v));
    # 如果设备名称以 "ipg", "rns", "xpi" 开头
    } else if (strncmp("ipg", p->opt.device, 3) == 0 ||
           strncmp("rns", p->opt.device, 3) == 0 ||    /* O2/200/2000 FDDI */
           strncmp("xpi", p->opt.device, 3) == 0) {
        # 设置链路类型为 DLT_FDDI
        p->linktype = DLT_FDDI;
        # 设置偏移量为3
        p->offset = 3;                /* XXX yeah? */
        # 设置链路头长度为13
        ll_hdrlen = 13;
    # 如果设备名称以 "ppp" 开头
    } else if (strncmp("ppp", p->opt.device, 3) == 0) {
        # 设置链路类型为 DLT_RAW
        p->linktype = DLT_RAW;
        # 设置链路头长度为0
        ll_hdrlen = 0;    /* DLT_RAW meaning "no PPP header, just the IP packet"? */
    # 如果设备名称以 "qfa" 开头
    } else if (strncmp("qfa", p->opt.device, 3) == 0) {
        # 设置链路类型为 DLT_IP_OVER_FC
        p->linktype = DLT_IP_OVER_FC;
        # 设置链路头长度为24
        ll_hdrlen = 24;
    } else if (strncmp("pl", p->opt.device, 2) == 0) {
        // 如果设备名称以 "pl" 开头，则设置链路类型为 DLT_RAW
        p->linktype = DLT_RAW;
        // Cray UNICOS/mp 伪链路，头部长度为 0
        ll_hdrlen = 0;    /* Cray UNICOS/mp pseudo link */
    } else if (strncmp("lo", p->opt.device, 2) == 0) {
        // 如果设备名称以 "lo" 开头，则设置链路类型为 DLT_NULL
        p->linktype = DLT_NULL;
        // 头部长度为 4
        ll_hdrlen = 4;
    } else {
        // 如果设备名称不匹配以上条件，则设置错误信息并跳转到 bad 标签处
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "snoop: unknown physical layer type");
        goto bad;
    }

    if (p->opt.rfmon) {
        /*
         * 在 Irix 上不支持监控模式（Irix 上不支持 Wi-Fi 设备）。
         */
        return (PCAP_ERROR_RFMON_NOTSUP);
    }

    /*
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为最大允许的值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     */
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        // 如果快照值小于等于 0 或大于最大快照长度，则将快照值设置为最大快照长度
        p->snapshot = MAXIMUM_SNAPLEN;
#ifdef SIOCGIFMTU
    /*
     * XXX - IRIX appears to give you an error if you try to set the
     * capture length to be greater than the MTU, so let's try to get the MTU first and, if that succeeds, trim the snap length
     * to be no greater than the MTU.
     */
    // 尝试获取网络接口的最大传输单元（MTU），如果成功，则将捕获长度修剪为不大于MTU
    (void)strncpy(ifr.ifr_name, p->opt.device, sizeof(ifr.ifr_name));
    // 使用ioctl函数获取网络接口的MTU
    if (ioctl(fd, SIOCGIFMTU, (char *)&ifr) < 0) {
        // 如果获取失败，则设置错误消息并跳转到错误处理
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGIFMTU");
        goto bad;
    }
    /*
     * OK, we got it.
     *
     * XXX - some versions of IRIX 6.5 define "ifr_mtu" and have an
     * "ifru_metric" member of the "ifr_ifru" union in an "ifreq"
     * structure, others don't.
     *
     * I've no idea what's going on, so, if "ifr_mtu" isn't defined,
     * we define it as "ifr_metric", as using that field appears to
     * work on the versions that lack "ifr_mtu" (and, on those that
     * don't lack it, "ifru_metric" and "ifru_mtu" are both "int"
     * members of the "ifr_ifru" union, which suggests that they
     * may be interchangeable in this case).
     */
    // 如果系统不定义ifr_mtu，则将其定义为ifr_metric
#ifndef ifr_mtu
#define ifr_mtu    ifr_metric
#endif
    // 如果捕获长度大于MTU加上链路层头部长度，则将捕获长度设置为MTU加上链路层头部长度
    if (p->snapshot > ifr.ifr_mtu + ll_hdrlen)
        p->snapshot = ifr.ifr_mtu + ll_hdrlen;
#endif

    /*
     * The argument to SIOCSNOOPLEN is the number of link-layer
     * payload bytes to capture - it doesn't count link-layer
     * header bytes.
     */
    // 计算需要捕获的链路层有效载荷字节数
    snooplen = p->snapshot - ll_hdrlen;
    if (snooplen < 0)
        snooplen = 0;
    // 设置捕获的链路层有效载荷字节数
    if (ioctl(fd, SIOCSNOOPLEN, &snooplen) < 0) {
        // 如果设置失败，则设置错误消息并跳转到错误处理
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCSNOOPLEN");
        goto bad;
    }
    // 启用网络接口的数据包捕获
    v = 1;
    if (ioctl(fd, SIOCSNOOPING, &v) < 0) {
        // 如果设置失败，则设置错误消息并跳转到错误处理
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCSNOOPING");
        goto bad;
    }

    // 设置数据包缓冲区大小为4096
    p->bufsize = 4096;                /* XXX */
    // 分配数据包缓冲区
    p->buffer = malloc(p->bufsize);
    # 如果缓冲区为空，则使用错误号和错误消息格式化错误消息到错误缓冲区，然后跳转到标签bad
    if (p->buffer == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        goto bad;
    }

    """
     * "p->fd" is a socket, so "select()" should work on it.
     * p->fd是一个套接字，所以应该可以在其上使用"select()"
     """
    # 将p->fd赋值给selectable_fd，表示p->fd是可选择的文件描述符
    p->selectable_fd = p->fd;

    # 设置读操作为pcap_read_snoop
    p->read_op = pcap_read_snoop;
    # 设置注入操作为pcap_inject_snoop
    p->inject_op = pcap_inject_snoop;
    # 设置过滤器操作为install_bpf_program，表示没有内核过滤
    p->setfilter_op = install_bpf_program;
    # 设置方向操作为NULL，表示未实现
    p->setdirection_op = NULL;
    # 设置数据链路类型操作为NULL，表示无法更改数据链路类型
    p->set_datalink_op = NULL;
    # 获取非阻塞操作为pcap_getnonblock_fd
    p->getnonblock_op = pcap_getnonblock_fd;
    # 设置非阻塞操作为pcap_setnonblock_fd
    p->setnonblock_op = pcap_setnonblock_fd;
    # 设置统计操作为pcap_stats_snoop
    p->stats_op = pcap_stats_snoop;

    # 返回0表示成功
    return (0);
 bad:
    # 清理现场并返回错误
    pcap_cleanup_live_common(p);
    return (PCAP_ERROR);
}

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    // 创建一个 pcap_t 结构体指针
    pcap_t *p;

    // 调用 PCAP_CREATE_COMMON 宏创建一个 pcap_t 结构体指针，并传入错误缓冲区和 pcap_snoop 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_snoop);
    // 如果创建失败，则返回空指针
    if (p == NULL)
        return (NULL);

    // 设置 pcap_t 结构体的 activate_op 指针为 pcap_activate_snoop 函数
    p->activate_op = pcap_activate_snoop;
    // 返回创建的 pcap_t 结构体指针
    return (p);
}

/*
 * XXX - there's probably a particular bind error that means "that device
 * doesn't support snoop"; if so, we should try a bind and use that.
 */
static int
can_be_bound(const char *name _U_)
{
    // 返回 1，表示设备可以被绑定
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

// 查找并返回系统中的网络接口列表
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    // 调用 pcap_findalldevs_interfaces 函数来获取网络接口列表
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