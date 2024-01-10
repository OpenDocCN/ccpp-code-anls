# `nmap\libpcap\pcap-common.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 条件：（1）源代码发布时保留以上版权声明和本段文字，（2）包含二进制代码的发布包括以上版权声明和本段文字在文档或其他提供的材料中，（3）所有提及此软件特性或使用的广告材料都显示以下声明：“本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 * pcap-common.c - pcap和pcapng文件的通用代码
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include "pcap-int.h"

#include "pcap-common.h"

#define LINKTYPE_NULL        DLT_NULL
#define LINKTYPE_ETHERNET    DLT_EN10MB    /* 也适用于100Mb及以上 */
#define LINKTYPE_EXP_ETHERNET    DLT_EN3MB    /* 3Mb实验性以太网 */
#define LINKTYPE_AX25        DLT_AX25
#define LINKTYPE_PRONET        DLT_PRONET
#define LINKTYPE_CHAOS        DLT_CHAOS
#define LINKTYPE_IEEE802_5    DLT_IEEE802    /* DLT_IEEE802用于802.5令牌环 */
#define LINKTYPE_ARCNET_BSD    DLT_ARCNET    /* BSD风格的头部 */
#define LINKTYPE_SLIP        DLT_SLIP
#define LINKTYPE_PPP        DLT_PPP
#define LINKTYPE_FDDI        DLT_FDDI
/*
 * 定义 LINKTYPE_PPP_HDLC 为 50，用于在数据包开头可能有 RFC 1662 PPP 在 HDLC 类似封装头部（在 PPP 协议字段之前有 0xff 0x03）的情况下使用。
 *
 * 这是用于总是有这样的头部的情况；地址字段可能为 0xff，用于常规 PPP，或者可能是 Cisco 点对点带有 HDLC 封装的地址字段，如 RFC 1547 的第 4.3.1 节中所述（“Cisco HDLC”）。例如，这是 NetBSD 的 DLT_PPP_SERIAL 所得到的内容。
 *
 * 我们给它与 NetBSD 的 DLT_PPP_SERIAL 相同的值，希望没有其他人会选择 DLT_ 值为 50，以便 DLT_PPP_SERIAL 捕获将以 NetBSD 的 tcpdump 可以读取的链路类型写出。
 */
#define LINKTYPE_PPP_HDLC    50        /* PPP in HDLC-like framing */

/*
 * 定义 LINKTYPE_PPP_ETHER 为 51，用于 NetBSD PPP-over-Ethernet
 */
#define LINKTYPE_PPP_ETHER    51        /* NetBSD PPP-over-Ethernet */

/*
 * 定义 LINKTYPE_SYMANTEC_FIREWALL 为 99，用于 Symantec Enterprise Firewall
 */
#define LINKTYPE_SYMANTEC_FIREWALL 99        /* Symantec Enterprise Firewall */

/*
 * 这些对应于在不同平台上具有不同值的 DLT_ 值；我们在捕获文件和由 pcap_datalink() 返回并传递给 pcap_open_dead() 的 DLT_ 值之间进行映射。
 */
#define LINKTYPE_ATM_RFC1483    100        /* LLC/SNAP-encapsulated ATM */
#define LINKTYPE_RAW        101        /* raw IP */
#define LINKTYPE_SLIP_BSDOS    102        /* BSD/OS SLIP BPF header */
#define LINKTYPE_PPP_BSDOS    103        /* BSD/OS PPP BPF header */

/*
 * 以 104 开头的值用于新分配的链路层头部类型值；对于这些链路层头部类型，由 pcap_datalink() 返回并传递给 pcap_open_dead() 的 DLT_ 值，以及在捕获文件中出现的 LINKTYPE_ 值是相同的。
 *
 * LINKTYPE_MATCHING_MIN 是最小的这样的值；LINKTYPE_MATCHING_MAX 是最大的这样的值。
 */
#define LINKTYPE_MATCHING_MIN    104        /* lowest value in the "matching" range */

/*
 * 定义 LINKTYPE_C_HDLC 为 104，用于 Cisco HDLC
 */
#define LINKTYPE_C_HDLC        104        /* Cisco HDLC */
# 定义各种链路类型的常量，用于标识不同类型的网络数据链路
#define LINKTYPE_IEEE802_11    105        /* IEEE 802.11 (wireless) */
#define LINKTYPE_ATM_CLIP    106        /* Linux Classical IP over ATM */
#define LINKTYPE_FRELAY        107        /* Frame Relay */
#define LINKTYPE_LOOP        108        /* OpenBSD loopback */
#define LINKTYPE_ENC        109        /* OpenBSD IPSEC enc */

/*
 * 以下两种类型保留以供将来使用。
 */
#define LINKTYPE_LANE8023    110        /* ATM LANE + 802.3 */
#define LINKTYPE_HIPPI        111        /* NetBSD HIPPI */

/*
 * 用于 NetBSD DLT_HDLC；从 NetBSD 中使用它的一个驱动程序来看，它是 Cisco HDLC，因此与 DLT_C_HDLC/LINKTYPE_C_HDLC 相同，但我们定义了一个单独的值，以避免在 NetBSD 上的一些程序兼容性问题。
 *
 * 所有代码应该将 LINKTYPE_NETBSD_HDLC 和 LINKTYPE_C_HDLC 视为相同。
 */
#define LINKTYPE_NETBSD_HDLC    112        /* NetBSD HDLC framing */

#define LINKTYPE_LINUX_SLL    113        /* Linux cooked socket capture */
#define LINKTYPE_LTALK        114        /* Apple LocalTalk hardware */
#define LINKTYPE_ECONET        115        /* Acorn Econet */

/*
 * 保留供 OpenBSD ipfilter 使用。
 */
#define LINKTYPE_IPFILTER    116

#define LINKTYPE_PFLOG        117        /* OpenBSD DLT_PFLOG */
#define LINKTYPE_CISCO_IOS    118        /* 供 Cisco 内部使用 */
#define LINKTYPE_IEEE802_11_PRISM 119        /* 802.11 加 Prism II 监视模式无线电元数据头 */
#define LINKTYPE_IEEE802_11_AIRONET 120        /* 802.11 加 FreeBSD Aironet 驱动程序无线电元数据头 */

/*
 * 保留供 Siemens HiPath HDLC 使用。
 */
#define LINKTYPE_HHDLC        121

#define LINKTYPE_IP_OVER_FC    122        /* RFC 2625 IP-over-Fibre Channel */
#define LINKTYPE_SUNATM        123        /* Solaris+SunATM */

/*
 * 根据 Kent Dahlgren <kent@praesum.com> 的请求保留为私人使用。
 */
#define LINKTYPE_RIO        124        /* RapidIO */
#define LINKTYPE_PCI_EXP    125        /* PCI Express */
# 定义 LINKTYPE_AURORA 常量，表示 Xilinx Aurora 链路层
#define LINKTYPE_AURORA        126        /* Xilinx Aurora link layer */

# 定义 LINKTYPE_IEEE802_11_RADIOTAP 常量，表示 802.11 加上 radiotap 无线元数据头
#define LINKTYPE_IEEE802_11_RADIOTAP 127    /* 802.11 plus radiotap radio metadata header */

# 定义 LINKTYPE_TZSP 常量，表示 Tazmen Sniffer Protocol 封装
# 用于包含元信息的通用封装，例如 802.11 数据包的信号强度和信道
#define LINKTYPE_TZSP        128        /* Tazmen Sniffer Protocol */

# 定义 LINKTYPE_ARCNET_LINUX 常量，表示 Linux 风格的头部
#define LINKTYPE_ARCNET_LINUX    129        /* Linux-style headers */

# 定义 LINKTYPE_JUNIPER_MLPPP 到 LINKTYPE_JUNIPER_ATM1 常量，表示 Juniper 私有数据链路类型
# 用于传递机箱内部的元信息，例如 QOS 配置文件等
#define LINKTYPE_JUNIPER_MLPPP  130
#define LINKTYPE_JUNIPER_MLFR   131
#define LINKTYPE_JUNIPER_ES     132
#define LINKTYPE_JUNIPER_GGSN   133
#define LINKTYPE_JUNIPER_MFR    134
#define LINKTYPE_JUNIPER_ATM2   135
#define LINKTYPE_JUNIPER_SERVICES 136
#define LINKTYPE_JUNIPER_ATM1   137

# 定义 LINKTYPE_APPLE_IP_OVER_IEEE1394 常量，表示 Apple IP-over-IEEE 1394 的封装头
#define LINKTYPE_APPLE_IP_OVER_IEEE1394 138    /* Apple IP-over-IEEE 1394 cooked header */

# 定义 LINKTYPE_MTP2_WITH_PHDR 到 LINKTYPE_SCCP 常量，表示 MTP2 到 SCCP 的链路类型
#define LINKTYPE_MTP2_WITH_PHDR    139
#define LINKTYPE_MTP2        140
#define LINKTYPE_MTP3        141
#define LINKTYPE_SCCP        142

# 定义 LINKTYPE_DOCSIS 常量，表示 DOCSIS MAC 帧
#define LINKTYPE_DOCSIS        143        /* DOCSIS MAC frames */

# 定义 LINKTYPE_LINUX_IRDA 常量，表示 Linux-IrDA
#define LINKTYPE_LINUX_IRDA    144        /* Linux-IrDA */

# 定义 LINKTYPE_IBM_SP 和 LINKTYPE_IBM_SN 常量，表示 IBM SP switch 和 IBM Next Federation switch 的链路类型
#define LINKTYPE_IBM_SP        145
#define LINKTYPE_IBM_SN        146
/*
 * 保留供私人使用。如果您有一些链路层头部类型想要在您的组织内使用，并且使用该链路层头部类型的捕获文件不会被发送到您的组织之外，那么您可以使用这些数值。
 *
 * 任何 libpcap 发行版都不会为任何目的使用这些数值，任何 tcpdump 发行版也不会使用它们。
 *
 * *不要*在您期望任何不使用您私人版本的捕获文件读取工具的人能够读取的捕获文件中使用这些数值；特别是，*不要*在产品中使用它们，否则您可能会发现人们无法使用 tcpdump、snort、Ethereal 等工具来读取来自您的防火墙/入侵检测/流量监控等设备的捕获文件，或者使用该 LINKTYPE_ 值的任何产品，您还可能发现这些应用程序的开发人员不会接受用于让它们读取这些文件的补丁。
 *
 * 同样，如果有人可能会使用这些数值发送捕获文件给您，而使用它们的工具需要读取您的私人类型的捕获文件，那么也不要使用它们。
 *
 * 在这些情况下，相反地，请根据 pcap/bpf.h 中的注释向 "tcpdump-workers@lists.tcpdump.org" 请求一个新的 DLT_ 和 LINKTYPE_ 值，并使用您得到的类型。
 */
#define LINKTYPE_USER0        147
#define LINKTYPE_USER1        148
#define LINKTYPE_USER2        149
#define LINKTYPE_USER3        150
#define LINKTYPE_USER4        151
#define LINKTYPE_USER5        152
#define LINKTYPE_USER6        153
#define LINKTYPE_USER7        154
#define LINKTYPE_USER8        155
#define LINKTYPE_USER9        156
#define LINKTYPE_USER10        157
#define LINKTYPE_USER11        158
#define LINKTYPE_USER12        159
#define LINKTYPE_USER13        160
#define LINKTYPE_USER14        161
#define LINKTYPE_USER15        162
/*
 * 用于将来与 802.11 捕获一起使用 - 由 AbsoluteValue Systems 定义，用于存储一定数量的链路层信息，包括无线电信息：
 *    http://www.shaftnet.org/~pizza/software/capturefrm.txt
 */
#define LINKTYPE_IEEE802_11_AVS    163    /* 802.11 加 AVS 无线电元数据头 */

/*
 * Juniper 私有数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求定义。相应的 DLT_s 用于传递机箱内部的元信息，如 QOS 配置文件等。
 */
#define LINKTYPE_JUNIPER_MONITOR 164

/*
 * BACnet MS/TP 帧。
 */
#define LINKTYPE_BACNET_MS_TP    165

/*
 * 另一种 PPP 变体，由 Karsten Keil <kkeil@suse.de> 请求定义。
 *
 * 在某些操作系统中，用于允许内核套接字过滤器区分传入和传出的数据包，用于供 pppd 使用的套接字，以便它可以进行按需拨号和缺乏需求时挂断；传入的数据包被过滤掉，以免导致 pppd 保持连接（不希望随机输入数据包，如端口扫描、来自旧丢失连接的数据包等，强制连接保持）。
 *
 * PPP 头的第一个字节（0xff03）被修改以适应方向 - 0x00 = IN，0x01 = OUT。
 */
#define LINKTYPE_PPP_PPPD    166

/*
 * Juniper 私有数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求定义。DLT_s 用于传递机箱内部的元信息，如 QOS 配置文件、cookies 等。
 */
#define LINKTYPE_JUNIPER_PPPOE     167
#define LINKTYPE_JUNIPER_PPPOE_ATM 168

#define LINKTYPE_GPRS_LLC    169        /* GPRS LLC */
#define LINKTYPE_GPF_T        170        /* GPF-T (ITU-T G.7041/Y.1303) */
#define LINKTYPE_GPF_F        171        /* GPF-F (ITU-T G.7041/Y.1303) */

/*
 * 由 Oolan Zimmer <oz@gcom.com> 请求，用于在 Gcom 的 T1/E1 线路监测设备中使用。
 */
#define LINKTYPE_GCOM_T1E1    172
# 定义了一系列数据链路类型的常量，用于标识不同类型的数据链路
#define LINKTYPE_GCOM_SERIAL    173

/*
 * Juniper 私有的数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求添加。
 * DLT_ 用于内部与物理接口卡 (PIC) 的通信
 */
#define LINKTYPE_JUNIPER_PIC_PEER    174

/*
 * Gregor Maier <gregor@endace.com> of Endace 测量系统请求的数据链路类型。
 * 它们在链路层头部之前添加了一个 ERF 头部 (参见 https://www.endace.com/support/EndaceRecordFormat.pdf)。
 */
#define LINKTYPE_ERF_ETH    175    /* 以太网 */
#define LINKTYPE_ERF_POS    176    /* Packet-over-SONET */

/*
 * Daniele Orlandi <daniele@orlandi.com> 为 vISDN (http://www.orlandi.com/visdn/) 请求的原始 LAPD 数据链路类型。
 * 其链路层头部在 LAPD 头部之前包含附加信息，因此不一定是通用的 LAPD 头部。
 */
#define LINKTYPE_LINUX_LAPD    177

/*
 * Juniper 私有的数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求添加。
 * 这些链路类型用于在标准以太网、PPP、Frelay 和 C-HDLC 帧之前添加接口索引、接口名称等元信息。
 */
#define LINKTYPE_JUNIPER_ETHER  178
#define LINKTYPE_JUNIPER_PPP    179
#define LINKTYPE_JUNIPER_FRELAY 180
#define LINKTYPE_JUNIPER_CHDLC  181

/*
 * 多链路帧中继 (FRF.16)
 */
#define LINKTYPE_MFR            182

/*
 * Juniper 私有的数据链路类型，根据 Hannes Gredler <hannes@juniper.net> 的请求添加。
 * DLT_ 用于与语音适配卡 (PIC) 进行内部通信
 */
#define LINKTYPE_JUNIPER_VP     183

/*
 * Arinc 429 帧。
 * Gianluca Varenni <gianluca.varenni@cacetech.com> 请求的 DLT_。
 * 每个帧包含一个 32 位的 A429 标签。
 * Arinc 429 的更多文档可以在 https://web.archive.org/web/20040616233302/https://www.condoreng.com/support/downloads/tutorials/ARINCTutorial.pdf 找到。
 */
#define LINKTYPE_A429           184
/*
 * Arinc 653 Interpartition Communication messages.
 * DLT_ requested by Gianluca Varenni <gianluca.varenni@cacetech.com>.
 * Please refer to the A653-1 standard for more information.
 */
#define LINKTYPE_A653_ICM       185

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
 */
#define LINKTYPE_USB_FREEBSD    186

/*
 * Bluetooth HCI UART transport layer (part H:4); requested by
 * Paolo Abeni.
 */
#define LINKTYPE_BLUETOOTH_HCI_H4    187

/*
 * IEEE 802.16 MAC Common Part Sublayer; requested by Maria Cruz
 * <cruz_petagay@bah.com>.
 */
#define LINKTYPE_IEEE802_16_MAC_CPS    188

/*
 * USB packets, beginning with a Linux USB header; requested by
 * Paolo Abeni <paolo.abeni@email.it>.
 */
#define LINKTYPE_USB_LINUX        189

/*
 * Controller Area Network (CAN) v. 2.0B packets.
 * DLT_ requested by Gianluca Varenni <gianluca.varenni@cacetech.com>.
 * Used to dump CAN packets coming from a CAN Vector board.
 * More documentation on the CAN v2.0B frames can be found at
 * http://www.can-cia.org/downloads/?269
 */
#define LINKTYPE_CAN20B         190

/*
 * IEEE 802.15.4, with address fields padded, as is done by Linux
 * drivers; requested by Juergen Schimmer.
 */
#define LINKTYPE_IEEE802_15_4_LINUX    191

/*
 * Per Packet Information encapsulated packets.
 * LINKTYPE_ requested by Gianluca Varenni <gianluca.varenni@cacetech.com>.
 */
#define LINKTYPE_PPI            192

/*
 * Header for 802.16 MAC Common Part Sublayer plus a radiotap radio header;
 * requested by Charles Clancy.
 */
#define LINKTYPE_IEEE802_16_MAC_CPS_RADIO    193
/*
 * Juniper-private data link type, as per request from
 * Hannes Gredler <hannes@juniper.net>.
 * The DLT_ is used for internal communication with a
 * integrated service module (ISM).
 */
#define LINKTYPE_JUNIPER_ISM    194

/*
 * IEEE 802.15.4, exactly as it appears in the spec (no padding, no
 * nothing), and with the FCS at the end of the frame; requested by
 * Mikko Saarnivala <mikko.saarnivala@sensinode.com>.
 *
 * This should only be used if the FCS is present at the end of the
 * frame; if the frame has no FCS, DLT_IEEE802_15_4_NOFCS should be
 * used.
 */
#define LINKTYPE_IEEE802_15_4_WITHFCS    195

/*
 * Various link-layer types, with a pseudo-header, for SITA
 * (https://www.sita.aero/); requested by Fulko Hew (fulko.hew@gmail.com).
 */
#define LINKTYPE_SITA        196

/*
 * Various link-layer types, with a pseudo-header, for Endace DAG cards;
 * encapsulates Endace ERF records.  Requested by Stephen Donnelly
 * <stephen@endace.com>.
 */
#define LINKTYPE_ERF        197

/*
 * Special header prepended to Ethernet packets when capturing from a
 * u10 Networks board.  Requested by Phil Mulholland
 * <phil@u10networks.com>.
 */
#define LINKTYPE_RAIF1        198

/*
 * IPMB packet for IPMI, beginning with a 2-byte header, followed by
 * the I2C slave address, followed by the netFn and LUN, etc..
 * Requested by Chanthy Toeung <chanthy.toeung@ca.kontron.com>.
 *
 * XXX - its DLT_ value used to be called DLT_IPMB, back when we got the
 * impression from the email thread requesting it that the packet
 * had no extra 2-byte header.  We've renamed it; if anybody used
 * DLT_IPMB and assumed no 2-byte header, this will cause the compile
 * to fail, at which point we'll have to figure out what to do about
 * the two header types using the same DLT_/LINKTYPE_ value.  If that
 * doesn't happen, we'll assume nobody used it and that the redefinition
 * is safe.
 */
#define LINKTYPE_IPMB_KONTRON    199
/*
 * Juniper-private data link type, as per request from
 * Hannes Gredler <hannes@juniper.net>.
 * The DLT_ is used for capturing data on a secure tunnel interface.
 */
#define LINKTYPE_JUNIPER_ST     200

/*
 * Bluetooth HCI UART transport layer (part H:4), with pseudo-header
 * that includes direction information; requested by Paolo Abeni.
 */
#define LINKTYPE_BLUETOOTH_HCI_H4_WITH_PHDR    201

/*
 * AX.25 packet with a 1-byte KISS header; see
 *
 *    http://www.ax25.net/kiss.htm
 *
 * as per Richard Stearn <richard@rns-stearn.demon.co.uk>.
 */
#define LINKTYPE_AX25_KISS    202

/*
 * LAPD packets from an ISDN channel, starting with the address field,
 * with no pseudo-header.
 * Requested by Varuna De Silva <varunax@gmail.com>.
 */
#define LINKTYPE_LAPD        203

/*
 * PPP, with a one-byte direction pseudo-header prepended - zero means
 * "received by this host", non-zero (any non-zero value) means "sent by
 * this host" - as per Will Barker <w.barker@zen.co.uk>.
 */
#define LINKTYPE_PPP_WITH_DIR    204    /* Don't confuse with LINKTYPE_PPP_PPPD */

/*
 * Cisco HDLC, with a one-byte direction pseudo-header prepended - zero
 * means "received by this host", non-zero (any non-zero value) means
 * "sent by this host" - as per Will Barker <w.barker@zen.co.uk>.
 */
#define LINKTYPE_C_HDLC_WITH_DIR 205    /* Cisco HDLC */

/*
 * Frame Relay, with a one-byte direction pseudo-header prepended - zero
 * means "received by this host" (DCE -> DTE), non-zero (any non-zero
 * value) means "sent by this host" (DTE -> DCE) - as per Will Barker
 * <w.barker@zen.co.uk>.
 */
#define LINKTYPE_FRELAY_WITH_DIR 206    /* Frame Relay */

/*
 * LAPB, with a one-byte direction pseudo-header prepended - zero means
 * "received by this host" (DCE -> DTE), non-zero (any non-zero value)
 * means "sent by this host" (DTE -> DCE)- as per Will Barker
 * <w.barker@zen.co.uk>.
 */
#define LINKTYPE_LAPB_WITH_DIR    207    /* LAPB */
/*
 * 208是为Will Barker保留的尚未指定的专有链路层类型。
 */

/*
 * 具有Linux特定伪标头的IPMB；由Alexey Neyman <avn@pigeonpoint.com>请求。
 */
#define LINKTYPE_IPMB_LINUX    209

/*
 * FlexRay汽车总线 - http://www.flexray.com/ - 由Hannes Kaelber <hannes.kaelber@x2e.de>请求。
 */
#define LINKTYPE_FLEXRAY    210

/*
 * 用于多媒体传输的Media Oriented Systems Transport (MOST)总线 - https://www.mostcooperation.com/ - 由Hannes Kaelber <hannes.kaelber@x2e.de>请求。
 */
#define LINKTYPE_MOST        211

/*
 * 用于车辆网络的Local Interconnect Network (LIN)总线 - http://www.lin-subbus.org/ - 由Hannes Kaelber <hannes.kaelber@x2e.de>请求。
 */
#define LINKTYPE_LIN        212

/*
 * 用于串行线捕获的X2E私有数据链路类型，由Hannes Kaelber <hannes.kaelber@x2e.de>请求。
 */
#define LINKTYPE_X2E_SERIAL    213

/*
 * 用于Xoraya数据记录仪系列的X2E私有数据链路类型，由Hannes Kaelber <hannes.kaelber@x2e.de>请求。
 */
#define LINKTYPE_X2E_XORAYA    214

/*
 * IEEE 802.15.4，与规范中完全相同（没有填充，没有其他内容），但对于非ASK PHYs的PHY级数据（4个八位字节的0作为前导，一个八位字节的SFD，一个八位字节的帧长度+保留位，然后是以帧控制字段开头的MAC层数据）。
 *
 * Max Filippov <jcmvbkbc@gmail.com>请求。
 */
#define LINKTYPE_IEEE802_15_4_NONASK_PHY    215

/*
 * David Gibson <david@gibson.dropbear.id.au>请求从Linux内核/dev/input/eventN设备捕获的数据。这用于从Linux内核向显示系统（如Xorg）传输按键和鼠标移动。
 */
#define LINKTYPE_LINUX_EVDEV    216

/*
 * GSM Um和Abis接口，前面带有“gsmtap”标头。
 *
 * Harald Welte <laforge@gnumonks.org>请求。
 */
# 定义 LINKTYPE_GSMTAP_UM 的值为 217，表示 GSM Um 接口的数据包
#define LINKTYPE_GSMTAP_ABIS 的值为 218，表示 GSM Abis 接口的数据包

# 定义 LINKTYPE_MPLS 的值为 219，表示 MPLS 数据包，带有 MPLS 标签作为链路层头部
# 由 OpenBSD 的 Michele Marchetto <michele@openbsd.org> 代表 OpenBSD 请求添加

# 定义 LINKTYPE_USB_LINUX_MMAPPED 的值为 220，表示 USB 数据包，以 Linux USB 头部开始，USB 头部填充到 64 字节；用于内存映射访问

# 定义 LINKTYPE_DECT 的值为 221，表示 DECT 数据包，带有伪头部；由 Matthias Wenzel <tcpdump@mazzoo.de> 请求添加

# 定义 LINKTYPE_AOS 的值为 222，表示 AOS 数据包，用于 AOS Space Data Link Protocol；由 "Lidwa, Eric (GSFC-582.0)[SGT INC]" <eric.lidwa-1@nasa.gov> 请求添加

# 定义 LINKTYPE_WIHART 的值为 223，表示 Wireless HART (Highway Addressable Remote Transducer) 数据包，由 HART Communication Foundation IES/PAS 62591 请求添加；由 Sam Roberts <vieuxtech@gmail.com> 请求添加

# 定义 LINKTYPE_FC_2 的值为 224，表示 Fibre Channel FC-2 帧，以 Frame_Header 开始；由 Kahou Lei <kahou82@gmail.com> 请求添加

# 定义 LINKTYPE_FC_2_WITH_FRAME_DELIMS 的值为 225，表示 Fibre Channel FC-2 帧，以帧分隔符的编码开始和结束；由 Kahou Lei <kahou82@gmail.com> 请求添加
/*
 * Solaris ipnet pseudo-header; requested by Darren Reed <Darren.Reed@Sun.COM>.
 *
 * The pseudo-header starts with a one-byte version number; for version 2,
 * the pseudo-header is:
 *
 * struct dl_ipnetinfo {
 *     uint8_t   dli_version;    // 版本号，当前版本为2
 *     uint8_t   dli_family;     // Solaris地址族值，IPv4为2，IPv6为26
 *     uint16_t  dli_htype;      // “hook类型” - 0表示传入数据包，1表示传出数据包，2表示来自同一台机器上另一个区域的数据包
 *     uint32_t  dli_pktlen;     // 伪头部后面数据包的长度
 *     uint32_t  dli_ifindex;    // 数据包到达的接口索引
 *     uint32_t  dli_grifindex;  // 组接口索引号（用于IPMP接口）
 *     uint32_t  dli_zsrc;       // 数据包源的区域标识符
 *     uint32_t  dli_zdst;       // 数据包目的地的区域标识符
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
#define LINKTYPE_IPNET        226

/*
 * CAN (Controller Area Network) frames, with a pseudo-header as supplied
 * by Linux SocketCAN, and with multi-byte numerical fields in that header
 * in big-endian byte order.
 *
 * See Documentation/networking/can.txt in the Linux source.
 *
 * Requested by Felix Obenhuber <felix@obenhuber.de>.
 */
#define LINKTYPE_CAN_SOCKETCAN    227
/*
 * 定义原始的 IPv4/IPv6 数据链路类型；与 DLT_RAW 不同之处在于 DLT_ 值指定了是 v4 还是 v6。由 Darren Reed <Darren.Reed@Sun.COM> 请求添加。
 */
#define LINKTYPE_IPV4        228
#define LINKTYPE_IPV6        229

/*
 * IEEE 802.15.4，与规范完全一致（没有填充，没有其他内容），并且帧末尾没有 FCS；由 Jon Smirl <jonsmirl@gmail.com> 请求添加。
 */
#define LINKTYPE_IEEE802_15_4_NOFCS        230

/*
 * 原始的 D-Bus：
 *
 *    https://www.freedesktop.org/wiki/Software/dbus
 *
 * 消息：
 *
 *    https://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol-messages
 *
 * 从大小端标志开始，后面是消息类型等，但在消息序列之前没有身份验证握手：
 *
 *    https://dbus.freedesktop.org/doc/dbus-specification.html#auth-protocol
 *
 * 由 Martin Vidner <martin@vidner.net> 请求添加。
 */
#define LINKTYPE_DBUS        231

/*
 * Juniper 私有数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求添加。
 */
#define LINKTYPE_JUNIPER_VS            232
#define LINKTYPE_JUNIPER_SRX_E2E        233
#define LINKTYPE_JUNIPER_FIBRECHANNEL        234

/*
 * DVB-CI（用于 PC 卡模块与 DVB 接收器之间通信的 DVB 通用接口）。参见
 *
 *    https://www.kaiser.cx/pcap-dvbci.html
 *
 * 了解规范。
 *
 * 由 Martin Kaiser <martin@kaiser.cx> 请求添加。
 */
#define LINKTYPE_DVB_CI        235

/*
 * 3GPP TS 27.010 多路复用协议的变种。由 Hans-Christoph Schemmel <hans-christoph.schemmel@cinterion.com> 请求添加。
 */
#define LINKTYPE_MUX27010    236

/*
 * STANAG 5066 D_PDUs。由 M. Baris Demiray <barisdemiray@gmail.com> 请求添加。
 */
#define LINKTYPE_STANAG_5066_D_PDU        237

/*
 * Juniper 私有数据链路类型，由 Hannes Gredler <hannes@juniper.net> 请求添加。
 */
#define LINKTYPE_JUNIPER_ATM_CEMIC        238
/*
 * 定义 NetFilter LOG 消息的链路类型
 * （netlink NFNL_SUBSYS_ULOG/NFULNL_MSG_PACKET 数据包的有效载荷）
 *
 * Jakub Zawadzki <darkjames-ws@darkjames.pl> 请求添加
 */
#define LINKTYPE_NFLOG        239

/*
 * Hilscher Gesellschaft fuer Systemautomation mbH 的链路类型
 * 用于带有 4 字节伪头部的以太网数据包，并且始终包括 FCS 的有效载荷
 * 由他们的 netANALYZER 硬件和软件提供
 *
 * Holger P. Frommer <HPfrommer@hilscher.com> 请求添加
 */
#define LINKTYPE_NETANALYZER    240

/*
 * Hilscher Gesellschaft fuer Systemautomation mbH 的链路类型
 * 用于带有 4 字节伪头部和 FCS 以及 1 字节 SFD 的以太网数据包
 * 由他们的 netANALYZER 硬件和软件提供
 *
 * Holger P. Frommer <HPfrommer@hilscher.com> 请求添加
 */
#define LINKTYPE_NETANALYZER_TRANSPARENT    241

/*
 * IP-over-InfiniBand，由 RFC 4391 指定
 *
 * Petr Sumbera <petr.sumbera@oracle.com> 请求添加
 */
#define LINKTYPE_IPOIB        242

/*
 * MPEG-2 传输流（ISO 13818-1/ITU-T H.222.0）
 *
 * Guy Martin <gmsoft@tuxicoman.be> 请求添加
 */
#define LINKTYPE_MPEG_2_TS    243

/*
 * ng4T GmbH 的 UMTS Iub/Iur-over-ATM 和 Iub/Iur-over-IP 格式
 * 由他们的 ng40 协议测试仪使用
 *
 * Jens Grimmer <jens.grimmer@ng4t.com> 请求添加
 */
#define LINKTYPE_NG40        244

/*
 * 伪头部提供适配器编号和标志，后跟 NFC（近场通信）逻辑链路控制协议（LLCP）PDU
 * 由 NFC Forum Logical Link Control Protocol Technical Specification LLCP 1.1 指定
 *
 * Mike Wakerly <mikey@google.com> 请求添加
 */
#define LINKTYPE_NFC_LLCP    245
/*
 * 定义了一系列新的链路层头类型值，用于不同的网络协议或设备
 * 为了避免与现有的链路层头类型值冲突，选择了一些新的数值
 * 每个数值对应一个特定的网络协议或设备
 */

#define LINKTYPE_PFSYNC        246
// 用于 pfsync 输出的链路层头类型值

#define LINKTYPE_INFINIBAND    247
// 用于 InfiniBand 网络协议的链路层头类型值

#define LINKTYPE_SCTP        248
// 用于 SCTP 协议的链路层头类型值，不包含 IPv4 或 IPv6

#define LINKTYPE_USBPCAP    249
// 用于 USB 数据包的链路层头类型值，以 USBPcap 头开始

#define LINKTYPE_RTAC_SERIAL        250
// 用于 Schweitzer Engineering Laboratories "RTAC" 产品的串行线数据包的链路层头类型值

#define LINKTYPE_BLUETOOTH_LE_LL    251
// 用于蓝牙低功耗空中接口链路层数据包的链路层头类型值

#define LINKTYPE_WIRESHARK_UPPER_PDU    252
// 用于 Wireshark 上层协议 PDU 保存的链路层头类型值

#define LINKTYPE_NETLINK        253
// 用于 netlink 协议的链路层头类型值，用于 nlmon 设备

#define LINKTYPE_BLUETOOTH_LINUX_MONITOR    254
// 用于蓝牙 Linux 监视器的链路层头类型值
# 定义蓝牙 Linux 监视器的链路类型
#define LINKTYPE_BLUETOOTH_LINUX_MONITOR    254

# 定义蓝牙基本速率/增强数据速率基带数据包的链路类型
#define LINKTYPE_BLUETOOTH_BREDR_BB    255

# 定义蓝牙低功耗链路层数据包的链路类型
#define LINKTYPE_BLUETOOTH_LE_LL_WITH_PHDR    256

# 定义PROFIBUS数据链路层的链路类型
#define LINKTYPE_PROFIBUS_DL        257

# 定义苹果的DLT_PKTAP头部的链路类型
# 详细说明了苹果使用DLT_USER2的原因以及为了兼容性而定义的LINKTYPE_PKTAP
#define LINKTYPE_PKTAP        258

# 定义以太网数据包，前面带有802.3-2012规范中指定的前导码的最后6个八位字节的链路类型
#define LINKTYPE_EPON        259

# 定义IPMI跟踪数据包的链路类型，符合PICMG HPM.2规范中的"Trace Data Block Format"表格
#define LINKTYPE_IPMI_HPM_2    260

# 定义Zwave捕获的格式，由Joshua Wright提供
#define LINKTYPE_ZWAVE_R1_R2    261
#define LINKTYPE_ZWAVE_R3    262

# 定义Wattstopper数字照明管理室总线串行协议捕获的格式，由Steve Karg提供
# 定义 LINKTYPE_WATTSTOPPER_DLM 的数值为 263
#define LINKTYPE_WATTSTOPPER_DLM 263

# 定义 LINKTYPE_ISO_14443 的数值为 264
/*
 * ISO 14443 contactless smart card messages.
 */
#define LINKTYPE_ISO_14443      264

# 定义 LINKTYPE_RDS 的数值为 265
/*
 * Radio data system (RDS) groups.  IEC 62106.
 * Per Jonathan Brucker <jonathan.brucke@gmail.com>.
 */
#define LINKTYPE_RDS        265

# 定义 LINKTYPE_USB_DARWIN 的数值为 266
/*
 * USB packets, beginning with a Darwin (macOS, etc.) header.
 */
#define LINKTYPE_USB_DARWIN    266

# 定义 LINKTYPE_OPENFLOW 的数值为 267
/*
 * OpenBSD DLT_OPENFLOW.
 */
#define LINKTYPE_OPENFLOW    267

# 定义 LINKTYPE_SDLC 的数值为 268
/*
 * SDLC frames containing SNA PDUs.
 */
#define LINKTYPE_SDLC        268

# 定义 LINKTYPE_TI_LLN_SNIFFER 的数值为 269
/*
 * per "Selvig, Bjorn" <b.selvig@ti.com> used for
 * TI protocol sniffer.
 */
#define LINKTYPE_TI_LLN_SNIFFER    269

# 定义 LINKTYPE_LORATAP 的数值为 270
/*
 * per: Erik de Jong <erikdejong at gmail.com> for
 *   https://github.com/eriknl/LoRaTap/releases/tag/v0.1
 */
#define LINKTYPE_LORATAP        270

# 定义 LINKTYPE_VSOCK 的数值为 271
/*
 * per: Stefanha at gmail.com for
 *   https://lists.sandelman.ca/pipermail/tcpdump-workers/2017-May/000772.html
 * and: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/vsockmon.h
 * for: https://qemu-project.org/Features/VirtioVsock
 */
#define LINKTYPE_VSOCK          271

# 定义 LINKTYPE_NORDIC_BLE 的数值为 272
/*
 * Nordic Semiconductor Bluetooth LE sniffer.
 */
#define LINKTYPE_NORDIC_BLE    272

# 定义 LINKTYPE_DOCSIS31_XRA31 的数值为 273
/*
 * Excentis DOCSIS 3.1 RF sniffer (XRA-31)
 *   per: bruno.verstuyft at excentis.com
 *        https://www.xra31.com/xra-header
 */
#define LINKTYPE_DOCSIS31_XRA31    273

# 定义 LINKTYPE_ETHERNET_MPACKET 的数值为 274
/*
 * mPackets, as specified by IEEE 802.3br Figure 99-4, starting
 * with the preamble and always ending with a CRC field.
 */
#define LINKTYPE_ETHERNET_MPACKET    274

# 定义 LINKTYPE_DISPLAYPORT_AUX 的数值为 275
/*
 * DisplayPort AUX channel monitoring data as specified by VESA
 * DisplayPort(DP) Standard preceded by a pseudo-header.
 *    per dirk.eibach at gdsys.cc
 */
#define LINKTYPE_DISPLAYPORT_AUX    275

# 定义 LINKTYPE_LINUX_SLL2 的数值为 276
/*
 * Linux cooked sockets v2.
 */
#define LINKTYPE_LINUX_SLL2    276

# 定义 LINKTYPE_SERCOS_MONITOR 的数值为 277
/*
 * Sercos Monitor, per Manuel Jacob <manuel.jacob at steinbeis-stg.de>
 */
#define LINKTYPE_SERCOS_MONITOR 277
/*
 * 定义 OpenVizsla 的数据封装格式编号
 * 参考链接：https://github.com/matwey/libopenvizsla/wiki/OpenVizsla-protocol-description
 */
#define LINKTYPE_OPENVIZSLA     278

/*
 * 定义 Elektrobit High Speed Capture and Replay (EBHSCR) 协议的编号
 * 参考链接：https://www.elektrobit.com/ebhscr
 * 联系人：Guenter.Ebermann at elektrobit.com
 */
#define LINKTYPE_EBHSCR            279

/*
 * 定义 https://fd.io vpp 图形调度跟踪器的数据包格式编号
 * 参考链接：https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/vnet.html#graph-dispatcher-pcap-tracing
 */
#define LINKTYPE_VPP_DISPATCH    280

/*
 * 定义 Broadcom 以太网交换机 (ROBO switch) 的 4 字节专有标记格式的编号
 */
#define LINKTYPE_DSA_TAG_BRCM    281
#define LINKTYPE_DSA_TAG_BRCM_PREPEND    282

/*
 * 定义带有伪头部和可选元数据 TLV 的 IEEE 802.15.4 数据包格式的编号
 * 请求者：James Ko <jck@exegin.com>
 * 规范链接：https://github.com/jkcko/ieee802.15.4-tap
 */
#define LINKTYPE_IEEE802_15_4_TAP       283

/*
 * 定义 Marvell (Ethertype) 分布式交换架构的专有标记格式的编号
 */
#define LINKTYPE_DSA_TAG_DSA    284
#define LINKTYPE_DSA_TAG_EDSA    285

/*
 * 定义使用 ELEE 协议的合法拦截数据包的有效载荷格式的编号
 * 规范链接：https://socket.hr/draft-dfranusic-opsawg-elee-00.xml
 * 规范链接：https://xml2rfc.tools.ietf.org/cgi-bin/xml2rfc.cgi?url=https://socket.hr/draft-dfranusic-opsawg-elee-00.xml&modeAsFormat=html/ascii
 */
#define LINKTYPE_ELEE        286
/*
 * 定义 Z-Wave 芯片和主机之间传输的串行帧。
 */
#define LINKTYPE_Z_WAVE_SERIAL    287

/*
 * USB 2.0、1.1 和 1.0 数据包在电缆上传输的格式。
 */
#define LINKTYPE_USB_2_0    288

/*
 * ATSC Link-Layer Protocol (A/330) 数据包。
 */
#define LINKTYPE_ATSC_ALP    289

#define LINKTYPE_MATCHING_MAX    289        /* "matching" 范围内的最大值 */

/*
 * "matching" 范围内的 DLT_ 和 LINKTYPE_ 值应该相同，因此 DLT_MATCHING_MAX 和 LINKTYPE_MATCHING_MAX 应该相同。
 */
#if LINKTYPE_MATCHING_MAX != DLT_MATCHING_MAX
#error LINKTYPE_ matching 范围与 DLT_ matching 范围不匹配
#endif

static struct linktype_map {
    int    dlt;
    int    linktype;
} map[] = {
    /*
     * 这些 DLT_* 代码具有与相应 DLT_* 代码的值相同的 LINKTYPE_* 代码。
     */
    { DLT_NULL,        LINKTYPE_NULL },
    { DLT_EN10MB,        LINKTYPE_ETHERNET },
    { DLT_EN3MB,        LINKTYPE_EXP_ETHERNET },
    { DLT_AX25,        LINKTYPE_AX25 },
    { DLT_PRONET,        LINKTYPE_PRONET },
    { DLT_CHAOS,        LINKTYPE_CHAOS },
    { DLT_IEEE802,        LINKTYPE_IEEE802_5 },
    { DLT_ARCNET,        LINKTYPE_ARCNET_BSD },
    { DLT_SLIP,        LINKTYPE_SLIP },
    { DLT_PPP,        LINKTYPE_PPP },
    { DLT_FDDI,        LINKTYPE_FDDI },
    { DLT_SYMANTEC_FIREWALL, LINKTYPE_SYMANTEC_FIREWALL },

    /*
     * 这些 DLT_* 代码在不同平台上具有不同的值；我们将它们映射到 LINKTYPE_* 代码，
     * 这些代码的值永远不会等于任何 DLT_* 代码的值。
     */
#ifdef DLT_FR
    /* BSD/OS Frame Relay */
    { DLT_FR,        LINKTYPE_FRELAY },
#endif

    { DLT_ATM_RFC1483,    LINKTYPE_ATM_RFC1483 },
    { DLT_RAW,        LINKTYPE_RAW },
    { DLT_SLIP_BSDOS,    LINKTYPE_SLIP_BSDOS },
    { DLT_PPP_BSDOS,    LINKTYPE_PPP_BSDOS },
    { DLT_HDLC,        LINKTYPE_NETBSD_HDLC },

    /* BSD/OS Cisco HDLC */
    # 将 DLT_C_HDLC 映射为 LINKTYPE_C_HDLC
    { DLT_C_HDLC,        LINKTYPE_C_HDLC },

    /*
     * 这些 DLT_* 代码并非所有平台都有，但是目前为止，
     * 似乎没有任何平台定义了其他具有相同数值的代码；我们仍然将它们映射到
     * 不同的 LINKTYPE_* 值，以防万一。
     */

    /* Linux ATM Classical IP */
    # 将 DLT_ATM_CLIP 映射为 LINKTYPE_ATM_CLIP
    { DLT_ATM_CLIP,        LINKTYPE_ATM_CLIP },

    /* NetBSD 同步/异步串行 PPP（或者 Cisco HDLC） */
    # 将 DLT_PPP_SERIAL 映射为 LINKTYPE_PPP_HDLC
    { DLT_PPP_SERIAL,    LINKTYPE_PPP_HDLC },

    /* NetBSD 以太网上的 PPP */
    # 将 DLT_PPP_ETHER 映射为 LINKTYPE_PPP_ETHER
    { DLT_PPP_ETHER,    LINKTYPE_PPP_ETHER },

    /*
     * LINKTYPE_ 值在 LINKTYPE_MATCHING_MIN 和 LINKTYPE_MATCHING_MAX 之间的所有值
     * 都映射为相同的 DLT_ 值。
     */

    # 将 -1 映射为 -1
    { -1,            -1 }
};

int
dlt_to_linktype(int dlt)
{
    int i;

    /*
     * DLTs that, on some platforms, have values in the matching range
     * but that *don't* have the same value as the corresponding
     * LINKTYPE because, for some reason, not all OSes have the
     * same value for that DLT (note that the DLT's value might be
     * outside the matching range on some of those OSes).
     */
    if (dlt == DLT_PFSYNC)
        return (LINKTYPE_PFSYNC);
    if (dlt == DLT_PKTAP)
        return (LINKTYPE_PKTAP);

    /*
     * For all other values in the matching range, the DLT
     * value is the same as the LINKTYPE value.
     */
    if (dlt >= DLT_MATCHING_MIN && dlt <= DLT_MATCHING_MAX)
        return (dlt);

    /*
     * Map the values outside that range.
     */
    for (i = 0; map[i].dlt != -1; i++) {
        if (map[i].dlt == dlt)
            return (map[i].linktype);
    }

    /*
     * If we don't have a mapping for this DLT, return an
     * error; that means that this is a value with no corresponding
     * LINKTYPE, and we need to assign one.
     */
    return (-1);
}

int
linktype_to_dlt(int linktype)
{
    int i;

    /*
     * LINKTYPEs in the matching range that *don't*
     * have the same value as the corresponding DLTs
     * because, for some reason, not all OSes have the
     * same value for that DLT.
     */
    if (linktype == LINKTYPE_PFSYNC)
        return (DLT_PFSYNC);
    if (linktype == LINKTYPE_PKTAP)
        return (DLT_PKTAP);

    /*
     * For all other values in the matching range, except for
     * LINKTYPE_ATM_CLIP, the LINKTYPE value is the same as
     * the DLT value.
     *
     * LINKTYPE_ATM_CLIP is a special case.  DLT_ATM_CLIP is
     * not on all platforms, but, so far, there don't appear
     * to be any platforms that define it as anything other
     * than 19; we define LINKTYPE_ATM_CLIP as something
     * other than 19, just in case.  That value is in the
     * matching range, so we have to check for it.
     */
    # 如果链路类型在指定范围内，并且不是指定的链路类型，则返回该链路类型
    if (linktype >= LINKTYPE_MATCHING_MIN &&
        linktype <= LINKTYPE_MATCHING_MAX &&
        linktype != LINKTYPE_ATM_CLIP)
        return (linktype);

    '''
     * 在指定范围外的数值进行映射处理
     '''
    # 遍历映射表，查找与给定链路类型相匹配的映射值
    for (i = 0; map[i].linktype != -1; i++) {
        if (map[i].linktype == linktype)
            return (map[i].dlt);
    }

    '''
     * 如果没有该链路类型的映射条目，则返回链路类型值；这可能是来自较新版本 libpcap 的 DLT
     '''
    # 如果没有该链路类型的映射条目，则返回链路类型值；这可能是来自较新版本 libpcap 的 DLT
    return linktype;
/*
 * 为给定的 DLT_ 值返回最大快照长度。
 *
 * 对于大多数链路层类型，我们使用 MAXIMUM_SNAPLEN。
 *
 * 对于 DLT_DBUS，最大长度为 128MiB，根据
 *
 *    https://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol-messages
 *
 * 对于 DLT_EBHSCR，最大长度为 8MiB，根据
 *
 *    https://www.elektrobit.com/ebhscr
 *
 * 对于 DLT_USBPCAP，最大长度为 1MiB，根据
 *
 *    https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=15985
 */
u_int
max_snaplen_for_dlt(int dlt)
{
    switch (dlt) {

    case DLT_DBUS:
        return 128*1024*1024;

    case DLT_EBHSCR:
        return 8*1024*1024;

    case DLT_USBPCAP:
        return 1024*1024;

    default:
        return MAXIMUM_SNAPLEN;
    }
}
```