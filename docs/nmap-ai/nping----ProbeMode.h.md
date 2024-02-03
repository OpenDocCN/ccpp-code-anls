# `nmap\nping\ProbeMode.h`

```cpp
// 如果未定义 __PROBEMODE_H__，则定义 __PROBEMODE_H__ 为 1
#ifndef __PROBEMODE_H__
#define __PROBEMODE_H__ 1

// 包含必要的头文件
#include "nping.h"
#include "nsock.h"
#include "NpingTarget.h"
#include "utils_net.h"
#include "utils.h"

// 定义不同类型的数据包
#define PKT_TYPE_TCP_CONNECT  1
#define PKT_TYPE_UDP_NORMAL   2
#define PKT_TYPE_TCP_RAW      3
#define PKT_TYPE_UDP_RAW      4
#define PKT_TYPE_ICMP_RAW     5
#define PKT_TYPE_ARP_RAW      6

/* sendpkt 结构体是 normalProbeMode() 函数传递给 nsock 事件处理程序的数据，
 * 它包含了发送一个探测包所需的必要信息。*/
typedef struct sendpkt{
    int type;       // 数据包类型
    u8 *pkt;        // 数据包内容
    int pktLen;     // 数据包长度
    int rawfd;      // 原始套接字文件描述符
    u32 seq;        // 序列号
    NpingTarget *target; // 目标地址
    u16 dstport;    // 目标端口
}sendpkt_t;

// ProbeMode 类
class ProbeMode  {

    private:

        nsock_pool nsp;        /**< 内部 Nsock 池 */
        bool nsock_init;       /**< 如果 nsock 池已初始化，则为 true */
        // 默认构造函数
        ProbeMode();
        // 析构函数
        ~ProbeMode();
        // 重置函数
        void reset();
        // 初始化网络套接字
        int init_nsock();
        // 启动函数
        int start();
        // 清理函数
        int cleanup();
        // 获取网络套接字池
        nsock_pool getNsockPool();

        // 创建 IPv4 数据包
        static int createIPv4(IPv4Header *i, PacketElement *next_element, const char *next_proto, NpingTarget *target);
        // 创建 IPv6 数据包
        static int createIPv6(IPv6Header *i, PacketElement *next_element, const char *next_proto, NpingTarget *target);
        // 通过套接字发送 IPv6 数据包
        static int doIPv6ThroughSocket(int rawfd);
        // 填充数据包（通用）
        static int fillPacket(NpingTarget *target, u16 port, u8 *buff, int bufflen, int *filledlen, int rawfd);
        // 填充 TCP 数据包
        static int fillPacketTCP(NpingTarget *target, u16 port, u8 *buff, int bufflen, int *filledlen, int rawfd);
        // 填充 UDP 数据包
        static int fillPacketUDP(NpingTarget *target, u16 port, u8 *buff, int bufflen, int *filledlen, int rawfd);
        // 填充 ICMP 数据包
        static int fillPacketICMP(NpingTarget *target, u8 *buff, int bufflen, int *filledlen, int rawfd);
        // 填充 ARP 数据包
        static int fillPacketARP(NpingTarget *target, u8 *buff, int bufflen, int *filledlen, int rawfd);
        // 获取 BPF 过滤器字符串
        static char *getBPFFilterString();
        // nping 事件处理函数
        static void probe_nping_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
        // 延迟输出处理函数
        static void probe_delayed_output_handler(nsock_pool nsp, nsock_event nse, void *mydata);
        // TCP 连接事件处理函数
        static void probe_tcpconnect_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
        // UDP 非特权事件处理函数
        static void probe_udpunpriv_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
}; /* End of class ProbeMode */

/* Handler wrappers */
// 定义 nping_event_handler 函数，处理 nsock_pool 和 nsock_event，带有一个指向 void 的参数
void nping_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义 tcpconnect_event_handler 函数，处理 nsock_pool 和 nsock_event，带有一个指向 void 的参数
void tcpconnect_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义 udpunpriv_event_handler 函数，处理 nsock_pool 和 nsock_event，带有一个指向 void 的参数
void udpunpriv_event_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义 delayed_output_handler 函数，处理 nsock_pool 和 nsock_event，带有一个指向 void 的参数
void delayed_output_handler(nsock_pool nsp, nsock_event nse, void *arg);

#endif /* __PROBEMODE_H__ */
```