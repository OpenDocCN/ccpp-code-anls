# `nmap\libpcap\fad-glifc.c`

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
#include <sys/param.h>  // 包含系统参数头文件
#include <sys/file.h>  // 包含文件操作头文件
#include <sys/ioctl.h>  // 包含输入输出控制头文件
#include <sys/socket.h>  // 包含套接字头文件
#ifdef HAVE_SYS_SOCKIO_H
#include <sys/sockio.h>  // 如果定义了 HAVE_SYS_SOCKIO_H，则包含套接字IO头文件
#endif
#include <sys/time.h>                /* concession to AIX */  // 包含时间头文件，适用于AIX系统

struct mbuf;        /* Squelch compiler warnings on some platforms for */  // 定义结构体 mbuf，用于抑制某些平台上的编译器警告
struct rtentry;        /* declarations in <net/if.h> */  // 定义结构体 rtentry，用于在 <net/if.h> 中声明
#include <net/if.h>  // 包含网络接口头文件
#include <netinet/in.h>  // 包含互联网地址族头文件

#include <errno.h>  // 包含错误码头文件
#include <memory.h>  // 包含内存操作头文件
#include <stdio.h>  // 包含标准输入输出头文件
#include <stdlib.h>  // 包含标准库头文件
#include <string.h>  // 包含字符串操作头文件
#include <unistd.h>  // 包含标准符号常量和类型头文件

#include "pcap-int.h"  // 包含 pcap 内部头文件

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"  // 如果定义了 HAVE_OS_PROTO_H，则包含操作系统协议头文件
#endif

/*
 * Get a list of all interfaces that are up and that we can open.
 * Returns -1 on error, 0 otherwise.
 * The list, as returned through "alldevsp", may be null if no interfaces
 * were up and could be opened.
 *
 * This is the implementation used on platforms that have SIOCGLIFCONF
 * but don't have "getifaddrs()".  (Solaris 8 and later; we use
 * SIOCGLIFCONF rather than SIOCGIFCONF in order to get IPv6 addresses.)
 */
int
pcap_findalldevs_interfaces(pcap_if_list_t *devlistp, char *errbuf,
    int (*check_usable)(const char *), get_if_flags_func get_flags_func)
{
    register int fd4, fd6, fd;  // 声明整型变量 fd4, fd6, fd
    register struct lifreq *ifrp, *ifend;  // 声明指向结构体 lifreq 的指针变量 ifrp, ifend
    struct lifnum ifn;  // 声明结构体 lifnum 变量 ifn
    struct lifconf ifc;  // 声明结构体 lifconf 变量 ifc
    char *buf = NULL;  // 声明字符指针变量 buf，并初始化为 NULL
    unsigned buf_size;  // 声明无符号整型变量 buf_size
#ifdef HAVE_SOLARIS
    char *p, *q;  // 如果定义了 HAVE_SOLARIS，则声明字符指针变量 p, q
#endif
    struct lifreq ifrflags, ifrnetmask, ifrbroadaddr, ifrdstaddr;  // 声明结构体 lifreq 变量 ifrflags, ifrnetmask, ifrbroadaddr, ifrdstaddr
    struct sockaddr *netmask, *broadaddr, *dstaddr;  // 声明指向结构体 sockaddr 的指针变量 netmask, broadaddr, dstaddr
    int ret = 0;  // 声明整型变量 ret，并初始化为 0

    /*
     * Create a socket from which to fetch the list of interfaces,
     * and from which to fetch IPv4 information.
     */
    fd4 = socket(AF_INET, SOCK_DGRAM, 0);  // 创建一个用于获取接口列表和 IPv4 信息的套接字
    if (fd4 < 0) {  // 如果套接字创建失败
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,  // 格式化错误消息
            errno, "socket: AF_INET");
        return (-1);  // 返回错误码 -1
    }

    /*
     * Create a socket from which to fetch IPv6 information.
     */
    fd6 = socket(AF_INET6, SOCK_DGRAM, 0);  // 创建一个用于获取 IPv6 信息的套接字
    # 如果 fd6 小于 0，表示出错
    if (fd6 < 0) {
        # 格式化错误消息，包含错误码和描述信息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "socket: AF_INET6");
        # 关闭 fd4
        (void)close(fd4);
        # 返回错误
        return (-1);
    }

    /*
     * SIOCGLIFCONF 会返回多少个条目？
     */
    # 设置 ifn 结构体的属性
    ifn.lifn_family = AF_UNSPEC;
    ifn.lifn_flags = 0;
    ifn.lifn_count = 0;
    # 调用 ioctl 函数获取条目数量
    if (ioctl(fd4, SIOCGLIFNUM, (char *)&ifn) < 0) {
        # 格式化错误消息，包含错误码和描述信息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGLIFNUM");
        # 关闭 fd6
        (void)close(fd6);
        # 关闭 fd4
        (void)close(fd4);
        # 返回错误
        return (-1);
    }

    /*
     * 为这些条目分配一个缓冲区。
     */
    # 计算缓冲区大小
    buf_size = ifn.lifn_count * sizeof (struct lifreq);
    # 分配缓冲区
    buf = malloc(buf_size);
    # 如果分配失败，返回错误
    if (buf == NULL) {
        # 格式化错误消息，包含错误码和描述信息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        # 关闭 fd6
        (void)close(fd6);
        # 关闭 fd4
        (void)close(fd4);
        # 返回错误
        return (-1);
    }

    /*
     * 获取这些条目。
     */
    # 设置 ifc 结构体的属性
    ifc.lifc_len = buf_size;
    ifc.lifc_buf = buf;
    ifc.lifc_family = AF_UNSPEC;
    ifc.lifc_flags = 0;
    # 将缓冲区清零
    memset(buf, 0, buf_size);
    # 调用 ioctl 函数获取条目
    if (ioctl(fd4, SIOCGLIFCONF, (char *)&ifc) < 0) {
        # 格式化错误消息，包含错误码和描述信息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGLIFCONF");
        # 关闭 fd6
        (void)close(fd6);
        # 关闭 fd4
        (void)close(fd4);
        # 释放缓冲区
        free(buf);
        # 返回错误
        return (-1);
    }

    /*
     * 遍历这些条目。
     */
    # 将 buf 转换为 struct lifreq 指针
    ifrp = (struct lifreq *)buf;
    # 计算最后一个条目的地址
    ifend = (struct lifreq *)(buf + ifc.lifc_len);
#ifdef HAVE_SOLARIS
        /*
         * 如果这个条目的名称末尾有一个冒号后面跟着一个数字，那么它是一个逻辑接口。
         * 这些是用来给真实接口分配多个 IP 地址的方式，因此逻辑接口的条目应该
         * 像真实接口的条目一样处理；我们通过去掉冒号和数字来实现这一点。
         */
        p = strchr(ifrp->lifr_name, ':');
        if (p != NULL) {
            /*
             * 我们有一个冒号；它后面跟着一个数字吗？
             */
            q = p + 1;
            while (PCAP_ISDIGIT(*q))
                q++;
            if (*q == '\0') {
                /*
                 * 冒号后面都是数字。去掉冒号和后面的所有内容。
                 */
                *p = '\0';
            }
        }
#endif

        /*
         * 将此地址的信息添加到列表中。
         */
        if (add_addr_to_if(devlistp, ifrp->lifr_name,
            ifrflags.lifr_flags, get_flags_func,
            (struct sockaddr *)&ifrp->lifr_addr,
            sizeof (struct sockaddr_storage),
            netmask, sizeof (struct sockaddr_storage),
            broadaddr, sizeof (struct sockaddr_storage),
            dstaddr, sizeof (struct sockaddr_storage), errbuf) < 0) {
            ret = -1;
            break;
        }
    }
    free(buf);
    (void)close(fd6);
    (void)close(fd4);

    return (ret);
}
```