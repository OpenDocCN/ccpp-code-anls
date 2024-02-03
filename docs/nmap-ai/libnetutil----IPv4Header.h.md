# `nmap\libnetutil\IPv4Header.h`

```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef IPV4HEADER_H
#define IPV4HEADER_H 1

#include "NetworkLayerElement.h"

#define IP_RF 0x8000               /* Reserved fragment flag         */
#define IP_DF 0x4000               /* Don't fragment flag            */
#define IP_MF 0x2000               /* More fragments flag            */
#define IP_OFFMASK 0x1fff          /* Mask for fragmenting bits      */
#define IP_HEADER_LEN 20           /* Length of the standard header  */
#define MAX_IP_OPTIONS_LEN 40      /* Max Length for IP Options      */

/* Default header values */
#define IPv4_DEFAULT_TOS      0    // 默认服务类型字段值
#define IPv4_DEFAULT_ID       0    // 默认标识字段值
#define IPv4_DEFAULT_TTL      64   // 默认生存时间字段值
#define IPv4_DEFAULT_PROTO    6    /* TCP */  // 默认协议字段值为 TCP

class IPv4Header : public NetworkLayerElement {

}; /* End of class IPv4Header */

#endif
```