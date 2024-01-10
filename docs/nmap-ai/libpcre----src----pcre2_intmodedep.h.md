# `nmap\libpcre\src\pcre2_intmodedep.h`

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
/* This module contains mode-dependent macro and structure definitions. The
file is #included by pcre2_internal.h if PCRE2_CODE_UNIT_WIDTH is defined.
These mode-dependent items are kept in a separate file so that they can also be
#included multiple times for different code unit widths by pcre2test in order
to have access to the hidden structures at all supported widths.

Some of the mode-dependent macros are required at different widths for
different parts of the pcre2test code (in particular, the included
pcre_printint.c file). We undefine them here so that they can be re-defined for
multiple inclusions. Not all of these are used in pcre2test, but it's easier
just to undefine them all. */

#undef ACROSSCHAR
#undef BACKCHAR
#undef BYTES2CU
#undef CHMAX_255
#undef CU2BYTES
#undef FORWARDCHAR
#undef FORWARDCHARTEST
#undef GET
#undef GET2
#undef GETCHAR
#undef GETCHARINC
#undef GETCHARINCTEST
#undef GETCHARLEN
#undef GETCHARLENTEST
#undef GETCHARTEST
#undef GET_EXTRALEN
#undef HAS_EXTRALEN
#undef IMM2_SIZE
#undef MAX_255
#undef MAX_MARK
#undef MAX_PATTERN_SIZE
#undef MAX_UTF_SINGLE_CU
#undef NOT_FIRSTCU
#undef PUT
#undef PUT2
#undef PUT2INC
#undef PUTCHAR
#undef PUTINC
#undef TABLE_GET



/* -------------------------- MACROS ----------------------------- */

/* PCRE keeps offsets in its compiled code as at least 16-bit quantities
(always stored in big-endian order in 8-bit mode) by default. These are used,
for example, to link from the start of a subpattern to its alternatives and its
end. The use of 16 bits per offset limits the size of an 8-bit compiled regex
to around 64K, which is big enough for almost everybody. However, I received a
request for an even bigger limit. For this reason, and also to make the code
/* ------------------- 8-bit support  ------------------ */

#if PCRE2_CODE_UNIT_WIDTH == 8

#if LINK_SIZE == 2
#define PUT(a,n,d)   \  # 定义宏PUT，将数据d存入数组a的第n和n+1位置
  (a[n] = (PCRE2_UCHAR)((d) >> 8)), \  # 将d的高8位存入a[n]
  (a[(n)+1] = (PCRE2_UCHAR)((d) & 255))  # 将d的低8位存入a[n+1]
#define GET(a,n) \  # 定义宏GET，从数组a的第n和n+1位置获取数据
  (unsigned int)(((a)[n] << 8) | (a)[(n)+1])  # 将a[n]左移8位后与a[n+1]进行或运算
#define MAX_PATTERN_SIZE (1 << 16)  # 最大模式大小为2^16

#elif LINK_SIZE == 3
#define PUT(a,n,d)       \  # 定义宏PUT，将数据d存入数组a的第n、n+1和n+2位置
  (a[n] = (PCRE2_UCHAR)((d) >> 16)),    \  # 将d的高16位存入a[n]
  (a[(n)+1] = (PCRE2_UCHAR)((d) >> 8)), \  # 将d的中间8位存入a[n+1]
  (a[(n)+2] = (PCRE2_UCHAR)((d) & 255))  # 将d的低8位存入a[n+2]
#define GET(a,n) \  # 定义宏GET，从数组a的第n、n+1和n+2位置获取数据
  (unsigned int)(((a)[n] << 16) | ((a)[(n)+1] << 8) | (a)[(n)+2])  # 将a[n]左移16位后与a[n+1]左移8位再与a[n+2]进行或运算
#define MAX_PATTERN_SIZE (1 << 24)  # 最大模式大小为2^24

#elif LINK_SIZE == 4
#define PUT(a,n,d)        \  # 定义宏PUT，将数据d存入数组a的第n、n+1、n+2和n+3位置
  (a[n] = (PCRE2_UCHAR)((d) >> 24)),     \  # 将d的最高8位存入a[n]
  (a[(n)+1] = (PCRE2_UCHAR)((d) >> 16)), \  # 将d的次高8位存入a[n+1]
  (a[(n)+2] = (PCRE2_UCHAR)((d) >> 8)),  \  # 将d的中间8位存入a[n+2]
  (a[(n)+3] = (PCRE2_UCHAR)((d) & 255))  # 将d的低8位存入a[n+3]
#define GET(a,n) \  # 定义宏GET，从数组a的第n、n+1、n+2和n+3位置获取数据
  (unsigned int)(((a)[n] << 24) | ((a)[(n)+1] << 16) | ((a)[(n)+2] << 8) | (a)[(n)+3])  # 将a[n]左移24位后与a[n+1]左移16位再与a[n+2]左移8位再与a[n+3]进行或运算
#define MAX_PATTERN_SIZE (1 << 30)   /* Keep it positive */  # 最大模式大小为2^30，保持为正数

#else
#error LINK_SIZE must be 2, 3, or 4  # 如果LINK_SIZE不是2、3或4，则报错
#endif


/* ------------------- 16-bit support  ------------------ */

#elif PCRE2_CODE_UNIT_WIDTH == 16

#if LINK_SIZE == 2
#undef LINK_SIZE  # 取消LINK_SIZE的定义
#define LINK_SIZE 1  # 重新定义LINK_SIZE为1
#define PUT(a,n,d)   \  # 定义宏PUT，将数据d存入数组a的第n位置
  (a[n] = (PCRE2_UCHAR)(d))  # 将d存入a[n]
#define GET(a,n) \  # 定义宏GET，从数组a的第n位置获取数据
  (a[n])  # 获取a[n]
#define MAX_PATTERN_SIZE (1 << 16)  # 最大模式大小为2^16

#elif LINK_SIZE == 3 || LINK_SIZE == 4
#undef LINK_SIZE  # 取消LINK_SIZE的定义
#define LINK_SIZE 2  # 重新定义LINK_SIZE为2
#define PUT(a,n,d)   \  # 定义宏PUT，将数据d存入数组a的第n和n+1位置
  (a[n] = (PCRE2_UCHAR)((d) >> 16)), \  # 将d的高16位存入a[n]
  (a[(n)+1] = (PCRE2_UCHAR)((d) & 65535))  # 将d的低16位存入a[n+1]
#define GET(a,n) \  # 定义宏GET，从数组a的第n和n+1位置获取数据
  (unsigned int)(((a)[n] << 16) | (a)[(n)+1])  # 将a[n]左移16位后与a[n+1]进行或运算
#define MAX_PATTERN_SIZE (1 << 30)  /* Keep it positive */  # 最大模式大小为2^30，保持为正数

#else
#error LINK_SIZE must be 2, 3, or 4  # 如果LINK_SIZE不是2、3或4，则报错
#endif


/* ------------------- 32-bit support  ------------------ */
#elif PCRE2_CODE_UNIT_WIDTH == 32
#undef LINK_SIZE
#define LINK_SIZE 1
#define PUT(a,n,d)   \
  (a[n] = (d))
#define GET(a,n) \
  (a[n])
#define MAX_PATTERN_SIZE (1 << 30)  /* Keep it positive */

#else
#error Unsupported compiling mode
#endif


/* --------------- Other mode-specific macros ----------------- */

/* PCRE uses some other (at least) 16-bit quantities that do not change when
the size of offsets changes. There are used for repeat counts and for other
things such as capturing parenthesis numbers in back references.

Define the number of code units required to hold a 16-bit count/offset, and
macros to load and store such a value. For reasons that I do not understand,
the expression in the 8-bit GET2 macro is treated by gcc as a signed
expression, even when a is declared as unsigned. It seems that any kind of
arithmetic results in a signed value. Hence the cast. */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define IMM2_SIZE 2
#define GET2(a,n) (unsigned int)(((a)[n] << 8) | (a)[(n)+1])
#define PUT2(a,n,d) a[n] = (d) >> 8, a[(n)+1] = (d) & 255

#else  /* Code units are 16 or 32 bits */
#define IMM2_SIZE 1
#define GET2(a,n) a[n]
#define PUT2(a,n,d) a[n] = d
#endif

/* Other macros that are different for 8-bit mode. The MAX_255 macro checks
whether its argument, which is assumed to be one code unit, is less than 256.
The CHMAX_255 macro does not assume one code unit. The maximum length of a MARK
name must fit in one code unit; currently it is set to 255 or 65535. The
TABLE_GET macro is used to access elements of tables containing exactly 256
items. Its argument is a code unit. When code points can be greater than 255, a
check is needed before accessing these tables. */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define MAX_255(c) TRUE
#define MAX_MARK ((1u << 8) - 1)
#define TABLE_GET(c, table, default) ((table)[c])
#ifdef SUPPORT_UNICODE
#define SUPPORT_WIDE_CHARS
#define CHMAX_255(c) ((c) <= 255u)
#else
#define CHMAX_255(c) TRUE
#endif  /* SUPPORT_UNICODE */
#else  /* Code units are 16 or 32 bits */
#define CHMAX_255(c) ((c) <= 255u)  // 定义一个宏，用于判断字符是否小于等于255
#define MAX_255(c) ((c) <= 255u)  // 定义一个宏，用于判断字符是否小于等于255
#define MAX_MARK ((1u << 16) - 1)  // 定义一个最大标记值
#define SUPPORT_WIDE_CHARS  // 定义一个宏，表示支持宽字符
#define TABLE_GET(c, table, default) (MAX_255(c)? ((table)[c]):(default))  // 定义一个宏，用于从表中获取字符，如果字符小于等于255则直接从表中获取，否则返回默认值
#endif


/* ----------------- Character-handling macros ----------------- */

/* There is a proposed future special "UTF-21" mode, in which only the lowest
21 bits of a 32-bit character are interpreted as UTF, with the remaining 11
high-order bits available to the application for other uses. In preparation for
the future implementation of this mode, there are macros that load a data item
and, if in this special mode, mask it to 21 bits. These macros all have names
starting with UCHAR21. In all other modes, including the normal 32-bit
library, the macros all have the same simple definitions. When the new mode is
implemented, it is expected that these definitions will be varied appropriately
using #ifdef when compiling the library that supports the special mode. */

#define UCHAR21(eptr)        (*(eptr))  // 定义一个宏，用于获取数据项并在特殊模式下掩码为21位
#define UCHAR21TEST(eptr)    (*(eptr))  // 定义一个宏，用于测试数据项并在特殊模式下掩码为21位
#define UCHAR21INC(eptr)     (*(eptr)++)  // 定义一个宏，用于递增数据项并在特殊模式下掩码为21位
#define UCHAR21INCTEST(eptr) (*(eptr)++)  // 定义一个宏，用于递增数据项并在特殊模式下掩码为21位

/* When UTF encoding is being used, a character is no longer just a single
byte in 8-bit mode or a single short in 16-bit mode. The macros for character
handling generate simple sequences when used in the basic mode, and more
complicated ones for UTF characters. GETCHARLENTEST and other macros are not
used when UTF is not supported. To make sure they can never even appear when
UTF support is omitted, we don't even define them. */

#ifndef SUPPORT_UNICODE

/* #define MAX_UTF_SINGLE_CU */
/* #define HAS_EXTRALEN(c) */
/* #define GET_EXTRALEN(c) */
/* #define NOT_FIRSTCU(c) */
#define GETCHAR(c, eptr) c = *eptr;  // 定义一个宏，用于获取字符
#define GETCHARTEST(c, eptr) c = *eptr;  // 定义一个宏，用于测试获取字符
#define GETCHARINC(c, eptr) c = *eptr++;  // 定义一个宏，用于获取字符并递增指针
#define GETCHARINCTEST(c, eptr) c = *eptr++;  // 定义一个宏，用于测试获取字符并递增指针
#define GETCHARLEN(c, eptr, len) c = *eptr;  // 定义一个宏，用于获取字符和长度
#define PUTCHAR(c, p) (*p = c, 1)
/* 定义宏 PUTCHAR，将字符 c 放入指针 p 指向的位置，并返回 1 */

/* #define GETCHARLENTEST(c, eptr, len) */
/* 定义宏 GETCHARLENTEST，但是注释掉了 */

/* #define BACKCHAR(eptr) */
/* 定义宏 BACKCHAR，但是注释掉了 */

/* #define FORWARDCHAR(eptr) */
/* 定义宏 FORWARDCHAR，但是注释掉了 */

/* #define FORWARCCHARTEST(eptr,end) */
/* 定义宏 FORWARCCHARTEST，但是注释掉了 */

/* #define ACROSSCHAR(condition, eptr, action) */
/* 定义宏 ACROSSCHAR，但是注释掉了 */

#else   /* SUPPORT_UNICODE */

/* ------------------- 8-bit support  ------------------ */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define MAYBE_UTF_MULTI          /* UTF chars may use multiple code units */
/* 定义宏 MAYBE_UTF_MULTI，表示 UTF 字符可能使用多个代码单元 */

/* The largest UTF code point that can be encoded as a single code unit. */
/* 可以编码为单个代码单元的最大 UTF 代码点。 */
#define MAX_UTF_SINGLE_CU 127

/* Tests whether the code point needs extra characters to decode. */
/* 测试代码点是否需要额外的字符来解码。 */
#define HAS_EXTRALEN(c) HASUTF8EXTRALEN(c)

/* Returns with the additional number of characters if IS_MULTICHAR(c) is TRUE.
Otherwise it has an undefined behaviour. */
/* 如果 IS_MULTICHAR(c) 为 TRUE，则返回额外的字符数。否则，它具有未定义的行为。 */
#define GET_EXTRALEN(c) (PRIV(utf8_table4)[(c) & 0x3fu])

/* Returns TRUE, if the given value is not the first code unit of a UTF
sequence. */
/* 如果给定值不是 UTF 序列的第一个代码单元，则返回 TRUE。 */
#define NOT_FIRSTCU(c) (((c) & 0xc0u) == 0x80u)

/* Get the next UTF-8 character, not advancing the pointer. This is called when
we know we are in UTF-8 mode. */
/* 获取下一个 UTF-8 字符，不推进指针。当我们知道我们处于 UTF-8 模式时调用。 */
#define GETCHAR(c, eptr) \
  c = *eptr; \
  if (c >= 0xc0u) GETUTF8(c, eptr);

/* Get the next UTF-8 character, testing for UTF-8 mode, and not advancing the
pointer. */
/* 获取下一个 UTF-8 字符，测试 UTF-8 模式，并且不推进指针。 */
#define GETCHARTEST(c, eptr) \
  c = *eptr; \
  if (utf && c >= 0xc0u) GETUTF8(c, eptr);

/* Get the next UTF-8 character, advancing the pointer. This is called when we
know we are in UTF-8 mode. */
/* 获取下一个 UTF-8 字符，推进指针。当我们知道我们处于 UTF-8 模式时调用。 */
#define GETCHARINC(c, eptr) \
  c = *eptr++; \
  if (c >= 0xc0u) GETUTF8INC(c, eptr);

/* Get the next character, testing for UTF-8 mode, and advancing the pointer.
This is called when we don't know if we are in UTF-8 mode. */
/* 获取下一个字符，测试 UTF-8 模式，并且推进指针。当我们不知道我们是否处于 UTF-8 模式时调用。 */
#define GETCHARINCTEST(c, eptr) \
  c = *eptr++; \
  if (utf && c >= 0xc0u) GETUTF8INC(c, eptr);

/* Get the next UTF-8 character, not advancing the pointer, incrementing length
if there are extra bytes. This is called when we know we are in UTF-8 mode. */
/* 获取下一个 UTF-8 字符，不推进指针，如果有额外的字节，则增加长度。当我们知道我们处于 UTF-8 模式时调用。 */
/* 定义宏，用于获取字符长度，包括处理 UTF-8 编码的情况 */
#define GETCHARLEN(c, eptr, len) \
  c = *eptr; \  // 获取指针指向的字符
  if (c >= 0xc0u) GETUTF8LEN(c, eptr, len);  // 如果字符大于等于 0xc0u，调用 GETUTF8LEN 宏获取字符长度

/* 获取下一个 UTF-8 字符，测试是否处于 UTF-8 模式，不移动指针，如果有额外的字节则增加长度。当我们不确定是否处于 UTF-8 模式时调用。 */
#define GETCHARLENTEST(c, eptr, len) \
  c = *eptr; \  // 获取指针指向的字符
  if (utf && c >= 0xc0u) GETUTF8LEN(c, eptr, len);  // 如果处于 UTF-8 模式且字符大于等于 0xc0u，调用 GETUTF8LEN 宏获取字符长度

/* 如果指针不在字符的起始位置，将其移回直到处于起始位置。仅在 UTF-8 模式下调用 - 我们不在宏内部放置测试，因为几乎所有调用都已经在 UTF-8 代码块内。 */
#define BACKCHAR(eptr) while((*eptr & 0xc0u) == 0x80u) eptr--  // 如果指针不在字符的起始位置，向前移动直到处于起始位置

/* 与上面相同，只是方向相反。 */
#define FORWARDCHAR(eptr) while((*eptr & 0xc0u) == 0x80u) eptr++  // 如果指针不在字符的起始位置，向后移动直到处于起始位置
#define FORWARDCHARTEST(eptr,end) while(eptr < end && (*eptr & 0xc0u) == 0x80u) eptr++  // 如果指针不在字符的起始位置且未到达末尾，向后移动直到处于起始位置

/* 允许完全可定制的形式。 */
#define ACROSSCHAR(condition, eptr, action) \
  while((condition) && ((*eptr) & 0xc0u) == 0x80u) action  // 在满足条件的情况下，如果指针不在字符的起始位置，执行指定的操作

/* 将字符存入内存，返回代码单元的数量。 */
#define PUTCHAR(c, p) ((utf && c > MAX_UTF_SINGLE_CU)? \
  PRIV(ord2utf)(c,p) : (*p = c, 1))  // 如果处于 UTF 模式且字符大于 MAX_UTF_SINGLE_CU，则调用 PRIV(ord2utf) 宏存入内存，否则直接存入内存并返回 1

/* ------------------- 16-bit support  ------------------ */

#elif PCRE2_CODE_UNIT_WIDTH == 16
#define MAYBE_UTF_MULTI          /* UTF chars may use multiple code units */

/* 可以编码为单个代码单元的最大 UTF 代码点。 */
#define MAX_UTF_SINGLE_CU 65535

/* 测试代码点是否需要额外的字符来解码。 */
#define HAS_EXTRALEN(c) (((c) & 0xfc00u) == 0xd800u)

/* 如果 IS_MULTICHAR(c) 为 TRUE，则返回额外的字符数。否则具有未定义的行为。 */
#define GET_EXTRALEN(c) 1

/* 如果给定值不是 UTF-16 字符的第一个代码单元，则返回 TRUE。 */
#define NOT_FIRSTCU(c) (((c) & 0xfc00u) == 0xdc00u)

/* 基本宏，用于获取 UTF-16 字符的低代理项，不是 UTF-16 字符的第一个代码单元时调用。 */
/* Base macro to pick up the low surrogate of a UTF-16 character, not
advancing the pointer. */

#define GETUTF16(c, eptr) \
   { c = (((c & 0x3ffu) << 10) | (eptr[1] & 0x3ffu)) + 0x10000u; }

/* Get the next UTF-16 character, not advancing the pointer. This is called when
we know we are in UTF-16 mode. */

#define GETCHAR(c, eptr) \
  c = *eptr; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16(c, eptr);

/* Get the next UTF-16 character, testing for UTF-16 mode, and not advancing the
pointer. */

#define GETCHARTEST(c, eptr) \
  c = *eptr; \
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16(c, eptr);

/* Base macro to pick up the low surrogate of a UTF-16 character, advancing
the pointer. */

#define GETUTF16INC(c, eptr) \
   { c = (((c & 0x3ffu) << 10) | (*eptr++ & 0x3ffu)) + 0x10000u; }

/* Get the next UTF-16 character, advancing the pointer. This is called when we
know we are in UTF-16 mode. */

#define GETCHARINC(c, eptr) \
  c = *eptr++; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16INC(c, eptr);

/* Get the next character, testing for UTF-16 mode, and advancing the pointer.
This is called when we don't know if we are in UTF-16 mode. */

#define GETCHARINCTEST(c, eptr) \
  c = *eptr++; \
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16INC(c, eptr);

/* Base macro to pick up the low surrogate of a UTF-16 character, not
advancing the pointer, incrementing the length. */

#define GETUTF16LEN(c, eptr, len) \
   { c = (((c & 0x3ffu) << 10) | (eptr[1] & 0x3ffu)) + 0x10000u; len++; }

/* Get the next UTF-16 character, not advancing the pointer, incrementing
length if there is a low surrogate. This is called when we know we are in
UTF-16 mode. */

#define GETCHARLEN(c, eptr, len) \
  c = *eptr; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16LEN(c, eptr, len);

/* Get the next UTF-816character, testing for UTF-16 mode, not advancing the
pointer, incrementing length if there is a low surrogate. This is called when
we do not know if we are in UTF-16 mode. */
/* 定义一个宏，用于获取字符的长度，并在 UTF-16 模式下进行测试 */
#define GETCHARLENTEST(c, eptr, len) \
  c = *eptr; \  // 将指针 eptr 指向的字符赋值给变量 c
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16LEN(c, eptr, len);  // 如果是 UTF-16 模式，并且字符的高 10 位为 110110，调用 GETUTF16LEN 宏获取字符长度

/* 如果指针不在字符的起始位置，将其向前移动直到起始位置。这仅在 UTF-16 模式下调用 - 我们不在宏内部放置测试，因为几乎所有调用都已经在 UTF-16 代码块内部。 */
#define BACKCHAR(eptr) if ((*eptr & 0xfc00u) == 0xdc00u) eptr--  // 如果指针指向的字符是低 10 位为 110110，将指针向前移动一位

/* 和上面的宏相同，只是方向相反。 */
#define FORWARDCHAR(eptr) if ((*eptr & 0xfc00u) == 0xdc00u) eptr++  // 如果指针指向的字符是低 10 位为 110110，将指针向后移动一位
#define FORWARDCHARTEST(eptr,end) if (eptr < end && (*eptr & 0xfc00u) == 0xdc00u) eptr++  // 如果指针在结束位置之前，并且指向的字符是低 10 位为 110110，将指针向后移动一位

/* 和上面的宏相同，但允许完全可定制的形式。 */
#define ACROSSCHAR(condition, eptr, action) \  // 定义一个宏，根据条件和指针指向的字符是否为低 10 位为 110110，执行相应的操作
  if ((condition) && ((*eptr) & 0xfc00u) == 0xdc00u) action

/* 将字符存入内存，返回代码单元的数量。 */
#define PUTCHAR(c, p) ((utf && c > MAX_UTF_SINGLE_CU)? \  // 如果是 UTF 模式，并且字符大于最大的单个代码单元，调用 PRIV(ord2utf) 宏将字符存入内存，否则将字符存入内存并返回 1
  PRIV(ord2utf)(c,p) : (*p = c, 1))


/* ------------------- 32-bit support  ------------------ */

#else

/* 对于 32 位库来说，这些都是微不足道的，因为所有的 UTF-32 字符都适合一个 PCRE2_UCHAR 单元。 */
#define MAX_UTF_SINGLE_CU (0x10ffffu)  // 最大的单个代码单元
#define HAS_EXTRALEN(c) (0)  // 是否有额外的长度
#define GET_EXTRALEN(c) (0)  // 获取额外的长度
#define NOT_FIRSTCU(c) (0)  // 不是第一个代码单元

/* 获取下一个 UTF-32 字符，不移动指针。这在我们知道我们处于 UTF-32 模式时调用。 */
#define GETCHAR(c, eptr) \
  c = *(eptr);

/* 获取下一个 UTF-32 字符，测试是否处于 UTF-32 模式，并且不移动指针。 */
#define GETCHARTEST(c, eptr) \
  c = *(eptr);

/* 获取下一个 UTF-32 字符，移动指针。这在我们知道我们处于 UTF-32 模式时调用。 */
#define GETCHARINC(c, eptr) \
  c = *((eptr)++);

/* 获取下一个字符，测试是否处于 UTF-32 模式，并且移动指针。这在我们不知道是否处于 UTF-32 模式时调用。 */
#define GETCHARINCTEST(c, eptr) \
  c = *((eptr)++);

/* 获取下一个 UTF-32 字符，不移动指针，不增加
/* 
   定义宏，用于获取下一个 UTF-32 字符，不测试是否处于 UTF-32 模式，不移动指针，不增加长度（因为所有 UTF-32 都是长度为1的）。
   在我们知道处于 UTF-32 模式时调用。
*/
#define GETCHARLEN(c, eptr, len) \
  GETCHAR(c, eptr)

/* 
   定义宏，用于获取下一个 UTF-32 字符，测试是否处于 UTF-32 模式，不移动指针，不增加长度（因为所有 UTF-32 都是长度为1的）。
   在我们不确定是否处于 UTF-32 模式时调用。
*/
#define GETCHARLENTEST(c, eptr, len) \
  GETCHARTEST(c, eptr)

/* 
   如果指针不在字符的起始位置，将其移回直到处于起始位置。仅在 UTF-32 模式下调用 - 我们不在宏内部放置测试，因为几乎所有调用已经在 UTF-32 代码块内部。
   这些都是空操作，因为所有 UTF-32 字符都适合一个 PCRE2_UCHAR。
*/
#define BACKCHAR(eptr) do { } while (0)

/* 
   与上面相同，只是方向相反。
*/
#define FORWARDCHAR(eptr) do { } while (0)
#define FORWARDCHARTEST(eptr,end) do { } while (0)

/* 
   与上面相同，但允许完全可定制的形式。
*/
#define ACROSSCHAR(condition, eptr, action) do { } while (0)

/* 
   将字符存入内存，返回代码单元的数量。
*/
#define PUTCHAR(c, p) (*p = c, 1)

#endif  /* UTF-32 字符处理 */
#endif  /* 支持 Unicode */

/* 在所有模式下具有相同定义的依赖模式宏。*/

#define CU2BYTES(x)     ((x)*((PCRE2_CODE_UNIT_WIDTH/8)))
#define BYTES2CU(x)     ((x)/((PCRE2_CODE_UNIT_WIDTH/8)))
#define PUTINC(a,n,d)   PUT(a,n,d), a += LINK_SIZE
#define PUT2INC(a,n,d)  PUT2(a,n,d), a += IMM2_SIZE

/* ----------------------- 隐藏结构 ----------------------------- */

/* 注意：所有这些结构 *必须* 以 pcre2_memctl 结构开始。使用它们的代码更简单，因为它假设了这一点。*/

/* 真正的通用上下文结构。目前它只包含自定义内存控制的数据。*/
typedef struct pcre2_real_general_context {
  pcre2_memctl memctl;
} pcre2_real_general_context;

/* 真正的编译上下文结构 */
/* 定义了用于编译正则表达式的上下文结构体 */
typedef struct pcre2_real_compile_context {
  pcre2_memctl memctl;  // 内存控制
  int (*stack_guard)(uint32_t, void *);  // 栈保护函数指针
  void *stack_guard_data;  // 栈保护数据
  const uint8_t *tables;  // 表格
  PCRE2_SIZE max_pattern_length;  // 最大模式长度
  uint16_t bsr_convention;  // 反斜杠转义约定
  uint16_t newline_convention;  // 换行符约定
  uint32_t parens_nest_limit;  // 括号嵌套限制
  uint32_t extra_options;  // 额外选项
} pcre2_real_compile_context;

/* 真正的匹配上下文结构体 */
typedef struct pcre2_real_match_context {
  pcre2_memctl memctl;  // 内存控制
#ifdef SUPPORT_JIT
  pcre2_jit_callback jit_callback;  // JIT 回调函数
  void *jit_callback_data;  // JIT 回调数据
#endif
  int    (*callout)(pcre2_callout_block *, void *);  // 回调函数指针
  void    *callout_data;  // 回调数据
  int    (*substitute_callout)(pcre2_substitute_callout_block *, void *);  // 替换回调函数指针
  void    *substitute_callout_data;  // 替换回调数据
  PCRE2_SIZE offset_limit;  // 偏移限制
  uint32_t heap_limit;  // 堆限制
  uint32_t match_limit;  // 匹配限制
  uint32_t depth_limit;  // 深度限制
} pcre2_real_match_context;

/* 真正的转换上下文结构体 */
typedef struct pcre2_real_convert_context {
  pcre2_memctl memctl;  // 内存控制
  uint32_t glob_separator;  // 全局分隔符
  uint32_t glob_escape;  // 全局转义符
} pcre2_real_convert_context;

/* 真正的编译代码结构体。blocksize字段的类型被特别定义，因为在pcre2_serialize_decode()中需要将可能不对齐的内存中的大小复制到相同类型的变量中。使用宏而不是typedef，以避免当pcre2test多次包含此文件时出现编译器警告。LOOKBEHIND_MAX指定支持的最大回顾。 （模式中的OP_REVERSE在8位和16位模式中具有16位参数，因此我们在这里不需要超过16位字段。） */
#undef  CODE_BLOCKSIZE_TYPE
#define CODE_BLOCKSIZE_TYPE size_t

#undef  LOOKBEHIND_MAX
#define LOOKBEHIND_MAX UINT16_MAX
/* 定义了一个名为 pcre2_real_code 的结构体，用于存储 PCRE2 正则表达式编译后的真实代码信息 */
typedef struct pcre2_real_code {
  pcre2_memctl memctl;            /* 内存控制字段 */
  const uint8_t *tables;          /* 字符表 */
  void    *executable_jit;        /* 指向 JIT 代码的指针 */
  uint8_t  start_bitmap[32];      /* 用于存储起始代码单元 < 256 的位图 */
  CODE_BLOCKSIZE_TYPE blocksize;  /* 分配的总字节数 */
  uint32_t magic_number;          /* 验证和字节顺序检查的魔术数字 */
  uint32_t compile_options;       /* 传递给 pcre2_compile() 的选项 */
  uint32_t overall_options;       /* 处理模式后的选项 */
  uint32_t extra_options;         /* 从编译上下文中获取的额外选项 */
  uint32_t flags;                 /* 各种状态标志 */
  uint32_t limit_heap;            /* 模式中设置的堆限制 */
  uint32_t limit_match;           /* 模式中设置的匹配限制 */
  uint32_t limit_depth;           /* 模式中设置的深度限制 */
  uint32_t first_codeunit;        /* 起始代码单元 */
  uint32_t last_codeunit;         /* 必须看到的代码单元 */
  uint16_t bsr_convention;        /* \R 匹配的约定 */
  uint16_t newline_convention;    /* 什么是换行符？ */
  uint16_t max_lookbehind;        /* 最长的回顾（字符数） */
  uint16_t minlength;             /* 匹配的最小长度 */
  uint16_t top_bracket;           /* 最高编号的组 */
  uint16_t top_backref;           /* 最高编号的反向引用 */
  uint16_t name_entry_size;       /* 表条目的大小（代码单元） */
  uint16_t name_count;            /* 表中的名称条目数 */
} pcre2_real_code;

/* 真实匹配数据结构。将 ovector 定义为可能的最大尺寸，以便数组边界检查器不会抱怨。
   通过调用 pcre2_match_data_create() 来获取此结构的内存，该函数将尺寸设置为 ovector 的偏移量加上每个可捕获字符串的一对元素，
   因此尺寸因调用而异。由于捕获的最大数量
*/
/* subpatterns is 65535 we must allow for 65536 strings to include the overall
match. (See also the heapframe structure below.) */

/* 定义 pcre2_real_match_data 结构体，用于存储匹配数据 */
struct heapframe;  /* Forward reference */

/* 定义 pcre2_real_match_data 结构体 */
typedef struct pcre2_real_match_data {
  pcre2_memctl     memctl;           /* Memory control fields */
  const pcre2_real_code *code;       /* The pattern used for the match */
  PCRE2_SPTR       subject;          /* The subject that was matched */
  PCRE2_SPTR       mark;             /* Pointer to last mark */
  struct heapframe *heapframes;      /* Backtracking frames heap memory */
  PCRE2_SIZE       heapframes_size;  /* Malloc-ed size */
  PCRE2_SIZE       leftchar;         /* Offset to leftmost code unit */
  PCRE2_SIZE       rightchar;        /* Offset to rightmost code unit */
  PCRE2_SIZE       startchar;        /* Offset to starting code unit */
  uint8_t          matchedby;        /* Type of match (normal, JIT, DFA) */
  uint8_t          flags;            /* Various flags */
  uint16_t         oveccount;        /* Number of pairs */
  int              rc;               /* The return code from the match */
  PCRE2_SIZE       ovector[131072];  /* Must be last in the structure */
} pcre2_real_match_data;

/* ----------------------- PRIVATE STRUCTURES ----------------------------- */

/* These structures are not needed for pcre2test. */

#ifndef PCRE2_PCRE2TEST

/* Structures for checking for mutual recursion when scanning compiled or
parsed code. */

/* 用于检查扫描编译或解析代码时的相互递归的结构体 */
typedef struct recurse_check {
  struct recurse_check *prev;
  PCRE2_SPTR group;
} recurse_check;

/* 用于检查扫描编译或解析代码时的相互递归的结构体 */
typedef struct parsed_recurse_check {
  struct parsed_recurse_check *prev;
  uint32_t *groupptr;
} parsed_recurse_check;

/* Structure for building a cache when filling in recursion offsets. */

/* 用于在填充递归偏移时构建缓存的结构体 */
typedef struct recurse_cache {
  PCRE2_SPTR group;
  int groupnumber;
} recurse_cache;

/* Structure for maintaining a chain of pointers to the currently incomplete
branches, for testing for left recursion while compiling. */

/* 用于维护指向当前不完整分支的指针链的结构体，用于在编译时测试左递归 */
/* 定义一个结构体，用于表示分支链，包含外部链和当前分支的指针 */
typedef struct branch_chain {
  struct branch_chain *outer;  /* 外部链指针 */
  PCRE2_UCHAR *current_branch;  /* 当前分支指针 */
} branch_chain;

/* 在编译的第一阶段构建命名组列表的结构体 */
typedef struct named_group {
  PCRE2_SPTR   name;          /* 指向模式中名称的指针 */
  uint32_t     number;        /* 组号 */
  uint16_t     length;        /* 名称长度 */
  uint16_t     isdup;         /* 如果是重复的则为TRUE */
} named_group;

/* 用于在编译过程中传递“静态”信息的结构体，以确保线程安全 */
typedef struct compile_block {
  /* 结构体内容 */
} compile_block;

/* 用于保存 JIT 匹配器中内存栈属性的结构体 */
typedef struct pcre2_real_jit_stack {
  pcre2_memctl memctl;  /* 内存控制 */
  void* stack;  /* 栈指针 */
} pcre2_real_jit_stack;

/* 用于表示在运行 pcre2_dfa_match() 时模式中显式递归调用的链表中的项目的结构体 */
typedef struct dfa_recursion_info {
  struct dfa_recursion_info *prevrec;  /* 前一个递归信息的指针 */
  PCRE2_SPTR subject_position;  /* 主题位置指针 */
  uint32_t group_num;  /* 组号 */
} dfa_recursion_info;

/* 用于在匹配过程中记住回溯位置的“栈”帧的结构体 */
typedef struct stack_frame {
  /* 结构体内容 */
} stack_frame;  /* 该结构体用于在向量中使用，因此大小必须是 PCRE2_SIZE 的倍数 */
typedef struct heapframe {

  /* The first set of fields are variables that have to be preserved over calls
  to RRMATCH(), but which do not need to be copied to new frames. */

  // 当前模式中的位置
  PCRE2_SPTR ecode;
  // 用于短期 PCRE_SPTR 值的临时变量
  PCRE2_SPTR temp_sptr[2];
  // 用于字符、字符串或代码长度的临时变量
  PCRE2_SIZE length;
  // 在 RRETURN 上需要减去的量
  PCRE2_SIZE back_frame;
  // 用于短期 PCRE2_SIZE 值的临时变量
  PCRE2_SIZE temp_size;
  // "递归" 深度
  uint32_t rdepth;
  // 组帧的类型信息
  uint32_t group_frame_type;
  // 用于短期 32 位或 BOOL 值的临时变量
  uint32_t temp_32[4];
  // 内部 "返回" 的跳转位置
  uint8_t return_id;
  // 处理操作码
  uint8_t op;

  /* At this point, the structure is 16-bit aligned. On most architectures
  the alignment requirement for a pointer will ensure that the eptr field below
  is 32-bit or 64-bit aligned. However, on m68k it is fine to have a pointer
  that is 16-bit aligned. We must therefore ensure that what comes between here
  and eptr is an odd multiple of 16 bits so as to get back into 32-bit
  alignment. This happens naturally when PCRE2_UCHAR is 8 bits wide, but needs
  fudges in the other cases. In the 32-bit case the padding comes first so that
  the occu field itself is 32-bit aligned. Without the padding, this structure
  is no longer a multiple of PCRE2_SIZE on m68k, and the check below fails. */

  // 根据 PCRE2_CODE_UNIT_WIDTH 的不同，使用不同的变量类型和对齐方式
#if PCRE2_CODE_UNIT_WIDTH == 8
  // 用于其他情况下的代码单元
  PCRE2_UCHAR occu[6];
#elif PCRE2_CODE_UNIT_WIDTH == 16
  // 用于其他情况下的代码单元
  PCRE2_UCHAR occu[2];
  // 确保 32 位对齐（见上文）
  uint8_t unused[2];
#else
  // 确保 32 位对齐（见上文）
  uint8_t unused[2];
  // 用于其他情况下的代码单元
  PCRE2_UCHAR occu[1];
#endif

  /* The rest have to be copied from the previous frame whenever a new frame
  becomes current. The final field is specified as a large vector so that
  runtime array bound checks don't catch references to it. However, for any
  specific call to pcre2_match() the memory allocated for each frame structure
  allows for exactly the right size ovector for the number of capturing
  parentheses. (See also the comment for pcre2_real_match_data above.) */

  PCRE2_SPTR eptr;           /* MUST BE FIRST */  // 定义指向 PCRE2_SPTR 类型的指针 eptr
  PCRE2_SPTR start_match;    /* Can be adjusted by \K */  // 定义指向 PCRE2_SPTR 类型的指针 start_match
  PCRE2_SPTR mark;           /* Most recent mark on the success path */  // 定义指向 PCRE2_SPTR 类型的指针 mark
  uint32_t current_recurse;  /* Current (deepest) recursion number */  // 定义无符号整数类型变量 current_recurse
  uint32_t capture_last;     /* Most recent capture */  // 定义无符号整数类型变量 capture_last
  PCRE2_SIZE last_group_offset;  /* Saved offset to most recent group frame */  // 定义 PCRE2_SIZE 类型变量 last_group_offset
  PCRE2_SIZE offset_top;     /* Offset after highest capture */  // 定义 PCRE2_SIZE 类型变量 offset_top
  PCRE2_SIZE ovector[131072]; /* Must be last in the structure */  // 定义大小为 131072 的 PCRE2_SIZE 类型数组 ovector，作为结构的最后一个字段
} heapframe;

/* This typedef is a check that the size of the heapframe structure is a
multiple of PCRE2_SIZE. See various comments above. */

typedef char check_heapframe_size[
  ((sizeof(heapframe) % sizeof(PCRE2_SIZE)) == 0)? (+1):(-1)];  // 检查 heapframe 结构的大小是否是 PCRE2_SIZE 的倍数

/* Structure for computing the alignment of heapframe. */

typedef struct heapframe_align {
  char unalign;    /* Completely unalign the current offset */  // 定义字符类型变量 unalign
  heapframe frame; /* Offset is its alignment */  // 定义 heapframe 类型变量 frame
} heapframe_align;

/* This define is the minimum alignment required for a heapframe, in bytes. */

#define HEAPFRAME_ALIGNMENT offsetof(heapframe_align, frame)  // 定义 HEAPFRAME_ALIGNMENT 宏，表示 heapframe 的最小对齐要求，以字节为单位

/* Structure for passing "static" information around between the functions
doing traditional NFA matching (pcre2_match() and friends). */

} match_block;

/* A similar structure is used for the same purpose by the DFA matching
functions. */
typedef struct dfa_match_block {
  pcre2_memctl memctl;            /* 用于一般用途的内存控制 */
  PCRE2_SPTR start_code;          /* 编译模式的起始位置 */
  PCRE2_SPTR start_subject ;      /* 主题字符串的起始位置 */
  PCRE2_SPTR end_subject;         /* 主题字符串的结束位置 */
  PCRE2_SPTR start_used_ptr;      /* 最早被查询的字符 */
  PCRE2_SPTR last_used_ptr;       /* 最近被查询的字符 */
  const uint8_t *tables;          /* 字符表 */
  PCRE2_SIZE start_offset;        /* 起始偏移值 */
  PCRE2_SIZE heap_limit;          /* 堆限制 */
  PCRE2_SIZE heap_used;           /* 堆使用情况 */
  uint32_t match_limit;           /* 匹配限制 */
  uint32_t match_limit_depth;     /* 匹配限制深度 */
  uint32_t match_call_count;      /* 内部函数调用次数 */
  uint32_t moptions;              /* 匹配选项 */
  uint32_t poptions;              /* 模式选项 */
  uint32_t nltype;                /* 换行符类型 */
  uint32_t nllen;                 /* 换行符字符串长度 */
  BOOL allowemptypartial;         /* 允许空的部分匹配 */
  PCRE2_UCHAR nl[4];              /* 固定换行符字符串 */
  uint16_t bsr_convention;        /* \R 解释 */
  pcre2_callout_block *cb;        /* 指向调用块的指针 */
  void *callout_data;             /* 传递给调用的数据 */
  int (*callout)(pcre2_callout_block *,void *);  /* 调用函数或空 */
  dfa_recursion_info *recursive;  /* 递归数据的链表 */
} dfa_match_block;

#endif  /* PCRE2_PCRE2TEST */

/* pcre2_intmodedep.h 的结尾 */
```