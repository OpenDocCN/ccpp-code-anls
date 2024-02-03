# `nmap\libssh2\src\transport.c`

```cpp
/*
 * 以下是版权声明和许可条款
 * 作者：Daniel Stenberg <daniel@haxx.se>
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否经过修改
 * 在再发布源代码时，必须保留以上版权声明、条件列表和以下免责声明
 * 在再发布二进制形式时，必须在文档和/或其他提供的材料中复制以上版权声明、条件列表和免责声明
 * 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 * 无论在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式），版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责
 * 即使已被告知可能发生此类损害，也不得对使用本软件的任何方式负责
 * 该文件处理读写SECSH传输层的功能，符合RFC4253标准
 */

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
#define MAX_BLOCKSIZE 32    /* 定义最大块大小，必须适应我们使用/获取的最大加密块大小 */
#define MAX_MACSIZE 64      /* 定义最大 MAC 长度，必须适应我们支持的最大 MAC 长度 */

#ifdef LIBSSH2DEBUG
#define UNPRINTABLE_CHAR '.'  /* 定义不可打印字符的替代符号 */
static void
debugdump(LIBSSH2_SESSION * session,
          const char *desc, const unsigned char *ptr, size_t size)
{
    size_t i;
    size_t c;
    unsigned int width = 0x10;  /* 设置每行显示的字节数 */
    char buffer[256];  /* 缓冲区大小，必须足够容纳 width*4 + 大约30左右 */
    size_t used;
    static const char *hex_chars = "0123456789ABCDEF";  /* 十六进制字符 */

    if(!(session->showmask & LIBSSH2_TRACE_TRANS)) {
        /* 如果未要求显示跟踪信息，则退出 */
        return;
    }

    used = snprintf(buffer, sizeof(buffer), "=> %s (%d bytes)\n",
                    desc, (int) size);  /* 将描述和大小格式化为字符串 */
    if(session->tracehandler)
        (session->tracehandler)(session, session->tracehandler_context,
                                buffer, used);  /* 如果有跟踪处理程序，则调用它 */
    else
        fprintf(stderr, "%s", buffer);  /* 否则，将字符串输出到标准错误流中 */
    # 以每行指定的宽度循环遍历数据
    for(i = 0; i < size; i += width) {

        # 将当前行的起始地址以十六进制格式写入缓冲区
        used = snprintf(buffer, sizeof(buffer), "%04lx: ", (long)i);

        # 如果十六进制显示未禁用，则将其添加到缓冲区
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

        # 将当前行的可打印字符添加到缓冲区
        for(c = 0; (c < width) && (i + c < size); c++) {
            buffer[used++] = isprint(ptr[i + c]) ?
                ptr[i + c] : UNPRINTABLE_CHAR;
        }
        buffer[used++] = '\n';
        buffer[used] = 0;

        # 如果存在跟踪处理程序，则调用它，否则将缓冲区内容输出到标准错误流
        if(session->tracehandler)
            (session->tracehandler)(session, session->tracehandler_context,
                                    buffer, used);
        else
            fprintf(stderr, "%s", buffer);
    }
}
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
    # 设置会话的完整数据包状态为闲置
    session->fullpacket_state = libssh2_NB_state_idle;
    
    # 返回会话的完整数据包类型
    return session->fullpacket_packet_type;
/*
 * _libssh2_transport_read
 *
 * Collect a packet into the input queue.
 *
 * Returns packet type added to input queue (0 if nothing added), or a
 * negative error number.
 */
/* 
 * 这个函数将一个数据包收集到输入队列中。
 * 
 * 返回添加到输入队列的数据包类型（如果没有添加则返回0），或者一个负数的错误码。
 */

/*
 * This function reads the binary stream as specified in chapter 6 of RFC4253
 * "The Secure Shell (SSH) Transport Layer Protocol"
 *
 * DOES NOT call _libssh2_error() for ANY error case.
 */
/*
 * 这个函数按照 RFC4253 第6章中指定的方式读取二进制流
 * "The Secure Shell (SSH) Transport Layer Protocol"
 * 
 * 对于任何错误情况都不调用 _libssh2_error() 函数。
 */
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
    /*
     * 当寻找更多传入数据时，所有通道、系统、子系统等最终都会到达这里。
     * 如果正在进行密钥交换（设置了 LIBSSH2_STATE_EXCHANGING_KEYS 位），则远程端将
     * 只发送与密钥交换相关的流量。在非阻塞模式下，有可能通过 EAGAIN 状态
     * 从 kex_exchange 函数中跳出，并且永远不会再回到它。如果 LIBSSH2_STATE_EXCHANGING_KEYS
     * 处于活动状态，则必须重定向到密钥交换。但是，如果 kex_exchange 处于活动状态
     * （如它是调用此 packet_read 执行的函数），则不要重定向，因为那将是一个无限循环！
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
}
    /*
     * =============================== NOTE ===============================
     * 我知道这很丑陋，而且不是真正使用 "goto" 的好方法，但是
     * 用任何其他方法来处理这个 case 语句会更加丑陋
     */
    // 如果会话的读取状态为 libssh2_NB_state_jump1
    if(session->readPack_state == libssh2_NB_state_jump1) {
        // 将会话的读取状态设置为 libssh2_NB_state_idle
        session->readPack_state = libssh2_NB_state_idle;
        // 将加密数据设置为会话的读取加密数据
        encrypted = session->readPack_encrypted;
        // 跳转到 libssh2_transport_read_point1 标签处
        goto libssh2_transport_read_point1;
    }

    } while(1);                /* loop */

    // 返回套接字接收错误
    return LIBSSH2_ERROR_SOCKET_RECV; /* we never reach this point */
    # 发送现有数据包
static int
send_existing(LIBSSH2_SESSION *session, const unsigned char *data,
              size_t data_len, ssize_t *ret)
{
    ssize_t rc;
    ssize_t length;
    struct transportpacket *p = &session->packet;

    # 如果不存在数据包，则返回无错误
    if(!p->olen) {
        *ret = 0;
        return LIBSSH2_ERROR_NONE;
    }

    /* 尽可能发送现有数据包 */
    if((data != p->odata) || (data_len != p->olen)) {
        /* 当我们即将完成发送数据包时，非常重要的是调用者不要尝试发送新的/不同的数据包，
           因为我们在上一个数据包发送完之前不会添加新的数据包。为了让调用者真正注意到他/她的错误，
           我们对这种情况返回错误 */
        return LIBSSH2_ERROR_BAD_USE;
    }

    *ret = 1;                   /* 设置以便让我们的父函数返回 */

    /* 剩余要发送的字节数 */
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
        /* 包的剩余部分已发送 */
        p->ototal_num = 0;
        p->olen = 0;
        /* 我们保留 *ret 设置，以便父函数返回，因为我们现在必须返回发送成功，
           这样我们就不会在后面冒险发送 EAGAIN，否则会混淆父函数 */
        return LIBSSH2_ERROR_NONE;

    }
}
    else if(rc < 0) {
        /* 如果返回值小于 0，表示没有发送任何数据 */
        if(rc != -EAGAIN)
            /* 如果返回值不是 -EAGAIN，表示发送失败 */
            return LIBSSH2_ERROR_SOCKET_SEND;

        session->socket_block_directions |= LIBSSH2_SESSION_BLOCK_OUTBOUND;
        return LIBSSH2_ERROR_EAGAIN;
    }

    p->osent += rc;         /* 更新已发送数据量 */

    return rc < length ? LIBSSH2_ERROR_EAGAIN : LIBSSH2_ERROR_NONE;
# 发送一个数据包，如果需要的话进行加密并添加 MAC 码
# 在成功时返回 0，在失败时返回非零值
# 该函数接收两部分数据，由该函数合并。'data' 部分会在 'data2' 之前立即发送。'data2' 可以设置为 NULL，以便只使用一个部分。
# 如果会阻塞或者整个数据包还没有发送完，则返回 LIBSSH2_ERROR_EAGAIN。如果是这样，调用者应该在可能发送更多数据时再次调用该函数，并且该函数必须使用相同的参数集（相同的数据指针和相同的数据长度）调用，直到返回 ERROR_NONE 或失败为止。
# 该函数在任何错误情况下都不会调用 _libssh2_error()
int _libssh2_transport_send(LIBSSH2_SESSION *session,
                            const unsigned char *data, size_t data_len,
                            const unsigned char *data2, size_t data2_len)
{
    # 如果当前状态是新密钥状态，则块大小为本地加密算法的块大小，否则为 8
    int blocksize =
        (session->state & LIBSSH2_STATE_NEWKEYS) ?
        session->local.crypt->blocksize : 8;
    # 填充长度
    int padding_length;
    # 数据包长度
    size_t packet_length;
    # 总长度
    int total_length;
    # 如果定义了 RANDOM_PADDING，则使用随机填充
    # FIXME: 使这个随机化
    # 随机填充的最大值
    int rand_max;
    # 种子为数据的第一个字节
    int seed = data[0];
    # 传输数据包
    struct transportpacket *p = &session->packet;
    # 是否加密
    int encrypted;
    # 是否压缩
    int compressed;
    # 返回值
    ssize_t ret;
    # 返回码
    int rc;
    # 原始数据
    const unsigned char *orgdata = data;
    # 原始数据长度
    size_t orgdata_len = data_len;

    # 如果上次读操作在密钥交换的中途被中断，必须在继续写入更多数据之前完成该密钥交换。
    # 有关更多详细信息，请参阅 _libssh2_transport_read 中的类似块。
    // 如果会话状态包含交换密钥状态，并且不包含密钥交换活跃状态
    if(session->state & LIBSSH2_STATE_EXCHANGING_KEYS &&
        !(session->state & LIBSSH2_STATE_KEX_ACTIVE)) {
        // 如果仍处于密钥交换过程中，则不写入任何新的数据包
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Redirecting into the"
                       " key re-exchange from _libssh2_transport_send");
        // 重新进行密钥交换
        rc = _libssh2_kex_exchange(session, 1, &session->startup_key_state);
        if(rc)
            return rc;
    }

    // 输出调试信息，显示未加密的数据
    debugdump(session, "libssh2_transport_write plain", data, data_len);
    // 如果存在第二个数据包，也输出调试信息
    if(data2)
        debugdump(session, "libssh2_transport_write plain2", data2, data2_len);

    // 首先，检查是否有待发送的数据。send_existing 只对 data 和 data_len 进行合法性检查，不包括 data2 和 data2_len
    rc = send_existing(session, data, data_len, &ret);
    if(rc)
        return rc;

    // 取消会话的出站阻塞状态
    session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_OUTBOUND;

    // 如果数据已发送，则返回
    if(ret)
        /* set by send_existing if data was sent */
        return rc;

    // 判断是否已加密
    encrypted = (session->state & LIBSSH2_STATE_NEWKEYS) ? 1 : 0;

    // 判断是否已压缩
    compressed =
        session->local.comp != NULL &&
        session->local.comp->compress &&
        ((session->state & LIBSSH2_STATE_AUTHENTICATED) ||
         session->local.comp->use_in_auth);
    # 如果数据需要加密、压缩，并且会话中存在本地压缩抽象，则执行以下操作
    if(encrypted && compressed && session->local.comp_abstract) {
        # 这里的想法是，如果输出超过了分配的缓冲区大小，这些函数必须失败，因此它们不检查输入大小，因为我们不知道压缩的大小
        size_t dest_len = MAX_SSH_PACKET_LEN-5-256;  # 设置目标缓冲区的大小
        size_t dest2_len = dest_len;  # 设置第二个目标缓冲区的大小

        # 直接压缩到目标缓冲区
        rc = session->local.comp->comp(session,
                                       &p->outbuf[5], &dest_len,
                                       data, data_len,
                                       &session->local.comp_abstract);
        if(rc)
            return rc;     # 压缩失败

        if(data2 && data2_len) {
            # 在前一个调用放置数据的位置之后，直接压缩到目标缓冲区
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
            return rc;     # 压缩失败

        data_len = dest_len + dest2_len;  # 使用组合长度
    }
    else {
        if((data_len + data2_len) >= (MAX_SSH_PACKET_LEN-0x100))
            # 数据包太大，对此返回错误，直到我们使此函数分割并发送多个 SSH 数据包
            return LIBSSH2_ERROR_INVAL;

        # 复制有效载荷数据
        memcpy(&p->outbuf[5], data, data_len);
        if(data2 && data2_len)
            memcpy(&p->outbuf[5 + data_len], data2, data2_len);
        data_len += data2_len;  # 使用组合长度
    }
    /* 根据 RFC4253 规定：'packet_length'、'padding_length'、'payload' 和 'random padding' 的长度必须是密钥块大小或 8 的倍数，取两者中较大的值 */

    /* 简单的数学计算：(4 + 1 + packet_length + padding_length) % blocksize == 0 */

    packet_length = data_len + 1 + 4;   /* 1 用于 padding_length 字段，4 用于 packet_length 字段 */

    /* 到这一步，我们已经得到了除了填充之外的所有内容 */

    /* 首先确定我们的最小填充量，使其成为偶数块大小 */
    padding_length = blocksize - (packet_length % blocksize);

    /* 如果填充量变得太小，我们添加另一个块大小的填充量（取自原始的 libssh2，没有任何真正的解释） */
    if(padding_length < 4) {
        padding_length += blocksize;
    }
#ifdef RANDOM_PADDING
    /* 如果定义了 RANDOM_PADDING，则执行以下代码块 */

    /* 计算随机数的最大值，用于生成填充长度 */
    rand_max = (255 - padding_length) / blocksize + 1;
    padding_length += blocksize * (seed % rand_max);
#endif

    /* 将填充长度加到数据包长度上 */
    packet_length += padding_length;

    /* 将 MAC 长度加到总长度上 */
    total_length =
        packet_length + (encrypted ? session->local.mac->mac_len : 0);

    /* 存储数据包长度，即整个数据包的大小减去 MAC 和数据包长度字段本身的大小 */
    _libssh2_htonu32(p->outbuf, packet_length - 4);
    /* 存储填充长度 */
    p->outbuf[4] = (unsigned char)padding_length;

    /* 用随机数据填充填充区域 */
    if(_libssh2_random(p->outbuf + 5 + data_len, padding_length)) {
        return _libssh2_error(session, LIBSSH2_ERROR_RANDGEN,
                              "Unable to get random bytes for packet padding");
    }
    if(encrypted) {
        size_t i;

        /* 计算 MAC 哈希值。将输出放在索引 packet_length 处，因为该大小包括整个数据包。
           MAC 是在整个未加密的数据包上计算的，包括所有字段，除了 MAC 字段本身。 */
        session->local.mac->hash(session, p->outbuf + packet_length,
                                 session->local.seqno, p->outbuf,
                                 packet_length, NULL, 0,
                                 &session->local.mac_abstract);

        /* 逐个块大小加密整个数据包数据。MAC 字段不被加密。 */
        for(i = 0; i < packet_length; i += session->local.crypt->blocksize) {
            unsigned char *ptr = &p->outbuf[i];
            if(session->local.crypt->crypt(session, ptr,
                                            session->local.crypt->blocksize,
                                            &session->local.crypt_abstract))
                return LIBSSH2_ERROR_ENCRYPT;     /* 加密失败 */
        }
    }

    session->local.seqno++;  // 递增序列号

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
    # 如果发送的字节数不等于总长度
    if(ret != total_length) {
        # 如果发送的字节数大于等于0或者等于-EAGAIN
        if(ret >= 0 || ret == -EAGAIN) {
            # 无法发送整个数据包，保存剩余部分
            session->socket_block_directions |= LIBSSH2_SESSION_BLOCK_OUTBOUND;
            p->odata = orgdata;
            p->olen = orgdata_len;
            p->osent = ret <= 0 ? 0 : ret;
            p->ototal_num = total_length;
            # 返回需要再次尝试的错误码
            return LIBSSH2_ERROR_EAGAIN;
        }
        # 返回套接字发送错误
        return LIBSSH2_ERROR_SOCKET_SEND;
    }

    # 整个数据包已经发送完成
    p->odata = NULL;
    p->olen = 0;

    # 返回无错误
    return LIBSSH2_ERROR_NONE;         # 一切正常
# 闭合前面的函数定义
```