# `nmap\libpcap\pcap-npf.c`

```cpp
/*
 * 版权声明，版权所有
 * 1. 允许在源代码和二进制形式下重新分发和使用，需满足以下条件：
 *    1）源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *    2）二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 2. 不得使用 Politecnico di Torino、CACE Technologies 或其贡献者的名称来认可或推广从本软件衍生的产品，除非事先书面许可。
 * 免责声明：本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他原因）产生的任何直接、间接、附带、特殊、惩罚性或后果性损害，版权所有者或贡献者均不承担责任。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <errno.h>
#include <limits.h> /* for INT_MAX */
#define PCAP_DONT_INCLUDE_PCAP_BPF_H
#include <Packet32.h>
#include <pcap-int.h>
#include <pcap/dlt.h>
/*
 * XXX - Packet32.h defines bpf_program, so we can't include
 * <pcap/bpf.h>, which also defines it; that's why we define
 * PCAP_DONT_INCLUDE_PCAP_BPF_H,
 *
 * However, no header in the WinPcap or Npcap SDKs defines the
 * macros for BPF code, so we have to define them ourselves.
 */
#define        BPF_RET        0x06  // 定义 BPF_RET 宏为十六进制 0x06
#define        BPF_K        0x00    // 定义 BPF_K 宏为十六进制 0x00

/* Old-school MinGW have these headers in a different place.
 */
#if defined(__MINGW32__) && !defined(__MINGW64_VERSION_MAJOR)
  #include <ddk/ntddndis.h>
  #include <ddk/ndis.h>
#else
  #include <ntddndis.h>  /* MSVC/TDM-MinGW/MinGW64 */
#endif

#ifdef HAVE_DAG_API
  #include <dagnew.h>
  #include <dagapi.h>
#endif /* HAVE_DAG_API */

#include "diag-control.h"

#include "pcap-airpcap.h"

static int pcap_setfilter_npf(pcap_t *, struct bpf_program *);  // 声明 pcap_setfilter_npf 函数
static int pcap_setfilter_win32_dag(pcap_t *, struct bpf_program *);  // 声明 pcap_setfilter_win32_dag 函数
static int pcap_getnonblock_npf(pcap_t *);  // 声明 pcap_getnonblock_npf 函数
static int pcap_setnonblock_npf(pcap_t *, int);  // 声明 pcap_setnonblock_npf 函数

/*dimension of the buffer in the pcap_t structure*/
#define    WIN32_DEFAULT_USER_BUFFER_SIZE 256000  // 定义 WIN32_DEFAULT_USER_BUFFER_SIZE 宏为 256000

/*dimension of the buffer in the kernel driver NPF */
#define    WIN32_DEFAULT_KERNEL_BUFFER_SIZE 1000000  // 定义 WIN32_DEFAULT_KERNEL_BUFFER_SIZE 宏为 1000000

/* Equivalent to ntohs(), but a lot faster under Windows */
#define SWAPS(_X) ((_X & 0xff) << 8) | (_X >> 8)  // 定义 SWAPS 宏为对 _X 进行字节序转换的操作

/*
 * Private data for capturing on WinPcap/Npcap devices.
 */
struct pcap_win {
    ADAPTER *adapter;        /* the packet32 ADAPTER for the device */  // 用于设备的 packet32 ADAPTER
    int nonblock;  // 非阻塞标志
    int rfmon_selfstart;        /* a flag tells whether the monitor mode is set by itself */  // 监控模式是否自启动的标志
    int filtering_in_kernel;    /* using kernel filter */  // 是否在内核中使用过滤器

#ifdef HAVE_DAG_API
    int    dag_fcs_bits;        /* Number of checksum bits from link layer */  // 链路层校验位数
#endif

#ifdef ENABLE_REMOTE
    int samp_npkt;            /* parameter needed for sampling, with '1 out of N' method has been requested */  // 用于采样的参数
    struct timeval samp_time;    /* parameter needed for sampling, with '1 every N ms' method has been requested */  // 用于采样的参数
#endif
};
#ifndef HAVE_NPCAP_PACKET_API
// 如果不是 Npcap，则定义监视模式支持的存根版本
static int
PacketIsMonitorModeSupported(PCHAR AdapterName _U_)
{
    // 不支持监视模式
    return (0);
}

static int
PacketSetMonitorMode(PCHAR AdapterName _U_, int mode _U_)
{
    // 这个函数不应该被调用，因为PacketIsMonitorModeSupported()将返回0，意味着“我们不支持监视模式，所以不要尝试打开或关闭它”。
    return (0);
}

static int
PacketGetMonitorMode(PCHAR AdapterName _U_)
{
    // 这应该失败，以便pcap_activate_npf()在我们的调用者请求监视模式时返回PCAP_ERROR_RFMON_NOTSUP。
    return (-1);
}
#endif

// PacketRequest()将使用DeviceIoControl()调用NPF驱动程序执行OID请求，使用BIOCQUERYOID ioctl。
// 如果OID请求不受OS或驱动程序支持，内核代码应该返回NDIS_STATUS_INVALID_OID、NDIS_STATUS_NOT_SUPPORTED或NDIS_STATUS_NOT_RECOGNIZED之一，但这似乎并不可靠地传递给PacketRequest()的调用者。
#define NDIS_STATUS_INVALID_OID        0xc0010017
#define NDIS_STATUS_NOT_SUPPORTED    0xc00000bb    /* STATUS_NOT_SUPPORTED */
#define NDIS_STATUS_NOT_RECOGNIZED    0x00010001

static int
oid_get_request(ADAPTER *adapter, bpf_u_int32 oid, void *data, size_t *lenp,
    char *errbuf)
{
    PACKET_OID_DATA *oid_data_arg;

    // 分配一个PACKET_OID_DATA结构以传递给PacketRequest()。它应该足够大，可以容纳“*lenp”字节的数据；实际上会稍微大一些，因为PACKET_OID_DATA在末尾有一个1字节的数据数组，代表实际存在的可变长度数据。
    oid_data_arg = malloc(sizeof (PACKET_OID_DATA) + *lenp);
    # 如果传入的 oid_data_arg 为空指针，则返回错误并设置错误信息
    if (oid_data_arg == NULL) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "Couldn't allocate argument buffer for PacketRequest");
        return (PCAP_ERROR);
    }

    /*
     * 不需要复制数据 - 我们正在进行获取操作。
     */
    # 将传入的 oid 和长度设置到 oid_data_arg 结构体中
    oid_data_arg->Oid = oid;
    oid_data_arg->Length = (ULONG)(*lenp);    /* XXX - check for ridiculously large value? */
    # 调用 PacketRequest 函数进行数据获取
    if (!PacketRequest(adapter, FALSE, oid_data_arg)) {
        # 如果调用失败，则设置错误信息并释放内存
        pcap_fmt_errmsg_for_win32_err(errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "Error calling PacketRequest");
        free(oid_data_arg);
        return (-1);
    }

    /*
     * 获取实际提供的数据长度。
     */
    # 更新 lenp 指向的值为实际获取的数据长度
    *lenp = oid_data_arg->Length;

    /*
     * 复制我们获取的数据。
     */
    # 将获取的数据复制到 data 指向的位置，并释放内存
    memcpy(data, oid_data_arg->Data, *lenp);
    free(oid_data_arg);
    return (0);
}
# 定义一个名为 pcap_stats_npf 的静态函数，接受一个 pcap_t 类型的指针 p 和一个 struct pcap_stat 类型的指针 ps 作为参数
static int
pcap_stats_npf(pcap_t *p, struct pcap_stat *ps) {
    # 获取 pcap_t 结构体指针 p 的私有成员 pw
    struct pcap_win *pw = p->priv;
    # 定义一个名为 bstats 的 struct bpf_stat 结构体
    struct bpf_stat bstats;

    """
    尝试获取统计信息。

    （请注意 - "struct pcap_stat" 和 WinPcap 的 "struct bpf_stat" 不是同一个结构体。
    它们可能当前具有相同的布局，但让我们不要欺骗。

    还要注意，我们不填充 ps_capt，因为我们可能被针对早期版本的 WinPcap 编译的代码调用，
    那个版本没有 ps_capt，如果填充它，就会覆盖传递给我们的结构体之后的任何内容。
    """
    # 如果无法获取统计信息，则返回错误
    if (!PacketGetStats(pw->adapter, &bstats)) {
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "PacketGetStats error");
        return (-1);
    }
    # 将接收的数据包数和丢弃的数据包数分别赋值给 ps 的成员变量 ps_recv 和 ps_drop
    ps->ps_recv = bstats.bs_recv;
    ps->ps_drop = bstats.bs_drop;

    """
    XXX - PacketGetStats() 不会填充这个值，所以我们只返回 0。
    """
    # 如果条件为真，则将 bstats 的 ps_ifdrop 赋值给 ps 的 ps_ifdrop，否则赋值为 0
    # 由于条件为假，所以 ps_ifdrop 被赋值为 0
#if 0
    ps->ps_ifdrop = bstats.ps_ifdrop;
#else
    ps->ps_ifdrop = 0;
#endif

    # 返回 0，表示成功获取统计信息
    return (0);
}
/*
 * 仅适用于 Win32 的获取统计信息的例程。
 *
 * 这种方式绝对比从用户空间传递 pcap_stat * 更安全。
 * 实际上，可能发生用户分配的变量不够大以容纳新结构，库将会写入未分配给该变量的区域。
 *
 * 通过这种方式，我们可以确保我们在为该变量分配的内存上进行写入。
 *
 * 但这种方式处理统计信息是错误的。相反，我们应该有一个 API，以类似于 pcapng 接口统计块的选项部分的形式返回数据：
 *
 *    https://xml2rfc.tools.ietf.org/cgi-bin/xml2rfc.cgi?url=https://raw.githubusercontent.com/pcapng/pcapng/master/draft-tuexen-opsawg-pcapng.xml&modeAsFormat=html/ascii&type=ascii#rfc.section.4.6
 *
 * 这样可以让我们直接添加新的统计信息，并指示我们提供和不提供哪些统计信息，而不是不得不为我们无法提供的统计信息提供可能虚假的值。
 */
static struct pcap_stat *
pcap_stats_ex_npf(pcap_t *p, int *pcap_stat_size)
{
    struct pcap_win *pw = p->priv;
    struct bpf_stat bstats;

    *pcap_stat_size = sizeof (p->stat);

    /*
     * 尝试获取统计信息。
     *
     * （请注意 - "struct pcap_stat" *不* 与 WinPcap 的 "struct bpf_stat" 相同。它目前可能具有相同的布局，但让我们不要欺骗。）
     */
    if (!PacketGetStatsEx(pw->adapter, &bstats)) {
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "PacketGetStatsEx error");
        return (NULL);
    }
    p->stat.ps_recv = bstats.bs_recv;
    p->stat.ps_drop = bstats.bs_drop;
    p->stat.ps_ifdrop = bstats.ps_ifdrop;
    /*
     * 以防万一，这个代码被编译到除 Windows 之外的目标上，这几乎是极不可能的，几乎是不可能的。
     */
#ifdef _WIN32
    p->stat.ps_capt = bstats.bs_capt;
#endif
}
    # 返回指向 p 结构体中 stat 成员的指针
    return (&p->stat);
/* 设置内核级捕获缓冲区的维度 */
static int
pcap_setbuff_npf(pcap_t *p, int dim)
{
    struct pcap_win *pw = p->priv;

    // 如果无法分配足够的内存来分配内核缓冲区，则返回错误
    if(PacketSetBuff(pw->adapter,dim)==FALSE)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "driver error: not enough memory to allocate the kernel buffer");
        return (-1);
    }
    return (0);
}

/* 设置驱动程序的工作模式 */
static int
pcap_setmode_npf(pcap_t *p, int mode)
{
    struct pcap_win *pw = p->priv;

    // 如果无法识别工作模式，则返回错误
    if(PacketSetMode(pw->adapter,mode)==FALSE)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "driver error: working mode not recognized");
        return (-1);
    }

    return (0);
}

/* 设置释放读取调用的最小数据量 */
static int
pcap_setmintocopy_npf(pcap_t *p, int size)
{
    struct pcap_win *pw = p->priv;

    // 如果无法设置请求的最小拷贝大小，则返回错误
    if(PacketSetMinToCopy(pw->adapter, size)==FALSE)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "driver error: unable to set the requested mintocopy size");
        return (-1);
    }
    return (0);
}

static HANDLE
pcap_getevent_npf(pcap_t *p)
{
    struct pcap_win *pw = p->priv;

    return (PacketGetReadEvent(pw->adapter));
}

static int
pcap_oid_get_request_npf(pcap_t *p, bpf_u_int32 oid, void *data, size_t *lenp)
{
    struct pcap_win *pw = p->priv;

    return (oid_get_request(pw->adapter, oid, data, lenp, p->errbuf));
}

static int
pcap_oid_set_request_npf(pcap_t *p, bpf_u_int32 oid, const void *data,
    size_t *lenp)
{
    struct pcap_win *pw = p->priv;
    PACKET_OID_DATA *oid_data_arg;

    /*
     * 分配一个 PACKET_OID_DATA 结构以传递给 PacketRequest()。
     * 它应该足够大，以容纳 "*lenp" 字节的数据；实际上会稍微大一些，因为 PACKET_OID_DATA 在最后有一个1字节的数据数组，代表实际存在的可变长度数据。
     */
    oid_data_arg = malloc(sizeof (PACKET_OID_DATA) + *lenp);
}
    # 如果传入的 oid_data_arg 为空指针，则返回错误并设置错误信息
    if (oid_data_arg == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Couldn't allocate argument buffer for PacketRequest");
        return (PCAP_ERROR);
    }

    # 设置 oid_data_arg 结构体的 Oid 字段为传入的 oid
    oid_data_arg->Oid = oid;
    # 设置 oid_data_arg 结构体的 Length 字段为传入的 lenp 指针所指向的值
    oid_data_arg->Length = (ULONG)(*lenp);    /* XXX - check for ridiculously large value? */
    # 将 data 指向的数据拷贝到 oid_data_arg 结构体的 Data 字段中
    memcpy(oid_data_arg->Data, data, *lenp);
    # 调用 PacketRequest 函数，传入适配器、TRUE 和 oid_data_arg 结构体
    if (!PacketRequest(pw->adapter, TRUE, oid_data_arg)) {
        # 如果调用失败，则设置错误信息并释放 oid_data_arg 结构体的内存，返回错误
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "Error calling PacketRequest");
        free(oid_data_arg);
        return (PCAP_ERROR);
    }

    # 获取实际拷贝的数据长度
    *lenp = oid_data_arg->Length;

    # 释放 oid_data_arg 结构体的内存
    free(oid_data_arg);
    # 返回成功
    return (0);
static u_int
pcap_sendqueue_transmit_npf(pcap_t *p, pcap_send_queue *queue, int sync)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_win *pw = p->priv;
    // 定义变量 res
    u_int res;

    // 调用 PacketSendPackets 函数发送队列中的数据包
    res = PacketSendPackets(pw->adapter,
        queue->buffer,
        queue->len,
        (BOOLEAN)sync);

    // 如果发送的数据包数量不等于队列中的数据包数量
    if(res != queue->len){
        // 格式化错误消息，记录错误信息到错误缓冲区
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "Error queueing packets");
    }

    // 返回发送的数据包数量
    return (res);
}

static int
pcap_setuserbuffer_npf(pcap_t *p, int size)
{
    // 定义指向新缓冲区的指针
    unsigned char *new_buff;

    // 如果 size 小于等于 0
    if (size<=0) {
        // 记录错误信息到错误缓冲区
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error: invalid size %d",size);
        // 返回错误
        return (-1);
    }

    // 分配新的缓冲区
    new_buff=(unsigned char*)malloc(sizeof(char)*size);

    // 如果分配失败
    if (!new_buff) {
        // 记录错误信息到错误缓冲区
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error: not enough memory");
        // 返回错误
        return (-1);
    }

    // 释放旧的缓冲区
    free(p->buffer);

    // 将指针指向新的缓冲区
    p->buffer=new_buff;
    // 更新缓冲区大小
    p->bufsize=size;

    // 返回成功
    return (0);
}

#ifdef HAVE_NPCAP_PACKET_API
/*
 * Kernel dump mode isn't supported in Npcap; calls to PacketSetDumpName(),
 * PacketSetDumpLimits(), and PacketIsDumpEnded() will get compile-time
 * deprecation warnings.
 *
 * Avoid calling them; just return errors indicating that kernel dump
 * mode isn't supported in Npcap.
 */
static int
pcap_live_dump_npf(pcap_t *p, char *filename _U_, int maxsize _U_,
    int maxpacks _U_)
{
    // 记录错误信息到错误缓冲区
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Npcap doesn't support kernel dump mode");
    // 返回错误
    return (-1);
}
static int
pcap_live_dump_ended_npf(pcap_t *p, int sync)
{
    // 记录错误信息到错误缓冲区
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Npcap doesn't support kernel dump mode");
    // 返回错误
    return (-1);
}
#else /* HAVE_NPCAP_PACKET_API */
static int
pcap_live_dump_npf(pcap_t *p, char *filename, int maxsize, int maxpacks)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_win *pw = p->priv;
    // 定义变量 res
    BOOLEAN res;

    // 设置数据包驱动程序为转储模式
    res = PacketSetMode(pw->adapter, PACKET_MODE_DUMP);
    # 如果设置 dump 模式失败，则返回错误信息并退出
    if(res == FALSE){
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error setting dump mode");
        return (-1);
    }

    # 设置 dump 文件的名称
    res = PacketSetDumpName(pw->adapter, filename, (int)strlen(filename));
    # 如果设置 dump 文件名称失败，则返回错误信息并退出
    if(res == FALSE){
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error setting kernel dump file name");
        return (-1);
    }

    # 设置 dump 文件的大小限制
    res = PacketSetDumpLimits(pw->adapter, maxsize, maxpacks);
    # 如果设置 dump 文件大小限制失败，则返回错误信息并退出
    if(res == FALSE) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Error setting dump limit");
        return (-1);
    }

    # 设置成功，返回 0
    return (0);
# 检查是否捕获到的数据包已经结束，并返回结果
static int
pcap_live_dump_ended_npf(pcap_t *p, int sync)
{
    # 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;

    # 调用 PacketIsDumpEnded 函数判断数据包是否结束，并返回结果
    return (PacketIsDumpEnded(pw->adapter, (BOOLEAN)sync));
}
#endif /* HAVE_NPCAP_PACKET_API */

#ifdef HAVE_AIRPCAP_API
# 获取 AirPcap 句柄
static PAirpcapHandle
pcap_get_airpcap_handle_npf(pcap_t *p)
{
    # 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;

    # 调用 PacketGetAirPcapHandle 函数获取 AirPcap 句柄
    return (PacketGetAirPcapHandle(pw->adapter));
}
#else /* HAVE_AIRPCAP_API */
# 如果没有 AirPcap API，则返回空指针
static PAirpcapHandle
pcap_get_airpcap_handle_npf(pcap_t *p _U_)
{
    return (NULL);
}
#endif /* HAVE_AIRPCAP_API */

# 读取数据包
static int
pcap_read_npf(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    # 定义变量
    PACKET Packet;
    int cc;
    int n;
    register u_char *bp, *ep;
    u_char *datap;
    # 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;

    # 获取数据包长度
    cc = p->cc;
    }
    else
        bp = p->bp;

    '''
     * 循环遍历每个数据包。
     *
     * 这假设单个数据包缓冲区将有 <= INT_MAX 个数据包，因此数据包计数不会溢出。
     */
#define bhp ((struct bpf_hdr *)bp)
    n = 0;
    ep = bp + cc;
    # 进入无限循环
    for (;;) {
        # 声明并初始化变量 caplen 和 hdrlen
        register u_int caplen, hdrlen;

        """
         * 检查是否调用了 "pcap_breakloop()"
         * 如果是，立即返回 - 如果我们还没有读取任何数据包，清除标志并返回 PCAP_ERROR_BREAK
         * 表示我们被告知要跳出循环，否则保持标志设置，这样下一次调用将在没有读取任何数据包的情况下跳出循环，并返回到目前为止处理的数据包数量。
         """
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (PCAP_ERROR_BREAK);
            } else {
                p->bp = bp;
                p->cc = (int) (ep - bp);
                return (n);
            }
        }
        # 如果数据包指针超出了数据包结束指针，跳出循环
        if (bp >= ep)
            break;

        # 获取数据包的长度和头部长度
        caplen = bhp->bh_caplen;
        hdrlen = bhp->bh_hdrlen;
        # 计算数据包的数据指针
        datap = bp + hdrlen;

        """
         * 短路求值：如果在内核中使用 BPF 过滤器，则无需现在进行过滤 - 我们已经知道数据包通过了过滤器。
         *
         * XXX - 如果程序为空指针，pcap_filter() 应该始终返回 TRUE，但它可能只是尝试“运行”过滤器，因此我们在这里进行检查。
         """
        if (pw->filtering_in_kernel ||
            p->fcode.bf_insns == NULL ||
            pcap_filter(p->fcode.bf_insns, datap, bhp->bh_datalen, caplen)) {
#ifdef ENABLE_REMOTE
            # 如果远程采样功能被启用，则执行以下代码块

            switch (p->rmt_samp.method) {
                # 根据远程采样方法进行不同的处理

                case PCAP_SAMP_1_EVERY_N:
                    # 如果远程采样方法为每N个包采样一个
                    pw->samp_npkt = (pw->samp_npkt + 1) % p->rmt_samp.value;

                    /* Discard all packets that are not '1 out of N' */
                    # 如果不是每N个包中的一个，则丢弃该包并继续下一个包
                    if (pw->samp_npkt != 0) {
                        bp += Packet_WORDALIGN(caplen + hdrlen);
                        continue;
                    }
                    break;

                case PCAP_SAMP_FIRST_AFTER_N_MS:
                    # 如果远程采样方法为每N毫秒后的第一个包
                    {
                    struct pcap_pkthdr *pkt_header = (struct pcap_pkthdr*) bp;

                    /*
                     * Check if the timestamp of the arrived
                     * packet is smaller than our target time.
                     */
                    # 检查到达的数据包的时间戳是否小于目标时间

                    if (pkt_header->ts.tv_sec < pw->samp_time.tv_sec ||
                       (pkt_header->ts.tv_sec == pw->samp_time.tv_sec && pkt_header->ts.tv_usec < pw->samp_time.tv_usec)) {
                        bp += Packet_WORDALIGN(caplen + hdrlen);
                        continue;
                    }

                    /*
                     * The arrived packet is suitable for being
                     * delivered to our caller, so let's update
                     * the target time.
                     */
                    # 到达的数据包适合传递给调用者，因此更新目标时间
                    pw->samp_time.tv_usec = pkt_header->ts.tv_usec + p->rmt_samp.value * 1000;
                    if (pw->samp_time.tv_usec > 1000000) {
                        pw->samp_time.tv_sec = pkt_header->ts.tv_sec + pw->samp_time.tv_usec / 1000000;
                        pw->samp_time.tv_usec = pw->samp_time.tv_usec % 1000000;
                    }
                    }
            }
#endif    /* ENABLE_REMOTE */

            /*
             * XXX A bpf_hdr matches a pcap_pkthdr.
             */
            // 调用回调函数处理捕获的数据包
            (*callback)(user, (struct pcap_pkthdr*)bp, datap);
            // 移动指针到下一个数据包的位置
            bp += Packet_WORDALIGN(caplen + hdrlen);
            // 如果达到指定的捕获数量，则返回捕获的数据包数量
            if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
                p->bp = bp;
                p->cc = (int) (ep - bp);
                return (n);
            }
        } else {
            /*
             * Skip this packet.
             */
            // 跳过当前数据包
            bp += Packet_WORDALIGN(caplen + hdrlen);
        }
    }
#undef bhp
    // 重置捕获缓冲区
    p->cc = 0;
    return (n);
}

#ifdef HAVE_DAG_API
static int
pcap_read_win32_dag(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 获取私有数据结构
    struct pcap_win *pw = p->priv;
    PACKET Packet;
    u_char *dp = NULL;
    int    packet_len = 0, caplen = 0;
    struct pcap_pkthdr    pcap_header;
    u_char *endofbuf;
    int n = 0;
    dag_record_t *header;
    unsigned erf_record_len;
    ULONGLONG ts;
    int cc;
    unsigned swt;
    unsigned dfp = pw->adapter->DagFastProcess;

    // 获取当前捕获的数据包数量
    cc = p->cc;
    // 如果已经处理完上一次读取的数据包，则获取新的数据包
    if (cc == 0) /* Get new packets only if we have processed all the ones of the previous read */
    {
        /*
         * 从网络获取新的数据包。
         *
         * PACKET 结构在 Windows 9x/Me 中有一堆额外的东西，但在我们支持的 Windows 版本中，其中唯一有趣的数据
         * 就是 p->buffer 的副本，p->buflen 的副本，以及从 PacketReceivePacket() 返回的实际读取的字节数，这些
         * 都不需要在调用之间保留，所以我们只在堆栈上保留一个。
         */
        PacketInitPacket(&Packet, (BYTE *)p->buffer, p->bufsize);
        if (!PacketReceivePacket(pw->adapter, &Packet, TRUE)) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "read error: PacketReceivePacket failed");
            return (-1);
        }
    
        cc = Packet.ulBytesReceived;
        if(cc == 0)
            /* 超时已过，但没有数据包到达 */
            return (0);
        header = (dag_record_t*)pw->adapter->DagBuffer;
    }
    else
        header = (dag_record_t*)p->bp;
    
    endofbuf = (char*)header + cc;
    
    /*
     * 这可能处理超过 INT_MAX 个数据包，这将导致数据包计数溢出，使其看起来像一个负数，从而导致我们返回一个
     * 看起来像错误的值，或者溢出回到正数领域，从而导致我们返回一个太低的计数。
     *
     * 因此，如果数据包计数是无限的，我们将其剪切到 INT_MAX；这个例程不应该无限处理数据包，所以这不是一个问题。
     */
    if (PACKET_COUNT_IS_UNLIMITED(cnt))
        cnt = INT_MAX;
    
    /*
     * 循环遍历数据包
     */
    do
    }
    while((u_char*)header < endofbuf);
    
    return (1);
/* 结束对 HAVE_DAG_API 的条件编译 */
#endif /* HAVE_DAG_API */

/* 向网络发送数据包 */
static int
pcap_inject_npf(pcap_t *p, const void *buf, int size)
{
    // 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;
    // 初始化 PACKET 结构体
    PACKET pkt;
    
    // 初始化数据包
    PacketInitPacket(&pkt, (PVOID)buf, size);
    // 发送数据包到网络
    if(PacketSendPacket(pw->adapter,&pkt,TRUE) == FALSE) {
        // 如果发送失败，设置错误消息并返回-1
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "send error: PacketSendPacket failed");
        return (-1);
    }

    /*
     * 如果 "PacketSendPacket()" 成功，我们假设所有数据都已发送。
     * "pcap_inject()" 应该返回发送的字节数。
     */
    return (size);
}

// 清理 NPF 设备
static void
pcap_cleanup_npf(pcap_t *p)
{
    // 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;

    // 如果适配器不为空，关闭适配器
    if (pw->adapter != NULL) {
        PacketCloseAdapter(pw->adapter);
        pw->adapter = NULL;
    }
    // 如果 rfmon_selfstart 为真，关闭监控模式
    if (pw->rfmon_selfstart)
    {
        PacketSetMonitorMode(p->opt.device, 0);
    }
    // 调用通用的清理函数
    pcap_cleanup_live_common(p);
}

// 中断捕获循环
static void
pcap_breakloop_npf(pcap_t *p)
{
    // 调用通用的中断捕获循环函数
    pcap_breakloop_common(p);
    // 获取 pcap_win 结构体指针
    struct pcap_win *pw = p->priv;

    /* XXX - 如果这里失败会怎么样？ */
    // 设置事件以中断读取
    SetEvent(PacketGetReadEvent(pw->adapter));
}
/*
 * 这些是 NTSTATUS 值：
 *    https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/87fba13e-bf06-450e-83b1-9241dc81e781
 * 其中设置了“Customer”位。如果驱动程序返回它们，则在用户空间不会将它们映射到 Windows 错误值；它们将由 GetLastError() 返回。
 *
 * 注意，“驱动程序”在这里包括 Npcap NPF 驱动程序，因为各种版本会获取 NT 状态值并在返回状态码之前设置“Customer”位。开始执行此操作的更改的提交消息是
 *    在 OID 请求中返回了用户定义的 NTSTATUS 以避免 NTSTATUS 到 Win32 错误代码的转换。
 * 但我不知道目标是避免该转换的原因。
 *
 * 尝试在 Microsoft Surface Pro 的移动宽带适配器上设置硬件过滤器会返回一个错误，该错误似乎是 NDIS_STATUS_NOT_SUPPORTED 与“Customer”位 OR 运算的结果，因此可能表示它不支持该功能。
 *
 * 很可能还有其他设备会抛出虚假错误，到那时，这将需要重构以有效地针对列表进行检查，但目前我们只需检查这个值。也许正确的做法是将各种带有“customer”位 OR 运算的 NDIS 错误进行比较。
 */
#define NT_STATUS_CUSTOMER_DEFINED    0x20000000

static int
pcap_activate_npf(pcap_t *p)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_win *pw = p->priv;
    // 网络类型
    NetType type;
    // 结果
    int res;
    // 状态
    int status = 0;
    // 总指令
    struct bpf_insn total_insn;
    // 总程序
    struct bpf_program total_prog;
    # 如果设置了监控模式
    if (p->opt.rfmon) {
        # 检查当前设备是否支持监控模式
        if (PacketGetMonitorMode(p->opt.device) == 1)
        {
            # 如果支持监控模式，则关闭自启动监控模式
            pw->rfmon_selfstart = 0;
        }
        else
        {
            # 如果不支持监控模式，则尝试设置监控模式
            if ((res = PacketSetMonitorMode(p->opt.device, 1)) != 1)
            {
                # 设置失败，关闭自启动监控模式
                pw->rfmon_selfstart = 0;
                # 如果不支持监控模式，则返回监控模式不支持错误
                if (res == 0)
                {
                    return PCAP_ERROR_RFMON_NOTSUP;
                }
                else
                {
                    return PCAP_ERROR;
                }
            }
            else
            {
                # 设置成功，开启自启动监控模式
                pw->rfmon_selfstart = 1;
            }
        }
    }

    # 如果 Winsock 没有初始化，则进行初始化
    pcap_wsockinit();

    # 打开适配器
    pw->adapter = PacketOpenAdapter(p->opt.device);

    # 如果适配器打开失败
    if (pw->adapter == NULL)
    {
        // 获取最近一次发生的错误代码
        DWORD errcode = GetLastError();

        /*
         * 尝试打开适配器时出现了什么错误？
         */
        switch (errcode) {

        case ERROR_BAD_UNIT:
            /*
             * 没有这样的设备。
             * 没有要添加的内容，所以清除错误消息。
             */
            p->errbuf[0] = '\0';
            return (PCAP_ERROR_NO_SUCH_DEVICE);

        case ERROR_ACCESS_DENIED:
            /*
             * 有这样的设备，但我们没有权限使用它。
             *
             * XXX - 如果用户对 UAC 提示说“否”，我们目前会得到 ERROR_BAD_UNIT。
             */
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "The helper program for \"Admin-only Mode\" must be allowed to make changes to your device");
            return (PCAP_ERROR_PERM_DENIED);

        default:
            /*
             * 未知错误 - 报告详细信息。
             */
            pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
                errcode, "Error opening adapter");
            if (pw->rfmon_selfstart)
            {
                PacketSetMonitorMode(p->opt.device, 0);
            }
            return (PCAP_ERROR);
        }
    }

    /*获取网络类型*/
    if(PacketGetNetType (pw->adapter,&type) == FALSE)
    {
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "Cannot determine the network type");
        goto bad;
    }

    /*设置链路类型*/
    switch (type.LinkType)
    {
    /*
     * NDIS-defined medium types.
     */
    case NdisMedium802_3:
        # 设置链路类型为以太网
        p->linktype = DLT_EN10MB;
        /*
         * 这是（大概）一个真正的以太网捕获；给它一个链路层类型列表，包括DLT_EN10MB和DLT_DOCSIS，这样一个应用程序可以让你选择它，以防你捕获了一个 Cisco Cable Modem Termination System 放在以太网上的 DOCSIS 流量（它不会在电线上放置以太网头，而是在低级以太网封装中放置原始的 DOCSIS 帧）。
         */
        p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
        /*
         * 如果分配内存失败，就让列表保持为空。
         */
        if (p->dlt_list != NULL) {
            p->dlt_list[0] = DLT_EN10MB;
            p->dlt_list[1] = DLT_DOCSIS;
            p->dlt_count = 2;
        }
        break;

    case NdisMedium802_5:
        /*
         * 令牌环网。
         */
        p->linktype = DLT_IEEE802;
        break;

    case NdisMediumFddi:
        # 设置链路类型为FDDI
        p->linktype = DLT_FDDI;
        break;

    case NdisMediumWan:
        # 设置链路类型为以太网
        p->linktype = DLT_EN10MB;
        break;

    case NdisMediumArcnetRaw:
        # 设置链路类型为ARCNET
        p->linktype = DLT_ARCNET;
        break;

    case NdisMediumArcnet878_2:
        # 设置链路类型为ARCNET
        p->linktype = DLT_ARCNET;
        break;

    case NdisMediumAtm:
        # 设置链路类型为ATM_RFC1483
        p->linktype = DLT_ATM_RFC1483;
        break;

    case NdisMediumWirelessWan:
        # 设置链路类型为原始数据
        p->linktype = DLT_RAW;
        break;

    case NdisMediumIP:
        # 设置链路类型为原始数据
        p->linktype = DLT_RAW;
        break;

    /*
     * Npcap-defined medium types.
     */
    case NdisMediumNull:
        # 设置链路类型为NULL
        p->linktype = DLT_NULL;
        break;

    case NdisMediumCHDLC:
        # 设置链路类型为CHDLC
        p->linktype = DLT_CHDLC;
        break;

    case NdisMediumPPPSerial:
        # 设置链路类型为PPP_SERIAL
        p->linktype = DLT_PPP_SERIAL;
        break;

    case NdisMediumBare80211:
        # 设置链路类型为IEEE802_11
        p->linktype = DLT_IEEE802_11;
        break;

    case NdisMediumRadio80211:
        # 设置链路类型为IEEE802_11_RADIO
        p->linktype = DLT_IEEE802_11_RADIO;
        break;
    # 如果是 NdisMediumPpi 类型的网络介质，设置链路类型为 DLT_PPI
    case NdisMediumPpi:
        p->linktype = DLT_PPI;
        break;

    # 如果是其他未知类型的网络介质，默认设置链路类型为 DLT_EN10MB
    default:
        /*
         * 假设未知的介质类型提供以太网头部；
         * 如果不是，用户将不得不报告它，
         * 以便确定介质类型和链路层头部类型。
         * 如果我们在这里失败，可能会在错误中得到链路层类型，
         * 但用户将无法获得捕获，因此我们无法确定链路层类型；
         * 我们报告一个带有链路层类型的警告，因此至少一些程序将报告警告。
         */
        p->linktype = DLT_EN10MB;
        # 设置错误缓冲区中的错误消息
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Unknown NdisMedium value %d, defaulting to DLT_EN10MB",
            type.LinkType);
        # 设置状态为警告
        status = PCAP_WARNING;
        break;
    }
#ifdef HAVE_PACKET_GET_TIMESTAMP_MODES
    /*
     * 如果定义了 HAVE_PACKET_GET_TIMESTAMP_MODES，则设置时间戳类型。
     * （是的，我们需要 PacketGetTimestampModes()，而不仅仅是 PacketSetTimestampMode()。
     * 如果我们有前者，我们就有后者，除非有人使用一个版本的 Npcap，他们已经修改以提供前者但没有后者；
     * 如果他们这样做了，要么他们困惑了，要么他们在和我们开玩笑。）
     */
    switch (p->opt.tstamp_type) {

    case PCAP_TSTAMP_HOST_HIPREC_UNSYNCED:
        /*
         * 比低分辨率好，但*不*与操作系统时钟同步。
         */
        if (!PacketSetTimestampMode(pw->adapter, TIMESTAMPMODE_SINGLE_SYNCHRONIZATION))
        {
            pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
                GetLastError(), "Cannot set the time stamp mode to TIMESTAMPMODE_SINGLE_SYNCHRONIZATION");
            goto bad;
        }
        break;

    case PCAP_TSTAMP_HOST_LOWPREC:
        /*
         * 低分辨率，但与操作系统时钟同步。
         */
        if (!PacketSetTimestampMode(pw->adapter, TIMESTAMPMODE_QUERYSYSTEMTIME))
        {
            pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
                GetLastError(), "Cannot set the time stamp mode to TIMESTAMPMODE_QUERYSYSTEMTIME");
            goto bad;
        }
        break;

    case PCAP_TSTAMP_HOST_HIPREC:
        /*
         * 高分辨率，并与操作系统时钟同步。
         */
        if (!PacketSetTimestampMode(pw->adapter, TIMESTAMPMODE_QUERYSYSTEMTIME_PRECISE))
        {
            pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
                GetLastError(), "Cannot set the time stamp mode to TIMESTAMPMODE_QUERYSYSTEMTIME_PRECISE");
            goto bad;
        }
        break;
    case PCAP_TSTAMP_HOST:
        /*
         * 对于 PCAP_TSTAMP_HOST 类型的时间戳，暂时执行默认操作。
         * 设置为与系统时钟同步的最高分辨率？
         */
        break;
    }
#endif /* HAVE_PACKET_GET_TIMESTAMP_MODES */

    /*
     * 将负的快照值（无效）、快照值为0（未指定）或大于正常最大值的值，转换为允许的最大值。
     *
     * 如果某些应用程序真的*需要*更大的快照长度，我们应该增加MAXIMUM_SNAPLEN。
     */
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    /* 设置混杂模式 */
    if (p->opt.promisc)
    }
    else
    {
        /*
         * NDIS_PACKET_TYPE_ALL_LOCAL选择“由安装的协议发送的所有数据包和网卡指示的所有数据包”，
         * 但如果没有安装协议驱动程序（如TCP/IP），则需要NDIS_PACKET_TYPE_DIRECTED、
         * NDIS_PACKET_TYPE_BROADCAST和NDIS_PACKET_TYPE_MULTICAST来捕获传入的帧。
         */
        if (PacketSetHwFilter(pw->adapter,
            NDIS_PACKET_TYPE_ALL_LOCAL |
            NDIS_PACKET_TYPE_DIRECTED |
            NDIS_PACKET_TYPE_BROADCAST |
            NDIS_PACKET_TYPE_MULTICAST) == FALSE)
        {
            DWORD errcode = GetLastError();

            /*
             * 抑制非兼容的MS Surface移动适配器生成的错误。
             */
            if (errcode != (NDIS_STATUS_NOT_SUPPORTED|NT_STATUS_CUSTOMER_DEFINED))
            {
                pcap_fmt_errmsg_for_win32_err(p->errbuf,
                    PCAP_ERRBUF_SIZE, errcode,
                    "failed to set hardware filter to non-promiscuous mode");
                goto bad;
            }
        }
    }

    /* 设置缓冲区大小 */
    p->bufsize = WIN32_DEFAULT_USER_BUFFER_SIZE;

    if(!(pw->adapter->Flags & INFO_FLAG_DAG_CARD))
    {
    /*
     * 传统适配器
     */
        /*
         * 如果缓冲区大小没有明确设置，则默认为 WIN32_DEFAULT_KERNEL_BUFFER_SIZE。
         */
        if (p->opt.buffer_size == 0)
            p->opt.buffer_size = WIN32_DEFAULT_KERNEL_BUFFER_SIZE;

        // 设置适配器的缓冲区大小
        if(PacketSetBuff(pw->adapter,p->opt.buffer_size)==FALSE)
        {
            // 如果内存不足以分配内核缓冲区，则设置错误信息并跳转到错误处理
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "driver error: not enough memory to allocate the kernel buffer");
            goto bad;
        }

        // 分配缓冲区内存
        p->buffer = malloc(p->bufsize);
        if (p->buffer == NULL)
        {
            // 如果内存分配失败，则设置错误信息并跳转到错误处理
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            goto bad;
        }

        // 如果设置了立即模式
        if (p->opt.immediate)
        {
            /* 告诉驱动程序一旦数据到达就立即复制缓冲区 */
            if(PacketSetMinToCopy(pw->adapter,0)==FALSE)
            {
                // 如果调用 PacketSetMinToCopy 出错，则设置错误信息并跳转到错误处理
                pcap_fmt_errmsg_for_win32_err(p->errbuf,
                    PCAP_ERRBUF_SIZE, GetLastError(),
                    "Error calling PacketSetMinToCopy");
                goto bad;
            }
        }
        else
        {
            /* 告诉驱动程序只有缓冲区至少包含 16K 时才复制缓冲区 */
            if(PacketSetMinToCopy(pw->adapter,16000)==FALSE)
            {
                // 如果调用 PacketSetMinToCopy 出错，则设置错误信息并跳转到错误处理
                pcap_fmt_errmsg_for_win32_err(p->errbuf,
                    PCAP_ERRBUF_SIZE, GetLastError(),
                    "Error calling PacketSetMinToCopy");
                goto bad;
            }
        }
    } else {
        /*
         * Dag Card
         */
#ifdef HAVE_DAG_API
        /*
         * We have DAG support.
         */
        // 定义变量
        LONG    status;
        HKEY    dagkey;
        DWORD    lptype;
        DWORD    lpcbdata;
        int        postype = 0;
        char    keyname[512];

        // 构建注册表键名
        snprintf(keyname, sizeof(keyname), "%s\\CardParams\\%s",
            "SYSTEM\\CurrentControlSet\\Services\\DAG",
            strstr(_strlwr(p->opt.device), "dag"));
        do
        {
            // 打开注册表键
            status = RegOpenKeyEx(HKEY_LOCAL_MACHINE, keyname, 0, KEY_READ, &dagkey);
            if(status != ERROR_SUCCESS)
                break;

            // 查询注册表键值
            status = RegQueryValueEx(dagkey,
                "PosType",
                NULL,
                &lptype,
                (char*)&postype,
                &lpcbdata);

            if(status != ERROR_SUCCESS)
            {
                postype = 0;
            }

            // 关闭注册表键
            RegCloseKey(dagkey);
        }
        while(FALSE);

        // 设置快照长度
        p->snapshot = PacketSetSnapLen(pw->adapter, p->snapshot);

        /* Set the length of the FCS associated to any packet. This value
         * will be subtracted to the packet length */
        // 设置FCS长度
        pw->dag_fcs_bits = pw->adapter->DagFcsLen;
#else /* HAVE_DAG_API */
        /*
         * No DAG support.
         */
        // 没有DAG支持，跳转到bad标签
        goto bad;
#endif /* HAVE_DAG_API */
    }

    /*
     * If there's no filter program installed, there's
     * no indication to the kernel of what the snapshot
     * length should be, so no snapshotting is done.
     *
     * Therefore, when we open the device, we install
     * an "accept everything" filter with the specified
     * snapshot length.
     */
    // 设置BPF指令
    total_insn.code = (u_short)(BPF_RET | BPF_K);
    total_insn.jt = 0;
    total_insn.jf = 0;
    total_insn.k = p->snapshot;

    total_prog.bf_len = 1;
    total_prog.bf_insns = &total_insn;
    # 如果 PacketSetBpf 函数返回 false，则设置错误消息并返回错误状态
    if (!PacketSetBpf(pw->adapter, &total_prog)) {
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "PacketSetBpf");
        status = PCAP_ERROR;
        goto bad;
    }

    # 设置读取超时时间
    PacketSetReadTimeout(pw->adapter, p->opt.timeout);

    # 如果请求禁用回环捕获
    if (p->opt.nocapture_local)
    {
        # 如果禁用回环捕获失败，则设置错误消息并返回错误状态
        if (!PacketSetLoopbackBehavior(pw->adapter, NPF_DISABLE_LOOPBACK))
        {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Unable to disable the capture of loopback packets.");
            goto bad;
        }
    }
#ifdef HAVE_DAG_API
    # 如果编译时包含 DAG API，则执行以下代码块
    if(pw->adapter->Flags & INFO_FLAG_DAG_CARD)
    {
        # 如果适配器标志包含 INFO_FLAG_DAG_CARD，则执行以下代码块
        /* install dag specific handlers for read and setfilter */
        # 为读取和设置过滤器安装特定于 DAG 卡的处理程序
        p->read_op = pcap_read_win32_dag;
        # 设置读取操作为 pcap_read_win32_dag 函数
        p->setfilter_op = pcap_setfilter_win32_dag;
        # 设置设置过滤器操作为 pcap_setfilter_win32_dag 函数
    }
    else
    {
#endif /* HAVE_DAG_API */
        # 如果不包含 DAG API，则执行以下代码块
        /* install traditional npf handlers for read and setfilter */
        # 为读取和设置过滤器安装传统的 npf 处理程序
        p->read_op = pcap_read_npf;
        # 设置读取操作为 pcap_read_npf 函数
        p->setfilter_op = pcap_setfilter_npf;
        # 设置设置过滤器操作为 pcap_setfilter_npf 函数
#ifdef HAVE_DAG_API
    }
#endif /* HAVE_DAG_API */
    # 结束编译时包含 DAG API 的条件编译
    p->setdirection_op = NULL;    /* Not implemented. */
    # 设置方向操作为 NULL，表示未实现
        /* XXX - can this be implemented on some versions of Windows? */
    # XXX - 这在某些 Windows 版本上可以实现吗？
    p->inject_op = pcap_inject_npf;
    # 设置注入操作为 pcap_inject_npf 函数
    p->set_datalink_op = NULL;    /* can't change data link type */
    # 设置数据链路类型操作为 NULL，表示无法更改数据链路类型
    p->getnonblock_op = pcap_getnonblock_npf;
    # 设置获取非阻塞操作为 pcap_getnonblock_npf 函数
    p->setnonblock_op = pcap_setnonblock_npf;
    # 设置设置非阻塞操作为 pcap_setnonblock_npf 函数
    p->stats_op = pcap_stats_npf;
    # 设置统计操作为 pcap_stats_npf 函数
    p->breakloop_op = pcap_breakloop_npf;
    # 设置中断循环操作为 pcap_breakloop_npf 函数
    p->stats_ex_op = pcap_stats_ex_npf;
    # 设置扩展统计操作为 pcap_stats_ex_npf 函数
    p->setbuff_op = pcap_setbuff_npf;
    # 设置设置缓冲区操作为 pcap_setbuff_npf 函数
    p->setmode_op = pcap_setmode_npf;
    # 设置设置模式操作为 pcap_setmode_npf 函数
    p->setmintocopy_op = pcap_setmintocopy_npf;
    # 设置设置最小拷贝操作为 pcap_setmintocopy_npf 函数
    p->getevent_op = pcap_getevent_npf;
    # 设置获取事件操作为 pcap_getevent_npf 函数
    p->oid_get_request_op = pcap_oid_get_request_npf;
    # 设置 OID 获取请求操作为 pcap_oid_get_request_npf 函数
    p->oid_set_request_op = pcap_oid_set_request_npf;
    # 设置 OID 设置请求操作为 pcap_oid_set_request_npf 函数
    p->sendqueue_transmit_op = pcap_sendqueue_transmit_npf;
    # 设置发送队列传输操作为 pcap_sendqueue_transmit_npf 函数
    p->setuserbuffer_op = pcap_setuserbuffer_npf;
    # 设置设置用户缓冲区操作为 pcap_setuserbuffer_npf 函数
    p->live_dump_op = pcap_live_dump_npf;
    # 设置实时转储操作为 pcap_live_dump_npf 函数
    p->live_dump_ended_op = pcap_live_dump_ended_npf;
    # 设置实时转储结束操作为 pcap_live_dump_ended_npf 函数
    p->get_airpcap_handle_op = pcap_get_airpcap_handle_npf;
    # 设置获取 AirPcap 句柄操作为 pcap_get_airpcap_handle_npf 函数
    p->cleanup_op = pcap_cleanup_npf;
    # 设置清理操作为 pcap_cleanup_npf 函数

    /*
     * XXX - this is only done because WinPcap supported
     * pcap_fileno() returning the hFile HANDLE from the
     * ADAPTER structure.  We make no general guarantees
     * that the caller can do anything useful with it.
     *
     * (Not that we make any general guarantee of that
     * sort on UN*X, either, any more, given that not
     * all capture devices are regular OS network
     * interfaces.)
     */
    # XXX - 这仅是因为 WinPcap 支持 pcap_fileno() 返回 ADAPTER 结构中的 hFile 句柄。我们不做任何一般性的保证，调用者可以对其进行任何有用的操作。
    #
    # （不是说我们在 UN*X 上做出了任何一般性的保证，因为并非所有捕获设备都是常规的操作系统网络接口。）
    p->handle = pw->adapter->hFile;
    # 设置句柄为 pw->adapter->hFile

    return (status);
bad:
    # 调用pcap_cleanup_npf函数，清理资源
    pcap_cleanup_npf(p);
    # 返回PCAP_ERROR错误码
    return (PCAP_ERROR);
/*
* 检查 Windows 系统上的 pcap_t 是否支持 rfmon 模式。
*/
static int
pcap_can_set_rfmon_npf(pcap_t *p)
{
    // 检查设备是否支持监控模式
    return (PacketIsMonitorModeSupported(p->opt.device) == 1);
}

/*
 * 获取时间戳类型列表。
 */
#ifdef HAVE_PACKET_GET_TIMESTAMP_MODES
static int
get_ts_types(const char *device, pcap_t *p, char *ebuf)
{
    char *device_copy = NULL;
    ADAPTER *adapter = NULL;
    ULONG num_ts_modes;
    BOOL ret;
    DWORD error = ERROR_SUCCESS;
    ULONG *modes = NULL;
    int status = 0;

    // 代码块结束标记
    } while (0);

    /* 清理临时分配的内存 */
    if (device_copy != NULL) {
        free(device_copy);
    }
    if (modes != NULL) {
        free(modes);
    }
    if (adapter != NULL) {
        PacketCloseAdapter(adapter);
    }

    return status;
}
#else /* HAVE_PACKET_GET_TIMESTAMP_MODES */
static int
get_ts_types(const char *device _U_, pcap_t *p _U_, char *ebuf _U_)
{
    /*
     * 没有需要获取的内容，因此始终“成功”。
     */
    return 0;
}
#endif /* HAVE_PACKET_GET_TIMESTAMP_MODES */

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    pcap_t *p;

    // 创建通用的 pcap_t 结构
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_win);
    if (p == NULL)
        return (NULL);

    // 设置激活操作为 pcap_activate_npf
    p->activate_op = pcap_activate_npf;
    // 设置可以设置 rfmon 模式的操作为 pcap_can_set_rfmon_npf

    if (get_ts_types(device, p, ebuf) == -1) {
        pcap_close(p);
        return (NULL);
    }
    return (p);
}

static int
pcap_setfilter_npf(pcap_t *p, struct bpf_program *fp)
{
    struct pcap_win *pw = p->priv;
    # 如果将 BPF 程序设置到适配器失败
    if(PacketSetBpf(pw->adapter,fp)==FALSE){
        '''
         * 内核过滤器未安装。
         *
         * XXX - 我们不知道这失败是因为：
         *
         *  内核拒绝了过滤程序，因为它无效，
         *  在这种情况下，我们应该回退到用户空间过滤；
         *
         *  内核拒绝了过滤程序，因为它太大，
         *  在这种情况下，我们应该再次回退到
         *  用户空间过滤；
         *
         *  还有其他问题，这种情况下我们
         *  应该报告一个错误。
         *
         * 对于 NPF 设备，无效过滤器的 Win32 状态将是
         * STATUS_INVALID_DEVICE_REQUEST，但我不知道其他问题的状态，
         * 对于一些其他设备，它可能根本没有设置。
         *
         * 所以我们在所有情况下都回退到用户空间过滤。
         '''

        '''
         * install_bpf_program() 验证程序的有效性。
         *
         * XXX - 如果我们已经在内核中有一个过滤器怎么办？
         '''
        # 如果安装 BPF 程序失败，返回 -1
        if (install_bpf_program(p, fp) < 0)
            return (-1);
        pw->filtering_in_kernel = 0;    /* 在用户空间进行过滤 */
        return (0);
    }

    '''
     * 它起作用了。
     '''
    pw->filtering_in_kernel = 1;    /* 在内核中进行过滤 */

    '''
     * 丢弃先前接收到的数据包，因为它们可能通过以前生效的任何过滤器，
     * 但可能不会通过这个过滤器（BIOCSETF 会丢弃内核缓冲的数据包，
     * 所以无论如何都可能会丢失数据包）。
     '''
    p->cc = 0;
    return (0);
}

/*
 * 在用户级别进行过滤，因为内核驱动程序不处理数据包
 */
static int
pcap_setfilter_win32_dag(pcap_t *p, struct bpf_program *fp) {

    if(!fp)
    {
        pcap_strlcpy(p->errbuf, "setfilter: No filter specified", sizeof(p->errbuf));
        return (-1);
    }

    /* 安装用户级别的过滤器 */
    if (install_bpf_program(p, fp) < 0)
        return (-1);

    return (0);
}

static int
pcap_getnonblock_npf(pcap_t *p)
{
    struct pcap_win *pw = p->priv;

    /*
     * XXX - 如果有 PacketGetReadTimeout() 调用，我们将使用它，
     * 如果超时为-1，则返回1，否则返回0。
     */
    return (pw->nonblock);
}

static int
pcap_setnonblock_npf(pcap_t *p, int nonblock)
{
    struct pcap_win *pw = p->priv;
    int newtimeout;

    if (nonblock) {
        /*
         * 将数据包缓冲区超时设置为-1，用于非阻塞模式。
         */
        newtimeout = -1;
    } else {
        /*
         * 恢复设备打开时设置的超时。
         * （请注意，这可能是-1，这样我们实际上并没有离开非阻塞模式。
         * 但是，尽管 pcap_set_timeout() 和 pcap_open_live() 的超时参数是 int 类型，
         * 但你不应该提供负值，所以“不应该发生”。）
         */
        newtimeout = p->opt.timeout;
    }
    if (!PacketSetReadTimeout(pw->adapter, newtimeout)) {
        pcap_fmt_errmsg_for_win32_err(p->errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "PacketSetReadTimeout");
        return (-1);
    }
    pw->nonblock = (newtimeout == -1);
    return (0);
}

static int
pcap_add_if_npf(pcap_if_list_t *devlistp, char *name, bpf_u_int32 flags,
    const char *description, char *errbuf)
{
    pcap_if_t *curdev;
    npf_if_addr if_addrs[MAX_NETWORK_ADDRESSES];
    LONG if_addr_size;
    int res = 0;

    if_addr_size = MAX_NETWORK_ADDRESSES;
    /*
     * 为这个接口添加一个条目，没有地址。
     */
    curdev = add_dev(devlistp, name, flags, description, errbuf);
    if (curdev == NULL) {
        /*
         * 失败。
         */
        return (-1);
    }

    /*
     * 获取接口的地址列表。
     */
    if (!PacketGetNetInfoEx((void *)name, if_addrs, &if_addr_size)) {
        /*
         * 失败。
         *
         * 我们不返回错误，因为这可能发生在 NdisWan 接口上，
         * 即使我们无法提供它们的地址，我们仍希望提供它们。
         *
         * 我们返回一个带有空地址列表的条目。
         */
        return (0);
    }

    /*
     * 现在添加地址。
     */
    while (if_addr_size-- > 0) {
        /*
         * "curdev" 是这个接口的条目；为其地址列表添加这个地址的条目。
         */
        res = add_addr_to_dev(curdev,
            (struct sockaddr *)&if_addrs[if_addr_size].IPAddress,
            sizeof (struct sockaddr_storage),
            (struct sockaddr *)&if_addrs[if_addr_size].SubnetMask,
            sizeof (struct sockaddr_storage),
            (struct sockaddr *)&if_addrs[if_addr_size].Broadcast,
            sizeof (struct sockaddr_storage),
            NULL,
            0,
            errbuf);
        if (res == -1) {
            /*
             * 失败。
             */
            break;
        }
    }

    return (res);
    # 获取网络接口的标志位
static int
get_if_flags(const char *name, bpf_u_int32 *flags, char *errbuf)
{
    char *name_copy;  # 定义一个字符串指针变量
    ADAPTER *adapter;  # 定义一个适配器指针变量
    int status;  # 定义一个整型变量
    size_t len;  # 定义一个长度变量
    NDIS_HARDWARE_STATUS hardware_status;  # 定义一个硬件状态变量
#ifdef OID_GEN_PHYSICAL_MEDIUM
    NDIS_PHYSICAL_MEDIUM phys_medium;  # 定义一个物理介质变量
    bpf_u_int32 gen_physical_medium_oids[] = {  # 定义一个 OID 数组
  #ifdef OID_GEN_PHYSICAL_MEDIUM_EX
        OID_GEN_PHYSICAL_MEDIUM_EX,  # 如果定义了 OID_GEN_PHYSICAL_MEDIUM_EX，则添加到数组中
  #endif
        OID_GEN_PHYSICAL_MEDIUM  # 添加 OID_GEN_PHYSICAL_MEDIUM 到数组中
    };
#define N_GEN_PHYSICAL_MEDIUM_OIDS    (sizeof gen_physical_medium_oids / sizeof gen_physical_medium_oids[0])  # 定义一个宏，计算 OID 数组的长度
    size_t i;  # 定义一个索引变量
#endif /* OID_GEN_PHYSICAL_MEDIUM */
#ifdef OID_GEN_LINK_STATE
    NDIS_LINK_STATE link_state;  # 定义一个链路状态变量
#endif
    int connect_status;  # 定义一个连接状态变量

    if (*flags & PCAP_IF_LOOPBACK) {  # 如果标志位中包含 PCAP_IF_LOOPBACK
        /*
         * Loopback interface, so the connection status doesn't
         * apply. and it's not wireless (or wired, for that
         * matter...).  We presume it's up and running.
         */
        *flags |= PCAP_IF_UP | PCAP_IF_RUNNING | PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;  # 设置标志位为上线、运行和连接状态不适用
        return (0);  # 返回 0
    }

    /*
     * We need to open the adapter to get this information.
     *
     * XXX - PacketOpenAdapter() takes a non-const pointer
     * as an argument, so we make a copy of the argument and
     * pass that to it.
     */
    name_copy = strdup(name);  # 复制接口名称
    adapter = PacketOpenAdapter(name_copy);  # 打开适配器
    free(name_copy);  # 释放复制的接口名称
    if (adapter == NULL) {  # 如果适配器为空
        /*
         * Give up; if they try to open this device, it'll fail.
         */
        return (0);  # 返回 0
    }

#ifdef HAVE_AIRPCAP_API
    /*
     * Airpcap.sys do not support the below 'OID_GEN_x' values.
     * Just set these flags (and none of the '*flags' entered with).
     */
    # 如果成功获取到AirPcap句柄
    if (PacketGetAirPcapHandle(adapter)) {
        # 设置标志位为“up”和“running”
        *flags = PCAP_IF_UP | PCAP_IF_RUNNING;

        # 设置标志位为无线设备
        *flags |= PCAP_IF_WIRELESS;

        # 设置标志位为“网络关联状态”不适用于AirPcap
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        
        # 关闭适配器
        PacketCloseAdapter(adapter);
        
        # 返回0表示成功
        return (0);
    }
#endif

    /*
     * 获取硬件状态，并从中推导出 "up" 和 "running"。
     */
    len = sizeof (hardware_status);
    status = oid_get_request(adapter, OID_GEN_HARDWARE_STATUS,
        &hardware_status, &len, errbuf);
    if (status == 0) {
        switch (hardware_status) {

        case NdisHardwareStatusReady:
            /*
             * "可用并能够在网络上传输和接收数据"，因此是 up 和 running。
             */
            *flags |= PCAP_IF_UP | PCAP_IF_RUNNING;
            break;

        case NdisHardwareStatusInitializing:
        case NdisHardwareStatusReset:
            /*
             * "初始化" 或 "重置"，因此是 up，但不是 running。
             */
            *flags |= PCAP_IF_UP;
            break;

        case NdisHardwareStatusClosing:
        case NdisHardwareStatusNotReady:
            /*
             * "关闭" 或 "未准备好"，因此既不是 up 也不是 running。
             */
            break;

        default:
            /*
             * 未知状态。
             */
            break;
        }
    } else {
        /*
         * 无法获取硬件状态，因此假设同时是 up 和 running。
         */
        *flags |= PCAP_IF_UP | PCAP_IF_RUNNING;
    }

    /*
     * 获取网络类型。
     */
#ifdef OID_GEN_PHYSICAL_MEDIUM
    /*
     * 尝试按顺序使用我们拥有的 OIDs。
     */
    for (i = 0; i < N_GEN_PHYSICAL_MEDIUM_OIDS; i++) {
        len = sizeof (phys_medium);
        status = oid_get_request(adapter, gen_physical_medium_oids[i],
            &phys_medium, &len, errbuf);
        if (status == 0) {
            /*
             * 成功。
             */
            break;
        }
        /*
         * 失败。我们无法确定失败是因为该特定的 OID 不受支持，还是因为发生了其他问题，因此我们继续尝试下一个 OID。
         */
    }
    // 如果状态为0，表示获取到了物理介质
    if (status == 0) {
        /*
         * 我们得到了物理介质。
         *
         * XXX - 我们可能想要检查NdisPhysicalMediumWiMax和NdisPhysicalMediumNative802_15_4
         * 是否在枚举中，并在“无线”情况下检查它们。
         */
DIAG_OFF_ENUM_SWITCH
        // 关闭诊断信息输出

        switch (phys_medium) {
            // 根据物理介质类型进行切换

        case NdisPhysicalMediumWirelessLan:
        case NdisPhysicalMediumWirelessWan:
        case NdisPhysicalMediumNative802_11:
        case NdisPhysicalMediumBluetooth:
        case NdisPhysicalMediumUWB:
        case NdisPhysicalMediumIrda:
            /*
             * 无线连接。
             */
            *flags |= PCAP_IF_WIRELESS;
            // 设置标志表示为无线连接
            break;

        default:
            /*
             * 非无线连接或未知
             */
            break;
        }
DIAG_ON_ENUM_SWITCH
    }
#endif

    /*
     * 获取连接状态。
     */
#ifdef OID_GEN_LINK_STATE
    len = sizeof(link_state);
    status = oid_get_request(adapter, OID_GEN_LINK_STATE, &link_state,
        &len, errbuf);
    if (status == 0) {
        /*
         * 注意：这也给出了接收和发送的连接状态。
         */
        switch (link_state.MediaConnectState) {

        case MediaConnectStateConnected:
            /*
             * 已连接。
             */
            *flags |= PCAP_IF_CONNECTION_STATUS_CONNECTED;
            // 设置标志表示为已连接
            break;

        case MediaConnectStateDisconnected:
            /*
             * 已断开连接。
             */
            *flags |= PCAP_IF_CONNECTION_STATUS_DISCONNECTED;
            // 设置标志表示为已断开连接
            break;

        case MediaConnectStateUnknown:
        default:
            /*
             * 未知是否已连接。
             */
            break;
        }
    }
#else
    /*
     * OID_GEN_LINK_STATE 不受支持，因为它不在我们的 SDK 中。
     */
    status = -1;
    // 设置状态为 -1，表示不支持
#endif
    if (status == -1) {
        /*
         * 如果 OID_GEN_LINK_STATE 没有起作用，尝试使用 OID_GEN_MEDIA_CONNECT_STATUS。
         */
        status = oid_get_request(adapter, OID_GEN_MEDIA_CONNECT_STATUS,
            &connect_status, &len, errbuf);
        if (status == 0) {
            switch (connect_status) {

            case NdisMediaStateConnected:
                /*
                 * 已连接。
                 */
                *flags |= PCAP_IF_CONNECTION_STATUS_CONNECTED;
                break;

            case NdisMediaStateDisconnected:
                /*
                 * 已断开连接。
                 */
                *flags |= PCAP_IF_CONNECTION_STATUS_DISCONNECTED;
                break;
            }
        }
    }
    PacketCloseAdapter(adapter);
    return (0);
}
// 查找网络适配器列表的函数
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    int ret = 0;  // 返回值初始化为0
    const char *desc;  // 适配器描述信息
    char *AdaptersName;  // 适配器名称
    ULONG NameLength;  // 适配器名称长度
    char *name;  // 适配器名称

    /*
     * Find out how big a buffer we need.
     *
     * This call should always return FALSE; if the error is
     * ERROR_INSUFFICIENT_BUFFER, NameLength will be set to
     * the size of the buffer we need, otherwise there's a
     * problem, and NameLength should be set to 0.
     *
     * It shouldn't require NameLength to be set, but,
     * at least as of WinPcap 4.1.3, it checks whether
     * NameLength is big enough before it checks for a
     * NULL buffer argument, so, while it'll still do
     * the right thing if NameLength is uninitialized and
     * whatever junk happens to be there is big enough
     * (because the pointer argument will be null), it's
     * still reading an uninitialized variable.
     */
    NameLength = 0;  // 初始化适配器名称长度为0
    if (!PacketGetAdapterNames(NULL, &NameLength))  // 获取适配器名称列表
    {
        DWORD last_error = GetLastError();  // 获取最后一个错误代码

        if (last_error != ERROR_INSUFFICIENT_BUFFER)  // 如果错误不是缓冲区不足
        {
            pcap_fmt_errmsg_for_win32_err(errbuf, PCAP_ERRBUF_SIZE,
                last_error, "PacketGetAdapterNames");  // 格式化错误消息
            return (-1);  // 返回-1
        }
    }

    if (NameLength <= 0)  // 如果适配器名称长度小于等于0
        return 0;  // 返回0
    AdaptersName = (char*) malloc(NameLength);  // 分配适配器名称内存空间
    if (AdaptersName == NULL)  // 如果分配失败
    {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "Cannot allocate enough memory to list the adapters.");  // 格式化错误消息
        return (-1);  // 返回-1
    }

    if (!PacketGetAdapterNames(AdaptersName, &NameLength)) {  // 获取适配器名称列表
        pcap_fmt_errmsg_for_win32_err(errbuf, PCAP_ERRBUF_SIZE,
            GetLastError(), "PacketGetAdapterNames");  // 格式化错误消息
        free(AdaptersName);  // 释放内存
        return (-1);  // 返回-1
    }
    /*
     * "PacketGetAdapterNames()"返回一个以空字符结尾的ASCII接口名称字符串列表，
     * 后面跟着一个以空字符串结尾的ASCII接口描述字符串列表。
     * 这意味着第一个列表的末尾有两个ASCII空字符。
     *
     * 找到第一个列表的末尾；这就是第二个列表的开始。
     */
    desc = &AdaptersName[0];
    while (*desc != '\0' || *(desc + 1) != '\0')
        desc++;

    /*
     * 找到了 - "desc"指向第一个列表末尾的两个空字符之一，所以描述列表的第一个字节在它之后两个字节处。
     */
    desc += 2;

    /*
     * 遍历第一个列表中的元素。
     */
    name = &AdaptersName[0];
    while (*name != '\0') {
        bpf_u_int32 flags = 0;
#ifdef HAVE_AIRPCAP_API
        /*
         * 如果这是一个 AirPcap 设备？
         * 如果是的话，忽略它；它将在后面由 AirPcap 代码添加。
         */
        if (device_is_airpcap(name, errbuf) == 1) {
            name += strlen(name) + 1;
            desc += strlen(desc) + 1;
            continue;
        }
#endif

#ifdef HAVE_PACKET_IS_LOOPBACK_ADAPTER
        /*
         * 这是一个环回接口吗？
         */
        if (PacketIsLoopbackAdapter(name)) {
            /* 是的 */
            flags |= PCAP_IF_LOOPBACK;
        }
#endif
        /*
         * 获取额外的标志。
         */
        if (get_if_flags(name, &flags, errbuf) == -1) {
            /*
             * 失败。
             */
            ret = -1;
            break;
        }

        /*
         * 为这个接口添加一个条目。
         */
        if (pcap_add_if_npf(devlistp, name, flags, desc,
            errbuf) == -1) {
            /*
             * 失败。
             */
            ret = -1;
            break;
        }
        name += strlen(name) + 1;
        desc += strlen(desc) + 1;
    }

    free(AdaptersName);
    return (ret);
}

/*
 * 返回连接到系统的网络接口的名称，如果找不到则返回 NULL。
 * 接口必须已配置为启用；首选最低单元编号；忽略环回接口。
 *
 * 在最理想的情况下，这将与 UN*X 上的情况相同，但可能有一些软件期望在第一个设备之后返回完整的设备列表。
 */
#define ADAPTERSNAME_LEN    8192
char *
pcap_lookupdev(char *errbuf)
{
    DWORD dwVersion;
    DWORD dwWindowsMajorVersion;
    /*
     * 在“新 API”模式下禁用此功能，因为：
     * 1）在 WinPcap/Npcap 中，它可能返回 UTF-16 字符串，出于向后兼容的原因，
     * 我们也禁用了使其工作的 hack，以避免超出字符串末尾的原因；
     * 2）我们希望它的行为是一致的。
     *
     * 此外，它不是线程安全的，因此我们已将其标记为已弃用。
     */
    if (pcap_new_api) {
        // 如果处于新 API 模式下，则返回错误信息并返回空指针
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "pcap_lookupdev() is deprecated and is not supported in programs calling pcap_init()");
        return (NULL);
    }
/* disable MSVC's GetVersion() deprecated warning here */
DIAG_OFF_DEPRECATION
    // 获取操作系统版本信息
    dwVersion = GetVersion();    /* get the OS version */
DIAG_ON_DEPRECATION
    // 获取 Windows 主版本号
    dwWindowsMajorVersion = (DWORD)(LOBYTE(LOWORD(dwVersion)));

    // 如果操作系统版本大于等于 Windows 95
    if (dwVersion >= 0x80000000 && dwWindowsMajorVersion >= 4) {
        /*
         * Windows 95, 98, ME.
         */
        ULONG NameLength = ADAPTERSNAME_LEN;
        static char AdaptersName[ADAPTERSNAME_LEN];

        // 获取适配器名称
        if (PacketGetAdapterNames(AdaptersName,&NameLength) )
            return (AdaptersName);
        else
            return NULL;
    quit:
        // 释放内存并返回适配器名称
        free(TAdaptersName);
        return (char *)(AdaptersName);
    }
}

/*
 * We can't use the same code that we use on UN*X, as that's doing
 * UN*X-specific calls.
 *
 * We don't just fetch the entire list of devices, search for the
 * particular device, and use its first IPv4 address, as that's too
 * much work to get just one device's netmask.
 */
int
pcap_lookupnet(const char *device, bpf_u_int32 *netp, bpf_u_int32 *maskp,
    char *errbuf)
{
    /*
     * We need only the first IPv4 address, so we must scan the array returned by PacketGetNetInfo()
     * in order to skip non IPv4 (i.e. IPv6 addresses)
     */
    // 定义变量
    npf_if_addr if_addrs[MAX_NETWORK_ADDRESSES];
    LONG if_addr_size = MAX_NETWORK_ADDRESSES;
    struct sockaddr_in *t_addr;
    LONG i;

    // 获取网络信息
    if (!PacketGetNetInfoEx((void *)device, if_addrs, &if_addr_size)) {
        *netp = *maskp = 0;
        return (0);
    }

    // 遍历网络地址数组
    for(i = 0; i < if_addr_size; i++)
    {
        // 如果是 IPv4 地址
        if(if_addrs[i].IPAddress.ss_family == AF_INET)
        {
            t_addr = (struct sockaddr_in *) &(if_addrs[i].IPAddress);
            *netp = t_addr->sin_addr.S_un.S_addr;
            t_addr = (struct sockaddr_in *) &(if_addrs[i].SubnetMask);
            *maskp = t_addr->sin_addr.S_un.S_addr;

            *netp &= *maskp;
            return (0);
        }

    }

    *netp = *maskp = 0;
    return (0);
}

static const char *pcap_lib_version_string;
#ifdef HAVE_VERSION_H
/*
 * libpcap being built for Windows, as part of a WinPcap/Npcap source
 * tree.  Include version.h from that source tree to get the WinPcap/Npcap
 * version.
 *
 * XXX - it'd be nice if we could somehow generate the WinPcap/Npcap version
 * number when building as part of WinPcap/Npcap.  (It'd be nice to do so
 * for the packet.dll version number as well.)
 */
#include "../../version.h"

static const char pcap_version_string[] =
    WINPCAP_PRODUCT_NAME " version " WINPCAP_VER_STRING ", based on " PCAP_VERSION_STRING;

const char *
pcap_lib_version(void)
{
    if (pcap_lib_version_string == NULL) {
        /*
         * Generate the version string.
         */
        const char *packet_version_string = PacketGetVersion();

        if (strcmp(WINPCAP_VER_STRING, packet_version_string) == 0) {
            /*
             * WinPcap/Npcap version string and packet.dll version
             * string are the same; just report the WinPcap/Npcap
             * version.
             */
            pcap_lib_version_string = pcap_version_string;
        } else {
            /*
             * WinPcap/Npcap version string and packet.dll version
             * string are different; that shouldn't be the
             * case (the two libraries should come from the
             * same version of WinPcap/Npcap), so we report both
             * versions.
             */
            char *full_pcap_version_string;

            if (pcap_asprintf(&full_pcap_version_string,
                WINPCAP_PRODUCT_NAME " version " WINPCAP_VER_STRING " (packet.dll version %s), based on " PCAP_VERSION_STRING,
                packet_version_string) != -1) {
                /* Success */
                pcap_lib_version_string = full_pcap_version_string;
            }
        }
    }
    return (pcap_lib_version_string);
}

#else /* HAVE_VERSION_H */

/*
 * libpcap being built for Windows, not as part of a WinPcap/Npcap source
 * tree.
 */
const char *
pcap_lib_version(void)
{
    // 检查 pcap_lib_version_string 是否为空
    if (pcap_lib_version_string == NULL) {
        /*
         * 生成版本字符串。报告 packet.dll 的版本。
         */
        // 声明 full_pcap_version_string 变量
        char *full_pcap_version_string;

        // 使用 pcap_asprintf 函数生成完整的 pcap 版本字符串，包括 packet.dll 的版本
        if (pcap_asprintf(&full_pcap_version_string,
            PCAP_VERSION_STRING " (packet.dll version %s)",
            PacketGetVersion()) != -1) {
            /* 成功 */
            // 将生成的完整版本字符串赋值给 pcap_lib_version_string
            pcap_lib_version_string = full_pcap_version_string;
        }
    }
    // 返回 pcap_lib_version_string
    return (pcap_lib_version_string);
}
#endif /* HAVE_VERSION_H */
```