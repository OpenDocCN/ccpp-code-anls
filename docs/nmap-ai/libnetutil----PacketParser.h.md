# `nmap\libnetutil\PacketParser.h`

```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef __PACKETPARSER_H__
#define __PACKETPARSER_H__ 1

#include "ApplicationLayerElement.h"
#include "ARPHeader.h"
#include "DataLinkLayerElement.h"
#include "EthernetHeader.h"
#include "ICMPHeader.h"
#include "ICMPv4Header.h"
#include "ICMPv6Header.h"
#include "ICMPv6Option.h"
#include "ICMPv6RRBody.h"
#include "IPv4Header.h"
#include "IPv6Header.h"
#include "NetworkLayerElement.h"
#include "PacketElement.h"
#include "RawData.h"
#include "TCPHeader.h"
#include "TransportLayerElement.h"
#include "UDPHeader.h"
#include "HopByHopHeader.h"
#include "DestOptsHeader.h"
#include "FragmentHeader.h"
#include "RoutingHeader.h"


#define LINK_LAYER         2
#define NETWORK_LAYER      3
#define TRANSPORT_LAYER    4
#define APPLICATION_LAYER  5
#define EXTHEADERS_LAYER   6

typedef struct header_type_string{
    u32 type;
    const char *str;
}header_type_string_t;


typedef struct packet_type{
    u32 type;
    u32 length;
}pkt_type_t;


class PacketParser {

    private:

    public:

    /* Misc */
    // 构造函数
    PacketParser();
    // 析构函数
    ~PacketParser();
    // 重置函数
    void reset();

    // 将整数类型的头部类型转换为字符串类型
    static const char *header_type2string(int val);
    // 解析数据包
    static pkt_type_t *parse_packet(const u8 *pkt, size_t pktlen, bool eth_included);
    // 打印数据包类型的虚拟函数，待移除
    static int dummy_print_packet_type(const u8 *pkt, size_t pktlen, bool eth_included); /* TODO: remove */
    // 打印数据包的虚拟函数，待移除
    static int dummy_print_packet(const u8 *pkt, size_t pktlen, bool eth_included); /* TODO: remove */
    // 获取有效载荷的偏移量
    static int payload_offset(const u8 *pkt, size_t pktlen, bool link_included);
    // 拆分数据包
    static PacketElement *split(const u8 *pkt, size_t pktlen, bool eth_included);
    // 拆分数据包
    static PacketElement *split(const u8 *pkt, size_t pktlen);
    // 释放数据包链
    static int freePacketChain(PacketElement *first);
    // 测试数据包解析
    static const char *test_packet_parser(PacketElement *test_pkt);
    // 判断是否为响应数据包
    static bool is_response(PacketElement *sent, PacketElement *rcvd);
    // 查找传输层数据包
    static PacketElement *find_transport_layer(PacketElement *chain);
}; /* End of class PacketParser */
#endif /* __PACKETPARSER_H__ */

这部分代码是C++中的类定义结束和头文件宏定义的结束。
```