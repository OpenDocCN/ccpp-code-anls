# `nmap\libpcap\fad-gifc.c`

```cpp
/*
 * 声明代码的模式、缩进等信息
 */
/* 
 * 版权声明
 */
/*
 * 在源代码和二进制形式中允许修改和重新分发，但需要满足以下条件：
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 */
/*
 * 本软件由授权人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的任何责任理论下，都不会有授权人或贡献者对任何直接、间接、偶发、特殊、惩罚性或后果性的损害承担责任，包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，即使已被告知可能发生此类损害。
 */
/*
 * 如果有配置文件，包含配置文件
 */
#include <config.h>
#endif
#include <sys/param.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#ifdef HAVE_SYS_SOCKIO_H
#include <sys/sockio.h>
#endif
#include <sys/time.h>                /* concession to AIX */

struct mbuf;        /* Squelch compiler warnings on some platforms for */
struct rtentry;        /* declarations in <net/if.h> */
#include <net/if.h>
#include <netinet/in.h>

#include <errno.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <limits.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * This is fun.
 *
 * In older BSD systems, socket addresses were fixed-length, and
 * "sizeof (struct sockaddr)" gave the size of the structure.
 * All addresses fit within a "struct sockaddr".
 *
 * In newer BSD systems, the socket address is variable-length, and
 * there's an "sa_len" field giving the length of the structure;
 * this allows socket addresses to be longer than 2 bytes of family
 * and 14 bytes of data.
 *
 * Some commercial UNIXes use the old BSD scheme, some use the RFC 2553
 * variant of the old BSD scheme (with "struct sockaddr_storage" rather
 * than "struct sockaddr"), and some use the new BSD scheme.
 *
 * Some versions of GNU libc use neither scheme, but has an "SA_LEN()"
 * macro that determines the size based on the address family.  Other
 * versions don't have "SA_LEN()" (as it was in drafts of RFC 2553
 * but not in the final version).
 *
 * We assume that a UNIX that doesn't have "getifaddrs()" and doesn't have
 * SIOCGLIFCONF, but has SIOCGIFCONF, uses "struct sockaddr" for the
 * address in an entry returned by SIOCGIFCONF.
 */
#ifndef SA_LEN
#ifdef HAVE_STRUCT_SOCKADDR_SA_LEN
#define SA_LEN(addr)    ((addr)->sa_len)
#else /* HAVE_STRUCT_SOCKADDR_SA_LEN */
#define SA_LEN(addr)    (sizeof (struct sockaddr))
#endif /* HAVE_STRUCT_SOCKADDR_SA_LEN */
#endif /* SA_LEN */
/*
 * This is also fun.
 * 这也很有趣。
 *
 * There is no ioctl that returns the amount of space required for all
 * the data that SIOCGIFCONF could return, and if a buffer is supplied
 * that's not large enough for all the data SIOCGIFCONF could return,
 * on at least some platforms it just returns the data that'd fit with
 * no indication that there wasn't enough room for all the data, much
 * less an indication of how much more room is required.
 * 没有 ioctl 返回 SIOCGIFCONF 可能返回的所有数据所需的空间量，如果提供的缓冲区对于 SIOCGIFCONF 可能返回的所有数据来说不够大，在至少一些平台上，它只返回适合的数据，而没有指示没有足够的空间来存储所有数据，更不用说指示需要多少更多的空间了。
 *
 * The only way to ensure that we got all the data is to pass a buffer
 * large enough that the amount of space in the buffer *not* filled in
 * is greater than the largest possible entry.
 * 确保我们获得所有数据的唯一方法是传递一个足够大的缓冲区，使得缓冲区中未填充的空间量大于最大可能的条目。
 *
 * We assume that's "sizeof(ifreq.ifr_name)" plus 255, under the assumption
 * that no address is more than 255 bytes (on systems where the "sa_len"
 * field in a "struct sockaddr" is 1 byte, e.g. newer BSDs, that's the
 * case, and addresses are unlikely to be bigger than that in any case).
 * 我们假设这是 "sizeof(ifreq.ifr_name)" 加上 255，假设没有地址超过 255 字节（在 "struct sockaddr" 中的 "sa_len" 字段是 1 字节的系统上，例如较新的 BSD，情况是这样的，而且地址不太可能比这更大）。
 */
#define MAX_SA_LEN    255

/*
 * Get a list of all interfaces that are up and that we can open.
 * Returns -1 on error, 0 otherwise.
 * The list, as returned through "alldevsp", may be null if no interfaces
 * were up and could be opened.
 * 获取所有已启用且可以打开的接口列表。
 * 出错时返回 -1，否则返回 0。
 * 通过 "alldevsp" 返回的列表，如果没有接口已启用且可以打开，则可能为空。
 *
 * This is the implementation used on platforms that have SIOCGIFCONF but
 * don't have any other mechanism for getting a list of interfaces.
 * 这是在具有 SIOCGIFCONF 但没有任何其他机制用于获取接口列表的平台上使用的实现。
 *
 * XXX - or platforms that have other, better mechanisms but for which
 * we don't yet have code to use that mechanism; I think there's a better
 * way on Linux, for example, but if that better way is "getifaddrs()",
 * we already have that.
 * 或者具有其他更好的机制但我们尚未有用于使用该机制的代码的平台；例如，我认为在 Linux 上有更好的方法，但如果那种更好的方法是 "getifaddrs()"，那我们已经有了。
 */
int
pcap_findalldevs_interfaces(pcap_if_list_t *devlistp, char *errbuf,
    int (*check_usable)(const char *), get_if_flags_func get_flags_func)
{
    register int fd;
    register struct ifreq *ifrp, *ifend, *ifnext;
    size_t n;
    struct ifconf ifc;
    char *buf = NULL;
    unsigned buf_size;
#if defined (HAVE_SOLARIS) || defined (HAVE_HPUX10_20_OR_LATER)
    char *p, *q;
#endif
    # 定义用于获取接口信息的结构体变量
    struct ifreq ifrflags, ifrnetmask, ifrbroadaddr, ifrdstaddr;
    # 定义用于存储网络掩码、广播地址、目的地址的指针变量
    struct sockaddr *netmask, *broadaddr, *dstaddr;
    # 定义存储网络掩码、广播地址、目的地址的大小变量
    size_t netmask_size, broadaddr_size, dstaddr_size;
    # 定义返回值变量
    int ret = 0;

    '''
     * 创建一个套接字，用于获取接口列表。
     '''
    # 创建一个 AF_INET 类型的套接字
    fd = socket(AF_INET, SOCK_DGRAM, 0);
    # 如果创建套接字失败，则返回错误信息
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "socket");
        return (-1);
    }

    '''
     * 从一个 8K 缓冲区开始，并不断增大缓冲区，直到缓冲区中剩余的字节数大于 "sizeof(ifrp->ifr_name) + MAX_SA_LEN"，
     * 或者由于其他原因无法获取接口列表（假定这里的 EINVAL 意味着 "缓冲区太小"）。
     '''
    # 初始化缓冲区大小为 8192
    buf_size = 8192;
    # 循环直到满足条件或者获取接口列表失败
    for (;;) {
        '''
         * 不要让缓冲区大小超过 INT_MAX。
         '''
        # 如果缓冲区大小超过 INT_MAX，则返回错误信息
        if (buf_size > INT_MAX) {
            (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "interface information requires more than %u bytes",
                INT_MAX);
            (void)close(fd);
            return (-1);
        }
        # 分配缓冲区内存
        buf = malloc(buf_size);
        # 如果分配内存失败，则返回错误信息
        if (buf == NULL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            (void)close(fd);
            return (-1);
        }

        # 设置 ifc 结构体的长度和缓冲区
        ifc.ifc_len = buf_size;
        ifc.ifc_buf = buf;
        # 将缓冲区清零
        memset(buf, 0, buf_size);
        # 如果获取接口列表失败且错误码不是 EINVAL，则返回错误信息
        if (ioctl(fd, SIOCGIFCONF, (char *)&ifc) < 0
            && errno != EINVAL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "SIOCGIFCONF");
            (void)close(fd);
            free(buf);
            return (-1);
        }
        # 如果缓冲区剩余空间大于 "sizeof(ifrp->ifr_name) + MAX_SA_LEN"，则跳出循环
        if (ifc.ifc_len < (int)buf_size &&
            (buf_size - ifc.ifc_len) > sizeof(ifrp->ifr_name) + MAX_SA_LEN)
            break;
        # 释放缓冲区内存
        free(buf);
        # 缓冲区大小翻倍
        buf_size *= 2;
    }

    # 将 buf 强制转换为 struct ifreq 类型的指针
    ifrp = (struct ifreq *)buf;
    # 计算接口列表的结束位置
    ifend = (struct ifreq *)(buf + ifc.ifc_len);
#if defined (HAVE_SOLARIS) || defined (HAVE_HPUX10_20_OR_LATER)
        /*
         * 如果此条目以冒号后跟数字结尾，则它是一个逻辑接口。这些只是用于将多个IP地址分配给真实接口的方式，因此逻辑接口的条目应该像真实接口的条目一样处理；我们通过去掉“：”和数字来做到这一点。
         */
        p = strchr(ifrp->ifr_name, ':');
        if (p != NULL) {
            /*
             * 我们有一个“：”；它后面跟着一个数字吗？
             */
            q = p + 1;
            while (PCAP_ISDIGIT(*q))
                q++;
            if (*q == '\0') {
                /*
                 * “：”后面都是数字。去掉“：”和它后面的所有内容。
                 */
                *p = '\0';
            }
        }
#endif

        /*
         * 将此地址的信息添加到列表中。
         */
        if (add_addr_to_if(devlistp, ifrp->ifr_name,
            ifrflags.ifr_flags, get_flags_func,
            &ifrp->ifr_addr, SA_LEN(&ifrp->ifr_addr),
            netmask, netmask_size, broadaddr, broadaddr_size,
            dstaddr, dstaddr_size, errbuf) < 0) {
            ret = -1;
            break;
        }
    }
    free(buf);
    (void)close(fd);

    return (ret);
}
```