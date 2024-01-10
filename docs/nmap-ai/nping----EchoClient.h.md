# `nmap\nping\EchoClient.h`

```
#ifndef __ECHOCLIENT_H__
#define __ECHOCLIENT_H__ 1

#include "nping.h"              // 包含 nping.h 头文件
#include "NpingTarget.h"        // 包含 NpingTarget.h 头文件
#include "NEPContext.h"         // 包含 NEPContext.h 头文件
#include "ProbeMode.h"          // 包含 ProbeMode.h 头文件

#define ECHO_CONNECT_TIMEOUT (10*1000) /* 10 Seconds */  // 定义 ECHO 连接超时时间为 10 秒
#define ECHO_READ_TIMEOUT    (10*1000)  // 定义 ECHO 读取超时时间为 10 秒
#define ECHO_WRITE_TIMEOUT   (10*1000)  // 定义 ECHO 写入超时时间为 10 秒

/* Max number of bytes that are supplied as data for the PAYLOAD_MAGIC specifier */
#define NEP_PAYLOADMAGIC_MAX_BYTES 8   // 定义 PAYLOAD_MAGIC 指定器所支持的最大字节数为 8

class EchoClient  {

    private:

        /* Attributes */
        nsock_pool nsp;               /**< Nsock pool (shared with ProbeMode) */  // Nsock 池（与 ProbeMode 共享）
        nsock_iod nsi;                /**< IOD for the side-channel tcp socket*/    // 用于侧通道 TCP 套接字的 IOD
        struct sockaddr_in srvaddr4;  /**< Server's IPv4 address */                // 服务器的 IPv4 地址
        struct sockaddr_in6 srvaddr6; /**< Server's IPv6 address */                // 服务器的 IPv6 地址
        int af;                       /**< Address family (AF_INET or AF_INET6)*/  // 地址族（AF_INET 或 AF_INET6）
        NEPContext ctx;               // NEP 上下文
        ProbeMode probe;              // 探测模式
        u8 lasthdr[MAX_NEP_PACKET_LENGTH];  // 最后一个头部
        size_t readbytes;              // 读取的字节数

        /* Methods */
        int nep_connect(NpingTarget *target, u16 port);  // NEP 连接方法
        int nep_handshake();                            // NEP 握手方法
        int nep_send_packet_spec();                      // NEP 发送数据包方法
        int nep_recv_ready();                            // NEP 接收准备方法
        int nep_recv_echo(u8 *packet, size_t packetlen);  // NEP 接收回显方法

        int parse_hs_server(u8 *pkt, size_t pktlen);     // 解析服务器握手方法
        int parse_hs_final(u8 *pkt, size_t pktlen);      // 解析最终握手方法
        int parse_ready(u8 *pkt, size_t pktlen);         // 解析准备方法
        int parse_echo(u8 *pkt, size_t pktlen);          // 解析回显方法
        int parse_error(u8 *pkt, size_t pktlen);         // 解析错误方法

        int generate_hs_client(EchoHeader *h);           // 生成客户端握手方法
        int generate_packet_spec(EchoHeader *h);        // 生成数据包方法
    // 公有成员函数声明部分

    // 默认构造函数
    EchoClient();

    // 析构函数
    ~EchoClient();

    // 重置函数
    void reset();

    // 启动函数，传入目标和端口号
    int start(NpingTarget *target, u16 port);

    // 清理函数
    int cleanup();

    // 接收到回显数据包的处理函数
    int nep_echoed_packet_handler(nsock_pool nsp, nsock_event nse, void *arg);

    // 接收到标准头部的处理函数
    int nep_recv_std_header_handler(nsock_pool nsp, nsock_event nse, void *arg);

    // 接收到握手服务器的处理函数
    int nep_recv_hs_server_handler(nsock_pool nsp, nsock_event nse, void *arg);

    // 接收到最终握手的处理函数
    int nep_recv_hs_final_handler(nsock_pool nsp, nsock_event nse, void *arg);

    // 接收到准备就绪的处理函数
    int nep_recv_ready_handler(nsock_pool nsp, nsock_event nse, void *arg);
}; /* End of class EchoClient */

/* Handler wrappers */
// 定义处理接收到回显数据包的函数
void echoed_packet_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理接收到标准头部数据的函数
void recv_std_header_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理连接完成的函数
void connect_done_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理写操作完成的函数
void write_done_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理接收到握手服务器数据的函数
void recv_hs_server_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理接收到最终握手数据的函数
void recv_hs_final_handler(nsock_pool nsp, nsock_event nse, void *arg);
// 定义处理接收准备就绪的函数
void recv_ready_handler(nsock_pool nsp, nsock_event nse, void *arg);

#endif /* __ECHOCLIENT_H__ */
```