# `nmap\libpcap\pcap-rpcap-int.h`

```cpp
/*
 * 版权声明，版权所有
 */
#ifndef __PCAP_RPCAP_INT_H__
#define __PCAP_RPCAP_INT_H__

#include "pcap.h"
#include "sockutils.h"    /* 需要一些在这里使用的结构（如 SOCKET、sockaddr_in） */
# 定义了一个文件，包含了 RPCAP 客户端和服务器使用的所有定义，除了协议定义
# 警告：所有允许返回包含错误描述的缓冲区的 RPCAP 函数可以返回最大 PCAP_ERRBUF_SIZE 个字符
# 但是不能保证字符串会被零终止
# 最佳实践是将 errbuf 变量定义为大小为 'PCAP_ERRBUF_SIZE+1' 的 char 类型，并手动在缓冲区末尾插入终止字符
# 这将确保即使使用 printf() 在屏幕上显示错误，也不会发生缓冲区溢出

# 通用定义/类型定义用于 RPCAP 协议
# 用于发送接收数据包的套接字函数使用的缓冲区
# 如果您计划拥有大于此值的消息，则必须增加它
#define RPCAP_NETBUF_SIZE 64000

# 导出的函数原型
# 创建 RPCAP 头部
void rpcap_createhdr(struct rpcap_header *header, uint8 type, uint16 value, uint32 length);
# 发送错误消息
int rpcap_senderror(SOCKET sock, char *error, unsigned short errcode, char *errbuf);
```