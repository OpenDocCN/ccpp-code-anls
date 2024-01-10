# `nmap\libz\inftrees.c`

```
# 生成用于高效解码的哈夫曼树
# 版权声明
/* inftrees.c -- generate Huffman trees for efficient decoding
 * Copyright (C) 1995-2022 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

# 包含必要的头文件
#include "zutil.h"
#include "inftrees.h"

# 定义最大位数
#define MAXBITS 15

# 定义版权声明
const char inflate_copyright[] =
   " inflate 1.2.13 Copyright 1995-2022 Mark Adler ";
/*
  If you use the zlib library in a product, an acknowledgment is welcome
  in the documentation of your product. If for some reason you cannot
  include such an acknowledgment, I would appreciate that you keep this
  copyright string in the executable of your product.
 */

# 构建一组表来解码提供的规范哈夫曼编码
# 长度为lens[0..codes-1]。结果从*table开始，其索引为0..2^bits-1。
# work是一个至少包含lens个short的可写数组，用作工作区。
# type是要生成的代码类型，CODES、LENS或DISTS。
# 返回值为0表示成功，-1表示无效代码，+1表示ENOUGH不够。
# 返回时，table指向下一个可用条目的地址。bits是请求的根表索引位数，返回时它是实际的根表索引位数。
# 如果请求大于最长代码或小于最短代码，则它将不同。
int ZLIB_INTERNAL inflate_table(type, lens, codes, table, bits, work)
codetype type;
unsigned short FAR *lens;
unsigned codes;
code FAR * FAR *table;
unsigned FAR *bits;
unsigned short FAR *work;
{
    unsigned len;               /* a code's length in bits */  # 代码的位长度
    unsigned sym;               /* index of code symbols */  # 代码符号的索引
    unsigned min, max;          /* minimum and maximum code lengths */  # 最小和最大代码长度
    unsigned root;              /* number of index bits for root table */  # 根表的索引位数
    unsigned curr;              /* number of index bits for current table */  # 当前表的索引位数
    unsigned drop;              /* code bits to drop for sub-table */  # 子表的代码位数
    int left;                   /* 剩余可用的前缀码数量 */
    unsigned used;              /* 表中已使用的码 */
    unsigned huff;              /* 哈夫曼码 */
    unsigned incr;              /* 用于增加码的索引 */
    unsigned fill;              /* 复制条目的索引 */
    unsigned low;               /* 当前根条目的低位 */
    unsigned mask;              /* 低位根码的掩码 */
    code here;                  /* 复制的表条目 */
    code FAR *next;             /* 表中下一个可用空间 */
    const unsigned short FAR *base;     /* 要使用的基本值表 */
    const unsigned short FAR *extra;    /* 要使用的额外位表 */
    unsigned match;             /* 对于符号 >= match 使用基本值和额外位 */
    unsigned short count[MAXBITS+1];    /* 每个长度的码的数量 */
    unsigned short offs[MAXBITS+1];     /* 每个长度在表中的偏移量 */
    static const unsigned short lbase[31] = { /* 长度码 257..285 的基数 */
        3, 4, 5, 6, 7, 8, 9, 10, 11, 13, 15, 17, 19, 23, 27, 31,
        35, 43, 51, 59, 67, 83, 99, 115, 131, 163, 195, 227, 258, 0, 0};
    static const unsigned short lext[31] = { /* 长度码 257..285 的额外位 */
        16, 16, 16, 16, 16, 16, 16, 16, 17, 17, 17, 17, 18, 18, 18, 18,
        19, 19, 19, 19, 20, 20, 20, 20, 21, 21, 21, 21, 16, 194, 65};
    static const unsigned short dbase[32] = { /* 距离码 0..29 的基数 */
        1, 2, 3, 4, 5, 7, 9, 13, 17, 25, 33, 49, 65, 97, 129, 193,
        257, 385, 513, 769, 1025, 1537, 2049, 3073, 4097, 6145,
        8193, 12289, 16385, 24577, 0, 0};
    static const unsigned short dext[32] = { /* 距离码 0..29 的额外位 */
        16, 16, 16, 16, 17, 17, 18, 18, 19, 19, 20, 20, 21, 21, 22, 22,
        23, 23, 24, 24, 25, 25, 26, 26, 27, 27,
        28, 28, 29, 29, 64, 64};
    /*
       处理一组码长，创建一个规范的哈夫曼编码。码长存储在lens[0..codes-1]中。每个码长对应于符号0..codes-1。哈夫曼编码的生成方式是首先按照码长从短到长对符号进行排序，并保留具有相等码长的符号顺序。然后，对于具有相同码长的符号，编码从最短长度的第一个码开始，依次递增，随着长度的增加，末尾添加零。对于deflate格式，这些位是按照它们更自然的整数递增顺序的反向存储的，因此在下面的大循环中构建解码表时，整数码是反向递增的。

       此例程假定，但不检查，lens[]中的所有条目都在0..MAXBITS范围内。调用者必须确保这一点。1..MAXBITS被解释为该码长。零表示该符号在此码中不存在。

       通过计算每个码长的代码数量来对代码进行排序，从而创建一个在排序表中每个长度的起始索引的表，并按顺序在排序表中输入符号。排序表是work[]，该空间由调用者提供。

       码长计数也用于其他目的，例如查找最小和最大长度码，确定是否存在任何码，检查有效的码长集，并在构建解码表时预先查看码长计数以确定子表大小。
     */

    /* 累积码长（假设lens[]都在0..MAXBITS范围内） */
    for (len = 0; len <= MAXBITS; len++)
        count[len] = 0;
    for (sym = 0; sym < codes; sym++)
        count[lens[sym]]++;

    /* 限制码长，强制根节点在码长范围内 */
    root = *bits;  # 将指针 bits 指向的值赋给 root
    for (max = MAXBITS; max >= 1; max--)  # 从最大值 MAXBITS 开始递减，直到 1
        if (count[max] != 0) break;  # 如果 count[max] 不等于 0，则跳出循环
    if (root > max) root = max;  # 如果 root 大于 max，则将 root 赋值为 max
    if (max == 0) {  # 如果 max 等于 0
        here.op = (unsigned char)64;  # 将 64 转换为无符号字符类型赋给 here.op
        here.bits = (unsigned char)1;  # 将 1 转换为无符号字符类型赋给 here.bits
        here.val = (unsigned short)0;  # 将 0 转换为无符号短整型赋给 here.val
        *(*table)++ = here;  # 将 here 的值存入 table 指针指向的位置，并将指针向后移动
        *(*table)++ = here;  # 将 here 的值再次存入 table 指针指向的位置，并将指针向后移动
        *bits = 1;  # 将 1 赋值给 bits 指向的值
        return 0;  # 返回 0
    }
    for (min = 1; min < max; min++)  # 从 1 开始递增，直到 max - 1
        if (count[min] != 0) break;  # 如果 count[min] 不等于 0，则跳出循环
    if (root < min) root = min;  # 如果 root 小于 min，则将 root 赋值为 min

    /* check for an over-subscribed or incomplete set of lengths */
    left = 1;  # 将 1 赋值给 left
    for (len = 1; len <= MAXBITS; len++) {  # 从 1 开始递增，直到 MAXBITS
        left <<= 1;  # left 左移 1 位
        left -= count[len];  # left 减去 count[len]
        if (left < 0) return -1;  # 如果 left 小于 0，则返回 -1（表示过度订阅）
    }
    if (left > 0 && (type == CODES || max != 1))  # 如果 left 大于 0 并且（type 等于 CODES 或者 max 不等于 1）
        return -1;  # 返回 -1（表示不完整的集合）

    /* generate offsets into symbol table for each length for sorting */
    offs[1] = 0;  # 将 0 赋值给 offs[1]
    for (len = 1; len < MAXBITS; len++)  # 从 1 开始递增，直到 MAXBITS - 1
        offs[len + 1] = offs[len] + count[len];  # 计算 offs[len + 1] 的值
    /* sort symbols by length, by symbol order within each length */
    for (sym = 0; sym < codes; sym++)  # 从 0 开始递增，直到 codes
        if (lens[sym] != 0) work[offs[lens[sym]]++] = (unsigned short)sym;  # 根据长度和符号顺序对符号进行排序
    /*
       创建并填充解码表。在这个循环中，被填充的表在 next 中，具有 curr 索引位。正在使用的代码是长度为 len 的 huff。该代码通过从底部删除 drop 位转换为索引。对于长度小于 drop + curr 的代码，顶部的 drop + curr - len 位通过所有值递增以填充表中的重复条目。

       root 是根表的索引位数。当 len 超过 root 时，将创建由 huff 的低 root 位索引指向的子表。这将保存在 low 中，以检查何时应开始新的子表。当填充根表时，drop 为零，当填充子表时，drop 为 root。

       当需要新的子表时，需要向前查看代码长度以确定需要多大的子表。长度计数用于此目的，因此 count[] 在输入表中的代码时递减。

       used 跟踪从提供的 *table 空间分配了多少表条目。对于 LENS 和 DIST 表，它与常量 ENOUGH_LENS 和 ENOUGH_DISTS 进行比较，以防止初始根表大小常量的更改。有关更多信息，请参阅 inftrees.h 中的注释。

       sym 递增遍历所有符号，当处理了长度为 max 的所有代码时，即所有代码时，循环终止。此例程允许不完整的代码，因此在此之后的另一个循环将使用无效代码标记填充解码表的其余部分。
     */

    /* 设置代码类型 */
    switch (type) {
    case CODES:
        base = extra = work;    /* 虚拟值--未使用 */
        match = 20;
        break;
    case LENS:
        base = lbase;
        extra = lext;
        match = 257;
        break;
    default:    /* DISTS */
        base = dbase;  /* 将dbase的值赋给base变量 */
        extra = dext;  /* 将dext的值赋给extra变量 */
        match = 0;     /* 将match变量的值设为0 */
    }

    /* initialize state for loop */
    huff = 0;                   /* 初始化huff变量为0，用作起始编码 */
    sym = 0;                    /* 初始化sym变量为0，用作起始编码符号 */
    len = min;                  /* 初始化len变量为min，用作起始编码长度 */
    next = *table;              /* 将table指向的值赋给next变量，用作当前要填充的表 */
    curr = root;                /* 将root的值赋给curr变量，用作当前表的索引位数 */
    drop = 0;                   /* 将drop变量的值设为0，用作当前要从编码中丢弃的位数 */
    low = (unsigned)(-1);       /* 将-1转换为无符号整数并赋给low变量，用作触发新的子表 */
    used = 1U << root;          /* 将1左移root位并赋给used变量，用作使用的根表条目 */
    mask = used - 1;            /* 将used减1的结果赋给mask变量，用作比较low的掩码 */

    /* check available table space */
    if ((type == LENS && used > ENOUGH_LENS) ||  /* 如果type为LENS并且used大于ENOUGH_LENS，或者 */
        (type == DISTS && used > ENOUGH_DISTS))   /* 如果type为DISTS并且used大于ENOUGH_DISTS */
        return 1;   /* 返回1 */

    /* process all codes and make table entries */
    }

    /* fill in remaining table entry if code is incomplete (guaranteed to have
       at most one remaining entry, since if the code is incomplete, the
       maximum code length that was allowed to get this far is one bit) */
    if (huff != 0) {
        here.op = (unsigned char)64;            /* 将64转换为无符号字符并赋给here.op，用作无效编码标记 */
        here.bits = (unsigned char)(len - drop);  /* 将len-drop的结果转换为无符号字符并赋给here.bits，用作编码长度 */
        here.val = (unsigned short)0;  /* 将0转换为无符号短整数并赋给here.val */
        next[huff] = here;  /* 将here赋值给next[huff]，用作填充剩余的表条目 */
    }

    /* set return parameters */
    *table += used;  /* 将table指向的值加上used，并赋给table，用作返回参数 */
    *bits = root;    /* 将root的值赋给bits，用作返回参数 */
    return 0;        /* 返回0 */
# 闭合前面的函数定义
```