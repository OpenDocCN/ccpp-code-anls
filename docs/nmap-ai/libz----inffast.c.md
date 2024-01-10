# `nmap\libz\inffast.c`

```
/* inffast.c -- 快速解码
 * 版权所有 (C) 1995-2017 Mark Adler
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

#include "zutil.h"
#include "inftrees.h"
#include "inflate.h"
#include "inffast.h"

#ifdef ASMINF
#  pragma message("Assembler code may have bugs -- use at your own risk")
#else

/*
   解码字面量、长度和距离编码，并写入结果字面量和匹配字节，直到输入或输出不足、遇到块结束，或遇到数据错误。
   当提供足够大的输入和输出缓冲区给 inflate()，例如，一个 16K 的输入缓冲区和一个 64K 的输出缓冲区时，超过 95% 的 inflate 执行时间都花费在这个例程中。

   进入假设：

        state->mode == LEN
        strm->avail_in >= 6
        strm->avail_out >= 258
        start >= strm->avail_out
        state->bits < 8

   返回时，state->mode 是以下之一：

        LEN -- 输出空间不足或可用输入不足
        TYPE -- 到达块结束码，inflate() 解释下一个块
        BAD -- 块数据错误

   注意：

    - 长度/距离对使用的最大输入位数是 15 位用于长度码，5 位用于长度额外，15 位用于距离码，13 位用于距离额外。总共 48 位，或者六个字节。
      因此，如果 strm->avail_in >= 6，则有足够的输入来避免在解码时检查可用输入。

    - 单个长度/距离对可以输出的最大字节数是 258 字节，这是可以编码的最大长度。inflate_fast() 要求每个循环 strm->avail_out >= 258，以避免检查输出空间。
 */
void ZLIB_INTERNAL inflate_fast(strm, start)
z_streamp strm;
unsigned start;         /* inflate() 的起始值，用于 strm->avail_out */
{
    struct inflate_state FAR *state;
    # 定义指向输入数据的指针，用于压缩流的下一个输入
    z_const unsigned char FAR *in;      /* local strm->next_in */
    # 定义指向输入数据的指针，用于在输入数据足够时停止
    z_const unsigned char FAR *last;    /* have enough input while in < last */
    # 定义指向输出数据的指针，用于压缩流的下一个输出
    unsigned char FAR *out;     /* local strm->next_out */
    # 定义指向输出数据的指针，用于inflate()函数的初始输出
    unsigned char FAR *beg;     /* inflate()'s initial strm->next_out */
    # 定义指向输出数据的指针，用于在输出空间足够时停止
    unsigned char FAR *end;     /* while out < end, enough space available */
#ifdef INFLATE_STRICT
    # 最大距离，从 zlib 头开始计算
    unsigned dmax;              
#endif
    # 窗口大小，如果不使用窗口则为零
    unsigned wsize;             
    # 窗口中有效字节数
    unsigned whave;             
    # 窗口写入索引
    unsigned wnext;             
    # 分配的滑动窗口，如果 wsize != 0
    unsigned char FAR *window;  
    # 本地 strm->hold
    unsigned long hold;         
    # 本地 strm->bits
    unsigned bits;              
    # 本地 strm->lencode
    code const FAR *lcode;      
    # 本地 strm->distcode
    code const FAR *dcode;      
    # 长度编码的第一级掩码
    unsigned lmask;             
    # 距离编码的第一级掩码
    unsigned dmask;             
    # 检索到的表项
    code const *here;           
    # 代码位、操作、额外位或窗口位置、要复制的窗口字节数
    unsigned op;                
    # 匹配长度，未使用的字节数
    unsigned len;               
    # 匹配距离
    unsigned dist;              
    # 从哪里复制匹配
    unsigned char FAR *from;    

    # 将状态复制到本地变量
    state = (struct inflate_state FAR *)strm->state;
    in = strm->next_in;
    last = in + (strm->avail_in - 5);
    out = strm->next_out;
    beg = out - (start - strm->avail_out);
    end = out + (strm->avail_out - 257);
#ifdef INFLATE_STRICT
    dmax = state->dmax;
#endif
    wsize = state->wsize;
    whave = state->whave;
    wnext = state->wnext;
    window = state->window;
    hold = state->hold;
    bits = state->bits;
    lcode = state->lencode;
    dcode = state->distcode;
    lmask = (1U << state->lenbits) - 1;
    dmask = (1U << state->distbits) - 1;

    # 解码文字和长度/距离，直到块结束或输入数据不足或输出空间不足
#ifdef INFLATE_STRICT
                # 如果距离太远，则设置错误消息并将状态设置为BAD
                if (dist > dmax) {
                    strm->msg = (char *)"invalid distance too far back";
                    state->mode = BAD;
                    break;
                }
#endif
                # 将hold右移op位，更新bits
                hold >>= op;
                bits -= op;
                # 打印距离信息
                Tracevv((stderr, "inflate:         distance %u\n", dist));
                # 计算最大输出距离
                op = (unsigned)(out - beg);     /* max distance in output */
                # 如果距离大于最大输出距离
                if (dist > op) {                /* see if copy from window */
                    # 计算从窗口中复制的距离
                    op = dist - op;             /* distance back in window */
                    # 如果复制的距离大于whave
                    if (op > whave) {
                        # 如果状态为sane，则设置错误消息并将状态设置为BAD
                        if (state->sane) {
                            strm->msg =
                                (char *)"invalid distance too far back";
                            state->mode = BAD;
                            break;
                        }
#ifdef INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR
                        # 如果长度小于等于op-whave
                        if (len <= op - whave) {
                            # 将out后面的len个字节设置为0
                            do {
                                *out++ = 0;
                            } while (--len);
                            continue;
                        }
                        # 更新长度
                        len -= op - whave;
                        # 将out后面的op-whave个字节设置为0
                        do {
                            *out++ = 0;
                        } while (--op > whave);
                        # 如果op为0
                        if (op == 0) {
                            # 计算from的位置
                            from = out - dist;
                            # 将from后面的len个字节复制到out后面
                            do {
                                *out++ = *from++;
                            } while (--len);
                            continue;
                        }
    } while (in < last && out < end);

    # 返回未使用的字节（在进入时，bits < 8，因此in不会太远）
    len = bits >> 3;
    in -= len;
    bits -= len << 3;
    hold &= (1U << bits) - 1;

    # 更新状态并返回
    strm->next_in = in;
    strm->next_out = out;
    strm->avail_in = (unsigned)(in < last ? 5 + (last - in) : 5 - (in - last));
    # 设置输出缓冲区的可用空间大小
    strm->avail_out = (unsigned)(out < end ?
                                 257 + (end - out) : 257 - (out - end));
    # 保存当前的状态信息
    state->hold = hold;
    state->bits = bits;
    # 返回
    return;
# 以下是一些关于 inflate_fast() 函数的注释，说明了一些尝试加速该函数的方法，以及它们在某些情况下反而变慢的原因
# 在 PowerPC G3 750CXe 上，以下方法实际上变慢了：
# - 使用位字段来定义代码结构
# - 使用不同的操作定义来避免额外位的 & 运算（对于表位进行 & 运算）
# - 为直接、窗口和 wnext == 0 的情况分别使用三个解码 do-循环
# - 对于距离大于 1 的拷贝，使用特殊情况来进行重叠加载和存储拷贝
# - 显式分支预测（基于测量的分支概率）
# - 推迟匹配拷贝，并将其与解码后续代码交错进行
# - 交换文字/长度 else
# - 交换窗口/直接 else
# - 更大规模的展开拷贝循环（三个大约是合适的）
# - 将 len -= 3 语句移到循环中间
```