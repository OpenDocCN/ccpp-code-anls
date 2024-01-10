# `nmap\libssh2\src\channel.c`

```
/*
 * 版权声明，列出了软件的版权归属和使用条件
 * 保留所有权利
 * 允许以源代码或二进制形式重新分发和使用，但需要满足以下条件：
 *   - 源代码的重新分发必须保留上述版权声明、条件列表和下面的免责声明
 *   - 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和下面的免责声明
 *   - 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责
 * 即使已被告知可能发生此类损害，也不得对使用本软件产生的任何责任方式负责
 */

#include "libssh2_priv.h"
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <fcntl.h>
#ifdef HAVE_INTTYPES_H
#include <inttypes.h>
#endif
#include <assert.h>

#include "channel.h"
#include "transport.h"
#include "packet.h"
#include "session.h"
*/
# 获取下一个可用的通道ID
uint32_t
_libssh2_channel_nextid(LIBSSH2_SESSION * session)
{
    # 获取下一个通道ID
    uint32_t id = session->next_channel;
    LIBSSH2_CHANNEL *channel;

    # 获取第一个通道
    channel = _libssh2_list_first(&session->channels);

    # 遍历通道列表，找到最大的通道ID
    while(channel) {
        if(channel->local.id > id) {
            id = channel->local.id;
        }
        channel = _libssh2_list_next(&channel->node);
    }

    # 避免等待关闭已经忘记的通道的数据包
    session->next_channel = id + 1;
    _libssh2_debug(session, LIBSSH2_TRACE_CONN, "Allocated new channel ID#%lu",
                   id);
    return id;
}

# 通过通道ID定位通道指针
LIBSSH2_CHANNEL *
_libssh2_channel_locate(LIBSSH2_SESSION *session, uint32_t channel_id)
{
    LIBSSH2_CHANNEL *channel;
    LIBSSH2_LISTENER *l;

    # 在会话中查找通道
    for(channel = _libssh2_list_first(&session->channels);
        channel;
        channel = _libssh2_list_next(&channel->node)) {
        if(channel->local.id == channel_id)
            return channel;
    }

    # 如果在会话中找不到通道，则检查监听器
    for(l = _libssh2_list_first(&session->listeners); l;
        l = _libssh2_list_next(&l->node)) {
        for(channel = _libssh2_list_first(&l->queue);
            channel;
            channel = _libssh2_list_next(&channel->node)) {
            if(channel->local.id == channel_id)
                return channel;
        }
    }

    return NULL;
}
# _libssh2_channel_open
# 建立一个通用的会话通道
def _libssh2_channel_open(LIBSSH2_SESSION * session, const char *channel_type,
                          uint32_t channel_type_len,
                          uint32_t window_size,
                          uint32_t packet_size,
                          const unsigned char *message,
                          size_t message_len)
{
    # 定义回复代码数组
    static const unsigned char reply_codes[3] = {
        SSH_MSG_CHANNEL_OPEN_CONFIRMATION,
        SSH_MSG_CHANNEL_OPEN_FAILURE,
        0
    };
    # 定义指向无符号字符的指针
    unsigned char *s;
    # 定义整型变量 rc
    int rc;

    # 如果会话的打开状态为 libssh2_NB_state_created
    if(session->open_state == libssh2_NB_state_created) {
        # 发送通道打开请求
        rc = _libssh2_transport_send(session,
                                     session->open_packet,
                                     session->open_packet_len,
                                     message, message_len)
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 报错并返回空值
            _libssh2_error(session, rc,
                           "Would block sending channel-open request");
            return NULL;
        }
        # 如果返回值不为 0
        else if(rc) {
            # 报错并跳转到 channel_error 标签
            _libssh2_error(session, rc,
                           "Unable to send channel-open request");
            goto channel_error;
        }

        # 修改会话的打开状态为 libssh2_NB_state_sent
        session->open_state = libssh2_NB_state_sent;
    }

  channel_error:

    # 如果会话的打开数据存在
    if(session->open_data) {
        # 释放会话的打开数据
        LIBSSH2_FREE(session, session->open_data);
        session->open_data = NULL;
    }
    # 如果会话的打开数据包存在
    if(session->open_packet) {
        # 释放会话的打开数据包
        LIBSSH2_FREE(session, session->open_packet);
        session->open_packet = NULL;
    }
}
    # 如果会话中存在已打开的通道
    if(session->open_channel) {
        # 用于存储通道ID的数组
        unsigned char channel_id[4];
        # 释放已打开通道的通道类型
        LIBSSH2_FREE(session, session->open_channel->channel_type);

        # 从会话的打开通道列表中移除该通道
        _libssh2_list_remove(&session->open_channel->node);

        # 清除针对该通道的数据包
        _libssh2_htonu32(channel_id, session->open_channel->local.id);
        while((_libssh2_packet_ask(session, SSH_MSG_CHANNEL_DATA,
                                    &session->open_data,
                                    &session->open_data_len, 1,
                                    channel_id, 4) >= 0)
               ||
               (_libssh2_packet_ask(session, SSH_MSG_CHANNEL_EXTENDED_DATA,
                                    &session->open_data,
                                    &session->open_data_len, 1,
                                    channel_id, 4) >= 0)) {
            # 释放已打开数据
            LIBSSH2_FREE(session, session->open_data);
            session->open_data = NULL;
        }

        # 释放已打开通道
        LIBSSH2_FREE(session, session->open_channel);
        session->open_channel = NULL;
    }

    # 设置会话的打开状态为空闲状态
    session->open_state = libssh2_NB_state_idle;
    # 返回空值
    return NULL;
# 结束 libssh2_channel_open_ex 函数的定义
}

# 定义 libssh2_channel_open_ex 函数，用于建立一个通用的会话通道
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_open_ex(LIBSSH2_SESSION *session, const char *type,
                        unsigned int type_len,
                        unsigned int window_size, unsigned int packet_size,
                        const char *msg, unsigned int msg_len)
{
    LIBSSH2_CHANNEL *ptr;  # 声明 LIBSSH2_CHANNEL 指针变量 ptr

    if(!session)  # 如果 session 为空，则返回空指针
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, session,  # 调用 BLOCK_ADJUST_ERRNO 宏，调用 _libssh2_channel_open 函数
                       _libssh2_channel_open(session, type, type_len,
                                             window_size, packet_size,
                                             (unsigned char *)msg,
                                             msg_len));
    return ptr;  # 返回指针变量 ptr
}

# 定义 channel_direct_tcpip 函数，用于通过 SSH 会话隧道传输 TCP/IP 连接到直接的主机/端口
static LIBSSH2_CHANNEL *
channel_direct_tcpip(LIBSSH2_SESSION * session, const char *host,
                     int port, const char *shost, int sport)
{
    LIBSSH2_CHANNEL *channel;  # 声明 LIBSSH2_CHANNEL 指针变量 channel
    unsigned char *s;  # 声明无符号字符指针变量 s
    # 如果直接状态为闲置，则执行以下操作
    if(session->direct_state == libssh2_NB_state_idle) {
        # 计算主机名和端口号的长度
        session->direct_host_len = strlen(host);
        session->direct_shost_len = strlen(shost);
        # 计算消息长度
        session->direct_message_len =
            session->direct_host_len + session->direct_shost_len + 16;

        # 打印调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting direct-tcpip session from %s:%d to %s:%d",
                       shost, sport, host, port);

        # 分配内存并将指针指向 direct_message
        s = session->direct_message =
            LIBSSH2_ALLOC(session, session->direct_message_len);
        # 如果分配内存失败，则返回空指针
        if(!session->direct_message) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for "
                           "direct-tcpip connection");
            return NULL;
        }
        # 存储主机名、端口号、目标主机名和目标端口号到 direct_message
        _libssh2_store_str(&s, host, session->direct_host_len);
        _libssh2_store_u32(&s, port);
        _libssh2_store_str(&s, shost, session->direct_shost_len);
        _libssh2_store_u32(&s, sport);
    }

    # 打开一个直接的 TCP/IP 通道
    channel =
        _libssh2_channel_open(session, "direct-tcpip",
                              sizeof("direct-tcpip") - 1,
                              LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                              LIBSSH2_CHANNEL_PACKET_DEFAULT,
                              session->direct_message,
                              session->direct_message_len);

    # 如果通道为空且最后的错误码为 LIBSSH2_ERROR_EAGAIN，则执行以下操作
    if(!channel &&
        libssh2_session_last_errno(session) == LIBSSH2_ERROR_EAGAIN) {
        # 将直接状态设置为已创建，以避免在下一次调用时重新创建包
        session->direct_state = libssh2_NB_state_created;
        return NULL;
    }
    # 默认情况下，将直接状态设置为闲置
    session->direct_state = libssh2_NB_state_idle;

    # 释放 direct_message 的内存并将其指针设置为空
    LIBSSH2_FREE(session, session->direct_message);
    session->direct_message = NULL;

    # 返回通道
    return channel;
# 结束 libssh2_channel_direct_tcpip_ex 函数的定义
}

# 定义 libssh2_channel_direct_tcpip_ex 函数，用于通过 SSH 会话隧道传输 TCP/IP 连接到直接主机/端口
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_direct_tcpip_ex(LIBSSH2_SESSION *session, const char *host,
                                int port, const char *shost, int sport)
{
    LIBSSH2_CHANNEL *ptr;  # 声明 LIBSSH2_CHANNEL 指针变量 ptr

    if(!session)  # 如果会话为空，则返回空指针
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, session,  # 调整错误码，调用 channel_direct_tcpip 函数，将结果赋给 ptr
                       channel_direct_tcpip(session, host, port,
                                            shost, sport));
    return ptr;  # 返回 ptr
}

# 定义 channel_forward_listen 函数，用于在远程主机上绑定端口并监听连接
static LIBSSH2_LISTENER *
channel_forward_listen(LIBSSH2_SESSION * session, const char *host,
                       int port, int *bound_port, int queue_maxsize)
{
    unsigned char *s;  # 声明无符号字符指针变量 s
    static const unsigned char reply_codes[3] =  # 声明并初始化静态无符号字符数组 reply_codes
        { SSH_MSG_REQUEST_SUCCESS, SSH_MSG_REQUEST_FAILURE, 0 };
    int rc;  # 声明整型变量 rc

    if(!host)  # 如果主机为空，则将主机设置为 "0.0.0.0"
        host = "0.0.0.0";
    # 如果转发监听状态为闲置
    if(session->fwdLstn_state == libssh2_NB_state_idle) {
        # 计算转发监听主机名的长度
        session->fwdLstn_host_len = strlen(host);
        # 计算转发监听数据包的长度
        # 14 = packet_type(1) + request_len(4) + want_replay(1) + host_len(4) + port(4)
        session->fwdLstn_packet_len =
            session->fwdLstn_host_len + (sizeof("tcpip-forward") - 1) + 14;

        # 将整个结构清零
        memset(&session->fwdLstn_packet_requirev_state, 0,
               sizeof(session->fwdLstn_packet_requirev_state));

        # 打印调试信息，请求建立 tcpip-forward 会话
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting tcpip-forward session for %s:%d", host,
                       port);

        # 分配转发监听数据包的内存空间
        s = session->fwdLstn_packet =
            LIBSSH2_ALLOC(session, session->fwdLstn_packet_len);
        # 如果内存分配失败
        if(!session->fwdLstn_packet) {
            # 报错，无法为设置环境变量数据包分配内存
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for setenv packet");
            # 返回空指针
            return NULL;
        }

        # 设置数据包的第一个字节为 SSH_MSG_GLOBAL_REQUEST
        *(s++) = SSH_MSG_GLOBAL_REQUEST;
        # 存储字符串 "tcpip-forward" 到数据包中
        _libssh2_store_str(&s, "tcpip-forward", sizeof("tcpip-forward") - 1);
        # 设置 want_reply 为 0x01
        *(s++) = 0x01;          /* want_reply */

        # 存储主机名到数据包中
        _libssh2_store_str(&s, host, session->fwdLstn_host_len);
        # 存储端口号到数据包中
        _libssh2_store_u32(&s, port);

        # 设置转发监听状态为已创建
        session->fwdLstn_state = libssh2_NB_state_created;
    }
    // 如果转发监听状态为已创建
    if(session->fwdLstn_state == libssh2_NB_state_created) {
        // 发送转发监听请求的全局请求数据包
        rc = _libssh2_transport_send(session,
                                     session->fwdLstn_packet,
                                     session->fwdLstn_packet_len,
                                     NULL, 0);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 设置错误信息并返回空指针
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending global-request packet for "
                           "forward listen request");
            return NULL;
        }
        // 如果返回值不为 0，表示发送全局请求数据包失败
        else if(rc) {
            // 设置错误信息并释放资源后返回空指针
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send global-request packet for forward "
                           "listen request");
            LIBSSH2_FREE(session, session->fwdLstn_packet);
            session->fwdLstn_packet = NULL;
            session->fwdLstn_state = libssh2_NB_state_idle;
            return NULL;
        }
        // 释放资源并将转发监听状态设置为已发送
        LIBSSH2_FREE(session, session->fwdLstn_packet);
        session->fwdLstn_packet = NULL;
        session->fwdLstn_state = libssh2_NB_state_sent;
    }

    // 将转发监听状态设置为空闲
    session->fwdLstn_state = libssh2_NB_state_idle;

    // 返回空指针
    return NULL;
# 结束 libssh2_channel_forward_listen_ex 函数的定义
LIBSSH2_API LIBSSH2_LISTENER *
libssh2_channel_forward_listen_ex(LIBSSH2_SESSION *session, const char *host,
                                  int port, int *bound_port, int queue_maxsize)
{
    # 声明指针变量 ptr
    LIBSSH2_LISTENER *ptr;

    # 如果 session 为空，则返回空指针
    if(!session)
        return NULL;

    # 调用 channel_forward_listen 函数，绑定远程主机上的端口并监听连接
    BLOCK_ADJUST_ERRNO(ptr, session,
                       channel_forward_listen(session, host, port, bound_port,
                                              queue_maxsize));
    # 返回指针变量 ptr
    return ptr;
}

# 停止监听远程端口并释放监听器
# 丢弃任何待处理的（未接受的）连接
# 成功返回 0，如果会阻塞则返回 LIBSSH2_ERROR_EAGAIN，出错返回 -1
int _libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener)
{
    # 获取监听器的会话信息
    LIBSSH2_SESSION *session = listener->session;
    # 声明指针变量 queued
    LIBSSH2_CHANNEL *queued;
    # 声明指针变量 packet 和 s
    unsigned char *packet, *s;
    # 计算主机名的长度
    size_t host_len = strlen(listener->host);
    # 计算数据包的长度
    # 14 = packet_type(1) + request_len(4) + want_replay(1) + host_len(4) + port(4)
    size_t packet_len =
        host_len + 14 + sizeof("cancel-tcpip-forward") - 1;
    # 声明变量 rc 和 retcode
    int rc;
    int retcode = 0;
    # 如果监听器的通道取消状态为闲置
    if(listener->chanFwdCncl_state == libssh2_NB_state_idle) {
        # 输出取消 tcpip-forward 会话的调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Cancelling tcpip-forward session for %s:%d",
                       listener->host, listener->port);

        # 分配内存用于存储数据包
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果分配失败，则输出错误信息并返回分配内存错误
        if(!packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for setenv packet");
            return LIBSSH2_ERROR_ALLOC;
        }

        # 设置数据包的第一个字节为 SSH_MSG_GLOBAL_REQUEST
        *(s++) = SSH_MSG_GLOBAL_REQUEST;
        # 存储字符串 "cancel-tcpip-forward" 到数据包中
        _libssh2_store_str(&s, "cancel-tcpip-forward",
                           sizeof("cancel-tcpip-forward") - 1);
        # 设置 want_reply 为 0
        *(s++) = 0x00;          /* want_reply */

        # 存储监听器的主机名和端口号到数据包中
        _libssh2_store_str(&s, listener->host, host_len);
        _libssh2_store_u32(&s, listener->port);

        # 设置监听器的通道取消状态为已创建
        listener->chanFwdCncl_state = libssh2_NB_state_created;
    }
    # 如果监听器的通道取消状态不为闲置
    else {
        # 使用之前创建的数据包
        packet = listener->chanFwdCncl_data;
    }

    # 如果监听器的通道取消状态为已创建
    if(listener->chanFwdCncl_state == libssh2_NB_state_created) {
        # 发送数据包，并获取返回状态
        rc = _libssh2_transport_send(session, packet, packet_len, NULL, 0);
        # 如果返回状态为阻塞，则输出错误信息并保存数据包，然后返回阻塞状态
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending forward request");
            listener->chanFwdCncl_data = packet;
            return rc;
        }
        # 如果返回状态为其他错误，则输出错误信息，并设置状态为已发送
        else if(rc) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send global-request packet for forward "
                           "listen request");
            listener->chanFwdCncl_state = libssh2_NB_state_sent;
            retcode = LIBSSH2_ERROR_SOCKET_SEND;
        }
        # 释放数据包的内存
        LIBSSH2_FREE(session, packet);

        # 设置监听器的通道取消状态为已发送
        listener->chanFwdCncl_state = libssh2_NB_state_sent;
    }
    # 从监听器的队列中获取第一个待处理的通道
    queued = _libssh2_list_first(&listener->queue);
    # 循环处理队列中的每个待处理的通道
    while(queued) {
        # 获取下一个待处理的通道
        LIBSSH2_CHANNEL *next = _libssh2_list_next(&queued->node);
        # 释放当前通道的资源
        rc = _libssh2_channel_free(queued);
        # 如果释放操作返回需要再次尝试的错误码，则立即返回该错误码
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 继续处理下一个待处理的通道
        queued = next;
    }
    # 释放监听器的主机信息内存
    LIBSSH2_FREE(session, listener->host);

    # 从父节点的监听器列表中移除当前监听器
    _libssh2_list_remove(&listener->node);

    # 释放当前监听器的资源
    LIBSSH2_FREE(session, listener);

    # 返回处理结果的错误码
    return retcode;
# 停止监听远程端口并释放监听器，丢弃任何未接受的连接
# 在成功时返回0，在阻塞时返回LIBSSH2_ERROR_EAGAIN，在错误时返回-1
LIBSSH2_API int
libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener)
{
    int rc;

    # 如果监听器为空，则返回错误码LIBSSH2_ERROR_BAD_USE
    if(!listener)
        return LIBSSH2_ERROR_BAD_USE;

    # 调整阻塞状态，调用_libssh2_channel_forward_cancel函数
    BLOCK_ADJUST(rc, listener->session,
                 _libssh2_channel_forward_cancel(listener));
    return rc;
}

# 接受一个连接
static LIBSSH2_CHANNEL *
channel_forward_accept(LIBSSH2_LISTENER *listener)
{
    int rc;

    # 循环读取传输层数据，直到返回值小于等于0
    do {
        rc = _libssh2_transport_read(listener->session);
    } while(rc > 0);

    # 如果监听器的队列中有通道
    if(_libssh2_list_first(&listener->queue)) {
        LIBSSH2_CHANNEL *channel = _libssh2_list_first(&listener->queue);

        # 从监听器的队列中移除通道
        _libssh2_list_remove(&channel->node);

        listener->queue_size--;

        # 将通道添加到会话的通道列表中
        _libssh2_list_add(&channel->session->channels, &channel->node);

        return channel;
    }

    # 如果返回值为LIBSSH2_ERROR_EAGAIN
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(listener->session, LIBSSH2_ERROR_EAGAIN,
                       "Would block waiting for packet");
    }
    else
        _libssh2_error(listener->session, LIBSSH2_ERROR_CHANNEL_UNKNOWN,
                       "Channel not found");
    return NULL;
}

# 接受一个连接
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_forward_accept(LIBSSH2_LISTENER *listener)
{
    LIBSSH2_CHANNEL *ptr;

    # 如果监听器为空，则返回空指针
    if(!listener)
        return NULL;

    # 调整错误码，调用channel_forward_accept函数
    BLOCK_ADJUST_ERRNO(ptr, listener->session,
                       channel_forward_accept(listener));
    return ptr;

}
# 设置通道的环境变量
static int channel_setenv(LIBSSH2_CHANNEL *channel,
                          const char *varname, unsigned int varname_len,
                          const char *value, unsigned int value_len)
{
    # 获取通道所属的会话
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s, *data;
    # 定义回复代码数组
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    size_t data_len;
    int rc;

    if(channel->setenv_state == libssh2_NB_state_idle) {
        # 计算设置环境变量所需的数据包长度
        # 21 = packet_type(1) + channel_id(4) + request_len(4) +
        # request(3)"env" + want_reply(1) + varname_len(4) + value_len(4)
        channel->setenv_packet_len = varname_len + value_len + 21;

        # 将整个数据包清零
        memset(&channel->setenv_packet_requirev_state, 0,
               sizeof(channel->setenv_packet_requirev_state));

        # 调试信息，设置远程环境变量
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Setting remote environment variable: %s=%s on "
                       "channel %lu/%lu",
                       varname, value, channel->local.id, channel->remote.id);

        # 分配内存并创建设置环境变量的数据包
        s = channel->setenv_packet =
            LIBSSH2_ALLOC(session, channel->setenv_packet_len);
        if(!channel->setenv_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory "
                                  "for setenv packet");
        }

        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, "env", sizeof("env") - 1);
        *(s++) = 0x01;
        _libssh2_store_str(&s, varname, varname_len);
        _libssh2_store_str(&s, value, value_len);

        channel->setenv_state = libssh2_NB_state_created;
    }
}
    # 如果通道的环境变量设置状态为已创建
    if(channel->setenv_state == libssh2_NB_state_created) {
        # 发送设置环境变量请求的数据包
        rc = _libssh2_transport_send(session,
                                     channel->setenv_packet,
                                     channel->setenv_packet_len,
                                     NULL, 0);
        # 如果返回值为需要再次调用，则返回错误码并提示“发送设置环境变量请求会阻塞”
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending setenv request");
            return rc;
        }
        # 如果返回值不为0，则释放内存，重置状态，并返回错误码并提示“无法发送通道请求数据包以进行环境变量设置”
        else if(rc) {
            LIBSSH2_FREE(session, channel->setenv_packet);
            channel->setenv_packet = NULL;
            channel->setenv_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send channel-request packet for "
                                  "setenv request");
        }
        # 释放内存，重置状态
        LIBSSH2_FREE(session, channel->setenv_packet);
        channel->setenv_packet = NULL;

        # 将本地通道 ID 转换为网络字节顺序
        _libssh2_htonu32(channel->setenv_local_channel, channel->local.id);

        # 设置环境变量状态为已发送
        channel->setenv_state = libssh2_NB_state_sent;
    }
    # 如果通道的环境变量设置状态为已发送
    if(channel->setenv_state == libssh2_NB_state_sent) {
        # 要求接收指定的回复码，获取数据和数据长度
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->setenv_local_channel, 4,
                                      &channel->
                                      setenv_packet_requirev_state);
        # 如果返回值为需要再次调用才能完成操作
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0
        if(rc) {
            # 将通道的环境变量设置状态设为空闲
            channel->setenv_state = libssh2_NB_state_idle;
            return rc;
        }
        # 如果数据长度小于1
        else if(data_len < 1) {
            # 将通道的环境变量设置状态设为空闲
            channel->setenv_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Unexpected packet size");
        }

        # 如果数据的第一个字节为 SSH_MSG_CHANNEL_SUCCESS
        if(data[0] == SSH_MSG_CHANNEL_SUCCESS) {
            # 释放数据内存
            LIBSSH2_FREE(session, data);
            # 将通道的环境变量设置状态设为空闲
            channel->setenv_state = libssh2_NB_state_idle;
            return 0;
        }

        # 释放数据内存
        LIBSSH2_FREE(session, data);
    }

    # 将通道的环境变量设置状态设为空闲
    channel->setenv_state = libssh2_NB_state_idle;
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for channel-setenv");
# 结束 libssh2_channel_setenv_ex 函数的定义
}

# 设置环境变量，用于在请求 shell/program/subsystem 之前设置环境变量
LIBSSH2_API int
libssh2_channel_setenv_ex(LIBSSH2_CHANNEL *channel,
                          const char *varname, unsigned int varname_len,
                          const char *value, unsigned int value_len)
{
    int rc;

    # 如果 channel 为空，则返回错误码 LIBSSH2_ERROR_BAD_USE
    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    # 调整 rc 的值，调用 channel_setenv 函数设置环境变量
    BLOCK_ADJUST(rc, channel->session,
                 channel_setenv(channel, varname, varname_len,
                                value, value_len));
    # 返回 rc
    return rc;
}

# 请求一个 PTY
static int channel_request_pty(LIBSSH2_CHANNEL *channel,
                               const char *term, unsigned int term_len,
                               const char *modes, unsigned int modes_len,
                               int width, int height,
                               int width_px, int height_px)
{
    # 获取 session 对象
    LIBSSH2_SESSION *session = channel->session;
    # 定义 reply_codes 数组
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    int rc;
    # 如果通道的请求PTY状态为闲置
    if(channel->reqPTY_state == libssh2_NB_state_idle) {
        # 计算请求包的长度，包括各个字段的长度
        # 41 = packet_type(1) + channel(4) + pty_req_len(4) + "pty_req"(7) + want_reply(1) + term_len(4) + width(4) + height(4) + width_px(4) + height_px(4) + modes_len(4)
        if(term_len + modes_len > 256) {
            # 如果 term 和 modes 的长度之和大于 256，则返回错误
            return _libssh2_error(session, LIBSSH2_ERROR_INVAL, "term + mode lengths too large");
        }

        # 计算请求包的总长度
        channel->reqPTY_packet_len = term_len + modes_len + 41;

        # 将请求包的内容清零
        memset(&channel->reqPTY_packet_requirev_state, 0, sizeof(channel->reqPTY_packet_requirev_state));

        # 调试信息，分配终端在通道上
        _libssh2_debug(session, LIBSSH2_TRACE_CONN, "Allocating tty on channel %lu/%lu", channel->local.id, channel->remote.id);

        # 设置请求包的内容
        s = channel->reqPTY_packet;
        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, (char *)"pty-req", sizeof("pty-req") - 1);
        *(s++) = 0x01;
        _libssh2_store_str(&s, term, term_len);
        _libssh2_store_u32(&s, width);
        _libssh2_store_u32(&s, height);
        _libssh2_store_u32(&s, width_px);
        _libssh2_store_u32(&s, height_px);
        _libssh2_store_str(&s, modes, modes_len);

        # 设置请求PTY状态为已创建
        channel->reqPTY_state = libssh2_NB_state_created;
    }
    # 如果通道的 reqPTY_state 状态为 libssh2_NB_state_created
    if(channel->reqPTY_state == libssh2_NB_state_created) {
        # 发送请求终端的数据包
        rc = _libssh2_transport_send(session, channel->reqPTY_packet,
                                     channel->reqPTY_packet_len,
                                     NULL, 0);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回错误码
            _libssh2_error(session, rc,
                           "Would block sending pty request");
            return rc;
        }
        # 如果返回值不为 0，表示发送失败
        else if(rc) {
            # 将 reqPTY_state 状态设置为 libssh2_NB_state_idle
            channel->reqPTY_state = libssh2_NB_state_idle;
            # 返回发送失败的错误信息
            return _libssh2_error(session, rc,
                                  "Unable to send pty-request packet");
        }
        # 将本地通道 ID 转换为网络字节序
        _libssh2_htonu32(channel->reqPTY_local_channel, channel->local.id);

        # 将 reqPTY_state 状态设置为 libssh2_NB_state_sent
        channel->reqPTY_state = libssh2_NB_state_sent;
    }

    # 如果通道的 reqPTY_state 状态为 libssh2_NB_state_sent
    if(channel->reqPTY_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        unsigned char code;
        # 从服务器接收数据包，并验证其合法性
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->reqPTY_local_channel, 4,
                                      &channel->reqPTY_packet_requirev_state);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试接收
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为 0，或者接收到的数据长度小于 1，表示接收失败
        else if(rc || data_len < 1) {
            # 将 reqPTY_state 状态设置为 libssh2_NB_state_idle
            channel->reqPTY_state = libssh2_NB_state_idle;
            # 返回接收失败的错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Failed to require the PTY package");
        }

        # 获取接收到的数据包的第一个字节，即状态码
        code = data[0];

        # 释放接收到的数据包的内存
        LIBSSH2_FREE(session, data);
        # 将 reqPTY_state 状态设置为 libssh2_NB_state_idle
        channel->reqPTY_state = libssh2_NB_state_idle;

        # 如果状态码为 SSH_MSG_CHANNEL_SUCCESS，表示请求成功
        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    # 返回请求失败的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for "
                          "channel request-pty");
    /**
     * channel_request_auth_agent
     * 请求认证代理的实际可重入方法。
     * */
    static int channel_request_auth_agent(LIBSSH2_CHANNEL *channel,
                                          const char *request_str,
                                          int request_str_len)
    {
        LIBSSH2_SESSION *session = channel->session;  // 获取通道的会话
        unsigned char *s;  // 定义一个无符号字符指针
        static const unsigned char reply_codes[3] =  // 定义一个静态的无符号字符数组
            { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
        int rc;  // 定义一个整型变量

        if(channel->req_auth_agent_state == libssh2_NB_state_idle) {  // 如果通道的认证代理状态为闲置
            /* 只有 "auth-agent-req" 和 "auth-agent-req_at_openssh.com" 是有效的选项，因此我们确保它实际上不会比最长可能的还要长。 */
            if(request_str_len > 26) {  // 如果请求字符串长度大于26
                return _libssh2_error(session, LIBSSH2_ERROR_INVAL,  // 返回错误信息
                                      "request_str length too large");
            }

            /*
             *  长度：24 或 36 = packet_type(1) + channel(4) + req_len(4) +
             *    request_str (variable) + want_reply (1) */
            channel->req_auth_agent_packet_len = 10 + request_str_len;  // 计算请求认证代理数据包的长度

            /* 将 requireev 状态清零以重置 */
            memset(&channel->req_auth_agent_requirev_state, 0,  // 将 requireev 状态清零
                   sizeof(channel->req_auth_agent_requirev_state));

            _libssh2_debug(session, LIBSSH2_TRACE_CONN,  // 调试信息
                           "Requesting auth agent on channel %lu/%lu",
                           channel->local.id, channel->remote.id);

            /*
             *  byte      SSH_MSG_CHANNEL_REQUEST
             *  uint32    recipient channel
             *  string    "auth-agent-req"
             *  boolean   want reply
             * */
            s = channel->req_auth_agent_packet;  // 将请求认证代理数据包的地址赋给 s
            *(s++) = SSH_MSG_CHANNEL_REQUEST;  // 将 SSH_MSG_CHANNEL_REQUEST 写入 s
            _libssh2_store_u32(&s, channel->remote.id);  // 将远程通道 ID 写入 s
            _libssh2_store_str(&s, (char *)request_str, request_str_len);  // 将请求字符串写入 s
            *(s++) = 0x01;  // 将 0x01 写入 s

            channel->req_auth_agent_state = libssh2_NB_state_created;  // 设置通道的认证代理状态为已创建
        }
    }
    # 如果通道的认证代理状态为已创建
    if(channel->req_auth_agent_state == libssh2_NB_state_created) {
        /* 发送数据包，可以使用sizeof()获取数据包的大小，因为它始终完全填充；没有可变长度字段。 */
        rc = _libssh2_transport_send(session, channel->req_auth_agent_packet,
                                     channel->req_auth_agent_packet_len,
                                     NULL, 0);

        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示发送认证代理请求会阻塞
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending auth-agent request");
        }
        # 如果返回值不为0
        else if(rc) {
            # 将通道的认证代理状态设置为空闲
            channel->req_auth_agent_state = libssh2_NB_state_idle;
            # 返回错误信息
            return _libssh2_error(session, rc,
                                  "Unable to send auth-agent request");
        }
        # 将通道的本地通道ID转换为网络字节顺序，并将认证代理状态设置为已发送
        _libssh2_htonu32(channel->req_auth_agent_local_channel,
                         channel->local.id);
        channel->req_auth_agent_state = libssh2_NB_state_sent;
    }

    # 如果通道的认证代理状态为已发送
    if(channel->req_auth_agent_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        unsigned char code;

        # 从会话中获取数据包，要求数据包的回复码与本地通道ID匹配
        rc = _libssh2_packet_requirev(
            session, reply_codes, &data, &data_len, 1,
            channel->req_auth_agent_local_channel,
            4, &channel->req_auth_agent_requirev_state);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0
        else if(rc) {
            # 将通道的认证代理状态设置为空闲
            channel->req_auth_agent_state = libssh2_NB_state_idle;
            # 返回错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Failed to request auth-agent");
        }

        # 获取数据包中的第一个字节作为回复码
        code = data[0];

        # 释放数据包内存
        LIBSSH2_FREE(session, data);
        # 将通道的认证代理状态设置为空闲

        channel->req_auth_agent_state = libssh2_NB_state_idle;

        # 如果回复码为SSH_MSG_CHANNEL_SUCCESS，则返回0
        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    # 返回请求认证代理的完成状态
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for auth-agent");
}
/**
 * libssh2_channel_request_auth_agent
 * 请求启用会话的代理转发。请求必须通过特定通道发送，这将在远程端启动代理监听器。一旦通道关闭，代理监听器将继续存在。
 * */
LIBSSH2_API int
libssh2_channel_request_auth_agent(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    /* 当前的代理转发 RFC 草案规定应该发送 "auth-agent-req"，但目前大多数 SSH 服务器实际上希望发送 "auth-agent-req@openssh.com"，因此我们首先尝试这个。 */
    if(channel->req_auth_agent_try_state == libssh2_NB_state_idle) {
        BLOCK_ADJUST(rc, channel->session,
                     channel_request_auth_agent(channel,
                                                "auth-agent-req@openssh.com",
                                                26));

        /* 如果失败（但不是 EAGAIN），则继续尝试下一个请求类型。 */
        if(rc != 0 && rc != LIBSSH2_ERROR_EAGAIN)
            channel->req_auth_agent_try_state = libssh2_NB_state_sent;
    }

    if(channel->req_auth_agent_try_state == libssh2_NB_state_sent) {
        BLOCK_ADJUST(rc, channel->session,
                     channel_request_auth_agent(channel,
                                                "auth-agent-req", 14));

        /* 如果失败且不是 EAGAIN，则继续进行下一个状态机。 */
        if(rc != 0 && rc != LIBSSH2_ERROR_EAGAIN)
            channel->req_auth_agent_try_state = libssh2_NB_state_sent1;
    }

    /* 如果一切顺利，重置尝试状态。 */
    if(rc == 0)
        channel->req_auth_agent_try_state = libssh2_NB_state_idle;

    return rc;
}

/*
 * libssh2_channel_request_pty_ex
 * 呃... 请求一个 PTY
 */
LIBSSH2_API int
# 发送请求以在通道上设置伪终端大小
libssh2_channel_request_pty_ex(LIBSSH2_CHANNEL *channel, const char *term,
                               unsigned int term_len, const char *modes,
                               unsigned int modes_len, int width, int height,
                               int width_px, int height_px)
{
    int rc;  # 声明整型变量 rc

    if(!channel)  # 如果通道为空
        return LIBSSH2_ERROR_BAD_USE;  # 返回错误码 LIBSSH2_ERROR_BAD_USE

    BLOCK_ADJUST(rc, channel->session,  # 调整 rc 的值
                 channel_request_pty(channel, term, term_len, modes,  # 调用 channel_request_pty 函数
                                     modes_len, width, height,
                                     width_px, height_px));
    return rc;  # 返回 rc 的值
}

static int
channel_request_pty_size(LIBSSH2_CHANNEL * channel, int width,  # 设置通道伪终端大小
                         int height, int width_px, int height_px)
{
    LIBSSH2_SESSION *session = channel->session;  # 获取通道的会话
    unsigned char *s;  # 声明无符号字符指针 s
    int rc;  # 声明整型变量 rc
    int retcode = LIBSSH2_ERROR_PROTO;  # 声明整型变量 retcode 并赋值为 LIBSSH2_ERROR_PROTO

    if(channel->reqPTY_state == libssh2_NB_state_idle) {  # 如果通道的 reqPTY_state 状态为 libssh2_NB_state_idle
        channel->reqPTY_packet_len = 39;  # 设置 reqPTY_packet_len 的值为 39

        /* Zero the whole thing out */
        memset(&channel->reqPTY_packet_requirev_state, 0,  # 将 reqPTY_packet_requirev_state 清零
               sizeof(channel->reqPTY_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,  # 调试信息
            "changing tty size on channel %lu/%lu",
            channel->local.id,
            channel->remote.id);

        s = channel->reqPTY_packet;  # 将 reqPTY_packet 赋值给 s

        *(s++) = SSH_MSG_CHANNEL_REQUEST;  # 将 SSH 消息类型写入 s
        _libssh2_store_u32(&s, channel->remote.id);  # 将远程通道 ID 写入 s
        _libssh2_store_str(&s, (char *)"window-change",  # 将字符串 "window-change" 写入 s
                           sizeof("window-change") - 1);
        *(s++) = 0x00;  # 写入字节 0x00
        _libssh2_store_u32(&s, width);  # 将宽度写入 s
        _libssh2_store_u32(&s, height);  # 将高度写入 s
        _libssh2_store_u32(&s, width_px);  # 将宽度像素写入 s
        _libssh2_store_u32(&s, height_px);  # 将高度像素写入 s

        channel->reqPTY_state = libssh2_NB_state_created;  # 设置 reqPTY_state 的状态为 libssh2_NB_state_created
    }
    # 如果通道的请求PTY状态为已创建
    if(channel->reqPTY_state == libssh2_NB_state_created) {
        # 发送请求PTY的数据包
        rc = _libssh2_transport_send(session, channel->reqPTY_packet,
                                     channel->reqPTY_packet_len,
                                     NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 设置错误信息并返回
            _libssh2_error(session, rc,
                           "Would block sending window-change request");
            return rc;
        }
        # 如果返回值不为0，表示发送失败
        else if(rc) {
            # 将请求PTY状态设置为空闲
            channel->reqPTY_state = libssh2_NB_state_idle;
            # 设置错误信息并返回
            return _libssh2_error(session, rc,
                                  "Unable to send window-change packet");
        }
        # 将本地通道ID转换为网络字节顺序
        _libssh2_htonu32(channel->reqPTY_local_channel, channel->local.id);
        # 设置返回码为无错误
        retcode = LIBSSH2_ERROR_NONE;
    }

    # 将请求PTY状态设置为空闲
    channel->reqPTY_state = libssh2_NB_state_idle;
    # 返回返回码
    return retcode;
# 定义一个名为 libssh2_channel_request_pty_size_ex 的函数，用于设置终端大小
LIBSSH2_API int
libssh2_channel_request_pty_size_ex(LIBSSH2_CHANNEL *channel, int width,
                                    int height, int width_px, int height_px)
{
    int rc;  # 声明一个整型变量 rc

    # 如果 channel 为空，则返回错误码 LIBSSH2_ERROR_BAD_USE
    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    # 调用 channel_request_pty_size 函数，设置终端大小，并将返回值赋给 rc
    BLOCK_ADJUST(rc, channel->session,
                 channel_request_pty_size(channel, width, height, width_px,
                                          height_px));
    return rc;  # 返回 rc 的值作为函数结果
}

# 定义一个名为 LIBSSH2_X11_RANDOM_COOKIE_LEN 的宏，表示 X11 随机 cookie 的长度为 32
#define LIBSSH2_X11_RANDOM_COOKIE_LEN       32

# 定义一个名为 channel_x11_req 的静态函数，用于请求 X11 转发
static int
channel_x11_req(LIBSSH2_CHANNEL *channel, int single_connection,
                const char *auth_proto, const char *auth_cookie,
                int screen_number)
{
    LIBSSH2_SESSION *session = channel->session;  # 获取 channel 对应的 session
    unsigned char *s;  # 声明一个指向无符号字符的指针 s
    static const unsigned char reply_codes[3] =  # 声明一个静态的无符号字符数组 reply_codes
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    size_t proto_len =  # 声明一个 size_t 类型的变量 proto_len，表示 auth_proto 的长度
        auth_proto ? strlen(auth_proto) : (sizeof("MIT-MAGIC-COOKIE-1") - 1);
    size_t cookie_len =  # 声明一个 size_t 类型的变量 cookie_len，表示 auth_cookie 的长度
        auth_cookie ? strlen(auth_cookie) : LIBSSH2_X11_RANDOM_COOKIE_LEN;
    int rc;  # 声明一个整型变量 rc
}
    # 如果 X11 请求状态为已创建
    if(channel->reqX11_state == libssh2_NB_state_created) {
        # 发送 X11 请求数据包
        rc = _libssh2_transport_send(session, channel->reqX11_packet,
                                     channel->reqX11_packet_len,
                                     NULL, 0);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回错误码
            _libssh2_error(session, rc,
                           "Would block sending X11-req packet");
            return rc;
        }
        # 如果返回值不为 0，表示发送失败
        if(rc) {
            # 释放内存，重置状态，并返回错误信息
            LIBSSH2_FREE(session, channel->reqX11_packet);
            channel->reqX11_packet = NULL;
            channel->reqX11_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send x11-req packet");
        }
        # 释放内存，重置状态
        LIBSSH2_FREE(session, channel->reqX11_packet);
        channel->reqX11_packet = NULL;
        # 设置状态为已发送
        channel->reqX11_state = libssh2_NB_state_sent;
    }

    # 如果 X11 请求状态为已发送
    if(channel->reqX11_state == libssh2_NB_state_sent) {
        # 定义变量
        size_t data_len;
        unsigned char *data;
        unsigned char code;

        # 接收 X11 请求的响应数据包
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->reqX11_local_channel, 4,
                                      &channel->reqX11_packet_requirev_state);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试接收
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为 0，或者数据长度小于 1，表示接收失败
        else if(rc || data_len < 1) {
            # 重置状态，并返回错误信息
            channel->reqX11_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "waiting for x11-req response packet");
        }

        # 获取响应数据包中的第一个字节作为响应码
        code = data[0];
        # 释放内存，重置状态
        LIBSSH2_FREE(session, data);
        channel->reqX11_state = libssh2_NB_state_idle;

        # 如果响应码为 SSH_MSG_CHANNEL_SUCCESS，表示请求成功
        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    # 返回请求被拒绝的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for channel x11-req");
}

/*
 * libssh2_channel_x11_req_ex
 * 请求 X11 转发
 */
LIBSSH2_API int
libssh2_channel_x11_req_ex(LIBSSH2_CHANNEL *channel, int single_connection,
                           const char *auth_proto, const char *auth_cookie,
                           int screen_number)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 channel_x11_req(channel, single_connection, auth_proto,
                                 auth_cookie, screen_number));
    return rc;
}


/*
 * _libssh2_channel_process_startup
 *
 * libssh2_channel_(shell|exec|subsystem) 的启动原语
 */
int
_libssh2_channel_process_startup(LIBSSH2_CHANNEL *channel,
                                 const char *request, size_t request_len,
                                 const char *message, size_t message_len)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    int rc;

    if(channel->process_state == libssh2_NB_state_end) {
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "Channel can not be reused");
    }
    # 如果通道的处理状态为闲置
    if(channel->process_state == libssh2_NB_state_idle) {
        # 计算处理数据包的长度，包括数据包类型、通道、请求长度和是否需要回复
        channel->process_packet_len = request_len + 10;

        # 将整个结构清零
        memset(&channel->process_packet_requirev_state, 0,
               sizeof(channel->process_packet_requirev_state));

        # 如果有消息，则增加4个字节的长度
        if(message)
            channel->process_packet_len += + 4;

        # 输出调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "starting request(%s) on channel %lu/%lu, message=%s",
                       request, channel->local.id, channel->remote.id,
                       message ? message : "<null>");
        # 分配内存给处理数据包
        s = channel->process_packet =
            LIBSSH2_ALLOC(session, channel->process_packet_len);
        # 如果内存分配失败，则返回错误
        if(!channel->process_packet)
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory "
                                  "for channel-process request");

        # 写入数据包类型
        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        # 写入通道的远程ID
        _libssh2_store_u32(&s, channel->remote.id);
        # 写入请求和请求长度
        _libssh2_store_str(&s, request, request_len);
        # 写入是否需要回复的标志
        *(s++) = 0x01;

        # 如果有消息，则写入消息的长度
        if(message)
            _libssh2_store_u32(&s, message_len);

        # 设置通道的处理状态为已创建
        channel->process_state = libssh2_NB_state_created;
    }
    # 如果通道的处理状态为已创建
    if(channel->process_state == libssh2_NB_state_created) {
        # 发送通道请求数据包
        rc = _libssh2_transport_send(session,
                                     channel->process_packet,
                                     channel->process_packet_len,
                                     (unsigned char *)message, message_len);
        # 如果返回值为需要再次尝试
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 设置错误信息并返回
            _libssh2_error(session, rc,
                           "Would block sending channel request");
            return rc;
        }
        # 如果返回值不为0
        else if(rc) {
            # 释放内存，设置通道处理状态为结束，并返回错误信息
            LIBSSH2_FREE(session, channel->process_packet);
            channel->process_packet = NULL;
            channel->process_state = libssh2_NB_state_end;
            return _libssh2_error(session, rc,
                                  "Unable to send channel request");
        }
        # 释放内存，设置通道处理状态为结束
        LIBSSH2_FREE(session, channel->process_packet);
        channel->process_packet = NULL;

        # 将本地通道 ID 转换为网络字节顺序，设置通道处理状态为已发送
        _libssh2_htonu32(channel->process_local_channel, channel->local.id);
        channel->process_state = libssh2_NB_state_sent;
    }

    # 如果通道的处理状态为已发送
    if(channel->process_state == libssh2_NB_state_sent) {
        # 定义变量
        unsigned char *data;
        size_t data_len;
        unsigned char code;
        # 要求接收数据包，检查返回值
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->process_local_channel, 4,
                                      &channel->process_packet_requirev_state);
        # 如果返回值为需要再次尝试
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0或者数据长度小于1
        else if(rc || data_len < 1) {
            # 设置通道处理状态为结束，并返回错误信息
            channel->process_state = libssh2_NB_state_end;
            return _libssh2_error(session, rc,
                                  "Failed waiting for channel success");
        }

        # 获取数据包中的第一个字节作为 code
        code = data[0];
        # 释放内存，设置通道处理状态为结束
        LIBSSH2_FREE(session, data);
        channel->process_state = libssh2_NB_state_end;

        # 如果 code 为 SSH_MSG_CHANNEL_SUCCESS，则返回0
        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }
    # 返回一个表示 libssh2 错误的值，指示通道请求被拒绝
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for "
                          "channel-process-startup");
}

/*
 * libssh2_channel_process_startup
 *
 * 用于 libssh2_channel_(shell|exec|subsystem) 的基本操作
 */
LIBSSH2_API int
libssh2_channel_process_startup(LIBSSH2_CHANNEL *channel,
                                const char *req, unsigned int req_len,
                                const char *msg, unsigned int msg_len)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_process_startup(channel, req, req_len,
                                                  msg, msg_len));
    return rc;
}


/*
 * libssh2_channel_set_blocking
 *
 * 设置通道的行为为阻塞或非阻塞。套接字仍将保持非阻塞状态。
 */
LIBSSH2_API void
libssh2_channel_set_blocking(LIBSSH2_CHANNEL * channel, int blocking)
{
    if(channel)
        (void) _libssh2_session_set_blocking(channel->session, blocking);
}

/*
 * _libssh2_channel_flush
 *
 * 从一个（或所有）流中刷新数据
 * 返回刷新的字节数，或失败时返回负数
 */
int
_libssh2_channel_flush(LIBSSH2_CHANNEL *channel, int streamid)
{
    }

    channel->read_avail -= channel->flush_flush_bytes;
    channel->remote.window_size -= channel->flush_flush_bytes;

    if(channel->flush_refund_bytes) {
        int rc =
            _libssh2_channel_receive_window_adjust(channel,
                                                   channel->flush_refund_bytes,
                                                   1, NULL);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
    }

    channel->flush_state = libssh2_NB_state_idle;

    return channel->flush_flush_bytes;
}

/*
 * libssh2_channel_flush_ex
 *
 * 从一个（或所有）流中刷新数据
 * 返回刷新的字节数，或失败时返回负数
 */
LIBSSH2_API int
libssh2_channel_flush_ex(LIBSSH2_CHANNEL *channel, int stream)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;
    # 调用_libssh2_channel_flush函数刷新通道的数据流，并将返回值赋给rc
    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_flush(channel, stream));
    # 返回rc的值
    return rc;
# 获取通道的程序退出状态。注意，实际协议提供的是完整的32位，而这个函数返回的是通道的程序退出状态。如果通道为空，则返回0。
LIBSSH2_API int
libssh2_channel_get_exit_status(LIBSSH2_CHANNEL *channel)
{
    # 如果通道为空，则返回0
    if(!channel)
        return 0;

    # 返回通道的退出状态
    return channel->exit_status;
}

# 获取退出信号（不包括前导的"SIG"）、错误消息和语言标签，并将它们放入指定长度的新分配的缓冲区中。调用者可以使用NULL指针来指示不设置该值。即使相应的字符串参数为NULL，*_len变量也会被设置为非NULL。成功时返回LIBSSH2_ERROR_NONE，否则返回API错误代码。
LIBSSH2_API int
libssh2_channel_get_exit_signal(LIBSSH2_CHANNEL *channel,
                                char **exitsignal,
                                size_t *exitsignal_len,
                                char **errmsg,
                                size_t *errmsg_len,
                                char **langtag,
                                size_t *langtag_len)
{
    # 初始化namelen变量
    size_t namelen = 0;
    # 如果通道存在，则获取通道所属会话
    if(channel) {
        LIBSSH2_SESSION *session = channel->session;

        # 如果通道有退出信号
        if(channel->exit_signal) {
            # 获取退出信号的长度
            namelen = strlen(channel->exit_signal);
            # 如果传入了 exitsignal 参数
            if(exitsignal) {
                # 分配内存给 exitsignal
                *exitsignal = LIBSSH2_ALLOC(session, namelen + 1);
                # 如果内存分配失败
                if(!*exitsignal) {
                    # 返回内存分配错误
                    return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                        "Unable to allocate memory for signal name");
                }
                # 将退出信号复制到 exitsignal
                memcpy(*exitsignal, channel->exit_signal, namelen);
                # 添加字符串结束符
                (*exitsignal)[namelen] = '\0';
            }
            # 如果传入了 exitsignal_len 参数
            if(exitsignal_len)
                # 设置退出信号的长度
                *exitsignal_len = namelen;
        }
        # 如果没有退出信号
        else {
            # 如果传入了 exitsignal 参数
            if(exitsignal)
                # 设置 exitsignal 为 NULL
                *exitsignal = NULL;
            # 如果传入了 exitsignal_len 参数
            if(exitsignal_len)
                # 设置 exitsignal_len 为 0
                *exitsignal_len = 0;
        }

        # 设置错误消息和语言标签的 TODO

        # 如果传入了 errmsg 参数
        if(errmsg)
            # 设置 errmsg 为 NULL
            *errmsg = NULL;

        # 如果传入了 errmsg_len 参数
        if(errmsg_len)
            # 设置 errmsg_len 为 0
            *errmsg_len = 0;

        # 如果传入了 langtag 参数
        if(langtag)
            # 设置 langtag 为 NULL
            *langtag = NULL;

        # 如果传入了 langtag_len 参数
        if(langtag_len)
            # 设置 langtag_len 为 0
            *langtag_len = 0;
    }

    # 返回无错误
    return LIBSSH2_ERROR_NONE;
/*
 * _libssh2_channel_receive_window_adjust
 *
 * 调整通道的接收窗口，通过调整字节数。如果要调整的量小于LIBSSH2_CHANNEL_MINADJUST，并且force为0，则调整量将被排队等待后续数据包处理。
 *
 * 调用_libssh2_error()函数！
 */
int
_libssh2_channel_receive_window_adjust(LIBSSH2_CHANNEL * channel,
                                       uint32_t adjustment,
                                       unsigned char force,
                                       unsigned int *store)
{
    int rc;

    // 如果store不为空，则将通道的远程窗口大小存储到store中
    if(store)
        *store = channel->remote.window_size;

    // 如果通道的调整状态为libssh2_NB_state_idle
    if(channel->adjust_state == libssh2_NB_state_idle) {
        // 如果不是强制调整，并且（调整量+调整队列 < LIBSSH2_CHANNEL_MINADJUST）
        if(!force
            && (adjustment + channel->adjust_queue <
                LIBSSH2_CHANNEL_MINADJUST)) {
            _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                           "Queueing %lu bytes for receive window adjustment "
                           "for channel %lu/%lu",
                           adjustment, channel->local.id, channel->remote.id);
            // 将调整量加入调整队列，等待后续数据包处理
            channel->adjust_queue += adjustment;
            return 0;
        }

        // 如果调整量为0，并且调整队列也为0，则直接返回
        if(!adjustment && !channel->adjust_queue) {
            return 0;
        }

        // 将调整量加上调整队列的量
        adjustment += channel->adjust_queue;
        channel->adjust_queue = 0;

        /* 根据刚刚释放的数据块调整窗口 */
        channel->adjust_adjust[0] = SSH_MSG_CHANNEL_WINDOW_ADJUST;
        _libssh2_htonu32(&channel->adjust_adjust[1], channel->remote.id);
        _libssh2_htonu32(&channel->adjust_adjust[5], adjustment);
        _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                       "Adjusting window %lu bytes for data on "
                       "channel %lu/%lu",
                       adjustment, channel->local.id, channel->remote.id);

        // 设置调整状态为libssh2_NB_state_created
        channel->adjust_state = libssh2_NB_state_created;
    }
}
    # 调用_libssh2_transport_send函数发送数据包，将channel->adjust_adjust作为参数传入，长度为9，无特殊标志，长度为0
    rc = _libssh2_transport_send(channel->session, channel->adjust_adjust, 9,
                                 NULL, 0);
    # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示发送操作被阻塞
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        # 在会话中记录错误信息，说明发送窗口调整操作被阻塞
        _libssh2_error(channel->session, rc,
                       "Would block sending window adjust");
        # 返回错误码
        return rc;
    }
    # 如果返回值不为0且不是LIBSSH2_ERROR_EAGAIN，表示发送出现了其他错误
    else if(rc) {
        # 将调整数据保存到channel->adjust_queue中
        channel->adjust_queue = adjustment;
        # 返回发送套接字错误，并记录错误信息
        return _libssh2_error(channel->session, LIBSSH2_ERROR_SOCKET_SEND,
                              "Unable to send transfer-window adjustment "
                              "packet, deferring");
    }
    # 如果发送成功
    else {
        # 更新远程窗口大小
        channel->remote.window_size += adjustment;
    }

    # 将调整状态设置为libssh2_NB_state_idle
    channel->adjust_state = libssh2_NB_state_idle;

    # 返回成功
    return 0;
}

/*
 * libssh2_channel_receive_window_adjust
 *
 * DEPRECATED
 *
 * 调整通道的接收窗口大小，通过调整字节数。如果要调整的量小于LIBSSH2_CHANNEL_MINADJUST，并且force为0，则调整量将被排队等待后续数据包。
 *
 * 返回接收窗口的新大小（远程端理解的大小）。注意它也可能返回EAGAIN，这是非常愚蠢的。
 *
 */
LIBSSH2_API unsigned long
libssh2_channel_receive_window_adjust(LIBSSH2_CHANNEL *channel,
                                      unsigned long adj,
                                      unsigned char force)
{
    unsigned int window;
    int rc;

    if(!channel)
        return (unsigned long)LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_receive_window_adjust(channel, adj,
                                                        force, &window));

    /* 愚蠢 - 但这是以前的工作方式，为了向后兼容性而保留 */
    return rc ? (unsigned long)rc : window;
}

/*
 * libssh2_channel_receive_window_adjust2
 *
 * 调整通道的接收窗口大小，通过调整字节数。如果要调整的量小于LIBSSH2_CHANNEL_MINADJUST，并且force为0，则调整量将被排队等待后续数据包。
 *
 * 将接收窗口的新大小存储在数据'window'指向的位置。
 *
 * 返回“正常”的错误代码：成功为0，失败为负数。
 */
LIBSSH2_API int
libssh2_channel_receive_window_adjust2(LIBSSH2_CHANNEL *channel,
                                       unsigned long adj,
                                       unsigned char force,
                                       unsigned int *window)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;
    # 调整通道接收窗口的大小
    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_receive_window_adjust(channel, adj, force,
                                                        window));
    # 返回调整后的结果
    return rc;
}

int
_libssh2_channel_extended_data(LIBSSH2_CHANNEL *channel, int ignore_mode)
{
    // 如果通道的扩展数据状态为闲置
    if(channel->extData2_state == libssh2_NB_state_idle) {
        // 调试信息：设置通道的扩展数据处理模式
        _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                       "Setting channel %lu/%lu handle_extended_data"
                       " mode to %d",
                       channel->local.id, channel->remote.id, ignore_mode);
        // 设置远程扩展数据的忽略模式
        channel->remote.extended_data_ignore_mode = (char)ignore_mode;

        // 设置扩展数据状态为已创建
        channel->extData2_state = libssh2_NB_state_created;
    }

    // 如果通道的扩展数据状态为闲置
    if(channel->extData2_state == libssh2_NB_state_idle) {
        // 如果忽略模式为忽略扩展数据
        if(ignore_mode == LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE) {
            // 刷新通道，处理扩展数据
            int rc =
                _libssh2_channel_flush(channel,
                                       LIBSSH2_CHANNEL_FLUSH_EXTENDED_DATA);
            // 如果返回值为 EAGAIN，则返回 rc
            if(LIBSSH2_ERROR_EAGAIN == rc)
                return rc;
        }
    }

    // 设置扩展数据状态为闲置
    channel->extData2_state = libssh2_NB_state_idle;
    // 返回 0
    return 0;
}

/*
 * libssh2_channel_handle_extended_data2()
 *
 */
LIBSSH2_API int
libssh2_channel_handle_extended_data2(LIBSSH2_CHANNEL *channel,
                                      int mode)
{
    int rc;

    // 如果通道为空，则返回错误码
    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    // 调整阻塞状态
    BLOCK_ADJUST(rc, channel->session, _libssh2_channel_extended_data(channel,
                                                                      mode));
    // 返回 rc
    return rc;
}

/*
 * libssh2_channel_handle_extended_data
 *
 * DEPRECATED DO NOTE USE!
 *
 * How should extended data look to the calling app?  Keep it in separate
 * channels[_read() _read_stdder()]? (NORMAL) Merge the extended data to the
 * standard data? [everything via _read()]? (MERGE) Ignore it entirely [toss
 * out packets as they come in]? (IGNORE)
 */
LIBSSH2_API void
libssh2_channel_handle_extended_data(LIBSSH2_CHANNEL *channel,
                                     int ignore_mode)
{
    // 调用 libssh2_channel_handle_extended_data2 处理扩展数据
    (void)libssh2_channel_handle_extended_data2(channel, ignore_mode);
}
/*
 * _libssh2_channel_read
 *
 * 从通道中读取数据
 *
 * 重要的是在当前读取的通道完成之前不返回0。如果我们从网络中读取数据，但没有有效数据填充缓冲区，我们必须确保返回LIBSSH2_ERROR_EAGAIN。
 *
 * 接收窗口必须由此函数的用户维护（扩大）。
 */
ssize_t _libssh2_channel_read(LIBSSH2_CHANNEL *channel, int stream_id,
                              char *buf, size_t buflen)
{
    LIBSSH2_SESSION *session = channel->session;  // 获取通道所属的会话
    int rc;  // 定义整型变量 rc
    size_t bytes_read = 0;  // 已读取的字节数
    size_t bytes_want;  // 需要读取的字节数
    int unlink_packet;  // 是否断开数据包
    LIBSSH2_PACKET *read_packet;  // 读取的数据包
    LIBSSH2_PACKET *read_next;  // 下一个要读取的数据包

    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                   "channel_read() wants %d bytes from channel %lu/%lu "
                   "stream #%d",
                   (int) buflen, channel->local.id, channel->remote.id,
                   stream_id);  // 调试信息

    /* 如果接收窗口变得太窄，首先扩展接收窗口 */
    if((channel->read_state == libssh2_NB_state_jump1) ||
       (channel->remote.window_size <
        channel->remote.window_size_initial / 4 * 3 + buflen) ) {

        uint32_t adjustment = channel->remote.window_size_initial + buflen -
            channel->remote.window_size;
        if(adjustment < LIBSSH2_CHANNEL_MINADJUST)
            adjustment = LIBSSH2_CHANNEL_MINADJUST;

        /* 实际的窗口调整可能尚未完成，因此我们需要在这里处理这种特殊状态 */
        channel->read_state = libssh2_NB_state_jump1;
        rc = _libssh2_channel_receive_window_adjust(channel, adjustment,
                                                    0, NULL);  // 调整接收窗口
        if(rc)
            return rc;

        channel->read_state = libssh2_NB_state_idle;
    }

    /* 处理所有待处理的传入数据包。测试证明这种方式可以产生更快的传输。 */
    do {
        rc = _libssh2_transport_read(session);  // 读取传入数据包
    } while(rc > 0);  # 当读取的字节数大于0时，继续循环

    if((rc < 0) && (rc != LIBSSH2_ERROR_EAGAIN))  # 如果读取返回值小于0且不是暂时不可用的错误码
        return _libssh2_error(session, rc, "transport read");  # 返回读取错误信息

    read_packet = _libssh2_list_first(&session->packets);  # 从会话的数据包列表中获取第一个数据包

    }  # 多余的右大括号，可能是代码错误

    if(!bytes_read) {  # 如果没有读取到任何字节
        /* 如果通道已经到达EOF或者已经关闭，我们需要返回相应的信息。
           我们可能在处理传入的传输层数据直到EAGAIN时得到了这些信息，所以我们不能被返回的错误码所迷惑。 */
        if(channel->remote.eof || channel->remote.close)  # 如果通道已经到达EOF或者已经关闭
            return 0;  # 返回0
        else if(rc != LIBSSH2_ERROR_EAGAIN)  # 否则如果返回的错误码不是暂时不可用
            return 0;  # 返回0

        /* 如果传输层返回EAGAIN，那么我们也返回EAGAIN */
        return _libssh2_error(session, rc, "would block");  # 返回暂时不可用的错误信息
    }

    channel->read_avail -= bytes_read;  # 通道可读数据减去已读取的字节数
    channel->remote.window_size -= bytes_read;  # 通道远端窗口大小减去已读取的字节数

    return bytes_read;  # 返回已读取的字节数
}

/*
 * libssh2_channel_read_ex
 *
 * 从通道中读取数据（根据设置的状态是阻塞还是非阻塞）
 *
 * 当以非阻塞方式进行读取时，重要的是在当前读取的通道完成之前不返回 0。如果我们从网络中读取数据但没有有效载荷数据填充缓冲区，我们必须确保返回 LIBSSH2_ERROR_EAGAIN。
 *
 * 此函数首先确保接收窗口足够大，以接收一个完整缓冲区的内容。应用程序可以选择调整接收窗口以增加传输性能。
 */
LIBSSH2_API ssize_t
libssh2_channel_read_ex(LIBSSH2_CHANNEL *channel, int stream_id, char *buf,
                        size_t buflen)
{
    int rc;
    unsigned long recv_window;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    recv_window = libssh2_channel_window_read_ex(channel, NULL, NULL);

    if(buflen > recv_window) {
        BLOCK_ADJUST(rc, channel->session,
                     _libssh2_channel_receive_window_adjust(channel, buflen,
                                                            1, NULL));
    }

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_read(channel, stream_id, buf, buflen));
    return rc;
}

/*
 * _libssh2_channel_packet_data_len
 *
 * 返回当前数据包的数据块大小，如果没有数据包则返回 0。
 */
size_t
_libssh2_channel_packet_data_len(LIBSSH2_CHANNEL * channel, int stream_id)
{
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_PACKET *read_packet;
    LIBSSH2_PACKET *next_packet;
    uint32_t read_local_id;

    read_packet = _libssh2_list_first(&session->packets);
    if(read_packet == NULL)
        return 0;
    # 当存在待读取的数据包时执行循环
    while(read_packet) {

        # 获取下一个待读取的数据包
        next_packet = _libssh2_list_next(&read_packet->node);

        # 如果读取的数据包长度小于5，则跳过当前数据包并记录错误信息，然后继续下一个数据包的处理
        if(read_packet->data_len < 5) {
            read_packet = next_packet;
            _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                           "Unexpected packet length");
            continue;
        }

        # 从数据包中读取本地 ID
        read_local_id = _libssh2_ntohu32(read_packet->data + 1);

        '''
         * 要么我们请求了特定的扩展数据流（并且数据可用），
         * 要么是标准流（并且数据可用），
         * 要么是启用了 extended_data_merge 的标准流并且数据可用
         '''
        # 如果满足以下条件之一，则返回读取的数据包长度减去数据头的长度
        if((stream_id
             && (read_packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA)
             && (channel->local.id == read_local_id)
             && (read_packet->data_len >= 9)
             && (stream_id == (int) _libssh2_ntohu32(read_packet->data + 5)))
            ||
            (!stream_id
             && (read_packet->data[0] == SSH_MSG_CHANNEL_DATA)
             && (channel->local.id == read_local_id))
            ||
            (!stream_id
             && (read_packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA)
             && (channel->local.id == read_local_id)
             && (channel->remote.extended_data_ignore_mode
                 == LIBSSH2_CHANNEL_EXTENDED_DATA_MERGE))) {
            return (read_packet->data_len - read_packet->data_head);
        }

        # 处理下一个数据包
        read_packet = next_packet;
    }

    # 如果没有符合条件的数据包，则返回0
    return 0;
# 结束_libssh2_channel_write函数的定义

/*
 * _libssh2_channel_write
 *
 * 向通道发送数据。注意，如果返回EAGAIN，则调用者必须再次使用相同的输入参数调用此函数。
 *
 * 返回：发送的字节数，如果返回负数，则为错误代码！
 */
ssize_t
_libssh2_channel_write(LIBSSH2_CHANNEL *channel, int stream_id,
                       const unsigned char *buf, size_t buflen)
{
    int rc = 0;
    LIBSSH2_SESSION *session = channel->session;
    ssize_t wrote = 0; /* 用于此特定调用的计数器 */

    /* 理论上，我们可以将较大的缓冲区分成几个较小的数据包，但在提供API/原型的同时进行这样的操作非常困难和令人讨厌。
     *
     * 相反，我们在此调用中只处理前32K的数据，并要求父函数再次调用以处理剩余的数据！32K是基于RFC4253第6.1节中的文本的保守限制。
     */
    if(buflen > 32700)
        buflen = 32700;

    }
    # 如果通道的写状态为已创建
    if(channel->write_state == libssh2_NB_state_created) {
        # 调用_libssh2_transport_send函数发送通道数据
        rc = _libssh2_transport_send(session, channel->write_packet,
                                     channel->write_packet_len,
                                     buf, channel->write_bufwrite);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回发送通道数据失败的错误信息
            return _libssh2_error(session, rc,
                                  "Unable to send channel data");
        }
        # 如果返回值不为0
        else if(rc) {
            # 将通道的写状态设置为空闲
            channel->write_state = libssh2_NB_state_idle;
            # 返回发送通道数据失败的错误信息
            return _libssh2_error(session, rc,
                                  "Unable to send channel data");
        }
        # 缩小本地窗口大小
        channel->local.window_size -= channel->write_bufwrite;

        # 增加已写入的数据量
        wrote += channel->write_bufwrite;

        # 由于_libssh2_transport_write()成功，现在必须返回，以允许调用者提供下一块数据。
        # 我们不能继续发送可能已在同一函数调用中提供的下一块数据，因为我们可能会得到EAGAIN，而我们不能同时返回有关已发送数据和EAGAIN的信息。
        # 因此，通过现在返回较短的数据，调用者将再次调用此函数，并提供新的要发送的数据
        channel->write_state = libssh2_NB_state_idle;

        # 返回已写入的数据量
        return wrote;
    }

    # 返回无效错误
    return LIBSSH2_ERROR_INVAL; # reaching this point is really bad
}

/*
 * libssh2_channel_write_ex
 *
 * 向通道发送数据
 */
LIBSSH2_API ssize_t
libssh2_channel_write_ex(LIBSSH2_CHANNEL *channel, int stream_id,
                         const char *buf, size_t buflen)
{
    ssize_t rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_write(channel, stream_id,
                                        (unsigned char *)buf, buflen));
    return rc;
}

/*
 * channel_send_eof
 *
 * 在通道上发送 EOF
 */
static int channel_send_eof(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char packet[5];    /* packet_type(1) + channelno(4) */
    int rc;

    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                   "Sending EOF on channel %lu/%lu",
                   channel->local.id, channel->remote.id);
    packet[0] = SSH_MSG_CHANNEL_EOF;
    _libssh2_htonu32(packet + 1, channel->remote.id);
    rc = _libssh2_transport_send(session, packet, 5, NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(session, rc,
                       "Would block sending EOF");
        return rc;
    }
    else if(rc) {
        return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                              "Unable to send EOF on channel");
    }
    channel->local.eof = 1;

    return 0;
}

/*
 * libssh2_channel_send_eof
 *
 * 在通道上发送 EOF
 */
LIBSSH2_API int
libssh2_channel_send_eof(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, channel_send_eof(channel));
    return rc;
}

/*
 * libssh2_channel_eof
 *
 * 读取通道的 EOF 状态
 */
LIBSSH2_API int
libssh2_channel_eof(LIBSSH2_CHANNEL * channel)
{
    LIBSSH2_SESSION *session;
    LIBSSH2_PACKET *packet;
    LIBSSH2_PACKET *next_packet;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    session = channel->session;
    # 从会话的数据包列表中获取第一个数据包
    packet = _libssh2_list_first(&session->packets);

    # 遍历数据包列表
    while(packet) {

        # 获取下一个数据包
        next_packet = _libssh2_list_next(&packet->node);

        # 如果数据包长度小于1，则记录错误信息并继续下一个数据包
        if(packet->data_len < 1) {
            packet = next_packet;
            _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                           "Unexpected packet length");
            continue;
        }

        # 如果数据包的第一个字节是 SSH_MSG_CHANNEL_DATA 或 SSH_MSG_CHANNEL_EXTENDED_DATA
        # 并且数据包长度大于等于5，并且本地通道 ID 与数据包中的 ID 相同
        # 则表示有数据等待读取，将 EOF 状态屏蔽并返回0
        if(((packet->data[0] == SSH_MSG_CHANNEL_DATA)
            || (packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA))
            && ((packet->data_len >= 5)
            && (channel->local.id == _libssh2_ntohu32(packet->data + 1)))) {
            /* There's data waiting to be read yet, mask the EOF status */
            return 0;
        }
        # 继续下一个数据包
        packet = next_packet;
    }

    # 返回通道的远程 EOF 状态
    return channel->remote.eof;
# }
# channel_wait_eof
#
# Awaiting channel EOF
#
# 等待通道的 EOF
static int channel_wait_eof(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc;

    if(channel->wait_eof_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Awaiting EOF for channel %lu/%lu", channel->local.id,
                       channel->remote.id);

        channel->wait_eof_state = libssh2_NB_state_created;
    }

    /*
     * While channel is not eof, read more packets from the network.
     * Either the EOF will be set or network timeout will occur.
     */
    do {
        if(channel->remote.eof) {
            break;
        }

        if((channel->remote.window_size == channel->read_avail) &&
            session->api_block_mode)
            return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_WINDOW_FULL,
                                  "Receiving channel window "
                                  "has been exhausted");

        rc = _libssh2_transport_read(session);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc < 0) {
            channel->wait_eof_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "_libssh2_transport_read() bailed out!");
        }
    } while(1);

    channel->wait_eof_state = libssh2_NB_state_idle;

    return 0;
}

/*
 * libssh2_channel_wait_eof
 *
 * Awaiting channel EOF
 */
LIBSSH2_API int
libssh2_channel_wait_eof(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, channel_wait_eof(channel));
    return rc;
}

int _libssh2_channel_close(LIBSSH2_CHANNEL * channel)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc = 0;
    # 如果通道已经关闭，则表现得像我们发送了另一个关闭指令一样，即使我们实际上没有发送...嘘...
    if(channel->local.close) {
        channel->close_state = libssh2_NB_state_idle;  # 设置关闭状态为闲置
        return 0;  # 返回0表示成功
    }

    # 如果通道还未到达文件末尾
    if(!channel->local.eof) {
        rc = channel_send_eof(channel)  # 发送 EOF 到远程主机
        if(rc) {
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                return rc  # 如果返回 EAGAIN 则表示需要再次调用
            }
            _libssh2_error(session, rc, "Unable to send EOF, but closing channel anyway")  # 发送错误信息
        }
    }

    # 忽略是否已经接收到远程主机的 EOF，因为现在已经太晚等待它了。继续关闭！
    
    # 如果关闭状态为闲置
    if(channel->close_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN, "Closing channel %lu/%lu",
                       channel->local.id, channel->remote.id)  # 调试信息，显示正在关闭的通道的本地和远程 ID

        channel->close_packet[0] = SSH_MSG_CHANNEL_CLOSE;  # 设置关闭包的第一个字节为 SSH_MSG_CHANNEL_CLOSE
        _libssh2_htonu32(channel->close_packet + 1, channel->remote.id);  # 设置关闭包的后续字节为远程 ID

        channel->close_state = libssh2_NB_state_created;  # 设置关闭状态为已创建
    }

    # 如果关闭状态为已创建
    if(channel->close_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->close_packet, 5,
                                     NULL, 0);  # 发送关闭包到远程主机
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc, "Would block sending close-channel")  # 发送错误信息
            return rc  # 如果返回 EAGAIN 则表示需要再次调用
        }
        else if(rc) {
            _libssh2_error(session, rc, "Unable to send close-channel request, but closing anyway")  # 发送错误信息
            # 跳过等待响应，继续执行下面的 LIBSSH2_CHANNEL_CLOSE
        }
        else
            channel->close_state = libssh2_NB_state_sent;  # 设置关闭状态为已发送
    }
    # 如果通道的关闭状态为libssh2_NB_state_sent，则需要等待远程SSH_MSG_CHANNEL_CLOSE消息
    if(channel->close_state == libssh2_NB_state_sent) {
        /* We must wait for the remote SSH_MSG_CHANNEL_CLOSE message */

        # 当远程关闭状态为false且rc为0，并且会话的套接字状态不是LIBSSH2_SOCKET_DISCONNECTED时，循环执行_libssh2_transport_read函数
        while(!channel->remote.close && !rc &&
               (session->socket_state != LIBSSH2_SOCKET_DISCONNECTED))
            rc = _libssh2_transport_read(session);
    }

    # 如果rc不等于LIBSSH2_ERROR_EAGAIN，则首先设置本地关闭状态，确保不会再出现EAGAIN
    if(rc != LIBSSH2_ERROR_EAGAIN) {
        /* set the local close state first when we're perfectly confirmed to
           not do any more EAGAINs */
        channel->local.close = 1;

        /* 最后调用回调函数，以确保在返回EAGAIN时保持本地数据 */
        if(channel->close_cb) {
            LIBSSH2_CHANNEL_CLOSE(session, channel);
        }

        # 将关闭状态设置为libssh2_NB_state_idle
        channel->close_state = libssh2_NB_state_idle;
    }

    /* 返回0或错误代码 */
    return rc >= 0 ? 0 : rc;
/*
 * libssh2_channel_close
 *
 * Close a channel
 */
LIBSSH2_API int
libssh2_channel_close(LIBSSH2_CHANNEL *channel)
{
    int rc;

    // 如果通道为空，则返回错误码
    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    // 调整阻塞状态，关闭通道
    BLOCK_ADJUST(rc, channel->session, _libssh2_channel_close(channel) );
    return rc;
}

/*
 * channel_wait_closed
 *
 * Awaiting channel close after EOF
 */
static int channel_wait_closed(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc;

    // 如果远程端未发送 EOF，则返回错误
    if(!channel->remote.eof) {
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "libssh2_channel_wait_closed() invoked when "
                              "channel is not in EOF state");
    }

    // 如果等待关闭状态为闲置，则创建等待关闭状态
    if(channel->wait_closed_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Awaiting close of channel %lu/%lu", channel->local.id,
                       channel->remote.id);

        channel->wait_closed_state = libssh2_NB_state_created;
    }

    /*
     * While channel is not closed, read more packets from the network.
     * Either the channel will be closed or network timeout will occur.
     */
    // 当通道未关闭时，从网络中读取更多数据包
    if(!channel->remote.close) {
        do {
            rc = _libssh2_transport_read(session);
            if(channel->remote.close)
                /* it is now closed, move on! */
                break;
        } while(rc > 0);
        if(rc < 0)
            return rc;
    }

    // 将等待关闭状态设置为闲置
    channel->wait_closed_state = libssh2_NB_state_idle;

    return 0;
}

/*
 * libssh2_channel_wait_closed
 *
 * Awaiting channel close after EOF
 */
LIBSSH2_API int
libssh2_channel_wait_closed(LIBSSH2_CHANNEL *channel)
{
    int rc;

    // 如果通道为空，则返回错误码
    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    // 调整阻塞状态，等待通道关闭
    BLOCK_ADJUST(rc, channel->session, channel_wait_closed(channel));
    return rc;
}
/*
 * _libssh2_channel_free
 *
 * 确保通道已关闭，然后从会话中移除通道并释放其资源
 *
 * 成功返回0，失败返回负值
 */
int _libssh2_channel_free(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;  // 获取通道所属的会话
    unsigned char channel_id[4];  // 通道ID
    unsigned char *data;  // 数据
    size_t data_len;  // 数据长度
    int rc;  // 返回值

    assert(session);  // 断言会话存在

    if(channel->free_state == libssh2_NB_state_idle) {  // 如果通道的释放状态为闲置
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,  // 调试信息
                       "Freeing channel %lu/%lu resources", channel->local.id,  // 释放通道资源
                       channel->remote.id);
        channel->free_state = libssh2_NB_state_created;  // 设置通道的释放状态为已创建
    }

    /* 即使套接字已经失去连接，也允许释放通道 */
    if(!channel->local.close  // 如果本地通道未关闭
        && (session->socket_state == LIBSSH2_SOCKET_CONNECTED)) {  // 并且套接字状态为已连接
        rc = _libssh2_channel_close(channel);  // 关闭通道

        if(rc == LIBSSH2_ERROR_EAGAIN)  // 如果返回值为EAGAIN
            return rc;  // 返回EAGAIN

        /* 忽略所有其他错误，否则可能会阻止通道的释放 */
    }

    channel->free_state = libssh2_NB_state_idle;  // 设置通道的释放状态为闲置

    if(channel->exit_signal) {  // 如果通道有退出信号
        LIBSSH2_FREE(session, channel->exit_signal);  // 释放退出信号
    }

    /*
     * channel->remote.close *might* not be set yet, Well...
     * We've sent the close packet, what more do you want?
     * Just let packet_add ignore it when it finally arrives
     */

    /* 清除针对该通道的数据包 */
    _libssh2_htonu32(channel_id, channel->local.id);  // 将本地通道ID转换为网络字节顺序
    while((_libssh2_packet_ask(session, SSH_MSG_CHANNEL_DATA, &data,  // 当仍有数据包时
                                &data_len, 1, channel_id, 4) >= 0)
           ||
           (_libssh2_packet_ask(session, SSH_MSG_CHANNEL_EXTENDED_DATA, &data,  // 或者有扩展数据包时
                                &data_len, 1, channel_id, 4) >= 0)) {
        LIBSSH2_FREE(session, data);  // 释放数据
    }

    /* 释放"channel_type" */
    if(channel->channel_type) {  // 如果通道类型存在
        LIBSSH2_FREE(session, channel->channel_type);  // 释放通道类型
    // 从通道列表中解除链接
    _libssh2_list_remove(&channel->node);

    /*
     * 确保状态变量中使用的所有内存都被释放
     */
    if(channel->setenv_packet) {
        // 如果存在 setenv_packet，则释放其内存
        LIBSSH2_FREE(session, channel->setenv_packet);
    }
    if(channel->reqX11_packet) {
        // 如果存在 reqX11_packet，则释放其内存
        LIBSSH2_FREE(session, channel->reqX11_packet);
    }
    if(channel->process_packet) {
        // 如果存在 process_packet，则释放其内存
        LIBSSH2_FREE(session, channel->process_packet);
    }

    // 释放通道的内存
    LIBSSH2_FREE(session, channel);

    // 返回 0，表示成功
    return 0;
}
/*
 * libssh2_channel_free
 *
 * 确保通道已关闭，然后从会话中移除通道并释放其资源
 *
 * 成功返回0，失败返回负值
 */
LIBSSH2_API int
libssh2_channel_free(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, _libssh2_channel_free(channel));
    return rc;
}
/*
 * libssh2_channel_window_read_ex
 *
 * 检查读取窗口的状态。返回远程端可以发送的字节数，而不会超出窗口限制。read_avail（如果传递）将被填充为实际可读取的字节数。window_size_initial（如果传递）将被填充为由channel_open请求定义的window_size_initial
 */
LIBSSH2_API unsigned long
libssh2_channel_window_read_ex(LIBSSH2_CHANNEL *channel,
                               unsigned long *read_avail,
                               unsigned long *window_size_initial)
{
    if(!channel)
        return 0; /* 没有通道，就没有窗口！ */

    if(window_size_initial) {
        *window_size_initial = channel->remote.window_size_initial;
    }
    # 如果 read_avail 不为空
    if(read_avail) {
        # 初始化已排队的字节数为 0
        size_t bytes_queued = 0;
        # 定义下一个数据包和当前数据包
        LIBSSH2_PACKET *next_packet;
        LIBSSH2_PACKET *packet =
            _libssh2_list_first(&channel->session->packets);

        # 遍历数据包列表
        while(packet) {
            # 定义数据包类型和下一个数据包
            unsigned char packet_type;
            next_packet = _libssh2_list_next(&packet->node);

            # 如果数据包长度小于 1，记录错误并继续下一个数据包
            if(packet->data_len < 1) {
                packet = next_packet;
                _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                               "Unexpected packet length");
                continue;
            }

            # 获取数据包类型
            packet_type = packet->data[0];

            # 如果数据包类型为 SSH_MSG_CHANNEL_DATA 或 SSH_MSG_CHANNEL_EXTENDED_DATA
            # 并且数据包长度大于等于 5，并且数据包中的通道 ID 与本地通道 ID 相同
            # 则累加已排队的字节数
            if(((packet_type == SSH_MSG_CHANNEL_DATA)
                || (packet_type == SSH_MSG_CHANNEL_EXTENDED_DATA))
               && ((packet->data_len >= 5)
                   && (_libssh2_ntohu32(packet->data + 1) ==
                       channel->local.id))) {
                bytes_queued += packet->data_len - packet->data_head;
            }

            # 处理下一个数据包
            packet = next_packet;
        }

        # 将已排队的字节数赋值给 read_avail
        *read_avail = bytes_queued;
    }

    # 返回远程窗口大小
    return channel->remote.window_size;
/*
 * libssh2_channel_window_write_ex
 *
 * 检查写窗口的状态，返回可以在通道上安全写入的字节数，而不会阻塞
 * window_size_initial（如果传递）将被填充为由channel_open请求定义的初始窗口的大小
 */
LIBSSH2_API unsigned long
libssh2_channel_window_write_ex(LIBSSH2_CHANNEL *channel,
                                unsigned long *window_size_initial)
{
    if(!channel)
        return 0; /* 没有通道，没有窗口！ */

    if(window_size_initial) {
        /* 对于本地启动的通道，这很常见为0，因此作为信息并不是*那么*有用 */
        *window_size_initial = channel->local.window_size_initial;
    }

    return channel->local.window_size;
}
```