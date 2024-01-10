# `nmap\libpcap\pcap\nflog.h`

```
/*
 * 版权声明
 * 2013年，Petar Alilovic，Faculty of Electrical Engineering and Computing, University of Zagreb
 * 保留所有权利
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都必须满足以下条件：
 *
 * * 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * * 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *
 * 本软件由作者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。无论在任何情况下，作者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）承担任何责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害。
 */

#ifndef lib_pcap_nflog_h
#define lib_pcap_nflog_h

#include <pcap/pcap-inttypes.h>

/*
 * NFLOG 头部和 TLV 部分的结构，如 https://www.tcpdump.org/linktypes/LINKTYPE_NFLOG.html 中所述
 *
 * NFLOG 头部是大端序的
 *
 * TLV 长度和类型以主机字节顺序存储。值要么是大端序，要么是某种外部指定字节顺序的字节数组（文本字符串、链路层地址、链路层头部、数据包数据等）
 */
typedef struct nflog_hdr {
    uint8_t        nflog_family;    /* 地址族 */
    # 定义一个8位无符号整数变量，用于存储版本信息
    uint8_t        nflog_version;    /* version */
    # 定义一个16位无符号整数变量，用于存储资源ID
    uint16_t    nflog_rid;    /* resource ID */
} nflog_hdr_t;

typedef struct nflog_tlv {
    uint16_t    tlv_length;    /* TLV的长度 */
    uint16_t    tlv_type;    /* TLV的类型 */
    /* 值跟在这之后 */
} nflog_tlv_t;

typedef struct nflog_packet_hdr {
    uint16_t    hw_protocol;    /* 硬件协议 */
    uint8_t        hook;        /* netfilter钩子 */
    uint8_t        pad;        /* 填充到32位 */
} nflog_packet_hdr_t;

typedef struct nflog_hwaddr {
    uint16_t    hw_addrlen;    /* 地址长度 */
    uint16_t    pad;        /* 填充到32位边界 */
    uint8_t        hw_addr[8];    /* 地址，最多8个字节 */
} nflog_hwaddr_t;

typedef struct nflog_timestamp {
    uint64_t    sec;
    uint64_t    usec;
} nflog_timestamp_t;

/*
 * TLV类型。
 */
#define NFULA_PACKET_HDR        1    /* nflog_packet_hdr_t */
#define NFULA_MARK            2    /* 来自skbuff的数据包标记 */
#define NFULA_TIMESTAMP            3    /* skbuff的时间戳的nflog_timestamp_t */
#define NFULA_IFINDEX_INDEV        4    /* 接收数据包的设备的ifindex（可能是桥接组） */
#define NFULA_IFINDEX_OUTDEV        5    /* 传输数据包的设备的ifindex（可能是桥接组） */
#define NFULA_IFINDEX_PHYSINDEV        6    /* 接收数据包的物理设备的ifindex（不是桥接组） */
#define NFULA_IFINDEX_PHYSOUTDEV    7    /* 传输数据包的物理设备的ifindex（不是桥接组） */
#define NFULA_HWADDR            8    /* 硬件地址的nflog_hwaddr_t */
#define NFULA_PAYLOAD            9    /* 数据包载荷 */
#define NFULA_PREFIX            10    /* 文本字符串 - 以空字符结尾，计数包括空字符 */
#define NFULA_UID            11    /* 发送/接收数据包的套接字的UID */
#define NFULA_SEQ            12    /* 此NFLOG套接字上数据包的序列号 */
#define NFULA_SEQ_GLOBAL        13    /* 所有NFLOG套接字上数据包的序列号 */
# 定义 NFULA_GID 常量，表示发送/接收数据包的套接字所属的 GID
#define NFULA_GID            14    /* GID owning socket on which packet was sent/received */
# 定义 NFULA_HWTYPE 常量，表示 skbuff 设备的 ARPHRD_ 类型
#define NFULA_HWTYPE            15    /* ARPHRD_ type of skbuff's device */
# 定义 NFULA_HWHEADER 常量，表示 skbuff 的 MAC 层头部
#define NFULA_HWHEADER            16    /* skbuff's MAC-layer header */
# 定义 NFULA_HWLEN 常量，表示 skbuff 的 MAC 层头部的长度
#define NFULA_HWLEN            17    /* length of skbuff's MAC-layer header */
# 结束条件编译指令
#endif
```