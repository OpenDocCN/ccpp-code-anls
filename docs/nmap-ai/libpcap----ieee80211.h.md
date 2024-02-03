# `nmap\libpcap\ieee80211.h`

```cpp
# 版权声明和许可条款
# 作者保留所有权利
# 允许以源代码或二进制形式重新分发和使用，需满足以下条件：
# 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
# 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
# 3. 未经特定事先书面许可，不得使用作者的名字来认可或推广基于此软件的产品
# 或者，可以根据自由软件基金会发布的 GNU 通用公共许可证（“GPL”）第2版的条款分发此软件
# 作者提供此软件“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
# 无论在任何情况下，作者均不对任何直接、间接、附带、特殊、示范性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害
# FreeBSD: src/sys/net80211/ieee80211.h,v 1.10 2005/07/22 16:55:27 sam Exp
# 如果条件满足，则定义 _NET80211_IEEE80211_H_
#ifndef _NET80211_IEEE80211_H_
#define _NET80211_IEEE80211_H_

# 802.11 协议定义
# 定义 IEEE80211_FC0_VERSION_MASK 为 0x03
# 定义 IEEE80211_FC0_VERSION_SHIFT 为 0
#define    IEEE80211_FC0_VERSION_MASK        0x03
#define    IEEE80211_FC0_VERSION_SHIFT        0
# 定义 IEEE80211_FC0_VERSION_0 值为 0x00
#define    IEEE80211_FC0_VERSION_0            0x00
# 定义 IEEE80211_FC0_TYPE_MASK 值为 0x0c
#define    IEEE80211_FC0_TYPE_MASK            0x0c
# 定义 IEEE80211_FC0_TYPE_SHIFT 值为 2
#define    IEEE80211_FC0_TYPE_SHIFT        2
# 定义 IEEE80211_FC0_TYPE_MGT 值为 0x00
#define    IEEE80211_FC0_TYPE_MGT            0x00
# 定义 IEEE80211_FC0_TYPE_CTL 值为 0x04
#define    IEEE80211_FC0_TYPE_CTL            0x04
# 定义 IEEE80211_FC0_TYPE_DATA 值为 0x08
#define    IEEE80211_FC0_TYPE_DATA            0x08

# 定义 IEEE80211_FC0_SUBTYPE_MASK 值为 0xf0
#define    IEEE80211_FC0_SUBTYPE_MASK        0xf0
# 定义 IEEE80211_FC0_SUBTYPE_SHIFT 值为 4
#define    IEEE80211_FC0_SUBTYPE_SHIFT        4
# 定义各种管理帧的子类型
#define    IEEE80211_FC0_SUBTYPE_ASSOC_REQ        0x00
#define    IEEE80211_FC0_SUBTYPE_ASSOC_RESP    0x10
#define    IEEE80211_FC0_SUBTYPE_REASSOC_REQ    0x20
#define    IEEE80211_FC0_SUBTYPE_REASSOC_RESP    0x30
#define    IEEE80211_FC0_SUBTYPE_PROBE_REQ        0x40
#define    IEEE80211_FC0_SUBTYPE_PROBE_RESP    0x50
#define    IEEE80211_FC0_SUBTYPE_BEACON        0x80
#define    IEEE80211_FC0_SUBTYPE_ATIM        0x90
#define    IEEE80211_FC0_SUBTYPE_DISASSOC        0xa0
#define    IEEE80211_FC0_SUBTYPE_AUTH        0xb0
#define    IEEE80211_FC0_SUBTYPE_DEAUTH        0xc0
# 定义各种控制帧的子类型
#define    IEEE80211_FC0_SUBTYPE_PS_POLL        0xa0
#define    IEEE80211_FC0_SUBTYPE_RTS        0xb0
#define    IEEE80211_FC0_SUBTYPE_CTS        0xc0
#define    IEEE80211_FC0_SUBTYPE_ACK        0xd0
#define    IEEE80211_FC0_SUBTYPE_CF_END        0xe0
#define    IEEE80211_FC0_SUBTYPE_CF_END_ACK    0xf0
# 定义数据帧的子类型
#define    IEEE80211_FC0_SUBTYPE_DATA        0x00
#define    IEEE80211_FC0_SUBTYPE_CF_ACK        0x10
#define    IEEE80211_FC0_SUBTYPE_CF_POLL        0x20
#define    IEEE80211_FC0_SUBTYPE_CF_ACPL        0x30
#define    IEEE80211_FC0_SUBTYPE_NODATA        0x40
#define    IEEE80211_FC0_SUBTYPE_NODATA_CF_ACK    0x50
#define    IEEE80211_FC0_SUBTYPE_NODATA_CF_POLL    0x60
#define    IEEE80211_FC0_SUBTYPE_NODATA_CF_ACPL    0x70
#define    IEEE80211_FC0_SUBTYPE_QOS        0x80
#define    IEEE80211_FC0_SUBTYPE_QOS_NULL        0xc0

# 定义 IEEE80211_FC1_DIR_MASK 值为 0x03
#define    IEEE80211_FC1_DIR_MASK            0x03
# 定义 IEEE80211_FC1_DIR_NODS 常量，表示 STA 到 STA 的数据传输方向
#define    IEEE80211_FC1_DIR_NODS            0x00    /* STA->STA */
# 定义 IEEE80211_FC1_DIR_TODS 常量，表示 STA 到 AP 的数据传输方向
#define    IEEE80211_FC1_DIR_TODS            0x01    /* STA->AP  */
# 定义 IEEE80211_FC1_DIR_FROMDS 常量，表示 AP 到 STA 的数据传输方向
#define    IEEE80211_FC1_DIR_FROMDS        0x02    /* AP ->STA */
# 定义 IEEE80211_FC1_DIR_DSTODS 常量，表示 AP 到 AP 的数据传输方向
#define    IEEE80211_FC1_DIR_DSTODS        0x03    /* AP ->AP  */

# 定义 IEEE80211_FC1_MORE_FRAG 常量，表示数据帧是否有更多的分片
#define    IEEE80211_FC1_MORE_FRAG            0x04
# 定义 IEEE80211_FC1_RETRY 常量，表示数据帧是否为重传帧
#define    IEEE80211_FC1_RETRY            0x08
# 定义 IEEE80211_FC1_PWR_MGT 常量，表示数据帧是否启用了电源管理
#define    IEEE80211_FC1_PWR_MGT            0x10
# 定义 IEEE80211_FC1_MORE_DATA 常量，表示数据帧是否有更多数据
#define    IEEE80211_FC1_MORE_DATA            0x20
# 定义 IEEE80211_FC1_WEP 常量，表示数据帧是否启用了 WEP 加密
#define    IEEE80211_FC1_WEP            0x40
# 定义 IEEE80211_FC1_ORDER 常量，表示数据帧是否启用了帧顺序控制
#define    IEEE80211_FC1_ORDER            0x80

# 定义 IEEE80211_SEQ_FRAG_MASK 常量，表示数据帧序列号的分片掩码
#define    IEEE80211_SEQ_FRAG_MASK            0x000f
# 定义 IEEE80211_SEQ_FRAG_SHIFT 常量，表示数据帧序列号的分片位移
#define    IEEE80211_SEQ_FRAG_SHIFT        0
# 定义 IEEE80211_SEQ_SEQ_MASK 常量，表示数据帧序列号的序列掩码
#define    IEEE80211_SEQ_SEQ_MASK            0xfff0
# 定义 IEEE80211_SEQ_SEQ_SHIFT 常量，表示数据帧序列号的序列位移
#define    IEEE80211_SEQ_SEQ_SHIFT            4

# 定义 IEEE80211_NWID_LEN 常量，表示 IEEE 802.11 网络标识符的长度
#define    IEEE80211_NWID_LEN            32

# 定义 IEEE80211_QOS_TXOP 常量，表示 QoS 数据帧的传输机会掩码
#define    IEEE80211_QOS_TXOP            0x00ff
# 定义 IEEE80211_QOS_ACKPOLICY 常量，表示 QoS 数据帧的 ACK 策略掩码
#define    IEEE80211_QOS_ACKPOLICY            0x60
# 定义 IEEE80211_QOS_ACKPOLICY_S 常量，表示 QoS 数据帧的 ACK 策略位移
#define    IEEE80211_QOS_ACKPOLICY_S        5
# 定义 IEEE80211_QOS_ESOP 常量，表示 QoS 数据帧的 ESOP 位
#define    IEEE80211_QOS_ESOP            0x10
# 定义 IEEE80211_QOS_ESOP_S 常量，表示 QoS 数据帧的 ESOP 位移
#define    IEEE80211_QOS_ESOP_S            4
# 定义 IEEE80211_QOS_TID 常量，表示 QoS 数据帧的 TID 位
#define    IEEE80211_QOS_TID            0x0f

# 定义 IEEE80211_MGT_SUBTYPE_NAMES 常量，表示管理帧子类型的名称数组
#define IEEE80211_MGT_SUBTYPE_NAMES {            \
    "assoc-req",        "assoc-resp",        \
    "reassoc-req",        "reassoc-resp",        \
    "probe-req",        "probe-resp",        \
    "reserved#6",        "reserved#7",        \
    "beacon",        "atim",            \
    "disassoc",        "auth",            \
    "deauth",        "reserved#13",        \
    "reserved#14",        "reserved#15"        \
}

# 定义 IEEE80211_CTL_SUBTYPE_NAMES 常量，表示控制帧子类型的名称数组
#define IEEE80211_CTL_SUBTYPE_NAMES {            \
    "reserved#0",        "reserved#1",        \
    "reserved#2",        "reserved#3",        \
    "reserved#3",        "reserved#5",        \
    "reserved#6",        "reserved#7",        \
    "reserved#8",        "reserved#9",        \
    "ps-poll",        "rts",            \
    "cts",            "ack",            \
    "cf-end",        "cf-end-ack"        \
}
# 定义 IEEE80211 数据子类型的名称列表
#define IEEE80211_DATA_SUBTYPE_NAMES {            \
    "data",            "data-cf-ack",        \
    "data-cf-poll",        "data-cf-ack-poll",    \
    "null",            "cf-ack",        \
    "cf-poll",        "cf-ack-poll",        \
    "qos-data",        "qos-data-cf-ack",    \
    "qos-data-cf-poll",    "qos-data-cf-ack-poll",    \
    "qos",            "reserved#13",        \
    "qos-cf-poll",        "qos-cf-ack-poll"    \
}

# 定义 IEEE80211 类型的名称列表
#define IEEE80211_TYPE_NAMES    { "mgt", "ctl", "data", "reserved#4" }

# 结束条件编译指令
#endif /* _NET80211_IEEE80211_H_ */
```