# `nmap\libpcap\pcap\pcap.h`

```cpp
/*
 * 设置 C 语言的编辑模式和缩进格式
 * 版权声明
 * 允许在源代码和二进制形式下进行再发布和使用
 * 保留源代码的版权声明、条件列表和下面的免责声明
 * 在二进制形式下再发布时，在文档和/或其他提供的材料中复制版权声明、条件列表和下面的免责声明
 * 所有提到此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 * 此软件由权利人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何责任理论下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式），都不会因使用本软件而产生，即使已被告知可能发生此类损害
 */
/*
 * 远程数据包捕获机制和WinPcap的扩展
 * 版权所有 (c) 2002 - 2003
 * NetGroup, 意大利都灵理工大学
 * 保留所有权利
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 *
 * 1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 * 2. 以二进制形式再分发必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 * 3. 不能使用都灵理工大学的名称或其贡献者的名称，未经特定事先书面许可，来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权行为 (包括疏忽或其他方式) 的任何理论下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害 (包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断) 负责，即使已被告知可能发生此类损害。
 *
 */
#ifndef lib_pcap_pcap_h
#define lib_pcap_pcap_h
/*
 * 检测是否定义了 _MSC_VER，如果定义了，则尝试取消定义，以便可靠地使用它来知道使用的是哪个编译器，如果是 Visual Studio，则知道使用的是哪个版本。
 */
#if defined(_MSC_VER)
  /*
   * 我们假设软件如 lwIP 在包含 pcap.h 之前会定义 _MSC_VER，并将其定义为 1500。
   * 
   * 尝试检测这一点，并取消定义 _MSC_VER，以便我们可以可靠地使用它来知道使用的是哪个编译器，如果是 Visual Studio，则知道使用的是哪个版本。
   */
  #if !defined(_MSC_FULL_VER)
    /*
     * 根据 https://sourceforge.net/p/predef/wiki/Compilers/ 的描述，"Visual C++ 6.0 Processor Pack"/Visual C++ 6.0 SP6 和之后的版本会定义 _MSC_FULL_VER，因此要么这是一个较旧的版本的 Visual C++，要么根本不是 Visual C++。
     * 
     * 对于 Visual C++ 6.0，_MSC_VER 被定义为 1200。
     */
    #if _MSC_VER > 1200
      /*
       * 如果这是 Visual C++，_MSC_FULL_VER 应该被定义，因此我们假设这不是 Visual C++，并取消定义它是 Visual C++ 的谎言。
       */
      #undef _MSC_VER
    #endif
  #endif
#endif

#include <pcap/funcattrs.h>

#include <pcap/pcap-inttypes.h>

#if defined(_WIN32)
  #include <winsock2.h>        /* u_int, u_char etc. */
  #include <io.h>        /* _get_osfhandle() */
#elif defined(MSDOS)
  #include <sys/types.h>    /* u_int, u_char etc. */
  #include <sys/socket.h>
#else /* UN*X */
  #include <sys/types.h>    /* u_int, u_char etc. */
  #include <sys/time.h>
#endif /* _WIN32/MSDOS/UN*X */

#include <pcap/socket.h>    /* for SOCKET, as the active-mode rpcap APIs use it */

#ifndef PCAP_DONT_INCLUDE_PCAP_BPF_H
#include <pcap/bpf.h>
#endif

#include <stdio.h>

#ifdef __cplusplus
extern "C" {
#endif

/*
 * 当前 pcap 文件格式的版本号。
 *
 * 注意：这不是 libpcap 库的版本号。
 * 要获取你正在使用的 libpcap 版本信息，请使用 pcap_lib_version()。
 */
#define PCAP_VERSION_MAJOR 2
#define PCAP_VERSION_MINOR 4

#define PCAP_ERRBUF_SIZE 256

/*
 * 为那些在 bpf.h 中有 64 位支持之前的系统提供兼容性。
 */
#if BPF_RELEASE - 0 < 199406
typedef    int bpf_int32;
typedef    u_int bpf_u_int32;
#endif

typedef struct pcap pcap_t;
typedef struct pcap_dumper pcap_dumper_t;
typedef struct pcap_if pcap_if_t;
typedef struct pcap_addr pcap_addr_t;
/*
 * 该文件中的第一个记录包含了在tcpdump打印阶段中使用的一些标志的保存值。
 * 这里的许多字段都是32位整数，因此编译器不会插入不需要的填充；这些文件需要在不同架构之间互换。
 * 文档：https://www.tcpdump.org/manpages/pcap-savefile.5.txt。
 *
 * 不要以任何方式改变此结构的布局（这包括仅影响此结构中字段长度的更改）。
 *
 * 同样，不要以任何方式改变此结构的任何成员的解释（这包括在“savefile.c”中定义的LINKTYPE_值以外的值在“linktype”字段中使用）。
 *
 * 而是：
 *
 *    如果结构的布局发生了变化，请引入一个新的结构用于新格式；
 *
 *    发送邮件至“tcpdump-workers@lists.tcpdump.org”，请求一个新的魔术数字用于新的捕获文件格式，并在获得新的魔术数字后将其放入“savefile.c”；
 *
 *    对具有更改的文件头的保存文件使用该魔术数字；
 *
 *    使“savefile.c”中的代码能够读取具有旧文件头以及具有新文件头的文件（使用魔术数字确定头格式）。
 *
 * 然后通过分叉分支提供更改
 *
 *    https://github.com/the-tcpdump-group/libpcap/tree/master
 *
 * 并发出拉取请求，以便libpcap的未来版本和使用它的程序（如tcpdump）能够读取您的新捕获文件格式。
 */
struct pcap_file_header {
    bpf_u_int32 magic;       // 用于标识文件格式的魔术数字
    u_short version_major;   // 主版本号
    u_short version_minor;   // 次版本号
    bpf_int32 thiszone;      // gmt到本地时间的修正；这总是0
    bpf_u_int32 sigfigs;     // 时间戳的精度；这总是0
    bpf_u_int32 snaplen;     // 每个数据包保存的最大长度
    bpf_u_int32 linktype;    // 数据链路类型（LINKTYPE_*）
};
/*
 * 定义宏，用于 pcap_datalink_ext() 返回的值。
 *
 * 如果 LT_FCS_LENGTH_PRESENT(x) 为真，则 LT_FCS_LENGTH(x) 宏给出捕获数据包的 FCS 长度。
 */
#define LT_FCS_LENGTH_PRESENT(x)    ((x) & 0x04000000)
#define LT_FCS_LENGTH(x)        (((x) & 0xF0000000) >> 28)
#define LT_FCS_DATALINK_EXT(x)        ((((x) & 0xF) << 28) | 0x04000000)

typedef enum {
       PCAP_D_INOUT = 0,  // 数据包的方向为输入输出
       PCAP_D_IN,         // 数据包的方向为输入
       PCAP_D_OUT         // 数据包的方向为输出
} pcap_direction_t;

/*
 * 由 libpcap 提供的通用每个数据包的信息。
 *
 * 时间戳可以且应该是 "struct timeval"，无论您的系统是否支持 "struct timeval" 中的 32 位 tv_sec，64 位 tv_sec，或者如果它同时支持 32 位和 64 位应用程序。保存文件的磁盘格式使用 32 位 tv_sec（和 tv_usec）；这个结构对此无关紧要。即使它们在同一平台上，32 位和 64 位版本的 libpcap 应该提供适当版本的 "struct timeval"，即使底层数据包捕获机制提供的不是这样的结构。
 */
struct pcap_pkthdr {
    struct timeval ts;    /* 时间戳 */
    bpf_u_int32 caplen;    /* 存在部分的长度 */
    bpf_u_int32 len;    /* 此数据包的长度（从网络上） */
};

/*
 * 由 pcap_stats() 返回的结构
 */
struct pcap_stat {
    u_int ps_recv;        /* 接收到的数据包数 */
    u_int ps_drop;        /* 丢弃的数据包数 */
    u_int ps_ifdrop;    /* 由接口丢弃的数据包数 -- 仅在某些平台上支持 */
#ifdef _WIN32
    u_int ps_capt;        /* 到达应用程序的数据包数 */
    u_int ps_sent;        /* 服务器在网络上发送的数据包数 */
    u_int ps_netdrop;    /* 在网络上丢失的数据包数 */
#endif /* _WIN32 */
};

#ifdef MSDOS
/*
 * 由 pcap_stats_ex() 返回的结构
 */
# 定义结构体，用于存储网络接口的统计信息
struct pcap_stat_ex {
       u_long  rx_packets;        /* total packets received       */
       u_long  tx_packets;        /* total packets transmitted    */
       u_long  rx_bytes;          /* total bytes received         */
       u_long  tx_bytes;          /* total bytes transmitted      */
       u_long  rx_errors;         /* bad packets received         */
       u_long  tx_errors;         /* packet transmit problems     */
       u_long  rx_dropped;        /* no space in Rx buffers       */
       u_long  tx_dropped;        /* no space available for Tx    */
       u_long  multicast;         /* multicast packets received   */
       u_long  collisions;

       /* detailed rx_errors: */
       u_long  rx_length_errors;
       u_long  rx_over_errors;    /* receiver ring buff overflow  */
       u_long  rx_crc_errors;     /* recv'd pkt with crc error    */
       u_long  rx_frame_errors;   /* recv'd frame alignment error */
       u_long  rx_fifo_errors;    /* recv'r fifo overrun          */
       u_long  rx_missed_errors;  /* recv'r missed packet         */

       /* detailed tx_errors */
       u_long  tx_aborted_errors;
       u_long  tx_carrier_errors;
       u_long  tx_fifo_errors;
       u_long  tx_heartbeat_errors;
       u_long  tx_window_errors;
     };
#endif

/*
 * 网络接口列表中的项目
 */
struct pcap_if {
    struct pcap_if *next;
    char *name;        /* 传递给 "pcap_open_live()" 的名称 */
    char *description;    /* 接口的文本描述，或者为 NULL */
    struct pcap_addr *addresses;
    bpf_u_int32 flags;    /* PCAP_IF_ 接口标志 */
};

#define PCAP_IF_LOOPBACK                0x00000001    /* 接口是环回接口 */
#define PCAP_IF_UP                    0x00000002    /* 接口已启用 */
#define PCAP_IF_RUNNING                    0x00000004    /* 接口正在运行 */
#define PCAP_IF_WIRELESS                0x00000008    /* 接口是无线接口（*不一定是 Wi-Fi！） */
# 定义网络接口连接状态的常量
#define PCAP_IF_CONNECTION_STATUS            0x00000030    /* connection status: */
#define PCAP_IF_CONNECTION_STATUS_UNKNOWN        0x00000000    /* unknown */
#define PCAP_IF_CONNECTION_STATUS_CONNECTED        0x00000010    /* connected */
#define PCAP_IF_CONNECTION_STATUS_DISCONNECTED        0x00000020    /* disconnected */
#define PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE    0x00000030    /* not applicable */

/*
 * 表示网络接口地址的结构体
 */
struct pcap_addr {
    struct pcap_addr *next;  /* 下一个地址结构体的指针 */
    struct sockaddr *addr;        /* 地址 */
    struct sockaddr *netmask;    /* 该地址的子网掩码 */
    struct sockaddr *broadaddr;    /* 该地址的广播地址 */
    struct sockaddr *dstaddr;    /* 该地址的点对点目标地址 */
};

typedef void (*pcap_handler)(u_char *, const struct pcap_pkthdr *,
                 const u_char *);

/*
 * pcap API 的错误代码
 * 这些代码都是负数，因此可以通过检查负值来判断调用是否成功或失败
 */
#define PCAP_ERROR            -1    /* 通用错误代码 */
#define PCAP_ERROR_BREAK        -2    /* 由 pcap_breakloop 终止的循环 */
#define PCAP_ERROR_NOT_ACTIVATED    -3    /* 捕获需要被激活 */
#define PCAP_ERROR_ACTIVATED        -4    /* 无法对已激活的捕获执行操作 */
#define PCAP_ERROR_NO_SUCH_DEVICE    -5    /* 不存在该设备 */
#define PCAP_ERROR_RFMON_NOTSUP        -6    /* 该设备不支持 rfmon（监视）模式 */
#define PCAP_ERROR_NOT_RFMON        -7    /* 仅在监视模式下支持的操作 */
#define PCAP_ERROR_PERM_DENIED        -8    /* 没有权限打开设备 */
#define PCAP_ERROR_IFACE_NOT_UP        -9    /* 接口未启动 */
#define PCAP_ERROR_CANTSET_TSTAMP_TYPE    -10    /* 该设备不支持设置时间戳类型 */
#define PCAP_ERROR_PROMISC_PERM_DENIED    -11    /* you don't have permission to capture in promiscuous mode */
#define PCAP_ERROR_TSTAMP_PRECISION_NOTSUP -12  /* the requested time stamp precision is not supported */

/*
 * Warning codes for the pcap API.
 * These will all be positive and non-zero, so they won't look like
 * errors.
 */
#define PCAP_WARNING            1    /* generic warning code */
#define PCAP_WARNING_PROMISC_NOTSUP    2    /* this device doesn't support promiscuous mode */
#define PCAP_WARNING_TSTAMP_TYPE_NOTSUP    3    /* the requested time stamp type is not supported */

/*
 * Value to pass to pcap_compile() as the netmask if you don't know what
 * the netmask is.
 */
#define PCAP_NETMASK_UNKNOWN    0xffffffff

/*
 * Initialize pcap.  If this isn't called, pcap is initialized to
 * a mode source-compatible and binary-compatible with older versions
 * that lack this routine.
 */

/*
 * Initialization options.
 * All bits not listed here are reserved for expansion.
 *
 * On UNIX-like systems, the local character encoding is assumed to be
 * UTF-8, so no character encoding transformations are done.
 *
 * On Windows, the local character encoding is the local ANSI code page.
 */
#define PCAP_CHAR_ENC_LOCAL    0x00000000U    /* strings are in the local character encoding */
#define PCAP_CHAR_ENC_UTF_8    0x00000001U    /* strings are in UTF-8 */

PCAP_AVAILABLE_1_10
PCAP_API int    pcap_init(unsigned int, char *);

/*
 * We're deprecating pcap_lookupdev() for various reasons (not
 * thread-safe, can behave weirdly with WinPcap).  Callers
 * should use pcap_findalldevs() and use the first device.
 */
PCAP_AVAILABLE_0_4
PCAP_DEPRECATED("use 'pcap_findalldevs' and use the first device")
PCAP_API char    *pcap_lookupdev(char *);

PCAP_AVAILABLE_0_4
PCAP_API int    pcap_lookupnet(const char *, bpf_u_int32 *, bpf_u_int32 *, char *);

PCAP_AVAILABLE_1_0
PCAP_API pcap_t    *pcap_create(const char *, char *);

PCAP_AVAILABLE_1_0
# 设置捕获数据包时的快照长度
PCAP_API int    pcap_set_snaplen(pcap_t *, int);

# 设置是否开启混杂模式
PCAP_API int    pcap_set_promisc(pcap_t *, int);

# 检查是否可以设置 RFMON 模式
PCAP_API int    pcap_can_set_rfmon(pcap_t *);

# 设置 RFMON 模式
PCAP_API int    pcap_set_rfmon(pcap_t *, int);

# 设置超时时间
PCAP_API int    pcap_set_timeout(pcap_t *, int);

# 设置时间戳类型
PCAP_API int    pcap_set_tstamp_type(pcap_t *, int);

# 设置立即模式
PCAP_API int    pcap_set_immediate_mode(pcap_t *, int);

# 设置缓冲区大小
PCAP_API int    pcap_set_buffer_size(pcap_t *, int);

# 设置时间戳精度
PCAP_API int    pcap_set_tstamp_precision(pcap_t *, int);

# 获取时间戳精度
PCAP_API int    pcap_get_tstamp_precision(pcap_t *);

# 激活数据包捕获
PCAP_API int    pcap_activate(pcap_t *);

# 列出时间戳类型
PCAP_API int    pcap_list_tstamp_types(pcap_t *, int **);

# 释放时间戳类型列表
PCAP_API void    pcap_free_tstamp_types(int *);

# 将时间戳类型名称转换为值
PCAP_API int    pcap_tstamp_type_name_to_val(const char *);

# 将时间戳类型值转换为名称
PCAP_API const char *pcap_tstamp_type_val_to_name(int);

# 将时间戳类型值转换为描述
PCAP_API const char *pcap_tstamp_type_val_to_description(int);

#ifdef __linux__
# 设置 Linux 网络接口的协议
PCAP_API int    pcap_set_protocol_linux(pcap_t *, int);
#endif

# 时间戳类型常量定义
#define PCAP_TSTAMP_HOST            0    /* 主机提供，未知特性 */
#define PCAP_TSTAMP_HOST_LOWPREC        1    /* 主机提供，低精度，与系统时钟同步 */
#define PCAP_TSTAMP_HOST_HIPREC            2    /* 主机提供，高精度，与系统时钟同步 */
#define PCAP_TSTAMP_ADAPTER            3    /* 设备提供，与系统时钟同步 */
#define PCAP_TSTAMP_ADAPTER_UNSYNCED        4    /* 设备提供，未与系统时钟同步 */
#define PCAP_TSTAMP_HOST_HIPREC_UNSYNCED    5    /* 主机提供，高精度，未与系统时钟同步 */
/*
 * 时间戳分辨率类型。
 * 并非所有系统和接口在进行实时捕获时都一定支持所有这些分辨率；
 * 在读取保存文件时可以请求所有这些分辨率。
 */
#define PCAP_TSTAMP_PRECISION_MICRO    0    /* 使用微秒精度时间戳，默认 */
#define PCAP_TSTAMP_PRECISION_NANO    1    /* 使用纳秒精度时间戳 */

PCAP_AVAILABLE_0_4
PCAP_API pcap_t    *pcap_open_live(const char *, int, int, int, char *);

PCAP_AVAILABLE_0_6
PCAP_API pcap_t    *pcap_open_dead(int, int);

PCAP_AVAILABLE_1_5
PCAP_API pcap_t    *pcap_open_dead_with_tstamp_precision(int, int, u_int);

PCAP_AVAILABLE_1_5
PCAP_API pcap_t    *pcap_open_offline_with_tstamp_precision(const char *, u_int, char *);

PCAP_AVAILABLE_0_4
PCAP_API pcap_t    *pcap_open_offline(const char *, char *);

#ifdef _WIN32
  PCAP_AVAILABLE_1_5
  PCAP_API pcap_t  *pcap_hopen_offline_with_tstamp_precision(intptr_t, u_int, char *);

  PCAP_API pcap_t  *pcap_hopen_offline(intptr_t, char *);
  /*
   * 如果我们正在构建 libpcap，这些是 savefile.c 中的内部例程，
   * 因此我们不能将它们定义为宏。
   *
   * 如果我们没有构建 libpcap，鉴于 libpcap 构建时所使用的 C 运行时版本
   * 可能与使用 libpcap 的应用程序构建时所使用的 C 运行时版本不同，
   * 并且 FILE 结构可能在这两个 C 运行时版本之间有所不同，
   * 对 _fileno() 的调用必须使用打开 FILE * 的 C 运行时版本中的 _fileno()，
   * 而不是使用 libpcap 构建时的 C 运行时版本中的 _fileno()。（也许一旦 Universal CRT 统治世界，这个问题就会消失。）
   */
  #ifndef BUILDING_PCAP
    #define pcap_fopen_offline_with_tstamp_precision(f,p,b) \
    pcap_hopen_offline_with_tstamp_precision(_get_osfhandle(_fileno(f)), p, b)
    #define pcap_fopen_offline(f,b) \
    pcap_hopen_offline(_get_osfhandle(_fileno(f)), b)
  #endif
#else /*_WIN32*/
  #ifdef _WIN32 指令的反义，表示不是 Windows 系统
  PCAP_AVAILABLE_1_5
  #if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 1 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 5 的条件编译指令
  PCAP_API pcap_t    *pcap_fopen_offline_with_tstamp_precision(FILE *, u_int, char *);
  #endif
  PCAP_AVAILABLE_0_9
  #if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 9 的条件编译指令
  PCAP_API pcap_t    *pcap_fopen_offline(FILE *, char *);
  #endif
#endif /*_WIN32*/
  #endif /*_WIN32*/ 结束条件编译指令

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API void    pcap_close(pcap_t *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API int    pcap_loop(pcap_t *, int, pcap_handler, u_char *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API int    pcap_dispatch(pcap_t *, int, pcap_handler, u_char *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API const u_char *pcap_next(pcap_t *, struct pcap_pkthdr *);
#endif

PCAP_AVAILABLE_0_8
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 8 的条件编译指令
PCAP_API int    pcap_next_ex(pcap_t *, struct pcap_pkthdr **, const u_char **);
#endif

PCAP_AVAILABLE_0_8
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 8 的条件编译指令
PCAP_API void    pcap_breakloop(pcap_t *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API int    pcap_stats(pcap_t *, struct pcap_stat *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API int    pcap_setfilter(pcap_t *, struct bpf_program *);
#endif

PCAP_AVAILABLE_0_9
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 9 的条件编译指令
PCAP_API int    pcap_setdirection(pcap_t *, pcap_direction_t);
#endif

PCAP_AVAILABLE_0_7
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 7 的条件编译指令
PCAP_API int    pcap_getnonblock(pcap_t *, char *);
#endif

PCAP_AVAILABLE_0_7
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 7 的条件编译指令
PCAP_API int    pcap_setnonblock(pcap_t *, int, char *);
#endif

PCAP_AVAILABLE_0_9
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 9 的条件编译指令
PCAP_API int    pcap_inject(pcap_t *, const void *, size_t);
#endif

PCAP_AVAILABLE_0_8
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 8 的条件编译指令
PCAP_API int    pcap_sendpacket(pcap_t *, const u_char *, int);
#endif

PCAP_AVAILABLE_1_0
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 1 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 0 的条件编译指令
PCAP_API const char *pcap_statustostr(int);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API const char *pcap_strerror(int);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API char    *pcap_geterr(pcap_t *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API void    pcap_perror(pcap_t *, const char *);
#endif

PCAP_AVAILABLE_0_4
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 4 的条件编译指令
PCAP_API int    pcap_compile(pcap_t *, struct bpf_program *, const char *, int,
        bpf_u_int32);
#endif

PCAP_AVAILABLE_0_5
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 5 的条件编译指令
PCAP_DEPRECATED("use pcap_open_dead(), pcap_compile() and pcap_close()")
PCAP_API int    pcap_compile_nopcap(int, int, struct bpf_program *,
        const char *, int, bpf_u_int32);
#endif

/* XXX - this took two arguments in 0.4 and 0.5 */
PCAP_AVAILABLE_0_6
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 0 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 6 的条件编译指令
PCAP_API void    pcap_freecode(struct bpf_program *);
#endif

PCAP_AVAILABLE_1_0
#if defined(PCAP_API) && defined(PCAP_VERSION_MAJOR) && PCAP_VERSION_MAJOR >= 1 && defined(PCAP_VERSION_MINOR) && PCAP_VERSION_MINOR >= 0 的条件编译指令
# 定义一个函数，用于离线过滤数据包
PCAP_API int    pcap_offline_filter(const struct bpf_program *,
        const struct pcap_pkthdr *, const u_char *);

# 获取数据链路类型
PCAP_API int    pcap_datalink(pcap_t *);

# 获取扩展数据链路类型
PCAP_API int    pcap_datalink_ext(pcap_t *);

# 列出所有支持的数据链路类型
PCAP_API int    pcap_list_datalinks(pcap_t *, int **);

# 设置数据链路类型
PCAP_API int    pcap_set_datalink(pcap_t *, int);

# 释放数据链路类型列表
PCAP_API void    pcap_free_datalinks(int *);

# 将数据链路类型名称转换为值
PCAP_API int    pcap_datalink_name_to_val(const char *);

# 将数据链路类型值转换为名称
PCAP_API const char *pcap_datalink_val_to_name(int);

# 将数据链路类型值转换为描述
PCAP_API const char *pcap_datalink_val_to_description(int);

# 将数据链路类型值转换为描述或 DLT
PCAP_API const char *pcap_datalink_val_to_description_or_dlt(int);

# 获取快照长度
PCAP_API int    pcap_snapshot(pcap_t *);

# 检查数据是否交换
PCAP_API int    pcap_is_swapped(pcap_t *);

# 获取主要版本号
PCAP_API int    pcap_major_version(pcap_t *);

# 获取次要版本号
PCAP_API int    pcap_minor_version(pcap_t *);

# 获取缓冲区大小
PCAP_API int    pcap_bufsize(pcap_t *);

# 获取文件指针
PCAP_API FILE    *pcap_file(pcap_t *);

# 获取文件描述符
PCAP_API int    pcap_fileno(pcap_t *);

# 在 Windows 上初始化套接字
PCAP_API int    pcap_wsockinit(void);

# 打开一个数据包转储文件
PCAP_API pcap_dumper_t *pcap_dump_open(pcap_t *, const char *);
#ifdef _WIN32
  PCAP_AVAILABLE_0_9
  PCAP_API pcap_dumper_t *pcap_dump_hopen(pcap_t *, intptr_t);

  /*
   * 如果我们正在构建 libpcap，这是 sf-pcap.c 中的一个内部例程，因此我们不应将其定义为宏。
   *
   * 如果我们没有构建 libpcap，鉴于构建 libpcap 的 C 运行时版本可能与使用 libpcap 的应用程序构建的 C 运行时版本不同，
   * 并且 FILE 结构可能在两个版本的 C 运行时之间有所不同，对 _fileno() 的调用必须使用用于打开 FILE * 的 C 运行时版本中的 _fileno()，
   * 而不是用于构建 libpcap 的 C 运行时版本中的版本。 (也许一旦 Universal CRT 统治世界，这个问题就会消失。)
   */
  #ifndef BUILDING_PCAP
    #define pcap_dump_fopen(p,f) \
    pcap_dump_hopen(p, _get_osfhandle(_fileno(f)))
  #endif
#else /*_WIN32*/
  PCAP_AVAILABLE_0_9
  PCAP_API pcap_dumper_t *pcap_dump_fopen(pcap_t *, FILE *fp);
#endif /*_WIN32*/

PCAP_AVAILABLE_1_7
PCAP_API pcap_dumper_t *pcap_dump_open_append(pcap_t *, const char *);

PCAP_AVAILABLE_0_8
PCAP_API FILE    *pcap_dump_file(pcap_dumper_t *);

PCAP_AVAILABLE_0_9
PCAP_API long    pcap_dump_ftell(pcap_dumper_t *);

PCAP_AVAILABLE_1_9
PCAP_API int64_t    pcap_dump_ftell64(pcap_dumper_t *);

PCAP_AVAILABLE_0_8
PCAP_API int    pcap_dump_flush(pcap_dumper_t *);

PCAP_AVAILABLE_0_4
PCAP_API void    pcap_dump_close(pcap_dumper_t *);

PCAP_AVAILABLE_0_4
PCAP_API void    pcap_dump(u_char *, const struct pcap_pkthdr *, const u_char *);

PCAP_AVAILABLE_0_7
PCAP_API int    pcap_findalldevs(pcap_if_t **, char *);

PCAP_AVAILABLE_0_7
PCAP_API void    pcap_freealldevs(pcap_if_t *);
/*
 * 我们返回指向版本字符串的指针，而不是直接导出版本字符串。
 *
 * 至少在一些 UNIX 系统上，如果从共享库中导入数据到程序中，数据会被绑定到程序二进制文件中，因此如果程序链接时使用的库的版本与程序运行时使用的库的版本中的字符串不同，可能会发生各种不良情况（警告、字符串是与程序链接时使用的库版本中的字符串，或者更奇怪的情况，比如字符串是来自库中的字符串但被截断）。
 *
 * 在 Windows 上，字符串是在运行时构造的。
 */
PCAP_AVAILABLE_0_8
PCAP_API const char *pcap_lib_version(void);

#if defined(_WIN32)

  /*
   * Win32 definitions
   */

  /*!
    \brief 一个原始数据包的队列，将使用 pcap_sendqueue_transmit() 发送到网络。
  */
  struct pcap_send_queue
  {
    u_int maxlen;    /* 队列的最大大小，以字节为单位。此变量包含缓冲区字段的大小。 */
    u_int len;    /* 队列的当前大小，以字节为单位。 */
    char *buffer;    /* 包含要发送的数据包的缓冲区。 */
  };

  typedef struct pcap_send_queue pcap_send_queue;

  /*!
    \brief 这个 typedef 是 pcap_get_airpcap_handle() 函数的支持
  */
  #if !defined(AIRPCAP_HANDLE__EAE405F5_0171_9592_B3C2_C19EC426AD34__DEFINED_)
    #define AIRPCAP_HANDLE__EAE405F5_0171_9592_B3C2_C19EC426AD34__DEFINED_
  # 定义一个指向 _AirpcapHandle 结构体的指针类型 PAirpcapHandle
  typedef struct _AirpcapHandle *PAirpcapHandle;
  #endif

  # 设置捕获缓冲区大小
  PCAP_API int pcap_setbuff(pcap_t *p, int dim);
  # 设置捕获模式
  PCAP_API int pcap_setmode(pcap_t *p, int mode);
  # 设置最小拷贝数据包大小
  PCAP_API int pcap_setmintocopy(pcap_t *p, int size);

  # 获取捕获事件的句柄
  PCAP_API HANDLE pcap_getevent(pcap_t *p);

  # 发送 OID 获取请求
  PCAP_AVAILABLE_1_8
  PCAP_API int pcap_oid_get_request(pcap_t *, bpf_u_int32, void *, size_t *);

  # 发送 OID 设置请求
  PCAP_AVAILABLE_1_8
  PCAP_API int pcap_oid_set_request(pcap_t *, bpf_u_int32, const void *, size_t *);

  # 分配发送队列
  PCAP_API pcap_send_queue* pcap_sendqueue_alloc(u_int memsize);

  # 销毁发送队列
  PCAP_API void pcap_sendqueue_destroy(pcap_send_queue* queue);

  # 将数据包加入发送队列
  PCAP_API int pcap_sendqueue_queue(pcap_send_queue* queue, const struct pcap_pkthdr *pkt_header, const u_char *pkt_data);

  # 传输发送队列中的数据包
  PCAP_API u_int pcap_sendqueue_transmit(pcap_t *p, pcap_send_queue* queue, int sync);

  # 获取扩展统计信息
  PCAP_API struct pcap_stat *pcap_stats_ex(pcap_t *p, int *pcap_stat_size);

  # 设置用户缓冲区大小
  PCAP_API int pcap_setuserbuffer(pcap_t *p, int size);

  # 实时捕获数据包并将其写入文件
  PCAP_API int pcap_live_dump(pcap_t *p, char *filename, int maxsize, int maxpacks);

  # 结束实时捕获并将数据包写入文件
  PCAP_API int pcap_live_dump_ended(pcap_t *p, int sync);

  # 启动 OEM 模式
  PCAP_API int pcap_start_oem(char* err_str, int flags);

  # 获取 Airpcap 句柄
  PCAP_API PAirpcapHandle pcap_get_airpcap_handle(pcap_t *p);

  # 定义捕获模式常量
  #define MODE_CAPT 0
  #define MODE_STAT 1
  #define MODE_MON 2
#elif defined(MSDOS)

  /*
   * MS-DOS definitions
   */

  // 如果定义了 MSDOS，则进行 MS-DOS 相关的定义

  PCAP_API int  pcap_stats_ex (pcap_t *, struct pcap_stat_ex *);
  // 定义 pcap_stats_ex 函数，用于获取扩展的统计信息
  PCAP_API void pcap_set_wait (pcap_t *p, void (*yield)(void), int wait);
  // 定义 pcap_set_wait 函数，用于设置等待
  PCAP_API u_long pcap_mac_packets (void);
  // 定义 pcap_mac_packets 函数，用于获取 MAC 数据包数量

#else /* UN*X */

  /*
   * UN*X definitions
   */

  // 如果不是 MSDOS，则进行 UN*X 相关的定义

  PCAP_AVAILABLE_0_8
  PCAP_API int    pcap_get_selectable_fd(pcap_t *);
  // 定义 pcap_get_selectable_fd 函数，用于获取可选择的文件描述符

  PCAP_AVAILABLE_1_9
  PCAP_API const struct timeval *pcap_get_required_select_timeout(pcap_t *);
  // 定义 pcap_get_required_select_timeout 函数，用于获取所需的选择超时时间

#endif /* _WIN32/MSDOS/UN*X */

/*
 * Remote capture definitions.
 *
 * These routines are only present if libpcap has been configured to
 * include remote capture support.
 */

// 远程捕获定义

/*
 * The maximum buffer size in which address, port, interface names are kept.
 *
 * In case the adapter name or such is larger than this value, it is truncated.
 * This is not used by the user; however it must be aware that an hostname / interface
 * name longer than this value will be truncated.
 */
#define PCAP_BUF_SIZE 1024
// 定义最大缓冲区大小

/*
 * The type of input source, passed to pcap_open().
 */
#define PCAP_SRC_FILE        2    /* local savefile */
#define PCAP_SRC_IFLOCAL    3    /* local network interface */
#define PCAP_SRC_IFREMOTE    4    /* interface on a remote host, using RPCAP */
// 定义输入源的类型，传递给 pcap_open 函数

/*
 * URL schemes for capture source.
 */
/*
 * This string indicates that the user wants to open a capture from a
 * local file.
 */
#define PCAP_SRC_FILE_STRING "file://"
// 定义打开本地文件捕获的 URL 方案字符串

/*
 * This string indicates that the user wants to open a capture from a
 * network interface.  This string does not necessarily involve the use
 * of the RPCAP protocol. If the interface required resides on the local
 * host, the RPCAP protocol is not involved and the local functions are used.
 */
#define PCAP_SRC_IF_STRING "rpcap://"
// 定义打开网络接口捕获的 URL 方案字符串

/*
 * Flags to pass to pcap_open().
 */

/*
 * Specifies whether promiscuous mode is to be used.
 */
#define PCAP_OPENFLAG_PROMISCUOUS        0x00000001
// 指定是否使用混杂模式
/*
 * 指定对于 RPCAP 捕获，数据传输（在远程捕获的情况下）是否使用 UDP 协议。
 *
 * 如果设置为 '1'，表示希望使用 UDP 数据连接；如果设置为 '0'，表示希望使用 TCP 数据连接；控制连接始终基于 TCP。
 * UDP 连接更轻量，但不能保证所有捕获的数据包都到达客户端工作站。此外，在网络拥塞的情况下可能会有害。
 * 如果源不是远程接口，则此标志无意义。在这种情况下，它会被简单地忽略。
 */
#define PCAP_OPENFLAG_DATATX_UDP        0x00000002

/*
 * 指定远程探测器是否捕获自己生成的流量。
 *
 * 如果远程探测器使用相同的接口来捕获流量并向调用者发送数据，则捕获的流量包括 RPCAP 流量。如果打开了此标志，RPCAP 流量将被排除在捕获之外，以便返回给收集器的跟踪不包括此流量。
 *
 * 对本地接口或保存文件没有影响。
 */
#define PCAP_OPENFLAG_NOCAPTURE_RPCAP        0x00000004

/*
 * 指定本地适配器是否捕获自己生成的流量。
 *
 * 此标志告诉底层捕获驱动程序丢弃自己发送的数据包。这在构建诸如应该忽略它们刚刚发送的流量的桥接应用程序时很有用。
 *
 * 仅在 Windows 上受支持。
 */
#define PCAP_OPENFLAG_NOCAPTURE_LOCAL        0x00000008
/*
 * 这个标志配置适配器以获得最大的响应性。
 *
 * 在 nbytes 的值很大的情况下，WinPcap 会等待多个数据包到达后再将数据复制给用户。这保证了系统调用的次数较少，即处理器使用率较低，即性能更好，这对于嗅探器等应用程序是有利的。如果用户设置了 PCAP_OPENFLAG_MAX_RESPONSIVENESS 标志，捕获驱动程序将在应用程序准备好接收数据包时立即复制数据包。这对于需要最佳响应性的实时应用程序（例如桥接）是建议的。
 *
 * 使用 pcap_create()/pcap_activate() 的等效方式是“立即模式”。
 */
#define PCAP_OPENFLAG_MAX_RESPONSIVENESS    0x00000010

/*
 * 远程身份验证方法。
 * 这些方法用于 pcap_rmtauth 结构的 'type' 成员中。
 */

/*
 * 空身份验证。
 *
 * 'NULL' 身份验证必须等于 'zero'，这样旧应用程序就可以将 pcap_rmtauth 结构的每个字段都设置为零，然后它就可以工作。
 */
#define RPCAP_RMTAUTH_NULL 0
/*
 * 用户名/密码身份验证。
 *
 * 使用这种类型的身份验证，RPCAP 协议将使用提供的用户名/密码在远程机器上对用户进行身份验证。如果身份验证成功（并且用户有权打开网络设备），RPCAP 连接将继续；否则将被丢弃。
 *
 * *******注意********：除非使用了 TLS，否则用户名和密码以明文形式通过网络发送到捕获服务器。在你不完全控制的网络上（即使用 rpcap:// 而不是 rpcaps://），不要使用这个功能！（并且在你对“完全控制”的定义中要非常小心！）
 */
#define RPCAP_RMTAUTH_PWD 1
/*
 * 这个结构保存了在远程机器上进行用户认证所需的信息。
 *
 * 远程机器可以根据提供的信息来授予或拒绝访问权限。
 * 如果需要空认证，'username' 和 'password' 都可以是 NULL 指针。
 *
 * 如果源不是远程接口，这个结构就没有意义；在这种情况下，需要这样一个结构的函数也可以接受 NULL 指针。
 */
struct pcap_rmtauth
{
    /*
     * \brief 所需认证的类型。
     *
     * 为了提供最大的灵活性，我们可以支持基于 'type' 变量值的不同类型的认证。
     * 当前支持的认证方法定义在 \link remote_auth_methods 远程认证方法部分\endlink 中。
     */
    int type;
    /*
     * \brief 包含在远程机器上用于认证的用户名的以零结尾的字符串。
     *
     * 在 RPCAP_RMTAUTH_NULL 认证的情况下，这个字段没有意义，可以是 NULL。
     */
    char *username;
    /*
     * \brief 包含在远程机器上用于认证的密码的以零结尾的字符串。
     *
     * 在 RPCAP_RMTAUTH_NULL 认证的情况下，这个字段没有意义，可以是 NULL。
     */
    char *password;
};
/*
 * This routine can open a savefile, a local device, or a device on
 * a remote machine running an RPCAP server.
 *
 * For opening a savefile, the pcap_open_offline routines can be used,
 * and will work just as well; code using them will work on more
 * platforms than code using pcap_open() to open savefiles.
 *
 * For opening a local device, pcap_open_live() can be used; it supports
 * most of the capabilities that pcap_open() supports, and code using it
 * will work on more platforms than code using pcap_open().  pcap_create()
 * and pcap_activate() can also be used; they support all capabilities
 * that pcap_open() supports, except for the Windows-only
 * PCAP_OPENFLAG_NOCAPTURE_LOCAL, and support additional capabilities.
 *
 * For opening a remote capture, pcap_open() is currently the only
 * API available.
 */
PCAP_AVAILABLE_1_9
PCAP_API pcap_t    *pcap_open(const char *source, int snaplen, int flags,
        int read_timeout, struct pcap_rmtauth *auth, char *errbuf);

PCAP_AVAILABLE_1_9
PCAP_API int    pcap_createsrcstr(char *source, int type, const char *host,
        const char *port, const char *name, char *errbuf);

PCAP_AVAILABLE_1_9
PCAP_API int    pcap_parsesrcstr(const char *source, int *type, char *host,
        char *port, char *name, char *errbuf);
/*
 * This routine can scan a directory for savefiles, list local capture
 * devices, or list capture devices on a remote machine running an RPCAP
 * server.
 *
 * For scanning for savefiles, it can be used on both UN*X systems and
 * Windows systems; for each directory entry it sees, it tries to open
 * the file as a savefile using pcap_open_offline(), and only includes
 * it in the list of files if the open succeeds, so it filters out
 * files for which the user doesn't have read permission, as well as
 * files that aren't valid savefiles readable by libpcap.
 *
 * For listing local capture devices, it's just a wrapper around
 * pcap_findalldevs(); code using pcap_findalldevs() will work on more
 * platforms than code using pcap_findalldevs_ex().
 *
 * For listing remote capture devices, pcap_findalldevs_ex() is currently
 * the only API available.
 */
PCAP_AVAILABLE_1_9
PCAP_API int    pcap_findalldevs_ex(const char *source,
        struct pcap_rmtauth *auth, pcap_if_t **alldevs, char *errbuf);

/*
 * Sampling methods.
 *
 * These allow pcap_loop(), pcap_dispatch(), pcap_next(), and pcap_next_ex()
 * to see only a sample of packets, rather than all packets.
 *
 * Currently, they work only on Windows local captures.
 */

/*
 * Specifies that no sampling is to be done on the current capture.
 *
 * In this case, no sampling algorithms are applied to the current capture.
 */
#define PCAP_SAMP_NOSAMP    0

/*
 * Specifies that only 1 out of N packets must be returned to the user.
 *
 * In this case, the 'value' field of the 'pcap_samp' structure indicates the
 * number of packets (minus 1) that must be discarded before one packet got
 * accepted.
 * In other words, if 'value = 10', the first packet is returned to the
 * caller, while the following 9 are discarded.
 */
#define PCAP_SAMP_1_EVERY_N    1
/*
 * Specifies that we have to return 1 packet every N milliseconds.
 *
 * In this case, the 'value' field of the 'pcap_samp' structure indicates
 * the 'waiting time' in milliseconds before one packet got accepted.
 * In other words, if 'value = 10', the first packet is returned to the
 * caller; the next returned one will be the first packet that arrives
 * when 10ms have elapsed.
 */
#define PCAP_SAMP_FIRST_AFTER_N_MS 2

/*
 * This structure defines the information related to sampling.
 *
 * In case the sampling is requested, the capturing device should read
 * only a subset of the packets coming from the source. The returned packets
 * depend on the sampling parameters.
 *
 * WARNING: The sampling process is applied *after* the filtering process.
 * In other words, packets are filtered first, then the sampling process
 * selects a subset of the 'filtered' packets and it returns them to the
 * caller.
 */
struct pcap_samp
{
    /*
     * Method used for sampling; see above.
     */
    int method;

    /*
     * This value depends on the sampling method defined.
     * For its meaning, see above.
     */
    int value;
};

/*
 * New functions.
 */
PCAP_AVAILABLE_1_9
PCAP_API struct pcap_samp *pcap_setsampling(pcap_t *p);

/*
 * RPCAP active mode.
 */

/* Maximum length of an host name (needed for the RPCAP active mode) */
#define RPCAP_HOSTLIST_SIZE 1024

PCAP_AVAILABLE_1_9
PCAP_API SOCKET    pcap_remoteact_accept(const char *address, const char *port,
        const char *hostlist, char *connectinghost,
        struct pcap_rmtauth *auth, char *errbuf);

PCAP_AVAILABLE_1_10
PCAP_API SOCKET    pcap_remoteact_accept_ex(const char *address, const char *port,
        const char *hostlist, char *connectinghost,
        struct pcap_rmtauth *auth, int uses_ssl, char *errbuf);

PCAP_AVAILABLE_1_9
PCAP_API int    pcap_remoteact_list(char *hostlist, char sep, int size,
        char *errbuf);

PCAP_AVAILABLE_1_9
# 定义一个返回整型值的函数，用于关闭远程操作
PCAP_API int    pcap_remoteact_close(const char *host, char *errbuf);

# 定义一个无返回值的函数，用于清理远程操作
PCAP_AVAILABLE_1_9
PCAP_API void    pcap_remoteact_cleanup(void);

# 如果是 C++ 环境，则结束 extern "C" 块
#ifdef __cplusplus
}
#endif

# 结束头文件的条件编译
#endif /* lib_pcap_pcap_h */
```