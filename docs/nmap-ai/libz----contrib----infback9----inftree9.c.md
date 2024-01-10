# `nmap\libz\contrib\infback9\inftree9.c`

```
/* inftree9.c -- 生成用于高效解码的哈夫曼树
 * 版权所有 (C) 1995-2022 Mark Adler
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

#include "zutil.h"
#include "inftree9.h"

#define MAXBITS 15

const char inflate9_copyright[] =
   " inflate9 1.2.13 版权所有 1995-2022 Mark Adler ";
/*
  如果您在产品中使用 zlib 库，欢迎在产品文档中提及。如果由于某种原因您无法包含此类确认，我将感激您在产品的可执行文件中保留此版权字符串。
 */

/*
   构建一组表来解码提供的规范哈夫曼编码。
   长度为 lens[0..codes-1]。结果从 *table 开始，其索引为 0..2^bits-1。
   work 是一个至少包含 lens 个 short 类型的可写数组，用作工作区。
   type 是要生成的代码类型，CODES、LENS 或 DISTS。
   返回值为零表示成功，-1 表示无效代码，+1 表示 ENOUGH 不够用。
   返回时，table 指向下一个可用条目的地址。
   bits 是请求的根表索引位数，返回时它是实际的根表索引位数。
   如果请求大于最长代码或小于最短代码，它将不同。
 */
int inflate_table9(type, lens, codes, table, bits, work)
codetype type;
unsigned short FAR *lens;
unsigned codes;
code FAR * FAR *table;
unsigned FAR *bits;
unsigned short FAR *work;
{
    unsigned len;               /* 代码的位长度 */
    unsigned sym;               /* 代码符号的索引 */
    unsigned min, max;          /* 最小和最大代码长度 */
    unsigned root;              /* 根表的索引位数 */
    unsigned curr;              /* 当前表的索引位数 */
    unsigned drop;              /* 子表的代码位数 */
    int left;                   /* 可用前缀代码的数量 */
    # 用于记录表中使用的编码条目数
    unsigned used;              /* code entries in table used */
    # 哈夫曼编码
    unsigned huff;              /* Huffman code */
    # 用于增加编码和索引
    unsigned incr;              /* for incrementing code, index */
    # 用于复制条目的索引
    unsigned fill;              /* index for replicating entries */
    # 当前根条目的低位
    unsigned low;               /* low bits for current root entry */
    # 低根位的掩码
    unsigned mask;              /* mask for low root bits */
    # 用于复制的表条目
    code this;                  /* table entry for duplication */
    # 表中下一个可用空间
    code FAR *next;             /* next available space in table */
    # 要使用的基本值表
    const unsigned short FAR *base;     /* base value table to use */
    # 要使用的额外位表
    const unsigned short FAR *extra;    /* extra bits table to use */
    # 使用基本和额外位表示符> end
    int end;                    /* use base and extra for symbol > end */
    # 每个长度的编码数
    unsigned short count[MAXBITS+1];    /* number of codes of each length */
    # 每个长度的表中的偏移量
    unsigned short offs[MAXBITS+1];     /* offsets in table for each length */
    # 长度代码257..285的基数
    static const unsigned short lbase[31] = { /* Length codes 257..285 base */
        3, 4, 5, 6, 7, 8, 9, 10, 11, 13, 15, 17,
        19, 23, 27, 31, 35, 43, 51, 59, 67, 83, 99, 115,
        131, 163, 195, 227, 3, 0, 0};
    # 长度代码257..285的额外位
    static const unsigned short lext[31] = { /* Length codes 257..285 extra */
        128, 128, 128, 128, 128, 128, 128, 128, 129, 129, 129, 129,
        130, 130, 130, 130, 131, 131, 131, 131, 132, 132, 132, 132,
        133, 133, 133, 133, 144, 194, 65};
    # 距离代码0..31的基数
    static const unsigned short dbase[32] = { /* Distance codes 0..31 base */
        1, 2, 3, 4, 5, 7, 9, 13, 17, 25, 33, 49,
        65, 97, 129, 193, 257, 385, 513, 769, 1025, 1537, 2049, 3073,
        4097, 6145, 8193, 12289, 16385, 24577, 32769, 49153};
    # 距离代码0..31的额外位
    static const unsigned short dext[32] = { /* Distance codes 0..31 extra */
        128, 128, 128, 128, 129, 129, 130, 130, 131, 131, 132, 132,
        133, 133, 134, 134, 135, 135, 136, 136, 137, 137, 138, 138,
        139, 139, 140, 140, 141, 141, 142, 142};
    /*
       处理一组码长，创建一个规范的哈夫曼编码。码长存储在lens[0..codes-1]中。每个码长对应于符号0..codes-1。哈夫曼编码是通过首先按照长度从短到长对符号进行排序，并保留具有相等长度的符号顺序来生成的。然后，对于最短长度的第一个代码，代码从所有零位开始，对于相同长度的代码，代码是整数递增，随着长度的增加，会附加零。对于deflate格式，这些位是以它们更自然的整数递增顺序的相反顺序存储的，因此当在下面的大循环中构建解码表时，整数代码是向后递增的。

       此例程假定，但不检查，lens[]中的所有条目都在0..MAXBITS范围内。调用者必须确保这一点。1..MAXBITS被解释为该码长。零表示该符号在此代码中不存在。

       通过计算每个长度的代码数量来对代码进行排序，从而创建一个在排序表中每个长度的起始索引的表，并按顺序在排序表中输入符号。排序表是work[]，该空间由调用者提供。

       长度计数也用于其他目的，例如查找最小和最大长度代码，确定是否有任何代码，检查有效的长度集，并在构建解码表时预先查看长度计数以确定子表大小。
     */

    /* 累积代码长度（假设lens[]都在0..MAXBITS范围内） */
    for (len = 0; len <= MAXBITS; len++)
        count[len] = 0;
    for (sym = 0; sym < codes; sym++)
        count[lens[sym]]++;

    /* 限制代码长度，强制根节点在代码长度范围内 */
    # 将 root 设置为 bits 指针所指向的值
    root = *bits;
    # 从最大位数开始向最小位数遍历，找到第一个不为0的位数
    for (max = MAXBITS; max >= 1; max--)
        if (count[max] != 0) break;
    # 如果 root 大于 max，则将 root 设置为 max
    if (root > max) root = max;
    # 如果 max 等于 0，则返回 -1，表示没有编码
    if (max == 0) return -1;            /* no codes! */
    # 从最小位数开始向最大位数遍历，找到第一个不为0的位数
    for (min = 1; min <= MAXBITS; min++)
        if (count[min] != 0) break;
    # 如果 root 小于 min，则将 root 设置为 min
    if (root < min) root = min;

    # 检查编码长度是否超出或不完整
    left = 1;
    # 遍历编码长度
    for (len = 1; len <= MAXBITS; len++) {
        left <<= 1;
        left -= count[len];
        # 如果 left 小于 0，则返回 -1，表示编码超出
        if (left < 0) return -1;        /* over-subscribed */
    }
    # 如果 left 大于 0 且（type == CODES 或 max 不等于 1），则返回 -1，表示编码不完整
    if (left > 0 && (type == CODES || max != 1))
        return -1;                      /* incomplete set */

    # 为每个长度生成符号表的偏移量，用于排序
    offs[1] = 0;
    # 遍历编码长度，计算偏移量
    for (len = 1; len < MAXBITS; len++)
        offs[len + 1] = offs[len] + count[len];

    # 按长度和符号顺序对符号进行排序
    for (sym = 0; sym < codes; sym++)
        if (lens[sym] != 0) work[offs[lens[sym]]++] = (unsigned short)sym;
    /*
       创建并填充解码表。在这个循环中，被填充的表在 next 处，具有 curr 索引位。正在使用的代码是长度为 len 的 huff。该代码通过从底部删除 drop 位转换为索引。对于长度小于 drop + curr 的代码，顶部的 drop + curr - len 位通过所有值递增以填充具有复制条目的表。

       root 是根表的索引位数。当 len 超过 root 时，将创建由根条目指向的子表，其索引为 huff 的低 root 位。这保存在 low 中，以检查何时应开始新的子表。当填充根表时，drop 为零，当填充子表时，drop 为 root。

       当需要新的子表时，需要向前查看代码长度以确定需要多大的子表。长度计数用于此目的，因此 count[] 在输入表中的代码时递减。

       used 跟踪从提供的 *table 空间分配了多少表条目。对于 LENS 和 DIST 表，它与常量 ENOUGH_LENS 和 ENOUGH_DISTS 进行比较，以防止初始根表大小常量的更改。有关更多信息，请参见 inftree9.h 中的注释。

       sym 递增通过所有符号，并且循环在处理所有长度为 max 的代码，即所有代码时终止。此例程允许不完整的代码，因此在此之后的另一个循环将使用无效代码标记填充其余的解码表。
     */

    /* 为代码类型设置准备 */
    switch (type) {
    case CODES:
        base = extra = work;    /* 虚拟值--未使用 */
        end = 19;
        break;
    # 根据不同的情况设置基础值和额外值
    case LENS:
        base = lbase;           # 设置基础值为 lbase
        base -= 257;            # 基础值减去 257
        extra = lext;           # 设置额外值为 lext
        extra -= 257;           # 额外值减去 257
        end = 256;              # 设置结束值为 256
        break;                  # 结束 switch 语句
    default:                    # 默认情况为 DISTS
        base = dbase;           # 设置基础值为 dbase
        extra = dext;           # 设置额外值为 dext
        end = -1;               # 设置结束值为 -1
    }

    # 初始化循环的状态
    huff = 0;                   # 起始编码
    sym = 0;                    # 起始编码符号
    len = min;                  # 起始编码长度
    next = *table;              # 当前要填充的表
    curr = root;                # 当前表的索引位数
    drop = 0;                   # 从编码中丢弃的当前位数
    low = (unsigned)(-1);       # 当 len > root 时触发新的子表
    used = 1U << root;          # 使用 root 表的条目
    mask = used - 1;            # 用于比较 low 的掩码

    # 检查可用的表空间
    if ((type == LENS && used >= ENOUGH_LENS) || (type == DISTS && used >= ENOUGH_DISTS))
        return 1;

    # 处理所有编码并创建表条目

    /*
       为不完整的编码填充表的其余部分。这个循环类似于上面的循环，用于增加表索引的 huff。
       假设 len 等于 curr + drop，因此不需要循环递增高索引位。当当前子表填满时，循环将返回到根表，
       填充那里的任何剩余条目。
     */
    this.op = (unsigned char)64;                # 无效的编码标记
    this.bits = (unsigned char)(len - drop);    # 设置位数
    this.val = (unsigned short)0;               # 设置值
    while (huff != 0) {
        /* 当哈夫曼编码不为0时执行循环 */

        /* 当处理完子表后，返回到根表 */
        if (drop != 0 && (huff & mask) != low) {
            drop = 0;
            len = root;
            next = *table;
            curr = root;
            this.bits = (unsigned char)len;
        }

        /* 在表中放入无效的编码标记 */
        next[huff >> drop] = this;

        /* 反向增加长度为len的哈夫曼编码 */
        incr = 1U << (len - 1);
        while (huff & incr)
            incr >>= 1;
        if (incr != 0) {
            huff &= incr - 1;
            huff += incr;
        }
        else
            huff = 0;
    }

    /* 设置返回参数 */
    *table += used;
    *bits = root;
    return 0;
# 闭合前面的函数定义
```