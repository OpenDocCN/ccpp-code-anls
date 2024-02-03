# `nmap\libnetutil\ICMPv4Header.h`

```cpp
/* 这段代码最初是 Nping 工具的一部分。*/

#ifndef ICMPv4HEADER_H
#define ICMPv4HEADER_H 1

#include "ICMPHeader.h"

/* ICMP 类型和代码。这些定义最初来自 Slirp 1.0 源文件 ip_icmp.h http://slirp.sourceforge.net/（BSD 许可），然后部分修改为 Nping。*/
#define ICMP_ECHOREPLY               0     /* 回显应答 */
#define ICMP_UNREACH                 3     /* 目的地不可达 */
#define    ICMP_UNREACH_NET            0   /*  --> 错误的网络 */
#define    ICMP_UNREACH_HOST           1   /*  --> 错误的主机 */
#define    ICMP_UNREACH_PROTOCOL       2   /*  --> 错误的协议 */
#define    ICMP_UNREACH_PORT           3   /*  --> 错误的端口 */
#define    ICMP_UNREACH_NEEDFRAG       4   /*  --> DF 标志导致数据包丢失 */
#define    ICMP_UNREACH_SRCFAIL        5   /*  --> 源路由失败 */
#define    ICMP_UNREACH_NET_UNKNOWN    6   /*  --> 未知网络 */
#define    ICMP_UNREACH_HOST_UNKNOWN   7   /*  --> 未知主机 */
#define    ICMP_UNREACH_ISOLATED       8   /*  --> 源主机被隔离 */
#define    ICMP_UNREACH_NET_PROHIB     9   /*  --> 禁止访问网络 */
#define    ICMP_UNREACH_HOST_PROHIB    10  /*  --> 禁止访问主机 */
#define    ICMP_UNREACH_TOSNET         11  /*  --> 网络的 TOS 错误 */
#define    ICMP_UNREACH_TOSHOST        12  /*  --> 主机的 TOS 错误 */
#define    ICMP_UNREACH_COMM_PROHIB    13  /*  --> 禁止通信 */
#define    ICMP_UNREACH_HOSTPRECEDENCE 14  /*  --> 主机优先级违规 */
#define    ICMP_UNREACH_PRECCUTOFF     15  /*  --> 优先级截断 */
#define ICMP_SOURCEQUENCH            4     /* 源端抑制 */
#define ICMP_REDIRECT                5     /* 重定向 */
# 定义 ICMP 重定向类型，用于指示重定向的对象是网络
#define    ICMP_REDIRECT_NET           0   /*  --> For the network           */
# 定义 ICMP 重定向类型，用于指示重定向的对象是主机
#define    ICMP_REDIRECT_HOST          1   /*  --> For the host              */
# 定义 ICMP 重定向类型，用于指示重定向的对象是 TOS 和网络
#define    ICMP_REDIRECT_TOSNET        2   /*  --> For the TOS and network   */
# 定义 ICMP 重定向类型，用于指示重定向的对象是 TOS 和主机
#define    ICMP_REDIRECT_TOSHOST       3   /*  --> For the TOS and host      */
# 定义 ICMP 回显请求类型
#define ICMP_ECHO                    8     /* Echo request                   */
# 定义 ICMP 路由器通告类型
#define ICMP_ROUTERADVERT            9     /* Router advertisement           */
# 定义移动 IP 代理使用的 ICMP 路由器通告类型
#define    ICMP_ROUTERADVERT_MOBILE    16  /* Used by mobile IP agents       */
# 定义 ICMP 路由器请求类型
#define ICMP_ROUTERSOLICIT           10    /* Router solicitation            */
# 定义 ICMP 时间超时类型
#define ICMP_TIMXCEED                11    /* Time exceeded:                 */
# 定义 ICMP 时间超时类型，用于指示 TTL==0 在传输过程中
#define    ICMP_TIMXCEED_INTRANS       0   /*  --> TTL==0 in transit         */
# 定义 ICMP 时间超时类型，用于指示 TTL==0 在重组过程中
#define    ICMP_TIMXCEED_REASS         1   /*  --> TTL==0 in reassembly      */
# 定义 ICMP 参数问题类型
#define ICMP_PARAMPROB               12    /* Parameter problem              */
# 定义 ICMP 参数问题类型，用于指示指针显示的问题
#define    ICMM_PARAMPROB_POINTER      0   /*  --> Pointer shows the problem */
# 定义 ICMP 参数问题类型，用于指示选项缺失
#define    ICMP_PARAMPROB_OPTABSENT    1   /*  --> Option missing            */
# 定义 ICMP 参数问题类型，用于指示坏数据报长度
#define    ICMP_PARAMPROB_BADLEN       2   /*  --> Bad datagram length       */
# 定义 ICMP 时间戳请求类型
#define ICMP_TSTAMP                  13    /* Timestamp request              */
# 定义 ICMP 时间戳回复类型
#define ICMP_TSTAMPREPLY             14    /* Timestamp reply                */
# 定义 ICMP 信息请求类型
#define ICMP_INFO                    15    /* Information request            */
# 定义 ICMP 信息回复类型
#define ICMP_INFOREPLY               16    /* Information reply              */
# 定义 ICMP 地址掩码请求类型
#define ICMP_MASK                    17    /* Address mask request           */
# 定义 ICMP 地址掩码回复类型
#define ICMP_MASKREPLY               18    /* Address mask reply             */
# 定义 ICMP 跟踪路由类型
#define ICMP_TRACEROUTE              30    /* Traceroute                     */
# 定义 ICMP 跟踪路由成功类型，用于指示数据报发送到下一个路由器
#define    ICMP_TRACEROUTE_SUCCESS     0   /*  --> Dgram sent to next router */
# 定义 ICMP 跟踪路由丢弃类型，用于指示数据报被丢弃
#define    ICMP_TRACEROUTE_DROPPED     1   /*  --> Dgram was dropped         */
# 定义 ICMP 域名请求类型
#define ICMP_DOMAINNAME              37    /* Domain name request            */
// 定义 ICMP 域名回复类型的编号
#define ICMP_DOMAINNAMEREPLY         38    /* Domain name reply              */
// 定义 ICMP 安全失败类型的编号
#define ICMP_SECURITYFAILURES        40    /* Security failures              */


// 定义 ICMP 标准头部长度
#define ICMP_STD_HEADER_LEN 8
// 定义 ICMP 最大有效载荷长度
#define ICMP_MAX_PAYLOAD_LEN 1500
// 定义最大路由器通告条目数
#define MAX_ROUTER_ADVERT_ENTRIES (((ICMP_MAX_PAYLOAD_LEN-4)/8)-1)


// 定义 ICMPv4Header 类，继承自 ICMPHeader 类
class ICMPv4Header : public ICMPHeader {

}; /* End of class ICMPv4Header */

#endif
```