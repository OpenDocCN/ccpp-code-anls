# `nmap\libdnet-stripped\src\ip-util.c`

```cpp
/*
 * ip-util.c
 *
 * 版权所有 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-util.c 595 2005-02-17 02:55:56Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <errno.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"
#include "crc32ct.h"

/* CRC-32C (Castagnoli). Public domain. */
// 计算 CRC-32C 校验值的函数
static unsigned long
_crc32c(unsigned char *buf, int len)
{
    int i;
    unsigned long crc32 = ~0L;
    unsigned long result;
    unsigned char byte0, byte1, byte2, byte3;

    for (i = 0; i < len; i++) {
        CRC32C(crc32, buf[i]);
    }

    result = ~crc32;

    byte0 =  result        & 0xff;
    byte1 = (result >>  8) & 0xff;
    byte2 = (result >> 16) & 0xff;
    byte3 = (result >> 24) & 0xff;
    crc32 = ((byte0 << 24) | (byte1 << 16) | (byte2 <<  8) | byte3);
    return crc32;
}

// 添加 IP 选项的函数
ssize_t
ip_add_option(void *buf, size_t len, int proto,
    const void *optbuf, size_t optlen)
{
    struct ip_hdr *ip;
    struct tcp_hdr *tcp = NULL;
    u_char *p;
    int hl, datalen, padlen;
    
    if (proto != IP_PROTO_IP && proto != IP_PROTO_TCP) {
        errno = EINVAL;
        return (-1);
    }
    ip = (struct ip_hdr *)buf;
    hl = ip->ip_hl << 2;
    p = (u_char *)buf + hl;
    
    if (proto == IP_PROTO_TCP) {
        tcp = (struct tcp_hdr *)p;
        hl = tcp->th_off << 2;
        p = (u_char *)tcp + hl;
    }
    datalen = (int) (ntohs(ip->ip_len) - (p - (u_char *)buf));
    
    /* 计算填充到下一个字边界的长度 */
    if ((padlen = 4 - (optlen % 4)) == 4)
        padlen = 0;

    /* XXX - IP_HDR_LEN_MAX == TCP_HDR_LEN_MAX */
    if (hl + optlen + padlen > IP_HDR_LEN_MAX ||
        ntohs(ip->ip_len) + optlen + padlen > len) {
        errno = EINVAL;
        return (-1);
    }
    /* XXX - IP_OPT_TYPEONLY() == TCP_OPT_TYPEONLY */
    if (IP_OPT_TYPEONLY(((struct ip_opt *)optbuf)->opt_type))
        optlen = 1;
    
    /* 移动任何现有数据 */
    # 如果数据长度不为0，则将指针p + optlen + padlen处的数据移动到指针p处，长度为datalen
    if (datalen) {
        memmove(p + optlen + padlen, p, datalen);
    }
    # 如果padlen不为0，则将指针p处的数据填充为IP_OPT_NOP，长度为padlen，并将指针p移动padlen个位置
    /* XXX - IP_OPT_NOP == TCP_OPT_NOP */
    if (padlen) {
        memset(p, IP_OPT_NOP, padlen);
        p += padlen;
    }
    # 将optbuf中的数据移动到指针p处，长度为optlen，并将指针p移动optlen个位置
    memmove(p, optbuf, optlen);
    p += optlen;
    optlen += padlen;
    
    # 如果协议为IP_PROTO_IP，则计算IP头部长度并赋值给ip->ip_hl
    if (proto == IP_PROTO_IP)
        ip->ip_hl = (int) ((p - (u_char *)ip) >> 2);
    # 如果协议为IP_PROTO_TCP，则计算TCP头部长度并赋值给tcp->th_off
    else if (proto == IP_PROTO_TCP)
        tcp->th_off = (int) ((p - (u_char *)tcp) >> 2);

    # 更新IP数据包的总长度，包括选项长度
    ip->ip_len = htons((u_short) (ntohs(ip->ip_len) + optlen));
    
    # 返回选项长度
    return (ssize_t)(optlen);
}  // 结束函数 ip_checksum

void
ip_checksum(void *buf, size_t len)
{
    struct ip_hdr *ip;  // 定义 IP 头结构体指针
    int hl, off, sum;  // 定义变量 hl, off, sum

    if (len < IP_HDR_LEN)  // 如果长度小于 IP 头部长度，返回
        return;
    
    ip = (struct ip_hdr *)buf;  // 将 buf 转换为 IP 头结构体指针
    hl = ip->ip_hl << 2;  // 计算 IP 头部长度
    ip->ip_sum = 0;  // 将 IP 校验和置为 0
    sum = ip_cksum_add(ip, hl, 0);  // 调用函数计算校验和
    ip->ip_sum = ip_cksum_carry(sum);  // 将计算得到的校验和赋值给 IP 头部的校验和字段

    off = htons(ip->ip_off);  // 将 IP 分片偏移字段转换为网络字节序
    
    if ((off & IP_OFFMASK) != 0 || (off & IP_MF) != 0)  // 如果分片偏移字段的偏移量不为 0 或者标志位 MF 不为 0，返回
        return;
    
    len -= hl;  // 减去 IP 头部长度得到有效载荷长度
    
    if (ip->ip_p == IP_PROTO_TCP) {  // 如果协议类型为 TCP
        struct tcp_hdr *tcp = (struct tcp_hdr *)((u_char *)ip + hl);  // 定义 TCP 头结构体指针并指向 TCP 头部的位置
        
        if (len >= TCP_HDR_LEN) {  // 如果有效载荷长度大于等于 TCP 头部长度
            tcp->th_sum = 0;  // 将 TCP 校验和置为 0
            sum = ip_cksum_add(tcp, len, 0) + htons((u_short)(ip->ip_p + len));  // 计算 TCP 校验和
            sum = ip_cksum_add(&ip->ip_src, 8, sum);  // 添加源 IP 地址到校验和
            tcp->th_sum = ip_cksum_carry(sum);  // 将计算得到的校验和赋值给 TCP 头部的校验和字段
        }
    } else if (ip->ip_p == IP_PROTO_UDP) {  // 如果协议类型为 UDP
        struct udp_hdr *udp = (struct udp_hdr *)((u_char *)ip + hl);  // 定义 UDP 头结构体指针并指向 UDP 头部的位置

        if (len >= UDP_HDR_LEN) {  // 如果有效载荷长度大于等于 UDP 头部长度
            udp->uh_sum = 0;  // 将 UDP 校验和置为 0
            sum = ip_cksum_add(udp, len, 0) + htons((u_short)(ip->ip_p + len));  // 计算 UDP 校验和
            sum = ip_cksum_add(&ip->ip_src, 8, sum);  // 添加源 IP 地址到校验和
            udp->uh_sum = ip_cksum_carry(sum);  // 将计算得到的校验和赋值给 UDP 头部的校验和字段
            if (!udp->uh_sum)
                udp->uh_sum = 0xffff;    // 如果 UDP 校验和为 0，将其置为 0xffff，符合 RFC 768
        }
    } else if (ip->ip_p == IP_PROTO_SCTP) {  // 如果协议类型为 SCTP
        struct sctp_hdr *sctp = (struct sctp_hdr *)((u_char *)ip + hl);  // 定义 SCTP 头结构体指针并指向 SCTP 头部的位置

        if (len >= SCTP_HDR_LEN) {  // 如果有效载荷长度大于等于 SCTP 头部长度
            sctp->sh_sum = 0;  // 将 SCTP 校验和置为 0
            sctp->sh_sum = htonl(_crc32c((u_char *)sctp, len));  // 计算 SCTP 校验和
        }
    } else if (ip->ip_p == IP_PROTO_ICMP || ip->ip_p == IP_PROTO_IGMP) {  // 如果协议类型为 ICMP 或 IGMP
        struct icmp_hdr *icmp = (struct icmp_hdr *)((u_char *)ip + hl);  // 定义 ICMP 头结构体指针并指向 ICMP 头部的位置
        
        if (len >= ICMP_HDR_LEN) {  // 如果有效载荷长度大于等于 ICMP 头部长度
            icmp->icmp_cksum = 0;  // 将 ICMP 校验和置为 0
            sum = ip_cksum_add(icmp, len, 0);  // 计算 ICMP 校验和
            icmp->icmp_cksum = ip_cksum_carry(sum);  // 将计算得到的校验和赋值给 ICMP 头部的校验和字段
        }
    }
}

int
ip_cksum_add(const void *buf, size_t len, int cksum)
{
    uint16_t *sp = (uint16_t *)buf;  // 定义指向 buf 的 uint16_t 类型指针
    int n, sn;  // 定义变量 n, sn
    # 计算输入长度的一半
    sn = (int) len / 2;
    # 计算n的值
    n = (sn + 15) / 16;

    /* XXX - unroll loop using Duff's device. */
    # 使用Duff's设备展开循环
    switch (sn % 16) {
    case 0:    do {
        cksum += *sp++;
    case 15:
        cksum += *sp++;
    case 14:
        cksum += *sp++;
    case 13:
        cksum += *sp++;
    case 12:
        cksum += *sp++;
    case 11:
        cksum += *sp++;
    case 10:
        cksum += *sp++;
    case 9:
        cksum += *sp++;
    case 8:
        cksum += *sp++;
    case 7:
        cksum += *sp++;
    case 6:
        cksum += *sp++;
    case 5:
        cksum += *sp++;
    case 4:
        cksum += *sp++;
    case 3:
        cksum += *sp++;
    case 2:
        cksum += *sp++;
    case 1:
        cksum += *sp++;
        } while (--n > 0);
    }
    # 如果输入长度为奇数，则执行下面的语句
    if (len & 1)
        cksum += htons(*(u_char *)sp << 8);
    # 返回cksum的值
    return (cksum);
# 闭合前面的函数定义
```