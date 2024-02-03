# `nmap\libz\contrib\infback9\infback9.c`

```cpp
/* infback9.c -- 使用回调接口解压缩 deflate64 数据
 * 版权所有 (C) 1995-2008 Mark Adler
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

#include "zutil.h"  // 包含 zlib 实用函数
#include "infback9.h"  // 包含 inflateBack9 相关声明
#include "inftree9.h"  // 包含 inftree9 相关声明
#include "inflate9.h"  // 包含 inflate9 相关声明

#define WSIZE 65536UL  // 定义窗口大小为 64K 字节

/*
   strm 提供了内存分配函数 zalloc 和 zfree，或者
   使用库内存分配函数的话传入 Z_NULL。

   window 是用户提供的窗口和输出缓冲区，大小为 64K 字节。
 */
int ZEXPORT inflateBack9Init_(strm, window, version, stream_size)
z_stream FAR *strm;  // 指向 z_stream 结构的指针
unsigned char FAR *window;  // 指向窗口和输出缓冲区的指针
const char *version;  // 版本信息
int stream_size;  // 流的大小
{
    struct inflate_state FAR *state;  // 指向 inflate_state 结构的指针

    if (version == Z_NULL || version[0] != ZLIB_VERSION[0] ||
        stream_size != (int)(sizeof(z_stream)))
        return Z_VERSION_ERROR;  // 如果版本信息为空或不匹配，或者流的大小不正确，则返回版本错误
    if (strm == Z_NULL || window == Z_NULL)
        return Z_STREAM_ERROR;  // 如果流或窗口为空，则返回流错误
    strm->msg = Z_NULL;  /* 以防我们返回错误 */
    if (strm->zalloc == (alloc_func)0) {
        strm->zalloc = zcalloc;  // 如果分配函数为空，则使用 zcalloc
        strm->opaque = (voidpf)0;
    }
    if (strm->zfree == (free_func)0) strm->zfree = zcfree;  // 如果释放函数为空，则使用 zcfree
    state = (struct inflate_state FAR *)ZALLOC(strm, 1,
                                               sizeof(struct inflate_state));  // 分配内存给 inflate_state 结构
    if (state == Z_NULL) return Z_MEM_ERROR;  // 如果分配失败，则返回内存错误
    Tracev((stderr, "inflate: allocated\n"));  // 跟踪信息
    strm->state = (voidpf)state;  // 设置流的状态
    state->window = window;  // 设置状态的窗口
    return Z_OK;  // 返回成功
}

/*
   为固定代码解码构建和输出长度和距离解码表。
 */
#ifdef MAKEFIXED
#include <stdio.h>

void makefixed9(void)
{
    unsigned sym, bits, low, size;
    code *next, *lenfix, *distfix;
    struct inflate_state state;
    code fixed[544];

    /* 字面/长度表 */
    sym = 0;
    while (sym < 144) state.lens[sym++] = 8;
    while (sym < 256) state.lens[sym++] = 9;
    while (sym < 280) state.lens[sym++] = 7;
    while (sym < 288) state.lens[sym++] = 8;
    next = fixed;
    lenfix = next;
    # 设置变量 bits 的值为 9
    bits = 9;
    # 调用函数 inflate_table9()，传入参数 LENS、state.lens、288、&(next)、&(bits)、state.work
    inflate_table9(LENS, state.lens, 288, &(next), &(bits), state.work);

    # 设置变量 sym 的值为 0
    sym = 0;
    # 循环，当 sym 小于 32 时，执行下面的语句
    while (sym < 32) state.lens[sym++] = 5;
    # 设置变量 distfix 的值为 next
    distfix = next;
    # 设置变量 bits 的值为 5
    bits = 5;
    # 调用函数 inflate_table9()，传入参数 DISTS、state.lens、32、&(next)、&(bits)、state.work

    inflate_table9(DISTS, state.lens, 32, &(next), &(bits), state.work);

    # 输出注释
    puts("    /* inffix9.h -- table for decoding deflate64 fixed codes");
    puts("     * Generated automatically by makefixed9().");
    puts("     */");
    puts("");
    puts("    /* WARNING: this file should *not* be used by applications.");
    puts("       It is part of the implementation of this library and is");
    puts("       subject to change. Applications should only use zlib.h.");
    puts("     */");
    puts("");
    # 设置变量 size 的值为 1 左移 9 位
    size = 1U << 9;
    # 输出格式化字符串，将 size 的值插入到字符串中
    printf("    static const code lenfix[%u] = {", size);
    # 设置变量 low 的值为 0
    low = 0;
    # 无限循环
    for (;;) {
        # 如果 low 除以 6 的余数为 0，则输出换行和缩进
        if ((low % 6) == 0) printf("\n        ");
        # 输出格式化字符串，将 lenfix[low] 的成员值插入到字符串中
        printf("{%u,%u,%d}", lenfix[low].op, lenfix[low].bits,
               lenfix[low].val);
        # 如果 low 的值加一等于 size，则跳出循环
        if (++low == size) break;
        # 输出逗号
        putchar(',');
    }
    # 输出换行和右大括号
    puts("\n    };");
    # 设置变量 size 的值为 1 左移 5 位
    size = 1U << 5;
    # 输出格式化字符串，将 size 的值插入到字符串中
    printf("\n    static const code distfix[%u] = {", size);
    # 设置变量 low 的值为 0
    low = 0;
    # 无限循环
    for (;;) {
        # 如果 low 除以 5 的余数为 0，则输出换行和缩进
        if ((low % 5) == 0) printf("\n        ");
        # 输出格式化字符串，将 distfix[low] 的成员值插入到字符串中
        printf("{%u,%u,%d}", distfix[low].op, distfix[low].bits,
               distfix[low].val);
        # 如果 low 的值加一等于 size，则跳出循环
        if (++low == size) break;
        # 输出逗号
        putchar(',');
    }
    # 输出换行和右大括号
    puts("\n    };");
#endif /* MAKEFIXED */

/* 定义宏：清空输入位累加器 */
#define INITBITS() \
    do { \
        hold = 0; \  // 初始化 hold 变量为 0
        bits = 0; \   // 初始化 bits 变量为 0
    } while (0)

/* 定义宏：确保有可用的输入。如果请求了输入但被拒绝，则从 inflateBack() 返回 Z_BUF_ERROR。 */
#define PULL() \
    do { \
        if (have == 0) { \  // 如果没有可用的输入
            have = in(in_desc, &next); \  // 从输入描述符中获取输入
            if (have == 0) { \  // 如果没有获取到输入
                next = Z_NULL; \  // 将 next 设置为 NULL
                ret = Z_BUF_ERROR; \  // 将 ret 设置为 Z_BUF_ERROR
                goto inf_leave; \  // 跳转到 inf_leave 标签处
            } \
        } \
    } while (0)

/* 定义宏：将一个字节的输入放入位累加器中，如果没有可用的输入，则从 inflateBack() 返回错误。 */
#define PULLBYTE() \
    do { \
        PULL(); \  // 调用 PULL() 宏
        have--; \  // have 减一
        hold += (unsigned long)(*next++) << bits; \  // 将输入的字节放入位累加器中
        bits += 8; \  // bits 加 8
    } while (0)

/* 定义宏：确保位累加器中至少有 n 位。如果没有足够的可用输入，则从 inflateBack() 返回错误。 */
#define NEEDBITS(n) \
    do { \
        while (bits < (unsigned)(n)) \  // 当位累加器中的位数小于 n 时
            PULLBYTE(); \  // 调用 PULLBYTE() 宏
    } while (0)

/* 定义宏：返回位累加器的低 n 位（n <= 16） */
#define BITS(n) \
    ((unsigned)hold & ((1U << (n)) - 1))  // 返回位累加器的低 n 位

/* 定义宏：从位累加器中移除 n 位 */
#define DROPBITS(n) \
    do { \
        hold >>= (n); \  // 将位累加器右移 n 位
        bits -= (unsigned)(n); \  // bits 减去 n
    } while (0)

/* 定义宏：根据需要移除零到七位，以达到字节边界 */
#define BYTEBITS() \
    do { \
        hold >>= bits & 7; \  // 将位累加器右移 bits & 7 位
        bits -= bits & 7; \  // bits 减去 bits & 7
    } while (0)

/* 定义宏：确保有可用的输出空间，如果窗口已满，则写出窗口。如果写入失败，则从 inflateBack() 返回 Z_BUF_ERROR。 */
#define ROOM() \
    # 使用 do-while 循环，确保至少执行一次
    do { \
        # 如果剩余未处理的数据长度为0
        if (left == 0) { \
            # 将窗口数据赋值给 put
            put = window; \
            # 重置剩余未处理数据长度为窗口大小
            left = WSIZE; \
            # 设置标志位为1，表示需要重新包装数据
            wrap = 1; \
            # 如果输出函数返回非0，表示输出错误，返回 Z_BUF_ERROR
            if (out(out_desc, put, (unsigned)left)) { \
                ret = Z_BUF_ERROR; \
                # 跳转到 inf_leave 标签处，结束函数
                goto inf_leave; \
            } \
        } \
    } while (0)
/*
   strm提供内存分配函数和输入的窗口缓冲区，并在返回时提供未使用的输入信息。对于Z_DATA_ERROR返回，strm还将提供错误消息。

   in()和out()是回调输入和输出函数。当inflateBack()需要更多输入时，它调用in()。当inflateBack()用输出填充窗口，或者当它完成窗口中的数据时，它调用out()来写出数据。应用程序在in()再次被调用或inflateBack()返回之前不得更改提供的输入。应用程序在inflateBack()返回之前不得更改窗口/输出缓冲区。

   in()和out()在inflateBack()调用中使用提供的描述符参数。此参数可以是一个结构，提供执行读取或写入所需的信息，以及输入和输出的累积信息，如总数和校验值。

   in()在失败时应返回零。out()在失败时应返回非零。如果in()或out()中的任何一个失败，则inflateBack()返回Z_BUF_ERROR。可以检查strm->next_in是否为Z_NULL，以查看是in()还是out()导致了错误。否则，inflateBack()在成功时返回Z_STREAM_END，在deflate格式错误时返回Z_DATA_ERROR，如果无法为状态分配内存，则返回Z_MEM_ERROR。如果输入参数不正确，即strm为Z_NULL或状态未初始化，则inflateBack()还可以返回Z_STREAM_ERROR。
 */
int ZEXPORT inflateBack9(strm, in, in_desc, out, out_desc)
z_stream FAR *strm;
in_func in;
void FAR *in_desc;
out_func out;
void FAR *out_desc;
{
    struct inflate_state FAR *state;
    z_const unsigned char FAR *next;    /* 下一个输入 */
    unsigned char FAR *put;     /* 下一个输出 */
    unsigned have;              /* 可用输入 */
    unsigned long left;         /* 可用输出 */
    // 当前解压缩模式
    inflate_mode mode;
    // 是否处理最后一个数据块
    int lastblock;
    // 窗口是否已经回绕
    int wrap;
    // 分配的滑动窗口，如果需要的话
    unsigned char FAR *window;
    // 位缓冲区
    unsigned long hold;
    // 位缓冲区中的位数
    unsigned bits;
    // 需要的额外位数
    unsigned extra;
    // 要复制的数据的长度
    unsigned long length;
    // 从哪里复制匹配的字节
    unsigned long offset;
    // 要复制的存储或匹配字节数
    unsigned long copy;
    // 从哪里复制匹配字节
    unsigned char FAR *from;
    // 长度/字面量代码的起始表
    code const FAR *lencode;
    // 距离代码的起始表
    code const FAR *distcode;
    // lencode的索引位
    unsigned lenbits;
    // distcode的索引位
    unsigned distbits;
    // 当前解码表条目
    code here;
    // 父表条目
    code last;
    // 重复的长度，要丢弃的位数
    unsigned len;
    // 返回代码
    int ret;
    // 长度码长度的排列
    static const unsigned short order[19] = {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};
#include "inffix9.h"

    /* 检查 strm 是否存在以及状态是否已初始化 */
    if (strm == Z_NULL || strm->state == Z_NULL)
        return Z_STREAM_ERROR;
    state = (struct inflate_state FAR *)strm->state;

    /* 重置状态 */
    strm->msg = Z_NULL;
    mode = TYPE;
    lastblock = 0;
    wrap = 0;
    window = state->window;
    next = strm->next_in;
    have = next != Z_NULL ? strm->avail_in : 0;
    hold = 0;
    bits = 0;
    put = window;
    left = WSIZE;
    lencode = Z_NULL;
    distcode = Z_NULL;

    /* 解压缩直到标记为最后的块 */
    /* 返回未使用的输入 */
  inf_leave:
    strm->next_in = next;
    strm->avail_in = have;
    return ret;
}

int ZEXPORT inflateBack9End(strm)
z_stream FAR *strm;
{
    if (strm == Z_NULL || strm->state == Z_NULL || strm->zfree == (free_func)0)
        return Z_STREAM_ERROR;
    ZFREE(strm, strm->state);
    strm->state = Z_NULL;
    Tracev((stderr, "inflate: end\n"));
    return Z_OK;
}
```