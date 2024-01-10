# `nmap\libnetutil\TCPHeader.h`

```
/* 这段代码最初是 Nping 工具的一部分。 */

#ifndef __TCPHEADER_H__
#define __TCPHEADER_H__ 1

#include "TransportLayerElement.h"

/* TCP 标志 */
#define TH_FIN   0x01  // FIN 标志
#define TH_SYN   0x02  // SYN 标志
#define TH_RST   0x04  // RST 标志
#define TH_PSH   0x08  // PSH 标志
#define TH_ACK   0x10  // ACK 标志
#define TH_URG   0x20  // URG 标志
#define TH_ECN   0x40  // ECN 标志
#define TH_CWR   0x80  // CWR 标志

/* TCP 选项 */
#define TCPOPT_EOL         0   // 选项列表结束 (RFC793)
#define TCPOPT_NOOP        1   // 无操作 (RFC793)
#define TCPOPT_MSS         2   // 最大段大小 (RFC793)
#define TCPOPT_WSCALE      3   // 窗口缩放选项 (RFC1323)
#define TCPOPT_SACKOK      4   // 允许 SACK (RFC2018)
#define TCPOPT_SACK        5   // SACK (RFC2018)
#define TCPOPT_ECHOREQ     6   // 回显 (已废弃) (RFC1072)(RFC6247)
#define TCPOPT_ECHOREP     7   // 回显回复 (已废弃) (RFC1072)(RFC6247)
#define TCPOPT_TSTAMP      8   // 时间戳选项 (RFC1323)
#define TCPOPT_POCP        9   // 允许部分顺序连接 (已废弃)
#define TCPOPT_POSP        10  // 部分顺序服务配置文件 (已废弃)
#define TCPOPT_CC          11  // CC (已废弃) (RFC1644)(RFC6247)
#define TCPOPT_CCNEW       12  // CC.NEW (已废弃) (RFC1644)(RFC6247)
#define TCPOPT_CCECHO      13  // CC.ECHO (已废弃) (RFC1644)(RFC6247)
#define TCPOPT_ALTCSUMREQ  14  // TCP 备用校验和请求 (已废弃)
#define TCPOPT_ALTCSUMDATA 15  // TCP 备用校验和数据 (已废弃)
#define TCPOPT_MD5         19  // MD5 签名选项 (已废弃) (RFC2385)
#define TCPOPT_SCPS        20  // SCPS 能力
#define TCPOPT_SNACK       21  // 选择性负面确认
#define TCPOPT_QSRES       27  // 快速启动响应 (RFC4782)
/* 定义 TCP 选项的类型和编号 */
#define TCPOPT_UTO         28  /* User Timeout Option (RFC5482)               */
#define TCPOPT_AO          29  /* TCP Authentication Option (RFC5925)         */

/* 内部常量 */
#define TCP_HEADER_LEN 20
#define MAX_TCP_OPTIONS_LEN 40
#define MAX_TCP_PAYLOAD_LEN 65495 /**< TCP 数据包的最大长度               */

/* 默认的头部数值 */
#define TCP_DEFAULT_SPORT 20
#define TCP_DEFAULT_DPORT 80
#define TCP_DEFAULT_SEQ   0
#define TCP_DEFAULT_ACK   0
#define TCP_DEFAULT_FLAGS 0x02
#define TCP_DEFAULT_WIN   8192
#define TCP_DEFAULT_URP   0

/*
+--------+--------+---------+--------...
|  Type  |  Len   |       Value
+--------+--------+---------+--------...
*/
/* 定义 TCP 选项的结构体 */
struct nping_tcp_opt {
    u8 type;                           /* 选项类型代码。           */
    u8 len;                            /* 选项长度。              */
    u8 *value;                         /* 选项值                */
}__attribute__((__packed__));
typedef struct nping_tcp_opt nping_tcp_opt_t;

/* 定义 TCPHeader 类，继承自 TransportLayerElement 类 */
class TCPHeader : public TransportLayerElement {

}; /* TCPHeader 类的结束 */

#endif /* __TCPHEADER_H__ */
```