# `nmap\libpcre\src\pcre2_compile.c`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2022 University of Cambridge

-----------------------------------------------------------------------------
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

    * Redistributions of source code must retain the above copyright notice,
      this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.

    * Neither the name of the University of Cambridge nor the names of its
      contributors may be used to endorse or promote products derived from
      this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
/*
  该部分代码是关于软件使用许可和免责声明的声明
*/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#define NLBLOCK cb             /* 包含换行信息的块 */
#define PSSTART start_pattern  /* 包含处理过的字符串起始位置的字段 */
#define PSEND   end_pattern    /* 包含处理过的字符串结束位置的字段 */

#include "pcre2_internal.h"

/* 在罕见的错误情况下，调试可能需要调用 pcre2_printint() */

#if 0
#ifdef EBCDIC
#define PRINTABLE(c) ((c) >= 64 && (c) < 255)
#else
#define PRINTABLE(c) ((c) >= 32 && (c) < 127)
#endif
#include "pcre2_printint.c"
#define DEBUG_CALL_PRINTINT
#endif

/* 可以通过这些定义启用其他调试代码 */

/* #define DEBUG_SHOW_CAPTURES */
/* #define DEBUG_SHOW_PARSED */

/* 有一些随着不同代码单元大小而变化的东西。通过定义宏来处理它们，以最小化 #if 的使用。 */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define STRING_UTFn_RIGHTPAR     STRING_UTF8_RIGHTPAR, 5
#define XDIGIT(c)                xdigitab[c]

#else  /* 16位或32位 */
#define XDIGIT(c)                (MAX_255(c)? xdigitab[c] : 0xff)

#if PCRE2_CODE_UNIT_WIDTH == 16
#define STRING_UTFn_RIGHTPAR     STRING_UTF16_RIGHTPAR, 6

#else  /* 32位 */
#define STRING_UTFn_RIGHTPAR     STRING_UTF32_RIGHTPAR, 6
#endif
#endif

/* 宏用于在解析的模式中存储和检索 PCRE2_SIZE 值，该模式由 uint32_t 元素组成。
假设如果 uint32_t 无法容纳它，那么两个 uint32_t 将能够容纳它（即假设是一个64位的世界）。 */

#if PCRE2_SIZE_MAX <= UINT32_MAX
#define PUTOFFSET(s,p) *p++ = s
#define GETOFFSET(s,p) s = *p++
#define GETPLUSOFFSET(s,p) s = *(++p)
#define READPLUSOFFSET(s,p) s = p[1]
#define SKIPOFFSET(p) p++
#define SIZEOFFSET 1
#else
#define PUTOFFSET(s,p) \
  { *p++ = (uint32_t)(s >> 32); *p++ = (uint32_t)(s & 0xffffffff); }
// 定义宏，用于获取偏移量
#define GETOFFSET(s,p) \
  { s = ((PCRE2_SIZE)p[0] << 32) | (PCRE2_SIZE)p[1]; p += 2; }
// 定义宏，用于获取带有加号的偏移量
#define GETPLUSOFFSET(s,p) \
  { s = ((PCRE2_SIZE)p[1] << 32) | (PCRE2_SIZE)p[2]; p += 2; }
// 定义宏，用于读取带有加号的偏移量
#define READPLUSOFFSET(s,p) \
  { s = ((PCRE2_SIZE)p[1] << 32) | (PCRE2_SIZE)p[2]; }
// 定义宏，用于跳过偏移量
#define SKIPOFFSET(p) p += 2
// 定义宏，用于偏移量的大小
#define SIZEOFFSET 2
#endif

/* 用于操作解析模式向量元素的宏 */

// 获取元数据的宏
#define META_CODE(x)   (x & 0xffff0000u)
// 获取元数据的宏
#define META_DATA(x)   (x & 0x0000ffffu)
// 获取元数据之间的差异的宏
#define META_DIFF(x,y) ((x-y)>>16)

/* 用于允许相互递归的函数定义 */

#ifdef SUPPORT_UNICODE
// 将列表添加到类内部的函数定义
static unsigned int
  add_list_to_class_internal(uint8_t *, PCRE2_UCHAR **, uint32_t,
    compile_block *, const uint32_t *, unsigned int);
#endif

// 编译正则表达式的函数定义
static int
  compile_regex(uint32_t, PCRE2_UCHAR **, uint32_t **, int *, uint32_t,
    uint32_t *, uint32_t *, uint32_t *, uint32_t *, branch_chain *,
    compile_block *, PCRE2_SIZE *);

// 获取分支长度的函数定义
static int
  get_branchlength(uint32_t **, int *, int *, parsed_recurse_check *,
    compile_block *);

// 设置回溯长度的函数定义
static BOOL
  set_lookbehind_lengths(uint32_t **, int *, int *, parsed_recurse_check *,
    compile_block *);

// 检查回溯的函数定义
static int
  check_lookbehinds(uint32_t *, uint32_t **, parsed_recurse_check *,
    compile_block *, int *);


/*************************************************
*      Code parameters and static tables         *
*************************************************/

// 最大分组号
#define MAX_GROUP_NUMBER   65535u
// 最大重复次数
#define MAX_REPEAT_COUNT   65535u
// 无限重复
#define REPEAT_UNLIMITED   (MAX_REPEAT_COUNT+1)

/* COMPILE_WORK_SIZE 指定堆栈工作空间的大小，它在不同的模式扫描中以不同的方式使用。
   解析和组识别预扫描用它来处理嵌套，并且需要它是16位对齐的。
   在第一次编译阶段，确定需要多少内存时，我们以代码单元的大小定义了它，
   我们设置 C16_WORK_SIZE 作为16位向量中的元素数量。

   在第一次编译阶段，确定需要多少内存时，我们以代码单元的大小定义了它，
   我们设置 C16_WORK_SIZE 作为16位向量中的元素数量。
/* 
   正则表达式部分编译到这个空间中，但编译部分会尽快丢弃，希望永远不会发生溢出。
   但是，代码确实会检查溢出，这可能会发生在病态模式中。工作空间的大小取决于 LINK_SIZE，因为编译项的长度随之变化。

   在实际编译阶段，这个工作空间目前没有使用。 
*/
#define COMPILE_WORK_SIZE (3000*LINK_SIZE)   /* Size in code units */

#define C16_WORK_SIZE \
  ((COMPILE_WORK_SIZE * sizeof(PCRE2_UCHAR))/sizeof(uint16_t))

/* 用于缓存捕获组大小信息的 uint32_t 向量，以提高性能。默认情况下，创建一个具有此大小的堆栈。 */
#define GROUPINFO_DEFAULT_SIZE 256

/* 溢出测试检查稍小一些的大小，以便在实际超出数据块末尾之前检测到溢出。 */
#define WORK_SIZE_SAFETY_MARGIN (100)

/* 此值确定用于在预编译期间记住命名组的初始向量的大小。它在堆栈上分配，但如果太小，它会以类似于工作空间的方式扩展。该值是列表中的插槽数。 */
#define NAMED_GROUP_LIST_SIZE  20

/* 模式的预编译通过在 uint32_t 向量中创建一个解析模式。对于短模式，这个向量存在于堆栈上，大小为此值。对于更长的模式，使用堆内存。 */
#define PARSED_PATTERN_DEFAULT_SIZE 1024

/* 用于确保保存编译模式长度的变量不会溢出的最大长度值。我们将其设置为比 INT_MAX 小一点，以便考虑到添加组终止代码单元，这样我们就不必每次都检查它们。 */
#define OFLOW_MAX (INT_MAX - 20)

/* 解析模式的代码值，存储在 32 位无符号整数向量中。小于 META_END 的值是文字数据值。 */
# 定义元字符的常量，用于解析模式元素
# 当这些定义被修改时，下面的每个代码的额外长度表（meta_extra_lengths）必须更新以保持同步
#define META_END              0x80000000u  /* 模式结束 */

#define META_ALT              0x80010000u  /* 交替 */
#define META_ATOMIC           0x80020000u  /* 原子组 */
#define META_BACKREF          0x80030000u  /* 反向引用 */
#define META_BACKREF_BYNAME   0x80040000u  /* \k'name' */
#define META_BIGVALUE         0x80050000u  /* 下一个是大于META_END的文字 */
#define META_CALLOUT_NUMBER   0x80060000u  /* (?C 带有数字参数 */
#define META_CALLOUT_STRING   0x80070000u  /* (?C 带有字符串参数 */
#define META_CAPTURE          0x80080000u  /* 捕获括号 */
#define META_CIRCUMFLEX       0x80090000u  /* ^ 元字符 */
#define META_CLASS            0x800a0000u  /* 开始非空类 */
#define META_CLASS_EMPTY      0x800b0000u  /* 空类 */
#define META_CLASS_EMPTY_NOT  0x800c0000u  /* 负空类 */
#define META_CLASS_END        0x800d0000u  /* 非空类结束 */
#define META_CLASS_NOT        0x800e0000u  /* 开始非空负类 */
#define META_COND_ASSERT      0x800f0000u  /* (?(?assertion)... */
#define META_COND_DEFINE      0x80100000u  /* (?(DEFINE)... */
#define META_COND_NAME        0x80110000u  /* (?(<name>)... */
#define META_COND_NUMBER      0x80120000u  /* (?(digits)... */
#define META_COND_RNAME       0x80130000u  /* (?(R&name)... */
#define META_COND_RNUMBER     0x80140000u  /* (?(Rdigits)... */
#define META_COND_VERSION     0x80150000u  /* (?(VERSION<op>x.y)... */
#define META_DOLLAR           0x80160000u  /* $ 元字符 */
#define META_DOT              0x80170000u  /* . 元字符 */
# 定义各种元字符的十六进制表示，用于正则表达式的解析和匹配
#define META_ESCAPE           0x80180000u  /* \d and friends */
#define META_KET              0x80190000u  /* closing parenthesis */
#define META_NOCAPTURE        0x801a0000u  /* no capture parens */
#define META_OPTIONS          0x801b0000u  /* (?i) and friends */
#define META_POSIX            0x801c0000u  /* POSIX class item */
#define META_POSIX_NEG        0x801d0000u  /* negative POSIX class item */
#define META_RANGE_ESCAPED    0x801e0000u  /* range with at least one escape */
#define META_RANGE_LITERAL    0x801f0000u  /* range defined literally */
#define META_RECURSE          0x80200000u  /* Recursion */
#define META_RECURSE_BYNAME   0x80210000u  /* (?&name) */
#define META_SCRIPT_RUN       0x80220000u  /* (*script_run:...) */

/* These must be kept together to make it easy to check that an assertion
is present where expected in a conditional group. */

# 定义各种条件断言的十六进制表示，用于正则表达式的条件匹配
#define META_LOOKAHEAD        0x80230000u  /* (?= */
#define META_LOOKAHEADNOT     0x80240000u  /* (?! */
#define META_LOOKBEHIND       0x80250000u  /* (?<= */
#define META_LOOKBEHINDNOT    0x80260000u  /* (?<! */

/* These cannot be conditions */

# 定义不能作为条件的断言的十六进制表示
#define META_LOOKAHEAD_NA     0x80270000u  /* (*napla: */
#define META_LOOKBEHIND_NA    0x80280000u  /* (*naplb: */

# 定义用于条件匹配的标记和操作的十六进制表示
#define META_MARK             0x80290000u  /* (*MARK) */
#define META_ACCEPT           0x802a0000u  /* (*ACCEPT) */
#define META_FAIL             0x802b0000u  /* (*FAIL) */
#define META_COMMIT           0x802c0000u  /* These               */
#define META_COMMIT_ARG       0x802d0000u  /*   pairs             */
#define META_PRUNE            0x802e0000u  /*     must            */
#define META_PRUNE_ARG        0x802f0000u  /*       be            */
#define META_SKIP             0x80300000u  /*         kept        */
#define META_SKIP_ARG         0x80310000u  /*           in        */
# 定义元字符的十六进制数值，表示不同的元字符
#define META_THEN             0x80320000u  /*             this    */
#define META_THEN_ARG         0x80330000u  /*               order */

# 定义不同的量词元字符的十六进制数值
/* These must be kept in groups of adjacent 3 values, and all together. */
#define META_ASTERISK         0x80340000u  /* *  */
#define META_ASTERISK_PLUS    0x80350000u  /* *+ */
#define META_ASTERISK_QUERY   0x80360000u  /* *? */
#define META_PLUS             0x80370000u  /* +  */
#define META_PLUS_PLUS        0x80380000u  /* ++ */
#define META_PLUS_QUERY       0x80390000u  /* +? */
#define META_QUERY            0x803a0000u  /* ?  */
#define META_QUERY_PLUS       0x803b0000u  /* ?+ */
#define META_QUERY_QUERY      0x803c0000u  /* ?? */
#define META_MINMAX           0x803d0000u  /* {n,m}  repeat */
#define META_MINMAX_PLUS      0x803e0000u  /* {n,m}+ repeat */
#define META_MINMAX_QUERY     0x803f0000u  /* {n,m}? repeat */

# 定义特殊的元字符，用于在字母断言表中区分不同的断言
#define META_FIRST_QUANTIFIER META_ASTERISK
#define META_LAST_QUANTIFIER  META_MINMAX_QUERY

# 定义特殊的元字符，用于在解析模式的表中区分不同的模式
/* This is a special "meta code" that is used only to distinguish (*asr: from
(*sr: in the table of aphabetic assertions. It is never stored in the parsed
pattern because (*asr: is turned into (*sr:(*atomic: at that stage. There is
therefore no need for it to have a length entry, so use a high value. */
#define META_ATOMIC_SCRIPT_RUN 0x8fff0000u

# 定义不同类型的跳过部分的枚举
/* Types for skipping parts of a parsed pattern. */
enum { PSKIP_ALT, PSKIP_CLASS, PSKIP_KET };

# 定义宏用于设置类位图中的单个位
/* Macro for setting individual bits in class bitmaps. It took some
experimenting to figure out how to stop gcc 5.3.0 from warning with
-Wconversion. This version gets a warning:

  #define SETBIT(a,b) a[(b)/8] |= (uint8_t)(1u << ((b)&7))

Let's hope the apparently less efficient version isn't actually so bad if the
compiler is clever with identical subexpressions. */
/* 定义一个宏，用于设置指定位置的位 */
#define SETBIT(a,b) a[(b)/8] = (uint8_t)(a[(b)/8] | (1u << ((b)&7)))

/* 用于无符号的xxcuflags变量的值和标志，这些变量与xxcu变量一起，涉及第一个和必需的代码单元。
大于或等于REQ_NONE的值表示“没有设置代码单元”；否则设置匹配的xxcu变量，并且低位值是相关的。 */
#define REQ_UNSET     0xffffffffu  /* 尚未找到任何内容 */
#define REQ_NONE      0xfffffffeu  /* 找到非固定字符 */
#define REQ_CASELESS  0x00000001u  /* xxcu中的代码单元是不区分大小写的 */
#define REQ_VARY      0x00000002u  /* 代码单元后面跟着非文字 */

/* 这些标志用于groupinfo向量。 */
#define GI_SET_FIXED_LENGTH    0x80000000u
#define GI_NOT_FIXED_LENGTH    0x40000000u
#define GI_FIXED_LENGTH_MASK   0x0000ffffu

/* 这个简单的十进制数字测试适用于ASCII/Unicode和EBCDIC，速度快（一个好的编译器可以将其转换为减法和无符号比较）。 */
#define IS_DIGIT(x) ((x) >= CHAR_0 && (x) <= CHAR_9)

/* 用于识别十六进制数字的表。chartables中的表依赖于区域设置，并且可能将任意字符标记为数字。我们只想识别0-9、a-z和A-Z作为十六进制数字，这就是为什么我们在这里有一个私有表。它需要256字节，但比进行字符值测试要快得多（至少在我计时的一些简单情况下），在一些应用中，我们希望PCRE2编译效率高，匹配效率高。表中的值是二进制十六进制数字值，对于非十六进制数字是0xff。 */

/* 这是“正常”情况，适用于ASCII系统和以UTF-8模式运行的EBCDIC系统。 */
#ifndef EBCDIC
# 定义一个静态的无符号8位整数数组，用于将字符转换为16进制数字
static const uint8_t xdigitab[] =
  {
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*   0-  7 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*   8- 15 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  16- 23 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  24- 31 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*    - '  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  ( - /  */
  0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07, /*  0 - 7  */
  0x08,0x09,0xff,0xff,0xff,0xff,0xff,0xff, /*  8 - ?  */
  0xff,0x0a,0x0b,0x0c,0x0d,0x0e,0x0f,0xff, /*  @ - G  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  H - O  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  P - W  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  X - _  */
  0xff,0x0a,0x0b,0x0c,0x0d,0x0e,0x0f,0xff, /*  ` - g  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  h - o  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  p - w  */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  x -127 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /* 128-135 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /* 136-143 */
  0xff,0xff,0xff,0xff,
# 定义一个包含256个元素的常量数组，用于将字符转换为16进制数字
static const uint8_t xdigitab[] =
  {
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*   0-  7  0 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*   8- 15    */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  16- 23 10 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  24- 31    */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  32- 39 20 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  40- 47    */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  48- 55 30 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  56- 63    */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*    - 71 40 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  72- |     */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  & - 87 50 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  88- 95    */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /*  - -103 60 */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /* 104- ?     */
  0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff, /* 112-119 70 */
  0xff,0xff,0xff,0xff,0xff,0xff,0
# 处理字母数字转义字符的表格。正返回值是简单的数据值；负值用于特殊的东西，比如 \d 等。零表示需要进一步处理（比如 \x），或者转义无效。

# 这是 ASCII 系统或以 UTF-8 模式运行的 EBCDIC 系统的“正常”表格。它从 '0' 到 'z'。
#ifndef EBCDIC
#define ESCAPES_FIRST       CHAR_0
#define ESCAPES_LAST        CHAR_z
#define UPPER_CASE(c)       (c-32)

static const short int escapes[] = {
     0,                       0,
     0,                       0,
     0,                       0,
     0,                       0,
     0,                       0,
     CHAR_COLON,              CHAR_SEMICOLON,
     CHAR_LESS_THAN_SIGN,     CHAR_EQUALS_SIGN,
     CHAR_GREATER_THAN_SIGN,  CHAR_QUESTION_MARK,
     CHAR_COMMERCIAL_AT,      -ESC_A,
     -ESC_B,                  -ESC_C,
     -ESC_D,                  -ESC_E,
     0,                       -ESC_G,
     -ESC_H,                  0,
     0,                       -ESC_K,
     0,                       0,
     -ESC_N,                  0,
     -ESC_P,                  -ESC_Q,
     -ESC_R,                  -ESC_S,
     0,                       0,
     -ESC_V,                  -ESC_W,
     -ESC_X,                  0,
     -ESC_Z,                  CHAR_LEFT_SQUARE_BRACKET,
     CHAR_BACKSLASH,          CHAR_RIGHT_SQUARE_BRACKET,
     CHAR_CIRCUMFLEX_ACCENT,  CHAR_UNDERSCORE,
     CHAR_GRAVE_ACCENT,       CHAR_BEL,
     -ESC_b,                  0,
     -ESC_d,                  CHAR_ESC,
     CHAR_FF,                 0,
     -ESC_h,                  0,
     0,                       -ESC_k,
     0,                       0,
     CHAR_LF,                 0,
     -ESC_p,                  0,
     CHAR_CR,                 -ESC_s,
     CHAR_HT,                 0,
     -ESC_v,                  -ESC_w,
     0,                       0,
     -ESC_z
};

#else
/* 这是针对没有 UTF-8 支持的 EBCDIC 系统的 "异常" 表格。
它从 'a' 到 '9' 运行。为了对 EBCDIC 功能进行一些最小的测试，有时会在 ASCII 系统上编译代码。
在这种情况下，我们不能使用 CHAR_a，因为它被定义为 'a'，这当然会选择 ASCII 值。 */

#if 'a' == 0x81                    /* 检查是否是真正的 EBCDIC 环境 */
#define ESCAPES_FIRST       CHAR_a
#define ESCAPES_LAST        CHAR_9
#define UPPER_CASE(c)       (c+64)
#else                              /* 在 ASCII 环境中进行测试 */
#define ESCAPES_FIRST  ((unsigned char)'\x81')   /* EBCDIC 'a' */
#define ESCAPES_LAST   ((unsigned char)'\xf9')   /* EBCDIC '9' */
#define UPPER_CASE(c)  (c-32)
#endif

static const short int escapes[] = {
/*  80 */         CHAR_BEL, -ESC_b,       0, -ESC_d, CHAR_ESC, CHAR_FF,      0,
/*  88 */ -ESC_h,        0,      0,     '{',      0,        0,       0,      0,
/*  90 */      0,        0, -ESC_k,       0,      0,  CHAR_LF,       0, -ESC_p,
/*  98 */      0,  CHAR_CR,      0,     '}',      0,        0,       0,      0,
/*  A0 */      0,      '~', -ESC_s, CHAR_HT,      0,   -ESC_v,  -ESC_w,      0,
/*  A8 */      0,   -ESC_z,      0,       0,      0,      '[',       0,      0,
/*  B0 */      0,        0,      0,       0,      0,        0,       0,      0,
/*  B8 */      0,        0,      0,       0,      0,      ']',     '=',    '-',
/*  C0 */    '{',   -ESC_A, -ESC_B,  -ESC_C, -ESC_D,   -ESC_E,       0, -ESC_G,
/*  C8 */ -ESC_H,        0,      0,       0,      0,        0,       0,      0,
/*  D0 */    '}',        0, -ESC_K,       0,      0,   -ESC_N,       0, -ESC_P,
/*  D8 */ -ESC_Q,   -ESC_R,      0,       0,      0,        0,       0,      0,
/*  E0 */   '\\',        0, -ESC_S,       0,      0,   -ESC_V,  -ESC_W, -ESC_X,
/*  E8 */      0,   -ESC_Z,      0,       0,      0,        0,       0,      0,
/*  F0 */      0,        0,      0,       0,      0,        0,       0,      0,
/*  F8 */      0,        0
};

/* We also need a table of characters that may follow \c in an EBCDIC
environment for characters 0-31. */

// 在 EBCDIC 环境中，需要一个字符表，用于表示字符 0-31 后可能跟随的字符

static unsigned char ebcdic_escape_c[] = "@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_";

#endif   /* EBCDIC */


/* Table of special "verbs" like (*PRUNE). This is a short table, so it is
searched linearly. Put all the names into a single string, in order to reduce
the number of relocations when a shared library is dynamically linked. The
string is built from string macros so that it works in UTF-8 mode on EBCDIC
platforms. */

// 特殊“动词”（如 (*PRUNE)）的表格。这是一个简短的表格，因此它是线性搜索的。将所有名称放入单个字符串中，以减少在动态链接共享库时的重定位数量。该字符串是从字符串宏构建的，以便在 EBCDIC 平台的 UTF-8 模式下工作。

typedef struct verbitem {
  unsigned int len;          /* Length of verb name */
  uint32_t meta;             /* Base META_ code */
  int has_arg;               /* Argument requirement */
} verbitem;

static const char verbnames[] =
  "\0"                       /* Empty name is a shorthand for MARK */
  STRING_MARK0
  STRING_ACCEPT0
  STRING_F0
  STRING_FAIL0
  STRING_COMMIT0
  STRING_PRUNE0
  STRING_SKIP0
  STRING_THEN;

static const verbitem verbs[] = {
  { 0, META_MARK,   +1 },  /* > 0 => must have an argument */
  { 4, META_MARK,   +1 },
  { 6, META_ACCEPT, -1 },  /* < 0 => Optional argument, convert to pre-MARK */
  { 1, META_FAIL,   -1 },
  { 4, META_FAIL,   -1 },
  { 6, META_COMMIT,  0 },
  { 5, META_PRUNE,   0 },  /* Optional argument; bump META code if found */
  { 4, META_SKIP,    0 },
  { 4, META_THEN,    0 }
};

static const int verbcount = sizeof(verbs)/sizeof(verbitem);

/* Verb opcodes, indexed by their META code offset from META_MARK. */

static const uint32_t verbops[] = {
  OP_MARK, OP_ACCEPT, OP_FAIL, OP_COMMIT, OP_COMMIT_ARG, OP_PRUNE,
  OP_PRUNE_ARG, OP_SKIP, OP_SKIP_ARG, OP_THEN, OP_THEN_ARG };

/* Table of "alpha assertions" like (*pla:...), similar to the (*VERB) table. */

typedef struct alasitem {
  unsigned int len;          /* Length of name */
  uint32_t meta;             /* Base META_ code */
} alasitem;
# 定义包含字符串的静态常量数组，用于存储正则表达式中的特殊字符名称
static const char alasnames[] =
  STRING_pla0                    # 正向预查
  STRING_plb0                    # 反向预查
  STRING_napla0                  # 非原子正向预查
  STRING_naplb0                  # 非原子反向预查
  STRING_nla0                    # 负向预查
  STRING_nlb0                    # 负向反向预查
  STRING_positive_lookahead0     # 正向预查
  STRING_positive_lookbehind0    # 反向预查
  STRING_non_atomic_positive_lookahead0  # 非原子正向预查
  STRING_non_atomic_positive_lookbehind0 # 非原子反向预查
  STRING_negative_lookahead0      # 负向预查
  STRING_negative_lookbehind0     # 负向反向预查
  STRING_atomic0                  # 原子组
  STRING_sr0                      # 脚本运行
  STRING_asr0                     # 原子脚本运行
  STRING_script_run0              # 脚本运行
  STRING_atomic_script_run        # 原子脚本运行

# 定义包含特殊字符元数据的静态常量数组
static const alasitem alasmeta[] = {
  {  3, META_LOOKAHEAD         },  # 预查
  {  3, META_LOOKBEHIND        },  # 反向预查
  {  5, META_LOOKAHEAD_NA      },  # 非原子预查
  {  5, META_LOOKBEHIND_NA     },  # 非原子反向预查
  {  3, META_LOOKAHEADNOT      },  # 负向预查
  {  3, META_LOOKBEHINDNOT     },  # 负向反向预查
  { 18, META_LOOKAHEAD         },  # 预查
  { 19, META_LOOKBEHIND        },  # 反向预查
  { 29, META_LOOKAHEAD_NA      },  # 非原子预查
  { 30, META_LOOKBEHIND_NA     },  # 非原子反向预查
  { 18, META_LOOKAHEADNOT      },  # 负向预查
  { 19, META_LOOKBEHINDNOT     },  # 负向反向预查
  {  6, META_ATOMIC            },  # 原子组
  {  2, META_SCRIPT_RUN        },  # 脚本运行
  {  3, META_ATOMIC_SCRIPT_RUN },  # 原子脚本运行
  { 10, META_SCRIPT_RUN        },  # 脚本运行
  { 17, META_ATOMIC_SCRIPT_RUN }   # 原子脚本运行
};

# 定义包含特殊字符元数据的静态常量数组的长度
static const int alascount = sizeof(alasmeta)/sizeof(alasitem);

# 定义用于存储不区分大小写和负重复操作码的偏移量数组
/* Offsets from OP_STAR for case-independent and negative repeat opcodes. */
static uint32_t chartypeoffset[] = {
  OP_STAR - OP_STAR,    OP_STARI - OP_STAR,    # 不区分大小写重复操作码的偏移量
  OP_NOTSTAR - OP_STAR, OP_NOTSTARI - OP_STAR  # 负重复操作码的偏移量
};

# 定义 POSIX 字符类名称和长度的表格
/* Tables of names of POSIX character classes and their lengths. The names are
now all in a single string, to reduce the number of relocations when a shared
library is dynamically loaded. The list of lengths is terminated by a zero
length entry. The first three must be alpha, lower, upper, as this is assumed
for handling case independence. The indices for graph, print, and punct are
needed, so identify them. */
// 定义包含 POSIX 类别名称的字符串
static const char posix_names[] =
  STRING_alpha0 STRING_lower0 STRING_upper0 STRING_alnum0
  STRING_ascii0 STRING_blank0 STRING_cntrl0 STRING_digit0
  STRING_graph0 STRING_print0 STRING_punct0 STRING_space0
  STRING_word0  STRING_xdigit;

// 定义包含 POSIX 类别名称长度的数组
static const uint8_t posix_name_lengths[] = {
  5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 6, 0 };

// 定义 POSIX 类别的常量
#define PC_GRAPH  8
#define PC_PRINT  9
#define PC_PUNCT 10

/* POSIX 类别的位图表。每个类别由一个基本位图组成，可选地添加或移除另一个位图。
然后，对于一些类别，还有一些额外的调整：对于[:blank:]，垂直空格字符被移除，
对于[:alpha:]和[:alnum:]，下划线字符被移除。表中的三元组由基本位图偏移、
第二个位图偏移或-1（如果没有第二个位图），以及一个非负值表示位图添加或
负值表示位图减法（如果有两个位图）。第三个字段的绝对值有以下含义：0 => 无调整，
1 => 移除垂直空格字符，2 => 移除下划线。 */
static const int posix_class_maps[] = {
  cbit_word,  cbit_digit, -2,             /* alpha */
  cbit_lower, -1,          0,             /* lower */
  cbit_upper, -1,          0,             /* upper */
  cbit_word,  -1,          2,             /* alnum - word without underscore */
  cbit_print, cbit_cntrl,  0,             /* ascii */
  cbit_space, -1,          1,             /* blank - a GNU extension */
  cbit_cntrl, -1,          0,             /* cntrl */
  cbit_digit, -1,          0,             /* digit */
  cbit_graph, -1,          0,             /* graph */
  cbit_print, -1,          0,             /* print */
  cbit_punct, -1,          0,             /* punct */
  cbit_space, -1,          0,             /* space */
  cbit_word,  -1,          0,             /* word - a Perl extension */
  cbit_xdigit,-1,          0              /* xdigit */
};

#ifdef SUPPORT_UNICODE
/* The POSIX class Unicode property substitutes that are used in UCP mode must
be in the order of the POSIX class names, defined above. */

// 定义用于UCP模式中的POSIX类Unicode属性替代物的数组，必须按照上面定义的POSIX类名称的顺序排列

static int posix_substitutes[] = {
  PT_GC, ucp_L,     /* alpha */
  PT_PC, ucp_Ll,    /* lower */
  PT_PC, ucp_Lu,    /* upper */
  PT_ALNUM, 0,      /* alnum */
  -1, 0,            /* ascii, treat as non-UCP */
  -1, 1,            /* blank, treat as \h */
  PT_PC, ucp_Cc,    /* cntrl */
  PT_PC, ucp_Nd,    /* digit */
  PT_PXGRAPH, 0,    /* graph */
  PT_PXPRINT, 0,    /* print */
  PT_PXPUNCT, 0,    /* punct */
  PT_PXSPACE, 0,    /* space */   /* Xps is POSIX space, but from 8.34 */
  PT_WORD, 0,       /* word  */   /* Perl and POSIX space are the same */
  -1, 0             /* xdigit, treat as non-UCP */
};
#define POSIX_SUBSIZE (sizeof(posix_substitutes) / (2*sizeof(uint32_t)))
#endif  /* SUPPORT_UNICODE */

/* Masks for checking option settings. When PCRE2_LITERAL is set, only a subset
are allowed. */

// 用于检查选项设置的掩码。当设置了PCRE2_LITERAL时，只允许一部分选项。

#define PUBLIC_LITERAL_COMPILE_OPTIONS \
  (PCRE2_ANCHORED|PCRE2_AUTO_CALLOUT|PCRE2_CASELESS|PCRE2_ENDANCHORED| \
   PCRE2_FIRSTLINE|PCRE2_LITERAL|PCRE2_MATCH_INVALID_UTF| \
   PCRE2_NO_START_OPTIMIZE|PCRE2_NO_UTF_CHECK|PCRE2_USE_OFFSET_LIMIT|PCRE2_UTF)

#define PUBLIC_COMPILE_OPTIONS \
  (PUBLIC_LITERAL_COMPILE_OPTIONS| \
   PCRE2_ALLOW_EMPTY_CLASS|PCRE2_ALT_BSUX|PCRE2_ALT_CIRCUMFLEX| \
   PCRE2_ALT_VERBNAMES|PCRE2_DOLLAR_ENDONLY|PCRE2_DOTALL|PCRE2_DUPNAMES| \
   PCRE2_EXTENDED|PCRE2_EXTENDED_MORE|PCRE2_MATCH_UNSET_BACKREF| \
   PCRE2_MULTILINE|PCRE2_NEVER_BACKSLASH_C|PCRE2_NEVER_UCP| \
   PCRE2_NEVER_UTF|PCRE2_NO_AUTO_CAPTURE|PCRE2_NO_AUTO_POSSESS| \
   PCRE2_NO_DOTSTAR_ANCHOR|PCRE2_UCP|PCRE2_UNGREEDY)

#define PUBLIC_LITERAL_COMPILE_EXTRA_OPTIONS \
   (PCRE2_EXTRA_MATCH_LINE|PCRE2_EXTRA_MATCH_WORD)

#define PUBLIC_COMPILE_EXTRA_OPTIONS \
   (PUBLIC_LITERAL_COMPILE_EXTRA_OPTIONS| \
    PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES|PCRE2_EXTRA_BAD_ESCAPE_IS_LITERAL| \
    PCRE2_EXTRA_ESCAPED_CR_IS_LF|PCRE2_EXTRA_ALT_BSUX| \
    # 设置 PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK 选项
    PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK
# 编译时错误代码编号。它们被赋予名称，以便更容易跟踪。当添加新的编号时，pcre2posix.c 中称为 eint1 和 eint2 的表可能需要更新，并且必须在 pcre2_error.c 中的 compile_error_texts 中添加新的错误文本。此外，pcre2.h.in 中的错误代码必须更新 - 它们的值比这些值大 100。
enum { ERR0 = COMPILE_ERROR_BASE,
       ERR1,  ERR2,  ERR3,  ERR4,  ERR5,  ERR6,  ERR7,  ERR8,  ERR9,  ERR10,
       ERR11, ERR12, ERR13, ERR14, ERR15, ERR16, ERR17, ERR18, ERR19, ERR20,
       ERR21, ERR22, ERR23, ERR24, ERR25, ERR26, ERR27, ERR28, ERR29, ERR30,
       ERR31, ERR32, ERR33, ERR34, ERR35, ERR36, ERR37, ERR38, ERR39, ERR40,
       ERR41, ERR42, ERR43, ERR44, ERR45, ERR46, ERR47, ERR48, ERR49, ERR50,
       ERR51, ERR52, ERR53, ERR54, ERR55, ERR56, ERR57, ERR58, ERR59, ERR60,
       ERR61, ERR62, ERR63, ERR64, ERR65, ERR66, ERR67, ERR68, ERR69, ERR70,
       ERR71, ERR72, ERR73, ERR74, ERR75, ERR76, ERR77, ERR78, ERR79, ERR80,
       ERR81, ERR82, ERR83, ERR84, ERR85, ERR86, ERR87, ERR88, ERR89, ERR90,
       ERR91, ERR92, ERR93, ERR94, ERR95, ERR96, ERR97, ERR98, ERR99 };

# 这是一个开始模式选项的表，例如 (*UTF) 和设置，例如 (*LIMIT_MATCH=nnnn) 和 (*CRLF)。为了完整性和向后兼容性，在相关库中支持 (*UTFn)，但 (*UTF) 是通用的并始终受支持。
enum { PSO_OPT,     # 值是一个选项位
       PSO_FLG,     # 值是一个标志位
       PSO_NL,      # 值是一个换行符类型
       PSO_BSR,     # 值是一个 \R 类型
       PSO_LIMH,    # 读取堆限制的整数值
       PSO_LIMM,    # 读取匹配限制的整数值
       PSO_LIMD };  # 读取深度限制的整数值

typedef struct pso {
  const uint8_t *name;
  uint16_t length;
  uint16_t type;
  uint32_t value;
} pso;

# 注意：STRING_UTFn_RIGHTPAR 包含长度
# 定义静态的 pso 结构体数组
static pso pso_list[] = {
  # 字符串 UTFn_RIGHTPAR 的指针，选项 PSO_OPT，编码 PCRE2_UTF
  { (uint8_t *)STRING_UTFn_RIGHTPAR,                  PSO_OPT, PCRE2_UTF },
  # 字符串 UTF_RIGHTPAR 的指针，长度 4，选项 PSO_OPT，编码 PCRE2_UTF
  { (uint8_t *)STRING_UTF_RIGHTPAR,                4, PSO_OPT, PCRE2_UTF },
  # 字符串 UCP_RIGHTPAR 的指针，长度 4，选项 PSO_OPT，编码 PCRE2_UCP
  { (uint8_t *)STRING_UCP_RIGHTPAR,                4, PSO_OPT, PCRE2_UCP },
  # 字符串 NOTEMPTY_RIGHTPAR 的指针，长度 9，标志 PSO_FLG，设置 PCRE2_NOTEMPTY_SET
  { (uint8_t *)STRING_NOTEMPTY_RIGHTPAR,           9, PSO_FLG, PCRE2_NOTEMPTY_SET },
  # 字符串 NOTEMPTY_ATSTART_RIGHTPAR 的指针，长度 17，标志 PSO_FLG，设置 PCRE2_NE_ATST_SET
  { (uint8_t *)STRING_NOTEMPTY_ATSTART_RIGHTPAR,  17, PSO_FLG, PCRE2_NE_ATST_SET },
  # 字符串 NO_AUTO_POSSESS_RIGHTPAR 的指针，长度 16，选项 PSO_OPT，设置 PCRE2_NO_AUTO_POSSESS
  { (uint8_t *)STRING_NO_AUTO_POSSESS_RIGHTPAR,   16, PSO_OPT, PCRE2_NO_AUTO_POSSESS },
  # 字符串 NO_DOTSTAR_ANCHOR_RIGHTPAR 的指针，长度 18，选项 PSO_OPT，设置 PCRE2_NO_DOTSTAR_ANCHOR
  { (uint8_t *)STRING_NO_DOTSTAR_ANCHOR_RIGHTPAR, 18, PSO_OPT, PCRE2_NO_DOTSTAR_ANCHOR },
  # 字符串 NO_JIT_RIGHTPAR 的指针，长度 7，标志 PSO_FLG，设置 PCRE2_NOJIT
  { (uint8_t *)STRING_NO_JIT_RIGHTPAR,             7, PSO_FLG, PCRE2_NOJIT },
  # 字符串 NO_START_OPT_RIGHTPAR 的指针，长度 13，选项 PSO_OPT，设置 PCRE2_NO_START_OPTIMIZE
  { (uint8_t *)STRING_NO_START_OPT_RIGHTPAR,      13, PSO_OPT, PCRE2_NO_START_OPTIMIZE },
  # 字符串 LIMIT_HEAP_EQ 的指针，长度 11，限制类型 PSO_LIMH，值为 0
  { (uint8_t *)STRING_LIMIT_HEAP_EQ,              11, PSO_LIMH, 0 },
  # 字符串 LIMIT_MATCH_EQ 的指针，长度 12，限制类型 PSO_LIMM，值为 0
  { (uint8_t *)STRING_LIMIT_MATCH_EQ,             12, PSO_LIMM, 0 },
  # 字符串 LIMIT_DEPTH_EQ 的指针，长度 12，限制类型 PSO_LIMD，值为 0
  { (uint8_t *)STRING_LIMIT_DEPTH_EQ,             12, PSO_LIMD, 0 },
  # 字符串 LIMIT_RECURSION_EQ 的指针，长度 16，限制类型 PSO_LIMD，值为 0
  { (uint8_t *)STRING_LIMIT_RECURSION_EQ,         16, PSO_LIMD, 0 },
  # 字符串 CR_RIGHTPAR 的指针，长度 3，标志 PSO_NL，设置 PCRE2_NEWLINE_CR
  { (uint8_t *)STRING_CR_RIGHTPAR,                 3, PSO_NL,  PCRE2_NEWLINE_CR },
  # 字符串 LF_RIGHTPAR 的指针，长度 3，标志 PSO_NL，设置 PCRE2_NEWLINE_LF
  { (uint8_t *)STRING_LF_RIGHTPAR,                 3, PSO_NL,  PCRE2_NEWLINE_LF },
  # 字符串 CRLF_RIGHTPAR 的指针，长度 5，标志 PSO_NL，设置 PCRE2_NEWLINE_CRLF
  { (uint8_t *)STRING_CRLF_RIGHTPAR,               5, PSO_NL,  PCRE2_NEWLINE_CRLF },
  # 字符串 ANY_RIGHTPAR 的指针，长度 4，标志 PSO_NL，设置 PCRE2_NEWLINE_ANY
  { (uint8_t *)STRING_ANY_RIGHTPAR,                4, PSO_NL,  PCRE2_NEWLINE_ANY },
  # 字符串 NUL_RIGHTPAR 的指针，长度 4，标志 PSO_NL，设置 PCRE2_NEWLINE_NUL
  { (uint8_t *)STRING_NUL_RIGHTPAR,                4, PSO_NL,  PCRE2_NEWLINE_NUL },
  # 字符串 ANYCRLF_RIGHTPAR 的指针，长度 8，标志 PSO_NL，设置 PCRE2_NEWLINE_ANYCRLF
  { (uint8_t *)STRING_ANYCRLF_RIGHTPAR,            8, PSO_NL,  PCRE2_NEWLINE_ANYCRLF },
  # 字符串 BSR_ANYCRLF_RIGHTPAR 的指针，长度 12，标志 PSO_BSR，设置 PCRE2_BSR_ANYCRLF
  { (uint8_t *)STRING_BSR_ANYCRLF_RIGHTPAR,       12, PSO_BSR, PCRE2_BSR_ANYCRLF },
  # 字符串 BSR_UNICODE_RIGHTPAR 的指针，长度 12，标志 PSO_BSR，设置 PCRE2_BSR_UNICODE
  { (uint8_t *)STRING_BSR_UNICODE_RIGHTPAR,       12, PSO_BSR, PCRE2_BSR_UNICODE }
};

/* This table is used when converting repeating opcodes into possessified
versions as a result of an explicit possessive quantifier such as ++. A zero
value means there is no possessified version - in those cases the item in
#ifdef DEBUG_SHOW_PARSED
/*************************************************
*     Show the parsed pattern for debugging      *
*************************************************/
// 用于调试预扫描，启用此代码以输出解析后的数据向量

static void show_parsed(compile_block *cb)
{
// 定义指向解析后模式的指针
uint32_t *pptr = cb->parsed_pattern;

// 无限循环，直到遇到 META_END
for (;;)
  {
  int max, min;
  PCRE2_SIZE offset;
  uint32_t i;
  uint32_t length;
  uint32_t meta_arg = META_DATA(*pptr);

  // 输出解析后的数据向量的偏移和值
  fprintf(stderr, "+++ %02d %.8x ", (int)(pptr - cb->parsed_pattern), *pptr);

  // 如果值小于 META_END
  if (*pptr < META_END)
    {
    // 如果值在可打印字符范围内，则输出字符并移动指针
    if (*pptr > 32 && *pptr < 128) fprintf(stderr, "%c", *pptr);
    pptr++;
    }

  // 如果值大于等于 META_END
  else switch (META_CODE(*pptr++))
    {
    default:
    // 未知的 META 值，输出错误信息并返回
    fprintf(stderr, "**** OOPS - unknown META value - giving up ****\n");
    return;

    case META_END:
    // 输出 META_END 并返回
    fprintf(stderr, "META_END\n");
    return;

    case META_CAPTURE:
    // 输出 META_CAPTURE 和参数值
    fprintf(stderr, "META_CAPTURE %d", meta_arg);
    break;

    case META_RECURSE:
    // 获取偏移值并输出 META_RECURSE 和参数值
    GETOFFSET(offset, pptr);
    fprintf(stderr, "META_RECURSE %d %zd", meta_arg, offset);
    break;

    case META_BACKREF:
    // 如果参数值小于 10，则使用预定义的偏移值，否则获取偏移值并输出 META_BACKREF 和参数值
    if (meta_arg < 10)
      offset = cb->small_ref_offset[meta_arg];
    else
      GETOFFSET(offset, pptr);
    fprintf(stderr, "META_BACKREF %d %zd", meta_arg, offset);
    break;

    case META_ESCAPE:
    // 如果参数值为 ESC_P 或 ESC_p，则获取类型和值并输出 META 和参数值
    if (meta_arg == ESC_P || meta_arg == ESC_p)
      {
      uint32_t ptype = *pptr >> 16;
      uint32_t pvalue = *pptr++ & 0xffff;
      fprintf(stderr, "META \\%c %d %d", (meta_arg == ESC_P)? 'P':'p',
        ptype, pvalue);
      }
    else
      {
      uint32_t cc;
      /* 如果 meta_arg 等于 ESC_g，则将 cc 设置为 CHAR_g */
      if (meta_arg == ESC_g) cc = CHAR_g;
      /* 否则遍历 escapes 表，查找与 meta_arg 相匹配的转义字符 */
      else for (cc = ESCAPES_FIRST; cc <= ESCAPES_LAST; cc++)
        {
        if (meta_arg == (uint32_t)(-escapes[cc - ESCAPES_FIRST])) break;
        }
      /* 如果未找到匹配的转义字符，则将 cc 设置为 CHAR_QUESTION_MARK */
      if (cc > ESCAPES_LAST) cc = CHAR_QUESTION_MARK;
      /* 输出 META 转义字符到标准错误流 */
      fprintf(stderr, "META \\%c", cc);
      }
    break;

    case META_MINMAX:
    /* 读取最小值和最大值 */
    min = *pptr++;
    max = *pptr++;
    /* 如果最大值不是 REPEAT_UNLIMITED，则输出 META {最小值,最大值} 到标准错误流 */
    if (max != REPEAT_UNLIMITED)
      fprintf(stderr, "META {%d,%d}", min, max);
    /* 否则输出 META {最小值,} 到标准错误流 */
    else
      fprintf(stderr, "META {%d,}", min);
    break;

    case META_MINMAX_QUERY:
    /* 读取最小值和最大值 */
    min = *pptr++;
    max = *pptr++;
    /* 如果最大值不是 REPEAT_UNLIMITED，则输出 META {最小值,最大值}? 到标准错误流 */
    if (max != REPEAT_UNLIMITED)
      fprintf(stderr, "META {%d,%d}?", min, max);
    /* 否则输出 META {最小值,}? 到标准错误流 */
    else
      fprintf(stderr, "META {%d,}?", min);
    break;

    case META_MINMAX_PLUS:
    /* 读取最小值和最大值 */
    min = *pptr++;
    max = *pptr++;
    /* 如果最大值不是 REPEAT_UNLIMITED，则输出 META {最小值,最大值}+ 到标准错误流 */
    if (max != REPEAT_UNLIMITED)
      fprintf(stderr, "META {%d,%d}+", min, max);
    /* 否则输出 META {最小值,}+ 到标准错误流 */
    else
      fprintf(stderr, "META {%d,}+", min);
    break;

    case META_BIGVALUE: 
    /* 输出 META_BIGVALUE 和对应的值到标准错误流 */
    fprintf(stderr, "META_BIGVALUE %.8x", *pptr++); 
    break;
    case META_CIRCUMFLEX: 
    /* 输出 META_CIRCUMFLEX 到标准错误流 */
    fprintf(stderr, "META_CIRCUMFLEX"); 
    break;
    case META_COND_ASSERT: 
    /* 输出 META_COND_ASSERT 到标准错误流 */
    fprintf(stderr, "META_COND_ASSERT"); 
    break;
    case META_DOLLAR: 
    /* 输出 META_DOLLAR 到标准错误流 */
    fprintf(stderr, "META_DOLLAR"); 
    break;
    case META_DOT: 
    /* 输出 META_DOT 到标准错误流 */
    fprintf(stderr, "META_DOT"); 
    break;
    case META_ASTERISK: 
    /* 输出 META * 到标准错误流 */
    fprintf(stderr, "META *"); 
    break;
    case META_ASTERISK_QUERY: 
    /* 输出 META *? 到标准错误流 */
    fprintf(stderr, "META *?"); 
    break;
    case META_ASTERISK_PLUS: 
    /* 输出 META *+ 到标准错误流 */
    fprintf(stderr, "META *+"); 
    break;
    case META_PLUS: 
    /* 输出 META + 到标准错误流 */
    fprintf(stderr, "META +"); 
    break;
    case META_PLUS_QUERY: 
    /* 输出 META +? 到标准错误流 */
    fprintf(stderr, "META +?"); 
    break;
    case META_PLUS_PLUS: 
    /* 输出 META ++ 到标准错误流 */
    fprintf(stderr, "META ++"); 
    break;
    case META_QUERY: 
    /* 输出 META ? 到标准错误流 */
    fprintf(stderr, "META ?"); 
    break;
    case META_QUERY_QUERY: 
    /* 输出 META ?? 到标准错误流 */
    fprintf(stderr, "META ??"); 
    break;
    case META_QUERY_PLUS: 
    /* 输出 META ?+ 到标准错误流 */
    fprintf(stderr, "META ?+"); 
    break;
    # 打印 META_ATOMIC 类型的元数据信息
    case META_ATOMIC: fprintf(stderr, "META (?>"); break;
    # 打印 META_NOCAPTURE 类型的元数据信息
    case META_NOCAPTURE: fprintf(stderr, "META (?:"); break;
    # 打印 META_LOOKAHEAD 类型的元数据信息
    case META_LOOKAHEAD: fprintf(stderr, "META (?="); break;
    # 打印 META_LOOKAHEADNOT 类型的元数据信息
    case META_LOOKAHEADNOT: fprintf(stderr, "META (?!"); break;
    # 打印 META_LOOKAHEAD_NA 类型的元数据信息
    case META_LOOKAHEAD_NA: fprintf(stderr, "META (*napla:"); break;
    # 打印 META_SCRIPT_RUN 类型的元数据信息
    case META_SCRIPT_RUN: fprintf(stderr, "META (*sr:"); break;
    # 打印 META_KET 类型的元数据信息
    case META_KET: fprintf(stderr, "META )"); break;
    # 打印 META_ALT 类型的元数据信息
    case META_ALT: fprintf(stderr, "META | %d", meta_arg); break;

    # 打印 META_CLASS 类型的元数据信息
    case META_CLASS: fprintf(stderr, "META ["); break;
    # 打印 META_CLASS_NOT 类型的元数据信息
    case META_CLASS_NOT: fprintf(stderr, "META [^"); break;
    # 打印 META_CLASS_END 类型的元数据信息
    case META_CLASS_END: fprintf(stderr, "META ]"); break;
    # 打印 META_CLASS_EMPTY 类型的元数据信息
    case META_CLASS_EMPTY: fprintf(stderr, "META []"); break;
    # 打印 META_CLASS_EMPTY_NOT 类型的元数据信息
    case META_CLASS_EMPTY_NOT: fprintf(stderr, "META [^]"); break;

    # 打印 META_RANGE_LITERAL 类型的元数据信息
    case META_RANGE_LITERAL: fprintf(stderr, "META - (literal)"); break;
    # 打印 META_RANGE_ESCAPED 类型的元数据信息
    case META_RANGE_ESCAPED: fprintf(stderr, "META - (escaped)"); break;

    # 打印 META_POSIX 类型的元数据信息
    case META_POSIX: fprintf(stderr, "META_POSIX %d", *pptr++); break;
    # 打印 META_POSIX_NEG 类型的元数据信息
    case META_POSIX_NEG: fprintf(stderr, "META_POSIX_NEG %d", *pptr++); break;

    # 打印 META_ACCEPT 类型的元数据信息
    case META_ACCEPT: fprintf(stderr, "META (*ACCEPT)"); break;
    # 打印 META_FAIL 类型的元数据信息
    case META_FAIL: fprintf(stderr, "META (*FAIL)"); break;
    # 打印 META_COMMIT 类型的元数据信息
    case META_COMMIT: fprintf(stderr, "META (*COMMIT)"); break;
    # 打印 META_PRUNE 类型的元数据信息
    case META_PRUNE: fprintf(stderr, "META (*PRUNE)"); break;
    # 打印 META_SKIP 类型的元数据信息
    case META_SKIP: fprintf(stderr, "META (*SKIP)"); break;
    # 打印 META_THEN 类型的元数据信息
    case META_THEN: fprintf(stderr, "META (*THEN)"); break;

    # 打印 META_OPTIONS 类型的元数据信息
    case META_OPTIONS: fprintf(stderr, "META_OPTIONS 0x%02x", *pptr++); break;

    # 打印 META_LOOKBEHIND 类型的元数据信息
    case META_LOOKBEHIND:
    fprintf(stderr, "META (?<= %d offset=", meta_arg);
    GETOFFSET(offset, pptr);
    fprintf(stderr, "%zd", offset);
    break;

    # 打印 META_LOOKBEHIND_NA 类型的元数据信息
    case META_LOOKBEHIND_NA:
    fprintf(stderr, "META (*naplb: %d offset=", meta_arg);
    GETOFFSET(offset, pptr);
    fprintf(stderr, "%zd", offset);
    break;

    # 打印 META_LOOKBEHINDNOT 类型的元数据信息
    case META_LOOKBEHINDNOT:
    fprintf(stderr, "META (?<! %d offset=", meta_arg);
    GETOFFSET(offset, pptr);
    # 输出当前偏移量到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_CALLOUT_NUMBER 情况
    case META_CALLOUT_NUMBER:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?C%d) next=%d/%d", pptr[2], pptr[0],
       pptr[1]);
    # 更新指针位置
    pptr += 3;
    # 结束当前的 switch 语句
    break;

    # 处理 META_CALLOUT_STRING 情况
    case META_CALLOUT_STRING:
      {
      # 读取下一个模式项的偏移量
      uint32_t patoffset = *pptr++;    /* Offset of next pattern item */
      # 读取下一个模式项的长度
      uint32_t patlength = *pptr++;    /* Length of next pattern item */
      # 输出调试信息到标准错误流
      fprintf(stderr, "META (?Cstring) length=%d offset=", *pptr++);
      # 获取偏移量
      GETOFFSET(offset, pptr);
      # 输出调试信息到标准错误流
      fprintf(stderr, "%zd next=%d/%d", offset, patoffset, patlength);
      }
    # 结束当前的 switch 语句
    break;

    # 处理 META_RECURSE_BYNAME 情况
    case META_RECURSE_BYNAME:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(&name) length=%d offset=", *pptr++);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_BACKREF_BYNAME 情况
    case META_BACKREF_BYNAME:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META_BACKREF_BYNAME length=%d offset=", *pptr++);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_NUMBER 情况
    case META_COND_NUMBER:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META_COND_NUMBER %d offset=", pptr[SIZEOFFSET]);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 更新指针位置
    pptr++;
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_DEFINE 情况
    case META_COND_DEFINE:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(DEFINE) offset=");
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_VERSION 情况
    case META_COND_VERSION:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(VERSION%s", (*pptr++ == 0)? "=" : ">=");
    # 输出调试信息到标准错误流
    fprintf(stderr, "%d.", *pptr++);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%d)", *pptr++);
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_NAME 情况
    case META_COND_NAME:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(<name>) length=%d offset=", *pptr++);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_RNAME 情况
    case META_COND_RNAME:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(R&name) length=%d offset=", *pptr++);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 结束当前的 switch 语句
    break;

    # 处理 META_COND_RNUMBER 情况
    case META_COND_RNUMBER:
    # 输出调试信息到标准错误流
    fprintf(stderr, "META (?(Rnumber) length=%d offset=", *pptr++);
    # 获取偏移量
    GETOFFSET(offset, pptr);
    # 输出调试信息到标准错误流
    fprintf(stderr, "%zd", offset);
    # 跳出当前循环或者 switch 语句
    break;

    # 匹配元字符 (*MARK:
    case META_MARK:
    # 输出 META (*MARK: 到标准错误流
    fprintf(stderr, "META (*MARK:");
    # 跳转到 SHOWARG 标签处
    goto SHOWARG;

    # 匹配元字符 (*COMMIT:
    case META_COMMIT_ARG:
    # 输出 META (*COMMIT: 到标准错误流
    fprintf(stderr, "META (*COMMIT:");
    # 跳转到 SHOWARG 标签处
    goto SHOWARG;

    # 匹配元字符 (*PRUNE:
    case META_PRUNE_ARG:
    # 输出 META (*PRUNE: 到标准错误流
    fprintf(stderr, "META (*PRUNE:");
    # 跳转到 SHOWARG 标签处
    goto SHOWARG;

    # 匹配元字符 (*SKIP:
    case META_SKIP_ARG:
    # 输出 META (*SKIP: 到标准错误流
    fprintf(stderr, "META (*SKIP:");
    # 跳转到 SHOWARG 标签处
    goto SHOWARG;

    # 匹配元字符 (*THEN:
    case META_THEN_ARG:
    # 输出 META (*THEN: 到标准错误流
    fprintf(stderr, "META (*THEN:");
    # 标签 SHOWARG：输出长度为 length 的字符或者转义字符
    SHOWARG:
    length = *pptr++;
    for (i = 0; i < length; i++)
      {
      uint32_t cc = *pptr++;
      if (cc > 32 && cc < 128) fprintf(stderr, "%c", cc);
        else fprintf(stderr, "\\x{%x}", cc);
      }
    # 输出长度信息到标准错误流
    fprintf(stderr, ") length=%u", length);
    # 结束当前 case 分支
    break;
    }
  # 输出换行符到标准错误流
  fprintf(stderr, "\n");
  }
# 返回空值
return;
}
#endif  /* DEBUG_SHOW_PARSED */

/*************************************************
*               复制已编译的代码               *
*************************************************/

/* 已编译的 JIT 代码无法被复制，因此新编译的块没有关联的 JIT 数据。 */

PCRE2_EXP_DEFN pcre2_code * PCRE2_CALL_CONVENTION
pcre2_code_copy(const pcre2_code *code)
{
PCRE2_SIZE* ref_count;
pcre2_code *newcode;

if (code == NULL) return NULL;
newcode = code->memctl.malloc(code->blocksize, code->memctl.memory_data);
if (newcode == NULL) return NULL;
memcpy(newcode, code, code->blocksize);
newcode->executable_jit = NULL;

/* 如果代码是已反序列化的代码，则增加解码表中的引用计数。 */

if ((code->flags & PCRE2_DEREF_TABLES) != 0)
  {
  ref_count = (PCRE2_SIZE *)(code->tables + TABLES_LENGTH);
  (*ref_count)++;
  }

return newcode;
}

/*************************************************
*     复制已编译的代码和字符表    *
*************************************************/

/* 已编译的 JIT 代码无法被复制，因此新编译的块没有关联的 JIT 数据。这个版本的 code_copy 还会单独复制字符表。 */

PCRE2_EXP_DEFN pcre2_code * PCRE2_CALL_CONVENTION
pcre2_code_copy_with_tables(const pcre2_code *code)
{
PCRE2_SIZE* ref_count;
pcre2_code *newcode;
uint8_t *newtables;

if (code == NULL) return NULL;
newcode = code->memctl.malloc(code->blocksize, code->memctl.memory_data);
if (newcode == NULL) return NULL;
memcpy(newcode, code, code->blocksize);
newcode->executable_jit = NULL;

newtables = code->memctl.malloc(TABLES_LENGTH + sizeof(PCRE2_SIZE),
  code->memctl.memory_data);
if (newtables == NULL)
  {
  code->memctl.free((void *)newcode, code->memctl.memory_data);
  return NULL;
  }
memcpy(newtables, code->tables, TABLES_LENGTH);
ref_count = (PCRE2_SIZE *)(newtables + TABLES_LENGTH);
*ref_count = 1;

newcode->tables = newtables;
# 设置 newcode 的标志位，使其包含对表的引用
newcode->flags |= PCRE2_DEREF_TABLES;
# 返回新的编译代码
return newcode;
}

# 释放已编译的代码
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_code_free(pcre2_code *code)
{
# 定义引用计数指针
PCRE2_SIZE* ref_count;

# 如果代码不为空
if (code != NULL)
  {
  # 如果支持 JIT 编译
  # 释放 JIT 编译的内存
  if (code->executable_jit != NULL)
    PRIV(jit_free)(code->executable_jit, &code->memctl);
  # 如果标志位包含对表的引用
  if ((code->flags & PCRE2_DEREF_TABLES) != 0)
    {
    # 解码的表属于反序列化后的代码，当没有更多引用时必须释放
    # *ref_count 应始终大于 0
    ref_count = (PCRE2_SIZE *)(code->tables + TABLES_LENGTH);
    if (*ref_count > 0)
      {
      (*ref_count)--;
      if (*ref_count == 0)
        code->memctl.free((void *)code->tables, code->memctl.memory_data);
      }
    }
  # 释放代码的内存
  code->memctl.free(code, code->memctl.memory_data);
  }
}

# 读取一个可能带符号的数字
# 用于在模式中读取数字
# 初始指针必须是数字的符号或第一个数字
# 当允许相对值（由 + 或 - 引入）时，它们是相对于组号的相对值，结果必须大于零
# 参数：
#   ptrptr      指向字符指针变量
#   ptrend      指向输入字符串的末尾
#   allow_sign  如果 < 0，不允许符号；如果 >= 0，符号是相对于此的
#   max_value   允许的最大数字
#   max_error   对于过大的数字给出的错误
#   intptr      存放结果的位置
#   errcodeptr  存放错误代码的位置
# 返回：
#   TRUE  - 读取到一个数字
#   FALSE - errorcode == 0 => 没有找到数字
#           errorcode != 0 => 发生了错误
static BOOL
# 读取一个数字，可以带符号，返回是否成功，并将结果存入指针指向的变量中
def read_number(PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend, int32_t allow_sign,
  uint32_t max_value, uint32_t max_error, int *intptr, int *errorcodeptr)
{
# 初始化符号和数字
int sign = 0;
uint32_t n = 0;
PCRE2_SPTR ptr = *ptrptr;
BOOL yield = FALSE;

# 初始化错误码
*errorcodeptr = 0;

# 检查是否允许带符号，并且指针未到达末尾
if (allow_sign >= 0 && ptr < ptrend)
  {
  # 如果下一个字符是加号，则设置符号为正，减去允许的符号数，指针后移
  if (*ptr == CHAR_PLUS)
    {
    sign = +1;
    max_value -= allow_sign;
    ptr++;
    }
  # 如果下一个字符是减号，则设置符号为负，指针后移
  else if (*ptr == CHAR_MINUS)
    {
    sign = -1;
    ptr++;
    }
  }

# 如果指针已经到达末尾或者下一个字符不是数字，则返回失败
if (ptr >= ptrend || !IS_DIGIT(*ptr)) return FALSE;
# 循环读取数字，计算结果
while (ptr < ptrend && IS_DIGIT(*ptr))
  {
  n = n * 10 + *ptr++ - CHAR_0;
  if (n > max_value)
    {
    *errorcodeptr = max_error;
    goto EXIT;
    }
  }

# 如果允许带符号并且符号不为0
if (allow_sign >= 0 && sign != 0)
  {
  # 如果数字为0，则返回错误码
  if (n == 0)
    {
    *errorcodeptr = ERR26;  /* +0 and -0 are not allowed */
    goto EXIT;
    }
  # 如果符号为正，则加上允许的符号数
  if (sign > 0) n += allow_sign;
  # 如果数字转换为整数后大于允许的符号数，则返回错误码
  else if ((int)n > allow_sign)
    {
    *errorcodeptr = ERR15;  /* Non-existent subpattern */
    goto EXIT;
    }
  # 否则计算结果
  else n = allow_sign + 1 - n;
  }

# 设置成功标志
yield = TRUE;

# 将结果存入指针指向的变量中
EXIT:
*intptr = n;
*ptrptr = ptr;
return yield;
}
# 返回：如果不是重复量词，则返回FALSE，错误码设置为零；如果出现错误，则返回FALSE，错误码设置为非零；如果成功，则返回TRUE，并将指针更新为 '}' 后面
static BOOL
read_repeat_counts(PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend, uint32_t *minp,
  uint32_t *maxp, int *errorcodeptr)
{
PCRE2_SPTR p;
BOOL yield = FALSE;
BOOL had_comma = FALSE;
int32_t min = 0;
int32_t max = REPEAT_UNLIMITED; /* This value is larger than MAX_REPEAT_COUNT */

/* 检查语法 */

*errorcodeptr = 0;
for (p = *ptrptr;; p++)
  {
  uint32_t c;
  if (p >= ptrend) return FALSE;
  c = *p;
  if (IS_DIGIT(c)) continue;
  if (c == CHAR_RIGHT_CURLY_BRACKET) break;
  if (c == CHAR_COMMA)
    {
    if (had_comma) return FALSE;
    had_comma = TRUE;
    }
  else return FALSE;
  }

/* read_number() 的唯一错误是数字太大。 */

p = *ptrptr;
if (!read_number(&p, ptrend, -1, MAX_REPEAT_COUNT, ERR5, &min, errorcodeptr))
  goto EXIT;

if (*p == CHAR_RIGHT_CURLY_BRACKET)
  {
  p++;
  max = min;
  }
else
  {
  if (*(++p) != CHAR_RIGHT_CURLY_BRACKET)
    {
    if (!read_number(&p, ptrend, -1, MAX_REPEAT_COUNT, ERR5, &max,
        errorcodeptr))
      goto EXIT;
    if (max < min)
      {
      *errorcodeptr = ERR4;
      goto EXIT;
      }
    }
  p++;
  }

yield = TRUE;
if (minp != NULL) *minp = (uint32_t)min;
if (maxp != NULL) *maxp = (uint32_t)max;

/* 更新模式指针 */

EXIT:
*ptrptr = p;
return yield;
}



/*************************************************
*            处理转义字符                      *
*************************************************/

/* 当遇到 \ 时调用此函数。它可以返回简单转义（如 \d）的正值，或者对于数据字符，返回 0，并将其放入 chptr 中。对组 n 的反向引用返回负值。在进入时，ptr 指向 \ 后面的字符。退出时，它指向转义序列的最后一个代码单元。
# 这个函数也被 pcre2_substitute() 调用，用于处理替换字符串中的转义序列。
# 在这种情况下，cb 参数为 NULL，并且对于需要进一步处理的转义，只识别定义数据字符的序列。
# isclass 参数不相关；options 参数是编译模式的最终值。

# 参数：
#   ptrptr         指向输入位置指针
#   ptrend         指向输入的结尾
#   chptr          指向返回的数据字符
#   errorcodeptr   指向 errorcode 变量（包含零）
#   options        当前选项位
#   isclass        如果在字符类中为 TRUE
#   cb             编译数据块，如果从 pcre2_substitute() 调用则为 NULL

# 返回值：         零 => 数据字符
#                 正数 => 特殊转义序列
#                 负数 => 数字反向引用
#                 出错时，errorcodeptr 被设置为非零
*/

int
PRIV(check_escape)(PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend, uint32_t *chptr,
  int *errorcodeptr, uint32_t options, uint32_t extra_options, BOOL isclass,
  compile_block *cb)
{
BOOL utf = (options & PCRE2_UTF) != 0;
PCRE2_SPTR ptr = *ptrptr;
uint32_t c, cc;
int escape = 0;
int i;

# 如果反斜杠在字符串的末尾，这是一个错误。
if (ptr >= ptrend)
  {
  *errorcodeptr = ERR1;
  return 0;
  }

GETCHARINCTEST(c, ptr);         /* 获取字符值，增加指针 */
*errorcodeptr = 0;              /* 乐观地假设 */

# 非字母数字字符是文字，所以我们只需将值留在 c 中。初始值测试可以节省对字母数字范围外的代码点的内存查找。
if (c < ESCAPES_FIRST || c > ESCAPES_LAST) {}  /* 明显是文字 */

# 否则，进行表查找。非零值在这里需要很少的处理。正值是类似 \n 这样的文字值。负值是
else if ((i = escapes[c - ESCAPES_FIRST]) != 0)
  {
  // 如果字符对应的 ESC_ 宏的值不为零，则进行处理
  if (i > 0)
    {
    // 如果值大于零，将其转换为 uint32_t 类型
    c = (uint32_t)i;
    // 如果转换后的字符为回车符，并且设置了 PCRE2_EXTRA_ESCAPED_CR_IS_LF 标志，则将其转换为换行符
    if (c == CHAR_CR && (extra_options & PCRE2_EXTRA_ESCAPED_CR_IS_LF) != 0)
      c = CHAR_LF;
    }
  else  /* Negative table entry */
    {
    // 处理负数的 ESC_ 宏
    escape = -i;                    /* Else return a special escape */
    // 如果回调函数不为空，并且 escape 为 ESC_P、ESC_p 或 ESC_X，则设置相应的标志位
    if (cb != NULL && (escape == ESC_P || escape == ESC_p || escape == ESC_X))
      cb->external_flags |= PCRE2_HASBKPORX;   /* Note \P, \p, or \X */

    /* Perl supports \N{name} for character names and \N{U+dddd} for numerical
    Unicode code points, as well as plain \N for "not newline". PCRE does not
    support \N{name}. However, it does support quantification such as \N{2,3},
    so if \N{ is not followed by U+dddd we check for a quantifier. */

    // 如果 escape 为 ESC_N，并且指针 ptr 指向的字符为左花括号，则进行处理
    if (escape == ESC_N && ptr < ptrend && *ptr == CHAR_LEFT_CURLY_BRACKET)
      {
      // 指针 p 指向 ptr 的下一个字符
      PCRE2_SPTR p = ptr + 1;

      /* \N{U+ can be handled by the \x{ code. However, this construction is
      not valid in EBCDIC environments because it specifies a Unicode
      character, not a codepoint in the local code. For example \N{U+0041}
      must be "A" in all environments. Also, in Perl, \N{U+ forces Unicode
      casing semantics for the entire pattern, so allow it only in UTF (i.e.
      Unicode) mode. */

      // 如果 ptrend - p 大于 1，并且 p 指向的字符为 U，p 的下一个字符为 +，则进行处理
      if (ptrend - p > 1 && *p == CHAR_U && p[1] == CHAR_PLUS)
        {
#ifdef EBCDIC
        *errorcodeptr = ERR93;
#else
        // 如果处于 UTF 模式，则将指针 ptr 移动到 p 的下一个字符，并将 escape 设为零
        if (utf)
          {
          ptr = p + 1;
          escape = 0;   /* Not a fancy escape after all */
          goto COME_FROM_NU;
          }
        else *errorcodeptr = ERR93;
#else
        }

      /* 如果接下来的内容不是量词，则报错，但不要覆盖量词读取器设置的错误（例如数字溢出） */
      else
        {
        if (!read_repeat_counts(&p, ptrend, NULL, NULL, errorcodeptr) &&
             *errorcodeptr == 0)
          *errorcodeptr = ERR37;
        }
      }
    }
  }

/* 需要进一步处理的转义字符，包括未知的转义字符，在查找表中有零值的转义字符。当从 pcre2_substitute() 调用时，只识别 \c、\o 和 \x（\u 和 \U 永远不会出现，因为它们用于强制大小写）。 */
else
  {
  int s;
  PCRE2_SPTR oldptr;
  BOOL overflow;
  BOOL alt_bsux =
    ((options & PCRE2_ALT_BSUX) | (extra_options & PCRE2_EXTRA_ALT_BSUX)) != 0;

  /* 过滤来自 pcre2_substitute() 的调用。 */
  if (cb == NULL)
    {
    if (c != CHAR_c && c != CHAR_o && c != CHAR_x)
      {
      *errorcodeptr = ERR3;
      return 0;
      }
    alt_bsux = FALSE;   /* 不修改 \x 处理 */
    }

  switch (c)
    {
    /* 许多 Perl 转义字符不被 PCRE 处理。我们给出明确的错误。 */
    case CHAR_F:
    case CHAR_l:
    case CHAR_L:
    *errorcodeptr = ERR37;
    break;

    /* 当未设置 PCRE2_ALT_BSUX 和 PCRE2_EXTRA_ALT_BSUX 时，\u 无法识别。否则，\u 必须后跟四个十六进制数字，或者如果设置了 PCRE2_EXTRA_ALT_BSUX，则后面可以跟任意数量的大括号中的十六进制数字。否则，它是一个小写的 u 字母。这在一定程度上与 ECMAScript（又名 JavaScript）兼容。 */
    case CHAR_u:
    // 如果 alt_bsux 为假，则将错误代码指针设置为 ERR37，否则执行下面的代码
    if (!alt_bsux) *errorcodeptr = ERR37; else
      {
      // 声明一个无符号 32 位整数 xc
      uint32_t xc;

      // 如果指针超出范围，则跳出循环
      if (ptr >= ptrend) break;
      // 如果当前字符为左花括号，并且额外选项中包含 PCRE2_EXTRA_ALT_BSUX
      if (*ptr == CHAR_LEFT_CURLY_BRACKET &&
          (extra_options & PCRE2_EXTRA_ALT_BSUX) != 0)
        {
        // 将 hptr 指向当前指针位置的下一个字符
        PCRE2_SPTR hptr = ptr + 1;
        // 初始化 cc 为 0
        cc = 0;

        // 当 hptr 未超出范围且当前字符为十六进制数字时执行循环
        while (hptr < ptrend && (xc = XDIGIT(*hptr)) != 0xff)
          {
          // 如果 cc 的高 4 位不为 0，则表示 32 位溢出
          if ((cc & 0xf0000000) != 0)  /* Test for 32-bit overflow */
            {
            *errorcodeptr = ERR77;
            ptr = hptr;   /* 显示位置 */
            break;        /* *hptr != } 将在下面再次跳出循环 */
            }
          // 将 xc 加入到 cc 的低 4 位
          cc = (cc << 4) | xc;
          hptr++;
          }

        // 如果没有十六进制数字、超出范围或者没有右花括号，则跳出循环
        if (hptr == ptr + 1 ||   /* No hex digits */
            hptr >= ptrend ||    /* Hit end of input */
            *hptr != CHAR_RIGHT_CURLY_BRACKET)  /* No } terminator */
          break;         /* Hex escape not recognized */

        // 将 cc 转换为字符并将指针指向右花括号后的位置
        c = cc;          /* Accept the code point */
        ptr = hptr + 1;
        }

      else  /* 必须是恰好 4 个十六进制数字 */
        {
        // 如果剩余字符少于 4 个，则跳出循环
        if (ptrend - ptr < 4) break;               /* Less than 4 chars */
        // 如果第一个字符不是十六进制数字，则跳出循环
        if ((cc = XDIGIT(ptr[0])) == 0xff) break;  /* Not a hex digit */
        // 如果第二个字符不是十六进制数字，则跳出循环
        if ((xc = XDIGIT(ptr[1])) == 0xff) break;  /* Not a hex digit */
        // 将前两个字符组成的十六进制数加入到 cc 的低 8 位
        cc = (cc << 4) | xc;
        // 如果第三个字符不是十六进制数字，则跳出循环
        if ((xc = XDIGIT(ptr[2])) == 0xff) break;  /* Not a hex digit */
        // 将第三个字符转换为十六进制数加入到 cc 的低 8 位
        cc = (cc << 4) | xc;
        // 如果第四个字符不是十六进制数字，则跳出循环
        if ((xc = XDIGIT(ptr[3])) == 0xff) break;  /* Not a hex digit */
        // 将第四个字符转换为十六进制数加入到 cc 的低 8 位
        c = (cc << 4) | xc;
        // 将指针向后移动 4 个位置
        ptr += 4;
        }

      // 如果是 UTF-8 编码
      if (utf)
        {
        // 如果 c 大于 0x10ffff，则将错误代码指针设置为 ERR77
        if (c > 0x10ffffU) *errorcodeptr = ERR77;
        // 如果 c 在代理对范围内，并且额外选项中不包含 PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES，则将错误代码指针设置为 ERR73
        else
          if (c >= 0xd800 && c <= 0xdfff &&
              (extra_options & PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES) == 0)
                *errorcodeptr = ERR73;
        }
      // 如果不是 UTF-8 编码且 c 大于最大非 UTF-8 字符，则将错误代码指针设置为 ERR77
      else if (c > MAX_NON_UTF_CHAR) *errorcodeptr = ERR77;
      }
    // 跳出循环
    break;

    /* \U 未被识别，除非设置了 PCRE2_ALT_BSUX 或 PCRE2_EXTRA_ALT_BSUX，此时它是一个大写字母。 */
    # 如果遇到字符 'U'，并且 alt_bsux 为假，则将错误码 ERR37 赋给 errorcodeptr
    case CHAR_U:
    if (!alt_bsux) *errorcodeptr = ERR37;
    break;

    # 在字符类中，\g 只是一个字面上的 "g"。在字符类外，\g 必须后跟以下特定内容之一：

    # (1) 一个数字，可以是普通的数字或者用大括号括起来的数字。如果是正数，它是一个绝对反向引用。如果是负数，它是一个相对反向引用。这是 Perl 5.10 的特性。

    # (2) Perl 5.10 还支持 \g{name} 作为对命名组的引用。这是 Perl 向统一的反向引用语法迈进的一部分。由于这与 \k{name} 是同义的，我们通过假装它实际上是 \k{name} 来处理。

    # (3) 为了与 Oniguruma 兼容，我们还支持 \g 后跟尖括号或单引号中的名称或数字。然而，这些是（可能是递归的）子例程调用，而不是反向引用。我们返回 ESC_g 代码。

    # 总结：对于数字反向引用返回一个负数，对于命名反向引用返回 ESC_k，对于命名或编号的子例程调用返回 ESC_g。
    */

    # 如果遇到字符 'g' 并且在字符类中，则跳过
    case CHAR_g:
    if (isclass) break;

    # 如果指针超出了 ptrend，则将错误码 ERR57 赋给 errorcodeptr
    if (ptr >= ptrend)
      {
      *errorcodeptr = ERR57;
      break;
      }

    # 如果指针指向 '<' 或者 '\''，则将 escape 赋值为 ESC_g
    if (*ptr == CHAR_LESS_THAN_SIGN || *ptr == CHAR_APOSTROPHE)
      {
      escape = ESC_g;
      break;
      }

    # 如果有大括号分隔符，则尝试读取一个数字引用。如果没有，则假设我们有一个名称，并将其视为 \k。
    if (*ptr == CHAR_LEFT_CURLY_BRACKET)
      {
      PCRE2_SPTR p = ptr + 1;
      if (!read_number(&p, ptrend, cb->bracount, MAX_GROUP_NUMBER, ERR61, &s,
          errorcodeptr))
        {
        if (*errorcodeptr == 0) escape = ESC_k;  /* 没有找到数字 */
        break;
        }
      if (p >= ptrend || *p != CHAR_RIGHT_CURLY_BRACKET)
        {
        *errorcodeptr = ERR57;
        break;
        }
      ptr = p + 1;
      }

    # 读取一个未分隔的数字
    else
      {
      // 如果没有找到数字，设置错误码并跳出循环
      if (!read_number(&ptr, ptrend, cb->bracount, MAX_GROUP_NUMBER, ERR61, &s,
          errorcodeptr))
        {
        // 如果没有找到数字，设置错误码为 ERR57
        if (*errorcodeptr == 0) *errorcodeptr = ERR57;  /* No number found */
        break;
        }
      }

    // 如果 s 小于等于 0，设置错误码并跳出循环
    if (s <= 0)
      {
      *errorcodeptr = ERR15;
      break;
      }

    // 设置转义值为负的 s
    escape = -s;
    break;

    /* 对由一串数字开头的转义序列的处理并不直接。Perl 在多年来有所改变。现在推荐使用 \g{} 表示反向引用和 \o{} 表示八进制，以避免旧语法中的歧义。

    在字符类外部，数字被解释为十进制数。如果数字小于 10，或者之前有相同数量的提取左括号，它就是一个反向引用。否则，最多三个八进制数字被读取以形成一个转义字符代码。因此 \123 很可能是八进制 123（与 \0123 相比，后者是八进制 012 后面跟着字面的 3）。

    在字符类内部，\ 后面跟着一个数字总是表示字面的 8 或 9，或者一个八进制数。 */

    case CHAR_1: case CHAR_2: case CHAR_3: case CHAR_4: case CHAR_5:
    case CHAR_6: case CHAR_7: case CHAR_8: case CHAR_9:
    # 如果不是字符类
    if (!isclass)
      {
      oldptr = ptr;
      ptr--;   /* 回到数字 */

      /* 由于我们知道我们在一个数字上，read_number() 的唯一可能错误是数字太大而无法成为组号。在这种情况下，我们将处理为非组引用。如果我们读取了一个足够小的数字，检查是否为反向引用。

      \1 到 \9 总是反向引用。\8x 和 \9x 也是；\1x 到 \7x 如果之前没有那么多的捕获，则是八进制转义。 */
      if (read_number(&ptr, ptrend, -1, INT_MAX/10 - 1, 0, &s, errorcodeptr) &&
          (s < 10 || oldptr[-1] >= CHAR_8 || s <= (int)cb->bracount))
        {
        if (s > (int)MAX_GROUP_NUMBER) *errorcodeptr = ERR61;
          else escape = -s;     /* 表示反向引用 */
        break;
        }

      ptr = oldptr;      /* 将指针放回原处并继续执行 */
      }

    /* 处理数字跟在 \ 后面时，该数字不是反向引用，或者我们在字符类中。如果第一个数字是 8 或 9，Perl 以前会生成一个二进制零，然后将该数字视为后面的文字。至少在 Perl 5.18 中，这种情况已经改变，不再插入二进制零。 */
    if (c >= CHAR_8) break;

    /* 继续执行 */

    /* \0 总是以八进制数字开头，但我们可能会以一个更大的第一个八进制数字掉入这里。原始代码只使用八进制数的最低有效 8 位（我认为这是早期 Perl 所做的）。现在，在 UTF-8 模式和 16 位模式下，我们允许更大的数字，但不超过 3 个八进制数字。 */
    case CHAR_0:
    c -= CHAR_0;
    while(i++ < 2 && ptr < ptrend && *ptr >= CHAR_0 && *ptr <= CHAR_7)
        c = c * 8 + *ptr++ - CHAR_0;
#if PCRE2_CODE_UNIT_WIDTH == 8
    # 如果 PCRE2_CODE_UNIT_WIDTH 等于 8
    if (!utf && c > 0xff) *errorcodeptr = ERR51;
    # 如果不是 UTF 编码并且字符大于 0xff，则设置错误代码为 ERR51
#endif
    break;
    # 结束当前的 case 分支

    /* \o is a relatively new Perl feature, supporting a more general way of
    specifying character codes in octal. The only supported form is \o{ddd}. */
    # \o 是相对较新的 Perl 特性，支持以八进制方式指定字符编码的更通用方式。唯一支持的形式是 \o{ddd}。

    case CHAR_o:
    # 当遇到字符 o 时执行以下操作
    if (ptr >= ptrend || *ptr++ != CHAR_LEFT_CURLY_BRACKET)
      {
      ptr--;
      *errorcodeptr = ERR55;
      }
    else if (ptr >= ptrend || *ptr == CHAR_RIGHT_CURLY_BRACKET)
      *errorcodeptr = ERR78;
    else
      {
      c = 0;
      overflow = FALSE;
      while (ptr < ptrend && *ptr >= CHAR_0 && *ptr <= CHAR_7)
        {
        cc = *ptr++;
        if (c == 0 && cc == CHAR_0) continue;     /* Leading zeroes */
        # 如果 c 等于 0 并且 cc 等于 0，则继续循环，跳过前导零
#if PCRE2_CODE_UNIT_WIDTH == 32
        if (c >= 0x20000000l) { overflow = TRUE; break; }
#endif
        c = (c << 3) + (cc - CHAR_0);
        # 将 c 左移 3 位，加上 (cc - CHAR_0) 的值
#if PCRE2_CODE_UNIT_WIDTH == 8
        if (c > (utf ? 0x10ffffU : 0xffU)) { overflow = TRUE; break; }
        # 如果 c 大于 (utf ? 0x10ffffU : 0xffU)，则设置溢出标志为 TRUE，并跳出循环
#elif PCRE2_CODE_UNIT_WIDTH == 16
        if (c > (utf ? 0x10ffffU : 0xffffU)) { overflow = TRUE; break; }
        # 如果 c 大于 (utf ? 0x10ffffU : 0xffffU)，则设置溢出标志为 TRUE，并跳出循环
#elif PCRE2_CODE_UNIT_WIDTH == 32
        if (utf && c > 0x10ffffU) { overflow = TRUE; break; }
        # 如果是 UTF 编码并且 c 大于 0x10ffffU，则设置溢出标志为 TRUE，并跳出循环
#endif
        }
      if (overflow)
        {
        while (ptr < ptrend && *ptr >= CHAR_0 && *ptr <= CHAR_7) ptr++;
        *errorcodeptr = ERR34;
        }
      else if (ptr < ptrend && *ptr++ == CHAR_RIGHT_CURLY_BRACKET)
        {
        if (utf && c >= 0xd800 && c <= 0xdfff &&
            (extra_options & PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES) == 0)
          {
          ptr--;
          *errorcodeptr = ERR73;
          }
        }
      else
        {
        ptr--;
        *errorcodeptr = ERR64;
        }
      }
    break;
    # 结束当前的 case 分支

    /* When PCRE2_ALT_BSUX or PCRE2_EXTRA_ALT_BSUX is set, \x must be followed
    by two hexadecimal digits. Otherwise it is a lowercase x letter. */
    # 当设置了 PCRE2_ALT_BSUX 或 PCRE2_EXTRA_ALT_BSUX 时，\x 后必须跟着两个十六进制数字。否则它就是一个小写的 x 字母。

    case CHAR_x:
    # 当遇到字符 x 时执行以下操作
    # 如果 alt_bsux 为真，则执行以下代码块
    if (alt_bsux)
      {
      # 声明变量 xc 为无符号 32 位整数
      uint32_t xc;
      # 如果指针 ptrend 减去 ptr 的结果小于 2，则跳出循环，字符数小于 2
      if (ptrend - ptr < 2) break;               /* Less than 2 characters */
      # 如果 ptr[0] 不是十六进制数字，则跳出循环
      if ((cc = XDIGIT(ptr[0])) == 0xff) break;  /* Not a hex digit */
      # 如果 ptr[1] 不是十六进制数字，则跳出循环
      if ((xc = XDIGIT(ptr[1])) == 0xff) break;  /* Not a hex digit */
      # 计算字符的十六进制值
      c = (cc << 4) | xc;
      # 指针向后移动 2 个位置
      ptr += 2;
      }

    # 否则执行以下代码块
    # 处理 Perl 风格的 \x。\x{ddd} 是一个字符代码，可以在 UTF-8 或非 8 位模式下大于 0xff，但只有 ddd 是十六进制数字。如果不是，则 { 被视为数据字符。然而，Perl 似乎会读取直到第一个非十六进制数字的十六进制数字，并忽略其余部分，因此，例如 \x{zz} 匹配二进制零。这似乎很疯狂，所以 PCRE 现在会报错。
    else
      {
      # 如果指针小于 ptrend 并且指针指向的字符是左花括号
      if (ptr < ptrend && *ptr == CHAR_LEFT_CURLY_BRACKET)
        {
#ifndef EBCDIC
        COME_FROM_NU:
#endif
        // 如果不是 EBCDIC 编码，则执行下面的代码
        if (++ptr >= ptrend || *ptr == CHAR_RIGHT_CURLY_BRACKET)
          {
          // 如果指针超出范围或者当前字符为右花括号，则设置错误码并跳出循环
          *errorcodeptr = ERR78;
          break;
          }
        c = 0;
        overflow = FALSE;

        while (ptr < ptrend && (cc = XDIGIT(*ptr)) != 0xff)
          {
          // 循环读取十六进制数字，直到遇到结束标志或者非法字符
          ptr++;
          if (c == 0 && cc == 0) continue;   /* Leading zeroes */
          // 如果当前字符为0且c为0，则跳过（处理前导零）
#if PCRE2_CODE_UNIT_WIDTH == 32
          if (c >= 0x10000000l) { overflow = TRUE; break; }
#endif
          // 如果 c 大于等于 0x10000000l，则设置溢出标志并跳出循环
          c = (c << 4) | cc;
          if ((utf && c > 0x10ffffU) || (!utf && c > MAX_NON_UTF_CHAR))
            {
            // 如果是 UTF 编码并且 c 大于 0x10ffffU，或者不是 UTF 编码并且 c 大于最大非 UTF 字符，则设置溢出标志并跳出循环
            overflow = TRUE;
            break;
            }
          }

        if (overflow)
          {
          // 如果溢出标志为真，则跳过剩余的十六进制数字并设置错误码
          while (ptr < ptrend && XDIGIT(*ptr) != 0xff) ptr++;
          *errorcodeptr = ERR34;
          }
        else if (ptr < ptrend && *ptr++ == CHAR_RIGHT_CURLY_BRACKET)
          {
          // 如果没有溢出并且当前字符为右花括号，则进行额外的检查
          if (utf && c >= 0xd800 && c <= 0xdfff &&
              (extra_options & PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES) == 0)
            {
            // 如果是 UTF 编码并且 c 在代理对范围内，并且不允许代理对转义，则设置错误码
            ptr--;
            *errorcodeptr = ERR73;
            }
          }

        /* If the sequence of hex digits does not end with '}', give an error.
        We used just to recognize this construct and fall through to the normal
        \x handling, but nowadays Perl gives an error, which seems much more
        sensible, so we do too. */
        // 如果十六进制数字序列不以 '}' 结尾，则设置错误码
        else
          {
          ptr--;
          *errorcodeptr = ERR67;
          }
        }   /* End of \x{} processing */

      /* Read a up to two hex digits after \x */
      // 读取 \x 后的最多两个十六进制数字
      else
        {
        c = 0;
        if (ptr >= ptrend || (cc = XDIGIT(*ptr)) == 0xff) break;  /* Not a hex digit */
        ptr++;
        c = cc;
        if (ptr >= ptrend || (cc = XDIGIT(*ptr)) == 0xff) break;  /* Not a hex digit */
        ptr++;
        c = (c << 4) | cc;
        }     /* End of \xdd handling */
      }       /* End of Perl-style \x handling */
    break;
    /*
    在ASCII和EBCDIC环境中，\c的处理方式是不同的。在ASCII（或Unicode）环境中，如果\c后面的字符不是可打印的ASCII字符，则会报错。否则，如果后面的字符是字母，则将其转换为大写，然后将0x40位取反。结果就是转义的值。

    在EBCDIC环境中，\c的处理方式与perlebcdic文档中的规范兼容。后面的字符必须是字母或少量特殊字符之一。这些字符提供了定义字符值0-31的方法。

    为了在ASCII环境中测试\c的EBCDIC处理方式，需要明确识别'c'的EBCDIC值。
    */
#if defined EBCDIC && 'a' != 0x81
    case 0x83:
#else
    case CHAR_c:
#endif
    // 如果定义了 EBCDIC 并且 'a' 不等于 0x81，则执行下面的语句
    if (ptr >= ptrend)
      {
      // 如果指针超出范围，将错误码设置为 ERR2
      *errorcodeptr = ERR2;
      break;
      }
    // 将指针指向的字符赋值给 c
    c = *ptr;
    // 如果 c 是小写字母，则转换为大写
    if (c >= CHAR_a && c <= CHAR_z) c = UPPER_CASE(c);

    /* Handle \c in an ASCII/Unicode environment. */

#ifndef EBCDIC    /* ASCII/UTF-8 coding */
    // 如果不是 EBCDIC 编码，则处理 ASCII/UTF-8 编码
    if (c < 32 || c > 126)  /* Excludes all non-printable ASCII */
      {
      // 如果 c 不是可打印字符，则将错误码设置为 ERR68
      *errorcodeptr = ERR68;
      break;
      }
    // 对 c 进行异或操作
    c ^= 0x40;

    /* Handle \c in an EBCDIC environment. The special case \c? is converted to
    255 (0xff) or 95 (0x5f) if other characters suggest we are using the
    POSIX-BC encoding. (This is the way Perl indicates that it handles \c?.)
    The other valid sequences correspond to a list of specific characters. */

#else
    // 如果是 EBCDIC 编码，则处理 EBCDIC 环境
    if (c == CHAR_QUESTION_MARK)
      c = ('\\' == 188 && '`' == 74)? 0x5f : 0xff;
    else
      {
      for (i = 0; i < 32; i++)
        {
        if (c == ebcdic_escape_c[i]) break;
        }
      if (i < 32) c = i; else *errorcodeptr = ERR68;
      }
#endif  /* EBCDIC */

    // 指针后移
    ptr++;
    break;

    /* Any other alphanumeric following \ is an error. Perl gives an error only
    if in warning mode, but PCRE doesn't have a warning mode. */

    default:
    // 其他情况下，将错误码设置为 ERR3
    *errorcodeptr = ERR3;
    *ptrptr = ptr - 1;     /* Point to the character at fault */
    return 0;
    }
  }

/* Set the pointer to the next character before returning. */

*ptrptr = ptr;
*chptr = c;
return escape;
}



#ifdef SUPPORT_UNICODE
/*************************************************
*               Handle \P and \p                 *
*************************************************/

/* This function is called after \P or \p has been encountered, provided that
PCRE2 is compiled with support for UTF and Unicode properties. On entry, the
contents of ptrptr are pointing after the P or p. On exit, it is left pointing
after the final code unit of the escape sequence.
# 定义函数 get_ucp，用于获取 Unicode 属性
static BOOL
get_ucp(PCRE2_SPTR *ptrptr, BOOL *negptr, uint16_t *ptypeptr,
  uint16_t *pdataptr, int *errorcodeptr, compile_block *cb)
{
# 定义变量
PCRE2_UCHAR c;
PCRE2_SIZE i, bot, top;
PCRE2_SPTR ptr = *ptrptr;
PCRE2_UCHAR name[50];
PCRE2_UCHAR *vptr = NULL;
uint16_t ptscript = PT_NOTSCRIPT;

# 如果指针超出了模式的末尾，跳转到错误返回
if (ptr >= cb->end_pattern) goto ERROR_RETURN;
c = *ptr++;
*negptr = FALSE;

# 如果字符为左花括号
if (c == CHAR_LEFT_CURLY_BRACKET)
  {
  # 如果指针超出了模式的末尾，跳转到错误返回
  if (ptr >= cb->end_pattern) goto ERROR_RETURN;

  # 如果下一个字符为插入符号
  if (*ptr == CHAR_CIRCUMFLEX_ACCENT)
    {
    *negptr = TRUE;
    ptr++;
    }

  # 遍历获取花括号中的名称
  for (i = 0; i < (int)(sizeof(name) / sizeof(PCRE2_UCHAR)) - 1; i++)
    {
    # 如果指针超出了模式的末尾，跳转到错误返回
    if (ptr >= cb->end_pattern) goto ERROR_RETURN;
    c = *ptr++;
    while (c == '_' || c == '-' || isspace(c))
      {
      # 如果指针超出了模式的末尾，跳转到错误返回
      if (ptr >= cb->end_pattern) goto ERROR_RETURN;
      c = *ptr++;
      }
    if (c == CHAR_NUL) goto ERROR_RETURN;
    if (c == CHAR_RIGHT_CURLY_BRACKET) break;
    name[i] = tolower(c);
    if ((c == ':' || c == '=') && vptr == NULL) vptr = name + i;
    }

  # 如果没有找到右花括号，跳转到错误返回
  if (c != CHAR_RIGHT_CURLY_BRACKET) goto ERROR_RETURN;
  name[i] = 0;
  }

# 如果不是左花括号，且字符为 ASCII 字母
else if (MAX_255(c) && (cb->ctypes[c] & ctype_letter) != 0)
  {
  name[0] = tolower(c);
  name[1] = 0;
  }
else goto ERROR_RETURN;

# 更新指针位置
*ptrptr = ptr;
/* 指定支持的属性，包括 Bidi_Class（别名为 bc）和 Script（别名为 sc），以及 Script_Extensions（别名为 scx），对应的属性名称分别为 "bidi<name>"、script name 和 script name
   由于支持的属性数量较少，目前直接检查属性名称。如果属性数量增加，可以使用排序表和开关语句来更加整洁。

   对于脚本属性，设置一个 PT_xxx 值，以便（1）可以进行区分和（2）可以诊断无效的脚本名称，这些名称恰好是另一个属性的名称。 */

if (vptr != NULL)
  {
  int offset = 0;
  PCRE2_UCHAR sname[8];

  *vptr = 0;   /* 终止属性名称 */
  if (PRIV(strcmp_c8)(name, STRING_bidiclass) == 0 ||
      PRIV(strcmp_c8)(name, STRING_bc) == 0)
    {
    offset = 4;
    sname[0] = CHAR_b;
    sname[1] = CHAR_i;  /* 没有 strcpy_c8 函数 */
    sname[2] = CHAR_d;
    sname[3] = CHAR_i;
    }

  else if (PRIV(strcmp_c8)(name, STRING_script) == 0 ||
           PRIV(strcmp_c8)(name, STRING_sc) == 0)
    ptscript = PT_SC;

  else if (PRIV(strcmp_c8)(name, STRING_scriptextensions) == 0 ||
           PRIV(strcmp_c8)(name, STRING_scx) == 0)
    ptscript = PT_SCX;

  else
    {
    *errorcodeptr = ERR47;
    return FALSE;
    }

  /* 根据需要调整 name[] 中的字符串 */

  memmove(name + offset, vptr + 1, (name + i - vptr)*sizeof(PCRE2_UCHAR));
  if (offset != 0) memmove(name, sname, offset*sizeof(PCRE2_UCHAR));
  }

/* 使用二分查找搜索已识别的属性。 */

bot = 0;
top = PRIV(utt_size);

while (bot < top)
  {
  int r;
  i = (bot + top) >> 1;
  r = PRIV(strcmp_c8)(name, PRIV(utt_names) + PRIV(utt)[i].name_offset);

  /* 当找到匹配的属性时，如果使用 \p{xx:yy} 语法且 xx 是 sc 或 scx，则需要进行一些额外的检查。 */

  if (r == 0)
    {
    *pdataptr = PRIV(utt)[i].value;
    if (vptr == NULL || ptscript == PT_NOTSCRIPT)
      {
      *ptypeptr = PRIV(utt)[i].type;
      return TRUE;
      }
    # 根据 utt 中的 type 属性进行切换
    switch (PRIV(utt)[i].type)
      {
      # 如果 type 为 PT_SC，则将 ptypeptr 指向 PT_SC，并返回 TRUE
      case PT_SC:
      *ptypeptr = PT_SC;
      return TRUE;

      # 如果 type 为 PT_SCX，则将 ptypeptr 指向 ptscript，并返回 TRUE
      case PT_SCX:
      *ptypeptr = ptscript;
      return TRUE;
      }

    # 结束 switch 语句，表示未找到脚本类型
    break;  /* Non-script found */
    }

  # 根据 r 的值更新 bot 或 top 的值
  if (r > 0) bot = i + 1; else top = i;
  }
*errorcodeptr = ERR47;   /* 将错误代码指针指向 ERR47，表示未识别的属性 */
return FALSE;           /* 返回 FALSE */

ERROR_RETURN:            /* 标签，表示错误返回 */
*errorcodeptr = ERR46;   /* 将错误代码指针指向 ERR46 */
*ptrptr = ptr;          /* 将指针指向 ptr */
return FALSE;           /* 返回 FALSE */
}
#endif                    /* 结束 if 语句 */


/*************************************************
*           检查 POSIX 类语法                     *
*************************************************/

/* 当在字符类中遇到序列 "[:" 或 "[." 或 "[=" 时，调用此函数。
   它检查这些序列后面是否跟着一系列字符，以匹配的 ":]" 或 ".]" 或 "=]" 结束。
   如果我们遇到未转义的 ']' 而没有特殊的前导字符，则返回 FALSE。

   最初，此函数只识别终结符之间的一系列字母，但似乎 Perl 识别任何字符序列，
   尽管当然未知的 POSIX 名称随后被拒绝。例如，Perl 对于 [:f\oo:] 给出了
   "Unknown POSIX class" 错误，而以前 PCRE 不认为这是一个 POSIX 类。
   同样对于 [:1234:]。

   尝试与 Perl 完全一样的问题在于转义的处理。我们必须确保 [abc[:x\]pqr] *不*
   被视为包含 POSIX 类，但 [abc[:x\]pqr:]] 是（以便生成错误）。下面的代码
   处理了特殊情况 \\ 和 \]，但不尝试进行任何其他转义处理。这使得它与 Perl
   在诸如 [:l\ower:] 这样的情况下不同，Perl 将其识别为 POSIX 类 "lower"，
   但 PCRE 不识别 "l\ower"。我认为这比当 Perl 诊断坏类时 PCRE 不诊断更小的
   罪恶。

   一个用户指出 PCRE 拒绝了 [:a[:digit:]] 而 Perl 没有。似乎嵌套 POSIX 类
   的出现取代了外部类的外观。例如，[:a[:digit:]b:] 匹配 "a"、"b"、":" 或
   数字。这通过在遇到具有相同终结符的新组的开头时返回 FALSE 来处理，因为
   下一个关闭序列必须关闭嵌套组，而不是外部组。
/*
In Perl, unescaped square brackets may also appear as part of class names. For
example, [:a[:abc]b:] gives unknown POSIX class "[:abc]b:]". However, for
[:a[:abc]b][b:] it gives unknown POSIX class "[:abc]b][b:]", which does not
seem right at all. PCRE does not allow closing square brackets in POSIX class
names.
*/

/*
Check the syntax of a POSIX class

Arguments:
  ptr      pointer to the character after the initial [ (colon, dot, equals)
  ptrend   pointer to the end of the pattern
  endptr   where to return a pointer to the terminating ':', '.', or '='

Returns:   TRUE or FALSE
*/
static BOOL
check_posix_syntax(PCRE2_SPTR ptr, PCRE2_SPTR ptrend, PCRE2_SPTR *endptr)
{
    PCRE2_UCHAR terminator;  /* Don't combine these lines; the Solaris cc */
    terminator = *ptr++;     /* compiler warns about "non-constant" initializer. */

    for (; ptrend - ptr >= 2; ptr++)
    {
        if (*ptr == CHAR_BACKSLASH &&
            (ptr[1] == CHAR_RIGHT_SQUARE_BRACKET || ptr[1] == CHAR_BACKSLASH))
            ptr++;
        else if ((*ptr == CHAR_LEFT_SQUARE_BRACKET && ptr[1] == terminator) ||
                 *ptr == CHAR_RIGHT_SQUARE_BRACKET) return FALSE;
        else if (*ptr == terminator && ptr[1] == CHAR_RIGHT_SQUARE_BRACKET)
        {
            *endptr = ptr;
            return TRUE;
        }
    }

    return FALSE;
}

/*************************************************
*          Check POSIX class name                *
*************************************************/

/* This function is called to check the name given in a POSIX-style class entry
such as [:alnum:].

Arguments:
  ptr        points to the first letter
  len        the length of the name

Returns:     a value representing the name, or -1 if unknown
*/
static int
check_posix_name(PCRE2_SPTR ptr, int len)
{
    const char *pn = posix_names;
    int yield = 0;
    while (posix_name_lengths[yield] != 0)
    {
        if (len == posix_name_lengths[yield] &&
            PRIV(strncmp_c8)(ptr, pn, (unsigned int)len) == 0) return yield;
        pn += posix_name_lengths[yield] + 1;
        yield++;
    }
    return -1;
}
# 从给定的指针位置读取子模式或 VERB 名称
# 当需要读取子模式或 (*VERB) 或 (*alpha_assertion) 的名称时，parse_regex() 函数会调用此函数
# 初始指针必须指向名称之前的字符。如果该字符是 '*'，则表示正在读取一个 VERB 或 alpha assertion 名称
# 指针将被更新，指向名称之后的位置，对于 VERB 或 alpha assertion 名称，或者名称终止符之后的位置，对于子模式名称
# 返回偏移量和名称指针是冗余信息，但有些调用者使用其中一个，有些使用另一个，所以最简单的方法就是都返回

# 参数：
#   ptrptr       指向字符指针变量
#   ptrend       指向输入字符串的末尾
#   utf          如果输入是 UTF 编码，则为真
#   terminator   子模式名称的终止符必须是这个字符
#   offsetptr    用于存放从模式开始的偏移量
#   nameptr      用于存放输入名称的指针
#   namelenptr   用于存放名称的长度
#   errcodeptr   用于存放错误代码
#   cb           指向编译数据块的指针

# 返回值：
#   如果读取了名称，则返回 TRUE
#   否则返回 FALSE，并设置错误代码
static BOOL
read_name(PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend, BOOL utf, uint32_t terminator,
  PCRE2_SIZE *offsetptr, PCRE2_SPTR *nameptr, uint32_t *namelenptr,
  int *errorcodeptr, compile_block *cb)
{
    PCRE2_SPTR ptr = *ptrptr;
    BOOL is_group = (*ptr != CHAR_ASTERISK);

    if (++ptr >= ptrend)               # 没有字符在名称中
    {
        *errorcodeptr = is_group? ERR62: # 预期子模式名称
                            ERR60;       # 未识别或格式错误的 VERB
        goto FAILED;
    }

    *nameptr = ptr;
    *offsetptr = (PCRE2_SIZE)(ptr - cb->start_pattern);

    # 在 UTF 模式下，组名称可以包含字母和十进制数字，如下定义
#ifdef SUPPORT_UNICODE
// 如果支持 Unicode，则进行以下操作
if (utf && is_group)
  {
  // 定义变量 c 和 type
  uint32_t c, type;
  // 从指针位置获取字符并赋值给 c，获取字符类型并赋值给 type
  GETCHAR(c, ptr);
  type = UCD_CHARTYPE(c);
  // 如果字符类型是数字，则设置错误码并跳转到失败标签
  if (type == ucp_Nd)
    {
    *errorcodeptr = ERR44;
    goto FAILED;
    }
  // 循环处理字符
  for(;;)
    {
    // 如果字符类型不是数字且不是字母且不是下划线，则跳出循环
    if (type != ucp_Nd && PRIV(ucp_gentype)[type] != ucp_L &&
        c != CHAR_UNDERSCORE) break;
    // 指针向前移动一个字符
    ptr++;
    // 检查指针是否超出范围
    FORWARDCHARTEST(ptr, ptrend);
    if (ptr >= ptrend) break;
    // 从指针位置获取字符并赋值给 c，获取字符类型并赋值给 type
    GETCHAR(c, ptr);
    type = UCD_CHARTYPE(c);
    }
  }
else
#else
(void)utf;  /* 避免编译器警告 */
#endif      /* SUPPORT_UNICODE */

/* 处理非组名和非 UTF 模式下的组名。组名不能以数字开头。如果其他名称以数字开头，则不会被识别。 */
  {
  // 如果是组名且以数字开头，则设置错误码并跳转到失败标签
  if (is_group && IS_DIGIT(*ptr))
    {
    *errorcodeptr = ERR44;
    goto FAILED;
    }
  // 循环处理字符
  while (ptr < ptrend && MAX_255(*ptr) && (cb->ctypes[*ptr] & ctype_word) != 0)
    {
    // 指针向前移动一个字符
    ptr++;
    }
  }

/* 检查名称长度 */
// 如果指针位置超出最大名称长度，则设置错误码并跳转到失败标签
if (ptr > *nameptr + MAX_NAME_SIZE)
  {
  *errorcodeptr = ERR48;
  goto FAILED;
  }
*namelenptr = (uint32_t)(ptr - *nameptr);

/* 子模式名称不能为空，并在此处检查其终止符。（动词或字母断言名称后面的内容将单独检查） */
// 如果是组名
if (is_group)
  {
  // 如果指针位置等于名称指针位置，则设置错误码并跳转到失败标签
  if (ptr == *nameptr)
    {
    *errorcodeptr = ERR62;   /* 预期子模式名称 */
    goto FAILED;
    }
  // 如果指针位置超出范围或者指针位置的字符不等于终止符，则设置错误码并跳转到失败标签
  if (ptr >= ptrend || *ptr != (PCRE2_UCHAR)terminator)
    {
    *errorcodeptr = ERR42;
    goto FAILED;
    }
  // 指针向前移动一个字符
  ptr++;
  }

*ptrptr = ptr;
return TRUE;

FAILED:
*ptrptr = ptr;
return FALSE;
}



/*************************************************
*          管理循环开始时的调用                   *
*************************************************/

/* 在 parse_regex() 中的新项开始时，我们能够记录前一个项的细节，并且设置自动调用（如果启用）。避免两个相邻的自动调用，
# 管理调用，更新调用指针和解析模式指针
static uint32_t *
manage_callouts(PCRE2_SPTR ptr, uint32_t **pcalloutptr, BOOL auto_callout,
  uint32_t *parsed_pattern, compile_block *cb)
{
# 保存前一个调用指针
uint32_t *previous_callout = *pcalloutptr;

# 如果前一个调用指针不为空，则更新其偏移量
if (previous_callout != NULL) previous_callout[2] = (uint32_t)(ptr -
  cb->start_pattern - (PCRE2_SIZE)previous_callout[1]);

# 如果自动调用开启，更新前一个调用指针
if (!auto_callout) previous_callout = NULL; else
  {
  # 如果前一个调用指针为空，或者不等于解析模式指针减4，或者第四个元素不等于255
  if (previous_callout == NULL ||
      previous_callout != parsed_pattern - 4 ||
      previous_callout[3] != 255)
    {
    # 设置新的自动调用
    previous_callout = parsed_pattern;
    parsed_pattern += 4;
    previous_callout[0] = META_CALLOUT_NUMBER;
    previous_callout[2] = 0;
    previous_callout[3] = 255;
    }
  previous_callout[1] = (uint32_t)(ptr - cb->start_pattern);
  }

# 更新调用指针
*pcalloutptr = previous_callout;
# 返回解析模式指针
return parsed_pattern;
}



/*************************************************
*      Parse regex and identify named groups     *
*************************************************/

/* This function is called first of all. It scans the pattern and does two
things: (1) It identifies capturing groups and makes a table of named capturing
groups so that information about them is fully available to both the compiling
scans. (2) It writes a parsed version of the pattern with comments omitted and
escapes processed into the parsed_pattern vector.
/* 定义函数 parse_regex，用于解析正则表达式的模式 */
static int parse_regex(PCRE2_SPTR ptr, uint32_t options, BOOL *has_lookbehind, compile_block *cb)
{
    uint32_t c;  /* 用于存储当前字符 */
    uint32_t delimiter;  /* 用于存储定界符 */
    uint32_t namelen;  /* 用于存储名称长度 */
    uint32_t class_range_state;  /* 用于存储字符类范围状态 */
    uint32_t *verblengthptr = NULL;  /* 用于存储修饰符长度指针，避免编译器警告 */
# 定义指向无符号32位整数的指针，初始值为空
uint32_t *verbstartptr = NULL;
# 定义指向无符号32位整数的指针，初始值为空
uint32_t *previous_callout = NULL;
# 定义指向 cb 结构体中 parsed_pattern 成员的指针
uint32_t *parsed_pattern = cb->parsed_pattern;
# 定义指向 cb 结构体中 parsed_pattern_end 成员的指针
uint32_t *parsed_pattern_end = cb->parsed_pattern_end;
# 定义无符号32位整数变量 meta_quantifier，初始值为0
uint32_t meta_quantifier = 0;
# 定义无符号32位整数变量 add_after_mark，初始值为0
uint32_t add_after_mark = 0;
# 定义无符号32位整数变量 extra_options，赋值为 cb 结构体中 cx 成员的 extra_options
uint32_t extra_options = cb->cx->extra_options;
# 定义无符号16位整数变量 nest_depth，初始值为0
uint16_t nest_depth = 0;
# 定义整数变量 after_manual_callout，初始值为0
int after_manual_callout = 0;
# 定义整数变量 expect_cond_assert，初始值为0
int expect_cond_assert = 0;
# 定义整数变量 errorcode，初始值为0
int errorcode = 0;
# 定义整数变量 escape
int escape;
# 定义整数变量 i
int i;
# 定义布尔值变量 inescq，初始值为 FALSE
BOOL inescq = FALSE;
# 定义布尔值变量 inverbname，初始值为 FALSE
BOOL inverbname = FALSE;
# 定义布尔值变量 utf，根据 options 中是否包含 PCRE2_UTF 来赋值
BOOL utf = (options & PCRE2_UTF) != 0;
# 定义布尔值变量 auto_callout，根据 options 中是否包含 PCRE2_AUTO_CALLOUT 来赋值
BOOL auto_callout = (options & PCRE2_AUTO_CALLOUT) != 0;
# 定义布尔值变量 isdupname
BOOL isdupname;
# 定义布尔值变量 negate_class
BOOL negate_class;
# 定义布尔值变量 okquantifier，初始值为 FALSE
BOOL okquantifier = FALSE;
# 定义指向 PCRE2_SPTR 类型的指针 thisptr
PCRE2_SPTR thisptr;
# 定义指向 PCRE2_SPTR 类型的指针 name
PCRE2_SPTR name;
# 定义指向 PCRE2_SPTR 类型的指针 ptrend，赋值为 cb 结构体中 end_pattern 成员
PCRE2_SPTR ptrend = cb->end_pattern;
# 定义指向 PCRE2_SPTR 类型的指针 verbnamestart，初始值为 NULL，用于避免编译器警告
PCRE2_SPTR verbnamestart = NULL;
# 定义指向 named_group 结构体的指针 ng
named_group *ng;
# 定义指向 nest_save 结构体的指针 top_nest，初始值为 NULL
nest_save *top_nest, *end_nests;

# 为了支持 pcre2grep，插入用于单词和行匹配的前导项
if ((extra_options & PCRE2_EXTRA_MATCH_LINE) != 0)
  {
  # 如果 extra_options 中包含 PCRE2_EXTRA_MATCH_LINE，则在 parsed_pattern 中插入 ^ 和 (?: 的元字符
  *parsed_pattern++ = META_CIRCUMFLEX;
  *parsed_pattern++ = META_NOCAPTURE;
  }
else if ((extra_options & PCRE2_EXTRA_MATCH_WORD) != 0)
  {
  # 如果 extra_options 中包含 PCRE2_EXTRA_MATCH_WORD，则在 parsed_pattern 中插入 \b 和 (?: 的元字符
  *parsed_pattern++ = META_ESCAPE + ESC_b;
  *parsed_pattern++ = META_NOCAPTURE;
  }

# 如果模式实际上是一个字面字符串，单独处理以避免干扰主循环
if ((options & PCRE2_LITERAL) != 0)
  {
  while (ptr < ptrend)
    {
    if (parsed_pattern >= parsed_pattern_end)
      {
      # 如果 parsed_pattern 已经超出了范围，则设置错误码为 ERR63，并跳转到 FAILED 标签处
      errorcode = ERR63;  /* Internal error (parsed pattern overflow) */
      goto FAILED;
      }
    thisptr = ptr;
    GETCHARINCTEST(c, ptr);
    if (auto_callout)
      # 如果 auto_callout 为真，则调用 manage_callouts 函数处理调用
      parsed_pattern = manage_callouts(thisptr, &previous_callout,
        auto_callout, parsed_pattern, cb);
    # 将字符 c 加入到 parsed_pattern 中
    PARSED_LITERAL(c, parsed_pattern);
    }
  # 跳转到 PARSED_END 标签处
  goto PARSED_END;
  }

# 处理可能包含元字符的真实正则表达式
top_nest = NULL;
end_nests = (nest_save *)(cb->start_workspace + cb->workspace_size);

# nest_save 结构体的大小可能不是 cb 结构体的大小的因子
/* 因此，我们必须将 end_nests 四舍五入，以便正确避免创建一个跨越工作区末尾的 nest_save。 */

end_nests = (nest_save *)((char *)end_nests -
  ((cb->workspace_size * sizeof(PCRE2_UCHAR)) % sizeof(nest_save)));

/* PCRE2_EXTENDED_MORE 暗示 PCRE2_EXTENDED */

if ((options & PCRE2_EXTENDED_MORE) != 0) options |= PCRE2_EXTENDED;

/* 现在扫描模式 */

while (ptr < ptrend)
  {
  int prev_expect_cond_assert;
  uint32_t min_repeat = 0, max_repeat = 0;
  uint32_t set, unset, *optset;
  uint32_t terminator;
  uint32_t prev_meta_quantifier;
  BOOL prev_okquantifier;
  PCRE2_SPTR tempptr;
  PCRE2_SIZE offset;

  if (parsed_pattern >= parsed_pattern_end)
    {
    errorcode = ERR63;  /* 内部错误（解析模式溢出） */
    goto FAILED;
    }

  if (nest_depth > cb->cx->parens_nest_limit)
    {
    errorcode = ERR19;
    goto FAILED;        /* 括号嵌套太深 */
    }

  /* 获取下一个输入字符，保存其位置以便处理调用 */

  thisptr = ptr;
  GETCHARINCTEST(c, ptr);

  /* 复制引用的文字直到 \E，允许自动调用，除非处理 (*VERB) "name"。 */

  if (inescq)
    {
    if (c == CHAR_BACKSLASH && ptr < ptrend && *ptr == CHAR_E)
      {
      inescq = FALSE;
      ptr++;   /* 跳过 E */
      }
    else
      {
      if (expect_cond_assert > 0)   /* 如果我们期望条件断言，则不允许文字 */
        {                           /* 但空的 \Q\E 序列是可以的。 */
        ptr--;
        errorcode = ERR28;
        goto FAILED;
        }
      if (inverbname)
        {                          /* 不使用 PARSED_LITERAL() 因为它设置 okquantifier。 */
#if PCRE2_CODE_UNIT_WIDTH == 32    
        if (c >= META_END) *parsed_pattern++ = META_BIGVALUE;
#endif
        # 如果不是特殊字符，将当前字符添加到解析后的模式中
        *parsed_pattern++ = c;
        }
      else
        {
        # 如果是特殊字符，检查是否需要处理调用
        if (after_manual_callout-- <= 0)
          parsed_pattern = manage_callouts(thisptr, &previous_callout,
            auto_callout, parsed_pattern, cb);
        # 将当前字符添加到解析后的模式中
        PARSED_LITERAL(c, parsed_pattern);
        }
      # 重置量词标志
      meta_quantifier = 0;
      }
    # 继续处理下一个字符
    continue;  /* Next character */
    }

  # 如果正在处理(*VERB:NAME)项的“name”部分，则所有字符都是文字，直到闭括号为止，除非设置了PCRE2_ALT_VERBNAMES。这会导致反斜杠解释，但只允许使用\Q和\E以及转义字符（不允许字符类型，如\d）。如果还设置了PCRE2_EXTENDED，则必须忽略空白和#注释。通过不进入特殊的(*VERB:NAME)处理来实现这一点-然后它们将在下面被捕获。注意，c是一个字符，而不是代码单元，因此我们不能使用MAX_255来测试其大小，因为MAX_255测试代码单元，并且在8位模式下假定为TRUE。

  if (inverbname &&
       (
        # 要么：不设置两个选项
        ((options & (PCRE2_EXTENDED | PCRE2_ALT_VERBNAMES)) !=
                    (PCRE2_EXTENDED | PCRE2_ALT_VERBNAMES)) ||
#ifdef SUPPORT_UNICODE
        # 或：字符> 255且不是Unicode Pattern White Space
        (c > 255 && (c|1) != 0x200f && (c|1) != 0x2029) ||
#endif
        # 或：不是#注释或isspace()空白
        (c < 256 && c != CHAR_NUMBER_SIGN && (cb->ctypes[c] & ctype_space) == 0
#ifdef SUPPORT_UNICODE
        # 并且在支持Unicode时不是CHAR_NEL
          && c != CHAR_NEL
#endif
       )))
    {
    # 记录(*VERB:NAME)的长度
    PCRE2_SIZE verbnamelength;

    switch(c)
      {
      default:                     # 不要使用PARSED_LITERAL()，因为它设置okquantifier。
#if PCRE2_CODE_UNIT_WIDTH == 32    # 设置大值元数据
      if (c >= META_END) *parsed_pattern++ = META_BIGVALUE;
#else
      *parsed_pattern++ = c;  # 将当前字符添加到解析后的模式中
      break;

      case CHAR_RIGHT_PARENTHESIS:  # 如果遇到右括号
      inverbname = FALSE;  # 将 inverbname 标记为假
      /* This is the length in characters */
      verbnamelength = (PCRE2_SIZE)(parsed_pattern - verblengthptr - 1);  # 计算名称的长度
      /* But the limit on the length is in code units */
      if (ptr - verbnamestart - 1 > (int)MAX_MARK)  # 如果长度超过最大限制
        {
        ptr--;  # 指针回退一步
        errorcode = ERR76;  # 设置错误代码
        goto FAILED;  # 跳转到失败的标签
        }
      *verblengthptr = (uint32_t)verbnamelength;  # 将名称长度写入 verblengthptr

      /* If this name was on a verb such as (*ACCEPT) which does not continue,
      a (*MARK) was generated for the name. We now add the original verb as the
      next item. */

      if (add_after_mark != 0)  # 如果 add_after_mark 不为0
        {
        *parsed_pattern++ = add_after_mark;  # 将 add_after_mark 添加到解析后的模式中
        add_after_mark = 0;  # 将 add_after_mark 置为0
        }
      break;

      case CHAR_BACKSLASH:  # 如果遇到反斜杠
      if ((options & PCRE2_ALT_VERBNAMES) != 0)  # 如果选项中包含 PCRE2_ALT_VERBNAMES
        {
        escape = PRIV(check_escape)(&ptr, ptrend, &c, &errorcode, options,
          cb->cx->extra_options, FALSE, cb);  # 检查转义序列
        if (errorcode != 0) goto FAILED;  # 如果有错误，跳转到失败的标签
        }
      else escape = 0;   /* Treat all as literal */  # 否则将 escape 置为0，表示将所有字符视为字面量

      switch(escape)  # 根据 escape 的值进行切换
        {
        case 0:                    /* Don't use PARSED_LITERAL() because it */
#if PCRE2_CODE_UNIT_WIDTH == 32    /* sets okquantifier. */
        if (c >= META_END) *parsed_pattern++ = META_BIGVALUE;  # 如果字符大于等于 META_END，将 META_BIGVALUE 添加到解析后的模式中
#endif
        *parsed_pattern++ = c;  # 将当前字符添加到解析后的模式中
        break;

        case ESC_Q:
        inescq = TRUE;  # 将 inescq 标记为真
        break;

        case ESC_E:           /* Ignore */
        break;

        default:
        errorcode = ERR40;    /* Invalid in verb name */  # 设置错误代码为无效的动词名称
        goto FAILED;  # 跳转到失败的标签
        }
      }
    continue;   /* Next character in pattern */  # 继续处理模式中的下一个字符
    }

  /* Not a verb name character. At this point we must process everything that
  must not change the quantification state. This is mainly comments, but we
  handle \Q and \E here as well, so that an item such as A\Q\E+ is treated as
  A+, as in Perl. An isolated \E is ignored. */

  if (c == CHAR_BACKSLASH && ptr < ptrend)  # 如果遇到反斜杠并且指针未到达模式的末尾
    {
    # 如果指针指向的字符是 'Q' 或 'E'，则进入条件判断
    if (*ptr == CHAR_Q || *ptr == CHAR_E)
      {
      # 如果指针指向的字符是 'Q'，则设置 inescq 为 True，否则设置为 False
      inescq = *ptr == CHAR_Q;
      # 指针向后移动一位
      ptr++;
      # 继续下一次循环
      continue;
      }
    }

  /* 在扩展模式下跳过空白和 # 注释。注意，c 是一个字符，而不是一个代码单元，所以我们不能使用 MAX_255 来测试其大小，因为 MAX_255 用于测试代码单元，并且在 8 位模式下被假定为 TRUE。空白字符是 Unicode 中指定的“模式空白”，即 isspace() 字符加上 CHAR_NEL（换行符），它在 Unicode 中是 U+0085，再加上 U+200E、U+200F、U+2028 和 U+2029。这些是与 \h 和 \v 匹配的空格字符的子集。 */
  if ((options & PCRE2_EXTENDED) != 0)
    {
    # 如果 c 小于 256 并且 cb->ctypes[c] 的 ctype_space 位被设置，则继续下一次循环
    if (c < 256 && (cb->ctypes[c] & ctype_space) != 0) continue;
#ifdef SUPPORT_UNICODE
    # 如果字符是NEL或者(c|1)等于0x200f或者0x2029，则跳过当前循环
    if (c == CHAR_NEL || (c|1) == 0x200f || (c|1) == 0x2029) continue;
#endif
    # 如果字符是#号
    if (c == CHAR_NUMBER_SIGN)
      {
      # 循环直到指针ptr指向的位置小于ptrend
      while (ptr < ptrend)
        {
        # 如果是换行符，根据换行符的长度移动指针ptr，并跳出循环
        if (IS_NEWLINE(ptr))      /* For non-fixed-length newline cases, */
          {                       /* IS_NEWLINE sets cb->nllen. */
          ptr += cb->nllen;
          break;
          }
        # 向前移动指针ptr
        ptr++;
#ifdef SUPPORT_UNICODE
        # 如果启用了utf，则测试并向前移动指针ptr
        if (utf) FORWARDCHARTEST(ptr, ptrend);
#endif
        }
      # 继续下一次循环
      continue;  /* Next character in pattern */
      }
    }

  /* Skip over bracketed comments */

  # 如果字符是(并且剩余长度大于等于2，并且下一个字符是?号并且下下个字符是#号
  if (c == CHAR_LEFT_PARENTHESIS && ptrend - ptr >= 2 &&
      ptr[0] == CHAR_QUESTION_MARK && ptr[1] == CHAR_NUMBER_SIGN)
    {
    # 循环直到指针ptr指向的位置大于等于ptrend或者指针ptr指向的位置的字符是)
    while (++ptr < ptrend && *ptr != CHAR_RIGHT_PARENTHESIS);
    # 如果指针ptr大于等于ptrend
    if (ptr >= ptrend)
      {
      errorcode = ERR18;  /* A special error for missing ) in a comment */
      goto FAILED;        /* to make it easier to debug. */
      }
    # 向前移动指针ptr
    ptr++;
    # 继续下一次循环
    continue;  /* Next character in pattern */
    }

  /* If the next item is not a quantifier, fill in length of any previous
  callout and create an auto callout if required. */

  # 如果下一个字符不是*、+、?或者{，或者是{但是读取重复次数失败
  if (c != CHAR_ASTERISK && c != CHAR_PLUS && c != CHAR_QUESTION_MARK &&
       (c != CHAR_LEFT_CURLY_BRACKET ||
         (tempptr = ptr,
         !read_repeat_counts(&tempptr, ptrend, NULL, NULL, &errorcode))))
    {
    # 如果after_manual_callout减到0以下
    if (after_manual_callout-- <= 0)
      # 管理调用，并在需要时创建自动调用
      parsed_pattern = manage_callouts(thisptr, &previous_callout, auto_callout,
        parsed_pattern, cb);
    }

  /* 如果 expect_cond_assert 为 2，表示刚刚通过了 (?(，现在期望一个断言，可能之前有一个调用。如果值为 1，表示刚刚有了调用，现在期望一个断言。在所有情况下，至少还需要 3 个字符。当 expect_cond_assert 为 2 时，我们知道当前字符是一个左括号，否则我们就不会在这里。然而，当它为 1 时，我们需要检查，最简单的方法就是总是检查。注意 expect_cond_assert 可能是负数，因为所有调用都会将其减小。 */

  if (expect_cond_assert > 0)
    {
    BOOL ok = c == CHAR_LEFT_PARENTHESIS && ptrend - ptr >= 3 &&
              (ptr[0] == CHAR_QUESTION_MARK || ptr[0] == CHAR_ASTERISK);
    if (ok)
      {
      if (ptr[0] == CHAR_ASTERISK)  /* 新的 alpha 断言格式，可能 */
        {
        ok = MAX_255(ptr[1]) && (cb->ctypes[ptr[1]] & ctype_lcletter) != 0;
        }
      else switch(ptr[1])  /* 传统的符号格式 */
        {
        case CHAR_C:
        ok = expect_cond_assert == 2;
        break;

        case CHAR_EQUALS_SIGN:
        case CHAR_EXCLAMATION_MARK:
        break;

        case CHAR_LESS_THAN_SIGN:
        ok = ptr[2] == CHAR_EQUALS_SIGN || ptr[2] == CHAR_EXCLAMATION_MARK;
        break;

        default:
        ok = FALSE;
        }
      }

    if (!ok)
      {
      ptr--;   /* 调整错误偏移量 */
      errorcode = ERR28;
      goto FAILED;
      }
    }

  /* 记住我们是否期望条件断言，并设置此项的默认值。 */
  prev_expect_cond_assert = expect_cond_assert;
  expect_cond_assert = 0;

  /* 记住前一个重要项的量化状态，然后设置此项的默认值。 */
  prev_okquantifier = okquantifier;
  prev_meta_quantifier = meta_quantifier;
  okquantifier = FALSE;
  meta_quantifier = 0;

  /* 如果前一个重要项是一个量词，如果后面有修饰符，调整解析的代码。
  基本元值始终按顺序后跟 PLUS 和 QUERY 值。我们在这里执行而不是在读取量词后执行，
  这样中间的注释和 /x 空白可以被忽略，而不必复制代码。 */
  if (prev_meta_quantifier != 0 && (c == CHAR_QUESTION_MARK || c == CHAR_PLUS))
    {
    parsed_pattern[(prev_meta_quantifier == META_MINMAX)? -3 : -1] =
      prev_meta_quantifier + ((c == CHAR_QUESTION_MARK)?
        0x00020000u : 0x00010000u);
    continue;  /* 在模式中的下一个字符 */
    }

  /* 处理模式主体中的下一个项。 */
  switch(c)
    {
    default:              /* 非特殊字符 */
    PARSED_LITERAL(c, parsed_pattern);
    break;

    /* ---- 转义序列 ---- */
    case CHAR_BACKSLASH:
    tempptr = ptr;
    escape = PRIV(check_escape)(&ptr, ptrend, &c, &errorcode, options,
      cb->cx->extra_options, FALSE, cb);
    if (errorcode != 0)
      {
      ESCAPE_FAILED:
      if ((extra_options & PCRE2_EXTRA_BAD_ESCAPE_IS_LITERAL) == 0)
        goto FAILED;
      ptr = tempptr;
      if (ptr >= ptrend) c = CHAR_BACKSLASH; else
        {
        GETCHARINCTEST(c, ptr);   /* 获取字符值，增加指针 */
        }
      escape = 0;                 /* 视为字面字符 */
      }

    /* 转义是数据转义或字面字符。 */
    if (escape == 0)
      {
      PARSED_LITERAL(c, parsed_pattern);
      }
    /* 如果转义是一个反向引用，则保留偏移量以便于对坏的反向引用给出更有用的诊断。对于小于10的组编号的引用，我们不能使用超过两个项目在parsed_pattern中，因为它们可能只是输入中的两个字符（在64位世界中，偏移可能需要两个元素）。因此，对于它们，第一次出现的偏移量保存在一个特殊的向量中。 */
    else if (escape < 0)
      {
      offset = (PCRE2_SIZE)(ptr - cb->start_pattern - 1);
      escape = -escape;
      *parsed_pattern++ = META_BACKREF | (uint32_t)escape;
      if (escape < 10)
        {
        if (cb->small_ref_offset[escape] == PCRE2_UNSET)
          cb->small_ref_offset[escape] = offset;
        }
      else
        {
        PUTOFFSET(offset, parsed_pattern);
        }
      okquantifier = TRUE;
      }

    /* 转义是字符类，如\d等，或其他特殊的转义指示符，如\A或\X。大多数情况下，它们只生成一个解析项，但\P和\p后面跟着一个16位类型和一个16位值。只有在Unicode可用时才支持它们。类型和值被打包成一个32位值，所以整个序列只使用解析向量中的两个元素。这是因为当设置了PCRE2_UCP时，如果将\d（例如）转换为\p{Nd}，则使用相同的编码。

    还有一些情况，转义序列后面跟着一个名称：\k{name}、\k<name>和\k'name'是按名称的反向引用，\g<name>和\g'name'是按名称的子例程调用；\g{name}是\k{name}的同义词。注意\g<number>和\g'number'由check_escape()处理，并作为负值返回（在上面处理）。名称被编码为模式中的偏移量和长度。 */
    else switch (escape)
      {
      case ESC_C:
#ifdef NEVER_BACKSLASH_C
      // 如果定义了 NEVER_BACKSLASH_C，设置错误码为 ERR85，跳转到 ESCAPE_FAILED 标签
      errorcode = ERR85;
      goto ESCAPE_FAILED;
#else
      // 如果未定义 NEVER_BACKSLASH_C，检查是否设置了 PCRE2_NEVER_BACKSLASH_C 选项
      if ((options & PCRE2_NEVER_BACKSLASH_C) != 0)
        {
        // 如果设置了 PCRE2_NEVER_BACKSLASH_C 选项，设置错误码为 ERR83，跳转到 ESCAPE_FAILED 标签
        errorcode = ERR83;
        goto ESCAPE_FAILED;
        }
#endif
      // 设置 okquantifier 为 TRUE
      okquantifier = TRUE;
      // 将 META_ESCAPE 加上转义字符添加到解析后的模式中
      *parsed_pattern++ = META_ESCAPE + escape;
      // 跳出 switch 语句
      break;

      case ESC_X:
#ifndef SUPPORT_UNICODE
      // 如果不支持 Unicode，设置错误码为 ERR45，跳转到 ESCAPE_FAILED 标签
      errorcode = ERR45;   /* Supported only with Unicode support */
      goto ESCAPE_FAILED;
#endif
      case ESC_H:
      case ESC_h:
      case ESC_N:
      case ESC_R:
      case ESC_V:
      case ESC_v:
      // 设置 okquantifier 为 TRUE
      okquantifier = TRUE;
      // 将 META_ESCAPE 加上转义字符添加到解析后的模式中
      *parsed_pattern++ = META_ESCAPE + escape;
      // 跳出 switch 语句
      break;

      default:  /* \A, \B, \b, \G, \K, \Z, \z cannot be quantified. */
      // 将 META_ESCAPE 加上转义字符添加到解析后的模式中
      *parsed_pattern++ = META_ESCAPE + escape;
      // 跳出 switch 语句
      break;

      /* Escapes that change in UCP mode. Note that PCRE2_UCP will never be set
      without Unicode support because it is checked when pcre2_compile() is
      called. */

      case ESC_d:
      case ESC_D:
      case ESC_s:
      case ESC_S:
      case ESC_w:
      case ESC_W:
      // 设置 okquantifier 为 TRUE
      okquantifier = TRUE;
      // 如果未设置 PCRE2_UCP 选项
      if ((options & PCRE2_UCP) == 0)
        {
        // 将 META_ESCAPE 加上转义字符添加到解析后的模式中
        *parsed_pattern++ = META_ESCAPE + escape;
        }
      else
        {
        // 如果设置了 PCRE2_UCP 选项，根据不同的转义字符进行处理
        *parsed_pattern++ = META_ESCAPE +
          ((escape == ESC_d || escape == ESC_s || escape == ESC_w)?
            ESC_p : ESC_P);
        switch(escape)
          {
          case ESC_d:
          case ESC_D:
          // 添加 Unicode 属性匹配信息到解析后的模式中
          *parsed_pattern++ = (PT_PC << 16) | ucp_Nd;
          break;

          case ESC_s:
          case ESC_S:
          // 添加 Unicode 属性匹配信息到解析后的模式中
          *parsed_pattern++ = PT_SPACE << 16;
          break;

          case ESC_w:
          case ESC_W:
          // 添加 Unicode 属性匹配信息到解析后的模式中
          *parsed_pattern++ = PT_WORD << 16;
          break;
          }
        }
      // 跳出 switch 语句
      break;

      /* Unicode property matching */

      case ESC_P:
      case ESC_p:
      // 处理 Unicode 属性匹配
#ifdef SUPPORT_UNICODE
        {
        # 定义变量 negated，ptype，pdata，初始化为 0
        BOOL negated;
        uint16_t ptype = 0, pdata = 0;
        # 调用 get_ucp 函数，解析 Unicode 属性，获取 negated，ptype，pdata，errorcode
        if (!get_ucp(&ptr, &negated, &ptype, &pdata, &errorcode, cb))
          # 如果解析失败，跳转到 ESCAPE_FAILED 标签
          goto ESCAPE_FAILED;
        # 如果 negated 为真，根据 escape 的值更新 escape
        if (negated) escape = (escape == ESC_P)? ESC_p : ESC_P;
        # 将 META_ESCAPE 和 escape 的值写入 parsed_pattern
        *parsed_pattern++ = META_ESCAPE + escape;
        # 将 ptype 左移 16 位并与 pdata 相加，写入 parsed_pattern
        *parsed_pattern++ = (ptype << 16) | pdata;
        # 设置 okquantifier 为真
        okquantifier = TRUE;
        }
#else
      # 如果不支持 Unicode，设置 errorcode 为 ERR45
      errorcode = ERR45;
      # 跳转到 ESCAPE_FAILED 标签
      goto ESCAPE_FAILED;
#else
      # 如果遇到 \P 和 \p，直接跳出循环
      break;  /* End \P and \p */

      /* 当 \g 与引号或尖括号一起使用作为定界符时，它是一个数字或命名子例程调用，控制流会来到这里。当与大括号定界符一起使用时，它是一个数字反向引用，不会来到这里，因为 check_escape() 直接将其作为引用返回。 \k 总是一个命名反向引用。 */

      case ESC_g:
      case ESC_k:
      # 如果指针超出范围，或者下一个字符不是左大括号、小于号或撇号
      if (ptr >= ptrend || (*ptr != CHAR_LEFT_CURLY_BRACKET &&
          *ptr != CHAR_LESS_THAN_SIGN && *ptr != CHAR_APOSTROPHE))
        {
        errorcode = (escape == ESC_g)? ERR57 : ERR69;
        goto ESCAPE_FAILED;
        }
      terminator = (*ptr == CHAR_LESS_THAN_SIGN)?
        CHAR_GREATER_THAN_SIGN : (*ptr == CHAR_APOSTROPHE)?
        CHAR_APOSTROPHE : CHAR_RIGHT_CURLY_BRACKET;

      /* 对于非大括号定界的 \g，检查数字递归。 */

      if (escape == ESC_g && terminator != CHAR_RIGHT_CURLY_BRACKET)
        {
        PCRE2_SPTR p = ptr + 1;

        if (read_number(&p, ptrend, cb->bracount, MAX_GROUP_NUMBER, ERR61, &i,
            &errorcode))
          {
          if (p >= ptrend || *p != terminator)
            {
            errorcode = ERR57;
            goto ESCAPE_FAILED;
            }
          ptr = p;
          goto SET_RECURSION;
          }
        if (errorcode != 0) goto ESCAPE_FAILED;
        }

      /* 不是数字递归 */

      if (!read_name(&ptr, ptrend, utf, terminator, &offset, &name, &namelen,
          &errorcode, cb)) goto ESCAPE_FAILED;

      /* \k 和 \g 在大括号中使用时是反向引用，而在引号或尖括号中使用时是递归 */

      *parsed_pattern++ =
        (escape == ESC_k || terminator == CHAR_RIGHT_CURLY_BRACKET)?
          META_BACKREF_BYNAME : META_RECURSE_BYNAME;
      *parsed_pattern++ = namelen;

      PUTOFFSET(offset, parsed_pattern);
      okquantifier = TRUE;
      break;  /* End special escape processing */
      }
    # 结束转义序列处理
    break;    /* End escape sequence processing */


    /* ---- 单字符特殊项 ---- */

    case CHAR_CIRCUMFLEX_ACCENT:
    *parsed_pattern++ = META_CIRCUMFLEX;
    break;

    case CHAR_DOLLAR_SIGN:
    *parsed_pattern++ = META_DOLLAR;
    break;

    case CHAR_DOT:
    *parsed_pattern++ = META_DOT;
    okquantifier = TRUE;
    break;


    /* ---- 单字符量词 ---- */

    case CHAR_ASTERISK:
    meta_quantifier = META_ASTERISK;
    goto CHECK_QUANTIFIER;

    case CHAR_PLUS:
    meta_quantifier = META_PLUS;
    goto CHECK_QUANTIFIER;

    case CHAR_QUESTION_MARK:
    meta_quantifier = META_QUERY;
    goto CHECK_QUANTIFIER;


    /* ---- 可能的 {n,m} 量词 ---- */

    case CHAR_LEFT_CURLY_BRACKET:
    if (!read_repeat_counts(&ptr, ptrend, &min_repeat, &max_repeat,
        &errorcode))
      {
      if (errorcode != 0) goto FAILED;     /* 量词错误 */
      PARSED_LITERAL(c, parsed_pattern);   /* 不是量词 */
      break;                               /* 不再处理量词 */
      }
    meta_quantifier = META_MINMAX;
    /* 继续执行 */


    /* ---- 量词后处理 ---- */

    /* 检查前一个项后是否允许量词 */
    CHECK_QUANTIFIER:
    if (!prev_okquantifier)
      {
      errorcode = ERR9;
      goto FAILED_BACK;
      }

    /* 大多数 (*VERB) 不允许被量化，但非贪婪量词对于 (*ACCEPT) 可以很有用 - 意思是“在回溯时成功”，一种否定的 (*COMMIT)。因此，我们允许 (*ACCEPT) 被非捕获括号量化，但我们必须允许前面有 (*MARK) 当 (*ACCEPT) 有参数时。 */
    if (parsed_pattern[-1] == META_ACCEPT)
      {
      uint32_t *p;
      for (p = parsed_pattern - 1; p >= verbstartptr; p--) p[1] = p[0];
      *verbstartptr = META_NOCAPTURE;
      parsed_pattern[1] = META_KET;
      parsed_pattern += 2;
      }
    /* 现在我们可以将量词放入解析后的模式向量中。在这个阶段，我们只有基本的量词。在循环的顶部，检查是否有后续的+或?修饰符，这是在移除任何中间注释之后进行的。 */

    *parsed_pattern++ = meta_quantifier;
    if (c == CHAR_LEFT_CURLY_BRACKET)
      {
      *parsed_pattern++ = min_repeat;
      *parsed_pattern++ = max_repeat;
      }
    break;


    /* ---- 字符类 ---- */

    case CHAR_LEFT_SQUARE_BRACKET:
    okquantifier = TRUE;

    /* 在另一个（POSIX）正则表达式库中，使用丑陋的语法 [[:<:]] 和 [[:>:]] 表示“单词开头”和“单词结尾”。由于这些在其他情况下是非法序列，我们通过识别它们来替换它们。它们分别被替换为 \b(?=\w) 和 \b(?<=\w)。类似 [a[:<:]] 这样的序列是错误的，将由下面的正常代码处理。 */

    if (ptrend - ptr >= 6 &&
         (PRIV(strncmp_c8)(ptr, STRING_WEIRD_STARTWORD, 6) == 0 ||
          PRIV(strncmp_c8)(ptr, STRING_WEIRD_ENDWORD, 6) == 0))
      {
      *parsed_pattern++ = META_ESCAPE + ESC_b;

      if (ptr[2] == CHAR_LESS_THAN_SIGN)
        {
        *parsed_pattern++ = META_LOOKAHEAD;
        }
      else
        {
        *parsed_pattern++ = META_LOOKBEHIND;
        *has_lookbehind = TRUE;

        /* 偏移量仅用于“非固定长度”错误；这里不会发生，所以只存储零。 */

        PUTOFFSET((PCRE2_SIZE)0, parsed_pattern);
        }

      if ((options & PCRE2_UCP) == 0)
        *parsed_pattern++ = META_ESCAPE + ESC_w;
      else
        {
        *parsed_pattern++ = META_ESCAPE + ESC_p;
        *parsed_pattern++ = PT_WORD << 16;
        }
      *parsed_pattern++ = META_KET;
      ptr += 6;
      break;
      }

    /* PCRE支持在类中支持POSIX类的内容。Perl在顶层遇到它们会报错，所以我们也会这样做。 */
    # 如果指针位置小于结束位置，并且当前字符是冒号、点或等号，并且符合 POSIX 语法，则执行以下操作
    if (ptr < ptrend && (*ptr == CHAR_COLON || *ptr == CHAR_DOT ||
         *ptr == CHAR_EQUALS_SIGN) &&
        check_posix_syntax(ptr, ptrend, &tempptr))
      {
      # 如果当前字符是冒号，则设置错误代码为 ERR12，否则设置为 ERR13
      errorcode = (*ptr-- == CHAR_COLON)? ERR12 : ERR13;
      # 跳转到失败标签
      goto FAILED;
      }

    /* 处理普通字符类。如果第一个字符是 '^'，则设置否定标志。如果前几个字符（^之前或之后）是 \Q\E 或 \E 或空格或制表符，在扩展模式下，我们也跳过它们。这样与 Perl 兼容。 */

    # 初始化否定标志为 FALSE
    negate_class = FALSE;
    # 循环处理字符类
    while (ptr < ptrend)
      {
      # 获取下一个字符
      GETCHARINCTEST(c, ptr);
      # 如果字符是反斜杠
      if (c == CHAR_BACKSLASH)
        {
        # 如果下一个字符是 'E'，则指针向后移动一位
        if (ptr < ptrend && *ptr == CHAR_E) ptr++;
        # 如果剩余长度大于等于 3 并且下一个 3 个字符是 \Q\E，则指针向后移动 3 位
        else if (ptrend - ptr >= 3 &&
             PRIV(strncmp_c8)(ptr, STR_Q STR_BACKSLASH STR_E, 3) == 0)
          ptr += 3;
        else
          break;
        }
      # 如果是扩展模式下的空格或制表符，则继续循环
      else if ((options & PCRE2_EXTENDED_MORE) != 0 &&
               (c == CHAR_SPACE || c == CHAR_HT))  /* 注意：只有这两个 */
        continue;
      # 如果不是否定类并且字符是 '^'，则设置否定标志为 TRUE
      else if (!negate_class && c == CHAR_CIRCUMFLEX_ACCENT)
        negate_class = TRUE;
      else break;
      }

    /* 现在是类的真正内容；c 是第一个 "真实" 字符。只有在设置了选项时才允许空类。 */

    # 如果字符是右方括号并且外部选项中设置了允许空类，则执行以下操作
    if (c == CHAR_RIGHT_SQUARE_BRACKET &&
        (cb->external_options & PCRE2_ALLOW_EMPTY_CLASS) != 0)
      {
      # 如果否定标志为真，则将 META_CLASS_EMPTY_NOT 写入解析模式，否则写入 META_CLASS_EMPTY
      *parsed_pattern++ = negate_class? META_CLASS_EMPTY_NOT : META_CLASS_EMPTY;
      # 跳出类处理
      break;
      }

    /* 处理非空类。*/

    # 如果否定标志为真，则将 META_CLASS_NOT 写入解析模式，否则写入 META_CLASS
    *parsed_pattern++ = negate_class? META_CLASS_NOT : META_CLASS;
    # 初始化类范围状态为 RANGE_NO

    /* 在 EBCDIC 环境中，Perl 对字母范围进行特殊处理，因为编码中存在空隙，简单地使用范围 A-Z（例如）会包括空隙中的字符。这仅适用于范围的两个值都是文字值；[\xC1-\xE9] 与 [A-Z] 是不同的 */
    # 在这方面进行处理。为了适应这一点，我们跟踪字符值是字面还是非字面的，并使用状态变量来处理范围。
    
    # 循环处理类的内容
#ifdef SUPPORT_UNICODE
        # 如果支持 Unicode，则执行以下代码块
        if ((options & PCRE2_UCP) != 0)
          {
          # 获取 POSIX 类别和值
          int ptype = posix_substitutes[2*posix_class];
          int pvalue = posix_substitutes[2*posix_class + 1];
          # 如果 POSIX 类别大于等于 0
          if (ptype >= 0)
            {
            # 将 POSIX 类别和值添加到解析模式中
            *parsed_pattern++ = META_ESCAPE + (posix_negate? ESC_P : ESC_p);
            *parsed_pattern++ = (ptype << 16) | pvalue;
            # 跳转到 CLASS_CONTINUE 标签处
            goto CLASS_CONTINUE;
            }

          # 如果值不为 0
          if (pvalue != 0)
            {
            # 添加 POSIX 类别到解析模式中
            *parsed_pattern++ = META_ESCAPE + (posix_negate? ESC_H : ESC_h);
            # 跳转到 CLASS_CONTINUE 标签处
            goto CLASS_CONTINUE;
            }

          # 继续执行下面的代码
          /* Fall through */
          }
#ifdef SUPPORT_UNICODE
            {
            # 定义变量 negated, ptype, pdata
            BOOL negated;
            uint16_t ptype = 0, pdata = 0;
            # 获取 Unicode 属性
            if (!get_ucp(&ptr, &negated, &ptype, &pdata, &errorcode, cb))
              # 跳转到 FAILED 标签处
              goto FAILED;
            # 如果是否定形式，则转换转义字符
            if (negated) escape = (escape == ESC_P)? ESC_p : ESC_P;
            # 添加转义字符和属性到解析模式中
            *parsed_pattern++ = META_ESCAPE + escape;
            *parsed_pattern++ = (ptype << 16) | pdata;
            }
#else
          # 如果不支持 Unicode，则设置错误码并跳转到 FAILED 标签处
          errorcode = ERR45;
          goto FAILED;
#endif
          # 跳出循环
          break;  /* End \P and \p */

          default:    /* All others are not allowed in a class */
          # 设置错误码并跳转到 FAILED 标签处
          errorcode = ERR7;
          ptr--;
          goto FAILED;
          }

        # 如果下一个字符是减号且不是类的结束符，则设置错误码并跳转到 FAILED 标签处
        if (ptr < ptrend - 1 && *ptr == CHAR_MINUS &&
            ptr[1] != CHAR_RIGHT_SQUARE_BRACKET)
          {
          errorcode = ERR50;
          goto FAILED;
          }
        }

      /* Proceed to next thing in the class. */

      CLASS_CONTINUE:
      # 如果指针超出范围，则设置错误码并跳转到 FAILED 标签处
      if (ptr >= ptrend)
        {
        errorcode = ERR6;  /* Missing terminating ']' */
        goto FAILED;
        }
      # 获取下一个字符
      GETCHARINCTEST(c, ptr);
      # 如果下一个字符是类的结束符且不在转义序列中，则跳出循环
      if (c == CHAR_RIGHT_SQUARE_BRACKET && !inescq) break;
      }     /* End of class-processing loop */

    /* -] at the end of a class is a literal '-' */
    # 如果字符类范围状态为 RANGE_STARTED
    if (class_range_state == RANGE_STARTED)
      {
      # 在解析模式的最后一个字符位置插入字符“-”
      parsed_pattern[-1] = CHAR_MINUS;
      # 将字符类范围状态设置为 RANGE_NO
      class_range_state = RANGE_NO;
      }

    # 在解析模式中插入 META_CLASS_END
    *parsed_pattern++ = META_CLASS_END;
    # 跳出 switch 语句，表示字符类结束
    break;  /* End of character class */


    # ---- 开括号 ----

    # 如果当前字符为左括号
    case CHAR_LEFT_PARENTHESIS:
    # 如果指针超出了解析模式的末尾，跳转到 UNCLOSED_PARENTHESIS 标签
    if (ptr >= ptrend) goto UNCLOSED_PARENTHESIS;

    # 如果 ( 后面不是 ?，则它可能是一个捕获组、特殊动词、字母断言或正向非原子前瞻
#ifdef SUPPORT_UNICODE
          # 如果支持 Unicode，则将解析模式指针指向 META_SCRIPT_RUN
          *parsed_pattern++ = META_SCRIPT_RUN;
          # 嵌套深度加一
          nest_depth++;
          # 指针向前移动一位
          ptr++;
          # 如果 meta 等于 META_ATOMIC_SCRIPT_RUN
          if (meta == META_ATOMIC_SCRIPT_RUN)
            {
            # 将解析模式指针指向 META_ATOMIC
            *parsed_pattern++ = META_ATOMIC;
            # 如果 top_nest 为空，则将其指向 cb->start_workspace，否则将其加一
            if (top_nest == NULL) top_nest = (nest_save *)(cb->start_workspace);
            else if (++top_nest >= end_nests)
              {
              # 错误码设为 ERR84，跳转到 FAILED 标签
              errorcode = ERR84;
              goto FAILED;
              }
            # 设置 top_nest 的嵌套深度、标志、选项
            top_nest->nest_depth = nest_depth;
            top_nest->flags = NSF_ATOMICSR;
            top_nest->options = options & PARSE_TRACKED_OPTIONS;
            }
          # 跳出循环
          break;
#else  /* SUPPORT_UNICODE */
          # 错误码设为 ERR96，跳转到 FAILED 标签
          errorcode = ERR96;
          goto FAILED;
    /* ---- Items starting (? ---- */

    # 以 (? 开头的项目类型由后面的内容决定。在 "default" 下处理 (?| 和选项更改，因为两者都需要在嵌套栈上创建一个新块。以 (?# 开头的注释在上面处理。注意，对于序列 (?- 存在一些歧义，因为如果后面跟着一个数字，那么它是相对递归或子例程调用，否则它是取消选项设置。
    # 如果指针向前移动一位后超出了 ptrend，则跳转到 UNCLOSED_PARENTHESIS 标签
    if (++ptr >= ptrend) goto UNCLOSED_PARENTHESIS;

    # 跳出循环，结束 ( 处理
    break;


    /* ---- Branch terminators ---- */

    # 分支终止符：如果在 (?| 组中，则重置捕获计数
    case CHAR_VERTICAL_LINE:
    if (top_nest != NULL && top_nest->nest_depth == nest_depth &&
        (top_nest->flags & NSF_RESET) != 0)
      {
      if (cb->bracount > top_nest->max_group)
        top_nest->max_group = (uint16_t)cb->bracount;
      cb->bracount = top_nest->reset_group;
      }
    # 将解析模式指针指向 META_ALT
    *parsed_pattern++ = META_ALT;
    # 跳出循环
    break;

    # 组结束；如果在 (?| 组中，则将捕获计数重置为最大值，并/或重置在解析过程中跟踪的选项。禁止断言条件的量词。
    case CHAR_RIGHT_PARENTHESIS:
    okquantifier = TRUE;
    # 如果顶部嵌套不为空且顶部嵌套的嵌套深度等于当前嵌套深度
    if (top_nest != NULL && top_nest->nest_depth == nest_depth)
      {
      # 更新选项，将已跟踪的选项清除并设置为顶部嵌套的选项
      options = (options & ~PARSE_TRACKED_OPTIONS) | top_nest->options;
      # 如果顶部嵌套的标志包含 NSF_RESET 并且最大组数大于当前 bracount
      if ((top_nest->flags & NSF_RESET) != 0 &&
          top_nest->max_group > cb->bracount)
        # 更新当前 bracount 为顶部嵌套的最大组数
        cb->bracount = top_nest->max_group;
      # 如果顶部嵌套的标志包含 NSF_CONDASSERT
      if ((top_nest->flags & NSF_CONDASSERT) != 0)
        # 将 okquantifier 设置为 FALSE
        okquantifier = FALSE;

      # 如果顶部嵌套的标志包含 NSF_ATOMICSR
      if ((top_nest->flags & NSF_ATOMICSR) != 0)
        {
        # 在解析模式中添加 META_KET
        *parsed_pattern++ = META_KET;
        }

      # 如果顶部嵌套等于起始工作空间的指针，则将顶部嵌套设置为空；否则将顶部嵌套指针向前移动
      if (top_nest == (nest_save *)(cb->start_workspace)) top_nest = NULL;
        else top_nest--;
      }
    # 如果嵌套深度为0，表示未匹配的右括号
    if (nest_depth == 0)    /* Unmatched closing parenthesis */
      {
      # 设置错误代码为 ERR22
      errorcode = ERR22;
      # 跳转到 FAILED_BACK 标签处
      goto FAILED_BACK;
      }
    # 减少嵌套深度
    nest_depth--;
    # 在解析模式中添加 META_KET
    *parsed_pattern++ = META_KET;
    # 跳出循环
    break;
    }  /* End of switch on pattern character */
  }    /* End of main character scan loop */
# 如果在动词名称的末尾达到模式的结束。检查是否缺少动词名称末尾的括号
if (inverbname && ptr >= ptrend)
  {
  errorcode = ERR60;
  goto FAILED;
  }

# 处理最终项的调用
PARSED_END:
parsed_pattern = manage_callouts(ptr, &previous_callout, auto_callout,
  parsed_pattern, cb);

# 为了单词和行匹配插入尾随项（为了pcre2grep的功能）
if ((extra_options & PCRE2_EXTRA_MATCH_LINE) != 0)
  {
  *parsed_pattern++ = META_KET;
  *parsed_pattern++ = META_DOLLAR;
  }
else if ((extra_options & PCRE2_EXTRA_MATCH_WORD) != 0)
  {
  *parsed_pattern++ = META_KET;
  *parsed_pattern++ = META_ESCAPE + ESC_b;
  }

# 终止解析的模式，然后如果所有组都关闭则返回成功。否则有未关闭的括号。
if (parsed_pattern >= parsed_pattern_end)
  {
  errorcode = ERR63;  /* 内部错误（解析模式溢出） */
  goto FAILED;
  }

*parsed_pattern = META_END;
if (nest_depth == 0) return 0;

# 未关闭的括号
UNCLOSED_PARENTHESIS:
errorcode = ERR14;

# 所有失败都到这里
FAILED:
cb->erroroffset = (PCRE2_SIZE)(ptr - cb->start_pattern);
return errorcode;

# 一些错误需要指示前一个字符
FAILED_BACK:
ptr--;
goto FAILED;

# 这种失败发生了多次
BAD_VERSION_CONDITION:
errorcode = ERR79;
goto FAILED;
}



/*************************************************
*       查找第一个重要的操作码            *
*************************************************/

# 这个函数被几个扫描编译表达式的函数调用，用于查找固定的第一个字符，或者锚定的操作码等。它跳过不影响这一点的内容。对于某些调用，跳过负向前断言和所有向后断言，以及\b断言是有意义的；对于其他调用则不是。

# 参数：
#   code         指向组的开始的指针
#   skipassert   如果要跳过某些断言，则为TRUE
# 返回指向第一个有效操作码的指针
static const PCRE2_UCHAR*
first_significant_code(PCRE2_SPTR code, BOOL skipassert)
{
# 无限循环，直到遇到 return 语句
for (;;)
  {
  # 根据操作码的类型进行不同的处理
  switch ((int)*code)
    {
    # 如果是断言相关的操作码，根据 skipassert 参数决定是否返回当前指针
    case OP_ASSERT_NOT:
    case OP_ASSERTBACK:
    case OP_ASSERTBACK_NOT:
    case OP_ASSERTBACK_NA:
    if (!skipassert) return code;
    # 跳过断言相关操作码
    do code += GET(code, 1); while (*code == OP_ALT);
    code += PRIV(OP_lengths)[*code];
    break;

    # 如果是单词边界相关的操作码，根据 skipassert 参数决定是否返回当前指针
    case OP_WORD_BOUNDARY:
    case OP_NOT_WORD_BOUNDARY:
    if (!skipassert) return code;
    # 继续执行下一个 case

    # 如果是调用相关的操作码，直接跳过
    case OP_CALLOUT:
    case OP_CREF:
    case OP_DNCREF:
    case OP_RREF:
    case OP_DNRREF:
    case OP_FALSE:
    case OP_TRUE:
    code += PRIV(OP_lengths)[*code];
    break;

    # 如果是调用相关的字符串操作码，跳过
    case OP_CALLOUT_STR:
    code += GET(code, 1 + 2*LINK_SIZE);
    break;

    # 如果是跳过零操作码，跳过
    case OP_SKIPZERO:
    code += 2 + GET(code, 2) + LINK_SIZE;
    break;

    # 如果是条件操作码，判断是否是 DEFINE 或者是否有多个分支
    case OP_COND:
    case OP_SCOND:
    if (code[1+LINK_SIZE] != OP_FALSE ||   /* Not DEFINE */
        code[GET(code, 1)] != OP_KET)      /* More than one branch */
      return code;
    code += GET(code, 1) + 1 + LINK_SIZE;
    break;

    # 如果是标记相关的操作码，跳过
    case OP_MARK:
    case OP_COMMIT_ARG:
    case OP_PRUNE_ARG:
    case OP_SKIP_ARG:
    case OP_THEN_ARG:
    code += code[1] + PRIV(OP_lengths)[*code];
    break;

    # 默认情况下返回当前指针
    default:
    return code;
    }
  }
# 不会执行到这里
}
# 支持 Unicode 模式下的获取其他大小写范围
/* 
Arguments:
  cptr        points to starting character value; updated
  d           end value
  ocptr       where to put start of othercase range
  odptr       where to put end of othercase range

Yield:        -1 when no more
               0 when a range is returned
              >0 the CASESET offset for char with multiple other cases
                in this case, ocptr contains the original
*/

static int
get_othercase_range(uint32_t *cptr, uint32_t d, uint32_t *ocptr,
  uint32_t *odptr)
{
// 定义变量
uint32_t c, othercase, next;
unsigned int co;

/* Find the first character that has an other case. If it has multiple other
cases, return its case offset value. */

// 遍历字符值范围
for (c = *cptr; c <= d; c++)
  {
  // 如果字符有多个其他大小写形式
  if ((co = UCD_CASESET(c)) != 0)
    {
    *ocptr = c++;   /* Character that has the set */
    *cptr = c;      /* Rest of input range */
    return (int)co;
    }
  // 如果字符有单个其他大小写形式
  if ((othercase = UCD_OTHERCASE(c)) != c) break;
  }

// 如果已经遍历完整个范围
if (c > d) return -1;  /* Reached end of range */

/* Found a character that has a single other case. Search for the end of the
range, which is either the end of the input range, or a character that has zero
or more than one other cases. */

// 找到有单个其他大小写形式的字符，搜索范围的结束
*ocptr = othercase;
next = othercase + 1;

for (++c; c <= d; c++)
  {
  // 如果字符有多个其他大小写形式或者下一个字符不是当前字符的其他大小写形式
  if ((co = UCD_CASESET(c)) != 0 || UCD_OTHERCASE(c) != next) break;
  next++;
  }

*odptr = next - 1;     /* End of othercase range */
*cptr = c;             /* Rest of input range */
return 0;
}
#endif  /* SUPPORT_UNICODE */



/*************************************************
* Add a character or range to a class (internal) *
*************************************************/

/* This function packages up the logic of adding a character or range of
characters to a class. The character values in the arguments will be within the
valid values for the current mode (8-bit, 16-bit, UTF, etc). This function is
called only from within the "add to class" group of functions, some of which
are recursive and mutually recursive. The external entry point is
# 将字符添加到类中。

# 参数：
#   classbits     用于字符<256的位图
#   uchardptr     指向额外数据的指针
#   options       选项字
#   cb            编译数据
#   start         范围字符的起始
#   end           范围字符的结束

# 返回值：        添加的<256字符的数量
#                更新指向额外数据的指针
*/

static unsigned int
add_to_class_internal(uint8_t *classbits, PCRE2_UCHAR **uchardptr,
  uint32_t options, compile_block *cb, uint32_t start, uint32_t end)
{
uint32_t c;
uint32_t classbits_end = (end <= 0xff ? end : 0xff);
unsigned int n8 = 0;

# 如果需要不区分大小写匹配，扫描范围并处理备用大小写。在Unicode中，有8位字符有大于255的备用大小写，反之亦然。有时我们可以扩展原始范围。

if ((options & PCRE2_CASELESS) != 0)
  {
#ifdef SUPPORT_UNICODE
  if ((options & (PCRE2_UTF|PCRE2_UCP)) != 0)
    {
    int rc;
    uint32_t oc, od;

    options &= ~PCRE2_CASELESS;   /* 递归调用时移除 */
    c = start;
    while ((rc = get_othercase_range(&c, end, &oc, &od)) >= 0)
      {
      /* 当还有其他情况时，处理单个具有多个其他情况的字符 */

      if (rc > 0) n8 += add_list_to_class_internal(classbits, uchardptr, options, cb,
        PRIV(ucd_caseless_sets) + rc, oc);
        /* 如果有其他情况范围在原始范围内，则不做任何操作 */

      else if (oc >= cb->class_range_start && od <= cb->class_range_end) continue;

      /* 如果有重叠，则扩展原始范围，注意如果 oc < c，则 od > end 是不可能的，因为子范围总是比基本范围短。否则，使用递归调用来添加额外的范围。 */

      else if (oc < start && od >= start - 1) start = oc; /* 向下扩展 */
      else if (od > end && oc <= end + 1)
        {
        end = od;       /* 向上扩展 */
        if (end > classbits_end) classbits_end = (end <= 0xff ? end : 0xff);
        }
      else n8 += add_to_class_internal(classbits, uchardptr, options, cb, oc, od);
      }
    }
  else
#endif  /* SUPPORT_UNICODE */

  /* Not UTF mode */

  for (c = start; c <= classbits_end; c++)
    {
    SETBIT(classbits, cb->fcc[c]);  // 设置类位图中对应字符的位
    n8++;  // 统计字符个数
    }
  }

/* Now handle the originally supplied range. Adjust the final value according
to the bit length - this means that the same lists of (e.g.) horizontal spaces
can be used in all cases. */

if ((options & PCRE2_UTF) == 0 && end > MAX_NON_UTF_CHAR)
  end = MAX_NON_UTF_CHAR;  // 如果不是 UTF 模式，并且结束值大于最大非 UTF 字符值，则将结束值调整为最大非 UTF 字符值

if (start > cb->class_range_start && end < cb->class_range_end) return n8;  // 如果起始值大于类范围的起始值，并且结束值小于类范围的结束值，则返回字符个数

/* Use the bitmap for characters < 256. Otherwise use extra data.*/

for (c = start; c <= classbits_end; c++)
  {
  /* Regardless of start, c will always be <= 255. */
  SETBIT(classbits, c);  // 设置类位图中对应字符的位
  n8++;  // 统计字符个数
  }

#ifdef SUPPORT_WIDE_CHARS
if (start <= 0xff) start = 0xff + 1;  // 如果起始值小于等于 0xff，则将起始值调整为 0xff + 1

if (end >= start)
  {
  PCRE2_UCHAR *uchardata = *uchardptr;  // 定义宽字符数据指针

#ifdef SUPPORT_UNICODE
  if ((options & PCRE2_UTF) != 0)  // 如果是 UTF 模式
    {
    if (start < end)
      {
      *uchardata++ = XCL_RANGE;  // 将 XCL_RANGE 写入宽字符数据
      uchardata += PRIV(ord2utf)(start, uchardata);  // 将起始值转换为 UTF 编码写入宽字符数据
      uchardata += PRIV(ord2utf)(end, uchardata);  // 将结束值转换为 UTF 编码写入宽字符数据
      }
    else if (start == end)
      {
      *uchardata++ = XCL_SINGLE;  // 将 XCL_SINGLE 写入宽字符数据
      uchardata += PRIV(ord2utf)(start, uchardata);  // 将起始值转换为 UTF 编码写入宽字符数据
      }
    }
  else
#endif  /* SUPPORT_UNICODE */

  /* Without UTF support, character values are constrained by the bit length,
  and can only be > 256 for 16-bit and 32-bit libraries. */

#if PCRE2_CODE_UNIT_WIDTH == 8
    {}
#else
  if (start < end)
    {
    *uchardata++ = XCL_RANGE;  // 将 XCL_RANGE 写入宽字符数据
    *uchardata++ = start;  // 将起始值写入宽字符数据
    *uchardata++ = end;  // 将结束值写入宽字符数据
    }
  else if (start == end)
    {
    *uchardata++ = XCL_SINGLE;  // 将 XCL_SINGLE 写入宽字符数据
    *uchardata++ = start;  // 将起始值写入宽字符数据
    }
#endif  /* PCRE2_CODE_UNIT_WIDTH == 8 */
  *uchardptr = uchardata;   /* Updata extra data pointer */  // 更新额外数据指针
  }
#else  /* SUPPORT_WIDE_CHARS */
  (void)uchardptr;          /* Avoid compiler warning */  // 避免编译器警告
#endif /* SUPPORT_WIDE_CHARS */

return n8;    /* Number of 8-bit characters */  // 返回 8 位字符的数量
}



#ifdef SUPPORT_UNICODE
/*************************************************
/* Add a list of characters to a class (internal) */
/*************************************************/

/* This function is used for adding a list of case-equivalent characters to a
class when in UTF mode. This function is called only from within
add_to_class_internal(), with which it is mutually recursive.

Arguments:
  classbits     the bit map for characters < 256
  uchardptr     points to the pointer for extra data
  options       the options word
  cb            contains pointers to tables etc.
  p             points to row of 32-bit values, terminated by NOTACHAR
  except        character to omit; this is used when adding lists of
                  case-equivalent characters to avoid including the one we
                  already know about

Returns:        the number of < 256 characters added
                the pointer to extra data is updated
*/

static unsigned int
add_list_to_class_internal(uint8_t *classbits, PCRE2_UCHAR **uchardptr,
  uint32_t options, compile_block *cb, const uint32_t *p, unsigned int except)
{
unsigned int n8 = 0;
while (p[0] < NOTACHAR)
  {
  unsigned int n = 0;
  if (p[0] != except)
    {
    while(p[n+1] == p[0] + n + 1) n++;
    n8 += add_to_class_internal(classbits, uchardptr, options, cb, p[0], p[n]);
    }
  p += n + 1;
  }
return n8;
}
#endif



/*************************************************
*   External entry point for add range to class  *
*************************************************/

/* This function sets the overall range so that the internal functions can try
to avoid duplication when handling case-independence.

Arguments:
  classbits     the bit map for characters < 256
  uchardptr     points to the pointer for extra data
  options       the options word
  cb            compile data
  start         start of range character
  end           end of range character

Returns:        the number of < 256 characters added
                the pointer to extra data is updated
*/

static unsigned int
# 将字符添加到类中
add_to_class(uint8_t *classbits, PCRE2_UCHAR **uchardptr, uint32_t options,
  compile_block *cb, uint32_t start, uint32_t end)
{
  # 设置类范围的起始和结束位置
  cb->class_range_start = start;
  cb->class_range_end = end;
  # 调用内部函数将字符添加到类中
  return add_to_class_internal(classbits, uchardptr, options, cb, start, end);
}


/*************************************************
*   External entry point for add list to class   *
*************************************************/

/* 该函数用于将水平或垂直空白字符列表添加到类中。列表必须按顺序排列，以便可以检测和适当处理字符范围。该函数设置整体范围，以便内部函数在处理大小写时尝试避免重复。

参数：
  classbits     字符<256的位图
  uchardptr     指向额外数据的指针
  options       选项字
  cb            包含指向表等的指针
  p             指向32位值行，以NOTACHAR结尾
  except        要省略的字符；在添加大小写等效字符列表时使用，以避免包含我们已知的字符

返回值：        添加的<256个字符的数量
                更新额外数据的指针
*/

static unsigned int
add_list_to_class(uint8_t *classbits, PCRE2_UCHAR **uchardptr, uint32_t options,
  compile_block *cb, const uint32_t *p, unsigned int except)
{
  unsigned int n8 = 0;
  while (p[0] < NOTACHAR)
  {
    unsigned int n = 0;
    if (p[0] != except)
    {
      while(p[n+1] == p[0] + n + 1) n++;
      cb->class_range_start = p[0];
      cb->class_range_end = p[n];
      n8 += add_to_class_internal(classbits, uchardptr, options, cb, p[0], p[n]);
    }
    p += n + 1;
  }
  return n8;
}



/*************************************************
*    Add characters not in a list to a class     *
*************************************************/
/* This function is used for adding the complement of a list of horizontal or
vertical whitespace to a class. The list must be in order.

Arguments:
  classbits     the bit map for characters < 256
  uchardptr     points to the pointer for extra data
  options       the options word
  cb            contains pointers to tables etc.
  p             points to row of 32-bit values, terminated by NOTACHAR

Returns:        the number of < 256 characters added
                the pointer to extra data is updated
*/
static unsigned int
add_not_list_to_class(uint8_t *classbits, PCRE2_UCHAR **uchardptr,
  uint32_t options, compile_block *cb, const uint32_t *p)
{
    // 检查是否为 UTF 模式
    BOOL utf = (options & PCRE2_UTF) != 0;
    // 初始化小于 256 的字符数为 0
    unsigned int n8 = 0;
    // 如果第一个值大于 0，则将小于该值的字符添加到类中
    if (p[0] > 0)
        n8 += add_to_class(classbits, uchardptr, options, cb, 0, p[0] - 1);
    // 遍历 32 位值数组
    while (p[0] < NOTACHAR)
    {
        // 如果下一个值与当前值相差 1，则跳过
        while (p[1] == p[0] + 1) p++;
        // 将指定范围内的字符添加到类中
        n8 += add_to_class(classbits, uchardptr, options, cb, p[0] + 1,
            (p[1] == NOTACHAR) ? (utf ? 0x10ffffu : 0xffffffffu) : p[1] - 1);
        p++;
    }
    // 返回小于 256 的字符数
    return n8;
}

/*************************************************
*    Find details of duplicate group names       *
*************************************************/

/* This is called from compile_branch() when it needs to know the index and
count of duplicates in the names table when processing named backreferences,
either directly, or as conditions.

Arguments:
  name          points to the name
  length        the length of the name
  indexptr      where to put the index
  countptr      where to put the count of duplicates
  errorcodeptr  where to put an error code
  cb            the compile block

Returns:        TRUE if OK, FALSE if not, error code set
*/
static BOOL
find_dupname_details(PCRE2_SPTR name, uint32_t length, int *indexptr,
  int *countptr, int *errorcodeptr, compile_block *cb)
{
    // 初始化变量
    uint32_t i, groupnumber;
    int count;
    PCRE2_UCHAR *slot = cb->name_table;

    // 找到名称表中的第一个条目
# 遍历已找到的名称列表
for (i = 0; i < cb->names_found; i++)
  {
  # 如果名称匹配，并且后面没有其他字符，跳出循环
  if (PRIV(strncmp)(name, slot+IMM2_SIZE, length) == 0 &&
      slot[IMM2_SIZE+length] == 0) break;
  # 移动指针到下一个名称的位置
  slot += cb->name_entry_size;
  }

# 如果找不到重复的名称，返回错误
if (i >= cb->names_found)
  {
  *errorcodeptr = ERR53;
  cb->erroroffset = name - cb->start_pattern;
  return FALSE;
  }

# 记录索引，然后查看有多少个重复的名称，更新反向引用映射和最大反向引用
*indexptr = i;
count = 0;

# 无限循环，统计重复名称的数量，并更新反向引用映射和最大反向引用
for (;;)
  {
  count++;
  groupnumber = GET2(slot,0);
  cb->backref_map |= (groupnumber < 32)? (1u << groupnumber) : 1;
  if (groupnumber > cb->top_backref) cb->top_backref = groupnumber;
  if (++i >= cb->names_found) break;
  slot += cb->name_entry_size;
  # 如果名称不匹配或者后面还有其他字符，跳出循环
  if (PRIV(strncmp)(name, slot+IMM2_SIZE, length) != 0 ||
    (slot+IMM2_SIZE)[length] != 0) break;
  }

# 将重复名称的数量赋值给 countptr，返回成功
*countptr = count;
return TRUE;
}



/*************************************************
*           Compile one branch                   *
*************************************************/

# 编译一个分支的解析模式，将其编译成 PCRE2_UCHAR 类型的向量。如果在分支过程中更改了选项，使用指针来改变外部选项位。此函数在预编译阶段用于确定所需内存量，也在实际编译阶段使用。lengthptr 的值区分了这两个阶段。
/*
  编译分支的函数，用于将正则表达式编译成可执行代码
  参数：
    optionsptr        指向选项位的指针
    codeptr           指向当前代码点的指针
    pptrptr           指向当前解析模式指针的指针
    errorcodeptr      指向错误代码变量的指针
    firstcuptr        存放第一个必需代码单元的位置
    firstcuflagsptr   存放第一个代码单元标志的位置
    reqcuptr          存放最后一个必需代码单元的位置
    reqcuflagsptr     存放最后一个必需代码单元标志的位置
    bcptr             指向当前分支链的指针
    cb                包含指向表格等的指针
    lengthptr         在实际编译阶段为NULL，在预编译阶段指向长度累加器

  返回值：
    0 出现错误，*errorcodeptr 非零
    +1 成功，此分支必须匹配至少一个字符
    -1 成功，此分支可以匹配空字符串
*/

static int
compile_branch(uint32_t *optionsptr, PCRE2_UCHAR **codeptr, uint32_t **pptrptr,
  int *errorcodeptr, uint32_t *firstcuptr, uint32_t *firstcuflagsptr,
  uint32_t *reqcuptr, uint32_t *reqcuflagsptr, branch_chain *bcptr,
  compile_block *cb, PCRE2_SIZE *lengthptr)
{
  int bravalue = 0;
  int okreturn = -1;
  int group_return = 0;
  uint32_t repeat_min = 0, repeat_max = 0;      /* 为了满足一些挑剔的编译器 */
  uint32_t greedy_default, greedy_non_default;
  uint32_t repeat_type, op_type;
  uint32_t options = *optionsptr;               /* 可能会动态改变 */
  uint32_t firstcu, reqcu;
  uint32_t zeroreqcu, zerofirstcu;
  uint32_t escape;
  uint32_t *pptr = *pptrptr;
  uint32_t meta, meta_arg;
  uint32_t firstcuflags, reqcuflags;
  uint32_t zeroreqcuflags, zerofirstcuflags;
  uint32_t req_caseopt, reqvary, tempreqvary;
  PCRE2_SIZE offset = 0;
  PCRE2_SIZE length_prevgroup = 0;
  PCRE2_UCHAR *code = *codeptr;
  PCRE2_UCHAR *last_code = code;
  PCRE2_UCHAR *orig_code = code;
  PCRE2_UCHAR *tempcode;
  PCRE2_UCHAR *previous = NULL;
  PCRE2_UCHAR op_previous;
*/
# 设置 groupsetfirstcu 变量为 FALSE
BOOL groupsetfirstcu = FALSE;
# 设置 had_accept 变量为 FALSE
BOOL had_accept = FALSE;
# 设置 matched_char 变量为 FALSE
BOOL matched_char = FALSE;
# 设置 previous_matched_char 变量为 FALSE
BOOL previous_matched_char = FALSE;
# 设置 reset_caseful 变量为 FALSE
BOOL reset_caseful = FALSE;
# 将 cb->cbits 赋值给 cbits 变量
const uint8_t *cbits = cb->cbits;
# 创建长度为 32 的 classbits 数组
uint8_t classbits[32];

# 如果支持 Unicode，则将 PCRE2_UTF 和 PCRE2_UCP 选项分别赋值给 utf 和 ucp 变量
#ifdef SUPPORT_UNICODE
BOOL utf = (options & PCRE2_UTF) != 0;
BOOL ucp = (options & PCRE2_UCP) != 0;
# 如果不支持 Unicode，则将 FALSE 赋值给 utf 变量
#else  /* No Unicode support */
BOOL utf = FALSE;
#endif

# 定义 class_uchardata 变量，始终赋值为 NULL
PCRE2_UCHAR *class_uchardata;
# 如果支持宽字符，则定义 xclass 和 class_uchardata_base 变量
#ifdef SUPPORT_WIDE_CHARS
BOOL xclass;
PCRE2_UCHAR *class_uchardata_base;
#endif

# 设置默认和非默认贪婪性设置
greedy_default = ((options & PCRE2_UNGREEDY) != 0);
greedy_non_default = greedy_default ^ 1;

# 初始化 firstcu、reqcu、zerofirstcu、zeroreqcu 变量为 0，以及对应的标志变量
firstcu = reqcu = zerofirstcu = zeroreqcu = 0;
firstcuflags = reqcuflags = zerofirstcuflags = zeroreqcuflags = REQ_UNSET;

# 变量 req_caseopt 包含 REQ_CASELESS 位或零，根据当前的大小写不敏感标志设置
# REQ_CASELESS 值
/* leaves the lower 28 bit empty. It is added into the firstcu or reqcu variables
to record the case status of the value. This is used only for ASCII characters.
*/

req_caseopt = ((options & PCRE2_CASELESS) != 0)? REQ_CASELESS : 0;

/* Switch on next META item until the end of the branch */

for (;; pptr++)
  {
#ifdef SUPPORT_WIDE_CHARS
  BOOL xclass_has_prop;
#endif
  BOOL negate_class;
  BOOL should_flip_negation;
  BOOL match_all_or_no_wide_chars;
  BOOL possessive_quantifier;
  BOOL note_group_empty;
  int class_has_8bitchar;
  uint32_t mclength;
  uint32_t skipunits;
  uint32_t subreqcu, subfirstcu;
  uint32_t groupnumber;
  uint32_t verbarglen, verbculen;
  uint32_t subreqcuflags, subfirstcuflags;
  open_capitem *oc;
  PCRE2_UCHAR mcbuffer[8];

  /* Get next META item in the pattern and its potential argument. */

  meta = META_CODE(*pptr);
  meta_arg = META_DATA(*pptr);

  /* If we are in the pre-compile phase, accumulate the length used for the
  previous cycle of this loop, unless the next item is a quantifier. */

  if (lengthptr != NULL)
    {
    if (code > cb->start_workspace + cb->workspace_size -
        WORK_SIZE_SAFETY_MARGIN)                       /* Check for overrun */
      {
      *errorcodeptr = (code >= cb->start_workspace + cb->workspace_size)?
        ERR52 : ERR86;
      return 0;
      }

    /* There is at least one situation where code goes backwards: this is the
    case of a zero quantifier after a class (e.g. [ab]{0}). When the quantifier
    is processed, the whole class is eliminated. However, it is created first,
    so we have to allow memory for it. Therefore, don't ever reduce the length
    at this point. */

    if (code < last_code) code = last_code;

    /* If the next thing is not a quantifier, we add the length of the previous
    item into the total, and reset the code pointer to the start of the
    workspace. Otherwise leave the previous item available to be quantified. */
    # 如果元字符的值小于 META_ASTERISK 或者大于 META_MINMAX_QUERY
    if (meta < META_ASTERISK || meta > META_MINMAX_QUERY)
      {
      # 如果长度指针的值加上代码长度超出了整型最大值
      if (OFLOW_MAX - *lengthptr < (PCRE2_SIZE)(code - orig_code))
        {
        *errorcodeptr = ERR20;   /* Integer overflow */  # 设置错误码为 ERR20，表示整数溢出
        return 0;  # 返回 0
        }
      *lengthptr += (PCRE2_SIZE)(code - orig_code);  # 更新长度指针的值
      # 如果长度指针的值超过了最大模式长度
      if (*lengthptr > MAX_PATTERN_SIZE)
        {
        *errorcodeptr = ERR20;   /* Pattern is too large */  # 设置错误码为 ERR20，表示模式太大
        return 0;  # 返回 0
        }
      code = orig_code;  # 重置代码指针的值为原始代码指针的值
      }

    # 记录当前代码项的起始位置，以便下次循环时检测是否“反向”
    last_code = code;
    }

  # 处理下一个解析的模式项。如果不是量词，则记住其起始位置，以便在后面跟随量词时进行量化。
  # 检查量词的合法性发生在 parse_regex() 中，除了在条件断言后面跟随量词的情况。
  if (meta < META_ASTERISK || meta > META_MINMAX_QUERY)
    {
    previous = code;  # 记录前一个代码项的位置
    if (matched_char && !had_accept) okreturn = 1;  # 如果匹配了字符并且没有接受，则设置 okreturn 为 1
    }

  previous_matched_char = matched_char;  # 记录前一个匹配的字符
  matched_char = FALSE;  # 重置匹配的字符标志为假
  note_group_empty = FALSE;  # 重置组为空的标志为假
  skipunits = 0;         # 大多数子组的默认值

  switch(meta)
    {
    # 分支在模式结束、| 或 ) 处终止
    case META_END:
    case META_ALT:
    case META_KET:
    *firstcuptr = firstcu;  # 更新第一个捕获指针的值
    *firstcuflagsptr = firstcuflags;  # 更新第一个捕获标志指针的值
    *reqcuptr = reqcu;  # 更新必需捕获指针的值
    *reqcuflagsptr = reqcuflags;  # 更新必需捕获标志指针的值
    *codeptr = code;  # 更新代码指针的值
    *pptrptr = pptr;  # 更新 pptr 指针的值
    return okreturn;  # 返回 okreturn

    # 处理单个字符元字符。在多行模式下，^ 禁用任何后续字符作为第一个字符的设置。
    case META_CIRCUMFLEX:
    if ((options & PCRE2_MULTILINE) != 0)
      {
      if (firstcuflags == REQ_UNSET)
        zerofirstcuflags = firstcuflags = REQ_NONE;  # 如果第一个捕获标志未设置，则将其设置为无
      *code++ = OP_CIRCM;  # 将 OP_CIRCM 添加到代码中
      }
    # 如果遇到 * ，则将 code 指针指向 OP_CIRC
    else *code++ = OP_CIRC;
    # 跳出 switch 语句
    break;

    # 如果遇到 $ ，则根据是否启用多行模式，将 code 指针指向 OP_DOLLM 或 OP_DOLL
    case META_DOLLAR:
    *code++ = ((options & PCRE2_MULTILINE) != 0)? OP_DOLLM : OP_DOLL;
    # 跳出 switch 语句
    break;

    # 如果遇到 . ，则根据是否启用 DOTALL 模式，将 code 指针指向 OP_ALLANY 或 OP_ANY
    case META_DOT:
    matched_char = TRUE;
    if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    zeroreqcu = reqcu;
    zeroreqcuflags = reqcuflags;
    *code++ = ((options & PCRE2_DOTALL) != 0)? OP_ALLANY: OP_ANY;
    # 跳出 switch 语句
    break;

    # 如果遇到空字符类，则根据是否启用 PCRE2_ALLOW_EMPTY_CLASS，将 code 指针指向 OP_ALLANY 或 OP_FAIL
    case META_CLASS_EMPTY:
    case META_CLASS_EMPTY_NOT:
    matched_char = TRUE;
    *code++ = (meta == META_CLASS_EMPTY_NOT)? OP_ALLANY : OP_FAIL;
    if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    # 跳出 switch 语句
    break;

    # 如果遇到非空字符类，则根据字符范围编译不同的操作码
    case META_CLASS:
    case META_CLASS_NOT:
    # 省略部分代码
    # ...
    # ...
    # ...
    /* 检查是否为 META_CLASS_NOT 或 META_CLASS */
    case META_CLASS_NOT:
    case META_CLASS:
    /* 匹配字符设为真 */
    matched_char = TRUE;
    /* 判断是否为 META_CLASS_NOT */
    negate_class = meta == META_CLASS_NOT;

    /* 如果字符类中只包含一个字符，可以优化为生成 OP_CHAR 或 OP_CHARI（如果是正的），或者生成 OP_NOT 或 OP_NOTI（如果是负的）。
    在负的情况下，如果这个项目是第一个，无论后面跟着什么重复计数，都不会有第一个字符。
    在 reqcu 的情况下，保存前一个值以便恢复。 */

    /* 注意：目前，如果在 32 位、非 UCP 模式下，字符类中唯一的字符的最高位被设置，这种优化是无效的。 */
    if (pptr[1] < META_END && pptr[2] == META_CLASS_END)
      {
#ifdef SUPPORT_UNICODE
      uint32_t d;  // 定义一个32位整数变量d，用于存储Unicode字符
#endif
      uint32_t c = pptr[1];  // 从指针pptr的第二个位置获取一个32位整数赋值给变量c

      pptr += 2;                 /* Move on to class end */  // 指针pptr向后移动2个位置，指向类结束位置
      if (meta == META_CLASS)    /* A positive one-char class can be */  // 如果meta等于META_CLASS
        {                        /* handled as a normal literal character. */  // 处理为普通的字面字符
        meta = c;                /* Set up the character */  // 将c赋值给meta，设置字符
        goto NORMAL_CHAR_SET;  // 跳转到NORMAL_CHAR_SET标签处
        }

      /* Handle a negative one-character class */

      zeroreqcu = reqcu;  // 将reqcu的值赋给zeroreqcu
      zeroreqcuflags = reqcuflags;  // 将reqcuflags的值赋给zeroreqcuflags
      if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;  // 如果firstcuflags等于REQ_UNSET，则将其设置为REQ_NONE
      zerofirstcu = firstcu;  // 将firstcu的值赋给zerofirstcu
      zerofirstcuflags = firstcuflags;  // 将firstcuflags的值赋给zerofirstcuflags

      /* For caseless UTF or UCP mode, check whether this character has more than one other case. If so, generate a special OP_NOTPROP item instead of OP_NOTI. */

#ifdef SUPPORT_UNICODE
      if ((utf||ucp) && (options & PCRE2_CASELESS) != 0 &&  // 如果支持Unicode或UCP，并且选项中包含PCRE2_CASELESS
          (d = UCD_CASESET(c)) != 0)  // 并且UCD_CASESET(c)不等于0，将UCD_CASESET(c)的值赋给d
        {
        *code++ = OP_NOTPROP;  // 将OP_NOTPROP添加到code指向的位置，然后code指针向后移动
        *code++ = PT_CLIST;  // 将PT_CLIST添加到code指向的位置，然后code指针向后移动
        *code++ = d;  // 将d添加到code指向的位置，然后code指针向后移动
        break;   /* We are finished with this class */  // 跳出循环，结束处理该类
        }
#endif
      /* Char has only one other case, or UCP not available */

      *code++ = ((options & PCRE2_CASELESS) != 0)? OP_NOTI: OP_NOT;  // 如果选项中包含PCRE2_CASELESS，则将OP_NOTI添加到code指向的位置，否则将OP_NOT添加到code指向的位置，然后code指针向后移动
      code += PUTCHAR(c, code);  // 调用PUTCHAR函数，将c添加到code指向的位置，然后code指针向后移动
      break;   /* We are finished with this class */  // 跳出循环，结束处理该类
      }        /* End of 1-char optimization */

    /* Handle character classes that contain more than just one literal character. If there are exactly two characters in a positive class, see if they are case partners. This can be optimized to generate a caseless single character match (which also sets first/required code units if relevant). */

    if (meta == META_CLASS && pptr[1] < META_END && pptr[2] < META_END &&  // 如果meta等于META_CLASS，并且pptr的第一个位置小于META_END，并且pptr的第二个位置小于META_END
        pptr[3] == META_CLASS_END)  // 并且pptr的第三个位置等于META_CLASS_END
      {
      uint32_t c = pptr[1];  // 从pptr的第一个位置获取一个32位整数赋值给变量c

#ifdef SUPPORT_UNICODE
      if (UCD_CASESET(c) == 0)  // 如果UCD_CASESET(c)等于0
#endif
        {
        uint32_t d;  // 定义一个32位整数变量d

#ifdef SUPPORT_UNICODE
        if ((utf || ucp) && c > 127) d = UCD_OTHERCASE(c); else  // 如果支持Unicode或UCP，并且c大于127，将UCD_OTHERCASE(c)的值赋给d，否则
#endif
          {
#if PCRE2_CODE_UNIT_WIDTH != 8
          // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则判断字符是否大于 255，如果是则赋值给 d，否则继续判断
          if (c > 255) d = c; else
#endif
          // 根据字符 c 在字符类 cb->fcc 中的位置获取对应的字符，赋值给 d
          d = TABLE_GET(c, cb->fcc, c);
          }

        // 如果字符 c 不等于 d 并且 pptr[2] 等于 d
        if (c != d && pptr[2] == d)
          {
          // 移动到字符类结束的位置
          pptr += 3;                 
          // 将 meta 设置为 c
          meta = c;
          // 如果选项中不包含 PCRE2_CASELESS，则设置 reset_caseful 为 TRUE，并将 PCRE2_CASELESS 加入选项中
          if ((options & PCRE2_CASELESS) == 0)
            {
            reset_caseful = TRUE;
            options |= PCRE2_CASELESS;
            req_caseopt = REQ_CASELESS;
            }
          // 跳转到 CLASS_CASELESS_CHAR 标签处
          goto CLASS_CASELESS_CHAR;
          }
        }
      }

    /* 如果非扩展字符类包含像 \S 这样的负特殊字符，我们需要在最后翻转否定标志，以便正确处理大于 255 的字符（它们都包含在字符类中）。
    扩展字符类可能需要插入特定的匹配或非匹配代码以匹配宽字符。 */

    should_flip_negation = match_all_or_no_wide_chars = FALSE;

    /* 当字符 > 255 时可能匹配时，将使用扩展字符类（xclass）。 */

#ifdef SUPPORT_WIDE_CHARS
    xclass = FALSE;
    // 为 XCLASS 项分配内存
    class_uchardata = code + LINK_SIZE + 2;   
    // 保存起始位置
    class_uchardata_base = class_uchardata;   
#endif

    /* 为了优化，我们跟踪字符类的一些属性：
    如果字符类包含至少一个小于 256 的字符，则 class_has_8bitchar 将为非零；如果字符类中存在 Unicode 属性检查，则 xclass_has_prop 将为 TRUE。 */

    class_has_8bitchar = 0;
#ifdef SUPPORT_WIDE_CHARS
    xclass_has_prop = FALSE;
#endif

    /* 将 256 位（32 字节）位图初始化为全零。我们在临时内存中构建位图，以防字符类包含少于两个 8 位字符，因为在这种情况下，编译后的代码不使用位图。 */

    memset(classbits, 0, 32 * sizeof(uint8_t));

    /* 处理项，直到达到 META_CLASS_END。 */
    # 当前元字符不是 META_CLASS_END 时循环执行
    while ((meta = *(++pptr)) != META_CLASS_END)
      {
      # 处理 POSIX 类，比如 [:alpha:] 等
      /* Handle POSIX classes such as [:alpha:] etc. */

      # 如果当前元字符是 META_POSIX 或 META_POSIX_NEG
      if (meta == META_POSIX || meta == META_POSIX_NEG)
        {
        # 判断是否为否定类
        BOOL local_negate = (meta == META_POSIX_NEG);
        # 获取 POSIX 类的值
        int posix_class = *(++pptr);
        # 定义变量
        int taboffset, tabopt;
        uint8_t pbits[32];

        # 根据是否为否定类来确定是否需要翻转否定
        should_flip_negation = local_negate;  /* Note negative special */

        # 如果匹配不区分大小写，并且 POSIX 类在表中的位置小于等于2，则将其转换为 alpha
        # 这依赖于类表以 alpha、lower、upper 作为前三个条目的事实
        if ((options & PCRE2_CASELESS) != 0 && posix_class <= 2)
          posix_class = 0;

        # 当 PCRE2_UCP 被设置时，一些 POSIX 类会被转换为使用 Unicode 属性 \p 或 \P 的不同转义序列
        # 其他一些无法通过 \p 或 \P 获得的 POSIX 类必须直接生成 XCL_PROP/XCL_NOTPROP，这在这里完成
        # 当 PCRE2_UCP 被设置时，一些 POSIX 类会被转换为使用 Unicode 属性 \p 或 \P 的不同转义序列
        # 其他一些无法通过 \p 或 \P 获得的 POSIX 类必须直接生成 XCL_PROP/XCL_NOTPROP，这在这里完成
#ifdef SUPPORT_UNICODE
        # 如果支持 Unicode，则根据选项和 POSIX 类别进行处理
        if ((options & PCRE2_UCP) != 0) switch(posix_class)
          {
          case PC_GRAPH:
          case PC_PRINT:
          case PC_PUNCT:
          *class_uchardata++ = local_negate? XCL_NOTPROP : XCL_PROP;
          *class_uchardata++ = (PCRE2_UCHAR)
            ((posix_class == PC_GRAPH)? PT_PXGRAPH :
             (posix_class == PC_PRINT)? PT_PXPRINT : PT_PXPUNCT);
          *class_uchardata++ = 0;
          xclass_has_prop = TRUE;
          # 转到 CONTINUE_CLASS 标签处继续处理
          goto CONTINUE_CLASS;

          /* 对于其他 POSIX 类别（ascii，xdigit），我们将转到非 UCP 情况，并为小于 256 的字符构建位图。然而，如果我们在一个否定的 POSIX 类别中，大于 255 的字符要么全部匹配，要么全部不匹配，取决于整个类别是否被否定。例如，对于 [[:^ascii:]... 它们必须全部匹配，而对于 [^[:^xdigit:]... 它们必须不匹配。

          在没有 xclass 项的特殊情况下，这会被 OP_CLASS 或 OP_NCLASS 的使用自动处理，但对于 OP_XCLASS，需要在后面生成范围。在 8 位库中，这仅在 utf 模式下才相关，因为否则不会存在宽字符。 */
          default:
#if PCRE2_CODE_UNIT_WIDTH == 8
          if (utf)
#endif
          # 如果在 utf 模式下，则匹配所有或不匹配所有宽字符
          match_all_or_no_wide_chars |= local_negate;
          break;
          }
#ifdef DEBUG_SHOW_PARSED
          # 输出未识别的解析模式项的调试信息
          fprintf(stderr, "** Unrecognized parsed pattern item 0x%.8x "
                          "in character class\n", meta);
#ifdef SUPPORT_UNICODE
          # 如果支持 Unicode
          case ESC_p:
          case ESC_P:
            {
            # 获取属性类型和属性数据
            uint32_t ptype = *(++pptr) >> 16;
            uint32_t pdata = *pptr & 0xffff;
            # 根据不同的转义符设置不同的属性类型
            *class_uchardata++ = (escape == ESC_p)? XCL_PROP : XCL_NOTPROP;
            *class_uchardata++ = ptype;
            *class_uchardata++ = pdata;
            xclass_has_prop = TRUE;
            class_has_8bitchar--;                /* 撤销! */
            }
          break;
#endif
          }

        # 转到继续处理类
        goto CONTINUE_CLASS;
        }  /* 结束处理 \d 类型转义 */

      /* 一个字面字符后面可能跟着一个范围元字符。在解析时，需要检查字符的顺序是否正确，范围是否相等，以及连字符是否能表示范围。因此，在这一点上，不需要进行检查。 */

      else
        {
        uint32_t c, d;

        CLASS_LITERAL:
        c = d = meta;

        /* 记住如果 \r 或 \n 被显式使用 */
        # 如果字符是 \r 或 \n，则设置外部标志
        if (c == CHAR_CR || c == CHAR_NL) cb->external_flags |= PCRE2_HASCRORLF;

        /* 处理字符范围 */
        # 如果下一个字符是范围元字符
        if (pptr[1] == META_RANGE_LITERAL || pptr[1] == META_RANGE_ESCAPED)
          {
#ifdef EBCDIC
          BOOL range_is_literal = (pptr[1] == META_RANGE_LITERAL);
#endif
          pptr += 2;
          d = *pptr;
          if (d == META_BIGVALUE) d = *(++pptr);

          /* 记住显式的 \r 或 \n，并将范围添加到类中。 */
          # 如果字符是 \r 或 \n，则设置外部标志
          if (d == CHAR_CR || d == CHAR_NL) cb->external_flags |= PCRE2_HASCRORLF;

          /* 在 EBCDIC 环境中，Perl 对字母范围进行特殊处理，因为编码中存在空缺，简单地使用范围 A-Z（例如）会包括空缺中的字符。这仅适用于字面范围；[\xC1-\xE9] 与 [A-Z] 不同。 */
#ifdef EBCDIC
          # 如果是 EBCDIC 编码
          if (range_is_literal &&
               (cb->ctypes[c] & ctype_letter) != 0 &&
               (cb->ctypes[d] & ctype_letter) != 0 &&
               (c <= CHAR_z) == (d <= CHAR_z))
            {
            uint32_t uc = (d <= CHAR_z)? 0 : 64;
            uint32_t C = c - uc;
            uint32_t D = d - uc;

            if (C <= CHAR_i)
              {
              # 如果字符在范围内，将字符添加到类别位图中
              class_has_8bitchar +=
                add_to_class(classbits, &class_uchardata, options, cb, C + uc,
                  ((D < CHAR_i)? D : CHAR_i) + uc);
              C = CHAR_j;
              }

            if (C <= D && C <= CHAR_r)
              {
              # 如果字符在范围内，将字符添加到类别位图中
              class_has_8bitchar +=
                add_to_class(classbits, &class_uchardata, options, cb, C + uc,
                  ((D < CHAR_r)? D : CHAR_r) + uc);
              C = CHAR_s;
              }

            if (C <= D)
              {
              # 如果字符在范围内，将字符添加到类别位图中
              class_has_8bitchar +=
                add_to_class(classbits, &class_uchardata, options, cb, C + uc,
                  D + uc);
              }
            }
          else
#endif
          /* 不是 EBCDIC 的特殊范围 */

          # 将字符添加到类别位图中
          class_has_8bitchar +=
            add_to_class(classbits, &class_uchardata, options, cb, c, d);
          # 跳转到下一个字符
          goto CONTINUE_CLASS;   /* 继续处理类别中的下一个字符 */
          }  /* 处理范围结束 */


        /* 处理单个字符 */

        # 将字符添加到类别位图中
        class_has_8bitchar +=
          add_to_class(classbits, &class_uchardata, options, cb, meta, meta);
        }

      /* 继续处理类别中的下一个项目 */

      CONTINUE_CLASS:
#ifdef SUPPORT_WIDE_CHARS
      /* 如果遇到任何宽字符或 Unicode 属性，设置 xclass 为 TRUE。
      然后，在预编译阶段，累积额外数据的长度并重置指针。
      这是为了避免包含大量宽字符或 Unicode 属性测试的非常大类别覆盖工作空间（位于堆栈上）。
      */
      if (class_uchardata > class_uchardata_base)
        {
        xclass = TRUE;
        if (lengthptr != NULL)
          {
          *lengthptr += class_uchardata - class_uchardata_base;
          class_uchardata = class_uchardata_base;
          }
        }
#endif

      continue;  /* 需要避免不支持宽字符时出现错误 */
      }   /* 主类处理循环结束 */

    /* 如果这个类是分支中的第一件事，无论重复次数如何，都不能有第一个字符设置。
    任何 reqcu 设置在任何重复后必须保持不变。
    */
    if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    zeroreqcu = reqcu;
    zeroreqcuflags = reqcuflags;

    /* 如果存在数值大于 255 的字符，或者 Unicode 属性设置（\p 或 \P），
    我们必须编译一个扩展类，带有自己的操作码，
    除非没有属性设置并且类中有否定的特殊字符，比如类中有 \S，并且未设置 PCRE2_UCP，
    因为在这种情况下，所有大于 255 的字符都在类中或不在类中，因此任何显式给出的字符也可以被忽略。
    */
    In the UCP case, if certain negated POSIX classes ([:^ascii:] or
    [^:xdigit:]) were present in a class, we either have to match or not match
    all wide characters (depending on whether the whole class is or is not
    negated). This requirement is indicated by match_all_or_no_wide_chars being
    true. We do this by including an explicit range, which works in both cases.
    */
    这仅适用于UTF和16位和32位的非UTF模式，因为在8位的非UTF模式中不能有任何宽字符。

    当在正UTF-8或任何16位或32位类中存在属性时，其中\S等在没有PCRE2_UCP的情况下会导致编译扩展类，我们确保所有大于255的字符都被包括在内，强制match_all_or_no_wide_chars为true。

    如果在生成xclass时没有字符<256，则我们可以在实际编译的代码中省略位图。*/
#ifdef SUPPORT_WIDE_CHARS  /* Defined for 16/32 bits, or 8-bit with Unicode */
    // 如果支持宽字符（16/32位），或者8位字符并且启用了Unicode支持，则执行以下代码
    if (xclass && (
#ifdef SUPPORT_UNICODE
        (options & PCRE2_UCP) != 0 ||
#endif
        xclass_has_prop || !should_flip_negation))
      {
      // 如果 xclass 存在，并且满足以下条件之一：启用了Unicode属性、xclass具有属性、或者不需要翻转否定
      if (match_all_or_no_wide_chars || (
#if PCRE2_CODE_UNIT_WIDTH == 8
           utf &&
#endif
           should_flip_negation && !negate_class && (options & PCRE2_UCP) == 0))
        {
        // 如果匹配所有或者没有宽字符，或者满足以下条件之一：8位字符并且启用了UTF支持、需要翻转否定并且不否定类、未启用Unicode属性
        *class_uchardata++ = XCL_RANGE;
        // 将 XCL_RANGE 写入 class_uchardata 指向的位置，并将指针向后移动
        if (utf)   /* Will always be utf in the 8-bit library */
          {
          // 如果启用了UTF支持，则执行以下代码（8位字符库中始终为UTF）
          class_uchardata += PRIV(ord2utf)(0x100, class_uchardata);
          // 调用 ord2utf 函数将0x100转换为UTF编码，并将指针向后移动
          class_uchardata += PRIV(ord2utf)(MAX_UTF_CODE_POINT, class_uchardata);
          // 调用 ord2utf 函数将MAX_UTF_CODE_POINT转换为UTF编码，并将指针向后移动
          }
        else       /* Can only happen for the 16-bit & 32-bit libraries */
          {
          // 否则，只会发生在16位和32位字符库中
#if PCRE2_CODE_UNIT_WIDTH == 16
          *class_uchardata++ = 0x100;
          *class_uchardata++ = 0xffffu;
#elif PCRE2_CODE_UNIT_WIDTH == 32
          *class_uchardata++ = 0x100;
          *class_uchardata++ = 0xffffffffu;
          // 根据字符宽度不同，将不同的值写入 class_uchardata 指向的位置，并将指针向后移动
#endif
#else
          }
        }
      *class_uchardata++ = XCL_END;    /* Marks the end of extra data */
      *code++ = OP_XCLASS;
      code += LINK_SIZE;
      *code = negate_class? XCL_NOT:0;
      if (xclass_has_prop) *code |= XCL_HASPROP;

      /* If the map is required, move up the extra data to make room for it;
      otherwise just move the code pointer to the end of the extra data. */

      if (class_has_8bitchar > 0)
        {
        *code++ |= XCL_MAP;
        (void)memmove(code + (32 / sizeof(PCRE2_UCHAR)), code,
          CU2BYTES(class_uchardata - code));
        if (negate_class && !xclass_has_prop)
          {
          /* Using 255 ^ instead of ~ avoids clang sanitize warning. */
          for (int i = 0; i < 32; i++) classbits[i] = 255 ^ classbits[i];
          }
        memcpy(code, classbits, 32);
        code = class_uchardata + (32 / sizeof(PCRE2_UCHAR));
        }
      else code = class_uchardata;

      /* Now fill in the complete length of the item */

      PUT(previous, 1, (int)(code - previous));
      break;   /* End of class handling */
      }
#endif  /* SUPPORT_WIDE_CHARS */

    /* If there are no characters > 255, or they are all to be included or
    excluded, set the opcode to OP_CLASS or OP_NCLASS, depending on whether the
    whole class was negated and whether there were negative specials such as \S
    (non-UCP) in the class. Then copy the 32-byte map into the code vector,
    negating it if necessary. */

    *code++ = (negate_class == should_flip_negation) ? OP_CLASS : OP_NCLASS;
    if (lengthptr == NULL)    /* Save time in the pre-compile phase */
      {
      if (negate_class)
        {
       /* Using 255 ^ instead of ~ avoids clang sanitize warning. */
       for (int i = 0; i < 32; i++) classbits[i] = 255 ^ classbits[i];
       }
      memcpy(code, classbits, 32);
      }
    code += 32 / sizeof(PCRE2_UCHAR);
    break;  /* End of class processing */
    # 处理(*VERB)s。

    # 在执行ACCEPT之前检查是否有未关闭的捕获，如果在同一断言级别内，则关闭这些捕获，并将ACCEPT转换为ASSERT_ACCEPT。在第一次遍历时，只累积所需的长度；否则，在许多嵌套括号内部执行(*ACCEPT)可能会导致工作空间溢出。不要在*ACCEPT之后设置firstcu。

    case META_ACCEPT:
    cb->had_accept = had_accept = TRUE;
    for (oc = cb->open_caps;
         oc != NULL && oc->assert_depth >= cb->assert_depth;
         oc = oc->next)
      {
      if (lengthptr != NULL)
        {
        *lengthptr += CU2BYTES(1) + IMM2_SIZE;
        }
      else
        {
        *code++ = OP_CLOSE;
        PUT2INC(code, 0, oc->number);
        }
      }
    *code++ = (cb->assert_depth > 0)? OP_ASSERT_ACCEPT : OP_ACCEPT;
    if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
    break;

    case META_PRUNE:
    case META_SKIP:
    cb->had_pruneorskip = TRUE;
    # 继续执行
    /* Fall through */
    case META_COMMIT:
    case META_FAIL:
    *code++ = verbops[(meta - META_MARK) >> 16];
    break;

    case META_THEN:
    cb->external_flags |= PCRE2_HASTHEN;
    *code++ = OP_THEN;
    break;

    # 处理带参数的动词。参数可能非常长，特别是在16位和32位模式下，可能会溢出工作空间。然而，参数长度受到限制，必须足够小以适应一个代码单元。这个检查发生在parse_regex()中。在第一次遍历时，我们只更新长度计数器并设置一个空参数，而不是将参数放入内存中。

    case META_THEN_ARG:
    cb->external_flags |= PCRE2_HASTHEN;
    goto VERB_ARG;

    case META_PRUNE_ARG:
    case META_SKIP_ARG:
    cb->had_pruneorskip = TRUE;
    # 继续执行
    /* Fall through */
    case META_MARK:
    case META_COMMIT_ARG:
    VERB_ARG:
    # 将指针指向的元数据减去 META_MARK，然后右移 16 位，得到的结果作为索引找到对应的操作符，赋值给 code 指针所指向的位置
    *code++ = verbops[(meta - META_MARK) >> 16];
    # 参数的长度以字符为单位
    /* The length is in characters. */
    verbarglen = *(++pptr);
    # 初始化当前参数的字符长度为 0
    verbculen = 0;
    # 将 code 指针的值赋给 tempcode，然后 code 指针自增
    tempcode = code++;
    # 遍历参数的每个字符
    for (int i = 0; i < (int)verbarglen; i++)
      {
      # 将指针指向的元数据赋值给 meta
      meta = *(++pptr);
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则根据 utf 标志将 meta 转换为 UTF 编码，得到 mcbuffer 的长度
      if (utf) mclength = PRIV(ord2utf)(meta, mcbuffer); else
#endif
        {
        # 否则，mclength 为 1，mcbuffer[0] 为 meta
        mclength = 1;
        mcbuffer[0] = meta;
        }
      # 如果 lengthptr 不为空，则将其指向的值加上 mclength；否则，将 mcbuffer 的内容复制到 code 中，并更新 code 指针和 verbculen
      if (lengthptr != NULL) *lengthptr += mclength; else
        {
        memcpy(code, mcbuffer, CU2BYTES(mclength));
        code += mclength;
        verbculen += mclength;
        }
      }

    *tempcode = verbculen;   /* 填充代码单元长度 */
    *code++ = 0;             /* 终止零 */
    break;


    /* ===================================================================*/
    /* 处理选项更改。必须将新设置传回以在后续分支中使用。重置 firstcu 和 reqcu 的贪婪默认值和大小写值。 */

    case META_OPTIONS:
    *optionsptr = options = *(++pptr);
    greedy_default = ((options & PCRE2_UNGREEDY) != 0);
    greedy_non_default = greedy_default ^ 1;
    req_caseopt = ((options & PCRE2_CASELESS) != 0)? REQ_CASELESS : 0;
    break;


    /* ===================================================================*/
    /* 处理条件子模式。 (?(Rdigits) 的情况是模棱两可的，因为它可能是对递归的数字检查，也可能是对设置组的名称检查。预处理设置 META_COND_RNUMBER 作为名称，以便我们可以以任何方式处理它。首先尝试名称；如果找不到，则处理数字。 */

    case META_COND_RNUMBER:   /* (?(Rdigits) */
    case META_COND_NAME:      /* (?(name) or (?'name') or ?(<name>) */
    case META_COND_RNAME:     /* (?(R&name) - test for recursion */
    goto GROUP_PROCESS_NOTE_EMPTY;

    /* DEFINE 条件始终为假。其内部组可能永远不会被调用，因此 matched_char 必须保持为假，因此跳转到 GROUP_PROCESS 而不是 GROUP_PROCESS_NOTE_EMPTY。 */

    case META_COND_DEFINE:
    bravalue = OP_COND;
    GETPLUSOFFSET(offset, pptr);
    code[1+LINK_SIZE] = OP_DEFINE;
    skipunits = 1;
    goto GROUP_PROCESS;
    # 条件测试一个组是否被设置
    case META_COND_NUMBER:
    bravalue = OP_COND;
    GETPLUSOFFSET(offset, pptr);
    groupnumber = *(++pptr);
    if (groupnumber > cb->bracount)
      {
      *errorcodeptr = ERR15;
      cb->erroroffset = offset;
      return 0;
      }
    if (groupnumber > cb->top_backref) cb->top_backref = groupnumber;
    offset -= 2;   # 指向初始 ( 以便报错
    code[1+LINK_SIZE] = OP_CREF;
    skipunits = 1+IMM2_SIZE;
    PUT2(code, 2+LINK_SIZE, groupnumber);
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 测试 PCRE2 版本
    case META_COND_VERSION:
    bravalue = OP_COND;
    if (pptr[1] > 0)
      code[1+LINK_SIZE] = ((PCRE2_MAJOR > pptr[2]) ||
        (PCRE2_MAJOR == pptr[2] && PCRE2_MINOR >= pptr[3]))?
          OP_TRUE : OP_FALSE;
    else
      code[1+LINK_SIZE] = (PCRE2_MAJOR == pptr[2] && PCRE2_MINOR == pptr[3])?
        OP_TRUE : OP_FALSE;
    skipunits = 1;
    pptr += 3;
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 条件是一个断言，可能在调用之前
    case META_COND_ASSERT:
    bravalue = OP_COND;
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 处理所有种类的嵌套括号组，非捕获，非条件情况在这里；其他情况通过 goto 到 GROUP_PROCESS 处理
    case META_LOOKAHEAD:
    bravalue = OP_ASSERT;
    cb->assert_depth += 1;
    goto GROUP_PROCESS;

    case META_LOOKAHEAD_NA:
    bravalue = OP_ASSERT_NA;
    cb->assert_depth += 1;
    goto GROUP_PROCESS;

    # 优化 (?!) 为 (*FAIL)，除非它被量化 - 这是一件奇怪的事情，但 Perl 允许所有断言被量化，当它们包含捕获括号时，可能会有潜在的用途。虽然这不适用于量化的 (?!)，但为了统一起见，我们允许它。
    case META_LOOKAHEADNOT:
    # 如果下一个字符是 META_KET 并且紧接着的字符不是 META_ASTERISK 或 META_MINMAX_QUERY
    if (pptr[1] == META_KET &&
         (pptr[2] < META_ASTERISK || pptr[2] > META_MINMAX_QUERY))
      {
      # 将 OP_FAIL 添加到代码中
      *code++ = OP_FAIL;
      # 移动指针到下一个字符
      pptr++;
      }
    else
      {
      # 设置 bravalue 为 OP_ASSERT_NOT
      bravalue = OP_ASSERT_NOT;
      # 增加断言深度
      cb->assert_depth += 1;
      # 跳转到 GROUP_PROCESS 标签处
      goto GROUP_PROCESS;
      }
    # 跳出 switch 语句
    break;

    # 如果遇到 META_LOOKBEHIND
    case META_LOOKBEHIND:
    # 设置 bravalue 为 OP_ASSERTBACK
    bravalue = OP_ASSERTBACK;
    # 增加断言深度
    cb->assert_depth += 1;
    # 跳转到 GROUP_PROCESS 标签处
    goto GROUP_PROCESS;

    # 如果遇到 META_LOOKBEHINDNOT
    case META_LOOKBEHINDNOT:
    # 设置 bravalue 为 OP_ASSERTBACK_NOT
    bravalue = OP_ASSERTBACK_NOT;
    # 增加断言深度
    cb->assert_depth += 1;
    # 跳转到 GROUP_PROCESS 标签处
    goto GROUP_PROCESS;

    # 如果遇到 META_LOOKBEHIND_NA
    case META_LOOKBEHIND_NA:
    # 设置 bravalue 为 OP_ASSERTBACK_NA
    bravalue = OP_ASSERTBACK_NA;
    # 增加断言深度
    cb->assert_depth += 1;
    # 跳转到 GROUP_PROCESS 标签处
    goto GROUP_PROCESS;

    # 如果遇到 META_ATOMIC
    case META_ATOMIC:
    # 设置 bravalue 为 OP_ONCE
    bravalue = OP_ONCE;
    # 跳转到 GROUP_PROCESS_NOTE_EMPTY 标签处
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 如果遇到 META_SCRIPT_RUN
    case META_SCRIPT_RUN:
    # 设置 bravalue 为 OP_SCRIPT_RUN
    bravalue = OP_SCRIPT_RUN;
    # 跳转到 GROUP_PROCESS_NOTE_EMPTY 标签处
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 如果遇到 META_NOCAPTURE
    case META_NOCAPTURE:
    # 设置 bravalue 为 OP_BRA
    bravalue = OP_BRA;
    # 继续执行下面的代码

    # 处理嵌套的括号正则表达式。维护嵌套深度以供 stackguard 函数使用。现在在 parse_regex() 中进行了对于嵌套深度过深的检测。断言和 DEFINE 组跳转到 GROUP_PROCESS；其他的跳转到 GROUP_PROCESS_NOTE_EMPTY，表示我们需要注意它们是否匹配空字符串。
    GROUP_PROCESS_NOTE_EMPTY:
    # 设置 note_group_empty 为 TRUE
    note_group_empty = TRUE;

    # 跳转到 GROUP_PROCESS 标签处
    GROUP_PROCESS:
    # 增加括号深度
    cb->parens_depth += 1;
    # 将 bravalue 添加到代码中
    *code = bravalue;
    # 移动指针到下一个字符
    pptr++;
    # 保存当前的 code
    tempcode = code;
    # 保存当前的 req_varyopt 值
    tempreqvary = cb->req_varyopt;        /* Save value before group */
    # 初始化 pre-compile 阶段的长度
    length_prevgroup = 0;                 /* Initialize for pre-compile phase */
    # 如果编译正则表达式的返回值为0，则表示出现错误，直接返回0
    if ((group_return =
         compile_regex(
         options,                         /* The option state */
         &tempcode,                       /* Where to put code (updated) */
         &pptr,                           /* Input pointer (updated) */
         errorcodeptr,                    /* Where to put an error message */
         skipunits,                       /* Skip over bracket number */
         &subfirstcu,                     /* For possible first char */
         &subfirstcuflags,
         &subreqcu,                       /* For possible last char */
         &subreqcuflags,
         bcptr,                           /* Current branch chain */
         cb,                              /* Compile data block */
         (lengthptr == NULL)? NULL :      /* Actual compile phase */
           &length_prevgroup              /* Pre-compile phase */
         )) == 0)
      return 0;  /* Error */

    # 减少括号的深度
    cb->parens_depth -= 1;

    # 如果刚刚编译的是一个非条件性的重要组（不是断言，也不是DEFINE），并且至少匹配了一个字符，则当前项匹配了一个字符
    if (note_group_empty && bravalue != OP_COND && group_return > 0)
      matched_char = TRUE;

    # 如果刚刚编译的是一个断言，减少断言深度
    if (bravalue >= OP_ASSERT && bravalue <= OP_ASSERTBACK_NA)
      cb->assert_depth -= 1;

    # 在编译结束时，code仍然指向组的开始，而tempcode已经更新为指向组的结尾。解析的模式指针（pptr）位于关闭的META_KET上。
    # 如果这是一个条件性的括号，则检查组中的分支不超过两个，或者如果是DEFINE组，则不超过一个。我们在实际编译阶段进行此操作，而不是在预处理阶段，因为整个组可能还不可用。
    # 如果条件为 OP_COND 并且 lengthptr 为空
    if (bravalue == OP_COND && lengthptr == NULL)
      {
      # 用于遍历代码的指针
      PCRE2_UCHAR *tc = code;
      # 条件计数器
      int condcount = 0;

      # 循环直到遇到 OP_KET 操作码
      do {
         condcount++;
         # 移动指针到下一个操作码
         tc += GET(tc,1);
         }
      while (*tc != OP_KET);

      # 如果条件为 OP_DEFINE 组，将其操作码改为 OP_FALSE
      if (code[LINK_SIZE+1] == OP_DEFINE)
        {
        # 如果条件数量大于 1，返回错误
        if (condcount > 1)
          {
          cb->erroroffset = offset;
          *errorcodeptr = ERR54;
          return 0;
          }
        # 将操作码改为 OP_FALSE
        code[LINK_SIZE+1] = OP_FALSE;
        # 设置 bravalue 为 OP_DEFINE，用于在下面禁止字符处理
        bravalue = OP_DEFINE;   /* A flag to suppress char handling below */
        }

      # 如果条件为普通条件组
      else
        {
        # 如果条件数量大于 2，返回错误
        if (condcount > 2)
          {
          cb->erroroffset = offset;
          *errorcodeptr = ERR27;
          return 0;
          }
        # 如果条件数量为 1，设置子模式的首选和必需标志为 REQ_NONE
        if (condcount == 1) subfirstcuflags = subreqcuflags = REQ_NONE;
        # 如果条件数量大于 1 且 group_return 大于 0，设置匹配字符标志为 TRUE
        else if (group_return > 0) matched_char = TRUE;
        }
      }

    # 在预编译阶段，更新组的长度，然后将编译后的代码减少为一组非捕获括号，以便在被量词复制时不占用太多内存
    # 如果长度指针不为空
    if (lengthptr != NULL)
      {
      # 如果当前长度加上前一个组的长度减去2减去2*LINK_SIZE超出了OFLOW_MAX
      if (OFLOW_MAX - *lengthptr < length_prevgroup - 2 - 2*LINK_SIZE)
        {
        # 设置错误码为ERR20
        *errorcodeptr = ERR20;
        # 返回0
        return 0;
        }
      # 更新长度指针，加上前一个组的长度减去2减去2*LINK_SIZE
      *lengthptr += length_prevgroup - 2 - 2*LINK_SIZE;
      # code指针自增1，这里已经包含了bravalue
      code++;   /* This already contains bravalue */
      # 在code指针位置插入指定长度和值的操作码
      PUTINC(code, 0, 1 + LINK_SIZE);
      # code指针自增1，插入OP_KET操作码
      *code++ = OP_KET;
      # 在code指针位置插入指定长度和值的操作码
      PUTINC(code, 0, 1 + LINK_SIZE);
      # 跳出循环，不需要处理特殊字符
      break;    /* No need to waste time with special character handling */
      }

    # 否则更新主代码指针到组的末尾
    code = tempcode;

    # 对于DEFINE组，不需要设置required和first字符
    if (bravalue == OP_DEFINE) break;

    # 处理其他类型组的required和first代码单元的更新
    # 更新正常括号和条件语句的required和first代码单元
    # 如果括号后面跟着一个重复次数为0的量词，需要回退
    # 因此在主循环之外定义zeroreqcu和zerofirstcu，以便在回退时访问它们
    zeroreqcu = reqcu;
    zeroreqcuflags = reqcuflags;
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    groupsetfirstcu = FALSE;
    # 如果 bralue 大于等于 OP_ONCE，则不是一个断言
    if (bravalue >= OP_ONCE)  
      {
      # 如果在这个分支中还没有设置 firstcu，从子模式中获取它，记住它是在这里设置的，以便重复超过一次时可以将其作为 reqcu 复制。如果子模式没有 firstcu，则为整个分支设置为 "none"。在这两种情况下，零重复将强制将 firstcu 设置为 "none"。
      if (firstcuflags == REQ_UNSET && subfirstcuflags != REQ_UNSET)
        {
        if (subfirstcuflags < REQ_NONE)
          {
          firstcu = subfirstcu;
          firstcuflags = subfirstcuflags;
          groupsetfirstcu = TRUE;
          }
        else firstcuflags = REQ_NONE;
        zerofirstcuflags = REQ_NONE;
        }

      # 如果之前设置了 firstcu，则将子模式的 firstcu 转换为 reqcu（如果没有设置），使用之前存在的 vary 标志。
      else if (subfirstcuflags < REQ_NONE && subreqcuflags >= REQ_NONE)
        {
        subreqcu = subfirstcu;
        subreqcuflags = subfirstcuflags | tempreqvary;
        }

      # 如果子模式设置了一个必需的代码单元（或者设置了一个不是真正的第一个代码单元的第一个代码单元 - 见上文），则设置它。
      if (subreqcuflags < REQ_NONE)
        {
        reqcu = subreqcu;
        reqcuflags = subreqcuflags;
        }
      }

    # 对于前向断言，如果设置了 reqcu，则取它，前提是组也设置了 firstcu。如果断言后面的模式没有设置不同的字符，这可能会有所帮助。例如，对于 /(?=abcde).+/。但是，我们不能为断言设置 firstcu，因为这会导致像 /(?=a)a.+/ 这样的模式产生错误的效果，其中 "真正的" "a" 然后会成为 reqcu 而不是 firstcu。如果没有设置 firstcu，则在最后进行扫描，寻找断言的第一个字符。对于像 /(?=.*X)X$ 这样的模式也有类似的效果
    # 只有在组也设置了第一个cu时，我们才需要采取reqcu。否则，在这个例子中，'X'最终会被设置为两者都适用。

    else if ((bravalue == OP_ASSERT || bravalue == OP_ASSERT_NA) &&
             subreqcuflags < REQ_NONE && subfirstcuflags < REQ_NONE)
      {
      reqcu = subreqcu;
      reqcuflags = subreqcuflags;
      }

    break;  # 嵌套组处理结束


    # 处理命名反向引用和递归。

    case META_BACKREF_BYNAME:
    break;


    # 处理数字调用。
    case META_CALLOUT_NUMBER:
    code[0] = OP_CALLOUT;
    PUT(code, 1, pptr[1]);               # 下一个模式项的偏移量
    PUT(code, 1 + LINK_SIZE, pptr[2]);   # 下一个模式项的长度
    code[1 + 2*LINK_SIZE] = pptr[3];
    pptr += 3;
    code += PRIV(OP_lengths)[OP_CALLOUT];
    break;


    # 处理带有字符串参数的调用。在预处理中，我们只计算长度而不生成任何内容。 pptr[3]中的长度包括两个定界符；在实际编译中，只复制第一个定界符，但会添加一个终止零。字符串中的任何重复定界符都会使其长度估计过长，但这并不值得担心。

    case META_CALLOUT_STRING:
    if (lengthptr != NULL)
      {
      *lengthptr += pptr[3] + (1 + 4*LINK_SIZE);
      pptr += 3;
      SKIPOFFSET(pptr);
      }

    # 在实际编译中，我们可以复制字符串。包括起始定界符，以便客户端在需要时可以发现它。我们还传递起始偏移量，以帮助脚本语言提供更好的错误消息。
    else
      {
      # 定义指针 pp，以及分隔符和长度变量
      PCRE2_SPTR pp;
      uint32_t delimiter;
      uint32_t length = pptr[3];
      # 获取调用字符串的指针
      PCRE2_UCHAR *callout_string = code + (1 + 4*LINK_SIZE);

      # 设置操作码为 OP_CALLOUT_STR
      code[0] = OP_CALLOUT_STR;
      # 设置下一个模式项的偏移量和长度
      PUT(code, 1, pptr[1]);               /* Offset to next pattern item */
      PUT(code, 1 + LINK_SIZE, pptr[2]);   /* Length of next pattern item */

      # 移动指针到下一个模式项
      pptr += 3;
      # 获取字符串在模式中的偏移量
      GETPLUSOFFSET(offset, pptr);         /* Offset to string in pattern */
      pp = cb->start_pattern + offset;
      delimiter = *callout_string++ = *pp++;
      # 如果分隔符是左花括号，则替换为右花括号
      if (delimiter == CHAR_LEFT_CURLY_BRACKET)
        delimiter = CHAR_RIGHT_CURLY_BRACKET;
      # 设置偏移量指向分隔符之后的位置
      PUT(code, 1 + 3*LINK_SIZE, (int)(offset + 1));  /* One after delimiter */

      /* The syntax of the pattern was checked in the parsing scan. The length
      includes both delimiters, but we have passed the opening one just above,
      so we reduce length before testing it. The test is for > 1 because we do
      not want to copy the final delimiter. This also ensures that pp[1] is
      accessible. */

      # 检查模式的语法，包括分隔符在内的长度
      while (--length > 1)
        {
        if (*pp == delimiter && pp[1] == delimiter)
          {
          *callout_string++ = delimiter;
          pp += 2;
          length--;
          }
        else *callout_string++ = *pp++;
        }
      *callout_string++ = CHAR_NUL;

      /* Set the length of the entire item, the advance to its end. */

      # 设置整个项的长度，并移动指针到末尾
      PUT(code, 1 + 2*LINK_SIZE, (int)(callout_string - code));
      code = callout_string;
      }
    break;


    /* ===================================================================*/
    /* Handle repetition. The different types are all sorted out in the parsing
    pass. */

    case META_MINMAX_PLUS:
    case META_MINMAX_QUERY:
    case META_MINMAX:
    # 获取重复的最小和最大次数
    repeat_min = *(++pptr);
    repeat_max = *(++pptr);
    # 跳转到重复处理的标签
    goto REPEAT;

    case META_ASTERISK:
    case META_ASTERISK_PLUS:
    case META_ASTERISK_QUERY:
    # 设置重复的最小和最大次数
    repeat_min = 0;
    repeat_max = REPEAT_UNLIMITED;
    # 跳转到重复处理的标签
    goto REPEAT;

    case META_PLUS:
    # 处理 META_PLUS_PLUS 和 META_PLUS_QUERY 情况
    case META_PLUS_PLUS:
    case META_PLUS_QUERY:
    # 重复次数的最小值为1
    repeat_min = 1;
    # 重复次数的最大值为无限
    repeat_max = REPEAT_UNLIMITED;
    # 跳转到 REPEAT 标签处执行

    # 处理 META_QUERY 情况
    case META_QUERY:
    # 处理 META_QUERY_PLUS 情况
    case META_QUERY_PLUS:
    # 处理 META_QUERY_QUERY 情况
    case META_QUERY_QUERY:
    # 重复次数的最小值为0
    repeat_min = 0;
    # 重复次数的最大值为1

    # REPEAT 标签处的逻辑
    REPEAT:
    # 如果前一个匹配的字符存在且重复次数的最小值大于0，则匹配字符为真
    if (previous_matched_char && repeat_min > 0) matched_char = TRUE;

    # 记录是否为可变长度重复，并默认为单字符操作码
    reqvary = (repeat_min == repeat_max)? 0 : REQ_VARY;
    op_type = 0;

    # 调整零重复的第一个和必需的代码单元
    if (repeat_min == 0)
      {
      firstcu = zerofirstcu;
      firstcuflags = zerofirstcuflags;
      reqcu = zeroreqcu;
      reqcuflags = zeroreqcuflags;
      }

    # 记录贪婪性和占有性
    switch (meta)
      {
      case META_MINMAX_PLUS:
      case META_ASTERISK_PLUS:
      case META_PLUS_PLUS:
      case META_QUERY_PLUS:
      # 重复类型为贪婪
      repeat_type = 0;                  
      possessive_quantifier = TRUE;
      break;

      case META_MINMAX_QUERY:
      case META_ASTERISK_QUERY:
      case META_PLUS_QUERY:
      case META_QUERY_QUERY:
      # 重复类型为非默认的贪婪
      repeat_type = greedy_non_default;
      possessive_quantifier = FALSE;
      break;

      default:
      # 重复类型为默认的贪婪
      repeat_type = greedy_default;
      possessive_quantifier = FALSE;
      break;
      }

    # 保存前一个项目的起始位置，以防我们需要将其移动到前面以插入内容，并记住它是什么
    tempcode = previous;
    op_previous = *previous;

    # 现在处理不同类型项目的重复。如果重复的最小值和最大值都为1，则对于非括号项，我们可以忽略量词，因为它们只有一个备选项。对于括号中的任何内容，如果{1}是占有的，我们不能忽略
    # 如果重复的最小值和最大值都为1，则对于非括号项，我们可以忽略量词，因为它们只有一个备选项。对于括号中的任何内容，如果{1}是占有的，我们不能忽略
    # 根据上一个操作符的类型进行不同的处理
    switch (op_previous)
      {
      /* 如果上一个操作符是字符或者否定字符匹配，取消该项并生成一个重复项。如果一个字符项的最小值大于一，确保它在 reqcu 中设置 - 如果一个序列如 x{3} 是分支中的第一件事，因为 x 将进入 firstcu 而不是 reqcu。 */

      case OP_CHAR:
      case OP_CHARI:
      case OP_NOT:
      case OP_NOTI:
      if (repeat_max == 1 && repeat_min == 1) goto END_REPEAT;
      op_type = chartypeoffset[op_previous - OP_CHAR];

      /* 处理占用多个代码单元的 UTF 字符。 */
#ifdef MAYBE_UTF_MULTI
      # 如果定义了 MAYBE_UTF_MULTI
      if (utf && NOT_FIRSTCU(code[-1]))
        {
        # 如果启用了 UTF，并且前一个字符不是第一个 UTF 字符
        PCRE2_UCHAR *lastchar = code - 1;
        BACKCHAR(lastchar);
        mclength = (uint32_t)(code - lastchar);   /* Length of UTF character */
        memcpy(mcbuffer, lastchar, CU2BYTES(mclength));  /* Save the char */
        }
      else
#endif  /* MAYBE_UTF_MULTI */

      /* 处理单个代码单元的情况 - 无论是没有 UTF 支持，还是禁用了 UTF，或者是单个代码单元的 UTF 字符。在后一种情况下，对于重复的正匹配，从前一个字符获取无大小写的标志，因为类似 [Aa] 这样的类设置了无大小写的 A，但是现在 req_caseopt 标志已经被重置。 */

        {
        mcbuffer[0] = code[-1];
        mclength = 1;
        if (op_previous <= OP_CHARI && repeat_min > 1)
          {
          reqcu = mcbuffer[0];
          reqcuflags = cb->req_varyopt;
          if (op_previous == OP_CHARI) reqcuflags |= REQ_CASELESS;
          }
        }
      goto OUTPUT_SINGLE_REPEAT;  /* Code shared with single character types */

      /* 如果前一个是字符类或反向引用，我们将重复的内容放在它后面，但如果重复是 {0,0}，则跳过该项。 */

#ifdef SUPPORT_WIDE_CHARS
      case OP_XCLASS:
    /* 如果重复后面的字符是 '+'，则 possessive_quantifier 为 TRUE。对于某些操作码，有特殊的替代操作码处理这种情况。对于其他情况，我们将整个重复的项包裹在 OP_ONCE 括号内。逻辑上，'+' 符号只是语法糖，取自 Sun 的 Java 包，但特殊的操作码可以对其进行优化。

    一些（但不是全部）具有贪婪重复的子模式已经在上面的代码中完全处理。对于它们，此时 possessive_quantifier 始终为 FALSE。请注意，重复的项从 tempcode 开始，而不是从 previous 开始，后者可能是字符串的第一部分，其
    /* 如果是占有量词，需要处理 */
    if (possessive_quantifier)
      {
      int len;

      /* 对于 EXACT 量词来说，占有化没有影响，所以我们可以忽略它。
      但是，QUERY、STAR 或 UPTO 可能会跟在后面（例如 {5,6}、{5,} 或 {5,10} 这样的量词）。
      我们跳过 EXACT 项；如果剩下的长度大于零，那么还有一个进一步处理的操作码。
      如果没有，就什么都不做，保持 EXACT 不变。 */

      switch(*tempcode)
        {
        case OP_TYPEEXACT:
        tempcode += PRIV(OP_lengths)[*tempcode] +
          ((tempcode[1 + IMM2_SIZE] == OP_PROP
          || tempcode[1 + IMM2_SIZE] == OP_NOTPROP)? 2 : 0);
        break;

        /* CHAR 操作码用于计数为 1 的 EXACT 量词。 */

        case OP_CHAR:
        case OP_CHARI:
        case OP_NOT:
        case OP_NOTI:
        case OP_EXACT:
        case OP_EXACTI:
        case OP_NOTEXACT:
        case OP_NOTEXACTI:
        tempcode += PRIV(OP_lengths)[*tempcode];
#ifdef SUPPORT_UNICODE
        # 如果支持 Unicode，并且当前字符是 UTF-8 编码的，跳过额外的长度
        if (utf && HAS_EXTRALEN(tempcode[-1]))
          tempcode += GET_EXTRALEN(tempcode[-1]);
#endif
        # 跳出循环
        break;

        /* 对于类操作码，重复操作符出现在末尾；
        调整 tempcode 指向它。 */

        case OP_CLASS:
        case OP_NCLASS:
        # 调整 tempcode 指向重复操作符
        tempcode += 1 + 32/sizeof(PCRE2_UCHAR);
        break;

#ifdef SUPPORT_WIDE_CHARS
        case OP_XCLASS:
        # 如果支持宽字符，根据指定的长度调整 tempcode 指向
        tempcode += GET(tempcode, 1);
        break;
#endif
        }

      /* 如果 tempcode 等于 code（指向重复项的末尾），意味着我们已经跳过了一个确切的项，但后面没有 QUERY、STAR 或 UPTO；len 的值将为 0，我们什么也不做。在所有其他情况下，tempcode 将指向重复操作符，并且小于 code，因此 len 的值将大于 0。 */

      len = (int)(code - tempcode);
      if (len > 0)
        {
        unsigned int repcode = *tempcode;

        /* 有一个用于 possessifying 操作码的表，所有这些操作码都小于 OP_CALLOUT。零条目表示没有 possessified 版本。 */

        if (repcode < OP_CALLOUT && opcode_possessify[repcode] > 0)
          *tempcode = opcode_possessify[repcode];

        /* 对于没有特殊 possessified 版本的操作码，将项目包装在 ONCE 括号中。 */

        else
          {
          (void)memmove(tempcode + 1 + LINK_SIZE, tempcode, CU2BYTES(len));
          code += 1 + LINK_SIZE;
          len += 1 + LINK_SIZE;
          tempcode[0] = OP_ONCE;
          *code++ = OP_KET;
          PUTINC(code, 0, len);
          PUT(tempcode, 1, len);
          }
        }
      }

    /* 如果之前遇到的 reqcus 没有设置“跟随变长字符串”标志，并且我们刚刚通过了一个变长项，则设置它。 */

    END_REPEAT:
    cb->req_varyopt |= reqvary;
    break;
    # 处理数值大于 META_END 的 32 位数据字符
    case META_BIGVALUE:
    # 指针向后移动一个位置
    pptr++;
    # 跳转到 NORMAL_CHAR 处理
    goto NORMAL_CHAR;

    # 处理数字引用，即元数据参数
    case META_BACKREF:
    # 如果 meta_arg 小于 10，则使用 cb->small_ref_offset[meta_arg] 作为偏移量
    if (meta_arg < 10) offset = cb->small_ref_offset[meta_arg];
      else GETPLUSOFFSET(offset, pptr);

    # 如果 meta_arg 大于 cb->bracount，则设置错误偏移量和错误码，返回 0
    if (meta_arg > cb->bracount)
      {
      cb->erroroffset = offset;
      *errorcodeptr = ERR15;  /* 不存在的子模式 */
      return 0;
      }

    # 处理单个引用
    HANDLE_SINGLE_REFERENCE:
    # 如果 firstcuflags 未设置，则将 zerofirstcuflags 和 firstcuflags 设置为 REQ_NONE
    if (firstcuflags == REQ_UNSET) zerofirstcuflags = firstcuflags = REQ_NONE;
    # 如果 options 包含 PCRE2_CASELESS，则将 OP_REFI 写入 code，否则写入 OP_REF
    *code++ = ((options & PCRE2_CASELESS) != 0)? OP_REFI : OP_REF;
    # 将 meta_arg 写入 code
    PUT2INC(code, 0, meta_arg);

    # 更新引用映射，并保持最高引用
    cb->backref_map |= (meta_arg < 32)? (1u << meta_arg) : 1;
    if (meta_arg > cb->top_backref) cb->top_backref = meta_arg;
    break;
    # 处理递归，通过在 OP_RECURSE 后插入被调用组的编号（即 meta 参数）来处理递归。在编译模式结束时，扫描这些数字，并将其替换为模式内的偏移量。这样做是为了避免前向引用的问题，并在组被复制和移动时调整偏移量（在以前的实现中发现的问题）。注意，递归没有设置第一个字符。
    case META_RECURSE:
    GETPLUSOFFSET(offset, pptr);
    if (meta_arg > cb->bracount)
      {
      cb->erroroffset = offset;
      *errorcodeptr = ERR15;  /* 不存在的子模式 */
      return 0;
      }
    HANDLE_NUMERICAL_RECURSION:
    *code = OP_RECURSE;
    PUT(code, 1, meta_arg);
    code += 1 + LINK_SIZE;
    groupsetfirstcu = FALSE;
    cb->had_recurse = TRUE;
    if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    break;

    # 处理捕获括号；数字是 meta 参数。
    case META_CAPTURE:
    bravalue = OP_CBRA;
    skipunits = IMM2_SIZE;
    PUT2(code, 1+LINK_SIZE, meta_arg);
    cb->lastcapture = meta_arg;
    goto GROUP_PROCESS_NOTE_EMPTY;

    # 处理转义序列项。对于像 \d 这样的转义序列，当 PCRE2_UCP 未设置时（这是唯一会出现在这里的情况），ESC_values 被安排为与默认情况下相应的 OP_values 相同。
    # 注意：\Q 和 \E 在这里从未出现过，因为它们在 parse_pattern() 中已处理。数值反向引用或递归也被转换为 META_BACKREF 或 META_RECURSE 项。当跟随名称时，\k 和 \g 被转换为 META_BACKREF_BYNAME 或 META_RECURSE_BYNAME。
    case META_ESCAPE:
    /* 如果元字符大于 ESC_b 并且小于 ESC_Z，则表示测试转义序列是否消耗一个字符，如果有新的转义序列被创建，这个范围可能需要改变。对于这些序列，如果第一个字符尚未设置，则禁用设置第一个字符。 */
    if (meta_arg > ESC_b && meta_arg < ESC_Z)
      {
      matched_char = TRUE;
      if (firstcuflags == REQ_UNSET) firstcuflags = REQ_NONE;
      }

    /* 如果后面跟着一个零重复，设置重置值。 */
    zerofirstcu = firstcu;
    zerofirstcuflags = firstcuflags;
    zeroreqcu = reqcu;
    zeroreqcuflags = reqcuflags;

    /* 如果不支持 Unicode，则不允许使用 \P 和 \p，并且在解析时会出错，因此这里永远不会出现这些情况。 */
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode
    if (meta_arg == ESC_P || meta_arg == ESC_p)
      {
      # 如果元参数是 ESC_P 或 ESC_p
      uint32_t ptype = *(++pptr) >> 16;
      # 从指针中获取属性类型
      uint32_t pdata = *pptr & 0xffff;
      # 从指针中获取属性数据

      /* The special case of \p{Any} is compiled to OP_ALLANY so as to benefit
      from the auto-anchoring code. */
      # 特殊情况下的 \p{Any} 被编译为 OP_ALLANY，以便从自动锚定代码中受益

      if (meta_arg == ESC_p && ptype == PT_ANY)
        {
        *code++ = OP_ALLANY;
        }
      else
        {
        *code++ = (meta_arg == ESC_p)? OP_PROP : OP_NOTPROP;
        *code++ = ptype;
        *code++ = pdata;
        }
      break;  /* End META_ESCAPE */
      }
#endif

    /* \K is forbidden in lookarounds since 10.38 because that's what Perl has
    done. However, there's an option, in case anyone was relying on it. */
    # \K 在环视中被禁止自 10.38 版本以来，因为 Perl 就是这样做的。但是，有一个选项，以防有人依赖它。

    if (cb->assert_depth > 0 && meta_arg == ESC_K &&
        (cb->cx->extra_options & PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK) == 0)
      {
      *errorcodeptr = ERR99;
      return 0;
      }

    /* For the rest (including \X when Unicode is supported - if not it's
    faulted at parse time), the OP value is the escape value when PCRE2_UCP is
    not set; if it is set, these escapes do not show up here because they are
    converted into Unicode property tests in parse_regex(). Note that \b and \B
    do a one-character lookbehind, and \A also behaves as if it does. */
    # 对于其余的情况（包括在支持 Unicode 时的 \X - 如果不支持，它在解析时会出错），当 PCRE2_UCP 没有设置时，OP 值是转义值；如果设置了，这些转义不会出现在这里，因为它们在 parse_regex() 中被转换为 Unicode 属性测试。注意 \b 和 \B 进行一个字符的回顾，\A 也会表现得像它一样。

    if (meta_arg == ESC_C) cb->external_flags |= PCRE2_HASBKC; /* Record */
    # 如果元参数是 ESC_C，则记录外部标志 PCRE2_HASBKC
    if ((meta_arg == ESC_b || meta_arg == ESC_B || meta_arg == ESC_A) &&
         cb->max_lookbehind == 0)
      cb->max_lookbehind = 1;
    # 如果元参数是 ESC_b、ESC_B 或 ESC_A，并且最大回顾长度为 0，则将最大回顾长度设置为 1

    /* In non-UTF mode, and for both 32-bit modes, we turn \C into OP_ALLANY
    instead of OP_ANYBYTE so that it works in DFA mode and in lookbehinds. */
    # 在非 UTF 模式下，以及两种 32 位模式下，我们将 \C 转换为 OP_ALLANY 而不是 OP_ANYBYTE，以便它在 DFA 模式和回顾中工作。

#if PCRE2_CODE_UNIT_WIDTH == 32
    *code++ = (meta_arg == ESC_C)? OP_ALLANY : meta_arg;
#else
    *code++ = (!utf && meta_arg == ESC_C)? OP_ALLANY : meta_arg;
#endif
    break;  /* End META_ESCAPE */


    /* ===================================================================*/
    /* 处理未识别的元值。解析的模式值小于 META_END 是一个字面值。否则我们有问题。 */

    // 默认情况
    default:
    // 如果元值大于等于 META_END
    if (meta >= META_END)
      {
#ifdef DEBUG_SHOW_PARSED
      // 如果定义了 DEBUG_SHOW_PARSED，则在标准错误流中输出未识别的解析模式项的信息
      fprintf(stderr, "** Unrecognized parsed pattern item 0x%.8x\n", *pptr);
#endif
      // 将错误码设置为 ERR89（内部错误 - 未识别）
      *errorcodeptr = ERR89;
      // 返回 0
      return 0;
      }

    /* 处理字面字符。在非 UTF 模式下，32 位的、值大于 META_END 的字符会跳转到这里。*/

    NORMAL_CHAR:
    // 获取完整的 32 位元数据
    meta = *pptr;
    NORMAL_CHAR_SET:  // 字符已经在元数据中
    // 匹配的字符为真
    matched_char = TRUE;

    /* 对于大小写不敏感的 UTF 或 UCP 模式，检查该字符是否有多个其他大小写形式。
    如果有，生成一个特殊的 OP_PROP 项，而不是 OP_CHARI。*/

#ifdef SUPPORT_UNICODE
    if ((utf||ucp) && (options & PCRE2_CASELESS) != 0)
      {
      // 获取字符的大小写形式集合
      uint32_t caseset = UCD_CASESET(meta);
      // 如果大小写形式集合不为空
      if (caseset != 0)
        {
        // 生成 OP_PROP 项
        *code++ = OP_PROP;
        *code++ = PT_CLIST;
        *code++ = caseset;
        // 如果首字符标志未设置，则设置为 REQ_NONE
        if (firstcuflags == REQ_UNSET)
          firstcuflags = zerofirstcuflags = REQ_NONE;
        // 结束处理该元数据项
        break;
        }
      }
#endif

    /* 大小写敏感的匹配，或者大小写不敏感但不是多重大小写字符之一。在正类包含只有两种大小写形式的字符的情况下，通过跳转到这里来设置 matched_char 为真，并在必要时调整选项。*/

    CLASS_CASELESS_CHAR:

    /* 将字符的代码单元存入 mcbuffer 中，长度存入 mclength。在非 UTF 模式下，长度始终为 1。*/

#ifdef SUPPORT_UNICODE
    if (utf) mclength = PRIV(ord2utf)(meta, mcbuffer); else
#endif
      {
      mclength = 1;
      mcbuffer[0] = meta;
      }

    /* 生成适当的代码*/

    *code++ = ((options & PCRE2_CASELESS) != 0)? OP_CHARI : OP_CHAR;
    memcpy(code, mcbuffer, CU2BYTES(mclength));
    code += mclength;

    /* 记住是否看到了 \r 或 \n */
    # 如果缓冲区的第一个字符是回车或换行符，则将外部标志设置为PCRE2_HASCRORLF
    if (mcbuffer[0] == CHAR_CR || mcbuffer[0] == CHAR_NL)
      cb->external_flags |= PCRE2_HASCRORLF;

    # 设置第一个和必需的代码单元。如果没有先前的第一个代码单元，则从这个字符设置它，但在零重复时恢复为无。否则，保持firstcu值不变，并且在零重复时不对其进行更改。
    if (firstcuflags == REQ_UNSET)
      {
      zerofirstcuflags = REQ_NONE;
      zeroreqcu = reqcu;
      zeroreqcuflags = reqcuflags;

      # 如果字符长度超过一个代码单元，只有在不进行无大小写匹配时，我们才能设置单个firstcu。多个可能的起始代码单元可以在后面的研究代码中捕获。
      if (mclength == 1 || req_caseopt == 0)
        {
        firstcu = mcbuffer[0];
        firstcuflags = req_caseopt;
        if (mclength != 1)
          {
          reqcu = code[-1];
          reqcuflags = cb->req_varyopt;
          }
        }
      else firstcuflags = reqcuflags = REQ_NONE;
      }

    # firstcu先前已设置；只有在长度为1或匹配大小写时，我们才能设置reqcu。
    else
      {
      zerofirstcu = firstcu;
      zerofirstcuflags = firstcuflags;
      zeroreqcu = reqcu;
      zeroreqcuflags = reqcuflags;
      if (mclength == 1 || req_caseopt == 0)
        {
        reqcu = code[-1];
        reqcuflags = req_caseopt | cb->req_varyopt;
        }
      }

    # 如果临时设置了无大小写匹配，则重置它。
    if (reset_caseful)
      {
      options &= ~PCRE2_CASELESS;
      req_caseopt = 0;
      reset_caseful = FALSE;
      }

    break;    /* 结束对字面字符的处理 */
    }         /* 大开关结束 */
  }           /* 大循环结束 */
/* Control never reaches here. */
/* 从不会到达这里。 */

/*************************************************
*   Compile regex: a sequence of alternatives    *
*************************************************/
/* 编译正则表达式：一系列的备选项 */

/* 在进入时，pptr 指向括号元字符之后，但返回时它指向闭括号或 META_END。code 变量指向 BRA 操作符存储的代码单元。
   此函数在预编译阶段用于查找所需内存量，以及在实际编译阶段使用。lengthptr 的值区分了这两个阶段。

   参数：
   options           选项位，包括此子模式的任何更改
   codeptr           -> 当前代码指针的地址
   pptrptr           -> 当前解析模式指针的地址
   errorcodeptr      -> 指向错误代码变量的指针
   skipunits         跳过这么多个代码单元（用于括号和 OP_COND）
   firstcuptr        放置第一个必需代码单元的位置
   firstcuflagsptr   放置第一个代码单元标志的位置
   reqcuptr          放置最后一个必需代码单元的位置
   reqcuflagsptr     放置最后一个必需代码单元标志的位置
   bcptr             指向当前打开分支链的指针
   cb                指向具有表指针等数据块的指针
   lengthptr         在实际编译阶段为 NULL
                     在预编译阶段指向长度累加器的指针

   返回值：           
   0 出现错误
   +1 成功，此组必须匹配至少一个字符
   -1 成功，此组可以匹配空字符串
*/
static int
compile_regex(uint32_t options, PCRE2_UCHAR **codeptr, uint32_t **pptrptr,
  int *errorcodeptr, uint32_t skipunits, uint32_t *firstcuptr,
  uint32_t *firstcuflagsptr, uint32_t *reqcuptr, uint32_t *reqcuflagsptr,
  branch_chain *bcptr, compile_block *cb, PCRE2_SIZE *lengthptr)
{
PCRE2_UCHAR *code = *codeptr;
# 定义指向正则表达式代码的指针
PCRE2_UCHAR *last_branch = code;
# 定义指向正则表达式起始括号的指针
PCRE2_UCHAR *start_bracket = code;
# 定义一个布尔变量，表示是否为后向断言
BOOL lookbehind;
# 定义一个用于保存捕获组信息的结构体
open_capitem capitem;
# 初始化捕获组编号
int capnumber = 0;
# 初始化成功返回值
int okreturn = 1;
# 定义指向指针的指针
uint32_t *pptr = *pptrptr;
# 初始化一些用于后续操作的变量
uint32_t firstcu, reqcu;
uint32_t lookbehindlength;
uint32_t firstcuflags, reqcuflags;
uint32_t branchfirstcu, branchreqcu;
uint32_t branchfirstcuflags, branchreqcuflags;
# 初始化一个用于保存长度的变量
PCRE2_SIZE length;
# 定义一个用于保存分支链信息的结构体
branch_chain bc;

# 如果设置了栈保护函数，并且栈保护函数返回真，则返回错误码
if (cb->cx->stack_guard != NULL &&
    cb->cx->stack_guard(cb->parens_depth, cb->cx->stack_guard_data))
  {
  *errorcodeptr= ERR33;
  return 0;
  }

# 杂项初始化

# 初始化分支链的外部指针
bc.outer = bcptr;
# 初始化当前分支的指针
bc.current_branch = code;

# 初始化一些用于预编译阶段的变量
firstcu = reqcu = 0;
firstcuflags = reqcuflags = REQ_UNSET;

# 累积长度，用于预编译阶段。从BRA和KET的长度开始，以及任何额外的代码单元
length = 2 + 2*LINK_SIZE + skipunits;

# 记录是否为后向断言，如果是，则保存其长度并跳过模式偏移
lookbehind = *code == OP_ASSERTBACK ||
             *code == OP_ASSERTBACK_NOT ||
             *code == OP_ASSERTBACK_NA;

if (lookbehind)
  {
  lookbehindlength = META_DATA(pptr[-1]);
  pptr += SIZEOFFSET;
  }
else lookbehindlength = 0;

# 如果这是一个捕获子模式，则添加到打开捕获项链中，以便在遇到(*ACCEPT)时检测到它们
# 如果当前操作码是 OP_CBRA
if (*code == OP_CBRA)
  {
  # 从代码中获取捕获组编号
  capnumber = GET2(code, 1 + LINK_SIZE);
  # 设置捕获项的编号和断言深度
  capitem.number = capnumber;
  capitem.next = cb->open_caps;
  capitem.assert_depth = cb->assert_depth;
  # 将捕获项添加到打开的捕获组链表中
  cb->open_caps = &capitem;
  }

# 将偏移量设置为零，标记该括号仍然是打开的
PUT(code, 1, 0);
code += 1 + LINK_SIZE + skipunits;

# 为每个替代分支循环
for (;;)
  {
  int branch_return;

  # 如果这是回顾断言，插入 OP_REVERSE
  if (lookbehind && lookbehindlength > 0)
    {
    *code++ = OP_REVERSE;
    PUTINC(code, 0, lookbehindlength);
    length += 1 + LINK_SIZE;
    }

  # 现在编译分支；在预编译阶段，其长度被添加到长度中
  if ((branch_return =
        compile_branch(&options, &code, &pptr, errorcodeptr, &branchfirstcu,
          &branchfirstcuflags, &branchreqcu, &branchreqcuflags, &bc,
          cb, (lengthptr == NULL)? NULL : &length)) == 0)
    return 0;

  # 如果一个分支可以匹配空字符串，整个组也可以
  if (branch_return < 0) okreturn = -1;

  # 在实际编译阶段，需要进行一些后处理
  if (lengthptr == NULL)
    {
    # 如果这是第一个分支，分支的第一个cu和reqcu值成为正则表达式的值
    if (*last_branch != OP_ALT)
      {
      firstcu = branchfirstcu;
      firstcuflags = branchfirstcuflags;
      reqcu = branchreqcu;
      reqcuflags = branchreqcuflags;
      }

    # 如果这不是第一个分支，第一个字符和reqcu必须与所有先前分支的值匹配，
    # 除非先前的reqcu值没有设置REQ_VARY，它仍然可以匹配，并且我们为该分支的值设置REQ_VARY
    else
      {
      /* 如果之前有一个 firstcu，但它与新分支不匹配，
      我们必须放弃 firstcu 对于正则表达式，但如果之前没有 reqcu，它将取代旧的 firstcu 的值。 */

      if (firstcuflags != branchfirstcuflags || firstcu != branchfirstcu)
        {
        if (firstcuflags < REQ_NONE)
          {
          if (reqcuflags >= REQ_NONE)
            {
            reqcu = firstcu;
            reqcuflags = firstcuflags;
            }
          }
        firstcuflags = REQ_NONE;
        }

      /* 如果我们（现在或之前）没有 firstcu，分支中的 firstcu 如果没有分支 reqcu，就变成 reqcu。 */

      if (firstcuflags >= REQ_NONE && branchfirstcuflags < REQ_NONE &&
          branchreqcuflags >= REQ_NONE)
        {
        branchreqcu = branchfirstcu;
        branchreqcuflags = branchfirstcuflags;
        }

      /* 现在确保 reqcus 匹配 */

      if (((reqcuflags & ~REQ_VARY) != (branchreqcuflags & ~REQ_VARY)) ||
          reqcu != branchreqcu)
        reqcuflags = REQ_NONE;
      else
        {
        reqcu = branchreqcu;
        reqcuflags |= branchreqcuflags; /* 如果存在，将 REQ_VARY "or" 进去 */
        }
      }
    }

  /* 处理达到表达式的结尾，即 ')' 或模式的结尾。
  在真正的编译阶段，回到替代分支并反转偏移链，BRA 项中的字段现在变成第一个替代的偏移量。
  如果没有替代项，它指向组的结尾。在结束的 ket 中的长度始终是括号项的长度。
  返回时将指针留在终止字符处。 */

  if (META_CODE(*pptr) != META_ALT)
    {
    # 如果长度指针为空
    if (lengthptr == NULL)
      {
      # 计算分支长度
      PCRE2_SIZE branch_length = code - last_branch;
      # 循环处理
      do
        {
        # 获取上一个分支的长度
        PCRE2_SIZE prev_length = GET(last_branch, 1);
        # 将当前分支的长度填入上一个分支的长度位置
        PUT(last_branch, 1, branch_length);
        # 更新分支长度为上一个分支的长度
        branch_length = prev_length;
        # 更新最后一个分支的位置
        last_branch -= branch_length;
        }
      while (branch_length > 0);
      }

    # 填充 ket

    *code = OP_KET;
    # 将当前位置到起始括号的偏移量填入长度字段
    PUT(code, 1, (int)(code - start_bracket));
    # 移动代码指针
    code += 1 + LINK_SIZE;

    # 如果是捕获子模式，从链表中移除该块
    if (capnumber > 0) cb->open_caps = cb->open_caps->next;

    # 设置要返回的值

    *codeptr = code;
    *pptrptr = pptr;
    *firstcuptr = firstcu;
    *firstcuflagsptr = firstcuflags;
    *reqcuptr = reqcu;
    *reqcuflagsptr = reqcuflags;
    # 如果长度指针不为空
    if (lengthptr != NULL)
      {
      # 检查长度是否会溢出
      if (OFLOW_MAX - *lengthptr < length)
        {
        *errorcodeptr = ERR20;
        return 0;
        }
      # 更新长度
      *lengthptr += length;
      }
    # 返回成功
    return okreturn;
    }

  # 另一个分支跟随。在预编译阶段，我们可以将代码指针移回到第一个分支的起始位置。（也就是假装每个分支都是唯一的。）

  # 在实际编译阶段，插入一个 ALT 节点。它的长度字段指向上一个分支，而括号保持打开状态。最后链表被反转。这样做是为了使括号的起始位置在关闭之前具有零偏移量，从而可以检测递归。

  # 如果长度指针不为空
  if (lengthptr != NULL)
    {
    # 移动代码指针
    code = *codeptr + 1 + LINK_SIZE + skipunits;
    # 更新长度
    length += 1 + LINK_SIZE;
    }
  else
    {
    # 设置 ALT 节点
    *code = OP_ALT;
    # 将当前位置到上一个分支的偏移量填入长度字段
    PUT(code, 1, (int)(code - last_branch));
    # 更新当前分支的位置
    bc.current_branch = last_branch = code;
    # 移动代码指针
    code += 1 + LINK_SIZE;
    }

  # 设置回顾长度（如果不在回顾中，值将为零），然后继续垂直线之后的操作
  lookbehindlength = META_DATA(*pptr);
  pptr++;
  }
/* Control never reaches here */
}

/*************************************************
*          Check for anchored pattern            *
*************************************************/

/* Try to find out if this is an anchored regular expression. Consider each
alternative branch. If they all start with OP_SOD or OP_CIRC, or with a bracket
all of whose alternatives start with OP_SOD or OP_CIRC (recurse ad lib), then
it's anchored. However, if this is a multiline pattern, then only OP_SOD will
be found, because ^ generates OP_CIRCM in that mode.

We can also consider a regex to be anchored if OP_SOM starts all its branches.
This is the code for \G, which means "match at start of match position, taking
into account the match offset".

A branch is also implicitly anchored if it starts with .* and DOTALL is set,
because that will try the rest of the pattern at all possible matching points,
so there is no point trying again.... er ....

.... except when the .* appears inside capturing parentheses, and there is a
subsequent back reference to those parentheses. We haven't enough information
to catch that case precisely.

At first, the best we could do was to detect when .* was in capturing brackets
and the highest back reference was greater than or equal to that level.
However, by keeping a bitmap of the first 31 back references, we can catch some
of the more common cases more precisely.

... A second exception is when the .* appears inside an atomic group, because
this prevents the number of characters it matches from being adjusted.

Arguments:
  code           points to start of the compiled pattern
  bracket_map    a bitmap of which brackets we are inside while testing; this
                   handles up to substring 31; after that we just have to take
                   the less precise approach
  cb             points to the compile data block
  atomcount      atomic group level
  inassert       TRUE if in an assertion

Returns:     TRUE or FALSE
*/
# 检查是否每个分支都以 ^ 或 .* 开头，以便在多行匹配和非 DOTALL 模式下的 .* 开头的模式中进行“首字符”处理以加快速度
# 与 is_anchored() 类似，需要考虑包含 .* 的捕获括号的反向引用，以及原子括号或断言中的 .* 的出现，或者包含 *PRUNE 或 *SKIP 的模式
# 参数：
#   code：指向编译模式或组的起始点
#   bracket_map：测试时所在的括号位图；处理最多 31 个子字符串；之后就只能采取不太精确的方法
#   cb：指向编译数据
#   atomcount：原子组级别
#   inassert：如果在断言中，则为 TRUE
# 返回：TRUE 或 FALSE
static BOOL
is_startline(PCRE2_SPTR code, unsigned int bracket_map, compile_block *cb,
  int atomcount, BOOL inassert)
{
    while (*code == OP_ALT);  # 循环处理每个分支
    return TRUE;
}

# 这个函数扫描编译模式，直到找到 OP_RECURSE 的实例
# 参数：
#   code：指向表达式的起始点
#   utf：在 UTF 模式下为 TRUE
# 返回：指向 OP_RECURSE 的操作码的指针，如果未找到则返回 NULL
static BOOL
is_startline(PCRE2_SPTR code, unsigned int bracket_map, compile_block *cb,
  int atomcount, BOOL inassert)
{
    while (*code == OP_ALT);  # 循环处理每个分支
    return TRUE;
}
/*
这是一个静态函数，用于在给定的正则表达式代码中查找递归调用的位置
*/
static PCRE2_SPTR
find_recurse(PCRE2_SPTR code, BOOL utf)
{
/*
无限循环，用于遍历给定的正则表达式代码
*/
for (;;)
  {
  /*
  获取当前字符
  */
  PCRE2_UCHAR c = *code;
  /*
  如果当前字符是结束符，则返回空指针
  */
  if (c == OP_END) return NULL;
  /*
  如果当前字符是递归调用操作符，则返回当前位置
  */
  if (c == OP_RECURSE) return code;

  /*
  XCLASS 用于表示无法仅通过位图表示的类，包括否定的单个高值字符。CALLOUT_STR 用于带有字符串参数的调用。在这两种情况下，表中的长度为零；实际长度存储在编译代码中。
  */
  if (c == OP_XCLASS) code += GET(code, 1);
    else if (c == OP_CALLOUT_STR) code += GET(code, 1 + 2*LINK_SIZE);

  /*
  否则，我们可以从表中获取项目的长度，除了对于重复的字符类型，我们必须测试 \p 和 \P，它们具有额外的两个代码单元的参数，并且对于带有参数的 MARK/PRUNE/SKIP/THEN，我们必须添加其长度。
  */
  else
    {
    switch(c)
      {
      case OP_TYPESTAR:
      case OP_TYPEMINSTAR:
      case OP_TYPEPLUS:
      case OP_TYPEMINPLUS:
      case OP_TYPEQUERY:
      case OP_TYPEMINQUERY:
      case OP_TYPEPOSSTAR:
      case OP_TYPEPOSPLUS:
      case OP_TYPEPOSQUERY:
      if (code[1] == OP_PROP || code[1] == OP_NOTPROP) code += 2;
      break;

      case OP_TYPEPOSUPTO:
      case OP_TYPEUPTO:
      case OP_TYPEMINUPTO:
      case OP_TYPEEXACT:
      if (code[1 + IMM2_SIZE] == OP_PROP || code[1 + IMM2_SIZE] == OP_NOTPROP)
        code += 2;
      break;

      case OP_MARK:
      case OP_COMMIT_ARG:
      case OP_PRUNE_ARG:
      case OP_SKIP_ARG:
      case OP_THEN_ARG:
      code += code[1];
      break;
      }

    /*
    从表中添加固定长度
    */
    code += PRIV(OP_lengths)[c];

    /*
    在 UTF-8 和 UTF-16 模式下，后面跟着字符的操作码可能后面跟着多单元字符。表中的长度是最小值，因此我们必须安排跳过额外的单元。
    */
#ifdef MAYBE_UTF_MULTI
    # 如果 utf 为真，则根据 c 的值进行不同的操作
    if (utf) switch(c)
      {
      # 根据不同的操作码进行不同的处理
      case OP_CHAR:
      case OP_CHARI:
      case OP_NOT:
      case OP_NOTI:
      case OP_EXACT:
      case OP_EXACTI:
      case OP_NOTEXACT:
      case OP_NOTEXACTI:
      case OP_UPTO:
      case OP_UPTOI:
      case OP_NOTUPTO:
      case OP_NOTUPTOI:
      case OP_MINUPTO:
      case OP_MINUPTOI:
      case OP_NOTMINUPTO:
      case OP_NOTMINUPTOI:
      case OP_POSUPTO:
      case OP_POSUPTOI:
      case OP_NOTPOSUPTO:
      case OP_NOTPOSUPTOI:
      case OP_STAR:
      case OP_STARI:
      case OP_NOTSTAR:
      case OP_NOTSTARI:
      case OP_MINSTAR:
      case OP_MINSTARI:
      case OP_NOTMINSTAR:
      case OP_NOTMINSTARI:
      case OP_POSSTAR:
      case OP_POSSTARI:
      case OP_NOTPOSSTAR:
      case OP_NOTPOSSTARI:
      case OP_PLUS:
      case OP_PLUSI:
      case OP_NOTPLUS:
      case OP_NOTPLUSI:
      case OP_MINPLUS:
      case OP_MINPLUSI:
      case OP_NOTMINPLUS:
      case OP_NOTMINPLUSI:
      case OP_POSPLUS:
      case OP_POSPLUSI:
      case OP_NOTPOSPLUS:
      case OP_NOTPOSPLUSI:
      case OP_QUERY:
      case OP_QUERYI:
      case OP_NOTQUERY:
      case OP_NOTQUERYI:
      case OP_MINQUERY:
      case OP_MINQUERYI:
      case OP_NOTMINQUERY:
      case OP_NOTMINQUERYI:
      case OP_POSQUERY:
      case OP_POSQUERYI:
      case OP_NOTPOSQUERY:
      case OP_NOTPOSQUERYI:
      # 如果操作码的最后一个字节有额外长度，则将 code 指针向后移动额外长度的字节数
      if (HAS_EXTRALEN(code[-1])) code += GET_EXTRALEN(code[-1]);
      break;
      }
#else
    (void)(utf);  /* Keep compiler happy by referencing function argument */
#endif  /* MAYBE_UTF_MULTI */
    }
  }
}

/*************************************************
*    Check for asserted fixed first code unit    *
*************************************************/

/* 在编译期间，正向断言的“第一个代码单元”设置被丢弃，因为它们可能与后续的实际文字发生冲突。然而，如果我们最终没有为未锚定的模式设置第一个代码单元，那么值得扫描正则表达式，看看是否有一个初始断言的第一个代码单元。如果所有分支都以相同的断言代码单元开头，或者以非条件括号开头，其所有替代方案都以相同的断言代码单元开头（递归自由），那么我们返回该代码单元，标志设置为零或REQ_CASELESS；否则返回零，标志为REQ_NONE。

参数：
  code       指向编译模式的起始点
  flags      指向第一个代码单元的标志
  inassert   如果在断言中，则为非零值

返回值：     固定的第一个代码单元，或者标志为REQ_NONE的零
*/

static uint32_t
find_firstassertedcu(PCRE2_SPTR code, uint32_t *flags, uint32_t inassert)
{
uint32_t c = 0;
uint32_t cflags = REQ_NONE;

*flags = REQ_NONE;
do {
   // 定义变量d和dflags
   uint32_t d;
   uint32_t dflags;
   // 如果当前操作码是OP_CBRA、OP_SCBRA、OP_CBRAPOS或OP_SCBRAPOS，则xl为IMM2_SIZE，否则为0
   int xl = (*code == OP_CBRA || *code == OP_SCBRA ||
             *code == OP_CBRAPOS || *code == OP_SCBRAPOS)? IMM2_SIZE:0;
   // 获取下一个重要的操作码的指针
   PCRE2_SPTR scode = first_significant_code(code + 1+LINK_SIZE + xl, TRUE);
   // 获取操作码
   PCRE2_UCHAR op = *scode;

   switch(op)
     {
     default:
     // 默认情况下返回0
     return 0;

     case OP_BRA:
     case OP_BRAPOS:
     case OP_CBRA:
     case OP_SCBRA:
     case OP_CBRAPOS:
     case OP_SCBRAPOS:
     case OP_ASSERT:
     case OP_ASSERT_NA:
     case OP_ONCE:
     case OP_SCRIPT_RUN:
     // 查找第一个断言的代码单元
     d = find_firstassertedcu(scode, &dflags, inassert +
       ((op == OP_ASSERT || op == OP_ASSERT_NA)?1:0));
     // 如果dflags大于等于REQ_NONE，则返回0
     if (dflags >= REQ_NONE) return 0;
     // 如果cflags大于等于REQ_NONE，则将c设置为d，cflags设置为dflags；否则如果c不等于d或cflags不等于dflags，则返回0
     if (cflags >= REQ_NONE) { c = d; cflags = dflags; }
       else if (c != d || cflags != dflags) return 0;
     break;

     case OP_EXACT:
     // scode指针向后移动IMM2_SIZE个位置
     scode += IMM2_SIZE;
     /* Fall through */

     case OP_CHAR:
     case OP_PLUS:
     case OP_MINPLUS:
     case OP_POSPLUS:
     // 如果不在断言内部，则返回0
     if (inassert == 0) return 0;
     // 如果cflags大于等于REQ_NONE，则将c设置为scode[1]，cflags设置为0；否则如果c不等于scode[1]，则返回0
     if (cflags >= REQ_NONE) { c = scode[1]; cflags = 0; }
       else if (c != scode[1]) return 0;
     break;

     case OP_EXACTI:
     // scode指针向后移动IMM2_SIZE个位置
     scode += IMM2_SIZE;
     /* Fall through */

     case OP_CHARI:
     case OP_PLUSI:
     case OP_MINPLUSI:
     case OP_POSPLUSI:
     // 如果不在断言内部，则返回0
     if (inassert == 0) return 0;

     // 如果字符的长度超过一个代码单元，则在进行不区分大小写匹配时无法设置其第一个代码单元
#ifdef SUPPORT_UNICODE
#if PCRE2_CODE_UNIT_WIDTH == 8
     if (scode[1] >= 0x80) return 0;
#elif PCRE2_CODE_UNIT_WIDTH == 16
     if (scode[1] >= 0xd800 && scode[1] <= 0xdfff) return 0;
#endif
#endif

     // 如果cflags大于等于REQ_NONE，则将c设置为scode[1]，cflags设置为REQ_CASELESS；否则如果c不等于scode[1]，则返回0
     if (cflags >= REQ_NONE) { c = scode[1]; cflags = REQ_CASELESS; }
       else if (c != scode[1]) return 0;
     break;
     }

   // code指针向后移动1个位置
   code += GET(code, 1);
   }
// 如果下一个操作码是OP_ALT，则继续循环
while (*code == OP_ALT);

// 将flags指向cflags，返回c
*flags = cflags;
return c;
}
/* Add an entry to the name/number table      *
*************************************************/

/* This function is called between compiling passes to add an entry to the
name/number table, maintaining alphabetical order. Checking for permitted
and forbidden duplicates has already been done.

Arguments:
  cb           the compile data block
  name         the name to add
  length       the length of the name
  groupno      the group number
  tablecount   the count of names in the table so far

Returns:       nothing
*/

static void
add_name_to_table(compile_block *cb, PCRE2_SPTR name, int length,
  unsigned int groupno, uint32_t tablecount)
{
  uint32_t i;
  PCRE2_UCHAR *slot = cb->name_table;

  for (i = 0; i < tablecount; i++)
  {
    int crc = memcmp(name, slot+IMM2_SIZE, CU2BYTES(length));
    if (crc == 0 && slot[IMM2_SIZE+length] != 0)
      crc = -1; /* Current name is a substring */

    /* Make space in the table and break the loop for an earlier name. For a
    duplicate or later name, carry on. We do this for duplicates so that in the
    simple case (when ?(| is not used) they are in order of their numbers. In all
    cases they are in the order in which they appear in the pattern. */

    if (crc < 0)
    {
      (void)memmove(slot + cb->name_entry_size, slot,
        CU2BYTES((tablecount - i) * cb->name_entry_size));
      break;
    }

    /* Continue the loop for a later or duplicate name */

    slot += cb->name_entry_size;
  }

  PUT2(slot, 0, groupno);
  memcpy(slot + IMM2_SIZE, name, CU2BYTES(length));

  /* Add a terminating zero and fill the rest of the slot with zeroes so that
  the memory is all initialized. Otherwise valgrind moans about uninitialized
  memory when saving serialized compiled patterns. */

  memset(slot + IMM2_SIZE + length, 0,
    CU2BYTES(cb->name_entry_size - length - IMM2_SIZE));
}
# 这个函数用于在查找回顾分支的长度时跳过解析模式的部分。它在(*ACCEPT)和(*FAIL)之后被调用以找到分支的结束，它在跳过内部环视或(DEFINE)组时被调用，也被调用以跳过类的结束，在这种情况下它永远不会遇到嵌套的组（但没有必要为此编写特殊代码）。

# 当被调用以找到分支或组的结束时，pptr必须指向分支内部的第一个元代码，而不是分支起始代码。在其他情况下，它可以指向导致调用该函数的项目。

# 参数：
#   pptr       要跳过的当前指针
#   skiptype   PSKIP_CLASS表示跳到类的结束
#              PSKIP_ALT表示META_ALT结束跳过
#              PSKIP_KET表示只有META_KET结束跳过

# 返回值：     pptr的新值
#              如果到达META_END，则返回NULL - 不应该发生
#                或对于未知的元值 - 同样也是如此
static uint32_t *
parsed_skip(uint32_t *pptr, uint32_t skiptype)
{
    uint32_t nestlevel = 0;

    for (;; pptr++)
    {
        uint32_t meta = META_CODE(*pptr);

        switch(meta)
        {
            default:  # 只是跳过大多数项目
                if (meta < META_END) continue  # 字面值
                break;

            # 这不应该发生。
            case META_END:
                return NULL;

            # 这些项目的数据长度是可变的。
            case META_BACKREF:  # 仅当组>=10时才存在偏移量
                if (META_DATA(*pptr) >= 10) pptr += SIZEOFFSET;
                break;

            case META_ESCAPE:   # 一些转义后面跟着数据项。
                switch (META_DATA(*pptr))
                {
                    case ESC_P:
                    case ESC_p:
                        pptr += 1;
                        break;

                    case ESC_g:
                    case ESC_k:
                        pptr += 1 + SIZEOFFSET;
                        break;
                }
                break;

            case META_MARK:     # 添加名称的长度。
            case META_COMMIT_ARG:
            case META_PRUNE_ARG:
            case META_SKIP_ARG:
            case META_THEN_ARG:
                pptr += pptr[1];
    # 终止当前循环
    break;

    # 这些是循环中的“活动”项目。

    # 如果遇到 META_CLASS_END，且 skiptype 为 PSKIP_CLASS，则返回当前指针位置
    case META_CLASS_END:
    if (skiptype == PSKIP_CLASS) return pptr;
    break;

    # 处理 META_ATOMIC, META_CAPTURE, META_COND_ASSERT 等情况，增加嵌套层级
    case META_ATOMIC:
    case META_CAPTURE:
    case META_COND_ASSERT:
    case META_COND_DEFINE:
    case META_COND_NAME:
    case META_COND_NUMBER:
    case META_COND_RNAME:
    case META_COND_RNUMBER:
    case META_COND_VERSION:
    case META_LOOKAHEAD:
    case META_LOOKAHEADNOT:
    case META_LOOKAHEAD_NA:
    case META_LOOKBEHIND:
    case META_LOOKBEHINDNOT:
    case META_LOOKBEHIND_NA:
    case META_NOCAPTURE:
    case META_SCRIPT_RUN:
    nestlevel++;
    break;

    # 处理 META_ALT，如果嵌套层级为0且 skiptype 为 PSKIP_ALT，则返回当前指针位置
    case META_ALT:
    if (nestlevel == 0 && skiptype == PSKIP_ALT) return pptr;
    break;

    # 处理 META_KET，如果嵌套层级为0，则返回当前指针位置，否则减少嵌套层级
    case META_KET:
    if (nestlevel == 0) return pptr;
    nestlevel--;
    break;
    }

  # 每个元数据的额外数据项长度在一个表中
  meta = (meta >> 16) & 0x7fff;
  if (meta >= sizeof(meta_extra_lengths)) return NULL;
  pptr += meta_extra_lengths[meta];
  }
/* Control never reaches here */
return pptr;
}

/*************************************************
*       Find length of a parsed group            *
*************************************************/

/* This is called for nested groups within a branch of a lookbehind whose
length is being computed. If all the branches in the nested group have the same
length, that is OK. On entry, the pointer must be at the first element after
the group initializing code. On exit it points to OP_KET. Caching is used to
improve processing speed when the same capturing group occurs many times.

Arguments:
  pptrptr     pointer to pointer in the parsed pattern
  isinline    FALSE if a reference or recursion; TRUE for inline group
  errcodeptr  pointer to the errorcode
  lcptr       pointer to the loop counter
  group       number of captured group or -1 for a non-capturing group
  recurses    chain of recurse_check to catch mutual recursion
  cb          pointer to the compile data

Returns:      the group length or a negative number
*/

static int
get_grouplength(uint32_t **pptrptr, BOOL isinline, int *errcodeptr, int *lcptr,
   int group, parsed_recurse_check *recurses, compile_block *cb)
{
  # Find the length of a parsed group
  int branchlength;
  int grouplength = -1;

  /* The cache can be used only if there is no possibility of there being two
  groups with the same number. We do not need to set the end pointer for a group
  that is being processed as a back reference or recursion, but we must do so for
  an inline group. */
  if (group > 0 && (cb->external_flags & PCRE2_DUPCAPUSED) == 0)
  {
    uint32_t groupinfo = cb->groupinfo[group];
    if ((groupinfo & GI_NOT_FIXED_LENGTH) != 0) return -1;
    if ((groupinfo & GI_SET_FIXED_LENGTH) != 0)
    {
      if (isinline) *pptrptr = parsed_skip(*pptrptr, PSKIP_KET);
      return groupinfo & GI_FIXED_LENGTH_MASK;
    }
  }

  /* Scan the group. In this case we find the end pointer of necessity. */
  # Scan the group to find the end pointer
for(;;)
  {
  // 获取分支的长度
  branchlength = get_branchlength(pptrptr, errcodeptr, lcptr, recurses, cb);
  // 如果分支长度小于0，跳转到ISNOTFIXED标签
  if (branchlength < 0) goto ISNOTFIXED;
  // 如果组长度为-1，将组长度设置为分支长度
  if (grouplength == -1) grouplength = branchlength;
    else if (grouplength != branchlength) goto ISNOTFIXED;
  // 如果指针指向的值为META_KET，跳出循环
  if (**pptrptr == META_KET) break;
  // 指针向后移动一位，跳过META_ALT
  *pptrptr += 1;   /* Skip META_ALT */
  }

if (group > 0)
  // 将组信息中的固定长度标志位置为1，并将组长度存入
  cb->groupinfo[group] |= (uint32_t)(GI_SET_FIXED_LENGTH | grouplength);
// 返回组长度
return grouplength;

ISNOTFIXED:
if (group > 0) 
  // 将组信息中的非固定长度标志位置为1
  cb->groupinfo[group] |= GI_NOT_FIXED_LENGTH;
// 返回-1
return -1;
}



/*************************************************
*        Find length of a parsed branch          *
*************************************************/

/* Return a fixed length for a branch in a lookbehind, giving an error if the
length is not fixed. On entry, *pptrptr points to the first element inside the
branch. On exit it is set to point to the ALT or KET.

Arguments:
  pptrptr     pointer to pointer in the parsed pattern
  errcodeptr  pointer to error code
  lcptr       pointer to loop counter
  recurses    chain of recurse_check to catch mutual recursion
  cb          pointer to compile block

Returns:      the length, or a negative value on error
*/

static int
get_branchlength(uint32_t **pptrptr, int *errcodeptr, int *lcptr,
  parsed_recurse_check *recurses, compile_block *cb)
{
int branchlength = 0;
int grouplength;
uint32_t lastitemlength = 0;
uint32_t *pptr = *pptrptr;
PCRE2_SIZE offset;
parsed_recurse_check this_recurse;

/* A large and/or complex regex can take too long to process. This can happen
more often when (?| groups are present in the pattern because their length
cannot be cached. */

if ((*lcptr)++ > 2000)
  {
  *errcodeptr = ERR35;  /* Lookbehind is too complicated */
  return -1;
  }

/* Scan the branch, accumulating the length. */

for (;; pptr++)
  {
  parsed_recurse_check *r;
  uint32_t *gptr, *gptrend;
  uint32_t escape;
  uint32_t group = 0;
  uint32_t itemlength = 0;

  if (*pptr < META_END)
    {
    itemlength = 1;
    }

  else switch (META_CODE(*pptr))
    {
    // 如果遇到 META_KET 或 META_ALT，则跳转到 EXIT 标签处
    case META_KET:
    case META_ALT:
    goto EXIT;

    /* (*ACCEPT) 和 (*FAIL) 终止分支，但我们必须跳过到实际的终止位置。 */

    // 如果遇到 META_ACCEPT 或 META_FAIL，则调用 parsed_skip 函数跳过当前分支，然后跳转到 EXIT 标签处
    case META_ACCEPT:
    case META_FAIL:
    pptr = parsed_skip(pptr, PSKIP_ALT);
    if (pptr == NULL) goto PARSED_SKIP_FAILED;
    goto EXIT;

    // 如果遇到 META_MARK、META_COMMIT_ARG、META_PRUNE_ARG、META_SKIP_ARG、META_THEN_ARG，则将指针向前移动相应的长度
    case META_MARK:
    case META_COMMIT_ARG:
    case META_PRUNE_ARG:
    case META_SKIP_ARG:
    case META_THEN_ARG:
    pptr += pptr[1] + 1;
    break;

    // 如果遇到 META_CIRCUMFLEX、META_COMMIT、META_DOLLAR、META_PRUNE、META_SKIP、META_THEN，则不做任何操作
    case META_CIRCUMFLEX:
    case META_COMMIT:
    case META_DOLLAR:
    case META_PRUNE:
    case META_SKIP:
    case META_THEN:
    break;

    // 如果遇到 META_OPTIONS，则将指针向前移动 1 个位置
    case META_OPTIONS:
    pptr += 1;
    break;

    // 如果遇到 META_BIGVALUE，则将 itemlength 设置为 1，然后将指针向前移动 1 个位置
    case META_BIGVALUE:
    itemlength = 1;
    pptr += 1;
    break;

    // 如果遇到 META_CLASS 或 META_CLASS_NOT，则将 itemlength 设置为 1，然后调用 parsed_skip 函数跳过相应的内容
    case META_CLASS:
    case META_CLASS_NOT:
    itemlength = 1;
    pptr = parsed_skip(pptr, PSKIP_CLASS);
    if (pptr == NULL) goto PARSED_SKIP_FAILED;
    break;

    // 如果遇到 META_CLASS_EMPTY_NOT 或 META_DOT，则将 itemlength 设置为 1
    case META_CLASS_EMPTY_NOT:
    case META_DOT:
    itemlength = 1;
    break;

    // 如果遇到 META_CALLOUT_NUMBER，则将指针向前移动 3 个位置
    case META_CALLOUT_NUMBER:
    pptr += 3;
    break;

    // 如果遇到 META_CALLOUT_STRING，则将指针向前移动 3 + SIZEOFFSET 个位置
    case META_CALLOUT_STRING:
    pptr += 3 + SIZEOFFSET;
    break;

    /* 只有一些转义符会消耗一个字符。其中，\R 和 \X 从不被允许，因为它们可能匹配多个字符。在 32 位和非 UTF 8/16 位模式下，\C 只允许。 */

    // 如果遇到 META_ESCAPE，则获取转义符的类型，如果是 \R 或 \X 则返回 -1，如果是 \C 且不在 32 位和非 UTF 8/16 位模式下，则返回 -1
    case META_ESCAPE:
    escape = META_DATA(*pptr);
    if (escape == ESC_R || escape == ESC_X) return -1;
    if (escape > ESC_b && escape < ESC_Z)
      {
#if PCRE2_CODE_UNIT_WIDTH != 32
      # 如果 PCRE2_CODE_UNIT_WIDTH 不等于 32
      if ((cb->external_options & PCRE2_UTF) != 0 && escape == ESC_C)
        {
        # 如果外部选项中包含 PCRE2_UTF 并且转义字符为 ESC_C，则设置错误码并返回-1
        *errcodeptr = ERR36;
        return -1;
        }
#endif
      # 设置 itemlength 为 1
      itemlength = 1;
      # 如果转义字符为 ESC_p 或 ESC_P，则跳过属性数据
      if (escape == ESC_p || escape == ESC_P) pptr++;  /* Skip prop data */
      }
    break;

    /* Lookaheads do not contribute to the length of this branch, but they may
    contain lookbehinds within them whose lengths need to be set. */

    case META_LOOKAHEAD:
    case META_LOOKAHEADNOT:
    case META_LOOKAHEAD_NA:
    # 检查后瞻断言中的后顾断言，并设置其长度
    *errcodeptr = check_lookbehinds(pptr + 1, &pptr, recurses, cb, lcptr);
    if (*errcodeptr != 0) return -1;

    /* Ignore any qualifiers that follow a lookahead assertion. */

    switch (pptr[1])
      {
      # 忽略后瞻断言后面的任何限定符
      case META_ASTERISK:
      case META_ASTERISK_PLUS:
      case META_ASTERISK_QUERY:
      case META_PLUS:
      case META_PLUS_PLUS:
      case META_PLUS_QUERY:
      case META_QUERY:
      case META_QUERY_PLUS:
      case META_QUERY_QUERY:
      pptr++;
      break;

      case META_MINMAX:
      case META_MINMAX_PLUS:
      case META_MINMAX_QUERY:
      pptr += 3;
      break;

      default:
      break;
      }
    break;

    /* A nested lookbehind does not contribute any length to this lookbehind,
    but must itself be checked and have its lengths set. */

    case META_LOOKBEHIND:
    case META_LOOKBEHINDNOT:
    case META_LOOKBEHIND_NA:
    # 设置后顾断言的长度
    if (!set_lookbehind_lengths(&pptr, errcodeptr, lcptr, recurses, cb))
      return -1;
    break;

    /* Back references and recursions are handled by very similar code. At this
    stage, the names generated in the parsing pass are available, but the main
    name table has not yet been created. So for the named varieties, scan the
    list of names in order to get the number of the first one in the pattern,
    and whether or not this name is duplicated. */

    case META_BACKREF_BYNAME:
    # 如果外部选项中包含 PCRE2_MATCH_UNSET_BACKREF，则跳转到 ISNOTFIXED 标签
    if ((cb->external_options & PCRE2_MATCH_UNSET_BACKREF) != 0)
      goto ISNOTFIXED;
    /* Fall through */
    // 继续执行下一个 case 分支

    case META_RECURSE_BYNAME:
      // 处理按名称递归的情况
      {
      int i;
      PCRE2_SPTR name;
      BOOL is_dupname = FALSE;
      named_group *ng = cb->named_groups;
      uint32_t meta_code = META_CODE(*pptr);
      uint32_t length = *(++pptr);
      // 获取偏移量
      GETPLUSOFFSET(offset, pptr);
      name = cb->start_pattern + offset;
      // 遍历已找到的命名组
      for (i = 0; i < cb->names_found; i++, ng++)
        {
        // 检查是否存在重复的命名组
        if (length == ng->length && PRIV(strncmp)(name, ng->name, length) == 0)
          {
          group = ng->number;
          is_dupname = ng->isdup;
          break;
          }
        }

      if (group == 0)
        {
        *errcodeptr = ERR15;  /* Non-existent subpattern */
        cb->erroroffset = offset;
        return -1;
        }
      // 处理按名称递归或非重复命名反向引用
      if (meta_code == META_RECURSE_BYNAME ||
          (!is_dupname && (cb->external_flags & PCRE2_DUPCAPUSED) == 0))
        goto RECURSE_OR_BACKREF_LENGTH;  /* Handle as a numbered version. */
      }
    goto ISNOTFIXED;                     /* Duplicate name or number */

    /* The offset values for back references < 10 are in a separate vector
    because otherwise they would use more than two parsed pattern elements on
    64-bit systems. */
    // 处理反向引用的情况
    case META_BACKREF:
    if ((cb->external_options & PCRE2_MATCH_UNSET_BACKREF) != 0 ||
        (cb->external_flags & PCRE2_DUPCAPUSED) != 0)
      goto ISNOTFIXED;
    group = META_DATA(*pptr);
    if (group < 10)
      {
      offset = cb->small_ref_offset[group];
      goto RECURSE_OR_BACKREF_LENGTH;
      }

    /* Fall through */
    // 继续执行下一个 case 分支
    /* For groups >= 10 - picking up group twice does no harm. */

    /* A true recursion implies not fixed length, but a subroutine call may
    be OK. Back reference "recursions" are also failed. */
    // 处理递归的情况
    case META_RECURSE:
    group = META_DATA(*pptr);
    # 获取偏移量和指针
    GETPLUSOFFSET(offset, pptr);

    # 递归或回溯长度
    RECURSE_OR_BACKREF_LENGTH:
    # 如果组大于括号计数，则设置错误偏移量并返回错误代码
    if (group > cb->bracount)
      {
      cb->erroroffset = offset;
      *errcodeptr = ERR15;  /* Non-existent subpattern */
      return -1;
      }
    # 如果组为0，则跳转到ISNOTFIXED，进行本地递归
    if (group == 0) goto ISNOTFIXED;  /* Local recursion */
    # 遍历解析后的模式，寻找对应的组
    for (gptr = cb->parsed_pattern; *gptr != META_END; gptr++)
      {
      # 如果遇到META_BIGVALUE，则跳过一个字符
      if (META_CODE(*gptr) == META_BIGVALUE) gptr++;
        # 如果找到对应的组，则跳出循环
        else if (*gptr == (META_CAPTURE | group)) break;
      }

    # 我们必须从组内的第一个元字符开始搜索组的结束位置。否则，它将被视为一个封闭的组。
    gptrend = parsed_skip(gptr + 1, PSKIP_KET);
    if (gptrend == NULL) goto PARSED_SKIP_FAILED;
    # 如果指针在组开始和结束之间，则跳转到ISNOTFIXED，进行本地递归
    if (pptr > gptr && pptr < gptrend) goto ISNOTFIXED;  /* Local recursion */
    # 遍历递归列表，检查是否存在相互递归
    for (r = recurses; r != NULL; r = r->prev) if (r->groupptr == gptr) break;
    if (r != NULL) goto ISNOTFIXED;   /* Mutual recursion */
    # 设置当前递归的前一个递归和组指针
    this_recurse.prev = recurses;
    this_recurse.groupptr = gptr;

    # 我们不需要知道组的结束位置，即在调用get_grouplength()后不再使用gptr。设置第二个参数为FALSE，当长度可以在缓存中找到时，停止扫描结束位置。
    gptr++;
    grouplength = get_grouplength(&gptr, FALSE, errcodeptr, lcptr, group, &this_recurse, cb);
    # 如果组长度小于0，则返回错误或跳转到ISNOTFIXED
    if (grouplength < 0)
      {
      if (*errcodeptr == 0) goto ISNOTFIXED;
      return -1;  /* Error already set */
      }
    itemlength = grouplength;
    break;

    # (DEFINE)组永远不会内联执行，因此不会影响此分支的长度。跳过以下项目到下一个未配对的右括号。
    case META_COND_DEFINE:
    pptr = parsed_skip(pptr + 1, PSKIP_KET);
    break;

    # 检查其他嵌套组 - 针对每种类型，前进到初始数据，然后使用get_grouplength()寻找固定长度。
    case META_COND_NAME:
    case META_COND_NUMBER:
    # 如果条件为 META_COND_RNAME 或 META_COND_RNUMBER，则指针向前移动2 + SIZEOFFSET个位置，然后跳转到CHECK_GROUP
    case META_COND_RNAME:
    case META_COND_RNUMBER:
    pptr += 2 + SIZEOFFSET;
    goto CHECK_GROUP;

    # 如果条件为 META_COND_ASSERT，则指针向前移动1个位置，然后跳转到CHECK_GROUP
    case META_COND_ASSERT:
    pptr += 1;
    goto CHECK_GROUP;

    # 如果条件为 META_COND_VERSION，则指针向前移动4个位置，然后跳转到CHECK_GROUP
    case META_COND_VERSION:
    pptr += 4;
    goto CHECK_GROUP;

    # 如果条件为 META_CAPTURE，则将group设置为META_DATA(*pptr)，然后继续执行下面的操作
    case META_CAPTURE:
    group = META_DATA(*pptr);
    /* Fall through */

    # 如果条件为 META_ATOMIC、META_NOCAPTURE 或 META_SCRIPT_RUN，则指针向前移动1个位置，然后跳转到CHECK_GROUP
    case META_ATOMIC:
    case META_NOCAPTURE:
    case META_SCRIPT_RUN:
    pptr++;
    CHECK_GROUP:
    # 调用get_grouplength函数计算组的长度，并进行相应的处理
    grouplength = get_grouplength(&pptr, TRUE, errcodeptr, lcptr, group, recurses, cb);
    if (grouplength < 0) return -1;
    itemlength = grouplength;
    break;

    # 如果条件为 META_MINMAX、META_MINMAX_PLUS 或 META_MINMAX_QUERY，则根据不同情况进行处理
    case META_MINMAX:
    case META_MINMAX_PLUS:
    case META_MINMAX_QUERY:
    if (pptr[1] == pptr[2])
      {
      switch(pptr[1])
        {
        # 如果重复次数为0，则需要减去已经添加的长度
        case 0:
        branchlength -= lastitemlength;
        break;

        # 如果重复次数为1，则将itemlength设置为0
        case 1:
        itemlength = 0;
        break;

        # 如果重复次数大于1，则需要计算itemlength
        default:  /* 检查整数溢出 */
        if (lastitemlength != 0 && INT_MAX/lastitemlength < pptr[1] - 1)
          {
          *errcodeptr = ERR87;  /* 整数溢出；回溯太大 */
          return -1;
          }
        itemlength = (pptr[1] - 1) * lastitemlength;
        break;
        }
      pptr += 2;
      break;
      }
    /* Fall through */

    # 如果条件不满足上述情况，则跳转到ISNOTFIXED
    default:
    ISNOTFIXED:
    *errcodeptr = ERR25;   /* 非固定长度 */
    return -1;
    }

  # 将itemlength添加到branchlength中，检查整数溢出和分支长度是否超过限制
  if (INT_MAX - branchlength < (int)itemlength || (branchlength += itemlength) > LOOKBEHIND_MAX)
    {
    *errcodeptr = ERR87;
    return -1;
    }
    /* 保存这个项目的长度，以便在下一个项目是定量词时使用 */
    lastitemlength = itemlength;
  }
/* 设置回顾环视中各分支的长度 */

/* 对每个回顾环视调用此函数，以设置其各分支的长度。如果任何分支的长度不是固定的且小于最大长度（65535），则会出现错误。在退出时，指针必须停在最终的 ket 上。

该函数还维护了 max_lookbehind 值。任何包含嵌套回顾环视的回顾环视分支可能实际上比分支的长度要更长。额外的长度从 get_branchlength() 作为 "extra" 值传回。

参数：
  pptrptr     指向解析模式中指针的指针
  errcodeptr  指向错误代码的指针
  lcptr       指向循环计数器的指针
  recurses    用于捕获相互递归的 recurse_check 链
  cb          指向编译块的指针

返回值：      如果一切正常则返回 TRUE
              否则返回 FALSE，同时设置错误代码和偏移量
*/

static BOOL
set_lookbehind_lengths(uint32_t **pptrptr, int *errcodeptr, int *lcptr,
  parsed_recurse_check *recurses, compile_block *cb)
{
PCRE2_SIZE offset;
int branchlength;
uint32_t *bptr = *pptrptr;

READPLUSOFFSET(offset, bptr);  /* 用于错误消息的偏移量 */
*pptrptr += SIZEOFFSET;

do
  {
  *pptrptr += 1;
  branchlength = get_branchlength(pptrptr, errcodeptr, lcptr, recurses, cb);
  if (branchlength < 0)
    {
    /* 错误代码和偏移量可能已经从嵌套回顾环视中设置。 */
    if (*errcodeptr == 0) *errcodeptr = ERR25;
    if (cb->erroroffset == PCRE2_UNSET) cb->erroroffset = offset;
    return FALSE;
    }
  if (branchlength > cb->max_lookbehind) cb->max_lookbehind = branchlength;
  *bptr |= branchlength;  /* 分支长度永远不会超过 65535 */
  bptr = *pptrptr;
  }
while (*bptr == META_ALT);

return TRUE;
}
/* 
*         Check parsed pattern lookbehinds       *
*************************************************/

/* This function is called at the end of parsing a pattern if any lookbehinds
were encountered. It scans the parsed pattern for them, calling
set_lookbehind_lengths() for each one. At the start, the errorcode is zero and
the error offset is marked unset. The enables the functions above not to
override settings from deeper nestings.

This function is called recursively from get_branchlength() for lookaheads in
order to process any lookbehinds that they may contain. It stops when it hits a
non-nested closing parenthesis in this case, returning a pointer to it.

Arguments
  pptr      points to where to start (start of pattern or start of lookahead)
  retptr    if not NULL, return the ket pointer here
  recurses  chain of recurse_check to catch mutual recursion
  cb        points to the compile block
  lcptr     points to loop counter

Returns:    0 on success, or an errorcode (cb->erroroffset will be set)
*/

static int
check_lookbehinds(uint32_t *pptr, uint32_t **retptr,
  parsed_recurse_check *recurses, compile_block *cb, int *lcptr)
{
// 初始化错误码为0，嵌套层级为0
int errorcode = 0;
int nestlevel = 0;

// 设置错误偏移量为未设置状态
cb->erroroffset = PCRE2_UNSET;

// 遍历解析后的模式
for (; *pptr != META_END; pptr++)
  {
  // 如果是字面量，继续下一次循环
  if (*pptr < META_END) continue;  /* Literal */

  // 根据不同的元字符进行处理
  switch (META_CODE(*pptr))
    {
    default:
    return ERR70;  /* Unrecognized meta code */

    case META_ESCAPE:
    // 如果是转义字符，跳过下一个字符
    if (*pptr - META_ESCAPE == ESC_P || *pptr - META_ESCAPE == ESC_p)
      pptr += 1;
    break;

    case META_KET:
    // 如果是右括号，嵌套层级减一
    if (--nestlevel < 0)
      {
      // 如果返回指针不为空，将当前指针赋值给返回指针
      if (retptr != NULL) *retptr = pptr;
      return 0;
      }
    break;

    case META_ATOMIC:
    case META_CAPTURE:
    case META_COND_ASSERT:
    case META_LOOKAHEAD:
    case META_LOOKAHEADNOT:
    case META_LOOKAHEAD_NA:
    case META_NOCAPTURE:
    case META_SCRIPT_RUN:
    // 如果是指定的元字符，嵌套层级加一
    nestlevel++;
    break;

    case META_ACCEPT:
    case META_ALT:
    case META_ASTERISK:
    case META_ASTERISK_PLUS:
    // 其他情况...
    }
  }
}
    # 处理各种元字符的情况
    case META_ASTERISK_QUERY:
    case META_BACKREF:
    case META_CIRCUMFLEX:
    case META_CLASS:
    case META_CLASS_EMPTY:
    case META_CLASS_EMPTY_NOT:
    case META_CLASS_END:
    case META_CLASS_NOT:
    case META_COMMIT:
    case META_DOLLAR:
    case META_DOT:
    case META_FAIL:
    case META_PLUS:
    case META_PLUS_PLUS:
    case META_PLUS_QUERY:
    case META_PRUNE:
    case META_QUERY:
    case META_QUERY_PLUS:
    case META_QUERY_QUERY:
    case META_RANGE_ESCAPED:
    case META_RANGE_LITERAL:
    case META_SKIP:
    case META_THEN:
    # 结束当前情况的处理
    break;

    # 处理递归的情况
    case META_RECURSE:
    pptr += SIZEOFFSET;
    # 结束当前情况的处理
    break;

    # 处理按名称引用反向引用的情况
    case META_BACKREF_BYNAME:
    case META_RECURSE_BYNAME:
    pptr += 1 + SIZEOFFSET;
    # 结束当前情况的处理
    break;

    # 处理条件定义的情况
    case META_COND_DEFINE:
    pptr += SIZEOFFSET;
    nestlevel++;
    # 结束当前情况的处理
    break;

    # 处理条件名称、数字、反向名称、反向数字的情况
    case META_COND_NAME:
    case META_COND_NUMBER:
    case META_COND_RNAME:
    case META_COND_RNUMBER:
    pptr += 1 + SIZEOFFSET;
    nestlevel++;
    # 结束当前情况的处理
    break;

    # 处理条件版本的情况
    case META_COND_VERSION:
    pptr += 3;
    nestlevel++;
    # 结束当前情况的处理
    break;

    # 处理调用字符串的情况
    case META_CALLOUT_STRING:
    pptr += 3 + SIZEOFFSET;
    # 结束当前情况的处理
    break;

    # 处理大值、选项、POSIX、负向POSIX的情况
    case META_BIGVALUE:
    case META_OPTIONS:
    case META_POSIX:
    case META_POSIX_NEG:
    pptr += 1;
    # 结束当前情况的处理
    break;

    # 处理最小最大值、最小最大值查询、最小最大值加号的情况
    case META_MINMAX:
    case META_MINMAX_QUERY:
    case META_MINMAX_PLUS:
    pptr += 2;
    # 结束当前情况的处理
    break;

    # 处理调用数字的情况
    case META_CALLOUT_NUMBER:
    pptr += 3;
    # 结束当前情况的处理
    break;

    # 处理标记、提交参数、剪枝参数、跳过参数、然后参数的情况
    case META_MARK:
    case META_COMMIT_ARG:
    case META_PRUNE_ARG:
    case META_SKIP_ARG:
    case META_THEN_ARG:
    pptr += 1 + pptr[1];
    # 结束当前情况的处理
    break;

    # 处理回顾后断言、否定回顾后断言、不适用回顾后断言的情况
    case META_LOOKBEHIND:
    case META_LOOKBEHINDNOT:
    case META_LOOKBEHIND_NA:
    # 如果无法设置回顾后断言的长度，则返回错误码
    if (!set_lookbehind_lengths(&pptr, &errorcode, lcptr, recurses, cb))
      return errorcode;
    # 结束当前情况的处理
    break;
    }
  }
# 返回值为0
return 0;
}



/*************************************************
*     External function to compile a pattern     *
*************************************************/

/* This function reads a regular expression in the form of a string and returns
a pointer to a block of store holding a compiled version of the expression.

Arguments:
  pattern       the regular expression
  patlen        the length of the pattern, or PCRE2_ZERO_TERMINATED
  options       option bits
  errorptr      pointer to errorcode
  erroroffset   pointer to error offset
  ccontext      points to a compile context or is NULL

Returns:        pointer to compiled data block, or NULL on error,
                with errorcode and erroroffset set
*/

# 定义 pcre2_compile 函数，接受正则表达式字符串、长度、选项、错误指针、错误偏移和编译上下文作为参数
PCRE2_EXP_DEFN pcre2_code * PCRE2_CALL_CONVENTION
pcre2_compile(PCRE2_SPTR pattern, PCRE2_SIZE patlen, uint32_t options,
   int *errorptr, PCRE2_SIZE *erroroffset, pcre2_compile_context *ccontext)
{
# 设置 TRUE 表示启用 UTF 模式
BOOL utf;
# 设置 TRUE 表示启用 UCP 模式
BOOL ucp;
# 设置为 TRUE 如果发现了 lookbehind
BOOL has_lookbehind = FALSE;
# 设置为 TRUE 表示模式以零结尾
BOOL zero_terminated;
# 将返回的编译版本的指针
pcre2_real_code *re = NULL;
# "静态"编译时数据
compile_block cb;
# 字符表的基本指针
const uint8_t *tables;

# 当前编译代码的指针
PCRE2_UCHAR *code;
# 编译代码的起始位置
PCRE2_SPTR codestart;
# 当前模式的指针
PCRE2_SPTR ptr;
# 解析模式的当前指针
uint32_t *pptr;

# 长度为1，用于最终的 END 操作码
PCRE2_SIZE length = 1;
# 实际使用的长度
PCRE2_SIZE usedlength;
# 内存块的大小
PCRE2_SIZE re_blocksize;
# 大于等于 0x80000000 的 32 位文字数
PCRE2_SIZE big32count = 0;
# 定义需要用于解析模式的大小
PCRE2_SIZE parsed_size_needed;        /* Needed for parsed pattern */

# 定义用于存储第一个和必需的代码单元类型的标志
uint32_t firstcuflags, reqcuflags;    /* Type of first/req code unit */

# 定义存储第一个和必需的代码单元值的变量
uint32_t firstcu, reqcu;              /* Value of first/req code unit */

# 设置标志变量，用于指示 NL 和 BSR 的设置
uint32_t setflags = 0;                /* NL and BSR set flags */

# 定义用于检查 (*UTF) 等的跳过起始位置的变量
uint32_t skipatstart;                 /* When checking (*UTF) etc */

# 设置堆和匹配限制的初始值
uint32_t limit_heap  = UINT32_MAX;
uint32_t limit_match = UINT32_MAX;    /* Unset match limits */
uint32_t limit_depth = UINT32_MAX;

# 初始化换行符和 BSR 变量
int newline = 0;                      /* Unset; can be set by the pattern */
int bsr = 0;                          /* Unset; can be set by the pattern */

# 初始化错误码和编译返回值
int errorcode = 0;                    /* Initialize to avoid compiler warn */
int regexrc;                          /* Return from compile */

# 定义本地循环计数器
uint32_t i;                           /* Local loop counter */

# 定义用于存储组信息和解析模式的堆栈
uint32_t stack_groupinfo[GROUPINFO_DEFAULT_SIZE];
uint32_t stack_parsed_pattern[PARSED_PATTERN_DEFAULT_SIZE];
named_group named_groups[NAMED_GROUP_LIST_SIZE];

# 工作空间在不同的编译阶段中以不同的方式使用，需要16位对齐
# 定义16位对齐的工作空间
uint32_t c16workspace[C16_WORK_SIZE];
PCRE2_UCHAR *cworkspace = (PCRE2_UCHAR *)c16workspace;


/* -------------- Check arguments and set up the pattern ----------------- */

# 检查错误码和偏移指针是否存在
# 如果不存在则返回空指针
if (errorptr == NULL || erroroffset == NULL) return NULL;
*errorptr = ERR0;
*erroroffset = 0;

# 检查模式是否存在
# 如果不存在则设置错误码并返回空指针
if (pattern == NULL)
  {
  *errorptr = ERR16;
  return NULL;
  }

# 如果编译上下文为空，则使用默认上下文
# 如果 PCRE2_MATCH_INVALID_UTF 包含在选项中，则设置 PCRE2_UTF
if (ccontext == NULL)
  ccontext = (pcre2_compile_context *)(&PRIV(default_compile_context));

if ((options & PCRE2_MATCH_INVALID_UTF) != 0) options |= PCRE2_UTF;

# 检查所有未定义的公共选项位是否为零
# 检查编译选项和额外选项是否超出公共编译选项的范围，如果是则返回错误
if ((options & ~PUBLIC_COMPILE_OPTIONS) != 0 ||
    (ccontext->extra_options & ~PUBLIC_COMPILE_EXTRA_OPTIONS) != 0)
  {
  *errorptr = ERR17;
  return NULL;
  }

# 如果选项中包含PCRE2_LITERAL，检查是否超出公共字面编译选项的范围，如果是则返回错误
if ((options & PCRE2_LITERAL) != 0 &&
    ((options & ~PUBLIC_LITERAL_COMPILE_OPTIONS) != 0 ||
     (ccontext->extra_options & ~PUBLIC_LITERAL_COMPILE_EXTRA_OPTIONS) != 0))
  {
  *errorptr = ERR92;
  return NULL;
  }

# 检查是否为零结尾的模式，如果是则将模式长度设置为字符串长度
if ((zero_terminated = (patlen == PCRE2_ZERO_TERMINATED)))
  patlen = PRIV(strlen)(pattern);

# 检查模式长度是否超出最大模式长度，如果是则返回错误
if (patlen > ccontext->max_pattern_length)
  {
  *errorptr = ERR88;
  return NULL;
  }

# 从这里开始，函数中所有的返回都应该通过EXIT标签

# 初始化静态编译数据
tables = (ccontext->tables != NULL)? ccontext->tables : PRIV(default_tables);
cb.lcc = tables + lcc_offset;          # Individual
cb.fcc = tables + fcc_offset;          #   character
cb.cbits = tables + cbits_offset;      #      tables
cb.ctypes = tables + ctypes_offset;

cb.assert_depth = 0;
cb.bracount = 0;
cb.cx = ccontext;
cb.dupnames = FALSE;
cb.end_pattern = pattern + patlen;
cb.erroroffset = 0;
cb.external_flags = 0;
cb.external_options = options;
cb.groupinfo = stack_groupinfo;
cb.had_recurse = FALSE;
cb.lastcapture = 0;
cb.max_lookbehind = 0;
cb.name_entry_size = 0;
cb.name_table = NULL;
cb.named_groups = named_groups;
cb.named_group_list_size = NAMED_GROUP_LIST_SIZE;
cb.names_found = 0;
cb.open_caps = NULL;
cb.parens_depth = 0;
cb.parsed_pattern = stack_parsed_pattern;
cb.req_varyopt = 0;
cb.start_code = cworkspace;
cb.start_pattern = pattern;
cb.start_workspace = cworkspace;
cb.workspace_size = COMPILE_WORK_SIZE;

# 最大反向引用和反向引用位图。位图记录了最多31个反向引用，以帮助决定(.*)是否可以被视为锚定的
cb.top_backref = 0;
# 设置 cb.backref_map 为 0
cb.backref_map = 0;

# 对于转义序列 \1 到 \9，它们始终是反向引用，但由于它们只有两个字符长，因此在 parsed_pattern 向量中只能使用两个元素。
# 第一个包含引用，我们希望使用第二个来记录模式中的偏移量，以便稍后可以使用偏移量诊断对不存在的组的前向引用。
# 但是，在 64 位系统上，PCRE2_SIZE 不适合。相反，我们有一个偏移量向量，用于 \1 到 \9 的第一次出现，由第二个 parsed_pattern 值索引。
# 所有其他引用都有足够的空间将偏移量放入解析模式中。
for (i = 0; i < 10; i++) cb.small_ref_offset[i] = PCRE2_UNSET;

# 开始检查模式
# 除非设置了 PCRE2_LITERAL，否则在模式开头检查全局一次性选项设置，并记住实际正则表达式的偏移量。
# 对于 valgrind 支持，使零终止模式的终止符不可访问。这可以捕获否则只会出现在非零终止模式中的错误。
#ifdef SUPPORT_VALGRIND
if (zero_terminated) VALGRIND_MAKE_MEM_NOACCESS(pattern + patlen, CU2BYTES(1));
#endif

# 指针指向模式
ptr = pattern;
skipatstart = 0;

# 如果未设置 PCRE2_LITERAL
if ((options & PCRE2_LITERAL) == 0)
  {
  # 在模式开头检查全局一次性选项设置，并记住实际正则表达式的偏移量
  while (patlen - skipatstart >= 2 &&
         ptr[skipatstart] == CHAR_LEFT_PARENTHESIS &&
         ptr[skipatstart+1] == CHAR_ASTERISK)
    {
    for (i = 0; i < sizeof(pso_list)/sizeof(pso); i++)
      {
      // 定义变量c和pp，以及指向pso_list中第i个元素的指针p
      uint32_t c, pp;
      pso *p = pso_list + i;

      // 检查是否匹配到指定的模式
      if (patlen - skipatstart - 2 >= p->length &&
          PRIV(strncmp_c8)(ptr + skipatstart + 2, (char *)(p->name),
            p->length) == 0)
        {
        // 更新skipatstart的值
        skipatstart += p->length + 2;
        // 根据p->type的值进行不同的操作
        switch(p->type)
          {
          // 如果是PSO_OPT类型，则将p->value的值添加到cb.external_options
          case PSO_OPT:
          cb.external_options |= p->value;
          break;

          // 如果是PSO_FLG类型，则将p->value的值添加到setflags
          case PSO_FLG:
          setflags |= p->value;
          break;

          // 如果是PSO_NL类型，则将p->value的值赋给newline，并将PCRE2_NL_SET添加到setflags
          case PSO_NL:
          newline = p->value;
          setflags |= PCRE2_NL_SET;
          break;

          // 如果是PSO_BSR类型，则将p->value的值赋给bsr，并将PCRE2_BSR_SET添加到setflags
          case PSO_BSR:
          bsr = p->value;
          setflags |= PCRE2_BSR_SET;
          break;

          // 如果是PSO_LIMM、PSO_LIMD或PSO_LIMH类型，则进行相应的限制设置操作
          case PSO_LIMM:
          case PSO_LIMD:
          case PSO_LIMH:
          c = 0;
          pp = skipatstart;
          // 检查ptr[pp]是否为数字
          if (!IS_DIGIT(ptr[pp]))
            {
            errorcode = ERR60;
            ptr += pp;
            goto HAD_EARLY_ERROR;
            }
          // 将连续的数字字符转换为数字
          while (IS_DIGIT(ptr[pp]))
            {
            if (c > UINT32_MAX / 10 - 1) break;   /* Integer overflow */
            c = c*10 + (ptr[pp++] - CHAR_0);
            }
          // 检查是否为右括号
          if (ptr[pp++] != CHAR_RIGHT_PARENTHESIS)
            {
            errorcode = ERR60;
            ptr += pp;
            goto HAD_EARLY_ERROR;
            }
          // 根据p->type的值将c赋给相应的限制变量，并更新skipatstart的值
          if (p->type == PSO_LIMH) limit_heap = c;
            else if (p->type == PSO_LIMM) limit_match = c;
            else limit_depth = c;
          skipatstart += pp - skipatstart;
          break;
          }
        break;   /* Out of the table scan loop */
        }
      }
    // 如果i大于等于pso_list的大小除以pso的大小，则跳出循环
    if (i >= sizeof(pso_list)/sizeof(pso)) break;   /* Out of pso loop */
    }
  }
/* End of pattern-start options; advance to start of real regex. */
ptr += skipatstart;

/* Can't support UTF or UCP if PCRE2 was built without Unicode support. */
#ifndef SUPPORT_UNICODE
if ((cb.external_options & (PCRE2_UTF|PCRE2_UCP)) != 0)
  {
  errorcode = ERR32;
  goto HAD_EARLY_ERROR;
  }
#endif

/* Check UTF. We have the original options in 'options', with that value as
modified by (*UTF) etc in cb->external_options. The extra option
PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES is not permitted in UTF-16 mode because the
surrogate code points cannot be represented in UTF-16. */
utf = (cb.external_options & PCRE2_UTF) != 0;
if (utf)
  {
  if ((options & PCRE2_NEVER_UTF) != 0)
    {
    errorcode = ERR74;
    goto HAD_EARLY_ERROR;
    }
  if ((options & PCRE2_NO_UTF_CHECK) == 0 &&
       (errorcode = PRIV(valid_utf)(pattern, patlen, erroroffset)) != 0)
    goto HAD_ERROR;  /* Offset was set by valid_utf() */
#if PCRE2_CODE_UNIT_WIDTH == 16
  if ((ccontext->extra_options & PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES) != 0)
    {
    errorcode = ERR91;
    goto HAD_EARLY_ERROR;
    }
#endif
  }

/* Check UCP lockout. */
ucp = (cb.external_options & PCRE2_UCP) != 0;
if (ucp && (cb.external_options & PCRE2_NEVER_UCP) != 0)
  {
  errorcode = ERR75;
  goto HAD_EARLY_ERROR;
  }

/* Process the BSR setting. */
if (bsr == 0) bsr = ccontext->bsr_convention;

/* Process the newline setting. */
if (newline == 0) newline = ccontext->newline_convention;
cb.nltype = NLTYPE_FIXED;
# 根据不同的换行符类型进行处理
switch(newline)
  {
  # 如果是 CR 换行符
  case PCRE2_NEWLINE_CR:
  # 设置换行符长度为 1
  cb.nllen = 1;
  # 设置换行符为 CR
  cb.nl[0] = CHAR_CR;
  # 结束 case
  break;

  # 如果是 LF 换行符
  case PCRE2_NEWLINE_LF:
  # 设置换行符长度为 1
  cb.nllen = 1;
  # 设置换行符为 LF
  cb.nl[0] = CHAR_NL;
  # 结束 case
  break;

  # 如果是 NUL 换行符
  case PCRE2_NEWLINE_NUL:
  # 设置换行符长度为 1
  cb.nllen = 1;
  # 设置换行符为 NUL
  cb.nl[0] = CHAR_NUL;
  # 结束 case
  break;

  # 如果是 CRLF 换行符
  case PCRE2_NEWLINE_CRLF:
  # 设置换行符长度为 2
  cb.nllen = 2;
  # 设置换行符为 CR
  cb.nl[0] = CHAR_CR;
  # 设置换行符为 LF
  cb.nl[1] = CHAR_NL;
  # 结束 case
  break;

  # 如果是任意换行符
  case PCRE2_NEWLINE_ANY:
  # 设置换行符类型为任意
  cb.nltype = NLTYPE_ANY;
  # 结束 case
  break;

  # 如果是任意 CRLF 换行符
  case PCRE2_NEWLINE_ANYCRLF:
  # 设置换行符类型为任意 CRLF
  cb.nltype = NLTYPE_ANYCRLF;
  # 结束 case
  break;

  # 默认情况下
  default:
  # 设置错误代码为 ERR56
  errorcode = ERR56;
  # 跳转到 HAD_EARLY_ERROR 标签处
  goto HAD_EARLY_ERROR;
  # 结束 case
  }

/* 预扫描模式以执行两件事：(1) 发现命名组及其对应的数字，以便这些信息始终可用于剩余处理。 (2) 同时解析模式并将处理后的版本放入 parsed_pattern 向量中。这样就可以解释转义并删除注释（等等）。

除了一个情况外，当未设置 PCRE2_AUTO_CALLOUT 时，解析模式中的无符号 32 位整数的数量受模式长度的限制，再加上一个（用于终止符），如果设置了 PCRE2_EXTRA_WORD 或 PCRE2_EXTRA_LINE，则再加上四个。特殊情况是在 32 位、非 UTF 模式下运行时，大于 META_END（0x80000000）的文字字符必须编码为两个单元。因此，在这种情况下，我们扫描模式以检查这样的值。 */

#if PCRE2_CODE_UNIT_WIDTH == 32
if (!utf)
  {
  PCRE2_SPTR p;
  for (p = ptr; p < cb.end_pattern; p++) if (*p >= META_END) big32count++;
  }
#endif

/* 确保解析模式缓冲区足够大。当设置了 PCRE2_AUTO_CALLOUT 时，我们必须假设每个字符都有一个数字调用（4 个元素），最后再加一个。这有些过度，但是现在内存很充裕。对于许多较小的模式，可以使用堆栈上的向量（在上面设置）。 */

parsed_size_needed = patlen - skipatstart + big32count;
# 如果额外选项中包含匹配单词或匹配行的标志位，则增加解析大小
if ((ccontext->extra_options & (PCRE2_EXTRA_MATCH_WORD|PCRE2_EXTRA_MATCH_LINE)) != 0)
  parsed_size_needed += 4;

# 如果选项中包含自动调用的标志位，则根据公式增加解析大小
if ((options & PCRE2_AUTO_CALLOUT) != 0)
  parsed_size_needed = (parsed_size_needed + 1) * 5;

# 如果解析大小超过默认大小，则分配堆内存来存储解析模式
if (parsed_size_needed >= PARSED_PATTERN_DEFAULT_SIZE)
  {
  uint32_t *heap_parsed_pattern = ccontext->memctl.malloc(
    (parsed_size_needed + 1) * sizeof(uint32_t), ccontext->memctl.memory_data);
  if (heap_parsed_pattern == NULL)
    {
    *errorptr = ERR21;
    goto EXIT;
    }
  cb.parsed_pattern = heap_parsed_pattern;
  }
# 设置解析模式的结束位置
cb.parsed_pattern_end = cb.parsed_pattern + parsed_size_needed + 1;

# 进行解析扫描
errorcode = parse_regex(ptr, cb.external_options, &has_lookbehind, &cb);
if (errorcode != 0) goto HAD_CB_ERROR;

# 需要工作空间来记住关于编号组的信息：组是否可以匹配空字符串以及其固定长度是多少
# 如果组数量较多，则需要分配特殊的向量来存储这些信息
if (cb.bracount >= GROUPINFO_DEFAULT_SIZE)
  {
  cb.groupinfo = ccontext->memctl.malloc(
    (cb.bracount + 1)*sizeof(uint32_t), ccontext->memctl.memory_data);
  if (cb.groupinfo == NULL)
    {
    errorcode = ERR21;
    cb.erroroffset = 0;
    goto HAD_CB_ERROR;
    }
  }
# 将分配的向量初始化为零
memset(cb.groupinfo, 0, (cb.bracount + 1) * sizeof(uint32_t));

# 如果存在回顾环，则扫描解析模式以计算其长度
if (has_lookbehind)
  {
  int loopcount = 0;
  errorcode = check_lookbehinds(cb.parsed_pattern, NULL, NULL, &cb, &loopcount);
  if (errorcode != 0) goto HAD_CB_ERROR;
  }
/* 调试时，可以启用此函数来显示解析后的数据向量 */
#ifdef DEBUG_SHOW_PARSED
fprintf(stderr, "+++ Pre-scan complete:\n");
show_parsed(&cb);
#endif

/* 调试捕获信息时，可以启用此代码 */
#ifdef DEBUG_SHOW_CAPTURES
{
named_group *ng = cb.named_groups;
fprintf(stderr, "+++Captures: %d\n", cb.bracount);
for (i = 0; i < cb.names_found; i++, ng++)
{
fprintf(stderr, "+++%3d %.*s\n", ng->number, ng->length, ng->name);
}
}
#endif

/* 假装编译模式，实际上只是累积 'length' 变量中所需的内存量。这种行为是通过向 compile_regex() 传递一个非空的最后一个参数来触发的。我们传递一个工作空间块 (cworkspace) 给它，以便将模式的部分编译进去；当不再需要编译的代码时，编译的代码将被丢弃，因此希望这个工作空间永远不会溢出，尽管有一个测试来检查它是否会溢出。

在出现错误时，errorcode 将被设置为非零，因此我们不需要查看函数的结果。初始选项已经放入了 cb 块中，但我们仍然必须传递一个单独的选项变量 (第一个参数)，因为选项可能会在处理模式时发生变化。 */
cb.erroroffset = patlen;   /* 对于不设置它的任何后续错误 */
pptr = cb.parsed_pattern;
code = cworkspace;
*code = OP_BRA;

(void)compile_regex(cb.external_options, &code, &pptr, &errorcode, 0, &firstcu,
&firstcuflags, &reqcu, &reqcuflags, NULL, &cb, &length);

if (errorcode != 0) goto HAD_CB_ERROR;  /* 偏移量在 cb.erroroffset 中 */

/* 这应该在 compile_regex() 中被捕获，但以防万一... */
if (length > MAX_PATTERN_SIZE)
{
errorcode = ERR20;
goto HAD_CB_ERROR;
}

/* 计算并获取编译模式和名称表的数据块的大小，然后对其进行初始化。由于现在我们限制了 cb.names_found 和
```cb.names_found``` 的最大值，因此不应再出现整数溢出的情况。*/
# 计算编译后的 pcre2_real_code 结构体的大小
re_blocksize = sizeof(pcre2_real_code) + CU2BYTES(length + (PCRE2_SIZE)cb.names_found * (PCRE2_SIZE)cb.name_entry_size);
# 为编译后的正则表达式分配内存空间
re = (pcre2_real_code *) ccontext->memctl.malloc(re_blocksize, ccontext->memctl.memory_data);
# 如果内存分配失败，则返回错误码 ERR21
if (re == NULL)
  {
  errorcode = ERR21;
  goto HAD_CB_ERROR;
  }

# 在设置字段之前，显式地向 pcre2_real_code 结构体的最后 8 个字节写入零值，以避免复制时读取未定义字节
memset((char *)re + sizeof(pcre2_real_code) - 8, 0, 8);
# 设置 pcre2_real_code 结构体的各个字段
re->memctl = ccontext->memctl;
re->tables = tables;
re->executable_jit = NULL;
memset(re->start_bitmap, 0, 32 * sizeof(uint8_t));
re->blocksize = re_blocksize;
re->magic_number = MAGIC_NUMBER;
re->compile_options = options;
re->overall_options = cb.external_options;
re->extra_options = ccontext->extra_options;
re->flags = PCRE2_CODE_UNIT_WIDTH/8 | cb.external_flags | setflags;
re->limit_heap = limit_heap;
re->limit_match = limit_match;
re->limit_depth = limit_depth;
re->first_codeunit = 0;
re->last_codeunit = 0;
re->bsr_convention = bsr;
re->newline_convention = newline;
re->max_lookbehind = 0;
re->minlength = 0;
re->top_bracket = 0;
re->top_backref = 0;
re->name_entry_size = cb.name_entry_size;
re->name_count = cb.names_found;

# 计算编译后的代码的起始位置
codestart = (PCRE2_SPTR)((uint8_t *)re + sizeof(pcre2_real_code)) + re->name_entry_size * re->name_count;

# 更新编译数据块，传递名称/编号转换表和代码的起始位置
# 开始/结束模式和初始选项已经设置好
# 设置括号深度和断言深度为0
cb.parens_depth = 0;
cb.assert_depth = 0;
# 上一个捕获组的索引
cb.lastcapture = 0;
# 创建一个空的名称表
cb.name_table = (PCRE2_UCHAR *)((uint8_t *)re + sizeof(pcre2_real_code));
# 设置起始代码
cb.start_code = codestart;
# 请求可变选项为0
cb.req_varyopt = 0;
# 是否有接受状态
cb.had_accept = FALSE;
# 是否有剪枝或跳过状态
cb.had_pruneorskip = FALSE;
# 开放的捕获组为空

# 如果发现了任何命名组，从预处理中创建的列表中创建名称/编号表
if (cb.names_found > 0)
  {
  named_group *ng = cb.named_groups;
  for (i = 0; i < cb.names_found; i++, ng++)
    add_name_to_table(&cb, ng->name, ng->length, ng->number, i);
  }

# 设置一个起始的、非提取的括号，然后编译表达式。在错误时，errorcode 将被设置为非零，所以我们不需要在这里查看函数的结果
pptr = cb.parsed_pattern;
code = (PCRE2_UCHAR *)codestart;
*code = OP_BRA;
regexrc = compile_regex(re->overall_options, &code, &pptr, &errorcode, 0,
  &firstcu, &firstcuflags, &reqcu, &reqcuflags, NULL, &cb, NULL);
if (regexrc < 0) re->flags |= PCRE2_MATCH_EMPTY;
re->top_bracket = cb.bracount;
re->top_backref = cb.top_backref;
re->max_lookbehind = cb.max_lookbehind;

# 如果有接受状态
if (cb.had_accept)
  {
  reqcu = 0;                     /* 必须在 (*ACCEPT) 之后禁用 */
  reqcuflags = REQ_NONE;
  re->flags |= PCRE2_HASACCEPT;  /* 禁用最小长度 */
  }

# 填充最终的操作码并检查灾难性的溢出。如果没有溢出，但估计长度超过了实际使用的长度，则调整 re->blocksize 的值，并且如果配置了 valgrind 支持，则将额外分配的内存标记为不可寻址，以便检测到任何越界读取
*code++ = OP_END;
usedlength = code - codestart;
if (usedlength > length) errorcode = ERR23; else
  {
  re->blocksize -= CU2BYTES(length - usedlength);
#ifdef SUPPORT_VALGRIND
  VALGRIND_MAKE_MEM_NOACCESS(code, CU2BYTES(length - usedlength));
#endif
  }

# 扫描模式以查找递归/子例程调用并转换组
/* 定义递归缓存的大小 */
#define RSCAN_CACHE_SIZE 8

/* 如果没有错误并且存在递归，处理递归组的重复以提高效率 */
if (errorcode == 0 && cb.had_recurse)
  {
  PCRE2_UCHAR *rcode;
  PCRE2_SPTR rgroup;
  unsigned int ccount = 0;
  int start = RSCAN_CACHE_SIZE;
  recurse_cache rc[RSCAN_CACHE_SIZE];

  /* 查找递归的起始位置 */
  for (rcode = (PCRE2_UCHAR *)find_recurse(codestart, utf);
       rcode != NULL;
       rcode = (PCRE2_UCHAR *)find_recurse(rcode + 1 + LINK_SIZE, utf))
    {
    int p, groupnumber;

    groupnumber = (int)GET(rcode, 1);
    /* 如果组号为0，则递归组为codestart，否则需要查找递归组的位置 */
    if (groupnumber == 0) rgroup = codestart; else
      {
      PCRE2_SPTR search_from = codestart;
      rgroup = NULL;
      for (i = 0, p = start; i < ccount; i++, p = (p + 1) & 7)
        {
        /* 如果找到了之前的递归组，直接使用之前的位置 */
        if (groupnumber == rc[p].groupnumber)
          {
          rgroup = rc[p].group;
          break;
          }

        /* 新的组号大于之前找到的组号，可以节省搜索时间 */
        if (groupnumber > rc[p].groupnumber) search_from = rc[p].group;
        }

      /* 如果之前没有找到，需要重新查找并缓存位置 */
      if (rgroup == NULL)
        {
        rgroup = PRIV(find_bracket)(search_from, utf, groupnumber);
        if (rgroup == NULL)
          {
          errorcode = ERR53;
          break;
          }
        if (--start < 0) start = RSCAN_CACHE_SIZE - 1;
        rc[start].groupnumber = groupnumber;
        rc[start].group = rgroup;
        if (ccount < RSCAN_CACHE_SIZE) ccount++;
        }
      }

    /* 将递归组的偏移量写入 */
    PUT(rcode, 1, rgroup - codestart);
    }
  }

/* 在极少数的调试情况下，有时需要查看编译后的代码 */
#ifdef DEBUG_CALL_PRINTINT
pcre2_printint(re, stderr, TRUE);
fprintf(stderr, "Length=%lu Used=%lu\n", length, usedlength);
#endif

/* 除非禁用，检查是否可以自动转换任何单字符迭代器的性质 */
/* 
如果错误码为0，并且未设置 PCRE2_NO_AUTO_POSSESS 选项，则进行自动优化处理
使用中间变量 "temp" 是因为至少有一个编译器会在函数调用时给出关于丢失 "const" 属性的警告，如果直接使用 (PCRE2_UCHAR *)codestart 进行类型转换
*/
if (errorcode == 0 && (re->overall_options & PCRE2_NO_AUTO_POSSESS) == 0)
  {
  PCRE2_UCHAR *temp = (PCRE2_UCHAR *)codestart;
  如果自动优化处理失败，则设置错误码为 ERR80
  if (PRIV(auto_possessify)(temp, &cb) != 0) errorcode = ERR80;
  }

/* 编译失败，或者后处理时出错 */
if (errorcode != 0) goto HAD_CB_ERROR;

/* 编译成功。如果未传递锚定选项，则在可以确定模式由于 ^ 字符或 \A 或其他原因而被锚定时设置它，例如在启用 DOTALL 且没有出现 *PRUNE 或 *SKIP 时以非原子 .* 开头（尽管有一个选项可以禁用此情况）。 */
if ((re->overall_options & PCRE2_ANCHORED) == 0 &&
     is_anchored(codestart, 0, &cb, 0, FALSE))
  re->overall_options |= PCRE2_ANCHORED;

/* 设置第一个代码单元或 startline 标志，所需的代码单元，然后对模式进行研究。如果设置了 PCRE2_NO_START_OPTIMIZE，则不需要遵守此代码，因为它创建的数据将不会被使用。请注意，第一个代码单元（但不是 startline 标志）对于锚定模式是有用的，因为它仍然可以快速给出 "无匹配" 并且避免搜索最后一个代码单元。 */
if ((re->overall_options & PCRE2_NO_START_OPTIMIZE) == 0)
  {
  int minminlength = 0;  /* 用于从第一个/所需的代码单元获得最小长度 */

  /* 如果我们没有第一个代码单元，则查看是否有一个被断言的代码单元（在编译期间不保存这些数据，因为它们可能与后面的实际文字发生冲突）。 */
  if (firstcuflags >= REQ_NONE)
    firstcu = find_firstassertedcu(codestart, &firstcuflags, 0);

  /* 保存第一个代码单元的数据。其存在意味着最小长度必须至少为1。 */
  if (firstcuflags < REQ_NONE)
    {
    # 设置正则表达式的第一个代码单元
    re->first_codeunit = firstcu;
    # 设置标志位，表示已经设置了第一个代码单元
    re->flags |= PCRE2_FIRSTSET;
    # 最小最小长度加一
    minminlength++;

    # 处理不区分大小写的第一个代码单元
    if ((firstcuflags & REQ_CASELESS) != 0)
      {
      # 如果第一个代码单元小于128，或者不是UTF和UCP模式下小于255的情况
      if (firstcu < 128 || (!utf && !ucp && firstcu < 255))
        {
        # 如果不区分大小写的情况下，第一个代码单元对应的字符不等于自身，则设置标志位
        if (cb.fcc[firstcu] != firstcu) re->flags |= PCRE2_FIRSTCASELESS;
        }

      # 如果第一个代码单元大于128且处于UTF或UCP模式下，或者大于255且不处于UTF或UCP模式下
      # 在8位UTF模式下，范围在128-255的码点是介绍码点，不会有另一个大小写形式
      # 但如果设置了UCP，则可能有
#ifdef SUPPORT_UNICODE
#if PCRE2_CODE_UNIT_WIDTH == 8
      // 如果支持 Unicode 并且代码单元宽度为 8
      else if (ucp && !utf && UCD_OTHERCASE(firstcu) != firstcu)
        // 如果启用了 Unicode 属性，并且不是 UTF 编码，并且 firstcu 的其他大小写形式不等于 firstcu
        re->flags |= PCRE2_FIRSTCASELESS;
#else
      // 否则
      else if ((utf || ucp) && firstcu <= MAX_UTF_CODE_POINT &&
               UCD_OTHERCASE(firstcu) != firstcu)
        // 如果是 UTF 编码或启用了 Unicode 属性，并且 firstcu 小于等于最大 UTF 码点，并且 firstcu 的其他大小写形式不等于 firstcu
        re->flags |= PCRE2_FIRSTCASELESS;
#endif
#endif  /* SUPPORT_UNICODE */
      }
    }

  /* 当没有第一个代码单元时，对于非锚定模式，看看是否可以设置 PCRE2_STARTLINE 标志。
  这对于多行匹配很有帮助，当所有分支都以 ^ 开头时，以及当所有分支都以非原子的 .* 开头时，对于非 DOTALL 匹配，当 *PRUNE 和 SKIP 不存在时。（有一个选项禁用此情况。） */
  else if ((re->overall_options & PCRE2_ANCHORED) == 0 &&
           is_startline(codestart, 0, &cb, 0, FALSE))
    // 如果整体选项中不包含 PCRE2_ANCHORED，并且 is_startline 返回真
    re->flags |= PCRE2_STARTLINE;

  /* 处理“必需代码单元”，如果设置了的话。在 UTF 情况下，只有在我们确信这确实是一个不同的字符而不是第一个字符的非起始代码单元时，我们才能增加最小最小长度，因为最小长度计数是按字符而不是代码单元计算的。 */
  if (reqcuflags < REQ_NONE)
    // 如果 reqcuflags 小于 REQ_NONE
    {
#if PCRE2_CODE_UNIT_WIDTH == 16
    if ((re->overall_options & PCRE2_UTF) == 0 ||   /* Not UTF */
        firstcuflags >= REQ_NONE ||                 /* First not set */
        (firstcu & 0xf800) != 0xd800 ||             /* First not surrogate */
        (reqcu & 0xfc00) != 0xdc00)                 /* Req not low surrogate */
#elif PCRE2_CODE_UNIT_WIDTH == 8
    if ((re->overall_options & PCRE2_UTF) == 0 ||   /* Not UTF */
        firstcuflags >= REQ_NONE ||                 /* First not set */
        (firstcu & 0x80) == 0 ||                    /* First is ASCII */
        (reqcu & 0x80) == 0)                        /* Req is ASCII */
#endif
      {
      minminlength++;
      }
    }
    /* 如果是 16 位代码单元宽度，或者不是 UTF 编码或者第一个代码单元标志大于等于 REQ_NONE或者第一个代码单元不是代理，或者必需代码单元不是低代理
       或者是 8 位代码单元宽度，或者不是 UTF 编码或者第一个代码单元标志大于等于 REQ_NONE或者第一个代码单元是 ASCII，或者必需代码单元是 ASCII
       则增加最小最小长度
    */
    # 检查是否模式中存在可变长度的项目
    if ((re->overall_options & PCRE2_ANCHORED) == 0 || (reqcuflags & REQ_VARY) != 0)
      {
      # 设置最后一个代码单元，并将标志位置为已设置
      re->last_codeunit = reqcu;
      re->flags |= PCRE2_LASTSET;

      # 处理不区分大小写的必需代码单元，与上面处理第一个代码单元的方式类似
      if ((reqcuflags & REQ_CASELESS) != 0)
        {
        # 如果是ASCII字符或者不是UTF和UCP字符并且小于255，则进行处理
        if (reqcu < 128 || (!utf && !ucp && reqcu < 255))
          {
          # 如果不区分大小写，则设置标志位PCRE2_LASTCASELESS
          if (cb.fcc[reqcu] != reqcu) re->flags |= PCRE2_LASTCASELESS;
          }
#ifdef SUPPORT_UNICODE
#if PCRE2_CODE_UNIT_WIDTH == 8
      // 如果支持 Unicode 并且代码单元宽度为 8
      else if (ucp && !utf && UCD_OTHERCASE(reqcu) != reqcu)
        // 如果启用了 Unicode 属性表，并且不是 UTF 编码，并且请求的代码单元的其他大小写形式不等于请求的代码单元
        re->flags |= PCRE2_LASTCASELESS;
#else
      // 否则
      else if ((utf || ucp) && reqcu <= MAX_UTF_CODE_POINT &&
               UCD_OTHERCASE(reqcu) != reqcu)
        // 如果是 UTF 编码或启用了 Unicode 属性表，并且请求的代码单元小于等于最大 UTF 代码点，并且请求的代码单元的其他大小写形式不等于请求的代码单元
        re->flags |= PCRE2_LASTCASELESS;
#endif
#endif  /* SUPPORT_UNICODE */
        }
      }
    }

  /* 对编译好的模式进行研究，设置起始代码单元的位图和最小匹配长度等信息 */

  if (PRIV(study)(re) != 0)
    {
    errorcode = ERR31;
    goto HAD_CB_ERROR;
    }

  /* 如果 study() 设置了起始代码单元的位图，意味着最小长度至少为 1 */

  if ((re->flags & PCRE2_FIRSTMAPSET) != 0 && minminlength == 0)
    minminlength = 1;

  /* 如果 study() 设置的最小长度（或未设置）小于所需代码单元隐含的最小长度，则覆盖它 */

  if (re->minlength < minminlength) re->minlength = minminlength;
  }   /* 匹配开始时的优化结束 */

/* 在所有情况下都会执行到这里。在 valgrind 下运行时，重新定义模式的终止零。如果为解析版本的模式获取了内存，则在返回之前释放它。同样释放命名组列表，如果需要获取更大的列表，以及组信息向量。 */

EXIT:
#ifdef SUPPORT_VALGRIND
if (zero_terminated) VALGRIND_MAKE_MEM_DEFINED(pattern + patlen, CU2BYTES(1));
#endif
if (cb.parsed_pattern != stack_parsed_pattern)
  ccontext->memctl.free(cb.parsed_pattern, ccontext->memctl.memory_data);
if (cb.named_group_list_size > NAMED_GROUP_LIST_SIZE)
  ccontext->memctl.free((void *)cb.named_groups, ccontext->memctl.memory_data);
if (cb.groupinfo != stack_groupinfo)
  ccontext->memctl.free((void *)cb.groupinfo, ccontext->memctl.memory_data);
return re;    /* 在错误发生后将为 NULL */

/* 在 parse_regex() 中发现的错误会设置编译中的偏移值 */
/* 在调用之前发现的错误必须从指针值计算出来。
   在调用parse_regex()之后，编译块中的偏移量被设置为模式的末尾，
   但是在compile_regex()中的某些错误可能会重置它，如果在解析模式中有可用的偏移量。 */

HAD_CB_ERROR:
ptr = pattern + cb.erroroffset;

HAD_EARLY_ERROR:
*erroroffset = ptr - pattern;

HAD_ERROR:
*errorptr = errorcode;
pcre2_code_free(re);
re = NULL;
goto EXIT;
}

/* 这些#undef是为了使CMake能够启用统一构建。 */

#undef NLBLOCK /* 包含换行符信息的块 */
#undef PSSTART /* 包含处理过的字符串起始位置的字段 */
#undef PSEND   /* 包含处理过的字符串结束位置的字段 */

/* pcre2_compile.c的结尾 */
```