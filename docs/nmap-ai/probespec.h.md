# `nmap\probespec.h`

```cpp
#ifndef PROBESPEC_H
#define PROBESPEC_H
#include <nbase.h>

// 定义 TCP 数据结构
struct probespec_tcpdata {
  u16 dport; // 目的端口号
  u8 flags; // 标志位
};

// 定义 UDP 数据结构
struct probespec_udpdata {
  u16 dport; // 目的端口号
};

// 定义 SCTP 数据结构
struct probespec_sctpdata {
  u16 dport; // 目的端口号
  u8 chunktype; // 块类型
};

// 定义 ICMP 数据结构
struct probespec_icmpdata {
  u8 type; // 类型
  u8 code; // 代码
};

// 定义 ICMPv6 数据结构
struct probespec_icmpv6data {
  u8 type; // 类型
  u8 code; // 代码
};

// 定义各种类型的常量
#define PS_NONE 0
#define PS_TCP 1
#define PS_UDP 2
#define PS_PROTO 3
#define PS_ICMP 4
#define PS_ARP 5
#define PS_CONNECTTCP 6
#define PS_SCTP 7
#define PS_ICMPV6 8
#define PS_ND 9

/* The size of this structure is critical, since there can be tens of
   thousands of them stored together ... */
// 定义探测规范结构
typedef struct probespec {
  /* To save space, I changed this from private enum (took 4 bytes) to
     u8 that uses #defines above */
  u8 type; // 类型
  u8 proto; /* If not PS_ARP -- Protocol number ... eg IPPROTO_TCP, etc. */
  union {
    struct probespec_tcpdata tcp; /* If type is PS_TCP or PS_CONNECTTCP. */
    struct probespec_udpdata udp; /* PS_UDP */
    struct probespec_sctpdata sctp; /* PS_SCTP */
    struct probespec_icmpdata icmp; /* PS_ICMP */
    struct probespec_icmpv6data icmpv6; /* PS_ICMPV6 */
    /* Nothing needed for PS_ARP, since src mac and target IP are
       avail from target structure anyway */
  } pd;
} probespec;

#endif
```