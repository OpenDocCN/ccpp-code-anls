# Nmap源码解析 97

# `libssh2/src/keepalive.c`

这段代码是一个C语言的函数，定义了一个名为"output"的函数。但这个函数在实现时并不会输出任何东西，它仅仅是在定义时对函数进行了说明。具体来说，这个函数的实现会在需要时产生一个字符串，其中包含了printf函数的一些信息，例如打印的格式、可变参数、如何处理输入参数等。这些信息对程序员来说非常重要，但程序不会因此获得任何输出。


```cpp
/* Copyright (C) 2010  Simon Josefsson
 * Author: Simon Josefsson
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
 *
 */

```

这段代码是针对libssh2_session类型的库函数，用于在SSH会话中设置保持活动状态（即客户端与服务器之间的连接状态）。

首先，在函数头部包含libssh2_priv.h和transport.h头文件，前者是SSH2客户端的私用头文件，后者是传输层的头文件，定义了一些与libssh2_session类型相关的函数和结构体。

函数体中包含两个主要函数：

1. libssh2_keepalive_config函数：该函数根据客户端发送的轮询类型（1表示客户端发送轮询请求，2表示客户端发送快照请求）和设置的轮询时间间隔（单位为秒）来决定客户端是否要保持与服务器的连接状态。如果轮询间隔为1，那么客户端在收到第一个轮询请求后，就保持与服务器的连接状态，否则在轮询时间间隔结束后，客户端重新发送连接请求。如果客户端发送了轮询请求，那么函数中将轮询设置为1，并将保持活动状态设置为需要回复客户端的轮询（即1）。

2. 在main函数中包含一些函数：

- 初始化libssh2_session结构的函数：这个函数在创建SSH会话时调用，创建一个新的libssh2_session结构，并调用libssh2_connect函数进行初始化。
- 调用libssh2_keepalive_config函数：这个函数在用户调用libssh2_connect函数成功后调用，设置libssh2_session的轮询间隔和保持活动状态。
- 设置轮询的时间间隔为1：这个函数在每次调用libssh2_keepalive_config函数时调用，确保每次libssh2_connect函数都有一个新的轮询。


```cpp
#include "libssh2_priv.h"
#include "transport.h" /* _libssh2_transport_write */

/* Keep-alive stuff. */

LIBSSH2_API void
libssh2_keepalive_config (LIBSSH2_SESSION *session,
                          int want_reply,
                          unsigned interval)
{
    if(interval == 1)
        session->keepalive_interval = 2;
    else
        session->keepalive_interval = interval;
    session->keepalive_want_reply = want_reply ? 1 : 0;
}

```



This function is a part of the SSH library, which allows users to connect to an SSH server and execute commands on the server.

The function `LIBSSH2_SESSION *session, int *seconds_to_next)`

This function is used to manage the keepalive functionality of the SSH connection.

The keepalive feature allows the server to send a message to the client periodically to check if the connection is still active.

The function first sets up a check for the keepalive interval and then checks if the server has any pending keepalive messages.

If there are no pending messages and the user has specified a keepalive interval, the server will send a keepalive message to the client as soon as possible.

If the server has pending keepalive messages, the function compares the current time with the timestamp of the last sent keepalive message.

If the difference between the current time and the timestamp of the last sent keepalive message is within the specified interval, the function sends a keepalive message to the client and updates the timestamp of the last sent keepalive message.

If the server has not sent any pending keepalive messages and the user has specified a keepalive interval, the server will send a keepalive message to the client after the specified interval.

The function also includes a check for the case where the keepalive interval has not been specified or the server does not have any pending keepalive messages.


```cpp
LIBSSH2_API int
libssh2_keepalive_send (LIBSSH2_SESSION *session,
                        int *seconds_to_next)
{
    time_t now;

    if(!session->keepalive_interval) {
        if(seconds_to_next)
            *seconds_to_next = 0;
        return 0;
    }

    now = time(NULL);

    if(session->keepalive_last_sent + session->keepalive_interval <= now) {
        /* Format is
           "SSH_MSG_GLOBAL_REQUEST || 4-byte len || str || want-reply". */
        unsigned char keepalive_data[]
            = "\x50\x00\x00\x00\x15keepalive@libssh2.orgW";
        size_t len = sizeof(keepalive_data) - 1;
        int rc;

        keepalive_data[len - 1] =
            (unsigned char)session->keepalive_want_reply;

        rc = _libssh2_transport_send(session, keepalive_data, len, NULL, 0);
        /* Silently ignore PACKET_EAGAIN here: if the write buffer is
           already full, sending another keepalive is not useful. */
        if(rc && rc != LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send keepalive message");
            return rc;
        }

        session->keepalive_last_sent = now;
        if(seconds_to_next)
            *seconds_to_next = session->keepalive_interval;
    }
    else if(seconds_to_next) {
        *seconds_to_next = (int) (session->keepalive_last_sent - now)
            + session->keepalive_interval;
    }

    return 0;
}

```

# `libssh2/src/kex.c`

This is a header file that defines the header information for software released under the GPL license. It includes information about the author, copyright, and license terms. The header file can be included in the source code of the software to indicate that it is released under the GPL license.


```cpp
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2010-2019, Daniel Stenberg <daniel@haxx.se>
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

这段代码是一个用于传输协议的库头文件，其中包括了对于SHA1哈希算法实现的宏定义和一些辅助函数。

具体来说，这段代码包含以下内容：

1. 引入了"libssh2_priv.h"，这是SSH2协议的私钥头文件，用于实现SSH2协议的加密、解密等操作。

2. 引入了"transport.h"，"comp.h"，"mac.h"，这些头文件，它们的定义和作用在后面会被详细解释。

3. 定义了一个名为"SHA1_DIGEST_LENGTH"的宏，其值为"SHA_DIGEST_LENGTH"。这个宏后面会被使用，用于定义SHA1哈希算法的输入长度。

4. 定义了一个名为"辅助函数名为kex_method_diffie_hellman_group1_sha1_key_exchange"，是一个未命名的函数，但使用了这个名称来定义一个辅助函数。

由于辅助函数未被实现，因此无法提供具体的函数接口，但可以推测出该函数与哈希算法相关，可能用于实现与客户端的交互等功能。


```cpp
#include "libssh2_priv.h"

#include "transport.h"
#include "comp.h"
#include "mac.h"

#include <assert.h>

/* define SHA1_DIGEST_LENGTH for the macro below */
#ifndef SHA1_DIGEST_LENGTH
#define SHA1_DIGEST_LENGTH SHA_DIGEST_LENGTH
#endif

/* TODO: Switch this to an inline and handle alloc() failures */
/* Helper macro called from
   kex_method_diffie_hellman_group1_sha1_key_exchange */

```

The `SHA_VALUE_HASH` function is used to create a hash of a specified `value` using the specified `digest_type` and `version` parameters. The hash is a combination of the `value` as well as some additional information, such as the `reqlen` and the `version` parameters.

Here is the function signature for the `SHA_VALUE_HASH` function:
```cpp
int libssh2_sha_value_hash(digest_type, value,               reqlen, version);
```
This function takes four parameters:

* `digest_type`: The type of hash algorithm to use. The possible values are `SHA256`, `SHA512`, and `SHA224`.
* `value`: The input value to hash. This value should be a non-empty `unsigned char` pointer.
* `reqlen`: The requested length of the output hash. This parameter should be a `unsigned long` and should be at least `SHA256_DIGEST_LENGTH` to ensure that the hash is long enough to hash `SHA256` digests.
* `version`: An optional parameter to specify the version of the `SHA256` hash algorithm to use. The default value is `0`.

The function returns an `int` representing the result of the hash operation. If the hash cannot be created due to any errors, the function will return `-1`.


```cpp
#define LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(value, reqlen, version)        \
    {                                                                       \
        if(type == LIBSSH2_EC_CURVE_NISTP256) {                             \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, value, reqlen, version); \
        }                                                                   \
        else if(type == LIBSSH2_EC_CURVE_NISTP384) {                        \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(384, value, reqlen, version); \
        }                                                                   \
        else if(type == LIBSSH2_EC_CURVE_NISTP521) {                        \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(512, value, reqlen, version); \
        }                                                                   \
    }                                                                       \


#define LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(digest_type, value,               \
                                          reqlen, version)                  \
{                                                                           \
    libssh2_sha##digest_type##_ctx hash;                                    \
    unsigned long len = 0;                                                  \
    if(!(value)) {                                                          \
        value = LIBSSH2_ALLOC(session,                                      \
                              reqlen + SHA##digest_type##_DIGEST_LENGTH);   \
    }                                                                       \
    if(value)                                                               \
        while(len < (unsigned long)reqlen) {                                \
            libssh2_sha##digest_type##_init(&hash);                         \
            libssh2_sha##digest_type##_update(hash,                         \
                                              exchange_state->k_value,      \
                                              exchange_state->k_value_len); \
            libssh2_sha##digest_type##_update(hash,                         \
                                              exchange_state->h_sig_comp,   \
                                         SHA##digest_type##_DIGEST_LENGTH); \
            if(len > 0) {                                                   \
                libssh2_sha##digest_type##_update(hash, value, len);        \
            }                                                               \
            else {                                                          \
                libssh2_sha##digest_type##_update(hash, (version), 1);      \
                libssh2_sha##digest_type##_update(hash, session->session_id,\
                                                  session->session_id_len); \
            }                                                               \
            libssh2_sha##digest_type##_final(hash, (value) + len);          \
            len += SHA##digest_type##_DIGEST_LENGTH;                        \
        }                                                                   \
}

```

这段代码是一个C语言的函数，名为`_libssh2_sha_algo_ctx_init`。它的作用是初始化SSH安全哈希算法为SHA-256、SHA-384或SHA-512。如果选择的算法不被支持，函数将返回一个错误信息。

函数有两个参数，一个是SHA算法，另一个是使用的安全哈希算法的上下文。函数首先检查所选的算法是否支持，然后使用相应的函数进行初始化。如果算法不支持，函数将抛出错误，并返回一个值。如果算法支持，函数将返回前一个函数的返回值。


```cpp
/*!
 * @note The following are wrapper functions used by diffie_hellman_sha_algo().
 * TODO: Switch backend SHA macros to functions to allow function pointers
 * @discussion Ideally these would be function pointers but the backend macros
 * don't allow it so we have to wrap them up in helper functions
 */

static void _libssh2_sha_algo_ctx_init(int sha_algo, void *ctx)
{
    if(sha_algo == 512) {
        libssh2_sha512_init((libssh2_sha512_ctx*)ctx);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_init((libssh2_sha384_ctx*)ctx);
    }
    else if(sha_algo == 256) {
        libssh2_sha256_init((libssh2_sha256_ctx*)ctx);
    }
    else if(sha_algo == 1) {
        libssh2_sha1_init((libssh2_sha1_ctx*)ctx);
    }
    else {
        assert(0);
    }
}

```

这段代码是一个名为 `_libssh2_sha_algo_ctx_update` 的函数，属于名为 `libssh2` 的库。它的作用是处理 SSH 哈希算法的动态参数（主要是输入数据）的更新。

具体来说，当 `sha_algo` 的值为 512 时，函数会执行大端哈希算法（Big-Endian）的动态参数更新。当 `sha_algo` 的值为 384 时，函数会执行小端哈希算法（Little-Endian）的动态参数更新。当 `sha_algo` 的值为 256 时，函数会执行散列算法（Hash Algorithm）的动态参数更新。当 `sha_algo` 的值为 1 时，函数会执行简单的哈希算法更新。当 `sha_algo` 的值为其他值时，函数会执行默认的哈希算法更新（可能包括一些系统哈希算法，如 MD5、SHA-1 等）。

函数的参数包括：

- `sha_algo`：需要更新的哈希算法类型。
- `ctx`：哈希算法的上下文结构，通常包括一个指向 `libssh2_sha512_ctx`、`libssh2_sha384_ctx` 或 `libssh2_sha256_ctx` 的指针，以及一个指向 `void` 类型的数据缓冲区的指针。
- `data`：需要更新的输入数据。
- `len`：输入数据缓冲区的大小，单位为字节。

函数的实现主要负责在给定的哈希算法类型下更新输入数据，以实现 SSH 哈希算法的动态参数更新。


```cpp
static void _libssh2_sha_algo_ctx_update(int sha_algo, void *ctx,
                                         void *data, size_t len)
{
    if(sha_algo == 512) {
        libssh2_sha512_ctx *_ctx = (libssh2_sha512_ctx*)ctx;
        libssh2_sha512_update(*_ctx, data, len);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_ctx *_ctx = (libssh2_sha384_ctx*)ctx;
        libssh2_sha384_update(*_ctx, data, len);
    }
    else if(sha_algo == 256) {
        libssh2_sha256_ctx *_ctx = (libssh2_sha256_ctx*)ctx;
        libssh2_sha256_update(*_ctx, data, len);
    }
    else if(sha_algo == 1) {
        libssh2_sha1_ctx *_ctx = (libssh2_sha1_ctx*)ctx;
        libssh2_sha1_update(*_ctx, data, len);
    }
    else {
```

这段代码是一个C语言程序，它定义了一个名为_libssh2_sha_algo_ctx_final的函数。

这个函数的作用是处理SSH算法哈希函数的最终计算。这个函数根据输入的SSH算法哈希函数来执行不同的操作，最终输出一个或多个字节。

具体来说，如果输入的哈希函数是512、384、256或1，函数会执行相应的操作并将结果存储到ctx所指向的内存位置。如果输入的哈希函数是2，函数会执行一个特殊的操作并将结果存储到hash所指向的内存位置。

函数的实现比较复杂，因为它需要根据不同的哈希函数执行不同的操作。同时，函数的实现没有进行检查，因此可能会导致一些潜在的错误。


```cpp
#if LIBSSH2DEBUG
        assert(0);
#endif
    }
}

static void _libssh2_sha_algo_ctx_final(int sha_algo, void *ctx,
                                        void *hash)
{
    if(sha_algo == 512) {
        libssh2_sha512_ctx *_ctx = (libssh2_sha512_ctx*)ctx;
        libssh2_sha512_final(*_ctx, hash);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_ctx *_ctx = (libssh2_sha384_ctx*)ctx;
        libssh2_sha384_final(*_ctx, hash);
    }
    else if(sha_algo == 256) {
        libssh2_sha256_ctx *_ctx = (libssh2_sha256_ctx*)ctx;
        libssh2_sha256_final(*_ctx, hash);
    }
    else if(sha_algo == 1) {
        libssh2_sha1_ctx *_ctx = (libssh2_sha1_ctx*)ctx;
        libssh2_sha1_final(*_ctx, hash);
    }
    else {
```

这段代码是一个 if 语句，它会根据传入的 SHA 算法版本来执行不同的哈希函数。这里定义了一个名为 `_libssh2_sha_algo_value_hash` 的函数，它的输入参数包括 `sha_algo`、`session`、`exchange_state` 和 `data`，以及数据长度 `data_len` 和版本 `version`。

在 if 语句块中，首先检查传入的算法版本是否为 512，如果是，则执行 `LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(512, *data, data_len, version)`，接着检查是否为 384，如果是，则执行 `LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(384, *data, data_len, version)`，以此类推，如果为 1，则执行 `LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(1, *data, data_len, version)`，否则执行默认的哈希函数。

这里的 `LIBSSH2_KEX_METHOD_SHA_VALUE_HASH` 函数是一个表示 SIDH 哈希函数的函数，根据传入的算法版本执行相应的哈希函数，返回哈希值。


```cpp
#if LIBSSH2DEBUG
        assert(0);
#endif
    }
}

static void _libssh2_sha_algo_value_hash(int sha_algo,
                                      LIBSSH2_SESSION *session,
                                      kmdhgGPshakex_state_t *exchange_state,
                                      unsigned char **data, size_t data_len,
                                      const unsigned char *version)
{
    if(sha_algo == 512) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(512, *data, data_len, version);
    }
    else if(sha_algo == 384) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(384, *data, data_len, version);
    }
    else if(sha_algo == 256) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, *data, data_len, version);
    }
    else if(sha_algo == 1) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(1, *data, data_len, version);
    }
    else {
```

This code block appears to be a part of a solution that handles the exchange of host keys between two Bitcoin wallets. The exchange is typically done using the libssh2 library, and it involves sending a DHE-based host key to the recipient's server, and waiting for a KEX reply to confirm that the host key has been successfully added to the recipient's wallet.

The relevant parts of the code block are as follows:

1. The initialization of the host key:
```cpp
exchange_state->hv = hv_init(session->channel);
```
This initializes the host key to be used for the exchange.

2. The loop that waits for a KEX reply and adds the host key to the recipient's wallet:
```cpp
while(exchange_state->state == libssh2_NB_state_sent1) {
   rc = _libssh2_packet_require(session, packet_type_reply,
                                    &exchange_state->s_packet,
                                    &exchange_state->s_packet_len, 0, NULL,
                                    0, &exchange_state->req_state);
   if(rc == LIBSSH2_ERROR_EAGAIN) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                "Timed out waiting for KEX reply");
       goto clean_exit;
   }

   if(exchange_state->s_packet_len < 5) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                "Unexpected packet length");
       goto clean_exit;
   }

   buf.data = exchange_state->s_packet;
   buf.len = exchange_state->s_packet_len;
   buf.dataptr = buf.data;
   buf.dataptr++; /* advance past type */

   if(session->server_hostkey)
       LIBSSH2_FREE(session, session->server_hostkey);

   if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                               &host_key_len)) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                "Could not copy host key");
       goto clean_exit;
   }

   session->server_hostkey_len = (uint32_t)host_key_len;

   ret = _libssh2_update_recipient_pubkey(session, exchange_state,
                                        &exchange_state->s_last_output);
   if(ret != 0) {
       goto clean_exit;
   }

   exchange_state->state = libssh2_nb_state_sent1;
}
```
3. The function that initializes the libssh2 packet receiver:
```cpp
hv_init(session->channel)
```
This initializes the libssh2 packet receiver and returns the handle (hv) to use it for sending and receiving packets.

4. The function that waits for a KEX reply and adds the host key to the recipient's wallet:
```cpp
while(exchange_state->state == libssh2_nb_state_sent1) {
   rc = _libssh2_packet_require(session, packet_type_reply,
                                    &exchange_state->s_packet,
                                    &exchange_state->s_packet_len, 0, NULL,
                                    0, &exchange_state->req_state);
   if(rc == LIBSSH2_ERROR_EAGAIN) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                "Timed out waiting for KEX reply");
       goto clean_exit;
   }

   if(exchange_state->s_packet_len < 5) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                "Unexpected packet length");
       goto clean_exit;
   }

   buf.data = exchange_state->s_packet;
   buf.len = exchange_state->s_packet_len;
   buf.dataptr = buf.data;
   buf.dataptr++; /* advance past type */

   if(session->server_hostkey)
       LIBSSH2_FREE(session, session->server_hostkey);

   if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                               &host_key_len)) {
       ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                "Could not copy host key");
       goto clean_exit;
   }

   session->server_hostkey_len = (uint32_t)host_key_len;

   ret = _libssh2_update_recipient_pubkey(session, exchange_state,
                                        &exchange_state->s_last_output);
   if(ret != 0) {
       goto clean_exit;
   }

   exchange_state->state = libssh2_nb_state_sent1;
}
```
5. The function that copies the host key to the recipient's wallet:
```cpp
if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                               &host_key_len)) {
   ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                "Could not copy host key");
   goto clean_exit;
}
```
This function copies the host key to the recipient's wallet by calling the `libssh2_copy_string` function, which copies a data string to another data string. The function takes two arguments: the session object and the data to be copied, and returns a retcode indicating whether the operation was successful.

6. The function that updates the recipient's wallet with the copied host key:
```cpp
if(exchange_state->s_last_output < 0 ||
   exchange_state->s


```
#if LIBSSH2DEBUG
        assert(0);
#endif
    }
}


/*!
 * @function diffie_hellman_sha_algo
 * @abstract Diffie Hellman Key Exchange, Group Agnostic,
 * SHA Algorithm Agnostic
 * @result 0 on success, error code on failure
 */
static int diffie_hellman_sha_algo(LIBSSH2_SESSION *session,
                                   _libssh2_bn *g,
                                   _libssh2_bn *p,
                                   int group_order,
                                   int sha_algo_value,
                                   void *exchange_hash_ctx,
                                   unsigned char packet_type_init,
                                   unsigned char packet_type_reply,
                                   unsigned char *midhash,
                                   unsigned long midhash_len,
                                   kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;

    int digest_len = 0;

    if(sha_algo_value == 512)
        digest_len = SHA512_DIGEST_LENGTH;
    else if(sha_algo_value == 384)
        digest_len = SHA384_DIGEST_LENGTH;
    else if(sha_algo_value == 256)
        digest_len = SHA256_DIGEST_LENGTH;
    else if(sha_algo_value == 1)
        digest_len = SHA1_DIGEST_LENGTH;
    else {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                            "sha algo value is unimplemented");
        goto clean_exit;
    }

    if(exchange_state->state == libssh2_NB_state_idle) {
        /* Setup initial values */
        exchange_state->e_packet = NULL;
        exchange_state->s_packet = NULL;
        exchange_state->k_value = NULL;
        exchange_state->ctx = _libssh2_bn_ctx_new();
        libssh2_dh_init(&exchange_state->x);
        exchange_state->e = _libssh2_bn_init(); /* g^x mod p */
        exchange_state->f = _libssh2_bn_init_from_bin(); /* g^(Random from
                                                            server) mod p */
        exchange_state->k = _libssh2_bn_init(); /* The shared secret: f^x mod
                                                   p */

        /* Zero the whole thing out */
        memset(&exchange_state->req_state, 0, sizeof(packet_require_state_t));

        /* Generate x and e */
        if(_libssh2_bn_bits(p) > LIBSSH2_DH_MAX_MODULUS_BITS) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                                 "dh modulus value is too large");
            goto clean_exit;
        }

        rc = libssh2_dh_key_pair(&exchange_state->x, exchange_state->e, g, p,
                                 group_order, exchange_state->ctx);
        if(rc)
            goto clean_exit;

        /* Send KEX init */
        /* packet_type(1) + String Length(4) + leading 0(1) */
        exchange_state->e_packet_len =
            _libssh2_bn_bytes(exchange_state->e) + 6;
        if(_libssh2_bn_bits(exchange_state->e) % 8) {
            /* Leading 00 not needed */
            exchange_state->e_packet_len--;
        }

        exchange_state->e_packet =
            LIBSSH2_ALLOC(session, exchange_state->e_packet_len);
        if(!exchange_state->e_packet) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Out of memory error");
            goto clean_exit;
        }
        exchange_state->e_packet[0] = packet_type_init;
        _libssh2_htonu32(exchange_state->e_packet + 1,
                         exchange_state->e_packet_len - 5);
        if(_libssh2_bn_bits(exchange_state->e) % 8) {
            _libssh2_bn_to_bin(exchange_state->e,
                               exchange_state->e_packet + 5);
        }
        else {
            exchange_state->e_packet[5] = 0;
            _libssh2_bn_to_bin(exchange_state->e,
                               exchange_state->e_packet + 6);
        }

        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sending KEX packet %d",
                       (int) packet_type_init);
        exchange_state->state = libssh2_NB_state_created;
    }

    if(exchange_state->state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, exchange_state->e_packet,
                                     exchange_state->e_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send KEX init message");
            goto clean_exit;
        }
        exchange_state->state = libssh2_NB_state_sent;
    }

    if(exchange_state->state == libssh2_NB_state_sent) {
        if(session->burn_optimistic_kexinit) {
            /* The first KEX packet to come along will be the guess initially
             * sent by the server.  That guess turned out to be wrong so we
             * need to silently ignore it */
            int burn_type;

            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Waiting for badly guessed KEX packet "
                           "(to be ignored)");
            burn_type =
                _libssh2_packet_burn(session, &exchange_state->burn_state);
            if(burn_type == LIBSSH2_ERROR_EAGAIN) {
                return burn_type;
            }
            else if(burn_type <= 0) {
                /* Failed to receive a packet */
                ret = burn_type;
                goto clean_exit;
            }
            session->burn_optimistic_kexinit = 0;

            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Burnt packet of type: %02x",
                           (unsigned int) burn_type);
        }

        exchange_state->state = libssh2_NB_state_sent1;
    }

    if(exchange_state->state == libssh2_NB_state_sent1) {
        /* Wait for KEX reply */
        struct string_buf buf;
        size_t host_key_len;

        rc = _libssh2_packet_require(session, packet_type_reply,
                                     &exchange_state->s_packet,
                                     &exchange_state->s_packet_len, 0, NULL,
                                     0, &exchange_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        if(rc) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                 "Timed out waiting for KEX reply");
            goto clean_exit;
        }

        /* Parse KEXDH_REPLY */
        if(exchange_state->s_packet_len < 5) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected packet length");
            goto clean_exit;
        }

        buf.data = exchange_state->s_packet;
        buf.len = exchange_state->s_packet_len;
        buf.dataptr = buf.data;
        buf.dataptr++; /* advance past type */

        if(session->server_hostkey)
            LIBSSH2_FREE(session, session->server_hostkey);

        if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                                &host_key_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Could not copy host key");
            goto clean_exit;
        }

        session->server_hostkey_len = (uint32_t)host_key_len;

```cpp

这段代码使用了MD5哈希算法对服务器主机密钥进行哈希，并在哈希过程中使用了一个MD5ctx类型的变量fingerprint_ctx。它主要的作用是判断输入的MD5哈希是否正确，如果正确则输出结果，否则输出无效。具体来说，它实现了以下步骤：

1. 初始化MD5哈希上下文fingerprint_ctx；
2. 如果成功，则对输入的服务器主机密钥进行MD5哈希，并获取MD5哈希结果；
3. 如果没有问题，则输出服务器主机密钥的MD5哈希结果是否有效；
4. 如果MD5哈希结果有误，则输出服务器主机密钥的MD5哈希结果无效。


```
#if LIBSSH2_MD5
        {
            libssh2_md5_ctx fingerprint_ctx;

            if(libssh2_md5_init(&fingerprint_ctx)) {
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                session->server_hostkey_md5_valid = FALSE;
            }
        }
```cpp

这段代码的作用是：

1. 如果定义了`LIBSSH2DEBUG`，则执行以下操作：
a. 生成一个长度为50的指针`fingerprint`，并将其指向`fprint`；
b. 使用一个循环，从0到15遍历`session->server_hostkey_md5`的16个字节位；
c. 将每个字节的低8位二进制数作为`fprint`的相应位；
d. `fprintf`的最后一个参数是一个字符串，用于打印服务器MD5指纹；
e. 调用`_libssh2_debug`函数，输出服务器MD5指纹；
2. 如果未定义`LIBSSH2DEBUG`，则执行以下操作：
a. 生成一个长度为50的指针`fingerprint_ctx`；
b. 如果定义了`LIBSSH2_MD5`，则执行以下操作：
i. 初始化`fingerprint_ctx`为`libssh2_sha1_init`的输出；
ii. 使用`libssh2_sha1_update`和`libssh2_sha1_final`函数生成服务器MD5指纹；
iii. 判断`fingerprint_ctx`是否有效，如果有效，则执行以下操作：
a. 更新`fingerprint_ctx`为服务器MD5指纹；
b. 判断`fingerprint_ctx`是否有效，如果有效，则`session->server_hostkey_sha1_valid`为真，否则为假。
3. 如果未定义`LIBSSH2_MD5`，则执行以下操作：
a. 生成一个长度为50的指针`fingerprint_ctx`；
b. 如果未定义`LIBSSH2DEBUG`，则执行以下操作：
i. 初始化`fingerprint_ctx`为`libssh2_sha1_init`的输出；
ii. 如果定义了`LIBSSH2DEBUG`，则执行以下操作：
a. 生成一个长度为50的指针`fingerprint`，并将其指向`fprint`；
b. 使用一个循环，从0到15遍历`session->server_hostkey_md5`的16个字节位；
c. 将每个字节的低8位二进制数作为`fprint`的相应位；
d. `fprintf`的最后一个参数是一个字符串，用于打印服务器MD5指纹；
e. 调用`_libssh2_debug`函数，输出服务器MD5指纹。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[50], *fprint = fingerprint;
            int i;
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            libssh2_sha1_ctx fingerprint_ctx;

            if(libssh2_sha1_init(&fingerprint_ctx)) {
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
```cpp

这段代码是用于在服务器的主机密钥中查找FIDI映射的。它分为两个部分，第一部分是在客户端本地生成一个FIDI映射，第二部分是在服务器上验证服务器的主机密钥是否匹配客户端计算出的FIDI映射。

第一部分代码中，首先定义了一个名为fingerprint的64字节字符数组，以及一个指向fingerprint的指针fprint。然后使用for循环，在每次循环中从服务器的主机密钥中读取32字节，并将其存储在fingerprint中。接着将前20个字节复制到fprint的后续位置，并最终在fprint的末尾加上一个空字符'\0'，以字符串结束。最后调用一个名为_libssh2_debug的函数，将客户端计算出的FIDI映射打印到控制台上，并使用%02x格式化字符串将fingerprint中的每个字节转换为两位十六进制数。

第二部分代码中，使用libssh2_sha256_init函数初始化一个名为fingerprint_ctx的SHA256ctx对象。然后使用libssh2_sha256_update函数将客户端计算出的FIDI映射与服务器的主机密钥匹配，并使用libssh2_sha256_final函数完成哈希算法。最后使用session->server_hostkey_sha256_valid判断服务器密钥是否有效，如果没有有效，则将session->server_hostkey_sha256_valid设置为FALSE。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[64], *fprint = fingerprint;
            int i;

            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */

        {
            libssh2_sha256_ctx fingerprint_ctx;

            if(libssh2_sha256_init(&fingerprint_ctx)) {
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
```cpp

I'm sorry, but I am unable to process this question as it is incomplete and does not provide enough information. It appears to be a question about how to handle a specific type of message in an SSH connection, but it is not clear what type of message or what is being sent.

If you have any additional information or context, please provide it so that I can better assist you.


```
#ifdef LIBSSH2DEBUG
        {
            char *base64Fingerprint = NULL;
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            if(base64Fingerprint != NULL) {
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
#endif /* LIBSSH2DEBUG */


        if(session->hostkey->init(session, session->server_hostkey,
                                   session->server_hostkey_len,
                                   &session->server_hostkey_abstract)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unable to initialize hostkey importer");
            goto clean_exit;
        }

        if(_libssh2_get_string(&buf, &(exchange_state->f_value),
                               &(exchange_state->f_value_len))) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unable to get f value");
            goto clean_exit;
        }

        _libssh2_bn_from_bin(exchange_state->f, exchange_state->f_value_len,
                             exchange_state->f_value);

        if(_libssh2_get_string(&buf, &(exchange_state->h_sig),
                               &(exchange_state->h_sig_len))) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unable to get h sig");
            goto clean_exit;
        }

        /* Compute the shared secret */
        libssh2_dh_secret(&exchange_state->x, exchange_state->k,
                          exchange_state->f, p, exchange_state->ctx);
        exchange_state->k_value_len = _libssh2_bn_bytes(exchange_state->k) + 5;
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            /* don't need leading 00 */
            exchange_state->k_value_len--;
        }
        exchange_state->k_value =
            LIBSSH2_ALLOC(session, exchange_state->k_value_len);
        if(!exchange_state->k_value) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate buffer for K");
            goto clean_exit;
        }
        _libssh2_htonu32(exchange_state->k_value,
                         exchange_state->k_value_len - 4);
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 4);
        }
        else {
            exchange_state->k_value[4] = 0;
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 5);
        }

        exchange_state->exchange_hash = (void *)&exchange_hash_ctx;
        _libssh2_sha_algo_ctx_init(sha_algo_value, exchange_hash_ctx);

        if(session->local.banner) {
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             strlen((char *) session->local.banner) - 2);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 4);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                session->local.banner,
                                strlen((char *) session->local.banner) - 2);
        }
        else {
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             sizeof(LIBSSH2_SSH_DEFAULT_BANNER) - 1);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 4);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                    (unsigned char *)
                                    LIBSSH2_SSH_DEFAULT_BANNER,
                                    sizeof(LIBSSH2_SSH_DEFAULT_BANNER) - 1);
        }

        _libssh2_htonu32(exchange_state->h_sig_comp,
                         strlen((char *) session->remote.banner));
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     session->remote.banner,
                                     strlen((char *) session->remote.banner));

        _libssh2_htonu32(exchange_state->h_sig_comp,
                         session->local.kexinit_len);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     session->local.kexinit,
                                     session->local.kexinit_len);

        _libssh2_htonu32(exchange_state->h_sig_comp,
                         session->remote.kexinit_len);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     session->remote.kexinit,
                                     session->remote.kexinit_len);

        _libssh2_htonu32(exchange_state->h_sig_comp,
                         session->server_hostkey_len);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     session->server_hostkey,
                                     session->server_hostkey_len);

        if(packet_type_init == SSH_MSG_KEX_DH_GEX_INIT) {
            /* diffie-hellman-group-exchange hashes additional fields */
```cpp

This function appears to be responsible for initializing the server to client compression. It does this by setting up a session, getting the remote server's public key, and setting up a internal key for the compressed data. It also performs some error handling and cleaning up any dynamically allocated memory.

It appears that the key initialization process, key generation and the like, is done using the libssh2 library. The key generation process uses the server's public key to generate a new RSA key, which is then encrypted using the server's private key and returned to the client. The compressed data is generated using the initialized server's public key and is sent to the client.


```
#ifdef LIBSSH2_DH_GEX_NEW
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             LIBSSH2_DH_GEX_MINGROUP);
            _libssh2_htonu32(exchange_state->h_sig_comp + 4,
                             LIBSSH2_DH_GEX_OPTGROUP);
            _libssh2_htonu32(exchange_state->h_sig_comp + 8,
                             LIBSSH2_DH_GEX_MAXGROUP);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 12);
#else
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             LIBSSH2_DH_GEX_OPTGROUP);
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 4);
#endif
        }

        if(midhash) {
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         midhash, midhash_len);
        }

        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->e_packet + 1,
                                     exchange_state->e_packet_len - 1);

        _libssh2_htonu32(exchange_state->h_sig_comp,
                         exchange_state->f_value_len);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->f_value,
                                     exchange_state->f_value_len);

        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->k_value,
                                     exchange_state->k_value_len);

        _libssh2_sha_algo_ctx_final(sha_algo_value, exchange_hash_ctx,
                                    exchange_state->h_sig_comp);

        if(session->hostkey->
           sig_verify(session, exchange_state->h_sig,
                      exchange_state->h_sig_len, exchange_state->h_sig_comp,
                      digest_len, &session->server_hostkey_abstract)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_SIGN,
                                 "Unable to verify hostkey signature");
            goto clean_exit;
        }

        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sending NEWKEYS message");
        exchange_state->c = SSH_MSG_NEWKEYS;

        exchange_state->state = libssh2_NB_state_sent2;
    }

    if(exchange_state->state == libssh2_NB_state_sent2) {
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send NEWKEYS message");
            goto clean_exit;
        }

        exchange_state->state = libssh2_NB_state_sent3;
    }

    if(exchange_state->state == libssh2_NB_state_sent3) {
        rc = _libssh2_packet_require(session, SSH_MSG_NEWKEYS,
                                     &exchange_state->tmp,
                                     &exchange_state->tmp_len, 0, NULL, 0,
                                     &exchange_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc, "Timed out waiting for NEWKEYS");
            goto clean_exit;
        }
        /* The first key exchange has been performed,
           switch to active crypt/comp/mac mode */
        session->state |= LIBSSH2_STATE_NEWKEYS;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Received NEWKEYS message");

        /* This will actually end up being just packet_type(1)
           for this packet type anyway */
        LIBSSH2_FREE(session, exchange_state->tmp);

        if(!session->session_id) {
            session->session_id = LIBSSH2_ALLOC(session, digest_len);
            if(!session->session_id) {
                ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                     "Unable to allocate buffer for "
                                     "SHA digest");
                goto clean_exit;
            }
            memcpy(session->session_id, exchange_state->h_sig_comp,
                   digest_len);
            session->session_id_len = digest_len;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "session_id calculated");
        }

        /* Cleanup any existing cipher */
        if(session->local.crypt->dtor) {
            session->local.crypt->dtor(session,
                                       &session->local.crypt_abstract);
        }

        /* Calculate IV/Secret/Key for each direction */
        if(session->local.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &iv,
                                         session->local.crypt->iv_len,
                                         (const unsigned char *)"A");

            if(!iv) {
                ret = -1;
                goto clean_exit;
            }
            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &secret,
                                         session->local.crypt->secret_len,
                                         (const unsigned char *)"C");

            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->local.crypt->
                init(session, session->local.crypt, iv, &free_iv, secret,
                     &free_secret, 1, &session->local.crypt_abstract)) {
                LIBSSH2_FREE(session, iv);
                LIBSSH2_FREE(session, secret);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->local.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->local.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server IV and Key calculated");

        if(session->remote.crypt->dtor) {
            /* Cleanup any existing cipher */
            session->remote.crypt->dtor(session,
                                        &session->remote.crypt_abstract);
        }

        if(session->remote.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &iv,
                                         session->remote.crypt->iv_len,
                                         (const unsigned char *)"B");
            if(!iv) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &secret,
                                         session->remote.crypt->secret_len,
                                         (const unsigned char *)"D");
            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->remote.crypt->
                init(session, session->remote.crypt, iv, &free_iv, secret,
                     &free_secret, 0, &session->remote.crypt_abstract)) {
                LIBSSH2_FREE(session, iv);
                LIBSSH2_FREE(session, secret);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->remote.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->remote.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client IV and Key calculated");

        if(session->local.mac->dtor) {
            session->local.mac->dtor(session, &session->local.mac_abstract);
        }

        if(session->local.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &key,
                                         session->local.mac->key_len,
                                         (const unsigned char *)"E");
            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->local.mac->init(session, key, &free_key,
                                     &session->local.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->local.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server HMAC Key calculated");

        if(session->remote.mac->dtor) {
            session->remote.mac->dtor(session, &session->remote.mac_abstract);
        }

        if(session->remote.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            _libssh2_sha_algo_value_hash(sha_algo_value, session,
                                         exchange_state, &key,
                                         session->remote.mac->key_len,
                                         (const unsigned char *)"F");
            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->remote.mac->init(session, key, &free_key,
                                      &session->remote.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->remote.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client HMAC Key calculated");

        /* Initialize compression for each direction */

        /* Cleanup any existing compression */
        if(session->local.comp && session->local.comp->dtor) {
            session->local.comp->dtor(session, 1,
                                      &session->local.comp_abstract);
        }

        if(session->local.comp && session->local.comp->init) {
            if(session->local.comp->init(session, 1,
                                          &session->local.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server compression initialized");

        if(session->remote.comp && session->remote.comp->dtor) {
            session->remote.comp->dtor(session, 0,
                                       &session->remote.comp_abstract);
        }

        if(session->remote.comp && session->remote.comp->init) {
            if(session->remote.comp->init(session, 0,
                                           &session->remote.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client compression initialized");

    }

  clean_exit:
    libssh2_dh_dtor(&exchange_state->x);
    _libssh2_bn_free(exchange_state->e);
    exchange_state->e = NULL;
    _libssh2_bn_free(exchange_state->f);
    exchange_state->f = NULL;
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;
    _libssh2_bn_ctx_free(exchange_state->ctx);
    exchange_state->ctx = NULL;

    if(exchange_state->e_packet) {
        LIBSSH2_FREE(session, exchange_state->e_packet);
        exchange_state->e_packet = NULL;
    }

    if(exchange_state->s_packet) {
        LIBSSH2_FREE(session, exchange_state->s_packet);
        exchange_state->s_packet = NULL;
    }

    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    exchange_state->state = libssh2_NB_state_idle;

    return ret;
}



```cpp

This is a function definition for a key exchange algorithm that uses the Diffie-Hellman key exchange algorithm.

It is used in the SSH protocol to establish a secure key connection between two clients.

The function takes a `session` object, a key for the Diffie-Hellman key exchange, and a Diffie-Hellman root.

It initializes the state to `libssh2_NB_state_idle` and sets up the parameters for the key exchange algorithm.

It then calls the `diffie_hellman_sha_algo` function to perform the key exchange.

If the key exchange is successful, the function returns `0`.

If the key exchange fails, the function returns `LIBSSH2_ERROR_EAGAIN`.


```
/* kex_method_diffie_hellman_group1_sha1_key_exchange
 * Diffie-Hellman Group1 (Actually Group2) Key Exchange using SHA1
 */
static int
kex_method_diffie_hellman_group1_sha1_key_exchange(LIBSSH2_SESSION *session,
                                                   key_exchange_state_low_t
                                                   * key_state)
{
    static const unsigned char p_value[128] = {
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xC9, 0x0F, 0xDA, 0xA2, 0x21, 0x68, 0xC2, 0x34,
        0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
        0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74,
        0x02, 0x0B, 0xBE, 0xA6, 0x3B, 0x13, 0x9B, 0x22,
        0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
        0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B,
        0x30, 0x2B, 0x0A, 0x6D, 0xF2, 0x5F, 0x14, 0x37,
        0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
        0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6,
        0xF4, 0x4C, 0x42, 0xE9, 0xA6, 0x37, 0xED, 0x6B,
        0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
        0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5,
        0xAE, 0x9F, 0x24, 0x11, 0x7C, 0x4B, 0x1F, 0xE6,
        0x49, 0x28, 0x66, 0x51, 0xEC, 0xE6, 0x53, 0x81,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };

    int ret;
    libssh2_sha1_ctx exchange_hash_ctx;

    if(key_state->state == libssh2_NB_state_idle) {
        /* g == 2 */
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */

        /* Initialize P and G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 128, p_value);

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group1 Key Exchange");

        key_state->state = libssh2_NB_state_created;
    }

    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 128, 1,
                                  (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state);
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }

    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;
    key_state->state = libssh2_NB_state_idle;

    return ret;
}


```cpp

这段代码定义了一个名为“diffie_hellman_group14_key_exchange”的函数，它采用Diffie-Hellman Group14算法进行哈希函数回调。

具体来说，这个函数接受四个参数：

1. LIBSSH2_SESSION：指用于进行哈希的SSH2会话。
2. _libssh2_bn：定义了一个名为“_libssh2_bn”的GNU General Purpose散列函数类型的变量，用于表示哈希的输入和输出。
3. int：表示输入和输出的整数类型。
4. int：表示待哈希的哈希输出。
5. void *：表示待哈希的输入数据指针。
6. unsigned char：表示输入数据类型。
7. unsigned char：表示输出数据类型。
8. unsigned long：表示待哈希的哈希输出。
9. kmdhgGPshakex_state_t：表示哈希函数的state变量，用于表示哈希函数的执行状态。

函数实现中，首先通过ssh2_set_missing_arg_乾坤胖输入参数的剩余部分，然后通过_libssh2_bn的to_base64_func将输入的哈希数据进行base64编码，最后输入到diffie_hellman_hash_func_t类型的哈希函数中进行哈希计算。计算得到的哈希值会通过LIBSSH2_SESSION的get_transport_ chancery方法进行输出。


```
/* kex_method_diffie_hellman_group14_key_exchange
 * Diffie-Hellman Group14 Key Exchange with hash function callback
 */
typedef int (*diffie_hellman_hash_func_t)(LIBSSH2_SESSION *,
                                          _libssh2_bn *,
                                          _libssh2_bn *,
                                          int,
                                          int,
                                          void *,
                                          unsigned char,
                                          unsigned char,
                                          unsigned char *,
                                          unsigned long,
                                          kmdhgGPshakex_state_t *);
static int
```cpp

This is a function definition for a key management function in the SSH2 protocol. The function appears to perform a key exchange with a server using the Diffie-Hellman Group14 algorithm.

The function takes a number of parameters, including a pointer to the key management object (`key_state`), which is expected to store the current state of the key exchange. The function also takes a pointer to a hash function (`hash_func`), which is used to compute the散列值 for the key exchange.

The function first initializes the key management object with the initial values for the Diffie-Hellman Group14 algorithm. It then enters a loop that performs the key exchange with the server.

After the key exchange, the function returns the result code from the previous call to the `hashfunc` function. If the new hash value is the same as the previous one, the function returns LIBSSH2_ERROR_EAGAIN to indicate that the key exchange did not complete successfully. If the new hash value is different, the function returns the code from the previous call to `libssh2_nb_key_update_exchange_confirmation`.


```
kex_method_diffie_hellman_group14_key_exchange(LIBSSH2_SESSION *session,
                                               key_exchange_state_low_t
                                               * key_state,
                                               int sha_algo_value,
                                               void *exchange_hash_ctx,
                                               diffie_hellman_hash_func_t
                                               hashfunc)
{
    static const unsigned char p_value[256] = {
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xC9, 0x0F, 0xDA, 0xA2, 0x21, 0x68, 0xC2, 0x34,
        0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
        0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74,
        0x02, 0x0B, 0xBE, 0xA6, 0x3B, 0x13, 0x9B, 0x22,
        0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
        0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B,
        0x30, 0x2B, 0x0A, 0x6D, 0xF2, 0x5F, 0x14, 0x37,
        0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
        0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6,
        0xF4, 0x4C, 0x42, 0xE9, 0xA6, 0x37, 0xED, 0x6B,
        0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
        0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5,
        0xAE, 0x9F, 0x24, 0x11, 0x7C, 0x4B, 0x1F, 0xE6,
        0x49, 0x28, 0x66, 0x51, 0xEC, 0xE4, 0x5B, 0x3D,
        0xC2, 0x00, 0x7C, 0xB8, 0xA1, 0x63, 0xBF, 0x05,
        0x98, 0xDA, 0x48, 0x36, 0x1C, 0x55, 0xD3, 0x9A,
        0x69, 0x16, 0x3F, 0xA8, 0xFD, 0x24, 0xCF, 0x5F,
        0x83, 0x65, 0x5D, 0x23, 0xDC, 0xA3, 0xAD, 0x96,
        0x1C, 0x62, 0xF3, 0x56, 0x20, 0x85, 0x52, 0xBB,
        0x9E, 0xD5, 0x29, 0x07, 0x70, 0x96, 0x96, 0x6D,
        0x67, 0x0C, 0x35, 0x4E, 0x4A, 0xBC, 0x98, 0x04,
        0xF1, 0x74, 0x6C, 0x08, 0xCA, 0x18, 0x21, 0x7C,
        0x32, 0x90, 0x5E, 0x46, 0x2E, 0x36, 0xCE, 0x3B,
        0xE3, 0x9E, 0x77, 0x2C, 0x18, 0x0E, 0x86, 0x03,
        0x9B, 0x27, 0x83, 0xA2, 0xEC, 0x07, 0xA2, 0x8F,
        0xB5, 0xC5, 0x5D, 0xF0, 0x6F, 0x4C, 0x52, 0xC9,
        0xDE, 0x2B, 0xCB, 0xF6, 0x95, 0x58, 0x17, 0x18,
        0x39, 0x95, 0x49, 0x7C, 0xEA, 0x95, 0x6A, 0xE5,
        0x15, 0xD2, 0x26, 0x18, 0x98, 0xFA, 0x05, 0x10,
        0x15, 0x72, 0x8E, 0x5A, 0x8A, 0xAC, 0xAA, 0x68,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };
    int ret;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */

        /* g == 2 */
        /* Initialize P and G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 256, p_value);

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group14 Key Exchange");

        key_state->state = libssh2_NB_state_created;
    }
    ret = hashfunc(session, key_state->g, key_state->p,
                256, sha_algo_value, exchange_hash_ctx, SSH_MSG_KEXDH_INIT,
                SSH_MSG_KEXDH_REPLY, NULL, 0, &key_state->exchange_state);
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }

    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;

    return ret;
}



```cpp

这段代码定义了一个名为“kex_method_diffie_hellman_group14_sha1_key_exchange”的函数，它属于“libssh2_session”和“key_exchange_state_low_t”类型的变量。

该函数接受两个参数：一个“LIBSSH2_SESSION”类型的变量表示当前SSH会话，另一个“key_exchange_state_low_t”类型的变量表示要执行的密钥交换状态。函数内部使用“kex_method_diffie_hellman_group14_key_exchange”函数进行密钥交换，并传入一个名为“diffie_hellman_sha_algo”的参数，这是Diffie-Hellman Group14算法。

由于没有定义函数的定义，所以我不清楚该函数的实际作用。建议参考输入输出以了解更多关于该函数的信息。


```
/* kex_method_diffie_hellman_group14_sha1_key_exchange
 * Diffie-Hellman Group14 Key Exchange using SHA1
 */
static int
kex_method_diffie_hellman_group14_sha1_key_exchange(LIBSSH2_SESSION *session,
                                                    key_exchange_state_low_t
                                                    * key_state)
{
    libssh2_sha1_ctx ctx;
    return kex_method_diffie_hellman_group14_key_exchange(session,
                                                    key_state, 1,
                                                    &ctx,
                                                    diffie_hellman_sha_algo);
}



```cpp

这段代码定义了一个名为“kex_method_diffie_hellman_group14_sha256_key_exchange”的函数，它属于“libssh2”库。

这个函数接受两个参数：一个“LIBSSH2_SESSION”类型的session，另一个是“key_exchange_state_low_t”类型的指针key_state，这个参数表示要执行的密钥交换状态。

函数内部首先定义了一个名为“ctx”的“libssh2_sha256_ctx”类型的变量，然后调用“kex_method_diffie_hellman_group14_key_exchange”函数，传入参数key_state和256，并且使用“diffie_hellman_sha_algo”算法进行 diffie-hellman 密钥交换。

由于libssh2库中已经有了“kex_method_diffie_hellman_group14_sha256_key_exchange”函数，所以它可以确保正确地调用这个函数，并返回其结果。


```
/* kex_method_diffie_hellman_group14_sha256_key_exchange
 * Diffie-Hellman Group14 Key Exchange using SHA256
 */
static int
kex_method_diffie_hellman_group14_sha256_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)
{
    libssh2_sha256_ctx ctx;
    return kex_method_diffie_hellman_group14_key_exchange(session,
                                                    key_state, 256,
                                                    &ctx,
                                                    diffie_hellman_sha_algo);
}

```cpp

This is a function definition for a key management function, specifically the Diffie-Hellman Group16 (DH) key exchange function. It is used for key derivation in SSH2 protocol.

The function takes a session object, a key type, and an initialization point for the key. It initializes the key state's hash context and sets the key state to the initial state.

If the initialization point is a 0xFF byte string, it sets the key state to the idle state and initializes the hash context and the key state's hash value.

The function then calls the `diffie_hellman_sha_algo` function to perform the DH key exchange算法. If the key exchange is successful, the function returns 0. If an error occurs, it returns LIBSSH2_ERROR_EAGAIN.

After the key exchange, the function sets the key state to the idle state and frees the internal variables.


```
/* kex_method_diffie_hellman_group16_sha512_key_exchange
* Diffie-Hellman Group16 Key Exchange using SHA512
*/
static int
kex_method_diffie_hellman_group16_sha512_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)

{
    static const unsigned char p_value[512] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xC9, 0x0F, 0xDA, 0xA2,
    0x21, 0x68, 0xC2, 0x34, 0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
    0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74, 0x02, 0x0B, 0xBE, 0xA6,
    0x3B, 0x13, 0x9B, 0x22, 0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
    0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B, 0x30, 0x2B, 0x0A, 0x6D,
    0xF2, 0x5F, 0x14, 0x37, 0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
    0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6, 0xF4, 0x4C, 0x42, 0xE9,
    0xA6, 0x37, 0xED, 0x6B, 0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
    0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5, 0xAE, 0x9F, 0x24, 0x11,
    0x7C, 0x4B, 0x1F, 0xE6, 0x49, 0x28, 0x66, 0x51, 0xEC, 0xE4, 0x5B, 0x3D,
    0xC2, 0x00, 0x7C, 0xB8, 0xA1, 0x63, 0xBF, 0x05, 0x98, 0xDA, 0x48, 0x36,
    0x1C, 0x55, 0xD3, 0x9A, 0x69, 0x16, 0x3F, 0xA8, 0xFD, 0x24, 0xCF, 0x5F,
    0x83, 0x65, 0x5D, 0x23, 0xDC, 0xA3, 0xAD, 0x96, 0x1C, 0x62, 0xF3, 0x56,
    0x20, 0x85, 0x52, 0xBB, 0x9E, 0xD5, 0x29, 0x07, 0x70, 0x96, 0x96, 0x6D,
    0x67, 0x0C, 0x35, 0x4E, 0x4A, 0xBC, 0x98, 0x04, 0xF1, 0x74, 0x6C, 0x08,
    0xCA, 0x18, 0x21, 0x7C, 0x32, 0x90, 0x5E, 0x46, 0x2E, 0x36, 0xCE, 0x3B,
    0xE3, 0x9E, 0x77, 0x2C, 0x18, 0x0E, 0x86, 0x03, 0x9B, 0x27, 0x83, 0xA2,
    0xEC, 0x07, 0xA2, 0x8F, 0xB5, 0xC5, 0x5D, 0xF0, 0x6F, 0x4C, 0x52, 0xC9,
    0xDE, 0x2B, 0xCB, 0xF6, 0x95, 0x58, 0x17, 0x18, 0x39, 0x95, 0x49, 0x7C,
    0xEA, 0x95, 0x6A, 0xE5, 0x15, 0xD2, 0x26, 0x18, 0x98, 0xFA, 0x05, 0x10,
    0x15, 0x72, 0x8E, 0x5A, 0x8A, 0xAA, 0xC4, 0x2D, 0xAD, 0x33, 0x17, 0x0D,
    0x04, 0x50, 0x7A, 0x33, 0xA8, 0x55, 0x21, 0xAB, 0xDF, 0x1C, 0xBA, 0x64,
    0xEC, 0xFB, 0x85, 0x04, 0x58, 0xDB, 0xEF, 0x0A, 0x8A, 0xEA, 0x71, 0x57,
    0x5D, 0x06, 0x0C, 0x7D, 0xB3, 0x97, 0x0F, 0x85, 0xA6, 0xE1, 0xE4, 0xC7,
    0xAB, 0xF5, 0xAE, 0x8C, 0xDB, 0x09, 0x33, 0xD7, 0x1E, 0x8C, 0x94, 0xE0,
    0x4A, 0x25, 0x61, 0x9D, 0xCE, 0xE3, 0xD2, 0x26, 0x1A, 0xD2, 0xEE, 0x6B,
    0xF1, 0x2F, 0xFA, 0x06, 0xD9, 0x8A, 0x08, 0x64, 0xD8, 0x76, 0x02, 0x73,
    0x3E, 0xC8, 0x6A, 0x64, 0x52, 0x1F, 0x2B, 0x18, 0x17, 0x7B, 0x20, 0x0C,
    0xBB, 0xE1, 0x17, 0x57, 0x7A, 0x61, 0x5D, 0x6C, 0x77, 0x09, 0x88, 0xC0,
    0xBA, 0xD9, 0x46, 0xE2, 0x08, 0xE2, 0x4F, 0xA0, 0x74, 0xE5, 0xAB, 0x31,
    0x43, 0xDB, 0x5B, 0xFC, 0xE0, 0xFD, 0x10, 0x8E, 0x4B, 0x82, 0xD1, 0x20,
    0xA9, 0x21, 0x08, 0x01, 0x1A, 0x72, 0x3C, 0x12, 0xA7, 0x87, 0xE6, 0xD7,
    0x88, 0x71, 0x9A, 0x10, 0xBD, 0xBA, 0x5B, 0x26, 0x99, 0xC3, 0x27, 0x18,
    0x6A, 0xF4, 0xE2, 0x3C, 0x1A, 0x94, 0x68, 0x34, 0xB6, 0x15, 0x0B, 0xDA,
    0x25, 0x83, 0xE9, 0xCA, 0x2A, 0xD4, 0x4C, 0xE8, 0xDB, 0xBB, 0xC2, 0xDB,
    0x04, 0xDE, 0x8E, 0xF9, 0x2E, 0x8E, 0xFC, 0x14, 0x1F, 0xBE, 0xCA, 0xA6,
    0x28, 0x7C, 0x59, 0x47, 0x4E, 0x6B, 0xC0, 0x5D, 0x99, 0xB2, 0x96, 0x4F,
    0xA0, 0x90, 0xC3, 0xA2, 0x23, 0x3B, 0xA1, 0x86, 0x51, 0x5B, 0xE7, 0xED,
    0x1F, 0x61, 0x29, 0x70, 0xCE, 0xE2, 0xD7, 0xAF, 0xB8, 0x1B, 0xDD, 0x76,
    0x21, 0x70, 0x48, 0x1C, 0xD0, 0x06, 0x91, 0x27, 0xD5, 0xB0, 0x5A, 0xA9,
    0x93, 0xB4, 0xEA, 0x98, 0x8D, 0x8F, 0xDD, 0xC1, 0x86, 0xFF, 0xB7, 0xDC,
    0x90, 0xA6, 0xC0, 0x8F, 0x4D, 0xF4, 0x35, 0xC9, 0x34, 0x06, 0x31, 0x99,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };
    int ret;
    libssh2_sha512_ctx exchange_hash_ctx;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */

        /* g == 2 */
        /* Initialize P and G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 512, p_value);

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group16 Key Exchange");

        key_state->state = libssh2_NB_state_created;
    }

    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 512,
                                  512, (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state);
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }

    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;

    return ret;
}

```cpp

This is a function definition for the Diffie-Hellman Group18 (DH) key exchange algorithm in the SSH protocol. It is used to establish a secure key-based authentication connection between two parties using the Diffie-Hellman key exchange algorithm.

The function takes a libssh2\_session object as the session, a key\_state object containing the initial state of the key\_state, and a buffer for the exchanged hash context. It returns an error code indicating whether the key exchange was successful.

If the initial key\_state is not valid or the key\_state is not created, the function will return LIBSSH2\_ERROR\_INVALID\_KEY\_ Alias and continue to run the key\_state checks.

If the key\_state is valid and the key\_exchange was successful, the function returns LIBSSH2\_TRACE\_SUCCESS.


```
/* kex_method_diffie_hellman_group16_sha512_key_exchange
* Diffie-Hellman Group18 Key Exchange using SHA512
*/
static int
kex_method_diffie_hellman_group18_sha512_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)

{
    static const unsigned char p_value[1024] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xC9, 0x0F, 0xDA, 0xA2,
    0x21, 0x68, 0xC2, 0x34, 0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
    0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74, 0x02, 0x0B, 0xBE, 0xA6,
    0x3B, 0x13, 0x9B, 0x22, 0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
    0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B, 0x30, 0x2B, 0x0A, 0x6D,
    0xF2, 0x5F, 0x14, 0x37, 0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
    0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6, 0xF4, 0x4C, 0x42, 0xE9,
    0xA6, 0x37, 0xED, 0x6B, 0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
    0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5, 0xAE, 0x9F, 0x24, 0x11,
    0x7C, 0x4B, 0x1F, 0xE6, 0x49, 0x28, 0x66, 0x51, 0xEC, 0xE4, 0x5B, 0x3D,
    0xC2, 0x00, 0x7C, 0xB8, 0xA1, 0x63, 0xBF, 0x05, 0x98, 0xDA, 0x48, 0x36,
    0x1C, 0x55, 0xD3, 0x9A, 0x69, 0x16, 0x3F, 0xA8, 0xFD, 0x24, 0xCF, 0x5F,
    0x83, 0x65, 0x5D, 0x23, 0xDC, 0xA3, 0xAD, 0x96, 0x1C, 0x62, 0xF3, 0x56,
    0x20, 0x85, 0x52, 0xBB, 0x9E, 0xD5, 0x29, 0x07, 0x70, 0x96, 0x96, 0x6D,
    0x67, 0x0C, 0x35, 0x4E, 0x4A, 0xBC, 0x98, 0x04, 0xF1, 0x74, 0x6C, 0x08,
    0xCA, 0x18, 0x21, 0x7C, 0x32, 0x90, 0x5E, 0x46, 0x2E, 0x36, 0xCE, 0x3B,
    0xE3, 0x9E, 0x77, 0x2C, 0x18, 0x0E, 0x86, 0x03, 0x9B, 0x27, 0x83, 0xA2,
    0xEC, 0x07, 0xA2, 0x8F, 0xB5, 0xC5, 0x5D, 0xF0, 0x6F, 0x4C, 0x52, 0xC9,
    0xDE, 0x2B, 0xCB, 0xF6, 0x95, 0x58, 0x17, 0x18, 0x39, 0x95, 0x49, 0x7C,
    0xEA, 0x95, 0x6A, 0xE5, 0x15, 0xD2, 0x26, 0x18, 0x98, 0xFA, 0x05, 0x10,
    0x15, 0x72, 0x8E, 0x5A, 0x8A, 0xAA, 0xC4, 0x2D, 0xAD, 0x33, 0x17, 0x0D,
    0x04, 0x50, 0x7A, 0x33, 0xA8, 0x55, 0x21, 0xAB, 0xDF, 0x1C, 0xBA, 0x64,
    0xEC, 0xFB, 0x85, 0x04, 0x58, 0xDB, 0xEF, 0x0A, 0x8A, 0xEA, 0x71, 0x57,
    0x5D, 0x06, 0x0C, 0x7D, 0xB3, 0x97, 0x0F, 0x85, 0xA6, 0xE1, 0xE4, 0xC7,
    0xAB, 0xF5, 0xAE, 0x8C, 0xDB, 0x09, 0x33, 0xD7, 0x1E, 0x8C, 0x94, 0xE0,
    0x4A, 0x25, 0x61, 0x9D, 0xCE, 0xE3, 0xD2, 0x26, 0x1A, 0xD2, 0xEE, 0x6B,
    0xF1, 0x2F, 0xFA, 0x06, 0xD9, 0x8A, 0x08, 0x64, 0xD8, 0x76, 0x02, 0x73,
    0x3E, 0xC8, 0x6A, 0x64, 0x52, 0x1F, 0x2B, 0x18, 0x17, 0x7B, 0x20, 0x0C,
    0xBB, 0xE1, 0x17, 0x57, 0x7A, 0x61, 0x5D, 0x6C, 0x77, 0x09, 0x88, 0xC0,
    0xBA, 0xD9, 0x46, 0xE2, 0x08, 0xE2, 0x4F, 0xA0, 0x74, 0xE5, 0xAB, 0x31,
    0x43, 0xDB, 0x5B, 0xFC, 0xE0, 0xFD, 0x10, 0x8E, 0x4B, 0x82, 0xD1, 0x20,
    0xA9, 0x21, 0x08, 0x01, 0x1A, 0x72, 0x3C, 0x12, 0xA7, 0x87, 0xE6, 0xD7,
    0x88, 0x71, 0x9A, 0x10, 0xBD, 0xBA, 0x5B, 0x26, 0x99, 0xC3, 0x27, 0x18,
    0x6A, 0xF4, 0xE2, 0x3C, 0x1A, 0x94, 0x68, 0x34, 0xB6, 0x15, 0x0B, 0xDA,
    0x25, 0x83, 0xE9, 0xCA, 0x2A, 0xD4, 0x4C, 0xE8, 0xDB, 0xBB, 0xC2, 0xDB,
    0x04, 0xDE, 0x8E, 0xF9, 0x2E, 0x8E, 0xFC, 0x14, 0x1F, 0xBE, 0xCA, 0xA6,
    0x28, 0x7C, 0x59, 0x47, 0x4E, 0x6B, 0xC0, 0x5D, 0x99, 0xB2, 0x96, 0x4F,
    0xA0, 0x90, 0xC3, 0xA2, 0x23, 0x3B, 0xA1, 0x86, 0x51, 0x5B, 0xE7, 0xED,
    0x1F, 0x61, 0x29, 0x70, 0xCE, 0xE2, 0xD7, 0xAF, 0xB8, 0x1B, 0xDD, 0x76,
    0x21, 0x70, 0x48, 0x1C, 0xD0, 0x06, 0x91, 0x27, 0xD5, 0xB0, 0x5A, 0xA9,
    0x93, 0xB4, 0xEA, 0x98, 0x8D, 0x8F, 0xDD, 0xC1, 0x86, 0xFF, 0xB7, 0xDC,
    0x90, 0xA6, 0xC0, 0x8F, 0x4D, 0xF4, 0x35, 0xC9, 0x34, 0x02, 0x84, 0x92,
    0x36, 0xC3, 0xFA, 0xB4, 0xD2, 0x7C, 0x70, 0x26, 0xC1, 0xD4, 0xDC, 0xB2,
    0x60, 0x26, 0x46, 0xDE, 0xC9, 0x75, 0x1E, 0x76, 0x3D, 0xBA, 0x37, 0xBD,
    0xF8, 0xFF, 0x94, 0x06, 0xAD, 0x9E, 0x53, 0x0E, 0xE5, 0xDB, 0x38, 0x2F,
    0x41, 0x30, 0x01, 0xAE, 0xB0, 0x6A, 0x53, 0xED, 0x90, 0x27, 0xD8, 0x31,
    0x17, 0x97, 0x27, 0xB0, 0x86, 0x5A, 0x89, 0x18, 0xDA, 0x3E, 0xDB, 0xEB,
    0xCF, 0x9B, 0x14, 0xED, 0x44, 0xCE, 0x6C, 0xBA, 0xCE, 0xD4, 0xBB, 0x1B,
    0xDB, 0x7F, 0x14, 0x47, 0xE6, 0xCC, 0x25, 0x4B, 0x33, 0x20, 0x51, 0x51,
    0x2B, 0xD7, 0xAF, 0x42, 0x6F, 0xB8, 0xF4, 0x01, 0x37, 0x8C, 0xD2, 0xBF,
    0x59, 0x83, 0xCA, 0x01, 0xC6, 0x4B, 0x92, 0xEC, 0xF0, 0x32, 0xEA, 0x15,
    0xD1, 0x72, 0x1D, 0x03, 0xF4, 0x82, 0xD7, 0xCE, 0x6E, 0x74, 0xFE, 0xF6,
    0xD5, 0x5E, 0x70, 0x2F, 0x46, 0x98, 0x0C, 0x82, 0xB5, 0xA8, 0x40, 0x31,
    0x90, 0x0B, 0x1C, 0x9E, 0x59, 0xE7, 0xC9, 0x7F, 0xBE, 0xC7, 0xE8, 0xF3,
    0x23, 0xA9, 0x7A, 0x7E, 0x36, 0xCC, 0x88, 0xBE, 0x0F, 0x1D, 0x45, 0xB7,
    0xFF, 0x58, 0x5A, 0xC5, 0x4B, 0xD4, 0x07, 0xB2, 0x2B, 0x41, 0x54, 0xAA,
    0xCC, 0x8F, 0x6D, 0x7E, 0xBF, 0x48, 0xE1, 0xD8, 0x14, 0xCC, 0x5E, 0xD2,
    0x0F, 0x80, 0x37, 0xE0, 0xA7, 0x97, 0x15, 0xEE, 0xF2, 0x9B, 0xE3, 0x28,
    0x06, 0xA1, 0xD5, 0x8B, 0xB7, 0xC5, 0xDA, 0x76, 0xF5, 0x50, 0xAA, 0x3D,
    0x8A, 0x1F, 0xBF, 0xF0, 0xEB, 0x19, 0xCC, 0xB1, 0xA3, 0x13, 0xD5, 0x5C,
    0xDA, 0x56, 0xC9, 0xEC, 0x2E, 0xF2, 0x96, 0x32, 0x38, 0x7F, 0xE8, 0xD7,
    0x6E, 0x3C, 0x04, 0x68, 0x04, 0x3E, 0x8F, 0x66, 0x3F, 0x48, 0x60, 0xEE,
    0x12, 0xBF, 0x2D, 0x5B, 0x0B, 0x74, 0x74, 0xD6, 0xE6, 0x94, 0xF9, 0x1E,
    0x6D, 0xBE, 0x11, 0x59, 0x74, 0xA3, 0x92, 0x6F, 0x12, 0xFE, 0xE5, 0xE4,
    0x38, 0x77, 0x7C, 0xB6, 0xA9, 0x32, 0xDF, 0x8C, 0xD8, 0xBE, 0xC4, 0xD0,
    0x73, 0xB9, 0x31, 0xBA, 0x3B, 0xC8, 0x32, 0xB6, 0x8D, 0x9D, 0xD3, 0x00,
    0x74, 0x1F, 0xA7, 0xBF, 0x8A, 0xFC, 0x47, 0xED, 0x25, 0x76, 0xF6, 0x93,
    0x6B, 0xA4, 0x24, 0x66, 0x3A, 0xAB, 0x63, 0x9C, 0x5A, 0xE4, 0xF5, 0x68,
    0x34, 0x23, 0xB4, 0x74, 0x2B, 0xF1, 0xC9, 0x78, 0x23, 0x8F, 0x16, 0xCB,
    0xE3, 0x9D, 0x65, 0x2D, 0xE3, 0xFD, 0xB8, 0xBE, 0xFC, 0x84, 0x8A, 0xD9,
    0x22, 0x22, 0x2E, 0x04, 0xA4, 0x03, 0x7C, 0x07, 0x13, 0xEB, 0x57, 0xA8,
    0x1A, 0x23, 0xF0, 0xC7, 0x34, 0x73, 0xFC, 0x64, 0x6C, 0xEA, 0x30, 0x6B,
    0x4B, 0xCB, 0xC8, 0x86, 0x2F, 0x83, 0x85, 0xDD, 0xFA, 0x9D, 0x4B, 0x7F,
    0xA2, 0xC0, 0x87, 0xE8, 0x79, 0x68, 0x33, 0x03, 0xED, 0x5B, 0xDD, 0x3A,
    0x06, 0x2B, 0x3C, 0xF5, 0xB3, 0xA2, 0x78, 0xA6, 0x6D, 0x2A, 0x13, 0xF8,
    0x3F, 0x44, 0xF8, 0x2D, 0xDF, 0x31, 0x0E, 0xE0, 0x74, 0xAB, 0x6A, 0x36,
    0x45, 0x97, 0xE8, 0x99, 0xA0, 0x25, 0x5D, 0xC1, 0x64, 0xF3, 0x1C, 0xC5,
    0x08, 0x46, 0x85, 0x1D, 0xF9, 0xAB, 0x48, 0x19, 0x5D, 0xED, 0x7E, 0xA1,
    0xB1, 0xD5, 0x10, 0xBD, 0x7E, 0xE7, 0x4D, 0x73, 0xFA, 0xF3, 0x6B, 0xC3,
    0x1E, 0xCF, 0xA2, 0x68, 0x35, 0x90, 0x46, 0xF4, 0xEB, 0x87, 0x9F, 0x92,
    0x40, 0x09, 0x43, 0x8B, 0x48, 0x1C, 0x6C, 0xD7, 0x88, 0x9A, 0x00, 0x2E,
    0xD5, 0xEE, 0x38, 0x2B, 0xC9, 0x19, 0x0D, 0xA6, 0xFC, 0x02, 0x6E, 0x47,
    0x95, 0x58, 0xE4, 0x47, 0x56, 0x77, 0xE9, 0xAA, 0x9E, 0x30, 0x50, 0xE2,
    0x76, 0x56, 0x94, 0xDF, 0xC8, 0x1F, 0x56, 0xE8, 0x80, 0xB9, 0x6E, 0x71,
    0x60, 0xC9, 0x80, 0xDD, 0x98, 0xED, 0xD3, 0xDF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF
    };
    int ret;
    libssh2_sha512_ctx exchange_hash_ctx;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */

        /* g == 2 */
        /* Initialize P and G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 1024, p_value);

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group18 Key Exchange");

        key_state->state = libssh2_NB_state_created;
    }

    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 1024,
                                  512, (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state);
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }

    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;

    return ret;
}

```cpp

这段代码定义了一个名为“kex_method_diffie_hellman_group_exchange_sha1_key_exchange”的函数，它属于“libssh2_ssh”目录。这个函数接受两个参数：一个“LIBSSH2_SESSION”类型的句柄（表示与远程主机通信的会话）和一个“key_exchange_state_low_t”类型的键交换状态低结构体。

这个函数的作用是使用Diffie-Hellman Group Exchange Key Exchange算法，通过与远程主机进行协商，来生成一个随机密钥对。这种算法基于对称密钥和消息摘要，因此它要求通信双方都有相同的密钥。

具体来说，这个函数首先检查键交换状态低结构体中当前的状态是否为“libssh2_NB_state_idle”，如果是，那么就表示需要建立P和G密钥对。然后，它分别初始化这两个密钥对，并请求一个随机数，用于生成消息摘要。

接着，函数调用“_libssh2_bn_init_from_bin”函数来完成P和G的生成，并返回这些密钥对。之后，如果成功交换了密钥，那么函数将返回0，否则返回-1。


```
/* kex_method_diffie_hellman_group_exchange_sha1_key_exchange
 * Diffie-Hellman Group Exchange Key Exchange using SHA1
 * Negotiates random(ish) group for secret derivation
 */
static int
kex_method_diffie_hellman_group_exchange_sha1_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init_from_bin();
        key_state->g = _libssh2_bn_init_from_bin();
        /* Ask for a P and G pair */
```cpp

这段代码是一个用于在SSH客户端和服务器之间进行Diffie-Hellman Group-Exchange（DHGE）的代码。它通过#ifdef和#else来判断客户端和服务器是否支持使用这个新方法，如果不支持，就执行旧方法。

具体来说，这段代码的作用是：

1. 如果客户端支持使用DHGE新方法，那么它将发送一个SSH_MSG_KEX_DH_GEX_REQUEST消息给服务器，并在请求中指定libssh2_dh_gex_mingroup、libssh2_dh_gex_optgroup和libssh2_dh_gex_maxgroup成员。然后，它会将请求长度设置为13（5个关键对（盐） + 3个关键对（数据） + 5个关键对（消息））。最后，它会向服务器发送一个DHGE旧方法的请求，并将请求长度设置为5。服务器在接收到请求后会执行相应的旧方法。

2. 如果客户端不支持DHGE新方法，那么它将执行一个类似于旧方法的代码。在这种情况下，它会将请求类型设置为SSH_MSG_KEX_DH_GEX_REQUEST_OLD，然后将关键对（盐）和关键对（数据）设置为libssh2_dh_gex_optgroup和libssh2_dh_gex_maxgroup，将请求长度设置为5。服务器在接收到请求后也会执行相应的旧方法。


```
#ifdef LIBSSH2_DH_GEX_NEW
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_MINGROUP);
        _libssh2_htonu32(key_state->request + 5, LIBSSH2_DH_GEX_OPTGROUP);
        _libssh2_htonu32(key_state->request + 9, LIBSSH2_DH_GEX_MAXGROUP);
        key_state->request_len = 13;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(New Method)");
#else
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST_OLD;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_OPTGROUP);
        key_state->request_len = 5;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(Old Method)");
```cpp

这段代码的作用是实现基于Diffie-Hellman-SHA1算法的SSH密钥交换。代码中定义了一个名为key_state的结构体，其中包含了一些与SSH算法相关的变量和状态。函数原型定义了一个`diffie_hellman_sha_algo`函数，用于执行Diffie-Hellman-SHA1算法。在函数内部，首先通过`libssh2_get_bignum_bytes`函数获取两个大整数的关键字和要点，然后通过`_libssh2_error`函数检查获取到的关键字和要点是否正确，并尝试使用Diffie-Hellman-SHA1算法计算消息的散列值。如果函数内部出现错误，可通过`dh_gex_clean_exit`标签返回错误代码。在`key_state`结构体中，通过`_libssh2_bn_from_bin`函数将字节转换为本地字节表示，然后使用`diffie_hellman_sha_algo`函数执行Diffie-Hellman-SHA1算法计算消息的散列值，并将结果与本地字节进行比较，如果结果不同，则表示密钥交换成功。最后，通过`LIBSSH2_FREE`函数释放相关变量，`_libssh2_nb_free`函数释放本地字节表示，`key_state`结构体状态设置为`libssh2_NB_state_idle`，标志着SSH密钥交换成功。


```
#endif

        key_state->state = libssh2_NB_state_created;
    }

    if(key_state->state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send Group Exchange Request");
            goto dh_gex_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent;
    }

    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_require(session, SSH_MSG_KEX_DH_GEX_GROUP,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for GEX_GROUP reply");
            goto dh_gex_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent1;
    }

    if(key_state->state == libssh2_NB_state_sent1) {
        size_t p_len, g_len;
        unsigned char *p, *g;
        struct string_buf buf;
        libssh2_sha1_ctx exchange_hash_ctx;

        if(key_state->data_len < 9) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto dh_gex_clean_exit;
        }

        buf.data = key_state->data;
        buf.dataptr = buf.data;
        buf.len = key_state->data_len;

        buf.dataptr++; /* increment to big num */

        if(_libssh2_get_bignum_bytes(&buf, &p, &p_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            goto dh_gex_clean_exit;
        }

        if(_libssh2_get_bignum_bytes(&buf, &g, &g_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            goto dh_gex_clean_exit;
        }

        _libssh2_bn_from_bin(key_state->p, p_len, p);
        _libssh2_bn_from_bin(key_state->g, g_len, g);

        ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p,
                                      p_len, 1,
                                      (void *)&exchange_hash_ctx,
                                      SSH_MSG_KEX_DH_GEX_INIT,
                                      SSH_MSG_KEX_DH_GEX_REPLY,
                                      key_state->data + 1,
                                      key_state->data_len - 1,
                                      &key_state->exchange_state);
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        LIBSSH2_FREE(session, key_state->data);
    }

  dh_gex_clean_exit:
    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;

    return ret;
}



```cpp

这段代码定义了一个名为 `kex_method_diffie_hellman_group_exchange_sha256_key_exchange` 的函数，属于 `libssh2_python_libssh2_库`。

该函数接受两个参数：`session` 是一个 `LIBSSH2_SESSION` 类型的变量，表示与远程主机进行安全套接字的会话；`key_state` 是一个 `key_exchange_state_low_t` 类型的变量，表示用于保存哈希算法所需的参数。

函数内部首先检查 `key_state` 的状态是否为 `libssh2_NB_state_idle`，如果是，就表示初始化，设置 `key_state->p` 和 `key_state->g` 分别为 `_libssh2_bn_init()` 函数返回的哈希值。然后调用 `_libssh2_request_t DeserializeHostKey()` 函数，要求用户输入对密钥进行哈希的散列名称，也就是要求输入一个字符串。

接着，函数再次检查 `key_state` 的状态是否为 `libssh2_NB_state_idle`，如果不是，就表示已经初始化，可以开始使用哈希算法。此时，函数调用 `_libssh2_diffie_hellman_exchange()` 函数，根据输入的哈希名称和计算出的 `key_state->p` 和 `key_state->g` 值，来计算出要散列的值，然后使用 `_libssh2_diffie_hellman_exchange()` 函数的 `key_exchange()` 函数，开始计算散列值。

函数最后返回 `0`，如果计算成功，否则返回 `-1`，如果有错误，则返回 `-2`。


```
/* kex_method_diffie_hellman_group_exchange_sha256_key_exchange
 * Diffie-Hellman Group Exchange Key Exchange using SHA256
 * Negotiates random(ish) group for secret derivation
 */
static int
kex_method_diffie_hellman_group_exchange_sha256_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init();
        key_state->g = _libssh2_bn_init();
        /* Ask for a P and G pair */
```cpp

这段代码是用于在 SSH2 协议中执行 Diffie-Hellman Group-Exchange（DHGE）握手。它实现了在新方法和旧方法（旧方法为 SHA1）之间进行 DHGE 握手。当使用新方法（SHA256）时，代码会执行到 libssh2_dh_gex_new() 函数，而当使用旧方法（SHA1）时，则执行到 libssh2_dh_gex_old() 函数。


```
#ifdef LIBSSH2_DH_GEX_NEW
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_MINGROUP);
        _libssh2_htonu32(key_state->request + 5, LIBSSH2_DH_GEX_OPTGROUP);
        _libssh2_htonu32(key_state->request + 9, LIBSSH2_DH_GEX_MAXGROUP);
        key_state->request_len = 13;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(New Method SHA256)");
#else
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST_OLD;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_OPTGROUP);
        key_state->request_len = 5;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(Old Method SHA256)");
```cpp

This function appears to implement the DHIMAX key exchange algorithm using the SSH-DH key exchange protocol. It uses the Diffie-Hellman key exchange algorithm to generate a shared key between the client and server, and then compares the server's key to the client's key to ensure that they are the same.

The function takes two parameters: the client's public key and the server's public key. The function returns LIBSSH2_ERROR if the key is not of the expected type, or if the key exchange fails.

If the key exchange is successful, the function returns 0. If it fails, the function returns LIBSSH2_ERROR.


```
#endif

        key_state->state = libssh2_NB_state_created;
    }

    if(key_state->state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send "
                                 "Group Exchange Request SHA256");
            goto dh_gex_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent;
    }

    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_require(session, SSH_MSG_KEX_DH_GEX_GROUP,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for GEX_GROUP reply SHA256");
            goto dh_gex_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent1;
    }

    if(key_state->state == libssh2_NB_state_sent1) {
        unsigned char *p, *g;
        size_t p_len, g_len;
        struct string_buf buf;
        libssh2_sha256_ctx exchange_hash_ctx;

        if(key_state->data_len < 9) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto dh_gex_clean_exit;
        }

        buf.data = key_state->data;
        buf.dataptr = buf.data;
        buf.len = key_state->data_len;

        buf.dataptr++; /* increment to big num */

        if(_libssh2_get_bignum_bytes(&buf, &p, &p_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            goto dh_gex_clean_exit;
        }

        if(_libssh2_get_bignum_bytes(&buf, &g, &g_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            goto dh_gex_clean_exit;
        }

        _libssh2_bn_from_bin(key_state->p, p_len, p);
        _libssh2_bn_from_bin(key_state->g, g_len, g);

        ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p,
                                      p_len, 256,
                                      (void *)&exchange_hash_ctx,
                                      SSH_MSG_KEX_DH_GEX_INIT,
                                      SSH_MSG_KEX_DH_GEX_REPLY,
                                      key_state->data + 1,
                                      key_state->data_len - 1,
                                      &key_state->exchange_state);
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        LIBSSH2_FREE(session, key_state->data);
    }

  dh_gex_clean_exit:
    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;

    return ret;
}


```cpp

这段代码定义了一个名为 LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY 的函数，用于创建并验证 EC SHA 哈希。它接受以下参数：

- V_C：客户端的唯一标识（CR 和 LF 除外）
- V_S：服务器的唯一标识（CR 和 LF 除外）
- I_C：客户端的 SSH_MSG_KEXINIT 数据负载
- I_S：服务器的 SSH_MSG_KEXINIT 数据负载
- K_S：服务器的公共主机密钥
- Q_C：客户端的暂时代替的公钥（仅用于计算）
- Q_S：服务器的暂时代替的公钥（仅用于计算）
- K：共享秘密，必须是 0 或 4096。

这个函数首先创建一个包含客户端和服务器公钥的哈希，然后使用哈希算法生成一个消息，该消息包含客户端的暂时代替公钥和消息的哈希。服务器必须首先验证客户端提供的消息和哈希是否与它们所提供的哈希相同。


```
/* LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY
 *
 * Macro that create and verifies EC SHA hash with a given digest bytes
 *
 * Payload format:
 *
 * string   V_C, client's identification string (CR and LF excluded)
 * string   V_S, server's identification string (CR and LF excluded)
 * string   I_C, payload of the client's SSH_MSG_KEXINIT
 * string   I_S, payload of the server's SSH_MSG_KEXINIT
 * string   K_S, server's public host key
 * string   Q_C, client's ephemeral public key octet string
 * string   Q_S, server's ephemeral public key octet string
 * mpint    K,   shared secret
 *
 */

```cpp

This is a function definition for `_libssh2_htonu32()` which takes a
public key length and an optional server public key length
as input and converts the public key to UTF-8 encoding. The function
is a part of the SSH key signature algorithm, where it is used to
validate the server's public key against the client's
signature.

The function takes two arguments:

- `exchange_state`: An `SSHExchangeState` object that
contains the public key and the signature.
- `public_key`: A pointer to the public key.
- `public_key_len`: The length of the public key.

The function returns `0` on success and `-1` on failure.

Note: The


```
#define LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(digest_type)       \
{                                                                       \
    libssh2_sha##digest_type##_ctx ctx;                                 \
    exchange_state->exchange_hash = (void *)&ctx;                       \
    libssh2_sha##digest_type##_init(&ctx);                              \
    if(session->local.banner) {                                         \
        _libssh2_htonu32(exchange_state->h_sig_comp,                    \
                         strlen((char *) session->local.banner) - 2);   \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          exchange_state->h_sig_comp, 4); \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          (char *) session->local.banner, \
                                          strlen((char *)               \
                                                 session->local.banner) \
                                          - 2);                         \
    }                                                                   \
    else {                                                              \
        _libssh2_htonu32(exchange_state->h_sig_comp,                    \
                         sizeof(LIBSSH2_SSH_DEFAULT_BANNER) - 1);       \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          exchange_state->h_sig_comp, 4); \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          LIBSSH2_SSH_DEFAULT_BANNER,   \
                                          sizeof(LIBSSH2_SSH_DEFAULT_BANNER) \
                                          - 1);                         \
    }                                                                   \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     strlen((char *) session->remote.banner));          \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->remote.banner,           \
                                      strlen((char *)                   \
                                             session->remote.banner));  \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     session->local.kexinit_len);                       \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->local.kexinit,           \
                                      session->local.kexinit_len);      \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     session->remote.kexinit_len);                      \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->remote.kexinit,          \
                                      session->remote.kexinit_len);     \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     session->server_hostkey_len);                      \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->server_hostkey,          \
                                      session->server_hostkey_len);     \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     public_key_len);                                   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      public_key,                       \
                                      public_key_len);                  \
                                                                        \
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     server_public_key_len);                            \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      server_public_key,                \
                                      server_public_key_len);           \
                                                                        \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->k_value,          \
                                      exchange_state->k_value_len);     \
                                                                        \
    libssh2_sha##digest_type##_final(ctx, exchange_state->h_sig_comp);  \
                                                                        \
    if(session->hostkey->                                               \
       sig_verify(session, exchange_state->h_sig,                       \
                  exchange_state->h_sig_len, exchange_state->h_sig_comp, \
                  SHA##digest_type##_DIGEST_LENGTH,                     \
                  &session->server_hostkey_abstract)) {                 \
        rc = -1;                                                        \
    }                                                                   \
}                                                                       \


```cpp

这段代码是一个函数，名为`kex_session_ecdh_curve_type`。它用于根据输入的`name`变量，返回一个名为`out_type`的`libssh2_curve_type`类型的值，或者返回一个负数表示没有找到匹配的曲线类型。

函数的实现采用了`if`语句和`static`关键字，以保证函数的局部性。函数内部首先定义了一个`type`变量，用于记录输入曲线的名称，然后通过`strcmp`函数比较输入的`name`和三个预定义的名称，然后将匹配的名称转换为相应的`libssh2_ec_curve_type`类型，并将其存储在`type`变量中。如果未找到匹配的名称，函数返回一个负数。最后，函数检查传入的`out_type`参数，如果`out_type`参数没有被定义，函数将`type`变量作为`out_type`的默认值。函数返回0，表示成功实现了函数功能。


```
#if LIBSSH2_ECDSA

/* kex_session_ecdh_curve_type
 * returns the EC curve type by name used in key exchange
 */

static int
kex_session_ecdh_curve_type(const char *name, libssh2_curve_type *out_type)
{
    int ret = 0;
    libssh2_curve_type type;

    if(name == NULL)
        return -1;

    if(strcmp(name, "ecdh-sha2-nistp256") == 0)
        type = LIBSSH2_EC_CURVE_NISTP256;
    else if(strcmp(name, "ecdh-sha2-nistp384") == 0)
        type = LIBSSH2_EC_CURVE_NISTP384;
    else if(strcmp(name, "ecdh-sha2-nistp521") == 0)
        type = LIBSSH2_EC_CURVE_NISTP521;
    else {
        ret = -1;
    }

    if(ret == 0 && out_type) {
        *out_type = type;
    }

    return ret;
}


```cpp

接下来，您需要将上面获取到的服务器公钥与私钥一起发送，以便确认消息。具体操作如下：
```perl
   if(exchange_state->state == libssh2_NB_state_transferred) {
       exchange_state->last_data_len = data_len;
       exchange_state->flags |= EXCHANGE_FLAG_SEND_HOST;

       rc = _libssh2_send_message(session, session->sockfd,
                                     0, data, data_len, 0);
       if(rc < 0) {
           ret = _libssh2_error(session, LIBSSH2_ERROR_SEND,
                                   "Unable to send host key to the client");
           goto clean_exit;
       }
   }
```cpp
如果上面发送主机会，需要等待客户机回应确认，这个确认的值应该是“够”。
```perl
   return 0;
```cpp
以上代码中，我们通过 `_libssh2_copy_string` 函数将服务器公钥的长度拿到，然后将获取的公钥与私钥一起发送，以确认消息。然后在服务器端，我们需要等待客户端的确认，收到客户端发送的确认后，将更新状态，以便下一步处理。


```
/* ecdh_sha2_nistp
 * Elliptic Curve Diffie Hellman Key Exchange
 */

static int ecdh_sha2_nistp(LIBSSH2_SESSION *session, libssh2_curve_type type,
                           unsigned char *data, size_t data_len,
                           unsigned char *public_key,
                           size_t public_key_len, _libssh2_ec_key *private_key,
                           kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;

    if(data_len < 5) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                            "Host key data is too short");
        return ret;
    }

    if(exchange_state->state == libssh2_NB_state_idle) {

        /* Setup initial values */
        exchange_state->k = _libssh2_bn_init();

        exchange_state->state = libssh2_NB_state_created;
    }

    if(exchange_state->state == libssh2_NB_state_created) {
        /* parse INIT reply data */

        /* host key K_S */
        unsigned char *server_public_key;
        size_t server_public_key_len;
        struct string_buf buf;

        buf.data = data;
        buf.len = data_len;
        buf.dataptr = buf.data;
        buf.dataptr++; /* Advance past packet type */

         if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                                &server_public_key_len)) {
             ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for a copy "
                                  "of the host key");
            goto clean_exit;
        }

        session->server_hostkey_len = (uint32_t)server_public_key_len;

```cpp

这段代码是使用OpenSSH库中的libssh2_md5功能，用于对服务器主机的密钥进行哈希并判断其有效性。

具体来说，代码会首先尝试初始化libssh2_md5_ctx函数，如果初始成功，则会执行以下操作：

1. 创建一个名为fingerprint_ctx的libssh2_md5ctx对象。
2. 使用libssh2_md5_init函数对fingerprint_ctx进行初始化，传递一个服务器主机的私钥(session->server_hostkey)和其长度(session->server_hostkey_len)，并返回libssh2_md5_init函数的返回值。
3. 使用libssh2_md5_update函数对fingerprint_ctx的输入域进行更新，即将session->server_hostkey和session->server_hostkey_len传递给该函数，并获取更新后的摘要。
4. 使用libssh2_md5_final函数对fingerprint_ctx进行final化，即对摘要进行哈希并获取其哈希值。
5. 通过libssh2_md5_get_features函数获取服务器主机的哈希特征，并判断其是否有效。如果哈希特征有效，则返回TRUE，否则返回FALSE。
6. 将判断结果赋值给session->server_hostkey_md5_valid，即服务器主机的哈希特征是否有效。

这段代码的作用是使用libssh2_md5库对服务器主机的密钥进行哈希并判断其有效性，以便在需要时进行验证。


```
#if LIBSSH2_MD5
        {
            libssh2_md5_ctx fingerprint_ctx;

            if(libssh2_md5_init(&fingerprint_ctx)) {
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                session->server_hostkey_md5_valid = FALSE;
            }
        }
```cpp

这段代码的作用是检查服务器的主机密钥是否包含MD5算法，并输出服务器MD5指纹。

具体来说，代码分为两部分。第一部分是一个条件判断，如果MD5算法存在（通过`#ifdef LIBSSH2DEBUG`判断），那么执行第2部分代码。第2部分代码的作用是获取服务器的主机密钥，并输出MD5指纹。首先定义了一个长度为50个字符的指纹数组`fingerprint`，以及一个指向该数组的指针`fprint`。然后使用一个循环，遍历服务器主机的MD5指纹，将每三个连续的指纹复制到`fprint`数组中。接着将`fprint`数组的最后一个元素（也就是`fprint`所指向的内存位置）指向`\0`，以结束输入。最后通过`_libssh2_debug`函数将服务器MD5指纹打印出来。

第二部分代码的作用是获取服务器的主机密钥，并检查其是否包含MD5算法。具体来说，代码首先定义了一个`libssh2_sha1_ctx`类型的`fingerprint_ctx`变量，然后使用`libssh2_sha1_init`函数初始化该ctx。接着使用`libssh2_sha1_update`和`libssh2_sha1_final`函数获取服务器主机的MD5指纹，并使用`session->server_hostkey_len`获取服务器主机的实际长度。最后比较`fingerprint_ctx`和`server_hostkey`的长度是否相等，如果不相等，则说明服务器的主机密钥不包含MD5算法，从而输出`FALSE`。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[50], *fprint = fingerprint;
            int i;
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            libssh2_sha1_ctx fingerprint_ctx;

            if(libssh2_sha1_init(&fingerprint_ctx)) {
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
```cpp

这段代码的作用是输出服务器的主机密钥的哈希值。具体来说，它包含以下几个步骤：

1. 如果定义了名为 "LIBSSH2DEBUG" 的预处理指令，那么执行以下操作：

  a. 定义一个长度为 64 的字符数组 "fingerprint"，以及一个指向该数组的指针 "fprint"。

  b. 定义一个整数 "i"，用于计数。

  c. 使用 for 循环，从 0 到 19 遍历。每次循环执行以下操作：

     - 计算客户端服务器哈希库中第 "i" 个哈希值的 ASCII 值。

     - 将 ASCII 值存储到 "fprint" 指向的下一个位置。

     - 循环结束后，通过 `_libssh2_debug` 函数将服务器的主机密钥的哈希值输出到控制台。

2. 如果定义了名为 "libssh2DEBUG" 的预处理指令，那么执行以下操作：

  a. 定义一个名为 "fingerprint_ctx" 的 libssh2_sha256_ctx 结构体指针。

  b. 如果成功创建并初始化 "fingerprint_ctx"，那么执行以下操作：

     - 使用 libssh2_sha256_init 函数初始化 "fingerprint_ctx" 的状态。

     - 使用 libssh2_sha256_update 函数将客户端服务器哈希库中的内容更新到 "fingerprint_ctx" 的状态。

     - 使用 libssh2_sha256_final 函数获取 "fingerprint_ctx" 状态下的哈希值。

     - 通过调用 session->server_hostkey_sha256 获取服务器的主机密钥，然后判断其是否有效。如果有效，则输出服务器的主机密钥的哈希值。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[64], *fprint = fingerprint;
            int i;

            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */

        /* SHA256 */
        {
            libssh2_sha256_ctx fingerprint_ctx;

            if(libssh2_sha256_init(&fingerprint_ctx)) {
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
```cpp

This is a function definition for the `init()` method of the `SSHKey` class in the OpenSSH library. The function appears to perform initialization for the HMAC key when a server client key is generated. It does this by setting up the HMAC abstract, cleaning up any existing compression, and initializing the compression for each direction (client to server, server to client).

The function takes three arguments: `session` is the session object, `key` is the generated key, and `free_key` is a pointer to the key to be freed when the function returns. It returns `0` on success, or a specific error code on failure.


```
#ifdef LIBSSH2DEBUG
        {
            char *base64Fingerprint = NULL;
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            if(base64Fingerprint != NULL) {
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
#endif /* LIBSSH2DEBUG */

        if(session->hostkey->init(session, session->server_hostkey,
                                   session->server_hostkey_len,
                                   &session->server_hostkey_abstract)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unable to initialize hostkey importer");
            goto clean_exit;
        }

        /* server public key Q_S */
        if(_libssh2_get_string(&buf, &server_public_key,
                               &server_public_key_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                     "Unexpected key length");
            goto clean_exit;
        }

        /* server signature */
        if(_libssh2_get_string(&buf, &exchange_state->h_sig,
           &(exchange_state->h_sig_len))) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unexpected ecdh server sig length");
            goto clean_exit;
        }

        /* Compute the shared secret K */
        rc = _libssh2_ecdh_gen_k(&exchange_state->k, private_key,
                                 server_public_key, server_public_key_len);
        if(rc != 0) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_KEX_FAILURE,
                                 "Unable to create ECDH shared secret");
            goto clean_exit;
        }

        exchange_state->k_value_len = _libssh2_bn_bytes(exchange_state->k) + 5;
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            /* don't need leading 00 */
            exchange_state->k_value_len--;
        }
        exchange_state->k_value =
        LIBSSH2_ALLOC(session, exchange_state->k_value_len);
        if(!exchange_state->k_value) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate buffer for K");
            goto clean_exit;
        }
        _libssh2_htonu32(exchange_state->k_value,
                         exchange_state->k_value_len - 4);
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 4);
        }
        else {
            exchange_state->k_value[4] = 0;
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 5);
        }

        /* verify hash */

        switch(type) {
            case LIBSSH2_EC_CURVE_NISTP256:
                LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(256);
                break;

            case LIBSSH2_EC_CURVE_NISTP384:
                LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(384);
                break;
            case LIBSSH2_EC_CURVE_NISTP521:
                LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(512);
                break;
        }

        if(rc != 0) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_SIGN,
                                 "Unable to verify hostkey signature");
            goto clean_exit;
        }

        exchange_state->c = SSH_MSG_NEWKEYS;
        exchange_state->state = libssh2_NB_state_sent;
    }

    if(exchange_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send NEWKEYS message");
            goto clean_exit;
        }

        exchange_state->state = libssh2_NB_state_sent2;
    }

    if(exchange_state->state == libssh2_NB_state_sent2) {
        rc = _libssh2_packet_require(session, SSH_MSG_NEWKEYS,
                                     &exchange_state->tmp,
                                     &exchange_state->tmp_len, 0, NULL, 0,
                                     &exchange_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc, "Timed out waiting for NEWKEYS");
            goto clean_exit;
        }

        /* The first key exchange has been performed,
         switch to active crypt/comp/mac mode */
        session->state |= LIBSSH2_STATE_NEWKEYS;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Received NEWKEYS message");

        /* This will actually end up being just packet_type(1)
         for this packet type anyway */
        LIBSSH2_FREE(session, exchange_state->tmp);

        if(!session->session_id) {

            size_t digest_length = 0;

            if(type == LIBSSH2_EC_CURVE_NISTP256)
                digest_length = SHA256_DIGEST_LENGTH;
            else if(type == LIBSSH2_EC_CURVE_NISTP384)
                digest_length = SHA384_DIGEST_LENGTH;
            else if(type == LIBSSH2_EC_CURVE_NISTP521)
                digest_length = SHA512_DIGEST_LENGTH;
            else{
                ret = _libssh2_error(session, LIBSSH2_ERROR_KEX_FAILURE,
                                     "Unknown SHA digest for EC curve");
                goto clean_exit;

            }
            session->session_id = LIBSSH2_ALLOC(session, digest_length);
            if(!session->session_id) {
                ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                     "Unable to allocate buffer for "
                                     "SHA digest");
                goto clean_exit;
            }
            memcpy(session->session_id, exchange_state->h_sig_comp,
                   digest_length);
             session->session_id_len = digest_length;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "session_id calculated");
        }

        /* Cleanup any existing cipher */
        if(session->local.crypt->dtor) {
            session->local.crypt->dtor(session,
                                       &session->local.crypt_abstract);
        }

        /* Calculate IV/Secret/Key for each direction */
        if(session->local.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(iv,
                                                 session->local.crypt->
                                                 iv_len, "A");
            if(!iv) {
                ret = -1;
                goto clean_exit;
            }

            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(secret,
                                                session->local.crypt->
                                                secret_len, "C");

            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->local.crypt->
                init(session, session->local.crypt, iv, &free_iv, secret,
                     &free_secret, 1, &session->local.crypt_abstract)) {
                    LIBSSH2_FREE(session, iv);
                    LIBSSH2_FREE(session, secret);
                    ret = LIBSSH2_ERROR_KEX_FAILURE;
                    goto clean_exit;
                }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->local.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->local.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server IV and Key calculated");

        if(session->remote.crypt->dtor) {
            /* Cleanup any existing cipher */
            session->remote.crypt->dtor(session,
                                        &session->remote.crypt_abstract);
        }

        if(session->remote.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(iv,
                                                 session->remote.crypt->
                                                 iv_len, "B");

            if(!iv) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(secret,
                                                 session->remote.crypt->
                                                 secret_len, "D");

            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->remote.crypt->
                init(session, session->remote.crypt, iv, &free_iv, secret,
                     &free_secret, 0, &session->remote.crypt_abstract)) {
                    LIBSSH2_FREE(session, iv);
                    LIBSSH2_FREE(session, secret);
                    ret = LIBSSH2_ERROR_KEX_FAILURE;
                    goto clean_exit;
                }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->remote.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->remote.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client IV and Key calculated");

        if(session->local.mac->dtor) {
            session->local.mac->dtor(session, &session->local.mac_abstract);
        }

        if(session->local.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(key,
                                                 session->local.mac->
                                                 key_len, "E");

            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->local.mac->init(session, key, &free_key,
                                     &session->local.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->local.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server HMAC Key calculated");

        if(session->remote.mac->dtor) {
            session->remote.mac->dtor(session, &session->remote.mac_abstract);
        }

        if(session->remote.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(key,
                                                 session->remote.mac->
                                                 key_len, "F");

            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->remote.mac->init(session, key, &free_key,
                                      &session->remote.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->remote.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client HMAC Key calculated");

        /* Initialize compression for each direction */

        /* Cleanup any existing compression */
        if(session->local.comp && session->local.comp->dtor) {
            session->local.comp->dtor(session, 1,
                                      &session->local.comp_abstract);
        }

        if(session->local.comp && session->local.comp->init) {
            if(session->local.comp->init(session, 1,
                                          &session->local.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Client to Server compression initialized");

        if(session->remote.comp && session->remote.comp->dtor) {
            session->remote.comp->dtor(session, 0,
                                       &session->remote.comp_abstract);
        }

        if(session->remote.comp && session->remote.comp->init) {
            if(session->remote.comp->init(session, 0,
                                           &session->remote.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Server to Client compression initialized");

    }

```cpp

这段代码是关于干净退出（clean up）交易所的函数，主要作用是释放之前分配的 Elliptic Curve Diffie Hellman 密钥。

首先，通过调用 _libssh2_bn_free(exchange_state->k) 函数，释放了之前分配的 Diffie Hellman 密钥，然后将该密钥的指针设置为 NULL，以确保不会被使用。

接下来，代码检查 exchange_state->k_value 是否为真，如果是，则通过调用 LIBSSH2_FREE(session, exchange_state->k_value) 函数释放该密钥，并将其设置为 NULL，以确保干净退出。

最后，将 exchange_state->state 设置为 libssh2_NB_state_idle，并返回干净退出状态的 ret 值。

总之，这段代码的主要目的是释放之前分配的 Elliptic Curve Diffie Hellman 密钥，确保干净退出。


```
clean_exit:
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;

    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    exchange_state->state = libssh2_NB_state_idle;

    return ret;
}

/* kex_method_ecdh_key_exchange
 *
 * Elliptic Curve Diffie Hellman Key Exchange
 * supports SHA256/384/512 hashes based on negotated ecdh method
 *
 */

```cpp

This function appears to handle the key confirmation (ECDH) process when using SSH2 protocol. It is responsible for sending the ECDH initiallation message, waiting for the response, and sending the ECDH reply message if required.

If there is an error sending the ECDH initiallation message, it will return the error code. If the key is not returned within the specified timeout, it will return LIBSSH2_ERROR_EAGAIN.

If the ECDH reply is received, it checks the type of the curve and performs a SHA2 hash on the key to ensure its authenticity. If the key is valid, the function returns the estimated number of octets used for the key in the EstimatedShare secret.

If an error occurs, it is printed out and the function returns LIBSSH2_ERROR.


```
static int
kex_method_ecdh_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc = 0;
    unsigned char *s;
    libssh2_curve_type type;

    if(key_state->state == libssh2_NB_state_idle) {

        key_state->public_key_oct = NULL;
        key_state->state = libssh2_NB_state_created;
    }

    if(key_state->state == libssh2_NB_state_created) {
        rc = kex_session_ecdh_curve_type(session->kex->name, &type);

        if(rc != 0) {
            ret = _libssh2_error(session, -1,
                                 "Unknown KEX nistp curve type");
            goto ecdh_clean_exit;
        }

        rc = _libssh2_ecdsa_create_key(session, &key_state->private_key,
                                       &key_state->public_key_oct,
                                       &key_state->public_key_oct_len, type);

        if(rc != 0) {
            ret = _libssh2_error(session, rc,
                                 "Unable to create private key");
            goto ecdh_clean_exit;
        }

        key_state->request[0] = SSH2_MSG_KEX_ECDH_INIT;
        s = key_state->request + 1;
        _libssh2_store_str(&s, (const char *)key_state->public_key_oct,
                           key_state->public_key_oct_len);
        key_state->request_len = key_state->public_key_oct_len + 5;

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating ECDH SHA2 NISTP256");

        key_state->state = libssh2_NB_state_sent;
    }

    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send ECDH_INIT");
            goto ecdh_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent1;
    }

    if(key_state->state == libssh2_NB_state_sent1) {
        rc = _libssh2_packet_require(session, SSH2_MSG_KEX_ECDH_REPLY,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for ECDH_REPLY reply");
            goto ecdh_clean_exit;
        }

        key_state->state = libssh2_NB_state_sent2;
    }

    if(key_state->state == libssh2_NB_state_sent2) {

        (void)kex_session_ecdh_curve_type(session->kex->name, &type);

        ret = ecdh_sha2_nistp(session, type, key_state->data,
                              key_state->data_len,
                              (unsigned char *)key_state->public_key_oct,
                              key_state->public_key_oct_len,
                              key_state->private_key,
                              &key_state->exchange_state);

        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        LIBSSH2_FREE(session, key_state->data);
    }

```cpp

这段代码是用于在 SSH 连接中处理 ECDSA 密钥的成功或失败。它主要实现了以下几个步骤：

1. 检查密钥状态中的 public_key_oct 和 private_key 是否为非空指针。如果是，那么释放这两个密钥并设置为 NULL，使得后续操作不再依赖它们。
2. 将密钥状态中的 state 设置为 libssh2_NB_state_idle，表明处于空闲状态，等待连接建立。
3. 在函数结束时返回 libssh2_老子。

简而言之，这段代码的作用是确保在 ECDSA 密钥连接建立成功后，释放 public_key_oct 和 private_key，并设置 state 为 idle，然后返回连接状态。


```
ecdh_clean_exit:

    if(key_state->public_key_oct) {
        LIBSSH2_FREE(session, key_state->public_key_oct);
        key_state->public_key_oct = NULL;
    }

    if(key_state->private_key) {
        _libssh2_ecdsa_free(key_state->private_key);
        key_state->private_key = NULL;
    }

    key_state->state = libssh2_NB_state_idle;

    return ret;
}

```cpp

It looks like there is a problem with the key data that is being passed to the server. Specifically, the message "Data is too short" suggests that the data being passed is not long enough to cover the entire key.

One possible solution would be to retrieve the remaining data in the buffer and try to use that to generate a new key. To do this, you would need to check the length of the remaining data in the buffer and make sure that it is greater than zero. If the remaining data is too short, you would need to generate more data or data_len is too short.

Another possible solution would be to check the server's public key to see if it is able to generate a new key. It is possible that the server's public key is not able to generate a new key, in which case you would need to generate a new one or use an older key.

It is also possible that there is a problem with the data being passed to the server, and that the key generation process is failing. In this case, you would need to investigate further to identify the root cause and take appropriate action.


```
#endif /*LIBSSH2_ECDSA*/


#if LIBSSH2_ED25519

/* curve25519_sha256
 * Elliptic Curve Key Exchange
 */

static int
curve25519_sha256(LIBSSH2_SESSION *session, unsigned char *data,
                  size_t data_len,
                  unsigned char public_key[LIBSSH2_ED25519_KEY_LEN],
                  unsigned char private_key[LIBSSH2_ED25519_KEY_LEN],
                  kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;
    int public_key_len = LIBSSH2_ED25519_KEY_LEN;

    if(data_len < 5) {
        return _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                              "Data is too short");
    }

    if(exchange_state->state == libssh2_NB_state_idle) {

        /* Setup initial values */
        exchange_state->k = _libssh2_bn_init();

        exchange_state->state = libssh2_NB_state_created;
    }

    if(exchange_state->state == libssh2_NB_state_created) {
        /* parse INIT reply data */
        unsigned char *server_public_key, *server_host_key;
        size_t server_public_key_len, hostkey_len;
        struct string_buf buf;

        if(data_len < 5) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto clean_exit;
        }

        buf.data = data;
        buf.len = data_len;
        buf.dataptr = buf.data;
        buf.dataptr++; /* advance past packet type */

        if(_libssh2_get_string(&buf, &server_host_key, &hostkey_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto clean_exit;
        }

        session->server_hostkey_len = (uint32_t)hostkey_len;
        session->server_hostkey = LIBSSH2_ALLOC(session,
                                                session->server_hostkey_len);
        if(!session->server_hostkey) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate memory for a copy "
                                 "of the host key");
            goto clean_exit;
        }

        memcpy(session->server_hostkey, server_host_key,
               session->server_hostkey_len);

```cpp

这段代码是一个 if 语句，它判断是否支持库文件 libssh2_md5，然后会做以下操作：

1. 初始化 libssh2_md5 库文件

```
libssh2_md5_init(&fingerprint_ctx);
```cpp

2. 接收服务器主机密钥

```
libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                  session->server_hostkey_len);
```cpp

3. 计算服务器主机密钥的 MD5 值

```
libssh2_md5_final(fingerprint_ctx,
                                 session->server_hostkey_md5);
```cpp

4. 判断服务器主机密钥的 MD5 值是否有效

```
session->server_hostkey_md5_valid = TRUE;
```cpp

这段代码的作用是检查服务器主机密钥是否正确，然后判断是否有效，如果正确，则返回 TRUE，否则返回 FALSE。


```
#if LIBSSH2_MD5
        {
            libssh2_md5_ctx fingerprint_ctx;

            if(libssh2_md5_init(&fingerprint_ctx)) {
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                session->server_hostkey_md5_valid = FALSE;
            }
        }
```cpp

这段代码包含两个条件判断和库函数调用。

第一个条件判断是判断是否启用了名为 "LIBSSH2DEBUG" 的库文件，如果是，那么就执行下面的代码。这段代码的作用是提取服务器主机的 MD5 指纹，并输出它的值。

第二个条件判断是判断是否支持 LIBSSH2_MD5 库文件。如果不是，那么就不执行下面的代码，否则就执行。这段代码的作用是获取服务器主机的指纹，并使用 LIBSSH2_MD5 库函数将指纹输出。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[50], *fprint = fingerprint;
            int i;
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                             "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            libssh2_sha1_ctx fingerprint_ctx;

            if(libssh2_sha1_init(&fingerprint_ctx)) {
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
```cpp

这段代码是一个 C 语言程序，它有以下两个主要部分：

1. 判断是否定义了名为 "LIBSSH2DEBUG" 的预处理指令。如果不定义，程序将跳过预处理指令部分，直接执行下面的内容。
2. 如果定义了 "LIBSSH2DEBUG"，那么执行以下操作：
a. 定义一个名为 "fingerprint" 的字符数组，其大小为 64。
b. 定义一个名为 "fprint" 的指向字符数组的指针。
c. 定义一个名为 "i" 的整数变量，用于计数器。
d. 进入循环，计数器从 0 开始，每次增加 2，直到计数器大于 20 时跳出循环。
e. 在循环内部，使用 snprintf 函数将指纹的每一行输出，每行输出 4 个字符，并输出一个 2 进制字符。
f. 将 fprint 的指针向前移动一位，为下一行的输出腾出位置。
g. 调用名为 "libssh2_debug" 的函数，传递参数 "session" 和 "LIBSSH2DEBUG"，用于输出服务器 SHA1 密钥的指纹。
i. 如果成功创建并完成指纹验证，将服务器服务器的主机密钥的 SHA256 有效标志设置为 TRUE。

接下来，程序会执行第二个部分，这一部分使用 SHA256 算法生成服务器的主机密钥的 SHA256 值。


```
#ifdef LIBSSH2DEBUG
        {
            char fingerprint[64], *fprint = fingerprint;
            int i;

            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            *(--fprint) = '\0';
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                             "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */

        /* SHA256 */
        {
            libssh2_sha256_ctx fingerprint_ctx;

            if(libssh2_sha256_init(&fingerprint_ctx)) {
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
```cpp

This is a Java function that performs the HMAC key exchange between a server and a client using the OpenSSH protocol. The HMAC key is used for encryption and decryption of the exchanged data.

The function takes three arguments:

- session: An instance of the SSH2Session class that represents the server session.
- key: An array of bytes that contains the server's HMAC key.
- free_key: An array of bytes that contains the server's free key after generating the HMAC key. The free key is used for encryption only.
- session->remote.mac: An instance of the MAC class that represents the client's HMAC key.
- session->remote.mac_abstract: An abstract representation of the client's HMAC key.

The function first initializes the remote MAC node with the server's HMAC key and then compares the server's free key with the free key to be sure that the server is not trying to send the free key to the client. If the server is sending the free key, the function calculates the HMAC key and returns it.

If the server's HMAC key is not equal to the free key, the function will return LIBSSH2_ERROR_KEX_FAILURE.

The function also initializes the compression for each direction (client to server and server to client) and cleans up any existing compression if it exists.

Finally, the function logs some debug information if the initialization was successful.


```
#ifdef LIBSSH2DEBUG
        {
            char *base64Fingerprint = NULL;
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            if(base64Fingerprint != NULL) {
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
#endif /* LIBSSH2DEBUG */

        if(session->hostkey->init(session, session->server_hostkey,
                                   session->server_hostkey_len,
                                   &session->server_hostkey_abstract)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unable to initialize hostkey importer");
            goto clean_exit;
        }

        /* server public key Q_S */
        if(_libssh2_get_string(&buf, &server_public_key,
                               &server_public_key_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                     "Unexpected key length");
            goto clean_exit;
        }

        if(server_public_key_len != LIBSSH2_ED25519_KEY_LEN) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unexpected curve25519 server "
                                 "public key length");
            goto clean_exit;
        }

        /* server signature */
        if(_libssh2_get_string(&buf, &exchange_state->h_sig,
           &(exchange_state->h_sig_len))) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                                 "Unexpected curve25519 server sig length");
            goto clean_exit;
        }

        /* Compute the shared secret K */
        rc = _libssh2_curve25519_gen_k(&exchange_state->k, private_key,
                                       server_public_key);
        if(rc != 0) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_KEX_FAILURE,
                                 "Unable to create ECDH shared secret");
            goto clean_exit;
        }

        exchange_state->k_value_len = _libssh2_bn_bytes(exchange_state->k) + 5;
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            /* don't need leading 00 */
            exchange_state->k_value_len--;
        }
        exchange_state->k_value =
        LIBSSH2_ALLOC(session, exchange_state->k_value_len);
        if(!exchange_state->k_value) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate buffer for K");
            goto clean_exit;
        }
        _libssh2_htonu32(exchange_state->k_value,
                         exchange_state->k_value_len - 4);
        if(_libssh2_bn_bits(exchange_state->k) % 8) {
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 4);
        }
        else {
            exchange_state->k_value[4] = 0;
            _libssh2_bn_to_bin(exchange_state->k, exchange_state->k_value + 5);
        }

        /*/ verify hash */
        LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(256);

        if(rc != 0) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_SIGN,
                                 "Unable to verify hostkey signature");
            goto clean_exit;
        }

        exchange_state->c = SSH_MSG_NEWKEYS;
        exchange_state->state = libssh2_NB_state_sent;
    }

    if(exchange_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send NEWKEYS message");
            goto clean_exit;
        }

        exchange_state->state = libssh2_NB_state_sent2;
    }

    if(exchange_state->state == libssh2_NB_state_sent2) {
        rc = _libssh2_packet_require(session, SSH_MSG_NEWKEYS,
                                     &exchange_state->tmp,
                                     &exchange_state->tmp_len, 0, NULL, 0,
                                     &exchange_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc, "Timed out waiting for NEWKEYS");
            goto clean_exit;
        }

        /* The first key exchange has been performed, switch to active
           crypt/comp/mac mode */

        session->state |= LIBSSH2_STATE_NEWKEYS;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Received NEWKEYS message");

        /* This will actually end up being just packet_type(1) for this packet
           type anyway */
        LIBSSH2_FREE(session, exchange_state->tmp);

        if(!session->session_id) {

            size_t digest_length = SHA256_DIGEST_LENGTH;
            session->session_id = LIBSSH2_ALLOC(session, digest_length);
            if(!session->session_id) {
                ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                     "Unable to allxcocate buffer for "
                                     "SHA digest");
                goto clean_exit;
            }
            memcpy(session->session_id, exchange_state->h_sig_comp,
                   digest_length);
            session->session_id_len = digest_length;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "session_id calculated");
        }

        /* Cleanup any existing cipher */
        if(session->local.crypt->dtor) {
            session->local.crypt->dtor(session,
                                        &session->local.crypt_abstract);
        }

        /* Calculate IV/Secret/Key for each direction */
        if(session->local.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, iv,
                                              session->local.crypt->
                                              iv_len, "A");
            if(!iv) {
                ret = -1;
                goto clean_exit;
            }

            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, secret,
                                              session->local.crypt->
                                              secret_len, "C");

            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->local.crypt->
                init(session, session->local.crypt, iv, &free_iv, secret,
                     &free_secret, 1, &session->local.crypt_abstract)) {
                    LIBSSH2_FREE(session, iv);
                    LIBSSH2_FREE(session, secret);
                    ret = LIBSSH2_ERROR_KEX_FAILURE;
                    goto clean_exit;
                }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->local.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->local.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Client to Server IV and Key calculated");

        if(session->remote.crypt->dtor) {
            /* Cleanup any existing cipher */
            session->remote.crypt->dtor(session,
                                        &session->remote.crypt_abstract);
        }

        if(session->remote.crypt->init) {
            unsigned char *iv = NULL, *secret = NULL;
            int free_iv = 0, free_secret = 0;

            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, iv,
                                              session->remote.crypt->
                                              iv_len, "B");

            if(!iv) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, secret,
                                              session->remote.crypt->
                                              secret_len, "D");

            if(!secret) {
                LIBSSH2_FREE(session, iv);
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            if(session->remote.crypt->
                init(session, session->remote.crypt, iv, &free_iv, secret,
                     &free_secret, 0, &session->remote.crypt_abstract)) {
                    LIBSSH2_FREE(session, iv);
                    LIBSSH2_FREE(session, secret);
                    ret = LIBSSH2_ERROR_KEX_FAILURE;
                    goto clean_exit;
                }

            if(free_iv) {
                _libssh2_explicit_zero(iv, session->remote.crypt->iv_len);
                LIBSSH2_FREE(session, iv);
            }

            if(free_secret) {
                _libssh2_explicit_zero(secret,
                                       session->remote.crypt->secret_len);
                LIBSSH2_FREE(session, secret);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Server to Client IV and Key calculated");

        if(session->local.mac->dtor) {
            session->local.mac->dtor(session, &session->local.mac_abstract);
        }

        if(session->local.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, key,
                                              session->local.mac->
                                              key_len, "E");

            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->local.mac->init(session, key, &free_key,
                                     &session->local.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->local.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Client to Server HMAC Key calculated");

        if(session->remote.mac->dtor) {
            session->remote.mac->dtor(session, &session->remote.mac_abstract);
        }

        if(session->remote.mac->init) {
            unsigned char *key = NULL;
            int free_key = 0;

            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, key,
                                              session->remote.mac->
                                              key_len, "F");

            if(!key) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
            session->remote.mac->init(session, key, &free_key,
                                      &session->remote.mac_abstract);

            if(free_key) {
                _libssh2_explicit_zero(key, session->remote.mac->key_len);
                LIBSSH2_FREE(session, key);
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Server to Client HMAC Key calculated");

        /* Initialize compression for each direction */

        /* Cleanup any existing compression */
        if(session->local.comp && session->local.comp->dtor) {
            session->local.comp->dtor(session, 1,
                                      &session->local.comp_abstract);
        }

        if(session->local.comp && session->local.comp->init) {
            if(session->local.comp->init(session, 1,
                                            &session->local.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Client to Server compression initialized");

        if(session->remote.comp && session->remote.comp->dtor) {
            session->remote.comp->dtor(session, 0,
                                        &session->remote.comp_abstract);
        }

        if(session->remote.comp && session->remote.comp->init) {
            if(session->remote.comp->init(session, 0,
                                             &session->remote.comp_abstract)) {
                ret = LIBSSH2_ERROR_KEX_FAILURE;
                goto clean_exit;
            }
        }
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Server to Client compression initialized");
    }

```cpp

这段代码是关于kex_method_curve25519_key_exchange函数的作用说明。该函数的主要作用是实现了椭圆曲线X25519的私钥签名和私钥恢复，并支持SHA256哈希。

首先，函数通过调用_libssh2_bn_free函数释放了传入的椭圆曲线的关键。然后，对输入的私钥数据执行了以下操作：

1. 如果私钥数据不为零，释放私钥数据并将其设置为零。
2. 将状态设置为libssh2_NB_state_idle。

然后，函数会输出一个ret值，代表着是否成功执行了私钥签名或私钥恢复操作。


```
clean_exit:
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;

    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    exchange_state->state = libssh2_NB_state_idle;

    return ret;
}

/* kex_method_curve25519_key_exchange
 *
 * Elliptic Curve X25519 Key Exchange with SHA256 hash
 *
 */

```cpp

This code is part of the OpenSSH library, which implements the SSH protocol.
It is an if statement that checks the current state of the key_state pointer.

The state of the key_state pointer can be one of the following:

* libssh2_NB_state_sent1: This state is entered when the initial key encryption is done and the server has sent the initial key data to the client. The key_state structure contains the initial key data and the public key used for the initial key encryption.
* libssh2_NB_state_sent2: This state is entered when the initial key encryption is done and the client has sent the initial key data to the server. The key_state structure contains the initial key data that the server will use to encrypt the key.

If the key_state pointer is in the libssh2_NB_state_sent1 state, it is the server that sends the ECDH_INIT message to the client, and it is possible to send the ECDH_REPLY message to the client using the LIBSSH2_transport_send function.

If the key_state pointer is in the libssh2_NB_state_sent2 state, it is the client that sends the ECDH_REPLY message to the server, and it is possible to receive the key data from the server using the curve25519_sha256 function and the LIBSSH2_packet\_require function.


```
static int
kex_method_curve25519_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc = 0;

    if(key_state->state == libssh2_NB_state_idle) {

        key_state->public_key_oct = NULL;
        key_state->state = libssh2_NB_state_created;
    }

    if(key_state->state == libssh2_NB_state_created) {
        unsigned char *s = NULL;

        rc = strcmp(session->kex->name, "curve25519-sha256@libssh.org");
        if(rc != 0)
            rc = strcmp(session->kex->name, "curve25519-sha256");

        if(rc != 0) {
            ret = _libssh2_error(session, -1,
                                 "Unknown KEX curve25519 curve type");
            goto clean_exit;
        }

        rc = _libssh2_curve25519_new(session,
                                     &key_state->curve25519_public_key,
                                     &key_state->curve25519_private_key);

        if(rc != 0) {
            ret = _libssh2_error(session, rc,
                                 "Unable to create private key");
            goto clean_exit;
        }

        key_state->request[0] = SSH2_MSG_KEX_ECDH_INIT;
        s = key_state->request + 1;
        _libssh2_store_str(&s, (const char *)key_state->curve25519_public_key,
                           LIBSSH2_ED25519_KEY_LEN);
        key_state->request_len = LIBSSH2_ED25519_KEY_LEN + 5;

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Initiating curve25519 SHA2");

        key_state->state = libssh2_NB_state_sent;
    }

    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send ECDH_INIT");
            goto clean_exit;
        }

        key_state->state = libssh2_NB_state_sent1;
    }

    if(key_state->state == libssh2_NB_state_sent1) {
        rc = _libssh2_packet_require(session, SSH2_MSG_KEX_ECDH_REPLY,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for ECDH_REPLY reply");
            goto clean_exit;
        }

        key_state->state = libssh2_NB_state_sent2;
    }

    if(key_state->state == libssh2_NB_state_sent2) {

        ret = curve25519_sha256(session, key_state->data, key_state->data_len,
                                key_state->curve25519_public_key,
                                key_state->curve25519_private_key,
                                &key_state->exchange_state);

        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        LIBSSH2_FREE(session, key_state->data);
    }

```cpp

这段代码是用来在SSH客户端和服务器之间进行通信的安全套接字（SSH）库中的一个函数，主要负责清除客户端和服务器之间的加密密钥。

具体来说，这段代码的作用是：

1. 如果客户端发送的密钥是公钥，则使用libssh2库中的explicit_zero函数将其清零，并使用free函数释放内存。然后将公钥设置为空，以保证下次不会被使用。

2. 如果客户端发送的密钥是私钥，则使用libssh2库中的explicit_zero函数将其清零，并使用free函数释放内存。然后将私钥设置为空，以保证下次不会被使用。

3. 更改客户端的狀態為libssh2库中的nb_state_idle，以便在将来执行其他操作。

4. 返回客户端传递给函数的整数，表示SSH库返回的结果。


```
clean_exit:

    if(key_state->curve25519_public_key) {
        _libssh2_explicit_zero(key_state->curve25519_public_key,
                               LIBSSH2_ED25519_KEY_LEN);
        LIBSSH2_FREE(session, key_state->curve25519_public_key);
        key_state->curve25519_public_key = NULL;
    }

    if(key_state->curve25519_private_key) {
        _libssh2_explicit_zero(key_state->curve25519_private_key,
                               LIBSSH2_ED25519_KEY_LEN);
        LIBSSH2_FREE(session, key_state->curve25519_private_key);
        key_state->curve25519_private_key = NULL;
    }

    key_state->state = libssh2_NB_state_idle;

    return ret;
}


```cpp

这段代码定义了两个常量，分别定义了 LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY 和 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 的值。

然后，定义了一个名为 kex_method_diffie_helman_group1_sha1 的结构体，包含了两个字段，分别是一个字符串类型的变量 diffie_hellman_group1_sha1，表示用于键交换的算法，另一个是整型变量 kex_method_diffie_hellman_group1_sha1_key_exchange，表示该算法支持的主机密钥类型。此外，该结构体还包含了一个 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 的字段，表示是否需要对签名的主机密钥进行校验。

接下来，定义了两个名为 kex_method_diffie_helman_group14_sha1 和 kex_method_diffie_helman_group1_sha1 的常量，分别表示 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 和 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 的值，与上面定义的 LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY 和 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 值保持一致。

最后，通过循环，将定义好的 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 和 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 常量复制到链表中，链表的第一个节点就是 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 的下一个节点，这样就定义了一个 LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY 的链表。


```
#endif /*LIBSSH2_ED25519*/


#define LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY     0x0001
#define LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY    0x0002

static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group1_sha1 = {
    "diffie-hellman-group1-sha1",
    kex_method_diffie_hellman_group1_sha1_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group14_sha1 = {
    "diffie-hellman-group14-sha1",
    kex_method_diffie_hellman_group14_sha1_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

```cpp

这段代码定义了三种在SSH客户端和服务器之间使用的密钥交换算法，包括Diffie-Hellman Group14和Diffie-Hellman Group16。这些算法使用不同的算法名称和标志，以便在将来的代码中更清楚地命名这些算法。

具体来说，这些算法实现了一个非对称加密密钥交换，客户端发送一个大小为256位的公钥到服务器，服务器随机生成一个大小为256位的私钥与客户端的公共 key 进行替换，然后服务器将客户端的公众密钥与服务器随机生成的私钥一起发送给客户端，客户端计算出与服务器服务器共享的密钥。

这些算法都是基于Diffie-Hellman算法的，其中第一个算法是Diffie-Hellman Group14，第二个算法是Diffie-Hellman Group16。第三个算法没有被实现。


```
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group14_sha256 = {
    "diffie-hellman-group14-sha256",
    kex_method_diffie_hellman_group14_sha256_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group16_sha512 = {
    "diffie-hellman-group16-sha512",
    kex_method_diffie_hellman_group16_sha512_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group18_sha512 = {
    "diffie-hellman-group18-sha512",
    kex_method_diffie_hellman_group18_sha512_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

```cpp

这段代码定义了两个常量，分别定义了使用Diffie-Hellman-Group-Exchange-SHA1和Diffie-Hellman-Group-Exchange-SHA256签名算法时的KeL流泪方法。

这两种算法都是基于Diffie-Hellman算法的集团交换算法，通常用于实现SSH协议的安全性。

其中，Diffie-Hellman-Group-Exchange-SHA1算法使用的是16-字节的SHA1算法，而Diffie-Hellman-Group-Exchange-SHA256算法使用的是256字节的SHA256算法。

这两种算法都要求用户输入一个哈希标识符(hostkey)，用于标识主机，同时要求用户输入一个非ce(counter-signature)，用于防止重放攻击。

KeL流泪方法使用libssh2库中的ECDSA(Extensible SSH数据的Atomic Signature Algorithm)进行签名，并使用算法自带的标签(LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY)。


```
static const LIBSSH2_KEX_METHOD
kex_method_diffie_helman_group_exchange_sha1 = {
    "diffie-hellman-group-exchange-sha1",
    kex_method_diffie_hellman_group_exchange_sha1_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD
kex_method_diffie_helman_group_exchange_sha256 = {
    "diffie-hellman-group-exchange-sha256",
    kex_method_diffie_hellman_group_exchange_sha256_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

#if LIBSSH2_ECDSA
```cpp

这段代码定义了两个常量，LIBSSH2_KEX_METHOD_ECDH_SHA2_NISTP256和LIBSSH2_KEX_METHOD_ECDH_SHA2_NISTP384，以及一个名为kex_method_ecdh_sha2_nistp384的常量。

LIBSSH2_KEX_METHOD_ECDH_SHA2_NISTP256表示ECDH(零知识证明)的NISTP256签名方法的哈希函数。

LIBSSH2_KEX_METHOD_ECDH_SHA2_NISTP384表示ECDH(零知识证明)的NISTP384签名方法的哈希函数。

LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY表示是否需要对主机密钥进行请求签名。

Keep in mind, this code is just a definition, it does not include any code to be executed.


```
static const LIBSSH2_KEX_METHOD
kex_method_ecdh_sha2_nistp256 = {
    "ecdh-sha2-nistp256",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD
kex_method_ecdh_sha2_nistp384 = {
    "ecdh-sha2-nistp384",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

static const LIBSSH2_KEX_METHOD
```cpp

这段代码定义了一个名为 `kex_method_ecdh_sha2_nistp521` 的结构体，包含了三个元素：

1. `"ecdh-sha2-nistp521"`：这是一个 Curve255 算法名称。
2. `kex_method_ecdh_key_exchange`：这是ECDH密钥交换算法的标识符。
3. `LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY`：这是标志，表示使用基于NISTP 521规范的ECDH签名。

因此，这个结构体定义了一个名为 `kex_method_ECDH_sha2_nistp521` 的常量，包含了与ECDH签名和NISTP 521规范相关的信息。


```
kex_method_ecdh_sha2_nistp521 = {
    "ecdh-sha2-nistp521",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};
#endif

#if LIBSSH2_ED25519
static const LIBSSH2_KEX_METHOD
kex_method_ssh_curve25519_sha256_libssh = {
    "curve25519-sha256@libssh.org",
    kex_method_curve25519_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};
static const LIBSSH2_KEX_METHOD
```cpp

这段代码定义了一个名为 `kex_method_ssh_curve25519_sha256` 的结构体，包含了三个元素：

1. `"curve25519-sha256"`：表示这是一个 Curve25519 签名算法。
2. `kex_method_curve25519_key_exchange`：表示这是一个用来交换消息的算法。
3. `LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY`：表示一个标志，指示是否需要对主机密钥进行要求。

根据 `libssh2_kex_methods` 数组中的元素，这段代码可能会被用于在 SSH 客户端和服务器之间进行加密和签名，以确保数据的安全性。


```
kex_method_ssh_curve25519_sha256 = {
    "curve25519-sha256",
    kex_method_curve25519_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};
#endif

static const LIBSSH2_KEX_METHOD *libssh2_kex_methods[] = {
#if LIBSSH2_ED25519
    &kex_method_ssh_curve25519_sha256,
    &kex_method_ssh_curve25519_sha256_libssh,
#endif
#if LIBSSH2_ECDSA
    &kex_method_ecdh_sha2_nistp256,
    &kex_method_ecdh_sha2_nistp384,
    &kex_method_ecdh_sha2_nistp521,
```cpp

这段代码定义了一个名为"kex_method_diffie_helman_group_exchange_sha256"的哈希函数，属于 kex 算法库(Keep-Exchange)。这个哈希函数的作用是实现从指定输入字符串到输出字符串的哈希映射，用于在 SSL/TLS 通信中实现数据的加密与解密。

具体来说，这个哈希函数接受一个字符串作为输入，然后输出一个固定长度的哈希值作为输出。这个哈希函数采用 Diffie-Hellman 组交换算法和 SHA-256 算法来实现。其中，Diffie-Hellman 组交换算法是一种安全密码学的算法，用于实现数据的加密与解密，而 SHA-256 算法则是一种哈希算法，用于将任意长度的消息压缩成一个固定长度的哈希值。

通过对输入字符串进行哈希运算，该哈希函数可以保证在不同的输入长度下，输出哈希值是唯一的。这个哈希函数的具体实现主要依赖于 kex 算法库的使用，因此，它可以被用于实现 SSL/TLS 通信中的数据安全。


```
#endif
    &kex_method_diffie_helman_group_exchange_sha256,
    &kex_method_diffie_helman_group16_sha512,
    &kex_method_diffie_helman_group18_sha512,
    &kex_method_diffie_helman_group14_sha256,
    &kex_method_diffie_helman_group14_sha1,
    &kex_method_diffie_helman_group1_sha1,
    &kex_method_diffie_helman_group_exchange_sha1,
  NULL
};

typedef struct _LIBSSH2_COMMON_METHOD
{
    const char *name;
} LIBSSH2_COMMON_METHOD;

```cpp

这段代码定义了一个名为 `kex_method_strlen` 的函数，它用于计算一个特定方法列表 resulting 的字符串的长度。该函数接收两个参数，一个指针变量 `method` 包含两个指向 LIBSSH2_COMMON_METHOD 的指针，另一个是指针变量 `method` 所指向的方法的名称。函数首先检查传入的参数是否为 `NULL`，如果是，则返回 0。否则，函数将遍历方法列表中的每个方法，并计算每个方法名称的字符串长度。最后，函数返回计算得到的方法名称字符串长度减 1，因为在最后一个方法名称后跟了一个空格，空格不占用字符串长度。


```
/* kex_method_strlen
 * Calculate the length of a particular method list's resulting string
 * Includes SUM(strlen() of each individual method plus 1 (for coma)) - 1
 * (because the last coma isn't used)
 * Another sign of bad coding practices gone mad.  Pretend you don't see this.
 */
static size_t
kex_method_strlen(LIBSSH2_COMMON_METHOD ** method)
{
    size_t len = 0;

    if(!method || !*method) {
        return 0;
    }

    while(*method && (*method)->name) {
        len += strlen((*method)->name) + 1;
        method++;
    }

    return len - 1;
}



```cpp

这段代码定义了一个名为“kex_method_list”的函数，它的作用是生成一个格式化好的偏好列表字符串，该列表包含了指定的一组 kex 方法。函数接受两个参数：一个字符指针buf，该指针要存储生成的偏好列表字符串；一个字符数组长度指针list_strlen，该指针指向要存储偏好列表的字符串的长度。函数返回一个字符数组长度，表示生成的偏好列表字符串的长度。

函数的实现包括以下几步：

1. 将buf和list_strlen初始化为0，分别指向一个空字符串和0字符串。

2. 如果buf和list_strlen为0，函数返回4，表示没有可读的kex方法。

3. 如果已经指定了method，函数会遍历method，并复制它的名称到buf中。为了使buf中的字符串居中，在复制完成后，函数会在buf的末尾添加一个'\0'字符。

4. 函数会循环处理method，直到method指向NULL为止。

5. 函数最后返回list_strlen加上生成的偏好列表字符串的长度，即list_strlen + 4。

总之，这段代码定义了一个用于生成格式化偏好列表字符串的函数，通过遍历指定的一组kex方法，将生成的偏好列表字符串存储到buf中，并实现了字符串居中的效果。


```
/* kex_method_list
 * Generate formatted preference list in buf
 */
static size_t
kex_method_list(unsigned char *buf, size_t list_strlen,
                LIBSSH2_COMMON_METHOD ** method)
{
    _libssh2_htonu32(buf, list_strlen);
    buf += 4;

    if(!method || !*method) {
        return 4;
    }

    while(*method && (*method)->name) {
        int mlen = strlen((*method)->name);
        memcpy(buf, (*method)->name, mlen);
        buf += mlen;
        *(buf++) = ',';
        method++;
    }

    return list_strlen + 4;
}



```cpp

这段代码定义了两个宏，一个是`LIBSSH2_METHOD_PREFS_LEN`，另一个是`LIBSSH2_METHOD_PREFS_STR`。

`LIBSSH2_METHOD_PREFS_LEN`定义了一个名为`prefvar`的预定义变量，如果没有定义该变量，则该变量将被赋予其默认值(通常是`LIBSSH2_METHOD_DEFAULT_VALUE`)。然后该宏计算了`prefvar`的预长字符数组大小，最后定义了一个`if`语句，检查`prefvar`是否被定义。如果是，则将`prefvar`的值转换为`NETSSH2_SSH_MSG_MD5_TRIPLEDGE`格式，并将其复制到`buf`中。如果不是，则使用`kex_method_list`函数获取`LIBSSH2_COMMON_METHOD**`类型的指针变量`defaultvar`，并使用该指针计算`LIBSSH2_METHOD_DEFAULT_METHOD_NAME`字符数组。最后，将`buf`和`defaultvar`的值连接在一起，并返回它们的长度。

`LIBSSH2_METHOD_PREFS_STR`定义了一个名为`buf`的输出缓冲区，以及两个名为`prefvarlen`和`defaultvar`的定义。该宏在`LIBSSH2_METHOD_PREFS_LEN`定义的基础上，对`buf`进行了一系列操作，包括复制`prefvar`的值到`buf`，使用`memcpy`将`prefvar`的值复制到`buf`，并使用`kex_method_list`获取`LIBSSH2_COMMON_METHOD**`类型的指针变量`defaultvar`，计算`LIBSSH2_METHOD_DEFAULT_METHOD_NAME`字符数组。然后，该宏将`buf`和`defaultvar`的值连接在一起，并返回它们的长度。最后，该宏将结果赋给`buf`。


```
#define LIBSSH2_METHOD_PREFS_LEN(prefvar, defaultvar)           \
    ((prefvar) ? strlen(prefvar) :                              \
     kex_method_strlen((LIBSSH2_COMMON_METHOD**)(defaultvar)))

#define LIBSSH2_METHOD_PREFS_STR(buf, prefvarlen, prefvar, defaultvar)  \
    if(prefvar) {                                                       \
        _libssh2_htonu32((buf), (prefvarlen));                          \
        buf += 4;                                                       \
        memcpy((buf), (prefvar), (prefvarlen));                         \
        buf += (prefvarlen);                                            \
    }                                                                   \
    else {                                                              \
        buf += kex_method_list((buf), (prefvarlen),                     \
                               (LIBSSH2_COMMON_METHOD**)(defaultvar));  \
    }

```cpp

This is a code snippet that deals with the prefs of a LibSSH2 session.

It starts by defining some constants that are used throughout the code, such as the length of the hostkey, the preferred method for encryption and macros, and the language environment.

Then, it sets the prefs for the hostkey, encryption, and macro methods according to the user preferences.

Finally, it checks whether an optimistic KEX packet followed by a reserved field is present. If it is not found, it sets the session flag for optimistic packets to be dealing with.

The full function is:
```
ssh_session *session->prefs(SESSION, hostkey_len, session->hostkey_prefs,
                                libssh2_hostkey_methods());
```cpp
The return value is the prefs of the session.


```
/* kexinit
 * Send SSH_MSG_KEXINIT packet
 */
static int kexinit(LIBSSH2_SESSION * session)
{
    /* 62 = packet_type(1) + cookie(16) + first_packet_follows(1) +
       reserved(4) + length longs(40) */
    size_t data_len = 62;
    size_t kex_len, hostkey_len = 0;
    size_t crypt_cs_len, crypt_sc_len;
    size_t comp_cs_len, comp_sc_len;
    size_t mac_cs_len, mac_sc_len;
    size_t lang_cs_len, lang_sc_len;
    unsigned char *data, *s;
    int rc;

    if(session->kexinit_state == libssh2_NB_state_idle) {
        kex_len =
            LIBSSH2_METHOD_PREFS_LEN(session->kex_prefs, libssh2_kex_methods);
        hostkey_len =
            LIBSSH2_METHOD_PREFS_LEN(session->hostkey_prefs,
                                     libssh2_hostkey_methods());
        crypt_cs_len =
            LIBSSH2_METHOD_PREFS_LEN(session->local.crypt_prefs,
                                     libssh2_crypt_methods());
        crypt_sc_len =
            LIBSSH2_METHOD_PREFS_LEN(session->remote.crypt_prefs,
                                     libssh2_crypt_methods());
        mac_cs_len =
            LIBSSH2_METHOD_PREFS_LEN(session->local.mac_prefs,
                                     _libssh2_mac_methods());
        mac_sc_len =
            LIBSSH2_METHOD_PREFS_LEN(session->remote.mac_prefs,
                                     _libssh2_mac_methods());
        comp_cs_len =
            LIBSSH2_METHOD_PREFS_LEN(session->local.comp_prefs,
                                     _libssh2_comp_methods(session));
        comp_sc_len =
            LIBSSH2_METHOD_PREFS_LEN(session->remote.comp_prefs,
                                     _libssh2_comp_methods(session));
        lang_cs_len =
            LIBSSH2_METHOD_PREFS_LEN(session->local.lang_prefs, NULL);
        lang_sc_len =
            LIBSSH2_METHOD_PREFS_LEN(session->remote.lang_prefs, NULL);

        data_len += kex_len + hostkey_len + crypt_cs_len + crypt_sc_len +
            comp_cs_len + comp_sc_len + mac_cs_len + mac_sc_len +
            lang_cs_len + lang_sc_len;

        s = data = LIBSSH2_ALLOC(session, data_len);
        if(!data) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory");
        }

        *(s++) = SSH_MSG_KEXINIT;

        if(_libssh2_random(s, 16)) {
            return _libssh2_error(session, LIBSSH2_ERROR_RANDGEN,
                                  "Unable to get random bytes "
                                  "for KEXINIT cookie");
        }
        s += 16;

        /* Ennumerating through these lists twice is probably (certainly?)
           inefficient from a CPU standpoint, but it saves multiple
           malloc/realloc calls */
        LIBSSH2_METHOD_PREFS_STR(s, kex_len, session->kex_prefs,
                                 libssh2_kex_methods);
        LIBSSH2_METHOD_PREFS_STR(s, hostkey_len, session->hostkey_prefs,
                                 libssh2_hostkey_methods());
        LIBSSH2_METHOD_PREFS_STR(s, crypt_cs_len, session->local.crypt_prefs,
                                 libssh2_crypt_methods());
        LIBSSH2_METHOD_PREFS_STR(s, crypt_sc_len, session->remote.crypt_prefs,
                                 libssh2_crypt_methods());
        LIBSSH2_METHOD_PREFS_STR(s, mac_cs_len, session->local.mac_prefs,
                                 _libssh2_mac_methods());
        LIBSSH2_METHOD_PREFS_STR(s, mac_sc_len, session->remote.mac_prefs,
                                 _libssh2_mac_methods());
        LIBSSH2_METHOD_PREFS_STR(s, comp_cs_len, session->local.comp_prefs,
                                 _libssh2_comp_methods(session));
        LIBSSH2_METHOD_PREFS_STR(s, comp_sc_len, session->remote.comp_prefs,
                                 _libssh2_comp_methods(session));
        LIBSSH2_METHOD_PREFS_STR(s, lang_cs_len, session->local.lang_prefs,
                                 NULL);
        LIBSSH2_METHOD_PREFS_STR(s, lang_sc_len, session->remote.lang_prefs,
                                 NULL);

        /* No optimistic KEX packet follows */
        /* Deal with optimistic packets
         * session->flags |= KEXINIT_OPTIMISTIC
         * session->flags |= KEXINIT_METHODSMATCH
         */
        *(s++) = 0;

        /* Reserved == 0 */
        _libssh2_htonu32(s, 0);

```cpp

This is a C-style function that takes a number of arguments and returns a pointer to a character pointer that points to a struct containing information about a key. The struct contains the key type, the key size, the raw bytes of the key, and the associated metadata.

The function first extracts the key type, key size, and raw bytes of the key from the input data. It then extracts the associated metadata from the input data and stores the information in the struct. Finally, it returns a pointer to the struct character pointer, which can be used to access the information in the struct.


```
#ifdef LIBSSH2DEBUG
        {
            /* Funnily enough, they'll all "appear" to be '\0' terminated */
            unsigned char *p = data + 21;   /* type(1) + cookie(16) + len(4) */

            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent KEX: %s", p);
            p += kex_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent HOSTKEY: %s", p);
            p += hostkey_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent CRYPT_CS: %s", p);
            p += crypt_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent CRYPT_SC: %s", p);
            p += crypt_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent MAC_CS: %s", p);
            p += mac_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent MAC_SC: %s", p);
            p += mac_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent COMP_CS: %s", p);
            p += comp_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent COMP_SC: %s", p);
            p += comp_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent LANG_CS: %s", p);
            p += lang_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent LANG_SC: %s", p);
            p += lang_sc_len + 4;
        }
```cpp

这段代码是一个 C 语言函数，属于 SSH2 协议的库函数，负责在客户端和服务器之间传递 KEXINIT 包。

代码的作用是：

1. 如果客户端已经连接好了服务器，那么就设置 session 的 kexinit_state 为 libssh2_NB_state_created，以便后续使用。

2. 如果客户端还没有连接服务器，那么就设置 session 的 kexinit_data 为 NULL，kexinit_data_len 为 0，以便后续使用。

3. 调用 _libssh2_transport_send 函数向服务器发送 KEXINIT 包，并将服务器返回的结果记录下来。

4. 如果返回的rc是 LIBSSH2_ERROR_EAGAIN，那么说明服务器端没有响应，需要重新发送，并返回 LIBSSH2_ERROR_INVALID_KEY。

5. 如果返回的rc 是 LIBSSH2_ERROR_UNEXPECTED，那么说明客户端没有收到服务器端的响应，需要重新发送，并返回 LIBSSH2_ERROR_UNEXPECTED。

6. 如果客户端 local.kexinit 为 NULL，那么说明客户端和服务器之间的数据已经准备好，可以继续使用。

7. 如果客户端 local.kexinit 为数据，那么说明客户端和服务器之间的数据还没有准备好，需要重新发送，并返回 LIBSSH2_ERROR_UNEXPECTED。

8. 如果客户端 local.kexinit_len 为数据长度，那么说明客户端和服务器之间的数据还没有准备好，需要重新发送，并返回 LIBSSH2_ERROR_UNEXPECTED。

9. 设置 session 的 kexinit_state 为 libssh2_NB_state_idle，以便后续使用。


```
#endif /* LIBSSH2DEBUG */

        session->kexinit_state = libssh2_NB_state_created;
    }
    else {
        data = session->kexinit_data;
        data_len = session->kexinit_data_len;
        /* zap the variables to ensure there is NOT a double free later */
        session->kexinit_data = NULL;
        session->kexinit_data_len = 0;
    }

    rc = _libssh2_transport_send(session, data, data_len, NULL, 0);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        session->kexinit_data = data;
        session->kexinit_data_len = data_len;
        return rc;
    }
    else if(rc) {
        LIBSSH2_FREE(session, data);
        session->kexinit_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Unable to send KEXINIT packet to remote host");

    }

    if(session->local.kexinit) {
        LIBSSH2_FREE(session, session->local.kexinit);
    }

    session->local.kexinit = data;
    session->local.kexinit_len = data_len;

    session->kexinit_state = libssh2_NB_state_idle;

    return 0;
}

```cpp

This is a function definition that appears to be a kex-specific variant of the `strstr()` function. It takes two arguments: a `haystack` pointer and a `needle`, and returns a pointer to a unique location in the `haystack` where the `needle` was found in the original string, or `NULL` if the `needle` is too long or the `haystack` is too short.

The function has several helper functions to simplify the search:

* `memchr()` function to check for a character `c` in the `needle`, which is before the first occurrence of `c` in the `needle`. The function returns a pointer to the first occurrence of `c` in the `needle`, or `NULL` if the `needle` does not contain `c` before the first occurrence.
* `strncmp()` function to compare the substring of the `haystack` and the `needle` against each other. This function takes two arguments: the starting index in the `needle` and the length of the `needle`. It returns a non-zero value if the substring is equal, and zero otherwise.
* `kex_agree_instr()` function自身， which is the main function that performs the actual search. This function takes two arguments: the `haystack` and the `needle`, and returns a pointer to the location in the `haystack` where the `needle` was found. It works by first checking if the `needle` is too short or the `haystack` is too small, and then searching for the `needle` in the `haystack` using several helper functions.

The `kex_agree_instr()` function first checks if the `needle` is too short or the `haystack` is too small, and then performs a linear search in the `haystack` using several helper functions. If the `needle` is found, the function returns the pointer to the location in the `haystack` where the `needle` was found. If the `needle` is too long or the `haystack` is too small, the function returns `NULL`.


```
/* kex_agree_instr
 * Kex specific variant of strstr()
 * Needle must be precede by BOL or ',', and followed by ',' or EOL
 */
static unsigned char *
kex_agree_instr(unsigned char *haystack, unsigned long haystack_len,
                const unsigned char *needle, unsigned long needle_len)
{
    unsigned char *s;
    unsigned char *end_haystack;
    unsigned long left;

    if(haystack == NULL || needle == NULL) {
        return NULL;
    }

    /* Haystack too short to bother trying */
    if(haystack_len < needle_len || needle_len == 0) {
        return NULL;
    }

    s = haystack;
    end_haystack = &haystack[haystack_len];
    left = end_haystack - s;

    /* Needle at start of haystack */
    if((strncmp((char *) haystack, (char *) needle, needle_len) == 0) &&
        (needle_len == haystack_len || haystack[needle_len] == ',')) {
        return haystack;
    }

    /* Search until we run out of comas or we run out of haystack,
       whichever comes first */
    while((s = (unsigned char *) memchr((char *) s, ',', left))) {
        /* Advance buffer past coma if we can */
        left = end_haystack - s;
        if((left >= 1) && (left <= haystack_len) && (left > needle_len)) {
            s++;
        }
        else {
            return NULL;
        }

        /* Needle at X position */
        if((strncmp((char *) s, (char *) needle, needle_len) == 0) &&
            (((s - haystack) + needle_len) == haystack_len
             || s[needle_len] == ',')) {
            return s;
        }
    }

    return NULL;
}



```cpp

这段代码定义了一个名为 `kex_get_method_by_name` 的函数，它接受一个字符串参数 `name`，一个表示方法列表的指针 `methodlist`，以及一个指向 LIBSSH2_COMMON_METHOD 类型的变量 *指针 `method_list`。

函数首先定义了一个名为 `name_len` 的静态变量，用于存储 `name` 的长度。然后，函数内部使用一个 while 循环，该循环会在方法列表中查找与 `name` 字符串相等的第一个方法。如果找到了该方法，函数将返回该方法在方法列表中的位置，否则继续遍历方法列表。

在 while 循环内部，函数还使用一个 strncmp 函数来比较 `name` 和 `method_list->name`。如果两个字符串相等，函数将返回方法列表中与 `name` 相等的位置；否则，继续遍历方法列表。

最后，函数返回一个指向 LIBSSH2_COMMON_METHOD 的指针，该指针将指向方法列表中的第一个方法。如果函数无法找到与 `name` 相等的方法，函数将返回 NULL。


```
/* kex_get_method_by_name
 */
static const LIBSSH2_COMMON_METHOD *
kex_get_method_by_name(const char *name, size_t name_len,
                       const LIBSSH2_COMMON_METHOD ** methodlist)
{
    while(*methodlist) {
        if((strlen((*methodlist)->name) == name_len) &&
            (strncmp((*methodlist)->name, name, name_len) == 0)) {
            return *methodlist;
        }
        methodlist++;
    }
    return NULL;
}



```cpp

This is a function that appears to check if a public key is valid for use with an SSL/TLS handshake. It takes a `public_key` parameter, which is a raw pointer to a public key represented in data that is used for encryption and signing.

The function first checks if the raw pointer is null, which would indicate that the public key cannot be used. If the pointer is not null, the function then checks if the public key is a valid SHA-256 host key. If the key is valid, the function checks if it is possible to use this host key for encryption or if it is only possible for the specified method (e.g. `LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY`)

If the key is valid for both encryption and signing, the function then checks if the key is a valid PEM encoded host key. If the key is valid, the function returns `0` to indicate that the public key is valid.

If the key is not valid for either encryption or signing, or if the key is not a valid PEM encoded host key, the function returns `-1` to indicate that the public key is invalid.

The function also appears to take into account the specific method that is required for use with the `LIBSSH2_KEX_METHOD_FLAG_REQ_*` constants. If the `method_flags` parameter is `LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY`, for example, the function will only check if the host key is valid for signing, not for encryption.

I hope this helps! Let me know if you have any further questions.


```
/* kex_agree_hostkey
 * Agree on a Hostkey which works with this kex
 */
static int kex_agree_hostkey(LIBSSH2_SESSION * session,
                             unsigned long kex_flags,
                             unsigned char *hostkey, unsigned long hostkey_len)
{
    const LIBSSH2_HOSTKEY_METHOD **hostkeyp = libssh2_hostkey_methods();
    unsigned char *s;

    if(session->hostkey_prefs) {
        s = (unsigned char *) session->hostkey_prefs;

        while(s && *s) {
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));
            if(kex_agree_instr(hostkey, hostkey_len, s, method_len)) {
                const LIBSSH2_HOSTKEY_METHOD *method =
                    (const LIBSSH2_HOSTKEY_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           hostkeyp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                /* So far so good, but does it suit our purposes? (Encrypting
                   vs Signing) */
                if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY) ==
                     0) || (method->encrypt)) {
                    /* Either this hostkey can do encryption or this kex just
                       doesn't require it */
                    if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY)
                         == 0) || (method->sig_verify)) {
                        /* Either this hostkey can do signing or this kex just
                           doesn't require it */
                        session->hostkey = method;
                        return 0;
                    }
                }
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    while(hostkeyp && (*hostkeyp) && (*hostkeyp)->name) {
        s = kex_agree_instr(hostkey, hostkey_len,
                            (unsigned char *) (*hostkeyp)->name,
                            strlen((*hostkeyp)->name));
        if(s) {
            /* So far so good, but does it suit our purposes? (Encrypting vs
               Signing) */
            if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY) == 0) ||
                ((*hostkeyp)->encrypt)) {
                /* Either this hostkey can do encryption or this kex just
                   doesn't require it */
                if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY) ==
                     0) || ((*hostkeyp)->sig_verify)) {
                    /* Either this hostkey can do signing or this kex just
                       doesn't require it */
                    session->hostkey = *hostkeyp;
                    return 0;
                }
            }
        }
        hostkeyp++;
    }

    return -1;
}



```cpp

This is a function that performs key exchange with a server, using thekees algorithm. The function takes a single parameter, a struct that contains the server's certificate data, and an optional parameter that specifies the type of key exchange to perform (KEX_INIT, KEX, KEX_EXchange, ...).

The function first checks if the server has a certificate data, and if it does, it performs the key exchange by typing the server's certificate data to the client's server key. If the client is unable to agree with the server's certificate data, it returns an error. If the client is able to agree with the server's certificate data, it performs the key exchange by typing the client's server key to the server's certificate data.

If the client is unable to agree with the server's certificate data, it performs the key exchange by typing the client's server key to the server's certificate data.

The function returns 0 if the key exchange is successful, or a negative value if it fails. If the function returns 0, it also sets the `session` variable to the newly generated server key, and if it is unable to burn the key it will return.


```
/* kex_agree_kex_hostkey
 * Agree on a Key Exchange method and a hostkey encoding type
 */
static int kex_agree_kex_hostkey(LIBSSH2_SESSION * session, unsigned char *kex,
                                 unsigned long kex_len, unsigned char *hostkey,
                                 unsigned long hostkey_len)
{
    const LIBSSH2_KEX_METHOD **kexp = libssh2_kex_methods;
    unsigned char *s;

    if(session->kex_prefs) {
        s = (unsigned char *) session->kex_prefs;

        while(s && *s) {
            unsigned char *q, *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));
            q = kex_agree_instr(kex, kex_len, s, method_len);
            if(q) {
                const LIBSSH2_KEX_METHOD *method = (const LIBSSH2_KEX_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           kexp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                /* We've agreed on a key exchange method,
                 * Can we agree on a hostkey that works with this kex?
                 */
                if(kex_agree_hostkey(session, method->flags, hostkey,
                                      hostkey_len) == 0) {
                    session->kex = method;
                    if(session->burn_optimistic_kexinit && (kex == q)) {
                        /* Server sent an optimistic packet, and client agrees
                         * with preference cancel burning the first KEX_INIT
                         * packet that comes in */
                        session->burn_optimistic_kexinit = 0;
                    }
                    return 0;
                }
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    while(*kexp && (*kexp)->name) {
        s = kex_agree_instr(kex, kex_len,
                            (unsigned char *) (*kexp)->name,
                            strlen((*kexp)->name));
        if(s) {
            /* We've agreed on a key exchange method,
             * Can we agree on a hostkey that works with this kex?
             */
            if(kex_agree_hostkey(session, (*kexp)->flags, hostkey,
                                  hostkey_len) == 0) {
                session->kex = *kexp;
                if(session->burn_optimistic_kexinit && (kex == s)) {
                    /* Server sent an optimistic packet, and client agrees
                     * with preference cancel burning the first KEX_INIT
                     * packet that comes in */
                    session->burn_optimistic_kexinit = 0;
                }
                return 0;
            }
        }
        kexp++;
    }
    return -1;
}



```cpp

This is a function definition for "kex\_agree\_crypt" which is used to determine whether the client and server agree on a certain cryptography method. It takes three arguments: a pointer to a LIBSSH2\_SESSION object, a pointer to a LIBSSH2\_ENDPOTRIGHT\_DATA object, and a pointer to a pointer to a byte array representing the client's certificate, as well as an additional pointer to a pointer to a LIBSSH2\_COMMON\_METHOD\* array representing the chosen cryptography method. The function returns 0 if the client and server agree on the chosen method, or -1 if they don't agree. If the function fails to reach an agreement, it will return -1.

The function first checks if the certificate provided by the client is valid and has a valid signature. Then it checks if the client has any pre-defined cryptography preferences and if it does, it uses that to determine the chosen cryptography method. If the client doesn't have any pre-defined preferences, the function uses the default method to determine the chosen cryptography method.

Then the function compares the chosen cryptography method to the one that the server expects. If they match, the function sets the server's certificate to the chosen method and returns 0. If they don't match, the function returns -1.


```
/* kex_agree_crypt
 * Agree on a cipher algo
 */
static int kex_agree_crypt(LIBSSH2_SESSION * session,
                           libssh2_endpoint_data *endpoint,
                           unsigned char *crypt,
                           unsigned long crypt_len)
{
    const LIBSSH2_CRYPT_METHOD **cryptp = libssh2_crypt_methods();
    unsigned char *s;

    (void) session;

    if(endpoint->crypt_prefs) {
        s = (unsigned char *) endpoint->crypt_prefs;

        while(s && *s) {
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            if(kex_agree_instr(crypt, crypt_len, s, method_len)) {
                const LIBSSH2_CRYPT_METHOD *method =
                    (const LIBSSH2_CRYPT_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           cryptp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                endpoint->crypt = method;
                return 0;
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    while(*cryptp && (*cryptp)->name) {
        s = kex_agree_instr(crypt, crypt_len,
                            (unsigned char *) (*cryptp)->name,
                            strlen((*cryptp)->name));
        if(s) {
            endpoint->crypt = *cryptp;
            return 0;
        }
        cryptp++;
    }

    return -1;
}



```cpp

这段代码定义了一个名为 `kex_agree_mac` 的函数，属于 `libssh2_ssh` 库。它的作用是确认客户端提供的消息认证哈希是否正确。

具体来说，该函数接收两个参数：`session` 和 `endpoint`，分别表示 SSH 会话句柄和远程主机数据。函数内部首先定义了一个名为 `mac` 的变量，用于存储消息认证哈希，以及一个名为 `mac_len` 的整数，用于存储哈希的长度。

接着，函数内部定义了一系列来自 `libssh2_mac_methods` 数组和名为 `kex_agree_instr` 的函数的指针，这些函数用于在客户端和服务器之间交换消息认证哈希。

函数的主要逻辑如下：

1. 首先检查远程主机是否支持消息认证哈希，如果支持，则执行以下操作：

a. 将 `mac` 和 `mac_len` 存储为远程主机已有的消息认证哈希。

b. 调用 `kex_agree_instr` 函数，使用远程主机已有的消息认证哈希、`mac` 和 `mac_len` 来计算客户端提供的消息认证哈希。

c. 如果计算出的消息认证哈希与远程主机已有的哈希不相等，则说明客户端提供的哈希不正确，函数返回 `-1`，表示失败。

2. 如果远程主机不支持消息认证哈希，则执行以下操作：

a. 遍历 `libssh2_mac_methods` 数组，查找名为 `kex_agree_instr` 的函数。

b. 如果找到了这个函数，则执行以下操作：

i. 将 `mac` 和 `mac_len` 存储为远程主机已有的消息认证哈希。

ii. 调用 `kex_agree_instr` 函数，使用远程主机已有的消息认证哈希、`mac` 和 `mac_len` 来计算客户端提供的消息认证哈希。

b. 如果调用 `kex_agree_instr` 函数后计算出的消息认证哈希与远程主机已有的哈希不相等，则说明客户端提供的哈希不正确，函数返回 `-1`，表示失败。

3. 如果函数在执行过程中没有遇到错误，则返回 `0`，表示成功。


```
/* kex_agree_mac
 * Agree on a message authentication hash
 */
static int kex_agree_mac(LIBSSH2_SESSION * session,
                         libssh2_endpoint_data * endpoint, unsigned char *mac,
                         unsigned long mac_len)
{
    const LIBSSH2_MAC_METHOD **macp = _libssh2_mac_methods();
    unsigned char *s;
    (void) session;

    if(endpoint->mac_prefs) {
        s = (unsigned char *) endpoint->mac_prefs;

        while(s && *s) {
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            if(kex_agree_instr(mac, mac_len, s, method_len)) {
                const LIBSSH2_MAC_METHOD *method = (const LIBSSH2_MAC_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           macp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                endpoint->mac = method;
                return 0;
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    while(*macp && (*macp)->name) {
        s = kex_agree_instr(mac, mac_len, (unsigned char *) (*macp)->name,
                            strlen((*macp)->name));
        if(s) {
            endpoint->mac = *macp;
            return 0;
        }
        macp++;
    }

    return -1;
}



```cpp

这段代码是一个名为 `kex_agree_comp` 的函数，它是用来在客户端和服务器之间协商压缩协议的。

具体来说，这个函数接受一个 `LIBSSH2_SESSION` 类型的会话，一个包含 `libssh2_endpoint_data` 结构的 `endpoint`，以及一个 `unsigned char` 类型的 `comp` 和一个 `unsigned long` 类型的 `comp_len`。函数的作用是在 `comp` 和 `comp_len` 之间选择一个压缩协议，并返回一个 indicating 是否成功的标头。

具体实现中，函数首先定义了一个名为 `kex_agree_instr` 的函数，用来在客户端计算校验和，函数接受 `comp` 和 `comp_len`，返回一个指向计算结果的正确整数的指针。接着定义了一个名为 `kex_agree_comp` 的函数，这个函数首先定义了一个名为 `kex_agree_instr` 的函数，然后在 `libssh2_comp_methods` 函数的数组中查找 `endpoint` 所属的 `LIBSSH2_COMP_METHOD`，然后使用这个方法计算校验和，最后将计算结果存储到 `comp` 中，并返回一个 indicating 是否成功的标头。


```
/* kex_agree_comp
 * Agree on a compression scheme
 */
static int kex_agree_comp(LIBSSH2_SESSION *session,
                          libssh2_endpoint_data *endpoint, unsigned char *comp,
                          unsigned long comp_len)
{
    const LIBSSH2_COMP_METHOD **compp = _libssh2_comp_methods(session);
    unsigned char *s;
    (void) session;

    if(endpoint->comp_prefs) {
        s = (unsigned char *) endpoint->comp_prefs;

        while(s && *s) {
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            if(kex_agree_instr(comp, comp_len, s, method_len)) {
                const LIBSSH2_COMP_METHOD *method =
                    (const LIBSSH2_COMP_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           compp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                endpoint->comp = method;
                return 0;
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    while(*compp && (*compp)->name) {
        s = kex_agree_instr(comp, comp_len, (unsigned char *) (*compp)->name,
                            strlen((*compp)->name));
        if(s) {
            endpoint->comp = *compp;
            return 0;
        }
        compp++;
    }

    return -1;
}


```cpp

This function appears to check if a server has sent an optimistic packet or not. It does this by first getting the server's MAC address and a corresponding checksum, and then it tries to guess the server's password using the Comp exhaustive attack.

The function returns -1 if the server发送了一个 humorous packet, or if it is unable to guess the server's password.

It is important to note that the function assumes that the server has sent the password using the Comp exhaustive attack. This attack is not considered a strong attack and should not be used as a security mechanism.


```
/* TODO: When in server mode we need to turn this logic on its head
 * The Client gets to make the final call on "agreed methods"
 */

/* kex_agree_methods
 * Decide which specific method to use of the methods offered by each party
 */
static int kex_agree_methods(LIBSSH2_SESSION * session, unsigned char *data,
                             unsigned data_len)
{
    unsigned char *kex, *hostkey, *crypt_cs, *crypt_sc, *comp_cs, *comp_sc,
        *mac_cs, *mac_sc;
    size_t kex_len, hostkey_len, crypt_cs_len, crypt_sc_len, comp_cs_len;
    size_t comp_sc_len, mac_cs_len, mac_sc_len;
    struct string_buf buf;

    if(data_len < 17)
        return -1;

    buf.data = (unsigned char *)data;
    buf.len = data_len;
    buf.dataptr = buf.data;
    buf.dataptr++; /* advance past packet type */

    /* Skip cookie, don't worry, it's preserved in the kexinit field */
    buf.dataptr += 16;

    /* Locate each string */
    if(_libssh2_get_string(&buf, &kex, &kex_len))
        return -1;
    if(_libssh2_get_string(&buf, &hostkey, &hostkey_len))
        return -1;
    if(_libssh2_get_string(&buf, &crypt_cs, &crypt_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &crypt_sc, &crypt_sc_len))
        return -1;
    if(_libssh2_get_string(&buf, &mac_cs, &mac_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &mac_sc, &mac_sc_len))
        return -1;
    if(_libssh2_get_string(&buf, &comp_cs, &comp_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &comp_sc, &comp_sc_len))
        return -1;

    /* If the server sent an optimistic packet, assume that it guessed wrong.
     * If the guess is determined to be right (by kex_agree_kex_hostkey)
     * This flag will be reset to zero so that it's not ignored */
    if(_libssh2_check_length(&buf, 1)) {
        session->burn_optimistic_kexinit = *(buf.dataptr++);
    }
    else {
        return -1;
    }

    /* Next uint32 in packet is all zeros (reserved) */

    if(kex_agree_kex_hostkey(session, kex, kex_len, hostkey, hostkey_len)) {
        return -1;
    }

    if(kex_agree_crypt(session, &session->local, crypt_cs, crypt_cs_len)
       || kex_agree_crypt(session, &session->remote, crypt_sc,
                          crypt_sc_len)) {
        return -1;
    }

    if(kex_agree_mac(session, &session->local, mac_cs, mac_cs_len) ||
        kex_agree_mac(session, &session->remote, mac_sc, mac_sc_len)) {
        return -1;
    }

    if(kex_agree_comp(session, &session->local, comp_cs, comp_cs_len) ||
        kex_agree_comp(session, &session->remote, comp_sc, comp_sc_len)) {
        return -1;
    }

```cpp

这段代码是一个用于在SSH客户端和服务器之间进行KEX验证的库函数。主要作用是判断客户端和服务器是否支持多种KEX方法（KEX算法），如果客户端和服务器都不支持，则返回-1。如果客户端支持所有KEX方法，则执行后续操作，包括：将客户端的KEX名称存储到`session->local.kex->name`，将客户端的HOSTKEY名称存储到`session->local.hhostkey->name`，将客户端的CRYPT_CS名称存储到`session->local.crypt->name`，将客户端的CRYPT_SC名称存储到`session->local.scrypt->name`，将客户端的MAC_CS名称存储到`session->local.mac->name`，将客户端的MAC_SC名称存储到`session->local.scmac->name`，将客户端的COMP_CS名称存储到`session->local.comp->name`，将客户端的COMP_SC名称存储到`session->local.scpemac->name`。然后将`libssh2_debug`函数打印一些日志信息，最后返回0。


```
#if 0
    if(libssh2_kex_agree_lang(session, &session->local, lang_cs, lang_cs_len)
        || libssh2_kex_agree_lang(session, &session->remote, lang_sc,
                                  lang_sc_len)) {
        return -1;
    }
#endif

    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on KEX method: %s",
                   session->kex->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on HOSTKEY method: %s",
                   session->hostkey->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on CRYPT_CS method: %s",
                   session->local.crypt->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on CRYPT_SC method: %s",
                   session->remote.crypt->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on MAC_CS method: %s",
                   session->local.mac->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on MAC_SC method: %s",
                   session->remote.mac->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on COMP_CS method: %s",
                   session->local.comp->name);
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on COMP_SC method: %s",
                   session->remote.comp->name);

    return 0;
}



```cpp

This function appears to handle the key initialization process for an SSH session. It does this by first initializing the SSH key, and then waiting for the key to be fully initialized before proceeding with the rest of the initialization process.

It is important to note that this function assumes that the initialization process has already been completed, and that the key has already been read from the server.

The function has several input parameters:

- `session`: The SSH session object.
- `key_state`: The key initialization object.
- `key_data`: The key data buffer.
- `key_state_low`: The lower 7 bits of the key data.
- `exchange_keys`: A flag indicating whether to exchange keys.

It returns one of the following values:

- `LIBSSH2_ERROR_KEY_EXCHANGE_FAILURE`: If the key exchange failed.
- `LIBSSH2_ERROR_KEX_FAILURE`: If the initialization of the key failed.
- `LIBSSH2_NEXT_STATUS`: If the initialization process is complete and the key has been fully initialized.

If the key initialization was successful, the function returns `0`.


```
/* _libssh2_kex_exchange
 * Exchange keys
 * Returns 0 on success, non-zero on failure
 *
 * Returns some errors without _libssh2_error()
 */
int
_libssh2_kex_exchange(LIBSSH2_SESSION * session, int reexchange,
                      key_exchange_state_t * key_state)
{
    int rc = 0;
    int retcode;

    session->state |= LIBSSH2_STATE_KEX_ACTIVE;

    if(key_state->state == libssh2_NB_state_idle) {
        /* Prevent loop in packet_add() */
        session->state |= LIBSSH2_STATE_EXCHANGING_KEYS;

        if(reexchange) {
            session->kex = NULL;

            if(session->hostkey && session->hostkey->dtor) {
                session->hostkey->dtor(session,
                                       &session->server_hostkey_abstract);
            }
            session->hostkey = NULL;
        }

        key_state->state = libssh2_NB_state_created;
    }

    if(!session->kex || !session->hostkey) {
        if(key_state->state == libssh2_NB_state_created) {
            /* Preserve in case of failure */
            key_state->oldlocal = session->local.kexinit;
            key_state->oldlocal_len = session->local.kexinit_len;

            session->local.kexinit = NULL;

            key_state->state = libssh2_NB_state_sent;
        }

        if(key_state->state == libssh2_NB_state_sent) {
            retcode = kexinit(session);
            if(retcode == LIBSSH2_ERROR_EAGAIN) {
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                return retcode;
            }
            else if(retcode) {
                session->local.kexinit = key_state->oldlocal;
                session->local.kexinit_len = key_state->oldlocal_len;
                key_state->state = libssh2_NB_state_idle;
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;
                return -1;
            }

            key_state->state = libssh2_NB_state_sent1;
        }

        if(key_state->state == libssh2_NB_state_sent1) {
            retcode =
                _libssh2_packet_require(session, SSH_MSG_KEXINIT,
                                        &key_state->data,
                                        &key_state->data_len, 0, NULL, 0,
                                        &key_state->req_state);
            if(retcode == LIBSSH2_ERROR_EAGAIN) {
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                return retcode;
            }
            else if(retcode) {
                if(session->local.kexinit) {
                    LIBSSH2_FREE(session, session->local.kexinit);
                }
                session->local.kexinit = key_state->oldlocal;
                session->local.kexinit_len = key_state->oldlocal_len;
                key_state->state = libssh2_NB_state_idle;
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;
                return -1;
            }

            if(session->remote.kexinit) {
                LIBSSH2_FREE(session, session->remote.kexinit);
            }
            session->remote.kexinit = key_state->data;
            session->remote.kexinit_len = key_state->data_len;

            if(kex_agree_methods(session, key_state->data,
                                  key_state->data_len))
                rc = LIBSSH2_ERROR_KEX_FAILURE;

            key_state->state = libssh2_NB_state_sent2;
        }
    }
    else {
        key_state->state = libssh2_NB_state_sent2;
    }

    if(rc == 0 && session->kex) {
        if(key_state->state == libssh2_NB_state_sent2) {
            retcode = session->kex->exchange_keys(session,
                                                  &key_state->key_state_low);
            if(retcode == LIBSSH2_ERROR_EAGAIN) {
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                return retcode;
            }
            else if(retcode) {
                rc = _libssh2_error(session,
                                    LIBSSH2_ERROR_KEY_EXCHANGE_FAILURE,
                                    "Unrecoverable error exchanging keys");
            }
        }
    }

    /* Done with kexinit buffers */
    if(session->local.kexinit) {
        LIBSSH2_FREE(session, session->local.kexinit);
        session->local.kexinit = NULL;
    }
    if(session->remote.kexinit) {
        LIBSSH2_FREE(session, session->remote.kexinit);
        session->remote.kexinit = NULL;
    }

    session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
    session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;

    key_state->state = libssh2_NB_state_idle;

    return rc;
}



```cpp

This function appears to be used to set the preferences for a particular SSH method. It takes a single parameter, `method_type`, which is either `SCP` or `EXP`, and returns an error code on success or failure.

The function first checks the input parameter and, if it is `EXP`, it checks that the method is supported by the client. If the method is supported, it creates a new preferences array and loops through any method profiles it finds. If the method is `SCP`, it loops through the `SCP-Preferences` field and stores the corresponding `SCP` field in the `prefs_len` buffer.

If the input `method_type` is `SCP`, the function first loops through the `SCP-Preferences` field and stores the corresponding `SCP` field in the `prefs_len` buffer. It then loops through the `SCP` field and stores the corresponding `SCP` field in the `prefs_len` buffer if it is not `EXP`. If the `SCP` field is `EXP`, the function creates a new `prefs_len` buffer and sets its `SCP` field to `method_type` and sets the `prefs` field to `method_type` if it is `EXP`. It then loops through any `SCP-Preferences` field and stores the corresponding `SCP` field in the `prefs_len` buffer.

If the function is unable to find any `SCP` fields or the `SCP-Preferences` field, it sets the `LIBSSH2_ERROR_METHOD_NOT_SUPPORTED` error code and returns it. If the function is able to set the preferences, it returns `0`.


```
/* libssh2_session_method_pref
 * Set preferred method
 */
LIBSSH2_API int
libssh2_session_method_pref(LIBSSH2_SESSION * session, int method_type,
                            const char *prefs)
{
    char **prefvar, *s, *newprefs;
    int prefs_len = strlen(prefs);
    const LIBSSH2_COMMON_METHOD **mlist;

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        prefvar = &session->kex_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_kex_methods;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        prefvar = &session->hostkey_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_hostkey_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
        prefvar = &session->local.crypt_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_SC:
        prefvar = &session->remote.crypt_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_MAC_CS:
        prefvar = &session->local.mac_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_MAC_SC:
        prefvar = &session->remote.mac_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_COMP_CS:
        prefvar = &session->local.comp_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    case LIBSSH2_METHOD_COMP_SC:
        prefvar = &session->remote.comp_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    case LIBSSH2_METHOD_LANG_CS:
        prefvar = &session->local.lang_prefs;
        mlist = NULL;
        break;

    case LIBSSH2_METHOD_LANG_SC:
        prefvar = &session->remote.lang_prefs;
        mlist = NULL;
        break;

    default:
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "Invalid parameter specified for method_type");
    }

    s = newprefs = LIBSSH2_ALLOC(session, prefs_len + 1);
    if(!newprefs) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Error allocated space for method preferences");
    }
    memcpy(s, prefs, prefs_len + 1);

    while(s && *s && mlist) {
        char *p = strchr(s, ',');
        int method_len = p ? (p - s) : (int) strlen(s);

        if(!kex_get_method_by_name(s, method_len, mlist)) {
            /* Strip out unsupported method */
            if(p) {
                memcpy(s, p + 1, strlen(s) - method_len);
            }
            else {
                if(s > newprefs) {
                    *(--s) = '\0';
                }
                else {
                    *s = '\0';
                }
            }
        }
        else {
            s = p ? (p + 1) : NULL;
        }
    }

    if(!*newprefs) {
        LIBSSH2_FREE(session, newprefs);
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "The requested method(s) are not currently "
                              "supported");
    }

    if(*prefvar) {
        LIBSSH2_FREE(session, *prefvar);
    }
    *prefvar = newprefs;

    return 0;
}

```cpp

The function `libssh2_parse_agent_algorithms` takes a list of algorithm names and returns the number of supported algorithms. If the number of supported algorithms is less than the number of algorithms provided, the function returns `LIBSSH2_ERROR_INVAL`.

The function first counts the number of algorithm names in the list by iterating through the list and getting the name of each algorithm. It then checks if the list is not empty and returns the number of algorithms if it is. If the list is empty, the function returns `LIBSSH2_ERROR_INVAL`, indicating that no algorithm was found.

If the list is not empty, the function allocates a buffer of the same size as the number of algorithm names in the list, and then iterates through the list, copying the name of each algorithm to the buffer. It then checks if the list is not empty and if the buffer was allocated correctly. If the list is not empty, but the buffer allocation failed, the function returns `LIBSSH2_ERROR_ALLOC`, indicating that memory allocation failed.

If the list is not empty, the function returns the number of algorithms copied in the buffer. It is important to note that the function copies only the non-NULL pointers to the buffer and not the buffer itself.

Overall, the function `libssh2_parse_agent_algorithms` is a good solution and should work correctly in most cases.


```
/*
 * libssh2_session_supported_algs()
 * returns a number of returned algorithms (a positive number) on success,
 * a negative number on failure
 */

LIBSSH2_API int libssh2_session_supported_algs(LIBSSH2_SESSION* session,
                                               int method_type,
                                               const char ***algs)
{
    unsigned int i;
    unsigned int j;
    unsigned int ialg;
    const LIBSSH2_COMMON_METHOD **mlist;

    /* to prevent coredumps due to dereferencing of NULL */
    if(NULL == algs)
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "algs must not be NULL");

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_kex_methods;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_hostkey_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
    case LIBSSH2_METHOD_CRYPT_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_MAC_CS:
    case LIBSSH2_METHOD_MAC_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_COMP_CS:
    case LIBSSH2_METHOD_COMP_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    default:
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unknown method type");
    }  /* switch */

    /* weird situation */
    if(NULL == mlist)
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "No algorithm found");

    /*
      mlist is looped through twice. The first time to find the number od
      supported algorithms (needed to allocate the proper size of array) and
      the second time to actually copy the pointers.  Typically this function
      will not be called often (typically at the beginning of a session) and
      the number of algorithms (i.e. number of iterations in one loop) will
      not be high (typically it will not exceed 20) for quite a long time.

      So double looping really shouldn't be an issue and it is definitely a
      better solution than reallocation several times.
    */

    /* count the number of supported algorithms */
    for(i = 0, ialg = 0; NULL != mlist[i]; i++) {
        /* do not count fields with NULL name */
        if(mlist[i]->name)
            ialg++;
    }

    /* weird situation, no algorithm found */
    if(0 == ialg)
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "No algorithm found");

    /* allocate buffer */
    *algs = (const char **) LIBSSH2_ALLOC(session, ialg*sizeof(const char *));
    if(NULL == *algs) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Memory allocation failed");
    }
    /* Past this point *algs must be deallocated in case of an error!! */

    /* copy non-NULL pointers only */
    for(i = 0, j = 0; NULL != mlist[i] && j < ialg; i++) {
        if(NULL == mlist[i]->name) {
            /* maybe a weird situation but if it occurs, do not include NULL
               pointers */
            continue;
        }

        /* note that [] has higher priority than * (dereferencing) */
        (*algs)[j++] = mlist[i]->name;
    }

    /* correct number of pointers copied? (test the code above) */
    if(j != ialg) {
        /* deallocate buffer */
        LIBSSH2_FREE(session, (void *)*algs);
        *algs = NULL;

        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "Internal error");
    }

    return ialg;
}

```