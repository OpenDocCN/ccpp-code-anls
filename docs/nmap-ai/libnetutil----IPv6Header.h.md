# `nmap\libnetutil\IPv6Header.h`

```cpp
/* 这段代码最初是 Nping 工具的一部分。 */

#ifndef IPV6HEADER_H
#define IPV6HEADER_H 1

#include "NetworkLayerElement.h"

#define IPv6_HEADER_LEN 40

/* 默认的头部数值 */
#define IPv6_DEFAULT_TCLASS    0
#define IPv6_DEFAULT_FLABEL    0
#define IPv6_DEFAULT_HOPLIM    64
#define IPv6_DEFAULT_NXTHDR    6 /* TCP */

class IPv6Header : public NetworkLayerElement {
    private:

  /*  IPv6 Header Format:
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |Version| Traffic Class |             Flow Label                |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |         Payload Length        |  Next Header  |   Hop Limit   |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                                                               |
        +--                                                           --+
        |                                                               |
        +--                      Source Address                       --+
        |                                                               |
        +--                                                           --+
        |                                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                                                               |
        +--                                                           --+
        |                                                               |
        +--                    Destination Address                    --+
        |                                                               |
        +--                                                           --+
        |                                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  */
    # 定义 IPv6 头部结构体
    struct nping_ipv6_hdr {
        u8  ip6_start[4];                /* 版本、流量和流标识 */
        u16 ip6_len;                     /* 负载长度 */
        u8  ip6_nh;                      /* 下一个头部 */
        u8  ip6_hopl;                    /* 跳数限制 */
        u8  ip6_src[16];                 /* 源 IP 地址 */
        u8  ip6_dst[16];                 /* 目的地 IP 地址 */
    }__attribute__((__packed__));       // 设置结构体紧凑排列
    
    typedef struct nping_ipv6_hdr nping_ipv6_hdr_t;  // 定义结构体类型
    
    nping_ipv6_hdr_t h;  // 创建 IPv6 头部结构体实例
        // 构造函数，初始化 IPv6 头部
        IPv6Header();

        // 析构函数，释放 IPv6 头部资源
        ~IPv6Header();

        // 重置 IPv6 头部数据
        void reset();

        // 获取 IPv6 头部数据的指针
        u8 *getBufferPointer();

        // 存储接收到的数据
        int storeRecvData(const u8 *buf, size_t len);

        // 获取协议 ID
        int protocol_id() const;

        // 验证 IPv6 头部数据的有效性
        int validate();

        // 打印 IPv6 头部数据
        int print(FILE *output, int detail) const;

        // 设置 IP 版本
        int setVersion();

        // 设置指定的 IP 版本
        int setVersion(u8 val);

        // 获取 IP 版本
        u8 getVersion() const;

        // 设置流量类别
        int setTrafficClass(u8 val);

        // 获取流量类别
        u8 getTrafficClass() const;

        // 设置流标签
        int setFlowLabel(u32 val);

        // 获取流标签
        u32 getFlowLabel() const;

        // 设置有效载荷长度
        int setPayloadLength(u16 val);

        // 设置有效载荷长度
        int setPayloadLength();

        // 获取有效载荷长度
        u16 getPayloadLength() const;

        // 设置下一个头部
        int setNextHeader(u8 val);

        // 设置下一个头部
        int setNextHeader(const char *p);

        // 获取下一个头部
        u8 getNextHeader() const;

        // 设置跳数限制
        int setHopLimit(u8 val);

        // 获取跳数限制
        u8 getHopLimit() const;

        // 设置源地址
        int setSourceAddress(u8 *val);

        // 设置源地址
        int setSourceAddress(struct in6_addr val);

        // 获取源地址
        const u8 *getSourceAddress() const;

        // 获取源地址
        struct in6_addr getSourceAddress(struct in6_addr *result) const;

        // 设置目标地址
        int setDestinationAddress(u8 *val);

        // 设置目标地址
        int setDestinationAddress(struct in6_addr val);

        // 获取目标地址
        const u8 *getDestinationAddress() const;

        // 获取目标地址
        struct in6_addr getDestinationAddress(struct in6_addr *result) const;

        // 获取地址长度
        u16 getAddressLength() const;
};
#endif



// 结束 C++ 的头文件宏定义
```