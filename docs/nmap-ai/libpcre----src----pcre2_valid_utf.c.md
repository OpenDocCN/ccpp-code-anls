# `nmap\libpcre\src\pcre2_valid_utf.c`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2020 University of Cambridge

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
   如果使用了这个软件，即使被告知可能会发生损害，也不会因任何原因对任何人负责。
   -----------------------------------------------------------------------------
*/

/* 
   这个模块包含一个用于验证 UTF 字符串的内部函数。这个文件也被 pcre2test 程序 #include，该程序使用宏将 _pcre2_xxx 的名称更改为 xxxx，从而避免与库的名称冲突。在这种情况下，定义了 PCRE2_PCRE2TEST。
*/

#ifndef PCRE2_PCRE2TEST           /* 我们正在编译库 */
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
#include "pcre2_internal.h"
#endif /* PCRE2_PCRE2TEST */


#ifndef SUPPORT_UNICODE
/*************************************************
*  当不支持 Unicode 时的虚拟函数  *
*************************************************/

/* 当不支持 Unicode 时，这个函数不应该被调用。 */

int
PRIV(valid_utf)(PCRE2_SPTR string, PCRE2_SIZE length, PCRE2_SIZE *erroroffset)
{
(void)string;
(void)length;
(void)erroroffset;
return 0;
}
#else  /* 支持 UTF */



/*************************************************
*           验证一个 UTF 字符串                *
*************************************************/

/* 这个函数在编译或匹配开始时（可选地）被调用，以检查一个假定的 UTF 字符串是否实际上是有效的。早期的检查意味着随后的代码可以假定它正在处理一个有效的字符串。可以关闭检查以获得最大性能，但是提供无效字符串的后果是未定义的。

参数：
  string       指向字符串
  length       字符串的长度
  errp         指向错误位置偏移变量的指针

返回值：       == 0    如果字符串是有效的 UTF 字符串
               != 0    否则，设置坏字符的偏移量
*/

int
PRIV(valid_utf)(PCRE2_SPTR string, PCRE2_SIZE length, PCRE2_SIZE *erroroffset)
{
PCRE2_SPTR p;
uint32_t c;
/* ----------------- 检查一个 UTF-8 字符串 ----------------- */

#if PCRE2_CODE_UNIT_WIDTH == 8

/* 最初，此函数根据 RFC 2279 进行检查，允许值在范围 0 到 0x7fffffff 之间，最长为 6 个字节，但确保它们处于规范格式。
一旦有人向我指出了 RFC 3629（它取代了 2279），我就增加了额外的限制。现在这些值被限制在 0 到 0x0010ffff 之间，最长不超过 4 个字节，并且排除了子范围 0xd000 到 0xdfff。
然而，仍然检查了 5 字节和 6 字节字符的格式。错误返回如下：

PCRE2_ERROR_UTF8_ERR1   字符串末尾缺少 1 个字节
PCRE2_ERROR_UTF8_ERR2   字符串末尾缺少 2 个字节
PCRE2_ERROR_UTF8_ERR3   字符串末尾缺少 3 个字节
PCRE2_ERROR_UTF8_ERR4   字符串末尾缺少 4 个字节
PCRE2_ERROR_UTF8_ERR5   字符串末尾缺少 5 个字节
PCRE2_ERROR_UTF8_ERR6   第二个字节的前两位不是 0x80
PCRE2_ERROR_UTF8_ERR7   第三个字节的前两位不是 0x80
PCRE2_ERROR_UTF8_ERR8   第四个字节的前两位不是 0x80
PCRE2_ERROR_UTF8_ERR9   第五个字节的前两位不是 0x80
PCRE2_ERROR_UTF8_ERR10  第六个字节的前两位不是 0x80
PCRE2_ERROR_UTF8_ERR11  RFC 3629 不允许 5 字节字符
PCRE2_ERROR_UTF8_ERR12  RFC 3629 不允许 6 字节字符
PCRE2_ERROR_UTF8_ERR13  值大于 0x10ffff 的 4 字节字符不被允许
PCRE2_ERROR_UTF8_ERR14  值在 0xd800-0xdfff 范围内的 3 字节字符不被允许
PCRE2_ERROR_UTF8_ERR15  过长的 2 字节序列
PCRE2_ERROR_UTF8_ERR16  过长的 3 字节序列
PCRE2_ERROR_UTF8_ERR17  过长的 4 字节序列
PCRE2_ERROR_UTF8_ERR18  过长的 5 字节序列（永远不会发生）
PCRE2_ERROR_UTF8_ERR19  过长的 6 字节序列（永远不会发生）
PCRE2_ERROR_UTF8_ERR20  孤立的 0x80 字节（不在 UTF-8 字符内）
PCRE2_ERROR_UTF8_ERR21  具有非法值 0xfe 或 0xff 的字节
*/
# 遍历字符串，逐个处理字符
for (p = string; length > 0; p++)
  {
  # 定义变量 ab 和 d
  uint32_t ab, d;

  # 获取当前字符
  c = *p;
  # 减少剩余长度
  length--;

  # 如果字符小于 128，则为 ASCII 字符，跳过
  if (c < 128) continue;                /* ASCII character */

  # 如果字符小于 0xc0，则为孤立的 10xx xxxx 字节，返回错误
  if (c < 0xc0)                         
    {
    *erroroffset = (PCRE2_SIZE)(p - string);
    return PCRE2_ERROR_UTF8_ERR20;
    }

  # 如果字符大于等于 0xfe，则为无效的 0xfe 或 0xff 字节，返回错误
  if (c >= 0xfe)                        
    {
    *erroroffset = (PCRE2_SIZE)(p - string);
    return PCRE2_ERROR_UTF8_ERR21;
    }

  # 获取附加字节数（1-5）
  ab = PRIV(utf8_table4)[c & 0x3f];     
  # 如果剩余长度小于附加字节数，返回错误
  if (length < ab)                      
    {
    *erroroffset = (PCRE2_SIZE)(p - string);
    switch(ab - length)
      {
      case 1: return PCRE2_ERROR_UTF8_ERR1;
      case 2: return PCRE2_ERROR_UTF8_ERR2;
      case 3: return PCRE2_ERROR_UTF8_ERR3;
      case 4: return PCRE2_ERROR_UTF8_ERR4;
      case 5: return PCRE2_ERROR_UTF8_ERR5;
      }
    }
  # 减少剩余长度
  length -= ab;                         

  # 检查第二个字节的高位
  if (((d = *(++p)) & 0xc0) != 0x80)
    {
    *erroroffset = (int)(p - string) - 1;
    return PCRE2_ERROR_UTF8_ERR6;
    }

  # 根据附加字节数进行不同的检查
  switch (ab)
    {
    # 2 字节字符。无需进一步检查 0x80。检查第一个字节是否为 xx00 000x（过长序列）
    case 1: if ((c & 0x3e) == 0)
      {
      *erroroffset = (int)(p - string) - 1;
      return PCRE2_ERROR_UTF8_ERR15;
      }
    break;

    # 3 字节字符。检查第三个字节是否为 0x80。然后检查前两个字节是否为 1110 0000, xx0x xxxx（过长序列）或 1110 1101, 1010 xxxx（0xd800 - 0xdfff）
    case 2:
    if ((*(++p) & 0xc0) != 0x80)     /* Third byte */
      {
      *erroroffset = (int)(p - string) - 2;
      return PCRE2_ERROR_UTF8_ERR7;
      }
    if (c == 0xe0 && (d & 0x20) == 0)
      {
      // 如果字符是 3 字节的 UTF-8 编码，并且第四位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 2;
      return PCRE2_ERROR_UTF8_ERR16;
      }
    if (c == 0xed && d >= 0xa0)
      {
      // 如果字符是 3 字节的 UTF-8 编码，并且第四位大于等于 0xa0，则返回错误
      *erroroffset = (int)(p - string) - 2;
      return PCRE2_ERROR_UTF8_ERR14;
      }
    break;

    /* 4-byte character. Check 3rd and 4th bytes for 0x80. Then check first 2
       bytes for for 1111 0000, xx00 xxxx (overlong sequence), then check for a
       character greater than 0x0010ffff (f4 8f bf bf) */

    case 3:
    if ((*(++p) & 0xc0) != 0x80)     /* Third byte */
      {
      // 如果字符是 4 字节的 UTF-8 编码，并且第三位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 2;
      return PCRE2_ERROR_UTF8_ERR7;
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Fourth byte */
      {
      // 如果字符是 4 字节的 UTF-8 编码，并且第四位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 3;
      return PCRE2_ERROR_UTF8_ERR8;
      }
    if (c == 0xf0 && (d & 0x30) == 0)
      {
      // 如果字符是 4 字节的 UTF-8 编码，并且第一个字节的后两位不是 00，则返回错误
      *erroroffset = (int)(p - string) - 3;
      return PCRE2_ERROR_UTF8_ERR17;
      }
    if (c > 0xf4 || (c == 0xf4 && d > 0x8f))
      {
      // 如果字符的编码大于 0x10ffff，则返回错误
      *erroroffset = (int)(p - string) - 3;
      return PCRE2_ERROR_UTF8_ERR13;
      }
    break;

    /* 5-byte and 6-byte characters are not allowed by RFC 3629, and will be
    rejected by the length test below. However, we do the appropriate tests
    here so that overlong sequences get diagnosed, and also in case there is
    ever an option for handling these larger code points. */

    /* 5-byte character. Check 3rd, 4th, and 5th bytes for 0x80. Then check for
    1111 1000, xx00 0xxx */

    case 4:
    if ((*(++p) & 0xc0) != 0x80)     /* Third byte */
      {
      // 如果字符是 5 字节的 UTF-8 编码，并且第三位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 2;
      return PCRE2_ERROR_UTF8_ERR7;
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Fourth byte */
      {
      // 如果字符是 5 字节的 UTF-8 编码，并且第四位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 3;
      return PCRE2_ERROR_UTF8_ERR8;
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Fifth byte */
      {
      // 如果字符是 5 字节的 UTF-8 编码，并且第五位不是 10，则返回错误
      *erroroffset = (int)(p - string) - 4;
      return PCRE2_ERROR_UTF8_ERR9;
      }
    if (c == 0xf8 && (d & 0x38) == 0)
      {
      *erroroffset = (int)(p - string) - 4;  // 设置错误偏移量为当前位置减去字符串起始位置再减4
      return PCRE2_ERROR_UTF8_ERR18;  // 返回 UTF-8 错误码 18
      }
    break;

    /* 6-byte character. Check 3rd-6th bytes for 0x80. Then check for
    1111 1100, xx00 00xx. */

    case 5:
    if ((*(++p) & 0xc0) != 0x80)     /* Third byte */  // 检查第三个字节是否符合 UTF-8 编码规范
      {
      *erroroffset = (int)(p - string) - 2;  // 设置错误偏移量为当前位置减去字符串起始位置再减2
      return PCRE2_ERROR_UTF8_ERR7;  // 返回 UTF-8 错误码 7
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Fourth byte */  // 检查第四个字节是否符合 UTF-8 编码规范
      {
      *erroroffset = (int)(p - string) - 3;  // 设置错误偏移量为当前位置减去字符串起始位置再减3
      return PCRE2_ERROR_UTF8_ERR8;  // 返回 UTF-8 错误码 8
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Fifth byte */  // 检查第五个字节是否符合 UTF-8 编码规范
      {
      *erroroffset = (int)(p - string) - 4;  // 设置错误偏移量为当前位置减去字符串起始位置再减4
      return PCRE2_ERROR_UTF8_ERR9;  // 返回 UTF-8 错误码 9
      }
    if ((*(++p) & 0xc0) != 0x80)     /* Sixth byte */  // 检查第六个字节是否符合 UTF-8 编码规范
      {
      *erroroffset = (int)(p - string) - 5;  // 设置错误偏移量为当前位置减去字符串起始位置再减5
      return PCRE2_ERROR_UTF8_ERR10;  // 返回 UTF-8 错误码 10
      }
    if (c == 0xfc && (d & 0x3c) == 0)
      {
      *erroroffset = (int)(p - string) - 5;  // 设置错误偏移量为当前位置减去字符串起始位置再减5
      return PCRE2_ERROR_UTF8_ERR19;  // 返回 UTF-8 错误码 19
      }
    break;
    }

  /* Character is valid under RFC 2279, but 4-byte and 5-byte characters are
  excluded by RFC 3629. The pointer p is currently at the last byte of the
  character. */

  if (ab > 3)
    {
    *erroroffset = (int)(p - string) - ab;  // 设置错误偏移量为当前位置减去字符串起始位置再减ab
    return (ab == 4)? PCRE2_ERROR_UTF8_ERR11 : PCRE2_ERROR_UTF8_ERR12;  // 如果ab等于4，返回 UTF-8 错误码11，否则返回 UTF-8 错误码12
    }
  }
# 返回值为0
return 0;


/* ----------------- Check a UTF-16 string ----------------- */

#elif PCRE2_CODE_UNIT_WIDTH == 16

# 对于 UTF-16 字符串，工作量和错误数量都不多。
# PCRE2_ERROR_UTF16_ERR1  字符串末尾缺少低代理项
# PCRE2_ERROR_UTF16_ERR2  无效的低代理项
# PCRE2_ERROR_UTF16_ERR3  孤立的低代理项

# 遍历 UTF-16 字符串
for (p = string; length > 0; p++)
  {
  c = *p;
  length--;

  # 普通的 UTF-16 代码点，既不是高代理项也不是低代理项
  if ((c & 0xf800) != 0xd800)
    {
    /* Normal UTF-16 code point. Neither high nor low surrogate. */
    }
  # 高代理项，必须后面跟着一个低代理项
  else if ((c & 0x0400) == 0)
    {
    if (length == 0)
      {
      *erroroffset = p - string;
      return PCRE2_ERROR_UTF16_ERR1;
      }
    p++;
    length--;
    if ((*p & 0xfc00) != 0xdc00)
      {
      *erroroffset = p - string - 1;
      return PCRE2_ERROR_UTF16_ERR2;
      }
    }
  else
    {
    # 孤立的低代理项，总是错误
    *erroroffset = p - string;
    return PCRE2_ERROR_UTF16_ERR3;
    }
  }
return 0;



/* ----------------- Check a UTF-32 string ----------------- */

#else

# 对于 UTF-32 字符串，几乎没有什么要做的。
# PCRE2_ERROR_UTF32_ERR1  代理字符
# PCRE2_ERROR_UTF32_ERR2  字符 > 0x10ffff

# 遍历 UTF-32 字符串
for (p = string; length > 0; length--, p++)
  {
  c = *p;
  # 普通的 UTF-32 代码点，既不是高代理项也不是低代理项
  if ((c & 0xfffff800u) != 0xd800u)
    {
    /* Normal UTF-32 code point. Neither high nor low surrogate. */
    if (c > 0x10ffffu)
      {
      *erroroffset = p - string;
      return PCRE2_ERROR_UTF32_ERR2;
      }
    }
  else
    {
    # 代理项
    *erroroffset = p - string;
    return PCRE2_ERROR_UTF32_ERR1;
    }
  }
return 0;
#endif  /* CODE_UNIT_WIDTH */
}
#endif  /* SUPPORT_UNICODE */

/* End of pcre2_valid_utf.c */
```