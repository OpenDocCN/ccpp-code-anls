# `nmap\libdnet-stripped\src\intf-win32.c`

```cpp
/*
 * intf-win32.c
 *
 * 版权所有 (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: intf-win32.c 632 2006-08-10 04:36:52Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <iphlpapi.h>

#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"
#include "pcap.h"
#include <Packet32.h>
#include <Ntddndis.h>

int g_has_npcap_loopback = 0;
#define _DEVICE_PREFIX "\\Device\\"

struct ifcombo {
    struct {
        DWORD    ipv4;
        DWORD    ipv6;
    } *idx;
    int         cnt;
    int         max;
};

/* XXX - ipifcons.h incomplete, use IANA ifTypes MIB */
#define MIB_IF_TYPE_TUNNEL    131
#define MIB_IF_TYPE_MAX        MAX_IF_TYPE

struct intf_handle {
    struct ifcombo     ifcombo[MIB_IF_TYPE_MAX];
    IP_ADAPTER_ADDRESSES    *iftable;
};

static char *
_ifcombo_name(int type)
{
    char *name = NULL;
    
    switch (type) {
        case IF_TYPE_ETHERNET_CSMACD:
        case IF_TYPE_IEEE80211:
            name = "eth";
            break;
        case IF_TYPE_ISO88025_TOKENRING:
            name = "tr";
            break;
        case IF_TYPE_PPP:
            name = "ppp";
            break;
        case IF_TYPE_SOFTWARE_LOOPBACK:
            name = "lo";
            break;
        case IF_TYPE_TUNNEL:
            name = "tun";
            break;
        default:
            name = "unk";
            break;
    }
    return (name);
}

static int
_ifcombo_type(const char *device)
{
    int type = INTF_TYPE_OTHER;
    
    if (strncmp(device, "eth", 3) == 0) {
        type = INTF_TYPE_ETH;
    } else if (strncmp(device, "tr", 2) == 0) {
        type = INTF_TYPE_TOKENRING;
    } else if (strncmp(device, "ppp", 3) == 0) {
        type = INTF_TYPE_PPP;
    } else if (strncmp(device, "lo", 2) == 0) {
        type = INTF_TYPE_LOOPBACK;
    } else if (strncmp(device, "tun", 3) == 0) {
        type = INTF_TYPE_TUN;
    }
    return (type);
}

static void
# 向 ifcombo 结构体中添加索引对应的 IPv4 和 IPv6 索引
_ifcombo_add(struct ifcombo *ifc, DWORD ipv4_idx, DWORD ipv6_idx)
{
    void* pmem = NULL;
    # 如果 ifcombo 中的计数等于最大值
    if (ifc->cnt == ifc->max) {
        # 如果 ifc->idx 不为空
        if (ifc->idx) {
            # 将最大值扩大一倍
            ifc->max *= 2;
            # 重新分配内存
            pmem = realloc(ifc->idx,
                sizeof(ifc->idx[0]) * ifc->max);
        } else {
            # 将最大值设为 8
            ifc->max = 8;
            # 分配新的内存
            pmem = malloc(sizeof(ifc->idx[0]) * ifc->max);
        }
        # 如果内存分配失败
        if (!pmem) {
            /* malloc or realloc failed. Restore state.
             * TODO: notify caller. */
            # 恢复状态并返回
            ifc->max = ifc->cnt;
            return;
        }
        # 更新 ifc->idx 指向新的内存
        ifc->idx = pmem;
    }
    # 将 IPv4 和 IPv6 索引添加到 ifc->idx 中
    ifc->idx[ifc->cnt].ipv4 = ipv4_idx;
    ifc->idx[ifc->cnt].ipv6 = ipv6_idx;
    # 计数加一
    ifc->cnt++;
}

/* 将 MIB 接口类型映射为内部接口类型。内部类型永远不会暴露给此库的用户；它们只存在于 intf_handle 中的 ifcombo 结构的排序接口类型的缘故。 intf_handle 中的条目不得存储或访问原始 MIB 类型号，因为如果设备名称与类型不完全匹配，则无法通过设备名称（如"net0"）找到它们。 */
static int
_if_type_canonicalize(int type)
{
       return _ifcombo_type(_ifcombo_name(type));
}

static void
_adapter_address_to_entry(intf_t *intf, IP_ADAPTER_ADDRESSES *a,
    struct intf_entry *entry)
{
    struct addr *ap, *lap;
    int i;
    int type;
    IP_ADAPTER_UNICAST_ADDRESS *addr;
    
    /* 可能会将条目的总长度传递到 entry 中。记住它并清除条目。 */
    u_int intf_len = entry->intf_len;
    memset(entry, 0, sizeof(*entry));
    entry->intf_len = intf_len;

    type = _if_type_canonicalize(a->IfType);
    for (i = 0; i < intf->ifcombo[type].cnt; i++) {
        if (intf->ifcombo[type].idx[i].ipv4 == a->IfIndex &&
            intf->ifcombo[type].idx[i].ipv6 == a->Ipv6IfIndex) {
            break;
        }
    }
    /* XXX - type matches MIB-II ifType. */
}
    # 使用 snprintf 将接口名和序号格式化为字符串，存储到 entry->intf_name 中
    snprintf(entry->intf_name, sizeof(entry->intf_name), "%s%lu",
        _ifcombo_name(a->IfType), i);
    # 将接口类型转换为 uint16_t 类型，存储到 entry->intf_type 中
    entry->intf_type = (uint16_t)type;
    
    # 获取接口标志
    entry->intf_flags = 0;
    # 如果接口操作状态为 IfOperStatusUp，则将 INTF_FLAG_UP 标志位设置到 entry->intf_flags 中
    if (a->OperStatus == IfOperStatusUp)
        entry->intf_flags |= INTF_FLAG_UP;
    # 如果接口类型为 IF_TYPE_SOFTWARE_LOOPBACK，则将 INTF_FLAG_LOOPBACK 标志位设置到 entry->intf_flags 中
    if (a->IfType == IF_TYPE_SOFTWARE_LOOPBACK)
        entry->intf_flags |= INTF_FLAG_LOOPBACK;
    # 否则，将 INTF_FLAG_MULTICAST 标志位设置到 entry->intf_flags 中
    else
        entry->intf_flags |= INTF_FLAG_MULTICAST;
    
    # 获取接口最大传输单元（MTU）
    entry->intf_mtu = a->Mtu;
    
    # 获取硬件地址
    if (a->PhysicalAddressLength == ETH_ADDR_LEN) {
        # 如果物理地址长度等于 ETH_ADDR_LEN，则将地址类型和地址位数存储到 entry->intf_link_addr 中
        entry->intf_link_addr.addr_type = ADDR_TYPE_ETH;
        entry->intf_link_addr.addr_bits = ETH_ADDR_BITS;
        # 将物理地址拷贝到 entry->intf_link_addr 中
        memcpy(&entry->intf_link_addr.addr_eth, a->PhysicalAddress,
            ETH_ADDR_LEN);
    }
    
    # 获取地址
    ap = entry->intf_alias_addrs;
    lap = ap + ((entry->intf_len - sizeof(*entry)) /
        sizeof(entry->intf_alias_addrs[0]));
    # 遍历接口的单播地址列表
    for (addr = a->FirstUnicastAddress; addr != NULL; addr = addr->Next) {
        IP_ADAPTER_PREFIX *prefix;
        unsigned short bits;

        # 查找子网掩码长度
        bits = 0;
        # 如果地址长度大于等于 48，则获取 OnLinkPrefixLength 作为子网掩码长度
        if (addr->Length >= 48) {
            bits = addr->OnLinkPrefixLength;
        }
    else {
        // 如果条件不成立，则执行以下代码块
        for (prefix = a->FirstPrefix; prefix != NULL; prefix = prefix->Next) {
            // 遍历链表中的前缀，直到找到与地址族相同的前缀
            if (prefix->Address.lpSockaddr->sa_family == addr->Address.lpSockaddr->sa_family) {
                // 如果找到与地址族相同的前缀，则将前缀长度赋给变量bits
                bits = (unsigned short) prefix->PrefixLength;
                break;
            }
        }
    }

        if (entry->intf_addr.addr_type == ADDR_TYPE_NONE) {
            /* Set primary address if unset. */
            // 如果接口地址类型未设置，则将地址转换为网络字节顺序，并赋给接口地址
            addr_ston(addr->Address.lpSockaddr, &entry->intf_addr);
            // 将前缀长度赋给接口地址的地址位数
            entry->intf_addr.addr_bits = bits;
        } else if (ap < lap) {
            /* Set aliases. */
            // 如果别名指针ap小于最大别名指针lap，则将地址转换为网络字节顺序，并赋给别名指针ap
            addr_ston(addr->Address.lpSockaddr, ap);
            // 将前缀长度赋给别名指针ap的地址位数
            ap->addr_bits = bits;
            // 别名指针ap向后移动一位
            ap++;
            // 接口别名数量加一
            entry->intf_alias_num++;
        }
    }
    // 计算接口长度，即别名指针ap减去接口指针entry的差值
    entry->intf_len = (u_int) ((u_char *)ap - (u_char *)entry);
}
#define NPCAP_SERVICE_REGISTRY_KEY "SYSTEM\\CurrentControlSet\\Services\\npcap"

/* 从注册表中获取 Npcap 循环适配器的名称，存储在 npcap 服务的注册表键中的 LoopbackAdapter 值中。
 * 对于旧版循环支持，这是一个类似 "NPF_{GUID}" 的名称，但对于较新的 Npcap，名称是 "NPF_Loopback" */
int intf_get_loopback_name(char *buffer, int buf_size)
{
    HKEY hKey;
    DWORD type;
    int size = buf_size;
    int res = 0;

    memset(buffer, 0, buf_size);

    // 打开注册表键
    if (RegOpenKeyExA(HKEY_LOCAL_MACHINE, NPCAP_SERVICE_REGISTRY_KEY "\\Parameters", 0, KEY_READ, &hKey) == ERROR_SUCCESS)
    {
        // 查询注册表键值
        if (RegQueryValueExA(hKey, "LoopbackAdapter", 0, &type, (LPBYTE)buffer, &size) == ERROR_SUCCESS && type == REG_SZ)
        {
            res = 1;
        }
        else
        {
            res = 0;
        }

        // 关闭注册表键
        RegCloseKey(hKey);
    }
    else
    {
        res = 0;
    }

    return res;
}

static IP_ADAPTER_ADDRESSES*
_update_tables_for_npcap_loopback(IP_ADAPTER_ADDRESSES *p)
{
    IP_ADAPTER_ADDRESSES *a_prev = NULL;
    IP_ADAPTER_ADDRESSES *a;
    IP_ADAPTER_ADDRESSES *a_original_loopback_prev = NULL;
    IP_ADAPTER_ADDRESSES *a_original_loopback = NULL;
    IP_ADAPTER_ADDRESSES *a_npcap_loopback = NULL;
    static char npcap_loopback_name[1024] = {0};

    /* 不要每次都访问注册表。对于长时间运行的进程不理想，但对于 Nmap 可行。 */
    if (npcap_loopback_name[0] == '\0')
        g_has_npcap_loopback = intf_get_loopback_name(npcap_loopback_name, 1024);
    else if (g_has_npcap_loopback == 0)
        return p;

    if (!p)
        return p;

    /* 遍历地址，寻找 Windows 中的虚拟循环接口。 */
    # 遍历链表，查找适配器
    for (a = p; a != NULL; a = a->Next) {
        # 如果是虚拟回环接口，记录下来
        if (a->IfType == IF_TYPE_SOFTWARE_LOOPBACK) {
            /* Dummy loopback. Keep track of it. */
            a_original_loopback = a;
            a_original_loopback_prev = a_prev;
        }
        # 如果是旧版回环适配器，记录下来
        else if (strcmp(a->AdapterName, npcap_loopback_name + strlen(_DEVICE_PREFIX) - 1) == 0) {
            /* Legacy loopback adapter. The modern one doesn't show up in GetAdaptersAddresses. */
            a_npcap_loopback = a;
        }
        # 记录上一个适配器
        a_prev = a;
    }

    # 如果系统中没有回环适配器，返回原始链表
    if (!a_original_loopback)
        return p;
    # 标记系统中存在旧版回环适配器
    g_has_npcap_loopback = 1;
    # 如果没有找到旧版回环适配器，使用现代适配器名称
    if (!a_npcap_loopback) {
        /* Overwrite the name we got from the Registry, in case it's a broken legacy
         * install, in which case we'll never find the legacy adapter anyway. */
        # 用 NPF_Loopback 替换从注册表中获取的名称
        strlcpy(npcap_loopback_name, _DEVICE_PREFIX "NPF_Loopback", 1024);
        # 用 NPF_Loopback 替换系统自带回环适配器的适配器名称
        a_original_loopback->AdapterName = npcap_loopback_name + sizeof(_DEVICE_PREFIX) - 1;
        return p;
    }
    else {
        /* 如果找到了传统的环回适配器，则从系统的环回适配器中复制一些关键信息。 */
        a_npcap_loopback->IfType = a_original_loopback->IfType;
        a_npcap_loopback->FirstUnicastAddress = a_original_loopback->FirstUnicastAddress;
        a_npcap_loopback->FirstPrefix = a_original_loopback->FirstPrefix;
        memset(a_npcap_loopback->PhysicalAddress, 0, ETH_ADDR_LEN);
        /* 将原始环回适配器从列表中解除链接。我们将使用 Npcap 的适配器。 */
        if (a_original_loopback_prev) {
            a_original_loopback_prev->Next = a_original_loopback_prev->Next->Next;
            return p;
        }
        else if (a_original_loopback == p) {
            return a_original_loopback->Next;
        }
        else {
            return p;
        }
    }
# 刷新网络接口表
static int
_refresh_tables(intf_t *intf):
    IP_ADAPTER_ADDRESSES *p;  # 定义 IP_ADAPTER_ADDRESSES 指针变量 p
    DWORD ret;  # 定义 DWORD 类型变量 ret
    ULONG len;  # 定义 ULONG 类型变量 len

    p = NULL;  # 将 p 初始化为 NULL
    # 调用 GetAdaptersAddresses 函数，如果 len 太小，则返回 ERROR_BUFFER_OVERFLOW 并将 len 设置为所需大小
    # 在 Windows 2003 上，第一次使用太小的 len 调用函数会返回 ERROR_BUFFER_OVERFLOW，但第二次会返回 ERROR_INVALID_PARAMETER
    # 因此，第一次调用时使用一个较大的 len
    len = 16384;  # 初始化 len 为 16384
    do:
        free(p);  # 释放 p 指向的内存
        p = malloc(len);  # 分配 len 大小的内存给 p
        if (p == NULL):
            return (-1);  # 如果分配内存失败，则返回 -1
        ret = GetAdaptersAddresses(AF_UNSPEC, GAA_FLAG_INCLUDE_PREFIX | GAA_FLAG_SKIP_ANYCAST | GAA_FLAG_SKIP_MULTICAST, NULL, p, &len);  # 调用 GetAdaptersAddresses 函数
    while (ret == ERROR_BUFFER_OVERFLOW);  # 当返回值为 ERROR_BUFFER_OVERFLOW 时循环

    if (ret != NO_ERROR):  # 如果返回值不是 NO_ERROR
        free(p);  # 释放 p 指向的内存
        return (-1);  # 返回 -1
    p = _update_tables_for_npcap_loopback(p);  # 调用 _update_tables_for_npcap_loopback 函数
    intf->iftable = p;  # 将 intf->iftable 指向 p

    # 将 "不友好" 的 win32 接口索引映射到我们的接口索引
    for (p = intf->iftable; p != NULL; p = p->Next):  # 遍历 intf->iftable
        int type;  # 定义 int 类型变量 type
        type = _if_type_canonicalize(p->IfType);  # 调用 _if_type_canonicalize 函数
        if (type < MIB_IF_TYPE_MAX):  # 如果 type 小于 MIB_IF_TYPE_MAX
            _ifcombo_add(&intf->ifcombo[type], p->IfIndex, p->Ipv6IfIndex);  # 调用 _ifcombo_add 函数
        else:
            return (-1);  # 返回 -1
    return (0);  # 返回 0
# 根据接口和设备名称查找适配器地址
_find_adapter_address(intf_t *intf, const char *device)
{
    IP_ADAPTER_ADDRESSES *a;  # 定义 IP 适配器地址指针
    char *p = (char *)device;  # 将设备名称转换为字符指针
    int n, type = _ifcombo_type(device);  # 定义整型变量 n 和 type，并调用 _ifcombo_type 函数

    while (isalpha((int) (unsigned char) *p)) p++;  # 循环直到 p 指向的字符不是字母
    n = atoi(p);  # 将 p 指向的字符串转换为整型数值

    for (a = intf->iftable; a != NULL; a = a->Next) {  # 遍历接口的适配器地址链表
        if ( intf->ifcombo[type].idx != NULL &&  # 如果接口类型的索引不为空
            intf->ifcombo[type].idx[n].ipv4 == a->IfIndex &&  # 如果接口类型的索引的 IPv4 地址等于当前适配器地址的 IfIndex
            intf->ifcombo[type].idx[n].ipv6 == a->Ipv6IfIndex) {  # 如果接口类型的索引的 IPv6 地址等于当前适配器地址的 Ipv6IfIndex
            return a;  # 返回当前适配器地址
        }
    }

    return NULL;  # 如果未找到匹配的适配器地址，则返回空指针
}

# 根据索引查找适配器地址
static IP_ADAPTER_ADDRESSES *
_find_adapter_address_by_index(intf_t *intf, int af, unsigned int index)
{
    IP_ADAPTER_ADDRESSES *a;  # 定义 IP 适配器地址指针

    for (a = intf->iftable; a != NULL; a = a->Next) {  # 遍历接口的适配器地址链表
        if (af == AF_INET && index == a->IfIndex)  # 如果地址族为 IPv4 并且索引等于当前适配器地址的 IfIndex
            return a;  # 返回当前适配器地址
        if (af == AF_INET6 && index == a->Ipv6IfIndex)  # 如果地址族为 IPv6 并且索引等于当前适配器地址的 Ipv6IfIndex
            return a;  # 返回当前适配器地址
    }

    return NULL;  # 如果未找到匹配的适配器地址，则返回空指针
}

# 打开接口
intf_t *
intf_open(void)
{
    return (calloc(1, sizeof(intf_t)));  # 分配内存空间并返回指向接口结构体的指针
}

# 获取接口信息
int
intf_get(intf_t *intf, struct intf_entry *entry)
{
    IP_ADAPTER_ADDRESSES *a;  # 定义 IP 适配器地址指针
    
    if (_refresh_tables(intf) < 0)  # 如果刷新表格失败
        return (-1);  # 返回错误码
    
    a = _find_adapter_address(intf, entry->intf_name);  # 根据接口名称查找适配器地址
    if (a == NULL)  # 如果未找到适配器地址
        return (-1);  # 返回错误码

    _adapter_address_to_entry(intf, a, entry);  # 将适配器地址转换为接口信息
    
    return (0);  # 返回成功
}

# 根据索引获取接口信息
int
intf_get_index(intf_t *intf, struct intf_entry *entry, int af, unsigned int index)
{
    IP_ADAPTER_ADDRESSES *a;  # 定义 IP 适配器地址指针

    if (_refresh_tables(intf) < 0)  # 如果刷新表格失败
        return (-1);  # 返回错误码

    a = _find_adapter_address_by_index(intf, af, index);  # 根据索引查找适配器地址
    if (a == NULL)  # 如果未找到适配器地址
        return (-1);  # 返回错误码

    _adapter_address_to_entry(intf, a, entry);  # 将适配器地址转换为接口信息

    return (0);  # 返回成功
}

# 获取源接口信息
int
intf_get_src(intf_t *intf, struct intf_entry *entry, struct addr *src)
{
    IP_ADAPTER_ADDRESSES *a;  # 定义 IP 适配器地址指针
    IP_ADAPTER_UNICAST_ADDRESS *addr;  # 定义 IP 适配器单播地址指针

    if (src->addr_type != ADDR_TYPE_IP) {  # 如果地址类型不是 IP
        errno = EINVAL;  # 设置错误码为无效参数
        return (-1);  # 返回错误码
    }
    if (_refresh_tables(intf) < 0)  # 如果刷新表格失败
        return (-1);  # 返回错误码
    # 遍历接口表中的每一个接口
    for (a = intf->iftable; a != NULL; a = a->Next) {
        # 遍历每个接口的单播地址
        for (addr = a->FirstUnicastAddress; addr != NULL; addr = addr->Next) {
            # 创建一个用于存储网络地址的结构体
            struct addr dnet_addr;
            # 将地址转换为网络地址结构体
            addr_ston(addr->Address.lpSockaddr, &dnet_addr);
            # 比较网络地址是否与给定源地址相同
            if (addr_cmp(&dnet_addr, src) == 0) {
                # 将接口、地址和结果存储到指定的条目中
                _adapter_address_to_entry(intf, a, entry);
                # 返回成功
                return (0);
            }
        }
    }
    # 设置错误码为设备不存在
    errno = ENXIO;
    # 返回失败
    return (-1);
}

int
intf_get_dst(intf_t *intf, struct intf_entry *entry, struct addr *dst)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 设置最后发生的错误为 ERROR_NOT_SUPPORTED
    SetLastError(ERROR_NOT_SUPPORTED);
    // 返回 -1
    return (-1);
}

int
intf_set(intf_t *intf, const struct intf_entry *entry)
{
    /*
     * XXX - could set interface up/down via SetIfEntry(),
     * but what about the rest of the configuration? :-(
     * {Add,Delete}IPAddress for 2000/XP only
     */
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 设置最后发生的错误为 ERROR_NOT_SUPPORTED
    SetLastError(ERROR_NOT_SUPPORTED);
    // 返回 -1
    return (-1);
}

int
intf_loop(intf_t *intf, intf_handler callback, void *arg)
{
    IP_ADAPTER_ADDRESSES *a;
    struct intf_entry *entry;
    u_char ebuf[1024];
    int ret = 0;

    // 如果刷新表失败，返回 -1
    if (_refresh_tables(intf) < 0)
        return (-1);
    
    // 将 ebuf 强制转换为 struct intf_entry 类型
    entry = (struct intf_entry *)ebuf;
    
    // 遍历接口表
    for (a = intf->iftable; a != NULL; a = a->Next) {
        // 设置 entry 的长度
        entry->intf_len = sizeof(ebuf);
        // 将适配器地址转换为 entry
        _adapter_address_to_entry(intf, a, entry);
        // 调用回调函数，如果返回值不为 0，跳出循环
        if ((ret = (*callback)(entry, arg)) != 0)
            break;
    }
    return (ret);
}

intf_t *
intf_close(intf_t *intf)
{
    int i;

    // 如果 intf 不为空
    if (intf != NULL) {
        // 释放 intf->ifcombo[i].idx
        for (i = 0; i < MIB_IF_TYPE_MAX; i++) {
            if (intf->ifcombo[i].idx)
                free(intf->ifcombo[i].idx);
        }
        // 释放 intf->iftable
        if (intf->iftable)
            free(intf->iftable);
        // 释放 intf
        free(intf);
    }
    // 返回 NULL
    return (NULL);
}

/* Converts a libdnet interface name to its pcap equivalent. The pcap name is
   stored in pcapdev up to a length of pcapdevlen, including the terminating
   '\0'. Returns -1 on error. */
int
intf_get_pcap_devname_cached(const char *intf_name, char *pcapdev, int pcapdevlen, int refresh)
{
    IP_ADAPTER_ADDRESSES *a;
    static pcap_if_t *pcapdevs = NULL;
    pcap_if_t *pdev;
    intf_t *intf;
    char errbuf[PCAP_ERRBUF_SIZE];

    // 打开接口
    if ((intf = intf_open()) == NULL)
        return (-1);
    // 如果刷新表失败，关闭接口并返回 -1
    if (_refresh_tables(intf) < 0) {
        intf_close(intf);
        return (-1);
    }
    // 查找适配器地址
    a = _find_adapter_address(intf, intf_name);
    // 如果 a 为空指针，则关闭接口并返回 -1
    if (a == NULL) {
        intf_close(intf);
        return (-1);
    }

    // 如果需要刷新，则释放所有设备并将 pcapdevs 置为空
    if (refresh) {
        pcap_freealldevs(pcapdevs);
        pcapdevs = NULL;
    }

    // 如果 pcapdevs 为空，则查找所有设备
    if (pcapdevs == NULL) {
        if (pcap_findalldevs(&pcapdevs, errbuf) == -1) {
            intf_close(intf);
            return (-1);
        }
    }

    /* 循环遍历所有 pcap 设备直到找到匹配项 */
    for (pdev = pcapdevs; pdev != NULL; pdev = pdev->next) {
        char *name;

        // 如果设备名为空或长度小于 _DEVICE_PREFIX 的大小，则继续下一次循环
        if (pdev->name == NULL || strlen(pdev->name) < sizeof(_DEVICE_PREFIX))
            continue;
        /* "\\Device\\NPF_{GUID}"
         * "\\Device\\NPF_Loopback"
         * 找到设备前缀后的 '{'
         */
        name = strchr(pdev->name + sizeof(_DEVICE_PREFIX) - 1, '{');
        if (name == NULL) {
            /* 如果没有 GUID，则匹配整个设备名 */
            name = pdev->name + sizeof(_DEVICE_PREFIX) - 1;
        }
        // 如果找到与 a->AdapterName 相同的设备名，则跳出循环
        if (strcmp(name, a->AdapterName) == 0)
            break;
    }
    // 如果找到匹配的设备，则将设备名拷贝到 pcapdev 中
    if (pdev != NULL)
        strlcpy(pcapdev, pdev->name, pcapdevlen);
    // 关闭接口
    intf_close(intf);
    // 如果未找到匹配的设备，则返回 -1，否则返回 0
    if (pdev == NULL)
        return -1;
    else
        return 0;
# 结束函数定义
}
# 定义一个返回类型为 int 的函数 intf_get_pcap_devname，参数为 intf_name, pcapdev, pcapdevlen
int
intf_get_pcap_devname(const char *intf_name, char *pcapdev, int pcapdevlen)
{
  # 调用 intf_get_pcap_devname_cached 函数，并返回结果
  return intf_get_pcap_devname_cached(intf_name, pcapdev, pcapdevlen, 0);
}
```