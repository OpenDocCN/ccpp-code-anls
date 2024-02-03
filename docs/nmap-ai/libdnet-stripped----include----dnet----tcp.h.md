# `nmap\libdnet-stripped\include\dnet\tcp.h`

```cpp
/*
 * tcp.h
 *
 * Transmission Control Protocol (RFC 793).
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: tcp.h 487 2004-02-23 10:02:11Z dugsong $
 */

#ifndef DNET_TCP_H
#define DNET_TCP_H

#define TCP_HDR_LEN    20        /* base TCP header length */
#define TCP_OPT_LEN    2        /* base TCP option length */
#define TCP_OPT_LEN_MAX    40
#define TCP_HDR_LEN_MAX    (TCP_HDR_LEN + TCP_OPT_LEN_MAX)

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * TCP header, without options
 */
struct tcp_hdr {
    uint16_t    th_sport;    /* source port */
    uint16_t    th_dport;    /* destination port */
    uint32_t    th_seq;        /* sequence number */
    uint32_t    th_ack;        /* acknowledgment number */
#if DNET_BYTESEX == DNET_BIG_ENDIAN
    uint8_t        th_off:4,    /* data offset */
            th_x2:4;    /* (unused) */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
    uint8_t        th_x2:4,
            th_off:4;
#else
# error "need to include <dnet.h>"
#endif
    uint8_t        th_flags;    /* control flags */
    uint16_t    th_win;        /* window */
    uint16_t    th_sum;        /* checksum */
    uint16_t    th_urp;        /* urgent pointer */
};

/*
 * TCP control flags (th_flags)
 */
#define TH_FIN        0x01        /* end of data */
#define TH_SYN        0x02        /* synchronize sequence numbers */
#define TH_RST        0x04        /* reset connection */
#define TH_PUSH        0x08        /* push */
#define TH_ACK        0x10        /* acknowledgment number set */
#define TH_URG        0x20        /* urgent pointer set */
#define TH_ECE        0x40        /* ECN echo, RFC 3168 */
#define TH_CWR        0x80        /* congestion window reduced */

#define TCP_PORT_MAX    65535        /* maximum port */
#define TCP_WIN_MAX    65535        /* maximum (unscaled) window */

/*
 * Sequence number comparison macros
 */
#define TCP_SEQ_LT(a,b)        ((int)((a)-(b)) < 0)
# 定义比较 TCP 序列号的宏，判断序列号 a 是否小于等于序列号 b
#define TCP_SEQ_LEQ(a,b)    ((int)((a)-(b)) <= 0)
# 定义比较 TCP 序列号的宏，判断序列号 a 是否大于序列号 b
#define TCP_SEQ_GT(a,b)        ((int)((a)-(b)) > 0)
# 定义比较 TCP 序列号的宏，判断序列号 a 是否大于等于序列号 b
#define TCP_SEQ_GEQ(a,b)    ((int)((a)-(b)) >= 0)

/*
 * TCP FSM states
 */
# 定义 TCP 状态常量，表示连接已关闭
#define TCP_STATE_CLOSED    0    /* closed */
# 定义 TCP 状态常量，表示正在监听连接
#define TCP_STATE_LISTEN    1    /* listening from connection */
# 定义 TCP 状态常量，表示主动连接，已发送 SYN
#define TCP_STATE_SYN_SENT    2    /* active, have sent SYN */
# 定义 TCP 状态常量，表示已发送和接收 SYN
#define TCP_STATE_SYN_RECEIVED    3    /* have sent and received SYN */

# 定义 TCP 状态常量，表示连接已建立
#define TCP_STATE_ESTABLISHED    4    /* established */
# 定义 TCP 状态常量，表示接收到 FIN，等待关闭
#define TCP_STATE_CLOSE_WAIT    5    /* rcvd FIN, waiting for close */

# 定义 TCP 状态常量，表示已关闭，已发送 FIN
#define TCP_STATE_FIN_WAIT_1    6    /* have closed, sent FIN */
# 定义 TCP 状态常量，表示关闭交换 FIN，等待 FIN-ACK
#define TCP_STATE_CLOSING    7    /* closed xchd FIN, await FIN-ACK */
# 定义 TCP 状态常量，表示已发送 FIN 和关闭，等待 FIN-ACK
#define TCP_STATE_LAST_ACK    8    /* had FIN and close, await FIN-ACK */

# 定义 TCP 状态常量，表示已关闭，FIN 已确认
#define TCP_STATE_FIN_WAIT_2    9    /* have closed, FIN is acked */
# 定义 TCP 状态常量，表示关闭后的 2*MSL 安静等待
#define TCP_STATE_TIME_WAIT    10    /* in 2*MSL quiet wait after close */
# 定义 TCP 状态常量的最大值
#define TCP_STATE_MAX        11

/*
 * Options (opt_type) - http://www.iana.org/assignments/tcp-parameters
 */
# 定义 TCP 选项常量，表示选项列表结束
#define TCP_OPT_EOL        0    /* end of option list */
# 定义 TCP 选项常量，表示无操作
#define TCP_OPT_NOP        1    /* no operation */
# 定义 TCP 选项常量，表示最大段大小
#define TCP_OPT_MSS        2    /* maximum segment size */
# 定义 TCP 选项常量，表示窗口缩放因子，RFC 1072
#define TCP_OPT_WSCALE        3    /* window scale factor, RFC 1072 */
# 定义 TCP 选项常量，表示允许 SACK，RFC 2018
#define TCP_OPT_SACKOK        4    /* SACK permitted, RFC 2018 */
# 定义 TCP 选项常量，表示 SACK，RFC 2018
#define TCP_OPT_SACK        5    /* SACK, RFC 2018 */
# 定义 TCP 选项常量，表示回显（已过时），RFC 1072
#define TCP_OPT_ECHO        6    /* echo (obsolete), RFC 1072 */
# 定义 TCP 选项常量，表示回显回复（已过时），RFC 1072
#define TCP_OPT_ECHOREPLY    7    /* echo reply (obsolete), RFC 1072 */
# 定义 TCP 选项常量，表示时间戳，RFC 1323
#define TCP_OPT_TIMESTAMP    8    /* timestamp, RFC 1323 */
# 定义 TCP 选项常量，表示部分顺序连接，RFC 1693
#define TCP_OPT_POCONN        9    /* partial order conn, RFC 1693 */
# 定义 TCP 选项常量，表示部分顺序服务，RFC 1693
#define TCP_OPT_POSVC        10    /* partial order service, RFC 1693 */
# 定义 TCP 选项常量，表示连接计数，RFC 1644
#define TCP_OPT_CC        11    /* connection count, RFC 1644 */
# 定义 TCP 选项常量，表示 CC.NEW，RFC 1644
#define TCP_OPT_CCNEW        12    /* CC.NEW, RFC 1644 */
# 定义 TCP 选项常量，表示 CC.ECHO，RFC 1644
#define TCP_OPT_CCECHO        13    /* CC.ECHO, RFC 1644 */
# 定义 TCP 选项常量，表示备用校验和请求，RFC 1146
#define TCP_OPT_ALTSUM        14    /* alt checksum request, RFC 1146 */
# 定义 TCP 选项的类型和编号
#define TCP_OPT_ALTSUMDATA    15    /* alt checksum data, RFC 1146 */
#define TCP_OPT_SKEETER        16    /* Skeeter */
#define TCP_OPT_BUBBA        17    /* Bubba */
#define TCP_OPT_TRAILSUM    18    /* trailer checksum */
#define TCP_OPT_MD5        19    /* MD5 signature, RFC 2385 */
#define TCP_OPT_SCPS        20    /* SCPS capabilities */
#define TCP_OPT_SNACK        21    /* selective negative acks */
#define TCP_OPT_REC        22    /* record boundaries */
#define TCP_OPT_CORRUPT        23    /* corruption experienced */
#define TCP_OPT_SNAP        24    /* SNAP */
#define TCP_OPT_TCPCOMP        26    /* TCP compression filter */
#define TCP_OPT_MAX        27

# 定义 TCP 选项类型的宏，用于判断是否为类型字段
#define TCP_OPT_TYPEONLY(type)    \
    ((type) == TCP_OPT_EOL || (type) == TCP_OPT_NOP)

# 定义 TCP 选项结构体
struct tcp_opt {
    uint8_t        opt_type;    /* option type */
    uint8_t        opt_len;    /* option length >= TCP_OPT_LEN */
    union tcp_opt_data {
        uint16_t    mss;        /* TCP_OPT_MSS */
        uint8_t        wscale;        /* TCP_OPT_WSCALE */
        uint16_t    sack[19];    /* TCP_OPT_SACK */
        uint32_t    echo;        /* TCP_OPT_ECHO{REPLY} */
        uint32_t    timestamp[2];    /* TCP_OPT_TIMESTAMP */
        uint32_t    cc;        /* TCP_OPT_CC{NEW,ECHO} */
        uint8_t        cksum;        /* TCP_OPT_ALTSUM */
        uint8_t        md5[16];    /* TCP_OPT_MD5 */
        uint8_t        data8[TCP_OPT_LEN_MAX - TCP_OPT_LEN];
    } opt_data;
} __attribute__((__packed__));

# 如果不是使用 GCC 编译器，则取消字节对齐
#ifndef __GNUC__
# pragma pack()
#endif

# 定义 TCP 头部打包的宏
#define tcp_pack_hdr(hdr, sport, dport, seq, ack, flags, win, urp) do {    \
    struct tcp_hdr *tcp_pack_p = (struct tcp_hdr *)(hdr);        \
    tcp_pack_p->th_sport = htons(sport);                \
    tcp_pack_p->th_dport = htons(dport);                \
    tcp_pack_p->th_seq = htonl(seq);                \
    tcp_pack_p->th_ack = htonl(ack);                \
    tcp_pack_p->th_x2 = 0; tcp_pack_p->th_off = 5;            \
    # 设置 TCP 包的标志位
    tcp_pack_p->th_flags = flags;                    
    # 设置 TCP 包的窗口大小，并将主机字节序转换为网络字节序
    tcp_pack_p->th_win = htons(win);                
    # 设置 TCP 包的紧急指针，并将主机字节序转换为网络字节序
    tcp_pack_p->th_urp = htons(urp);                
} while (0)
// 使用 do-while 循环来确保即使条件不满足，也至少执行一次循环体
#endif /* DNET_TCP_H */
// 结束条件编译指令，用于条件编译，当条件满足时包含在编译中，否则被忽略
```