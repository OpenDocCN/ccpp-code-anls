# `nmap\tcpip.h`

```
/* $Id$ */

// 如果 TCPIP_H 未定义，则定义 TCPIP_H
#ifndef TCPIP_H
#define TCPIP_H

// 包含 nbase.h 文件
#include "nbase.h"

// 包含 pcap.h 文件
#include <pcap.h>

// 声明 Target 类
class Target;

// 如果未定义 INET_ADDRSTRLEN，则定义为 16
#ifndef INET_ADDRSTRLEN
#define INET_ADDRSTRLEN 16
#endif

// 创建原始套接字
int nmap_raw_socket();

// 用于跟踪发送或接收的所有数据包（例如 --packet-trace 选项）
class PacketTrace {
 public:
  static const int SENT=1; // 这两个值不能更改
  static const int RCVD=2;
  typedef int pdirection;
  // 如果启用数据包跟踪，则接收 IP 数据包并打印
  // 'packet' 必须指向 IPv4 头部。方向必须是 PacketTrace::SENT 或 PacketTrace::RCVD。
  // 可选的 'now' 参数通过避免调用 gettimeofday() 函数，使此函数稍微更有效率。
  static void trace(pdirection pdir, const u8 *packet, u32 len,
                    struct timeval *now=NULL);
  // 如果启用数据包跟踪，则在尝试连接时添加跟踪条目
  // 作为协议传递 IPPROTO_TCP 或 IPPROTO_UDP。sock 可能是 sockaddr_in 或 sockaddr_in6。
  // 将 connect 的返回代码传递给 connectrc。如果返回代码是 -1，则获取 errno 并将其作为 connect_errno 传递。
  static void traceConnect(u8 proto, const struct sockaddr *sock,
                           int socklen, int connectrc, int connect_errno,
                           const struct timeval *now);
  // 如果启用数据包跟踪，则接收 ARP 数据包并打印
  // 'frame' 必须指向 14 字节的以太网头部（例如以目标地址开头）。
  // 方向必须是 PacketTrace::SENT 或 PacketTrace::RCVD。
  // 可选的 'now' 参数通过避免调用 gettimeofday() 函数，使此函数稍微更有效率。
  static void traceArp(pdirection pdir, const u8 *frame, u32 len,
                                    struct timeval *now);
  // 如果启用数据包跟踪，则接收 ND 数据包并打印
  static void traceND(pdirection pdir, const u8 *frame, u32 len,
                                    struct timeval *now);
};
// 定义一个名为 PacketCounter 的类
class PacketCounter {
 public:
  // 构造函数，初始化发送数据包数、发送字节数、接收数据包数、接收字节数
  PacketCounter() : sendPackets(0), sendBytes(0), recvPackets(0), recvBytes(0) {}
  // 根据操作系统不同定义不同长度的整型变量
#if WIN32
  unsigned __int64
#else
  unsigned long long
#endif
          sendPackets, sendBytes, recvPackets, recvBytes;
};

/* Some systems might not have this */
// 如果系统没有定义 IPPROTO_IGMP，则定义 IPPROTO_IGMP 为 2
#ifndef IPPROTO_IGMP
#define IPPROTO_IGMP 2
#endif

/* Prototypes */
// 将 sockaddr_storage 中的 IP 地址转换为 IPv4 或 IPv6 的 IP 地址字符串
const char *inet_socktop(const struct sockaddr_storage *ss);

// 尝试解析给定的主机名（或字面 IP）为 sockaddr 结构体
// 调用 getaddrinfo 函数并返回相同的 addrinfo 链表
// 解析失败或出错时返回 NULL
struct addrinfo *resolve_all(const char *hostname, int pf);

// 根据目标地址（dst）确定发送到该地址所需的源地址和接口
// 如果找不到路由，则返回 false，rnfo 未定义
// 如果找到路由，则返回 true，并用路由详细信息填充 rnfo
int nmap_route_dst(const struct sockaddr_storage *dst, struct route_nfo *rnfo);

// 发送预先构建的 IPv4 或 IPv6 数据包
int send_ip_packet(int sd, const struct eth_nfo *eth,
  const struct sockaddr_storage *dst,
  const u8 *packet, unsigned int packetlen);

// 通过给定信息构建 IP 数据包（包括 IP 头）
// 分配新缓冲区存储数据包内容，并返回该缓冲区
// 该函数不会实际发送数据包，调用者在使用完数据包后必须删除缓冲区
// 数据包长度存储在 packetlen 中，packetlen 必须是有效的 int 指针
# 构建一个原始的 IPv4 数据包，包括 IP 头部，根据给定的信息填充字段。分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。这个函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将存储在 packetlen 中，packetlen 必须是一个有效的整型指针。
u8 *build_ip_raw(const struct in_addr *source, const struct in_addr *victim,
                 u8 proto,
                 int ttl, u16 ipid, u8 tos, bool df,
                 const u8* ipopt, int ipoptlen,
                 const char *data, u16 datalen,
                 u32 *packetlen);

# 构建一个原始的 IPv6 数据包，根据给定的信息填充字段。分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。这个函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将存储在 outpacketlen 中，outpacketlen 必须是一个有效的整型指针。
u8 *build_ipv6_raw(const struct in6_addr *source,
                   const struct in6_addr *victim, u8 tc, u32 flowlabel,
                   u8 nextheader, int hoplimit,
                   const char *data, u16 datalen, u32 *outpacketlen);

# 构建一个原始的 TCP 数据包（包括 IP 头部），根据给定的信息填充字段。分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。这个函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将存储在 packetlen 中，packetlen 必须是一个有效的整型指针。
u8 *build_tcp_raw(const struct in_addr *source, const struct in_addr *victim,
                  int ttl, u16 ipid, u8 tos, bool df,
                  const u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                  const u8 *options, int optlen,
                  const char *data, u16 datalen,
                  u32 *packetlen);

# 构建一个原始的 TCP 数据包（包括 IPv6 头部），根据给定的信息填充字段。分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。这个函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将存储在 packetlen 中，packetlen 必须是一个有效的整型指针。
u8 *build_tcp_raw_ipv6(const struct in6_addr *source,
                       const struct in6_addr *victim, u8 tc, u32 flowlabel,
                       u8 hoplimit, u16 sport, u16 dport, u32 seq, u32 ack,
                       u8 reserved, u8 flags, u16 window, u16 urp,
                       const u8 *tcpopt, int tcpoptlen, const char *data,
                       u16 datalen, u32 *packetlen);

# 构建并发送一个原始的 TCP 数据包。如果 TTL 是 -1，则选择一个部分随机的 TTL（但可能足够大）。
# 发送原始 TCP 数据包，包括以太网信息、源地址、目标地址、TTL、DF标志、IP选项、源端口、目标端口、序列号、确认号、保留字段、标志位、窗口大小、紧急指针、选项、数据和数据长度
int send_tcp_raw(int sd, const struct eth_nfo *eth,
                  const struct in_addr *source, const struct in_addr *victim,
                  int ttl, bool df,
                  u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                  u8 *options, int optlen,
                  const char *data, u16 datalen);

# 发送伪装的原始 TCP 数据包，包括以太网信息、目标地址、TTL、DF标志、IP选项、源端口、目标端口、序列号、确认号、保留字段、标志位、窗口大小、紧急指针、选项、数据和数据长度
int send_tcp_raw_decoys(int sd, const struct eth_nfo *eth,
                         const struct in_addr *victim,
                         int ttl, bool df,
                         u8* ipopt, int ipoptlen,
                         u16 sport, u16 dport,
                         u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                         u8 *options, int optlen,
                         const char *data, u16 datalen);

# 构建 UDP 数据包（包括 IP 头），通过给定信息填充字段。分配新的缓冲区来存储数据包内容，然后返回该缓冲区。此函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将以 packetlen 返回，packetlen 必须是有效的 int 指针。
u8 *build_udp_raw(const struct in_addr *source, const struct in_addr *victim,
       int ttl, u16 ipid, u8 tos, bool df,
                  u8* ipopt, int ipoptlen,
       u16 sport, u16 dport,
       const char *data, u16 datalen,
       u32 *packetlen);

# 构建 IPv6 的 UDP 数据包，包括源地址、目标地址、流量类别、流标签、跳数限制、源端口、目标端口、数据和数据长度。数据包长度将以 packetlen 返回，packetlen 必须是有效的 int 指针。
u8 *build_udp_raw_ipv6(const struct in6_addr *source,
                       const struct in6_addr *victim, u8 tc, u32 flowlabel,
                       u8 hoplimit, u16 sport, u16 dport,
                       const char *data, u16 datalen, u32 *packetlen);
# 发送原始 UDP 数据包
int send_udp_raw(int sd, const struct eth_nfo *eth,
                  struct in_addr *source, const struct in_addr *victim,
                  int ttl, u16 ipid,
                  u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  const char *data, u16 datalen);

# 发送伪装的原始 UDP 数据包
int send_udp_raw_decoys(int sd, const struct eth_nfo *eth,
                         const struct in_addr *victim,
                         int ttl, u16 ipid,
                         u8* ipops, int ip,
                         u16 sport, u16 dport,
                         const char *data, u16 datalen);

# 构建 SCTP 数据包（包括 IP 头），通过给定的信息填充字段。分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。此函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将以 packetlen 返回，packetlen 必须是有效的 int 指针。
u8 *build_sctp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u16 sport, u16 dport,
                   u32 vtag, char *chunks, int chunkslen,
                   const char *data, u16 datalen,
                   u32 *packetlen);

# 构建 IPv6 的 SCTP 数据包，通过给定的信息填充字段。返回数据包长度，并分配一个新的缓冲区来存储数据包内容。
u8 *build_sctp_raw_ipv6(const struct in6_addr *source,
                        const struct in6_addr *victim, u8 tc, u32 flowlabel,
                        u8 hoplimit, u16 sport, u16 dport, u32 vtag,
                        char *chunks, int chunkslen, const char *data, u16 datalen,
                        u32 *packetlen);
/* Builds an ICMP packet (including an IP header) by packing the
   fields with the given information.  It allocates a new buffer to
   store the packet contents, and then returns that buffer.  The
   packet is not actually sent by this function.  Caller must delete
   the buffer when finished with the packet.  The packet length is
   returned in packetlen, which must be a valid int pointer. The
   id/seq will be converted to network byte order (if it differs from
   HBO) */
u8 *build_icmp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u16 seq, unsigned short id, u8 ptype, u8 pcode,
                   const char *data, u16 datalen, u32 *packetlen);
// 构建一个 ICMP 数据包（包括 IP 头），通过给定的信息填充字段。它分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。该函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将在 packetlen 中返回，packetlen 必须是一个有效的 int 指针。id/seq 将被转换为网络字节顺序（如果与 HBO 不同）

u8 *build_icmpv6_raw(const struct in6_addr *source,
                     const struct in6_addr *victim, u8 tc, u32 flowlabel,
                     u8 hoplimit, u16 seq, u16 id, u8 ptype, u8 pcode,
                     const char *data, u16 datalen, u32 *packetlen);
// 构建一个 ICMPv6 数据包（包括 IP 头），通过给定的信息填充字段。它分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。该函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将在 packetlen 中返回，packetlen 必须是一个有效的 int 指针。

/* Builds an IGMP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in packetlen,
   which must be a valid int pointer.
 */
u8 *build_igmp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u8 ptype, u8 pcode,
                   const char *data, u16 datalen, u32 *packetlen);
// 通过给定的信息填充字段，构建一个 IGMP 数据包（包括 IP 头）。它分配一个新的缓冲区来存储数据包内容，然后返回该缓冲区。该函数不会实际发送数据包。调用者在完成数据包后必须删除缓冲区。数据包长度将在 packetlen 中返回，packetlen 必须是一个有效的 int 指针。

// 返回从 libpcap（通过 readip_pcap() 获得的）获得的数据包接收时间值是否应被视为有效。当无效时（Windows 和 Amiga），readip_pcap 返回你调用它的时间。
bool pcap_recv_timeval_valid();
// 返回从 libpcap（通过 readip_pcap() 获得的）获得的数据包接收时间值是否应被视为有效。当无效时（Windows 和 Amiga），readip_pcap 返回你调用它的时间。
/* 打印从 pcap 描述符中获取的统计信息（接收和丢弃的数据包数量） */
void pcap_print_stats(int logt, pcap_t *pd);

/* 一个我编写的用于调试的简单函数，显示 TCP 数据包的重要字段 */
int readtcppacket(const u8 *packet, int readdata);
int readudppacket(const u8 *packet, int readdata);

/* 用数据包统计信息填充 buf（最多 buflen 个字符，如果必要则截断，但始终终止）。返回 buf。如果出现问题，则中止。 */
char *getFinalPacketStats(char *buf, int buflen);

/* 此函数尝试从接收到的数据包中确定目标的以太网 MAC 地址，方法如下：
   1）如果 linkhdr 是以太网头部，则获取源 MAC 地址（否则放弃）
   2）如果 overwrite 为 0 并且已为此目标设置了 MAC 地址，则放弃。
   3）如果数据包源地址不是目标，则放弃。
   4）使用路由表尝试确定目标是否直接连接到运行 Nmap 的源主机。如果是，则设置 MAC 地址。

   如果最终设置了 MAC 地址，则此函数返回 0，否则返回非零值
*/
int setTargetMACIfAvailable(Target *target, struct link_header *linkhdr,
                            const struct sockaddr_storage *src, int overwrite);

/* 此函数确保目标的下一跳 MAC 地址已填充。如果目标直接连接，则此地址为目标自己的 MAC 地址，否则为下一跳 MAC 地址。如果函数结束时地址已设置，则返回 true，否则返回 false。此函数首先检查是否已设置，如果没有，则尝试 ARP 缓存，如果失败，则自己发送 ARP 请求。如果涉及许多直接连接的机器，则应在 ARP 扫描后调用此函数。 */
bool setTargetNextHopMAC(Target *target);

bool getNextHopMAC(const char *iface, const u8 *srcmac, const struct sockaddr_storage *srcss,
                   const struct sockaddr_storage *dstss, u8 *dstmac);
/* 如果 rcvdtime 非空并且返回一个数据包，rcvd 将被填充为 pcap 捕获数据包的时间。如果 linknfo 不为空，lnkinfo->headerlen 和 lnkinfo->header 将被填充为适当的值。 */
const u8 *readipv4_pcap(pcap_t *pd, unsigned int *len, long to_usec,
                    struct timeval *rcvdtime, struct link_header *linknfo, bool validate);

const u8 *readip_pcap(pcap_t *pd, unsigned int *len, long to_usec,
                  struct timeval *rcvdtime, struct link_header *linknfo, bool validate);

/* 检查给定的 tcp 数据包并获取 TCP 时间戳选项信息（如果可用）。请注意，调用者必须确保 "tcp" 包含有效的头部（特别是 th_off 必须是真实的数据包长度，tcp 必须包含它）。如果在头部中找到有效的时间戳选项，则返回非零值，并且 'timestamp' 和 'echots' 参数将填充为适当的值（如果非空）。否则返回 0，并且参数（如果非空）将填充为 0。请记住正确检查错误的方法是查看返回值，因为零的时间戳或 echots 可能是有效的。 */
int gettcpopt_ts(const struct tcp_hdr *tcp, u32 *timestamp, u32 *echots);

/* 最大化套接字描述符的接收缓冲区（最多 500K） */
void max_rcvbuf(int sd);

/* 在套接字上执行接收（recv()），并将结果（最多 len）放入 buf 中。在 'seconds' 后放弃。返回读取的字节数（或在出现错误的情况下返回-1）。它只执行一次 recv（它不会一直进行，直到读取 len 字节）。如果 timedout 不为空，它将被设置为零（没有超时发生）或 1（发生了超时）。 */
int recvtime(int sd, char *buf, int len, int seconds, int *timedout);

#endif /*TCPIP_H*/
```