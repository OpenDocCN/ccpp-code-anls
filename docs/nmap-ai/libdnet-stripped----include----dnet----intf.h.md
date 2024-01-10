# `nmap\libdnet-stripped\include\dnet\intf.h`

```
#ifndef DNET_INTF_H
#define DNET_INTF_H

// 如果未定义 DNET_INTF_H，则定义 DNET_INTF_H


#define INTF_NAME_LEN    16

// 定义接口名称的最大长度为 16


struct intf_entry {
    u_int        intf_len;            /* length of entry */
    char        intf_name[INTF_NAME_LEN];   /* interface name */
    u_int        intf_index;            /* interface index (r/o) */
    u_short        intf_type;            /* interface type (r/o) */
    u_short        intf_flags;            /* interface flags */
    u_int        intf_mtu;            /* interface MTU */
    struct addr    intf_addr;            /* interface address */
    struct addr    intf_dst_addr;            /* point-to-point dst */
    struct addr    intf_link_addr;            /* link-layer address */
    u_int        intf_alias_num;            /* number of aliases */
    struct addr    intf_alias_addrs __flexarr; /* array of aliases */
};

// 定义接口条目结构体，包括长度、名称、索引、类型、标志、MTU、地址、目的地址、链路地址、别名数量和别名地址数组


#define INTF_TYPE_OTHER        1    /* other */
#define INTF_TYPE_ETH        6    /* Ethernet */
#define INTF_TYPE_TOKENRING    9    /* Token Ring */
#define INTF_TYPE_FDDI        15    /* FDDI */
#define INTF_TYPE_PPP        23    /* Point-to-Point Protocol */
#define INTF_TYPE_LOOPBACK    24    /* software loopback */
#define INTF_TYPE_SLIP        28    /* Serial Line Interface Protocol */
#define INTF_TYPE_TUN        53    /* proprietary virtual/internal */

// 定义不同接口类型的常量值


#define INTF_FLAG_UP        0x01    /* enable interface */
#define INTF_FLAG_LOOPBACK    0x02    /* is a loopback net (r/o) */
#define INTF_FLAG_POINTOPOINT    0x04    /* point-to-point link (r/o) */
#define INTF_FLAG_NOARP        0x08    /* disable ARP */
#define INTF_FLAG_BROADCAST    0x10    /* supports broadcast (r/o) */
#define INTF_FLAG_MULTICAST    0x20    /* supports multicast (r/o) */

// 定义不同接口标志的常量值


#endif

// 结束条件编译指令 DNET_INTF_H
// 定义 intf_t 结构体类型
typedef struct intf_handle intf_t;

// 定义 intf_handler 函数指针类型，接受 struct intf_entry 结构体指针和 void 指针作为参数，返回整型
typedef int (*intf_handler)(const struct intf_entry *entry, void *arg);

// 声明以下函数为 C 语言函数
__BEGIN_DECLS

// 打开接口，返回 intf_t 指针
intf_t    *intf_open(void);

// 获取接口信息，接受 intf_t 指针和 struct intf_entry 结构体指针作为参数，返回整型
int     intf_get(intf_t *i, struct intf_entry *entry);

// 根据索引获取接口信息，接受 intf_t 指针、struct intf_entry 结构体指针、地址族和索引作为参数，返回整型
int     intf_get_index(intf_t *intf, struct intf_entry *entry, int af, unsigned int index);

// 获取接口的源地址，接受 intf_t 指针、struct intf_entry 结构体指针和地址结构体指针作为参数，返回整型
int     intf_get_src(intf_t *i, struct intf_entry *entry, struct addr *src);

// 获取接口的目的地址，接受 intf_t 指针、struct intf_entry 结构体指针和地址结构体指针作为参数，返回整型
int     intf_get_dst(intf_t *i, struct intf_entry *entry, struct addr *dst);

// 获取接口的 pcap 设备名，接受接口名和 pcap 设备名指针作为参数，返回整型
int     intf_get_pcap_devname(const char *intf_name, char *pcapdev, int pcapdevlen);

// 获取接口的缓存 pcap 设备名，接受接口名、pcap 设备名指针、缓存长度和刷新标志作为参数，返回整型
int     intf_get_pcap_devname_cached(const char *intf_name, char *pcapdev, int pcapdevlen, int refresh);

// 设置接口信息，接受 intf_t 指针和 struct intf_entry 结构体指针作为参数，返回整型
int     intf_set(intf_t *i, const struct intf_entry *entry);

// 循环遍历接口信息，接受 intf_t 指针、回调函数和参数指针作为参数，返回整型
int     intf_loop(intf_t *i, intf_handler callback, void *arg);

// 关闭接口，接受 intf_t 指针作为参数，返回 intf_t 指针
intf_t    *intf_close(intf_t *i);

// 声明以下函数为 C 语言函数结束
__END_DECLS

// 结束条件编译，结束 intf.h 文件的定义
#endif /* DNET_INTF_H */
```