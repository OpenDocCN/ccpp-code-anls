# `nmap\ncat\ncat_core.h`

```cpp
/* $Id$ */

#ifndef NCAT_CORE_H
#define NCAT_CORE_H

#include "nsock.h"
#include "nbase.h"
#include "util.h"
#include "sockaddr_u.h"

/* 最大 srcaddrs 数组的大小。在这种情况下，最多为两个，因为我们最多只能有一个 IPV4 INADDR_ANY 和一个 IPV6 in6addr_any，或者用户定义的地址 */
#define NUM_LISTEN_ADDRS 2

/* 用于存储已解析地址的链表结构 */
struct sockaddr_list {
    union sockaddr_u addr;
    size_t addrlen;
    struct sockaddr_list* next;
};

extern union sockaddr_u listenaddrs[NUM_LISTEN_ADDRS];
extern int num_listenaddrs;

extern union sockaddr_u srcaddr;
extern size_t srcaddrlen;

extern struct sockaddr_list *targetaddrs;

enum exec_mode {
    EXEC_PLAIN,
    EXEC_SHELL,
    EXEC_LUA,
};

/* 代理 DNS 解析选项（掩码位） */
#define PROXYDNS_LOCAL  1
#define PROXYDNS_REMOTE 2

struct options {
    unsigned int portno;

    int verbose;
    int debug;
    char *target;
    int af;
    /* IPPROTO_TCP, IPPROTO_SCTP, or IPPROTO_UDP */
    int proto;
    int broker;
    int listen;
    int keepopen;
    int sendonly;
    int recvonly;
    int noshutdown;
    int telnet;
    int linedelay;
    int chat;
    int nodns;
    const char *normlog;
    const char *hexlog;
    int normlogfd;
    int hexlogfd;
    int append;
    int idletimeout;
    int crlf;
    /* 是否有特别允许的主机？如果有，拒绝所有其他主机。 */
    int allow;
    int deny;
    struct addrset *allowset;
    struct addrset *denyset;
    int httpserver;
    int nsock_engine;
    /* 对 stderr 输出有用的测试消息？ */
    int test;

    /* 宽松的源路由信息 */
    struct in_addr srcrtes[8];
    int numsrcrtes;
    int srcrteptr;

    /* 最大同时连接数 */
    int conn_limit;
    int conntimeout;

    /* 当 execmode == EXEC_LUA 时，cmdexec 是要运行的文件名。 */
    char *cmdexec;
    enum exec_mode execmode;
    char *proxy_auth;
    char *proxytype;
    char *proxyaddr;
    # 定义一个整型变量，用于存储代理 DNS 的值
    int proxydns;
    
    # 定义一个整型变量，用于存储 SSL 的值
    int ssl;
    
    # 定义一个指向 SSL 证书的指针
    char *sslcert;
    
    # 定义一个指向 SSL 密钥的指针
    char *sslkey;
    
    # 定义一个整型变量，用于存储 SSL 验证的值
    int sslverify;
    
    # 定义一个指向 SSL 信任文件的指针
    char *ssltrustfile;
    
    # 定义一个指向 SSL 密钥协商算法的指针
    char *sslciphers;
    
    # 定义一个指向 SSL 服务器名称的指针
    char* sslservername;
    
    # 定义一个指向 SSL 应用层协议协商的指针
    char *sslalpn;
    
    # 定义一个整型变量，用于存储零字节的值
    int zerobyte;
# 结构体 options 的外部声明
extern struct options o;

# 程序启动时的时间，用于连接模式下的退出统计
extern struct timeval start_time;

# 初始化全局选项为它们的默认值
void options_init(void);

# 使用 getaddrinfo 解析给定的主机名或 IP 地址，并将第一个结果（如果有）存储在 *ss 和 *sslen 中
# 如果端口不重要，将在 *ss 中设置为 0
# af 可能是 AF_UNSPEC，在这种情况下，getaddrinfo 可能返回 IPv4 和 IPv6 结果；哪个是第一个取决于系统配置
# 成功返回 0，失败返回 getaddrinfo 的返回码（适用于传递给 gai_strerror）
# 当此函数返回 0 时，*ss 和 *sslen 总是定义的
# 如果全局变量 o.nodns 为 true，则不使用 DNS 解析任何名称
int resolve(const char *hostname, unsigned short port,
            struct sockaddr_storage *ss, size_t *sslen, int af);

# 使用 getaddrinfo 解析给定的主机名或 IP 地址，并将第一个结果（如果有）存储在 *ss 和 *sslen 中
# 如果端口不重要，将在 *ss 中设置为 0
# af 可能是 AF_UNSPEC，在这种情况下，getaddrinfo 可能返回 IPv4 和 IPv6 结果；哪个是第一个取决于系统配置
# 成功返回 0，失败返回 getaddrinfo 的返回码（适用于传递给 gai_strerror）
# 当此函数返回 0 时，*ss 和 *sslen 总是定义的
# 如果全局变量 o.proxydns 包括 PROXYDNS_LOCAL，则仅使用 DNS 解析主机名
int proxyresolve(const char *hostname, unsigned short port,
            struct sockaddr_storage *ss, size_t *sslen, int af);

# 使用 getaddrinfo 解析给定的主机名或 IP
/* 释放存储地址列表结构体所占用的内存 */
void free_sockaddr_list(struct sockaddr_list *sl);

/* 关闭文件描述符信息结构体 */
int fdinfo_close(struct fdinfo *fdn);
/* 从文件描述符接收数据 */
int fdinfo_recv(struct fdinfo *fdn, char *buf, size_t size);
/* 向文件描述符发送数据 */
int fdinfo_send(struct fdinfo *fdn, const char *buf, size_t size);
/* 检查文件描述符是否有待处理的数据 */
int fdinfo_pending(struct fdinfo *fdn);

/* 从文件描述符接收数据，同时返回是否有待处理的数据 */
int ncat_recv(struct fdinfo *fdn, char *buf, size_t size, int *pending);
/* 向文件描述符发送数据 */
int ncat_send(struct fdinfo *fdn, const char *buf, size_t size);

/* 向文件描述符列表中的所有描述符广播消息，如果发送失败则返回-1 */
extern int ncat_broadcast(fd_set *fds, const fd_list_t *fdlist, const char *msg, size_t size);

/* 执行 Telnet 协议的 WILL/WONT DO/DONT 协商 */
extern void dotelnet(int s, unsigned char *buf, size_t bufsiz);

/* 实现跨平台的延时函数，阻塞直到指定的时间过去，然后返回1 */
extern int ncat_delay_timer(int delayval);

/* 打开一个日志文件进行写入，返回打开的文件描述符 */
extern int ncat_openlog(const char *logfile, int append);

/* 记录发送的数据到日志 */
extern void ncat_log_send(const char *data, size_t len);

/* 记录接收的数据到日志 */
extern void ncat_log_recv(const char *data, size_t len);

/* 检查远程 IP 地址是否有权限访问指定的文件 */
extern int ncat_hostaccess(char *matchaddr, char *filename, char *remoteip);

/* 设置从控制台读取的行结束符为 \n（而不是 \r\n） */
extern void set_lf_mode(void);

/* 获取地址的地址族（IPv4 或 IPv6） */
extern int getaddrfamily(const char *addr);
/* 设置环境变量，兼容不同平台 */
extern int setenv_portable(const char *name, const char *value);
/* 设置环境变量 */
extern void setup_environment(struct fdinfo *fdinfo);

#endif
```