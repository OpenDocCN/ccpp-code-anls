# `nmap\libpcre\src\pcre2_error.c`

```
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
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

// 定义宏，将参数转换为字符串
#define STRING(a)  # a
#define XSTRING(s) STRING(s)

/* 编译时错误消息的文本。编译时错误号从 COMPILE_ERROR_BASE (100) 开始。

这里曾经是一个字符串表，但为了减少动态加载共享库时所需的重定位数量，现在是一个长字符串。我们不能使用偏移量表，因为诸如 XSTRING(MAX_NAME_SIZE) 这样的插入的长度是未知的。相反，pcre2_get_error_message() 通过计数来找到所需的字符串 - 这不是性能问题，因为这些字符串只在出现错误时使用。

每个子字符串以 \0 结尾以插入一个空字符。这包括最后的子字符串，因此整个字符串以 \0\0 结尾，可以在计数时检测到。

#ifndef EBCDIC
  "\\c must be followed by a printable ASCII character\0"
#else
  "\\c must be followed by a letter or one of [\\]^_?\0"
#endif
  "\\k is not followed by a braced, angle-bracketed, or quoted name\0"  // 错误消息：\k 后面没有跟着大括号、尖括号或引号的名称
  /* 70 */
  "internal error: unknown meta code in check_lookbehinds()\0"  // 内部错误：在 check_lookbehinds() 中未知的元代码
  "\\N is not supported in a class\0"  // 错误消息：在类中不支持 \N
  "callout string is too long\0"  // 错误消息：调用字符串太长
  "disallowed Unicode code point (>= 0xd800 && <= 0xdfff)\0"  // 错误消息：不允许的 Unicode 代码点 (>= 0xd800 && <= 0xdfff)
  "using UTF is disabled by the application\0"  // 错误消息：应用程序禁用了使用 UTF
  /* 75 */
  "using UCP is disabled by the application\0"  // 错误消息：应用程序禁用了使用 UCP
  "name is too long in (*MARK), (*PRUNE), (*SKIP), or (*THEN)\0"  // 错误消息：在 (*MARK), (*PRUNE), (*SKIP), 或 (*THEN) 中名称太长
  "character code point value in \\u.... sequence is too large\0"  // 错误消息：在 \\u.... 序列中的字符代码点值太大
  "digits missing in \\x{} or \\o{} or \\N{U+}\0"  // 错误消息：在 \\x{} 或 \\o{} 或 \\N{U+} 中缺少数字
  "syntax error or number too big in (?(VERSION condition\0"  // 错误消息：在 (?(VERSION condition 中的语法错误或数字太大
  /* 80 */
  "internal error: unknown opcode in auto_possessify()\0"  // 内部错误：在 auto_possessify() 中未知的操作码
  "missing terminating delimiter for callout with string argument\0"  // 错误消息：缺少调用带有字符串参数的终止定界符
  "unrecognized string delimiter follows (?C\0"  // 错误消息：在 (?C 后面跟着无法识别的字符串定界符
  "using \\C is disabled by the application\0"  // 错误消息：应用程序禁用了使用 \\C
  "(?| and/or (?J: or (?x: parentheses are too deeply nested\0"  // 错误消息：(?| 和/或 (?J: 或 (?x: 括号嵌套太深
  /* 85 */
  "using \\C is disabled in this PCRE2 library\0"  // 错误消息：在此 PCRE2 库中禁用了使用 \\C
  "regular expression is too complicated\0"  // 错误消息：正则表达式太复杂
  "lookbehind assertion is too long\0"  // 错误消息：回顾断言太长
  "pattern string is longer than the limit set by the application\0"  // 错误消息：模式字符串比应用程序设置的限制更长
  "internal error: unknown code in parsed pattern\0"  // 内部错误：在解析模式中未知的代码
  /* 90 */
  "internal error: bad code value in parsed_skip()\0"  // 内部错误：在 parsed_skip() 中的坏代码值
  "PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES is not allowed in UTF-16 mode\0"  // 错误消息：在 UTF-16 模式下不允许使用 PCRE2_EXTRA_ALLOW_SURROGATE_ESCAPES
  "invalid option bits with PCRE2_LITERAL\0"  // 错误消息：在 PCRE2_LITERAL 中使用了无效的选项位
  "\\N{U+dddd} is supported only in Unicode (UTF) mode\0"  // 错误消息：\\N{U+dddd} 仅在 Unicode (UTF) 模式下支持
  "invalid hyphen in option setting\0"  // 错误消息：选项设置中的连字符无效
  /* 95 */
  "(*alpha_assertion) not recognized\0"  // 错误消息：(*alpha_assertion) 未被识别
  "script runs require Unicode support, which this version of PCRE2 does not have\0"  // 错误消息：脚本运行需要 Unicode 支持，而此版本的 PCRE2 没有
  "too many capturing groups (maximum 65535)\0"  // 错误消息：捕获组太多 (最大 65535)
  "atomic assertion expected after (?( or (?(?C)\0"  // 错误消息：在 (?( 或 (?(?C) 后面期望原子断言
  "\\K is not allowed in lookarounds (but see PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK)\0"  // 错误消息：在环视中不允许使用 \K (但请参阅 PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK)
  ;

/* Match-time and UTF error texts are in the same format. */

/*************************************************
*            Return error message                *
/*************************************************/

/* This function copies an error message into a buffer whose units are of an
appropriate width. Error numbers are positive for compile-time errors, and
negative for match-time errors (except for UTF errors), but the numbers are all
distinct.

Arguments:
  enumber       error number
  buffer        where to put the message (zero terminated)
  size          size of the buffer in code units

Returns:        length of message if all is well
                negative on error
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_get_error_message(int enumber, PCRE2_UCHAR *buffer, PCRE2_SIZE size)
{
const unsigned char *message;  /* Pointer to the error message */
PCRE2_SIZE i;  /* Counter for the message length */
int n;  /* Counter for the error number */

if (size == 0) return PCRE2_ERROR_NOMEMORY;  /* Return error if the buffer size is 0 */

if (enumber >= COMPILE_ERROR_BASE)  /* Check if it's a compile error */
  {
  message = compile_error_texts;  /* Set the message pointer to the compile error texts */
  n = enumber - COMPILE_ERROR_BASE;  /* Calculate the error number for compile errors */
  }
else if (enumber < 0)               /* Check if it's a match or UTF error */
  {
  message = match_error_texts;  /* Set the message pointer to the match error texts */
  n = -enumber;  /* Calculate the error number for match or UTF errors */
  }
else                                /* Invalid error number */
  {
  message = (unsigned char *)"\0";  /* Set the message pointer to an empty message list */
  n = 1;  /* Set the error number to 1 */
  }

for (; n > 0; n--)  /* Loop through the error messages */
  {
  while (*message++ != CHAR_NUL) {};  /* Move the message pointer to the end of the message */
  if (*message == CHAR_NUL) return PCRE2_ERROR_BADDATA;  /* Return error if the message is empty */
  }

for (i = 0; *message != 0; i++)  /* Loop through the characters of the error message */
  {
  if (i >= size - 1)  /* Check if the buffer is full */
    {
    buffer[i] = 0;     /* Terminate partial message */
    return PCRE2_ERROR_NOMEMORY;  /* Return error if the buffer is full */
    }
  buffer[i] = *message++;  /* Copy the character to the buffer */
  }

buffer[i] = 0;  /* Terminate the message with zero */
return (int)i;  /* Return the length of the message */
}

/* End of pcre2_error.c */
```