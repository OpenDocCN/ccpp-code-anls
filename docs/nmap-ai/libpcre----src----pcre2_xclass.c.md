# `nmap\libpcre\src\pcre2_xclass.c`

```cpp
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
/* This module contains an internal function that is used to match an extended
class. It is used by pcre2_auto_possessify() and by both pcre2_match() and
pcre2_def_match(). */
/* 这个模块包含一个用于匹配扩展类的内部函数。它被 pcre2_auto_possessify() 以及 pcre2_match() 和 pcre2_def_match() 使用。 */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
/* 如果有 config.h 文件，则包含它 */

#include "pcre2_internal.h"
/* 包含 pcre2_internal.h 文件 */

/*************************************************
*       Match character against an XCLASS        *
*************************************************/
/* 匹配字符与 XCLASS */

/* This function is called to match a character against an extended class that
might contain codepoints above 255 and/or Unicode properties.

Arguments:
  c           the character
  data        points to the flag code unit of the XCLASS data
  utf         TRUE if in UTF mode

Returns:      TRUE if character matches, else FALSE
*/
/* 这个函数用于匹配一个字符与可能包含大于 255 的码点和/或 Unicode 属性的扩展类。

参数：
  c           字符
  data        指向 XCLASS 数据的标志代码单元
  utf         如果处于 UTF 模式，则为 TRUE

返回值：如果字符匹配，则为 TRUE，否则为 FALSE */
BOOL
PRIV(xclass)(uint32_t c, PCRE2_SPTR data, BOOL utf)
{
PCRE2_UCHAR t;
BOOL negated = (*data & XCL_NOT) != 0;

#if PCRE2_CODE_UNIT_WIDTH == 8
/* In 8 bit mode, this must always be TRUE. Help the compiler to know that. */
utf = TRUE;
#endif
/* 在 8 位模式下，这必须始终为 TRUE。帮助编译器知道这一点。 */

/* Code points < 256 are matched against a bitmap, if one is present. If not,
we still carry on, because there may be ranges that start below 256 in the
additional data. */
/* 小于 256 的码点与位图匹配，如果存在的话。如果没有，我们仍然继续，因为在附加数据中可能有小于 256 的范围。 */
if (c < 256)
  {
  if ((*data & XCL_HASPROP) == 0)
    {
    if ((*data & XCL_MAP) == 0) return negated;
    return (((uint8_t *)(data + 1))[c/8] & (1u << (c&7))) != 0;
    }
  if ((*data & XCL_MAP) != 0 &&
    (((uint8_t *)(data + 1))[c/8] & (1u << (c&7))) != 0)
    return !negated; /* char found */
  }
/* 如果字符小于 256，则根据位图进行匹配。如果没有，返回 negated。 */

/* First skip the bit map if present. Then match against the list of Unicode
properties or large chars or ranges that end with a large char. We won't ever
encounter XCL_PROP or XCL_NOTPROP when UTF support is not compiled. */
/* 首先跳过位图（如果存在）。然后与 Unicode 属性列表或大字符或以大字符结尾的范围进行匹配。当没有编译 UTF 支持时，我们永远不会遇到 XCL_PROP 或 XCL_NOTPROP。 */
if ((*data++ & XCL_MAP) != 0) data += 32 / sizeof(PCRE2_UCHAR);
while ((t = *data++) != XCL_END)
  {
  // 当数据指针指向的值不等于 XCL_END 时执行循环
  uint32_t x, y;
  // 定义两个无符号整型变量 x 和 y
  if (t == XCL_SINGLE)
    {
    // 如果当前值等于 XCL_SINGLE
#ifdef SUPPORT_UNICODE
    if (utf)
      {
      // 如果支持 Unicode 并且 utf 为真
      GETCHARINC(x, data); /* macro generates multiple statements */
      // 调用 GETCHARINC 宏，生成多个语句
      }
    else
#endif
    x = *data++;
    // 否则将当前数据指针指向的值赋给 x，并将数据指针向后移动一位
    if (c == x) return !negated;
    // 如果 c 等于 x，则返回非 negated 的值
    }
  else if (t == XCL_RANGE)
    {
    // 否则如果当前值等于 XCL_RANGE
#ifdef SUPPORT_UNICODE
    if (utf)
      {
      // 如果支持 Unicode 并且 utf 为真
      GETCHARINC(x, data); /* macro generates multiple statements */
      // 调用 GETCHARINC 宏，生成多个语句
      GETCHARINC(y, data); /* macro generates multiple statements */
      // 再次调用 GETCHARINC 宏，生成多个语句
      }
    else
#endif
      {
      x = *data++;
      // 否则将当前数据指针指向的值赋给 x，并将数据指针向后移动一位
      y = *data++;
      // 将下一个数据指针指向的值赋给 y，并将数据指针向后移动一位
      }
    if (c >= x && c <= y) return !negated;
    // 如果 c 大于等于 x 并且小于等于 y，则返回非 negated 的值
    }

#ifdef SUPPORT_UNICODE
  else  /* XCL_PROP & XCL_NOTPROP */
    {
    // 否则如果当前值为 XCL_PROP 或 XCL_NOTPROP
    const ucd_record *prop = GET_UCD(c);
    // 定义一个指向 ucd_record 结构体的指针 prop，指向 GET_UCD(c) 的返回值
    BOOL isprop = t == XCL_PROP;
    // 定义一个布尔型变量 isprop，判断 t 是否等于 XCL_PROP
    BOOL ok;
    // 定义一个布尔型变量 ok

    data += 2;
    // 数据指针向后移动两位
    }
#else
  (void)utf;  /* Avoid compiler warning */
  // 否则忽略 utf，避免编译器警告
#endif  /* SUPPORT_UNICODE */
  }

return negated;   /* char did not match */
// 返回 negated，表示字符不匹配
}

/* End of pcre2_xclass.c */
// pcre2_xclass.c 结束
```