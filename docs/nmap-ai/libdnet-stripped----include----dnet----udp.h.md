# `nmap\libdnet-stripped\include\dnet\udp.h`

```
/*
 * udp.h
 *
 * User Datagram Protocol (RFC 768).
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: udp.h 330 2002-04-02 05:05:39Z dugsong $
 */

#ifndef DNET_UDP_H
#define DNET_UDP_H

#define UDP_HDR_LEN    8

// 定义 UDP 头部结构体
struct udp_hdr {
    uint16_t    uh_sport;    /* source port */  // 源端口
    uint16_t    uh_dport;    /* destination port */  // 目的端口
    uint16_t    uh_ulen;    /* udp length (including header) */  // UDP 长度（包括头部）
    uint16_t    uh_sum;        /* udp checksum */  // UDP 校验和
};

#define UDP_PORT_MAX    65535

// 封装 UDP 头部
#define udp_pack_hdr(hdr, sport, dport, ulen) do {        \
    struct udp_hdr *udp_pack_p = (struct udp_hdr *)(hdr);    \
    udp_pack_p->uh_sport = htons(sport);            \
    udp_pack_p->uh_dport = htons(dport);            \
    udp_pack_p->uh_ulen = htons(ulen);            \
} while (0)

#endif /* DNET_UDP_H */
```