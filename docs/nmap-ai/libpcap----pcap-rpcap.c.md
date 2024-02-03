# `nmap\libpcap\pcap-rpcap.c`

```cpp
/*
 * 版权声明，版权所有
 * 1. 源代码的再分发和使用需要保留版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发需要在文档和/或其他提供的材料中重现版权声明、条件列表和以下免责声明
 * 3. 不得使用 Politecnico di Torino、CACE Technologies 或其贡献者的名称来认可或推广基于此软件的产品，除非事先得到特定的书面许可
 * 免责声明：本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。在任何情况下，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他），都不承担任何直接、间接、附带、特殊、惩罚性或后果性的损害赔偿责任，即使已被告知可能发生此类损害。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "ftmacros.h"
#include "diag-control.h"

#include <string.h>        /* 用于 strlen() 等函数 */
#include <stdlib.h>        /* 用于 malloc()、free() 等函数 */
#include <stdarg.h>        /* 用于具有可变参数的函数 */
#include <errno.h>        /* for the errno variable */
#include <limits.h>        /* for INT_MAX */
#include "sockutils.h"
#include "pcap-int.h"
#include "pcap-util.h"
#include "rpcap-protocol.h"
#include "pcap-rpcap.h"

#ifdef _WIN32
#include "charconv.h"        /* for utf_8_to_acp_truncated() */
#endif

#ifdef HAVE_OPENSSL
#include "sslutils.h"
#endif

/*
 * This file contains the pcap module for capturing from a remote machine's
 * interfaces using the RPCAP protocol.
 *
 * WARNING: All the RPCAP functions that are allowed to return a buffer
 * containing the error description can return max PCAP_ERRBUF_SIZE characters.
 * However there is no guarantees that the string will be zero-terminated.
 * Best practice is to define the errbuf variable as a char of size
 * 'PCAP_ERRBUF_SIZE+1' and to insert manually a NULL character at the end
 * of the buffer. This will guarantee that no buffer overflows occur even
 * if we use the printf() to show the error on the screen.
 *
 * XXX - actually, null-terminating the error string is part of the
 * contract for the pcap API; if there's any place in the pcap code
 * that doesn't guarantee null-termination, even at the expense of
 * cutting the message short, that's a bug and needs to be fixed.
 */

#define PCAP_STATS_STANDARD    0    /* Used by pcap_stats_rpcap to see if we want standard or extended statistics */
#ifdef _WIN32
#define PCAP_STATS_EX        1    /* Used by pcap_stats_rpcap to see if we want standard or extended statistics */
#endif
/*
 * \brief 保持活动模式中所有打开连接的列表。
 *
 * 此结构定义了一个链表，其中包含管理活动模式所需的信息。
 * 换句话说，当活动模式中启动新连接时，将更新此结构，以便反映当前打开的活动模式连接列表。
 * findalldevs() 和 open_remote() 需要此结构，以查看它们是否需要向主机打开新的控制连接，或者它们已经建立了控制连接。
 */
struct activehosts
{
    struct sockaddr_storage host;  // 存储主机地址信息
    SOCKET sockctrl;  // 控制连接的套接字
    SSL *ssl;  // SSL 连接
    uint8 protocol_version;  // 协议版本
    int byte_swapped;  // 字节顺序是否被交换
    struct activehosts *next;  // 下一个活动连接的指针
};

/* 保持活动模式中所有打开连接的列表。 */
static struct activehosts *activeHosts;  // 活动连接的链表指针

/*
 * 当我们想要接受新的远程连接（仅限活动模式）时，保持主套接字标识符。
 * 有关更多详细信息，请参阅 pcap_remoteact_accept() 和 pcap_remoteact_cleanup() 的文档。
 */
static SOCKET sockmain;  // 主套接字标识符
static SSL *ssl_main;  // 主 SSL 连接

/*
 * 使用 rpcap 协议进行远程捕获的私有数据。
 */
struct pcap_rpcap {
    /*
     * 如果我们是网络客户端，则为 '1'；多个函数（如 pcap_setfilter()）需要此信息，
     * 以确定它们是使用套接字还是打开本地适配器。
     */
    int rmt_clientside;  // 是否是网络客户端

    SOCKET rmt_sockctrl;  // 用于控制连接的套接字 ID
    SOCKET rmt_sockdata;  // 用于数据连接的套接字 ID
    SSL *ctrl_ssl, *data_ssl;  // 通过 TLS 传输 rmt_sockctrl 和 rmt_sockdata 的可选传输
    int rmt_flags;  // 需要保存标志，因为它们由 pcap_open_live() 传递，但由 pcap_startcapture() 使用
};
    # 标记捕获是否已经开始（需要知道是否需要调用pcap_startcapture()）
    int rmt_capstarted;
    # 指向一个在运行时分配的缓冲区的指针，用于存储当前过滤器。在打开标志PCAP_OPENFLAG_NOCAPTURE_RPCAP时需要
    char *currentfilter;

    # 协商的协议版本
    uint8 protocol_version;
    # 用户是否要求使用rpcaps方案
    uint8 uses_ssl;
    # 服务器的字节顺序是否与我们的相反
    int byte_swapped;

    # 保留网络丢弃的数据包数量
    unsigned int TotNetDrops;

    # 保留应用程序接收的数据包数量
    # 内核缓冲区丢弃的数据包不计入此变量
    # 除了远程捕获的情况，我们还有在传输中的数据包，即已由远程主机传输但尚未从客户端接收的数据包
    # 在这种情况下，（TotAccepted - TotDrops - TotNetDrops）会给出错误的结果，因为这个数字并不总是对应于应用程序接收的数据包数量
    # 因此，在远程捕获中，我们需要另一个变量，它考虑了应用程序实际接收的数据包数量
    unsigned int TotCapt;

    # pcap统计信息
    struct pcap_stat stat;
    # 下一个打开的pcap列表，在关闭时需要清除一些内容
    struct pcap *next;
// 本地定义的函数
static struct pcap_stat *rpcap_stats_rpcap(pcap_t *p, struct pcap_stat *ps, int mode);
static int pcap_pack_bpffilter(pcap_t *fp, char *sendbuf, int *sendbufidx, struct bpf_program *prog);
static int pcap_createfilter_norpcappkt(pcap_t *fp, struct bpf_program *prog);
static int pcap_updatefilter_remote(pcap_t *fp, struct bpf_program *prog);
static void pcap_save_current_filter_rpcap(pcap_t *fp, const char *filter);
static int pcap_setfilter_rpcap(pcap_t *fp, struct bpf_program *prog);
static int pcap_setsampling_remote(pcap_t *fp);
static int pcap_startcapture_remote(pcap_t *fp);
static int rpcap_recv_msg_header(SOCKET sock, SSL *, struct rpcap_header *header, char *errbuf);
static int rpcap_check_msg_ver(SOCKET sock, SSL *, uint8 expected_ver, struct rpcap_header *header, char *errbuf);
static int rpcap_check_msg_type(SOCKET sock, SSL *, uint8 request_type, struct rpcap_header *header, uint16 *errcode, char *errbuf);
static int rpcap_process_msg_header(SOCKET sock, SSL *, uint8 ver, uint8 request_type, struct rpcap_header *header, char *errbuf);
static int rpcap_recv(SOCKET sock, SSL *, void *buffer, size_t toread, uint32 *plen, char *errbuf);
static void rpcap_msg_err(SOCKET sockctrl, SSL *, uint32 plen, char *remote_errbuf);
static int rpcap_discard(SOCKET sock, SSL *, uint32 len, char *errbuf);
static int rpcap_read_packet_msg(struct pcap_rpcap const *, pcap_t *p, size_t size);
/*
 * 可能的 IPv4 地址族值，除了指定的在网络上传输的值之外，
 * 这个值是 2（因为大家都使用 2 来表示 AF_INET4）。
 */
#define SOCKADDR_IN_LEN        16    /* struct sockaddr_in 的长度 */
#define SOCKADDR_IN6_LEN    28    /* struct sockaddr_in6 的长度 */
#define NEW_BSD_AF_INET_BE    ((SOCKADDR_IN_LEN << 8) | 2)    /* 新的 BSD AF_INET 大端格式 */
#define NEW_BSD_AF_INET_LE    (SOCKADDR_IN_LEN << 8)    /* 新的 BSD AF_INET 小端格式 */

/*
 * 可能的 IPv6 地址族值，除了指定的在网络上传输的值之外，
 * 这个值是 23（因为 Windows 使用这个值，而大多数 RPCAP 服务器可能在运行 Windows，
 * 因为 WinPcap 包含了服务器，但很少有 UN*X 构建和发布它）。
 *
 * 新的 BSD sockaddr 结构格式在 4.4-Lite 之前就已经存在了，所以所有的自由软件 BSD 都使用它。
 */
#define NEW_BSD_AF_INET6_BSD_BE        ((SOCKADDR_IN6_LEN << 8) | 24)    /* NetBSD, OpenBSD, BSD/OS */
#define NEW_BSD_AF_INET6_FREEBSD_BE    ((SOCKADDR_IN6_LEN << 8) | 28)    /* FreeBSD, DragonFly BSD */
#define NEW_BSD_AF_INET6_DARWIN_BE    ((SOCKADDR_IN6_LEN << 8) | 30)    /* macOS, iOS, 基于 Darwin 的任何其他系统 */
#define NEW_BSD_AF_INET6_LE        (SOCKADDR_IN6_LEN << 8)    /* 新的 BSD AF_INET6 小端格式 */
#define LINUX_AF_INET6            10    /* Linux AF_INET6 值 */
#define HPUX_AF_INET6            22    /* HPUX AF_INET6 值 */
#define AIX_AF_INET6            24    /* AIX AF_INET6 值 */
#define SOLARIS_AF_INET6        26    /* Solaris AF_INET6 值 */

static int
rpcap_deseraddr(struct rpcap_sockaddr *sockaddrin, struct sockaddr_storage **sockaddrout, char *errbuf)
{
    /* 警告：我们只支持 AF_INET 和 AF_INET6 */
    switch (ntohs(sockaddrin->family))
    {
    case RPCAP_AF_INET:
    case NEW_BSD_AF_INET_BE:
    # 如果是新的 BSD AF_INET_LE 情况
    case NEW_BSD_AF_INET_LE:
        {
        # 定义 IPv4 地址结构体指针
        struct rpcap_sockaddr_in *sockaddrin_ipv4;
        # 定义 IPv4 地址结构体指针
        struct sockaddr_in *sockaddrout_ipv4;

        # 为输出地址结构体分配内存
        (*sockaddrout) = (struct sockaddr_storage *) malloc(sizeof(struct sockaddr_in));
        # 如果内存分配失败
        if ((*sockaddrout) == NULL)
        {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc() failed");
            # 返回错误
            return -1;
        }
        # 强制类型转换为 IPv4 地址结构体指针
        sockaddrin_ipv4 = (struct rpcap_sockaddr_in *) sockaddrin;
        # 强制类型转换为 IPv4 地址结构体指针
        sockaddrout_ipv4 = (struct sockaddr_in *) (*sockaddrout);
        # 设置地址族为 AF_INET
        sockaddrout_ipv4->sin_family = AF_INET;
        # 设置端口号，转换为网络字节序
        sockaddrout_ipv4->sin_port = ntohs(sockaddrin_ipv4->port);
        # 复制地址
        memcpy(&sockaddrout_ipv4->sin_addr, &sockaddrin_ipv4->addr, sizeof(sockaddrout_ipv4->sin_addr));
        # 清空填充字节
        memset(sockaddrout_ipv4->sin_zero, 0, sizeof(sockaddrout_ipv4->sin_zero));
        # 跳出 switch 语句块
        break;
        }
#ifdef AF_INET6
    case RPCAP_AF_INET6:
    case NEW_BSD_AF_INET6_BSD_BE:
    case NEW_BSD_AF_INET6_FREEBSD_BE:
    case NEW_BSD_AF_INET6_DARWIN_BE:
    case NEW_BSD_AF_INET6_LE:
    case LINUX_AF_INET6:
    case HPUX_AF_INET6:
    case AIX_AF_INET6:
    case SOLARIS_AF_INET6:
        {
        // 定义 IPv6 地址结构体指针
        struct rpcap_sockaddr_in6 *sockaddrin_ipv6;
        struct sockaddr_in6 *sockaddrout_ipv6;

        // 为输出地址结构体分配内存
        (*sockaddrout) = (struct sockaddr_storage *) malloc(sizeof(struct sockaddr_in6));
        // 检查内存分配是否成功
        if ((*sockaddrout) == NULL)
        {
            // 如果内存分配失败，设置错误消息并返回 -1
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno, "malloc() failed");
            return -1;
        }
        // 将输入地址结构体转换为 IPv6 地址结构体
        sockaddrin_ipv6 = (struct rpcap_sockaddr_in6 *) sockaddrin;
        sockaddrout_ipv6 = (struct sockaddr_in6 *) (*sockaddrout);
        // 设置输出地址结构体的各个字段
        sockaddrout_ipv6->sin6_family = AF_INET6;
        sockaddrout_ipv6->sin6_port = ntohs(sockaddrin_ipv6->port);
        sockaddrout_ipv6->sin6_flowinfo = ntohl(sockaddrin_ipv6->flowinfo);
        memcpy(&sockaddrout_ipv6->sin6_addr, &sockaddrin_ipv6->addr, sizeof(sockaddrout_ipv6->sin6_addr));
        sockaddrout_ipv6->sin6_scope_id = ntohl(sockaddrin_ipv6->scope_id);
        // 结束当前 case 分支
        break;
        }
#endif

    default:
        /*
         * It is neither AF_INET nor AF_INET6 (or, if the OS doesn't
         * support AF_INET6, it's not AF_INET).
         */
        // 如果既不是 AF_INET 也不是 AF_INET6，则将输出地址结构体置空
        *sockaddrout = NULL;
        // 结束当前 case 分支
        break;
    }
    // 返回 0，表示成功
    return 0;
}
/*
 * 这个函数从网络套接字中读取数据包。它不会将数据包传递给 pcap_dispatch()/pcap_loop() 回调函数（因此在函数名中带有“nocb”字符串）。
 *
 * 这个函数被 pcap_read_rpcap() 调用。
 *
 * 警告：出于选择，这个函数不使用信号量。一个更聪明的实现应该在数据线程中放置一个信号量，并且一旦套接字缓冲区中有数据，就会发出信号。
 * 然而，这很复杂，并且在从网络读取数据时并没有带来任何优势，因为网络延迟可能比这些优化更重要。因此，我们选择了以下方法：
 * - 用户选择的“超时”被分成两部分（一半在服务器端，具有通常的含义，一半在客户端）
 * - 这个函数检查数据包；如果没有数据包，它会等待超时的一半，然后再次检查。如果数据包仍然缺失，它就返回，否则它就读取数据包。
 */
static int pcap_read_nocb_remote(pcap_t *p, struct pcap_pkthdr *pkt_header, u_char **pkt_data)
{
    struct pcap_rpcap *pr = p->priv;    /* 在进行远程实时捕获时使用的结构体 */
    struct rpcap_header *header;        /* 符合 RPCAP 格式的通用头部 */
    struct rpcap_pkthdr *net_pkt_header;    /* 来自消息的数据包头部 */
    u_char *net_pkt_data;            /* 来自消息的数据包数据 */
    uint32 plen;
    int retval = 0;                /* 通用返回值 */
    int msglen;

    /* 用于 select() 调用的结构 */
    struct timeval tv;            /* select() 可以阻塞等待数据的最长时间 */
    fd_set rfds;                /* 我们需要检查的套接字描述符集合 */

    /*
     * 定义数据包缓冲区超时时间，用于在 select() 中使用
     * 'timeout' 在 pcap_t 中以毫秒为单位；我们需要将其转换为秒和微秒
     */
    tv.tv_sec = p->opt.timeout / 1000;
    # 将毫秒转换为微秒，并赋值给tv.tv_usec
    tv.tv_usec = (suseconds_t)((p->opt.timeout - tv.tv_sec * 1000) * 1000);
#ifdef HAVE_OPENSSL
    /* 检查最后一个解码的 TLS 记录中是否仍有可用字节。
     * 如果是这样，我们知道 SSL_read 不会阻塞。 */
    retval = pr->data_ssl && SSL_pending(pr->data_ssl) > 0;
#endif
    if (! retval)
    {
        /* 观察 sockdata 是否有输入 */
        FD_ZERO(&rfds);

        /*
         * 在调用 select() 之前，'fp->rmt_sockdata' 必须始终设置，
         * 因为它会被 select() 清除
         */
        FD_SET(pr->rmt_sockdata, &rfds);

#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
        retval = 1;
#else
        retval = select((int) pr->rmt_sockdata + 1, &rfds, NULL, NULL, &tv);
#endif

        if (retval == -1)
        {
#ifndef _WIN32
            if (errno == EINTR)
            {
                /* 被中断。 */
                return 0;
            }
#endif
            sock_geterrmsg(p->errbuf, PCAP_ERRBUF_SIZE,
                "select() failed");
            return -1;
        }
    }

    /* 没有等待的数据，所以返回 '0' */
    if (retval == 0)
        return 0;

    /*
     * 我们必须将 'header' 定义为一个指向更大缓冲区的指针，
     * 因为在 UDP 的情况下，我们必须在单个调用中读取所有消息
     */
    header = (struct rpcap_header *) p->buffer;
    net_pkt_header = (struct rpcap_pkthdr *) ((char *)p->buffer + sizeof(struct rpcap_header));
    net_pkt_data = (u_char *)p->buffer + sizeof(struct rpcap_header) + sizeof(struct rpcap_pkthdr);

    if (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP)
    {
        /* 从网络中读取整个消息 */
        msglen = sock_recv_dgram(pr->rmt_sockdata, pr->data_ssl, p->buffer,
            p->bufsize, p->errbuf, PCAP_ERRBUF_SIZE);
        if (msglen == -1)
        {
            /* 网络错误。 */
            return -1;
        }
        if (msglen == -3)
        {
            /* 接收被中断。 */
            return 0;
        }
        if ((size_t)msglen < sizeof(struct rpcap_header))
        {
            /*
             * 消息长度小于一个 rpcap 头部。
             */
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "UDP 数据包消息长度小于一个 rpcap 头部");
            return -1;
        }
        plen = ntohl(header->plen);
        if ((size_t)msglen < sizeof(struct rpcap_header) + plen)
        {
            /*
             * 消息长度小于头部声明的长度。
             */
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "UDP 数据包消息长度小于其 rpcap 头部声明的长度");
            return -1;
        }
    }
    else
    {
        // 定义整型变量status
        int status;

        // 如果读取的数据长度小于包头的长度
        if ((size_t)p->cc < sizeof(struct rpcap_header))
        {
            /*
             * 我们还没有读取任何数据包头。
             * 我们应该获取的大小是数据包头的大小。
             */
            // 读取数据包消息
            status = rpcap_read_packet_msg(pr, p, sizeof(struct rpcap_header));
            // 如果返回-1，表示网络错误
            if (status == -1)
            {
                /* 网络错误。 */
                return -1;
            }
            // 如果返回-3，表示接收被中断
            if (status == -3)
            {
                /* 接收被中断。 */
                return 0;
            }
        }

        /*
         * 我们已经有了包头，所以我们知道消息负载的长度。
         * 我们应该获取的大小是数据包头加上负载的大小。
         */
        // 获取消息负载的大小
        plen = ntohl(header->plen);
        // 如果消息负载的大小大于缓冲区大小减去包头的大小
        if (plen > p->bufsize - sizeof(struct rpcap_header))
        {
            /*
             * 这比我们期望的最大记录还要大。
             * (我们通过减法来避免溢出。)
             */
            // 将错误信息写入错误缓冲区
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Server sent us a message larger than the largest expected packet message");
            return -1;
        }
        // 读取数据包消息
        status = rpcap_read_packet_msg(pr, p, sizeof(struct rpcap_header) + plen);
        // 如果返回-1，表示网络错误
        if (status == -1)
        {
            /* 网络错误。 */
            return -1;
        }
        // 如果返回-3，表示接收被中断
        if (status == -3)
        {
            /* 接收被中断。 */
            return 0;
        }

        /*
         * 我们已经有了整个消息；重置缓冲区指针和计数，
         * 因为下一次读取应该开始一个新的消息。
         */
        // 重置缓冲区指针和计数
        p->bp = p->buffer;
        p->cc = 0;
    }

    /*
     * 我们已经有了整个消息。
     */
    // 设置包头中的消息负载大小
    header->plen = plen;

    /*
     * 服务器是否指定了我们协商的版本？
     */
    # 检查远程数据包捕获协议的版本是否匹配，如果不匹配则返回0表示未接收到数据包
    if (rpcap_check_msg_ver(pr->rmt_sockdata, pr->data_ssl, pr->protocol_version,
        header, p->errbuf) == -1)
    {
        return 0;    /* 返回'未接收到数据包' */
    }

    /*
     * 这是一个 RPCAP_MSG_PACKET 消息吗？
     */
    if (header->type != RPCAP_MSG_PACKET)
    {
        return 0;    /* 返回'未接收到数据包' */
    }

    # 如果捕获的数据包长度大于指定的长度，则返回错误信息
    if (ntohl(net_pkt_header->caplen) > plen)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Packet's captured data goes past the end of the received packet message.");
        return -1;
    }

    /* 填充数据包头部信息 */
    pkt_header->caplen = ntohl(net_pkt_header->caplen);
    pkt_header->len = ntohl(net_pkt_header->len);
    pkt_header->ts.tv_sec = ntohl(net_pkt_header->timestamp_sec);
    pkt_header->ts.tv_usec = ntohl(net_pkt_header->timestamp_usec);

    /* 提供数据包数据的起始指针 */
    *pkt_data = net_pkt_data;

    /*
     * 我不更新网络丢包计数器，因为我们使用TCP，因此没有数据包丢失。
     * 只需正确更新接收到的数据包数量
     */
    pr->TotCapt++;

    if (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP)
    {
        unsigned int npkt;

        /* 我们使用UDP，因此需要更新网络丢包计数器 */
        npkt = ntohl(net_pkt_header->npkt);

        if (pr->TotCapt != npkt)
        {
            pr->TotNetDrops += (npkt - pr->TotCapt);
            pr->TotCapt = npkt;
        }
    }

    /* 数据包成功读取 */
    return 1;
}
/*
 * 从网络套接字读取数据包
 *
 * 此函数依赖于 pcap_read_nocb_remote 来传递数据包。不同之处在于，一旦读取到数据包，就通过回调函数将其传递给应用程序。
 */
static int pcap_read_rpcap(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 远程捕获时使用的结构体
    struct pcap_rpcap *pr = p->priv;
    // 数据包头
    struct pcap_pkthdr pkt_header;
    // 数据包内容
    u_char *pkt_data;
    // 数据包计数
    int n = 0;
    // 返回值
    int ret;

    /*
     * 如果这是客户端，并且我们还没有开始捕获，现在开始捕获。
     */
    if (pr->rmt_clientside)
    {
        /* 我们在远程捕获 */
        if (!pr->rmt_capstarted)
        {
            /*
             * 捕获还没有开始，尝试开始捕获。
             */
            if (pcap_startcapture_remote(p))
                return -1;
        }
    }

    /*
     * 这可能处理超过 INT_MAX 个数据包，这将导致数据包计数溢出，要么看起来像是负数，从而导致我们返回一个看起来像是错误的值，要么溢出回到正数领域，从而导致我们返回一个太低的计数。
     *
     * 因此，如果数据包计数是无限的，我们将其剪切到 INT_MAX；这个例程不应该无限处理数据包，所以这不是一个问题。
     */
    if (PACKET_COUNT_IS_UNLIMITED(cnt))
        cnt = INT_MAX;

    while (n < cnt || PACKET_COUNT_IS_UNLIMITED(cnt))
    {
        /*
         * 检查是否调用了 "pcap_breakloop()"
         */
        if (p->break_loop) {
            /*
             * 是 - 清除指示已调用的标志，并返回 PCAP_ERROR_BREAK 以指示我们被告知跳出循环。
             */
            p->break_loop = 0;
            return (PCAP_ERROR_BREAK);
        }

        /*
         * 读取一些数据包。
         */
        ret = pcap_read_nocb_remote(p, &pkt_header, &pkt_data);
        if (ret == 1)
        {
            /*
             * 我们收到一个数据包。
             *
             * 进行任何必要的后处理，将其传递给回调函数，并计数以便返回计数。
             */
            pcap_post_process(p->linktype, pr->byte_swapped,
                &pkt_header, pkt_data);
            (*callback)(user, &pkt_header, pkt_data);
            n++;
        }
        else if (ret == -1)
        {
            /* 出错。 */
            return ret;
        }
        else
        {
            /*
             * 没有数据包；这可能意味着超时，或者被中断，或者收到了一个坏数据包。
             *
             * 我们被告知跳出循环了吗？
             */
            if (p->break_loop) {
                /*
                 * 是。
                 */
                p->break_loop = 0;
                return (PCAP_ERROR_BREAK);
            }
            /* 否 - 返回我们处理的数据包数量。 */
            return n;
        }
    }
    return n;
}
/*
 * 如果我们处于被动模式，则此函数向捕获服务器发送CLOSE命令；如果我们处于主动模式，则向捕获服务器发送ENDCAP命令。
 * 当用户调用pcap_close()时调用此函数。它向我们的对等方发送一个命令，表示'好的，让我们停止捕获'。
 * 警告：由于我们正在关闭连接，我们不检查错误。
 */
static void pcap_cleanup_rpcap(pcap_t *fp)
{
    struct pcap_rpcap *pr = fp->priv;    /* 在进行远程实时捕获时使用的结构体 */
    struct rpcap_header header;        /* RPCAP数据包的头部 */
    struct activehosts *temp;        /* 扫描主机列表链以检测是否处于主动模式所需的临时变量 */
    int active = 0;                /* 是否处于主动模式？ */

    /* 检测是否处于主动模式 */
    temp = activeHosts;
    while (temp)
    {
        if (temp->sockctrl == pr->rmt_sockctrl)
        {
            active = 1;
            break;
        }
        temp = temp->next;
    }

    if (!active)
    {
        rpcap_createhdr(&header, pr->protocol_version,
            RPCAP_MSG_CLOSE, 0, 0);

        /*
         * 发送关闭请求；不报告任何错误，因为我们正在关闭此pcap_t，并且没有地方报告错误。不会对此消息发送回复。
         */
        (void)sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, (char *)&header,
            sizeof(struct rpcap_header), NULL, 0);
    }
    else
    {
        // 创建一个RPCAP头部，用于结束抓包请求
        rpcap_createhdr(&header, pr->protocol_version,
            RPCAP_MSG_ENDCAP_REQ, 0, 0);

        /*
         * 发送结束抓包请求；不报告任何错误，因为我们正在关闭这个pcap_t，没有地方报告错误。
         */
        if (sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, (char *)&header,
            sizeof(struct rpcap_header), NULL, 0) == 0)
        {
            /*
             * 等待答复；不报告任何错误，因为我们正在关闭这个pcap_t，没有地方报告错误。
             */
            if (rpcap_process_msg_header(pr->rmt_sockctrl, pr->ctrl_ssl,
                pr->protocol_version, RPCAP_MSG_ENDCAP_REQ,
                &header, NULL) == 0)
            {
                // 丢弃数据包
                (void)rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl,
                    header.plen, NULL);
            }
        }
    }

    // 如果远程数据套接字存在
    if (pr->rmt_sockdata)
    {
#ifdef HAVE_OPENSSL
        if (pr->data_ssl)
        {
            // 如果使用了 OpenSSL，完成对数据套接字的 SSL 处理
            // 必须在套接字关闭之前完成
            ssl_finish(pr->data_ssl);
            pr->data_ssl = NULL;
        }
#endif
        // 关闭远程数据套接字
        sock_close(pr->rmt_sockdata, NULL, 0);
        pr->rmt_sockdata = 0;
    }

    // 如果不是主动模式，并且存在远程控制套接字
    if ((!active) && (pr->rmt_sockctrl))
    {
#ifdef HAVE_OPENSSL
        if (pr->ctrl_ssl)
        {
            // 如果使用了 OpenSSL，完成对控制套接字的 SSL 处理
            // 必须在套接字关闭之前完成
            ssl_finish(pr->ctrl_ssl);
            pr->ctrl_ssl = NULL;
        }
#endif
        // 关闭远程控制套接字
        sock_close(pr->rmt_sockctrl, NULL, 0);
    }

    // 重置远程控制套接字和控制 SSL
    pr->rmt_sockctrl = 0;
    pr->ctrl_ssl = NULL;

    // 如果存在当前过滤器，释放内存
    if (pr->currentfilter)
    {
        free(pr->currentfilter);
        pr->currentfilter = NULL;
    }

    // 清理 pcap_live_common
    pcap_cleanup_live_common(fp);

    /* 为了避免 sock_init() 的数量不一致 */
    // 清理套接字
    sock_cleanup();
}

/*
 * 从对等方检索网络统计信息；
 * 仅提供标准统计信息。
 */
static int pcap_stats_rpcap(pcap_t *p, struct pcap_stat *ps)
{
    struct pcap_stat *retval;

    // 通过 rpcap_stats_rpcap() 检索统计信息
    retval = rpcap_stats_rpcap(p, ps, PCAP_STATS_STANDARD);

    // 如果成功检索到统计信息，返回 0
    if (retval)
        return 0;
    // 否则返回 -1
    else
        return -1;
}

#ifdef _WIN32
/*
 * 从对等方检索网络统计信息；
 * 提供 pcap_stats_ex() 支持的额外统计信息。
 */
static struct pcap_stat *pcap_stats_ex_rpcap(pcap_t *p, int *pcap_stat_size)
{
    *pcap_stat_size = sizeof (p->stat);

    /* PCAP_STATS_EX (third param) 表示 '扩展的 pcap_stats()' */
    // 返回通过 rpcap_stats_rpcap() 检索到的统计信息
    return (rpcap_stats_rpcap(p, &(p->stat), PCAP_STATS_EX));
}
#endif
/*
 * This function retrieves network statistics from our peer.  It
 * is used by the two previous functions.
 *
 * It can be called in two modes:
 * - PCAP_STATS_STANDARD: if we want just standard statistics (i.e.,
 *   for pcap_stats())
 * - PCAP_STATS_EX: if we want extended statistics (i.e., for
 *   pcap_stats_ex())
 *
 * This 'mode' parameter is needed because in pcap_stats() the variable that
 * keeps the statistics is allocated by the user. On Windows, this structure
 * has been extended in order to keep new stats. However, if the user has a
 * smaller structure and it passes it to pcap_stats(), this function will
 * try to fill in more data than the size of the structure, so that memory
 * after the structure will be overwritten.
 *
 * So, we need to know it we have to copy just the standard fields, or the
 * extended fields as well.
 *
 * In case we want to copy the extended fields as well, the problem of
 * memory overflow no longer exists because the structure that's filled
 * in is part of the pcap_t, so that it can be guaranteed to be large
 * enough for the additional statistics.
 *
 * \param p: the pcap_t structure related to the current instance.
 *
 * \param ps: a pointer to a 'pcap_stat' structure, needed for compatibility
 * with pcap_stat(), where the structure is allocated by the user. In case
 * of pcap_stats_ex(), this structure and the function return value point
 * to the same variable.
 *
 * \param mode: one of PCAP_STATS_STANDARD or PCAP_STATS_EX.
 *
 * \return The structure that keeps the statistics, or NULL in case of error.
 * The error string is placed in the pcap_t structure.
 */
static struct pcap_stat *rpcap_stats_rpcap(pcap_t *p, struct pcap_stat *ps, int mode)
{
    // structure used when doing a remote live capture
    struct pcap_rpcap *pr = p->priv;
    // header of the RPCAP packet
    struct rpcap_header header;
    // statistics sent on the network
    struct rpcap_stats netstats;
    uint32 plen;                /* 定义一个名为plen的32位无符号整数变量，表示消息中剩余的数据长度 */
#ifdef _WIN32
    // 如果操作系统是 Windows，并且统计模式不是标准模式或扩展模式，则返回错误
    if (mode != PCAP_STATS_STANDARD && mode != PCAP_STATS_EX)
#else
    // 如果操作系统不是 Windows，并且统计模式不是标准模式，则返回错误
    if (mode != PCAP_STATS_STANDARD)
#endif
    {
        // 如果统计模式无效，则设置错误信息并返回空指针
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Invalid stats mode %d", mode);
        return NULL;
    }

    /*
     * 如果捕获尚未开始，我们无法从对等方请求捕获的统计信息，因此对于所有统计信息都返回0，因为尚未看到任何内容。
     */
    if (!pr->rmt_capstarted)
    {
        // 如果捕获尚未开始，则将所有统计信息设置为0
        ps->ps_drop = 0;
        ps->ps_ifdrop = 0;
        ps->ps_recv = 0;
#ifdef _WIN32
        // 如果操作系统是 Windows 并且统计模式是扩展模式，则设置额外的统计信息为0
        if (mode == PCAP_STATS_EX)
        {
            ps->ps_capt = 0;
            ps->ps_sent = 0;
            ps->ps_netdrop = 0;
        }
#endif /* _WIN32 */

        return ps;
    }

    // 创建 RPCAP 消息头
    rpcap_createhdr(&header, pr->protocol_version,
        RPCAP_MSG_STATS_REQ, 0, 0);

    /* 发送 PCAP_STATS 命令 */
    if (sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, (char *)&header,
        sizeof(struct rpcap_header), p->errbuf, PCAP_ERRBUF_SIZE) < 0)
        return NULL;        /* 不可恢复的网络错误 */

    /* 接收并处理回复消息头 */
    if (rpcap_process_msg_header(pr->rmt_sockctrl, pr->ctrl_ssl, pr->protocol_version,
        RPCAP_MSG_STATS_REQ, &header, p->errbuf) == -1)
        return NULL;        /* 错误 */

    plen = header.plen;

    /* 读取回复体 */
    if (rpcap_recv(pr->rmt_sockctrl, pr->ctrl_ssl, (char *)&netstats,
        sizeof(struct rpcap_stats), &plen, p->errbuf) == -1)
        goto error;

    // 将网络统计信息转换为主机字节顺序
    ps->ps_drop = ntohl(netstats.krnldrop);
    ps->ps_ifdrop = ntohl(netstats.ifdrop);
    ps->ps_recv = ntohl(netstats.ifrecv);
#ifdef _WIN32
    // 如果操作系统是 Windows 并且统计模式是扩展模式，则设置额外的统计信息
    if (mode == PCAP_STATS_EX)
    {
        ps->ps_capt = pr->TotCapt;
        ps->ps_netdrop = pr->TotNetDrops;
        ps->ps_sent = ntohl(netstats.svrcapt);
    }
#endif /* _WIN32 */

    /* 丢弃消息的其余部分 */
    # 如果调用rpcap_discard函数返回-1，表示出错，跳转到error_nodiscard标签处处理错误
    if (rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, plen, p->errbuf) == -1)
        goto error_nodiscard;

    # 返回ps变量的值
    return ps;
{
    /*
     * 丢弃消息的其余部分。
     * 我们已经报告了一个错误；如果这里出现错误，就继续执行。
     */
    (void)rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, plen, NULL);
    
error_nodiscard:
    return NULL;
}

/*
 * 此函数返回此活动连接的活动主机列表中的条目（仅限活动模式），如果没有活动连接或发生错误，则返回NULL。仅供内部使用。
 *
 * \param host：一个字符串，保存要获取其活动连接的套接字ID的主机名。
 *
 * \param error：一个指向int的指针，如果发生错误则设置为1，否则为0。
 *
 * \param errbuf：一个指向用户分配的缓冲区（大小为PCAP_ERRBUF_SIZE），将包含错误消息（如果有）。
 *
 * \return 如果找到此主机在活动连接列表中的条目，则返回该条目，如果未找到或出现错误，则返回NULL。
 */
static struct activehosts *
rpcap_remoteact_getsock(const char *host, int *error, char *errbuf)
{
    struct activehosts *temp;            /* 用于扫描主机列表链的临时变量 */
    struct addrinfo hints, *addrinfo, *ai_next;    /* 用于将主机名转换为其地址的临时变量 */
    int retval;

    /* 检索与 'host' 对应的网络地址 */
    addrinfo = NULL;
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_family = PF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    retval = sock_initaddress(host, NULL, &hints, &addrinfo, errbuf, PCAP_ERRBUF_SIZE);
    if (retval != 0)
    {
        *error = 1;
        return NULL;
    }

    temp = activeHosts;

    while (temp)
    {
        // 设置下一个地址信息为初始地址信息
        ai_next = addrinfo;
        // 循环遍历地址信息链表
        while (ai_next)
        {
            // 如果当前地址信息与临时地址信息的主机地址相同
            if (sock_cmpaddr(&temp->host, (struct sockaddr_storage *) ai_next->ai_addr) == 0)
            {
                // 设置错误码为 0
                *error = 0;
                // 释放地址信息链表
                freeaddrinfo(addrinfo);
                // 返回临时地址信息
                return temp;
            }
    
            // 设置下一个地址信息为当前地址信息的下一个地址信息
            ai_next = ai_next->ai_next;
        }
        // 设置临时地址信息为下一个地址信息
        temp = temp->next;
    }
    
    // 如果地址信息链表不为空，则释放地址信息链表
    if (addrinfo)
        freeaddrinfo(addrinfo);
    
    /*
     * 要获取套接字 ID 的主机没有活动连接。
     */
    // 设置错误码为 0
    *error = 0;
    // 返回空指针
    return NULL;
    }
# 这个函数启动远程抓包。
# 这个函数是必需的，因为 RPCAP 协议将“打开”和“开始抓包”功能解耦了。
# 这个函数获取所有需要的参数（这些参数已经存储在 pcap_t 结构中），并将它们发送到服务器。
# \param fp: 当前打开设备的 pcap_t 描述符。
# \return 如果一切正常则返回'0'，否则返回'-1'。错误消息（如果有）将返回到 pcap_t 结构的 'errbuf' 字段中。
static int pcap_startcapture_remote(pcap_t *fp)
{
    // 用于进行远程实时抓包时使用的结构
    struct pcap_rpcap *pr = fp->priv;
    // 临时缓冲区，用于缓冲要发送的数据
    char sendbuf[RPCAP_NETBUF_SIZE];
    // 当前缓冲的字节数索引
    int sendbufidx = 0;
    // 用于保存数据连接的网络端口的临时变量
    uint16 portdata = 0;
    // '1' 表示我们处于活动模式
    int active = 0;
    // 用于扫描主机列表链以检测是否处于活动模式所需的临时变量
    struct activehosts *temp;
    // 其他主机的数字名称
    char host[INET6_ADDRSTRLEN + 1];

    /* 与套接字相关的变量 */
    // 临时变量，用于打开套接字连接
    struct addrinfo hints;
    // 临时变量，用于打开套接字连接
    struct addrinfo *addrinfo;
    // 数据连接的套接字描述符
    SOCKET sockdata = 0;
    // 用于检索本地机器上选择的网络数据端口的临时变量
    struct sockaddr_storage saddr;
    // 用于检索本地机器上选择的网络数据端口的临时变量
    socklen_t saddrlen;
    // 控制连接使用的地址族
    int ai_family;
    struct sockaddr_in *sin4;
    struct sockaddr_in6 *sin6;

    /* 与 RPCAP 相关的变量 */
    # 定义RPCAP数据包的头部
    struct rpcap_header header;            /* header of the RPCAP packet */
    # 定义开始抓包请求消息
    struct rpcap_startcapreq *startcapreq;        /* start capture request message */
    # 定义开始抓包回复消息
    struct rpcap_startcapreply startcapreply;    /* start capture reply message */

    # 与缓冲区设置相关的变量
    int res;
    socklen_t itemp;
    int sockbufsize = 0;
    uint32 server_sockbufsize;

    # 在任何错误发生之前，清空pr->data_ssl，因为似乎p->priv在malloc后没有被清零
    # 现在应该已经被清零了，因为它是由pcap_alloc_pcap_t()分配的，它执行了calloc()
    pr->data_ssl = NULL;

    # 检查是否需要抽样，如果需要，则先设置抽样
    if (pcap_setsampling_remote(fp) != 0)
        return -1;

    # 检测是否处于主动模式
    temp = activeHosts;
    while (temp)
    {
        if (temp->sockctrl == pr->rmt_sockctrl)
        {
            active = 1;
            break;
        }
        temp = temp->next;
    }

    # 初始化addrinfo
    addrinfo = NULL;

    # 获取用于控制连接的完整sockaddr结构
    # 这是为了获取控制套接字的地址族
    # 提示：我无法在pcap_t结构中保存ctrl套接字的ai_family，
    # 因为在主动模式下，ctrl套接字可能已经打开；
    # 所以我仍然需要调用getpeername()
    saddrlen = sizeof(struct sockaddr_storage);
    if (getpeername(pr->rmt_sockctrl, (struct sockaddr *) &saddr, &saddrlen) == -1)
    {
        sock_geterrmsg(fp->errbuf, PCAP_ERRBUF_SIZE,
            "getsockname() failed");
        goto error_nodiscard;
    }
    ai_family = ((struct sockaddr_storage *) &saddr)->ss_family;

    # 获取我们连接到的远程主机的数值地址
    if (getnameinfo((struct sockaddr *) &saddr, saddrlen, host,
        sizeof(host), NULL, 0, NI_NUMERICHOST))
    {
        // 使用错误缓冲区和错误消息格式化字符串，获取错误消息并存储在错误缓冲区中
        sock_geterrmsg(fp->errbuf, PCAP_ERRBUF_SIZE, "getnameinfo() failed");
        // 跳转到错误处理标签，不丢弃数据
        goto error_nodiscard;
    }

    /*
     * 如果：
     * - 我们正在使用 TCP，并且用户希望我们处于主动模式
     * - 我们正在使用 UDP
     */
    if ((active) || (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP))
    {
        /*
         * We have to create a new socket to receive packets
         * We have to do that immediately, since we have to tell the other
         * end which network port we picked up
         */
        // 为接收数据包创建新的套接字
        // 我们必须立即这样做，因为我们必须告诉另一端我们选择了哪个网络端口
        memset(&hints, 0, sizeof(struct addrinfo));
        // 在活动状态下，addrinfo为NULL
        hints.ai_family = ai_family;    // 使用与控制套接字相同的地址族
        hints.ai_socktype = (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP) ? SOCK_DGRAM : SOCK_STREAM;
        hints.ai_flags = AI_PASSIVE;    // 数据连接由服务器向客户端打开

        // 让服务器为我们选择一个空闲的网络端口
        if (sock_initaddress(NULL, NULL, &hints, &addrinfo, fp->errbuf, PCAP_ERRBUF_SIZE) == -1)
            goto error_nodiscard;

        if ((sockdata = sock_open(NULL, addrinfo, SOCKOPEN_SERVER,
            1 /* max 1 connection in queue */, fp->errbuf, PCAP_ERRBUF_SIZE)) == INVALID_SOCKET)
            goto error_nodiscard;

        // addrinfo不再使用
        freeaddrinfo(addrinfo);
        addrinfo = NULL;

        // 获取数据连接中使用的完整sockaddr结构
        saddrlen = sizeof(struct sockaddr_storage);
        if (getsockname(sockdata, (struct sockaddr *) &saddr, &saddrlen) == -1)
        {
            sock_geterrmsg(fp->errbuf, PCAP_ERRBUF_SIZE,
                "getsockname() failed");
            goto error_nodiscard;
        }

        switch (saddr.ss_family) {

        case AF_INET:
            sin4 = (struct sockaddr_in *)&saddr;
            portdata = sin4->sin_port;
            break;

        case AF_INET6:
            sin6 = (struct sockaddr_in6 *)&saddr;
            portdata = sin6->sin6_port;
            break;

        default:
            snprintf(fp->errbuf, PCAP_ERRBUF_SIZE,
                "Local address has unknown address family %u",
                saddr.ss_family);
            goto error_nodiscard;
        }
    }
    """
    现在是时候开始使用 RPCAP 协议了
    RPCAP 开始捕获命令：创建请求消息
    """
    # 将 RPCAP 头部的大小添加到发送缓冲区
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        goto error_nodiscard;

    # 创建 RPCAP 头部
    rpcap_createhdr((struct rpcap_header *) sendbuf,
        pr->protocol_version, RPCAP_MSG_STARTCAP_REQ, 0,
        sizeof(struct rpcap_startcapreq) + sizeof(struct rpcap_filter) + fp->fcode.bf_len * sizeof(struct rpcap_filterbpf_insn));

    # 填充远程打开适配器所需的结构
    startcapreq = (struct rpcap_startcapreq *) &sendbuf[sendbufidx];

    # 将 RPCAP 开始捕获请求的大小添加到发送缓冲区
    if (sock_bufferize(NULL, sizeof(struct rpcap_startcapreq), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        goto error_nodiscard;

    # 将 startcapreq 结构清零
    memset(startcapreq, 0, sizeof(struct rpcap_startcapreq));

    # 默认情况下，在一侧应用一半的超时时间，在另一侧应用另一半的超时时间
    fp->opt.timeout = fp->opt.timeout / 2;
    startcapreq->read_timeout = htonl(fp->opt.timeout);

    # 如果是主动模式或者 rmt_flags 包含 PCAP_OPENFLAG_DATATX_UDP，则 portdata 在 openreq 中是有意义的
    if ((active) || (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP))
    {
        startcapreq->portdata = portdata;
    }

    # 设置捕获数据包的长度
    startcapreq->snaplen = htonl(fp->snapshot);
    startcapreq->flags = 0;

    # 如果 rmt_flags 包含 PCAP_OPENFLAG_PROMISCUOUS，则设置 RPCAP_STARTCAPREQ_FLAG_PROMISC 标志
    if (pr->rmt_flags & PCAP_OPENFLAG_PROMISCUOUS)
        startcapreq->flags |= RPCAP_STARTCAPREQ_FLAG_PROMISC;
    # 如果 rmt_flags 包含 PCAP_OPENFLAG_DATATX_UDP，则设置 RPCAP_STARTCAPREQ_FLAG_DGRAM 标志
    if (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP)
        startcapreq->flags |= RPCAP_STARTCAPREQ_FLAG_DGRAM;
    # 如果是主动模式，则设置 RPCAP_STARTCAPREQ_FLAG_SERVEROPEN 标志
    if (active)
        startcapreq->flags |= RPCAP_STARTCAPREQ_FLAG_SERVEROPEN;

    # 将标志字段转换为网络字节顺序
    startcapreq->flags = htons(startcapreq->flags);

    # 打包捕获过滤器
    if (pcap_pack_bpffilter(fp, &sendbuf[sendbufidx], &sendbufidx, &fp->fcode))
        goto error_nodiscard;
    # 如果发送数据失败，则跳转到错误处理代码，不丢弃数据
    if (sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, sendbuf, sendbufidx, fp->errbuf,
        PCAP_ERRBUF_SIZE) < 0)
        goto error_nodiscard;

    # 接收并处理回复消息头部
    if (rpcap_process_msg_header(pr->rmt_sockctrl, pr->ctrl_ssl, pr->protocol_version,
        RPCAP_MSG_STARTCAP_REQ, &header, fp->errbuf) == -1)
        goto error_nodiscard;

    # 获取消息体长度
    plen = header.plen;

    # 接收开始抓包的回复消息
    if (rpcap_recv(pr->rmt_sockctrl, pr->ctrl_ssl, (char *)&startcapreply,
        sizeof(struct rpcap_startcapreply), &plen, fp->errbuf) == -1)
        goto error;

    '''
     * 如果是 UDP 数据流，则连接已经由守护进程打开
     * 因此，此情况已经被上面的代码覆盖了
     * 现在，我们仍然需要处理 TCP 连接，因为：
     * - 如果我们处于主动模式，则必须等待远程连接
     * - 如果我们处于被动模式，则必须开始一个连接
     *
     * 我们必须分两步进行工作，因为如果我们正在打开一个 TCP 连接，我们必须告诉远程端我们正在使用的端口；
     * 如果我们正在接受一个 TCP 连接，我们必须等待远程端发送这个信息。
     '''
    # 如果不是 UDP 数据流
    if (!(pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP))
    {
        // 如果连接不活跃
        if (!active)
        {
            char portstring[PCAP_BUF_SIZE];

            // 将 hints 结构体清零
            memset(&hints, 0, sizeof(struct addrinfo));
            // 使用与控制套接字相同的地址族
            hints.ai_family = ai_family;        
            // 根据远程标志设置套接字类型
            hints.ai_socktype = (pr->rmt_flags & PCAP_OPENFLAG_DATATX_UDP) ? SOCK_DGRAM : SOCK_STREAM;
            // 将端口号转换为字符串
            snprintf(portstring, PCAP_BUF_SIZE, "%d", ntohs(startcapreply.portdata));

            /* 让服务器为我们选择一个空闲的网络端口 */
            if (sock_initaddress(host, portstring, &hints, &addrinfo, fp->errbuf, PCAP_ERRBUF_SIZE) == -1)
                goto error;

            // 打开客户端套接字
            if ((sockdata = sock_open(host, addrinfo, SOCKOPEN_CLIENT, 0, fp->errbuf, PCAP_ERRBUF_SIZE)) == INVALID_SOCKET)
                goto error;

            /* addrinfo 不再使用 */
            freeaddrinfo(addrinfo);
            addrinfo = NULL;
        }
        else
        {
            SOCKET socktemp;    /* 我们需要另一个套接字，因为我们将要接受()一个连接 */

            /* 创建连接 */
            saddrlen = sizeof(struct sockaddr_storage);

            socktemp = accept(sockdata, (struct sockaddr *) &saddr, &saddrlen);

            if (socktemp == INVALID_SOCKET)
            {
                sock_geterrmsg(fp->errbuf, PCAP_ERRBUF_SIZE,
                    "accept() failed");
                goto error;
            }

            /* 现在我接受了连接，服务器套接字不再需要 */
            sock_close(sockdata, fp->errbuf, PCAP_ERRBUF_SIZE);
            sockdata = socktemp;
        }
    }

    /* 让我们保存数据连接的套接字 */
    pr->rmt_sockdata = sockdata;
#ifdef HAVE_OPENSSL
    // 如果使用了 OpenSSL，为数据套接字进行 SSL 升级
    if (pr->uses_ssl)
    {
        pr->data_ssl = ssl_promotion(0, sockdata, fp->errbuf, PCAP_ERRBUF_SIZE);
        if (! pr->data_ssl) goto error;
    }
#endif

    /*
     * 设置数据套接字的套接字缓冲区大小。
     * 它与连接另一侧使用的本地捕获缓冲区大小相同。
     */
    server_sockbufsize = ntohl(startcapreply.bufsize);

    /* 获取套接字缓冲区的实际大小 */
    itemp = sizeof(sockbufsize);

    res = getsockopt(sockdata, SOL_SOCKET, SO_RCVBUF, (char *)&sockbufsize, &itemp);
    if (res == -1)
    {
        // 获取套接字选项失败，设置错误消息并跳转到错误处理
        sock_geterrmsg(fp->errbuf, PCAP_ERRBUF_SIZE,
            "pcap_startcapture_remote(): getsockopt() failed");
        goto error;
    }

    /*
     * 警告：在一些内核上（例如 Linux），用户缓冲区的大小不考虑 pcap_header 等内容，
     * 而是设置为与 snaplen 相等。
     *
     * 在我看来，这是错误的（bufsize 的含义变得有点奇怪）。
     * 因此，这里 bufsize 是用户缓冲区的整个大小。
     * 如果返回的 bufsize 太小，让我们相应地调整它。
     */
    if (server_sockbufsize <= (u_int) fp->snapshot)
        server_sockbufsize += sizeof(struct pcap_pkthdr);

    /* 如果当前套接字缓冲区比期望的要小 */
    if ((u_int) sockbufsize < server_sockbufsize)
    {
        /*
         * 循环直到缓冲区大小合适或原始套接字缓冲区大小大于此大小。
         */
        for (;;)
        {
            // 设置套接字接收缓冲区大小
            res = setsockopt(sockdata, SOL_SOCKET, SO_RCVBUF,
                (char *)&(server_sockbufsize),
                sizeof(server_sockbufsize));

            // 如果设置成功，跳出循环
            if (res == 0)
                break;

            /*
             * 如果出现问题，将缓冲区大小减半
             * （确保不会比当前缓冲区大小更小）。
             */
            server_sockbufsize /= 2;

            if ((u_int) sockbufsize >= server_sockbufsize)
            {
                server_sockbufsize = sockbufsize;
                break;
            }
        }
    }

    /*
     * 让我们分配数据包；这是为了在从套接字中提取数据时将数据包放在某个地方所必需的。
     * 由于套接字缓冲区中已经进行了缓冲，因此我们只需要一个大小等于快照大小的最大可能数据包消息的缓冲区，
     * 即消息头的长度加上数据包头的长度加上快照长度。
     */
    fp->bufsize = sizeof(struct rpcap_header) + sizeof(struct rpcap_pkthdr) + fp->snapshot;

    // 分配缓冲区
    fp->buffer = (u_char *)malloc(fp->bufsize);
    if (fp->buffer == NULL)
    {
        // 如果分配失败，设置错误消息并跳转到错误处理部分
        pcap_fmt_errmsg_for_errno(fp->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        goto error;
    }

    /*
     * 缓冲区当前为空。
     */
    fp->bp = fp->buffer;
    fp->cc = 0;

    /* 丢弃消息的其余部分。 */
    if (rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, plen, fp->errbuf) == -1)
        goto error_nodiscard;
    /*
     * 如果用户不想捕获 RPCAP 数据包，我们需要更新过滤器
     * 我们必须在这里更新它（而不是在发送“StartCapture”消息时将其发送进去）
     * 因为当我们生成“start capture”时，我们还不知道当前正在使用的所有端口
     */
    if (pr->rmt_flags & PCAP_OPENFLAG_NOCAPTURE_RPCAP)
    {
        // 创建 BPF 程序对象
        struct bpf_program fcode;

        // 如果创建过滤器失败，则跳转到错误处理
        if (pcap_createfilter_norpcappkt(fp, &fcode) == -1)
            goto error;

        /* 我们不能使用 'pcap_setfilter_rpcap'，因为严格来说捕获还没有开始 */
        /* （'pr->rmt_capstarted' 变量将在下面的几行中更新） */
        // 更新远程过滤器
        if (pcap_updatefilter_remote(fp, &fcode) == -1)
            goto error;

        // 释放过滤器对象
        pcap_freecode(&fcode);
    }

    // 更新捕获已开始标志
    pr->rmt_capstarted = 1;
    // 返回 0 表示成功
    return 0;
error:
    /*
     * 当连接建立后，我们必须关闭它。因此，在这个函数的开头，如果发生错误，我们立即返回 NULL；当连接建立时，我们必须在这里 ('goto error;') 正确关闭所有内容。
     */

    /*
     * 丢弃消息的剩余部分。
     * 我们已经报告了一个错误；如果这里出现错误，就继续执行。
     */
    (void)rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, plen, NULL);

error_nodiscard:
#ifdef HAVE_OPENSSL
    if (pr->data_ssl)
    {
        // 完成使用数据套接字的 SSL 句柄。
        // 这必须在套接字关闭之前完成。
        ssl_finish(pr->data_ssl);
        pr->data_ssl = NULL;
    }
#endif

    /* 我们可能会在这里，因为 sockdata 说 'error' */
    if ((sockdata != 0) && (sockdata != INVALID_SOCKET))
        sock_close(sockdata, NULL, 0);

    if (!active)
    {
#ifdef HAVE_OPENSSL
        if (pr->ctrl_ssl)
        {
            // 完成使用控制套接字的 SSL 句柄。
            // 这必须在套接字关闭之前完成。
            ssl_finish(pr->ctrl_ssl);
            pr->ctrl_ssl = NULL;
        }
#endif
        sock_close(pr->rmt_sockctrl, NULL, 0);
    }

    if (addrinfo != NULL)
        freeaddrinfo(addrinfo);

    /*
     * 我们不必在这里调用 pcap_close()，因为这个函数总是在发生问题时由用户调用
     */
#if 0
    if (fp)
    {
        pcap_close(fp);
        fp= NULL;
    }
#endif

    return -1;
}
/*
 * This function takes a bpf program and sends it to the other host.
 *
 * This function can be called in two cases:
 * - pcap_startcapture_remote() is called (we have to send the filter
 *   along with the 'start capture' command)
 * - we want to update the filter during a capture (i.e. pcap_setfilter()
 *   after the capture has been started)
 *
 * This function serializes the filter into the sending buffer ('sendbuf',
 * passed as a parameter) and return back. It does not send anything on
 * the network.
 *
 * \param fp: the pcap_t descriptor of the device currently opened.
 *
 * \param sendbuf: the buffer on which the serialized data has to copied.
 *
 * \param sendbufidx: it is used to return the abounf of bytes copied into the buffer.
 *
 * \param prog: the bpf program we have to copy.
 *
 * \return '0' if everything is fine, '-1' otherwise. The error message (if one)
 * is returned into the 'errbuf' field of the pcap_t structure.
 */
static int pcap_pack_bpffilter(pcap_t *fp, char *sendbuf, int *sendbufidx, struct bpf_program *prog)
{
    struct rpcap_filter *filter;
    struct rpcap_filterbpf_insn *insn;
    struct bpf_insn *bf_insn;
    struct bpf_program fake_prog;        /* To be used just in case the user forgot to set a filter */
    unsigned int i;

    if (prog->bf_len == 0)    /* No filters have been specified; so, let's apply a "fake" filter */
    {
        if (pcap_compile(fp, &fake_prog, NULL /* buffer */, 1, 0) == -1)
            return -1;

        prog = &fake_prog;
    }

    filter = (struct rpcap_filter *) sendbuf;

    // Check if there is enough space in the send buffer for the filter
    if (sock_bufferize(NULL, sizeof(struct rpcap_filter), NULL, sendbufidx,
        RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    // Set the filter type to indicate it's a BPF filter and convert to network byte order
    filter->filtertype = htons(RPCAP_UPDATEFILTER_BPF);
    // Set the number of items in the filter and convert to network byte order
    filter->nitems = htonl((int32)prog->bf_len);
    # 如果无法将过滤器规则缓冲区化，返回-1
    if (sock_bufferize(NULL, prog->bf_len * sizeof(struct rpcap_filterbpf_insn),
        NULL, sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    # 将过滤器规则缓冲区的下一个位置强制转换为rpcap_filterbpf_insn结构体指针
    insn = (struct rpcap_filterbpf_insn *) (filter + 1);
    # 获取过滤器规则程序的第一个指令
    bf_insn = prog->bf_insns;

    # 遍历过滤器规则程序的每一条指令
    for (i = 0; i < prog->bf_len; i++)
    {
        # 将过滤器规则程序中的指令转换为网络字节顺序，并存储到insn中
        insn->code = htons(bf_insn->code);
        insn->jf = bf_insn->jf;
        insn->jt = bf_insn->jt;
        # 将过滤器规则程序中的k值转换为网络字节顺序，并存储到insn中
        insn->k = htonl(bf_insn->k);

        # 指针移动到下一个位置
        insn++;
        bf_insn++;
    }

    # 返回0表示成功
    return 0;
/*
 * This function updates a filter on a remote host.
 * It is called when the user wants to update a filter.
 * In case we're capturing from the network, it sends the filter to our
 * peer.
 * This function is *not* called automatically when the user calls
 * pcap_setfilter().
 * There will be two cases:
 * - the capture has been started: in this case, pcap_setfilter_rpcap()
 *   calls pcap_updatefilter_remote()
 * - the capture has not started yet: in this case, pcap_setfilter_rpcap()
 *   stores the filter into the pcap_t structure, and then the filter is
 *   sent with pcap_startcap().
 *
 * WARNING This function *does not* clear the packet currently into the
 * buffers. Therefore, the user has to expect to receive some packets
 * that are related to the previous filter.  If you want to discard all
 * the packets before applying a new filter, you have to close the
 * current capture session and start a new one.
 *
 * XXX - we really should have pcap_setfilter() always discard packets
 * received with the old filter, and have a separate pcap_setfilter_noflush()
 * function that doesn't discard any packets.
 */
static int pcap_updatefilter_remote(pcap_t *fp, struct bpf_program *prog)
{
    // structure used when doing a remote live capture
    struct pcap_rpcap *pr = fp->priv;
    // temporary buffer in which data to be sent is buffered
    char sendbuf[RPCAP_NETBUF_SIZE];
    // index which keeps the number of bytes currently buffered
    int sendbufidx = 0;
    // To keep the reply message
    struct rpcap_header header;

    // Check if there is enough space in the buffer for the header
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL, &sendbufidx,
        RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    // Create the header for the update filter request
    rpcap_createhdr((struct rpcap_header *) sendbuf,
        pr->protocol_version, RPCAP_MSG_UPDATEFILTER_REQ, 0,
        sizeof(struct rpcap_filter) + prog->bf_len * sizeof(struct rpcap_filterbpf_insn));
}
    # 使用 BPF 过滤器过滤数据包，并将结果存储到 sendbuf 中
    if (pcap_pack_bpffilter(fp, &sendbuf[sendbufidx], &sendbufidx, prog))
        return -1;

    # 通过套接字发送 sendbuf 中的数据包到远程主机
    if (sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, sendbuf, sendbufidx, fp->errbuf,
        PCAP_ERRBUF_SIZE) < 0)
        return -1;

    # 接收并处理回复消息的消息头
    if (rpcap_process_msg_header(pr->rmt_sockctrl, pr->ctrl_ssl, pr->protocol_version,
        RPCAP_MSG_UPDATEFILTER_REQ, &header, fp->errbuf) == -1)
        return -1;

    # 如果消息头有内容，则丢弃它
    if (rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, header.plen, fp->errbuf) == -1)
        return -1;

    # 返回成功
    return 0;
}
static void
pcap_save_current_filter_rpcap(pcap_t *fp, const char *filter)
{
    struct pcap_rpcap *pr = fp->priv;    /* structure used when doing a remote live capture */

    /*
     * Check if:
     *  - We are on an remote capture
     *  - we do not want to capture RPCAP traffic
     *
     * If so, we have to save the current filter, because we have to
     * add some piece of stuff later
     */
    if (pr->rmt_clientside &&
        (pr->rmt_flags & PCAP_OPENFLAG_NOCAPTURE_RPCAP))
    {
        if (pr->currentfilter)
            free(pr->currentfilter);  // 释放之前保存的过滤器

        if (filter == NULL)
            filter = "";  // 如果过滤器为空，则设置为空字符串

        pr->currentfilter = strdup(filter);  // 复制过滤器内容到当前过滤器
    }
}

/*
 * This function sends a filter to a remote host.
 *
 * This function is called when the user wants to set a filter.
 * It sends the filter to our peer.
 * This function is called automatically when the user calls pcap_setfilter().
 *
 * Parameters and return values are exactly the same of pcap_setfilter().
 */
static int pcap_setfilter_rpcap(pcap_t *fp, struct bpf_program *prog)
{
    struct pcap_rpcap *pr = fp->priv;    /* structure used when doing a remote live capture */

    if (!pr->rmt_capstarted)
    {
        /* copy filter into the pcap_t structure */
        if (install_bpf_program(fp, prog) == -1)  // 将过滤器安装到 pcap_t 结构中
            return -1;
        return 0;
    }

    /* we have to update a filter during run-time */
    if (pcap_updatefilter_remote(fp, prog))  // 在运行时更新过滤器
        return -1;

    return 0;
}

/*
 * This function updates the current filter in order not to capture rpcap
 * packets.
 *
 * This function is called *only* when the user wants exclude RPCAP packets
 * related to the current session from the captured packets.
 *
 * \return '0' if everything is fine, '-1' otherwise. The error message (if one)
 * is returned into the 'errbuf' field of the pcap_t structure.
 */
static int pcap_createfilter_norpcappkt(pcap_t *fp, struct bpf_program *prog)
{
    # 定义一个指向 pcap_rpcap 结构体的指针 pr，用于进行远程实时抓包
    struct pcap_rpcap *pr = fp->priv;    /* structure used when doing a remote live capture */
    # 初始化 RetVal 为 0
    int RetVal = 0;

    # 如果设置了不捕获 RPCAP 流量的标志位，更新过滤器
    if (pr->rmt_flags & PCAP_OPENFLAG_NOCAPTURE_RPCAP)
    }

    # 返回 RetVal 的值
    return RetVal;
static int pcap_setsampling_remote(pcap_t *fp)
{
    // 获取 pcap_t 结构体中的私有数据结构 pcap_rpcap
    struct pcap_rpcap *pr = fp->priv;    
    // 用于发送数据的临时缓冲区
    char sendbuf[RPCAP_NETBUF_SIZE];
    // 当前缓冲区中已经发送的字节数
    int sendbufidx = 0;            
    // 用于保存回复消息的头部
    struct rpcap_header header;        
    // 用于发送采样参数到远程主机的结构体
    struct rpcap_sampling *sampling_pars;    

    // 如果没有请求采样，返回 'ok'
    if (fp->rmt_samp.method == PCAP_SAMP_NOSAMP)
        return 0;

    /*
     * 检查不适合放入消息中的采样参数。
     * 我们将让服务器抱怨适合放入消息中的无效参数。
     */
    if (fp->rmt_samp.method < 0 || fp->rmt_samp.method > 255) {
        // 将错误信息写入 errbuf 成员
        snprintf(fp->errbuf, PCAP_ERRBUF_SIZE,
            "Invalid sampling method %d", fp->rmt_samp.method);
        return -1;
    }
    if (fp->rmt_samp.value < 0 || fp->rmt_samp.value > 65535) {
        // 将错误信息写入 errbuf 成员
        snprintf(fp->errbuf, PCAP_ERRBUF_SIZE,
            "Invalid sampling value %d", fp->rmt_samp.value);
        return -1;
    }

    // 检查是否有足够的空间来放入消息头部
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        return -1;
}
    # 创建一个 RPCAP 消息头，用于设置抽样参数请求
    rpcap_createhdr((struct rpcap_header *) sendbuf,
        pr->protocol_version, RPCAP_MSG_SETSAMPLING_REQ, 0,
        sizeof(struct rpcap_sampling));

    # 填充远程打开适配器所需的结构
    sampling_pars = (struct rpcap_sampling *) &sendbuf[sendbufidx];

    # 将结构体缓冲区化，检查是否满足发送条件
    if (sock_bufferize(NULL, sizeof(struct rpcap_sampling), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, fp->errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    # 将结构体缓冲区清零
    memset(sampling_pars, 0, sizeof(struct rpcap_sampling));

    # 设置抽样方法和值
    sampling_pars->method = (uint8)fp->rmt_samp.method;
    sampling_pars->value = (uint16)htonl(fp->rmt_samp.value);

    # 发送消息头
    if (sock_send(pr->rmt_sockctrl, pr->ctrl_ssl, sendbuf, sendbufidx, fp->errbuf,
        PCAP_ERRBUF_SIZE) < 0)
        return -1;

    # 接收和处理回复消息头
    if (rpcap_process_msg_header(pr->rmt_sockctrl, pr->ctrl_ssl, pr->protocol_version,
        RPCAP_MSG_SETSAMPLING_REQ, &header, fp->errbuf) == -1)
        return -1;

    # 如果有内容，则丢弃
    if (rpcap_discard(pr->rmt_sockctrl, pr->ctrl_ssl, header.plen, fp->errbuf) == -1)
        return -1;

    # 返回成功
    return 0;
static int rpcap_doauth(SOCKET sockctrl, SSL *ssl, uint8 *ver,
/* 
 * 这个函数执行身份验证和协议版本协商。
 * 它是必需的，以便与另一端建立连接。
 * 它在控制套接字上发送身份验证参数并读取回复。
 * 如果回复是成功的指示，它检查回复是否包括服务器支持的最小和最大版本；
 * 如果没有，它假定两者都是0，因为这意味着它是一个不返回支持的版本号的较旧的服务器，
 * 因此它只支持版本0。
 * 然后它尝试确定我们和服务器都支持的最大版本。
 * 如果它能找到这样的版本，它设置我们使用该版本；否则，它失败，表示我们和服务器都不支持版本。
 *
 * \param sock:我们当前使用的套接字。
 *
 * \param ver:指向要设置的协议版本号的变量的指针。
 *
 * \param byte_swapped:指向要设置的变量的指针，如果服务器说它的字节顺序与我们的不同，则设置为1，否则设置为0（无论它与我们的相同还是未知）。
 *
 * \param auth:必须发送的身份验证参数。
 *
 * \param errbuf:指向用户分配的缓冲区的指针（大小为PCAP_ERRBUF_SIZE），它将包含错误消息（如果有的话）。
 * 它可能是网络问题或授权失败的原因。
 *
 * \return 如果一切正常，则返回'0'，如果出现错误，则返回'-1'。对于错误，错误消息字符串将在'errbuf'变量中返回。
 */
    // 定义一个指向整型的指针变量byte_swapped
    int *byte_swapped, 
    // 定义一个结构体指针变量auth，指向pcap_rmtauth类型的结构体
    struct pcap_rmtauth *auth, 
    // 定义一个指向字符的指针变量errbuf
    char *errbuf)
{
    char sendbuf[RPCAP_NETBUF_SIZE];    /* 临时缓冲区，用于缓存需要发送的数据 */
    int sendbufidx = 0;            /* 当前缓冲的字节数索引 */
    uint16 length;                /* 消息负载的长度 */
    struct rpcap_auth *rpauth;    /* RPCAP 认证结构体指针 */
    uint16 auth_type;             /* 认证类型 */
    struct rpcap_header header;   /* RPCAP 消息头 */
    size_t str_length;            /* 字符串长度 */
    uint32 plen;                  /* 消息长度 */
    struct rpcap_authreply authreply;    /* 认证回复消息 */
    uint8 ourvers;                /* 我们的版本 */
    int has_byte_order;           /* 服务器发送了它的字节顺序魔术数 */
    u_int their_byte_order_magic; /* 它的字节顺序魔术数 */

    if (auth)    /* 如果有认证信息 */
    {
        switch (auth->type)    /* 根据认证类型进行处理 */
        {
        case RPCAP_RMTAUTH_NULL:    /* 空认证 */
            length = sizeof(struct rpcap_auth);
            break;

        case RPCAP_RMTAUTH_PWD:    /* 密码认证 */
            length = sizeof(struct rpcap_auth);
            if (auth->username)    /* 如果有用户名 */
            {
                str_length = strlen(auth->username);
                if (str_length > 65535)    /* 用户名过长 */
                {
                    snprintf(errbuf, PCAP_ERRBUF_SIZE, "User name is too long (> 65535 bytes)");
                    return -1;
                }
                length += (uint16)str_length;
            }
            if (auth->password)    /* 如果有密码 */
            {
                str_length = strlen(auth->password);
                if (str_length > 65535)    /* 密码过长 */
                {
                    snprintf(errbuf, PCAP_ERRBUF_SIZE, "Password is too long (> 65535 bytes)");
                    return -1;
                }
                length += (uint16)str_length;
            }
            break;

        default:    /* 其他未识别的认证类型 */
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "Authentication type not recognized.");
            return -1;
        }

        auth_type = (uint16)auth->type;    /* 设置认证类型 */
    }
    else    /* 如果没有认证信息 */
    {
        auth_type = RPCAP_RMTAUTH_NULL;    /* 默认为空认证 */
        length = sizeof(struct rpcap_auth);
    }
}
    # 如果发送缓冲区的大小不足以容纳指定大小的数据包，返回-1
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    # 创建一个RPCAP消息头
    rpcap_createhdr((struct rpcap_header *) sendbuf, 0,
        RPCAP_MSG_AUTH_REQ, 0, length);

    # 将RPCAP认证结构体的指针指向发送缓冲区的当前位置
    rpauth = (struct rpcap_auth *) &sendbuf[sendbufidx];

    # 如果发送缓冲区的大小不足以容纳指定大小的数据包，返回-1
    if (sock_bufferize(NULL, sizeof(struct rpcap_auth), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, errbuf, PCAP_ERRBUF_SIZE))
        return -1;

    # 将RPCAP认证结构体的内容初始化为0
    memset(rpauth, 0, sizeof(struct rpcap_auth));

    # 设置认证类型
    rpauth->type = htons(auth_type);

    # 如果认证类型为RPCAP_RMTAUTH_PWD
    if (auth_type == RPCAP_RMTAUTH_PWD)
    {
        # 如果认证用户名不为空，设置用户名长度
        if (auth->username)
            rpauth->slen1 = (uint16)strlen(auth->username);
        else
            rpauth->slen1 = 0;

        # 如果发送缓冲区的大小不足以容纳指定大小的数据包，返回-1
        if (sock_bufferize(auth->username, rpauth->slen1, sendbuf,
            &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_BUFFERIZE, errbuf, PCAP_ERRBUF_SIZE))
            return -1;

        # 如果认证密码不为空，设置密码长度
        if (auth->password)
            rpauth->slen2 = (uint16)strlen(auth->password);
        else
            rpauth->slen2 = 0;

        # 如果发送缓冲区的大小不足以容纳指定大小的数据包，返回-1
        if (sock_bufferize(auth->password, rpauth->slen2, sendbuf,
            &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_BUFFERIZE, errbuf, PCAP_ERRBUF_SIZE))
            return -1;

        # 将用户名和密码长度转换为网络字节顺序
        rpauth->slen1 = htons(rpauth->slen1);
        rpauth->slen2 = htons(rpauth->slen2);
    }

    # 如果发送数据失败，返回-1
    if (sock_send(sockctrl, ssl, sendbuf, sendbufidx, errbuf,
        PCAP_ERRBUF_SIZE) < 0)
        return -1;

    # 接收和处理回复消息头
    if (rpcap_process_msg_header(sockctrl, ssl, 0, RPCAP_MSG_AUTH_REQ,
        &header, errbuf) == -1)
        return -1;

    # 如果回复消息的长度不为0
    plen = header.plen;
    if (plen != 0)
    {
        # 处理额外的信息
    }
    else
    {
        # 处理其他情况
    }
    {
        /* 不支持除版本 0 以外的版本 */
        authreply.minvers = 0;
        authreply.maxvers = 0;

        /*
         * 并且它没有告诉我们它的字节顺序是什么；假设它和我们的一样。
         */
        has_byte_order = 0;
        their_byte_order_magic = RPCAP_BYTE_ORDER_MAGIC;
    }

    /*
     * 好的，让我们从服务器支持的最大版本开始。
     */
    ourvers = authreply.maxvers;
#if RPCAP_MIN_VERSION != 0
    /*
     * 如果 RPCAP_MIN_VERSION 不等于 0
     * 如果我们的版本小于我们支持的最小版本，我们无法通信。
     */
    if (ourvers < RPCAP_MIN_VERSION)
        goto novers;
#endif

    /*
     * 如果我们的版本大于我们支持的最大版本，选择我们支持的最大版本。
     */
    if (ourvers > RPCAP_MAX_VERSION)
    {
        ourvers = RPCAP_MAX_VERSION;

        /*
         * 如果我们的版本小于他们支持的最小版本，我们无法通信。
         */
        if (ourvers < authreply.minvers)
            goto novers;
    }

    /*
     * 服务器的字节顺序与我们的相反吗？
     */
    if (their_byte_order_magic == RPCAP_BYTE_ORDER_MAGIC)
    {
        /* 不，它和我们的一样。 */
        *byte_swapped = 0;
    }
    else if (their_byte_order_magic == RPCAP_BYTE_ORDER_MAGIC_SWAPPED)
    {
        /* 是的，它和我们的相反。 */
        *byte_swapped = 1;
    }
    else
    {
        /* 他们发送了一些错误的内容。 */
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "服务器未向我们发送有效的字节顺序值");
        return -1;
    }

    *ver = ourvers;
    return 0;

novers:
    /*
     * 我们没有共同支持的版本；这是一个致命错误。
     */
    snprintf(errbuf, PCAP_ERRBUF_SIZE,
        "服务器不支持我们支持的任何协议版本");
    return -1;
}

/* 我们目前不支持非阻塞模式。 */
static int
pcap_getnonblock_rpcap(pcap_t *p)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "使用 rpcap 远程捕获不支持非阻塞模式");
    return (-1);
}

static int
pcap_setnonblock_rpcap(pcap_t *p, int nonblock _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "使用 rpcap 远程捕获不支持非阻塞模式");
    return (-1);
}

static int
rpcap_setup_session(const char *source, struct pcap_rmtauth *auth,
    int *activep, SOCKET *sockctrlp, uint8 *uses_sslp, SSL **sslp,
    # 定义整型变量 rmt_flags，用于存储远程标志
    int rmt_flags, 
    # 定义指向无符号8位整型的指针 protocol_versionp，用于存储协议版本
    uint8 *protocol_versionp, 
    # 定义指向整型的指针 byte_swappedp，用于存储字节交换信息
    int *byte_swappedp,
    # 定义字符指针变量 host，用于存储主机名
    char *host, 
    # 定义字符指针变量 port，用于存储端口号
    char *port, 
    # 定义字符指针变量 iface，用于存储接口信息
    char *iface, 
    # 定义字符指针变量 errbuf，用于存储错误信息
    char *errbuf)
{
    // 定义一个整型变量 type
    int type;
    // 定义一个指向 struct activehosts 结构体的指针 activeconn，表示活动连接（如果有的话）
    struct activehosts *activeconn;        
    // 定义一个整型变量 error，表示 rpcap_remoteact_getsock 是否出错
    int error;                

    /*
     * 确定源的类型（NULL、文件、本地、远程）。
     * 即使我们处于活动模式，也必须有一个有效的源字符串，
     * 因为否则调用以下函数将失败。
     */
    if (pcap_parsesrcstr_ex(source, &type, host, port, iface, uses_sslp,
        errbuf) == -1)
        return -1;

    /*
     * 必须是远程的。
     */
    if (type != PCAP_SRC_IFREMOTE)
    {
        // 如果不是远程接口，则返回错误信息
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "Non-remote interface passed to remote capture routine");
        return -1;
    }

    /*
     * 我们还不支持 DTLS，因此如果用户要求 TLS 连接并要求数据包通过 UDP 发送，
     * 我们必须放弃。
     */
    if (*uses_sslp && (rmt_flags & PCAP_OPENFLAG_DATATX_UDP))
    {
        // 如果用户要求 TLS 连接并要求数据包通过 UDP 发送，则返回错误信息
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "TLS not supported with UDP forward of remote packets");
        return -1;
    }

    /* 警告：这个调用可能是用户调用的第一个调用。 */
    /* 因此，我们必须初始化 Winsock 支持。 */
    if (sock_init(errbuf, PCAP_ERRBUF_SIZE) == -1)
        return -1;

    /* 检查是否处于活动模式 */
    activeconn = rpcap_remoteact_getsock(host, &error, errbuf);
    if (activeconn != NULL)
    {
        // 如果处于活动模式，则设置 activep 为 1，并设置其他相关参数
        *activep = 1;
        *sockctrlp = activeconn->sockctrl;
        *sslp = activeconn->ssl;
        *protocol_versionp = activeconn->protocol_version;
        *byte_swappedp = activeconn->byte_swapped;
    }
    else
        # 设置指针 activep 指向的值为 0
        *activep = 0;
        # 声明一个用于解析主机名的临时变量 hints
        struct addrinfo hints;        /* temp variable needed to resolve hostnames into to socket representation */
        # 声明一个用于存储解析后的主机信息的临时变量 addrinfo
        struct addrinfo *addrinfo;    /* temp variable needed to resolve hostnames into to socket representation */

        # 如果存在错误
        if (error)
        {
            '''
             * 调用失败。
             */
            # 返回 -1
            return -1;
        }

        '''
         * 我们不处于活动模式；让我们尝试打开一个新的
         * 控制连接。
         */
        # 将 hints 变量清零
        memset(&hints, 0, sizeof(struct addrinfo));
        # 设置 hints 变量的属性
        hints.ai_family = PF_UNSPEC;
        hints.ai_socktype = SOCK_STREAM;

        # 如果端口未指定
        if (port[0] == 0)
        {
            ''' 
            # 用户选择不指定端口
            # 初始化地址信息并存储在 addrinfo 中
            if (sock_initaddress(host, RPCAP_DEFAULT_NETPORT,
                &hints, &addrinfo, errbuf, PCAP_ERRBUF_SIZE) == -1)
                return -1;
        }
        else
        {
            # 初始化地址信息并存储在 addrinfo 中
            if (sock_initaddress(host, port, &hints, &addrinfo,
                errbuf, PCAP_ERRBUF_SIZE) == -1)
                return -1;
        }

        # 打开一个新的控制连接
        if ((*sockctrlp = sock_open(host, addrinfo, SOCKOPEN_CLIENT, 0,
            errbuf, PCAP_ERRBUF_SIZE)) == INVALID_SOCKET)
        {
            # 释放 addrinfo 变量
            freeaddrinfo(addrinfo);
            # 返回 -1
            return -1;
        }

        # addrinfo 变量不再使用
        freeaddrinfo(addrinfo);
        addrinfo = NULL;

        # 如果使用 SSL
        if (*uses_sslp)
        {
#ifdef HAVE_OPENSSL
            // 如果有 OpenSSL 支持，则进行 SSL 提升
            *sslp = ssl_promotion(0, *sockctrlp, errbuf, PCAP_ERRBUF_SIZE);
            // 如果 SSL 提升失败，则关闭控制套接字并返回 -1
            if (!*sslp)
            {
                sock_close(*sockctrlp, NULL, 0);
                return -1;
            }
#else
            // 如果没有 OpenSSL 支持，则返回错误信息并关闭控制套接字并返回 -1
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "No TLS support");
            sock_close(*sockctrlp, NULL, 0);
            return -1;
#endif
        }

        // 进行身份验证
        if (rpcap_doauth(*sockctrlp, *sslp, protocol_versionp, byte_swappedp, auth, errbuf) == -1)
        {
#ifdef HAVE_OPENSSL
            // 如果有 OpenSSL 支持，并且 SSL 提升成功，则结束使用 SSL 句柄
            if (*sslp)
            {
                // Finish using the SSL handle for the socket.
                // This must be done *before* the socket is
                // closed.
                ssl_finish(*sslp);
            }
#endif
            // 关闭控制套接字并返回 -1
            sock_close(*sockctrlp, NULL, 0);
            return -1;
        }
    }
    // 返回 0 表示成功
    return 0;
}
/*
 * This function opens a remote adapter by opening an RPCAP connection and
 * so on.
 *
 * It does the job of pcap_open_live() for a remote interface; it's called
 * by pcap_open() for remote interfaces.
 *
 * We do not start the capture until pcap_startcapture_remote() is called.
 *
 * This is because, when doing a remote capture, we cannot start capturing
 * data as soon as the 'open adapter' command is sent. Suppose the remote
 * adapter is already overloaded; if we start a capture (which, by default,
 * has a NULL filter) the new traffic can saturate the network.
 *
 * Instead, we want to "open" the adapter, then send a "start capture"
 * command only when we're ready to start the capture.
 * This function does this job: it sends an "open adapter" command
 * (according to the RPCAP protocol), but it does not start the capture.
 *
 * Since the other libpcap functions do not share this way of life, we
 * have to do some dirty things in order to make everything work.
 *
 * \param source: see pcap_open().
 * \param snaplen: see pcap_open().
 * \param flags: see pcap_open().
 * \param read_timeout: see pcap_open().
 * \param auth: see pcap_open().
 * \param errbuf: see pcap_open().
 *
 * \return a pcap_t pointer in case of success, NULL otherwise. In case of
 * success, the pcap_t pointer can be used as a parameter to the following
 * calls (pcap_compile() and so on). In case of problems, errbuf contains
 * a text explanation of error.
 *
 * WARNING: In case we call pcap_compile() and the capture has not yet
 * been started, the filter will be saved into the pcap_t structure,
 * and it will be sent to the other host later (when
 * pcap_startcapture_remote() is called).
 */
pcap_t *pcap_open_rpcap(const char *source, int snaplen, int flags, int read_timeout, struct pcap_rmtauth *auth, char *errbuf)
{
    // Declare a pointer to a pcap_t structure
    pcap_t *fp;
    // Declare a pointer to a string for the source
    char *source_str;
    // Declare a pointer to a pcap_rpcap structure for remote live capture
    struct pcap_rpcap *pr;        /* structure used when doing a remote live capture */
    // 定义存储主机名、控制端口、接口名的字符数组
    char host[PCAP_BUF_SIZE], ctrlport[PCAP_BUF_SIZE], iface[PCAP_BUF_SIZE];
    // 定义控制套接字
    SOCKET sockctrl;
    // 定义 SSL 变量并初始化为 NULL
    SSL *ssl = NULL;
    // 协商的协议版本
    uint8 protocol_version;
    // 服务器是否已知为字节交换
    int byte_swapped;
    // 是否处于活动状态
    int active;
    // 数据包长度
    uint32 plen;
    // 用于缓冲待发送数据的临时缓冲区
    char sendbuf[RPCAP_NETBUF_SIZE];
    // 当前缓冲的字节数索引
    int sendbufidx = 0;

    /* 与 RPCAP 相关的变量 */
    // RPCAP 数据包的头部
    struct rpcap_header header;
    // 打开回复消息
    struct rpcap_openreply openreply;

    // 创建一个 pcap_rpcap 结构体
    fp = PCAP_CREATE_COMMON(errbuf, struct pcap_rpcap);
    // 如果创建失败，则返回空指针
    if (fp == NULL)
    {
        return NULL;
    }
    // 复制源字符串
    source_str = strdup(source);
    // 如果复制失败，则返回空指针
    if (source_str == NULL) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return NULL;
    }

    /*
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为最大允许的值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     *
     * XXX - 我们应该把这个留给远程服务器来做吗？
     */
    if (snaplen <= 0 || snaplen > MAXIMUM_SNAPLEN)
        snaplen = MAXIMUM_SNAPLEN;

    // 设置设备名称
    fp->opt.device = source_str;
    // 设置快照长度
    fp->snapshot = snaplen;
    // 设置超时时间
    fp->opt.timeout = read_timeout;
    // 获取私有数据
    pr = fp->priv;
    // 设置远程标志
    pr->rmt_flags = flags;

    /*
     * 尝试与服务器建立会话。
     */
    if (rpcap_setup_session(fp->opt.device, auth, &active, &sockctrl,
        &pr->uses_ssl, &ssl, flags, &protocol_version, &byte_swapped,
        host, ctrlport, iface, errbuf) == -1)
    {
        /* 会话设置失败。 */
        // 关闭 pcap
        pcap_close(fp);
        return NULL;
    }

    // 一切顺利，保存 SSL 处理程序
    # 将 ssl 赋值给 ssl_main
    ssl_main = ssl;

    # 现在是时候开始使用 RPCAP 协议了
    # RPCAP open 命令：创建请求消息
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL,
        &sendbufidx, RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, errbuf, PCAP_ERRBUF_SIZE))
        goto error_nodiscard;

    # 创建 RPCAP 消息头
    rpcap_createhdr((struct rpcap_header *) sendbuf, protocol_version,
        RPCAP_MSG_OPEN_REQ, 0, (uint32) strlen(iface));

    # 将 iface 缓冲到 sendbuf 中
    if (sock_bufferize(iface, (int) strlen(iface), sendbuf, &sendbufidx,
        RPCAP_NETBUF_SIZE, SOCKBUF_BUFFERIZE, errbuf, PCAP_ERRBUF_SIZE))
        goto error_nodiscard;

    # 发送消息
    if (sock_send(sockctrl, ssl, sendbuf, sendbufidx, errbuf,
        PCAP_ERRBUF_SIZE) < 0)
        goto error_nodiscard;

    # 接收和处理回复消息头
    if (rpcap_process_msg_header(sockctrl, ssl, protocol_version,
        RPCAP_MSG_OPEN_REQ, &header, errbuf) == -1)
        goto error_nodiscard;
    plen = header.plen;

    # 读取回复体
    if (rpcap_recv(sockctrl, ssl, (char *)&openreply,
        sizeof(struct rpcap_openreply), &plen, errbuf) == -1)
        goto error;

    # 丢弃消息的其余部分（如果有的话）
    if (rpcap_discard(sockctrl, ssl, plen, errbuf) == -1)
        goto error_nodiscard;

    # 设置 pcap_t 结构的正确字段
    fp->linktype = ntohl(openreply.linktype);
    pr->rmt_sockctrl = sockctrl;
    pr->ctrl_ssl = ssl;
    pr->protocol_version = protocol_version;
    pr->byte_swapped = byte_swapped;
    pr->rmt_clientside = 1;

    # 从函数末尾复制的代码
    fp->read_op = pcap_read_rpcap;
    fp->save_current_filter_op = pcap_save_current_filter_rpcap;
    fp->setfilter_op = pcap_setfilter_rpcap;
    fp->getnonblock_op = pcap_getnonblock_rpcap;
    fp->setnonblock_op = pcap_setnonblock_rpcap;
    fp->stats_op = pcap_stats_rpcap;
#ifdef _WIN32
    // 如果是在 Windows 平台下编译，设置 fp 结构体的 stats_ex_op 指向 pcap_stats_ex_rpcap 函数
    fp->stats_ex_op = pcap_stats_ex_rpcap;
#endif
    // 设置 fp 结构体的 cleanup_op 指向 pcap_cleanup_rpcap 函数
    fp->cleanup_op = pcap_cleanup_rpcap;

    // 将 fp 结构体的 activated 成员设置为 1
    fp->activated = 1;
    // 返回 fp 结构体指针
    return fp;

error:
    /*
     * 当连接建立后，如果出现错误，需要立即返回 NULL；
     * 当连接建立后，如果出现错误，需要跳转到 error 标签处，以便正确关闭连接。
     */

    /*
     * 丢弃消息的剩余部分。
     * 已经报告了一个错误；如果这里出现错误，只需继续执行。
     */
    (void)rpcap_discard(sockctrl, pr->ctrl_ssl, plen, NULL);

error_nodiscard:
    // 如果连接未激活
    if (!active)
    {
#ifdef HAVE_OPENSSL
        // 如果使用了 OpenSSL
        if (ssl)
        {
            // 结束使用 SSL 句柄
            // 必须在关闭套接字之前完成
            ssl_finish(ssl);
        }
#endif
        // 关闭控制连接套接字
        sock_close(sockctrl, NULL, 0);
    }

    // 关闭 fp 结构体指向的 pcap_t 对象
    pcap_close(fp);
    // 返回 NULL
    return NULL;
}

/* 在 pcap_findalldevs_ex() 中使用的字符串标识符 */
#define PCAP_TEXT_SOURCE_ADAPTER "Network adapter"
#define PCAP_TEXT_SOURCE_ADAPTER_LEN (sizeof PCAP_TEXT_SOURCE_ADAPTER - 1)
/* 在 pcap_findalldevs_ex() 中使用的字符串标识符 */
#define PCAP_TEXT_SOURCE_ON_REMOTE_HOST "on remote node"
#define PCAP_TEXT_SOURCE_ON_REMOTE_HOST_LEN (sizeof PCAP_TEXT_SOURCE_ON_REMOTE_HOST - 1)

static void
freeaddr(struct pcap_addr *addr)
{
    // 释放地址结构体中的成员内存
    free(addr->addr);
    free(addr->netmask);
    free(addr->broadaddr);
    free(addr->dstaddr);
    free(addr);
}

int
pcap_findalldevs_ex_remote(const char *source, struct pcap_rmtauth *auth, pcap_if_t **alldevs, char *errbuf)
{
    uint8 protocol_version;        /* 协议版本 */
    int byte_swapped;        /* 服务器字节顺序与我们的不同 */
    SOCKET sockctrl;        /* 控制连接的套接字描述符 */
    SSL *ssl = NULL;        /* 可选的 sockctrl 的 SSL 处理程序 */
    uint32 plen;
    struct rpcap_header header;    /* 定义一个结构体，用于保存rpcap协议的通用头部信息 */
    int i, j;        /* 临时变量 */
    int nif;        /* 列出的接口数量 */
    int active;            /* 如果远程对等方处于活动模式，则为'true' */
    uint8 uses_ssl;        /* 使用SSL加密的标志 */
    char host[PCAP_BUF_SIZE], port[PCAP_BUF_SIZE];    /* 存储主机和端口信息的数组 */
    char tmpstring[PCAP_BUF_SIZE + 1];        /* 用于将名称和描述从'旧'语法转换为'新'语法 */
    pcap_if_t *lastdev;    /* pcap_if_t列表中的最后一个设备 */
    pcap_if_t *dev;        /* 要添加到pcap_if_t列表中的设备 */

    /* 列表开始为空 */
    (*alldevs) = NULL;
    lastdev = NULL;

    /*
     * 尝试与服务器建立会话。
     */
    if (rpcap_setup_session(source, auth, &active, &sockctrl, &uses_ssl,
        &ssl, 0, &protocol_version, &byte_swapped, host, port, NULL,
        errbuf) == -1)
    {
        /* 会话设置失败。 */
        return -1;
    }

    /* RPCAP findalldevs命令 */
    rpcap_createhdr(&header, protocol_version, RPCAP_MSG_FINDALLIF_REQ,
        0, 0);

    if (sock_send(sockctrl, ssl, (char *)&header, sizeof(struct rpcap_header),
        errbuf, PCAP_ERRBUF_SIZE) < 0)
        goto error_nodiscard;

    /* 接收并处理回复消息头 */
    if (rpcap_process_msg_header(sockctrl, ssl, protocol_version,
        RPCAP_MSG_FINDALLIF_REQ, &header, errbuf) == -1)
        goto error_nodiscard;

    plen = header.plen;

    /* 读取接口数量 */
    nif = ntohs(header.value);

    /* 循环直到接收到所有接口 */
    for (i = 0; i < nif; i++)
    }

    /* 丢弃消息的其余部分 */
    if (rpcap_discard(sockctrl, ssl, plen, errbuf) == 1)
        goto error_nodiscard;

    /* 只有在远程机器处于被动模式时才需要关闭控制连接 */
    if (!active)
        /* 不发送 RPCAP_CLOSE，因为我们没有打开 pcap_t；不需要释放资源 */
#ifdef HAVE_OPENSSL
        // 如果有 OpenSSL 支持
        if (ssl)
        {
            // 如果使用 SSL 处理套接字
            // 这必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        // 关闭控制套接字
        if (sock_close(sockctrl, errbuf, PCAP_ERRBUF_SIZE))
            return -1;
    }

    /* 为了避免 sock_init() 的数量不一致 */
    // 清理套接字
    sock_cleanup();

    // 返回 0
    return 0;

error:
    /*
     * 如果出现错误，我不想用新的错误覆盖它
     * 如果以下调用失败，我希望始终返回原始错误。
     *
     * 注意：当我们尝试关闭连接时，我不想用新的错误覆盖它。
     * 这是因为 rpcapd 中的先前错误要求关闭连接。在这种情况下，我们已经在 rpspck_isheaderok() 中识别到了这一点，并且我们已经确认了关闭。
     * 在这种意义上，这个调用在这里是无用的（但是在客户端生成错误的情况下是需要的）。
     *
     * 检查是否已经读取了所有数据；如果没有，丢弃多余的数据
     */
    // 丢弃控制连接中的数据
    (void) rpcap_discard(sockctrl, ssl, plen, NULL);

error_nodiscard:
    /* 仅在远程机器处于被动模式时关闭控制连接 */
    if (!active)
    {
#ifdef HAVE_OPENSSL
        // 如果有 OpenSSL 支持
        if (ssl)
        {
            // 如果使用 SSL 处理套接字
            // 这必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        // 关闭控制套接字
        sock_close(sockctrl, NULL, 0);
    }

    /* 为了避免 sock_init() 的数量不一致 */
    // 清理套接字
    sock_cleanup();

    // 释放我们分配的所有接口
    pcap_freealldevs(*alldevs);

    // 返回 -1
    return -1;
}

/*
 * 主动模式例程。
 *
 * 旧的 libpcap API 有些丑陋，并且使得实现主动模式变得困难；
 * 我们为它提供了一些仅与 rpcap 一起使用的 API。
 */
# 定义一个名为 pcap_remoteact_accept_ex 的函数，接受参数 address, port, hostlist, connectinghost, auth, uses_ssl, errbuf
SOCKET pcap_remoteact_accept_ex(const char *address, const char *port, const char *hostlist, char *connectinghost, struct pcap_rmtauth *auth, int uses_ssl, char *errbuf)
{
    /* 用于存储与套接字相关的变量 */
    struct addrinfo hints;            /* 临时结构体，用于保存打开新套接字所需的设置 */
    struct addrinfo *addrinfo;        /* 保存 addrinfo 链，打开新套接字所需 */
    struct sockaddr_storage from;    /* 通用的 sockaddr_storage 变量 */
    socklen_t fromlen;                /* 保存 sockaddr_storage 变量的长度 */
    SOCKET sockctrl;                /* 保存主套接字标识符 */
    SSL *ssl = NULL;                /* 可选的 sockctrl 的 SSL 处理程序 */
    uint8 protocol_version;            /* 协商的协议版本 */
    int byte_swapped;            /* 如果服务器字节顺序已知与我们相反，则为 1 */
    struct activehosts *temp, *prev;    /* 扫描主机列表链所需的临时变量 */

    *connectinghost = 0;        /* 以防万一 */

    /* 准备打开一个新的服务器套接字 */
    memset(&hints, 0, sizeof(struct addrinfo));
    /* 警告：目前仅支持 ipv4 和 IPv6 中的一种套接字族 */
    hints.ai_family = AF_INET;        /* PF_UNSPEC 以同时支持 IPv4 和 IPv6 服务器 */
    hints.ai_flags = AI_PASSIVE;    /* 准备绑定() 套接字 */
    hints.ai_socktype = SOCK_STREAM;

    /* 警告：这个调用可能是用户调用的第一个调用 */
    /* 因此，我们必须初始化 Winsock 支持 */
    if (sock_init(errbuf, PCAP_ERRBUF_SIZE) == -1)
        return (SOCKET)-1;

    /* 开始工作 */
    if ((port == NULL) || (port[0] == 0))
    {
        if (sock_initaddress(address, RPCAP_DEFAULT_NETPORT_ACTIVE, &hints, &addrinfo, errbuf, PCAP_ERRBUF_SIZE) == -1)
        {
            return (SOCKET)-2;
        }
    }
    else
    {
        // 如果初始化地址失败，返回错误码-2
        if (sock_initaddress(address, port, &hints, &addrinfo, errbuf, PCAP_ERRBUF_SIZE) == -1)
        {
            return (SOCKET)-2;
        }
    }


    // 打开一个服务器套接字
    if ((sockmain = sock_open(NULL, addrinfo, SOCKOPEN_SERVER, 1, errbuf, PCAP_ERRBUF_SIZE)) == INVALID_SOCKET)
    {
        // 释放地址信息
        freeaddrinfo(addrinfo);
        // 返回错误码-2
        return (SOCKET)-2;
    }
    // 释放地址信息
    freeaddrinfo(addrinfo);

    /* Connection creation */
    // 设置 fromlen 的大小
    fromlen = sizeof(struct sockaddr_storage);

    // 接受客户端的连接请求
    sockctrl = accept(sockmain, (struct sockaddr *) &from, &fromlen);

    /* We're not using sock_close, since we do not want to send a shutdown */
    /* (which is not allowed on a non-connected socket) */
    // 关闭主套接字
    closesocket(sockmain);
    // 将主套接字置为0
    sockmain = 0;

    // 如果接受连接失败，返回错误信息
    if (sockctrl == INVALID_SOCKET)
    {
        // 获取错误信息
        sock_geterrmsg(errbuf, PCAP_ERRBUF_SIZE, "accept() failed");
        // 返回错误码-2
        return (SOCKET)-2;
    }

    // 如果使用 SSL，提前升级为 SSL，避免发送错误消息
    if (uses_ssl)
    {
#ifdef HAVE_OPENSSL
        // 如果有 OpenSSL 支持，则进行 SSL 升级
        ssl = ssl_promotion(0, sockctrl, errbuf, PCAP_ERRBUF_SIZE);
        // 如果 SSL 升级失败，则关闭控制套接字并返回错误
        if (! ssl)
        {
            sock_close(sockctrl, NULL, 0);
            return (SOCKET)-1;
        }
#else
        // 如果没有 OpenSSL 支持，则返回错误信息并关闭控制套接字
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "No TLS support");
        sock_close(sockctrl, NULL, 0);
        return (SOCKET)-1;
#endif
    }

    /* 获取连接主机的名称的数值形式 */
    if (getnameinfo((struct sockaddr *) &from, fromlen, connectinghost, RPCAP_HOSTLIST_SIZE, NULL, 0, NI_NUMERICHOST))
    {
        // 如果获取连接主机名称失败，则发送错误信息并关闭套接字
        sock_geterrmsg(errbuf, PCAP_ERRBUF_SIZE,
            "getnameinfo() failed");
        rpcap_senderror(sockctrl, ssl, 0, PCAP_ERR_REMOTEACCEPT, errbuf, NULL);
#ifdef HAVE_OPENSSL
        if (ssl)
        {
            // 完成使用套接字的 SSL 句柄
            // 这必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        sock_close(sockctrl, NULL, 0);
        return (SOCKET)-1;
    }

    /* 检查连接主机是否在允许的主机列表中 */
    if (sock_check_hostlist((char *)hostlist, RPCAP_HOSTLIST_SEP, &from, errbuf, PCAP_ERRBUF_SIZE) < 0)
    {
        // 如果连接主机不在允许的主机列表中，则发送错误信息并关闭套接字
        rpcap_senderror(sockctrl, ssl, 0, PCAP_ERR_REMOTEACCEPT, errbuf, NULL);
#ifdef HAVE_OPENSSL
        if (ssl)
        {
            // 完成使用套接字的 SSL 句柄
            // 这必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        sock_close(sockctrl, NULL, 0);
        return (SOCKET)-1;
    }

    /*
     * 向远程机器发送认证信息。
     */
    if (rpcap_doauth(sockctrl, ssl, &protocol_version, &byte_swapped,
        auth, errbuf) == -1)
    {
        /* 无法恢复的错误。发送错误信息并关闭套接字 */
        rpcap_senderror(sockctrl, ssl, 0, PCAP_ERR_REMOTEACCEPT, errbuf, NULL);
#ifdef HAVE_OPENSSL
        // 如果支持 OpenSSL，则执行以下代码块
        if (ssl)
        {
            // 完成对套接字的 SSL 处理
            // 必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        // 关闭控制套接字
        sock_close(sockctrl, NULL, 0);
        // 返回错误码 -3
        return (SOCKET)-3;
    }

    /* 检查该主机是否已经建立了控制连接 */

    /* 初始化指针 */
    temp = activeHosts;
    prev = NULL;

    while (temp)
    {
        /* 该主机已经建立了活动连接，因此无需更新主机列表 */
        if (sock_cmpaddr(&temp->host, &from) == 0)
            return sockctrl;

        prev = temp;
        temp = temp->next;
    }

    /* 列表中不存在该主机；因此需要更新列表 */
    if (prev)
    {
        prev->next = (struct activehosts *) malloc(sizeof(struct activehosts));
        temp = prev->next;
    }
    else
    {
        activeHosts = (struct activehosts *) malloc(sizeof(struct activehosts));
        temp = activeHosts;
    }

    if (temp == NULL)
    {
        // 分配内存失败，发送错误消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc() failed");
        rpcap_senderror(sockctrl, ssl, protocol_version, PCAP_ERR_REMOTEACCEPT, errbuf, NULL);
#ifdef HAVE_OPENSSL
        if (ssl)
        {
            // 完成对套接字的 SSL 处理
            // 必须在套接字关闭之前完成
            ssl_finish(ssl);
        }
#endif
        // 关闭控制套接字
        sock_close(sockctrl, NULL, 0);
        // 返回错误码 -1
        return (SOCKET)-1;
    }

    // 复制主机信息到临时结构体
    memcpy(&temp->host, &from, fromlen);
    temp->sockctrl = sockctrl;
    temp->ssl = ssl;
    temp->protocol_version = protocol_version;
    temp->byte_swapped = byte_swapped;
    temp->next = NULL;

    // 返回控制套接字
    return sockctrl;
}

// 远程接受连接
SOCKET pcap_remoteact_accept(const char *address, const char *port, const char *hostlist, char *connectinghost, struct pcap_rmtauth *auth, char *errbuf)
{
    # 调用pcap_remoteact_accept_ex函数，传入参数address, port, hostlist, connectinghost, auth, 0, errbuf，并返回结果
    return pcap_remoteact_accept_ex(address, port, hostlist, connectinghost, auth, 0, errbuf);
    }
    // 关闭远程活动主机的连接
    int pcap_remoteact_close(const char *host, char *errbuf)
    {
        // 临时变量，用于扫描主机列表链
        struct activehosts *temp, *prev;
        // 临时变量，用于将主机名转换为其地址
        struct addrinfo hints, *addrinfo, *ai_next;
        int retval;

        // 初始化临时变量
        temp = activeHosts;
        prev = NULL;

        // 获取与 'host' 对应的网络地址
        addrinfo = NULL;
        memset(&hints, 0, sizeof(struct addrinfo));
        hints.ai_family = PF_UNSPEC;
        hints.ai_socktype = SOCK_STREAM;

        // 初始化地址信息
        retval = sock_initaddress(host, NULL, &hints, &addrinfo, errbuf, PCAP_ERRBUF_SIZE);
        if (retval != 0)
        {
            return -1;
        }

        // 遍历主机列表
        while (temp)
        {
            ai_next = addrinfo;
            while (ai_next)
            {
                // 比较主机地址
                if (sock_cmpaddr(&temp->host, (struct sockaddr_storage *) ai_next->ai_addr) == 0)
                {
                    struct rpcap_header header;
                    int status = 0;

                    // 创建关闭连接的消息头
                    rpcap_createhdr(&header, temp->protocol_version, RPCAP_MSG_CLOSE, 0, 0);

                    // 发送消息头，关闭连接
                    if (sock_send(temp->sockctrl, temp->ssl, (char *)&header, sizeof(struct rpcap_header), errbuf, PCAP_ERRBUF_SIZE) < 0)
                    {
                        // 如果发送失败，处理错误
#ifdef HAVE_OPENSSL
                        if (temp->ssl)
                        {
                            // 完成使用 SSL 句柄
                            // 这必须在关闭套接字之前完成
                            ssl_finish(temp->ssl);
                        }
#endif
                    (void)sock_close(temp->sockctrl, NULL,
                       0);
                    status = -1;
                }
                else
                {
#ifdef HAVE_OPENSSL
                    if (temp->ssl)
                    {
                        // Finish using the SSL handle
                        // for the socket.
                        // This must be done *before*
                        // the socket is closed.
                        ssl_finish(temp->ssl);
                    }
#endif
                    if (sock_close(temp->sockctrl, errbuf,
                       PCAP_ERRBUF_SIZE) == -1)
                        status = -1;
                }

                /*
                 * Remove the host from the list of active
                 * hosts.
                 */
                if (prev)
                    prev->next = temp->next;
                else
                    activeHosts = temp->next;

                freeaddrinfo(addrinfo);

                free(temp);

                /* To avoid inconsistencies in the number of sock_init() */
                sock_cleanup();

                return status;
            }

            ai_next = ai_next->ai_next;
        }
        prev = temp;
        temp = temp->next;
    }

    if (addrinfo)
        freeaddrinfo(addrinfo);

    /* To avoid inconsistencies in the number of sock_init() */
    sock_cleanup();

    snprintf(errbuf, PCAP_ERRBUF_SIZE, "The host you want to close the active connection is not known");
    return -1;
}

void pcap_remoteact_cleanup(void)
{
#    ifdef HAVE_OPENSSL
    if (ssl_main)
    {
        // Finish using the SSL handle for the main active socket.
        // This must be done *before* the socket is closed.
        ssl_finish(ssl_main);
        ssl_main = NULL;
    }
#    endif

    /* Very dirty, but it works */
    if (sockmain)
    {
        // 关闭主套接字
        closesocket(sockmain);

        /* 为了避免在 sock_init() 的数量上出现不一致 */
        // 清理套接字
        sock_cleanup();
    }
}

这是一个函数的结束标记。


int pcap_remoteact_list(char *hostlist, char sep, int size, char *errbuf)

定义了一个名为pcap_remoteact_list的函数，接受四个参数：hostlist，sep，size和errbuf。


struct activehosts *temp;    /* temp var needed to scan the host list chain */

定义了一个名为temp的结构体指针变量，用于扫描主机列表链。


size_t len;
char hoststr[RPCAP_HOSTLIST_SIZE + 1];

定义了一个名为len的size_t类型变量和一个名为hoststr的字符数组，用于存储主机列表和主机字符串。


temp = activeHosts;

将activeHosts的值赋给temp。


len = 0;
*hostlist = 0;

将len的值设为0，将hostlist指向的位置的值设为0。


while (temp)
{
    /*int sock_getascii_addrport(const struct sockaddr_storage *sockaddr, char *address, int addrlen, char *port, int portlen, int flags, char *errbuf, int errbuflen) */

进入while循环，条件为temp不为空。


/* Get the numeric form of the name of the connecting host */
if (sock_getascii_addrport((struct sockaddr_storage *) &temp->host, hoststr,
    RPCAP_HOSTLIST_SIZE, NULL, 0, NI_NUMERICHOST, errbuf, PCAP_ERRBUF_SIZE) != -1)
    /*    if (getnameinfo( (struct sockaddr *) &temp->host, sizeof (struct sockaddr_storage), hoststr, */
    /*        RPCAP_HOSTLIST_SIZE, NULL, 0, NI_NUMERICHOST) ) */
{
    /*    sock_geterrmsg(errbuf, PCAP_ERRBUF_SIZE, */
    /*        "getnameinfo() failed");             */
    return -1;
}

获取连接主机的名称的数值形式。


len = len + strlen(hoststr) + 1 /* the separator */;

计算len的值，包括主机字符串的长度和一个分隔符。


if ((size < 0) || (len >= (size_t)size))
{
    snprintf(errbuf, PCAP_ERRBUF_SIZE, "The string you provided is not able to keep "
        "the hostnames for all the active connections");
    return -1;
}

如果size小于0或者len大于等于size，则返回-1。


pcap_strlcat(hostlist, hoststr, PCAP_ERRBUF_SIZE);
hostlist[len - 1] = sep;
hostlist[len] = 0;

将hoststr的内容拼接到hostlist上，并在末尾添加分隔符。


temp = temp->next;

将temp指向下一个节点。


return 0;

返回0。


/*
 * Receive the header of a message.
 */
static int rpcap_recv_msg_header(SOCKET sock, SSL *ssl, struct rpcap_header *header, char *errbuf)

定义了一个名为rpcap_recv_msg_header的静态函数，接受四个参数：sock，ssl，header和errbuf。


int nrecv;

定义了一个名为nrecv的整型变量。


nrecv = sock_recv(sock, ssl, (char *) header, sizeof(struct rpcap_header),
    SOCK_RECEIVEALL_YES|SOCK_EOF_IS_ERROR, errbuf,
    PCAP_ERRBUF_SIZE);
if (nrecv == -1)
{
    /* Network error. */
    return -1;
}

调用sock_recv函数接收消息头，如果接收失败则返回-1。
    # 将header结构体中的plen字段从网络字节顺序转换为主机字节顺序
    header->plen = ntohl(header->plen);
    # 返回0，表示函数执行成功
    return 0;
/*
 * 确保接收到的消息的协议版本与我们期望的版本一致。
 */
static int rpcap_check_msg_ver(SOCKET sock, SSL *ssl, uint8 expected_ver, struct rpcap_header *header, char *errbuf)
{
    /*
     * 服务器是否指定了我们协商的版本？
     */
    if (header->ver != expected_ver)
    {
        /*
         * 丢弃消息的剩余部分。
         */
        if (rpcap_discard(sock, ssl, header->plen, errbuf) == -1)
            return -1;

        /*
         * 告诉调用者这不是协商的版本。
         */
        if (errbuf != NULL)
        {
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "服务器发送了一个版本为 %u 的消息，而我们期望的是 %u",
                header->ver, expected_ver);
        }
        return -1;
    }
    return 0;
}

/*
 * 检查接收到的消息的消息类型，应该是期望的消息类型或者 RPCAP_MSG_ERROR。
 */
static int rpcap_check_msg_type(SOCKET sock, SSL *ssl, uint8 request_type, struct rpcap_header *header, uint16 *errcode, char *errbuf)
{
    const char *request_type_string;
    const char *msg_type_string;

    /*
     * 这是什么类型的消息？
     */
    if (header->type == RPCAP_MSG_ERROR)
    {
        /*
         * 服务器报告了一个错误。
         * 将该错误返回给调用者。
         */
        *errcode = ntohs(header->value);
        rpcap_msg_err(sock, ssl, header->plen, errbuf);
        return -1;
    }

    *errcode = 0;

    /*
     * 对于给定的请求类型值，期望的回复类型值是请求类型值与 RPCAP_MSG_IS_REPLY 进行 OR 运算。
     */
    if (header->type != (request_type | RPCAP_MSG_IS_REPLY))
*/
    {
        /*
         * 这不是我们发送的请求的回复。
         */

        /*
         * 丢弃消息的其余部分。
         */
        if (rpcap_discard(sock, ssl, header->plen, errbuf) == -1)
            return -1;

        /*
         * 告诉调用者。
         */
        request_type_string = rpcap_msg_type_string(request_type);
        msg_type_string = rpcap_msg_type_string(header->type);
        if (errbuf != NULL)
        {
            if (request_type_string == NULL)
            {
                /* 这不应该发生。 */
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "rpcap_check_msg_type called for request message with type %u",
                    request_type);
                return -1;
            }
            if (msg_type_string != NULL)
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "%s message received in response to a %s message",
                    msg_type_string, request_type_string);
            else
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "Message of unknown type %u message received in response to a %s request",
                    header->type, request_type_string);
        }
        return -1;
    }

    return 0;
}
/*
 * 接收并处理消息的头部。
 */
static int rpcap_process_msg_header(SOCKET sock, SSL *ssl, uint8 expected_ver, uint8 request_type, struct rpcap_header *header, char *errbuf)
{
    uint16 errcode;

    if (rpcap_recv_msg_header(sock, ssl, header, errbuf) == -1)
    {
        /* 网络错误。 */
        return -1;
    }

    /*
     * 服务器是否指定了我们协商的版本？
     */
    if (rpcap_check_msg_ver(sock, ssl, expected_ver, header, errbuf) == -1)
        return -1;

    /*
     * 检查消息类型。
     */
    return rpcap_check_msg_type(sock, ssl, request_type, header,
        &errcode, errbuf);
}

/*
 * 从消息中读取数据。
 * 如果我们尝试读取的数据超过了剩余的数据量，将错误消息放入errmsgbuf中并返回-2。
 * 否则，尝试读取数据，如果成功，从剩余的数据量中减去已读取的数据量。
 * 成功返回0，网络错误时记录消息并返回-1。
 */
static int rpcap_recv(SOCKET sock, SSL *ssl, void *buffer, size_t toread, uint32 *plen, char *errbuf)
{
    int nread;

    if (toread > *plen)
    {
        /* 服务器发送了一个错误的消息 */
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "消息负载太短");
        return -1;
    }
    nread = sock_recv(sock, ssl, buffer, toread,
        SOCK_RECEIVEALL_YES|SOCK_EOF_IS_ERROR, errbuf, PCAP_ERRBUF_SIZE);
    if (nread == -1)
    {
        return -1;
    }
    *plen -= nread;
    return 0;
}

/*
 * 处理RPCAP_MSG_ERROR消息。
 */
static void rpcap_msg_err(SOCKET sockctrl, SSL *ssl, uint32 plen, char *remote_errbuf)
{
    char errbuf[PCAP_ERRBUF_SIZE];

    if (plen >= PCAP_ERRBUF_SIZE)
        {
            /*
             * 如果消息太长，则尽可能多地读取到提供的缓冲区中，并丢弃其余部分。
             */
            // 如果从套接字接收数据失败
            if (sock_recv(sockctrl, ssl, remote_errbuf, PCAP_ERRBUF_SIZE - 1,
                SOCK_RECEIVEALL_YES|SOCK_EOF_IS_ERROR, errbuf,
                PCAP_ERRBUF_SIZE) == -1)
            {
                // 网络错误
                DIAG_OFF_FORMAT_TRUNCATION
                // 将错误消息写入 remote_errbuf
                snprintf(remote_errbuf, PCAP_ERRBUF_SIZE, "Read of error message from client failed: %s", errbuf);
                DIAG_ON_FORMAT_TRUNCATION
                // 返回
                return;
            }

            /*
             * 给它加上空字符
             */
            // 在末尾加上空字符
            remote_errbuf[PCAP_ERRBUF_SIZE - 1] = '\0';
        }
#ifdef _WIN32
        /*
         * 如果我们不在 UTF-8 模式下，将其转换为本地代码页。
         */
        if (!pcap_utf_8_mode)
            utf_8_to_acp_truncated(remote_errbuf);
#endif

        /*
         * 丢弃其余部分。
         */
        (void)rpcap_discard(sockctrl, ssl, plen - (PCAP_ERRBUF_SIZE - 1), remote_errbuf);
    }
    else if (plen == 0)
    {
        /* 空错误字符串。 */
        remote_errbuf[0] = '\0';
    }
    else
    {
        if (sock_recv(sockctrl, ssl, remote_errbuf, plen,
            SOCK_RECEIVEALL_YES|SOCK_EOF_IS_ERROR, errbuf,
            PCAP_ERRBUF_SIZE) == -1)
        {
            // 网络错误。
            DIAG_OFF_FORMAT_TRUNCATION
            snprintf(remote_errbuf, PCAP_ERRBUF_SIZE, "Read of error message from client failed: %s", errbuf);
            DIAG_ON_FORMAT_TRUNCATION
            return;
        }

        /*
         * 空字符结尾。
         */
        remote_errbuf[plen] = '\0';
    }
}

/*
 * 丢弃连接中的数据。
 * 主要用于丢弃错误大小的消息。
 * 成功返回 0，出现网络错误则记录消息并返回 -1。
 */
static int rpcap_discard(SOCKET sock, SSL *ssl, uint32 len, char *errbuf)
{
    if (len != 0)
    {
        if (sock_discard(sock, ssl, len, errbuf, PCAP_ERRBUF_SIZE) == -1)
        {
            // 网络错误。
            return -1;
        }
    }
    return 0;
}

/*
 * 读取字节到 pcap_t 的缓冲区，直到读取指定数量的字节或出现错误或中断指示。
 */
static int rpcap_read_packet_msg(struct pcap_rpcap const *rp, pcap_t *p, size_t size)
{
    u_char *bp;
    int cc;
    int bytes_read;

    bp = p->bp;
    cc = p->cc;

    /*
     * 循环直到我们获得请求的数据量或出现错误或中断。
     */
    while ((size_t)cc < size)
    {
        /*
         * 我们还没有完全读取数据包头部。
         * 读取剩下的部分，这可能是全部内容。
         */
        bytes_read = sock_recv(rp->rmt_sockdata, rp->data_ssl, bp, size - cc,
            SOCK_RECEIVEALL_NO|SOCK_EOF_IS_ERROR, p->errbuf,
            PCAP_ERRBUF_SIZE);

        if (bytes_read == -1)
        {
            /*
             * 网络错误。更新读取指针和字节数，并返回错误指示。
             */
            p->bp = bp;
            p->cc = cc;
            return -1;
        }
        if (bytes_read == -3)
        {
            /*
             * 接收被中断。更新读取指针和字节数，并返回中断指示。
             */
            p->bp = bp;
            p->cc = cc;
            return -3;
        }
        if (bytes_read == 0)
        {
            /*
             * EOF - 服务器终止了连接。
             * 更新读取指针和字节数，并返回错误指示。
             */
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "The server terminated the connection.");
            return -1;
        }
        bp += bytes_read;
        cc += bytes_read;
    }
    p->bp = bp;
    p->cc = cc;
    return 0;
# 闭合前面的函数定义
```