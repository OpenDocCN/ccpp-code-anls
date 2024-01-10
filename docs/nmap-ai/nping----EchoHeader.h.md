# `nmap\nping\EchoHeader.h`

```
#ifndef __ECHOHEADER_H__
#define __ECHOHEADER_H__ 1

#include "nping.h"

#define ECHO_CURRENT_PROTO_VER  0x01

/* Lengths */
// 定义常用 NEP 头部长度
#define STD_NEP_HEADER_LEN 16      
// 消息认证码的长度
#define MAC_LENGTH 32              
// 客户端/服务器随机数的长度
#define NONCE_LEN 32               
// Partner IP 字段的长度
#define PARTNER_IP_LEN 16          
// 数据包规范的长度
#define PACKETSPEC_FIELD_LEN 108   
// NEP_ERROR 消息字符串的长度
#define ERROR_MSG_LEN 80           

// 握手消息的长度
#define NEP_HANDSHAKE_SERVER_LEN  96
#define NEP_HANDSHAKE_CLIENT_LEN 144
#define NEP_HANDSHAKE_FINAL_LEN  112
#define NEP_PACKETSPEC_LEN 160
#define NEP_READY_LEN 48
#define NEP_ERROR_LEN 128

// 回显数据包头部的长度
#define ECHOED_PKT_HEADER_LEN 4    
// 回显数据包的最大长度
#define MAX_ECHOED_PACKET_LEN 9212 
// 数据的最大长度
#define MAX_DATA_LEN (ECHOED_PKT_HEADER_LEN + MAX_ECHOED_PACKET_LEN + MAC_LENGTH)
// NEP_ECHO 消息的最小长度
#define NEP_ECHO_MIN_LEN 64
// NEP_ECHO 消息的最大长度
#define NEP_ECHO_MAX_LEN ( STD_NEP_HEADER_LEN + MAX_DATA_LEN )
// NEP 数据包的最大长度
#define MAX_NEP_PACKET_LENGTH  ( STD_NEP_HEADER_LEN + MAX_DATA_LEN )

/* Message types */
// 消息类型
#define TYPE_NEP_HANDSHAKE_SERVER   0x01
#define TYPE_NEP_HANDSHAKE_CLIENT   0x02
#define TYPE_NEP_HANDSHAKE_FINAL    0x03
#define TYPE_NEP_PACKET_SPEC        0x04
#define TYPE_NEP_READY              0x05
#define TYPE_NEP_ECHO               0x06
#define TYPE_NEP_ERROR              0x07

/* Field specifiers */
// 字段标识符
#define PSPEC_IPv4_TOS       0xA0
#define PSPEC_IPv4_ID        0xA1
#define PSPEC_IPv4_FRAGOFF   0xA2
#define PSPEC_IPv4_PROTO     0xA3
#define PSPEC_IPv6_TCLASS    0xB0
#define PSPEC_IPv6_FLOW      0xB1
#define PSPEC_IPv6_NHDR      0xB2
#define PSPEC_TCP_SPORT      0xC0
#define PSPEC_TCP_DPORT      0xC1
#define PSPEC_TCP_SEQ        0xC2
#define PSPEC_TCP_ACK        0xC3
#define PSPEC_TCP_FLAGS      0xC4
#define PSPEC_TCP_WIN        0xC5
# 定义TCP协议的URP字段
#define PSPEC_TCP_URP        0xC6
# 定义ICMP协议的类型字段
#define PSPEC_ICMP_TYPE      0xD0
# 定义ICMP协议的代码字段
#define PSPEC_ICMP_CODE      0xD1
# 定义UDP协议的源端口字段
#define PSPEC_UDP_SPORT      0xE0
# 定义UDP协议的目的端口字段
#define PSPEC_UDP_DPORT      0xE1
# 定义UDP协议的长度字段
#define PSPEC_UDP_LEN        0xE2
# 定义数据包的魔术字段
#define PSPEC_PAYLOAD_MAGIC  0xFF

# 定义NEP_PACKET_SPEC协议的TCP标识符
#define PSPEC_PROTO_TCP      0x06
# 定义NEP_PACKET_SPEC协议的UDP标识符
#define PSPEC_PROTO_UDP      0x11
# 定义NEP_PACKET_SPEC协议的ICMP标识符
#define PSPEC_PROTO_ICMP     0x01

# 定义数据链路层类型为无数据链路头
#define DLT_NODATALINKHEADERINCLUDED 0x0000
# 定义了一个名为 EchoHeader 的类，继承自 ApplicationLayerElement
class EchoHeader : public ApplicationLayerElement {

    # 定义了一个私有方法 getFieldLength，接收一个无符号8位整数作为参数
    private:
        int getFieldLength(u8 field);

};

# 结束类定义
#endif /* __ECHOHEADER_H__ */
```