# `nmap\libpcap\pcap\dlt.h`

```cpp
# 版权声明，版权归加利福尼亚大学所有
# 代码来源于Stanford/CMU enet数据包过滤器(net/enet.c)，作为4.3BSD的一部分分发，由Steven McCanne和Van Jacobson在劳伦斯伯克利实验室贡献代码给伯克利
# 允许以源代码和二进制形式重新分发和使用，但需要满足以下条件：
# 1. 源代码的再分发必须保留上述版权声明、条件列表和下面的免责声明
# 2. 二进制形式的再分发必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和下面的免责声明
# 3. 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品
# 本软件由大学和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权(包括疏忽或其他方式)的任何责任理论，都不会有大学或贡献者对任何直接、间接、附带、特殊、惩罚性或后果性的损害承担责任，包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，即使已被告知可能发生此类损害
# bpf.h的版本信息
#ifndef lib_pcap_dlt_h
#define lib_pcap_dlt_h
/*
 * Link-layer header type codes.
 *
 * Do *NOT* add new values to this list without asking
 * "tcpdump-workers@lists.tcpdump.org" for a value.  Otherwise, you run
 * the risk of using a value that's already being used for some other
 * purpose, and of having tools that read libpcap-format captures not
 * being able to handle captures with your new DLT_ value, with no hope
 * that they will ever be changed to do so (as that would destroy their
 * ability to read captures using that value for that other purpose).
 *
 * See
 *
 *    https://www.tcpdump.org/linktypes.html
 *
 * for detailed descriptions of some of these link-layer header types.
 */

/*
 * These are the types that are the same on all platforms, and that
 * have been defined by <net/bpf.h> for ages.
 */
#define DLT_NULL    0    /* BSD loopback encapsulation */
#define DLT_EN10MB    1    /* Ethernet (10Mb) */
#define DLT_EN3MB    2    /* Experimental Ethernet (3Mb) */
#define DLT_AX25    3    /* Amateur Radio AX.25 */
#define DLT_PRONET    4    /* Proteon ProNET Token Ring */
#define DLT_CHAOS    5    /* Chaos */
#define DLT_IEEE802    6    /* 802.5 Token Ring */
#define DLT_ARCNET    7    /* ARCNET, with BSD-style header */
#define DLT_SLIP    8    /* Serial Line IP */
#define DLT_PPP        9    /* Point-to-point Protocol */
#define DLT_FDDI    10    /* FDDI */

/*
 * These are types that are different on some platforms, and that
 * have been defined by <net/bpf.h> for ages.  We use #ifdefs to
 * detect the BSDs that define them differently from the traditional
 * libpcap <net/bpf.h>
 *
 * XXX - DLT_ATM_RFC1483 is 13 in BSD/OS, and DLT_RAW is 14 in BSD/OS,
 * but I don't know what the right #define is for BSD/OS.
 */
#define DLT_ATM_RFC1483    11    /* LLC-encapsulated ATM */

#ifdef __OpenBSD__
#define DLT_RAW        14    /* raw IP */
#else
#define DLT_RAW        12    /* raw IP */
#endif
/*
 * 给定目前只有 BSD/OS 生成 BSD/OS SLIP 或 PPP 的操作系统，可以说每个人都应该选择其数值
 * 为 DLT_SLIP_BSDOS 和 DLT_PPP_BSDOS，它们分别是 15 和 16，但他们没有这样做。所以就这样吧。
 */
#if defined(__NetBSD__) || defined(__FreeBSD__)
#ifndef DLT_SLIP_BSDOS
#define DLT_SLIP_BSDOS    13    /* BSD/OS Serial Line IP */
#define DLT_PPP_BSDOS    14    /* BSD/OS Point-to-point Protocol */
#endif
#else
#define DLT_SLIP_BSDOS    15    /* BSD/OS Serial Line IP */
#define DLT_PPP_BSDOS    16    /* BSD/OS Point-to-point Protocol */
#endif
/*
 * NetBSD使用15来表示HIPPI。
 *
 * 从快速查看sys/net/if_hippi.h和sys/net/if_hippisubr.c
 * 在NetBSD的旧版本中，头部似乎是：
 *
 *    一个1字节的ULP字段（ULP-id）？
 *
 *    一个1字节的标志字段；
 *
 *    一个2字节的“偏移”字段；
 *
 *    一个4字节的“D2长度”字段（D2_Size）；
 *
 *    一个4字节的“目标交换机”字段（或者一个1字节的字段
 *    包含转发类、双宽和消息类型子字段，后跟3字节的目标交换机地址
 *    字段？，HIPPI-LE 3.4风格？）；
 *
 *    一个4字节的“源交换机”字段（或者一个1字节的字段包含目标地址类型和源地址类型字段，
 *    后跟3字节的源交换机地址字段，HIPPI-LE 3.4风格？）；
 *
 *    一个2字节的保留字段；
 *
 *    一个6字节的目标地址字段；
 *
 *    一个2字节的“本地管理”字段；
 *
 *    一个6字节的源地址字段；
 *
 * 紧随其后的是802.2 LLC头部。
 *
 * 这看起来有点像从HIPPI-FP 4.4派生的东西
 * Header_Area，后跟一个包含带有HIPPI-LE 3.4（ANSI X3.218-1993）标头的D1数据集的HIPPI-FP 4.4 D1_Area，
 * 紧随其后的是包含802.2 LLC头部和有效载荷的HIPPI-FP 4.4 D2_Area（没有偏移）？或者“偏移”字段包含D2_Offset，
 * 在有效载荷之前有这么多字节的偏移？
 *
 * 请参阅http://wotug.org/parallel/standards/hippi/以获取HIPPI规范的存档。
 *
 * RFC 2067施加了一些额外的限制。它说偏移始终为零
 *
 * HIPPI早已消失，而在NetBSD的旧版本中找到的源文件似乎不在主要的CVS分支中，因此我们可能永远不会看到具有此链路层类型的捕获。
 */
#if defined(__NetBSD__)
#define DLT_HIPPI    15    /* HIPPI */
#endif
/*
 * NetBSD 使用 16 代表 DLT_HDLC；参见下文。
 * BSD/OS 使用它代表 PPP；参见上文。
 * 据我所知，没有其他操作系统将其用于其他用途；不要将其用于其他任何用途。
 */

/*
 * OpenBSD 曾经使用 17 代表 DLT_PFLOG；现在不再使用。
 *
 * 在 SuSE 6.3 中，它代表 DLT_LANE8023，因此我们将 LINKTYPE_PFLOG 定义为 117，
 * 以便 pflog 捕获使用一个不会与任何其他值冲突的链路层头类型值。
 * 在除 OpenBSD 外的所有平台上，我们将 DLT_PFLOG 定义为 117，并在 LINKTYPE_PFLOG 和 DLT_PFLOG 之间进行映射。
 *
 * 最终 OpenBSD 也切换到使用 117 代表 DLT_PFLOG。
 *
 * 不要将 17 用于其他任何用途。
 */

/*
 * 18 用于 OpenBSD、NetBSD、DragonFly BSD 和 macOS 中的 DLT_PFSYNC；不要将其用于其他任何用用途。
 * （FreeBSD 使用 121，与 DLT_HHDLC 冲突，尽管它不使用 18 代表任何东西，似乎也从未使用过。）
 *
 * 我们在这些平台上将其定义为 18；不幸的是，在 Suse 6.3 中用于 DLT_CIP，因此我们不将其一般定义为 DLT_PFSYNC。
 * 由于其数据包格式与 DLT_PFLOG 一样，不仅依赖于操作系统，还依赖于操作系统版本，因此我们不支持在除了具有相关头文件的操作系统外的平台上打印它，因此在其他平台上它并不那么有用。
 */
#if defined(__OpenBSD__) || defined(__NetBSD__) || defined(__DragonFly__) || defined(__APPLE__)
#define DLT_PFSYNC    18
#endif

#define DLT_ATM_CLIP    19    /* Linux Classical IP over ATM */

/*
 * 显然 Redback 在其 SmartEdge 400/800 中使用此值。希望没有其他人也决定使用它。
 */
#define DLT_REDBACK_SMARTEDGE    32

/*
 * 这些值由 NetBSD 定义；其他平台应该避免将它们用于其他目的，以便在所有平台上可以将具有 50 或 51 链路类型的 NetBSD 保存文件读取为此类型。
 */
#define DLT_PPP_SERIAL    50    /* PPP over serial with HDLC encapsulation */
# 定义 PPP over Ethernet 的链路层类型值为 51
#define DLT_PPP_ETHER    51    /* PPP over Ethernet */

# 定义 Symantec Enterprise Firewall 使用的链路层类型值为 99
/*
 * The Axent Raptor firewall - now the Symantec Enterprise Firewall - uses
 * a link-layer type of 99 for the tcpdump it supplies.  The link-layer
 * header has 6 bytes of unknown data, something that appears to be an
 * Ethernet type, and 36 bytes that appear to be 0 in at least one capture
 * I've seen.
 */
#define DLT_SYMANTEC_FIREWALL    99

# 值在 100 和 103 之间的链路层类型值用于捕获文件头中的 LINKTYPE_ 值，对应于不同平台之间的 DLT_ 类型；不要将这些值用于新的 DLT_ 类型
/*
 * Values between 100 and 103 are used in capture file headers as
 * link-layer header type LINKTYPE_ values corresponding to DLT_ types
 * that differ between platforms; don't use those values for new DLT_
 * new types.
 */

# 以 104 开头的值用于新分配的链路层类型值；对于这些链路层类型，由 pcap_datalink() 返回的 DLT_ 值和传递给 pcap_open_dead() 的 LINKTYPE_ 值是相同的
/*
 * Values starting with 104 are used for newly-assigned link-layer
 * header type values; for those link-layer header types, the DLT_
 * value returned by pcap_datalink() and passed to pcap_open_dead(),
 * and the LINKTYPE_ value that appears in capture files, are the
 * same.
 *
 * DLT_MATCHING_MIN is the lowest such value; DLT_MATCHING_MAX is
 * the highest such value.
 */
#define DLT_MATCHING_MIN    104

# 这个值是由 libpcap 0.5 定义的；具有不同值的平台应该在这里定义它为该值 - 保存文件中的 104 链路类型将被映射到 DLT_C_HDLC，无论该值是多少，因此程序将正确处理具有该链路类型的文件
/*
 * This value was defined by libpcap 0.5; platforms that have defined
 * it with a different value should define it here with that value -
 * a link type of 104 in a save file will be mapped to DLT_C_HDLC,
 * whatever value that happens to be, so programs will correctly
 * handle files with that link type regardless of the value of
 * DLT_C_HDLC.
 *
 * The name DLT_C_HDLC was used by BSD/OS; we use that name for source
 * compatibility with programs written for BSD/OS.
 *
 * libpcap 0.5 defined it as DLT_CHDLC; we define DLT_CHDLC as well,
 * for source compatibility with programs written for libpcap 0.5.
 */
#define DLT_C_HDLC    104    /* Cisco HDLC */
#define DLT_CHDLC    DLT_C_HDLC

# 定义 IEEE 802.11 无线的链路层类型值为 105
#define DLT_IEEE802_11    105    /* IEEE 802.11 wireless */
/*
 * 106 is reserved for Linux Classical IP over ATM; it's like DLT_RAW,
 * except when it isn't.  (I.e., sometimes it's just raw IP, and
 * sometimes it isn't.)  We currently handle it as DLT_LINUX_SLL,
 * so that we don't have to worry about the link-layer header.)
 */
#define DLT_FRELAY    107

/*
 * Frame Relay; BSD/OS has a DLT_FR with a value of 11, but that collides
 * with other values.
 * DLT_FR and DLT_FRELAY packets start with the Q.922 Frame Relay header
 * (DLCI, etc.).
 */
#define DLT_FRELAY    107

/*
 * OpenBSD DLT_LOOP, for loopback devices; it's like DLT_NULL, except
 * that the AF_ type in the link-layer header is in network byte order.
 *
 * DLT_LOOP is 12 in OpenBSD, but that's DLT_RAW in other OSes, so
 * we don't use 12 for it in OSes other than OpenBSD; instead, we
 * use the same value as LINKTYPE_LOOP.
 */
#ifdef __OpenBSD__
#define DLT_LOOP    12
#else
#define DLT_LOOP    108
#endif

/*
 * Encapsulated packets for IPsec; DLT_ENC is 13 in OpenBSD, but that's
 * DLT_SLIP_BSDOS in NetBSD, so we don't use 13 for it in OSes other
 * than OpenBSD; instead, we use the same value as LINKTYPE_ENC.
 */
#ifdef __OpenBSD__
#define DLT_ENC        13
#else
#define DLT_ENC        109
#endif

/*
 * Values 110 and 111 are reserved for use in capture file headers
 * as link-layer types corresponding to DLT_ types that might differ
 * between platforms; don't use those values for new DLT_ types
 * other than the corresponding DLT_ types.
 */

/*
 * NetBSD uses 16 for (Cisco) "HDLC framing".  For other platforms,
 * we define it to have the same value as LINKTYPE_NETBSD_HDLC.
 */
#if defined(__NetBSD__)
#define DLT_HDLC    16    /* Cisco HDLC */
#else
#define DLT_HDLC    112
#endif

/*
 * Linux cooked sockets.
 */
#define DLT_LINUX_SLL    113

/*
 * Apple LocalTalk hardware.
 */
#define DLT_LTALK    114

/*
 * Acorn Econet.
 */
#define DLT_ECONET    115

/*
 * Reserved for use with OpenBSD ipfilter.
 */
#define DLT_IPFILTER    116

/*
 * OpenBSD DLT_PFLOG.
 */
# 定义数据链路类型为 PFLOG 的常量
#define DLT_PFLOG    117

/*
 * 用于 Cisco 内部使用的注册数据链路类型
 */
#define DLT_CISCO_IOS    118

/*
 * 用于使用 Prism II 芯片的 802.11 卡，包括 Prism 监控模式信息和 802.11 头部的链路层头部
 */
#define DLT_PRISM_HEADER    119

/*
 * 保留给 Aironet 802.11 卡使用，带有 Aironet 链路层头部（参见 Doug Ambrisko 的 FreeBSD 补丁）
 */
#define DLT_AIRONET_HEADER    120

#ifdef __FreeBSD__
#define DLT_PFSYNC        121
#else
#define DLT_HHDLC        121
#endif

/*
 * 用于 RFC 2625 中的 IP-over-Fibre Channel
 *
 * 不适用于原始光纤通道，其中链路层头部以光纤通道帧头部开头；而是适用于 IP-over-FC，其中链路层头部以 RFC 2625 Network_Header 字段开头
 */
#define DLT_IP_OVER_FC        122

/*
 * 用于 Solaris 上的 Full Frontal ATM，带有伪头部，后跟 AALn PDU
 *
 * 可能还有其他操作系统上的其他形式的 Full Frontal ATM，带有不同的伪头部
 *
 * 如果 ATM 软件返回带有 VPI/VCI 信息的伪头部（最好还有数据包类型信息，例如信令、ILMI、LANE、LLC 多路复用流量等），则不应使用 DLT_ATM_RFC1483，而应获取新的 DLT_ 值，以便 tcpdump 等软件不必推断伪头部的存在或不存在以及伪头部的形式
 */
#define DLT_SUNATM        123    /* Solaris+SunATM */

/*
 * 根据 Kent Dahlgren <kent@praesum.com> 的请求保留为私人使用
 */
#define DLT_RIO                 124     /* RapidIO */
#define DLT_PCI_EXP             125     /* PCI Express */
#define DLT_AURORA              126     /* Xilinx Aurora link layer */

/*
 * 802.11 头部加上一些链路层信息的头部，包括无线电信息，由一些最近的 BSD 驱动程序以及 Linux 的 madwifi Atheros 驱动程序使用
 */
# 定义 IEEE 802.11 无线电头部的数据链路类型
#define DLT_IEEE802_11_RADIO    127    /* 802.11 plus radiotap radio header */

/*
 * 保留给 TZSP 封装，根据 Chris Waters <chris.waters@networkchemistry.com> 的请求
 * TZSP 是用于任何其他链路类型的通用封装，
 * 其中包括一种方法来在数据包中包含元信息，
 * 例如 802.11 数据包的信号强度和信道
 */
#define DLT_TZSP                128     /* Tazmen Sniffer Protocol */

/*
 * BSD 的 ARCNET 头部在数据包开头包含源主机、目标主机和类型；
 * 这是通过 BPF 传递给用户空间的。
 *
 * 然而，Linux 的 ARCNET 头部在主机 ID 和类型之间有一个 2 字节的偏移字段；
 * 这是通过 PF_PACKET sockets 传递给用户空间的。
 *
 * 因此，我们必须为它们分别定义不同的 DLT_ 值。
 */
#define DLT_ARCNET_LINUX    129    /* ARCNET */

/*
 * Juniper 私有的数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求。
 * DLT_ 用于传递机箱内部的元信息，例如 QOS 配置文件等。
 */
#define DLT_JUNIPER_MLPPP       130
#define DLT_JUNIPER_MLFR        131
#define DLT_JUNIPER_ES          132
#define DLT_JUNIPER_GGSN        133
#define DLT_JUNIPER_MFR         134
#define DLT_JUNIPER_ATM2        135
#define DLT_JUNIPER_SERVICES    136
#define DLT_JUNIPER_ATM1        137

/*
 * Apple IP-over-IEEE 1394，根据 Dieter Siegmund <dieter@apple.com> 的请求。
 * 提供的头部是类似以太网的头部：
 *
 *    #define FIREWIRE_EUI64_LEN    8
 *    struct firewire_header {
 *        u_char  firewire_dhost[FIREWIRE_EUI64_LEN];
 *        u_char  firewire_shost[FIREWIRE_EUI64_LEN];
 *        u_short firewire_type;
 *    };
 *
 * 其中 "firewire_type" 是以太网类型值，而不是例如原始 GASP 帧的传递。
 */
#define DLT_APPLE_IP_OVER_IEEE1394    138
/*
 * Various SS7 encapsulations, as per a request from Jeff Morriss
 * <jeff.morriss[AT]ulticom.com> and subsequent discussions.
 */
# 定义不同的 SS7 封装，根据 Jeff Morriss 的请求和后续讨论
#define DLT_MTP2_WITH_PHDR    139    /* pseudo-header with various info, followed by MTP2 */
#define DLT_MTP2        140    /* MTP2, without pseudo-header */
#define DLT_MTP3        141    /* MTP3, without pseudo-header or MTP2 */
#define DLT_SCCP        142    /* SCCP, without pseudo-header or MTP2 or MTP3 */

/*
 * DOCSIS MAC frames.
 */
# 定义 DOCSIS MAC 帧
#define DLT_DOCSIS        143

/*
 * Linux-IrDA packets. Protocol defined at https://www.irda.org.
 * Those packets include IrLAP headers and above (IrLMP...), but
 * don't include Phy framing (SOF/EOF/CRC & byte stuffing), because Phy
 * framing can be handled by the hardware and depend on the bitrate.
 * This is exactly the format you would get capturing on a Linux-IrDA
 * interface (irdaX), but not on a raw serial port.
 * Note the capture is done in "Linux-cooked" mode, so each packet include
 * a fake packet header (struct sll_header). This is because IrDA packet
 * decoding is dependent on the direction of the packet (incoming or
 * outgoing).
 * When/if other platform implement IrDA capture, we may revisit the
 * issue and define a real DLT_IRDA...
 * Jean II
 */
# 定义 Linux-IrDA 数据包
#define DLT_LINUX_IRDA        144

/*
 * Reserved for IBM SP switch and IBM Next Federation switch.
 */
# 保留给 IBM SP switch 和 IBM Next Federation switch
#define DLT_IBM_SP        145
#define DLT_IBM_SN        146
/*
 * 保留给私人使用。如果您有一些链路层头部类型想要在您的组织内使用，并且使用该链路层头部类型的捕获文件不会被发送到您的组织之外，那么您可以使用这些数值。
 *
 * 任何 libpcap 发行版都不会为任何目的使用这些值，任何 tcpdump 发行版也不会使用它们。
 *
 * *不要*在您期望任何不使用您私人版本的捕获文件读取工具的人能够读取的捕获文件中使用这些值；特别是*不要*在产品中使用它们，否则您可能会发现人们无法使用 tcpdump、snort、Ethereal 等工具来读取来自您的防火墙/入侵检测/流量监控等设备的捕获文件，或者使用该 DLT_ 值的任何产品，您还可能会发现这些应用程序的开发人员不接受让它们读取这些文件的补丁。
 *
 * 同样，如果有人可能会使用它们发送捕获文件，而使用它们的工具来读取*您*的私人类型的捕获文件，那么也不要使用它们。
 *
 * 相反，根据上面的注释，向 "tcpdump-workers@lists.tcpdump.org" 请求一个新的 DLT_ 值，并使用您得到的类型。
 */
#define DLT_USER0        147
#define DLT_USER1        148
#define DLT_USER2        149
#define DLT_USER3        150
#define DLT_USER4        151
#define DLT_USER5        152
#define DLT_USER6        153
#define DLT_USER7        154
#define DLT_USER8        155
#define DLT_USER9        156
#define DLT_USER10        157
#define DLT_USER11        158
#define DLT_USER12        159
#define DLT_USER13        160
#define DLT_USER14        161
#define DLT_USER15        162
/*
 * 用于将来与 802.11 捕获一起使用 - 由 AbsoluteValue Systems 定义，用于存储一定数量的链路层信息，包括无线电信息：
 *    http://www.shaftnet.org/~pizza/software/capturefrm.txt
 * 但现在或将来可能被一些非 AVS 驱动程序使用。
 */
#define DLT_IEEE802_11_RADIO_AVS 163    /* 802.11 加 AVS 无线电头 */

/*
 * Juniper 私有数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求。DLT_s 用于传递机箱内部的元信息，如 QOS 配置文件等。
 */
#define DLT_JUNIPER_MONITOR     164

/*
 * BACnet MS/TP 帧。
 */
#define DLT_BACNET_MS_TP    165

/*
 * 另一种 PPP 变体，根据 Karsten Keil <kkeil@suse.de> 的请求。
 * 在某些操作系统中，允许内核套接字过滤器区分传入和传出的数据包，用于供 pppd 提供传出数据包，以便它可以进行按需拨号和缺乏需求时挂断；传入数据包被过滤掉，以免导致 pppd 保持连接（不希望随机输入数据包，如端口扫描、来自旧丢失连接的数据包等，导致连接保持）。
 * PPP 头的第一个字节（0xff03）被修改以适应方向 - 0x00 = IN，0x01 = OUT。
 */
#define DLT_PPP_PPPD        166

/*
 * 用于向后兼容旧版本某些 PPP 软件的名称；新软件应该使用 DLT_PPP_PPPD。
 */
#define DLT_PPP_WITH_DIRECTION    DLT_PPP_PPPD
#define DLT_LINUX_PPP_WITHDIRECTION    DLT_PPP_PPPD

/*
 * Juniper 私有数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求。DLT_s 用于传递机箱内部的元信息，如 QOS 配置文件、cookies 等。
 */
#define DLT_JUNIPER_PPPOE       167
#define DLT_JUNIPER_PPPOE_ATM   168

#define DLT_GPRS_LLC        169    /* GPRS LLC */
# 定义数据链路类型常量，表示不同的数据链路类型
#define DLT_GPF_T        170    /* GPF-T (ITU-T G.7041/Y.1303) */
#define DLT_GPF_F        171    /* GPF-F (ITU-T G.7041/Y.1303) */

/*
 * 由Oolan Zimmer <oz@gcom.com>请求，用于Gcom的T1/E1线路监测设备。
 */
#define DLT_GCOM_T1E1        172
#define DLT_GCOM_SERIAL        173

/*
 * Juniper私有数据链路类型，由Hannes Gredler <hannes@juniper.net>请求。
 * DLT_用于内部与物理接口卡（PIC）的通信。
 */
#define DLT_JUNIPER_PIC_PEER    174

/*
 * 由Endace Measurement Systems的Gregor Maier <gregor@endace.com>请求的数据链路类型。
 * 它在链路层头部之前添加了ERF头部。
 */
#define DLT_ERF_ETH        175    /* 以太网 */
#define DLT_ERF_POS        176    /* Packet-over-SONET */

/*
 * 由Daniele Orlandi <daniele@orlandi.com>请求，用于vISDN的原始LAPD。
 * 其链路层头部在LAPD头部之前包含附加信息，因此不一定是通用的LAPD头部。
 */
#define DLT_LINUX_LAPD        177

/*
 * Juniper私有数据链路类型，由Hannes Gredler <hannes@juniper.net>请求。
 * DLT_用于在标准以太网、PPP、Frelay和C-HDLC帧之前添加元信息，如接口索引、接口名称。
 */
#define DLT_JUNIPER_ETHER       178
#define DLT_JUNIPER_PPP         179
#define DLT_JUNIPER_FRELAY      180
#define DLT_JUNIPER_CHDLC       181

/*
 * 多链路帧中继（FRF.16）
 */
#define DLT_MFR                 182

/*
 * Juniper私有数据链路类型，由Hannes Gredler <hannes@juniper.net>请求。
 * DLT_用于与语音适配器卡（PIC）进行内部通信。
 */
#define DLT_JUNIPER_VP          183
/*
 * Arinc 429 frames.
 * DLT_ requested by Gianluca Varenni <gianluca.varenni@cacetech.com>.
 * Every frame contains a 32bit A429 label.
 * More documentation on Arinc 429 can be found at
 * https://web.archive.org/web/20040616233302/https://www.condoreng.com/support/downloads/tutorials/ARINCTutorial.pdf
 */
#define DLT_A429                184

/*
 * Arinc 653 Interpartition Communication messages.
 * DLT_ requested by Gianluca Varenni <gianluca.varenni@cacetech.com>.
 * Please refer to the A653-1 standard for more information.
 */
#define DLT_A653_ICM            185

/*
 * This used to be "USB packets, beginning with a USB setup header;
 * requested by Paolo Abeni <paolo.abeni@email.it>."
 *
 * However, that header didn't work all that well - it left out some
 * useful information - and was abandoned in favor of the DLT_USB_LINUX
 * header.
 *
 * This is now used by FreeBSD for its BPF taps for USB; that has its
 * own headers.  So it is written, so it is done.
 *
 * For source-code compatibility, we also define DLT_USB to have this
 * value.  We do it numerically so that, if code that includes this
 * file (directly or indirectly) also includes an OS header that also
 * defines DLT_USB as 186, we don't get a redefinition warning.
 * (NetBSD 7 does that.)
 */
#define DLT_USB_FREEBSD        186
#define DLT_USB            186

/*
 * Bluetooth HCI UART transport layer (part H:4); requested by
 * Paolo Abeni.
 */
#define DLT_BLUETOOTH_HCI_H4    187

/*
 * IEEE 802.16 MAC Common Part Sublayer; requested by Maria Cruz
 * <cruz_petagay@bah.com>.
 */
#define DLT_IEEE802_16_MAC_CPS    188

/*
 * USB packets, beginning with a Linux USB header; requested by
 * Paolo Abeni <paolo.abeni@email.it>.
 */
#define DLT_USB_LINUX        189
/*
 * 定义 Controller Area Network (CAN) v. 2.0B 数据包的数据链路类型
 * 由 Gianluca Varenni <gianluca.varenni@cacetech.com> 请求添加
 * 用于从 CAN Vector 板接收 CAN 数据包
 * 更多关于 CAN v2.0B 数据帧的文档可以在以下链接找到
 * http://www.can-cia.org/downloads/?269
 */
#define DLT_CAN20B              190

/*
 * IEEE 802.15.4 数据链路类型，地址字段被填充，与 Linux 驱动程序一致
 * 由 Juergen Schimmer 请求添加
 */
#define DLT_IEEE802_15_4_LINUX    191

/*
 * 封装了每个数据包的信息的数据链路类型
 * 由 Gianluca Varenni <gianluca.varenni@cacetech.com> 请求添加
 */
#define DLT_PPI            192

/*
 * 802.16 MAC 公共部分子层的头部加上 radiotap 无线电头部的数据链路类型
 * 由 Charles Clancy 请求添加
 */
#define DLT_IEEE802_16_MAC_CPS_RADIO    193

/*
 * Juniper 私有数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求添加
 * 用于与集成服务模块 (ISM) 进行内部通信
 */
#define DLT_JUNIPER_ISM         194

/*
 * IEEE 802.15.4 数据链路类型，与规范完全一致（没有填充等）
 * 由 Mikko Saarnivala <mikko.saarnivala@sensinode.com> 请求添加
 * 对于这个类型，我们期望帧的末尾有 FCS；
 * 如果帧没有 FCS，则应该使用 DLT_IEEE802_15_4_NOFCS
 *
 * 我们保留 DLT_IEEE802_15_4 的名称作为向后兼容的别名，但是，这应该 *仅* 用于包含 FCS 的 802.15.4 帧
 */
#define DLT_IEEE802_15_4_WITHFCS    195
#define DLT_IEEE802_15_4        DLT_IEEE802_15_4_WITHFCS

/*
 * 用于 SITA（https://www.sita.aero/）的各种带有伪头部的数据链路类型
 * 由 Fulko Hew (fulko.hew@gmail.com) 请求添加
 */
#define DLT_SITA        196

/*
 * 用于 Endace DAG 卡的各种带有伪头部的数据链路类型
 * 封装了 Endace ERF 记录
 * 由 Stephen Donnelly <stephen@endace.com> 请求添加
 */
#define DLT_ERF            197
/*
 * 定义了一个特殊的以太网数据包头部，用于捕获从u10 Networks板卡上的数据包。这是根据Phil Mulholland <phil@u10networks.com>的要求添加的。
 */
#define DLT_RAIF1        198

/*
 * 用于 IPMI 的 IPMB 数据包，以 2 字节的头部开始，然后是 I2C 从设备地址，接着是 netFn 和 LUN 等等。
 * 这是根据Chanthy Toeung <chanthy.toeung@ca.kontron.com>的要求添加的。
 *
 * XXX - 这个曾经被称为 DLT_IPMB，当时我们从邮件线程中得到的印象是数据包没有额外的 2 字节头部。我们已经重命名了它；如果有人使用 DLT_IPMB 并假设没有 2 字节的头部，这将导致编译失败，届时我们将不得不弄清楚如何处理使用相同 DLT_/LINKTYPE_ 值的两种头部类型。如果没有发生这种情况，我们将假设没有人使用它，并且重新定义是安全的。
 */
#define DLT_IPMB_KONTRON    199

/*
 * Juniper 私有的数据链路类型，根据Hannes Gredler <hannes@juniper.net>的要求添加。
 * DLT_ 用于在安全隧道接口上捕获数据。
 */
#define DLT_JUNIPER_ST          200

/*
 * 带有伪头部的蓝牙 HCI UART 传输层（第 H:4 部分），包括方向信息；由Paolo Abeni要求添加。
 */
#define DLT_BLUETOOTH_HCI_H4_WITH_PHDR    201

/*
 * 带有 1 字节 KISS 头部的 AX.25 数据包；参见
 *
 *    http://www.ax25.net/kiss.htm
 *
 * 根据Richard Stearn <richard@rns-stearn.demon.co.uk>的要求添加。
 */
#define DLT_AX25_KISS        202

/*
 * 来自 ISDN 通道的 LAPD 数据包，以地址字段开头，没有伪头部。
 * 根据Varuna De Silva <varunax@gmail.com>的要求添加。
 */
#define DLT_LAPD        203
/*
 * 定义 PPP，带有一个字节的方向伪头部前缀 - 零表示“由此主机接收”，非零（任何非零值）表示“由此主机发送” - 根据 Will Barker <w.barker@zen.co.uk> 的说法。
 * 不要将其与 DLT_PPP_WITH_DIRECTION 混淆，后者是现在称为 DLT_PPP_PPPD 的旧名称。
 */
#define DLT_PPP_WITH_DIR    204

/*
 * Cisco HDLC，带有一个字节的方向伪头部前缀 - 零表示“由此主机接收”，非零（任何非零值）表示“由此主机发送” - 根据 Will Barker <w.barker@zen.co.uk> 的说法。
 */
#define DLT_C_HDLC_WITH_DIR    205

/*
 * Frame Relay，带有一个字节的方向伪头部前缀 - 零表示“由此主机接收”（DCE -> DTE），非零（任何非零值）表示“由此主机发送”（DTE -> DCE）- 根据 Will Barker <w.barker@zen.co.uk> 的说法。
 */
#define DLT_FRELAY_WITH_DIR    206

/*
 * LAPB，带有一个字节的方向伪头部前缀 - 零表示“由此主机接收”（DCE -> DTE），非零（任何非零值）表示“由此主机发送”（DTE -> DCE）- 根据 Will Barker <w.barker@zen.co.uk> 的说法。
 */
#define DLT_LAPB_WITH_DIR    207

/*
 * 208 保留给尚未指定的专有链路层类型，根据 Will Barker 的要求。
 */

/*
 * 具有 Linux 特定伪头部的 IPMB；由 Alexey Neyman <avn@pigeonpoint.com> 请求。
 */
#define DLT_IPMB_LINUX        209

/*
 * FlexRay 汽车总线 - http://www.flexray.com/ - 由 Hannes Kaelber <hannes.kaelber@x2e.de> 请求。
 */
#define DLT_FLEXRAY        210

/*
 * 用于多媒体传输的 Media Oriented Systems Transport (MOST) 总线 - https://www.mostcooperation.com/ - 由 Hannes Kaelber <hannes.kaelber@x2e.de> 请求。
 */
#define DLT_MOST        211

/*
 * 用于车辆网络的 Local Interconnect Network (LIN) 总线 - http://www.lin-subbus.org/ - 由 Hannes Kaelber <hannes.kaelber@x2e.de> 请求。
 */
#define DLT_LIN            212
/*
 * 用于串行线捕获的 X2E 私有数据链路类型，由 Hannes Kaelber <hannes.kaelber@x2e.de> 请求添加。
 */
#define DLT_X2E_SERIAL        213

/*
 * 用于 Xoraya 数据记录器系列的 X2E 私有数据链路类型，由 Hannes Kaelber <hannes.kaelber@x2e.de> 请求添加。
 */
#define DLT_X2E_XORAYA        214

/*
 * IEEE 802.15.4，与规范完全一致（没有填充，没有其他内容），但对于非 ASK PHY 的 PHY 级数据（4 个八位组的前导码为 0，一个八位组的 SFD，一个八位组的帧长度+保留位，然后是以帧控制字段开头的 MAC 层数据）。
 *
 * 由 Max Filippov <jcmvbkbc@gmail.com> 请求添加。
 */
#define DLT_IEEE802_15_4_NONASK_PHY    215

/*
 * 由 David Gibson <david@gibson.dropbear.id.au> 请求添加，用于从 Linux 内核 /dev/input/eventN 设备捕获。这用于将 Linux 内核中的按键和鼠标移动传输到显示系统，如 Xorg。
 */
#define DLT_LINUX_EVDEV        216

/*
 * GSM Um 和 Abis 接口，以 "gsmtap" 头部开头。
 *
 * 由 Harald Welte <laforge@gnumonks.org> 请求添加。
 */
#define DLT_GSMTAP_UM        217
#define DLT_GSMTAP_ABIS        218

/*
 * MPLS，带有 MPLS 标签作为链路层头部。
 * 由 Michele Marchetto <michele@openbsd.org> 代表 OpenBSD 请求添加。
 */
#define DLT_MPLS        219

/*
 * USB 数据包，以 Linux USB 头部开头，USB 头部填充到 64 个字节；用于内存映射访问。
 */
#define DLT_USB_LINUX_MMAPPED    220

/*
 * DECT 数据包，带有伪头部；由 Matthias Wenzel <tcpdump@mazzoo.de> 请求添加。
 */
#define DLT_DECT        221

/*
 * 来自："Lidwa, Eric (GSFC-582.0)[SGT INC]" <eric.lidwa-1@nasa.gov>
 * 日期：2009 年 5 月 11 日 11:18:30 -0500
 *
 * DLT_AOS。我们需要它用于 AOS 空间数据链路协议。
 *   我已经为其编写了解析器，但在提交补丁之前需要法律部门的批准。
 *
 */
#define DLT_AOS                 222
// 定义 AOS 数据链路类型的常量值为 222

/*
 * Wireless HART (Highway Addressable Remote Transducer)
 * From the HART Communication Foundation
 * IES/PAS 62591
 *
 * Requested by Sam Roberts <vieuxtech@gmail.com>.
 */
#define DLT_WIHART        223
// 定义 WIHART 数据链路类型的常量值为 223，该类型来自 HART 通信基金会，由 Sam Roberts 请求添加

/*
 * Fibre Channel FC-2 frames, beginning with a Frame_Header.
 * Requested by Kahou Lei <kahou82@gmail.com>.
 */
#define DLT_FC_2        224
// 定义 FC-2 数据链路类型的常量值为 224，该类型以 Frame_Header 开头，由 Kahou Lei 请求添加

/*
 * Fibre Channel FC-2 frames, beginning with an encoding of the
 * SOF, and ending with an encoding of the EOF.
 *
 * The encodings represent the frame delimiters as 4-byte sequences
 * representing the corresponding ordered sets, with K28.5
 * represented as 0xBC, and the D symbols as the corresponding
 * byte values; for example, SOFi2, which is K28.5 - D21.5 - D1.2 - D21.2,
 * is represented as 0xBC 0xB5 0x55 0x55.
 *
 * Requested by Kahou Lei <kahou82@gmail.com>.
 */
#define DLT_FC_2_WITH_FRAME_DELIMS    225
// 定义带有帧分隔符的 FC-2 数据链路类型的常量值为 225，该类型以 SOF 编码开始，以 EOF 编码结束，由 Kahou Lei 请求添加
/*
 * Solaris ipnet pseudo-header; requested by Darren Reed <Darren.Reed@Sun.COM>.
 *
 * The pseudo-header starts with a one-byte version number; for version 2,
 * the pseudo-header is:
 *
 * struct dl_ipnetinfo {
 *     uint8_t   dli_version;  // 版本号，当前版本为2
 *     uint8_t   dli_family;   // Solaris地址族值，IPv4为2，IPv6为26
 *     uint16_t  dli_htype;    // “hook类型” - 0表示传入数据包，1表示传出数据包，2表示来自同一台机器上另一个区域的数据包
 *     uint32_t  dli_pktlen;   // 伪头部后面数据包的长度
 *     uint32_t  dli_ifindex;  // 数据包到达的接口索引
 *     uint32_t  dli_grifindex;  // 组接口索引号（用于IPMP接口）
 *     uint32_t  dli_zsrc;     // 数据包源的区域标识符
 *     uint32_t  dli_zdst;     // 数据包目的地的区域标识符
 * };
 *
 * dli_version为伪头部的当前版本为2
 *
 * dli_family是Solaris地址族值，所以IPv4为2，IPv6为26
 *
 * dli_htype是“hook类型” - 0表示传入数据包，1表示传出数据包，2表示来自同一台机器上另一个区域的数据包
 *
 * dli_pktlen是伪头部后面数据包的长度（所以捕获长度减去dli_pktlen就是伪头部的长度，假设整个伪头部都被捕获）
 *
 * dli_ifindex是数据包到达的接口索引
 *
 * dli_grifindex是组接口索引号（用于IPMP接口）
 *
 * dli_zsrc是数据包源的区域标识符
 *
 * dli_zdst是数据包目的地的区域标识符
 *
 * 区域编号为0表示全局区域；区域编号为0xffffffff表示数据包来自网络上的另一台主机，而不是来自同一台机器上的另一个区域
 *
 * 伪头部后面跟着IPv4或IPv6数据报；dli_family指示其中的哪一个
 */
#define DLT_IPNET        226

/*
 * CAN（Controller Area Network）帧，带有Linux SocketCAN提供的伪头部，并且该头部中的多字节数值字段以大端字节顺序提供。
 *
 * 请参阅Linux源代码中的Documentation/networking/can.txt。
 *
 * 由Felix Obenhuber <felix@obenhuber.de>请求。
 */
#define DLT_CAN_SOCKETCAN    227
/*
 * 定义原始的 IPv4/IPv6 数据链路类型；与 DLT_RAW 不同之处在于 DLT_ 值指定了是 v4 还是 v6。由 Darren Reed <Darren.Reed@Sun.COM> 请求添加。
 */
#define DLT_IPV4        228
#define DLT_IPV6        229

/*
 * IEEE 802.15.4，与规范中完全一致（没有填充，没有其他内容），帧末尾没有 FCS；由 Jon Smirl <jonsmirl@gmail.com> 请求添加。
 */
#define DLT_IEEE802_15_4_NOFCS    230

/*
 * 原始的 D-Bus：
 *
 *    https://www.freedesktop.org/wiki/Software/dbus
 *
 * 消息：
 *
 *    https://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol-messages
 *
 * 以大小端标志开头，后跟消息类型等，但在消息序列之前没有身份验证握手：
 *
 *    https://dbus.freedesktop.org/doc/dbus-specification.html#auth-protocol
 *
 * 由 Martin Vidner <martin@vidner.net> 请求添加。
 */
#define DLT_DBUS        231

/*
 * Juniper 私有的数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求添加。
 */
#define DLT_JUNIPER_VS            232
#define DLT_JUNIPER_SRX_E2E        233
#define DLT_JUNIPER_FIBRECHANNEL    234

/*
 * DVB-CI（用于 PC 卡模块与 DVB 接收器之间通信的 DVB 通用接口）。参见
 *
 *    https://www.kaiser.cx/pcap-dvbci.html
 *
 * 规范。
 *
 * 由 Martin Kaiser <martin@kaiser.cx> 请求添加。
 */
#define DLT_DVB_CI        235

/*
 * 3GPP TS 27.010 多路复用协议的变种（类似于但不同于 27.010）。由 Hans-Christoph Schemmel <hans-christoph.schemmel@cinterion.com> 请求添加。
 */
#define DLT_MUX27010        236

/*
 * STANAG 5066 D_PDUs。由 M. Baris Demiray <barisdemiray@gmail.com> 请求添加。
 */
#define DLT_STANAG_5066_D_PDU    237

/*
 * Juniper 私有的数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求添加。
 */
#define DLT_JUNIPER_ATM_CEMIC    238
# 定义 NetFilter LOG 消息的数据链路层类型
# (netlink NFNL_SUBSYS_ULOG/NFULNL_MSG_PACKET 数据包的有效载荷)
# 由 Jakub Zawadzki <darkjames-ws@darkjames.pl> 请求
#define DLT_NFLOG        239

# 定义 Hilscher Gesellschaft fuer Systemautomation mbH 的链路层类型
# 用于带有 4 字节伪头部的以太网数据包，并始终包括 FCS 的有效载荷
# 由他们的 netANALYZER 硬件和软件提供
# 由 Holger P. Frommer <HPfrommer@hilscher.com> 请求
#define DLT_NETANALYZER        240

# 定义 Hilscher Gesellschaft fuer Systemautomation mbH 的链路层类型
# 用于带有 4 字节伪头部和 FCS 的以太网数据包
# 以太网头部之前有 7 字节前导码和 1 字节 SFD
# 由他们的 netANALYZER 硬件和软件提供
# 由 Holger P. Frommer <HPfrommer@hilscher.com> 请求
#define DLT_NETANALYZER_TRANSPARENT    241

# 定义 IP-over-InfiniBand 的数据链路层类型，由 RFC 4391 指定
# 由 Petr Sumbera <petr.sumbera@oracle.com> 请求
#define DLT_IPOIB        242

# 定义 MPEG-2 传输流的数据链路层类型 (ISO 13818-1/ITU-T H.222.0)
# 由 Guy Martin <gmsoft@tuxicoman.be> 请求
#define DLT_MPEG_2_TS        243

# 定义 ng4T GmbH 的 UMTS Iub/Iur-over-ATM 和 Iub/Iur-over-IP 格式的数据链路层类型
# 由他们的 ng40 协议测试仪使用
# 由 Jens Grimmer <jens.grimmer@ng4t.com> 请求
#define DLT_NG40        244

# 定义伪头部，包括适配器编号和标志，后跟 NFC (近场通信) 逻辑链路控制协议 (LLCP) PDU
# 如 NFC Forum Logical Link Control Protocol Technical Specification LLCP 1.1 所指定
# 由 Mike Wakerly <mikey@google.com> 请求
#define DLT_NFC_LLCP        245

# 246 用作 LINKTYPE_PFSYNC；不要用于其他目的
# DLT_PFSYNC 在不同平台上具有不同的值，并且所有这些值都与其他地方使用的某些值冲突
# 在尚未定义的平台上，将其定义为 246
#if !defined(__FreeBSD__) && !defined(__OpenBSD__) && !defined(__NetBSD__) && !defined(__DragonFly__) && !defined(__APPLE__)
#define DLT_PFSYNC        246
#endif

如果操作系统不是FreeBSD、OpenBSD、NetBSD、DragonFly或者苹果系统，则定义DLT_PFSYNC为246。


/*
 * Raw InfiniBand packets, starting with the Local Routing Header.
 *
 * Requested by Oren Kladnitsky <orenk@mellanox.com>.
 */
#define DLT_INFINIBAND        247

定义DLT_INFINIBAND为247，表示原始InfiniBand数据包，以本地路由头开始。


/*
 * SCTP, with no lower-level protocols (i.e., no IPv4 or IPv6).
 *
 * Requested by Michael Tuexen <Michael.Tuexen@lurchi.franken.de>.
 */
#define DLT_SCTP        248

定义DLT_SCTP为248，表示没有底层协议（即没有IPv4或IPv6）的SCTP数据包。


/*
 * USB packets, beginning with a USBPcap header.
 *
 * Requested by Tomasz Mon <desowin@gmail.com>
 */
#define DLT_USBPCAP        249

定义DLT_USBPCAP为249，表示以USBPcap头开始的USB数据包。


/*
 * Schweitzer Engineering Laboratories "RTAC" product serial-line
 * packets.
 *
 * Requested by Chris Bontje <chris_bontje@selinc.com>.
 */
#define DLT_RTAC_SERIAL        250

定义DLT_RTAC_SERIAL为250，表示Schweitzer Engineering Laboratories "RTAC"产品的串行线路数据包。


/*
 * Bluetooth Low Energy air interface link-layer packets.
 *
 * Requested by Mike Kershaw <dragorn@kismetwireless.net>.
 */
#define DLT_BLUETOOTH_LE_LL    251

定义DLT_BLUETOOTH_LE_LL为251，表示蓝牙低功耗空中接口链路层数据包。


/*
 * DLT type for upper-protocol layer PDU saves from Wireshark.
 *
 * the actual contents are determined by two TAGs, one or more of
 * which is stored with each packet:
 *
 *   EXP_PDU_TAG_DISSECTOR_NAME      the name of the Wireshark dissector
 *                     that can make sense of the data stored.
 *
 *   EXP_PDU_TAG_HEUR_DISSECTOR_NAME the name of the Wireshark heuristic
 *                     dissector that can make sense of the
 *                     data stored.
 */
#define DLT_WIRESHARK_UPPER_PDU    252

定义DLT_WIRESHARK_UPPER_PDU为252，表示来自Wireshark的上层协议层PDU保存的DLT类型。


/*
 * DLT type for the netlink protocol (nlmon devices).
 */
#define DLT_NETLINK        253

定义DLT_NETLINK为253，表示netlink协议的DLT类型（nlmon设备）。


/*
 * Bluetooth Linux Monitor headers for the BlueZ stack.
 */
#define DLT_BLUETOOTH_LINUX_MONITOR    254

定义DLT_BLUETOOTH_LINUX_MONITOR为254，表示BlueZ堆栈的蓝牙Linux监视器头。


/*
 * Bluetooth Basic Rate/Enhanced Data Rate baseband packets, as
 * captured by Ubertooth.
 */
#define DLT_BLUETOOTH_BREDR_BB    255

定义DLT_BLUETOOTH_BREDR_BB为255，表示Ubertooth捕获的蓝牙基本速率/增强数据速率基带数据包。


/*
 * Bluetooth Low Energy link layer packets, as captured by Ubertooth.
 */
#define DLT_BLUETOOTH_LE_LL_WITH_PHDR    256

定义DLT_BLUETOOTH_LE_LL_WITH_PHDR为256，表示Ubertooth捕获的蓝牙低功耗链路层数据包。
/*
 * PROFIBUS数据链路层。
 */
#define DLT_PROFIBUS_DL        257

#ifdef __APPLE__
#define DLT_PKTAP    DLT_USER2
#else
#define DLT_PKTAP    258
#endif

/*
 * 以802.3-2012第65条款第65.1.3.2节“传输”规定的前导码的最后6个八位字节为头部的以太网数据包。
 */
#define DLT_EPON    259

/*
 * IPMI跟踪数据包，如PICMG HPM.2规范中的表3-20“跟踪数据块格式”所规定。
 */
#define DLT_IPMI_HPM_2    260

/*
 * 根据Joshua Wright <jwright@hasborg.com>的说法，Zwave捕获的格式。
 */
#define DLT_ZWAVE_R1_R2  261
#define DLT_ZWAVE_R3     262

/*
 * 根据Steve Karg <skarg@users.sourceforge.net>的说法，Wattstopper数字照明管理室总线串行协议捕获的格式。
 */
#define DLT_WATTSTOPPER_DLM     263

/*
 * ISO 14443非接触式智能卡消息。
 */
#define DLT_ISO_14443    264

/*
 * 无线电数据系统（RDS）组。IEC 62106。
 * 根据Jonathan Brucker <jonathan.brucke@gmail.com>的说法。
 */
#define DLT_RDS        265

/*
 * USB数据包，以Darwin（macOS等）头部开始。
 */
#define DLT_USB_DARWIN    266

/*
 * OpenBSD DLT_OPENFLOW。
 */
#define DLT_OPENFLOW    267

/*
 * 包含SNA PDU的SDLC帧。
 */
#define DLT_SDLC    268

/*
 * 根据"Selvig, Bjorn" <b.selvig@ti.com>的说法，用于TI协议嗅探器。
 */
#define DLT_TI_LLN_SNIFFER    269

/*
 * 根据Erik de Jong <erikdejong at gmail.com>的说法，用于LoRaTap的格式。
 */
#define DLT_LORATAP             270

/*
 * 根据Stefanha at gmail.com的说法，用于VirtioVsock的格式。
 */
#define DLT_VSOCK               271

/*
 * Nordic Semiconductor蓝牙LE嗅探器。
 */
#define DLT_NORDIC_BLE        272
/*
 * Excentis DOCSIS 3.1 RF sniffer (XRA-31)
 *   per: bruno.verstuyft at excentis.com
 *        https://www.xra31.com/xra-header
 */
#define DLT_DOCSIS31_XRA31    273

/*
 * mPackets, as specified by IEEE 802.3br Figure 99-4, starting
 * with the preamble and always ending with a CRC field.
 */
#define DLT_ETHERNET_MPACKET    274

/*
 * DisplayPort AUX channel monitoring data as specified by VESA
 * DisplayPort(DP) Standard preceded by a pseudo-header.
 *    per dirk.eibach at gdsys.cc
 */
#define DLT_DISPLAYPORT_AUX    275

/*
 * Linux cooked sockets v2.
 */
#define DLT_LINUX_SLL2    276

/*
 * Sercos Monitor, per Manuel Jacob <manuel.jacob at steinbeis-stg.de>
 */
#define DLT_SERCOS_MONITOR 277

/*
 * OpenVizsla http://openvizsla.org is open source USB analyzer hardware.
 * It consists of FPGA with attached USB phy and FTDI chip for streaming
 * the data to the host PC.
 *
 * Current OpenVizsla data encapsulation format is described here:
 * https://github.com/matwey/libopenvizsla/wiki/OpenVizsla-protocol-description
 *
 */
#define DLT_OPENVIZSLA            278

/*
 * The Elektrobit High Speed Capture and Replay (EBHSCR) protocol is produced
 * by a PCIe Card for interfacing high speed automotive interfaces.
 *
 * The specification for this frame format can be found at:
 *   https://www.elektrobit.com/ebhscr
 *
 * for Guenter.Ebermann at elektrobit.com
 *
 */
#define DLT_EBHSCR            279

/*
 * The https://fd.io vpp graph dispatch tracer produces pcap trace files
 * in the format documented here:
 * https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/vnet.html#graph-dispatcher-pcap-tracing
 */
#define DLT_VPP_DISPATCH    280

/*
 * Broadcom Ethernet switches (ROBO switch) 4 bytes proprietary tagging format.
 */
#define DLT_DSA_TAG_BRCM    281
#define DLT_DSA_TAG_BRCM_PREPEND    282
/*
 * IEEE 802.15.4 with pseudo-header and optional meta-data TLVs, PHY payload
 * exactly as it appears in the spec (no padding, no nothing), and FCS if
 * specified by FCS Type TLV;  requested by James Ko <jck@exegin.com>.
 * Specification at https://github.com/jkcko/ieee802.15.4-tap
 */
#define DLT_IEEE802_15_4_TAP    283

/*
 * Marvell (Ethertype) Distributed Switch Architecture proprietary tagging format.
 */
#define DLT_DSA_TAG_DSA        284
#define DLT_DSA_TAG_EDSA    285

/*
 * Payload of lawful intercept packets using the ELEE protocol;
 * https://socket.hr/draft-dfranusic-opsawg-elee-00.xml
 * https://xml2rfc.tools.ietf.org/cgi-bin/xml2rfc.cgi?url=https://socket.hr/draft-dfranusic-opsawg-elee-00.xml&modeAsFormat=html/ascii
 */
#define DLT_ELEE        286

/*
 * Serial frames transmitted between a host and a Z-Wave chip.
 */
#define DLT_Z_WAVE_SERIAL    287

/*
 * USB 2.0, 1.1, and 1.0 packets as transmitted over the cable.
 */
#define DLT_USB_2_0        288

/*
 * ATSC Link-Layer Protocol (A/330) packets.
 */
#define DLT_ATSC_ALP        289

/*
 * In case the code that includes this file (directly or indirectly)
 * has also included OS files that happen to define DLT_MATCHING_MAX,
 * with a different value (perhaps because that OS hasn't picked up
 * the latest version of our DLT definitions), we undefine the
 * previous value of DLT_MATCHING_MAX.
 */
#ifdef DLT_MATCHING_MAX
#undef DLT_MATCHING_MAX
#endif
#define DLT_MATCHING_MAX    289    /* highest value in the "matching" range */

/*
 * DLT and savefile link type values are split into a class and
 * a member of that class.  A class value of 0 indicates a regular
 * DLT_/LINKTYPE_ value.
 */
#define DLT_CLASS(x)        ((x) & 0x03ff0000)
/*
 * NetBSD-specific generic "raw" link type.  The class value indicates
 * that this is the generic raw type, and the lower 16 bits are the
 * address family we're dealing with.  Those values are NetBSD-specific;
 * do not assume that they correspond to AF_ values for your operating
 * system.
 */
# 定义 NetBSD 特定的通用“原始”链路类型。类值表示这是通用的原始类型，低 16 位是我们处理的地址族。这些值是特定于 NetBSD 的；不要假设它们对应于您操作系统的 AF_ 值。
#define    DLT_CLASS_NETBSD_RAWAF    0x02240000
# 定义一个宏，用于创建特定地址族的 NetBSD 原始链路类型
#define    DLT_NETBSD_RAWAF(af)    (DLT_CLASS_NETBSD_RAWAF | (af))
# 定义一个宏，用于获取给定值的地址族
#define    DLT_NETBSD_RAWAF_AF(x)    ((x) & 0x0000ffff)
# 定义一个宏，用于检查给定值是否是 NetBSD 原始链路类型
#define    DLT_IS_NETBSD_RAWAF(x)    (DLT_CLASS(x) == DLT_CLASS_NETBSD_RAWAF)

#endif /* !defined(lib_pcap_dlt_h) */
```