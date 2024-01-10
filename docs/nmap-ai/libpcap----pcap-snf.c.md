# `nmap\libpcap\pcap-snf.c`

```
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifndef _WIN32
#include <sys/param.h>
#endif /* !_WIN32 */

#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <limits.h> /* for INT_MAX */

#ifndef _WIN32
#include <netinet/in.h>
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#endif /* !_WIN32 */

#include <snf.h>
#if SNF_VERSION_API >= 0x0003
#define SNF_HAVE_INJECT_API
#endif

#include "pcap-int.h"
#include "pcap-snf.h"

/*
 * Private data for capturing on SNF devices.
 */
struct pcap_snf {
    snf_handle_t snf_handle; /* opaque device handle */
    snf_ring_t   snf_ring;   /* opaque device ring handle */
#ifdef SNF_HAVE_INJECT_API
    snf_inject_t snf_inj;    /* inject handle, if inject is used */
#endif
    int          snf_timeout;
    int          snf_boardnum;
};

static int
snf_set_datalink(pcap_t *p, int dlt)
{
    p->linktype = dlt;
    return (0);
}
// 获取 SNF 设备的统计信息
static int
snf_pcap_stats(pcap_t *p, struct pcap_stat *ps)
{
    struct snf_ring_stats stats;
    struct pcap_snf *snfps = p->priv;
    int rc;

    // 获取 SNF 设备的统计信息
    if ((rc = snf_ring_getstats(snfps->snf_ring, &stats))) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            rc, "snf_get_stats");
        return -1;
    }
    // 设置接收的数据包数量
    ps->ps_recv = stats.ring_pkt_recv + stats.ring_pkt_overflow;
    // 设置丢弃的数据包数量
    ps->ps_drop = stats.ring_pkt_overflow;
    // 设置接口丢弃的数据包数量
    ps->ps_ifdrop = stats.nic_pkt_overflow + stats.nic_pkt_bad;
    return 0;
}
// 清理 SNF 平台相关资源
static void
snf_platform_cleanup(pcap_t *p)
{
    struct pcap_snf *ps = p->priv;

#ifdef SNF_HAVE_INJECT_API
    if (ps->snf_inj)
        snf_inject_close(ps->snf_inj);
#endif
    snf_ring_close(ps->snf_ring);
    snf_close(ps->snf_handle);
    pcap_cleanup_live_common(p);
}
// 获取非阻塞模式
static int
snf_getnonblock(pcap_t *p)
{
    struct pcap_snf *ps = p->priv;

    return (ps->snf_timeout == 0);
}
// 设置非阻塞模式
static int
snf_setnonblock(pcap_t *p, int nonblock)
{
    struct pcap_snf *ps = p->priv;

    if (nonblock)
        ps->snf_timeout = 0;
    else {
        # 如果超时时间小于等于0，则设置为-1，表示永远不超时
        if (p->opt.timeout <= 0)
            ps->snf_timeout = -1; /* forever */
        # 否则，将超时时间设置为传入参数的超时时间
        else
            ps->snf_timeout = p->opt.timeout;
    }
    # 返回0表示成功
    return (0);
}
#define _NSEC_PER_SEC 1000000000

static inline
struct timeval
snf_timestamp_to_timeval(const int64_t ts_nanosec, const int tstamp_precision)
{
    // 定义时间结构体
    struct timeval tv;
    long tv_nsec;
    // 定义静态的零时间结构体
    const static struct timeval zero_timeval;

    // 如果时间戳为0，返回零时间结构体
    if (ts_nanosec == 0)
        return zero_timeval;

    // 计算秒数
    tv.tv_sec = ts_nanosec / _NSEC_PER_SEC;
    tv_nsec = (ts_nanosec % _NSEC_PER_SEC);

    /* libpcap expects tv_usec to be nanos if using nanosecond precision. */
    // 如果时间戳精度为纳秒，则微秒等于纳秒
    if (tstamp_precision == PCAP_TSTAMP_PRECISION_NANO)
        tv.tv_usec = tv_nsec;
    else
        tv.tv_usec = tv_nsec / 1000;

    // 返回时间结构体
    return tv;
}

static int
snf_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 获取私有数据结构
    struct pcap_snf *ps = p->priv;
    // 定义数据包头
    struct pcap_pkthdr hdr;
    int i, flags, err, caplen, n;
    struct snf_recv_req req;
    int nonblock, timeout;

    // 如果 pcap_t 为空，返回-1
    if (!p)
        return -1;

    /*
     * This can conceivably process more than INT_MAX packets,
     * which would overflow the packet count, causing it either
     * to look like a negative number, and thus cause us to
     * return a value that looks like an error, or overflow
     * back into positive territory, and thus cause us to
     * return a too-low count.
     *
     * Therefore, if the packet count is unlimited, we clip
     * it at INT_MAX; this routine is not expected to
     * process packets indefinitely, so that's not an issue.
     */
    // 如果数据包数量为无限，将其限制在INT_MAX以内
    if (PACKET_COUNT_IS_UNLIMITED(cnt))
        cnt = INT_MAX;

    // 初始化数据包数量
    n = 0;
    // 获取超时时间
    timeout = ps->snf_timeout;
    while (n < cnt) {
        /*
         * Has "pcap_breakloop()" been called?
         */
        # 检查是否调用了"pcap_breakloop()"
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (-2);
            } else {
                return (n);
            }
        }

        err = snf_ring_recv(ps->snf_ring, timeout, &req);

        if (err) {
            if (err == EBUSY || err == EAGAIN) {
                return (n);
            }
            else if (err == EINTR) {
                timeout = 0;
                continue;
            }
            else {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    PCAP_ERRBUF_SIZE, err, "snf_read");
                return -1;
            }
        }

        caplen = req.length;
        if (caplen > p->snapshot)
            caplen = p->snapshot;

        if ((p->fcode.bf_insns == NULL) ||
             pcap_filter(p->fcode.bf_insns, req.pkt_addr, req.length, caplen)) {
            hdr.ts = snf_timestamp_to_timeval(req.timestamp, p->opt.tstamp_precision);
            hdr.caplen = caplen;
            hdr.len = req.length;
            callback(user, &hdr, req.pkt_addr);
            n++;
        }

        /* After one successful packet is received, we won't block
        * again for that timeout. */
        # 在成功接收一个数据包后，不再为超时而阻塞
        if (timeout != 0)
            timeout = 0;
    }
    return (n);
}
# 定义一个静态函数snf_inject，用于向网络接口发送数据包
static int
snf_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
#ifdef SNF_HAVE_INJECT_API
    # 获取 pcap_t 结构体中的私有数据
    struct pcap_snf *ps = p->priv;
    int rc;
    # 如果 snf_inj 为空，则打开 snf_inject
    if (ps->snf_inj == NULL) {
        rc = snf_inject_open(ps->snf_boardnum, 0, &ps->snf_inj);
        # 如果打开失败，则返回错误信息
        if (rc) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                rc, "snf_inject_open");
            return (-1);
        }
    }

    # 发送数据包
    rc = snf_inject_send(ps->snf_inj, -1, 0, buf, size);
    # 如果发送成功，则返回发送的数据包大小
    if (!rc) {
        return (size);
    }
    # 如果发送失败，则返回错误信息
    else {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            rc, "snf_inject_send");
        return (-1);
    }
#else
    # 如果不支持发送数据包，则返回错误信息
    pcap_strlcpy(p->errbuf, "Sending packets isn't supported with this snf version",
        PCAP_ERRBUF_SIZE);
    return (-1);
#endif
}

# 定义一个静态函数snf_activate，用于激活网络接口
static int
snf_activate(pcap_t* p)
{
    # 获取 pcap_t 结构体中的私有数据
    struct pcap_snf *ps = p->priv;
    char *device = p->opt.device;
    const char *nr = NULL;
    int err;
    int flags = -1, ring_id = -1;

    # 如果设备为空，则返回错误信息
    if (device == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "device is NULL");
        return -1;
    }

    /* In Libpcap, we set pshared by default if NUM_RINGS is set to > 1.
     * Since libpcap isn't thread-safe */
    # 根据环境变量设置 flags
    if ((nr = getenv("SNF_FLAGS")) && *nr)
        flags = strtol(nr, NULL, 0);
    else if ((nr = getenv("SNF_NUM_RINGS")) && *nr && atoi(nr) > 1)
        flags = SNF_F_PSHARED;
    else
        nr = NULL;


        /* Allow pcap_set_buffer_size() to set dataring_size.
         * Default is zero which allows setting from env SNF_DATARING_SIZE.
         * pcap_set_buffer_size() is in bytes while snf_open() accepts values
         * between 0 and 1048576 in Megabytes. Values in this range are
         * mapped to 1MB.
         */
    // 使用 snf_open 函数打开 SNF 设备，获取错误码
    err = snf_open(ps->snf_boardnum,
            0, /* let SNF API parse SNF_NUM_RINGS, if set */
            NULL, /* default RSS, or use SNF_RSS_FLAGS env */
                        (p->opt.buffer_size > 0 && p->opt.buffer_size < 1048576) ? 1048576 : p->opt.buffer_size, /* default to SNF_DATARING_SIZE from env */
            flags, /* may want pshared */
            &ps->snf_handle);
    // 如果打开失败，格式化错误消息并返回 -1
    if (err != 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            err, "snf_open failed");
        return -1;
    }

    // 获取环的 ID，如果存在的话
    if ((nr = getenv("SNF_PCAP_RING_ID")) && *nr) {
        ring_id = (int) strtol(nr, NULL, 0);
    }
    // 使用 snf_ring_open_id 函数打开指定 ID 的环，获取错误码
    err = snf_ring_open_id(ps->snf_handle, ring_id, &ps->snf_ring);
    // 如果打开失败，格式化错误消息并返回 -1
    if (err != 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            err, "snf_ring_open_id(ring=%d) failed", ring_id);
        return -1;
    }

    /*
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为允许的最大值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     */
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    // 如果超时值小于等于 0，则设置为 -1，否则设置为指定的超时值
    if (p->opt.timeout <= 0)
        ps->snf_timeout = -1;
    else
        ps->snf_timeout = p->opt.timeout;

    // 启动 SNF 设备，获取错误码
    err = snf_start(ps->snf_handle);
    // 如果启动失败，格式化错误消息并返回 -1
    if (err != 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            err, "snf_start failed");
        return -1;
    }

    /*
     * "select()" and "poll()" don't work on snf descriptors.
     */
#ifndef _WIN32
    // 如果不是在 Windows 平台上
    p->selectable_fd = -1;
#endif /* !_WIN32 */
    // 设置链路类型为以太网
    p->linktype = DLT_EN10MB;
    // 设置读操作为 snf_read
    p->read_op = snf_read;
    // 设置注入操作为 snf_inject
    p->inject_op = snf_inject;
    // 设置过滤器操作为 install_bpf_program
    p->setfilter_op = install_bpf_program;
    // 设置方向操作为 NULL，未实现
    p->setdirection_op = NULL; /* Not implemented.*/
    // 设置数据链路类型操作为 snf_set_datalink
    p->set_datalink_op = snf_set_datalink;
    // 设置非阻塞读取操作为 snf_getnonblock
    p->getnonblock_op = snf_getnonblock;
    // 设置非阻塞写入操作为 snf_setnonblock
    p->setnonblock_op = snf_setnonblock;
    // 设置统计操作为 snf_pcap_stats
    p->stats_op = snf_pcap_stats;
    // 设置清理操作为 snf_platform_cleanup
    p->cleanup_op = snf_platform_cleanup;
#ifdef SNF_HAVE_INJECT_API
    // 如果支持注入 API，则设置 snf_inj 为 NULL
    ps->snf_inj = NULL;
#endif
    // 返回 0 表示成功
    return 0;
}

// 定义最大描述长度为 128
#define MAX_DESC_LENGTH 128
int
snf_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
    pcap_if_t *dev;
#ifdef _WIN32
    struct sockaddr_in addr;
#endif
    struct snf_ifaddrs *ifaddrs, *ifa;
    char name[MAX_DESC_LENGTH];
    char desc[MAX_DESC_LENGTH];
    int ret, allports = 0, merge = 0;
    const char *nr = NULL;

    // 初始化 SNF
    if (snf_init(SNF_VERSION_API)) {
        (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "snf_getifaddrs: snf_init failed");
        return (-1);
    }

    // 获取接口地址信息
    if (snf_getifaddrs(&ifaddrs) || ifaddrs == NULL)
    {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "snf_getifaddrs");
        return (-1);
    }
    // 获取环境变量 SNF_FLAGS 的值
    if ((nr = getenv("SNF_FLAGS")) && *nr) {
        errno = 0;
        // 将字符串转换为整数
        merge = strtol(nr, NULL, 0);
        if (errno) {
            (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "snf_getifaddrs: SNF_FLAGS is not a valid number");
            return (-1);
        }
        // 对 merge 进行位运算
        merge = merge & SNF_F_AGGREGATE_PORTMASK;
    }
#ifdef _WIN32
            /*
             * 在 Windows 上，根据设备名称填充 IP# 
             */
                        ret = inet_pton(AF_INET, dev->name, &addr.sin_addr);
                        if (ret == 1) {
                /*
                 * 成功将设备名称转换为 IPv4 地址。
                 */
                addr.sin_family = AF_INET;
                if (add_addr_to_dev(dev, &addr, sizeof(addr),
                    NULL, 0, NULL, 0, NULL, 0, errbuf) == -1)
                    return -1;
                        } else if (ret == -1) {
                /*
                 * 出错。
                 */
                pcap_fmt_errmsg_for_errno(errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "sinf_findalldevs inet_pton");
                                return -1;
                        }
#endif _WIN32
        }
    }
    snf_freeifaddrs(ifaddrs);
    /*
     * 如果启用了端口聚合，则创建一个 snfX 条目
     */
    if (merge) {
        /*
         * 添加一个新条目，包含所有端口的位掩码
         */
        (void)snprintf(name,MAX_DESC_LENGTH,"snf%d",allports);
        (void)snprintf(desc,MAX_DESC_LENGTH,"Myricom Merge Bitmask All Ports snf%d",
            allports);
        /*
         * XXX - 是否有任何关于此设备的 "up" 和 "running" 的概念，
         * 考虑到它处理多个端口？
         *
         * 大概没有 "connected" vs. "disconnected" 的概念，
         * 因为 "这个插入了网络吗？" 是每个端口的属性。
         */
        if (add_dev(devlistp, name,
            PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE, desc,
            errbuf) == NULL)
            return (-1);
        /*
         * XXX - 我们应该给它一个地址列表，包含所有端口的所有地址吗？
         */
    }

    return 0;
}

pcap_t *
snf_create(const char *device, char *ebuf, int *is_ours)
    // 定义一个 pcap_t 类型的指针 p
    pcap_t *p;
    // 初始化 boardnum 为 -1
    int boardnum = -1;
    // 定义一个 snf_ifaddrs 类型的指针 ifaddrs 和 ifa
    struct snf_ifaddrs *ifaddrs, *ifa;
    // 定义一个 size_t 类型的变量 devlen
    size_t devlen;
    // 定义一个 pcap_snf 类型的指针 ps

    // 如果无法初始化 SNF API，则没有 SNF 设备，设置 is_ours 为 0，返回空指针
    if (snf_init(SNF_VERSION_API)) {
        *is_ours = 0;
        return NULL;
    }

    // 获取 SNF 接口地址列表
    if (snf_getifaddrs(&ifaddrs) || ifaddrs == NULL) {
        // 无法获取 SNF 地址，设置 is_ours 为 0，返回空指针
        *is_ours = 0;
        return NULL;
    }
    // 计算设备名的长度
    devlen = strlen(device) + 1;
    ifa = ifaddrs;
    // 遍历接口地址列表，匹配给定的接口名，获取对应的板号
    while (ifa) {
        if (strncmp(device, ifa->snf_ifa_name, devlen) == 0) {
            boardnum = ifa->snf_ifa_boardnum;
            break;
        }
        ifa = ifa->snf_ifa_next;
    }
    // 释放接口地址列表
    snf_freeifaddrs(ifaddrs);

    // 如果找不到设备名，支持 "snfX" 和 "snf10gX" 的格式，其中 X 是板号
    if (ifa == NULL) {
        if (sscanf(device, "snf10g%d", &boardnum) != 1 &&
            sscanf(device, "snf%d", &boardnum) != 1) {
            // 不是支持的设备名，设置 is_ours 为 0，返回空指针
            *is_ours = 0;
            return NULL;
        }
    }

    // 设备可能是我们的，设置 is_ours 为 1
    *is_ours = 1;

    // 创建一个 pcap_snf 类型的 pcap_t 对象
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_snf);
    if (p == NULL)
        return NULL;
    ps = p->priv;

    // 支持微秒和纳秒时间戳
    p->tstamp_precision_list = malloc(2 * sizeof(u_int));
    if (p->tstamp_precision_list == NULL) {
        // 分配内存失败，设置错误消息，关闭 pcap_t 对象，返回空指针
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno,
            "malloc");
        pcap_close(p);
        return NULL;
    }
    p->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
    p->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
    p->tstamp_precision_count = 2;

    // 设置激活操作为 snf_activate
    p->activate_op = snf_activate;
    // 设置 ps 的 snf_boardnum 为 boardnum
    ps->snf_boardnum = boardnum;
    // 返回 pcap_t 对象
    return p;
#ifdef SNF_ONLY
/*
 * This libpcap build supports only SNF cards, not regular network
 * interfaces..
 */
// 如果定义了 SNF_ONLY，则表示该 libpcap 构建仅支持 SNF 卡，而不支持常规网络接口

/*
 * There are no regular interfaces, just SNF interfaces.
 */
// 没有常规接口，只有 SNF 接口
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    return (0);
}

/*
 * Attempts to open a regular interface fail.
 */
// 尝试打开常规接口会失败
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
    snprintf(errbuf, PCAP_ERRBUF_SIZE,
        "This version of libpcap only supports SNF cards");
    return NULL;
}

/*
 * Libpcap version string.
 */
// Libpcap 版本字符串
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING " (SNF-only)");
}
#endif
```