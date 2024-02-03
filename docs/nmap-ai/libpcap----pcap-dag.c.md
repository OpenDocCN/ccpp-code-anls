# `nmap\libpcap\pcap-dag.c`

```cpp
/*
 * pcap-dag.c: Endace DAG 卡的数据包捕获接口。
 *
 * 作者：Richard Littin, Sean Irvine ({richard,sean}@reeltwo.com)
 * 修改：Jesper Peterson
 *      Koryn Grant
 *      Stephen Donnelly <stephen.donnelly@endace.com>
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>            /* 可选获取 BSD 定义 */

#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include "pcap-int.h"

#include <netinet/in.h>
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

struct mbuf;        /* 在一些平台上声明 <net/if.h> 中的结构以消除编译器警告 */
struct rtentry;        /* 在一些平台上声明 <net/if.h> 中的结构以消除编译器警告 */
#include <net/if.h>

#include "dagnew.h"
#include "dagapi.h"
#include "dagpci.h"
#include "dag_config_api.h"

#include "pcap-dag.h"

/*
 * DAG 设备的名称以 "dag" 开头，后面跟着一个从 0 到 DAG_MAX_BOARDS 的数字，然后可选地是一个冒号和从 0 到 DAG_STREAM_MAX 的流编号。
 */
#ifndef DAG_MAX_BOARDS
#define DAG_MAX_BOARDS 32
#endif


#ifndef ERF_TYPE_AAL5
#define ERF_TYPE_AAL5               4
#endif

#ifndef ERF_TYPE_MC_HDLC
#define ERF_TYPE_MC_HDLC            5
#endif

#ifndef ERF_TYPE_MC_RAW
#define ERF_TYPE_MC_RAW             6
#endif

#ifndef ERF_TYPE_MC_ATM
#define ERF_TYPE_MC_ATM             7
#endif

#ifndef ERF_TYPE_MC_RAW_CHANNEL
#define ERF_TYPE_MC_RAW_CHANNEL     8
#endif

#ifndef ERF_TYPE_MC_AAL5
#define ERF_TYPE_MC_AAL5            9
#endif

#ifndef ERF_TYPE_COLOR_HDLC_POS
#define ERF_TYPE_COLOR_HDLC_POS     10
#endif

#ifndef ERF_TYPE_COLOR_ETH
#define ERF_TYPE_COLOR_ETH          11
#endif

#ifndef ERF_TYPE_MC_AAL2
#define ERF_TYPE_MC_AAL2            12
#endif

#ifndef ERF_TYPE_IP_COUNTER
#define ERF_TYPE_IP_COUNTER         13
#endif

#ifndef ERF_TYPE_TCP_FLOW_COUNTER
#define ERF_TYPE_TCP_FLOW_COUNTER   14
#endif

#ifndef ERF_TYPE_DSM_COLOR_HDLC_POS
#define ERF_TYPE_DSM_COLOR_HDLC_POS 15
#endif
#ifndef ERF_TYPE_DSM_COLOR_ETH
#define ERF_TYPE_DSM_COLOR_ETH      16
#endif
// 如果未定义 ERF_TYPE_DSM_COLOR_ETH，则定义为 16

#ifndef ERF_TYPE_COLOR_MC_HDLC_POS
#define ERF_TYPE_COLOR_MC_HDLC_POS  17
#endif
// 如果未定义 ERF_TYPE_COLOR_MC_HDLC_POS，则定义为 17

#ifndef ERF_TYPE_AAL2
#define ERF_TYPE_AAL2               18
#endif
// 如果未定义 ERF_TYPE_AAL2，则定义为 18

#ifndef ERF_TYPE_COLOR_HASH_POS
#define ERF_TYPE_COLOR_HASH_POS     19
#endif
// 如果未定义 ERF_TYPE_COLOR_HASH_POS，则定义为 19

#ifndef ERF_TYPE_COLOR_HASH_ETH
#define ERF_TYPE_COLOR_HASH_ETH     20
#endif
// 如果未定义 ERF_TYPE_COLOR_HASH_ETH，则定义为 20

#ifndef ERF_TYPE_INFINIBAND
#define ERF_TYPE_INFINIBAND         21
#endif
// 如果未定义 ERF_TYPE_INFINIBAND，则定义为 21

#ifndef ERF_TYPE_IPV4
#define ERF_TYPE_IPV4               22
#endif
// 如果未定义 ERF_TYPE_IPV4，则定义为 22

#ifndef ERF_TYPE_IPV6
#define ERF_TYPE_IPV6               23
#endif
// 如果未定义 ERF_TYPE_IPV6，则定义为 23

#ifndef ERF_TYPE_RAW_LINK
#define ERF_TYPE_RAW_LINK           24
#endif
// 如果未定义 ERF_TYPE_RAW_LINK，则定义为 24

#ifndef ERF_TYPE_INFINIBAND_LINK
#define ERF_TYPE_INFINIBAND_LINK    25
#endif
// 如果未定义 ERF_TYPE_INFINIBAND_LINK，则定义为 25

#ifndef ERF_TYPE_META
#define ERF_TYPE_META               27
#endif
// 如果未定义 ERF_TYPE_META，则定义为 27

#ifndef ERF_TYPE_PAD
#define ERF_TYPE_PAD                48
#endif
// 如果未定义 ERF_TYPE_PAD，则定义为 48

#define ATM_CELL_SIZE        52
// 定义 ATM_CELL_SIZE 为 52

#define ATM_HDR_SIZE        4
// 定义 ATM_HDR_SIZE 为 4

/*
 * A header containing additional MTP information.
 */
#define MTP2_SENT_OFFSET        0    /* 1 byte */
#define MTP2_ANNEX_A_USED_OFFSET    1    /* 1 byte */
#define MTP2_LINK_NUMBER_OFFSET        2    /* 2 bytes */
#define MTP2_HDR_LEN            4    /* length of the header */
// 定义 MTP2_SENT_OFFSET 为 0，MTP2_ANNEX_A_USED_OFFSET 为 1，MTP2_LINK_NUMBER_OFFSET 为 2，MTP2_HDR_LEN 为 4

#define MTP2_ANNEX_A_NOT_USED      0
#define MTP2_ANNEX_A_USED          1
#define MTP2_ANNEX_A_USED_UNKNOWN  2
// 定义 MTP2_ANNEX_A_NOT_USED 为 0，MTP2_ANNEX_A_USED 为 1，MTP2_ANNEX_A_USED_UNKNOWN 为 2

/* SunATM pseudo header */
struct sunatm_hdr {
    unsigned char    flags;        /* destination and traffic type */
    unsigned char    vpi;        /* VPI */
    unsigned short    vci;        /* VCI */
};
// 定义结构体 sunatm_hdr，包含 flags、vpi、vci 三个成员变量

/*
 * Private data for capturing on DAG devices.
 */
struct pcap_dag {
    struct pcap_stat stat;
    u_char    *dag_mem_bottom;    /* DAG card current memory bottom pointer */
    u_char    *dag_mem_top;    /* DAG card current memory top pointer */
    int    dag_fcs_bits;    /* Number of checksum bits from link layer */
    int    dag_flags;    /* Flags */
    int    dag_stream;    /* DAG stream number */
};
// 定义结构体 pcap_dag，包含 stat、dag_mem_bottom、dag_mem_top、dag_fcs_bits、dag_flags、dag_stream 六个成员变量
    # 用于指定给 pcap_open_live 函数的超时时间
    int    dag_timeout;    /* timeout specified to pcap_open_live.
                 * Same as in linux above, introduce
                 * generally? */
    # DAG 配置/状态 API 的卡引用
    dag_card_ref_t dag_ref; /* DAG Configuration/Status API card reference */
    # DAG CSAPI 根组件
    dag_component_t dag_root;    /* DAG CSAPI Root component */
    # DAG 流丢弃属性的句柄，如果可用
    attr_uuid_t drop_attr;  /* DAG Stream Drop Attribute handle, if available */
    # 调用者在事件循环中必须使用的超时时间
    struct timeval required_select_timeout;
                /* Timeout caller must use in event loops */
};

// 定义结构体 pcap_dag_node，包含指向下一个节点的指针、pcap_t 结构体指针和进程 ID
typedef struct pcap_dag_node {
    struct pcap_dag_node *next;
    pcap_t *p;
    pid_t pid;
} pcap_dag_node_t;

// 初始化 pcap_dags 为 NULL，atexit_handler_installed 为 0，endian_test_word 为 0x0100
static pcap_dag_node_t *pcap_dags = NULL;
static int atexit_handler_installed = 0;
static const unsigned short endian_test_word = 0x0100;

// 定义宏 IS_BIGENDIAN()，用于判断系统是否为大端序
#define IS_BIGENDIAN() (*((unsigned char *)&endian_test_word))

// 定义最大数据包大小为 65536
#define MAX_DAG_PACKET 65536

// 定义静态数组 TempPkt，大小为 MAX_DAG_PACKET
static unsigned char TempPkt[MAX_DAG_PACKET];

// 如果没有定义 HAVE_DAG_LARGE_STREAMS_API，则定义一系列宏和类型别名
#ifndef HAVE_DAG_LARGE_STREAMS_API
#define dag_attach_stream64(a, b, c, d) dag_attach_stream(a, b, c, d)
#define dag_get_stream_poll64(a, b, c, d, e) dag_get_stream_poll(a, b, c, d, e)
#define dag_set_stream_poll64(a, b, c, d, e) dag_set_stream_poll(a, b, c, d, e)
#define dag_size_t uint32_t
#endif

// 声明静态函数 dag_stats、dag_set_datalink、dag_get_datalink、dag_setnonblock
static int dag_stats(pcap_t *p, struct pcap_stat *ps);
static int dag_set_datalink(pcap_t *p, int dlt);
static int dag_get_datalink(pcap_t *p);
static int dag_setnonblock(pcap_t *p, int nonblock);

// 定义函数 delete_pcap_dag，用于删除指定的 pcap_t 结构体
static void
delete_pcap_dag(const pcap_t *p)
{
    pcap_dag_node_t *curr = NULL, *prev = NULL;

    // 遍历 pcap_dags 链表，找到要删除的节点
    for (prev = NULL, curr = pcap_dags; curr != NULL && curr->p != p; prev = curr, curr = curr->next) {
        /* empty */
    }

    // 如果找到要删除的节点，则删除它
    if (curr != NULL && curr->p == p) {
        if (prev != NULL) {
            prev->next = curr->next;
        } else {
            pcap_dags = curr->next;
        }
    }
}

// 定义函数 dag_platform_cleanup，用于优雅地关闭 DAG 卡、释放 pcap_t 结构体中的动态内存和关闭 DAG 卡的文件描述符
static void
dag_platform_cleanup(pcap_t *p)
{
    struct pcap_dag *pd = p->priv;

    // 停止 DAG 卡的数据流
    if(dag_stop_stream(p->fd, pd->dag_stream) < 0)
        fprintf(stderr,"dag_stop_stream: %s\n", strerror(errno));

    // 分离 DAG 卡的数据流
    if(dag_detach_stream(p->fd, pd->dag_stream) < 0)
        fprintf(stderr,"dag_detach_stream: %s\n", strerror(errno));
}
    # 如果pd->dag_ref不为空
    if(pd->dag_ref != NULL) {
        # 释放pd->dag_ref所指向的DAG配置
        dag_config_dispose(pd->dag_ref);
        """
         * 注意：我们不需要调用close(p->fd)或dag_close(p->fd)，
         * 因为dag_config_dispose(pd->dag_ref)会执行这些操作。
         *
         * 将p->fd设置为-1，以确保不会执行这些操作。
         """
        # 将p->fd设置为-1
        p->fd = -1;
        # 将pd->dag_ref设置为NULL
        pd->dag_ref = NULL;
    }
    # 删除pcap_dag对象
    delete_pcap_dag(p);
    # 清理pcap_live_common对象
    pcap_cleanup_live_common(p);
# 退出时的处理函数，用于清理资源
static void
atexit_handler(void)
{
    # 循环处理所有的 pcap_dags
    while (pcap_dags != NULL) {
        # 如果当前进程的 PID 与 pcap_dags 的 PID 相同
        if (pcap_dags->pid == getpid()) {
            # 如果 pcap_dags 的指针不为空，则调用 dag_platform_cleanup 清理资源
            if (pcap_dags->p != NULL)
                dag_platform_cleanup(pcap_dags->p);
        } else {
            # 否则，删除 pcap_dags 的 p
            delete_pcap_dag(pcap_dags->p);
        }
    }
}

# 创建新的 pcap_dag
static int
new_pcap_dag(pcap_t *p)
{
    # 分配内存给 pcap_dag_node_t 结构体
    pcap_dag_node_t *node = NULL;

    if ((node = malloc(sizeof(pcap_dag_node_t))) == NULL) {
        return -1;
    }

    # 如果 atexit_handler_installed 为假，则注册 atexit_handler 函数
    if (!atexit_handler_installed) {
        atexit(atexit_handler);
        atexit_handler_installed = 1;
    }

    # 将新创建的 node 添加到 pcap_dags 链表的头部
    node->next = pcap_dags;
    node->p = p;
    node->pid = getpid();

    pcap_dags = node;

    return 0;
}

# 计算 ERF 扩展头的数量
static unsigned int
dag_erf_ext_header_count(const uint8_t *erf, size_t len)
{
    uint32_t hdr_num = 0;
    uint8_t  hdr_type;

    # 基本的合法性检查
    if ( erf == NULL )
        return 0;
    if ( len < 16 )
        return 0;

    # 检查是否有任何扩展头
    if ( (erf[8] & 0x80) == 0x00 )
        return 0;

    # 循环处理扩展头
    do {
        # 检查是否有足够的字节
        if ( len < (24 + (hdr_num * 8)) )
            return hdr_num;

        # 获取头类型
        hdr_type = erf[(16 + (hdr_num * 8))];
        hdr_num++;

    } while ( hdr_type & 0x80 );

    return hdr_num;
}

# 从捕获流中读取最多 max_packets 个数据包，并对每个数据包调用回调函数
static int
dag_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    # 获取 pcap_dag 结构体
    struct pcap_dag *pd = p->priv;
    unsigned int processed = 0;
    unsigned int nonblocking = pd->dag_flags & DAGF_NONBLOCK;
    unsigned int num_ext_hdr = 0;
    unsigned int ticks_per_second;

    # 获取下一个数据包的缓冲区（如果需要）
    while (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size) {
        /*
         * 检查是否调用了 "pcap_breakloop()"
         */
        if (p->break_loop) {
            /*
             * 是 - 清除指示已调用的标志，并返回 -2 以指示我们被告知跳出循环。
             */
            p->break_loop = 0;
            return -2;
        }

        /* 
         * dag_advance_stream() 将阻塞（除非调用了非阻塞），直到累积了64kB的数据。
         * 如果设置了 to_ms，它将在累积了64kB之前超时。
         * 我们等待64kB是因为一次处理几个数据包可能会在高数据包速率（>200kpps）下出现问题，因为效率低下。
         * 这意味着如果未指定 to_ms，则捕获可能会在数据速率极慢（<64kB/sec）时长时间“挂起”。
         * 如果指定了非阻塞，它将立即返回。然后用户需要负责效率。
         */
        if ( NULL == (pd->dag_mem_top = dag_advance_stream(p->fd, pd->dag_stream, &(pd->dag_mem_bottom))) ) {
             return -1;
        }

        if (nonblocking && (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size))
        {
            /* Pcap 配置为仅处理可用数据包，并且没有任何数据包可用，立即返回。 */
            return 0;
        }

        if(!nonblocking &&
           pd->dag_timeout &&
           (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size))
        {
            /* 阻塞模式，但设置了超时并且没有数据到达，无论如何都返回。*/
            return 0;
        }

    }

    /*
     * 处理数据包。
     *
     * 这假设一个数据包的单个缓冲区将有 <= INT_MAX 个数据包，因此数据包计数不会溢出。
     */
#ifdef DLT_MTP2_WITH_PHDR
                # 如果数据链路类型为DLT_MTP2_WITH_PHDR
                if (p->linktype == DLT_MTP2_WITH_PHDR) {
                    /* Add the MTP2 Pseudo Header */
                    # 增加MTP2伪头部
                    caplen += MTP2_HDR_LEN;
                    packet_len += MTP2_HDR_LEN;

                    TempPkt[MTP2_SENT_OFFSET] = 0;
                    TempPkt[MTP2_ANNEX_A_USED_OFFSET] = MTP2_ANNEX_A_USED_UNKNOWN;
                    *(TempPkt+MTP2_LINK_NUMBER_OFFSET) = ((header->rec.mc_hdlc.mc_header>>16)&0x01);
                    *(TempPkt+MTP2_LINK_NUMBER_OFFSET+1) = ((header->rec.mc_hdlc.mc_header>>24)&0xff);
                    memcpy(TempPkt+MTP2_HDR_LEN, dp, caplen);
                    dp = TempPkt;
                }
    }

    return processed;
}

static int
dag_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
    # 设置错误消息，因为DAG卡不支持发送数据包
    pcap_strlcpy(p->errbuf, "Sending packets isn't supported on DAG cards",
        PCAP_ERRBUF_SIZE);
    return (-1);
}

/*
 *  Get a handle for a live capture from the given DAG device.  Passing a NULL
 *  device will result in a failure.  The promisc flag is ignored because DAG
 *  cards are always promiscuous.  The to_ms parameter is used in setting the
 *  API polling parameters.
 *
 *  snaplen is now also ignored, until we get per-stream slen support. Set
 *  slen with appropriate DAG tool BEFORE pcap_activate().
 *
 *  See also pcap(3).
 */
static int dag_activate(pcap_t* p)
{
    # 获取私有数据结构
    struct pcap_dag *pd = p->priv;
    char *s;
    int n;
    daginf_t* daginf;
    char * newDev = NULL;
    char * device = p->opt.device;
    int ret;
    dag_size_t mindata;
    struct timeval maxwait;
    struct timeval poll;

    if (device == NULL) {
        # 如果设备为空，设置错误消息并返回错误
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "device is NULL");
        return PCAP_ERROR;
    }

    /* Initialize some components of the pcap structure. */
    # 初始化pcap结构的一些组件
    newDev = (char *)malloc(strlen(device) + 16);
    # 如果新设备为空，则分配设备名称字符串失败，设置错误码并格式化错误消息
    if (newDev == NULL) {
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't allocate string for device name");
        goto fail;
    }

    # 解析输入的设备名称，获取 DAG 设备和流编号（如果提供）
    if (dag_parse_name(device, newDev, strlen(device) + 16, &pd->dag_stream) < 0) {
        # 如果解析失败，设置错误码并格式化错误消息
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_parse_name");
        goto fail;
    }
    # 更新设备名称为解析后的新设备名称
    device = newDev;

    # 如果 DAG 流编号为奇数，则设置错误码并格式化错误消息
    if (pd->dag_stream%2) {
        ret = PCAP_ERROR;
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "dag_parse_name: tx (even numbered) streams not supported for capture");
        goto fail;
    }

    # 设置设备参数
    if((pd->dag_ref = dag_config_init((char *)device)) == NULL) {
        # 如果初始化配置失败，根据不同的错误类型设置错误码并格式化错误消息
        if (errno == ENOENT) {
            ret = PCAP_ERROR_NO_SUCH_DEVICE;
            p->errbuf[0] = '\0';
        } else if (errno == EPERM || errno == EACCES) {
            ret = PCAP_ERROR_PERM_DENIED;
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open %s failed with %s - additional privileges may be required",
                device, (errno == EPERM) ? "EPERM" : "EACCES");
        } else {
            ret = PCAP_ERROR;
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "dag_config_init %s", device);
        }
        goto fail;
    }
    # 如果获取 DAG 卡的文件描述符小于 0，则表示失败
    if((p->fd = dag_config_get_card_fd(pd->dag_ref)) < 0) {
        # 设置错误码并格式化错误消息
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_config_get_card_fd %s", device);
        # 跳转到失败关闭的标签
        goto failclose;
    }

    # 打开请求的流，如果已经被锁定或出现错误则可能失败
    if (dag_attach_stream64(p->fd, pd->dag_stream, 0, 0) < 0) {
        # 设置错误码并格式化错误消息
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_attach_stream");
        # 跳转到失败关闭的标签
        goto failclose;
    }

    # 尝试查找流丢包属性
    pd->drop_attr = kNullAttributeUuid;
    pd->dag_root = dag_config_get_root_component(pd->dag_ref);
    if ( dag_component_get_subcomponent(pd->dag_root, kComponentStreamFeatures, 0) )
    {
        pd->drop_attr = dag_config_get_indexed_attribute_uuid(pd->dag_ref, kUint32AttributeStreamDropCount, pd->dag_stream/2);
    }

    # 设置流的默认轮询参数，可以被 pcap_set_nonblock() 覆盖
    if (dag_get_stream_poll64(p->fd, pd->dag_stream,
                &mindata, &maxwait, &poll) < 0) {
        # 设置错误码并格式化错误消息
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_get_stream_poll");
        # 跳转到失败分离的标签
        goto faildetach;
    }

    # 使用轮询时间作为调用者使用 select()/等待数据包到达的事件循环的必需选择超时时间
    pd->required_select_timeout = poll;
    p->required_select_timeout = &pd->required_select_timeout;

    # 将负快照值（无效）、快照值为 0（未指定）或大于正常最大值的值转换为允许的最大值
    # 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN
    # 如果快照长度小于等于0或者大于最大快照长度，则将快照长度设置为最大快照长度
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    # 如果设置了立即回调选项，则立即调用回调函数
    # XXX - 这样做对吗？
    if (p->opt.immediate) {
        mindata = 0;
    } else {
        # 在调用回调函数之前收集的数据量（以字节为单位）
        # 对于效率很重要，但如果未设置 to_ms，则可能会引入延迟
        mindata = 65536;
    }

    # 如果设置了超时时间，则遵守 opt.timeout（以前是 to_ms）。这是一个好主意！
    # 建议设置为 10-100 毫秒。即使没有数据到达，调用也会超时。
    maxwait.tv_sec = p->opt.timeout/1000;
    maxwait.tv_usec = (p->opt.timeout%1000) * 1000;

    # 如果调用 dag_set_stream_poll64 失败，则设置错误信息并跳转到 faildetach 标签
    if (dag_set_stream_poll64(p->fd, pd->dag_stream,
                mindata, &maxwait, &poll) < 0) {
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_set_stream_poll");
        goto faildetach;
    }

    # XXX 不调用 dag_configure() 来设置 slen；在多流环境中这是不安全的，因为 gpp 配置是全局的。
    # 一旦固件提供了“每个流的 slen”，就可以通过 Config API 再次支持这个功能，而不会产生副作用
#if 0
    /* 设置捕获数据包的长度为指定的 snaplen 参数 */
    /* 这是一个非常糟糕的主意，因为不同的网卡有不同的有效 snaplen 范围。应该在 Config API 中修复。 */
    if (p->snapshot == 0 || p->snapshot > MAX_DAG_SNAPLEN) {
        p->snapshot = MAX_DAG_SNAPLEN;
    } else if (snaplen < MIN_DAG_SNAPLEN) {
        p->snapshot = MIN_DAG_SNAPLEN;
    }
    /* snap len 必须是 4 的倍数 */
#endif

    // 如果启动 DAG 数据流失败
    if(dag_start_stream(p->fd, pd->dag_stream) < 0) {
        ret = PCAP_ERROR;
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_start_stream %s", device);
        // 跳转到失败分支
        goto faildetach;
    }

    /*
     * 重要！你必须确保在启动时 bottom 被正确初始化为零，如果犯了这个错误，编译器不会给出警告！
     */
    pd->dag_mem_bottom = 0;
    pd->dag_mem_top = 0;

    /*
     * 找出应该剥离多少个 FCS 位。
     * 首先，查询网卡以查看它是否剥离了 FCS。
     */
    daginf = dag_info(p->fd);
    if ((0x4200 == daginf->device_code) || (0x4230 == daginf->device_code))    {
        /* DAG 4.2S 和 4.23S 已经剥离了 FCS。再次剥离最后一个字会截断数据包。 */
        pd->dag_fcs_bits = 0;

        /* 注意：不会提供 FCS。 */
        p->linktype_ext = LT_FCS_DATALINK_EXT(0);
    }
    } else {
        /*
         * Start out assuming it's 32 bits.
         */
        // 初始化数据包的 FCS 位数为 32 位
        pd->dag_fcs_bits = 32;

        /* Allow an environment variable to override. */
        // 允许环境变量覆盖默认值
        if ((s = getenv("ERF_FCS_BITS")) != NULL) {
            // 如果环境变量存在且符合条件，则将其值赋给 pd->dag_fcs_bits
            if ((n = atoi(s)) == 0 || n == 16 || n == 32) {
                pd->dag_fcs_bits = n;
            } else {
                // 如果环境变量值不符合条件，则返回错误
                ret = PCAP_ERROR;
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "pcap_activate %s: bad ERF_FCS_BITS value (%d) in environment", device, n);
                goto failstop;
            }
        }

        /*
         * Did the user request that they not be stripped?
         */
        // 检查用户是否要求不剥离 FCS
        if ((s = getenv("ERF_DONT_STRIP_FCS")) != NULL) {
            /* Yes.  Note the number of 16-bit words that will be
               supplied. */
            // 如果用户要求不剥离 FCS，则记录每个数据包中包含的 16 位字数
            p->linktype_ext = LT_FCS_DATALINK_EXT(pd->dag_fcs_bits/16);

            /* And don't strip them. */
            // 并且不剥离 FCS
            pd->dag_fcs_bits = 0;
        }
    }

    pd->dag_timeout    = p->opt.timeout;

    p->linktype = -1;
    // 获取数据链路类型
    if (dag_get_datalink(p) < 0) {
        ret = PCAP_ERROR;
        goto failstop;
    }

    p->bufsize = 0;

    // 初始化 DAG 设备
    if (new_pcap_dag(p) < 0) {
        ret = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "new_pcap_dag %s", device);
        goto failstop;
    }

    /*
     * "select()" and "poll()" don't work on DAG device descriptors.
     */
    // DAG 设备描述符不支持 select() 和 poll() 函数
    p->selectable_fd = -1;

    // 释放 newDev 指针指向的内存
    if (newDev != NULL) {
        free((char *)newDev);
    }

    // 设置操作函数指针
    p->read_op = dag_read;
    p->inject_op = dag_inject;
    p->setfilter_op = install_bpf_program;
    p->setdirection_op = NULL; /* Not implemented.*/
    p->set_datalink_op = dag_set_datalink;
    p->getnonblock_op = pcap_getnonblock_fd;
    p->setnonblock_op = dag_setnonblock;
    p->stats_op = dag_stats;
    p->cleanup_op = dag_platform_cleanup;
    pd->stat.ps_drop = 0;
    pd->stat.ps_recv = 0;
    pd->stat.ps_ifdrop = 0;
    return 0;
# 定义一个标签，用于在后续代码中进行跳转
failstop:
    # 如果停止数据流失败
    if (dag_stop_stream(p->fd, pd->dag_stream) < 0) {
        # 打印错误信息
        fprintf(stderr,"dag_stop_stream: %s\n", strerror(errno));
    }

faildetach:
    # 如果分离数据流失败
    if (dag_detach_stream(p->fd, pd->dag_stream) < 0)
        # 打印错误信息
        fprintf(stderr,"dag_detach_stream: %s\n", strerror(errno));

failclose:
    # 释放 DAG 配置资源
    dag_config_dispose(pd->dag_ref);
    '''
     * 注意：我们不需要调用 close(p->fd) 或 dag_close(p->fd),
     * 因为 dag_config_dispose(pd->dag_ref) 会执行这些操作。
     *
     * 将 p->fd 设置为 -1 以确保不会执行这些操作。
     '''
    p->fd = -1;
    pd->dag_ref = NULL;
    # 删除 pcap_dag 结构体
    delete_pcap_dag(p);

fail:
    # 清理 pcap_dag 结构体
    pcap_cleanup_live_common(p);
    # 如果 newDev 不为空，则释放其内存
    if (newDev != NULL) {
        free((char *)newDev);
    }
    # 返回结果
    return ret;
}

# 创建一个名为 dag_create 的函数，用于创建 DAG 设备
pcap_t *dag_create(const char *device, char *ebuf, int *is_ours)
{
    const char *cp;
    char *cpend;
    long devnum;
    pcap_t *p;
    long stream = 0;

    # 检查设备名是否符合 DAG 设备的命名规则
    cp = strrchr(device, '/');
    if (cp == NULL)
        cp = device;
    # 检查设备名是否以 "dag" 开头
    if (strncmp(cp, "dag", 3) != 0) {
        # 如果不是以 "dag" 开头，则不是我们的设备
        *is_ours = 0;
        return NULL;
    }
    # 检查设备名后面是否跟着从 0 到 DAG_MAX_BOARDS-1 的数字
    cp += 3;
    devnum = strtol(cp, &cpend, 10);
    if (*cpend == ':') {
        # 如果后面跟着一个流号
        stream = strtol(++cpend, &cpend, 10);
    }

    if (cpend == cp || *cpend != '\0') {
        # 如果后面没有跟着数字
        *is_ours = 0;
        return NULL;
    }

    if (devnum < 0 || devnum >= DAG_MAX_BOARDS) {
        # 如果跟着的数字不在有效范围内
        *is_ours = 0;
        return NULL;
    }

    if (stream <0 || stream >= DAG_STREAM_MAX) {
        # 如果流号不在有效范围内
        *is_ours = 0;
        return NULL;
    }

    # 符合 DAG 设备命名规则，设备可能是我们的设备
    *is_ours = 1;

    # 创建一个 pcap_dag 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dag);
    if (p == NULL)
        return NULL;

    # 设置激活操作为 dag_activate
    p->activate_op = dag_activate;
    """
    We claim that we support microsecond and nanosecond time
    stamps.

    XXX Our native precision is 2^-32s, but libpcap doesn't support
    power of two precisions yet. We can convert to either MICRO or NANO.
    """
    # 分配内存以支持微秒和纳秒时间戳
    p->tstamp_precision_list = malloc(2 * sizeof(u_int));
    # 如果内存分配失败，则设置错误消息并关闭 pcap
    if (p->tstamp_precision_list == NULL) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        pcap_close(p);
        return NULL;
    }
    # 设置时间戳精度列表的值
    p->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
    p->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
    # 设置时间戳精度列表的长度
    p->tstamp_precision_count = 2;
    # 返回 pcap 对象
    return p;
# 静态函数，用于获取 DAG 设备的统计信息
static int
dag_stats(pcap_t *p, struct pcap_stat *ps) {
    # 获取 pcap_t 结构体中的私有数据，即 pcap_dag 结构体
    struct pcap_dag *pd = p->priv;
    uint32_t stream_drop;
    dag_err_t dag_error;

    '''
    接收到的数据包记录（ps_recv）在 dag_read() 中计数。
    丢弃的数据包记录（ps_drop）从流丢弃属性中读取，如果不存在，则在 dag_read() 中整合 ERF 头部的 lctr 计数（如果可用）。
    我们报告卡/驱动程序没有丢弃任何记录（ps_ifdrop）。
    '''

    # 如果存在丢弃属性
    if(pd->drop_attr != kNullAttributeUuid) {
        '''
        注意：此计数器在捕获开始时被清除，并将在 UINT_MAX 处回绕。
        应用程序负责频繁轮询 ps_drop，以便检测每次回绕并将总丢弃整合到更广泛的计数器中
        '''
        # 从 DAG 配置中获取流丢弃属性的值，并将其赋给 ps_drop
        if ((dag_error = dag_config_get_uint32_attribute_ex(pd->dag_ref, pd->drop_attr, &stream_drop)) == kDagErrNone) {
            pd->stat.ps_drop = stream_drop;
        } else {
            # 如果获取失败，将错误信息写入 errbuf，并返回 -1
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "reading stream drop attribute: %s",
                 dag_config_strerror(dag_error));
            return -1;
        }
    }

    # 将私有数据结构中的统计信息赋给传入的结构体指针
    *ps = pd->stat;

    # 返回 0 表示成功
    return 0;
}

'''
添加所有 DAG 设备。
'''
# 查找所有 DAG 设备
int
dag_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
    char name[12];    /* XXX - pick a size */
    int c;
    char dagname[DAGNAME_BUFSIZE];
    int dagstream;
    int dagfd;
    dag_card_inf_t *inf;
    char *description;
    int stream, rxstreams;

    # 尝试所有 DAG 设备，编号从 0 到 DAG_MAX_BOARDS
    # 遍历 DAG 最大板卡数
    for (c = 0; c < DAG_MAX_BOARDS; c++) {
        # 格式化板卡名称
        snprintf(name, 12, "dag%d", c);
        # 解析板卡名称，获取板卡信息
        if (-1 == dag_parse_name(name, dagname, DAGNAME_BUFSIZE, &dagstream))
        {
            # 如果板卡名称无法解析，返回错误
            (void) snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "dag: device name %s can't be parsed", name);
            return (-1);
        }
        # 打开板卡，获取文件描述符
        if ( (dagfd = dag_open(dagname)) >= 0 ) {
            description = NULL;
            # 获取板卡的 PCI 信息
            if ((inf = dag_pciinfo(dagfd)))
                description = dag_device_name(inf->device_code, 1);
            # 添加板卡到设备列表
            if (add_dev(devlistp, name, 0, description, errbuf) == NULL) {
                # 添加失败，返回错误
                return (-1);
            }
            # 获取板卡的接收流数量
            rxstreams = dag_rx_get_stream_count(dagfd);
            # 遍历板卡的接收流
            for(stream=0;stream<DAG_STREAM_MAX;stream+=2) {
                # 如果成功附加流
                if (0 == dag_attach_stream64(dagfd, stream, 0, 0)) {
                    # 分离流
                    dag_detach_stream(dagfd, stream);
                    # 格式化流名称
                    snprintf(name,  10, "dag%d:%d", c, stream);
                    # 添加流到设备列表
                    if (add_dev(devlistp, name, 0, description, errbuf) == NULL) {
                        # 添加失败，返回错误
                        return (-1);
                    }
                    # 减少接收流数量
                    rxstreams--;
                    # 如果接收流数量小于等于0，跳出循环
                    if(rxstreams <= 0) {
                        break;
                    }
                }
            }
            # 关闭板卡
            dag_close(dagfd);
        }
    }
    # 返回成功
    return (0);
# 设置数据链路类型
static int
dag_set_datalink(pcap_t *p, int dlt)
{
    # 设置 pcap_t 结构体中的数据链路类型
    p->linktype = dlt;

    # 返回 0 表示成功
    return (0);
}

# 设置非阻塞模式
static int
dag_setnonblock(pcap_t *p, int nonblock)
{
    # 获取 pcap_dag 结构体
    struct pcap_dag *pd = p->priv;
    dag_size_t mindata;
    struct timeval maxwait;
    struct timeval poll;

    # 设置文件描述符为非阻塞模式
    if (pcap_setnonblock_fd(p, nonblock) < 0)
        return (-1);

    # 获取数据流的轮询参数
    if (dag_get_stream_poll64(p->fd, pd->dag_stream,
                &mindata, &maxwait, &poll) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_get_stream_poll");
        return -1;
    }

    # 设置数据收集的最小数据量
    if(nonblock)
        mindata = 0;
    else
        mindata = 65536;

    # 设置数据流的轮询参数
    if (dag_set_stream_poll64(p->fd, pd->dag_stream,
                mindata, &maxwait, &poll) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "dag_set_stream_poll");
        return -1;
    }

    # 设置非阻塞标志
    if (nonblock) {
        pd->dag_flags |= DAGF_NONBLOCK;
    } else {
        pd->dag_flags &= ~DAGF_NONBLOCK;
    }
    return (0);
}

# 获取数据链路类型
static int
dag_get_datalink(pcap_t *p)
{
    # 获取 pcap_dag 结构体
    struct pcap_dag *pd = p->priv;
    int index=0, dlt_index=0;
    uint8_t types[255];

    # 初始化 types 数组
    memset(types, 0, 255);

    # 分配内存给 dlt_list
    if (p->dlt_list == NULL && (p->dlt_list = malloc(255*sizeof(*(p->dlt_list)))) == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "malloc");
        return (-1);
    }

    # 设置数据链路类型为 0
    p->linktype = 0;

    # 获取可能的 ERF 类型列表
#ifdef HAVE_DAG_GET_STREAM_ERF_TYPES
    /* Get list of possible ERF types for this card */
    # 如果从文件描述符和 DAG 流中获取 ERF 类型失败
    if (dag_get_stream_erf_types(p->fd, pd->dag_stream, types, 255) < 0) {
        # 格式化错误消息，将错误信息存储到 errbuf 中
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "dag_get_stream_erf_types");
        # 返回 -1，表示获取失败
        return (-1);
    }

    # 循环遍历 ERF 类型数组
    while (types[index]) {
#elif defined HAVE_DAG_GET_ERF_TYPES
    /* 如果定义了 HAVE_DAG_GET_ERF_TYPES，则获取此卡可能的 ERF 类型列表 */
    if (dag_get_erf_types(p->fd, types, 255) < 0) {
        // 如果获取失败，格式化错误消息并返回 -1
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "dag_get_erf_types");
        return (-1);
    }

    while (types[index]) {
#else
    /* 否则，通过 dagapi 调用检查类型。 */
    types[index] = dag_linktype(p->fd);

    {
    }

    // 将 DLT_ERF 添加到 dlt_list 中
    p->dlt_list[dlt_index++] = DLT_ERF;

    // 更新 dlt_count
    p->dlt_count = dlt_index;

    // 如果 linktype 为空，则设置为 DLT_ERF
    if(!p->linktype)
        p->linktype = DLT_ERF;

    // 返回 linktype
    return p->linktype;
}

#ifdef DAG_ONLY
/*
 * 此 libpcap 构建仅支持 DAG 卡，而不是常规网络接口。
 */

/*
 * 没有常规接口，只有 DAG 接口。
 */
int
pcap_platform_finddevs(pcap_if_list_t *devlistp _U_, char *errbuf)
{
    return (0);
}

/*
 * 尝试打开常规接口失败。
 */
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
    snprintf(errbuf, PCAP_ERRBUF_SIZE,
        "This version of libpcap only supports DAG cards");
    return NULL;
}

/*
 * Libpcap 版本字符串。
 */
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING " (DAG-only)");
}
#endif
```