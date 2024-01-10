# `nmap\libdnet-stripped\include\dnet\ip6.h`

```
# 定义 IPv6 地址长度
#define IP6_ADDR_LEN    16
# 定义 IPv6 地址位数
#define IP6_ADDR_BITS    128

# 定义 IPv6 头部长度
#define IP6_HDR_LEN    40        /* IPv6 header length */
# 定义 IPv6 最小长度
#define IP6_LEN_MIN    IP6_HDR_LEN
# 定义 IPv6 最大长度
#define IP6_LEN_MAX    65535        /* non-jumbo payload */

# 定义 IPv6 最小传输单元
#define IP6_MTU_MIN    1280        /* minimum MTU (1024 + 256) */

# 定义 IPv6 地址结构
typedef struct ip6_addr {
    uint8_t         data[IP6_ADDR_LEN];
} ip6_addr_t;

# 如果不是 GCC 编译器，则定义 __attribute__ 为空
#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

# IPv6 头部结构
struct ip6_hdr {
    union {
        struct ip6_hdr_ctl {
            uint32_t    ip6_un1_flow; /* 20 bits of flow ID */
            uint16_t    ip6_un1_plen; /* payload length */
            uint8_t        ip6_un1_nxt;  /* next header */
            uint8_t        ip6_un1_hlim; /* hop limit */
        } ip6_un1;
        uint8_t    ip6_un2_vfc;    /* 4 bits version, top 4 bits class */
    } ip6_ctlun;
    ip6_addr_t    ip6_src;
    ip6_addr_t    ip6_dst;
} __attribute__((__packed__));

# 定义 IPv6 版本
#define ip6_vfc        ip6_ctlun.ip6_un2_vfc
# 定义 IPv6 流标识
#define ip6_flow    ip6_ctlun.ip6_un1.ip6_un1_flow
# 定义 IPv6 负载长度
#define ip6_plen    ip6_ctlun.ip6_un1.ip6_un1_plen
# 定义 IPv6 下一个头部
#define ip6_nxt        ip6_ctlun.ip6_un1.ip6_un1_nxt    /* IP_PROTO_* */
# 定义 IPv6 跳数限制
#define ip6_hlim    ip6_ctlun.ip6_un1.ip6_un1_hlim

# 定义 IPv6 版本号
#define IP6_VERSION        0x60
# 定义 IPv6 版本掩码
#define IP6_VERSION_MASK    0xf0        /* ip6_vfc version */

# 根据字节序定义 IPv6 流信息掩码和标签掩码
#if DNET_BYTESEX == DNET_BIG_ENDIAN
#define IP6_FLOWINFO_MASK    0x0fffffff    /* ip6_flow info (28 bits) */
#define IP6_FLOWLABEL_MASK    0x000fffff    /* ip6_flow label (20 bits) */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
#define IP6_FLOWINFO_MASK    0xffffff0f    /* ip6_flow info (28 bits) */
#define IP6_FLOWLABEL_MASK    0xffff0f00    /* ip6_flow label (20 bits) */
#endif

# 定义 IPv6 跳数限制的默认值
#define IP6_HLIM_DEFAULT    64
#define IP6_HLIM_MAX        255
// 定义 IPv6 最大生存时间为 255

/*
 * Preferred extension header order from RFC 2460, 4.1:
 *
 * IP_PROTO_IPV6, IP_PROTO_HOPOPTS, IP_PROTO_DSTOPTS, IP_PROTO_ROUTING,
 * IP_PROTO_FRAGMENT, IP_PROTO_AH, IP_PROTO_ESP, IP_PROTO_DSTOPTS, IP_PROTO_*
 */
// 从 RFC 2460, 4.1 中获取首选的扩展头顺序

/*
 * Routing header data (IP_PROTO_ROUTING)
 */
// 路由头数据（IP_PROTO_ROUTING）
struct ip6_ext_data_routing {
    uint8_t  type;            /* routing type */
    uint8_t  segleft;        /* segments left */
    /* followed by routing type specific data */
} __attribute__((__packed__));

struct ip6_ext_data_routing0 {
    uint8_t  type;            /* always zero */
    uint8_t  segleft;        /* segments left */
    uint8_t  reserved;        /* reserved field */
    uint8_t  slmap[3];        /* strict/loose bit map */
    ip6_addr_t  addr[1];        /* up to 23 addresses */
} __attribute__((__packed__));

/*
 * Fragment header data (IP_PROTO_FRAGMENT)
 */
// 片段头数据（IP_PROTO_FRAGMENT）
struct ip6_ext_data_fragment {
    uint16_t  offlg;        /* offset, reserved, and flag */
    uint32_t  ident;        /* identification */
} __attribute__((__packed__));

/*
 * Fragmentation offset, reserved, and flags (offlg)
 */
// 分段偏移、保留和标志（offlg）
#if DNET_BYTESEX == DNET_BIG_ENDIAN
#define IP6_OFF_MASK        0xfff8    /* mask out offset from offlg */
#define IP6_RESERVED_MASK    0x0006    /* reserved bits in offlg */
#define IP6_MORE_FRAG        0x0001    /* more-fragments flag */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
#define IP6_OFF_MASK        0xf8ff    /* mask out offset from offlg */
#define IP6_RESERVED_MASK    0x0600    /* reserved bits in offlg */
#define IP6_MORE_FRAG        0x0100    /* more-fragments flag */
#endif

/*
 * Option types, for IP_PROTO_HOPOPTS, IP_PROTO_DSTOPTS headers
 */
// 选项类型，用于 IP_PROTO_HOPOPTS、IP_PROTO_DSTOPTS 头部
#define IP6_OPT_PAD1        0x00    /* 00 0 00000 */
#define IP6_OPT_PADN        0x01    /* 00 0 00001 */
#define IP6_OPT_JUMBO        0xC2    /* 11 0 00010 = 194 */
#define IP6_OPT_JUMBO_LEN    6
#define IP6_OPT_RTALERT        0x05    /* 00 0 00101 */
#define IP6_OPT_RTALERT_LEN    4
# 定义 IPv6 选项类型常量
#define IP6_OPT_RTALERT_MLD    0    /* 数据报包含 MLD 消息 */
#define IP6_OPT_RTALERT_RSVP    1    /* 数据报包含 RSVP 消息 */
#define IP6_OPT_RTALERT_ACTNET    2     /* 包含 Active Networks 消息 */
#define IP6_OPT_LEN_MIN        2

# 定义 IPv6 选项类型宏
#define IP6_OPT_TYPE(o)        ((o) & 0xC0)    /* opt_type 的高 2 位 */
#define IP6_OPT_TYPE_SKIP    0x00    /* 失败时继续处理 */
#define IP6_OPT_TYPE_DISCARD    0x40    /* 失败时丢弃数据包 */
#define IP6_OPT_TYPE_FORCEICMP    0x80    /* 失败时丢弃并发送 ICMP 数据包 */
#define IP6_OPT_TYPE_ICMP    0xC0    /* ...仅当目的地非多播时 */

# 定义可变选项标志
#define IP6_OPT_MUTABLE        0x20    /* 选项数据可能在传输过程中改变 */

# 定义 IPv6 扩展头结构体
/*
 * Extension header (chained via {ip6,ext}_nxt, following IPv6 header)
 */
struct ip6_ext_hdr {
    uint8_t  ext_nxt;    /* 下一个头部 */
    uint8_t  ext_len;    /* 后续长度，以 8 个字节为单位 */
    union {
        struct ip6_ext_data_routing    routing;
        struct ip6_ext_data_fragment    fragment;
    } ext_data;
} __attribute__((__packed__));

# 定义保留地址
/*
 * Reserved addresses
 */
#define IP6_ADDR_UNSPEC    \
    "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
#define IP6_ADDR_LOOPBACK \
    "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01"

# 定义 IPv6 头部打包宏
#define ip6_pack_hdr(hdr, fc, fl, plen, nxt, hlim, src, dst) do {    \
    struct ip6_hdr *ip6 = (struct ip6_hdr *)(hdr);            \
    ip6->ip6_flow = htonl(((uint32_t)(fc) << 20) |            \
        (0x000fffff & (fl)));                    \
    ip6->ip6_vfc = (IP6_VERSION | ((fc) >> 4));            \
    ip6->ip6_plen = htons((plen));                    \
    ip6->ip6_nxt = (nxt); ip6->ip6_hlim = (hlim);            \
    memmove(&ip6->ip6_src, &(src), IP6_ADDR_LEN);            \
    memmove(&ip6->ip6_dst, &(dst), IP6_ADDR_LEN);            \
} while (0);

# 开始 C 语言声明
__BEGIN_DECLS
# 定义一个函数，将 IPv6 地址转换为可读的字符串格式，并存储到目标地址中，返回目标地址
char    *ip6_ntop(const ip6_addr_t *ip6, char *dst, size_t size);

# 定义一个函数，将字符串格式的 IPv6 地址转换为二进制格式，并存储到目标地址中，返回转换结果
int     ip6_pton(const char *src, ip6_addr_t *dst);

# 定义一个函数，将 IPv6 地址转换为点分十进制的字符串格式，并返回结果
char    *ip6_ntoa(const ip6_addr_t *ip6);

# 定义一个宏，将点分十进制的字符串格式的 IPv6 地址转换为二进制格式，并存储到目标地址中，返回转换结果
#define     ip6_aton ip6_pton

# 定义一个函数，计算 IPv6 数据包的校验和
void     ip6_checksum(void *buf, size_t len);

# 结束 C 语言的声明
__END_DECLS

# 结束 if 条件编译，结束 IPv6 相关的头文件声明
#endif /* DNET_IP6_H */
```