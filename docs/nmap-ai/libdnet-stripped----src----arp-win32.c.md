# `nmap\libdnet-stripped\src\arp-win32.c`

```cpp
/*
 * arp-win32.c
 *
 * 版权所有 (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp-win32.c 539 2005-01-23 07:36:54Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <ws2tcpip.h>
#include <iphlpapi.h>

#include <errno.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

struct arp_handle {
    MIB_IPNET_TABLE2 *iptable;
};

// 打开 ARP 处理器，返回一个 arp_t 结构
arp_t *
arp_open(void)
{
    return (calloc(1, sizeof(arp_t)));
}

// 添加 ARP 条目
int
arp_add(arp_t *arp, const struct arp_entry *entry)
{
    MIB_IPFORWARDROW ipfrow;
    MIB_IPNETROW iprow;
    
    // 获取最佳路由
    if (GetBestRoute(entry->arp_pa.addr_ip,
        IP_ADDR_ANY, &ipfrow) != NO_ERROR)
        return (-1);

    iprow.dwIndex = ipfrow.dwForwardIfIndex;
    iprow.dwPhysAddrLen = ETH_ADDR_LEN;
    memcpy(iprow.bPhysAddr, &entry->arp_ha.addr_eth, ETH_ADDR_LEN);
    iprow.dwAddr = entry->arp_pa.addr_ip;
    iprow.dwType = 4;    /* XXX - static */

    // 创建 ARP 条目
    if (CreateIpNetEntry(&iprow) != NO_ERROR)
        return (-1);

    return (0);
}

// 删除 ARP 条目
int
arp_delete(arp_t *arp, const struct arp_entry *entry)
{
    MIB_IPFORWARDROW ipfrow;
    MIB_IPNETROW iprow;

    // 获取最佳路由
    if (GetBestRoute(entry->arp_pa.addr_ip,
        IP_ADDR_ANY, &ipfrow) != NO_ERROR)
        return (-1);

    memset(&iprow, 0, sizeof(iprow));
    iprow.dwIndex = ipfrow.dwForwardIfIndex;
    iprow.dwAddr = entry->arp_pa.addr_ip;

    // 删除 ARP 条目
    if (DeleteIpNetEntry(&iprow) != NO_ERROR) {
        errno = ENXIO;
        return (-1);
    }
    return (0);
}

// 获取 ARP 条目
static int
_arp_get_entry(const struct arp_entry *entry, void *arg)
{
    struct arp_entry *e = (struct arp_entry *)arg;
    
    if (addr_cmp(&entry->arp_pa, &e->arp_pa) == 0) {
        memcpy(&e->arp_ha, &entry->arp_ha, sizeof(e->arp_ha));
        return (1);
    }
    return (0);
}

int
arp_get(arp_t *arp, struct arp_entry *entry)
{
    // 循环获取 ARP 条目
    if (arp_loop(arp, _arp_get_entry, entry) != 1) {
        errno = ENXIO;
        SetLastError(ERROR_NO_DATA);
        return (-1);
    }
    return (0);
}

int
# 定义一个名为 arp_loop 的函数，接受 arp_t 结构体指针、arp_handler 回调函数和 void 指针作为参数
arp_loop(arp_t *arp, arp_handler callback, void *arg)
{
    # 定义一个名为 entry 的 arp_entry 结构体变量
    struct arp_entry entry;
    # 定义一个名为 ret 的整型变量
    int ret;

    # 如果 arp 结构体中的 iptable 不为空，则释放其内存
    if (arp->iptable)
        FreeMibTable(arp->iptable);
    # 调用 GetIpNetTable2 函数获取 IP 地址和 MAC 地址的映射表
    ret = GetIpNetTable2(AF_UNSPEC, &arp->iptable);
    # 根据返回值进行不同的处理
    switch (ret) {
        # 如果返回 NO_ERROR，则继续执行
        case NO_ERROR:
            break;
        # 如果返回 ERROR_NOT_FOUND，则返回 0
        case ERROR_NOT_FOUND:
            return 0;
            break;
        # 其他情况下返回 -1
        default:
            return -1;
            break;
    }
    
    # 设置 entry 结构体中的 arp_ha 的地址类型和地址位数
    entry.arp_ha.addr_type = ADDR_TYPE_ETH;
    entry.arp_ha.addr_bits = ETH_ADDR_BITS;
    
    # 遍历 iptable 中的每一条记录
    for (ULONG i = 0; i < arp->iptable->NumEntries; i++) {
        # 获取当前记录
        MIB_IPNET_ROW2 *row = &arp->iptable->Table[i];
        # 如果记录中的物理地址长度不等于 ETH_ADDR_LEN，或者不可达，或者状态小于 NlnsReachable，则跳过当前记录
        if (row->PhysicalAddressLength != ETH_ADDR_LEN ||
                row->IsUnreachable ||
                row->State < NlnsReachable)
            continue;
        # 根据地址族进行不同的处理
        switch (row->Address.si_family) {
            # 如果是 AF_INET 地址族，则设置 entry 结构体中的 arp_pa 的地址类型、地址位数和地址
            case AF_INET:
                entry.arp_pa.addr_type = ADDR_TYPE_IP;
                entry.arp_pa.addr_bits = IP_ADDR_BITS;
                entry.arp_pa.addr_ip = row->Address.Ipv4.sin_addr.S_un.S_addr;
                break;
            # 如果是 AF_INET6 地址族，则设置 entry 结构体中的 arp_pa 的地址类型、地址位数和地址
            case AF_INET6:
                entry.arp_pa.addr_type = ADDR_TYPE_IP6;
                entry.arp_pa.addr_bits = IP6_ADDR_BITS;
                memcpy(&entry.arp_pa.addr_ip6,
                        row->Address.Ipv6.sin6_addr.u.Byte, IP6_ADDR_LEN);
                break;
            # 其他情况下跳过当前记录
            default:
                continue;
                break;
        }
        # 将记录中的物理地址复制到 entry 结构体中的 arp_ha 中
        memcpy(&entry.arp_ha.addr_eth,
            row->PhysicalAddress, ETH_ADDR_LEN);
        
        # 调用回调函数，并根据返回值进行处理
        if ((ret = (*callback)(&entry, arg)) != 0)
            return (ret);
    }
    # 返回 0
    return (0);
}

# 定义一个名为 arp_close 的函数，接受 arp_t 结构体指针作为参数
arp_t *
arp_close(arp_t *arp)
{
    # 如果 arp 不为空，则释放其内存，并返回 NULL
    if (arp != NULL) {
        if (arp->iptable != NULL)
            FreeMibTable(arp->iptable);
        free(arp);
    }
    return (NULL);
}
```