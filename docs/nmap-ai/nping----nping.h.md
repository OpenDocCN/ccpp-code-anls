# `nmap\nping\nping.h`

```
#ifndef NPING_H
#define NPING_H 1

/* Common library requirements and definitions *******************************/

#include <stdio.h>          // 标准输入输出
#include <math.h>           // 数学函数
#include <assert.h>         // 断言
#include <nbase.h>          // 自定义库
#include <fcntl.h>          // 文件控制
#include <stdarg.h>         // 可变参数
#include <errno.h>          // 错误码
#include <ctype.h>          // 字符处理
#include <sys/types.h>      // 系统数据类型
#include <sys/stat.h>       // 文件状态

#include "../libnetutil/netutil.h"  // 网络工具库
#include "../libnetutil/npacket.h"   // 网络数据包

#ifdef HAVE_CONFIG_H
    #include "nping_config.h"       // 配置文件
#else
    #ifdef WIN32
        #include "nping_winconfig.h" // Windows 配置文件
    #endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifndef WIN32
    #include <sysexits.h>           // 系统退出码
#endif

#if HAVE_UNISTD_H
    #include <unistd.h>             // 标准符号常量和类型
#endif

#ifdef STDC_HEADERS
    #include <stdlib.h>             // 标准库
#else
    void *malloc();                 // 分配内存
    void *realloc();                // 重新分配内存
#endif

#if STDC_HEADERS || HAVE_STRING_H
    #include <string.h>             // 字符串操作
    #if !STDC_HEADERS && HAVE_MEMORY_H
        #include <memory.h>         // 内存操作
    #endif
#endif

#if HAVE_STRINGS_H
    #include <strings.h>            // 字符串操作
#endif

#ifdef HAVE_BSTRING_H
    #include <bstring.h>            // 字符串操作
#endif

#ifndef WIN32
    #include <sys/wait.h>           // 进程等待
#endif /* !WIN32 */

#if HAVE_SYS_SOCKET_H
    #include <sys/socket.h>         // 套接字
#endif

#if HAVE_NETINET_IN_H
    #include <netinet/in.h>         // 网络地址和端口
#endif

#if HAVE_NETDB_H
    #include <netdb.h>              // 网络数据库操作
#endif

#if TIME_WITH_SYS_TIME
    #include <sys/time.h>           // 时间
    #include <time.h>               // 时间
#else
    #if HAVE_SYS_TIME_H
        #include <sys/time.h>       // 时间
    #else
        #include <time.h>           // 时间
    # endif
#endif

#ifdef HAVE_PWD_H
    #include <pwd.h>                // 口令文件
#endif

#if HAVE_ARPA_INET_H
    #include <arpa/inet.h>          // 网络地址和端口
#endif

#if HAVE_SYS_RESOURCE_H
    #include <sys/resource.h>       // 资源限制
#endif

/* Keep assert() defined for security reasons */
#undef NDEBUG

#define MAXLINE 255                 // 最大行长度

/* CONSTANT DEFINES ***********************************************************
 * @warning It's better not to play with these, because the code may make     *
 * SOME assumptions like "defined value A is an integer greater than defined  *
 * value B" or "value C is an odd integer greater than 0", etc.               */
/* VERBOSITY LEVELS */
/* Less verbosity */
#define QT_4 0   /**< No output at all                                       */
#define QT_3 1   /**< Fatal error messages, help info, version number        */
#define QT_2 2   /**< Warnings and very limited output(just some statistics) */
#define QT_1 3   /**< Start and timing information but no sent/recv packets  */

/* Base level (QT_0 is provided for consistency but should not be used)      */
#define QT_0 4   /**< Normal info (sent/recv packets, statistics...) (DEFAULT */
#define VB_0 4   /**< Normal info (sent/recv packets, statistics...) (DEFAULT)*/

/* More verbosity */
#define VB_1 5   /**< Detailed information about times, flags, etc.          */
#define VB_2 6   /**< Very detailed information about packets,               */
#define VB_3 7   /**< Reserved for future use                                */
#define VB_4 8   /**< Reserved for future use                                */



/* DEBUGGING LEVELS */
#define DBG_0 30 /**< No debug information at all (DEFAULT)                  */
#define DBG_1 31 /**< Very important or high level debug information         */
#define DBG_2 32 /**< Important or medium level debug information            */
#define DBG_3 33 /**< Regular and low level debug information                */
#define DBG_4 34 /**< Messages only a real Nping freak would want to see     */
#define DBG_5 35 /**< Enables Nsock (and other libs) basic tracing           */
#define DBG_6 36 /**< Enables full Nsock (and other libs) tracing            */
#define DBG_7 37 /**< Reserved for future use                                */
#define DBG_8 38 /**< Reserved for future use                                */
#define DBG_9 39 /**< Reserved for future use                                */


#define MAX_IP_PACKET_LEN 65535   /**< Max len of an IP datagram             */
#define MAX_UDP_PAYLOAD_LEN 65507 /**< Check comments in UDPHeader::setSum() */
# 定义最大网络接口名称长度
#define MAX_DEV_LEN 128           /**< Max network interface name length     */

# 在 nping_fatal()、nping_warning() 和 nping_print() 中使用的标志，表示不换行
#define NO_NEWLINE 0x8000 /**< Used in nping_fatal(), nping_warning() and nping_print() */

# 用于数字解析函数的位数计数
/** Bit count for number parsing functions */
#define RANGE_8_BITS  8
#define RANGE_16_BITS 16
#define RANGE_32_BITS 32
#define RANGE_64_BITS 64

# 加密长度
/* Crypto Lengths */
#define CIPHER_BLOCK_SIZE (128/8)
#define CIPHER_KEY_LEN (128/8)
#define MAC_KEY_LEN (128/8)

# 通用可调整定义
/* General tunable defines  **************************************************/
#define NPING_NAME "Nping"
#define NPING_URL "https://nmap.org/nping"
#define NPING_VERSION "0.7.94SVN"

# 默认详细程度
#define DEFAULT_VERBOSITY VB_0
# 默认调试级别
#define DEFAULT_DEBUGGING DBG_0

# 默认发送到每个目标的探测次数
/**< Default number of probes that are sent to each target */
#define DEFAULT_PACKET_COUNT 5          

# 当进行 traceroute 时，发送到每个主机的数据包数量必须更高，因为5个可能不足以到达平均互联网目标。下面的论文建议，互联网主机之间的跳数不超过30，因此在设置 --traceroute 时，将数据包计数设置为48似乎是一个安全的选择。
/* When doing traceroute, the number of packets sent to each host must be
 * higher because 5 is probably not enough to reach the average target on the
 * Internet. The following paper suggests that internet hosts are no more than
 * 30 hops apart, so setting the packet count to 48 when --traceroute is set
 * seems like a safe choice.
 *    Cheng, J., Haining, W. and Kang, GS. (2006). Hop-Count Filtering: An
 *    Effective Defense Against Spoofed DDoS Traffic. Australian Telecommu-
 *    nication Networks & Applications Conference (ATNAC). Australia.
 *    <http://portal.acm.org/citation.cfm?id=948109.948116>
 */
#define TRACEROUTE_PACKET_COUNT 48

# 每个探测之间的默认延迟时间（毫秒）
#define DEFAULT_DELAY 1000              /**< Milliseconds between each probe */

# 所有探测发送后，Nping 等待回复的默认时间（毫秒）
 /** Milliseconds Nping waits for replies after all probes have been sent */
#define DEFAULT_WAIT_AFTER_PROBES 1000 

# 默认 IP 存活时间
#define DEFAULT_IP_TTL 64               /**< Default IP Time To Live         */
# 默认 IP 服务类型
#define DEFAULT_IP_TOS 0                /**< Default IP Type of Service      */

# 默认 IPv6 跳数限制
#define DEFAULT_IPv6_TTL 64             /**< Default IPv6 Hop Limit          */
# 默认 IPv6 通信流量类别
#define DEFAULT_IPv6_TRAFFIC_CLASS 0x00 /**< Default IPv6 Traffic Class      */
# 默认的 TCP 目标端口
#define DEFAULT_TCP_TARGET_PORT 80      /**< Default TCP target port         */

# 默认的 UDP 目标端口
#define DEFAULT_UDP_TARGET_PORT 40125   /**< Default UDP target port         */

# 默认的 UDP 源端口
#define DEFAULT_UDP_SOURCE_PORT 53      /**< Default UDP source port         */

# 默认的 TCP 窗口大小
#define DEFAULT_TCP_WINDOW_SIZE 1480    /**< Default TCP Window size         */

# 当用户只提供选项 -f 而没有 MTU 值时使用的 MTU
#define DEFAULT_MTU_FOR_FRAGMENTATION 72   

# 默认的 ICMP 消息类型：回显请求
#define DEFAULT_ICMP_TYPE 8  /**< Default ICMP message: Echo Request         */

# 默认的 ICMP 代码：0（标准）
#define DEFAULT_ICMP_CODE 0  /**< Default ICMP code: 0 (standard)            */

# 默认的 ICMPv6 消息类型：回显请求
#define DEFAULT_ICMPv6_TYPE 128 /**< Default ICMPv6 message: Echo Request    */

# 默认的 ICMPv6 代码：0（标准）
#define DEFAULT_ICMPv6_CODE 0   /**< Default ICMPv6 code: 0 (standard)       */

# 默认的 ARP 操作：ARP 请求
#define DEFAULT_ARP_OP 1   /**< Default ARP operation: OP_ARP_REQUEST      */

# UDP 和 TCP 负载的最大长度
#define MAX_PAYLOAD_ALLOWED 65400

# 在 GNU/Linux 2.6.24 上测试，当使用环回接口时，整个 IP 数据包的长度超过 16436，或者当使用普通网络接口时，超过 1500 时，内核会报错“消息太长”。这显然是由配置的 MTU 引起的。因此，尽管我们允许用户指定最多 MAX_PAYLOAD_ALLOWED 字节的负载，但是当我们生成随机负载时，我们将我们的限制设置为 1500-20-20=1460 字节。让我们保守一点，认为 IP 数据包有 40 字节的选项，TCP 有 20 字节。因此最大长度应该是 1500-60-40 = 1400。
#define MAX_RANDOM_PAYLOAD  1400
#define MAX_RECOMMENDED_PAYLOAD 1400

# 在 resolveChached() 和 gethostbynameCached() 中缓存的主机数
#define MAX_CACHED_HOSTS 512
# 缓存的主机名的最大长度
#define MAX_CACHED_HOSTNAME_LEN 512

# 9929 是质数，尚未被 IANA 分配
// 定义默认的回显端口号
#define DEFAULT_ECHO_PORT 9929

/* 回显服务器尝试在回显网络数据包之前将应用层数据清零。
 * 但是，有时我们可能无法成功解析给定的数据包（决定数据包是否包含应用程序数据），
 * 因此这个定义指定了在这种情况下服务器不清零的数据包的字节数。
 * 40字节允许IPv4+TCP，一个IPv6头部，一个IPv4+UDP+12字节的有效载荷等。
 * 在UDP的情况下，前12个数据字节将被泄露。
 * 但是，我们应该能够解析简单的IPv4-UDP数据包而不会出现问题，因此这种情况不应该发生。
 * 我们期望在接收到的数据包非常奇怪时使用这个常量（例如，隧道流量，我们不理解的协议等）。
 * 40字节是在丢弃数据包但提供对回显服务器的数据泄露的完全保护之间的折衷，
 * 并在攻击者能够欺骗回显服务器回显并非由他发起的数据包时提供一些灵活性的风险。
 */
#define PAYLOAD_ECHO_BYTES_IN_DOUBT 40

// 定义无限的套接字超时时间
#define NSOCK_INFINITE -1

/* nping.cc共享函数的原型 */
char *getBPFFilterString();

#endif
```