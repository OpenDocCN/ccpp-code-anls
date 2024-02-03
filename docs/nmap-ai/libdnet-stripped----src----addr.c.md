# `nmap\libdnet-stripped\src\addr.c`

```cpp
/*
 * addr.c
 *
 * Network address operations.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: addr.c 610 2005-06-26 18:23:26Z dugsong $
 */

#ifdef WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <sys/types.h>
#ifdef HAVE_NET_IF_H
# include <sys/socket.h>
# include <net/if.h>
#endif
#ifdef HAVE_NET_IF_DL_H
# include <net/if_dl.h>
#endif
#ifdef HAVE_NET_RAW_H
# include <net/raw.h>
#endif

#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

#ifndef MAXHOSTNAMELEN
# define MAXHOSTNAMELEN    256
#endif

union sockunion {
#ifdef HAVE_NET_IF_DL_H
    struct sockaddr_dl    sdl;
#endif
    struct sockaddr_in    sin;
#ifdef HAVE_SOCKADDR_IN6
    struct sockaddr_in6    sin6;
#endif
    struct sockaddr        sa;
#ifdef AF_RAW
    struct sockaddr_raw    sr;
#endif
};

int
addr_cmp(const struct addr *a, const struct addr *b)
{
    int i, j, k;

    /* 比较两个地址结构的地址类型和地址位数 */
    if ((i = a->addr_type - b->addr_type) != 0)
        return (i);
    
    /* 比较两个地址结构的地址位数 */
    if ((i = a->addr_bits - b->addr_bits) != 0)
        return (i);
    
    j = b->addr_bits / 8;

    for (i = 0; i < j; i++) {
        /* 逐个比较地址结构中的数据 */
        if ((k = a->addr_data8[i] - b->addr_data8[i]) != 0)
            return (k);
    }
    if ((k = b->addr_bits % 8) == 0)
        return (0);

    k = (~(unsigned int)0) << (8 - k);
    i = b->addr_data8[j] & k;
    j = a->addr_data8[j] & k;
    
    return (j - i);
}

int
addr_net(const struct addr *a, struct addr *b)
{
    uint32_t mask;
    int i, j;

    if (a->addr_type == ADDR_TYPE_IP) {
        /* 根据地址位数生成掩码，对地址进行网络地址转换 */
        addr_btom(a->addr_bits, &mask, IP_ADDR_LEN);
        b->addr_type = ADDR_TYPE_IP;
        b->addr_bits = IP_ADDR_BITS;
        b->addr_ip = a->addr_ip & mask;
    } else if (a->addr_type == ADDR_TYPE_ETH) {
        /* 对以太网地址进行处理 */
        memcpy(b, a, sizeof(*b));
        if (a->addr_data8[0] & 0x1)
            memset(b->addr_data8 + 3, 0, 3);
        b->addr_bits = ETH_ADDR_BITS;
    } else if (a->addr_type == ADDR_TYPE_IP6) {
      // 如果地址类型是 IP6
      if (a->addr_bits > IP6_ADDR_BITS)
        // 如果地址位数大于 IP6 地址位数，返回错误
        return (-1);
        // 设置 b 的地址类型为 IP6
        b->addr_type = ADDR_TYPE_IP6;
        // 设置 b 的地址位数为 IP6 地址位数
        b->addr_bits = IP6_ADDR_BITS;
        // 将 b 的 IP6 地址数据全部置为 0
        memset(&b->addr_ip6, 0, IP6_ADDR_LEN);
        
        // 根据地址位数计算需要复制的 32 位数据块的个数
        switch ((i = a->addr_bits / 32)) {
        // 根据不同的数据块个数进行复制
        case 4: b->addr_data32[3] = a->addr_data32[3];
        case 3: b->addr_data32[2] = a->addr_data32[2];
        case 2: b->addr_data32[1] = a->addr_data32[1];
        case 1: b->addr_data32[0] = a->addr_data32[0];
        }
        // 如果还有剩余的位数
        if ((j = a->addr_bits % 32) > 0) {
            // 根据剩余位数生成掩码
            addr_btom(j, &mask, sizeof(mask));
            // 将原地址数据与掩码进行与操作，得到新的地址数据
            b->addr_data32[i] = a->addr_data32[i] & mask;
        }
    } else
        // 如果地址类型不是 IP6，返回错误
        return (-1);
    
    // 返回成功
    return (0);
# 定义一个函数，用于计算广播地址
int
addr_bcast(const struct addr *a, struct addr *b)
{
    # 定义一个用于存储子网掩码的变量
    struct addr mask;
    
    # 如果地址类型为 IP 地址
    if (a->addr_type == ADDR_TYPE_IP) {
        # 根据地址位数生成子网掩码
        addr_btom(a->addr_bits, &mask.addr_ip, IP_ADDR_LEN);
        # 设置 b 的地址类型为 IP 地址
        b->addr_type = ADDR_TYPE_IP;
        # 设置 b 的地址位数为 IP 地址的位数
        b->addr_bits = IP_ADDR_BITS;
        # 计算广播地址
        b->addr_ip = (a->addr_ip & mask.addr_ip) | (~0L & ~mask.addr_ip);
    } 
    # 如果地址类型为以太网地址
    else if (a->addr_type == ADDR_TYPE_ETH) {
        # 设置 b 的地址类型为以太网地址
        b->addr_type = ADDR_TYPE_ETH;
        # 设置 b 的地址位数为以太网地址的位数
        b->addr_bits = ETH_ADDR_BITS;
        # 将以太网广播地址复制给 b
        memcpy(&b->addr_eth, ETH_ADDR_BROADCAST, ETH_ADDR_LEN);
    } 
    # 如果地址类型不是 IP 地址也不是以太网地址
    else {
        # 设置错误码为无效参数
        errno = EINVAL;
        # 返回错误
        return (-1);
    }
    # 返回成功
    return (0);
}

# 定义一个函数，用于将地址转换为可打印的字符串形式
char *
addr_ntop(const struct addr *src, char *dst, size_t size)
{
    # 如果地址类型为 IP 地址且大小足够
    if (src->addr_type == ADDR_TYPE_IP && size >= 20) {
        # 将 IP 地址转换为可打印的字符串形式
        if (ip_ntop(&src->addr_ip, dst, size) != NULL) {
            # 如果地址位数不等于 IP 地址的位数，将地址位数添加到字符串末尾
            if (src->addr_bits != IP_ADDR_BITS)
                sprintf(dst + strlen(dst), "/%d", src->addr_bits);
            # 返回转换后的字符串
            return (dst);
        }
    } 
    # 如果地址类型为 IPv6 地址且大小足够
    else if (src->addr_type == ADDR_TYPE_IP6 && size >= 42) {
        # 将 IPv6 地址转换为可打印的字符串形式
        if (ip6_ntop(&src->addr_ip6, dst, size) != NULL) {
            # 如果地址位数不等于 IPv6 地址的位数，将地址位数添加到字符串末尾
            if (src->addr_bits != IP6_ADDR_BITS)
                sprintf(dst + strlen(dst), "/%d", src->addr_bits);
            # 返回转换后的字符串
            return (dst);
        }
    } 
    # 如果地址类型为以太网地址且大小足够
    else if (src->addr_type == ADDR_TYPE_ETH && size >= 18) {
        # 如果地址位数等于以太网地址的位数，将以太网地址转换为可打印的字符串形式
        if (src->addr_bits == ETH_ADDR_BITS)
            return (eth_ntop(&src->addr_eth, dst, size));
    }
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回空指针
    return (NULL);
}

# 定义一个函数，用于将字符串形式的地址转换为二进制形式
int
addr_pton(const char *src, struct addr *dst)
{
    # 定义一个用于存储主机信息的变量
    struct hostent *hp;
    # 定义一个用于存储临时字符串的变量
    char *ep, tmp[300];
    # 初始化地址位数为 -1
    long bits = -1;
    # 定义一个整型变量
    int i;
    # 遍历源字符串的每个字符
    for (i = 0; i < (int)sizeof(tmp) - 1; i++) {
        # 如果当前字符是斜杠
        if (src[i] == '/') {
            # 将临时字符串中当前位置的字符设为结束符
            tmp[i] = '\0';
            # 如果斜杠后面包含点号，表示是 IP 地址掩码
            if (strchr(&src[i + 1], '.')) {
                uint32_t m;
                uint16_t b;
                # 解析 IP 地址掩码
                /* XXX - mask is specified like /255.0.0.0 */
                if (ip_pton(&src[i + 1], &m) != 0) {
                    errno = EINVAL;
                    return (-1);
                }
                # 将掩码转换为位数
                addr_mtob(&m, sizeof(m), &b);
                bits = b;
            } else {
                # 否则将斜杠后面的字符串转换为整数，表示位数
                bits = strtol(&src[i + 1], &ep, 10);
                # 检查转换是否成功
                if (ep == src || *ep != '\0' || bits < 0) {
                    errno = EINVAL;
                    return (-1);
                }
            }
            # 处理完毕，跳出循环
            break;
        } else if ((tmp[i] = src[i]) == '\0')
            break;
    }
    # 根据临时字符串解析出的地址类型和位数，设置目标地址结构体
    if (ip_pton(tmp, &dst->addr_ip) == 0) {
        dst->addr_type = ADDR_TYPE_IP;
        dst->addr_bits = IP_ADDR_BITS;
    } else if (eth_pton(tmp, &dst->addr_eth) == 0) {
        dst->addr_type = ADDR_TYPE_ETH;
        dst->addr_bits = ETH_ADDR_BITS;
    } else if (ip6_pton(tmp, &dst->addr_ip6) == 0) {
        dst->addr_type = ADDR_TYPE_IP6;
        dst->addr_bits = IP6_ADDR_BITS;
    } else if ((hp = gethostbyname(tmp)) != NULL) {
        memcpy(&dst->addr_ip, hp->h_addr, IP_ADDR_LEN);
        dst->addr_type = ADDR_TYPE_IP;
        dst->addr_bits = IP_ADDR_BITS;
    } else {
        errno = EINVAL;
        return (-1);
    }
    # 如果指定了位数，检查是否合法，并更新目标地址结构体的位数
    if (bits >= 0) {
        if (bits > dst->addr_bits) {
            errno = EINVAL;
            return (-1);
        }
        dst->addr_bits = (uint16_t)bits;
    }
    # 处理完毕，返回成功
    return (0);
}
# 将地址转换为点分十进制字符串
char *
addr_ntoa(const struct addr *a)
{
    static char *p, buf[BUFSIZ];
    char *q = NULL;
    
    # 如果 p 为空或者 p 超出缓冲区大小减去 64 的范围，则将 p 指向缓冲区起始位置
    if (p == NULL || p > buf + sizeof(buf) - 64 /* XXX */)
        p = buf;
    
    # 将地址转换为点分十进制字符串
    if (addr_ntop(a, p, (buf + sizeof(buf)) - p) != NULL) {
        q = p;
        p += strlen(p) + 1;
    }
    return (q);
}

# 将地址转换为套接字地址结构
int
addr_ntos(const struct addr *a, struct sockaddr *sa)
{
    union sockunion *so = (union sockunion *)sa;
    
    # 根据地址类型进行不同的处理
    switch (a->addr_type) {
    case ADDR_TYPE_ETH:
        # 处理以太网地址类型
#ifdef HAVE_NET_IF_DL_H
        memset(&so->sdl, 0, sizeof(so->sdl));
# ifdef HAVE_SOCKADDR_SA_LEN
        so->sdl.sdl_len = sizeof(so->sdl);
# endif
# ifdef AF_LINK
        so->sdl.sdl_family = AF_LINK;
# else
        so->sdl.sdl_family = AF_UNSPEC;
# endif
        so->sdl.sdl_alen = ETH_ADDR_LEN;
        memcpy(LLADDR(&so->sdl), &a->addr_eth, ETH_ADDR_LEN);
#else
        memset(sa, 0, sizeof(*sa));
# ifdef AF_LINK
        sa->sa_family = AF_LINK;
# else
        sa->sa_family = AF_UNSPEC;
# endif
        memcpy(sa->sa_data, &a->addr_eth, ETH_ADDR_LEN);
#endif
        break;
#ifdef HAVE_SOCKADDR_IN6
    case ADDR_TYPE_IP6:
        # 处理 IPv6 地址类型
        memset(&so->sin6, 0, sizeof(so->sin6));
#ifdef HAVE_SOCKADDR_SA_LEN
        so->sin6.sin6_len = sizeof(so->sin6);
#endif
        so->sin6.sin6_family = AF_INET6;
        memcpy(&so->sin6.sin6_addr, &a->addr_ip6, IP6_ADDR_LEN);
        break;
#endif
    case ADDR_TYPE_IP:
        # 处理 IPv4 地址类型
        memset(&so->sin, 0, sizeof(so->sin));
#ifdef HAVE_SOCKADDR_SA_LEN
        so->sin.sin_len = sizeof(so->sin);
#endif
        so->sin.sin_family = AF_INET;
        so->sin.sin_addr.s_addr = a->addr_ip;
        break;
    default:
        errno = EINVAL;
        return (-1);
    }
    return (0);
}

# 将套接字地址结构转换为地址
int
addr_ston(const struct sockaddr *sa, struct addr *a)
{
    union sockunion *so = (union sockunion *)sa;
    
    # 初始化地址结构
    memset(a, 0, sizeof(*a));
    
    # 根据套接字地址结构的地址家族进行不同的处理
    switch (sa->sa_family) {
#ifdef HAVE_NET_IF_DL_H
# ifdef AF_LINK
    # 如果地址族是 AF_LINK
    case AF_LINK:
        # 如果硬件地址长度不等于以太网地址长度
        if (so->sdl.sdl_alen != ETH_ADDR_LEN) {
            # 设置错误码为无效参数
            errno = EINVAL;
            # 返回 -1
            return (-1);
        }
        # 设置地址类型为以太网
        a->addr_type = ADDR_TYPE_ETH;
        # 设置地址位数为以太网地址位数
        a->addr_bits = ETH_ADDR_BITS;
        # 将硬件地址拷贝到地址结构体中
        memcpy(&a->addr_eth, LLADDR(&so->sdl), ETH_ADDR_LEN);
        # 跳出循环
        break;
    # 根据不同的地址族和协议类型设置地址类型和地址位数
    case AF_UNSPEC:
    case ARP_HRD_ETH:    /* XXX- Linux arp(7) */
    case ARP_HRD_APPLETALK: /* AppleTalk DDP */
    case ARP_HRD_INFINIBAND: /* InfiniBand */
    case ARP_HDR_IEEE80211: /* IEEE 802.11 */
    case ARP_HRD_IEEE80211_PRISM: /* IEEE 802.11 + prism header */
    case ARP_HRD_IEEE80211_RADIOTAP: /* IEEE 802.11 + radiotap header */
        a->addr_type = ADDR_TYPE_ETH;  # 设置地址类型为以太网
        a->addr_bits = ETH_ADDR_BITS;   # 设置地址位数为以太网地址位数
        memcpy(&a->addr_eth, sa->sa_data, ETH_ADDR_LEN);  # 复制地址数据到以太网地址结构体
        break;
        
#ifdef AF_RAW
    case AF_RAW:        /* XXX - IRIX raw(7f) */
        a->addr_type = ADDR_TYPE_ETH;  # 设置地址类型为以太网
        a->addr_bits = ETH_ADDR_BITS;   # 设置地址位数为以太网地址位数
        memcpy(&a->addr_eth, so->sr.sr_addr, ETH_ADDR_LEN);  # 复制地址数据到以太网地址结构体
        break;
#endif
#ifdef HAVE_SOCKADDR_IN6
    case AF_INET6:
        a->addr_type = ADDR_TYPE_IP6;   # 设置地址类型为 IPv6
        a->addr_bits = IP6_ADDR_BITS;    # 设置地址位数为 IPv6 地址位数
        memcpy(&a->addr_ip6, &so->sin6.sin6_addr, IP6_ADDR_LEN);  # 复制地址数据到 IPv6 地址结构体
        break;
#endif
    case AF_INET:
        a->addr_type = ADDR_TYPE_IP;    # 设置地址类型为 IPv4
        a->addr_bits = IP_ADDR_BITS;     # 设置地址位数为 IPv4 地址位数
        a->addr_ip = so->sin.sin_addr.s_addr;  # 设置 IPv4 地址
        break;
    case ARP_HRD_VOID:
        memset(&a->addr_eth, 0, ETH_ADDR_LEN);  # 将以太网地址结构体清零
        break;
    default:
        errno = EINVAL;
        return (-1);  # 返回错误
    }
    return (0);  # 返回成功

}

int
addr_btos(uint16_t bits, struct sockaddr *sa)
{
    union sockunion *so = (union sockunion *)sa;

#ifdef HAVE_SOCKADDR_IN6
    if (bits > IP_ADDR_BITS && bits <= IP6_ADDR_BITS) {
        memset(&so->sin6, 0, sizeof(so->sin6));  # 将 IPv6 地址结构体清零
#ifdef HAVE_SOCKADDR_SA_LEN
        so->sin6.sin6_len = IP6_ADDR_LEN + (bits / 8) + (bits % 8);  # 设置 IPv6 地址结构体长度
#endif
        so->sin6.sin6_family = AF_INET6;  # 设置地址族为 IPv6
        return (addr_btom(bits, &so->sin6.sin6_addr, IP6_ADDR_LEN));  # 转换地址位数为 IPv6 地址
    } else
#endif
    if (bits <= IP_ADDR_BITS) {
        memset(&so->sin, 0, sizeof(so->sin));  # 将 IPv4 地址结构体清零
#ifdef HAVE_SOCKADDR_SA_LEN
        so->sin.sin_len = IP_ADDR_LEN + (bits / 8) + (bits % 8);  # 设置 IPv4 地址结构体长度
#endif
        # 设置套接字地址族为 IPv4
        so->sin.sin_family = AF_INET;
        # 调用函数将位表示的 IP 地址转换为二进制形式
        return (addr_btom(bits, &so->sin.sin_addr, IP_ADDR_LEN));
    }
    # 设置错误码为无效参数
    errno = EINVAL;
    return (-1);
}

int
addr_stob(const struct sockaddr *sa, uint16_t *bits)
{
    # 将套接字地址转换为联合体类型
    union sockunion *so = (union sockunion *)sa;
    int i, j, len;
    uint16_t n;
    u_char *p;

#ifdef HAVE_SOCKADDR_IN6
    # 如果套接字地址族为 IPv6
    if (sa->sa_family == AF_INET6) {
        # 将套接字地址转换为 IPv6 地址的字节流
        p = (u_char *)&so->sin6.sin6_addr;
#ifdef HAVE_SOCKADDR_SA_LEN
        # 计算套接字地址的长度
        len = sa->sa_len - ((void *) p - (void *) sa);
        # 处理套接字地址长度为 0 的特殊情况
        if (len < 0)
            len = 0;
        else if (len > IP6_ADDR_LEN)
            len = IP6_ADDR_LEN;
#else
        # 设置 IPv6 地址的长度
        len = IP6_ADDR_LEN;
#endif
    } else
#endif
    {
        # 将套接字地址转换为 IPv4 地址的字节流
        p = (u_char *)&so->sin.sin_addr.s_addr;
#ifdef HAVE_SOCKADDR_SA_LEN
        # 计算套接字地址的长度
        len = sa->sa_len - ((void *) p - (void *) sa);
        # 处理套接字地址长度为 0 的特殊情况
        if (len < 0)
            len = 0;
        else if (len > IP_ADDR_LEN)
            len = IP_ADDR_LEN;
#else
        # 设置 IPv4 地址的长度
        len = IP_ADDR_LEN;
#endif
    }
    # 遍历字节流，计算 IP 地址的位表示
    for (n = i = 0; i < len; i++, n += 8) {
        if (p[i] != 0xff)
            break;
    }
    # 如果字节流中的值不全为 1，则继续计算 IP 地址的位表示
    if (i != len && p[i]) {
        for (j = 7; j > 0; j--, n++) {
            if ((p[i] & (1 << j)) == 0)
                break;
        }
    }
    # 将计算得到的位表示赋值给 bits
    *bits = n;
    
    return (0);
}
    
int
addr_btom(uint16_t bits, void *mask, size_t size)
{
    int net, host;
    u_char *p;

    # 如果 IP 地址长度为 IPv4 地址长度
    if (size == IP_ADDR_LEN) {
        # 如果位表示超出 IPv4 地址的位数范围，设置错误码为无效参数
        if (bits > IP_ADDR_BITS) {
            errno = EINVAL;
            return (-1);
        }
        # 将位表示转换为子网掩码
        *(uint32_t *)mask = bits ?
            htonl(~(uint32_t)0 << (IP_ADDR_BITS - bits)) : 0;
    } else {
        # 如果 size * 8 小于 bits，则设置 errno 为 EINVAL，返回 -1
        if (size * 8 < bits) {
            errno = EINVAL;
            return (-1);
        }
        # 将 mask 强制转换为 u_char 指针类型
        p = (u_char *)mask;
        
        # 如果 bits 除以 8 大于 0，则将 p 所指向的内存块的前 net 个字节设置为 0xff
        if ((net = bits / 8) > 0)
            memset(p, 0xff, net);
        
        # 如果 bits 除以 8 的余数大于 0，则将 p[net] 设置为 0xff 左移 (8 - host) 位，将 p[net + 1] 到 p[size - net - 1] 设置为 0
        if ((host = bits % 8) > 0) {
            p[net] = 0xff << (8 - host);
            memset(&p[net + 1], 0, size - net - 1);
        } else
            # 否则，将 p[net] 到 p[size - net] 设置为 0
            memset(&p[net], 0, size - net);
    }
    # 返回 0
    return (0);
# 定义一个返回整数类型的函数，函数名为addr_mtob，参数为mask（指向常量的指针）、size（大小）、bits（指向uint16_t类型的指针）
int
addr_mtob(const void *mask, size_t size, uint16_t *bits)
{
    uint16_t n;  # 定义一个uint16_t类型的变量n
    u_char *p;   # 定义一个u_char类型的指针变量p
    int i, j;    # 定义两个整型变量i和j

    p = (u_char *)mask;  # 将mask强制转换为u_char类型，并赋值给指针变量p
    
    for (n = i = 0; i < (int)size; i++, n += 8) {  # 循环，初始化n和i为0，当i小于size时执行循环，每次循环i加1，n加8
        if (p[i] != 0xff)  # 如果p[i]不等于0xff
            break;         # 跳出循环
    }
    if (i != (int)size && p[i]) {  # 如果i不等于size且p[i]不为0
        for (j = 7; j > 0; j--, n++) {  # 循环，初始化j为7，当j大于0时执行循环，每次循环j减1，n加1
            if ((p[i] & (1 << j)) == 0)  # 如果p[i]与(1左移j位)的结果等于0
                break;                   # 跳出循环
        }
    }
    *bits = n;  # 将n的值赋给bits指向的变量

    return (0);  # 返回0
}
```