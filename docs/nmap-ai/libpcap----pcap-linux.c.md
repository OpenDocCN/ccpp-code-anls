# `nmap\libpcap\pcap-linux.c`

```cpp
#define _GNU_SOURCE
// 定义_GNU_SOURCE，启用 GNU 扩展

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
// 如果有配置文件，则包含配置文件

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <limits.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <sys/ioctl.h>
#include <sys/utsname.h>
#include <sys/mman.h>
#include <linux/if.h>
#include <linux/if_packet.h>
#include <linux/sockios.h>
#include <linux/ethtool.h>
#include <netinet/in.h>
#include <linux/if_ether.h>
#include <linux/if_arp.h>
#include <poll.h>
#include <dirent.h>
#include <sys/eventfd.h>
// 包含所需的头文件

#include "pcap-int.h"
#include "pcap/sll.h"
#include "pcap/vlan.h"
#include "pcap/can_socketcan.h"
// 包含自定义的头文件

#include "diag-control.h"
// 包含诊断控制的头文件

/*
 * We require TPACKET_V2 support.
 */
#ifndef TPACKET2_HDRLEN
#error "Libpcap will only work if TPACKET_V2 is supported; you must build for a 2.6.27 or later kernel"
#endif
// 检查是否支持 TPACKET_V2，如果不支持则报错

/* check for memory mapped access avaibility. We assume every needed
 * struct is defined if the macro TPACKET_HDRLEN is defined, because it
 * uses many ring related structs and macros */
#ifdef TPACKET3_HDRLEN
# define HAVE_TPACKET3
#endif /* TPACKET3_HDRLEN */
// 检查内存映射访问的可用性，并假设如果定义了宏 TPACKET_HDRLEN，则假定已定义了所有需要的结构体

/*
 * Not all compilers that are used to compile code to run on Linux have
 * these builtins.  For example, older versions of GCC don't, and at
 * least some people are doing cross-builds for MIPS with older versions
 * of GCC.
 */
#ifndef HAVE___ATOMIC_LOAD_N
#define __atomic_load_n(ptr, memory_model)        (*(ptr))
#endif
#ifndef HAVE___ATOMIC_STORE_N
#define __atomic_store_n(ptr, val, memory_model)    *(ptr) = (val)
#endif
// 定义__atomic_load_n和__atomic_store_n宏，用于加载和存储原子变量

#define packet_mmap_acquire(pkt) \
    (__atomic_load_n(&pkt->tp_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)
#define packet_mmap_release(pkt) \
    (__atomic_store_n(&pkt->tp_status, TP_STATUS_KERNEL, __ATOMIC_RELEASE))
#define packet_mmap_v3_acquire(pkt) \
    (__atomic_load_n(&pkt->hdr.bh1.block_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)
#define packet_mmap_v3_release(pkt) \
// 定义packet_mmap_acquire、packet_mmap_release、packet_mmap_v3_acquire和packet_mmap_v3_release宏，用于获取和释放内存映射
    # 使用原子操作将pkt->hdr.bh1.block_status的值设置为TP_STATUS_KERNEL，并以释放模式进行操作
    (__atomic_store_n(&pkt->hdr.bh1.block_status, TP_STATUS_KERNEL, __ATOMIC_RELEASE))
#include <linux/types.h>
#include <linux/filter.h>

#ifdef HAVE_LINUX_NET_TSTAMP_H
#include <linux/net_tstamp.h>
#endif

/*
 * For checking whether a device is a bonding device.
 */
#include <linux/if_bonding.h>

/*
 * Got libnl?
 */
#ifdef HAVE_LIBNL
#include <linux/nl80211.h>

#include <netlink/genl/genl.h>
#include <netlink/genl/family.h>
#include <netlink/genl/ctrl.h>
#include <netlink/msg.h>
#include <netlink/attr.h>
#endif /* HAVE_LIBNL */

#ifndef HAVE_SOCKLEN_T
typedef int        socklen_t;
#endif

#define MAX_LINKHEADER_SIZE    256

/*
 * When capturing on all interfaces we use this as the buffer size.
 * Should be bigger then all MTUs that occur in real life.
 * 64kB should be enough for now.
 */
#define BIGGER_THAN_ALL_MTUS    (64*1024)

/*
 * Private data for capturing on Linux PF_PACKET sockets.
 */
struct pcap_linux {
    long long sysfs_dropped; /* packets reported dropped by /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors */
    struct pcap_stat stat;

    char    *device;    /* device name */
    int    filter_in_userland; /* must filter in userland */
    int    blocks_to_filter_in_userland;
    int    must_do_on_close; /* stuff we must do when we close */
    int    timeout;    /* timeout for buffering */
    int    cooked;        /* using SOCK_DGRAM rather than SOCK_RAW */
    int    ifindex;    /* interface index of device we're bound to */
    int    lo_ifindex;    /* interface index of the loopback device */
    int    netdown;    /* we got an ENETDOWN and haven't resolved it */
    bpf_u_int32 oldmode;    /* mode to restore when turning monitor mode off */
    char    *mondevice;    /* mac80211 monitor device we created */
    u_char    *mmapbuf;    /* memory-mapped region pointer */
    size_t    mmapbuflen;    /* size of region */
    int    vlan_offset;    /* offset at which to insert vlan tags; if -1, don't insert */
    u_int    tp_version;    /* version of tpacket_hdr for mmaped ring */
    # 无符号整型变量，表示用于 mmaped 环的 tpacket_hdr 的头部长度
    u_int    tp_hdrlen;    
    # 无符号字符指针，用于存储数据包的一次性缓冲区
    u_char    *oneshot_buffer; 
    # 整型变量，表示在 poll() 函数中使用的超时时间
    int    poll_timeout;    
#ifdef HAVE_TPACKET3
    unsigned char *current_packet; /* 当前 TPACKET_V3 块中的数据包。如果为 NULL，则移动到下一个块。 */
    int packets_left; /* 在上一次调用 pcap_read_linux_mmap_v3 时，TPACKET_V3 块中剩余未处理的数据包数量。 */
#endif
    int poll_breakloop_fd; /* 用于中断阻塞操作的事件文件描述符 */
};

/*
 * 关闭时需要执行的操作。
 */
#define MUST_CLEAR_RFMON    0x00000001    /* 清除 rfmon（监控）模式 */
#define MUST_DELETE_MONIF    0x00000002    /* 删除监控模式接口 */

/*
 * 内部函数和方法的原型。
 */
static int get_if_flags(const char *, bpf_u_int32 *, char *);
static int is_wifi(const char *);
static void map_arphrd_to_dlt(pcap_t *, int, const char *, int);
static int pcap_activate_linux(pcap_t *);
static int setup_socket(pcap_t *, int);
static int setup_mmapped(pcap_t *, int *);
static int pcap_can_set_rfmon_linux(pcap_t *);
static int pcap_inject_linux(pcap_t *, const void *, int);
static int pcap_stats_linux(pcap_t *, struct pcap_stat *);
static int pcap_setfilter_linux(pcap_t *, struct bpf_program *);
static int pcap_setdirection_linux(pcap_t *, pcap_direction_t);
static int pcap_set_datalink_linux(pcap_t *, int);
static void pcap_cleanup_linux(pcap_t *);

union thdr {
    struct tpacket2_hdr        *h2;
#ifdef HAVE_TPACKET3
    struct tpacket_block_desc    *h3;
#endif
    u_char                *raw;
};

#define RING_GET_FRAME_AT(h, offset) (((u_char **)h->buffer)[(offset)])
#define RING_GET_CURRENT_FRAME(h) RING_GET_FRAME_AT(h, h->offset)

static void destroy_ring(pcap_t *handle);
static int create_ring(pcap_t *handle, int *status);
static int prepare_tpacket_socket(pcap_t *handle);
static int pcap_read_linux_mmap_v2(pcap_t *, int, pcap_handler , u_char *);
#ifdef HAVE_TPACKET3
static int pcap_read_linux_mmap_v3(pcap_t *, int, pcap_handler , u_char *);
#endif
static int pcap_setnonblock_linux(pcap_t *p, int nonblock);
static int pcap_getnonblock_linux(pcap_t *p);
static void pcap_oneshot_linux(u_char *user, const struct pcap_pkthdr *h,
    const u_char *bytes);

/*
 * 在 3.0 之前的内核中，tp_vlan_tci 字段被设置为 skbuff 中的 vlan_tci 字段的值。
 * 0 可以表示“不在 VLAN 上”或“在 VLAN 0 上”。tp_status 字段中没有设置标志来区分它们。
 *
 * 在 3.0 及之后的内核中，如果存在 VLAN 标签，tp_vlan_tci 字段将被设置为 VLAN 标签，
 * 并且在 tp_status 字段中设置 TP_STATUS_VLAN_VALID 标志；否则，tp_vlan_tci 字段被设置为 0，
 * 并且在 tp_status 字段中 TP_STATUS_VLAN_VALID 标志未被设置。
 *
 * 在 3.0 之前的内核中，我们无法区分没有 VLAN 标签的数据包和在 VLAN 0 上的数据包，
 * 因此我们将错误处理一些数据包，而我们无法做任何处理。
 *
 * 因此，在那些从不设置 TP_STATUS_VLAN_VALID 标志的系统上，我们继续早期 libpcap 的行为，
 * 其中我们将 VLAN 标签为 0 的数据包视为没有 VLAN 标签的数据包而不是在 VLAN 0 上的数据包。
 * 我们通过将 tp_vlan_tci 为 0 且 tp_status 中未设置 TP_STATUS_VLAN_VALID 标志的数据包视为没有 VLAN 标签来实现这一点。
 * 这在 3.0 及之后的内核上做得很好，并在 3.0 之前的内核上继续旧的不可修复的不完美行为。
 *
 * 如果 TP_STATUS_VLAN_VALID 未定义，我们将其测试为 0x10 位；在 3.0 及之后的内核中，它具有该值。
 */
#ifdef TP_STATUS_VLAN_VALID
  #define VLAN_VALID(hdr, hv)    ((hv)->tp_vlan_tci != 0 || ((hdr)->tp_status & TP_STATUS_VLAN_VALID))
#else
  /*
   * 如果在缺少 TP_STATUS_VLAN_VALID 的系统上编译，我们将测试它在 3.0 及更高版本内核中的值，
   * 这样我们就可以在拥有该值的系统上进行测试。（如果在没有该值的系统上运行，它将不会在 tp_status 字段中设置，
   * 因此对它的测试将始终失败；这意味着我们的行为与引入此宏之前的行为相同。）
   */
  #define VLAN_VALID(hdr, hv)    ((hv)->tp_vlan_tci != 0 || ((hdr)->tp_status & 0x10))
#endif

#ifdef TP_STATUS_VLAN_TPID_VALID
# define VLAN_TPID(hdr, hv)    (((hv)->tp_vlan_tpid || ((hdr)->tp_status & TP_STATUS_VLAN_TPID_VALID)) ? (hv)->tp_vlan_tpid : ETH_P_8021Q)
#else
# define VLAN_TPID(hdr, hv)    ETH_P_8021Q
#endif

/*
 * 如果我们正在轮询“接口消失”指示，那么所需的选择超时时间为 1 毫秒。
 */
static const struct timeval netdown_timeout = {
    0, 1000        /* 1000 微秒 = 1 毫秒 */
};

/*
 * 包装一些 ioctl 调用
 */
static int    iface_get_id(int fd, const char *device, char *ebuf);
static int    iface_get_mtu(int fd, const char *device, char *ebuf);
static int    iface_get_arptype(int fd, const char *device, char *ebuf);
static int    iface_bind(int fd, int ifindex, char *ebuf, int protocol);
static int    enter_rfmon_mode(pcap_t *handle, int sock_fd,
    const char *device);
static int    iface_get_ts_types(const char *device, pcap_t *handle,
    char *ebuf);
static int    iface_get_offload(pcap_t *handle);

static int    fix_program(pcap_t *handle, struct sock_fprog *fcode);
static int    fix_offset(pcap_t *handle, struct bpf_insn *p);
static int    set_kernel_filter(pcap_t *handle, struct sock_fprog *fcode);
static int    reset_kernel_filter(pcap_t *handle);

static struct sock_filter    total_insn
    = BPF_STMT(BPF_RET | BPF_K, 0);
static struct sock_fprog    total_fcode
    = { 1, &total_insn };

static int    iface_dsa_get_proto_info(const char *device, pcap_t *handle);
pcap_t *
pcap_create_interface(const char *device, char *ebuf)
{
    pcap_t *handle;  // 定义一个 pcap_t 类型的指针变量

    handle = PCAP_CREATE_COMMON(ebuf, struct pcap_linux);  // 调用 PCAP_CREATE_COMMON 函数创建一个 pcap_linux 结构体，并将其地址赋给 handle
    if (handle == NULL)  // 如果 handle 为空
        return NULL;  // 返回空指针

    handle->activate_op = pcap_activate_linux;  // 设置 activate_op 指向 pcap_activate_linux 函数
    handle->can_set_rfmon_op = pcap_can_set_rfmon_linux;  // 设置 can_set_rfmon_op 指向 pcap_can_set_rfmon_linux 函数

    /*
     * See what time stamp types we support.
     */
    if (iface_get_ts_types(device, handle, ebuf) == -1) {  // 调用 iface_get_ts_types 函数获取时间戳类型
        pcap_close(handle);  // 关闭 handle
        return NULL;  // 返回空指针
    }

    /*
     * We claim that we support microsecond and nanosecond time
     * stamps.
     *
     * XXX - with adapter-supplied time stamps, can we choose
     * microsecond or nanosecond time stamps on arbitrary
     * adapters?
     */
    handle->tstamp_precision_list = malloc(2 * sizeof(u_int));  // 分配存储时间戳精度的内存空间
    if (handle->tstamp_precision_list == NULL) {  // 如果分配失败
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");  // 格式化错误消息
        pcap_close(handle);  // 关闭 handle
        return NULL;  // 返回空指针
    }
    handle->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;  // 设置第一个时间戳精度为微秒
    handle->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;  // 设置第二个时间戳精度为纳秒
    handle->tstamp_precision_count = 2;  // 设置时间戳精度数量为2

    struct pcap_linux *handlep = handle->priv;  // 定义一个 pcap_linux 结构体指针变量指向 handle 的 priv 成员
    handlep->poll_breakloop_fd = eventfd(0, EFD_NONBLOCK);  // 设置 poll_breakloop_fd 为非阻塞的 eventfd

    return handle;  // 返回 handle
}

#ifdef HAVE_LIBNL
/*
 * Is this a mac80211 device?  If so, fill in the physical device path and
 * return 1; if not, return 0.  On an error, fill in handle->errbuf and
 * return PCAP_ERROR.
 */
static int
get_mac80211_phydev(pcap_t *handle, const char *device, char *phydev_path,
    size_t phydev_max_pathlen)
{
    char *pathstr;  // 定义一个字符指针变量
    ssize_t bytes_read;  // 定义一个 ssize_t 类型的变量

    /*
     * Generate the path string for the symlink to the physical device.
     */
    if (asprintf(&pathstr, "/sys/class/net/%s/phy80211", device) == -1) {  // 生成指向物理设备的符号链接的路径字符串
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: Can't generate path name string for /sys/class/net device",
            device);  // 格式化错误消息
        return PCAP_ERROR;  // 返回错误值
    }
    # 调用 readlink 函数读取路径的符号链接内容，并将结果保存在 bytes_read 中
    bytes_read = readlink(pathstr, phydev_path, phydev_max_pathlen);
    # 如果读取失败
    if (bytes_read == -1) {
        # 如果错误码是文件不存在或者无效参数
        if (errno == ENOENT || errno == EINVAL) {
            '''
            如果路径不存在，或者不是一个符号链接；假设它不是一个 mac80211 设备
            释放 pathstr 的内存
            返回 0
            '''
            free(pathstr);
            return 0;
        }
        # 格式化错误消息，将错误信息存储在 handle->errbuf 中
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "%s: Can't readlink %s", device, pathstr);
        # 释放 pathstr 的内存
        free(pathstr);
        # 返回错误码
        return PCAP_ERROR;
    }
    # 释放 pathstr 的内存
    free(pathstr);
    # 在读取的符号链接内容后面添加字符串结束符
    phydev_path[bytes_read] = '\0';
    # 返回 1
    return 1;
# 结构体定义，包含了与 nl80211 相关的状态信息
struct nl80211_state {
    struct nl_sock *nl_sock;  // netlink 套接字
    struct nl_cache *nl_cache;  // netlink 缓存
    struct genl_family *nl80211;  // nl80211 族
};

# 初始化 nl80211 状态
static int
nl80211_init(pcap_t *handle, struct nl80211_state *state, const char *device)
{
    int err;

    state->nl_sock = nl_socket_alloc();  // 分配 netlink 套接字
    if (!state->nl_sock) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: failed to allocate netlink handle", device);
        return PCAP_ERROR;
    }

    if (genl_connect(state->nl_sock)) {  // 连接 generic netlink
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: failed to connect to generic netlink", device);
        goto out_handle_destroy;
    }

    err = genl_ctrl_alloc_cache(state->nl_sock, &state->nl_cache);  // 分配 generic netlink 缓存
    if (err < 0) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: failed to allocate generic netlink cache: %s",
            device, nl_geterror(-err));
        goto out_handle_destroy;
    }

    state->nl80211 = genl_ctrl_search_by_name(state->nl_cache, "nl80211");  // 通过名字在缓存中搜索 nl80211 族
    if (!state->nl80211) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: nl80211 not found", device);
        goto out_cache_free;
    }

    return 0;

out_cache_free:
    nl_cache_free(state->nl_cache);  // 释放缓存
out_handle_destroy:
    nl_socket_free(state->nl_sock);  // 释放套接字
    return PCAP_ERROR;
}

# 清理 nl80211 状态
static void
nl80211_cleanup(struct nl80211_state *state)
{
    genl_family_put(state->nl80211);  // 释放族
    nl_cache_free(state->nl_cache);  // 释放缓存
    nl_socket_free(state->nl_sock);  // 释放套接字
}

# 删除监控接口
static int
del_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice);

# 添加监控接口
static int
add_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice)
{
    struct pcap_linux *handlep = handle->priv;
    int ifindex;
    struct nl_msg *msg;
    int err;

    ifindex = iface_get_id(sock_fd, device, handle->errbuf);  // 获取接口 ID
    if (ifindex == -1)
        return PCAP_ERROR;

    msg = nlmsg_alloc();  // 分配 netlink 消息
    # 如果消息为空，则设置错误信息并返回错误代码
    if (!msg) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: failed to allocate netlink msg", device);
        return PCAP_ERROR;
    }
    
    # 在消息中添加通用 Netlink 消息头
    genlmsg_put(msg, 0, 0, genl_family_get_id(state->nl80211), 0,
            0, NL80211_CMD_NEW_INTERFACE, 0);
    
    # 在消息中添加一个32位的无符号整数属性，表示接口的索引
    NLA_PUT_U32(msg, NL80211_ATTR_IFINDEX, ifindex);
# 关闭对诊断信息的窄化
DIAG_OFF_NARROWING
    # 向消息中添加接口名称
    NLA_PUT_STRING(msg, NL80211_ATTR_IFNAME, mondevice);
# 打开对诊断信息的窄化
DIAG_ON_NARROWING
    # 向消息中添加接口类型
    NLA_PUT_U32(msg, NL80211_ATTR_IFTYPE, NL80211_IFTYPE_MONITOR);

    # 发送消息并等待自动完成
    err = nl_send_auto_complete(state->nl_sock, msg);
    # 如果发送失败
    if (err < 0) {
        # 如果错误是设备不可用
        if (err == -NLE_FAILURE) {
            '''
             * 设备不可用；我们的调用者应该继续尝试。
             * （libnl 2.x 将 ENFILE 映射到 NLE_FAILURE；
             * 它也可以将其他错误映射到那个，但我们对此无能为力。）
             '''
            nlmsg_free(msg);
            return 0;
        } else {
            '''
             * 真正的失败，不仅仅是“该设备不可用”。
             '''
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "%s: nl_send_auto_complete failed adding %s interface: %s",
                device, mondevice, nl_geterror(-err));
            nlmsg_free(msg);
            return PCAP_ERROR;
        }
    }
    # 等待确认消息的应答
    err = nl_wait_for_ack(state->nl_sock);
    # 如果等待失败
    if (err < 0) {
        # 如果错误是设备不可用
        if (err == -NLE_FAILURE) {
            '''
             * 设备不可用；我们的调用者应该继续尝试。
             * （libnl 2.x 将 ENFILE 映射到 NLE_FAILURE；
             * 它也可以将其他错误映射到那个，但我们对此无能为力。）
             '''
            nlmsg_free(msg);
            return 0;
        } else {
            '''
             * 真正的失败，不仅仅是“该设备不可用”。
             '''
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "%s: nl_wait_for_ack failed adding %s interface: %s",
                device, mondevice, nl_geterror(-err));
            nlmsg_free(msg);
            return PCAP_ERROR;
        }
    }

    '''
     * 成功。
     '''
    nlmsg_free(msg);

    '''
     * 尝试记住监视设备。
     '''
    handlep->mondevice = strdup(mondevice);
    # 如果监视设备为空
    if (handlep->mondevice == NULL) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "strdup");
        '''
        * 丢弃监视设备。
        '''
        # 删除监视设备
        del_mon_if(handle, sock_fd, state, device, mondevice);
        # 返回错误代码
        return PCAP_ERROR;
    }
    # 返回成功代码
    return 1;
# 定义一个函数，处理在添加监控模式接口时的错误情况
nla_put_failure:
    # 格式化错误信息到错误缓冲区
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "%s: nl_put failed adding %s interface",
        device, mondevice);
    # 释放消息对象
    nlmsg_free(msg);
    # 返回错误码
    return PCAP_ERROR;
}

# 定义一个函数，用于删除监控模式接口
static int
del_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice)
{
    int ifindex;
    struct nl_msg *msg;
    int err;

    # 获取监控模式接口的索引
    ifindex = iface_get_id(sock_fd, mondevice, handle->errbuf);
    # 如果获取失败，返回错误码
    if (ifindex == -1)
        return PCAP_ERROR;

    # 分配一个新的 netlink 消息对象
    msg = nlmsg_alloc();
    # 如果分配失败，格式化错误信息到错误缓冲区，返回错误码
    if (!msg) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: failed to allocate netlink msg", device);
        return PCAP_ERROR;
    }

    # 在消息对象中设置通用 netlink 消息头
    genlmsg_put(msg, 0, 0, genl_family_get_id(state->nl80211), 0,
            0, NL80211_CMD_DEL_INTERFACE, 0);
    # 在消息对象中添加属性，表示要删除的接口的索引
    NLA_PUT_U32(msg, NL80211_ATTR_IFINDEX, ifindex);

    # 发送消息对象到 netlink 套接字，如果发送失败，格式化错误信息到错误缓冲区，释放消息对象，返回错误码
    err = nl_send_auto_complete(state->nl_sock, msg);
    if (err < 0) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: nl_send_auto_complete failed deleting %s interface: %s",
            device, mondevice, nl_geterror(-err));
        nlmsg_free(msg);
        return PCAP_ERROR;
    }
    # 等待 netlink 套接字的应答，如果失败，格式化错误信息到错误缓冲区，释放消息对象，返回错误码
    err = nl_wait_for_ack(state->nl_sock);
    if (err < 0) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "%s: nl_wait_for_ack failed adding %s interface: %s",
            device, mondevice, nl_geterror(-err));
        nlmsg_free(msg);
        return PCAP_ERROR;
    }

    # 成功，释放消息对象，返回成功码
    nlmsg_free(msg);
    return 1;

# 定义一个函数，处理在删除监控模式接口时的错误情况
nla_put_failure:
    # 格式化错误信息到错误缓冲区
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "%s: nl_put failed deleting %s interface",
        device, mondevice);
    # 释放消息对象
    nlmsg_free(msg);
    # 返回错误码
    return PCAP_ERROR;
}
#endif /* HAVE_LIBNL */

# 定义一个函数，获取 pcap_t 结构体中的协议字段
static int pcap_protocol(pcap_t *handle)
{
    int protocol;

    # 获取协议字段的值
    protocol = handle->opt.protocol;
    # 如果协议字段的值为0，将其设置为 ETH_P_ALL
    if (protocol == 0)
        protocol = ETH_P_ALL;

    # 返回协议字段的值的网络字节序
    return htons(protocol);
}

# 定义一个函数，判断是否可以设置 Linux 系统下的 RFMON 模式
static int
pcap_can_set_rfmon_linux(pcap_t *handle)
{
#ifdef HAVE_LIBNL
    // 定义一个字符数组，用于存储物理设备路径，大小为 PATH_MAX+1
    char phydev_path[PATH_MAX+1];
    // 定义一个整型变量，用于存储函数返回值
    int ret;
#endif

    if (strcmp(handle->opt.device, "any") == 0) {
        /*
         * 如果设备为"any"，则监控模式没有意义。
         */
        return 0;
    }

#ifdef HAVE_LIBNL
    /*
     * 啊，没有找到一种通过 libnl 向 mac80211 设备询问其是否支持监控模式的方法；
     * 我们只能检查设备是否是 mac80211 设备，如果是，就假设设备支持监控模式。
     */
    ret = get_mac80211_phydev(handle, handle->opt.device, phydev_path,
        PATH_MAX);
    if (ret < 0)
        return ret;    /* 错误 */
    if (ret == 1)
        return 1;    /* mac80211 设备 */
#endif

    return 0;
}

/*
 * 从 /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors 中获取接口丢失数据包的数量。
 *
 * 与 /proc/net/dev 相比，这避免了计算软件丢包，但可能未实现并返回 0。
 * 作者没有找到检查支持的简单方法。
 */
static long long int
linux_get_stat(const char * if_name, const char * stat) {
    ssize_t bytes_read;
    int fd;
    char buffer[PATH_MAX];

    snprintf(buffer, sizeof(buffer), "/sys/class/net/%s/statistics/%s", if_name, stat);
    fd = open(buffer, O_RDONLY);
    if (fd == -1)
        return 0;

    bytes_read = read(fd, buffer, sizeof(buffer) - 1);
    close(fd);
    if (bytes_read == -1)
        return 0;
    buffer[bytes_read] = '\0';

    return strtoll(buffer, NULL, 10);
}

static long long int
linux_if_drops(const char * if_name)
{
    long long int missed = linux_get_stat(if_name, "rx_missed_errors");
    long long int fifo = linux_get_stat(if_name, "rx_fifo_errors");
    return missed + fifo;
}
# 在 Linux 平台清理 pcap_t 对象的资源
static void    pcap_cleanup_linux( pcap_t *handle )
{
    # 获取 pcap_linux 结构体指针
    struct pcap_linux *handlep = handle->priv;
    # 如果必须在关闭时执行某些操作
    if (handlep->must_do_on_close != 0) {
        # 如果必须删除监控模式接口
        if (handlep->must_do_on_close & MUST_DELETE_MONIF) {
            # 初始化 nl80211 状态
            ret = nl80211_init(handle, &nlstate, handlep->device);
            # 如果初始化成功
            if (ret >= 0) {
                # 删除监控接口
                ret = del_mon_if(handle, handle->fd, &nlstate,
                    handlep->device, handlep->mondevice);
                # 清理 nl80211 状态
                nl80211_cleanup(&nlstate);
            }
            # 如果删除失败
            if (ret < 0) {
                # 输出错误信息
                fprintf(stderr,
                    "Can't delete monitor interface %s (%s).\n"
                    "Please delete manually.\n",
                    handlep->mondevice, handle->errbuf);
            }
        }
        # 从需要关闭的 pcap_t 列表中移除当前 pcap_t
        pcap_remove_from_pcaps_to_close(handle);
    }
    # 如果文件描述符有效
    if (handle->fd != -1) {
        # 销毁环形缓冲区
        destroy_ring(handle);
    }
    # 如果单次捕获缓冲区不为空
    if (handlep->oneshot_buffer != NULL) {
        # 释放单次捕获缓冲区
        free(handlep->oneshot_buffer);
        handlep->oneshot_buffer = NULL;
    }
}
    # 如果监控设备不为空
    if (handlep->mondevice != NULL) {
        # 释放监控设备内存
        free(handlep->mondevice);
        # 将监控设备指针置为空
        handlep->mondevice = NULL;
    }
    # 如果设备不为空
    if (handlep->device != NULL) {
        # 释放设备内存
        free(handlep->device);
        # 将设备指针置为空
        handlep->device = NULL;
    }

    # 如果中断轮询的文件描述符不等于-1
    if (handlep->poll_breakloop_fd != -1) {
        # 关闭中断轮询的文件描述符
        close(handlep->poll_breakloop_fd);
        # 将中断轮询的文件描述符置为-1
        handlep->poll_breakloop_fd = -1;
    }
    # 清理现场共享资源
    pcap_cleanup_live_common(handle);
#ifdef HAVE_TPACKET3
/*
 * Some versions of TPACKET_V3 have annoying bugs/misfeatures
 * around which we have to work.  Determine if we have those
 * problems or not.
 * 3.19 is the first release with a fixed version of
 * TPACKET_V3.  We treat anything before that as
 * not having a fixed version; that may really mean
 * it has *no* version.
 */
static int has_broken_tpacket_v3(void)
{
    struct utsname utsname;
    const char *release;
    long major, minor;
    int matches, verlen;

    /* No version information, assume broken. */
    if (uname(&utsname) == -1)
        return 1;
    release = utsname.release;

    /* A malformed version, ditto. */
    matches = sscanf(release, "%ld.%ld%n", &major, &minor, &verlen);
    if (matches != 2)
        return 1;
    if (release[verlen] != '.' && release[verlen] != '\0')
        return 1;

    /* OK, a fixed version. */
    if (major > 3 || (major == 3 && minor >= 19))
        return 0;

    /* Too old :( */
    return 1;
}
#endif

/*
 * Set the timeout to be used in poll() with memory-mapped packet capture.
 */
static void
set_poll_timeout(struct pcap_linux *handlep)
{
#ifdef HAVE_TPACKET3
    int broken_tpacket_v3 = has_broken_tpacket_v3();
#endif
    if (handlep->timeout == 0) {
#ifdef HAVE_TPACKET3
        /*
         * XXX - due to a set of (mis)features in the TPACKET_V3
         * kernel code prior to the 3.19 kernel, blocking forever
         * with a TPACKET_V3 socket can, if few packets are
         * arriving and passing the socket filter, cause most
         * packets to be dropped.  See libpcap issue #335 for the
         * full painful story.
         *
         * The workaround is to have poll() time out very quickly,
         * so we grab the frames handed to us, and return them to
         * the kernel, ASAP.
         */
        if (handlep->tp_version == TPACKET_V3 && broken_tpacket_v3)
            handlep->poll_timeout = 1;    /* don't block for very long */
        else
#endif
            handlep->poll_timeout = -1;    /* block forever */
    } else if (handlep->timeout > 0) {
#ifdef HAVE_TPACKET3
        /*
         * 对于 TPACKET_V3，超时由内核处理，因此永远阻塞；
         * 这样，我们就不会得到额外的超时。
         * 但是，如果 TPACKET_V3 损坏，就不要这样做。
         */
        if (handlep->tp_version == TPACKET_V3 && !broken_tpacket_v3)
            handlep->poll_timeout = -1;    /* 永远阻塞，让 TPACKET_V3 唤醒我们 */
        else
#endif
            handlep->poll_timeout = handlep->timeout;    /* 阻塞指定的时间 */
    } else {
        /*
         * 非阻塞模式；我们调用 poll() 来获取错误指示，但我们不希望它等待任何东西。
         */
        handlep->poll_timeout = 0;
    }
}

static void pcap_breakloop_linux(pcap_t *handle)
{
    pcap_breakloop_common(handle);
    struct pcap_linux *handlep = handle->priv;

    uint64_t value = 1;
    /* XXX - 如果这失败了怎么办？ */
    if (handlep->poll_breakloop_fd != -1)
        (void)write(handlep->poll_breakloop_fd, &value, sizeof(value));
}

/*
 * 设置插入 VLAN 标记的偏移量。
 * 应该是类型字段的偏移量。
 */
static void
set_vlan_offset(pcap_t *handle)
{
    struct pcap_linux *handlep = handle->priv;

    switch (handle->linktype) {

    case DLT_EN10MB:
        /*
         * 类型字段在目的地址和源地址之后。
         */
        handlep->vlan_offset = 2 * ETH_ALEN;
        break;

    case DLT_LINUX_SLL:
        /*
         * 类型字段在 DLT_LINUX_SLL 标头的最后 2 个字节中。
         */
        handlep->vlan_offset = SLL_HDR_LEN - 2;
        break;

    default:
        handlep->vlan_offset = -1; /* 未知 */
        break;
    }
}
/*
 * 从给定设备获取一个实时捕获的句柄。你可以传入 NULL 作为设备来获取所有数据包（当然没有链路层信息）。如果你传入 1 作为 promisc，接口将被设置为混杂模式（XXX: 我认为这种用法应该被弃用，并且应该添加函数以后允许修改这些值 -- Torsten）。
 */
static int
pcap_activate_linux(pcap_t *handle)
{
    struct pcap_linux *handlep = handle->priv;
    const char    *device;
    int        is_any_device;
    struct ifreq    ifr;
    int        status = 0;
    int        status2 = 0;
    int        ret;

    device = handle->opt.device;

    /*
     * 确保我们传入的设备名称能够适应我们可能在设备上执行的 ioctl；如果不能，返回“没有这样的设备”的指示，因为 Linux 内核不应该支持创建名称无法适应这些 ioctl 的设备。
     *
     * “适应”意味着“适应，包括空终止符”，因此如果长度（不包括空终止符）大于或等于我们将要复制到其中的字段的大小，那就不适应。
     */
    if (strlen(device) >= sizeof(ifr.ifr_name)) {
        /*
         * 没有更多要说的了，所以清除错误消息。
         */
        handle->errbuf[0] = '\0';
        status = PCAP_ERROR_NO_SUCH_DEVICE;
        goto fail;
    }

    /*
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为允许的最大值。
     *
     * 如果某些应用程序真的 *需要* 更大的快照长度，我们应该只需增加 MAXIMUM_SNAPLEN。
     */
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;

    handlep->device    = strdup(device);
    # 如果设备为空，则将错误信息格式化为errno，并返回错误状态
    if (handlep->device == NULL) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "strdup");
        status = PCAP_ERROR;
        goto fail;
    }

    # 如果设备为"any"，则不绑定到特定设备，而是查看所有设备
    is_any_device = (strcmp(device, "any") == 0);
    if (is_any_device) {
        # 如果处于混杂模式，则将其关闭，并生成警告信息
        if (handle->opt.promisc) {
            handle->opt.promisc = 0;
            /* Just a warning. */
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Promiscuous mode not supported on the \"any\" device");
            status = PCAP_WARNING_PROMISC_NOTSUP;
        }
    }

    # 复制超时值
    handlep->timeout = handle->opt.timeout;

    # 如果处于混杂模式，则获取接口丢包计数
    if (handle->opt.promisc)
        handlep->sysfs_dropped = linux_if_drops(handlep->device);

    # 如果指定了"any"设备，则尝试打开SOCK_DGRAM；否则打开SOCK_RAW
    ret = setup_socket(handle, is_any_device);
    if (ret < 0) {
        # 如果设置套接字失败，则返回错误状态
        status = ret;
        goto fail;
    }
    # 尝试设置内存映射访问
    ret = setup_mmapped(handle, &status);
    if (ret == -1) {
        # 如果设置内存映射失败，则返回错误状态
        goto fail;
    }
    /*
     * 我们成功了。status已经设置为要返回的状态，可能是0，也可能是PCAP_WARNING_值。
     */
    /*
     * 现在我们已经激活了mmap环，我们可以设置正确的协议。
     */
    if ((status2 = iface_bind(handle->fd, handlep->ifindex,
        handle->errbuf, pcap_protocol(handle))) != 0) {
        status = status2;
        goto fail;
    }

    handle->inject_op = pcap_inject_linux;  // 设置Linux下的数据包注入操作
    handle->setfilter_op = pcap_setfilter_linux;  // 设置Linux下的过滤器操作
    handle->setdirection_op = pcap_setdirection_linux;  // 设置Linux下的数据流向操作
    handle->set_datalink_op = pcap_set_datalink_linux;  // 设置Linux下的数据链路类型操作
    handle->setnonblock_op = pcap_setnonblock_linux;  // 设置Linux下的非阻塞操作
    handle->getnonblock_op = pcap_getnonblock_linux;  // 获取Linux下的非阻塞状态
    handle->cleanup_op = pcap_cleanup_linux;  // 清理Linux下的资源
    handle->stats_op = pcap_stats_linux;  // 获取Linux下的统计信息
    handle->breakloop_op = pcap_breakloop_linux;  // 中断Linux下的循环

    switch (handlep->tp_version) {

    case TPACKET_V2:
        handle->read_op = pcap_read_linux_mmap_v2;  // 设置Linux下的mmap方式读取数据包操作
        break;
#ifdef HAVE_TPACKET3
    // 如果定义了 HAVE_TPACKET3，则执行以下代码块
    case TPACKET_V3:
        // 设置 handle 的 read_op 为 pcap_read_linux_mmap_v3 函数
        handle->read_op = pcap_read_linux_mmap_v3;
        // 退出 switch 语句
        break;
#endif
    // 设置 handle 的 oneshot_callback 为 pcap_oneshot_linux 函数
    handle->oneshot_callback = pcap_oneshot_linux;
    // 设置 handle 的 selectable_fd 为 handle 的 fd
    handle->selectable_fd = handle->fd;

    // 返回 status
    return status;

fail:
    // 调用 pcap_cleanup_linux 函数，清理 handle
    pcap_cleanup_linux(handle);
    // 返回 status
    return status;
}

static int
pcap_set_datalink_linux(pcap_t *handle, int dlt)
{
    // 设置 handle 的 linktype 为 dlt
    handle->linktype = dlt;

    /*
     * Update the offset at which to insert VLAN tags for the
     * new link-layer type.
     */
    // 调用 set_vlan_offset 函数，更新插入 VLAN 标签的偏移量
    set_vlan_offset(handle);

    // 返回 0
    return 0;
}

/*
 * linux_check_direction()
 *
 * Do checks based on packet direction.
 */
static inline int
linux_check_direction(const pcap_t *handle, const struct sockaddr_ll *sll)
{
    // 将 handle 的 priv 强制转换为 pcap_linux 结构体指针，赋值给 handlep
    struct pcap_linux    *handlep = handle->priv;
    # 如果这是一个传出的数据包
    if (sll->sll_pkttype == PACKET_OUTGOING) {
        '''
        # 传出的数据包。
        # 如果这是来自回环设备的数据包，则拒绝它；
        # 我们将会把这个数据包视为传入的数据包，
        # 我们不希望看到它两次。
        '''
        if (sll->sll_ifindex == handlep->lo_ifindex)
            return 0;

        '''
        # 如果这是一个传出的 CAN 或 CAN FD 帧，并且用户不仅想要传出的数据包，
        # 则拒绝它；CAN 设备和驱动程序以及 CAN 栈总是安排将传输的数据包回环，
        # 因此它们也会出现为传入的数据包。我们不希望重复的数据包，
        # 并且我们无法轻松区分 CAN 层回环的数据包和 CAN 层接收的数据包，
        # 因此我们消除这个数据包。
        #
        # 我们通过检查设备的硬件类型是否为 ARPHRD_CAN 来检查这是否是 CAN 或 CAN FD 帧。
        '''
        if (sll->sll_hatype == ARPHRD_CAN &&
             handle->direction != PCAP_D_OUT)
            return 0;

        '''
        # 如果用户只想要传入的数据包，则拒绝它。
        '''
        if (handle->direction == PCAP_D_IN)
            return 0;
    } else {
        '''
        # 传入的数据包。
        # 如果用户只想要传出的数据包，则拒绝它。
        '''
        if (handle->direction == PCAP_D_OUT)
            return 0;
    }
    # 返回 1，表示接受这个数据包
    return 1;
/*
 * 检查 pcap_t 绑定的设备是否仍然存在。
 * 我们通过询问套接字绑定的地址来实现，然后检查地址中的 ifindex 是否为 -1，表示“该设备已经不存在”，或者是其他值，表示“该设备仍然存在”。
 */
static int
device_still_exists(pcap_t *handle)
{
    struct pcap_linux *handlep = handle->priv;
    struct sockaddr_ll addr;
    socklen_t addr_len;

    /*
     * 如果 handlep->ifindex 为 -1，则套接字未绑定，意味着我们正在捕获“任意”设备；该设备永远不会消失。（它也不应该被配置为关闭，所以我们甚至不应该到达这里，但让我们确保一下。）
     */
    if (handlep->ifindex == -1)
        return (1);    /* 它仍然存在 */

    /*
     * 现在尝试获取套接字的地址。
     */
    addr_len = sizeof (addr);
    if (getsockname(handle->fd, (struct sockaddr *) &addr, &addr_len) == -1) {
        /*
         * 错误 - 报告错误并返回 -1。
         */
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "getsockname failed");
        return (-1);
    }
    if (addr.sll_ifindex == -1) {
        /*
         * 这意味着设备已经消失。
         */
        return (0);
    }

    /*
     * 设备可能刚刚下线。
     */
    return (1);
}

static int
pcap_inject_linux(pcap_t *handle, const void *buf, int size)
{
    struct pcap_linux *handlep = handle->priv;
    int ret;

    if (handlep->ifindex == -1) {
        /*
         * 我们不支持在“任意”设备上发送数据包。
         */
        pcap_strlcpy(handle->errbuf,
            "Sending packets isn't supported on the \"any\" device",
            PCAP_ERRBUF_SIZE);
        return (-1);
    }
}
    # 如果套接字处于 cooked 模式
    if (handlep->cooked) {
        # 不支持在 cooked 模式下发送数据包
        # XXX - 如何在绑定的 cooked 模式套接字上发送数据？
        # 需要使用 "sendto()" 吗？
        pcap_strlcpy(handle->errbuf,
            "Sending packets isn't supported in cooked mode",
            PCAP_ERRBUF_SIZE);
        # 返回错误
        return (-1);
    }

    # 发送数据包
    ret = (int)send(handle->fd, buf, size, 0);
    # 如果发送失败
    if (ret == -1) {
        # 格式化错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        # 返回错误
        return (-1);
    }
    # 返回发送的字节数
    return (ret);
}
/*
 * 获取给定数据包捕获句柄的统计信息。
 */
static int
pcap_stats_linux(pcap_t *handle, struct pcap_stat *stats)
{
    // 获取私有数据结构 pcap_linux 指针
    struct pcap_linux *handlep = handle->priv;
#ifdef HAVE_TPACKET3
    /*
     * 对于使用 TPACKET_V2 的套接字，结构体 tpacket_stats_v3 的末尾的额外内容将不会被填充，
     * 我们也不会查看它，所以即使对于这些套接字也是可以的。
     * 此外，内核中的 PF_PACKET 套接字代码只使用长度参数来计算要复制多少数据出去，并指示复制出去多少数据，
     * 所以基于 tpacket_stats 结构体的大小是可以的。
     *
     * XXX - 实际上，对于 V3 套接字，可能只使用 tpacket_stats 结构体也是可以的，因为我们不关心 tp_freeze_q_cnt 统计信息。
     */
    struct tpacket_stats_v3 kstats;
#else /* HAVE_TPACKET3 */
    struct tpacket_stats kstats;
#endif /* HAVE_TPACKET3 */
    // 结构体 tpacket_stats 的长度
    socklen_t len = sizeof (struct tpacket_stats);

    long long if_dropped = 0;

    /*
     * 要填充 ps_ifdrop，我们解析 /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors
     * 来获取数字
     */
    if (handle->opt.promisc)
    {
        /*
         * XXX - is there any reason to do this by remembering
         * the last counts value, subtracting it from the
         * current counts value, and adding that to stat.ps_ifdrop,
         * maintaining stat.ps_ifdrop as a count, rather than just
         * saving the *initial* counts value and setting
         * stat.ps_ifdrop to the difference between the current
         * value and the initial value?
         *
         * One reason might be to handle the count wrapping
         * around, on platforms where the count is 32 bits
         * and where you might get more than 2^32 dropped
         * packets; is there any other reason?
         *
         * (We maintain the count as a long long int so that,
         * if the kernel maintains the counts as 64-bit even
         * on 32-bit platforms, we can handle the real count.
         *
         * Unfortunately, we can't report 64-bit counts; we
         * need a better API for reporting statistics, such as
         * one that reports them in a style similar to the
         * pcapng Interface Statistics Block, so that 1) the
         * counts are 64-bit, 2) it's easier to add new statistics
         * without breaking the ABI, and 3) it's easier to
         * indicate to a caller that wants one particular
         * statistic that it's not available by just not supplying
         * it.)
         */
        // 保存上一次的 dropped 数据
        if_dropped = handlep->sysfs_dropped;
        // 获取当前的 dropped 数据
        handlep->sysfs_dropped = linux_if_drops(handlep->device);
        // 将当前 dropped 数据与上一次的 dropped 数据相减，加到统计数据中
        handlep->stat.ps_ifdrop += (u_int)(handlep->sysfs_dropped - if_dropped);
    }

    /*
     * Try to get the packet counts from the kernel.
     */
    }

    // 将错误信息格式化为错误消息并返回 -1
    pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE, errno,
        "failed to get statistics from socket");
    return -1;
}

/*
 * Description string for the "any" device.
 */
static const char any_descr[] = "Pseudo-device that captures on all interfaces";

/*
 * A PF_PACKET socket can be bound to any network interface.
 */
static int
can_be_bound(const char *name _U_)
{
    return (1);
}

/*
 * Get a socket to use with various interface ioctls.
 */
static int
get_if_ioctl_socket(void)
{
    int fd;

    // 创建一个 AF_NETLINK 套接字
    fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC);
    if (fd != -1) {
        /*
         * OK, let's make sure we can do an SIOCGIFNAME
         * ioctl.
         */
        struct ifreq ifr;

        // 初始化 ifreq 结构体
        memset(&ifr, 0, sizeof(ifr));
        // 执行 SIOCGIFNAME ioctl
        if (ioctl(fd, SIOCGIFNAME, &ifr) == 0 ||
            errno != EOPNOTSUPP) {
            /*
             * It succeeded, or failed for some reason
             * other than "netlink sockets don't support
             * device ioctls".  Go with the AF_NETLINK
             * socket.
             */
            return (fd);
        }

        /*
         * OK, that didn't work, so it's as bad as "netlink
         * sockets aren't available".  Close the socket and
         * drive on.
         */
        close(fd);
    }

    /*
     * Now try an AF_UNIX socket.
     */
    // 创建一个 AF_UNIX 套接字
    fd = socket(AF_UNIX, SOCK_RAW, 0);
    if (fd != -1) {
        /*
         * OK, we got it!
         */
        return (fd);
    }

    /*
     * Now try an AF_INET6 socket.
     */
    // 创建一个 AF_INET6 套接字
    fd = socket(AF_INET6, SOCK_DGRAM, 0);
    if (fd != -1) {
        return (fd);
    }

    /*
     * Now try an AF_INET socket.
     *
     * XXX - if that fails, is there anything else we should try?
     * AF_CAN, for embedded systems in vehicles, in case they're
     * built without Internet protocol support?  Any other socket
     * types popular in non-Internet embedded systems?
     */
    // 创建一个 AF_INET 套接字
    return (socket(AF_INET, SOCK_DGRAM, 0));
}

/*
 * Get additional flags for a device, using SIOCGIFMEDIA.
 */
static int
get_if_flags(const char *name, bpf_u_int32 *flags, char *errbuf)
{
    int sock;
    # 打开文件句柄
    FILE *fh;
    # 用于存储网络接口类型的变量
    unsigned int arptype;
    # 用于存储网络接口信息的结构体
    struct ifreq ifr;
    # 用于存储网络接口的 ethtool 信息的结构体
    struct ethtool_value info;

    # 如果是回环设备，则不是无线设备，不适用于“连接”/“断开连接”
    if (*flags & PCAP_IF_LOOPBACK) {
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        return 0;
    }

    # 获取网络接口的 ioctl 套接字
    sock = get_if_ioctl_socket();
    # 如果获取失败，则返回错误
    if (sock == -1) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno,
            "Can't create socket to get ethtool information for %s",
            name);
        return -1;
    }

    # 判断网络类型，是有线还是无线
    if (is_wifi(name)) {
        # 如果是 Wi-Fi，则是无线设备
        *flags |= PCAP_IF_WIRELESS;
    } else {
        /*
         * OK, what does /sys/class/net/{if_name}/type contain?
         * (We don't use that for Wi-Fi, as it'll report
         * "Ethernet", i.e. ARPHRD_ETHER, for non-monitor-
         * mode devices.)
         */
        // 定义一个字符指针变量
        char *pathstr;

        // 生成路径字符串，如果失败则返回错误
        if (asprintf(&pathstr, "/sys/class/net/%s/type", name) == -1) {
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "%s: Can't generate path name string for /sys/class/net device",
                name);
            close(sock);
            return -1;
        }
        // 以只读方式打开文件
        fh = fopen(pathstr, "r");
        // 如果文件打开成功
        if (fh != NULL) {
            // 从文件中读取一个无符号整数
            if (fscanf(fh, "%u", &arptype) == 1) {
                /*
                 * OK, we got an ARPHRD_ type; what is it?
                 */
                // 根据不同的类型进行不同的处理
                switch (arptype) {

                case ARPHRD_LOOPBACK:
                    /*
                     * These are types to which
                     * "connected" and "disconnected"
                     * don't apply, so don't bother
                     * asking about it.
                     *
                     * XXX - add other types?
                     */
                    // 关闭套接字和文件，释放内存，然后返回0
                    close(sock);
                    fclose(fh);
                    free(pathstr);
                    return 0;

                case ARPHRD_IRDA:
                case ARPHRD_IEEE80211:
                case ARPHRD_IEEE80211_PRISM:
                case ARPHRD_IEEE80211_RADIOTAP:
#ifdef ARPHRD_IEEE802154
                // 如果是 IEEE802154 类型的网络接口
                case ARPHRD_IEEE802154:
#endif
#ifdef ARPHRD_IEEE802154_MONITOR
                // 如果是 IEEE802154_MONITOR 类型的网络接口
                case ARPHRD_IEEE802154_MONITOR:
#endif
#ifdef ARPHRD_6LOWPAN
                // 如果是 6LOWPAN 类型的网络接口
                case ARPHRD_6LOWPAN:
#endif
                    /*
                     * 各种无线类型。
                     */
                    // 设置标志位，表示是无线接口
                    *flags |= PCAP_IF_WIRELESS;
                    // 结束 case 语句块
                    break;
                }
            }
            // 关闭文件句柄
            fclose(fh);
        }
        // 释放动态分配的内存
        free(pathstr);
    }

#ifdef ETHTOOL_GLINK
    // 初始化 ifr 结构体
    memset(&ifr, 0, sizeof(ifr));
    // 将接口名拷贝到 ifr 结构体中
    pcap_strlcpy(ifr.ifr_name, name, sizeof(ifr.ifr_name));
    // 设置 info 结构体的命令为 ETHTOOL_GLINK
    info.cmd = ETHTOOL_GLINK;
    /*
     * XXX - while Valgrind handles SIOCETHTOOL and knows that
     * the ETHTOOL_GLINK command sets the .data member of the
     * structure, Memory Sanitizer doesn't yet do so:
     *
     *    https://bugs.llvm.org/show_bug.cgi?id=45814
     *
     * For now, we zero it out to squelch warnings; if the bug
     * in question is fixed, we can remove this.
     */
    // 将 info 结构体的数据成员清零，以消除警告
    info.data = 0;
    // 设置 ifr 结构体的数据成员为 info 结构体的地址
    ifr.ifr_data = (caddr_t)&info;
    # 如果调用 ioctl 函数失败
    if (ioctl(sock, SIOCETHTOOL, &ifr) == -1) {
        # 保存错误码
        int save_errno = errno;

        # 根据错误码进行不同的处理
        switch (save_errno) {

        # 如果不支持该操作或参数无效
        case EOPNOTSUPP:
        case EINVAL:
            # 设置连接状态不适用标志
            *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
            # 关闭套接字
            close(sock);
            return 0;

        # 如果设备不存在
        case ENODEV:
            # 关闭套接字
            close(sock);
            return 0;

        # 其他错误
        default:
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                save_errno,
                "%s: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed",
                name);
            # 关闭套接字
            close(sock);
            return -1;
        }
    }

    # 检查设备是否连接
    if (info.data) {
        # 设置连接状态已连接标志
        *flags |= PCAP_IF_CONNECTION_STATUS_CONNECTED;
    } else {
        # 设置连接状态未连接标志
        *flags |= PCAP_IF_CONNECTION_STATUS_DISCONNECTED;
    }
#endif

    # 关闭套接字
    close(sock);
    # 返回 0 表示成功
    return 0;
}

# 查找网络接口设备
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    '''
     * 首先获取常规接口的列表。
     '''
    if (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
        get_if_flags) == -1)
        return (-1);    # 失败

    '''
     * 添加 "any" 设备。
     * 因为它指的是所有网络设备，而不是任何特定的网络设备，所以 "连接" vs. "未连接" 的概念不适用。
     '''
    if (add_dev(devlistp, "any",
        PCAP_IF_UP|PCAP_IF_RUNNING|PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
        any_descr, errbuf) == NULL)
        return (-1);

    return (0);
}

'''
 * 设置方向标志：在转发单个设备上我们接受哪些数据包？IN、OUT 还是两者都？
 '''
static int
pcap_setdirection_linux(pcap_t *handle, pcap_direction_t d)
{
    '''
     * 在这一点上，可以保证 d 是一个有效的方向值。
     '''
    handle->direction = d;
    return 0;
}

static int
is_wifi(const char *device)
{
    char *pathstr;
    struct stat statb;

    '''
     * 查看是否有 sysfs 无线目录。
     * 如果有，那么它就是一个无线接口。
     '''
    if (asprintf(&pathstr, "/sys/class/net/%s/wireless", device) == -1) {
        '''
         * 在这里放弃。
         '''
        return 0;
    }
    if (stat(pathstr, &statb) == 0) {
        free(pathstr);
        return 1;
    }
    free(pathstr);

    return 0;
}
/*
 *  Linux使用ARP硬件类型来识别接口的类型。pcap使用DLT_xxx常量来表示这一点。
 *  此函数接受一个指向"pcap_t"的指针和一个ARPHRD_xxx常量作为参数，并将"handle->linktype"设置为适当的DLT_XXX常量，并将"handle->offset"设置为适当的值（使"handle->offset"加上链路层头长度成为4的倍数，以便在捕获数据包时链路层有效载荷对齐到4字节边界）。
 *  （如果偏移量在这里没有设置，它将为0；在不应为0的情况下添加相应的代码。）
 *
 *  如果"cooked_ok"非零，我们可以使用DLT_LINUX_SLL并以cooked模式捕获；否则，我们无法使用cooked模式，因此必须选择一些在原始模式下工作的类型，或者失败。
 *
 *  如果无法映射类型，则将链路类型设置为-1。
 */
static void map_arphrd_to_dlt(pcap_t *handle, int arptype,
                  const char *device, int cooked_ok)
{
    static const char cdma_rmnet[] = "cdma_rmnet";

    switch (arptype) {

    case ARPHRD_METRICOM:
    case ARPHRD_LOOPBACK:
        handle->linktype = DLT_EN10MB;
        handle->offset = 2;
        break;

    case ARPHRD_EETHER:
        handle->linktype = DLT_EN3MB;
        break;

    case ARPHRD_AX25:
        handle->linktype = DLT_AX25_KISS;
        break;

    case ARPHRD_PRONET:
        handle->linktype = DLT_PRONET;
        break;

    case ARPHRD_CHAOS:
        handle->linktype = DLT_CHAOS;
        break;
#ifndef ARPHRD_CAN
#define ARPHRD_CAN 280
#endif
    case ARPHRD_CAN:
        handle->linktype = DLT_CAN_SOCKETCAN;
        break;

#ifndef ARPHRD_IEEE802_TR
#define ARPHRD_IEEE802_TR 800    /* From Linux 2.4 */
#endif
    case ARPHRD_IEEE802_TR:
    case ARPHRD_IEEE802:
        handle->linktype = DLT_IEEE802;
        handle->offset = 2;
        break;

    case ARPHRD_ARCNET:
        handle->linktype = DLT_ARCNET_LINUX;
        break;
#ifndef ARPHRD_FDDI    /* From Linux 2.2.13 */
#define ARPHRD_FDDI    774
#endif
    // 如果未定义 ARPHRD_FDDI，则定义为 774

    case ARPHRD_FDDI:
        // 如果是 FDDI 类型的网络
        handle->linktype = DLT_FDDI;
        // 设置数据链路类型为 FDDI
        handle->offset = 3;
        // 设置偏移量为 3
        break;

#ifndef ARPHRD_ATM  /* FIXME: How to #include this? */
#define ARPHRD_ATM 19
#endif
    // 如果未定义 ARPHRD_ATM，则定义为 19
    case ARPHRD_ATM:
        /*
         * ATM网络的经典IP实现在Linux中支持RFC 1483所称的“LLC封装”，
         * 其中每个数据包都有一个LLC头，可能还有一个SNAP头，以及
         * RFC 1483所称的“基于VC的多路复用”，其中
         * 不同的虚拟电路携带不同的网络层协议，并且数据包前面没有附加头部。
         *
         * 它们都具有ARPHRD_类型为ARPHRD_ATM，因此
         * 无法使用ARPHRD_类型来确定捕获的数据包是否有LLC头，
         * 而且，虽然有一个套接字ioctl来*设置*封装类型，但没有ioctl来*获取*封装类型。
         *
         * 这意味着
         *
         *    分析Linux经典IP帧的程序
         *    必须检查是否有LLC头，并根据是否看到LLC头来分析
         *    作为LLC封装或原始IP的帧（我不知道除了IP之外是否还有其他流量
         *    会出现在套接字上，或者Linux经典IP代码中是否有IPv6的支持）;
         *
         *    过滤表达式必须编译成
         *    检查是否有LLC头并执行正确操作的代码。
         *
         * 这两者都很麻烦 - 而且，至少在支持PF_PACKET套接字的系统上，
         * 我们不必忍受这些麻烦；相反，我们可以直接以混合模式捕获。如果可以的话，我们就这样做。
         * 否则，我们就失败。
         */
        if (cooked_ok)
            handle->linktype = DLT_LINUX_SLL;
        else
            handle->linktype = -1;
        break;
#ifndef ARPHRD_IEEE80211  /* From Linux 2.4.6 */
#define ARPHRD_IEEE80211 801
#endif
    // 如果未定义 ARPHRD_IEEE80211，则定义为 801

    case ARPHRD_IEEE80211:
        // 如果是 ARPHRD_IEEE80211 类型的设备
        handle->linktype = DLT_IEEE802_11;
        // 设置数据链路类型为 DLT_IEEE802_11
        break;

#ifndef ARPHRD_IEEE80211_PRISM  /* From Linux 2.4.18 */
#define ARPHRD_IEEE80211_PRISM 802
#endif
    // 如果未定义 ARPHRD_IEEE80211_PRISM，则定义为 802

    case ARPHRD_IEEE80211_PRISM:
        // 如果是 ARPHRD_IEEE80211_PRISM 类型的设备
        handle->linktype = DLT_PRISM_HEADER;
        // 设置数据链路类型为 DLT_PRISM_HEADER
        break;

#ifndef ARPHRD_IEEE80211_RADIOTAP /* new */
#define ARPHRD_IEEE80211_RADIOTAP 803
#endif
    // 如果未定义 ARPHRD_IEEE80211_RADIOTAP，则定义为 803

    case ARPHRD_IEEE80211_RADIOTAP:
        // 如果是 ARPHRD_IEEE80211_RADIOTAP 类型的设备
        handle->linktype = DLT_IEEE802_11_RADIO;
        // 设置数据链路类型为 DLT_IEEE802_11_RADIO
        break;
    # 如果是 ARPHRD_PPP 类型的接口
    case ARPHRD_PPP:
        '''
        # 一些内核中的 PPP 代码根本不向 PF_PACKET 套接字提供链路层头部；
        # 其他 PPP 代码提供 PPP 链路层头部（"syncppp.c"）；
        # 一些 PPP 代码可能提供随机的链路层头部（例如 PPP over ISDN - Ethereal 中有处理 PPP-over-ISDN 捕获的代码，Ethereal 开发人员必须试图启发式地确定特定数据包具有哪种奇怪的链路层头部）。
        #
        # 因此，我们只是放任不管，并在可能的情况下以 cooked 模式运行所有 PPP 接口；否则，我们暂时将其视为 DLT_RAW - 如果有人需要在提供链路层头部的 2.0[.x] 内核上捕获 PPP 设备，他们将不得不在这里添加代码来映射到适当的 DLT_ 类型（如果需要，可能添加新的 DLT_ 类型）。
        '''
        # 如果 cooked 模式可用，则设置链路类型为 DLT_LINUX_SLL
        if (cooked_ok)
            handle->linktype = DLT_LINUX_SLL;
        else:
            '''
            # XXX - 在这里处理 ISDN 类型？我们无法退回到 cooked 套接字，因此我们必须从设备名称中弄清楚它使用的链路层封装类型，并将其映射到适当的 DLT_ 值，这意味着我们将 "isdnN" 设备映射到 DLT_RAW（它们提供没有链路层头部的原始 IP 数据包），将 "isdY" 设备映射到一个新的 DLT_I4L_IP 类型，它只有以太网数据包类型作为链路层头部。
            #
            # 但有时在捕获 ISDN 设备时，我们似乎会得到随机的垃圾数据在链路层头部....
            '''
            # 否则，设置链路类型为 DLT_RAW
            handle->linktype = DLT_RAW;
        break;
#ifndef ARPHRD_CISCO
#define ARPHRD_CISCO 513 /* previously ARPHRD_HDLC */
#endif
    // 如果 ARPHRD_CISCO 未定义，则定义为 513，之前是 ARPHRD_HDLC
    case ARPHRD_CISCO:
        // 设置链路类型为 DLT_C_HDLC
        handle->linktype = DLT_C_HDLC;
        break;

    /* Not sure if this is correct for all tunnels, but it
     * works for CIPE */
    // 如果 ARPHRD_TUNNEL 未定义，则定义为 776，来自 Linux 2.2.13
    case ARPHRD_TUNNEL:
#ifndef ARPHRD_SIT
#define ARPHRD_SIT 776    /* From Linux 2.2.13 */
#endif
    case ARPHRD_SIT:
    case ARPHRD_CSLIP:
    case ARPHRD_SLIP6:
    case ARPHRD_CSLIP6:
    case ARPHRD_ADAPT:
    case ARPHRD_SLIP:
#ifndef ARPHRD_RAWHDLC
#define ARPHRD_RAWHDLC 518
#endif
    case ARPHRD_RAWHDLC:
#ifndef ARPHRD_DLCI
#define ARPHRD_DLCI 15
#endif
    case ARPHRD_DLCI:
        /*
         * XXX - should some of those be mapped to DLT_LINUX_SLL
         * instead?  Should we just map all of them to DLT_LINUX_SLL?
         */
        // 设置链路类型为 DLT_RAW
        handle->linktype = DLT_RAW;
        break;

#ifndef ARPHRD_FRAD
#define ARPHRD_FRAD 770
#endif
    case ARPHRD_FRAD:
        // 设置链路类型为 DLT_FRELAY
        handle->linktype = DLT_FRELAY;
        break;

    case ARPHRD_LOCALTLK:
        // 设置链路类型为 DLT_LTALK
        handle->linktype = DLT_LTALK;
        break;

    case 18:
        /*
         * RFC 4338 defines an encapsulation for IP and ARP
         * packets that's compatible with the RFC 2625
         * encapsulation, but that uses a different ARP
         * hardware type and hardware addresses.  That
         * ARP hardware type is 18; Linux doesn't define
         * any ARPHRD_ value as 18, but if it ever officially
         * supports RFC 4338-style IP-over-FC, it should define
         * one.
         *
         * For now, we map it to DLT_IP_OVER_FC, in the hopes
         * that this will encourage its use in the future,
         * should Linux ever officially support RFC 4338-style
         * IP-over-FC.
         */
        // 设置链路类型为 DLT_IP_OVER_FC
        handle->linktype = DLT_IP_OVER_FC;
        break;

#ifndef ARPHRD_FCPP
#define ARPHRD_FCPP    784
#endif
    case ARPHRD_FCPP:
#ifndef ARPHRD_FCAL
#define ARPHRD_FCAL    785
#endif
    case ARPHRD_FCAL:
#ifndef ARPHRD_FCPL
#define ARPHRD_FCPL    786
#endif
    # 如果是以太网类型为FCPL
#ifndef ARPHRD_FCFABRIC
#define ARPHRD_FCFABRIC    787
#endif
#ifndef ARPHRD_IRDA
#define ARPHRD_IRDA    783
#endif
    case ARPHRD_IRDA:
        /* 不期望从这些接口中得到 IP 数据包... */
        handle->linktype = DLT_LINUX_IRDA;
        /* 我们需要保存 IrDA 解码的数据包方向，
         * 所以让我们使用 "Linux-cooked" 模式。Jean II
         *
         * XXX - 这在 setup_socket() 中处理。 */
        /* handlep->cooked = 1; */
        break;

    /* ARPHRD_LAPD 是非官方的，随机分配的，如果需要重新分配，请报告给 <daniele@orlandi.com> */
#ifndef ARPHRD_LAPD
#define ARPHRD_LAPD    8445
#endif
    case ARPHRD_LAPD:
        /* 不期望从这些接口中得到 IP 数据包... */
        handle->linktype = DLT_LINUX_LAPD;
        break;

#ifndef ARPHRD_NONE
#define ARPHRD_NONE    0xFFFE
#endif
    case ARPHRD_NONE:
        /*
         * 没有链路层头部；数据包只是 IP 数据包，所以使用 DLT_RAW。
         */
        handle->linktype = DLT_RAW;
        break;

#ifndef ARPHRD_IEEE802154
#define ARPHRD_IEEE802154      804
#endif
       case ARPHRD_IEEE802154:
               handle->linktype =  DLT_IEEE802_15_4_NOFCS;
               break;

#ifndef ARPHRD_NETLINK
#define ARPHRD_NETLINK    824
#endif
    case ARPHRD_NETLINK:
        handle->linktype = DLT_NETLINK;
        /*
         * 我们需要使用 cooked 模式，这样在 sll_protocol 中
         * 我们可以获取 netlink 协议类型，比如 NETLINK_ROUTE,
         * NETLINK_GENERIC, NETLINK_FIB_LOOKUP 等。
         *
         * XXX - 这在 setup_socket() 中处理。
         */
        /* handlep->cooked = 1; */
        break;

#ifndef ARPHRD_VSOCKMON
#define ARPHRD_VSOCKMON    826
#endif
    case ARPHRD_VSOCKMON:
        handle->linktype = DLT_VSOCK;
        break;

    default:
        handle->linktype = -1;
        break;
    }
}

static void
set_dlt_list_cooked(pcap_t *handle)
    # 为了支持 DLT_LINUX_SLL 和 DLT_LINUX_SLL2，分配一个包含两个元素的整数数组内存空间
    handle->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);

    # 如果内存分配失败，将列表保持为空
    if (handle->dlt_list != NULL) {
        # 将 DLT_LINUX_SLL 和 DLT_LINUX_SLL2 分别赋值给数组的两个元素
        handle->dlt_list[0] = DLT_LINUX_SLL;
        handle->dlt_list[1] = DLT_LINUX_SLL2;
        # 设置数组的元素个数为 2
        handle->dlt_count = 2;
    }
}
/*
 * 尝试设置一个 PF_PACKET 套接字。
 * 成功返回 0，失败返回 PCAP_ERROR_ 值。
 */
static int
setup_socket(pcap_t *handle, int is_any_device)
{
    // 获取 pcap_linux 结构体指针
    struct pcap_linux *handlep = handle->priv;
    // 获取设备名称
    const char        *device = handle->opt.device;
    int            status = 0;
    int            sock_fd, arptype;
    int            val;
    int            err = 0;
    struct packet_mreq    mr;
#if defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT)
    int            bpf_extensions;
    socklen_t        len = sizeof(bpf_extensions);
#endif

    /*
     * 使用协议族 packet 打开一个套接字。如果 is_any_device 为真，
     * 则为 cooked 接口打开一个 SOCK_DGRAM 套接字，否则为 raw 接口打开一个 SOCK_RAW 套接字。
     *
     * 协议设置为 0。这意味着在绑定套接字之前，我们将不会接收任何数据包。
     * 这允许我们设置环形缓冲区而不丢失任何数据包。
     */
    sock_fd = is_any_device ?
        socket(PF_PACKET, SOCK_DGRAM, 0) :
        socket(PF_PACKET, SOCK_RAW, 0);

    if (sock_fd == -1) {
        if (errno == EPERM || errno == EACCES) {
            /*
             * 没有权限打开套接字。
             */
            status = PCAP_ERROR_PERM_DENIED;
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to create packet socket failed - CAP_NET_RAW may be required");
        } else {
            /*
             * 其他错误。
             */
            status = PCAP_ERROR;
        }
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "socket");
        return status;
    }
}
    /*
     * 获取回环设备的接口索引。
     * 如果尝试失败，不要报错，只需将 "handlep->lo_ifindex" 设置为 -1。
     *
     * XXX - 是否可能有多个设备将数据包回送，即除了 "lo" 以外的设备？如果是这样，
     * 我们需要找到它们所有，并为它们创建一个索引数组，在 "pcap_read_packet()" 中检查所有这些索引。
     */
    handlep->lo_ifindex = iface_get_id(sock_fd, "lo", handle->errbuf);

    /*
     * 将链路层有效载荷对齐到4字节边界的默认偏移量。
     */
    handle->offset     = 0;

    /*
     * 我们需要处理什么类型的帧？如果我们有一个未知的接口类型或者我们知道在原始模式下工作不好的类型，
     * 则回退到 cooked 模式。
     */
    } else {
        /*
         * "any" 设备。
         */
        if (handle->opt.rfmon) {
            /*
             * 它不支持监控模式。
             */
            close(sock_fd);
            return PCAP_ERROR_RFMON_NOTSUP;
        }

        /*
         * 它使用 cooked 模式。
         */
        handlep->cooked = 1;
        handle->linktype = DLT_LINUX_SLL;
        handle->dlt_list = NULL;
        handle->dlt_count = 0;
        set_dlt_list_cooked(handle);

        /*
         * 我们没有绑定到任何设备。
         * 目前，我们将其用作无法传输的指示；只有在我们弄清楚如何在 cooked 模式下传输时才停止这样做。
         */
        handlep->ifindex = -1;
    }
    /*
     * 如果设置了 "promisc"，则选择混杂模式。
     * 如果我们不选择混杂模式，则不要打开 allmulti 模式 - 在某些设备上（例如 Orinoco 无线接口），不支持 allmulti 模式，
     * 并且驱动程序通过打开混杂模式来实现它，这会破坏卡作为正常网络接口的操作，在我知道的任何其他平台上，启动非混杂捕获不会影响接口接收的多播数据包。
     */

    /*
     * 嗯，我们如何在所有接口上设置混杂模式？
     * 我不确定这是否可能。目前，我们悄悄地忽略尝试为 "any" 设备打开混杂模式的尝试（因此您不必在诸如 tcpdump 等程序中显式禁用它）。
     */

    if (!is_any_device && handle->opt.promisc) {
        // 初始化 mr 结构体
        memset(&mr, 0, sizeof(mr));
        // 设置 mr 结构体的成员
        mr.mr_ifindex = handlep->ifindex;
        mr.mr_type    = PACKET_MR_PROMISC;
        // 设置套接字选项，将接口加入多播组
        if (setsockopt(sock_fd, SOL_PACKET, PACKET_ADD_MEMBERSHIP,
            &mr, sizeof(mr)) == -1) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno, "setsockopt (PACKET_ADD_MEMBERSHIP)");
            // 关闭套接字
            close(sock_fd);
            // 返回错误
            return PCAP_ERROR;
        }
    }

    /*
     * 启用辅助数据并保留用于重建 VLAN 标头的空间。
     *
     * XXX - 现在我们只支持内存映射捕获，启用辅助数据是否必要？内核的内存映射捕获代码似乎不检查辅助数据是否已启用，似乎无论是否提供辅助数据，它都会提供。
     */
    // 设置 val 为 1
    val = 1;
    # 如果设置套接字选项失败，并且错误码不是 ENOPROTOOPT，则格式化错误消息，关闭套接字并返回错误
    if (setsockopt(sock_fd, SOL_PACKET, PACKET_AUXDATA, &val,
               sizeof(val)) == -1 && errno != ENOPROTOOPT) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "setsockopt (PACKET_AUXDATA)");
        close(sock_fd);
        return PCAP_ERROR;
    }
    # 偏移量增加 VLAN 标签的长度
    handle->offset += VLAN_TAG_LEN;

    """
     如果处于 cooked 模式，则使快照长度足够大，以容纳“cooked mode”头部加上 1 字节的数据（这样我们就不会向“recvfrom()”传递字节计数为 0）。
     XXX - 我们不知道这将是 DLT_LINUX_SLL 还是 DLT_LINUX_SLL2，因此确保它足够大以容纳 DLT_LINUX_SLL2 的“cooked mode”头部；那么小的快照长度也是愚蠢的。
    """
    if (handlep->cooked) {
        if (handle->snapshot < SLL2_HDR_LEN + 1)
            handle->snapshot = SLL2_HDR_LEN + 1;
    }
    # 设置缓冲区大小为快照长度
    handle->bufsize = handle->snapshot;

    # 设置插入 VLAN 标签的偏移量
    set_vlan_offset(handle);

    # 如果时间戳精度为纳秒，则设置套接字选项为 SO_TIMESTAMPNS
    if (handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO) {
        int nsec_tstamps = 1;

        if (setsockopt(sock_fd, SOL_SOCKET, SO_TIMESTAMPNS, &nsec_tstamps, sizeof(nsec_tstamps)) < 0) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "setsockopt: unable to set SO_TIMESTAMPNS");
            close(sock_fd);
            return PCAP_ERROR;
        }
    }

    # 成功后，将套接字文件描述符保存在 pcap 结构中
    handle->fd = sock_fd;
#if defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT)
    /*
     * 检查是否可以生成用于 VLAN 检查的特殊代码
     * （XXX - 如果我们需要特殊代码但操作系统不支持怎么办？这种情况可能吗？）
     */
    if (getsockopt(sock_fd, SOL_SOCKET, SO_BPF_EXTENSIONS,
        &bpf_extensions, &len) == 0) {
        if (bpf_extensions >= SKF_AD_VLAN_TAG_PRESENT) {
            /*
             * 是的，我们可以。请求执行特殊处理。
             */
            handle->bpf_codegen_flags |= BPF_SPECIAL_VLAN_HANDLING;
        }
    }
#endif /* defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT) */

    return status;
}

/*
 * 尝试设置内存映射访问。
 *
 * 成功时返回 1，并将 *status 设置为 0（如果没有警告）或设置为 PCAP_WARNING_ 代码（如果有警告）。
 *
 * 出错时返回 -1，并将 *status 设置为适当的错误代码；如果是 PCAP_ERROR，则将 handle->errbuf 设置为适当的消息。
 */
static int
setup_mmapped(pcap_t *handle, int *status)
{
    struct pcap_linux *handlep = handle->priv;
    int ret;

    /*
     * 尝试分配一个缓冲区来保存一个数据包的内容，供 oneshot 回调使用。
     */
    handlep->oneshot_buffer = malloc(handle->snapshot);
    if (handlep->oneshot_buffer == NULL) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't allocate oneshot buffer");
        *status = PCAP_ERROR;
        return -1;
    }

    if (handle->opt.buffer_size == 0) {
        /* 默认情况下请求 2M 的环形缓冲区 */
        handle->opt.buffer_size = 2*1024*1024;
    }
    ret = prepare_tpacket_socket(handle);
    if (ret == -1) {
        free(handlep->oneshot_buffer);
        handlep->oneshot_buffer = NULL;
        *status = PCAP_ERROR;
        return ret;
    }
    ret = create_ring(handle, status);
    if (ret == -1) {
        /*
         * 如果 ret 等于 -1，表示尝试启用内存映射捕获时出错；
         * 失败。create_ring() 已经设置了 *status。
         */
        // 释放内存并将指针置为空
        free(handlep->oneshot_buffer);
        handlep->oneshot_buffer = NULL;
        // 返回 -1 表示失败
        return -1;
    }

    /*
     * 成功。*status 已经被设置为 0（如果没有警告）或者一个 PCAP_WARNING_ 值（如果有警告）。
     *
     * handle->offset 用于获取当前位置到 rx 环中。
     * handle->cc 用于存储环的大小。
     */

    /*
     * 在返回之前设置在 poll() 中使用的超时时间。
     */
    // 设置 poll() 使用的超时时间
    set_poll_timeout(handlep);

    // 返回 1 表示成功
    return 1;
/*
 * 尝试将套接字设置为指定版本的内存映射头部。
 *
 * 如果成功，返回 0；如果因为不支持该版本而失败，返回 1；如果有其他错误，返回 -1，并设置 handle->errbuf。
 */
static int
init_tpacket(pcap_t *handle, int version, const char *version_str)
{
    // 获取 pcap_linux 结构体指针
    struct pcap_linux *handlep = handle->priv;
    // 设置 val 为指定版本
    int val = version;
    // 设置 len 为 val 的大小
    socklen_t len = sizeof(val);

    /*
     * 探测内核是否支持指定的 TPACKET 版本；
     * 这也会获取该版本的头部长度。
     *
     * 此套接字选项是在 2.6.27 中引入的，也是第一个支持 TPACKET_V2 的版本。
     */
    if (getsockopt(handle->fd, SOL_PACKET, PACKET_HDRLEN, &val, &len) < 0) {
        if (errno == EINVAL) {
            /*
             * EINVAL 表示不支持这个特定版本的 TPACKET。
             * 告诉调用者他们可以尝试其他版本；如果他们已经尝试完所有版本，让他们适当设置错误消息。
             */
            return 1;
        }

        /*
         * 所有其他错误都是致命的。
         */
        if (errno == ENOPROTOOPT) {
            /*
             * 不支持 PACKET_HDRLEN，这意味着不支持内存映射捕获。
             * 在消息中指示这一点。
             */
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Kernel doesn't support memory-mapped capture; a 2.6.27 or later 2.x kernel is required, with CONFIG_PACKET_MMAP specified for 2.x kernels");
        } else {
            /*
             * 一些意外的错误。
             */
            pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
                errno, "can't get %s header len on packet socket",
                version_str);
        }
        return -1;
    }
    // 设置 handlep->tp_hdrlen 为 val
    handlep->tp_hdrlen = val;

    // 设置 val 为 version
    val = version;
}
    # 如果设置套接字选项失败
    if (setsockopt(handle->fd, SOL_PACKET, PACKET_VERSION, &val,
               sizeof(val)) < 0) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't activate %s on packet socket", version_str);
        # 返回-1表示失败
        return -1;
    }
    # 设置处理程序的tp_version为版本号
    handlep->tp_version = version;
    
    # 返回0表示成功
    return 0;
/*
 * 尝试将套接字设置为内存映射头部的第3版，
 * 如果失败是因为不支持第3版，则尝试回退到第2版。
 * 如果不支持第2版，则失败。
 *
 * 如果成功返回0，其他错误返回-1，并设置handle->errbuf。
 */
static int
prepare_tpacket_socket(pcap_t *handle)
{
    int ret;

#ifdef HAVE_TPACKET3
    /*
     * 尝试将版本设置为TPACKET_V3。
     *
     * 在PF_PACKET套接字上进行缓冲的唯一模式是TPACKET_V3模式，
     * 因此可能不会立即传递数据包。
     *
     * 在该模式下无法禁用缓冲，因此如果用户请求立即模式，我们不使用TPACKET_V3。
     */
    if (!handle->opt.immediate) {
        ret = init_tpacket(handle, TPACKET_V3, "TPACKET_V3");
        if (ret == 0) {
            /*
             * 成功。
             */
            return 0;
        }
        if (ret == -1) {
            /*
             * 由于某些原因而失败，而不是“内核不支持TPACKET_V3”。
             */
            return -1;
        }

        /*
         * 这意味着它返回1，表示“内核不支持TPACKET_V3”；尝试TPACKET_V2。
         */
    }
#endif /* HAVE_TPACKET3 */

    /*
     * 尝试将版本设置为TPACKET_V2。
     */
    ret = init_tpacket(handle, TPACKET_V2, "TPACKET_V2");
    if (ret == 0) {
        /*
         * 成功。
         */
        return 0;
    }

    if (ret == 1) {
        /*
         * 好的，内核支持内存映射捕获，但不支持TPACKET_V2。适当设置错误消息。
         */
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "内核不支持TPACKET_V2；需要2.6.27或更高版本的内核");
    }

    /*
     * 失败。
     */
    return -1;
}

#define MAX(a,b) ((a)>(b)?(a):(b))
/*
 * 尝试设置内存映射访问。
 *
 * 成功时返回1，并且如果没有警告则将*status设置为0，如果有警告则设置为PCAP_WARNING_代码。
 *
 * 出错时返回-1，并将*status设置为适当的错误代码；如果是PCAP_ERROR，则将handle->errbuf设置为适当的消息。
 */
static int
create_ring(pcap_t *handle, int *status)
{
    struct pcap_linux *handlep = handle->priv;
    unsigned i, j, frames_per_block;
#ifdef HAVE_TPACKET3
    /*
     * 对于使用TPACKET_V2的套接字，struct tpacket_req3结构的末尾的额外内容将被忽略，因此即使对于这些套接字也是可以的。
     */
    struct tpacket_req3 req;
#else
    struct tpacket_req req;
#endif
    socklen_t len;
    unsigned int sk_type, tp_reserve, maclen, tp_hdrlen, netoff, macoff;
    unsigned int frame_size;

    /*
     * 开始假设没有警告或错误。
     */
    *status = 0;

    /*
     * 为VLAN标记重建保留空间。
     */
    tp_reserve = VLAN_TAG_LEN;

    /*
     * 如果我们在cooked模式下捕获，为DLT_LINUX_SLL2头部保留空间；我们还不知道我们将使用DLT_LINUX_SLL还是DLT_LINUX_SLL2，因为这可以在打开设备时更改，所以我们为两者中较大的一个保留空间。
     *
     * XXX - 我们假设内核仍然添加16字节的额外空间，所以我们从SLL2_HDR_LEN中减去16来得到额外所需的空间。（他们对DLT_LINUX_SLL是否也这样做，其链路层头部为16字节？）
     *
     * XXX - 我们应该使用TPACKET_ALIGN(SLL2_HDR_LEN - 16)吗？
     */
    if (handlep->cooked)
        tp_reserve += SLL2_HDR_LEN - 16;

    /*
     * 尝试请求该保留空间的数量。
     * 这必须在创建环形缓冲区之前完成。
     */
    len = sizeof(tp_reserve);
    # 如果设置套接字选项失败，则执行以下代码块
    if (setsockopt(handle->fd, SOL_PACKET, PACKET_RESERVE,
        &tp_reserve, len) < 0) {
        # 格式化错误消息，将错误信息存储到 handle->errbuf 中
        pcap_fmt_errmsg_for_errno(handle->errbuf,
            PCAP_ERRBUF_SIZE, errno,
            "setsockopt (PACKET_RESERVE)");
        # 设置状态为错误
        *status = PCAP_ERROR;
        # 返回 -1，表示设置套接字选项失败
        return -1;
    }

    # 根据 handlep->tp_version 的值执行不同的操作
    switch (handlep->tp_version) {
#ifdef HAVE_TPACKET3
    // 如果定义了 HAVE_TPACKET3，则执行以下代码块
    case TPACKET_V3:
        /* The "frames" for this are actually buffers that
         * contain multiple variable-sized frames.
         *
         * We pick a "frame" size of MAXIMUM_SNAPLEN to leave
         * enough room for at least one reasonably-sized packet
         * in the "frame". */
        // 设置请求的帧大小为 MAXIMUM_SNAPLEN，以确保每个帧至少能容纳一个合理大小的数据包
        req.tp_frame_size = MAXIMUM_SNAPLEN;
        /*
         * Round the buffer size up to a multiple of the
         * "frame" size (rather than rounding down, which
         * would give a buffer smaller than our caller asked
         * for, and possibly give zero "frames" if the requested
         * buffer size is too small for one "frame").
         */
        // 将缓冲区大小向上舍入为帧大小的倍数（而不是向下舍入），以确保缓冲区大小不小于调用者请求的大小，并且如果请求的缓冲区大小对于一个帧来说太小，则可能会得到零个“帧”。
        req.tp_frame_nr = (handle->opt.buffer_size + req.tp_frame_size - 1)/req.tp_frame_size;
        break;
#endif
    default:
        // 如果不是 TPACKET_V3，则输出错误信息到错误缓冲区，设置状态为错误，并返回 -1
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "Internal error: unknown TPACKET_ value %u",
            handlep->tp_version);
        *status = PCAP_ERROR;
        return -1;
    }

    /* compute the minimum block size that will handle this frame.
     * The block has to be page size aligned.
     * The max block size allowed by the kernel is arch-dependent and
     * it's not explicitly checked here. */
    // 计算能够处理该帧的最小块大小。块大小必须与页面大小对齐。
    req.tp_block_size = getpagesize();
    while (req.tp_block_size < req.tp_frame_size)
        req.tp_block_size <<= 1;

    frames_per_block = req.tp_block_size/req.tp_frame_size;

    /*
     * PACKET_TIMESTAMP was added after linux/net_tstamp.h was,
     * so we check for PACKET_TIMESTAMP.  We check for
     * linux/net_tstamp.h just in case a system somehow has
     * PACKET_TIMESTAMP but not linux/net_tstamp.h; that might
     * be unnecessary.
     *
     * SIOCSHWTSTAMP was introduced in the patch that introduced
     * linux/net_tstamp.h, so we don't bother checking whether
     * SIOCSHWTSTAMP is defined (if your Linux system has
     * linux/net_tstamp.h but doesn't define SIOCSHWTSTAMP, your
     * Linux system is badly broken).
     */
    // 检查是否定义了 PACKET_TIMESTAMP 和 linux/net_tstamp.h，以确保系统支持时间戳
#if defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP)
    /*
     * 如果定义了 HAVE_LINUX_NET_TSTAMP_H 和 PACKET_TIMESTAMP，就向内核和驱动程序请求使用硬件时间戳。
     *
     * 硬件时间戳仅在使用内存映射捕获时支持。
     */
    }
#endif /* HAVE_LINUX_NET_TSTAMP_H && PACKET_TIMESTAMP */

    /* 请求内核创建环形缓冲区 */
retry:
    req.tp_block_nr = req.tp_frame_nr / frames_per_block;

    /* 请求 req.tp_frame_nr 与 frames_per_block*req.tp_block_nr 匹配 */
    req.tp_frame_nr = req.tp_block_nr * frames_per_block;

#ifdef HAVE_TPACKET3
    /* 超时值用于撤销块 - 使用配置的缓冲超时，如果小于0，则使用默认值。 */
    if (handlep->timeout > 0) {
        /* 使用用户指定的超时作为块超时 */
        req.tp_retire_blk_tov = handlep->timeout;
    } else if (handlep->timeout == 0) {
        /*
         * 在 pcap 中，这意味着“无限超时”；TPACKET_V3 不支持这一点，因此将其设置为 UINT_MAX 毫秒。
         * 在 TPACKET_V3 循环中，如果超时为 0，并且我们还没有收到任何数据包，并且我们阻塞了但仍然没有收到任何数据包，我们将一直阻塞直到收到数据包。
         */
        req.tp_retire_blk_tov = UINT_MAX;
    } else {
        /*
         * XXX - 这是无效的；暂时使用 0，表示“由内核选择默认值”。
         */
        req.tp_retire_blk_tov = 0;
    }
    /* 未使用私有数据 */
    req.tp_sizeof_priv = 0;
    /* Rx 环 - 特性请求位 - 无（rxhash 将不会被填充） */
    req.tp_feature_req_word = 0;
#endif
    # 设置套接字选项，用于接收数据包
    if (setsockopt(handle->fd, SOL_PACKET, PACKET_RX_RING,
                    (void *) &req, sizeof(req))) {
        # 如果内存不足，并且请求的环大小大于1，则尝试减小请求的环大小
        if ((errno == ENOMEM) && (req.tp_block_nr > 1)) {
            '''
            内存失败；尝试减小请求的环大小。
            
            以前我们是减半，现在改为减少5%。
            这可能会导致更多的迭代和更长的启动时间，但用户会对结果的缓冲区大小更满意。
            '''
            if (req.tp_frame_nr < 20)
                req.tp_frame_nr -= 1;
            else
                req.tp_frame_nr -= req.tp_frame_nr/20;
            # 重新尝试设置套接字选项
            goto retry;
        }
        # 格式化错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't create rx ring on packet socket");
        *status = PCAP_ERROR;
        return -1;
    }

    # 内存映射接收环
    handlep->mmapbuflen = req.tp_block_nr * req.tp_block_size;
    handlep->mmapbuf = mmap(0, handlep->mmapbuflen,
        PROT_READ|PROT_WRITE, MAP_SHARED, handle->fd, 0);
    if (handlep->mmapbuf == MAP_FAILED) {
        # 格式化错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't mmap rx ring");

        # 在错误时清除分配的环
        destroy_ring(handle);
        *status = PCAP_ERROR;
        return -1;
    }

    # 为每个帧头指针分配一个环
    handle->cc = req.tp_frame_nr;
    handle->buffer = malloc(handle->cc * sizeof(union thdr *));
    if (!handle->buffer) {
        # 格式化错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't allocate ring of frame headers");

        destroy_ring(handle);
        *status = PCAP_ERROR;
        return -1;
    }

    # 用适当的帧指针填充头环
    handle->offset = 0;
    # 遍历每个数据块
    for (i=0; i<req.tp_block_nr; ++i) {
        # 计算当前数据块的起始地址
        u_char *base = &handlep->mmapbuf[i*req.tp_block_size];
        # 遍历当前数据块中的每个帧
        for (j=0; j<frames_per_block; ++j, ++handle->offset) {
            # 将当前帧的地址存入环形缓冲区
            RING_GET_CURRENT_FRAME(handle) = base;
            # 更新下一个帧的起始地址
            base += req.tp_frame_size;
        }
    }

    # 设置缓冲区大小为帧大小
    handle->bufsize = req.tp_frame_size;
    # 重置偏移量
    handle->offset = 0;
    # 返回成功
    return 1;
/* 释放所有与环相关的资源 */
static void
destroy_ring(pcap_t *handle)
{
    struct pcap_linux *handlep = handle->priv;

    /*
     * 告诉内核销毁环。
     * 我们不检查 setsockopt 的失败，因为 1）我们无法从错误中恢复，2）我们可能还没有设置它。
     */
    struct tpacket_req req;
    memset(&req, 0, sizeof(req));
    (void)setsockopt(handle->fd, SOL_PACKET, PACKET_RX_RING,
                (void *) &req, sizeof(req));

    /* 如果环被映射，取消映射 */
    if (handlep->mmapbuf) {
        /* 不测试 mmap 失败，因为我们无法从任何错误中恢复 */
        (void)munmap(handlep->mmapbuf, handlep->mmapbuflen);
        handlep->mmapbuf = NULL;
    }
}

/*
 * 特殊的一次性回调，用于 Linux mmapped 捕获。
 *
 * 问题在于 pcap_next() 和 pcap_next_ex() 期望回调传递的数据在回调返回后仍然有效，
 * 但是 pcap_read_linux_mmap() 必须在回调返回后立即释放该数据包
 * （否则，内核会认为环中仍然有至少一个未处理的数据包可用，因此 select() 将立即返回指示有数据可处理），
 * 因此，在回调中，我们必须对数据包进行复制。
 *
 * 是的，这意味着，如果捕获使用环缓冲区，使用 pcap_next() 或 pcap_next_ex() 需要比使用 pcap_loop() 或 pcap_dispatch() 多进行更多的复制。
 * 如果这让你烦恼，就不要使用 pcap_next() 或 pcap_next_ex()。
 */
static void
pcap_oneshot_linux(u_char *user, const struct pcap_pkthdr *h,
    const u_char *bytes)
{
    struct oneshot_userdata *sp = (struct oneshot_userdata *)user;
    pcap_t *handle = sp->pd;
    struct pcap_linux *handlep = handle->priv;

    *sp->hdr = *h;
    memcpy(handlep->oneshot_buffer, bytes, h->caplen);
    *sp->pkt = handlep->oneshot_buffer;
}

static int
# 获取 Linux 平台下 pcap_t 结构体中的非阻塞状态
pcap_getnonblock_linux(pcap_t *handle)
{
    # 获取 pcap_t 结构体中的私有数据结构 pcap_linux
    struct pcap_linux *handlep = handle->priv;

    /* 使用负值的超时时间表示非阻塞操作 */
    return (handlep->timeout<0);
}

# 设置 Linux 平台下 pcap_t 结构体中的非阻塞状态
static int
pcap_setnonblock_linux(pcap_t *handle, int nonblock)
{
    # 获取 pcap_t 结构体中的私有数据结构 pcap_linux
    struct pcap_linux *handlep = handle->priv;

    '''
     * 将文件描述符设置为非阻塞模式，因为我们将用它来发送数据包。
     '''
    if (pcap_setnonblock_fd(handle, nonblock) == -1)
        return -1;

    '''
     * 将每个值映射到它们对应的否定值，以保留使用 pcap_set_timeout 提供的超时值。
     '''
    if (nonblock) {
        if (handlep->timeout >= 0) {
            '''
             * 表示我们正在切换到非阻塞模式。
             '''
            handlep->timeout = ~handlep->timeout;
        }
        if (handlep->poll_breakloop_fd != -1) {
            ''' 
            关闭 eventfd；在非阻塞模式下我们不需要它。
            '''
            close(handlep->poll_breakloop_fd);
            handlep->poll_breakloop_fd = -1;
        }
    } else {
        if (handlep->poll_breakloop_fd == -1) {
            ''' 
            如果我们没有 eventfd，则在切换到阻塞模式时现在打开一个。
            '''
            if ( ( handlep->poll_breakloop_fd = eventfd(0, EFD_NONBLOCK) ) == -1 ) {
                int save_errno = errno;
                snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                        "Could not open eventfd: %s",
                        strerror(errno));
                errno = save_errno;
                return -1;
            }
        }
        if (handlep->timeout < 0) {
            handlep->timeout = ~handlep->timeout;
        }
    }
    ''' 
    更新用于 poll() 的超时时间。
    '''
    set_poll_timeout(handlep);
    return 0;
}

'''
 * 获取指定偏移量处环形缓冲区帧的状态字段。
 '''
static inline u_int
pcap_get_ring_frame_status(pcap_t *handle, int offset)
{
    # 获取 pcap_t 结构体中的私有数据结构 pcap_linux
    struct pcap_linux *handlep = handle->priv;
    # 定义一个名为 h 的联合体变量，用于存储数据包头部信息
    union thdr h;
    
    # 从指定偏移处获取数据包的帧，并将其存储到 h.raw 中
    h.raw = RING_GET_FRAME_AT(handle, offset);
    
    # 根据 handlep 指向的数据包处理器的版本进行判断
    switch (handlep->tp_version) {
        # 如果版本为 TPACKET_V2，则执行以下代码
        case TPACKET_V2:
            # 使用原子操作加载 h.h2->tp_status 的值，并以原子方式获取，返回结果
            return __atomic_load_n(&h.h2->tp_status, __ATOMIC_ACQUIRE);
            # 结束 case
            break;
#ifdef HAVE_TPACKET3
    // 如果定义了 HAVE_TPACKET3，则执行以下代码
    case TPACKET_V3:
        // 返回 h.h3->hdr.bh1.block_status 的原子加载值，使用原子操作保证读取的一致性
        return __atomic_load_n(&h.h3->hdr.bh1.block_status, __ATOMIC_ACQUIRE);
        // 结束 case 语句
        break;
#endif
    // 结束 switch 语句
    }
    // 如果执行到这里，说明出现了意外情况，返回 0
    /* This should not happen. */
    return 0;
}

/*
 * Block waiting for frames to be available.
 */
// 等待帧可用的阻塞函数
static int pcap_wait_for_frames_mmap(pcap_t *handle)
{
    // 获取 pcap_linux 结构体指针
    struct pcap_linux *handlep = handle->priv;
    int timeout;
    struct ifreq ifr;
    int ret;
    struct pollfd pollinfo[2];
    int numpollinfo;
    // 设置 pollinfo[0] 的文件描述符和事件
    pollinfo[0].fd = handle->fd;
    pollinfo[0].events = POLLIN;
    // 如果没有设置 poll_breakloop_fd
    if ( handlep->poll_breakloop_fd == -1 ) {
        numpollinfo = 1;
        pollinfo[1].revents = 0;
        /*
         * We set pollinfo[1].revents to zero, even though
         * numpollinfo = 1 meaning that poll() doesn't see
         * pollinfo[1], so that we do not have to add a
         * conditional of numpollinfo > 1 below when we
         * test pollinfo[1].revents.
         */
    } else {
        // 设置 pollinfo[1] 的文件描述符和事件
        pollinfo[1].fd = handlep->poll_breakloop_fd;
        pollinfo[1].events = POLLIN;
        numpollinfo = 2;
    }

    }
    // 返回 0
    return 0;
}

/* handle a single memory mapped packet */
// 处理单个内存映射数据包
static int pcap_handle_packet_mmap(
        pcap_t *handle,
        pcap_handler callback,
        u_char *user,
        unsigned char *frame,
        unsigned int tp_len,
        unsigned int tp_mac,
        unsigned int tp_snaplen,
        unsigned int tp_sec,
        unsigned int tp_usec,
        int tp_vlan_tci_valid,
        __u16 tp_vlan_tci,
        __u16 tp_vlan_tpid)
{
    // 获取 pcap_linux 结构体指针
    struct pcap_linux *handlep = handle->priv;
    unsigned char *bp;
    struct sockaddr_ll *sll;
    struct pcap_pkthdr pcaphdr;
    pcap_can_socketcan_hdr *canhdr;
    unsigned int snaplen = tp_snaplen;
    struct utsname utsname;

    /* perform sanity check on internal offset. */
    // 对内部偏移进行合理性检查
    # 如果数据包的偏移量加上数据包捕获长度大于缓冲区大小，则表示数据包损坏
    if (tp_mac + tp_snaplen > handle->bufsize) {
        /*
         * 作为调试辅助，报告一些系统信息。
         */
        # 如果能够获取系统信息，则将系统信息添加到错误信息中
        if (uname(&utsname) != -1) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "corrupted frame on kernel ring mac "
                "offset %u + caplen %u > frame len %d "
                "(kernel %.32s version %s, machine %.16s)",
                tp_mac, tp_snaplen, handle->bufsize,
                utsname.release, utsname.version,
                utsname.machine);
        } else {
            # 否则只添加基本的错误信息
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "corrupted frame on kernel ring mac "
                "offset %u + caplen %u > frame len %d",
                tp_mac, tp_snaplen, handle->bufsize);
        }
        # 返回错误代码
        return -1;
    }

    /* 运行过滤器来过滤接收到的数据包
     * 如果内核过滤器已启用，我们需要运行过滤器直到在创建过滤器时存在于环形缓冲区中的所有帧都被处理。
     * 在这种情况下，blocks_to_filter_in_userland 用作我们需要过滤的数据包的计数器。
     * 注意：另一种可能的方法是在环形缓冲区变为空时停止应用过滤器，但这可能会发生得更晚... */
    bp = frame + tp_mac;

    /* 如果需要，在原地构建 sll 头部 */
    sll = (void *)(frame + TPACKET_ALIGN(handlep->tp_hdrlen));

    # 如果用户态过滤器已启用并且存在过滤器指令，则执行过滤器
    if (handlep->filter_in_userland && handle->fcode.bf_insns) {
        struct pcap_bpf_aux_data aux_data;

        aux_data.vlan_tag_present = tp_vlan_tci_valid;
        aux_data.vlan_tag = tp_vlan_tci & 0x0fff;

        # 如果过滤器通过，则返回 0
        if (pcap_filter_with_aux_data(handle->fcode.bf_insns,
                          bp,
                          tp_len,
                          snaplen,
                          &aux_data) == 0)
            return 0;
    }

    # 如果数据包方向不符合要求，则返回 0
    if (!linux_check_direction(handle, sll))
        return 0;

    /* 从环形缓冲区头部获取所需的数据包信息 */
    # 设置 pcap 包头的时间戳秒部分
    pcaphdr.ts.tv_sec = tp_sec;
    # 设置 pcap 包头的时间戳微秒部分
    pcaphdr.ts.tv_usec = tp_usec;
    # 设置 pcap 包头的捕获长度
    pcaphdr.caplen = tp_snaplen;
    # 设置 pcap 包头的实际长度
    pcaphdr.len = tp_len;

    # 如果需要，在现有位置构建 sll 头部
    if (handlep->cooked) {
        # 更新数据包长度
        if (handle->linktype == DLT_LINUX_SLL2) {
            pcaphdr.caplen += SLL2_HDR_LEN;
            pcaphdr.len += SLL2_HDR_LEN;
        } else {
            pcaphdr.caplen += SLL_HDR_LEN;
            pcaphdr.len += SLL_HDR_LEN;
        }
    }

    # 如果 VLAN 标签有效，并且 VLAN 偏移不为 -1，并且捕获长度大于等于 VLAN 偏移
    if (tp_vlan_tci_valid &&
        handlep->vlan_offset != -1 &&
        tp_snaplen >= (unsigned int) handlep->vlan_offset)
    {
        # 定义 VLAN 标签结构体指针
        struct vlan_tag *tag;

        # 将头部中除了类型字段之外的所有内容向下移动 VLAN_TAG_LEN 字节，以便在这些内容和类型字段之间插入 VLAN 标签
        bp -= VLAN_TAG_LEN;
        memmove(bp, bp + VLAN_TAG_LEN, handlep->vlan_offset);

        # 现在插入标签
        tag = (struct vlan_tag *)(bp + handlep->vlan_offset);
        tag->vlan_tpid = htons(tp_vlan_tpid);
        tag->vlan_tci = htons(tp_vlan_tci);

        # 将标签长度添加到数据包长度中
        pcaphdr.caplen += VLAN_TAG_LEN;
        pcaphdr.len += VLAN_TAG_LEN;
    }

    # 唯一的方法是通过过滤程序告诉内核在快照长度处截断数据包；如果没有过滤程序，内核将不会截断数据包。
    # 将快照长度修剪为不超过指定的快照长度。
    # XXX - 另一种方法是在激活例程中在套接字上放置一个过滤器，包括一个 "ret <snaplen>" 指令，这样即使没有人指定过滤器，内核也会在内核中进行截断；这意味着在内存映射缓冲区中消耗的缓冲区空间更少。
    # 如果捕获的数据包长度大于快照长度，则将捕获的数据包长度设置为快照长度
    if (pcaphdr.caplen > (bpf_u_int32)handle->snapshot)
        pcaphdr.caplen = handle->snapshot;

    # 将捕获的数据包传递给用户回调函数
    callback(user, &pcaphdr, bp);

    # 返回 1，表示成功处理数据包
    return 1;
    # 获取 pcap_t 结构体中的私有数据结构 pcap_linux
    struct pcap_linux *handlep = handle->priv;
    # 定义一个联合体 thdr，用于存储数据包头部信息
    union thdr h;
    # 初始化数据包计数器
    int pkts = 0;
    # 定义返回值变量
    int ret;

    # 等待数据帧可用
    h.raw = RING_GET_CURRENT_FRAME(handle);
    # 如果无法获取数据包，则等待内核分配数据包
    if (!packet_mmap_acquire(h.h2)) {
        ret = pcap_wait_for_frames_mmap(handle);
        # 如果等待数据包失败，则返回错误
        if (ret) {
            return ret;
        }
    }

    # 如果数据包计数不受限制，则将其限制在 INT_MAX 以内
    if (PACKET_COUNT_IS_UNLIMITED(max_packets))
        max_packets = INT_MAX;
    # 当已捕获数据包数量小于最大数量时，执行循环
    while (pkts < max_packets) {
        '''
         * 获取当前环形缓冲区帧，并在仍由内核拥有时中断
         '''
        h.raw = RING_GET_CURRENT_FRAME(handle);
        # 如果无法获取数据包，则中断循环
        if (!packet_mmap_acquire(h.h2))
            break;

        # 调用 pcap_handle_packet_mmap 处理数据包
        ret = pcap_handle_packet_mmap(
                handle,
                callback,
                user,
                h.raw,
                h.h2->tp_len,
                h.h2->tp_mac,
                h.h2->tp_snaplen,
                h.h2->tp_sec,
                handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO ? h.h2->tp_nsec : h.h2->tp_nsec / 1000,
                VLAN_VALID(h.h2, h.h2),
                h.h2->tp_vlan_tci,
                VLAN_TPID(h.h2, h.h2));
        # 如果返回值为1，增加已捕获数据包数量
        if (ret == 1) {
            pkts++;
        # 如果返回值小于0，直接返回错误码
        } else if (ret < 0) {
            return ret;
        }

        '''
         * 将此块返回给内核，并在内核过滤后需要在用户空间再次过滤时计数
         '''
        packet_mmap_release(h.h2);
        # 如果需要在用户空间过滤的块数大于0，减少计数并在计数为0时取消用户空间过滤
        if (handlep->blocks_to_filter_in_userland > 0) {
            handlep->blocks_to_filter_in_userland--;
            if (handlep->blocks_to_filter_in_userland == 0) {
                '''
                 * 不再需要在用户空间过滤块
                 '''
                handlep->filter_in_userland = 0;
            }
        }

        # 下一个块
        if (++handle->offset >= handle->cc)
            handle->offset = 0;

        # 检查中断循环条件
        if (handle->break_loop) {
            handle->break_loop = 0;
            return PCAP_ERROR_BREAK;
        }
    }
    # 返回已捕获数据包数量
    return pkts;
#ifdef HAVE_TPACKET3
static int
pcap_read_linux_mmap_v3(pcap_t *handle, int max_packets, pcap_handler callback,
        u_char *user)
{
    // 获取对应的私有数据结构
    struct pcap_linux *handlep = handle->priv;
    // 定义联合体变量h
    union thdr h;
    // 初始化包计数器
    int pkts = 0;
    int ret;

again:
    // 如果当前数据包为空
    if (handlep->current_packet == NULL) {
        /* wait for frames availability.*/
        // 获取当前数据包的指针
        h.raw = RING_GET_CURRENT_FRAME(handle);
        // 如果无法获取数据包
        if (!packet_mmap_v3_acquire(h.h3)) {
            /*
             * The current frame is owned by the kernel; wait
             * for a frame to be handed to us.
             */
            // 等待数据包被传递给我们
            ret = pcap_wait_for_frames_mmap(handle);
            // 如果出现错误
            if (ret) {
                return ret;
            }
        }
    }
    // 获取当前数据包的指针
    h.raw = RING_GET_CURRENT_FRAME(handle);
    // 如果无法获取数据包
    if (!packet_mmap_v3_acquire(h.h3)) {
        // 如果包计数为0且超时为0
        if (pkts == 0 && handlep->timeout == 0) {
            /* Block until we see a packet. */
            // 阻塞直到看到一个数据包
            goto again;
        }
        return pkts;
    }

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
    // 如果包计数是无限的，将其限制在INT_MAX
    if (PACKET_COUNT_IS_UNLIMITED(max_packets))
        max_packets = INT_MAX;

    }
    // 如果包计数为0且超时为0
    if (pkts == 0 && handlep->timeout == 0) {
        /* Block until we see a packet. */
        // 阻塞直到看到一个数据包
        goto again;
    }
    return pkts;
}
#endif /* HAVE_TPACKET3 */

/*
 *  Attach the given BPF code to the packet capture device.
 */
static int
pcap_setfilter_linux(pcap_t *handle, struct bpf_program *filter)
{
    // 获取对应的私有数据结构
    struct pcap_linux *handlep;
    // 定义过滤器代码结构
    struct sock_fprog    fcode;
    // 是否可以在内核中过滤
    int            can_filter_in_kernel;
    int            err = 0;
    # 声明整型变量 n 和 offset
    int            n, offset;

    # 如果句柄为空，则返回-1
    if (!handle)
        return -1;
    
    # 如果过滤器为空，则将错误信息写入句柄的错误缓冲区，返回-1
    if (!filter) {
            pcap_strlcpy(handle->errbuf, "setfilter: No filter specified",
            PCAP_ERRBUF_SIZE);
        return -1;
    }

    # 获取句柄的私有数据
    handlep = handle->priv;

    # 创建过滤器的私有副本
    if (install_bpf_program(handle, filter) < 0)
        # 如果安装 BPF 程序失败，则返回-1
        /* install_bpf_program() filled in errbuf */
        return -1;

    # 默认情况下运行用户级数据包过滤器，如果安装内核过滤器成功，则会被覆盖
    handlep->filter_in_userland = 1;

    # 如果可能，安装内核级过滤器
#ifdef USHRT_MAX
    // 如果定义了 USHRT_MAX，检查过滤器代码长度是否超过 USHRT_MAX
    if (handle->fcode.bf_len > USHRT_MAX) {
        /*
         * fcode.len is an unsigned short for current kernel.
         * I have yet to see BPF-Code with that much
         * instructions but still it is possible. So for the
         * sake of correctness I added this check.
         */
        // 如果过滤器代码长度超过 USHRT_MAX，输出警告信息
        fprintf(stderr, "Warning: Filter too complex for kernel\n");
        // 将 fcode.len 置为 0，fcode.filter 置为 NULL
        fcode.len = 0;
        fcode.filter = NULL;
        // 设置 can_filter_in_kernel 为 0
        can_filter_in_kernel = 0;
    } else
#endif /* USHRT_MAX */
    {
        /*
         * Oh joy, the Linux kernel uses struct sock_fprog instead
         * of struct bpf_program and of course the length field is
         * of different size. Pointed out by Sebastian
         *
         * Oh, and we also need to fix it up so that all "ret"
         * instructions with non-zero operands have MAXIMUM_SNAPLEN
         * as the operand if we're not capturing in memory-mapped
         * mode, and so that, if we're in cooked mode, all memory-
         * reference instructions use special magic offsets in
         * references to the link-layer header and assume that the
         * link-layer payload begins at 0; "fix_program()" will do
         * that.
         */
        // 调用 fix_program() 函数修正过滤器代码
        switch (fix_program(handle, &fcode)) {

        case -1:
        default:
            /*
             * Fatal error; just quit.
             * (The "default" case shouldn't happen; we
             * return -1 for that reason.)
             */
            // 出现致命错误，直接退出
            return -1;

        case 0:
            /*
             * The program performed checks that we can't make
             * work in the kernel.
             */
            // 过滤器代码执行了无法在内核中实现的检查
            can_filter_in_kernel = 0;
            break;

        case 1:
            /*
             * We have a filter that'll work in the kernel.
             */
            // 过滤器可以在内核中工作
            can_filter_in_kernel = 1;
            break;
        }
    }
    /*
     * 注意：在这一点上，我们已经设置了"fcode"的"len"和"filter"字段。
     * 至少在2.6.32.4内核中，这些是"sock_fprog"结构的唯一成员，
     * 因此我们初始化该结构的每个成员。
     *
     * 如果"fcode"中有任何未初始化的内容，
     * 那么它要么是在以后的内核版本中添加的字段，
     * 要么是填充。
     *
     * 如果添加了新的字段，那么需要更新此代码以正确设置它。
     *
     * 如果没有其他字段，那么：
     *
     *    如果Linux内核查看填充，那么它有bug；
     *
     *    如果Linux内核不查看填充，那么如果某个工具抱怨我们向内核传递未初始化的数据，
     *    那么该工具有bug，需要理解这只是填充。
     */
    # 如果可以在内核中过滤
    if (can_filter_in_kernel) {
        # 尝试在内核中设置过滤器，如果成功则不需要在用户空间中过滤
        if ((err = set_kernel_filter(handle, &fcode)) == 0)
        {
            # 安装成功 - 使用内核过滤器，因此不需要在用户空间中过滤
            handlep->filter_in_userland = 0;
        }
        # 如果出现非致命错误
        else if (err == -1)    /* Non-fatal error */
        {
            # 如果由于除了“此内核未配置为支持套接字过滤器”之外的其他原因而无法安装过滤器，则打印警告
            if (errno == ENOMEM) {
                # 内核内存分配失败，或者为此套接字分配了太多“其他/选项内存”。建议增加“其他/选项内存”限制。
                fprintf(stderr,
                    "Warning: Couldn't allocate kernel memory for filter: try increasing net.core.optmem_max with sysctl\n");
            } else if (errno != ENOPROTOOPT && errno != EOPNOTSUPP) {
                # 如果内核过滤器失败，则打印警告
                fprintf(stderr,
                    "Warning: Kernel filter failed: %s\n",
                    pcap_strerror(errno));
            }
        }
    }

    # 如果不使用内核过滤器，则清除以前可能存在的内核过滤器，例如因为先前的过滤器可以在内核中工作，或者因为一些其他代码通过某种方式附加了过滤器到套接字，而不是调用“pcap_setfilter()”。否则，内核过滤器可能会过滤掉通过新的用户空间过滤器的数据包。
    # 如果在用户空间过滤数据包
    if (handlep->filter_in_userland) {
        # 重置内核过滤器，如果失败则设置错误消息并返回-2
        if (reset_kernel_filter(handle) == -1) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "can't remove kernel filter");
            err = -2;    /* fatal error */
        }
    }

    /*
     * 释放由 "fix_program()" 创建的过滤器的副本
     */
    if (fcode.filter != NULL)
        free(fcode.filter);

    # 如果发生致命错误，则返回-1
    if (err == -2)
        /* Fatal error */
        return -1;

    /*
     * 如果在用户空间过滤数据包，则不需要做任何操作；
     * 新的过滤器将用于下一个数据包。
     */
    if (handlep->filter_in_userland)
        return 0;

    /*
     * 我们在内核中过滤数据包；当前环形缓冲区中的所有块中的数据包
     * 已经通过旧的过滤器过滤，因此需要使用新的过滤器在用户空间过滤。
     *
     * 获取这样的块数量的上限；首先，向后遍历环形缓冲区并计算空闲块的数量。
     */
    offset = handle->offset;
    if (--offset < 0)
        offset = handle->cc - 1;
    for (n=0; n < handle->cc; ++n) {
        if (--offset < 0)
            offset = handle->cc - 1;
        if (pcap_get_ring_frame_status(handle, offset) != TP_STATUS_KERNEL)
            break;
    }

    /*
     * 如果找到空闲块，则将空闲块数量减1，以防我们在计数时与另一个
     * 线程竞争，该线程在我们更改过滤器之前添加了一个数据包并运行了过滤器。
     *
     * XXX - 是否可能以这种方式添加多个块？
     *
     * XXX - 是否有办法避免这种竞争，例如等待所有通过旧过滤器的数据包
     * 被添加到环形缓冲区？
     */
    if (n != 0)
        n--;
    /*
     * 将要在用户空间过滤的数据包块数设置为环形缓冲区中总块数减去找到的空闲块数，并且打开用户空间过滤。
     * （要在用户空间过滤的数据包块数保证不为零 - 上面的 n 不能被设置为大于 handle->cc 的值，如果它等于 handle->cc，它也不会是零，因此会被递减到 handle->cc - 1。）
     */
    handlep->blocks_to_filter_in_userland = handle->cc - n;
    handlep->filter_in_userland = 1;

    return 0;
/*
 *  返回给定设备名称的索引。在失败时填充 ebuf 并返回 -1。
 */
static int
iface_get_id(int fd, const char *device, char *ebuf)
{
    // 初始化 ifreq 结构体
    struct ifreq    ifr;
    memset(&ifr, 0, sizeof(ifr));
    // 将设备名称复制到 ifr_name 字段
    pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

    // 使用 ioctl 获取设备索引
    if (ioctl(fd, SIOCGIFINDEX, &ifr) == -1) {
        // 如果失败，填充错误信息到 ebuf 并返回 -1
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "SIOCGIFINDEX");
        return -1;
    }

    // 返回设备索引
    return ifr.ifr_ifindex;
}

/*
 *  将与 FD 关联的套接字绑定到给定的设备。
 *  成功时返回 0，硬错误时返回 PCAP_ERROR_ 值。
 */
static int
iface_bind(int fd, int ifindex, char *ebuf, int protocol)
{
    // 初始化 sockaddr_ll 结构体
    struct sockaddr_ll    sll;
    int            ret, err;
    socklen_t        errlen = sizeof(err);

    memset(&sll, 0, sizeof(sll));
    sll.sll_family        = AF_PACKET;
    sll.sll_ifindex        = ifindex < 0 ? 0 : ifindex;
    sll.sll_protocol    = protocol;

    // 将套接字绑定到给定设备
    if (bind(fd, (struct sockaddr *) &sll, sizeof(sll)) == -1) {
        if (errno == ENETDOWN) {
            /*
             * 返回“网络关闭”指示，以便应用程序可以报告而不是
             * 说我们有一个神秘的失败，并建议他们向
             * libpcap 开发人员报告问题。
             */
            return PCAP_ERROR_IFACE_NOT_UP;
        }
        if (errno == ENODEV) {
            /*
             * 没有更多要说的，所以清除错误消息。
             */
            ebuf[0] = '\0';
            ret = PCAP_ERROR_NO_SUCH_DEVICE;
        } else {
            ret = PCAP_ERROR;
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "bind");
        }
        return ret;
    }

    /* 有任何待处理的错误，例如，网络关闭了吗？ */
}
    # 检查套接字错误状态
    if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &errlen) == -1) {
        # 格式化错误消息，返回错误代码
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "getsockopt (SO_ERROR)");
        return PCAP_ERROR;
    }

    # 如果错误代码为 ENETDOWN，表示网络不可用
    if (err == ENETDOWN) {
        '''
         * 返回“网络不可用”指示，以便应用程序可以报告这一点，
         * 而不是说我们遇到了一个神秘的故障，
         * 并建议他们向 libpcap 开发人员报告问题。
         '''
        return PCAP_ERROR_IFACE_NOT_UP;
    } else if (err > 0) {
        # 格式化错误消息，返回错误代码
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, err, "bind");
        return PCAP_ERROR;
    }

    # 没有错误，返回 0
    return 0;
}
/*
 * 尝试进入监控模式。
 * 如果我们有 libnl，尝试创建一个新的监控模式设备并在其上捕获；否则，只是说“不支持”。
 */
#ifdef HAVE_LIBNL
static int
enter_rfmon_mode(pcap_t *handle, int sock_fd, const char *device)
{
    struct pcap_linux *handlep = handle->priv;
    int ret;
    char phydev_path[PATH_MAX+1];
    struct nl80211_state nlstate;
    struct ifreq ifr;
    u_int n;

    /*
     * 这是一个 mac80211 设备吗？
     */
    ret = get_mac80211_phydev(handle, device, phydev_path, PATH_MAX);
    if (ret < 0)
        return ret;    /* 错误 */
    if (ret == 0)
        return 0;    /* 没有错误，但不是 mac80211 设备 */

    /*
     * XXX - 这已经是一个 monN 设备了吗？
     * 如果是，我们就完成了。
     */

    /*
     * 好的，显然这是一个 mac80211 设备。
     * 尝试为其找到一个未使用的 monN 设备。
     */
    ret = nl80211_init(handle, &nlstate, device);
    if (ret != 0)
        return ret;
    for (n = 0; n < UINT_MAX; n++) {
        /*
         * 尝试 mon{n}。
         */
        char mondevice[3+10+1];    /* mon{UINT_MAX}\0 */

        snprintf(mondevice, sizeof mondevice, "mon%u", n);
        ret = add_mon_if(handle, sock_fd, &nlstate, device, mondevice);
        if (ret == 1) {
            /*
             * 成功。我们暂时不清理 libnl 状态，因为稍后还会用到它。
             */
            goto added;
        }
        if (ret < 0) {
            /*
             * 严重故障。只需返回 ret；handle->errbuf 已经被设置。
             */
            nl80211_cleanup(&nlstate);
            return ret;
        }
    }

    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "%s: 没有空闲的 monN 接口", device);
    nl80211_cleanup(&nlstate);
    return PCAP_ERROR;

added:

#if 0
    /*
     * 休眠 0.1 秒。
     */
    delay.tv_sec = 0;
    delay.tv_nsec = 500000000;
    nanosleep(&delay, NULL);
#endif
    /*
     * 如果我们还没有这样做，就安排在退出时调用"pcap_close_all()"
     */
    if (!pcap_do_addexit(handle)) {
        /*
         * "atexit()" 失败；不要将接口设置为 rfmon 模式，直接放弃
         */
        del_mon_if(handle, sock_fd, &nlstate, device,
            handlep->mondevice);
        nl80211_cleanup(&nlstate);
        return PCAP_ERROR;
    }

    /*
     * 现在配置监视接口
     */
    memset(&ifr, 0, sizeof(ifr));
    pcap_strlcpy(ifr.ifr_name, handlep->mondevice, sizeof(ifr.ifr_name));
    if (ioctl(sock_fd, SIOCGIFFLAGS, &ifr) == -1) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "%s: 无法获取 %s 的标志", device,
            handlep->mondevice);
        del_mon_if(handle, sock_fd, &nlstate, device,
            handlep->mondevice);
        nl80211_cleanup(&nlstate);
        return PCAP_ERROR;
    }
    ifr.ifr_flags |= IFF_UP|IFF_RUNNING;
    if (ioctl(sock_fd, SIOCSIFFLAGS, &ifr) == -1) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "%s: 无法为 %s 设置标志", device,
            handlep->mondevice);
        del_mon_if(handle, sock_fd, &nlstate, device,
            handlep->mondevice);
        nl80211_cleanup(&nlstate);
        return PCAP_ERROR;
    }

    /*
     * 成功。清理 libnl 状态
     */
    nl80211_cleanup(&nlstate);

    /*
     * 注意，当我们关闭句柄时，必须删除监视设备
     */
    handlep->must_do_on_close |= MUST_DELETE_MONIF;

    /*
     * 将其添加到要在退出时关闭的 pcap 列表中
     */
    pcap_add_to_pcaps_to_close(handle);

    return 1;
}
#else /* HAVE_LIBNL */
// 如果没有 libnl 库，则无法进行监控模式
static int
enter_rfmon_mode(pcap_t *handle _U_, int sock_fd _U_, const char *device _U_)
{
    /*
     * We don't have libnl, so we can't do monitor mode.
     */
    return 0;
}
#endif /* HAVE_LIBNL */

#if defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP)
/*
 * Map SOF_TIMESTAMPING_ values to PCAP_TSTAMP_ values.
 */
// 定义一个结构体数组，用于映射 SOF_TIMESTAMPING_ 值到 PCAP_TSTAMP_ 值
static const struct {
    int soft_timestamping_val;
    int pcap_tstamp_val;
} sof_ts_type_map[3] = {
    { SOF_TIMESTAMPING_SOFTWARE, PCAP_TSTAMP_HOST },
    { SOF_TIMESTAMPING_SYS_HARDWARE, PCAP_TSTAMP_ADAPTER },
    { SOF_TIMESTAMPING_RAW_HARDWARE, PCAP_TSTAMP_ADAPTER_UNSYNCED }
};
#define NUM_SOF_TIMESTAMPING_TYPES    (sizeof sof_ts_type_map / sizeof sof_ts_type_map[0])

/*
 * Set the list of time stamping types to include all types.
 */
// 设置时间戳类型列表，包括所有类型
static int
iface_set_all_ts_types(pcap_t *handle, char *ebuf)
{
    u_int i;

    handle->tstamp_type_list = malloc(NUM_SOF_TIMESTAMPING_TYPES * sizeof(u_int));
    if (handle->tstamp_type_list == NULL) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return -1;
    }
    for (i = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++)
        handle->tstamp_type_list[i] = sof_ts_type_map[i].pcap_tstamp_val;
    handle->tstamp_type_count = NUM_SOF_TIMESTAMPING_TYPES;
    return 0;
}

/*
 * Get a list of time stamp types.
 */
#ifdef ETHTOOL_GET_TS_INFO
// 获取时间戳类型列表
static int
iface_get_ts_types(const char *device, pcap_t *handle, char *ebuf)
{
    int fd;
    struct ifreq ifr;
    struct ethtool_ts_info info;
    int num_ts_types;
    u_int i, j;

    /*
     * This doesn't apply to the "any" device; you can't say "turn on
     * hardware time stamping for all devices that exist now and arrange
     * that it be turned on for any device that appears in the future",
     * and not all devices even necessarily *support* hardware time
     * stamping, so don't report any time stamp types.
     */
    # 如果设备名为"any"，则将时间戳类型列表设为NULL并返回0
    if (strcmp(device, "any") == 0) {
        handle->tstamp_type_list = NULL;
        return 0;
    }

    # 创建一个套接字以获取时间戳功能
    fd = get_if_ioctl_socket();
    # 如果套接字创建失败，设置错误消息并返回-1
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "socket for SIOCETHTOOL(ETHTOOL_GET_TS_INFO)");
        return -1;
    }

    # 初始化ifr结构体，并将设备名拷贝到ifr.ifr_name中
    memset(&ifr, 0, sizeof(ifr));
    pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));
    # 初始化info结构体，并设置cmd为ETHTOOL_GET_TS_INFO
    memset(&info, 0, sizeof(info));
    info.cmd = ETHTOOL_GET_TS_INFO;
    ifr.ifr_data = (caddr_t)&info;
    # 发送SIOCETHTOOL命令获取时间戳信息，如果失败则根据错误类型处理
    if (ioctl(fd, SIOCETHTOOL, &ifr) == -1) {
        int save_errno = errno;
        # 关闭套接字
        close(fd);
        # 根据不同的错误类型进行处理
        switch (save_errno) {

        case EOPNOTSUPP:
        case EINVAL:
            # 如果不支持获取时间戳类型，则返回所有可能的类型
            if (iface_set_all_ts_types(handle, ebuf) == -1)
                return -1;
            return 0;

        case ENODEV:
            # 如果设备不存在，则将时间戳类型列表设为NULL并返回0
            handle->tstamp_type_list = NULL;
            return 0;

        default:
            # 其他错误，设置错误消息并返回-1
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                save_errno,
                "%s: SIOCETHTOOL(ETHTOOL_GET_TS_INFO) ioctl failed",
                device);
            return -1;
        }
    }
    # 关闭套接字
    close(fd);

    # 检查是否支持对所有数据包进行硬件时间戳
    # 如果 info.rx_filters 中不包含 HWTSTAMP_FILTER_ALL 标志位
    if (!(info.rx_filters & (1 << HWTSTAMP_FILTER_ALL))) {
        # 不报告任何时间戳类型
        #
        # 一些设备可能不报告支持 HWTSTAMP_FILTER_ALL，或者报告支持但只对少数 PTP 数据包进行时间戳。
        # 可能在后续修复了这个问题。
        handle->tstamp_type_list = NULL;
        # 返回 0，表示成功
        return 0;
    }

    # 初始化时间戳类型数量为 0
    num_ts_types = 0;
    # 遍历 NUM_SOF_TIMESTAMPING_TYPES
    for (i = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++) {
        # 如果 info.so_timestamping 中包含 sof_ts_type_map[i].soft_timestamping_val 标志位
        if (info.so_timestamping & sof_ts_type_map[i].soft_timestamping_val)
            # 时间戳类型数量加一
            num_ts_types++;
    }
    # 如果时间戳类型数量不为 0
    if (num_ts_types != 0) {
        # 分配存储时间戳类型的内存空间
        handle->tstamp_type_list = malloc(num_ts_types * sizeof(u_int));
        # 如果分配内存失败
        if (handle->tstamp_type_list == NULL) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "malloc");
            # 返回 -1，表示失败
            return -1;
        }
        # 遍历 NUM_SOF_TIMESTAMPING_TYPES
        for (i = 0, j = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++) {
            # 如果 info.so_timestamping 中包含 sof_ts_type_map[i].soft_timestamping_val 标志位
            if (info.so_timestamping & sof_ts_type_map[i].soft_timestamping_val) {
                # 将 sof_ts_type_map[i].pcap_tstamp_val 存入时间戳类型列表
                handle->tstamp_type_list[j] = sof_ts_type_map[i].pcap_tstamp_val;
                # j 自增
                j++;
            }
        }
        # 设置时间戳类型数量
        handle->tstamp_type_count = num_ts_types;
    } else
        # 时间戳类型列表为空
        handle->tstamp_type_list = NULL;

    # 返回 0，表示成功
    return 0;
#else /* ETHTOOL_GET_TS_INFO */
static int
iface_get_ts_types(const char *device, pcap_t *handle, char *ebuf)
{
    /*
     * 如果设备名为"any"，表示对所有设备都不进行硬件时间戳处理，将时间戳类型列表设为NULL，并返回0
     */
    if (strcmp(device, "any") == 0) {
        handle->tstamp_type_list = NULL;
        return 0;
    }

    /*
     * 由于没有 ioctl 可以用来询问支持的时间戳类型，因此假设支持所有类型
     */
    if (iface_set_all_ts_types(handle, ebuf) == -1)
        return -1;
    return 0;
}
#endif /* ETHTOOL_GET_TS_INFO */
#else  /* defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP) */
static int
iface_get_ts_types(const char *device _U_, pcap_t *p _U_, char *ebuf _U_)
{
    /*
     * 没有需要获取的内容，因此总是“成功”
     */
    return 0;
}
#endif /* defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP) */
/*
 * 检查是否存在任何形式的分段/重组卸载。
 *
 * 我们使用 SIOCETHTOOL 来检查各种类型的卸载；如果未定义 SIOCETHTOOL，或者我们没有任何关于任何类型卸载的 #define，那么我们无法进行检查，所以我们只能说“不，没有”。
 *
 * 我们将 EOPNOTSUPP、EINVAL 和（如果 eperm_ok 为真）EPERM 视为操作不受支持的指示。我们对 EPERM 的处理方式很奇怪，因为后续内核中的 SIOCETHTOOL 代码 1）不支持 ETHTOOL_GUFO，2）也没有将其包括在不需要 CAP_NET_ADMIN 权限的 ethtool 操作列表中，3）在执行“这是允许的”检查之前执行“这是甚至支持的”检查，因此它会因为“这是不允许的”而失败，而不是“这甚至不支持”。为了解决这个烦恼，我们只将 EPERM 视为第一个特性的错误，并假设它们都执行相同的权限检查，因此如果第一个特性被允许，那么如果支持，所有其他特性都被允许。
 */
#if defined(SIOCETHTOOL) && (defined(ETHTOOL_GTSO) || defined(ETHTOOL_GUFO) || defined(ETHTOOL_GGSO) || defined(ETHTOOL_GFLAGS) || defined(ETHTOOL_GGRO))
static int
iface_ethtool_flag_ioctl(pcap_t *handle, int cmd, const char *cmdname,
    int eperm_ok)
{
    struct ifreq    ifr;
    struct ethtool_value eval;

    // 初始化 ifreq 结构体
    memset(&ifr, 0, sizeof(ifr));
    // 将设备名称复制到 ifreq 结构体中
    pcap_strlcpy(ifr.ifr_name, handle->opt.device, sizeof(ifr.ifr_name));
    // 设置 ethtool_value 结构体的 cmd 和 data 字段
    eval.cmd = cmd;
    eval.data = 0;
    ifr.ifr_data = (caddr_t)&eval;
    # 如果调用 ioctl 函数失败
    if (ioctl(handle->fd, SIOCETHTOOL, &ifr) == -1) {
        # 如果错误码是 EOPNOTSUPP、EINVAL 或者 (errno == EPERM && eperm_ok)
        if (errno == EOPNOTSUPP || errno == EINVAL ||
            (errno == EPERM && eperm_ok)) {
            '''
             * OK, let's just return 0, which, in our
             * case, either means "no, what we're asking
             * about is not enabled" or "all the flags
             * are clear (i.e., nothing is enabled)".
             '''
            # 返回 0，表示所询问的内容未启用或者所有标志都清除了（即，没有内容被启用）
            return 0;
        }
        # 格式化错误消息到错误缓冲区
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "%s: SIOCETHTOOL(%s) ioctl failed",
            handle->opt.device, cmdname);
        # 返回 -1，表示调用失败
        return -1;
    }
    # 返回 eval.data
    return eval.data;
/*
 * XXX - it's annoying that we have to check for offloading at all, but,
 * given that we have to, it's still annoying that we have to check for
 * particular types of offloading, especially that shiny new types of
 * offloading may be added - and, worse, may not be checkable with
 * a particular ETHTOOL_ operation; ETHTOOL_GFEATURES would, in
 * theory, give those to you, but the actual flags being used are
 * opaque (defined in a non-uapi header), and there doesn't seem to
 * be any obvious way to ask the kernel what all the offloading flags
 * are - at best, you can ask for a set of strings(!) to get *names*
 * for various flags.  (That whole mechanism appears to have been
 * designed for the sole purpose of letting ethtool report flags
 * by name and set flags by name, with the names having no semantics
 * ethtool understands.)
 */
static int
iface_get_offload(pcap_t *handle)
{
    int ret;

#ifdef ETHTOOL_GTSO
    ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GTSO, "ETHTOOL_GTSO", 0);
    if (ret == -1)
        return -1;
    if (ret)
        return 1;    /* TCP segmentation offloading on */
#endif

#ifdef ETHTOOL_GGSO
    /*
     * XXX - will this cause large unsegmented packets to be
     * handed to PF_PACKET sockets on transmission?  If not,
     * this need not be checked.
     */
    ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GGSO, "ETHTOOL_GGSO", 0);
    if (ret == -1)
        return -1;
    if (ret)
        return 1;    /* generic segmentation offloading on */
#endif

#ifdef ETHTOOL_GFLAGS
    ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GFLAGS, "ETHTOOL_GFLAGS", 0);
    if (ret == -1)
        return -1;
    if (ret & ETH_FLAG_LRO)
        return 1;    /* large receive offloading on */
#endif

#ifdef ETHTOOL_GGRO
    /*
     * XXX - will this cause large reassembled packets to be
     * handed to PF_PACKET sockets on receipt?  If not,
     * this need not be checked.
     */
    # 调用接口进行以太网工具标志的 IOCTL 操作，获取通用接收卸载的状态
    ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GGRO, "ETHTOOL_GGRO", 0);
    # 如果返回结果为 -1，表示操作失败，直接返回 -1
    if (ret == -1)
        return -1;
    # 如果返回结果为真，表示通用（大）接收卸载已开启，返回 1
    if (ret)
        return 1;    # generic (large) receive offloading on
#ifdef ETHTOOL_GUFO
    /*
     * 在后续的内核版本中移除了对此的支持，对于这些内核，它会返回 EPERM 而不是 EOPNOTSUPP（参见 iface_ethtool_flag_ioctl() 的注释）
     * 因此，将其放在最后执行
     */
    ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GUFO, "ETHTOOL_GUFO", 1);
    if (ret == -1)
        return -1;
    if (ret)
        return 1;    /* UDP 分片卸载已开启 */
#endif

    return 0;
}
#else /* SIOCETHTOOL */
static int
iface_get_offload(pcap_t *handle _U_)
{
    /*
     * XXX - 如果没有 ethtool ioctls，我们是否需要获取此信息？
     * 如果需要，我们该如何做？
     */
    return 0;
}
#endif /* SIOCETHTOOL */

static struct dsa_proto {
    const char *name;
    bpf_u_int32 linktype;
} dsa_protos[] = {
    /*
     * None 是特殊的，表示接口没有配置任何标记协议，因此是标准以太网接口
     */
    { "none", DLT_EN10MB },
    { "brcm", DLT_DSA_TAG_BRCM },
    { "brcm-prepend", DLT_DSA_TAG_BRCM_PREPEND },
    { "dsa", DLT_DSA_TAG_DSA },
    { "edsa", DLT_DSA_TAG_EDSA },
};

static int
iface_dsa_get_proto_info(const char *device, pcap_t *handle)
{
    char *pathstr;
    unsigned int i;
    /*
     * 将其设置为比 PCAP_ERRBUF_SIZE 小得多的值；
     * 标记 *不应该*有非常长的名称，将其设置得更小可以避免新版本的 GCC 抱怨如果我们不支持该标记，错误消息可能会溢出错误消息缓冲区
     */
    char buf[128];
    ssize_t r;
    int fd;

    fd = asprintf(&pathstr, "/sys/class/net/%s/dsa/tagging", device);
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
                      fd, "asprintf");
        return PCAP_ERROR;
    }

    fd = open(pathstr, O_RDONLY);
    free(pathstr);
    /*
     * 如果文件描述符小于0，则返回0，这不是致命错误，内核>=4.20 *可能*会暴露这个属性
     */
    if (fd < 0)
        return 0;

    // 读取文件内容到缓冲区中
    r = read(fd, buf, sizeof(buf) - 1);
    // 如果读取失败
    if (r <= 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
                      errno, "read");
        // 关闭文件描述符
        close(fd);
        return PCAP_ERROR;
    }
    // 关闭文件描述符
    close(fd);

    /*
     * 缓冲区应该以LF结尾
     */
    if (buf[r - 1] == '\n')
        r--;
    buf[r] = '\0';

    // 遍历DSA协议数组
    for (i = 0; i < sizeof(dsa_protos) / sizeof(dsa_protos[0]); i++) {
        // 如果协议名长度与缓冲区长度相等，并且协议名与缓冲区内容相同
        if (strlen(dsa_protos[i].name) == (size_t)r &&
            strcmp(buf, dsa_protos[i].name) == 0) {
            // 设置链路类型
            handle->linktype = dsa_protos[i].linktype;
            switch (dsa_protos[i].linktype) {
            case DLT_EN10MB:
                return 0;
            default:
                return 1;
            }
        }
    }

    // 格式化错误消息，表示不支持的DSA标签
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
              "unsupported DSA tag: %s", buf);

    return PCAP_ERROR;
/*
 *  查询给定接口的 MTU（最大传输单元）。
 */
static int
iface_get_mtu(int fd, const char *device, char *ebuf)
{
    // 定义一个 ifreq 结构体
    struct ifreq    ifr;

    // 如果设备名为空，则返回一个大于所有 MTU 的值
    if (!device)
        return BIGGER_THAN_ALL_MTUS;

    // 将 ifr 结构体清零
    memset(&ifr, 0, sizeof(ifr));
    // 将设备名拷贝到 ifr 结构体中
    pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

    // 调用 ioctl 函数获取接口的 MTU
    if (ioctl(fd, SIOCGIFMTU, &ifr) == -1) {
        // 如果出错，格式化错误消息并返回 -1
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGIFMTU");
        return -1;
    }

    // 返回接口的 MTU
    return ifr.ifr_mtu;
}

/*
 *  获取给定接口的硬件类型，返回 ARPHRD_xxx 常量。
 */
static int
iface_get_arptype(int fd, const char *device, char *ebuf)
{
    // 定义一个 ifreq 结构体和一个返回值
    struct ifreq    ifr;
    int        ret;

    // 将 ifr 结构体清零
    memset(&ifr, 0, sizeof(ifr));
    // 将设备名拷贝到 ifr 结构体中
    pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

    // 调用 ioctl 函数获取接口的硬件地址
    if (ioctl(fd, SIOCGIFHWADDR, &ifr) == -1) {
        // 如果出错，根据不同的错误类型进行处理
        if (errno == ENODEV) {
            // 如果是设备不存在的错误，返回对应的错误码并清空错误消息
            ret = PCAP_ERROR_NO_SUCH_DEVICE;
            ebuf[0] = '\0';
        } else {
            // 其他错误，格式化错误消息并返回错误码
            ret = PCAP_ERROR;
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "SIOCGIFHWADDR");
        }
        return ret;
    }

    // 返回接口的硬件地址类型
    return ifr.ifr_hwaddr.sa_family;
}

// 修复程序
static int
fix_program(pcap_t *handle, struct sock_fprog *fcode)
{
    // 获取 pcap_linux 结构体
    struct pcap_linux *handlep = handle->priv;
    size_t prog_size;
    register int i;
    register struct bpf_insn *p;
    struct bpf_insn *f;
    int len;

    /*
     * 复制过滤器，并在必要时修改该副本。
     */
    // 计算过滤器的大小
    prog_size = sizeof(*handle->fcode.bf_insns) * handle->fcode.bf_len;
    len = handle->fcode.bf_len;
    // 分配内存用于存储过滤器的副本
    f = (struct bpf_insn *)malloc(prog_size);
    // 如果分配内存失败，格式化错误消息并返回 -1
    if (f == NULL) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return -1;
    }
*/
    # 将 handle->fcode.bf_insns 指向的内存块的内容复制到 f 指向的内存块中，大小为 prog_size
    memcpy(f, handle->fcode.bf_insns, prog_size);
    # 设置 fcode->len 的值为 len
    fcode->len = len;
    # 设置 fcode->filter 指向 f 指向的内存块
    fcode->filter = (struct sock_filter *) f;

    # 遍历 f 数组
    for (i = 0; i < len; ++i) {
        p = &f[i];
        '''
         * 这是什么类型的指令？
         */
        # 根据指令的类别进行判断
        switch (BPF_CLASS(p->code)) {

        case BPF_LD:
        case BPF_LDX:
            '''
             * 这是一个加载指令；它是从数据包中加载吗？
             */
            # 根据加载指令的模式进行判断
            switch (BPF_MODE(p->code)) {

            case BPF_ABS:
            case BPF_IND:
            case BPF_MSH:
                '''
                 * 是的；我们处于 cooked 模式吗？
                 */
                # 如果处于 cooked 模式，则需要修正这个指令
                if (handlep->cooked) {
                    '''
                     * 是的，所以我们需要修正这个指令。
                     */
                    # 如果修正偏移量失败，则返回 0，通知调用者转到用户空间处理
                    if (fix_offset(handle, p) < 0) {
                        '''
                         * 我们未能这样做。
                         * 返回 0，这样我们的调用者
                         * 知道转到用户空间处理。
                         */
                        return 0;
                    }
                }
                break;
            }
            break;
        }
    }
    # 成功执行，返回 1
    return 1;    /* we succeeded */
    // 检查偏移量是否需要修正
    if (p->k >= (bpf_u_int32)SKF_AD_OFF)
        return 0;
    // 如果数据链路类型是 DLT_LINUX_SLL2
    if (handle->linktype == DLT_LINUX_SLL2) {
        // 如果偏移量大于 SLL2 头部长度
        if (p->k >= SLL2_HDR_LEN) {
            // 减去 SLL2 头部长度
            p->k -= SLL2_HDR_LEN;
        } else if (p->k == 0) {
            // 将偏移量映射到协议字段的特殊内核偏移量
            p->k = SKF_AD_OFF + SKF_AD_PROTOCOL;
        } else if (p->k == 4) {
            // 将偏移量映射到 ifindex 字段的特殊内核偏移量
            p->k = SKF_AD_OFF + SKF_AD_IFINDEX;
        } else if (p->k == 10) {
            // 将偏移量映射到数据包类型字段的特殊内核偏移量
            p->k = SKF_AD_OFF + SKF_AD_PKTTYPE;
        } else if ((bpf_int32)(p->k) > 0) {
            // 如果偏移量在头部内，但不是上述字段之一，则无法在内核中处理，返回错误
            return -1;
        }
    } else {
        /*
         * 如果不是以上情况，说明偏移量是什么？
         */
        if (p->k >= SLL_HDR_LEN) {
            /*
             * 偏移量在链路层有效载荷内；从内核数据包过滤器的角度来看，它从偏移量0开始，因此需要减去链路层头的长度。
             */
            p->k -= SLL_HDR_LEN;
        } else if (p->k == 0) {
            /*
             * 这是数据包类型字段；将其映射到该字段的特殊内核偏移量。
             */
            p->k = SKF_AD_OFF + SKF_AD_PKTTYPE;
        } else if (p->k == 14) {
            /*
             * 这是协议字段；将其映射到该字段的特殊内核偏移量。
             */
            p->k = SKF_AD_OFF + SKF_AD_PROTOCOL;
        } else if ((bpf_int32)(p->k) > 0) {
            /*
             * 它在头部内，但不是那些字段之一；我们无法在内核中处理，因此返回到用户空间。
             */
            return -1;
        }
    }
    // 返回0表示成功
    return 0;
# 设置内核过滤器
static int
set_kernel_filter(pcap_t *handle, struct sock_fprog *fcode)
{
    # 初始化变量
    int total_filter_on = 0;
    int save_mode;
    int ret;
    int save_errno;

    """
    设置内核过滤器代码不会丢弃在更改过滤器时排队的所有数据包；
    这意味着在将新过滤器放到套接字上后，不符合新过滤器的数据包可能会出现，如果这些数据包尚未被读取。
    
    这意味着，例如，如果使用过滤器进行 tcpdump 捕获，捕获中的前几个数据包可能是不符合过滤器的数据包。
    
    因此，在设置内核过滤器时，我们丢弃在套接字上排队的所有数据包。（对于用户空间过滤器来说，因为用户空间过滤是在数据包排队后进行的，所以这不是一个问题。）
    
    为了清空这些数据包，我们将套接字设置为只读模式，并从套接字中读取数据包，直到没有更多数据包可读取。
    
    为了防止这成为一个无限循环 - 即，在我们清空队列时防止更多数据包到达 - 我们在清空队列之前将“总过滤器”（一个拒绝所有数据包的过滤器）放到套接字上。
    
    这段代码故意忽略任何错误，这样如果发生错误，您可能会得到错误的数据包，而不是在内核中进行过滤，即使可以在内核中进行过滤。
    """
    # 如果成功将过滤器应用到套接字上
    if (setsockopt(handle->fd, SOL_SOCKET, SO_ATTACH_FILTER,
               &total_fcode, sizeof(total_fcode)) == 0) {
        # 创建一个用于读取数据的缓冲区
        char drain[1];

        '''
         * 注意：我们已经将总过滤器应用到套接字上。
         '''
        # 标记总过滤器已经应用
        total_filter_on = 1;

        '''
         * 保存套接字的当前模式，并将其设置为非阻塞模式；
         * 通过读取数据包来清空套接字，直到出现错误（通常是“没有更多可读取的”错误）。
         '''
        # 保存套接字的当前模式
        save_mode = fcntl(handle->fd, F_GETFL, 0);
        # 如果无法获取套接字标志位，则返回错误
        if (save_mode == -1) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "can't get FD flags when changing filter");
            return -2;
        }
        # 将套接字设置为非阻塞模式
        if (fcntl(handle->fd, F_SETFL, save_mode | O_NONBLOCK) < 0) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "can't set nonblocking mode when changing filter");
            return -2;
        }
        # 通过读取数据包来清空套接字，直到出现错误
        while (recv(handle->fd, &drain, sizeof drain, MSG_TRUNC) >= 0)
            ;
        # 保存错误号
        save_errno = errno;
        # 如果错误号不是 EAGAIN，则表示出现致命错误
        if (save_errno != EAGAIN) {
            '''
             * 致命错误。
             *
             * 如果无法恢复模式或重置内核过滤器，则无能为力。
             '''
            # 恢复套接字模式或重置内核过滤器失败，则返回错误
            (void)fcntl(handle->fd, F_SETFL, save_mode);
            (void)reset_kernel_filter(handle);
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, save_errno,
                "recv failed when changing filter");
            return -2;
        }
        # 恢复套接字模式
        if (fcntl(handle->fd, F_SETFL, save_mode) == -1) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "can't restore FD flags when changing filter");
            return -2;
        }
    }

    '''
     * 现在附加新的过滤器。
     '''
    # 设置套接字选项，用于设置过滤器
    ret = setsockopt(handle->fd, SOL_SOCKET, SO_ATTACH_FILTER,
             fcode, sizeof(*fcode));
    # 如果设置失败，并且总过滤器已经开启
    if (ret == -1 && total_filter_on) {
        # 保存错误码
        save_errno = errno;

        # 尝试移除内核总过滤器，如果失败则报告为致命错误
        if (reset_kernel_filter(handle) == -1) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "can't remove kernel total filter");
            return -2;    # 致命错误
        }

        # 恢复之前保存的错误码
        errno = save_errno;
    }
    # 返回设置结果
    return ret;
# 重置内核过滤器
static int
reset_kernel_filter(pcap_t *handle)
{
    int ret;
    # 设置一个虚拟参数，因为 setsockopt() 需要一个参数，valgrind 需要初始化数值
    int dummy = 0;

    # 设置 SO_DETACH_FILTER 选项，移除内核过滤器
    ret = setsockopt(handle->fd, SOL_SOCKET, SO_DETACH_FILTER,
                   &dummy, sizeof(dummy));
    # 忽略 ENOENT 错误 - 表示“没有过滤器”，因此没有过滤器需要移除
    # 同样忽略 ENONET 错误，因为很多内核版本返回的是 ENONET 而不是 ENOENT
    if (ret == -1 && errno != ENOENT && errno != ENONET)
        return -1;
    return 0;
}

# 设置 Linux 网络数据包捕获的协议
int
pcap_set_protocol_linux(pcap_t *p, int protocol)
{
    # 检查是否已经激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    # 设置协议
    p->opt.protocol = protocol;
    return (0);
}

# 返回 Libpcap 版本字符串
const char *
pcap_lib_version(void)
{
    # 如果定义了 HAVE_TPACKET3，则返回带有 TPACKET_V3 的版本字符串，否则返回带有 TPACKET_V2 的版本字符串
#if defined(HAVE_TPACKET3)
    return (PCAP_VERSION_STRING " (with TPACKET_V3)");
#else
    return (PCAP_VERSION_STRING " (with TPACKET_V2)");
#endif
}
```