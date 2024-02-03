# `nmap\libz\trees.c`

```cpp
/* trees.c -- output deflated data using Huffman coding
 * Copyright (C) 1995-2021 Jean-loup Gailly
 * detect_data_type() function provided freely by Cosmin Truta, 2006
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

/*
 *  ALGORITHM
 *
 *      The "deflation" process uses several Huffman trees. The more
 *      common source values are represented by shorter bit sequences.
 *
 *      Each code tree is stored in a compressed form which is itself
 * a Huffman encoding of the lengths of all the code strings (in
 * ascending order by source values).  The actual code strings are
 * reconstructed from the lengths in the inflate process, as described
 * in the deflate specification.
 *
 *  REFERENCES
 *
 *      Deutsch, L.P.,"'Deflate' Compressed Data Format Specification".
 *      Available in ftp.uu.net:/pub/archiving/zip/doc/deflate-1.1.doc
 *
 *      Storer, James A.
 *          Data Compression:  Methods and Theory, pp. 49-50.
 *          Computer Science Press, 1988.  ISBN 0-7167-8156-5.
 *
 *      Sedgewick, R.
 *          Algorithms, p290.
 *          Addison-Wesley, 1983. ISBN 0-201-06672-6.
 */

/* @(#) $Id$ */

/* #define GEN_TREES_H */

#include "deflate.h"

#ifdef ZLIB_DEBUG
#  include <ctype.h>
#endif

/* ===========================================================================
 * Constants
 */

#define MAX_BL_BITS 7
/* Bit length codes must not exceed MAX_BL_BITS bits */

#define END_BLOCK 256
/* end of block literal code */

#define REP_3_6      16
/* repeat previous bit length 3-6 times (2 bits of repeat count) */

#define REPZ_3_10    17
/* repeat a zero length 3-10 times  (3 bits of repeat count) */

#define REPZ_11_138  18
/* repeat a zero length 11-138 times  (7 bits of repeat count) */

local const int extra_lbits[LENGTH_CODES] /* extra bits for each length code */
   = {0,0,0,0,0,0,0,0,1,1,1,1,2,2,2,2,3,3,3,3,4,4,4,4,5,5,5,5,0};
# 定义额外位数的数组，用于每个距离码
local const int extra_dbits[D_CODES] /* extra bits for each distance code */
   = {0,0,0,0,1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10,11,11,12,12,13,13};

# 定义额外位数的数组，用于每个位长度码
local const int extra_blbits[BL_CODES]/* extra bits for each bit length code */
   = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,2,3,7};

# 定义位长度码的顺序，按概率递减的顺序发送，以避免传输未使用的位长度码的长度
local const uch bl_order[BL_CODES]
   = {16,17,18,0,8,7,9,6,10,5,11,4,12,3,13,2,14,1,15};
/* The lengths of the bit length codes are sent in order of decreasing
 * probability, to avoid transmitting the lengths for unused bit length codes.
 */

/* ===========================================================================
 * 本地数据。这些数据只初始化一次。
 */

# 定义距离码长度，见下面的 dist_code 数组的定义
#define DIST_CODE_LEN  512 /* see definition of array dist_code below */

# 如果定义了 GEN_TREES_H 或者未定义 STDC，则使用下面的代码块
#if defined(GEN_TREES_H) || !defined(STDC)
/* non ANSI compilers may not accept trees.h */

# 定义静态字面树，由于位长度是固定的，所以在堆构建过程中不需要额外的 L_CODES 代码
local ct_data static_ltree[L_CODES+2];
/* The static literal tree. Since the bit lengths are imposed, there is no
 * need for the L_CODES extra codes used during heap construction. However
 * The codes 286 and 287 are needed to build a canonical tree (see _tr_init
 * below).
 */

# 定义静态距离树，由于所有码都使用 5 位，所以实际上是一个简单的树
local ct_data static_dtree[D_CODES];
/* The static distance tree. (Actually a trivial tree since all codes use
 * 5 bits.)
 */

# 距离码，前 256 个值对应距离 3 到 258，后 256 个值对应 15 位距离的前 8 位
uch _dist_code[DIST_CODE_LEN];
/* Distance codes. The first 256 values correspond to the distances
 * 3 .. 258, the last 256 values correspond to the top 8 bits of
 * the 15 bit distances.
 */

# 每个标准化匹配长度的长度码（0 == MIN_MATCH）
uch _length_code[MAX_MATCH-MIN_MATCH+1];
/* length code for each normalized match length (0 == MIN_MATCH) */

# 每个码的第一个标准化长度（0 = MIN_MATCH）
local int base_length[LENGTH_CODES];
/* First normalized length for each code (0 = MIN_MATCH) */

# 每个码的第一个标准化距离（0 = 距离 1）
local int base_dist[D_CODES];
/* First normalized distance for each code (0 = distance of 1) */

# 否则，包含 trees.h 文件
#else
#  include "trees.h"
#endif /* GEN_TREES_H */

# 静态树描述结构体
struct static_tree_desc_s {
    const ct_data *static_tree;  /* static tree or NULL */
    const intf *extra_bits;      /* extra bits for each code or NULL */
    int     extra_base;          /* 用于额外位的基本索引 */
    int     elems;               /* 树中元素的最大数量 */
    int     max_length;          /* 代码的最大位长度 */
# 定义静态树描述结构体，包括静态树、额外位数、起始索引、代码数、最大位数
local const static_tree_desc  static_l_desc =
{static_ltree, extra_lbits, LITERALS+1, L_CODES, MAX_BITS};

# 定义静态树描述结构体，包括静态树、额外位数、起始索引、代码数、最大位数
local const static_tree_desc  static_d_desc =
{static_dtree, extra_dbits, 0,          D_CODES, MAX_BITS};

# 定义静态树描述结构体，包括静态树、额外位数、起始索引、代码数、最大位数
local const static_tree_desc  static_bl_desc =
{(const ct_data *)0, extra_blbits, 0,   BL_CODES, MAX_BL_BITS};

# 本文件中的本地（静态）函数
local void tr_static_init OF((void));
local void init_block     OF((deflate_state *s));
local void pqdownheap     OF((deflate_state *s, ct_data *tree, int k));
local void gen_bitlen     OF((deflate_state *s, tree_desc *desc));
local void gen_codes      OF((ct_data *tree, int max_code, ushf *bl_count));
local void build_tree     OF((deflate_state *s, tree_desc *desc));
local void scan_tree      OF((deflate_state *s, ct_data *tree, int max_code));
local void send_tree      OF((deflate_state *s, ct_data *tree, int max_code));
local int  build_bl_tree  OF((deflate_state *s));
local void send_all_trees OF((deflate_state *s, int lcodes, int dcodes, int blcodes));
local void compress_block OF((deflate_state *s, const ct_data *ltree, const ct_data *dtree));
local int  detect_data_type OF((deflate_state *s));
local unsigned bi_reverse OF((unsigned code, int len));
local void bi_windup      OF((deflate_state *s));
local void bi_flush       OF((deflate_state *s));

# 如果定义了 GEN_TREES_H，则定义生成树头文件的本地函数
#ifdef GEN_TREES_H
local void gen_trees_header OF((void));
#endif

# 如果未定义 ZLIB_DEBUG，则定义发送代码的宏，发送给定树的代码，c 和 tree 不能有副作用
#ifndef ZLIB_DEBUG
#  define send_code(s, c, tree) send_bits(s, tree[c].Code, tree[c].Len)
   /* Send a code of the given tree. c and tree must not have side effects */

# 否则，定义发送代码的宏，如果 z_verbose 大于 2，则在标准错误流中打印代码，然后发送给定树的代码
#else /* !ZLIB_DEBUG */
#  define send_code(s, c, tree) \
     { if (z_verbose>2) fprintf(stderr,"\ncd %3d ",(c)); \
       send_bits(s, tree[c].Code, tree[c].Len); }
#endif
/* ===========================================================================
 * Output a short LSB first on the stream.
 * IN assertion: there is enough room in pendingBuf.
 */
# 将一个短的 LSB（Least Significant Bit，最低有效位）输出到流中。
# 输入断言：pendingBuf 中有足够的空间。

#define put_short(s, w) { \
    put_byte(s, (uch)((w) & 0xff)); \  # 将 w 的最低 8 位写入流中
    put_byte(s, (uch)((ush)(w) >> 8)); \  # 将 w 的高 8 位写入流中
}

/* ===========================================================================
 * Send a value on a given number of bits.
 * IN assertion: length <= 16 and value fits in length bits.
 */
#ifdef ZLIB_DEBUG
local void send_bits      OF((deflate_state *s, int value, int length));

local void send_bits(s, value, length)
    deflate_state *s;
    int value;  /* value to send */  # 要发送的值
    int length; /* number of bits */  # 位数
{
    Tracevv((stderr," l %2d v %4x ", length, value));  # 调试信息
    Assert(length > 0 && length <= 15, "invalid length");  # 断言，长度大于 0 且小于等于 15
    s->bits_sent += (ulg)length;  # 更新已发送位数

    /* If not enough room in bi_buf, use (valid) bits from bi_buf and
     * (16 - bi_valid) bits from value, leaving (width - (16 - bi_valid))
     * unused bits in value.
     */
    if (s->bi_valid > (int)Buf_size - length) {  # 如果 bi_buf 中的空间不够
        s->bi_buf |= (ush)value << s->bi_valid;  # 将 value 的部分位存入 bi_buf
        put_short(s, s->bi_buf);  # 将 bi_buf 中的数据写入流中
        s->bi_buf = (ush)value >> (Buf_size - s->bi_valid);  # 将 value 剩余的位存入 bi_buf
        s->bi_valid += length - Buf_size;  # 更新 bi_valid
    } else {
        s->bi_buf |= (ush)value << s->bi_valid;  # 将 value 存入 bi_buf
        s->bi_valid += length;  # 更新 bi_valid
    }
}
#else /* !ZLIB_DEBUG */

#define send_bits(s, value, length) \
{ int len = length;\
  if (s->bi_valid > (int)Buf_size - len) {\
    int val = (int)value;\
    s->bi_buf |= (ush)val << s->bi_valid;\
    put_short(s, s->bi_buf);\
    s->bi_buf = (ush)val >> (Buf_size - s->bi_valid);\
    s->bi_valid += len - Buf_size;\
  } else {\
    s->bi_buf |= (ush)(value) << s->bi_valid;\
    s->bi_valid += len;\
  }\
}
#endif /* ZLIB_DEBUG */


/* the arguments must not have side effects */

/* ===========================================================================
 * Initialize the various 'constant' tables.
 */
local void tr_static_init()
{
#if defined(GEN_TREES_H) || !defined(STDC)
    // 定义静态变量，用于标记静态初始化是否已完成
    static int static_init_done = 0;
    int n;        /* iterates over tree elements */  // 迭代树元素
    int bits;     /* bit counter */  // 位计数器
    int length;   /* length value */  // 长度值
    int code;     /* code value */  // 编码值
    int dist;     /* distance index */  // 距离索引
    ush bl_count[MAX_BITS+1];
    /* number of codes at each bit length for an optimal tree */
    // 用于存储每个位长度上的代码数量，用于构建最优树

    if (static_init_done) return;

    /* For some embedded targets, global variables are not initialized: */
#ifdef NO_INIT_GLOBAL_POINTERS
    static_l_desc.static_tree = static_ltree;
    static_l_desc.extra_bits = extra_lbits;
    static_d_desc.static_tree = static_dtree;
    static_d_desc.extra_bits = extra_dbits;
    static_bl_desc.extra_bits = extra_blbits;
#endif

    /* Initialize the mapping length (0..255) -> length code (0..28) */
    length = 0;
    for (code = 0; code < LENGTH_CODES-1; code++) {
        base_length[code] = length;
        for (n = 0; n < (1 << extra_lbits[code]); n++) {
            _length_code[length++] = (uch)code;
        }
    }
    Assert (length == 256, "tr_static_init: length != 256");
    /* Note that the length 255 (match length 258) can be represented
     * in two different ways: code 284 + 5 bits or code 285, so we
     * overwrite length_code[255] to use the best encoding:
     */
    _length_code[length - 1] = (uch)code;

    /* Initialize the mapping dist (0..32K) -> dist code (0..29) */
    dist = 0;
    for (code = 0 ; code < 16; code++) {
        base_dist[code] = dist;
        for (n = 0; n < (1 << extra_dbits[code]); n++) {
            _dist_code[dist++] = (uch)code;
        }
    }
    Assert (dist == 256, "tr_static_init: dist != 256");
    dist >>= 7; /* from now on, all distances are divided by 128 */
    for ( ; code < D_CODES; code++) {
        base_dist[code] = dist << 7;
        for (n = 0; n < (1 << (extra_dbits[code] - 7)); n++) {
            _dist_code[256 + dist++] = (uch)code;
        }
    }
    # 检查 dist 是否等于 256，如果不等于则抛出异常
    Assert (dist == 256, "tr_static_init: 256 + dist != 512");

    # 构建静态字面树的编码
    for (bits = 0; bits <= MAX_BITS; bits++) bl_count[bits] = 0;
    n = 0;
    while (n <= 143) static_ltree[n++].Len = 8, bl_count[8]++;
    while (n <= 255) static_ltree[n++].Len = 9, bl_count[9]++;
    while (n <= 279) static_ltree[n++].Len = 7, bl_count[7]++;
    while (n <= 287) static_ltree[n++].Len = 8, bl_count[8]++;
    # 代码 286 和 287 不存在，但是必须包含它们在树的构建中，以获得规范的哈夫曼树（最长的代码都是1）
    gen_codes((ct_data *)static_ltree, L_CODES+1, bl_count);

    # 静态距离树是微不足道的：
    for (n = 0; n < D_CODES; n++) {
        static_dtree[n].Len = 5;
        static_dtree[n].Code = bi_reverse((unsigned)n, 5);
    }
    static_init_done = 1;
# 如果定义了 GEN_TREES_H，则生成树头文件
#ifdef GEN_TREES_H
    # 生成树头文件
    gen_trees_header();
# endif
#endif /* defined(GEN_TREES_H) || !defined(STDC) */

/* ===========================================================================
 * 生成描述静态树的文件 trees.h
 */
#ifdef GEN_TREES_H
#  ifndef ZLIB_DEBUG
#    include <stdio.h>
#  endif

#  定义一个宏，用于在生成树头文件时插入分隔符
#  参数 i: 当前索引, last: 最后一个索引, width: 每行的宽度
#  如果当前索引等于最后一个索引，则插入换行符和结束符号
#  否则，如果当前索引是当前行的最后一个元素，则插入逗号和换行符
#  否则，插入逗号和空格
void gen_trees_header()
{
    FILE *header = fopen("trees.h", "w");
    int i;

    // 断言头文件是否成功打开
    Assert (header != NULL, "Can't open trees.h");
    // 在头文件中写入注释
    fprintf(header,
            "/* header created automatically with -DGEN_TREES_H */\n\n");

    // 写入静态树的本地数据
    fprintf(header, "local const ct_data static_ltree[L_CODES+2] = {\n");
    for (i = 0; i < L_CODES+2; i++) {
        fprintf(header, "{{%3u},{%3u}}%s", static_ltree[i].Code,
                static_ltree[i].Len, SEPARATOR(i, L_CODES+1, 5));
    }

    // 写入静态树的本地数据
    fprintf(header, "local const ct_data static_dtree[D_CODES] = {\n");
    for (i = 0; i < D_CODES; i++) {
        fprintf(header, "{{%2u},{%2u}}%s", static_dtree[i].Code,
                static_dtree[i].Len, SEPARATOR(i, D_CODES-1, 5));
    }

    // 写入内部压缩码的数据
    fprintf(header, "const uch ZLIB_INTERNAL _dist_code[DIST_CODE_LEN] = {\n");
    for (i = 0; i < DIST_CODE_LEN; i++) {
        fprintf(header, "%2u%s", _dist_code[i],
                SEPARATOR(i, DIST_CODE_LEN-1, 20));
    }

    // 写入长度码的数据
    fprintf(header,
        "const uch ZLIB_INTERNAL _length_code[MAX_MATCH-MIN_MATCH+1]= {\n");
    for (i = 0; i < MAX_MATCH-MIN_MATCH+1; i++) {
        fprintf(header, "%2u%s", _length_code[i],
                SEPARATOR(i, MAX_MATCH-MIN_MATCH, 20));
    }

    // 写入长度码的基础数据
    fprintf(header, "local const int base_length[LENGTH_CODES] = {\n");
    for (i = 0; i < LENGTH_CODES; i++) {
        fprintf(header, "%1u%s", base_length[i],
                SEPARATOR(i, LENGTH_CODES-1, 20));
    }

    // 写入距离码的基础数据
    fprintf(header, "local const int base_dist[D_CODES] = {\n");
    # 遍历循环，从 0 到 D_CODES-1
    for (i = 0; i < D_CODES; i++) {
        # 将 base_dist[i] 格式化为 5 个字符的宽度并写入到 header 文件中
        fprintf(header, "%5u%s", base_dist[i],
                # 调用 SEPARATOR 函数，根据 i 的值和 D_CODES-1 以及 10 作为参数，生成分隔符并写入到 header 文件中
                SEPARATOR(i, D_CODES-1, 10));
    }
    # 关闭 header 文件
    fclose(header);
#endif /* GEN_TREES_H */

/* ===========================================================================
 * 为新的 zlib 流初始化树数据结构。
 */
void ZLIB_INTERNAL _tr_init(s)
    deflate_state *s;
{
    tr_static_init();  // 调用静态初始化函数

    s->l_desc.dyn_tree = s->dyn_ltree;  // 将动态 ltree 赋值给 l_desc 的动态树
    s->l_desc.stat_desc = &static_l_desc;  // 将静态 l_desc 赋值给 static_l_desc

    s->d_desc.dyn_tree = s->dyn_dtree;  // 将动态 dtree 赋值给 d_desc 的动态树
    s->d_desc.stat_desc = &static_d_desc;  // 将静态 d_desc 赋值给 static_d_desc

    s->bl_desc.dyn_tree = s->bl_tree;  // 将动态 bl_tree 赋值给 bl_desc 的动态树
    s->bl_desc.stat_desc = &static_bl_desc;  // 将静态 bl_desc 赋值给 static_bl_desc

    s->bi_buf = 0;  // 初始化 bi_buf 为 0
    s->bi_valid = 0;  // 初始化 bi_valid 为 0
#ifdef ZLIB_DEBUG
    s->compressed_len = 0L;  // 如果定义了 ZLIB_DEBUG，则初始化 compressed_len 为 0
    s->bits_sent = 0L;  // 如果定义了 ZLIB_DEBUG，则初始化 bits_sent 为 0
#endif

    /* 初始化第一个文件的第一个块： */
    init_block(s);  // 调用初始化块函数
}

/* ===========================================================================
 * 初始化一个新的块。
 */
local void init_block(s)
    deflate_state *s;
{
    int n; /* iterates over tree elements */  // 迭代树元素

    /* 初始化树。*/
    for (n = 0; n < L_CODES;  n++) s->dyn_ltree[n].Freq = 0;  // 将动态 ltree 的频率初始化为 0
    for (n = 0; n < D_CODES;  n++) s->dyn_dtree[n].Freq = 0;  // 将动态 dtree 的频率初始化为 0
    for (n = 0; n < BL_CODES; n++) s->bl_tree[n].Freq = 0;  // 将 bl_tree 的频率初始化为 0

    s->dyn_ltree[END_BLOCK].Freq = 1;  // 将动态 ltree 的结束块频率初始化为 1
    s->opt_len = s->static_len = 0L;  // 初始化 opt_len 和 static_len 为 0
    s->sym_next = s->matches = 0;  // 初始化 sym_next 和 matches 为 0
}

#define SMALLEST 1
/* Huffman 树中最不频繁节点在堆数组中的索引 */

/* ===========================================================================
 * 从堆中移除最小的元素，并重新创建一个少一个元素的堆。更新堆和堆长度。
 */
#define pqremove(s, tree, top) \
{\
    top = s->heap[SMALLEST]; \  // 将堆中最小元素赋值给 top
    s->heap[SMALLEST] = s->heap[s->heap_len--]; \  // 将堆中最后一个元素赋值给堆中最小元素，同时更新堆长度
    pqdownheap(s, tree, SMALLEST); \  // 调用 pqdownheap 函数重新创建堆
}

/* ===========================================================================
 * 比较两个子树，使用树深度作为绑定器当子树具有相等频率时。这最小化了最坏情况长度。
 */
/* 定义一个宏函数，用于比较树节点的频率和深度，返回较小的节点的索引 */
#define smaller(tree, n, m, depth) \
   (tree[n].Freq < tree[m].Freq || \
   (tree[n].Freq == tree[m].Freq && depth[n] <= depth[m]))

/* ===========================================================================
 * 通过从节点 k 开始向下移动树来恢复堆属性，如果必要，交换节点和其两个儿子中较小的那个，
 * 当堆属性被重新建立时停止（每个父节点都比其两个儿子小）。
 */
local void pqdownheap(s, tree, k)
    deflate_state *s;
    ct_data *tree;  /* 需要恢复的树 */
    int k;               /* 需要向下移动的节点 */
{
    int v = s->heap[k];
    int j = k << 1;  /* k 的左儿子 */
    while (j <= s->heap_len) {
        /* 将 j 设置为两个儿子中较小的那个： */
        if (j < s->heap_len &&
            smaller(tree, s->heap[j + 1], s->heap[j], s->depth)) {
            j++;
        }
        /* 如果 v 比两个儿子都小，则退出循环 */
        if (smaller(tree, v, s->heap[j], s->depth)) break;

        /* 用较小的儿子交换 v */
        s->heap[k] = s->heap[j];  k = j;

        /* 然后继续向下移动树，将 j 设置为 k 的左儿子 */
        j <<= 1;
    }
    s->heap[k] = v;
}

/* ===========================================================================
 * 计算树的最佳位长度，并更新当前块的总位长度。
 * 输入断言：已设置 freq 和 dad 字段，heap[heap_max] 及以上为按频率递增排序的树节点。
 * 输出断言：len 字段设置为最佳位长度，bl_count 数组包含每个位长度的频率。
 * opt_len 被更新；如果 stree 不为空，则 static_len 也被更新。
 */
local void gen_bitlen(s, desc)
    deflate_state *s;
    tree_desc *desc;    /* 树描述符 */
{
    ct_data *tree        = desc->dyn_tree;
    int max_code         = desc->max_code;
    // 从描述符中获取静态树
    const ct_data *stree = desc->stat_desc->static_tree;
    // 从描述符中获取额外位
    const intf *extra    = desc->stat_desc->extra_bits;
    // 从描述符中获取额外位的基数
    int base             = desc->stat_desc->extra_base;
    // 从描述符中获取最大长度
    int max_length       = desc->stat_desc->max_length;
    int h;              /* 堆索引 */
    int n, m;           /* 遍历树元素 */
    int bits;           /* 位长度 */
    int xbits;          /* 额外位 */
    ush f;              /* 频率 */
    int overflow = 0;   /* 位长度过大的元素数量 */

    // 初始化位长度计数数组
    for (bits = 0; bits <= MAX_BITS; bits++) s->bl_count[bits] = 0;

    /* 在第一次遍历中，计算最佳位长度（在位长度树溢出的情况下） */
    tree[s->heap[s->heap_max]].Len = 0; /* 堆的根节点 */

    for (h = s->heap_max + 1; h < HEAP_SIZE; h++) {
        n = s->heap[h];
        bits = tree[tree[n].Dad].Len + 1;
        if (bits > max_length) bits = max_length, overflow++;
        tree[n].Len = (ush)bits;
        /* 覆盖不再需要的 tree[n].Dad */

        if (n > max_code) continue; /* 非叶节点 */

        s->bl_count[bits]++;
        xbits = 0;
        if (n >= base) xbits = extra[n - base];
        f = tree[n].Freq;
        s->opt_len += (ulg)f * (unsigned)(bits + xbits);
        if (stree) s->static_len += (ulg)f * (unsigned)(stree[n].Len + xbits);
    }
    if (overflow == 0) return;

    Tracev((stderr,"\nbit length overflow\n"));
    /* 例如，在 Calgary 语料库的 obj2 和 pic 中会发生这种情况 */

    /* 找到可能增加的第一个位长度： */
    do {
        // 设置初始比特位数为最大长度减一
        bits = max_length - 1;
        // 找到第一个非零的叶子节点的比特位数
        while (s->bl_count[bits] == 0) bits--;
        // 将一个叶子节点向下移动一步
        s->bl_count[bits]--;        
        // 将一个溢出项的兄弟节点向上移动两步
        s->bl_count[bits + 1] += 2; 
        // 减少最大长度对应的叶子节点数量
        s->bl_count[max_length]--;
        /* 溢出项的兄弟节点也向上移动一步，
         * 但这不会影响bl_count[max_length]
         */
        overflow -= 2;
    } while (overflow > 0);

    /* 现在重新计算所有比特长度，按照频率递增的顺序扫描。
     * h仍然等于HEAP_SIZE。（重新构建所有长度比仅修复错误的长度更简单。
     * 这个想法来自于Haruhiko Okumura编写的'ar'。）
     */
    for (bits = max_length; bits != 0; bits--) {
        n = s->bl_count[bits];
        while (n != 0) {
            m = s->heap[--h];
            if (m > max_code) continue;
            if ((unsigned) tree[m].Len != (unsigned) bits) {
                // 如果树中的节点长度与当前比特位数不匹配，则更新长度并计算优化长度
                Tracev((stderr,"code %d bits %d->%d\n", m, tree[m].Len, bits));
                s->opt_len += ((ulg)bits - tree[m].Len) * tree[m].Freq;
                tree[m].Len = (ush)bits;
            }
            n--;
        }
    }
# 生成给定树和位计数的代码（不一定是最佳的）。
# 输入断言：bl_count数组包含给定树的位长度统计信息，并且所有树元素的len字段都已设置。
# 输出断言：对于所有非零代码长度的树元素，都已设置code字段。
local void gen_codes(tree, max_code, bl_count)
    ct_data *tree;             /* 要装饰的树 */
    int max_code;              /* 具有非零频率的最大代码 */
    ushf *bl_count;            /* 每个位长度的代码数量 */

{
    ush next_code[MAX_BITS+1]; /* 每个位长度的下一个代码值 */
    unsigned code = 0;         /* 运行的代码值 */
    int bits;                  /* 位索引 */
    int n;                     /* 代码索引 */

    /* 首先使用分布计数来生成代码值，而不进行位反转。 */
    for (bits = 1; bits <= MAX_BITS; bits++) {
        code = (code + bl_count[bits - 1]) << 1;
        next_code[bits] = (ush)code;
    }
    /* 检查bl_count中的位计数是否一致。最后一个代码必须全为1。 */
    Assert (code + bl_count[MAX_BITS] - 1 == (1 << MAX_BITS) - 1,
            "inconsistent bit counts");
    Tracev((stderr,"\ngen_codes: max_code %d ", max_code));

    for (n = 0;  n <= max_code; n++) {
        int len = tree[n].Len;
        if (len == 0) continue;
        /* 现在反转位 */
        tree[n].Code = (ush)bi_reverse(next_code[len]++, len);

        Tracecv(tree != static_ltree, (stderr,"\nn %3d %c l %2d c %4x (%x) ",
            n, (isgraph(n) ? n : ' '), len, tree[n].Code, next_code[len] - 1));
    }
}
# 构建一个哈夫曼树并分配代码位字符串和长度，更新当前块的总位长度
# 输入断言：对于所有树元素，freq字段都已设置
# 输出断言：len和code字段设置为最佳位长度和相应的代码，更新opt_len；如果stree不为空，static_len也会更新。max_code字段设置。
local void build_tree(s, desc)
    deflate_state *s;
    tree_desc *desc; /* 树描述符 */
{
    ct_data *tree         = desc->dyn_tree;  # 动态树
    const ct_data *stree  = desc->stat_desc->static_tree;  # 静态树
    int elems             = desc->stat_desc->elems;  # 元素数量
    int n, m;          /* 遍历堆元素 */
    int max_code = -1; /* 非零频率的最大代码 */
    int node;          /* 正在创建的新节点 */

    /* 构建初始堆，最不频繁的元素在堆[SMALLEST]中。堆[n]的子节点是堆[2*n]和堆[2*n + 1]。
     * 堆[0]未使用。
     */
    s->heap_len = 0, s->heap_max = HEAP_SIZE;

    for (n = 0; n < elems; n++) {
        if (tree[n].Freq != 0) {
            s->heap[++(s->heap_len)] = max_code = n;
            s->depth[n] = 0;
        } else {
            tree[n].Len = 0;
        }
    }

    /* pkzip格式要求至少存在一个距离代码，并且即使只有一个可能的代码，也应发送至少一个位。
     * 因此，为了避免以后的特殊检查，我们强制至少有两个非零频率的代码。
     */
    while (s->heap_len < 2) {
        node = s->heap[++(s->heap_len)] = (max_code < 2 ? ++max_code : 0);
        tree[node].Freq = 1;
        s->depth[node] = 0;
        s->opt_len--; if (stree) s->static_len -= stree[node].Len;
        /* 节点为0或1，因此它没有额外的位 */
    }
    desc->max_code = max_code;
}
    /* 堆中的元素 heap[heap_len/2 + 1 .. heap_len] 是树的叶子节点，
     * 建立递增长度的子堆：
     */
    for (n = s->heap_len/2; n >= 1; n--) pqdownheap(s, tree, n);

    /* 通过反复组合出现频率最低的两个节点来构建哈夫曼树 */
    node = elems;              /* 树的下一个内部节点 */
    do {
        pqremove(s, tree, n);  /* n = 出现频率最低的节点 */
        m = s->heap[SMALLEST]; /* m = 出现频率次低的节点 */

        s->heap[--(s->heap_max)] = n; /* 保持节点按频率排序 */
        s->heap[--(s->heap_max)] = m;

        /* 创建一个新节点，作为 n 和 m 的父节点 */
        tree[node].Freq = tree[n].Freq + tree[m].Freq;
        s->depth[node] = (uch)((s->depth[n] >= s->depth[m] ?
                                s->depth[n] : s->depth[m]) + 1);
        tree[n].Dad = tree[m].Dad = (ush)node;
#ifdef DUMP_BL_TREE
        # 如果需要输出 BL 树的信息，则在标准错误输出流中打印节点信息
        if (tree == s->bl_tree) {
            fprintf(stderr,"\nnode %d(%d), sons %d(%d) %d(%d)",
                    node, tree[node].Freq, n, tree[n].Freq, m, tree[m].Freq);
        }
#endif
        /* and insert the new node in the heap */
        # 将新节点插入堆中
        s->heap[SMALLEST] = node++;
        pqdownheap(s, tree, SMALLEST);

    } while (s->heap_len >= 2);

    # 将堆中最小的节点移动到堆的末尾
    s->heap[--(s->heap_max)] = s->heap[SMALLEST];

    /* At this point, the fields freq and dad are set. We can now
     * generate the bit lengths.
     */
    # 在这一点上，freq 和 dad 字段已经设置好了，现在可以生成位长度
    gen_bitlen(s, (tree_desc *)desc);

    /* The field len is now set, we can generate the bit codes */
    # 现在字段 len 已经设置好了，可以生成位编码
    gen_codes ((ct_data *)tree, max_code, s->bl_count);
}

/* ===========================================================================
 * Scan a literal or distance tree to determine the frequencies of the codes
 * in the bit length tree.
 */
local void scan_tree(s, tree, max_code)
    deflate_state *s;
    ct_data *tree;   /* the tree to be scanned */
    int max_code;    /* and its largest code of non zero frequency */
{
    int n;                     /* iterates over all tree elements */
    int prevlen = -1;          /* last emitted length */
    int curlen;                /* length of current code */
    int nextlen = tree[0].Len; /* length of next code */
    int count = 0;             /* repeat count of the current code */
    int max_count = 7;         /* max repeat count */
    int min_count = 4;         /* min repeat count */

    if (nextlen == 0) max_count = 138, min_count = 3;
    tree[max_code + 1].Len = (ush)0xffff; /* guard */
    # 遍历从 0 到最大编码的范围
    for (n = 0; n <= max_code; n++) {
        # 当前长度等于下一个长度，更新当前长度和下一个长度
        curlen = nextlen; nextlen = tree[n + 1].Len;
        # 如果计数小于最大计数并且当前长度等于下一个长度，则继续下一次循环
        if (++count < max_count && curlen == nextlen) {
            continue;
        # 如果计数小于最小计数
        } else if (count < min_count) {
            # 更新静态树的频率
            s->bl_tree[curlen].Freq += count;
        # 如果当前长度不为 0
        } else if (curlen != 0) {
            # 如果当前长度不等于上一个长度，更新静态树的频率
            if (curlen != prevlen) s->bl_tree[curlen].Freq++;
            # 更新静态树的频率
            s->bl_tree[REP_3_6].Freq++;
        # 如果计数小于等于 10
        } else if (count <= 10) {
            # 更新静态树的频率
            s->bl_tree[REPZ_3_10].Freq++;
        # 否则
        } else {
            # 更新静态树的频率
            s->bl_tree[REPZ_11_138].Freq++;
        }
        # 重置计数和上一个长度
        count = 0; prevlen = curlen;
        # 如果下一个长度为 0
        if (nextlen == 0) {
            # 更新最大计数和最小计数
            max_count = 138, min_count = 3;
        # 如果当前长度等于下一个长度
        } else if (curlen == nextlen) {
            # 更新最大计数和最小计数
            max_count = 6, min_count = 3;
        # 否则
        } else {
            # 更新最大计数和最小计数
            max_count = 7, min_count = 4;
        }
    }
}
/* ===========================================================================
 * 以压缩形式发送字面量或距离树，使用 bl_tree 中的代码。
 */
local void send_tree(s, tree, max_code)
    deflate_state *s;
    ct_data *tree; /* 要扫描的树 */
    int max_code;       /* 非零频率的最大代码 */
{
    int n;                     /* 遍历所有树元素 */
    int prevlen = -1;          /* 上次发出的长度 */
    int curlen;                /* 当前代码的长度 */
    int nextlen = tree[0].Len; /* 下一个代码的长度 */
    int count = 0;             /* 当前代码的重复次数 */
    int max_count = 7;         /* 最大重复次数 */
    int min_count = 4;         /* 最小重复次数 */

    /* tree[max_code + 1].Len = -1; */  /* 守卫已经设置 */
    if (nextlen == 0) max_count = 138, min_count = 3;

    for (n = 0; n <= max_code; n++) {
        curlen = nextlen; nextlen = tree[n + 1].Len;
        if (++count < max_count && curlen == nextlen) {
            continue;
        } else if (count < min_count) {
            do { send_code(s, curlen, s->bl_tree); } while (--count != 0);

        } else if (curlen != 0) {
            if (curlen != prevlen) {
                send_code(s, curlen, s->bl_tree); count--;
            }
            Assert(count >= 3 && count <= 6, " 3_6?");
            send_code(s, REP_3_6, s->bl_tree); send_bits(s, count - 3, 2);

        } else if (count <= 10) {
            send_code(s, REPZ_3_10, s->bl_tree); send_bits(s, count - 3, 3);

        } else {
            send_code(s, REPZ_11_138, s->bl_tree); send_bits(s, count - 11, 7);
        }
        count = 0; prevlen = curlen;
        if (nextlen == 0) {
            max_count = 138, min_count = 3;
        } else if (curlen == nextlen) {
            max_count = 6, min_count = 3;
        } else {
            max_count = 7, min_count = 4;
        }
    }
}
# 构建用于比特长度的哈夫曼树，并返回在 bl_order 中要发送的最后一个比特长度代码的索引
def build_bl_tree(s):
    # 非零频率的最后一个比特长度代码的索引
    max_blindex;

    # 确定文本和距离树的比特长度频率
    scan_tree(s, (ct_data *)s->dyn_ltree, s->l_desc.max_code)
    scan_tree(s, (ct_data *)s->dyn_dtree, s->d_desc.max_code)

    # 构建比特长度树
    build_tree(s, (tree_desc *)(&(s->bl_desc)))
    # opt_len 现在包括树表示的长度，除了比特长度代码的长度和用于计数的 5 + 5 + 4 位。

    # 确定要发送的比特长度代码的数量。pkzip 格式要求至少发送 4 个比特长度代码。（appnote.txt 说 3 但实际使用的值是 4。）
    for (max_blindex = BL_CODES-1; max_blindex >= 3; max_blindex--):
        if (s->bl_tree[bl_order[max_blindex]].Len != 0) break;
    # 更新 opt_len 以包括比特长度树和计数
    s->opt_len += 3*((ulg)max_blindex + 1) + 5 + 5 + 4;
    Tracev((stderr, "\ndyn trees: dyn %ld, stat %ld", s->opt_len, s->static_len))

    return max_blindex;

# 使用动态哈夫曼树发送块的头部：计数、比特长度代码的长度、文本树和距离树
# 输入断言：lcodes >= 257, dcodes >= 1, blcodes >= 4
def send_all_trees(s, lcodes, dcodes, blcodes):
    # 在 bl_order 中的索引
    rank;

    Assert (lcodes >= 257 && dcodes >= 1 && blcodes >= 4, "not enough codes")
    # 检查码的数量是否超出预期范围，如果是则抛出异常
    Assert (lcodes <= L_CODES && dcodes <= D_CODES && blcodes <= BL_CODES,
            "too many codes");
    # 打印调试信息，输出 bl counts
    Tracev((stderr, "\nbl counts: "));
    # 发送 lcodes - 257 的比特位，长度为 5
    send_bits(s, lcodes - 257, 5);  /* not +255 as stated in appnote.txt */
    # 发送 dcodes - 1 的比特位，长度为 5
    send_bits(s, dcodes - 1,   5);
    # 发送 blcodes - 4 的比特位，长度为 4
    send_bits(s, blcodes - 4,  4);  /* not -3 as stated in appnote.txt */
    # 遍历 blcodes，发送每个 bl_order[rank] 对应的比特位
    for (rank = 0; rank < blcodes; rank++) {
        # 打印调试信息，输出 bl code 和对应的序号
        Tracev((stderr, "\nbl code %2d ", bl_order[rank]));
        # 发送 s->bl_tree[bl_order[rank]].Len 的比特位，长度为 3
        send_bits(s, s->bl_tree[bl_order[rank]].Len, 3);
    }
    # 打印调试信息，输出 bl tree: sent 和已发送的比特位数
    Tracev((stderr, "\nbl tree: sent %ld", s->bits_sent));

    # 发送 literal tree 的比特位
    send_tree(s, (ct_data *)s->dyn_ltree, lcodes - 1);  /* literal tree */
    # 打印调试信息，输出 lit tree: sent 和已发送的比特位数
    Tracev((stderr, "\nlit tree: sent %ld", s->bits_sent));

    # 发送 distance tree 的比特位
    send_tree(s, (ct_data *)s->dyn_dtree, dcodes - 1);  /* distance tree */
    # 打印调试信息，输出 dist tree: sent 和已发送的比特位数
    Tracev((stderr, "\ndist tree: sent %ld", s->bits_sent));
/* ===========================================================================
 * 发送一个存储块
 */
void ZLIB_INTERNAL _tr_stored_block(s, buf, stored_len, last)
    deflate_state *s;
    charf *buf;       /* 输入块 */
    ulg stored_len;   /* 输入块的长度 */
    int last;         /* 如果这是文件的最后一个块，则为1 */
{
    send_bits(s, (STORED_BLOCK<<1) + last, 3);  /* 发送块类型 */
    bi_windup(s);        /* 对齐到字节边界 */
    put_short(s, (ush)stored_len);  /* 存储块的长度 */
    put_short(s, (ush)~stored_len);  /* 存储块长度的反码 */
    if (stored_len)
        zmemcpy(s->pending_buf + s->pending, (Bytef *)buf, stored_len);  /* 如果存储块长度不为0，则将输入块复制到待处理缓冲区中 */
    s->pending += stored_len;  /* 更新待处理缓冲区的长度 */
#ifdef ZLIB_DEBUG
    s->compressed_len = (s->compressed_len + 3 + 7) & (ulg)~7L;  /* 更新压缩长度 */
    s->compressed_len += (stored_len + 4) << 3;  /* 更新压缩长度 */
    s->bits_sent += 2*16;  /* 更新发送的位数 */
    s->bits_sent += stored_len << 3;  /* 更新发送的位数 */
#endif
}

/* ===========================================================================
 * 刷新位缓冲区中的位到待处理输出（最多保留7位）
 */
void ZLIB_INTERNAL _tr_flush_bits(s)
    deflate_state *s;
{
    bi_flush(s);  /* 刷新位缓冲区 */
}

/* ===========================================================================
 * 发送一个空的静态块，以便为解压提供足够的前瞻
 * 这需要10位，其中7位可能保留在位缓冲区中
 */
void ZLIB_INTERNAL _tr_align(s)
    deflate_state *s;
{
    send_bits(s, STATIC_TREES<<1, 3);  /* 发送块类型 */
    send_code(s, END_BLOCK, static_ltree);  /* 发送代码 */
#ifdef ZLIB_DEBUG
    s->compressed_len += 10L; /* 3位用于块类型，7位用于EOB */
#endif
    bi_flush(s);  /* 刷新位缓冲区 */
}

/* ===========================================================================
 * 确定当前块的最佳编码：动态树、静态树或存储，并写出编码块
 */
void ZLIB_INTERNAL _tr_flush_block(s, buf, stored_len, last)
    deflate_state *s;
    charf *buf;       /* 输入块，如果太旧则为NULL */
    ulg stored_len;   /* 输入块的长度 */
    int last;         /* 定义一个整型变量last，用于表示是否为文件的最后一个块 */
{
    ulg opt_lenb, static_lenb; /* opt_len and static_len in bytes */  // 定义变量 opt_lenb 和 static_lenb，表示以字节为单位的 opt_len 和 static_len
    int max_blindex = 0;  /* index of last bit length code of non zero freq */  // 定义变量 max_blindex，表示非零频率的最后一个比特长度码的索引

    /* Build the Huffman trees unless a stored block is forced */
    if (s->level > 0) {  // 如果压缩级别大于 0

        /* Check if the file is binary or text */
        if (s->strm->data_type == Z_UNKNOWN)  // 如果数据类型未知
            s->strm->data_type = detect_data_type(s);  // 调用 detect_data_type 函数检测数据类型

        /* Construct the literal and distance trees */
        build_tree(s, (tree_desc *)(&(s->l_desc)));  // 构建字面量和距离树
        Tracev((stderr, "\nlit data: dyn %ld, stat %ld", s->opt_len,
                s->static_len));  // 输出调试信息

        build_tree(s, (tree_desc *)(&(s->d_desc)));  // 构建距离树
        Tracev((stderr, "\ndist data: dyn %ld, stat %ld", s->opt_len,
                s->static_len));  // 输出调试信息
        /* At this point, opt_len and static_len are the total bit lengths of
         * the compressed block data, excluding the tree representations.
         */

        /* Build the bit length tree for the above two trees, and get the index
         * in bl_order of the last bit length code to send.
         */
        max_blindex = build_bl_tree(s);  // 构建比特长度树，并获取最后一个比特长度码的索引

        /* Determine the best encoding. Compute the block lengths in bytes. */
        opt_lenb = (s->opt_len + 3 + 7) >> 3;  // 计算最佳编码的块长度（以字节为单位）
        static_lenb = (s->static_len + 3 + 7) >> 3;  // 计算静态编码的块长度（以字节为单位）
        Tracev((stderr, "\nopt %lu(%lu) stat %lu(%lu) stored %lu lit %u ",
                opt_lenb, s->opt_len, static_lenb, s->static_len, stored_len,
                s->sym_next / 3));  // 输出调试信息

#ifndef FORCE_STATIC
        if (static_lenb <= opt_lenb || s->strategy == Z_FIXED)  // 如果静态编码长度小于等于最佳编码长度，或者策略为 Z_FIXED
#endif
            opt_lenb = static_lenb;  // 则将最佳编码长度设为静态编码长度

    } else {
        Assert(buf != (char*)0, "lost buf");  // 断言 buf 不为空，否则输出错误信息
        opt_lenb = static_lenb = stored_len + 5; /* force a stored block */  // 强制存储块，设置最佳编码长度和静态编码长度为存储长度加 5
    }

#ifdef FORCE_STORED
    if (buf != (char*)0) { /* force stored block */  // 如果 buf 不为空，强制存储块
#else
    if (stored_len + 4 <= opt_lenb && buf != (char*)0) {  // 如果存储长度加 4小于等于最佳编码长度，并且 buf 不为空
                       /* 4: two words for the lengths */
#endif
        /* 如果 LIT_BUFSIZE > WSIZE，则需要检查 buf 是否为 NULL。
         * 否则，自上次块刷新以来，我们不可能处理超过 WSIZE 字节的输入数据，
         * 因为压缩已经成功。如果 LIT_BUFSIZE <= WSIZE，则将块转换为存储块永远不会太迟。
         */
        _tr_stored_block(s, buf, stored_len, last);

    } else if (static_lenb == opt_lenb) {
        send_bits(s, (STATIC_TREES<<1) + last, 3);
        compress_block(s, (const ct_data *)static_ltree,
                       (const ct_data *)static_dtree);
#ifdef ZLIB_DEBUG
        s->compressed_len += 3 + s->static_len;
#endif
    } else {
        send_bits(s, (DYN_TREES<<1) + last, 3);
        send_all_trees(s, s->l_desc.max_code + 1, s->d_desc.max_code + 1,
                       max_blindex + 1);
        compress_block(s, (const ct_data *)s->dyn_ltree,
                       (const ct_data *)s->dyn_dtree);
#ifdef ZLIB_DEBUG
        s->compressed_len += 3 + s->opt_len;
#endif
    }
    Assert (s->compressed_len == s->bits_sent, "bad compressed size");
    /* 上述检查是模 2^32 运算，用于大于 512 MB 的文件和在 32 位系统上实现的 uLong。 */
    init_block(s);

    if (last) {
        bi_windup(s);
#ifdef ZLIB_DEBUG
        s->compressed_len += 7;  /* 对齐到字节边界 */
#endif
    }
    Tracev((stderr,"\ncomprlen %lu(%lu) ", s->compressed_len >> 3,
           s->compressed_len - 7*last));
}

/* ===========================================================================
 * 保存匹配信息并统计频率计数。如果当前块必须刷新，则返回 true。
 */
int ZLIB_INTERNAL _tr_tally(s, dist, lc)
    deflate_state *s;
    unsigned dist;  /* 匹配字符串的距离 */
    unsigned lc;    /* 匹配长度 - MIN_MATCH 或未匹配的字符（dist==0） */
{
    s->sym_buf[s->sym_next++] = (uch)dist;
    s->sym_buf[s->sym_next++] = (uch)(dist >> 8);
    # 将字符 lc 转换为无符号字符并存储到 sym_buf 中，同时更新 sym_next 指针
    s->sym_buf[s->sym_next++] = (uch)lc;
    # 如果距离为 0，则 lc 是未匹配的字符
    if (dist == 0) {
        # 增加 lc 对应的动态字母树的频率
        s->dyn_ltree[lc].Freq++;
    } else {
        # 匹配成功，增加匹配次数
        s->matches++;
        # 在这里，lc 是匹配长度 - MIN_MATCH
        dist--;             # dist = 匹配距离 - 1
        # 断言距离和 lc 的取值范围符合条件
        Assert((ush)dist < (ush)MAX_DIST(s) &&
               (ush)lc <= (ush)(MAX_MATCH-MIN_MATCH) &&
               (ush)d_code(dist) < (ush)D_CODES,  "_tr_tally: bad match");

        # 增加长度码对应的动态字母树的频率
        s->dyn_ltree[_length_code[lc] + LITERALS + 1].Freq++;
        # 增加距离码对应的动态距离树的频率
        s->dyn_dtree[d_code(dist)].Freq++;
    }
    # 返回是否已经处理完所有输入字符
    return (s->sym_next == s->sym_end);
# 压缩块数据，使用给定的哈夫曼树进行压缩
def compress_block(s, ltree, dtree):
    # 距离匹配的字符串
    dist;      
    # 匹配长度或未匹配的字符（如果距离为0）
    lc;             
    # sym_buf中的运行索引
    sx = 0;    
    # 要发送的代码
    code;      
    # 要发送的额外位数
    extra;          

    # 如果sym_next不为0，则执行以下操作
    if (s.sym_next != 0) do:
        # 从sym_buf中获取距离
        dist = s.sym_buf[sx++] & 0xff;
        dist += (unsigned)(s.sym_buf[sx++] & 0xff) << 8;
        lc = s.sym_buf[sx++];
        # 如果距离为0
        if (dist == 0):
            # 发送一个字面字节
            send_code(s, lc, ltree); 
            Tracecv(isgraph(lc), (stderr," '%c' ", lc));
        else:
            # 在这里，lc是匹配长度 - MIN_MATCH
            code = _length_code[lc];
            # 发送长度代码
            send_code(s, code + LITERALS + 1, ltree);   
            extra = extra_lbits[code];
            # 如果额外位数不为0
            if (extra != 0):
                lc -= base_length[code];
                # 发送额外长度位
                send_bits(s, lc, extra);       
            dist--; # 现在dist是匹配距离 - 1
            code = d_code(dist);
            Assert (code < D_CODES, "bad d_code");

            # 发送距离代码
            send_code(s, code, dtree);       
            extra = extra_dbits[code];
            # 如果额外位数不为0
            if (extra != 0):
                dist -= (unsigned)base_dist[code];
                # 发送额外距离位
                send_bits(s, dist, extra);   
        # 字面或匹配对？
        # 检查pending_buf和sym_buf之间的叠加是否正确
        Assert(s.pending < s.lit_bufsize + sx, "pendingBuf overflow");

    # 循环直到sx < s.sym_next
    while (sx < s.sym_next):
        # 发送结束块代码
        send_code(s, END_BLOCK, ltree);
/* ===========================================================================
 * 检查数据类型是否为文本或二进制，使用以下算法：
 * - 如果满足以下两个条件，则为文本：
 *    a) 没有属于“block list”（0..6, 14..25, 28..31）的非可移植控制字符。
 *    b) 至少有一个属于“allow list”（9 {TAB}, 10 {LF}, 13 {CR}, 32..255）的可打印字符。
 * - 否则为二进制。
 * - 以下部分可移植控制字符构成“gray list”，在此检测算法中被忽略：
 *   (7 {BEL}, 8 {BS}, 11 {VT}, 12 {FF}, 26 {SUB}, 27 {ESC}).
 * 断言：dyn_ltree的Freq字段已设置。
 */
local int detect_data_type(s)
    deflate_state *s;
{
    /* block_mask是块列表字节的位掩码
     * 设置位0..6, 14..25, 和 28..31
     * 0xf3ffc07f = 二进制 11110011111111111100000001111111
     */
    unsigned long block_mask = 0xf3ffc07fUL;
    int n;

    /* 检查非文本（“block-listed”）字节。 */
    for (n = 0; n <= 31; n++, block_mask >>= 1)
        if ((block_mask & 1) && (s->dyn_ltree[n].Freq != 0))
            return Z_BINARY;

    /* 检查文本（“allow-listed”）字节。 */
    if (s->dyn_ltree[9].Freq != 0 || s->dyn_ltree[10].Freq != 0
            || s->dyn_ltree[13].Freq != 0)
        return Z_TEXT;
    for (n = 32; n < LITERALS; n++)
        if (s->dyn_ltree[n].Freq != 0)
            return Z_TEXT;

    /* 没有“block-listed”或“allow-listed”字节：
     * 此流要么为空，要么只有被容忍的（“gray-listed”）字节。
     */
    return Z_BINARY;
}

/* ===========================================================================
 * 反转代码的前len位，使用直接的代码（更快的方法会使用表）
 * 断言：1 <= len <= 15
 */
local unsigned bi_reverse(code, len)
    unsigned code; /* 要反转的值 */
    int len;       /* 其位长度 */
    # 定义一个无符号整数变量 res，并初始化为 0
    register unsigned res = 0;
    # 使用 do-while 循环，对 code 进行位运算，直到 len 变为 0
    do {
        # 将 code 和 1 进行按位或运算，并将结果赋值给 res
        res |= code & 1;
        # 将 code 右移一位，将结果赋值给 code；将 res 左移一位，将结果赋值给 res
        code >>= 1, res <<= 1;
    } while (--len > 0);
    # 返回 res 右移一位的结果
    return res >> 1;
# 刷新位缓冲区，保留最多7位
local void bi_flush(s)
    # 如果位缓冲区中有16位数据
    if (s->bi_valid == 16) {
        # 将位缓冲区中的数据以短整型形式输出，并重置缓冲区和有效位数
        put_short(s, s->bi_buf);
        s->bi_buf = 0;
        s->bi_valid = 0;
    # 如果位缓冲区中有8位或更多数据
    } else if (s->bi_valid >= 8) {
        # 将位缓冲区中的数据以字节形式输出，并右移8位，同时减少有效位数
        put_byte(s, (Byte)s->bi_buf);
        s->bi_buf >>= 8;
        s->bi_valid -= 8;
    }
}

# 刷新位缓冲区并将输出对齐到字节边界
local void bi_windup(s)
    # 如果位缓冲区中的有效位数大于8
    if (s->bi_valid > 8) {
        # 将位缓冲区中的数据以短整型形式输出
        put_short(s, s->bi_buf);
    # 如果位缓冲区中的有效位数大于0
    } else if (s->bi_valid > 0) {
        # 将位缓冲区中的数据以字节形式输出
        put_byte(s, (Byte)s->bi_buf);
    # 重置位缓冲区和有效位数
    s->bi_buf = 0;
    s->bi_valid = 0;
    # 在调试模式下，调整已发送的位数，使其按字节对齐
#ifdef ZLIB_DEBUG
    s->bits_sent = (s->bits_sent + 7) & ~7;
#endif
}
```