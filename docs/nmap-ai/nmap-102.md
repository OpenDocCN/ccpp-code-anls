# Nmap源码解析 102

# `libssh2/src/packet.c`

This is a C program that provides a simple solution to a system of linear equations by using Gaussian elimination to eliminate the highest-degree solutions. The program takes a matrix of linear equations, a vector of upper bounds on the solution variables, and a scalar factor as input, and outputs the vector of unique solutions.

The program starts by sorting the input matrix in decreasing order of the element-wise product of the elements and the variable. This is done to optimize the elimination process by reducing the size of the matrix that needs to be processed.

The program then enters a loop that iterates through the matrix, beginning with the last row and ending with the first row. In each iteration, the program calculates the determinant of the row vector that starts from the variable, and uses the formula for the determinant of a matrix to calculate the determinant of the entire row vector.

If the determinant is zero, the program skips that row and continues to the next row. Otherwise, the program uses the formula for the inverse of a matrix to calculate the inverse of the determinant, and updates the variable with the inverse multiplied by the original variable.

The program also keeps track of the maximum and minimum determinants that have been found so far, and updates them if necessary. This is done to keep track of the best and worst possible solution, and to avoid repeating the same solution multiple times.

The program also includes a check for the user to provide a scalar factor, which is applied to all elements in the solution vector. This allows the user to solve the system of equations with a multiple of the original factor as the scaling factor.

The program ends when the user enters a non-empty value for the scalar factor, or when the matrix has been processed fully.


```cpp
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2005,2006 Mikhail Gusarov
 * Copyright (c) 2009-2014 by Daniel Stenberg
 * Copyright (c) 2010 Simon Josefsson
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

```

这段代码是一个用于在Linux系统上执行SSH命令的标准头文件。它包括了从`libssh2_priv.h`中导入的用于与SSH客户端通信的头文件，从`errno.h`和`fcntl.h`中导入的与错误和文件输入输出相关的头文件，以及从`unistd.h`中导入的用于使用`unistd.h`定义的系统头文件。

具体来说，这段代码会告诉编译器在编译之前包含这些头文件。如果系统支持`unistd.h`，那么它还会包含`<unistd.h>`以便编译器能够识别该文件。

此外，这段代码还包含了`#ifdef`和`#define`预处理指令。这些指令允许开发人员定义自己的头文件和函数，并使用它们来定义编译器的预处理指令。这样可以增加代码的灵活性和可读性。


```cpp
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

```

这段代码定义了一个名为 `libssh2_packet_queue_listener` 的函数，属于 `transport.h` 文件。其主要作用是接收输入为 `transport.h` 结构体，然后根据传递的参数（如 `channel` 和 `packet` 数据结构）对输入数据进行处理。

具体来说，这段代码实现了一个 `SSH2_MSG_REQUEUE` 类型的函数，用于将接收到的连接请求数据加入已连接的请求队列中。函数首先根据 `transport.h` 结构体中的 `conn` 字段确定要接收的连接请求数据类型，然后检查是否支持该类型。如果不支持，函数会输出一个警告信息，然后返回。如果支持，函数会接收输入数据，将其添加到请求队列中，最后返回。


```cpp
/* Needed for struct iovec on some platforms */
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
 * Queue a connection request for a listener
 */
```

This function appears to handle the logic of opening a connection to a remote host and sending a channel opening confirmation。 It does this by first trying to establish a connection using the `libssh2_connect` function, and if that fails, it sends a `频道打开失败` 包给远程主机。

如果成功建立连接，它将进入频道已就绪状态，并向该频道发送一个 `频道开放确认` 包。然后，它将遍历频道队列中的下一个元素，并将新的频道链接到队列的末尾。

如果所有步骤都成功完成，它将返回 0，否则它会返回一个错误代码。如果任何步骤失败，它将返回一个错误代码。


```cpp
static inline int
packet_queue_listener(LIBSSH2_SESSION * session, unsigned char *data,
                      unsigned long datalen,
                      packet_queue_listener_state_t *listen_state)
{
    /*
     * Look for a matching listener
     */
    /* 17 = packet_type(1) + channel(4) + reason(4) + descr(4) + lang(4) */
    unsigned long packet_len = 17 + (sizeof(FwdNotReq) - 1);
    unsigned char *p;
    LIBSSH2_LISTENER *listn = _libssh2_list_first(&session->listeners);
    char failure_code = SSH_OPEN_ADMINISTRATIVELY_PROHIBITED;
    int rc;

    if(listen_state->state == libssh2_NB_state_idle) {
        unsigned long offset = (sizeof("forwarded-tcpip") - 1) + 5;
        size_t temp_len = 0;
        struct string_buf buf;
        buf.data = data;
        buf.dataptr = buf.data;
        buf.len = datalen;

        if(datalen < offset) {
            return _libssh2_error(session, LIBSSH2_ERROR_OUT_OF_BOUNDARY,
                                  "Unexpected packet size");
        }

        buf.dataptr += offset;

        if(_libssh2_get_u32(&buf, &(listen_state->sender_channel))) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting channel");
        }
        if(_libssh2_get_u32(&buf, &(listen_state->initial_window_size))) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting window size");
        }
        if(_libssh2_get_u32(&buf, &(listen_state->packet_size))) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting packet");
        }
        if(_libssh2_get_string(&buf, &(listen_state->host), &temp_len)) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting host");
        }
        listen_state->host_len = (uint32_t)temp_len;

        if(_libssh2_get_u32(&buf, &(listen_state->port))) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting port");
        }
        if(_libssh2_get_string(&buf, &(listen_state->shost), &temp_len)) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting shost");
        }
        listen_state->shost_len = (uint32_t)temp_len;

        if(_libssh2_get_u32(&buf, &(listen_state->sport))) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Data too short extracting sport");
        }

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Remote received connection from %s:%ld to %s:%ld",
                       listen_state->shost, listen_state->sport,
                       listen_state->host, listen_state->port);

        listen_state->state = libssh2_NB_state_allocated;
    }

    if(listen_state->state != libssh2_NB_state_sent) {
        while(listn) {
            if((listn->port == (int) listen_state->port) &&
                (strlen(listn->host) == listen_state->host_len) &&
                (memcmp (listn->host, listen_state->host,
                         listen_state->host_len) == 0)) {
                /* This is our listener */
                LIBSSH2_CHANNEL *channel = NULL;
                listen_state->channel = NULL;

                if(listen_state->state == libssh2_NB_state_allocated) {
                    if(listn->queue_maxsize &&
                        (listn->queue_maxsize <= listn->queue_size)) {
                        /* Queue is full */
                        failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
                        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                       "Listener queue full, ignoring");
                        listen_state->state = libssh2_NB_state_sent;
                        break;
                    }

                    channel = LIBSSH2_CALLOC(session, sizeof(LIBSSH2_CHANNEL));
                    if(!channel) {
                        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                       "Unable to allocate a channel for "
                                       "new connection");
                        failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
                        listen_state->state = libssh2_NB_state_sent;
                        break;
                    }
                    listen_state->channel = channel;

                    channel->session = session;
                    channel->channel_type_len = sizeof("forwarded-tcpip") - 1;
                    channel->channel_type = LIBSSH2_ALLOC(session,
                                                          channel->
                                                          channel_type_len +
                                                          1);
                    if(!channel->channel_type) {
                        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                       "Unable to allocate a channel for new"
                                       " connection");
                        LIBSSH2_FREE(session, channel);
                        failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
                        listen_state->state = libssh2_NB_state_sent;
                        break;
                    }
                    memcpy(channel->channel_type, "forwarded-tcpip",
                           channel->channel_type_len + 1);

                    channel->remote.id = listen_state->sender_channel;
                    channel->remote.window_size_initial =
                        LIBSSH2_CHANNEL_WINDOW_DEFAULT;
                    channel->remote.window_size =
                        LIBSSH2_CHANNEL_WINDOW_DEFAULT;
                    channel->remote.packet_size =
                        LIBSSH2_CHANNEL_PACKET_DEFAULT;

                    channel->local.id = _libssh2_channel_nextid(session);
                    channel->local.window_size_initial =
                        listen_state->initial_window_size;
                    channel->local.window_size =
                        listen_state->initial_window_size;
                    channel->local.packet_size = listen_state->packet_size;

                    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                   "Connection queued: channel %lu/%lu "
                                   "win %lu/%lu packet %lu/%lu",
                                   channel->local.id, channel->remote.id,
                                   channel->local.window_size,
                                   channel->remote.window_size,
                                   channel->local.packet_size,
                                   channel->remote.packet_size);

                    p = listen_state->packet;
                    *(p++) = SSH_MSG_CHANNEL_OPEN_CONFIRMATION;
                    _libssh2_store_u32(&p, channel->remote.id);
                    _libssh2_store_u32(&p, channel->local.id);
                    _libssh2_store_u32(&p,
                                       channel->remote.window_size_initial);
                    _libssh2_store_u32(&p, channel->remote.packet_size);

                    listen_state->state = libssh2_NB_state_created;
                }

                if(listen_state->state == libssh2_NB_state_created) {
                    rc = _libssh2_transport_send(session, listen_state->packet,
                                                 17, NULL, 0);
                    if(rc == LIBSSH2_ERROR_EAGAIN)
                        return rc;
                    else if(rc) {
                        listen_state->state = libssh2_NB_state_idle;
                        return _libssh2_error(session, rc,
                                              "Unable to send channel "
                                              "open confirmation");
                    }

                    /* Link the channel into the end of the queue list */
                    if(listen_state->channel) {
                        _libssh2_list_add(&listn->queue,
                                          &listen_state->channel->node);
                        listn->queue_size++;
                    }

                    listen_state->state = libssh2_NB_state_idle;
                    return 0;
                }
            }

            listn = _libssh2_list_next(&listn->node);
        }

        listen_state->state = libssh2_NB_state_sent;
    }

    /* We're not listening to you */
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

```

This is a function definition for `libssh2_channel_open` function of the SSH2 protocol.
This function handle the channel opening failure and send the failure message to the client.
It takes user provided packet to send to the X11 server, the function will store the failure
It is used to link the channel into the session and pass control to the callback.
It is important to note that this function should be called by thelibssh2\*libssh2\* function to handle the channel opening failure.


```cpp
/*
 * packet_x11_open
 *
 * Accept a forwarded X11 connection
 */
static inline int
packet_x11_open(LIBSSH2_SESSION * session, unsigned char *data,
                unsigned long datalen,
                packet_x11_open_state_t *x11open_state)
{
    int failure_code = SSH_OPEN_CONNECT_FAILED;
    /* 17 = packet_type(1) + channel(4) + reason(4) + descr(4) + lang(4) */
    unsigned long packet_len = 17 + (sizeof(X11FwdUnAvil) - 1);
    unsigned char *p;
    LIBSSH2_CHANNEL *channel = x11open_state->channel;
    int rc;

    if(x11open_state->state == libssh2_NB_state_idle) {

        unsigned long offset = (sizeof("x11") - 1) + 5;
        size_t temp_len = 0;
        struct string_buf buf;
        buf.data = data;
        buf.dataptr = buf.data;
        buf.len = datalen;

        if(datalen < offset) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected data length");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }

        buf.dataptr += offset;

        if(_libssh2_get_u32(&buf, &(x11open_state->sender_channel))) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected sender channel size");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }
        if(_libssh2_get_u32(&buf, &(x11open_state->initial_window_size))) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected window size");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }
        if(_libssh2_get_u32(&buf, &(x11open_state->packet_size))) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected window size");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }
        if(_libssh2_get_string(&buf, &(x11open_state->shost), &temp_len)) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected host size");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }
        x11open_state->shost_len = (uint32_t)temp_len;

        if(_libssh2_get_u32(&buf, &(x11open_state->sport))) {
            _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                           "unexpected port size");
            failure_code = SSH_OPEN_CONNECT_FAILED;
            goto x11_exit;
        }

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "X11 Connection Received from %s:%ld on channel %lu",
                       x11open_state->shost, x11open_state->sport,
                       x11open_state->sender_channel);

        x11open_state->state = libssh2_NB_state_allocated;
    }

    if(session->x11) {
        if(x11open_state->state == libssh2_NB_state_allocated) {
            channel = LIBSSH2_CALLOC(session, sizeof(LIBSSH2_CHANNEL));
            if(!channel) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "allocate a channel for new connection");
                failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
                goto x11_exit;
            }

            channel->session = session;
            channel->channel_type_len = sizeof("x11") - 1;
            channel->channel_type = LIBSSH2_ALLOC(session,
                                                  channel->channel_type_len +
                                                  1);
            if(!channel->channel_type) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "allocate a channel for new connection");
                LIBSSH2_FREE(session, channel);
                failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
                goto x11_exit;
            }
            memcpy(channel->channel_type, "x11",
                   channel->channel_type_len + 1);

            channel->remote.id = x11open_state->sender_channel;
            channel->remote.window_size_initial =
                LIBSSH2_CHANNEL_WINDOW_DEFAULT;
            channel->remote.window_size = LIBSSH2_CHANNEL_WINDOW_DEFAULT;
            channel->remote.packet_size = LIBSSH2_CHANNEL_PACKET_DEFAULT;

            channel->local.id = _libssh2_channel_nextid(session);
            channel->local.window_size_initial =
                x11open_state->initial_window_size;
            channel->local.window_size = x11open_state->initial_window_size;
            channel->local.packet_size = x11open_state->packet_size;

            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "X11 Connection established: channel %lu/%lu "
                           "win %lu/%lu packet %lu/%lu",
                           channel->local.id, channel->remote.id,
                           channel->local.window_size,
                           channel->remote.window_size,
                           channel->local.packet_size,
                           channel->remote.packet_size);
            p = x11open_state->packet;
            *(p++) = SSH_MSG_CHANNEL_OPEN_CONFIRMATION;
            _libssh2_store_u32(&p, channel->remote.id);
            _libssh2_store_u32(&p, channel->local.id);
            _libssh2_store_u32(&p, channel->remote.window_size_initial);
            _libssh2_store_u32(&p, channel->remote.packet_size);

            x11open_state->state = libssh2_NB_state_created;
        }

        if(x11open_state->state == libssh2_NB_state_created) {
            rc = _libssh2_transport_send(session, x11open_state->packet, 17,
                                         NULL, 0);
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                return rc;
            }
            else if(rc) {
                x11open_state->state = libssh2_NB_state_idle;
                return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                      "Unable to send channel open "
                                      "confirmation");
            }

            /* Link the channel into the session */
            _libssh2_list_add(&session->channels, &channel->node);

            /*
             * Pass control to the callback, they may turn right around and
             * free the channel, or actually use it
             */
            LIBSSH2_X11_OPEN(channel, (char *)x11open_state->shost,
                             x11open_state->sport);

            x11open_state->state = libssh2_NB_state_idle;
            return 0;
        }
    }
    else
        failure_code = SSH_OPEN_RESOURCE_SHORTAGE;
    /* fall-trough */
  x11_exit:
    p = x11open_state->packet;
    *(p++) = SSH_MSG_CHANNEL_OPEN_FAILURE;
    _libssh2_store_u32(&p, x11open_state->sender_channel);
    _libssh2_store_u32(&p, failure_code);
    _libssh2_store_str(&p, X11FwdUnAvil, sizeof(X11FwdUnAvil) - 1);
    _libssh2_htonu32(p, 0);

    rc = _libssh2_transport_send(session, x11open_state->packet, packet_len,
                                 NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc) {
        x11open_state->state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc, "Unable to send open failure");
    }
    x11open_state->state = libssh2_NB_state_idle;
    return 0;
}

```

这段代码的作用是处理 SSH 协议中的 CHANNEL_DATA 消息。当接收到这个消息时，会根据消息中携带的 channelno 和 data_type_code 参数，查找对应的频道，并将数据取出。

具体来说，代码中首先定义了一个常量 packet，表示要发送的请求失败的数据。然后，在接收到 CHANNEL_DATA 消息时，会执行下面的操作：

1. 将 packet 添加到当前会话的 packets 数组中。
2. 根据 channelno 参数查找对应的频道。
3. 如果找到了频道，取出数据。
4. 如果找不到对应的频道，或者取出的数据长度小于数据头，会将 channelno 置为 LIBSSH2_ERROR_CHANNEL_UNKNOWN，并向客户端发送一个错误消息，然后关闭频道。
5. 将 data 字段清零，表示已经处理完了这个数据包。
6. 将当前状态设置为 LIBSSH2_NB_state_idle，然后返回 0。


```cpp
/*
 * _libssh2_packet_add
 *
 * Create a new packet and attach it to the brigade. Called from the transport
 * layer when it has received a packet.
 *
 * The input pointer 'data' is pointing to allocated data that this function
 * is asked to deal with so on failure OR success, it must be freed fine.
 * The only exception is when the return code is LIBSSH2_ERROR_EAGAIN.
 *
 * This function will always be called with 'datalen' greater than zero.
 */
int
_libssh2_packet_add(LIBSSH2_SESSION * session, unsigned char *data,
                    size_t datalen, int macstate)
{
    int rc = 0;
    unsigned char *message = NULL;
    unsigned char *language = NULL;
    size_t message_len = 0;
    size_t language_len = 0;
    LIBSSH2_CHANNEL *channelp = NULL;
    size_t data_head = 0;
    unsigned char msg = data[0];

    switch(session->packAdd_state) {
    case libssh2_NB_state_idle:
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Packet type %d received, length=%d",
                       (int) msg, (int) datalen);

        if((macstate == LIBSSH2_MAC_INVALID) &&
            (!session->macerror ||
             LIBSSH2_MACERROR(session, (char *) data, datalen))) {
            /* Bad MAC input, but no callback set or non-zero return from the
               callback */

            LIBSSH2_FREE(session, data);
            return _libssh2_error(session, LIBSSH2_ERROR_INVALID_MAC,
                                  "Invalid MAC received");
        }
        session->packAdd_state = libssh2_NB_state_allocated;
        break;
    case libssh2_NB_state_jump1:
        goto libssh2_packet_add_jump_point1;
    case libssh2_NB_state_jump2:
        goto libssh2_packet_add_jump_point2;
    case libssh2_NB_state_jump3:
        goto libssh2_packet_add_jump_point3;
    case libssh2_NB_state_jump4:
        goto libssh2_packet_add_jump_point4;
    case libssh2_NB_state_jump5:
        goto libssh2_packet_add_jump_point5;
    default: /* nothing to do */
        break;
    }

    if(session->packAdd_state == libssh2_NB_state_allocated) {
        /* A couple exceptions to the packet adding rule: */
        switch(msg) {

            /*
              byte      SSH_MSG_DISCONNECT
              uint32    reason code
              string    description in ISO-10646 UTF-8 encoding [RFC3629]
              string    language tag [RFC3066]
            */

        case SSH_MSG_DISCONNECT:
            if(datalen >= 5) {
                uint32_t reason = 0;
                struct string_buf buf;
                buf.data = (unsigned char *)data;
                buf.dataptr = buf.data;
                buf.len = datalen;
                buf.dataptr++; /* advance past type */

                _libssh2_get_u32(&buf, &reason);
                _libssh2_get_string(&buf, &message, &message_len);
                _libssh2_get_string(&buf, &language, &language_len);

                if(session->ssh_msg_disconnect) {
                    LIBSSH2_DISCONNECT(session, reason, (const char *)message,
                                       message_len, (const char *)language,
                                       language_len);
                }

                _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                               "Disconnect(%d): %s(%s)", reason,
                               message, language);
            }

            LIBSSH2_FREE(session, data);
            session->socket_state = LIBSSH2_SOCKET_DISCONNECTED;
            session->packAdd_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_DISCONNECT,
                                  "socket disconnect");
            /*
              byte      SSH_MSG_IGNORE
              string    data
            */

        case SSH_MSG_IGNORE:
            if(datalen >= 2) {
                if(session->ssh_msg_ignore) {
                    LIBSSH2_IGNORE(session, (char *) data + 1, datalen - 1);
                }
            }
            else if(session->ssh_msg_ignore) {
                LIBSSH2_IGNORE(session, "", 0);
            }
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;

            /*
              byte      SSH_MSG_DEBUG
              boolean   always_display
              string    message in ISO-10646 UTF-8 encoding [RFC3629]
              string    language tag [RFC3066]
            */

        case SSH_MSG_DEBUG:
            if(datalen >= 2) {
                int always_display = data[1];

                if(datalen >= 6) {
                    struct string_buf buf;
                    buf.data = (unsigned char *)data;
                    buf.dataptr = buf.data;
                    buf.len = datalen;
                    buf.dataptr += 2; /* advance past type & always display */

                    _libssh2_get_string(&buf, &message, &message_len);
                    _libssh2_get_string(&buf, &language, &language_len);
                }

                if(session->ssh_msg_debug) {
                    LIBSSH2_DEBUG(session, always_display,
                                  (const char *)message,
                                  message_len, (const char *)language,
                                  language_len);
                }
            }

            /*
             * _libssh2_debug will actually truncate this for us so
             * that it's not an inordinate about of data
             */
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                           "Debug Packet: %s", message);
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;

            /*
              byte      SSH_MSG_GLOBAL_REQUEST
              string    request name in US-ASCII only
              boolean   want reply
              ....      request-specific data follows
            */

        case SSH_MSG_GLOBAL_REQUEST:
            if(datalen >= 5) {
                uint32_t len = 0;
                unsigned char want_reply = 0;
                len = _libssh2_ntohu32(data + 1);
                if((len <= (UINT_MAX - 6)) && (datalen >= (6 + len))) {
                    want_reply = data[5 + len];
                    _libssh2_debug(session,
                                   LIBSSH2_TRACE_CONN,
                                   "Received global request type %.*s (wr %X)",
                                   len, data + 5, want_reply);
                }


                if(want_reply) {
                    static const unsigned char packet =
                        SSH_MSG_REQUEST_FAILURE;
                  libssh2_packet_add_jump_point5:
                    session->packAdd_state = libssh2_NB_state_jump5;
                    rc = _libssh2_transport_send(session, &packet, 1, NULL, 0);
                    if(rc == LIBSSH2_ERROR_EAGAIN)
                        return rc;
                }
            }
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;

            /*
              byte      SSH_MSG_CHANNEL_EXTENDED_DATA
              uint32    recipient channel
              uint32    data_type_code
              string    data
            */

        case SSH_MSG_CHANNEL_EXTENDED_DATA:
            /* streamid(4) */
            data_head += 4;

            /* fall-through */

            /*
              byte      SSH_MSG_CHANNEL_DATA
              uint32    recipient channel
              string    data
            */

        case SSH_MSG_CHANNEL_DATA:
            /* packet_type(1) + channelno(4) + datalen(4) */
            data_head += 9;

            if(datalen >= data_head)
                channelp =
                    _libssh2_channel_locate(session,
                                            _libssh2_ntohu32(data + 1));

            if(!channelp) {
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_UNKNOWN,
                               "Packet received for unknown channel");
                LIBSSH2_FREE(session, data);
                session->packAdd_state = libssh2_NB_state_idle;
                return 0;
            }
```

This function is part of the `libssh2_session` class in the `libssh2` library, and it handles the sending and receiving of key exchange messages between the client and server.

When a `KEYEXCHANGE_MSG_KEYINIT` message is received from the server, the function first checks if the `session->state` bitmask contains any conditions that prevent sending `KEYEXCHANGE_MSG_KEYINIT` messages. If not, the function sends the message.

If the function has sent the message and the server has not responded with an error, it then resets the `session->packAdd_state` bitmask to `libssh2_NB_state_idle` and returns 0.

If the function has sent the message and the server has responded with an error, it then resets the `session->fullpacket_state` bitmask to `libssh2_NB_state_idle`, sets the `session->packAdd_state` bitmask to `libssh2_NB_state_idle`, and sets the `session->startup_key_state` bitmask to the key exchange state.

If the key exchange initialization failed, the function then returns the error code.


```cpp
#ifdef LIBSSH2DEBUG
            {
                uint32_t stream_id = 0;
                if(msg == SSH_MSG_CHANNEL_EXTENDED_DATA)
                    stream_id = _libssh2_ntohu32(data + 5);

                _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                               "%d bytes packet_add() for %lu/%lu/%lu",
                               (int) (datalen - data_head),
                               channelp->local.id,
                               channelp->remote.id,
                               stream_id);
            }
#endif
            if((channelp->remote.extended_data_ignore_mode ==
                 LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE) &&
                (msg == SSH_MSG_CHANNEL_EXTENDED_DATA)) {
                /* Pretend we didn't receive this */
                LIBSSH2_FREE(session, data);

                _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                               "Ignoring extended data and refunding %d bytes",
                               (int) (datalen - 13));
                if(channelp->read_avail + datalen - data_head >=
                    channelp->remote.window_size)
                    datalen = channelp->remote.window_size -
                        channelp->read_avail + data_head;

                channelp->remote.window_size -= datalen - data_head;
                _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                               "shrinking window size by %lu bytes to %lu, "
                               "read_avail %lu",
                               datalen - data_head,
                               channelp->remote.window_size,
                               channelp->read_avail);

                session->packAdd_channelp = channelp;

                /* Adjust the window based on the block we just freed */
              libssh2_packet_add_jump_point1:
                session->packAdd_state = libssh2_NB_state_jump1;
                rc = _libssh2_channel_receive_window_adjust(session->
                                                            packAdd_channelp,
                                                            datalen - 13,
                                                            1, NULL);
                if(rc == LIBSSH2_ERROR_EAGAIN)
                    return rc;

                session->packAdd_state = libssh2_NB_state_idle;
                return 0;
            }

            /*
             * REMEMBER! remote means remote as source of data,
             * NOT remote window!
             */
            if(channelp->remote.packet_size < (datalen - data_head)) {
                /*
                 * Spec says we MAY ignore bytes sent beyond
                 * packet_size
                 */
                _libssh2_error(session,
                               LIBSSH2_ERROR_CHANNEL_PACKET_EXCEEDED,
                               "Packet contains more data than we offered"
                               " to receive, truncating");
                datalen = channelp->remote.packet_size + data_head;
            }
            if(channelp->remote.window_size <= channelp->read_avail) {
                /*
                 * Spec says we MAY ignore bytes sent beyond
                 * window_size
                 */
                _libssh2_error(session,
                               LIBSSH2_ERROR_CHANNEL_WINDOW_EXCEEDED,
                               "The current receive window is full,"
                               " data ignored");
                LIBSSH2_FREE(session, data);
                session->packAdd_state = libssh2_NB_state_idle;
                return 0;
            }
            /* Reset EOF status */
            channelp->remote.eof = 0;

            if(channelp->read_avail + datalen - data_head >
                channelp->remote.window_size) {
                _libssh2_error(session,
                               LIBSSH2_ERROR_CHANNEL_WINDOW_EXCEEDED,
                               "Remote sent more data than current "
                               "window allows, truncating");
                datalen = channelp->remote.window_size -
                    channelp->read_avail + data_head;
            }

            /* Update the read_avail counter. The window size will be
             * updated once the data is actually read from the queue
             * from an upper layer */
            channelp->read_avail += datalen - data_head;

            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "increasing read_avail by %lu bytes to %lu/%lu",
                           (long)(datalen - data_head),
                           (long)channelp->read_avail,
                           (long)channelp->remote.window_size);

            break;

            /*
              byte      SSH_MSG_CHANNEL_EOF
              uint32    recipient channel
            */

        case SSH_MSG_CHANNEL_EOF:
            if(datalen >= 5)
                channelp =
                    _libssh2_channel_locate(session,
                                            _libssh2_ntohu32(data + 1));
            if(!channelp)
                /* We may have freed already, just quietly ignore this... */
                ;
            else {
                _libssh2_debug(session,
                               LIBSSH2_TRACE_CONN,
                               "EOF received for channel %lu/%lu",
                               channelp->local.id,
                               channelp->remote.id);
                channelp->remote.eof = 1;
            }
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;

            /*
              byte      SSH_MSG_CHANNEL_REQUEST
              uint32    recipient channel
              string    request type in US-ASCII characters only
              boolean   want reply
              ....      type-specific data follows
            */

        case SSH_MSG_CHANNEL_REQUEST:
            if(datalen >= 9) {
                uint32_t channel = _libssh2_ntohu32(data + 1);
                uint32_t len = _libssh2_ntohu32(data + 5);
                unsigned char want_reply = 1;

                if((len + 9) < datalen)
                    want_reply = data[len + 9];

                _libssh2_debug(session,
                               LIBSSH2_TRACE_CONN,
                               "Channel %d received request type %.*s (wr %X)",
                               channel, len, data + 9, want_reply);

                if(len == sizeof("exit-status") - 1
                    && (sizeof("exit-status") - 1 + 9) <= datalen
                    && !memcmp("exit-status", data + 9,
                               sizeof("exit-status") - 1)) {

                    /* we've got "exit-status" packet. Set the session value */
                    if(datalen >= 20)
                        channelp =
                            _libssh2_channel_locate(session, channel);

                    if(channelp && (sizeof("exit-status") + 13) <= datalen) {
                        channelp->exit_status =
                            _libssh2_ntohu32(data + 9 + sizeof("exit-status"));
                        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                       "Exit status %lu received for "
                                       "channel %lu/%lu",
                                       channelp->exit_status,
                                       channelp->local.id,
                                       channelp->remote.id);
                    }

                }
                else if(len == sizeof("exit-signal") - 1
                         && (sizeof("exit-signal") - 1 + 9) <= datalen
                         && !memcmp("exit-signal", data + 9,
                                    sizeof("exit-signal") - 1)) {
                    /* command terminated due to signal */
                    if(datalen >= 20)
                        channelp = _libssh2_channel_locate(session, channel);

                    if(channelp && (sizeof("exit-signal") + 13) <= datalen) {
                        /* set signal name (without SIG prefix) */
                        uint32_t namelen =
                            _libssh2_ntohu32(data + 9 + sizeof("exit-signal"));

                        if(namelen <= UINT_MAX - 1) {
                            channelp->exit_signal =
                                LIBSSH2_ALLOC(session, namelen + 1);
                        }
                        else {
                            channelp->exit_signal = NULL;
                        }

                        if(!channelp->exit_signal)
                            rc = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                                "memory for signal name");
                        else if((sizeof("exit-signal") + 13 + namelen <=
                                 datalen)) {
                            memcpy(channelp->exit_signal,
                                   data + 13 + sizeof("exit-signal"), namelen);
                            channelp->exit_signal[namelen] = '\0';
                            /* TODO: save error message and language tag */
                            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                           "Exit signal %s received for "
                                           "channel %lu/%lu",
                                           channelp->exit_signal,
                                           channelp->local.id,
                                           channelp->remote.id);
                        }
                    }
                }


                if(want_reply) {
                    unsigned char packet[5];
                  libssh2_packet_add_jump_point4:
                    session->packAdd_state = libssh2_NB_state_jump4;
                    packet[0] = SSH_MSG_CHANNEL_FAILURE;
                    memcpy(&packet[1], data + 1, 4);
                    rc = _libssh2_transport_send(session, packet, 5, NULL, 0);
                    if(rc == LIBSSH2_ERROR_EAGAIN)
                        return rc;
                }
            }
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return rc;

            /*
              byte      SSH_MSG_CHANNEL_CLOSE
              uint32    recipient channel
            */

        case SSH_MSG_CHANNEL_CLOSE:
            if(datalen >= 5)
                channelp =
                    _libssh2_channel_locate(session,
                                            _libssh2_ntohu32(data + 1));
            if(!channelp) {
                /* We may have freed already, just quietly ignore this... */
                LIBSSH2_FREE(session, data);
                session->packAdd_state = libssh2_NB_state_idle;
                return 0;
            }
            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "Close received for channel %lu/%lu",
                           channelp->local.id,
                           channelp->remote.id);

            channelp->remote.close = 1;
            channelp->remote.eof = 1;

            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;

            /*
              byte      SSH_MSG_CHANNEL_OPEN
              string    "session"
              uint32    sender channel
              uint32    initial window size
              uint32    maximum packet size
            */

        case SSH_MSG_CHANNEL_OPEN:
            if(datalen < 17)
                ;
            else if((datalen >= (sizeof("forwarded-tcpip") + 4)) &&
                     ((sizeof("forwarded-tcpip") - 1) ==
                      _libssh2_ntohu32(data + 1))
                     &&
                     (memcmp(data + 5, "forwarded-tcpip",
                             sizeof("forwarded-tcpip") - 1) == 0)) {

                /* init the state struct */
                memset(&session->packAdd_Qlstn_state, 0,
                       sizeof(session->packAdd_Qlstn_state));

              libssh2_packet_add_jump_point2:
                session->packAdd_state = libssh2_NB_state_jump2;
                rc = packet_queue_listener(session, data, datalen,
                                           &session->packAdd_Qlstn_state);
            }
            else if((datalen >= (sizeof("x11") + 4)) &&
                     ((sizeof("x11") - 1) == _libssh2_ntohu32(data + 1)) &&
                     (memcmp(data + 5, "x11", sizeof("x11") - 1) == 0)) {

                /* init the state struct */
                memset(&session->packAdd_x11open_state, 0,
                       sizeof(session->packAdd_x11open_state));

              libssh2_packet_add_jump_point3:
                session->packAdd_state = libssh2_NB_state_jump3;
                rc = packet_x11_open(session, data, datalen,
                                     &session->packAdd_x11open_state);
            }
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;

            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return rc;

            /*
              byte      SSH_MSG_CHANNEL_WINDOW_ADJUST
              uint32    recipient channel
              uint32    bytes to add
            */
        case SSH_MSG_CHANNEL_WINDOW_ADJUST:
            if(datalen < 9)
                ;
            else {
                uint32_t bytestoadd = _libssh2_ntohu32(data + 5);
                channelp =
                    _libssh2_channel_locate(session,
                                            _libssh2_ntohu32(data + 1));
                if(channelp) {
                    channelp->local.window_size += bytestoadd;

                    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                   "Window adjust for channel %lu/%lu, "
                                   "adding %lu bytes, new window_size=%lu",
                                   channelp->local.id,
                                   channelp->remote.id,
                                   bytestoadd,
                                   channelp->local.window_size);
                }
            }
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return 0;
        default:
            break;
        }

        session->packAdd_state = libssh2_NB_state_sent;
    }

    if(session->packAdd_state == libssh2_NB_state_sent) {
        LIBSSH2_PACKET *packetp =
            LIBSSH2_ALLOC(session, sizeof(LIBSSH2_PACKET));
        if(!packetp) {
            _libssh2_debug(session, LIBSSH2_ERROR_ALLOC,
                           "memory for packet");
            LIBSSH2_FREE(session, data);
            session->packAdd_state = libssh2_NB_state_idle;
            return LIBSSH2_ERROR_ALLOC;
        }
        packetp->data = data;
        packetp->data_len = datalen;
        packetp->data_head = data_head;

        _libssh2_list_add(&session->packets, &packetp->node);

        session->packAdd_state = libssh2_NB_state_sent1;
    }

    if((msg == SSH_MSG_KEXINIT &&
         !(session->state & LIBSSH2_STATE_EXCHANGING_KEYS)) ||
        (session->packAdd_state == libssh2_NB_state_sent2)) {
        if(session->packAdd_state == libssh2_NB_state_sent1) {
            /*
             * Remote wants new keys
             * Well, it's already in the brigade,
             * let's just call back into ourselves
             */
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Renegotiating Keys");

            session->packAdd_state = libssh2_NB_state_sent2;
        }

        /*
         * The KEXINIT message has been added to the queue.  The packAdd and
         * readPack states need to be reset because _libssh2_kex_exchange
         * (eventually) calls upon _libssh2_transport_read to read the rest of
         * the key exchange conversation.
         */
        session->readPack_state = libssh2_NB_state_idle;
        session->packet.total_num = 0;
        session->packAdd_state = libssh2_NB_state_idle;
        session->fullpacket_state = libssh2_NB_state_idle;

        memset(&session->startup_key_state, 0, sizeof(key_exchange_state_t));

        /*
         * If there was a key reexchange failure, let's just hope we didn't
         * send NEWKEYS yet, otherwise remote will drop us like a rock
         */
        rc = _libssh2_kex_exchange(session, 1, &session->startup_key_state);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
    }

    session->packAdd_state = libssh2_NB_state_idle;
    return 0;
}

```

这段代码是一个名为`_libssh2_packet_ask`的函数，它是用来扫描 brace（选定的两个或多个）中匹配的包，如果需要，还可以使用 `poll` 函数来查找早期版本的包。

函数接受一个 `LIBSSH2_SESSION` 类型的会话，一个指向 `unsigned char` 类型数据缓冲区的指针，以及一个指向 `size_t` 类型数据有效长度的指针。还有两个指向 `LIBSSH2_PACKET` 类型结构体的指针，一个用于存储搜索到的第一个匹配的包，另一个用于存储匹配到的包的数据。

函数的核心部分是一系列的循环，它们从 `session->packets` 开始，遍历 `packets` 链表，检查每个包是否匹配所要查找的包。如果找到了匹配的包，函数会将匹配的包的数据复制到输入缓冲区，然后使用 `_libssh2_list_remove` 函数从 `session->packets` 中删除该包，最后释放该包的内存并返回 0。

如果所有匹配的包均未找到，函数将返回 -1，或者没有匹配的包。


```cpp
/*
 * _libssh2_packet_ask
 *
 * Scan the brigade for a matching packet type, optionally poll the socket for
 * a packet first
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

            /* unlink struct from session->packets */
            _libssh2_list_remove(&packet->node);

            LIBSSH2_FREE(session, packet);

            return 0;
        }
        packet = _libssh2_list_next(&packet->node);
    }
    return -1;
}

```

这段代码是一个名为`_libssh2_packet_askv`的函数，它用于扫描服务器监听的套接字中收到的数据包类型。

函数接收两个参数：一个`LIBSSH2_SESSION`类型的`session`表示当前SSH会话，一个包含字节数组`packet_types`的指针。另一个参数是一个`unsigned char *`类型的指针`data`，用于存储接收到的数据包，以及一个包含字节数组`match_buf`的指针，用于存储与数据包匹配的感兴趣的字符串。

函数内部首先定义了一个`packet_types_len`变量，用于存储`packet_types`字符串的长度。接下来，函数遍历`packet_types`字符串中的所有元素。对于每个数据包类型，函数调用一个名为`_libssh2_packet_ask`的函数，传递一个`unsigned char *`类型的数据包指针`packet`，一个指向数据的长度`data_len`，一个选项参数`match_ofs`表示匹配数据的位置，以及一个指向字符串的`unsigned char *`类型的补增缓冲区`match_buf`。函数返回一个整数，表示匹配到感兴趣数据包的数量，或者-1，表示没有匹配到感兴趣的数据包。

如果某个数据包类型可以被成功匹配到，函数将返回0；否则，函数返回-1。


```cpp
/*
 * libssh2_packet_askv
 *
 * Scan for any of a list of packet types in the brigade, optionally poll the
 * socket for a packet first
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

```



This function appears to be responsible for selecting a packet from a packet queue based on certain conditions. Here's how it works:

1. It checks if there is a packet queue available for selection. If there is, it checks if the current time is after the start time of the queue. If it is, the function retrieves the packet from the queue.

2. If the queue does not have any packet available, it waits until there is a packet available or the timeout occurs.

3. If the socket is still connected, it reads data from the socket.

4. If the socket is not connected, it waits until it is or until the timeout occurs.

5. If the timeout occurs, the function returns an error indicating that there is no packet available.

6. If the socket disconnects, the function returns an error indicating that


```cpp
/*
 * _libssh2_packet_require
 *
 * Loops _libssh2_transport_read() until the packet requested is available
 * SSH_DISCONNECT or a SOCKET_DISCONNECTED will cause a bailout
 *
 * Returns negative on error
 * Returns 0 when it has taken care of the requested packet.
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
            /* A packet was available in the packet brigade */
            return 0;
        }

        state->start = time(NULL);
    }

    while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        int ret = _libssh2_transport_read(session);
        if(ret == LIBSSH2_ERROR_EAGAIN)
            return ret;
        else if(ret < 0) {
            state->start = 0;
            /* an error which is not just because of blocking */
            return ret;
        }
        else if(ret == packet_type) {
            /* Be lazy, let packet_ask pull it out of the brigade */
            ret = _libssh2_packet_ask(session, packet_type, data, data_len,
                                      match_ofs, match_buf, match_len);
            state->start = 0;
            return ret;
        }
        else if(ret == 0) {
            /* nothing available, wait until data arrives or we time out */
            long left = LIBSSH2_READ_TIMEOUT - (long)(time(NULL) -
                                                      state->start);

            if(left <= 0) {
                state->start = 0;
                return LIBSSH2_ERROR_TIMEOUT;
            }
            return -1; /* no packet available yet */
        }
    }

    /* Only reached if the socket died */
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}

```

这段代码是一个用于在监听套接字上轮询数据的函数。它将所有可用的数据包提取出来，并将它们发送到服务器。以下是函数的实现细节：

```cpp
int libssh2_nb_轮询(libssh2_session_t* session,
                           void* all_packets,
                           int all_packets_len,
                           int current_packet,
                           int max_轮询_retries,
                           int* state_idle_timeout,
                           int* state_active_timeout)
{
   int i, packet_idle;
   unsigned char data[256];
   int data_len;
   int ret;
   int state;

   if(_libssh2_packet_askv(session, all_packets,
                                        data, &data_len, 0,
                                            NULL, 0) == 0) {
       state = LIBSSH2_NB_state_idle;
       return 0;
   }

   state = LIBSSH2_NB_state_active;
   max_轮询_retries = 0;

   while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
       ret = _libssh2_transport_read(session);
       if(ret == LIBSSH2_ERROR_EAGAIN) {
           return 0;
       }
       else if(ret < 0) {
           *state = libssh2_NB_state_idle;
           return 0;
       }
       else if(ret == 0) {
           /* FIXME: this might busyloop */
           continue;
       }

       /* Be lazy, let packet_ask pull it out of the brigade */
       if(0 ==
           _libssh2_packet_ask(session, (unsigned char)ret,
                                        &data, &data_len, 0, NULL, 0)) {
           /* Smoke 'em if you got 'em */
           LIBSSH2_FREE(session, data);
           *state = libssh2_NB_state_active;
           return ret;
       }
       else {
           state_idle_timeout = 0;
           state_active_timeout = 0;
           max_轮询_retries++;
       }
   }

   return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}
```

这个函数接受一个套接字对象，一个可用的数据包缓冲区，以及发送给服务器的数据缓冲区。它尝试从队列中获取一个可用的数据包，并发送它到服务器。如果数据包可用，它将更新状态并返回状态。如果套接字连接出现故障，它将重新轮询，并继续尝试发送数据包。

该函数的核心是使用 `_libssh2_packet_ask` 函数来获取可用的数据包。如果这个函数返回一个负数，那么说明数据包队列中暂时没有可用的数据包，此时函数将进入一个轮询状态，不断尝试发送数据包。如果调用 `_libssh2_transport_read` 函数来读取数据，如果读取数据成功，将数据解析为字节并更新状态。


```cpp
/*
 * _libssh2_packet_burn
 *
 * Loops _libssh2_transport_read() until any packet is available and promptly
 * discards it.
 * Used during KEX exchange to discard badly guessed KEX_INIT packets
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
            /* A packet was available in the packet brigade, burn it */
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

        /* Be lazy, let packet_ask pull it out of the brigade */
        if(0 ==
            _libssh2_packet_ask(session, (unsigned char)ret,
                                         &data, &data_len, 0, NULL, 0)) {
            /* Smoke 'em if you got 'em */
            LIBSSH2_FREE(session, data);
            *state = libssh2_NB_state_idle;
            return ret;
        }
    }

    /* Only reached if the socket died */
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}

```

这段代码是一个在 SSH 协议中处理数据包的函数。负责接收来自服务器的数据包，并在包衣柜中查找匹配的数据包。如果找到匹配的数据包，则将其从包衣柜中取出，并更新接收者的状态。如果所有的数据包都被取出，则状态机将进入等待状态，直到客户端发送新的数据包。

函数参数说明：

- data: 要接收的数据包的缓冲区，数组长度为 data_len。
- match_ofs: 要匹配匹配的 offset。
- match_buf: 要匹配的缓冲区，长度为 match_len。
- state: 状态机指针，用于跟踪当前状态。

函数实现中，首先判断客户端是否接收到了数据包。如果接收到数据包，则执行以下操作：

1. 将数据包从包衣柜中取出。
2. 更新接收者的状态。

如果所有的数据包都被取出，则状态机将进入等待状态，直到客户端发送新的数据包。函数还实现了一个辅助函数，用于计算从客户端到服务器的时延。


```cpp
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
    if(_libssh2_packet_askv(session, packet_types, data, data_len, match_ofs,
                             match_buf, match_len) == 0) {
        /* One of the packets listed was available in the packet brigade */
        state->start = 0;
        return 0;
    }

    if(state->start == 0) {
        state->start = time(NULL);
    }

    while(session->socket_state != LIBSSH2_SOCKET_DISCONNECTED) {
        int ret = _libssh2_transport_read(session);
        if((ret < 0) && (ret != LIBSSH2_ERROR_EAGAIN)) {
            state->start = 0;
            return ret;
        }
        if(ret <= 0) {
            long left = LIBSSH2_READ_TIMEOUT -
                (long)(time(NULL) - state->start);

            if(left <= 0) {
                state->start = 0;
                return LIBSSH2_ERROR_TIMEOUT;
            }
            else if(ret == LIBSSH2_ERROR_EAGAIN) {
                return ret;
            }
        }

        if(strchr((char *) packet_types, ret)) {
            /* Be lazy, let packet_ask pull it out of the brigade */
            int ret = _libssh2_packet_askv(session, packet_types, data,
                                        data_len, match_ofs, match_buf,
                                        match_len);
            state->start = 0;
            return ret;
        }
    }

    /* Only reached if the socket died */
    state->start = 0;
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}


```