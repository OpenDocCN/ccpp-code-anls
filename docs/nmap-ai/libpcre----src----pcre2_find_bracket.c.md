# `nmap\libpcre\src\pcre2_find_bracket.c`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2018 University of Cambridge

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

# 如果定义了 HAVE_CONFIG_H，则包含 config.h 文件


#include "pcre2_internal.h"

# 包含 pcre2_internal.h 文件


PCRE2_SPTR
PRIV(find_bracket)(PCRE2_SPTR code, BOOL utf, int number)

# 定义名为 PRIV(find_bracket) 的函数，参数为 code、utf 和 number


for (;;)

# 无限循环


{
  PCRE2_UCHAR c = *code;

# 定义变量 c，赋值为指针 code 所指向的值


if (c == OP_END) return NULL;

# 如果 c 的值等于 OP_END，则返回 NULL


if (c == OP_XCLASS) code += GET(code, 1);
    else if (c == OP_CALLOUT_STR) code += GET(code, 1 + 2*LINK_SIZE);

# 如果 c 的值等于 OP_XCLASS，则将 code 的值增加 GET(code, 1)；否则如果 c 的值等于 OP_CALLOUT_STR，则将 code 的值增加 GET(code, 1 + 2*LINK_SIZE)


else if (c == OP_REVERSE)
    {
    if (number < 0) return (PCRE2_UCHAR *)code;
    code += PRIV(OP_lengths)[c];
    }

# 否则如果 c 的值等于 OP_REVERSE，则如果 number 小于 0，则返回 code 的值转换为 PCRE2_UCHAR 类型；否则将 code 的值增加 PRIV(OP_lengths)[c]


else if (c == OP_CBRA || c == OP_SCBRA ||
           c == OP_CBRAPOS || c == OP_SCBRAPOS)
    {
    int n = (int)GET2(code, 1+LINK_SIZE);
    if (n == number) return (PCRE2_UCHAR *)code;
    code += PRIV(OP_lengths)[c];

# 否则如果 c 的值等于 OP_CBRA 或者 c 的值等于 OP_SCBRA 或者 c 的值等于 OP_CBRAPOS 或者 c 的值等于 OP_SCBRAPOS，则定义变量 n，赋值为 GET2(code, 1+LINK_SIZE) 的值转换为 int 类型；如果 n 等于 number，则返回 code 的值转换为 PCRE2_UCHAR 类型；否则将 code 的值增加 PRIV(OP_lengths)[c] 的值
    }

  /* 否则，我们可以从表中获取项目的长度，除了重复字符类型外，我们必须测试 \p 和 \P，它们有额外的两个字节参数，以及带有参数的 MARK/PRUNE/SKIP/THEN，我们必须加上它的长度。 */

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

      case OP_TYPEUPTO:
      case OP_TYPEMINUPTO:
      case OP_TYPEEXACT:
      case OP_TYPEPOSUPTO:
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

    /* 从表中添加固定长度 */

    code += PRIV(OP_lengths)[c];

  /* 在 UTF-8 和 UTF-16 模式下，后面跟着字符的操作码可能后面跟着多字节字符。表中的长度是最小值，所以我们必须安排跳过额外的字节。 */
#ifdef MAYBE_UTF_MULTI
    # 如果定义了 MAYBE_UTF_MULTI，则根据 utf 变量的值进行条件判断
    if (utf) switch(c)
      {
      # 根据不同的操作码进行条件判断
      case OP_CHAR:
      case OP_CHARI:
      case OP_NOT:
      ...
      # 根据操作码的额外长度，更新 code 指针位置
      if (HAS_EXTRALEN(code[-1])) code += GET_EXTRALEN(code[-1]);
      break;
      }
#else
    # 如果未定义 MAYBE_UTF_MULTI，则保持编译器对函数参数的引用，以避免警告
    (void)(utf);  /* Keep compiler happy by referencing function argument */
#endif  /* MAYBE_UTF_MULTI */
    }
  }
}

/* End of pcre2_find_bracket.c */
```