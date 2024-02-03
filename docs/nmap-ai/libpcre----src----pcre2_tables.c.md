# `nmap\libpcre\src\pcre2_tables.c`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2021 University of Cambridge

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
/* This module contains some fixed tables that are used by more than one of the
PCRE2 code modules. The tables are also #included by the pcre2test program,
which uses macros to change their names from _pcre2_xxx to xxxx, thereby
avoiding name clashes with the library. In this case, PCRE2_PCRE2TEST is
defined. */

#ifndef PCRE2_PCRE2TEST           /* We're compiling the library */
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
#include "pcre2_internal.h"
#endif /* PCRE2_PCRE2TEST */

/* Table of sizes for the fixed-length opcodes. It's defined in a macro so that
the definition is next to the definition of the opcodes in pcre2_internal.h.
This is mode-dependent, so it is skipped when this file is included by
pcre2test. */

#ifndef PCRE2_PCRE2TEST
const uint8_t PRIV(OP_lengths)[] = { OP_LENGTHS };
#endif

/* Tables of horizontal and vertical whitespace characters, suitable for
adding to classes. */

const uint32_t PRIV(hspace_list)[] = { HSPACE_LIST };
const uint32_t PRIV(vspace_list)[] = { VSPACE_LIST };

/* These tables are the pairs of delimiters that are valid for callout string
arguments. For each starting delimiter there must be a matching ending
delimiter, which in fact is different only for bracket-like delimiters. */

const uint32_t PRIV(callout_start_delims)[] = {
  CHAR_GRAVE_ACCENT, CHAR_APOSTROPHE, CHAR_QUOTATION_MARK,
  CHAR_CIRCUMFLEX_ACCENT, CHAR_PERCENT_SIGN, CHAR_NUMBER_SIGN,
  CHAR_DOLLAR_SIGN, CHAR_LEFT_CURLY_BRACKET, 0 };

const uint32_t PRIV(callout_end_delims[]) = {
  CHAR_GRAVE_ACCENT, CHAR_APOSTROPHE, CHAR_QUOTATION_MARK,
  CHAR_CIRCUMFLEX_ACCENT, CHAR_PERCENT_SIGN, CHAR_NUMBER_SIGN,
  CHAR_DOLLAR_SIGN, CHAR_RIGHT_CURLY_BRACKET, 0 };


/*************************************************
*           Tables for UTF-8 support             *
/*************************************************/

/* 这些表格在16位或32位模式下由pcre2test需要，同时也为8位模式下的库所需，因为pcre2test在内部使用UTF-8处理宽字符。 */

#if defined PCRE2_PCRE2TEST || \
   (defined SUPPORT_UNICODE && \
    defined PCRE2_CODE_UNIT_WIDTH && \
    PCRE2_CODE_UNIT_WIDTH == 8)

/* 这些是UTF-8字符中不同字节数的断点。 */

const int PRIV(utf8_table1)[] =
  { 0x7f, 0x7ff, 0xffff, 0x1fffff, 0x3ffffff, 0x7fffffff};

const int PRIV(utf8_table1_size) = sizeof(PRIV(utf8_table1)) / sizeof(int);

/* 这些是指示位和掩码，用于设置字符的第一个字节中的数据位，按照附加字节数进行索引。 */

const int PRIV(utf8_table2)[] = { 0,    0xc0, 0xe0, 0xf0, 0xf8, 0xfc};
const int PRIV(utf8_table3)[] = { 0xff, 0x1f, 0x0f, 0x07, 0x03, 0x01};

/* 通过第一个字节与0x3f进行掩码操作，得到额外字节数的表格。有效UTF-8第一个字节的最大值实际上是0x3d。 */

const uint8_t PRIV(utf8_table4)[] = {
  1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
  1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
  2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,
  3,3,3,3,3,3,3,3,4,4,4,4,5,5,5,5 };

#endif /* UTF-8 support needed */

/* 与Unicode属性相关的表格仅在启用Unicode支持时才相关。另请参阅pcre2_ucptables.c文件，该文件是由Python脚本从Unicode数据文件生成的。 */

#ifdef SUPPORT_UNICODE

/* 用于将特定类型值转换为通用值的表格。 */
const uint32_t PRIV(ucp_gentype)[] = {
  ucp_C, ucp_C, ucp_C, ucp_C, ucp_C,  /* Cc, Cf, Cn, Co, Cs */
  ucp_L, ucp_L, ucp_L, ucp_L, ucp_L,  /* Ll, Lu, Lm, Lo, Lt */
  ucp_M, ucp_M, ucp_M,                /* Mc, Me, Mn */
  ucp_N, ucp_N, ucp_N,                /* Nd, Nl, No */
  ucp_P, ucp_P, ucp_P, ucp_P, ucp_P,  /* Pc, Pd, Pe, Pf, Pi */
  ucp_P, ucp_P,                       /* Ps, Po */
  ucp_S, ucp_S, ucp_S, ucp_S,         /* Sc, Sk, Sm, So */
  ucp_Z, ucp_Z, ucp_Z                 /* Zl, Zp, Zs */
};
/* 定义一个名为 PRIV(ucp_gentype) 的常量数组，包含 Unicode 代码点的通用类型 */

/* This table encodes the rules for finding the end of an extended grapheme
cluster. Every code point has a grapheme break property which is one of the
ucp_gbXX values defined in pcre2_ucp.h. These changed between Unicode versions
10 and 11. The 2-dimensional table is indexed by the properties of two adjacent
code points. The left property selects a word from the table, and the right
property selects a bit from that word like this:

  PRIV(ucp_gbtable)[left-property] & (1u << right-property)

The value is non-zero if a grapheme break is NOT permitted between the relevant
two code points. The breaking rules are as follows:

1. Break at the start and end of text (pretty obviously).

2. Do not break between a CR and LF; otherwise, break before and after
   controls.

3. Do not break Hangul syllable sequences, the rules for which are:

    L may be followed by L, V, LV or LVT
    LV or V may be followed by V or T
    LVT or T may be followed by T

4. Do not break before extending characters or zero-width-joiner (ZWJ).

The following rules are only for extended grapheme clusters (but that's what we
are implementing).

5. Do not break before SpacingMarks.

6. Do not break after Prepend characters.
/*
   7. 如果字符具有 Extended_Pictographic 属性，则不要在表情符号修饰符序列或表情符号零宽连接序列中断。允许在字符之间使用 Extend 和 ZWJ 字符；这个表格无法表示，代码必须处理它。

   8. 如果在表情符号国旗序列中断，则不要在区域指示符（RI）符号之间中断，如果中断点之前有奇数个 RI 字符。这个表格编码了“连接 RI 字符”；代码必须处理检查前面的相邻 RI。

   9. 否则，可以在任何地方中断。
*/

#define ESZ (1<<ucp_gbExtend)|(1<<ucp_gbSpacingMark)|(1<<ucp_gbZWJ)

const uint32_t PRIV(ucp_gbtable)[] = {
   (1u<<ucp_gbLF),                                      /*  0 CR */
   0,                                                   /*  1 LF */
   0,                                                   /*  2 Control */
   ESZ,                                                 /*  3 Extend */
   ESZ|(1u<<ucp_gbPrepend)|                             /*  4 Prepend */
       (1u<<ucp_gbL)|(1u<<ucp_gbV)|(1u<<ucp_gbT)|
       (1u<<ucp_gbLV)|(1u<<ucp_gbLVT)|(1u<<ucp_gbOther)|
       (1u<<ucp_gbRegional_Indicator),
   ESZ,                                                 /*  5 SpacingMark */
   ESZ|(1u<<ucp_gbL)|(1u<<ucp_gbV)|(1u<<ucp_gbLV)|      /*  6 L */
       (1u<<ucp_gbLVT),
   ESZ|(1u<<ucp_gbV)|(1u<<ucp_gbT),                     /*  7 V */
   ESZ|(1u<<ucp_gbT),                                   /*  8 T */
   ESZ|(1u<<ucp_gbV)|(1u<<ucp_gbT),                     /*  9 LV */
   ESZ|(1u<<ucp_gbT),                                   /* 10 LVT */
   (1u<<ucp_gbRegional_Indicator),                      /* 11 Regional Indicator */
   ESZ,                                                 /* 12 Other */
   ESZ,                                                 /* 13 ZWJ */
   ESZ|(1u<<ucp_gbExtended_Pictographic)                /* 14 Extended Pictographic */
};

#undef ESZ
#ifdef SUPPORT_JIT
/* 如果支持即时编译，则定义并初始化一个反转 PRIV(ucp_gentype) 的表，可以节省内存加载的成本 */

const int PRIV(ucp_typerange)[] = {
  ucp_Cc, ucp_Cs,
  ucp_Ll, ucp_Lu,
  ucp_Mc, ucp_Mn,
  ucp_Nd, ucp_No,
  ucp_Pc, ucp_Ps,
  ucp_Sc, ucp_So,
  ucp_Zl, ucp_Zs,
};
#endif /* SUPPORT_JIT */

/* 最后，包含从 Unicode 数据文件自动生成的表 */

#include "pcre2_ucptables.c"

#endif /* SUPPORT_UNICODE */

/* pcre2_tables.c 的结束 */
```