# `nmap\libnetutil\EthernetHeader.h`

```
/* This code was originally part of the Nping tool.                        */
// 这段代码最初是 Nping 工具的一部分

#ifndef ETHERNETHEADER_H
#define ETHERNETHEADER_H 1
// 如果未定义 ETHERNETHEADER_H，则定义为 1

#include "DataLinkLayerElement.h"
// 包含 DataLinkLayerElement.h 文件

/* Ether Types. (From RFC 5342 http://www.rfc-editor.org/rfc/rfc5342.txt)     */
// 以太网类型（来自 RFC 5342 http://www.rfc-editor.org/rfc/rfc5342.txt）
#define ETHTYPE_IPV4       0x0800 /* Internet Protocol Version 4              */
#define ETHTYPE_ARP        0x0806 /* Address Resolution Protocol              */
#define ETHTYPE_FRAMERELAY 0x0808 /* Frame Relay ARP                          */
#define ETHTYPE_PPTP       0x880B /* Point-to-Point Tunneling Protocol        */
#define ETHTYPE_GSMP       0x880C /* General Switch Management Protocol       */
#define ETHTYPE_RARP       0x8035 /* Reverse Address Resolution Protocol      */
#define ETHTYPE_IPV6       0x86DD /* Internet Protocol Version 6              */
#define ETHTYPE_MPLS       0x8847 /* MPLS                                     */
#define ETHTYPE_MPS_UAL    0x8848 /* MPLS with upstream-assigned label        */
#define ETHTYPE_MCAP       0x8861 /* Multicast Channel Allocation Protocol    */
#define ETHTYPE_PPPOE_D    0x8863 /* PPP over Ethernet Discovery Stage        */
#define ETHTYPE_PPOE_S     0x8864 /* PPP over Ethernet Session Stage          */
#define ETHTYPE_CTAG       0x8100 /* Customer VLAN Tag Type                   */
#define ETHTYPE_EPON       0x8808 /* Ethernet Passive Optical Network         */
#define ETHTYPE_PBNAC      0x888E /* Port-based network access control        */
#define ETHTYPE_STAG       0x88A8 /* Service VLAN tag identifier              */
#define ETHTYPE_ETHEXP1    0x88B5 /* Local Experimental Ethertype             */
#define ETHTYPE_ETHEXP2    0x88B6 /* Local Experimental Ethertype             */
#define ETHTYPE_ETHOUI     0x88B7 /* OUI Extended Ethertype                   */
#define ETHTYPE_PREAUTH    0x88C7 /* Pre-Authentication                       */
#define ETHTYPE_LLDP       0x88CC /* Link Layer Discovery Protocol (LLDP)     */
// 定义以太网协议类型的常量值
#define ETHTYPE_MACSEC     0x88E5 /* Media Access Control Security            */
#define ETHTYPE_MVRP       0x88F5 /* Multiple VLAN Registration Protocol      */
#define ETHTYPE_MMRP       0x88F6 /* Multiple Multicast Registration Protocol */
#define ETHTYPE_FRRR       0x890D /* Fast Roaming Remote Request              */

// 定义以太网头部长度的常量值
#define ETH_HEADER_LEN 14

// 定义以太网头部类，继承自数据链路层元素类
class EthernetHeader : public DataLinkLayerElement {

    private:

        // 定义以太网头部结构体
        struct nping_eth_hdr{
            u8 eth_dmac[6]; // 目的 MAC 地址
            u8 eth_smac[6]; // 源 MAC 地址
            u16 eth_type;   // 以太网类型
        }__attribute__((__packed__)); // 设置为紧凑模式，避免结构体填充

        typedef struct nping_eth_hdr nping_eth_hdr_t; // 定义以太网头部结构体类型

        nping_eth_hdr_t h; // 以太网头部结构体实例

    public:

        EthernetHeader(); // 构造函数
        ~EthernetHeader(); // 析构函数
        void reset(); // 重置以太网头部
        u8 *getBufferPointer(); // 获取缓冲区指针
        int storeRecvData(const u8 *buf, size_t len); // 存储接收到的数据
        int protocol_id() const; // 获取协议 ID
        int validate(); // 验证以太网头部
        int print(FILE *output, int detail) const; // 打印以太网头部信息

        int setSrcMAC(const u8 *m); // 设置源 MAC 地址
        const u8 *getSrcMAC() const; // 获取源 MAC 地址

        int setDstMAC(u8 *m); // 设置目的 MAC 地址
        const u8 *getDstMAC() const; // 获取目的 MAC 地址

        int setEtherType(u16 val); // 设置以太网类型
        u16 getEtherType() const; // 获取以太网类型

};

#endif
```