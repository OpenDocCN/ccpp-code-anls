# `nmap\libdnet-stripped\src\route-win32.c`

```
/*
 * route-win32.c
 *
 * 版权所有 (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-win32.c 589 2005-02-15 07:11:32Z dugsong $
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

typedef DWORD (WINAPI *GETIPFORWARDTABLE2)(ADDRESS_FAMILY, PMIB_IPFORWARD_TABLE2 *);

struct route_handle {
    HINSTANCE iphlpapi;
    MIB_IPFORWARDTABLE *ipftable;
    MIB_IPFORWARD_TABLE2 *ipftable2;
};

route_t *
route_open(void)
{
    route_t *r;

    r = calloc(1, sizeof(route_t)); // 分配内存空间给路由对象
    if (r == NULL)
        return NULL;
    r->iphlpapi = GetModuleHandle("iphlpapi.dll"); // 获取iphlpapi.dll模块的句柄

    return r; // 返回路由对象
}

int
route_add(route_t *route, const struct route_entry *entry)
{
    MIB_IPFORWARDROW ipfrow; // 定义MIB_IPFORWARDROW结构体
    struct addr net; // 定义网络地址结构体

    memset(&ipfrow, 0, sizeof(ipfrow)); // 将ipfrow结构体清零

    if (GetBestInterface(entry->route_gw.addr_ip,
        &ipfrow.dwForwardIfIndex) != NO_ERROR) // 获取最佳接口索引
        return (-1);

    if (addr_net(&entry->route_dst, &net) < 0 || // 获取目的地址的网络地址
        net.addr_type != ADDR_TYPE_IP)
        return (-1);
    
    ipfrow.dwForwardDest = net.addr_ip; // 设置目的地址
    addr_btom(entry->route_dst.addr_bits,
        &ipfrow.dwForwardMask, IP_ADDR_LEN); // 将地址位数转换为子网掩码
    ipfrow.dwForwardNextHop = entry->route_gw.addr_ip; // 设置下一跳地址
    ipfrow.dwForwardType = 4;    /* XXX - next hop != final dest */ // 设置路由类型
    ipfrow.dwForwardProto = 3;    /* XXX - MIB_PROTO_NETMGMT */ // 设置路由协议
    
    if (CreateIpForwardEntry(&ipfrow) != NO_ERROR) // 创建IP路由表项
        return (-1);
    
    return (0); // 返回成功
}

int
route_delete(route_t *route, const struct route_entry *entry)
{
    MIB_IPFORWARDROW ipfrow; // 定义MIB_IPFORWARDROW结构体
    DWORD mask; // 定义子网掩码
    
    if (entry->route_dst.addr_type != ADDR_TYPE_IP ||
        GetBestRoute(entry->route_dst.addr_ip,
        IP_ADDR_ANY, &ipfrow) != NO_ERROR) // 获取最佳路由
        return (-1);

    addr_btom(entry->route_dst.addr_bits, &mask, IP_ADDR_LEN); // 将地址位数转换为子网掩码
}
    # 如果目的地址或子网掩码与路由表中的不匹配
    if (ipfrow.dwForwardDest != entry->route_dst.addr_ip ||
        ipfrow.dwForwardMask != mask) {
        # 设置错误码为设备不存在
        errno = ENXIO;
        # 设置最后发生的错误为没有数据
        SetLastError(ERROR_NO_DATA);
        # 返回-1表示失败
        return (-1);
    }
    # 如果删除 IP 转发条目失败
    if (DeleteIpForwardEntry(&ipfrow) != NO_ERROR)
        # 返回-1表示失败
        return (-1);
    
    # 返回0表示成功
    return (0);
}

int
route_get(route_t *route, struct route_entry *entry)
{
    MIB_IPFORWARDROW ipfrow;  // 定义 MIB_IPFORWARDROW 结构体变量
    DWORD mask;  // 定义 DWORD 类型的变量 mask
    intf_t *intf;  // 定义指向 intf_t 结构体的指针变量 intf
    struct intf_entry intf_entry;  // 定义 struct intf_entry 结构体变量 intf_entry

    if (entry->route_dst.addr_type != ADDR_TYPE_IP ||  // 如果目的地址类型不是 IP 地址
        GetBestRoute(entry->route_dst.addr_ip,  // 调用 GetBestRoute 函数
        IP_ADDR_ANY, &ipfrow) != NO_ERROR)  // 如果获取路由失败
        return (-1);  // 返回 -1

    if (ipfrow.dwForwardProto == 2 &&    /* XXX - MIB_IPPROTO_LOCAL */  // 如果协议类型为 2 并且下一跳地址不是回环地址
        (ipfrow.dwForwardNextHop|IP_CLASSA_NET) !=  // 如果下一跳地址与 IP_CLASSA_NET 不相等
        (IP_ADDR_LOOPBACK|IP_CLASSA_NET) &&
        !IP_LOCAL_GROUP(ipfrow.dwForwardNextHop)) {  // 如果不是本地组地址
        errno = ENXIO;  // 设置错误码为 ENXIO
        SetLastError(ERROR_NO_DATA);  // 设置最后发生的错误为 ERROR_NO_DATA
        return (-1);  // 返回 -1
    }
    addr_btom(entry->route_dst.addr_bits, &mask, IP_ADDR_LEN);  // 调用 addr_btom 函数

    entry->route_gw.addr_type = ADDR_TYPE_IP;  // 设置网关地址类型为 IP 地址
    entry->route_gw.addr_bits = IP_ADDR_BITS;  // 设置网关地址位数为 IP 地址位数
    entry->route_gw.addr_ip = ipfrow.dwForwardNextHop;  // 设置网关地址为下一跳地址
    entry->metric = ipfrow.dwForwardMetric1;  // 设置路由的度量值为下一跳的度量值

    entry->intf_name[0] = '\0';  // 将接口名称的第一个字符设置为空字符
    intf = intf_open();  // 调用 intf_open 函数打开接口
    if (intf_get_index(intf, &intf_entry,  // 如果成功获取接口索引
        AF_INET, ipfrow.dwForwardIfIndex) == 0) {
        strlcpy(entry->intf_name, intf_entry.intf_name, sizeof(entry->intf_name));  // 将接口名称拷贝到 entry->intf_name 中
    }
    intf_close(intf);  // 关闭接口
    return (0);  // 返回 0
}

static int
route_loop_getipforwardtable(route_t *r, route_handler callback, void *arg)
{
     struct route_entry entry;  // 定义 struct route_entry 结构体变量 entry
    intf_t *intf;  // 定义指向 intf_t 结构体的指针变量 intf
    ULONG len;  // 定义 ULONG 类型的变量 len
    int i, ret;  // 定义整型变量 i 和 ret

    for (len = sizeof(r->ipftable[0]); ; ) {  // 循环获取 IP 转发表
        if (r->ipftable)  // 如果 r->ipftable 不为空
            free(r->ipftable);  // 释放 r->ipftable 的内存
        r->ipftable = malloc(len);  // 分配 len 大小的内存给 r->ipftable
        if (r->ipftable == NULL)  // 如果分配内存失败
            return (-1);  // 返回 -1
        ret = GetIpForwardTable(r->ipftable, &len, FALSE);  // 调用 GetIpForwardTable 函数获取 IP 转发表
        if (ret == NO_ERROR)  // 如果获取成功
            break;  // 跳出循环
        else if (ret != ERROR_INSUFFICIENT_BUFFER)  // 如果获取失败且不是缓冲区不足的错误
            return (-1);  // 返回 -1
    }

    intf = intf_open();  // 打开接口
    ret = 0;  // 设置 ret 为 0
    # 遍历路由表中的每一条路由记录
    for (i = 0; i < (int)r->ipftable->dwNumEntries; i++) {
        # 创建一个接口条目结构体
        struct intf_entry intf_entry;

        # 设置路由目的地址的类型和位数
        entry.route_dst.addr_type = ADDR_TYPE_IP;
        entry.route_dst.addr_bits = IP_ADDR_BITS;

        # 设置路由网关地址的类型和位数
        entry.route_gw.addr_type = ADDR_TYPE_IP;
        entry.route_gw.addr_bits = IP_ADDR_BITS;

        # 设置路由目的地址为路由表中的目的地址
        entry.route_dst.addr_ip = r->ipftable->table[i].dwForwardDest;
        # 将路由表中的子网掩码转换为地址位数
        addr_mtob(&r->ipftable->table[i].dwForwardMask, IP_ADDR_LEN,
            &entry.route_dst.addr_bits);
        # 设置路由网关地址为路由表中的下一跳地址
        entry.route_gw.addr_ip =
            r->ipftable->table[i].dwForwardNextHop;
        # 设置路由的度量值为路由表中的度量值
        entry.metric = r->ipftable->table[i].dwForwardMetric1;

        # 查找接口名称
        entry.intf_name[0] = '\0';
        # 设置接口条目结构体的长度
        intf_entry.intf_len = sizeof(intf_entry);
        # 如果成功获取到接口信息
        if (intf_get_index(intf, &intf_entry,
            AF_INET, r->ipftable->table[i].dwForwardIfIndex) == 0) {
            # 将接口名称复制到路由条目中
            strlcpy(entry.intf_name, intf_entry.intf_name, sizeof(entry.intf_name));
        }
        
        # 调用回调函数处理路由条目，如果返回值不为0则跳出循环
        if ((ret = (*callback)(&entry, arg)) != 0)
            break;
    }

    # 关闭接口
    intf_close(intf);

    # 返回处理结果
    return ret;
# 定义一个静态函数，用于获取 IP 转发表的信息
static int
route_loop_getipforwardtable2(GETIPFORWARDTABLE2 GetIpForwardTable2,
    route_t *r, route_handler callback, void *arg)
{
    # 定义路由表条目和接口指针
    struct route_entry entry;
    intf_t *intf;
    ULONG i;
    int ret;
    
    # 获取 IP 转发表信息
    ret = GetIpForwardTable2(AF_UNSPEC, &r->ipftable2);
    if (ret != NO_ERROR)
        return (-1);

    # 打开接口
    intf = intf_open();

    # 初始化返回值
    ret = 0;
    # 遍历 IP 转发表的每一条目
    for (i = 0; i < r->ipftable2->NumEntries; i++) {
        # 定义接口条目和 IP 转发表行
        struct intf_entry intf_entry;
        MIB_IPFORWARD_ROW2 *row;
        MIB_IPINTERFACE_ROW ifrow;
        ULONG metric;

        # 获取当前行的信息
        row = &r->ipftable2->Table[i];
        # 将目的地址和下一跳地址转换成网络字节序
        addr_ston((struct sockaddr *) &row->DestinationPrefix.Prefix, &entry.route_dst);
        entry.route_dst.addr_bits = row->DestinationPrefix.PrefixLength;
        addr_ston((struct sockaddr *) &row->NextHop, &entry.route_gw);

        # 查找接口名称
        entry.intf_name[0] = '\0';
        intf_entry.intf_len = sizeof(intf_entry);
        if (intf_get_index(intf, &intf_entry,
            row->DestinationPrefix.Prefix.si_family,
            row->InterfaceIndex) == 0) {
            strlcpy(entry.intf_name, intf_entry.intf_name, sizeof(entry.intf_name));
        }

        # 获取接口信息
        ifrow.Family = row->DestinationPrefix.Prefix.si_family;
        ifrow.InterfaceLuid = row->InterfaceLuid;
        ifrow.InterfaceIndex = row->InterfaceIndex;
        if (GetIpInterfaceEntry(&ifrow) != NO_ERROR) {
            return (-1);
        }
        metric = ifrow.Metric + row->Metric;
        if (metric < INT_MAX)
            entry.metric = metric;
        else
            entry.metric = INT_MAX;
        
        # 调用回调函数处理当前条目
        if ((ret = (*callback)(&entry, arg)) != 0)
            break;
    }

    # 关闭接口
    intf_close(intf);

    # 返回处理结果
    return ret;
}

# 定义一个函数，用于遍历路由表
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    GETIPFORWARDTABLE2 GetIpForwardTable2;

    # GetIpForwardTable2 只在 Vista 及更高版本可用，需要动态加载
    GetIpForwardTable2 = NULL;
    # 如果 r->iphlpapi 不为空，则将 GetIpForwardTable2 设置为 GetProcAddress(r->iphlpapi, "GetIpForwardTable2") 的返回值
    if (r->iphlpapi != NULL)
        GetIpForwardTable2 = (GETIPFORWARDTABLE2) GetProcAddress(r->iphlpapi, "GetIpForwardTable2");

    # 如果 GetIpForwardTable2 为空，则调用 route_loop_getipforwardtable 函数并返回结果
    if (GetIpForwardTable2 == NULL)
        return route_loop_getipforwardtable(r, callback, arg);
    # 否则调用 route_loop_getipforwardtable2 函数并返回结果
    else
        return route_loop_getipforwardtable2(GetIpForwardTable2, r, callback, arg);
# 释放路由表资源并关闭路由
route_t *
route_close(route_t *r)
{
    # 检查路由是否存在
    if (r != NULL) {
        # 释放ipftable资源
        if (r->ipftable != NULL)
            free(r->ipftable);
        # 释放ipftable2资源
        if (r->ipftable2 != NULL)
            FreeMibTable(r->ipftable2);
        # 释放路由资源
        free(r);
    }
    # 返回空指针
    return (NULL);
}
```