# `nmap\libssh2\src\misc.c`

```
/*
 * 版权声明，版权所有人和贡献者的声明
 * 允许在源代码和二进制形式下重新分发和使用，需满足以下条件
 * 在源代码中保留版权声明、条件列表和以下免责声明
 * 在二进制形式中在文档和/或其他提供的材料中重现版权声明、条件列表和免责声明
 * 未经特定事先书面许可，不得使用版权所有者或其他贡献者的名称来认可或推广基于此软件的产品
 * 版权所有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责
 * 即使已被告知可能发生此类损害，也不得在任何责任理论下对使用本软件产生的任何方式的任何损害负责
 */

#include "libssh2_priv.h"
#include "misc.h"
#include "blf.h"

#ifdef HAVE_STDLIB_H
#include <stdlib.h>
#endif

#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif

#ifdef HAVE_SYS_TIME_H
#include <sys/time.h>
#endif

#if defined(HAVE_DECL_SECUREZEROMEMORY) && HAVE_DECL_SECUREZEROMEMORY
#ifdef HAVE_WINDOWS_H
#include <windows.h>
#endif
#endif

#include <stdio.h>
#include <errno.h>

int _libssh2_error_flags(LIBSSH2_SESSION* session, int errcode,
                         const char *errmsg, int errflags)
{
    // 如果错误标志包含重复标志，释放先前的错误消息
    if(session->err_flags & LIBSSH2_ERR_FLAG_DUP)
        LIBSSH2_FREE(session, (char *)session->err_msg);

    // 设置错误代码和清空错误标志
    session->err_code = errcode;
    session->err_flags = 0;

    // 如果错误消息不为空且包含重复标志，则复制错误消息
    if((errmsg != NULL) && ((errflags & LIBSSH2_ERR_FLAG_DUP) != 0)) {
        size_t len = strlen(errmsg);
        char *copy = LIBSSH2_ALLOC(session, len + 1);
        if(copy) {
            memcpy(copy, errmsg, len + 1);
            session->err_flags = LIBSSH2_ERR_FLAG_DUP;
            session->err_msg = copy;
        }
        else
            // 内存不足：这种情况非常不太可能发生
            session->err_msg = "former error forgotten (OOM)";
    }
    else
        session->err_msg = errmsg;

#ifdef LIBSSH2DEBUG
    if((errcode == LIBSSH2_ERROR_EAGAIN) && !session->api_block_mode)
        // 如果是 EAGAIN 并且处于非阻塞模式，则不生成调试输出
        return errcode;
    _libssh2_debug(session, LIBSSH2_TRACE_ERROR, "%d - %s", session->err_code,
                   session->err_msg);
#endif

    return errcode;
}

int _libssh2_error(LIBSSH2_SESSION* session, int errcode, const char *errmsg)
{
    return _libssh2_error_flags(session, errcode, errmsg, 0);
}

#ifdef WIN32
static int wsa2errno(void)
{
    switch(WSAGetLastError()) {
    case WSAEWOULDBLOCK:
        return EAGAIN;

    case WSAENOTSOCK:
        return EBADF;

    case WSAEINTR:
        return EINTR;

    default:
        // 最重要的是确保当发生不同的错误时，errno 不会保持在 EAGAIN，因此将 errno 设置为通用错误
        return EIO;
    }
}
#endif

/* _libssh2_recv
 *
 * 替换标准的 recv，失败时返回 -errno
 */
ssize_t
# 从指定的套接字接收数据，存储到指定的缓冲区中
_libssh2_recv(libssh2_socket_t sock, void *buffer, size_t length,
              int flags, void **abstract)
{
    ssize_t rc;

    (void) abstract;

    # 调用系统的 recv 函数接收数据
    rc = recv(sock, buffer, length, flags);
#ifdef WIN32
    # 如果接收失败，返回 Windows 套接字错误码
    if(rc < 0)
        return -wsa2errno();
#else
    if(rc < 0) {
        # 如果接收失败，根据不同的系统错误码返回不同的错误码
        /* 有时第一个 recv() 函数调用会在 Solaris 和 HP-UX 上将 errno 设置为 ENOENT */
        if(errno == ENOENT)
            return -EAGAIN;
#ifdef EWOULDBLOCK /* 对于 VMS 和其他特殊的 Unix 系统 */
        else if(errno == EWOULDBLOCK)
          return -EAGAIN;
#endif
        else
            return -errno;
    }
#endif
    return rc;
}

/* _libssh2_send
 *
 * 替代标准的 send 函数，在失败时返回 -errno
 */
ssize_t
_libssh2_send(libssh2_socket_t sock, const void *buffer, size_t length,
              int flags, void **abstract)
{
    ssize_t rc;

    (void) abstract;

    # 调用系统的 send 函数发送数据
    rc = send(sock, buffer, length, flags);
#ifdef WIN32
    # 如果发送失败，返回 Windows 套接字错误码
    if(rc < 0)
        return -wsa2errno();
#else
    if(rc < 0) {
#ifdef EWOULDBLOCK /* 对于 VMS 和其他特殊的 Unix 系统 */
      if(errno == EWOULDBLOCK)
        return -EAGAIN;
#endif
      return -errno;
    }
#endif
    return rc;
}

/* libssh2_ntohu32
 */
unsigned int
_libssh2_ntohu32(const unsigned char *buf)
{
    # 将网络字节序的无符号 32 位整数转换为主机字节序
    return (((unsigned int)buf[0] << 24)
           | ((unsigned int)buf[1] << 16)
           | ((unsigned int)buf[2] << 8)
           | ((unsigned int)buf[3]));
}


/* _libssh2_ntohu64
 */
libssh2_uint64_t
_libssh2_ntohu64(const unsigned char *buf)
{
    unsigned long msl, lsl;

    # 将网络字节序的无符号 64 位整数转换为主机字节序
    msl = ((libssh2_uint64_t)buf[0] << 24) | ((libssh2_uint64_t)buf[1] << 16)
        | ((libssh2_uint64_t)buf[2] << 8) | (libssh2_uint64_t)buf[3];
    lsl = ((libssh2_uint64_t)buf[4] << 24) | ((libssh2_uint64_t)buf[5] << 16)
        | ((libssh2_uint64_t)buf[6] << 8) | (libssh2_uint64_t)buf[7];

    return ((libssh2_uint64_t)msl <<32) | lsl;
}

/* _libssh2_htonu32
 */
void
_libssh2_htonu32(unsigned char *buf, uint32_t value)
{
    # 将32位整数value按照大端序写入到缓冲区buf中
    buf[0] = (value >> 24) & 0xFF;
    buf[1] = (value >> 16) & 0xFF;
    buf[2] = (value >> 8) & 0xFF;
    buf[3] = value & 0xFF;
/* _libssh2_store_u32
 */
// 存储一个32位的无符号整数到缓冲区中
void _libssh2_store_u32(unsigned char **buf, uint32_t value)
{
    // 将32位无符号整数转换为网络字节顺序，并存储到缓冲区中
    _libssh2_htonu32(*buf, value);
    // 移动缓冲区指针到下一个位置
    *buf += sizeof(uint32_t);
}

/* _libssh2_store_str
 */
// 存储一个字符串到缓冲区中
void _libssh2_store_str(unsigned char **buf, const char *str, size_t len)
{
    // 存储字符串的长度到缓冲区中
    _libssh2_store_u32(buf, (uint32_t)len);
    // 如果字符串长度不为0，则将字符串内容复制到缓冲区中，并移动缓冲区指针到下一个位置
    if(len) {
        memcpy(*buf, str, len);
        *buf += len;
    }
}

/* Base64 Conversion */

// Base64 反转表
static const short base64_reverse_table[256] = {
    // 省略了大量的数值，用于将 Base64 编码的字符转换为对应的数值
};

/* libssh2_base64_decode
 *
 * 解码一个 Base64 编码的块，并将其存储到一个新分配的缓冲区中
 */
// 解码一个 Base64 编码的块，并将其存储到一个新分配的缓冲区中
LIBSSH2_API int
libssh2_base64_decode(LIBSSH2_SESSION *session, char **data,
                      unsigned int *datalen, const char *src,
                      unsigned int src_len)
{
    unsigned char *s, *d;
    short v;
    int i = 0, len = 0;

    // 分配一个新的缓冲区用于存储解码后的数据
    *data = LIBSSH2_ALLOC(session, (3 * src_len / 4) + 1);
    d = (unsigned char *) *data;
    # 如果输入参数 d 为空，则返回内存分配错误
    if(!d) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for base64 decoding");
    }

    # 遍历输入的 base64 编码数据
    for(s = (unsigned char *) src; ((char *) s) < (src + src_len); s++) {
        # 使用 base64 反转表将字符转换为对应的值
        v = base64_reverse_table[*s];
        # 如果值小于 0，则跳过
        if(v < 0)
            continue;
        # 根据 base64 编码规则进行解码
        switch(i % 4) {
        case 0:
            d[len] = (unsigned char)(v << 2);
            break;
        case 1:
            d[len++] |= v >> 4;
            d[len] = (unsigned char)(v << 4);
            break;
        case 2:
            d[len++] |= v >> 2;
            d[len] = (unsigned char)(v << 6);
            break;
        case 3:
            d[len++] |= v;
            break;
        }
        i++;
    }
    # 如果解码后的数据长度不是 4 的倍数，则返回无效的 base64 编码错误
    if((i % 4) == 1) {
        /* Invalid -- We have a byte which belongs exclusively to a partial
           octet */
        LIBSSH2_FREE(session, *data);
        *data = NULL;
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL, "Invalid base64");
    }

    # 将解码后的数据长度赋值给输出参数 datalen
    *datalen = len;
    # 返回解码结果
    return 0;
/* ---- Base64编码/解码表 ---- */
static const char table64[]=
  "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

/*
 * _libssh2_base64_encode()
 *
 * 返回新创建的base64字符串的长度。第三个参数是指向分配的存储base64数据的区域的指针。如果出现问题，返回0。
 *
 */
size_t _libssh2_base64_encode(LIBSSH2_SESSION *session,
                              const char *inp, size_t insize, char **outptr)
{
    unsigned char ibuf[3]; // 用于存储输入数据的缓冲区
    unsigned char obuf[4]; // 用于存储输出数据的缓冲区
    int i; // 循环计数器
    int inputparts; // 输入数据的分块数
    char *output; // 输出数据的指针
    char *base64data; // base64数据的指针
    const char *indata = inp; // 输入数据的指针

    *outptr = NULL; /* 在到达结尾之前，设置为NULL以防出现故障 */

    if(0 == insize)
        insize = strlen(indata); // 如果输入大小为0，则计算输入数据的长度

    base64data = output = LIBSSH2_ALLOC(session, insize * 4 / 3 + 4); // 分配存储base64数据的区域
    if(NULL == output)
        return 0; // 如果分配失败，则返回0
    # 当输入数据大小大于 0 时执行循环
    while(insize > 0) {
        # 初始化输入部分和索引 i
        for(i = inputparts = 0; i < 3; i++) {
            # 如果输入数据大小大于 0，则读取一个字节到输入缓冲区，同时更新输入数据指针和大小
            if(insize > 0) {
                inputparts++;
                ibuf[i] = *indata;
                indata++;
                insize--;
            }
            # 如果输入数据大小不大于 0，则将输入缓冲区置为 0
            else
                ibuf[i] = 0;
        }

        # 根据输入缓冲区的内容计算输出缓冲区的值
        obuf[0] = (unsigned char)  ((ibuf[0] & 0xFC) >> 2);
        obuf[1] = (unsigned char) (((ibuf[0] & 0x03) << 4) | \
                                   ((ibuf[1] & 0xF0) >> 4));
        obuf[2] = (unsigned char) (((ibuf[1] & 0x0F) << 2) | \
                                   ((ibuf[2] & 0xC0) >> 6));
        obuf[3] = (unsigned char)   (ibuf[2] & 0x3F);

        # 根据输入部分的数量选择不同的输出格式
        switch(inputparts) {
        case 1: /* 只读取一个字节 */
            snprintf(output, 5, "%c%c==",
                     table64[obuf[0]],
                     table64[obuf[1]]);
            break;
        case 2: /* 读取两个字节 */
            snprintf(output, 5, "%c%c%c=",
                     table64[obuf[0]],
                     table64[obuf[1]],
                     table64[obuf[2]]);
            break;
        default:
            snprintf(output, 5, "%c%c%c%c",
                     table64[obuf[0]],
                     table64[obuf[1]],
                     table64[obuf[2]],
                     table64[obuf[3]]);
            break;
        }
        # 更新输出缓冲区指针
        output += 4;
    }
    # 将输出缓冲区的最后一个字符置为 0
    *output = 0;
    # 将输出指针指向 base64data，使其返回实际数据内存
    *outptr = base64data; 
    # 返回新数据的长度
    return strlen(base64data); 
/* ---- End of Base64 Encoding ---- */
/* Base64 编码结束 */

LIBSSH2_API void
libssh2_free(LIBSSH2_SESSION *session, void *ptr)
{
    LIBSSH2_FREE(session, ptr);
    // 释放由 libssh2_malloc 分配的内存
}

#ifdef LIBSSH2DEBUG
#include <stdarg.h>

LIBSSH2_API int
libssh2_trace(LIBSSH2_SESSION * session, int bitmask)
{
    session->showmask = bitmask;
    return 0;
    // 设置会话的跟踪标志
}

LIBSSH2_API int
libssh2_trace_sethandler(LIBSSH2_SESSION *session, void *handler_context,
                         libssh2_trace_handler_func callback)
{
    session->tracehandler = callback;
    session->tracehandler_context = handler_context;
    return 0;
    // 设置会话的跟踪处理程序
}

void
_libssh2_debug(LIBSSH2_SESSION * session, int context, const char *format, ...)
{
    char buffer[1536];
    int len, msglen, buflen = sizeof(buffer);
    va_list vargs;
    struct timeval now;
    static int firstsec;
    static const char *const contexts[] = {
        "Unknown",
        "Transport",
        "Key Ex",
        "Userauth",
        "Conn",
        "SCP",
        "SFTP",
        "Failure Event",
        "Publickey",
        "Socket",
    };
    const char *contexttext = contexts[0];
    unsigned int contextindex;

    if(!(session->showmask & context)) {
        /* no such output asked for */
        return;
        // 如果没有请求这样的输出，则返回
    }

    /* Find the first matching context string for this message */
    for(contextindex = 0; contextindex < ARRAY_SIZE(contexts);
         contextindex++) {
        if((context & (1 << contextindex)) != 0) {
            contexttext = contexts[contextindex];
            break;
        }
    }

    _libssh2_gettimeofday(&now, NULL);
    if(!firstsec) {
        firstsec = now.tv_sec;
    }
    now.tv_sec -= firstsec;

    len = snprintf(buffer, buflen, "[libssh2] %d.%06d %s: ",
                   (int)now.tv_sec, (int)now.tv_usec, contexttext);

    if(len >= buflen)
        msglen = buflen - 1;
    // 计算要打印的消息的长度
    else {
        # 减去已经处理的长度
        buflen -= len;
        # 保存当前处理的长度
        msglen = len;
        # 开始处理可变参数
        va_start(vargs, format);
        # 格式化输出到缓冲区
        len = vsnprintf(buffer + msglen, buflen, format, vargs);
        # 结束可变参数处理
        va_end(vargs);
        # 更新消息长度
        msglen += len < buflen ? len : buflen - 1;
    }

    # 如果有跟踪处理函数，则调用它
    if(session->tracehandler)
        (session->tracehandler)(session, session->tracehandler_context, buffer,
                                msglen);
    # 否则，将消息输出到标准错误流
    else
        fprintf(stderr, "%s\n", buffer);
}

#else
LIBSSH2_API int
libssh2_trace(LIBSSH2_SESSION * session, int bitmask)
{
    (void) session;  // 忽略参数 session
    (void) bitmask;  // 忽略参数 bitmask
    return 0;  // 返回 0
}

LIBSSH2_API int
libssh2_trace_sethandler(LIBSSH2_SESSION *session, void *handler_context,
                         libssh2_trace_handler_func callback)
{
    (void) session;  // 忽略参数 session
    (void) handler_context;  // 忽略参数 handler_context
    (void) callback;  // 忽略参数 callback
    return 0;  // 返回 0
}
#endif

/* 初始化链表头部 */
void _libssh2_list_init(struct list_head *head)
{
    head->first = head->last = NULL;  // 将头部的 first 和 last 指针都设置为 NULL
}

/* 向链表中添加节点 */
void _libssh2_list_add(struct list_head *head,
                       struct list_node *entry)
{
    /* 存储对头部的指针 */
    entry->head = head;

    /* 我们将此条目添加到“顶部”，因此它没有下一个 */
    entry->next = NULL;

    /* 使我们的 prev 指向头部认为的最后一个 */
    entry->prev = head->last;

    /* 并且现在头部的最后一个是我们 */
    head->last = entry;

    /* 确保我们的“prev”节点指向我们的下一个 */
    if(entry->prev)
        entry->prev->next = entry;
    else
        head->first = entry;
}

/* 返回此头部指向的链表中的“第一个”节点 */
void *_libssh2_list_first(struct list_head *head)
{
    return head->first;
}

/* 返回链表中的下一个节点 */
void *_libssh2_list_next(struct list_node *node)
{
    return node->next;
}

/* 返回链表中的上一个节点 */
void *_libssh2_list_prev(struct list_node *node)
{
    return node->prev;
}

/* 从链表中移除此节点 */
void _libssh2_list_remove(struct list_node *entry)
{
    if(entry->prev)
        entry->prev->next = entry->next;
    else
        entry->head->first = entry->next;

    if(entry->next)
        entry->next->prev = entry->prev;
    else
        entry->head->last = entry->prev;
}

#if 0
/* 在给定的“after”条目之前插入一个节点 */
void _libssh2_list_insert(struct list_node *after, /* 在此之前插入 */
                          struct list_node *entry)
{
    /* 'after' 在 'entry' 之后 */
    # 将当前节点的下一个指针指向after节点
    bentry->next = after;

    # 将当前节点的前一个指针指向after节点的前一个节点
    entry->prev = after->prev;

    # 如果当前节点的前一个节点存在，则将其next指针指向当前节点，否则将头指针指向当前节点
    if(entry->prev)
        entry->prev->next = entry;
    else
        after->head->first = entry;

    # 将after节点的前一个指针指向当前节点
    after->prev = entry;

    # after节点的下一个指针仍然指向之前的节点

    # 当前节点的头指针与after节点的头指针相同
    entry->head = after->head;
#endif

#ifdef LIBSSH2_GETTIMEOFDAY_WIN32
/*
 * gettimeofday
 * Implementation according to:
 * The Open Group Base Specifications Issue 6
 * IEEE Std 1003.1, 2004 Edition
 */

/*
 *  THIS SOFTWARE IS NOT COPYRIGHTED
 *
 *  This source code is offered for use in the public domain. You may
 *  use, modify or distribute it freely.
 *
 *  This code is distributed in the hope that it will be useful but
 *  WITHOUT ANY WARRANTY. ALL WARRANTIES, EXPRESS OR IMPLIED ARE HEREBY
 *  DISCLAIMED. This includes but is not limited to warranties of
 *  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 *  Contributed by:
 *  Danny Smith <dannysmith@users.sourceforge.net>
 */

/* Offset between 1/1/1601 and 1/1/1970 in 100 nanosec units */
#define _W32_FT_OFFSET (116444736000000000)

// 定义获取时间的函数
int __cdecl _libssh2_gettimeofday(struct timeval *tp, void *tzp)
{
    union {
        unsigned __int64 ns100; /*time since 1 Jan 1601 in 100ns units */
        FILETIME ft;
    } _now;
    (void)tzp;
    if(tp) {
        // 获取当前系统时间
        GetSystemTimeAsFileTime(&_now.ft);
        // 计算微秒
        tp->tv_usec = (long)((_now.ns100 / 10) % 1000000);
        // 计算秒
        tp->tv_sec = (long)((_now.ns100 - _W32_FT_OFFSET) / 10000000);
    }
    /* Always return 0 as per Open Group Base Specifications Issue 6.
       Do not set errno on error.  */
    return 0;
}


#endif

// 分配内存并初始化为0
void *_libssh2_calloc(LIBSSH2_SESSION* session, size_t size)
{
    void *p = LIBSSH2_ALLOC(session, size);
    if(p) {
        // 将分配的内存初始化为0
        memset(p, 0, size);
    }
    return p;
}

/* 对输入缓冲区进行异或操作，结果存储在输出缓冲区中。
   可以安全地使用输入缓冲区作为输出缓冲区。 */
void _libssh2_xor_data(unsigned char *output,
                       const unsigned char *input1,
                       const unsigned char *input2,
                       size_t length)
{
    size_t i;

    for(i = 0; i < length; i++)
        // 对应位置的字节进行异或操作
        *output++ = *input1++ ^ *input2++;
}
/* Increments an AES CTR buffer to prepare it for use with the
   next AES block. */
void _libssh2_aes_ctr_increment(unsigned char *ctr,
                                size_t length)
{
    // 定义指针变量和无符号整型变量
    unsigned char *pc;
    unsigned int val, carry;

    // 将指针指向最后一个字节
    pc = ctr + length - 1;
    carry = 1;

    // 逐个字节递增，处理进位
    while(pc >= ctr) {
        val = (unsigned int)*pc + carry;
        *pc-- = val & 0xFF;
        carry = val >> 8;
    }
}

#ifdef WIN32
static void * (__cdecl * const volatile memset_libssh)(void *, int, size_t) =
    memset;
#else
static void * (* const volatile memset_libssh)(void *, int, size_t) = memset;
#endif

void _libssh2_explicit_zero(void *buf, size_t size)
{
    // 根据不同的平台使用不同的内存清零函数
#if defined(HAVE_DECL_SECUREZEROMEMORY) && HAVE_DECL_SECUREZEROMEMORY
    SecureZeroMemory(buf, size);
    (void)memset_libssh; /* Silence unused variable warning */
#elif defined(HAVE_MEMSET_S)
    (void)memset_s(buf, size, 0, size);
    (void)memset_libssh; /* Silence unused variable warning */
#else
    memset_libssh(buf, 0, size);
#endif
}

/* String buffer */

struct string_buf* _libssh2_string_buf_new(LIBSSH2_SESSION *session)
{
    // 分配内存并初始化字符串缓冲区
    struct string_buf *ret;
    ret = _libssh2_calloc(session, sizeof(*ret));
    if(ret == NULL)
        return NULL;

    return ret;
}

void _libssh2_string_buf_free(LIBSSH2_SESSION *session, struct string_buf *buf)
{
    // 释放字符串缓冲区的内存
    if(buf == NULL)
        return;

    if(buf->data != NULL)
        LIBSSH2_FREE(session, buf->data);

    LIBSSH2_FREE(session, buf);
    buf = NULL;
}

int _libssh2_get_u32(struct string_buf *buf, uint32_t *out)
{
    // 从字符串缓冲区中读取32位无符号整数
    if(!_libssh2_check_length(buf, 4)) {
        return -1;
    }

    *out = _libssh2_ntohu32(buf->dataptr);
    buf->dataptr += 4;
    return 0;
}

int _libssh2_get_u64(struct string_buf *buf, libssh2_uint64_t *out)
{
    // 从字符串缓冲区中读取64位无符号整数
    if(!_libssh2_check_length(buf, 8)) {
        return -1;
    }

    *out = _libssh2_ntohu64(buf->dataptr);
    buf->dataptr += 8;
    return 0;
}

int _libssh2_match_string(struct string_buf *buf, const char *match)
{
    // 匹配字符串缓冲区中的字符串
    # 声明一个指向无符号字符的指针变量 out
    unsigned char *out;
    # 声明一个变量 len，并初始化为 0
    size_t len = 0;
    # 如果 _libssh2_get_string 函数返回非零，或者 len 不等于 match 的长度，或者 out 和 match 不相等，则执行下面的语句
    if(_libssh2_get_string(buf, &out, &len) || len != strlen(match) ||
        strncmp((char *)out, match, strlen(match)) != 0) {
        # 返回 -1
        return -1;
    }
    # 返回 0
    return 0;
# 获取字符串数据，并将其存储在指定的输出缓冲区中
int _libssh2_get_string(struct string_buf *buf, unsigned char **outbuf,
                        size_t *outlen)
{
    # 读取数据长度
    uint32_t data_len;
    if(_libssh2_get_u32(buf, &data_len) != 0) {
        return -1;
    }
    # 检查数据长度是否有效
    if(!_libssh2_check_length(buf, data_len)) {
        return -1;
    }
    # 将数据存储在输出缓冲区中
    *outbuf = buf->dataptr;
    buf->dataptr += data_len;

    if(outlen)
        *outlen = (size_t)data_len;

    return 0;
}

# 复制字符串数据到指定的输出缓冲区中
int _libssh2_copy_string(LIBSSH2_SESSION *session, struct string_buf *buf,
                         unsigned char **outbuf, size_t *outlen)
{
    size_t str_len;
    unsigned char *str;

    # 获取字符串数据
    if(_libssh2_get_string(buf, &str, &str_len)) {
        return -1;
    }
    # 分配内存并复制字符串数据
    *outbuf = LIBSSH2_ALLOC(session, str_len);
    if(*outbuf) {
        memcpy(*outbuf, str, str_len);
    }
    else {
        return -1;
    }

    if(outlen)
        *outlen = str_len;

    return 0;
}

# 获取大数的字节数据
int _libssh2_get_bignum_bytes(struct string_buf *buf, unsigned char **outbuf,
                              size_t *outlen)
{
    uint32_t data_len;
    uint32_t bn_len;
    unsigned char *bnptr;

    # 获取数据长度
    if(_libssh2_get_u32(buf, &data_len)) {
        return -1;
    }
    # 检查数据长度是否有效
    if(!_libssh2_check_length(buf, data_len)) {
        return -1;
    }

    bn_len = data_len;
    bnptr = buf->dataptr;

    # 去除前导零
    while(bn_len > 0 && *bnptr == 0x00) {
        bn_len--;
        bnptr++;
    }

    *outbuf = bnptr;
    buf->dataptr += data_len;

    if(outlen)
        *outlen = (size_t)bn_len;

    return 0;
}

# 检查缓冲区中是否有足够的数据长度
int _libssh2_check_length(struct string_buf *buf, size_t len)
{
    unsigned char *endp = &buf->data[buf->len];
    size_t left = endp - buf->dataptr;
    return ((len <= left) && (left <= buf->len));
}

# 包装函数
# 使用bcrypt_pbkdf函数计算PBKDF值
int _libssh2_bcrypt_pbkdf(const char *pass,  # 输入参数：密码
                          size_t passlen,     # 输入参数：密码长度
                          const uint8_t *salt,  # 输入参数：盐值
                          size_t saltlen,       # 输入参数：盐值长度
                          uint8_t *key,         # 输出参数：密钥
                          size_t keylen,        # 输出参数：密钥长度
                          unsigned int rounds)  # 输入参数：迭代次数
{
    /* defined in bcrypt_pbkdf.c */
    # 调用bcrypt_pbkdf函数计算PBKDF值并返回结果
    return bcrypt_pbkdf(pass,
                        passlen,
                        salt,
                        saltlen,
                        key,
                        keylen,
                        rounds);
}
```