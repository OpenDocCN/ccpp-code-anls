# `nmap\libnetutil\RoutingHeader.h`

```cpp
/* This code was originally part of the Nping tool.                        */
// 这段代码最初是 Nping 工具的一部分

#ifndef __ROUTING_HEADER_H__
#define __ROUTING_HEADER_H__ 1
// 如果未定义 __ROUTING_HEADER_H__，则定义为 1

#include "IPv6ExtensionHeader.h"
// 包含名为 "IPv6ExtensionHeader.h" 的头文件

#define ROUTING_HEADER_MIN_LEN 8
#define ROUTING_HEADER_MAX_LEN (8 + 256*8)
#define ROUTING_MAX_DATA_LEN 256*8
#define ROUTING_TYPE_2_HEADER_LEN 24
#define ROUTING_TYPE_0_MIN_LEN 8
// 定义了一些常量

class RoutingHeader : public IPv6ExtensionHeader {
// 定义了一个名为 RoutingHeader 的类，继承自 IPv6ExtensionHeader 类

    public:
        RoutingHeader();
        ~RoutingHeader();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);
        int protocol_id() const;
        int validate();
        int print(FILE *output, int detail) const;
        // 声明了一些公共方法

        /* Protocol specific methods */
        int setNextHeader(u8 val);
        u8 getNextHeader();

        int setRoutingType(u8 val);
        u8 getRoutingType();

        int setSegmentsLeft(u8 val);
        u8 getSegmentsLeft();

        int addAddress(struct in6_addr val);
        // 声明了一些协议特定的方法

}; /* End of class RoutingHeader */
// 结束了类定义

#endif
// 结束了条件编译指令
```