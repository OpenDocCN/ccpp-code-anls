# `nmap\libnetutil\netutil.h`

```cpp
# 定义网络工具的头文件 netutil.h
#ifndef _NETUTIL_H_
#define _NETUTIL_H_ 1

#ifdef __cplusplus
extern "C" {
#endif
#include <pcap.h>
#ifdef __cplusplus
}
#endif

#include "dnet.h"
#include <nbase.h>

# 定义操作结果的枚举，OP_FAILURE 为 -1，OP_SUCCESS 为 0
enum { OP_FAILURE = -1, OP_SUCCESS = 0 };

# 对于没有定义 IPPROTO_SCTP 的系统（如 MacOS X 或 Win），定义 IPPROTO_SCTP 为 132
#ifndef IPPROTO_SCTP
#define IPPROTO_SCTP 132
#endif

# 定义抽象 IP 头部结构体，包含 IPv4 和 IPv6 公共信息
struct abstract_ip_hdr {
  u8 version; # 版本号，4 或 6
  struct sockaddr_storage src; # 源地址
  struct sockaddr_storage dst; # 目的地址
  u8 proto; # IPv4 协议或 IPv6 下一个头部
  u8 ttl; # IPv4 TTL 或 IPv6 跳数限制
  u32 ipid; # IPv4 IP ID 或 IPv6 流标签
};

# 根据编译器类型定义 NORETURN 宏
#if defined(__GNUC__)
#define NORETURN __attribute__((noreturn))
#elif defined(_MSC_VER)
#define NORETURN __declspec(noreturn)
#else
#define NORETURN
#endif

# 定义 netutil_fatal 函数，用于输出错误信息并终止程序
NORETURN void netutil_fatal(const char *str, ...)
     __attribute__ ((format (printf, 1, 2)));

# 定义 netutil_error 函数，用于输出错误信息
int netutil_error(const char *str, ...)
     __attribute__ ((format (printf, 1, 2)));
# 这个函数将以零结尾的 'txt' 字符串转换为二进制 'data'。
# 用于解析用户输入的 IP 选项。一些可能的输入字符串和结果示例：
#    '\x01*2\xA2'    -> [0x01,0x01,0xA2]    // 使用 'x' 表示十六进制
#    '\01\01\255'    -> [0x01,0x01,0xFF]    // 不使用 'x' 表示十进制
#    '\x01\x00*2'    -> [0x01,0x00,0x00]    // '*' 表示复制字符
#    'R'        -> 具有 9 个插槽的记录路由
#    'S 192.168.0.1 172.16.0.1' -> 具有 2 个插槽的严格路由
#    'L 192.168.0.1 172.16.0.1' -> 具有 2 个插槽的宽松路由
#    'T'        -> 具有 9 个插槽的记录时间戳
#    'U'        -> 具有 4 个插槽的记录时间戳和 IP 地址
# 成功时，函数返回存储在 "data" 中的最终二进制选项的长度。
# 在出现错误的情况下，返回 OP_FAILURE，并将 "errstr" 缓冲区填充错误消息（除非为 NULL）。
# 请注意，返回的错误消息不包含末尾的换行符。

int parse_ip_options(const char *txt, u8 *data, int datalen, int* firsthopoff, int* lasthopoff, char *errstr, size_t errstrlen);

# 使用 getaddrinfo 解析给定的主机名或 IP 地址，并将第一个结果（如果有）存储在 *ss 和 *sslen 中。
# 如果不关心端口的值将设置在 *ss 中；如果不关心端口，将设置为 0。
# af 可能是 AF_UNSPEC，在这种情况下，getaddrinfo 可能返回 IPv4 和 IPv6 结果；哪个是第一个取决于系统配置。
# 成功时返回 0，失败时返回 getaddrinfo 返回代码（适用于传递给 gai_strerror）。
# 当此函数返回 0 时，*ss 和 *sslen 总是定义的。

int resolve(const char *hostname, unsigned short port, struct sockaddr_storage *ss, size_t *sslen, int af);

# 与 resolve 类似，但不对主机名进行 DNS 解析；第一个参数必须是数字 IP 地址的字符串表示形式。
# 解析数值型 IP 地址和端口，将结果存储在 sockaddr_storage 结构体中
int resolve_numeric(const char *ip, unsigned short port,
  struct sockaddr_storage *ss, size_t *sslen, int af);

/*
 * 如果这是一个保留的 IP 地址，则返回 1，其中“保留”指的是私有地址、不可路由地址，甚至是未分配但极有可能被黑洞化的地址。
 *
 * 我们在进行测试时尽量优化速度。这种优化假设输入中的所有字节值都是同等可能的。
 *
 * 警告：这个函数需要经常关注，因为 IANA 每年分配地址块很多次（尽管这种趋势能够持续多久还是值得怀疑的）。
 *
 * 检查最新的分配情况，请查看 <http://www.iana.org/assignments/ipv4-address-space/ipv4-address-space.txt>，
 * 以及 <http://www.cymru.com/Documents/bogon-bn-nonagg.txt> 查看 bogon 网段。
 */
int ip_is_reserved(struct in_addr *ip);


/* 一些简单的函数，用于维护 IP 到 MAC 地址条目的缓存。函数 mac_cache_get() 在 ss 中查找 IPv4 地址，并填充 'mac' 参数，如果找到则返回 true。否则（未找到），返回 false。
 * 函数 mac_cache_set() 添加一个具有给定 IP（ss）和 MAC 地址的条目。对于 IP ss 的现有条目将被新的 MAC 地址覆盖。mac_cache_set() 总是返回 true。 */
int mac_cache_get(const struct sockaddr_storage *ss, u8 *mac);
int mac_cache_set(const struct sockaddr_storage *ss, u8 *mac);

const void *ip_get_data(const void *packet, unsigned int *len,
  struct abstract_ip_hdr *hdr);
const void *ip_get_data_any(const void *packet, unsigned int *len,
  struct abstract_ip_hdr *hdr);
/* 从 IPv4 数据包中获取上层协议。 */
const void *ipv4_get_data(const struct ip *ip, unsigned int *len);
/* 从 IPv6 数据包中获取上层协议。这将跳过已知的扩展头部。上层载荷的长度存储在 *len 中。协议存储在 *nxt 中。在出现错误时返回 NULL。 */
const void *ipv6_get_data(const struct ip6_hdr *ip6, unsigned int *len, u8 *nxt);

/* 从 IPv6 数据包中获取任意上层协议。这将跳过已知的扩展头部。上层载荷的长度存储在 *len 中。协议存储在 *nxt 中。在出现错误时返回 NULL。 */
const void *ipv6_get_data_any(const struct ip6_hdr *ip6, unsigned int *len, u8 *nxt);

/* 标准的 BSD 互联网校验和例程。 */
unsigned short in_cksum(u16 *ptr, int nbytes);

/* 计算给定数据与 IPv4 伪头部连接的互联网校验和。参见 RFC 1071 和 TCP/IP Illustrated 第 3.2、11.3 和 17.3 节。 */
unsigned short ipv4_pseudoheader_cksum(const struct in_addr *src,
  const struct in_addr *dst, u8 proto, u16 len, const void *hstart);

/* 计算给定数据与 IPv6 伪头部连接的互联网校验和。参见 RFC 2460 第 8.1 节。 */
u16 ipv6_pseudoheader_cksum(const struct in6_addr *src,
  const struct in6_addr *dst, u8 nxt, u32 len, const void *hstart);

/* 设置套接字选项，包括 IP 头部。 */
void sethdrinclude(int sd);

/* 设置套接字的 IP 选项。 */
void set_ipoptions(int sd, void *opts, size_t optslen);

/* 设置套接字的 TTL（生存时间）值。 */
void set_ttl(int sd, int ttl);

/* 返回系统是否正确支持 pcap_get_selectable_fd()。 */
int pcap_selectable_fd_valid();

/* 返回是否一对一支持 pcap_get_selectable_fd()。 */
int pcap_selectable_fd_one_to_one();

/* 调用此函数而不是直接调用 pcap_get_selectable_fd（否则你的代码在 Windows 上将无法编译）。在似乎不正确支持 pcap_get_selectable_fd() 函数的系统上，返回 -1，否则简单地调用 pcap_selectable_fd 并返回结果。如果你只想测试函数是否受支持，使用 pcap_selectable_fd_valid()。 */
int my_pcap_get_selectable_fd(pcap_t *p);
/* 这两个函数在无法在 pcap 设备上使用 select() 时返回 -1，超时返回 0，成功返回 >0。如果 select() 失败，我们会因为无法使用从 my_pcap_get_selectable_fd() 获取的文件描述符而退出 */
int pcap_select(pcap_t *p, struct timeval *timeout);
int pcap_select(pcap_t *p, long usecs);

typedef enum { devt_ethernet, devt_loopback, devt_p2p, devt_other  } devtype;

#define MAX_LINK_HEADERSZ 24
struct link_header {
  int datalinktype; /* pcap_datalink()，比如 DLT_EN10MB */
  int headerlen; /* 如果头部太大或不可用，则为 0 */
  u8 header[MAX_LINK_HEADERSZ];
};

/* 关于接口的与 Nmap 相关的信息 */
struct interface_info {
  char devname[16];
  char devfullname[16]; /* 可包括别名信息，比如 eth0:2 */
  struct sockaddr_storage addr;
  u16 netmask_bits; /* CIDR 样式。所以 24 表示 C 类（255.255.255.0）*/
  devtype device_type; /* devt_ethernet, devt_loopback, devt_p2p, devt_other */
  unsigned int ifindex; /* 索引（由 if_indextoname 和 sin6_scope_id 使用） */
  int device_up; /* 如果设备启用，则为真 */
  int mtu; /* 接口的 MTU 大小 */
  u8 mac[6]; /* 如果 device_type 为 devt_ethernet，则为接口的 MAC 地址 */
};

struct route_nfo {
  struct interface_info ii;

/* 如果目标直接连接在网络上（不需要路由），则为真。 */
  int direct_connect;

/* 这是应该由数据包使用的源地址。如果您正在使用本地主机接口扫描机器上另一个接口的 IP，则可能与 ii.addr 不同 */
  struct sockaddr_storage srcaddr;

  /* 如果 direct_connect 为 0，则填入下一个必须路由到目标的跳数 */
  struct sockaddr_storage nexthop;
};

struct sys_route {
  struct interface_info *device;
  struct sockaddr_storage dest;
  u16 netmask_bits;
  struct sockaddr_storage gw; /* 网关 - 如果没有则为 0 */
  int metric;
};
# 定义一个结构体，包含源MAC地址、目的MAC地址、eth_t指针和设备名称
struct eth_nfo {
  char srcmac[6]; // 源MAC地址
  char dstmac[6]; // 目的MAC地址
  eth_t *ethsd; // 可选，但可以提高性能。如果不可用，则设置为NULL
  char devname[16]; // 如果ethsd为NULL，则需要设备名称
};

/* 一个简单的函数，用于缓存dnet中一个设备的eth_t，以避免多次打开、关闭和重新打开它。如果给定不同的设备，此函数将关闭第一个设备。因此，这绝对不应该被需要同时处理多个设备的程序使用。此外，绝对不要对从此函数获取的设备进行eth_close()。相反，可以调用eth_close_cached()来关闭缓存的任何设备（如果有的话）。如果打开设备失败，则返回NULL。 */
eth_t *eth_open_cached(const char *device);

/* 参见eth_open_cached的描述 */
void eth_close_cached();

/* 获取类似IPPROTO_TCP、IPPROTO_UDP或IPPROTO_IP的协议号，并返回其小写形式的ASCII表示（如果无法识别该数字，则返回"unknown"）。返回的字符串为小写形式。 */
const char *proto2ascii_lowercase(u8 proto) ;

/* 与proto2ascii()相同，但返回大写形式的字符串。 */
const char *proto2ascii_uppercase(u8 proto);

/* 获取指向长度为len的tcp选项的optp，并将结果存储在结果缓冲区中。结果可能类似于"<mss 1452,sackOK,timestamp 45848914 0,nop,wscale 7>" */
void tcppacketoptinfo(u8 *optp, int len, char *result, int bufsize);

/* 使用该地址将IP地址转换为设备（即ppp0 eth0）。提供的“dev”必须至少能够容纳32个字节。成功返回0，错误返回-1。 */
int ipaddr2devname( char *dev, const struct sockaddr_storage *addr );

/* 将网络接口名称（即ppp0 eth0）转换为IP地址。成功返回0，错误返回-1。 */
int devname2ipaddr(char *dev, struct sockaddr_storage *addr);
/* 比较两个 sockaddr_storage 结构是否相等，返回 1 表示相等，0 表示不相等 */
int sockaddr_equal(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b);

/* 比较两个 sockaddr_storage 结构是否相等，同时考虑网络掩码，nbits 表示网络掩码位数 */
int sockaddr_equal_netmask(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b, u16 nbits);

/* 检查给定的 sockaddr_storage 结构是否为零 */
int sockaddr_equal_zero(const struct sockaddr_storage *s);

/* 返回一个分配的 struct interface_info 数组，表示可用的接口。接口数量存储在 howmany 中。
   这个函数只是对结果进行缓存；真正的工作是在 getinterfaces_dnet() 或 getinterfaces_siocgifconf() 中完成。
   出错时返回 NULL，howmany 设置为 -1，如果 errstr 不为 NULL，则错误缓冲区中将包含错误消息。 */
struct interface_info *getinterfaces(int *howmany, char *errstr, size_t errstrlen);

/* 释放 getinterfaces 使用的缓存的 struct interface_info 数组。可以用于强制刷新或释放内存。 */
void freeinterfaces(void);

/* 这个结构被滥用，根据使用的函数不同，可以携带路由或接口。 */
struct dnet_collector_route_nfo {
  struct sys_route *routes;
  int numroutes;
  int capacity; /* 路由或接口的容量，根据上下文而定 */
  struct interface_info *ifaces;
  int numifaces;
};

/* 查找具有给定名称（iname）和地址族类型的接口，并返回相应的 interface_info（如果找到）。
   将接受 devname 或 devfullname 的匹配。如果找不到，则返回 NULL */
struct interface_info *getInterfaceByName(const char *iname, int af);
/* 解析系统路由表，将每个路由转换为sys_route条目。返回sys_routes数组。numroutes设置为数组中的路由数。只有在第一次调用时才读取路由表--后续结果将被缓存。返回的路由数组按netmask排序，最具体的匹配排在前面。出错时，返回NULL，howmany设置为-1，如果errstr不为NULL，则提供的错误缓冲区"errstr"将包含错误消息。 */
struct sys_route *getsysroutes(int *howmany, char *errstr, size_t errstrlen);

/* 尝试确定提供的地址是否对应于本地主机。(例如：地址类似于127.x.x.x，地址与本地网络接口的地址匹配，等等)。如果地址被认为是本地主机，则返回1，否则返回0 */
int islocalhost(const struct sockaddr_storage *ss);

/* 确定提供的地址是否对应于私有的、不可通过互联网路由的地址。参见RFC1918了解详情。还根据RFC3927检查链路本地地址。如果地址是私有的，则返回1，否则返回0 */
int isipprivate(const struct sockaddr_storage *addr);

/* 获取IPv4数据包的IP选项字段中的二进制数据，并返回包含找到的选项的ASCII描述的字符串。该函数返回指向静态缓冲区的指针，后续调用将覆盖该缓冲区。出错时，返回NULL */
char *format_ip_options(const u8* ipopt, int ipoptlen);
/* 返回一个关于 IP 数据包的 ASCII 信息的缓冲区，可能看起来像 "TCP 127.0.0.1:50923 > 127.0.0.1:3 S ttl=61 id=39516 iplen=40 seq=625950769" 或 "ICMP PING (0/1) ttl=61 id=39516 iplen=40"。
 * 返回的缓冲区是静态的，因此在多线程环境中调用此函数时需要适当的同步保护，或者在同一个句子中调用两次（例如作为两个 printf 参数）是不安全的。
 * 显然，调用者不应尝试释放缓冲区。返回的缓冲区保证是以 NULL 结尾的，但不应做出关于其长度的任何假设。
 *
 * 该函数完全支持 IPv4、TCP、UDP、SCTP 和 ICMPv4。
 * 它还支持标准的 IPv6，但不支持其扩展头部。如果一个 IPv6 数据包包含 ICMPv6 头部，输出将反映这一点，但不会对 ICMPv6 内容进行解析。
 *
 * 输出有三种不同的详细级别。参数 "detail" 决定输出的详细程度。它应该取以下值之一：
 *
 *    LOW_DETAIL    (0x01): 传统输出。
 *    MEDIUM_DETAIL (0x02): 比传统输出更详细。
 *    HIGH_DETAIL   (0x03): 协议头的几乎每个字段的内容。
 */
#define LOW_DETAIL     1
#define MEDIUM_DETAIL  2
#define HIGH_DETAIL    3
const char *ippackethdrinfo(const u8 *packet, u32 len, int detail);
/* Takes an IPv4 destination address (dst) and tries to determine the
 * source address and interface necessary to route to this address.
 * If no route is found, 0 is returned and "rnfo" is undefined.  If
 * a route is found, 1 is returned and "rnfo" is filled in with all
 * of the routing details. If the source address needs to be spoofed,
 * it should be passed through "spoofss" (otherwise NULL should be
 * specified), along with a suitable network device (parameter "device").
 * Even if spoofss is NULL, if user specified a network device with -e,
 * it should still be passed. Note that it's OK to pass either NULL or
 * an empty string as the "device", as long as spoofss==NULL. */
int route_dst(const struct sockaddr_storage *dst, struct route_nfo *rnfo,
              const char *device, const struct sockaddr_storage *spoofss);
/* Send an IP packet over a raw socket. */
int send_ip_packet_sd(int sd, const struct sockaddr_in *dst, const u8 *packet, unsigned int packetlen);
/* Send an IP packet over an ethernet handle. */
int send_ip_packet_eth(const struct eth_nfo *eth, const u8 *packet, unsigned int packetlen);
/* Sends the supplied pre-built IPv4 packet. The packet is sent through
 * the raw socket "sd" if "eth" is NULL. Otherwise, it gets sent at raw
 * ethernet level. */
int send_ip_packet_eth_or_sd(int sd, const struct eth_nfo *eth,
  const struct sockaddr_in *dst, const u8 *packet, unsigned int packetlen);
/* Sends an IPv4 packet. */
int send_ipv6_packet_eth_or_sd(int sd, const struct eth_nfo *eth,
  const struct sockaddr_in6 *dst, const u8 *packet, unsigned int packetlen);
/* Create and send all fragments of a pre-built IPv4 packet.
 * Minimal MTU for IPv4 is 68 and maximal IPv4 header size is 60
 * which gives us a right to cut TCP header after 8th byte */
int send_frag_ip_packet(int sd, const struct eth_nfo *eth,
  const struct sockaddr_in *dst,
  const u8 *packet, unsigned int packetlen, u32 mtu);
/* Wrapper for system function sendto(), which retries a few times when
 * the call fails. It also prints informational messages about the
 * errors encountered. It returns the number of bytes sent or -1 in
 * case of error. */
int Sendto(const char *functionname, int sd, const unsigned char *packet,
           int len, unsigned int flags, struct sockaddr *to, int tolen);
/* 封装了系统函数 sendto()，在调用失败时进行几次重试。还会打印有关遇到的错误的信息。如果发生错误，则返回发送的字节数，否则返回-1。 */

/* This function is  used to obtain a packet capture handle to look at
 * packets on the network. It is actually a wrapper for libpcap's
 * pcap_open_live() that takes care of compatibility issues and error
 * checking.  Prints an error and fatal()s if the call fails, so a
 * valid pcap_t will always be returned. */
pcap_t *my_pcap_open_live(const char *device, int snaplen, int promisc, int to_ms);
/* 该函数用于获取数据包捕获句柄，以查看网络上的数据包。实际上是 libpcap 的 pcap_open_live() 的包装器，负责处理兼容性问题和错误检查。如果调用失败，则打印错误并执行 fatal()，因此始终会返回有效的 pcap_t。 */

/* Set a pcap filter */
void set_pcap_filter(const char *device, pcap_t *pd, const char *bpf, ...);
/* 设置一个 pcap 过滤器 */

/* Issues an ARP request for the MAC of targetss (which will be placed
   in targetmac if obtained) from the source IP (srcip) and source mac
   (srcmac) given.  "The request is ussued using device dev to the
   broadcast MAC address.  The transmission is attempted up to 3
   times.  If none of these elicit a response, false will be returned.
   If the mac is determined, true is returned. The last parameter is
   a pointer to a callback function that can be used for packet traceing.
   This is intended to be used by Nmap only. Any other calling this
   should pass NULL instead. */
bool doArp(const char *dev, const u8 *srcmac,
                  const struct sockaddr_storage *srcip,
                  const struct sockaddr_storage *targetip,
                  u8 *targetmac,
                  void (*traceArp_callback)(int, const u8 *, u32 , struct timeval *));
/* 为目标的 MAC 地址发出 ARP 请求（如果获取到，则放入 targetmac 中），源 IP（srcip）和源 MAC（srcmac）给定。使用设备 dev 发出请求到广播 MAC 地址。尝试传输最多 3 次。如果这些尝试都没有得到响应，则返回 false。如果确定了 MAC 地址，则返回 true。最后一个参数是一个指向回调函数的指针，该函数可用于数据包跟踪。这仅适用于 Nmap 使用。其他调用此函数的应该传递 NULL。 */
/* 从源 IP 和源 MAC 发送邻居请求，获取目标 MAC 的地址
   如果获取到目标 MAC 地址，将其放入 targetmac 中
   使用设备 dev 发送到多播 MAC 地址
   最多尝试 3 次传输
   如果没有响应，则返回 false
   如果确定了 MAC 地址，则返回 true
   最后一个参数是用于数据包跟踪的回调函数的指针
   这仅供 Nmap 使用，其他调用此函数的应该传入 NULL */
bool doND(const char *dev, const u8 *srcmac,
                  const struct sockaddr_storage *srcip,
                   const struct sockaddr_storage *targetip,
                   u8 *targetmac,
                   void (*traceArp_callback)(int, const u8 *, u32 , struct timeval *)
                    ) ;

/* 尝试从 pcap 描述符 pd 中读取一个 IPv4/Ethernet ARP 回复数据包
   如果接收到一个，填充 sendermac（必须传入 6 个字节）、senderIP 和 rcvdtime（如果不关心可以传入 NULL），并返回 1
   如果超时并且没有读取到 ARP 请求，则返回 0
   to_usec 是超时时间（微秒），使用 0 尽量避免阻塞
   如果出现错误，则返回 -1 或退出
   最后一个参数是用于数据包跟踪的回调函数的指针
   这仅供 Nmap 使用，其他调用此函数的应该传入 NULL */
int read_arp_reply_pcap(pcap_t *pd, u8 *sendermac,
                        struct in_addr *senderIP, long to_usec,
                        struct timeval *rcvdtime,
                        void (*traceArp_callback)(int, const u8 *, u32 , struct timeval *));
/* 从 pcap 描述符 pd 中读取一个 IP 数据包。输入参数包括 pd、to_usec 和 accept_callback。
   如果接收到的帧通过 accept_callback，则填充输出参数 p、head、rcvdtime、datalink 和 offset，
   并返回 1。如果在超时之前没有通过任何帧，则返回 0，输出参数的值未定义。 */
int read_reply_pcap(pcap_t *pd, long to_usec,
  bool (*accept_callback)(const unsigned char *, const struct pcap_pkthdr *, int, size_t),
  const unsigned char **p, struct pcap_pkthdr **head, struct timeval *rcvdtime,
  int *datalink, size_t *offset);

/* 从文件中读取单个主机规范，用于 -iL 和 --excludefile。返回读取的字符串长度；
   如果返回值 >= n，则表示溢出。如果没有要读取的规范，则返回 0。缓冲区始终以空字符结尾。 */
size_t read_host_from_file(FILE *fp, char *buf, size_t n);

/* 从提供的流中返回下一个目标主机规范。
   如果参数 "random" 设置为 true，则函数将返回一个随机的、非保留的 IP 地址的十进制点表示法。 */
const char *grab_next_host_spec(FILE *inputfd, bool random, int argc, const char **fakeargv);

#ifdef WIN32
/* 将 dnet 接口名称转换为长 pcap 样式。这也会缓存数据以加快速度。
   填充 pcapdev（最多 pcapdevlen）并在找到任何内容时返回 true。否则返回 false。仅在 Windows 上需要。 */
int DnetName2PcapName(const char *dnetdev, char *pcapdev, int pcapdevlen);
#endif
/** 尝试增加此进程的打开文件描述符限制。
  * @param "desired" 是期望的最大打开描述符数。传入负值以设置最大允许值。
  * @return 可以设置的最大打开描述符数，或者在失败的情况下返回 0。
  * @warning 如果 "desired" 小于当前限制，则不执行任何操作。此函数只能用于增加限制，不能用于减少限制。 */
int set_max_open_descriptors(int desired_max);

/** 返回此进程的打开文件描述符限制。
  * @return 最大打开描述符数，或者在失败的情况下返回 0。 */
int get_max_open_descriptors();

/* 将此进程的打开文件描述符限制最大化，达到最大允许值 */
int max_sd();

#endif /* _NETUTIL_H_ */
```