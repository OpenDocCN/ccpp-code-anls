# `nmap\libdnet-stripped\src\eth-win32.c`

```
/*
 * eth-win32.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-win32.c 613 2005-09-26 02:46:57Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

/* XXX - VC++ 6.0 的错误 */
#define sockaddr_storage sockaddr
#undef sockaddr_storage

#include <errno.h>
#include <stdlib.h>

#include "dnet.h"
#include <winsock2.h>
#include "pcap.h"
#include <Packet32.h>
#include <Ntddndis.h>

/* 从 Npcap 的 Loopback.h 中获取 */
/*
 * * DLT_NULL 头部的结构。
 * */
typedef struct _DLT_NULL_HEADER
{
    UINT  null_type;
} DLT_NULL_HEADER, *PDLT_NULL_HEADER;

/*
 * * 组合头部的长度。
 * */
#define DLT_NULL_HDR_LEN  sizeof(DLT_NULL_HEADER)

/*
 * * DLT_NULL (Loopback) 头部中的类型。
 * */
#define DLTNULLTYPE_IP    0x00000002  /* IP 协议 */
#define DLTNULLTYPE_IPV6  0x00000018 /* IPv6 */
/* END Loopback.h */

struct eth_handle {
    LPADAPTER     lpa;
    LPPACKET     pkt;
    NetType    type;
};

eth_t *
eth_open(const char *device)
{
    eth_t *eth;
    char pcapdev[128];

    // 获取设备的 pcap 名称
    if (eth_get_pcap_devname(device, pcapdev, sizeof(pcapdev)) != 0)
        return (NULL);

    // 分配并初始化 eth 结构
    if ((eth = calloc(1, sizeof(*eth))) == NULL)
        return (NULL);
    // 打开适配器
    eth->lpa = PacketOpenAdapter(pcapdev);
    if (eth->lpa == NULL) {
        eth_close(eth);
        return (NULL);
    }
    // 设置缓冲区大小
    PacketSetBuff(eth->lpa, 512000);
    // 分配数据包
    eth->pkt = PacketAllocatePacket();
    if (eth->pkt == NULL) {
        eth_close(eth);
        return NULL;
    }
    // 获取网络类型
    if (!PacketGetNetType(eth->lpa, &eth->type)) {
      eth_close(eth);
      return NULL;
  }

    return (eth);
}

ssize_t
eth_send(eth_t *eth, const void *buf, size_t len)
{
  /* 14 字节的以太网头部，但 DLT_NULL 是 4 字节的头部。跳过差异 */
  DLT_NULL_HEADER *hdr = (DLT_NULL_HEADER *)((uint8_t *)buf + ETH_HDR_LEN - DLT_NULL_HDR_LEN);
  if (eth->type.LinkType == NdisMediumNull) {
    # 根据接收到的以太网数据包中的协议类型进行判断和处理
    switch (ntohs(((struct eth_hdr *)buf)->eth_type)) {
      # 如果是 IP 协议类型
      case ETH_TYPE_IP:
        # 设置数据链路层类型为 DLTNULLTYPE_IP
        hdr->null_type = DLTNULLTYPE_IP;
        break;
      # 如果是 IPv6 协议类型
      case ETH_TYPE_IPV6:
        # 设置数据链路层类型为 DLTNULLTYPE_IPV6
        hdr->null_type = DLTNULLTYPE_IPV6;
        break;
      # 其他情况
      default:
        # 设置数据链路层类型为 0
        hdr->null_type = 0;
        break;
    }
    # 初始化数据包，根据以太网数据包的内容和长度
    PacketInitPacket(eth->pkt, (void *)((uint8_t *)buf + ETH_HDR_LEN - DLT_NULL_HDR_LEN), (UINT) (len - ETH_HDR_LEN + DLT_NULL_HDR_LEN));
    # 发送数据包
    PacketSendPacket(eth->lpa, eth->pkt, TRUE);
  }
  # 如果不是以太网数据包
  else {
    # 初始化数据包，根据数据包的内容和长度
    PacketInitPacket(eth->pkt, (void *)buf, (UINT) len);
    # 发送数据包
    PacketSendPacket(eth->lpa, eth->pkt, TRUE);
  }
    # 返回数据包的长度
    return (ssize_t)(len);
# 释放以太网对象的资源并返回空指针
eth_t *
eth_close(eth_t *eth)
{
    # 检查以太网对象是否为空
    if (eth != NULL) {
        # 释放以太网对象中的数据包
        if (eth->pkt != NULL)
            PacketFreePacket(eth->pkt);
        # 关闭以太网适配器
        if (eth->lpa != NULL)
            PacketCloseAdapter(eth->lpa);
        # 释放以太网对象的内存
        free(eth);
    }
    # 返回空指针
    return (NULL);
}

# 获取以太网适配器的物理地址
int
eth_get(eth_t *eth, eth_addr_t *ea)
{
    PACKET_OID_DATA *data;
    u_char buf[512];

    # 将 buf 强制转换为 PACKET_OID_DATA 类型
    data = (PACKET_OID_DATA *)buf;
    # 设置 OID 为获取当前地址
    data->Oid = OID_802_3_CURRENT_ADDRESS;
    # 设置数据长度为以太网地址长度
    data->Length = ETH_ADDR_LEN;

    # 发送获取物理地址的请求，并将结果存储在 ea 中
    if (PacketRequest(eth->lpa, FALSE, data) == TRUE) {
        memcpy(ea, data->Data, ETH_ADDR_LEN);
        return (0);
    }
    # 获取失败，返回 -1
    return (-1);
}

# 设置以太网适配器的物理地址
int
eth_set(eth_t *eth, const eth_addr_t *ea)
{
    PACKET_OID_DATA *data;
    u_char buf[512];

    # 将 buf 强制转换为 PACKET_OID_DATA 类型
    data = (PACKET_OID_DATA *)buf;
    # 设置 OID 为设置当前地址
    data->Oid = OID_802_3_CURRENT_ADDRESS;
    # 将以太网地址拷贝到数据中
    memcpy(data->Data, ea, ETH_ADDR_LEN);
    # 设置数据长度为以太网地址长度
    data->Length = ETH_ADDR_LEN;
    
    # 发送设置物理地址的请求，并返回结果
    if (PacketRequest(eth->lpa, TRUE, data) == TRUE)
        return (0);
    
    # 设置失败，返回 -1
    return (-1);
}

# 获取 pcap 设备名称
int
eth_get_pcap_devname(const char *intf_name, char *pcapdev, int pcapdevlen)
{
    # 调用 intf_get_pcap_devname 函数获取 pcap 设备名称
    return intf_get_pcap_devname(intf_name, pcapdev, pcapdevlen);
}
```