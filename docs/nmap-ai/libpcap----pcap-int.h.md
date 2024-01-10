# `nmap\libpcap\pcap-int.h`

```
# 版权声明，版权所有，禁止未经授权的源代码和二进制形式的再分发和修改
# 条件：1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
#      2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
#      3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
#         本产品包括由劳伦斯伯克利实验室的计算机系统工程组开发的软件
#      4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
# 免责声明：本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。在任何情况下，无论是合同、严格责任还是侵权（包括疏忽或其他方式），都不会有版权所有者或贡献者对任何直接、间接、偶发、特殊、惩罚性或后果性的损害承担责任，即使已被告知可能发生此类损害。
# ifndef 指令，如果未定义 pcap_int_h，则包含以下内容
# 包含标准库头文件
# 包含信号处理头文件
# 包含 libpcap 头文件
#ifdef MSDOS
  #include <fcntl.h>  // 包含文件操作相关的头文件
  #include <io.h>  // 包含输入输出相关的头文件
#endif

#include "varattrs.h"  // 包含自定义的头文件 varattrs.h
#include "fmtutils.h"  // 包含自定义的头文件 fmtutils.h

#include <stdarg.h>  // 包含标准参数处理相关的头文件

#include "portability.h"  // 包含自定义的头文件 portability.h

/*
 * 如果我们使用 Visual Studio 编译，确保我们至少有 VS 2015 或更高版本，以便有足够的 C99 支持。
 *
 * XXX - 验证我们在 UN*X 上至少有 C99 支持？
 *
 * MinGW 或各种 DOS 工具链呢？我们目前假设这些地方有足够的 C99 支持。
 */
#if defined(_MSC_VER)
  /*
   * 编译器是 MSVC。确保我们有 VS 2015 或更高版本。
   */
  #if _MSC_VER < 1900
    #error "Building libpcap requires VS 2015 or later"  // 如果不满足条件，输出错误信息
  #endif
#endif

/*
 * 版本字符串。
 * 使用 config.h 中的 PACKAGE_VERSION。
 */
#define PCAP_VERSION_STRING "libpcap version " PACKAGE_VERSION  // 定义 libpcap 版本字符串

#ifdef __cplusplus
extern "C" {
#endif

/*
 * 如果 pcap_new_api 被设置，我们禁用 pcap_lookupdev()，因为：
 *
 *    它在所有平台上都不是线程安全的，并且被标记为废弃的；
 *
 *    在 Windows 上，它可能返回 UTF-16LE 字符串，然后程序可能将其传递给 pcap_create()（或者传递给 pcap_open_live()，然后再传递给 pcap_create()），这要求 pcap_create() 使用一个 hack 来检查 UTF-16LE 字符串，而这个 hack 1) *不能* 100% 可靠，2) 有可能超出字符串的末尾。
 *
 * 我们保留它以兼容旧版本。
 *
 * 我们还在 pcap_create() 中禁用了上述的 hack。
 */
extern int pcap_new_api;  // 声明 pcap_new_api 变量

/*
 * 如果 pcap_utf_8_mode 被设置，在 Windows 上我们将字符串视为 UTF-8。
 *
 * 在 UN*X 上，我们假设所有字符串都是并应该是 UTF-8，不管此标志的设置如何。
 */
extern int pcap_utf_8_mode;  // 声明 pcap_utf_8_mode 变量

/*
 * 在大端机器上交换无符号长长整型时间戳的字节顺序。
 */
# 定义一个宏，用于将无符号长长整型数的字节顺序进行交换
#define SWAPLL(ull)  ((ull & 0xff00000000000000ULL) >> 56) | \
                      ((ull & 0x00ff000000000000ULL) >> 40) | \
                      ((ull & 0x0000ff0000000000ULL) >> 24) | \
                      ((ull & 0x000000ff00000000ULL) >> 8)  | \
                      ((ull & 0x00000000ff000000ULL) << 8)  | \
                      ((ull & 0x0000000000ff0000ULL) << 24) | \
                      ((ull & 0x000000000000ff00ULL) << 40) | \
                      ((ull & 0x00000000000000ffULL) << 56)

/*
 * 最大快照长度。
 *
 * 有些是任意的，但选择为：
 *
 *    1) 足够大以容纳最大大小的 Linux 回环数据包（65549）
 *       和一些使用 USBPcap 捕获的 USB 数据包：
 *
 *           https://desowin.org/usbpcap/
 *
 *       (> 131072, < 262144)
 *
 * 和
 *
 *    2) 足够小，不会导致尝试分配大量内存；一些应用程序可能会使用快照长度在保存文件头中控制它们分配的缓冲区的大小，因此大小为 2^31-1 可能不太好用。（libpcap 将其用作提示，但不会一开始分配大于 2 KiB 的缓冲区，并根据需要增加缓冲区的大小，但不会超过每个链路类型的最大快照长度。其他代码可能会天真地使用它；我们希望避免写入太大的快照长度，以免导致该代码出现问题。）
 *
 * 我们不在 pcap_set_snaplen() 中强制执行这一点，但我们在内部使用它。
 */
#define MAXIMUM_SNAPLEN        262144

/*
 * 用于测试字符类型的与地区无关的宏。
 * 这些可以传递任何整数值，而不用担心，例如，符号扩展字符值，与 C 宏不同。
 */
#define PCAP_ISDIGIT(c) \
    ((c) >= '0' && (c) <= '9')
#define PCAP_ISXDIGIT(c) \
    (((c) >= '0' && (c) <= '9') || \
     ((c) >= 'A' && (c) <= 'F') || \
     ((c) >= 'a' && (c) <= 'f'))

# 定义一个结构体，用于存储 pcap 选项
struct pcap_opt {
    char    *device;
    # 缓冲区超时时间
    int    timeout;    /* timeout for buffering */
    # 缓冲区大小
    u_int    buffer_size;
    # 混杂模式
    int    promisc;
    # 监控模式
    int    rfmon;        /* monitor mode */
    # 立即模式 - 尽快传递数据包
    int    immediate;    /* immediate mode - deliver packets as soon as they arrive */
    # 非阻塞模式 - 不等待数据包传递，返回“没有可用的数据包”
    int    nonblock;    /* non-blocking mode - don't wait for packets to be delivered, return "no packets available" */
    # 时间戳类型
    int    tstamp_type;
    # 时间戳精度
    int    tstamp_precision;

    '''
     * 平台相关选项
     */
#ifdef __linux__
    int    protocol;    /* 在创建 PF_PACKET 套接字时要使用的协议 */
#endif
#ifdef _WIN32
    int    nocapture_local;/* 禁用 NPF 回环 */
#endif
};

typedef int    (*activate_op_t)(pcap_t *);    /* 激活操作函数指针 */
typedef int    (*can_set_rfmon_op_t)(pcap_t *);    /* 是否可以设置 RFMON 操作函数指针 */
typedef int    (*read_op_t)(pcap_t *, int cnt, pcap_handler, u_char *);    /* 读取操作函数指针 */
typedef int    (*next_packet_op_t)(pcap_t *, struct pcap_pkthdr *, u_char **);    /* 下一个数据包操作函数指针 */
typedef int    (*inject_op_t)(pcap_t *, const void *, int);    /* 注入操作函数指针 */
typedef void    (*save_current_filter_op_t)(pcap_t *, const char *);    /* 保存当前过滤器操作函数指针 */
typedef int    (*setfilter_op_t)(pcap_t *, struct bpf_program *);    /* 设置过滤器操作函数指针 */
typedef int    (*setdirection_op_t)(pcap_t *, pcap_direction_t);    /* 设置方向操作函数指针 */
typedef int    (*set_datalink_op_t)(pcap_t *, int);    /* 设置数据链路操作函数指针 */
typedef int    (*getnonblock_op_t)(pcap_t *);    /* 获取非阻塞操作函数指针 */
typedef int    (*setnonblock_op_t)(pcap_t *, int);    /* 设置非阻塞操作函数指针 */
typedef int    (*stats_op_t)(pcap_t *, struct pcap_stat *);    /* 统计操作函数指针 */
typedef void    (*breakloop_op_t)(pcap_t *);    /* 中断循环操作函数指针 */
#ifdef _WIN32
typedef struct pcap_stat *(*stats_ex_op_t)(pcap_t *, int *);    /* 扩展统计操作函数指针 */
typedef int    (*setbuff_op_t)(pcap_t *, int);    /* 设置缓冲区操作函数指针 */
typedef int    (*setmode_op_t)(pcap_t *, int);    /* 设置模式操作函数指针 */
typedef int    (*setmintocopy_op_t)(pcap_t *, int);    /* 设置最小拷贝操作函数指针 */
typedef HANDLE    (*getevent_op_t)(pcap_t *);    /* 获取事件操作函数指针 */
typedef int    (*oid_get_request_op_t)(pcap_t *, bpf_u_int32, void *, size_t *);    /* OID 获取请求操作函数指针 */
typedef int    (*oid_set_request_op_t)(pcap_t *, bpf_u_int32, const void *, size_t *);    /* OID 设置请求操作函数指针 */
typedef u_int    (*sendqueue_transmit_op_t)(pcap_t *, pcap_send_queue *, int);    /* 发送队列传输操作函数指针 */
typedef int    (*setuserbuffer_op_t)(pcap_t *, int);    /* 设置用户缓冲区操作函数指针 */
typedef int    (*live_dump_op_t)(pcap_t *, char *, int, int);    /* 实时转储操作函数指针 */
typedef int    (*live_dump_ended_op_t)(pcap_t *, int);    /* 实时转储结束操作函数指针 */
typedef PAirpcapHandle    (*get_airpcap_handle_op_t)(pcap_t *);    /* 获取 Airpcap 句柄操作函数指针 */
#endif
typedef void    (*cleanup_op_t)(pcap_t *);    /* 清理操作函数指针 */

/*
 * 我们将所有在读取代码路径中使用的内容放在开头，
 * 以尝试将它们保持在同一个缓存行或多个缓存行中。
 */
struct pcap {
    /*
     * 调用以在实时捕获中读取数据包的方法。
     */
    read_op_t read_op;    /* 读取操作函数指针 */
    /*
     * 从保存文件中读取下一个数据包的方法
     */
    next_packet_op_t next_packet_op;
#ifdef _WIN32
    HANDLE handle;  // 如果是 Windows 系统，定义一个句柄变量
#else
    int fd;  // 如果不是 Windows 系统，定义一个文件描述符变量
#endif /* _WIN32 */

    /*
     * Read buffer.
     */
    u_int bufsize;  // 读取缓冲区大小
    void *buffer;  // 缓冲区指针
    u_char *bp;  // 字节指针
    int cc;  // 字节计数

    sig_atomic_t break_loop; /* flag set to force break from packet-reading loop */  // 用于强制中断数据包读取循环的标志

    void *priv;        /* private data for methods */  // 方法的私有数据

#ifdef ENABLE_REMOTE
    struct pcap_samp rmt_samp;    /* parameters related to the sampling process. */  // 与采样过程相关的参数
#endif

    int swapped;  // 是否进行了字节交换
    FILE *rfile;        /* null if live capture, non-null if savefile */  // 如果是实时捕获则为空，如果是保存文件则非空
    u_int fddipad;  // FDDI 填充
    struct pcap *next;    /* list of open pcaps that need stuff cleared on close */  // 需要在关闭时清除的打开 pcap 列表

    /*
     * File version number; meaningful only for a savefile, but we
     * keep it here so that apps that (mistakenly) ask for the
     * version numbers will get the same zero values that they
     * always did.
     */
    int version_major;  // 文件主版本号，仅对保存文件有意义
    int version_minor;  // 文件次版本号，仅对保存文件有意义

    int snapshot;  // 快照长度
    int linktype;        /* Network linktype */  // 网络链路类型
    int linktype_ext;    /* Extended information stored in the linktype field of a file */  // 存储在文件的链路类型字段中的扩展信息
    int offset;        /* offset for proper alignment */  // 适当对齐的偏移量
    int activated;        /* true if the capture is really started */  // 如果捕获真正开始，则为 true
    int oldstyle;        /* if we're opening with pcap_open_live() */  // 如果使用 pcap_open_live() 打开，则为 true

    struct pcap_opt opt;  // pcap 选项

    /*
     * Place holder for pcap_next().
     */
    u_char *pkt;  // pcap_next() 的占位符

#ifdef _WIN32
    struct pcap_stat stat;    /* used for pcap_stats_ex() */  // 用于 pcap_stats_ex() 的结构体
#endif

    /* We're accepting only packets in this direction/these directions. */
    pcap_direction_t direction;  // 只接受这个方向/这些方向的数据包

    /*
     * Flags to affect BPF code generation.
     */
    int bpf_codegen_flags;  // 影响 BPF 代码生成的标志

#if !defined(_WIN32) && !defined(MSDOS)
    int selectable_fd;    /* FD on which select()/poll()/epoll_wait()/kevent()/etc. can be done */  // 可以进行 select()/poll()/epoll_wait()/kevent()/等操作的文件描述符
    /*
     * 如果没有可选择的文件描述符，或者有但不一定有效（例如，如果在缓冲区填满之前不会被通知数据包捕获超时到期），则应该使用此指向的超时时间
     * 在 select()/poll()/epoll_wait()/kevent() 调用中。 pcap_t 应该被设置为非阻塞模式，如果调用超时，则应尝试使用所需的超时从所有 pcap_t 中读取数据包，并且代码必须准备好在尝试中不看到任何数据包。
     */
    const struct timeval *required_select_timeout;
#endif

    /*
     * 如果内核中没有BPF，则用于过滤代码的占位符。
     */
    struct bpf_program fcode;

    char errbuf[PCAP_ERRBUF_SIZE + 1];
#ifdef _WIN32
    char acp_errbuf[PCAP_ERRBUF_SIZE + 1];    /* 用于本地代码页错误字符串的缓冲区 */
#endif
    int dlt_count;
    u_int *dlt_list;
    int tstamp_type_count;
    u_int *tstamp_type_list;
    int tstamp_precision_count;
    u_int *tstamp_precision_list;

    struct pcap_pkthdr pcap_header;    /* 这对于pcap_next_ex()函数是必需的 */

    /*
     * 更多方法。
     */
    activate_op_t activate_op;
    can_set_rfmon_op_t can_set_rfmon_op;
    inject_op_t inject_op;
    save_current_filter_op_t save_current_filter_op;
    setfilter_op_t setfilter_op;
    setdirection_op_t setdirection_op;
    set_datalink_op_t set_datalink_op;
    getnonblock_op_t getnonblock_op;
    setnonblock_op_t setnonblock_op;
    stats_op_t stats_op;
    breakloop_op_t breakloop_op;

    /*
     * 用作pcap_next()/pcap_next_ex()回调的例程。
     */
    pcap_handler oneshot_callback;

#ifdef _WIN32
    /*
     * 这些至少目前是特定于Win32 NPF驱动程序的。
     */
    stats_ex_op_t stats_ex_op;
    setbuff_op_t setbuff_op;
    setmode_op_t setmode_op;
    setmintocopy_op_t setmintocopy_op;
    getevent_op_t getevent_op;
    oid_get_request_op_t oid_get_request_op;
    oid_set_request_op_t oid_set_request_op;
    sendqueue_transmit_op_t sendqueue_transmit_op;
    setuserbuffer_op_t setuserbuffer_op;
    live_dump_op_t live_dump_op;
    live_dump_ended_op_t live_dump_ended_op;
    get_airpcap_handle_op_t get_airpcap_handle_op;
#endif
    cleanup_op_t cleanup_op;
};

/*
 * BPF代码生成标志。
 */
#define BPF_SPECIAL_VLAN_HANDLING    0x00000001    /* Linux的特殊VLAN处理 */
/*
 * 这是保存在保存文件中的时间戳。
 * 它必须在任何地方使用相同的类型，与实际的 `struct timeval' 无关；
 * `struct timeval' 在某些平台上具有 32 位的 tv_sec 值，在其他平台上具有 64 位的 tv_sec 值，
 * 并且写出本地 `struct timeval' 值意味着文件只能在具有与写入文件的系统相同的 tv_sec 大小的系统上读取。
 */

struct pcap_timeval {
    bpf_int32 tv_sec;        /* 秒 */
    bpf_int32 tv_usec;        /* 微秒 */
};

/*
 * 这是实际保存在保存文件中的 `pcap_pkthdr'。
 *
 * 不要以任何方式更改此结构的格式（这包括仅影响此结构中字段长度的更改），
 * 并且不要使时间戳除了秒和微秒之外的任何其他东西（例如，秒和纳秒）。
 * 而是：
 *
 *    为新格式引入一个新的结构；
 *
 *    发送邮件到 "tcpdump-workers@lists.tcpdump.org"，请求新的魔术数字用于新的捕获文件格式，
 *    并且在获得新的魔术数字时，将其放入 "savefile.c"；
 *
 *    对具有更改记录头的保存文件使用该魔术数字；
 *
 *    使 "savefile.c" 中的代码能够读取具有旧记录头以及具有新记录头的文件（使用魔术数字确定头格式）。
 *
 * 然后通过分叉分支提供更改
 *
 *    https://github.com/the-tcpdump-group/libpcap/tree/master
 *
 * 并发出拉取请求，以便 libpcap 的未来版本和使用它的程序（如 tcpdump）能够读取您的新捕获文件格式。
 */

struct pcap_sf_pkthdr {
    struct pcap_timeval ts;    /* 时间戳 */
    bpf_u_int32 caplen;        /* 存在部分的长度 */
    bpf_u_int32 len;        /* 此数据包的长度（从网络上） */
};
/*
 * 定义一个结构体，用于存储被一些修补版本的 libpcap（例如 Red Hat Linux 6.1 和 6.2 中的版本）写入的保存文件中的 `pcap_pkthdr`。
 * 不要以任何方式改变此结构的格式（包括仅影响此结构中字段长度的更改）。而是按照上述，引入一个新的结构。
 */
struct pcap_sf_patched_pkthdr {
    struct pcap_timeval ts;    /* 时间戳 */
    bpf_u_int32 caplen;        /* 存在部分的长度 */
    bpf_u_int32 len;           /* 此数据包的长度（从网络上） */
    int        index;
    unsigned short protocol;
    unsigned char pkt_type;
};

/*
 * 用于 pcap_next() 和 pcap_next_ex() 的一次性回调的用户数据结构。
 */
struct oneshot_userdata {
    struct pcap_pkthdr *hdr;
    const u_char **pkt;
    pcap_t *pd;
};

#ifndef min
#define min(a, b) ((a) > (b) ? (b) : (a))
#endif

/*
 * 用于 pcap_offline_read() 的 pcap 实现的非阻塞模式的常用例程。
 */
int    pcap_offline_read(pcap_t *, int, pcap_handler, u_char *);

/*
 * 模块的读取例程的数据包计数参数是否表示“提供数据包直到没有数据包为止”？
 */
#define PACKET_COUNT_IS_UNLIMITED(count)    ((count) <= 0)

/*
 * 大多数 pcap 实现可以用于非阻塞模式的内部接口。
 */
#if !defined(_WIN32) && !defined(MSDOS)
int    pcap_getnonblock_fd(pcap_t *);
int    pcap_setnonblock_fd(pcap_t *p, int);
#endif

/*
 * “pcap_create()” 的内部接口。
 *
 * “pcap_create_interface()” 是在常规网络接口上执行 pcap_create 的例程。有多个实现，每个平台类型（Linux、BPF、DLPI 等）都有一个，由配置脚本选择使用的实现。
 *
 * “pcap_create_common()” 分配并填充一个用于 pcap_create 例程的 pcap_t。
 */
pcap_t    *pcap_create_interface(const char *, char *);
/*
 * 定义一个宏，用于创建一个通用的 pcap_t 结构体
 * 该宏接受一个错误缓冲区指针和一个用于私有数据的类型，并调用 pcap_create_common() 函数
 * 传递错误缓冲区指针、私有数据类型的大小（以字节为单位）和私有数据在结构体中的偏移量（以字节为单位）
 */
#define PCAP_CREATE_COMMON(ebuf, type) \
    pcap_create_common(ebuf, \
        sizeof (struct { pcap_t __common; type __private; }), \
        offsetof (struct { pcap_t __common; type __private; }, __private))
// 声明一个指向 pcap_t 结构体的指针
pcap_t    *pcap_create_common(char *, size_t, size_t);
// 添加退出处理程序到 pcap_t 结构体
int    pcap_do_addexit(pcap_t *);
// 将 pcap_t 结构体添加到待关闭的列表中
void    pcap_add_to_pcaps_to_close(pcap_t *);
// 从待关闭的列表中移除 pcap_t 结构体
void    pcap_remove_from_pcaps_to_close(pcap_t *);
// 清理 pcap_t 结构体的公共部分
void    pcap_cleanup_live_common(pcap_t *);
// 检查 pcap_t 结构体是否已激活
int    pcap_check_activated(pcap_t *);
// 中断 pcap 循环
void    pcap_breakloop_common(pcap_t *);

/*
 * "pcap_findalldevs()" 的内部接口
 *
 * pcap_if_list_t * 是一个指向设备列表的引用
 *
 * get_if_flags_func 是一个平台相关的函数，用于获取额外的接口标志
 *
 * "pcap_platform_finddevs()" 是用于查找本地网络接口的平台相关例程
 *
 * "pcap_findalldevs_interfaces()" 是一个辅助函数，用于使用“标准”机制（SIOCGIFCONF、“getifaddrs()”等）查找这些接口
 *
 * "add_dev()" 向 pcap_if_list_t 添加一个条目
 *
 * "find_dev()" 尝试在 pcap_if_list_t 中按名称查找设备
 *
 * "find_or_add_dev()" 检查设备是否已经在 pcap_if_list_t 中，如果没有，则添加一个条目
 */
struct pcap_if_list;
typedef struct pcap_if_list pcap_if_list_t;
typedef int (*get_if_flags_func)(const char *, bpf_u_int32 *, char *);
// 在非 Windows 平台和非 MSDOS 平台上查找网络接口
int    pcap_platform_finddevs(pcap_if_list_t *, char *);
#if !defined(_WIN32) && !defined(MSDOS)
int    pcap_findalldevs_interfaces(pcap_if_list_t *, char *,
        int (*)(const char *), get_if_flags_func);
#endif
// 在给定的设备列表中查找或添加设备，如果设备不存在则添加，存在则返回该设备
pcap_if_t *find_or_add_dev(pcap_if_list_t *, const char *, bpf_u_int32,
        get_if_flags_func, const char *, char *);
// 在给定的设备列表中查找设备，如果存在则返回该设备
pcap_if_t *find_dev(pcap_if_list_t *, const char *);
// 向给定的设备列表中添加设备
pcap_if_t *add_dev(pcap_if_list_t *, const char *, bpf_u_int32, const char *,
        char *);
// 向给定的设备添加地址
int    add_addr_to_dev(pcap_if_t *, struct sockaddr *, size_t,
        struct sockaddr *, size_t, struct sockaddr *, size_t,
        struct sockaddr *dstaddr, size_t, char *errbuf);
// 在非 Windows 系统下，查找或添加接口到设备列表中
pcap_if_t *find_or_add_if(pcap_if_list_t *, const char *, bpf_u_int32,
        get_if_flags_func, char *);
// 向接口列表中添加地址
int    add_addr_to_if(pcap_if_list_t *, const char *, bpf_u_int32,
        get_if_flags_func,
        struct sockaddr *, size_t, struct sockaddr *, size_t,
        struct sockaddr *, size_t, struct sockaddr *, size_t, char *);

/*
 * "pcap_open_offline_common()"分配并填充一个 pcap_t 结构，用于 pcap_open_offline 函数使用。
 *
 * "pcap_adjust_snapshot()" 调整快照长度为非零并适合 int 类型。
 *
 * "sf_cleanup()" 关闭与 pcap_t 关联的文件句柄，如果适用的话，并释放所有处理保存文件类型的模块的公共数据。
 *
 * "charset_fopen()" 在 Windows 的 UTF-8 模式下，使用 treat the pathname as being in UTF-8 的 fopen()，而不是本地代码页。
 */

/*
 * 这个包装器接受一个错误缓冲区指针和一个用于私有数据的类型，并调用 pcap_create_common()，传递错误缓冲区指针、私有数据类型的大小（以字节为单位）以及私有数据从结构开始的偏移量（以字节为单位）。
 */
#define PCAP_OPEN_OFFLINE_COMMON(ebuf, type) \
    pcap_open_offline_common(ebuf, \
        sizeof (struct { pcap_t __common; type __private; }), \
        offsetof (struct { pcap_t __common; type __private; }, __private))
/*
 * 打开离线捕获文件，返回 pcap_t 结构体指针
 */
pcap_t *pcap_open_offline_common(char *ebuf, size_t total_size, size_t private_data);

/*
 * 调整快照长度，返回调整后的快照长度
 */
bpf_u_int32 pcap_adjust_snapshot(bpf_u_int32 linktype, bpf_u_int32 snaplen);

/*
 * 清理 pcap_t 结构体指针
 */
void sf_cleanup(pcap_t *p);

#ifdef _WIN32
/*
 * 在 Windows 平台上，使用特定字符集打开文件
 */
FILE *charset_fopen(const char *path, const char *mode);
#else
/*
 * 在其他操作系统上，使用标准的 fopen() 函数
 */
#define charset_fopen(path, mode) fopen((path), (mode))
#endif

/*
 * 在运行时加载代码的内部接口
 */
#ifdef _WIN32
#define pcap_code_handle_t HMODULE
#define pcap_funcptr_t FARPROC

pcap_code_handle_t pcap_load_code(const char *);
pcap_funcptr_t pcap_find_function(pcap_code_handle_t, const char *);
#endif

/*
 * 用户模式过滤数据包和验证过滤程序的内部接口
 */
/*
 * 辅助数据结构，用于解释 Linux 内核拒绝的过滤器
 */
struct pcap_bpf_aux_data {
    u_short vlan_tag_present;
    u_short vlan_tag;
};

/*
 * 带有辅助数据的过滤函数
 */
u_int pcap_filter_with_aux_data(const struct bpf_insn *, const u_char *, u_int, u_int, const struct pcap_bpf_aux_data *);

/*
 * 不带辅助数据的过滤函数
 */
u_int pcap_filter(const struct bpf_insn *, const u_char *, u_int, u_int);

/*
 * 验证 BPF 程序的函数
 */
int pcap_validate_filter(const struct bpf_insn *, int);

/*
 * "pcap_create()" 和打开保存文件的内部接口
 *
 * "pcap_oneshot()" 是 "pcap_next()" 和 "pcap_next_ex()" 的标准一次性回调函数
 */
void pcap_oneshot(u_char *, const struct pcap_pkthdr *, const u_char *);

int install_bpf_program(pcap_t *, struct bpf_program *);

int pcap_strcasecmp(const char *, const char *);
/*
 * pcap_createsrcstr_ex函数的内部接口，用于创建包含SSL支持信息的源字符串(rpcap:// vs. rpcaps://)
 */
int    pcap_createsrcstr_ex(char *, int, const char *, const char *,
    const char *, unsigned char, char *);
/*
 * pcap_parsesrcstr_ex函数的内部接口，用于解析包含SSL支持信息的源字符串
 */
int    pcap_parsesrcstr_ex(const char *, int *, char *, char *,
    char *, unsigned char *, char *);

#ifdef YYDEBUG
extern int pcap_debug;
#endif

#ifdef __cplusplus
}
#endif

#endif
```