# `nmap\libpcap\pcap\sll.h`

```cpp
/*
 * 对于在 Linux cooked sockets 上的捕获，我们构造一个假的头部
 * 包括：
 *
 *    2字节的“数据包类型”，可以是以下之一：
 *
 *        LINUX_SLL_HOST        数据包发送给我们
 *        LINUX_SLL_BROADCAST    数据包广播
 *        LINUX_SLL_MULTICAST    数据包组播
 *        LINUX_SLL_OTHERHOST    数据包发送给其他人
 *        LINUX_SLL_OUTGOING    数据包由我们发送；
 *
 *    2字节的以太网协议字段；
 *
 *    2字节的链路层类型；
 *
 *    2字节的链路层地址长度；
 *
 *    8字节的源链路层地址，其实际长度由前一个值指定。
 *
 * 除了链路层地址之外的所有字段都采用网络字节顺序。
 *
 * 不要更改此结构的布局，也不要更改下面的任何 LINUX_SLL_ 值。如果必须更改“cooked”Linux捕获的链路层头部，请引入一个新的 DLT_ 类型（向“tcpdump-workers@lists.tcpdump.org”请求一个，以便不会给它赋予已经使用的值），并在该类型的捕获中使用新的头部，以便能够处理 DLT_LINUX_SLL 捕获的程序将继续正确处理它们，而无需进行任何更改，并且可以区分具有不同头部的捕获文件，并且可以解析其中的数据包的程序。
 */

#ifndef lib_pcap_sll_h
#define lib_pcap_sll_h

#include <pcap/pcap-inttypes.h>

/*
 * 一个 DLT_LINUX_SLL 假的链路层头部。
 */
#define SLL_HDR_LEN    16        /* 总头部长度 */
#define SLL_ADDRLEN    8        /* 地址字段的长度 */

struct sll_header {
    uint16_t sll_pkttype;        /* 数据包类型 */
    uint16_t sll_hatype;        /* 链路层地址类型 */
    uint16_t sll_halen;        /* 链路层地址长度 */
    uint8_t  sll_addr[SLL_ADDRLEN];    /* 链路层地址 */
    uint16_t sll_protocol;        /* 协议 */
};
/*
 * A DLT_LINUX_SLL2 fake link-layer header.
 */
# 定义一个假的 DLT_LINUX_SLL2 链路层头部

#define SLL2_HDR_LEN    20        /* total header length */
# 定义 SLL2 头部的总长度为 20

struct sll2_header {
    uint16_t sll2_protocol;            /* protocol */
    uint16_t sll2_reserved_mbz;        /* reserved - must be zero */
    uint32_t sll2_if_index;            /* 1-based interface index */
    uint16_t sll2_hatype;            /* link-layer address type */
    uint8_t  sll2_pkttype;            /* packet type */
    uint8_t  sll2_halen;            /* link-layer address length */
    uint8_t  sll2_addr[SLL_ADDRLEN];    /* link-layer address */
};
# 定义了一个结构体 sll2_header，包含了各种字段用于描述链路层头部的信息

/*
 * The LINUX_SLL_ values for "sll_pkttype" and LINUX_SLL2_ values for
 * "sll2_pkttype"; these correspond to the PACKET_ values on Linux,
 * which are defined by a header under include/uapi in the current
 * kernel source, and are thus not going to change on Linux.  We
 * define them here so that they're available even on systems other
 * than Linux.
 */
# 定义了 LINUX_SLL_ 和 LINUX_SLL2_ 的值，对应于 Linux 上的 PACKET_ 值，这些值在当前内核源码的 include/uapi 中定义，因此在 Linux 上不会改变。我们在这里定义它们，以便它们在非 Linux 系统上也可用。

#define LINUX_SLL_HOST        0
#define LINUX_SLL_BROADCAST    1
#define LINUX_SLL_MULTICAST    2
#define LINUX_SLL_OTHERHOST    3
#define LINUX_SLL_OUTGOING    4
# 定义了 LINUX_SLL_ 的几个值，用于描述 sll_pkttype 和 sll2_pkttype 的不同情况
/*
 * 定义了 LINUX_SLL_ 中的 "sll_protocol" 和 LINUX_SLL2_ 中的 "sll2_protocol" 值；
 * 这些值对应于 Linux 上的 ETH_P_ 值，但在这里定义，以便它们即使在非 Linux 系统上也是可用的。
 * 我们目前假设 ETH_P_ 值在 Linux 中不会改变；如果它们改变了，那么：
 *
 *    如果我们在 "pcap-linux.c" 中不对它们进行转换，那么在定义了与这些值不匹配的 ETH_P_ 值的系统上捕获的捕获文件可能无法读取；
 *
 *    如果我们在 "pcap-linux.c" 中对它们进行转换，那么这将使 BPF 代码生成器的生活变得不愉快，因为在内核中测试的值并不是在读取捕获文件时测试的值，因此在传递给内核的 BPF 程序上运行的修复代码最终需要做更多的工作。
 *
 * 根据需要在这里添加其他值，以处理可能出现在非以太网、非 802.x 网络上的数据包类型。（我怀疑 Linux 的 "if_ether.h" 中并不是所有的值都会出现在捕获中。）
 */
#define LINUX_SLL_P_802_3    0x0001    /* 不带 802.2 LLC 头的 Novell 802.3 帧 */
#define LINUX_SLL_P_802_2    0x0004    /* 802.2 帧（不是 D/I/X 以太网） */
#define LINUX_SLL_P_CAN        0x000C    /* 带有 SocketCAN 伪头的 CAN 帧 */
#define LINUX_SLL_P_CANFD    0x000D    /* 带有 SocketCAN 伪头的 CAN FD 帧 */

#endif
```