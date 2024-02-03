# `nmap\nbase\inet_pton.c`

```cpp
/*
 * 版权所有（c）1996年由Internet Software Consortium。
 *
 * 在此授予使用、复制、修改和分发此软件的权限，无论是否收取费用，前提是上述版权声明和此许可声明出现在所有副本中。
 *
 * 本软件按“原样”提供，Internet Software Consortium不对本软件的所有方面包括所有暗示的适销性和适用性的任何暗示的保证负责。在任何情况下，Internet Software Consortium均不对因使用或执行本软件而产生的任何特殊、直接、间接或后果性损害或任何损害负责，无论是在合同诉讼、疏忽或其他侵权行为中产生的，与使用或执行本软件有关或与本软件的性能有关。
 */

/* 由Fyodor（fyodor@nmap.org）修改，用于Nmap安全扫描仪的包含。
 *
 * $Id$
 */

#include "nbase.h"

#ifndef HAVE_INET_PTON

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
#if HAVE_STRING_H
#include <string.h>
#endif
#if HAVE_ERRNO_H
#include <errno.h>
#endif

#ifndef IN6ADDRSZ
#define IN6ADDRSZ   16
#endif

#ifndef INT16SZ
#define INT16SZ 2
#endif

#ifndef INADDRSZ
#define INADDRSZ    4
#endif

#if !defined(EAFNOSUPPORT) && defined(WSAEAFNOSUPPORT)
#define EAFNOSUPPORT WSAEAFNOSUPPORT
#endif

/*
 * 警告：甚至不要考虑在sizeof(int) < 4的系统上编译这个程序。sizeof(int) > 4没问题；世界并非都是VAX。
 */

// 将IPv4地址从文本格式转换为二进制格式
static int    inet_pton4 (const char *src, unsigned char *dst);
#if HAVE_IPV6
// 将IPv6地址从文本格式转换为二进制格式
static int    inet_pton6 (const char *src, unsigned char *dst);
#endif
/* int
 * inet_pton(af, src, dst)
 *    将表示格式（通常是ASCII可打印字符）转换为网络格式（通常是某种二进制格式）。
 * return:
 *    如果地址对指定的地址族有效，则返回1
 *    如果地址无效，则返回0（在这种情况下，`dst'不变）
 *    如果发生其他错误，则返回-1（在这种情况下，`dst'也不变）
 * author:
 *    Paul Vixie, 1996.
 */
int
inet_pton(int af, const char *src, void *dst)
{
    switch (af) {
    case AF_INET:
        return (inet_pton4(src, (unsigned char *) dst));
#if HAVE_IPV6
    case AF_INET6:
        return (inet_pton6(src, (unsigned char *) dst));
#endif
    default:
#ifndef WIN32
        errno = EAFNOSUPPORT;
#endif
        return (-1);
    }
    /* NOTREACHED */
}

/* int
 * inet_pton4(src, dst)
 *    类似于inet_aton()，但没有十六进制和简写。
 * return:
 *    如果`src'是有效的点分十进制格式，则返回1，否则返回0。
 * notice:
 *    除非返回1，否则不会影响`dst'。
 * author:
 *    Paul Vixie, 1996.
 */
static int
inet_pton4(const char *src, unsigned char *dst)
{
    static const char digits[] = "0123456789";
    int saw_digit, octets, ch;
    unsigned char tmp[INADDRSZ], *tp;

    saw_digit = 0;
    octets = 0;
    *(tp = tmp) = 0;
    while ((ch = *src++) != '\0') {
        const char *pch;

        if ((pch = strchr(digits, ch)) != NULL) {
            unsigned int newval = (unsigned int) (*tp * 10 + (pch - digits));

            if (newval > 255)
                return (0);
            *tp = newval;
            if (! saw_digit) {
                if (++octets > 4)
                    return (0);
                saw_digit = 1;
            }
        } else if (ch == '.' && saw_digit) {
            if (octets == 4)
                return (0);
            *++tp = 0;
            saw_digit = 0;
        } else
            return (0);
    }
    if (octets < 4)
        return (0);

    memcpy(dst, tmp, INADDRSZ);
    # 返回整数1
    return (1);
# 如果系统支持 IPv6
#if HAVE_IPV6
/* int
 * inet_pton6(src, dst)
 *    将表示级别的地址转换为网络顺序的二进制形式。
 * return:
 *    如果 `src' 是有效的[RFC1884 2.2]地址，则返回1，否则返回0。
 * notice:
 *    (1) 除非返回1，否则不会触及 `dst'。
 *    (2) 完整地址中的 :: 会被静默忽略。
 * credit:
 *    受 Mark Andrews 启发。
 * author:
 *    Paul Vixie, 1996.
 */
static int
inet_pton6(const char *src, unsigned char *dst)
{
    static const char xdigits_l[] = "0123456789abcdef",
              xdigits_u[] = "0123456789ABCDEF";
    unsigned char tmp[IN6ADDRSZ], *tp, *endp, *colonp;
    const char *xdigits, *curtok;
    int ch, saw_xdigit;
    unsigned int val;

    memset((tp = tmp), '\0', IN6ADDRSZ);
    endp = tp + IN6ADDRSZ;
    colonp = NULL;
    /* Leading :: requires some special handling. */
    if (*src == ':')
        if (*++src != ':')
            return (0);
    curtok = src;
    saw_xdigit = 0;
    val = 0;
    # 当源字符串的字符不为'\0'时，执行循环
    while ((ch = *src++) != '\0') {
        const char *pch;

        # 在xdigits_l中查找ch字符，如果找不到则在xdigits_u中查找
        if ((pch = strchr((xdigits = xdigits_l), ch)) == NULL)
            pch = strchr((xdigits = xdigits_u), ch);
        # 如果pch不为空
        if (pch != NULL) {
            # 将val左移4位，并将pch-xdigits的值存入val
            val <<= 4;
            val |= (pch - xdigits);
            # 如果val大于0xffff，返回0
            if (val > 0xffff)
                return (0);
            saw_xdigit = 1;
            continue;
        }
        # 如果ch为':'时
        if (ch == ':') {
            curtok = src;
            # 如果之前没有遇到十六进制数字，且已经遇到过冒号，返回0
            if (!saw_xdigit) {
                if (colonp)
                    return (0);
                colonp = tp;
                continue;
            }
            # 如果tp+INT16SZ大于endp，返回0
            if (tp + INT16SZ > endp)
                return (0);
            # 将val的高8位和低8位分别存入tp指向的位置
            *tp++ = (unsigned char) (val >> 8) & 0xff;
            *tp++ = (unsigned char) val & 0xff;
            saw_xdigit = 0;
            val = 0;
            continue;
        }
        # 如果ch为'.'，且tp+INADDRSZ小于等于endp，并且inet_pton4返回大于0
        if (ch == '.' && ((tp + INADDRSZ) <= endp) &&
            inet_pton4(curtok, tp) > 0) {
            # tp向后移动INADDRSZ
            tp += INADDRSZ;
            saw_xdigit = 0;
            break;    /* '\0' was seen by inet_pton4(). */
        }
        # 返回0
        return (0);
    }
    # 如果saw_xdigit为真
    if (saw_xdigit) {
        # 如果tp+INT16SZ大于endp，返回0
        if (tp + INT16SZ > endp)
            return (0);
        # 将val的高8位和低8位分别存入tp指向的位置
        *tp++ = (unsigned char) (val >> 8) & 0xff;
        *tp++ = (unsigned char) val & 0xff;
    }
    # 如果colonp不为空
    if (colonp != NULL) {
        # 计算tp-colonp的值
        const int n = tp - colonp;
        int i;

        # 将endp的倒数n个字符向后移动n位
        for (i = 1; i <= n; i++) {
            endp[- i] = colonp[n - i];
            colonp[n - i] = 0;
        }
        # tp指向endp
        tp = endp;
    }
    # 如果tp不等于endp，返回0
    if (tp != endp)
        return (0);
    # 将tmp的内容拷贝到dst中
    memcpy(dst, tmp, IN6ADDRSZ);
    # 返回1
    return (1);
}
# 结束 if 条件判断的代码块
#endif
# 结束 if 条件判断的代码块
#endif /* ifndef HAVE_INET_PTON */
# 结束 if 条件判断的代码块，并且定义了 HAVE_INET_PTON 的宏未定义时执行
```