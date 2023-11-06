# Nmap源码解析 94

# `libssh2/src/channel.c`

This is a text-based markup email that appears to be a Microsoft Word document. It appears to be a大地构造图，即在地图上標记一些地物或设施的图例。

The email address at the bottom indicates that it was sent from "dottedmag.net" and the sender is Daniel Stenberg.

The recipient's email address is missing.

The email body contains a series of instructions on how to navigate the PDF document. It is a PDF document that includes images, tables, and a hyperlink. The PDF has a watermark that says "Notic deck 2018. All rights reserved."

The document is a map of the city of双十字，瑞士和中国，展示了大量建筑物，道路，和其他设施。


```cpp
/* Copyright (c) 2004-2007 Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2005 Mikhail Gusarov <dottedmag@dottedmag.net>
 * Copyright (c) 2008-2019 by Daniel Stenberg
 *
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

这段代码是一个基于libssh2库的通道客户端，它包括了多态的输入输出库，因此它允许客户端使用不同的协议进行连接。

首先，它通过头文件引入了"libssh2_priv.h"，这是libssh2库的私有头文件，它定义了与SSH2协议相关的函数。接下来，它通过条件编译语句检查是否启用了Unix客户端支持的多态特性，如果是，它就引入了<unistd.h>头文件。

接下来，它通过条件编译语句检查是否启用了不同数据类型的支持，如果是，它就引入了<inttypes.h>头文件。然后，它通过头文件引入了"channel.h"、"transport.h"、"packet.h"和"session.h"，这些头文件定义了与通道相关的接口。

最后，它定义了一系列函数，包括"connect"、"send"和"send_char"，用于连接到远程主机，发送数据和发送字符。


```cpp
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

```

这段代码是一个用于 Linux SSH 的函数，它的作用是确定可以分配给当前会话的下一个频道号。

代码首先通过 `session->next_channel` 属性获取当前会话的下一个频道号，然后遍历当前频道，查找频道号是否已经分配给该频道，如果是，则使用下一个频道号，如果不是，则将频道号递增，并将该频道加入列表中。

接下来，代码使用一个短路循环来避免在忘记的频道上浪费时间，该循环会从遍历的下一个频道开始，直到找到第一个频道号大于当前频道号的频道为止。

最后，代码将当前频道号递增，并将结果输出到 `_libssh2_debug` 函数中，用于调试信息。


```cpp
/*
 *  _libssh2_channel_nextid
 *
 * Determine the next channel ID we can use at our end
 */
uint32_t
_libssh2_channel_nextid(LIBSSH2_SESSION * session)
{
    uint32_t id = session->next_channel;
    LIBSSH2_CHANNEL *channel;

    channel = _libssh2_list_first(&session->channels);

    while(channel) {
        if(channel->local.id > id) {
            id = channel->local.id;
        }
        channel = _libssh2_list_next(&channel->node);
    }

    /* This is a shortcut to avoid waiting for close packets on channels we've
     * forgotten about, This *could* be a problem if we request and close 4
     * billion or so channels in too rapid succession for the remote end to
     * respond, but the worst case scenario is that some data meant for
     * another channel Gets picked up by the new one.... Pretty unlikely all
     * told...
     */
    session->next_channel = id + 1;
    _libssh2_debug(session, LIBSSH2_TRACE_CONN, "Allocated new channel ID#%lu",
                   id);
    return id;
}

```

这段代码定义了一个名为`_libssh2_channel_locate`的函数，它的作用是查找一个名为`channel_id`的频道 pointer。频道 pointer 是一种数据结构，用于表示频道对象，通常包括频道的 ID、名称、位置和状态等。

函数接受两个参数：一个`LIBSSH2_SESSION`结构体指针和一个`uint32_t`类型的整数。函数首先定义了一个`channel`变量，用于存储当前正在查找的频道指针。接着，定义了一个`LIBSSH2_LISTENER`结构体指针`l`，用于存储所有监听器。

函数的核心部分使用两个循环来遍历`session->channels`和`session->listeners`。在`channel`循环中，如果当前频道ID与`channel_id`相等，就返回该频道指针。如果循环结束后仍然没有找到频道ID，就检查每个监听器，看看是否包含当前频道ID。如果是，则返回监听器中的频道指针。

如果循环仍然没有找到频道ID，那么函数返回`NULL`，表示没有找到符合条件的频道。


```cpp
/*
 * _libssh2_channel_locate
 *
 * Locate a channel pointer by number
 */
LIBSSH2_CHANNEL *
_libssh2_channel_locate(LIBSSH2_SESSION *session, uint32_t channel_id)
{
    LIBSSH2_CHANNEL *channel;
    LIBSSH2_LISTENER *l;

    for(channel = _libssh2_list_first(&session->channels);
        channel;
        channel = _libssh2_list_next(&channel->node)) {
        if(channel->local.id == channel_id)
            return channel;
    }

    /* We didn't find the channel in the session, let's then check its
       listeners since each listener may have its own set of pending channels
    */
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

```

This is a function in the OpenSSH library that handles the case where a channel has failed open. It takes in a `session` object, which is an instance of the `SSH2` class, and handles the situation accordingly.

The function has several options:

1. If the session has an open data channel, it frees the data and sets the data to `null`.
2. If the session has an open packet channel, it frees the packet and sets the packet to `null`.
3. If the session has an open channel, it frees the channel data, frees the local ID of the channel, and sets the packet to `null`. It also clears out any packets meant for this channel.

If the function completes successfully, the session's open state is updated to `libssh2_NB_state_idle`. If any errors occur, the function returns a `NULL` pointer.


```cpp
/*
 * _libssh2_channel_open
 *
 * Establish a generic session channel
 */
LIBSSH2_CHANNEL *
_libssh2_channel_open(LIBSSH2_SESSION * session, const char *channel_type,
                      uint32_t channel_type_len,
                      uint32_t window_size,
                      uint32_t packet_size,
                      const unsigned char *message,
                      size_t message_len)
{
    static const unsigned char reply_codes[3] = {
        SSH_MSG_CHANNEL_OPEN_CONFIRMATION,
        SSH_MSG_CHANNEL_OPEN_FAILURE,
        0
    };
    unsigned char *s;
    int rc;

    if(session->open_state == libssh2_NB_state_idle) {
        session->open_channel = NULL;
        session->open_packet = NULL;
        session->open_data = NULL;
        /* 17 = packet_type(1) + channel_type_len(4) + sender_channel(4) +
         * window_size(4) + packet_size(4) */
        session->open_packet_len = channel_type_len + 17;
        session->open_local_channel = _libssh2_channel_nextid(session);

        /* Zero the whole thing out */
        memset(&session->open_packet_requirev_state, 0,
               sizeof(session->open_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Opening Channel - win %d pack %d", window_size,
                       packet_size);
        session->open_channel =
            LIBSSH2_CALLOC(session, sizeof(LIBSSH2_CHANNEL));
        if(!session->open_channel) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate space for channel data");
            return NULL;
        }
        session->open_channel->channel_type_len = channel_type_len;
        session->open_channel->channel_type =
            LIBSSH2_ALLOC(session, channel_type_len);
        if(!session->open_channel->channel_type) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Failed allocating memory for channel type name");
            LIBSSH2_FREE(session, session->open_channel);
            session->open_channel = NULL;
            return NULL;
        }
        memcpy(session->open_channel->channel_type, channel_type,
               channel_type_len);

        /* REMEMBER: local as in locally sourced */
        session->open_channel->local.id = session->open_local_channel;
        session->open_channel->remote.window_size = window_size;
        session->open_channel->remote.window_size_initial = window_size;
        session->open_channel->remote.packet_size = packet_size;
        session->open_channel->session = session;

        _libssh2_list_add(&session->channels,
                          &session->open_channel->node);

        s = session->open_packet =
            LIBSSH2_ALLOC(session, session->open_packet_len);
        if(!session->open_packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate temporary space for packet");
            goto channel_error;
        }
        *(s++) = SSH_MSG_CHANNEL_OPEN;
        _libssh2_store_str(&s, channel_type, channel_type_len);
        _libssh2_store_u32(&s, session->open_local_channel);
        _libssh2_store_u32(&s, window_size);
        _libssh2_store_u32(&s, packet_size);

        /* Do not copy the message */

        session->open_state = libssh2_NB_state_created;
    }

    if(session->open_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session,
                                     session->open_packet,
                                     session->open_packet_len,
                                     message, message_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending channel-open request");
            return NULL;
        }
        else if(rc) {
            _libssh2_error(session, rc,
                           "Unable to send channel-open request");
            goto channel_error;
        }

        session->open_state = libssh2_NB_state_sent;
    }

    if(session->open_state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_requirev(session, reply_codes,
                                      &session->open_data,
                                      &session->open_data_len, 1,
                                      session->open_packet + 5 +
                                      channel_type_len, 4,
                                      &session->open_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN, "Would block");
            return NULL;
        }
        else if(rc) {
            _libssh2_error(session, rc, "Unexpected error");
            goto channel_error;
        }

        if(session->open_data_len < 1) {
            _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                           "Unexpected packet size");
            goto channel_error;
        }

        if(session->open_data[0] == SSH_MSG_CHANNEL_OPEN_CONFIRMATION) {

            if(session->open_data_len < 17) {
                _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                               "Unexpected packet size");
                goto channel_error;
            }

            session->open_channel->remote.id =
                _libssh2_ntohu32(session->open_data + 5);
            session->open_channel->local.window_size =
                _libssh2_ntohu32(session->open_data + 9);
            session->open_channel->local.window_size_initial =
                _libssh2_ntohu32(session->open_data + 9);
            session->open_channel->local.packet_size =
                _libssh2_ntohu32(session->open_data + 13);
            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "Connection Established - ID: %lu/%lu win: %lu/%lu"
                           " pack: %lu/%lu",
                           session->open_channel->local.id,
                           session->open_channel->remote.id,
                           session->open_channel->local.window_size,
                           session->open_channel->remote.window_size,
                           session->open_channel->local.packet_size,
                           session->open_channel->remote.packet_size);
            LIBSSH2_FREE(session, session->open_packet);
            session->open_packet = NULL;
            LIBSSH2_FREE(session, session->open_data);
            session->open_data = NULL;

            session->open_state = libssh2_NB_state_idle;
            return session->open_channel;
        }

        if(session->open_data[0] == SSH_MSG_CHANNEL_OPEN_FAILURE) {
            unsigned int reason_code =
                _libssh2_ntohu32(session->open_data + 5);
            switch(reason_code) {
            case SSH_OPEN_ADMINISTRATIVELY_PROHIBITED:
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Channel open failure "
                               "(administratively prohibited)");
                break;
            case SSH_OPEN_CONNECT_FAILED:
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Channel open failure (connect failed)");
                break;
            case SSH_OPEN_UNKNOWN_CHANNELTYPE:
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Channel open failure (unknown channel type)");
                break;
            case SSH_OPEN_RESOURCE_SHORTAGE:
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Channel open failure (resource shortage)");
                break;
            default:
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Channel open failure");
            }
        }
    }

  channel_error:

    if(session->open_data) {
        LIBSSH2_FREE(session, session->open_data);
        session->open_data = NULL;
    }
    if(session->open_packet) {
        LIBSSH2_FREE(session, session->open_packet);
        session->open_packet = NULL;
    }
    if(session->open_channel) {
        unsigned char channel_id[4];
        LIBSSH2_FREE(session, session->open_channel->channel_type);

        _libssh2_list_remove(&session->open_channel->node);

        /* Clear out packets meant for this channel */
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
            LIBSSH2_FREE(session, session->open_data);
            session->open_data = NULL;
        }

        LIBSSH2_FREE(session, session->open_channel);
        session->open_channel = NULL;
    }

    session->open_state = libssh2_NB_state_idle;
    return NULL;
}

```

这段代码是一个用于 libssh2_channel_open_ex 的函数，它用于建立一个通用会话通道。函数接受一个 libssh2_Session 类型的输入参数，以及一个字符串类型的参数，用于指定会话类型和消息类型。

函数内部首先检查输入参数是否为空，如果是，则直接返回 NULL。然后进行一系列错误处理，包括调整 block 调整和返回通道指针。最后，返回新创建的 libssh2_channel_t 类型的指针，它包含了建立好的会话 channel。


```cpp
/*
 * libssh2_channel_open_ex
 *
 * Establish a generic session channel
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_open_ex(LIBSSH2_SESSION *session, const char *type,
                        unsigned int type_len,
                        unsigned int window_size, unsigned int packet_size,
                        const char *msg, unsigned int msg_len)
{
    LIBSSH2_CHANNEL *ptr;

    if(!session)
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, session,
                       _libssh2_channel_open(session, type, type_len,
                                             window_size, packet_size,
                                             (unsigned char *)msg,
                                             msg_len));
    return ptr;
}

```

This function appears to be used to create a direct-tcpip session from a specified host and port to the specified host and port. It does this by opening a new session, creating a direct-tcpip connection, and sending the connection details to the remote host.

The function takes four arguments: a session object, a host string, a port number, and a specified host string. It returns a Boolean value indicating whether the session was successful. If the function fails, it returns NULL.


```cpp
/*
 * libssh2_channel_direct_tcpip_ex
 *
 * Tunnel TCP/IP connect through the SSH session to direct host/port
 */
static LIBSSH2_CHANNEL *
channel_direct_tcpip(LIBSSH2_SESSION * session, const char *host,
                     int port, const char *shost, int sport)
{
    LIBSSH2_CHANNEL *channel;
    unsigned char *s;

    if(session->direct_state == libssh2_NB_state_idle) {
        session->direct_host_len = strlen(host);
        session->direct_shost_len = strlen(shost);
        /* host_len(4) + port(4) + shost_len(4) + sport(4) */
        session->direct_message_len =
            session->direct_host_len + session->direct_shost_len + 16;

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting direct-tcpip session from %s:%d to %s:%d",
                       shost, sport, host, port);

        s = session->direct_message =
            LIBSSH2_ALLOC(session, session->direct_message_len);
        if(!session->direct_message) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for "
                           "direct-tcpip connection");
            return NULL;
        }
        _libssh2_store_str(&s, host, session->direct_host_len);
        _libssh2_store_u32(&s, port);
        _libssh2_store_str(&s, shost, session->direct_shost_len);
        _libssh2_store_u32(&s, sport);
    }

    channel =
        _libssh2_channel_open(session, "direct-tcpip",
                              sizeof("direct-tcpip") - 1,
                              LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                              LIBSSH2_CHANNEL_PACKET_DEFAULT,
                              session->direct_message,
                              session->direct_message_len);

    if(!channel &&
        libssh2_session_last_errno(session) == LIBSSH2_ERROR_EAGAIN) {
        /* The error code is still set to LIBSSH2_ERROR_EAGAIN, set our state
           to created to avoid re-creating the package on next invoke */
        session->direct_state = libssh2_NB_state_created;
        return NULL;
    }
    /* by default we set (keep?) idle state... */
    session->direct_state = libssh2_NB_state_idle;

    LIBSSH2_FREE(session, session->direct_message);
    session->direct_message = NULL;

    return channel;
}

```

这段代码定义了一个名为`libssh2_channel_direct_tcpip_ex`的函数，属于`libssh2_channel`类型。

它的作用是建立通过SSH会话直接连接到指定主机和端口的TCP/IP隧道。

具体实现过程如下：

1. 首先确保`session`参数指针不为空。
2. 然后定义`host`和`port`参数，分别指定了要连接的主机名和端口号。
3. 接着定义`shost`参数，指定了要连接到的主机，它会被传递给`channel_direct_tcpip`函数。
4. `port`参数同样指定了要连接的端口号。
5. 在`channel_direct_tcpip`函数中，首先检查`session`是否为空，如果是，则返回`NULL`。
6. 否则，通过`channel_direct_tcpip`函数建立TCP/IP隧道，返回建立的`CHANNEL`指针。
7. 最后，将建立的`CHANNEL`指针返回，这样就可以通过这个函数来连接到指定的主机和端口了。


```cpp
/*
 * libssh2_channel_direct_tcpip_ex
 *
 * Tunnel TCP/IP connect through the SSH session to direct host/port
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_direct_tcpip_ex(LIBSSH2_SESSION *session, const char *host,
                                int port, const char *shost, int sport)
{
    LIBSSH2_CHANNEL *ptr;

    if(!session)
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, session,
                       channel_direct_tcpip(session, host, port,
                                            shost, sport));
    return ptr;
}

```

This function appears to be a part of a library that implements the SSH protocol. It appears to be responsible for managing the flow of data between the client and server, and it includes a function for dynamically allocated and deallocated listening ports.

The function takes a session object, a pointer to an SSH2 listener object, and a pointer to a data structure that contains information about the incoming data. It appears to parse the data and extract the offending port and then returns the previous listener.

If the first element in the data structure is not an SSH2 port, the function converts it to an integer and returns it. It also appears to append the new listener to the global list of listeners and update the queue maximum and maximum queue size.

It looks like the function also handles the case where the data contains a request to establish an SSH connection and a failure to do so. If the request fails, the function returns without the new listener. If the request succeeds, the function sets the session's forward listening state and returns the new listener.


```cpp
/*
 * channel_forward_listen
 *
 * Bind a port on the remote host and listen for connections
 */
static LIBSSH2_LISTENER *
channel_forward_listen(LIBSSH2_SESSION * session, const char *host,
                       int port, int *bound_port, int queue_maxsize)
{
    unsigned char *s;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_REQUEST_SUCCESS, SSH_MSG_REQUEST_FAILURE, 0 };
    int rc;

    if(!host)
        host = "0.0.0.0";

    if(session->fwdLstn_state == libssh2_NB_state_idle) {
        session->fwdLstn_host_len = strlen(host);
        /* 14 = packet_type(1) + request_len(4) + want_replay(1) + host_len(4)
           + port(4) */
        session->fwdLstn_packet_len =
            session->fwdLstn_host_len + (sizeof("tcpip-forward") - 1) + 14;

        /* Zero the whole thing out */
        memset(&session->fwdLstn_packet_requirev_state, 0,
               sizeof(session->fwdLstn_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting tcpip-forward session for %s:%d", host,
                       port);

        s = session->fwdLstn_packet =
            LIBSSH2_ALLOC(session, session->fwdLstn_packet_len);
        if(!session->fwdLstn_packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for setenv packet");
            return NULL;
        }

        *(s++) = SSH_MSG_GLOBAL_REQUEST;
        _libssh2_store_str(&s, "tcpip-forward", sizeof("tcpip-forward") - 1);
        *(s++) = 0x01;          /* want_reply */

        _libssh2_store_str(&s, host, session->fwdLstn_host_len);
        _libssh2_store_u32(&s, port);

        session->fwdLstn_state = libssh2_NB_state_created;
    }

    if(session->fwdLstn_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session,
                                     session->fwdLstn_packet,
                                     session->fwdLstn_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending global-request packet for "
                           "forward listen request");
            return NULL;
        }
        else if(rc) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send global-request packet for forward "
                           "listen request");
            LIBSSH2_FREE(session, session->fwdLstn_packet);
            session->fwdLstn_packet = NULL;
            session->fwdLstn_state = libssh2_NB_state_idle;
            return NULL;
        }
        LIBSSH2_FREE(session, session->fwdLstn_packet);
        session->fwdLstn_packet = NULL;

        session->fwdLstn_state = libssh2_NB_state_sent;
    }

    if(session->fwdLstn_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      0, NULL, 0,
                                      &session->fwdLstn_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN, "Would block");
            return NULL;
        }
        else if(rc || (data_len < 1)) {
            _libssh2_error(session, LIBSSH2_ERROR_PROTO, "Unknown");
            session->fwdLstn_state = libssh2_NB_state_idle;
            return NULL;
        }

        if(data[0] == SSH_MSG_REQUEST_SUCCESS) {
            LIBSSH2_LISTENER *listener;

            listener = LIBSSH2_CALLOC(session, sizeof(LIBSSH2_LISTENER));
            if(!listener)
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "Unable to allocate memory for listener queue");
            else {
                listener->host =
                    LIBSSH2_ALLOC(session, session->fwdLstn_host_len + 1);
                if(!listener->host) {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory "
                                   "for listener queue");
                    LIBSSH2_FREE(session, listener);
                    listener = NULL;
                }
                else {
                    listener->session = session;
                    memcpy(listener->host, host, session->fwdLstn_host_len);
                    listener->host[session->fwdLstn_host_len] = 0;
                    if(data_len >= 5 && !port) {
                        listener->port = _libssh2_ntohu32(data + 1);
                        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                                       "Dynamic tcpip-forward port "
                                       "allocated: %d",
                                       listener->port);
                    }
                    else
                        listener->port = port;

                    listener->queue_size = 0;
                    listener->queue_maxsize = queue_maxsize;

                    /* append this to the parent's list of listeners */
                    _libssh2_list_add(&session->listeners, &listener->node);

                    if(bound_port) {
                        *bound_port = listener->port;
                    }
                }
            }

            LIBSSH2_FREE(session, data);
            session->fwdLstn_state = libssh2_NB_state_idle;
            return listener;
        }
        else if(data[0] == SSH_MSG_REQUEST_FAILURE) {
            LIBSSH2_FREE(session, data);
            _libssh2_error(session, LIBSSH2_ERROR_REQUEST_DENIED,
                           "Unable to complete request for forward-listen");
            session->fwdLstn_state = libssh2_NB_state_idle;
            return NULL;
        }
    }

    session->fwdLstn_state = libssh2_NB_state_idle;

    return NULL;
}

```

这段代码是一个用于监听 SSH 连接的 C 语言函数，属于 SSH2 库的一部分。它的作用是绑定一个远程主机的某个端口，监听来自该主机的新连接，并将连接转发给服务器。以下是代码的主要步骤：

1. 首先定义一个名为 `libssh2_channel_forward_listen_ex` 的函数，它接受一个 `LIBSSH2_SESSION` 类型的参数 `session`，一个指向字符串 `host` 的整数参数 `port`，以及一个指向整数 `bound_port` 的整数参数 `queue_maxsize`。

2. 在函数内部，首先检查传递给它的 `session` 是否为 `NULL`，如果是，则返回 `NULL`，表示函数没有正确处理输入参数。

3. 如果 `session` 不是 `NULL`，那么执行以下操作：首先尝试调用 `channel_forward_listen` 函数，将其作为参数传入，得到一个指向 `LIBSSH2_LISTENER` 类型的指针 `ptr`。然后，使用 `BLOCK_ADJUST_ERRNO` 函数来处理可能返回的错误，确保函数调用成功。

4. 最后，返回 `ptr`，它是一个指向 `LIBSSH2_LISTENER` 类型的指针，指向远程主机上的监听器。

函数的作用是监听来自远程主机的一个端口，如果有新连接生成，则将其转发给服务器。返回的 `LIBSSH2_LISTENER` 类型的指针可用于进一步与服务器建立连接。


```cpp
/*
 * libssh2_channel_forward_listen_ex
 *
 * Bind a port on the remote host and listen for connections
 */
LIBSSH2_API LIBSSH2_LISTENER *
libssh2_channel_forward_listen_ex(LIBSSH2_SESSION *session, const char *host,
                                  int port, int *bound_port, int queue_maxsize)
{
    LIBSSH2_LISTENER *ptr;

    if(!session)
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, session,
                       channel_forward_listen(session, host, port, bound_port,
                                              queue_maxsize));
    return ptr;
}

```

This function appears to handle the process of sending a forward request from the listening SSH connection to the remote host. It does this by first creating a new global-request packet and sending it to the remote host using the libssh2_transport\_send function. If the send fails, it sets the state of the listening channel to libssh2\_NB\_state\_sent and returns an error code. If the send succeeds, it sets the state of the listening channel to libssh2\_NB\_state\_sent and returns 0.
It should be noted that this function should be called from a function that is responsible for managing the connection, such as the handle\_client function in an SSH server.


```cpp
/*
 * _libssh2_channel_forward_cancel
 *
 * Stop listening on a remote port and free the listener
 * Toss out any pending (un-accept()ed) connections
 *
 * Return 0 on success, LIBSSH2_ERROR_EAGAIN if would block, -1 on error
 */
int _libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener)
{
    LIBSSH2_SESSION *session = listener->session;
    LIBSSH2_CHANNEL *queued;
    unsigned char *packet, *s;
    size_t host_len = strlen(listener->host);
    /* 14 = packet_type(1) + request_len(4) + want_replay(1) + host_len(4) +
       port(4) */
    size_t packet_len =
        host_len + 14 + sizeof("cancel-tcpip-forward") - 1;
    int rc;
    int retcode = 0;

    if(listener->chanFwdCncl_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Cancelling tcpip-forward session for %s:%d",
                       listener->host, listener->port);

        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for setenv packet");
            return LIBSSH2_ERROR_ALLOC;
        }

        *(s++) = SSH_MSG_GLOBAL_REQUEST;
        _libssh2_store_str(&s, "cancel-tcpip-forward",
                           sizeof("cancel-tcpip-forward") - 1);
        *(s++) = 0x00;          /* want_reply */

        _libssh2_store_str(&s, listener->host, host_len);
        _libssh2_store_u32(&s, listener->port);

        listener->chanFwdCncl_state = libssh2_NB_state_created;
    }
    else {
        packet = listener->chanFwdCncl_data;
    }

    if(listener->chanFwdCncl_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, packet, packet_len, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending forward request");
            listener->chanFwdCncl_data = packet;
            return rc;
        }
        else if(rc) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send global-request packet for forward "
                           "listen request");
            /* set the state to something we don't check for, for the
               unfortunate situation where we get an EAGAIN further down
               when trying to bail out due to errors! */
            listener->chanFwdCncl_state = libssh2_NB_state_sent;
            retcode = LIBSSH2_ERROR_SOCKET_SEND;
        }
        LIBSSH2_FREE(session, packet);

        listener->chanFwdCncl_state = libssh2_NB_state_sent;
    }

    queued = _libssh2_list_first(&listener->queue);
    while(queued) {
        LIBSSH2_CHANNEL *next = _libssh2_list_next(&queued->node);

        rc = _libssh2_channel_free(queued);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        queued = next;
    }
    LIBSSH2_FREE(session, listener->host);

    /* remove this entry from the parent's list of listeners */
    _libssh2_list_remove(&listener->node);

    LIBSSH2_FREE(session, listener);

    return retcode;
}

```

这段代码是一个名为 `libssh2_channel_forward_cancel` 的函数，属于 `libssh2_client` 系列的函数。它的作用是取消监听一个远程端口，释放所有处于等待连接的资源，并返回 0，如果在取消过程中出现错误，则返回 LIBSSH2_ERROR_EAGAIN。

具体来说，这段代码执行以下操作：

1. 检查传入的 `listener` 是否为 `NULL`，如果是，直接返回 LIBSSH2_ERROR_BAD_USE，表示函数无法执行。
2. 对传入的 `listener` 和 `session` 进行 BlockingAdjust 操作，使得函数可以暂停、唤醒或者唤醒。
3. 使用 `_libssh2_channel_forward_cancel` 函数取消监听远程端口，并得到一个内部函数返回的的状态码。
4. 如果取消成功，即返回 0，否则，返回 LIBSSH2_ERROR_EAGAIN，表示函数执行失败。


```cpp
/*
 * libssh2_channel_forward_cancel
 *
 * Stop listening on a remote port and free the listener
 * Toss out any pending (un-accept()ed) connections
 *
 * Return 0 on success, LIBSSH2_ERROR_EAGAIN if would block, -1 on error
 */
LIBSSH2_API int
libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener)
{
    int rc;

    if(!listener)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, listener->session,
                 _libssh2_channel_forward_cancel(listener));
    return rc;
}

```

这段代码定义了一个名为 `channel_forward_accept` 的函数，它是 LIBSSH2 库中的一个函数，用于接受连接。

这个函数接收一个 LIBSSH2 监听器（SSH2 客户端），使用这个监听器来创建一个 channel。通过调用 `_libssh2_transport_read` 函数，监听器可以读取 channel 的数据。然后，通过一系列循环，这个函数接收了一个 channel，并将其从监听器的队列中删除，然后将 channel 添加到 session 的 channel 列表中，最后返回 channel。

如果遇到任何错误，函数将返回 LIBSSH2 库的错误信息。


```cpp
/*
 * channel_forward_accept
 *
 * Accept a connection
 */
static LIBSSH2_CHANNEL *
channel_forward_accept(LIBSSH2_LISTENER *listener)
{
    int rc;

    do {
        rc = _libssh2_transport_read(listener->session);
    } while(rc > 0);

    if(_libssh2_list_first(&listener->queue)) {
        LIBSSH2_CHANNEL *channel = _libssh2_list_first(&listener->queue);

        /* detach channel from listener's queue */
        _libssh2_list_remove(&channel->node);

        listener->queue_size--;

        /* add channel to session's channel list */
        _libssh2_list_add(&channel->session->channels, &channel->node);

        return channel;
    }

    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(listener->session, LIBSSH2_ERROR_EAGAIN,
                       "Would block waiting for packet");
    }
    else
        _libssh2_error(listener->session, LIBSSH2_ERROR_CHANNEL_UNKNOWN,
                       "Channel not found");
    return NULL;
}

```

这段代码定义了一个名为`libssh2_channel_forward_accept`的函数，属于`libssh2_channel`类型。

这个函数的参数是一个`LIBSSH2_LISTENER`类型的指针，表示一个`LIBSSH2_CHANNEL`结构体。函数返回一个指向`LIBSSH2_CHANNEL`类型的指针，用于接受连接。

函数首先检查`listener`是否为`NULL`，如果是，则直接返回`NULL`。否则，函数调用`channel_forward_accept`函数，将`listener`作为参数传递，并将返回结果存储在`ptr`指向的`LIBSSH2_CHANNEL`结构体中。

最后，函数返回`ptr`，表示接受连接的`LIBSSH2_CHANNEL`结构体。


```cpp
/*
 * libssh2_channel_forward_accept
 *
 * Accept a connection
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_forward_accept(LIBSSH2_LISTENER *listener)
{
    LIBSSH2_CHANNEL *ptr;

    if(!listener)
        return NULL;

    BLOCK_ADJUST_ERRNO(ptr, listener->session,
                       channel_forward_accept(listener));
    return ptr;

}

```

This function appears to handle the process of sending a `setenv request` packet


```cpp
/*
 * channel_setenv
 *
 * Set an environment variable prior to requesting a shell/program/subsystem
 */
static int channel_setenv(LIBSSH2_CHANNEL *channel,
                          const char *varname, unsigned int varname_len,
                          const char *value, unsigned int value_len)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s, *data;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    size_t data_len;
    int rc;

    if(channel->setenv_state == libssh2_NB_state_idle) {
        /* 21 = packet_type(1) + channel_id(4) + request_len(4) +
         * request(3)"env" + want_reply(1) + varname_len(4) + value_len(4) */
        channel->setenv_packet_len = varname_len + value_len + 21;

        /* Zero the whole thing out */
        memset(&channel->setenv_packet_requirev_state, 0,
               sizeof(channel->setenv_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Setting remote environment variable: %s=%s on "
                       "channel %lu/%lu",
                       varname, value, channel->local.id, channel->remote.id);

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

    if(channel->setenv_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session,
                                     channel->setenv_packet,
                                     channel->setenv_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending setenv request");
            return rc;
        }
        else if(rc) {
            LIBSSH2_FREE(session, channel->setenv_packet);
            channel->setenv_packet = NULL;
            channel->setenv_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send channel-request packet for "
                                  "setenv request");
        }
        LIBSSH2_FREE(session, channel->setenv_packet);
        channel->setenv_packet = NULL;

        _libssh2_htonu32(channel->setenv_local_channel, channel->local.id);

        channel->setenv_state = libssh2_NB_state_sent;
    }

    if(channel->setenv_state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->setenv_local_channel, 4,
                                      &channel->
                                      setenv_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        if(rc) {
            channel->setenv_state = libssh2_NB_state_idle;
            return rc;
        }
        else if(data_len < 1) {
            channel->setenv_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Unexpected packet size");
        }

        if(data[0] == SSH_MSG_CHANNEL_SUCCESS) {
            LIBSSH2_FREE(session, data);
            channel->setenv_state = libssh2_NB_state_idle;
            return 0;
        }

        LIBSSH2_FREE(session, data);
    }

    channel->setenv_state = libssh2_NB_state_idle;
    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for channel-setenv");
}

```

这段代码定义了一个名为 `libssh2_channel_setenv_ex` 的函数，属于 `libssh2_channel` 类的成员函数。

它的作用是在向远程主机发送 SSH 连接请求之前设置一个环境变量。可以为设置的环境变量指定一个名称（例如，设置名为 "MY环境变量"），也可以指定这个变量的有效载荷长度。

函数接受两个参数：一个 `LIBSSH2_CHANNEL` 类型的通道对象，一个指向 `const char` 类型变量的指针，以及一个 `const char` 类型的变量名称和它的有效载荷长度。

函数首先检查传递的 `channel` 参数是否为 `NULL`，如果是，函数返回 `LIBSSH2_ERROR_BAD_USE`，通常这是一个表示错误条件的前缀。

然后，函数通过调用 `channel_setenv` 函数，将指定的环境变量设置到 `channel` 对象的 `session` 内部。

最后，函数返回 `rc`，表示是否成功设置环境变量。


```cpp
/*
 * libssh2_channel_setenv_ex
 *
 * Set an environment variable prior to requesting a shell/program/subsystem
 */
LIBSSH2_API int
libssh2_channel_setenv_ex(LIBSSH2_CHANNEL *channel,
                          const char *varname, unsigned int varname_len,
                          const char *value, unsigned int value_len)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 channel_setenv(channel, varname, varname_len,
                                value, value_len));
    return rc;
}

```

This is a function in the OpenSSH library that handles the sending of a PTY request packet. The function takes a session object, a return code, and a reason code as


```cpp
/*
 * channel_request_pty
 * Duh... Request a PTY
 */
static int channel_request_pty(LIBSSH2_CHANNEL *channel,
                               const char *term, unsigned int term_len,
                               const char *modes, unsigned int modes_len,
                               int width, int height,
                               int width_px, int height_px)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    int rc;

    if(channel->reqPTY_state == libssh2_NB_state_idle) {
        /* 41 = packet_type(1) + channel(4) + pty_req_len(4) + "pty_req"(7) +
         * want_reply(1) + term_len(4) + width(4) + height(4) + width_px(4) +
         * height_px(4) + modes_len(4) */
        if(term_len + modes_len > 256) {
            return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                                  "term + mode lengths too large");
        }

        channel->reqPTY_packet_len = term_len + modes_len + 41;

        /* Zero the whole thing out */
        memset(&channel->reqPTY_packet_requirev_state, 0,
               sizeof(channel->reqPTY_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Allocating tty on channel %lu/%lu", channel->local.id,
                       channel->remote.id);

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

        channel->reqPTY_state = libssh2_NB_state_created;
    }

    if(channel->reqPTY_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->reqPTY_packet,
                                     channel->reqPTY_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending pty request");
            return rc;
        }
        else if(rc) {
            channel->reqPTY_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send pty-request packet");
        }
        _libssh2_htonu32(channel->reqPTY_local_channel, channel->local.id);

        channel->reqPTY_state = libssh2_NB_state_sent;
    }

    if(channel->reqPTY_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        unsigned char code;
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->reqPTY_local_channel, 4,
                                      &channel->reqPTY_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc || data_len < 1) {
            channel->reqPTY_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Failed to require the PTY package");
        }

        code = data[0];

        LIBSSH2_FREE(session, data);
        channel->reqPTY_state = libssh2_NB_state_idle;

        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for "
                          "channel request-pty");
}

```

This is a function in the SSH client library that handles the sending of an auth-agent request by an agent. The function takes a session object, a return code, and a data buffer. It first checks if the session object has a valid auth-agent state, and if not, it sets it to idle. Then it sends the data buffer to the auth-agent using the libssh2_packet_send function, and if the return code is 0, it returns. If the return code is any other non-zero value, it handles it.


```cpp
/**
 * channel_request_auth_agent
 * The actual re-entrant method which requests an auth agent.
 * */
static int channel_request_auth_agent(LIBSSH2_CHANNEL *channel,
                                      const char *request_str,
                                      int request_str_len)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    int rc;

    if(channel->req_auth_agent_state == libssh2_NB_state_idle) {
        /* Only valid options are "auth-agent-req" and
         * "auth-agent-req_at_openssh.com" so we make sure it is not
         * actually longer than the longest possible. */
        if(request_str_len > 26) {
            return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                                  "request_str length too large");
        }

        /*
         *  Length: 24 or 36 = packet_type(1) + channel(4) + req_len(4) +
         *    request_str (variable) + want_reply (1) */
        channel->req_auth_agent_packet_len = 10 + request_str_len;

        /* Zero out the requireev state to reset */
        memset(&channel->req_auth_agent_requirev_state, 0,
               sizeof(channel->req_auth_agent_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting auth agent on channel %lu/%lu",
                       channel->local.id, channel->remote.id);

        /*
         *  byte      SSH_MSG_CHANNEL_REQUEST
         *  uint32    recipient channel
         *  string    "auth-agent-req"
         *  boolean   want reply
         * */
        s = channel->req_auth_agent_packet;
        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, (char *)request_str, request_str_len);
        *(s++) = 0x01;

        channel->req_auth_agent_state = libssh2_NB_state_created;
    }

    if(channel->req_auth_agent_state == libssh2_NB_state_created) {
        /* Send the packet, we can use sizeof() on the packet because it
         * is always completely filled; there are no variable length fields. */
        rc = _libssh2_transport_send(session, channel->req_auth_agent_packet,
                                     channel->req_auth_agent_packet_len,
                                     NULL, 0);

        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending auth-agent request");
        }
        else if(rc) {
            channel->req_auth_agent_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send auth-agent request");
        }
        _libssh2_htonu32(channel->req_auth_agent_local_channel,
                         channel->local.id);
        channel->req_auth_agent_state = libssh2_NB_state_sent;
    }

    if(channel->req_auth_agent_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        unsigned char code;

        rc = _libssh2_packet_requirev(
            session, reply_codes, &data, &data_len, 1,
            channel->req_auth_agent_local_channel,
            4, &channel->req_auth_agent_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            channel->req_auth_agent_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Failed to request auth-agent");
        }

        code = data[0];

        LIBSSH2_FREE(session, data);
        channel->req_auth_agent_state = libssh2_NB_state_idle;

        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for auth-agent");
}

```



This function appears to be a part of the SSH2 protocol, specifically the authentication agent feature. It is used to allow an SSH client to send authentication requests to an SSH server using an authentication agent specified by the client.

The function takes a single parameter, a `LIBSSH2_CHANNEL` object representing the channel to use for the authentication request. The function returns an integer representing the result of the authentication request.

The function has several helper functions:

- `channel_request_auth_agent`: This function is used to send the authentication agent request to the server specified by the `auth_agent_location` parameter in an SSH client's `新参文` (new request) message. It takes two arguments: the `auth_agent_location` parameter and a maximum number of seconds to wait before sending the request. This function returns an integer indicating whether the request was successfully sent or not.
- `libssh2_error_eagain`: This function is used to return an error code if the authentication agent could not resend the authentication request due to an EAGAIN error. It takes a single argument, the error code returned by `libssh2_error_eagain`.
- `libssh2_nb_state_idle`, `libssh2_nb_state_sent1`: These functions are used to represent the current state of the authentication agent request in the SSH2 protocol. They are returned by `channel_request_auth_agent` in the function definition.


```cpp
/**
 * libssh2_channel_request_auth_agent
 * Requests that agent forwarding be enabled for the session. The
 * request must be sent over a specific channel, which starts the agent
 * listener on the remote side. Once the channel is closed, the agent
 * listener continues to exist.
 * */
LIBSSH2_API int
libssh2_channel_request_auth_agent(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    /* The current RFC draft for agent forwarding says you're supposed to
     * send "auth-agent-req," but most SSH servers out there right now
     * actually expect "auth-agent-req@openssh.com", so we try that
     * first. */
    if(channel->req_auth_agent_try_state == libssh2_NB_state_idle) {
        BLOCK_ADJUST(rc, channel->session,
                     channel_request_auth_agent(channel,
                                                "auth-agent-req@openssh.com",
                                                26));

        /* If we failed (but not with EAGAIN), then we move onto
         * the next step to try another request type. */
        if(rc != 0 && rc != LIBSSH2_ERROR_EAGAIN)
            channel->req_auth_agent_try_state = libssh2_NB_state_sent;
    }

    if(channel->req_auth_agent_try_state == libssh2_NB_state_sent) {
        BLOCK_ADJUST(rc, channel->session,
                     channel_request_auth_agent(channel,
                                                "auth-agent-req", 14));

        /* If we failed without an EAGAIN, then move on with this
         * state machine. */
        if(rc != 0 && rc != LIBSSH2_ERROR_EAGAIN)
            channel->req_auth_agent_try_state = libssh2_NB_state_sent1;
    }

    /* If things are good, reset the try state. */
    if(rc == 0)
        channel->req_auth_agent_try_state = libssh2_NB_state_idle;

    return rc;
}

```

这段代码是一个名为 `libssh2_channel_request_pty_ex` 的函数，属于 SSH2 协议的 Channel 层。它的作用是请求通过 PTY（Parallel Terminal）连接的主机允许在客户端之间传输数据。

具体来说，函数接收通道句柄（channel）、允许使用的 PTY 模式（modes）以及 PTY 允许使用的最大宽度（width）和最大高度（height）。函数首先检查传递的参数是否为有效的 channel 句柄，然后执行通道请求 PTY 允许的数据传输。

函数实现中，首先定义了一些局部变量，然后执行以下操作：

1. 将允许的最大宽度（width）和最大高度（height）存储在 libssh2_channel_request_pty_ex 函数内部，以便在请求失败时回滚到默认值。
2. 通过调用通道内置函数 channel_request_pty，传递允许使用的 PTY 模式和最大宽度，获取 PTY 允许使用的最大宽度（width_px）和最大高度（height_px）。
3. 如果上述操作成功，就返回 0，否则返回 LIBSSH2_ERROR_BAD_USE 错误代码。


```cpp
/*
 * libssh2_channel_request_pty_ex
 * Duh... Request a PTY
 */
LIBSSH2_API int
libssh2_channel_request_pty_ex(LIBSSH2_CHANNEL *channel, const char *term,
                               unsigned int term_len, const char *modes,
                               unsigned int modes_len, int width, int height,
                               int width_px, int height_px)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 channel_request_pty(channel, term, term_len, modes,
                                     modes_len, width, height,
                                     width_px, height_px));
    return rc;
}

```

This is a function for handling a window-change request from a client to a server over SSH. The function takes a `channel` object, which represents the SSH channel, and handles the packet received from the client.

The function first converts the packet to escape the encoding and extract the `SSH_MSG_CHANNEL_REQUEST` message. Then it sets the `remote_id` field of the packet to the local channel ID and appends the window-change payload to the packet.

The function then sends the packet over the SSH channel using the `_libssh2_transport_send` function. If the initial send fails, the function returns an error. If the send completes successfully, the function sets the `reqPTY_state` field of the channel to `libssh2_NB_state_idle` and returns `LIBSSH2_ERROR_NONE`.

Finally, the function converts the local channel ID to a unigned 32-bit integer and returns it.


```cpp
static int
channel_request_pty_size(LIBSSH2_CHANNEL * channel, int width,
                         int height, int width_px, int height_px)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s;
    int rc;
    int retcode = LIBSSH2_ERROR_PROTO;

    if(channel->reqPTY_state == libssh2_NB_state_idle) {
        channel->reqPTY_packet_len = 39;

        /* Zero the whole thing out */
        memset(&channel->reqPTY_packet_requirev_state, 0,
               sizeof(channel->reqPTY_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
            "changing tty size on channel %lu/%lu",
            channel->local.id,
            channel->remote.id);

        s = channel->reqPTY_packet;

        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, (char *)"window-change",
                           sizeof("window-change") - 1);
        *(s++) = 0x00; /* Don't reply */
        _libssh2_store_u32(&s, width);
        _libssh2_store_u32(&s, height);
        _libssh2_store_u32(&s, width_px);
        _libssh2_store_u32(&s, height_px);

        channel->reqPTY_state = libssh2_NB_state_created;
    }

    if(channel->reqPTY_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->reqPTY_packet,
                                     channel->reqPTY_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending window-change request");
            return rc;
        }
        else if(rc) {
            channel->reqPTY_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send window-change packet");
        }
        _libssh2_htonu32(channel->reqPTY_local_channel, channel->local.id);
        retcode = LIBSSH2_ERROR_NONE;
    }

    channel->reqPTY_state = libssh2_NB_state_idle;
    return retcode;
}

```

这段代码是一个名为`libssh2_channel_request_pty_size_ex`的函数，属于名为`libssh2_channel`的库。它的作用是设置一个SSH通道的宽度和高度，以及将其转换为像素宽度。以下是具体的实现步骤：

1. 首先检查传递给该函数的参数`channel`是否为空，如果是，则返回`LIBSSH2_ERROR_BAD_USE`错误。
2. 如果`channel`不为空，则执行以下操作：
a. 创建一个名为`rc`的整数变量。
b. 通过调用名为`channel_request_pty_size`的函数，将`channel`和`width`（即通道的宽度）作为参数传递给该函数。该函数的第一个参数是`channel`，第二个参数是`width`，第三个参数是`height`，第四个参数是`width_px`（即宽度），第五个参数是`height_px`（即高度）。该函数返回一个整数，代表上一步操作的结果，即`rc`的值。
c. 将`rc`的值作为函数返回，如果没有错误发生。
3. 如果存在错误，函数将返回`LIBSSH2_ERROR_BAD_USE`。


```cpp
LIBSSH2_API int
libssh2_channel_request_pty_size_ex(LIBSSH2_CHANNEL *channel, int width,
                                    int height, int width_px, int height_px)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 channel_request_pty_size(channel, width, height, width_px,
                                          height_px));
    return rc;
}

```

This function appears to be responsible for sending an X11 request packet to the remote host using the SSH protocol. It is part of a larger feature that allows users to run graphical applications remotely using X11.

The function takes in a connection object (`session`), a `channel` object, and a `local_channel` integer. It first sets the state of the `channel` object to `libssh2_NB_state_idle`, and then sets the `reqX11_state` property of the `channel` object to `libssh2_NB_state_sent`.

It then calls the `_libssh2_packet_requirev` function to send the packet, passing in the `reply_codes` parameter to specify that it should be a reply to a `channel_request_t` object. If the function returns an error, it returns an error with a specific code, such as `LIBSSH2_ERROR_EAGAIN`.

If the function succeeds in sending the packet, it checks the return code of the `_libssh2_packet_requirev` function to see if the packet was successfully sent. If the packet was successfully sent, the function returns 0. If not, it returns an error with a specific error code.


```cpp
/* Keep this an even number */
#define LIBSSH2_X11_RANDOM_COOKIE_LEN       32

/*
 * channel_x11_req
 * Request X11 forwarding
 */
static int
channel_x11_req(LIBSSH2_CHANNEL *channel, int single_connection,
                const char *auth_proto, const char *auth_cookie,
                int screen_number)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *s;
    static const unsigned char reply_codes[3] =
        { SSH_MSG_CHANNEL_SUCCESS, SSH_MSG_CHANNEL_FAILURE, 0 };
    size_t proto_len =
        auth_proto ? strlen(auth_proto) : (sizeof("MIT-MAGIC-COOKIE-1") - 1);
    size_t cookie_len =
        auth_cookie ? strlen(auth_cookie) : LIBSSH2_X11_RANDOM_COOKIE_LEN;
    int rc;

    if(channel->reqX11_state == libssh2_NB_state_idle) {
        /* 30 = packet_type(1) + channel(4) + x11_req_len(4) + "x11-req"(7) +
         * want_reply(1) + single_cnx(1) + proto_len(4) + cookie_len(4) +
         * screen_num(4) */
        channel->reqX11_packet_len = proto_len + cookie_len + 30;

        /* Zero the whole thing out */
        memset(&channel->reqX11_packet_requirev_state, 0,
               sizeof(channel->reqX11_packet_requirev_state));

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Requesting x11-req for channel %lu/%lu: single=%d "
                       "proto=%s cookie=%s screen=%d",
                       channel->local.id, channel->remote.id,
                       single_connection,
                       auth_proto ? auth_proto : "MIT-MAGIC-COOKIE-1",
                       auth_cookie ? auth_cookie : "<random>", screen_number);

        s = channel->reqX11_packet =
            LIBSSH2_ALLOC(session, channel->reqX11_packet_len);
        if(!channel->reqX11_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for pty-request");
        }

        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, "x11-req", sizeof("x11-req") - 1);

        *(s++) = 0x01;          /* want_reply */
        *(s++) = single_connection ? 0x01 : 0x00;

        _libssh2_store_str(&s, auth_proto ? auth_proto : "MIT-MAGIC-COOKIE-1",
                           proto_len);

        _libssh2_store_u32(&s, cookie_len);
        if(auth_cookie) {
            memcpy(s, auth_cookie, cookie_len);
        }
        else {
            int i;
            /* note: the extra +1 below is necessary since the sprintf()
               loop will always write 3 bytes so the last one will write
               the trailing zero at the LIBSSH2_X11_RANDOM_COOKIE_LEN/2
               border */
            unsigned char buffer[(LIBSSH2_X11_RANDOM_COOKIE_LEN / 2) + 1];

            if(_libssh2_random(buffer, LIBSSH2_X11_RANDOM_COOKIE_LEN / 2)) {
                return _libssh2_error(session, LIBSSH2_ERROR_RANDGEN,
                                      "Unable to get random bytes "
                                      "for x11-req cookie");
            }
            for(i = 0; i < (LIBSSH2_X11_RANDOM_COOKIE_LEN / 2); i++) {
                snprintf((char *)&s[i*2], 3, "%02X", buffer[i]);
            }
        }
        s += cookie_len;

        _libssh2_store_u32(&s, screen_number);
        channel->reqX11_state = libssh2_NB_state_created;
    }

    if(channel->reqX11_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->reqX11_packet,
                                     channel->reqX11_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending X11-req packet");
            return rc;
        }
        if(rc) {
            LIBSSH2_FREE(session, channel->reqX11_packet);
            channel->reqX11_packet = NULL;
            channel->reqX11_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send x11-req packet");
        }
        LIBSSH2_FREE(session, channel->reqX11_packet);
        channel->reqX11_packet = NULL;

        _libssh2_htonu32(channel->reqX11_local_channel, channel->local.id);

        channel->reqX11_state = libssh2_NB_state_sent;
    }

    if(channel->reqX11_state == libssh2_NB_state_sent) {
        size_t data_len;
        unsigned char *data;
        unsigned char code;

        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->reqX11_local_channel, 4,
                                      &channel->reqX11_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc || data_len < 1) {
            channel->reqX11_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "waiting for x11-req response packet");
        }

        code = data[0];
        LIBSSH2_FREE(session, data);
        channel->reqX11_state = libssh2_NB_state_idle;

        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for channel x11-req");
}

```

这段代码是一个名为`libssh2_channel_x11_req_ex`的函数，属于`libssh2_channel`类的实现。它的作用是请求 X11 转发，并在函数内部对`channel`参数所指向的通道进行封装。

具体来说，这段代码实现了一个 block 函数，其中`rc`变量用于保存封装结果。`channel_x11_req`函数用于原始的 X11 请求，将用户提供的认证协议、认证 cookie 和屏幕号作为参数传入。如果传入了无效的参数，函数会返回 LIBSSH2_ERROR_BAD_USE。

通过封装后的 block 函数，可以将原始的 X11 请求传递给底层系统，从而实现 X11 转发。在函数实现中，通过 `BLOCK_ADJUST` 函数来调整 block 的大小，从而影响底层系统对 X11 请求的执行效率。


```cpp
/*
 * libssh2_channel_x11_req_ex
 * Request X11 forwarding
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


```

This is a function in the SSH2 library that handles the sending of a channel request to another process. The function takes in a session object, a channel object, and a buffer representing the data to be sent as an argument.

It first checks if the channel is already in an established state, and if not, it calls the `libssh2_channel_connect` function to establish a connection and create a channel. Then it checks the channel's request status and if it's successful, it sets the channel's state to sent and returns 0. If the request is denied, it returns LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED.


```cpp
/*
 * _libssh2_channel_process_startup
 *
 * Primitive for libssh2_channel_(shell|exec|subsystem)
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

    if(channel->process_state == libssh2_NB_state_idle) {
        /* 10 = packet_type(1) + channel(4) + request_len(4) + want_reply(1) */
        channel->process_packet_len = request_len + 10;

        /* Zero the whole thing out */
        memset(&channel->process_packet_requirev_state, 0,
               sizeof(channel->process_packet_requirev_state));

        if(message)
            channel->process_packet_len += + 4;

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "starting request(%s) on channel %lu/%lu, message=%s",
                       request, channel->local.id, channel->remote.id,
                       message ? message : "<null>");
        s = channel->process_packet =
            LIBSSH2_ALLOC(session, channel->process_packet_len);
        if(!channel->process_packet)
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory "
                                  "for channel-process request");

        *(s++) = SSH_MSG_CHANNEL_REQUEST;
        _libssh2_store_u32(&s, channel->remote.id);
        _libssh2_store_str(&s, request, request_len);
        *(s++) = 0x01;

        if(message)
            _libssh2_store_u32(&s, message_len);

        channel->process_state = libssh2_NB_state_created;
    }

    if(channel->process_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session,
                                     channel->process_packet,
                                     channel->process_packet_len,
                                     (unsigned char *)message, message_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending channel request");
            return rc;
        }
        else if(rc) {
            LIBSSH2_FREE(session, channel->process_packet);
            channel->process_packet = NULL;
            channel->process_state = libssh2_NB_state_end;
            return _libssh2_error(session, rc,
                                  "Unable to send channel request");
        }
        LIBSSH2_FREE(session, channel->process_packet);
        channel->process_packet = NULL;

        _libssh2_htonu32(channel->process_local_channel, channel->local.id);

        channel->process_state = libssh2_NB_state_sent;
    }

    if(channel->process_state == libssh2_NB_state_sent) {
        unsigned char *data;
        size_t data_len;
        unsigned char code;
        rc = _libssh2_packet_requirev(session, reply_codes, &data, &data_len,
                                      1, channel->process_local_channel, 4,
                                      &channel->process_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc || data_len < 1) {
            channel->process_state = libssh2_NB_state_end;
            return _libssh2_error(session, rc,
                                  "Failed waiting for channel success");
        }

        code = data[0];
        LIBSSH2_FREE(session, data);
        channel->process_state = libssh2_NB_state_end;

        if(code == SSH_MSG_CHANNEL_SUCCESS)
            return 0;
    }

    return _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED,
                          "Unable to complete request for "
                          "channel-process-startup");
}

```

这段代码定义了一个名为 `libssh2_channel_process_startup` 的函数，它是 `libssh2_channel_` 类的原始成员函数。

这个函数的作用是处理 SSH2 通道的启动过程，包括通道的创建、设置和关闭等。函数接受三个参数：一个 `LIBSSH2_CHANNEL` 类型的通道对象，一个字符串 `req`，它是启动请求的参数，以及一个字符数组 `msg` 和一个字符数组 `msg_len`，它们分别是通道返回的响应和错误信息的长度。

函数内部首先检查传入的 `channel` 是否为 `NULL`，如果是，则直接返回 `LIBSSH2_ERROR_BAD_USE`，表示函数无法处理。然后进行了一系列辅助函数，包括 `_libssh2_channel_process_startup` 函数，这个函数会处理 `channel` 对象和启动请求的参数。最后，函数返回结果，成功返回 `0`，失败返回 `LIBSSH2_ERROR_BAD_USE`。


```cpp
/*
 * libssh2_channel_process_startup
 *
 * Primitive for libssh2_channel_(shell|exec|subsystem)
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


```

这段代码定义了一个名为`libssh2_channel_set_blocking`的函数，它属于`libssh2_client_session`库。

这个函数的主要作用是设置一个通道（channel）的行为为阻塞（BEHAVIOR_BLOCKING或BEHAVIOR_LISTENING）。

具体来说，如果通道已经存在，函数将调用`_libssh2_session_set_blocking`函数，传递给`channel->session`参数，从而设置整个会话的阻塞行为。

如果调用这个函数，但没有传递任何有效的`channel`参数，那么函数的行为将是不确定的，可能会导致程序崩溃。


```cpp
/*
 * libssh2_channel_set_blocking
 *
 * Set a channel's BEHAVIOR blocking on or off. The socket will remain non-
 * blocking.
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
 * Flush data from one (or all) stream
 * Returns number of bytes flushed, or negative on failure
 */
```



This function appears to handle the flushing of data from a stream on a remote server to a local client. It takes in a packet stream id and a packet object, and如果是 SSH通道 data, it adds a new packet to the stream.

It works as follows:

1. If the packet stream id matches one of the streams, it adds the packet to the stream and extracts any metadata.

2. If the packet is SSH channel data, it adds the packet to the stream and increments the refund bytes.

3. It calls the function `_libssh2_channel_receive_window_adjust` for the remote channel to adjust the window size based on the refund bytes.

4. It sets the state of the channel to the next state after the event.

5. It returns the number of bytes that were flushed.


```cpp
int
_libssh2_channel_flush(LIBSSH2_CHANNEL *channel, int streamid)
{
    if(channel->flush_state == libssh2_NB_state_idle) {
        LIBSSH2_PACKET *packet =
            _libssh2_list_first(&channel->session->packets);
        channel->flush_refund_bytes = 0;
        channel->flush_flush_bytes = 0;

        while(packet) {
            unsigned char packet_type;
            LIBSSH2_PACKET *next = _libssh2_list_next(&packet->node);

            if(packet->data_len < 1) {
                packet = next;
                _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                               "Unexpected packet length");
                continue;
            }

            packet_type = packet->data[0];

            if(((packet_type == SSH_MSG_CHANNEL_DATA)
                || (packet_type == SSH_MSG_CHANNEL_EXTENDED_DATA))
               && ((packet->data_len >= 5)
                   && (_libssh2_ntohu32(packet->data + 1)
                       == channel->local.id))) {
                /* It's our channel at least */
                int packet_stream_id;

                if(packet_type == SSH_MSG_CHANNEL_DATA) {
                    packet_stream_id = 0;
                }
                else if(packet->data_len >= 9) {
                    packet_stream_id = _libssh2_ntohu32(packet->data + 5);
                }
                else {
                    channel->flush_state = libssh2_NB_state_idle;
                    return _libssh2_error(channel->session,
                                          LIBSSH2_ERROR_PROTO,
                                          "Unexpected packet length");
                }

                if((streamid == LIBSSH2_CHANNEL_FLUSH_ALL)
                    || ((packet_type == SSH_MSG_CHANNEL_EXTENDED_DATA)
                        && ((streamid == LIBSSH2_CHANNEL_FLUSH_EXTENDED_DATA)
                            || (streamid == packet_stream_id)))
                    || ((packet_type == SSH_MSG_CHANNEL_DATA)
                        && (streamid == 0))) {
                    size_t bytes_to_flush = packet->data_len -
                        packet->data_head;

                    _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                                   "Flushing %d bytes of data from stream "
                                   "%lu on channel %lu/%lu",
                                   bytes_to_flush, packet_stream_id,
                                   channel->local.id, channel->remote.id);

                    /* It's one of the streams we wanted to flush */
                    channel->flush_refund_bytes += packet->data_len - 13;
                    channel->flush_flush_bytes += bytes_to_flush;

                    LIBSSH2_FREE(channel->session, packet->data);

                    /* remove this packet from the parent's list */
                    _libssh2_list_remove(&packet->node);
                    LIBSSH2_FREE(channel->session, packet);
                }
            }
            packet = next;
        }

        channel->flush_state = libssh2_NB_state_created;
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

```

这段代码是一个名为 `libssh2_channel_flush_ex` 的函数，它是 `libssh2_channel` 类的实现。它的作用是将从一个（或所有）流中清除数据。它返回一个整数表示已flush的数据字节数，或者在失败时返回一个负数。

函数的实现包括以下几个步骤：

1. 检查输入参数 `channel` 是否为 `NULL`，如果是，则返回 `LIBSSH2_ERROR_BAD_USE`。
2. 执行内部函数 `_libssh2_channel_flush`，传递 `channel` 和 `stream` 参数。
3. 将结果存储在 `rc` 变量中。
4. 返回 `rc`。

该函数在传输过程中，确保所有连续写入的数据都被立即写入，从而保证数据不会丢失。


```cpp
/*
 * libssh2_channel_flush_ex
 *
 * Flush data from one (or all) stream
 * Returns number of bytes flushed, or negative on failure
 */
LIBSSH2_API int
libssh2_channel_flush_ex(LIBSSH2_CHANNEL *channel, int stream)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_flush(channel, stream));
    return rc;
}

```

这段代码定义了一个名为`libssh2_channel_get_exit_status`的函数，它接受一个名为`channel`的`LIBSSH2_CHANNEL`类型的参数。

函数首先检查`channel`是否为空，如果是，则返回0。否则，它返回通道的`exit_status`成员。注意，函数实际返回的是一个32位整数，而不是通常的`int`类型。

通道的`exit_status`成员是一个表示通道关闭状态的整数，它可能是在一个SSH会话中发生的各种错误代码之一。


```cpp
/*
 * libssh2_channel_get_exit_status
 *
 * Return the channel's program exit status. Note that the actual protocol
 * provides the full 32bit this function returns.  We cannot abuse it to
 * return error values in case of errors so we return a zero if channel is
 * NULL.
 */
LIBSSH2_API int
libssh2_channel_get_exit_status(LIBSSH2_CHANNEL *channel)
{
    if(!channel)
        return 0;

    return channel->exit_status;
}

```

0 21415011859
You're looking for the output of the `libssh2_channel_get_exit_signal` function, which is a part of the SSH2 protocol.

The output should contain the return value (0 or an error code), as well as any additional information that can be used to diagnose an error.

The output may include information about the exit signal, such as its name and the language tag used for the signal. It may also include information about any errors that occurred during the transition to the exit signal, such as any memory deallocations or failure to set the exit signal.

If there was an error in the function, it may be included in the output with a return code and additional information about the error.


```cpp
/*
 * libssh2_channel_get_exit_signal
 *
 * Get exit signal (without leading "SIG"), error message, and language
 * tag into newly allocated buffers of indicated length.  Caller can
 * use NULL pointers to indicate that the value should not be set.  The
 * *_len variables are set if they are non-NULL even if the
 * corresponding string parameter is NULL.  Returns LIBSSH2_ERROR_NONE
 * on success, or an API error code.
 */
LIBSSH2_API int
libssh2_channel_get_exit_signal(LIBSSH2_CHANNEL *channel,
                                char **exitsignal,
                                size_t *exitsignal_len,
                                char **errmsg,
                                size_t *errmsg_len,
                                char **langtag,
                                size_t *langtag_len)
{
    size_t namelen = 0;

    if(channel) {
        LIBSSH2_SESSION *session = channel->session;

        if(channel->exit_signal) {
            namelen = strlen(channel->exit_signal);
            if(exitsignal) {
                *exitsignal = LIBSSH2_ALLOC(session, namelen + 1);
                if(!*exitsignal) {
                    return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                        "Unable to allocate memory for signal name");
                }
                memcpy(*exitsignal, channel->exit_signal, namelen);
                (*exitsignal)[namelen] = '\0';
            }
            if(exitsignal_len)
                *exitsignal_len = namelen;
        }
        else {
            if(exitsignal)
                *exitsignal = NULL;
            if(exitsignal_len)
                *exitsignal_len = 0;
        }

        /* TODO: set error message and language tag */

        if(errmsg)
            *errmsg = NULL;

        if(errmsg_len)
            *errmsg_len = 0;

        if(langtag)
            *langtag = NULL;

        if(langtag_len)
            *langtag_len = 0;
    }

    return LIBSSH2_ERROR_NONE;
}

```

这段代码的作用是调整阻塞窗口，以便可以发送更多的数据。具体来说，它会根据当前调整量和阻塞窗口大小来计算下一个窗口调整量，然后发送给远程服务器。如果成功发送，则返回 0，否则继续阻塞发送窗口调整。

这里有一些潜在的问题需要注意：

1.代码没有进行完整的测试，因此在实际使用中需要进行更多的测试和验证。

2. 这段代码已经非常冗长和复杂，因此需要更多的代码和注释来提高可读性和可维护性。

3. 如果使用的是异步IO，可能需要添加更多的错误处理来处理非-blocking操作的情况。


```cpp
/*
 * _libssh2_channel_receive_window_adjust
 *
 * Adjust the receive window for a channel by adjustment bytes. If the amount
 * to be adjusted is less than LIBSSH2_CHANNEL_MINADJUST and force is 0 the
 * adjustment amount will be queued for a later packet.
 *
 * Calls _libssh2_error() !
 */
int
_libssh2_channel_receive_window_adjust(LIBSSH2_CHANNEL * channel,
                                       uint32_t adjustment,
                                       unsigned char force,
                                       unsigned int *store)
{
    int rc;

    if(store)
        *store = channel->remote.window_size;

    if(channel->adjust_state == libssh2_NB_state_idle) {
        if(!force
            && (adjustment + channel->adjust_queue <
                LIBSSH2_CHANNEL_MINADJUST)) {
            _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                           "Queueing %lu bytes for receive window adjustment "
                           "for channel %lu/%lu",
                           adjustment, channel->local.id, channel->remote.id);
            channel->adjust_queue += adjustment;
            return 0;
        }

        if(!adjustment && !channel->adjust_queue) {
            return 0;
        }

        adjustment += channel->adjust_queue;
        channel->adjust_queue = 0;

        /* Adjust the window based on the block we just freed */
        channel->adjust_adjust[0] = SSH_MSG_CHANNEL_WINDOW_ADJUST;
        _libssh2_htonu32(&channel->adjust_adjust[1], channel->remote.id);
        _libssh2_htonu32(&channel->adjust_adjust[5], adjustment);
        _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                       "Adjusting window %lu bytes for data on "
                       "channel %lu/%lu",
                       adjustment, channel->local.id, channel->remote.id);

        channel->adjust_state = libssh2_NB_state_created;
    }

    rc = _libssh2_transport_send(channel->session, channel->adjust_adjust, 9,
                                 NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(channel->session, rc,
                       "Would block sending window adjust");
        return rc;
    }
    else if(rc) {
        channel->adjust_queue = adjustment;
        return _libssh2_error(channel->session, LIBSSH2_ERROR_SOCKET_SEND,
                              "Unable to send transfer-window adjustment "
                              "packet, deferring");
    }
    else {
        channel->remote.window_size += adjustment;
    }

    channel->adjust_state = libssh2_NB_state_idle;

    return 0;
}

```

这段代码定义了一个名为 `libssh2_channel_receive_window_adjust` 的函数，它是 `libssh2_channel_receive_window_adjust` 函数的别名。这个函数的主要作用是调整一个频道（channel）的接收窗口（receive window）。通过调整接收窗口中的字节数，可以控制数据包的接收速率。函数接受三个参数：`channel` 表示要调整的频道，`adj` 表示要调整的字节数，`force` 表示是否强制调整（default 是 false）。函数返回一个新的 `window` 字段，表示调整后接收窗口的大小。注意，函数可能会返回 `EAGAIN`，这种情况通常被认为是严重的错误。


```cpp
/*
 * libssh2_channel_receive_window_adjust
 *
 * DEPRECATED
 *
 * Adjust the receive window for a channel by adjustment bytes. If the amount
 * to be adjusted is less than LIBSSH2_CHANNEL_MINADJUST and force is 0 the
 * adjustment amount will be queued for a later packet.
 *
 * Returns the new size of the receive window (as understood by remote end).
 * Note that it might return EAGAIN too which is highly stupid.
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

    /* stupid - but this is how it was made to work before and this is just
       kept for backwards compatibility */
    return rc ? (unsigned long)rc : window;
}

```

这段代码定义了一个名为 `libssh2_channel_receive_window_adjust2` 的函数，它是 `libssh2_channel_receive_window_adjust` 的别名。这个函数的主要作用是调整一个通道的接收窗口大小。它接受三个参数：`channel` 是用于传输数据的 `libssh2_channel` 结构的指针，`adj` 是要调整的接收窗口大小，`force` 是是否强制调整，`window` 是一个指向 `unsigned int` 类型的指针，用于保存调整后的接收窗口大小。

函数实现中首先检查 `channel` 是否为 `NULL`，如果是，则表示函数无法执行，返回错误码 `LIBSSH2_ERROR_BAD_USE`。然后进行底层调优，根据 `adj` 调整接收窗口大小，并将结果保存到 `window` 指向的内存空间中。最后，函数返回 `0`，表示成功，如果没有错误，则返回 `LIBSSH2_ERROR_NONE`。


```cpp
/*
 * libssh2_channel_receive_window_adjust2
 *
 * Adjust the receive window for a channel by adjustment bytes. If the amount
 * to be adjusted is less than LIBSSH2_CHANNEL_MINADJUST and force is 0 the
 * adjustment amount will be queued for a later packet.
 *
 * Stores the new size of the receive window in the data 'window' points to.
 *
 * Returns the "normal" error code: 0 for success, negative for failure.
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

    BLOCK_ADJUST(rc, channel->session,
                 _libssh2_channel_receive_window_adjust(channel, adj, force,
                                                        window));
    return rc;
}

```

这段代码是一个名为 `_libssh2_channel_extended_data` 的函数，它是 `libssh2_channel_extended_data` 函数的扩展。

该函数的作用是，当您使用 `libssh2_channel_extended_data` 函数时，如果当前连接处于空闲状态，并且 `ignore_mode` 设置为 0，那么将设置 `channel` 的远程扩展数据设置为指定的值，并将 `channel` 的远程扩展数据忽略模式设置为指定的值。如果当前连接处于活动状态，或者 `ignore_mode` 设置为 `libssh2_channel_extended_data_ignore`，那么执行 `_libssh2_channel_flush` 函数，将其保留在内存中。

总之，该函数的主要作用是确保远程连接的扩展数据正确设置，并在需要时将其保留在内存中。


```cpp
int
_libssh2_channel_extended_data(LIBSSH2_CHANNEL *channel, int ignore_mode)
{
    if(channel->extData2_state == libssh2_NB_state_idle) {
        _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                       "Setting channel %lu/%lu handle_extended_data"
                       " mode to %d",
                       channel->local.id, channel->remote.id, ignore_mode);
        channel->remote.extended_data_ignore_mode = (char)ignore_mode;

        channel->extData2_state = libssh2_NB_state_created;
    }

    if(channel->extData2_state == libssh2_NB_state_idle) {
        if(ignore_mode == LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE) {
            int rc =
                _libssh2_channel_flush(channel,
                                       LIBSSH2_CHANNEL_FLUSH_EXTENDED_DATA);
            if(LIBSSH2_ERROR_EAGAIN == rc)
                return rc;
        }
    }

    channel->extData2_state = libssh2_NB_state_idle;
    return 0;
}

```

这段代码定义了一个名为 `libssh2_channel_handle_extended_data2()` 的函数，它是 `libssh2_channel_handle_extended_data()` 的扩展版本。

函数接受两个参数：一个 `LIBSSH2_CHANNEL` 类型的通道对象和一个新的 `int` 类型的模式参数。函数的主要作用是确保 `channel` 参数的有效性，然后执行传递给 `_libssh2_channel_extended_data()` 函数的扩展数据处理。

函数首先检查 `channel` 是否为 `NULL`，如果是，则返回 `LIBSSH2_ERROR_BAD_USE`，通常这意味着您的程序遇到了错误。

接下来，函数调用 `_libssh2_channel_extended_data()` 函数处理 `channel` 和 `mode` 提供的数据。函数使用 `BLOCK_ADJUST()` 函数来确保 `rc` 变量的正确值。最后，函数返回 `rc` 的值。


```cpp
/*
 * libssh2_channel_handle_extended_data2()
 *
 */
LIBSSH2_API int
libssh2_channel_handle_extended_data2(LIBSSH2_CHANNEL *channel,
                                      int mode)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, _libssh2_channel_extended_data(channel,
                                                                      mode));
    return rc;
}

```

这段代码定义了一个名为`libssh2_channel_handle_extended_data`的函数，属于`libssh2_channel_handle_extended_data`函数家族。它的作用是处理SSH2通道中的扩展数据，包括读取和写入。

函数接受一个`LIBSSH2_CHANNEL`类型的参数`channel`，一个表示是否忽略扩展数据的`int`类型的参数`ignore_mode`。函数内部调用了`libssh2_channel_handle_extended_data2`函数，将`ignore_mode`参数传给该函数，以实现对扩展数据的正确处理。


```cpp
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
    (void)libssh2_channel_handle_extended_data2(channel, ignore_mode);
}



```

This function appears to be a part of a network library in the context of SSH, where it is responsible for reading data from a remote host's stream identified by a given stream ID. It works with the SSH2 protocol and is able to detect when the channel is at the end of data (EOF) or even closed.

Here's how the function works:

1. It initializes a variable called `bytes_want` to the amount of data to read from the remote host.
2. It initializes a variable called `channel` to a pointer to the stream object.
3. It initializes a variable called `readpkt` to a pointer to the current packet containing data.
4. It sets up a loop that reads data from the remote host until it reaches the end of the channel or the stream is closed.
5. It copies the data from the current packet to the `buf` array, taking care of any buffer overflows or shortages.
6. It advances the pointer to the next packet and the counter, and checks whether the stream has sent the final stream ID by comparing the `unlink_packet` flag.
7. If the stream has sent the final stream ID, it signals to the host that the connection is closed and any resources should be freed.
8. If the channel is already at EOF or even closed, it returns an error.
9. If the transport layer returns an error indicating that the operation was blocked, it returns an error.

The variable `read_packet` is defined last but not least, and it is passed as an argument to the `read_next` function, which is called by the main loop.

This function is quite useful in ensuring that the client is not alone when it is trying to retrieve data from the server, and it is also very helpful in case the stream is already closed or EOF.


```cpp
/*
 * _libssh2_channel_read
 *
 * Read data from a channel
 *
 * It is important to not return 0 until the currently read channel is
 * complete. If we read stuff from the wire but it was no payload data to fill
 * in the buffer with, we MUST make sure to return LIBSSH2_ERROR_EAGAIN.
 *
 * The receive window must be maintained (enlarged) by the user of this
 * function.
 */
ssize_t _libssh2_channel_read(LIBSSH2_CHANNEL *channel, int stream_id,
                              char *buf, size_t buflen)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc;
    size_t bytes_read = 0;
    size_t bytes_want;
    int unlink_packet;
    LIBSSH2_PACKET *read_packet;
    LIBSSH2_PACKET *read_next;

    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                   "channel_read() wants %d bytes from channel %lu/%lu "
                   "stream #%d",
                   (int) buflen, channel->local.id, channel->remote.id,
                   stream_id);

    /* expand the receiving window first if it has become too narrow */
    if((channel->read_state == libssh2_NB_state_jump1) ||
       (channel->remote.window_size <
        channel->remote.window_size_initial / 4 * 3 + buflen) ) {

        uint32_t adjustment = channel->remote.window_size_initial + buflen -
            channel->remote.window_size;
        if(adjustment < LIBSSH2_CHANNEL_MINADJUST)
            adjustment = LIBSSH2_CHANNEL_MINADJUST;

        /* the actual window adjusting may not finish so we need to deal with
           this special state here */
        channel->read_state = libssh2_NB_state_jump1;
        rc = _libssh2_channel_receive_window_adjust(channel, adjustment,
                                                    0, NULL);
        if(rc)
            return rc;

        channel->read_state = libssh2_NB_state_idle;
    }

    /* Process all pending incoming packets. Tests prove that this way
       produces faster transfers. */
    do {
        rc = _libssh2_transport_read(session);
    } while(rc > 0);

    if((rc < 0) && (rc != LIBSSH2_ERROR_EAGAIN))
        return _libssh2_error(session, rc, "transport read");

    read_packet = _libssh2_list_first(&session->packets);
    while(read_packet && (bytes_read < buflen)) {
        /* previously this loop condition also checked for
           !channel->remote.close but we cannot let it do this:

           We may have a series of packets to read that are still pending even
           if a close has been received. Acknowledging the close too early
           makes us flush buffers prematurely and loose data.
        */

        LIBSSH2_PACKET *readpkt = read_packet;

        /* In case packet gets destroyed during this iteration */
        read_next = _libssh2_list_next(&readpkt->node);

        if(readpkt->data_len < 5) {
            read_packet = read_next;
            _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                           "Unexpected packet length");
            continue;
        }

        channel->read_local_id =
            _libssh2_ntohu32(readpkt->data + 1);

        /*
         * Either we asked for a specific extended data stream
         * (and data was available),
         * or the standard stream (and data was available),
         * or the standard stream with extended_data_merge
         * enabled and data was available
         */
        if((stream_id
             && (readpkt->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA)
             && (channel->local.id == channel->read_local_id)
             && (readpkt->data_len >= 9)
             && (stream_id == (int) _libssh2_ntohu32(readpkt->data + 5)))
            || (!stream_id && (readpkt->data[0] == SSH_MSG_CHANNEL_DATA)
                && (channel->local.id == channel->read_local_id))
            || (!stream_id
                && (readpkt->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA)
                && (channel->local.id == channel->read_local_id)
                && (channel->remote.extended_data_ignore_mode ==
                    LIBSSH2_CHANNEL_EXTENDED_DATA_MERGE))) {

            /* figure out much more data we want to read */
            bytes_want = buflen - bytes_read;
            unlink_packet = FALSE;

            if(bytes_want >= (readpkt->data_len - readpkt->data_head)) {
                /* we want more than this node keeps, so adjust the number and
                   delete this node after the copy */
                bytes_want = readpkt->data_len - readpkt->data_head;
                unlink_packet = TRUE;
            }

            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "channel_read() got %d of data from %lu/%lu/%d%s",
                           bytes_want, channel->local.id,
                           channel->remote.id, stream_id,
                           unlink_packet?" [ul]":"");

            /* copy data from this struct to the target buffer */
            memcpy(&buf[bytes_read],
                   &readpkt->data[readpkt->data_head], bytes_want);

            /* advance pointer and counter */
            readpkt->data_head += bytes_want;
            bytes_read += bytes_want;

            /* if drained, remove from list */
            if(unlink_packet) {
                /* detach readpkt from session->packets list */
                _libssh2_list_remove(&readpkt->node);

                LIBSSH2_FREE(session, readpkt->data);
                LIBSSH2_FREE(session, readpkt);
            }
        }

        /* check the next struct in the chain */
        read_packet = read_next;
    }

    if(!bytes_read) {
        /* If the channel is already at EOF or even closed, we need to signal
           that back. We may have gotten that info while draining the incoming
           transport layer until EAGAIN so we must not be fooled by that
           return code. */
        if(channel->remote.eof || channel->remote.close)
            return 0;
        else if(rc != LIBSSH2_ERROR_EAGAIN)
            return 0;

        /* if the transport layer said EAGAIN then we say so as well */
        return _libssh2_error(session, rc, "would block");
    }

    channel->read_avail -= bytes_read;
    channel->remote.window_size -= bytes_read;

    return bytes_read;
}

```

这段代码定义了一个名为`libssh2_channel_read_ex`的函数，它是`libssh2_channel_read`的辅助函数。

这个函数的主要作用是读取一个SSH2channel中的数据，以阻塞或非阻塞方式取决于设置状态。在非阻塞模式下，函数需要确保在当前正在读取的频道完全加载完数据之后，不会返回0。否则，应该返回`libssh2_error_eagain`。

具体实现中，函数首先会检查接收窗口是否足够大以接收一个完整的缓冲区的内容。如果应用可能需要调整接收窗口以提高传输性能，则可以根据需要调整接收窗口大小。


```cpp
/*
 * libssh2_channel_read_ex
 *
 * Read data from a channel (blocking or non-blocking depending on set state)
 *
 * When this is done non-blocking, it is important to not return 0 until the
 * currently read channel is complete. If we read stuff from the wire but it
 * was no payload data to fill in the buffer with, we MUST make sure to return
 * LIBSSH2_ERROR_EAGAIN.
 *
 * This function will first make sure there's a receive window enough to
 * receive a full buffer's wort of contents. An application may choose to
 * adjust the receive window more to increase transfer performance.
 */
LIBSSH2_API ssize_t
```

这段代码是一个用于从SSH通道中读取数据的函数。其作用是将读取的数据存储在buf数组中，并返回读取结果。以下是代码的更详细解释：

```cppc
#include <ssh2.h>
#include <stdint.h>
```

```cppc
int libssh2_channel_read_ex(LIBSSH2_CHANNEL *channel, int stream_id, char *buf,
                       size_t buflen)
```

函数名，说明：
- libssh2_channel_read_ex: 返回值类型为LIBSSH2_CHANNEL类型的整数类型。
- channel：指向要读取数据的SSH通道的指针。
- stream_id：要读取的通道的ID,0表示全部通道，1表示只读通道，2表示只写通道。
- buf：用于存储读取到的数据的缓冲区，由user指定。
- buflen：指定了从channel中读取的数据的最大长度，单位是字节。

```cppc
   int rc;
   unsigned long recv_window;
```

- rc:int类型的函数返回值，用于表示读取操作的返回状态，可能是有用的错误信息。
- recv_window:unsigned long类型的局部变量，用于记录从channel中读取的数据窗口大小，即正在读取的数据长度。

```cppc
   if(!channel)
       return LIBSSH2_ERROR_BAD_USE;
```

- channel不能为0，否则函数会返回LIBSSH2_ERROR_BAD_USE错误。

```cppc
   recv_window = libssh2_channel_window_read_ex(channel, NULL, NULL);
```

- function调用libssh2_channel_window_read_ex，从channel中读取数据窗口。
- NULL表示没有从channel中读取到有效数据，函数不会做任何错误处理，直接返回。

```cppc
   if(buflen > recv_window) {
       BLOCK_ADJUST(rc, channel->session,
                    _libssh2_channel_receive_window_adjust(channel, buflen,
                                                           1, NULL));
   }
```

- 如果读取数据的长度比数据窗口长度还长，就需要调整接收窗口大小，以减少需要读取的数据长度。
- BLOCK_ADJUST函数用于在libssh2_channel_window_read_ex函数中执行异步调整，以保证读取操作的顺利进行。
- _libssh2_channel_receive_window_adjust函数是一个异步函数，其作用是调整接收窗口大小，以允许正确的数据长度被读取。函数的第一个参数是一个指向LIBSSH2_CHANNEL类型的channel指向，第二个参数是一个表示数据窗口大小的int类型，第三个参数是一个int类型，用于指定调整方向，即从channel中读取数据时，从channel中读取的数据窗口大小会减小。第四个参数是一个void指向，表示不需要执行任何操作。
- 函数实现完毕，返回rc,rc为0表示函数正常返回，否则根据错误代码进行处理。

```cppc
   BLOCK_ADJUST(rc, channel->session,
                    _libssh2_channel_read(channel, stream_id, buf, buflen));
   return rc;
```

- BLOCK_ADJUST函数用于在libssh2_channel_window_read_ex函数中执行异步调整，以保证读取操作的顺利进行。函数的第一个参数是一个指向LIBSSH2_CHANNEL类型的channel指向，第二个参数是一个表示数据窗口大小的int类型，第三个参数是一个int类型，用于指定调整方向，即从channel中读取数据时，从channel中读取的数据窗口大小会减小。第四个参数是一个void指向，表示不需要执行任何操作。
- _libssh2_channel_read函数用于从channel中读取数据，实现从channel中读取数据的一行数据，并把数据存到buf数组中。函数的第一个参数是一个指向LIBSSH2_CHANNEL类型的channel指向，第二个参数是一个表示数据大小的int类型，第三个参数是一个int类型，用于指定从channel中读取的数据行数。第四个参数是一个int类型，用于指定buf数组的大小。函数返回rc,rc为0表示函数正常返回，否则根据错误代码进行处理。
- 函数实现完毕，返回rc,rc为0表示函数正常返回，否则根据错误代码进行处理。


```cpp
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

```

这段代码是libssh2_ssh_channel_connect函数，负责处理客户端连接到服务器端的连接建立和数据接收。

while(read_packet)循环接收数据，每次接收的数据可能是客户端发送的下一个数据包，或者是服务器端返回的数据。

首先，代码检查接收的数据是否是下一个数据包，如果不是，则继续接收下一个数据包。如果接收到了下一个数据包，就获取该数据包中的本地ID，并检查该ID是否是一个已知的扩展数据流ID，如果是，则执行后续操作；否则，继续接收下一个数据包。

如果是扩展数据流，则根据选项分别执行不同的操作：

- 如果扩展数据流ID是已知的，并且该数据流支持合并，代码将合并两个数据流并覆盖next_packet，使用channel->local.id作为合并后的数据的本地ID。
- 如果扩展数据流ID是已知的，并且该数据流不支持合并，代码将执行下一步操作，即使用channel->local.id作为本地ID，将next_packet中的数据作为数据的下一部分。
- 如果扩展数据流ID是一个未知的数据流，代码将接收该数据流中的所有数据，并将next_packet中的数据作为扩展数据流。

在循环结束后，如果已经接收了所有需要接收的数据，则代码将返回0，否则将执行下一次循环。


```cpp
/*
 * _libssh2_channel_packet_data_len
 *
 * Return the size of the data block of the current packet, or 0 if there
 * isn't a packet.
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

    while(read_packet) {

        next_packet = _libssh2_list_next(&read_packet->node);

        if(read_packet->data_len < 5) {
            read_packet = next_packet;
            _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                           "Unexpected packet length");
            continue;
        }

        read_local_id = _libssh2_ntohu32(read_packet->data + 1);

        /*
         * Either we asked for a specific extended data stream
         * (and data was available),
         * or the standard stream (and data was available),
         * or the standard stream with extended_data_merge
         * enabled and data was available
         */
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

        read_packet = next_packet;
    }

    return 0;
}

```

This function appears to be a part of the SSH client library, written in C. It is used to handle the writing of data to the SSH connection.

It receives a packet from the write channel, consisting of a piece of data and its length, and sends it to the server using the `libssh2_transport_send()` function.

If the send operation is successful, the function returns immediately, otherwise it returns an error. If the send operation fails, the function returns an error, explaining the cause of the failure.

It appears that the function also handles the case where the write buffer is larger than the amount of data to be sent, and it will shrink the local window size and update the write state accordingly.

It is important to note that the function is not thread safe, and should be called from a thread safe function, or even better, from a thread safe storage.


```cpp
/*
 * _libssh2_channel_write
 *
 * Send data to a channel. Note that if this returns EAGAIN, the caller must
 * call this function again with the SAME input arguments.
 *
 * Returns: number of bytes sent, or if it returns a negative number, that is
 * the error code!
 */
ssize_t
_libssh2_channel_write(LIBSSH2_CHANNEL *channel, int stream_id,
                       const unsigned char *buf, size_t buflen)
{
    int rc = 0;
    LIBSSH2_SESSION *session = channel->session;
    ssize_t wrote = 0; /* counter for this specific this call */

    /* In theory we could split larger buffers into several smaller packets
     * but it turns out to be really hard and nasty to do while still offering
     * the API/prototype.
     *
     * Instead we only deal with the first 32K in this call and for the parent
     * function to call it again with the remainder! 32K is a conservative
     * limit based on the text in RFC4253 section 6.1.
     */
    if(buflen > 32700)
        buflen = 32700;

    if(channel->write_state == libssh2_NB_state_idle) {
        unsigned char *s = channel->write_packet;

        _libssh2_debug(channel->session, LIBSSH2_TRACE_CONN,
                       "Writing %d bytes on channel %lu/%lu, stream #%d",
                       (int) buflen, channel->local.id, channel->remote.id,
                       stream_id);

        if(channel->local.close)
            return _libssh2_error(channel->session,
                                  LIBSSH2_ERROR_CHANNEL_CLOSED,
                                  "We've already closed this channel");
        else if(channel->local.eof)
            return _libssh2_error(channel->session,
                                  LIBSSH2_ERROR_CHANNEL_EOF_SENT,
                                  "EOF has already been received, "
                                  "data might be ignored");

        /* drain the incoming flow first, mostly to make sure we get all
         * pending window adjust packets */
        do
            rc = _libssh2_transport_read(session);
        while(rc > 0);

        if((rc < 0) && (rc != LIBSSH2_ERROR_EAGAIN)) {
            return _libssh2_error(channel->session, rc,
                                  "Failure while draining incoming flow");
        }

        if(channel->local.window_size <= 0) {
            /* there's no room for data so we stop */

            /* Waiting on the socket to be writable would be wrong because we
             * would be back here immediately, but a readable socket might
             * herald an incoming window adjustment.
             */
            session->socket_block_directions = LIBSSH2_SESSION_BLOCK_INBOUND;

            return (rc == LIBSSH2_ERROR_EAGAIN?rc:0);
        }

        channel->write_bufwrite = buflen;

        *(s++) = stream_id ? SSH_MSG_CHANNEL_EXTENDED_DATA :
            SSH_MSG_CHANNEL_DATA;
        _libssh2_store_u32(&s, channel->remote.id);
        if(stream_id)
            _libssh2_store_u32(&s, stream_id);

        /* Don't exceed the remote end's limits */
        /* REMEMBER local means local as the SOURCE of the data */
        if(channel->write_bufwrite > channel->local.window_size) {
            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "Splitting write block due to %lu byte "
                           "window_size on %lu/%lu/%d",
                           channel->local.window_size, channel->local.id,
                           channel->remote.id, stream_id);
            channel->write_bufwrite = channel->local.window_size;
        }
        if(channel->write_bufwrite > channel->local.packet_size) {
            _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                           "Splitting write block due to %lu byte "
                           "packet_size on %lu/%lu/%d",
                           channel->local.packet_size, channel->local.id,
                           channel->remote.id, stream_id);
            channel->write_bufwrite = channel->local.packet_size;
        }
        /* store the size here only, the buffer is passed in as-is to
           _libssh2_transport_send() */
        _libssh2_store_u32(&s, channel->write_bufwrite);
        channel->write_packet_len = s - channel->write_packet;

        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Sending %d bytes on channel %lu/%lu, stream_id=%d",
                       (int) channel->write_bufwrite, channel->local.id,
                       channel->remote.id, stream_id);

        channel->write_state = libssh2_NB_state_created;
    }

    if(channel->write_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->write_packet,
                                     channel->write_packet_len,
                                     buf, channel->write_bufwrite);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, rc,
                                  "Unable to send channel data");
        }
        else if(rc) {
            channel->write_state = libssh2_NB_state_idle;
            return _libssh2_error(session, rc,
                                  "Unable to send channel data");
        }
        /* Shrink local window size */
        channel->local.window_size -= channel->write_bufwrite;

        wrote += channel->write_bufwrite;

        /* Since _libssh2_transport_write() succeeded, we must return
           now to allow the caller to provide the next chunk of data.

           We cannot move on to send the next piece of data that may
           already have been provided in this same function call, as we
           risk getting EAGAIN for that and we can't return information
           both about sent data as well as EAGAIN. So, by returning short
           now, the caller will call this function again with new data to
           send */

        channel->write_state = libssh2_NB_state_idle;

        return wrote;
    }

    return LIBSSH2_ERROR_INVAL; /* reaching this point is really bad */
}

```

这段代码是一个名为 `libssh2_channel_write_ex` 的函数，它是 `libssh2_channel` 系列的库函数，用于向远程主机发送数据到远程主机的某个频道（channel）中。

具体来说，这段代码的作用是：

1. 检查传递给它的 `channel` 是否为 `NULL`，如果是，则表示给错误的参数，应该返回 `LIBSSH2_ERROR_BAD_USE`。
2. 如果 `channel` 不是 `NULL`，那么进行以下操作：

a. 获取远程主机对 `channel` 的会话，并将其存储在 `channel->session` 指向的内存区域中。

b. 使用 `_libssh2_channel_write` 函数将 `channel` 中的 `stream_id` 和 `buf` 发送到远程主机。

c. 使用 `BLOCK_ADJUST` 函数来调整块大小，以便可以正确写入数据。

d. 返回 `rc`，表示函数是否成功完成。

这段代码是用于向远程主机发送数据到远程主机的某个频道，并可以支持交互式会话（stream）的发送。


```cpp
/*
 * libssh2_channel_write_ex
 *
 * Send data to a channel
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

```

这段代码定义了一个名为 `channel_send_eof` 的函数，属于名为 `LIBSSH2_CHANNEL` 的库中的函数。它的作用是向名为 `channel` 的 Channels 的会话发送完成 EOF（End-of-Frame）数据包，以便表示数据传输结束。以下是函数的实现细节：

1. 首先，获取名为 `session` 的会话对象和名为 `channel` 的 Channels 对象。
2. 创建一个名为 `packet` 的 5 字节长的字符数组，用于存储数据包。
3. 设置数据包的第一个字节为 `SSH_MSG_CHANNEL_EOF`，然后设置 `channel` 的远程 ID。
4. 调用 `_libssh2_transport_send` 函数，将数据包发送到 `channel` 的远程 IP 地址。
5. 如果发送失败，函数将返回错误代码，并打印错误信息。
6. 如果成功发送，函数将设置 `channel` 的本地 EOF 为 `1`，以便在以后的数据传输中使用。
7. 函数返回 `0`，表示成功发送 EOF。


```cpp
/*
 * channel_send_eof
 *
 * Send EOF on channel
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

```

这段代码定义了一个名为`libssh2_channel_send_eof`的函数，它属于`libssh2_channel`类。

这个函数的目的是在给定的`LIBSSH2_CHANNEL`对象上发送EOF（ End-of-File ）字符，以便接收方可以在此后读取缓冲区中的数据并关闭连接。

具体来说，函数首先检查`channel`参数是否为有效的`LIBSSH2_CHANNEL`对象。如果是，函数调用一个名为`channel_send_eof`的函数，传递给`channel_send_eof`一个`LIBSSH2_CHANNEL`对象和一个名为`session`的参数，`session`是在`channel`和`channel_send_eof`之间传递的一个元数据结构。

函数接着使用`BLOCK_ADJUST`函数来调整返回值，确保其符合`LIBSSH2_ERROR_BAD_USE`到`LIBSSH2_ERROR_OK`的预定义范围。如果函数成功执行，它将返回`LIBSSH2_ERROR_OK`，否则它将返回其他错误代码。


```cpp
/*
 * libssh2_channel_send_eof
 *
 * Send EOF on channel
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

```

这段代码是一个名为 `libssh2_channel_eof` 的函数，它是 `libssh2_channel` 类的成员函数。它的作用是读取一个 `LIBSSH2_CHANNEL` 类型的数据 channel 中数据的变化情况，当 channel 中没有数据时，函数返回 EOF。

具体来说，这段代码首先检查 channel 是否为空，如果是，函数返回 EOF。然后，它创建一个 `LIBSSH2_SESSION` 类型的变量 `session` 和一个包含 `LIBSSH2_PACKET` 结构体数组的 `LIBSSH2_PACKET` 变量 `packet`。接着，它遍历 channel 中所有的数据包，并在每个数据包中检查其数据长度是否大于 1。如果是，并且数据是读取可用的，函数就返回 0，表示没有 EOF 状态。否则，函数继续遍历数据包，直到遍历完所有的数据包。

函数的实现中，首先定义了一个名为 `next_packet` 的变量，用于存储下一个数据包。然后，它遍历 channel 中所有的数据包，并在每个数据包中创建一个名为 `packet` 的新的 `LIBSSH2_PACKET` 变量，用于存储下一个数据包。通过 `next_packet` 变量，实现了数据包的循环遍历，直到所有数据包都被处理完毕。


```cpp
/*
 * libssh2_channel_eof
 *
 * Read channel's eof status
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
    packet = _libssh2_list_first(&session->packets);

    while(packet) {

        next_packet = _libssh2_list_next(&packet->node);

        if(packet->data_len < 1) {
            packet = next_packet;
            _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                           "Unexpected packet length");
            continue;
        }

        if(((packet->data[0] == SSH_MSG_CHANNEL_DATA)
            || (packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA))
            && ((packet->data_len >= 5)
            && (channel->local.id == _libssh2_ntohu32(packet->data + 1)))) {
            /* There's data waiting to be read yet, mask the EOF status */
            return 0;
        }
        packet = next_packet;
    }

    return channel->remote.eof;
}

```

这段代码是一个名为 `channel_wait_eof` 的函数，它是 `libssh2_channel` 结构的一个成员函数。它的作用是等待信道 EOF，以便在收到 EOF 前继续从网络上读取数据。

具体来说，这段代码在以下几个方面进行操作：

1. 检查信道是否已满，如果是，则等待 EOF 并设置 `wait_eof_state` 状态为 `libssh2_NB_state_created`。
2. 进入一个循环，直到信道已满并收到 EOF。在循环中，会尝试从网络上读取数据，如果遇到设置 `api_block_mode` 为 `1` 的出错，则退出循环并返回错误信息。
3. 如果循环中没有收到 EOF，或者信道已满，则将 `wait_eof_state` 设置为 `libssh2_NB_state_idle`，以便进入下一个循环。
4. 如果循环中发生了错误，则返回错误信息。

这段代码是在等待信道 EOF 时确保了出错处理，可以在信道已满时进行错误处理，并可以防止因网络超时而导致的死锁。


```cpp
/*
 * channel_wait_eof
 *
 * Awaiting channel EOF
 */
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

```

这段代码是一个名为`libssh2_channel_wait_eof`的函数，它是`libssh2_channel`类的扩展，用于等待SSH通道EOF（终点）事件的发生。

具体来说，这个函数接受一个`LIBSSH2_CHANNEL`类型的参数，代表一个SSH通道对象。函数内部首先检查传递给自己的参数是否为空，如果是，则返回一个错误代码。然后，函数调用了`channel_wait_eof`函数，这个函数可能是从`libssh2_channel_poll`函数中继承而来的。在`channel_wait_eof`函数中，可能执行了一些与通道EOF事件相关的操作，例如清除超时时间或者发送SIGQUIT信号。最后，函数返回`rc`作为结果。


```cpp
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

```



This is a function definition for `libssh2_channel_close` which appears to handle the logic for closing a secure connection (SSH) channel. Here's how the function behaves:

1. It checks the close state of the channel. If it's `libssh2_NB_state_created`, the function sends a close packet to the remote host using `_libssh2_transport_send` and then sets the local close state to 1 using `channel->local.close = 1`.
2. If the function returns LIBSSH2_ERROR_EAGAIN, it waits for an EAGAIN response from the remote host and then falls through to the next possible action (e.g., `LIBSSH2_CHANNEL_CLOSE`).
3. If the function returns 0 or an error, it sets the local close state to `libssh2_NB_state_idle` and calls the close callback function, if it exists.

It's worth noting that the close callback function is passed to `libssh2_channel_open` and is called whenever the channel is closed. The close callback function receives a pointer to a `LIBSSH2_CHANNEL` struct, which contains information about the closed channel, and should use this information to clean up any resources or calls to other functions.


```cpp
int _libssh2_channel_close(LIBSSH2_CHANNEL * channel)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc = 0;

    if(channel->local.close) {
        /* Already closed, act like we sent another close,
         * even though we didn't... shhhhhh */
        channel->close_state = libssh2_NB_state_idle;
        return 0;
    }

    if(!channel->local.eof) {
        rc = channel_send_eof(channel);
        if(rc) {
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                return rc;
            }
            _libssh2_error(session, rc,
                "Unable to send EOF, but closing channel anyway");
        }
    }

    /* ignore if we have received a remote eof or not, as it is now too
       late for us to wait for it. Continue closing! */

    if(channel->close_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN, "Closing channel %lu/%lu",
                       channel->local.id, channel->remote.id);

        channel->close_packet[0] = SSH_MSG_CHANNEL_CLOSE;
        _libssh2_htonu32(channel->close_packet + 1, channel->remote.id);

        channel->close_state = libssh2_NB_state_created;
    }

    if(channel->close_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, channel->close_packet, 5,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, rc,
                           "Would block sending close-channel");
            return rc;

        }
        else if(rc) {
            _libssh2_error(session, rc,
                           "Unable to send close-channel request, "
                           "but closing anyway");
            /* skip waiting for the response and fall through to
               LIBSSH2_CHANNEL_CLOSE below */

        }
        else
            channel->close_state = libssh2_NB_state_sent;
    }

    if(channel->close_state == libssh2_NB_state_sent) {
        /* We must wait for the remote SSH_MSG_CHANNEL_CLOSE message */

        while(!channel->remote.close && !rc &&
               (session->socket_state != LIBSSH2_SOCKET_DISCONNECTED))
            rc = _libssh2_transport_read(session);
    }

    if(rc != LIBSSH2_ERROR_EAGAIN) {
        /* set the local close state first when we're perfectly confirmed to
           not do any more EAGAINs */
        channel->local.close = 1;

        /* We call the callback last in this function to make it keep the local
           data as long as EAGAIN is returned. */
        if(channel->close_cb) {
            LIBSSH2_CHANNEL_CLOSE(session, channel);
        }

        channel->close_state = libssh2_NB_state_idle;
    }

    /* return 0 or an error */
    return rc >= 0 ? 0 : rc;
}

```

这段代码是一个名为 `libssh2_channel_close` 的函数，它是 LibSSH2 库中的一个函数，用于关闭一个通道。该函数接受一个 `LIBSSH2_CHANNEL` 类型的参数，返回一个整数类型的结果。

具体来说，这段代码实现了一个 blocking 操作，即在函数内部对传入的 `channel` 参数进行了阻塞，直到函数返回了结果或者是遇到错误。如果 `channel` 为 `NULL`，函数将返回 LIBSSH2_ERROR_BAD_USE，否则，函数将调用 `_libssh2_channel_close` 函数，传递给 `channel` 的 `session` 参数，并且返回该函数的返回值。

这段代码的具体实现可能还涉及到一些其他的底层操作，比如关闭通道时可能需要清空该通道的相关数据，或者将该通道的状态记录到其他地方等等。


```cpp
/*
 * libssh2_channel_close
 *
 * Close a channel
 */
LIBSSH2_API int
libssh2_channel_close(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, _libssh2_channel_close(channel) );
    return rc;
}

```

这段代码是一个名为`channel_wait_closed`的函数，属于`libssh2_channel`结构的子结构。它的作用是等待远程关闭的频道，并在频道关闭时通知主程序。

具体来说，代码首先检查频道是否处于EOF状态，如果不是，则说明频道已经准备好关闭，可以返回。如果频道已经关闭，则进入等待状态，直到频道真正关闭，或者主程序调用这个函数。

如果频道没有关闭，代码会从远程服务器持续读取数据，直到收到正确的关闭信号。注意，如果主程序调用这个函数，则在远程服务器没有正确关闭频道时，程序不会继续等待，而是直接返回。


```cpp
/*
 * channel_wait_closed
 *
 * Awaiting channel close after EOF
 */
static int channel_wait_closed(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;
    int rc;

    if(!channel->remote.eof) {
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "libssh2_channel_wait_closed() invoked when "
                              "channel is not in EOF state");
    }

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

    channel->wait_closed_state = libssh2_NB_state_idle;

    return 0;
}

```

这段代码是一个名为 `libssh2_channel_wait_closed` 的函数，它是 `libssh2_channel` 类的原语。它接受一个 `LIBSSH2_CHANNEL` 类型的参数，代表一个已连接的 SSH 通道。

函数的作用是等待通道关闭，并在通道关闭时返回一个状态码。它首先检查传入的参数是否为 `NULL`，如果是，则返回 `LIBSSH2_ERROR_BAD_USE`，表示出错。然后，它调用 `channel_wait_closed` 函数，将参数 `channel` 和一些辅助函数与当前连接的会话进行 block 调整。最后，它返回调用它的函数的返回值，并将结果保存回 `rc` 变量中。

channel_wait_closed 函数是一个辅助函数，它接受一个 `LIBSSH2_CHANNEL` 类型的参数，代表一个已连接的 SSH 通道，并等待 channel 关闭。它返回一个整数，表示通道关闭时返回的状态码。这里大致是实现了一个阻塞调用，当通道关闭时返回 0，否则返回一个错误码。


```cpp
/*
 * libssh2_channel_wait_closed
 *
 * Awaiting channel close after EOF
 */
LIBSSH2_API int
libssh2_channel_wait_closed(LIBSSH2_CHANNEL *channel)
{
    int rc;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, channel->session, channel_wait_closed(channel));
    return rc;
}

```

This function appears to be a part of the SSH server, and it发了ChannelData message to a channel, then it handles the packet and close the channel connection.

Here is a step-by-step explanation of what this function does:

1. It checks if the remote server has an exit signal for this channel. If it does, it frees the exit signal.

2. It checks if the remote server has a close signal for the channel. If it does, it sends the close packet to the server.

3. It checks for any existing packets for this channel and frees them.

4. It removes the node from the channel list and links it to the node's parent.

5. It frees the memory allocated for the setenv\_packet, reqX11\_packet, and process\_packet.

6. It returns 0 to indicate success.


```cpp
/*
 * _libssh2_channel_free
 *
 * Make sure a channel is closed, then remove the channel from the session
 * and free its resource(s)
 *
 * Returns 0 on success, negative on failure
 */
int _libssh2_channel_free(LIBSSH2_CHANNEL *channel)
{
    LIBSSH2_SESSION *session = channel->session;
    unsigned char channel_id[4];
    unsigned char *data;
    size_t data_len;
    int rc;

    assert(session);

    if(channel->free_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                       "Freeing channel %lu/%lu resources", channel->local.id,
                       channel->remote.id);

        channel->free_state = libssh2_NB_state_created;
    }

    /* Allow channel freeing even when the socket has lost its connection */
    if(!channel->local.close
        && (session->socket_state == LIBSSH2_SOCKET_CONNECTED)) {
        rc = _libssh2_channel_close(channel);

        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;

        /* ignore all other errors as they otherwise risk blocking the channel
           free from happening */
    }

    channel->free_state = libssh2_NB_state_idle;

    if(channel->exit_signal) {
        LIBSSH2_FREE(session, channel->exit_signal);
    }

    /*
     * channel->remote.close *might* not be set yet, Well...
     * We've sent the close packet, what more do you want?
     * Just let packet_add ignore it when it finally arrives
     */

    /* Clear out packets meant for this channel */
    _libssh2_htonu32(channel_id, channel->local.id);
    while((_libssh2_packet_ask(session, SSH_MSG_CHANNEL_DATA, &data,
                                &data_len, 1, channel_id, 4) >= 0)
           ||
           (_libssh2_packet_ask(session, SSH_MSG_CHANNEL_EXTENDED_DATA, &data,
                                &data_len, 1, channel_id, 4) >= 0)) {
        LIBSSH2_FREE(session, data);
    }

    /* free "channel_type" */
    if(channel->channel_type) {
        LIBSSH2_FREE(session, channel->channel_type);
    }

    /* Unlink from channel list */
    _libssh2_list_remove(&channel->node);

    /*
     * Make sure all memory used in the state variables are free
     */
    if(channel->setenv_packet) {
        LIBSSH2_FREE(session, channel->setenv_packet);
    }
    if(channel->reqX11_packet) {
        LIBSSH2_FREE(session, channel->reqX11_packet);
    }
    if(channel->process_packet) {
        LIBSSH2_FREE(session, channel->process_packet);
    }

    LIBSSH2_FREE(session, channel);

    return 0;
}

```

这段代码定义了一个名为`libssh2_channel_free`的函数，它是`libssh2_channel_free`函数的别名。该函数接收一个`LIBSSH2_CHANNEL`类型的参数，代表一个SSH会话的频道。函数首先确保频道已经关闭，然后从会话中移除频道并将频道资源释放。最终，函数返回0表示成功，否则返回负数。

具体来说，函数内部首先检查传入的参数是否为`NULL`，如果是，则直接返回`LIBSSH2_ERROR_BAD_USE`。然后，函数调用`_libssh2_channel_free`函数，传递给它的参数是整个`LIBSSH2_CHANNEL`结构体。

接着，函数内部使用`BLOCK_ADJUST`函数来调整传给`_libssh2_channel_free`的参数，确保参数的`rc`值在调用这个函数时被正确分配。函数调用成功后，函数返回`0`。


```cpp
/*
 * libssh2_channel_free
 *
 * Make sure a channel is closed, then remove the channel from the session
 * and free its resource(s)
 *
 * Returns 0 on success, negative on failure
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
```

This function appears to be reading the window size of a channel and setting it to the `window_size_initial` parameter if passed as an argument. It also checks if the window size is already defined and updates it if it is not.

The function takes an optional `read_avail` parameter which is the expected size of the buffer to be populated. It also takes an optional `window_size_initial` parameter which is the window size to be initialized if not passed.

The function reads the packet data of the channel and extracts the packet type and data length. If the packet type is `SSH_MSG_CHANNEL_DATA` or `SSH_MSG_CHANNEL_EXTENDED_DATA`, the function extracts the `id` field from the packet and uses it to calculate the number of bytes in the packet data. The function then updates the `read_avail` parameter with the number of bytes in the packet data.

If the `window_size_initial` parameter is not passed, the function defaults to the size of the channel's window.


```cpp
/*
 * libssh2_channel_window_read_ex
 *
 * Check the status of the read window. Returns the number of bytes which the
 * remote end may send without overflowing the window limit read_avail (if
 * passed) will be populated with the number of bytes actually available to be
 * read window_size_initial (if passed) will be populated with the
 * window_size_initial as defined by the channel_open request
 */
LIBSSH2_API unsigned long
libssh2_channel_window_read_ex(LIBSSH2_CHANNEL *channel,
                               unsigned long *read_avail,
                               unsigned long *window_size_initial)
{
    if(!channel)
        return 0; /* no channel, no window! */

    if(window_size_initial) {
        *window_size_initial = channel->remote.window_size_initial;
    }

    if(read_avail) {
        size_t bytes_queued = 0;
        LIBSSH2_PACKET *next_packet;
        LIBSSH2_PACKET *packet =
            _libssh2_list_first(&channel->session->packets);

        while(packet) {
            unsigned char packet_type;
               next_packet = _libssh2_list_next(&packet->node);

            if(packet->data_len < 1) {
                packet = next_packet;
                _libssh2_debug(channel->session, LIBSSH2_TRACE_ERROR,
                               "Unexpected packet length");
                continue;
            }

            packet_type = packet->data[0];

            if(((packet_type == SSH_MSG_CHANNEL_DATA)
                || (packet_type == SSH_MSG_CHANNEL_EXTENDED_DATA))
               && ((packet->data_len >= 5)
                   && (_libssh2_ntohu32(packet->data + 1) ==
                       channel->local.id))) {
                bytes_queued += packet->data_len - packet->data_head;
            }

            packet = next_packet;
        }

        *read_avail = bytes_queued;
    }

    return channel->remote.window_size;
}

```

这段代码是一个名为 `libssh2_channel_window_write_ex` 的函数，它是 `libssh2_channel` 类的成员函数。它的作用是检查写入窗口的状态，并在不需要阻塞窗口大小时返回可以安全写入的数据量。

函数接收两个参数：一个 `LIBSSH2_CHANNEL` 类型的指针和一个指向 `unsigned long` 类型的指针，分别表示要写入的数据窗口大小和初始化窗口大小的值。如果是远程主机的通道，那么函数首先检查远程主机是否有设置窗口大小，如果没有设置，就使用本地通道的初始窗口大小作为窗口大小。

函数返回通道的当前写入窗口大小，如果通道没有设置窗口大小，则返回 0。


```cpp
/*
 * libssh2_channel_window_write_ex
 *
 * Check the status of the write window Returns the number of bytes which may
 * be safely written on the channel without blocking window_size_initial (if
 * passed) will be populated with the size of the initial window as defined by
 * the channel_open request
 */
LIBSSH2_API unsigned long
libssh2_channel_window_write_ex(LIBSSH2_CHANNEL *channel,
                                unsigned long *window_size_initial)
{
    if(!channel)
        return 0; /* no channel, no window! */

    if(window_size_initial) {
        /* For locally initiated channels this is very often 0, so it's not
         * *that* useful as information goes */
        *window_size_initial = channel->local.window_size_initial;
    }

    return channel->local.window_size;
}

```