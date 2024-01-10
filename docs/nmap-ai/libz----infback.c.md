# `nmap\libz\infback.c`

```
/* infback.c -- inflate using a call-back interface
 * Copyright (C) 1995-2022 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */
# infback.c -- 使用回调接口进行解压缩
# 版权所有 1995-2022 Mark Adler
# 分发和使用条件，请参阅 zlib.h 中的版权声明

/*
   This code is largely copied from inflate.c.  Normally either infback.o or
   inflate.o would be linked into an application--not both.  The interface
   with inffast.c is retained so that optimized assembler-coded versions of
   inflate_fast() can be used with either inflate.c or infback.c.
 */
# 这段代码大部分是从 inflate.c 复制而来。通常情况下，应用程序中只链接 infback.o 或 inflate.o 中的一个，而不是两者都链接。保留与 inffast.c 的接口，以便可以将优化的汇编版本的 inflate_fast() 与 inflate.c 或 infback.c 一起使用。

#include "zutil.h"
#include "inftrees.h"
#include "inflate.h"
#include "inffast.h"

/* function prototypes */
local void fixedtables OF((struct inflate_state FAR *state));
# 函数原型
# 定义了一个名为 fixedtables 的函数，参数为指向结构体 inflate_state 的指针

/*
   strm provides memory allocation functions in zalloc and zfree, or
   Z_NULL to use the library memory allocation functions.

   windowBits is in the range 8..15, and window is a user-supplied
   window and output buffer that is 2**windowBits bytes.
 */
# strm 提供 zalloc 和 zfree 中的内存分配函数，或者使用库内存分配函数的 Z_NULL。
# windowBits 在范围 8..15 之间，window 是用户提供的窗口和输出缓冲区，大小为 2**windowBits 字节。

int ZEXPORT inflateBackInit_(strm, windowBits, window, version, stream_size)
z_streamp strm;
int windowBits;
unsigned char FAR *window;
const char *version;
int stream_size;
{
    struct inflate_state FAR *state;

    if (version == Z_NULL || version[0] != ZLIB_VERSION[0] ||
        stream_size != (int)(sizeof(z_stream)))
        return Z_VERSION_ERROR;
    if (strm == Z_NULL || window == Z_NULL ||
        windowBits < 8 || windowBits > 15)
        return Z_STREAM_ERROR;
    strm->msg = Z_NULL;                 /* in case we return an error */
    if (strm->zalloc == (alloc_func)0) {
#ifdef Z_SOLO
        return Z_STREAM_ERROR;
#else
        strm->zalloc = zcalloc;
        strm->opaque = (voidpf)0;
#endif
    }
    if (strm->zfree == (free_func)0)
#ifdef Z_SOLO
        return Z_STREAM_ERROR;
#else
    strm->zfree = zcfree;
#endif
    state = (struct inflate_state FAR *)ZALLOC(strm, 1,
                                               sizeof(struct inflate_state));
    if (state == Z_NULL) return Z_MEM_ERROR;
    Tracev((stderr, "inflate: allocated\n"));
# 初始化状态变量 state，如果分配内存失败则返回内存错误
    # 将指针 strm 指向的 state 结构体设置为 state 结构体的地址
    strm->state = (struct internal_state FAR *)state;
    # 设置 state 结构体的 dmax 属性为 32768
    state->dmax = 32768U;
    # 将 windowBits 赋值给 state 结构体的 wbits 属性
    state->wbits = (uInt)windowBits;
    # 将 2 的 windowBits 次方赋值给 state 结构体的 wsize 属性
    state->wsize = 1U << windowBits;
    # 将 window 赋值给 state 结构体的 window 属性
    state->window = window;
    # 将 0 赋值给 state 结构体的 wnext 属性
    state->wnext = 0;
    # 将 0 赋值给 state 结构体的 whave 属性
    state->whave = 0;
    # 将 1 赋值给 state 结构体的 sane 属性
    state->sane = 1;
    # 返回 Z_OK 表示函数执行成功
    return Z_OK;
}
/*
   返回具有长度和距离解码表以及索引大小设置为固定代码解码的状态。
   通常情况下，这将从inffixed.h返回固定表。
   如果定义了BUILDFIXED，则此例程首次调用时构建表，并在首次调用后返回这些表。
   这将减少约2K字节的代码大小，换取一点执行时间。
   但是，不应该在多线程应用程序中使用BUILDFIXED，因为表和virgin的重写可能不是线程安全的。
 */
local void fixedtables(state)
struct inflate_state FAR *state;
{
#ifdef BUILDFIXED
    static int virgin = 1;
    static code *lenfix, *distfix;
    static code fixed[544];

    /* 如果是第一次调用，则构建固定的哈夫曼表（可能不是线程安全的） */
    if (virgin) {
        unsigned sym, bits;
        static code *next;

        /* 字面/长度表 */
        sym = 0;
        while (sym < 144) state->lens[sym++] = 8;
        while (sym < 256) state->lens[sym++] = 9;
        while (sym < 280) state->lens[sym++] = 7;
        while (sym < 288) state->lens[sym++] = 8;
        next = fixed;
        lenfix = next;
        bits = 9;
        inflate_table(LENS, state->lens, 288, &(next), &(bits), state->work);

        /* 距离表 */
        sym = 0;
        while (sym < 32) state->lens[sym++] = 5;
        distfix = next;
        bits = 5;
        inflate_table(DISTS, state->lens, 32, &(next), &(bits), state->work);

        /* 只执行一次 */
        virgin = 0;
    }
#else /* !BUILDFIXED */
#   include "inffixed.h"
#endif /* BUILDFIXED */
    state->lencode = lenfix;
    state->lenbits = 9;
    state->distcode = distfix;
    state->distbits = 5;
}
/* inflateBack()的宏： */

/* 从inflate_fast()加载返回的状态 */
#define LOAD() \
    # 使用 do-while 循环，目的是为了定义一段代码块，不管条件是否成立都会执行一次
    do { \
        # 将指针 strm->next_out 的值赋给 put
        put = strm->next_out; \
        # 将指针 strm->avail_out 的值赋给 left
        left = strm->avail_out; \
        # 将指针 strm->next_in 的值赋给 next
        next = strm->next_in; \
        # 将指针 strm->avail_in 的值赋给 have
        have = strm->avail_in; \
        # 将状态 state->hold 的值赋给 hold
        hold = state->hold; \
        # 将状态 state->bits 的值赋给 bits
        bits = state->bits; \
    } while (0)
/* 为 inflate_fast() 函数从寄存器中设置状态 */
#define RESTORE() \
    do { \
        strm->next_out = put; \  // 设置输出数据的指针
        strm->avail_out = left; \  // 设置输出数据的剩余空间
        strm->next_in = next; \  // 设置输入数据的指针
        strm->avail_in = have; \  // 设置输入数据的剩余空间
        state->hold = hold; \  // 恢复 hold 变量的值
        state->bits = bits; \  // 恢复 bits 变量的值
    } while (0)

/* 清空输入位累加器 */
#define INITBITS() \
    do { \
        hold = 0; \  // 将 hold 变量清零
        bits = 0; \  // 将 bits 变量清零
    } while (0)

/* 确保有可用的输入数据。如果请求了输入但被拒绝，则从 inflateBack() 返回 Z_BUF_ERROR。 */
#define PULL() \
    do { \
        if (have == 0) { \  // 如果没有剩余的输入数据
            have = in(in_desc, &next); \  // 从输入描述符中获取输入数据
            if (have == 0) { \  // 如果获取的输入数据为空
                next = Z_NULL; \  // 将 next 指针置为空
                ret = Z_BUF_ERROR; \  // 将返回值设置为 Z_BUF_ERROR
                goto inf_leave; \  // 跳转到 inf_leave 标签处
            } \
        } \
    } while (0)

/* 将一个字节的输入数据放入位累加器中，如果没有可用的输入数据则从 inflateBack() 返回错误。 */
#define PULLBYTE() \
    do { \
        PULL(); \  // 确保有可用的输入数据
        have--; \  // 剩余的输入数据减一
        hold += (unsigned long)(*next++) << bits; \  // 将输入数据放入位累加器中
        bits += 8; \  // 更新位累加器的位数
    } while (0)

/* 确保位累加器中至少有 n 位。如果没有足够的可用输入数据，则从 inflateBack() 返回错误。 */
#define NEEDBITS(n) \
    do { \
        while (bits < (unsigned)(n)) \  // 当位累加器中的位数小于 n 时
            PULLBYTE(); \  // 从输入数据中获取足够的位数
    } while (0)

/* 返回位累加器中的低 n 位 (n < 16) */
#define BITS(n) \
    ((unsigned)hold & ((1U << (n)) - 1))  // 返回位累加器中的低 n 位

/* 从位累加器中移除 n 位 */
#define DROPBITS(n) \
    do { \
        hold >>= (n); \  // 将位累加器中的值右移 n 位
        bits -= (unsigned)(n); \  // 更新位累加器中的位数
    } while (0)

/* 根据需要移除零到七位，以便到达字节边界 */
#define BYTEBITS() \
    do { \
        hold >>= bits & 7; \  // 将位累加器中的值右移至字节边界
        bits -= bits & 7; \  // 更新位累加器中的位数至字节边界
    } while (0)
/* 确保有足够的输出空间，如果窗口已满则写出。如果写入失败，则在inflateBack()中返回Z_BUF_ERROR。 */
#define ROOM() \
    do { \
        if (left == 0) { \  // 如果剩余空间为0
            put = state->window; \  // 将窗口的起始位置赋给put
            left = state->wsize; \  // 将窗口大小赋给left
            state->whave = left; \  // 更新已使用的窗口大小
            if (out(out_desc, put, left)) { \  // 如果写出失败
                ret = Z_BUF_ERROR; \  // 返回Z_BUF_ERROR
                goto inf_leave; \  // 跳转到inf_leave标签
            } \
        } \
    } while (0)  // 结束do-while循环
/*
   strm提供内存分配函数和输入的窗口缓冲区，并在返回时提供未使用的输入信息。对于Z_DATA_ERROR返回，strm还将提供错误消息。

   in()和out()是回调输入和输出函数。当inflateBack()需要更多输入时，它调用in()。当inflateBack()用输出填充窗口，或者当它完成窗口中的数据时，它调用out()来写出数据。应用程序在再次调用in()或inflateBack()返回之前不得更改提供的输入。应用程序在inflateBack()返回之前不得更改窗口/输出缓冲区。

   in()和out()在inflateBack()调用中使用提供的描述符参数。此参数可以是一个结构，提供执行读取或写入所需的信息，以及输入和输出的累积信息，如总数和校验值。

   in()在失败时应返回零。out()在失败时应返回非零。如果in()或out()中的任何一个失败，则inflateBack()返回Z_BUF_ERROR。可以检查strm->next_in是否为Z_NULL，以查看是in()还是out()导致了错误。否则，inflateBack()在成功时返回Z_STREAM_END，在deflate格式错误时返回Z_DATA_ERROR，如果无法为状态分配内存，则返回Z_MEM_ERROR。如果输入参数不正确，即strm为Z_NULL或状态未初始化，则inflateBack()还可以返回Z_STREAM_ERROR。
 */
int ZEXPORT inflateBack(strm, in, in_desc, out, out_desc)
z_streamp strm;
in_func in;
void FAR *in_desc;
out_func out;
void FAR *out_desc;
{
    struct inflate_state FAR *state;
    z_const unsigned char FAR *next;    /* 下一个输入 */
    unsigned char FAR *put;     /* 下一个输出 */
    unsigned have, left;        /* 可用输入和输出 */
    unsigned long hold;         /* 位缓冲 */
    # 位缓冲区中的位数
    unsigned bits;
    # 要复制的存储或匹配字节数
    unsigned copy;
    # 要从哪里复制匹配字节
    unsigned char FAR *from;
    # 当前解码表条目
    code here;
    # 父表条目
    code last;
    # 重复的长度，要丢弃的位数
    unsigned len;
    # 返回代码
    int ret;
    # 长度为19的无符号短整型数组，用于排列代码长度
    static const unsigned short order[19] = 
        {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};

    # 检查 strm 是否存在，以及状态是否已初始化
    if (strm == Z_NULL || strm->state == Z_NULL)
        return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)strm->state;

    # 重置状态
    strm->msg = Z_NULL;
    state->mode = TYPE;
    state->last = 0;
    state->whave = 0;
    next = strm->next_in;
    have = next != Z_NULL ? strm->avail_in : 0;
    hold = 0;
    bits = 0;
    put = state->window;
    left = state->wsize;

    # 解压缩直到标记为最后的块结束
#ifndef PKZIP_BUG_WORKAROUND
            // 如果长度或距离符号过多，则设置错误消息并跳出循环
            if (state->nlen > 286 || state->ndist > 30) {
                strm->msg = (char *)"too many length or distance symbols";
                state->mode = BAD;
                break;
            }
    /* Write leftover output and return unused input */
  inf_leave:
    // 如果还有剩余的输出，则写入并返回未使用的输入
    if (left < state->wsize) {
        if (out(out_desc, state->window, state->wsize - left) &&
            ret == Z_STREAM_END)
            ret = Z_BUF_ERROR;
    }
    // 重置输入流指针和可用输入大小，然后返回结果
    strm->next_in = next;
    strm->avail_in = have;
    return ret;
}

int ZEXPORT inflateBackEnd(strm)
z_streamp strm;
{
    // 检查输入流和状态是否有效
    if (strm == Z_NULL || strm->state == Z_NULL || strm->zfree == (free_func)0)
        return Z_STREAM_ERROR;
    // 释放状态内存并重置状态指针
    ZFREE(strm, strm->state);
    strm->state = Z_NULL;
    // 输出结束消息并返回成功状态
    Tracev((stderr, "inflate: end\n"));
    return Z_OK;
}
```