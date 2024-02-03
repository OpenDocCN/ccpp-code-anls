# `nmap\libpcap\pcap-airpcap.c`

```cpp
/*
 * 版权声明，版权所有
 */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "pcap-int.h"

#include <airpcap.h>

#include "pcap-airpcap.h"

/* 用户空间分配的默认缓冲区大小 */
#define    AIRPCAP_DEFAULT_USER_BUFFER_SIZE 256000

/* AirPcap 适配器的默认缓冲区大小 */
// 定义默认的 AIRPCAP 内核缓冲区大小为 1000000
#define AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE 1000000

//
// 我们动态加载 AirPcap DLL，这样代码就可以在安装或未安装 AirPcap 的情况下工作，
// 而且不需要有两个不同版本的库，一个链接到 AirPcap 库，另一个不链接到它。
//
static pcap_code_handle_t airpcap_lib;

// 定义 AirpcapGetLastErrorHandler 函数指针类型
typedef PCHAR (*AirpcapGetLastErrorHandler)(PAirpcapHandle);
// 定义 AirpcapGetDeviceListHandler 函数指针类型
typedef BOOL (*AirpcapGetDeviceListHandler)(PAirpcapDeviceDescription *, PCHAR);
// 定义 AirpcapFreeDeviceListHandler 函数指针类型
typedef VOID (*AirpcapFreeDeviceListHandler)(PAirpcapDeviceDescription);
// 定义 AirpcapOpenHandler 函数指针类型
typedef PAirpcapHandle (*AirpcapOpenHandler)(PCHAR, PCHAR);
// 定义 AirpcapCloseHandler 函数指针类型
typedef VOID (*AirpcapCloseHandler)(PAirpcapHandle);
// 定义 AirpcapSetDeviceMacFlagsHandler 函数指针类型
typedef BOOL (*AirpcapSetDeviceMacFlagsHandler)(PAirpcapHandle, UINT);
// 定义 AirpcapSetLinkTypeHandler 函数指针类型
typedef BOOL (*AirpcapSetLinkTypeHandler)(PAirpcapHandle, AirpcapLinkType);
// 定义 AirpcapGetLinkTypeHandler 函数指针类型
typedef BOOL (*AirpcapGetLinkTypeHandler)(PAirpcapHandle, PAirpcapLinkType);
// 定义 AirpcapSetKernelBufferHandler 函数指针类型
typedef BOOL (*AirpcapSetKernelBufferHandler)(PAirpcapHandle, UINT);
// 定义 AirpcapSetFilterHandler 函数指针类型
typedef BOOL (*AirpcapSetFilterHandler)(PAirpcapHandle, PVOID, UINT);
// 定义 AirpcapSetMinToCopyHandler 函数指针类型
typedef BOOL (*AirpcapSetMinToCopyHandler)(PAirpcapHandle, UINT);
// 定义 AirpcapGetReadEventHandler 函数指针类型
typedef BOOL (*AirpcapGetReadEventHandler)(PAirpcapHandle, HANDLE *);
// 定义 AirpcapReadHandler 函数指针类型
typedef BOOL (*AirpcapReadHandler)(PAirpcapHandle, PBYTE, UINT, PUINT);
// 定义 AirpcapWriteHandler 函数指针类型
typedef BOOL (*AirpcapWriteHandler)(PAirpcapHandle, PCHAR, ULONG);
// 定义 AirpcapGetStatsHandler 函数指针类型
typedef BOOL (*AirpcapGetStatsHandler)(PAirpcapHandle, PAirpcapStats);

// 定义 Airpcap 库函数指针
static AirpcapGetLastErrorHandler p_AirpcapGetLastError;
static AirpcapGetDeviceListHandler p_AirpcapGetDeviceList;
static AirpcapFreeDeviceListHandler p_AirpcapFreeDeviceList;
static AirpcapOpenHandler p_AirpcapOpen;
static AirpcapCloseHandler p_AirpcapClose;
static AirpcapSetDeviceMacFlagsHandler p_AirpcapSetDeviceMacFlags;
static AirpcapSetLinkTypeHandler p_AirpcapSetLinkType;
static AirpcapGetLinkTypeHandler p_AirpcapGetLinkType;
static AirpcapSetKernelBufferHandler p_AirpcapSetKernelBuffer;
static AirpcapSetFilterHandler p_AirpcapSetFilter;
static AirpcapSetMinToCopyHandler p_AirpcapSetMinToCopy;
# 声明一个指向 AirpcapGetReadEventHandler 函数的指针变量
static AirpcapGetReadEventHandler p_AirpcapGetReadEvent;
# 声明一个指向 AirpcapReadHandler 函数的指针变量
static AirpcapReadHandler p_AirpcapRead;
# 声明一个指向 AirpcapWriteHandler 函数的指针变量
static AirpcapWriteHandler p_AirpcapWrite;
# 声明一个指向 AirpcapGetStatsHandler 函数的指针变量
static AirpcapGetStatsHandler p_AirpcapGetStats;

# 定义一个枚举类型，表示 Airpcap API 的加载状态
typedef enum LONG
{
    AIRPCAP_API_UNLOADED = 0,  # 未加载
    AIRPCAP_API_LOADED,        # 已加载
    AIRPCAP_API_CANNOT_LOAD,   # 无法加载
    AIRPCAP_API_LOADING        # 加载中
} AIRPCAP_API_LOAD_STATUS;

# 声明一个静态变量，表示 Airpcap API 的加载状态
static AIRPCAP_API_LOAD_STATUS    airpcap_load_status;

# 定义一个静态函数，用于加载 Airpcap 相关函数
static AIRPCAP_API_LOAD_STATUS
load_airpcap_functions(void)
{
    AIRPCAP_API_LOAD_STATUS current_status;

    # 检查当前加载状态并进行相应处理
    current_status = InterlockedCompareExchange((LONG *)&airpcap_load_status,
        AIRPCAP_API_LOADING, AIRPCAP_API_UNLOADED);
    /*
     * 如果状态为AIRPCAP_API_UNLOADED，则将其设置为AIRPCAP_API_LOADING，因为我们将加载库，但current_status为AIRPCAP_API_UNLOADED。
     * 如果状态为AIRPCAP_API_LOADING，表示其他线程正在尝试加载库，循环等待直到其完成，并将状态设置为反映其是否成功的值。
     */
    while (current_status == AIRPCAP_API_LOADING) {
        current_status = InterlockedCompareExchange((LONG*)&airpcap_load_status,
            AIRPCAP_API_LOADING, AIRPCAP_API_LOADING);
        Sleep(10);
    }

    /*
     * 此时，current_status可能是：
     *    AIRPCAP_API_LOADED，表示其他线程已加载库，完成；
     *    AIRPCAP_API_CANNOT_LOAD，表示其他线程尝试加载库但失败，完成 - 我们不会再尝试加载；
     *    AIRPCAP_API_LOADING，表示我们正在加载库，现在应该尝试加载。
     */
    if (current_status == AIRPCAP_API_LOADED)
        return AIRPCAP_API_LOADED;

    if (current_status == AIRPCAP_API_CANNOT_LOAD)
        return AIRPCAP_API_CANNOT_LOAD;

    /*
     * 假设我们无法加载它。
     */
    current_status = AIRPCAP_API_CANNOT_LOAD;

    airpcap_lib = pcap_load_code("airpcap.dll");
    }

    if (current_status != AIRPCAP_API_LOADED) {
        /*
         * 加载失败；如果找到了DLL，则关闭其句柄。
         */
        if (airpcap_lib != NULL) {
            FreeLibrary(airpcap_lib);
            airpcap_lib = NULL;
        }
    }

    /*
     * 现在适当地设置状态 - 并且是原子操作。
     */
    InterlockedExchange((LONG *)&airpcap_load_status, current_status);

    return current_status;
    */
}

/*
 * 用于在 AirPcap 设备上捕获数据的私有数据。
 */
struct pcap_airpcap {
    PAirpcapHandle adapter;  // AirPcap 适配器句柄
    int filtering_in_kernel;  // 是否在内核中进行过滤
    int nonblock;  // 是否为非阻塞模式
    int read_timeout;  // 读取超时时间
    HANDLE read_event;  // 读取事件句柄
    struct pcap_stat stat;  // 统计信息
};

static int
airpcap_setfilter(pcap_t *p, struct bpf_program *fp)
{
    struct pcap_airpcap *pa = p->priv;  // 获取私有数据结构体指针

    if (!p_AirpcapSetFilter(pa->adapter, fp->bf_insns,
        fp->bf_len * sizeof(struct bpf_insn))) {
        /*
         * 内核过滤器未安装。
         *
         * XXX - 我们不知道这失败是因为：
         *
         *  内核拒绝了过滤程序，因为它是无效的，
         *  在这种情况下，我们应该回退到用户空间过滤；
         *
         *  内核拒绝了过滤程序，因为它太大，
         *  在这种情况下，我们应该再次回退到用户空间过滤；
         *
         *  还有其他问题，这种情况下，我们可能应该报告一个错误；
         *
         * 因此，我们在所有情况下都回退到用户空间过滤。
         */

        /*
         * install_bpf_program() 验证程序。
         *
         * XXX - 如果我们已经在内核中有一个过滤器怎么办？
         */
        if (install_bpf_program(p, fp) < 0)
            return (-1);
        pa->filtering_in_kernel = 0;    /* 在用户空间进行过滤 */
        return (0);
    }

    /*
     * 它起作用了。
     */
    pa->filtering_in_kernel = 1;    /* 在内核中进行过滤 */

    /*
     * 丢弃先前接收的数据包，因为它们可能已经通过以前生效的任何过滤器，
     * 但可能不会通过这个过滤器（BIOCSETF 会丢弃内核中缓冲的数据包，所以无论如何都可能丢失数据包）。
     */
    p->cc = 0;
    return (0);
}

static int
airpcap_set_datalink(pcap_t *p, int dlt)
{
    struct pcap_airpcap *pa = p->priv;  // 获取私有数据结构体指针
    AirpcapLinkType type;

    switch (dlt) {
    # 如果数据链路类型为 DLT_IEEE802_11_RADIO，则设置为 AIRPCAP_LT_802_11_PLUS_RADIO
    case DLT_IEEE802_11_RADIO:
        type = AIRPCAP_LT_802_11_PLUS_RADIO;
        break;

    # 如果数据链路类型为 DLT_PPI，则设置为 AIRPCAP_LT_802_11_PLUS_PPI
    case DLT_PPI:
        type = AIRPCAP_LT_802_11_PLUS_PPI;
        break;

    # 如果数据链路类型为 DLT_IEEE802_11，则设置为 AIRPCAP_LT_802_11
    case DLT_IEEE802_11:
        type = AIRPCAP_LT_802_11;
        break;

    # 如果数据链路类型不在以上三种情况，则返回 0
    default:
        /* This can't happen; just return. */
        return (0);
    }

    # 如果设置数据链路类型失败，则返回 -1，并设置错误信息到 errbuf
    if (!p_AirpcapSetLinkType(pa->adapter, type)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapSetLinkType() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (-1);
    }

    # 设置数据链路类型为 dlt
    p->linktype = dlt;
    # 返回 0，表示成功
    return (0);
# 获取当前 pcap_t 对象的非阻塞状态
static int
airpcap_getnonblock(pcap_t *p)
{
    # 获取 pcap_t 对象的私有数据结构 pcap_airpcap
    struct pcap_airpcap *pa = p->priv;

    # 返回私有数据结构中的非阻塞状态
    return (pa->nonblock);
}

# 设置当前 pcap_t 对象的非阻塞状态
static int
airpcap_setnonblock(pcap_t *p, int nonblock)
{
    # 获取 pcap_t 对象的私有数据结构 pcap_airpcap
    struct pcap_airpcap *pa = p->priv;
    int newtimeout;

    if (nonblock) {
        '''
        如果设置为非阻塞模式，则将数据包缓冲区超时设置为-1。
        '''
        newtimeout = -1;
    } else {
        '''
        恢复设备打开时设置的超时时间。
        （请注意，这可能是-1，如果是-1，则实际上并没有离开非阻塞模式。
        但是，尽管 pcap_set_timeout() 和 pcap_open_live() 的超时参数是一个 int，
        但你不应该提供一个负值，所以“不应该发生”。）
        '''
        newtimeout = p->opt.timeout;
    }
    # 更新读取超时时间
    pa->read_timeout = newtimeout;
    # 更新非阻塞状态
    pa->nonblock = (newtimeout == -1);
    return (0);
}

# 获取当前 pcap_t 对象的统计信息
static int
airpcap_stats(pcap_t *p, struct pcap_stat *ps)
{
    # 获取 pcap_t 对象的私有数据结构 pcap_airpcap
    struct pcap_airpcap *pa = p->priv;
    AirpcapStats tas;

    '''
    尝试获取统计信息。
    如果 p_AirpcapGetStats() 失败，则设置错误信息到 errbuf 中并返回 -1。
    '''
    if (!p_AirpcapGetStats(pa->adapter, &tas)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetStats() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (-1);
    }

    # 将获取到的统计信息设置到传入的结构体中
    ps->ps_drop = tas.Drops;
    ps->ps_recv = tas.Recvs;
    ps->ps_ifdrop = tas.IfDrops;

    return (0);
}
/*
 * 仅适用于 Win32 的获取统计信息的例程。
 *
 * 这种方式绝对比从用户空间传递 pcap_stat * 更安全。
 * 实际上，可能发生用户分配的变量不够大以容纳新结构体，库将会写入未分配给该变量的区域。
 *
 * 通过这种方式，我们可以确保我们在为该变量分配的内存上进行写入。
 *
 * 但这种方式处理统计信息是错误的。相反，我们应该有一个 API，以类似于 pcapng 接口统计块的选项部分的形式返回数据：
 *
 *    https://xml2rfc.tools.ietf.org/cgi-bin/xml2rfc.cgi?url=https://raw.githubusercontent.com/pcapng/pcapng/master/draft-tuexen-opsawg-pcapng.xml&modeAsFormat=html/ascii&type=ascii#rfc.section.4.6
 *
 * 这样可以让我们直接添加新的统计信息，并指示我们提供和不提供哪些统计信息，而不是不得不为我们无法提供的统计信息提供可能错误的值。
 */
static struct pcap_stat *
airpcap_stats_ex(pcap_t *p, int *pcap_stat_size)
{
    struct pcap_airpcap *pa = p->priv;
    AirpcapStats tas;

    *pcap_stat_size = sizeof (p->stat);

    /*
     * 尝试获取统计信息。
     */
    if (!p_AirpcapGetStats(pa->adapter, &tas)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetStats() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (NULL);
    }

    p->stat.ps_recv = tas.Recvs;
    p->stat.ps_drop = tas.Drops;
    p->stat.ps_ifdrop = tas.IfDrops;
    /*
     * 以防万一，如果这个代码被编译到除了 Windows 之外的目标平台，这几乎是极不可能的。
     */
#ifdef _WIN32
    p->stat.ps_capt = tas.Capt;
#endif
    return (&p->stat);
}

/* 设置内核级捕获缓冲区的维度 */
static int
airpcap_setbuff(pcap_t *p, int dim)
{
    struct pcap_airpcap *pa = p->priv;
    # 如果调用 p_AirpcapSetKernelBuffer 函数失败，则执行以下操作
    if (!p_AirpcapSetKernelBuffer(pa->adapter, dim)) {
        # 格式化错误信息到 p->errbuf 中
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapSetKernelBuffer() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        # 返回 -1，表示失败
        return (-1);
    }
    # 如果 p_AirpcapSetKernelBuffer 函数调用成功，则返回 0，表示成功
    return (0);
/* 设置驱动程序的工作模式 */
static int
airpcap_setmode(pcap_t *p, int mode)
{
     // 如果模式不是捕获模式，则返回错误信息
     if (mode != MODE_CAPT) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Only MODE_CAPT is supported on an AirPcap adapter");
        return (-1);
     }
     return (0);
}

/* 设置释放读取调用的最小数据量 */
static int
airpcap_setmintocopy(pcap_t *p, int size)
{
    struct pcap_airpcap *pa = p->priv;

    // 如果设置最小数据量失败，则返回错误信息
    if (!p_AirpcapSetMinToCopy(pa->adapter, size)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapSetMinToCopy() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (-1);
    }
    return (0);
}

static HANDLE
airpcap_getevent(pcap_t *p)
{
    struct pcap_airpcap *pa = p->priv;

    // 返回读取事件句柄
    return (pa->read_event);
}

static int
airpcap_oid_get_request(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_,
    size_t *lenp _U_)
{
    // 返回不支持在AirPcap适配器上获取OID值的错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Getting OID values is not supported on an AirPcap adapter");
    return (PCAP_ERROR);
}

static int
airpcap_oid_set_request(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
    // 返回不支持在AirPcap适配器上设置OID值的错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Setting OID values is not supported on an AirPcap adapter");
    return (PCAP_ERROR);
}

static u_int
airpcap_sendqueue_transmit(pcap_t *p, pcap_send_queue *queue _U_, int sync _U_)
{
    // 返回不支持在AirPcap适配器上排队传输数据包的错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Cannot queue packets for transmission on an AirPcap adapter");
    return (0);
}

static int
airpcap_setuserbuffer(pcap_t *p, int size)
{
    unsigned char *new_buff;

    // 如果大小小于等于0，则返回错误信息
    if (size <= 0) {
        /* 无效的参数 */
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error: invalid size %d",size);
        return (-1);
    }

    /* 分配缓冲区 */
    new_buff = (unsigned char *)malloc(sizeof(char)*size);

    // 如果分配失败，则返回错误信息
    if (!new_buff) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Error: not enough memory");
        return (-1);
    # 释放 p 指针所指向的 buffer 内存
    free(p->buffer);
    # 将 new_buff 的地址赋给 p 指针所指向的 buffer
    p->buffer = new_buff;
    # 更新 p 指针所指向的 buffer 的大小为 size
    p->bufsize = size;
    # 返回 0，表示函数执行成功
    return (0);
}
// 用于在 AirPcap 适配器上进行实时抓包的函数，但是 AirPcap 适配器不支持实时抓包，因此直接返回错误
static int
airpcap_live_dump(pcap_t *p, char *filename _U_, int maxsize _U_,
    int maxpacks _U_)
{
    // 将错误信息写入错误缓冲区
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "AirPcap adapters don't support live dump");
    return (-1);
}

// 用于检查 AirPcap 适配器是否支持实时抓包，但是 AirPcap 适配器不支持实时抓包，因此直接返回错误
static int
airpcap_live_dump_ended(pcap_t *p, int sync _U_)
{
    // 将错误信息写入错误缓冲区
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "AirPcap adapters don't support live dump");
    return (-1);
}

// 获取 AirPcap 适配器的句柄
static PAirpcapHandle
airpcap_get_airpcap_handle(pcap_t *p)
{
    // 获取私有数据结构 pcap_airpcap 的指针
    struct pcap_airpcap *pa = p->priv;

    return (pa->adapter);
}

// 从 AirPcap 适配器上读取数据
static int
airpcap_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 获取私有数据结构 pcap_airpcap 的指针
    struct pcap_airpcap *pa = p->priv;
    int cc;
    int n;
    register u_char *bp, *ep;
    UINT bytes_read;
    u_char *datap;

    cc = p->cc;
    if (cc == 0) {
        /*
         * Has "pcap_breakloop()" been called?
         */
        if (p->break_loop) {
            /*
             * Yes - clear the flag that indicates that it
             * has, and return PCAP_ERROR_BREAK to indicate
             * that we were told to break out of the loop.
             */
            p->break_loop = 0;
            return (PCAP_ERROR_BREAK);
        }

        //
        // If we're not in non-blocking mode, wait for data to
        // arrive.
        //
        if (pa->read_timeout != -1) {
            WaitForSingleObject(pa->read_event,
                (pa->read_timeout ==0 )? INFINITE: pa->read_timeout);
        }

        //
        // Read the data.
        // p_AirpcapRead doesn't block.
        //
        if (!p_AirpcapRead(pa->adapter, (PBYTE)p->buffer,
            p->bufsize, &bytes_read)) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "AirpcapRead() failed: %s",
                p_AirpcapGetLastError(pa->adapter));
            return (-1);
        }
        cc = bytes_read;
        bp = (u_char *)p->buffer;
    } else
        bp = p->bp;
    /*
     * 循环遍历每个数据包。
     *
     * 这假设一个数据包的缓冲区将有 <= INT_MAX 个数据包，因此数据包计数不会溢出。
     */
#define bhp ((AirpcapBpfHeader *)bp)
    // 定义宏，将bp强制转换为AirpcapBpfHeader类型，并赋值给bhp

    n = 0;
    // 初始化计数器n为0
    ep = bp + cc;
    // 计算结束位置ep为bp加上cc

    for (;;) {
        // 无限循环

        register u_int caplen, hdrlen;
        // 声明并注册无符号整型变量caplen和hdrlen

        /*
         * Has "pcap_breakloop()" been called?
         * If so, return immediately - if we haven't read any
         * packets, clear the flag and return PCAP_ERROR_BREAK
         * to indicate that we were told to break out of the loop,
         * otherwise leave the flag set, so that the *next* call
         * will break out of the loop without having read any
         * packets, and return the number of packets we've
         * processed so far.
         */
        // 检查是否调用了"pcap_breakloop()"函数
        if (p->break_loop) {
            // 如果break_loop标志被设置
            if (n == 0) {
                // 如果还没有读取任何数据包
                p->break_loop = 0;
                // 清除break_loop标志
                return (PCAP_ERROR_BREAK);
                // 返回PCAP_ERROR_BREAK表示被告知跳出循环
            } else {
                p->bp = bp;
                p->cc = (int) (ep - bp);
                return (n);
                // 返回已处理的数据包数量
            }
        }
        if (bp >= ep)
            break;
        // 如果bp大于等于ep，跳出循环

        caplen = bhp->Caplen;
        // 获取数据包的捕获长度
        hdrlen = bhp->Hdrlen;
        // 获取数据包的头部长度
        datap = bp + hdrlen;
        // 数据指针指向数据包头部之后的位置

        /*
         * Short-circuit evaluation: if using BPF filter
         * in the AirPcap adapter, no need to do it now -
         * we already know the packet passed the filter.
         */
        // 短路评估：如果在AirPcap适配器中使用BPF过滤器，则无需现在进行过滤
        // 因为我们已经知道数据包通过了过滤器

        if (pa->filtering_in_kernel ||
            p->fcode.bf_insns == NULL ||
            pcap_filter(p->fcode.bf_insns, datap, bhp->Originallen, caplen)) {
            // 如果在内核中进行过滤或者bf_insns为空或者数据包通过了过滤器
            struct pcap_pkthdr pkthdr;
            // 声明pcap_pkthdr结构体变量pkthdr

            pkthdr.ts.tv_sec = bhp->TsSec;
            // 设置时间戳的秒部分
            pkthdr.ts.tv_usec = bhp->TsUsec;
            // 设置时间戳的微秒部分
            pkthdr.caplen = caplen;
            // 设置捕获长度
            pkthdr.len = bhp->Originallen;
            // 设置原始长度
            (*callback)(user, &pkthdr, datap);
            // 调用回调函数处理数据包
            bp += AIRPCAP_WORDALIGN(caplen + hdrlen);
            // 移动数据指针到下一个数据包的位置
            if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
                // 如果已处理的数据包数量大于等于cnt并且cnt不是无限制
                p->bp = bp;
                p->cc = (int)(ep - bp);
                return (n);
                // 返回已处理的数据包数量
            }
        } else {
            /*
             * Skip this packet.
             */
            // 跳过这个数据包
            bp += AIRPCAP_WORDALIGN(caplen + hdrlen);
            // 移动数据指针到下一个数据包的位置
        }
    }
#undef bhp
    # 将 bhp 宏定义取消，这里可能是为了避免重复定义
    p->cc = 0;
    # 将 p 指针指向的 cc 成员变量设置为 0
    return (n);
    # 返回 n 变量的值
}

static int
airpcap_inject(pcap_t *p, const void *buf, int size)
{
    struct pcap_airpcap *pa = p->priv;

    /*
     * XXX - the second argument to AirpcapWrite() *should* have been declared as a const pointer - a write function that stomps on what it writes is *extremely* rude - but such is life.  We assume it is, in fact, not going to write on our buffer.
     */
    # 对于 AirpcapWrite() 函数的第二个参数应该声明为 const 指针，但是实际上并没有这样声明，我们假设它实际上不会在我们的缓冲区上写入数据
    if (!p_AirpcapWrite(pa->adapter, (void *)buf, size)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapWrite() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (-1);
    }

    /*
     * We assume it all got sent if "AirpcapWrite()" succeeded.
     * "pcap_inject()" is expected to return the number of bytes sent.
     */
    # 如果 "AirpcapWrite()" 成功，我们假设所有数据都已发送。"pcap_inject()" 应该返回发送的字节数
    return (size);
    # 返回 size 变量的值
}

static void
airpcap_cleanup(pcap_t *p)
{
    struct pcap_airpcap *pa = p->priv;

    if (pa->adapter != NULL) {
        p_AirpcapClose(pa->adapter);
        pa->adapter = NULL;
    }
    pcap_cleanup_live_common(p);
}

static void
airpcap_breakloop(pcap_t *p)
{
    HANDLE read_event;

    pcap_breakloop_common(p);
    struct pcap_airpcap *pa = p->priv;

    /* XXX - what if either of these fail? */
    # 如果其中一个失败会怎么样？
    """
     * XXX - will SetEvent() force a wakeup and, if so, will the AirPcap read code handle that sanely?
     * SetEvent() 会强制唤醒吗？如果是，AirPcap 读取代码会如何处理？
     """
    if (!p_AirpcapGetReadEvent(pa->adapter, &read_event))
        return;
    SetEvent(read_event);
}

static int
airpcap_activate(pcap_t *p)
{
    struct pcap_airpcap *pa = p->priv;
    char *device = p->opt.device;
    char airpcap_errbuf[AIRPCAP_ERRBUF_SIZE];
    BOOL status;
    AirpcapLinkType link_type;

    pa->adapter = p_AirpcapOpen(device, airpcap_errbuf);
    if (pa->adapter == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "%s", airpcap_errbuf);
        return (PCAP_ERROR);
    }

    """
     * Set monitor mode appropriately.
     * Always turn off the "ACK frames sent to the card" mode.
     """
    # 适当设置监视模式。始终关闭“发送到卡的 ACK 帧”模式。
    # 如果设置了监控模式，则调用 p_AirpcapSetDeviceMacFlags 设置适配器的 MAC 标志
    if (p->opt.rfmon) {
        status = p_AirpcapSetDeviceMacFlags(pa->adapter,
            AIRPCAP_MF_MONITOR_MODE_ON);
    } else
        # 否则调用 p_AirpcapSetDeviceMacFlags 设置适配器的 ACK 帧标志
        status = p_AirpcapSetDeviceMacFlags(pa->adapter,
            AIRPCAP_MF_ACK_FRAMES_ON);
    # 如果设置失败，则关闭适配器，设置错误信息并返回错误
    if (!status) {
        p_AirpcapClose(pa->adapter);
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapSetDeviceMacFlags() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        return (PCAP_ERROR);
    }

    '''
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为允许的最大值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该只需增加 MAXIMUM_SNAPLEN。
     '''
    # 如果快照值小于等于 0 或大于最大快照长度，则将快照值设置为最大快照长度
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    '''
     * 如果缓冲区大小没有显式设置，则默认为 AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE。
     '''
    # 如果缓冲区大小为 0，则将其设置为默认内核缓冲区大小
    if (p->opt.buffer_size == 0)
        p->opt.buffer_size = AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE;

    # 如果设置内核缓冲区大小失败，则设置错误信息并跳转到 bad 标签
    if (!p_AirpcapSetKernelBuffer(pa->adapter, p->opt.buffer_size)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapSetKernelBuffer() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        goto bad;
    }

    # 获取读取事件，如果失败则设置错误信息并跳转到 bad 标签
    if(!p_AirpcapGetReadEvent(pa->adapter, &pa->read_event)) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetReadEvent() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        goto bad;
    }

    # 设置缓冲区大小，并分配内存
    p->bufsize = AIRPCAP_DEFAULT_USER_BUFFER_SIZE;
    p->buffer = malloc(p->bufsize);
    # 如果分配内存失败，则设置错误信息并跳转到 bad 标签
    if (p->buffer == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        goto bad;
    }
    if (p->opt.immediate) {
        /* 如果设置了立即模式，告诉驱动程序一旦数据到达就立即复制缓冲区 */
        if (!p_AirpcapSetMinToCopy(pa->adapter, 0)) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "AirpcapSetMinToCopy() failed: %s",
                p_AirpcapGetLastError(pa->adapter));
            goto bad;
        }
    } else {
        /*
         * 如果没有设置立即模式，告诉驱动程序只有缓冲区至少包含16K时才复制
         */
        if (!p_AirpcapSetMinToCopy(pa->adapter, 16000)) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "AirpcapSetMinToCopy() failed: %s",
                p_AirpcapGetLastError(pa->adapter));
            goto bad;
        }
    }

    /*
     * 获取默认的链路层头类型，并将 p->datalink 设置为该类型
     *
     * 我们不强制将其设置为其他值，因为可能有一些程序使用 WinPcap/Npcap，
     * 在 AirPcap 设备上捕获时，假设使用 AirPcap 配置程序设置的默认值
     */
    if (!p_AirpcapGetLinkType(pa->adapter, &link_type)) {
        /* 获取失败 */
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetLinkType() failed: %s",
            p_AirpcapGetLastError(pa->adapter));
        goto bad;
    }
    switch (link_type) {

    case AIRPCAP_LT_802_11_PLUS_RADIO:
        p->linktype = DLT_IEEE802_11_RADIO;
        break;

    case AIRPCAP_LT_802_11_PLUS_PPI:
        p->linktype = DLT_PPI;
        break;

    case AIRPCAP_LT_802_11:
        p->linktype = DLT_IEEE802_11;
        break;

    case AIRPCAP_LT_UNKNOWN:
    default:
        /* 未知类型 */
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetLinkType() returned unknown link type %u",
            link_type);
        goto bad;
    }
    /*
     * 现在提供所有支持的类型的列表；我们假设它们都有效。我们将 radiotap 放在最前面，
     * 然后是 PPI，然后是“无无线电元数据”。
     */
    // 分配一个包含3个 u_int 元素的内存空间
    p->dlt_list = (u_int *) malloc(sizeof(u_int) * 3);
    // 如果内存分配失败，则跳转到标签 bad
    if (p->dlt_list == NULL)
        goto bad;
    // 设置支持的数据链路类型列表
    p->dlt_list[0] = DLT_IEEE802_11_RADIO;
    p->dlt_list[1] = DLT_PPI;
    p->dlt_list[2] = DLT_IEEE802_11;
    // 设置支持的数据链路类型数量
    p->dlt_count = 3;

    // 设置读取操作的函数指针
    p->read_op = airpcap_read;
    // 设置数据注入操作的函数指针
    p->inject_op = airpcap_inject;
    // 设置过滤器操作的函数指针
    p->setfilter_op = airpcap_setfilter;
    // 设置数据流向操作的函数指针为 NULL，表示未实现
    p->setdirection_op = NULL;    /* Not implemented. */
    // 设置数据链路类型操作的函数指针
    p->set_datalink_op = airpcap_set_datalink;
    // 设置非阻塞模式获取操作的函数指针
    p->getnonblock_op = airpcap_getnonblock;
    // 设置非阻塞模式设置操作的函数指针
    p->setnonblock_op = airpcap_setnonblock;
    // 设置中断捕获操作的函数指针
    p->breakloop_op = airpcap_breakloop;
    // 设置统计信息获取操作的函数指针
    p->stats_op = airpcap_stats;
    // 设置扩展统计信息获取操作的函数指针
    p->stats_ex_op = airpcap_stats_ex;
    // 设置缓冲区大小设置操作的函数指针
    p->setbuff_op = airpcap_setbuff;
    // 设置模式设置操作的函数指针
    p->setmode_op = airpcap_setmode;
    // 设置最小拷贝操作的函数指针
    p->setmintocopy_op = airpcap_setmintocopy;
    // 设置事件获取操作的函数指针
    p->getevent_op = airpcap_getevent;
    // 设置 OID 获取请求操作的函数指针
    p->oid_get_request_op = airpcap_oid_get_request;
    // 设置 OID 设置请求操作的函数指针
    p->oid_set_request_op = airpcap_oid_set_request;
    // 设置发送队列传输操作的函数指针
    p->sendqueue_transmit_op = airpcap_sendqueue_transmit;
    // 设置用户缓冲区设置操作的函数指针
    p->setuserbuffer_op = airpcap_setuserbuffer;
    // 设置实时捕获操作的函数指针
    p->live_dump_op = airpcap_live_dump;
    // 设置实时捕获结束操作的函数指针
    p->live_dump_ended_op = airpcap_live_dump_ended;
    // 设置获取 AirPcap 句柄操作的函数指针
    p->get_airpcap_handle_op = airpcap_get_airpcap_handle;
    // 设置清理操作的函数指针
    p->cleanup_op = airpcap_cleanup;

    // 返回 0，表示成功
    return (0);
 bad:
    // 调用清理操作
    airpcap_cleanup(p);
    // 返回错误码
    return (PCAP_ERROR);
}

/*
 * Monitor mode is supported.
 */
static int
airpcap_can_set_rfmon(pcap_t *p)
{
    return (1);  // 返回1表示支持监控模式
}

int
device_is_airpcap(const char *device, char *ebuf)
{
    static const char airpcap_prefix[] = "\\\\.\\airpcap";  // 定义AirPcap设备的前缀

    /*
     * We don't determine this by calling AirpcapGetDeviceList()
     * and looking at the list, as that appears to be a costly
     * operation.
     *
     * Instead, we just check whether it begins with "\\.\airpcap".
     */
    if (strncmp(device, airpcap_prefix, sizeof airpcap_prefix - 1) == 0) {
        /*
         * Yes, it's an AirPcap device.
         */
        return (1);  // 返回1表示是AirPcap设备
    }

    /*
     * No, it's not an AirPcap device.
     */
    return (0);  // 返回0表示不是AirPcap设备
}

pcap_t *
airpcap_create(const char *device, char *ebuf, int *is_ours)
{
    int ret;
    pcap_t *p;

    /*
     * This can be called before we've tried loading the library,
     * so do so if we haven't already tried to do so.
     */
    if (load_airpcap_functions() != AIRPCAP_API_LOADED) {
        /*
         * We assume this means that we don't have the AirPcap
         * software installed, which probably means we don't
         * have an AirPcap device.
         *
         * Don't treat that as an error.
         */
        *is_ours = 0;  // 将is_ours设置为0
        return (NULL);  // 返回空指针
    }

    /*
     * Is this an AirPcap device?
     */
    ret = device_is_airpcap(device, ebuf);  // 检查设备是否是AirPcap设备
    if (ret == 0) {
        /* No. */
        *is_ours = 0;  // 将is_ours设置为0
        return (NULL);  // 返回空指针
    }

    /*
     * Yes.
     */
    *is_ours = 1;  // 将is_ours设置为1
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_airpcap);  // 创建一个pcap_t对象
    if (p == NULL)
        return (NULL);  // 如果创建失败则返回空指针

    p->activate_op = airpcap_activate;  // 设置activate_op为airpcap_activate函数
    p->can_set_rfmon_op = airpcap_can_set_rfmon;  // 设置can_set_rfmon_op为airpcap_can_set_rfmon函数
    return (p);  // 返回pcap_t对象
}

/*
 * Add all AirPcap devices.
 */
int
airpcap_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
    AirpcapDeviceDescription *airpcap_devices, *airpcap_device;
    char airpcap_errbuf[AIRPCAP_ERRBUF_SIZE];
    /*
     * 在尝试加载库之前可以调用这个函数，
     * 如果我们还没有尝试加载库，那么就尝试加载库。
     */
    if (load_airpcap_functions() != AIRPCAP_API_LOADED) {
        /*
         * XXX - 除非错误是“没有这样的 DLL”，否则将其报告为错误，而不是“没有 AirPcap 设备”？
         */
        return (0);
    }

    if (!p_AirpcapGetDeviceList(&airpcap_devices, airpcap_errbuf)) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "AirpcapGetDeviceList() failed: %s", airpcap_errbuf);
        return (-1);
    }

    for (airpcap_device = airpcap_devices; airpcap_device != NULL;
        airpcap_device = airpcap_device->next) {
        if (add_dev(devlistp, airpcap_device->Name, 0,
            airpcap_device->Description, errbuf) == NULL) {
            /*
             * 失败。
             */
            p_AirpcapFreeDeviceList(airpcap_devices);
            return (-1);
        }
    }
    p_AirpcapFreeDeviceList(airpcap_devices);
    return (0);
# 闭合前面的函数定义
```