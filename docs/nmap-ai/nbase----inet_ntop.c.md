# `nmap\nbase\inet_ntop.c`

```
/*
 * 1996年由Internet Software Consortium版权所有。
 * 
 * 在所有副本中，无论是否收费，都可以使用、复制、修改和分发此软件，只要包含上述版权声明和此许可声明。
 * 
 * 本软件按“原样”提供，Internet Software Consortium不承担与本软件有关的所有担保责任，包括所有暗示的担保责任，无论是适销性还是适用性。在任何情况下，Internet Software Consortium均不对使用或执行本软件的结果承担任何特殊、直接、间接或后果性的损害或任何损害责任，无论是因合同、疏忽或其他侵权行为，还是与使用或执行本软件有关。
 */

/* 由Fyodor (fyodor@nmap.org)修改，用于Nmap安全扫描器的包含。
 *
 * $Id$
 */

#include "nbase.h"

#ifndef HAVE_INET_NTOP

#if HAVE_SYS_TYPES_H
#include <sys/types.h>
#endif
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>
#endif
#if HAVE_ARPA_INET_H
#include <arpa/inet.h>
#endif
#include <string.h>
#if HAVE_ERRNO_H
#include <errno.h>
#endif
#include <stdio.h>

#ifndef IN6ADDRSZ
#define IN6ADDRSZ   16
#endif

#ifndef INT16SZ
#define INT16SZ sizeof(u16)
#endif

#if !defined(EAFNOSUPPORT) && defined(WSAEAFNOSUPPORT)
#define EAFNOSUPPORT WSAEAFNOSUPPORT
#endif

/*
 * 警告：甚至不要考虑在sizeof(int) < 4的系统上编译这个程序。sizeof(int) > 4没问题；世界上并非所有都是VAX。
 */

// 将IPv4地址转换为点分十进制字符串
static const char *inet_ntop4(const unsigned char *src, char *dst, size_t size);
#if HAVE_IPV6
// 将IPv6地址转换为点分十进制字符串
static const char *inet_ntop6(const unsigned char *src, char *dst, size_t size);
#endif
/* char *
 * inet_ntop(af, src, dst, size)
 *        convert a network format address to presentation format.
 * return:
 *        pointer to presentation format address (`dst'), or NULL (see errno).
 * author:
 *        Paul Vixie, 1996.
 */
const char *
inet_ntop(int af, const void *src, char *dst, size_t size)
{
        switch (af) {
        case AF_INET:
                return (inet_ntop4((const unsigned char *) src, dst, size));
#if HAVE_IPV6
        case AF_INET6:
                return (inet_ntop6((const unsigned char *) src, dst, size));
#endif
        default:
#ifndef WIN32
                errno = EAFNOSUPPORT;
#endif
                return (NULL);
        }
        /* NOTREACHED */
}

/* const char *
 * inet_ntop4(src, dst, size)
 *        format an IPv4 address, more or less like inet_ntoa()
 * return:
 *        `dst' (as a const)
 * notes:
 *        (1) uses no statics
 *        (2) takes a u_char* not an in_addr as input
 * author:
 *        Paul Vixie, 1996.
 */
static const char *
inet_ntop4(const unsigned char *src, char *dst, size_t size)
{
        const size_t MIN_SIZE = 16; /* space for 255.255.255.255\0 */
        int n = 0;
        char *next = dst;

        if (size < MIN_SIZE) {
#ifndef WIN32
            errno = ENOSPC;
#endif
            return NULL;
        }
        do {
            unsigned char u = *src++;
            if (u > 99) {
                *next++ = '0' + u/100;
                u %= 100;
                *next++ = '0' + u/10;
                u %= 10;
            }
            else if (u > 9) {
                *next++ = '0' + u/10;
                u %= 10;
            }
            *next++ = '0' + u;
            *next++ = '.';
            n++;
        } while (n < 4);
        *--next = 0;
        return dst;
}

#if HAVE_IPV6
/* const char *
 * inet_ntop6(src, dst, size)
 *        convert IPv6 binary address into presentation (printable) format
 * author:
 *        Paul Vixie, 1996.
 */
static const char *
    # 定义一个临时数组，用于存储格式化后的 IPv6 地址
    char tmp[sizeof "ffff:ffff:ffff:ffff:ffff:ffff:255.255.255.255"], *tp;
    # 定义一个结构体，用于存储最长的连续 0 的起始位置和长度
    struct { int base, len; } best, cur;
    # 定义一个数组，用于存储 IPv6 地址的每个 16 位的值
    u16 words[IN6ADDRSZ / INT16SZ];
    # 定义一个整型变量 i，用于循环计数
    int i;
    # 定义指向源地址的指针和源地址的结束位置
    const unsigned char *next_src, *src_end;
    # 定义指向目标地址的指针
    u16 *next_dest;

    # 将输入的字节流数组转换为 16 位的数组，并找到最长的连续 0 的起始位置和长度
    next_src = src;
    src_end = src + IN6ADDRSZ;
    next_dest = words;
    best.base = -1;
    cur.base = -1;
    i = 0;
    do {
        unsigned int next_word = (unsigned int)*next_src++;
        next_word <<= 8;
        next_word |= (unsigned int)*next_src++;
        *next_dest++ = next_word;

        if (next_word == 0) {
            if (cur.base == -1) {
                cur.base = i;
                cur.len = 1;
            }
            else {
                cur.len++;
            }
        } else {
            if (cur.base != -1) {
                if (best.base == -1 || cur.len > best.len) {
                    best = cur;
                }
                cur.base = -1;
            }
        }

        i++;
    } while (next_src < src_end);

    if (cur.base != -1) {
        if (best.base == -1 || cur.len > best.len) {
            best = cur;
        }
    }
    if (best.base != -1 && best.len < 2) {
        best.base = -1;
    }

    # 格式化结果的指针指向临时数组
    tp = tmp;
    for (i = 0; i < (IN6ADDRSZ / INT16SZ);) {
        /* 循环遍历IPv6地址的16位整数数组 */
        /* Are we inside the best run of 0x00's? */
        /* 检查是否在最佳的0x00连续段内 */
        if (i == best.base) {
            *tp++ = ':';
            i += best.len;
            continue;
        }
        /* Are we following an initial run of 0x00s or any real hex? */
        /* 检查是否在初始的0x00连续段后或者是真实的十六进制数 */
        if (i != 0) {
            *tp++ = ':';
        }
        /* Is this address an encapsulated IPv4? */
        /* 检查是否是封装的IPv4地址 */
        if (i == 6 && best.base == 0 &&
            (best.len == 6 || (best.len == 5 && words[5] == 0xffff))) {
            if (!inet_ntop4(src+12, tp, sizeof tmp - (tp - tmp))) {
                return (NULL);
            }
            tp += strlen(tp);
            break;
        }
        tp += Snprintf(tp, sizeof tmp - (tp - tmp), "%x", words[i]);
        i++;
    }
    /* Was it a trailing run of 0x00's? */
    /* 检查是否是0x00的尾随连续段 */
    if (best.base != -1 && (best.base + best.len) == (IN6ADDRSZ / INT16SZ)) {
        *tp++ = ':';
    }
    *tp++ = '\0';

    /*
     * Check for overflow, copy, and we're done.
     */
    /* 检查是否溢出，复制，然后结束 */
    if ((size_t)(tp - tmp) > size) {
#ifndef WIN32
        // 如果不是在 Windows 平台下，设置错误码为 ENOSPC
        errno = ENOSPC;
#endif
        // 返回空指针
        return (NULL);
    }
    // 将 tmp 拷贝到 dst 中，最多拷贝 size 个字符
    strncpy(dst, tmp, size);
    // 返回 dst
    return (dst);
}
#endif

#endif /* ifndef HAVE_INET_NTOP */
```