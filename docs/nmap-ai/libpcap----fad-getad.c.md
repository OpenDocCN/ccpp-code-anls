# `nmap\libpcap\fad-getad.c`

```
/*
 * 设置代码的模式为 C，制表符宽度为 8，缩进使用制表符，基本偏移量为 8
 * 版权声明，版权归加利福尼亚大学校董会所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有宣传材料提及此软件的特性或使用必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 * 本软件由校董和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，校董或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害
 */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#include <net/if.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ifaddrs.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * 在 Solaris 11 及更高版本上不需要这样做，因为似乎接口上没有 AF_PACKET 地址，所以我们不需要这个，
 * 并且我们最终会包含操作系统的 <net/bpf.h> 和我们的 <pcap/bpf.h>，它们的一些数据结构的定义会发生冲突。
 */
#if (defined(linux) || defined(__Lynx__)) && defined(AF_PACKET)
# ifdef HAVE_NETPACKET_PACKET_H
/* 具有较新 glibc 的 Linux 发行版 */
#  include <netpacket/packet.h>
# else /* HAVE_NETPACKET_PACKET_H */
/* LynxOS，具有较旧 glibc 的 Linux 发行版 */
# ifdef __Lynx__
/* LynxOS */
#  include <netpacket/if_packet.h>
# else /* __Lynx__ */
/* Linux */
#  include <linux/types.h>
#  include <linux/if_packet.h>
# endif /* __Lynx__ */
# endif /* HAVE_NETPACKET_PACKET_H */
#endif /* (defined(linux) || defined(__Lynx__)) && defined(AF_PACKET) */
#ifndef SA_LEN
#ifdef HAVE_STRUCT_SOCKADDR_SA_LEN
#define SA_LEN(addr)    ((addr)->sa_len)
#else /* HAVE_STRUCT_SOCKADDR_SA_LEN */
#ifdef HAVE_STRUCT_SOCKADDR_STORAGE
static size_t
get_sa_len(struct sockaddr *addr)
{
    switch (addr->sa_family) {

#ifdef AF_INET
    case AF_INET:
        return (sizeof (struct sockaddr_in));
#endif

#ifdef AF_INET6
    case AF_INET6:
        return (sizeof (struct sockaddr_in6));
#endif

#if (defined(linux) || defined(__Lynx__)) && defined(AF_PACKET)
    case AF_PACKET:
        return (sizeof (struct sockaddr_ll));
#endif

    default:
        return (sizeof (struct sockaddr));
    }
}
#define SA_LEN(addr)    (get_sa_len(addr))
#else /* HAVE_STRUCT_SOCKADDR_STORAGE */
#define SA_LEN(addr)    (sizeof (struct sockaddr))
#endif /* HAVE_STRUCT_SOCKADDR_STORAGE */
#endif /* HAVE_STRUCT_SOCKADDR_SA_LEN */
#endif /* SA_LEN */
/*
 * 获取所有已启用且可打开的接口列表。
 * 在错误时返回-1，否则返回0。
 * 通过 "alldevsp" 返回的列表可能为空，如果没有可以打开的接口。
 */
int
pcap_findalldevs_interfaces(pcap_if_list_t *devlistp, char *errbuf,
    int (*check_usable)(const char *), get_if_flags_func get_flags_func)
{
    // 定义变量
    struct ifaddrs *ifap, *ifa;
    struct sockaddr *addr, *netmask, *broadaddr, *dstaddr;
    size_t addr_size, broadaddr_size, dstaddr_size;
    int ret = 0;
    char *p, *q;

    /*
     * 获取接口地址列表。
     *
     * 注意：这不会返回没有地址的接口的信息，因此，如果平台上有没有可以捕获流量的接口，
     * 我们必须检查这些接口（例如，在 Linux 上所做的）。
     *
     * 局域网接口可能会有链路层地址；我不知道所有的 "getifaddrs()" 实现现在或将来是否会返回这些地址。
     */
    if (getifaddrs(&ifap) != 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "getifaddrs");
        return (-1);
    }
    }

    // 释放接口地址列表
    freeifaddrs(ifap);

    return (ret);
}
```