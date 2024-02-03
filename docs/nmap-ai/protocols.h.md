# `nmap\protocols.h`

```cpp
/* $Id$ */

// 如果 PROTOCOLS_H 未定义，则定义 PROTOCOLS_H
#ifndef PROTOCOLS_H
#define PROTOCOLS_H

// 包含 nbase.h 文件
#include "nbase.h"

// 如果 IPPROTO_SCTP 未定义，则包含 netutil.h 文件
#ifndef IPPROTO_SCTP
#include "libnetutil/netutil.h"
#endif

// 定义 nprotoent 结构体
struct nprotoent {
  const char *p_name; // 协议名称
  u16 p_proto; // 协议编号
};

// 声明函数 addprotocolsfromservmask，接受 mask 和 porttbl 两个参数
int addprotocolsfromservmask(char *mask, u8 *porttbl);
// 声明函数 nmap_getprotbynum，接受 num 作为参数
const struct nprotoent *nmap_getprotbynum(int num);
// 声明函数 nmap_getprotbyname，接受 name 作为参数
const struct nprotoent *nmap_getprotbyname(const char *name);

// 定义最大的协议字符串长度为 4
#define MAX_IPPROTOSTRLEN 4
// 定义宏 IPPROTO2STR，根据协议编号返回对应的字符串
#define IPPROTO2STR(p)        \
  ((p)==IPPROTO_TCP ? "tcp" :    \
   (p)==IPPROTO_UDP ? "udp" :    \
   (p)==IPPROTO_SCTP ? "sctp" :    \
   "n/a")
// 定义宏 IPPROTO2STR_UC，根据协议编号返回对应的大写字符串
#define IPPROTO2STR_UC(p)    \
  ((p)==IPPROTO_TCP ? "TCP" :    \
   (p)==IPPROTO_UDP ? "UDP" :    \
   (p)==IPPROTO_SCTP ? "SCTP" :    \
   "N/A")

// 结束条件编译指令
#endif
```