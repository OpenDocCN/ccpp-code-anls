# `nmap\libpcre\src\pcre2posix.c`

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
/* This module is a wrapper that provides a POSIX API to the underlying PCRE2
functions. The operative functions are called pcre2_regcomp(), etc., with
wrappers that use the plain POSIX names. In addition, pcre2posix.h defines the
POSIX names as macros for the pcre2_xxx functions, so any program that includes
it and uses the POSIX names will call the base functions directly. This makes
it easier for an application to be sure it gets the PCRE2 versions in the
presence of other POSIX regex libraries. */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

/* Ensure that the PCRE2POSIX_EXP_xxx macros are set appropriately for
compiling these functions. This must come before including pcre2posix.h, where
they are set for an application (using these functions) if they have not
previously been set. */

#if defined(_WIN32) && !defined(PCRE2_STATIC)
#  define PCRE2POSIX_EXP_DECL extern __declspec(dllexport)
#  define PCRE2POSIX_EXP_DEFN __declspec(dllexport)
#endif

/* Older versions of MSVC lack snprintf(). This define allows for
warning/error-free compilation and testing with MSVC compilers back to at least
MSVC 10/2010. Except for VC6 (which is missing some fundamentals and fails). */

#if defined(_MSC_VER) && (_MSC_VER < 1900)
#define snprintf _snprintf
#endif

/* Compile-time error numbers start at this value. It should probably never be
changed. This #define is a copy of the one in pcre2_internal.h. */

#define COMPILE_ERROR_BASE 100

/* Standard C headers */
#include <ctype.h>
#include <limits.h>
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* PCRE2 headers */
#include "pcre2.h"
#include "pcre2posix.h"

/* Table to translate PCRE2 compile time error codes into POSIX error codes. */
# 定义一个静态的整型数组，用于存储 PCRE2 错误码对应的特殊 POSIX 错误码
static const int eint1[] = {
  0,           /* No error */
  REG_EESCAPE, /* \ at end of pattern */
  REG_EESCAPE, /* \c at end of pattern */
  REG_EESCAPE, /* unrecognized character follows \ */
  REG_BADBR,   /* numbers out of order in {} quantifier */
  /* 5 */
  REG_BADBR,   /* number too big in {} quantifier */
  REG_EBRACK,  /* missing terminating ] for character class */
  REG_ECTYPE,  /* invalid escape sequence in character class */
  REG_ERANGE,  /* range out of order in character class */
  REG_BADRPT,  /* nothing to repeat */
  /* 10 */
  REG_ASSERT,  /* internal error: unexpected repeat */
  REG_BADPAT,  /* unrecognized character after (? or (?- */
  REG_BADPAT,  /* POSIX named classes are supported only within a class */
  REG_BADPAT,  /* POSIX collating elements are not supported */
  REG_EPAREN,  /* missing ) */
  /* 15 */
  REG_ESUBREG, /* reference to non-existent subpattern */
  REG_INVARG,  /* pattern passed as NULL */
  REG_INVARG,  /* unknown compile-time option bit(s) */
  REG_EPAREN,  /* missing ) after (?# comment */
  REG_ESIZE,   /* parentheses nested too deeply */
  /* 20 */
  REG_ESIZE,   /* regular expression too large */
  REG_ESPACE,  /* failed to get memory */
  REG_EPAREN,  /* unmatched closing parenthesis */
  REG_ASSERT   /* internal error: code overflow */
  };

# 定义另一个静态的整型数组，用于存储特殊的 PCRE2 错误码对应的 POSIX 错误码
static const int eint2[] = {
  30, REG_ECTYPE,  /* unknown POSIX class name */
  32, REG_INVARG,  /* this version of PCRE2 does not have Unicode support */
  37, REG_EESCAPE, /* PCRE2 does not support \L, \l, \N{name}, \U, or \u */
  56, REG_INVARG,  /* internal error: unknown newline setting */
  92, REG_INVARG,  /* invalid option bits with PCRE2_LITERAL */
  99, REG_EESCAPE  /* \K in lookaround */
};

# 定义一个表格，用于存储与 POSIX 错误码对应的文本
// 定义一个字符串数组，包含了错误信息的字符串
static const char *const pstring[] = {
  "",                                /* Dummy for value 0 */
  "internal error",                  /* REG_ASSERT */
  "invalid repeat counts in {}",     /* BADBR      */
  "pattern error",                   /* BADPAT     */
  "? * + invalid",                   /* BADRPT     */
  "unbalanced {}",                   /* EBRACE     */
  "unbalanced []",                   /* EBRACK     */
  "collation error - not relevant",  /* ECOLLATE   */
  "bad class",                       /* ECTYPE     */
  "bad escape sequence",             /* EESCAPE    */
  "empty expression",                /* EMPTY      */
  "unbalanced ()",                   /* EPAREN     */
  "bad range inside []",             /* ERANGE     */
  "expression too big",              /* ESIZE      */
  "failed to get memory",            /* ESPACE     */
  "bad back reference",              /* ESUBREG    */
  "bad argument",                    /* INVARG     */
  "match failed"                     /* NOMATCH    */
};

// 下面的代码是为了 10.33 版本创建的（参见 ChangeLog 10.33 #4），当 POSIX 函数被赋予 pcre2_... 名称而不是传统的 POSIX 名称时。然而，它被证明比有用更麻烦。至少有两种情况，一个程序链接了两个其他程序，其中一个使用 POSIX 库，另一个使用 PCRE2 POSIX 函数，因此导致两个实例的 POSIX 函数存在，引起麻烦。对于 10.37 版本，这段代码被注释掉了。在适当的时候，如果没有问题，它可以被移除。唯一的小担心是下面的注释，关于不包括 pcre2posix.h 的语言。如果有这样的情况，它们将不得不使用 PCRE2 名称。

/*************************************************
*      Wrappers with traditional POSIX names     *
*************************************************/

/* Keep defining them to preseve the ABI for applications linked to the pcre2
/* 
在 pcre2posix.h 中将这些名称更改为宏之前的 POSIX 库。
这也确保了 POSIX 名称可以从不包括 pcre2posix.h 的语言中调用。
非常重要的是要取消 pcre2posix.h 中的宏定义！
*/

// 取消 regerror 宏定义，并定义 regerror 函数
#undef regerror
PCRE2POSIX_EXP_DECL size_t regerror(int, const regex_t *, char *, size_t);
PCRE2POSIX_EXP_DEFN size_t PCRE2_CALL_CONVENTION
regerror(int errcode, const regex_t *preg, char *errbuf, size_t errbuf_size)
{
    return pcre2_regerror(errcode, preg, errbuf, errbuf_size);
}

// 取消 regfree 宏定义，并定义 regfree 函数
#undef regfree
PCRE2POSIX_EXP_DECL void regfree(regex_t *);
PCRE2POSIX_EXP_DEFN void PCRE2_CALL_CONVENTION
regfree(regex_t *preg)
{
    pcre2_regfree(preg);
}

// 取消 regcomp 宏定义，并定义 regcomp 函数
#undef regcomp
PCRE2POSIX_EXP_DECL int regcomp(regex_t *, const char *, int);
PCRE2POSIX_EXP_DEFN int PCRE2_CALL_CONVENTION
regcomp(regex_t *preg, const char *pattern, int cflags)
{
    return pcre2_regcomp(preg, pattern, cflags);
}

// 取消 regexec 宏定义，并定义 regexec 函数
#undef regexec
PCRE2POSIX_EXP_DECL int regexec(const regex_t *, const char *, size_t, regmatch_t *, int);
PCRE2POSIX_EXP_DEFN int PCRE2_CALL_CONVENTION
regexec(const regex_t *preg, const char *string, size_t nmatch, regmatch_t pmatch[], int eflags)
{
    return pcre2_regexec(preg, string, nmatch, pmatch, eflags);
}
#endif

/*************************************************
*          将错误代码转换为字符串                  *
*************************************************/

// 定义将错误代码转换为字符串的函数 pcre2_regerror
PCRE2POSIX_EXP_DEFN size_t PCRE2_CALL_CONVENTION
pcre2_regerror(int errcode, const regex_t *preg, char *errbuf, size_t errbuf_size)
{
    int used;
    const char *message;

    // 如果错误代码小于等于 0 或大于等于 pstring 数组的大小，则返回 "unknown error code"，否则返回对应的错误信息
    message = (errcode <= 0 || errcode >= (int)(sizeof(pstring)/sizeof(char *)))? "unknown error code" : pstring[errcode];

    // 如果 preg 不为空且 re_erroffset 不为 -1，则将错误信息和偏移量写入 errbuf，否则只写入错误信息
    if (preg != NULL && (int)preg->re_erroffset != -1)
    {
        used = snprintf(errbuf, errbuf_size, "%s at offset %-6d", message, (int)preg->re_erroffset);
    }
    else
    {
        used = snprintf(errbuf, errbuf_size, "%s", message);
    }

    return used + 1;
}
/* 释放正则表达式占用的内存 */
PCRE2POSIX_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_regfree(regex_t *preg)
{
    // 释放正则表达式匹配数据占用的内存
    pcre2_match_data_free(preg->re_match_data);
    // 释放正则表达式编译结果占用的内存
    pcre2_code_free(preg->re_pcre2_code);
}

/* 编译正则表达式 */
/*************************************************
*            Compile a regular expression        *
*************************************************/
/*
参数：
  preg        指向用于记录编译表达式的结构体
  pattern     要编译的模式
  cflags      编译标志

返回值：      成功返回 0
              失败返回各种非零代码
*/
PCRE2POSIX_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_regcomp(regex_t *preg, const char *pattern, int cflags)
{
    PCRE2_SIZE erroffset;
    PCRE2_SIZE patlen;
    int errorcode;
    int options = 0;
    int re_nsub = 0;

    // 如果编译标志包含 REG_PEND，则计算模式的长度
    patlen = ((cflags & REG_PEND) != 0)? (PCRE2_SIZE)(preg->re_endp - pattern) : PCRE2_ZERO_TERMINATED;

    // 根据编译标志设置选项
    if ((cflags & REG_ICASE) != 0)    options |= PCRE2_CASELESS;
    if ((cflags & REG_NEWLINE) != 0)  options |= PCRE2_MULTILINE;
    if ((cflags & REG_DOTALL) != 0)   options |= PCRE2_DOTALL;
    if ((cflags & REG_NOSPEC) != 0)   options |= PCRE2_LITERAL;
    if ((cflags & REG_UTF) != 0)      options |= PCRE2_UTF;
    if ((cflags & REG_UCP) != 0)      options |= PCRE2_UCP;
    if ((cflags & REG_UNGREEDY) != 0) options |= PCRE2_UNGREEDY;

    // 记录编译标志
    preg->re_cflags = cflags;
    // 编译正则表达式
    preg->re_pcre2_code = pcre2_compile((PCRE2_SPTR)pattern, patlen, options, &errorcode, &erroffset, NULL);
    preg->re_erroffset = erroffset;

    // 如果编译失败
    if (preg->re_pcre2_code == NULL)
    {
        unsigned int i;

        /* A negative value is a UTF error; otherwise all error codes are greater than COMPILE_ERROR_BASE, but check, just in case. */
        // 负值表示 UTF 错误；否则所有错误代码都大于 COMPILE_ERROR_BASE，但为了安全起见，还是检查一下
        if (errorcode < COMPILE_ERROR_BASE) return REG_BADPAT;
        errorcode -= COMPILE_ERROR_BASE;

        // 返回相应的错误代码
        if (errorcode < (int)(sizeof(eint1)/sizeof(const int)))
            return eint1[errorcode];
        for (i = 0; i < sizeof(eint2)/sizeof(const int); i += 2)
    # 如果错误代码等于 eint2[i]，则返回 eint2[i+1]
    if (errorcode == eint2[i]) return eint2[i+1];
    # 如果不满足上述条件，则返回 REG_BADPAT
    return REG_BADPAT;
  }
(void)pcre2_pattern_info((const pcre2_code *)preg->re_pcre2_code,
  PCRE2_INFO_CAPTURECOUNT, &re_nsub);
preg->re_nsub = (size_t)re_nsub;
preg->re_match_data = pcre2_match_data_create(re_nsub + 1, NULL);
preg->re_erroffset = (size_t)(-1);  /* No meaning after successful compile */

if (preg->re_match_data == NULL)
  {
  /* LCOV_EXCL_START */
  pcre2_code_free(preg->re_pcre2_code);
  return REG_ESPACE;
  /* LCOV_EXCL_STOP */
  }

return 0;
}

/*************************************************
*              Match a regular expression        *
*************************************************/

/* A suitable match_data block, large enough to hold all possible captures, was
obtained when the pattern was compiled, to save having to allocate and free it
for each match. If REG_NOSUB was specified at compile time, the nmatch and
pmatch arguments are ignored, and the only result is yes/no/error. */

# 匹配正则表达式
PCRE2POSIX_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_regexec(const regex_t *preg, const char *string, size_t nmatch,
  regmatch_t pmatch[], int eflags)
{
int rc, so, eo;
int options = 0;
pcre2_match_data *md = (pcre2_match_data *)preg->re_match_data;

# 如果输入字符串为空，则返回无效参数错误
if (string == NULL) return REG_INVARG;

# 根据 eflags 设置 PCRE2 的选项
if ((eflags & REG_NOTBOL) != 0) options |= PCRE2_NOTBOL;
if ((eflags & REG_NOTEOL) != 0) options |= PCRE2_NOTEOL;
if ((eflags & REG_NOTEMPTY) != 0) options |= PCRE2_NOTEMPTY;

# 当编译时指定了 REG_NOSUB，或者没有传入 pmatch 来存放捕获的字符串时，确保 nmatch 为零
# 这将阻止任何尝试写入 pmatch
if ((preg->re_cflags & REG_NOSUB) != 0 || pmatch == NULL) nmatch = 0;

# REG_STARTEND 是 BSD 的扩展，允许非 NUL 结尾的字符串。
# OS X 的 man 页面说 "REG_STARTEND 只影响字符串的位置，不影响匹配方式"。
# 这就是为什么 "so" 值用于增加起始位置，而不是作为 PCRE2 的 "starting offset" 传递的原因
# 如果设置了 REG_STARTEND 标志位
if ((eflags & REG_STARTEND) != 0)
  {
  # 如果 pmatch 为空，返回无效参数错误
  if (pmatch == NULL) return REG_INVARG;
  # 获取匹配的起始位置和结束位置
  so = pmatch[0].rm_so;
  eo = pmatch[0].rm_eo;
  }
# 如果未设置 REG_STARTEND 标志位
else
  {
  # 起始位置为 0
  so = 0;
  # 结束位置为字符串长度
  eo = (int)strlen(string);
  }

# 调用 pcre2_match 函数进行正则表达式匹配
rc = pcre2_match((const pcre2_code *)preg->re_pcre2_code,
  (PCRE2_SPTR)string + so, (eo - so), 0, options, md, NULL);

# 如果匹配成功
if (rc >= 0)
  {
  # 定义变量 i 和 ovector
  size_t i;
  PCRE2_SIZE *ovector = pcre2_get_ovector_pointer(md);
  # 如果匹配的数量大于 nmatch，将 rc 设置为 nmatch
  if ((size_t)rc > nmatch) rc = (int)nmatch;
  # 遍历匹配结果，更新 pmatch 数组的起始位置和结束位置
  for (i = 0; i < (size_t)rc; i++)
    {
    pmatch[i].rm_so = (ovector[i*2] == PCRE2_UNSET)? -1 :
      (int)(ovector[i*2] + so);
    pmatch[i].rm_eo = (ovector[i*2+1] == PCRE2_UNSET)? -1 :
      (int)(ovector[i*2+1] + so);
    }
  # 将剩余的 pmatch 数组元素的起始位置和结束位置设置为 -1
  for (; i < nmatch; i++) pmatch[i].rm_so = pmatch[i].rm_eo = -1;
  # 返回匹配成功
  return 0;
  }

# 如果匹配不成功
if (rc <= PCRE2_ERROR_UTF8_ERR1 && rc >= PCRE2_ERROR_UTF8_ERR21)
  return REG_INVARG;

# 根据错误码返回相应的错误类型
switch(rc)
  {
  case PCRE2_ERROR_HEAPLIMIT: return REG_ESPACE;
  case PCRE2_ERROR_NOMATCH: return REG_NOMATCH;

  # 以下错误码在测试中不太可能出现，因此排除在覆盖率之外
  # LCOV_EXCL_START
  case PCRE2_ERROR_BADMODE: return REG_INVARG;
  case PCRE2_ERROR_BADMAGIC: return REG_INVARG;
  case PCRE2_ERROR_BADOPTION: return REG_INVARG;
  case PCRE2_ERROR_BADUTFOFFSET: return REG_INVARG;
  case PCRE2_ERROR_MATCHLIMIT: return REG_ESPACE;
  case PCRE2_ERROR_NOMEMORY: return REG_ESPACE;
  case PCRE2_ERROR_NULL: return REG_INVARG;
  default: return REG_ASSERT;
  # LCOV_EXCL_STOP
  }
}

# 结束 pcre2posix.c
```