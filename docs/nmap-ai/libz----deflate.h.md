# `nmap\libz\deflate.h`

```cpp
/* deflate.h -- internal compression state
 * 压缩状态的内部定义
 * 版权所有 (C) 1995-2018 Jean-loup Gailly
 * 分发和使用条件，请参见 zlib.h 中的版权声明
 */

/* 警告：应用程序不应使用此文件。它是压缩库实现的一部分，可能会更改。应用程序应仅使用 zlib.h。
 */

/* @(#) $Id$ */

#ifndef DEFLATE_H
#define DEFLATE_H

#include "zutil.h"

/* 如果要在编译时禁用 deflate() 创建 gzip 头和尾部，请在编译时定义 NO_GZIP。NO_GZIP 用于避免在不需要时链接 crc 代码。对于共享库，应保留 gzip 编码。 */
#ifndef NO_GZIP
#  define GZIP
#endif

/* ===========================================================================
 * 内部压缩状态。
 */

#define LENGTH_CODES 29
/* 长度码的数量，不包括特殊的 END_BLOCK 码 */

#define LITERALS  256
/* 字面量字节的数量 0..255 */

#define L_CODES (LITERALS+1+LENGTH_CODES)
/* 字面量或长度码的数量，包括 END_BLOCK 码 */

#define D_CODES   30
/* 距离码的数量 */

#define BL_CODES  19
/* 用于传输位长度的码的数量 */

#define HEAP_SIZE (2*L_CODES+1)
/* 最大堆大小 */

#define MAX_BITS 15
/* 所有码的位数不能超过 MAX_BITS 位 */

#define Buf_size 16
/* bi_buf 中位缓冲区的大小 */

#define INIT_STATE    42    /* zlib 头 -> BUSY_STATE */
#ifdef GZIP
#  define GZIP_STATE  57    /* gzip 头 -> BUSY_STATE | EXTRA_STATE */
#endif
#define EXTRA_STATE   69    /* gzip 额外块 -> NAME_STATE */
#define NAME_STATE    73    /* gzip 文件名 -> COMMENT_STATE */
#define COMMENT_STATE 91    /* gzip 注释 -> HCRC_STATE */
#define HCRC_STATE   103    /* gzip 头 CRC -> BUSY_STATE */
#define BUSY_STATE   113    /* deflate -> FINISH_STATE */
#define FINISH_STATE 666    /* 流完成 */
/* 流状态 */
/* 描述单个值及其编码字符串的数据结构。 */
typedef struct ct_data_s {
    union {
        ush  freq;       /* 频率计数 */
        ush  code;       /* 位字符串 */
    } fc;
    union {
        ush  dad;        /* 哈夫曼树中的父节点 */
        ush  len;        /* 位字符串的长度 */
    } dl;
} FAR ct_data;

#define Freq fc.freq
#define Code fc.code
#define Dad  dl.dad
#define Len  dl.len

typedef struct static_tree_desc_s  static_tree_desc;

typedef struct tree_desc_s {
    ct_data *dyn_tree;           /* 动态树 */
    int     max_code;            /* 非零频率的最大编码 */
    const static_tree_desc *stat_desc;  /* 对应的静态树 */
} FAR tree_desc;

typedef ush Pos;
typedef Pos FAR Posf;
typedef unsigned IPos;

/* Pos 是字符窗口中的索引。我们使用 short 而不是 int 来节省空间。IPos 仅用于参数传递。 */

typedef struct internal_state {
    z_streamp strm;      /* 指向该 zlib 流的指针 */
    int   status;        /* 如名称所示 */
    Bytef *pending_buf;  /* 输出仍在等待的缓冲区 */
    ulg   pending_buf_size; /* pending_buf 的大小 */
    Bytef *pending_out;  /* 下一个要输出到流中的待处理字节 */
    ulg   pending;       /* 待处理缓冲区中的字节数 */
    int   wrap;          /* 位 0 表示 zlib，位 1 表示 gzip */
    gz_headerp  gzhead;  /* 要写入的 gzip 头信息 */
    ulg   gzindex;       /* 在额外信息、名称或注释中的位置 */
    Byte  method;        /* 只能是 DEFLATED */
    int   last_flush;    /* 上一次调用 deflate 时的 flush 参数值 */

                /* deflate.c 中使用的： */

    uInt  w_size;        /* LZ77 窗口大小（默认为 32K） */
    uInt  w_bits;        /* log2(w_size)（8..16） */
    uInt  w_mask;        /* w_size - 1 */

    Bytef *window;
    /* 滑动窗口。输入字节被读取到窗口的后半部分，稍后移动到前半部分，以保持至少 wSize 字节的字典。通过这种组织方式，匹配限制在距离为 wSize-MAX_MATCH 字节的范围内，但这确保了 IO 总是以块大小的倍数进行。此外，它将窗口大小限制在 64K，这在 MSDOS 上非常有用。待办事项：使用用户输入缓冲区作为滑动窗口。 */

    ulg window_size;
    /* 窗口的实际大小：2*wSize，除非用户输入缓冲区直接用作滑动窗口。 */

    Posf *prev;
    /* 与相同哈希索引的旧字符串的链接。为了将此数组的大小限制在 64K，此链接仅维护最后 32K 个字符串。因此，此数组中的索引是窗口索引模 32K。 */

    Posf *head; /* 哈希链的头部或 NIL。 */

    uInt  ins_h;          /* 要插入的字符串的哈希索引 */
    uInt  hash_size;      /* 哈希表中的元素数量 */
    uInt  hash_bits;      /* log2(hash_size) */
    uInt  hash_mask;      /* hash_size-1 */

    uInt  hash_shift;
    /* 每次输入步骤时 ins_h 必须向右移动的位数。它必须满足：在 MIN_MATCH 步骤之后，最老的字节不再参与哈希键，即：hash_shift * MIN_MATCH >= hash_bits */

    long block_start;
    /* 当前输出块开始时的窗口位置。当窗口向后移动时，它变为负数。 */

    uInt match_length;           /* 最佳匹配的长度 */
    IPos prev_match;             /* 上一个匹配 */
    int match_available;         /* 如果存在上一个匹配，则设置为真 */
    uInt strstart;               /* 要插入的字符串的起始位置 */
    uInt match_start;            /* 匹配字符串的起始位置 */
    uInt lookahead;              /* 窗口中前方的有效字节数 */
    # 之前步骤中最佳匹配的长度。不大于此长度的匹配将被丢弃。这在惰性匹配评估中使用。
    uInt prev_length;

    # 为了加快压缩速度，哈希链不会搜索超过这个长度。更高的限制可以提高压缩比，但会降低速度。
    uInt max_chain_length;

    # 仅当当前匹配严格小于此值时，才尝试寻找更好的匹配。此机制仅用于压缩级别 >= 4。
    uInt max_lazy_match;
# 定义最大插入长度为最大懒惰匹配长度
# 只有在匹配长度不大于此长度时，才将新字符串插入哈希表中。这样可以节省时间，但会降低压缩率。
# max_insert_length 仅用于压缩级别 <= 3。

int level;    /* 压缩级别（1..9）*/
int strategy; /* 偏好或强制使用哈夫曼编码 */

uInt good_match;
/* 当前匹配长度大于此值时，使用更快的搜索 */

int nice_match; /* 当前匹配超过此值时停止搜索 */

/* trees.c 中使用：*/
/* 下面没有使用 ct_data typedef 来抑制编译器警告 */
struct ct_data_s dyn_ltree[HEAP_SIZE];   /* 文字和长度树 */
struct ct_data_s dyn_dtree[2*D_CODES+1]; /* 距离树 */
struct ct_data_s bl_tree[2*BL_CODES+1];  /* 位长度的哈夫曼树 */

struct tree_desc_s l_desc;               /* 文字树的描述 */
struct tree_desc_s d_desc;               /* 距离树的描述 */
struct tree_desc_s bl_desc;              /* 位长度树的描述 */

ush bl_count[MAX_BITS+1];
/* 最佳树中每个位长度的代码数量 */

int heap[2*L_CODES+1];      /* 用于构建哈夫曼树的堆 */
int heap_len;               /* 堆中元素的数量 */
int heap_max;               /* 频率最大的元素 */
/* 堆的子节点为 heap[n] 的元素为 heap[2*n] 和 heap[2*n+1]。heap[0] 未使用。
 * 相同的堆数组用于构建所有树。
 */

uch depth[2*L_CODES+1];
/* 用作相等频率树的决策树的每个子树的深度 */

uchf *sym_buf;        /* 用于距离和文字/长度的缓冲区 */

uInt  lit_bufsize;
    /* 匹配缓冲区的大小，用于存储字面量/长度。将 lit_bufsize 限制在 64K 有 4 个原因：
     *   - 频率可以用 16 位计数器来保存
     *   - 如果对第一个块的压缩不成功，所有输入数据仍然在窗口中，所以即使输入来自标准输入，我们仍然可以发出一个存储块。（如果 lit_bufsize 不大于 32K，对所有块都可以这样做。）
     *   - 如果对小于 64K 的文件的压缩不成功，我们甚至可以发出一个存储文件而不是存储块（节省 5 个字节）。这仅适用于 zip（不适用于 gzip 或 zlib）。
     *   - 创建新的哈夫曼树的频率不太可能快速适应输入数据统计的变化。（以一个二进制文件和一个高度可压缩的字符串表为例。）较小的缓冲区大小可以快速适应，但当然会有更频繁传输树的开销。
     *   - 我数不到 4 个以上的原因
     */

    uInt sym_next;      /* sym_buf 中的运行索引 */
    uInt sym_end;       /* 当 sym_next 达到此值时，符号表已满 */

    ulg opt_len;        /* 使用最佳树的当前块的位长度 */
    ulg static_len;     /* 使用静态树的当前块的位长度 */
    uInt matches;       /* 当前块中字符串匹配的数量 */
    uInt insert;        /* 窗口末尾剩余要插入的字节 */
#ifdef ZLIB_DEBUG
    ulg compressed_len; /* total bit length of compressed file mod 2^32 */
    ulg bits_sent;      /* bit length of compressed data sent mod 2^32 */
#endif

    ush bi_buf;
    /* 输出缓冲区。位从底部（最不重要的位）开始插入。 */
    int bi_valid;
    /* bi_buf 中有效位的数量。所有在最后一个有效位之上的位始终为零。 */

    ulg high_water;
    /* 窗口中已初始化字节的高水位偏移量 - 在此之上的字节被设置为零，以避免在最长匹配例程访问输入之外的字节时出现内存检查警告。然后更新为新的高水位标记。 */

} FAR deflate_state;

/* 在流上输出一个字节。
 * IN 断言：pending_buf 中有足够的空间。
 */
#define put_byte(s, c) {s->pending_buf[s->pending++] = (Bytef)(c);}


#define MIN_LOOKAHEAD (MAX_MATCH+MIN_MATCH+1)
/* 最小的前瞻量，除了在输入文件的末尾。
 * 有关 MIN_MATCH+1 的注释，请参见 deflate.c。
 */

#define MAX_DIST(s)  ((s)->w_size-MIN_LOOKAHEAD)
/* 为了简化代码，特别是在 16 位机器上，匹配距离被限制为 MAX_DIST，而不是 WSIZE。 */

#define WIN_INIT MAX_MATCH
/* 在窗口中数据结束后初始化的字节数，以避免在最长匹配例程访问输入之外的字节时出现内存检查错误 */

        /* 在 trees.c 中 */
void ZLIB_INTERNAL _tr_init OF((deflate_state *s));
int ZLIB_INTERNAL _tr_tally OF((deflate_state *s, unsigned dist, unsigned lc));
void ZLIB_INTERNAL _tr_flush_block OF((deflate_state *s, charf *buf,
                        ulg stored_len, int last));
void ZLIB_INTERNAL _tr_flush_bits OF((deflate_state *s));
void ZLIB_INTERNAL _tr_align OF((deflate_state *s));
void ZLIB_INTERNAL _tr_stored_block OF((deflate_state *s, charf *buf,
                        ulg stored_len, int last));
#define d_code(dist) \
   ((dist) < 256 ? _dist_code[dist] : _dist_code[256+((dist)>>7)])
/* 将距离映射为距离编码。dist是距离-1，不能有副作用。_dist_code[256]和_dist_code[257]从未被使用。 */

#ifndef ZLIB_DEBUG
/* 为了提高速度而内联的_tr_tally的版本： */

#if defined(GEN_TREES_H) || !defined(STDC)
  extern uch ZLIB_INTERNAL _length_code[];
  extern uch ZLIB_INTERNAL _dist_code[];
#else
  extern const uch ZLIB_INTERNAL _length_code[];
  extern const uch ZLIB_INTERNAL _dist_code[];
#endif

# define _tr_tally_lit(s, c, flush) \
  { uch cc = (c); \
    s->sym_buf[s->sym_next++] = 0; \
    s->sym_buf[s->sym_next++] = 0; \
    s->sym_buf[s->sym_next++] = cc; \
    s->dyn_ltree[cc].Freq++; \
    flush = (s->sym_next == s->sym_end); \
   }
# define _tr_tally_dist(s, distance, length, flush) \
  { uch len = (uch)(length); \
    ush dist = (ush)(distance); \
    s->sym_buf[s->sym_next++] = (uch)dist; \
    s->sym_buf[s->sym_next++] = (uch)(dist >> 8); \
    s->sym_buf[s->sym_next++] = len; \
    dist--; \
    s->dyn_ltree[_length_code[len]+LITERALS+1].Freq++; \
    s->dyn_dtree[d_code(dist)].Freq++; \
    flush = (s->sym_next == s->sym_end); \
  }
#else
# define _tr_tally_lit(s, c, flush) flush = _tr_tally(s, 0, c)
# define _tr_tally_dist(s, distance, length, flush) \
              flush = _tr_tally(s, distance, length)
#endif

#endif /* DEFLATE_H */
```