# `nmap\mswin32\NET\if_arp.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学校董会所有
 * 允许以源代码或二进制形式重新分发和使用，无论是否经过修改
 * 需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由加利福尼亚大学伯克利分校及其贡献者开发的软件
 * 4. 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品
 *
 * 本软件由校董会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，校董会或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性的损害承担责任
 * （包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）
 * 无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害
 *
 *    @(#)if_arp.h    8.1 (Berkeley) 6/10/93
 * $FreeBSD: src/sys/net/if_arp.h,v 1.10.2.2 1999/08/29 16:28:15 peter Exp $
 */

#ifndef _NET_IF_ARP_H_
#define    _NET_IF_ARP_H_

/*
 * Address Resolution Protocol.
 *
 * See RFC 826 for protocol description.  ARP packets are variable
 * in size; the arphdr structure defines the fixed-length portion.
 * Protocol type values are the same as those for 10 Mb/s Ethernet.
 * It is followed by the variable-sized fields ar_sha, arp_spa,
 * arp_tha and arp_tpa in that order, according to the lengths
 * specified.  Field names used correspond to RFC 826.
 */
struct    arphdr {
    u_short    ar_hrd;        /* format of hardware address */
#define ARPHRD_ETHER     1    /* ethernet hardware format */
#define ARPHRD_IEEE802    6    /* token-ring hardware format */
#define ARPHRD_FRELAY     15    /* frame relay hardware format */
    u_short    ar_pro;        /* format of protocol address */
    u_char    ar_hln;        /* length of hardware address */
    u_char    ar_pln;        /* length of protocol address */
    u_short    ar_op;        /* one of: */
#define    ARPOP_REQUEST    1    /* request to resolve address */
#define    ARPOP_REPLY    2    /* response to previous request */
#define    ARPOP_REVREQUEST 3    /* request protocol address given hardware */
#define    ARPOP_REVREPLY    4    /* response giving protocol address */
#define ARPOP_INVREQUEST 8     /* request to identify peer */
#define ARPOP_INVREPLY    9    /* response identifying peer */
/*
 * The remaining fields are variable in size,
 * according to the sizes above.
 */
#ifdef COMMENT_ONLY
    u_char    ar_sha[];    /* sender hardware address */
    u_char    ar_spa[];    /* sender protocol address */
    u_char    ar_tha[];    /* target hardware address */
    u_char    ar_tpa[];    /* target protocol address */
#endif
};

/*
 * ARP ioctl request
 */
struct arpreq {
    struct    sockaddr arp_pa;        /* protocol address */
    struct    sockaddr arp_ha;        /* hardware address */
    int    arp_flags;            /* flags */
};
/*  arp_flags and at_flags field values */
#define    ATF_INUSE    0x01    /* entry in use */
#define ATF_COM        0x02    /* completed entry (enaddr valid) */
#define    ATF_PERM    0x04    /* permanent entry */
#define    ATF_PUBL    0x08    /* publish entry (respond for other host) */
#define    ATF_USETRAILERS    0x10    /* has requested trailers */

#ifdef KERNEL
/*
 * Structure shared between the ethernet driver modules and
 * the address resolution code.  For example, each ec_softc or il_softc
 * begins with this structure.
 */
struct    arpcom {
    /*
     * The ifnet struct _must_ be at the head of this structure.
     */
    struct     ifnet ac_if;        /* network-visible interface */
    u_char    ac_enaddr[6];        /* ethernet hardware address */
    int    ac_multicnt;        /* length of ac_multiaddrs list */
};

extern u_char    etherbroadcastaddr[6];
#endif

#endif /* !_NET_IF_ARP_H_ */
```