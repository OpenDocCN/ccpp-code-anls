# `nmap\libz\deflate.c`

```
/* deflate.c -- compress data using the deflation algorithm
 * Copyright (C) 1995-2022 Jean-loup Gailly and Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

/* @(#) $Id$ */

#include "deflate.h"

const char deflate_copyright[] =
   " deflate 1.2.13 Copyright 1995-2022 Jean-loup Gailly and Mark Adler ";
/*
  If you use the zlib library in a product, an acknowledgment is welcome
  in the documentation of your product. If for some reason you cannot
  include such an acknowledgment, I would appreciate that you keep this
  copyright string in the executable of your product.
 */

/* ===========================================================================
 *  Function prototypes.
 */
typedef enum {
    need_more,      /* block not completed, need more input or more output */
    block_done,     /* block flush performed */
    finish_started, /* finish started, need only more output at next deflate */
    finish_done     /* finish done, accept no more input or output */
} block_state;

typedef block_state (*compress_func) OF((deflate_state *s, int flush));
/* Compression function. Returns the block state after the call. */

local int deflateStateCheck      OF((z_streamp strm));
local void slide_hash     OF((deflate_state *s));
local void fill_window    OF((deflate_state *s));
local block_state deflate_stored OF((deflate_state *s, int flush));
local block_state deflate_fast   OF((deflate_state *s, int flush));
#ifndef FASTEST
local block_state deflate_slow   OF((deflate_state *s, int flush));
#endif
local block_state deflate_rle    OF((deflate_state *s, int flush));
local block_state deflate_huff   OF((deflate_state *s, int flush));
local void lm_init        OF((deflate_state *s));
local void putShortMSB    OF((deflate_state *s, uInt b));
local void flush_pending  OF((z_streamp strm));
local unsigned read_buf   OF((z_streamp strm, Bytef *buf, unsigned size));
local uInt longest_match  OF((deflate_state *s, IPos cur_match));
#ifdef ZLIB_DEBUG
local  void check_match OF((deflate_state *s, IPos start, IPos match,
                            int length));
#endif

#ifdef ZLIB_DEBUG 用于条件编译，如果定义了 ZLIB_DEBUG 则编译 check_match 函数，否则不编译。


/* ===========================================================================
 * Local data
 */

定义一个注释，说明下面是关于本地数据的定义。


#define NIL 0

定义一个宏，NIL 的值为 0。


#ifndef TOO_FAR
#  define TOO_FAR 4096
#endif

如果 TOO_FAR 未定义，则定义它的值为 4096。


/* Values for max_lazy_match, good_match and max_chain_length, depending on
 * the desired pack level (0..9). The values given below have been tuned to
 * exclude worst case performance for pathological files. Better values may be
 * found for specific files.
 */

对于所需的压缩级别（0..9），根据最大懒惰匹配、好匹配和最大链长度的值。下面给出的值已经调整，以排除对病态文件的最坏性能。对于特定文件，可能会找到更好的值。


typedef struct config_s {
   ush good_length; /* reduce lazy search above this match length */
   ush max_lazy;    /* do not perform lazy search above this match length */
   ush nice_length; /* quit search above this match length */
   ush max_chain;
   compress_func func;
} config;

定义了一个名为 config_s 的结构体，包含了 good_length、max_lazy、nice_length、max_chain 和 func 这几个成员变量。


#ifdef FASTEST
local const config configuration_table[2] = {
/*      good lazy nice chain */
/* 0 */ {0,    0,  0,    0, deflate_stored},  /* store only */
/* 1 */ {4,    4,  8,    4, deflate_fast}}; /* max speed, no lazy matches */
#else
local const config configuration_table[10] = {
/*      good lazy nice chain */
/* 0 */ {0,    0,  0,    0, deflate_stored},  /* store only */
/* 1 */ {4,    4,  8,    4, deflate_fast}, /* max speed, no lazy matches */
/* 2 */ {4,    5, 16,    8, deflate_fast},
/* 3 */ {4,    6, 32,   32, deflate_fast},

/* 4 */ {4,    4, 16,   16, deflate_slow},  /* lazy matches */
/* 5 */ {8,   16, 32,   32, deflate_slow},
/* 6 */ {8,   16, 128, 128, deflate_slow},
/* 7 */ {8,   32, 128, 256, deflate_slow},
/* 8 */ {32, 128, 258, 1024, deflate_slow},
/* 9 */ {32, 258, 258, 4096, deflate_slow}}; /* max compression */
#endif

根据定义的宏 FASTEST，选择不同的配置表。FASTEST 定义时，使用包含两个配置的配置表；否则，使用包含十个配置的配置表。


/* Note: the deflate() code requires max_lazy >= MIN_MATCH and max_chain >= 4
 * For deflate_fast() (levels <= 3) good is ignored and lazy has a different
 * meaning.
 */

注意：deflate() 代码要求 max_lazy >= MIN_MATCH 和 max_chain >= 4。对于 deflate_fast()（级别小于等于 3），good 被忽略，lazy 有不同的含义。
/* rank Z_BLOCK between Z_NO_FLUSH and Z_PARTIAL_FLUSH */
#define RANK(f) (((f) * 2) - ((f) > 4 ? 9 : 0))
// 定义一个宏，根据给定的参数 f 计算出一个值，用于在 Z_NO_FLUSH 和 Z_PARTIAL_FLUSH 之间进行排名

/* ===========================================================================
 * Update a hash value with the given input byte
 * IN  assertion: all calls to UPDATE_HASH are made with consecutive input
 *    characters, so that a running hash key can be computed from the previous
 *    key instead of complete recalculation each time.
 */
#define UPDATE_HASH(s,h,c) (h = (((h) << s->hash_shift) ^ (c)) & s->hash_mask)
// 定义一个宏，用给定的输入字节更新哈希值，根据断言，所有对 UPDATE_HASH 的调用都是使用连续的输入字符进行的，因此可以从前一个键计算出一个运行中的哈希键，而不是每次都完全重新计算。

/* ===========================================================================
 * Insert string str in the dictionary and set match_head to the previous head
 * of the hash chain (the most recent string with same hash key). Return
 * the previous length of the hash chain.
 * If this file is compiled with -DFASTEST, the compression level is forced
 * to 1, and no hash chains are maintained.
 * IN  assertion: all calls to INSERT_STRING are made with consecutive input
 *    characters and the first MIN_MATCH bytes of str are valid (except for
 *    the last MIN_MATCH-1 bytes of the input file).
 */
#ifdef FASTEST
#define INSERT_STRING(s, str, match_head) \
   (UPDATE_HASH(s, s->ins_h, s->window[(str) + (MIN_MATCH-1)]), \
    match_head = s->head[s->ins_h], \
    s->head[s->ins_h] = (Pos)(str))
#else
#define INSERT_STRING(s, str, match_head) \
   (UPDATE_HASH(s, s->ins_h, s->window[(str) + (MIN_MATCH-1)]), \
    match_head = s->prev[(str) & s->w_mask] = s->head[s->ins_h], \
    s->head[s->ins_h] = (Pos)(str))
#endif
// 定义一个宏，用于将字符串 str 插入字典中，并将 match_head 设置为哈希链的前一个头部（具有相同哈希键的最近字符串）。如果使用 -DFASTEST 编译此文件，则压缩级别被强制设置为 1，并且不维护哈希链。

/* ===========================================================================
 * Initialize the hash table (avoiding 64K overflow for 16 bit systems).
 * prev[] will be initialized on the fly.
 */
#define CLEAR_HASH(s) \
    do { \
        s->head[s->hash_size - 1] = NIL; \
        zmemzero((Bytef *)s->head, \
                 (unsigned)(s->hash_size - 1)*sizeof(*s->head)); \
    } while (0)
// 定义一个宏，用于初始化哈希表（避免 16 位系统的 64K 溢出）。prev[] 将动态初始化。
/* ===========================================================================
 * Slide the hash table when sliding the window down (could be avoided with 32
 * bit values at the expense of memory usage). We slide even when level == 0 to
 * keep the hash table consistent if we switch back to level > 0 later.
 */
# 滑动哈希表，当窗口向下滑动时（可以通过使用32位值来避免，但会增加内存使用）。即使 level == 0 时也要进行滑动，以保持哈希表的一致性，如果以后切换回 level > 0。
local void slide_hash(s)
    deflate_state *s;
{
    unsigned n, m;
    Posf *p;
    uInt wsize = s->w_size;

    n = s->hash_size;
    p = &s->head[n];
    do {
        m = *--p;
        *p = (Pos)(m >= wsize ? m - wsize : NIL);
    } while (--n);
    n = wsize;
#ifndef FASTEST
    p = &s->prev[n];
    do {
        m = *--p;
        *p = (Pos)(m >= wsize ? m - wsize : NIL);
        /* If n is not on any hash chain, prev[n] is garbage but
         * its value will never be used.
         */
    } while (--n);
#endif
}

/* ========================================================================= */
int ZEXPORT deflateInit_(strm, level, version, stream_size)
    z_streamp strm;
    int level;
    const char *version;
    int stream_size;
{
    return deflateInit2_(strm, level, Z_DEFLATED, MAX_WBITS, DEF_MEM_LEVEL,
                         Z_DEFAULT_STRATEGY, version, stream_size);
    /* To do: ignore strm->next_in if we use it as window */
}

/* ========================================================================= */
int ZEXPORT deflateInit2_(strm, level, method, windowBits, memLevel, strategy,
                  version, stream_size)
    z_streamp strm;
    int  level;
    int  method;
    int  windowBits;
    int  memLevel;
    int  strategy;
    const char *version;
    int stream_size;
{
    deflate_state *s;
    int wrap = 1;
    static const char my_version[] = ZLIB_VERSION;

    if (version == Z_NULL || version[0] != my_version[0] ||
        stream_size != sizeof(z_stream)) {
        return Z_VERSION_ERROR;
    }
    if (strm == Z_NULL) return Z_STREAM_ERROR;

    strm->msg = Z_NULL;
    if (strm->zalloc == (alloc_func)0) {
#ifdef Z_SOLO
        return Z_STREAM_ERROR;
#else
        // 如果不是 Z_SOLO 模式，则设置 strm->zalloc 为 zcalloc
        strm->zalloc = zcalloc;
        // 设置 strm->opaque 为 0
        strm->opaque = (voidpf)0;
#endif
    }
    // 如果 strm->zfree 为 0，则根据 Z_SOLO 模式返回 Z_STREAM_ERROR
    if (strm->zfree == (free_func)0)
#ifdef Z_SOLO
        return Z_STREAM_ERROR;
#else
        // 如果不是 Z_SOLO 模式，则设置 strm->zfree 为 zcfree
        strm->zfree = zcfree;
#endif

#ifdef FASTEST
    // 如果定义了 FASTEST，则如果 level 不等于 0，则将 level 设置为 1
    if (level != 0) level = 1;
#else
    // 如果没有定义 FASTEST，则如果 level 等于 Z_DEFAULT_COMPRESSION，则将 level 设置为 6
    if (level == Z_DEFAULT_COMPRESSION) level = 6;
#endif

    // 如果 windowBits 小于 0，则取消 zlib 包装
    if (windowBits < 0) { /* suppress zlib wrapper */
        wrap = 0;
        // 如果 windowBits 小于 -15，则返回 Z_STREAM_ERROR
        if (windowBits < -15)
            return Z_STREAM_ERROR;
        // 将 windowBits 取反
        windowBits = -windowBits;
    }
#ifdef GZIP
    // 如果定义了 GZIP，并且 windowBits 大于 15，则设置 wrap 为 2，同时将 windowBits 减去 16
    else if (windowBits > 15) {
        wrap = 2;       /* write gzip wrapper instead */
        windowBits -= 16;
    }
#endif
    // 如果 memLevel 小于 1 或者大于 MAX_MEM_LEVEL，或者 method 不等于 Z_DEFLATED，或者 windowBits 不在 8 到 15 之间，或者 level 不在 0 到 9 之间，或者 strategy 不在 0 到 Z_FIXED 之间，或者 (windowBits 等于 8 并且 wrap 不等于 1)，则返回 Z_STREAM_ERROR
    if (memLevel < 1 || memLevel > MAX_MEM_LEVEL || method != Z_DEFLATED ||
        windowBits < 8 || windowBits > 15 || level < 0 || level > 9 ||
        strategy < 0 || strategy > Z_FIXED || (windowBits == 8 && wrap != 1)) {
        return Z_STREAM_ERROR;
    }
    // 如果 windowBits 等于 8，则将 windowBits 设置为 9
    if (windowBits == 8) windowBits = 9;  /* until 256-byte window bug fixed */
    // 分配内存给 s，大小为 sizeof(deflate_state)
    s = (deflate_state *) ZALLOC(strm, 1, sizeof(deflate_state));
    // 如果分配内存失败，则返回 Z_MEM_ERROR
    if (s == Z_NULL) return Z_MEM_ERROR;
    // 设置 strm->state 为 s
    strm->state = (struct internal_state FAR *)s;
    // 设置 s->strm 为 strm
    s->strm = strm;
    // 设置 s->status 为 INIT_STATE
    s->status = INIT_STATE;     /* to pass state test in deflateReset() */

    // 设置 s->wrap 为 wrap
    s->wrap = wrap;
    // 设置 s->gzhead 为 Z_NULL
    s->gzhead = Z_NULL;
    // 设置 s->w_bits 为 windowBits
    s->w_bits = (uInt)windowBits;
    // 设置 s->w_size 为 2 的 s->w_bits 次方
    s->w_size = 1 << s->w_bits;
    // 设置 s->w_mask 为 s->w_size - 1
    s->w_mask = s->w_size - 1;

    // 设置 s->hash_bits 为 memLevel + 7
    s->hash_bits = (uInt)memLevel + 7;
    // 设置 s->hash_size 为 2 的 s->hash_bits 次方
    s->hash_size = 1 << s->hash_bits;
    // 设置 s->hash_mask 为 s->hash_size - 1
    s->hash_mask = s->hash_size - 1;
    // 设置 s->hash_shift 为 (s->hash_bits + MIN_MATCH-1) / MIN_MATCH
    s->hash_shift =  ((s->hash_bits + MIN_MATCH-1) / MIN_MATCH);

    // 分配内存给 s->window，大小为 s->w_size * 2*sizeof(Byte)
    s->window = (Bytef *) ZALLOC(strm, s->w_size, 2*sizeof(Byte));
    // 分配内存给 s->prev，大小为 s->w_size * sizeof(Pos)
    s->prev   = (Posf *)  ZALLOC(strm, s->w_size, sizeof(Pos));
    // 分配内存给 s->head，大小为 s->hash_size * sizeof(Pos)
    s->head   = (Posf *)  ZALLOC(strm, s->hash_size, sizeof(Pos));

    // 设置 s->high_water 为 0
    s->high_water = 0;      /* nothing written to s->window yet */

    // 设置 s->lit_bufsize 为 2 的 (memLevel + 6) 次方
    s->lit_bufsize = 1 << (memLevel + 6); /* 16K elements by default */
    # 分配内存给 pending_buf，大小为 s->lit_bufsize * 4
    s->pending_buf = (uchf *) ZALLOC(strm, s->lit_bufsize, 4);
    # 设置 pending_buf_size 为 s->lit_bufsize * 4
    s->pending_buf_size = (ulg)s->lit_bufsize * 4;

    # 检查是否有任何指针为空，如果有则设置状态为 FINISH_STATE，返回内存错误
    if (s->window == Z_NULL || s->prev == Z_NULL || s->head == Z_NULL ||
        s->pending_buf == Z_NULL) {
        s->status = FINISH_STATE;
        strm->msg = ERR_MSG(Z_MEM_ERROR);
        deflateEnd (strm);
        return Z_MEM_ERROR;
    }
    # 设置 sym_buf 为 pending_buf 的偏移量为 lit_bufsize
    s->sym_buf = s->pending_buf + s->lit_bufsize;
    # 设置 sym_end 为 lit_bufsize - 1 的三倍
    s->sym_end = (s->lit_bufsize - 1) * 3
    # 设置压缩等级为 level
    s->level = level;
    # 设置压缩策略为 strategy
    s->strategy = strategy;
    # 设置压缩方法为 method
    s->method = (Byte)method;

    # 返回 deflateReset 函数的结果
    return deflateReset(strm);
# 检查压缩流状态是否有效，如果有效返回0，否则返回1
local int deflateStateCheck(strm)
    z_streamp strm;
{
    deflate_state *s;
    # 如果传入的压缩流为空，或者分配函数或释放函数为空，则返回1
    if (strm == Z_NULL ||
        strm->zalloc == (alloc_func)0 || strm->zfree == (free_func)0)
        return 1;
    s = strm->state;
    # 如果状态为空，或者状态不是初始状态、GZIP状态、额外状态、名称状态、注释状态、HCRC状态、忙状态、完成状态，则返回1
    if (s == Z_NULL || s->strm != strm || (s->status != INIT_STATE &&
#ifdef GZIP
                                           s->status != GZIP_STATE &&
#endif
                                           s->status != EXTRA_STATE &&
                                           s->status != NAME_STATE &&
                                           s->status != COMMENT_STATE &&
                                           s->status != HCRC_STATE &&
                                           s->status != BUSY_STATE &&
                                           s->status != FINISH_STATE))
        return 1;
    return 0;
}

# 设置压缩流的字典
int ZEXPORT deflateSetDictionary(strm, dictionary, dictLength)
    z_streamp strm;
    const Bytef *dictionary;
    uInt  dictLength;
{
    deflate_state *s;
    uInt str, n;
    int wrap;
    unsigned avail;
    z_const unsigned char *next;

    # 如果压缩流状态无效或者字典为空，则返回流错误
    if (deflateStateCheck(strm) || dictionary == Z_NULL)
        return Z_STREAM_ERROR;
    s = strm->state;
    wrap = s->wrap;
    # 如果wrap为2，或者wrap为1且状态不是初始状态，或者前瞻不为空，则返回流错误
    if (wrap == 2 || (wrap == 1 && s->status != INIT_STATE) || s->lookahead)
        return Z_STREAM_ERROR;

    # 当使用zlib包装器时，为提供的字典计算Adler-32校验值
    if (wrap == 1)
        strm->adler = adler32(strm->adler, dictionary, dictLength);
    s->wrap = 0;                    # 避免在read_buf中计算Adler-32

    # 如果字典将填满窗口，则只需替换历史记录
    # 如果字典长度大于等于窗口大小
    if (dictLength >= s->w_size) {
        # 如果wrap为0，表示窗口已经为空，否则不做处理
        if (wrap == 0) {            /* already empty otherwise */
            # 清空哈希表
            CLEAR_HASH(s);
            # 窗口起始位置清零
            s->strstart = 0;
            # 块起始位置清零
            s->block_start = 0L;
            # 插入位置清零
            s->insert = 0;
        }
        # 将字典指针向后移动，以使用尾部内容
        dictionary += dictLength - s->w_size;  /* use the tail */
        # 字典长度设置为窗口大小
        dictLength = s->w_size;
    }

    # 将字典插入到窗口和哈希表中
    avail = strm->avail_in;
    next = strm->next_in;
    strm->avail_in = dictLength;
    strm->next_in = (z_const Bytef *)dictionary;
    fill_window(s);
    # 当前窗口中的可用字节数大于等于最小匹配长度时，执行循环
    while (s->lookahead >= MIN_MATCH) {
        str = s->strstart;
        n = s->lookahead - (MIN_MATCH-1);
        do {
            # 更新哈希表
            UPDATE_HASH(s, s->ins_h, s->window[str + MIN_MATCH-1]);
#ifndef FASTEST
            // 如果不是最快模式，将当前字符串的哈希值存入哈希表
            s->prev[str & s->w_mask] = s->head[s->ins_h];
#endif
            // 将当前字符串的哈希值存入哈希表
            s->head[s->ins_h] = (Pos)str;
            // 移动字符串指针
            str++;
        } while (--n);
        // 更新字符串起始位置和向前看字符数
        s->strstart = str;
        s->lookahead = MIN_MATCH-1;
        // 填充滑动窗口
        fill_window(s);
    }
    // 更新字符串起始位置
    s->strstart += s->lookahead;
    // 设置块的起始位置
    s->block_start = (long)s->strstart;
    // 设置插入位置和向前看字符数
    s->insert = s->lookahead;
    s->lookahead = 0;
    // 设置匹配长度和前一个匹配长度
    s->match_length = s->prev_length = MIN_MATCH-1;
    // 设置匹配是否可用
    s->match_available = 0;
    // 设置输入流的下一个位置和可用长度
    strm->next_in = next;
    strm->avail_in = avail;
    // 设置包装标志
    s->wrap = wrap;
    // 返回压缩成功
    return Z_OK;
}

/* ========================================================================= */
// 获取压缩字典
int ZEXPORT deflateGetDictionary(strm, dictionary, dictLength)
    z_streamp strm;
    Bytef *dictionary;
    uInt  *dictLength;
{
    deflate_state *s;
    uInt len;

    // 检查压缩状态
    if (deflateStateCheck(strm))
        return Z_STREAM_ERROR;
    // 获取压缩状态
    s = strm->state;
    len = s->strstart + s->lookahead;
    // 如果长度超过窗口大小，截断长度
    if (len > s->w_size)
        len = s->w_size;
    // 如果字典不为空且长度不为0，将窗口中的数据复制到字典中
    if (dictionary != Z_NULL && len)
        zmemcpy(dictionary, s->window + s->strstart + s->lookahead - len, len);
    // 如果字典长度不为空，将长度赋值给dictLength
    if (dictLength != Z_NULL)
        *dictLength = len;
    // 返回成功
    return Z_OK;
}

/* ========================================================================= */
// 重置压缩状态但保留数据
int ZEXPORT deflateResetKeep(strm)
    z_streamp strm;
{
    deflate_state *s;

    // 检查压缩状态
    if (deflateStateCheck(strm)) {
        return Z_STREAM_ERROR;
    }

    // 重置输入输出总数、消息、数据类型
    strm->total_in = strm->total_out = 0;
    strm->msg = Z_NULL; /* use zfree if we ever allocate msg dynamically */
    strm->data_type = Z_UNKNOWN;

    // 获取压缩状态
    s = (deflate_state *)strm->state;
    s->pending = 0;
    s->pending_out = s->pending_buf;

    // 如果包装标志为负数，取绝对值
    if (s->wrap < 0) {
        s->wrap = -s->wrap; /* was made negative by deflate(..., Z_FINISH); */
    }
    // 设置状态和校验值
    s->status =
#ifdef GZIP
        s->wrap == 2 ? GZIP_STATE :
#endif
        INIT_STATE;
    strm->adler =
#ifdef GZIP
        s->wrap == 2 ? crc32(0L, Z_NULL, 0) :
#endif
        adler32(0L, Z_NULL, 0);
    s->last_flush = -2;
}
    # 调用_tr_init函数，对参数s进行初始化
    _tr_init(s);
    
    # 返回Z_OK，表示函数执行成功
    return Z_OK;
/* ========================================================================= */
// 重置压缩流的状态
int ZEXPORT deflateReset(strm)
    z_streamp strm;
{
    int ret;

    // 调用deflateResetKeep函数重置压缩流的状态
    ret = deflateResetKeep(strm);
    // 如果重置成功，初始化压缩流的内部状态
    if (ret == Z_OK)
        lm_init(strm->state);
    return ret;
}

/* ========================================================================= */
// 设置压缩流的头部信息
int ZEXPORT deflateSetHeader(strm, head)
    z_streamp strm;
    gz_headerp head;
{
    // 检查压缩流的状态和包装类型是否符合要求
    if (deflateStateCheck(strm) || strm->state->wrap != 2)
        return Z_STREAM_ERROR;
    // 设置压缩流的头部信息
    strm->state->gzhead = head;
    return Z_OK;
}

/* ========================================================================= */
// 获取压缩流中待处理的数据和位数
int ZEXPORT deflatePending(strm, pending, bits)
    unsigned *pending;
    int *bits;
    z_streamp strm;
{
    // 检查压缩流的状态
    if (deflateStateCheck(strm)) return Z_STREAM_ERROR;
    // 如果传入的指针不为空，将压缩流中待处理的数据和位数赋值给相应的指针
    if (pending != Z_NULL)
        *pending = strm->state->pending;
    if (bits != Z_NULL)
        *bits = strm->state->bi_valid;
    return Z_OK;
}

/* ========================================================================= */
// 在压缩流中预留一些位
int ZEXPORT deflatePrime(strm, bits, value)
    z_streamp strm;
    int bits;
    int value;
{
    deflate_state *s;
    int put;

    // 检查压缩流的状态
    if (deflateStateCheck(strm)) return Z_STREAM_ERROR;
    s = strm->state;
    // 检查预留位数和待处理数据的关系，以及位数的合法性
    if (bits < 0 || bits > 16 ||
        s->sym_buf < s->pending_out + ((Buf_size + 7) >> 3))
        return Z_BUF_ERROR;
    // 循环预留位数
    do {
        put = Buf_size - s->bi_valid;
        if (put > bits)
            put = bits;
        s->bi_buf |= (ush)((value & ((1 << put) - 1)) << s->bi_valid);
        s->bi_valid += put;
        _tr_flush_bits(s);
        value >>= put;
        bits -= put;
    } while (bits);
    return Z_OK;
}

/* ========================================================================= */
// 设置压缩流的参数
int ZEXPORT deflateParams(strm, level, strategy)
    z_streamp strm;
    int level;
    int strategy;
{
    deflate_state *s;
    compress_func func;

    // 检查压缩流的状态
    if (deflateStateCheck(strm)) return Z_STREAM_ERROR;
    s = strm->state;

#ifdef FASTEST
    # 如果 level 不等于 0，则将 level 设置为 1；否则不做任何操作
    if (level != 0) level = 1;
#else
    # 如果压缩级别为默认值，则将级别设置为6
    if (level == Z_DEFAULT_COMPRESSION) level = 6;
#endif
    # 如果级别小于0或大于9，或者策略小于0或大于Z_FIXED，则返回流错误
    if (level < 0 || level > 9 || strategy < 0 || strategy > Z_FIXED) {
        return Z_STREAM_ERROR;
    }
    # 获取配置表中对应级别的函数
    func = configuration_table[s->level].func;

    # 如果策略不等于当前策略，或者函数不等于当前级别对应的函数，并且上次刷新不是-2
    if ((strategy != s->strategy || func != configuration_table[level].func) &&
        s->last_flush != -2) {
        /* Flush the last buffer: */
        # 刷新最后一个缓冲区
        int err = deflate(strm, Z_BLOCK);
        if (err == Z_STREAM_ERROR)
            return err;
        if (strm->avail_in || (s->strstart - s->block_start) + s->lookahead)
            return Z_BUF_ERROR;
    }
    # 如果当前级别不等于指定级别
    if (s->level != level) {
        # 如果当前级别为0且匹配数不为0
        if (s->level == 0 && s->matches != 0) {
            # 如果匹配数为1，则滑动哈希表
            if (s->matches == 1)
                slide_hash(s);
            else
                CLEAR_HASH(s);
            s->matches = 0;
        }
        # 设置当前级别为指定级别，并根据指定级别设置相关参数
        s->level = level;
        s->max_lazy_match   = configuration_table[level].max_lazy;
        s->good_match       = configuration_table[level].good_length;
        s->nice_match       = configuration_table[level].nice_length;
        s->max_chain_length = configuration_table[level].max_chain;
    }
    # 设置策略为指定策略
    s->strategy = strategy;
    return Z_OK;
}

/* ========================================================================= */
# 调整压缩参数
int ZEXPORT deflateTune(strm, good_length, max_lazy, nice_length, max_chain)
    z_streamp strm;
    int good_length;
    int max_lazy;
    int nice_length;
    int max_chain;
{
    deflate_state *s;

    # 检查压缩状态，如果有问题则返回流错误
    if (deflateStateCheck(strm)) return Z_STREAM_ERROR;
    s = strm->state;
    # 设置压缩参数
    s->good_match = (uInt)good_length;
    s->max_lazy_match = (uInt)max_lazy;
    s->nice_match = nice_length;
    s->max_chain_length = (uInt)max_chain;
    return Z_OK;
}
# 定义函数deflateBound，用于计算给定源数据长度的压缩后的上限值
uLong ZEXPORT deflateBound(strm, sourceLen)
    # 定义指向deflate状态的指针s
    z_streamp strm;
    # 定义源数据的长度sourceLen
    uLong sourceLen;
{
    # 定义指向deflate状态的指针s
    deflate_state *s;
    # 定义固定块的长度上限fixedlen，使用9位字面量和长度255（memLevel == 2，可能不使用存储块的最低级别）-- ~13%的开销加上一个小常数
    fixedlen = sourceLen + (sourceLen >> 3) + (sourceLen >> 8) + (sourceLen >> 9) + 4;

    # 定义存储块的长度上限storelen，使用长度127的存储块（memLevel == 1）-- ~4%的开销加上一个小常数
    # 计算存储长度，使用源长度的一定比例进行计算
    storelen = sourceLen + (sourceLen >> 5) + (sourceLen >> 7) + (sourceLen >> 11) + 7;

    # 如果无法获取参数，则返回较大的边界加上一个 zlib 包装
    if (deflateStateCheck(strm))
        return (fixedlen > storelen ? fixedlen : storelen) + 6;

    # 计算包装长度
    s = strm->state;
    switch (s->wrap) {
    case 0:                                 # 原始压缩
        wraplen = 0;
        break;
    case 1:                                 # zlib 包装
        wraplen = 6 + (s->strstart ? 4 : 0);
        break;
#ifdef GZIP
    #ifdef GZIP 指令的条件编译，如果定义了 GZIP，则执行下面的代码块
    case 2:                                 /* gzip wrapper */
        # 对于 case 2，表示使用 gzip 封装
        wraplen = 18;
        # 设置封装长度为 18
        if (s->gzhead != Z_NULL) {          /* user-supplied gzip header */
            # 如果存在用户提供的 gzip 头部信息
            Bytef *str;
            # 定义一个字节指针 str
            if (s->gzhead->extra != Z_NULL)
                # 如果 gzip 头部的额外信息不为空
                wraplen += 2 + s->gzhead->extra_len;
                # 封装长度增加 2 加上额外信息的长度
            str = s->gzhead->name;
            # 获取 gzip 头部的文件名
            if (str != Z_NULL)
                # 如果文件名不为空
                do {
                    wraplen++;
                    # 循环计算文件名的长度
                } while (*str++);
            str = s->gzhead->comment;
            # 获取 gzip 头部的注释信息
            if (str != Z_NULL)
                # 如果注释信息不为空
                do {
                    wraplen++;
                    # 循环计算注释信息的长度
                } while (*str++);
            if (s->gzhead->hcrc)
                # 如果存在头部 CRC 校验
                wraplen += 2;
                # 封装长度增加 2
        }
        break;
        # 结束 case 2
#endif
    # 结束条件编译
    default:                                /* for compiler happiness */
        # 默认情况下，为了编译器的愉悦
        wraplen = 6;
        # 设置封装长度为 6
    }

    /* if not default parameters, return one of the conservative bounds */
    # 如果不是默认参数，返回其中一个保守的边界值
    if (s->w_bits != 15 || s->hash_bits != 8 + 7)
        # 如果 w_bits 不等于 15 或者 hash_bits 不等于 8 + 7
        return (s->w_bits <= s->hash_bits ? fixedlen : storelen) + wraplen;
        # 返回固定长度或存储长度中较小的一个，并加上封装长度

    /* default settings: return tight bound for that case -- ~0.03% overhead
       plus a small constant */
    # 默认设置：返回该情况下的紧密边界值 -- 大约 0.03% 的额外开销加上一个小常数
    return sourceLen + (sourceLen >> 12) + (sourceLen >> 14) +
           (sourceLen >> 25) + 13 - 6 + wraplen;
           # 返回源长度加上一些位移操作和常数，再减去 6 并加上封装长度
}

/* =========================================================================
 * Put a short in the pending buffer. The 16-bit value is put in MSB order.
 * IN assertion: the stream state is correct and there is enough room in
 * pending_buf.
 */
# 将一个 short 值放入待处理缓冲区中，16 位的值按 MSB 顺序放置
# 输入断言：流状态正确，并且 pending_buf 中有足够的空间
local void putShortMSB(s, b)
    # 定义一个本地函数 putShortMSB，参数为 s 和 b
    deflate_state *s;
    # 定义 deflate_state 结构体指针 s
    uInt b;
    # 定义一个无符号整数 b
{
    put_byte(s, (Byte)(b >> 8));
    # 调用 put_byte 函数，将 b 的高 8 位写入缓冲区
    put_byte(s, (Byte)(b & 0xff));
    # 调用 put_byte 函数，将 b 的低 8 位写入缓冲区
}

/* =========================================================================
 * Flush as much pending output as possible. All deflate() output, except for
 * some deflate_stored() output, goes through this function so some
 * applications may wish to modify it to avoid allocating a large
 * strm->next_out buffer and copying into it. (See also read_buf()).
 */
# 尽可能刷新尽可能多的待处理输出。所有 deflate() 输出，除了一些 deflate_stored() 输出，都通过这个函数，因此一些应用程序可能希望修改它，以避免分配一个大的 strm->next_out 缓冲区并将数据复制到其中。（另请参阅 read_buf()）
local void flush_pending(strm)
    # 定义一个指向z_stream结构体的指针变量
    z_streamp strm;
{
    // 定义一个无符号整型变量 len
    unsigned len;
    // 获取压缩状态
    deflate_state *s = strm->state;

    // 刷新输出缓冲区中的位
    _tr_flush_bits(s);
    // 获取待处理的数据长度
    len = s->pending;
    // 如果待处理数据长度大于输出缓冲区可用空间，则将长度设置为输出缓冲区可用空间
    if (len > strm->avail_out) len = strm->avail_out;
    // 如果长度为 0，则直接返回
    if (len == 0) return;

    // 将 s->pending_out 中的数据复制到 strm->next_out 中
    zmemcpy(strm->next_out, s->pending_out, len);
    // 更新 strm->next_out 指针
    strm->next_out  += len;
    // 更新 s->pending_out 指针
    s->pending_out  += len;
    // 更新输出总字节数
    strm->total_out += len;
    // 更新输出缓冲区可用空间
    strm->avail_out -= len;
    // 更新待处理数据长度
    s->pending      -= len;
    // 如果待处理数据长度为 0，则重置 s->pending_out 指针
    if (s->pending == 0) {
        s->pending_out = s->pending_buf;
    }
}

/* ===========================================================================
 * 使用 s->pending_buf[beg..s->pending - 1] 的字节更新头部 CRC
 */
#define HCRC_UPDATE(beg) \
    do { \
        if (s->gzhead->hcrc && s->pending > (beg)) \
            strm->adler = crc32(strm->adler, s->pending_buf + (beg), \
                                s->pending - (beg)); \
    } while (0)

/* ========================================================================= */
// 压缩函数
int ZEXPORT deflate(strm, flush)
    z_streamp strm;
    int flush;
{
    int old_flush; /* 上一次调用 deflate 时的 flush 参数值 */
    deflate_state *s;

    // 检查压缩状态和 flush 参数的合法性
    if (deflateStateCheck(strm) || flush > Z_BLOCK || flush < 0) {
        return Z_STREAM_ERROR;
    }
    s = strm->state;

    // 检查输出和输入缓冲区的合法性
    if (strm->next_out == Z_NULL ||
        (strm->avail_in != 0 && strm->next_in == Z_NULL) ||
        (s->status == FINISH_STATE && flush != Z_FINISH)) {
        ERR_RETURN(strm, Z_STREAM_ERROR);
    }
    // 如果输出缓冲区已满，则返回缓冲区错误
    if (strm->avail_out == 0) ERR_RETURN(strm, Z_BUF_ERROR);

    // 保存上一次调用 deflate 时的 flush 参数值
    old_flush = s->last_flush;
    s->last_flush = flush;

    /* 尽可能刷新尚未处理的输出 */
}
    # 如果有未处理的数据
    if (s->pending != 0) {
        # 刷新未处理的数据
        flush_pending(strm);
        # 如果输出空间已满
        if (strm->avail_out == 0) {
            # 设置状态为需要更多输出空间，但不是错误情况
            s->last_flush = -1;
            return Z_OK;
        }

    # 确保有工作要做，并避免重复连续的刷新操作
    } else if (strm->avail_in == 0 && RANK(flush) <= RANK(old_flush) &&
               flush != Z_FINISH) {
        # 如果没有输入数据且不是 Z_FINISH 操作，则返回缓冲区错误
        ERR_RETURN(strm, Z_BUF_ERROR);
    }

    # 在第一个 FINISH 状态后，用户不能再提供更多输入
    if (s->status == FINISH_STATE && strm->avail_in != 0) {
        # 如果状态为 FINISH_STATE 且还有输入数据，则返回缓冲区错误
        ERR_RETURN(strm, Z_BUF_ERROR);
    }

    # 写入头部信息
    if (s->status == INIT_STATE && s->wrap == 0)
        # 如果状态为 INIT_STATE 且 wrap 为 0，则将状态设置为 BUSY_STATE
        s->status = BUSY_STATE;
    # 如果状态为初始状态
    if (s->status == INIT_STATE) {
        # 计算 zlib 头部信息
        uInt header = (Z_DEFLATED + ((s->w_bits - 8) << 4)) << 8;
        uInt level_flags;

        # 根据策略和压缩级别设置级别标志
        if (s->strategy >= Z_HUFFMAN_ONLY || s->level < 2)
            level_flags = 0;
        else if (s->level < 6)
            level_flags = 1;
        else if (s->level == 6)
            level_flags = 2;
        else
            level_flags = 3;
        header |= (level_flags << 6);
        # 如果存在预设字典，设置预设字典标志
        if (s->strstart != 0) header |= PRESET_DICT;
        header += 31 - (header % 31);

        # 将头部信息写入输出流
        putShortMSB(s, header);

        # 保存预设字典的 adler32 校验值
        if (s->strstart != 0) {
            putShortMSB(s, (uInt)(strm->adler >> 16));
            putShortMSB(s, (uInt)(strm->adler & 0xffff));
        }
        # 重置 adler32 校验值
        strm->adler = adler32(0L, Z_NULL, 0);
        # 设置状态为忙碌状态
        s->status = BUSY_STATE;

        # 压缩必须从一个空的待处理缓冲区开始
        flush_pending(strm);
        # 如果存在待处理数据，返回 Z_OK
        if (s->pending != 0) {
            s->last_flush = -1;
            return Z_OK;
        }
    }
#ifdef GZIP
    # 如果状态为 GZIP_STATE
    if (s->status == GZIP_STATE) {
        /* gzip header */
        # 计算并设置 Adler-32 校验值
        strm->adler = crc32(0L, Z_NULL, 0);
        # 写入 gzip 头部信息
        put_byte(s, 31);
        put_byte(s, 139);
        put_byte(s, 8);
        # 如果 gzhead 为空
        if (s->gzhead == Z_NULL) {
            # 写入额外的标志位
            put_byte(s, 0);
            put_byte(s, 0);
            put_byte(s, 0);
            put_byte(s, 0);
            put_byte(s, 0);
            put_byte(s, s->level == 9 ? 2 :
                     (s->strategy >= Z_HUFFMAN_ONLY || s->level < 2 ?
                      4 : 0));
            put_byte(s, OS_CODE);
            # 设置状态为 BUSY_STATE
            s->status = BUSY_STATE;

            /* Compression must start with an empty pending buffer */
            # 刷新 pending 缓冲区
            flush_pending(strm);
            # 如果 pending 不为 0，则返回 Z_OK
            if (s->pending != 0) {
                s->last_flush = -1;
                return Z_OK;
            }
        }
        else {
            # 写入额外的标志位
            put_byte(s, (s->gzhead->text ? 1 : 0) +
                     (s->gzhead->hcrc ? 2 : 0) +
                     (s->gzhead->extra == Z_NULL ? 0 : 4) +
                     (s->gzhead->name == Z_NULL ? 0 : 8) +
                     (s->gzhead->comment == Z_NULL ? 0 : 16)
                     );
            put_byte(s, (Byte)(s->gzhead->time & 0xff));
            put_byte(s, (Byte)((s->gzhead->time >> 8) & 0xff));
            put_byte(s, (Byte)((s->gzhead->time >> 16) & 0xff));
            put_byte(s, (Byte)((s->gzhead->time >> 24) & 0xff));
            put_byte(s, s->level == 9 ? 2 :
                     (s->strategy >= Z_HUFFMAN_ONLY || s->level < 2 ?
                      4 : 0));
            put_byte(s, s->gzhead->os & 0xff);
            # 如果额外数据不为空
            if (s->gzhead->extra != Z_NULL) {
                put_byte(s, s->gzhead->extra_len & 0xff);
                put_byte(s, (s->gzhead->extra_len >> 8) & 0xff);
            }
            # 如果启用了 hcrc
            if (s->gzhead->hcrc)
                # 计算并设置 Adler-32 校验值
                strm->adler = crc32(strm->adler, s->pending_buf,
                                    s->pending);
            # 重置 gzindex，设置状态为 EXTRA_STATE
            s->gzindex = 0;
            s->status = EXTRA_STATE;
        }
    }
    # 如果状态为 EXTRA_STATE
    if (s->status == EXTRA_STATE) {
        # 如果 gzhead 的 extra 不为空
        if (s->gzhead->extra != Z_NULL) {
            # 记录需要更新 CRC 的起始位置
            ulg beg = s->pending;   /* start of bytes to update crc */
            # 计算剩余需要处理的额外数据长度
            uInt left = (s->gzhead->extra_len & 0xffff) - s->gzindex;
            # 如果待处理数据加上剩余数据长度大于缓冲区大小
            while (s->pending + left > s->pending_buf_size) {
                # 计算需要拷贝的数据长度
                uInt copy = s->pending_buf_size - s->pending;
                # 将额外数据拷贝到待处理数据缓冲区中
                zmemcpy(s->pending_buf + s->pending,
                        s->gzhead->extra + s->gzindex, copy);
                # 更新待处理数据位置
                s->pending = s->pending_buf_size;
                # 更新 CRC
                HCRC_UPDATE(beg);
                # 更新额外数据索引
                s->gzindex += copy;
                # 刷新待处理数据
                flush_pending(strm);
                # 如果待处理数据不为空
                if (s->pending != 0) {
                    # 重置最后刷新状态
                    s->last_flush = -1;
                    # 返回处理成功状态
                    return Z_OK;
                }
                # 重置起始位置
                beg = 0;
                # 更新剩余数据长度
                left -= copy;
            }
            # 将剩余额外数据拷贝到待处理数据缓冲区中
            zmemcpy(s->pending_buf + s->pending,
                    s->gzhead->extra + s->gzindex, left);
            # 更新待处理数据位置
            s->pending += left;
            # 更新 CRC
            HCRC_UPDATE(beg);
            # 重置额外数据索引
            s->gzindex = 0;
        }
        # 更新状态为 NAME_STATE
        s->status = NAME_STATE;
    }
    # 如果状态为 NAME_STATE
    if (s->status == NAME_STATE) {
        # 如果 gzhead 的 name 不为空
        if (s->gzhead->name != Z_NULL) {
            # 记录需要更新 CRC 的起始位置
            ulg beg = s->pending;   /* start of bytes to update crc */
            # 定义变量 val
            int val;
            # 循环处理文件名数据
            do {
                # 如果待处理数据已满
                if (s->pending == s->pending_buf_size) {
                    # 更新 CRC
                    HCRC_UPDATE(beg);
                    # 刷新待处理数据
                    flush_pending(strm);
                    # 如果待处理数据不为空
                    if (s->pending != 0) {
                        # 重置最后刷新状态
                        s->last_flush = -1;
                        # 返回处理成功状态
                        return Z_OK;
                    }
                    # 重置起始位置
                    beg = 0;
                }
                # 读取文件名数据并写入待处理数据缓冲区
                val = s->gzhead->name[s->gzindex++];
                put_byte(s, val);
            } while (val != 0);
            # 更新 CRC
            HCRC_UPDATE(beg);
            # 重置文件名数据索引
            s->gzindex = 0;
        }
        # 更新状态为 COMMENT_STATE
        s->status = COMMENT_STATE;
    }
    # 如果状态为COMMENT_STATE，则执行以下操作
    if (s->status == COMMENT_STATE) {
        # 如果存在注释内容
        if (s->gzhead->comment != Z_NULL) {
            # 记录需要更新 CRC 的起始位置
            ulg beg = s->pending;   /* start of bytes to update crc */
            int val;
            # 循环读取注释内容并更新 CRC
            do {
                # 如果待处理的数据大小等于缓冲区大小，则更新 CRC 并刷新缓冲区
                if (s->pending == s->pending_buf_size) {
                    HCRC_UPDATE(beg);
                    flush_pending(strm);
                    # 如果刷新后仍有数据待处理，则重置最后刷新状态并返回 Z_OK
                    if (s->pending != 0) {
                        s->last_flush = -1;
                        return Z_OK;
                    }
                    beg = 0;
                }
                # 读取注释内容并写入到缓冲区
                val = s->gzhead->comment[s->gzindex++];
                put_byte(s, val);
            } while (val != 0);  # 直到读取到注释结束符为止
            HCRC_UPDATE(beg);  # 更新 CRC
        }
        s->status = HCRC_STATE;  # 将状态更新为 HCRC_STATE
    }
    # 如果状态为 HCRC_STATE，则执行以下操作
    if (s->status == HCRC_STATE) {
        # 如果需要校验 CRC
        if (s->gzhead->hcrc) {
            # 如果待处理数据大小加上 2 大于缓冲区大小，则刷新缓冲区
            if (s->pending + 2 > s->pending_buf_size) {
                flush_pending(strm);
                # 如果刷新后仍有数据待处理，则重置最后刷新状态并返回 Z_OK
                if (s->pending != 0) {
                    s->last_flush = -1;
                    return Z_OK;
                }
            }
            # 将 Adler-32 校验值的低 8 位和高 8 位写入缓冲区
            put_byte(s, (Byte)(strm->adler & 0xff));
            put_byte(s, (Byte)((strm->adler >> 8) & 0xff));
            strm->adler = crc32(0L, Z_NULL, 0);  # 重置 Adler-32 校验值
        }
        s->status = BUSY_STATE;  # 将状态更新为 BUSY_STATE

        # 压缩必须从空的待处理缓冲区开始
        flush_pending(strm);
        # 如果刷新后仍有数据待处理，则重置最后刷新状态并返回 Z_OK
        if (s->pending != 0) {
            s->last_flush = -1;
            return Z_OK;
        }
    }
#endif
    /* 结束当前的代码块或开始新的代码块 */
    }

    if (flush != Z_FINISH) return Z_OK;
    if (s->wrap <= 0) return Z_STREAM_END;

    /* 写入压缩数据的尾部 */
#ifdef GZIP
    if (s->wrap == 2) {
        put_byte(s, (Byte)(strm->adler & 0xff));
        put_byte(s, (Byte)((strm->adler >> 8) & 0xff));
        put_byte(s, (Byte)((strm->adler >> 16) & 0xff));
        put_byte(s, (Byte)((strm->adler >> 24) & 0xff));
        put_byte(s, (Byte)(strm->total_in & 0xff));
        put_byte(s, (Byte)((strm->total_in >> 8) & 0xff));
        put_byte(s, (Byte)((strm->total_in >> 16) & 0xff));
        put_byte(s, (Byte)((strm->total_in >> 24) & 0xff));
    }
    else
#endif
    {
        putShortMSB(s, (uInt)(strm->adler >> 16));
        putShortMSB(s, (uInt)(strm->adler & 0xffff));
    }
    flush_pending(strm);
    /* 如果 avail_out 为零，应用程序将再次调用 deflate 来刷新剩余的数据 */
    if (s->wrap > 0) s->wrap = -s->wrap; /* 只写入尾部一次！ */
    return s->pending != 0 ? Z_OK : Z_STREAM_END;
}

/* ========================================================================= */
int ZEXPORT deflateEnd(strm)
    z_streamp strm;
{
    int status;

    if (deflateStateCheck(strm)) return Z_STREAM_ERROR;

    status = strm->state->status;

    /* 以分配的相反顺序释放内存： */
    TRY_FREE(strm, strm->state->pending_buf);
    TRY_FREE(strm, strm->state->head);
    TRY_FREE(strm, strm->state->prev);
    TRY_FREE(strm, strm->state->window);

    ZFREE(strm, strm->state);
    strm->state = Z_NULL;

    return status == BUSY_STATE ? Z_DATA_ERROR : Z_OK;
}

/* =========================================================================
 * 将源状态复制到目标状态。
 * 为了简化源代码，不支持 16 位 MSDOS（无论如何内存不足以复制压缩状态）。
 */
int ZEXPORT deflateCopy(dest, source)
    z_streamp dest;
    # 声明一个指向z_stream结构体的指针变量source
    z_streamp source;
{
#ifdef MAXSEG_64K
    // 如果定义了 MAXSEG_64K，则返回流错误
    return Z_STREAM_ERROR;
#else
    // 定义变量
    deflate_state *ds;
    deflate_state *ss;

    // 检查源和目标是否有效
    if (deflateStateCheck(source) || dest == Z_NULL) {
        // 如果无效则返回流错误
        return Z_STREAM_ERROR;
    }

    // 获取源的状态
    ss = source->state;

    // 将源的内容拷贝到目标
    zmemcpy((voidpf)dest, (voidpf)source, sizeof(z_stream));

    // 分配目标的状态
    ds = (deflate_state *) ZALLOC(dest, 1, sizeof(deflate_state));
    if (ds == Z_NULL) return Z_MEM_ERROR;
    dest->state = (struct internal_state FAR *) ds;
    // 将源状态的内容拷贝到目标状态
    zmemcpy((voidpf)ds, (voidpf)ss, sizeof(deflate_state));
    ds->strm = dest;

    // 分配内存空间
    ds->window = (Bytef *) ZALLOC(dest, ds->w_size, 2*sizeof(Byte));
    ds->prev   = (Posf *)  ZALLOC(dest, ds->w_size, sizeof(Pos));
    ds->head   = (Posf *)  ZALLOC(dest, ds->hash_size, sizeof(Pos));
    ds->pending_buf = (uchf *) ZALLOC(dest, ds->lit_bufsize, 4);

    // 检查内存分配是否成功
    if (ds->window == Z_NULL || ds->prev == Z_NULL || ds->head == Z_NULL ||
        ds->pending_buf == Z_NULL) {
        // 如果失败则结束压缩过程并返回内存错误
        deflateEnd (dest);
        return Z_MEM_ERROR;
    }
    /* following zmemcpy do not work for 16-bit MSDOS */
    // 将源状态的内容拷贝到目标状态
    zmemcpy(ds->window, ss->window, ds->w_size * 2 * sizeof(Byte));
    zmemcpy((voidpf)ds->prev, (voidpf)ss->prev, ds->w_size * sizeof(Pos));
    zmemcpy((voidpf)ds->head, (voidpf)ss->head, ds->hash_size * sizeof(Pos));
    zmemcpy(ds->pending_buf, ss->pending_buf, (uInt)ds->pending_buf_size);

    // 更新指针位置
    ds->pending_out = ds->pending_buf + (ss->pending_out - ss->pending_buf);
    ds->sym_buf = ds->pending_buf + ds->lit_bufsize;

    // 设置动态树
    ds->l_desc.dyn_tree = ds->dyn_ltree;
    ds->d_desc.dyn_tree = ds->dyn_dtree;
    ds->bl_desc.dyn_tree = ds->bl_tree;

    // 返回压缩成功
    return Z_OK;
#endif /* MAXSEG_64K */
}
# 从当前输入流中读取新的缓冲区，更新 adler32 和读取的总字节数。所有 deflate() 输入都经过这个函数，因此一些应用程序可能希望修改它，以避免分配一个大的 strm->next_in 缓冲区并从中复制数据。（另请参阅 flush_pending()）。
local unsigned read_buf(strm, buf, size)
    z_streamp strm;
    Bytef *buf;
    unsigned size;
{
    unsigned len = strm->avail_in;

    if (len > size) len = size;
    if (len == 0) return 0;

    strm->avail_in  -= len;

    zmemcpy(buf, strm->next_in, len);
    if (strm->state->wrap == 1) {
        strm->adler = adler32(strm->adler, buf, len);
    }
#ifdef GZIP
    else if (strm->state->wrap == 2) {
        strm->adler = crc32(strm->adler, buf, len);
    }
#endif
    strm->next_in  += len;
    strm->total_in += len;

    return len;
}

# 为新的 zlib 流初始化“最长匹配”例程
local void lm_init(s)
    deflate_state *s;
{
    s->window_size = (ulg)2L*s->w_size;

    CLEAR_HASH(s);

    # 设置默认的配置参数：
    s->max_lazy_match   = configuration_table[s->level].max_lazy;
    s->good_match       = configuration_table[s->level].good_length;
    s->nice_match       = configuration_table[s->level].nice_length;
    s->max_chain_length = configuration_table[s->level].max_chain;

    s->strstart = 0;
    s->block_start = 0L;
    s->lookahead = 0;
    s->insert = 0;
    s->match_length = s->prev_length = MIN_MATCH-1;
    s->match_available = 0;
    s->ins_h = 0;
}

# 如果不是最快模式，则执行以下代码
#ifndef FASTEST
# 设置 match_start 为从给定字符串开始的最长匹配，并返回其长度。
# 长度小于或等于 prev_length 的匹配将被丢弃，此时结果等于 prev_length，match_start 是垃圾。
# 输入断言：cur_match 是当前字符串（strstart）的哈希链的头部，其距离 <= MAX_DIST，prev_length >= 1
# 输出断言：匹配长度不大于 s->lookahead。
local uInt longest_match(s, cur_match)
    deflate_state *s;
    IPos cur_match;                             /* current match */
{
    unsigned chain_length = s->max_chain_length;  /* 最大哈希链长度 */
    register Bytef *scan = s->window + s->strstart;  /* 当前字符串 */
    register Bytef *match;                      /* 匹配的字符串 */
    register int len;                           /* 当前匹配的长度 */
    int best_len = (int)s->prev_length;         /* 到目前为止最佳匹配长度 */
    int nice_match = s->nice_match;             /* 如果匹配足够长则停止 */
    IPos limit = s->strstart > (IPos)MAX_DIST(s) ?
        s->strstart - (IPos)MAX_DIST(s) : NIL;  /* 当 cur_match 变成 <= limit 时停止。为了简化代码，我们防止与窗口索引 0 的字符串匹配。 */
    Posf *prev = s->prev;                       /* 前一个匹配的位置 */
    uInt wmask = s->w_mask;                     /* 窗口掩码 */

#ifdef UNALIGNED_OK
    /* 比较两个字节。注意：这并不总是有益的。尝试使用和不使用 -DUNALIGNED_OK 来检查。 */
    register Bytef *strend = s->window + s->strstart + MAX_MATCH - 1;
    register ush scan_start = *(ushf*)scan;
    register ush scan_end   = *(ushf*)(scan + best_len - 1);
#else
    register Bytef *strend = s->window + s->strstart + MAX_MATCH;
    register Byte scan_end1  = scan[best_len - 1];
    register Byte scan_end   = scan[best_len];
#endif
    /* 代码针对 HASH_BITS >= 8 和 MAX_MATCH-2 是 16 的倍数进行了优化。
     * 如果有必要，可以轻松摆脱这种优化。
     */
    Assert(s->hash_bits >= 8 && MAX_MATCH == 258, "Code too clever");

    /* 如果已经有一个很好的匹配，不要浪费太多时间： */
    if (s->prev_length >= s->good_match) {
        chain_length >>= 2;
    }
    /* 不要寻找超出输入末尾的匹配。这是必要的，以使压缩变得确定性。 */
    if ((uInt)nice_match > s->lookahead) nice_match = (int)s->lookahead;

    Assert((ulg)s->strstart <= s->window_size - MIN_LOOKAHEAD,
           "need lookahead");

    do {
        Assert(cur_match < s->strstart, "no future");
        match = s->window + cur_match;

        /* 如果匹配长度不能增加，或者匹配长度小于 2，则跳过下一个匹配。
         * 请注意，由于性能原因，对于不足的前瞻，下面的检查只偶尔发生。
         * 因此将访问未初始化的内存，并且将进行取决于这些值的条件跳转。
         * 但是匹配的长度限制为前瞻，因此 deflate 的输出不受未初始化值的影响。
         */
#if (defined(UNALIGNED_OK) && MAX_MATCH == 258)
        /* 如果定义了 UNALIGNED_OK 并且 MAX_MATCH 等于 258，则执行以下代码块 */
        /* This code assumes sizeof(unsigned short) == 2. Do not use
         * UNALIGNED_OK if your compiler uses a different size.
         */
        /* 这段代码假设 sizeof(unsigned short) == 2。如果你的编译器使用不同的大小，请不要使用 UNALIGNED_OK */
        if (*(ushf*)(match + best_len - 1) != scan_end ||
            *(ushf*)match != scan_start) continue;
        /* 如果 match + best_len - 1 处的值不等于 scan_end，或者 match 处的值不等于 scan_start，则继续下一次循环 */

        /* It is not necessary to compare scan[2] and match[2] since they are
         * always equal when the other bytes match, given that the hash keys
         * are equal and that HASH_BITS >= 8. Compare 2 bytes at a time at
         * strstart + 3, + 5, up to strstart + 257. We check for insufficient
         * lookahead only every 4th comparison; the 128th check will be made
         * at strstart + 257. If MAX_MATCH-2 is not a multiple of 8, it is
         * necessary to put more guard bytes at the end of the window, or
         * to check more often for insufficient lookahead.
         */
        /* 不需要比较 scan[2] 和 match[2]，因为当其他字节匹配时，它们总是相等，假设哈希键相等且 HASH_BITS >= 8。每次比较两个字节，从 strstart + 3，+ 5，一直到 strstart + 257。我们只在每4次比较时检查不足的前瞻；第128次检查将在 strstart + 257 处进行。如果 MAX_MATCH-2 不是8的倍数，则需要在窗口末尾放置更多的保护字节，或者更频繁地检查不足的前瞻。 */
        Assert(scan[2] == match[2], "scan[2]?");
        scan++, match++;
        do {
        } while (*(ushf*)(scan += 2) == *(ushf*)(match += 2) &&
                 *(ushf*)(scan += 2) == *(ushf*)(match += 2) &&
                 *(ushf*)(scan += 2) == *(ushf*)(match += 2) &&
                 *(ushf*)(scan += 2) == *(ushf*)(match += 2) &&
                 scan < strend);
        /* The funny "do {}" generates better code on most compilers */
        /* 有趣的 "do {}" 在大多数编译器上生成更好的代码 */

        /* Here, scan <= window + strstart + 257 */
        /* 在这里，scan <= window + strstart + 257 */
        Assert(scan <= s->window + (unsigned)(s->window_size - 1),
               "wild scan");
        /* 断言，确保 scan 不会超出窗口范围 */

        if (*scan == *match) scan++;
        /* 如果 *scan 等于 *match，则将 scan 向前移动一位 */

        len = (MAX_MATCH - 1) - (int)(strend - scan);
        /* 计算匹配长度 */

        scan = strend - (MAX_MATCH-1);
        /* 重新定位 scan 的位置 */
#else /* UNALIGNED_OK */

        if (match[best_len]     != scan_end  ||
            match[best_len - 1] != scan_end1 ||
            *match              != *scan     ||
            *++match            != scan[1])      continue;

        /* The check at best_len - 1 can be removed because it will be made
         * again later. (This heuristic is not always a win.)
         * It is not necessary to compare scan[2] and match[2] since they
         * are always equal when the other bytes match, given that
         * the hash keys are equal and that HASH_BITS >= 8.
         */
        scan += 2, match++;
        Assert(*scan == *match, "match[2]?");

        /* We check for insufficient lookahead only every 8th comparison;
         * the 256th check will be made at strstart + 258.
         */
        do {
        } while (*++scan == *++match && *++scan == *++match &&
                 *++scan == *++match && *++scan == *++match &&
                 *++scan == *++match && *++scan == *++match &&
                 *++scan == *++match && *++scan == *++match &&
                 scan < strend);

        Assert(scan <= s->window + (unsigned)(s->window_size - 1),
               "wild scan");

        len = MAX_MATCH - (int)(strend - scan);
        scan = strend - MAX_MATCH;

#endif /* UNALIGNED_OK */

        if (len > best_len) {
            s->match_start = cur_match;
            best_len = len;
            if (len >= nice_match) break;
#ifdef UNALIGNED_OK
            scan_end = *(ushf*)(scan + best_len - 1);
#else
            scan_end1  = scan[best_len - 1];
            scan_end   = scan[best_len];
#endif
        }
    } while ((cur_match = prev[cur_match & wmask]) > limit
             && --chain_length != 0);

    if ((uInt)best_len <= s->lookahead) return (uInt)best_len;
    return s->lookahead;
}

#else /* FASTEST */

/* ---------------------------------------------------------------------------
 * Optimized version for FASTEST only
 */
# 定义函数，用于查找最长匹配
local uInt longest_match(s, cur_match)
    deflate_state *s;
    IPos cur_match;                             /* current match */

{
    register Bytef *scan = s->window + s->strstart; /* current string */  # 定义指向当前字符串的指针
    register Bytef *match;                       /* matched string */  # 定义指向匹配字符串的指针
    register int len;                           /* length of current match */  # 定义当前匹配的长度
    register Bytef *strend = s->window + s->strstart + MAX_MATCH;  # 定义指向当前字符串末尾的指针

    /* The code is optimized for HASH_BITS >= 8 and MAX_MATCH-2 multiple of 16.
     * It is easy to get rid of this optimization if necessary.
     */
    Assert(s->hash_bits >= 8 && MAX_MATCH == 258, "Code too clever");  # 断言，确保哈希位数大于等于8且最大匹配长度为258

    Assert((ulg)s->strstart <= s->window_size - MIN_LOOKAHEAD, "need lookahead");  # 断言，确保当前字符串的起始位置小于等于窗口大小减去最小预读长度

    Assert(cur_match < s->strstart, "no future");  # 断言，确保当前匹配位置小于当前字符串的起始位置

    match = s->window + cur_match;  # 将匹配位置的指针指向当前匹配位置

    /* Return failure if the match length is less than 2:
     */
    if (match[0] != scan[0] || match[1] != scan[1]) return MIN_MATCH-1;  # 如果匹配长度小于2，则返回匹配失败

    /* The check at best_len - 1 can be removed because it will be made
     * again later. (This heuristic is not always a win.)
     * It is not necessary to compare scan[2] and match[2] since they
     * are always equal when the other bytes match, given that
     * the hash keys are equal and that HASH_BITS >= 8.
     */
    scan += 2, match += 2;  # 移动指针到下一个位置
    Assert(*scan == *match, "match[2]?");  # 断言，确保当前位置的字符匹配

    /* We check for insufficient lookahead only every 8th comparison;
     * the 256th check will be made at strstart + 258.
     */
    do {
    } while (*++scan == *++match && *++scan == *++match &&  # 循环，比较每8个字符是否匹配
             *++scan == *++match && *++scan == *++match &&
             *++scan == *++match && *++scan == *++match &&
             *++scan == *++match && *++scan == *++match &&
             scan < strend);  # 循环条件，确保指针未超出当前字符串末尾

    Assert(scan <= s->window + (unsigned)(s->window_size - 1), "wild scan");  # 断言，确保指针未超出窗口范围

    len = MAX_MATCH - (int)(strend - scan);  # 计算匹配长度

    if (len < MIN_MATCH) return MIN_MATCH - 1;  # 如果匹配长度小于最小匹配长度，则返回匹配失败

    s->match_start = cur_match;  # 设置匹配起始位置
    # 如果len小于等于s->lookahead，则返回len，否则返回s->lookahead
    return (uInt)len <= s->lookahead ? (uInt)len : s->lookahead;
#ifdef ZLIB_DEBUG

#define EQUAL 0
/* result of memcmp for equal strings */

/* ===========================================================================
 * Check that the match at match_start is indeed a match.
 */
local void check_match(s, start, match, length)
    deflate_state *s;
    IPos start, match;
    int length;
{
    /* check that the match is indeed a match */
    if (zmemcmp(s->window + match,
                s->window + start, length) != EQUAL) {
        fprintf(stderr, " start %u, match %u, length %d\n",
                start, match, length);
        do {
            fprintf(stderr, "%c%c", s->window[match++], s->window[start++]);
        } while (--length != 0);
        z_error("invalid match");
    }
    if (z_verbose > 1) {
        fprintf(stderr,"\\[%d,%d]", start - match, length);
        do { putc(s->window[start++], stderr); } while (--length != 0);
    }
}
#else
#  define check_match(s, start, match, length)
#endif /* ZLIB_DEBUG */

/* ===========================================================================
 * Fill the window when the lookahead becomes insufficient.
 * Updates strstart and lookahead.
 *
 * IN assertion: lookahead < MIN_LOOKAHEAD
 * OUT assertions: strstart <= window_size-MIN_LOOKAHEAD
 *    At least one byte has been read, or avail_in == 0; reads are
 *    performed for at least two bytes (required for the zip translate_eol
 *    option -- not supported here).
 */
local void fill_window(s)
    deflate_state *s;
{
    unsigned n;
    unsigned more;    /* Amount of free space at the end of the window. */
    uInt wsize = s->w_size;

    Assert(s->lookahead < MIN_LOOKAHEAD, "already enough lookahead");

#if MIN_MATCH != 3
            Call UPDATE_HASH() MIN_MATCH-3 more times
#endif
            while (s->insert) {
                UPDATE_HASH(s, s->ins_h, s->window[str + MIN_MATCH-1]);
#ifndef FASTEST
                s->prev[str & s->w_mask] = s->head[s->ins_h];
#endif
                // 如果没有可用的输入数据，或者插入的字节数为0，则跳出循环
                s->head[s->ins_h] = (Pos)str;
                // 将字符串的地址赋值给头部数组的指定位置
                str++;
                // 字符串指针向后移动一位
                s->insert--;
                // 插入的字节数减一
                if (s->lookahead + s->insert < MIN_MATCH)
                    // 如果前瞻字节数和插入的字节数之和小于最小匹配长度，则跳出循环
                    break;
            }
        }
        /* 如果整个输入的字节数小于最小匹配长度，ins_h 是垃圾值，
         * 但这并不重要，因为只有文字字节会被输出。
         */

    } while (s->lookahead < MIN_LOOKAHEAD && s->strm->avail_in != 0);

    /* 如果当前数据结束后的 WIN_INIT 字节从未被写入，
     * 则将这些字节清零，以避免内存检查报告最长匹配例程使用未初始化的字节。
     * 更新下一次通过这里的高水位标记。WIN_INIT 设置为 MAX_MATCH，因为最长匹配例程允许扫描到 strstart + MAX_MATCH，忽略前瞻。
     */
    # 如果当前的高水位小于窗口大小
    if (s->high_water < s->window_size) {
        # 计算当前数据的结束位置
        ulg curr = s->strstart + (ulg)(s->lookahead);
        ulg init;

        # 如果高水位小于当前数据的结束位置
        if (s->high_water < curr) {
            # 将高水位以下的数据清零，清零的长度为窗口大小减去当前数据结束位置
            init = s->window_size - curr;
            if (init > WIN_INIT)
                init = WIN_INIT;
            zmemzero(s->window + curr, (unsigned)init);
            s->high_water = curr + init;
        }
        # 如果高水位在当前数据结束位置和当前数据结束位置加上WIN_INIT之间
        else if (s->high_water < (ulg)curr + WIN_INIT) {
            # 将高水位到当前数据结束位置加上WIN_INIT之间的数据清零
            init = (ulg)curr + WIN_INIT - s->high_water;
            if (init > s->window_size - s->high_water)
                init = s->window_size - s->high_water;
            zmemzero(s->window + s->high_water, (unsigned)init);
            s->high_water += init;
        }
    }

    # 确保窗口中有足够的空间进行搜索
    Assert((ulg)s->strstart <= s->window_size - MIN_LOOKAHEAD,
           "not enough room for search");
/* ===========================================================================
 * Flush the current block, with given end-of-file flag.
 * IN assertion: strstart is set to the end of the current match.
 */
# 刷新当前块，带有给定的文件结束标志。
# 在断言中：strstart 设置为当前匹配的结束。
#define FLUSH_BLOCK_ONLY(s, last) { \
   _tr_flush_block(s, (s->block_start >= 0L ? \
                   (charf *)&s->window[(unsigned)s->block_start] : \
                   (charf *)Z_NULL), \
                (ulg)((long)s->strstart - s->block_start), \
                (last)); \
   s->block_start = s->strstart; \
   flush_pending(s->strm); \
   Tracev((stderr,"[FLUSH]")); \
}

/* Same but force premature exit if necessary. */
# 相同，但如果需要的话强制提前退出。
#define FLUSH_BLOCK(s, last) { \
   FLUSH_BLOCK_ONLY(s, last); \
   if (s->strm->avail_out == 0) return (last) ? finish_started : need_more; \
}

/* Maximum stored block length in deflate format (not including header). */
# deflate 格式中最大存储块长度（不包括头部）。
#define MAX_STORED 65535

/* Minimum of a and b. */
# a 和 b 的最小值。
#define MIN(a, b) ((a) > (b) ? (b) : (a))

/* ===========================================================================
 * Copy without compression as much as possible from the input stream, return
 * the current block state.
 *
 * In case deflateParams() is used to later switch to a non-zero compression
 * level, s->matches (otherwise unused when storing) keeps track of the number
 * of hash table slides to perform. If s->matches is 1, then one hash table
 * slide will be done when switching. If s->matches is 2, the maximum value
 * allowed here, then the hash table will be cleared, since two or more slides
 * is the same as a clear.
 *
 * deflate_stored() is written to minimize the number of times an input byte is
 * copied. It is most efficient with large input and output buffers, which
 * maximizes the opportunities to have a single copy from next_in to next_out.
 */
# 尽可能多地从输入流中复制而不压缩，返回当前块状态。
# 如果稍后使用 deflateParams() 切换到非零压缩级别，则 s->matches（在存储时未使用）会跟踪要执行的哈希表滑动次数。
# 如果 s->matches 为 1，则在切换时将执行一个哈希表滑动。如果 s->matches 为 2，这里允许的最大值，则哈希表将被清除，因为两次或更多次滑动等同于清除。
# deflate_stored() 被编写为最小化输入字节被复制的次数。它在有大的输入和输出缓冲区时效率最高，这最大化了从 next_in 到 next_out 的单次复制的机会。
local block_state deflate_stored(s, flush)
    deflate_state *s;
    int flush;
{
    /* 当不刷新或完成时，最小值的块大小。默认情况下为32K。当memLevel == 1时，可以小至507字节。对于大输入和输出缓冲区，存储块大小将更大。 */
    unsigned min_block = MIN(s->pending_buf_size - 5, s->w_size);

    /* 尽可能将与min_block大小或更大的存储块直接复制到next_out。如果刷新，则将剩余的可用输入作为存储块复制到next_out，如果有足够的空间。 */
    unsigned len, left, have, last = 0;
    unsigned used = s->strm->avail_in;
#ifdef ZLIB_DEBUG
        /* 如果定义了 ZLIB_DEBUG 宏，则更新压缩长度和比特数 */
        s->compressed_len += len << 3;
        s->bits_sent += len << 3;
#endif

        /* 从窗口中复制未压缩的字节到 next_out */
        if (left) {
            if (left > len)
                left = len;
            zmemcpy(s->strm->next_out, s->window + s->block_start, left);
            s->strm->next_out += left;
            s->strm->avail_out -= left;
            s->strm->total_out += left;
            s->block_start += left;
            len -= left;
        }

        /* 直接从 next_in 复制未压缩的字节到 next_out，并更新校验值 */
        if (len) {
            read_buf(s->strm, s->strm->next_out, len);
            s->strm->next_out += len;
            s->strm->avail_out -= len;
            s->strm->total_out += len;
        }
    } while (last == 0);

    /* 更新滑动窗口，使用已复制数据的最后 s->w_size 字节，如果复制的字节少于 s->w_size，则将所有复制的数据追加到现有窗口中。
    同时更新要插入哈希表的字节数，以防 deflateParams() 切换到非零压缩级别。 */
    used -= s->strm->avail_in;      /* 直接复制的输入字节数 */
    if (used) {
        /* 如果有任何输入被使用，则窗口中不会有未使用的输入，因此 s->block_start == s->strstart。*/
        if (used >= s->w_size) {    /* 替换先前的历史记录 */
            s->matches = 2;         /* 清除哈希 */
            zmemcpy(s->window, s->strm->next_in - s->w_size, s->w_size);
            s->strstart = s->w_size;
            s->insert = s->strstart;
        }
        else {
            if (s->window_size - s->strstart <= used) {
                /* 将窗口向下滑动。*/
                s->strstart -= s->w_size;
                zmemcpy(s->window, s->window + s->w_size, s->strstart);
                if (s->matches < 2)
                    s->matches++;   /* 添加一个待处理的 slide_hash() */
                if (s->insert > s->strstart)
                    s->insert = s->strstart;
            }
            zmemcpy(s->window + s->strstart, s->strm->next_in - used, used);
            s->strstart += used;
            s->insert += MIN(used, s->w_size - s->insert);
        }
        s->block_start = s->strstart;
    }
    if (s->high_water < s->strstart)
        s->high_water = s->strstart;

    /* 如果上一个块已写入 next_out，则完成。*/
    if (last)
        return finish_done;

    /* 如果刷新并且所有输入都已消耗，则完成。*/
    if (flush != Z_NO_FLUSH && flush != Z_FINISH &&
        s->strm->avail_in == 0 && (long)s->strstart == s->block_start)
        return block_done;

    /* 用任何剩余的输入填充窗口。*/
    have = s->window_size - s->strstart;
    # 如果输入缓冲区中的数据大于当前可用数据，并且块的起始位置大于等于滑动窗口的大小
    if (s->strm->avail_in > have && s->block_start >= (long)s->w_size) {
        # 将窗口向下滑动
        s->block_start -= s->w_size;
        s->strstart -= s->w_size;
        zmemcpy(s->window, s->window + s->w_size, s->strstart);
        if (s->matches < 2)
            s->matches++;           # 添加一个待处理的 slide_hash()
        have += s->w_size;          # 现在有更多的空间
        if (s->insert > s->strstart)
            s->insert = s->strstart;
    }
    # 如果有的数据大于输入缓冲区中的可用数据
    if (have > s->strm->avail_in)
        have = s->strm->avail_in;
    # 如果有数据
    if (have) {
        read_buf(s->strm, s->window + s->strstart, have);
        s->strstart += have;
        s->insert += MIN(have, s->w_size - s->insert);
    }
    # 如果高水位小于当前的起始位置
    if (s->high_water < s->strstart)
        s->high_water = s->strstart;

    # 计算需要的字节数
    have = (s->bi_valid + 42) >> 3;         # 头部字节数
    have = MIN(s->pending_buf_size - have, MAX_STORED);  # 最大存储块长度
    min_block = MIN(have, s->w_size);
    left = s->strstart - s->block_start;
    # 如果剩余的数据大于等于最小块长度，或者（剩余数据不为零或者处于 Z_FINISH 状态）并且不是 Z_NO_FLUSH 状态，并且输入缓冲区中没有数据，并且剩余数据小于等于可用的数据
    if (left >= min_block ||
        ((left || flush == Z_FINISH) && flush != Z_NO_FLUSH &&
         s->strm->avail_in == 0 && left <= have)) {
        len = MIN(left, have);
        last = flush == Z_FINISH && s->strm->avail_in == 0 &&
               len == left ? 1 : 0;
        _tr_stored_block(s, (charf *)s->window + s->block_start, len, last);
        s->block_start += len;
        flush_pending(s->strm);
    }

    # 已经处理了所有可用的输入和输出
    return last ? finish_started : need_more;
# 结束当前的代码块

/* ===========================================================================
 * 从输入流中尽可能多地进行压缩，返回当前的块状态。
 * 此函数不执行匹配的惰性评估，并且仅为无匹配的字符串或短匹配在字典中插入新字符串。
 * 仅用于快速压缩选项。
 */
# 定义一个名为deflate_fast的本地函数，接受参数s和flush
local block_state deflate_fast(s, flush)
    # 定义IPos类型的变量hash_head，表示哈希链的头部
    IPos hash_head;       
    # 定义整型变量bflush，如果当前块必须被刷新，则设置为1
    int bflush;           
    for (;;) {
        /* 确保我们始终有足够的预读，除非在输入文件的末尾。我们需要 MAX_MATCH 字节
         * 用于下一个匹配，再加上 MIN_MATCH 字节来插入下一个匹配后面的字符串。
         */
        if (s->lookahead < MIN_LOOKAHEAD) {
            fill_window(s);
            if (s->lookahead < MIN_LOOKAHEAD && flush == Z_NO_FLUSH) {
                return need_more;
            }
            if (s->lookahead == 0) break; /* 刷新当前块 */
        }

        /* 将字符串 window[strstart .. strstart + 2] 插入字典，并将 hash_head 设置为哈希链的头部：
         */
        hash_head = NIL;
        if (s->lookahead >= MIN_MATCH) {
            INSERT_STRING(s, s->strstart, hash_head);
        }

        /* 查找最长匹配，丢弃那些 <= prev_length 的匹配。
         * 在这一点上，我们总是有 match_length < MIN_MATCH
         */
        if (hash_head != NIL && s->strstart - hash_head <= MAX_DIST(s)) {
            /* 为了简化代码，我们防止与窗口索引 0 的字符串匹配
             * （特别是在输入文件开头避免字符串与自身匹配）。
             */
            s->match_length = longest_match (s, hash_head);
            /* longest_match() 设置 match_start */
        }
        if (s->match_length >= MIN_MATCH) {
            check_match(s, s->strstart, s->match_start, s->match_length);

            _tr_tally_dist(s, s->strstart - s->match_start,
                           s->match_length - MIN_MATCH, bflush);

            s->lookahead -= s->match_length;

            /* 仅当匹配长度不太大时，在哈希表中插入新字符串。这样可以节省时间但降低压缩效果。
             */
#ifndef FASTEST
            // 如果不是最快模式，且匹配长度小于最大插入长度，并且前瞻大于最小匹配长度
            if (s->match_length <= s->max_insert_length &&
                s->lookahead >= MIN_MATCH) {
                // 匹配长度减一，因为字符串在表中
                s->match_length--; /* string at strstart already in table */
                // 循环直到匹配长度为0
                do {
                    // 字符串起始位置向后移动一位
                    s->strstart++;
                    // 插入字符串到哈希表中
                    INSERT_STRING(s, s->strstart, hash_head);
                    /* strstart never exceeds WSIZE-MAX_MATCH, so there are
                     * always MIN_MATCH bytes ahead.
                     */
                } while (--s->match_length != 0);
                // 字符串起始位置再向后移动一位
                s->strstart++;
            } else
#endif
            {
                // 字符串起始位置向后移动匹配长度的距离
                s->strstart += s->match_length;
                // 匹配长度置为0
                s->match_length = 0;
                // 更新插入哈希值
                s->ins_h = s->window[s->strstart];
                // 更新哈希值
                UPDATE_HASH(s, s->ins_h, s->window[s->strstart + 1]);
#if MIN_MATCH != 3
                // 调用UPDATE_HASH() MIN_MATCH-3次
                Call UPDATE_HASH() MIN_MATCH-3 more times
#endif
                /* If lookahead < MIN_MATCH, ins_h is garbage, but it does not
                 * matter since it will be recomputed at next deflate call.
                 */
            }
        } else {
            /* No match, output a literal byte */
            // 输出一个字面字节
            Tracevv((stderr,"%c", s->window[s->strstart]));
            _tr_tally_lit(s, s->window[s->strstart], bflush);
            // 前瞻减一
            s->lookahead--;
            // 字符串起始位置向后移动一位
            s->strstart++;
        }
        // 如果需要刷新，则刷新块
        if (bflush) FLUSH_BLOCK(s, 0);
    }
    // 插入位置为字符串起始位置和最小匹配长度-1的较小值
    s->insert = s->strstart < MIN_MATCH-1 ? s->strstart : MIN_MATCH-1;
    // 如果是Z_FINISH模式
    if (flush == Z_FINISH) {
        // 刷新块并返回完成状态
        FLUSH_BLOCK(s, 1);
        return finish_done;
    }
    // 如果sym_next不为空
    if (s->sym_next)
        // 刷新块
        FLUSH_BLOCK(s, 0);
    // 返回块完成状态
    return block_done;
}

#ifndef FASTEST
/* ===========================================================================
 * Same as above, but achieves better compression. We use a lazy
 * evaluation for matches: a match is finally adopted only if there is
 * no better match at the next window position.
 */
local block_state deflate_slow(s, flush)
    deflate_state *s;
    int flush;
{
    # 哈希链表的头部
    IPos hash_head;          
    # 当前块是否需要刷新
    int bflush;              

    # 处理输入块
    for (;;) {
        # 确保始终有足够的预读，除非在输入文件末尾。我们需要 MAX_MATCH 字节用于下一个匹配，再加上 MIN_MATCH 字节来插入下一个匹配后面的字符串。
        if (s->lookahead < MIN_LOOKAHEAD) {
            fill_window(s);
            if (s->lookahead < MIN_LOOKAHEAD && flush == Z_NO_FLUSH) {
                return need_more;
            }
            if (s->lookahead == 0) break; /* 刷新当前块 */
        }

        # 将字符串 window[strstart .. strstart + 2] 插入字典，并将 hash_head 设置为哈希链表的头部
        hash_head = NIL;
        if (s->lookahead >= MIN_MATCH) {
            INSERT_STRING(s, s->strstart, hash_head);
        }

        # 查找最长匹配，丢弃那些 <= prev_length 的匹配
        s->prev_length = s->match_length, s->prev_match = s->match_start;
        s->match_length = MIN_MATCH-1;

        if (hash_head != NIL && s->prev_length < s->max_lazy_match &&
            s->strstart - hash_head <= MAX_DIST(s)) {
            # 为了简化代码，我们阻止与窗口索引 0 的字符串匹配（特别是在输入文件开头避免与自身匹配的字符串）
            s->match_length = longest_match (s, hash_head);
            # longest_match() 设置 match_start

            if (s->match_length <= 5 && (s->strategy == Z_FILTERED
#if TOO_FAR <= 32767
                || (s->match_length == MIN_MATCH &&
                    s->strstart - s->match_start > TOO_FAR)
    }
    Assert (flush != Z_NO_FLUSH, "no flush?");
    if (s->match_available) {
        Tracevv((stderr,"%c", s->window[s->strstart - 1]));
        _tr_tally_lit(s, s->window[s->strstart - 1], bflush);
        s->match_available = 0;
    }
    s->insert = s->strstart < MIN_MATCH-1 ? s->strstart : MIN_MATCH-1;
    if (flush == Z_FINISH) {
        FLUSH_BLOCK(s, 1);
        return finish_done;
    }
    if (s->sym_next)
        FLUSH_BLOCK(s, 0);
    return block_done;
}
#endif /* FASTEST */

/* ===========================================================================
 * For Z_RLE, simply look for runs of bytes, generate matches only of distance
 * one.  Do not maintain a hash table.  (It will be regenerated if this run of
 * deflate switches away from Z_RLE.)
 */
local block_state deflate_rle(s, flush)
    deflate_state *s;
    int flush;
{
    int bflush;             /* set if current block must be flushed */
    uInt prev;              /* byte at distance one to match */
    Bytef *scan, *strend;   /* scan goes up to strend for length of run */

    }
    s->insert = 0;
    if (flush == Z_FINISH) {
        FLUSH_BLOCK(s, 1);
        return finish_done;
    }
    if (s->sym_next)
        FLUSH_BLOCK(s, 0);
    return block_done;
}

/* ===========================================================================
 * For Z_HUFFMAN_ONLY, do not look for matches.  Do not maintain a hash table.
 * (It will be regenerated if this run of deflate switches away from Huffman.)
 */
local block_state deflate_huff(s, flush)
    deflate_state *s;
    int flush;
{
    int bflush;             /* set if current block must be flushed */
    for (;;) {
        /* 无限循环，直到条件满足跳出循环 */
        /* 确保我们有一个要写入的字面量 */
        if (s->lookahead == 0) {
            /* 填充窗口 */
            fill_window(s);
            if (s->lookahead == 0) {
                if (flush == Z_NO_FLUSH)
                    return need_more;  /* 如果没有数据可读，且不需要刷新，则返回需要更多数据 */
                break;      /* 刷新当前块 */
            }
        }

        /* 输出一个字面字节 */
        s->match_length = 0;  /* 匹配长度置零 */
        Tracevv((stderr,"%c", s->window[s->strstart]));  /* 输出调试信息 */
        _tr_tally_lit(s, s->window[s->strstart], bflush);  /* 记录字面量 */
        s->lookahead--;  /* 可读数据减一 */
        s->strstart++;  /* 起始位置后移一位 */
        if (bflush) FLUSH_BLOCK(s, 0);  /* 如果需要刷新，则刷新块 */
    }
    s->insert = 0;  /* 插入位置置零 */
    if (flush == Z_FINISH) {
        FLUSH_BLOCK(s, 1);  /* 如果需要刷新，则刷新块 */
        return finish_done;  /* 返回完成状态 */
    }
    if (s->sym_next)
        FLUSH_BLOCK(s, 0);  /* 如果有下一个符号，则刷新块 */
    return block_done;  /* 返回块完成状态 */
# 闭合前面的函数定义
```