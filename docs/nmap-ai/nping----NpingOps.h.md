# `nmap\nping\NpingOps.h`

```
#ifndef NPING_NPINGOPS_H
#define NPING_NPINGOPS_H

/* Probe Modes */
#define TCP_CONNECT   0xF1  // 定义 TCP 连接探测模式的标识
#define TCP           0xF2    // 定义 TCP 探测模式的标识
#define UDP           0xF3    // 定义 UDP 探测模式的标识
#define UDP_UNPRIV    0xF4    // 定义非特权 UDP 探测模式的标识
#define ICMP          0xF5    // 定义 ICMP 探测模式的标识
#define ARP           0xF6    // 定义 ARP 探测模式的标识

/* Roles */
#define ROLE_NORMAL 0x22    // 定义普通角色的标识
#define ROLE_CLIENT 0x44    // 定义客户端角色的标识
#define ROLE_SERVER 0x66    // 定义服务器角色的标识

/* Payload types */
#define PL_NONE 0x00    // 定义无载荷类型的标识
#define PL_HEX  0xAA    // 定义十六进制载荷类型的标识
#define PL_RAND 0xBB    // 定义随机载荷类型的标识
#define PL_FILE 0xCC    // 定义文件载荷类型的标识
#define PL_STRING 0xDD  // 定义字符串载荷类型的标识

/* Misc */
#define ARP_TYPE_REQUEST  0x01    // 定义 ARP 请求类型的标识
#define ARP_TYPE_REPLY    0x02    // 定义 ARP 回复类型的标识
#define RARP_TYPE_REQUEST 0x03    // 定义 RARP 请求类型的标识
#define RARP_TYPE_REPLY   0x04    // 定义 RARP 回复类型的标识

#define FLAG_CWR  0  /* Do not change these values because they */  // 定义标志位 CWR
#define FLAG_ECN  1  /* are used as indexes of an array         */  // 定义标志位 ECN
#define FLAG_URG  2  // 定义标志位 URG
#define FLAG_ACK  3  // 定义标志位 ACK
#define FLAG_PSH  4  // 定义标志位 PSH
#define FLAG_RST  5  // 定义标志位 RST
#define FLAG_SYN  6  // 定义标志位 SYN
#define FLAG_FIN  7  // 定义标志位 FIN

#define PACKET_SEND_NOPREF 1 /* These have been taken from NmapOps.h */  // 定义发送数据包的优先级
#define PACKET_SEND_ETH_WEAK 2  // 定义以弱以太网发送数据包的标识
#define PACKET_SEND_ETH_STRONG 4  // 定义以强以太网发送数据包的标识
#define PACKET_SEND_ETH 6  // 定义以以太网发送数据包的标识
#define PACKET_SEND_IP_WEAK 8  // 定义以弱 IP 发送数据包的标识
#define PACKET_SEND_IP_STRONG 16  // 定义以强 IP 发送数据包的标识
#define PACKET_SEND_IP 24  // 定义以 IP 发送数据包的标识

#define IP_VERSION_4 0x04  // 定义 IPv4 版本的标识
#define IP_VERSION_6 0x06  // 定义 IPv6 版本的标识

#define NOT_SET -1  // 定义未设置的标识
#define SET_RANDOM -2  // 定义随机设置的标识

#define MAX_ICMP_ADVERT_ENTRIES 128  // 定义 ICMP 广告条目的最大数量

#include "nping.h"
#include "global_structures.h"
#include "stats.h"
#include "NpingTargets.h"
#include <string>

class NpingOps {

  private:

    /* Probe modes */
    int mode;                 /* Probe mode (TCP,UDP,ICMP,ARP,RARP...) */  // 探测模式
    bool mode_set;  // 探测模式是否已设置
    bool traceroute;          /* Is traceroute mode enabled?           */  // 是否启用路由跟踪模式
    bool traceroute_set;  // 路由跟踪模式是否已设置

    /* Output */
    int vb;                   /* Current Verbosity level               */  // 当前详细程度
    bool vb_set;  // 详细程度是否已设置
    int dbg;                  /* Current Debugging level               */  // 当前调试级别
    bool dbg_set;  // 调试级别是否已设置
    bool show_sent_pkts;      /* Print packets sent by Nping?          */  // 是否打印 Nping 发送的数据包
    bool show_sent_pkts_set;  // 是否已设置打印 Nping 发送的数据包

    /* Operation and Performance */
    u32 pcount;               /* No of packets 2be sent to each target */  // 发送到每个目标的数据包数量
    bool pcount_set;  # 标记变量，表示 pcount 是否已经设置
    int sendpref;             /* Sending preference: eth or raw ip     */  # 发送偏好：以太网或原始 IP
    bool sendpref_set;  # 标记变量，表示 sendpref 是否已经设置
    bool send_eth;            /* True: send at raw ethernet level      */  # 是否以原始以太网级别发送
    bool send_eth_set;  # 标记变量，表示 send_eth 是否已经设置
    long delay;               /* Delay between each probe              */  # 每次探测之间的延迟
    bool delay_set;  # 标记变量，表示 delay 是否已经设置
    char device[MAX_DEV_LEN]; /* Network interface                     */  # 网络接口
    bool device_set;  # 标记变量，表示 device 是否已经设置
    bool spoofsource;         /* Did user request IP spoofing?         */  # 用户是否请求 IP 欺骗
    bool spoofsource_set;  # 标记变量，表示 spoofsource 是否已经设置
    char *bpf_filter_spec;    /* Custom, user-supplied BPF filter spec */  # 自定义的用户提供的 BPF 过滤器规范
    bool bpf_filter_spec_set;  # 标记变量，表示 bpf_filter_spec 是否已经设置
    int current_round;        /** Current round. Used in traceroute mode */  # 当前轮次，在 traceroute 模式中使用
    bool have_pcap;           /* True if we have access to libpcap     */  # 如果我们可以访问 libpcap，则为真
    bool disable_packet_capture; /* If false, no packets are captured  */  # 如果为假，则不捕获任何数据包
    bool disable_packet_capture_set;  # 标记变量，表示 disable_packet_capture 是否已经设置

    /* Privileges */
    bool isr00t;              /* True if current user has root privs   */  # 如果当前用户具有 root 权限，则为真

    /* Payloads */
    int payload_type;         /* Type of payload (RAND,HEX,FILE)       */  # 负载类型（RAND,HEX,FILE）
    bool payload_type_set;  # 标记变量，表示 payload_type 是否已经设置
    u8 *payload_buff;         /* Pointer 2buff with the actual payload */  # 指向实际负载的缓冲区指针
    bool payload_buff_set;  # 标记变量，表示 payload_buff 是否已经设置
    int payload_len;          /* Length of payload                     */  # 负载长度
    bool payload_len_set;  # 标记变量，表示 payload_len 是否已经设置

    /* Roles */
    int role;                 /* Nping's role: normal|client|server.  */  # Nping 的角色：normal|client|server
    bool role_set;  # 标记变量，表示 role 是否已经设置

    /* IPv4 */
    u8 ttl;                   /* IPv4 TTL / IPv6 Hop limit             */  # IPv4 TTL / IPv6 跳数限制
    bool ttl_set;  # 标记变量，表示 ttl 是否已经设置
    u8 tos;                   /* Type of service                       */  # 服务类型
    bool tos_set;  # 标记变量，表示 tos 是否已经设置
    u16 identification;       /* Identification field                  */  # 标识字段
    bool identification_set;  # 标记变量，表示 identification 是否已经设置
    bool mf;                  /* More fragments flag                   */  # 更多分片标志
    bool mf_set;  # 标记变量，表示 mf 是否已经设置
    bool df;                  /* Don't fragment flag                   */  # 不分片标志
    bool df_set;  # 标记变量，表示 df 是否已经设置
    bool rf;                  /* Reserved / Evil flag                  */  # 保留 / 恶意标志
    bool rf_set;  # 标记变量，表示 rf 是否已经设置
    # 自定义的 MTU 长度（用于 IP 分片）
    u32 mtu;
    # MTU 是否已设置的标志
    bool mtu_set;
    # 在 TCP/UDP 中生成无效的校验和
    bool badsum_ip;
    # 是否已设置生成无效的校验和的标志
    bool badsum_ip_set;
    # 要在所有数据包中使用的 IP 版本
    u8 ipversion;
    # 是否已设置 IP 版本的标志
    bool ipversion_set;
    # 源 IPv4 地址
    struct in_addr ipv4_src_address;
    # 是否已设置源 IPv4 地址的标志
    bool ipv4_src_address_set;
    # IP 选项
    char *ip_options;
    # 是否已设置 IP 选项的标志
    bool ip_options_set;

    # IPv6
    # 传输类别
    u8 ipv6_tclass;
    # 是否已设置传输类别的标志
    bool ipv6_tclass_set;
    # 流标签
    u32 ipv6_flowlabel;
    # 是否已设置流标签的标志
    bool ipv6_flowlabel_set;
    # 源 IPv6 地址
    struct in6_addr ipv6_src_address;
    # 是否已设置源 IPv6 地址的标志
    bool ipv6_src_address_set;

    # TCP / UDP
    # 目标端口数组的指针
    u16 *target_ports;
    # 目标端口的总数
    int tportcount;
    # 是否已设置目标端口的标志
    bool target_ports_set;
    # TCP/UDP 数据包的源端口
    u16 source_port;
    # 是否已设置源端口的标志
    bool source_port_set;
    # TCP 序列号
    u32 tcpseq;
    # 是否已设置 TCP 序列号的标志
    bool tcpseq_set;
    # TCP 确认号
    u32 tcpack;
    # 是否已设置 TCP 确认号的标志
    bool tcpack_set;
    # TCP 标志
    u8 tcpflags[8];
    # 是否已设置 TCP 标志的标志
    bool tcpflags_set;
    # TCP 窗口大小
    u16 tcpwin;
    # 是否已设置 TCP 窗口大小的标志
    bool tcpwin_set;
    # 是否生成无效的 TCP/UDP 校验和
    bool badsum;
    # 是否已设置生成无效的 TCP/UDP 校验和的标志
    bool badsum_set;

    # ICMP
    # ICMP 类型
    u8 icmp_type;
    # 是否已设置 ICMP 类型的标志
    bool icmp_type_set;
    # ICMP 代码
    u8 icmp_code;
    # 是否已设置 ICMP 代码的标志
    bool icmp_code_set;
    # 是否生成无效的 ICMP 校验和
    bool badsum_icmp;
    # 是否已设置生成无效的 ICMP 校验和的标志
    bool badsum_icmp_set;
    # ICMP 重定向地址
    struct in_addr icmp_redir_addr; /* ICMP Redirect Address */ /* ##TODO## Turn this into an IPAddress object */
    # 标记 ICMP 重定向地址是否已设置
    bool icmp_redir_addr_set;
    # ICMP 参数问题指针
    u8 icmp_paramprob_pnt;    /* ICMP Parameter Problem pointer        */
    # 标记 ICMP 参数问题指针是否已设置
    bool icmp_paramprob_pnt_set;
    # ICMP 路由通告生存时间
    u16 icmp_routeadv_ltime;  /* ICMP Router Advertisement lifetime    */
    # 标记 ICMP 路由通告生存时间是否已设置
    bool icmp_routeadv_ltime_set;
    # ICMP 消息标识符
    u16 icmp_id;              /* ICMP message identifier               */
    # 标记 ICMP 消息标识符是否已设置
    bool icmp_id_set;
    # ICMP 序列号
    u16 icmp_seq;             /* ICMP sequence number                  */
    # 标记 ICMP 序列号是否已设置
    bool icmp_seq_set;
    # ICMP 发起时间戳
    u32 icmp_orig_time;       /* ICMP originate timestamp              */
    # 标记 ICMP 发起时间戳是否已设置
    bool icmp_orig_time_set;
    # ICMP 接收时间戳
    u32 icmp_recv_time;       /* ICMP receive timestamp                */
    # 标记 ICMP 接收时间戳是否已设置
    bool icmp_recv_time_set;
    # ICMP 传输时间戳
    u32 icmp_trans_time;      /* ICMP transmit timestamp               */
    # 标记 ICMP 传输时间戳是否已设置
    bool icmp_trans_time_set;
    # ICMP 路由通告条目地址
    struct in_addr icmp_advert_entry_addr[MAX_ICMP_ADVERT_ENTRIES];
    # ICMP 路由通告条目优先级
    u32 icmp_advert_entry_pref[MAX_ICMP_ADVERT_ENTRIES];
    # ICMP 路由通告条目数量
    int icmp_advert_entry_count;
    # 标记 ICMP 路由通告条目是否已设置
    bool icmp_advert_entry_set;

    # 以太网
    # 源 MAC 地址
    u8 src_mac[6];            /* Source MAC address                    */
    # 标记源 MAC 地址是否已设置
    bool src_mac_set;
    # 目的 MAC 地址
    u8 dst_mac[6];            /* Destination MAC address               */
    # 标记目的 MAC 地址是否已设置
    bool dst_mac_set;
    # 以太网帧的 EtherType 字段
    u16 eth_type;             /* EtherType field of the Ethernet frame */
    # 标记 EtherType 字段是否已设置
    bool eth_type_set;

    # ARP/RARP
    # ARP 硬件类型
    u16 arp_htype;            /* ARP Hardware type                     */
    # 标记 ARP 硬件类型是否已设置
    bool arp_htype_set;
    # ARP 协议类型
    u16 arp_ptype;            /* ARP Protocol type                     */
    # 标记 ARP 协议类型是否已设置
    bool arp_ptype_set;
    # ARP 硬件地址长度
    u8 arp_hlen;              /* ARP Hardware address length           */
    # 标记 ARP 硬件地址长度是否已设置
    bool arp_hlen_set;
    # ARP 协议地址长度
    u8 arp_plen;              /* ARP protocol address length           */
    # 标记 ARP 协议地址长度是否已设置
    bool arp_plen_set;
    # ARP 操作码
    u16 arp_opcode;           /* ARP Operation code                    */
    # 标记 ARP 操作码是否已设置
    bool arp_opcode_set;
    # ARP 发送者硬件地址
    u8 arp_sha[6];            /* ARP Sender hardware address           */
    // 标识是否设置了 ARP 发送者硬件地址
    bool arp_sha_set;
    // ARP 目标硬件地址
    u8 arp_tha[6];            
    // 标识是否设置了 ARP 目标硬件地址
    bool arp_tha_set;
    // ARP 发送者协议地址
    struct in_addr arp_spa;   
    // 标识是否设置了 ARP 发送者协议地址
    bool arp_spa_set;
    // ARP 目标协议地址
    struct in_addr arp_tpa;   
    // 标识是否设置了 ARP 目标协议地址
    bool arp_tpa_set;

    /* 回显模式 */
    // 回显端口，用于监听或连接
    u16 echo_port;           
    // 标识是否设置了回显端口
    bool echo_port_set;
    // 用户口令
    char echo_passphrase[1024]; 
    // 标识是否设置了用户口令
    bool echo_passphrase_set;
    // 是否进行加密和认证会话
    bool do_crypto;          
    // 是否回显应用层负载
    bool echo_payload;       
    // 标识是否设置了回显应用层负载
    bool echo_payload_set;   
    // 是否仅为一个客户端运行服务器并退出
    bool echo_server_once;   
    // 标识是否设置了仅运行一次服务器
    bool echo_server_once_set;
    // 上次发送数据包的时间
    struct timeval last_sent_pkt_time; 
    // 延迟接收的输出字符串
    char *delayed_rcvd_str;    
    // 是否有延迟接收的字符串
    bool delayed_rcvd_str_set; 
    // 延迟接收的 Nsock 事件
    nsock_event_id delayed_rcvd_event; 

  public:
    // Nping 目标
    NpingTargets targets;
    // Nping 统计信息
    NpingStats stats;                      

  public:

    /* 构造函数/析构函数 */
    NpingOps();
    ~NpingOps();

    /* 探测模式 */
    int setMode(int md);
    int getMode();
    char *mode2Ascii(int md);
    bool issetMode();

    bool getTraceroute();
    bool enableTraceroute();
    bool disableTraceroute();
    bool issetTraceroute();

    /* 输出 */
    int setVerbosity(int level);
    int getVerbosity();
    int increaseVerbosity();
    int decreaseVerbosity();
    bool issetVerbosity();

    int setDebugging(int level);
    int getDebugging();
    int increaseDebugging();
    bool issetDebugging();

    int setShowSentPackets(bool val);
    bool showSentPackets();
    bool issetShowSentPackets();

    /* 操作和性能 */
    int setHostTimeout(long t);
    // 获取主机超时时间
    long getHostTimeout();
    // 检查主机超时时间是否已设置
    bool issetHostTimeout();

    // 设置延迟时间
    int setDelay(long t);
    // 获取延迟时间
    long getDelay();
    // 检查延迟时间是否已设置
    bool issetDelay();

    // 设置数据包数量
    int setPacketCount(u32 val);
    // 获取数据包数量
    u32 getPacketCount();
    // 检查数据包数量是否已设置
    bool issetPacketCount();

    // 设置发送偏好
    int setSendPreference(int v);
    // 获取发送偏好
    int getSendPreference();
    // 检查发送偏好是否已设置
    bool issetSendPreference();
    // 检查是否以太网发送偏好
    bool sendPreferenceEthernet();
    // 检查是否 IP 发送偏好
    bool sendPreferenceIP();

    // 设置是否发送以太网帧
    int setSendEth(bool val);
    // 检查是否发送以太网帧
    bool sendEth();
    // 检查是否已设置发送以太网帧
    bool issetSendEth();

    // 设置设备名称
    int setDevice(char *n);
    // 获取设备名称
    char *getDevice();
    // 检查设备名称是否已设置
    bool issetDevice();

    // 设置伪造源地址
    int setSpoofSource();
    // 检查是否伪造源地址
    bool spoofSource();
    // 获取是否伪造源地址
    bool getSpoofSource();
    // 检查是否已设置伪造源地址
    bool issetSpoofSource();

    // 设置 BPF 过滤器规范
    int setBPFFilterSpec(char *val);
    // 获取 BPF 过滤器规范
    char *getBPFFilterSpec();
    // 检查 BPF 过滤器规范是否已设置
    bool issetBPFFilterSpec();

    // 设置当前轮次
    int setCurrentRound(int val);
    // 获取当前轮次
    int getCurrentRound();
    // 检查当前轮次是否已设置
    bool issetCurrentRound();

    // 检查是否有 pcap
    bool havePcap();
    // 设置是否有 pcap
    int setHavePcap(bool val);

    // 设置是否禁用数据包捕获
    int setDisablePacketCapture(bool val);
    // 检查是否禁用数据包捕获
    bool disablePacketCapture();
    // 检查是否已设置禁用数据包捕获
    bool issetDisablePacketCapture();

    // 设置 IP 版本
    int setIPVersion(u8 val);
    // 获取 IP 版本
    int getIPVersion();
    // 检查 IP 版本是否已设置
    bool issetIPVersion();
    // 检查是否 IPv4
    bool ipv4();
    // 检查是否 IPv6
    bool ipv6();
    // 检查是否使用套接字的 IPv6
    bool ipv6UsingSocket();
    // 获取地址族
    int af();

    /* Privileges */
    // 设置是否为 root 用户
    int setIsRoot(int v);
    // 获取是否为 root 用户
    int setIsRoot();
    // 检查是否为 root 用户
    bool isRoot();

    /* Payloads */
    // 设置负载类型
    int setPayloadType(int t);
    // 获取负载类型
    int getPayloadType();
    // 检查负载类型是否已设置
    bool issetPayloadType();
    // 设置负载缓冲区
    int setPayloadBuffer(u8 *p, int len);
    // 获取负载缓冲区
    u8 *getPayloadBuffer();
    // 检查负载缓冲区是否已设置
    bool issetPayloadBuffer();
    // 获取负载长度
    int getPayloadLen();
    // 检查负载长度是否已设置
    bool issetPayloadLen();

    /* Roles */
    // 设置角色
    int setRole(int r);
    // 设置为客户端角色
    int setRoleClient();
    // 设置为服务器角色
    int setRoleServer();
    // 设置为普通角色
    int setRoleNormal();
    // 获取角色
    int getRole();
    // 检查角色是否已设置
    bool issetRole();

    /* IPv4 */
    // 启用错误校验和的 IPv4
    bool enableBadsumIP();
    // 禁用错误校验和的 IPv4
    bool disableBadsumIP();
    // 获取是否启用错误校验和的 IPv4
    bool getBadsumIP();
    // 检查是否已设置错误校验和的 IPv4
    bool issetBadsumIP();

    // 设置 TTL
    int setTTL(u8 t);
    // 获取 TTL
    u8 getTTL();
    // 检查 TTL 是否已设置
    bool issetTTL();

    // 设置 TOS
    int setTOS(u8 tos);
    // 获取 TOS
    u8 getTOS();
    // 检查 TOS 是否已设置
    bool issetTOS();

    // 设置标识
    int setIdentification(u16 i);
    // 获取标识
    u16 getIdentification();
    // 检查标识是否已设置
    bool issetIdentification();
    // 设置MF（More Fragments）标志位
    int setMF();
    // 获取MF（More Fragments）标志位
    bool getMF();
    // 检查MF（More Fragments）标志位是否被设置
    bool issetMF();

    // 设置DF（Don't Fragment）标志位
    int setDF();
    // 获取DF（Don't Fragment）标志位
    bool getDF();
    // 检查DF（Don't Fragment）标志位是否被设置
    bool issetDF();

    // 设置RF（Reserved Flag）标志位
    int setRF();
    // 获取RF（Reserved Flag）标志位
    bool getRF();
    // 检查RF（Reserved Flag）标志位是否被设置
    bool issetRF();

    // 获取IPv4源地址
    struct in_addr getIPv4SourceAddress();
    // 设置IPv4源地址
    int setIPv4SourceAddress(struct in_addr i);
    // 检查IPv4源地址是否被设置
    bool issetIPv4SourceAddress();

    // 设置IP选项
    int setIPOptions(char *txt);
    // 获取IP选项
    char *getIPOptions();
    // 检查IP选项是否被设置
    bool issetIPOptions();

    // 设置MTU（Maximum Transmission Unit）
    int setMTU(u32 t);
    // 获取MTU（Maximum Transmission Unit）
    u32 getMTU();
    // 检查MTU（Maximum Transmission Unit）是否被设置
    bool issetMTU();

    /* IPv6 */
    // 设置流量类别
    int setTrafficClass(u8 val);
    // 获取流量类别
    u8 getTrafficClass();
    // 检查流量类别是否被设置
    bool issetTrafficClass();

    // 设置流标签
    int setFlowLabel(u32 val);
    // 获取流标签
    u32 getFlowLabel();
    // 检查流标签是否被设置
    bool issetFlowLabel();

    // 设置跳限制
    int setHopLimit(u8 t);
    // 获取跳限制
    u8 getHopLimit();
    // 检查跳限制是否被设置
    bool issetHopLimit();

    // 设置IPv6源地址
    int setIPv6SourceAddress(u8 *val);
    int setIPv6SourceAddress(struct in6_addr val);    
    // 获取IPv6源地址
    struct in6_addr getIPv6SourceAddress();
    // 检查IPv6源地址是否被设置

    // 获取源套接字地址
    struct sockaddr_storage *getSourceSockAddr();
    struct sockaddr_storage *getSourceSockAddr(struct sockaddr_storage *ss);

    /* TCP / UDP */
    // 获取目标端口
    u16 *getTargetPorts( int *len );
    // 设置目标端口
    int setTargetPorts( u16 *pnt, int n );
    // 检查目标端口是否被设置
    bool issetTargetPorts();
    // 检查扫描模式是否使用目标端口
    bool scan_mode_uses_target_ports(int mode);

    // 设置源端口
    int setSourcePort(u16 val);
    // 获取源端口
    u16 getSourcePort();
    // 检查源端口是否被设置

    // 启用校验和错误
    bool enableBadsum();
    // 禁用校验和错误
    bool disableBadsum();
    // 获取校验和错误状态
    bool getBadsum();
    // 检查校验和错误是否被设置

    // 设置TCP标志位
    int setFlagTCP(int flag);
    // 设置所有TCP标志位
    int setAllFlagsTCP();
    // 清除所有TCP标志位
    int unsetAllFlagsTCP();
    // 获取指定TCP标志位
    int getFlagTCP(int flag);
    // 获取TCP标志
    u8 getTCPFlags();
    // 检查TCP标志是否被设置

    // 设置TCP序列号
    int setTCPSequence(u32 val);
    // 获取TCP序列号
    u32 getTCPSequence();
    // 检查TCP序列号是否被设置

    // 设置TCP确认号
    int setTCPAck(u32 val);
    // 获取TCP确认号
    u32 getTCPAck();
    // 检查TCP确认号是否被设置

    // 设置TCP窗口大小
    int setTCPWindow(u16 val);
    // 获取TCP窗口大小
    u16 getTCPWindow();
    // 检查TCP窗口大小是否被设置

    /* ICMP */
    // 设置ICMP类型
    int setICMPType(u8 type);
    // 获取ICMP类型
    u8 getICMPType();
    // 检查ICMP类型是否被设置

    // 设置ICMP代码
    int setICMPCode(u8 val);
    // 获取ICMP代码
    u8 getICMPCode();
    // 检查ICMP代码是否被设置

    // 启用ICMP校验和错误
    bool enableBadsumICMP();
    # 禁用错误校验和的 ICMP
    bool disableBadsumICMP();
    # 获取错误校验和的 ICMP
    bool getBadsumICMP();
    # 检查是否设置了错误校验和的 ICMP
    
    bool issetBadsumICMP();
    
    # 设置 ICMP 重定向地址
    int setICMPRedirectAddress(struct in_addr val);
    # 获取 ICMP 重定向地址
    struct in_addr getICMPRedirectAddress();
    # 检查是否设置了 ICMP 重定向地址
    bool issetICMPRedirectAddress();
    
    # 设置 ICMP 参数问题指针
    int setICMPParamProblemPointer(u8 val);
    # 获取 ICMP 参数问题指针
    u8 getICMPParamProblemPointer();
    # 检查是否设置了 ICMP 参数问题指针
    bool issetICMPParamProblemPointer();
    
    # 设置 ICMP 路由器通告生存时间
    int setICMPRouterAdvLifetime(u16 val);
    # 获取 ICMP 路由器通告生存时间
    u16 getICMPRouterAdvLifetime();
    # 检查是否设置了 ICMP 路由器通告生存时间
    bool issetICMPRouterAdvLifetime();
    
    # 设置 ICMP 标识符
    int setICMPIdentifier(u16 val);
    # 获取 ICMP 标识符
    u16 getICMPIdentifier();
    # 检查是否设置了 ICMP 标识符
    bool issetICMPIdentifier();
    
    # 设置 ICMP 序列号
    int setICMPSequence(u16 val);
    # 获取 ICMP 序列号
    u16 getICMPSequence();
    # 检查是否设置了 ICMP 序列号
    bool issetICMPSequence();
    
    # 设置 ICMP 发起时间戳
    int setICMPOriginateTimestamp(u32 val);
    # 获取 ICMP 发起时间戳
    u32 getICMPOriginateTimestamp();
    # 检查是否设置了 ICMP 发起时间戳
    bool issetICMPOriginateTimestamp();
    
    # 设置 ICMP 接收时间戳
    int setICMPReceiveTimestamp(u32 val);
    # 获取 ICMP 接收时间戳
    u32 getICMPReceiveTimestamp();
    # 检查是否设置了 ICMP 接收时间戳
    bool issetICMPReceiveTimestamp();
    
    # 设置 ICMP 发送时间戳
    int setICMPTransmitTimestamp(u32 val);
    # 获取 ICMP 发送时间戳
    u32 getICMPTransmitTimestamp();
    # 检查是否设置了 ICMP 发送时间戳
    bool issetICMPTransmitTimestamp();
    
    # 添加 ICMP 通告条目
    int addICMPAdvertEntry(struct in_addr addr, u32 pref );
    # 获取 ICMP 通告条目
    int getICMPAdvertEntry(int num, struct in_addr *addr, u32 *pref);
    # 获取 ICMP 通告条目数量
    int getICMPAdvertEntryCount();
    # 检查是否设置了 ICMP 通告条目
    bool issetICMPAdvertEntry();
    
    # 设置源 MAC 地址
    int setSourceMAC(u8 * val);
    # 获取源 MAC 地址
    u8 * getSourceMAC();
    # 检查是否设置了源 MAC 地址
    bool issetSourceMAC();
    
    # 设置目的 MAC 地址
    int setDestMAC(u8 * val);
    # 获取目的 MAC 地址
    u8 * getDestMAC();
    # 检查是否设置了目的 MAC 地址
    bool issetDestMAC();
    
    # 设置以太网类型
    int setEtherType(u16 val);
    # 获取以太网类型
    u16 getEtherType();
    # 检查是否设置了以太网类型
    
    bool issetEtherType();
    
    # 设置 ARP/RARP 硬件类型
    int setARPHardwareType(u16 val);
    # 获取 ARP/RARP 硬件类型
    u16 getARPHardwareType();
    # 检查是否设置了 ARP/RARP 硬件类型
    bool issetARPHardwareType();
    
    # 设置 ARP/RARP 协议类型
    int setARPProtocolType(u16 val);
    # 获取 ARP/RARP 协议类型
    u16 getARPProtocolType();
    # 检查是否设置了 ARP/RARP 协议类型
    bool issetARPProtocolType();
    
    # 设置 ARP/RARP 硬件地址长度
    int setARPHwAddrLen(u8 val);
    # 获取 ARP/RARP 硬件地址长度
    u8 getARPHwAddrLen();
    # 检查是否设置了 ARP/RARP 硬件地址长度
    bool issetARPHwAddrLen();
    
    # 设置 ARP/RARP 协议地址长度
    int setARPProtoAddrLen(u8 val);
    # 获取 ARP/RARP 协议地址长度
    u8 getARPProtoAddrLen();
    # 检查是否设置了 ARP/RARP 协议地址长度
    bool issetARPProtoAddrLen();
    
    # 设置 ARP/RARP 操作码
    int setARPOpCode(u16 val);
    # 获取 ARP/RARP 操作码
    u16 getARPOpCode();
    # 检查是否设置了 ARP/RARP 操作码
    bool issetARPOpCode();
    # 设置 ARP 发送者硬件地址
    int setARPSenderHwAddr(u8 * val);
    # 获取 ARP 发送者硬件地址
    u8 * getARPSenderHwAddr();
    # 检查是否设置了 ARP 发送者硬件地址
    bool issetARPSenderHwAddr();

    # 设置 ARP 目标硬件地址
    int setARPTargetHwAddr(u8 * val);
    # 获取 ARP 目标硬件地址
    u8 * getARPTargetHwAddr();
    # 检查是否设置了 ARP 目标硬件地址
    bool issetARPTargetHwAddr();

    # 设置 ARP 发送者协议地址
    int setARPSenderProtoAddr(struct in_addr val);
    # 获取 ARP 发送者协议地址
    struct in_addr getARPSenderProtoAddr();
    # 检查是否设置了 ARP 发送者协议地址
    bool issetARPSenderProtoAddr();

    # 设置 ARP 目标协议地址
    int setARPTargetProtoAddr(struct in_addr val);
    # 获取 ARP 目标协议地址
    struct in_addr getARPTargetProtoAddr();
    # 检查是否设置了 ARP 目标协议地址
    bool issetARPTargetProtoAddr();

    # 设置回显模式端口
    int setEchoPort(u16 val);
    # 获取回显模式端口
    u16 getEchoPort();
    # 检查是否设置了回显模式端口
    bool issetEchoPort();

    # 设置回显模式口令
    int setEchoPassphrase(const char *str);
    # 获取回显模式口令
    char *getEchoPassphrase();
    # 检查是否设置了回显模式口令
    bool issetEchoPassphrase();

    # 执行加密操作
    bool doCrypto();
    int doCrypto(bool value);

    # 回显负载
    bool echoPayload();
    int echoPayload(bool value);

    # 设置一次性标志
    int setOnce(bool val);
    # 获取一次性标志
    bool once();

    # 验证选项
    void validateOptions();
    # 是否可以在无特权情况下运行 UDP
    bool canRunUDPWithoutPrivileges();
    # 是否可以通过套接字进行 IPv6
    bool canDoIPv6ThroughSocket();
    # 是否可以通过以太网进行 IPv6
    bool canDoIPv6Ethernet();
    # 选择网络接口
    char *select_network_iface();

    # 其他
    void displayNpingDoneMsg();
    void displayStatistics();
    int cleanup();
    int setDefaultHeaderValues();
    int getTotalProbes();

    # 设置最后发送数据包的时间
    int setLastPacketSentTime(struct timeval t);
    # 获取最后发送数据包的时间
    struct timeval getLastPacketSentTime();

    # 设置延迟接收的数据
    int setDelayedRcvd(const char *str, nsock_event_id id);
    # 获取延迟接收的数据
    char *getDelayedRcvd(nsock_event_id *id);
}; /* End of class NpingOps */

#endif // NPING_NPINGOPS_H

这段代码是C++中的类定义结束和头文件结束的标记。在这里，"};"表示类定义的结束，"#endif"表示条件编译的结束，即#ifndef的结束。
```