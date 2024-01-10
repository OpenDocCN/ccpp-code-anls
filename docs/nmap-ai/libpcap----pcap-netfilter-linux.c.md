# `nmap\libpcap\pcap-netfilter-linux.c`

```
/*
 * 版权声明
 * 版权所有（c）2011 Jakub Zawadzki
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *
 * 1. 源代码的重新分发必须保留上述版权声明、此条件列表和以下免责声明。
 * 2. 以二进制形式重新分发时，必须在提供的文档和/或其他材料中复制上述版权声明、此条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者提供，"按原样"提供，任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保，都是不被允许的。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "pcap-int.h"
#include "diag-control.h"

#ifdef NEED_STRERROR_H
#include "strerror.h"
#endif

#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#include <time.h>
#include <sys/time.h>
#include <netinet/in.h>
#include <linux/types.h>

#include <linux/netlink.h>
#include <linux/netfilter.h>
#include <linux/netfilter/nfnetlink.h>
#include <linux/netfilter/nfnetlink_log.h>
#include <linux/netfilter/nfnetlink_queue.h>

/* 注意：如果你的程序在调用 pcap_activate() 后降低权限，它将无法与 nfqueue 一起工作。
 *       我花了很长时间来调试 ;/
 *
 *       向 nfnetlink 套接字发送任何数据都需要 CAP_NET_ADMIN 权限，
 *       而在 nfqueue 中，我们需要在接收数据包后发送决定性回复。
 *
 *       在 tcpdump 中，你可以使用 -Z root 禁用降低权限
 */

#include "pcap-netfilter-linux.h"

#define HDR_LENGTH (NLMSG_LENGTH(NLMSG_ALIGN(sizeof(struct nfgenmsg))))

#define NFLOG_IFACE "nflog"
#define NFQUEUE_IFACE "nfqueue"

typedef enum { OTHER = -1, NFLOG, NFQUEUE } nftype_t;

/*
 * Linux netfilter 套接字捕获的私有数据。
 */
struct pcap_netfilter {
    u_int    packets_read;    /* 使用 recvfrom() 读取的数据包计数 */
    u_int   packets_nobufs; /* ENOBUFS 计数 */
};

static int nfqueue_send_verdict(const pcap_t *handle, uint16_t group_id, u_int32_t id, u_int32_t verdict);


static int
netfilter_read_linux(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user)
{
    struct pcap_netfilter *handlep = handle->priv;
    register u_char *bp, *ep;
    int count = 0;
    ssize_t len;

    /*
     * "pcap_breakloop()" 被调用了吗？
     */
    if (handle->break_loop) {
        /*
         * 是的 - 清除指示它已经被调用的标志，并返回 PCAP_ERROR_BREAK 以指示我们被告知中断循环。
         */
        handle->break_loop = 0;
        return PCAP_ERROR_BREAK;
    }
    len = handle->cc;
    # 如果接收缓冲区为空
    if (len == 0) {
        """
        缓冲区为空；重新填充缓冲区。

        我们忽略 EINTR，因为这可能只是由于信号的传递而导致的 - 如果信号应该中断循环，信号处理程序应该调用 pcap_breakloop() 来设置 handle->break_loop（我们在其他平台上也忽略它）。
        """
        do {
            # 从套接字接收数据到缓冲区
            len = recv(handle->fd, handle->buffer, handle->bufsize, 0);
            # 如果需要中断循环，则设置 handle->break_loop 并返回中断错误
            if (handle->break_loop) {
                handle->break_loop = 0;
                return PCAP_ERROR_BREAK;
            }
            # 如果错误码为 ENOBUFS，则增加无缓冲区的数据包计数
            if (errno == ENOBUFS)
                handlep->packets_nobufs++;
        } while ((len == -1) && (errno == EINTR || errno == ENOBUFS));

        # 如果接收数据出错
        if (len < 0) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE, errno, "Can't receive packet");
            return PCAP_ERROR;
        }

        # 将缓冲区指针指向接收到的数据
        bp = (unsigned char *)handle->buffer;
    } else
        # 否则，将缓冲区指针指向已有的数据
        bp = handle->bp;

    """
    遍历每个消息。

    这假设单个消息的缓冲区将有 <= INT_MAX 个数据包，因此消息计数不会溢出。
    """
    ep = bp + len;
    }

    # 重置捕获计数器并返回数据包数量
    handle->cc = 0;
    return count;
# 设置网络过滤器的数据链路类型
static int
netfilter_set_datalink(pcap_t *handle, int dlt)
{
    # 设置网络过滤器的数据链路类型为给定的值
    handle->linktype = dlt;
    # 返回 0 表示成功
    return 0;
}

# 获取 Linux 网络过滤器的统计信息
static int
netfilter_stats_linux(pcap_t *handle, struct pcap_stat *stats)
{
    # 获取网络过滤器的私有数据
    struct pcap_netfilter *handlep = handle->priv;

    # 将私有数据中的数据包接收数和丢弃数赋值给统计信息结构体
    stats->ps_recv = handlep->packets_read;
    stats->ps_drop = handlep->packets_nobufs;
    stats->ps_ifdrop = 0;
    # 返回 0 表示成功
    return 0;
}

# 在 Linux 上注入数据包
static int
netfilter_inject_linux(pcap_t *handle, const void *buf _U_, int size _U_)
{
    # 设置错误信息为不支持在 netfilter 设备上进行数据包注入
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "Packet injection is not supported on netfilter devices");
    # 返回 -1 表示失败
    return (-1);
}

# 发送配置消息到 netfilter
static int
netfilter_send_config_msg(const pcap_t *handle, uint16_t msg_type, int ack, u_int8_t family, u_int16_t res_id, const struct my_nfattr *mynfa)
{
    # 创建一个缓冲区用于存储消息
    char buf[1024] __attribute__ ((aligned));
    # 将缓冲区清零
    memset(buf, 0, sizeof(buf));

    # 设置消息头和消息体的指针
    struct nlmsghdr *nlh = (struct nlmsghdr *) buf;
    struct nfgenmsg *nfg = (struct nfgenmsg *) (buf + sizeof(struct nlmsghdr));

    # 设置网络地址结构
    struct sockaddr_nl snl;
    static unsigned int seq_id;

    # 如果序列号为 0，则设置为当前时间
    if (!seq_id)
DIAG_OFF_NARROWING
        seq_id = time(NULL);
DIAG_ON_NARROWING
    ++seq_id;

    # 设置消息头的长度、类型、标志、进程 ID 和序列号
    nlh->nlmsg_len = NLMSG_LENGTH(sizeof(struct nfgenmsg));
    nlh->nlmsg_type = msg_type;
    nlh->nlmsg_flags = NLM_F_REQUEST | (ack ? NLM_F_ACK : 0);
    nlh->nlmsg_pid = 0;    # 发送到内核
    nlh->nlmsg_seq = seq_id;

    # 设置消息体的族、版本和资源 ID
    nfg->nfgen_family = family;
    nfg->version = NFNETLINK_V0;
    nfg->res_id = htons(res_id);

    # 如果有自定义属性，则设置自定义属性
    if (mynfa) {
        struct nfattr *nfa = (struct nfattr *) (buf + NLMSG_ALIGN(nlh->nlmsg_len));

        nfa->nfa_type = mynfa->nfa_type;
        nfa->nfa_len = NFA_LENGTH(mynfa->nfa_len);
        memcpy(NFA_DATA(nfa), mynfa->data, mynfa->nfa_len);
        nlh->nlmsg_len = NLMSG_ALIGN(nlh->nlmsg_len) + NFA_ALIGN(nfa->nfa_len);
    }

    # 设置网络地址结构的族为 AF_NETLINK
    memset(&snl, 0, sizeof(snl));
    snl.nl_family = AF_NETLINK;
}
    # 发送消息到指定的套接字
    if (sendto(handle->fd, nlh, nlh->nlmsg_len, 0, (struct sockaddr *) &snl, sizeof(snl)) == -1)
        return -1;

    # 如果没有收到确认消息，直接返回0
    if (!ack)
        return 0;

    /* 等待回复的循环 */
    do {
        socklen_t addrlen = sizeof(snl);
        int len;

        /* 忽略中断系统调用错误 */
        do {
            /*
             * 缓冲区大小不会超出int的范围
             */
            len = (int)recvfrom(handle->fd, buf, sizeof(buf), 0, (struct sockaddr *) &snl, &addrlen);
        } while ((len == -1) && (errno == EINTR));

        # 如果接收到的数据长度小于等于0，直接返回长度
        if (len <= 0)
            return len;

        # 如果地址长度不等于snl的大小，或者snl的网络类型不是AF_NETLINK，设置错误码并返回-1
        if (addrlen != sizeof(snl) || snl.nl_family != AF_NETLINK) {
            errno = EINVAL;
            return -1;
        }

        # 将接收到的数据转换为nlmsghdr结构体
        nlh = (struct nlmsghdr *) buf;
        # 如果不是来自内核或者序列号不匹配，继续下一次循环
        if (snl.nl_pid != 0 || seq_id != nlh->nlmsg_seq)    
            continue;

        # 循环处理接收到的消息
        while ((u_int)len >= NLMSG_SPACE(0) && NLMSG_OK(nlh, (u_int)len)) {
            # 如果消息类型是NLMSG_ERROR，或者消息类型是NLMSG_DONE并且消息标志包含NLM_F_MULTI
            if (nlh->nlmsg_type == NLMSG_ERROR || (nlh->nlmsg_type == NLMSG_DONE && nlh->nlmsg_flags & NLM_F_MULTI)) {
                # 如果消息长度小于nlmsgerr结构体的大小，设置错误码并返回-1
                if (nlh->nlmsg_len < NLMSG_ALIGN(sizeof(struct nlmsgerr))) {
                    errno = EBADMSG;
                    return -1;
                }
                # 将nlmsgerr结构体中的错误码取反，并返回结果
                errno = -(*((int *)NLMSG_DATA(nlh)));
                return (errno == 0) ? 0 : -1;
            }
            # 移动到下一个消息
            nlh = NLMSG_NEXT(nlh, len);
        }
    } while (1);

    # 永远不会执行到这里，返回-1
    return -1; 
    # 发送配置消息到 NFLOG，包括协议族、组 ID 和自定义属性
    static int
    nflog_send_config_msg(const pcap_t *handle, uint8_t family, u_int16_t group_id, const struct my_nfattr *mynfa)
    {
        # 调用 netfilter_send_config_msg 函数，发送配置消息到 NFLOG
        return netfilter_send_config_msg(handle, (NFNL_SUBSYS_ULOG << 8) | NFULNL_MSG_CONFIG, 1, family, group_id, mynfa);
    }

    # 发送配置命令到 NFLOG，包括组 ID、命令和协议族
    static int
    nflog_send_config_cmd(const pcap_t *handle, uint16_t group_id, u_int8_t cmd, u_int8_t family)
    {
        # 创建 nfulnl_msg_config_cmd 结构体
        struct nfulnl_msg_config_cmd msg;
        # 创建 my_nfattr 结构体
        struct my_nfattr nfa;

        # 设置命令
        msg.command = cmd;

        # 设置自定义属性的数据、类型和长度
        nfa.data = &msg;
        nfa.nfa_type = NFULA_CFG_CMD;
        nfa.nfa_len = sizeof(msg);

        # 调用 nflog_send_config_msg 函数，发送配置消息到 NFLOG
        return nflog_send_config_msg(handle, family, group_id, &nfa);
    }

    # 发送配置模式到 NFLOG，包括组 ID、拷贝模式和拷贝范围
    static int
    nflog_send_config_mode(const pcap_t *handle, uint16_t group_id, u_int8_t copy_mode, u_int32_t copy_range)
    {
        # 创建 nfulnl_msg_config_mode 结构体
        struct nfulnl_msg_config_mode msg;
        # 创建 my_nfattr 结构体
        struct my_nfattr nfa;

        # 设置拷贝范围和拷贝模式
        msg.copy_range = htonl(copy_range);
        msg.copy_mode = copy_mode;

        # 设置自定义属性的数据、类型和长度
        nfa.data = &msg;
        nfa.nfa_type = NFULA_CFG_MODE;
        nfa.nfa_len = sizeof(msg);

        # 调用 nflog_send_config_msg 函数，发送配置消息到 NFLOG
        return nflog_send_config_msg(handle, AF_UNSPEC, group_id, &nfa);
    }

    # 发送决定消息到 NFQUEUE，包括组 ID、ID 和决定值
    static int
    nfqueue_send_verdict(const pcap_t *handle, uint16_t group_id, u_int32_t id, u_int32_t verdict)
    {
        # 创建 nfqnl_msg_verdict_hdr 结构体
        struct nfqnl_msg_verdict_hdr msg;
        # 创建 my_nfattr 结构体
        struct my_nfattr nfa;

        # 设置 ID 和决定值
        msg.id = htonl(id);
        msg.verdict = htonl(verdict);

        # 设置自定义属性的数据、类型和长度
        nfa.data = &msg;
        nfa.nfa_type = NFQA_VERDICT_HDR;
        nfa.nfa_len = sizeof(msg);

        # 调用 netfilter_send_config_msg 函数，发送配置消息到 NFQUEUE
        return netfilter_send_config_msg(handle, (NFNL_SUBSYS_QUEUE << 8) | NFQNL_MSG_VERDICT, 0, AF_UNSPEC, group_id, &nfa);
    }

    # 发送配置消息到 NFQUEUE，包括协议族、组 ID 和自定义属性
    static int
    nfqueue_send_config_msg(const pcap_t *handle, uint8_t family, u_int16_t group_id, const struct my_nfattr *mynfa)
    {
        # 调用 netfilter_send_config_msg 函数，发送配置消息到 NFQUEUE
        return netfilter_send_config_msg(handle, (NFNL_SUBSYS_QUEUE << 8) | NFQNL_MSG_CONFIG, 1, family, group_id, mynfa);
    }

    # 发送配置命令到 NFQUEUE，包括组 ID、命令和协议族
    static int
    nfqueue_send_config_cmd(const pcap_t *handle, uint16_t group_id, u_int8_t cmd, u_int16_t pf)
    {
        # 创建 nfqnl_msg_config_cmd 结构体
        struct nfqnl_msg_config_cmd msg;
        # 创建 my_nfattr 结构体
        struct my_nfattr nfa;

        # 设置命令和协议族
        msg.command = cmd;
        msg.pf = htons(pf);

        # 设置自定义属性的数据、类型和长度
        nfa.data = &msg;
        nfa.nfa_type = NFQA_CFG_CMD;
    # 设置nfa.nfa_len为消息的大小
    nfa.nfa_len = sizeof(msg);
    
    # 发送配置消息给nfqueue，指定地址族为AF_UNSPEC，组ID为group_id，消息为nfa
    return nfqueue_send_config_msg(handle, AF_UNSPEC, group_id, &nfa);
static int
nfqueue_send_config_mode(const pcap_t *handle, uint16_t group_id, u_int8_t copy_mode, u_int32_t copy_range)
{
    // 定义 nfqnl_msg_config_params 结构体变量 msg
    struct nfqnl_msg_config_params msg;
    // 定义 my_nfattr 结构体变量 nfa
    struct my_nfattr nfa;

    // 设置 msg 的 copy_range 字段为网络字节序的 copy_range
    msg.copy_range = htonl(copy_range);
    // 设置 msg 的 copy_mode 字段为 copy_mode
    msg.copy_mode = copy_mode;

    // 设置 nfa 的 data 字段为 msg 的地址
    nfa.data = &msg;
    // 设置 nfa 的 nfa_type 字段为 NFQA_CFG_PARAMS
    nfa.nfa_type = NFQA_CFG_PARAMS;
    // 设置 nfa 的 nfa_len 字段为 msg 的大小
    nfa.nfa_len = sizeof(msg);

    // 调用 nfqueue_send_config_msg 函数，发送配置消息
    return nfqueue_send_config_msg(handle, AF_UNSPEC, group_id, &nfa);
}

static int
netfilter_activate(pcap_t* handle)
{
    // 获取设备名称
    const char *dev = handle->opt.device;
    // 定义存储 netfilter 组 ID 的数组
    unsigned short groups[32];
    // 初始化组计数器
    int group_count = 0;
    // 初始化 netfilter 类型为 OTHER
    nftype_t type = OTHER;
    // 初始化循环变量 i
    int i;

    // 如果设备名称以 NFLOG_IFACE 开头
    if (strncmp(dev, NFLOG_IFACE, strlen(NFLOG_IFACE)) == 0) {
        // 截取设备名称
        dev += strlen(NFLOG_IFACE);
        // 设置 netfilter 类型为 NFLOG
        type = NFLOG;

    // 如果设备名称以 NFQUEUE_IFACE 开头
    } else if (strncmp(dev, NFQUEUE_IFACE, strlen(NFQUEUE_IFACE)) == 0) {
        // 截取设备名称
        dev += strlen(NFQUEUE_IFACE);
        // 设置 netfilter 类型为 NFQUEUE
        type = NFQUEUE;
    }

    // 如果 netfilter 类型不为 OTHER 并且设备名称以 : 开头
    if (type != OTHER && *dev == ':') {
        // 截取设备名称
        dev++;
        // 遍历设备名称
        while (*dev) {
            // 定义组 ID 和结束指针
            long int group_id;
            char *end_dev;

            // 如果组计数器已经达到 32
            if (group_count == 32) {
                // 设置错误信息并返回错误
                snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                        "Maximum 32 netfilter groups! dev: %s",
                        handle->opt.device);
                return PCAP_ERROR;
            }

            // 将设备名称转换为长整型的组 ID
            group_id = strtol(dev, &end_dev, 0);
            // 如果转换成功
            if (end_dev != dev) {
                // 如果组 ID 超出范围
                if (group_id < 0 || group_id > 65535) {
                    // 设置错误信息并返回错误
                    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                            "Netfilter group range from 0 to 65535 (got %ld)",
                            group_id);
                    return PCAP_ERROR;
                }

                // 将组 ID 存入数组中，增加组计数器
                groups[group_count++] = (unsigned short) group_id;
                dev = end_dev;
            }
            // 如果设备名称不是逗号，结束循环
            if (*dev != ',')
                break;
            dev++;
        }
    }
}
    // 如果类型为其他或者设备名称不为空，则设置错误信息并返回错误代码
    if (type == OTHER || *dev) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Can't get netfilter group(s) index from %s",
                handle->opt.device);
        return PCAP_ERROR;
    }

    // 如果没有组，添加默认组：0
    if (!group_count) {
        groups[0] = 0;
        group_count = 1;
    }

    /*
     * 将负的快照值（无效）、快照值为0（未指定）或者大于正常最大值的值，转换为允许的最大值
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加MAXIMUM_SNAPLEN
     */
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;

    // 初始化 pcap 结构的一些组件
    handle->bufsize = 128 + handle->snapshot;
    handle->offset = 0;
    handle->read_op = netfilter_read_linux;
    handle->inject_op = netfilter_inject_linux;
    handle->setfilter_op = install_bpf_program; /* no kernel filtering */
    handle->setdirection_op = NULL;
    handle->set_datalink_op = netfilter_set_datalink;
    handle->getnonblock_op = pcap_getnonblock_fd;
    handle->setnonblock_op = pcap_setnonblock_fd;
    handle->stats_op = netfilter_stats_linux;

    // 创建 netlink 套接字
    handle->fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_NETFILTER);
    if (handle->fd < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't create raw socket");
        return PCAP_ERROR;
    }

    // 如果类型为 NFLOG，则设置链路类型为 DLT_NFLOG，并分配内存给 dlt_list
    if (type == NFLOG) {
        handle->linktype = DLT_NFLOG;
        handle->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
        if (handle->dlt_list != NULL) {
            handle->dlt_list[0] = DLT_NFLOG;
            handle->dlt_list[1] = DLT_IPV4;
            handle->dlt_count = 2;
        }

    } else
        // 否则，设置链路类型为 DLT_IPV4
        handle->linktype = DLT_IPV4;

    // 分配内存给 buffer
    handle->buffer = malloc(handle->bufsize);
    // 如果句柄的缓冲区为空
    if (!handle->buffer) {
        // 格式化错误消息，用于无法分配转储缓冲区的情况
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't allocate dump buffer");
        // 转到关闭失败的标签
        goto close_fail;
    }

    // 如果类型是 NFLOG
    if (type == NFLOG) {
        // 如果发送配置命令失败
        if (nflog_send_config_cmd(handle, 0, NFULNL_CFG_CMD_PF_UNBIND, AF_INET) < 0) {
            // 格式化错误消息，用于 NFULNL_CFG_CMD_PF_UNBIND 失败的情况
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "NFULNL_CFG_CMD_PF_UNBIND");
            // 转到关闭失败的标签
            goto close_fail;
        }

        // 如果发送配置命令失败
        if (nflog_send_config_cmd(handle, 0, NFULNL_CFG_CMD_PF_BIND, AF_INET) < 0) {
            // 格式化错误消息，用于 NFULNL_CFG_CMD_PF_BIND 失败的情况
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno, "NFULNL_CFG_CMD_PF_BIND");
            // 转到关闭失败的标签
            goto close_fail;
        }

        // 将套接字绑定到 nflog 组
        for (i = 0; i < group_count; i++) {
            // 如果发送配置命令失败
            if (nflog_send_config_cmd(handle, groups[i], NFULNL_CFG_CMD_BIND, AF_UNSPEC) < 0) {
                // 格式化错误消息，用于无法监听组索引的情况
                pcap_fmt_errmsg_for_errno(handle->errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "Can't listen on group index");
                // 转到关闭失败的标签
                goto close_fail;
            }

            // 如果发送配置模式失败
            if (nflog_send_config_mode(handle, groups[i], NFULNL_COPY_PACKET, handle->snapshot) < 0) {
                // 格式化错误消息，用于 NFULNL_COPY_PACKET 失败的情况
                pcap_fmt_errmsg_for_errno(handle->errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "NFULNL_COPY_PACKET");
                // 转到关闭失败的标签
                goto close_fail;
            }
        }
    }
    } else {
        // 如果不是在混杂模式下
        // 解绑定 IPv4 地址族
        if (nfqueue_send_config_cmd(handle, 0, NFQNL_CFG_CMD_PF_UNBIND, AF_INET) < 0) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno, "NFQNL_CFG_CMD_PF_UNBIND");
            // 跳转到关闭失败的标签
            goto close_fail;
        }

        // 绑定 IPv4 地址族
        if (nfqueue_send_config_cmd(handle, 0, NFQNL_CFG_CMD_PF_BIND, AF_INET) < 0) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno, "NFQNL_CFG_CMD_PF_BIND");
            // 跳转到关闭失败的标签
            goto close_fail;
        }

        /* 将套接字绑定到 nfqueue 组 */
        for (i = 0; i < group_count; i++) {
            if (nfqueue_send_config_cmd(handle, groups[i], NFQNL_CFG_CMD_BIND, AF_UNSPEC) < 0) {
                // 格式化错误消息
                pcap_fmt_errmsg_for_errno(handle->errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "Can't listen on group index");
                // 跳转到关闭失败的标签
                goto close_fail;
            }

            if (nfqueue_send_config_mode(handle, groups[i], NFQNL_COPY_PACKET, handle->snapshot) < 0) {
                // 格式化错误消息
                pcap_fmt_errmsg_for_errno(handle->errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "NFQNL_COPY_PACKET");
                // 跳转到关闭失败的标签
                goto close_fail;
            }
        }
    }

    // 如果是混杂模式
    if (handle->opt.rfmon) {
        /*
         * 监控模式不适用于 netfilter 设备。
         */
        // 清理并返回监控模式不支持的错误
        pcap_cleanup_live_common(handle);
        return PCAP_ERROR_RFMON_NOTSUP;
    }

    // 如果设置了缓冲区大小
    if (handle->opt.buffer_size != 0) {
        /*
         * 将套接字缓冲区大小设置为指定值。
         */
        // 设置套接字选项
        if (setsockopt(handle->fd, SOL_SOCKET, SO_RCVBUF, &handle->opt.buffer_size, sizeof(handle->opt.buffer_size)) == -1) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno, "SO_RCVBUF");
            // 跳转到关闭失败的标签
            goto close_fail;
        }
    }

    // 设置可选择的文件描述符为套接字文件描述符
    handle->selectable_fd = handle->fd;
    // 返回成功
    return 0;
    // 关闭失败时的处理
close_fail:
    // 清理 pcap_live_common 结构
    pcap_cleanup_live_common(handle);
    // 返回错误代码
    return PCAP_ERROR;
}

// 创建 netfilter 设备
pcap_t *
netfilter_create(const char *device, char *ebuf, int *is_ours)
{
    const char *cp;
    pcap_t *p;

    /* 检查设备名是否以 NFLOG_IFACE 或 NFQUEUE_IFACE 开头 */
    cp = strrchr(device, '/');
    if (cp == NULL)
        cp = device;

    /* 如果以 NFLOG_IFACE 或 NFQUEUE_IFACE 开头，则移动指针 */
    if (strncmp(cp, NFLOG_IFACE, sizeof NFLOG_IFACE - 1) == 0)
        cp += sizeof NFLOG_IFACE - 1;
    else if (strncmp(cp, NFQUEUE_IFACE, sizeof NFQUEUE_IFACE - 1) == 0)
        cp += sizeof NFQUEUE_IFACE - 1;
    else {
        /* 如果不是以 NFLOG_IFACE 或 NFQUEUE_IFACE 开头，则标记为非 netfilter 设备 */
        *is_ours = 0;
        return NULL;
    }

    /*
     * 是 netfilter 设备 - 检查是否以冒号结尾
     */
    if (*cp != ':' && *cp != '\0') {
        /* 如果不是以冒号结尾，则标记为非 netfilter 设备 */
        *is_ours = 0;
        return NULL;
    }

    /* 符合条件，标记为 netfilter 设备 */
    *is_ours = 1;

    // 创建 pcap_netfilter 结构
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_netfilter);
    if (p == NULL)
        return (NULL);

    // 设置激活操作为 netfilter_activate
    p->activate_op = netfilter_activate;
    return (p);
}

// 查找所有 netfilter 设备
int
netfilter_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    int sock;

    // 创建 AF_NETLINK 套接字
    sock = socket(AF_NETLINK, SOCK_RAW, NETLINK_NETFILTER);
    if (sock < 0) {
        /* 如果不支持 netlink，则不是致命错误 */
        if (errno == EAFNOSUPPORT || errno == EPROTONOSUPPORT)
            return 0;
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(err_str, PCAP_ERRBUF_SIZE,
            errno, "Can't open netlink socket");
        return -1;
    }
    // 关闭套接字
    close(sock);

    /*
     * "connected" vs. "disconnected" 的概念不适用
     * XXX - "up" 和 "running" 呢？
     */
    // 添加 NFLOG_IFACE 设备到设备列表
    if (add_dev(devlistp, NFLOG_IFACE,
        PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
        "Linux netfilter log (NFLOG) interface", err_str) == NULL)
        return -1;
}
    # 如果添加 NFQUEUE 接口到设备列表失败，则返回 -1
    if (add_dev(devlistp, NFQUEUE_IFACE,
        PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
        "Linux netfilter queue (NFQUEUE) interface", err_str) == NULL)
        return -1;
    # 添加成功则返回 0
    return 0;
# 闭合前面的函数定义
```