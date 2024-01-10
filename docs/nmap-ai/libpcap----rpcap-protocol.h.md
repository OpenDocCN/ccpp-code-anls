# `nmap\libpcap\rpcap-protocol.h`

```
/*
 * 版权声明，声明了代码的版权和使用条件
 * 版权所有，禁止未经许可的再分发和修改
 * 在源代码的再分发中必须保留版权声明、条件列表和下面的免责声明
 * 在二进制形式的再分发中必须在文档和/或其他提供的材料中重现版权声明、条件列表和下面的免责声明
 * 不得使用 Politecnico di Torino、CACE Technologies 或其贡献者的名称来认可或推广从本软件派生的产品，除非有特定的事先书面许可
 * 版权持有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、附带、特殊、惩罚性或后果性损害，包括但不限于替代商品或服务的采购、使用损失、数据或利润损失或业务中断，版权持有者或贡献者均不承担任何责任
 */
#ifndef __RPCAP_PROTOCOL_H__
#define __RPCAP_PROTOCOL_H__

#define RPCAP_DEFAULT_NETPORT "2002" /* RPCAP守护进程等待连接的默认端口 */
/* 在主动模式下，客户端工作站等待连接的默认端口 */
# 定义默认的RPCAP活动网络端口为"2003"
#define RPCAP_DEFAULT_NETPORT_ACTIVE "2003"
# 定义RPCAP守护进程绑定的默认网络地址为空字符串
#define RPCAP_DEFAULT_NETADDR ""    /* Default network address on which the RPCAP daemon binds to. */
/*
 * 协议的最小和最大支持版本。
 *
 * 如果添加了新的消息类型，协议版本必须更改，
 * 这样客户端就可以根据协商的协议版本知道可以向服务器发送哪些消息。
 *
 * 如果更改了现有消息类型的格式，协议版本必须更改，
 * 这样每一方就可以根据协商的协议版本知道应该使用什么格式。
 *
 * RPCAP_MSG_ERROR 格式不得更改，因为它用于报告“不正确的版本号”错误，
 * 如果格式更改，发送消息的一方可能不知道接收方会理解哪些版本，
 * 或者可能知道他们支持的版本（他们发送的版本号）但可能不知道该版本中消息的格式。
 *
 * 其他消息版本不应更改，因为这会使解释消息的过程变得复杂，使其依赖于版本。
 * 最好是引入一个新的具有新格式的消息。
 *
 * 版本协商是作为认证过程的一部分进行的：
 *
 * 客户端发送一个带有版本号为0的认证请求。所有服务器必须接受版本号为0的认证请求，即使它们不支持版本0进行任何其他请求。
 *
 * 服务器尝试对客户端进行认证。如果成功，只支持版本0的旧服务器将发送一个没有有效负载的认证回复。可能支持其他版本的新服务器将发送一个带有有效负载的认证回复，其中包含其支持的最小和最大版本。
 *
 * 客户端尝试找到其支持版本范围和服务器支持版本中都存在的最大版本号。如果失败，就放弃；否则，就使用该版本。
 */
#define RPCAP_MIN_VERSION 0
#define RPCAP_MAX_VERSION 0
/*
 * Version numbers are unsigned, so if RPCAP_MIN_VERSION is 0, they
 * are >= the minimum version, by definition; don't check against
 * RPCAP_MIN_VERSION, as you may get compiler warnings that the
 * comparison will always succeed.
 */
#if RPCAP_MIN_VERSION == 0
#define RPCAP_VERSION_IS_SUPPORTED(v)    \
    ((v) <= RPCAP_MAX_VERSION)
#else
#define RPCAP_VERSION_IS_SUPPORTED(v)    \
    ((v) >= RPCAP_MIN_VERSION && (v) <= RPCAP_MAX_VERSION)
#endif

/*
 * Separators used for the host list.
 *
 * It is used:
 * - by the rpcapd daemon, when you types a list of allowed connecting hosts
 * - by the rpcap client in active mode, when the client waits for incoming
 * connections from other hosts
 */
#define RPCAP_HOSTLIST_SEP " ,;\n\r"

/*********************************************************
 *                                                       *
 * Protocol messages formats                             *
 *                                                       *
 *********************************************************/
/*
 * WARNING: This file defines some structures that are used to transfer
 * data on the network.
 * Note that your compiler MUST not insert padding into these structures
 * for better alignment.
 * These structures have been created in order to be correctly aligned to
 * a 32-bit boundary, but be careful in any case.
 *
 * The layout of these structures MUST not be changed.  If a packet
 * format is different in different versions of the protocol, versions
 * of the structure should be provided for all the different versions or
 * version ranges (if more than one version of the protocol has the same
 * layout) that we support.
 */

/*
 * WARNING: These typedefs MUST be of a specific size.
 * You might have to change them on your platform.
 *
 * XXX - use the C99 types?  Microsoft's newer versions of Visual Studio
 * support them.
 */
#ifndef __HAIKU__
typedef unsigned char uint8;    /* 8-bit unsigned integer */
# 定义 16 位无符号整数类型 uint16
typedef unsigned short uint16;    /* 16-bit unsigned integer */
# 定义 32 位无符号整数类型 uint32
typedef unsigned int uint32;    /* 32-bit unsigned integer */
# 定义 32 位有符号整数类型 int32
typedef int int32;        /* 32-bit signed integer */
#endif

# 所有 RPCAP 消息的通用头部
struct rpcap_header
{
    uint8 ver;    /* RPCAP 版本号 */
    uint8 type;    /* RPCAP 消息类型（错误，findalldevs，...） */
    uint16 value;    /* 消息相关值（不一定总是使用） */
    uint32 plen;    /* 此 RPCAP 消息的有效载荷长度 */
};

# 出现在身份验证回复末尾的数据格式，给出服务器支持的协议的最小和最大版本
# 旧服务器不提供此信息；它们仅支持版本 0
struct rpcap_authreply
{
    uint8 minvers;            /* 支持的最小版本 */
    uint8 maxvers;            /* 支持的最大版本 */
    uint8 pad[2];            /* 填充到 4 字节边界 **/
    uint32 byte_order_magic;    /* RPCAP_BYTE_ORDER_MAGIC，以服务器字节顺序 */
};

# 这与 pcap 文件魔术数字之间的任何相似之处纯属巧合，相信我
#define RPCAP_BYTE_ORDER_MAGIC        0xa1b2c3d4U
#define RPCAP_BYTE_ORDER_MAGIC_SWAPPED    0xd4c3b2a1U

# 旧版本的身份验证回复格式，没有字节顺序指示和填充
struct rpcap_authreply_old
{
    uint8 minvers;    /* 支持的最小版本 */
    uint8 maxvers;    /* 支持的最大版本 */
};

# 接口描述消息的格式（findalldevs 命令）
struct rpcap_findalldevs_if
{
    uint16 namelen;    /* 接口名称的长度 */
    uint16 desclen;    /* 接口描述的长度 */
    uint32 flags;    /* 接口标志 */
    uint16 naddr;    /* 地址数量 */
    uint16 dummy;    /* 必须为零 */
};
# 定义一个结构体，表示在网络上传输的地址格式
# 不要使用 struct sockaddr_storage，因为其布局是依赖于机器的
# RFC 2553给出了两个示例布局，都是128字节长，都在8字节边界上对齐，都在地址数据之前有2个字节
# 但是，一个在开头有一个2字节的地址族值，另一个有一个1字节的地址长度值和一个1字节的地址族值；这反映了原始的BSD sockaddr结构在添加对可变长度OSI网络层地址支持时，从2字节的地址族值改为1字节的地址长度值和1字节的地址族值
# 此外，Solaris的struct sockaddr_storage是256字节长
# 这个结构应该在8字节边界上对齐；消息头是8字节长，所以我们不需要做任何事情来确保它在数据包内部是对齐的，所以我们只定义它为128字节长，带有一个2字节的地址族。这样，它和Windows上的sockaddr_storage大小相同，并且看起来像一个旧的Windows客户端期望的样子
# 此外，不要使用主机的AF_值作为地址的值，因为AF_INET6的值是依赖于机器的。我们使用Windows的值，这样它看起来像一个旧的Windows客户端期望的样子
# （Windows客户端是作为*pcap的标准部分分发的唯一一个；UNIX客户端可能是由用户或管理员从源代码构建的，因此他们更有能力升级旧的客户端。因此，我们尝试使传输的内容看起来像来自Windows服务器的内容）
struct rpcap_sockaddr
{
    uint16    family;            /* 地址族 */
    char    data[128-2];        /* 数据 */
};
/*
 * Format of an IPv4 address as sent over the wire.
 */
#define RPCAP_AF_INET    2        /* Value on all OSes */
struct rpcap_sockaddr_in
{
    uint16    family;            /* Address family */
    uint16    port;            /* Port number */
    uint32    addr;            /* IPv4 address */
    uint8    zero[8];        /* Padding */
};

/*
 * Format of an IPv6 address as sent over the wire.
 */
#define RPCAP_AF_INET6    23        /* Value on Windows */
struct rpcap_sockaddr_in6
{
    uint16    family;            /* Address family */
    uint16    port;            /* Port number */
    uint32    flowinfo;        /* IPv6 flow information */
    uint8    addr[16];        /* IPv6 address */
    uint32    scope_id;        /* Scope zone index */
};

/* Format of the message for the address listing (findalldevs command) */
struct rpcap_findalldevs_ifaddr
{
    struct rpcap_sockaddr addr;        /* Network address */
    struct rpcap_sockaddr netmask;        /* Netmask for that address */
    struct rpcap_sockaddr broadaddr;    /* Broadcast address for that address */
    struct rpcap_sockaddr dstaddr;        /* P2P destination address for that address */
};

/*
 * \brief Format of the message of the connection opening reply (open command).
 *
 * This structure transfers over the network some of the values useful on the client side.
 */
struct rpcap_openreply
{
    int32 linktype;    /* Link type */
    int32 tzoff;    /* Timezone offset - not used by newer clients */
};

/* Format of the message that starts a remote capture (startcap command) */
struct rpcap_startcapreq
{
    uint32 snaplen;        /* Length of the snapshot (number of bytes to capture for each packet) */
    uint32 read_timeout;    /* Read timeout in milliseconds */
    uint16 flags;        /* Flags (see RPCAP_STARTCAPREQ_FLAG_xxx) */
    uint16 portdata;    /* Network port on which the client is waiting at (if 'serveropen') */
};
/* Format of the reply message that devoted to start a remote capture (startcap reply command) */
struct rpcap_startcapreply
{
    int32 bufsize;        /* Size of the user buffer allocated by WinPcap; it can be different from the one we chose */
    uint16 portdata;    /* Network port on which the server is waiting at (passive mode only) */
    uint16 dummy;        /* Must be zero */
};

/*
 * \brief Format of the header which encapsulates captured packets when transmitted on the network.
 *
 * This message requires the general header as well, since we want to be able to exchange
 * more information across the network in the future (for example statistics, and kind like that).
 */
struct rpcap_pkthdr
{
    /*
     * This protocol needs to be updated with a new version before
     * 2038-01-19 03:14:07 UTC.
     */
    uint32 timestamp_sec;    /* 'struct timeval' compatible, it represents the 'tv_sec' field */
    uint32 timestamp_usec;    /* 'struct timeval' compatible, it represents the 'tv_usec' field */
    uint32 caplen;        /* Length of portion present in the capture */
    uint32 len;        /* Real length of this packet (off wire) */
    uint32 npkt;        /* Ordinal number of the packet (i.e. the first one captured has '1', the second one '2', etc) */
};

/* General header used for the pcap_setfilter() command; keeps just the number of BPF instructions */
struct rpcap_filter
{
    uint16 filtertype;    /* type of the filter transferred (BPF instructions, ...) */
    uint16 dummy;        /* Must be zero */
    uint32 nitems;        /* Number of items contained into the filter (e.g. BPF instructions for BPF filters) */
};

/* Structure that keeps a single BPF instruction; it is repeated 'ninsn' times according to the 'rpcap_filterbpf' header */
struct rpcap_filterbpf_insn
{
    uint16 code;    /* opcode of the instruction */
    uint8 jt;    /* relative offset to jump to in case of 'true' */
    # 用于存储在条件为假时跳转的相对偏移量
    uint8 jf;    /* relative offset to jump to in case of 'false' */
    # 用于存储指令相关的数值
    int32 k;    /* instruction-dependent value */
/* 结构体，保存远程主机身份验证所需的数据 */
struct rpcap_auth
{
    uint16 type;    /* 身份验证类型 */
    uint16 dummy;    /* 必须为零 */
    uint16 slen1;    /* 第一个身份验证项的长度（例如用户名） */
    uint16 slen2;    /* 第二个身份验证项的长度（例如密码） */
};

/* 结构体，保存有关捕获的数据包数量、丢弃的数据包数量等的统计信息 */
struct rpcap_stats
{
    uint32 ifrecv;        /* 内核过滤器接收的数据包数量（即 pcap_stats.ps_recv） */
    uint32 ifdrop;        /* 网络接口丢弃的数据包数量（例如缓冲区不足）（即 pcap_stats.ps_ifdrop） */
    uint32 krnldrop;    /* 内核过滤器丢弃的数据包数量（即 pcap_stats.ps_drop） */
    uint32 svrcapt;        /* RPCAP守护程序捕获并发送到网络上的数据包数量 */
};

/* 结构体，设置采样参数所需的数据 */
struct rpcap_sampling
{
    uint8 method;    /* 采样方法 */
    uint8 dummy1;    /* 必须为零 */
    uint16 dummy2;    /* 必须为零 */
    uint32 value;    /* 与采样方法相关的参数 */
};

/*
 * 消息字段编码。
 *
 * 这些值用于在网络上发送的消息中，并且不得更改。
 */
#define RPCAP_MSG_IS_REPLY        0x080    /* 表示回复的标志 */

#define RPCAP_MSG_ERROR            0x01    /* 保留错误通知的消息 */
#define RPCAP_MSG_FINDALLIF_REQ        0x02    /* 列出所有远程接口的请求 */
#define RPCAP_MSG_OPEN_REQ        0x03    /* 请求打开远程设备 */
#define RPCAP_MSG_STARTCAP_REQ        0x04    /* 请求在远程设备上开始捕获 */
#define RPCAP_MSG_UPDATEFILTER_REQ    0x05    /* 将编译后的过滤器发送到远程设备 */
#define RPCAP_MSG_CLOSE            0x06    /* 关闭与远程对等体的连接 */
# 定义RPCAP_MSG_PACKET为0x07，表示这是一个“数据”消息，携带网络数据包
#define RPCAP_MSG_PACKET        0x07    

# 定义RPCAP_MSG_AUTH_REQ为0x08，表示保持身份验证参数的消息
#define RPCAP_MSG_AUTH_REQ        0x08    

# 定义RPCAP_MSG_STATS_REQ为0x09，表示需要获取网络统计信息
#define RPCAP_MSG_STATS_REQ        0x09    

# 定义RPCAP_MSG_ENDCAP_REQ为0x0A，表示停止当前捕获，保持设备打开
#define RPCAP_MSG_ENDCAP_REQ        0x0A    

# 定义RPCAP_MSG_SETSAMPLING_REQ为0x0B，表示设置采样参数
#define RPCAP_MSG_SETSAMPLING_REQ    0x0B    

# 定义RPCAP_MSG_FINDALLIF_REPLY为RPCAP_MSG_FINDALLIF_REQ | RPCAP_MSG_IS_REPLY，表示保持所有远程接口的列表
#define RPCAP_MSG_FINDALLIF_REPLY    (RPCAP_MSG_FINDALLIF_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_OPEN_REPLY为RPCAP_MSG_OPEN_REQ | RPCAP_MSG_IS_REPLY，表示远程设备已正确打开
#define RPCAP_MSG_OPEN_REPLY        (RPCAP_MSG_OPEN_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_STARTCAP_REPLY为RPCAP_MSG_STARTCAP_REQ | RPCAP_MSG_IS_REPLY，表示捕获已正确开始
#define RPCAP_MSG_STARTCAP_REPLY    (RPCAP_MSG_STARTCAP_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_UPDATEFILTER_REPLY为RPCAP_MSG_UPDATEFILTER_REQ | RPCAP_MSG_IS_REPLY，表示筛选已正确应用在远程设备上
#define RPCAP_MSG_UPDATEFILTER_REPLY    (RPCAP_MSG_UPDATEFILTER_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_AUTH_REPLY为RPCAP_MSG_AUTH_REQ | RPCAP_MSG_IS_REPLY，表示发送一条消息，表示“ok，授权成功”
#define RPCAP_MSG_AUTH_REPLY        (RPCAP_MSG_AUTH_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_STATS_REPLY为RPCAP_MSG_STATS_REQ | RPCAP_MSG_IS_REPLY，表示保持网络统计信息的消息
#define RPCAP_MSG_STATS_REPLY        (RPCAP_MSG_STATS_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_ENDCAP_REPLY为RPCAP_MSG_ENDCAP_REQ | RPCAP_MSG_IS_REPLY，表示确认捕获已成功停止
#define RPCAP_MSG_ENDCAP_REPLY        (RPCAP_MSG_ENDCAP_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_MSG_SETSAMPLING_REPLY为RPCAP_MSG_SETSAMPLING_REQ | RPCAP_MSG_IS_REPLY，表示确认捕获已成功停止
#define RPCAP_MSG_SETSAMPLING_REPLY    (RPCAP_MSG_SETSAMPLING_REQ | RPCAP_MSG_IS_REPLY)    

# 定义RPCAP_STARTCAPREQ_FLAG_PROMISC为0x00000001，表示启用混杂模式（默认：禁用）
#define RPCAP_STARTCAPREQ_FLAG_PROMISC        0x00000001    

# 定义RPCAP_STARTCAPREQ_FLAG_DGRAM为0x00000002，表示使用数据报（即UDP）连接进行数据流（默认：使用TCP）
#define RPCAP_STARTCAPREQ_FLAG_DGRAM        0x00000002    

# 定义RPCAP_STARTCAPREQ_FLAG_SERVEROPEN为0x00000004，表示服务器必须向客户端打开数据连接
#define RPCAP_STARTCAPREQ_FLAG_SERVEROPEN    0x00000004    
# 定义 RPCAP_STARTCAPREQ_FLAG_INBOUND 常量，表示仅捕获入站数据包（注意：在启用混杂模式时，该标志无效）
#define RPCAP_STARTCAPREQ_FLAG_INBOUND        0x00000008    /* Capture only inbound packets (take care: the flag has no effect with promiscuous enabled) */
# 定义 RPCAP_STARTCAPREQ_FLAG_OUTBOUND 常量，表示仅捕获出站数据包（注意：在启用混杂模式时，该标志无效）
#define RPCAP_STARTCAPREQ_FLAG_OUTBOUND        0x00000010    /* Capture only outbound packets (take care: the flag has no effect with promiscuous enabled) */

# 定义 RPCAP_UPDATEFILTER_BPF 常量，表示过滤器使用 BPF/NPF 语法编码
#define RPCAP_UPDATEFILTER_BPF 1            /* This code tells us that the filter is encoded with the BPF/NPF syntax */

# 网络错误代码
# 这些值用于在网络上发送的消息中，不得更改
#define PCAP_ERR_NETW            1    /* Network error */
#define PCAP_ERR_INITTIMEOUT        2    /* The RPCAP initial timeout has expired */
#define PCAP_ERR_AUTH            3    /* Generic authentication error */
#define PCAP_ERR_FINDALLIF        4    /* Generic findalldevs error */
#define PCAP_ERR_NOREMOTEIF        5    /* The findalldevs was ok, but the remote end had no interfaces to list */
#define PCAP_ERR_OPEN            6    /* Generic pcap_open error */
#define PCAP_ERR_UPDATEFILTER        7    /* Generic updatefilter error */
#define PCAP_ERR_GETSTATS        8    /* Generic pcap_stats error */
#define PCAP_ERR_READEX            9    /* Generic pcap_next_ex error */
#define PCAP_ERR_HOSTNOAUTH        10    /* The host is not authorized to connect to this server */
#define PCAP_ERR_REMOTEACCEPT        11    /* Generic pcap_remoteaccept error */
#define PCAP_ERR_STARTCAPTURE        12    /* Generic pcap_startcapture error */
#define PCAP_ERR_ENDCAPTURE        13    /* Generic pcap_endcapture error */
#define PCAP_ERR_RUNTIMETIMEOUT        14    /* The RPCAP run-time timeout has expired */
#define PCAP_ERR_SETSAMPLING        15    /* Error during the settings of sampling parameters */
#define PCAP_ERR_WRONGMSG        16    /* The other end endpoint sent a message which has not been recognized */
/* 定义错误代码，表示对端端点的版本号与我们不兼容 */
#define PCAP_ERR_WRONGVER        17    
/* 用户无法通过身份验证 */
#define PCAP_ERR_AUTH_FAILED        18    
/* 服务器要求使用 TLS 进行连接 */
#define PCAP_ERR_TLS_REQUIRED        19    
/* 不支持的身份验证类型 */
#define PCAP_ERR_AUTH_TYPE_NOTSUP    20    

/*
 * \brief 用于发送和接收数据包的套接字函数使用的缓冲区。
 * 如果您计划发送大于此值的消息，则必须增加它。
 */
#define RPCAP_NETBUF_SIZE 64000

/*********************************************************
 *                                                       *
 * rpcap 客户端和 rpcap 守护程序使用的例程              *
 *                                                       *
 *********************************************************/

#include "sockutils.h"
#include "sslutils.h"

/* 创建 rpcap 头部 */
extern void rpcap_createhdr(struct rpcap_header *header, uint8 ver, uint8 type, uint16 value, uint32 length);
/* 返回 rpcap 消息类型的字符串表示 */
extern const char *rpcap_msg_type_string(uint8 type);
/* 发送错误消息 */
extern int rpcap_senderror(SOCKET sock, SSL *ssl, uint8 ver, uint16 errcode, const char *error, char *errbuf);

#endif
```