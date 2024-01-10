# `nmap\nping\utils_net.h`

```
// 包含 NpingTarget.h 头文件
#include "NpingTarget.h"

// 如果未定义 UTILS_NET_H，则定义为 1
#ifndef UTILS_NET_H
#define UTILS_NET_H 1

// 如果未定义 NETINET_IN_SYSTM_H，则包含 netinet/in_systm.h 头文件，并定义 NETINET_IN_SYSTM_H 为 1
#ifndef NETINET_IN_SYSTM_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/in_systm.h> /* defines n_long needed for netinet/ip.h */
#define NETINET_IN_SYSTM_H
#endif

// 如果未定义 NETINET_IP_H，则包含 netinet/ip.h 头文件，并定义 NETINET_IP_H 为 1
#ifndef NETINET_IP_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/ip.h>
#define NETINET_IP_H
#endif

// 函数声明
int atoIP(const char *hostname, struct in_addr *dst);
int atoIP(const char *hostname, struct sockaddr_storage *ss, int family);
char *IPtoa(u32 i);
char *IPtoa(struct sockaddr_storage *ss);
char *IPtoa(struct in_addr addr);
char *IPtoa(struct in6_addr addr);
char *IPtoa(struct sockaddr_storage ss);
char *IPtoa(struct sockaddr_storage *ss, int family);
char *IPtoa(u8 *ipv6addr);
u16 sockaddr2port(struct sockaddr_storage *ss);
u16 sockaddr2port(struct sockaddr_storage ss);
u16 sockaddr2port(struct sockaddr_in *s4);
u16 sockaddr2port(struct sockaddr_in6 *s6);
int setsockaddrfamily(struct sockaddr_storage *ss, int family);
int setsockaddrany(struct sockaddr_storage *ss);
bool isICMPType(u8 type);
bool isICMPCode(u8 code);
bool isICMPCode(u8 code, u8 type);
int getPacketStrInfo(const char *proto, const u8 *packet, u32 len, u8 *dstbuff, u32 dstlen, struct sockaddr_storage *ss_src, struct sockaddr_storage *ss_dst);
int getPacketStrInfo(const char *proto, const u8 *packet, u32 len, u8 *dstbuff, u32 dstlen);
int getNetworkInterfaceName(u32 destination, char *dev);
int getNetworkInterfaceName(struct sockaddr_storage *dst, char *dev);
int nping_getpts_simple(const char *origexpr, u16 **list, int *count);
int resolveCached(char *host, struct sockaddr_storage *ss, size_t *sslen,int pf) ;
struct hostent *gethostbynameCached(char *host);
struct hostent *hostentcpy(struct hostent *src);
int hostentfree(struct hostent *src);
int parseMAC(const char *txt, u8 *targetbuff);
char *MACtoa(u8 *mac);
const char *arppackethdrinfo(const u8 *packet, u32 len, int detail );
# 获取 ARP 数据包头部信息
int arppackethdrinfo(const u8 *packet, u32 len, u8 *dstbuff, u32 dstlen);

# 获取 TCP 数据包头部信息
int tcppackethdrinfo(const u8 *packet, size_t len, u8 *dstbuff, size_t dstlen, int detail, struct sockaddr_storage *src, struct sockaddr_storage *dst);

# 获取 UDP 数据包头部信息
int udppackethdrinfo(const u8 *packet, size_t len, u8 *dstbuff, size_t dstlen, int detail, struct sockaddr_storage *src, struct sockaddr_storage *dst);

# 获取随机文本负载
const char *getRandomTextPayload();

# 发送数据包
int send_packet(NpingTarget *target, int rawfd, u8 *pkt, size_t pktLen);

# 打印网络接口信息
int print_dnet_interface(const struct intf_entry *entry, void *arg);

# 打印所有网络接口信息
int print_interfaces_dnet();

# 从 IP 数据包中获取源地址信息
struct sockaddr_storage *getSrcSockAddrFromIPPacket(u8 *pkt, size_t pktLen);

# 获取 UDP 头部位置
u8 *getUDPheaderLocation(u8 *pkt, size_t pktLen);

# 获取 TCP 头部位置
u8 *getTCPheaderLocation(u8 *pkt, size_t pktLen);

# 从 IP 数据包中获取协议类型
u8 getProtoFromIPPacket(u8 *pkt, size_t pktLen);

# 从 IP 数据包中获取源端口
u16 *getSrcPortFromIPPacket(u8 *pkt, size_t pktLen);

# 从 IP 数据包中获取目标端口
u16 *getDstPortFromIPPacket(u8 *pkt, size_t pktLen);

# 从 TCP 头部中获取目标端口
u16 *getDstPortFromTCPHeader(u8 *pkt, size_t pktLen);

# 从 UDP 头部中获取目标端口
u16 *getDstPortFromUDPHeader(u8 *pkt, size_t pktLen);

# 获取原始套接字
int obtainRawSocket();

# 定义设备名称长度
#define DEVNAMELEN 16

# 定义 IPv6 接口信息文件路径
#define PATH_PROC_IFINET6 "/proc/net/if_inet6"

# 定义 IPv6 接口结构体
typedef struct ipv6_interface{
  char devname[DEVNAMELEN];            /* Interface name                    */
  struct sockaddr_storage ss;          /* Address as a sockaddr_storage var */
  u8 addr[16];                         /* Address as a 128bit array         */        
  u16 netmask_bits;                    /* Prefix length                     */
  u8 dev_no;                           /* Netlink device number             */
  u8 scope;                            /* Scope                             */
  u8 flags;                            /* Interface flags                   */
  u8 mac[6];                           /* MAC addr. (I wish we could get it)*/
}if6_t;

# 获取 IPv6 接口信息
int getinterfaces_inet6_linux(if6_t *ifbuf, int max_ifaces);

# 定义 IPv6 路由信息文件路径
#define PATH_PROC_IPV6ROUTE "/proc/net/ipv6_route"
/* 定义 IPv6 路由表项的数据结构 */
typedef struct sys_route6 {
  struct in6_addr dst_net;             /* 目标网络地址 */
  u8 dst_prefix;                       /* 目标网络前缀长度 */
  struct in6_addr src_net;             /* 源网络地址 */
  u8 src_prefix;                       /* 源网络前缀长度 */
  struct in6_addr next_hop;            /* 下一跳地址 - 如果没有则为 0 */
  u32 metric;                          /* 路由的度量值 */
  u32 ref_count;                       /* 引用计数 */
  u32 use_count;                       /* 使用计数 */
  u32 flags;                           /* 标志位 */
  char devname[DEVNAMELEN];            /* 设备名称 */
}route6_t;

/* 获取 Linux 系统中的 IPv6 路由表项 */
int getroutes_inet6_linux(route6_t *rtbuf, int max_routes);

/* 根据目标地址获取 Linux 系统中的 IPv6 路由表项 */
route6_t *route_dst_ipv6_linux(const struct sockaddr_storage *const dst);

#endif /* UTILS_NET_H */
```