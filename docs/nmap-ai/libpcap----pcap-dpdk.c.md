# `nmap\libpcap\pcap-dpdk.c`

```cpp
/*
 * 版权所有（C）2018 jingle YANG。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 *
 *   1. 源代码的重新分发必须保留上述版权声明、此条件列表和以下免责声明。
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
 *
 * 本软件由作者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。无论在任何情况下，作者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购；使用、数据或利润的损失；或业务中断）承担任何责任，无论是在合同责任、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能性。
 */

/*
日期：2018年12月16日
描述：
1. Pcap-dpdk提供了使用DPDK的libpcap能力，设备名称为dpdk:{portid}，例如dpdk:0。
2. DPDK是一组用于快速数据包处理的库和驱动程序。（https://www.dpdk.org/）
3. testprogs/capturetest在Intel 10-Gigabit X540-AT2上使用DPDK 18.11提供了6.4Gbps/800,000 pps。

限制：
1. 如果DPDK可用，则将启用DPDK支持。如果手动安装了DPDK，请在./configure中设置--with-dpdk[=DIR]或在cmake中设置-DDPDK_DIR[=DIR]。
2. 仅支持动态链接libdpdk.so，因为libdpdk.a将无法正常工作。
 */
# 仅支持读操作，尚不支持数据包注入
# 使用说明：
# 1. 编译 DPDK 作为共享库并安装。(https://github.com/DPDK/dpdk.git)
# 需要修改文件 $RTE_SDK/$RTE_TARGET/.config 并设置：
# CONFIG_RTE_BUILD_SHARED_LIB=y
# 使用以下命令：
# sed -i 's/CONFIG_RTE_BUILD_SHARED_LIB=n/CONFIG_RTE_BUILD_SHARED_LIB=y/' $RTE_SDK/$RTE_TARGET/.config
# 2. 正确启动 l2fwd，这是 DPDK 示例之一，并获取设备信息
# 需要学习如何通过 $RTE_SDK/usertools/dpdk-devbind.py 将网卡绑定到 DPDK 兼容的驱动程序，例如 igb_uio
# 并通过 dpdk-setup.sh 启用大页内存
# 然后使用动态驱动程序支持启动 l2fwd。例如：
# $RTE_SDK/examples/l2fwd/$RTE_TARGET/l2fwd -dlibrte_pmd_e1000.so -dlibrte_pmd_ixgbe.so -dlibrte_mempool_ring.so -- -p 0x1
# 3. 使用 dpdk 选项编译 libpcap
# 如果 DPDK 没有被自动找到，需要导出 DPDK 环境变量，用于编译 DPDK。然后将 $RTE_SDK/$RTE_TARGET 传递给 --with-dpdk 或 -DDPDK_DIR
# export RTE_SDK={your DPDK base directory}
# export RTE_TARGET={your target name}
# 3.1 使用 configure
# ./configure --with-dpdk=$RTE_SDK/$RTE_TARGET && make -s all && make -s testprogs && make install
# 3.2 使用 cmake
# mkdir -p build && cd build && cmake -DDPDK_DIR=$RTE_SDK/$RTE_TARGET ../ && make -s all && make -s testprogs && make install
# 4. 将自己的程序与 libpcap 链接，并使用设备名称为 dpdk:{portid} 的 DPDK，例如 dpdk:0
# 需要通过环境变量 DPDK_CFG 设置 DPDK 配置选项
# 例如，testprogs/capturetest 可以通过以下方式启动：
# env DPDK_CFG="--log-level=debug -l0 -dlibrte_pmd_e1000.so -dlibrte_pmd_ixgbe.so -dlibrte_mempool_ring.so" ./capturetest -i dpdk:0

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
#include <errno.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <limits.h> /* for INT_MAX */
#include <time.h>
#include <sys/time.h>

// 调用 DPDK 所需的头文件
#include <rte_config.h>
#include <rte_common.h>
#include <rte_errno.h>
#include <rte_log.h>
#include <rte_malloc.h>
#include <rte_memory.h>
#include <rte_eal.h>
#include <rte_launch.h>
#include <rte_atomic.h>
#include <rte_cycles.h>
#include <rte_lcore.h>
#include <rte_per_lcore.h>
#include <rte_branch_prediction.h>
#include <rte_interrupts.h>
#include <rte_random.h>
#include <rte_debug.h>
#include <rte_ether.h>
#include <rte_ethdev.h>
#include <rte_mempool.h>
#include <rte_mbuf.h>
#include <rte_bus.h>

#include "pcap-int.h"
#include "pcap-dpdk.h"

/*
 * 处理破坏源代码兼容性的 API 更改
 */

#ifdef HAVE_STRUCT_RTE_ETHER_ADDR
#define ETHER_ADDR_TYPE    struct rte_ether_addr
#else
#define ETHER_ADDR_TYPE    struct ether_addr
#endif

#define DPDK_DEF_LOG_LEV RTE_LOG_ERR
//
// 如果尚未初始化 DPDK，则设置为 0；如果成功初始化，则设置为 1；如果尝试初始化并出现错误，则设置为 rte_eal_init() 的 rte_errno 的负值
//
static int is_dpdk_pre_inited=0;
#define DPDK_LIB_NAME "libpcap_dpdk"
#define DPDK_DESC "Data Plane Development Kit (DPDK) Interface"
#define DPDK_ERR_PERM_MSG "permission denied, DPDK needs root permission"
#define DPDK_ARGC_MAX 64
#define DPDK_CFG_MAX_LEN 1024
#define DPDK_DEV_NAME_MAX 32
#define DPDK_DEV_DESC_MAX 512
#define DPDK_CFG_ENV_NAME "DPDK_CFG"
#define DPDK_DEF_MIN_SLEEP_MS 1
static char dpdk_cfg_buf[DPDK_CFG_MAX_LEN];
#define DPDK_MAC_ADDR_SIZE 32
#define DPDK_DEF_MAC_ADDR "00:00:00:00:00:00"
#define DPDK_PCI_ADDR_SIZE 16
#define DPDK_DEF_CFG "--log-level=error -l0 -dlibrte_pmd_e1000.so -dlibrte_pmd_ixgbe.so -dlibrte_mempool_ring.so"
#define DPDK_PREFIX "dpdk:"
#define DPDK_PORTID_MAX 65535U
#define MBUF_POOL_NAME "mbuf_pool"
#define DPDK_TX_BUF_NAME "tx_buffer"
// mbuf 池中的元素数量
#define DPDK_NB_MBUFS 8192U
#define MEMPOOL_CACHE_SIZE 256
#define MAX_PKT_BURST 32
// 可配置的RX/TX环描述符数量
#define RTE_TEST_RX_DESC_DEFAULT 1024
#define RTE_TEST_TX_DESC_DEFAULT 1024

// 默认的RX/TX环描述符数量
static uint16_t nb_rxd = RTE_TEST_RX_DESC_DEFAULT;
static uint16_t nb_txd = RTE_TEST_TX_DESC_DEFAULT;

// 如果定义了RTE_ETHER_MAX_JUMBO_FRAME_LEN，则使用其值作为抓包的快照长度，否则使用ETHER_MAX_JUMBO_FRAME_LEN
#ifdef RTE_ETHER_MAX_JUMBO_FRAME_LEN
#define RTE_ETH_PCAP_SNAPLEN RTE_ETHER_MAX_JUMBO_FRAME_LEN
#else
#define RTE_ETH_PCAP_SNAPLEN ETHER_MAX_JUMBO_FRAME_LEN
#endif

// 用于DPDK时间戳辅助结构的定义
struct dpdk_ts_helper{
    struct timeval start_time;
    uint64_t start_cycles;
    uint64_t hz;
};
// 用于DPDK抓包结构的定义
struct pcap_dpdk{
    pcap_t * orig;
    uint16_t portid; // DPDK的端口ID
    int must_clear_promisc;
    uint64_t bpf_drop;
    int nonblock;
    struct timeval required_select_timeout;
    struct timeval prev_ts;
    struct rte_eth_stats prev_stats;
    struct timeval curr_ts;
    struct rte_eth_stats curr_stats;
    uint64_t pps;
    uint64_t bps;
    struct rte_mempool * pktmbuf_pool;
    struct dpdk_ts_helper ts_helper;
    ETHER_ADDR_TYPE eth_addr;
    char mac_addr[DPDK_MAC_ADDR_SIZE];
    char pci_addr[DPDK_PCI_ADDR_SIZE];
    unsigned char pcap_tmp_buf[RTE_ETH_PCAP_SNAPLEN];
};

// 用于DPDK端口配置的结构定义
static struct rte_eth_conf port_conf = {
    .rxmode = {
        .split_hdr_size = 0,
    },
    .txmode = {
        .mq_mode = ETH_MQ_TX_NONE,
    },
};

// 根据格式、参数和rte_errno生成错误消息，包括rte_errno的消息
static void dpdk_fmt_errmsg_for_rte_errno(char *errbuf, size_t errbuflen,
    int errnum, const char *fmt, ...)
{
    va_list ap;
    size_t msglen;
    char *p;
    size_t errbuflen_remaining;

    va_start(ap, fmt);
    vsnprintf(errbuf, errbuflen, fmt, ap);
    va_end(ap);
    msglen = strlen(errbuf);
    # 检查是否有足够的空间来追加“: ”
    # 包括终止的 '\0'，需要 3 个字节
    if (msglen + 3 > errbuflen):
        # 没有足够空间 - 只返回我们已经生成的部分
        return
    # 设置指针 p 指向错误缓冲区的末尾
    p = errbuf + msglen
    # 计算剩余的错误缓冲区长度
    errbuflen_remaining = errbuflen - msglen
    # 追加 ':'
    *p++ = ':'
    # 追加空格
    *p++ = ' '
    # 设置字符串终止符
    *p = '\0'
    # 更新消息长度
    msglen += 2
    # 更新剩余的错误缓冲区长度
    errbuflen_remaining -= 2

    # 现在追加错误代码的字符串
    # rte_strerror() 是线程安全的，至少在 dpdk 18.11 中是这样
    # 不像 strerror() - 它使用 strerror_r() 而不是 strerror() 来处理 UN*X 错误值
    # 并且针对 DPDK 错误，它打印到我假设是每个线程的缓冲区
    # （基于 "RTE_DEFINE_PER_LCORE" 中使用的 "PER_LCORE" 来静态声明缓冲区）
    snprintf(p, errbuflen_remaining, "%s", rte_strerror(errnum))
static int dpdk_init_timer(struct pcap_dpdk *pd){
    // 获取当前时间并存储在pd->ts_helper.start_time中
    gettimeofday(&(pd->ts_helper.start_time),NULL);
    // 获取当前CPU周期数并存储在pd->ts_helper.start_cycles中
    pd->ts_helper.start_cycles = rte_get_timer_cycles();
    // 获取CPU的时钟频率并存储在pd->ts_helper.hz中
    pd->ts_helper.hz = rte_get_timer_hz();
    // 如果时钟频率为0，则返回-1
    if (pd->ts_helper.hz == 0){
        return -1;
    }
    // 返回0
    return 0;
}
static inline void calculate_timestamp(struct dpdk_ts_helper *helper,struct timeval *ts)
{
    uint64_t cycles;
    // 计算当前CPU周期数
    cycles = rte_get_timer_cycles() - helper->start_cycles;
    // 计算当前时间
    struct timeval cur_time;
    cur_time.tv_sec = (time_t)(cycles/helper->hz);
    cur_time.tv_usec = (suseconds_t)((cycles%helper->hz)*1e6/helper->hz);
    // 将当前时间与起始时间相加，存储在ts中
    timeradd(&(helper->start_time), &cur_time, ts);
}

static uint32_t dpdk_gather_data(unsigned char *data, uint32_t len, struct rte_mbuf *mbuf)
{
    uint32_t total_len = 0;
    // 循环读取rte_mbuf中的数据，直到达到指定长度或者mbuf为空
    while (mbuf && (total_len+mbuf->data_len) < len ){
        // 将rte_mbuf中的数据复制到data中
        rte_memcpy(data+total_len, rte_pktmbuf_mtod(mbuf,void *),mbuf->data_len);
        // 更新已读取数据的总长度
        total_len+=mbuf->data_len;
        // 移动到下一个rte_mbuf
        mbuf=mbuf->next;
    }
    // 返回已读取数据的总长度
    return total_len;
}


static int dpdk_read_with_timeout(pcap_t *p, struct rte_mbuf **pkts_burst, const uint16_t burst_cnt){
    // 获取pcap_dpdk结构体指针
    struct pcap_dpdk *pd = (struct pcap_dpdk*)(p->priv);
    int nb_rx = 0;
    // 获取超时时间
    int timeout_ms = p->opt.timeout;
    int sleep_ms = 0;
    // 如果处于非阻塞模式，则只读取一次数据包
    if (pd->nonblock){
        nb_rx = (int)rte_eth_rx_burst(pd->portid, 0, pkts_burst, burst_cnt);
    }else{
        // 在阻塞模式下，多次读取直到捕获到数据包或超时或设置了break_loop。
        // 如果 timeout_ms == 0，则可能会永远阻塞。
        while (timeout_ms == 0 || sleep_ms < timeout_ms){
            // 从指定端口接收数据包，最多接收 burst_cnt 个数据包
            nb_rx = (int)rte_eth_rx_burst(pd->portid, 0, pkts_burst, burst_cnt);
            if (nb_rx){ // 在 timeout_ms 内收到数据包
                break;
            }else{ // 在本轮中没有收到数据包
                if (p->break_loop){
                    break;
                }
                // 睡眠一小段时间
                // 阻塞睡眠是唯一的选择，因为使用 usleep() 会严重影响性能。
                rte_delay_us_block(DPDK_DEF_MIN_SLEEP_MS*1000);
                sleep_ms += DPDK_DEF_MIN_SLEEP_MS;
            }
        }
    }
    // 返回接收到的数据包数量
    return nb_rx;
static int pcap_dpdk_dispatch(pcap_t *p, int max_cnt, pcap_handler cb, u_char *cb_arg)
{
    // 获取 pcap_dpdk 结构体指针
    struct pcap_dpdk *pd = (struct pcap_dpdk*)(p->priv);
    // 初始化变量 burst_cnt 和 nb_rx
    int burst_cnt = 0;
    int nb_rx = 0;
    // 创建一个包裹最大包裹数量的 rte_mbuf 结构体数组
    struct rte_mbuf *pkts_burst[MAX_PKT_BURST];
    // 创建一个 rte_mbuf 结构体指针 m
    struct rte_mbuf *m;
    // 创建一个 pcap_pkthdr 结构体 pcap_header
    struct pcap_pkthdr pcap_header;
    // 在 DPDK 中，pkt_len 是所有段的长度之和。而 data_len 是一个段的长度
    uint32_t pkt_len = 0;
    uint32_t caplen = 0;
    // 创建一个 u_char 类型的指针 bp，并初始化为 NULL
    u_char *bp = NULL;
    // 初始化变量 i 和 gather_len
    int i=0;
    unsigned int gather_len =0;
    // 初始化变量 pkt_cnt
    int pkt_cnt = 0;
    // 创建一个 u_char 类型的指针 large_buffer，并初始化为 NULL
    u_char *large_buffer=NULL;
    // 获取超时时间
    int timeout_ms = p->opt.timeout;

    /*
     * 这可能处理超过 INT_MAX 个数据包，
     * 这将导致数据包计数溢出，要么看起来像一个负数，从而导致我们返回一个看起来像一个错误的值，
     * 要么溢出回到正数领域，从而导致我们返回一个太低的计数。
     *
     * 因此，如果数据包计数是无限的，我们将其剪切到 INT_MAX；这个例程不应该无限处理数据包，所以这不是一个问题。
     */
    if (PACKET_COUNT_IS_UNLIMITED(max_cnt))
        max_cnt = INT_MAX;

    // 如果最大计数小于 MAX_PKT_BURST，则 burst_cnt 等于最大计数，否则等于 MAX_PKT_BURST
    if (max_cnt < MAX_PKT_BURST){
        burst_cnt = max_cnt;
    }else{
        burst_cnt = MAX_PKT_BURST;
    }

    // 返回数据包计数
    return pkt_cnt;
}

static int pcap_dpdk_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
    // 尚未实现
    // 将错误信息写入 errbuf
    pcap_strlcpy(p->errbuf,
        "dpdk error: Inject function has not been implemented yet",
        PCAP_ERRBUF_SIZE);
    // 返回错误
    return PCAP_ERROR;
}

static void pcap_dpdk_close(pcap_t *p)
{
    // 获取 pcap_dpdk 结构体指针
    struct pcap_dpdk *pd = p->priv;
    // 如果 pd 为空，则返回
    if (pd==NULL)
    {
        return;
    }
    // 如果必须清除混杂模式，则禁用混杂模式
    if (pd->must_clear_promisc)
    {
        rte_eth_promiscuous_disable(pd->portid);
    }
    // 停止设备
    rte_eth_dev_stop(pd->portid);
    // 关闭设备
    rte_eth_dev_close(pd->portid);
    // 清理 pcap
    pcap_cleanup_live_common(p);
}

static void nic_stats_display(struct pcap_dpdk *pd)
{
    // 从传入的数据包结构体中获取端口ID
    uint16_t portid = pd->portid;
    // 创建一个结构体用于存储以太网设备的统计信息
    struct rte_eth_stats stats;
    // 获取指定端口的以太网设备统计信息
    rte_eth_stats_get(portid, &stats);
    // 打印端口ID、接收数据包数量、接收错误数量、接收字节数、接收丢失数量的日志信息
    RTE_LOG(INFO,USER1, "portid:%d, RX-packets: %-10"PRIu64"  RX-errors:  %-10"PRIu64
           "  RX-bytes:  %-10"PRIu64"  RX-Imissed:  %-10"PRIu64"\n", portid, stats.ipackets, stats.ierrors,
           stats.ibytes,stats.imissed);
    // 打印端口ID、接收数据包速率、接收数据速率的日志信息
    RTE_LOG(INFO,USER1, "portid:%d, RX-PPS: %-10"PRIu64" RX-Mbps: %.2lf\n", portid, pd->pps, pd->bps/1e6f );
}
// 获取 DPDK 网卡统计信息
static int pcap_dpdk_stats(pcap_t *p, struct pcap_stat *ps)
{
    // 获取 pcap_dpdk 结构体指针
    struct pcap_dpdk *pd = p->priv;
    // 计算时间戳
    calculate_timestamp(&(pd->ts_helper), &(pd->curr_ts));
    // 获取网卡统计信息
    rte_eth_stats_get(pd->portid,&(pd->curr_stats));
    // 如果传入了统计结构体指针，则填充统计信息
    if (ps){
        ps->ps_recv = pd->curr_stats.ipackets;
        ps->ps_drop = pd->curr_stats.ierrors;
        ps->ps_drop += pd->bpf_drop;
        ps->ps_ifdrop = pd->curr_stats.imissed;
    }
    // 计算数据包数量的增量
    uint64_t delta_pkt = pd->curr_stats.ipackets - pd->prev_stats.ipackets;
    // 计算时间戳的增量
    struct timeval delta_tm;
    timersub(&(pd->curr_ts),&(pd->prev_ts), &delta_tm);
    // 计算时间增量的微秒数
    uint64_t delta_usec = delta_tm.tv_sec*1e6+delta_tm.tv_usec;
    // 计算比特数的增量
    uint64_t delta_bit = (pd->curr_stats.ibytes-pd->prev_stats.ibytes)*8;
    // 打印调试信息
    RTE_LOG(DEBUG, USER1, "delta_usec: %-10"PRIu64" delta_pkt: %-10"PRIu64" delta_bit: %-10"PRIu64"\n", delta_usec, delta_pkt, delta_bit);
    // 计算每秒数据包数和比特数
    pd->pps = (uint64_t)(delta_pkt*1e6f/delta_usec);
    pd->bps = (uint64_t)(delta_bit*1e6f/delta_usec);
    // 显示网卡统计信息
    nic_stats_display(pd);
    // 更新前一次的统计信息和时间戳
    pd->prev_stats = pd->curr_stats;
    pd->prev_ts = pd->curr_ts;
    return 0;
}

// 设置非阻塞模式
static int pcap_dpdk_setnonblock(pcap_t *p, int nonblock){
    // 获取 pcap_dpdk 结构体指针
    struct pcap_dpdk *pd = (struct pcap_dpdk*)(p->priv);
    // 设置非阻塞模式
    pd->nonblock = nonblock;
    return 0;
}

// 获取非阻塞模式
static int pcap_dpdk_getnonblock(pcap_t *p){
    // 获取 pcap_dpdk 结构体指针
    struct pcap_dpdk *pd = (struct pcap_dpdk*)(p->priv);
    // 返回非阻塞模式
    return pd->nonblock;
}

// 检查链路状态
static int check_link_status(uint16_t portid, struct rte_eth_link *plink)
{
    // 等待最多 9 秒获取链路状态
    rte_eth_link_get(portid, plink);
    // 返回链路状态是否为 ETH_LINK_UP
    return plink->link_status == ETH_LINK_UP;
}

// 将以太网地址转换为字符串
static void eth_addr_str(ETHER_ADDR_TYPE *addrp, char* mac_str, int len)
{
    int offset=0;
    // 如果地址为空，则使用默认 MAC 地址
    if (addrp == NULL){
        snprintf(mac_str, len-1, DPDK_DEF_MAC_ADDR);
        return;
    }
    // 遍历地址的每个字节
    for (int i=0; i<6; i++)
    {
        // 如果偏移量大于等于长度，表示缓冲区溢出，直接返回
        if (offset >= len)
        { // buffer overflow
            return;
        }
        // 如果是第一个字节，使用格式化字符串将地址字节转换为十六进制，并将结果存入mac_str中
        if (i==0)
        {
            snprintf(mac_str+offset, len-1-offset, "%02X",addrp->addr_bytes[i]);
            offset+=2; // FF
        }else{
            // 如果不是第一个字节，使用格式化字符串将地址字节转换为十六进制，并在前面加上冒号，然后将结果存入mac_str中
            snprintf(mac_str+offset, len-1-offset, ":%02X", addrp->addr_bytes[i]);
            offset+=3; // :FF
        }
    }
    // 返回结果
    return;
// 根据设备名称返回端口ID，如果找不到则返回-1
static uint16_t portid_by_device(char * device)
{
    uint16_t ret = DPDK_PORTID_MAX; // 初始化返回值为最大端口ID
    int len = strlen(device); // 获取设备名称的长度
    int prefix_len = strlen(DPDK_PREFIX); // 获取DPDK前缀的长度
    unsigned long ret_ul = 0L; // 初始化返回值为长整型0
    char *pEnd; // 定义指向字符串结尾的指针
    if (len<=prefix_len || strncmp(device, DPDK_PREFIX, prefix_len)) // 检查设备名称是否以DPDK前缀开头
    {
        return ret; // 如果不是，则返回最大端口ID
    }
    // 检查所有字符是否都是数字
    for (int i=prefix_len; device[i]; i++){
        if (device[i]<'0' || device[i]>'9'){
            return ret; // 如果不是，则返回最大端口ID
        }
    }
    ret_ul = strtoul(&(device[prefix_len]), &pEnd, 10); // 将数字部分转换为长整型
    if (pEnd == &(device[prefix_len]) || *pEnd != '\0'){ // 检查转换后的字符串是否合法
        return ret; // 如果不合法，则返回最大端口ID
    }
    // 如果转换后的值超过最大端口ID，则返回最大端口ID
    if (ret_ul >= DPDK_PORTID_MAX){
        return ret;
    }
    ret = (uint16_t)ret_ul; // 将转换后的值赋给返回值
    return ret; // 返回端口ID
}

// 解析DPDK配置
static int parse_dpdk_cfg(char* dpdk_cfg,char** dargv)
{
    int cnt=0; // 初始化参数计数器
    memset(dargv,0,sizeof(dargv[0])*DPDK_ARGC_MAX); // 将参数数组清零
    int skip_space = 1; // 初始化跳过空格标志
    int i=0; // 初始化循环变量
    RTE_LOG(INFO, USER1,"dpdk cfg: %s\n",dpdk_cfg); // 记录DPDK配置信息
    // 寻找第一个非空格字符
    // 最后一个选项为NULL
    for (i=0;dpdk_cfg[i] && cnt<DPDK_ARGC_MAX-1;i++){
        if (skip_space && dpdk_cfg[i]!=' '){ // 如果是空格，则跳过
            skip_space=!skip_space; // 跳过正常字符
            dargv[cnt++] = dpdk_cfg+i; // 将参数地址赋给参数数组
        }
        if (!skip_space && dpdk_cfg[i]==' '){ // 如果是空格，则结束该选项
            dpdk_cfg[i]=0x00; // 将空格替换为结束符
            skip_space=!skip_space; // 跳过空格字符
        }
    }
    dargv[cnt]=NULL; // 最后一个选项为NULL
    return cnt; // 返回参数个数
}

// 仅调用一次
// 返回：
//    1 表示成功；
//    0 表示“EAL无法在此系统上初始化”，我们将其视为“DPDK不可用”；
//    其他错误的PCAP_ERROR_代码。
// 如果eaccess_not_fatal非零，则将“权限问题”视为“EAL无法在此系统上初始化”。我们在尝试查找DPDK设备时使用它，因为我们不希望无法返回
// 初始化 DPDK，如果无法支持 DPDK，则在尝试打开设备时需要返回权限错误
static int dpdk_pre_init(char * ebuf, int eaccess_not_fatal)
{
    int dargv_cnt=0;  // 初始化参数计数
    char *dargv[DPDK_ARGC_MAX];  // 初始化参数数组
    char *ptr_dpdk_cfg = NULL;  // 初始化 DPDK 配置指针
    int ret;  // 初始化返回值

    // 检查是否已经进行了 DPDK 的预初始化
    if (is_dpdk_pre_inited != 0)
    {
        // 如果已经初始化过，则判断是否成功
        if (is_dpdk_pre_inited < 0)
        {
            // 如果初始化失败，则跳转到错误处理
            goto error;
        }
        else
        {
            // 如果初始化成功，则返回 1
            return 1;
        }
    }

    // 初始化 EAL
    ptr_dpdk_cfg = getenv(DPDK_CFG_ENV_NAME);
    // 设置默认日志级别为调试
    rte_log_set_global_level(DPDK_DEF_LOG_LEV);

    // 如果未设置 DPDK 配置，则使用默认配置
    if (ptr_dpdk_cfg == NULL)
    {
        RTE_LOG(INFO,USER1,"env $DPDK_CFG is unset, so using default: %s\n",DPDK_DEF_CFG);
        ptr_dpdk_cfg = DPDK_DEF_CFG;
    }

    // 清空 DPDK 配置缓冲区
    memset(dpdk_cfg_buf,0,sizeof(dpdk_cfg_buf));
    // 格式化 DPDK 配置字符串
    snprintf(dpdk_cfg_buf,DPDK_CFG_MAX_LEN-1,"%s %s",DPDK_LIB_NAME,ptr_dpdk_cfg);
    // 解析 DPDK 配置
    dargv_cnt = parse_dpdk_cfg(dpdk_cfg_buf,dargv);
    // 初始化 EAL
    ret = rte_eal_init(dargv_cnt,dargv);

    // 如果初始化失败，则设置 is_dpdk_pre_inited 为错误码的负值，并处理错误
    if (ret == -1)
    {
        is_dpdk_pre_inited = -rte_errno;
        goto error;
    }

    // 初始化成功，不需要再次初始化
    is_dpdk_pre_inited = 1;
    return 1;

error:
    // 处理错误
    switch (-is_dpdk_pre_inited)
    {
        // 错误
        return PCAP_ERROR;
    }
}

// 激活 DPDK
static int pcap_dpdk_activate(pcap_t *p)
{
    struct pcap_dpdk *pd = p->priv;
    pd->orig = p;
    int ret = PCAP_ERROR;
    uint16_t nb_ports=0;
    uint16_t portid= DPDK_PORTID_MAX;
    unsigned nb_mbufs = DPDK_NB_MBUFS;
    struct rte_eth_rxconf rxq_conf;
    struct rte_eth_txconf txq_conf;
    struct rte_eth_conf local_port_conf = port_conf;
    struct rte_eth_dev_info dev_info;
    int is_port_up = 0;
}
    // 定义一个名为link的rte_eth_link结构体变量
    struct rte_eth_link link;
    // 使用do-while循环，目的是为了在do块中执行一次操作，然后检查条件是否满足，如果满足则继续执行，否则退出循环
    }while(0);

    // 如果返回值小于等于PCAP_ERROR，表示出现各种错误
    if (ret <= PCAP_ERROR) // all kinds of error code
    {
        // 调用pcap_cleanup_live_common函数清理资源
        pcap_cleanup_live_common(p);
    }else{
        // 根据端口ID和网卡的PCI地址获取网卡设备名称
        rte_eth_dev_get_name_by_port(portid,pd->pci_addr);
        // 打印端口ID、设备名称、MAC地址和PCI地址的日志信息
        RTE_LOG(INFO, USER1,"Port %d device: %s, MAC:%s, PCI:%s\n", portid, p->opt.device, pd->mac_addr, pd->pci_addr);
        // 打印端口ID、链路速度和双工模式的日志信息
        RTE_LOG(INFO, USER1,"Port %d Link Up. Speed %u Mbps - %s\n",
                            portid, link.link_speed,
                    (link.link_duplex == ETH_LINK_FULL_DUPLEX) ?
                        ("full-duplex") : ("half-duplex\n"));
    }
    // 返回ret变量的值
    return ret;
// 为了使用 DPDK，设备名称应该以 dpdk:number 的形式，例如 dpdk:0
pcap_t * pcap_dpdk_create(const char *device, char *ebuf, int *is_ours)
{
    pcap_t *p=NULL;
    *is_ours = 0;

    // 检查设备名称是否以 "dpdk:" 开头，如果是则将 is_ours 设置为 1，表示是我们需要的设备
    *is_ours = !strncmp(device, "dpdk:", 5);
    if (! *is_ours)
        return NULL;
    // 使用 memset 对结构进行初始化
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dpdk);

    if (p == NULL)
        return NULL;
    // 设置激活操作为 pcap_dpdk_activate
    p->activate_op = pcap_dpdk_activate;
    return p;
}

// 查找所有的 DPDK 设备
int pcap_dpdk_findalldevs(pcap_if_list_t *devlistp, char *ebuf)
{
    int ret=0;
    unsigned int nb_ports = 0;
    char dpdk_name[DPDK_DEV_NAME_MAX];
    char dpdk_desc[DPDK_DEV_DESC_MAX];
    ETHER_ADDR_TYPE eth_addr;
    char mac_addr[DPDK_MAC_ADDR_SIZE];
    char pci_addr[DPDK_PCI_ADDR_SIZE];
    // 初始化 EAL；如果权限不足，则返回 "DPDK not available"
    char dpdk_pre_init_errbuf[PCAP_ERRBUF_SIZE];
    ret = dpdk_pre_init(dpdk_pre_init_errbuf, 1);
    if (ret < 0)
    {
        // 在出现错误时返回负值
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "Can't look for DPDK devices: %s",
            dpdk_pre_init_errbuf);
        ret = PCAP_ERROR;
        break;
    }
    if (ret == 0)
    {
        // 这意味着此机器上没有可用的 DPDK
        // 这只是意味着 "不返回任何设备"
        break;
    }
    nb_ports = rte_eth_dev_count_avail();
    if (nb_ports == 0)
    {
        // 这只是意味着 "不返回任何设备"
        ret = 0;
        break;
    }
    for (unsigned int i=0; i<nb_ports; i++){
        snprintf(dpdk_name, DPDK_DEV_NAME_MAX-1,
            "%s%u", DPDK_PREFIX, i);
        // mac 地址
        rte_eth_macaddr_get(i, &eth_addr);
        eth_addr_str(&eth_addr,mac_addr,DPDK_MAC_ADDR_SIZE);
        // PCI 地址
        rte_eth_dev_get_name_by_port(i,pci_addr);
        snprintf(dpdk_desc,DPDK_DEV_DESC_MAX-1,"%s %s, MAC:%s, PCI:%s", DPDK_DESC, dpdk_name, mac_addr, pci_addr);
        if (add_dev(devlistp, dpdk_name, 0, dpdk_desc, ebuf)==NULL){
            ret = PCAP_ERROR;
            break;
        }
    }
    return ret;
#ifdef DPDK_ONLY
/*
 * This libpcap build supports only DPDK, not regular network interfaces.
 */
// 这个 libpcap 构建仅支持 DPDK，不支持常规网络接口。

/*
 * There are no regular interfaces, just DPDK interfaces.
 */
// 没有常规接口，只有 DPDK 接口。
int
pcap_platform_finddevs(pcap_if_list_t *devlistp _U_, char *errbuf)
{
    return (0);
}

/*
 * Attempts to open a regular interface fail.
 */
// 尝试打开常规接口会失败。
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
    snprintf(errbuf, PCAP_ERRBUF_SIZE,
        "This version of libpcap only supports DPDK");
    return NULL;
}

/*
 * Libpcap version string.
 */
// Libpcap 版本字符串。
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING " (DPDK-only)");
}
#endif
```