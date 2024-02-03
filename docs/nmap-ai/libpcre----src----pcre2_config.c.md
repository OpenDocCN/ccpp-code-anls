# `nmap\libpcre\src\pcre2_config.c`

```cpp
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
   任何方式使用此软件引起的任何问题，即使已被告知可能会发生这样的损害。
-----------------------------------------------------------------------------
*/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

/* 保存配置的链接大小，以字节为单位。在16位和32位模式下，它的值会被pcre2_intmodedep.h（由pcre2_internal.h包含）更改为代码单元。 */
static int configured_link_size = LINK_SIZE;

#include "pcre2_internal.h"

/* 这些宏是将未引用的文本转换为C字符串的标准方法。
它们允许像PCRE2_MAJOR这样的宏被定义而不带引号，这对于希望测试它们的值的用户程序非常方便。 */
#define STRING(a)  # a
#define XSTRING(s) STRING(s)

/*************************************************
* 返回有关已配置功能的信息 *
*************************************************/

/* 如果where为NULL，则返回所需内存的长度。

参数：
  what             需要什么信息
  where            放置信息的位置

返回值：           如果返回数值，则为0
                   如果返回字符串值，则为>= 0
                   如果“where”未被识别，则为PCRE2_ERROR_BADOPTION
                     或者当未启用JIT时请求JIT目标
*/
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_config(uint32_t what, void *where)
{
if (where == NULL)  /* 请求长度 */
  {
  switch(what)
    {
    default:
    return PCRE2_ERROR_BADOPTION;

    case PCRE2_CONFIG_BSR:
    case PCRE2_CONFIG_COMPILED_WIDTHS:
    case PCRE2_CONFIG_DEPTHLIMIT:
    case PCRE2_CONFIG_HEAPLIMIT:
    case PCRE2_CONFIG_JIT:
    case PCRE2_CONFIG_LINKSIZE:
    case PCRE2_CONFIG_MATCHLIMIT:
    case PCRE2_CONFIG_NEVER_BACKSLASH_C:
    case PCRE2_CONFIG_NEWLINE:
    case PCRE2_CONFIG_PARENSLIMIT:
    case PCRE2_CONFIG_STACKRECURSE:    /* Obsolete */
    case PCRE2_CONFIG_TABLES_LENGTH:
    case PCRE2_CONFIG_UNICODE:
    # 返回一个 32 位无符号整数的大小
    return sizeof(uint32_t);

    # 下面的情况将在下面处理

    # 当配置为 PCRE2_CONFIG_JITTARGET 时
    # 当配置为 PCRE2_CONFIG_UNICODE_VERSION 时
    # 当配置为 PCRE2_CONFIG_VERSION 时
    # 跳出 switch 语句
    break;
    }
# 根据给定的配置参数进行不同的操作
switch (what)
  {
  # 默认情况下返回错误代码 PCRE2_ERROR_BADOPTION
  default:
  return PCRE2_ERROR_BADOPTION;

  # 对于 PCRE2_CONFIG_BSR 配置参数
  case PCRE2_CONFIG_BSR:
  # 如果支持 BSR_ANYCRLF，则将 where 指向的位置设置为 PCRE2_BSR_ANYCRLF
#ifdef BSR_ANYCRLF
  *((uint32_t *)where) = PCRE2_BSR_ANYCRLF;
# 否则将 where 指向的位置设置为 PCRE2_BSR_UNICODE
#else
  *((uint32_t *)where) = PCRE2_BSR_UNICODE;
#endif
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_COMPILED_WIDTHS 配置参数
  case PCRE2_CONFIG_COMPILED_WIDTHS:
  # 将 where 指向的位置设置为 0 加上支持的编译宽度
  *((uint32_t *)where) = 0
#ifdef SUPPORT_PCRE2_8
  + 1
#endif
#ifdef SUPPORT_PCRE2_16
  + 2
#endif
#ifdef SUPPORT_PCRE2_32
  + 4
#endif
  ;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_DEPTHLIMIT 配置参数
  case PCRE2_CONFIG_DEPTHLIMIT:
  # 将 where 指向的位置设置为 MATCH_LIMIT_DEPTH
  *((uint32_t *)where) = MATCH_LIMIT_DEPTH;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_HEAPLIMIT 配置参数
  case PCRE2_CONFIG_HEAPLIMIT:
  # 将 where 指向的位置设置为 HEAP_LIMIT
  *((uint32_t *)where) = HEAP_LIMIT;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_JIT 配置参数
  case PCRE2_CONFIG_JIT:
  # 如果支持 JIT，则将 where 指向的位置设置为 1，否则设置为 0
#ifdef SUPPORT_JIT
  *((uint32_t *)where) = 1;
#else
  *((uint32_t *)where) = 0;
#endif
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_JITTARGET 配置参数
  case PCRE2_CONFIG_JITTARGET:
  # 如果支持 JIT
#ifdef SUPPORT_JIT
    {
    # 获取 JIT 目标，将其长度返回
    const char *v = PRIV(jit_get_target)();
    return (int)(1 + ((where == NULL)?
      strlen(v) : PRIV(strcpy_c8)((PCRE2_UCHAR *)where, v)));
    }
# 否则返回错误代码 PCRE2_ERROR_BADOPTION
#else
  return PCRE2_ERROR_BADOPTION;
#endif

  # 对于 PCRE2_CONFIG_LINKSIZE 配置参数
  case PCRE2_CONFIG_LINKSIZE:
  # 将 where 指向的位置设置为 configured_link_size 的无符号 32 位整数值
  *((uint32_t *)where) = (uint32_t)configured_link_size;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_MATCHLIMIT 配置参数
  case PCRE2_CONFIG_MATCHLIMIT:
  # 将 where 指向的位置设置为 MATCH_LIMIT
  *((uint32_t *)where) = MATCH_LIMIT;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_NEWLINE 配置参数
  case PCRE2_CONFIG_NEWLINE:
  # 将 where 指向的位置设置为 NEWLINE_DEFAULT
  *((uint32_t *)where) = NEWLINE_DEFAULT;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_NEVER_BACKSLASH_C 配置参数
  case PCRE2_CONFIG_NEVER_BACKSLASH_C:
  # 如果支持 NEVER_BACKSLASH_C，则将 where 指向的位置设置为 1，否则设置为 0
#ifdef NEVER_BACKSLASH_C
  *((uint32_t *)where) = 1;
#else
  *((uint32_t *)where) = 0;
#endif
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_PARENSLIMIT 配置参数
  case PCRE2_CONFIG_PARENSLIMIT:
  # 将 where 指向的位置设置为 PARENS_NEST_LIMIT
  *((uint32_t *)where) = PARENS_NEST_LIMIT;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_STACKRECURSE 配置参数
  # 将 where 指向的位置设置为 0
  case PCRE2_CONFIG_STACKRECURSE:
  *((uint32_t *)where) = 0;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_TABLES_LENGTH 配置参数
  case PCRE2_CONFIG_TABLES_LENGTH:
  # 将 where 指向的位置设置为 TABLES_LENGTH
  *((uint32_t *)where) = TABLES_LENGTH;
  # 结束 case
  break;

  # 对于 PCRE2_CONFIG_UNICODE_VERSION 配置参数
    {
# 如果支持 UNICODE
#if defined SUPPORT_UNICODE
    # 获取 UNICODE 版本信息
    const char *v = PRIV(unicode_version);
# 否则设置 UNICODE 不支持的提示信息
#else
    const char *v = "Unicode not supported";
#endif
    # 返回一个整数值，表示条件表达式的结果
    return (int)(1 + ((where == NULL)? strlen(v) : PRIV(strcpy_c8)((PCRE2_UCHAR *)where, v)));
   }
  break;

  # 根据配置设置 Unicode 相关的选项
  case PCRE2_CONFIG_UNICODE:
#if defined SUPPORT_UNICODE
  # 如果定义了 SUPPORT_UNICODE 宏
  *((uint32_t *)where) = 1;
#else
  # 如果未定义 SUPPORT_UNICODE 宏
  *((uint32_t *)where) = 0;
#endif
  # 结束条件编译指令

  /* 下面设置 "v" 的技巧是为了处理 PCRE2_PRERELEASE 设置为空字符串的情况（对于真正的发布版本来说是这样）。
  如果在这种情况下使用第二个选择，它不会在日期之前留下空格。
  另一方面，如果在 PCRE2_PRERELEASE 不为空时将所有四个宏放入单个 XSTRING 中，会插入一个不需要的空格。
  使用“显而易见”的方法存在问题，比如：

     XSTRING(PCRE2_MAJOR) "." XSTRING(PCRE_MINOR)
     XSTRING(PCRE2_PRERELEASE) " " XSTRING(PCRE_DATE)

  因为当 PCRE2_PRERELEASE 为空时，这会导致尝试扩展 STRING()。C 标准规定：“如果（在参数替换之前）任何参数由于没有预处理标记而为空，则行为是未定义的。”
  结果是 gcc 将这种情况视为单个空字符串 - 这正是我们真正想要的 - 但 Visual C 对于宏的缺少参数会抱怨。不幸的是，两者都是正确的。由于似乎没有办法在编译时测试宏的值是否为空，我们必须采用运行时测试。 */

  case PCRE2_CONFIG_VERSION:
    # 在 PCRE2_CONFIG_VERSION 情况下
    {
    const char *v = (XSTRING(Z PCRE2_PRERELEASE)[1] == 0)?
      XSTRING(PCRE2_MAJOR.PCRE2_MINOR PCRE2_DATE) :
      XSTRING(PCRE2_MAJOR.PCRE2_MINOR) XSTRING(PCRE2_PRERELEASE PCRE2_DATE);
    # 设置 v 的值
    return (int)(1 + ((where == NULL)?
      strlen(v) : PRIV(strcpy_c8)((PCRE2_UCHAR *)where, v)));
    # 返回结果
    }
  }

return 0;
# 返回 0
}

/* End of pcre2_config.c */
# pcre2_config.c 结束
```