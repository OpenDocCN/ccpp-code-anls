# `nmap\libdnet-stripped\include\dnet\eth.h`

```
#ifndef DNET_ETH_H
#define DNET_ETH_H

#define ETH_ADDR_LEN    6    // 以太网地址长度
#define ETH_ADDR_BITS    48    // 以太网地址位数
#define ETH_TYPE_LEN    2    // 以太网类型长度
#define ETH_CRC_LEN    4    // 以太网 CRC 长度
#define ETH_HDR_LEN    14    // 以太网头部长度

#define ETH_LEN_MIN    64        // 带 CRC 的最小帧长度
#define ETH_LEN_MAX    1518        // 带 CRC 的最大帧长度

#define ETH_MTU        (ETH_LEN_MAX - ETH_HDR_LEN - ETH_CRC_LEN)    // 以太网最大传输单元
#define ETH_MIN        (ETH_LEN_MIN - ETH_HDR_LEN - ETH_CRC_LEN)    // 以太网最小传输单元

typedef struct eth_addr {
    uint8_t        data[ETH_ADDR_LEN];    // 以太网地址结构体
} eth_addr_t;

struct eth_hdr {
    eth_addr_t    eth_dst;    /* destination address */    // 目的地址
    eth_addr_t    eth_src;    /* source address */    // 源地址
    uint16_t    eth_type;    /* payload type */    // 负载类型
};

/*
 * 以太网负载类型 - http://standards.ieee.org/regauth/ethertype
 */
#define ETH_TYPE_PUP    0x0200        // PUP 协议
#define ETH_TYPE_IP    0x0800        // IP 协议
#define ETH_TYPE_ARP    0x0806        // 地址解析协议
#define ETH_TYPE_REVARP    0x8035        // 反向地址解析协议
#define ETH_TYPE_8021Q    0x8100        // IEEE 802.1Q VLAN 标记
#define ETH_TYPE_IPV6    0x86DD        // IPv6 协议
#define ETH_TYPE_MPLS    0x8847        // MPLS
#define ETH_TYPE_MPLS_MCAST    0x8848    // MPLS 多播
#define ETH_TYPE_PPPOEDISC    0x8863    // 以太网发现阶段的 PPP Over Ethernet
#define ETH_TYPE_PPPOE    0x8864        // 以太网会话阶段的 PPP Over Ethernet
#define ETH_TYPE_LOOPBACK    0x9000    // 用于测试接口的回环

#define ETH_IS_MULTICAST(ea)    (*(ea) & 0x01) /* is address mcast/bcast? */    // 判断地址是否为多播/广播地址

#define ETH_ADDR_BROADCAST    "\xff\xff\xff\xff\xff\xff"    // 广播地址

#define eth_pack_hdr(h, dst, src, type) do {            \
    struct eth_hdr *eth_pack_p = (struct eth_hdr *)(h);    // 以太网头部指针
    memmove(&eth_pack_p->eth_dst, &(dst), ETH_ADDR_LEN);    // 将目的地址复制到以太网头部
    # 从源地址的内存位置复制数据到目标地址的内存位置
    memmove(&eth_pack_p->eth_src, &(src), ETH_ADDR_LEN);    \
    # 设置以太网帧的类型字段，并将主机字节顺序转换为网络字节顺序
    eth_pack_p->eth_type = htons(type);            \
} while (0)  // do-while 循环的结束标记

typedef struct eth_handle eth_t;  // 定义了一个名为 eth_t 的结构体类型 eth_handle

__BEGIN_DECLS  // 开始声明 C 语言函数

eth_t    *eth_open(const char *device);  // 打开一个以太网设备，并返回一个指向 eth_t 结构体的指针
int     eth_get(eth_t *e, eth_addr_t *ea);  // 获取以太网设备的地址
int     eth_set(eth_t *e, const eth_addr_t *ea);  // 设置以太网设备的地址
ssize_t     eth_send(eth_t *e, const void *buf, size_t len);  // 发送数据到以太网设备
eth_t    *eth_close(eth_t *e);  // 关闭以太网设备

int     eth_get_pcap_devname(const char *ifname, char *pcapdev, int pcapdevlen);  // 获取 pcap 设备的名称

char    *eth_ntop(const eth_addr_t *eth, char *dst, size_t len);  // 将以太网地址转换为可读的字符串形式
int     eth_pton(const char *src, eth_addr_t *dst);  // 将字符串形式的以太网地址转换为二进制形式
char    *eth_ntoa(const eth_addr_t *eth);  // 将以太网地址转换为点分十进制形式的字符串
#define     eth_aton eth_pton  // 定义 eth_aton 宏，与 eth_pton 函数功能相同
__END_DECLS  // 结束声明 C 语言函数

#endif /* DNET_ETH_H */  // 结束 eth.h 文件的条件编译
```