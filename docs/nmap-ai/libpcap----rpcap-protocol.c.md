# `nmap\libpcap\rpcap-protocol.c`

```cpp
/*
 * 版权声明
 * 2002年至2005年，NetGroup，意大利都灵理工大学（Politecnico di Torino）保留所有权利
 * 2005年至2008年，CACE Technologies，美国加利福尼亚州戴维斯市保留所有权利
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都需要满足以下条件：
 *
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明。
 * 3. 不能使用都灵理工大学（Politecnico di Torino）、CACE Technologies或其贡献者的名称来认可或推广从本软件衍生的产品，除非事先得到特定的书面许可。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）承担责任，即使已被告知可能发生此类损害的可能性。
 */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <string.h>        /* 用于 strlen() 等函数 */
#include <stdlib.h>        /* 用于 malloc()、free() 等函数 */
#include <stdarg.h>        /* 用于具有可变参数的函数 */
#include <errno.h>        /* 用于 errno 变量 */
/*
 * 包含必要的头文件
 */
#include "sockutils.h"
#include "portability.h"
#include "rpcap-protocol.h"
#include <pcap/pcap.h>

/*
 * 该文件包含了 rpcap 客户端和 rpcap 守护进程都会使用的函数。
 */

/*
 * 该函数向对等方发送一个 RPCAP 错误。
 * 当主程序检测到错误时，必须调用此函数。
 * 它将发送给对等方用户指定的 'buffer'。
 * 该函数 *不会* 请求 RPCAP CLOSE 连接。程序必须显式发送 CLOSE 命令，
 * 因为我们不知道错误是否可以以某种方式恢复，或者是否是不可恢复的。
 *
 * \param sock: 当前正在使用的套接字。
 *
 * \param ssl: 如果使用 openssl 编译，可选的 ssl 处理程序，用于上述套接字。
 *
 * \param ver: 我们希望在回复中放置的协议版本。
 *
 * \param errcode: 一个整数，告诉对方我们遇到的错误类型。
 *
 * \param error: 一个用户分配的（以 '0' 结尾的）缓冲区，其中包含要传输给对等方的错误描述。
 * 错误消息的长度不能超过 PCAP_ERRBUF_SIZE。
 *
 * \param errbuf: 指向用户分配的缓冲区（大小为 PCAP_ERRBUF_SIZE）的指针，该缓冲区将包含错误消息（如果有的话）。
 * 它可能是网络问题。
 *
 * \return 如果一切正常，则返回 '0'，如果发生错误，则返回 '-1'。错误消息将在 'errbuf' 变量中返回。
 */
int
rpcap_senderror(SOCKET sock, SSL *ssl, uint8 ver, unsigned short errcode, const char *error, char *errbuf)
{
    char sendbuf[RPCAP_NETBUF_SIZE];    /* 临时缓冲区，用于缓冲要发送的数据 */
    int sendbufidx = 0;            /* 索引，用于记录当前缓冲的字节数 */
    uint16 length;

    length = (uint16)strlen(error);

    if (length > PCAP_ERRBUF_SIZE)
        length = PCAP_ERRBUF_SIZE;

    rpcap_createhdr((struct rpcap_header *) sendbuf, ver, RPCAP_MSG_ERROR, errcode, length);
    # 如果将 NULL 作为参数传递给 sock_bufferize 函数，以及指定的结构体大小，进行缓冲区检查
    if (sock_bufferize(NULL, sizeof(struct rpcap_header), NULL, &sendbufidx,
        RPCAP_NETBUF_SIZE, SOCKBUF_CHECKONLY, errbuf, PCAP_ERRBUF_SIZE))
        # 返回 -1，表示出错
        return -1;
    
    # 将错误消息和长度作为参数传递给 sock_bufferize 函数，将数据缓冲到 sendbuf 中
    if (sock_bufferize(error, length, sendbuf, &sendbufidx,
        RPCAP_NETBUF_SIZE, SOCKBUF_BUFFERIZE, errbuf, PCAP_ERRBUF_SIZE))
        # 返回 -1，表示出错
        return -1;
    
    # 将数据发送到套接字或 SSL 连接
    if (sock_send(sock, ssl, sendbuf, sendbufidx, errbuf, PCAP_ERRBUF_SIZE) < 0)
        # 返回 -1，表示出错
        return -1;
    
    # 返回 0，表示成功
    return 0;
/*
 * This function fills in a structure of type rpcap_header.
 *
 * It is provided just because the creation of an rpcap header is a common
 * task. It accepts all the values that appears into an rpcap_header, and
 * it puts them in place using the proper hton() calls.
 *
 * \param header: a pointer to a user-allocated buffer which will contain
 * the serialized header, ready to be sent on the network.
 *
 * \param ver: a value (in the host byte order) which will be placed into the
 * header.ver field and that represents the protocol version number of the
 * current message.
 *
 * \param type: a value (in the host byte order) which will be placed into the
 * header.type field and that represents the type of the current message.
 *
 * \param value: a value (in the host byte order) which will be placed into
 * the header.value field and that has a message-dependent meaning.
 *
 * \param length: a value (in the host by order) which will be placed into
 * the header.length field, representing the payload length of the message.
 *
 * \return Nothing. The serialized header is returned into the 'header'
 * variable.
 */
void
rpcap_createhdr(struct rpcap_header *header, uint8 ver, uint8 type, uint16 value, uint32 length)
{
    // 初始化 rpcap_header 结构体，将其内容全部置为 0
    memset(header, 0, sizeof(struct rpcap_header));

    // 将 ver 参数的值赋给 header 结构体的 ver 字段
    header->ver = ver;
    // 将 type 参数的值赋给 header 结构体的 type 字段
    header->type = type;
    // 将 value 参数的值经过 htons() 转换后赋给 header 结构体的 value 字段
    header->value = htons(value);
    // 将 length 参数的值经过 htonl() 转换后赋给 header 结构体的 plen 字段
    header->plen = htonl(length);
}

/*
 * Convert a message type to a string containing the type name.
 */
static const char *requests[] =
{
    NULL,                /* not a valid message type */
    "RPCAP_MSG_ERROR",
    "RPCAP_MSG_FINDALLIF_REQ",
    "RPCAP_MSG_OPEN_REQ",
    "RPCAP_MSG_STARTCAP_REQ",
    "RPCAP_MSG_UPDATEFILTER_REQ",
    "RPCAP_MSG_CLOSE",
    "RPCAP_MSG_PACKET",
    "RPCAP_MSG_AUTH_REQ",
    "RPCAP_MSG_STATS_REQ",
    "RPCAP_MSG_ENDCAP_REQ",
    "RPCAP_MSG_SETSAMPLING_REQ",
};
#define NUM_REQ_TYPES    (sizeof requests / sizeof requests[0])

static const char *replies[] =
{
    NULL,
    NULL,            /* this would be a reply to RPCAP_MSG_ERROR */
    "RPCAP_MSG_FINDALLIF_REPLY",  /* 回复 RPCAP_MSG_FINDALLIF 消息 */
    "RPCAP_MSG_OPEN_REPLY",  /* 回复 RPCAP_MSG_OPEN 消息 */
    "RPCAP_MSG_STARTCAP_REPLY",  /* 回复 RPCAP_MSG_STARTCAP 消息 */
    "RPCAP_MSG_UPDATEFILTER_REPLY",  /* 回复 RPCAP_MSG_UPDATEFILTER 消息 */
    NULL,            /* this would be a reply to RPCAP_MSG_CLOSE */  /* 这将是对 RPCAP_MSG_CLOSE 的回复 */
    NULL,            /* this would be a reply to RPCAP_MSG_PACKET */  /* 这将是对 RPCAP_MSG_PACKET 的回复 */
    "RPCAP_MSG_AUTH_REPLY",  /* 回复 RPCAP_MSG_AUTH 消息 */
    "RPCAP_MSG_STATS_REPLY",  /* 回复 RPCAP_MSG_STATS 消息 */
    "RPCAP_MSG_ENDCAP_REPLY",  /* 回复 RPCAP_MSG_ENDCAP 消息 */
    "RPCAP_MSG_SETSAMPLING_REPLY",  /* 回复 RPCAP_MSG_SETSAMPLING 消息 */
// 定义回复类型的数量
#define NUM_REPLY_TYPES    (sizeof replies / sizeof replies[0])

// 根据消息类型返回消息类型的字符串表示
const char *
rpcap_msg_type_string(uint8 type)
{
    // 如果消息类型是回复类型
    if (type & RPCAP_MSG_IS_REPLY) {
        // 清除回复类型标志位
        type &= ~RPCAP_MSG_IS_REPLY;
        // 如果消息类型超出回复类型的范围，返回空指针
        if (type >= NUM_REPLY_TYPES)
            return NULL;
        // 返回对应的回复类型字符串
        return replies[type];
    } 
    // 如果消息类型不是回复类型
    else {
        // 如果消息类型超出请求类型的范围，返回空指针
        if (type >= NUM_REQ_TYPES)
            return NULL;
        // 返回对应的请求类型字符串
        return requests[type];
    }
}
```