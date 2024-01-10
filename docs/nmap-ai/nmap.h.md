# `nmap\nmap.h`

```
/* $Id$ */

#ifndef NMAP_H
#define NMAP_H

/************************INCLUDES**********************************/

#ifdef HAVE_CONFIG_H
#include "nmap_config.h"
#else
#ifdef WIN32
#include "nmap_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifdef __amigaos__
#include "nmap_amigaos.h"
#endif

#if HAVE_UNISTD_H
#include <unistd.h>
#endif

#ifdef HAVE_BSTRING_H
#include <bstring.h>
#endif

/* Keep assert() defined for security reasons */
#undef NDEBUG

#include <assert.h>

/*#include <net/if_arp.h> *//* defines struct arphdr needed for if_ether.h */
// #if HAVE_NET_IF_H
// #ifndef NET_IF_H  /* why doesn't OpenBSD do this?! */
// #include <net/if.h>
// #define NET_IF_H
// #endif
// #endif
// #if HAVE_NETINET_IF_ETHER_H
// #ifndef NETINET_IF_ETHER_H
// #include <netinet/if_ether.h>
// #define NETINET_IF_ETHER_H
// #endif /* NETINET_IF_ETHER_H */
// #endif /* HAVE_NETINET_IF_ETHER_H */

/*******  DEFINES  ************/

#ifdef NMAP_OEM
#include "../nmap-build/nmap-oem.h"
#endif

#ifndef NMAP_NAME
#define NMAP_NAME "Nmap"
#endif
#define NMAP_URL "https://nmap.org"

#define _STR(X) #X
#define STR(X)  _STR(X)

#ifndef NMAP_VERSION
/* Edit this definition only within the quotes, because it is read from this
   file by the makefiles. */
#define NMAP_MAJOR 7
#define NMAP_MINOR 94
#define NMAP_BUILD 2
/* SVN, BETA, etc. */
#define NMAP_SPECIAL "SVN"

#define NMAP_VERSION STR(NMAP_MAJOR) "." STR(NMAP_MINOR) NMAP_SPECIAL
#define NMAP_NUM_VERSION STR(NMAP_MAJOR) "." STR(NMAP_MINOR) "." STR(NMAP_BUILD) ".0"
#endif

#define NMAP_XMLOUTPUTVERSION "1.05"

/* User configurable #defines: */
#define MAX_PROBE_PORTS 10     /* How many TCP probe ports are allowed ? */
/* Default number of ports in parallel.  Doesn't always involve actual
   sockets.  Can also adjust with the -M command line option.  */
#define MAX_SOCKETS 36
# 定义最大超时次数，即在一系列连接尝试中有多少次超时后我们认定主机已经不可用
#define MAX_TIMEOUTS MAX_SOCKETS   
# 默认的 TCP 探测端口，如果用户未指定，则 TCP ping 探测将使用该端口
#define DEFAULT_TCP_PROBE_PORT 80 
# 默认的 TCP 探测端口的字符串表示，如果用户未指定，则使用该字符串
#define DEFAULT_TCP_PROBE_PORT_SPEC STR(DEFAULT_TCP_PROBE_PORT)
# 默认的 UDP 探测端口，如果用户未指定，则 UDP ping 探测将使用该端口
#define DEFAULT_UDP_PROBE_PORT 40125 
# 默认的 UDP 探测端口的字符串表示，如果用户未指定，则使用该字符串
#define DEFAULT_UDP_PROBE_PORT_SPEC STR(DEFAULT_UDP_PROBE_PORT)
# 默认的 SCTP 探测端口，如果用户未指定，则 SCTP 探测将使用该端口
#define DEFAULT_SCTP_PROBE_PORT 80 
# 默认的 SCTP 探测端口的字符串表示，如果用户未指定，则使用该字符串
#define DEFAULT_SCTP_PROBE_PORT_SPEC STR(DEFAULT_SCTP_PROBE_PORT)
# 默认的协议探测端口的字符串表示，如果用户未指定，则使用该字符串
#define DEFAULT_PROTO_PROBE_PORT_SPEC "1,2,4" 
# 允许的最大伪装主机数量
#define MAX_DECOYS 128 
# TCP SYN 探测的选项：MSS 1460
#define TCP_SYN_PROBE_OPTIONS "\x02\x04\x05\xb4"
# TCP SYN 探测选项的长度
#define TCP_SYN_PROBE_OPTIONS_LEN (sizeof(TCP_SYN_PROBE_OPTIONS)-1)
# 默认发送到同一主机的探测之间的最大延迟
#ifndef MAX_TCP_SCAN_DELAY
#define MAX_TCP_SCAN_DELAY 1000
#endif
# 默认发送到同一主机的 UDP 探测之间的最大延迟
#ifndef MAX_UDP_SCAN_DELAY
#define MAX_UDP_SCAN_DELAY 1000
#endif
# 默认发送到同一主机的 SCTP 探测之间的最大延迟
#ifndef MAX_SCTP_SCAN_DELAY
#define MAX_SCTP_SCAN_DELAY 1000
#endif
# 输出额外服务信息字段时考虑的最大额外主机名、操作系统和设备数量
#define MAX_SERVICE_INFO_FIELDS 5
# 默认等待响应的最小超时时间，过长的等待时间可能导致在极低延迟网络上无法及时检测到丢包
#ifndef MIN_RTT_TIMEOUT
#define MIN_RTT_TIMEOUT 100
#ifndef MAX_RTT_TIMEOUT
#define MAX_RTT_TIMEOUT 10000 /* Never allow more than 10 secs for packet round
                             trip */
#endif
// 如果未定义最大往返时间超时，则定义最大往返时间超时为10000毫秒，不允许超过10秒的数据包往返时间

#define INITIAL_RTT_TIMEOUT 1000 /* Allow 1 second initially for packet responses */
#define INITIAL_ARP_RTT_TIMEOUT 200 /* The initial timeout for ARP is lower */
// 定义初始往返时间超时为1000毫秒，允许初始1秒钟的数据包响应时间
// 定义初始ARP超时为200毫秒，ARP的初始超时较低

#ifndef MAX_RETRANSMISSIONS
#define MAX_RETRANSMISSIONS 10    /* 11 probes to port at maximum */
#endif
// 如果未定义最大重传次数，则定义最大重传次数为10次，最多11次探测端口

#define PING_GROUP_SZ 4096
// 定义预先ping和扫描的主机数量为4096个

/* DO NOT change stuff after this point */
#define UC(b)   (((int)b)&0xff)
#define SA    struct sockaddr  /*Ubertechnique from R. Stevens */
// 不要在此点之后更改内容
// 定义UC宏，将b转换为int类型并取低8位
// 定义SA为struct sockaddr类型，来自R. Stevens的Ubertechnique技术

#define HOST_UNKNOWN 0
#define HOST_UP 1
#define HOST_DOWN 2
// 定义主机状态常量：未知、上线、下线

#define PINGTYPE_UNKNOWN 0
#define PINGTYPE_NONE 1
#define PINGTYPE_ICMP_PING 2
#define PINGTYPE_ICMP_MASK 4
#define PINGTYPE_ICMP_TS 8
#define PINGTYPE_TCP  16
#define PINGTYPE_TCP_USE_ACK 32
#define PINGTYPE_TCP_USE_SYN 64
#define PINGTYPE_CONNECTTCP 256
#define PINGTYPE_UDP  512
#define PINGTYPE_PROTO 2048
#define PINGTYPE_SCTP_INIT 4096
// 定义不同类型的ping常量

#define DEFAULT_IPV4_PING_TYPES (PINGTYPE_ICMP_PING|PINGTYPE_TCP|PINGTYPE_TCP_USE_ACK|PINGTYPE_TCP_USE_SYN|PINGTYPE_ICMP_TS)
#define DEFAULT_IPV6_PING_TYPES (PINGTYPE_ICMP_PING|PINGTYPE_TCP|PINGTYPE_TCP_USE_ACK|PINGTYPE_TCP_USE_SYN)
#define DEFAULT_PING_ACK_PORT_SPEC "80"
#define DEFAULT_PING_SYN_PORT_SPEC "443"
#define DEFAULT_PING_CONNECT_PORT_SPEC "80,443"
// 定义默认的IPv4和IPv6 ping类型，以及默认的ping端口规范
/* 每行主题指纹的最大长度，用于换行 */
#define FP_RESULT_WRAP_LINE_LEN 74

/* 最长的 DNS 名称长度 */
#define FQDN_LEN 254

/* 最大有效载荷：最坏情况是 IPv4 有40字节的选项和 TCP 有20字节的选项 */
#define MAX_PAYLOAD_ALLOWED 65535-60-40

#ifndef recvfrom6_t
#  define recvfrom6_t int
#endif

/***********************PROTOTYPES**********************************/

/* 重命名 main 函数，以便在必要时进行交互模式的预处理 */
int nmap_main(int argc, char *argv[]);

/* 从文件中获取数据，存储在指定的缓冲区中 */
int nmap_fetchfile(char *filename_returned, int bufferlen, const char *file);

/* 收集日志文件的恢复状态 */
int gather_logfile_resumption_state(char *fname, int *myargc, char ***myargv);

#endif /* NMAP_H */
```