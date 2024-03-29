# `nmap\libdnet-stripped\include\dnet\arp.h`

```cpp
/*
 * arp.h
 * 
 * Address Resolution Protocol.
 * RFC 826
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp.h 416 2003-03-16 17:39:18Z dugsong $
 */

#ifndef DNET_ARP_H
#define DNET_ARP_H

#define ARP_HDR_LEN    8    /* base ARP header length */
#define ARP_ETHIP_LEN    20    /* base ARP message length */

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * ARP header
 */
struct arp_hdr {
    uint16_t    ar_hrd;    /* format of hardware address */
    uint16_t    ar_pro;    /* format of protocol address */
    uint8_t        ar_hln;    /* length of hardware address (ETH_ADDR_LEN) */
    uint8_t        ar_pln;    /* length of protocol address (IP_ADDR_LEN) */
    uint16_t    ar_op;    /* operation */
};

/*
 * Hardware address format
 */
#define ARP_HRD_ETH     0x0001    /* ethernet hardware */
#define ARP_HRD_IEEE802    0x0006    /* IEEE 802 hardware */

#define ARP_HRD_INFINIBAND 0x0020 /* InfiniBand */
#define ARP_HRD_APPLETALK 0x0309 /* AppleTalk DDP */
#define ARP_HDR_IEEE80211 0x0321  /* IEEE 802.11 */
#define ARP_HRD_IEEE80211_PRISM 0x0322  /* IEEE 802.11 + prism header */
#define ARP_HRD_IEEE80211_RADIOTAP 0x0323  /* IEEE 802.11 + radiotap header */
#define ARP_HRD_VOID 0xFFFF            /* Void type, nothing is known */

/*
 * Protocol address format
 */
#define ARP_PRO_IP    0x0800    /* IP protocol */

/*
 * ARP operation
 */
#define    ARP_OP_REQUEST        1    /* request to resolve ha given pa */
#define    ARP_OP_REPLY        2    /* response giving hardware address */
#define    ARP_OP_REVREQUEST    3    /* request to resolve pa given ha */
#define    ARP_OP_REVREPLY        4    /* response giving protocol address */

/*
 * Ethernet/IP ARP message
 */
struct arp_ethip {
    uint8_t        ar_sha[ETH_ADDR_LEN];    /* sender hardware address */
    uint8_t        ar_spa[IP_ADDR_LEN];    /* sender protocol address */
    # 定义一个名为ar_tha的无符号8位整数数组，用于存储目标硬件地址
    uint8_t        ar_tha[ETH_ADDR_LEN];    /* target hardware address */
    # 定义一个名为ar_tpa的无符号8位整数数组，用于存储目标协议地址
    uint8_t        ar_tpa[IP_ADDR_LEN];    /* target protocol address */
};

/*
 * ARP cache entry
 */
// 定义 ARP 缓存条目结构体
struct arp_entry {
    struct addr    arp_pa;            /* protocol address */
    struct addr    arp_ha;            /* hardware address */
};

#ifndef __GNUC__
# pragma pack()
#endif

// 定义宏，用于打包以太网和 IP 地址的 ARP 头部
#define arp_pack_hdr_ethip(hdr, op, sha, spa, tha, tpa) do {    \
    struct arp_hdr *pack_arp_p = (struct arp_hdr *)(hdr);    \
    struct arp_ethip *pack_ethip_p = (struct arp_ethip *)    \
        ((uint8_t *)(hdr) + ARP_HDR_LEN);        \
    pack_arp_p->ar_hrd = htons(ARP_HRD_ETH);        \
    pack_arp_p->ar_pro = htons(ARP_PRO_IP);            \
    pack_arp_p->ar_hln = ETH_ADDR_LEN;            \
    pack_arp_p->ar_pln = IP_ADDR_LEN;            \
    pack_arp_p->ar_op = htons(op);                \
    memmove(pack_ethip_p->ar_sha, &(sha), ETH_ADDR_LEN);    \
    memmove(pack_ethip_p->ar_spa, &(spa), IP_ADDR_LEN);    \
    memmove(pack_ethip_p->ar_tha, &(tha), ETH_ADDR_LEN);    \
    memmove(pack_ethip_p->ar_tpa, &(tpa), IP_ADDR_LEN);    \
} while (0)

// 定义 ARP 处理器结构
typedef struct arp_handle arp_t;

// 定义 ARP 处理器的回调函数
typedef int (*arp_handler)(const struct arp_entry *entry, void *arg);

__BEGIN_DECLS
// 打开 ARP 处理器
arp_t    *arp_open(void);
// 添加 ARP 条目
int     arp_add(arp_t *arp, const struct arp_entry *entry);
// 删除 ARP 条目
int     arp_delete(arp_t *arp, const struct arp_entry *entry);
// 获取 ARP 条目
int     arp_get(arp_t *arp, struct arp_entry *entry);
// 循环遍历 ARP 条目并调用回调函数
int     arp_loop(arp_t *arp, arp_handler callback, void *arg);
// 关闭 ARP 处理器
arp_t    *arp_close(arp_t *arp);
__END_DECLS

#endif /* DNET_ARP_H */
```