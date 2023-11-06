# Nmap源码解析 105

# `libssh2/src/sftp.c`

Hi, how can I help you with the code?


```cpp
/* Copyright (c) 2004-2008, Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2007 Eli Fant <elifantu@mail.ru>
 * Copyright (c) 2009-2019 by Daniel Stenberg
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

这段代码是一个用于连接到服务器并执行SSH命令的Python库。它包含了SSH2的私用头文件、SSH2的SFTP文件操作头文件、并集成了libssh2-sftp、channel、session和SFTP的文件操作头文件。通过引入这些头文件，我们可以使用它们提供的功能，从而实现与服务器建立SSH连接，执行命令行或程序等功能。


```cpp
#include <assert.h>

#include "libssh2_priv.h"
#include "libssh2_sftp.h"
#include "channel.h"
#include "session.h"
#include "sftp.h"

/* Note: Version 6 was documented at the time of writing
 * However it was marked as "DO NOT IMPLEMENT" due to pending changes
 *
 * This release of libssh2 implements Version 5 with automatic downgrade
 * based on server's declaration
 */

```

这段代码定义了SFTP包的类型，包括SSH_FXP_INIT、SSH_FXP_VERSION、SSH_FXP_OPEN、SSH_FXP_CLOSE、SSH_FXP_READ、SSH_FXP_WRITE、SSH_FXP_LSTAT、SSH_FXP_FSTAT、SSH_FXP_SETSTAT和SSH_FXP_FSETSTAT等。这些定义用于在SSH_FXP数据包中传输文件状态信息。

具体来说，这段代码定义了以下几个常量：

- SSH_FXP_INIT：表示SFTP包的初始化请求。
- SSH_FXP_VERSION：表示SFTP包的版本号。
- SSH_FXP_OPEN：表示SFTP包的打开状态。
- SSH_FXP_CLOSE：表示SFTP包的关闭状态。
- SSH_FXP_READ：表示SFTP包的读取状态。
- SSH_FXP_WRITE：表示SFTP包的写入状态。
- SSH_FXP_LSTAT：表示SFTP包的LSTAT状态。
- SSH_FXP_FSTAT：表示SFTP包的FSTAT状态。
- SSH_FXP_SETSTAT：表示SFTP包的SETSTAT状态。
- SSH_FXP_FSETSTAT：表示SFTP包的FSETSTAT状态。
- SSH_FXP_OPENDIR：表示SFTP包的OPENDIR状态。
- SSH_FXP_READDIR：表示SFTP包的READDIR状态。
- SSH_FXP_REMOVE：表示SFTP包的REMOVE状态。
- SSH_FXP_MKDIR：表示SFTP包的MKDIR状态。


```cpp
/* SFTP packet types */
#define SSH_FXP_INIT                            1
#define SSH_FXP_VERSION                         2
#define SSH_FXP_OPEN                            3
#define SSH_FXP_CLOSE                           4
#define SSH_FXP_READ                            5
#define SSH_FXP_WRITE                           6
#define SSH_FXP_LSTAT                           7
#define SSH_FXP_FSTAT                           8
#define SSH_FXP_SETSTAT                         9
#define SSH_FXP_FSETSTAT                        10
#define SSH_FXP_OPENDIR                         11
#define SSH_FXP_READDIR                         12
#define SSH_FXP_REMOVE                          13
#define SSH_FXP_MKDIR                           14
```

这段代码是 C 语言中的预处理指令，定义了一系列的宏，用于定义文件操作系统的文件元数据结构。

具体来说，这段代码定义了以下几个宏：

- SSH_FXP_RMDIR：表示删除目录 "SSH_FXP" 下的所有文件。
- SSH_FXP_REALPATH：表示获取文件实际的路径名。
- SSH_FXP_STAT：表示打印文件状态信息。
- SSH_FXP_RENAME：表示重命名文件。
- SSH_FXP_READLINK：表示创建符号链接。
- SSH_FXP_SYMLINK：表示创建符号链接。
- SSH_FXP_STATUS：表示打印文件状态信息。
- SSH_FXP_HANDLE：表示打印文件句柄。
- SSH_FXP_DATA：表示打印文件数据。
- SSH_FXP_NAME：表示打印文件名。
- SSH_FXP_ATTRS：表示打印文件元数据结构。
- SSH_FXP_EXTENDED：表示文件扩展的支持程度，取值范围为 0-200。
- SSH_FXP_EXTENDED_REPLY：表示文件扩展的回复信息，取值范围为 0-201。


```cpp
#define SSH_FXP_RMDIR                           15
#define SSH_FXP_REALPATH                        16
#define SSH_FXP_STAT                            17
#define SSH_FXP_RENAME                          18
#define SSH_FXP_READLINK                        19
#define SSH_FXP_SYMLINK                         20
#define SSH_FXP_STATUS                          101
#define SSH_FXP_HANDLE                          102
#define SSH_FXP_DATA                            103
#define SSH_FXP_NAME                            104
#define SSH_FXP_ATTRS                           105
#define SSH_FXP_EXTENDED                        200
#define SSH_FXP_EXTENDED_REPLY                  201

/* S_IFREG */
```

这段代码定义了 SSH2 SFTP 文件的文件类型属性。

首先定义了 SFTP文件的文件类型为 SFTP 文件，通过宏定义 #define LIBSSH2_SFTP_SFTPFILETYPE_FILE SFTPFILETYPE 来表示。

然后定义了 SFTP文件的文件类型为目录，通过宏定义 #define LIBSSH2_SFTP_SFTPFILETYPE_DIR  SFTPFILETYPE 来表示。

接下来定义了最大数据包长度为 256 * 1024，通过宏定义 #define LIBSSH2_SFTP_PACKET_MAXLEN LIBSSH2_SFTP_PACKET_MAXLEN 来表示。

接着定义了几个宏，分别表示 SFTP 文件是否可读写，表示文件是否支持 ATOM 类型，表示是否允许在文件中写入二进制数据。

最后定义了两个函数，分别表示创建数据包和刷新数据包。

由于没有调用这些函数，所以不会输出任何数据包。


```cpp
#define LIBSSH2_SFTP_ATTR_PFILETYPE_FILE        0100000
/* S_IFDIR */
#define LIBSSH2_SFTP_ATTR_PFILETYPE_DIR         0040000

#define SSH_FXE_STATVFS_ST_RDONLY               0x00000001
#define SSH_FXE_STATVFS_ST_NOSUID               0x00000002

/* This is the maximum packet length to accept, as larger than this indicate
   some kind of server problem. */
#define LIBSSH2_SFTP_PACKET_MAXLEN  (256 * 1024)

static int sftp_packet_ask(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                           uint32_t request_id, unsigned char **data,
                           size_t *data_len);
static void sftp_packet_flush(LIBSSH2_SFTP *sftp);

```

这段代码定义了一个名为 `sftp_attrsize` 的函数，它的作用是计算 Sftp 属性的大小。这个函数接收一个 `unsigned long` 类型的 `flags` 参数，作为输入参数。函数的实现中，首先定义了一个包含 4 个整数的变量 `return_value`，然后根据输入的 `flags` 每一位，对 `return_value` 进行加法运算，最后将得到的结果作为函数的返回值。

具体来说，这段代码的作用是计算 sftp 属性在二进制结构中所需占据的存储空间。sftp 属性包括文件权限、访问权限、保留时间、最大权限等。通过调用 `sftp_attrsize` 函数，可以保证 sftp 属性的正确存储。


```cpp
/* sftp_attrsize
 * Size that attr with this flagset will occupy when turned into a bin struct
 */
static int sftp_attrsize(unsigned long flags)
{
    return (4 +                                 /* flags(4) */
            ((flags & LIBSSH2_SFTP_ATTR_SIZE) ? 8 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_UIDGID) ? 8 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) ? 4 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_ACMODTIME) ? 8 : 0));
    /* atime + mtime as u32 */
}

/* _libssh2_store_u64
 */
```

该函数的目的是在内存中存储一个 64 位无符号整数中的值，并将其存储到指针变量 **ptr 所指向的内存区域中。

函数的实现包括以下步骤：

1. 将输入的 64 位无符号整数中的值除以 64，取余数，得到一个 32 位的二进制数 msl。
2. 将 msl 分割成 8 位二进制数，并将其转换为相应的 ASCII 字符。
3. 将每个 ASCII 字符存储到输出缓冲区的对应位置。
4. 将输出缓冲区的 ASCII 字符数组向后移动 8 位，以便为下一个存储 ASCII 字符的位置留下空间。
5. 循环存储完所有 ASCII 字符后，将指针 **ptr 向后移动 8 位，以便为下一个存储 ASCII 字符的位置留下空间。

函数的实现有助于在需要时动态地存储和显示 64 位无符号整数中的值，而不会占用过多的内存空间。


```cpp
static void _libssh2_store_u64(unsigned char **ptr, libssh2_uint64_t value)
{
    uint32_t msl = (uint32_t)(value >> 32);
    unsigned char *buf = *ptr;

    buf[0] = (unsigned char)((msl >> 24) & 0xFF);
    buf[1] = (unsigned char)((msl >> 16) & 0xFF);
    buf[2] = (unsigned char)((msl >> 8)  & 0xFF);
    buf[3] = (unsigned char)( msl        & 0xFF);

    buf[4] = (unsigned char)((value >> 24) & 0xFF);
    buf[5] = (unsigned char)((value >> 16) & 0xFF);
    buf[6] = (unsigned char)((value >> 8)  & 0xFF);
    buf[7] = (unsigned char)( value        & 0xFF);

    *ptr += 8;
}

```

这段代码是一个C语言函数，名为find_zombie_request，它用于在给定的SSH2 Sftp对象中查找一个Zombie Request ID。

该函数接收两个参数：一个SSH2 Sftp对象和一个Zombie Request ID。首先，函数通过调用sftp->zombie_requests得到一个Zombie Request对象指针，然后使用libssh2_list_first()函数将其添加到Zombie Request列表中。

接下来，使用while循环遍历整个列表，直到找到一个与传入的Request ID相同的对象。如果找到该对象，循环就结束，并返回该对象的指针。否则，遍历整个列表，并将下一个对象添加到next指针中。

最后，该函数返回找到Zombie Request ID的指针，如果没有找到，则返回NULL。


```cpp
/*
 * Search list of zombied FXP_READ request IDs.
 *
 * Returns NULL if ID not in list.
 */
static struct sftp_zombie_requests *
find_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    struct sftp_zombie_requests *zombie =
        _libssh2_list_first(&sftp->zombie_requests);

    while(zombie) {
        if(zombie->request_id == request_id)
            break;
        else
            zombie = _libssh2_list_next(&zombie->node);
    }

    return zombie;
}

```

这段代码是一个用于remove_zombie_request函数，它属于LIBSSH2库。它的作用是移除SFTP传输过程中出现的无效或已删除的请求。

首先，它创建一个LIBSSH2_SESSION类型的变量session，用于与SFTP会话相关联。

然后，它调用find_zombie_request函数来获取输入的请求ID，该函数将在输入的zombie_requests结构体中查找与请求ID相应的zombie请求。

如果找到zombie请求，函数将使用_libssh2_debug函数打印一条日志信息，然后使用_libssh2_list_remove函数从zombie请求列表中删除该请求，最后使用LIBSSH2_FREE函数释放zombie请求内存。


```cpp
static void
remove_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    LIBSSH2_SESSION *session = sftp->channel->session;

    struct sftp_zombie_requests *zombie = find_zombie_request(sftp,
                                                              request_id);
    if(zombie) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Removing request ID %ld from the list of "
                       "zombie requests",
                       request_id);

        _libssh2_list_remove(&zombie->node);
        LIBSSH2_FREE(session, zombie);
    }
}

```

该函数是一个名为add_zombie_request的静态函数，属于LIBSSH2_SFTP类。其作用是接收一个SFTP会话对象sftp，以及一个请求ID，将请求ID标记为“zombie”并添加到sftp中定义的“zombie_requests”链表中。

具体来说，函数首先获取sftp会话对象中与传入的request_id相关的SFTP会话 session，然后使用session中实现的_libssh2_debug函数将request_id标为“zombie”并输出一条日志信息。

接着，使用libssh2_allocate函数从sftp会话对象中分配内存空间一个sftp_zombie_requests结构体，将其request_id字段设置为传入的request_id，然后使用_libssh2_list_add函数将sftp_zombie_requests结构体节点加入一个名为sftp_zombie_requests的链表中。

最后，函数返回一个名为LIBSSH2_ERROR_NONE的错误码，表示成功将请求ID标记为“zombie”并将其添加到sftp中定义的“zombie_requests”链表中。


```cpp
static int
add_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    LIBSSH2_SESSION *session = sftp->channel->session;

    struct sftp_zombie_requests *zombie;

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Marking request ID %ld as a zombie request", request_id);

    zombie = LIBSSH2_ALLOC(sftp->channel->session,
                           sizeof(struct sftp_zombie_requests));
    if(!zombie)
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "malloc fail for zombie request  ID");
    else {
        zombie->request_id = request_id;
        _libssh2_list_add(&sftp->zombie_requests, &zombie->node);
        return LIBSSH2_ERROR_NONE;
    }
}

```

This is a function definition for the `ssh2_send_packet` function of the SSH2 protocol. It determines whether the packet should be added to the `sftp->packets` list, and if it should be sent or not.

The function takes a single parameter, `packet`, which is a pointer to an `LIBSSH2_SFTP_PACKET` structure. It then checks whether the packet's data equal to the expected data for an SSH2 `SSH_FXP_DATA`, `SSH_FXP_NAME`, `SSH_FXP_ATTRS`, or `SSH_FXP_EXTENDED` packet, and如果是， it adds the packet to the `sftp->packets` list and returns `LIBSSH2_ERROR_NONE`.

If the packet's data does not match any of the expected packet formats, the function returns `LIBSSH2_ERROR_USERWARrant`, indicating that the packet is not formatted correctly. If the packet is a `SSH_FXP_STATUS` or `SSH_FXP_DATA` packet, the function adds the packet to the `sftp->packets` list and returns `LIBSSH2_ERROR_NONE`, indicating that the packet was received correctly. If the packet is a `SSH_FXP_EXTENDED` packet, the function checks if there was an error with the packet and, if there was, it returns `LIBSSH2_ERROR_NONE`. If the packet is a `SSH_FXP_NAME` or `SSH_FXP_ATTRS` packet, the function returns `LIBSSH2_ERROR_USERWARrant`, indicating that the packet is not formatted correctly.


```cpp
/*
 * sftp_packet_add
 *
 * Add a packet to the SFTP packet brigade
 */
static int
sftp_packet_add(LIBSSH2_SFTP *sftp, unsigned char *data,
                size_t data_len)
{
    LIBSSH2_SESSION *session = sftp->channel->session;
    LIBSSH2_SFTP_PACKET *packet;
    uint32_t request_id;

    if(data_len < 5) {
        return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
    }

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Received packet type %d (len %d)",
                   (int) data[0], data_len);

    /*
     * Experience shows that if we mess up EAGAIN handling somewhere or
     * otherwise get out of sync with the channel, this is where we first get
     * a wrong byte and if so we need to bail out at once to aid tracking the
     * problem better.
     */

    switch(data[0]) {
    case SSH_FXP_INIT:
    case SSH_FXP_VERSION:
    case SSH_FXP_OPEN:
    case SSH_FXP_CLOSE:
    case SSH_FXP_READ:
    case SSH_FXP_WRITE:
    case SSH_FXP_LSTAT:
    case SSH_FXP_FSTAT:
    case SSH_FXP_SETSTAT:
    case SSH_FXP_FSETSTAT:
    case SSH_FXP_OPENDIR:
    case SSH_FXP_READDIR:
    case SSH_FXP_REMOVE:
    case SSH_FXP_MKDIR:
    case SSH_FXP_RMDIR:
    case SSH_FXP_REALPATH:
    case SSH_FXP_STAT:
    case SSH_FXP_RENAME:
    case SSH_FXP_READLINK:
    case SSH_FXP_SYMLINK:
    case SSH_FXP_STATUS:
    case SSH_FXP_HANDLE:
    case SSH_FXP_DATA:
    case SSH_FXP_NAME:
    case SSH_FXP_ATTRS:
    case SSH_FXP_EXTENDED:
    case SSH_FXP_EXTENDED_REPLY:
        break;
    default:
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Out of sync with the world");
    }

    request_id = _libssh2_ntohu32(&data[1]);

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Received packet id %d",
                   request_id);

    /* Don't add the packet if it answers a request we've given up on. */
    if((data[0] == SSH_FXP_STATUS || data[0] == SSH_FXP_DATA)
       && find_zombie_request(sftp, request_id)) {

        /* If we get here, the file ended before the response arrived. We
           are no longer interested in the request so we discard it */

        LIBSSH2_FREE(session, data);

        remove_zombie_request(sftp, request_id);
        return LIBSSH2_ERROR_NONE;
    }

    packet = LIBSSH2_ALLOC(session, sizeof(LIBSSH2_SFTP_PACKET));
    if(!packet) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate datablock for SFTP packet");
    }

    packet->data = data;
    packet->data_len = data_len;
    packet->request_id = request_id;

    _libssh2_list_add(&sftp->packets, &packet->node);

    return LIBSSH2_ERROR_NONE;
}

```

This is a function definition for `sftp_write_direct(session, buffer, length)`. It is a part of an SSH2 negotiate protocol client library in C.

The function takes an SFT (Secure Copy Tool) session, a pointer to a buffer, and the length of the buffer. It is used to write data to the remote host through SSH2 negotiate protocol.

The function first initializes the session and buffer, and then reads the packet from the buffer. If the packet is larger than a buffer with a specified length, the function reads as much as possible of the packet and stores it in the buffer.

Then, the function checks the state of the packet. If it is in LIBSSH2_ERROR_EAGAIN state, it returns the error code and saves the packet in the buffer. If the packet state is LIBSSH2_NB_state_sent1, it stores the packet type in the buffer and returns it.

If the packet state is LIBSSH2_ERROR_SUCCESS, it stores the packet type in the buffer and returns the packet type. If the packet state is LIBSSH2_FIRMW_ERROR or LIBSSH2_CONNECTION_ERROR, it frees the packet and buffer and returns the error code.

If the function returns LIBSSH2_ERROR_EAGAIN or LIBSSH2_ERROR_SUCCESS, it passes the error code to the parent function or returns an error code according to the error code return value.


```cpp
/*
 * sftp_packet_read
 *
 * Frame an SFTP packet off the channel
 */
static int
sftp_packet_read(LIBSSH2_SFTP *sftp)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    unsigned char *packet = NULL;
    ssize_t rc;
    unsigned long recv_window;
    int packet_type;

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "recv packet");

    switch(sftp->packet_state) {
    case libssh2_NB_state_sent: /* EAGAIN from window adjusting */
        sftp->packet_state = libssh2_NB_state_idle;

        packet = sftp->partial_packet;
        goto window_adjust;

    case libssh2_NB_state_sent1: /* EAGAIN from channel read */
        sftp->packet_state = libssh2_NB_state_idle;

        packet = sftp->partial_packet;

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "partial read cont, len: %lu", sftp->partial_len);
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "partial read cont, already recvd: %lu",
                       sftp->partial_received);
        /* fall-through */
    default:
        if(!packet) {
            /* only do this if there's not already a packet buffer allocated
               to use */

            /* each packet starts with a 32 bit length field */
            rc = _libssh2_channel_read(channel, 0,
                                       (char *)&sftp->partial_size[
                                           sftp->partial_size_len],
                                       4 - sftp->partial_size_len);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
            else if(rc < 0)
                return _libssh2_error(session, rc, "channel read");

            sftp->partial_size_len += rc;

            if(4 != sftp->partial_size_len)
                /* we got a short read for the length part */
                return LIBSSH2_ERROR_EAGAIN;

            sftp->partial_len = _libssh2_ntohu32(sftp->partial_size);
            /* make sure we don't proceed if the packet size is unreasonably
               large */
            if(sftp->partial_len > LIBSSH2_SFTP_PACKET_MAXLEN) {
                libssh2_channel_flush(channel);
                sftp->partial_size_len = 0;
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_CHANNEL_PACKET_EXCEEDED,
                                      "SFTP packet too large");
            }

            if(sftp->partial_len == 0)
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_ALLOC,
                                      "Unable to allocate empty SFTP packet");

            _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                           "Data begin - Packet Length: %lu",
                           sftp->partial_len);
            packet = LIBSSH2_ALLOC(session, sftp->partial_len);
            if(!packet)
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "Unable to allocate SFTP packet");
            sftp->partial_size_len = 0;
            sftp->partial_received = 0; /* how much of the packet already
                                           received */
            sftp->partial_packet = packet;

          window_adjust:
            recv_window = libssh2_channel_window_read_ex(channel, NULL, NULL);

            if(sftp->partial_len > recv_window) {
                /* ask for twice the data amount we need at once */
                rc = _libssh2_channel_receive_window_adjust(channel,
                                                            sftp->partial_len
                                                            * 2,
                                                            1, NULL);
                /* store the state so that we continue with the correct
                   operation at next invoke */
                sftp->packet_state = (rc == LIBSSH2_ERROR_EAGAIN)?
                    libssh2_NB_state_sent:
                    libssh2_NB_state_idle;

                if(rc == LIBSSH2_ERROR_EAGAIN)
                    return rc;
            }
        }

        /* Read as much of the packet as we can */
        while(sftp->partial_len > sftp->partial_received) {
            rc = _libssh2_channel_read(channel, 0,
                                       (char *)&packet[sftp->partial_received],
                                       sftp->partial_len -
                                       sftp->partial_received);

            if(rc == LIBSSH2_ERROR_EAGAIN) {
                /*
                 * We received EAGAIN, save what we have and return EAGAIN to
                 * the caller. Set 'partial_packet' so that this function
                 * knows how to continue on the next invoke.
                 */
                sftp->packet_state = libssh2_NB_state_sent1;
                return rc;
            }
            else if(rc < 0) {
                LIBSSH2_FREE(session, packet);
                sftp->partial_packet = NULL;
                return _libssh2_error(session, rc,
                                      "Error waiting for SFTP packet");
            }
            sftp->partial_received += rc;
        }

        sftp->partial_packet = NULL;

        /* sftp_packet_add takes ownership of the packet and might free it
           so we take a copy of the packet type before we call it. */
        packet_type = packet[0];
        rc = sftp_packet_add(sftp, packet, sftp->partial_len);
        if(rc) {
            LIBSSH2_FREE(session, packet);
            return rc;
        }
        else {
            return packet_type;
        }
    }
    /* WON'T REACH */
}
```

这段代码的作用是移除包片列表（packet_list）中所有未完成的包，并清空包片队列中的相应包，使得后续发送的数据能够正常进入SFTP包队。

具体来说，代码首先遍历包片列表，对每一个包片，获取其相关信息（如请求ID，数据，数据长度等），然后通过调用sftp_packet_ask函数，获取该包片的实际数据和下一个包片的指针。接着，如果前一个包片已经发送，则跳过这个包片；否则，标记为 zombie 状态，以便后续处理。如果这个包片已经发送并且还没有被标记为 zombie，会在包片队列中将其删除，并释放其内存。最后，将下一个包片加入包片队列。

整个函数的实现较为简单，主要目的是为了清空包片列表中的未完成包，以便后续数据正常进入包队。


```cpp
/*
 * sftp_packetlist_flush
 *
 * Remove all pending packets in the packet_list and the corresponding one(s)
 * in the SFTP packet brigade.
 */
static void sftp_packetlist_flush(LIBSSH2_SFTP_HANDLE *handle)
{
    struct sftp_pipeline_chunk *chunk;
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_SESSION *session = sftp->channel->session;

    /* remove pending packets, if any */
    chunk = _libssh2_list_first(&handle->packet_list);
    while(chunk) {
        unsigned char *data;
        size_t data_len;
        int rc;
        struct sftp_pipeline_chunk *next = _libssh2_list_next(&chunk->node);

        rc = sftp_packet_ask(sftp, SSH_FXP_STATUS,
                             chunk->request_id, &data, &data_len);
        if(rc)
            rc = sftp_packet_ask(sftp, SSH_FXP_DATA,
                                 chunk->request_id, &data, &data_len);

        if(!rc)
            /* we found a packet, free it */
            LIBSSH2_FREE(session, data);
        else if(chunk->sent)
            /* there was no incoming packet for this request, mark this
               request as a zombie if it ever sent the request */
            add_zombie_request(sftp, chunk->request_id);

        _libssh2_list_remove(&chunk->node);
        LIBSSH2_FREE(session, chunk);
        chunk = next;
    }
}


```

这段代码是一个名为 `sftp_packet_ask()` 的函数，它用于检查是否有一个可匹配的 SFTP 数据包。该函数接收三个参数：`LIBSSH2_SFTP` 表示 SFTP 句柄，`unsigned char packet_type` 表示要检查的 SFTP 数据包类型，`uint32_t request_id` 表示请求 ID，`unsigned char **data` 表示要检查的数据包数据，`size_t *data_len` 表示数据包数据的缓存大小。

函数首先检查是否有一个匹配的 SFTP 数据包。如果找到了匹配的数据包，就返回 0，否则继续查找。如果找不到匹配的 SFTP 数据包，函数返回 -1。


```cpp
/*
 * sftp_packet_ask()
 *
 * Checks if there's a matching SFTP packet available.
 */
static int
sftp_packet_ask(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                uint32_t request_id, unsigned char **data,
                size_t *data_len)
{
    LIBSSH2_SESSION *session = sftp->channel->session;
    LIBSSH2_SFTP_PACKET *packet = _libssh2_list_first(&sftp->packets);

    if(!packet)
        return -1;

    /* Special consideration when getting VERSION packet */

    while(packet) {
        if((packet->data[0] == packet_type) &&
           ((packet_type == SSH_FXP_VERSION) ||
            (packet->request_id == request_id))) {

            /* Match! Fetch the data */
            *data = packet->data;
            *data_len = packet->data_len;

            /* unlink and free this struct */
            _libssh2_list_remove(&packet->node);
            LIBSSH2_FREE(session, packet);

            return 0;
        }
        /* check next struct in the list */
        packet = _libssh2_list_next(&packet->node);
    }
    return -1;
}

```

This is a function that handles the receiving of a packet from the SFTP server
on a given SFTP session. It takes as input the packet received from the server, of type sftp\_packet\_t, and an optional buffer of size required\_size.
It first checks if the packet is valid and has the required length, and if not returns LIBSSH2\_ERROR\_BAD\_USE.
Then it calls the sftp\_packet\_ask() function to receive the packet, passing in the required\_size if passed.
It then checks if the received packet is valid and has the required length, and if not returns LIBSSH2\_ERROR\_BUFFER\_TOO\_SMALL.
It continues to do so until the socket is closed, and returns LIBSSH2\_ERROR\_SOCKET\_DISCONNECT if the socket is closed正常.


```cpp
/* sftp_packet_require
 * A la libssh2_packet_require
 */
static int
sftp_packet_require(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                    uint32_t request_id, unsigned char **data,
                    size_t *data_len, size_t required_size)
{
    LIBSSH2_SESSION *session = sftp->channel->session;
    int rc;

    if(data == NULL || data_len == NULL || required_size == 0) {
        return LIBSSH2_ERROR_BAD_USE;
    }

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Requiring packet %d id %ld",
                   (int) packet_type, request_id);

    if(sftp_packet_ask(sftp, packet_type, request_id, data, data_len) == 0) {
        /* The right packet was available in the packet brigade */
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Got %d",
                       (int) packet_type);

        if (*data_len < required_size) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }

        return LIBSSH2_ERROR_NONE;
    }

    while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        rc = sftp_packet_read(sftp);
        if(rc < 0)
            return rc;

        /* data was read, check the queue again */
        if(!sftp_packet_ask(sftp, packet_type, request_id, data, data_len)) {
            /* The right packet was available in the packet brigade */
            _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Got %d",
                           (int) packet_type);

            if (*data_len < required_size) {
                return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
            }

            return LIBSSH2_ERROR_NONE;
        }
    }

    /* Only reached if the socket died */
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}

```

This is a function in the Sftp library that is responsible for attempting to establish a connection to a remote host using the SSH protocol and retrieving data from the host. It is called `libssh2_connect_remote`.

The function takes in a number of parameters, including `sftp`, `valid_responses`, `num_valid_responses`, and `required_size`. The `sftp` parameter is an instance of the `Sftp` class, `valid_responses` is an array of `SftpValidResponse` objects representing the valid responses to a particular Sftp command, `num_valid_responses` is the number of valid responses in `valid_responses`, and `required_size` is the expected size of a `SftpResponse` object.

The function starts by attempting to connect to the remote host using the specified credentials and IP address. If the connection is successful, the function retrieves the required data from the host and returns it in an error code. If the connection fails, the function returns a different error code. If the connection is successful, the function returns LIBSSH2_ERROR_NONE. If the connection is lost or the required data cannot be retrieved, the function returns LIBSSH2_ERROR_EAGAIN or LIBSSH2_ERROR_BUFFER_TOO_SMALL.


```cpp
/* sftp_packet_requirev
 * Require one of N possible responses
 */
static int
sftp_packet_requirev(LIBSSH2_SFTP *sftp, int num_valid_responses,
                     const unsigned char *valid_responses,
                     uint32_t request_id, unsigned char **data,
                     size_t *data_len, size_t required_size)
{
    int i;
    int rc;

    if(data == NULL || data_len == NULL || required_size == 0) {
        return LIBSSH2_ERROR_BAD_USE;
    }

    /* If no timeout is active, start a new one */
    if(sftp->requirev_start == 0)
        sftp->requirev_start = time(NULL);

    while(sftp->channel->session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        for(i = 0; i < num_valid_responses; i++) {
            if(sftp_packet_ask(sftp, valid_responses[i], request_id,
                                data, data_len) == 0) {
                /*
                 * Set to zero before all returns to say
                 * the timeout is not active
                 */
                sftp->requirev_start = 0;

                if (*data_len < required_size) {
                    return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                }

                return LIBSSH2_ERROR_NONE;
            }
        }

        rc = sftp_packet_read(sftp);
        if((rc < 0) && (rc != LIBSSH2_ERROR_EAGAIN)) {
            sftp->requirev_start = 0;
            return rc;
        }
        else if(rc <= 0) {
            /* prevent busy-looping */
            long left =
                LIBSSH2_READ_TIMEOUT -
                (long)(time(NULL) - sftp->requirev_start);

            if(left <= 0) {
                sftp->requirev_start = 0;
                return LIBSSH2_ERROR_TIMEOUT;
            }
            else if(rc == LIBSSH2_ERROR_EAGAIN) {
                return rc;
            }
        }
    }

    sftp->requirev_start = 0;

    /* Only reached if the socket died */
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}

```


This function takes an array of SFTP attribute flags and converts them into a binary representation that can be stored in an SFTP block. It is used by the SFTP client to populate the SFTP block with the attribute values associated with the given SFTP attribute flags.

The function takes into account the different attribute flags and any relevant SFTP attribute size. It stores the attribute values in the provided array of bytes, and returns the number of bytes elapsed in the conversion process.


```cpp
/* sftp_attr2bin
 * Populate attributes into an SFTP block
 */
static ssize_t
sftp_attr2bin(unsigned char *p, const LIBSSH2_SFTP_ATTRIBUTES * attrs)
{
    unsigned char *s = p;
    uint32_t flag_mask =
        LIBSSH2_SFTP_ATTR_SIZE | LIBSSH2_SFTP_ATTR_UIDGID |
        LIBSSH2_SFTP_ATTR_PERMISSIONS | LIBSSH2_SFTP_ATTR_ACMODTIME;

    /* TODO: When we add SFTP4+ functionality flag_mask can get additional
       bits */

    if(!attrs) {
        _libssh2_htonu32(s, 0);
        return 4;
    }

    _libssh2_store_u32(&s, attrs->flags & flag_mask);

    if(attrs->flags & LIBSSH2_SFTP_ATTR_SIZE) {
        _libssh2_store_u64(&s, attrs->filesize);
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_UIDGID) {
        _libssh2_store_u32(&s, attrs->uid);
        _libssh2_store_u32(&s, attrs->gid);
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) {
        _libssh2_store_u32(&s, attrs->permissions);
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_ACMODTIME) {
        _libssh2_store_u32(&s, attrs->atime);
        _libssh2_store_u32(&s, attrs->mtime);
    }

    return (s - p);
}

```



This is a Java method that serializes an SSH file attribute to an SSH file attribute structure, which is then stored in the given `FileAttrs` object. The method takes as input an `FileAttrs` object, as well as a pointer to an input buffer (`buf`) that will be used to store the file attribute data.

The method first sets the `flags` attribute of the `FileAttrs` object to the desired file attribute flags, which are then stored in the `flags` field.

If the file attribute data is going to be stored in a larger input buffer (`attrs->filesize` is larger than 64), the method retrieves the file size from the input buffer and sets the `filesize` attribute of the `FileAttrs` object.

The method then checks the file attribute data and, if the `flags` and `filesize` attributes are not present, returns an error.

The method then sets the `uid` and `gid` attributes of the `FileAttrs` object based on the file's ownership and group membership.

Finally, the method sets the `atime` and `mtime` attributes of the `FileAttrs` object based on the file's creation and modification times.

The method returns the number of bytes that were allocated for the `FileAttrs` object, including the `flags`, `attrs->filesize`, `uid`, `gid`, `atime`, and `mtime` attributes.


```cpp
/* sftp_bin2attr
 */
static int
sftp_bin2attr(LIBSSH2_SFTP_ATTRIBUTES *attrs, const unsigned char *p,
              size_t data_len)
{
    struct string_buf buf;
    uint32_t flags = 0;
    buf.data = (unsigned char *)p;
    buf.dataptr = buf.data;
    buf.len = data_len;

    if(_libssh2_get_u32(&buf, &flags) != 0) {
        return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
    }
    attrs->flags = flags;

    if(attrs->flags & LIBSSH2_SFTP_ATTR_SIZE) {
        if(_libssh2_get_u64(&buf, &(attrs->filesize)) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_UIDGID) {
        uint32_t uid = 0;
        uint32_t gid = 0;
        if(_libssh2_get_u32(&buf, &uid) != 0 ||
           _libssh2_get_u32(&buf, &gid) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        attrs->uid = uid;
        attrs->gid = gid;
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) {
        uint32_t permissions;
        if(_libssh2_get_u32(&buf, &permissions) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        attrs->permissions = permissions;
    }

    if(attrs->flags & LIBSSH2_SFTP_ATTR_ACMODTIME) {
        uint32_t atime;
        uint32_t mtime;
        if(_libssh2_get_u32(&buf, &atime) != 0 ||
           _libssh2_get_u32(&buf, &mtime) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        attrs->atime = atime;
        attrs->mtime = mtime;
    }

    return (buf.dataptr - buf.data);
}

```

这段代码定义了一个名为libssh2_sftp_dtor的函数，它是一个SFTP（Secure File Transfer Protocol）实现。这个函数在SFTP通道关闭时执行。

具体来说，当SFTP通道关闭时，函数会关闭与通道相关的资源，包括关闭与通道抽象相关的套接字，清除与SFTP数据传输相关的缓冲区，最后释放与SFTP数据传输相关的内存。

通过调用这个函数，可以确保在SFTP通道关闭时，所有与SFTP数据传输相关的资源都被正确关闭，以避免潜在的数据泄漏或其他安全问题。


```cpp
/* ************
 * SFTP API *
 ************ */

LIBSSH2_CHANNEL_CLOSE_FUNC(libssh2_sftp_dtor);

/* libssh2_sftp_dtor
 * Shutdown an SFTP stream when the channel closes
 */
LIBSSH2_CHANNEL_CLOSE_FUNC(libssh2_sftp_dtor)
{
    LIBSSH2_SFTP *sftp = (LIBSSH2_SFTP *) (*channel_abstract);

    (void) session_abstract;
    (void) channel;

    /* Free the partial packet storage for sftp_packet_read */
    if(sftp->partial_packet) {
        LIBSSH2_FREE(session, sftp->partial_packet);
    }

    /* Free the packet storage for _libssh2_sftp_packet_readdir */
    if(sftp->readdir_packet) {
        LIBSSH2_FREE(session, sftp->readdir_packet);
    }

    LIBSSH2_FREE(session, sftp);
}

```

This function appears to be a part of an SSH2 Sftp client, and it's responsible for initializing the SFTP session and setting up a file to be extracted.

It takes two arguments, one of which is a pointer to an SFTP data structure, the other of which is a pointer to an optional data buffer.

First, it checks if the data buffer is too small, and if it is, it returns an error and halts the SFTP client.

Then, it extracts the file extension name from the SFTP data structure and wraps it in a data buffer.

Finally, it closes the SFTP channel, sets the SFTP data structure pointer to `NULL`, and sets the initial state of the SFTP session to `libssh2_NB_state_idle`.

It also sets a pointer to a function that will be called when the channel is closed to `sftp_handle->channel->abstract`, and a function pointer that points to a dtor function to be called when the SFTP channel is closed to `libssh2_sftp_dtor`.


```cpp
/*
 * sftp_init
 *
 * Startup an SFTP session
 */
static LIBSSH2_SFTP *sftp_init(LIBSSH2_SESSION *session)
{
    unsigned char *data;
    size_t data_len;
    ssize_t rc;
    LIBSSH2_SFTP *sftp_handle;
    struct string_buf buf;
    unsigned char *endp;

    if(session->sftpInit_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Initializing SFTP subsystem");

        /*
         * The 'sftpInit_sftp' and 'sftpInit_channel' struct fields within the
         * session struct are only to be used during the setup phase. As soon
         * as the SFTP session is created they are cleared and can thus be
         * re-used again to allow any amount of SFTP handles per sessions.
         *
         * Note that you MUST NOT try to call libssh2_sftp_init() again to get
         * another handle until the previous call has finished and either
         * successfully made a handle or failed and returned error (not
         * including *EAGAIN).
         */

        assert(session->sftpInit_sftp == NULL);
        session->sftpInit_sftp = NULL;
        session->sftpInit_state = libssh2_NB_state_created;
    }

    sftp_handle = session->sftpInit_sftp;

    if(session->sftpInit_state == libssh2_NB_state_created) {
        session->sftpInit_channel =
            _libssh2_channel_open(session, "session", sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL, 0);
        if(!session->sftpInit_channel) {
            if(libssh2_session_last_errno(session) == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Unable to startup channel");
                session->sftpInit_state = libssh2_NB_state_idle;
            }
            return NULL;
        }

        session->sftpInit_state = libssh2_NB_state_sent;
    }

    if(session->sftpInit_state == libssh2_NB_state_sent) {
        int ret = _libssh2_channel_process_startup(session->sftpInit_channel,
                                                   "subsystem",
                                                   sizeof("subsystem") - 1,
                                                   "sftp",
                                                   strlen("sftp"));
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block to request SFTP subsystem");
            return NULL;
        }
        else if(ret) {
            _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                           "Unable to request SFTP subsystem");
            goto sftp_init_error;
        }

        session->sftpInit_state = libssh2_NB_state_sent1;
    }

    if(session->sftpInit_state == libssh2_NB_state_sent1) {
        rc = _libssh2_channel_extended_data(session->sftpInit_channel,
                                         LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting handle extended data");
            return NULL;
        }

        sftp_handle =
            session->sftpInit_sftp =
            LIBSSH2_CALLOC(session, sizeof(LIBSSH2_SFTP));
        if(!sftp_handle) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a new SFTP structure");
            goto sftp_init_error;
        }
        sftp_handle->channel = session->sftpInit_channel;
        sftp_handle->request_id = 0;

        _libssh2_htonu32(session->sftpInit_buffer, 5);
        session->sftpInit_buffer[4] = SSH_FXP_INIT;
        _libssh2_htonu32(session->sftpInit_buffer + 5, LIBSSH2_SFTP_VERSION);
        session->sftpInit_sent = 0; /* nothing's sent yet */

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Sending FXP_INIT packet advertising "
                       "version %d support",
                       (int) LIBSSH2_SFTP_VERSION);

        session->sftpInit_state = libssh2_NB_state_sent2;
    }

    if(session->sftpInit_state == libssh2_NB_state_sent2) {
        /* sent off what's left of the init buffer to send */
        rc = _libssh2_channel_write(session->sftpInit_channel, 0,
                                    session->sftpInit_buffer +
                                    session->sftpInit_sent,
                                    9 - session->sftpInit_sent);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending SSH_FXP_INIT");
            return NULL;
        }
        else if(rc < 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send SSH_FXP_INIT");
            goto sftp_init_error;
        }
        else {
            /* add up the number of bytes sent */
            session->sftpInit_sent += rc;

            if(session->sftpInit_sent == 9)
                /* move on */
                session->sftpInit_state = libssh2_NB_state_sent3;

            /* if less than 9, we remain in this state to send more later on */
        }
    }

    rc = sftp_packet_require(sftp_handle, SSH_FXP_VERSION,
                             0, &data, &data_len, 5);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                       "Would block receiving SSH_FXP_VERSION");
        return NULL;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                       "Invalid SSH_FXP_VERSION response");
        goto sftp_init_error;
    }
    else if(rc) {
        _libssh2_error(session, rc,
                       "Timeout waiting for response from SFTP subsystem");
        goto sftp_init_error;
    }

    buf.data = data;
    buf.dataptr = buf.data + 1;
    buf.len = data_len;
    endp = &buf.data[data_len];

    if(_libssh2_get_u32(&buf, &(sftp_handle->version)) != 0) {
        LIBSSH2_FREE(session, data);
        rc = LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        goto sftp_init_error;
    }

    if(sftp_handle->version > LIBSSH2_SFTP_VERSION) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Truncating remote SFTP version from %lu",
                       sftp_handle->version);
        sftp_handle->version = LIBSSH2_SFTP_VERSION;
    }
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Enabling SFTP version %lu compatibility",
                   sftp_handle->version);
    while(buf.dataptr < endp) {
        unsigned char *extname, *extdata;

        if(_libssh2_get_string(&buf, &extname, NULL)) {
            LIBSSH2_FREE(session, data);
            _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Data too short when extracting extname");
            goto sftp_init_error;
        }

        if(_libssh2_get_string(&buf, &extdata, NULL)) {
            LIBSSH2_FREE(session, data);
            _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Data too short when extracting extdata");
            goto sftp_init_error;
        }
    }
    LIBSSH2_FREE(session, data);

    /* Make sure that when the channel gets closed, the SFTP service is shut
       down too */
    sftp_handle->channel->abstract = sftp_handle;
    sftp_handle->channel->close_cb = libssh2_sftp_dtor;

    session->sftpInit_state = libssh2_NB_state_idle;

    /* clear the sftp and channel pointers in this session struct now */
    session->sftpInit_sftp = NULL;
    session->sftpInit_channel = NULL;

    _libssh2_list_init(&sftp_handle->sftp_handles);

    return sftp_handle;

  sftp_init_error:
    while(_libssh2_channel_free(session->sftpInit_channel) ==
           LIBSSH2_ERROR_EAGAIN);
    session->sftpInit_channel = NULL;
    if(session->sftpInit_sftp) {
        LIBSSH2_FREE(session, session->sftpInit_sftp);
        session->sftpInit_sftp = NULL;
    }
    session->sftpInit_state = libssh2_NB_state_idle;
    return NULL;
}

```

这段代码是一个用于初始化安全外壳（SSH）SFTP文件的库函数。它接受一个SSH2会话对象（LIBSSH2_SESSION）作为参数，并在尝试创建一个SFTP文件会话时进行初始化。以下是代码的功能解释：

1. 检查输入参数是否为空：如果为空，函数返回NULL，表示无法创建SFTP会话。

2. 检查输入参数的会话状态是否与预定义的LIBSSH2_STATE_AUTHENTICATED相等：如果不相等，函数会输出错误并返回NULL，表示未经授权的连接。

3.调用sftp_init函数，初始化SFTP会话：sftp_init函数将在会话成功创建后返回一个指向SFTP文件会话对象的指针。

4. 返回上面执行成功的指针：现在，经过初始化并成功创建SFTP会话后，你可以使用它来执行SFTP文件操作。


```cpp
/*
 * libssh2_sftp_init
 *
 * Startup an SFTP session
 */
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session)
{
    LIBSSH2_SFTP *ptr;

    if(!session)
        return NULL;

    if(!(session->state & LIBSSH2_STATE_AUTHENTICATED)) {
        _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                       "session not authenticated yet");
        return NULL;
    }

    BLOCK_ADJUST_ERRNO(ptr, session, sftp_init(session));
    return ptr;
}

```

This function appears to handle the handshake with an SFTP client, and it is called by the main function. It takes control over several functions that are responsible for setting up and tearing down the SFTP connection, as well as handling the SFTP file system.

The function first calls the SFTP client's `connect` function, passing in the required credentials. It then calls the client's `write_desc` function, passing in a buffer and telling it not to print a directory. It also gets the SFTP client's `allow_other_github` option and sets it to `1`.

After that, the function starts monitoring the SFTP client for the `connect_response` event. When it receives one, it calls the `parse_server_public_key` function, passing in the client's public key and telling it to allow new server keys.

The function then enters a loop where it waits for the SFTP client to establish a timeout. If the client times out, it calls the `handle_deadline` function, which杀死并 remove the SFTP client.

The function also handles the SFTP client's `new_file` and `remove_file` functions. When a new file is created, it prints the `七九七九七` message to the console, and when a file is removed, it prints the `八八八八八` message to the console.

The function also prints a message to the console when the SFTP client releases the SFTP connection, and when the connection is completely closed, it calls the `print_status` function, which prints the SFTP client's status to the console.


```cpp
/*
 * sftp_shutdown
 *
 * Shuts down the SFTP subsystem
 */
static int
sftp_shutdown(LIBSSH2_SFTP *sftp)
{
    int rc;
    LIBSSH2_SESSION *session = sftp->channel->session;
    /*
     * Make sure all memory used in the state variables are free
     */
    if(sftp->partial_packet) {
        LIBSSH2_FREE(session, sftp->partial_packet);
        sftp->partial_packet = NULL;
    }
    if(sftp->open_packet) {
        LIBSSH2_FREE(session, sftp->open_packet);
        sftp->open_packet = NULL;
    }
    if(sftp->readdir_packet) {
        LIBSSH2_FREE(session, sftp->readdir_packet);
        sftp->readdir_packet = NULL;
    }
    if(sftp->fstat_packet) {
        LIBSSH2_FREE(session, sftp->fstat_packet);
        sftp->fstat_packet = NULL;
    }
    if(sftp->unlink_packet) {
        LIBSSH2_FREE(session, sftp->unlink_packet);
        sftp->unlink_packet = NULL;
    }
    if(sftp->rename_packet) {
        LIBSSH2_FREE(session, sftp->rename_packet);
        sftp->rename_packet = NULL;
    }
    if(sftp->fstatvfs_packet) {
        LIBSSH2_FREE(session, sftp->fstatvfs_packet);
        sftp->fstatvfs_packet = NULL;
    }
    if(sftp->statvfs_packet) {
        LIBSSH2_FREE(session, sftp->statvfs_packet);
        sftp->statvfs_packet = NULL;
    }
    if(sftp->mkdir_packet) {
        LIBSSH2_FREE(session, sftp->mkdir_packet);
        sftp->mkdir_packet = NULL;
    }
    if(sftp->rmdir_packet) {
        LIBSSH2_FREE(session, sftp->rmdir_packet);
        sftp->rmdir_packet = NULL;
    }
    if(sftp->stat_packet) {
        LIBSSH2_FREE(session, sftp->stat_packet);
        sftp->stat_packet = NULL;
    }
    if(sftp->symlink_packet) {
        LIBSSH2_FREE(session, sftp->symlink_packet);
        sftp->symlink_packet = NULL;
    }
    if(sftp->fsync_packet) {
        LIBSSH2_FREE(session, sftp->fsync_packet);
        sftp->fsync_packet = NULL;
    }

    sftp_packet_flush(sftp);

    /* TODO: We should consider walking over the sftp_handles list and kill
     * any remaining sftp handles ... */

    rc = _libssh2_channel_free(sftp->channel);

    return rc;
}

```

这段代码是一个用于关闭 Sftp 子系统的库函数，名为 libssh2_sftp_shutdown。它的作用是接收一个 SFTP 对象，然后执行关闭操作并返回结果。如果输入的 SFTP 对象为空或无法访问，则会返回 LIBSSH2_ERROR_BAD_USE。

具体来说，代码首先检查输入的 SFTP 对象是否为空，如果是，则直接返回 LIBSSH2_ERROR_BAD_USE。然后代码调用了一个名为 sftp_shutdown 的函数，该函数接收一个 SFTP 对象和一个通道对象，并执行关闭操作。最后，代码使用 BLOCK_ADJUST 函数来调整返回值，以便正确处理返回状态码。


```cpp
/* libssh2_sftp_shutdown
 * Shutsdown the SFTP subsystem
 */
LIBSSH2_API int
libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session, sftp_shutdown(sftp));
    return rc;
}

/* *******************************
 * SFTP File and Directory Ops *
 ******************************* */

```

This is a function that opens an SFTP file handle and returns it. It takes as input a pointer to a SFTP data block (`data`), and as output it returns a pointer to the newly allocated memory block that contains the SFTP file handle. It checks that the passed input `data` is not too small and then it allocates memory for the SFTP file handle and returns it.

The function also returns a pointer to the parent struct that holds the SFTP data block.

It is important to note that the function assumes that the passed `data` contains the complete SFTP data and that it is properly formatted. It also assumes that the SFTP file handle is the first element in the `data` block.


```cpp
/* sftp_open
 */
static LIBSSH2_SFTP_HANDLE *
sftp_open(LIBSSH2_SFTP *sftp, const char *filename,
          size_t filename_len, uint32_t flags, long mode,
          int open_type)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_SFTP_HANDLE *fp;
    LIBSSH2_SFTP_ATTRIBUTES attrs = {
        LIBSSH2_SFTP_ATTR_PERMISSIONS, 0, 0, 0, 0, 0, 0
    };
    unsigned char *s;
    ssize_t rc;
    int open_file = (open_type == LIBSSH2_SFTP_OPENFILE)?1:0;

    if(sftp->open_state == libssh2_NB_state_idle) {
        /* packet_len(4) + packet_type(1) + request_id(4) + filename_len(4) +
           flags(4) */
        sftp->open_packet_len = filename_len + 13 +
            (open_file? (4 +
                         sftp_attrsize(LIBSSH2_SFTP_ATTR_PERMISSIONS)) : 0);

        /* surprise! this starts out with nothing sent */
        sftp->open_packet_sent = 0;
        s = sftp->open_packet = LIBSSH2_ALLOC(session, sftp->open_packet_len);
        if(!sftp->open_packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for FXP_OPEN or "
                           "FXP_OPENDIR packet");
            return NULL;
        }
        /* Filetype in SFTP 3 and earlier */
        attrs.permissions = mode |
            (open_file ? LIBSSH2_SFTP_ATTR_PFILETYPE_FILE :
             LIBSSH2_SFTP_ATTR_PFILETYPE_DIR);

        _libssh2_store_u32(&s, sftp->open_packet_len - 4);
        *(s++) = open_file? SSH_FXP_OPEN : SSH_FXP_OPENDIR;
        sftp->open_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->open_request_id);
        _libssh2_store_str(&s, filename, filename_len);

        if(open_file) {
            _libssh2_store_u32(&s, flags);
            s += sftp_attr2bin(s, &attrs);
        }

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Sending %s open request",
                       open_file? "file" : "directory");

        sftp->open_state = libssh2_NB_state_created;
    }

    if(sftp->open_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->open_packet+
                                    sftp->open_packet_sent,
                                    sftp->open_packet_len -
                                    sftp->open_packet_sent);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending FXP_OPEN or "
                           "FXP_OPENDIR command");
            return NULL;
        }
        else if(rc < 0) {
            _libssh2_error(session, rc, "Unable to send FXP_OPEN*");
            LIBSSH2_FREE(session, sftp->open_packet);
            sftp->open_packet = NULL;
            sftp->open_state = libssh2_NB_state_idle;
            return NULL;
        }

        /* bump the sent counter and remain in this state until the whole
           data is off */
        sftp->open_packet_sent += rc;

        if(sftp->open_packet_len == sftp->open_packet_sent) {
            LIBSSH2_FREE(session, sftp->open_packet);
            sftp->open_packet = NULL;

            sftp->open_state = libssh2_NB_state_sent;
        }
    }

    if(sftp->open_state == libssh2_NB_state_sent) {
        size_t data_len;
        unsigned char *data;
        static const unsigned char fopen_responses[2] =
            { SSH_FXP_HANDLE, SSH_FXP_STATUS };
        rc = sftp_packet_requirev(sftp, 2, fopen_responses,
                                  sftp->open_request_id, &data,
                                  &data_len, 1);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block waiting for status message");
            return NULL;
        }
        else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
            if(data_len > 0) {
                LIBSSH2_FREE(session, data);
            }
            _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                           "Response too small");
            return NULL;
        }
        sftp->open_state = libssh2_NB_state_idle;
        if(rc) {
            _libssh2_error(session, rc, "Timeout waiting for status message");
            return NULL;
        }

        /* OPEN can basically get STATUS or HANDLE back, where HANDLE implies
           a fine response while STATUS means error. It seems though that at
           times we get an SSH_FX_OK back in a STATUS, followed the "real"
           HANDLE so we need to properly deal with that. */
        if(data[0] == SSH_FXP_STATUS) {
            int badness = 1;

            if(data_len < 9) {
                _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                               "Too small FXP_STATUS");
                LIBSSH2_FREE(session, data);
                return NULL;
            }

            sftp->last_errno = _libssh2_ntohu32(data + 5);

            if(LIBSSH2_FX_OK == sftp->last_errno) {
                _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                               "got HANDLE FXOK!");

                LIBSSH2_FREE(session, data);

                /* silly situation, but check for a HANDLE */
                rc = sftp_packet_require(sftp, SSH_FXP_HANDLE,
                                         sftp->open_request_id, &data,
                                         &data_len, 10);
                if(rc == LIBSSH2_ERROR_EAGAIN) {
                    /* go back to sent state and wait for something else */
                    sftp->open_state = libssh2_NB_state_sent;
                    return NULL;
                }
                else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
                    if(data_len > 0) {
                        LIBSSH2_FREE(session, data);
                    }
                    _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                   "Too small FXP_HANDLE");
                    return NULL;
                }
                else if(!rc)
                    /* we got the handle so this is not a bad situation */
                    badness = 0;
            }

            if(badness) {
                _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                               "Failed opening remote file");
                _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                               "got FXP_STATUS %d",
                               sftp->last_errno);
                LIBSSH2_FREE(session, data);
                return NULL;
            }
        }

        if(data_len < 10) {
            _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                           "Too small FXP_HANDLE");
            LIBSSH2_FREE(session, data);
            return NULL;
        }

        fp = LIBSSH2_CALLOC(session, sizeof(LIBSSH2_SFTP_HANDLE));
        if(!fp) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate new SFTP handle structure");
            LIBSSH2_FREE(session, data);
            return NULL;
        }
        fp->handle_type = open_file ? LIBSSH2_SFTP_HANDLE_FILE :
            LIBSSH2_SFTP_HANDLE_DIR;

        fp->handle_len = _libssh2_ntohu32(data + 5);
        if(fp->handle_len > SFTP_HANDLE_MAXLEN)
            /* SFTP doesn't allow handles longer than 256 characters */
            fp->handle_len = SFTP_HANDLE_MAXLEN;

        if(fp->handle_len > (data_len - 9))
            /* do not reach beyond the end of the data we got */
            fp->handle_len = data_len - 9;

        memcpy(fp->handle, data + 9, fp->handle_len);

        LIBSSH2_FREE(session, data);

        /* add this file handle to the list kept in the sftp session */
        _libssh2_list_add(&sftp->sftp_handles, &fp->node);

        fp->sftp = sftp; /* point to the parent struct */

        fp->u.file.offset = 0;
        fp->u.file.offset_sent = 0;

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Open command successful");
        return fp;
    }
    return NULL;
}

```

这段代码定义了一个名为 `libssh2_sftp_open_ex` 的函数，它是 `libssh2_sftp_open` 的扩展版本，用于在 SFTP 设备上打开文件。函数接受五个参数：

1. `sftp`：一个 SFTP 对象，可以是 `libssh2_sftp` 类型的对象。
2. `filename`：要打开的文件名，包括文件类型和文件扩展名。
3. `filename_len`：文件名长度，以字节为单位。
4. `flags`：打开文件的权限，可以是任何有效的 SFTP 标志，如 `LIBSSH2_SFTP_FLAG_OPEN_EX`。
5. `mode`：文件模式，可以是任何有效的文件模式，如 `LIBSSH2_SFTP_MODE_RWX`。
6. `open_type`：打开文件类型，可以是 `LIBSSH2_SFTP_OPEN_EX` 或 `LIBSSH2_SFTP_OPEN_APPEND`。

函数首先检查传入的 `sftp` 是否为空，然后执行打开文件的操作，并返回打开的 SFTP 对象的 handle。

如果 `sftp` 为空，该函数将返回 NULL，因为没有任何东西可以调用 `libssh2_sftp_open` 函数。


```cpp
/* libssh2_sftp_open_ex
 */
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_open_ex(LIBSSH2_SFTP *sftp, const char *filename,
                     unsigned int filename_len, unsigned long flags, long mode,
                     int open_type)
{
    LIBSSH2_SFTP_HANDLE *hnd;

    if(!sftp)
        return NULL;

    BLOCK_ADJUST_ERRNO(hnd, sftp->channel->session,
                       sftp_open(sftp, filename, filename_len, flags, mode,
                                 open_type));
    return hnd;
}

```

This function appears to be reading data from an SFTP file and returning it in chunks. It is doing this by first receiving the FXP_DATA packet from the SFTP server, extracting the data from it, and then processing it. It is then copying this data to the buffer, processing it, and copying it back to the buffer. If the data is larger than the buffer, it creates a new chunk to hold the data and copies it in.

It is important to note that the function may raise an error if it receives an invalid SFTP packet or if it cannot allocate enough space in the buffer. It is also important to check that the buffer is large enough to hold the data before processing it.


```cpp
/*
 * sftp_read
 *
 * Read from an SFTP file handle
 *
 */
static ssize_t sftp_read(LIBSSH2_SFTP_HANDLE * handle, char *buffer,
                         size_t buffer_size)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t count = 0;
    struct sftp_pipeline_chunk *chunk;
    struct sftp_pipeline_chunk *next;
    ssize_t rc;
    struct _libssh2_sftp_handle_file_data *filep =
        &handle->u.file;
    size_t bytes_in_buffer = 0;
    char *sliding_bufferp = buffer;

    /* This function can be interrupted in three different places where it
       might need to wait for data from the network.  It returns EAGAIN to
       allow non-blocking clients to do other work but these client are
       expected to call this function again (possibly many times) to finish
       the operation.

       The tricky part is that if we previously aborted a sftp_read due to
       EAGAIN, we must continue at the same spot to continue the previously
       interrupted operation.  This is done using a state machine to record
       what phase of execution we were at.  The state is stored in
       sftp->read_state.

       libssh2_NB_state_idle: The first phase is where we prepare multiple
       FXP_READ packets to do optimistic read-ahead.  We send off as many as
       possible in the second phase without waiting for a response to each
       one; this is the key to fast reads. But we may have to adjust the
       channel window size to do this which may interrupt this function while
       waiting.  The state machine saves the phase as libssh2_NB_state_idle so
       it returns here on the next call.

       libssh2_NB_state_sent: The second phase is where we send the FXP_READ
       packets.  Writing them to the channel can be interrupted with EAGAIN
       but the state machine ensures we skip the first phase on the next call
       and resume sending.

       libssh2_NB_state_sent2: In the third phase (indicated by ) we read the
       data from the responses that have arrived so far.  Reading can be
       interrupted with EAGAIN but the state machine ensures we skip the first
       and second phases on the next call and resume sending.
    */

    switch(sftp->read_state) {
    case libssh2_NB_state_idle:

        /* Some data may already have been read from the server in the
           previous call but didn't fit in the buffer at the time.  If so, we
           return that now as we can't risk being interrupted later with data
           partially written to the buffer. */
        if(filep->data_left) {
            size_t copy = MIN(buffer_size, filep->data_left);

            memcpy(buffer, &filep->data[ filep->data_len - filep->data_left],
                   copy);

            filep->data_left -= copy;
            filep->offset += copy;

            if(!filep->data_left) {
                LIBSSH2_FREE(session, filep->data);
                filep->data = NULL;
            }

            return copy;
        }

        if(filep->eof) {
            return 0;
        }
        else {
            /* We allow a number of bytes being requested at any given time
               without having been acked - until we reach EOF. */

            /* Number of bytes asked for that haven't been acked yet */
            size_t already = (size_t)(filep->offset_sent - filep->offset);

            size_t max_read_ahead = buffer_size*4;
            unsigned long recv_window;

            if(max_read_ahead > LIBSSH2_CHANNEL_WINDOW_DEFAULT*4)
                max_read_ahead = LIBSSH2_CHANNEL_WINDOW_DEFAULT*4;

            /* if the buffer_size passed in now is smaller than what has
               already been sent, we risk getting count become a very large
               number */
            if(max_read_ahead > already)
                count = max_read_ahead - already;

            /* 'count' is how much more data to ask for, and 'already' is how
               much data that already has been asked for but not yet returned.
               Specifically, 'count' means how much data that have or will be
               asked for by the nodes that are already added to the linked
               list. Some of those read requests may not actually have been
               sent off successfully yet.

               If 'already' is very large it should be perfectly fine to have
               count set to 0 as then we don't have to ask for more data
               (right now).

               buffer_size*4 is just picked more or less out of the air. The
               idea is that when reading SFTP from a remote server, we send
               away multiple read requests guessing that the client will read
               more than only this 'buffer_size' amount of memory. So we ask
               for maximum buffer_size*4 amount of data so that we can return
               them very fast in subsequent calls.
            */

            recv_window = libssh2_channel_window_read_ex(sftp->channel,
                                                         NULL, NULL);
            if(max_read_ahead > recv_window) {
                /* more data will be asked for than what the window currently
                   allows, expand it! */

                rc = _libssh2_channel_receive_window_adjust(sftp->channel,
                                                            max_read_ahead*8,
                                                            1, NULL);
                /* if this returns EAGAIN, we will get back to this function
                   at next call */
                assert(rc != LIBSSH2_ERROR_EAGAIN || !filep->data_left);
                assert(rc != LIBSSH2_ERROR_EAGAIN || !filep->eof);
                if(rc)
                    return rc;
            }
        }

        while(count > 0) {
            unsigned char *s;

            /* 25 = packet_len(4) + packet_type(1) + request_id(4) +
               handle_len(4) + offset(8) + count(4) */
            uint32_t packet_len = (uint32_t)handle->handle_len + 25;
            uint32_t request_id;

            uint32_t size = count;
            if(size < buffer_size)
                size = buffer_size;
            if(size > MAX_SFTP_READ_SIZE)
                size = MAX_SFTP_READ_SIZE;

            chunk = LIBSSH2_ALLOC(session, packet_len +
                                  sizeof(struct sftp_pipeline_chunk));
            if(!chunk)
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "malloc fail for FXP_WRITE");

            chunk->offset = filep->offset_sent;
            chunk->len = size;
            chunk->lefttosend = packet_len;
            chunk->sent = 0;

            s = chunk->packet;

            _libssh2_store_u32(&s, packet_len - 4);
            *s++ = SSH_FXP_READ;
            request_id = sftp->request_id++;
            chunk->request_id = request_id;
            _libssh2_store_u32(&s, request_id);
            _libssh2_store_str(&s, handle->handle, handle->handle_len);
            _libssh2_store_u64(&s, filep->offset_sent);
            filep->offset_sent += size; /* advance offset at once */
            _libssh2_store_u32(&s, size);

            /* add this new entry LAST in the list */
            _libssh2_list_add(&handle->packet_list, &chunk->node);
            count -= MIN(size, count); /* deduct the size we used, as we might
                                        * have to create more packets */
            _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                           "read request id %d sent (offset: %d, size: %d)",
                           request_id, (int)chunk->offset, (int)chunk->len);
        }
        /* FALL-THROUGH */
    case libssh2_NB_state_sent:

        sftp->read_state = libssh2_NB_state_idle;

        /* move through the READ packets that haven't been sent and send as
           many as possible - remember that we don't block */
        chunk = _libssh2_list_first(&handle->packet_list);

        while(chunk) {
            if(chunk->lefttosend) {

                rc = _libssh2_channel_write(channel, 0,
                                            &chunk->packet[chunk->sent],
                                            chunk->lefttosend);
                if(rc < 0) {
                    sftp->read_state = libssh2_NB_state_sent;
                    return rc;
                }

                /* remember where to continue sending the next time */
                chunk->lefttosend -= rc;
                chunk->sent += rc;

                if(chunk->lefttosend) {
                    /* We still have data left to send for this chunk.
                     * If there is at least one completely sent chunk,
                     * we can get out of this loop and start reading.  */
                    if(chunk != _libssh2_list_first(&handle->packet_list)) {
                        break;
                    }
                    else {
                        continue;
                    }
                }
            }

            /* move on to the next chunk with data to send */
            chunk = _libssh2_list_next(&chunk->node);
        }
        /* FALL-THROUGH */

    case libssh2_NB_state_sent2:

        sftp->read_state = libssh2_NB_state_idle;

        /*
         * Count all ACKed packets and act on the contents of them.
         */
        chunk = _libssh2_list_first(&handle->packet_list);

        while(chunk) {
            unsigned char *data;
            size_t data_len;
            uint32_t rc32;
            static const unsigned char read_responses[2] = {
                SSH_FXP_DATA, SSH_FXP_STATUS
            };

            if(chunk->lefttosend) {
                /* if the chunk still has data left to send, we shouldn't wait
                   for an ACK for it just yet */
                if(bytes_in_buffer > 0) {
                    return bytes_in_buffer;
                }
                else {
                    /* we should never reach this point */
                    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                          "sftp_read() internal error");
                }
            }

            rc = sftp_packet_requirev(sftp, 2, read_responses,
                                      chunk->request_id, &data, &data_len, 9);
            if(rc == LIBSSH2_ERROR_EAGAIN && bytes_in_buffer != 0) {
                /* do not return EAGAIN if we have already
                 * written data into the buffer */
                return bytes_in_buffer;
            }

            if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
                if(data_len > 0) {
                    LIBSSH2_FREE(session, data);
                }
                return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                      "Response too small");
            }
            else if(rc < 0) {
                sftp->read_state = libssh2_NB_state_sent2;
                return rc;
            }

            /*
             * We get DATA or STATUS back. STATUS can be error, or it is
             * FX_EOF when we reach the end of the file.
             */

            switch(data[0]) {
            case SSH_FXP_STATUS:
                /* remove the chunk we just processed */

                _libssh2_list_remove(&chunk->node);
                LIBSSH2_FREE(session, chunk);

                /* we must remove all outstanding READ requests, as either we
                   got an error or we're at end of file */
                sftp_packetlist_flush(handle);

                rc32 = _libssh2_ntohu32(data + 5);
                LIBSSH2_FREE(session, data);

                if(rc32 == LIBSSH2_FX_EOF) {
                    filep->eof = TRUE;
                    return bytes_in_buffer;
                }
                else {
                    sftp->last_errno = rc32;
                    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                          "SFTP READ error");
                }
                break;

            case SSH_FXP_DATA:
                if(chunk->offset != filep->offset) {
                    /* This could happen if the server returns less bytes than
                       requested, which shouldn't happen for normal files. See:
                       https://tools.ietf.org/html/draft-ietf-secsh-filexfer-02
                       #section-6.4
                    */
                    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                          "Read Packet At Unexpected Offset");
                }

                rc32 = _libssh2_ntohu32(data + 5);
                if(rc32 > (data_len - 9))
                    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                          "SFTP Protocol badness");

                if(rc32 > chunk->len) {
                    /* A chunk larger than we requested was returned to us.
                       This is a protocol violation and we don't know how to
                       deal with it. Bail out! */
                    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                          "FXP_READ response too big");
                }

                if(rc32 != chunk->len) {
                    /* a short read does not imply end of file, but we must
                       adjust the offset_sent since it was advanced with a
                       full chunk->len before */
                    filep->offset_sent -= (chunk->len - rc32);
                }

                if((bytes_in_buffer + rc32) > buffer_size) {
                    /* figure out the overlap amount */
                    filep->data_left = (bytes_in_buffer + rc32) - buffer_size;

                    /* getting the full packet would overflow the buffer, so
                       only get the correct amount and keep the remainder */
                    rc32 = (uint32_t)buffer_size - bytes_in_buffer;

                    /* store data to keep for next call */
                    filep->data = data;
                    filep->data_len = data_len;
                }
                else
                    filep->data_len = 0;

                /* copy the received data from the received FXP_DATA packet to
                   the buffer at the correct index */
                memcpy(sliding_bufferp, data + 9, rc32);
                filep->offset += rc32;
                bytes_in_buffer += rc32;
                sliding_bufferp += rc32;

                if(filep->data_len == 0)
                    /* free the allocated data if not stored to keep */
                    LIBSSH2_FREE(session, data);

                /* remove the chunk we just processed keeping track of the
                 * next one in case we need it */
                next = _libssh2_list_next(&chunk->node);
                _libssh2_list_remove(&chunk->node);
                LIBSSH2_FREE(session, chunk);

                /* check if we have space left in the buffer
                 * and either continue to the next chunk or stop
                 */
                if(bytes_in_buffer < buffer_size) {
                    chunk = next;
                }
                else {
                    chunk = NULL;
                }

                break;
            default:
                return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                      "SFTP Protocol badness: unrecognised "
                                      "read request response");
            }
        }

        if(bytes_in_buffer > 0)
            return bytes_in_buffer;

        break;

    default:
        assert(!"State machine error; unrecognised read state");
    }

    /* we should never reach this point */
    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                          "sftp_read() internal error");
}

```

这段代码定义了一个名为`libssh2_sftp_read`的函数，它接受一个SFTP文件 handle和最多2MB的缓冲区。这个函数的作用是读取文件内容到给定的缓冲区中。

函数的实现包括以下几个步骤：

1. 检查输入参数是否为空，如果是，则返回错误码`LIBSSH2_ERROR_BAD_USE`。
2. 如果输入参数`hnd`为空，则函数跳过。
3. 初始化文件输入操作，并将`hnd`指向`sftp`对象的`channel`成员的`session`成员的`sftp_read`函数。
4. 使用`sftp_read`函数从文件中读取数据，并将其存储在`buffer`参数中。
5. 使用`BLOCK_ADJUST`函数来调整读取操作的块大小，以确保它不会跨越文件大小和缓冲区最大长度。
6. 返回`rc`，表示文件读取操作的返回码。


```cpp
/* libssh2_sftp_read
 * Read from an SFTP file handle
 */
LIBSSH2_API ssize_t
libssh2_sftp_read(LIBSSH2_SFTP_HANDLE *hnd, char *buffer,
                  size_t buffer_maxlen)
{
    ssize_t rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_read(hnd, buffer, buffer_maxlen));
    return rc;
}

```

这段代码是 Sftp 客户端（sftp）在使用 SFTP 协议从服务器上读取目录时处理状态变化的代码。

代码会根据 SFTP 协议的不同状态做出不同的处理，包括读取状态、写入状态、异常状态等。

具体来说，代码会等待服务器返回状态信息，如果返回的结果不是有效的状态，会尝试重新连接并继续等待，直到连接成功并返回有效的状态信息。

在代码中，通过 LIBSSH2_FREE() 函数释放了读取的文件和携带数据，通过 LIBSSH2_ERROR() 函数处理了错误，并返回错误信息。


```cpp
/* sftp_readdir
 * Read from an SFTP directory handle
 */
static ssize_t sftp_readdir(LIBSSH2_SFTP_HANDLE *handle, char *buffer,
                            size_t buffer_maxlen, char *longentry,
                            size_t longentry_maxlen,
                            LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    uint32_t num_names;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len = handle->handle_len + 13;
    unsigned char *s, *data;
    static const unsigned char read_responses[2] = {
        SSH_FXP_NAME, SSH_FXP_STATUS };
    ssize_t retcode;

    if(sftp->readdir_state == libssh2_NB_state_idle) {
        if(handle->u.dir.names_left) {
            /*
             * A prior request returned more than one directory entry,
             * feed it back from the buffer
             */
            LIBSSH2_SFTP_ATTRIBUTES attrs_dummy;
            size_t real_longentry_len;
            size_t real_filename_len;
            size_t filename_len;
            size_t longentry_len;
            size_t names_packet_len = handle->u.dir.names_packet_len;
            int attr_len = 0;

            if(names_packet_len >= 4) {
                s = (unsigned char *) handle->u.dir.next_name;
                real_filename_len = _libssh2_ntohu32(s);
                s += 4;
                names_packet_len -= 4;
            }
            else {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            filename_len = real_filename_len;
            if(filename_len >= buffer_maxlen) {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            if(buffer_maxlen >= filename_len && names_packet_len >=
               filename_len) {
                memcpy(buffer, s, filename_len);
                buffer[filename_len] = '\0';           /* zero terminate */
                s += real_filename_len;
                names_packet_len -= real_filename_len;
            }
            else {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            if(names_packet_len >= 4) {
                real_longentry_len = _libssh2_ntohu32(s);
                s += 4;
                names_packet_len -= 4;
            }
            else {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            if(longentry && (longentry_maxlen>1)) {
                longentry_len = real_longentry_len;

                if(longentry_len >= longentry_maxlen ||
                   longentry_len > names_packet_len) {
                    filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                    goto end;
                }

                memcpy(longentry, s, longentry_len);
                longentry[longentry_len] = '\0'; /* zero terminate */
            }

            if(real_longentry_len <= names_packet_len) {
                s += real_longentry_len;
                names_packet_len -= real_longentry_len;
            }
            else {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            if(attrs)
                memset(attrs, 0, sizeof(LIBSSH2_SFTP_ATTRIBUTES));

            attr_len = sftp_bin2attr(attrs ? attrs : &attrs_dummy, s,
                                     names_packet_len);

            if(attr_len >= 0) {
                s += attr_len;
                names_packet_len -= attr_len;
            }
            else {
                filename_len = (size_t)LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                goto end;
            }

            handle->u.dir.next_name = (char *) s;
            handle->u.dir.names_packet_len = names_packet_len;
          end:

            if((--handle->u.dir.names_left) == 0)
                LIBSSH2_FREE(session, handle->u.dir.names_packet);

            _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                           "libssh2_sftp_readdir_ex() return %d",
                           filename_len);
            return (ssize_t)filename_len;
        }

        /* Request another entry(entries?) */

        s = sftp->readdir_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->readdir_packet)
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "FXP_READDIR packet");

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_READDIR;
        sftp->readdir_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->readdir_request_id);
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        sftp->readdir_state = libssh2_NB_state_created;
    }

    if(sftp->readdir_state == libssh2_NB_state_created) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Reading entries from directory handle");
        retcode = _libssh2_channel_write(channel, 0, sftp->readdir_packet,
                                         packet_len);
        if(retcode == LIBSSH2_ERROR_EAGAIN) {
            return retcode;
        }
        else if((ssize_t)packet_len != retcode) {
            LIBSSH2_FREE(session, sftp->readdir_packet);
            sftp->readdir_packet = NULL;
            sftp->readdir_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }

        LIBSSH2_FREE(session, sftp->readdir_packet);
        sftp->readdir_packet = NULL;

        sftp->readdir_state = libssh2_NB_state_sent;
    }

    retcode = sftp_packet_requirev(sftp, 2, read_responses,
                                   sftp->readdir_request_id, &data,
                                   &data_len, 9);
    if(retcode == LIBSSH2_ERROR_EAGAIN)
        return retcode;
    else if(retcode == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Status message too short");
    }
    else if(retcode) {
        sftp->readdir_state = libssh2_NB_state_idle;
        return _libssh2_error(session, retcode,
                              "Timeout waiting for status message");
    }

    if(data[0] == SSH_FXP_STATUS) {
        retcode = _libssh2_ntohu32(data + 5);
        LIBSSH2_FREE(session, data);
        if(retcode == LIBSSH2_FX_EOF) {
            sftp->readdir_state = libssh2_NB_state_idle;
            return 0;
        }
        else {
            sftp->last_errno = retcode;
            sftp->readdir_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    sftp->readdir_state = libssh2_NB_state_idle;

    num_names = _libssh2_ntohu32(data + 5);
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%lu entries returned",
                   num_names);
    if(!num_names) {
        LIBSSH2_FREE(session, data);
        return 0;
    }

    handle->u.dir.names_left = num_names;
    handle->u.dir.names_packet = data;
    handle->u.dir.next_name = (char *) data + 9;
    handle->u.dir.names_packet_len = data_len - 9;

    /* use the name popping mechanism from the start of the function */
    return sftp_readdir(handle, buffer, buffer_maxlen, longentry,
                        longentry_maxlen, attrs);
}

```

这段代码是一个名为 `libssh2_sftp_readdir_ex` 的函数，它是 Libssh2 SSH 客户端的一个内部函数，用于从 SFTP 目录 handle 中读取目录内容。

具体来说，这段代码的作用如下：

1. 读取 SFTP 目录 handle：首先，通过调用 `libssh2_sftp_open` 函数，创建一个 SFTP 连接，然后使用 `libssh2_sftp_get_鉴别码` 函数获取鉴别码，从而获取到一个 SFTP 目录 handle。
2. 读取目录内容：使用这个 directory handle，通过调用 `sftp_readdir` 函数，从 SFTP 目录中读取目录内容，并将读取到的内容存储到缓冲区中。注意，这个函数需要传递一个指向 SFTP 目录内容的指针（通过 `libssh2_sftp_get_contents` 函数可以获取到），以及一个最大读取长度（通过 `size_t buffer_maxlen` 参数设置）。
3. 返回结果：将读取到的目录内容存储到缓冲区中，然后使用 `rc` 参数返回状态码。如果返回状态码时，函数调用失败，则返回 LIBSSH2_ERROR_BAD_USE；否则，函数返回 0。

这段代码定义在 `libssh2_sftp_readdir_ex.h` 头文件中，是 Libssh2 SSH 客户端中的一个内部函数。


```cpp
/* libssh2_sftp_readdir_ex
 * Read from an SFTP directory handle
 */
LIBSSH2_API int
libssh2_sftp_readdir_ex(LIBSSH2_SFTP_HANDLE *hnd, char *buffer,
                        size_t buffer_maxlen, char *longentry,
                        size_t longentry_maxlen,
                        LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    int rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_readdir(hnd, buffer, buffer_maxlen, longentry,
                              longentry_maxlen, attrs));
    return rc;
}

```

这段代码是一个名为 `sftp_write` 的函数，其作用是将数据写入到 SFTP handle 中。它接受一个数据缓冲区作为参数，并返回向缓冲区中写入的数据字节数，或者是一个负的错误码。

该函数的实现过程如下：

1. 首先，它检查是否可以发送数据，即检查缓冲区中还有多少数据可以写入 SFTP handle。
2. 如果还有数据可以写入，该函数将数据缓冲区中的所有数据分成不会超过 `N` 的数据包。
3. 然后，该函数将所有创建的 outgoing 数据包添加到已有的数据包链表中。
4. 接着，该函数遍历数据包链表，并发送所有尚未发送的数据包，直到达到数据缓冲区的末尾或者调用者已经输入了足够的数据来调用这个函数。
5. 在发送数据包的过程中，该函数会检查是否接收到了确认（ACK）消息。如果数据包已经确认收到并被正确发送，该函数将数据包从数据包链表中移除，并增加确认计数器。
6. 最后，该函数返回已经确认收到的数据字节数。

需要注意的是，该函数还包含一个警告，即不要返回超过指定缓冲区大小的数据。


```cpp
/*
 * sftp_write
 *
 * Write data to an SFTP handle. Returns the number of bytes written, or
 * a negative error code.
 *
 * We recommend sending very large data buffers to this function!
 *
 * Concept:
 *
 * - Detect how much of the given buffer that was already sent in a previous
 *   call by inspecting the linked list of outgoing chunks. Make sure to skip
 *   passed the data that has already been taken care of.
 *
 * - Split all (new) outgoing data in chunks no larger than N.
 *
 * - Each N bytes chunk gets created as a separate SFTP packet.
 *
 * - Add all created outgoing packets to the linked list.
 *
 * - Walk through the list and send the chunks that haven't been sent,
 *   as many as possible until EAGAIN. Some of the chunks may have been put
 *   in the list in a previous invoke.
 *
 * - For all the chunks in the list that have been completely sent off, check
 *   for ACKs. If a chunk has been ACKed, it is removed from the linked
 *   list and the "acked" counter gets increased with that data amount.
 *
 * - Return TOTAL bytes acked so far.
 *
 * Caveats:
 * -  be careful: we must not return a higher number than what was given!
 *
 * TODO:
 *   Introduce an option that disables this sort of "speculative" ahead writing
 *   as there's a risk that it will do harm to some app.
 */

```



This is a function for the SVG (Simple and Generator) toolkit that parses and validates an SVG file. It reads in an SVG file and returns it in its original form.

The function takes in a SVG file string and returns the parsed and validated SVG file. It does this by first reading in the file and then parsing it using a combination of different parsing libraries, such as libxml2 and lib滨海溴丛林。一旦 parsing is done, it will validate the file by checking it for syntax errors and then return it in its original form.

The function has several options that can be used to control its behavior, such as the encoding of the SVG file, the directory where the file is located, and the number of errors to accept before returning.

It is important to note that this function only reads in SVG files and does not perform any kind of error handling or data validation. It is the responsibility of the calling application to handle any errors or unexpected events that may occur during the parsing process.


```cpp
static ssize_t sftp_write(LIBSSH2_SFTP_HANDLE *handle, const char *buffer,
                          size_t count)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    uint32_t retcode;
    uint32_t packet_len;
    unsigned char *s, *data;
    ssize_t rc;
    struct sftp_pipeline_chunk *chunk;
    struct sftp_pipeline_chunk *next;
    size_t acked = 0;
    size_t org_count = count;
    size_t already;

    switch(sftp->write_state) {
    default:
    case libssh2_NB_state_idle:

        /* Number of bytes sent off that haven't been acked and therefore we
           will get passed in here again.

           Also, add up the number of bytes that actually already have been
           acked but we haven't been able to return as such yet, so we will
           get that data as well passed in here again.
        */
        already = (size_t) (handle->u.file.offset_sent -
                            handle->u.file.offset)+
            handle->u.file.acked;

        if(count >= already) {
            /* skip the part already made into packets */
            buffer += already;
            count -= already;
        }
        else
            /* there is more data already fine than what we got in this call */
            count = 0;

        sftp->write_state = libssh2_NB_state_idle;
        while(count) {
            /* TODO: Possibly this should have some logic to prevent a very
               very small fraction to be left but lets ignore that for now */
            uint32_t size = MIN(MAX_SFTP_OUTGOING_SIZE, count);
            uint32_t request_id;

            /* 25 = packet_len(4) + packet_type(1) + request_id(4) +
               handle_len(4) + offset(8) + count(4) */
            packet_len = handle->handle_len + size + 25;

            chunk = LIBSSH2_ALLOC(session, packet_len +
                                  sizeof(struct sftp_pipeline_chunk));
            if(!chunk)
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "malloc fail for FXP_WRITE");

            chunk->len = size;
            chunk->sent = 0;
            chunk->lefttosend = packet_len;

            s = chunk->packet;
            _libssh2_store_u32(&s, packet_len - 4);

            *(s++) = SSH_FXP_WRITE;
            request_id = sftp->request_id++;
            chunk->request_id = request_id;
            _libssh2_store_u32(&s, request_id);
            _libssh2_store_str(&s, handle->handle, handle->handle_len);
            _libssh2_store_u64(&s, handle->u.file.offset_sent);
            handle->u.file.offset_sent += size; /* advance offset at once */
            _libssh2_store_str(&s, buffer, size);

            /* add this new entry LAST in the list */
            _libssh2_list_add(&handle->packet_list, &chunk->node);

            buffer += size;
            count -= size; /* deduct the size we used, as we might have
                              to create more packets */
        }

        /* move through the WRITE packets that haven't been sent and send as
           many as possible - remember that we don't block */
        chunk = _libssh2_list_first(&handle->packet_list);

        while(chunk) {
            if(chunk->lefttosend) {
                rc = _libssh2_channel_write(channel, 0,
                                            &chunk->packet[chunk->sent],
                                            chunk->lefttosend);
                if(rc < 0)
                    /* remain in idle state */
                    return rc;

                /* remember where to continue sending the next time */
                chunk->lefttosend -= rc;
                chunk->sent += rc;

                if(chunk->lefttosend)
                    /* data left to send, get out of loop */
                    break;
            }

            /* move on to the next chunk with data to send */
            chunk = _libssh2_list_next(&chunk->node);
        }

        /* fall-through */
    case libssh2_NB_state_sent:

        sftp->write_state = libssh2_NB_state_idle;
        /*
         * Count all ACKed packets
         */
        chunk = _libssh2_list_first(&handle->packet_list);

        while(chunk) {
            if(chunk->lefttosend)
                /* if the chunk still has data left to send, we shouldn't wait
                   for an ACK for it just yet */
                break;

            else if(acked)
                /* if we have sent data that is acked, we must return that
                   info before we call a function that might return EAGAIN */
                break;

            /* we check the packets in order */
            rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                                     chunk->request_id, &data, &data_len, 9);
            if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
                if(data_len > 0) {
                    LIBSSH2_FREE(session, data);
                }
                return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                      "FXP write packet too short");
            }
            else if(rc < 0) {
                if(rc == LIBSSH2_ERROR_EAGAIN)
                    sftp->write_state = libssh2_NB_state_sent;
                return rc;
            }

            retcode = _libssh2_ntohu32(data + 5);
            LIBSSH2_FREE(session, data);

            sftp->last_errno = retcode;
            if(retcode == LIBSSH2_FX_OK) {
                acked += chunk->len; /* number of payload data that was acked
                                        here */

                /* we increase the offset value for all acks */
                handle->u.file.offset += chunk->len;

                next = _libssh2_list_next(&chunk->node);

                _libssh2_list_remove(&chunk->node); /* remove from list */
                LIBSSH2_FREE(session, chunk); /* free memory */

                chunk = next;
            }
            else {
                /* flush all pending packets from the outgoing list */
                sftp_packetlist_flush(handle);

                /* since we return error now, the application will not get any
                   outstanding data acked, so we need to rewind the offset to
                   where the application knows it has reached with acked
                   data */
                handle->u.file.offset -= handle->u.file.acked;

                /* then reset the offset_sent to be the same as the offset */
                handle->u.file.offset_sent = handle->u.file.offset;

                /* clear the acked counter since we can have no pending data to
                   ack after an error */
                handle->u.file.acked = 0;

                /* the server returned an error for that written chunk,
                   propagate this back to our parent function */
                return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                      "FXP write failed");
            }
        }
        break;
    }

    /* if there were acked data in a previous call that wasn't returned then,
       add that up and try to return it all now. This can happen if the app
       first sends a huge buffer of data, and then in a second call it sends a
       smaller one. */
    acked += handle->u.file.acked;

    if(acked) {
        ssize_t ret = MIN(acked, org_count);
        /* we got data acked so return that amount, but no more than what
           was asked to get sent! */

        /* store the remainder. 'ret' is always equal to or less than 'acked'
           here */
        handle->u.file.acked = acked - ret;

        return ret;
    }

    else
        return 0; /* nothing was acked, and no EAGAIN was received! */
}

```

这段代码定义了一个名为 `libssh2_sftp_write` 的函数，它是 `libssh2_sftp_handle` 结构的指针函数，它的作用是向通过 `libssh2_sftp_handle` 指向的文件 handle 写入数据。

具体来说，函数接受三个参数：

1. `hnd`：文件 handle，指向一个 `LIBSSH2_SFTP_HANDLE` 类型的结构体，它通常是一个 `libssh2_sftp_handle` 类型的指针，指向一个 SFTP 通道的文件 handle。
2. `buffer`：一个字符数组，用于存储要写入文件的数据。
3. `count`：要写入文件的数据量，以字节为单位。

函数首先检查 `hnd` 是否为 `NULL`，如果是，函数返回 `LIBSSH2_ERROR_BAD_USE`，表示出错。否则，函数调用 `sftp_write` 函数，将 `buffer` 中的数据写入到 `hnd->sftp->channel->session` 指针所指向的文件 handle 中，然后返回该函数的返回值。

注意，`libssh2_sftp_write` 函数没有检查要写入的数据是否在合法范围内，因此可能会导致安全漏洞。


```cpp
/* libssh2_sftp_write
 * Write data to a file handle
 */
LIBSSH2_API ssize_t
libssh2_sftp_write(LIBSSH2_SFTP_HANDLE *hnd, const char *buffer,
                   size_t count)
{
    ssize_t rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_write(hnd, buffer, count));
    return rc;

}

```



This function appears to be part of the SSH2 protocol for sending a file-transfer request to an SFTP (Secure Shell Transfer Protocol) server. It performs the following steps:

1. It checks for an error code. If the return code from `libssh2_channel_write()` is negative, it sets the `fsync_state` to `libssh2_NB_state_idle`, which is the idle state for the SFTP protocol. This indicates that no file transfer is currently active.

2. It checks that the SFTP server has received the data and has sent a confirm收到 the data packet.

3. It verifies the data end-pointmajor is equal to 0 and end-endian is equal to 0.

4. It initializes the file transfer session.

5. It calls the `sftp_packet_require()` function to check that the data received from the SFTP server is complete.

6. If the call to `sftp_packet_require()` fails, it returns the error code.

7. If the data received from the SFTP server is incomplete, it frees the data and returns an error code.

8. If the SFTP server has received all the data, it sets the `fsync_state` to `libssh2_NB_state_idle`.

9. It returns 0.

It should be noted that `libssh2_error()` function is not defined in the code provided, but based on its return value it appears to return an error code indicating that the file transfer failed.


```cpp
static int sftp_fsync(LIBSSH2_SFTP_HANDLE *handle)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    /* 34 = packet_len(4) + packet_type(1) + request_id(4) +
       string_len(4) + strlen("fsync@openssh.com")(17) + handle_len(4) */
    uint32_t packet_len = handle->handle_len + 34;
    size_t data_len;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    uint32_t retcode;

    if(sftp->fsync_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Issuing fsync command");
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_EXTENDED "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_EXTENDED;
        sftp->fsync_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->fsync_request_id);
        _libssh2_store_str(&s, "fsync@openssh.com", 17);
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        sftp->fsync_state = libssh2_NB_state_created;
    }
    else {
        packet = sftp->fsync_packet;
    }

    if(sftp->fsync_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN ||
            (0 <= rc && rc < (ssize_t)packet_len)) {
            sftp->fsync_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }

        LIBSSH2_FREE(session, packet);
        sftp->fsync_packet = NULL;

        if(rc < 0) {
            sftp->fsync_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        sftp->fsync_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->fsync_request_id, &data, &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fsync packet too short");
    }
    else if(rc) {
        sftp->fsync_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    sftp->fsync_state = libssh2_NB_state_idle;

    retcode = _libssh2_ntohu32(data + 5);
    LIBSSH2_FREE(session, data);

    if(retcode != LIBSSH2_FX_OK) {
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "fsync failed");
    }

    return 0;
}

```

这段代码定义了一个名为`libssh2_sftp_fsync`的函数，它属于`libssh2_sftp_private`类型。

这个函数的作用是将挂载在`hnd`对象上的`sftp`对象中的数据写入到`hnd`所指定的目标文件中，并在完成写入后返回一个状态码。

具体实现中，首先对`hnd`进行判断，如果`hnd`为`NULL`，则返回`LIBSSH2_ERROR_BAD_USE`，表明函数无法正常返回。如果`hnd`存在，则调用`sftp_fsync`函数，将`hnd`中的`sftp`对象中的数据写入到目标文件中，并使用`BLOCK_ADJUST`函数来调整IO缓冲区的位置，以便在写入数据时能够正确提交IO缓冲区的修改。最后，函数返回状态码`rc`，表示写入操作的完成。如果`rc`为负数，则说明在执行写入操作时出现了错误，需要进行相应的错误处理。


```cpp
/* libssh2_sftp_fsync
 * Commit data on the handle to disk.
 */
LIBSSH2_API int
libssh2_sftp_fsync(LIBSSH2_SFTP_HANDLE *hnd)
{
    int rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_fsync(hnd));
    return rc;
}


```

This is a function definition for `libssh2_sftp_fstat` which takes an SFTP file descriptor, `data`, the contents of the file, and the file length as input arguments and returns either a success or failure status along with an error message.

The function first checks if the SFTP file descriptor is valid and then checks if the file is a regular file. If the file is not a regular file, the function attempts to retrieve the file status from the SFTP file descriptor. If the file status cannot be retrieved, the function waits for a response from the SFTP server.

If the file is a regular file, the function checks if the file status is valid by comparing the contents of the file to a known good file status. If the contents of the file are not a valid file status, the function returns an error with a relevant error message.

If the file is not a regular file and the SFTP file descriptor does not provide a valid file status, the function returns an error with a relevant error message.

If the file is a regular file and the SFTP file descriptor provides a valid file status, the function returns 0.

If there is an error in the file or the file is not a regular file, the function returns an error with a relevant error message.


```cpp
/*
 * sftp_fstat
 *
 * Get or Set stat on a file
 */
static int sftp_fstat(LIBSSH2_SFTP_HANDLE *handle,
                      LIBSSH2_SFTP_ATTRIBUTES *attrs, int setstat)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len =
        handle->handle_len + 13 + (setstat ? sftp_attrsize(attrs->flags) : 0);
    unsigned char *s, *data;
    static const unsigned char fstat_responses[2] =
        { SSH_FXP_ATTRS, SSH_FXP_STATUS };
    ssize_t rc;

    if(sftp->fstat_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Issuing %s command",
                       setstat ? "set-stat" : "stat");
        s = sftp->fstat_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->fstat_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "FSTAT/FSETSTAT packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = setstat ? SSH_FXP_FSETSTAT : SSH_FXP_FSTAT;
        sftp->fstat_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->fstat_request_id);
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        if(setstat) {
            s += sftp_attr2bin(s, attrs);
        }

        sftp->fstat_state = libssh2_NB_state_created;
    }

    if(sftp->fstat_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->fstat_packet,
                                    packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if((ssize_t)packet_len != rc) {
            LIBSSH2_FREE(session, sftp->fstat_packet);
            sftp->fstat_packet = NULL;
            sftp->fstat_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  (setstat ? "Unable to send FXP_FSETSTAT"
                                   : "Unable to send FXP_FSTAT command"));
        }
        LIBSSH2_FREE(session, sftp->fstat_packet);
        sftp->fstat_packet = NULL;

        sftp->fstat_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_requirev(sftp, 2, fstat_responses,
                              sftp->fstat_request_id, &data,
                              &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fstat packet too short");
    }
    else if(rc) {
        sftp->fstat_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Timeout waiting for status message");
    }

    sftp->fstat_state = libssh2_NB_state_idle;

    if(data[0] == SSH_FXP_STATUS) {
        uint32_t retcode;

        retcode = _libssh2_ntohu32(data + 5);
        LIBSSH2_FREE(session, data);
        if(retcode == LIBSSH2_FX_OK) {
            return 0;
        }
        else {
            sftp->last_errno = retcode;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    if(sftp_bin2attr(attrs, data + 5, data_len - 5) < 0) {
        LIBSSH2_FREE(session, data);
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Attributes too short in SFTP fstat");
    }

    LIBSSH2_FREE(session, data);

    return 0;
}

```

这段代码定义了一个名为 `libssh2_sftp_fstat_ex` 的函数，它是 `libssh2_sftp_fstat` 的扩展函数。这个函数接受两个参数，一个是 `hnd` 指向的 SFTP 句柄，另一个是 `attrs` 是一个指向 `libssh2_sftp_attributes` 类型的变量，这个变量描述了 SFTP 文件的各种属性。函数的作用是获取或设置文件的状态，具体实现由第二个参数 `setstat` 控制。

函数首先检查输入参数是否为空，如果是，则返回一个错误码。然后，函数调用 `sftp_fstat` 函数来获取或设置文件状态，传递给 `attrs` 参数，再将得到的结果返回。注意，第二个参数 `setstat` 是一个布尔值，如果为 0，那么函数不会执行文件状态的设置，从而不会返回任何结果。


```cpp
/* libssh2_sftp_fstat_ex
 * Get or Set stat on a file
 */
LIBSSH2_API int
libssh2_sftp_fstat_ex(LIBSSH2_SFTP_HANDLE *hnd,
                      LIBSSH2_SFTP_ATTRIBUTES *attrs, int setstat)
{
    int rc;
    if(!hnd || !attrs)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_fstat(hnd, attrs, setstat));
    return rc;
}


```

这段代码是用于设置 Sftp 文件的读/写指针为文件中的任意位置的函数。具体来说，它执行以下操作：

1. 如果传入了空指针或者文件偏移量超出了文件大小，函数将直接返回，不需要做其他处理。

2. 如果指针已经指向了文件结束位置，或者已经发送了所有数据，函数将返回，同样不需要做其他处理。

3. 如果指针还在文件中，函数将把文件指针和已发送的数据偏移量都设置为给定的偏移量。然后，函数调用 sftp_packetlist_flush 函数来将所有 pending requests 立即取消，并清空文件中读取的数据。

4. 如果文件中还有数据，函数将释放左收到的缓冲数据，并使用 LIBSSH2_FREE 函数释放文件数据。

5. 最后，函数将 EOF 标志设置为假，并把文件指针和数据是否为空存储到 handle->u.file.data_len 和 handle->u.file.data 中。


```cpp
/* libssh2_sftp_seek64
 * Set the read/write pointer to an arbitrary position within the file
 */
LIBSSH2_API void
libssh2_sftp_seek64(LIBSSH2_SFTP_HANDLE *handle, libssh2_uint64_t offset)
{
    if(!handle)
        return;
    if(handle->u.file.offset == offset && handle->u.file.offset_sent == offset)
        return;

    handle->u.file.offset = handle->u.file.offset_sent = offset;
    /* discard all pending requests and currently read data */
    sftp_packetlist_flush(handle);

    /* free the left received buffered data */
    if(handle->u.file.data_left) {
        LIBSSH2_FREE(handle->sftp->channel->session, handle->u.file.data);
        handle->u.file.data_left = handle->u.file.data_len = 0;
        handle->u.file.data = NULL;
    }

    /* reset EOF to False */
    handle->u.file.eof = FALSE;
}

```

这段代码定义了两个函数，libssh2_sftp_seek 和 libssh2_sftp_tell。它们用于在 SSH2 Sftp 对象中设置文件指针的读/写位置。

libssh2_sftp_seek 函数接受一个 SFTP 处理对象和一个文件偏移量作为参数。它使用 libssh2_sftp_seek64 函数将文件指针的读/写位置设置为文件偏移量的任意位置。这个函数确保了文件指针不会被截断，即使文件偏移量超出了可接受范围。

libssh2_sftp_tell 函数返回当前读/写指针的文件偏移量。它首先检查是否传递了一个有效的 SFTP 处理对象，如果没有传递，它将返回 0。然后，它返回文件偏移量，这个偏移量将作为文件指针相对于文件的偏移量。

这两个函数是互相关联的，libssh2_sftp_seek 函数调用 libssh2_sftp_tell 函数来获取当前文件指针的偏移量，然后使用 libssh2_sftp_seek64 函数将其偏移量转换为 SFTP 文件指针的偏移量。


```cpp
/* libssh2_sftp_seek
 * Set the read/write pointer to an arbitrary position within the file
 */
LIBSSH2_API void
libssh2_sftp_seek(LIBSSH2_SFTP_HANDLE *handle, size_t offset)
{
    libssh2_sftp_seek64(handle, (libssh2_uint64_t)offset);
}

/* libssh2_sftp_tell
 * Return the current read/write pointer's offset
 */
LIBSSH2_API size_t
libssh2_sftp_tell(LIBSSH2_SFTP_HANDLE *handle)
{
    if(!handle)
        return 0; /* no handle, no size */

    /* NOTE: this may very well truncate the size if it is larger than what
       size_t can hold, so libssh2_sftp_tell64() is really the function you
       should use */
    return (size_t)(handle->u.file.offset);
}

```

这段代码定义了一个名为`libssh2_sftp_tell64`的函数，它是`libssh2_sftp_tell64`函数的别名。它的作用是返回当前读/写指针的偏移量，即文件在SFTP文件中的偏移量。

函数的第一个参数是一个指向`LIBSSH2_SFTP_HANDLE`类型的指针，代表了一个SFTP文件的handle。第二个参数是一个`libssh2_uint64_t`类型的变量，用于存储文件中的偏移量。函数返回这个偏移量的值。

该函数没有进行任何实际的文件读写操作，但是它通过返回文件的偏移量来告诉用户文件中还有哪些数据可以使用。


```cpp
/* libssh2_sftp_tell64
 * Return the current read/write pointer's offset
 */
LIBSSH2_API libssh2_uint64_t
libssh2_sftp_tell64(LIBSSH2_SFTP_HANDLE *handle)
{
    if(!handle)
        return 0; /* no handle, no size */

    return handle->u.file.offset;
}

/*
 * Flush all remaining incoming SFTP packets and zombies.
 */
```

该函数的作用是 flush（同步）SFTP（安全文件传输协议）数据包。它将连接到服务器的主机上的 SFTP 数据包发送到服务器，并将所有的数据包发送完毕后，将连接断开。

具体来说，函数接受一个 SFTP 连接对象（LIBSSH2_SFTP 类型），首先遍历所有的数据包，对于每个数据包，它将获取其数据并获取其下一个数据包。然后，函数将所有数据包发送到服务器，并重复这个过程，直到所有数据包都被发送完毕。

对于每个数据包，函数首先获取其下一个数据包，然后创建一个 Zombie Requests 结构体，它包含一个指向下一个数据包的指针，以及一个指向请求状态的指针。接着，函数使用 Zombie Requests 结构体，获取所有的 zombie 请求，并使用 libssh2_list_first 和 libssh2_list_next 函数来遍历所有的 requests。对于每个 request，函数创建一个 struct sftp\_zombie\_requests 结构体，并使用 libssh2\_free 和 libssh2\_list\_remove 函数来释放内存和删除请求。

最后，函数在循环中，还处理了所有的 zombie 请求。它使用 libssh2\_list\_next 和 libssh2\_list\_remove 函数来遍历所有的 zombie 请求，并使用 libssh2\_free 和 libssh2\_list\_remove 函数来释放内存和删除请求。


```cpp
static void sftp_packet_flush(LIBSSH2_SFTP *sftp)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_SFTP_PACKET *packet = _libssh2_list_first(&sftp->packets);
    struct sftp_zombie_requests *zombie =
        _libssh2_list_first(&sftp->zombie_requests);

    while(packet) {
        LIBSSH2_SFTP_PACKET *next;

        /* check next struct in the list */
        next =  _libssh2_list_next(&packet->node);
        _libssh2_list_remove(&packet->node);
        LIBSSH2_FREE(session, packet->data);
        LIBSSH2_FREE(session, packet);

        packet = next;
    }

    while(zombie) {
        /* figure out the next node */
        struct sftp_zombie_requests *next = _libssh2_list_next(&zombie->node);
        /* unlink the current one */
        _libssh2_list_remove(&zombie->node);
        /* free the memory */
        LIBSSH2_FREE(session, zombie);
        zombie = next;
    }

}

```

This function appears to handle the FXP_CLOSE command in the context of the SFTP protocol. When an SFTP client connects to an SFTP server, it sends a FXP_CLOSE command to the server to initiate the disconnect operation.

The function takes several arguments:

- `session`: The SFTP session handle
- `handle`: The SFTP handle for the directory or file being transferred
- `data`: The data being transferred in this directory or file

It returns the return code of the SFTP server's FXP_CLOSE command.

The function first checks if the handle has any data to transfer. If it does, it calls the SFTP server's FXP_ACCEPT command to receive the data. If the server returns an error, the function handles it and returns an error code.

If the handle does not have any data to transfer, the function enters an infinite loop that waits for a status message from the server.

If the server returns a success status code, the function closes the SFTP handle and returns the return code. If the server returns an error status code, the function handles it and returns the return code.

If the handle is a directory handle, the function also handles the directory's close operation. It removes the directory from the server's list of open directories and frees the directory's data if it has any.

If the handle is a file handle, the function also handles the file's close operation. It removes the file from the server's list of open files and frees the file's data if it has any.


```cpp
/* sftp_close_handle
 *
 * Close a file or directory handle.
 * Also frees handle resource and unlinks it from the SFTP structure.
 * The handle is no longer usable after return of this function, unless
 * the return value is LIBSSH2_ERROR_EAGAIN in which case this function
 * should be called again.
 */
static int
sftp_close_handle(LIBSSH2_SFTP_HANDLE *handle)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len = handle->handle_len + 13;
    unsigned char *s, *data = NULL;
    int rc = 0;

    if(handle->close_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Closing handle");
        s = handle->close_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!handle->close_packet) {
            handle->close_state = libssh2_NB_state_idle;
            rc = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for FXP_CLOSE "
                                "packet");
        }
        else {

            _libssh2_store_u32(&s, packet_len - 4);
            *(s++) = SSH_FXP_CLOSE;
            handle->close_request_id = sftp->request_id++;
            _libssh2_store_u32(&s, handle->close_request_id);
            _libssh2_store_str(&s, handle->handle, handle->handle_len);
            handle->close_state = libssh2_NB_state_created;
        }
    }

    if(handle->close_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, handle->close_packet,
                                    packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if((ssize_t)packet_len != rc) {
            handle->close_state = libssh2_NB_state_idle;
            rc = _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                "Unable to send FXP_CLOSE command");
        }
        else
            handle->close_state = libssh2_NB_state_sent;

        LIBSSH2_FREE(session, handle->close_packet);
        handle->close_packet = NULL;
    }

    if(handle->close_state == libssh2_NB_state_sent) {
        rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                                 handle->close_request_id, &data,
                                 &data_len, 9);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
            if(data_len > 0) {
                LIBSSH2_FREE(session, data);
            }
            data = NULL;
            _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                           "Packet too short in FXP_CLOSE command");
        }
        else if(rc) {
            _libssh2_error(session, rc,
                           "Error waiting for status message");
        }

        handle->close_state = libssh2_NB_state_sent1;
    }

    if(!data) {
        /* if it reaches this point with data unset, something unwanted
           happened for which we should have set an error code */
        assert(rc);

    }
    else {
        int retcode = _libssh2_ntohu32(data + 5);
        LIBSSH2_FREE(session, data);

        if(retcode != LIBSSH2_FX_OK) {
            sftp->last_errno = retcode;
            handle->close_state = libssh2_NB_state_idle;
            rc = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                "SFTP Protocol Error");
        }
    }

    /* remove this handle from the parent's list */
    _libssh2_list_remove(&handle->node);

    if(handle->handle_type == LIBSSH2_SFTP_HANDLE_DIR) {
        if(handle->u.dir.names_left)
            LIBSSH2_FREE(session, handle->u.dir.names_packet);
    }
    else if(handle->handle_type == LIBSSH2_SFTP_HANDLE_FILE) {
        if(handle->u.file.data)
            LIBSSH2_FREE(session, handle->u.file.data);
    }

    sftp_packetlist_flush(handle);
    sftp->read_state = libssh2_NB_state_idle;

    handle->close_state = libssh2_NB_state_idle;

    LIBSSH2_FREE(session, handle);

    return rc;
}

```

这段代码定义了一个名为`libssh2_sftp_close_handle`的函数，它是`libssh2_sftp_handle`的成员函数。

该函数接收一个`LIBSSH2_SFTP_HANDLE`类型的参数，它是一个SFTP文件或目录句柄。函数首先检查输入参数是否为空，如果是，则返回错误码。然后，函数调用一个名为`sftp_close_handle`的函数，该函数用于关闭文件或目录句柄，并释放相关的资源。最后，函数返回`rc`作为整数，表示SFTP文件或目录句柄关闭的结果。

总之，该函数的作用是关闭一个SFTP文件或目录句柄，并释放相关的资源。


```cpp
/* libssh2_sftp_close_handle
 *
 * Close a file or directory handle
 * Also frees handle resource and unlinks it from the SFTP structure
 */
LIBSSH2_API int
libssh2_sftp_close_handle(LIBSSH2_SFTP_HANDLE *hnd)
{
    int rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session, sftp_close_handle(hnd));
    return rc;
}

```

This is a function in the SFTPDebug类， which is used to handle the process of sending a SFTP unlink command to a remote server via FXP (Extended X11 Forwarding Protocol).

Here's the function signature:
```cppc
int sftp_send_unlink(sftp, remote_server, command, remote_user, remote_port, data,
                        data_len);
```
参数说明：
- `sftp`: the SFTP client session
- `remote_server`: the remote server address
- `command`: the SFTP unlink command
- `remote_user`: the username of the remote user
- `remote_port`: the port number for the remote server
- `data`: the SFTP data that will be sent
- `data_len`: the length of the SFTP data

函数实现：
```cppscss
int sftp_send_unlink(sftp, remote_server, command, remote_user, remote_port, data,
                        data_len) {
   // open the connection to the remote server
   int ret = sftp_open(sftp, remote_server, 2, SFTP_MEM_SET);
   if (ret != 0) {
       return -1;
   }

   // send the SFTP unlink command to the remote server
   int ret = sftp_write(sftp, remote_server, command, sizeof(command),
                         远程服务器端发送数据的长度， 0);
   if (ret != 0) {
       // retry sending the command
       ret = sftp_write(sftp, remote_server, command,
                          sizeof(command), remote服务器端发送数据的长度， 0);
       if (ret != 0) {
           ret = -1;
       }
   }

   // close the connection to the remote server
   sftp_close(sftp, remote_server);

   // return 0 if the command was successfully sent
   return 0;
}
```
解释：

- `sftp_send_unlink`函数接收四个参数，第一个参数是SFTP客户端session，第二个参数是远程服务器地址，第三个参数是SFTP命令字符串，第四个参数是远程用户名和远程端口号，第五个参数是数据数据。
- `sftp_write`函数用于将数据写入远程服务器，`write`函数第二个参数是目标服务器，第三个参数是数据要发送的字节数，第四个参数是当前写入的字节数，第五个参数是用于指示是否在写入数据之前已经有一个缓冲区。


```cpp
/* sftp_unlink
 * Delete a file from the remote server
 */
static int sftp_unlink(LIBSSH2_SFTP *sftp, const char *filename,
                       size_t filename_len)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + filename_len(4) */
    uint32_t packet_len = filename_len + 13;
    unsigned char *s, *data;
    int rc;

    if(sftp->unlink_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Unlinking %s", filename);
        s = sftp->unlink_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->unlink_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_REMOVE "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_REMOVE;
        sftp->unlink_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->unlink_request_id);
        _libssh2_store_str(&s, filename, filename_len);
        sftp->unlink_state = libssh2_NB_state_created;
    }

    if(sftp->unlink_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->unlink_packet,
                                    packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if((ssize_t)packet_len != rc) {
            LIBSSH2_FREE(session, sftp->unlink_packet);
            sftp->unlink_packet = NULL;
            sftp->unlink_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_REMOVE command");
        }
        LIBSSH2_FREE(session, sftp->unlink_packet);
        sftp->unlink_packet = NULL;

        sftp->unlink_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->unlink_request_id, &data,
                             &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP unlink packet too short");
    }
    else if(rc) {
        sftp->unlink_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    sftp->unlink_state = libssh2_NB_state_idle;

    retcode = _libssh2_ntohu32(data + 5);
    LIBSSH2_FREE(session, data);

    if(retcode == LIBSSH2_FX_OK) {
        return 0;
    }
    else {
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
}

```

这段代码定义了一个名为 `libssh2_sftp_unlink_ex` 的函数，属于 libssh2_sftp 库。它的作用是删除远程服务器上指定文件名（包括文件名长度）的副本，并返回结果。

具体来说，函数接受两个参数：`LIBSSH2_SFTP` 类型的 sftp 对象和文件名（不包括长度）。函数内部首先检查 sftp 对象是否为空，如果是，则表示函数无法完成，返回 LIBSSH2_ERROR_BAD_USE。否则，函数调用 sftp 对象的 `_unlink` 函数，将指定的文件从远程服务器上删除。在函数内部，还使用了 BLOCK_ADJUST 函数来允许函数在运行时动态调整 SFTP 块大小，以提高性能。


```cpp
/* libssh2_sftp_unlink_ex
 * Delete a file from the remote server
 */
LIBSSH2_API int
libssh2_sftp_unlink_ex(LIBSSH2_SFTP *sftp, const char *filename,
                       unsigned int filename_len)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_unlink(sftp, filename, filename_len));
    return rc;
}

```



This function appears to handle the case where a SFTP packet with a too-small data length is received from the server. It does this by first freeing any data associated with the packet, and then it sets the last error number to an appropriate value (LIBSSH2_ERROR_NONE) depending on the type of error that occurred.

If an error occurred, the function returns an error code to indicate the cause. If no error occurred, the function returns LIBSSH2_ERROR_NONE.

It is important to note that this function assumes that the SFTP server on the client side is capable of handling the packet and not requiring an SFTP error to be returned.


```cpp
/*
 * sftp_rename
 *
 * Rename a file on the remote server
 */
static int sftp_rename(LIBSSH2_SFTP *sftp, const char *source_filename,
                       unsigned int source_filename_len,
                       const char *dest_filename,
                       unsigned int dest_filename_len, long flags)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    uint32_t packet_len =
        source_filename_len + dest_filename_len + 17 + (sftp->version >=
                                                        5 ? 4 : 0);
    /* packet_len(4) + packet_type(1) + request_id(4) +
       source_filename_len(4) + dest_filename_len(4) + flags(4){SFTP5+) */
    unsigned char *data;
    ssize_t rc;

    if(sftp->version < 2) {
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Server does not support RENAME");
    }

    if(sftp->rename_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Renaming %s to %s",
                       source_filename, dest_filename);
        sftp->rename_s = sftp->rename_packet =
            LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->rename_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_RENAME "
                                  "packet");
        }

        _libssh2_store_u32(&sftp->rename_s, packet_len - 4);
        *(sftp->rename_s++) = SSH_FXP_RENAME;
        sftp->rename_request_id = sftp->request_id++;
        _libssh2_store_u32(&sftp->rename_s, sftp->rename_request_id);
        _libssh2_store_str(&sftp->rename_s, source_filename,
                           source_filename_len);
        _libssh2_store_str(&sftp->rename_s, dest_filename, dest_filename_len);

        if(sftp->version >= 5)
            _libssh2_store_u32(&sftp->rename_s, flags);

        sftp->rename_state = libssh2_NB_state_created;
    }

    if(sftp->rename_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->rename_packet,
                                    sftp->rename_s - sftp->rename_packet);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if((ssize_t)packet_len != rc) {
            LIBSSH2_FREE(session, sftp->rename_packet);
            sftp->rename_packet = NULL;
            sftp->rename_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_RENAME command");
        }
        LIBSSH2_FREE(session, sftp->rename_packet);
        sftp->rename_packet = NULL;

        sftp->rename_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->rename_request_id, &data,
                             &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rename packet too short");
    }
    else if(rc) {
        sftp->rename_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    sftp->rename_state = libssh2_NB_state_idle;

    retcode = _libssh2_ntohu32(data + 5);
    LIBSSH2_FREE(session, data);

    sftp->last_errno = retcode;

    /* now convert the SFTP error code to libssh2 return code or error
       message */
    switch(retcode) {
    case LIBSSH2_FX_OK:
        retcode = LIBSSH2_ERROR_NONE;
        break;

    case LIBSSH2_FX_FILE_ALREADY_EXISTS:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "File already exists and "
                                 "SSH_FXP_RENAME_OVERWRITE not specified");
        break;

    case LIBSSH2_FX_OP_UNSUPPORTED:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "Operation Not Supported");
        break;

    default:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "SFTP Protocol Error");
        break;
    }

    return retcode;
}

```

这段代码定义了一个名为 `libssh2_sftp_rename_ex` 的函数，属于 `libssh2_sftp` 系列的库。
这个函数的作用是将在本地服务器上文件的本地路径作为参数的远程服务器上的文件名。
函数接受四个参数：
1. `sftp`：指向 SSH2 SFTP 客户端的指针。
2. `source_filename`：要递归的文件在远程服务器上的路径名。
3. `source_filename_len`：`source_filename` 的长度，单位是字节数。
4. `dest_filename`：目标文件名，用于将文件从远程服务器上复制到本地。
5. `dest_filename_len`：`dest_filename` 的长度，单位是字节数。
6. `flags`：设置 SFTP 标志的参数，其中包含 `LIBSSH2_FLAG_INSERT_FLAGS` 和 `LIBSSH2_FLAG_RETR_FLAGS`，用于在递归复制时插入文件头和还原文件头。

函数首先检查 `sftp` 是否为空，如果是，函数返回 `LIBSSH2_ERROR_BAD_USE`，表示使用错误。

函数调用 `sftp_rename` 函数，将文件从远程服务器上复制到本地。这个函数的实现超出了 `libssh2_sftp` 的文档，因此需要预先了解相关代码。


```cpp
/* libssh2_sftp_rename_ex
 * Rename a file on the remote server
 */
LIBSSH2_API int
libssh2_sftp_rename_ex(LIBSSH2_SFTP *sftp, const char *source_filename,
                       unsigned int source_filename_len,
                       const char *dest_filename,
                       unsigned int dest_filename_len, long flags)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_rename(sftp, source_filename, source_filename_len,
                             dest_filename, dest_filename_len, flags));
    return rc;
}

```

This function appears to be processing an SFTP file transfer to a server, using the SSH2 protocol. It starts by in an idle state, and then sets the file system state to also be in an idle state.

It then sets the file transfer to a blocking state, and sets the output buffer's file size and remaining space information.

It continues to process the SFTP file transfer by doing various checks, and then sets the output buffer's file attributes and user flags.

It also sets the name of the file being transferred and sets a flag indicating whether the file is a regular file (RFC-2571) or a special file (RFC-2572).

After all the processing is done, it returns an error code.


```cpp
/*
 * sftp_fstatvfs
 *
 * Get file system statistics
 */
static int sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_STATVFS *st)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 17 = packet_len(4) + packet_type(1) + request_id(4) + ext_len(4)
       + handle_len (4) */
    /* 20 = strlen ("fstatvfs@openssh.com") */
    uint32_t packet_len = handle->handle_len + 20 + 17;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    unsigned int flag;
    static const unsigned char responses[2] =
        { SSH_FXP_EXTENDED_REPLY, SSH_FXP_STATUS };

    if(sftp->fstatvfs_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Getting file system statistics");
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_EXTENDED "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_EXTENDED;
        sftp->fstatvfs_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->fstatvfs_request_id);
        _libssh2_store_str(&s, "fstatvfs@openssh.com", 20);
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        sftp->fstatvfs_state = libssh2_NB_state_created;
    }
    else {
        packet = sftp->fstatvfs_packet;
    }

    if(sftp->fstatvfs_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN ||
            (0 <= rc && rc < (ssize_t)packet_len)) {
            sftp->fstatvfs_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }

        LIBSSH2_FREE(session, packet);
        sftp->fstatvfs_packet = NULL;

        if(rc < 0) {
            sftp->fstatvfs_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        sftp->fstatvfs_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_requirev(sftp, 2, responses, sftp->fstatvfs_request_id,
                              &data, &data_len, 9);

    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rename packet too short");
    }
    else if(rc) {
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    if(data[0] == SSH_FXP_STATUS) {
        int retcode = _libssh2_ntohu32(data + 5);
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        LIBSSH2_FREE(session, data);
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }

    if(data_len < 93) {
        LIBSSH2_FREE(session, data);
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error: short response");
    }

    sftp->fstatvfs_state = libssh2_NB_state_idle;

    st->f_bsize = _libssh2_ntohu64(data + 5);
    st->f_frsize = _libssh2_ntohu64(data + 13);
    st->f_blocks = _libssh2_ntohu64(data + 21);
    st->f_bfree = _libssh2_ntohu64(data + 29);
    st->f_bavail = _libssh2_ntohu64(data + 37);
    st->f_files = _libssh2_ntohu64(data + 45);
    st->f_ffree = _libssh2_ntohu64(data + 53);
    st->f_favail = _libssh2_ntohu64(data + 61);
    st->f_fsid = _libssh2_ntohu64(data + 69);
    flag = (unsigned int)_libssh2_ntohu64(data + 77);
    st->f_namemax = _libssh2_ntohu64(data + 85);

    st->f_flag = (flag & SSH_FXE_STATVFS_ST_RDONLY)
        ? LIBSSH2_SFTP_ST_RDONLY : 0;
    st->f_flag |= (flag & SSH_FXE_STATVFS_ST_NOSUID)
        ? LIBSSH2_SFTP_ST_NOSUID : 0;

    LIBSSH2_FREE(session, data);
    return 0;
}

```

这段代码定义了一个名为`libssh2_sftp_fstatvfs`的函数，属于`libssh2_sftp_pkg`包。它的作用是获取文件系统的空间和inode利用率。它依赖于`fstatvfs`函数，这个函数在服务器上支持，因此需要通过服务器上的支持来加载。

函数接受两个参数，一个是`LIBSSH2_SFTP_HANDLE`类型的输入指针`handle`，另一个是`LIBSSH2_SFTP_STATVFS`类型的输出指针`st`。函数内部首先检查输入指针`handle`是否为0或无效，然后执行一个名为`sftp_fstatvfs`的函数，并将结果存储在输出指针`st`指向的内存区域。

函数的最后部分定义了一个返回值，用于表示函数执行成功或失败的结果。如果函数执行成功，返回0；如果执行失败，返回一个负的错误代码。


```cpp
/* libssh2_sftp_fstatvfs
 * Get filesystem space and inode utilization (requires fstatvfs@openssh.com
 * support on the server)
 */
LIBSSH2_API int
libssh2_sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_STATVFS *st)
{
    int rc;
    if(!handle || !st)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, handle->sftp->channel->session,
                 sftp_fstatvfs(handle, st));
    return rc;
}

```

This is a function definition for `_libssh2_ntohlookup`. It appears to be a helper function for the `libssh2_lookup_file` function, which looks up the contents of an SFTP file.

The function takes a single parameter, `filename`, which is a file name in the SFTP file format. It returns the file data, as a buffer of `ssh_msgsymm` data types.

The function first sets the state of the SFTP file to `libssh2_NB_state_idle`. This is the default state for an SFTP file, and indicates that the file has not yet been completely downloaded.

It then sets the `f_bsize`, `f_frsize`, `f_blocks`, `f_bfree`, `f_bavail`, `f_files`, `f_ffree`, `f_favail`, `f_fsid`, `f_namemax`, and `f_flag` members of the `st` structure, which represents the SFTP file data.

The function sets the `f_bsize` to the length of the file data, `f_frsize` to the number of frames in the file, `f_blocks` to the number of blocks in the file, `f_bfree` to the number of free blocks in the file, `f_bavail` to the number of bytes available in the file, `f_files` to the number of files in the SFTP file, `f_ffree` to the number of free frames in the file, `f_favail` to the number of bytes available in the file, `f_fsid` to the file's SFTP file ID, `f_namemax` to the name of the file, and `f_flag` to a flag indicating the file's state and whether it is read-only.

It then calls the `_libssh2_lookup_file` function with the `filename` parameter and the `st` structure as arguments.

The function returns the file data in the format `ssh_msgsymm`


```cpp
/*
 * sftp_statvfs
 *
 * Get file system statistics
 */
static int sftp_statvfs(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, LIBSSH2_SFTP_STATVFS *st)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 17 = packet_len(4) + packet_type(1) + request_id(4) + ext_len(4)
       + path_len (4) */
    /* 19 = strlen ("statvfs@openssh.com") */
    uint32_t packet_len = path_len + 19 + 17;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    unsigned int flag;
    static const unsigned char responses[2] =
        { SSH_FXP_EXTENDED_REPLY, SSH_FXP_STATUS };

    if(sftp->statvfs_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Getting file system statistics of %s", path);
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_EXTENDED "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_EXTENDED;
        sftp->statvfs_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->statvfs_request_id);
        _libssh2_store_str(&s, "statvfs@openssh.com", 19);
        _libssh2_store_str(&s, path, path_len);

        sftp->statvfs_state = libssh2_NB_state_created;
    }
    else {
        packet = sftp->statvfs_packet;
    }

    if(sftp->statvfs_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN ||
            (0 <= rc && rc < (ssize_t)packet_len)) {
            sftp->statvfs_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }

        LIBSSH2_FREE(session, packet);
        sftp->statvfs_packet = NULL;

        if(rc < 0) {
            sftp->statvfs_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        sftp->statvfs_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_requirev(sftp, 2, responses, sftp->statvfs_request_id,
                              &data, &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fstat packet too short");
    }
    else if(rc) {
        sftp->statvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    if(data[0] == SSH_FXP_STATUS) {
        int retcode = _libssh2_ntohu32(data + 5);
        sftp->statvfs_state = libssh2_NB_state_idle;
        LIBSSH2_FREE(session, data);
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }

    if(data_len < 93) {
        LIBSSH2_FREE(session, data);
        sftp->statvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error: short response");
    }

    sftp->statvfs_state = libssh2_NB_state_idle;

    st->f_bsize = _libssh2_ntohu64(data + 5);
    st->f_frsize = _libssh2_ntohu64(data + 13);
    st->f_blocks = _libssh2_ntohu64(data + 21);
    st->f_bfree = _libssh2_ntohu64(data + 29);
    st->f_bavail = _libssh2_ntohu64(data + 37);
    st->f_files = _libssh2_ntohu64(data + 45);
    st->f_ffree = _libssh2_ntohu64(data + 53);
    st->f_favail = _libssh2_ntohu64(data + 61);
    st->f_fsid = _libssh2_ntohu64(data + 69);
    flag = (unsigned int)_libssh2_ntohu64(data + 77);
    st->f_namemax = _libssh2_ntohu64(data + 85);

    st->f_flag = (flag & SSH_FXE_STATVFS_ST_RDONLY)
        ? LIBSSH2_SFTP_ST_RDONLY : 0;
    st->f_flag |= (flag & SSH_FXE_STATVFS_ST_NOSUID)
        ? LIBSSH2_SFTP_ST_NOSUID : 0;

    LIBSSH2_FREE(session, data);
    return 0;
}

```

这段代码定义了一个名为 `libssh2_sftp_statvfs` 的函数，属于 libssh2_sftp 库。它的作用是获取文件系统空间和 inode 利用率，requires statvfs@openssh.com support on the server。

具体来说，这段代码实现以下几个步骤：

1. 检查输入参数是否为空，如果是，则返回 LIBSSH2_ERROR_BAD_USE。
2. 如果输入参数 `sftp` 和 `st` 都不为空，则执行以下操作：

  a. 如果 `sftp` 是 SFTP 类型，执行 `sftp_statvfs` 函数，并且将 `path` 和 `path_len` 参数传递给它，得到一个 `LIBSSH2_SFTP_STATVFS` 类型的变量 `st`。

  b. 如果 `sftp` 是 SFTP 类型，执行 `statvfs` 函数，并且将 `path` 和 `path_len` 参数传递给它，得到一个 `LIBSSH2_SFTP_STATVFS` 类型的变量 `st`。

  c. 使用 `BLOCK_ADJUST` 函数来调整块大小，这个函数的作用是确保块大小足够大，以满足 SFTP 连接的要求。这个函数的实现可能因 SFTP 实现而异。

  d. 返回 `rc`，即 libssh2_sftp_statvfs 的出错码。

这段代码是用来执行 SFTP 连接的文件系统调用，它会获取文件系统空间和 inode 利用率，并在需要时执行 `sftp_statvfs` 和 `statvfs` 函数，这两个函数的具体实现可以在上面的分析中找到。


```cpp
/* libssh2_sftp_statvfs_ex
 * Get filesystem space and inode utilization (requires statvfs@openssh.com
 * support on the server)
 */
LIBSSH2_API int
libssh2_sftp_statvfs(LIBSSH2_SFTP *sftp, const char *path,
                     size_t path_len, LIBSSH2_SFTP_STATVFS *st)
{
    int rc;
    if(!sftp || !st)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session, sftp_statvfs(sftp, path, path_len,
                                                          st));
    return rc;
}


```

This function appears to be a part of the SSH2 protocol, and it is used to send a file to an SFTP (Secure Shell Remote Device) server.

It takes in a session object, a packet string, and a file path. The function first checks if the SFTP session is active and in the correct state. If the session is active and in the correct state, it then sends the file to the SFTP server using the `send_file` function.

If the session is not active or in the wrong state, the function returns an error and any error handling that is needed.

It appears that the function has error handling for issues such as duplicate file identifier and buffer too small errors. It also sends the file to the SFTP server using the `send_file` function and upon successful delivery it logs a message.


```cpp
/*
 * sftp_mkdir
 *
 * Create an SFTP directory
 */
static int sftp_mkdir(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len, long mode)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_SFTP_ATTRIBUTES attrs = {
        0, 0, 0, 0, 0, 0, 0
    };
    size_t data_len;
    int retcode;
    ssize_t packet_len;
    unsigned char *packet, *s, *data;
    int rc;

    if(mode != LIBSSH2_SFTP_DEFAULT_MODE) {
        /* Filetype in SFTP 3 and earlier */
        attrs.flags = LIBSSH2_SFTP_ATTR_PERMISSIONS;
        attrs.permissions = mode | LIBSSH2_SFTP_ATTR_PFILETYPE_DIR;
    }

    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    packet_len = path_len + 13 + sftp_attrsize(attrs.flags);

    if(sftp->mkdir_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Creating directory %s with mode 0%lo", path, mode);
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_MKDIR "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_MKDIR;
        sftp->mkdir_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->mkdir_request_id);
        _libssh2_store_str(&s, path, path_len);

        s += sftp_attr2bin(s, &attrs);

        sftp->mkdir_state = libssh2_NB_state_created;
    }
    else {
        packet = sftp->mkdir_packet;
    }

    if(sftp->mkdir_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            sftp->mkdir_packet = packet;
            return rc;
        }
        if(packet_len != rc) {
            LIBSSH2_FREE(session, packet);
            sftp->mkdir_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        LIBSSH2_FREE(session, packet);
        sftp->mkdir_state = libssh2_NB_state_sent;
        sftp->mkdir_packet = NULL;
    }

    rc = sftp_packet_require(sftp, SSH_FXP_STATUS, sftp->mkdir_request_id,
                             &data, &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP mkdir packet too short");
    }
    else if(rc) {
        sftp->mkdir_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    sftp->mkdir_state = libssh2_NB_state_idle;

    retcode = _libssh2_ntohu32(data + 5);
    LIBSSH2_FREE(session, data);

    if(retcode == LIBSSH2_FX_OK) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "OK!");
        return 0;
    }
    else {
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
}

```

这段代码定义了一个名为 `libssh2_sftp_mkdir_ex` 的函数，属于名为 `libssh2_sftp` 的库函数。

这个函数的作用是创建一个 SFTP 目录，如果已存在相同路径的目录，则不会创建。函数的第一个参数是一个 SFTP 对象，第二个参数是一个字符指针，指定了要创建的目录路径，第三个参数是路径长度，第四个参数是一个模式，指定了 SFTP 目录的权限模式。函数返回一个整数，表示 SFTP 命令执行结果。

函数的实现基于 `libssh2_sftp` 和 `libssh2_api` 库函数。首先，函数检查传入的参数 `sftp` 是否为空，如果是，则返回错误代码。然后，函数使用 `sftp_mkdir` 函数在 SFTP 通道中创建目录，并将 `rc` 赋值为 `LIBSSH2_ERROR_BAD_USE`，表示错误。如果 `rc` 值为0，说明创建目录成功。


```cpp
/*
 * libssh2_sftp_mkdir_ex
 *
 * Create an SFTP directory
 */
LIBSSH2_API int
libssh2_sftp_mkdir_ex(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len, long mode)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_mkdir(sftp, path, path_len, mode));
    return rc;
}

```

This function appears to handle the sending of the FXP_RMDIR command to a remote SFTP server. It does this by first trying to send a regular SFTP packet to the server, using the information in the command. If that fails, it attempts to send a packet with theFXP_RMDIR command again, using the last error returned by the previous attempt.

The function takes in a session object, a SFTP packet object, and a pointer to an SFTP data packet. It returns 0 on success, or the last error returned by the server in case of an error.

It is保护性循环调用库函数，如果出现错误，可以通过最后一个错误重新开始尝试，直到成功或最后调用失败。


```cpp
/* sftp_rmdir
 * Remove a directory
 */
static int sftp_rmdir(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    ssize_t packet_len = path_len + 13;
    unsigned char *s, *data;
    int rc;

    if(sftp->rmdir_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Removing directory: %s",
                       path);
        s = sftp->rmdir_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->rmdir_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_RMDIR "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_RMDIR;
        sftp->rmdir_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->rmdir_request_id);
        _libssh2_store_str(&s, path, path_len);

        sftp->rmdir_state = libssh2_NB_state_created;
    }

    if(sftp->rmdir_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->rmdir_packet,
                                    packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(packet_len != rc) {
            LIBSSH2_FREE(session, sftp->rmdir_packet);
            sftp->rmdir_packet = NULL;
            sftp->rmdir_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_RMDIR command");
        }
        LIBSSH2_FREE(session, sftp->rmdir_packet);
        sftp->rmdir_packet = NULL;

        sftp->rmdir_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->rmdir_request_id, &data, &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rmdir packet too short");
    }
    else if(rc) {
        sftp->rmdir_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    sftp->rmdir_state = libssh2_NB_state_idle;

    retcode = _libssh2_ntohu32(data + 5);
    LIBSSH2_FREE(session, data);

    if(retcode == LIBSSH2_FX_OK) {
        return 0;
    }
    else {
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
}

```

这段代码是一个名为 `libssh2_sftp_rmdir_ex` 的函数，属于 SSH2 Sftp 客户端库。它的作用是删除指定路径名称（包含子目录和文件）的目录及其子目录，并返回状态码。

具体来说，函数接收三个参数：`sftp` 是 SSH2 Sftp 客户端句柄，`path` 是要删除的目录名称，`path_len` 是目录名称的长度（不包括子目录和文件）。函数首先检查输入参数 `sftp` 是否为空，如果是，则返回一个错误码。否则，函数调用 `sftp_rmdir` 函数，传递输入参数 `sftp` 和 `path`，并调整 Block 调整量以确保所有安全操作。最后，函数返回状态码。


```cpp
/* libssh2_sftp_rmdir_ex
 * Remove a directory
 */
LIBSSH2_API int
libssh2_sftp_rmdir_ex(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_rmdir(sftp, path, path_len));
    return rc;
}

```

This is a function in the Socket and绝望 SSH2 library for the Java language that handles the SFTP protocol. It is used to handle the sending of a stat packet to an SFTP server.

The function takes in a pointer to the data buffer and a pointer to the data buffer size. It first checks if the data buffer is within a minimum size of 4 bytes. If the data buffer is too small, an error is thrown.

It then sends the data buffer to the SFTP server using the SFTP protocol. If there is an error sending the data, an error is thrown.

It then sets the state of the SFTP client to idle and returns.

If the server responds with a status code, the function extracts the status code and returns it.

It is important to note that the function has a variable name `attrs` which is supposed to be initialized as an empty array of bytes. It is passed as an argument to the `sftp_bin2attr` function, which is used to convert the attributes of an SFTP file system object to an array of bytes.

It is also important to note that the function has a variable name `data_len` which is passed as an argument to the function `sftp_bin2attr` and is used to pass the SFTP file system object's data buffer to the `sftp_bin2attr` function.


```cpp
/* sftp_stat
 * Stat a file or symbolic link
 */
static int sftp_stat(LIBSSH2_SFTP *sftp, const char *path,
                     unsigned int path_len, int stat_type,
                     LIBSSH2_SFTP_ATTRIBUTES * attrs)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    ssize_t packet_len =
        path_len + 13 +
        ((stat_type ==
          LIBSSH2_SFTP_SETSTAT) ? sftp_attrsize(attrs->flags) : 0);
    unsigned char *s, *data;
    static const unsigned char stat_responses[2] =
        { SSH_FXP_ATTRS, SSH_FXP_STATUS };
    int rc;

    if(sftp->stat_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%s %s",
                       (stat_type == LIBSSH2_SFTP_SETSTAT) ? "Set-statting" :
                       (stat_type ==
                        LIBSSH2_SFTP_LSTAT ? "LStatting" : "Statting"), path);
        s = sftp->stat_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->stat_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_*STAT "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);

        switch(stat_type) {
        case LIBSSH2_SFTP_SETSTAT:
            *(s++) = SSH_FXP_SETSTAT;
            break;

        case LIBSSH2_SFTP_LSTAT:
            *(s++) = SSH_FXP_LSTAT;
            break;

        case LIBSSH2_SFTP_STAT:
        default:
            *(s++) = SSH_FXP_STAT;
        }
        sftp->stat_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->stat_request_id);
        _libssh2_store_str(&s, path, path_len);

        if(stat_type == LIBSSH2_SFTP_SETSTAT)
            s += sftp_attr2bin(s, attrs);

        sftp->stat_state = libssh2_NB_state_created;
    }

    if(sftp->stat_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, sftp->stat_packet, packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(packet_len != rc) {
            LIBSSH2_FREE(session, sftp->stat_packet);
            sftp->stat_packet = NULL;
            sftp->stat_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send STAT/LSTAT/SETSTAT command");
        }
        LIBSSH2_FREE(session, sftp->stat_packet);
        sftp->stat_packet = NULL;

        sftp->stat_state = libssh2_NB_state_sent;
    }

    rc = sftp_packet_requirev(sftp, 2, stat_responses,
                              sftp->stat_request_id, &data, &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP stat packet too short");
    }
    else if(rc) {
        sftp->stat_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Timeout waiting for status message");
    }

    sftp->stat_state = libssh2_NB_state_idle;

    if(data[0] == SSH_FXP_STATUS) {
        int retcode;

        retcode = _libssh2_ntohu32(data + 5);
        LIBSSH2_FREE(session, data);
        if(retcode == LIBSSH2_FX_OK) {
            memset(attrs, 0, sizeof(LIBSSH2_SFTP_ATTRIBUTES));
            return 0;
        }
        else {
            sftp->last_errno = retcode;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    memset(attrs, 0, sizeof(LIBSSH2_SFTP_ATTRIBUTES));
    if(sftp_bin2attr(attrs, data + 5, data_len - 5) < 0) {
        LIBSSH2_FREE(session, data);
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Attributes too short in SFTP fstat");
    }

    LIBSSH2_FREE(session, data);

    return 0;
}

```

这段代码定义了一个名为 `libssh2_sftp_stat_ex` 的函数，属于 libssh2_sftp 库。

这个函数的作用是统计指定路径下文件或符号链接的属性，然后返回结果。以下是函数的实现细节：

1. 如果 `sftp` 参数为 `NULL`，则返回 LIBSSH2_ERROR_BAD_USE。
2. 函数首先检查 `sftp` 是否为 `NULL`，如果是，则直接返回。
3. 如果 `sftp` 不是 `NULL`，那么它需要调用 `sftp_stat` 函数来完成统计。这个函数需要传递一个 `LIBSSH2_SFTP` 类型的对象，一个指向文件的路径名，以及一个表示文件大小的指示符 `stat_type`，还需要传递一个指向 `LIBSSH2_SFTP_ATTRIBUTES` 类型的指针，这个类型定义了需要返回的属性信息。
4. `sftp_stat` 函数的实现如下：
```cppperl
int libssh2_sftp_stat(LIBSSH2_SFTP sftp, const char *path,
                        unsigned int path_len, int stat_type,
                        LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
   int ret_val;
   ret_val = stat_file(sftp, path, path_len, stat_type, attrs);
   if (ret_val < 0) {
       return ret_val;
   }
   return 0;
}
```
这个实现中，`stat_file` 函数是统计文件的属性，它会根据 `stat_type` 参数来决定要返回哪些属性信息。然后，函数调用 `stat_file` 并传递给它的参数，包括 `sftp`、路径名和 `stat_type`，以及一个指向 `LIBSSH2_SFTP_ATTRIBUTES` 类型的指针，这个指针定义了需要返回的属性信息。

总之，这个函数接受一个 `LIBSSH2_SFTP` 类型的对象，一个指向文件的路径名，以及一个表示文件大小的指示符 `stat_type`。它返回一个整数，表示文件属性的统计结果。如果函数在调用 `sftp_stat` 函数时遇到错误，函数将返回 LIBSSH2_ERROR_BAD_USE。


```cpp
/* libssh2_sftp_stat_ex
 * Stat a file or symbolic link
 */
LIBSSH2_API int
libssh2_sftp_stat_ex(LIBSSH2_SFTP *sftp, const char *path,
                     unsigned int path_len, int stat_type,
                     LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_stat(sftp, path, path_len, stat_type, attrs));
    return rc;
}

```



This is a function definition for `_libssh2_ntohl32` which converts a 32-bit value from a raw byte array to a signed 32-bit value. The input argument is a raw byte array `data`, and the output is a signed 32-bit value `target`.

This function is used in the `libssh2_session` library to convert the SFTP `u32` status packet to a signed 32-bit value for storing in the `target` array. The `link_len` field in the status packet is used to determine the length of the `target` array.

If the `link_len` is too small, the `libssh2_error` function will be called with the error code `LIBSSH2_ERROR_BUFFER_TOO_SMALL`.


```cpp
/* sftp_symlink
 * Read or set a symlink
 */
static int sftp_symlink(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, char *target,
                        unsigned int target_len, int link_type)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len, link_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    ssize_t packet_len =
        path_len + 13 +
        ((link_type == LIBSSH2_SFTP_SYMLINK) ? (4 + target_len) : 0);
    unsigned char *s, *data;
    static const unsigned char link_responses[2] =
        { SSH_FXP_NAME, SSH_FXP_STATUS };
    int retcode;

    if((sftp->version < 3) && (link_type != LIBSSH2_SFTP_REALPATH)) {
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Server does not support SYMLINK or READLINK");
    }

    if(sftp->symlink_state == libssh2_NB_state_idle) {
        s = sftp->symlink_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->symlink_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "SYMLINK/READLINK/REALPATH packet");
        }

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%s %s on %s",
                       (link_type ==
                        LIBSSH2_SFTP_SYMLINK) ? "Creating" : "Reading",
                       (link_type ==
                        LIBSSH2_SFTP_REALPATH) ? "realpath" : "symlink", path);

        _libssh2_store_u32(&s, packet_len - 4);

        switch(link_type) {
        case LIBSSH2_SFTP_REALPATH:
            *(s++) = SSH_FXP_REALPATH;
            break;

        case LIBSSH2_SFTP_SYMLINK:
            *(s++) = SSH_FXP_SYMLINK;
            break;

        case LIBSSH2_SFTP_READLINK:
        default:
            *(s++) = SSH_FXP_READLINK;
        }
        sftp->symlink_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->symlink_request_id);
        _libssh2_store_str(&s, path, path_len);

        if(link_type == LIBSSH2_SFTP_SYMLINK)
            _libssh2_store_str(&s, target, target_len);

        sftp->symlink_state = libssh2_NB_state_created;
    }

    if(sftp->symlink_state == libssh2_NB_state_created) {
        ssize_t rc = _libssh2_channel_write(channel, 0, sftp->symlink_packet,
                                            packet_len);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        else if(packet_len != rc) {
            LIBSSH2_FREE(session, sftp->symlink_packet);
            sftp->symlink_packet = NULL;
            sftp->symlink_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send SYMLINK/READLINK command");
        }
        LIBSSH2_FREE(session, sftp->symlink_packet);
        sftp->symlink_packet = NULL;

        sftp->symlink_state = libssh2_NB_state_sent;
    }

    retcode = sftp_packet_requirev(sftp, 2, link_responses,
                                   sftp->symlink_request_id, &data,
                                   &data_len, 9);
    if(retcode == LIBSSH2_ERROR_EAGAIN)
        return retcode;
    else if(retcode == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP symlink packet too short");
    }
    else if(retcode) {
        sftp->symlink_state = libssh2_NB_state_idle;
        return _libssh2_error(session, retcode,
                              "Error waiting for status message");
    }

    sftp->symlink_state = libssh2_NB_state_idle;

    if(data[0] == SSH_FXP_STATUS) {
        retcode = _libssh2_ntohu32(data + 5);
        LIBSSH2_FREE(session, data);
        if(retcode == LIBSSH2_FX_OK)
            return LIBSSH2_ERROR_NONE;
        else {
            sftp->last_errno = retcode;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    if(_libssh2_ntohu32(data + 5) < 1) {
        LIBSSH2_FREE(session, data);
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Invalid READLINK/REALPATH response, "
                              "no name entries");
    }

    if(data_len < 13) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP stat packet too short");
    }

    /* this reads a u32 and stores it into a signed 32bit value */
    link_len = _libssh2_ntohu32(data + 9);
    if(link_len < target_len) {
        memcpy(target, data + 13, link_len);
        target[link_len] = 0;
        retcode = (int)link_len;
    }
    else
        retcode = LIBSSH2_ERROR_BUFFER_TOO_SMALL;
    LIBSSH2_FREE(session, data);

    return retcode;
}

```

这段代码是一个用于在SSH2 Sftp协议中读取或设置符号链接的函数。函数接收三个参数：一个SSH2 Sftp对象、一个路径名称和目标路径，以及一个目标路径长度和链接类型。函数首先检查输入是否为空，如果是，则返回LIBSSH2_ERROR_BAD_USE错误。然后，函数将块调整到适当的适度，然后使用sftp_symlink函数将符号链接设置为指定的路径和目标，并将结果返回。


```cpp
/* libssh2_sftp_symlink_ex
 * Read or set a symlink
 */
LIBSSH2_API int
libssh2_sftp_symlink_ex(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, char *target,
                        unsigned int target_len, int link_type)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_symlink(sftp, path, path_len, target, target_len,
                              link_type));
    return rc;
}

```

这段代码定义了一个名为`libssh2_sftp_last_error`的函数，它接受一个`LIBSSH2_SFTP`类型的参数。这个函数的作用是返回SFTP最近的错误码，以便用户在处理SFTP数据时知道可能的错误情况。

函数的实现首先检查`sftp`是否为空，如果是，则直接返回0，表示没有错误。否则，函数调用`sftp->last_errno`获取错误码，并将其作为函数的返回值。

接下来，定义了一个名为`libssh2_sftp_get_channel`的函数，它接受一个`LIBSSH2_SFTP`类型的参数。这个函数的作用是获取SFTP的当前通道，并将其返回。调用者可以根据自己的需要控制通道的行为，比如设置目标通道、处理连接状态等等。


```cpp
/* libssh2_sftp_last_error
 * Returns the last error code reported by SFTP
 */
LIBSSH2_API unsigned long
libssh2_sftp_last_error(LIBSSH2_SFTP *sftp)
{
    if(!sftp)
        return 0;

    return sftp->last_errno;
}

/* libssh2_sftp_get_channel
 * Return the channel of sftp, then caller can control the channel's behavior.
 */
```

这段代码定义了一个名为`libssh2_sftp_get_channel`的函数，它接受一个`LIBSSH2_SFTP`类型的参数。这个函数返回一个指向`LIBSSH2_CHANNEL`类型的指针，它用于访问SFTP数据传输通道。

函数首先检查`sftp`参数是否为空，如果是，则返回`NULL`。否则，它调用`sftp->channel`获取数据传输通道。注意，`sftp`必须是一个`LIBSSH2_SFTP`类型的对象才能调用这个函数。


```cpp
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp)
{
    if(!sftp)
        return NULL;

    return sftp->channel;
}

```