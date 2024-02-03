# `nmap\libnetutil\UDPHeader.h`

```cpp
/* This code was originally part of the Nping tool.                        */
// 定义了 UDPHeader 类，继承自 TransportLayerElement 类，用于处理 UDP 头部信息

#ifndef UDPHEADER_H
#define UDPHEADER_H 1

#include "TransportLayerElement.h"

#define UDP_HEADER_LEN 8
// UDP 头部长度为 8 个字节

/* Default header values */
#define UDP_DEFAULT_SPORT 53
#define UDP_DEFAULT_DPORT 53
// 默认的源端口和目的端口号分别为 53

class UDPHeader : public TransportLayerElement {

    private:
        /*
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |          Source Port          |       Destination Port        |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |            Length             |           Checksum            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        */
        // 定义了 UDP 头部结构体 nping_udp_hdr，包含源端口、目的端口、长度和校验和
        struct nping_udp_hdr{
          u16 uh_sport;
          u16 uh_dport;
          u16 uh_ulen;
          u16 uh_sum;
        }__attribute__((__packed__));
        // 使用 __attribute__((__packed__)) 属性确保结构体按照字节对齐方式进行打包

        typedef struct nping_udp_hdr nping_udp_hdr_t;

        nping_udp_hdr_t h;

    public:

        UDPHeader();
        ~UDPHeader();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);
        int protocol_id() const;
        int validate();
        int print(FILE *output, int detail) const;

        int setSourcePort(u16 p);
        u16 getSourcePort() const;

        int setDestinationPort(u16 p);
        u16 getDestinationPort() const;

        int setTotalLength();
        int setTotalLength(u16 l);
        u16 getTotalLength() const;

        int setSum(struct in_addr source, struct in_addr destination);
        int setSum(u16 s);
        int setSum();
        int setSumRandom();
        int setSumRandom(struct in_addr source, struct in_addr destination);
        u16 getSum() const;

}; /* End of class UDPHeader */


#endif
```