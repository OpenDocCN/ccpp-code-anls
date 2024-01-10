# `nmap\libpcap\gencode.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，需满足以下条件：
 * 1. 源代码发布需保留以上版权声明和本段文字
 * 2. 包含二进制代码的发布需在文档或其他提供的材料中包含以上版权声明和本段文字
 * 3. 所有提及此软件特性或使用的广告材料需显示以下声明：
 *    “本产品包含由加利福尼亚大学、劳伦斯伯克利实验室及其贡献者开发的软件。”
 *    未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifdef _WIN32
  #include <ws2tcpip.h>
#else
  #include <sys/socket.h>

  #ifdef __NetBSD__
    #include <sys/param.h>
  #endif

  #include <netinet/in.h>
  #include <arpa/inet.h>
#endif /* _WIN32 */

#include <stdlib.h>
#include <string.h>
#include <memory.h>
#include <setjmp.h>
#include <stdarg.h>
#include <stdio.h>

#ifdef MSDOS
#include "pcap-dos.h"
#endif

#include "pcap-int.h"

#include "extract.h"

#include "ethertype.h"
#include "nlpid.h"
#include "llc.h"
#include "gencode.h"
#include "ieee80211.h"
#include "atmuni31.h"
#include "sunatmpos.h"
#include "pflog.h"
#include "ppp.h"
#include "pcap/sll.h"
#include "pcap/ipnet.h"
#include "arcnet.h"
#include "diag-control.h"

#include "scanner.h"

#if defined(linux)
#include <linux/types.h>
#include <linux/if_packet.h>
#include <linux/filter.h>
#endif

#ifndef offsetof
#define offsetof(s, e) ((size_t)&((s *)0)->e)
#endif

#ifdef _WIN32
  #ifdef INET6
    #if defined(__MINGW32__) && defined(DEFINE_ADDITIONAL_IPV6_STUFF)
/* IPv6 address */
struct in6_addr
  {
    union
      {
    uint8_t        u6_addr8[16];
    uint16_t    u6_addr16[8];
    uint32_t    u6_addr32[4];
      } in6_u;
#define s6_addr            in6_u.u6_addr8
#define s6_addr16        in6_u.u6_addr16
#define s6_addr32        in6_u.u6_addr32
#define s6_addr64        in6_u.u6_addr64
  };

typedef unsigned short    sa_family_t;

#define    __SOCKADDR_COMMON(sa_prefix) \
  sa_family_t sa_prefix##family

/* Ditto, for IPv6.  */
struct sockaddr_in6
  {
    __SOCKADDR_COMMON (sin6_);
    uint16_t sin6_port;        /* Transport layer port # */
    uint32_t sin6_flowinfo;    /* IPv6 flow information */
    struct in6_addr sin6_addr;    /* IPv6 address */
  };

      #ifndef EAI_ADDRFAMILY
struct addrinfo {
    int    ai_flags;    /* AI_PASSIVE, AI_CANONNAME */
    int    ai_family;    /* PF_xxx */
    int    ai_socktype;    /* SOCK_xxx */
    int    ai_protocol;    /* 0 or IPPROTO_xxx for IPv4 and IPv6 */
    size_t    ai_addrlen;    /* length of ai_addr */
    char    *ai_canonname;    /* canonical name for hostname */
    struct sockaddr *ai_addr;    /* binary address */
    struct addrinfo *ai_next;    /* next structure in linked list */
};
      #endif /* EAI_ADDRFAMILY */
    #endif /* defined(__MINGW32__) && defined(DEFINE_ADDITIONAL_IPV6_STUFF) */
  #endif /* INET6 */
#else /* _WIN32 */
  #include <netdb.h>    /* for "struct addrinfo" */
#endif /* _WIN32 */
#include <pcap/namedb.h>

#include "nametoaddr.h"

#define ETHERMTU    1500

#ifndef IPPROTO_HOPOPTS
#define IPPROTO_HOPOPTS 0
#endif
#ifndef IPPROTO_ROUTING
#define IPPROTO_ROUTING 43
#endif
#ifndef IPPROTO_FRAGMENT
#define IPPROTO_FRAGMENT 44
#endif
#ifndef IPPROTO_DSTOPTS
#define IPPROTO_DSTOPTS 60
#endif
#ifndef IPPROTO_SCTP
#define IPPROTO_SCTP 132
#endif

如果未定义 IPPROTO_SCTP，则定义 IPPROTO_SCTP 为 132


#define GENEVE_PORT 6081

定义 GENEVE_PORT 为 6081


#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

如果定义了 HAVE_OS_PROTO_H，则包含 os-proto.h 文件


#define JMP(c) ((c)|BPF_JMP|BPF_K)

定义 JMP 宏，用于设置跳转指令


#define PUSH_LINKHDR(cs, new_linktype, new_is_variable, new_constant_part, new_reg) \
{ \
    (cs)->prevlinktype = (cs)->linktype; \
    (cs)->off_prevlinkhdr = (cs)->off_linkhdr; \
    (cs)->linktype = (new_linktype); \
    (cs)->off_linkhdr.is_variable = (new_is_variable); \
    (cs)->off_linkhdr.constant_part = (new_constant_part); \
    (cs)->off_linkhdr.reg = (new_reg); \
    (cs)->is_geneve = 0; \
}

定义 PUSH_LINKHDR 宏，用于设置新的链路层头部信息


#define OFFSET_NOT_SET    0xffffffffU

定义 OFFSET_NOT_SET 为 0xffffffffU


typedef struct {
    int    is_variable;
    u_int    constant_part;
    int    reg;
} bpf_abs_offset;

定义 bpf_abs_offset 结构体，用于表示绝对偏移量信息


enum e_offrel {
    OR_PACKET,        /* full packet data */
    OR_LINKHDR,        /* link-layer header */
    OR_PREVLINKHDR,        /* previous link-layer header */
    OR_LLC,            /* 802.2 LLC header */

定义 e_offrel 枚举类型，表示偏移量相对于的位置
    OR_PREVMPLSHDR,        /* 之前的MPLS头部 */
    OR_LINKTYPE,        /* 链路层类型 */
    OR_LINKPL,        /* 链路层有效载荷 */
    OR_LINKPL_NOSNAP,    /* 链路层有效载荷，没有SNAP头部 */
    OR_TRAN_IPV4,        /* 传输层头部，带有IPv4网络层 */
    OR_TRAN_IPV6        /* 传输层头部，带有IPv6网络层 */
};

/*
 * 我们分配内存块而不是每次调用malloc，这样就不必担心内存泄漏。如果所有这些内存都被浪费了，这可能并不是什么大问题，但如果这个代码库被用在一个库中，那可能就不是一个好主意了。
 *
 * XXX - 这确实是一个库中的代码....
 */
#define NCHUNKS 16  // 定义内存块的数量
#define CHUNK0SIZE 1024  // 定义第一个内存块的大小
struct chunk {
    size_t n_left;  // 剩余可用内存大小
    void *m;  // 内存块指针
};

/* 代码生成器状态 */

struct _compiler_state {
    jmp_buf top_ctx;  // 保存当前执行上下文的缓冲区
    pcap_t *bpf_pcap;  // pcap句柄
    int error_set;  // 错误标志

    struct icode ic;  // 代码生成器

    int snaplen;  // 抓包长度

    int linktype;  // 链路类型
    int prevlinktype;  // 前一个链路类型
    int outermostlinktype;  // 最外层链路类型

    bpf_u_int32 netmask;  // 网络掩码
    int no_optimize;  // 不优化标志

    /* 用于处理VLAN和MPLS堆栈的Hack。 */
    u_int label_stack_depth;  // 标签堆栈深度
    u_int vlan_stack_depth;  // VLAN堆栈深度

    /* XXX */
    u_int pcap_fddipad;  // pcap FDDI填充

    /*
     * 由于错误是通过longjmp处理的，所以必须在longjmp处理程序中释放分配的任何东西，因此它必须从处理程序中可达。
     *
     * 分配的一个东西是pcap_nametoaddrinfo()的结果；必须使用freeaddrinfo()释放它。这个变量指向任何需要被释放的addrinfo结构。
     */
    struct addrinfo *ai;  // 地址信息结构指针

    /*
     * 另一个分配的东西是pcap_ether_aton()的结果；必须使用free()释放它。这个变量指向任何需要被释放的地址。
     */
    u_char *e;  // 地址指针

    /*
     * 各种代码结构需要知道数据包的布局。这些值给出了从数据包开头的必要偏移量。
     */

    /*
     * 链路层头部的绝对偏移量。
     */
    bpf_abs_offset off_linkhdr;  // 链路层头部的偏移量

    /*
     * 如果我们正在检查另一个协议层封装的数据包的链路层头部，这是从原始数据包数据开头的前几层的链路层头部的等效信息。
     */
    # 定义一个变量，用于存储前一个链路头的绝对偏移量
    bpf_abs_offset off_prevlinkhdr;
    
    # 这是最外层链路头的等效信息
    bpf_abs_offset off_outermostlinkhdr;
    
    # 链路层有效载荷的起始绝对偏移量
    bpf_abs_offset off_linkpl;
    
    # "off_linktype"是链路层头部中给出数据包类型信息的偏移量。这是从数据包开始的绝对偏移量。
    # 对于以太网，它是以太网类型字段的偏移量；这意味着它必须有一个跳过VLAN标记的值。
    # 对于始终使用802.2头部的链路层类型，它是LLC头部的偏移量；这意味着它必须有一个跳过VLAN标记的值。
    # 对于PPP，它是PPP类型字段的偏移量。
    # 对于Cisco HDLC，它是CHDLC类型字段的偏移量。
    # 对于BSD环回，它是AF_值的偏移量。
    # 对于Linux cooked sockets，它是类型字段的偏移量。
    # off_linktype.constant_part设置为OFFSET_NOT_SET表示没有封装，此时假定为IP。
    bpf_abs_offset off_linktype;
    
    # 如果链路层包含ATM伪头，则为TRUE。
    int is_atm;
    
    # 如果过滤器中出现"geneve"，则为TRUE；这会导致我们生成检查Geneve头并假定后续过滤器适用于封装载荷的代码。
    int is_geneve;
    
    # 如果需要VLAN偏移的可变长度部分，则为TRUE。
    int is_vlan_vloffset;
    
    # 这些是ATM伪头的偏移量。
    u_int off_vpi;
    u_int off_vci;
    u_int off_proto;
    
    # 这些是MTP2字段的偏移量。
    u_int off_li;
    u_int off_li_hsl;
    
    # 这些是MTP3字段的偏移量。
    u_int off_sio;
    u_int off_opc;
    # 定义偏移量off_dpc，用于存储DPC（Data Path Component）的偏移量
    u_int off_dpc;
    # 定义偏移量off_sls，用于存储SLS（Segmentation Layer Service）的偏移量
    u_int off_sls;

    # 定义偏移量off_payload，用于存储ATM伪头之后的第一个字节的偏移量，如果没有ATM伪头则为-1
    u_int off_payload;

    # 定义偏移量off_nl，用于存储网络层头部的偏移量，相对于链路层负载的开始位置
    u_int off_nl;
    # 定义偏移量off_nl_nosnap，用于存储网络层头部的偏移量，相对于链路层负载的开始位置，如果没有SNAP头部则与off_nl相同
    u_int off_nl_nosnap;

    # 用于存储寄存器的使用情况，如果寄存器分配过多则分配器会放弃
    int regused[BPF_MEMWORDS];
    # 当前寄存器的索引
    int curreg;

    # 存储内存块
    struct chunk chunks[NCHUNKS];
    # 当前内存块的索引
    int cur_chunk;
};

/*
 * 用于此文件外部的例程使用。
 */
/* VARARGS */
void
bpf_set_error(compiler_state_t *cstate, const char *fmt, ...)
{
    va_list ap;

    /*
     * 如果我们已经设置了错误，则不要覆盖它。
     * 词法分析器通过设置错误然后返回 LEX_ERROR 标记来报告一些错误，该标记不被任何语法规则识别，因此强制解析停止。
     * 我们不希望词法分析器报告的错误被语法错误覆盖。
     */
    if (!cstate->error_set) {
        va_start(ap, fmt);
        (void)vsnprintf(cstate->bpf_pcap->errbuf, PCAP_ERRBUF_SIZE,
            fmt, ap);
        va_end(ap);
        cstate->error_set = 1;
    }
}

/*
 * 仅供此文件中的例程使用。
 */
static void PCAP_NORETURN bpf_error(compiler_state_t *, const char *, ...)
    PCAP_PRINTFLIKE(2, 3);

/* VARARGS */
static void PCAP_NORETURN
bpf_error(compiler_state_t *cstate, const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    (void)vsnprintf(cstate->bpf_pcap->errbuf, PCAP_ERRBUF_SIZE,
        fmt, ap);
    va_end(ap);
    longjmp(cstate->top_ctx, 1);
    /*NOTREACHED*/
#ifdef _AIX
    PCAP_UNREACHABLE
#endif /* _AIX */
}

static int init_linktype(compiler_state_t *, pcap_t *);

static void init_regs(compiler_state_t *);
static int alloc_reg(compiler_state_t *);
static void free_reg(compiler_state_t *, int);

static void initchunks(compiler_state_t *cstate);
static void *newchunk_nolongjmp(compiler_state_t *cstate, size_t);
static void *newchunk(compiler_state_t *cstate, size_t);
static void freechunks(compiler_state_t *cstate);
static inline struct block *new_block(compiler_state_t *cstate, int);
static inline struct slist *new_stmt(compiler_state_t *cstate, int);
static struct block *gen_retblk(compiler_state_t *cstate, int);
static inline void syntax(compiler_state_t *cstate);

static void backpatch(struct block *, struct block *);
# 合并两个块
static void merge(struct block *, struct block *);

# 生成比较操作的块
static struct block *gen_cmp(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32);
static struct block *gen_cmp_gt(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32);
static struct block *gen_cmp_ge(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32);
static struct block *gen_cmp_lt(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32);
static struct block *gen_cmp_le(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32);

# 生成多字段比较操作的块
static struct block *gen_mcmp(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32, bpf_u_int32);

# 生成比较二进制数据操作的块
static struct block *gen_bcmp(compiler_state_t *, enum e_offrel, u_int, u_int, const u_char *);

# 生成比较操作的块，支持不同的比较类型
static struct block *gen_ncmp(compiler_state_t *, enum e_offrel, u_int, u_int, bpf_u_int32, int, int, bpf_u_int32);

# 生成加载绝对偏移和相对偏移的操作序列
static struct slist *gen_load_absoffsetrel(compiler_state_t *, bpf_abs_offset *, u_int, u_int);
static struct slist *gen_load_a(compiler_state_t *, enum e_offrel, u_int, u_int);

# 生成加载 IP 头长度的操作序列
static struct slist *gen_loadx_iphdrlen(compiler_state_t *);

# 生成无条件跳转的块
static struct block *gen_uncond(compiler_state_t *, int);

# 生成始终为真的块
static inline struct block *gen_true(compiler_state_t *);

# 生成始终为假的块
static inline struct block *gen_false(compiler_state_t *);

# 生成以太网链路类型的块
static struct block *gen_ether_linktype(compiler_state_t *, bpf_u_int32);

# 生成 IP 网络链路类型的块
static struct block *gen_ipnet_linktype(compiler_state_t *, bpf_u_int32);

# 生成 Linux SLL 链路类型的块
static struct block *gen_linux_sll_linktype(compiler_state_t *, bpf_u_int32);

# 生成加载 pflog 链路层前缀长度的操作序列
static struct slist *gen_load_pflog_llprefixlen(compiler_state_t *);

# 生成加载 prism 链路层前缀长度的操作序列
static struct slist *gen_load_prism_llprefixlen(compiler_state_t *);

# 生成加载 avs 链路层前缀长度的操作序列
static struct slist *gen_load_avs_llprefixlen(compiler_state_t *);

# 生成加载 radiotap 链路层前缀长度的操作序列
static struct slist *gen_load_radiotap_llprefixlen(compiler_state_t *);

# 生成加载 ppi 链路层前缀长度的操作序列
static struct slist *gen_load_ppi_llprefixlen(compiler_state_t *);

# 插入计算虚拟偏移量的操作序列
static void insert_compute_vloffsets(compiler_state_t *, struct block *);
# 生成绝对偏移变量部分的静态链表
static struct slist *gen_abs_offset_varpart(compiler_state_t *, bpf_abs_offset *);

# 将以太网类型转换为 PPP 类型
static bpf_u_int32 ethertype_to_ppptype(bpf_u_int32);

# 生成链路类型的代码块
static struct block *gen_linktype(compiler_state_t *, bpf_u_int32);

# 生成 SNAP 类型的代码块
static struct block *gen_snap(compiler_state_t *, bpf_u_int32, bpf_u_int32);

# 生成 LLC 链路类型的代码块
static struct block *gen_llc_linktype(compiler_state_t *, bpf_u_int32);

# 生成主机操作的代码块
static struct block *gen_hostop(compiler_state_t *, bpf_u_int32, bpf_u_int32, int, bpf_u_int32, u_int, u_int);

# 如果支持 IPv6，则生成 IPv6 主机操作的代码块
#ifdef INET6
static struct block *gen_hostop6(compiler_state_t *, struct in6_addr *, struct in6_addr *, int, bpf_u_int32, u_int, u_int);
#endif

# 生成地址主机操作的代码块
static struct block *gen_ahostop(compiler_state_t *, const u_char *, int);

# 生成以太网主机操作的代码块
static struct block *gen_ehostop(compiler_state_t *, const u_char *, int);

# 生成 FDDI 主机操作的代码块
static struct block *gen_fhostop(compiler_state_t *, const u_char *, int);

# 生成令牌环网主机操作的代码块
static struct block *gen_thostop(compiler_state_t *, const u_char *, int);

# 生成 WLAN 主机操作的代码块
static struct block *gen_wlanhostop(compiler_state_t *, const u_char *, int);

# 生成 IP over FC 主机操作的代码块
static struct block *gen_ipfchostop(compiler_state_t *, const u_char *, int);

# 生成域名主机操作的代码块
static struct block *gen_dnhostop(compiler_state_t *, bpf_u_int32, int);

# 生成 MPLS 链路类型的代码块
static struct block *gen_mpls_linktype(compiler_state_t *, bpf_u_int32);

# 生成主机操作的代码块
static struct block *gen_host(compiler_state_t *, bpf_u_int32, bpf_u_int32, int, int, int);

# 如果支持 IPv6，则生成 IPv6 主机操作的代码块
#ifdef INET6
static struct block *gen_host6(compiler_state_t *, struct in6_addr *, struct in6_addr *, int, int, int);
#endif

# 如果不支持 IPv6，则生成网关操作的代码块
#ifndef INET6
static struct block *gen_gateway(compiler_state_t *, const u_char *, struct addrinfo *, int, int);
#endif

# 生成 IP 分片操作的代码块
static struct block *gen_ipfrag(compiler_state_t *);

# 生成端口原子操作的代码块
static struct block *gen_portatom(compiler_state_t *, int, bpf_u_int32);

# 生成端口范围原子操作的代码块
static struct block *gen_portrangeatom(compiler_state_t *, u_int, bpf_u_int32, bpf_u_int32);

# 生成 IPv6 端口原子操作的代码块
static struct block *gen_portatom6(compiler_state_t *, int, bpf_u_int32);

# 生成 IPv6 端口范围原子操作的代码块
static struct block *gen_portrangeatom6(compiler_state_t *, u_int, bpf_u_int32, bpf_u_int32);
# 定义一个静态函数，生成端口操作的块
static struct block *gen_portop(compiler_state_t *, u_int, u_int, int);
# 定义一个静态函数，生成端口的块
static struct block *gen_port(compiler_state_t *, u_int, int, int);
# 定义一个静态函数，生成端口范围操作的块
static struct block *gen_portrangeop(compiler_state_t *, u_int, u_int, bpf_u_int32, int);
# 定义一个静态函数，生成端口范围的块
static struct block *gen_portrange(compiler_state_t *, u_int, u_int, int, int);
# 定义一个静态函数，生成 IPv6 端口操作的块
struct block *gen_portop6(compiler_state_t *, u_int, u_int, int);
# 定义一个静态函数，生成 IPv6 端口的块
static struct block *gen_port6(compiler_state_t *, u_int, int, int);
# 定义一个静态函数，生成 IPv6 端口范围操作的块
static struct block *gen_portrangeop6(compiler_state_t *, u_int, u_int, bpf_u_int32, int);
# 定义一个静态函数，生成 IPv6 端口范围的块
static struct block *gen_portrange6(compiler_state_t *, u_int, u_int, int, int);
# 定义一个静态函数，查找协议
static int lookup_proto(compiler_state_t *, const char *, int);
# 如果未定义 NO_PROTOCHAIN，则定义一个静态函数，生成协议链的块
#if !defined(NO_PROTOCHAIN)
static struct block *gen_protochain(compiler_state_t *, bpf_u_int32, int);
#endif /* !defined(NO_PROTOCHAIN) */
# 定义一个静态函数，生成协议的块
static struct block *gen_proto(compiler_state_t *, bpf_u_int32, int, int);
# 定义一个静态函数，将算术操作转换为 X 寄存器
static struct slist *xfer_to_x(compiler_state_t *, struct arth *);
# 定义一个静态函数，将算术操作转换为 A 寄存器
static struct slist *xfer_to_a(compiler_state_t *, struct arth *);
# 定义一个静态函数，生成 MAC 多播地址检查的块
static struct block *gen_mac_multicast(compiler_state_t *, int);
# 定义一个静态函数，生成长度检查的块
static struct block *gen_len(compiler_state_t *, int, int);
# 定义一个静态函数，生成 802.11 数据帧检查的块
static struct block *gen_check_802_11_data_frame(compiler_state_t *);
# 定义一个静态函数，生成 Geneve 链路层检查的块
static struct block *gen_geneve_ll_check(compiler_state_t *cstate);
# 定义一个静态函数，生成 PPI 数据链路层类型检查的块
static struct block *gen_ppi_dlt_check(compiler_state_t *);
# 定义一个静态函数，生成 ATM 字段代码内部检查的块
static struct block *gen_atmfield_code_internal(compiler_state_t *, int, bpf_u_int32, int, int);
# 定义一个静态函数，生成 ATM 类型 LLC 检查的块
static struct block *gen_atmtype_llc(compiler_state_t *);
# 定义一个静态函数，生成消息缩写的块
static struct block *gen_msg_abbrev(compiler_state_t *, int type);

# 初始化块数组
static void
initchunks(compiler_state_t *cstate)
{
    int i;

    for (i = 0; i < NCHUNKS; i++) {
        cstate->chunks[i].n_left = 0;
        cstate->chunks[i].m = NULL;
    }
    cstate->cur_chunk = 0;
}

# 分配新的块
static void *
newchunk_nolongjmp(compiler_state_t *cstate, size_t n)
{
    struct chunk *cp;
    int k;
    size_t size;

    #ifndef __NetBSD__
    /* XXX Round up to nearest long. */
    # 将 n 增加到 long 类型的大小，然后将结果向上取整到最接近的 long 类型的倍数
    n = (n + sizeof(long) - 1) & ~(sizeof(long) - 1);
#else
    /* 如果条件不成立，则执行以下代码 */
    /* XXX 四舍五入到结构边界 */
    n = ALIGN(n);
#endif

    /* 指向当前块的指针 */
    cp = &cstate->chunks[cstate->cur_chunk];
    /* 如果需要的空间大于当前块的剩余空间 */
    if (n > cp->n_left) {
        /* 切换到下一个块 */
        ++cp;
        k = ++cstate->cur_chunk;
        /* 如果超过了块的数量上限 */
        if (k >= NCHUNKS) {
            bpf_set_error(cstate, "out of memory");
            return (NULL);
        }
        /* 计算新块的大小 */
        size = CHUNK0SIZE << k;
        /* 分配新块的内存空间 */
        cp->m = (void *)malloc(size);
        /* 如果分配内存失败 */
        if (cp->m == NULL) {
            bpf_set_error(cstate, "out of memory");
            return (NULL);
        }
        /* 将新块的内存空间清零 */
        memset((char *)cp->m, 0, size);
        /* 更新新块的剩余空间 */
        cp->n_left = size;
        /* 如果需要的空间大于新块的大小 */
        if (n > size) {
            bpf_set_error(cstate, "out of memory");
            return (NULL);
        }
    }
    /* 更新当前块的剩余空间 */
    cp->n_left -= n;
    /* 返回分配的内存空间的指针 */
    return (void *)((char *)cp->m + cp->n_left);
}

/* 分配新块的内存空间 */
static void *
newchunk(compiler_state_t *cstate, size_t n)
{
    void *p;

    p = newchunk_nolongjmp(cstate, n);
    /* 如果分配内存失败，则跳转到错误处理 */
    if (p == NULL) {
        longjmp(cstate->top_ctx, 1);
        /*NOTREACHED*/
    }
    return (p);
}

/* 释放所有块的内存空间 */
static void
freechunks(compiler_state_t *cstate)
{
    int i;

    for (i = 0; i < NCHUNKS; ++i)
        if (cstate->chunks[i].m != NULL)
            free(cstate->chunks[i].m);
}

/*
 * 一个在代码生成结束后释放分配的内存空间的 strdup。
 * 这被词法分析器使用，所以它不能 longjmp；它只是在分配错误时返回 NULL，调用者必须检查它。
 */
char *
sdup(compiler_state_t *cstate, const char *s)
{
    size_t n = strlen(s) + 1;
    char *cp = newchunk_nolongjmp(cstate, n);

    if (cp == NULL)
        return (NULL);
    pcap_strlcpy(cp, s, n);
    return (cp);
}

/* 分配新的块并初始化为指定的代码 */
static inline struct block *
new_block(compiler_state_t *cstate, int code)
{
    struct block *p;

    p = (struct block *)newchunk(cstate, sizeof(*p));
    p->s.code = code;
    p->head = p;

    return p;
}

/* 分配新的块并初始化为指定的代码 */
static inline struct slist *
new_stmt(compiler_state_t *cstate, int code)
{
    struct slist *p;

    p = (struct slist *)newchunk(cstate, sizeof(*p));
    p->s.code = code;
    # 返回变量 p 的值
    return p;
}

// 生成一个返回指定值的新的块
static struct block *
gen_retblk(compiler_state_t *cstate, int v)
{
    // 创建一个新的块，类型为 BPF_RET|BPF_K
    struct block *b = new_block(cstate, BPF_RET|BPF_K);

    // 设置块的 k 值为 v
    b->s.k = v;
    // 返回该块
    return b;
}

// 语法错误处理函数
static inline PCAP_NORETURN_DEF void
syntax(compiler_state_t *cstate)
{
    // 抛出语法错误异常，提示过滤表达式中存在语法错误
    bpf_error(cstate, "syntax error in filter expression");
}

// 编译过滤器表达式
int
pcap_compile(pcap_t *p, struct bpf_program *program,
         const char *buf, int optimize, bpf_u_int32 mask)
{
#ifdef _WIN32
    static int done = 0;
#endif
    compiler_state_t cstate;
    const char * volatile xbuf = buf;
    yyscan_t scanner = NULL;
    volatile YY_BUFFER_STATE in_buffer = NULL;
    u_int len;
    int  rc;

    /*
     * If this pcap_t hasn't been activated, it doesn't have a
     * link-layer type, so we can't use it.
     */
    // 如果 pcap_t 还未被激活，则无法使用它，抛出错误
    if (!p->activated) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "not-yet-activated pcap_t passed to pcap_compile");
        return (PCAP_ERROR);
    }

#ifdef _WIN32
    // 在 Windows 平台下，初始化 Winsock
    if (!done)
        pcap_wsockinit();
    done = 1;
#endif

#ifdef ENABLE_REMOTE
    /*
     * If the device on which we're capturing need to be notified
     * that a new filter is being compiled, do so.
     *
     * This allows them to save a copy of it, in case, for example,
     * they're implementing a form of remote packet capture, and
     * want the remote machine to filter out the packets in which
     * it's sending the packets it's captured.
     *
     * XXX - the fact that we happen to be compiling a filter
     * doesn't necessarily mean we'll be installing it as the
     * filter for this pcap_t; we might be running it from userland
     * on captured packets to do packet classification.  We really
     * need a better way of handling this, but this is all that
     * the WinPcap remote capture code did.
     */
    // 如果需要通知捕获数据的设备正在编译新的过滤器，则执行通知操作
    if (p->save_current_filter_op != NULL)
        (p->save_current_filter_op)(p, buf);
#endif

    // 初始化编译器状态
    initchunks(&cstate);
    cstate.no_optimize = 0;
#ifdef INET6
    cstate.ai = NULL;
#endif
    cstate.e = NULL;
}
    # 设置 cstate.ic.root 为 NULL
    cstate.ic.root = NULL;
    # 设置 cstate.ic.cur_mark 为 0
    cstate.ic.cur_mark = 0;
    # 将 p 赋值给 cstate.bpf_pcap
    cstate.bpf_pcap = p;
    # 将 cstate.error_set 设置为 0
    cstate.error_set = 0;
    # 初始化寄存器
    init_regs(&cstate);

    # 设置 cstate.netmask 为 mask
    cstate.netmask = mask;

    # 获取 pcap 快照长度并赋值给 cstate.snaplen
    cstate.snaplen = pcap_snapshot(p);
    # 如果快照长度为 0，则设置错误信息并返回错误码
    if (cstate.snaplen == 0) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
             "snaplen of 0 rejects all packets");
        rc = PCAP_ERROR;
        goto quit;
    }

    # 如果初始化词法分析器失败，则设置错误信息
    if (pcap_lex_init(&scanner) != 0)
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't initialize scanner");
    # 将 xbuf 转换为字符串并赋值给 in_buffer
    in_buffer = pcap__scan_string(xbuf ? xbuf : "", scanner);

    # 将编译器状态与词法分析器状态关联起来
    pcap_set_extra(&cstate, scanner);

    # 如果初始化链路类型失败，则返回错误码
    if (init_linktype(&cstate, p) == -1) {
        rc = PCAP_ERROR;
        goto quit;
    }
    # 如果解析失败，则返回错误码
    if (pcap_parse(scanner, &cstate) != 0) {
#ifdef INET6
        // 如果定义了INET6，则释放地址信息
        if (cstate.ai != NULL)
            freeaddrinfo(cstate.ai);
#endif
        // 释放过滤器表达式
        if (cstate.e != NULL)
            free(cstate.e);
        // 设置错误码为PCAP_ERROR
        rc = PCAP_ERROR;
        // 跳转到退出标签
        goto quit;
    }

    // 如果过滤器根节点为空
    if (cstate.ic.root == NULL) {
        /*
         * 捕获gen_retblk()报告的错误
         */
        // 如果gen_retblk()报告错误，则跳转到退出标签
        if (setjmp(cstate.top_ctx)) {
            rc = PCAP_ERROR;
            goto quit;
        }
        // 生成过滤器块
        cstate.ic.root = gen_retblk(&cstate, cstate.snaplen);
    }

    // 如果需要优化并且不禁用优化
    if (optimize && !cstate.no_optimize) {
        // 对过滤器进行优化
        if (bpf_optimize(&cstate.ic, p->errbuf) == -1) {
            /* 失败 */
            rc = PCAP_ERROR;
            goto quit;
        }
        // 如果过滤器根节点为空或者拒绝所有数据包
        if (cstate.ic.root == NULL ||
            (cstate.ic.root->s.code == (BPF_RET|BPF_K) && cstate.ic.root->s.k == 0)) {
            (void)snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "expression rejects all packets");
            rc = PCAP_ERROR;
            goto quit;
        }
    }
    // 将中间代码转换为最终代码
    program->bf_insns = icode_to_fcode(&cstate.ic,
        cstate.ic.root, &len, p->errbuf);
    // 如果转换失败，则设置错误码为PCAP_ERROR
    if (program->bf_insns == NULL) {
        /* 失败 */
        rc = PCAP_ERROR;
        goto quit;
    }
    // 设置最终代码的长度
    program->bf_len = len;

    // 设置错误码为0，表示一切正常
    rc = 0;  /* 一切正常 */

quit:
    /*
     * 清理词法分析器的所有内容
     */
    // 如果输入缓冲区不为空，则删除输入缓冲区
    if (in_buffer != NULL)
        pcap__delete_buffer(in_buffer, scanner);
    // 如果词法分析器不为空，则销毁词法分析器
    if (scanner != NULL)
        pcap_lex_destroy(scanner);

    /*
     * 清理我们分配的内存
     */
    // 释放cstate中的所有内存块
    freechunks(&cstate);

    // 返回错误码
    return (rc);
}

/*
 * 使用没有打开pcap的编译器的入口点
 * 显式传入所有需要的东西
 */
int
pcap_compile_nopcap(int snaplen_arg, int linktype_arg,
            struct bpf_program *program,
         const char *buf, int optimize, bpf_u_int32 mask)
{
    pcap_t *p;
    int ret;

    // 打开一个虚拟的pcap
    p = pcap_open_dead(linktype_arg, snaplen_arg);
    // 如果打开失败，则返回错误码PCAP_ERROR
    if (p == NULL)
        return (PCAP_ERROR);
    # 使用给定的过滤程序编译过滤规则，将结果存储在 ret 中
    ret = pcap_compile(p, program, buf, optimize, mask);
    # 关闭 pcap 对象
    pcap_close(p);
    # 返回编译结果
    return (ret);
/*
 * 清理一个 "struct bpf_program"，释放其中分配的所有内存
 */
void
pcap_freecode(struct bpf_program *program)
{
    // 将 bf_len 设置为 0
    program->bf_len = 0;
    // 如果 bf_insns 不为空，则释放其内存并将其指针置为 NULL
    if (program->bf_insns != NULL) {
        free((char *)program->bf_insns);
        program->bf_insns = NULL;
    }
}

/*
 * 将 'list' 中的块回填到 'target'。'sense' 字段指示了 jt 和 jf 字段中哪个已经解析，哪个是指向另一个未解析块的指针（或者为 nil）。每个块中至少有一个字段已经解析。
 */
static void
backpatch(struct block *list, struct block *target)
{
    struct block *next;

    while (list) {
        if (!list->sense) {
            next = JT(list);
            JT(list) = target;
        } else {
            next = JF(list);
            JF(list) = target;
        }
        list = next;
    }
}

/*
 * 合并 b0 和 b1 中的列表，使用 'sense' 字段指示 jt 和 jf 哪个是链接。
 */
static void
merge(struct block *b0, struct block *b1)
{
    register struct block **p = &b0;

    /* 找到列表的末尾。 */
    while (*p)
        p = !((*p)->sense) ? &JT(*p) : &JF(*p);

    /* 连接这两个列表。 */
    *p = b1;
}

int
finish_parse(compiler_state_t *cstate, struct block *p)
{
    struct block *ppi_dlt_check;

    /*
     * 捕获我们和下面的例程报告的错误，并在出现错误时返回 -1。
     */
    if (setjmp(cstate->top_ctx))
        return (-1);
}
    /*
     * 在第一个（根）块的语句之前插入任何需要加载变量长度的可变长度头部的长度到寄存器的语句。
     *
     * XXX - 一个更复杂的策略是在使用这些长度并且没有使用它们的前任块的所有块的语句之前插入这些长度，这样我们只有在需要时才计算长度。
     * 可能还有更好的方法。
     *
     * 然而，这些策略会更加复杂，因为我们不会为程序没有使用长度的测试生成计算长度的代码，而且大多数测试可能会使用这些长度，所以我们只是推迟计算长度，
     * 这样就不会为早期失败的测试计算长度，而且这样做是否值得努力还不清楚。
     */
    insert_compute_vloffsets(cstate, p->head);

    /*
     * 对于 DLT_PPI 捕获，生成一个检查每个数据包的 DLT 值是否为 DLT_IEEE802_11 的检查。
     *
     * XXX - TurboCap 卡使用 DLT_PPI 用于以太网。
     * 我们是否可以定义一些 DLT_ETHERNET_WITH_PHDR 伪头部，其中包含适当的以太网信息，并使用它，而不是使用诸如 DLT_PPI 这样的东西，
     * 在那里你直到运行时才知道链路层头部类型，在一般情况下，这将迫使我们生成以太网 *和* 802.11 代码（*和* 任何其他使用 PPI 的东西），
     * 并在 BPF 程序的早期选择它们之间。
     */
    ppi_dlt_check = gen_ppi_dlt_check(cstate);
    if (ppi_dlt_check != NULL)
        gen_and(ppi_dlt_check, p);

    backpatch(p, gen_retblk(cstate, cstate->snaplen));
    p->sense = !p->sense;
    backpatch(p, gen_retblk(cstate, 0));
    cstate->ic.root = p->head;
    return (0);
# 生成并操作，将 b0 的尾部指向 b1 的头部
void
gen_and(struct block *b0, struct block *b1)
{
    # 将 b0 的尾部指向 b1 的头部
    backpatch(b0, b1->head);
    # 反转 b0 的 sense 值
    b0->sense = !b0->sense;
    # 反转 b1 的 sense 值
    b1->sense = !b1->sense;
    # 合并 b1 和 b0
    merge(b1, b0);
    # 再次反转 b1 的 sense 值
    b1->sense = !b1->sense;
    # 将 b1 的头部指向 b0 的头部
    b1->head = b0->head;
}

# 生成或操作，将 b0 的尾部指向 b1 的头部
void
gen_or(struct block *b0, struct block *b1)
{
    # 反转 b0 的 sense 值
    b0->sense = !b0->sense;
    # 将 b0 的尾部指向 b1 的头部
    backpatch(b0, b1->head);
    # 再次反转 b0 的 sense 值
    b0->sense = !b0->sense;
    # 合并 b1 和 b0
    merge(b1, b0);
    # 将 b1 的头部指向 b0 的头部
    b1->head = b0->head;
}

# 生成非操作，反转给定块的 sense 值
void
gen_not(struct block *b)
{
    # 反转给定块的 sense 值
    b->sense = !b->sense;
}

# 生成比较操作，返回一个块
static struct block *
gen_cmp(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, 0xffffffff, BPF_JEQ, 0, v);
}

# 生成大于比较操作，返回一个块
static struct block *
gen_cmp_gt(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, 0xffffffff, BPF_JGT, 0, v);
}

# 生成大于等于比较操作，返回一个块
static struct block *
gen_cmp_ge(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, 0xffffffff, BPF_JGE, 0, v);
}

# 生成小于比较操作，返回一个块
static struct block *
gen_cmp_lt(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, 0xffffffff, BPF_JGE, 1, v);
}

# 生成小于等于比较操作，返回一个块
static struct block *
gen_cmp_le(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, 0xffffffff, BPF_JGT, 1, v);
}

# 生成多条件比较操作，返回一个块
static struct block *
gen_mcmp(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 v, bpf_u_int32 mask)
{
    # 调用 gen_ncmp 函数，传入相应参数
    return gen_ncmp(cstate, offrel, offset, size, mask, BPF_JEQ, 0, v);
}

# 生成字节比较操作，返回一个块
static struct block *
gen_bcmp(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, const u_char *v)
{
    # 初始化两个块指针
    register struct block *b, *tmp;
    # 将 b 初始化为 NULL
    b = NULL;
    # 当 size 大于等于 4 时，循环执行以下操作
    while (size >= 4) {
        # 定义指针 p 指向 v 数组中倒数第四个元素
        register const u_char *p = &v[size - 4];
        # 调用 gen_cmp 函数，比较偏移量为 offset + size - 4 的位置处的四个字节数据与 cstate 的关系，生成比较结果
        tmp = gen_cmp(cstate, offrel, offset + size - 4, BPF_W, EXTRACT_BE_U_4(p));
        # 如果 b 不为空，则将 tmp 与 b 进行逻辑与操作
        if (b != NULL)
            gen_and(b, tmp);
        # 将 tmp 赋值给 b
        b = tmp;
        # size 减去 4
        size -= 4;
    }
    # 当 size 大于等于 2 时，循环执行以下操作
    while (size >= 2) {
        # 定义指针 p 指向 v 数组中倒数第二个元素
        register const u_char *p = &v[size - 2];
        # 调用 gen_cmp 函数，比较偏移量为 offset + size - 2 的位置处的两个字节数据与 cstate 的关系，生成比较结果
        tmp = gen_cmp(cstate, offrel, offset + size - 2, BPF_H, EXTRACT_BE_U_2(p));
        # 如果 b 不为空，则将 tmp 与 b 进行逻辑与操作
        if (b != NULL)
            gen_and(b, tmp);
        # 将 tmp 赋值给 b
        b = tmp;
        # size 减去 2
        size -= 2;
    }
    # 如果 size 大于 0，则执行以下操作
    if (size > 0) {
        # 调用 gen_cmp 函数，比较偏移量为 offset 的位置处的一个字节数据与 cstate 的关系，生成比较结果
        tmp = gen_cmp(cstate, offrel, offset, BPF_B, v[0]);
        # 如果 b 不为空，则将 tmp 与 b 进行逻辑与操作
        if (b != NULL)
            gen_and(b, tmp);
        # 将 tmp 赋值给 b
        b = tmp;
    }
    # 返回结果 b
    return b;
}

/*
 * AND the field of size "size" at offset "offset" relative to the header
 * specified by "offrel" with "mask", and compare it with the value "v"
 * with the test specified by "jtype"; if "reverse" is true, the test
 * should test the opposite of "jtype".
 */
static struct block *
gen_ncmp(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size, bpf_u_int32 mask, int jtype, int reverse,
    bpf_u_int32 v)
{
    struct slist *s, *s2;
    struct block *b;

    s = gen_load_a(cstate, offrel, offset, size);

    if (mask != 0xffffffff) {
        s2 = new_stmt(cstate, BPF_ALU|BPF_AND|BPF_K);
        s2->s.k = mask;
        sappend(s, s2);
    }

    b = new_block(cstate, JMP(jtype));
    b->stmts = s;
    b->s.k = v;
    if (reverse && (jtype == BPF_JGT || jtype == BPF_JGE))
        gen_not(b);
    return b;
}

static int
init_linktype(compiler_state_t *cstate, pcap_t *p)
{
    cstate->pcap_fddipad = p->fddipad;

    /*
     * We start out with only one link-layer header.
     */
    cstate->outermostlinktype = pcap_datalink(p);
    cstate->off_outermostlinkhdr.constant_part = 0;
    cstate->off_outermostlinkhdr.is_variable = 0;
    cstate->off_outermostlinkhdr.reg = -1;

    cstate->prevlinktype = cstate->outermostlinktype;
    cstate->off_prevlinkhdr.constant_part = 0;
    cstate->off_prevlinkhdr.is_variable = 0;
    cstate->off_prevlinkhdr.reg = -1;

    cstate->linktype = cstate->outermostlinktype;
    cstate->off_linkhdr.constant_part = 0;
    cstate->off_linkhdr.is_variable = 0;
    cstate->off_linkhdr.reg = -1;

    /*
     * XXX
     */
    cstate->off_linkpl.constant_part = 0;
    cstate->off_linkpl.is_variable = 0;
    cstate->off_linkpl.reg = -1;

    cstate->off_linktype.constant_part = 0;
    cstate->off_linktype.is_variable = 0;
    cstate->off_linktype.reg = -1;

    /*
     * Assume it's not raw ATM with a pseudo-header, for now.
     */
    cstate->is_atm = 0;
    cstate->off_vpi = OFFSET_NOT_SET;
}
    # 设置偏移量为未设置状态
    cstate->off_vci = OFFSET_NOT_SET;
    cstate->off_proto = OFFSET_NOT_SET;
    cstate->off_payload = OFFSET_NOT_SET;

    # 设置不是 Geneve 协议
    cstate->is_geneve = 0;

    # 默认情况下没有可变长度的 VLAN 偏移量
    cstate->is_vlan_vloffset = 0;

    # 假设不是 SS7 协议
    cstate->off_li = OFFSET_NOT_SET;
    cstate->off_li_hsl = OFFSET_NOT_SET;
    cstate->off_sio = OFFSET_NOT_SET;
    cstate->off_opc = OFFSET_NOT_SET;
    cstate->off_dpc = OFFSET_NOT_SET;
    cstate->off_sls = OFFSET_NOT_SET;

    # 初始化标签栈深度和 VLAN 栈深度
    cstate->label_stack_depth = 0;
    cstate->vlan_stack_depth = 0;

    # 根据链路类型进行不同的设置
    switch (cstate->linktype) {

    case DLT_ARCNET:
        # 设置 ARCNET 的常量部分偏移量
        cstate->off_linktype.constant_part = 2;
        cstate->off_linkpl.constant_part = 6;
        cstate->off_nl = 0;        # 实际上是可变的
        cstate->off_nl_nosnap = 0;    # 没有 802.2 LLC
        break;

    case DLT_ARCNET_LINUX:
        # 设置 ARCNET LINUX 的常量部分偏移量
        cstate->off_linktype.constant_part = 4;
        cstate->off_linkpl.constant_part = 8;
        cstate->off_nl = 0;        # 实际上是可变的
        cstate->off_nl_nosnap = 0;    # 没有 802.2 LLC
        break;

    case DLT_EN10MB:
        # 设置 EN10MB 的常量部分偏移量
        cstate->off_linktype.constant_part = 12;
        cstate->off_linkpl.constant_part = 14;    # 以太网头部长度
        cstate->off_nl = 0;        # 以太网 II
        cstate->off_nl_nosnap = 3;    # 802.3+802.2
        break;

    case DLT_SLIP:
        # SLIP 没有链路层类型，16 字节头部被硬编码到 SLIP 驱动程序中
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 16;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    # 没有 802.2 LLC
        break;
    case DLT_SLIP_BSDOS:
        /* 设置链路类型偏移量为未设置 */
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        /* 设置链路协议数据包头部偏移量为24 */
        cstate->off_linkpl.constant_part = 24;
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;

    case DLT_NULL:
    case DLT_LOOP:
        // 设置链路类型偏移量为0
        cstate->off_linktype.constant_part = 0;
        // 设置链路协议数据包头部偏移量为4
        cstate->off_linkpl.constant_part = 4;
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;

    case DLT_ENC:
        // 设置链路类型偏移量为0
        cstate->off_linktype.constant_part = 0;
        // 设置链路协议数据包头部偏移量为12
        cstate->off_linkpl.constant_part = 12;
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;

    case DLT_PPP:
    case DLT_PPP_PPPD:
    case DLT_C_HDLC:        /* BSD/OS Cisco HDLC */
    case DLT_HDLC:            /* NetBSD (Cisco) HDLC */
    case DLT_PPP_SERIAL:        /* NetBSD sync/async serial PPP */
        // 设置链路类型偏移量为2，跳过 HDLC 类型的帧
        cstate->off_linktype.constant_part = 2;    /* skip HDLC-like framing */
        // 设置链路协议数据包头部偏移量为4，跳过 HDLC 类型的帧和协议字段
        cstate->off_linkpl.constant_part = 4;    /* skip HDLC-like framing and protocol field */
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;

    case DLT_PPP_ETHER:
        /*
         * 这不包括以太网头部，仅包括会话状态。
         */
        // 设置链路类型偏移量为6
        cstate->off_linktype.constant_part = 6;
        // 设置链路协议数据包头部偏移量为8
        cstate->off_linkpl.constant_part = 8;
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;

    case DLT_PPP_BSDOS:
        // 设置链路类型偏移量为5
        cstate->off_linktype.constant_part = 5;
        // 设置链路协议数据包头部偏移量为24
        cstate->off_linkpl.constant_part = 24;
        // 设置网络层偏移量为0
        cstate->off_nl = 0;
        // 设置无 SNAP 的网络层偏移量为0
        cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */
        break;
    # 如果是 FDDI 类型的数据链路层，设置链路类型偏移量为 LLC 头的偏移量
    case DLT_FDDI:
        # FDDI 实际上没有一个链接级类型字段
        # 我们将 "off_linktype" 设置为 LLC 头的偏移量
        #
        # 要检查以太网类型，我们假设 SSAP = SNAP 被使用，并提取封装的以太网类型。
        # XXX - 我们应该生成代码来检查 SNAP 吗？
        cstate->off_linktype.constant_part = 13;
        cstate->off_linktype.constant_part += cstate->pcap_fddipad;
        cstate->off_linkpl.constant_part = 13;    /* FDDI MAC header length */
        cstate->off_linkpl.constant_part += cstate->pcap_fddipad;
        cstate->off_nl = 8;        /* 802.2+SNAP */
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        break;
    case DLT_IEEE802:
        /*
         * Token Ring doesn't really have a link-level type field.
         * We set "off_linktype" to the offset of the LLC header.
         *
         * To check for Ethernet types, we assume that SSAP = SNAP
         * is being used and pick out the encapsulated Ethernet type.
         * XXX - should we generate code to check for SNAP?
         *
         * XXX - the header is actually variable-length.
         * Some various Linux patched versions gave 38
         * as "off_linktype" and 40 as "off_nl"; however,
         * if a token ring packet has *no* routing
         * information, i.e. is not source-routed, the correct
         * values are 20 and 22, as they are in the vanilla code.
         *
         * A packet is source-routed iff the uppermost bit
         * of the first byte of the source address, at an
         * offset of 8, has the uppermost bit set.  If the
         * packet is source-routed, the total number of bytes
         * of routing information is 2 plus bits 0x1F00 of
         * the 16-bit value at an offset of 14 (shifted right
         * 8 - figure out which byte that is).
         */
        // 设置"off_linktype"为LLC头的偏移量
        cstate->off_linktype.constant_part = 14;
        // 设置"off_linkpl"为LLC头的偏移量，Token Ring MAC头的长度
        cstate->off_linkpl.constant_part = 14;    /* Token Ring MAC header length */
        // 设置"off_nl"为802.2+SNAP
        cstate->off_nl = 8;        /* 802.2+SNAP */
        // 设置"off_nl_nosnap"为802.2
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        // 跳出switch语句
        break;

    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
        // 设置"off_linkhdr.is_variable"为1，表示802.11的链路头是可变长度的
        cstate->off_linkhdr.is_variable = 1;
        // 继续执行，802.11没有可变的链路前缀，但其他方面是一样的
        /* Fall through, 802.11 doesn't have a variable link
         * prefix but is otherwise the same. */
        /* FALLTHROUGH */
    # 如果数据链路类型是 IEEE802_11
    case DLT_IEEE802_11:
        '''
        # 802.11实际上没有链路级类型字段。
        # 我们将"off_linktype.constant_part"设置为LLC标头的偏移量。
        # 为了检查以太网类型，我们假设使用SSAP = SNAP，并提取封装的以太网类型。
        # XXX - 我们是否应该生成代码来检查SNAP？
        # 我们还在这里处理可变长度的无线电标头。
        # Prism标头在理论上是可变长度的，但实际上总是144字节长。
        # 但是，Linux上的一些驱动程序使用ARPHRD_IEEE80211_PRISM，但有时或总是提供AVS标头，因此我们必须检查无线电标头是Prism标头还是AVS标头，因此在实践中，它是可变长度。
        '''
        # 设置链路类型常量部分的偏移量
        cstate->off_linktype.constant_part = 24;
        # 链路层标头是可变长度
        cstate->off_linkpl.constant_part = 0;
        cstate->off_linkpl.is_variable = 1;
        # 802.2+SNAP
        cstate->off_nl = 8;
        # 802.2
        cstate->off_nl_nosnap = 3;
        break;
    case DLT_PPI:
        /*
         * 目前我们将 PPI 与普通的 Radiotap 编码数据一样对待。不同之处在于生成代码的函数在计算头部长度时的不同。
         * 由于 PPI 的代码生成器仅支持裸的 802.11 封装（即封装的 DLT 应该是 DLT_IEEE802_11），我们也生成代码来检查这一点。
         */
        cstate->off_linktype.constant_part = 24;
        cstate->off_linkpl.constant_part = 0;    /* 链路层头部是可变长度的 */
        cstate->off_linkpl.is_variable = 1;
        cstate->off_linkhdr.is_variable = 1;
        cstate->off_nl = 8;        /* 802.2+SNAP */
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        break;

    case DLT_ATM_RFC1483:
    case DLT_ATM_CLIP:    /* Linux ATM 定义了这个 */
        /*
         * 假设路由的非 ISO PDU
         * （即，LLC = 0xAA-AA-03，OUT = 0x00-00-00）
         *
         * XXX - 那么 ISO PDU 怎么办，比如 CLNP、ISIS、ESIS，或者带有 PPP NLPID 的 PPP（例如 PPPoA）？后者可能会被视为 PPPoE 应该被处理的方式，所以你可以执行 "pppoe and udp port 2049" 或 "pppoa and tcp port 80" 并检查 PPPo{A,E} 和一个 PPP 协议的 IP 和...
         */
        cstate->off_linktype.constant_part = 0;
        cstate->off_linkpl.constant_part = 0;    /* 数据包以 LLC 头开始 */
        cstate->off_nl = 8;        /* 802.2+SNAP */
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        break;
    case DLT_SUNATM:
        /*
         * 全面的 ATM；您将获得带有 ATM 伪头的 AALn PDU。
         */
        cstate->is_atm = 1;
        cstate->off_vpi = SUNATM_VPI_POS;
        cstate->off_vci = SUNATM_VCI_POS;
        cstate->off_proto = PROTO_POS;
        cstate->off_payload = SUNATM_PKT_BEGIN_POS;
        cstate->off_linktype.constant_part = cstate->off_payload;
        cstate->off_linkpl.constant_part = cstate->off_payload;    /* 如果是 LLC 封装的 */
        cstate->off_nl = 8;        /* 802.2+SNAP */
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        break;

    case DLT_RAW:
    case DLT_IPV4:
    case DLT_IPV6:
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 0;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有 802.2 LLC */
        break;

    case DLT_LINUX_SLL:    /* Linux cooked socket v1 的伪头 */
        cstate->off_linktype.constant_part = 14;
        cstate->off_linkpl.constant_part = 16;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有 802.2 LLC */
        break;

    case DLT_LINUX_SLL2:    /* Linux cooked socket v2 的伪头 */
        cstate->off_linktype.constant_part = 0;
        cstate->off_linkpl.constant_part = 20;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有 802.2 LLC */
        break;

    case DLT_LTALK:
        /*
         * LocalTalk 在 LLAP 头中有一个 1 字节的类型字段，
         * 但实际上它只是指示是否有一个“短”或“长”DDP数据包跟随。
         */
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 0;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有 802.2 LLC */
        break;
    case DLT_IP_OVER_FC:
        /*
         * RFC 2625 IP-over-Fibre-Channel并没有真正的链路级类型字段。
         * 我们将"off_linktype"设置为LLC头的偏移量。
         *
         * 要检查以太网类型，我们假设使用SSAP = SNAP，并提取封装的以太网类型。
         * XXX - 我们应该生成代码来检查SNAP吗？RFC 2625说应该使用SNAP。
         */
        cstate->off_linktype.constant_part = 16;
        cstate->off_linkpl.constant_part = 16;
        cstate->off_nl = 8;        /* 802.2+SNAP */
        cstate->off_nl_nosnap = 3;    /* 802.2 */
        break;

    case DLT_FRELAY:
        /*
         * XXX - 我们应该设置这个来处理SNAP封装的帧（NLPID为0x80）。
         */
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 0;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有802.2 LLC */
        break;

                /*
                 * 唯一BPF感兴趣的FRF.16帧是非控制帧；
                 * Frame Relay具有可变长度的链路层，
                 * 所以我们现在从偏移量4开始，并稍后递增（FIXME）；
                 */
    case DLT_MFR:
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 0;
        cstate->off_nl = 4;
        cstate->off_nl_nosnap = 0;    /* XXX - 现在->没有802.2 LLC */
        break;

    case DLT_APPLE_IP_OVER_IEEE1394:
        cstate->off_linktype.constant_part = 16;
        cstate->off_linkpl.constant_part = 18;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;    /* 没有802.2 LLC */
        break;
    # 设置链接类型偏移量为6
    case DLT_SYMANTEC_FIREWALL:
        cstate->off_linktype.constant_part = 6;
        # 设置链接层协议数据偏移量为44
        cstate->off_linkpl.constant_part = 44;
        # 设置网络层协议数据偏移量为0（以太网 II）
        cstate->off_nl = 0;
        # 设置无 SNAP 协议的网络层协议数据偏移量为0（对于 802.3 数据包的处理）
        cstate->off_nl_nosnap = 0;
        break;

    # 设置链接类型偏移量为0
    case DLT_PFLOG:
        cstate->off_linktype.constant_part = 0;
        # 设置链接层协议数据偏移量为0（可变长度的链路层头部）
        cstate->off_linkpl.constant_part = 0;
        # 将链接层协议数据偏移量标记为可变
        cstate->off_linkpl.is_variable = 1;
        # 设置网络层协议数据偏移量为0
        cstate->off_nl = 0;
        # 设置无 SNAP 协议的网络层协议数据偏移量为0（没有 802.2 LLC）
        cstate->off_nl_nosnap = 0;
        break;

    # 设置链接类型偏移量为4
    case DLT_JUNIPER_MFR:
    case DLT_JUNIPER_MLFR:
    case DLT_JUNIPER_MLPPP:
    case DLT_JUNIPER_PPP:
    case DLT_JUNIPER_CHDLC:
    case DLT_JUNIPER_FRELAY:
        cstate->off_linktype.constant_part = 4;
        # 设置链接层协议数据偏移量为4
        cstate->off_linkpl.constant_part = 4;
        # 设置网络层协议数据偏移量为0
        cstate->off_nl = 0;
        # 设置无 SNAP 协议的网络层协议数据偏移量为 OFFSET_NOT_SET（没有 802.2 LLC）
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        break;

    # 设置链接类型偏移量为4（实际上在4-8之间可变）
    case DLT_JUNIPER_ATM1:
        cstate->off_linktype.constant_part = 4;
        # 设置链接层协议数据偏移量为4（实际上在4-8之间可变）
        cstate->off_linkpl.constant_part = 4;
        # 设置网络层协议数据偏移量为0
        cstate->off_nl = 0;
        # 设置无 SNAP 协议的网络层协议数据偏移量为10
        cstate->off_nl_nosnap = 10;
        break;

    # 设置链接类型偏移量为8（实际上在8-12之间可变）
    case DLT_JUNIPER_ATM2:
        cstate->off_linktype.constant_part = 8;
        # 设置链接层协议数据偏移量为8（实际上在8-12之间可变）
        cstate->off_linkpl.constant_part = 8;
        # 设置网络层协议数据偏移量为0
        cstate->off_nl = 0;
        # 设置无 SNAP 协议的网络层协议数据偏移量为10
        cstate->off_nl_nosnap = 10;
        break;

    # 设置链接层协议数据偏移量为14
    # 设置链接类型偏移量为16
    # 设置网络层协议数据偏移量为18（以太网 II）
    # 设置无 SNAP 协议的网络层协议数据偏移量为21（802.3+802.2）
    case DLT_JUNIPER_PPPOE:
    case DLT_JUNIPER_ETHER:
        cstate->off_linkpl.constant_part = 14;
        cstate->off_linktype.constant_part = 16;
        cstate->off_nl = 18;
        cstate->off_nl_nosnap = 21;
        break;
    # 设置链接类型偏移量为4，链路层协议偏移量为6，网络层偏移量为0，无 802.2 LLC
    case DLT_JUNIPER_PPPOE_ATM:
        cstate->off_linktype.constant_part = 4;
        cstate->off_linkpl.constant_part = 6;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    /* no 802.2 LLC */
        break;

    # 设置链接类型偏移量为6，链路层协议偏移量为12，网络层偏移量为0，无 802.2 LLC
    case DLT_JUNIPER_GGSN:
        cstate->off_linktype.constant_part = 6;
        cstate->off_linkpl.constant_part = 12;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    /* no 802.2 LLC */
        break;

    # 设置链接类型偏移量为6，链路层协议偏移量为 OFFSET_NOT_SET，网络层偏移量为 OFFSET_NOT_SET，无 802.2 LLC
    case DLT_JUNIPER_ES:
        cstate->off_linktype.constant_part = 6;
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;    /* not really a network layer but raw IP addresses */
        cstate->off_nl = OFFSET_NOT_SET;    /* not really a network layer but raw IP addresses */
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    /* no 802.2 LLC */
        break;

    # 设置链接类型偏移量为12，链路层协议偏移量为12，网络层偏移量为0，无 802.2 LLC
    case DLT_JUNIPER_MONITOR:
        cstate->off_linktype.constant_part = 12;
        cstate->off_linkpl.constant_part = 12;
        cstate->off_nl = 0;            /* raw IP/IP6 header */
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    /* no 802.2 LLC */
        break;

    # 设置链接类型偏移量为 OFFSET_NOT_SET，链路层协议偏移量为 OFFSET_NOT_SET，网络层偏移量为 OFFSET_NOT_SET，无 802.2 LLC
    case DLT_BACNET_MS_TP:
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        cstate->off_nl = OFFSET_NOT_SET;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        break;

    # 设置链接类型偏移量为12，链路层协议偏移量为 OFFSET_NOT_SET，网络层偏移量为 OFFSET_NOT_SET，无 802.2 LLC
    case DLT_JUNIPER_SERVICES:
        cstate->off_linktype.constant_part = 12;
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;    /* L3 proto location dep. on cookie type */
        cstate->off_nl = OFFSET_NOT_SET;    /* L3 proto location dep. on cookie type */
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    /* no 802.2 LLC */
        break;

    # 设置链接类型偏移量为18，链路层协议偏移量为 OFFSET_NOT_SET，网络层偏移量为 OFFSET_NOT_SET，无 802.2 LLC
    case DLT_JUNIPER_VP:
        cstate->off_linktype.constant_part = 18;
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        cstate->off_nl = OFFSET_NOT_SET;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        break;
    # 如果数据链路类型是 DLT_JUNIPER_ST
    case DLT_JUNIPER_ST:
        # 设置链路类型的偏移量为18
        cstate->off_linktype.constant_part = 18;
        # 设置链路数据包长度的偏移量为未设置
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        # 设置网络层偏移量为未设置
        cstate->off_nl = OFFSET_NOT_SET;
        # 设置无快照网络层偏移量为未设置
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束当前case
        break;

    # 如果数据链路类型是 DLT_JUNIPER_ISM
    case DLT_JUNIPER_ISM:
        # 设置链路类型的偏移量为8
        cstate->off_linktype.constant_part = 8;
        # 设置链路数据包长度的偏移量为未设置
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        # 设置网络层偏移量为未设置
        cstate->off_nl = OFFSET_NOT_SET;
        # 设置无快照网络层偏移量为未设置
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束当前case
        break;

    # 如果数据链路类型是 DLT_JUNIPER_VS, DLT_JUNIPER_SRX_E2E, DLT_JUNIPER_FIBRECHANNEL, DLT_JUNIPER_ATM_CEMIC
    case DLT_JUNIPER_VS:
    case DLT_JUNIPER_SRX_E2E:
    case DLT_JUNIPER_FIBRECHANNEL:
    case DLT_JUNIPER_ATM_CEMIC:
        # 设置链路类型的偏移量为8
        cstate->off_linktype.constant_part = 8;
        # 设置链路数据包长度的偏移量为未设置
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        # 设置网络层偏移量为未设置
        cstate->off_nl = OFFSET_NOT_SET;
        # 设置无快照网络层偏移量为未设置
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束当前case
        break;

    # 如果数据链路类型是 DLT_MTP2
    case DLT_MTP2:
        # 设置链路标识的偏移量为2
        cstate->off_li = 2;
        # 设置链路标识和信号单位长度的偏移量为4
        cstate->off_li_hsl = 4;
        # 设置信号单位标识的偏移量为3
        cstate->off_sio = 3;
        # 设置原始路径代码的偏移量为4
        cstate->off_opc = 4;
        # 设置目的路径代码的偏移量为4
        cstate->off_dpc = 4;
        # 设置信号信号序列号的偏移量为7
        cstate->off_sls = 7;
        # 设置链路类型的偏移量为未设置
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        # 设置链路数据包长度的偏移量为未设置
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        # 设置网络层偏移量为未设置
        cstate->off_nl = OFFSET_NOT_SET;
        # 设置无快照网络层偏移量为未设置
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束当前case
        break;

    # 如果数据链路类型是 DLT_MTP2_WITH_PHDR
    case DLT_MTP2_WITH_PHDR:
        # 设置链路标识的偏移量为6
        cstate->off_li = 6;
        # 设置链路标识和信号单位长度的偏移量为8
        cstate->off_li_hsl = 8;
        # 设置信号单位标识的偏移量为7
        cstate->off_sio = 7;
        # 设置原始路径代码的偏移量为8
        cstate->off_opc = 8;
        # 设置目的路径代码的偏移量为8
        cstate->off_dpc = 8;
        # 设置信号信号序列号的偏移量为11
        cstate->off_sls = 11;
        # 设置链路类型的偏移量为未设置
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        # 设置链路数据包长度的偏移量为未设置
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        # 设置网络层偏移量为未设置
        cstate->off_nl = OFFSET_NOT_SET;
        # 设置无快照网络层偏移量为未设置
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束当前case
        break;
    # 如果数据链路类型是 DLT_ERF
    case DLT_ERF:
        # 设置偏移量
        cstate->off_li = 22;
        cstate->off_li_hsl = 24;
        cstate->off_sio = 23;
        cstate->off_opc = 24;
        cstate->off_dpc = 24;
        cstate->off_sls = 27;
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        cstate->off_nl = OFFSET_NOT_SET;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束
        break;

    # 如果数据链路类型是 DLT_PFSYNC
    case DLT_PFSYNC:
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;
        cstate->off_linkpl.constant_part = 4;
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = 0;
        # 结束
        break;

    # 如果数据链路类型是 DLT_AX25_KISS
    case DLT_AX25_KISS:
        '''
        * 目前只支持原始的 "link[N:M]" 过滤。
        '''
        cstate->off_linktype.constant_part = OFFSET_NOT_SET;    # 变量，最小15，最大71，步长为7
        cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
        cstate->off_nl = OFFSET_NOT_SET;    # 变量，最小16，最大71，步长为7
        cstate->off_nl_nosnap = OFFSET_NOT_SET;    # 没有802.2 LLC
        # 结束
        break;

    # 如果数据链路类型是 DLT_IPNET
    case DLT_IPNET:
        cstate->off_linktype.constant_part = 1;
        cstate->off_linkpl.constant_part = 24;    # ipnet 头部长度
        cstate->off_nl = 0;
        cstate->off_nl_nosnap = OFFSET_NOT_SET;
        # 结束
        break;

    # 如果数据链路类型是 DLT_NETANALYZER
    case DLT_NETANALYZER:
        cstate->off_linkhdr.constant_part = 4;    # 以太网头部在4字节伪头部之后
        cstate->off_linktype.constant_part = cstate->off_linkhdr.constant_part + 12;
        cstate->off_linkpl.constant_part = cstate->off_linkhdr.constant_part + 14;    # 伪头部+以太网头部长度
        cstate->off_nl = 0;        # 以太网 II
        cstate->off_nl_nosnap = 3;    # 802.3+802.2
        # 结束
        break;
    # 如果数据链路类型为DLT_NETANALYZER_TRANSPARENT
    case DLT_NETANALYZER_TRANSPARENT:
        # 设置链路头部的偏移量为12，即MAC头部跳过4字节伪头部、前导码和SFD
        cstate->off_linkhdr.constant_part = 12;
        # 设置链路类型的偏移量为链路头部偏移量加12
        cstate->off_linktype.constant_part = cstate->off_linkhdr.constant_part + 12;
        # 设置链路数据的偏移量为链路头部偏移量加14，即伪头部+前导码+SFD+以太网头部长度
        cstate->off_linkpl.constant_part = cstate->off_linkhdr.constant_part + 14;
        # 设置网络层偏移量为0，表示以太网II协议
        cstate->off_nl = 0;
        # 设置无SNAP的网络层偏移量为3，表示802.3+802.2协议
        cstate->off_nl_nosnap = 3;
        # 结束当前case
        break;

    # 如果数据链路类型不在已知范围内
    default:
        # 如果数据链路类型在已知范围内，则只支持原始的"link[N:M]"过滤
        if (cstate->linktype >= DLT_MATCHING_MIN &&
            cstate->linktype <= DLT_MATCHING_MAX) {
            # 将链路类型的偏移量设置为未设置状态
            cstate->off_linktype.constant_part = OFFSET_NOT_SET;
            # 将链路数据的偏移量设置为未设置状态
            cstate->off_linkpl.constant_part = OFFSET_NOT_SET;
            # 将网络层偏移量设置为未设置状态
            cstate->off_nl = OFFSET_NOT_SET;
            # 将无SNAP的网络层偏移量设置为未设置状态
            cstate->off_nl_nosnap = OFFSET_NOT_SET;
        } else {
            # 如果数据链路类型不在已知范围内，则设置错误信息并返回-1
            bpf_set_error(cstate, "unknown data link type %d (min %d, max %d)",
                cstate->linktype, DLT_MATCHING_MIN, DLT_MATCHING_MAX);
            return (-1);
        }
        # 结束当前case
        break;
    }

    # 设置最外层链路头部偏移量和上一个链路头部偏移量为当前链路头部偏移量
    cstate->off_outermostlinkhdr = cstate->off_prevlinkhdr = cstate->off_linkhdr;
    # 返回0表示成功
    return (0);
}
/*
 * Load a value relative to the specified absolute offset.
 */
static struct slist *
gen_load_absoffsetrel(compiler_state_t *cstate, bpf_abs_offset *abs_offset,
    u_int offset, u_int size)
{
    struct slist *s, *s2;

    // 生成绝对偏移的变量部分
    s = gen_abs_offset_varpart(cstate, abs_offset);

    /*
     * If "s" is non-null, it has code to arrange that the X register
     * contains the variable part of the absolute offset, so we
     * generate a load relative to that, with an offset of
     * abs_offset->constant_part + offset.
     *
     * Otherwise, we can do an absolute load with an offset of
     * abs_offset->constant_part + offset.
     */
    if (s != NULL) {
        /*
         * "s" points to a list of statements that puts the
         * variable part of the absolute offset into the X register.
         * Do an indirect load, to use the X register as an offset.
         */
        s2 = new_stmt(cstate, BPF_LD|BPF_IND|size);
        s2->s.k = abs_offset->constant_part + offset;
        sappend(s, s2);
    } else {
        /*
         * There is no variable part of the absolute offset, so
         * just do an absolute load.
         */
        s = new_stmt(cstate, BPF_LD|BPF_ABS|size);
        s->s.k = abs_offset->constant_part + offset;
    }
    return s;
}

/*
 * Load a value relative to the beginning of the specified header.
 */
static struct slist *
gen_load_a(compiler_state_t *cstate, enum e_offrel offrel, u_int offset,
    u_int size)
{
    struct slist *s, *s2;

    /*
     * Squelch warnings from compilers that *don't* assume that
     * offrel always has a valid enum value and therefore don't
     * assume that we'll always go through one of the case arms.
     *
     * If we have a default case, compilers that *do* assume that
     * will then complain about the default case code being
     * unreachable.
     *
     * Damned if you do, damned if you don't.
     */
    s = NULL;

    switch (offrel) {
    # 如果是 OR_PACKET 类型的操作码
    case OR_PACKET:
        # 创建一个新的语句对象，加载绝对偏移量处的数据到寄存器中
        s = new_stmt(cstate, BPF_LD|BPF_ABS|size);
        # 设置加载的偏移量
        s->s.k = offset;
        break;

    # 如果是 OR_LINKHDR 类型的操作码
    case OR_LINKHDR:
        # 生成加载绝对偏移量相对于 linkhdr 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkhdr, offset, size);
        break;

    # 如果是 OR_PREVLINKHDR 类型的操作码
    case OR_PREVLINKHDR:
        # 生成加载绝对偏移量相对于 prevlinkhdr 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_prevlinkhdr, offset, size);
        break;

    # 如果是 OR_LLC 类型的操作码
    case OR_LLC:
        # 生成加载绝对偏移量相对于 linkpl 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkpl, offset, size);
        break;

    # 如果是 OR_PREVMPLSHDR 类型的操作码
    case OR_PREVMPLSHDR:
        # 生成加载绝对偏移量相对于 linkpl 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkpl, cstate->off_nl - 4 + offset, size);
        break;

    # 如果是 OR_LINKPL 类型的操作码
    case OR_LINKPL:
        # 生成加载绝对偏移量相对于 linkpl 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkpl, cstate->off_nl + offset, size);
        break;

    # 如果是 OR_LINKPL_NOSNAP 类型的操作码
    case OR_LINKPL_NOSNAP:
        # 生成加载绝对偏移量相对于 linkpl 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkpl, cstate->off_nl_nosnap + offset, size);
        break;

    # 如果是 OR_LINKTYPE 类型的操作码
    case OR_LINKTYPE:
        # 生成加载绝对偏移量相对于 linktype 偏移量的数据的语句对象
        s = gen_load_absoffsetrel(cstate, &cstate->off_linktype, offset, size);
        break;

    # 如果是 OR_TRAN_IPV4 类型的操作码
    case OR_TRAN_IPV4:
        '''
        加载 X 寄存器中的数据为 IPv4 头部的长度（加上链路层头部的偏移量，如果它之前有一个变长的头部，比如无线电头部），单位为字节。
        '''
        s = gen_loadx_iphdrlen(cstate);

        '''
        加载偏移量为 link-layer payload 的偏移量 + 相对于 link-layer payload 起始的偏移量 + IPv4 头部的长度 + 指定的偏移量的数据。
        
        如果 link-layer payload 的偏移量是可变的，那么该偏移量的可变部分包含在 X 寄存器的值中，我们将常量部分包含在加载的偏移量中。
        '''
        s2 = new_stmt(cstate, BPF_LD|BPF_IND|size);
        s2->s.k = cstate->off_linkpl.constant_part + cstate->off_nl + offset;
        sappend(s, s2);
        break;
    # 如果是 IPv6 协议的数据包
    case OR_TRAN_IPV6:
        # 生成加载绝对偏移相对偏移的操作，更新链路层头部的偏移量
        s = gen_load_absoffsetrel(cstate, &cstate->off_linkpl, cstate->off_nl + 40 + offset, size);
        # 跳出 switch 语句
        break;
    }
    # 返回操作结果
    return s;
}
/*
 * 生成代码，将IPv4头部的长度和链路层有效载荷偏移的可变部分的和加载到X寄存器中
 */
static struct slist *
gen_loadx_iphdrlen(compiler_state_t *cstate)
{
    struct slist *s, *s2;

    // 生成加载链路层有效载荷偏移的可变部分的绝对偏移的代码
    s = gen_abs_offset_varpart(cstate, &cstate->off_linkpl);
    if (s != NULL) {
        /*
         * 链路层有效载荷的偏移有一个可变部分。"s"指向一个语句列表，将该偏移的可变部分放入X寄存器中。
         *
         * 4*([k]&0xf)寻址模式不能使用，因为我们没有一个常量偏移，所以我们必须将问题中的值加载到A寄存器中，并将X寄存器中的值加上它。
         */
        s2 = new_stmt(cstate, BPF_LD|BPF_IND|BPF_B);
        s2->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
        sappend(s, s2);
        s2 = new_stmt(cstate, BPF_ALU|BPF_AND|BPF_K);
        s2->s.k = 0xf;
        sappend(s, s2);
        s2 = new_stmt(cstate, BPF_ALU|BPF_LSH|BPF_K);
        s2->s.k = 2;
        sappend(s, s2);

        /*
         * 现在A寄存器包含IP头部的长度。
         * 我们需要将链路层有效载荷偏移的可变部分（仍然在X寄存器中）加到它上，并将结果移动到X寄存器中。
         */
        sappend(s, new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X));
        sappend(s, new_stmt(cstate, BPF_MISC|BPF_TAX));
    } else {
        /*
         * 链路层有效载荷的偏移量是一个常数，
         * 因此没有生成代码来加载（不存在的）
         * 偏移量的变量部分。
         *
         * 这意味着我们可以使用 4*([k]&0xf) 寻址模式。
         * 加载 IPv4 头部的长度，它位于从链路层有效载荷的开头
         * 偏移量 cstate->off_nl 处，因此位于从原始数据包的开头
         * 偏移量 cstate->off_linkpl.constant_part + cstate->off_nl 处，
         * 使用该寻址模式。
         */
        s = new_stmt(cstate, BPF_LDX|BPF_MSH|BPF_B);
        s->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
    }
    return s;
# 生成一个无条件跳转的代码块
static struct block *
gen_uncond(compiler_state_t *cstate, int rsense)
{
    # 创建一个新的语句节点，加载立即数指令
    s = new_stmt(cstate, BPF_LD|BPF_IMM);
    # 设置立即数的值为 rsense 的取反
    s->s.k = !rsense;
    # 创建一个新的代码块，跳转条件为 BPF_JEQ
    b = new_block(cstate, JMP(BPF_JEQ));
    # 将新创建的语句节点添加到代码块的语句列表中
    b->stmts = s;

    # 返回生成的代码块
    return b;
}

# 生成一个条件为真的代码块
static inline struct block *
gen_true(compiler_state_t *cstate)
{
    # 调用 gen_uncond 函数，传入参数 1
    return gen_uncond(cstate, 1);
}

# 生成一个条件为假的代码块
static inline struct block *
gen_false(compiler_state_t *cstate)
{
    # 调用 gen_uncond 函数，传入参数 0
    return gen_uncond(cstate, 0);
}

# 定义一个宏，用于对 32 位数字进行字节交换
#define    SWAPLONG(y) \
((((y)&0xff)<<24) | (((y)&0xff00)<<8) | (((y)&0xff0000)>>8) | (((y)>>24)&0xff))

# 生成匹配特定数据包类型的代码
static struct block *
gen_ether_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    # 定义两个代码块指针
    struct block *b0, *b1;

    # 根据 ll_proto 的值进行分支判断
    switch (ll_proto) {

    case LLCSAP_ISONS:
    case LLCSAP_IP:
    case LLCSAP_NETBEUI:
        # 生成比较大于指定值的代码块
        b0 = gen_cmp_gt(cstate, OR_LINKTYPE, 0, BPF_H, ETHERMTU);
        # 生成取反代码块
        gen_not(b0);
        # 生成比较指定值的代码块
        b1 = gen_cmp(cstate, OR_LLC, 0, BPF_H, (ll_proto << 8) | ll_proto);
        # 生成与操作代码块
        gen_and(b0, b1);
        # 返回生成的代码块
        return b1;

    case ETHERTYPE_ATALK:
    # 如果以太网类型为AARP
    case ETHERTYPE_AARP:
        '''
         * EtherTalk (AppleTalk protocols on Ethernet link
         * layer) may use 802.2 encapsulation.
         * EtherTalk（以太网链路上的AppleTalk协议）可能使用802.2封装。
         '''

        '''
         * Check for 802.2 encapsulation (EtherTalk phase 2?);
         * we check for an Ethernet type field less than
         * 1500, which means it's an 802.3 length field.
         * 检查802.2封装（EtherTalk第2阶段？）；
         * 我们检查以太网类型字段是否小于1500，这意味着它是一个802.3长度字段。
         '''
        b0 = gen_cmp_gt(cstate, OR_LINKTYPE, 0, BPF_H, ETHERMTU);
        gen_not(b0);

        '''
         * 802.2-encapsulated ETHERTYPE_ATALK packets are
         * SNAP packets with an organization code of
         * 0x080007 (Apple, for Appletalk) and a protocol
         * type of ETHERTYPE_ATALK (Appletalk).
         *
         * 802.2-encapsulated ETHERTYPE_AARP packets are
         * SNAP packets with an organization code of
         * 0x000000 (encapsulated Ethernet) and a protocol
         * type of ETHERTYPE_AARP (Appletalk ARP).
         * 802.2封装的ETHERTYPE_ATALK数据包是
         * 具有组织代码为0x080007（Apple，用于Appletalk）和协议类型为ETHERTYPE_ATALK（Appletalk）的SNAP数据包。
         * 802.2封装的ETHERTYPE_AARP数据包是
         * 具有组织代码为0x000000（封装的以太网）和协议类型为ETHERTYPE_AARP（Appletalk ARP）的SNAP数据包。
         '''
        if (ll_proto == ETHERTYPE_ATALK)
            b1 = gen_snap(cstate, 0x080007, ETHERTYPE_ATALK);
        else    /* ll_proto == ETHERTYPE_AARP */
            b1 = gen_snap(cstate, 0x000000, ETHERTYPE_AARP);
        gen_and(b0, b1);

        '''
         * Check for Ethernet encapsulation (Ethertalk
         * phase 1?); we just check for the Ethernet
         * protocol type.
         * 检查以太网封装（Ethertalk第1阶段？）；我们只检查以太网协议类型。
         '''
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);

        gen_or(b0, b1);
        return b1;
    # 默认情况下
    default:
        # 如果逻辑链路控制协议小于等于以太网最大传输单元
        if (ll_proto <= ETHERMTU) {
            """
            这是一个逻辑链路控制SAP值，因此匹配的帧将是802.2帧。
            检查帧是否是802.2帧（即长度/类型字段是长度字段，<= ETHERMTU），然后检查DSAP。
            """
            # 生成比较大于操作，比较链路类型是否大于0
            b0 = gen_cmp_gt(cstate, OR_LINKTYPE, 0, BPF_H, ETHERMTU);
            # 生成非操作
            gen_not(b0);
            # 生成比较操作，比较链路类型是否等于ll_proto
            b1 = gen_cmp(cstate, OR_LINKTYPE, 2, BPF_B, ll_proto);
            # 生成与操作
            gen_and(b0, b1);
            # 返回比较结果
            return b1;
        } else {
            """
            这是一个以太网类型，因此将长度/类型字段与之进行比较
            （如果帧是802.2帧，则长度字段将<= ETHERMTU，
            而且"ll_proto" > ETHERMTU，这个测试将失败，帧将不匹配，这是我们想要的）。
            """
            # 返回比较结果
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);
        }
    }
# 生成一个与回环接口相关的链路类型的数据块
static struct block *
gen_loopback_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    """
    # 对于 DLT_NULL，链路层头部是一个32位的字，其中包含主机字节顺序中的 AF_ 值，对于 DLT_ENC，链路层头部以主机字节顺序中的 AF_ 值开始。
    # 此外，如果我们正在读取一个保存的捕获文件，捕获中的主机字节顺序可能与此机器上的主机字节顺序不同。
    # 对于 DLT_LOOP，链路层头部是一个32位字，其中包含网络字节顺序中的 AF_ 值。
    """
    if (cstate->linktype == DLT_NULL || cstate->linktype == DLT_ENC) {
        """
        # AF_ 值以主机字节顺序表示，但是 BPF 解释器将其转换为网络字节顺序。
        # 如果这是一个保存文件，并且它来自与我们相反字节顺序的机器，我们将对 AF_ 值进行字节交换。
        # 然后我们通过 "htonl()" 运行它，并生成用于与结果进行比较的代码。
        """
        if (cstate->bpf_pcap->rfile != NULL && cstate->bpf_pcap->swapped)
            ll_proto = SWAPLONG(ll_proto);
        ll_proto = htonl(ll_proto);
    }
    return (gen_cmp(cstate, OR_LINKHDR, 0, BPF_W, ll_proto));
}

"""
# "proto" 是以太网类型值，对于 IPNET，如果它不是 IPv4 或 IPv6，则会出现错误。
static struct block *
gen_ipnet_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    switch (ll_proto) {
        case ETHERTYPE_IP:
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, IPH_AF_INET);
            # 不会到达此处
        case ETHERTYPE_IPV6:
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, IPH_AF_INET6);
            # 不会到达此处
        default:
            break;
    }
    return gen_false(cstate);
}
/*
 * 生成代码以匹配特定的数据包类型。
 *
 * “ll_proto”是以太网类型值，如果大于ETHERMTU，则为LLC SAP值，如果小于或等于ETHERMTU。我们使用它来确定是匹配类型字段还是检查类型字段是否具有特殊的LINUX_SLL_P_802_2值，然后执行适当的测试。
 */
static struct block *
gen_linux_sll_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    struct block *b0, *b1;

    switch (ll_proto) {

    case LLCSAP_ISONS:
    case LLCSAP_IP:
    case LLCSAP_NETBEUI:
        /*
         * OSI协议和NetBEUI始终使用802.2封装，因此我们检查DSAP和SSAP。
         *
         * LLCSAP_IP检查IP-over-802.2，而不是IP-over-Ethernet或IP-over-SNAP。
         *
         * XXX - 我们应该像这样同时检查DSAP和SSAP，还是应该像我们对其他类型<= ETHERMTU（即其他SAP值）所做的那样，只检查DSAP？
         */
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, LINUX_SLL_P_802_2);
        b1 = gen_cmp(cstate, OR_LLC, 0, BPF_H, (ll_proto << 8) | ll_proto);
        gen_and(b0, b1);
        return b1;
    # 对于 LLCSAP_IPX 类型的数据包进行处理
    case LLCSAP_IPX:
        '''
        *    Ethernet_II frames, which are Ethernet
        *    frames with a frame type of ETHERTYPE_IPX;
        *
        *    Ethernet_802.3 frames, which have a frame
        *    type of LINUX_SLL_P_802_3;
        *
        *    Ethernet_802.2 frames, which are 802.3
        *    frames with an 802.2 LLC header (i.e, have
        *    a frame type of LINUX_SLL_P_802_2) and
        *    with the IPX LSAP as the DSAP in the LLC
        *    header;
        *
        *    Ethernet_SNAP frames, which are 802.3
        *    frames with an LLC header and a SNAP
        *    header and with an OUI of 0x000000
        *    (encapsulated Ethernet) and a protocol
        *    ID of ETHERTYPE_IPX in the SNAP header.
        *
        * First, do the checks on LINUX_SLL_P_802_2
        * frames; generate the check for either
        * Ethernet_802.2 or Ethernet_SNAP frames, and
        * then put a check for LINUX_SLL_P_802_2 frames
        * before it.
        '''
        # 生成比较操作码，比较 LLC 类型的数据包是否为 LLCSAP_IPX
        b0 = gen_cmp(cstate, OR_LLC, 0, BPF_B, LLCSAP_IPX);
        # 生成 SNAP 操作码，比较 OUI 和协议 ID 是否为 ETHERTYPE_IPX
        b1 = gen_snap(cstate, 0x000000, ETHERTYPE_IPX);
        # 生成逻辑或操作码，将之前的比较结果与 SNAP 操作码进行逻辑或
        gen_or(b0, b1);
        # 生成比较操作码，比较链路类型是否为 LINUX_SLL_P_802_2
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, LINUX_SLL_P_802_2);
        # 生成逻辑与操作码，将之前的比较结果与链路类型比较结果进行逻辑与
        gen_and(b0, b1);

        '''
        * Now check for 802.3 frames and OR that with
        * the previous test.
        '''
        # 生成比较操作码，比较链路类型是否为 LINUX_SLL_P_802_3
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, LINUX_SLL_P_802_3);
        # 生成逻辑或操作码，将之前的比较结果与链路类型比较结果进行逻辑或
        gen_or(b0, b1);

        '''
        * Now add the check for Ethernet_II frames, and
        * do that before checking for the other frame
        * types.
        '''
        # 生成比较操作码，比较链路类型是否为 ETHERTYPE_IPX
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ETHERTYPE_IPX);
        # 生成逻辑或操作码，将之前的比较结果与链路类型比较结果进行逻辑或
        gen_or(b0, b1);
        # 返回最终的操作码
        return b1;

    case ETHERTYPE_ATALK:
    # 如果以太网类型是AARP
    case ETHERTYPE_AARP:
        '''
         * EtherTalk (AppleTalk protocols on Ethernet link
         * layer) may use 802.2 encapsulation.
         * EtherTalk（以太网链路上的苹果Talk协议）可能使用802.2封装。
         '''

        '''
         * Check for 802.2 encapsulation (EtherTalk phase 2?);
         * we check for the 802.2 protocol type in the
         * "Ethernet type" field.
         * 检查802.2封装（EtherTalk第2阶段？）；
         * 我们在“以太网类型”字段中检查802.2协议类型。
         '''
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, LINUX_SLL_P_802_2);

        '''
         * 802.2-encapsulated ETHERTYPE_ATALK packets are
         * SNAP packets with an organization code of
         * 0x080007 (Apple, for Appletalk) and a protocol
         * type of ETHERTYPE_ATALK (Appletalk).
         *
         * 802.2-encapsulated ETHERTYPE_AARP packets are
         * SNAP packets with an organization code of
         * 0x000000 (encapsulated Ethernet) and a protocol
         * type of ETHERTYPE_AARP (Appletalk ARP).
         * 802.2封装的ETHERTYPE_ATALK数据包是具有组织代码0x080007（Apple，用于Appletalk）和协议类型为ETHERTYPE_ATALK（Appletalk）的SNAP数据包。
         * 802.2封装的ETHERTYPE_AARP数据包是具有组织代码0x000000（封装以太网）和协议类型为ETHERTYPE_AARP（Appletalk ARP）的SNAP数据包。
         '''
        if (ll_proto == ETHERTYPE_ATALK)
            b1 = gen_snap(cstate, 0x080007, ETHERTYPE_ATALK);
        else    /* ll_proto == ETHERTYPE_AARP */
            b1 = gen_snap(cstate, 0x000000, ETHERTYPE_AARP);
        gen_and(b0, b1);

        '''
         * Check for Ethernet encapsulation (Ethertalk
         * phase 1?); we just check for the Ethernet
         * protocol type.
         * 检查以太网封装（Ethertalk第1阶段？）；我们只检查以太网协议类型。
         '''
        b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);

        gen_or(b0, b1);
        return b1;
    default:
        if (ll_proto <= ETHERMTU) {
            /*
             * 如果 ll_proto 小于等于 ETHERMTU，则这是一个 LLC SAP 值，
             * 因此匹配的帧将是 802.2 帧。
             * 检查“以太网类型”字段中的 802.2 协议类型，
             * 然后检查 DSAP。
             */
            b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, LINUX_SLL_P_802_2);
            b1 = gen_cmp(cstate, OR_LINKHDR, cstate->off_linkpl.constant_part, BPF_B,
                 ll_proto);
            gen_and(b0, b1);
            return b1;
        } else {
            /*
             * 这是一个以太网类型，因此将其与长度/类型字段进行比较
             * （如果帧是 802.2 帧，则长度字段将小于等于 ETHERMTU，
             * 而“ll_proto”大于 ETHERMTU，这个测试将失败，帧将不匹配，
             * 这正是我们想要的）。
             */
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);
        }
    }
/*
 * 从链路层头部的 pflog 头部之后加载一个值
 */
static struct slist *
gen_load_pflog_llprefixlen(compiler_state_t *cstate)
{
    struct slist *s1, *s2;

    /*
     * 生成代码，将 pflog 头部的长度加载到分配给该长度的寄存器中
     * （如果已经分配了寄存器，则生成代码将长度加载到该寄存器中）
     * （如果没有分配寄存器，则我们生成的代码不使用该前缀，因此不需要生成任何加载代码）
     */
    if (cstate->off_linkpl.reg != -1) {
        /*
         * 长度在头部的第一个字节中
         */
        s1 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        s1->s.k = 0;

        /*
         * 将其向上舍入到4的倍数
         * 加3，然后清除低2位
         */
        s2 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
        s2->s.k = 3;
        sappend(s1, s2);
        s2 = new_stmt(cstate, BPF_ALU|BPF_AND|BPF_K);
        s2->s.k = 0xfffffffc;
        sappend(s1, s2);

        /*
         * 现在分配一个寄存器来保存该值并存储它
         */
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linkpl.reg;
        sappend(s1, s2);

        /*
         * 现在将其移动到 X 寄存器中
         */
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        return (s1);
    } else
        return (NULL);
}

/*
 * 生成加载 prism 头部的长度的代码
 */
static struct slist *
gen_load_prism_llprefixlen(compiler_state_t *cstate)
{
    struct slist *s1, *s2;
    struct slist *sjeq_avs_cookie;
    struct slist *sjcommon;

    /*
     * 此代码与优化器不兼容，因为我们在正常指令列表中生成 jmp 指令
     */
    cstate->no_optimize = 1;
}
    /*
     * 生成代码以将无线电头的长度加载到分配给该长度的寄存器中，如果已经分配了寄存器。
     * （如果没有分配寄存器，我们生成的代码中没有使用该前缀，因此我们不需要生成任何加载它的代码。）
     *
     * 一些 Linux 驱动程序使用 ARPHRD_IEEE80211_PRISM，但有时或总是使用 AVS 头而不是 Prism 头。
     * 我们在原始数据包的开头加载一个 4 字节的大端值，并查看是否与 0xFFFFF000 掩码相与后等于 0x80211000。
     * 如果是，那就表示它是 AVS 头（被掩码掉的位是版本号）。否则，它是 Prism 头。
     *
     * XXX - 理论上，Prism 头也是可变长度的，但没有已知的软件生成不是 144 字节长的头。
     */
    } else
        return (NULL);
}
// 生成加载 AVS 头长度的代码，如果已经分配了寄存器来保存长度
static struct slist *
gen_load_avs_llprefixlen(compiler_state_t *cstate)
{
    struct slist *s1, *s2;

    /*
     * 生成代码，将 AVS 头的长度加载到分配的寄存器中，如果已经分配了的话
     * （如果没有分配，我们生成的代码中也不使用该前缀，所以不需要生成任何加载代码）
     */
    if (cstate->off_linkhdr.reg != -1) {
        /*
         * 从 AVS 头的偏移 4 处开始的 4 个字节是 AVS 头的长度
         * 该字段是大端序
         */
        s1 = new_stmt(cstate, BPF_LD|BPF_W|BPF_ABS);
        s1->s.k = 4;

        /*
         * 现在分配一个寄存器来保存该值并将其存储
         */
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linkhdr.reg;
        sappend(s1, s2);

        /*
         * 现在将其移动到 X 寄存器中
         */
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        return (s1);
    } else
        return (NULL);
}

// 生成加载 radiotap 头长度的代码，如果已经分配了寄存器来保存长度
static struct slist *
gen_load_radiotap_llprefixlen(compiler_state_t *cstate)
{
    struct slist *s1, *s2;

    /*
     * 生成代码，将 radiotap 头的长度加载到分配的寄存器中，如果已经分配了的话
     * （如果没有分配，我们生成的代码中也不使用该前缀，所以不需要生成任何加载代码）
     */
    if (cstate->off_linkhdr.reg != -1) {
        /*
         * 如果链路头部的偏移寄存器不等于-1，则执行以下操作
         */

        /*
         * 从 radiotap 头部的开始位置偏移2和3处的2个字节是 radiotap 头部的长度；
         * 不幸的是，它是小端字节序，所以我们必须逐字节加载并构造值。
         */

        /*
         * 加载高位字节，在偏移3处，左移一个字节，并将结果放入 X 寄存器。
         */
        s1 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        s1->s.k = 3;
        s2 = new_stmt(cstate, BPF_ALU|BPF_LSH|BPF_K);
        sappend(s1, s2);
        s2->s.k = 8;
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        /*
         * 加载下一个字节，在偏移2处，并将 X 寄存器中的值与其进行或操作。
         */
        s2 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        sappend(s1, s2);
        s2->s.k = 2;
        s2 = new_stmt(cstate, BPF_ALU|BPF_OR|BPF_X);
        sappend(s1, s2);

        /*
         * 现在分配一个寄存器来保存该值并存储它。
         */
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linkhdr.reg;
        sappend(s1, s2);

        /*
         * 现在将其移入 X 寄存器。
         */
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        return (s1);
    } else
        return (NULL);
    /*
     * 如果链路头部的偏移寄存器等于-1，则返回空指针
     */
}
/*
 * 目前，我们将 PPI 视为普通的 Radiotap 编码数据包。不同之处在于生成代码的函数，用于计算头部长度的代码生成器。由于 PPI 的代码生成器仅支持裸 802.11 封装（即封装的 DLT 应为 DLT_IEEE802_11），我们还生成代码来检查这一点；这是在 finish_parse() 中完成的。
 */
static struct slist *
gen_load_ppi_llprefixlen(compiler_state_t *cstate)
{
    struct slist *s1, *s2;

    /*
     * 生成代码，将 radiotap 头部的长度加载到分配给该长度的寄存器中，如果已经分配了寄存器。
     */
    if (cstate->off_linkhdr.reg != -1) {
        /*
         * 如果链路头部的偏移寄存器不等于-1，则执行以下操作
         */

        /*
         * 从 radiotap 头部的开始位置偏移2和3处的2个字节是 radiotap 头部的长度；
         * 不幸的是，它是小端字节序，所以我们必须逐字节加载并构造值。
         */

        /*
         * 加载高位字节，在偏移3处，左移一个字节，并将结果放入 X 寄存器。
         */
        s1 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        s1->s.k = 3;
        s2 = new_stmt(cstate, BPF_ALU|BPF_LSH|BPF_K);
        sappend(s1, s2);
        s2->s.k = 8;
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        /*
         * 加载下一个字节，在偏移2处，并将 X 寄存器中的值与其进行或操作。
         */
        s2 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        sappend(s1, s2);
        s2->s.k = 2;
        s2 = new_stmt(cstate, BPF_ALU|BPF_OR|BPF_X);
        sappend(s1, s2);

        /*
         * 现在分配一个寄存器来保存该值并存储它。
         */
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linkhdr.reg;
        sappend(s1, s2);

        /*
         * 现在将其移入 X 寄存器。
         */
        s2 = new_stmt(cstate, BPF_MISC|BPF_TAX);
        sappend(s1, s2);

        return (s1);
    } else
        return (NULL);
    /*
     * 如果链路头部的偏移寄存器等于-1，则返回空值
     */
}
/*
 * 从802.11头部之后的链路层头部加载一个值，即LLC_SNAP。
 * 链路层头部不一定从数据包的开头开始；可能有一个包含无线电信息的可变长度前缀。
 */
static struct slist *
gen_load_802_11_header_len(compiler_state_t *cstate, struct slist *s, struct slist *snext)
{
    struct slist *s2;
    struct slist *sjset_data_frame_1;
    struct slist *sjset_data_frame_2;
    struct slist *sjset_qos;
    struct slist *sjset_radiotap_flags_present;
    struct slist *sjset_radiotap_ext_present;
    struct slist *sjset_radiotap_tsft_present;
    struct slist *sjset_tsft_datapad, *sjset_notsft_datapad;
    struct slist *s_roundup;

    if (cstate->off_linkpl.reg == -1) {
        /*
         * 尚未为链路层负载的偏移量分配寄存器，这意味着没有人需要它；不必计算它 - 只需返回我们已经有的内容。
         */
        return (s);
    }

    /*
     * 此代码与优化器不兼容，因为我们在正常的指令slist中生成jmp指令
     */
    cstate->no_optimize = 1;

    /*
     * 如果"s"非空，则它包含代码，以确保X寄存器包含链路层头部之前的前缀的长度。
     *
     * 否则，链路层头部之前的前缀的长度为"off_outermostlinkhdr.constant_part"。
     */
    # 如果 s 为空指针
    if (s == NULL) {
        # 没有变长头部在链路层头部之前
        # 将固定长度前缀的长度加载到 X 寄存器中，并存储在 cstate->off_linkpl.reg 寄存器中
        # 长度为 off_outermostlinkhdr.constant_part
        s = new_stmt(cstate, BPF_LDX|BPF_IMM);
        s->s.k = cstate->off_outermostlinkhdr.constant_part;
    }

    # X 寄存器包含链路层头部开始的偏移量；将其加上 24，这是数据帧的最小 MAC 头部长度，并存储在 cstate->off_linkpl.reg 中
    # 然后加载位于 X 寄存器偏移量处的 Frame Control 字段，使用索引加载
    s2 = new_stmt(cstate, BPF_MISC|BPF_TXA);
    sappend(s, s2);
    s2 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s2->s.k = 24;
    sappend(s, s2);
    s2 = new_stmt(cstate, BPF_ST);
    s2->s.k = cstate->off_linkpl.reg;
    sappend(s, s2);

    # 加载 Frame Control 字段，查看是否为数据帧；数据帧的 Frame Control 字段中 0x08 位（b3）被设置，0x04 位（b2）被清除
    s2 = new_stmt(cstate, BPF_LD|BPF_IND|BPF_B);
    s2->s.k = 0;
    sappend(s, s2);

    # 如果 b3 被设置，测试 b2，否则跳转到程序的其余部分的第一条语句
    sjset_data_frame_1 = new_stmt(cstate, JMP(BPF_JSET));
    sjset_data_frame_1->s.k = 0x08;
    sappend(s, sjset_data_frame_1);

    # 如果 b2 未设置，这是一个数据帧；测试 QoS 位
    # 否则，跳转到程序的其余部分的第一条语句
    # 设置数据帧的第二个字节为 s.jt
    sjset_data_frame_2->s.jt = snext;
    # 设置数据帧的第一个字节为 sjset_qos，并创建一个新的语句对象
    sjset_data_frame_2->s.jf = sjset_qos = new_stmt(cstate, JMP(BPF_JSET));
    # 设置 QoS 位为 0x80
    sjset_qos->s.k = 0x80;    /* QoS bit */
    # 将 sjset_qos 添加到语句列表中
    sappend(s, sjset_qos);

    """
     如果设置了 QoS 位，则将 cstate->off_linkpl.reg 加 2，以跳过 QoS 字段。
     否则，转到程序的其余部分的第一个语句。
    """
    # 如果设置了 QoS 位，则将 cstate->off_linkpl.reg 加 2，以跳过 QoS 字段
    sjset_qos->s.jt = s2 = new_stmt(cstate, BPF_LD|BPF_MEM);
    s2->s.k = cstate->off_linkpl.reg;
    sappend(s, s2);
    s2 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_IMM);
    s2->s.k = 2;
    sappend(s, s2);
    s2 = new_stmt(cstate, BPF_ST);
    s2->s.k = cstate->off_linkpl.reg;
    sappend(s, s2);

    """
     如果我们有一个 radiotap 头部，查看它以查看 MAC 层头部和有效载荷之间是否有 Atheros 填充。
     
     注意：radiotap 头部中的所有字段都是小端字节序，因此我们需要对所有测试的值进行字节交换，因为它们将被加载为大端字节序的值。
     
     XXX - 在一般情况下，如果存在多个字的存在位，则我们将不得不扫描*所有*存在位。这将需要一个循环，这意味着我们将无法在内核中运行过滤器。
     
     我们在这里假设插入恼人填充的 Atheros 适配器没有多个天线，因此不会生成具有多个存在字的 radiotap 头部。
    """
    # 如果没有 radiotap 头部，则将 sjset_qos->s.jf 设置为 snext
    } else
        sjset_qos->s.jf = snext;

    # 返回语句列表
    return s;
    # 插入计算偏移量的函数，接受编译器状态和块作为参数
static void
insert_compute_vloffsets(compiler_state_t *cstate, struct block *b)
{
    struct slist *s;

    /* 存在链路负载和链路头之间的隐式依赖关系，因为负载计算包括头的可变部分。因此，如果没有其他人为链路头分配寄存器并且我们需要它，现在就分配。 */
    if (cstate->off_linkpl.reg != -1 && cstate->off_linkhdr.is_variable &&
        cstate->off_linkhdr.reg == -1)
        cstate->off_linkhdr.reg = alloc_reg(cstate);

    """
    对于具有可变长度头的链路层类型，生成代码以将链路层头的偏移量加载到分配给该偏移量的寄存器中（如果有的话）。
    XXX - 这个和下一个 switch 语句，无法处理在其他协议栈中封装 802.11 或 802.11+radio 信息的情况。这会更加复杂。
    """
    switch (cstate->outermostlinktype) {

    case DLT_PRISM_HEADER:
        s = gen_load_prism_llprefixlen(cstate);
        break;

    case DLT_IEEE802_11_RADIO_AVS:
        s = gen_load_avs_llprefixlen(cstate);
        break;

    case DLT_IEEE802_11_RADIO:
        s = gen_load_radiotap_llprefixlen(cstate);
        break;

    case DLT_PPI:
        s = gen_load_ppi_llprefixlen(cstate);
        break;

    default:
        s = NULL;
        break;
    }

    """
    对于具有可变长度链路层头的链路层类型，生成代码以将链路层负载的偏移量加载到分配给该偏移量的寄存器中（如果有的话）。
    """
    switch (cstate->outermostlinktype) {

    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
    case DLT_PPI:
        s = gen_load_802_11_header_len(cstate, s, b->stmts);
        break;
    # 如果数据链路类型为 PFLOG，则生成加载 PFLOG 的 LL 前缀长度
    case DLT_PFLOG:
        s = gen_load_pflog_llprefixlen(cstate);
        break;
    }

    """
    如果还没有初始化，并且需要为 VLAN 初始化可变长度偏移量，则将它们初始化为零
    """
    if (s == NULL && cstate->is_vlan_vloffset) {
        struct slist *s2;

        if (cstate->off_linkpl.reg == -1)
            cstate->off_linkpl.reg = alloc_reg(cstate);
        if (cstate->off_linktype.reg == -1)
            cstate->off_linktype.reg = alloc_reg(cstate);

        s = new_stmt(cstate, BPF_LD|BPF_W|BPF_IMM);
        s->s.k = 0;
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linkpl.reg;
        sappend(s, s2);
        s2 = new_stmt(cstate, BPF_ST);
        s2->s.k = cstate->off_linktype.reg;
        sappend(s, s2);
    }

    """
    如果有任何偏移量加载代码，则将所有现有语句追加到这些语句中，并将结果列表作为块的语句列表
    """
    if (s != NULL) {
        sappend(s, b->stmts);
        b->stmts = s;
    }
}

// 生成 PPI 数据链路类型检查的代码块
static struct block *
gen_ppi_dlt_check(compiler_state_t *cstate)
{
    // 声明加载数据链路类型的语句列表和代码块
    struct slist *s_load_dlt;
    struct block *b;

    // 如果数据链路类型是 DLT_PPI
    if (cstate->linktype == DLT_PPI)
    {
        /* 创建检查数据链路类型的语句
         */
        s_load_dlt = new_stmt(cstate, BPF_LD|BPF_W|BPF_ABS);
        s_load_dlt->s.k = 4;

        // 创建跳转代码块
        b = new_block(cstate, JMP(BPF_JEQ));

        // 将加载数据链路类型的语句添加到代码块中
        b->stmts = s_load_dlt;
        // 设置跳转条件为 DLT_IEEE802_11
        b->s.k = SWAPLONG(DLT_IEEE802_11);
    }
    else
    {
        // 如果数据链路类型不是 DLT_PPI，则代码块为空
        b = NULL;
    }

    return b;
}

/*
 * 获取绝对偏移量，并生成相应的代码：
 *
 *    如果没有变量部分，返回 NULL；
 *
 *    如果有变量部分，生成加载包含该变量部分的寄存器的代码，返回指向该代码的指针 - 如果尚未为该偏移量的寄存器分配寄存器，则首先分配它。
 *
 * （设置该寄存器的代码将稍后生成，但将放置在代码序列中的较早位置。）
 */
static struct slist *
gen_abs_offset_varpart(compiler_state_t *cstate, bpf_abs_offset *off)
{
    // 声明语句列表
    struct slist *s;

    // 如果偏移量是变量
    if (off->is_variable) {
        // 如果寄存器为 -1，表示尚未为偏移量的变量部分分配寄存器
        if (off->reg == -1) {
            /*
             * 尚未为链路层头部的变量部分的偏移量分配寄存器。
             */
            off->reg = alloc_reg(cstate);
        }

        /*
         * 将包含链路层头部偏移量的变量部分的寄存器加载到 X 寄存器中。
         */
        s = new_stmt(cstate, BPF_LDX|BPF_MEM);
        s->s.k = off->reg;
        return s;
    } else {
        /*
         * 该偏移量不是变量，没有变量部分，因此不需要生成任何代码。
         */
        return NULL;
    }
}

/*
 * 将以太网类型映射到等效的 PPP 类型。
 */
static bpf_u_int32
ethertype_to_ppptype(bpf_u_int32 ll_proto)
{
    switch (ll_proto) {
    # 如果以太网类型为 IP，则设置链路层协议为 PPP_IP
    case ETHERTYPE_IP:
        ll_proto = PPP_IP;
        break;

    # 如果以太网类型为 IPv6，则设置链路层协议为 PPP_IPV6
    case ETHERTYPE_IPV6:
        ll_proto = PPP_IPV6;
        break;

    # 如果以太网类型为 DECNET，则设置链路层协议为 PPP_DECNET
    case ETHERTYPE_DN:
        ll_proto = PPP_DECNET;
        break;

    # 如果以太网类型为 AppleTalk，则设置链路层协议为 PPP_APPLE
    case ETHERTYPE_ATALK:
        ll_proto = PPP_APPLE;
        break;

    # 如果以太网类型为 NS，则设置链路层协议为 PPP_NS
    case ETHERTYPE_NS:
        ll_proto = PPP_NS;
        break;

    # 如果以太网类型为 ISO NS，则设置链路层协议为 PPP_OSI
    case LLCSAP_ISONS:
        ll_proto = PPP_OSI;
        break;

    # 如果以太网类型为 802.1D，则设置链路层协议为 PPP_BRPDU
    case LLCSAP_8021D:
        '''
         * 我假设通过 PPP 传输的“桥接 PDU”是生成树协议的桥接 PDU。
         '''
        ll_proto = PPP_BRPDU;
        break;

    # 如果以太网类型为 IPX，则设置链路层协议为 PPP_IPX
    case LLCSAP_IPX:
        ll_proto = PPP_IPX;
        break;
    }
    # 返回链路层协议
    return (ll_proto);
}
/*
 * 生成任何测试，用于检查封装在另一个协议栈中的链路层数据包，需要检查这些链路层数据包（并且尚未通过检查封装来检查）。
 */
static struct block *
gen_prevlinkhdr_check(compiler_state_t *cstate)
{
    struct block *b0;

    if (cstate->is_geneve)
        return gen_geneve_ll_check(cstate);

    switch (cstate->prevlinktype) {

    case DLT_SUNATM:
        /*
         * 这是 LANE 封装的以太网；检查 LANE 数据包是否以 LE 控制标记开头，即数据而不是控制消息。
         *
         * （我们已经为 LANE 生成了一个测试。）
         */
        b0 = gen_cmp(cstate, OR_PREVLINKHDR, SUNATM_PKT_BEGIN_POS, BPF_H, 0xFF00);
        gen_not(b0);
        return b0;

    default:
        /*
         * 不需要进行此类测试。
         */
        return NULL;
    }
    /*NOTREACHED*/
}

/*
 * 我们在检查 DLT_NULL 的 IPv6 数据包时应该检查的三种不同值。
 */
#define BSD_AFNUM_INET6_BSD    24    /* NetBSD, OpenBSD, BSD/OS, Npcap */
#define BSD_AFNUM_INET6_FREEBSD    28    /* FreeBSD */
#define BSD_AFNUM_INET6_DARWIN    30    /* macOS, iOS, other Darwin-based OSes */

/*
 * 生成代码，通过匹配 802.2 LLC 头中的链路层类型字段或字段来匹配特定的数据包类型。
 *
 * “proto” 是以太网类型值（如果 > ETHERMTU），或 LLC SAP 值（如果 <= ETHERMTU）。
 */
static struct block *
gen_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    struct block *b0, *b1, *b2;
    const char *description;

    /* 我们是否在检查 MPLS 封装的数据包？ */
    if (cstate->label_stack_depth > 0)
        return gen_mpls_linktype(cstate, ll_proto);

    switch (cstate->linktype) {

    case DLT_EN10MB:
    case DLT_NETANALYZER:
    # 如果数据链路类型为DLT_NETANALYZER_TRANSPARENT
    case DLT_NETANALYZER_TRANSPARENT:
        # 如果不是Geneve协议
        if (!cstate->is_geneve)
            # 生成前一个链路头部检查
            b0 = gen_prevlinkhdr_check(cstate);
        else
            # 否则置空
            b0 = NULL;

        # 生成以太网链路类型
        b1 = gen_ether_linktype(cstate, ll_proto);
        # 如果b0不为空，则进行与操作
        if (b0 != NULL)
            gen_and(b0, b1);
        # 返回b1
        return b1;
        # 不会执行到这里

    # 如果数据链路类型为DLT_C_HDLC或DLT_HDLC
    case DLT_C_HDLC:
    case DLT_HDLC:
        switch (ll_proto) {

        # 如果链路层协议为LLCSAP_ISONS
        case LLCSAP_ISONS:
            # 将链路层协议左移8位并与LLCSAP_ISONS进行或操作
            ll_proto = (ll_proto << 8 | LLCSAP_ISONS);
            # 继续执行下面的代码

        # 默认情况
        default:
            # 生成比较指令
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);
            # 不会执行到这里

    # 如果数据链路类型为DLT_IEEE802_11、DLT_PRISM_HEADER、DLT_IEEE802_11_RADIO_AVS、DLT_IEEE802_11_RADIO或DLT_PPI
    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
    case DLT_PPI:
        """
         * 检查是否为数据帧
         """
        # 生成检查802.11数据帧的指令
        b0 = gen_check_802_11_data_frame(cstate);

        """
         * 现在检查指定的链路层类型
         """
        # 生成LLC链路类型的指令
        b1 = gen_llc_linktype(cstate, ll_proto);
        # 进行与操作
        gen_and(b0, b1);
        # 返回b1
        return b1;
        # 不会执行到这里

    # 如果数据链路类型为DLT_FDDI
    case DLT_FDDI:
        """
         * XXX - 检查LLC帧
         """
        # 返回LLC链路类型的指令
        return gen_llc_linktype(cstate, ll_proto);
        # 不会执行到这里

    # 如果数据链路类型为DLT_IEEE802
    case DLT_IEEE802:
        """
         * XXX - 检查LLC PDU，如IEEE 802.5所述
         """
        # 返回LLC链路类型的指令
        return gen_llc_linktype(cstate, ll_proto);
        # 不会执行到这里

    # 如果数据链路类型为DLT_ATM_RFC1483、DLT_ATM_CLIP或DLT_IP_OVER_FC
    case DLT_ATM_RFC1483:
    case DLT_ATM_CLIP:
    case DLT_IP_OVER_FC:
        # 返回LLC链路类型的指令
        return gen_llc_linktype(cstate, ll_proto);
        # 不会执行到这里
    case DLT_SUNATM:
        /*
         * 检查此协议的 LLC 封装版本；
         * 如果我们正在检查 LANE，则 linktype 将不再是 DLT_SUNATM。
         *
         * 检查 LLC 封装，然后检查协议。
         */
        b0 = gen_atmfield_code_internal(cstate, A_PROTOTYPE, PT_LLC, BPF_JEQ, 0);
        b1 = gen_llc_linktype(cstate, ll_proto);
        gen_and(b0, b1);
        return b1;
        /*NOTREACHED*/

    case DLT_LINUX_SLL:
        return gen_linux_sll_linktype(cstate, ll_proto);
        /*NOTREACHED*/

    case DLT_SLIP:
    case DLT_SLIP_BSDOS:
    case DLT_RAW:
        /*
         * 这些类型不提供任何类型字段；数据包始终为 IPv4 或 IPv6。
         *
         * XXX - 对于 IPv4，检查版本号是否为 4，对于 IPv6，检查版本号是否为 6？
         */
        switch (ll_proto) {

        case ETHERTYPE_IP:
            /* 检查版本号是否为 4。 */
            return gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, 0x40, 0xF0);

        case ETHERTYPE_IPV6:
            /* 检查版本号是否为 6。 */
            return gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, 0x60, 0xF0);

        default:
            return gen_false(cstate);    /* 始终为假 */
        }
        /*NOTREACHED*/

    case DLT_IPV4:
        /*
         * 原始 IPv4，因此没有类型字段。
         */
        if (ll_proto == ETHERTYPE_IP)
            return gen_true(cstate);    /* 始终为真 */

        /* 检查是否为 IPv4 以外的内容；始终为假 */
        return gen_false(cstate);
        /*NOTREACHED*/

    case DLT_IPV6:
        /*
         * 原始 IPv6，因此没有类型字段。
         */
        if (ll_proto == ETHERTYPE_IPV6)
            return gen_true(cstate);    /* 始终为真 */

        /* 检查是否为 IPv6 以外的内容；始终为假 */
        return gen_false(cstate);
        /*NOTREACHED*/

    case DLT_PPP:
    case DLT_PPP_PPPD:
    # 如果数据链路类型是PPP串行或PPP以太网
    case DLT_PPP_SERIAL:
    case DLT_PPP_ETHER:
        '''
        在libpcap中，我们使用以太网协议类型；将它们映射到相应的PPP协议类型。
        '''
        return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H,
            ethertype_to_ppptype(ll_proto));
        # 不会执行到这里

    # 如果数据链路类型是PPP BSDOS
    case DLT_PPP_BSDOS:
        '''
        在libpcap中，我们使用以太网协议类型；将它们映射到相应的PPP协议类型。
        '''
        switch (ll_proto) {

        # 如果以太网协议类型是IP
        case ETHERTYPE_IP:
            '''
            同时检查Van Jacobson压缩的IP。
            XXX - 对于PPP的其他形式也这样做吗？
            '''
            b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, PPP_IP);
            b1 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, PPP_VJC);
            gen_or(b0, b1);
            b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, PPP_VJNC);
            gen_or(b1, b0);
            return b0;

        # 如果以太网协议类型不是IP
        default:
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H,
                ethertype_to_ppptype(ll_proto));
        }
        # 不会执行到这里

    # 如果数据链路类型是NULL或LOOP
    case DLT_NULL:
    case DLT_LOOP:
    # 如果数据链路类型是以太网帧
    case DLT_ENC:
        # 根据链路层协议类型进行判断
        switch (ll_proto) {

        # 如果是 IPv4 协议
        case ETHERTYPE_IP:
            # 返回 IPv4 的环回链路类型
            return (gen_loopback_linktype(cstate, AF_INET));

        # 如果是 IPv6 协议
        case ETHERTYPE_IPV6:
            # 如果是从保存文件中读取数据
            if (cstate->bpf_pcap->rfile != NULL) {
                # 从保存文件中检查所有可能的 IPv6 值
                b0 = gen_loopback_linktype(cstate, BSD_AFNUM_INET6_BSD);
                b1 = gen_loopback_linktype(cstate, BSD_AFNUM_INET6_FREEBSD);
                gen_or(b0, b1);
                b0 = gen_loopback_linktype(cstate, BSD_AFNUM_INET6_DARWIN);
                gen_or(b0, b1);
                return (b1);
            } else {
                # 如果是实时捕获，只需要检查此平台上使用的值
#ifdef _WIN32
                /*
                 * 如果是在 Windows 平台下编译，Npcap 不使用 Windows 的 AF_INET6，
                 * 因为在一些 BSD 上会与 AF_IPX 冲突（它们都有值 23）。
                 * 相反，它使用 24。
                 */
                return (gen_loopback_linktype(cstate, 24));
#else /* _WIN32 */
#ifdef AF_INET6
                // 如果支持 AF_INET6，使用它来生成环回链路类型
                return (gen_loopback_linktype(cstate, AF_INET6));
#else /* AF_INET6 */
                /*
                 * 我猜这个平台不支持 IPv6，所以我们拒绝所有数据包。
                 */
                return gen_false(cstate);
#endif /* AF_INET6 */
#endif /* _WIN32 */
            }

        default:
            /*
             * 不是我们支持过滤的类型。
             * XXX - 至少支持在此平台上定义了 AF_ 值的类型？
             */
            return gen_false(cstate);
        }

    case DLT_PFLOG:
        /*
         * af 字段是主机字节顺序，与数据包的其余部分相反。
         */
        if (ll_proto == ETHERTYPE_IP)
            return (gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, af),
                BPF_B, AF_INET));
        else if (ll_proto == ETHERTYPE_IPV6)
            return (gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, af),
                BPF_B, AF_INET6));
        else
            return gen_false(cstate);
        /*NOTREACHED*/

    case DLT_ARCNET:
    # 根据不同的链路类型和协议类型生成不同的过滤器
    case DLT_ARCNET_LINUX:
        # 对于 ARCNET_LINUX 链路类型
        switch (ll_proto) {
            # 根据协议类型进行不同的处理
            default:
                # 默认情况下返回假
                return gen_false(cstate);

            case ETHERTYPE_IPV6:
                # 如果是 IPv6 协议类型，返回与 ARCTYPE_INET6 比较的结果
                return (gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_INET6));

            case ETHERTYPE_IP:
                # 如果是 IP 协议类型
                # 生成与 ARCTYPE_IP 比较的结果
                b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_IP);
                # 生成与 ARCTYPE_IP_OLD 比较的结果
                b1 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_IP_OLD);
                # 生成 b0 和 b1 的逻辑或结果
                gen_or(b0, b1);
                # 返回 b1 的结果
                return (b1);

            case ETHERTYPE_ARP:
                # 如果是 ARP 协议类型
                # 生成与 ARCTYPE_ARP 比较的结果
                b0 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_ARP);
                # 生成与 ARCTYPE_ARP_OLD 比较的结果
                b1 = gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_ARP_OLD);
                # 生成 b0 和 b1 的逻辑或结果
                gen_or(b0, b1);
                # 返回 b1 的结果
                return (b1);

            case ETHERTYPE_REVARP:
                # 如果是 REVARP 协议类型，返回与 ARCTYPE_REVARP 比较的结果
                return (gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_REVARP));

            case ETHERTYPE_ATALK:
                # 如果是 ATALK 协议类型，返回与 ARCTYPE_ATALK 比较的结果
                return (gen_cmp(cstate, OR_LINKTYPE, 0, BPF_B, ARCTYPE_ATALK));
        }
        # 不会执行到这里

    case DLT_LTALK:
        # 对于 LTALK 链路类型
        switch (ll_proto) {
            case ETHERTYPE_ATALK:
                # 如果是 ATALK 协议类型，返回真
                return gen_true(cstate);
            default:
                # 默认情况下返回假
                return gen_false(cstate);
        }
        # 不会执行到这里
    case DLT_FRELAY:
        /*
         * XXX - assumes a 2-byte Frame Relay header with
         * DLCI and flags.  What if the address is longer?
         */
        # 如果地址更长会怎么样？假设一个包含 DLCI 和标志的 2 字节 Frame Relay 头部
        switch (ll_proto) {

        case ETHERTYPE_IP:
            /*
             * Check for the special NLPID for IP.
             */
            # 检查 IP 的特殊 NLPID
            return gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | 0xcc);

        case ETHERTYPE_IPV6:
            /*
             * Check for the special NLPID for IPv6.
             */
            # 检查 IPv6 的特殊 NLPID
            return gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | 0x8e);

        case LLCSAP_ISONS:
            /*
             * Check for several OSI protocols.
             *
             * Frame Relay packets typically have an OSI
             * NLPID at the beginning; we check for each
             * of them.
             *
             * What we check for is the NLPID and a frame
             * control field of UI, i.e. 0x03 followed
             * by the NLPID.
             */
            # 检查几种 OSI 协议
            # Frame Relay 数据包通常在开头有一个 OSI NLPID；我们检查每一个
            # 我们检查的是 NLPID 和一个 UI 的帧控制字段，即 0x03 后跟着 NLPID
            b0 = gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | ISO8473_CLNP);
            b1 = gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | ISO9542_ESIS);
            b2 = gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | ISO10589_ISIS);
            gen_or(b1, b2);
            gen_or(b0, b2);
            return b2;

        default:
            return gen_false(cstate);
        }
        /*NOTREACHED*/

    case DLT_MFR:
        bpf_error(cstate, "Multi-link Frame Relay link-layer type filtering not implemented");

        case DLT_JUNIPER_MFR:
        case DLT_JUNIPER_MLFR:
        case DLT_JUNIPER_MLPPP:
    case DLT_JUNIPER_ATM1:
    case DLT_JUNIPER_ATM2:
    case DLT_JUNIPER_PPPOE:
    case DLT_JUNIPER_PPPOE_ATM:
        // 处理 Juniper PPPoE ATM 类型的数据链路层
        case DLT_JUNIPER_GGSN:
        // 处理 Juniper GGSN 类型的数据链路层
        case DLT_JUNIPER_ES:
        // 处理 Juniper ES 类型的数据链路层
        case DLT_JUNIPER_MONITOR:
        // 处理 Juniper Monitor 类型的数据链路层
        case DLT_JUNIPER_SERVICES:
        // 处理 Juniper Services 类型的数据链路层
        case DLT_JUNIPER_ETHER:
        // 处理 Juniper Ethernet 类型的数据链路层
        case DLT_JUNIPER_PPP:
        // 处理 Juniper PPP 类型的数据链路层
        case DLT_JUNIPER_FRELAY:
        // 处理 Juniper Frame Relay 类型的数据链路层
        case DLT_JUNIPER_CHDLC:
        // 处理 Juniper Cisco HDLC 类型的数据链路层
        case DLT_JUNIPER_VP:
        // 处理 Juniper Virtual Path 类型的数据链路层
        case DLT_JUNIPER_ST:
        // 处理 Juniper ST 类型的数据链路层
        case DLT_JUNIPER_ISM:
        // 处理 Juniper ISM 类型的数据链路层
        case DLT_JUNIPER_VS:
        // 处理 Juniper VS 类型的数据链路层
        case DLT_JUNIPER_SRX_E2E:
        // 处理 Juniper SRX E2E 类型的数据链路层
        case DLT_JUNIPER_FIBRECHANNEL:
        // 处理 Juniper Fibre Channel 类型的数据链路层
    case DLT_JUNIPER_ATM_CEMIC:
        // 处理 Juniper ATM CEMIC 类型的数据链路层
        /* just lets verify the magic number for now -
         * on ATM we may have up to 6 different encapsulations on the wire
         * and need a lot of heuristics to figure out that the payload
         * might be;
         *
         * FIXME encapsulation specific BPF_ filters
         */
        // 比较魔术数字，用于验证 ATM 类型的数据链路层
        return gen_mcmp(cstate, OR_LINKHDR, 0, BPF_W, 0x4d474300, 0xffffff00); /* compare the magic number */

    case DLT_BACNET_MS_TP:
        // 处理 BACnet MS/TP 类型的数据链路层
        return gen_mcmp(cstate, OR_LINKHDR, 0, BPF_W, 0x55FF0000, 0xffff0000);

    case DLT_IPNET:
        // 处理 IPnet 类型的数据链路层
        return gen_ipnet_linktype(cstate, ll_proto);

    case DLT_LINUX_IRDA:
        // 报错，未实现 IrDA 类型的数据链路层过滤
        bpf_error(cstate, "IrDA link-layer type filtering not implemented");

    case DLT_DOCSIS:
        // 报错，未实现 DOCSIS 类型的数据链路层过滤
        bpf_error(cstate, "DOCSIS link-layer type filtering not implemented");

    case DLT_MTP2:
    case DLT_MTP2_WITH_PHDR:
        // 报错，未实现 MTP2 类型的数据链路层过滤
        bpf_error(cstate, "MTP2 link-layer type filtering not implemented");

    case DLT_ERF:
        // 报错，未实现 ERF 类型的数据链路层过滤
        bpf_error(cstate, "ERF link-layer type filtering not implemented");

    case DLT_PFSYNC:
        // 报错，未实现 PFSYNC 类型的数据链路层过滤
        bpf_error(cstate, "PFSYNC link-layer type filtering not implemented");

    case DLT_LINUX_LAPD:
        // 报错，未实现 LAPD 类型的数据链路层过滤
        bpf_error(cstate, "LAPD link-layer type filtering not implemented");

    case DLT_USB_FREEBSD:
    case DLT_USB_LINUX:
    case DLT_USB_LINUX_MMAPPED:
    case DLT_USBPCAP:
        // 报错，未实现 USB 类型的数据链路层过滤
        bpf_error(cstate, "USB link-layer type filtering not implemented");

    case DLT_BLUETOOTH_HCI_H4:
    # 如果是蓝牙链路层类型，则报错，因为蓝牙链路层类型过滤未实现
    case DLT_BLUETOOTH_HCI_H4_WITH_PHDR:
        bpf_error(cstate, "Bluetooth link-layer type filtering not implemented");

    # 如果是CAN 2.0B或CAN SocketCAN链路层类型，则报错，因为CAN链路层类型过滤未实现
    case DLT_CAN20B:
    case DLT_CAN_SOCKETCAN:
        bpf_error(cstate, "CAN link-layer type filtering not implemented");

    # 如果是IEEE 802.15.4链路层类型，则报错，因为IEEE 802.15.4链路层类型过滤未实现
    case DLT_IEEE802_15_4:
    case DLT_IEEE802_15_4_LINUX:
    case DLT_IEEE802_15_4_NONASK_PHY:
    case DLT_IEEE802_15_4_NOFCS:
    case DLT_IEEE802_15_4_TAP:
        bpf_error(cstate, "IEEE 802.15.4 link-layer type filtering not implemented");

    # 如果是IEEE 802.16链路层类型，则报错，因为IEEE 802.16链路层类型过滤未实现
    case DLT_IEEE802_16_MAC_CPS_RADIO:
        bpf_error(cstate, "IEEE 802.16 link-layer type filtering not implemented");

    # 如果是SITA链路层类型，则报错，因为SITA链路层类型过滤未实现
    case DLT_SITA:
        bpf_error(cstate, "SITA link-layer type filtering not implemented");

    # 如果是RAIF1链路层类型，则报错，因为RAIF1链路层类型过滤未实现
    case DLT_RAIF1:
        bpf_error(cstate, "RAIF1 link-layer type filtering not implemented");

    # 如果是IPMB Kontron或IPMB Linux链路层类型，则报错，因为IPMB链路层类型过滤未实现
    case DLT_IPMB_KONTRON:
    case DLT_IPMB_LINUX:
        bpf_error(cstate, "IPMB link-layer type filtering not implemented");

    # 如果是AX.25 KISS链路层类型，则报错，因为AX.25链路层类型过滤未实现
    case DLT_AX25_KISS:
        bpf_error(cstate, "AX.25 link-layer type filtering not implemented");

    # 如果是NFLOG链路层类型，则报错，因为NFLOG链路层类型过滤未实现
    case DLT_NFLOG:
        /* 使用固定大小的NFLOG头部，只能告诉数据包的地址族，其他有意义的数据要么缺失，要么在TLV后面 */
        bpf_error(cstate, "NFLOG link-layer type filtering not implemented");
    default:
        /*
         * Does this link-layer header type have a field
         * indicating the type of the next protocol?  If
         * so, off_linktype.constant_part will be the offset of that
         * field in the packet; if not, it will be OFFSET_NOT_SET.
         */
        // 如果这个链路层头部类型有一个字段指示下一个协议的类型吗？如果有，off_linktype.constant_part 将是数据包中该字段的偏移量；如果没有，它将是 OFFSET_NOT_SET。
        if (cstate->off_linktype.constant_part != OFFSET_NOT_SET) {
            /*
             * Yes; assume it's an Ethernet type.  (If
             * it's not, it needs to be handled specially
             * above.)
             */
            // 是的；假设它是以太网类型。（如果不是，它需要在上面特殊处理。）
            return gen_cmp(cstate, OR_LINKTYPE, 0, BPF_H, ll_proto);
            /*NOTREACHED */
        } else {
            /*
             * No; report an error.
             */
            // 不是；报告一个错误。
            description = pcap_datalink_val_to_description_or_dlt(cstate->linktype);
            bpf_error(cstate, "%s link-layer type filtering not implemented",
                description);
            /*NOTREACHED */
        }
    }
}

/*
 * 检查具有给定组织代码和协议类型的 LLC SNAP 数据包；
 * 我们检查整个 802.2 LLC 和 snap 头的内容，检查 LLC 头中的 DSAP 和 SSAP 是否为 SNAP，控制字段是否为 0x03，
 * 以及在 SNAP 头中指定的组织代码和协议类型。
 */
static struct block *
gen_snap(compiler_state_t *cstate, bpf_u_int32 orgcode, bpf_u_int32 ptype)
{
    u_char snapblock[8];

    snapblock[0] = LLCSAP_SNAP;        /* DSAP = SNAP */
    snapblock[1] = LLCSAP_SNAP;        /* SSAP = SNAP */
    snapblock[2] = 0x03;            /* control = UI */
    snapblock[3] = (u_char)(orgcode >> 16);    /* organization code 的高 8 位 */
    snapblock[4] = (u_char)(orgcode >> 8);    /* organization code 的中间 8 位 */
    snapblock[5] = (u_char)(orgcode >> 0);    /* organization code 的低 8 位 */
    snapblock[6] = (u_char)(ptype >> 8);    /* protocol type 的高 8 位 */
    snapblock[7] = (u_char)(ptype >> 0);    /* protocol type 的低 8 位 */
    return gen_bcmp(cstate, OR_LLC, 0, 8, snapblock);
}

/*
 * 生成匹配具有 LLC 头的帧的代码。
 */
static struct block *
gen_llc_internal(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    switch (cstate->linktype) {

    case DLT_EN10MB:
        /*
         * 我们检查以太网类型字段小于 1500，这意味着它是 802.3 长度字段。
         */
        b0 = gen_cmp_gt(cstate, OR_LINKTYPE, 0, BPF_H, ETHERMTU);
        gen_not(b0);

        /*
         * 现在检查所谓的 DSAP 和 SSAP 是否不是 0xFF，以排除 NetWare-over-802.3。
         */
        b1 = gen_cmp(cstate, OR_LLC, 0, BPF_H, 0xFFFF);
        gen_not(b1);
        gen_and(b0, b1);
        return b1;

    case DLT_SUNATM:
        /*
         * 我们检查 LLC 交通。
         */
        b0 = gen_atmtype_llc(cstate);
        return b0;
    case DLT_IEEE802:    /* Token Ring */
        /*
         * 对于 Token Ring，检查是否为 LLC 帧。
         */
        return gen_true(cstate);

    case DLT_FDDI:
        /*
         * 对于 FDDI，检查是否为 LLC 帧。
         */
        return gen_true(cstate);

    case DLT_ATM_RFC1483:
        /*
         * 对于 ATM_RFC1483，LLC 封装定义了一个 802.2 LLC 头部。
         *
         * 对于 VC 封装，没有 LLC 头部，但是没有办法检查；VC 上使用的协议是通过带外协商的。
         */
        return gen_true(cstate);

    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_PPI:
        /*
         * 检查是否为数据帧。
         */
        b0 = gen_check_802_11_data_frame(cstate);
        return b0;

    default:
        bpf_error(cstate, "'llc' not supported for %s",
              pcap_datalink_val_to_description_or_dlt(cstate->linktype));
        /*NOTREACHED*/
    }
}

struct block *
gen_llc(compiler_state_t *cstate)
{
    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    return gen_llc_internal(cstate);
}

struct block *
gen_llc_i(compiler_state_t *cstate)
{
    struct block *b0, *b1;
    struct slist *s;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * Check whether this is an LLC frame.
     */
    // 检查是否为 LLC 帧
    b0 = gen_llc_internal(cstate);

    /*
     * Load the control byte and test the low-order bit; it must
     * be clear for I frames.
     */
    // 加载控制字节并测试低位比特；对于 I 帧必须清除
    s = gen_load_a(cstate, OR_LLC, 2, BPF_B);
    b1 = new_block(cstate, JMP(BPF_JSET));
    b1->s.k = 0x01;
    b1->stmts = s;
    gen_not(b1);
    gen_and(b0, b1);
    return b1;
}

struct block *
gen_llc_s(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * Check whether this is an LLC frame.
     */
    // 检查是否为 LLC 帧
    b0 = gen_llc_internal(cstate);

    /*
     * Now compare the low-order 2 bit of the control byte against
     * the appropriate value for S frames.
     */
    // 现在比较控制字节的低 2 位与 S 帧的适当值
    b1 = gen_mcmp(cstate, OR_LLC, 2, BPF_B, LLC_S_FMT, 0x03);
    gen_and(b0, b1);
    return b1;
}

struct block *
gen_llc_u(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * Check whether this is an LLC frame.
     */
    // 检查是否为 LLC 帧
    b0 = gen_llc_internal(cstate);

    /*
     * Now compare the low-order 2 bit of the control byte against
     * the appropriate value for U frames.
     */
    // 现在比较控制字节的低 2 位与 U 帧的适当值
    # 生成一个用于匹配 LLC_U_FMT 和 0x03 的 BPF 指令
    b1 = gen_mcmp(cstate, OR_LLC, 2, BPF_B, LLC_U_FMT, 0x03);
    # 生成一个与操作的 BPF 指令，将结果存储在 b0 中
    gen_and(b0, b1);
    # 返回 b1
    return b1;
}

# 生成 LLC S 子类型的代码块
struct block *
gen_llc_s_subtype(compiler_state_t *cstate, bpf_u_int32 subtype)
{
    struct block *b0, *b1;

    """
    捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
    """
    if (setjmp(cstate->top_ctx))
        return (NULL);

    """
    检查这是否是一个 LLC 帧。
    """
    b0 = gen_llc_internal(cstate);

    """
    现在检查具有适当类型的 S 帧。
    """
    b1 = gen_mcmp(cstate, OR_LLC, 2, BPF_B, subtype, LLC_S_CMD_MASK);
    gen_and(b0, b1);
    return b1;
}

# 生成 LLC U 子类型的代码块
struct block *
gen_llc_u_subtype(compiler_state_t *cstate, bpf_u_int32 subtype)
{
    struct block *b0, *b1;

    """
    捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
    """
    if (setjmp(cstate->top_ctx))
        return (NULL);

    """
    检查这是否是一个 LLC 帧。
    """
    b0 = gen_llc_internal(cstate);

    """
    现在检查具有适当类型的 U 帧。
    """
    b1 = gen_mcmp(cstate, OR_LLC, 2, BPF_B, subtype, LLC_U_CMD_MASK);
    gen_and(b0, b1);
    return b1;
}

"""
为使用 802.2 LLC 头的链路层类型生成匹配特定数据包类型的代码。
这不适用于以太网；以太网使用“gen_ether_linktype()”处理这个问题 - 它处理了 D/I/X 以太网与 802.3+802.2 的问题。
“proto”是以太网类型值，如果大于 ETHERMTU，或者是 LLC SAP 值，如果小于等于 ETHERMTU。我们使用它来确定是匹配 DSAP 还是同时匹配 DSAP 和 LSAP，或者检查 SNAP 头中的 OUI 和协议 ID。
"""
static struct block *
gen_llc_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    """
    XXX - 处理令牌环可变长度头。
    """
    switch (ll_proto) {

    case LLCSAP_IP:
    case LLCSAP_ISONS:
    # 如果是LLCSAP_NETBEUI类型的数据包
    case LLCSAP_NETBEUI:
        '''
         * XXX - should we check both the DSAP and the
         * SSAP, like this, or should we check just the
         * DSAP, as we do for other SAP values?
         '''
        # 生成一个比较操作，比较LLC头部中的DSAP和SSAP字段
        return gen_cmp(cstate, OR_LLC, 0, BPF_H, (bpf_u_int32)
                 ((ll_proto << 8) | ll_proto));

    # 如果是LLCSAP_IPX类型的数据包
    case LLCSAP_IPX:
        '''
         * XXX - are there ever SNAP frames for IPX on
         * non-Ethernet 802.x networks?
         '''
        # 生成一个比较操作，比较LLC头部中的SAP字段是否为LLCSAP_IPX

        return gen_cmp(cstate, OR_LLC, 0, BPF_B, LLCSAP_IPX);

    # 如果是ETHERTYPE_ATALK类型的数据包
    case ETHERTYPE_ATALK:
        '''
         * 802.2-encapsulated ETHERTYPE_ATALK packets are
         * SNAP packets with an organization code of
         * 0x080007 (Apple, for Appletalk) and a protocol
         * type of ETHERTYPE_ATALK (Appletalk).
         *
         * XXX - check for an organization code of
         * encapsulated Ethernet as well?
         '''
        # 生成一个SNAP操作，检查组织代码和协议类型是否为Appletalk
        return gen_snap(cstate, 0x080007, ETHERTYPE_ATALK);
    default:
        /*
         * XXX - we don't have to check for IPX 802.3
         * here, but should we check for the IPX Ethertype?
         */
        // 如果逻辑链路层协议小于或等于以太网最大传输单元（ETHERMTU），则为逻辑链路控制（LLC）SAP值，检查DSAP
        if (ll_proto <= ETHERMTU) {
            /*
             * This is an LLC SAP value, so check
             * the DSAP.
             */
            // 生成比较操作，比较LLC SAP值
            return gen_cmp(cstate, OR_LLC, 0, BPF_B, ll_proto);
        } else {
            /*
             * This is an Ethernet type; we assume that it's
             * unlikely that it'll appear in the right place
             * at random, and therefore check only the
             * location that would hold the Ethernet type
             * in a SNAP frame with an organization code of
             * 0x000000 (encapsulated Ethernet).
             *
             * XXX - if we were to check for the SNAP DSAP and
             * LSAP, as per XXX, and were also to check for an
             * organization code of 0x000000 (encapsulated
             * Ethernet), we'd do
             *
             *    return gen_snap(cstate, 0x000000, ll_proto);
             *
             * here; for now, we don't, as per the above.
             * I don't know whether it's worth the extra CPU
             * time to do the right check or not.
             */
            // 这是一个以太网类型；我们假设它不太可能随机出现在正确的位置，因此只检查在具有组织代码0x000000（封装以太网）的SNAP帧中保存以太网类型的位置
            // 生成比较操作，比较以太网类型
            return gen_cmp(cstate, OR_LLC, 6, BPF_H, ll_proto);
        }
    }
}
# 定义一个静态函数，用于生成主机操作的数据块
static struct block *
gen_hostop(compiler_state_t *cstate, bpf_u_int32 addr, bpf_u_int32 mask,
    int dir, bpf_u_int32 ll_proto, u_int src_off, u_int dst_off)
{
    struct block *b0, *b1;
    u_int offset;

    # 根据方向选择偏移量
    switch (dir) {

    case Q_SRC:
        offset = src_off;
        break;

    case Q_DST:
        offset = dst_off;
        break;

    case Q_AND:
        # 递归调用 gen_hostop 函数，生成并操作的数据块
        b0 = gen_hostop(cstate, addr, mask, Q_SRC, ll_proto, src_off, dst_off);
        b1 = gen_hostop(cstate, addr, mask, Q_DST, ll_proto, src_off, dst_off);
        gen_and(b0, b1);
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        # 递归调用 gen_hostop 函数，生成或操作的数据块
        b0 = gen_hostop(cstate, addr, mask, Q_SRC, ll_proto, src_off, dst_off);
        b1 = gen_hostop(cstate, addr, mask, Q_DST, ll_proto, src_off, dst_off);
        gen_or(b0, b1);
        return b1;

    case Q_ADDR1:
        # 抛出错误，'addr1' 和 'address1' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'addr1' and 'address1' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR2:
        # 抛出错误，'addr2' 和 'address2' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'addr2' and 'address2' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR3:
        # 抛出错误，'addr3' 和 'address3' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'addr3' and 'address3' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR4:
        # 抛出错误，'addr4' 和 'address4' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'addr4' and 'address4' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_RA:
        # 抛出错误，'ra' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'ra' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_TA:
        # 抛出错误，'ta' 不是 802.11 MAC 地址的有效限定符
        bpf_error(cstate, "'ta' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    default:
        # 终止程序
        abort();
        /*NOTREACHED*/
    }
    # 生成链路类型的数据块
    b0 = gen_linktype(cstate, ll_proto);
    # 生成多条件匹配的数据块
    b1 = gen_mcmp(cstate, OR_LINKPL, offset, BPF_W, addr, mask);
    gen_and(b0, b1);
    return b1;
}

#ifdef INET6
static struct block *
// 生成 IPv6 主机地址匹配操作
gen_hostop6(compiler_state_t *cstate, struct in6_addr *addr,
    struct in6_addr *mask, int dir, bpf_u_int32 ll_proto, u_int src_off,
    u_int dst_off)
{
    struct block *b0, *b1;
    u_int offset;
    uint32_t *a, *m;

    switch (dir) {

    case Q_SRC:
        // 如果是源地址，使用源偏移量
        offset = src_off;
        break;

    case Q_DST:
        // 如果是目的地址，使用目的偏移量
        offset = dst_off;
        break;

    case Q_AND:
        // 如果是 AND 操作，生成源地址和目的地址的匹配操作，并返回结果
        b0 = gen_hostop6(cstate, addr, mask, Q_SRC, ll_proto, src_off, dst_off);
        b1 = gen_hostop6(cstate, addr, mask, Q_DST, ll_proto, src_off, dst_off);
        gen_and(b0, b1);
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        // 如果是默认或者 OR 操作，生成源地址和目的地址的匹配操作，并返回结果
        b0 = gen_hostop6(cstate, addr, mask, Q_SRC, ll_proto, src_off, dst_off);
        b1 = gen_hostop6(cstate, addr, mask, Q_DST, ll_proto, src_off, dst_off);
        gen_or(b0, b1);
        return b1;

    case Q_ADDR1:
        // 如果是地址1，报错并终止程序
        bpf_error(cstate, "'addr1' and 'address1' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR2:
        // 如果是地址2，报错并终止程序
        bpf_error(cstate, "'addr2' and 'address2' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR3:
        // 如果是地址3，报错并终止程序
        bpf_error(cstate, "'addr3' and 'address3' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_ADDR4:
        // 如果是地址4，报错并终止程序
        bpf_error(cstate, "'addr4' and 'address4' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_RA:
        // 如果是 RA，报错并终止程序
        bpf_error(cstate, "'ra' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    case Q_TA:
        // 如果是 TA，报错并终止程序
        bpf_error(cstate, "'ta' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    default:
        // 默认情况下，终止程序
        abort();
        /*NOTREACHED*/
    }
    /* this order is important */
    // 将 IPv6 地址和掩码转换为 32 位整数
    a = (uint32_t *)addr;
    m = (uint32_t *)mask;
}
    # 生成比较指令，比较偏移量为 offset+12 处的数据和 ntohl(a[3])、ntohl(m[3])，结果保存在 b1 中
    b1 = gen_mcmp(cstate, OR_LINKPL, offset + 12, BPF_W, ntohl(a[3]), ntohl(m[3]));
    # 生成比较指令，比较偏移量为 offset+8 处的数据和 ntohl(a[2])、ntohl(m[2])，结果保存在 b0 中
    b0 = gen_mcmp(cstate, OR_LINKPL, offset + 8, BPF_W, ntohl(a[2]), ntohl(m[2]));
    # 生成与操作指令，将 b0 和 b1 的结果进行与操作
    gen_and(b0, b1);
    # 生成比较指令，比较偏移量为 offset+4 处的数据和 ntohl(a[1])、ntohl(m[1])，结果保存在 b0 中
    b0 = gen_mcmp(cstate, OR_LINKPL, offset + 4, BPF_W, ntohl(a[1]), ntohl(m[1]));
    # 生成与操作指令，将 b0 和 b1 的结果进行与操作
    gen_and(b0, b1);
    # 生成比较指令，比较偏移量为 offset+0 处的数据和 ntohl(a[0])、ntohl(m[0])，结果保存在 b0 中
    b0 = gen_mcmp(cstate, OR_LINKPL, offset + 0, BPF_W, ntohl(a[0]), ntohl(m[0]));
    # 生成与操作指令，将 b0 和 b1 的结果进行与操作
    gen_and(b0, b1);
    # 生成链路类型比较指令，比较链路类型是否为 ll_proto，结果保存在 b0 中
    b0 = gen_linktype(cstate, ll_proto);
    # 生成与操作指令，将 b0 和 b1 的结果进行与操作
    gen_and(b0, b1);
    # 返回最终结果 b1
    return b1;
    }
#endif

# 为以太网地址生成过滤器
static struct block *
gen_ehostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    register struct block *b0, *b1;

    # 根据方向选择生成过滤器
    switch (dir) {
    case Q_SRC:
        # 生成比较源地址的过滤器
        return gen_bcmp(cstate, OR_LINKHDR, 6, 6, eaddr);

    case Q_DST:
        # 生成比较目的地址的过滤器
        return gen_bcmp(cstate, OR_LINKHDR, 0, 6, eaddr);

    case Q_AND:
        # 生成与逻辑的过滤器
        b0 = gen_ehostop(cstate, eaddr, Q_SRC);
        b1 = gen_ehostop(cstate, eaddr, Q_DST);
        gen_and(b0, b1);
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        # 生成或逻辑的过滤器
        b0 = gen_ehostop(cstate, eaddr, Q_SRC);
        b1 = gen_ehostop(cstate, eaddr, Q_DST);
        gen_or(b0, b1);
        return b1;

    case Q_ADDR1:
        # 802.11帧中只支持'addr1'和'address1'
        bpf_error(cstate, "'addr1' and 'address1' are only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/

    case Q_ADDR2:
        # 802.11帧中只支持'addr2'和'address2'
        bpf_error(cstate, "'addr2' and 'address2' are only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/

    case Q_ADDR3:
        # 802.11帧中只支持'addr3'和'address3'
        bpf_error(cstate, "'addr3' and 'address3' are only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/

    case Q_ADDR4:
        # 802.11帧中只支持'addr4'和'address4'
        bpf_error(cstate, "'addr4' and 'address4' are only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/

    case Q_RA:
        # 802.11帧中只支持'ra'
        bpf_error(cstate, "'ra' is only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/

    case Q_TA:
        # 802.11帧中只支持'ta'
        bpf_error(cstate, "'ta' is only supported on 802.11 with 802.11 headers");
        /*NOTREACHED*/
    }
    # 异常处理
    abort();
    /*NOTREACHED*/
}

/*
 * Like gen_ehostop, but for DLT_FDDI
 */
static struct block *
gen_fhostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    struct block *b0, *b1;

    switch (dir) {
    case Q_SRC:
        # 生成比较源地址的过滤器
        return gen_bcmp(cstate, OR_LINKHDR, 6 + 1 + cstate->pcap_fddipad, 6, eaddr);

    case Q_DST:
        # 生成比较目的地址的过滤器
        return gen_bcmp(cstate, OR_LINKHDR, 0 + 1 + cstate->pcap_fddipad, 6, eaddr);
    # 如果条件为 Q_AND，则生成过滤器操作码，将源地址和目的地址进行与操作
    case Q_AND:
        b0 = gen_fhostop(cstate, eaddr, Q_SRC);
        b1 = gen_fhostop(cstate, eaddr, Q_DST);
        gen_and(b0, b1);
        return b1;

    # 如果条件为 Q_DEFAULT 或 Q_OR，则生成过滤器操作码，将源地址和目的地址进行或操作
    case Q_DEFAULT:
    case Q_OR:
        b0 = gen_fhostop(cstate, eaddr, Q_SRC);
        b1 = gen_fhostop(cstate, eaddr, Q_DST);
        gen_or(b0, b1);
        return b1;

    # 如果条件为 Q_ADDR1 到 Q_ADDR4、Q_RA 或 Q_TA，则在802.11网络上不支持这些条件，抛出错误
    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_RA:
        bpf_error(cstate, "'ra' is only supported on 802.11");
        /*NOTREACHED*/

    case Q_TA:
        bpf_error(cstate, "'ta' is only supported on 802.11");
        /*NOTREACHED*/
    }
    # 终止程序执行
    abort();
    /*NOTREACHED*/
}

/*
 * 与 gen_ehostop 类似，但适用于 DLT_IEEE802（Token Ring）
 */
static struct block *
gen_thostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    register struct block *b0, *b1;

    switch (dir) {
    case Q_SRC:
        return gen_bcmp(cstate, OR_LINKHDR, 8, 6, eaddr);  // 生成比较块，比较源地址

    case Q_DST:
        return gen_bcmp(cstate, OR_LINKHDR, 2, 6, eaddr);  // 生成比较块，比较目的地址

    case Q_AND:
        b0 = gen_thostop(cstate, eaddr, Q_SRC);  // 递归调用 gen_thostop，生成源地址比较块
        b1 = gen_thostop(cstate, eaddr, Q_DST);  // 递归调用 gen_thostop，生成目的地址比较块
        gen_and(b0, b1);  // 生成与操作块
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        b0 = gen_thostop(cstate, eaddr, Q_SRC);  // 递归调用 gen_thostop，生成源地址比较块
        b1 = gen_thostop(cstate, eaddr, Q_DST);  // 递归调用 gen_thostop，生成目的地址比较块
        gen_or(b0, b1);  // 生成或操作块
        return b1;

    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_RA:
        bpf_error(cstate, "'ra' is only supported on 802.11");
        /*NOTREACHED*/

    case Q_TA:
        bpf_error(cstate, "'ta' is only supported on 802.11");
        /*NOTREACHED*/
    }
    abort();  // 终止程序
    /*NOTREACHED*/
}

/*
 * 与 gen_ehostop 类似，但适用于 DLT_IEEE802_11（802.11 无线局域网）和各种 802.11 + 无线电头部
 */
static struct block *
gen_wlanhostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    register struct block *b0, *b1, *b2;
    register struct slist *s;

#ifdef ENABLE_WLAN_FILTERING_PATCH
    /*
     * TODO GV 20070613
     * 我们需要禁用优化器，因为优化器存在错误，并且会擦除下面代码生成的一些 LD 指令，以验证帧控制位
     */
    cstate->no_optimize = 1;
#endif /* ENABLE_WLAN_FILTERING_PATCH */

# 根据不同的方向进行处理
switch (dir) {
    # 如果是与操作
    case Q_AND:
        # 生成源地址的过滤器
        b0 = gen_wlanhostop(cstate, eaddr, Q_SRC);
        # 生成目的地址的过滤器
        b1 = gen_wlanhostop(cstate, eaddr, Q_DST);
        # 生成两个过滤器的与操作
        gen_and(b0, b1);
        # 返回目的地址的过滤器
        return b1;

    # 如果是默认或者或操作
    case Q_DEFAULT:
    case Q_OR:
        # 生成源地址的过滤器
        b0 = gen_wlanhostop(cstate, eaddr, Q_SRC);
        # 生成目的地址的过滤器
        b1 = gen_wlanhostop(cstate, eaddr, Q_DST);
        # 生成两个过滤器的或操作
        gen_or(b0, b1);
        # 返回目的地址的过滤器
        return b1;

    # 如果是地址1
    case Q_ADDR1:
        # 返回比较结果
        return (gen_bcmp(cstate, OR_LINKHDR, 4, 6, eaddr));

    # 如果是地址2
    case Q_ADDR2:
        # 生成比较操作
        b0 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_TYPE_CTL,
            IEEE80211_FC0_TYPE_MASK);
        # 生成非操作
        gen_not(b0);
        # 生成比较操作
        b1 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_SUBTYPE_CTS,
            IEEE80211_FC0_SUBTYPE_MASK);
        # 生成非操作
        gen_not(b1);
        # 生成比较操作
        b2 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_SUBTYPE_ACK,
            IEEE80211_FC0_SUBTYPE_MASK);
        # 生成非操作
        gen_not(b2);
        # 生成与操作
        gen_and(b1, b2);
        # 生成或操作
        gen_or(b0, b2);
        # 生成比较操作
        b1 = gen_bcmp(cstate, OR_LINKHDR, 10, 6, eaddr);
        # 生成与操作
        gen_and(b2, b1);
        # 返回比较结果
        return b1;

    # 如果是地址3
    case Q_ADDR3:
        # 生成比较操作
        b0 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_TYPE_CTL,
            IEEE80211_FC0_TYPE_MASK);
        # 生成非操作
        gen_not(b0);
        # 生成比较操作
        b1 = gen_bcmp(cstate, OR_LINKHDR, 16, 6, eaddr);
        # 生成与操作
        gen_and(b0, b1);
        # 返回比较结果
        return b1;

    # 如果是地址4
    case Q_ADDR4:
        # 生成比较操作
        b0 = gen_mcmp(cstate, OR_LINKHDR, 1, BPF_B,
            IEEE80211_FC1_DIR_DSTODS, IEEE80211_FC1_DIR_MASK);
        # 生成比较操作
        b1 = gen_bcmp(cstate, OR_LINKHDR, 24, 6, eaddr);
        # 生成与操作
        gen_and(b0, b1);
        # 返回比较结果
        return b1;
    # 如果是 Q_RA 情况
    case Q_RA:
        '''
        * 管理帧中不存在；在其他帧中 addr1 存在。
        '''

        '''
        * 如果类型值的高位为 0，则这是一个管理帧。
        * 即，检查 "(link[0] & 0x08)"。
        '''
        s = gen_load_a(cstate, OR_LINKHDR, 0, BPF_B);
        b1 = new_block(cstate, JMP(BPF_JSET));
        b1->s.k = 0x08;
        b1->stmts = s;

        '''
        * 检查 addr1。
        '''
        b0 = gen_bcmp(cstate, OR_LINKHDR, 4, 6, eaddr);

        '''
        * 与 addr1 的检查进行 AND 操作。
        '''
        gen_and(b1, b0);
        return (b0);

    # 如果是 Q_TA 情况
    case Q_TA:
        '''
        * 管理帧中不存在；在其他帧中存在 addr2（如果存在）。
        '''

        '''
        * 在 CTS 或 ACK 控制帧中不存在。
        '''
        b0 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_TYPE_CTL,
            IEEE80211_FC0_TYPE_MASK);
        gen_not(b0);
        b1 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_SUBTYPE_CTS,
            IEEE80211_FC0_SUBTYPE_MASK);
        gen_not(b1);
        b2 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, IEEE80211_FC0_SUBTYPE_ACK,
            IEEE80211_FC0_SUBTYPE_MASK);
        gen_not(b2);
        gen_and(b1, b2);
        gen_or(b0, b2);

        '''
        * 如果类型值的高位为 0，则这是一个管理帧。
        * 即，检查 "(link[0] & 0x08)"。
        '''
        s = gen_load_a(cstate, OR_LINKHDR, 0, BPF_B);
        b1 = new_block(cstate, JMP(BPF_JSET));
        b1->s.k = 0x08;
        b1->stmts = s;

        '''
        * 与非 CTS 和 ACK 帧的检查进行 AND 操作。
        '''
        gen_and(b1, b2);

        '''
        * 检查 addr2。
        '''
        b1 = gen_bcmp(cstate, OR_LINKHDR, 10, 6, eaddr);
        gen_and(b2, b1);
        return b1;
    }
    abort();
    /*NOTREACHED*/
/*
 * Like gen_ehostop, but for RFC 2625 IP-over-Fibre-Channel.
 * (We assume that the addresses are IEEE 48-bit MAC addresses,
 * as the RFC states.)
 */
static struct block *
gen_ipfchostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    register struct block *b0, *b1;

    switch (dir) {
    case Q_SRC:
        // 生成一个比较块，用于匹配源地址
        return gen_bcmp(cstate, OR_LINKHDR, 10, 6, eaddr);

    case Q_DST:
        // 生成一个比较块，用于匹配目的地址
        return gen_bcmp(cstate, OR_LINKHDR, 2, 6, eaddr);

    case Q_AND:
        // 生成两个比较块，分别用于匹配源地址和目的地址，然后进行逻辑与操作
        b0 = gen_ipfchostop(cstate, eaddr, Q_SRC);
        b1 = gen_ipfchostop(cstate, eaddr, Q_DST);
        gen_and(b0, b1);
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        // 生成两个比较块，分别用于匹配源地址和目的地址，然后进行逻辑或操作
        b0 = gen_ipfchostop(cstate, eaddr, Q_SRC);
        b1 = gen_ipfchostop(cstate, eaddr, Q_DST);
        gen_or(b0, b1);
        return b1;

    case Q_ADDR1:
        // 报错，'addr1' 和 'address1' 仅在 802.11 上支持
        bpf_error(cstate, "'addr1' and 'address1' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR2:
        // 报错，'addr2' 和 'address2' 仅在 802.11 上支持
        bpf_error(cstate, "'addr2' and 'address2' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR3:
        // 报错，'addr3' 和 'address3' 仅在 802.11 上支持
        bpf_error(cstate, "'addr3' and 'address3' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_ADDR4:
        // 报错，'addr4' 和 'address4' 仅在 802.11 上支持
        bpf_error(cstate, "'addr4' and 'address4' are only supported on 802.11");
        /*NOTREACHED*/

    case Q_RA:
        // 报错，'ra' 仅在 802.11 上支持
        bpf_error(cstate, "'ra' is only supported on 802.11");
        /*NOTREACHED*/

    case Q_TA:
        // 报错，'ta' 仅在 802.11 上支持
        bpf_error(cstate, "'ta' is only supported on 802.11");
        /*NOTREACHED*/
    }
    // 终止程序执行
    abort();
    /*NOTREACHED*/
}
/*
 * 这段代码非常棘手，因为在 DECNET 头部前面可能有填充字节，然后有两种可能的数据包格式，
 * 它们都携带源地址和目的地址，另外还有 5 种数据包类型采用一种只携带源节点的格式，
 * 还有 2 种类型采用不同的格式，也只携带源节点。
 *
 * 真烦人。
 *
 * 我们不打算把这些都做对，我们只寻找携带 0 或 1 个字节填充的数据包。
 * 如果你想查看其他数据包，那将需要进行更多的修改。
 *
 * 要添加对 DECNET "区域"（网络号）的过滤支持，需要在这个函数中添加一个 "mask" 参数。
 * 这将使过滤器更加低效，尽管如果掩码为 0xFFFF，可以聪明地不生成掩码指令。
 */
static struct block *
gen_dnhostop(compiler_state_t *cstate, bpf_u_int32 addr, int dir)
{
    struct block *b0, *b1, *b2, *tmp;
    u_int offset_lh;    /* 如果接收到长头部，则偏移量 */
    u_int offset_sh;    /* 如果接收到短头部，则偏移量 */

    switch (dir) {

    case Q_DST:
        offset_sh = 1;    /* 跟随标志位 */
        offset_lh = 7;    /* 标志位，目的区域，目的子区域，HIORD 之后 */
        break;

    case Q_SRC:
        offset_sh = 3;    /* 跟随标志位，目的节点 */
        offset_lh = 15;    /* 标志位，目的区域，目的子区域，目的 ID，源区域，源子区域，HIORD 之后 */
        break;

    case Q_AND:
        /* 低效，因为我们做了两次 Calvinball 舞蹈 */
        b0 = gen_dnhostop(cstate, addr, Q_SRC);
        b1 = gen_dnhostop(cstate, addr, Q_DST);
        gen_and(b0, b1);
        return b1;

    case Q_DEFAULT:
    case Q_OR:
        /* 低效，因为我们做了两次 Calvinball 舞蹈 */
        b0 = gen_dnhostop(cstate, addr, Q_SRC);
        b1 = gen_dnhostop(cstate, addr, Q_DST);
        gen_or(b0, b1);
        return b1;
    # 如果匹配到地址1，抛出错误信息
    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 如果匹配到地址2，抛出错误信息
    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 如果匹配到地址3，抛出错误信息
    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 如果匹配到地址4，抛出错误信息
    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are not valid qualifiers for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 如果匹配到RA地址，抛出错误信息
    case Q_RA:
        bpf_error(cstate, "'ra' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 如果匹配到TA地址，抛出错误信息
    case Q_TA:
        bpf_error(cstate, "'ta' is not a valid qualifier for addresses other than 802.11 MAC addresses");
        /*NOTREACHED*/

    # 默认情况下，中止程序
    default:
        abort();
        /*NOTREACHED*/
    }
    # 生成链路类型为ETHERTYPE_DN的指令
    b0 = gen_linktype(cstate, ETHERTYPE_DN);
    # 检查是否为pad = 1，长头部情况
    tmp = gen_mcmp(cstate, OR_LINKPL, 2, BPF_H,
        (bpf_u_int32)ntohs(0x0681), (bpf_u_int32)ntohs(0x07FF));
    b1 = gen_cmp(cstate, OR_LINKPL, 2 + 1 + offset_lh,
        BPF_H, (bpf_u_int32)ntohs((u_short)addr));
    gen_and(tmp, b1);
    # 检查是否为pad = 0，长头部情况
    tmp = gen_mcmp(cstate, OR_LINKPL, 2, BPF_B, (bpf_u_int32)0x06,
        (bpf_u_int32)0x7);
    b2 = gen_cmp(cstate, OR_LINKPL, 2 + offset_lh, BPF_H,
        (bpf_u_int32)ntohs((u_short)addr));
    gen_and(tmp, b2);
    gen_or(b2, b1);
    # 检查是否为pad = 1，短头部情况
    tmp = gen_mcmp(cstate, OR_LINKPL, 2, BPF_H,
        (bpf_u_int32)ntohs(0x0281), (bpf_u_int32)ntohs(0x07FF));
    b2 = gen_cmp(cstate, OR_LINKPL, 2 + 1 + offset_sh, BPF_H,
        (bpf_u_int32)ntohs((u_short)addr));
    gen_and(tmp, b2);
    gen_or(b2, b1);
    # 检查是否为pad = 0，短头部情况
    # 生成一个比较指令，比较链路层协议中偏移量为2的字节和0x02
    tmp = gen_mcmp(cstate, OR_LINKPL, 2, BPF_B, (bpf_u_int32)0x02,
        (bpf_u_int32)0x7);
    # 生成一个比较指令，比较链路层协议中偏移量为2加上偏移量sh的字节和addr的网络字节序的16位无符号整数
    b2 = gen_cmp(cstate, OR_LINKPL, 2 + offset_sh, BPF_H,
        (bpf_u_int32)ntohs((u_short)addr));
    # 生成一个与操作指令，将tmp和b2进行与操作
    gen_and(tmp, b2);
    # 生成一个或操作指令，将b2和b1进行或操作
    gen_or(b2, b1);

    # 与操作指令，将b0和b1进行与操作
    gen_and(b0, b1);
    # 返回b1
    return b1;
}

/*
 * 为 MPLS 封装的数据包生成 IPv4 或 IPv6 的检查；
 * 检查底部堆栈位，然后检查 IP 头中的版本号字段。
 */
static struct block *
gen_mpls_linktype(compiler_state_t *cstate, bpf_u_int32 ll_proto)
{
    struct block *b0, *b1;

    switch (ll_proto) {

    case ETHERTYPE_IP:
        /* 匹配底部堆栈位 */
        b0 = gen_mcmp(cstate, OR_LINKPL, (u_int)-2, BPF_B, 0x01, 0x01);
        /* 匹配 IPv4 版本号 */
        b1 = gen_mcmp(cstate, OR_LINKPL, 0, BPF_B, 0x40, 0xf0);
        gen_and(b0, b1);
        return b1;

    case ETHERTYPE_IPV6:
        /* 匹配底部堆栈位 */
        b0 = gen_mcmp(cstate, OR_LINKPL, (u_int)-2, BPF_B, 0x01, 0x01);
        /* 匹配 IPv6 版本号 */
        b1 = gen_mcmp(cstate, OR_LINKPL, 0, BPF_B, 0x60, 0xf0);
        gen_and(b0, b1);
        return b1;

    default:
        /* FIXME 添加其他 L3 协议 ID */
        bpf_error(cstate, "unsupported protocol over mpls");
        /*NOTREACHED*/
    }
}

static struct block *
gen_host(compiler_state_t *cstate, bpf_u_int32 addr, bpf_u_int32 mask,
    int proto, int dir, int type)
{
    struct block *b0, *b1;
    const char *typestr;

    if (type == Q_NET)
        typestr = "net";
    else
        typestr = "host";

    switch (proto) {

    case Q_DEFAULT:
        b0 = gen_host(cstate, addr, mask, Q_IP, dir, type);
        /*
         * 只有在不检查 MPLS 封装的数据包时，才检查非 IPv4 地址。
         */
        if (cstate->label_stack_depth == 0) {
            b1 = gen_host(cstate, addr, mask, Q_ARP, dir, type);
            gen_or(b0, b1);
            b0 = gen_host(cstate, addr, mask, Q_RARP, dir, type);
            gen_or(b1, b0);
        }
        return b0;
    # 如果出现 Q_LINK，表示链接层修改器应用于某个类型，抛出错误
    case Q_LINK:
        bpf_error(cstate, "link-layer modifier applied to %s", typestr);

    # 如果出现 Q_IP，返回基于 IP 地址的主机操作
    case Q_IP:
        return gen_hostop(cstate, addr, mask, dir, ETHERTYPE_IP, 12, 16);

    # 如果出现 Q_RARP，返回基于 RARP 地址的主机操作
    case Q_RARP:
        return gen_hostop(cstate, addr, mask, dir, ETHERTYPE_REVARP, 14, 24);

    # 如果出现 Q_ARP，返回基于 ARP 地址的主机操作
    case Q_ARP:
        return gen_hostop(cstate, addr, mask, dir, ETHERTYPE_ARP, 14, 24);

    # 如果出现 Q_SCTP，抛出错误
    case Q_SCTP:
        bpf_error(cstate, "'sctp' modifier applied to %s", typestr);

    # 如果出现 Q_TCP，抛出错误
    case Q_TCP:
        bpf_error(cstate, "'tcp' modifier applied to %s", typestr);

    # 如果出现 Q_UDP，抛出错误
    case Q_UDP:
        bpf_error(cstate, "'udp' modifier applied to %s", typestr);

    # 如果出现 Q_ICMP，抛出错误
    case Q_ICMP:
        bpf_error(cstate, "'icmp' modifier applied to %s", typestr);

    # 如果出现 Q_IGMP，抛出错误
    case Q_IGMP:
        bpf_error(cstate, "'igmp' modifier applied to %s", typestr);

    # 如果出现 Q_IGRP，抛出错误
    case Q_IGRP:
        bpf_error(cstate, "'igrp' modifier applied to %s", typestr);

    # 如果出现 Q_ATALK，抛出错误，因为 AppleTalk 主机过滤未实现
    case Q_ATALK:
        bpf_error(cstate, "AppleTalk host filtering not implemented");

    # 如果出现 Q_DECNET，返回基于 DECNET 地址的主机操作
    case Q_DECNET:
        return gen_dnhostop(cstate, addr, dir);

    # 如果出现 Q_LAT，抛出错误，因为 LAT 主机过滤未实现
    case Q_LAT:
        bpf_error(cstate, "LAT host filtering not implemented");

    # 如果出现 Q_SCA，抛出错误，因为 SCA 主机过滤未实现
    case Q_SCA:
        bpf_error(cstate, "SCA host filtering not implemented");

    # 如果出现 Q_MOPRC，抛出错误，因为 MOPRC 主机过滤未实现
    case Q_MOPRC:
        bpf_error(cstate, "MOPRC host filtering not implemented");

    # 如果出现 Q_MOPDL，抛出错误，因为 MOPDL 主机过滤未实现
    case Q_MOPDL:
        bpf_error(cstate, "MOPDL host filtering not implemented");

    # 如果出现 Q_IPV6，抛出错误，因为 'ip6' 修改器应用于 IP 主机
    case Q_IPV6:
        bpf_error(cstate, "'ip6' modifier applied to ip host");

    # 如果出现 Q_ICMPV6，抛出错误，因为 'icmp6' 修改器应用于某个类型
    case Q_ICMPV6:
        bpf_error(cstate, "'icmp6' modifier applied to %s", typestr);

    # 如果出现 Q_AH，抛出错误，因为 'ah' 修改器应用于某个类型
    case Q_AH:
        bpf_error(cstate, "'ah' modifier applied to %s", typestr);

    # 如果出现 Q_ESP，抛出错误，因为 'esp' 修改器应用于某个类型
    case Q_ESP:
        bpf_error(cstate, "'esp' modifier applied to %s", typestr);

    # 如果出现 Q_PIM，抛出错误，因为 'pim' 修改器应用于某个类型
    case Q_PIM:
        bpf_error(cstate, "'pim' modifier applied to %s", typestr);

    # 如果出现 Q_VRRP，抛出错误，因为 'vrrp' 修改器应用于某个类型
    case Q_VRRP:
        bpf_error(cstate, "'vrrp' modifier applied to %s", typestr);

    # 如果出现 Q_AARP，抛出错误，因为 AARP 主机过滤未实现
    case Q_AARP:
        bpf_error(cstate, "AARP host filtering not implemented");
    # 如果过滤器类型为 Q_ISO，则抛出错误，表示未实现 ISO 主机过滤
    case Q_ISO:
        bpf_error(cstate, "ISO host filtering not implemented");

    # 如果过滤器类型为 Q_ESIS，则抛出错误，表示"'esis' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ESIS:
        bpf_error(cstate, "'esis' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS，则抛出错误，表示"'isis' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS:
        bpf_error(cstate, "'isis' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_CLNP，则抛出错误，表示"'clnp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_CLNP:
        bpf_error(cstate, "'clnp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_STP，则抛出错误，表示"'stp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_STP:
        bpf_error(cstate, "'stp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_IPX，则抛出错误，表示未实现 IPX 主机过滤
    case Q_IPX:
        bpf_error(cstate, "IPX host filtering not implemented");

    # 如果过滤器类型为 Q_NETBEUI，则抛出错误，表示"'netbeui' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_NETBEUI:
        bpf_error(cstate, "'netbeui' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_L1，则抛出错误，表示"'l1' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_L1:
        bpf_error(cstate, "'l1' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_L2，则抛出错误，表示"'l2' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_L2:
        bpf_error(cstate, "'l2' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_IIH，则抛出错误，表示"'iih' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_IIH:
        bpf_error(cstate, "'iih' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_SNP，则抛出错误，表示"'snp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_SNP:
        bpf_error(cstate, "'snp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_CSNP，则抛出错误，表示"'csnp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_CSNP:
        bpf_error(cstate, "'csnp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_PSNP，则抛出错误，表示"'psnp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_PSNP:
        bpf_error(cstate, "'psnp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_ISIS_LSP，则抛出错误，表示"'lsp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_ISIS_LSP:
        bpf_error(cstate, "'lsp' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_RADIO，则抛出错误，表示"'radio' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_RADIO:
        bpf_error(cstate, "'radio' modifier applied to %s", typestr);

    # 如果过滤器类型为 Q_CARP，则抛出错误，表示"'carp' 修饰符应用于 %s"，其中 %s 为 typestr
    case Q_CARP:
        bpf_error(cstate, "'carp' modifier applied to %s", typestr);

    # 默认情况下，中止程序
    default:
        abort();
    }
    /*NOTREACHED*/
    # 如果定义了 INET6
    static struct block *
    gen_host6(compiler_state_t *cstate, struct in6_addr *addr,
        struct in6_addr *mask, int proto, int dir, int type)
    {
        const char *typestr;

        # 如果类型是 Q_NET，则设置类型字符串为 "net"，否则设置为 "host"
        if (type == Q_NET)
            typestr = "net";
        else
            typestr = "host";

        # 根据协议类型进行不同的处理
        switch (proto) {

        # 如果协议类型是 Q_DEFAULT，则递归调用 gen_host6 函数
        case Q_DEFAULT:
            return gen_host6(cstate, addr, mask, Q_IPV6, dir, type);

        # 如果协议类型是 Q_LINK，则报错
        case Q_LINK:
            bpf_error(cstate, "link-layer modifier applied to ip6 %s", typestr);

        # 如果协议类型是 Q_IP，则报错
        case Q_IP:
            bpf_error(cstate, "'ip' modifier applied to ip6 %s", typestr);

        # 其他协议类型依次类推，都报错
        ...
        ...
        ...

        # 如果协议类型是 Q_IPV6，则调用 gen_hostop6 函数
        case Q_IPV6:
            return gen_hostop6(cstate, addr, mask, dir, ETHERTYPE_IPV6, 8, 24);
    # 如果遇到 ICMPv6 协议，抛出错误
    case Q_ICMPV6:
        bpf_error(cstate, "'icmp6' modifier applied to ip6 %s", typestr)

    # 如果遇到 AH 协议，抛出错误
    case Q_AH:
        bpf_error(cstate, "'ah' modifier applied to ip6 %s", typestr)

    # 如果遇到 ESP 协议，抛出错误
    case Q_ESP:
        bpf_error(cstate, "'esp' modifier applied to ip6 %s", typestr)

    # 如果遇到 PIM 协议，抛出错误
    case Q_PIM:
        bpf_error(cstate, "'pim' modifier applied to ip6 %s", typestr)

    # 如果遇到 VRRP 协议，抛出错误
    case Q_VRRP:
        bpf_error(cstate, "'vrrp' modifier applied to ip6 %s", typestr)

    # 如果遇到 AARP 协议，抛出错误
    case Q_AARP:
        bpf_error(cstate, "'aarp' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISO 协议，抛出错误
    case Q_ISO:
        bpf_error(cstate, "'iso' modifier applied to ip6 %s", typestr)

    # 如果遇到 ESIS 协议，抛出错误
    case Q_ESIS:
        bpf_error(cstate, "'esis' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS 协议，抛出错误
    case Q_ISIS:
        bpf_error(cstate, "'isis' modifier applied to ip6 %s", typestr)

    # 如果遇到 CLNP 协议，抛出错误
    case Q_CLNP:
        bpf_error(cstate, "'clnp' modifier applied to ip6 %s", typestr)

    # 如果遇到 STP 协议，抛出错误
    case Q_STP:
        bpf_error(cstate, "'stp' modifier applied to ip6 %s", typestr)

    # 如果遇到 IPX 协议，抛出错误
    case Q_IPX:
        bpf_error(cstate, "'ipx' modifier applied to ip6 %s", typestr)

    # 如果遇到 NETBEUI 协议，抛出错误
    case Q_NETBEUI:
        bpf_error(cstate, "'netbeui' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS L1 协议，抛出错误
    case Q_ISIS_L1:
        bpf_error(cstate, "'l1' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS L2 协议，抛出错误
    case Q_ISIS_L2:
        bpf_error(cstate, "'l2' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS IIH 协议，抛出错误
    case Q_ISIS_IIH:
        bpf_error(cstate, "'iih' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS SNP 协议，抛出错误
    case Q_ISIS_SNP:
        bpf_error(cstate, "'snp' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS CSNP 协议，抛出错误
    case Q_ISIS_CSNP:
        bpf_error(cstate, "'csnp' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS PSNP 协议，抛出错误
    case Q_ISIS_PSNP:
        bpf_error(cstate, "'psnp' modifier applied to ip6 %s", typestr)

    # 如果遇到 ISIS LSP 协议，抛出错误
    case Q_ISIS_LSP:
        bpf_error(cstate, "'lsp' modifier applied to ip6 %s", typestr)

    # 如果遇到 RADIO 协议，抛出错误
    case Q_RADIO:
        bpf_error(cstate, "'radio' modifier applied to ip6 %s", typestr)
    # 如果匹配到 Q_CARP 情况，则输出错误信息，指明 'carp' 修饰符应用于 ip6 类型的情况
    case Q_CARP:
        bpf_error(cstate, "'carp' modifier applied to ip6 %s", typestr);

    # 如果不匹配任何情况，则终止程序
    default:
        abort();
    }
    # 标记代码不会执行到这里
    /*NOTREACHED*/
    # 结束符号，表示条件编译结束
    }
#endif

#ifndef INET6
# 定义一个函数，生成网关的过滤器块
static struct block *
gen_gateway(compiler_state_t *cstate, const u_char *eaddr,
    struct addrinfo *alist, int proto, int dir)
{
    struct block *b0, *b1, *tmp;
    struct addrinfo *ai;
    struct sockaddr_in *sin;

    # 如果方向不为0，抛出错误
    if (dir != 0)
        bpf_error(cstate, "direction applied to 'gateway'");

    # 根据协议类型进行不同的处理
    switch (proto) {
    case Q_DEFAULT:
    case Q_IP:
    case Q_ARP:
    }
    # 抛出错误，表示'gateway'的修饰符不合法
    bpf_error(cstate, "illegal modifier of 'gateway'");
    /*NOTREACHED*/
}
#endif

# 定义一个函数，生成协议缩写的内部过滤器块
static struct block *
gen_proto_abbrev_internal(compiler_state_t *cstate, int proto)
{
    struct block *b0;
    struct block *b1;

    # 根据协议类型进行不同的处理
    switch (proto) {

    case Q_SCTP:
        b1 = gen_proto(cstate, IPPROTO_SCTP, Q_DEFAULT, Q_DEFAULT);
        break;

    case Q_TCP:
        b1 = gen_proto(cstate, IPPROTO_TCP, Q_DEFAULT, Q_DEFAULT);
        break;

    case Q_UDP:
        b1 = gen_proto(cstate, IPPROTO_UDP, Q_DEFAULT, Q_DEFAULT);
        break;

    case Q_ICMP:
        b1 = gen_proto(cstate, IPPROTO_ICMP, Q_IP, Q_DEFAULT);
        break;

# 如果未定义IPPROTO_IGMP，则定义为2
#ifndef    IPPROTO_IGMP
#define    IPPROTO_IGMP    2
#endif

    case Q_IGMP:
        b1 = gen_proto(cstate, IPPROTO_IGMP, Q_IP, Q_DEFAULT);
        break;

# 如果未定义IPPROTO_IGRP，则定义为9
#ifndef    IPPROTO_IGRP
#define    IPPROTO_IGRP    9
#endif
    case Q_IGRP:
        b1 = gen_proto(cstate, IPPROTO_IGRP, Q_IP, Q_DEFAULT);
        break;

# 如果未定义IPPROTO_PIM，则定义为103
#ifndef IPPROTO_PIM
#define IPPROTO_PIM    103
#endif

    case Q_PIM:
        b1 = gen_proto(cstate, IPPROTO_PIM, Q_DEFAULT, Q_DEFAULT);
        break;

# 如果未定义IPPROTO_VRRP，则定义为112
#ifndef IPPROTO_VRRP
#define IPPROTO_VRRP    112
#endif

    case Q_VRRP:
        b1 = gen_proto(cstate, IPPROTO_VRRP, Q_IP, Q_DEFAULT);
        break;

# 如果未定义IPPROTO_CARP，则定义为112
#ifndef IPPROTO_CARP
#define IPPROTO_CARP    112
#endif

    case Q_CARP:
        b1 = gen_proto(cstate, IPPROTO_CARP, Q_IP, Q_DEFAULT);
        break;

    case Q_IP:
        b1 = gen_linktype(cstate, ETHERTYPE_IP);
        break;

    case Q_ARP:
        b1 = gen_linktype(cstate, ETHERTYPE_ARP);
        break;
    # 如果是 Q_RARP，生成以太网类型为 ETHERTYPE_REVARP 的过滤器
    case Q_RARP:
        b1 = gen_linktype(cstate, ETHERTYPE_REVARP);
        break;

    # 如果是 Q_LINK，报错，因为链路层应用在错误的上下文中
    case Q_LINK:
        bpf_error(cstate, "link layer applied in wrong context");

    # 如果是 Q_ATALK，生成以太网类型为 ETHERTYPE_ATALK 的过滤器
    case Q_ATALK:
        b1 = gen_linktype(cstate, ETHERTYPE_ATALK);
        break;

    # 如果是 Q_AARP，生成以太网类型为 ETHERTYPE_AARP 的过滤器
    case Q_AARP:
        b1 = gen_linktype(cstate, ETHERTYPE_AARP);
        break;

    # 如果是 Q_DECNET，生成以太网类型为 ETHERTYPE_DN 的过滤器
    case Q_DECNET:
        b1 = gen_linktype(cstate, ETHERTYPE_DN);
        break;

    # 如果是 Q_SCA，生成以太网类型为 ETHERTYPE_SCA 的过滤器
    case Q_SCA:
        b1 = gen_linktype(cstate, ETHERTYPE_SCA);
        break;

    # 如果是 Q_LAT，生成以太网类型为 ETHERTYPE_LAT 的过滤器
    case Q_LAT:
        b1 = gen_linktype(cstate, ETHERTYPE_LAT);
        break;

    # 如果是 Q_MOPDL，生成以太网类型为 ETHERTYPE_MOPDL 的过滤器
    case Q_MOPDL:
        b1 = gen_linktype(cstate, ETHERTYPE_MOPDL);
        break;

    # 如果是 Q_MOPRC，生成以太网类型为 ETHERTYPE_MOPRC 的过滤器
    case Q_MOPRC:
        b1 = gen_linktype(cstate, ETHERTYPE_MOPRC);
        break;

    # 如果是 Q_IPV6，生成以太网类型为 ETHERTYPE_IPV6 的过滤器
    case Q_IPV6:
        b1 = gen_linktype(cstate, ETHERTYPE_IPV6);
        break;
#ifndef IPPROTO_ICMPV6
#define IPPROTO_ICMPV6    58
#endif
    // 如果未定义 IPPROTO_ICMPV6，则定义为 58
    case Q_ICMPV6:
        // 生成协议过滤器，过滤 ICMPv6 协议
        b1 = gen_proto(cstate, IPPROTO_ICMPV6, Q_IPV6, Q_DEFAULT);
        break;

#ifndef IPPROTO_AH
#define IPPROTO_AH    51
#endif
    // 如果未定义 IPPROTO_AH，则定义为 51
    case Q_AH:
        // 生成协议过滤器，过滤 AH 协议
        b1 = gen_proto(cstate, IPPROTO_AH, Q_DEFAULT, Q_DEFAULT);
        break;

#ifndef IPPROTO_ESP
#define IPPROTO_ESP    50
#endif
    // 如果未定义 IPPROTO_ESP，则定义为 50
    case Q_ESP:
        // 生成协议过滤器，过滤 ESP 协议
        b1 = gen_proto(cstate, IPPROTO_ESP, Q_DEFAULT, Q_DEFAULT);
        break;

    case Q_ISO:
        // 生成链路类型过滤器，过滤 ISO 协议
        b1 = gen_linktype(cstate, LLCSAP_ISONS);
        break;

    case Q_ESIS:
        // 生成协议过滤器，过滤 ES-IS 协议
        b1 = gen_proto(cstate, ISO9542_ESIS, Q_ISO, Q_DEFAULT);
        break;

    case Q_ISIS:
        // 生成协议过滤器，过滤 IS-IS 协议
        b1 = gen_proto(cstate, ISO10589_ISIS, Q_ISO, Q_DEFAULT);
        break;

    case Q_ISIS_L1: /* all IS-IS Level1 PDU-Types */
        // 生成协议过滤器，过滤 IS-IS Level1 PDU-Types
        b0 = gen_proto(cstate, ISIS_L1_LAN_IIH, Q_ISIS, Q_DEFAULT);
        b1 = gen_proto(cstate, ISIS_PTP_IIH, Q_ISIS, Q_DEFAULT); /* FIXME extract the circuit-type bits */
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L1_LSP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L1_CSNP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L1_PSNP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        break;

    case Q_ISIS_L2: /* all IS-IS Level2 PDU-Types */
        // 生成协议过滤器，过滤 IS-IS Level2 PDU-Types
        b0 = gen_proto(cstate, ISIS_L2_LAN_IIH, Q_ISIS, Q_DEFAULT);
        b1 = gen_proto(cstate, ISIS_PTP_IIH, Q_ISIS, Q_DEFAULT); /* FIXME extract the circuit-type bits */
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L2_LSP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L2_CSNP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        b0 = gen_proto(cstate, ISIS_L2_PSNP, Q_ISIS, Q_DEFAULT);
        gen_or(b0, b1);
        break;
    # 如果是 ISIS Hello PDU 类型
    case Q_ISIS_IIH: /* all IS-IS Hello PDU-Types */
        # 生成 ISIS L1 LAN IIH 协议
        b0 = gen_proto(cstate, ISIS_L1_LAN_IIH, Q_ISIS, Q_DEFAULT);
        # 生成 ISIS L2 LAN IIH 协议
        b1 = gen_proto(cstate, ISIS_L2_LAN_IIH, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        # 生成 ISIS PTP IIH 协议
        b0 = gen_proto(cstate, ISIS_PTP_IIH, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        break;

    # 如果是 ISIS LSP 类型
    case Q_ISIS_LSP:
        # 生成 ISIS L1 LSP 协议
        b0 = gen_proto(cstate, ISIS_L1_LSP, Q_ISIS, Q_DEFAULT);
        # 生成 ISIS L2 LSP 协议
        b1 = gen_proto(cstate, ISIS_L2_LSP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        break;

    # 如果是 ISIS SNP 类型
    case Q_ISIS_SNP:
        # 生成 ISIS L1 CSNP 协议
        b0 = gen_proto(cstate, ISIS_L1_CSNP, Q_ISIS, Q_DEFAULT);
        # 生成 ISIS L2 CSNP 协议
        b1 = gen_proto(cstate, ISIS_L2_CSNP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        # 生成 ISIS L1 PSNP 协议
        b0 = gen_proto(cstate, ISIS_L1_PSNP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        # 生成 ISIS L2 PSNP 协议
        b0 = gen_proto(cstate, ISIS_L2_PSNP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        break;

    # 如果是 ISIS CSNP 类型
    case Q_ISIS_CSNP:
        # 生成 ISIS L1 CSNP 协议
        b0 = gen_proto(cstate, ISIS_L1_CSNP, Q_ISIS, Q_DEFAULT);
        # 生成 ISIS L2 CSNP 协议
        b1 = gen_proto(cstate, ISIS_L2_CSNP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        break;

    # 如果是 ISIS PSNP 类型
    case Q_ISIS_PSNP:
        # 生成 ISIS L1 PSNP 协议
        b0 = gen_proto(cstate, ISIS_L1_PSNP, Q_ISIS, Q_DEFAULT);
        # 生成 ISIS L2 PSNP 协议
        b1 = gen_proto(cstate, ISIS_L2_PSNP, Q_ISIS, Q_DEFAULT);
        # 将 b0 和 b1 进行逻辑或操作
        gen_or(b0, b1);
        break;

    # 如果是 CLNP 类型
    case Q_CLNP:
        # 生成 ISO8473 CLNP 协议
        b1 = gen_proto(cstate, ISO8473_CLNP, Q_ISO, Q_DEFAULT);
        break;

    # 如果是 STP 类型
    case Q_STP:
        # 生成 LLCSAP 802.1D 协议
        b1 = gen_linktype(cstate, LLCSAP_8021D);
        break;

    # 如果是 IPX 类型
    case Q_IPX:
        # 生成 LLCSAP IPX 协议
        b1 = gen_linktype(cstate, LLCSAP_IPX);
        break;

    # 如果是 NETBEUI 类型
    case Q_NETBEUI:
        # 生成 LLCSAP NETBEUI 协议
        b1 = gen_linktype(cstate, LLCSAP_NETBEUI);
        break;

    # 如果是 RADIO 类型
    case Q_RADIO:
        # 报错，'radio' 不是有效的协议类型
        bpf_error(cstate, "'radio' is not a valid protocol type");

    # 默认情况
    default:
        # 中止程序
        abort();
    }
    # 返回 b1
    return b1;
}
# 生成用于简化协议的结构体指针
struct block *
gen_proto_abbrev(compiler_state_t *cstate, int proto)
{
    '''
    捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
    '''
    if (setjmp(cstate->top_ctx))
        return (NULL);

    return gen_proto_abbrev_internal(cstate, proto);
}

# 生成 IP 分片的结构体指针
static struct block *
gen_ipfrag(compiler_state_t *cstate)
{
    struct slist *s;
    struct block *b;

    # 不是 IPv4 分片，除了第一个分片
    s = gen_load_a(cstate, OR_LINKPL, 6, BPF_H);
    b = new_block(cstate, JMP(BPF_JSET));
    b->s.k = 0x1fff;
    b->stmts = s;
    gen_not(b);

    return b;
}

'''
生成对传输层头部中指定偏移处的端口值的比较
XXX - 这处理了位于链路层头部之前的可变长度前缀，如 radiotap 或 AVS 无线电前缀，但不处理可变长度的链路层头部（如 Token Ring 或 802.11 头部）。
'''
static struct block *
gen_portatom(compiler_state_t *cstate, int off, bpf_u_int32 v)
{
    return gen_cmp(cstate, OR_TRAN_IPV4, off, BPF_H, v);
}

static struct block *
gen_portatom6(compiler_state_t *cstate, int off, bpf_u_int32 v)
{
    return gen_cmp(cstate, OR_TRAN_IPV6, off, BPF_H, v);
}

static struct block *
gen_portop(compiler_state_t *cstate, u_int port, u_int proto, int dir)
{
    struct block *b0, *b1, *tmp;

    # IP 协议 'proto' 并且不是除了第一个分片之外的分片
    tmp = gen_cmp(cstate, OR_LINKPL, 9, BPF_B, proto);
    b0 = gen_ipfrag(cstate);
    gen_and(tmp, b0);

    switch (dir) {
    case Q_SRC:
        b1 = gen_portatom(cstate, 0, port);
        break;

    case Q_DST:
        b1 = gen_portatom(cstate, 2, port);
        break;

    case Q_AND:
        tmp = gen_portatom(cstate, 0, port);
        b1 = gen_portatom(cstate, 2, port);
        gen_and(tmp, b1);
        break;

    case Q_DEFAULT:
    # 如果是 Q_OR 情况，生成端口原子条件
    case Q_OR:
        tmp = gen_portatom(cstate, 0, port);
        b1 = gen_portatom(cstate, 2, port);
        # 生成逻辑或条件
        gen_or(tmp, b1);
        # 退出 switch 语句
        break;

    # 如果是 Q_ADDR1 情况，报错并标记为不可达
    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are not valid qualifiers for ports");
        /*NOTREACHED*/

    # 如果是 Q_ADDR2 情况，报错并标记为不可达
    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are not valid qualifiers for ports");
        /*NOTREACHED*/

    # 如果是 Q_ADDR3 情况，报错并标记为不可达
    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are not valid qualifiers for ports");
        /*NOTREACHED*/

    # 如果是 Q_ADDR4 情况，报错并标记为不可达
    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are not valid qualifiers for ports");
        /*NOTREACHED*/

    # 如果是 Q_RA 情况，报错并标记为不可达
    case Q_RA:
        bpf_error(cstate, "'ra' is not a valid qualifier for ports");
        /*NOTREACHED*/

    # 如果是 Q_TA 情况，报错并标记为不可达
    case Q_TA:
        bpf_error(cstate, "'ta' is not a valid qualifier for ports");
        /*NOTREACHED*/

    # 默认情况下，中止程序
    default:
        abort();
        /*NOTREACHED*/
    }
    # 生成逻辑与条件
    gen_and(b0, b1);

    # 返回 b1
    return b1;
}
static struct block *
gen_port(compiler_state_t *cstate, u_int port, int ip_proto, int dir)
{
    struct block *b0, *b1, *tmp;

    /*
     * ether proto ip
     *
     * For FDDI, RFC 1188 says that SNAP encapsulation is used,
     * not LLC encapsulation with LLCSAP_IP.
     *
     * For IEEE 802 networks - which includes 802.5 token ring
     * (which is what DLT_IEEE802 means) and 802.11 - RFC 1042
     * says that SNAP encapsulation is used, not LLC encapsulation
     * with LLCSAP_IP.
     *
     * For LLC-encapsulated ATM/"Classical IP", RFC 1483 and
     * RFC 2225 say that SNAP encapsulation is used, not LLC
     * encapsulation with LLCSAP_IP.
     *
     * So we always check for ETHERTYPE_IP.
     */
    b0 = gen_linktype(cstate, ETHERTYPE_IP);  // 生成以太网类型为 IP 的过滤器

    switch (ip_proto) {
    case IPPROTO_UDP:
    case IPPROTO_TCP:
    case IPPROTO_SCTP:
        b1 = gen_portop(cstate, port, (u_int)ip_proto, dir);  // 生成指定端口和协议的过滤器
        break;

    case PROTO_UNDEF:
        tmp = gen_portop(cstate, port, IPPROTO_TCP, dir);  // 生成 TCP 协议的指定端口过滤器
        b1 = gen_portop(cstate, port, IPPROTO_UDP, dir);  // 生成 UDP 协议的指定端口过滤器
        gen_or(tmp, b1);  // 将两个过滤器进行或运算
        tmp = gen_portop(cstate, port, IPPROTO_SCTP, dir);  // 生成 SCTP 协议的指定端口过滤器
        gen_or(tmp, b1);  // 将两个过滤器进行或运算
        break;

    default:
        abort();  // 终止程序
    }
    gen_and(b0, b1);  // 将两个过滤器进行与运算
    return b1;  // 返回生成的过滤器
}

struct block *
gen_portop6(compiler_state_t *cstate, u_int port, u_int proto, int dir)
{
    struct block *b0, *b1, *tmp;

    /* ip6 proto 'proto' */
    /* XXX - catch the first fragment of a fragmented packet? */
    b0 = gen_cmp(cstate, OR_LINKPL, 6, BPF_B, proto);  // 生成 IPv6 协议类型为指定值的过滤器

    switch (dir) {
    case Q_SRC:
        b1 = gen_portatom6(cstate, 0, port);  // 生成 IPv6 源端口为指定值的过滤器
        break;

    case Q_DST:
        b1 = gen_portatom6(cstate, 2, port);  // 生成 IPv6 目的端口为指定值的过滤器
        break;

    case Q_AND:
        tmp = gen_portatom6(cstate, 0, port);  // 生成 IPv6 源端口为指定值的过滤器
        b1 = gen_portatom6(cstate, 2, port);  // 生成 IPv6 目的端口为指定值的过滤器
        gen_and(tmp, b1);  // 将两个过滤器进行与运算
        break;

    case Q_DEFAULT:
    # 如果是逻辑或操作符
    case Q_OR:
        # 生成第一个端口原子
        tmp = gen_portatom6(cstate, 0, port);
        # 生成第二个端口原子
        b1 = gen_portatom6(cstate, 2, port);
        # 生成逻辑或操作
        gen_or(tmp, b1);
        # 结束当前 case
        break;

    # 如果是其他情况
    default:
        # 中止程序
        abort();
    }
    # 生成逻辑与操作
    gen_and(b0, b1);

    # 返回第二个端口原子
    return b1;
}
# 生成 IPv6 端口过滤器
static struct block *
gen_port6(compiler_state_t *cstate, u_int port, int ip_proto, int dir)
{
    struct block *b0, *b1, *tmp;

    # 生成连接类型为 IPv6 的过滤器
    b0 = gen_linktype(cstate, ETHERTYPE_IPV6);

    # 根据 IP 协议类型生成端口操作过滤器
    switch (ip_proto) {
    case IPPROTO_UDP:
    case IPPROTO_TCP:
    case IPPROTO_SCTP:
        b1 = gen_portop6(cstate, port, (u_int)ip_proto, dir);
        break;

    case PROTO_UNDEF:
        tmp = gen_portop6(cstate, port, IPPROTO_TCP, dir);
        b1 = gen_portop6(cstate, port, IPPROTO_UDP, dir);
        gen_or(tmp, b1);
        tmp = gen_portop6(cstate, port, IPPROTO_SCTP, dir);
        gen_or(tmp, b1);
        break;

    default:
        abort();
    }
    # 生成连接类型和端口操作的交集过滤器
    gen_and(b0, b1);
    return b1;
}

# 生成端口范围原子过滤器
static struct block *
gen_portrangeatom(compiler_state_t *cstate, u_int off, bpf_u_int32 v1,
    bpf_u_int32 v2)
{
    struct block *b1, *b2;

    if (v1 > v2) {
        # 如果端口范围不是从小到大，则交换端口范围的值
        bpf_u_int32 vtemp;

        vtemp = v1;
        v1 = v2;
        v2 = vtemp;
    }

    # 生成大于等于 v1 和小于等于 v2 的比较过滤器
    b1 = gen_cmp_ge(cstate, OR_TRAN_IPV4, off, BPF_H, v1);
    b2 = gen_cmp_le(cstate, OR_TRAN_IPV4, off, BPF_H, v2);

    # 生成大于等于 v1 和小于等于 v2 的交集过滤器
    gen_and(b1, b2);

    return b2;
}

# 生成端口范围操作过滤器
static struct block *
gen_portrangeop(compiler_state_t *cstate, u_int port1, u_int port2,
    bpf_u_int32 proto, int dir)
{
    struct block *b0, *b1, *tmp;

    # IP 协议类型为 'proto'，并且不是第一个分片
    tmp = gen_cmp(cstate, OR_LINKPL, 9, BPF_B, proto);
    b0 = gen_ipfrag(cstate);
    gen_and(tmp, b0);

    # 根据方向生成端口范围原子过滤器
    switch (dir) {
    case Q_SRC:
        b1 = gen_portrangeatom(cstate, 0, port1, port2);
        break;

    case Q_DST:
        b1 = gen_portrangeatom(cstate, 2, port1, port2);
        break;

    case Q_AND:
        tmp = gen_portrangeatom(cstate, 0, port1, port2);
        b1 = gen_portrangeatom(cstate, 2, port1, port2);
        gen_and(tmp, b1);
        break;

    case Q_DEFAULT:
    # 如果是 Q_OR 情况，生成端口范围原子表达式，并连接成 OR 表达式
    case Q_OR:
        tmp = gen_portrangeatom(cstate, 0, port1, port2);
        b1 = gen_portrangeatom(cstate, 2, port1, port2);
        gen_or(tmp, b1);
        break;

    # 如果是 Q_ADDR1 情况，报错并标记为不可达
    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are not valid qualifiers for port ranges");
        /*NOTREACHED*/

    # 如果是 Q_ADDR2 情况，报错并标记为不可达
    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are not valid qualifiers for port ranges");
        /*NOTREACHED*/

    # 如果是 Q_ADDR3 情况，报错并标记为不可达
    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are not valid qualifiers for port ranges");
        /*NOTREACHED*/

    # 如果是 Q_ADDR4 情况，报错并标记为不可达
    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are not valid qualifiers for port ranges");
        /*NOTREACHED*/

    # 如果是 Q_RA 情况，报错并标记为不可达
    case Q_RA:
        bpf_error(cstate, "'ra' is not a valid qualifier for port ranges");
        /*NOTREACHED*/

    # 如果是 Q_TA 情况，报错并标记为不可达
    case Q_TA:
        bpf_error(cstate, "'ta' is not a valid qualifier for port ranges");
        /*NOTREACHED*/

    # 默认情况下，中止程序并标记为不可达
    default:
        abort();
        /*NOTREACHED*/
    }
    # 生成并连接 AND 表达式
    gen_and(b0, b1);

    # 返回结果
    return b1;
}
# 结束函数定义

static struct block *
gen_portrange(compiler_state_t *cstate, u_int port1, u_int port2, int ip_proto,
    int dir)
{
    struct block *b0, *b1, *tmp;

    /* link proto ip */
    # 生成一个比较链路层协议为 IP 的块
    b0 = gen_linktype(cstate, ETHERTYPE_IP);

    switch (ip_proto) {
    case IPPROTO_UDP:
    case IPPROTO_TCP:
    case IPPROTO_SCTP:
        # 生成一个端口范围操作的块
        b1 = gen_portrangeop(cstate, port1, port2, (bpf_u_int32)ip_proto,
            dir);
        break;

    case PROTO_UNDEF:
        # 生成一个 TCP 端口范围操作的块
        tmp = gen_portrangeop(cstate, port1, port2, IPPROTO_TCP, dir);
        # 生成一个 UDP 端口范围操作的块
        b1 = gen_portrangeop(cstate, port1, port2, IPPROTO_UDP, dir);
        # 将两个块进行逻辑或操作
        gen_or(tmp, b1);
        # 生成一个 SCTP 端口范围操作的块
        tmp = gen_portrangeop(cstate, port1, port2, IPPROTO_SCTP, dir);
        # 将两个块进行逻辑或操作
        gen_or(tmp, b1);
        break;

    default:
        # 终止程序执行
        abort();
    }
    # 生成一个逻辑与操作的块
    gen_and(b0, b1);
    # 返回生成的块
    return b1;
}

static struct block *
gen_portrangeatom6(compiler_state_t *cstate, u_int off, bpf_u_int32 v1,
    bpf_u_int32 v2)
{
    struct block *b1, *b2;

    if (v1 > v2) {
        '''
         * Reverse the order of the ports, so v1 is the lower one.
         '''
        # 交换端口的顺序，使得 v1 是较小的端口
        bpf_u_int32 vtemp;

        vtemp = v1;
        v1 = v2;
        v2 = vtemp;
    }

    # 生成一个比较操作的块，比较 off 偏移处的值是否大于等于 v1
    b1 = gen_cmp_ge(cstate, OR_TRAN_IPV6, off, BPF_H, v1);
    # 生成一个比较操作的块，比较 off 偏移处的值是否小于等于 v2
    b2 = gen_cmp_le(cstate, OR_TRAN_IPV6, off, BPF_H, v2);

    # 生成一个逻辑与操作的块
    gen_and(b1, b2);

    # 返回生成的块
    return b2;
}

static struct block *
gen_portrangeop6(compiler_state_t *cstate, u_int port1, u_int port2,
    bpf_u_int32 proto, int dir)
{
    struct block *b0, *b1, *tmp;

    /* ip6 proto 'proto' */
    # 生成一个比较操作的块，比较链路层协议为 IPv6
    # XXX - catch the first fragment of a fragmented packet?
    b0 = gen_cmp(cstate, OR_LINKPL, 6, BPF_B, proto);

    switch (dir) {
    case Q_SRC:
        # 生成一个 IPv6 源端口范围原子操作的块
        b1 = gen_portrangeatom6(cstate, 0, port1, port2);
        break;

    case Q_DST:
        # 生成一个 IPv6 目的端口范围原子操作的块
        b1 = gen_portrangeatom6(cstate, 2, port1, port2);
        break;

    case Q_AND:
        # 生成一个 IPv6 源端口范围原子操作的块
        tmp = gen_portrangeatom6(cstate, 0, port1, port2);
        # 生成一个 IPv6 目的端口范围原子操作的块
        b1 = gen_portrangeatom6(cstate, 2, port1, port2);
        # 将两个块进行逻辑与操作
        gen_and(tmp, b1);
        break;
    # 如果匹配到默认情况或者逻辑或情况
    case Q_DEFAULT:
    case Q_OR:
        # 生成端口范围原子表达式，设置为0
        tmp = gen_portrangeatom6(cstate, 0, port1, port2);
        # 生成端口范围原子表达式，设置为2
        b1 = gen_portrangeatom6(cstate, 2, port1, port2);
        # 生成逻辑或表达式
        gen_or(tmp, b1);
        # 跳出switch语句
        break;

    # 如果匹配到其他情况
    default:
        # 中止程序
        abort();
    }
    # 生成逻辑与表达式
    gen_and(b0, b1);

    # 返回b1
    return b1;
}
# 定义一个静态函数，生成一个端口范围的过滤器，针对 IPv6 地址
static struct block *
gen_portrange6(compiler_state_t *cstate, u_int port1, u_int port2, int ip_proto,
    int dir)
{
    struct block *b0, *b1, *tmp;

    /* link proto ip6 */
    # 生成一个过滤器，匹配 IPv6 协议
    b0 = gen_linktype(cstate, ETHERTYPE_IPV6);

    # 根据 IP 协议类型进行不同的处理
    switch (ip_proto) {
    case IPPROTO_UDP:
    case IPPROTO_TCP:
    case IPPROTO_SCTP:
        # 生成一个端口范围的过滤器，匹配指定的端口范围和协议类型
        b1 = gen_portrangeop6(cstate, port1, port2, (bpf_u_int32)ip_proto,
            dir);
        break;

    case PROTO_UNDEF:
        # 生成一个端口范围的过滤器，匹配指定的端口范围和 TCP 协议类型
        tmp = gen_portrangeop6(cstate, port1, port2, IPPROTO_TCP, dir);
        # 生成一个端口范围的过滤器，匹配指定的端口范围和 UDP 协议类型
        b1 = gen_portrangeop6(cstate, port1, port2, IPPROTO_UDP, dir);
        # 将两个过滤器进行逻辑或操作
        gen_or(tmp, b1);
        # 生成一个端口范围的过滤器，匹配指定的端口范围和 SCTP 协议类型
        tmp = gen_portrangeop6(cstate, port1, port2, IPPROTO_SCTP, dir);
        # 将两个过滤器进行逻辑或操作
        gen_or(tmp, b1);
        break;

    default:
        # 终止程序执行
        abort();
    }
    # 生成一个逻辑与操作的过滤器
    gen_and(b0, b1);
    # 返回生成的过滤器
    return b1;
}

# 查找协议类型对应的数值
static int
lookup_proto(compiler_state_t *cstate, const char *name, int proto)
{
    register int v;

    # 根据不同的协议类型进行处理
    switch (proto) {

    case Q_DEFAULT:
    case Q_IP:
    case Q_IPV6:
        # 根据协议名称查找对应的协议数值
        v = pcap_nametoproto(name);
        if (v == PROTO_UNDEF)
            # 抛出错误
            bpf_error(cstate, "unknown ip proto '%s'", name);
        break;

    case Q_LINK:
        # 根据链路层协议名称查找对应的协议数值
        v = pcap_nametoeproto(name);
        if (v == PROTO_UNDEF) {
            # 如果找不到，再根据 LLC 协议名称查找对应的协议数值
            v = pcap_nametollc(name);
            if (v == PROTO_UNDEF)
                # 抛出错误
                bpf_error(cstate, "unknown ether proto '%s'", name);
        }
        break;

    case Q_ISO:
        # 根据 OSI 协议名称查找对应的协议数值
        if (strcmp(name, "esis") == 0)
            v = ISO9542_ESIS;
        else if (strcmp(name, "isis") == 0)
            v = ISO10589_ISIS;
        else if (strcmp(name, "clnp") == 0)
            v = ISO8473_CLNP;
        else
            # 抛出错误
            bpf_error(cstate, "unknown osi proto '%s'", name);
        break;

    default:
        # 默认设置为未定义的协议数值
        v = PROTO_UNDEF;
        break;
    }
    # 返回查找到的协议数值
    return v;
}

# 生成一个协议链的过滤器
#if !defined(NO_PROTOCHAIN)
static struct block *
gen_protochain(compiler_state_t *cstate, bpf_u_int32 v, int proto)
{
    # 定义指向结构体 block 的指针 b0 和 b
    struct block *b0, *b;
    # 定义指向结构体 slist 的指针数组 s，数组大小为 100
    struct slist *s[100];
    # 定义整型变量 fix2, fix3, fix4, fix5, ahcheck, again, end, i, max
    int fix2, fix3, fix4, fix5;
    int ahcheck, again, end;
    int i, max;
    # 为 reg2 分配一个寄存器
    int reg2 = alloc_reg(cstate);

    # 将数组 s 的内容全部初始化为 0
    memset(s, 0, sizeof(s));
    # 将 fix3, fix4, fix5 的值都初始化为 0
    fix3 = fix4 = fix5 = 0;

    # 根据 proto 的值进行不同的操作
    switch (proto) {
    case Q_IP:
    case Q_IPV6:
        # 如果 proto 为 Q_IP 或 Q_IPV6，则直接跳出 switch 语句
        break;
    case Q_DEFAULT:
        # 如果 proto 为 Q_DEFAULT，则生成 Q_IP 和 Q_IPV6 的 protochain，并返回结果
        b0 = gen_protochain(cstate, v, Q_IP);
        b = gen_protochain(cstate, v, Q_IPV6);
        gen_or(b0, b);
        return b;
    default:
        # 如果 proto 不在上述三种情况中，则报错并标记为不可达
        bpf_error(cstate, "bad protocol applied for 'protochain'");
        /*NOTREACHED*/
    }

    # 如果链路层偏移量为可变长度，则报错
    if (cstate->off_linkpl.is_variable)
        bpf_error(cstate, "'protochain' not supported with variable length headers");

    # 关闭优化器
    cstate->no_optimize = 1;

    # 创建一个虚拟的语句 s[0]，用于保护其他 BPF 指令不受未初始化变量 "fix" 的影响
    i = 0;
    s[i] = new_stmt(cstate, 0);    /*dummy*/
    i++;
    # 根据协议类型进行不同的处理
    switch (proto) {
    case Q_IP:
        # 生成以太网类型为 IP 的过滤器
        b0 = gen_linktype(cstate, ETHERTYPE_IP);

        /* A = ip->ip_p */
        # 从绝对位置读取一个字节，存入寄存器 A
        s[i] = new_stmt(cstate, BPF_LD|BPF_ABS|BPF_B);
        s[i]->s.k = cstate->off_linkpl.constant_part + cstate->off_nl + 9;
        i++;
        /* X = ip->ip_hl << 2 */
        # 将寄存器 X 设置为 ip->ip_hl 左移 2 位的值
        s[i] = new_stmt(cstate, BPF_LDX|BPF_MSH|BPF_B);
        s[i]->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
        i++;
        break;

    case Q_IPV6:
        # 生成以太网类型为 IPv6 的过滤器
        b0 = gen_linktype(cstate, ETHERTYPE_IPV6);

        /* A = ip6->ip_nxt */
        # 从绝对位置读取一个字节，存入寄存器 A
        s[i] = new_stmt(cstate, BPF_LD|BPF_ABS|BPF_B);
        s[i]->s.k = cstate->off_linkpl.constant_part + cstate->off_nl + 6;
        i++;
        /* X = sizeof(struct ip6_hdr) */
        # 将寄存器 X 设置为结构体 ip6_hdr 的大小
        s[i] = new_stmt(cstate, BPF_LDX|BPF_IMM);
        s[i]->s.k = 40;
        i++;
        break;

    default:
        # 抛出错误，表示不支持的协议类型
        bpf_error(cstate, "unsupported proto to gen_protochain");
        /*NOTREACHED*/
    }

    /* again: if (A == v) goto end; else fall through; */
    # 记录当前位置，用于后续的跳转指令
    again = i;
    # 如果寄存器 A 的值等于 v，则跳转到 end；否则继续执行下一条指令
    s[i] = new_stmt(cstate, BPF_JMP|BPF_JEQ|BPF_K);
    s[i]->s.k = v;
    s[i]->s.jt = NULL;        /*later*/
    s[i]->s.jf = NULL;        /*update in next stmt*/
    fix5 = i;
    i++;
#ifndef IPPROTO_NONE
#define IPPROTO_NONE    59
#endif
    /* 如果 A 等于 IPPROTO_NONE，则跳转到 end */
    s[i] = new_stmt(cstate, BPF_JMP|BPF_JEQ|BPF_K);
    s[i]->s.jt = NULL;    /*稍后更新*/
    s[i]->s.jf = NULL;    /*在下一个语句中更新*/
    s[i]->s.k = IPPROTO_NONE;
    s[fix5]->s.jf = s[i];
    fix2 = i;
    i++;

    } else {
        /* 空操作 */
        s[i] = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
        s[i]->s.k = 0;
        s[fix2]->s.jf = s[i];
        i++;
    }

    /* ahcheck: */
    ahcheck = i;
    /* 如果 A 等于 IPPROTO_AH，则继续执行；否则跳转到 end */
    s[i] = new_stmt(cstate, BPF_JMP|BPF_JEQ|BPF_K);
    s[i]->s.jt = NULL;    /*稍后更新*/
    s[i]->s.jf = NULL;    /*稍后更新*/
    s[i]->s.k = IPPROTO_AH;
    if (fix3)
        s[fix3]->s.jf = s[ahcheck];
    fix4 = i;
    i++;

    /*
     * 简而言之，
     * A = P[X];
     * X = X + (P[X + 1] + 2) * 4;
     */
    /* A = X */
    s[i - 1]->s.jt = s[i] = new_stmt(cstate, BPF_MISC|BPF_TXA);
    i++;
    /* A = P[X + packet head]; */
    s[i] = new_stmt(cstate, BPF_LD|BPF_IND|BPF_B);
    s[i]->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
    i++;
    /* MEM[reg2] = A */
    s[i] = new_stmt(cstate, BPF_ST);
    s[i]->s.k = reg2;
    i++;
    /* A = X */
    s[i - 1]->s.jt = s[i] = new_stmt(cstate, BPF_MISC|BPF_TXA);
    i++;
    /* A += 1 */
    s[i] = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s[i]->s.k = 1;
    i++;
    /* X = A */
    s[i] = new_stmt(cstate, BPF_MISC|BPF_TAX);
    i++;
    /* A = P[X + packet head] */
    s[i] = new_stmt(cstate, BPF_LD|BPF_IND|BPF_B);
    s[i]->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
    i++;
    /* A += 2 */
    s[i] = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s[i]->s.k = 2;
    i++;
    /* A *= 4 */
    s[i] = new_stmt(cstate, BPF_ALU|BPF_MUL|BPF_K);
    s[i]->s.k = 4;
    i++;
    /* X = A; */
    s[i] = new_stmt(cstate, BPF_MISC|BPF_TAX);
    i++;
    /* A = MEM[reg2] */
    s[i] = new_stmt(cstate, BPF_LD|BPF_MEM);
    # 将寄存器 reg2 的值赋给 s[i]->s.k
    s[i]->s.k = reg2;
    # i 自增
    i++;

    # 使用 BPF_JA 进行回退跳转，跳转到 again 标签处
    s[i] = new_stmt(cstate, BPF_JMP|BPF_JA);
    s[i]->s.k = again - i - 1;
    i++;

    # end: nop
    # 记录 end 的位置
    end = i;
    # 在 s[i] 处创建一个空操作的语句
    s[i] = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s[i]->s.k = 0;
    # 修正 fix2、fix4、fix5 的跳转目标为 end
    s[fix2]->s.jt = s[end];
    s[fix4]->s.jf = s[end];
    s[fix5]->s.jt = s[end];
    i++;

    # 创建 slist 链
    max = i;
    for (i = 0; i < max - 1; i++)
        s[i]->next = s[i + 1];
    s[max - 1]->next = NULL;

    # 生成最终检查
    # 创建一个新的块
    b = new_block(cstate, JMP(BPF_JEQ));
    # 将 s[1] 赋给 b->stmts，注意 s[0] 是虚拟的
    b->stmts = s[1];
    b->s.k = v;

    # 释放寄存器 reg2
    free_reg(cstate, reg2);

    # 生成并连接 b0 和 b
    gen_and(b0, b);
    # 返回块 b
    return b;
    }
#endif /* !defined(NO_PROTOCHAIN) */

// 生成检查 802.11 数据帧的代码
static struct block *
gen_check_802_11_data_frame(compiler_state_t *cstate)
{
    struct slist *s;
    struct block *b0, *b1;

    /*
     * A data frame has the 0x08 bit (b3) in the frame control field set
     * and the 0x04 bit (b2) clear.
     */
    // 生成加载链路头中偏移为0的字节的指令
    s = gen_load_a(cstate, OR_LINKHDR, 0, BPF_B);
    // 创建一个新的块，用于检查指定位是否被设置
    b0 = new_block(cstate, JMP(BPF_JSET));
    b0->s.k = 0x08;
    b0->stmts = s;

    // 生成加载链路头中偏移为0的字节的指令
    s = gen_load_a(cstate, OR_LINKHDR, 0, BPF_B);
    // 创建一个新的块，用于检查指定位是否被清除
    b1 = new_block(cstate, JMP(BPF_JSET));
    b1->s.k = 0x04;
    b1->stmts = s;
    // 生成指令，对b1中的条件取反
    gen_not(b1);

    // 生成指令，对b0和b1中的条件进行与操作
    gen_and(b1, b0);

    // 返回生成的块
    return b0;
}

/*
 * 生成代码，检查数据包是否是协议<proto>的数据包，并检查该协议头中的类型字段是否具有值<v>，
 * 例如，如果<proto>是Q_IP，则检查是否是IP数据包，并检查IP头中的协议号是否与<v>相匹配。
 *
 * 如果<proto>是Q_DEFAULT，即只指定了"proto"，则检查Q_IP和Q_IPV6。
 */
static struct block *
gen_proto(compiler_state_t *cstate, bpf_u_int32 v, int proto, int dir)
{
    struct block *b0, *b1;
    struct block *b2;

    if (dir != Q_DEFAULT)
        // 如果指定了方向，则报错
        bpf_error(cstate, "direction applied to 'proto'");

    switch (proto) {
    case Q_DEFAULT:
        // 生成检查Q_IP和Q_IPV6的代码块
        b0 = gen_proto(cstate, v, Q_IP, dir);
        b1 = gen_proto(cstate, v, Q_IPV6, dir);
        // 生成指令，对b0和b1中的条件进行或操作
        gen_or(b0, b1);
        return b1;

    case Q_LINK:
        // 生成检查链路类型的代码块
        return gen_linktype(cstate, v);
    # 对于 Q_IP 情况，根据不同网络类型的 RFC 规范，使用不同的封装方式，这里总是检查 ETHERTYPE_IP。
    case Q_IP:
        # 生成一个表示 ETHERTYPE_IP 的过滤器
        b0 = gen_linktype(cstate, ETHERTYPE_IP);
        # 生成一个比较操作，比较链路层协议数据的偏移量为9的字节，与给定的值v进行比较
        b1 = gen_cmp(cstate, OR_LINKPL, 9, BPF_B, v);
        # 生成一个与操作，将两个过滤器进行与操作
        gen_and(b0, b1);
        # 返回结果过滤器
        return b1;

    # 对于 Q_ARP 情况，ARP 不封装其他协议，报错
    case Q_ARP:
        bpf_error(cstate, "arp does not encapsulate another protocol");
        /*NOTREACHED*/

    # 对于 Q_RARP 情况，RARP 不封装其他协议，报错
    case Q_RARP:
        bpf_error(cstate, "rarp does not encapsulate another protocol");
        /*NOTREACHED*/

    # 对于 Q_SCTP 情况，SCTP 协议不合法，报错
    case Q_SCTP:
        bpf_error(cstate, "'sctp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_TCP 情况，TCP 协议不合法，报错
    case Q_TCP:
        bpf_error(cstate, "'tcp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_UDP 情况，UDP 协议不合法，报错
    case Q_UDP:
        bpf_error(cstate, "'udp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_ICMP 情况，ICMP 协议不合法，报错
    case Q_ICMP:
        bpf_error(cstate, "'icmp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_IGMP 情况，IGMP 协议不合法，报错
    case Q_IGMP:
        bpf_error(cstate, "'igmp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_IGRP 情况，IGRP 协议不合法，报错
    case Q_IGRP:
        bpf_error(cstate, "'igrp proto' is bogus");
        /*NOTREACHED*/

    # 对于 Q_ATALK 情况，AppleTalk 封装不可指定，报错
    case Q_ATALK:
        bpf_error(cstate, "AppleTalk encapsulation is not specifiable");
        /*NOTREACHED*/

    # 对于 Q_DECNET 情况，DECNET 封装不可指定，报错
    case Q_DECNET:
        bpf_error(cstate, "DECNET encapsulation is not specifiable");
        /*NOTREACHED*/

    # 对于 Q_LAT 情况，LAT 协议不封装其他协议，报错
    case Q_LAT:
        bpf_error(cstate, "LAT does not encapsulate another protocol");
        /*NOTREACHED*/
    # 如果遇到 Q_SCA 情况，报错并且不会执行下面的代码
    case Q_SCA:
        bpf_error(cstate, "SCA does not encapsulate another protocol");
        /*NOTREACHED*/

    # 如果遇到 Q_MOPRC 情况，报错并且不会执行下面的代码
    case Q_MOPRC:
        bpf_error(cstate, "MOPRC does not encapsulate another protocol");
        /*NOTREACHED*/

    # 如果遇到 Q_MOPDL 情况，报错并且不会执行下面的代码
    case Q_MOPDL:
        bpf_error(cstate, "MOPDL does not encapsulate another protocol");
        /*NOTREACHED*/

    # 如果遇到 Q_IPV6 情况
    case Q_IPV6:
        # 生成以太网类型为 ETHERTYPE_IPV6 的指令
        b0 = gen_linktype(cstate, ETHERTYPE_IPV6);
        """
         * 在最终的头部之前也检查片段头部。
         """
        # 生成比较指令，检查链路层协议中是否包含 IPPROTO_FRAGMENT
        b2 = gen_cmp(cstate, OR_LINKPL, 6, BPF_B, IPPROTO_FRAGMENT);
        # 生成比较指令，检查链路层协议中是否包含长度为 40 的头部
        b1 = gen_cmp(cstate, OR_LINKPL, 40, BPF_B, v);
        # 生成与操作指令
        gen_and(b2, b1);
        # 生成比较指令，检查链路层协议中是否包含长度为 6 的头部
        b2 = gen_cmp(cstate, OR_LINKPL, 6, BPF_B, v);
        # 生成或操作指令
        gen_or(b2, b1);
        # 生成与操作指令
        gen_and(b0, b1);
        # 返回结果
        return b1;

    # 如果遇到 Q_ICMPV6 情况，报错并且不会执行下面的代码
    case Q_ICMPV6:
        bpf_error(cstate, "'icmp6 proto' is bogus");
        /*NOTREACHED*/

    # 如果遇到 Q_AH 情况，报错并且不会执行下面的代码
    case Q_AH:
        bpf_error(cstate, "'ah proto' is bogus");
        /*NOTREACHED*/

    # 如果遇到 Q_ESP 情况，报错并且不会执行下面的代码
    case Q_ESP:
        bpf_error(cstate, "'esp proto' is bogus");
        /*NOTREACHED*/

    # 如果遇到 Q_PIM 情况，报错并且不会执行下面的代码
    case Q_PIM:
        bpf_error(cstate, "'pim proto' is bogus");
        /*NOTREACHED*/

    # 如果遇到 Q_VRRP 情况，报错并且不会执行下面的代码
    case Q_VRRP:
        bpf_error(cstate, "'vrrp proto' is bogus");
        /*NOTREACHED*/

    # 如果遇到 Q_AARP 情况，报错并且不会执行下面的代码
    case Q_AARP:
        bpf_error(cstate, "'aarp proto' is bogus");
        /*NOTREACHED*/
    # 如果协议类型是 ISO
    case Q_ISO:
        # 根据链路类型进行不同的处理
        switch (cstate->linktype) {

        # 如果链路类型是 DLT_FRELAY
        case DLT_FRELAY:
            '''
            Frame Relay 数据包通常在开头有一个 OSI NLPID；
            "gen_linktype(cstate, LLCSAP_ISONS)" 生成代码来检查所有 OSI NLPID，
            因此调用它然后添加一个检查我们正在寻找的特定 NLPID 的代码是错误的，
            因为我们可以直接检查 NLPID。
            
            我们要检查的是 NLPID 和一个帧控制字段值为 UI，即 0x03 后跟 NLPID。
            
            XXX - 假设一个 2 字节的 Frame Relay 头部带有 DLCI 和标志。如果地址更长怎么办？
            
            XXX - SNAP 封装的帧怎么处理？
            '''
            return gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, (0x03<<8) | v);
            # 不会执行到这里

        # 如果链路类型是 DLT_C_HDLC 或 DLT_HDLC
        case DLT_C_HDLC:
        case DLT_HDLC:
            '''
            Cisco 使用类似以太网类型的东西 - 对于 OSI，它是 0xfefe。
            '''
            b0 = gen_linktype(cstate, LLCSAP_ISONS<<8 | LLCSAP_ISONS);
            # C-HDLC 中的 OSI 被填充了一个伪字节
            b1 = gen_cmp(cstate, OR_LINKPL_NOSNAP, 1, BPF_B, v);
            gen_and(b0, b1);
            return b1;

        # 其他情况
        default:
            b0 = gen_linktype(cstate, LLCSAP_ISONS);
            b1 = gen_cmp(cstate, OR_LINKPL_NOSNAP, 0, BPF_B, v);
            gen_and(b0, b1);
            return b1;
        }

    # 如果协议类型是 ESIS
    case Q_ESIS:
        bpf_error(cstate, "'esis proto' is bogus");
        # 不会执行到这里

    # 如果协议类型是 ISIS
    case Q_ISIS:
        b0 = gen_proto(cstate, ISO10589_ISIS, Q_ISO, Q_DEFAULT);
        '''
        4 是相对于 IS-IS 头部的 PDU 类型的偏移量。
        '''
        b1 = gen_cmp(cstate, OR_LINKPL_NOSNAP, 4, BPF_B, v);
        gen_and(b0, b1);
        return b1;
    # 如果匹配到 Q_CLNP，抛出错误信息，表示不支持该协议
    case Q_CLNP:
        bpf_error(cstate, "'clnp proto' is not supported");
        /*NOTREACHED*/

    # 如果匹配到 Q_STP，抛出错误信息，表示该协议是虚假的
    case Q_STP:
        bpf_error(cstate, "'stp proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_IPX，抛出错误信息，表示该协议是虚假的
    case Q_IPX:
        bpf_error(cstate, "'ipx proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_NETBEUI，抛出错误信息，表示该协议是虚假的
    case Q_NETBEUI:
        bpf_error(cstate, "'netbeui proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_L1，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_L1:
        bpf_error(cstate, "'l1 proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_L2，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_L2:
        bpf_error(cstate, "'l2 proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_IIH，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_IIH:
        bpf_error(cstate, "'iih proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_SNP，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_SNP:
        bpf_error(cstate, "'snp proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_CSNP，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_CSNP:
        bpf_error(cstate, "'csnp proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_PSNP，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_PSNP:
        bpf_error(cstate, "'psnp proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_ISIS_LSP，抛出错误信息，表示该协议是虚假的
    case Q_ISIS_LSP:
        bpf_error(cstate, "'lsp proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_RADIO，抛出错误信息，表示该协议是虚假的
    case Q_RADIO:
        bpf_error(cstate, "'radio proto' is bogus");
        /*NOTREACHED*/

    # 如果匹配到 Q_CARP，抛出错误信息，表示该协议是虚假的
    case Q_CARP:
        bpf_error(cstate, "'carp proto' is bogus");
        /*NOTREACHED*/

    # 默认情况下，终止程序执行
    default:
        abort();
        /*NOTREACHED*/
    }
    /*NOTREACHED*/
}

// 生成过滤器代码块
struct block *
gen_scode(compiler_state_t *cstate, const char *name, struct qual q)
{
    int proto = q.proto;  // 从结构体中获取协议类型
    int dir = q.dir;  // 从结构体中获取方向
    int tproto;  // 临时存储协议类型
    u_char *eaddr;  // 存储地址
    bpf_u_int32 mask, addr;  // 存储掩码和地址
    struct addrinfo *res, *res0;  // 地址信息结构体指针
    struct sockaddr_in *sin4;  // IPv4地址结构体指针
#ifdef INET6
    int tproto6;  // IPv6协议类型
    struct sockaddr_in6 *sin6;  // IPv6地址结构体指针
    struct in6_addr mask128;  // IPv6掩码
#endif /*INET6*/
    struct block *b, *tmp;  // 代码块指针
    int port, real_proto;  // 端口和真实协议类型
    int port1, port2;  // 端口1和端口2

    /*
     * 捕获我们报告的错误和下面的例程报告的错误，并在出错时返回 NULL
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点
        return (NULL);  // 出错时返回 NULL

    switch (q.addr) {  // 根据地址类型进行判断

    case Q_NET:  // 如果是网络地址
        addr = pcap_nametonetaddr(name);  // 将名称转换为网络地址
        if (addr == 0)  // 如果地址为0
            bpf_error(cstate, "unknown network '%s'", name);  // 报告未知网络错误
        /* 左对齐网络地址并计算其网络掩码 */
        mask = 0xffffffff;  // 初始化掩码
        while (addr && (addr & 0xff000000) == 0) {  // 左对齐网络地址并计算掩码
            addr <<= 8;
            mask <<= 8;
        }
        return gen_host(cstate, addr, mask, proto, dir, q.addr);  // 生成主机过滤器代码块

    case Q_DEFAULT:  // 如果是默认地址
#ifdef    DECNETLIB
                bpf_error(cstate, "unknown decnet host name '%s'\n", name);  // 报告未知 DECNET 主机名错误
#else
                bpf_error(cstate, "decnet name support not included, '%s' cannot be translated\n",
                    name);  // 报告 DECNET 名称支持未包含错误
#endif
            }
            /*
             * 我认为 DECNET 主机不能是多宿主的，所以不需要构建地址列表
             */
            return (gen_host(cstate, dn_addr, 0, proto, dir, q.addr));  // 生成主机过滤器代码块
        } else {
#ifdef INET6
            memset(&mask128, 0xff, sizeof(mask128));  // 初始化 IPv6 掩码
#endif
            res0 = res = pcap_nametoaddrinfo(name);  // 将名称转换为地址信息
            if (res == NULL)  // 如果地址信息为空
                bpf_error(cstate, "unknown host '%s'", name);  // 报告未知主机错误
            cstate->ai = res;  // 存储地址信息
            b = tmp = NULL;  // 初始化代码块指针
            tproto = proto;  // 临时存储协议类型
#ifdef INET6
            tproto6 = proto;  // 临时存储 IPv6 协议类型
#endif
            // 如果偏移量未设置且传输协议为默认值，则设置传输协议为 IPv4
            if (cstate->off_linktype.constant_part == OFFSET_NOT_SET &&
                tproto == Q_DEFAULT) {
                tproto = Q_IP;
#ifdef INET6
                // 如果支持 IPv6，则设置传输协议为 IPv6
                tproto6 = Q_IPV6;
#endif
            }
            // 遍历地址信息链表
            for (res = res0; res; res = res->ai_next) {
                // 根据地址族进行不同的处理
                switch (res->ai_family) {
                case AF_INET:
#ifdef INET6
                    // 如果支持 IPv6 并且传输协议为 IPv6，则跳过当前循环
                    if (tproto == Q_IPV6)
                        continue;
#endif
                    // 获取 IPv4 地址结构体指针
                    sin4 = (struct sockaddr_in *)
                        res->ai_addr;
                    // 生成 IPv4 主机地址过滤器
                    tmp = gen_host(cstate, ntohl(sin4->sin_addr.s_addr),
                        0xffffffff, tproto, dir, q.addr);
                    break;
#ifdef INET6
                case AF_INET6:
                    // 如果支持 IPv6 并且传输协议为 IPv4，则跳过当前循环
                    if (tproto6 == Q_IP)
                        continue;
                    // 获取 IPv6 地址结构体指针
                    sin6 = (struct sockaddr_in6 *)
                        res->ai_addr;
                    // 生成 IPv6 主机地址过滤器
                    tmp = gen_host6(cstate, &sin6->sin6_addr,
                        &mask128, tproto6, dir, q.addr);
                    break;
#endif
                default:
                    // 如果地址族不是 IPv4 或 IPv6，则继续下一次循环
                    continue;
                }
                // 如果 b 不为空，则将 tmp 与 b 进行逻辑或操作
                if (b)
                    gen_or(b, tmp);
                // 将 tmp 赋值给 b
                b = tmp;
            }
            // 重置地址信息指针
            cstate->ai = NULL;
            // 释放地址信息链表
            freeaddrinfo(res0);
            // 如果 b 为空，则抛出错误信息
            if (b == NULL) {
                bpf_error(cstate, "unknown host '%s'%s", name,
                    (proto == Q_DEFAULT)
                    ? ""
                    : " for specified address family");
            }
            // 返回过滤器
            return b;
        }
    # 如果匹配到端口关键字
    case Q_PORT:
        # 如果协议不是默认值，也不是UDP、TCP、SCTP中的一个，报错
        if (proto != Q_DEFAULT &&
            proto != Q_UDP && proto != Q_TCP && proto != Q_SCTP)
            bpf_error(cstate, "illegal qualifier of 'port'");
        # 将端口名转换为端口号和真实协议类型
        if (pcap_nametoport(name, &port, &real_proto) == 0)
            bpf_error(cstate, "unknown port '%s'", name);
        # 如果协议是UDP
        if (proto == Q_UDP) {
            # 如果真实协议是TCP，报错
            if (real_proto == IPPROTO_TCP)
                bpf_error(cstate, "port '%s' is tcp", name);
            # 如果真实协议是SCTP，报错
            else if (real_proto == IPPROTO_SCTP)
                bpf_error(cstate, "port '%s' is sctp", name);
            # 否则，覆盖PROTO_UNDEF
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_UDP;
        }
        # 如果协议是TCP
        if (proto == Q_TCP) {
            # 如果真实协议是UDP，报错
            if (real_proto == IPPROTO_UDP)
                bpf_error(cstate, "port '%s' is udp", name);
            # 如果真实协议是SCTP，报错
            else if (real_proto == IPPROTO_SCTP)
                bpf_error(cstate, "port '%s' is sctp", name);
            # 否则，覆盖PROTO_UNDEF
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_TCP;
        }
        # 如果协议是SCTP
        if (proto == Q_SCTP) {
            # 如果真实协议是UDP，报错
            if (real_proto == IPPROTO_UDP)
                bpf_error(cstate, "port '%s' is udp", name);
            # 如果真实协议是TCP，报错
            else if (real_proto == IPPROTO_TCP)
                bpf_error(cstate, "port '%s' is tcp", name);
            # 否则，覆盖PROTO_UNDEF
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_SCTP;
        }
        # 如果端口号小于0，报错
        if (port < 0)
            bpf_error(cstate, "illegal port number %d < 0", port);
        # 如果端口号大于65535，报错
        if (port > 65535)
            bpf_error(cstate, "illegal port number %d > 65535", port);
        # 生成端口过滤器
        b = gen_port(cstate, port, real_proto, dir);
        # 生成IPv6的端口过滤器，并与之前的结果进行或运算
        gen_or(gen_port6(cstate, port, real_proto, dir), b);
        # 返回结果
        return b;
    # 如果指定了协议，并且不是默认、UDP、TCP、SCTP之一，则报错
    case Q_PORTRANGE:
        if (proto != Q_DEFAULT &&
            proto != Q_UDP && proto != Q_TCP && proto != Q_SCTP)
            bpf_error(cstate, "illegal qualifier of 'portrange'");
        # 将名称转换为端口范围，如果失败则报错
        if (pcap_nametoportrange(name, &port1, &port2, &real_proto) == 0)
            bpf_error(cstate, "unknown port in range '%s'", name);
        # 如果指定了UDP协议，但实际协议是TCP或SCTP，则报错
        if (proto == Q_UDP) {
            if (real_proto == IPPROTO_TCP)
                bpf_error(cstate, "port in range '%s' is tcp", name);
            else if (real_proto == IPPROTO_SCTP)
                bpf_error(cstate, "port in range '%s' is sctp", name);
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_UDP;
        }
        # 如果指定了TCP协议，但实际协议是UDP或SCTP，则报错
        if (proto == Q_TCP) {
            if (real_proto == IPPROTO_UDP)
                bpf_error(cstate, "port in range '%s' is udp", name);
            else if (real_proto == IPPROTO_SCTP)
                bpf_error(cstate, "port in range '%s' is sctp", name);
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_TCP;
        }
        # 如果指定了SCTP协议，但实际协议是UDP或TCP，则报错
        if (proto == Q_SCTP) {
            if (real_proto == IPPROTO_UDP)
                bpf_error(cstate, "port in range '%s' is udp", name);
            else if (real_proto == IPPROTO_TCP)
                bpf_error(cstate, "port in range '%s' is tcp", name);
            else
                /* override PROTO_UNDEF */
                real_proto = IPPROTO_SCTP;
        }
        # 如果端口号小于0，则报错
        if (port1 < 0)
            bpf_error(cstate, "illegal port number %d < 0", port1);
        # 如果端口号大于65535，则报错
        if (port1 > 65535)
            bpf_error(cstate, "illegal port number %d > 65535", port1);
        # 如果端口号小于0，则报错
        if (port2 < 0)
            bpf_error(cstate, "illegal port number %d < 0", port2);
        # 如果端口号大于65535，则报错
        if (port2 > 65535)
            bpf_error(cstate, "illegal port number %d > 65535", port2);

        # 生成端口范围的BPF代码
        b = gen_portrange(cstate, port1, port2, real_proto, dir);
        # 生成IPv6的端口范围的BPF代码，并与之前生成的BPF代码进行或运算
        gen_or(gen_portrange6(cstate, port1, port2, real_proto, dir), b);
        # 返回生成的BPF代码
        return b;
    # 如果条件为 Q_GATEWAY，则执行以下代码块
#ifndef INET6
        // 如果不支持 IPv6，则根据名称获取以太网地址
        eaddr = pcap_ether_hostton(name);
        // 如果获取失败，则报错
        if (eaddr == NULL)
            bpf_error(cstate, "unknown ether host: %s", name);

        // 根据名称获取地址信息
        res = pcap_nametoaddrinfo(name);
        // 将地址信息保存到 cstate->ai 中
        cstate->ai = res;
        // 如果获取失败，则报错
        if (res == NULL)
            bpf_error(cstate, "unknown host '%s'", name);
        // 生成网关代码
        b = gen_gateway(cstate, eaddr, res, proto, dir);
        // 清空 cstate->ai
        cstate->ai = NULL;
        // 释放地址信息
        freeaddrinfo(res);
        // 如果生成失败，则报错
        if (b == NULL)
            bpf_error(cstate, "unknown host '%s'", name);
        // 返回生成的代码块
        return b;
#else
        // 如果支持 IPv6，则报错
        bpf_error(cstate, "'gateway' not supported in this configuration");
#endif /*INET6*/

    case Q_PROTO:
        // 查找协议名称对应的实际协议值
        real_proto = lookup_proto(cstate, name, proto);
        // 如果找到，则生成协议代码块
        if (real_proto >= 0)
            return gen_proto(cstate, real_proto, proto, dir);
        // 如果未找到，则报错
        else
            bpf_error(cstate, "unknown protocol: %s", name);

#if !defined(NO_PROTOCHAIN)
    case Q_PROTOCHAIN:
        // 查找协议名称对应的实际协议值
        real_proto = lookup_proto(cstate, name, proto);
        // 如果找到，则生成协议链代码块
        if (real_proto >= 0)
            return gen_protochain(cstate, real_proto, proto);
        // 如果未找到，则报错
        else
            bpf_error(cstate, "unknown protocol: %s", name);
#endif /* !defined(NO_PROTOCHAIN) */

    case Q_UNDEF:
        // 语法错误，报错
        syntax(cstate);
        /*NOTREACHED*/
    }
    // 中止程序
    abort();
    /*NOTREACHED*/
}

struct block *
gen_mcode(compiler_state_t *cstate, const char *s1, const char *s2,
    bpf_u_int32 masklen, struct qual q)
{
    register int nlen, mlen;
    bpf_u_int32 n, m;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 捕获我们和下面程序报告的错误，并在出错时返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    // 将字符串转换为 IPv4 地址
    nlen = __pcap_atoin(s1, &n);
    // 如果转换失败，则报错
    if (nlen < 0)
        bpf_error(cstate, "invalid IPv4 address '%s'", s1);
    /* Promote short ipaddr */
    // 将短的 IP 地址扩展为完整的 IP 地址
    n <<= 32 - nlen;
    # 如果子网掩码不为空
    if (s2 != NULL) {
        # 将字符串形式的子网掩码转换为整数形式，并返回子网掩码的长度
        mlen = __pcap_atoin(s2, &m);
        # 如果子网掩码长度小于0，则表示子网掩码无效，抛出错误
        if (mlen < 0)
            bpf_error(cstate, "invalid IPv4 address '%s'", s2);
        # 将 IP 地址根据子网掩码长度进行位移，用于将短的 IP 地址转换为完整的 IP 地址
        m <<= 32 - mlen;
        # 如果 IP 地址与取反后的子网掩码不等于0，则表示存在非网络位，抛出错误
        if ((n & ~m) != 0)
            bpf_error(cstate, "non-network bits set in \"%s mask %s\"",
                s1, s2);
    } else {
        # 如果子网掩码长度大于32，则抛出错误
        if (masklen > 32)
            bpf_error(cstate, "mask length must be <= 32");
        # 如果子网掩码长度为0，则将 m 置为0
        if (masklen == 0) {
            /*
             * X << 32 is not guaranteed by C to be 0; it's
             * undefined.
             */
            m = 0;
        } else
            # 否则根据子网掩码长度生成对应的子网掩码
            m = 0xffffffff << (32 - masklen);
        # 如果 IP 地址与取反后的子网掩码不等于0，则表示存在非网络位，抛出错误
        if ((n & ~m) != 0)
            bpf_error(cstate, "non-network bits set in \"%s/%d\"",
                s1, masklen);
    }

    # 根据地址类型进行不同的处理
    switch (q.addr) {

    case Q_NET:
        # 如果地址类型为网络地址，则调用 gen_host 函数生成对应的主机地址
        return gen_host(cstate, n, m, q.proto, q.dir, q.addr);

    default:
        # 如果地址类型不是网络地址，则抛出错误
        bpf_error(cstate, "Mask syntax for networks only");
        /*NOTREACHED*/
    }
    /*NOTREACHED*/
    }
    // 生成网络代码块
    struct block *gen_ncode(compiler_state_t *cstate, const char *s, bpf_u_int32 v, struct qual q)
    {
        bpf_u_int32 mask;
        int proto;
        int dir;
        register int vlen;

        /*
         * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
         */
        if (setjmp(cstate->top_ctx))
            return (NULL);

        proto = q.proto;
        dir = q.dir;
        if (s == NULL)
            vlen = 32;
        else if (q.proto == Q_DECNET) {
            vlen = __pcap_atodn(s, &v);
            if (vlen == 0)
                bpf_error(cstate, "malformed decnet address '%s'", s);
        } else {
            vlen = __pcap_atoin(s, &v);
            if (vlen < 0)
                bpf_error(cstate, "invalid IPv4 address '%s'", s);
        }

        switch (q.addr) {

        case Q_DEFAULT:
        case Q_HOST:
        case Q_NET:
            if (proto == Q_DECNET)
                return gen_host(cstate, v, 0, proto, dir, q.addr);
            else if (proto == Q_LINK) {
                bpf_error(cstate, "illegal link layer address");
            } else {
                mask = 0xffffffff;
                if (s == NULL && q.addr == Q_NET) {
                    /* 提升短网络号 */
                    while (v && (v & 0xff000000) == 0) {
                        v <<= 8;
                        mask <<= 8;
                    }
                } else {
                    /* 提升短 ip 地址 */
                    v <<= 32 - vlen;
                    mask <<= 32 - vlen ;
                }
                return gen_host(cstate, v, mask, proto, dir, q.addr);
            }
    # 如果是端口限定词，根据协议类型设置对应的协议号
    case Q_PORT:
        if (proto == Q_UDP)
            proto = IPPROTO_UDP;
        else if (proto == Q_TCP)
            proto = IPPROTO_TCP;
        else if (proto == Q_SCTP)
            proto = IPPROTO_SCTP;
        else if (proto == Q_DEFAULT)
            proto = PROTO_UNDEF;
        else
            bpf_error(cstate, "illegal qualifier of 'port'");

        # 如果端口号大于65535，报错
        if (v > 65535)
            bpf_error(cstate, "illegal port number %u > 65535", v);

        {
        # 生成端口过滤器
        struct block *b;
        b = gen_port(cstate, v, proto, dir);
        # 生成 IPv6 的端口过滤器，并与之前生成的过滤器进行或操作
        gen_or(gen_port6(cstate, v, proto, dir), b);
        # 返回生成的过滤器
        return b;
        }

    # 如果是端口范围限定词，根据协议类型设置对应的协议号
    case Q_PORTRANGE:
        if (proto == Q_UDP)
            proto = IPPROTO_UDP;
        else if (proto == Q_TCP)
            proto = IPPROTO_TCP;
        else if (proto == Q_SCTP)
            proto = IPPROTO_SCTP;
        else if (proto == Q_DEFAULT)
            proto = PROTO_UNDEF;
        else
            bpf_error(cstate, "illegal qualifier of 'portrange'");

        # 如果端口号大于65535，报错
        if (v > 65535)
            bpf_error(cstate, "illegal port number %u > 65535", v);

        {
        # 生成端口范围过滤器
        struct block *b;
        b = gen_portrange(cstate, v, v, proto, dir);
        # 生成 IPv6 的端口范围过滤器，并与之前生成的过滤器进行或操作
        gen_or(gen_portrange6(cstate, v, v, proto, dir), b);
        # 返回生成的过滤器
        return b;
        }

    # 如果是'gateway'限定词，报错，需要提供名称
    case Q_GATEWAY:
        bpf_error(cstate, "'gateway' requires a name");
        /*NOTREACHED*/

    # 如果是协议限定词，生成对应的协议过滤器
    case Q_PROTO:
        return gen_proto(cstate, v, proto, dir);
#if !defined(NO_PROTOCHAIN)
    # 如果未定义 NO_PROTOCHAIN，则执行以下代码
    case Q_PROTOCHAIN:
        # 对于 Q_PROTOCHAIN 情况，生成协议链
        return gen_protochain(cstate, v, proto);
#endif

    case Q_UNDEF:
        # 对于 Q_UNDEF 情况，调用 syntax 函数
        syntax(cstate);
        /*NOTREACHED*/

    default:
        # 默认情况下，调用 abort 函数
        abort();
        /*NOTREACHED*/
    }
    /*NOTREACHED*/
}

#ifdef INET6
struct block *
gen_mcode6(compiler_state_t *cstate, const char *s1, const char *s2,
    bpf_u_int32 masklen, struct qual q)
{
    struct addrinfo *res;
    struct in6_addr *addr;
    struct in6_addr mask;
    struct block *b;
    uint32_t *a, *m;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    # 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (s2)
        # 如果 s2 存在，则报错，不支持带掩码的情况
        bpf_error(cstate, "no mask %s supported", s2);

    res = pcap_nametoaddrinfo(s1);
    if (!res)
        # 如果解析失败，则报错
        bpf_error(cstate, "invalid ip6 address %s", s1);
    cstate->ai = res;
    if (res->ai_next)
        # 如果解析结果包含多个地址，则报错
        bpf_error(cstate, "%s resolved to multiple address", s1);
    addr = &((struct sockaddr_in6 *)res->ai_addr)->sin6_addr;

    if (masklen > sizeof(mask.s6_addr) * 8)
        # 如果掩码长度超过地址长度，则报错
        bpf_error(cstate, "mask length must be <= %u", (unsigned int)(sizeof(mask.s6_addr) * 8));
    memset(&mask, 0, sizeof(mask));
    memset(&mask.s6_addr, 0xff, masklen / 8);
    if (masklen % 8) {
        mask.s6_addr[masklen / 8] =
            (0xff << (8 - masklen % 8)) & 0xff;
    }

    a = (uint32_t *)addr;
    m = (uint32_t *)&mask;
    if ((a[0] & ~m[0]) || (a[1] & ~m[1])
     || (a[2] & ~m[2]) || (a[3] & ~m[3])) {
        # 如果地址和掩码不匹配，则报错
        bpf_error(cstate, "non-network bits set in \"%s/%d\"", s1, masklen);
    }

    switch (q.addr) {

    case Q_DEFAULT:
    case Q_HOST:
        if (masklen != 128)
            # 如果掩码长度不为 128，则报错
            bpf_error(cstate, "Mask syntax for networks only");
        /* FALLTHROUGH */

    case Q_NET:
        # 对于 Q_DEFAULT、Q_HOST、Q_NET 情况，生成 IPv6 地址匹配规则
        b = gen_host6(cstate, addr, &mask, q.proto, q.dir, q.addr);
        cstate->ai = NULL;
        freeaddrinfo(res);
        return b;
    # 如果没有匹配的情况，报告错误并提示对 IPv6 地址使用了无效的限定词
    default:
        bpf_error(cstate, "invalid qualifier against IPv6 address");
        /*NOTREACHED*/
    }
}
#endif /*INET6*/

struct block *
gen_ecode(compiler_state_t *cstate, const char *s, struct qual q)
{
    struct block *b, *tmp;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 设置错误跳转点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if ((q.addr == Q_HOST || q.addr == Q_DEFAULT) && q.proto == Q_LINK) {
        // 将字符串形式的以太网地址转换为二进制形式
        cstate->e = pcap_ether_aton(s);
        if (cstate->e == NULL)
            // 如果转换失败，报错并返回
            bpf_error(cstate, "malloc");
        switch (cstate->linktype) {
        case DLT_EN10MB:
        case DLT_NETANALYZER:
        case DLT_NETANALYZER_TRANSPARENT:
            // 生成以太网头部检查代码块
            tmp = gen_prevlinkhdr_check(cstate);
            // 生成以太网目的地址匹配代码块
            b = gen_ehostop(cstate, cstate->e, (int)q.dir);
            if (tmp != NULL)
                // 生成并连接两个代码块
                gen_and(tmp, b);
            break;
        case DLT_FDDI:
            // 生成 FDDI 目的地址匹配代码块
            b = gen_fhostop(cstate, cstate->e, (int)q.dir);
            break;
        case DLT_IEEE802:
            // 生成 IEEE 802 目的地址匹配代码块
            b = gen_thostop(cstate, cstate->e, (int)q.dir);
            break;
        case DLT_IEEE802_11:
        case DLT_PRISM_HEADER:
        case DLT_IEEE802_11_RADIO_AVS:
        case DLT_IEEE802_11_RADIO:
        case DLT_PPI:
            // 生成 WLAN 目的地址匹配代码块
            b = gen_wlanhostop(cstate, cstate->e, (int)q.dir);
            break;
        case DLT_IP_OVER_FC:
            // 生成 IP over FC 目的地址匹配代码块
            b = gen_ipfchostop(cstate, cstate->e, (int)q.dir);
            break;
        default:
            // 释放以太网地址内存，报错并返回
            free(cstate->e);
            cstate->e = NULL;
            bpf_error(cstate, "ethernet addresses supported only on ethernet/FDDI/token ring/802.11/ATM LANE/Fibre Channel");
            /*NOTREACHED*/
        }
        // 释放以太网地址内存，返回生成的代码块
        free(cstate->e);
        cstate->e = NULL;
        return (b);
    }
    // 如果地址类型不是以太网，报错并返回
    bpf_error(cstate, "ethernet address used in non-ether expression");
    /*NOTREACHED*/
}

void
sappend(struct slist *s0, struct slist *s1)
{
    /*
     * This is definitely not the best way to do this, but the
     * lists will rarely get long.
     */
    // 将 s1 链接到 s0 的末尾
    while (s0->next)
        s0 = s0->next;
    s0->next = s1;
}
# 定义一个函数，将传入的算术操作转换为加载寄存器指令，并返回生成的指令链表
static struct slist *
xfer_to_x(compiler_state_t *cstate, struct arth *a)
{
    # 声明一个指向结构体 slist 的指针 s
    struct slist *s;
    # 调用函数 new_stmt 创建一个新的加载寄存器指令，并将其赋值给 s
    s = new_stmt(cstate, BPF_LDX|BPF_MEM);
    # 设置 s 的 k 属性为 a 的寄存器编号
    s->s.k = a->regno;
    # 返回指令链表 s
    return s;
}

# 定义一个函数，将传入的算术操作转换为加载数据到寄存器指令，并返回生成的指令链表
static struct slist *
xfer_to_a(compiler_state_t *cstate, struct arth *a)
{
    # 声明一个指向结构体 slist 的指针 s
    struct slist *s;
    # 调用函数 new_stmt 创建一个新的加载数据到寄存器指令，并将其赋值给 s
    s = new_stmt(cstate, BPF_LD|BPF_MEM);
    # 设置 s 的 k 属性为 a 的寄存器编号
    s->s.k = a->regno;
    # 返回指令链表 s
    return s;
}

# 定义一个函数，生成加载数据到寄存器的内部操作，并返回生成的算术操作结构体
static struct arth *
gen_load_internal(compiler_state_t *cstate, int proto, struct arth *inst, bpf_u_int32 size)
{
    # 声明变量
    int size_code;
    struct slist *s, *tmp;
    struct block *b;
    int regno = alloc_reg(cstate);
    # 释放 inst 寄存器的使用
    free_reg(cstate, inst->regno);
    # 根据数据大小选择相应的 BPF 指令
    switch (size) {
        default:
            bpf_error(cstate, "data size must be 1, 2, or 4");
            /*NOTREACHED*/
        case 1:
            size_code = BPF_B;
            break;
        case 2:
            size_code = BPF_H;
            break;
        case 4:
            size_code = BPF_W;
            break;
    }
    # 根据协议选择相应的操作
    switch (proto) {
        default:
            bpf_error(cstate, "unsupported index operation");
    # 如果数据包中包含无线电头部信息，则偏移量是相对于数据包开头的
    # 如果没有无线电头部信息，则会出现错误
    if (cstate->linktype != DLT_IEEE802_11_RADIO_AVS &&
        cstate->linktype != DLT_IEEE802_11_RADIO &&
        cstate->linktype != DLT_PRISM_HEADER)
        bpf_error(cstate, "radio information not present in capture");

    # 将计算得到的偏移量加载到 X 寄存器中
    s = xfer_to_x(cstate, inst);

    # 加载该偏移量处的项目
    tmp = new_stmt(cstate, BPF_LD|BPF_IND|size_code);
    sappend(s, tmp);
    sappend(inst->s, s);
    break;
    # 如果是 Q_LINK 情况
    case Q_LINK:
        '''
         * 偏移量是相对于链路层头部的开头。
         *
         * XXX - ATM LANE 怎么办？索引是相对于 AAL5 帧的开头，所以
         * 0 是指 LE 控制字段的开头，还是相对于 LAN 帧的开头，所以
         * 0 是指目的地址的开头（对于以太网 LANE）？
         '''
        # 生成绝对偏移变量部分
        s = gen_abs_offset_varpart(cstate, &cstate->off_linkhdr);

        '''
         * 如果 "s" 非空，它包含代码来确保 X 寄存器包含链路层头部之前的前缀的长度。
         * 将计算出的偏移量添加到由 "index" 指定的寄存器中，并将其移动到 X 寄存器中。
         * 否则，只需将计算出的偏移量加载到由 "index" 指定的寄存器中的 X 寄存器中。
         '''
        if (s != NULL) {
            sappend(s, xfer_to_a(cstate, inst));
            sappend(s, new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X));
            sappend(s, new_stmt(cstate, BPF_MISC|BPF_TAX));
        } else
            s = xfer_to_x(cstate, inst);

        '''
         * 加载位于我们放入 X 寄存器的偏移量和链路层头部开始的偏移量之和的项目
         * （如果无线电头部是可变长度的，则该头部长度是我们放入 X 寄存器然后添加到索引的值）。
         '''
        tmp = new_stmt(cstate, BPF_LD|BPF_IND|size_code);
        tmp->s.k = cstate->off_linkhdr.constant_part;
        sappend(s, tmp);
        sappend(inst->s, s);
        break;

    # 其他情况
    case Q_IP:
    case Q_ARP:
    case Q_RARP:
    case Q_ATALK:
    case Q_DECNET:
    case Q_SCA:
    case Q_LAT:
    case Q_MOPRC:
    case Q_MOPDL:
    # 如果协议类型是 IPv6
    case Q_IPV6:
        '''
         * 偏移量是相对于网络层头部的开始位置。
         * XXX - 是否有任何情况下我们想要使用 cstate->off_nl_nosnap？
         '''
        # 生成绝对偏移变量部分
        s = gen_abs_offset_varpart(cstate, &cstate->off_linkpl);

        '''
         * 如果 "s" 非空，它包含代码以确保 X 寄存器包含链路层有效载荷的变量部分的偏移量。
         * 将其与由 "index" 指定的寄存器中计算的偏移量相加，并将其移动到 X 寄存器中。
         * 否则，只需将计算出的偏移量加载到 X 寄存器中。
         '''
        if (s != NULL) {
            sappend(s, xfer_to_a(cstate, inst));
            sappend(s, new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X));
            sappend(s, new_stmt(cstate, BPF_MISC|BPF_TAX));
        } else
            s = xfer_to_x(cstate, inst);

        '''
         * 加载位于我们放入 X 寄存器的偏移量、网络层头部开始位置的偏移量和链路层有效载荷开始位置的常量部分的和处的项目。
         '''
        tmp = new_stmt(cstate, BPF_LD|BPF_IND|size_code);
        tmp->s.k = cstate->off_linkpl.constant_part + cstate->off_nl;
        sappend(s, tmp);
        sappend(inst->s, s);

        '''
         * 只有在数据包包含所讨论的协议时才进行计算。
         '''
        b = gen_proto_abbrev_internal(cstate, proto);
        if (inst->b)
            gen_and(inst->b, b);
        inst->b = b;
        break;

    # 如果协议类型是 SCTP、TCP、UDP、ICMP、IGMP、IGRP、PIM、VRRP
    case Q_SCTP:
    case Q_TCP:
    case Q_UDP:
    case Q_ICMP:
    case Q_IGMP:
    case Q_IGRP:
    case Q_PIM:
    case Q_VRRP:
    # 如果协议是 ICMPV6，则执行以下操作
    case Q_ICMPV6:
        '''
        * 只有当数据包包含所需的协议时才进行计算。
        '''
        # 生成 IPv6 协议的缩写
        b = gen_proto_abbrev_internal(cstate, Q_IPV6);
        # 如果已经有 b，则执行与操作
        if (inst->b) {
            gen_and(inst->b, b);
        }
        # 将 b 赋值给 inst->b
        inst->b = b;

        '''
        * 检查是否有 icmp6 下一个标头
        '''
        # 生成比较操作，检查是否有 icmp6 下一个标头
        b = gen_cmp(cstate, OR_LINKPL, 6, BPF_B, 58);
        # 如果已经有 b，则执行与操作
        if (inst->b) {
            gen_and(inst->b, b);
        }
        # 将 b 赋值给 inst->b
        inst->b = b;

        # 生成绝对偏移变量部分
        s = gen_abs_offset_varpart(cstate, &cstate->off_linkpl);
        '''
        * 如果 "s" 非空，则它包含代码，安排 X 寄存器包含偏移的变量部分
        * 通过将它添加到由 "index" 指定的寄存器计算的偏移，并将其移动到 X 寄存器中
        * 否则，只需将由 "index" 指定的寄存器计算的偏移加载到 X 寄存器中
        '''
        # 如果 "s" 非空，则执行以下操作
        if (s != NULL) {
            sappend(s, xfer_to_a(cstate, inst));
            sappend(s, new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X));
            sappend(s, new_stmt(cstate, BPF_MISC|BPF_TAX));
        } else {
            # 将由 "index" 指定的寄存器计算的偏移加载到 X 寄存器中
            s = xfer_to_x(cstate, inst);
        }

        '''
        * 加载位于我们放入 X 寄存器的偏移量、网络层头部开始偏移量和链路层有效载荷开始偏移量的和处的项目
        '''
        # 生成加载指令，加载偏移量的项目
        tmp = new_stmt(cstate, BPF_LD|BPF_IND|size_code);
        tmp->s.k = cstate->off_linkpl.constant_part + cstate->off_nl + 40;

        # 将 tmp 添加到 s 中
        sappend(s, tmp);
        # 将 s 添加到 inst->s 中
        sappend(inst->s, s);

        # 跳出循环
        break;
    }
    # 将 regno 赋值给 inst->regno
    inst->regno = regno;
    # 生成存储指令
    s = new_stmt(cstate, BPF_ST);
    s->s.k = regno;
    # 将 s 添加到 inst->s 中
    sappend(inst->s, s);

    # 返回 inst
    return inst;
}

struct arth *
gen_load(compiler_state_t *cstate, int proto, struct arth *inst,
    bpf_u_int32 size)
{
    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点，捕获上层和当前层报告的错误
        return (NULL);  // 如果有错误发生，返回空指针

    return gen_load_internal(cstate, proto, inst, size);  // 调用内部函数处理加载操作
}

static struct block *
gen_relation_internal(compiler_state_t *cstate, int code, struct arth *a0,
    struct arth *a1, int reversed)
{
    struct slist *s0, *s1, *s2;
    struct block *b, *tmp;

    s0 = xfer_to_x(cstate, a1);  // 将 a1 转换为 X 寄存器
    s1 = xfer_to_a(cstate, a0);  // 将 a0 转换为 A 寄存器
    if (code == BPF_JEQ) {  // 如果条件码为等于
        s2 = new_stmt(cstate, BPF_ALU|BPF_SUB|BPF_X);  // 创建一个新的语句
        b = new_block(cstate, JMP(code));  // 创建一个新的块
        sappend(s1, s2);  // 将 s2 追加到 s1 后面
    }
    else
        b = new_block(cstate, BPF_JMP|code|BPF_X);  // 创建一个新的块
    if (reversed)
        gen_not(b);  // 生成取反操作

    sappend(s0, s1);  // 将 s1 追加到 s0 后面
    sappend(a1->s, s0);  // 将 s0 追加到 a1 的 s 列表后面
    sappend(a0->s, a1->s);  // 将 a1 的 s 列表追加到 a0 的 s 列表后面

    b->stmts = a0->s;  // 将 a0 的 s 列表赋值给 b 的 stmts

    free_reg(cstate, a0->regno);  // 释放 a0 的寄存器
    free_reg(cstate, a1->regno);  // 释放 a1 的寄存器

    /* 'and' together protocol checks */
    if (a0->b) {
        if (a1->b) {
            gen_and(a0->b, tmp = a1->b);  // 生成并操作，并将结果赋值给 tmp
        }
        else
            tmp = a0->b;  // 如果 a1 没有 b，则将 a0 的 b 赋值给 tmp
    } else
        tmp = a1->b;  // 如果 a0 没有 b，则将 a1 的 b 赋值给 tmp

    if (tmp)
        gen_and(tmp, b);  // 如果 tmp 存在，则生成并操作，并将结果赋值给 b

    return b;  // 返回块
}

struct block *
gen_relation(compiler_state_t *cstate, int code, struct arth *a0,
    struct arth *a1, int reversed)
{
    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点，捕获上层和当前层报告的错误
        return (NULL);  // 如果有错误发生，返回空指针

    return gen_relation_internal(cstate, code, a0, a1, reversed);  // 调用内部函数处理关系操作
}

struct arth *
gen_loadlen(compiler_state_t *cstate)
{
    int regno;
    struct arth *a;
    struct slist *s;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点，捕获上层和当前层报告的错误
        return (NULL);  // 如果有错误发生，返回空指针

    regno = alloc_reg(cstate);  // 分配寄存器
    a = (struct arth *)newchunk(cstate, sizeof(*a));  // 分配内存
    # 创建一个新的BPF_LD|BPF_LEN类型的语句，并将其赋值给变量s
    s = new_stmt(cstate, BPF_LD|BPF_LEN);
    # 创建一个新的BPF_ST类型的语句，并将其赋值给变量s的下一个语句
    s->next = new_stmt(cstate, BPF_ST);
    # 将寄存器编号赋值给下一个语句的s.k属性
    s->next->s.k = regno;
    # 将s赋值给a的s属性
    a->s = s;
    # 将寄存器编号赋值给a的regno属性
    a->regno = regno;

    # 返回a
    return a;
}

static struct arth *
gen_loadi_internal(compiler_state_t *cstate, bpf_u_int32 val)
{
    struct arth *a;  // 声明一个结构体指针变量a
    struct slist *s;  // 声明一个结构体指针变量s
    int reg;  // 声明一个整型变量reg

    a = (struct arth *)newchunk(cstate, sizeof(*a));  // 为a分配内存空间

    reg = alloc_reg(cstate);  // 调用alloc_reg函数分配寄存器

    s = new_stmt(cstate, BPF_LD|BPF_IMM);  // 调用new_stmt函数创建一个新的指令
    s->s.k = val;  // 设置指令的值为val
    s->next = new_stmt(cstate, BPF_ST);  // 创建一个新的存储指令
    s->next->s.k = reg;  // 设置存储指令的值为reg
    a->s = s;  // 结构体指针a的成员变量s指向s
    a->regno = reg;  // 结构体指针a的成员变量regno赋值为reg

    return a;  // 返回结构体指针a
}

struct arth *
gen_loadi(compiler_state_t *cstate, bpf_u_int32 val)
{
    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 如果发生错误，则跳转到setjmp返回的位置
        return (NULL);  // 返回NULL

    return gen_loadi_internal(cstate, val);  // 调用gen_loadi_internal函数
}

/*
 * The a_arg dance is to avoid annoying whining by compilers that
 * a might be clobbered by longjmp - yeah, it might, but *WHO CARES*?
 * It's not *used* after setjmp returns.
 */
struct arth *
gen_neg(compiler_state_t *cstate, struct arth *a_arg)
{
    struct arth *a = a_arg;  // 声明一个结构体指针变量a，并赋值为a_arg
    struct slist *s;  // 声明一个结构体指针变量s

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 如果发生错误，则跳转到setjmp返回的位置
        return (NULL);  // 返回NULL

    s = xfer_to_a(cstate, a);  // 调用xfer_to_a函数
    sappend(a->s, s);  // 将s追加到a的成员变量s的末尾
    s = new_stmt(cstate, BPF_ALU|BPF_NEG);  // 创建一个新的算术指令
    s->s.k = 0;  // 设置指令的值为0
    sappend(a->s, s);  // 将s追加到a的成员变量s的末尾
    s = new_stmt(cstate, BPF_ST);  // 创建一个新的存储指令
    s->s.k = a->regno;  // 设置指令的值为a的成员变量regno
    sappend(a->s, s);  // 将s追加到a的成员变量s的末尾

    return a;  // 返回结构体指针a
}

/*
 * The a0_arg dance is to avoid annoying whining by compilers that
 * a0 might be clobbered by longjmp - yeah, it might, but *WHO CARES*?
 * It's not *used* after setjmp returns.
 */
struct arth *
gen_arth(compiler_state_t *cstate, int code, struct arth *a0_arg,
    struct arth *a1)
{
    struct arth *a0 = a0_arg;  // 声明一个结构体指针变量a0，并赋值为a0_arg
    struct slist *s0, *s1, *s2;  // 声明三个结构体指针变量s0, s1, s2

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))  // 如果发生错误，则跳转到setjmp返回的位置
        return (NULL);  // 返回NULL
    /*
     * 禁止除以零或取模零；我们在这里执行这个操作，即使优化器被禁用也能执行。
     *
     * 同样，禁止移位超过31位；我们在这里执行这个操作，出于同样的原因。
     */
    if (code == BPF_DIV) {
        if (a1->s->s.code == (BPF_LD|BPF_IMM) && a1->s->s.k == 0)
            bpf_error(cstate, "division by zero");
    } else if (code == BPF_MOD) {
        if (a1->s->s.code == (BPF_LD|BPF_IMM) && a1->s->s.k == 0)
            bpf_error(cstate, "modulus by zero");
    } else if (code == BPF_LSH || code == BPF_RSH) {
        if (a1->s->s.code == (BPF_LD|BPF_IMM) && a1->s->s.k > 31)
            bpf_error(cstate, "shift by more than 31 bits");
    }
    s0 = xfer_to_x(cstate, a1); // 将 a1 转移到 X 寄存器
    s1 = xfer_to_a(cstate, a0); // 将 a0 转移到 A 寄存器
    s2 = new_stmt(cstate, BPF_ALU|BPF_X|code); // 创建一个新的 ALU 操作语句

    sappend(s1, s2); // 将 s2 追加到 s1 后面
    sappend(s0, s1); // 将 s1 追加到 s0 后面
    sappend(a1->s, s0); // 将 s0 追加到 a1->s 后面
    sappend(a0->s, a1->s); // 将 a1->s 追加到 a0->s 后面

    free_reg(cstate, a0->regno); // 释放 a0 寄存器
    free_reg(cstate, a1->regno); // 释放 a1 寄存器

    s0 = new_stmt(cstate, BPF_ST); // 创建一个新的存储操作语句
    a0->regno = s0->s.k = alloc_reg(cstate); // 分配一个寄存器给 a0，并将其值赋给 s0->s.k
    sappend(a0->s, s0); // 将 s0 追加到 a0->s 后面

    return a0; // 返回 a0
/*
 * 初始化使用寄存器的表和当前寄存器。
 */
static void
init_regs(compiler_state_t *cstate)
{
    cstate->curreg = 0;  // 初始化当前寄存器为0
    memset(cstate->regused, 0, sizeof cstate->regused);  // 将使用寄存器的表初始化为0
}

/*
 * 返回下一个空闲寄存器。
 */
static int
alloc_reg(compiler_state_t *cstate)
{
    int n = BPF_MEMWORDS;  // 初始化n为BPF_MEMWORDS的值

    while (--n >= 0) {  // 循环遍历寄存器表
        if (cstate->regused[cstate->curreg])  // 如果当前寄存器已被使用
            cstate->curreg = (cstate->curreg + 1) % BPF_MEMWORDS;  // 则将当前寄存器移动到下一个位置
        else {
            cstate->regused[cstate->curreg] = 1;  // 标记当前寄存器为已使用
            return cstate->curreg;  // 返回当前寄存器的编号
        }
    }
    bpf_error(cstate, "too many registers needed to evaluate expression");  // 如果需要的寄存器数量过多，则报错
    /*NOTREACHED*/  // 不会执行到这里
}

/*
 * 将寄存器返回给表，以便以后可以使用。
 */
static void
free_reg(compiler_state_t *cstate, int n)
{
    cstate->regused[n] = 0;  // 将指定的寄存器标记为未使用
}

static struct block *
gen_len(compiler_state_t *cstate, int jmp, int n)
{
    struct slist *s;
    struct block *b;

    s = new_stmt(cstate, BPF_LD|BPF_LEN);  // 创建一个加载长度的语句
    b = new_block(cstate, JMP(jmp));  // 创建一个跳转指令的块
    b->stmts = s;  // 将加载长度的语句添加到块中
    b->s.k = n;  // 设置块的附加参数为n

    return b;  // 返回生成的块
}

struct block *
gen_greater(compiler_state_t *cstate, int n)
{
    /*
     * 捕获我们和下面的例程报告的错误，并在错误时返回NULL。
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点
        return (NULL);  // 如果发生错误，则返回NULL

    return gen_len(cstate, BPF_JGE, n);  // 生成一个长度大于等于n的块
}

/*
 * 实际上，这是小于或等于。
 */
struct block *
gen_less(compiler_state_t *cstate, int n)
{
    struct block *b;

    /*
     * 捕获我们和下面的例程报告的错误，并在错误时返回NULL。
     */
    if (setjmp(cstate->top_ctx))  // 设置错误跳转点
        return (NULL);  // 如果发生错误，则返回NULL

    b = gen_len(cstate, BPF_JGT, n);  // 生成一个长度大于n的块
    gen_not(b);  // 对生成的块进行取反操作

    return b;  // 返回生成的块
}
# 为给定的“byte {idx} {op} {val}”生成一个过滤块；“idx”被视为相对于链路层头部的位置。
# XXX - 这意味着您无法测试 radiotap 头部中的值，但是由于该头部通常很难甚至不可能一般性地解析而无法使用循环，这可能不是一个严重的问题。可以为此添加一个新关键字“radio”，尽管您真正想要的是一种测试特定无线电头部值的方法，这将生成适用于相应无线电头部的代码。
struct block *
gen_byteop(compiler_state_t *cstate, int op, int idx, bpf_u_int32 val)
{
    struct block *b;
    struct slist *s;

    # 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (op) {
    default:
        abort();

    case '=':
        return gen_cmp(cstate, OR_LINKHDR, (u_int)idx, BPF_B, val);

    case '<':
        b = gen_cmp_lt(cstate, OR_LINKHDR, (u_int)idx, BPF_B, val);
        return b;

    case '>':
        b = gen_cmp_gt(cstate, OR_LINKHDR, (u_int)idx, BPF_B, val);
        return b;

    case '|':
        s = new_stmt(cstate, BPF_ALU|BPF_OR|BPF_K);
        break;

    case '&':
        s = new_stmt(cstate, BPF_ALU|BPF_AND|BPF_K);
        break;
    }
    s->s.k = val;
    b = new_block(cstate, JMP(BPF_JEQ));
    b->stmts = s;
    gen_not(b);

    return b;
}

static const u_char abroadcast[] = { 0x0 };

# 生成广播地址的过滤块
struct block *
gen_broadcast(compiler_state_t *cstate, int proto)
{
    bpf_u_int32 hostmask;
    struct block *b0, *b1, *b2;
    static const u_char ebroadcast[] = { 0xff, 0xff, 0xff, 0xff, 0xff, 0xff };

    # 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (proto) {

    case Q_DEFAULT:
    # 如果过滤器类型是 Q_LINK
    case Q_LINK:
        # 根据连接类型进行不同的处理
        switch (cstate->linktype) {
            # 如果连接类型是 DLT_ARCNET 或 DLT_ARCNET_LINUX
            case DLT_ARCNET:
            case DLT_ARCNET_LINUX:
                # 返回生成的主机广播地址过滤器
                return gen_ahostop(cstate, abroadcast, Q_DST);
            # 如果连接类型是 DLT_EN10MB 或 DLT_NETANALYZER 或 DLT_NETANALYZER_TRANSPARENT
            case DLT_EN10MB:
            case DLT_NETANALYZER:
            case DLT_NETANALYZER_TRANSPARENT:
                # 生成前一个链接头部检查
                b1 = gen_prevlinkhdr_check(cstate);
                # 生成以太网主机广播地址过滤器
                b0 = gen_ehostop(cstate, ebroadcast, Q_DST);
                # 如果 b1 不为空，则生成与操作
                if (b1 != NULL)
                    gen_and(b1, b0);
                # 返回生成的以太网主机广播地址过滤器
                return b0;
            # 如果连接类型是 DLT_FDDI
            case DLT_FDDI:
                # 返回生成的 FDDI 主机广播地址过滤器
                return gen_fhostop(cstate, ebroadcast, Q_DST);
            # 如果连接类型是 DLT_IEEE802
            case DLT_IEEE802:
                # 返回生成的 Token Ring 主机广播地址过滤器
                return gen_thostop(cstate, ebroadcast, Q_DST);
            # 如果连接类型是 DLT_IEEE802_11 或 DLT_PRISM_HEADER 或 DLT_IEEE802_11_RADIO_AVS 或 DLT_IEEE802_11_RADIO 或 DLT_PPI
            case DLT_IEEE802_11:
            case DLT_PRISM_HEADER:
            case DLT_IEEE802_11_RADIO_AVS:
            case DLT_IEEE802_11_RADIO:
            case DLT_PPI:
                # 返回生成的 WLAN 主机广播地址过滤器
                return gen_wlanhostop(cstate, ebroadcast, Q_DST);
            # 如果连接类型是 DLT_IP_OVER_FC
            case DLT_IP_OVER_FC:
                # 返回生成的 IP over FC 主机广播地址过滤器
                return gen_ipfchostop(cstate, ebroadcast, Q_DST);
            # 如果连接类型不在以上列出的类型中
            default:
                # 抛出错误，表示不是广播连接
                bpf_error(cstate, "not a broadcast link");
        }
        # 不会执行到这里
        /*NOTREACHED*/

    # 如果过滤器类型是 Q_IP
    case Q_IP:
        # 如果网络掩码是未知的
        if (cstate->netmask == PCAP_NETMASK_UNKNOWN)
            # 抛出错误，表示不支持 'ip broadcast'
            bpf_error(cstate, "netmask not known, so 'ip broadcast' not supported");
        # 生成以太网类型为 ETHERTYPE_IP 的过滤器
        b0 = gen_linktype(cstate, ETHERTYPE_IP);
        # 计算主机掩码
        hostmask = ~cstate->netmask;
        # 生成比较操作，用于匹配主机地址
        b1 = gen_mcmp(cstate, OR_LINKPL, 16, BPF_W, 0, hostmask);
        b2 = gen_mcmp(cstate, OR_LINKPL, 16, BPF_W,
                  ~0 & hostmask, hostmask);
        # 生成或操作
        gen_or(b1, b2);
        # 生成与操作
        gen_and(b0, b2);
        # 返回生成的过滤器
        return b2;
    # 如果过滤器类型不是 Q_LINK 或 Q_IP
    bpf_error(cstate, "only link-layer/IP broadcast filters supported");
    # 不会执行到这里
    /*NOTREACHED*/
}

/*
 * 生成用于测试 MAC 地址的低位比特（即第一个字节的最低位）的代码。
 */
static struct block *
gen_mac_multicast(compiler_state_t *cstate, int offset)
{
    register struct block *b0;
    register struct slist *s;

    /* 获取 link[offset] 的值，并检查最低位是否不为 0 */
    s = gen_load_a(cstate, OR_LINKHDR, offset, BPF_B);
    b0 = new_block(cstate, JMP(BPF_JSET));
    b0->s.k = 1;
    b0->stmts = s;
    return b0;
}

struct block *
gen_multicast(compiler_state_t *cstate, int proto)
{
    register struct block *b0, *b1, *b2;
    register struct slist *s;

    /*
     * 捕获我们报告的错误以及下面的例程报告的错误，并在出现错误时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (proto) {

    case Q_DEFAULT:
    case Q_IP:
        b0 = gen_linktype(cstate, ETHERTYPE_IP);
        b1 = gen_cmp_ge(cstate, OR_LINKPL, 16, BPF_B, 224);
        gen_and(b0, b1);
        return b1;

    case Q_IPV6:
        b0 = gen_linktype(cstate, ETHERTYPE_IPV6);
        b1 = gen_cmp(cstate, OR_LINKPL, 24, BPF_B, 255);
        gen_and(b0, b1);
        return b1;
    }
    bpf_error(cstate, "link-layer multicast filters supported only on ethernet/FDDI/token ring/ARCNET/802.11/ATM LANE/Fibre Channel");
    /*NOTREACHED*/
}

struct block *
gen_ifindex(compiler_state_t *cstate, int ifindex)
{
    register struct block *b0;

    /*
     * 捕获我们报告的错误以及下面的例程报告的错误，并在出现错误时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * 只有一些数据链路类型支持 ifindex 限定符。
     */
    switch (cstate->linktype) {
    case DLT_LINUX_SLL2:
        /* 匹配在此接口上的数据包 */
        b0 = gen_cmp(cstate, OR_LINKHDR, 4, BPF_W, ifindex);
        break;
        default:
#if defined(linux)
        /*
         * 如果定义了 linux，则需要 PF_PACKET 支持。
         * 如果是 *live* 捕获，我们可以查看过滤表达式中的特殊元数据；
         * 如果是保存文件，我们就不能这样做。
         */
        if (cstate->bpf_pcap->rfile != NULL) {
            /* 我们有一个 FILE *，所以这是一个保存文件 */
            bpf_error(cstate, "ifindex not supported on %s when reading savefiles",
                pcap_datalink_val_to_description_or_dlt(cstate->linktype));
            b0 = NULL;
            /*NOTREACHED*/
        }
        /* 匹配 ifindex */
        b0 = gen_cmp(cstate, OR_LINKHDR, SKF_AD_OFF + SKF_AD_IFINDEX, BPF_W,
                     ifindex);
#else /* defined(linux) */
        bpf_error(cstate, "ifindex not supported on %s",
            pcap_datalink_val_to_description_or_dlt(cstate->linktype));
        /*NOTREACHED*/
#endif /* defined(linux) */
    }
    return (b0);
}

/*
 * 过滤入站（dir == 0）或出站（dir == 1）流量。
 * 出站流量是由本机发送的，而入站流量是由远程机器发送的（可能包括发送到我们未订阅的单播或组播链路层地址的数据包）。
 * 这些是由 pcap_setdirection() 实现的相同定义。
 * 只捕获发送到本主机的单播流量可能更好地使用更高层的过滤器来实现。
 */
struct block *
gen_inbound(compiler_state_t *cstate, int dir)
{
    register struct block *b0;

    /*
     * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * 只有一些数据链路类型支持入站/出站限定符。
     */
    switch (cstate->linktype) {
    # 根据不同的数据链路类型进行条件判断和生成过滤器指令

    case DLT_SLIP:
        # 如果数据链路类型是SLIP，则生成对应的过滤器指令
        b0 = gen_relation_internal(cstate, BPF_JEQ,
              gen_load_internal(cstate, Q_LINK, gen_loadi_internal(cstate, 0), 1),
              gen_loadi_internal(cstate, 0),
              dir);
        break;

    case DLT_IPNET:
        if (dir) {
            # 匹配出站数据包
            b0 = gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, IPNET_OUTBOUND);
        } else {
            # 匹配入站数据包
            b0 = gen_cmp(cstate, OR_LINKHDR, 2, BPF_H, IPNET_INBOUND);
        }
        break;

    case DLT_LINUX_SLL:
        # 匹配出站数据包
        b0 = gen_cmp(cstate, OR_LINKHDR, 0, BPF_H, LINUX_SLL_OUTGOING);
        if (!dir) {
            # 为了过滤入站流量，反转匹配条件
            gen_not(b0);
        }
        break;

    case DLT_LINUX_SLL2:
        # 匹配出站数据包
        b0 = gen_cmp(cstate, OR_LINKHDR, 10, BPF_B, LINUX_SLL_OUTGOING);
        if (!dir) {
            # 为了过滤入站流量，反转匹配条件
            gen_not(b0);
        }
        break;

    case DLT_PFLOG:
        # 根据流量方向生成过滤器指令
        b0 = gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, dir), BPF_B,
            ((dir == 0) ? PF_IN : PF_OUT));
        break;

    case DLT_PPP_PPPD:
        if (dir) {
            # 匹配出站数据包
            b0 = gen_cmp(cstate, OR_LINKHDR, 0, BPF_B, PPP_PPPD_OUT);
        } else {
            # 匹配入站数据包
            b0 = gen_cmp(cstate, OR_LINKHDR, 0, BPF_B, PPP_PPPD_IN);
        }
        break;

    case DLT_JUNIPER_MFR:
    case DLT_JUNIPER_MLFR:
    case DLT_JUNIPER_MLPPP:
    case DLT_JUNIPER_ATM1:
    case DLT_JUNIPER_ATM2:
    case DLT_JUNIPER_PPPOE:
        # 其他数据链路类型暂时未处理
    case DLT_JUNIPER_PPPOE_ATM:
        // 如果是 Juniper 的 PPPoE ATM 类型
        case DLT_JUNIPER_GGSN:
        // 如果是 Juniper 的 GGSN 类型
        case DLT_JUNIPER_ES:
        // 如果是 Juniper 的 ES 类型
        case DLT_JUNIPER_MONITOR:
        // 如果是 Juniper 的 Monitor 类型
        case DLT_JUNIPER_SERVICES:
        // 如果是 Juniper 的 Services 类型
        case DLT_JUNIPER_ETHER:
        // 如果是 Juniper 的以太网类型
        case DLT_JUNIPER_PPP:
        // 如果是 Juniper 的 PPP 类型
        case DLT_JUNIPER_FRELAY:
        // 如果是 Juniper 的 FRELAY 类型
        case DLT_JUNIPER_CHDLC:
        // 如果是 Juniper 的 CHDLC 类型
        case DLT_JUNIPER_VP:
        // 如果是 Juniper 的 VP 类型
        case DLT_JUNIPER_ST:
        // 如果是 Juniper 的 ST 类型
        case DLT_JUNIPER_ISM:
        // 如果是 Juniper 的 ISM 类型
        case DLT_JUNIPER_VS:
        // 如果是 Juniper 的 VS 类型
        case DLT_JUNIPER_SRX_E2E:
        // 如果是 Juniper 的 SRX E2E 类型
        case DLT_JUNIPER_FIBRECHANNEL:
        // 如果是 Juniper 的 FibreChannel 类型
    case DLT_JUNIPER_ATM_CEMIC:
        // 如果是 Juniper 的 ATM CEMIC 类型

        /* juniper flags (including direction) are stored
         * the byte after the 3-byte magic number */
        // Juniper 标志（包括方向）存储在 3 字节魔术数字后的字节中
        if (dir) {
            // 如果有方向信息，匹配出站数据包
            b0 = gen_mcmp(cstate, OR_LINKHDR, 3, BPF_B, 0, 0x01);
        } else {
            // 如果有方向信息，匹配入站数据包
            b0 = gen_mcmp(cstate, OR_LINKHDR, 3, BPF_B, 1, 0x01);
        }
        break;

    default:
        /*
         * 如果我们有指示方向的数据包元数据，并且该元数据可以被 BPF 代码检查，那么进行检查。
         * 否则放弃，因为这种链路层类型在数据包中没有任何内容。
         *
         * 目前，唯一支持 BPF 过滤器检查元数据的平台是具有内核内 BPF 解释器的 Linux。
         * 如果其他数据包捕获机制和 BPF 过滤器也支持这一点，那将是很好的。
         * 如果它们还提供了该元数据，以便我们可以在新的捕获 API 中提供它，并将其保存在 pcapng 文件中，那将更好。
         */
#if defined(linux)
        /*
         * 如果定义了 linux，则需要 PF_PACKET 支持。
         * 如果是 *live* 抓包，可以查看过滤表达式中的特殊元数据；
         * 如果是保存文件，就不行。
         */
        if (cstate->bpf_pcap->rfile != NULL) {
            /* 有一个 FILE *，所以这是一个保存文件 */
            bpf_error(cstate, "inbound/outbound not supported on %s when reading savefiles",
                pcap_datalink_val_to_description_or_dlt(cstate->linktype));
            /*NOTREACHED*/
        }
        /* 匹配出站数据包 */
        b0 = gen_cmp(cstate, OR_LINKHDR, SKF_AD_OFF + SKF_AD_PKTTYPE, BPF_H,
                     PACKET_OUTGOING);
        if (!dir) {
            /* 要过滤入站流量，反转匹配条件 */
            gen_not(b0);
        }
#else /* defined(linux) */
        bpf_error(cstate, "inbound/outbound not supported on %s",
            pcap_datalink_val_to_description_or_dlt(cstate->linktype));
        /*NOTREACHED*/
#endif /* defined(linux) */
    }
    return (b0);
}

/* PF 防火墙日志匹配接口 */
struct block *
gen_pf_ifname(compiler_state_t *cstate, const char *ifname)
{
    struct block *b0;
    u_int len, off;

    /*
     * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "ifname supported only on PF linktype");
        /*NOTREACHED*/
    }
    len = sizeof(((struct pfloghdr *)0)->ifname);
    off = offsetof(struct pfloghdr, ifname);
    if (strlen(ifname) >= len) {
        bpf_error(cstate, "ifname interface names can only be %d characters",
            len-1);
        /*NOTREACHED*/
    }
    b0 = gen_bcmp(cstate, OR_LINKHDR, off, (u_int)strlen(ifname),
        (const u_char *)ifname);
    return (b0);
}

/* PF 防火墙日志规则集名称 */
struct block *
# 生成 PF 规则集
gen_pf_ruleset(compiler_state_t *cstate, char *ruleset)
{
    struct block *b0;

    '''
    * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
    '''
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "ruleset supported only on PF linktype");
        /*NOTREACHED*/
    }

    if (strlen(ruleset) >= sizeof(((struct pfloghdr *)0)->ruleset)) {
        bpf_error(cstate, "ruleset names can only be %ld characters",
            (long)(sizeof(((struct pfloghdr *)0)->ruleset) - 1));
        /*NOTREACHED*/
    }

    b0 = gen_bcmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, ruleset),
        (u_int)strlen(ruleset), (const u_char *)ruleset);
    return (b0);
}

# PF 防火墙日志规则编号
struct block *
gen_pf_rnr(compiler_state_t *cstate, int rnr)
{
    struct block *b0;

    '''
    * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
    '''
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "rnr supported only on PF linktype");
        /*NOTREACHED*/
    }

    b0 = gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, rulenr), BPF_W,
         (bpf_u_int32)rnr);
    return (b0);
}

# PF 防火墙日志子规则编号
struct block *
gen_pf_srnr(compiler_state_t *cstate, int srnr)
{
    struct block *b0;

    '''
    * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
    '''
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "srnr supported only on PF linktype");
        /*NOTREACHED*/
    }

    b0 = gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, subrulenr), BPF_W,
        (bpf_u_int32)srnr);
    return (b0);
}

# PF 防火墙日志原因代码
struct block *
gen_pf_reason(compiler_state_t *cstate, int reason)
    # 定义一个指向结构体 block 的指针 b0
    struct block *b0;

    # 设置一个错误捕获点，如果出现错误则返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    # 如果链路类型不是 DLT_PFLOG，则报错并终止程序
    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "reason supported only on PF linktype");
        /*NOTREACHED*/
    }

    # 生成一个比较指令，比较偏移为 offsetof(struct pfloghdr, reason) 的位置上的数据和给定的 reason
    b0 = gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, reason), BPF_B,
        (bpf_u_int32)reason);
    # 返回生成的指令
    return (b0);
}

/* PF firewall log action */
// 生成用于匹配 PF 防火墙日志动作的过滤器块
struct block *
gen_pf_action(compiler_state_t *cstate, int action)
{
    struct block *b0;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 捕获我们和下面例程报告的错误，并在出现错误时返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    if (cstate->linktype != DLT_PFLOG) {
        bpf_error(cstate, "action supported only on PF linktype");
        /*NOTREACHED*/
    }

    b0 = gen_cmp(cstate, OR_LINKHDR, offsetof(struct pfloghdr, action), BPF_B,
        (bpf_u_int32)action);
    return (b0);
}

/* IEEE 802.11 wireless header */
// 生成用于匹配 IEEE 802.11 无线头部的过滤器块
struct block *
gen_p80211_type(compiler_state_t *cstate, bpf_u_int32 type, bpf_u_int32 mask)
{
    struct block *b0;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 捕获我们和下面例程报告的错误，并在出现错误时返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (cstate->linktype) {

    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
        b0 = gen_mcmp(cstate, OR_LINKHDR, 0, BPF_B, type, mask);
        break;

    default:
        bpf_error(cstate, "802.11 link-layer types supported only on 802.11");
        /*NOTREACHED*/
    }

    return (b0);
}

// 生成用于匹配 IEEE 802.11 帧方向的过滤器块
struct block *
gen_p80211_fcdir(compiler_state_t *cstate, bpf_u_int32 fcdir)
{
    struct block *b0;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 捕获我们和下面例程报告的错误，并在出现错误时返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (cstate->linktype) {

    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
        break;

    default:
        bpf_error(cstate, "frame direction supported only with 802.11 headers");
        /*NOTREACHED*/
    }

    b0 = gen_mcmp(cstate, OR_LINKHDR, 1, BPF_B, fcdir,
        IEEE80211_FC1_DIR_MASK);

    return (b0);
}

// 生成用于匹配特定条件的过滤器块
struct block *
gen_acode(compiler_state_t *cstate, const char *s, struct qual q)
    // 声明一个指向结构体 block 的指针 b
    struct block *b;

    /*
     * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    // 根据链路类型进行不同的处理
    switch (cstate->linktype) {

    case DLT_ARCNET:
    case DLT_ARCNET_LINUX:
        // 如果地址是 Q_HOST 或者 Q_DEFAULT，并且协议是 Q_LINK
        if ((q.addr == Q_HOST || q.addr == Q_DEFAULT) &&
            q.proto == Q_LINK) {
            // 将字符串转换为以太网地址
            cstate->e = pcap_ether_aton(s);
            if (cstate->e == NULL)
                bpf_error(cstate, "malloc");
            // 生成一个主机地址操作码
            b = gen_ahostop(cstate, cstate->e, (int)q.dir);
            // 释放以太网地址的内存
            free(cstate->e);
            cstate->e = NULL;
            return (b);
        } else
            // 报告 ARCnet 地址在非 ARC 表达式中使用的错误
            bpf_error(cstate, "ARCnet address used in non-arc expression");
        /*NOTREACHED*/

    default:
        // 报告只有 ARCnet 支持的错误
        bpf_error(cstate, "aid supported only on ARCnet");
        /*NOTREACHED*/
    }
}

static struct block *
gen_ahostop(compiler_state_t *cstate, const u_char *eaddr, int dir)
{
    register struct block *b0, *b1;

    switch (dir) {
    /* 如果是源地址，生成比较指令 */
    case Q_SRC:
        return gen_bcmp(cstate, OR_LINKHDR, 0, 1, eaddr);

    /* 如果是目的地址，生成比较指令 */
    case Q_DST:
        return gen_bcmp(cstate, OR_LINKHDR, 1, 1, eaddr);

    /* 如果是与操作，递归生成源地址和目的地址的比较指令，并进行与操作 */
    case Q_AND:
        b0 = gen_ahostop(cstate, eaddr, Q_SRC);
        b1 = gen_ahostop(cstate, eaddr, Q_DST);
        gen_and(b0, b1);
        return b1;

    /* 如果是默认或者或操作，递归生成源地址和目的地址的比较指令，并进行或操作 */
    case Q_DEFAULT:
    case Q_OR:
        b0 = gen_ahostop(cstate, eaddr, Q_SRC);
        b1 = gen_ahostop(cstate, eaddr, Q_DST);
        gen_or(b0, b1);
        return b1;

    /* 如果是地址1，报错，802.11不支持 */
    case Q_ADDR1:
        bpf_error(cstate, "'addr1' and 'address1' are only supported on 802.11");
        /*NOTREACHED*/

    /* 如果是地址2，报错，802.11不支持 */
    case Q_ADDR2:
        bpf_error(cstate, "'addr2' and 'address2' are only supported on 802.11");
        /*NOTREACHED*/

    /* 如果是地址3，报错，802.11不支持 */
    case Q_ADDR3:
        bpf_error(cstate, "'addr3' and 'address3' are only supported on 802.11");
        /*NOTREACHED*/

    /* 如果是地址4，报错，802.11不支持 */
    case Q_ADDR4:
        bpf_error(cstate, "'addr4' and 'address4' are only supported on 802.11");
        /*NOTREACHED*/

    /* 如果是RA，报错，802.11不支持 */
    case Q_RA:
        bpf_error(cstate, "'ra' is only supported on 802.11");
        /*NOTREACHED*/

    /* 如果是TA，报错，802.11不支持 */
    case Q_TA:
        bpf_error(cstate, "'ta' is only supported on 802.11");
        /*NOTREACHED*/
    }
    abort();
    /*NOTREACHED*/
}

static struct block *
gen_vlan_tpid_test(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    /* 检查是否是 VLAN，包括 802.1ad 和 QinQ */
    b0 = gen_linktype(cstate, ETHERTYPE_8021Q);
    b1 = gen_linktype(cstate, ETHERTYPE_8021AD);
    gen_or(b0,b1);
    b0 = b1;
    b1 = gen_linktype(cstate, ETHERTYPE_8021QINQ);
    gen_or(b0,b1);

    return b1;
}

static struct block *
gen_vlan_vid_test(compiler_state_t *cstate, bpf_u_int32 vlan_num)
{
    # 如果 VLAN 号大于 0x0fff，则抛出错误
    if (vlan_num > 0x0fff) {
        bpf_error(cstate, "VLAN tag %u greater than maximum %u",
            vlan_num, 0x0fff);
    }
    # 生成一个比较操作码，用于匹配数据链路层中的 VLAN 号
    return gen_mcmp(cstate, OR_LINKPL, 0, BPF_H, vlan_num, 0x0fff);
}
# 生成一个 VLAN 无 BPF 扩展的结构体块
static struct block *
gen_vlan_no_bpf_extensions(compiler_state_t *cstate, bpf_u_int32 vlan_num,
    int has_vlan_tag)
{
    struct block *b0, *b1;

    # 生成 VLAN TPID 测试的结构体块
    b0 = gen_vlan_tpid_test(cstate);

    # 如果存在 VLAN 标签
    if (has_vlan_tag) {
        # 生成 VLAN VID 测试的结构体块
        b1 = gen_vlan_vid_test(cstate, vlan_num);
        # 生成并操作，将 b0 和 b1 进行与操作
        gen_and(b0, b1);
        b0 = b1;
    }

    """
    由于 VLAN 标签之后同时跟随载荷和链路头类型，因此两者都需要更新。
    """
    # 更新链路载荷偏移的常量部分
    cstate->off_linkpl.constant_part += 4;
    # 更新链路类型偏移的常量部分
    cstate->off_linktype.constant_part += 4;

    # 返回结构体块
    return b0;
}

# 如果定义了 SKF_AD_VLAN_TAG_PRESENT
# 将 v 添加到 off 的变量部分
static void
gen_vlan_vloffset_add(compiler_state_t *cstate, bpf_abs_offset *off,
    bpf_u_int32 v, struct slist *s)
{
    struct slist *s2;

    # 如果 off 不是变量，则将其设置为变量
    if (!off->is_variable)
        off->is_variable = 1;
    # 如果 off 的寄存器为 -1，则分配一个寄存器
    if (off->reg == -1)
        off->reg = alloc_reg(cstate);

    # 创建一个 BPF_LD|BPF_MEM 类型的新语句
    s2 = new_stmt(cstate, BPF_LD|BPF_MEM);
    s2->s.k = off->reg;
    sappend(s, s2);
    # 创建一个 BPF_ALU|BPF_ADD|BPF_IMM 类型的新语句
    s2 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_IMM);
    s2->s.k = v;
    sappend(s, s2);
    # 创建一个 BPF_ST 类型的新语句
    s2 = new_stmt(cstate, BPF_ST);
    s2->s.k = off->reg;
    sappend(s, s2);
}

# 将 VLAN TPID 测试的结构体块进行修补，以首先更新链路载荷和链路类型偏移的变量部分
static void
gen_vlan_patch_tpid_test(compiler_state_t *cstate, struct block *b_tpid)
{
    struct slist s;

    # 偏移在运行时确定，移动变量部分
    s.next = NULL;
    cstate->is_vlan_vloffset = 1;
    # 添加 4 到链路载荷偏移的变量部分
    gen_vlan_vloffset_add(cstate, &cstate->off_linkpl, 4, &s);
    # 添加 4 到链路类型偏移的变量部分
    gen_vlan_vloffset_add(cstate, &cstate->off_linktype, 4, &s);

    # 获取一个或连接块链的指针，修补其中的第一个块
    sappend(s.next, b_tpid->head->stmts);
    b_tpid->head->stmts = s.next;
}

# 修补 VLAN id 测试的结构体块，根据 SKF_AD_VLAN_TAG_PRESENT 是否为真，从数据包元数据中加载 VID 值
static void
gen_vlan_patch_vid_test(compiler_state_t *cstate, struct block *b_vid)
    # 定义三个指向结构体的指针变量和一个无符号整型变量
    struct slist *s, *s2, *sjeq;
    unsigned cnt;
    
    # 创建一个新的指令，并设置操作码和操作数
    s = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
    s->s.k = SKF_AD_OFF + SKF_AD_VLAN_TAG_PRESENT;
    
    # 创建一个新的条件跳转指令，设置条件为真时跳转到下一条指令，条件为假时跳转到b_vid块的开头
    sjeq = new_stmt(cstate, JMP(BPF_JEQ));
    sjeq->s.k = 1;
    sjeq->s.jf = b_vid->stmts;
    sappend(s, sjeq);
    
    # 创建一个新的指令，并设置操作码和操作数
    s2 = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
    s2->s.k = SKF_AD_OFF + SKF_AD_VLAN_TAG;
    sappend(s, s2);
    sjeq->s.jt = s2;
    
    # 计算b_vid块中指令的数量
    cnt = 0;
    for (s2 = b_vid->stmts; s2; s2 = s2->next)
        cnt++;
    
    # 创建一个新的无条件跳转指令，设置跳转偏移量为b_vid块中指令数量减一
    s2 = new_stmt(cstate, JMP(BPF_JA));
    s2->s.k = cnt - 1;
    sappend(s, s2);
    
    # 将当前指令插入到b_vid块的开头
    sappend(s, b_vid->stmts);
    b_vid->stmts = s;
/*
 * 生成对支持 BPF 扩展的系统上的 "vlan" 或 "vlan <id>" 的检查。
 * 即使内核支持 VLAN BPF 扩展，（最外层）VLAN 标记可以存在于元数据中，也可以存在于数据包数据中；
 * 因此，如果 SKF_AD_VLAN_TAG_PRESENT 测试为负，我们需要在链路头中检查 VLAN 标记。
 * 由于决定是在运行时进行的，我们需要更新偏移量的变量部分
 */
static struct block *
gen_vlan_bpf_extensions(compiler_state_t *cstate, bpf_u_int32 vlan_num,
    int has_vlan_tag)
{
        struct block *b0, *b_tpid, *b_vid = NULL;
        struct slist *s;

        /* 基于提取数据包元数据生成新的过滤器代码 */
        s = new_stmt(cstate, BPF_LD|BPF_B|BPF_ABS);
        s->s.k = SKF_AD_OFF + SKF_AD_VLAN_TAG_PRESENT;

        b0 = new_block(cstate, JMP(BPF_JEQ));
        b0->stmts = s;
        b0->s.k = 1;

    /*
     * 这有点棘手。我们需要在传统的 TPID 和 VID 测试之前插入更新偏移量的变量部分的语句，
     * 以便它们在 SKF_AD_VLAN_TAG_PRESENT 失败时被调用，但我们不希望此更新影响这些检查。
     * 这就是为什么我们首先生成两个测试块，并在此之后插入更新偏移量的变量部分的语句。
     * 如果在进入此函数时已经存在可变长度的链路头，这种方法就行不通了，
     * 但在这种情况下不会调用 gen_vlan_bpf_extensions()。
     */
    b_tpid = gen_vlan_tpid_test(cstate);
    if (has_vlan_tag)
        b_vid = gen_vlan_vid_test(cstate, vlan_num);

    gen_vlan_patch_tpid_test(cstate, b_tpid);
    gen_or(b0, b_tpid);
    b0 = b_tpid;

    if (has_vlan_tag) {
        gen_vlan_patch_vid_test(cstate, b_vid);
        gen_and(b0, b_vid);
        b0 = b_vid;
    }

        return b0;
}
#endif

/*
 * 支持以太网上的 IEEE 802.1Q VLAN 干线
 */
struct block *
gen_vlan(compiler_state_t *cstate, bpf_u_int32 vlan_num, int has_vlan_tag)
{
    # 定义一个指向结构体的指针变量b0
    struct    block    *b0;

    """
     * 捕获我们报告的错误和下面的例程报告的错误，并在出现错误时返回NULL
     """
    # 如果setjmp返回非零值，表示有错误发生，返回NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    """
     * 无法在MPLS内部检查VLAN封装的数据包
     """
    # 如果标签堆栈深度大于0，表示在MPLS内部，抛出错误
    if (cstate->label_stack_depth > 0)
        bpf_error(cstate, "no VLAN match after MPLS");
    /*
     * 检查是否为 VLAN 数据包，然后更改偏移量以指向 VLAN 数据包内的类型和数据字段。只是增加偏移量，以便支持层次结构，例如
     * "vlan 300 && vlan 200" 以捕获 VLAN 100 中封装的 VLAN 200。
     *
     * XXX - 这有点笨拙。如果我们将编译器分成解析表达式并生成表达式树的解析器，以及接受表达式树（可以来自我们的解析器或其他解析器）并生成 BPF 代码的代码生成器，我们或许可以将偏移量作为例程的参数，并在“AND”节点的处理程序中传递调整后的偏移量给 VLAN 节点以外的子节点。
     *
     * 这意味着“vlan”不会改变其后的所有测试的行为，而是仅改变与其 AND 运算的测试的行为。这将改变“vlan”的文档化语义，这可能会破坏一些表达式。但是，这意味着“(vlan and ip) or ip”将同时检查 VLAN 封装的 IP 和以太网上的 IP，而不仅仅是检查 VLAN 封装的 IP，因此这可能仍然值得做；这不会破坏形式为“vlan and ...”或“vlan N and ...”的表达式，我怀疑这些是涉及“vlan”的最常见表达式。“vlan or ...”现在不一定会做用户真正想要的事情，因为所有“or ...”测试都会假定为 VLAN，即使“or”可以被视为“或者，如果这不是一个 VLAN 数据包...”。
     */
    switch (cstate->linktype) {

    case DLT_EN10MB:
    case DLT_NETANALYZER:
    case DLT_NETANALYZER_TRANSPARENT:
#if defined(SKF_AD_VLAN_TAG_PRESENT)
        /* 如果定义了SKF_AD_VLAN_TAG_PRESENT，则执行以下代码块 */
        /* 验证这是否是数据包的外部部分，而不是以某种方式封装的 */
        if (cstate->vlan_stack_depth == 0 && !cstate->off_linkhdr.is_variable &&
            cstate->off_linkhdr.constant_part ==
            cstate->off_outermostlinkhdr.constant_part) {
            /*
             * 是否需要特殊的VLAN处理？
             */
            if (cstate->bpf_pcap->bpf_codegen_flags & BPF_SPECIAL_VLAN_HANDLING)
                b0 = gen_vlan_bpf_extensions(cstate, vlan_num,
                    has_vlan_tag);
            else
                b0 = gen_vlan_no_bpf_extensions(cstate,
                    vlan_num, has_vlan_tag);
        } else
#endif
            b0 = gen_vlan_no_bpf_extensions(cstate, vlan_num,
                has_vlan_tag);
                break;

    case DLT_IEEE802_11:
    case DLT_PRISM_HEADER:
    case DLT_IEEE802_11_RADIO_AVS:
    case DLT_IEEE802_11_RADIO:
        b0 = gen_vlan_no_bpf_extensions(cstate, vlan_num, has_vlan_tag);
        break;

    default:
        bpf_error(cstate, "no VLAN support for %s",
              pcap_datalink_val_to_description_or_dlt(cstate->linktype));
        /*NOTREACHED*/
    }

        cstate->vlan_stack_depth++;

    return (b0);
}

/*
 * support for MPLS
 *
 * The label_num_arg dance is to avoid annoying whining by compilers that
 * label_num might be clobbered by longjmp - yeah, it might, but *WHO CARES*?
 * It's not *used* after setjmp returns.
 */
struct block *
gen_mpls(compiler_state_t *cstate, bpf_u_int32 label_num_arg,
    int has_label_num)
{
    volatile bpf_u_int32 label_num = label_num_arg;
    struct    block    *b0, *b1;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);  # 如果发生错误，返回空指针

        if (cstate->label_stack_depth > 0) {
            /* just match the bottom-of-stack bit clear */
            b0 = gen_mcmp(cstate, OR_PREVMPLSHDR, 2, BPF_B, 0, 0x01);  # 如果标签栈深度大于0，匹配栈底位清零
        } else {
            /*
             * We're not in an MPLS stack yet, so check the link-layer
             * type against MPLS.
             */
            switch (cstate->linktype) {

            case DLT_C_HDLC: /* fall through */
            case DLT_HDLC:
            case DLT_EN10MB:
            case DLT_NETANALYZER:
            case DLT_NETANALYZER_TRANSPARENT:
                    b0 = gen_linktype(cstate, ETHERTYPE_MPLS);  # 生成与 MPLS 相关的链路类型
                    break;

            case DLT_PPP:
                    b0 = gen_linktype(cstate, PPP_MPLS_UCAST);  # 生成与 PPP MPLS 单播相关的链路类型
                    break;

                    /* FIXME add other DLT_s ...
                     * for Frame-Relay/and ATM this may get messy due to SNAP headers
                     * leave it for now */

            default:
                    bpf_error(cstate, "no MPLS support for %s",
                          pcap_datalink_val_to_description_or_dlt(cstate->linktype));  # 对于其他链路类型，报错并返回
                    /*NOTREACHED*/
            }
        }

    /* If a specific MPLS label is requested, check it */
    if (has_label_num) {
        if (label_num > 0xFFFFF) {
            bpf_error(cstate, "MPLS label %u greater than maximum %u",
                label_num, 0xFFFFF);  # 如果请求的 MPLS 标签大于最大值，报错
        }
        label_num = label_num << 12; /* label is shifted 12 bits on the wire */
        b1 = gen_mcmp(cstate, OR_LINKPL, 0, BPF_W, label_num,
            0xfffff000); /* only compare the first 20 bits */  # 比较第一个 20 位
        gen_and(b0, b1);  # 生成与操作
        b0 = b1;
    }
        /*
         * 将偏移量更改为指向 MPLS 数据包中的类型和数据字段。
         * 只需增加偏移量，以便支持层次结构，例如 "mpls 100000 && mpls 1024"，
         * 以捕获外部标签为 100000，内部标签为 1024 的数据包。
         *
         * 同样增加 MPLS 栈深度；这表示我们正在检查 MPLS 封装的标头，
         * 以确保更高级别的代码生成器不会尝试匹配诸如 Q_ARP、Q_RARP 等与 IP 相关的协议。
         *
         * XXX - 这有点笨拙。请参阅 gen_vlan() 中的注释。
         */
        cstate->off_nl_nosnap += 4;
        cstate->off_nl += 4;
        cstate->label_stack_depth++;
    return (b0);
}

/*
 * Support PPPOE discovery and session.
 */
struct block *
gen_pppoed(compiler_state_t *cstate)
{
    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /* check for PPPoE discovery */
    return gen_linktype(cstate, ETHERTYPE_PPPOED);
}

struct block *
gen_pppoes(compiler_state_t *cstate, bpf_u_int32 sess_num, int has_sess_num)
{
    struct block *b0, *b1;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    /*
     * Test against the PPPoE session link-layer type.
     */
    b0 = gen_linktype(cstate, ETHERTYPE_PPPOES);

    /* If a specific session is requested, check PPPoE session id */
    if (has_sess_num) {
        if (sess_num > 0x0000ffff) {
            bpf_error(cstate, "PPPoE session number %u greater than maximum %u",
                sess_num, 0x0000ffff);
        }
        b1 = gen_mcmp(cstate, OR_LINKPL, 0, BPF_W, sess_num, 0x0000ffff);
        gen_and(b0, b1);
        b0 = b1;
    }

    /*
     * Change the offsets to point to the type and data fields within
     * the PPP packet, and note that this is PPPoE rather than
     * raw PPP.
     *
     * XXX - this is a bit of a kludge.  See the comments in
     * gen_vlan().
     *
     * The "network-layer" protocol is PPPoE, which has a 6-byte
     * PPPoE header, followed by a PPP packet.
     *
     * There is no HDLC encapsulation for the PPP packet (it's
     * encapsulated in PPPoES instead), so the link-layer type
     * starts at the first byte of the PPP packet.  For PPPoE,
     * that offset is relative to the beginning of the total
     * link-layer payload, including any 802.2 LLC header, so
     * it's 6 bytes past cstate->off_nl.
     */
    # 将 PPP 链路头信息推送到捕获状态中
    PUSH_LINKHDR(cstate, DLT_PPP, cstate->off_linkpl.is_variable,
        cstate->off_linkpl.constant_part + cstate->off_nl + 6, /* 6 bytes past the PPPoE header */
        cstate->off_linkpl.reg);

    # 设置链路类型偏移量为链路头偏移量
    cstate->off_linktype = cstate->off_linkhdr;
    # 设置链路头常量部分偏移量为链路头偏移量常量部分加 2
    cstate->off_linkpl.constant_part = cstate->off_linkhdr.constant_part + 2;

    # 设置网络层偏移量为 0
    cstate->off_nl = 0;
    # 设置无 SNAP 的网络层偏移量为 0
    cstate->off_nl_nosnap = 0;    /* no 802.2 LLC */

    # 返回捕获状态
    return b0;
/* 检查是否为 Geneve 协议，并且如果指定了 VNI，则检查 VNI 是否正确。参数化以处理 IPv4 和 IPv6。 */
static struct block *
gen_geneve_check(compiler_state_t *cstate,
    struct block *(*gen_portfn)(compiler_state_t *, u_int, int, int),
    enum e_offrel offrel, bpf_u_int32 vni, int has_vni)
{
    struct block *b0, *b1;

    // 生成目的端口为 GENEVE_PORT，协议为 IPPROTO_UDP 的过滤器块
    b0 = gen_portfn(cstate, GENEVE_PORT, IPPROTO_UDP, Q_DST);

    /* 检查是否使用版本 0。否则无法解码其余字段。版本是 Geneve 头部的第一个字节中的 2 位。 */
    b1 = gen_mcmp(cstate, offrel, 8, BPF_B, 0, 0xc0);
    gen_and(b0, b1);
    b0 = b1;

    if (has_vni) {
        if (vni > 0xffffff) {
            bpf_error(cstate, "Geneve VNI %u 大于最大值 %u",
                vni, 0xffffff);
        }
        vni <<= 8; /* VNI 在前 3 个字节中 */
        b1 = gen_mcmp(cstate, offrel, 12, BPF_W, vni, 0xffffff00);
        gen_and(b0, b1);
        b0 = b1;
    }

    return b0;
}

/* IPv4 和 IPv6 的 Geneve 检查需要执行两件事：
 * - 验证是否为正确的 Geneve 协议，并且具有正确的 VNI。
 * - 将 IP 头长度（加上变长的链路前缀，如果需要）放入寄存器 A 中，以便稍后用于计算内部数据包的偏移量。 */
static struct block *
gen_geneve4(compiler_state_t *cstate, bpf_u_int32 vni, int has_vni)
{
    struct block *b0, *b1;
    struct slist *s, *s1;

    b0 = gen_geneve_check(cstate, gen_port, OR_TRAN_IPV4, vni, has_vni);

    /* 将 IP 头长度加载到寄存器 A 中。 */
    s = gen_loadx_iphdrlen(cstate);

    s1 = new_stmt(cstate, BPF_MISC|BPF_TXA);
    sappend(s, s1);

    /* 强制将这些语句追加到协议检查的真条件中，通过创建一个始终为真的新块，并进行 AND 运算。 */
    b1 = new_block(cstate, BPF_JMP|BPF_JEQ|BPF_X);
    b1->stmts = s;
    b1->s.k = 0;

    gen_and(b0, b1);

    return b1;
}

static struct block *
gen_geneve6(compiler_state_t *cstate, bpf_u_int32 vni, int has_vni)
{
    struct block *b0, *b1;
    struct slist *s, *s1;

    // 生成检查 Geneve 协议的代码块
    b0 = gen_geneve_check(cstate, gen_port6, OR_TRAN_IPV6, vni, has_vni);

    /* Load the IP header length. We need to account for a variable length link prefix if there is one. */
    // 加载 IP 头长度，需要考虑可变长度的链路前缀
    s = gen_abs_offset_varpart(cstate, &cstate->off_linkpl);
    if (s) {
        // 创建加载立即数的语句，加载 IP 头长度
        s1 = new_stmt(cstate, BPF_LD|BPF_IMM);
        s1->s.k = 40;
        // 将语句追加到 s 列表中
        sappend(s, s1);

        // 创建 ALU 加法运算语句，将 IP 头长度加上 0
        s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X);
        s1->s.k = 0;
        // 将语句追加到 s 列表中
        sappend(s, s1);
    } else {
        // 创建加载立即数的语句，加载 IP 头长度
        s = new_stmt(cstate, BPF_LD|BPF_IMM);
        s->s.k = 40;
    }

    /* Forcibly append these statements to the true condition of the protocol check by creating a new block that is always true and ANDing them. */
    // 强制将这些语句追加到协议检查的真条件中，通过创建一个始终为真的新块并对它们进行 AND 运算
    s1 = new_stmt(cstate, BPF_MISC|BPF_TAX);
    // 将语句追加到 s 列表中
    sappend(s, s1);

    // 创建一个新的块，用于存储 s 列表，并设置条件为 0
    b1 = new_block(cstate, BPF_JMP|BPF_JEQ|BPF_X);
    b1->stmts = s;
    b1->s.k = 0;

    // 生成并运算，将 b0 和 b1 进行 AND 运算
    gen_and(b0, b1);

    // 返回新生成的块
    return b1;
}

/* We need to store three values based on the Geneve header::
 * - The offset of the linktype.
 * - The offset of the end of the Geneve header.
 * - The offset of the end of the encapsulated MAC header. */
static struct slist *
gen_geneve_offsets(compiler_state_t *cstate)
{
    struct slist *s, *s1, *s_proto;

    /* First we need to calculate the offset of the Geneve header itself. This is composed of the IP header previously calculated (include any variable link prefix) and stored in A plus the fixed sized headers (fixed link prefix, MAC length, and UDP header). */
    // 首先需要计算 Geneve 头本身的偏移量。这由之前计算的 IP 头（包括任何可变链路前缀）和存储在 A 中的固定大小的头部（固定链路前缀、MAC 长度和 UDP 头）组成
    s = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s->s.k = cstate->off_linkpl.constant_part + cstate->off_nl + 8;

    // 将结果存储在 X 中，因为稍后会用到
    s1 = new_stmt(cstate, BPF_MISC|BPF_TAX);
    sappend(s, s1);

    // Geneve 中的 EtherType 在第 2 个字节中，计算并存储它
    s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    # 设置结构体 s1 的成员变量 s.k 为 2
    s1->s.k = 2;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 为 cstate 结构体的 off_linktype.reg 成员变量分配寄存器
    cstate->off_linktype.reg = alloc_reg(cstate);
    # 设置 cstate 结构体的 off_linktype.is_variable 成员变量为 1
    cstate->off_linktype.is_variable = 1;
    # 设置 cstate 结构体的 off_linktype.constant_part 成员变量为 0
    cstate->off_linktype.constant_part = 0;

    # 创建一个新的 BPF_ST 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ST);
    # 设置结构体 s1 的成员变量 s.k 为 cstate 结构体的 off_linktype.reg 成员变量
    s1->s.k = cstate->off_linktype.reg;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 创建一个新的 BPF_LD|BPF_IND|BPF_B 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_LD|BPF_IND|BPF_B);
    # 设置结构体 s1 的成员变量 s.k 为 0
    s1->s.k = 0;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 创建一个新的 BPF_ALU|BPF_AND|BPF_K 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ALU|BPF_AND|BPF_K);
    # 设置结构体 s1 的成员变量 s.k 为 0x3f
    s1->s.k = 0x3f;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 创建一个新的 BPF_ALU|BPF_MUL|BPF_K 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ALU|BPF_MUL|BPF_K);
    # 设置结构体 s1 的成员变量 s.k 为 4
    s1->s.k = 4;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 创建一个新的 BPF_ALU|BPF_ADD|BPF_K 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    # 设置结构体 s1 的成员变量 s.k 为 8
    s1->s.k = 8;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 创建一个新的 BPF_ALU|BPF_ADD|BPF_X 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_X);
    # 设置结构体 s1 的成员变量 s.k 为 0
    s1->s.k = 0;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 将 DLT_EN10MB 类型的链路头信息推入 cstate 结构体的链路头栈中
    PUSH_LINKHDR(cstate, DLT_EN10MB, 1, 0, alloc_reg(cstate));

    # 创建一个新的 BPF_ST 类型的语句结构体 s1
    s1 = new_stmt(cstate, BPF_ST);
    # 设置结构体 s1 的成员变量 s.k 为 cstate 结构体的 off_linkhdr.reg 成员变量
    s1->s.k = cstate->off_linkhdr.reg;
    # 将结构体 s1 添加到结构体 s 的末尾
    sappend(s, s1);

    # 设置 cstate 结构体的 no_optimize 成员变量为 1，表示不使用优化器
    cstate->no_optimize = 1;
    /* 在 Geneve 头部的第 2 个字节加载 EtherType */
    s1 = new_stmt(cstate, BPF_LD|BPF_IND|BPF_H);
    s1->s.k = 2;
    sappend(s, s1);

    /* 用 Geneve 头部的末尾加载 X */
    s1 = new_stmt(cstate, BPF_LDX|BPF_MEM);
    s1->s.k = cstate->off_linkhdr.reg;
    sappend(s, s1);

    /* 检查 EtherType 是否为 Transparent Ethernet Bridging。在此检查结束时，X 中应该有总长度。在非以太网情况下，它已经在那里。 */
    s_proto = new_stmt(cstate, JMP(BPF_JEQ));
    s_proto->s.k = ETHERTYPE_TEB;
    sappend(s, s_proto);

    s1 = new_stmt(cstate, BPF_MISC|BPF_TXA);
    sappend(s, s1);
    s_proto->s.jt = s1;

    /* 由于这是以太网，直接使用负载的 EtherType 作为链路类型。覆盖我们已经有的内容。 */
    s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s1->s.k = 12;
    sappend(s, s1);

    s1 = new_stmt(cstate, BPF_ST);
    s1->s.k = cstate->off_linktype.reg;
    sappend(s, s1);

    /* 进一步前进两个字节，以获取以太网头部的末尾。 */
    s1 = new_stmt(cstate, BPF_ALU|BPF_ADD|BPF_K);
    s1->s.k = 2;
    sappend(s, s1);

    /* 将结果移动到 X。 */
    s1 = new_stmt(cstate, BPF_MISC|BPF_TAX);
    sappend(s, s1);

    /* 存储我们的 linkpl 计算的最终结果。 */
    cstate->off_linkpl.reg = alloc_reg(cstate);
    cstate->off_linkpl.is_variable = 1;
    cstate->off_linkpl.constant_part = 0;

    s1 = new_stmt(cstate, BPF_STX);
    s1->s.k = cstate->off_linkpl.reg;
    sappend(s, s1);
    s_proto->s.jf = s1;

    cstate->off_nl = 0;

    return s;
}

/* 检查是否为 Geneve 数据包 */
struct block *
gen_geneve(compiler_state_t *cstate, bpf_u_int32 vni, int has_vni)
{
    struct block *b0, *b1;
    struct slist *s;

    /*
     * 捕获我们和下面的例程报告的错误，并在出现错误时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    // 生成 Geneve4 数据包过滤器
    b0 = gen_geneve4(cstate, vni, has_vni);
    // 生成 Geneve6 数据包过滤器
    b1 = gen_geneve6(cstate, vni, has_vni);

    // 对两个过滤器进行逻辑或操作
    gen_or(b0, b1);
    b0 = b1;

    /* 后续过滤器应该作用于 Geneve 帧的有效载荷，更新所有头指针。附加此代码以确保在 Geneve 过滤器匹配时执行。 */
    s = gen_geneve_offsets(cstate);

    // 生成一个始终为真的过滤器
    b1 = gen_true(cstate);
    // 将偏移量列表附加到 b1 的语句列表中
    sappend(s, b1->stmts);
    b1->stmts = s;

    // 对两个过滤器进行逻辑与操作
    gen_and(b0, b1);

    // 设置标志表示为 Geneve 数据包
    cstate->is_geneve = 1;

    return b1;
}

/* 检查封装帧是否具有以太网过滤器的链路层头部 */
static struct block *
gen_geneve_ll_check(compiler_state_t *cstate)
{
    struct block *b0;
    struct slist *s, *s1;

    /* 最简单的方法是检查链路层头部和有效载荷是否不同。 */

    /* Geneve 总是生成纯变量偏移量，因此我们只能比较寄存器。 */
    s = new_stmt(cstate, BPF_LD|BPF_MEM);
    s->s.k = cstate->off_linkhdr.reg;

    s1 = new_stmt(cstate, BPF_LDX|BPF_MEM);
    s1->s.k = cstate->off_linkpl.reg;
    sappend(s, s1);

    b0 = new_block(cstate, BPF_JMP|BPF_JEQ|BPF_X);
    b0->stmts = s;
    b0->s.k = 0;
    gen_not(b0);

    return b0;
}

static struct block *
gen_atmfield_code_internal(compiler_state_t *cstate, int atmfield,
    bpf_u_int32 jvalue, int jtype, int reverse)
{
    struct block *b0;

    switch (atmfield) {
    # 如果匹配到 A_VPI 情况
    case A_VPI:
        # 如果不是 ATM 原始数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'vpi' supported only on raw ATM");
        # 如果偏移量未设置，则中止程序
        if (cstate->off_vpi == OFFSET_NOT_SET)
            abort();
        # 生成比较指令，比较链路头部偏移量处的数据和给定值
        b0 = gen_ncmp(cstate, OR_LINKHDR, cstate->off_vpi, BPF_B,
            0xffffffffU, jtype, reverse, jvalue);
        # 跳出循环
        break;

    # 如果匹配到 A_VCI 情况
    case A_VCI:
        # 如果不是 ATM 原始数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'vci' supported only on raw ATM");
        # 如果偏移量未设置，则中止程序
        if (cstate->off_vci == OFFSET_NOT_SET)
            abort();
        # 生成比较指令，比较链路头部偏移量处的数据和给定值
        b0 = gen_ncmp(cstate, OR_LINKHDR, cstate->off_vci, BPF_H,
            0xffffffffU, jtype, reverse, jvalue);
        # 跳出循环
        break;

    # 如果匹配到 A_PROTOTYPE 情况
    case A_PROTOTYPE:
        # 如果偏移量未设置，则中止程序
        if (cstate->off_proto == OFFSET_NOT_SET)
            abort();    /* XXX - this isn't on FreeBSD */
        # 生成比较指令，比较链路头部偏移量处的数据和给定值
        b0 = gen_ncmp(cstate, OR_LINKHDR, cstate->off_proto, BPF_B,
            0x0fU, jtype, reverse, jvalue);
        # 跳出循环
        break;

    # 如果匹配到 A_MSGTYPE 情况
    case A_MSGTYPE:
        # 如果偏移量未设置，则中止程序
        if (cstate->off_payload == OFFSET_NOT_SET)
            abort();
        # 生成比较指令，比较链路头部偏移量加上消息类型位置处的数据和给定值
        b0 = gen_ncmp(cstate, OR_LINKHDR, cstate->off_payload + MSG_TYPE_POS, BPF_B,
            0xffffffffU, jtype, reverse, jvalue);
        # 跳出循环
        break;

    # 如果匹配到 A_CALLREFTYPE 情况
    case A_CALLREFTYPE:
        # 如果不是 ATM 原始数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'callref' supported only on raw ATM");
        # 如果偏移量未设置，则中止程序
        if (cstate->off_proto == OFFSET_NOT_SET)
            abort();
        # 生成比较指令，比较链路头部偏移量处的数据和给定值
        b0 = gen_ncmp(cstate, OR_LINKHDR, cstate->off_proto, BPF_B,
            0xffffffffU, jtype, reverse, jvalue);
        # 跳出循环
        break;

    # 默认情况下中止程序
    default:
        abort();
    }
    # 返回生成的比较指令
    return b0;
}

static struct block *
gen_atmtype_metac(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    // 生成一个用于匹配 A_VPI 字段为 0 的 BPF 代码块
    b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
    // 生成一个用于匹配 A_VCI 字段为 1 的 BPF 代码块
    b1 = gen_atmfield_code_internal(cstate, A_VCI, 1, BPF_JEQ, 0);
    // 生成一个逻辑与操作的 BPF 代码块
    gen_and(b0, b1);
    // 返回生成的 BPF 代码块
    return b1;
}

static struct block *
gen_atmtype_sc(compiler_state_t *cstate)
{
    struct block *b0, *b1;

    // 生成一个用于匹配 A_VPI 字段为 0 的 BPF 代码块
    b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
    // 生成一个用于匹配 A_VCI 字段为 5 的 BPF 代码块
    b1 = gen_atmfield_code_internal(cstate, A_VCI, 5, BPF_JEQ, 0);
    // 生成一个逻辑与操作的 BPF 代码块
    gen_and(b0, b1);
    // 返回生成的 BPF 代码块
    return b1;
}

static struct block *
gen_atmtype_llc(compiler_state_t *cstate)
{
    struct block *b0;

    // 生成一个用于匹配 A_PROTOTYPE 字段为 PT_LLC 的 BPF 代码块
    b0 = gen_atmfield_code_internal(cstate, A_PROTOTYPE, PT_LLC, BPF_JEQ, 0);
    // 设置当前链路类型为上一个链路类型
    cstate->linktype = cstate->prevlinktype;
    // 返回生成的 BPF 代码块
    return b0;
}

struct block *
gen_atmfield_code(compiler_state_t *cstate, int atmfield,
    bpf_u_int32 jvalue, int jtype, int reverse)
{
    /*
     * 捕获我们和下面的例程报告的错误，并在出错时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    // 调用内部函数生成 ATM 字段匹配的 BPF 代码块
    return gen_atmfield_code_internal(cstate, atmfield, jvalue, jtype,
        reverse);
}

struct block *
gen_atmtype_abbrev(compiler_state_t *cstate, int type)
{
    struct block *b0, *b1;

    /*
     * 捕获我们和下面的例程报告的错误，并在出错时返回 NULL。
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (type) {

    case A_METAC:
        /* 获取所有 Meta 信令电路中的数据包 */
        if (!cstate->is_atm)
            bpf_error(cstate, "'metac' supported only on raw ATM");
        // 生成用于匹配 Meta 信令电路的 BPF 代码块
        b1 = gen_atmtype_metac(cstate);
        break;
    # 如果是 A_BCC 情况
    case A_BCC:
        # 获取广播电路中的所有数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 格式，则报错
            bpf_error(cstate, "'bcc' supported only on raw ATM");
        # 生成 VPI 为 0，VCI 为 2 的 ATM 字段匹配代码
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 2, BPF_JEQ, 0);
        # 生成并操作，将两个条件合并
        gen_and(b0, b1);
        break;

    # 如果是 A_OAMF4SC 情况
    case A_OAMF4SC:
        # 获取段 OAM F4 电路中的所有数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 格式，则报错
            bpf_error(cstate, "'oam4sc' supported only on raw ATM");
        # 生成 VPI 为 0，VCI 为 3 的 ATM 字段匹配代码
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 3, BPF_JEQ, 0);
        # 生成并操作，将两个条件合并
        gen_and(b0, b1);
        break;

    # 如果是 A_OAMF4EC 情况
    case A_OAMF4EC:
        # 获取端到端 OAM F4 电路中的所有数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 格式，则报错
            bpf_error(cstate, "'oam4ec' supported only on raw ATM");
        # 生成 VPI 为 0，VCI 为 4 的 ATM 字段匹配代码
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 4, BPF_JEQ, 0);
        # 生成并操作，将两个条件合并
        gen_and(b0, b1);
        break;

    # 如果是 A_SC 情况
    case A_SC:
        # 获取连接信令电路中的所有数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 格式，则报错
            bpf_error(cstate, "'sc' supported only on raw ATM");
        # 生成连接信令电路的 ATM 类型匹配代码
        b1 = gen_atmtype_sc(cstate);
        break;

    # 如果是 A_ILMIC 情况
    case A_ILMIC:
        # 获取 ILMI 电路中的所有数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 格式，则报错
            bpf_error(cstate, "'ilmic' supported only on raw ATM");
        # 生成 VPI 为 0，VCI 为 16 的 ATM 字段匹配代码
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 16, BPF_JEQ, 0);
        # 生成并操作，将两个条件合并
        gen_and(b0, b1);
        break;
    # 如果是 A_LANE 情况
    case A_LANE:
        # 获取所有的 LANE 数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 原始数据，报错
            bpf_error(cstate, "'lane' supported only on raw ATM");
        # 生成用于匹配 LANE 协议的 BPF 代码
        b1 = gen_atmfield_code_internal(cstate, A_PROTOTYPE, PT_LANE, BPF_JEQ, 0);

        # 设置后续测试都假设是 LANE 封装而不是 LLC 封装的数据包，并设置适用于 LANE 封装以太网的偏移量
        # 我们假设 LANE 意味着以太网，而不是令牌环
        PUSH_LINKHDR(cstate, DLT_EN10MB, 0,
            cstate->off_payload + 2,    # 以太网头部
            -1);
        cstate->off_linktype.constant_part = cstate->off_linkhdr.constant_part + 12;
        cstate->off_linkpl.constant_part = cstate->off_linkhdr.constant_part + 14;    # 以太网
        cstate->off_nl = 0;            # 以太网 II
        cstate->off_nl_nosnap = 3;        # 802.3+802.2
        break;

    # 如果是 A_LLC 情况
    case A_LLC:
        # 获取所有 LLC 封装的数据包
        if (!cstate->is_atm)
            # 如果不是 ATM 原始数据，报错
            bpf_error(cstate, "'llc' supported only on raw ATM");
        # 生成用于匹配 LLC 类型的 BPF 代码
        b1 = gen_atmtype_llc(cstate);
        break;

    # 默认情况
    default:
        # 终止程序
        abort();
    }
    # 返回生成的 BPF 代码
    return b1;
}

/*
 * 根据 li 值过滤 MTP2 消息
 * FISU，长度为 null
 * LSSU，长度为 1 或 2
 * MSU，长度为 3 或更多
 * 对于 MTP2_HSL，序列占 2 个字节，长度占 9 位
 */
struct block *
gen_mtp2type_abbrev(compiler_state_t *cstate, int type)
{
    struct block *b0, *b1;

    /*
     * 捕获我们和下面的例程报告的错误，并在出错时返回 NULL
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (type) {

    case M_FISU:
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'fisu' supported only on MTP2");
        /* gen_ncmp(cstate, offrel, offset, size, mask, jtype, reverse, value) */
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li, BPF_B,
            0x3fU, BPF_JEQ, 0, 0U);
        break;

    case M_LSSU:
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'lssu' supported only on MTP2");
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li, BPF_B,
            0x3fU, BPF_JGT, 1, 2U);
        b1 = gen_ncmp(cstate, OR_PACKET, cstate->off_li, BPF_B,
            0x3fU, BPF_JGT, 0, 0U);
        gen_and(b1, b0);
        break;

    case M_MSU:
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'msu' supported only on MTP2");
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li, BPF_B,
            0x3fU, BPF_JGT, 0, 2U);
        break;
    # 如果消息类型是 MH_FISU
    case MH_FISU:
        # 如果链路类型不是 DLT_MTP2、DLT_ERF、DLT_MTP2_WITH_PHDR 中的任何一种，抛出错误
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'hfisu' supported only on MTP2_HSL");
        # 生成一个比较指令，用于比较偏移量处的数据和给定的值
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li_hsl, BPF_H,
            0xff80U, BPF_JEQ, 0, 0U);
        # 结束当前的 case
        break;

    # 如果消息类型是 MH_LSSU
    case MH_LSSU:
        # 如果链路类型不是 DLT_MTP2、DLT_ERF、DLT_MTP2_WITH_PHDR 中的任何一种，抛出错误
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'hlssu' supported only on MTP2_HSL");
        # 生成一个比较指令，用于比较偏移量处的数据和给定的值
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li_hsl, BPF_H,
            0xff80U, BPF_JGT, 1, 0x0100U);
        # 生成一个比较指令，用于比较偏移量处的数据和给定的值
        b1 = gen_ncmp(cstate, OR_PACKET, cstate->off_li_hsl, BPF_H,
            0xff80U, BPF_JGT, 0, 0U);
        # 生成一个与操作指令
        gen_and(b1, b0);
        # 结束当前的 case
        break;

    # 如果消息类型是 MH_MSU
    case MH_MSU:
        # 如果链路类型不是 DLT_MTP2、DLT_ERF、DLT_MTP2_WITH_PHDR 中的任何一种，抛出错误
        if ( (cstate->linktype != DLT_MTP2) &&
             (cstate->linktype != DLT_ERF) &&
             (cstate->linktype != DLT_MTP2_WITH_PHDR) )
            bpf_error(cstate, "'hmsu' supported only on MTP2_HSL");
        # 生成一个比较指令，用于比较偏移量处的数据和给定的值
        b0 = gen_ncmp(cstate, OR_PACKET, cstate->off_li_hsl, BPF_H,
            0xff80U, BPF_JGT, 0, 0x0100U);
        # 结束当前的 case
        break;

    # 如果消息类型不是 MH_FISU、MH_LSSU、MH_MSU 中的任何一种，终止程序
    default:
        abort();
    }
    # 返回生成的比较指令
    return b0;
}
/*
 * The jvalue_arg dance is to avoid annoying whining by compilers that
 * jvalue might be clobbered by longjmp - yeah, it might, but *WHO CARES*?
 * It's not *used* after setjmp returns.
 */
// 为了避免编译器因为 jvalue 可能被 longjmp 覆盖而发出烦人的警告，使用 jvalue_arg 进行处理，尽管它可能会被覆盖，但在 setjmp 返回后就不会再被使用了
struct block *
gen_mtp3field_code(compiler_state_t *cstate, int mtp3field,
    bpf_u_int32 jvalue_arg, int jtype, int reverse)
{
    volatile bpf_u_int32 jvalue = jvalue_arg; // 使用 volatile 修饰 jvalue，以避免编译器优化
    struct block *b0;
    bpf_u_int32 val1 , val2 , val3;
    u_int newoff_sio;
    u_int newoff_opc;
    u_int newoff_dpc;
    u_int newoff_sls;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    // 捕获我们和下面例程报告的错误，并在出错时返回 NULL
    if (setjmp(cstate->top_ctx))
        return (NULL);

    newoff_sio = cstate->off_sio;
    newoff_opc = cstate->off_opc;
    newoff_dpc = cstate->off_dpc;
    newoff_sls = cstate->off_sls;
    switch (mtp3field) {

    case MH_SIO:
        newoff_sio += 3; /* offset for MTP2_HSL */
        /* FALLTHROUGH */

    case M_SIO:
        if (cstate->off_sio == OFFSET_NOT_SET)
            bpf_error(cstate, "'sio' supported only on SS7");
        // sio 只支持在 SS7 上
        /* sio coded on 1 byte so max value 255 */
        // sio 编码在 1 字节上，所以最大值为 255
        if(jvalue > 255)
                bpf_error(cstate, "sio value %u too big; max value = 255",
                    jvalue);
        b0 = gen_ncmp(cstate, OR_PACKET, newoff_sio, BPF_B, 0xffffffffU,
            jtype, reverse, jvalue);
        break;
    # 如果是 MH_OPC 操作码，则偏移量增加 3
    case MH_OPC:
        newoff_opc += 3;

        /* FALLTHROUGH */
        # 如果是 M_OPC 操作码
        case M_OPC:
            # 如果 cstate->off_opc 未设置偏移量，则报错
            if (cstate->off_opc == OFFSET_NOT_SET)
            bpf_error(cstate, "'opc' supported only on SS7");
        /* opc coded on 14 bits so max value 16383 */
        # 如果 jvalue 大于 16383，则报错
        if (jvalue > 16383)
                bpf_error(cstate, "opc value %u too big; max value = 16383",
                    jvalue);
        # 下面的指令用于将 jvalue 转换为写入 SS7 消息中 opc 的形式
        val1 = jvalue & 0x00003c00;
        val1 = val1 >>10;
        val2 = jvalue & 0x000003fc;
        val2 = val2 <<6;
        val3 = jvalue & 0x00000003;
        val3 = val3 <<22;
        jvalue = val1 + val2 + val3;
        b0 = gen_ncmp(cstate, OR_PACKET, newoff_opc, BPF_W, 0x00c0ff0fU,
            jtype, reverse, jvalue);
        break;

    # 如果是 MH_DPC 操作码
    case MH_DPC:
        newoff_dpc += 3;
        /* FALLTHROUGH */

    # 如果是 M_DPC 操作码
    case M_DPC:
            # 如果 cstate->off_dpc 未设置偏移量，则报错
            if (cstate->off_dpc == OFFSET_NOT_SET)
            bpf_error(cstate, "'dpc' supported only on SS7");
        /* dpc coded on 14 bits so max value 16383 */
        # 如果 jvalue 大于 16383，则报错
        if (jvalue > 16383)
                bpf_error(cstate, "dpc value %u too big; max value = 16383",
                    jvalue);
        # 下面的指令用于将 jvalue 转换为写入 SS7 消息中 dpc 的形式
        val1 = jvalue & 0x000000ff;
        val1 = val1 << 24;
        val2 = jvalue & 0x00003f00;
        val2 = val2 << 8;
        jvalue = val1 + val2;
        b0 = gen_ncmp(cstate, OR_PACKET, newoff_dpc, BPF_W, 0xff3f0000U,
            jtype, reverse, jvalue);
        break;

    # 如果是 MH_SLS 操作码
    case MH_SLS:
        newoff_sls += 3;
        /* FALLTHROUGH */
    # 根据不同的情况进行处理
    case M_SLS:
            # 如果偏移量未设置，则在 SS7 上不支持'sls'
            if (cstate->off_sls == OFFSET_NOT_SET)
            bpf_error(cstate, "'sls' supported only on SS7");
        # sls 编码为 4 位，所以最大值为 15
        if (jvalue > 15)
                 bpf_error(cstate, "sls value %u too big; max value = 15",
                     jvalue);
        # 以下指令用于将 jvalue 转换为在 SS7 消息中写入 sls 的形式
        jvalue = jvalue << 4;
        b0 = gen_ncmp(cstate, OR_PACKET, newoff_sls, BPF_B, 0xf0U,
            jtype, reverse, jvalue);
        break;

    # 默认情况下，终止程序
    default:
        abort();
    }
    # 返回结果
    return b0;
}

static struct block *
gen_msg_abbrev(compiler_state_t *cstate, int type)
{
    struct block *b1;

    /*
     * Q.2931 signalling protocol messages for handling virtual circuits
     * establishment and teardown
     */
    switch (type) {

    case A_SETUP:
        // 生成用于匹配 A_SETUP 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, SETUP, BPF_JEQ, 0);
        break;

    case A_CALLPROCEED:
        // 生成用于匹配 A_CALLPROCEED 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, CALL_PROCEED, BPF_JEQ, 0);
        break;

    case A_CONNECT:
        // 生成用于匹配 A_CONNECT 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, CONNECT, BPF_JEQ, 0);
        break;

    case A_CONNECTACK:
        // 生成用于匹配 A_CONNECTACK 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, CONNECT_ACK, BPF_JEQ, 0);
        break;

    case A_RELEASE:
        // 生成用于匹配 A_RELEASE 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, RELEASE, BPF_JEQ, 0);
        break;

    case A_RELEASE_DONE:
        // 生成用于匹配 A_RELEASE_DONE 消息类型的 BPF 代码块
        b1 = gen_atmfield_code_internal(cstate, A_MSGTYPE, RELEASE_DONE, BPF_JEQ, 0);
        break;

    default:
        // 如果类型不匹配，则终止程序
        abort();
    }
    return b1;
}

struct block *
gen_atmmulti_abbrev(compiler_state_t *cstate, int type)
{
    struct block *b0, *b1;

    /*
     * Catch errors reported by us and routines below us, and return NULL
     * on an error.
     */
    if (setjmp(cstate->top_ctx))
        return (NULL);

    switch (type) {

    case A_OAM:
        if (!cstate->is_atm)
            // 如果不是原始 ATM 数据，则报错
            bpf_error(cstate, "'oam' supported only on raw ATM");
        /* OAM F4 type */
        // 生成用于匹配 A_OAM 消息类型的 BPF 代码块
        b0 = gen_atmfield_code_internal(cstate, A_VCI, 3, BPF_JEQ, 0);
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 4, BPF_JEQ, 0);
        // 将两个代码块进行逻辑或操作
        gen_or(b0, b1);
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        // 将两个代码块进行逻辑与操作
        gen_and(b0, b1);
        break;
    # 如果是 A_OAMF4 类型的情况
    case A_OAMF4:
        # 如果不是 ATM 数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'oamf4' supported only on raw ATM");
        # 生成比特码，用于匹配 VCI 字段为 3 的情况
        b0 = gen_atmfield_code_internal(cstate, A_VCI, 3, BPF_JEQ, 0);
        # 生成比特码，用于匹配 VCI 字段为 4 的情况
        b1 = gen_atmfield_code_internal(cstate, A_VCI, 4, BPF_JEQ, 0);
        # 生成或逻辑操作的比特码
        gen_or(b0, b1);
        # 生成比特码，用于匹配 VPI 字段为 0 的情况
        b0 = gen_atmfield_code_internal(cstate, A_VPI, 0, BPF_JEQ, 0);
        # 生成与逻辑操作的比特码
        gen_and(b0, b1);
        # 结束当前情况的处理
        break;

    # 如果是 A_CONNECTMSG 类型的情况
    case A_CONNECTMSG:
        # 如果不是 ATM 数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'connectmsg' supported only on raw ATM");
        # 生成 Q.2931 签名消息的简称比特码
        b0 = gen_msg_abbrev(cstate, A_SETUP);
        b1 = gen_msg_abbrev(cstate, A_CALLPROCEED);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_CONNECT);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_CONNECTACK);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_RELEASE);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_RELEASE_DONE);
        gen_or(b0, b1);
        b0 = gen_atmtype_sc(cstate);
        gen_and(b0, b1);
        # 结束当前情况的处理
        break;

    # 如果是 A_METACONNECT 类型的情况
    case A_METACONNECT:
        # 如果不是 ATM 数据流，则报错
        if (!cstate->is_atm)
            bpf_error(cstate, "'metaconnect' supported only on raw ATM");
        # 生成 Q.2931 签名消息的简称比特码
        b0 = gen_msg_abbrev(cstate, A_SETUP);
        b1 = gen_msg_abbrev(cstate, A_CALLPROCEED);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_CONNECT);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_RELEASE);
        gen_or(b0, b1);
        b0 = gen_msg_abbrev(cstate, A_RELEASE_DONE);
        gen_or(b0, b1);
        b0 = gen_atmtype_metac(cstate);
        gen_and(b0, b1);
        # 结束当前情况的处理
        break;

    # 如果是其他情况
    default:
        # 中止程序
        abort();
    }
    # 返回比特码 b1
    return b1;
# 闭合前面的函数定义
```