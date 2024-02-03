# `nmap\libpcap\ethertype.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 条件：（1）源代码发布时，保留以上版权声明和本段文字；（2）包含二进制代码的发布，包括以上版权声明和本段文字在内的完整文档或其他提供的材料；（3）所有提及此软件特性或使用的广告材料都显示以下声明：“本产品包含由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

/*
 * 以太网类型。
 *
 * 我们使用 #ifdef 包裹声明，这样如果一个文件包含 <netinet/if_ether.h>，该文件可能会声明其中一些值，我们就不会因为这些值的重定义而收到 C 编译器的大量投诉。
 *
 * 我们在这里声明所有这些值，这样就不需要在任何文件中包含 <netinet/if_ether.h>，如果它所需要的只是 ETHERTYPE_ 值。
 */

#ifndef ETHERTYPE_PUP
#define ETHERTYPE_PUP        0x0200    /* PUP 协议 */
#endif
#ifndef ETHERTYPE_IP
#define ETHERTYPE_IP        0x0800    /* IP 协议 */
#endif
#ifndef ETHERTYPE_ARP
#define ETHERTYPE_ARP        0x0806    /* 地址解析协议 */
#endif
#ifndef ETHERTYPE_NS
#define ETHERTYPE_NS        0x0600
#endif
#ifndef    ETHERTYPE_SPRITE
#define ETHERTYPE_SPRITE    0x0500
#endif
#ifndef ETHERTYPE_TRAIL
#ifndef    ETHERTYPE_TRAIL
#define ETHERTYPE_TRAIL        0x1000
#endif
#ifndef    ETHERTYPE_MOPDL
#define ETHERTYPE_MOPDL        0x6001
#endif
#ifndef    ETHERTYPE_MOPRC
#define ETHERTYPE_MOPRC        0x6002
#endif
#ifndef    ETHERTYPE_DN
#define ETHERTYPE_DN        0x6003
#endif
#ifndef    ETHERTYPE_LAT
#define ETHERTYPE_LAT        0x6004
#endif
#ifndef ETHERTYPE_SCA
#define ETHERTYPE_SCA        0x6007
#endif
#ifndef ETHERTYPE_TEB
#define ETHERTYPE_TEB        0x6558
#endif
#ifndef ETHERTYPE_REVARP
#define ETHERTYPE_REVARP    0x8035    /* reverse Addr. resolution protocol */
#endif
#ifndef    ETHERTYPE_LANBRIDGE
#define ETHERTYPE_LANBRIDGE    0x8038
#endif
#ifndef    ETHERTYPE_DECDNS
#define ETHERTYPE_DECDNS    0x803c
#endif
#ifndef    ETHERTYPE_DECDTS
#define ETHERTYPE_DECDTS    0x803e
#endif
#ifndef    ETHERTYPE_VEXP
#define ETHERTYPE_VEXP        0x805b
#endif
#ifndef    ETHERTYPE_VPROD
#define ETHERTYPE_VPROD        0x805c
#endif
#ifndef ETHERTYPE_ATALK
#define ETHERTYPE_ATALK        0x809b
#endif
#ifndef ETHERTYPE_AARP
#define ETHERTYPE_AARP        0x80f3
#endif
#ifndef ETHERTYPE_8021Q
#define ETHERTYPE_8021Q        0x8100
#endif
#ifndef ETHERTYPE_IPX
#define ETHERTYPE_IPX        0x8137
#endif
#ifndef ETHERTYPE_IPV6
#define ETHERTYPE_IPV6        0x86dd
#endif
#ifndef ETHERTYPE_MPLS
#define ETHERTYPE_MPLS        0x8847
#endif
#ifndef ETHERTYPE_MPLS_MULTI
#define ETHERTYPE_MPLS_MULTI    0x8848
#endif
#ifndef ETHERTYPE_PPPOED
#define ETHERTYPE_PPPOED    0x8863
#endif
#ifndef ETHERTYPE_PPPOES
#define ETHERTYPE_PPPOES    0x8864
#endif
#ifndef ETHERTYPE_8021AD
#define ETHERTYPE_8021AD    0x88a8
#endif
#ifndef    ETHERTYPE_LOOPBACK
#define ETHERTYPE_LOOPBACK    0x9000
#endif
#ifndef ETHERTYPE_8021QINQ
#define ETHERTYPE_8021QINQ    0x9100
#endif
```