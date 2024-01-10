# `nmap\libdnet-stripped\src\ip6.c`

```
/*
 * ip6.c
 *
 * 版权所有 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip6.c 539 2005-01-23 07:36:54Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include "dnet.h"

#define IP6_IS_EXT(n)    \
    ((n) == IP_PROTO_HOPOPTS || (n) == IP_PROTO_DSTOPTS || \
     (n) == IP_PROTO_ROUTING || (n) == IP_PROTO_FRAGMENT)

void
ip6_checksum(void *buf, size_t len)
{
    struct ip6_hdr *ip6 = (struct ip6_hdr *)buf;  // 定义 IPv6 头结构体指针并初始化
    struct ip6_ext_hdr *ext;  // 定义 IPv6 扩展头结构体指针
    u_char *p, nxt;  // 定义无符号字符指针和下一个协议字段
    int i, sum;  // 定义整型变量 i 和 sum
    
    nxt = ip6->ip6_nxt;  // 获取下一个协议字段
    
    for (i = IP6_HDR_LEN; IP6_IS_EXT(nxt); i += (ext->ext_len + 1) << 3) {  // 循环遍历 IPv6 扩展头
        if (i >= (int)len) return;  // 如果超出长度则返回
        ext = (struct ip6_ext_hdr *)((u_char *)buf + i);  // 获取当前扩展头结构体指针
        nxt = ext->ext_nxt;  // 获取下一个协议字段
    }
    p = (u_char *)buf + i;  // 获取指向数据的指针
    len -= i;  // 减去已经处理的长度
    
    if (nxt == IP_PROTO_TCP) {  // 如果下一个协议字段是 TCP
        struct tcp_hdr *tcp = (struct tcp_hdr *)p;  // 定义 TCP 头结构体指针并初始化
        
        if (len >= TCP_HDR_LEN) {  // 如果长度足够
            tcp->th_sum = 0;  // 将校验和字段置零
            sum = ip_cksum_add(tcp, len, 0) + htons(nxt + (u_short)len);  // 计算校验和
            sum = ip_cksum_add(&ip6->ip6_src, 32, sum);  // 添加源地址到校验和
            tcp->th_sum = ip_cksum_carry(sum);  // 计算进位后的校验和
        }
    } else if (nxt == IP_PROTO_UDP) {  // 如果下一个协议字段是 UDP
        struct udp_hdr *udp = (struct udp_hdr *)p;  // 定义 UDP 头结构体指针并初始化

        if (len >= UDP_HDR_LEN) {  // 如果长度足够
            udp->uh_sum = 0;  // 将校验和字段置零
            sum = ip_cksum_add(udp, len, 0) + htons(nxt + (u_short)len);  // 计算校验和
            sum = ip_cksum_add(&ip6->ip6_src, 32, sum);  // 添加源地址到校验和
            if ((udp->uh_sum = ip_cksum_carry(sum)) == 0)  // 如果校验和为零
                udp->uh_sum = 0xffff;  // 将校验和置为 0xffff
        }
    } else if (nxt == IP_PROTO_ICMPV6) {  // 如果下一个协议字段是 ICMPv6
        struct icmp_hdr *icmp = (struct icmp_hdr *)p;  // 定义 ICMPv6 头结构体指针并初始化

        if (len >= ICMP_HDR_LEN) {  // 如果长度足够
            icmp->icmp_cksum = 0;  // 将校验和字段置零
            sum = ip_cksum_add(icmp, len, 0) + htons(nxt + (u_short)len);  // 计算校验和
            sum = ip_cksum_add(&ip6->ip6_src, 32, sum);  // 添加源地址到校验和
            icmp->icmp_cksum = ip_cksum_carry(sum);  // 计算进位后的校验和
        }        
    } else if (nxt == IP_PROTO_ICMP || nxt == IP_PROTO_IGMP) {
        # 如果下一个协议是 ICMP 或者 IGMP
        struct icmp_hdr *icmp = (struct icmp_hdr *)p;
        # 创建指向 ICMP 头部的指针
        
        if (len >= ICMP_HDR_LEN) {
            # 如果数据长度大于等于 ICMP 头部长度
            icmp->icmp_cksum = 0;
            # 将 ICMP 校验和字段置零
            sum = ip_cksum_add(icmp, len, 0);
            # 计算 ICMP 校验和
            icmp->icmp_cksum = ip_cksum_carry(sum);
            # 将计算得到的校验和赋值给 ICMP 校验和字段
        }
    }
# 闭合前面的函数定义
```