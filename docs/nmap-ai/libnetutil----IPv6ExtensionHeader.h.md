# `nmap\libnetutil\IPv6ExtensionHeader.h`

```
/* This code was originally part of the Nping tool.                        */
// 定义了 IPv6 扩展头部的选项代码

#ifndef __IPv6_EXTENSION_HEADER_H__
#define __IPv6_EXTENSION_HEADER_H__ 1

#include "PacketElement.h"

/* Extension header option codes */
// 定义了扩展头部选项的代码
#define EXTOPT_PAD1        0x00   /* Pad1 (RFC 2460)                          */
#define EXTOPT_PADN        0x01   /* PadN (RFC 2460)                          */
#define EXTOPT_JUMBO       0xC2   /* Jumbo Payload (RFC 2675)                 */
#define EXTOPT_TUNENCAPLIM 0x04   /* Tunnel Encapsulation Limit (RFC 2473)    */
#define EXTOPT_ROUTERALERT 0x05   /* Router Alert (RFC 2711)                  */
#define EXTOPT_QUICKSTART  0x26   /* Quick-Start (RFC 4782)                   */
#define EXTOPT_CALIPSO     0x07   /* CALIPSO (RFC 5570)                       */
#define EXTOPT_HOMEADDR    0xC9   /* Home Address (RFC 6275)                  */

class IPv6ExtensionHeader : public PacketElement {

};

#endif
```