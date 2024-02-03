# `nmap\libz\inflate.c`

```cpp
/* inflate.c -- zlib decompression
 * zlib解压缩的代码文件
 * 版权所有 (C) 1995-2022 Mark Adler
 * 分发和使用条件，请参见zlib.h中的版权声明
 */

#include "zutil.h"
#include "inftrees.h"
#include "inflate.h"
#include "inffast.h"

#ifdef MAKEFIXED
#  ifndef BUILDFIXED
#    define BUILDFIXED
#  endif
#endif

/* 函数原型 */
local int inflateStateCheck OF((z_streamp strm));
local void fixedtables OF((struct inflate_state FAR *state));
local int updatewindow OF((z_streamp strm, const unsigned char FAR *end,
                           unsigned copy));
#ifdef BUILDFIXED
   void makefixed OF((void));
#endif
local unsigned syncsearch OF((unsigned FAR *have, const unsigned char FAR *buf,
                              unsigned len));

/* 检查解压缩状态的函数 */
local int inflateStateCheck(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;
    if (strm == Z_NULL ||
        strm->zalloc == (alloc_func)0 || strm->zfree == (free_func)0)
        return 1;
    state = (struct inflate_state FAR *)strm->state;
    if (state == Z_NULL || state->strm != strm ||
        state->mode < HEAD || state->mode > SYNC)
        return 1;
    return 0;
}

/* 重置解压缩状态，保持已分配的内存 */
int ZEXPORT inflateResetKeep(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;

    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)strm->state;
    strm->total_in = strm->total_out = state->total = 0;
    strm->msg = Z_NULL;
    if (state->wrap)        /* to support ill-conceived Java test suite */
        strm->adler = state->wrap & 1;
    state->mode = HEAD;
    state->last = 0;
    state->havedict = 0;
    state->flags = -1;
    state->dmax = 32768U;
    state->head = Z_NULL;
    state->hold = 0;
    state->bits = 0;
    state->lencode = state->distcode = state->next = state->codes;
    state->sane = 1;
    state->back = -1;
    Tracev((stderr, "inflate: reset\n"));
    return Z_OK;
}

/* 重置解压缩状态 */
int ZEXPORT inflateReset(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;
    # 检查压缩状态，如果不符合要求则返回 Z_STREAM_ERROR
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    # 将压缩状态转换为 inflate_state 结构体类型
    state = (struct inflate_state FAR *)strm->state;
    # 重置压缩状态的窗口大小为 0
    state->wsize = 0;
    # 重置压缩状态的已有数据大小为 0
    state->whave = 0;
    # 重置压缩状态的下一个数据位置为 0
    state->wnext = 0;
    # 调用函数进行保持压缩状态的重置
    return inflateResetKeep(strm);
# 重置解压缩器的状态，以便重新开始解压缩过程
int ZEXPORT inflateReset2(strm, windowBits)
z_streamp strm;
int windowBits;
{
    int wrap;
    struct inflate_state FAR *state;

    # 检查解压缩器状态是否正常
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    # 获取解压缩器的状态
    state = (struct inflate_state FAR *)strm->state;

    # 从 windowBits 参数中提取 wrap 请求
    if (windowBits < 0) {
        if (windowBits < -15)
            return Z_STREAM_ERROR;
        wrap = 0;
        windowBits = -windowBits;
    }
    else {
        wrap = (windowBits >> 4) + 5;
        # 如果定义了 GUNZIP，则对 windowBits 进行处理
        #ifdef GUNZIP
        if (windowBits < 48)
            windowBits &= 15;
        #endif
    }

    # 设置窗口位数，如果不同则释放窗口
    if (windowBits && (windowBits < 8 || windowBits > 15))
        return Z_STREAM_ERROR;
    if (state->window != Z_NULL && state->wbits != (unsigned)windowBits) {
        ZFREE(strm, state->window);
        state->window = Z_NULL;
    }

    # 更新状态并重置其余部分
    state->wrap = wrap;
    state->wbits = (unsigned)windowBits;
    return inflateReset(strm);
}

# 初始化解压缩器的状态
int ZEXPORT inflateInit2_(strm, windowBits, version, stream_size)
z_streamp strm;
int windowBits;
const char *version;
int stream_size;
{
    int ret;
    struct inflate_state FAR *state;

    # 检查版本和流大小是否匹配
    if (version == Z_NULL || version[0] != ZLIB_VERSION[0] ||
        stream_size != (int)(sizeof(z_stream)))
        return Z_VERSION_ERROR;
    # 检查解压缩器状态是否正常
    if (strm == Z_NULL) return Z_STREAM_ERROR;
    strm->msg = Z_NULL;                 /* in case we return an error */
    # 如果未指定内存分配函数，则使用默认函数
    if (strm->zalloc == (alloc_func)0) {
        #ifdef Z_SOLO
        return Z_STREAM_ERROR;
        #else
        strm->zalloc = zcalloc;
        strm->opaque = (voidpf)0;
        #endif
    }
    # 如果未指定内存释放函数，则使用默认函数
    if (strm->zfree == (free_func)0)
        #ifdef Z_SOLO
        return Z_STREAM_ERROR;
        #else
        strm->zfree = zcfree;
        #endif
    # 分配内存给解压缩器状态
    state = (struct inflate_state FAR *)
            ZALLOC(strm, 1, sizeof(struct inflate_state));
    if (state == Z_NULL) return Z_MEM_ERROR;
    Tracev((stderr, "inflate: allocated\n"));
}
    # 将结构体指针 state 赋值给 strm 的 state 成员
    strm->state = (struct internal_state FAR *)state;
    # 将 strm 赋值给 state 的 strm 成员
    state->strm = strm;
    # 将 Z_NULL 赋值给 state 的 window 成员
    state->window = Z_NULL;
    # 将 HEAD 赋值给 state 的 mode 成员，用于在 inflateReset2() 中通过状态测试
    state->mode = HEAD;     /* to pass state test in inflateReset2() */
    # 调用 inflateReset2() 函数，传入 strm 和 windowBits 参数，将返回值赋值给 ret
    ret = inflateReset2(strm, windowBits);
    # 如果返回值不等于 Z_OK
    if (ret != Z_OK) {
        # 释放 strm 和 state 的内存
        ZFREE(strm, state);
        # 将 Z_NULL 赋值给 strm 的 state 成员
        strm->state = Z_NULL;
    }
    # 返回 ret
    return ret;
}



int ZEXPORT inflateInit_(strm, version, stream_size)
z_streamp strm;
const char *version;
int stream_size;
{
    // 调用 inflateInit2_ 函数，使用默认的窗口大小
    return inflateInit2_(strm, DEF_WBITS, version, stream_size);
}



int ZEXPORT inflatePrime(strm, bits, value)
z_streamp strm;
int bits;
int value;
{
    struct inflate_state FAR *state;

    // 检查解压状态是否正常
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 获取解压状态
    state = (struct inflate_state FAR *)strm->state;
    // 如果 bits 小于 0，则重置 hold 和 bits，并返回 Z_OK
    if (bits < 0) {
        state->hold = 0;
        state->bits = 0;
        return Z_OK;
    }
    // 如果 bits 大于 16 或者 state->bits + bits 大于 32，则返回 Z_STREAM_ERROR
    if (bits > 16 || state->bits + (uInt)bits > 32) return Z_STREAM_ERROR;
    // 对 value 进行位运算，保留低 bits 位的值
    value &= (1L << bits) - 1;
    // 将 value 左移 state->bits 位，然后加到 hold 中
    state->hold += (unsigned)value << state->bits;
    // 更新 state->bits
    state->bits += (uInt)bits;
    return Z_OK;
}



/*
   Return state with length and distance decoding tables and index sizes set to
   fixed code decoding.  Normally this returns fixed tables from inffixed.h.
   If BUILDFIXED is defined, then instead this routine builds the tables the
   first time it's called, and returns those tables the first time and
   thereafter.  This reduces the size of the code by about 2K bytes, in
   exchange for a little execution time.  However, BUILDFIXED should not be
   used for threaded applications, since the rewriting of the tables and virgin
   may not be thread-safe.
 */
local void fixedtables(state)
struct inflate_state FAR *state;
{
    // 如果 BUILDFIXED 被定义，则构建固定的哈夫曼表
    #ifdef BUILDFIXED
    static int virgin = 1;
    static code *lenfix, *distfix;
    static code fixed[544];

    /* build fixed huffman tables if first call (may not be thread safe) */
    // 如果是第一次调用，则构建固定的哈夫曼表
    ```
    # 如果是第一次执行该代码块
    if (virgin) {
        # 定义无符号整型变量 sym 和 bits，以及静态指针 next
        unsigned sym, bits;
        static code *next;

        # literal/length 表
        sym = 0;
        while (sym < 144) state->lens[sym++] = 8;
        while (sym < 256) state->lens[sym++] = 9;
        while (sym < 280) state->lens[sym++] = 7;
        while (sym < 288) state->lens[sym++] = 8;
        next = fixed;
        lenfix = next;
        bits = 9;
        # 调用 inflate_table 函数，生成 literal/length 表
        inflate_table(LENS, state->lens, 288, &(next), &(bits), state->work);

        # distance 表
        sym = 0;
        while (sym < 32) state->lens[sym++] = 5;
        distfix = next;
        bits = 5;
        # 调用 inflate_table 函数，生成 distance 表
        inflate_table(DISTS, state->lens, 32, &(next), &(bits), state->work);

        # 只执行一次的操作，将 virgin 置为 0
        virgin = 0;
    }
#else /* !BUILDFIXED */
#   include "inffixed.h"
#endif /* BUILDFIXED */
    // 如果没有定义 BUILDFIXED，则包含 inffixed.h 文件
    state->lencode = lenfix;
    // 将 lenfix 赋值给 state->lencode
    state->lenbits = 9;
    // 将 9 赋值给 state->lenbits
    state->distcode = distfix;
    // 将 distfix 赋值给 state->distcode
    state->distbits = 5;
    // 将 5 赋值给 state->distbits
}

#ifdef MAKEFIXED
#include <stdio.h>

/*
   Write out the inffixed.h that is #include'd above.  Defining MAKEFIXED also
   defines BUILDFIXED, so the tables are built on the fly.  makefixed() writes
   those tables to stdout, which would be piped to inffixed.h.  A small program
   can simply call makefixed to do this:

    void makefixed(void);

    int main(void)
    {
        makefixed();
        return 0;
    }

   Then that can be linked with zlib built with MAKEFIXED defined and run:

    a.out > inffixed.h
 */
void makefixed()
{
    unsigned low, size;
    struct inflate_state state;

    fixedtables(&state);
    // 调用 fixedtables 函数，传入 state 结构体的地址
    puts("    /* inffixed.h -- table for decoding fixed codes");
    puts("     * Generated automatically by makefixed().");
    puts("     */");
    puts("");
    puts("    /* WARNING: this file should *not* be used by applications.");
    puts("       It is part of the implementation of this library and is");
    puts("       subject to change. Applications should only use zlib.h.");
    puts("     */");
    puts("");
    size = 1U << 9;
    // 将 1 左移 9 位的结果赋值给 size
    printf("    static const code lenfix[%u] = {", size);
    // 输出静态数组 lenfix 的声明
    low = 0;
    for (;;) {
        if ((low % 7) == 0) printf("\n        ");
        // 如果 low 是 7 的倍数，则输出换行和缩进
        printf("{%u,%u,%d}", (low & 127) == 99 ? 64 : state.lencode[low].op,
               state.lencode[low].bits, state.lencode[low].val);
        // 输出静态数组 lenfix 的元素
        if (++low == size) break;
        putchar(',');
    }
    puts("\n    };");
    size = 1U << 5;
    // 将 1 左移 5 位的结果赋值给 size
    printf("\n    static const code distfix[%u] = {", size);
    // 输出静态数组 distfix 的声明
    low = 0;
    for (;;) {
        if ((low % 6) == 0) printf("\n        ");
        // 如果 low 是 6 的倍数，则输出换行和缩进
        printf("{%u,%u,%d}", state.distcode[low].op, state.distcode[low].bits,
               state.distcode[low].val);
        // 输出静态数组 distfix 的元素
        if (++low == size) break;
        putchar(',');
    }
    puts("\n    };");
}
#endif /* MAKEFIXED */

'''
   在返回之前，使用最后写入的 wsize（通常为32K）字节更新窗口。如果窗口尚不存在，则创建它。只有在窗口已经在使用中，或者在此次解压缩调用期间已经写入输出，但尚未到达压缩流的末尾时才会调用此函数。还会在加载字典数据时创建窗口。

   将大于32K的输出缓冲区提供给inflate()应该会提供速度优势，因为只有在从inflate()返回时将最后32K的输出复制到滑动窗口中，而且自第一个32K输出之后的所有距离都将落在输出数据中，使得匹配复制更简单更快。这种优势可能取决于处理器数据缓存的大小。
'''
local int updatewindow(strm, end, copy)
z_streamp strm;
const Bytef *end;
unsigned copy;
{
    struct inflate_state FAR *state;
    unsigned dist;

    state = (struct inflate_state FAR *)strm->state;

    /* 如果尚未完成，为窗口分配空间 */
    if (state->window == Z_NULL) {
        state->window = (unsigned char FAR *)
                        ZALLOC(strm, 1U << state->wbits,
                               sizeof(unsigned char));
        if (state->window == Z_NULL) return 1;
    }

    /* 如果窗口尚未使用，进行初始化 */
    if (state->wsize == 0) {
        state->wsize = 1U << state->wbits;
        state->wnext = 0;
        state->whave = 0;
    }

    /* 将state->wsize或更少的输出字节复制到循环窗口中 */
    if (copy >= state->wsize) {
        zmemcpy(state->window, end - state->wsize, state->wsize);
        state->wnext = 0;
        state->whave = state->wsize;
    }
    # 如果条件不成立，则执行以下操作
    else {
        # 计算可复制的数据长度
        dist = state->wsize - state->wnext;
        # 如果可复制的数据长度大于指定长度，则取指定长度
        if (dist > copy) dist = copy;
        # 复制数据到窗口中
        zmemcpy(state->window + state->wnext, end - copy, dist);
        # 更新剩余需要复制的数据长度
        copy -= dist;
        # 如果还有剩余数据需要复制
        if (copy) {
            # 继续复制数据到窗口中
            zmemcpy(state->window, end - copy, copy);
            # 更新下一个写入位置
            state->wnext = copy;
            # 更新已经写入的数据长度
            state->whave = state->wsize;
        }
        # 如果没有剩余数据需要复制
        else {
            # 更新下一个写入位置
            state->wnext += dist;
            # 如果下一个写入位置等于窗口大小，则重置为0
            if (state->wnext == state->wsize) state->wnext = 0;
            # 如果已经写入的数据长度小于窗口大小，则更新已写入的数据长度
            if (state->whave < state->wsize) state->whave += dist;
        }
    }
    # 返回0
    return 0;
/* Macros for inflate(): */

/* 定义一个宏，根据是否为 GUNZIP，选择使用 adler32() 还是 crc32() 来进行校验 */
#ifdef GUNZIP
#  define UPDATE_CHECK(check, buf, len) \
    (state->flags ? crc32(check, buf, len) : adler32(check, buf, len))
#else
#  define UPDATE_CHECK(check, buf, len) adler32(check, buf, len)
#endif

/* 定义一个宏，根据是否为 GUNZIP，选择使用 crc2() 还是 crc4() 来进行头部校验 */
#ifdef GUNZIP
#  define CRC2(check, word) \
    do { \
        hbuf[0] = (unsigned char)(word); \
        hbuf[1] = (unsigned char)((word) >> 8); \
        check = crc32(check, hbuf, 2); \
    } while (0)

#  define CRC4(check, word) \
    do { \
        hbuf[0] = (unsigned char)(word); \
        hbuf[1] = (unsigned char)((word) >> 8); \
        hbuf[2] = (unsigned char)((word) >> 16); \
        hbuf[3] = (unsigned char)((word) >> 24); \
        check = crc32(check, hbuf, 4); \
    } while (0)
#endif

/* 定义一个宏，用于在 inflate() 中加载寄存器以提高速度 */
#define LOAD() \
    do { \
        put = strm->next_out; \
        left = strm->avail_out; \
        next = strm->next_in; \
        have = strm->avail_in; \
        hold = state->hold; \
        bits = state->bits; \
    } while (0)

/* 定义一个宏，用于在 inflate() 中从寄存器中恢复状态 */
#define RESTORE() \
    do { \
        strm->next_out = put; \
        strm->avail_out = left; \
        strm->next_in = next; \
        strm->avail_in = have; \
        state->hold = hold; \
        state->bits = bits; \
    } while (0)

/* 定义一个宏，用于在输入位累加器中清除位 */
#define INITBITS() \
    do { \
        hold = 0; \
        bits = 0; \
    } while (0)

/* 定义一个宏，用于将输入的一个字节加载到位累加器中，如果没有可用输入则返回 */
#define PULLBYTE() \
    do { \
        if (have == 0) goto inf_leave; \
        have--; \
        hold += (unsigned long)(*next++) << bits; \
        bits += 8; \
    } while (0}

/* 确保位累加器中至少有 n 位。如果没有足够的可用输入，则从 inflate() 返回 */
# 定义宏，用于确保位操作时有足够的比特数
#define NEEDBITS(n) \
    do { \
        while (bits < (unsigned)(n)) \  # 当比特数小于指定值时，继续读取字节
            PULLBYTE(); \  # 调用PULLBYTE()函数
    } while (0)

# 返回比特累加器的低n位（n < 16）
#define BITS(n) \
    ((unsigned)hold & ((1U << (n)) - 1))  # 返回比特累加器的低n位

# 从比特累加器中移除n位
#define DROPBITS(n) \
    do { \
        hold >>= (n); \  # 将比特累加器右移n位
        bits -= (unsigned)(n); \  # 减去移除的比特数
    } while (0)

# 根据需要移除零到七位，以便到达字节边界
#define BYTEBITS() \
    do { \
        hold >>= bits & 7; \  # 将比特累加器右移至字节边界
        bits -= bits & 7; \  # 减去移除的比特数
    } while (0)

'''
   inflate()使用状态机在返回之前处理尽可能多的输入数据并生成尽可能多的输出数据。状态机的结构大致如下：

    for (;;) switch (state) {
    ...
    case STATEn:
        if (not enough input data or output space to make progress)
            return;
        ... make progress ...
        state = STATEm;
        break;
    ...
'''  # 描述了inflate()函数使用的状态机结构
    # 当再次调用inflate()时，尝试相同的情况，如果提供了适当的资源，机器就会继续到下一个状态
    # NEEDBITS()宏通常是状态评估是否可以继续或应该返回的方式
    # 如果请求的位数不可用，NEEDBITS()会返回
    # BITS宏的典型用法是：
    #     NEEDBITS(n);
    #     ... 使用BITS(n)做一些操作 ...
    #     DROPBITS(n);
    # 其中，如果没有足够的输入剩下来加载n位到累加器中，NEEDBITS(n)要么从inflate()返回，要么继续
    # BITS(n)给出累加器中的低n位
    # 完成后，DROPBITS(n)从累加器中删除低n位
    # INITBITS()清除累加器并将可用位数设置为零
    # BYTEBITS()丢弃足够的位数，使累加器处于字节边界上
    # 在BYTEBITS()和NEEDBITS(8)之后，BITS(8)将返回流中的下一个字节
    # NEEDBITS(n)使用PULLBYTE()获取可用的输入字节，或者如果没有可用输入，则返回
    # 变长编码的解码直接使用PULLBYTE()，以便仅获取足够的字节来解码下一个代码，而不多
    # 一些状态会循环直到获得足够的输入，确保保持足够的状态信息以便在循环中断时继续
    # 例如，如果NEEDBITS()在循环中返回，want、need和keep都必须实际上是保存状态的一部分：
    # case STATEw:
    #     while (want < need) {
    #         NEEDBITS(n);
    #         keep[want++] = BITS(n);
    #         DROPBITS(n);
    #     }
    #     state = STATEx;
    # 如果下一个状态也是下一个 case，则省略 break
    # 如果状态返回，可能是因为没有足够的输出空间来完成该状态。这些状态包括复制存储的数据、写入一个字面字节和复制匹配的字符串。
    # 返回时，使用 "goto inf_leave" 来更新总计数器、更新校验值，并确定在 inflate() 调用期间是否有任何进展，以便返回正确的返回码。进展被定义为 strm->avail_in 或 strm->avail_out 的变化。
    # 当存在窗口时，goto inf_leave 将使用最后写入的输出更新窗口。如果在解压缩过程中出现 goto inf_leave，并且当前没有窗口，则 goto inf_leave 将创建一个窗口，并将输出复制到窗口，以便下一次调用 inflate()。
    # 在这个实现中，inflate() 的 flush 参数只影响返回码（根据 zlib.h）。inflate() 总是尽可能多地写入到 strm->next_out，给定的空间和提供的输入--这是在 zlib.h 中 Z_SYNC_FLUSH 的文档效果。此外，inflate() 总是推迟分配和复制到滑动窗口，直到必要时，这提供了在整个输入流可用时，Z_FINISH 的 zlib.h 文档效果。因此，flush 参数实际上只做了一件事：当 flush 设置为 Z_FINISH 时，如果尚未到达流的末尾，inflate() 将无法返回 Z_OK。相反，如果尚未到达流的末尾，它将返回 Z_BUF_ERROR。
# 定义函数，用于解压缩数据
int ZEXPORT inflate(strm, flush)
z_streamp strm;
int flush;
{
    # 定义指向解压缩状态的指针
    struct inflate_state FAR *state;
    # 指向下一个输入数据的指针
    z_const unsigned char FAR *next;    /* next input */
    # 指向下一个输出数据的指针
    unsigned char FAR *put;     /* next output */
    # 可用的输入和输出数据量
    unsigned have, left;        /* available input and output */
    # 位缓冲区和其中的位数
    unsigned long hold;         /* bit buffer */
    unsigned bits;              /* bits in bit buffer */
    # 保存起始的可用输入和输出数据
    unsigned in, out;           /* save starting available input and output */
    # 要复制的存储或匹配字节数
    unsigned copy;              /* number of stored or match bytes to copy */
    # 从哪里复制匹配字节
    unsigned char FAR *from;    /* where to copy match bytes from */
    # 当前解码表条目
    code here;                  /* current decoding table entry */
    # 父解码表条目
    code last;                  /* parent table entry */
    # 重复的长度，要丢弃的位数
    unsigned len;               /* length to copy for repeats, bits to drop */
    # 返回代码
    int ret;                    /* return code */
    # 如果是解压缩 GZIP 格式的数据，用于计算头部 CRC 的缓冲区
#ifdef GUNZIP
    unsigned char hbuf[4];      /* buffer for gzip header crc calculation */
#endif
    # 长度为 19 的无符号短整型数组，用于排列码长
    static const unsigned short order[19] = /* permutation of code lengths */
        {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};

    # 检查解压缩状态和输入输出指针是否有效
    if (inflateStateCheck(strm) || strm->next_out == Z_NULL ||
        (strm->next_in == Z_NULL && strm->avail_in != 0))
        return Z_STREAM_ERROR;

    # 将解压缩状态指针转换为 inflate_state 类型
    state = (struct inflate_state FAR *)strm->state;
    # 如果解压缩模式为 TYPE，则将其设置为 TYPEDO，跳过检查
    if (state->mode == TYPE) state->mode = TYPEDO;      /* skip check */
    # 载入输入数据
    LOAD();
    in = have;
    out = left;
    ret = Z_OK;
    # 无限循环，处理解压缩过程
    for (;;)
        switch (state->mode) {
        # 头部解压缩模式
        case HEAD:
            # 如果是不包含头部的数据，则将模式设置为 TYPEDO
            if (state->wrap == 0) {
                state->mode = TYPEDO;
                break;
            }
            # 需要 16 位的比特数据
            NEEDBITS(16);
#ifdef GUNZIP
            # 如果启用了 GUNZIP，且 hold 的值为 0x8b1f，表示为 gzip 头部
            if ((state->wrap & 2) && hold == 0x8b1f) {  /* gzip header */
                # 如果当前的窗口位数为 0，则设置为 15
                if (state->wbits == 0)
                    state->wbits = 15;
                # 初始化校验和为 0，计算 gzip 头部的 CRC 校验和
                state->check = crc32(0L, Z_NULL, 0);
                CRC2(state->check, hold);
                INITBITS();
                # 设置解压模式为 FLAGS
                state->mode = FLAGS;
                break;
            }
            # 如果 state->head 不为空，则将其 done 属性设置为 -1
            if (state->head != Z_NULL)
                state->head->done = -1;
            # 如果未启用 zlib 头部，或者校验失败，则设置解压模式为 BAD
            if (!(state->wrap & 1) ||   /* check if zlib header allowed */
#else
            # 如果未启用 GUNZIP，则执行以下代码
            if (
#endif
                # 检查头部的校验和是否正确
                ((BITS(8) << 8) + (hold >> 8)) % 31) {
                strm->msg = (char *)"incorrect header check";
                state->mode = BAD;
                break;
            }
            # 检查压缩方法是否为 Z_DEFLATED
            if (BITS(4) != Z_DEFLATED) {
                strm->msg = (char *)"unknown compression method";
                state->mode = BAD;
                break;
            }
            # 丢弃 4 位
            DROPBITS(4);
            # 读取长度信息，计算窗口位数
            len = BITS(4) + 8;
            if (state->wbits == 0)
                state->wbits = len;
            # 检查窗口大小是否合法
            if (len > 15 || len > state->wbits) {
                strm->msg = (char *)"invalid window size";
                state->mode = BAD;
                break;
            }
            # 设置最大距离值
            state->dmax = 1U << len;
            # 设置标志位为 0，表示为 zlib 头部
            state->flags = 0;               /* indicate zlib header */
            Tracev((stderr, "inflate:   zlib header ok\n"));
            # 计算 adler32 校验和，设置解压模式为 DICTID 或 TYPE
            strm->adler = state->check = adler32(0L, Z_NULL, 0);
            state->mode = hold & 0x200 ? DICTID : TYPE;
            INITBITS();
            break;
#ifndef PKZIP_BUG_WORKAROUND
            # 如果长度或距离符号过多，则设置解压模式为 BAD
            if (state->nlen > 286 || state->ndist > 30) {
                strm->msg = (char *)"too many length or distance symbols";
                state->mode = BAD;
                break;
            }
#ifdef INFLATE_STRICT
            # 如果偏移量超出最大距离值，则设置解压模式为 BAD
            if (state->offset > state->dmax) {
                strm->msg = (char *)"invalid distance too far back";
                state->mode = BAD;
                break;
            }
#endif
            // 如果定义了 Tracevv 宏，则输出当前距离值
            Tracevv((stderr, "inflate:         distance %u\n", state->offset));
            // 设置状态为匹配
            state->mode = MATCH;
                /* fallthrough */
        case MATCH:
            // 如果剩余输出为0，则跳转到 inf_leave 标签
            if (left == 0) goto inf_leave;
            // 计算需要复制的数据长度
            copy = out - left;
            // 如果当前距离大于复制的数据长度，则从窗口中复制数据
            if (state->offset > copy) {         /* copy from window */
                copy = state->offset - copy;
                // 如果复制长度大于窗口中可用的数据长度
                if (copy > state->whave) {
                    // 如果状态为 sane，则设置错误消息并将状态设置为 BAD
                    if (state->sane) {
                        strm->msg = (char *)"invalid distance too far back";
                        state->mode = BAD;
                        break;
                    }
                    // 如果定义了 INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR 宏，则执行以下代码
#ifdef INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR
                    // 输出错误消息
                    Trace((stderr, "inflate.c too far\n"));
                    copy -= state->whave;
                    if (copy > state->length) copy = state->length;
                    if (copy > left) copy = left;
                    left -= copy;
                    state->length -= copy;
                    // 将输出缓冲区中的数据设置为0
                    do {
                        *put++ = 0;
                    } while (--copy);
                    // 如果长度为0，则将状态设置为 LEN
                    if (state->length == 0) state->mode = LEN;
                    break;
            # 如果当前模式是拷贝
            # 从输入中拷贝数据
            # 如果拷贝的长度大于窗口中剩余的数据
            if (copy > state->wnext) {
                copy -= state->wnext;
                from = state->window + (state->wsize - copy);
            }
            # 否则从窗口中拷贝数据
            else
                from = state->window + (state->wnext - copy);
            # 如果拷贝的长度大于当前块的长度，则将拷贝的长度设为当前块的长度
            if (copy > state->length) copy = state->length;
            # 如果当前模式是字面量
            # 如果剩余的空间为0，则跳转到inf_leave
            *put++ = (unsigned char)(state->length);
            left--;
            state->mode = LEN;
            break;
        # 如果当前模式是检查
        # 如果有包装
        # 需要32位的比特
        # 减去剩余的空间
        # 更新输出总数
        # 更新检查总数
        # 如果有标志位并且输出不为空
        # 更新检查值
        # 将输出值设为剩余空间
        # 如果有标志位并且
        # 如果状态标志位不为0，则保持
        # 否则交换32位的值
        # 如果检查值不等于状态检查值
        # 设置错误消息
        # 设置模式为BAD
        # 初始化比特
        # 输出检查匹配尾部
#ifdef GUNZIP
            state->mode = LENGTH;
                /* fallthrough */
        case LENGTH:
            # 如果使用了GUNZIP，则将状态设置为LENGTH
            if (state->wrap && state->flags) {
                # 如果wrap和flags都存在
                NEEDBITS(32);
                # 需要32位比特
                if ((state->wrap & 4) && hold != (state->total & 0xffffffff)) {
                    # 如果wrap的第三位存在，并且hold不等于total的低32位
                    strm->msg = (char *)"incorrect length check";
                    state->mode = BAD;
                    break;
                }
                INITBITS();
                # 初始化比特
                Tracev((stderr, "inflate:   length matches trailer\n"));
                # 输出调试信息
            }
#endif
            state->mode = DONE;
                /* fallthrough */
        case DONE:
            # 设置状态为DONE
            ret = Z_STREAM_END;
            # 返回Z_STREAM_END
            goto inf_leave;
            # 跳转到inf_leave标签
        case BAD:
            # 如果状态为BAD
            ret = Z_DATA_ERROR;
            # 返回Z_DATA_ERROR
            goto inf_leave;
            # 跳转到inf_leave标签
        case MEM:
            # 如果状态为MEM
            return Z_MEM_ERROR;
            # 返回Z_MEM_ERROR
        case SYNC:
                /* fallthrough */
        default:
            # 默认情况
            return Z_STREAM_ERROR;
            # 返回Z_STREAM_ERROR
        }

    /*
       Return from inflate(), updating the total counts and the check value.
       If there was no progress during the inflate() call, return a buffer
       error.  Call updatewindow() to create and/or update the window state.
       Note: a memory error from inflate() is non-recoverable.
     */
  inf_leave:
    # inf_leave标签
    RESTORE();
    # 恢复状态
    if (state->wsize || (out != strm->avail_out && state->mode < BAD &&
            (state->mode < CHECK || flush != Z_FINISH)))
        # 如果wsize存在，或者out不等于avail_out并且状态小于BAD并且（状态小于CHECK或者flush不等于Z_FINISH）
        if (updatewindow(strm, strm->next_out, out - strm->avail_out)) {
            # 如果调用updatewindow函数返回真
            state->mode = MEM;
            # 设置状态为MEM
            return Z_MEM_ERROR;
            # 返回Z_MEM_ERROR
        }
    in -= strm->avail_in;
    # 减去avail_in
    out -= strm->avail_out;
    # 减去avail_out
    strm->total_in += in;
    # total_in增加in
    strm->total_out += out;
    # total_out增加out
    state->total += out;
    # total增加out
    if ((state->wrap & 4) && out)
        # 如果wrap的第三位存在，并且out存在
        strm->adler = state->check =
            UPDATE_CHECK(state->check, strm->next_out - out, out);
            # 更新adler和check
    # 根据状态信息计算数据类型，包括位数、是否最后一块、模式等信息
    strm->data_type = (int)state->bits + (state->last ? 64 : 0) +
                      (state->mode == TYPE ? 128 : 0) +
                      (state->mode == LEN_ || state->mode == COPY_ ? 256 : 0);
    # 如果输入和输出都为0，或者需要刷新并且返回值为Z_OK，则将返回值设置为Z_BUF_ERROR
    if (((in == 0 && out == 0) || flush == Z_FINISH) && ret == Z_OK)
        ret = Z_BUF_ERROR;
    # 返回结果值
    return ret;
}

int ZEXPORT inflateEnd(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;
    // 检查解压缩状态，如果有问题则返回流错误
    if (inflateStateCheck(strm))
        return Z_STREAM_ERROR;
    // 获取解压缩状态
    state = (struct inflate_state FAR *)strm->state;
    // 如果窗口不为空，则释放窗口内存
    if (state->window != Z_NULL) ZFREE(strm, state->window);
    // 释放状态内存
    ZFREE(strm, strm->state);
    // 状态置为空
    strm->state = Z_NULL;
    // 输出结束信息到跟踪日志
    Tracev((stderr, "inflate: end\n"));
    // 返回解压缩成功
    return Z_OK;
}

int ZEXPORT inflateGetDictionary(strm, dictionary, dictLength)
z_streamp strm;
Bytef *dictionary;
uInt *dictLength;
{
    struct inflate_state FAR *state;

    /* check state */
    // 检查解压缩状态，如果有问题则返回流错误
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 获取解压缩状态
    state = (struct inflate_state FAR *)strm->state;

    /* copy dictionary */
    // 复制字典
    if (state->whave && dictionary != Z_NULL) {
        zmemcpy(dictionary, state->window + state->wnext,
                state->whave - state->wnext);
        zmemcpy(dictionary + state->whave - state->wnext,
                state->window, state->wnext);
    }
    // 如果字典长度不为空，则将状态中的字典长度赋值给它
    if (dictLength != Z_NULL)
        *dictLength = state->whave;
    // 返回解压缩成功
    return Z_OK;
}

int ZEXPORT inflateSetDictionary(strm, dictionary, dictLength)
z_streamp strm;
const Bytef *dictionary;
uInt dictLength;
{
    struct inflate_state FAR *state;
    unsigned long dictid;
    int ret;

    /* check state */
    // 检查解压缩状态，如果有问题则返回流错误
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 获取解压缩状态
    state = (struct inflate_state FAR *)strm->state;
    // 如果包装类型不为0且模式不为DICT，则返回流错误
    if (state->wrap != 0 && state->mode != DICT)
        return Z_STREAM_ERROR;

    /* check for correct dictionary identifier */
    // 检查正确的字典标识符
    if (state->mode == DICT) {
        dictid = adler32(0L, Z_NULL, 0);
        dictid = adler32(dictid, dictionary, dictLength);
        // 如果字典标识符不等于状态中的校验值，则返回数据错误
        if (dictid != state->check)
            return Z_DATA_ERROR;
    }

    /* copy dictionary to window using updatewindow(), which will amend the
       existing dictionary if appropriate */
    // 使用updatewindow()将字典复制到窗口中，如果适当的话会修改现有的字典
    ret = updatewindow(strm, dictionary + dictLength, dictLength);
    // 如果返回值为真，则将模式置为MEM，返回内存错误
    if (ret) {
        state->mode = MEM;
        return Z_MEM_ERROR;
    }
    // 设置已有字典标志为1
    state->havedict = 1;
    # 打印信息到标准错误流，表示字典已设置
    Tracev((stderr, "inflate:   dictionary set\n"));
    # 返回 Z_OK，表示操作成功
    return Z_OK;
}
# 定义一个函数，用于获取压缩流的头部信息
int ZEXPORT inflateGetHeader(strm, head)
z_streamp strm;
gz_headerp head;
{
    struct inflate_state FAR *state;

    /* 检查状态 */
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)strm->state;
    if ((state->wrap & 2) == 0) return Z_STREAM_ERROR;

    /* 保存头部结构 */
    state->head = head;
    head->done = 0;
    return Z_OK;
}

/*
   在 buf[0..len-1] 中搜索模式：0, 0, 0xff, 0xff。找到模式或输入结束时返回。
   调用时，*have 是迄今为止按顺序找到的模式字节数，范围在 0..3。
   返回时，*have 更新为新状态。如果返回时 *have 等于四，则找到了模式，
   返回值是读取的字节数，包括模式的最后一个字节。如果 *have 小于四，则
   尚未找到模式，返回值是 len。在后一种情况下，可以再次调用 syncsearch()
   以获取更多数据和 *have 状态。对于第一次调用，*have 初始化为零。
 */
local unsigned syncsearch(have, buf, len)
unsigned FAR *have;
const unsigned char FAR *buf;
unsigned len;
{
    unsigned got;
    unsigned next;

    got = *have;
    next = 0;
    while (next < len && got < 4) {
        if ((int)(buf[next]) == (got < 2 ? 0 : 0xff))
            got++;
        else if (buf[next])
            got = 0;
        else
            got = 4 - got;
        next++;
    }
    *have = got;
    return next;
}

# 定义一个函数，用于在压缩流中搜索特定模式
int ZEXPORT inflateSync(strm)
z_streamp strm;
{
    unsigned len;               /* 要查看或已查看的字节数 */
    int flags;                  /* 临时保存头部状态 */
    unsigned long in, out;      /* 临时保存 total_in 和 total_out */
    unsigned char buf[4];       /* 用于将位缓冲区恢复为字节字符串 */
    struct inflate_state FAR *state;

    /* 检查参数 */
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    # 将 strm->state 强制类型转换为 inflate_state 指针，并赋值给 state
    state = (struct inflate_state FAR *)strm->state;
    # 如果输入可用字节数为 0 并且状态位数小于 8，则返回缓冲区错误
    if (strm->avail_in == 0 && state->bits < 8) return Z_BUF_ERROR;

    # 如果是第一次，从位缓冲区开始搜索
    if (state->mode != SYNC):
        state->mode = SYNC
        state->hold <<= state->bits & 7
        state->bits -= state->bits & 7
        len = 0
        while (state->bits >= 8):
            buf[len++] = (unsigned char)(state->hold)
            state->hold >>= 8
            state->bits -= 8
        state->have = 0
        syncsearch(&(state->have), buf, len)

    # 搜索可用输入
    len = syncsearch(&(state->have), strm->next_in, strm->avail_in)
    strm->avail_in -= len
    strm->next_in += len
    strm->total_in += len

    # 返回错误或设置为在新块上重新启动 inflate()
    if (state->have != 4) return Z_DATA_ERROR
    if (state->flags == -1)
        state->wrap = 0    # 如果还没有头部，视为原始数据
    else
        state->wrap &= ~4  # 现在计算校验值没有意义
    flags = state->flags
    in = strm->total_in  out = strm->total_out
    inflateReset(strm)
    strm->total_in = in  strm->total_out = out
    state->flags = flags
    state->mode = TYPE
    return Z_OK
}

/*
   如果inflate当前位于由Z_SYNC_FLUSH或Z_FULL_FLUSH生成的块的末尾，则返回true。某个PPP实现使用此函数提供额外的安全检查。PPP使用Z_SYNC_FLUSH，但删除结果为空的存储块的长度字节。在解压缩时，PPP检查在输入数据包的末尾，inflate是否正在等待这些长度字节。
 */
int ZEXPORT inflateSyncPoint(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;

    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)strm->state;
    return state->mode == STORED && state->bits == 0;
}

int ZEXPORT inflateCopy(dest, source)
z_streamp dest;
z_streamp source;
{
    struct inflate_state FAR *state;
    struct inflate_state FAR *copy;
    unsigned char FAR *window;
    unsigned wsize;

    /* 检查输入 */
    if (inflateStateCheck(source) || dest == Z_NULL)
        return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)source->state;

    /* 分配空间 */
    copy = (struct inflate_state FAR *)
           ZALLOC(source, 1, sizeof(struct inflate_state));
    if (copy == Z_NULL) return Z_MEM_ERROR;
    window = Z_NULL;
    if (state->window != Z_NULL) {
        window = (unsigned char FAR *)
                 ZALLOC(source, 1U << state->wbits, sizeof(unsigned char));
        if (window == Z_NULL) {
            ZFREE(source, copy);
            return Z_MEM_ERROR;
        }
    }

    /* 复制状态 */
    zmemcpy((voidpf)dest, (voidpf)source, sizeof(z_stream));
    zmemcpy((voidpf)copy, (voidpf)state, sizeof(struct inflate_state));
    copy->strm = dest;
    if (state->lencode >= state->codes &&
        state->lencode <= state->codes + ENOUGH - 1) {
        copy->lencode = copy->codes + (state->lencode - state->codes);
        copy->distcode = copy->codes + (state->distcode - state->codes);
    }
    copy->next = copy->codes + (state->next - state->codes);
    # 如果窗口不为空
    if (window != Z_NULL) {
        # 计算窗口大小
        wsize = 1U << state->wbits;
        # 将状态对象中的窗口数据复制到当前窗口
        zmemcpy(window, state->window, wsize);
    }
    # 将当前窗口赋给拷贝对象的窗口
    copy->window = window;
    # 将拷贝对象的状态指针指向目标对象的内部状态
    dest->state = (struct internal_state FAR *)copy;
    # 返回操作成功的状态码
    return Z_OK;
# 定义一个名为 inflateUndermine 的函数，接受两个参数：z_streamp 类型的指针 strm 和整型 subvert
int ZEXPORT inflateUndermine(strm, subvert)
z_streamp strm;
int subvert;
{
    struct inflate_state FAR *state;  // 定义一个指向 inflate_state 结构体的指针变量 state

    // 检查当前的解压缩状态，如果有问题则返回 Z_STREAM_ERROR
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 将 strm->state 强制转换为 inflate_state 结构体指针类型，并赋值给 state
    state = (struct inflate_state FAR *)strm->state;
    // 如果定义了 INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR 宏
    #ifdef INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR
    // 设置 state->sane 的值为 subvert 的逻辑非
    state->sane = !subvert;
    // 返回 Z_OK
    return Z_OK;
    // 如果没有定义 INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR 宏
    #else
    // 忽略 subvert 参数
    (void)subvert;
    // 设置 state->sane 的值为 1
    state->sane = 1;
    // 返回 Z_DATA_ERROR
    return Z_DATA_ERROR;
    // 结束条件编译指令
    #endif
}

# 定义一个名为 inflateValidate 的函数，接受两个参数：z_streamp 类型的指针 strm 和整型 check
int ZEXPORT inflateValidate(strm, check)
z_streamp strm;
int check;
{
    struct inflate_state FAR *state;  // 定义一个指向 inflate_state 结构体的指针变量 state

    // 检查当前的解压缩状态，如果有问题则返回 Z_STREAM_ERROR
    if (inflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 将 strm->state 强制转换为 inflate_state 结构体指针类型，并赋值给 state
    state = (struct inflate_state FAR *)strm->state;
    // 如果 check 为真 并且 state->wrap 不为 0
    if (check && state->wrap)
        // 将 state->wrap 的值按位或上 4
        state->wrap |= 4;
    else
        // 将 state->wrap 的值按位与上取反后的 4
        state->wrap &= ~4;
    // 返回 Z_OK
    return Z_OK;
}

# 定义一个名为 inflateMark 的函数，接受一个参数：z_streamp 类型的指针 strm
long ZEXPORT inflateMark(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;  // 定义一个指向 inflate_state 结构体的指针变量 state

    // 检查当前的解压缩状态，如果有问题则返回 -(1L << 16)
    if (inflateStateCheck(strm))
        return -(1L << 16);
    // 将 strm->state 强制转换为 inflate_state 结构体指针类型，并赋值给 state
    state = (struct inflate_state FAR *)strm->state;
    // 返回一个长整型值，根据 state->mode 的不同计算不同的结果
    return (long)(((unsigned long)((long)state->back)) << 16) +
        (state->mode == COPY ? state->length :
            (state->mode == MATCH ? state->was - state->length : 0));
}

# 定义一个名为 inflateCodesUsed 的函数，接受一个参数：z_streamp 类型的指针 strm
unsigned long ZEXPORT inflateCodesUsed(strm)
z_streamp strm;
{
    struct inflate_state FAR *state;  // 定义一个指向 inflate_state 结构体的指针变量 state
    // 检查当前的解压缩状态，如果有问题则返回一个特定值
    if (inflateStateCheck(strm)) return (unsigned long)-1;
    // 将 strm->state 强制转换为 inflate_state 结构体指针类型，并赋值给 state
    state = (struct inflate_state FAR *)strm->state;
    // 返回一个无符号长整型值，表示 state->next 和 state->codes 之间的差值
    return (unsigned long)(state->next - state->codes);
}
```