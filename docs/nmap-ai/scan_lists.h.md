# `nmap\scan_lists.h`

```
#ifndef SCAN_LISTS_H
#define SCAN_LISTS_H

/* 定义标志位，指示特定端口号应该进行 TCP 扫描、UDP 扫描，或者两者都进行 */
#define SCAN_TCP_PORT    (1 << 0)
#define SCAN_UDP_PORT    (1 << 1)
#define SCAN_SCTP_PORT    (1 << 2)
#define SCAN_PROTOCOLS    (1 << 3)

/* 我们可以进行的各种端口/协议扫描类型
 * 每个元素指向一个端口/协议号数组
 */
struct scan_lists {
        /* "synprobes" 也用于执行 connect() ping */
        unsigned short *syn_ping_ports;
        unsigned short *ack_ping_ports;
        unsigned short *udp_ping_ports;
        unsigned short *sctp_ping_ports;
        unsigned short *proto_ping_ports;
        int syn_ping_count;
        int ack_ping_count;
        int udp_ping_count;
        int sctp_ping_count;
        int proto_ping_count;
        // 上述字段仅用于主机发现
        // 下面的字段仅用于端口扫描
        unsigned short *tcp_ports;
        int tcp_count;
        unsigned short *udp_ports;
        int udp_count;
        unsigned short *sctp_ports;
        int sctp_count;
        unsigned short *prots;
        int prot_count;
};

typedef enum {
  STYPE_UNKNOWN,
  HOST_DISCOVERY,
  ACK_SCAN,
  SYN_SCAN,
  FIN_SCAN,
  XMAS_SCAN,
  UDP_SCAN,
  CONNECT_SCAN,
  NULL_SCAN,
  WINDOW_SCAN,
  SCTP_INIT_SCAN,
  SCTP_COOKIE_ECHO_SCAN,
  MAIMON_SCAN,
  IPPROT_SCAN,
  PING_SCAN,
  PING_SCAN_ARP,
  IDLE_SCAN,
  BOUNCE_SCAN,
  SERVICE_SCAN,
  OS_SCAN,
  SCRIPT_PRE_SCAN,
  SCRIPT_SCAN,
  SCRIPT_POST_SCAN,
  TRACEROUTE,
  PING_SCAN_ND
} stype;

/* 端口操作函数 */
void getpts(const char *expr, struct scan_lists * ports); /* 有人偷了 getports() 这个名字！ */
void getpts_simple(const char *origexpr, int range_type,
                   unsigned short **list, int *count);
void removepts(const char *expr, struct scan_lists * ports);
void free_scan_lists(struct scan_lists *ports);

/* 通用辅助函数 */
// 声明一个函数，该函数接受一个名为scantype的参数，返回一个指向const char类型的指针
const char *scantype2str(stype scantype);
// 结束条件预处理指令，结束头文件的定义
#endif /* SCAN_LISTS_H */
```