# `nmap\struct_ip.h`

```cpp
/* The C library on AIX defines the names of various members of struct ip to
   something else in <netinet/ip.h>:

struct ip {
        struct    ip_firstfour ip_ff;
#define    ip_v    ip_ff.ip_fv
#define    ip_hl    ip_ff.ip_fhl
#define    ip_vhl    ip_ff.ip_fvhl
#define    ip_tos    ip_ff.ip_ftos
#define    ip_len    ip_ff.ip_flen

   This breaks code that actually wants to use names like ip_v for its own
   purposes, like struct ip_hdr in libdnet. The AIX definitions will work
   if they are included late in the list of includes, before other code that
   might want to use the above names has already been preprocessed. The
   includes that end up defining struct ip are therefore limited to this
   file, so it can be included in a .cc file after other .h have been
   included. */

#ifdef WIN32
typedef unsigned __int32 u_int32_t;
typedef unsigned __int16 u_int16_t;
typedef unsigned __int8 u_int8_t;
#endif
/* Linux uses these defines in netinet/ip.h to use the correct struct ip */
#ifndef __FAVOR_BSD
#define __FAVOR_BSD
#endif
#ifndef __USE_BSD
#define __USE_BSD
#endif
#ifndef _BSD_SOURCE
#define _BSD_SOURCE
#endif

/* BSDI needs this to insure the correct struct ip */
#undef _IP_VHL

#ifndef NETINET_IN_SYSTM_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/in_systm.h> /* defines n_long needed for netinet/ip.h */
#define NETINET_IN_SYSTM_H
#endif
#if HAVE_NET_IF_H
#ifndef NET_IF_H /* This guarding is needed for at least some versions of OpenBSD */
#include <net/if.h>
#define NET_IF_H
#endif
#endif
#ifndef NETINET_IP_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/ip.h>
#define NETINET_IP_H
#endif
#include <net/if_arp.h>

#ifndef WIN32
#include <netinet/ip_icmp.h>
#endif

#ifndef HAVE_STRUCT_IP
#define HAVE_STRUCT_IP

/* From Linux glibc, which apparently borrowed it from
   BSD code.  Slightly modified for portability --fyodor@nmap.org */
/*
 * 定义了一个裸露的互联网头部结构，不包含选项。
 */
struct ip
  {
#if WORDS_BIGENDIAN
    u_int8_t ip_v:4;                    /* 版本 */
    u_int8_t ip_hl:4;                   /* 头部长度 */
#else
    u_int8_t ip_hl:4;                   /* 头部长度 */
    u_int8_t ip_v:4;                    /* 版本 */
#endif
    u_int8_t ip_tos;                    /* 服务类型 */
    u_short ip_len;                     /* 总长度 */
    u_short ip_id;                      /* 标识 */
    u_short ip_off;                     /* 分段偏移字段 */
#define IP_RF 0x8000                    /* 保留分段标志 */
#define IP_DF 0x4000                    /* 不分段标志 */
#define IP_MF 0x2000                    /* 更多分段标志 */
#define IP_OFFMASK 0x1fff               /* 分段位掩码 */
    u_int8_t ip_ttl;                    /* 存活时间 */
    u_int8_t ip_p;                      /* 协议 */
    u_short ip_sum;                     /* 校验和 */
    struct in_addr ip_src, ip_dst;      /* 源地址和目的地址 */
  };

#endif /* HAVE_STRUCT_IP */

#ifndef HAVE_STRUCT_ICMP
#define HAVE_STRUCT_ICMP
/* 来自 Linux /usr/include/netinet/ip_icmp.h GLIBC */

/*
 * ICMP路由器通告的内部结构
 */
struct icmp_ra_addr
{
  u_int32_t ira_addr;
  u_int32_t ira_preference;
};

struct icmp
{
  u_int8_t  icmp_type;  /* 消息类型，见下文 */
  u_int8_t  icmp_code;  /* 类型子码 */
  u_int16_t icmp_cksum; /* 结构的反码校验和 */
  union
  {
    struct ih_idseq             /* 回显数据报 */
    {
      u_int16_t icd_id;
      u_int16_t icd_seq;
    } ih_idseq;
    u_int32_t ih_void;

    /* ICMP_UNREACH_NEEDFRAG -- 路径MTU发现（RFC1191） */
    struct ih_pmtu
    {
      u_int16_t ipm_void;
      u_int16_t ipm_nextmtu;
    } ih_pmtu;

    struct ih_rtradv
    {
      u_int8_t irt_num_addrs;
      u_int8_t irt_wpa;
      u_int16_t irt_lifetime;
  } ih_rtradv;  // 定义一个名为ih_rtradv的结构体变量
  } icmp_hun;  // 定义一个名为icmp_hun的结构体变量
  /* 从联合体和#定义中删除icmp_pptr和icmp_gwaddr，因为它们与dnet冲突 */  // 移除联合体中的icmp_pptr和icmp_gwaddr，并且移除它们的#定义，因为它们与dnet冲突
// 定义 ICMP 报文中的标识字段
#define icmp_id         icmp_hun.ih_idseq.icd_id
// 定义 ICMP 报文中的序列号字段
#define icmp_seq        icmp_hun.ih_idseq.icd_seq
// 定义 ICMP 报文中的 void 字段
#define icmp_void       icmp_hun.ih_void
// 定义 ICMP 报文中的路径 MTU 探测报文的 void 字段
#define icmp_pmvoid     icmp_hun.ih_pmtu.ipm_void
// 定义 ICMP 报文中的路径 MTU 探测报文的下一个 MTU 字段
#define icmp_nextmtu    icmp_hun.ih_pmtu.ipm_nextmtu
// 定义 ICMP 报文中的路由器通告报文的地址数量字段
#define icmp_num_addrs  icmp_hun.ih_rtradv.irt_num_addrs
// 定义 ICMP 报文中的路由器通告报文的 WPA 字段
#define icmp_wpa        icmp_hun.ih_rtradv.irt_wpa
// 定义 ICMP 报文中的路由器通告报文的生存时间字段
#define icmp_lifetime   icmp_hun.ih_rtradv.irt_lifetime
// 定义 ICMP 报文中的数据联合体
  union
  {
    // 定义 ICMP 报文中的时间戳报文的发送时间字段
    struct
    {
      u_int32_t its_otime;
      // 定义 ICMP 报文中的时间戳报文的接收时间字段
      u_int32_t its_rtime;
      // 定义 ICMP 报文中的时间戳报文的传输时间字段
      u_int32_t its_ttime;
    } id_ts;
    // 定义 ICMP 报文中的 IP 报文
    struct
    {
      struct ip idi_ip;
      /* options and then 64 bits of data */
    } id_ip;
    // 定义 ICMP 报文中的路由器通告报文的地址字段
    struct icmp_ra_addr id_radv;
    // 定义 ICMP 报文中的掩码字段
    u_int32_t   id_mask;
    // 定义 ICMP 报文中的数据字段
    u_int8_t    id_data[1];
  } icmp_dun;
// 定义 ICMP 报文中的时间戳报文的发送时间字段
#define icmp_otime      icmp_dun.id_ts.its_otime
// 定义 ICMP 报文中的时间戳报文的接收时间字段
#define icmp_rtime      icmp_dun.id_ts.its_rtime
// 定义 ICMP 报文中的时间戳报文的传输时间字段
#define icmp_ttime      icmp_dun.id_ts.its_ttime
// 定义 ICMP 报文中的 IP 报文字段
#define icmp_ip         icmp_dun.id_ip.idi_ip
// 定义 ICMP 报文中的路由器通告报文字段
#define icmp_radv       icmp_dun.id_radv
// 定义 ICMP 报文中的掩码字段
#define icmp_mask       icmp_dun.id_mask
// 定义 ICMP 报文中的数据字段
#define icmp_data       icmp_dun.id_data
};
#endif /* HAVE_STRUCT_ICMP */
```