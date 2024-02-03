# `nmap\libpcap\pcap\ipnet.h`

```cpp
# 定义 IP 地址的协议簇为 AF_INET，对应的数值为 2，与 Solaris 的 AF_INET 匹配
#define    IPH_AF_INET    2        /* Matches Solaris's AF_INET */

# 定义 IP 地址的协议簇为 AF_INET6，对应的数值为 26，与 Solaris 的 AF_INET6 匹配
#define    IPH_AF_INET6    26        /* Matches Solaris's AF_INET6 */

# 定义 IP 地址的出站流量标识为 1
#define    IPNET_OUTBOUND        1

# 定义 IP 地址的入站流量标识为 2
#define    IPNET_INBOUND        2
```