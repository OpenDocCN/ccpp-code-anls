# Nmap源码解析 106

# `libssh2/src/transport.c`

This is a valid Haxx SSH server configuration file. The file specifies the server to be used for connecting to the Haxx protocol, including the server address, port number, username, and password. The file is intended to be read and modified by the client, but it is recommended to make a backup of the original file before making any changes. The Haxx protocol uses the SECSh transport layer, which is defined in RFC4253.


```cpp
/* Copyright (C) 2007 The Written Word, Inc.  All rights reserved.
 * Copyright (C) 2009-2010 by Daniel Stenberg
 * Author: Daniel Stenberg <daniel@haxx.se>
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
 * This file handles reading and writing to the SECSH transport layer. RFC4253.
 */

```

这段代码是一个用于在Linux系统上实现SSH客户端连接到远程服务器的安全套接字（SSH）库，它的作用是允许用户安全地连接到远程服务器并执行命令。以下是它的主要组成部分：

1. 引入"libssh2_priv.h"头文件，它定义了SSH客户端和SSH库之间的抽象接口，使得代码能够与SSH库进行交互。
2. 引入了"errno.h"和"fcntl.h"头文件，它们分别定义了错误对象和文件输入输出所需的函数。
3. 引入了"ctype.h"头文件，它定义了字符类型，如用于文件输入输出中的字符个数。
4. 包含了"libssh2_sys.h"和"libssh2_sshd辨别器.h"头文件，它们定义了SSH客户端和SSH库的核心部分，允许实现更高级别的功能。
5. 通过包含"transport.h"头文件，引入了负责实现网络传输的代码。
6. 通过包含"mac.h"头文件，引入了负责处理MAC地址的代码。
7. 定义了MAX_BLOCKSIZE和MAX_MACSIZE，用于定义SSH客户端支持的最大块大小和MAC地址长度。
8. 通过引入assert.h"头文件，使得代码能够正确地检测输入是否为真。


```cpp
#include "libssh2_priv.h"
#include <errno.h>
#include <fcntl.h>
#include <ctype.h>
#ifdef LIBSSH2DEBUG
#include <stdio.h>
#endif

#include <assert.h>

#include "transport.h"
#include "mac.h"

#define MAX_BLOCKSIZE 32    /* MUST fit biggest crypto block size we use/get */
#define MAX_MACSIZE 64      /* MUST fit biggest MAC length we support */

```



This is a C function that truncates and formats a large file or input stream into a more manageable form for transmission or storage. It uses the `libssh2_trace_transport` option to determine if the user has requested the entire file or just a portion of it.

The function takes a pointer to a buffer for storing the formatted data, as well as the size of the file or stream. It then reads the contents of the file or stream and formats the data into a string, printing any printable characters along the way.

The function uses a loop to iterate through the buffer and print out the formatted data in a hex format, showing the upper and lowercase hex values for each byte. If the user has not requested the entire file, the function returns early, indicating that it has finished writing the data.

The function also includes logic to determine if the user has disabled the display of the hex values for each byte. This is done by checking the `LIBSSH2_TRACE_TRANS` option, which is a combination of the `LIBSSH2_TRACE_BASIC` and `LIBSSH2_TRACE_TRANSMIT` options. If this option is set to `0`, then the function will not display the hex values for each byte.


```cpp
#ifdef LIBSSH2DEBUG
#define UNPRINTABLE_CHAR '.'
static void
debugdump(LIBSSH2_SESSION * session,
          const char *desc, const unsigned char *ptr, size_t size)
{
    size_t i;
    size_t c;
    unsigned int width = 0x10;
    char buffer[256];  /* Must be enough for width*4 + about 30 or so */
    size_t used;
    static const char *hex_chars = "0123456789ABCDEF";

    if(!(session->showmask & LIBSSH2_TRACE_TRANS)) {
        /* not asked for, bail out */
        return;
    }

    used = snprintf(buffer, sizeof(buffer), "=> %s (%d bytes)\n",
                    desc, (int) size);
    if(session->tracehandler)
        (session->tracehandler)(session, session->tracehandler_context,
                                buffer, used);
    else
        fprintf(stderr, "%s", buffer);

    for(i = 0; i < size; i += width) {

        used = snprintf(buffer, sizeof(buffer), "%04lx: ", (long)i);

        /* hex not disabled, show it */
        for(c = 0; c < width; c++) {
            if(i + c < size) {
                buffer[used++] = hex_chars[(ptr[i + c] >> 4) & 0xF];
                buffer[used++] = hex_chars[ptr[i + c] & 0xF];
            }
            else {
                buffer[used++] = ' ';
                buffer[used++] = ' ';
            }

            buffer[used++] = ' ';
            if((width/2) - 1 == c)
                buffer[used++] = ' ';
        }

        buffer[used++] = ':';
        buffer[used++] = ' ';

        for(c = 0; (c < width) && (i + c < size); c++) {
            buffer[used++] = isprint(ptr[i + c]) ?
                ptr[i + c] : UNPRINTABLE_CHAR;
        }
        buffer[used++] = '\n';
        buffer[used] = 0;

        if(session->tracehandler)
            (session->tracehandler)(session, session->tracehandler_context,
                                    buffer, used);
        else
            fprintf(stderr, "%s", buffer);
    }
}
```

这段代码定义了一个名为“debugdump”的函数，但并没有给它具体的含义。它的存在是为了在编译时检查源代码是否正确。如果代码中出现了这个函数定义，那么它将会在编译时检查源代码，确保所有定义都已经被定义过。如果没有出现这个函数定义，那么编译器将会忽略这个定义。

总的来说，这段代码不会对程序的运行产生影响，但是可能会在编译时帮助开发者进行代码检查。


```cpp
#else
#define debugdump(a,x,y,z)
#endif


/* decrypt() decrypts 'len' bytes from 'source' to 'dest'.
 *
 * returns 0 on success and negative on failure
 */

static int
decrypt(LIBSSH2_SESSION * session, unsigned char *source,
        unsigned char *dest, int len)
{
    struct transportpacket *p = &session->packet;
    int blocksize = session->remote.crypt->blocksize;

    /* if we get called with a len that isn't an even number of blocksizes
       we risk losing those extra bytes */
    assert((len % blocksize) == 0);

    while(len >= blocksize) {
        if(session->remote.crypt->crypt(session, source, blocksize,
                                         &session->remote.crypt_abstract)) {
            LIBSSH2_FREE(session, p->payload);
            return LIBSSH2_ERROR_DECRYPT;
        }

        /* if the crypt() function would write to a given address it
           wouldn't have to memcpy() and we could avoid this memcpy()
           too */
        memcpy(dest, source, blocksize);

        len -= blocksize;       /* less bytes left */
        dest += blocksize;      /* advance write pointer */
        source += blocksize;    /* advance read pointer */
    }
    return LIBSSH2_ERROR_NONE;         /* all is fine */
}

```

This function appears to be the entry point for thelibssh2_transport_read() function, which reads data from the remote host over SSH. It does this by first checking if a compressed buffer for the remote host's data is present. If it is, it is then used to decompress the data if it has not already been compressed. If the data has not been compressed, the function continues by reading the data from the remote host's packets and adding it to the buffer.

After the data has been added to the buffer, the function sets the session'sfullpacket_state to libssh2_NB_state_idle and returns the session->fullpacket_packet_type. If an error occurs, the function returns LIBSSH2_ERROR_EAGAIN.


```cpp
/*
 * fullpacket() gets called when a full packet has been received and properly
 * collected.
 */
static int
fullpacket(LIBSSH2_SESSION * session, int encrypted /* 1 or 0 */ )
{
    unsigned char macbuf[MAX_MACSIZE];
    struct transportpacket *p = &session->packet;
    int rc;
    int compressed;

    if(session->fullpacket_state == libssh2_NB_state_idle) {
        session->fullpacket_macstate = LIBSSH2_MAC_CONFIRMED;
        session->fullpacket_payload_len = p->packet_length - 1;

        if(encrypted) {

            /* Calculate MAC hash */
            session->remote.mac->hash(session, macbuf,  /* store hash here */
                                      session->remote.seqno,
                                      p->init, 5,
                                      p->payload,
                                      session->fullpacket_payload_len,
                                      &session->remote.mac_abstract);

            /* Compare the calculated hash with the MAC we just read from
             * the network. The read one is at the very end of the payload
             * buffer. Note that 'payload_len' here is the packet_length
             * field which includes the padding but not the MAC.
             */
            if(memcmp(macbuf, p->payload + session->fullpacket_payload_len,
                       session->remote.mac->mac_len)) {
                session->fullpacket_macstate = LIBSSH2_MAC_INVALID;
            }
        }

        session->remote.seqno++;

        /* ignore the padding */
        session->fullpacket_payload_len -= p->padding_length;

        /* Check for and deal with decompression */
        compressed =
            session->local.comp != NULL &&
            session->local.comp->compress &&
            ((session->state & LIBSSH2_STATE_AUTHENTICATED) ||
             session->local.comp->use_in_auth);

        if(compressed && session->remote.comp_abstract) {
            /*
             * The buffer for the decompression (remote.comp_abstract) is
             * initialised in time when it is needed so as long it is NULL we
             * cannot decompress.
             */

            unsigned char *data;
            size_t data_len;
            rc = session->remote.comp->decomp(session,
                                              &data, &data_len,
                                              LIBSSH2_PACKET_MAXDECOMP,
                                              p->payload,
                                              session->fullpacket_payload_len,
                                              &session->remote.comp_abstract);
            LIBSSH2_FREE(session, p->payload);
            if(rc)
                return rc;

            p->payload = data;
            session->fullpacket_payload_len = data_len;
        }

        session->fullpacket_packet_type = p->payload[0];

        debugdump(session, "libssh2_transport_read() plain",
                  p->payload, session->fullpacket_payload_len);

        session->fullpacket_state = libssh2_NB_state_created;
    }

    if(session->fullpacket_state == libssh2_NB_state_created) {
        rc = _libssh2_packet_add(session, p->payload,
                                 session->fullpacket_payload_len,
                                 session->fullpacket_macstate);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        if(rc) {
            session->fullpacket_state = libssh2_NB_state_idle;
            return rc;
        }
    }

    session->fullpacket_state = libssh2_NB_state_idle;

    return session->fullpacket_packet_type;
}


```

这段代码定义了一个名为`_libssh2_transport_read`的函数，它用于将一个二进制数据流放入输入队列中。具体来说，这个函数接收一个二进制数据流，将其添加到输入队列中，并返回数据包类型。

该函数的实现基于`libssh2_transport_event`和`libssh2_private_event`库。它使用`_libssh2_event_read_private`函数将接收到的数据包读入到输入队列中，并将其添加到输入队列的头部。如果数据包为空，该函数将返回0；否则，它将返回一个负的错误码。

该函数的作用是用于将二进制数据流放入输入队列中，以供下一个函数处理。它不涉及错误处理，因此如果输入数据包不符合规范，它将输出一个错误码。


```cpp
/*
 * _libssh2_transport_read
 *
 * Collect a packet into the input queue.
 *
 * Returns packet type added to input queue (0 if nothing added), or a
 * negative error number.
 */

/*
 * This function reads the binary stream as specified in chapter 6 of RFC4253
 * "The Secure Shell (SSH) Transport Layer Protocol"
 *
 * DOES NOT call _libssh2_error() for ANY error case.
 */
```

It looks like there's a bug in the code, specifically in the line where it sets `p->total_num` to zero.

It should be `p->total_num = 0` instead of `p->total_num = 0`.

This is because in the `read_packet` function, at the end, it should be called before the return, in order to clear the `p->total_num` before it is reset.

Also, in the same line, it should be `numbytes` instead of `numbytes`


```cpp
int _libssh2_transport_read(LIBSSH2_SESSION * session)
{
    int rc;
    struct transportpacket *p = &session->packet;
    int remainbuf;
    int remainpack;
    int numbytes;
    int numdecrypt;
    unsigned char block[MAX_BLOCKSIZE];
    int blocksize;
    int encrypted = 1;

    /* default clear the bit */
    session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_INBOUND;

    /*
     * All channels, systems, subsystems, etc eventually make it down here
     * when looking for more incoming data. If a key exchange is going on
     * (LIBSSH2_STATE_EXCHANGING_KEYS bit is set) then the remote end will
     * ONLY send key exchange related traffic. In non-blocking mode, there is
     * a chance to break out of the kex_exchange function with an EAGAIN
     * status, and never come back to it. If LIBSSH2_STATE_EXCHANGING_KEYS is
     * active, then we must redirect to the key exchange. However, if
     * kex_exchange is active (as in it is the one that calls this execution
     * of packet_read, then don't redirect, as that would be an infinite loop!
     */

    if(session->state & LIBSSH2_STATE_EXCHANGING_KEYS &&
        !(session->state & LIBSSH2_STATE_KEX_ACTIVE)) {

        /* Whoever wants a packet won't get anything until the key re-exchange
         * is done!
         */
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Redirecting into the"
                       " key re-exchange from _libssh2_transport_read");
        rc = _libssh2_kex_exchange(session, 1, &session->startup_key_state);
        if(rc)
            return rc;
    }

    /*
     * =============================== NOTE ===============================
     * I know this is very ugly and not a really good use of "goto", but
     * this case statement would be even uglier to do it any other way
     */
    if(session->readPack_state == libssh2_NB_state_jump1) {
        session->readPack_state = libssh2_NB_state_idle;
        encrypted = session->readPack_encrypted;
        goto libssh2_transport_read_point1;
    }

    do {
        if(session->socket_state == LIBSSH2_SOCKET_DISCONNECTED) {
            return LIBSSH2_ERROR_SOCKET_DISCONNECT;
        }

        if(session->state & LIBSSH2_STATE_NEWKEYS) {
            blocksize = session->remote.crypt->blocksize;
        }
        else {
            encrypted = 0;      /* not encrypted */
            blocksize = 5;      /* not strictly true, but we can use 5 here to
                                   make the checks below work fine still */
        }

        /* read/use a whole big chunk into a temporary area stored in
           the LIBSSH2_SESSION struct. We will decrypt data from that
           buffer into the packet buffer so this temp one doesn't have
           to be able to keep a whole SSH packet, just be large enough
           so that we can read big chunks from the network layer. */

        /* how much data there is remaining in the buffer to deal with
           before we should read more from the network */
        remainbuf = p->writeidx - p->readidx;

        /* if remainbuf turns negative we have a bad internal error */
        assert(remainbuf >= 0);

        if(remainbuf < blocksize) {
            /* If we have less than a blocksize left, it is too
               little data to deal with, read more */
            ssize_t nread;

            /* move any remainder to the start of the buffer so
               that we can do a full refill */
            if(remainbuf) {
                memmove(p->buf, &p->buf[p->readidx], remainbuf);
                p->readidx = 0;
                p->writeidx = remainbuf;
            }
            else {
                /* nothing to move, just zero the indexes */
                p->readidx = p->writeidx = 0;
            }

            /* now read a big chunk from the network into the temp buffer */
            nread =
                LIBSSH2_RECV(session, &p->buf[remainbuf],
                              PACKETBUFSIZE - remainbuf,
                              LIBSSH2_SOCKET_RECV_FLAGS(session));
            if(nread <= 0) {
                /* check if this is due to EAGAIN and return the special
                   return code if so, error out normally otherwise */
                if((nread < 0) && (nread == -EAGAIN)) {
                    session->socket_block_directions |=
                        LIBSSH2_SESSION_BLOCK_INBOUND;
                    return LIBSSH2_ERROR_EAGAIN;
                }
                _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                               "Error recving %d bytes (got %d)",
                               PACKETBUFSIZE - remainbuf, -nread);
                return LIBSSH2_ERROR_SOCKET_RECV;
            }
            _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                           "Recved %d/%d bytes to %p+%d", nread,
                           PACKETBUFSIZE - remainbuf, p->buf, remainbuf);

            debugdump(session, "libssh2_transport_read() raw",
                      &p->buf[remainbuf], nread);
            /* advance write pointer */
            p->writeidx += nread;

            /* update remainbuf counter */
            remainbuf = p->writeidx - p->readidx;
        }

        /* how much data to deal with from the buffer */
        numbytes = remainbuf;

        if(!p->total_num) {
            size_t total_num;

            /* No payload package area allocated yet. To know the
               size of this payload, we need to decrypt the first
               blocksize data. */

            if(numbytes < blocksize) {
                /* we can't act on anything less than blocksize, but this
                   check is only done for the initial block since once we have
                   got the start of a block we can in fact deal with fractions
                */
                session->socket_block_directions |=
                    LIBSSH2_SESSION_BLOCK_INBOUND;
                return LIBSSH2_ERROR_EAGAIN;
            }

            if(encrypted) {
                rc = decrypt(session, &p->buf[p->readidx], block, blocksize);
                if(rc != LIBSSH2_ERROR_NONE) {
                    return rc;
                }
                /* save the first 5 bytes of the decrypted package, to be
                   used in the hash calculation later down. */
                memcpy(p->init, block, 5);
            }
            else {
                /* the data is plain, just copy it verbatim to
                   the working block buffer */
                memcpy(block, &p->buf[p->readidx], blocksize);
            }

            /* advance the read pointer */
            p->readidx += blocksize;

            /* we now have the initial blocksize bytes decrypted,
             * and we can extract packet and padding length from it
             */
            p->packet_length = _libssh2_ntohu32(block);
            if(p->packet_length < 1) {
                return LIBSSH2_ERROR_DECRYPT;
            }
            else if(p->packet_length > LIBSSH2_PACKET_MAXPAYLOAD) {
                return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
            }

            p->padding_length = block[4];
            if(p->padding_length > p->packet_length - 1) {
                return LIBSSH2_ERROR_DECRYPT;
            }


            /* total_num is the number of bytes following the initial
               (5 bytes) packet length and padding length fields */
            total_num =
                p->packet_length - 1 +
                (encrypted ? session->remote.mac->mac_len : 0);

            /* RFC4253 section 6.1 Maximum Packet Length says:
             *
             * "All implementations MUST be able to process
             * packets with uncompressed payload length of 32768
             * bytes or less and total packet size of 35000 bytes
             * or less (including length, padding length, payload,
             * padding, and MAC.)."
             */
            if(total_num > LIBSSH2_PACKET_MAXPAYLOAD || total_num == 0) {
                return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
            }

            /* Get a packet handle put data into. We get one to
               hold all data, including padding and MAC. */
            p->payload = LIBSSH2_ALLOC(session, total_num);
            if(!p->payload) {
                return LIBSSH2_ERROR_ALLOC;
            }
            p->total_num = total_num;
            /* init write pointer to start of payload buffer */
            p->wptr = p->payload;

            if(blocksize > 5) {
                /* copy the data from index 5 to the end of
                   the blocksize from the temporary buffer to
                   the start of the decrypted buffer */
                if(blocksize - 5 <= (int) total_num) {
                    memcpy(p->wptr, &block[5], blocksize - 5);
                    p->wptr += blocksize - 5;       /* advance write pointer */
                }
                else {
                    if(p->payload)
                        LIBSSH2_FREE(session, p->payload);
                    return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
                }
            }

            /* init the data_num field to the number of bytes of
               the package read so far */
            p->data_num = p->wptr - p->payload;

            /* we already dealt with a blocksize worth of data */
            numbytes -= blocksize;
        }

        /* how much there is left to add to the current payload
           package */
        remainpack = p->total_num - p->data_num;

        if(numbytes > remainpack) {
            /* if we have more data in the buffer than what is going into this
               particular packet, we limit this round to this packet only */
            numbytes = remainpack;
        }

        if(encrypted) {
            /* At the end of the incoming stream, there is a MAC,
               and we don't want to decrypt that since we need it
               "raw". We MUST however decrypt the padding data
               since it is used for the hash later on. */
            int skip = session->remote.mac->mac_len;

            /* if what we have plus numbytes is bigger than the
               total minus the skip margin, we should lower the
               amount to decrypt even more */
            if((p->data_num + numbytes) > (p->total_num - skip)) {
                numdecrypt = (p->total_num - skip) - p->data_num;
            }
            else {
                int frac;
                numdecrypt = numbytes;
                frac = numdecrypt % blocksize;
                if(frac) {
                    /* not an aligned amount of blocks,
                       align it */
                    numdecrypt -= frac;
                    /* and make it no unencrypted data
                       after it */
                    numbytes = 0;
                }
            }
        }
        else {
            /* unencrypted data should not be decrypted at all */
            numdecrypt = 0;
        }

        /* if there are bytes to decrypt, do that */
        if(numdecrypt > 0) {
            /* now decrypt the lot */
            rc = decrypt(session, &p->buf[p->readidx], p->wptr, numdecrypt);
            if(rc != LIBSSH2_ERROR_NONE) {
                p->total_num = 0;   /* no packet buffer available */
                return rc;
            }

            /* advance the read pointer */
            p->readidx += numdecrypt;
            /* advance write pointer */
            p->wptr += numdecrypt;
            /* increase data_num */
            p->data_num += numdecrypt;

            /* bytes left to take care of without decryption */
            numbytes -= numdecrypt;
        }

        /* if there are bytes to copy that aren't decrypted, simply
           copy them as-is to the target buffer */
        if(numbytes > 0) {

            if(numbytes <= (int)(p->total_num - (p->wptr - p->payload))) {
                memcpy(p->wptr, &p->buf[p->readidx], numbytes);
            }
            else {
                if(p->payload)
                    LIBSSH2_FREE(session, p->payload);
                return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
            }

            /* advance the read pointer */
            p->readidx += numbytes;
            /* advance write pointer */
            p->wptr += numbytes;
            /* increase data_num */
            p->data_num += numbytes;
        }

        /* now check how much data there's left to read to finish the
           current packet */
        remainpack = p->total_num - p->data_num;

        if(!remainpack) {
            /* we have a full packet */
          libssh2_transport_read_point1:
            rc = fullpacket(session, encrypted);
            if(rc == LIBSSH2_ERROR_EAGAIN) {

                if(session->packAdd_state != libssh2_NB_state_idle) {
                    /* fullpacket only returns LIBSSH2_ERROR_EAGAIN if
                     * libssh2_packet_add returns LIBSSH2_ERROR_EAGAIN. If
                     * that returns LIBSSH2_ERROR_EAGAIN but the packAdd_state
                     * is idle, then the packet has been added to the brigade,
                     * but some immediate action that was taken based on the
                     * packet type (such as key re-exchange) is not yet
                     * complete.  Clear the way for a new packet to be read
                     * in.
                     */
                    session->readPack_encrypted = encrypted;
                    session->readPack_state = libssh2_NB_state_jump1;
                }

                return rc;
            }

            p->total_num = 0;   /* no packet buffer available */

            return rc;
        }
    } while(1);                /* loop */

    return LIBSSH2_ERROR_SOCKET_RECV; /* we never reach this point */
}

```

This function appears to be the purpose of the `send()` function in the SSH2 protocol. It takes a pointer to a `send_output` structure, which contains information about the data to be sent and the number of bytes that have been sent. The function sends the specified data to the server and returns an error code.

Here's what the function does step by step:

1.  Initialize the `length` variable to the difference between the total number of bytes and the number of bytes that have already been sent.
2.  Initialize the `p->outbuf` pointer to the start of the data to be sent.
3.  Initialize the `p->osent` variable to the number of bytes that have been sent.
4.  Initialize the `p->ototal_num` variable to the total number of bytes that are to be sent.
5.  Initialize the `p->呕吐` variable to a negative value.
6.  Call the `LIBSSH2_SEND()` function, passing in the `session` object, the `outbuf` pointer, and the `length` flag. This function sends the data to the server and returns an error code.
7.  If the error code is negative, the function logs the error and returns an error code for the `LIBSSH2_ERROR_NONE` case. If the error code is 0, the function logs the error and returns the `LIBSSH2_ERROR_EAGAIN` case. If the error code is negative or not defined, the function logs the error and returns `LIBSSH2_ERROR_SOCKET_SEND`.
8.  If the function has not finished sending the data, it sets the `session->socket_block_directions` flag to indicate that the data transfer is complete and returns the `LIBSSH2_ERROR_NONE` error code.

Note that the `send()` function also checks for the case where the server has sent all of the data and returns an error code for the `EAGAIN` case. If this error code is encountered, it is important to note that the `send()` function has already completed sending the data and should not be called again.


```cpp
static int
send_existing(LIBSSH2_SESSION *session, const unsigned char *data,
              size_t data_len, ssize_t *ret)
{
    ssize_t rc;
    ssize_t length;
    struct transportpacket *p = &session->packet;

    if(!p->olen) {
        *ret = 0;
        return LIBSSH2_ERROR_NONE;
    }

    /* send as much as possible of the existing packet */
    if((data != p->odata) || (data_len != p->olen)) {
        /* When we are about to complete the sending of a packet, it is vital
           that the caller doesn't try to send a new/different packet since
           we don't add this one up until the previous one has been sent. To
           make the caller really notice his/hers flaw, we return error for
           this case */
        return LIBSSH2_ERROR_BAD_USE;
    }

    *ret = 1;                   /* set to make our parent return */

    /* number of bytes left to send */
    length = p->ototal_num - p->osent;

    rc = LIBSSH2_SEND(session, &p->outbuf[p->osent], length,
                       LIBSSH2_SOCKET_SEND_FLAGS(session));
    if(rc < 0)
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Error sending %d bytes: %d", length, -rc);
    else {
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Sent %d/%d bytes at %p+%d", rc, length, p->outbuf,
                       p->osent);
        debugdump(session, "libssh2_transport_write send()",
                  &p->outbuf[p->osent], rc);
    }

    if(rc == length) {
        /* the remainder of the package was sent */
        p->ototal_num = 0;
        p->olen = 0;
        /* we leave *ret set so that the parent returns as we MUST return back
           a send success now, so that we don't risk sending EAGAIN later
           which then would confuse the parent function */
        return LIBSSH2_ERROR_NONE;

    }
    else if(rc < 0) {
        /* nothing was sent */
        if(rc != -EAGAIN)
            /* send failure! */
            return LIBSSH2_ERROR_SOCKET_SEND;

        session->socket_block_directions |= LIBSSH2_SESSION_BLOCK_OUTBOUND;
        return LIBSSH2_ERROR_EAGAIN;
    }

    p->osent += rc;         /* we sent away this much data */

    return rc < length ? LIBSSH2_ERROR_EAGAIN : LIBSSH2_ERROR_NONE;
}

```

这段代码定义了一个名为`libssh2_transport_send`的函数，它的作用是发送一个数据包，并在需要时对数据包进行加密，并添加一个MAC代码。

具体来说，这个函数接受两个数据区域，将它们组合起来，然后将数据区域发送给目标主机。如果在发送数据包过程中遇到问题，比如阻塞或者整个数据包没有发送完，那么函数将会返回`LIBSSH2_ERROR_EAGAIN`，调用者应该在可能的情况下再次调用这个函数，并将相同的参数设置好。

需要注意的是，这个函数不会调用`_libssh2_error()`函数来处理任何错误，因此如果你在使用这个函数时遇到错误，你可能需要自己处理异常情况。


```cpp
/*
 * libssh2_transport_send
 *
 * Send a packet, encrypting it and adding a MAC code if necessary
 * Returns 0 on success, non-zero on failure.
 *
 * The data is provided as _two_ data areas that are combined by this
 * function.  The 'data' part is sent immediately before 'data2'. 'data2' may
 * be set to NULL to only use a single part.
 *
 * Returns LIBSSH2_ERROR_EAGAIN if it would block or if the whole packet was
 * not sent yet. If it does so, the caller should call this function again as
 * soon as it is likely that more data can be sent, and this function MUST
 * then be called with the same argument set (same data pointer and same
 * data_len) until ERROR_NONE or failure is returned.
 *
 * This function DOES NOT call _libssh2_error() on any errors.
 */
```

The code you provided is an SSH client library that includes support for sending raw SSH packets over a network. The library includes functionality to dynamically allocate the size of the packet, copy data from the input data to the output, and send the packet over the network.

The library has a number of error codes:

* LIBSSH2_ERROR_INVAL: The functionallity is invalid. This error occurs if the input data cannot be successfully parsed or the packet cannot be formatted as an SSH packet.
* LIBSSH2_ERROR_TRY_AGAIN: The user failed to authenticate with the remote host.
* LIBSSH2_ERROR_USER_CAN_NOT_TRUST: The user's authentication credentials are not trusted.
* LIBSSH2_ERROR_PASSWORD_SALT_NOT_SATISFIED: The password salt generated by the server is not satisfied.
* LIBSSH2_ERROR_PASSWORD_Too_Short: The password is too short.
* LIBSSH2_ERROR_REMOTE_USER_NAME_QUALIFY: The remote user name does not match the expected name.
* LIBSSH2_ERROR_REMOTE_USER_NAME_NOT_FOUND: The remote user name cannot be found.
* LIBSSH2_ERROR_REMOTE_USER_PASSWORD_QUALIFY: The remote user password does not match the expected password.
* LIBSSH2_ERROR_REMOTE_USER_PASSWORD_TO_LONG: The remote user password is too long.
* LIBSSH2_ERROR_TAG: An invalid tag was specified.
* LIBSSH2_ERROR_USER_ABorted: The user was forced to disconnect.

The library also includes a function to format the packet, which has a required length of 4+1+2+1=8 bytes. This function is used to allocate space for the packet header and to ensure that the packet is properly formatted before it is sent over the network.

Overall, this library appears to be a well-structured and well-documented implementation of SSH functionality that is easy to use and error-free.


```cpp
int _libssh2_transport_send(LIBSSH2_SESSION *session,
                            const unsigned char *data, size_t data_len,
                            const unsigned char *data2, size_t data2_len)
{
    int blocksize =
        (session->state & LIBSSH2_STATE_NEWKEYS) ?
        session->local.crypt->blocksize : 8;
    int padding_length;
    size_t packet_length;
    int total_length;
#ifdef RANDOM_PADDING
    int rand_max;
    int seed = data[0];         /* FIXME: make this random */
#endif
    struct transportpacket *p = &session->packet;
    int encrypted;
    int compressed;
    ssize_t ret;
    int rc;
    const unsigned char *orgdata = data;
    size_t orgdata_len = data_len;

    /*
     * If the last read operation was interrupted in the middle of a key
     * exchange, we must complete that key exchange before continuing to write
     * further data.
     *
     * See the similar block in _libssh2_transport_read for more details.
     */
    if(session->state & LIBSSH2_STATE_EXCHANGING_KEYS &&
        !(session->state & LIBSSH2_STATE_KEX_ACTIVE)) {
        /* Don't write any new packets if we're still in the middle of a key
         * exchange. */
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Redirecting into the"
                       " key re-exchange from _libssh2_transport_send");
        rc = _libssh2_kex_exchange(session, 1, &session->startup_key_state);
        if(rc)
            return rc;
    }

    debugdump(session, "libssh2_transport_write plain", data, data_len);
    if(data2)
        debugdump(session, "libssh2_transport_write plain2", data2, data2_len);

    /* FIRST, check if we have a pending write to complete. send_existing
       only sanity-check data and data_len and not data2 and data2_len!! */
    rc = send_existing(session, data, data_len, &ret);
    if(rc)
        return rc;

    session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_OUTBOUND;

    if(ret)
        /* set by send_existing if data was sent */
        return rc;

    encrypted = (session->state & LIBSSH2_STATE_NEWKEYS) ? 1 : 0;

    compressed =
        session->local.comp != NULL &&
        session->local.comp->compress &&
        ((session->state & LIBSSH2_STATE_AUTHENTICATED) ||
         session->local.comp->use_in_auth);

    if(encrypted && compressed && session->local.comp_abstract) {
        /* the idea here is that these function must fail if the output gets
           larger than what fits in the assigned buffer so thus they don't
           check the input size as we don't know how much it compresses */
        size_t dest_len = MAX_SSH_PACKET_LEN-5-256;
        size_t dest2_len = dest_len;

        /* compress directly to the target buffer */
        rc = session->local.comp->comp(session,
                                       &p->outbuf[5], &dest_len,
                                       data, data_len,
                                       &session->local.comp_abstract);
        if(rc)
            return rc;     /* compression failure */

        if(data2 && data2_len) {
            /* compress directly to the target buffer right after where the
               previous call put data */
            dest2_len -= dest_len;

            rc = session->local.comp->comp(session,
                                           &p->outbuf[5 + dest_len],
                                           &dest2_len,
                                           data2, data2_len,
                                           &session->local.comp_abstract);
        }
        else
            dest2_len = 0;
        if(rc)
            return rc;     /* compression failure */

        data_len = dest_len + dest2_len; /* use the combined length */
    }
    else {
        if((data_len + data2_len) >= (MAX_SSH_PACKET_LEN-0x100))
            /* too large packet, return error for this until we make this
               function split it up and send multiple SSH packets */
            return LIBSSH2_ERROR_INVAL;

        /* copy the payload data */
        memcpy(&p->outbuf[5], data, data_len);
        if(data2 && data2_len)
            memcpy(&p->outbuf[5 + data_len], data2, data2_len);
        data_len += data2_len; /* use the combined length */
    }


    /* RFC4253 says: Note that the length of the concatenation of
       'packet_length', 'padding_length', 'payload', and 'random padding'
       MUST be a multiple of the cipher block size or 8, whichever is
       larger. */

    /* Plain math: (4 + 1 + packet_length + padding_length) % blocksize == 0 */

    packet_length = data_len + 1 + 4;   /* 1 is for padding_length field
                                           4 for the packet_length field */

    /* at this point we have it all except the padding */

    /* first figure out our minimum padding amount to make it an even
       block size */
    padding_length = blocksize - (packet_length % blocksize);

    /* if the padding becomes too small we add another blocksize worth
       of it (taken from the original libssh2 where it didn't have any
       real explanation) */
    if(padding_length < 4) {
        padding_length += blocksize;
    }
```



This function appears to be part of the SSH2 protocol, specifically the use of the libssh2 library.

It is a function called `libssh2_transport_write`, which presumably writes data to a remote host over an SSH connection using libssh2. The function takes an input packet (`p`) containing the data to be written, and a handle (`h`) for the connection.

The function is跳过了一些调试输出，但似乎会处理成功的情况。根据函数的实现，如果写入失败，函数将返回 LIBSSH2_ERROR_ENCRYPT，但不会返回具体的错误代码。如果写入成功，函数将返回 LIBSSH2_TRACE_SOCKET，并且会在输出日志中记录发送的包数和实际发送的数据字节数。

根据函数的实现，它看起来是一个简单的 API，用于将数据写入到远程主机，但具体的实现细节可能因为libssh2库的不同而有所不同。


```cpp
#ifdef RANDOM_PADDING
    /* FIXME: we can add padding here, but that also makes the packets
       bigger etc */

    /* now we can add 'blocksize' to the padding_length N number of times
       (to "help thwart traffic analysis") but it must be less than 255 in
       total */
    rand_max = (255 - padding_length) / blocksize + 1;
    padding_length += blocksize * (seed % rand_max);
#endif

    packet_length += padding_length;

    /* append the MAC length to the total_length size */
    total_length =
        packet_length + (encrypted ? session->local.mac->mac_len : 0);

    /* store packet_length, which is the size of the whole packet except
       the MAC and the packet_length field itself */
    _libssh2_htonu32(p->outbuf, packet_length - 4);
    /* store padding_length */
    p->outbuf[4] = (unsigned char)padding_length;

    /* fill the padding area with random junk */
    if(_libssh2_random(p->outbuf + 5 + data_len, padding_length)) {
        return _libssh2_error(session, LIBSSH2_ERROR_RANDGEN,
                              "Unable to get random bytes for packet padding");
    }

    if(encrypted) {
        size_t i;

        /* Calculate MAC hash. Put the output at index packet_length,
           since that size includes the whole packet. The MAC is
           calculated on the entire unencrypted packet, including all
           fields except the MAC field itself. */
        session->local.mac->hash(session, p->outbuf + packet_length,
                                 session->local.seqno, p->outbuf,
                                 packet_length, NULL, 0,
                                 &session->local.mac_abstract);

        /* Encrypt the whole packet data, one block size at a time.
           The MAC field is not encrypted. */
        for(i = 0; i < packet_length; i += session->local.crypt->blocksize) {
            unsigned char *ptr = &p->outbuf[i];
            if(session->local.crypt->crypt(session, ptr,
                                            session->local.crypt->blocksize,
                                            &session->local.crypt_abstract))
                return LIBSSH2_ERROR_ENCRYPT;     /* encryption failure */
        }
    }

    session->local.seqno++;

    ret = LIBSSH2_SEND(session, p->outbuf, total_length,
                        LIBSSH2_SOCKET_SEND_FLAGS(session));
    if(ret < 0)
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Error sending %d bytes: %d", total_length, -ret);
    else {
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET, "Sent %d/%d bytes at %p",
                       ret, total_length, p->outbuf);
        debugdump(session, "libssh2_transport_write send()", p->outbuf, ret);
    }

    if(ret != total_length) {
        if(ret >= 0 || ret == -EAGAIN) {
            /* the whole packet could not be sent, save the rest */
            session->socket_block_directions |= LIBSSH2_SESSION_BLOCK_OUTBOUND;
            p->odata = orgdata;
            p->olen = orgdata_len;
            p->osent = ret <= 0 ? 0 : ret;
            p->ototal_num = total_length;
            return LIBSSH2_ERROR_EAGAIN;
        }
        return LIBSSH2_ERROR_SOCKET_SEND;
    }

    /* the whole thing got sent away */
    p->odata = NULL;
    p->olen = 0;

    return LIBSSH2_ERROR_NONE;         /* all is good */
}

```

# `libssh2/src/userauth.c`

DottedMag is a software package for mathematical table manipulation. It includes support for creating, manipulating, and exporting tables in various formats, such as LaTeX, MathML, and HTML. DottedMag is distributed under the GPL license, which allows for free and open-source use, modification, and distribution of the software as long as the copyright notice and other specified conditions are included. The software can be run on various platforms, including Windows, Linux, and macOS.


```cpp
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2005 Mikhail Gusarov <dottedmag@dottedmag.net>
 * Copyright (c) 2009-2014 by Daniel Stenberg
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

这段代码是一个用于SSH客户端连接的C应用程序，它包括以下几个部分：

1. `#include "libssh2_priv.h"`：引入了SSH2的私有数据结构头文件。
2. `#include <ctype.h>`：引入了C语言的CTYPE库，用于输入文件和字符串操作。
3. `#include <stdio.h>`：引入了C语言的标准输入输出库。
4. `#include <assert.h>`：引入了数学中的assert库，用于在编译时检查输入的正确性。
5. `#include "transport.h"`：引入了传输层的头文件，用于在客户端和服务器之间传输数据。
6. `#include "session.h"`：引入了会话层的头文件，用于在客户端和服务器之间建立并管理会话。
7. `#include "userauth.h"`：引入了用户认证层的头文件，用于在客户端和服务器之间进行用户认证。
8. `#ifdef HAVE_SYS_UIO_H`：这是一个条件编译，判断Linux系统是否支持SYS_UIO头文件。如果支持，就不需要包含`#include <sys/uio.h>`。
9. `#include "ssh2.h"`：包含SSH2客户端的接口函数。

综上所述，这段代码是一个用于SSH客户端连接的C应用程序，它支持输入文件、输出文件、CTYPE库、数学库、传输层库、会话层库、用户认证层库以及SSH2客户端的接口函数。


```cpp
#include "libssh2_priv.h"

#include <ctype.h>
#include <stdio.h>

#include <assert.h>

/* Needed for struct iovec on some platforms */
#ifdef HAVE_SYS_UIO_H
#include <sys/uio.h>
#endif

#include "transport.h"
#include "session.h"
#include "userauth.h"

```

This function appears to handle the authentication process for a user, using the user's authentication methods. It does this by first checking if the authentication methods are present in the user's authentication list. If the methods are not present, it creates a new authentication list and sets the user's authentication state to LIBSSH2\_STATE\_AUTHENTICATED.

If the authentication methods are present, it checks the length of


```cpp
/* libssh2_userauth_list
 *
 * List authentication methods
 * Will yield successful login if "none" happens to be allowable for this user
 * Not a common configuration for any SSH server though
 * username should be NULL, or a null terminated string
 */
static char *userauth_list(LIBSSH2_SESSION *session, const char *username,
                           unsigned int username_len)
{
    static const unsigned char reply_codes[3] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE, 0 };
    /* packet_type(1) + username_len(4) + service_len(4) +
       service(14)"ssh-connection" + method_len(4) = 27 */
    unsigned long methods_len;
    unsigned char *s;
    int rc;

    if(session->userauth_list_state == libssh2_NB_state_idle) {
        /* Zero the whole thing out */
        memset(&session->userauth_list_packet_requirev_state, 0,
               sizeof(session->userauth_list_packet_requirev_state));

        session->userauth_list_data_len = username_len + 27;

        s = session->userauth_list_data =
            LIBSSH2_ALLOC(session, session->userauth_list_data_len);
        if(!session->userauth_list_data) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for userauth_list");
            return NULL;
        }

        *(s++) = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&s, username, username_len);
        _libssh2_store_str(&s, "ssh-connection", 14);
        _libssh2_store_u32(&s, 4); /* send "none" separately */

        session->userauth_list_state = libssh2_NB_state_created;
    }

    if(session->userauth_list_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, session->userauth_list_data,
                                     session->userauth_list_data_len,
                                     (unsigned char *)"none", 4);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting userauth list");
            return NULL;
        }
        /* now free the packet that was sent */
        LIBSSH2_FREE(session, session->userauth_list_data);
        session->userauth_list_data = NULL;

        if(rc) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send userauth-none request");
            session->userauth_list_state = libssh2_NB_state_idle;
            return NULL;
        }

        session->userauth_list_state = libssh2_NB_state_sent;
    }

    if(session->userauth_list_state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_requirev(session, reply_codes,
                                      &session->userauth_list_data,
                                      &session->userauth_list_data_len, 0,
                                      NULL, 0,
                               &session->userauth_list_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting userauth list");
            return NULL;
        }
        else if(rc || (session->userauth_list_data_len < 1)) {
            _libssh2_error(session, rc, "Failed getting response");
            session->userauth_list_state = libssh2_NB_state_idle;
            return NULL;
        }

        if(session->userauth_list_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
            /* Wow, who'dve thought... */
            _libssh2_error(session, LIBSSH2_ERROR_NONE, "No error");
            LIBSSH2_FREE(session, session->userauth_list_data);
            session->userauth_list_data = NULL;
            session->state |= LIBSSH2_STATE_AUTHENTICATED;
            session->userauth_list_state = libssh2_NB_state_idle;
            return NULL;
        }

        if(session->userauth_list_data_len < 5) {
            LIBSSH2_FREE(session, session->userauth_list_data);
            session->userauth_list_data = NULL;
            _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                           "Unexpected packet size");
            return NULL;
        }

        methods_len = _libssh2_ntohu32(session->userauth_list_data + 1);
        if(methods_len >= session->userauth_list_data_len - 5) {
            _libssh2_error(session, LIBSSH2_ERROR_OUT_OF_BOUNDARY,
                           "Unexpected userauth list size");
            return NULL;
        }

        /* Do note that the memory areas overlap! */
        memmove(session->userauth_list_data, session->userauth_list_data + 5,
                methods_len);
        session->userauth_list_data[methods_len] = '\0';
        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Permitted auth methods: %s",
                       session->userauth_list_data);
    }

    session->userauth_list_state = libssh2_NB_state_idle;
    return (char *) session->userauth_list_data;
}

```

这段代码定义了一个名为 `libssh2_userauth_list` 的函数，属于 `libssh2_ssh` 库。它的作用是返回一个整数，用于指定用户验证的方法。以下是该函数的实现细节：

1. 函数参数：
	* `session`：当前 SSH 会话的句柄。
	* `user`：要验证的用户名，最好是一个 null terminated string。
	* `user_len`：用户名的长度，以字节为单位。
2. 函数实现：
	* 在函数内部，首先调用 `userauth_list` 函数，传入 `session` 和 `user` 参数，以及用户名的长度。将返回的结果存储在 `ptr` 变量中。
	* 如果 `user` 是一个空字符串，函数会直接返回 0，而不是一个用户名验证的返回值。
	* 函数返回值是一个整数，表示用户验证的失败次数，或者 success次数（如果所有验证都成功）。

该函数的作用是用于一个 SSH 服务器，让服务器知道要验证哪种用户身份，并返回一个整数，用于指示验证结果。成功验证的结果为 0，失败的结果为验证失败的数量。


```cpp
/* libssh2_userauth_list
 *
 * List authentication methods
 * Will yield successful login if "none" happens to be allowable for this user
 * Not a common configuration for any SSH server though
 * username should be NULL, or a null terminated string
 */
LIBSSH2_API char *
libssh2_userauth_list(LIBSSH2_SESSION * session, const char *user,
                      unsigned int user_len)
{
    char *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session,
                       userauth_list(session, user, user_len));
    return ptr;
}

```

这段代码是一个用于 SSH 认证的函数，它的作用是判断用户是否已经认证。具体来说，它接收一个 SSH2 会话（SSH2 session）对象，然后返回一个整数，表示用户是否已经认证。如果用户已经登录，函数返回 1，否则返回 0。

函数中的第一个参数 `session` 是一个指向 SSH2 会话对象的指针。它用来判断用户是否已经登录，并且作为函数的局部变量保存。函数中的第二个参数 `LIBSSH2_STATE_AUTHENTICATED` 是 SSH2 状态字，用于指示会话是否已准备好登录。如果这个字设置为 1，说明用户已经登录，函数返回 1；否则，函数返回 0。


```cpp
/*
 * libssh2_userauth_authenticated
 *
 * Returns: 0 if not yet authenticated
 *          1 if already authenticated
 */
LIBSSH2_API int
libssh2_userauth_authenticated(LIBSSH2_SESSION * session)
{
    return (session->state & LIBSSH2_STATE_AUTHENTICATED)?1:0;
}



/* userauth_password
 * Plain ol' login
 */
```

This is a function in the OpenSSH library that handles the authentication flow for


```cpp
static int
userauth_password(LIBSSH2_SESSION *session,
                  const char *username, unsigned int username_len,
                  const unsigned char *password, unsigned int password_len,
                  LIBSSH2_PASSWD_CHANGEREQ_FUNC((*passwd_change_cb)))
{
    unsigned char *s;
    static const unsigned char reply_codes[4] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE,
          SSH_MSG_USERAUTH_PASSWD_CHANGEREQ, 0
        };
    int rc;

    if(session->userauth_pswd_state == libssh2_NB_state_idle) {
        /* Zero the whole thing out */
        memset(&session->userauth_pswd_packet_requirev_state, 0,
               sizeof(session->userauth_pswd_packet_requirev_state));

        /*
         * 40 = packet_type(1) + username_len(4) + service_len(4) +
         * service(14)"ssh-connection" + method_len(4) + method(8)"password" +
         * chgpwdbool(1) + password_len(4) */
        session->userauth_pswd_data_len = username_len + 40;

        session->userauth_pswd_data0 =
            (unsigned char) ~SSH_MSG_USERAUTH_PASSWD_CHANGEREQ;

        /* TODO: remove this alloc with a fixed buffer in the session
           struct */
        s = session->userauth_pswd_data =
            LIBSSH2_ALLOC(session, session->userauth_pswd_data_len);
        if(!session->userauth_pswd_data) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "userauth-password request");
        }

        *(s++) = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&s, username, username_len);
        _libssh2_store_str(&s, "ssh-connection", sizeof("ssh-connection") - 1);
        _libssh2_store_str(&s, "password", sizeof("password") - 1);
        *s++ = '\0';
        _libssh2_store_u32(&s, password_len);
        /* 'password' is sent separately */

        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting to login using password authentication");

        session->userauth_pswd_state = libssh2_NB_state_created;
    }

    if(session->userauth_pswd_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, session->userauth_pswd_data,
                                     session->userauth_pswd_data_len,
                                     password, password_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block writing password request");
        }

        /* now free the sent packet */
        LIBSSH2_FREE(session, session->userauth_pswd_data);
        session->userauth_pswd_data = NULL;

        if(rc) {
            session->userauth_pswd_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-password request");
        }

        session->userauth_pswd_state = libssh2_NB_state_sent;
    }

  password_response:

    if((session->userauth_pswd_state == libssh2_NB_state_sent)
        || (session->userauth_pswd_state == libssh2_NB_state_sent1)
        || (session->userauth_pswd_state == libssh2_NB_state_sent2)) {
        if(session->userauth_pswd_state == libssh2_NB_state_sent) {
            rc = _libssh2_packet_requirev(session, reply_codes,
                                          &session->userauth_pswd_data,
                                          &session->userauth_pswd_data_len,
                                          0, NULL, 0,
                                          &session->
                                          userauth_pswd_packet_requirev_state);

            if(rc) {
                if(rc != LIBSSH2_ERROR_EAGAIN)
                    session->userauth_pswd_state = libssh2_NB_state_idle;

                return _libssh2_error(session, rc,
                                      "Waiting for password response");
            }
            else if(session->userauth_pswd_data_len < 1) {
                session->userauth_pswd_state = libssh2_NB_state_idle;
                return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                      "Unexpected packet size");
            }

            if(session->userauth_pswd_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
                _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                               "Password authentication successful");
                LIBSSH2_FREE(session, session->userauth_pswd_data);
                session->userauth_pswd_data = NULL;
                session->state |= LIBSSH2_STATE_AUTHENTICATED;
                session->userauth_pswd_state = libssh2_NB_state_idle;
                return 0;
            }
            else if(session->userauth_pswd_data[0] ==
                    SSH_MSG_USERAUTH_FAILURE) {
                _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                               "Password authentication failed");
                LIBSSH2_FREE(session, session->userauth_pswd_data);
                session->userauth_pswd_data = NULL;
                session->userauth_pswd_state = libssh2_NB_state_idle;
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                                      "Authentication failed "
                                      "(username/password)");
            }

            session->userauth_pswd_newpw = NULL;
            session->userauth_pswd_newpw_len = 0;

            session->userauth_pswd_state = libssh2_NB_state_sent1;
        }

        if(session->userauth_pswd_data_len < 1) {
            session->userauth_pswd_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Unexpected packet size");
        }

        if((session->userauth_pswd_data[0] ==
             SSH_MSG_USERAUTH_PASSWD_CHANGEREQ)
            || (session->userauth_pswd_data0 ==
                SSH_MSG_USERAUTH_PASSWD_CHANGEREQ)) {
            session->userauth_pswd_data0 = SSH_MSG_USERAUTH_PASSWD_CHANGEREQ;

            if((session->userauth_pswd_state == libssh2_NB_state_sent1) ||
                (session->userauth_pswd_state == libssh2_NB_state_sent2)) {
                if(session->userauth_pswd_state == libssh2_NB_state_sent1) {
                    _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                                   "Password change required");
                    LIBSSH2_FREE(session, session->userauth_pswd_data);
                    session->userauth_pswd_data = NULL;
                }
                if(passwd_change_cb) {
                    if(session->userauth_pswd_state ==
                       libssh2_NB_state_sent1) {
                        passwd_change_cb(session,
                                         &session->userauth_pswd_newpw,
                                         &session->userauth_pswd_newpw_len,
                                         &session->abstract);
                        if(!session->userauth_pswd_newpw) {
                            return _libssh2_error(session,
                                                LIBSSH2_ERROR_PASSWORD_EXPIRED,
                                                  "Password expired, and "
                                                  "callback failed");
                        }

                        /* basic data_len + newpw_len(4) */
                        if(username_len + password_len + 44 <= UINT_MAX) {
                            session->userauth_pswd_data_len =
                                username_len + password_len + 44;
                            s = session->userauth_pswd_data =
                                LIBSSH2_ALLOC(session,
                                              session->userauth_pswd_data_len);
                        }
                        else {
                            s = session->userauth_pswd_data = NULL;
                            session->userauth_pswd_data_len = 0;
                        }

                        if(!session->userauth_pswd_data) {
                            LIBSSH2_FREE(session,
                                         session->userauth_pswd_newpw);
                            session->userauth_pswd_newpw = NULL;
                            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                                  "Unable to allocate memory "
                                                  "for userauth password "
                                                  "change request");
                        }

                        *(s++) = SSH_MSG_USERAUTH_REQUEST;
                        _libssh2_store_str(&s, username, username_len);
                        _libssh2_store_str(&s, "ssh-connection",
                                           sizeof("ssh-connection") - 1);
                        _libssh2_store_str(&s, "password",
                                           sizeof("password") - 1);
                        *s++ = 0x01;
                        _libssh2_store_str(&s, (char *)password, password_len);
                        _libssh2_store_u32(&s,
                                           session->userauth_pswd_newpw_len);
                        /* send session->userauth_pswd_newpw separately */

                        session->userauth_pswd_state = libssh2_NB_state_sent2;
                    }

                    if(session->userauth_pswd_state ==
                       libssh2_NB_state_sent2) {
                        rc = _libssh2_transport_send(session,
                                            session->userauth_pswd_data,
                                            session->userauth_pswd_data_len,
                                            (unsigned char *)
                                            session->userauth_pswd_newpw,
                                            session->userauth_pswd_newpw_len);
                        if(rc == LIBSSH2_ERROR_EAGAIN) {
                            return _libssh2_error(session,
                                                  LIBSSH2_ERROR_EAGAIN,
                                                  "Would block waiting");
                        }

                        /* free the allocated packets again */
                        LIBSSH2_FREE(session, session->userauth_pswd_data);
                        session->userauth_pswd_data = NULL;
                        LIBSSH2_FREE(session, session->userauth_pswd_newpw);
                        session->userauth_pswd_newpw = NULL;

                        if(rc) {
                            return _libssh2_error(session,
                                                  LIBSSH2_ERROR_SOCKET_SEND,
                                                  "Unable to send userauth "
                                                  "password-change request");
                        }

                        /*
                         * Ugliest use of goto ever.  Blame it on the
                         * askN => requirev migration.
                         */
                        session->userauth_pswd_state = libssh2_NB_state_sent;
                        goto password_response;
                    }
                }
            }
            else {
                session->userauth_pswd_state = libssh2_NB_state_idle;
                return _libssh2_error(session, LIBSSH2_ERROR_PASSWORD_EXPIRED,
                                      "Password Expired, and no callback "
                                      "specified");
            }
        }
    }

    /* FAILURE */
    LIBSSH2_FREE(session, session->userauth_pswd_data);
    session->userauth_pswd_data = NULL;
    session->userauth_pswd_state = libssh2_NB_state_idle;

    return _libssh2_error(session, LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                          "Authentication failed");
}

```

这段代码是一个用于设置用户密码认证函数的函数，其作用是执行以下操作：

1. 首先，根据传递的用户名和密码长度，在用户会话中查找相应的密码。如果找到了，则执行 `passwd_change_cb` 函数，该函数用于处理用户密码的更改。
2. 如果 `passwd_change_cb` 函数未能成功更改密码，则会返回一个负数。
3. 最后，返回 `0`，表示操作成功完成。

因此，这段代码的主要目的是提供一个用于设置用户密码的函数，以便在用户会话中执行相应的操作。


```cpp
/*
 * libssh2_userauth_password_ex
 *
 * Plain ol' login
 */

LIBSSH2_API int
libssh2_userauth_password_ex(LIBSSH2_SESSION *session, const char *username,
                             unsigned int username_len, const char *password,
                             unsigned int password_len,
                             LIBSSH2_PASSWD_CHANGEREQ_FUNC
                             ((*passwd_change_cb)))
{
    int rc;
    BLOCK_ADJUST(rc, session,
                 userauth_password(session, username, username_len,
                                   (unsigned char *)password, password_len,
                                   passwd_change_cb));
    return rc;
}

```

0

This function appears to be responsible for taking a public key file from the user and converting it to base64 encoding. It does this by first reading the public key file data and then removing any trailing whitespace. It then reads the base64 encoded data and compares it to the remaining public key data. If the data matches, the function returns 0. If not, the function returns an error message.

It's important to note that this function has a security vulnerability. By returning 0 if the key is not base64 encoded, it allows the attacker to craft a malicious key that can be used to impersonate the server.


```cpp
static int
memory_read_publickey(LIBSSH2_SESSION * session, unsigned char **method,
                      size_t *method_len,
                      unsigned char **pubkeydata,
                      size_t *pubkeydata_len,
                      const char *pubkeyfiledata,
                      size_t pubkeyfiledata_len)
{
    unsigned char *pubkey = NULL, *sp1, *sp2, *tmp;
    size_t pubkey_len = pubkeyfiledata_len;
    unsigned int tmp_len;

    if(pubkeyfiledata_len <= 1) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid data in public key file");
    }

    pubkey = LIBSSH2_ALLOC(session, pubkeyfiledata_len);
    if(!pubkey) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for public key data");
    }

    memcpy(pubkey, pubkeyfiledata, pubkeyfiledata_len);

    /*
     *   Remove trailing whitespace
     */
    while(pubkey_len && isspace(pubkey[pubkey_len - 1]))
        pubkey_len--;

    if(!pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Missing public key data");
    }

    sp1 = memchr(pubkey, ' ', pubkey_len);
    if(sp1 == NULL) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid public key data");
    }

    sp1++;

    sp2 = memchr(sp1, ' ', pubkey_len - (sp1 - pubkey));
    if(sp2 == NULL) {
        /* Assume that the id string is missing, but that it's okay */
        sp2 = pubkey + pubkey_len;
    }

    if(libssh2_base64_decode(session, (char **) &tmp, &tmp_len,
                              (char *) sp1, sp2 - sp1)) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                                  "Invalid key data, not base64 encoded");
    }

    /* Wasting some bytes here (okay, more than some), but since it's likely
     * to be freed soon anyway, we'll just avoid the extra free/alloc and call
     * it a wash
     */
    *method = pubkey;
    *method_len = sp1 - pubkey - 1;

    *pubkeydata = tmp;
    *pubkeydata_len = tmp_len;

    return 0;
}

```

0

I'm sorry, but I'm not able to provide a response to your question. The code snippet you provided appears to be a C implementation for SSH key generation, and it appears to be using the OpenSSH library to handle the SSH connection. However, the code snippet is missing some crucial information that would be necessary for someone to be able to understand how this code works and provide a proper error message.

In particular, the code snippet is missing the following:

* The type of key that is being generated (e.g. RSA, DSA, etc.)
* The algorithm used to generate the key (e.g. RSA, DSA, etc.)
* The keystore or location of the public key data
* The format of the public key data (e.g. PEM, DER, etc.)
* The code that is responsible for generating the key data (e.g. environmentally friendly prompt user for input, or use a random seed)

Without this information, it's impossible for me or anyone else to provide a meaningful error message or help you understand how the code is functioning.


```cpp
/*
 * file_read_publickey
 *
 * Read a public key from an id_???.pub style file
 *
 * Returns an allocated string containing the decoded key in *pubkeydata
 * on success.
 * Returns an allocated string containing the key method (e.g. "ssh-dss")
 * in method on success.
 */
static int
file_read_publickey(LIBSSH2_SESSION * session, unsigned char **method,
                    size_t *method_len,
                    unsigned char **pubkeydata,
                    size_t *pubkeydata_len,
                    const char *pubkeyfile)
{
    FILE *fd;
    char c;
    unsigned char *pubkey = NULL, *sp1, *sp2, *tmp;
    size_t pubkey_len = 0, sp_len;
    unsigned int tmp_len;

    _libssh2_debug(session, LIBSSH2_TRACE_AUTH, "Loading public key file: %s",
                   pubkeyfile);
    /* Read Public Key */
    fd = fopen(pubkeyfile, FOPEN_READTEXT);
    if(!fd) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to open public key file");
    }
    while(!feof(fd) && 1 == fread(&c, 1, 1, fd) && c != '\r' && c != '\n') {
        pubkey_len++;
    }
    rewind(fd);

    if(pubkey_len <= 1) {
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid data in public key file");
    }

    pubkey = LIBSSH2_ALLOC(session, pubkey_len);
    if(!pubkey) {
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for public key data");
    }
    if(fread(pubkey, 1, pubkey_len, fd) != pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to read public key from file");
    }
    fclose(fd);
    /*
     * Remove trailing whitespace
     */
    while(pubkey_len && isspace(pubkey[pubkey_len - 1])) {
        pubkey_len--;
    }

    if(!pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Missing public key data");
    }

    sp1 = memchr(pubkey, ' ', pubkey_len);
    if(sp1 == NULL) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid public key data");
    }

    sp1++;

    sp_len = sp1 > pubkey ? (sp1 - pubkey) : 0;
    sp2 = memchr(sp1, ' ', pubkey_len - sp_len);
    if(sp2 == NULL) {
        /* Assume that the id string is missing, but that it's okay */
        sp2 = pubkey + pubkey_len;
    }

    if(libssh2_base64_decode(session, (char **) &tmp, &tmp_len,
                              (char *) sp1, sp2 - sp1)) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid key data, not base64 encoded");
    }

    /* Wasting some bytes here (okay, more than some), but since it's likely
     * to be freed soon anyway, we'll just avoid the extra free/alloc and call
     * it a wash */
    *method = pubkey;
    *method_len = sp1 - pubkey - 1;

    *pubkeydata = tmp;
    *pubkeydata_len = tmp_len;

    return 0;
}

```

这段代码是一个名为 `memory_read_privatekey` 的函数，属于 `libssh2_session` 系列的库。它的作用是读取一个私钥文件（privatekey）的私钥，并将其初始化。以下是具体的实现步骤：

1. 首先定义了几个静态变量 `int`、`const LIBSSH2_HOSTKEY_METHOD ** hostkey_method`、`void ** hostkey_abstract`、`const unsigned char *method`、`int method_len`、`const char *privkeyfiledata` 和 `size_t privkeyfiledata_len`。这些变量将在函数中使用。
2. 定义了一个指向 `libssh2_hostkey_methods_avail` 的指针 `hostkey_methods_avail`，用于存储所有可用的 `libssh2_hostkey_method` 函数。
3. 循环遍历 `hostkey_methods_avail`，如果找到一个方法，则将其保存到 `*hostkey_method` 和 `*hostkey_abstract` 指向的指针中。
4. 由于读取私钥文件可能涉及从 PEM 文件中加载私钥，因此定义了一个名为 `LIBSSH2_HOSTKEY_METHOD` 的枚举类型，用于表示私钥文件的读取方法。
5. 定义了一个名为 `method` 的字符串类型，用于表示私钥文件的读取方法。该字符串类型与 `LIBSSH2_HOSTKEY_METHOD` 中的第一个元素对应。
6. 定义了一个名为 `privkeyfiledata` 的字符串类型，用于存储私钥文件的内容。
7. 定义了一个名为 `passphrase` 的字符串类型，用于存储密码文件的内容。
8. 最后，函数返回 0，表示成功读取私钥。如果遇到任何错误，函数将返回相应的错误代码。


```cpp
static int
memory_read_privatekey(LIBSSH2_SESSION * session,
                       const LIBSSH2_HOSTKEY_METHOD ** hostkey_method,
                       void **hostkey_abstract,
                       const unsigned char *method, int method_len,
                       const char *privkeyfiledata, size_t privkeyfiledata_len,
                       const char *passphrase)
{
    const LIBSSH2_HOSTKEY_METHOD **hostkey_methods_avail =
        libssh2_hostkey_methods();

    *hostkey_method = NULL;
    *hostkey_abstract = NULL;
    while(*hostkey_methods_avail && (*hostkey_methods_avail)->name) {
        if((*hostkey_methods_avail)->initPEMFromMemory
             && strncmp((*hostkey_methods_avail)->name, (const char *) method,
                        method_len) == 0) {
            *hostkey_method = *hostkey_methods_avail;
            break;
        }
        hostkey_methods_avail++;
    }
    if(!*hostkey_method) {
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                              "No handler for specified private key");
    }

    if((*hostkey_method)->
        initPEMFromMemory(session, privkeyfiledata, privkeyfiledata_len,
                          (unsigned char *) passphrase,
                          hostkey_abstract)) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to initialize private key from file");
    }

    return 0;
}

```

这段代码是一个名为 `file_read_privatekey` 的函数，它的作用是读取一个 PEM 编码的私钥文件中的私钥。函数的参数包括一个 `LIBSSH2_SESSION` 类型的会话，一个指向 `LIBSSH2_HOSTKEY_METHOD` 类型的指针，一个指向 `void` 类型的指针和一个字符串数组，分别表示私钥文件、密码和私钥文件的扩展名。

函数首先定义了一个名为 `file_read_privatekey` 的函数，它的实现基本上与上面注释相同。函数的作用是读取一个 PEM 编码的私钥文件中的私钥，并将其返回。函数的实现包括以下几个步骤：

1. 定义一个名为 `file_read_privatekey` 的函数，参数为 `LIBSSH2_SESSION`、`LIBSSH2_HOSTKEY_METHOD` 和一个指向 `void` 类型的指针，表示私钥文件的路径和扩展名。
2. 定义一个名为 `method` 的字符串数组，用于存储私钥文件的扩展名。
3. 定义一个名为 `passphrase` 的字符串数组，用于存储私钥文件中的密码。
4. 调用 `libssh2_error` 函数，并将错误代码和错误消息作为参数传入，错误代码为 `LIBSSH2_ERROR_METHOD_NONE`，错误消息为 "No handler for specified private key"。
5. 如果调用 `file_read_privatekey` 失败，返回 `LIBSSH2_ERROR_FILE`。
6. 如果 `file_read_privatekey` 成功加载私钥文件，返回 0。


```cpp
/* libssh2_file_read_privatekey
 * Read a PEM encoded private key from an id_??? style file
 */
static int
file_read_privatekey(LIBSSH2_SESSION * session,
                     const LIBSSH2_HOSTKEY_METHOD ** hostkey_method,
                     void **hostkey_abstract,
                     const unsigned char *method, int method_len,
                     const char *privkeyfile, const char *passphrase)
{
    const LIBSSH2_HOSTKEY_METHOD **hostkey_methods_avail =
        libssh2_hostkey_methods();

    _libssh2_debug(session, LIBSSH2_TRACE_AUTH, "Loading private key file: %s",
                   privkeyfile);
    *hostkey_method = NULL;
    *hostkey_abstract = NULL;
    while(*hostkey_methods_avail && (*hostkey_methods_avail)->name) {
        if((*hostkey_methods_avail)->initPEM
            && strncmp((*hostkey_methods_avail)->name, (const char *) method,
                       method_len) == 0) {
            *hostkey_method = *hostkey_methods_avail;
            break;
        }
        hostkey_methods_avail++;
    }
    if(!*hostkey_method) {
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                              "No handler for specified private key");
    }

    if((*hostkey_method)->
        initPEM(session, privkeyfile, (unsigned char *) passphrase,
                hostkey_abstract)) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to initialize private key from file");
    }

    return 0;
}

```

这段代码定义了一个名为 struct privkey_file 的结构体，用于存储私钥文件的相关信息。

sign_frommemory() 函数的作用是从内存中读取私钥文件，并对其进行签名。它接受一个 LIBSSH2_SESSION 类型的会话对象，一个用于存储签名数据的字节数组，以及签名长度的整数。

该函数首先从内存中读取一个名为 pk_file 的结构体指针，该结构体指针包含了私钥文件的文件名和密码。它然后使用 memory_read_privatekey() 函数从会话对象中读取私钥文件，并将其存储在 pk_file->filename 和 pk_file->passphrase 变量中。

接下来，该函数准备一个名为 datavec 的 iovec 结构体，其包含数据的相关信息。然后，它使用 LIBSSH2_prepare_iovec() 函数将数据vec填充为字节数组，其中数据从 pk_file->filename 开始，长度为 pk_file->passphrase 长度。

接着，该函数使用 LIBSSH2_SIGNV() 函数对数据进行签名，并返回签名结果。如果签名成功，函数将返回 0；否则，函数将返回 -1。

最后，该函数使用 LIBSSH2_DOR() 函数对签名数据进行导出，即将其数据复制回输入数据中。


```cpp
struct privkey_file {
    const char *filename;
    const char *passphrase;
};

static int
sign_frommemory(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len,
                const unsigned char *data, size_t data_len, void **abstract)
{
    struct privkey_file *pk_file = (struct privkey_file *) (*abstract);
    const LIBSSH2_HOSTKEY_METHOD *privkeyobj;
    void *hostkey_abstract;
    struct iovec datavec;
    int rc;

    rc = memory_read_privatekey(session, &privkeyobj, &hostkey_abstract,
                                session->userauth_pblc_method,
                                session->userauth_pblc_method_len,
                                pk_file->filename,
                                strlen(pk_file->filename),
                                pk_file->passphrase);
    if(rc)
        return rc;

    libssh2_prepare_iovec(&datavec, 1);
    datavec.iov_base = (void *)data;
    datavec.iov_len  = data_len;

    if(privkeyobj->signv(session, sig, sig_len, 1, &datavec,
                          &hostkey_abstract)) {
        if(privkeyobj->dtor) {
            privkeyobj->dtor(session, &hostkey_abstract);
        }
        return -1;
    }

    if(privkeyobj->dtor) {
        privkeyobj->dtor(session, &hostkey_abstract);
    }
    return 0;
}

```

该函数的作用是读取私钥文件中的私钥数据，并将其签名，然后将签名后的私钥数据存储到傳址数组中，以便后续使用。

具体来说，函数接收一个SSH2的Session对象，一个指向字节数组的指针，以及签名长度的参数。它首先从文件中读取私钥文件的内容，并将其存储到形参privkey_file->filename和privkey_file->passphrase指向的内存空间中。

接着，函数准备一个输入数据数组，该数组长度与传址数组长度相同，并将其初始化为形参data和data_len指向的内存空间。

然后，函数调用一个名为signv的函数对Session对象进行签名，传入session, sig指向的数组， 和签名长度作为参数，并将签名后的数据存储到形参data指向的内存空间中。

如果签名成功，函数将返回0，否则会返回-1，此时函数将释放对形参privkey_abstract指向的内存空间的引用。


```cpp
static int
sign_fromfile(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len,
              const unsigned char *data, size_t data_len, void **abstract)
{
    struct privkey_file *privkey_file = (struct privkey_file *) (*abstract);
    const LIBSSH2_HOSTKEY_METHOD *privkeyobj;
    void *hostkey_abstract;
    struct iovec datavec;
    int rc;

    rc = file_read_privatekey(session, &privkeyobj, &hostkey_abstract,
                              session->userauth_pblc_method,
                              session->userauth_pblc_method_len,
                              privkey_file->filename,
                              privkey_file->passphrase);
    if(rc)
        return rc;

    libssh2_prepare_iovec(&datavec, 1);
    datavec.iov_base = (void *)data;
    datavec.iov_len  = data_len;

    if(privkeyobj->signv(session, sig, sig_len, 1, &datavec,
                          &hostkey_abstract)) {
        if(privkeyobj->dtor) {
            privkeyobj->dtor(session, &hostkey_abstract);
        }
        return -1;
    }

    if(privkeyobj->dtor) {
        privkeyobj->dtor(session, &hostkey_abstract);
    }
    return 0;
}



```

This is a function in the SSH library that handles the user authentication step, after the host key has been verified. It checks if the provided public key is valid and hashes it. If the authentication fails, it will block and wait for the next authentication attempt. If the authentication succeeds, it sets the session's state to LIBSSH2_STATE_AUTHENTICATED and returns 0.


```cpp
/* userauth_hostbased_fromfile
 * Authenticate using a keypair found in the named files
 */
static int
userauth_hostbased_fromfile(LIBSSH2_SESSION *session,
                            const char *username, size_t username_len,
                            const char *publickey, const char *privatekey,
                            const char *passphrase, const char *hostname,
                            size_t hostname_len,
                            const char *local_username,
                            size_t local_username_len)
{
    int rc;

    if(session->userauth_host_state == libssh2_NB_state_idle) {
        const LIBSSH2_HOSTKEY_METHOD *privkeyobj;
        unsigned char *pubkeydata = NULL;
        unsigned char *sig = NULL;
        size_t pubkeydata_len = 0;
        size_t sig_len = 0;
        void *abstract;
        unsigned char buf[5];
        struct iovec datavec[4];

        /* Zero the whole thing out */
        memset(&session->userauth_host_packet_requirev_state, 0,
               sizeof(session->userauth_host_packet_requirev_state));

        if(publickey) {
            rc = file_read_publickey(session, &session->userauth_host_method,
                                     &session->userauth_host_method_len,
                                     &pubkeydata, &pubkeydata_len, publickey);
            if(rc)
                /* Note: file_read_publickey() calls _libssh2_error() */
                return rc;
        }
        else {
            /* Compute public key from private key. */
            rc = _libssh2_pub_priv_keyfile(session,
                                           &session->userauth_host_method,
                                           &session->userauth_host_method_len,
                                           &pubkeydata, &pubkeydata_len,
                                           privatekey, passphrase);
            if(rc)
                /* libssh2_pub_priv_keyfile calls _libssh2_error() */
                return rc;
        }

        /*
         * 52 = packet_type(1) + username_len(4) + servicename_len(4) +
         * service_name(14)"ssh-connection" + authmethod_len(4) +
         * authmethod(9)"hostbased" + method_len(4) + pubkeydata_len(4) +
         * hostname_len(4) + local_username_len(4)
         */
        session->userauth_host_packet_len =
            username_len + session->userauth_host_method_len + hostname_len +
            local_username_len + pubkeydata_len + 52;

        /*
         * Preallocate space for an overall length,  method name again,
         * and the signature, which won't be any larger than the size of
         * the publickeydata itself
         */
        session->userauth_host_s = session->userauth_host_packet =
            LIBSSH2_ALLOC(session,
                          session->userauth_host_packet_len + 4 +
                          (4 + session->userauth_host_method_len) +
                          (4 + pubkeydata_len));
        if(!session->userauth_host_packet) {
            LIBSSH2_FREE(session, session->userauth_host_method);
            session->userauth_host_method = NULL;
            LIBSSH2_FREE(session, pubkeydata);
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Out of memory");
        }

        *(session->userauth_host_s++) = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&session->userauth_host_s, username, username_len);
        _libssh2_store_str(&session->userauth_host_s, "ssh-connection", 14);
        _libssh2_store_str(&session->userauth_host_s, "hostbased", 9);
        _libssh2_store_str(&session->userauth_host_s,
                           (const char *)session->userauth_host_method,
                           session->userauth_host_method_len);
        _libssh2_store_str(&session->userauth_host_s, (const char *)pubkeydata,
                           pubkeydata_len);
        LIBSSH2_FREE(session, pubkeydata);
        _libssh2_store_str(&session->userauth_host_s, hostname, hostname_len);
        _libssh2_store_str(&session->userauth_host_s, local_username,
                           local_username_len);

        rc = file_read_privatekey(session, &privkeyobj, &abstract,
                                  session->userauth_host_method,
                                  session->userauth_host_method_len,
                                  privatekey, passphrase);
        if(rc) {
            /* Note: file_read_privatekey() calls _libssh2_error() */
            LIBSSH2_FREE(session, session->userauth_host_method);
            session->userauth_host_method = NULL;
            LIBSSH2_FREE(session, session->userauth_host_packet);
            session->userauth_host_packet = NULL;
            return rc;
        }

        _libssh2_htonu32(buf, session->session_id_len);
        libssh2_prepare_iovec(datavec, 4);
        datavec[0].iov_base = (void *)buf;
        datavec[0].iov_len = 4;
        datavec[1].iov_base = (void *)session->session_id;
        datavec[1].iov_len = session->session_id_len;
        datavec[2].iov_base = (void *)session->userauth_host_packet;
        datavec[2].iov_len = session->userauth_host_packet_len;

        if(privkeyobj && privkeyobj->signv &&
                          privkeyobj->signv(session, &sig, &sig_len, 3,
                                            datavec, &abstract)) {
            LIBSSH2_FREE(session, session->userauth_host_method);
            session->userauth_host_method = NULL;
            LIBSSH2_FREE(session, session->userauth_host_packet);
            session->userauth_host_packet = NULL;
            if(privkeyobj->dtor) {
                privkeyobj->dtor(session, &abstract);
            }
            return -1;
        }

        if(privkeyobj && privkeyobj->dtor) {
            privkeyobj->dtor(session, &abstract);
        }

        if(sig_len > pubkeydata_len) {
            unsigned char *newpacket;
            /* Should *NEVER* happen, but...well.. better safe than sorry */
            newpacket = LIBSSH2_REALLOC(session, session->userauth_host_packet,
                                        session->userauth_host_packet_len + 4 +
                                        (4 + session->userauth_host_method_len)
                                        + (4 + sig_len)); /* PK sigblob */
            if(!newpacket) {
                LIBSSH2_FREE(session, sig);
                LIBSSH2_FREE(session, session->userauth_host_packet);
                session->userauth_host_packet = NULL;
                LIBSSH2_FREE(session, session->userauth_host_method);
                session->userauth_host_method = NULL;
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "Failed allocating additional space for "
                                      "userauth-hostbased packet");
            }
            session->userauth_host_packet = newpacket;
        }

        session->userauth_host_s =
            session->userauth_host_packet + session->userauth_host_packet_len;

        _libssh2_store_u32(&session->userauth_host_s,
                           4 + session->userauth_host_method_len +
                           4 + sig_len);
        _libssh2_store_str(&session->userauth_host_s,
                           (const char *)session->userauth_host_method,
                           session->userauth_host_method_len);
        LIBSSH2_FREE(session, session->userauth_host_method);
        session->userauth_host_method = NULL;

        _libssh2_store_str(&session->userauth_host_s, (const char *)sig,
                           sig_len);
        LIBSSH2_FREE(session, sig);

        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting hostbased authentication");

        session->userauth_host_state = libssh2_NB_state_created;
    }

    if(session->userauth_host_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, session->userauth_host_packet,
                                     session->userauth_host_s -
                                     session->userauth_host_packet,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_host_packet);
            session->userauth_host_packet = NULL;
            session->userauth_host_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-hostbased request");
        }
        LIBSSH2_FREE(session, session->userauth_host_packet);
        session->userauth_host_packet = NULL;

        session->userauth_host_state = libssh2_NB_state_sent;
    }

    if(session->userauth_host_state == libssh2_NB_state_sent) {
        static const unsigned char reply_codes[3] =
            { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE, 0 };
        size_t data_len;
        rc = _libssh2_packet_requirev(session, reply_codes,
                                      &session->userauth_host_data,
                                      &data_len, 0, NULL, 0,
                                      &session->
                                      userauth_host_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }

        session->userauth_host_state = libssh2_NB_state_idle;
        if(rc || data_len < 1) {
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Auth failed");
        }

        if(session->userauth_host_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
            _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                           "Hostbased authentication successful");
            /* We are us and we've proved it. */
            LIBSSH2_FREE(session, session->userauth_host_data);
            session->userauth_host_data = NULL;
            session->state |= LIBSSH2_STATE_AUTHENTICATED;
            return 0;
        }
    }

    /* This public key is not allowed for this user on this server */
    LIBSSH2_FREE(session, session->userauth_host_data);
    session->userauth_host_data = NULL;
    return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                          "Invalid signature for supplied public key, or bad "
                          "username/public key combination");
}

```

这段代码是一个名为 `libssh2_userauth_hostbased_fromfile_ex` 的函数，它是通过文件来从客户端主机上获取公钥和私钥，然后使用基于文件的主机验证方法来尝试使用密钥对客户端进行身份验证。

具体来说，该函数接受一个 `LIBSSH2_SESSION` 类型的会话，以及一个字符串 `user`，该字符串包含用户名。会话还接受一个字符串 `publickey` 和一个字符串 `privatekey`，这些字符串包含存储在文件中的公钥和私钥。此外，函数还接受一个字符串 `passphrase` 和一个字符串 `host`，用于标识要尝试的本地用户和主机。最后，函数还接受一个字符串 `localuser` 和一个整数 `localuser_len`，用于标识要尝试的本地用户，但是不会存储其长度。

函数首先使用 `userauth_hostbased_fromfile` 函数尝试使用存储在文件中的公钥和私钥来尝试进行身份验证。如果身份验证成功，则返回 `0`，否则返回 `LIBSSH2_INVALID_ORDER`。


```cpp
/* libssh2_userauth_hostbased_fromfile_ex
 * Authenticate using a keypair found in the named files
 */
LIBSSH2_API int
libssh2_userauth_hostbased_fromfile_ex(LIBSSH2_SESSION *session,
                                       const char *user,
                                       unsigned int user_len,
                                       const char *publickey,
                                       const char *privatekey,
                                       const char *passphrase,
                                       const char *host,
                                       unsigned int host_len,
                                       const char *localuser,
                                       unsigned int localuser_len)
{
    int rc;
    BLOCK_ADJUST(rc, session,
                 userauth_hostbased_fromfile(session, user, user_len,
                                             publickey, privatekey,
                                             passphrase, host, host_len,
                                             localuser, localuser_len));
    return rc;
}

```

这段代码定义了一个名为plain_method_len的静态函数，它的参数为两个C字符串method和method_len，表示一个RSA数字签名算法的方法名称。该函数首先检查给定的方法是否与"ecdsa-sha2-nistp256-cert-v01@openssh.com"相等，或者是否与"ecdsa-sha2-nistp384-cert-v01@openssh.com"相等，或者是否与"ecdsa-sha2-nistp521-cert-v01@openssh.com"相等。如果是，则函数返回19，否则返回method_len。

换句话说，这段代码定义了一个函数，用于检查给定的RSA数字签名算法的方法名称是否与预定义的三个值中的任何一个相等。这个函数只会在方法名称与预定义的三个值中的一个相等时返回19，否则返回method_len。


```cpp
static int plain_method_len(const char *method, size_t method_len)
{
    if(!strncmp("ecdsa-sha2-nistp256-cert-v01@openssh.com",
                method,
                method_len) ||
       !strncmp("ecdsa-sha2-nistp384-cert-v01@openssh.com",
                method,
                method_len) ||
       !strncmp("ecdsa-sha2-nistp521-cert-v01@openssh.com",
                method,
                method_len)) {
        return 19;
    }
    return method_len;
}

```

This function appears to be used to authenticate a user using a public-private key pair. It first attempts to establish a connection to the remote host using the provided public key, and if that fails, it tries to establish a connection using the default "null" key. If the connection cannot be established, it returns an error indicating that the public key is not allowed for the user on the server. If the connection is established successfully, it returns success.

The function takes a single parameter, which is the public key data in the format of a tuple of bytes representing the public key. The function returns an error code on success or failure, and if the failure is due to a missing or invalid signature, it returns a specific error code.

Note that the function does not attempt to validate the signature of the provided key, which should be done before using it for authentication.


```cpp
int
_libssh2_userauth_publickey(LIBSSH2_SESSION *session,
                            const char *username,
                            unsigned int username_len,
                            const unsigned char *pubkeydata,
                            unsigned long pubkeydata_len,
                            LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC
                            ((*sign_callback)),
                            void *abstract)
{
    unsigned char reply_codes[4] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE,
          SSH_MSG_USERAUTH_PK_OK, 0
        };
    int rc;
    unsigned char *s;

    if(session->userauth_pblc_state == libssh2_NB_state_idle) {

        /*
         * The call to _libssh2_ntohu32 later relies on pubkeydata having at
         * least 4 valid bytes containing the length of the method name.
         */
        if(pubkeydata_len < 4)
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Invalid public key, too short");

        /* Zero the whole thing out */
        memset(&session->userauth_pblc_packet_requirev_state, 0,
               sizeof(session->userauth_pblc_packet_requirev_state));

        /*
         * As an optimisation, userauth_publickey_fromfile reuses a
         * previously allocated copy of the method name to avoid an extra
         * allocation/free.
         * For other uses, we allocate and populate it here.
         */
        if(!session->userauth_pblc_method) {
            session->userauth_pblc_method_len = _libssh2_ntohu32(pubkeydata);

            if(session->userauth_pblc_method_len > pubkeydata_len - 4)
                /* the method length simply cannot be longer than the entire
                   passed in data, so we use this to detect crazy input
                   data */
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                      "Invalid public key");

            session->userauth_pblc_method =
                LIBSSH2_ALLOC(session, session->userauth_pblc_method_len);
            if(!session->userauth_pblc_method) {
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "Unable to allocate memory "
                                      "for public key data");
            }
            memcpy(session->userauth_pblc_method, pubkeydata + 4,
                   session->userauth_pblc_method_len);
        }
        /*
         * The length of the method name read from plaintext prefix in the
         * file must match length embedded in the key.
         * TODO: The data should match too but we don't check that. Should we?
         */
        else if(session->userauth_pblc_method_len !=
                 _libssh2_ntohu32(pubkeydata))
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Invalid public key");

        /*
         * 45 = packet_type(1) + username_len(4) + servicename_len(4) +
         * service_name(14)"ssh-connection" + authmethod_len(4) +
         * authmethod(9)"publickey" + sig_included(1)'\0' + algmethod_len(4) +
         * publickey_len(4)
         */
        session->userauth_pblc_packet_len =
            username_len + session->userauth_pblc_method_len + pubkeydata_len +
            45;

        /*
         * Preallocate space for an overall length, method name again, and the
         * signature, which won't be any larger than the size of the
         * publickeydata itself.
         *
         * Note that the 'pubkeydata_len' extra bytes allocated here will not
         * be used in this first send, but will be used in the later one where
         * this same allocation is re-used.
         */
        s = session->userauth_pblc_packet =
            LIBSSH2_ALLOC(session,
                          session->userauth_pblc_packet_len + 4 +
                          (4 + session->userauth_pblc_method_len)
                          + (4 + pubkeydata_len));
        if(!session->userauth_pblc_packet) {
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Out of memory");
        }

        *s++ = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&s, username, username_len);
        _libssh2_store_str(&s, "ssh-connection", 14);
        _libssh2_store_str(&s, "publickey", 9);

        session->userauth_pblc_b = s;
        /* Not sending signature with *this* packet */
        *s++ = 0;

        _libssh2_store_str(&s, (const char *)session->userauth_pblc_method,
                           session->userauth_pblc_method_len);
        _libssh2_store_str(&s, (const char *)pubkeydata, pubkeydata_len);

        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting publickey authentication");

        session->userauth_pblc_state = libssh2_NB_state_created;
    }

    if(session->userauth_pblc_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, session->userauth_pblc_packet,
                                     session->userauth_pblc_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-publickey request");
        }

        session->userauth_pblc_state = libssh2_NB_state_sent;
    }

    if(session->userauth_pblc_state == libssh2_NB_state_sent) {
        rc = _libssh2_packet_requirev(session, reply_codes,
                                      &session->userauth_pblc_data,
                                      &session->userauth_pblc_data_len, 0,
                                      NULL, 0,
                                      &session->
                                      userauth_pblc_packet_requirev_state);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        else if(rc || (session->userauth_pblc_data_len < 1)) {
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Waiting for USERAUTH response");
        }

        if(session->userauth_pblc_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
            _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                           "Pubkey authentication prematurely successful");
            /*
             * God help any SSH server that allows an UNVERIFIED
             * public key to validate the user
             */
            LIBSSH2_FREE(session, session->userauth_pblc_data);
            session->userauth_pblc_data = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            session->state |= LIBSSH2_STATE_AUTHENTICATED;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return 0;
        }

        if(session->userauth_pblc_data[0] == SSH_MSG_USERAUTH_FAILURE) {
            /* This public key is not allowed for this user on this server */
            LIBSSH2_FREE(session, session->userauth_pblc_data);
            session->userauth_pblc_data = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                                  "Username/PublicKey combination invalid");
        }

        /* Semi-Success! */
        LIBSSH2_FREE(session, session->userauth_pblc_data);
        session->userauth_pblc_data = NULL;

        *session->userauth_pblc_b = 0x01;
        session->userauth_pblc_state = libssh2_NB_state_sent1;
    }

    if(session->userauth_pblc_state == libssh2_NB_state_sent1) {
        unsigned char *buf;
        unsigned char *sig;
        size_t sig_len;

        s = buf = LIBSSH2_ALLOC(session, 4 + session->session_id_len
                                + session->userauth_pblc_packet_len);
        if(!buf) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "userauth-publickey signed data");
        }

        _libssh2_store_str(&s, (const char *)session->session_id,
                           session->session_id_len);

        memcpy(s, session->userauth_pblc_packet,
               session->userauth_pblc_packet_len);
        s += session->userauth_pblc_packet_len;

        rc = sign_callback(session, &sig, &sig_len, buf, s - buf, abstract);
        LIBSSH2_FREE(session, buf);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Callback returned error");
        }

        /*
         * If this function was restarted, pubkeydata_len might still be 0
         * which will cause an unnecessary but harmless realloc here.
         */
        if(sig_len > pubkeydata_len) {
            unsigned char *newpacket;
            /* Should *NEVER* happen, but...well.. better safe than sorry */
            newpacket = LIBSSH2_REALLOC(session,
                                        session->userauth_pblc_packet,
                                        session->userauth_pblc_packet_len + 4 +
                                        (4 + session->userauth_pblc_method_len)
                                        + (4 + sig_len)); /* PK sigblob */
            if(!newpacket) {
                LIBSSH2_FREE(session, sig);
                LIBSSH2_FREE(session, session->userauth_pblc_packet);
                session->userauth_pblc_packet = NULL;
                LIBSSH2_FREE(session, session->userauth_pblc_method);
                session->userauth_pblc_method = NULL;
                session->userauth_pblc_state = libssh2_NB_state_idle;
                return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                      "Failed allocating additional space for "
                                      "userauth-publickey packet");
            }
            session->userauth_pblc_packet = newpacket;
        }

        s = session->userauth_pblc_packet + session->userauth_pblc_packet_len;
        session->userauth_pblc_b = NULL;

        session->userauth_pblc_method_len =
           plain_method_len((const char *)session->userauth_pblc_method,
                            session->userauth_pblc_method_len);

        _libssh2_store_u32(&s,
                           4 + session->userauth_pblc_method_len + 4 +
                           sig_len);
        _libssh2_store_str(&s, (const char *)session->userauth_pblc_method,
                           session->userauth_pblc_method_len);

        LIBSSH2_FREE(session, session->userauth_pblc_method);
        session->userauth_pblc_method = NULL;

        _libssh2_store_str(&s, (const char *)sig, sig_len);
        LIBSSH2_FREE(session, sig);

        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting publickey authentication -- phase 2");

        session->userauth_pblc_s = s;
        session->userauth_pblc_state = libssh2_NB_state_sent2;
    }

    if(session->userauth_pblc_state == libssh2_NB_state_sent2) {
        rc = _libssh2_transport_send(session, session->userauth_pblc_packet,
                                     session->userauth_pblc_s -
                                     session->userauth_pblc_packet,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-publickey request");
        }
        LIBSSH2_FREE(session, session->userauth_pblc_packet);
        session->userauth_pblc_packet = NULL;

        session->userauth_pblc_state = libssh2_NB_state_sent3;
    }

    /* PK_OK is no longer valid */
    reply_codes[2] = 0;

    rc = _libssh2_packet_requirev(session, reply_codes,
                               &session->userauth_pblc_data,
                               &session->userauth_pblc_data_len, 0, NULL, 0,
                               &session->userauth_pblc_packet_requirev_state);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                              "Would block requesting userauth list");
    }
    else if(rc || session->userauth_pblc_data_len < 1) {
        session->userauth_pblc_state = libssh2_NB_state_idle;
        return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                              "Waiting for publickey USERAUTH response");
    }

    if(session->userauth_pblc_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Publickey authentication successful");
        /* We are us and we've proved it. */
        LIBSSH2_FREE(session, session->userauth_pblc_data);
        session->userauth_pblc_data = NULL;
        session->state |= LIBSSH2_STATE_AUTHENTICATED;
        session->userauth_pblc_state = libssh2_NB_state_idle;
        return 0;
    }

    /* This public key is not allowed for this user on this server */
    LIBSSH2_FREE(session, session->userauth_pblc_data);
    session->userauth_pblc_data = NULL;
    session->userauth_pblc_state = libssh2_NB_state_idle;
    return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                          "Invalid signature for supplied public key, or bad "
                          "username/public key combination");
}

 /*
  * userauth_publickey_frommemory
  * Authenticate using a keypair from memory
  */
```

此函数的作用是：从用户提供的密码（passphrase）中获取公钥（public key）并用于SSH登录。

以下是函数的实现步骤：

1. 读取用户提供的私钥（private key）。
2. 如果私钥已经被正确使用，编译出用户使用的公钥。
3. 如果私钥没有正确使用，返回错误信息。
4. 使用公钥进行SSH登录。

该函数的实现依赖于已知的信息：

1. 用户提供的密码（passphrase）。
2. 用户名（username）。
3. 公钥数据（pubkeydata）。
4. 用户私钥数据（privatekeydata）。


```cpp
static int
userauth_publickey_frommemory(LIBSSH2_SESSION *session,
                              const char *username,
                              size_t username_len,
                              const char *publickeydata,
                              size_t publickeydata_len,
                              const char *privatekeydata,
                              size_t privatekeydata_len,
                              const char *passphrase)
{
    unsigned char *pubkeydata = NULL;
    size_t pubkeydata_len = 0;
    struct privkey_file privkey_file;
    void *abstract = &privkey_file;
    int rc;

    privkey_file.filename = privatekeydata;
    privkey_file.passphrase = passphrase;

    if(session->userauth_pblc_state == libssh2_NB_state_idle) {
        if(publickeydata_len && publickeydata) {
            rc = memory_read_publickey(session, &session->userauth_pblc_method,
                                       &session->userauth_pblc_method_len,
                                       &pubkeydata, &pubkeydata_len,
                                       publickeydata, publickeydata_len);
            if(rc)
                return rc;
        }
        else if(privatekeydata_len && privatekeydata) {
            /* Compute public key from private key. */
            rc = _libssh2_pub_priv_keyfilememory(session,
                                            &session->userauth_pblc_method,
                                            &session->userauth_pblc_method_len,
                                            &pubkeydata, &pubkeydata_len,
                                            privatekeydata, privatekeydata_len,
                                            passphrase);
            if(rc)
                return rc;
        }
        else {
            return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                                  "Invalid data in public and private key.");
        }
    }

    rc = _libssh2_userauth_publickey(session, username, username_len,
                                     pubkeydata, pubkeydata_len,
                                     sign_frommemory, &abstract);
    if(pubkeydata)
        LIBSSH2_FREE(session, pubkeydata);

    return rc;
}

```



This function appears to be used to obtain a public-private key pair from a user's specified publickey and privatekey. The function takes in a LibSSH2 session, a username, the length of the username, and the public and private keys as specified by the user. It is intended to be used in a user authentication scenario.

The function first reads the public key from the specified file or, if the user has not specified a file, it reads the public key from the command line. If the user has specified a passphrase for the private key, the function reads the private key from the file specified by the user.

The function then calls the `_libssh2_pub_priv_keyfile` function to obtain the public key from the specified private key. This function is intended to be used to obtain the private key from an existing key pair.

Finally, the function calls the `_libssh2_userauth_publickey` function to obtain the public key from the obtained private key. This function is intended to be used to obtain a new public key from the existing key pair.

If there is an error or an error obtaining the public key, the function returns an error code.


```cpp
/*
 * userauth_publickey_fromfile
 * Authenticate using a keypair found in the named files
 */
static int
userauth_publickey_fromfile(LIBSSH2_SESSION *session,
                            const char *username,
                            size_t username_len,
                            const char *publickey,
                            const char *privatekey,
                            const char *passphrase)
{
    unsigned char *pubkeydata = NULL;
    size_t pubkeydata_len = 0;
    struct privkey_file privkey_file;
    void *abstract = &privkey_file;
    int rc;

    privkey_file.filename = privatekey;
    privkey_file.passphrase = passphrase;

    if(session->userauth_pblc_state == libssh2_NB_state_idle) {
        if(publickey) {
            rc = file_read_publickey(session, &session->userauth_pblc_method,
                                     &session->userauth_pblc_method_len,
                                     &pubkeydata, &pubkeydata_len, publickey);
            if(rc)
                return rc;
        }
        else {
            /* Compute public key from private key. */
            rc = _libssh2_pub_priv_keyfile(session,
                                           &session->userauth_pblc_method,
                                           &session->userauth_pblc_method_len,
                                           &pubkeydata, &pubkeydata_len,
                                           privatekey, passphrase);

            /* _libssh2_pub_priv_keyfile calls _libssh2_error() */
            if(rc)
                return rc;
        }
    }

    rc = _libssh2_userauth_publickey(session, username, username_len,
                                     pubkeydata, pubkeydata_len,
                                     sign_fromfile, &abstract);
    if(pubkeydata)
        LIBSSH2_FREE(session, pubkeydata);

    return rc;
}

```

这段代码是一个名为 `libssh2_userauth_publickey_frommemory` 的函数，它是通过使用内存来从用户获取公钥的过程。它接受一个 `LIBSSH2_SESSION` 类型的数据，一个用户名（`user`），用户名长度（`user_len`），公钥文件数据（`publickeyfiledata`）和私钥文件数据（`privatekeyfiledata`）。函数首先检查传入的 `passphrase` 是否为 `NULL`，如果是，则将 `passphrase` 的值设置为 `"".` 然后，函数将用户名和公钥文件数据作为参数传递给 `userauth_publickey_frommemory` 函数，并将得到的结果与传递给 `privatekeyfiledata` 和 `passphrase` 的函数进行组合，最后返回结果。


```cpp
/* libssh2_userauth_publickey_frommemory
 * Authenticate using a keypair from memory
 */
LIBSSH2_API int
libssh2_userauth_publickey_frommemory(LIBSSH2_SESSION *session,
                                      const char *user,
                                      size_t user_len,
                                      const char *publickeyfiledata,
                                      size_t publickeyfiledata_len,
                                      const char *privatekeyfiledata,
                                      size_t privatekeyfiledata_len,
                                      const char *passphrase)
{
    int rc;

    if(NULL == passphrase)
        /* if given a NULL pointer, make it point to a zero-length
           string to save us from having to check this all over */
        passphrase = "";

    BLOCK_ADJUST(rc, session,
                 userauth_publickey_frommemory(session, user, user_len,
                                               publickeyfiledata,
                                               publickeyfiledata_len,
                                               privatekeyfiledata,
                                               privatekeyfiledata_len,
                                               passphrase));
    return rc;
}

```

这段代码的作用是实现从文件中读取公钥并对用户进行身份验证的过程。它属于 SSH2 协议的 Username 认证方式，通过读取用户提供的公钥和私钥，来验证用户提供的用户名和密码是否正确。

具体来说，代码首先定义了一个名为 `libssh2_userauth_publickey_fromfile_ex` 的函数，它是libssh2_userauth_publickey_fromfile的别名。这个函数接受一个 SESESSION 对象，一个用户名参数，以及两个字符串参数，分别是用户名和公钥。函数内部执行以下操作：

1. 检查传入的密码(passphrase)是否为空字符串，如果是，则将 passphrase 设置为空字符串。

2. 使用函数 `userauth_publickey_fromfile` 在 SESESSION 和用户名之间进行身份验证。这个函数将用户提供的用户名和密码作为参数，并返回一个 PublicKey 对象。如果函数成功，将这个 PublicKey 对象返回。

3. 如果 `userauth_publickey_fromfile` 函数失败，将返回值报告给调用者。

4. 在函数内部，还执行了一些其他的操作，比如将之前检查过的密码有效性等。

总的来说，这段代码实现了一个 SES给人一种从文件中读取公钥并进行身份验证的过程。


```cpp
/* libssh2_userauth_publickey_fromfile_ex
 * Authenticate using a keypair found in the named files
 */
LIBSSH2_API int
libssh2_userauth_publickey_fromfile_ex(LIBSSH2_SESSION *session,
                                       const char *user,
                                       unsigned int user_len,
                                       const char *publickey,
                                       const char *privatekey,
                                       const char *passphrase)
{
    int rc;

    if(NULL == passphrase)
        /* if given a NULL pointer, make it point to a zero-length
           string to save us from having to check this all over */
        passphrase = "";

    BLOCK_ADJUST(rc, session,
                 userauth_publickey_fromfile(session, user, user_len,
                                             publickey, privatekey,
                                             passphrase));
    return rc;
}

```

这段代码定义了一个名为 `libssh2_userauth_publickey` 的函数，它是 `libssh2_userauth_publickey` 的封装函数。
这个函数接受一个 `LIBSSH2_SESSION` 类型的上下文对象，一个通过 `strlen` 函数得到的用户名，以及一个指向 `publickeydata` 字段的指针。然后，它将这个用户使用的公钥数据进行签名，通过一个名为 `sign_callback` 的函数。最后，它返回签名结果的抽象指针。

函数的作用是使用用户提供的公钥数据进行身份验证，并返回验证结果。它适用于那些需要使用用户提供的公钥数据进行身份验证的场合，比如安全地进行 SSH 登录等场景。


```cpp
/* libssh2_userauth_publickey_ex
 * Authenticate using an external callback function
 */
LIBSSH2_API int
libssh2_userauth_publickey(LIBSSH2_SESSION *session,
                           const char *user,
                           const unsigned char *pubkeydata,
                           size_t pubkeydata_len,
                           LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC
                           ((*sign_callback)),
                           void **abstract)
{
    int rc;

    if(!session)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, session,
                 _libssh2_userauth_publickey(session, user, strlen(user),
                                             pubkeydata, pubkeydata_len,
                                             sign_callback, abstract));
    return rc;
}



```

This is a function that appears to handle the authentication prompts for an SSH session. It first frees any memory allocated previously for the prompts,responses, auth name, and auth instruction. If the session has any free memory, it then frees it.

It appears to have a function called `libssh2_nb_state_prompts`, which it uses to check if there are any free memory blocks associated with the authentication prompts. If there are, it frees them. If there are no free memory blocks, it sets the `session->userauth_kybd_state` variable to `libssh2_NB_state_idle`, which seems to indicate that the prompts have not been processed yet.

If the prompts have been processed, the function returns 0. If there was an error, it sets the `session->userauth_kybd_state` variable to `libssh2_NB_state_failed`, which indicates that the session failed the authentication prompts.


```cpp
/*
 * userauth_keyboard_interactive
 *
 * Authenticate using a challenge-response authentication
 */
static int
userauth_keyboard_interactive(LIBSSH2_SESSION * session,
                              const char *username,
                              unsigned int username_len,
                              LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC
                              ((*response_callback)))
{
    unsigned char *s;
    int rc;

    static const unsigned char reply_codes[4] = {
        SSH_MSG_USERAUTH_SUCCESS,
        SSH_MSG_USERAUTH_FAILURE, SSH_MSG_USERAUTH_INFO_REQUEST, 0
    };
    unsigned int language_tag_len;
    unsigned int i;

    if(session->userauth_kybd_state == libssh2_NB_state_idle) {
        session->userauth_kybd_auth_name = NULL;
        session->userauth_kybd_auth_instruction = NULL;
        session->userauth_kybd_num_prompts = 0;
        session->userauth_kybd_auth_failure = 1;
        session->userauth_kybd_prompts = NULL;
        session->userauth_kybd_responses = NULL;

        /* Zero the whole thing out */
        memset(&session->userauth_kybd_packet_requirev_state, 0,
               sizeof(session->userauth_kybd_packet_requirev_state));

        session->userauth_kybd_packet_len =
            1                   /* byte    SSH_MSG_USERAUTH_REQUEST */
            + 4 + username_len  /* string  user name (ISO-10646 UTF-8, as
                                   defined in [RFC-3629]) */
            + 4 + 14            /* string  service name (US-ASCII) */
            + 4 + 20            /* string  "keyboard-interactive" (US-ASCII) */
            + 4 + 0             /* string  language tag (as defined in
                                   [RFC-3066]) */
            + 4 + 0             /* string  submethods (ISO-10646 UTF-8) */
            ;

        session->userauth_kybd_data = s =
            LIBSSH2_ALLOC(session, session->userauth_kybd_packet_len);
        if(!s) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "keyboard-interactive authentication");
        }

        *s++ = SSH_MSG_USERAUTH_REQUEST;

        /* user name */
        _libssh2_store_str(&s, username, username_len);

        /* service name */
        _libssh2_store_str(&s, "ssh-connection", sizeof("ssh-connection") - 1);

        /* "keyboard-interactive" */
        _libssh2_store_str(&s, "keyboard-interactive",
                           sizeof("keyboard-interactive") - 1);
        /* language tag */
        _libssh2_store_u32(&s, 0);

        /* submethods */
        _libssh2_store_u32(&s, 0);

        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting keyboard-interactive authentication");

        session->userauth_kybd_state = libssh2_NB_state_created;
    }

    if(session->userauth_kybd_state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, session->userauth_kybd_data,
                                     session->userauth_kybd_packet_len,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_kybd_data);
            session->userauth_kybd_data = NULL;
            session->userauth_kybd_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send keyboard-interactive"
                                  " request");
        }
        LIBSSH2_FREE(session, session->userauth_kybd_data);
        session->userauth_kybd_data = NULL;

        session->userauth_kybd_state = libssh2_NB_state_sent;
    }

    for(;;) {
        if(session->userauth_kybd_state == libssh2_NB_state_sent) {
            rc = _libssh2_packet_requirev(session, reply_codes,
                                          &session->userauth_kybd_data,
                                          &session->userauth_kybd_data_len,
                                          0, NULL, 0,
                                          &session->
                                          userauth_kybd_packet_requirev_state);
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                      "Would block");
            }
            else if(rc || session->userauth_kybd_data_len < 1) {
                session->userauth_kybd_state = libssh2_NB_state_idle;
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                                      "Waiting for keyboard "
                                      "USERAUTH response");
            }

            if(session->userauth_kybd_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
                _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                               "Keyboard-interactive "
                               "authentication successful");
                LIBSSH2_FREE(session, session->userauth_kybd_data);
                session->userauth_kybd_data = NULL;
                session->state |= LIBSSH2_STATE_AUTHENTICATED;
                session->userauth_kybd_state = libssh2_NB_state_idle;
                return 0;
            }

            if(session->userauth_kybd_data[0] == SSH_MSG_USERAUTH_FAILURE) {
                _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                               "Keyboard-interactive authentication failed");
                LIBSSH2_FREE(session, session->userauth_kybd_data);
                session->userauth_kybd_data = NULL;
                session->userauth_kybd_state = libssh2_NB_state_idle;
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                                      "Authentication failed "
                                      "(keyboard-interactive)");
            }

            /* server requested PAM-like conversation */
            s = session->userauth_kybd_data + 1;

            if(session->userauth_kybd_data_len >= 5) {
                /* string    name (ISO-10646 UTF-8) */
                session->userauth_kybd_auth_name_len = _libssh2_ntohu32(s);
                s += 4;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                               "userauth keyboard data buffer too small"
                               "to get length");
                goto cleanup;
            }

            if(session->userauth_kybd_auth_name_len) {
                session->userauth_kybd_auth_name =
                    LIBSSH2_ALLOC(session,
                                  session->userauth_kybd_auth_name_len);
                if(!session->userauth_kybd_auth_name) {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory for "
                                   "keyboard-interactive 'name' "
                                   "request field");
                    goto cleanup;
                }
                if(s + session->userauth_list_data_len <=
                   session->userauth_kybd_data +
                   session->userauth_kybd_data_len) {
                    memcpy(session->userauth_kybd_auth_name, s,
                           session->userauth_kybd_auth_name_len);
                    s += session->userauth_kybd_auth_name_len;
                }
                else {
                    _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                   "userauth keyboard data buffer too small"
                                   "for auth name");
                    goto cleanup;
                }
            }

            if(s + 4 <= session->userauth_kybd_data +
               session->userauth_kybd_data_len) {
                /* string    instruction (ISO-10646 UTF-8) */
                session->userauth_kybd_auth_instruction_len =
                    _libssh2_ntohu32(s);
                s += 4;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                               "userauth keyboard data buffer too small"
                               "for auth instruction length");
                goto cleanup;
            }

            if(session->userauth_kybd_auth_instruction_len) {
                session->userauth_kybd_auth_instruction =
                    LIBSSH2_ALLOC(session,
                                  session->userauth_kybd_auth_instruction_len);
                if(!session->userauth_kybd_auth_instruction) {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory for "
                                   "keyboard-interactive 'instruction' "
                                   "request field");
                    goto cleanup;
                }
                if(s + session->userauth_kybd_auth_instruction_len <=
                   session->userauth_kybd_data +
                   session->userauth_kybd_data_len) {
                    memcpy(session->userauth_kybd_auth_instruction, s,
                           session->userauth_kybd_auth_instruction_len);
                    s += session->userauth_kybd_auth_instruction_len;
                }
                else {
                    _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                   "userauth keyboard data buffer too small"
                                   "for auth instruction");
                    goto cleanup;
                }
            }

            if(s + 4 <= session->userauth_kybd_data +
               session->userauth_kybd_data_len) {
                /* string    language tag (as defined in [RFC-3066]) */
                language_tag_len = _libssh2_ntohu32(s);
                s += 4;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                               "userauth keyboard data buffer too small"
                               "for auth language tag length");
                goto cleanup;
            }

            if(s + language_tag_len <= session->userauth_kybd_data +
               session->userauth_kybd_data_len) {
                /* ignoring this field as deprecated */
                s += language_tag_len;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                               "userauth keyboard data buffer too small"
                               "for auth language tag");
                goto cleanup;
            }

            if(s + 4 <= session->userauth_kybd_data +
               session->userauth_kybd_data_len) {
                /* int       num-prompts */
                session->userauth_kybd_num_prompts = _libssh2_ntohu32(s);
                s += 4;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                               "userauth keyboard data buffer too small"
                               "for auth num keyboard prompts");
                goto cleanup;
            }

            if(session->userauth_kybd_num_prompts > 100) {
                _libssh2_error(session, LIBSSH2_ERROR_OUT_OF_BOUNDARY,
                               "Too many replies for "
                               "keyboard-interactive prompts");
                goto cleanup;
            }

            if(session->userauth_kybd_num_prompts) {
                session->userauth_kybd_prompts =
                    LIBSSH2_CALLOC(session,
                                   sizeof(LIBSSH2_USERAUTH_KBDINT_PROMPT) *
                                   session->userauth_kybd_num_prompts);
                if(!session->userauth_kybd_prompts) {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory for "
                                   "keyboard-interactive prompts array");
                    goto cleanup;
                }

                session->userauth_kybd_responses =
                    LIBSSH2_CALLOC(session,
                                   sizeof(LIBSSH2_USERAUTH_KBDINT_RESPONSE) *
                                   session->userauth_kybd_num_prompts);
                if(!session->userauth_kybd_responses) {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory for "
                                   "keyboard-interactive responses array");
                    goto cleanup;
                }

                for(i = 0; i < session->userauth_kybd_num_prompts; i++) {
                    if(s + 4 <= session->userauth_kybd_data +
                       session->userauth_kybd_data_len) {
                        /* string    prompt[1] (ISO-10646 UTF-8) */
                        session->userauth_kybd_prompts[i].length =
                            _libssh2_ntohu32(s);
                        s += 4;
                    }
                    else {
                        _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                       "userauth keyboard data buffer too "
                                       "small for auth keyboard "
                                       "prompt length");
                        goto cleanup;
                    }

                    session->userauth_kybd_prompts[i].text =
                        LIBSSH2_CALLOC(session,
                                       session->userauth_kybd_prompts[i].
                                       length);
                    if(!session->userauth_kybd_prompts[i].text) {
                        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                       "Unable to allocate memory for "
                                       "keyboard-interactive prompt message");
                        goto cleanup;
                    }

                    if(s + session->userauth_kybd_prompts[i].length <=
                       session->userauth_kybd_data +
                       session->userauth_kybd_data_len) {
                        memcpy(session->userauth_kybd_prompts[i].text, s,
                               session->userauth_kybd_prompts[i].length);
                        s += session->userauth_kybd_prompts[i].length;
                    }
                    else {
                        _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                       "userauth keyboard data buffer too "
                                       "small for auth keyboard prompt");
                        goto cleanup;
                    }
                    if(s < session->userauth_kybd_data +
                       session->userauth_kybd_data_len) {
                        /* boolean   echo[1] */
                        session->userauth_kybd_prompts[i].echo = *s++;
                    }
                    else {
                        _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                       "userauth keyboard data buffer too "
                                       "small for auth keyboard prompt echo");
                        goto cleanup;
                    }
                }
            }

            response_callback(session->userauth_kybd_auth_name,
                              session->userauth_kybd_auth_name_len,
                              session->userauth_kybd_auth_instruction,
                              session->userauth_kybd_auth_instruction_len,
                              session->userauth_kybd_num_prompts,
                              session->userauth_kybd_prompts,
                              session->userauth_kybd_responses,
                              &session->abstract);

            _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                           "Keyboard-interactive response callback function"
                           " invoked");

            session->userauth_kybd_packet_len =
                1 /* byte      SSH_MSG_USERAUTH_INFO_RESPONSE */
                + 4             /* int       num-responses */
                ;

            for(i = 0; i < session->userauth_kybd_num_prompts; i++) {
                /* string    response[1] (ISO-10646 UTF-8) */
                if(session->userauth_kybd_responses[i].length <=
                   (SIZE_MAX - 4 - session->userauth_kybd_packet_len) ) {
                    session->userauth_kybd_packet_len +=
                        4 + session->userauth_kybd_responses[i].length;
                }
                else {
                    _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                   "Unable to allocate memory for keyboard-"
                                   "interactive response packet");
                    goto cleanup;
                }
            }

            /* A new userauth_kybd_data area is to be allocated, free the
               former one. */
            LIBSSH2_FREE(session, session->userauth_kybd_data);

            session->userauth_kybd_data = s =
                LIBSSH2_ALLOC(session, session->userauth_kybd_packet_len);
            if(!s) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "Unable to allocate memory for keyboard-"
                               "interactive response packet");
                goto cleanup;
            }

            *s = SSH_MSG_USERAUTH_INFO_RESPONSE;
            s++;
            _libssh2_store_u32(&s, session->userauth_kybd_num_prompts);

            for(i = 0; i < session->userauth_kybd_num_prompts; i++) {
                _libssh2_store_str(&s,
                                   session->userauth_kybd_responses[i].text,
                                   session->userauth_kybd_responses[i].length);
            }

            session->userauth_kybd_state = libssh2_NB_state_sent1;
        }

        if(session->userauth_kybd_state == libssh2_NB_state_sent1) {
            rc = _libssh2_transport_send(session, session->userauth_kybd_data,
                                         session->userauth_kybd_packet_len,
                                         NULL, 0);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                      "Would block");
            if(rc) {
                _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                               "Unable to send userauth-keyboard-interactive"
                               " request");
                goto cleanup;
            }

            session->userauth_kybd_auth_failure = 0;
        }

      cleanup:
        /*
         * It's safe to clean all the data here, because unallocated pointers
         * are filled by zeroes
         */

        LIBSSH2_FREE(session, session->userauth_kybd_data);
        session->userauth_kybd_data = NULL;

        if(session->userauth_kybd_prompts) {
            for(i = 0; i < session->userauth_kybd_num_prompts; i++) {
                LIBSSH2_FREE(session, session->userauth_kybd_prompts[i].text);
                session->userauth_kybd_prompts[i].text = NULL;
            }
        }

        if(session->userauth_kybd_responses) {
            for(i = 0; i < session->userauth_kybd_num_prompts; i++) {
                LIBSSH2_FREE(session,
                             session->userauth_kybd_responses[i].text);
                session->userauth_kybd_responses[i].text = NULL;
            }
        }

        if(session->userauth_kybd_prompts) {
            LIBSSH2_FREE(session, session->userauth_kybd_prompts);
            session->userauth_kybd_prompts = NULL;
        }
        if(session->userauth_kybd_responses) {
            LIBSSH2_FREE(session, session->userauth_kybd_responses);
            session->userauth_kybd_responses = NULL;
        }
        if(session->userauth_kybd_auth_name) {
            LIBSSH2_FREE(session, session->userauth_kybd_auth_name);
            session->userauth_kybd_auth_name = NULL;
        }
        if(session->userauth_kybd_auth_instruction) {
            LIBSSH2_FREE(session, session->userauth_kybd_auth_instruction);
            session->userauth_kybd_auth_instruction = NULL;
        }

        if(session->userauth_kybd_auth_failure) {
            session->userauth_kybd_state = libssh2_NB_state_idle;
            return -1;
        }

        session->userauth_kybd_state = libssh2_NB_state_sent;
    }
}

```

这段代码定义了一个名为`libssh2_userauth_keyboard_interactive_ex`的函数，属于`libssh2_userauth_keyboard_interactive`函数家族。

该函数的作用是使用键盘交互式的身份验证流程，它将用户提供的用户名和用户名长度作为参数传递给`userauth_keyboard_interactive`函数，该函数将根据用户名生成随机数，用于身份验证。

该函数的实现还调用了`response_callback`参数，该函数将接收到的用户名和随机生成的随机数作为参数，用于响应键盘交互式的身份验证结果。


```cpp
/*
 * libssh2_userauth_keyboard_interactive_ex
 *
 * Authenticate using a challenge-response authentication
 */
LIBSSH2_API int
libssh2_userauth_keyboard_interactive_ex(LIBSSH2_SESSION *session,
                                         const char *user,
                                         unsigned int user_len,
                                         LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC
                                         ((*response_callback)))
{
    int rc;
    BLOCK_ADJUST(rc, session,
                 userauth_keyboard_interactive(session, user, user_len,
                                               response_callback));
    return rc;
}

```