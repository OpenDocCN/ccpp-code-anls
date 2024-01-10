# `nmap\libnetutil\ARPHeader.h`

```
/* 这段代码最初是 Nping 工具的一部分。 */

#ifndef __ARPHEADER_H__
#define __ARPHEADER_H__ 1

#include "NetworkLayerElement.h"

/* 长度 */
#define ARP_HEADER_LEN 28
#define IPv4_ADDRESS_LEN 4
#define ETH_ADDRESS_LEN  6

/* 硬件类型 */
#define HDR_RESERVED      0    /* [RFC5494]                                   */
#define HDR_ETH10MB       1    /* 以太网 (10Mb)                             */
#define HDR_ETH3MB        2    /* 实验性以太网 (3Mb)                         */
#define HDR_AX25          3    /* 业余无线电 AX.25                           */
#define HDR_PRONET_TR     4    /* Proteon ProNET 令牌环                       */
#define HDR_CHAOS         5    /* 混沌网络                                   */
#define HDR_IEEE802       6    /* IEEE 802 网络                             */
#define HDR_ARCNET        7    /* ARCNET [RFC1201]                            */
#define HDR_HYPERCHANNEL  8    /* 超通道                                    */
#define HDR_LANSTAR       9    /* Lanstar                                     */
#define HDR_AUTONET       10   /* Autonet 短地址                            */
#define HDR_LOCALTALK     11   /* LocalTalk                                   */
#define HDR_LOCALNET      12   /* LocalNet (IBM PCNet 或 SYTEK LocalNET)      */
#define HDR_ULTRALINK     13   /* 超链接                                    */
#define HDR_SMDS          14   /* SMDS                                        */
#define HDR_FRAMERELAY    15   /* 帧中继                                    */
#define HDR_ATM           16   /* 异步传输模式 (ATM)                        */
#define HDR_HDLC          17   /* HDLC                                        */
#define HDR_FIBRE         18   /* 光纤通道 [RFC4338]                         */
#define HDR_ATMb          19   /* 异步传输模式 (ATM)                        */
#define HDR_SERIAL        20   /* 串行线                                    */
# 定义各种协议类型的常量
#define HDR_ATMc          21   /* Asynchronous Transmission Mode [RFC2225]    */
#define HDR_MILSTD        22   /* MIL-STD-188-220                             */
#define HDR_METRICOM      23   /* Metricom                                    */
#define HDR_IEEE1394      24   /* IEEE 1394.199                               */
#define HDR_MAPOS         25   /* MAPOS [RFC2176]                             */
#define HDR_TWINAXIAL     26   /* Twinaxial                                   */
#define HDR_EUI64         27   /* EUI-64                                      */
#define HDR_HIPARP        28   /* HIPARP                                      */
#define HDR_ISO7816       29   /* IP and ARP over ISO 7816-3                  */
#define HDR_ARPSEC        30   /* ARPSec                                      */
#define HDR_IPSEC         31   /* IPsec tunnel                                */
#define HDR_INFINIBAND    32   /* InfiniBand (TM)                             */
#define HDR_TIA102        33   /* TIA-102 Project 25 Common Air Interface     */
#define HDR_WIEGAND       34   /* Wiegand Interface                           */
#define HDR_PUREIP        35   /* Pure IP                                     */
#define HDR_HW_EXP1       36   /* HW_EXP1 [RFC5494]                           */
#define HDR_HW_EXP2       37   /* HW_EXP2 [RFC5494]                           */

/* 定义各种操作码的常量 */
#define OP_ARP_REQUEST    1     /* ARP Request                                */
#define OP_ARP_REPLY      2     /* ARP Reply                                  */
#define OP_RARP_REQUEST   3     /* Reverse ARP Request                        */
#define OP_RARP_REPLY     4     /* Reverse ARP Reply                          */
#define OP_DRARP_REQUEST  5     /* DRARP-Request                              */
#define OP_DRARP_REPLY    6     /* DRARP-Reply                                */
#define OP_DRARP_ERROR    7     /* DRARP-Error                                */
# 定义 ARP 操作码的常量，对应不同的 ARP 操作
#define OP_INARP_REQUEST  8     /* InARP-Request                              */
#define OP_INARP_REPLY    9     /* InARP-Reply                                */
#define OP_ARPNAK         10    /* ARP-NAK                                    */
#define OP_MARS_REQUEST   11    /* MARS-Request                               */
#define OP_MARS_MULTI     12    /* MARS-Multi                                 */
#define OP_MARS_MSERV     13    /* MARS-MServ                                 */
#define OP_MARS_JOIN      14    /* MARS-Join                                  */
#define OP_MARS_LEAVE     15    /* MARS-Leave                                 */
#define OP_MARS_NAK       16    /* MARS-NAK                                   */
#define OP_MARS_UNSERV    17    /* MARS-Unserv                                */
#define OP_MARS_SJOIN     18    /* MARS-SJoin                                 */
#define OP_MARS_SLEAVE    19    /* MARS-SLeave                                */
#define OP_MARS_GL_REQ    20    /* MARS-Grouplist-Request                     */
#define OP_MARS_GL_REP    21    /* MARS-Grouplist-Reply                       */
#define OP_MARS_REDIR_MAP 22    /* MARS-Redirect-Map                          */
#define OP_MAPOS_UNARP    23    /* MAPOS-UNARP [RFC2176]                      */
#define OP_EXP1           24    /* OP_EXP1 [RFC5494]                          */
#define OP_EXP2           25    /* OP_EXP2 [RFC5494]                          */
#define OP_RESERVED       65535 /* Reserved [RFC5494]                         */

# 定义 ARPHeader 类，继承自 NetworkLayerElement 类
class ARPHeader : public NetworkLayerElement {
    private:

        // 定义一个结构体 nping_arp_hdr，用于表示 ARP 协议头部
        struct nping_arp_hdr{
            u16 ar_hrd;       /* Hardware Type.                               */  // 硬件类型
            u16 ar_pro;       /* Protocol Type.                               */  // 协议类型
            u8  ar_hln;       /* Hardware Address Length.                     */  // 硬件地址长度
            u8  ar_pln;       /* Protocol Address Length.                     */  // 协议地址长度
            u16 ar_op;        /* Operation Code.                              */  // 操作码
            u8 data[20];      // 数据

            // 由于对齐问题，无法使用下面的字段，这里注释掉
            //u8  ar_sha[6];    /* Sender Hardware Address.                     */
            //u32 ar_sip;       /* Sender Protocol Address (IPv4 address).      */
            //u8  ar_tha[6];    /* Target Hardware Address.                     */
            //u32 ar_tip;       /* Target Protocol Address (IPv4 address).      */
        }__attribute__((__packed__));  // 使用 __packed__ 属性进行紧凑排列

        // 定义一个类型别名 nping_arp_hdr_t，指向 nping_arp_hdr 结构体
        typedef struct nping_arp_hdr nping_arp_hdr_t;

        // 创建一个 nping_arp_hdr_t 类型的变量 h
        nping_arp_hdr_t h;
    public:

        /* Misc */
        // 构造函数，初始化 ARP 头部
        ARPHeader();
        // 析构函数，释放 ARP 头部资源
        ~ARPHeader();
        // 重置 ARP 头部
        void reset();
        // 获取缓冲区指针
        u8 *getBufferPointer();
        // 存储接收到的数据
        int storeRecvData(const u8 *buf, size_t len);
        // 获取协议 ID
        int protocol_id() const;
        // 验证 ARP 头部
        int validate();
        // 打印 ARP 头部信息
        int print(FILE *output, int detail) const;

        /* Hardware Type */
        // 设置硬件类型
        int setHardwareType(u16 t);
        // 获取硬件类型
        int setHardwareType();
        // 获取硬件类型
        u16 getHardwareType();

        /* Protocol Type */
        // 设置协议类型
        int setProtocolType(u16 t);
        // 获取协议类型
        int setProtocolType();
        // 获取协议类型
        u16 getProtocolType();

        /* Hardware Address Length */
        // 设置硬件地址长度
        int setHwAddrLen(u8 v);
        // 获取硬件地址长度
        int setHwAddrLen();
        // 获取硬件地址长度
        u8 getHwAddrLen();

        /* Protocol Address Length */
        // 设置协议地址长度
        int setProtoAddrLen(u8 v);
        // 获取协议地址长度
        int setProtoAddrLen();
        // 获取协议地址长度
        u8 getProtoAddrLen();

        /* Operation Code */
        // 设置操作码
        int setOpCode(u16 c);
        // 获取操作码
        u16 getOpCode();

        /* Sender Hardware Address */
        // 设置发送者硬件地址
        int setSenderMAC(const u8 *m);
        // 获取发送者硬件地址
        u8 *getSenderMAC();

        /* Sender Protocol address */
        // 设置发送者协议地址
        int setSenderIP(struct in_addr i);
        // 获取发送者协议地址
        u32 getSenderIP();

        /* Target Hardware Address */
        // 设置目标硬件地址
        int setTargetMAC(u8 *m);
        // 获取目标硬件地址
        u8 *getTargetMAC();

        /* Target Protocol Address */
        // 设置目标协议地址
        int setTargetIP(struct in_addr i);
        // 获取目标协议地址
        u32 getTargetIP();
}; /* End of class ARPHeader */

#endif /* __ARPHEADER_H__ */
```