# `nmap\libdnet-stripped\include\dnet\icmpv6.h`

```
/*
 * icmpv6.h
 *
 * ICMPv6.
 * RFC 4443
 *
 * $Id: $
 */

#ifndef DNET_ICMPV6_H
#define DNET_ICMPV6_H

#define ICMPV6_HDR_LEN    4    /* base ICMPv6 header length */

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * ICMPv6 header
 */
struct icmpv6_hdr {
    uint8_t        icmpv6_type;    /* type of message, see below */
    uint8_t        icmpv6_code;    /* type sub code */
    uint16_t    icmpv6_cksum;    /* ones complement cksum of struct */
};

/*
 * Types (icmpv6_type) and codes (icmpv6_code) -
 * http://www.iana.org/assignments/icmpv6-parameters
 */
#define        ICMPV6_CODE_NONE    0        /* for types without codes */
#define ICMPV6_UNREACH        1        /* dest unreachable, codes: */
#define        ICMPV6_UNREACH_NOROUTE        0    /* no route to dest */
#define        ICMPV6_UNREACH_PROHIB        1    /* admin prohibited */
#define        ICMPV6_UNREACH_SCOPE        2    /* beyond scope of source address */
#define        ICMPV6_UNREACH_ADDR        3    /* address unreach */
#define        ICMPV6_UNREACH_PORT        4    /* port unreach */
#define        ICMPV6_UNREACH_FILTER_PROHIB    5    /* src failed ingress/egress policy */
#define        ICMPV6_UNREACH_REJECT_ROUTE    6    /* reject route */
#define ICMPV6_TIMEXCEED    3        /* time exceeded, code: */
#define        ICMPV6_TIMEXCEED_INTRANS    0    /* hop limit exceeded in transit */
#define        ICMPV6_TIMEXCEED_REASS        1    /* fragmetn reassembly time exceeded */
#define ICMPV6_PARAMPROBLEM    4        /* parameter problem, code: */
#define     ICMPV6_PARAMPROBLEM_FIELD    0    /* erroneous header field encountered */
#define     ICMPV6_PARAMPROBLEM_NEXTHEADER    1    /* unrecognized Next Header type encountered */
#define     ICMPV6_PARAMPROBLEM_OPTION    2    /* unrecognized IPv6 option encountered */
#define ICMPV6_ECHO        128        /* echo request */
#define ICMPV6_ECHOREPLY    129        /* echo reply */
/*
 * Neighbor discovery types (RFC 4861)
 */
#define    ICMPV6_NEIGHBOR_SOLICITATION    135  // 定义邻居请求消息类型的值
#define    ICMPV6_NEIGHBOR_ADVERTISEMENT    136  // 定义邻居通告消息类型的值

#define    ICMPV6_INFOTYPE(type) (((type) & 0x80) != 0)  // 判断消息类型是否为信息类型

/*
 * Echo message data
 */
struct icmpv6_msg_echo {
    uint16_t    icmpv6_id;  // ICMPv6消息的ID
    uint16_t    icmpv6_seq;  // ICMPv6消息的序列号
    uint8_t        icmpv6_data __flexarr;    /* optional data */  // 可选数据
};

/* Neighbor solicitation or advertisement (single hardcoded option).
   RFC 4861, sections 4.3 and 4.4. */
struct icmpv6_msg_nd {
    uint32_t    icmpv6_flags;  // ICMPv6消息的标志
    ip6_addr_t    icmpv6_target;  // ICMPv6消息的目标地址
    uint8_t        icmpv6_option_type;  // ICMPv6消息的选项类型
    uint8_t        icmpv6_option_length;  // ICMPv6消息的选项长度
    eth_addr_t    icmpv6_mac;  // ICMPv6消息的MAC地址
};

/*
 * ICMPv6 message union
 */
union icmpv6_msg {
    struct icmpv6_msg_echo       echo;    /* ICMPV6_ECHO{REPLY} */  // ICMPv6回显消息
    struct icmpv6_msg_nd       nd;        /* ICMPV6_NEIGHBOR_{SOLICITATION,ADVERTISEMENT} */  // ICMPv6邻居请求或通告消息
};

#ifndef __GNUC__
# pragma pack()
#endif

#define icmpv6_pack_hdr(hdr, type, code) do {                \
    struct icmpv6_hdr *icmpv6_pack_p = (struct icmpv6_hdr *)(hdr);    \
    icmpv6_pack_p->icmpv6_type = type; icmpv6_pack_p->icmpv6_code = code;    \
} while (0)  // 封装ICMPv6消息头部

#define icmpv6_pack_hdr_echo(hdr, type, code, id, seq, data, len) do {    \
    struct icmpv6_msg_echo *echo_pack_p = (struct icmpv6_msg_echo *)\
        ((uint8_t *)(hdr) + ICMPV6_HDR_LEN);            \
    icmpv6_pack_hdr(hdr, type, code);                \
    echo_pack_p->icmpv6_id = htons(id);                \
    echo_pack_p->icmpv6_seq = htons(seq);                \
    memmove(echo_pack_p->icmpv6_data, data, len);            \
} while (0)  // 封装ICMPv6回显消息头部

#define icmpv6_pack_hdr_ns_mac(hdr, targetip, srcmac) do {        \
    struct icmpv6_msg_nd *nd_pack_p = (struct icmpv6_msg_nd *)    \
        ((uint8_t *)(hdr) + ICMPV6_HDR_LEN);            \
    icmpv6_pack_hdr(hdr, ICMPV6_NEIGHBOR_SOLICITATION, 0);        \
    nd_pack_p->icmpv6_flags = 0;                    \
    # 将目标 IP 地址的内容复制到 ICMPv6 数据包的目标地址字段中
    memmove(&nd_pack_p->icmpv6_target, &(targetip), IP6_ADDR_LEN);    \
    # 设置 ICMPv6 数据包的选项类型为 1
    nd_pack_p->icmpv6_option_type = 1;                \
    # 设置 ICMPv6 数据包的选项长度为 1
    nd_pack_p->icmpv6_option_length = 1;                \
    # 将源 MAC 地址的内容复制到 ICMPv6 数据包的 MAC 地址字段中
    memmove(&nd_pack_p->icmpv6_mac, &(srcmac), ETH_ADDR_LEN);    \
} while (0)  # 使用 do-while 循环，确保至少执行一次循环体
#endif /* DNET_ICMPV6_H */  # 结束条件编译指令，关闭 ICMPV6 头文件的定义
```