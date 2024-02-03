# `nmap\libdnet-stripped\include\dnet\icmp.h`

```cpp
# 定义 ICMP 头部长度常量
#define ICMP_HDR_LEN    4    /* base ICMP header length */
# 定义 ICMP 消息的最小长度常量，包括头部
#define ICMP_LEN_MIN    8    /* minimum ICMP message size, with header */

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

# ICMP 头部结构体
struct icmp_hdr {
    uint8_t        icmp_type;    /* 消息类型，参见下面的链接 */
    uint8_t        icmp_code;    /* 类型子码 */
    uint16_t    icmp_cksum;    /* 结构体的校验和的反码 */
};

# ICMP 消息类型和代码
# http://www.iana.org/assignments/icmp-parameters
#define        ICMP_CODE_NONE        0    /* 无代码的类型 */
#define    ICMP_ECHOREPLY        0        /* 回显应答 */
#define    ICMP_UNREACH        3        /* 目的地不可达，代码如下： */
#define        ICMP_UNREACH_NET        0    /* 错误的网络 */
#define        ICMP_UNREACH_HOST        1    /* 错误的主机 */
#define        ICMP_UNREACH_PROTO        2    /* 错误的协议 */
#define        ICMP_UNREACH_PORT        3    /* 错误的端口 */
#define        ICMP_UNREACH_NEEDFRAG        4    /* IP_DF 导致丢包 */
#define        ICMP_UNREACH_SRCFAIL        5    /* 源路由失败 */
#define        ICMP_UNREACH_NET_UNKNOWN    6    /* 未知网络 */
#define        ICMP_UNREACH_HOST_UNKNOWN    7    /* 未知主机 */
#define        ICMP_UNREACH_ISOLATED        8    /* 源主机被隔离 */
#define        ICMP_UNREACH_NET_PROHIB        9    /* 用于加密设备 */
#define        ICMP_UNREACH_HOST_PROHIB    10    /* 同上 */
#define        ICMP_UNREACH_TOSNET        11    /* 网络的 TOS 错误 */
#define        ICMP_UNREACH_TOSHOST        12    /* 主机的 TOS 错误 */
#define        ICMP_UNREACH_FILTER_PROHIB    13    /* 禁止访问 */
# 定义 ICMP 报文类型和代码的常量
#define        ICMP_UNREACH_HOST_PRECEDENCE    14    /* precedence error */
#define        ICMP_UNREACH_PRECEDENCE_CUTOFF    15    /* precedence cutoff */
#define    ICMP_SRCQUENCH        4        /* packet lost, slow down */
#define    ICMP_REDIRECT        5        /* shorter route, codes: */
#define        ICMP_REDIRECT_NET        0    /* for network */
#define        ICMP_REDIRECT_HOST        1    /* for host */
#define        ICMP_REDIRECT_TOSNET        2    /* for tos and net */
#define        ICMP_REDIRECT_TOSHOST        3    /* for tos and host */
#define    ICMP_ALTHOSTADDR    6        /* alternate host address */
#define    ICMP_ECHO        8        /* echo service */
#define    ICMP_RTRADVERT        9        /* router advertise, codes: */
#define        ICMP_RTRADVERT_NORMAL        0    /* normal */
#define        ICMP_RTRADVERT_NOROUTE_COMMON 16    /* selective routing */
#define    ICMP_RTRSOLICIT        10        /* router solicitation */
#define    ICMP_TIMEXCEED        11        /* time exceeded, code: */
#define        ICMP_TIMEXCEED_INTRANS        0    /* ttl==0 in transit */
#define        ICMP_TIMEXCEED_REASS        1    /* ttl==0 in reass */
#define    ICMP_PARAMPROB        12        /* ip header bad */
#define        ICMP_PARAMPROB_ERRATPTR        0    /* req. opt. absent */
#define        ICMP_PARAMPROB_OPTABSENT    1    /* req. opt. absent */
#define        ICMP_PARAMPROB_LENGTH        2    /* bad length */
#define    ICMP_TSTAMP        13        /* timestamp request */
#define    ICMP_TSTAMPREPLY    14        /* timestamp reply */
#define    ICMP_INFO        15        /* information request */
#define    ICMP_INFOREPLY        16        /* information reply */
#define    ICMP_MASK        17        /* address mask request */
#define    ICMP_MASKREPLY        18        /* address mask reply */
#define ICMP_TRACEROUTE        30        /* traceroute */
#define ICMP_DATACONVERR    31        /* data conversion error */
# 定义 ICMP 消息类型和对应的编号
#define ICMP_MOBILE_REDIRECT    32        /* 移动主机重定向 */
#define ICMP_IPV6_WHEREAREYOU    33        /* IPv6 where-are-you */
#define ICMP_IPV6_IAMHERE    34        /* IPv6 i-am-here */
#define ICMP_MOBILE_REG        35        /* 移动注册请求 */
#define ICMP_MOBILE_REGREPLY    36        /* 移动注册回复 */
#define ICMP_DNS        37        /* 域名请求 */
#define ICMP_DNSREPLY        38        /* 域名回复 */
#define ICMP_SKIP        39        /* SKIP */
#define ICMP_PHOTURIS        40        /* Photuris */
#define        ICMP_PHOTURIS_UNKNOWN_INDEX    0    /* 未知的安全索引 */
#define        ICMP_PHOTURIS_AUTH_FAILED    1    /* 认证失败 */
#define        ICMP_PHOTURIS_DECOMPRESS_FAILED    2    /* 解压失败 */
#define        ICMP_PHOTURIS_DECRYPT_FAILED    3    /* 解密失败 */
#define        ICMP_PHOTURIS_NEED_AUTHN    4    /* 无需认证 */
#define        ICMP_PHOTURIS_NEED_AUTHZ    5    /* 无需授权 */
#define    ICMP_TYPE_MAX        40    # ICMP 消息类型的最大值

# 定义 ICMP_INFOTYPE 宏，用于判断是否为信息类型的 ICMP 消息
#define    ICMP_INFOTYPE(type)                        \
    ((type) == ICMP_ECHOREPLY || (type) == ICMP_ECHO ||        \
    (type) == ICMP_RTRADVERT || (type) == ICMP_RTRSOLICIT ||    \
    (type) == ICMP_TSTAMP || (type) == ICMP_TSTAMPREPLY ||        \
    (type) == ICMP_INFO || (type) == ICMP_INFOREPLY ||        \
    (type) == ICMP_MASK || (type) == ICMP_MASKREPLY)

# 定义 ICMP 消息中的 Echo 数据结构
/*
 * Echo message data
 */
struct icmp_msg_echo {
    uint16_t    icmp_id;    # ICMP 标识符
    uint16_t    icmp_seq;    # ICMP 序列号
    uint8_t        icmp_data __flexarr;    /* 可选数据 */
};

# 定义 ICMP 消息中的 Fragmentation-needed 数据结构
/*
 * Fragmentation-needed (unreachable) message data
 */
struct icmp_msg_needfrag {
    uint16_t    icmp_void;        /* 必须为零 */
    uint16_t    icmp_mtu;        /* 下一跳网络的 MTU */
    uint8_t        icmp_ip __flexarr;    /* IP 头 + 8 字节数据包 */
};

# 定义 ICMP 消息中的 Unreachable, source quench, redirect, time exceeded, parameter problem 数据结构
struct icmp_msg_quote {
    # 定义一个名为icmp_void的32位无符号整数变量，必须为零
    uint32_t    icmp_void;        /* must be zero */
#define icmp_gwaddr    icmp_void        /* 要使用的路由器 IP 地址 */
#define icmp_pptr    icmp_void        /* 指向错误八位字段的指针 */
    uint8_t        icmp_ip __flexarr;    /* IP 头部 + 8 字节的数据包 */
};

/*
 * 路由器通告消息数据，RFC 1256
 */
struct icmp_msg_rtradvert {
    uint8_t        icmp_num_addrs;        /* 地址/前缀对的数量 */
    uint8_t        icmp_wpa;        /* 每个地址的字数 == 2 */
    uint16_t    icmp_lifetime;        /* 路由寿命（以秒为单位） */
    struct icmp_msg_rtr_data {
        uint32_t    icmp_void;
#define icmp_gwaddr        icmp_void    /* 路由器 IP 地址 */
        uint32_t    icmp_pref;    /* 路由器偏好（通常为 0） */
    } icmp_rtr __flexarr;            /* 可变数量的路由器 */
};
#define ICMP_RTR_PREF_NODEFAULT    0x80000000    /* 不要用作默认网关 */

/*
 * 时间戳消息数据
 */
struct icmp_msg_tstamp {
    uint32_t    icmp_id;        /* 标识符 */
    uint32_t    icmp_seq;        /* 序列号 */
    uint32_t    icmp_ts_orig;        /* 发起时间戳 */
    uint32_t    icmp_ts_rx;        /* 接收时间戳 */
    uint32_t    icmp_ts_tx;        /* 传输时间戳 */
};

/*
 * 地址掩码消息数据，RFC 950
 */
struct icmp_msg_mask {
    uint32_t    icmp_id;        /* 标识符 */
    uint32_t    icmp_seq;        /* 序列号 */
    uint32_t    icmp_mask;        /* 地址掩码 */
};

/*
 * Traceroute 消息数据，RFC 1393, RFC 1812
 */
struct icmp_msg_traceroute {
    uint16_t    icmp_id;        /* 标识符 */
    uint16_t    icmp_void;        /* 未使用 */
    uint16_t    icmp_ohc;        /* 出站跳数 */
    uint16_t    icmp_rhc;        /* 返回跳数 */
    uint32_t    icmp_speed;        /* 链路速度，字节/秒 */
    uint32_t    icmp_mtu;        /* 字节单位的 MTU */
};

/*
 * 域名回复消息数据，RFC 1788
 */
struct icmp_msg_dnsreply {
    uint16_t    icmp_id;        /* 标识符 */
    # 16位的无符号整数，表示 ICMP 报文的序列号
    uint16_t    icmp_seq;        /* sequence number */
    # 32位的无符号整数，表示 ICMP 报文的生存时间
    uint32_t    icmp_ttl;        /* time-to-live */
    # 可变数量的名称数组，用于存储 ICMP 报文中的名称
    uint8_t        icmp_names __flexarr;    /* variable number of names */
/*
 * 结构体，包含 ICMP 消息的通用标识符和序列号数据
 */
struct icmp_msg_idseq {
    uint16_t    icmp_id;    // ICMP 标识符
    uint16_t    icmp_seq;    // ICMP 序列号
};

/*
 * ICMP 消息的联合体
 */
union icmp_msg {
    struct icmp_msg_echo       echo;    // ICMP_ECHO{REPLY}
    struct icmp_msg_quote       unreach;    // ICMP_UNREACH
    struct icmp_msg_needfrag   needfrag;    // ICMP_UNREACH_NEEDFRAG
    struct icmp_msg_quote       srcquench;    // ICMP_SRCQUENCH
    struct icmp_msg_quote       redirect;    // ICMP_REDIRECT (set to 0)
    uint32_t           rtrsolicit;    // ICMP_RTRSOLICIT
    struct icmp_msg_rtradvert  rtradvert;    // ICMP_RTRADVERT
    struct icmp_msg_quote       timexceed;    // ICMP_TIMEXCEED
    struct icmp_msg_quote       paramprob;    // ICMP_PARAMPROB
    struct icmp_msg_tstamp       tstamp;    // ICMP_TSTAMP{REPLY}
    struct icmp_msg_idseq       info;    // ICMP_INFO{REPLY}
    struct icmp_msg_mask       mask;    // ICMP_MASK{REPLY}
    struct icmp_msg_traceroute traceroute;    // ICMP_TRACEROUTE
    struct icmp_msg_idseq       dns;        // ICMP_DNS
    struct icmp_msg_dnsreply   dnsreply;    // ICMP_DNSREPLY
};

#ifndef __GNUC__
# pragma pack()
#endif

#define icmp_pack_hdr(hdr, type, code) do {                \
    struct icmp_hdr *icmp_pack_p = (struct icmp_hdr *)(hdr);    \
    icmp_pack_p->icmp_type = type; icmp_pack_p->icmp_code = code;    \
} while (0)

#define icmp_pack_hdr_echo(hdr, type, code, id, seq, data, len) do {    \
    struct icmp_msg_echo *echo_pack_p = (struct icmp_msg_echo *)    \
        ((uint8_t *)(hdr) + ICMP_HDR_LEN);            \
    icmp_pack_hdr(hdr, type, code);                    \
    echo_pack_p->icmp_id = htons(id);                \
    echo_pack_p->icmp_seq = htons(seq);                \
    memmove(echo_pack_p->icmp_data, data, len);            \
} while (0)

#define icmp_pack_hdr_quote(hdr, type, code, word, pkt, len) do {    \
    # 将指针 quote_pack_p 指向 ICMP 报文中报文引用部分的起始位置
    struct icmp_msg_quote *quote_pack_p = (struct icmp_msg_quote *)    \
        ((uint8_t *)(hdr) + ICMP_HDR_LEN);            \
    # 调用 icmp_pack_hdr 函数，设置 ICMP 报文的头部信息
    icmp_pack_hdr(hdr, type, code);                    \
    # 将 word 转换为网络字节序，并赋值给 quote_pack_p->icmp_void
    quote_pack_p->icmp_void = htonl(word);                \
    # 将数据包 pkt 中的内容拷贝到 quote_pack_p->icmp_ip 中，长度为 len
    memmove(quote_pack_p->icmp_ip, pkt, len);            \
#define icmp_pack_hdr_mask(hdr, type, code, id, seq, mask) do {        \
    // 定义一个宏，用于填充 ICMP 报文头部和 mask 数据
    struct icmp_msg_mask *mask_pack_p = (struct icmp_msg_mask *)    \
        ((uint8_t *)(hdr) + ICMP_HDR_LEN);            \
    // 调用 icmp_pack_hdr 函数填充 ICMP 报文头部
    icmp_pack_hdr(hdr, type, code);                    \
    // 设置 mask_pack_p 结构体中的字段值
    mask_pack_p->icmp_id = htons(id);                \
    mask_pack_p->icmp_seq = htons(seq);                \
    mask_pack_p->icmp_mask = htonl(mask);                \
} while (0)

#define icmp_pack_hdr_needfrag(hdr, type, code, mtu, pkt, len) do {    \
    // 定义一个宏，用于填充 ICMP 报文头部和 needfrag 数据
    struct icmp_msg_needfrag *frag_pack_p =                \
    (struct icmp_msg_needfrag *)((uint8_t *)(hdr) + ICMP_HDR_LEN);    \
    // 调用 icmp_pack_hdr 函数填充 ICMP 报文头部
    icmp_pack_hdr(hdr, type, code);                    \
    // 设置 frag_pack_p 结构体中的字段值
    frag_pack_p->icmp_void = 0;                    \
    frag_pack_p->icmp_mtu = htons(mtu);                \
    // 将 pkt 中的数据拷贝到 frag_pack_p 结构体中的 icmp_ip 字段
    memmove(frag_pack_p->icmp_ip, pkt, len);            \
} while (0)

#endif /* DNET_ICMP_H */
```