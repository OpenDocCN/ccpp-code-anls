# `nmap\libdnet-stripped\include\dnet\sctp.h`

```cpp
/*
 * sctp.h
 *
 * Stream Control Transmission Protocol (RFC 4960).
 *
 * Copyright (c) 2008-2009 Daniel Roethlisberger <daniel@roe.ch>
 *
 * $Id: sctp.h 653 2009-07-05 21:00:00Z daniel@roe.ch $
 */

#ifndef DNET_SCTP_H
#define DNET_SCTP_H

#ifndef __GNUC__
# ifndef __attribute__
#  define __attribute__(x)
# endif
# pragma pack(1)
#endif

#define SCTP_HDR_LEN    12

// 定义 SCTP 头部结构体
struct sctp_hdr {
    uint16_t    sh_sport;    /* source port */  // 源端口
    uint16_t    sh_dport;    /* destination port */  // 目的端口
    uint32_t    sh_vtag;    /* sctp verification tag */  // SCTP 验证标签
    uint32_t    sh_sum;        /* sctp checksum */  // SCTP 校验和
} __attribute__((__packed__));

#define SCTP_PORT_MAX    65535

// 定义宏，用于打包 SCTP 头部
#define sctp_pack_hdr(hdr, sport, dport, vtag) do {            \
    struct sctp_hdr *sctp_pack_p = (struct sctp_hdr *)(hdr);    \
    sctp_pack_p->sh_sport = htons(sport);                \
    sctp_pack_p->sh_dport = htons(dport);                \
    sctp_pack_p->sh_vtag = htonl(vtag);                \
} while (0)

// 定义 SCTP chunk 头部结构体
struct dnet_sctp_chunkhdr {
    uint8_t        sch_type;    /* chunk type */  // chunk 类型
    uint8_t        sch_flags;    /* chunk flags */  // chunk 标志
    uint16_t    sch_length;    /* chunk length */  // chunk 长度
} __attribute__((__packed__));

// 定义各种 chunk 类型的常量
/* chunk types */
#define SCTP_DATA        0x00
#define SCTP_INIT        0x01
#define SCTP_INIT_ACK        0x02
#define SCTP_SACK        0x03
#define SCTP_HEARTBEAT        0x04
#define SCTP_HEARTBEAT_ACK    0x05
#define SCTP_ABORT        0x06
#define SCTP_SHUTDOWN        0x07
#define SCTP_SHUTDOWN_ACK    0x08
#define SCTP_ERROR        0x09
#define SCTP_COOKIE_ECHO    0x0a
#define SCTP_COOKIE_ACK        0x0b
#define SCTP_ECNE        0x0c
#define SCTP_CWR        0x0d
#define SCTP_SHUTDOWN_COMPLETE    0x0e
#define SCTP_AUTH        0x0f    /* RFC 4895 */
#define SCTP_ASCONF_ACK        0x80    /* RFC 5061 */
#define SCTP_PKTDROP        0x81    /* draft-stewart-sctp-pktdrprep-08 */
#define SCTP_PAD        0x84    /* RFC 4820 */
#define SCTP_FORWARD_TSN    0xc0    /* RFC 3758 */
#define SCTP_ASCONF        0xc1    /* RFC 5061 */

/* chunk types bitmask flags */
#define SCTP_TYPEFLAG_REPORT    1
#define SCTP_TYPEFLAG_SKIP    2

#define sctp_pack_chunkhdr(hdr, type, flags, length) do {        \
    struct dnet_sctp_chunkhdr *sctp_pack_chp = (struct dnet_sctp_chunkhdr *)(hdr);\
    sctp_pack_chp->sch_type = type;                    \  # 将传入的 type 值赋给 sctp_pack_chp 结构体的 sch_type 成员
    sctp_pack_chp->sch_flags = flags;                \  # 将传入的 flags 值赋给 sctp_pack_chp 结构体的 sch_flags 成员
    sctp_pack_chp->sch_length = htons(length);            \  # 将传入的 length 值转换为网络字节顺序后赋给 sctp_pack_chp 结构体的 sch_length 成员
} while (0)

/*
 * INIT chunk
 */
struct sctp_chunkhdr_init {
    struct dnet_sctp_chunkhdr chunkhdr;

    uint32_t    schi_itag;    /* Initiate Tag */  \  # 初始化标签
    uint32_t    schi_arwnd;    /* Advertised Receiver Window Credit */  \  # 广告接收窗口信用
    uint16_t    schi_nos;    /* Number of Outbound Streams */  \  # 出站流的数量
    uint16_t    schi_nis;    /* Number of Inbound Streams */  \  # 入站流的数量
    uint32_t    schi_itsn;    /* Initial TSN */  \  # 初始 TSN
} __attribute__((__packed__));

#define sctp_pack_chunkhdr_init(hdr, type, flags, length, itag,        \
                arwnd, nos, nis, itsn) do {        \
    struct sctp_chunkhdr_init *sctp_pack_chip =            \
            (struct sctp_chunkhdr_init *)(hdr);        \  # 将传入的 hdr 强制类型转换为 sctp_chunkhdr_init 结构体指针
    sctp_pack_chunkhdr(sctp_pack_chip, type, flags, length);    \  # 调用 sctp_pack_chunkhdr 函数，传入参数并打包 chunk 头部
    sctp_pack_chip->schi_itag = htonl(itag);            \  # 将传入的 itag 值转换为网络字节顺序后赋给 sctp_pack_chip 结构体的 schi_itag 成员
    sctp_pack_chip->schi_arwnd = htonl(arwnd);            \  # 将传入的 arwnd 值转换为网络字节顺序后赋给 sctp_pack_chip 结构体的 schi_arwnd 成员
    sctp_pack_chip->schi_nos = htons(nos);                \  # 将传入的 nos 值转换为网络字节顺序后赋给 sctp_pack_chip 结构体的 schi_nos 成员
    sctp_pack_chip->schi_nis = htons(nis);                \  # 将传入的 nis 值转换为网络字节顺序后赋给 sctp_pack_chip 结构体的 schi_nis 成员
    sctp_pack_chip->schi_itsn = htonl(itsn);            \  # 将传入的 itsn 值转换为网络字节顺序后赋给 sctp_pack_chip 结构体的 schi_itsn 成员
} while (0)

/*
 * INIT ACK chunk
 */
struct sctp_chunkhdr_init_ack {
    struct dnet_sctp_chunkhdr chunkhdr;

    uint32_t    schia_itag;    /* Initiate Tag */  \  # 初始化标签
    uint32_t    schia_arwnd;    /* Advertised Receiver Window Credit */  \  # 广告接收窗口信用
    uint16_t    schia_nos;    /* Number of Outbound Streams */  \  # 出站流的数量
    uint16_t    schia_nis;    /* Number of Inbound Streams */  \  # 入站流的数量
    uint32_t    schia_itsn;    /* Initial TSN */  \  # 初始 TSN
} __attribute__((__packed__));
#define sctp_pack_chunkhdr_init_ack(hdr, type, flags, length, itag,    \
                arwnd, nos, nis, itsn) do {        \
    // 初始化 SCTP INIT ACK 头部
    struct sctp_chunkhdr_init_ack *sctp_pack_chip =            \
            (struct sctp_chunkhdr_init_ack *)(hdr);        \
    // 调用通用的打包头部函数，设置类型、标志和长度
    sctp_pack_chunkhdr(sctp_pack_chip, type, flags, length);    \
    // 设置初始化标记
    sctp_pack_chip->schia_itag = htonl(itag);            \
    // 设置接收窗口大小
    sctp_pack_chip->schia_arwnd = htonl(arwnd);            \
    // 设置发送序列号
    sctp_pack_chip->schia_nos = htons(nos);                \
    // 设置接收序列号
    sctp_pack_chip->schia_nis = htons(nis);                \
    // 设置初始 TSN
    sctp_pack_chip->schia_itsn = htonl(itsn);            \
} while (0)

/*
 * ABORT chunk
 */
struct sctp_chunkhdr_abort {
    struct dnet_sctp_chunkhdr chunkhdr;

    /* empty */
} __attribute__((__packed__));

#define sctp_pack_chunkhdr_abort(hdr, type, flags, length) do {        \
    // 初始化 SCTP ABORT 头部
    struct sctp_chunkhdr_abort *sctp_pack_chip =            \
            (struct sctp_chunkhdr_abort *)(hdr);        \
    // 调用通用的打包头部函数，设置类型、标志和长度
    sctp_pack_chunkhdr(sctp_pack_chip, type, flags, length);    \
} while (0)

/*
 * SHUTDOWN ACK chunk
 */
struct sctp_chunkhdr_shutdown_ack {
    struct dnet_sctp_chunkhdr chunkhdr;

    /* empty */
} __attribute__((__packed__));

#define sctp_pack_chunkhdr_shutdown_ack(hdr, type, flags, length) do {    \
    // 初始化 SCTP SHUTDOWN ACK 头部
    struct sctp_chunkhdr_shutdown_ack *sctp_pack_chip =        \
            (struct sctp_chunkhdr_shutdown_ack *)(hdr);    \
    // 调用通用的打包头部函数，设置类型、标志和长度
    sctp_pack_chunkhdr(sctp_pack_chip, type, flags, length);    \
} while (0)

/*
 * COOKIE ECHO chunk
 */
struct sctp_chunkhdr_cookie_echo {
    struct dnet_sctp_chunkhdr chunkhdr;

    /* empty */
} __attribute__((__packed__));

#define sctp_pack_chunkhdr_cookie_echo(hdr, type, flags, length) do {    \
    // 初始化 SCTP COOKIE ECHO 头部
    struct sctp_chunkhdr_cookie_echo *sctp_pack_chip =        \
            (struct sctp_chunkhdr_cookie_echo *)(hdr);    \
    // 调用通用的打包头部函数，设置类型、标志和长度
    sctp_pack_chunkhdr(sctp_pack_chip, type, flags, length);    \
} while (0)

#ifndef __GNUC__
# pragma pack()
#endif

#endif /* DNET_SCTP_H */
```