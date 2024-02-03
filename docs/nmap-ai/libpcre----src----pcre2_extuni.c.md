# `nmap\libpcre\src\pcre2_extuni.c`

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
/* 
   如果没有配置 Unicode 支持，则提供一个虚拟函数，以避免一些编译器不喜欢没有函数的源文件
*/
#ifndef SUPPORT_UNICODE
PCRE2_SPTR
PRIV(extuni)(uint32_t c, PCRE2_SPTR eptr, PCRE2_SPTR start_subject,
  PCRE2_SPTR end_subject, BOOL utf, int *xcount)
{
(void)c;
(void)eptr;
(void)start_subject;
(void)end_subject;
(void)utf;
(void)xcount;
return NULL;
}
#else

/*************************************************
*      匹配扩展的 Unicode 字符序列              *
*************************************************/

/*
参数：
  c              第一个字符
  eptr           指向下一个字符的指针
  start_subject  主题的起始指针
  end_subject    主题的结束指针
  utf            如果处于 UTF 模式，则为 TRUE
  xcount         指向额外字符计数的指针，如果不需要计数则为 NULL

返回值：         序列结束后的指针
*/

PCRE2_SPTR
PRIV(extuni)(uint32_t c, PCRE2_SPTR eptr, PCRE2_SPTR start_subject,
  PCRE2_SPTR end_subject, BOOL utf, int *xcount)
{
// 获取字符的 Unicode 通用断字符属性
int lgb = UCD_GRAPHBREAK(c);
# 当指针 eptr 小于 end_subject 时执行循环
while (eptr < end_subject)
  {
  # 定义变量 rgb，len 初始化为 1
  int rgb;
  int len = 1;
  # 如果不是 UTF 编码，则将字符赋值给 c；否则使用 GETCHARLEN 函数获取字符和长度
  if (!utf) c = *eptr; else { GETCHARLEN(c, eptr, len); }
  # 获取字符 c 的图形断点属性
  rgb = UCD_GRAPHBREAK(c);
  # 如果字符 c 的图形断点属性不在 lgb 对应的位上，则跳出循环
  if ((PRIV(ucp_gbtable)[lgb] & (1u << rgb)) == 0) break;

  /* 在区域指示符之间不允许断开，除非前面有偶数个区域指示符。 */

  # 如果 lgb 和 rgb 都是区域指示符
  if (lgb == ucp_gbRegional_Indicator && rgb == ucp_gbRegional_Indicator)
    {
    # 初始化区域指示符计数为 0，bptr 指向 eptr 左边的字符
    int ricount = 0;
    PCRE2_SPTR bptr = eptr - 1;
    if (utf) BACKCHAR(bptr);

    /* bptr is pointing to the left-hand character */

    # 当 bptr 大于 start_subject 时执行循环
    while (bptr > start_subject)
      {
      # bptr 向左移动一个字符
      bptr--;
      # 如果是 UTF 编码，则继续向左移动一个字符并获取字符 c；否则直接获取字符 c
      if (utf)
        {
        BACKCHAR(bptr);
        GETCHAR(c, bptr);
        }
      else
      c = *bptr;
      # 如果字符 c 的图形断点属性不是区域指示符，则跳出循环
      if (UCD_GRAPHBREAK(c) != ucp_gbRegional_Indicator) break;
      # 区域指示符计数加一
      ricount++;
      }
    # 如果区域指示符计数是奇数，则跳出循环，需要进行图形断点
    if ((ricount & 1) != 0) break;  /* Grapheme break required */
    }

  /* 如果 Extend 或 ZWJ 跟在 Extended_Pictographic 后面，则不更新 lgb；这允许在后面的 Extended_Pictographic 之前有任意数量的 Extend 或 ZWJ。 */

  # 如果 rgb 不是 Extend 或 ZWJ，或者 lgb 不是 Extended_Pictographic，则更新 lgb 为 rgb
  if ((rgb != ucp_gbExtend && rgb != ucp_gbZWJ) ||
       lgb != ucp_gbExtended_Pictographic)
    lgb = rgb;

  # eptr 向前移动 len 个位置
  eptr += len;
  # 如果 xcount 不为空，则将其值加一
  if (xcount != NULL) *xcount += 1;
  }

# 返回 eptr
return eptr;
}

# 结束 SUPPORT_UNICODE 部分

/* End of pcre2_extuni.c */
```