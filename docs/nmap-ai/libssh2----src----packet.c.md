# `nmap\libssh2\src\packet.c`

```cpp
/*
 * 版权声明，列出了软件的版权归属和使用条件
 * 未经许可，不得用版权所有者或其他贡献者的名字来推广衍生产品
 * 软件提供者不对软件的任何使用提供明示或暗示的担保，包括但不限于适销性和特定用途的适用性
 * 无论在何种情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害承担责任
 * 包含了一些系统头文件，如errno.h、fcntl.h、unistd.h、sys/time.h、inttypes.h
 */
#include "libssh2_priv.h"
#include <errno.h>
#include <fcntl.h>

#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif

#ifdef HAVE_SYS_TIME_H
#include <sys/time.h>
#endif

#ifdef HAVE_INTTYPES_H
#include <inttypes.h>
#endif
/* 在某些平台上需要 struct iovec */
#ifdef HAVE_SYS_UIO_H
#include <sys/uio.h>
#endif

#include <sys/types.h>

#include "transport.h"
#include "channel.h"
#include "packet.h"

/*
 * libssh2_packet_queue_listener
 *
 * 为监听器排队连接请求
 */
static inline int
packet_queue_listener(LIBSSH2_SESSION * session, unsigned char *data,
                      unsigned long datalen,
                      packet_queue_listener_state_t *listen_state)
{
    /*
     * 寻找匹配的监听器
     */
    /* 17 = packet_type(1) + channel(4) + reason(4) + descr(4) + lang(4) */
    unsigned long packet_len = 17 + (sizeof(FwdNotReq) - 1);
    unsigned char *p;
    LIBSSH2_LISTENER *listn = _libssh2_list_first(&session->listeners);
    char failure_code = SSH_OPEN_ADMINISTRATIVELY_PROHIBITED;
    int rc;

    }

    }

    /* 我们不监听你 */
    p = listen_state->packet;
    *(p++) = SSH_MSG_CHANNEL_OPEN_FAILURE;
    _libssh2_store_u32(&p, listen_state->sender_channel);
    _libssh2_store_u32(&p, failure_code);
    _libssh2_store_str(&p, FwdNotReq, sizeof(FwdNotReq) - 1);
    _libssh2_htonu32(p, 0);

    rc = _libssh2_transport_send(session, listen_state->packet,
                                 packet_len, NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc) {
        listen_state->state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc, "Unable to send open failure");

    }
    listen_state->state = libssh2_NB_state_idle;
    return 0;
}

/*
 * packet_x11_open
 *
 * 接受转发的 X11 连接
 */
static inline int
packet_x11_open(LIBSSH2_SESSION * session, unsigned char *data,
                unsigned long datalen,
                packet_x11_open_state_t *x11open_state)
{
    int failure_code = SSH_OPEN_CONNECT_FAILED;
    /* 17 = packet_type(1) + channel(4) + reason(4) + descr(4) + lang(4) */
    unsigned long packet_len = 17 + (sizeof(X11FwdUnAvil) - 1);
    // 声明一个指向无符号字符的指针变量 p
    unsigned char *p;
    // 声明一个指向 LIBSSH2_CHANNEL 结构体的指针变量 channel，并初始化为 x11open_state->channel
    LIBSSH2_CHANNEL *channel = x11open_state->channel;
    // 声明一个整型变量 rc
    int rc;

    // 如果 channel 为空，则设置失败代码为 SSH_OPEN_RESOURCE_SHORTAGE
    if (!channel) {
        failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
    }
    // 继续执行以下代码
    /* fall-trough */
  x11_exit:
    // 将指针 p 指向 x11open_state->packet
    p = x11open_state->packet;
    // 将 SSH_MSG_CHANNEL_OPEN_FAILURE 写入 packet 中
    *(p++) = SSH_MSG_CHANNEL_OPEN_FAILURE;
    // 将 x11open_state->sender_channel 写入 packet 中
    _libssh2_store_u32(&p, x11open_state->sender_channel);
    // 将 failure_code 写入 packet 中
    _libssh2_store_u32(&p, failure_code);
    // 将 X11FwdUnAvil 的内容写入 packet 中
    _libssh2_store_str(&p, X11FwdUnAvil, sizeof(X11FwdUnAvil) - 1);
    // 将 0 写入 packet 中
    _libssh2_htonu32(p, 0);

    // 发送 packet 数据到会话 session，如果返回值为 LIBSSH2_ERROR_EAGAIN，则返回 rc
    rc = _libssh2_transport_send(session, x11open_state->packet, packet_len,
                                 NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    // 如果返回值不为 0，则设置 x11open_state->state 为 libssh2_NB_state_idle，返回发送打开失败的错误信息
    else if(rc) {
        x11open_state->state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc, "Unable to send open failure");
    }
    // 设置 x11open_state->state 为 libssh2_NB_state_idle，返回 0
    x11open_state->state = libssh2_NB_state_idle;
    return 0;
# 定义_libssh2_packet_add函数，用于创建一个新的数据包并将其附加到brigade中。在传输层接收到数据包时调用。
# 输入指针'data'指向分配的数据，该函数在失败或成功时都必须释放它。唯一的例外是返回码为LIBSSH2_ERROR_EAGAIN时。
# 此函数始终以'datalen'大于零的情况调用。
int
_libssh2_packet_add(LIBSSH2_SESSION * session, unsigned char *data,
                    size_t datalen, int macstate)
{
    int rc = 0;  # 初始化变量rc为0
    unsigned char *message = NULL;  # 初始化变量message为空
    unsigned char *language = NULL;  # 初始化变量language为空
    size_t message_len = 0;  # 初始化变量message_len为0
    size_t language_len = 0;  # 初始化变量language_len为0
    LIBSSH2_CHANNEL *channelp = NULL;  # 初始化变量channelp为空
    size_t data_head = 0;  # 初始化变量data_head为0
    unsigned char msg = data[0];  # 获取data的第一个字节，赋值给变量msg

    switch(session->packAdd_state) {  # 根据session的packAdd_state进行判断
    case libssh2_NB_state_idle:  # 如果packAdd_state为libssh2_NB_state_idle
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,  # 调用_libssh2_debug函数，输出调试信息
                       "Packet type %d received, length=%d",
                       (int) msg, (int) datalen);

        if((macstate == LIBSSH2_MAC_INVALID) &&  # 如果macstate为LIBSSH2_MAC_INVALID，并且session的macerror为假或者调用LIBSSH2_MACERROR函数返回真
            (!session->macerror ||
             LIBSSH2_MACERROR(session, (char *) data, datalen))) {
            # 坏的MAC输入，但没有设置回调或回调返回非零
            LIBSSH2_FREE(session, data);  # 释放data的内存
            return _libssh2_error(session, LIBSSH2_ERROR_INVALID_MAC,  # 返回错误码和错误信息
                                  "Invalid MAC received");
        }
        session->packAdd_state = libssh2_NB_state_allocated;  # 将packAdd_state设置为libssh2_NB_state_allocated
        break;
    case libssh2_NB_state_jump1:  # 如果packAdd_state为libssh2_NB_state_jump1
        goto libssh2_packet_add_jump_point1;  # 跳转到libssh2_packet_add_jump_point1标签处
    case libssh2_NB_state_jump2:  # 如果packAdd_state为libssh2_NB_state_jump2
        goto libssh2_packet_add_jump_point2;  # 跳转到libssh2_packet_add_jump_point2标签处
    case libssh2_NB_state_jump3:  # 如果packAdd_state为libssh2_NB_state_jump3
        goto libssh2_packet_add_jump_point3;  # 跳转到libssh2_packet_add_jump_point3标签处
    case libssh2_NB_state_jump4:  # 如果packAdd_state为libssh2_NB_state_jump4
        goto libssh2_packet_add_jump_point4;  # 跳转到libssh2_packet_add_jump_point4标签处
    case libssh2_NB_state_jump5:  # 如果packAdd_state为libssh2_NB_state_jump5
        goto libssh2_packet_add_jump_point5;  # 跳转到libssh2_packet_add_jump_point5标签处
    # 默认情况下什么都不做
    default: /* nothing to do */
        # 跳出当前的循环或者 switch 语句
        break;
    }
#ifdef LIBSSH2DEBUG
            {
                // 如果定义了 LIBSSH2DEBUG 宏，则输出调试信息
                uint32_t stream_id = 0;
                // 如果消息类型为 SSH_MSG_CHANNEL_EXTENDED_DATA，则获取流ID
                if(msg == SSH_MSG_CHANNEL_EXTENDED_DATA)
                    stream_id = _libssh2_ntohu32(data + 5);

                // 输出调试信息，包括数据长度、本地通道ID、远程通道ID和流ID
                _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                               "%d bytes packet_add() for %lu/%lu/%lu",
                               (int) (datalen - data_head),
                               channelp->local.id,
                               channelp->remote.id,
                               stream_id);
            }
    }

    // 如果状态为 libssh2_NB_state_sent
    if(session->packAdd_state == libssh2_NB_state_sent) {
        // 分配内存用于存储数据包
        LIBSSH2_PACKET *packetp =
            LIBSSH2_ALLOC(session, sizeof(LIBSSH2_PACKET));
        // 如果内存分配失败，则输出错误信息，释放数据并返回错误代码
        if(!packetp) {
            _libssh2_debug(session, LIBSSH2_ERROR_ALLOC,
                           "memory for packet");
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return LIBSSH2_ERROR_ALLOC;
        }
        // 将数据和相关信息存储到数据包中
        packetp->data = data;
        packetp->data_len = datalen;
        packetp->data_head = data_head;

        // 将数据包添加到会话的数据包列表中
        _libssh2_list_add(&session->packets, &packetp->node);

        // 更新状态为 libssh2_NB_state_sent1
        session->packAdd_state = libssh2_NB_state_sent1;
    }
    # 如果消息是 SSH_MSG_KEXINIT 并且会话状态不包含正在交换密钥的状态，或者 packAdd_state 等于 libssh2_NB_state_sent2
    if((msg == SSH_MSG_KEXINIT &&
         !(session->state & LIBSSH2_STATE_EXCHANGING_KEYS)) ||
        (session->packAdd_state == libssh2_NB_state_sent2)) {
        # 如果 packAdd_state 等于 libssh2_NB_state_sent1
        if(session->packAdd_state == libssh2_NB_state_sent1) {
            '''
             * 远程端想要新的密钥
             * 好吧，它已经在队列中了，
             * 让我们直接回调到自己
             '''
            # 调试信息：重新协商密钥
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Renegotiating Keys");

            # 设置 packAdd_state 为 libssh2_NB_state_sent2
            session->packAdd_state = libssh2_NB_state_sent2;
        }

        '''
         * KEXINIT 消息已经添加到队列中。packAdd 和 readPack 状态需要被重置，因为 _libssh2_kex_exchange
         * (最终) 调用 _libssh2_transport_read 来读取密钥交换会话的其余部分。
         '''
        # 重置 readPack_state、packet.total_num、packAdd_state 和 fullpacket_state
        session->readPack_state = libssh2_NB_state_idle;
        session->packet.total_num = 0;
        session->packAdd_state = libssh2_NB_state_idle;
        session->fullpacket_state = libssh2_NB_state_idle;

        # 清空 startup_key_state
        memset(&session->startup_key_state, 0, sizeof(key_exchange_state_t));

        '''
         * 如果密钥重新交换失败，让我们希望我们还没有发送 NEWKEYS，否则远程端会像石头一样把我们丢掉
         '''
        # 调用 _libssh2_kex_exchange 进行密钥交换，如果返回值是 LIBSSH2_ERROR_EAGAIN 则直接返回
        rc = _libssh2_kex_exchange(session, 1, &session->startup_key_state);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
    }

    # 设置 packAdd_state 为 libssh2_NB_state_idle
    session->packAdd_state = libssh2_NB_state_idle;
    # 返回 0
    return 0;
/*
 * _libssh2_packet_ask
 *
 * 扫描数据流，查找匹配的数据包类型，可选择先轮询套接字以获取数据包
 */
int
_libssh2_packet_ask(LIBSSH2_SESSION * session, unsigned char packet_type,
                    unsigned char **data, size_t *data_len,
                    int match_ofs, const unsigned char *match_buf,
                    size_t match_len)
{
    LIBSSH2_PACKET *packet = _libssh2_list_first(&session->packets);

    _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                   "Looking for packet of type: %d", (int) packet_type);

    while(packet) {
        if(packet->data[0] == packet_type
            && (packet->data_len >= (match_ofs + match_len))
            && (!match_buf ||
                (memcmp(packet->data + match_ofs, match_buf,
                        match_len) == 0))) {
            *data = packet->data;
            *data_len = packet->data_len;

            /* 从 session->packets 中移除该数据包 */
            _libssh2_list_remove(&packet->node);

            LIBSSH2_FREE(session, packet);

            return 0;
        }
        packet = _libssh2_list_next(&packet->node);
    }
    return -1;
}

/*
 * libssh2_packet_askv
 *
 * 在数据流中扫描一系列数据包类型，可选择先轮询套接字以获取数据包
 */
int
_libssh2_packet_askv(LIBSSH2_SESSION * session,
                     const unsigned char *packet_types,
                     unsigned char **data, size_t *data_len,
                     int match_ofs,
                     const unsigned char *match_buf,
                     size_t match_len)
{
    int i, packet_types_len = strlen((char *) packet_types);

    for(i = 0; i < packet_types_len; i++) {
        if(0 == _libssh2_packet_ask(session, packet_types[i], data,
                                     data_len, match_ofs,
                                     match_buf, match_len)) {
            return 0;
        }
    }

    return -1;
}
*/
/*
 * _libssh2_packet_require
 *
 * 循环调用 _libssh2_transport_read() 直到请求的数据包可用
 * SSH_DISCONNECT 或 SOCKET_DISCONNECTED 会导致中止
 *
 * 在出错时返回负值
 * 在处理请求的数据包时返回 0
 */
int
_libssh2_packet_require(LIBSSH2_SESSION * session, unsigned char packet_type,
                        unsigned char **data, size_t *data_len,
                        int match_ofs,
                        const unsigned char *match_buf,
                        size_t match_len,
                        packet_require_state_t *state)
{
    if(state->start == 0) {
        if(_libssh2_packet_ask(session, packet_type, data, data_len,
                                match_ofs, match_buf,
                                match_len) == 0) {
            /* 数据包在数据包队列中可用 */
            return 0;
        }

        state->start = time(NULL);
    }
}
    # 当会话的套接字状态为已连接时，执行循环
    while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        # 调用_libssh2_transport_read函数读取传输数据
        int ret = _libssh2_transport_read(session);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次尝试读取
        if(ret == LIBSSH2_ERROR_EAGAIN)
            return ret;
        # 如果返回值小于0，表示出现错误
        else if(ret < 0) {
            # 重置起始位置
            state->start = 0;
            # 返回错误代码
            return ret;
        }
        # 如果返回值等于packet_type，表示读取到指定类型的数据包
        else if(ret == packet_type) {
            # 调用_libssh2_packet_ask函数从数据包中提取数据
            ret = _libssh2_packet_ask(session, packet_type, data, data_len,
                                      match_ofs, match_buf, match_len);
            # 重置起始位置
            state->start = 0;
            # 返回提取数据的结果
            return ret;
        }
        # 如果返回值为0，表示没有可用数据，需要等待数据到达或超时
        else if(ret == 0) {
            # 计算剩余时间
            long left = LIBSSH2_READ_TIMEOUT - (long)(time(NULL) -
                                                      state->start);
            # 如果剩余时间小于等于0，表示超时
            if(left <= 0) {
                # 重置起始位置
                state->start = 0;
                # 返回超时错误代码
                return LIBSSH2_ERROR_TIMEOUT;
            }
            # 返回-1，表示暂时没有可用数据包
            return -1; /* no packet available yet */
        }
    }

    # 只有在套接字断开时才会执行到这里
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
/*
 * _libssh2_packet_burn
 *
 * 循环调用 _libssh2_transport_read() 直到有任何数据包可用，并立即丢弃它。
 * 在 KEX 交换期间用于丢弃猜测不准确的 KEX_INIT 数据包
 */
int
_libssh2_packet_burn(LIBSSH2_SESSION * session,
                     libssh2_nonblocking_states * state)
{
    unsigned char *data;
    size_t data_len;
    unsigned char i, all_packets[255];
    int ret;

    if(*state == libssh2_NB_state_idle) {
        for(i = 1; i < 255; i++) {
            all_packets[i - 1] = i;
        }
        all_packets[254] = 0;

        if(_libssh2_packet_askv(session, all_packets, &data, &data_len, 0,
                                 NULL, 0) == 0) {
            i = data[0];
            /* 数据包在数据包队列中可用，丢弃它 */
            LIBSSH2_FREE(session, data);
            return i;
        }

        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Blocking until packet becomes available to burn");
        *state = libssh2_NB_state_created;
    }

    while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        ret = _libssh2_transport_read(session);
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }
        else if(ret < 0) {
            *state = libssh2_NB_state_idle;
            return ret;
        }
        else if(ret == 0) {
            /* FIXME: this might busyloop */
            continue;
        }

        /* 懒惰一点，让 packet_ask 从队列中取出数据包 */
        if(0 ==
            _libssh2_packet_ask(session, (unsigned char)ret,
                                         &data, &data_len, 0, NULL, 0)) {
            /* 如果有数据包，就丢弃它 */
            LIBSSH2_FREE(session, data);
            *state = libssh2_NB_state_idle;
            return ret;
        }
    }

    /* 只有在套接字断开时才会执行到这里 */
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}
/*
 * _libssh2_packet_requirev
 *
 * Loops _libssh2_transport_read() until one of a list of packet types
 * requested is available. SSH_DISCONNECT or a SOCKET_DISCONNECTED will cause
 * a bailout. packet_types is a null terminated list of packet_type numbers
 */

int
_libssh2_packet_requirev(LIBSSH2_SESSION *session,
                         const unsigned char *packet_types,
                         unsigned char **data, size_t *data_len,
                         int match_ofs,
                         const unsigned char *match_buf, size_t match_len,
                         packet_requirev_state_t * state)
{
    // 如果_packet_askv()返回0，表示列表中的一个数据包已经在数据包队列中
    if(_libssh2_packet_askv(session, packet_types, data, data_len, match_ofs,
                             match_buf, match_len) == 0) {
        /* 列表中的一个数据包已经在数据包队列中 */
        state->start = 0;
        return 0;
    }

    // 如果状态的起始时间为0，则设置为当前时间
    if(state->start == 0) {
        state->start = time(NULL);
    }

    // 循环直到会话的套接字状态为LIBSSH2_SOCKET_DISCONNECTED
    while(session->socket_state != LIBSSH2_SOCKET_DISCONNECTED) {
        int ret = _libssh2_transport_read(session);
        // 如果返回值小于0且不是LIBSSH2_ERROR_EAGAIN，则重置状态的起始时间并返回返回值
        if((ret < 0) && (ret != LIBSSH2_ERROR_EAGAIN)) {
            state->start = 0;
            return ret;
        }
        // 如果返回值小于等于0
        if(ret <= 0) {
            long left = LIBSSH2_READ_TIMEOUT -
                (long)(time(NULL) - state->start);

            // 如果剩余时间小于等于0，则重置状态的起始时间并返回LIBSSH2_ERROR_TIMEOUT
            if(left <= 0) {
                state->start = 0;
                return LIBSSH2_ERROR_TIMEOUT;
            }
            // 如果返回值为LIBSSH2_ERROR_EAGAIN，则返回返回值
            else if(ret == LIBSSH2_ERROR_EAGAIN) {
                return ret;
            }
        }

        // 如果返回值在packet_types中
        if(strchr((char *) packet_types, ret)) {
            /* 懒惰一点，让packet_ask从队列中取出数据包 */
            int ret = _libssh2_packet_askv(session, packet_types, data,
                                        data_len, match_ofs, match_buf,
                                        match_len);
            state->start = 0;
            return ret;
        }
    }

    /* 只有在套接字断开连接时才会执行到这里 */
    state->start = 0;
}
    # 返回 LIBSSH2_ERROR_SOCKET_DISCONNECT 错误代码
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
# 代码块结束
```