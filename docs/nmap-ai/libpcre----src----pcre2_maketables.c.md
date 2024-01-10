# `nmap\libpcre\src\pcre2_maketables.c`

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
   该模块包含外部函数 pcre2_maketables()，用于在当前区域设置下构建 PCRE2 的字符表。
   该文件作为 PCRE2 库的一部分单独编译。它也作为 pcre2_dftables.c 编译的一部分，作为一个独立程序包含在其中，此时宏 PCRE2_DFTABLES 被定义。
*/

#ifndef PCRE2_DFTABLES    /* 编译库 */
#  ifdef HAVE_CONFIG_H
#  include "config.h"
#  endif
#  include "pcre2_internal.h"
#endif

/*************************************************
*           创建 PCRE2 字符表                    *
*************************************************/

/* 
   该函数构建一组字符表，供 PCRE2 使用，并返回指向它们的指针。它们是使用 ctype 函数构建的，因此它们的内容将取决于当前的区域设置。
   当作为库的一部分编译时，如果提供了通用上下文 malloc，则通过通用上下文 malloc 获取存储空间，但是当定义了 PCRE2_DFTABLES（编译 pcre2_dftables 辅助程序时）时，将使用 malloc()，并且该函数具有不同的名称，以避免与 pcre2.h 中的原型冲突。

   参数：如果定义了 PCRE2_DFTABLES，则为无；否则为 PCRE2 通用上下文或 NULL
   返回：指向连续数据块的指针；如果内存分配失败，则为 NULL
*/

#ifdef PCRE2_DFTABLES  /* 包含在独立的 pcre2_dftables 程序中 */
static const uint8_t *maketables(void)
{
uint8_t *yield = (uint8_t *)malloc(TABLES_LENGTH);

#else  /* 非 PCRE2_DFTABLES，即编译库 */
PCRE2_EXP_DEFN const uint8_t * PCRE2_CALL_CONVENTION
pcre2_maketables(pcre2_general_context *gcontext)
{
// 申明一个指向 uint8_t 类型的指针变量 yield
uint8_t *yield = (uint8_t *)((gcontext != NULL)?
  // 如果 gcontext 不为空，则使用 gcontext 的内存分配函数分配 TABLES_LENGTH 大小的内存
  gcontext->memctl.malloc(TABLES_LENGTH, gcontext->memctl.memory_data) :
  // 如果 gcontext 为空，则使用标准库函数 malloc 分配 TABLES_LENGTH 大小的内存
  malloc(TABLES_LENGTH));
#endif  /* PCRE2_DFTABLES */

// 申明一个 int 类型的变量 i
int i;
// 申明一个指向 uint8_t 类型的指针变量 p
uint8_t *p;

// 如果 yield 为空，则返回空指针
if (yield == NULL) return NULL;
// 将 p 指向 yield
p = yield;

// 遍历 0 到 255 的整数，将每个整数的小写字母形式写入 p 所指向的内存
for (i = 0; i < 256; i++) *p++ = tolower(i);

// 遍历 0 到 255 的整数，将每个整数的大小写字母形式写入 p 所指向的内存
for (i = 0; i < 256; i++) *p++ = islower(i)? toupper(i) : tolower(i);

// 接下来是字符类别表。不要试图在排他性的字符类别上节省工作，因为在某些区域设置中可能会有所不同。
// 注意，"space" 表中包括 "isspace" 给出的所有内容，包括默认区域设置中的 VT。这使得它适用于 POSIX 类 [:space:]。
// 从 PCRE1 版本 8.34 开始，对于所有 PCRE2 版本，它也适用于 Perl 空格，因为 Perl 在 5.18 版本中添加了 VT。
// 还要注意，字符可能是 alnum 或 alpha 而不是 lower 或 upper，例如在 fr_FR 区域设置中的 "male and female ordinals" (\xAA 和 \xBA)（至少在 Debian Linux 的区域设置中是这样）。
// 因此，我们必须特别测试 alnum。
memset(p, 0, cbit_length);
for (i = 0; i < 256; i++)
  {
  if (isdigit(i))  p[cbit_digit  + i/8] |= 1u << (i&7);
  if (isupper(i))  p[cbit_upper  + i/8] |= 1u << (i&7);
  if (islower(i))  p[cbit_lower  + i/8] |= 1u << (i&7);
  if (isalnum(i))  p[cbit_word   + i/8] |= 1u << (i&7);
  if (i == '_')    p[cbit_word   + i/8] |= 1u << (i&7);
  if (isspace(i))  p[cbit_space  + i/8] |= 1u << (i&7);
  if (isxdigit(i)) p[cbit_xdigit + i/8] |= 1u << (i&7);
  if (isgraph(i))  p[cbit_graph  + i/8] |= 1u << (i&7);
  if (isprint(i))  p[cbit_print  + i/8] |= 1u << (i&7);
  if (ispunct(i))  p[cbit_punct  + i/8] |= 1u << (i&7);
  if (iscntrl(i))  p[cbit_cntrl  + i/8] |= 1u << (i&7);
  }
p += cbit_length;

// 最后，字符类型表。在这里，我们过去会将 VT 从空白字符中排除，因为 Perl 对于 \s 和 \S 不将其识别为这样的字符。
/* 注释：在正则表达式中的注释。然而，Perl 在 5.18 版本发生了变化，所以 PCRE1 在 8.34 版本发生了变化，而对于 PCRE2 来说，它一直都是这样。*/

for (i = 0; i < 256; i++)
{
  int x = 0;
  if (isspace(i)) x += ctype_space;  // 如果是空白字符，加上空白字符标记
  if (isalpha(i)) x += ctype_letter;  // 如果是字母，加上字母标记
  if (islower(i)) x += ctype_lcletter;  // 如果是小写字母，加上小写字母标记
  if (isdigit(i)) x += ctype_digit;  // 如果是数字，加上数字标记
  if (isalnum(i) || i == '_') x += ctype_word;  // 如果是字母数字或下划线，加上单词标记
  *p++ = x;  // 将 x 的值赋给指针 p 所指向的位置，然后递增指针 p
}

return yield;  // 返回 yield 变量的值

#ifndef PCRE2_DFTABLES   /* 编译库 */
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_maketables_free(pcre2_general_context *gcontext, const uint8_t *tables)
{
  if (gcontext)
    gcontext->memctl.free((void *)tables, gcontext->memctl.memory_data);  // 如果 gcontext 存在，则使用 gcontext 的内存控制器释放 tables
  else
    free((void *)tables);  // 否则，直接释放 tables
}
#endif

/* End of pcre2_maketables.c */  // pcre2_maketables.c 文件结束
```