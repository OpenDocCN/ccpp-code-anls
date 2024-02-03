# `nmap\nmap_amigaos.h`

```cpp
// 定义了一个宏，用于替换 pcap_open_live 函数，使用 MiamiPCapOpenLive 函数
#define pcap_open_live(a, b, c, d...)    MiamiPCapOpenLive(a, b, 0, d)
// 定义了一个宏，用于替换 pcap_filter 函数，使用 MiamiPCapFilter 函数
#define pcap_filter(args...)        MiamiPCapFilter(args)
// 定义了一个宏，用于替换 pcap_close 函数，使用 MiamiPCapClose 函数
#define pcap_close(args...)        MiamiPCapClose(args)
// 定义了一个宏，用于替换 pcap_datalink 函数，使用 MiamiPCapDatalink 函数
#define pcap_datalink(args...)        MiamiPCapDatalink(args)
// 定义了一个宏，用于替换 pcap_geterr 函数，使用 MiamiPCapGeterr 函数
#define pcap_geterr(args...)        MiamiPCapGeterr(args)
// 定义了一个宏，用于替换 pcap_next 函数，使用 MiamiPCapNext 函数
#define pcap_next(args...)        MiamiPCapNext(args)
// 定义了一个宏，用于替换 pcap_lookupnet 函数，使用 MiamiPCapLookupnet 函数
#define pcap_lookupnet(args...)        MiamiPCapLookupnet(args)
// 定义了一个宏，用于替换 pcap_compile 函数，使用 MiamiPCapCompile 函数
#define pcap_compile(args...)        MiamiPCapCompile(args)
// 定义了一个宏，用于替换 pcap_setfilter 函数，使用 MiamiPCapSetfilter 函数
#define pcap_setfilter(args...)        MiamiPCapSetfilter(args)

// 如果 DLT_MIAMI 未定义，则定义 DLT_MIAMI 为 100
#ifndef DLT_MIAMI
#define DLT_MIAMI 100
#endif

// 如果 NI_NAMEREQD 未定义，则定义 NI_NAMEREQD 为 4
#ifndef NI_NAMEREQD
#define NI_NAMEREQD 4
#endif

// 定义了一个结构体 addrinfo，用于存储地址信息
struct addrinfo {
  long        ai_flags;        /* AI_PASSIVE, AI_CANONNAME */
  long        ai_family;        /* PF_xxx */
  long        ai_socktype;        /* SOCK_xxx */
  long        ai_protocol;        /* IPPROTO_xxx for IPv4 and IPv6 */
  size_t    ai_addrlen;        /* length of ai_addr */
  char        *ai_canonname;        /* canonical name for host */
  struct sockaddr    *ai_addr;    /* binary address */
  struct addrinfo    *ai_next;    /* next structure in linked list */
};

#endif /* _NMAP_AMIGAOS_H_ */
```