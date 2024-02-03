# `nmap\libpcap\msdos\pktdrvr.h`

```cpp
#ifndef __PKTDRVR_H
#define __PKTDRVR_H

#define PUBLIC        // 定义公共变量
#define LOCAL        static    // 定义静态变量

#define RX_BUF_SIZE  ETH_MTU   /* buffer size variables. NB !! */    // 接收缓冲区大小
#define TX_BUF_SIZE  ETH_MTU   /* must be same as in pkt_rx*.* */    // 发送缓冲区大小，必须与 pkt_rx*.* 中的相同

#ifdef __HIGHC__
#pragma Off(Align_members)    // 如果是高C编译器，则关闭成员对齐
#else
#pragma pack(1)    // 否则设置为1字节对齐
#endif

typedef enum  {                /* Packet-driver classes */    // 数据包驱动程序类别枚举
        PD_ETHER      = 1,    // 以太网
        PD_PRONET10   = 2,    // PRONET10
        PD_IEEE8025   = 3,    // IEEE8025
        PD_OMNINET    = 4,    // OMNINET
        PD_APPLETALK  = 5,    // APPLETALK
        PD_SLIP       = 6,    // SLIP
        PD_STARTLAN   = 7,    // STARTLAN
        PD_ARCNET     = 8,    // ARCNET
        PD_AX25       = 9,    // AX25
        PD_KISS       = 10,   // KISS
        PD_IEEE8023_2 = 11,   // IEEE8023_2
        PD_FDDI8022   = 12,   // FDDI8022
        PD_X25        = 13,   // X25
        PD_LANstar    = 14,   // LANstar
        PD_PPP        = 18    // PPP
      } PKT_CLASS;

typedef enum  {             /* Packet-driver receive modes    */    // 数据包驱动程序接收模式枚举
        PDRX_OFF    = 1,    /* turn off receiver              */    // 关闭接收器
        PDRX_DIRECT,        /* receive only to this interface */    // 仅接收到此接口
        PDRX_BROADCAST,     /* DIRECT + broadcast packets     */    // 直接 + 广播数据包
        PDRX_MULTICAST1,    /* BROADCAST + limited multicast  */    // 广播 + 有限组播
        PDRX_MULTICAST2,    /* BROADCAST + all multicast      */    // 广播 + 所有组播
        PDRX_ALL_PACKETS,   /* receive all packets on network */    // 接收网络上的所有数据包
      } PKT_RX_MODE;

typedef struct {
        char type[8];    // 类型
        char len;        // 长度
      } PKT_FRAME;
# 定义一个结构体，用于存储网络包的信息
typedef struct {
        BYTE  class;        /* 网络包类型，对于DEC/Interl/Xerox以太网为1 */
        BYTE  number;       /* 网络适配器编号，单个LAN适配器为0 */
        WORD  type;         /* 适配器类型，3C523为13 */
        BYTE  funcs;        /* 基本/扩展/高性能功能 */
        WORD  intr;         /* 用户中断向量号 */
        WORD  handle;       /* 与会话关联的句柄 */
        BYTE  name [15];    /* 适配器接口的名称，例如3C523 */
        BOOL  quiet;        /* 是否将错误打印到标准输出 */
        const char *error;  /* 错误字符串的地址 */
        BYTE  majVer;       /* 主驱动程序实现版本 */
        BYTE  minVer;       /* 次要驱动程序实现版本 */
        BYTE  dummyLen;     /* 以下数据的长度 */
        WORD  MAClength;    /* 高性能数据，不适用 */
        WORD  MTU;          /* 高性能数据，不适用 */
        WORD  multicast;    /* 高性能数据，不适用 */
        WORD  rcvrBuffers;  /* 有效的接收缓冲区 */
        WORD  UMTbufs;      /* 仅适用于高性能驱动程序 */
        WORD  postEOIintr;  /* 使用情况 ?? */
      } PKT_INFO;

# 定义一个宏，表示结构体的大小
#define PKT_PARAM_SIZE  14    /* 成员majVer - postEOIintr */

# 定义一个结构体，用于存储网络包的统计信息
typedef struct {
        DWORD inPackets;          /* 接收的数据包数量 */
        DWORD outPackets;         /* 发送的数据包数量 */
        DWORD inBytes;            /* 接收的字节数 */
        DWORD outBytes;           /* 发送的字节数 */
        DWORD inErrors;           /* 接收错误的数量 */
        DWORD outErrors;          /* 发送错误的数量 */
        DWORD lost;               /* 丢失的数据包数量（接收） */
      } PKT_STAT;

# 定义一个结构体，用于存储发送的网络包信息
typedef struct {
        ETHER destin;            /* 目的地MAC地址 */
        ETHER source;            /* 源MAC地址 */
        WORD  proto;             /* 协议类型 */
        BYTE  data [TX_BUF_SIZE];  /* 数据缓冲区 */
      } TX_ELEMENT;
typedef struct {
        WORD  firstCount;         /* # of bytes on 1st         */
        WORD  secondCount;        /* and 2nd upcall            */
        WORD  handle;             /* instance that upcalled    */
        ETHER destin;             /* E-net destination address */
        ETHER source;             /* E-net source address      */
        WORD  proto;              /* protocol number           */
        BYTE  data [RX_BUF_SIZE];
      } RX_ELEMENT;


#ifdef __HIGHC__
#pragma pop(Align_members)
#else
#pragma pack()
#endif


/*
 * Prototypes for publics
 */

#ifdef __cplusplus
extern "C" {
#endif

extern PKT_STAT    pktStat;     /* statistics for packets */
extern PKT_INFO    pktInfo;     /* packet-driver information */

extern PKT_RX_MODE receiveMode;
extern ETHER       myAddress, ethBroadcast;

extern BOOL  PktInitDriver (PKT_RX_MODE mode);
extern BOOL  PktExitDriver (void);

extern const char *PktGetErrorStr    (int errNum);
extern const char *PktGetClassName   (WORD class);
extern const char *PktRXmodeStr      (PKT_RX_MODE mode);
extern BOOL        PktSearchDriver   (void);
extern int         PktReceive        (BYTE *buf, int max);
extern BOOL        PktTransmit       (const void *eth, int len);
extern DWORD       PktRxDropped      (void);
extern BOOL        PktReleaseHandle  (WORD handle);
extern BOOL        PktTerminHandle   (WORD handle);
extern BOOL        PktResetInterface (WORD handle);
extern BOOL        PktSetReceiverMode(PKT_RX_MODE  mode);
extern BOOL        PktGetReceiverMode(PKT_RX_MODE *mode);
extern BOOL        PktGetStatistics  (WORD handle);
extern BOOL        PktSessStatistics (WORD handle);
extern BOOL        PktResetStatistics(WORD handle);
extern BOOL        PktGetAddress     (ETHER *addr);
extern BOOL        PktSetAddress     (const ETHER *addr);
extern BOOL        PktGetDriverInfo  (void);
extern BOOL        PktGetDriverParam (void);
extern void        PktQueueBusy      (BOOL busy);
extern WORD        PktBuffersUsed    (void);
#ifdef __cplusplus
}  // 结束 C++ 的 extern "C" 块
#endif

#endif /* __PKTDRVR_H */  // 结束条件编译，关闭头文件的 ifndef 指令
```