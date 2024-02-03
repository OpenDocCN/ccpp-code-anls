# `nmap\libpcre\src\pcre2_internal.h`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE2 is a library of functions to support regular expressions whose syntax
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
#ifndef PCRE2_INTERNAL_H_IDEMPOTENT_GUARD
#define PCRE2_INTERNAL_H_IDEMPOTENT_GUARD

/* 如果 PCRE2_INTERNAL_H_IDEMPOTENT_GUARD 未定义，则定义它，以避免重复包含 */

/* 我们不支持同时使用 EBCDIC 和 Unicode。"configure" 脚本会阻止同时选择两者，但并非所有人都使用 "configure"。EBCDIC 仅支持 8 位库，但检查这一点必须在文件的后部进行，因为前部不依赖于宽度，并且在 CODE_UNIT_WIDTH == 0 时由 pcre2test.c 包含。 */

#if defined EBCDIC && defined SUPPORT_UNICODE
#error 不支持同时使用 EBCDIC 和 SUPPORT_UNICODE。
#endif

/* 标准 C 头文件 */

#include <ctype.h>
#include <limits.h>
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 宏使布尔值更明显。#ifndef 用于平息在这些宏在其他地方已定义的环境中的编译器警告。不幸的是，对于 typedef，没有办法做到同样。 */

typedef int BOOL;
#ifndef FALSE
#define FALSE   0
#define TRUE    1
#endif

/* Valgrind（memcheck）支持 */

#ifdef SUPPORT_VALGRIND
#include <valgrind/memcheck.h>
#endif

/* -ftrivial-auto-var-init 支持初始化所有局部变量，以避免某些类别的错误，但这可能会导致在热函数中对大型栈数组的不可接受的减速。此宏允许我们注释这样的数组。 */

#ifdef HAVE_ATTRIBUTE_UNINITIALIZED
#define PCRE2_KEEP_UNINITIALIZED __attribute__((uninitialized))
#else
#define PCRE2_KEEP_UNINITIALIZED
#endif

/* 旧版本的 MSVC 缺乏 snprintf()。此定义允许在至少 MSVC 10/2010 之前的 MSVC 编译器中进行无警告/错误的编译和测试。除了 VC6（它缺少一些基本功能并失败）。 */

#if defined(_MSC_VER) && (_MSC_VER < 1900)
#define snprintf _snprintf
#endif
/* 当在 Windows 上编译 DLL 时，导出的符号必须使用一些 MS 魔法声明。
   我在这个网页上找到了一些有用的信息：http://msdn2.microsoft.com/en-us/library/y4h7bcy6(VS.80).aspx。
   根据那里的信息，使用 __declspec(dllexport) 而不带 "extern" 就是定义；带 "extern" 就是声明。
   这里的设置覆盖了 pcre2.h 中的设置（该文件在下面包含）；它只定义了 PCRE2_EXP_DECL，这对于应用程序来说就足够了（它们只是导入符号）。
   我们使用：
     PCRE2_EXP_DECL    用于声明
     PCRE2_EXP_DEFN    用于定义

   将此包装在 #ifndef PCRE2_EXP_DECL 中是为了让 pcre2test（一个应用程序，但需要导入此文件以便 "窥视" 内部），
   可以先 #include pcre2.h 以获得应用程序视角。

   原则上，为了编译非 Windows、非类 Unix（即不常见的、特殊用途的环境）的人可能希望在导出的符号前面添加其他内容。
   这就是为什么在非 Windows 情况下，只有在 PCRE2_EXP_DEFN 没有设置时，我们才设置 PCRE2_EXP_DEFN。 */

#ifndef PCRE2_EXP_DECL
#  ifdef _WIN32
#    ifndef PCRE2_STATIC
#      define PCRE2_EXP_DECL       extern __declspec(dllexport)
#      define PCRE2_EXP_DEFN       __declspec(dllexport)
#    else
#      define PCRE2_EXP_DECL       extern
#      define PCRE2_EXP_DEFN
#    endif
#  else
#    ifdef __cplusplus
#      define PCRE2_EXP_DECL       extern "C"
#    else
#      define PCRE2_EXP_DECL       extern
#    endif
#    ifndef PCRE2_EXP_DEFN
#      define PCRE2_EXP_DEFN       PCRE2_EXP_DECL
#    endif
#  endif
#endif

/* 包含公共 PCRE2 头文件和 UCP 字符属性值的定义。这必须在上面设置的 PCRE2_EXP_DECL 之后。 */

#include "pcre2.h"
#include "pcre2_ucp.h"

/* 当将 PCRE2 编译为 C++ 库时，主题指针可以被替换为自定义类型。
   例如，这使得可能允许 pcre2_match() 使用自定义类型的主题指针。 */
/* 
处理不连续的主题字符串，使用智能指针类。由于它的回溯方式，必须始终能够检查 pcre2_match() 中的所有主题字符串。
警告：这对 PCRE2 尚未经过测试。
*/
#ifdef CUSTOM_SUBJECT_PTR
#undef PCRE2_SPTR
#define PCRE2_SPTR CUSTOM_SUBJECT_PTR
#endif

/* 在 pcre2_compile() 中检查整数溢出时，我们需要处理大整数。如果有 64 位整数类型可用，我们可以使用它。
否则，我们必须转换为 double，这当然需要浮点运算。通过定义适当类型的宏来处理这个问题。 */
#if defined INT64_MAX || defined int64_t
#define INT64_OR_DOUBLE int64_t
#else
#define INT64_OR_DOUBLE double
#endif

/* 外部（在 C 语言中）函数和表对于库来说是私有的，总是使用 PRIV 宏来引用它们。
这使得 pcre2test.c 可以使用不同的 PRIV 定义来包含一些来自库的源文件，以避免名称冲突。它还清楚地表明了代码中正在引用一个非静态对象。 */
#ifndef PRIV
#define PRIV(name) _pcre2_##name
#endif

/* 当编译用于 Virtual Pascal 编译器时，这些函数的名称需要更改。PCRE2 必须在命令行上使用 -DVPCOMPAT 选项进行编译。 */
#ifdef VPCOMPAT
#define strlen(s)        _strlen(s)
#define strncmp(s1,s2,m) _strncmp(s1,s2,m)
#define memcmp(s,c,n)    _memcmp(s,c,n)
#define memcpy(d,s,n)    _memcpy(d,s,n)
#define memmove(d,s,n)   _memmove(d,s,n)
#define memset(s,c,n)    _memset(s,c,n)
#else  /* VPCOMPAT */

/* 否则，为了应对 SunOS4 和其他缺乏 memmove() 的系统，定义一个调用模拟函数的宏。 */
#ifndef HAVE_MEMMOVE
#undef  memmove          /* 一些系统可能有一个宏 */
#define memmove(a, b, c) PRIV(memmove)(a, b, c)
#endif   /* not HAVE_MEMMOVE */
#endif   /* not VPCOMPAT */
/* This is an unsigned int value that no UTF character can ever have, as
Unicode doesn't go beyond 0x0010ffff. */
定义一个无符号整数值，超出 Unicode 编码范围的值，用于表示不是有效的 UTF 字符。

#define NOTACHAR 0xffffffff
定义名为 NOTACHAR 的宏，赋值为 0xffffffff。

/* This is the largest valid UTF/Unicode code point. */
定义一个有效的 UTF/Unicode 编码点的最大值。

#define MAX_UTF_CODE_POINT 0x10ffff
定义名为 MAX_UTF_CODE_POINT 的宏，赋值为 0x10ffff。

/* Compile-time positive error numbers (all except UTF errors, which are
negative) start at this value. It should probably never be changed, in case
some application is checking for specific numbers. There is a copy of this
#define in pcre2posix.c (which now no longer includes this file). Ideally, a
way of having a single definition should be found, but as the number is
unlikely to change, this is not a pressing issue. The original reason for
having a base other than 0 was to keep the absolute values of compile-time and
run-time error numbers numerically different, but in the event the code does
not rely on this. */
定义编译时的正数错误号（除了 UTF 错误，它们是负数），从这个值开始。这个值可能永远不会改变，以防某些应用程序检查特定的错误号。在 pcre2posix.c 中有一个副本（现在不再包括这个文件）。理想情况下，应该找到一个单一的定义方式，但由于这个数字不太可能改变，这不是一个紧迫的问题。最初选择非 0 的基数是为了保持编译时和运行时错误号的绝对值在数值上的差异，但实际上代码并不依赖于这一点。

#define COMPILE_ERROR_BASE 100
定义名为 COMPILE_ERROR_BASE 的宏，赋值为 100。

/* The initial frames vector for remembering pcre2_match() backtracking points
is allocated on the heap, of this size (bytes) or ten times the frame size if
larger, unless the heap limit is smaller. Typical frame sizes are a few hundred
bytes (it depends on the number of capturing parentheses) so 20KiB handles
quite a few frames. A larger vector on the heap is obtained for matches that
need more frames, subject to the heap limit. */
用于记住 pcre2_match() 回溯点的初始帧向量分配在堆上，大小为这个值（字节），或者如果更大，则为帧大小的十倍，除非堆限制更小。典型的帧大小是几百字节（这取决于捕获括号的数量），所以 20KiB 可以处理相当多的帧。对于需要更多帧的匹配，可以获得堆上的更大向量，但受堆限制。

#define START_FRAMES_SIZE 20480
定义名为 START_FRAMES_SIZE 的宏，赋值为 20480。

/* For DFA matching, an initial internal workspace vector is allocated on the
stack. The heap is used only if this turns out to be too small. */
对于 DFA 匹配，初始的内部工作空间向量分配在堆栈上。只有在这个空间太小的情况下才使用堆。

#define DFA_START_RWS_SIZE 30720
定义名为 DFA_START_RWS_SIZE 的宏，赋值为 30720。

/* Define the default BSR convention. */
定义默认的 BSR 约定。

#ifdef BSR_ANYCRLF
#define BSR_DEFAULT PCRE2_BSR_ANYCRLF
#else
#define BSR_DEFAULT PCRE2_BSR_UNICODE
#endif
如果定义了 BSR_ANYCRLF，则将 BSR_DEFAULT 宏定义为 PCRE2_BSR_ANYCRLF，否则定义为 PCRE2_BSR_UNICODE。

/* ---------------- Basic UTF-8 macros ---------------- */

/* These UTF-8 macros are always defined because they are used in pcre2test for
handling wide characters in 16-bit and 32-bit modes, even if an 8-bit library
is not supported. */
这些 UTF-8 宏总是被定义，因为它们在 pcre2test 中用于处理 16 位和 32 位模式下的宽字符，即使不支持 8 位库。
/* Tests whether a UTF-8 code point needs extra bytes to decode. */
/* 检查一个 UTF-8 编码点是否需要额外的字节来解码 */

#define HASUTF8EXTRALEN(c) ((c) >= 0xc0)
/* 定义宏，用于检查 UTF-8 编码点是否需要额外的字节来解码 */

/* The following macros were originally written in the form of loops that used
data from the tables whose names start with PRIV(utf8_table). They were
rewritten by a user so as not to use loops, because in some environments this
gives a significant performance advantage, and it seems never to do any harm.
*/
/* 以下的宏最初是以使用以 PRIV(utf8_table) 开头的表中的数据的循环形式编写的。它们被用户重写，以便不使用循环，因为在某些环境中，这样做会带来显著的性能优势，并且似乎从来没有任何害处。 */

/* Base macro to pick up the remaining bytes of a UTF-8 character, not
advancing the pointer. */
/* 基本宏，用于获取 UTF-8 字符的剩余字节，不推进指针。 */

#define GETUTF8(c, eptr) \
    { \
    if ((c & 0x20u) == 0) \
      c = ((c & 0x1fu) << 6) | (eptr[1] & 0x3fu); \
    else if ((c & 0x10u) == 0) \
      c = ((c & 0x0fu) << 12) | ((eptr[1] & 0x3fu) << 6) | (eptr[2] & 0x3fu); \
    else if ((c & 0x08u) == 0) \
      c = ((c & 0x07u) << 18) | ((eptr[1] & 0x3fu) << 12) | \
      ((eptr[2] & 0x3fu) << 6) | (eptr[3] & 0x3fu); \
    else if ((c & 0x04u) == 0) \
      c = ((c & 0x03u) << 24) | ((eptr[1] & 0x3fu) << 18) | \
          ((eptr[2] & 0x3fu) << 12) | ((eptr[3] & 0x3fu) << 6) | \
          (eptr[4] & 0x3fu); \
    else \
      c = ((c & 0x01u) << 30) | ((eptr[1] & 0x3fu) << 24) | \
          ((eptr[2] & 0x3fu) << 18) | ((eptr[3] & 0x3fu) << 12) | \
          ((eptr[4] & 0x3fu) << 6) | (eptr[5] & 0x3fu); \
    }
/* 定义宏，用于获取 UTF-8 字符的剩余字节，不推进指针 */

/* Base macro to pick up the remaining bytes of a UTF-8 character, advancing
the pointer. */
/* 基本宏，用于获取 UTF-8 字符的剩余字节，推进指针。 */

#define GETUTF8INC(c, eptr) \
    { \
    if ((c & 0x20u) == 0) \
      c = ((c & 0x1fu) << 6) | (*eptr++ & 0x3fu); \
    else if ((c & 0x10u) == 0) \
      { \
      c = ((c & 0x0fu) << 12) | ((*eptr & 0x3fu) << 6) | (eptr[1] & 0x3fu); \
      eptr += 2; \
      } \
    else if ((c & 0x08u) == 0) \
      { \
      c = ((c & 0x07u) << 18) | ((*eptr & 0x3fu) << 12) | \
          ((eptr[1] & 0x3fu) << 6) | (eptr[2] & 0x3fu); \
      eptr += 3; \
      } \
    }
/* 定义宏，用于获取 UTF-8 字符的剩余字节，推进指针 */
    else if ((c & 0x04u) == 0) \  # 如果 c 的第三位为 0
      { \  # 进入条件判断语句块
      c = ((c & 0x03u) << 24) | ((*eptr & 0x3fu) << 18) | \  # 对 c 进行位运算
          ((eptr[1] & 0x3fu) << 12) | ((eptr[2] & 0x3fu) << 6) | \  # 对 eptr 数组进行位运算
          (eptr[3] & 0x3fu); \  # 对 eptr 数组进行位运算
      eptr += 4; \  # eptr 指针向后移动 4 位
      } \  # 结束条件判断语句块
    else \  # 否则
      { \  # 进入条件判断语句块
      c = ((c & 0x01u) << 30) | ((*eptr & 0x3fu) << 24) | \  # 对 c 进行位运算
          ((eptr[1] & 0x3fu) << 18) | ((eptr[2] & 0x3fu) << 12) | \  # 对 eptr 数组进行位运算
          ((eptr[3] & 0x3fu) << 6) | (eptr[4] & 0x3fu); \  # 对 eptr 数组进行位运算
      eptr += 5; \  # eptr 指针向后移动 5 位
      } \  # 结束条件判断语句块
    }
/* 宏定义：获取 UTF-8 字符的剩余字节，不移动指针，增加长度 */

#define GETUTF8LEN(c, eptr, len) \
    { \
    if ((c & 0x20u) == 0) \  // 如果第一个字节的第三位为0
      { \
      c = ((c & 0x1fu) << 6) | (eptr[1] & 0x3fu); \  // 将第一个字节的后五位左移6位，然后与第二个字节的后六位进行或运算
      len++; \  // 长度加1
      } \
    else if ((c & 0x10u)  == 0) \  // 如果第一个字节的第四位为0
      { \
      c = ((c & 0x0fu) << 12) | ((eptr[1] & 0x3fu) << 6) | (eptr[2] & 0x3fu); \  // 将第一个字节的后四位左移12位，第二个字节的后六位左移6位，与第三个字节的后六位进行或运算
      len += 2; \  // 长度加2
      } \
    else if ((c & 0x08u)  == 0) \  // 如果第一个字节的第五位为0
      {\
      c = ((c & 0x07u) << 18) | ((eptr[1] & 0x3fu) << 12) | \
          ((eptr[2] & 0x3fu) << 6) | (eptr[3] & 0x3fu); \  // 将第一个字节的后三位左移18位，第二个字节的后六位左移12位，第三个字节的后六位左移6位，与第四个字节的后六位进行或运算
      len += 3; \  // 长度加3
      } \
    else if ((c & 0x04u)  == 0) \  // 如果第一个字节的第六位为0
      { \
      c = ((c & 0x03u) << 24) | ((eptr[1] & 0x3fu) << 18) | \
          ((eptr[2] & 0x3fu) << 12) | ((eptr[3] & 0x3fu) << 6) | \
          (eptr[4] & 0x3fu); \  // 将第一个字节的后两位左移24位，第二个字节的后六位左移18位，第三个字节的后六位左移12位，第四个字节的后六位左移6位，与第五个字节的后六位进行或运算
      len += 4; \  // 长度加4
      } \
    else \  // 如果第一个字节的第七位为0
      {\
      c = ((c & 0x01u) << 30) | ((eptr[1] & 0x3fu) << 24) | \
          ((eptr[2] & 0x3fu) << 18) | ((eptr[3] & 0x3fu) << 12) | \
          ((eptr[4] & 0x3fu) << 6) | (eptr[5] & 0x3fu); \  // 将第一个字节的最后一位左移30位，第二个字节的后六位左移24位，第三个字节的后六位左移18位，第四个字节的后六位左移12位，第五个字节的后六位左移6位，与第六个字节的后六位进行或运算
      len += 5; \  // 长度加5
      } \
    }

/* --------------- 空白字符宏 ---------------- */

/* Unicode 水平和垂直空白字符的测试必须检查多个不同的值。使用 switch 语句可以生成最快的代码（无循环，无内存访问），并且解释器代码中有几个地方会这样做。为了确保所有 case 列表保持同步，我们使用宏，这样只有一个地方定义列表。

这些值在 pcre2_compile.c 处理字符类中的 \h、\H、\v 和 \V 时也作为列表。列表在 pcre2_tables.c 中定义，但宏定义的值在这里，以便所有定义都在一起。列表必须按升序字符顺序排列，并以 NOTACHAR（0xffffffff）终止。

任何更改都应确保各种宏与每个
/* -------------- ASCII/Unicode environments -------------- */

#ifndef EBCDIC

/* Character U+180E (Mongolian Vowel Separator) is not included in the list of
spaces in the Unicode file PropList.txt, and Perl does not recognize it as a
space. However, in many other sources it is listed as a space and has been in
PCRE (both APIs) for a long time. */

#define HSPACE_LIST \
  CHAR_HT, CHAR_SPACE, CHAR_NBSP, \
  0x1680, 0x180e, 0x2000, 0x2001, 0x2002, 0x2003, 0x2004, 0x2005, \
  0x2006, 0x2007, 0x2008, 0x2009, 0x200A, 0x202f, 0x205f, 0x3000, \
  NOTACHAR

#define HSPACE_MULTIBYTE_CASES \
  case 0x1680:  /* OGHAM SPACE MARK */ \
  case 0x180e:  /* MONGOLIAN VOWEL SEPARATOR */ \
  case 0x2000:  /* EN QUAD */ \
  case 0x2001:  /* EM QUAD */ \
  case 0x2002:  /* EN SPACE */ \
  case 0x2003:  /* EM SPACE */ \
  case 0x2004:  /* THREE-PER-EM SPACE */ \
  case 0x2005:  /* FOUR-PER-EM SPACE */ \
  case 0x2006:  /* SIX-PER-EM SPACE */ \
  case 0x2007:  /* FIGURE SPACE */ \
  case 0x2008:  /* PUNCTUATION SPACE */ \
  case 0x2009:  /* THIN SPACE */ \
  case 0x200A:  /* HAIR SPACE */ \
  case 0x202f:  /* NARROW NO-BREAK SPACE */ \
  case 0x205f:  /* MEDIUM MATHEMATICAL SPACE */ \
  case 0x3000   /* IDEOGRAPHIC SPACE */

#define HSPACE_BYTE_CASES \
  case CHAR_HT: \
  case CHAR_SPACE: \
  case CHAR_NBSP

#define HSPACE_CASES \
  HSPACE_BYTE_CASES: \
  HSPACE_MULTIBYTE_CASES

#define VSPACE_LIST \
  CHAR_LF, CHAR_VT, CHAR_FF, CHAR_CR, CHAR_NEL, 0x2028, 0x2029, NOTACHAR

#define VSPACE_MULTIBYTE_CASES \
  case 0x2028:    /* LINE SEPARATOR */ \
  case 0x2029     /* PARAGRAPH SEPARATOR */

#define VSPACE_BYTE_CASES \
  case CHAR_LF: \
  case CHAR_VT: \
  case CHAR_FF: \
  case CHAR_CR: \
  case CHAR_NEL

#define VSPACE_CASES \
  VSPACE_BYTE_CASES: \
  VSPACE_MULTIBYTE_CASES

/* -------------- EBCDIC environments -------------- */

#else
#define HSPACE_LIST CHAR_HT, CHAR_SPACE, CHAR_NBSP, NOTACHAR
#define HSPACE_BYTE_CASES \  # 定义水平空白字节的情况
  case CHAR_HT: \  # 水平制表符
  case CHAR_SPACE: \  # 空格
  case CHAR_NBSP  # 不换行空格

#define HSPACE_CASES HSPACE_BYTE_CASES  # 定义水平空白的情况

#ifdef EBCDIC_NL25  # 如果定义了 EBCDIC_NL25
#define VSPACE_LIST \  # 定义垂直空白列表
  CHAR_VT, CHAR_FF, CHAR_CR, CHAR_NEL, CHAR_LF, NOTACHAR  # 垂直制表符、换页符、回车符、下一行、换行符、非字符
#else  # 否则
#define VSPACE_LIST \  # 定义垂直空白列表
  CHAR_VT, CHAR_FF, CHAR_CR, CHAR_LF, CHAR_NEL, NOTACHAR  # 垂直制表符、换页符、回车符、换行符、下一行、非字符
#endif  # 结束条件编译

#define VSPACE_BYTE_CASES \  # 定义垂直空白字节的情况
  case CHAR_LF: \  # 换行符
  case CHAR_VT: \  # 垂直制表符
  case CHAR_FF: \  # 换页符
  case CHAR_CR: \  # 回车符
  case CHAR_NEL  # 下一行

#define VSPACE_CASES VSPACE_BYTE_CASES  # 定义垂直空白的情况
#endif  /* EBCDIC */  # 结束条件编译

/* -------------- End of whitespace macros -------------- */
# 结束空白宏的定义

/* PCRE2 is able to support several different kinds of newline (CR, LF, CRLF,
"any" and "anycrlf" at present). The following macros are used to package up
testing for newlines. NLBLOCK, PSSTART, and PSEND are defined in the various
modules to indicate in which datablock the parameters exist, and what the
start/end of string field names are. */

/* PCRE2 能够支持几种不同类型的换行符（CR、LF、CRLF、"any" 和 "anycrlf"）。以下宏用于打包测试换行符。NLBLOCK、PSSTART 和 PSEND 在各个模块中被定义，用于指示参数存在于哪个数据块中，以及字符串字段名称的起始/结束位置。 */

#define NLTYPE_FIXED    0     /* Newline is a fixed length string */  # 固定长度字符串的换行符类型
#define NLTYPE_ANY      1     /* Newline is any Unicode line ending */  # 任意 Unicode 行尾的换行符类型
#define NLTYPE_ANYCRLF  2     /* Newline is CR, LF, or CRLF */  # CR、LF 或 CRLF 的换行符类型

/* This macro checks for a newline at the given position */

/* 此宏检查给定位置是否有换行符 */

#define IS_NEWLINE(p) \  # 检查给定位置是否有换行符
  ((NLBLOCK->nltype != NLTYPE_FIXED)? \  # 如果换行符类型不是固定长度
    ((p) < NLBLOCK->PSEND && \  # 位置小于结束位置
     PRIV(is_newline)((p), NLBLOCK->nltype, NLBLOCK->PSEND, \  # 调用 is_newline 函数检查换行符
       &(NLBLOCK->nllen), utf)) \  # 传入参数并获取换行符长度
    : \  # 否则
    ((p) <= NLBLOCK->PSEND - NLBLOCK->nllen && \  # 位置小于等于结束位置减去换行符长度
     UCHAR21TEST(p) == NLBLOCK->nl[0] && \  # 检查第一个字符是否是换行符的第一个字符
     (NLBLOCK->nllen == 1 || UCHAR21TEST(p+1) == NLBLOCK->nl[1])       \  # 如果换行符长度为1或者第二个字符也是换行符的第二个字符
    ) \  # 结束条件
  )

/* This macro checks for a newline immediately preceding the given position */

/* 此宏检查给定位置之前是否有换行符 */

#define WAS_NEWLINE(p) \  # 检查给定位置之前是否有换行符
  ((NLBLOCK->nltype != NLTYPE_FIXED)? \  # 如果换行符类型不是固定长度
    ((p) > NLBLOCK->PSSTART && \  # 位置大于起始位置
     PRIV(was_newline)((p), NLBLOCK->nltype, NLBLOCK->PSSTART, \  # 调用 was_newline 函数检查换行符
       &(NLBLOCK->nllen), utf)) \  # 传入参数并获取换行符长度
    : \  # 否则
    # 检查指针 p 是否大于等于 NLBLOCK->PSSTART + NLBLOCK->nllen，并且
    # 检查指针 p - NLBLOCK->nllen 处的字符是否等于 NLBLOCK->nl[0]，以及
    # 如果 NLBLOCK->nllen 等于 1，则检查指针 p - NLBLOCK->nllen + 1 处的字符是否等于 NLBLOCK->nl[1]
  )
/* Private flags containing information about the compiled pattern. The first
three must not be changed, because whichever is set is actually the number of
bytes in a code unit in that mode. */

#define PCRE2_MODE8         0x00000001  /* compiled in 8 bit mode */
#define PCRE2_MODE16        0x00000002  /* compiled in 16 bit mode */
#define PCRE2_MODE32        0x00000004  /* compiled in 32 bit mode */
#define PCRE2_FIRSTSET      0x00000010  /* first_code unit is set */
#define PCRE2_FIRSTCASELESS 0x00000020  /* caseless first code unit */
#define PCRE2_FIRSTMAPSET   0x00000040  /* bitmap of first code units is set */
#define PCRE2_LASTSET       0x00000080  /* last code unit is set */
#define PCRE2_LASTCASELESS  0x00000100  /* caseless last code unit */
#define PCRE2_STARTLINE     0x00000200  /* start after \n for multiline */
#define PCRE2_JCHANGED      0x00000400  /* j option used in pattern */
#define PCRE2_HASCRORLF     0x00000800  /* explicit \r or \n in pattern */
#define PCRE2_HASTHEN       0x00001000  /* pattern contains (*THEN) */
#define PCRE2_MATCH_EMPTY   0x00002000  /* pattern can match empty string */
#define PCRE2_BSR_SET       0x00004000  /* BSR was set in the pattern */
#define PCRE2_NL_SET        0x00008000  /* newline was set in the pattern */
#define PCRE2_NOTEMPTY_SET  0x00010000  /* (*NOTEMPTY) used        ) keep */
#define PCRE2_NE_ATST_SET   0x00020000  /* (*NOTEMPTY_ATSTART) used) together */
#define PCRE2_DEREF_TABLES  0x00040000  /* release character tables */
#define PCRE2_NOJIT         0x00080000  /* (*NOJIT) used */
#define PCRE2_HASBKPORX     0x00100000  /* contains \P, \p, or \X */
#define PCRE2_DUPCAPUSED    0x00200000  /* contains (?| */
#define PCRE2_HASBKC        0x00400000  /* contains \C */
#define PCRE2_HASACCEPT     0x00800000  /* contains (*ACCEPT) */

#define PCRE2_MODE_MASK     (PCRE2_MODE8 | PCRE2_MODE16 | PCRE2_MODE32)

/* Values for the matchedby field in a match data block. */
# 定义枚举类型，表示匹配是由哪种方法进行的
enum { PCRE2_MATCHEDBY_INTERPRETER,     /* pcre2_match() */
       PCRE2_MATCHEDBY_DFA_INTERPRETER, /* pcre2_dfa_match() */
       PCRE2_MATCHEDBY_JIT };           /* pcre2_jit_match() */

# 匹配数据块中标志字段的数值
#define PCRE2_MD_COPIED_SUBJECT  0x01u

# 用于检查传入的数据是否正确的魔术数
#define MAGIC_NUMBER  0x50435245UL   /* 'PCRE' */

# 从锚定模式中搜索匹配的最大剩余主题长度
#if PCRE2_CODE_UNIT_WIDTH == 8
#define REQ_CU_MAX       5000
#else
#define REQ_CU_MAX       2000
#endif

# 在 cbits 表中位图表的偏移量
#define cbit_space     0      /* [:space:] or \s */
#define cbit_xdigit   32      /* [:xdigit:] */
#define cbit_digit    64      /* [:digit:] or \d */
#define cbit_upper    96      /* [:upper:] */
#define cbit_lower   128      /* [:lower:] */
#define cbit_word    160      /* [:word:] or \w */
#define cbit_graph   192      /* [:graph:] */
#define cbit_print   224      /* [:print:] */
#define cbit_punct   256      /* [:punct:] */
#define cbit_cntrl   288      /* [:cntrl:] */
#define cbit_length  320      /* Length of the cbits table */

# ctypes 表中条目的位定义
#define ctype_space    0x01
#define ctype_letter   0x02
#define ctype_lcletter 0x04
#define ctype_digit    0x08
#define ctype_word     0x10    /* alphanumeric or '_' */

# 从基本表指针的各种表的偏移量和表的总长度
#define lcc_offset      0                           /* Lower case */
/* 定义偏移量，用于指示 Flip case 的位置 */
#define fcc_offset    256                           /* Flip case */
/* 定义偏移量，用于指示字符类的位置 */
#define cbits_offset  512                           /* Character classes */
/* 定义偏移量，用于指示字符类型的位置 */
#define ctypes_offset (cbits_offset + cbit_length)  /* Character types */
/* 定义表的长度 */
#define TABLES_LENGTH (ctypes_offset + 256)


/* -------------------- Character and string names ------------------------ */

/* 如果 PCRE2 要在 EBCDIC 平台上支持 UTF-8，我们不能使用普通的字符常量，比如 '*'，因为编译器会输出它们的 EBCDIC 代码，这与它们的 ASCII/UTF-8 代码不同。相反，我们为字符定义宏，这样当启用 UTF-8 支持时，它们总是使用 ASCII/UTF-8 代码。当未启用 UTF-8 支持时，这些定义使用字符文字。每个字符和字符串版本都是必需的，还有一些更长的字符串。

这意味着，在 EBCDIC 平台上，PCRE2 库可以处理 EBCDIC 或 UTF-8，但不能同时处理两者。要在同一编译的库中支持两者，需要根据是否设置了 PCRE2_UTF 来进行不同的查找。这将使在 switch/case 语句中使用字符变得不可能，这将降低性能。对于理论上的使用（没有人要求过）在少数领域（EBCDIC 平台），这是不明智的。任何确实需要两者的应用程序都可以编译两个版本的库，使用宏为函数提供不同的名称。 */

#ifndef SUPPORT_UNICODE

/* 未启用 UTF-8 支持；使用平台相关的字符文字，以便 PCRE2 在 ASCII 和 EBCDIC 环境中都能工作，但只能在非 UTF 模式下工作。换行字符在 EBCDIC 中有问题。虽然它有 CR 和 LF 字符，但在类似 C 的处理环境中，通常的做法是使用其 NL（0x15）字符作为行终止符。然而，根据这个 Unicode 文档，有时也会使用 LF（0x25）字符：

http://unicode.org/standard/reports/tr13/tr13-5.html
#ifdef EBCDIC
// 如果是 EBCDIC 码表环境
#ifndef EBCDIC_NL25
// 如果没有定义 EBCDIC_NL25
#define CHAR_NL                     '\x15'
// 定义换行符为 0x15
#define CHAR_NEL                    '\x25'
// 定义NEL为0x25
#define STR_NL                      "\x15"
// 定义换行符字符串为 "\x15"
#define STR_NEL                     "\x25"
// 定义NEL字符串为 "\x25"
#else
// 如果定义了 EBCDIC_NL25
#define CHAR_NL                     '\x25'
// 定义换行符为 0x25
#define CHAR_NEL                    '\x15'
// 定义NEL为0x15
#define STR_NL                      "\x25"
// 定义换行符字符串为 "\x25"
#define STR_NEL                     "\x15"
// 定义NEL字符串为 "\x15"
#endif

#define CHAR_LF                     CHAR_NL
// 定义换行符为 NL
#define STR_LF                      STR_NL
// 定义换行符字符串为 NL
#define CHAR_ESC                    '\047'
// 定义 ESC 为十进制 47 对应的字符
#define CHAR_DEL                    '\007'
// 定义 DEL 为十进制 7 对应的字符
#define CHAR_NBSP                   ((unsigned char)'\x41')
// 定义 NBSP 为十六进制 41 对应的字符
#define STR_ESC                     "\047"
// 定义 ESC 字符串为 "\047"
#define STR_DEL                     "\007"
// 定义 DEL 字符串为 "\007"

#else  /* Not EBCDIC */

/* In ASCII/Unicode, linefeed is '\n' and we equate this to NL for
compatibility. NEL is the Unicode newline character; make sure it is
a positive value. */

#define CHAR_LF                     '\n'
// 定义换行符为 \n
#define CHAR_NL                     CHAR_LF
// 定义 NL 为 LF
#define CHAR_NEL                    ((unsigned char)'\x85')
// 定义 NEL 为十六进制 85 对应的字符
#define CHAR_ESC                    '\033'
// 定义 ESC 为十进制 33 对应的字符
#define CHAR_DEL                    '\177'
// 定义 DEL 为十进制 177 对应的字符
#define CHAR_NBSP                   ((unsigned char)'\xa0')
// 定义 NBSP 为十六进制 a0 对应的字符

#define STR_LF                      "\n"
// 定义换行符字符串为 "\n"
#define STR_NL                      STR_LF
// 定义 NL 字符串为 LF
#define STR_NEL                     "\x85"
// 定义 NEL 字符串为 "\x85"
#define STR_ESC                     "\033"
// 定义 ESC 字符串为 "\033"
#define STR_DEL                     "\177"
// 定义 DEL 字符串为 "\177"

#endif  /* EBCDIC */

/* The remaining definitions work in both environments. */

#define CHAR_NUL                    '\0'
// 定义 NUL 为空字符
#define CHAR_HT                     '\t'
// 定义 HT 为制表符
#define CHAR_VT                     '\v'
// 定义 VT 为垂直制表符
#define CHAR_FF                     '\f'
// 定义 FF 为换页符
#define CHAR_CR                     '\r'
// 定义 CR 为回车符
#define CHAR_BS                     '\b'
// 定义 BS 为退格符
# 定义控制字符 BEL
#define CHAR_BEL                    '\a'

# 定义空格字符
#define CHAR_SPACE                  ' '
# 定义感叹号字符
#define CHAR_EXCLAMATION_MARK       '!'
# 定义双引号字符
#define CHAR_QUOTATION_MARK         '"'
# 定义井号字符
#define CHAR_NUMBER_SIGN            '#'
# 定义美元符号字符
#define CHAR_DOLLAR_SIGN            '$'
# 定义百分号字符
#define CHAR_PERCENT_SIGN           '%'
# 定义和字符
#define CHAR_AMPERSAND              '&'
# 定义撇号字符
#define CHAR_APOSTROPHE             '\''
# 定义左括号字符
#define CHAR_LEFT_PARENTHESIS       '('
# 定义右括号字符
#define CHAR_RIGHT_PARENTHESIS      ')'
# 定义星号字符
#define CHAR_ASTERISK               '*'
# 定义加号字符
#define CHAR_PLUS                   '+'
# 定义逗号字符
#define CHAR_COMMA                  ','
# 定义减号字符
#define CHAR_MINUS                  '-'
# 定义句号字符
#define CHAR_DOT                    '.'
# 定义斜杠字符
#define CHAR_SLASH                  '/'
# 定义数字 0 字符
#define CHAR_0                      '0'
# 定义数字 1 字符
#define CHAR_1                      '1'
# 定义数字 2 字符
#define CHAR_2                      '2'
# 定义数字 3 字符
#define CHAR_3                      '3'
# 定义数字 4 字符
#define CHAR_4                      '4'
# 定义数字 5 字符
#define CHAR_5                      '5'
# 定义数字 6 字符
#define CHAR_6                      '6'
# 定义数字 7 字符
#define CHAR_7                      '7'
# 定义数字 8 字符
#define CHAR_8                      '8'
# 定义数字 9 字符
#define CHAR_9                      '9'
# 定义冒号字符
#define CHAR_COLON                  ':'
# 定义分号字符
#define CHAR_SEMICOLON              ';'
# 定义小于号字符
#define CHAR_LESS_THAN_SIGN         '<'
# 定义等于号字符
#define CHAR_EQUALS_SIGN            '='
# 定义大于号字符
#define CHAR_GREATER_THAN_SIGN      '>'
# 定义问号字符
#define CHAR_QUESTION_MARK          '?'
# 定义商用符号字符
#define CHAR_COMMERCIAL_AT          '@'
# 定义大写字母 A 字符
#define CHAR_A                      'A'
# 定义大写字母 B 字符
#define CHAR_B                      'B'
# 定义大写字母 C 字符
#define CHAR_C                      'C'
# 定义大写字母 D 字符
#define CHAR_D                      'D'
# 定义大写字母 E 字符
#define CHAR_E                      'E'
# 定义大写字母 F 字符
#define CHAR_F                      'F'
# 定义大写字母 G 字符
#define CHAR_G                      'G'
# 定义大写字母 H 字符
#define CHAR_H                      'H'
# 定义大写字母 I 字符
#define CHAR_I                      'I'
# 定义大写字母 J 字符
#define CHAR_J                      'J'
# 定义大写字母 K 字符
#define CHAR_K                      'K'
# 定义大写字母 L 字符
#define CHAR_L                      'L'
# 定义大写字母 M 字符
#define CHAR_M                      'M'
# 定义大写字母 N 字符
#define CHAR_N                      'N'
# 定义大写字母 O 字符
#define CHAR_O                      'O'
// 定义字符常量
#define CHAR_P                      'P'  // 定义字符常量 P
#define CHAR_Q                      'Q'  // 定义字符常量 Q
#define CHAR_R                      'R'  // 定义字符常量 R
#define CHAR_S                      'S'  // 定义字符常量 S
#define CHAR_T                      'T'  // 定义字符常量 T
#define CHAR_U                      'U'  // 定义字符常量 U
#define CHAR_V                      'V'  // 定义字符常量 V
#define CHAR_W                      'W'  // 定义字符常量 W
#define CHAR_X                      'X'  // 定义字符常量 X
#define CHAR_Y                      'Y'  // 定义字符常量 Y
#define CHAR_Z                      'Z'  // 定义字符常量 Z
#define CHAR_LEFT_SQUARE_BRACKET    '['  // 定义字符常量 [
#define CHAR_BACKSLASH              '\\'  // 定义字符常量 \
#define CHAR_RIGHT_SQUARE_BRACKET   ']'  // 定义字符常量 ]
#define CHAR_CIRCUMFLEX_ACCENT      '^'  // 定义字符常量 ^
#define CHAR_UNDERSCORE             '_'  // 定义字符常量 _
#define CHAR_GRAVE_ACCENT           '`'  // 定义字符常量 `
#define CHAR_a                      'a'  // 定义字符常量 a
#define CHAR_b                      'b'  // 定义字符常量 b
#define CHAR_c                      'c'  // 定义字符常量 c
#define CHAR_d                      'd'  // 定义字符常量 d
#define CHAR_e                      'e'  // 定义字符常量 e
#define CHAR_f                      'f'  // 定义字符常量 f
#define CHAR_g                      'g'  // 定义字符常量 g
#define CHAR_h                      'h'  // 定义字符常量 h
#define CHAR_i                      'i'  // 定义字符常量 i
#define CHAR_j                      'j'  // 定义字符常量 j
#define CHAR_k                      'k'  // 定义字符常量 k
#define CHAR_l                      'l'  // 定义字符常量 l
#define CHAR_m                      'm'  // 定义字符常量 m
#define CHAR_n                      'n'  // 定义字符常量 n
#define CHAR_o                      'o'  // 定义字符常量 o
#define CHAR_p                      'p'  // 定义字符常量 p
#define CHAR_q                      'q'  // 定义字符常量 q
#define CHAR_r                      'r'  // 定义字符常量 r
#define CHAR_s                      's'  // 定义字符常量 s
#define CHAR_t                      't'  // 定义字符常量 t
#define CHAR_u                      'u'  // 定义字符常量 u
#define CHAR_v                      'v'  // 定义字符常量 v
#define CHAR_w                      'w'  // 定义字符常量 w
#define CHAR_x                      'x'  // 定义字符常量 x
#define CHAR_y                      'y'  // 定义字符常量 y
#define CHAR_z                      'z'  // 定义字符常量 z
#define CHAR_LEFT_CURLY_BRACKET     '{'  // 定义字符常量 {
#define CHAR_VERTICAL_LINE          '|'  // 定义字符常量 |
#define CHAR_RIGHT_CURLY_BRACKET    '}'  // 定义字符常量 }
#define CHAR_TILDE                  '~'  // 定义字符常量 ~

// 定义字符串常量
#define STR_HT                      "\t"  // 定义字符串常量 \t
#define STR_VT                      "\v"  // 定义字符串常量 \v
# 定义转义字符 \f 对应的字符串常量
#define STR_FF                      "\f"
# 定义转义字符 \r 对应的字符串常量
#define STR_CR                      "\r"
# 定义转义字符 \b 对应的字符串常量
#define STR_BS                      "\b"
# 定义转义字符 \a 对应的字符串常量
#define STR_BEL                     "\a"

# 定义空格对应的字符串常量
#define STR_SPACE                   " "
# 定义感叹号对应的字符串常量
#define STR_EXCLAMATION_MARK        "!"
# 定义双引号对应的字符串常量
#define STR_QUOTATION_MARK          "\""
# 定义井号对应的字符串常量
#define STR_NUMBER_SIGN             "#"
# 定义美元符号对应的字符串常量
#define STR_DOLLAR_SIGN             "$"
# 定义百分号对应的字符串常量
#define STR_PERCENT_SIGN            "%"
# 定义和符号对应的字符串常量
#define STR_AMPERSAND               "&"
# 定义单引号对应的字符串常量
#define STR_APOSTROPHE              "'"
# 定义左括号对应的字符串常量
#define STR_LEFT_PARENTHESIS        "("
# 定义右括号对应的字符串常量
#define STR_RIGHT_PARENTHESIS       ")"
# 定义星号对应的字符串常量
#define STR_ASTERISK                "*"
# 定义加号对应的字符串常量
#define STR_PLUS                    "+"
# 定义逗号对应的字符串常量
#define STR_COMMA                   ","
# 定义减号对应的字符串常量
#define STR_MINUS                   "-"
# 定义句号对应的字符串常量
#define STR_DOT                     "."
# 定义斜杠对应的字符串常量
#define STR_SLASH                   "/"
# 定义数字 0 对应的字符串常量
#define STR_0                       "0"
# 定义数字 1 对应的字符串常量
#define STR_1                       "1"
# 定义数字 2 对应的字符串常量
#define STR_2                       "2"
# 定义数字 3 对应的字符串常量
#define STR_3                       "3"
# 定义数字 4 对应的字符串常量
#define STR_4                       "4"
# 定义数字 5 对应的字符串常量
#define STR_5                       "5"
# 定义数字 6 对应的字符串常量
#define STR_6                       "6"
# 定义数字 7 对应的字符串常量
#define STR_7                       "7"
# 定义数字 8 对应的字符串常量
#define STR_8                       "8"
# 定义数字 9 对应的字符串常量
#define STR_9                       "9"
# 定义冒号对应的字符串常量
#define STR_COLON                   ":"
# 定义分号对应的字符串常量
#define STR_SEMICOLON               ";"
# 定义小于号对应的字符串常量
#define STR_LESS_THAN_SIGN          "<"
# 定义等号对应的字符串常量
#define STR_EQUALS_SIGN             "="
# 定义大于号对应的字符串常量
#define STR_GREATER_THAN_SIGN       ">"
# 定义问号对应的字符串常量
#define STR_QUESTION_MARK           "?"
# 定义商用符号对应的字符串常量
#define STR_COMMERCIAL_AT           "@"
# 定义大写字母 A 对应的字符串常量
#define STR_A                       "A"
# 定义大写字母 B 对应的字符串常量
#define STR_B                       "B"
# 定义大写字母 C 对应的字符串常量
#define STR_C                       "C"
# 定义大写字母 D 对应的字符串常量
#define STR_D                       "D"
# 定义大写字母 E 对应的字符串常量
#define STR_E                       "E"
# 定义大写字母 F 对应的字符串常量
#define STR_F                       "F"
# 定义大写字母 G 对应的字符串常量
#define STR_G                       "G"
# 定义大写字母 H 对应的字符串常量
#define STR_H                       "H"
# 定义大写字母 I 对应的字符串常量
#define STR_I                       "I"
# 定义大写字母 J 对应的字符串常量
#define STR_J                       "J"
# 定义大写字母 K 对应的字符串常量
#define STR_K                       "K"
# 定义大写字母 L 对应的字符串常量
#define STR_L                       "L"
# 定义字符串常量
#define STR_M                       "M"
#define STR_N                       "N"
#define STR_O                       "O"
#define STR_P                       "P"
#define STR_Q                       "Q"
#define STR_R                       "R"
#define STR_S                       "S"
#define STR_T                       "T"
#define STR_U                       "U"
#define STR_V                       "V"
#define STR_W                       "W"
#define STR_X                       "X"
#define STR_Y                       "Y"
#define STR_Z                       "Z"
#define STR_LEFT_SQUARE_BRACKET     "["
#define STR_BACKSLASH               "\\"
#define STR_RIGHT_SQUARE_BRACKET    "]"
#define STR_CIRCUMFLEX_ACCENT       "^"
#define STR_UNDERSCORE              "_"
#define STR_GRAVE_ACCENT            "`"
#define STR_a                       "a"
#define STR_b                       "b"
#define STR_c                       "c"
#define STR_d                       "d"
#define STR_e                       "e"
#define STR_f                       "f"
#define STR_g                       "g"
#define STR_h                       "h"
#define STR_i                       "i"
#define STR_j                       "j"
#define STR_k                       "k"
#define STR_l                       "l"
#define STR_m                       "m"
#define STR_n                       "n"
#define STR_o                       "o"
#define STR_p                       "p"
#define STR_q                       "q"
#define STR_r                       "r"
#define STR_s                       "s"
#define STR_t                       "t"
#define STR_u                       "u"
#define STR_v                       "v"
#define STR_w                       "w"
#define STR_x                       "x"
#define STR_y                       "y"
#define STR_z                       "z"
#define STR_LEFT_CURLY_BRACKET      "{"
#define STR_VERTICAL_LINE           "|"
#define STR_RIGHT_CURLY_BRACKET     "}"
# 定义波浪线字符串常量
#define STR_TILDE                   "~"

# 定义字符串常量
#define STRING_ACCEPT0               "ACCEPT\0"
#define STRING_COMMIT0               "COMMIT\0"
#define STRING_F0                    "F\0"
#define STRING_FAIL0                 "FAIL\0"
#define STRING_MARK0                 "MARK\0"
#define STRING_PRUNE0                "PRUNE\0"
#define STRING_SKIP0                 "SKIP\0"
#define STRING_THEN                  "THEN"

# 定义字符串常量
#define STRING_atomic0               "atomic\0"
#define STRING_pla0                  "pla\0"
#define STRING_plb0                  "plb\0"
#define STRING_napla0                "napla\0"
#define STRING_naplb0                "naplb\0"
#define STRING_nla0                  "nla\0"
#define STRING_nlb0                  "nlb\0"
#define STRING_sr0                   "sr\0"
#define STRING_asr0                  "asr\0"
#define STRING_positive_lookahead0   "positive_lookahead\0"
#define STRING_positive_lookbehind0  "positive_lookbehind\0"
#define STRING_non_atomic_positive_lookahead0   "non_atomic_positive_lookahead\0"
#define STRING_non_atomic_positive_lookbehind0  "non_atomic_positive_lookbehind\0"
#define STRING_negative_lookahead0   "negative_lookahead\0"
#define STRING_negative_lookbehind0  "negative_lookbehind\0"
#define STRING_script_run0           "script_run\0"
#define STRING_atomic_script_run     "atomic_script_run"

# 定义字符串常量
#define STRING_alpha0                "alpha\0"
#define STRING_lower0                "lower\0"
#define STRING_upper0                "upper\0"
#define STRING_alnum0                "alnum\0"
#define STRING_ascii0                "ascii\0"
#define STRING_blank0                "blank\0"
#define STRING_cntrl0                "cntrl\0"
#define STRING_digit0                "digit\0"
#define STRING_graph0                "graph\0"
#define STRING_print0                "print\0"
#define STRING_punct0                "punct\0"
#define STRING_space0                "space\0"
#define STRING_word0                 "word\0"
#define STRING_xdigit                "xdigit"
#define STRING_DEFINE                "DEFINE"  // 定义字符串常量"DEFINE"
#define STRING_VERSION               "VERSION"  // 定义字符串常量"VERSION"
#define STRING_WEIRD_STARTWORD       "[:<:]]"  // 定义字符串常量"[:<:]]"
#define STRING_WEIRD_ENDWORD         "[:>:]]"  // 定义字符串常量"[:>:]]"

#define STRING_CR_RIGHTPAR                "CR)"  // 定义字符串常量"CR)"
#define STRING_LF_RIGHTPAR                "LF)"  // 定义字符串常量"LF)"
#define STRING_CRLF_RIGHTPAR              "CRLF)"  // 定义字符串常量"CRLF)"
#define STRING_ANY_RIGHTPAR               "ANY)"  // 定义字符串常量"ANY)"
#define STRING_ANYCRLF_RIGHTPAR           "ANYCRLF)"  // 定义字符串常量"ANYCRLF)"
#define STRING_NUL_RIGHTPAR               "NUL)"  // 定义字符串常量"NUL)"
#define STRING_BSR_ANYCRLF_RIGHTPAR       "BSR_ANYCRLF)"  // 定义字符串常量"BSR_ANYCRLF)"
#define STRING_BSR_UNICODE_RIGHTPAR       "BSR_UNICODE)"  // 定义字符串常量"BSR_UNICODE)"
#define STRING_UTF8_RIGHTPAR              "UTF8)"  // 定义字符串常量"UTF8)"
#define STRING_UTF16_RIGHTPAR             "UTF16)"  // 定义字符串常量"UTF16)"
#define STRING_UTF32_RIGHTPAR             "UTF32)"  // 定义字符串常量"UTF32)"
#define STRING_UTF_RIGHTPAR               "UTF)"  // 定义字符串常量"UTF)"
#define STRING_UCP_RIGHTPAR               "UCP)"  // 定义字符串常量"UCP)"
#define STRING_NO_AUTO_POSSESS_RIGHTPAR   "NO_AUTO_POSSESS)"  // 定义字符串常量"NO_AUTO_POSSESS)"
#define STRING_NO_DOTSTAR_ANCHOR_RIGHTPAR "NO_DOTSTAR_ANCHOR)"  // 定义字符串常量"NO_DOTSTAR_ANCHOR)"
#define STRING_NO_JIT_RIGHTPAR            "NO_JIT)"  // 定义字符串常量"NO_JIT)"
#define STRING_NO_START_OPT_RIGHTPAR      "NO_START_OPT)"  // 定义字符串常量"NO_START_OPT)"
#define STRING_NOTEMPTY_RIGHTPAR          "NOTEMPTY)"  // 定义字符串常量"NOTEMPTY)"
#define STRING_NOTEMPTY_ATSTART_RIGHTPAR  "NOTEMPTY_ATSTART)"  // 定义字符串常量"NOTEMPTY_ATSTART)"
#define STRING_LIMIT_HEAP_EQ              "LIMIT_HEAP="  // 定义字符串常量"LIMIT_HEAP="
#define STRING_LIMIT_MATCH_EQ             "LIMIT_MATCH="  // 定义字符串常量"LIMIT_MATCH="
#define STRING_LIMIT_DEPTH_EQ             "LIMIT_DEPTH="  // 定义字符串常量"LIMIT_DEPTH="
#define STRING_LIMIT_RECURSION_EQ         "LIMIT_RECURSION="  // 定义字符串常量"LIMIT_RECURSION="
#define STRING_MARK                       "MARK"  // 定义字符串常量"MARK"

#define STRING_bc                         "bc"  // 定义字符串常量"bc"
#define STRING_bidiclass                  "bidiclass"  // 定义字符串常量"bidiclass"
#define STRING_sc                         "sc"  // 定义字符串常量"sc"
#define STRING_script                     "script"  // 定义字符串常量"script"
#define STRING_scriptextensions           "scriptextensions"  // 定义字符串常量"scriptextensions"
#define STRING_scx                        "scx"  // 定义字符串常量"scx"

#else  /* SUPPORT_UNICODE */

/* UTF-8 support is enabled; always use UTF-8 (=ASCII) character codes. This
works in both modes non-EBCDIC platforms, and on EBCDIC platforms in UTF-8 mode
only. */
# 定义常量：水平制表符
#define CHAR_HT                     '\011'
# 定义常量：垂直制表符
#define CHAR_VT                     '\013'
# 定义常量：换页符
#define CHAR_FF                     '\014'
# 定义常量：回车符
#define CHAR_CR                     '\015'
# 定义常量：换行符
#define CHAR_LF                     '\012'
# 定义常量：换行符
#define CHAR_NL                     CHAR_LF
# 定义常量：下一行
#define CHAR_NEL                    ((unsigned char)'\x85')
# 定义常量：退格符
#define CHAR_BS                     '\010'
# 定义常量：响铃
#define CHAR_BEL                    '\007'
# 定义常量：转义
#define CHAR_ESC                    '\033'
# 定义常量：删除
#define CHAR_DEL                    '\177'

# 定义常量：空字符
#define CHAR_NUL                    '\0'
# 定义常量：空格
#define CHAR_SPACE                  '\040'
# 定义常量：感叹号
#define CHAR_EXCLAMATION_MARK       '\041'
# 定义常量：引号
#define CHAR_QUOTATION_MARK         '\042'
# 定义常量：井号
#define CHAR_NUMBER_SIGN            '\043'
# 定义常量：美元符号
#define CHAR_DOLLAR_SIGN            '\044'
# 定义常量：百分号
#define CHAR_PERCENT_SIGN           '\045'
# 定义常量：和号
#define CHAR_AMPERSAND              '\046'
# 定义常量：撇号
#define CHAR_APOSTROPHE             '\047'
# 定义常量：左括号
#define CHAR_LEFT_PARENTHESIS       '\050'
# 定义常量：右括号
#define CHAR_RIGHT_PARENTHESIS      '\051'
# 定义常量：星号
#define CHAR_ASTERISK               '\052'
# 定义常量：加号
#define CHAR_PLUS                   '\053'
# 定义常量：逗号
#define CHAR_COMMA                  '\054'
# 定义常量：减号
#define CHAR_MINUS                  '\055'
# 定义常量：句点
#define CHAR_DOT                    '\056'
# 定义常量：斜杠
#define CHAR_SLASH                  '\057'
# 定义常量：数字0
#define CHAR_0                      '\060'
# 定义常量：数字1
#define CHAR_1                      '\061'
# 定义常量：数字2
#define CHAR_2                      '\062'
# 定义常量：数字3
#define CHAR_3                      '\063'
# 定义常量：数字4
#define CHAR_4                      '\064'
# 定义常量：数字5
#define CHAR_5                      '\065'
# 定义常量：数字6
#define CHAR_6                      '\066'
# 定义常量：数字7
#define CHAR_7                      '\067'
# 定义常量：数字8
#define CHAR_8                      '\070'
# 定义常量：数字9
#define CHAR_9                      '\071'
# 定义常量：冒号
#define CHAR_COLON                  '\072'
# 定义常量：分号
#define CHAR_SEMICOLON              '\073'
# 定义常量：小于号
#define CHAR_LESS_THAN_SIGN         '\074'
# 定义常量：等于号
#define CHAR_EQUALS_SIGN            '\075'
# 定义常量：大于号
#define CHAR_GREATER_THAN_SIGN      '\076'
# 定义常量：问号
#define CHAR_QUESTION_MARK          '\077'
# 定义常量：商用符号
#define CHAR_COMMERCIAL_AT          '\100'
# 定义常量：大写字母A
#define CHAR_A                      '\101'
// 定义字符常量 B，值为 ASCII 码对应的字符
#define CHAR_B                      '\102'
// 定义字符常量 C，值为 ASCII 码对应的字符
#define CHAR_C                      '\103'
// 定义字符常量 D，值为 ASCII 码对应的字符
#define CHAR_D                      '\104'
// 定义字符常量 E，值为 ASCII 码对应的字符
#define CHAR_E                      '\105'
// 定义字符常量 F，值为 ASCII 码对应的字符
#define CHAR_F                      '\106'
// 定义字符常量 G，值为 ASCII 码对应的字符
#define CHAR_G                      '\107'
// 定义字符常量 H，值为 ASCII 码对应的字符
#define CHAR_H                      '\110'
// 定义字符常量 I，值为 ASCII 码对应的字符
#define CHAR_I                      '\111'
// 定义字符常量 J，值为 ASCII 码对应的字符
#define CHAR_J                      '\112'
// 定义字符常量 K，值为 ASCII 码对应的字符
#define CHAR_K                      '\113'
// 定义字符常量 L，值为 ASCII 码对应的字符
#define CHAR_L                      '\114'
// 定义字符常量 M，值为 ASCII 码对应的字符
#define CHAR_M                      '\115'
// 定义字符常量 N，值为 ASCII 码对应的字符
#define CHAR_N                      '\116'
// 定义字符常量 O，值为 ASCII 码对应的字符
#define CHAR_O                      '\117'
// 定义字符常量 P，值为 ASCII 码对应的字符
#define CHAR_P                      '\120'
// 定义字符常量 Q，值为 ASCII 码对应的字符
#define CHAR_Q                      '\121'
// 定义字符常量 R，值为 ASCII 码对应的字符
#define CHAR_R                      '\122'
// 定义字符常量 S，值为 ASCII 码对应的字符
#define CHAR_S                      '\123'
// 定义字符常量 T，值为 ASCII 码对应的字符
#define CHAR_T                      '\124'
// 定义字符常量 U，值为 ASCII 码对应的字符
#define CHAR_U                      '\125'
// 定义字符常量 V，值为 ASCII 码对应的字符
#define CHAR_V                      '\126'
// 定义字符常量 W，值为 ASCII 码对应的字符
#define CHAR_W                      '\127'
// 定义字符常量 X，值为 ASCII 码对应的字符
#define CHAR_X                      '\130'
// 定义字符常量 Y，值为 ASCII 码对应的字符
#define CHAR_Y                      '\131'
// 定义字符常量 Z，值为 ASCII 码对应的字符
#define CHAR_Z                      '\132'
// 定义字符常量 [，值为 ASCII 码对应的字符
#define CHAR_LEFT_SQUARE_BRACKET    '\133'
// 定义字符常量 \，值为 ASCII 码对应的字符
#define CHAR_BACKSLASH              '\134'
// 定义字符常量 ]，值为 ASCII 码对应的字符
#define CHAR_RIGHT_SQUARE_BRACKET   '\135'
// 定义字符常量 ^，值为 ASCII 码对应的字符
#define CHAR_CIRCUMFLEX_ACCENT      '\136'
// 定义字符常量 _，值为 ASCII 码对应的字符
#define CHAR_UNDERSCORE             '\137'
// 定义字符常量 `，值为 ASCII 码对应的字符
#define CHAR_GRAVE_ACCENT           '\140'
// 定义字符常量 a，值为 ASCII 码对应的字符
#define CHAR_a                      '\141'
// 定义字符常量 b，值为 ASCII 码对应的字符
#define CHAR_b                      '\142'
// 定义字符常量 c，值为 ASCII 码对应的字符
#define CHAR_c                      '\143'
// 定义字符常量 d，值为 ASCII 码对应的字符
#define CHAR_d                      '\144'
// 定义字符常量 e，值为 ASCII 码对应的字符
#define CHAR_e                      '\145'
// 定义字符常量 f，值为 ASCII 码对应的字符
#define CHAR_f                      '\146'
// 定义字符常量 g，值为 ASCII 码对应的字符
#define CHAR_g                      '\147'
// 定义字符常量 h，值为 ASCII 码对应的字符
#define CHAR_h                      '\150'
// 定义字符常量 i，值为 ASCII 码对应的字符
#define CHAR_i                      '\151'
// 定义字符常量 j，值为 ASCII 码对应的字符
#define CHAR_j                      '\152'
// 定义字符常量 k，值为 ASCII 码对应的字符
#define CHAR_k                      '\153'
// 定义字符常量 l，值为 ASCII 码对应的字符
#define CHAR_l                      '\154'
// 定义字符常量 m，值为 ASCII 码对应的字符
#define CHAR_m                      '\155'
// 定义字符常量 n，值为 ASCII 码对应的字符
#define CHAR_n                      '\156'
// 定义字符常量 o，值为 ASCII 码对应的字符
#define CHAR_o                      '\157'
# 定义字符常量
#define CHAR_p                      '\160'  # 定义字符常量 p
#define CHAR_q                      '\161'  # 定义字符常量 q
#define CHAR_r                      '\162'  # 定义字符常量 r
#define CHAR_s                      '\163'  # 定义字符常量 s
#define CHAR_t                      '\164'  # 定义字符常量 t
#define CHAR_u                      '\165'  # 定义字符常量 u
#define CHAR_v                      '\166'  # 定义字符常量 v
#define CHAR_w                      '\167'  # 定义字符常量 w
#define CHAR_x                      '\170'  # 定义字符常量 x
#define CHAR_y                      '\171'  # 定义字符常量 y
#define CHAR_z                      '\172'  # 定义字符常量 z
#define CHAR_LEFT_CURLY_BRACKET     '\173'  # 定义字符常量 {
#define CHAR_VERTICAL_LINE          '\174'  # 定义字符常量 |
#define CHAR_RIGHT_CURLY_BRACKET    '\175'  # 定义字符常量 }
#define CHAR_TILDE                  '\176'  # 定义字符常量 ~
#define CHAR_NBSP                   ((unsigned char)'\xa0')  # 定义字符常量 NBSP

# 定义字符串常量
#define STR_HT                      "\011"  # 定义字符串常量 HT
#define STR_VT                      "\013"  # 定义字符串常量 VT
#define STR_FF                      "\014"  # 定义字符串常量 FF
#define STR_CR                      "\015"  # 定义字符串常量 CR
#define STR_NL                      "\012"  # 定义字符串常量 NL
#define STR_BS                      "\010"  # 定义字符串常量 BS
#define STR_BEL                     "\007"  # 定义字符串常量 BEL
#define STR_ESC                     "\033"  # 定义字符串常量 ESC
#define STR_DEL                     "\177"  # 定义字符串常量 DEL

# 定义字符串常量
#define STR_SPACE                   "\040"  # 定义字符串常量 SPACE
#define STR_EXCLAMATION_MARK        "\041"  # 定义字符串常量 !
#define STR_QUOTATION_MARK          "\042"  # 定义字符串常量 "
#define STR_NUMBER_SIGN             "\043"  # 定义字符串常量 #
#define STR_DOLLAR_SIGN             "\044"  # 定义字符串常量 $
#define STR_PERCENT_SIGN            "\045"  # 定义字符串常量 %
#define STR_AMPERSAND               "\046"  # 定义字符串常量 &
#define STR_APOSTROPHE              "\047"  # 定义字符串常量 '
#define STR_LEFT_PARENTHESIS        "\050"  # 定义字符串常量 (
#define STR_RIGHT_PARENTHESIS       "\051"  # 定义字符串常量 )
#define STR_ASTERISK                "\052"  # 定义字符串常量 *
#define STR_PLUS                    "\053"  # 定义字符串常量 +
#define STR_COMMA                   "\054"  # 定义字符串常量 ,
#define STR_MINUS                   "\055"  # 定义字符串常量 -
#define STR_DOT                     "\056"  # 定义字符串常量 .
#define STR_SLASH                   "\057"  # 定义字符串常量 /
#define STR_0                       "\060"  # 定义字符串常量 0
#define STR_1                       "\061"  # 定义字符串常量 1
#define STR_2                       "\062"  # 定义字符串常量 2
#define STR_3                       "\063"  # 定义字符串常量 3
#define STR_4                       "\064"  # 定义字符串常量 4
# 定义字符串常量，表示对应的 ASCII 字符
#define STR_5                       "\065"
#define STR_6                       "\066"
#define STR_7                       "\067"
#define STR_8                       "\070"
#define STR_9                       "\071"
#define STR_COLON                   "\072"
#define STR_SEMICOLON               "\073"
#define STR_LESS_THAN_SIGN          "\074"
#define STR_EQUALS_SIGN             "\075"
#define STR_GREATER_THAN_SIGN       "\076"
#define STR_QUESTION_MARK           "\077"
#define STR_COMMERCIAL_AT           "\100"
#define STR_A                       "\101"
#define STR_B                       "\102"
#define STR_C                       "\103"
#define STR_D                       "\104"
#define STR_E                       "\105"
#define STR_F                       "\106"
#define STR_G                       "\107"
#define STR_H                       "\110"
#define STR_I                       "\111"
#define STR_J                       "\112"
#define STR_K                       "\113"
#define STR_L                       "\114"
#define STR_M                       "\115"
#define STR_N                       "\116"
#define STR_O                       "\117"
#define STR_P                       "\120"
#define STR_Q                       "\121"
#define STR_R                       "\122"
#define STR_S                       "\123"
#define STR_T                       "\124"
#define STR_U                       "\125"
#define STR_V                       "\126"
#define STR_W                       "\127"
#define STR_X                       "\130"
#define STR_Y                       "\131"
#define STR_Z                       "\132"
#define STR_LEFT_SQUARE_BRACKET     "\133"
#define STR_BACKSLASH               "\134"
#define STR_RIGHT_SQUARE_BRACKET    "\135"
#define STR_CIRCUMFLEX_ACCENT       "\136"
#define STR_UNDERSCORE              "\137"
#define STR_GRAVE_ACCENT            "\140"
#define STR_a                       "\141"
#define STR_b                       "\142"
// 定义字符串常量，表示字母 c
#define STR_c                       "\143"
// 定义字符串常量，表示字母 d
#define STR_d                       "\144"
// 定义字符串常量，表示字母 e
#define STR_e                       "\145"
// 定义字符串常量，表示字母 f
#define STR_f                       "\146"
// 定义字符串常量，表示字母 g
#define STR_g                       "\147"
// 定义字符串常量，表示字母 h
#define STR_h                       "\150"
// 定义字符串常量，表示字母 i
#define STR_i                       "\151"
// 定义字符串常量，表示字母 j
#define STR_j                       "\152"
// 定义字符串常量，表示字母 k
#define STR_k                       "\153"
// 定义字符串常量，表示字母 l
#define STR_l                       "\154"
// 定义字符串常量，表示字母 m
#define STR_m                       "\155"
// 定义字符串常量，表示字母 n
#define STR_n                       "\156"
// 定义字符串常量，表示字母 o
#define STR_o                       "\157"
// 定义字符串常量，表示字母 p
#define STR_p                       "\160"
// 定义字符串常量，表示字母 q
#define STR_q                       "\161"
// 定义字符串常量，表示字母 r
#define STR_r                       "\162"
// 定义字符串常量，表示字母 s
#define STR_s                       "\163"
// 定义字符串常量，表示字母 t
#define STR_t                       "\164"
// 定义字符串常量，表示字母 u
#define STR_u                       "\165"
// 定义字符串常量，表示字母 v
#define STR_v                       "\166"
// 定义字符串常量，表示字母 w
#define STR_w                       "\167"
// 定义字符串常量，表示字母 x
#define STR_x                       "\170"
// 定义字符串常量，表示字母 y
#define STR_y                       "\171"
// 定义字符串常量，表示字母 z
#define STR_z                       "\172"
// 定义字符串常量，表示左花括号
#define STR_LEFT_CURLY_BRACKET      "\173"
// 定义字符串常量，表示竖线
#define STR_VERTICAL_LINE           "\174"
// 定义字符串常量，表示右花括号
#define STR_RIGHT_CURLY_BRACKET     "\175"
// 定义字符串常量，表示波浪号
#define STR_TILDE                   "\176"

// 定义字符串常量，表示 ACCEPT0
#define STRING_ACCEPT0               STR_A STR_C STR_C STR_E STR_P STR_T "\0"
// 定义字符串常量，表示 COMMIT0
#define STRING_COMMIT0               STR_C STR_O STR_M STR_M STR_I STR_T "\0"
// 定义字符串常量，表示 F0
#define STRING_F0                    STR_F "\0"
// 定义字符串常量，表示 FAIL0
#define STRING_FAIL0                 STR_F STR_A STR_I STR_L "\0"
// 定义字符串常量，表示 MARK0
#define STRING_MARK0                 STR_M STR_A STR_R STR_K "\0"
// 定义字符串常量，表示 PRUNE0
#define STRING_PRUNE0                STR_P STR_R STR_U STR_N STR_E "\0"
// 定义字符串常量，表示 SKIP0
#define STRING_SKIP0                 STR_S STR_K STR_I STR_P "\0"
// 定义字符串常量，表示 THEN
#define STRING_THEN                  STR_T STR_H STR_E STR_N

// 定义字符串常量，表示 atomic0
#define STRING_atomic0               STR_a STR_t STR_o STR_m STR_i STR_c "\0"
// 定义字符串常量，表示 pla0
#define STRING_pla0                  STR_p STR_l STR_a "\0"
// 定义字符串常量，表示 plb0
#define STRING_plb0                  STR_p STR_l STR_b "\0"
# 定义字符串常量 "napla0"，值为 "napla\0"
#define STRING_napla0                STR_n STR_a STR_p STR_l STR_a "\0"
# 定义字符串常量 "naplb0"，值为 "naplb\0"
#define STRING_naplb0                STR_n STR_a STR_p STR_l STR_b "\0"
# 定义字符串常量 "nla0"，值为 "nla\0"
#define STRING_nla0                  STR_n STR_l STR_a "\0"
# 定义字符串常量 "nlb0"，值为 "nlb\0"
#define STRING_nlb0                  STR_n STR_l STR_b "\0"
# 定义字符串常量 "sr0"，值为 "sr\0"
#define STRING_sr0                   STR_s STR_r "\0"
# 定义字符串常量 "asr0"，值为 "asr\0"
#define STRING_asr0                  STR_a STR_s STR_r "\0"
# 定义字符串常量 "positive_lookahead0"，值为 "positive_lookahead\0"
#define STRING_positive_lookahead0   STR_p STR_o STR_s STR_i STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_a STR_h STR_e STR_a STR_d "\0"
# 定义字符串常量 "positive_lookbehind0"，值为 "positive_lookbehind\0"
#define STRING_positive_lookbehind0  STR_p STR_o STR_s STR_i STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_b STR_e STR_h STR_i STR_n STR_d "\0"
# 定义字符串常量 "non_atomic_positive_lookahead0"，值为 "non_atomic_positive_lookahead\0"
#define STRING_non_atomic_positive_lookahead0   STR_n STR_o STR_n STR_UNDERSCORE STR_a STR_t STR_o STR_m STR_i STR_c STR_UNDERSCORE STR_p STR_o STR_s STR_i STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_a STR_h STR_e STR_a STR_d "\0"
# 定义字符串常量 "non_atomic_positive_lookbehind0"，值为 "non_atomic_positive_lookbehind\0"
#define STRING_non_atomic_positive_lookbehind0  STR_n STR_o STR_n STR_UNDERSCORE STR_a STR_t STR_o STR_m STR_i STR_c STR_UNDERSCORE STR_p STR_o STR_s STR_i STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_b STR_e STR_h STR_i STR_n STR_d "\0"
# 定义字符串常量 "negative_lookahead0"，值为 "negative_lookahead\0"
#define STRING_negative_lookahead0   STR_n STR_e STR_g STR_a STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_a STR_h STR_e STR_a STR_d "\0"
# 定义字符串常量 "negative_lookbehind0"，值为 "negative_lookbehind\0"
#define STRING_negative_lookbehind0  STR_n STR_e STR_g STR_a STR_t STR_i STR_v STR_e STR_UNDERSCORE STR_l STR_o STR_o STR_k STR_b STR_e STR_h STR_i STR_n STR_d "\0"
# 定义字符串常量 "script_run0"，值为 "script_run\0"
#define STRING_script_run0           STR_s STR_c STR_r STR_i STR_p STR_t STR_UNDERSCORE STR_r STR_u STR_n "\0"
# 定义字符串常量 "atomic_script_run"，值为 "atomic_script_run"
#define STRING_atomic_script_run     STR_a STR_t STR_o STR_m STR_i STR_c STR_UNDERSCORE STR_s STR_c STR_r STR_i STR_p STR_t STR_UNDERSCORE STR_r STR_u STR_n

# 定义字符串常量 "alpha0"，值为 "alpha\0"
#define STRING_alpha0                STR_a STR_l STR_p STR_h STR_a "\0"
# 定义字符串常量 "lower0"，值为 "lower\0"
#define STRING_lower0                STR_l STR_o STR_w STR_e STR_r "\0"
# 定义字符串常量，表示大写字母
#define STRING_upper0                STR_u STR_p STR_p STR_e STR_r "\0"
# 定义字符串常量，表示字母和数字
#define STRING_alnum0                STR_a STR_l STR_n STR_u STR_m "\0"
# 定义字符串常量，表示 ASCII 字符
#define STRING_ascii0                STR_a STR_s STR_c STR_i STR_i "\0"
# 定义字符串常量，表示空白字符
#define STRING_blank0                STR_b STR_l STR_a STR_n STR_k "\0"
# 定义字符串常量，表示控制字符
#define STRING_cntrl0                STR_c STR_n STR_t STR_r STR_l "\0"
# 定义字符串常量，表示数字字符
#define STRING_digit0                STR_d STR_i STR_g STR_i STR_t "\0"
# 定义字符串常量，表示可打印字符
#define STRING_graph0                STR_g STR_r STR_a STR_p STR_h "\0"
# 定义字符串常量，表示可打印字符
#define STRING_print0                STR_p STR_r STR_i STR_n STR_t "\0"
# 定义字符串常量，表示标点符号
#define STRING_punct0                STR_p STR_u STR_n STR_c STR_t "\0"
# 定义字符串常量，表示空白字符
#define STRING_space0                STR_s STR_p STR_a STR_c STR_e "\0"
# 定义字符串常量，表示单词字符
#define STRING_word0                 STR_w STR_o STR_r STR_d       "\0"
# 定义字符串常量，表示十六进制字符
#define STRING_xdigit                STR_x STR_d STR_i STR_g STR_i STR_t

# 定义字符串常量，表示宏定义
#define STRING_DEFINE                STR_D STR_E STR_F STR_I STR_N STR_E
# 定义字符串常量，表示版本
#define STRING_VERSION               STR_V STR_E STR_R STR_S STR_I STR_O STR_N
# 定义字符串常量，表示奇怪的起始词
#define STRING_WEIRD_STARTWORD       STR_LEFT_SQUARE_BRACKET STR_COLON STR_LESS_THAN_SIGN STR_COLON STR_RIGHT_SQUARE_BRACKET STR_RIGHT_SQUARE_BRACKET
# 定义字符串常量，表示奇怪的结束词
#define STRING_WEIRD_ENDWORD         STR_LEFT_SQUARE_BRACKET STR_COLON STR_GREATER_THAN_SIGN STR_COLON STR_RIGHT_SQUARE_BRACKET STR_RIGHT_SQUARE_BRACKET

# 定义字符串常量，表示回车符和右括号
#define STRING_CR_RIGHTPAR                STR_C STR_R STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示换行符和右括号
#define STRING_LF_RIGHTPAR                STR_L STR_F STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示回车换行符和右括号
#define STRING_CRLF_RIGHTPAR              STR_C STR_R STR_L STR_F STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示任意字符和右括号
#define STRING_ANY_RIGHTPAR               STR_A STR_N STR_Y STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示任意字符和回车换行符和右括号
#define STRING_ANYCRLF_RIGHTPAR           STR_A STR_N STR_Y STR_C STR_R STR_L STR_F STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示空字符和右括号
#define STRING_NUL_RIGHTPAR               STR_N STR_U STR_L STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示反斜杠和任意字符和回车换行符和右括号
#define STRING_BSR_ANYCRLF_RIGHTPAR       STR_B STR_S STR_R STR_UNDERSCORE STR_A STR_N STR_Y STR_C STR_R STR_L STR_F STR_RIGHT_PARENTHESIS
# 定义字符串常量，表示右括号
#define STRING_BSR_UNICODE_RIGHTPAR       STR_B STR_S STR_R STR_UNDERSCORE STR_U STR_N STR_I STR_C STR_O STR_D STR_E STR_RIGHT_PARENTHESIS
#define STRING_UTF8_RIGHTPAR              STR_U STR_T STR_F STR_8 STR_RIGHT_PARENTHESIS
#define STRING_UTF16_RIGHTPAR             STR_U STR_T STR_F STR_1 STR_6 STR_RIGHT_PARENTHESIS
#define STRING_UTF32_RIGHTPAR             STR_U STR_T STR_F STR_3 STR_2 STR_RIGHT_PARENTHESIS
#define STRING_UTF_RIGHTPAR               STR_U STR_T STR_F STR_RIGHT_PARENTHESIS
#define STRING_UCP_RIGHTPAR               STR_U STR_C STR_P STR_RIGHT_PARENTHESIS
#define STRING_NO_AUTO_POSSESS_RIGHTPAR   STR_N STR_O STR_UNDERSCORE STR_A STR_U STR_T STR_O STR_UNDERSCORE STR_P STR_O STR_S STR_S STR_E STR_S STR_RIGHT_PARENTHESIS
#define STRING_NO_DOTSTAR_ANCHOR_RIGHTPAR STR_N STR_O STR_UNDERSCORE STR_D STR_O STR_T STR_S STR_T STR_A STR_R STR_UNDERSCORE STR_A STR_N STR_C STR_H STR_O STR_R STR_RIGHT_PARENTHESIS
#define STRING_NO_JIT_RIGHTPAR            STR_N STR_O STR_UNDERSCORE STR_J STR_I STR_T STR_RIGHT_PARENTHESIS
#define STRING_NO_START_OPT_RIGHTPAR      STR_N STR_O STR_UNDERSCORE STR_S STR_T STR_A STR_R STR_T STR_UNDERSCORE STR_O STR_P STR_T STR_RIGHT_PARENTHESIS
#define STRING_NOTEMPTY_RIGHTPAR          STR_N STR_O STR_T STR_E STR_M STR_P STR_T STR_Y STR_RIGHT_PARENTHESIS
#define STRING_NOTEMPTY_ATSTART_RIGHTPAR  STR_N STR_O STR_T STR_E STR_M STR_P STR_T STR_Y STR_UNDERSCORE STR_A STR_T STR_S STR_T STR_A STR_R STR_T STR_RIGHT_PARENTHESIS
#define STRING_LIMIT_HEAP_EQ              STR_L STR_I STR_M STR_I STR_T STR_UNDERSCORE STR_H STR_E STR_A STR_P STR_EQUALS_SIGN
#define STRING_LIMIT_MATCH_EQ             STR_L STR_I STR_M STR_I STR_T STR_UNDERSCORE STR_M STR_A STR_T STR_C STR_H STR_EQUALS_SIGN
#define STRING_LIMIT_DEPTH_EQ             STR_L STR_I STR_M STR_I STR_T STR_UNDERSCORE STR_D STR_E STR_P STR_T STR_H STR_EQUALS_SIGN
/* 定义递归相等的字符串限制 */
#define STRING_LIMIT_RECURSION_EQ         STR_L STR_I STR_M STR_I STR_T STR_UNDERSCORE STR_R STR_E STR_C STR_U STR_R STR_S STR_I STR_O STR_N STR_EQUALS_SIGN
/* 定义标记字符串 */
#define STRING_MARK                       STR_M STR_A STR_R STR_K

/* 定义字符串 bc */
#define STRING_bc                         STR_b STR_c
/* 定义字符串 bidiclass */
#define STRING_bidiclass                  STR_b STR_i STR_d STR_i STR_c STR_l STR_a STR_s STR_s
/* 定义字符串 sc */
#define STRING_sc                         STR_s STR_c
/* 定义字符串 script */
#define STRING_script                     STR_s STR_c STR_r STR_i STR_p STR_t
/* 定义字符串 scriptextensions */
#define STRING_scriptextensions           STR_s STR_c STR_r STR_i STR_p STR_t STR_e STR_x STR_t STR_e STR_n STR_s STR_i STR_o STR_n STR_s
/* 定义字符串 scx */
#define STRING_scx                        STR_s STR_c STR_x

#endif  /* SUPPORT_UNICODE */

/* -------------------- 字符和字符串名称结束 -------------------*/

/* -------------------- 编译模式的定义 -------------------*/

/* 不同类型的 Unicode 属性代码。如果更改这些定义，必须更新 pcre2_auto_possess.c 中的自动占有表以匹配。 */

/* 任意属性 - 匹配所有字符 */
#define PT_ANY        0    
/* L& - Lu、Ll、Lt 的并集 */
#define PT_LAMP       1    
/* 指定的一般特性（例如 L） */
#define PT_GC         2    
/* 指定的特定特性（例如 Lu） */
#define PT_PC         3    
/* 仅脚本（例如 Han） */
#define PT_SC         4    
/* 脚本扩展（包括 SC） */
#define PT_SCX        5    
/* 字母数字 - L 和 N 的并集 */
#define PT_ALNUM      6    
/* Perl 空格 - 一般类别 Z 加上 9、10、12、13 */
#define PT_SPACE      7    
/* POSIX 空格 - Z 加上 9、10、11、12、13 */
#define PT_PXSPACE    8    
/* 单词 - L 加上 N 加上下划线 */
#define PT_WORD       9    
/* 伪属性：匹配字符列表 */
#define PT_CLIST     10    
/* 可通用字符名字符 */
#define PT_UCNC      11    
/* 指定的双向类 */
#define PT_BIDICL    12    
/* 布尔属性 */
#define PT_BOOL      13    
/* 定义 PT_TABSIZE 为 14，用于自动转换测试的方形表的大小 */

/* 以下特殊属性仅在 XCLASS 项中使用，当指定 POSIX 类并设置 PCRE2_UCP 时使用 - 换句话说，用于处理这些类的 Unicode。它们不通过 \p 或 \P 转义序列可用，因此它们不参与自动转换表。 */

/* 定义 PT_PXGRAPH 为 14，表示 [:graph:] - 标记纸张的字符 */
/* 定义 PT_PXPRINT 为 15，表示 [:print:] - [:graph:] 加上非控制空格 */
/* 定义 PT_PXPUNCT 为 16，表示 [:punct:] - 标点字符 */

/* 此值用于解析 \p 和 \P 转义，表示尚未遇到 \p{script:...} 或 \p{scx:...}。 */

/* 扩展类（OP_XCLASS）的标志位和数据类型，用于包含值大于 255 的字符的类。 */

/* 定义 XCL_NOT 为 0x01，表示这是一个负类 */
/* 定义 XCL_MAP 为 0x02，表示存在一个 32 字节的映射 */
/* 定义 XCL_HASPROP 为 0x04，表示存在属性检查 */

/* 定义 XCL_END 为 0，标记单个项的结束 */
/* 定义 XCL_SINGLE 为 1，表示后面跟着一个单个项（一个多字节字符） */
/* 定义 XCL_RANGE 为 2，表示后面跟着一个范围（两个多字节字符） */
/* 定义 XCL_PROP 为 3，表示 Unicode 属性（后面跟着 2 字节的属性代码） */
/* 定义 XCL_NOTPROP 为 4，表示 Unicode 反转属性（同上） */

/* 这些是不仅仅是特定数据值的编码的转义项，比如 \n。它们必须具有非零值，因为 check_escape() 对于数据字符返回 0。在 pcre2_compile.c 中的 escapes[] 表中，它们的值被取反，以区分它们与数据值。

它们必须按照下面的操作码定义的顺序出现在这里，直到 ESC_z。对于 OP_ALLANY，有一个虚拟值，因为它对应于 DOTALL 模式中的 "." 而不是一个转义序列。它也用于 JavaScript 中的 [^]。
/* 兼容模式，用于非 UTF 模式下的 \C。在非 DOTALL 模式下，"." 的行为类似于 \N。 */

/* 负数用于在 check_escape() 中编码反向引用（\1、\2、\3 等）。代码中有对大于 ESC_b 且小于 ESC_Z 的转义的测试，以检测可能重复的类型。这些是消耗字符的类型。如果在它们之间放入任何不消耗字符的新转义，那么该代码将需要更改。 */

enum { ESC_A = 1, ESC_G, ESC_K, ESC_B, ESC_b, ESC_D, ESC_d, ESC_S, ESC_s,
       ESC_W, ESC_w, ESC_N, ESC_dum, ESC_C, ESC_P, ESC_p, ESC_R, ESC_H,
       ESC_h, ESC_V, ESC_v, ESC_X, ESC_Z, ESC_z,
       ESC_E, ESC_Q, ESC_g, ESC_k };

/********************** 操作码定义 ******************/

/****** 注意 注意 注意 ******

从 1 开始（即 OP_END 之后），直到 OP_EOD 必须按照上面的转义列表依次对应。此外，必须在不调整 pcre2_auto_possess.c 中的 autoposstab 表的情况下，不更改值直到 OP_DOLLM。

每当更新此列表时，必须更新以下两个宏定义以匹配。pcre2_compile.c 中称为 "opcode_possessify" 的 possessification 表也必须更新，还有 pcre2_dfa_match.c 中称为 "coptable" 和 "poptable" 的表也必须更新。

****** 注意 注意 注意 ******/

/* 在 FIRST_AUTOTAB_OP 和 LAST_AUTOTAB_RIGHT_OP 之间的值（包括这两个值）用于决定是否可以自动占有重复字符类型的表。 */

#define FIRST_AUTOTAB_OP       OP_NOT_DIGIT
#define LAST_AUTOTAB_LEFT_OP   OP_EXTUNI
#define LAST_AUTOTAB_RIGHT_OP  OP_DOLLM

};

/* *** 注意 注意 注意 *** 每当更新上面的列表时，以下两个宏定义也必须更新以匹配。还有 pcre2_compile.c 中称为 "opcode_possessify" 的表，以及 pcre2_dfa_match.c 中称为 "coptable" 和 "poptable" 的表也必须更新。 */

/* 此宏为所有操作码定义了文本名称。这些名称仅用于 */
/* for debugging, and some of them are only partial names. The macro is referenced
only in pcre2_printint.c, which fills out the full names in many cases (and in
some cases doesn't actually use these names at all). */

/* This macro defines the length of fixed length operations in the compiled
regex. The lengths are used when searching for specific things, and also in the
debugging printing of a compiled regex. We use a macro so that it can be
defined close to the definitions of the opcodes themselves.

As things have been extended, some of these are no longer fixed lenths, but are
minima instead. For example, the length of a single-character repeat may vary
in UTF-8 mode. The code that uses this table must know about such things. */

/* A magic value for OP_RREF to indicate the "any recursion" condition. */

#define RREF_ANY  0xffff


/* ---------- Private structures that are mode-independent. ---------- */

/* Structure to hold data for custom memory management. */

typedef struct pcre2_memctl {
  void *    (*malloc)(size_t, void *);
  void      (*free)(void *, void *);
  void      *memory_data;
} pcre2_memctl;

/* Structure for building a chain of open capturing subpatterns during
compiling, so that instructions to close them can be compiled when (*ACCEPT) is
encountered. */

typedef struct open_capitem {
  struct open_capitem *next;    /* Chain link */
  uint16_t number;              /* Capture number */
  uint16_t assert_depth;        /* Assertion depth when opened */
} open_capitem;

/* Layout of the UCP type table that translates property names into types and
codes. Each entry used to point directly to a name, but to reduce the number of
relocations in shared libraries, it now has an offset into a single string
instead. */

typedef struct {
  uint16_t name_offset;
  uint16_t type;
  uint16_t value;
} ucp_type_table;

/* Unicode character database (UCD) record format */
# 定义了一个结构体，包含了表示字符属性的各个字段
typedef struct {
  uint8_t script;     /* 表示脚本类型，如 ucp_Arabic 等 */
  uint8_t chartype;   /* 表示字符类型，如 ucp_Cc 等（通用类别） */
  uint8_t gbprop;     /* 表示断字属性，如 ucp_gbControl 等（断字属性） */
  uint8_t caseset;    /* 多字符其他情况的偏移量，如果没有则为零 */
  int32_t other_case; /* 其他情况的偏移量，如果没有则为零 */
  uint16_t scriptx_bidiclass; /* 脚本扩展（11位）和双向类（5位）值 */
  uint16_t bprops;    /* 二进制属性的偏移量 */
} ucd_record;

/* UCD 访问宏 */

#define UCD_BLOCK_SIZE 128
#define REAL_GET_UCD(ch) (PRIV(ucd_records) + \
        PRIV(ucd_stage2)[PRIV(ucd_stage1)[(int)(ch) / UCD_BLOCK_SIZE] * \
        UCD_BLOCK_SIZE + (int)(ch) % UCD_BLOCK_SIZE])

#if PCRE2_CODE_UNIT_WIDTH == 32
#define GET_UCD(ch) ((ch > MAX_UTF_CODE_POINT)? \
  PRIV(dummy_ucd_record) : REAL_GET_UCD(ch))
#else
#define GET_UCD(ch) REAL_GET_UCD(ch)
#endif

#define UCD_SCRIPTX_MASK 0x3ff
#define UCD_BIDICLASS_SHIFT 11
#define UCD_BPROPS_MASK 0xfff

#define UCD_SCRIPTX_PROP(prop) ((prop)->scriptx_bidiclass & UCD_SCRIPTX_MASK)
#define UCD_BIDICLASS_PROP(prop) ((prop)->scriptx_bidiclass >> UCD_BIDICLASS_SHIFT)
#define UCD_BPROPS_PROP(prop) ((prop)->bprops & UCD_BPROPS_MASK)

#define UCD_CHARTYPE(ch)    GET_UCD(ch)->chartype
#define UCD_SCRIPT(ch)      GET_UCD(ch)->script
#define UCD_CATEGORY(ch)    PRIV(ucp_gentype)[UCD_CHARTYPE(ch)]
#define UCD_GRAPHBREAK(ch)  GET_UCD(ch)->gbprop
#define UCD_CASESET(ch)     GET_UCD(ch)->caseset
#define UCD_OTHERCASE(ch)   ((uint32_t)((int)ch + (int)(GET_UCD(ch)->other_case)))
#define UCD_SCRIPTX(ch)     UCD_SCRIPTX_PROP(GET_UCD(ch))
#define UCD_BPROPS(ch)      UCD_BPROPS_PROP(GET_UCD(ch))
#define UCD_BIDICLASS(ch)   UCD_BIDICLASS_PROP(GET_UCD(ch))

/* "scriptx" 和 bprops 字段包含指向32位字的偏移量，形成表示脚本或布尔属性列表的位图。这些宏通过编号测试或设置地图中的位。 */
/* 定义宏，用于获取位图中指定位置的位 */
#define MAPBIT(map,n) ((map)[(n)/32]&(1u<<((n)%32)))
/* 定义宏，用于设置位图中指定位置的位 */
#define MAPSET(map,n) ((map)[(n)/32]|=(1u<<((n)%32)))

/* 用于序列化的 pcre2 代码的头部 */
typedef struct pcre2_serialized_data {
  uint32_t magic;           // 魔数
  uint32_t version;         // 版本号
  uint32_t config;          // 配置信息
  int32_t  number_of_codes; // 代码数量
} pcre2_serialized_data;

/* ----------------- 需要 PCRE2_CODE_UNIT_WIDTH 的项目 ----------------- */

/* 当该文件被 pcre2test 包含时，PCRE2_CODE_UNIT_WIDTH 被定义为 0，因此以下项目被省略 */

#if defined PCRE2_CODE_UNIT_WIDTH && PCRE2_CODE_UNIT_WIDTH != 0

/* 仅支持 8 位库的 EBCDIC */
#if defined EBCDIC && PCRE2_CODE_UNIT_WIDTH != 8
#error EBCDIC 不支持 16 位或 32 位库
#endif

/* 这是最大的非 UTF 码点 */
#define MAX_NON_UTF_CHAR (0xffffffffU >> (32 - PCRE2_CODE_UNIT_WIDTH))

/* 内部共享数据表和变量。这些被多个导出的公共函数使用。它们在 C 语言中必须是 "external"，但不是 PCRE2 公共 API 的一部分。尽管某些库的数据是相同的，但它们必须有不同的名称，以便可以同时链接多个库到单个应用程序。然而，只有在编译 8 位库时才需要 UTF-8 表。 */
#if PCRE2_CODE_UNIT_WIDTH == 8
extern const int              PRIV(utf8_table1)[];      // UTF-8 表 1
extern const int              PRIV(utf8_table1_size);   // UTF-8 表 1 大小
extern const int              PRIV(utf8_table2)[];      // UTF-8 表 2
extern const int              PRIV(utf8_table3)[];      // UTF-8 表 3
extern const uint8_t          PRIV(utf8_table4)[];      // UTF-8 表 4
#endif

#define _pcre2_OP_lengths              PCRE2_SUFFIX(_pcre2_OP_lengths_)              // 操作长度
#define _pcre2_callout_end_delims      PCRE2_SUFFIX(_pcre2_callout_end_delims_)      // 回调结束分隔符
#define _pcre2_callout_start_delims    PCRE2_SUFFIX(_pcre2_callout_start_delims_)    // 回调开始分隔符
#define _pcre2_default_compile_context PCRE2_SUFFIX(_pcre2_default_compile_context_) // 默认编译上下文
# 定义宏，用于设置默认的转换上下文、匹配上下文和表格
#define _pcre2_default_convert_context PCRE2_SUFFIX(_pcre2_default_convert_context_)
#define _pcre2_default_match_context   PCRE2_SUFFIX(_pcre2_default_match_context_)
#define _pcre2_default_tables          PCRE2_SUFFIX(_pcre2_default_tables_)
#if PCRE2_CODE_UNIT_WIDTH == 32
#define _pcre2_dummy_ucd_record        PCRE2_SUFFIX(_pcre2_dummy_ucd_record_)
#endif
#define _pcre2_hspace_list             PCRE2_SUFFIX(_pcre2_hspace_list_)
#define _pcre2_vspace_list             PCRE2_SUFFIX(_pcre2_vspace_list_)
#define _pcre2_ucd_boolprop_sets       PCRE2_SUFFIX(_pcre2_ucd_boolprop_sets_)
#define _pcre2_ucd_caseless_sets       PCRE2_SUFFIX(_pcre2_ucd_caseless_sets_)
#define _pcre2_ucd_digit_sets          PCRE2_SUFFIX(_pcre2_ucd_digit_sets_)
#define _pcre2_ucd_script_sets         PCRE2_SUFFIX(_pcre2_ucd_script_sets_)
#define _pcre2_ucd_records             PCRE2_SUFFIX(_pcre2_ucd_records_)
#define _pcre2_ucd_stage1              PCRE2_SUFFIX(_pcre2_ucd_stage1_)
#define _pcre2_ucd_stage2              PCRE2_SUFFIX(_pcre2_ucd_stage2_)
#define _pcre2_ucp_gbtable             PCRE2_SUFFIX(_pcre2_ucp_gbtable_)
#define _pcre2_ucp_gentype             PCRE2_SUFFIX(_pcre2_ucp_gentype_)
#define _pcre2_ucp_typerange           PCRE2_SUFFIX(_pcre2_ucp_typerange_)
#define _pcre2_unicode_version         PCRE2_SUFFIX(_pcre2_unicode_version_)
#define _pcre2_utt                     PCRE2_SUFFIX(_pcre2_utt_)
#define _pcre2_utt_names               PCRE2_SUFFIX(_pcre2_utt_names_)
#define _pcre2_utt_size                PCRE2_SUFFIX(_pcre2_utt_size_)

# 声明全局变量，用于存储长度、调用结束分隔符、调用开始分隔符、默认编译上下文、默认转换上下文和默认匹配上下文
extern const uint8_t                   PRIV(OP_lengths)[];
extern const uint32_t                  PRIV(callout_end_delims)[];
extern const uint32_t                  PRIV(callout_start_delims)[];
extern const pcre2_compile_context     PRIV(default_compile_context);
extern const pcre2_convert_context     PRIV(default_convert_context);
extern const pcre2_match_context       PRIV(default_match_context);
// 声明并定义了一系列常量数组，用于存储默认表、水平空格列表、垂直空格列表、UCD 布尔属性集、UCD 大小写无关集、UCD 数字集、UCD 脚本集、UCD 记录等数据
extern const uint8_t                   PRIV(default_tables)[];
extern const uint32_t                  PRIV(hspace_list)[];
extern const uint32_t                  PRIV(vspace_list)[];
extern const uint32_t                  PRIV(ucd_boolprop_sets)[];
extern const uint32_t                  PRIV(ucd_caseless_sets)[];
extern const uint32_t                  PRIV(ucd_digit_sets)[];
extern const uint32_t                  PRIV(ucd_script_sets)[];
extern const ucd_record                PRIV(ucd_records)[];
#if PCRE2_CODE_UNIT_WIDTH == 32
extern const ucd_record                PRIV(dummy_ucd_record)[];
#endif
extern const uint16_t                  PRIV(ucd_stage1)[];
extern const uint16_t                  PRIV(ucd_stage2)[];
extern const uint32_t                  PRIV(ucp_gbtable)[];
extern const uint32_t                  PRIV(ucp_gentype)[];
#ifdef SUPPORT_JIT
extern const int                       PRIV(ucp_typerange)[];
#endif
extern const char                     *PRIV(unicode_version);
extern const ucp_type_table            PRIV(utt)[];
extern const char                      PRIV(utt_names)[];
extern const size_t                    PRIV(utt_size);

// 在另一个文件中定义了一些模式相关的宏和隐藏的私有结构，根据 PCRE2_CODE_UNIT_WIDTH 的定义选择合适宽度的结构，并设置私有结构的后缀宏
#define branch_chain                 PCRE2_SUFFIX(branch_chain_)
#define compile_block                PCRE2_SUFFIX(compile_block_)
#define dfa_match_block              PCRE2_SUFFIX(dfa_match_block_)
#define match_block                  PCRE2_SUFFIX(match_block_)
#define named_group                  PCRE2_SUFFIX(named_group_)

// 包含了另一个文件 pcre2_intmodedep.h，该文件中定义了一些私有的“外部”函数，这些函数在内部被调用
#include "pcre2_intmodedep.h"
# 定义一系列以 _pcre2_ 开头的宏，用于在不同模块中引用定义在其他模块中的函数
# 这些宏在 PCRE2 的公共 API 中并不可见，是 C 语言中的“外部”定义
# 当没有可用的代码单元宽度时，这些宏不应该被定义
#define _pcre2_auto_possessify       PCRE2_SUFFIX(_pcre2_auto_possessify_)
#define _pcre2_check_escape          PCRE2_SUFFIX(_pcre2_check_escape_)
#define _pcre2_extuni                PCRE2_SUFFIX(_pcre2_extuni_)
#define _pcre2_find_bracket          PCRE2_SUFFIX(_pcre2_find_bracket_)
#define _pcre2_is_newline            PCRE2_SUFFIX(_pcre2_is_newline_)
#define _pcre2_jit_free_rodata       PCRE2_SUFFIX(_pcre2_jit_free_rodata_)
#define _pcre2_jit_free              PCRE2_SUFFIX(_pcre2_jit_free_)
#define _pcre2_jit_get_size          PCRE2_SUFFIX(_pcre2_jit_get_size_)
#define _pcre2_jit_get_target        PCRE2_SUFFIX(_pcre2_jit_get_target_)
#define _pcre2_memctl_malloc         PCRE2_SUFFIX(_pcre2_memctl_malloc_)
#define _pcre2_ord2utf               PCRE2_SUFFIX(_pcre2_ord2utf_)
#define _pcre2_script_run            PCRE2_SUFFIX(_pcre2_script_run_)
#define _pcre2_strcmp                PCRE2_SUFFIX(_pcre2_strcmp_)
#define _pcre2_strcmp_c8             PCRE2_SUFFIX(_pcre2_strcmp_c8_)
#define _pcre2_strcpy_c8             PCRE2_SUFFIX(_pcre2_strcpy_c8_)
#define _pcre2_strlen                PCRE2_SUFFIX(_pcre2_strlen_)
#define _pcre2_strncmp               PCRE2_SUFFIX(_pcre2_strncmp_)
#define _pcre2_strncmp_c8            PCRE2_SUFFIX(_pcre2_strncmp_c8_)
#define _pcre2_study                 PCRE2_SUFFIX(_pcre2_study_)
#define _pcre2_valid_utf             PCRE2_SUFFIX(_pcre2_valid_utf_)
#define _pcre2_was_newline           PCRE2_SUFFIX(_pcre2_was_newline_)
#define _pcre2_xclass                PCRE2_SUFFIX(_pcre2_xclass_)

# 声明一个名为 _pcre2_auto_possessify 的函数，接受 PCRE2_UCHAR 类型的指针和 compile_block 结构体指针作为参数，返回整型
extern int          _pcre2_auto_possessify(PCRE2_UCHAR *,
                      const compile_block *);
# 定义外部函数 _pcre2_check_escape，用于检查转义字符
extern int          _pcre2_check_escape(PCRE2_SPTR *, PCRE2_SPTR, uint32_t *,
                      int *, uint32_t, uint32_t, BOOL, compile_block *);
# 定义外部函数 _pcre2_extuni，用于处理扩展的 Unicode 字符
extern PCRE2_SPTR   _pcre2_extuni(uint32_t, PCRE2_SPTR, PCRE2_SPTR, PCRE2_SPTR,
                      BOOL, int *);
# 定义外部函数 _pcre2_find_bracket，用于查找括号
extern PCRE2_SPTR   _pcre2_find_bracket(PCRE2_SPTR, BOOL, int);
# 定义外部函数 _pcre2_is_newline，用于检查是否为换行符
extern BOOL         _pcre2_is_newline(PCRE2_SPTR, uint32_t, PCRE2_SPTR,
                      uint32_t *, BOOL);
# 定义外部函数 _pcre2_jit_free_rodata，用于释放只读数据的 JIT 编译
extern void         _pcre2_jit_free_rodata(void *, void *);
# 定义外部函数 _pcre2_jit_free，用于释放 JIT 编译
extern void         _pcre2_jit_free(void *, pcre2_memctl *);
# 定义外部函数 _pcre2_jit_get_size，用于获取 JIT 编译的大小
extern size_t       _pcre2_jit_get_size(void *);
# 定义外部函数 _pcre2_jit_get_target，用于获取 JIT 编译的目标
const char *        _pcre2_jit_get_target(void);
# 定义外部函数 _pcre2_memctl_malloc，用于在内存控制块中分配内存
extern void *       _pcre2_memctl_malloc(size_t, pcre2_memctl *);
# 定义外部函数 _pcre2_ord2utf，用于将 Unicode 编码转换为 UTF-8 编码
extern unsigned int _pcre2_ord2utf(uint32_t, PCRE2_UCHAR *);
# 定义外部函数 _pcre2_script_run，用于运行脚本
extern BOOL         _pcre2_script_run(PCRE2_SPTR, PCRE2_SPTR, BOOL);
# 定义外部函数 _pcre2_strcmp，用于比较字符串
extern int          _pcre2_strcmp(PCRE2_SPTR, PCRE2_SPTR);
# 定义外部函数 _pcre2_strcmp_c8，用于比较字符串和 C 字符串
extern int          _pcre2_strcmp_c8(PCRE2_SPTR, const char *);
# 定义外部函数 _pcre2_strcpy_c8，用于将 C 字符串复制到 UTF-8 编码的字符串
extern PCRE2_SIZE   _pcre2_strcpy_c8(PCRE2_UCHAR *, const char *);
# 定义外部函数 _pcre2_strlen，用于获取字符串的长度
extern PCRE2_SIZE   _pcre2_strlen(PCRE2_SPTR);
# 定义外部函数 _pcre2_strncmp，用于比较部分字符串
extern int          _pcre2_strncmp(PCRE2_SPTR, PCRE2_SPTR, size_t);
# 定义外部函数 _pcre2_strncmp_c8，用于比较部分字符串和 C 字符串
extern int          _pcre2_strncmp_c8(PCRE2_SPTR, const char *, size_t);
# 定义外部函数 _pcre2_study，用于对正则表达式进行优化
extern int          _pcre2_study(pcre2_real_code *);
# 定义外部函数 _pcre2_valid_utf，用于验证 UTF-8 编码的字符串
extern int          _pcre2_valid_utf(PCRE2_SPTR, PCRE2_SIZE, PCRE2_SIZE *);
# 定义外部函数 _pcre2_was_newline，用于检查之前是否为换行符
extern BOOL         _pcre2_was_newline(PCRE2_SPTR, uint32_t, PCRE2_SPTR,
                      uint32_t *, BOOL);
# 定义外部函数 _pcre2_xclass，用于处理字符类

/* 当 memmove() 不可用时，需要此函数 */
#if !defined(VPCOMPAT) && !defined(HAVE_MEMMOVE)
# 定义外部函数 _pcre2_memmove，用于移动内存块
#define _pcre2_memmove               PCRE2_SUFFIX(_pcre2_memmove)
extern void *       _pcre2_memmove(void *, const void *, size_t);
#endif

# 结束 pcre2_internal.h
#endif  /* PCRE2_CODE_UNIT_WIDTH */
#endif  /* PCRE2_INTERNAL_H_IDEMPOTENT_GUARD */
```