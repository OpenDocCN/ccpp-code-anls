# `nmap\libnetutil\ICMPv6RRBody.h`

```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef ICMPv6RRBODY_H
#define ICMPv6RRBODY_H 1

#include "NetworkLayerElement.h"

/* Packet header diagrams included in this file have been taken from the
 * following IETF RFC documents: RFC 2894 */

/* Nping ICMPv6RRBody Class internal definitions */
#define ICMPv6_RR_MATCH_PREFIX_LEN 24
#define ICMPv6_RR_USE_PREFIX_LEN   32
#define ICMPv6_RR_RESULT_MSG_LEN   24
/* This must the MAX() of all values defined above*/
#define ICMPv6_RR_MAX_LENGTH (ICMPv6_RR_USE_PREFIX_LEN)
#define ICMPv6_RR_MIN_LENGTH (ICMPv6_RR_MATCH_PREFIX_LEN)


class ICMPv6RRBody : public NetworkLayerElement {

    public:
        // 构造函数
        ICMPv6RRBody();
        // 析构函数
        ~ICMPv6RRBody();
        // 重置函数
        void reset();
        // 获取缓冲区指针函数
        u8 *getBufferPointer();
        // 存储接收数据函数
        int storeRecvData(const u8 *buf, size_t len);

}; /* End of class ICMPv6RRBody */

#endif
```