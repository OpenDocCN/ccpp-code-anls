# `nmap\libpcre\src\pcre2posix.h`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE2 is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language. This is
the public header file to be #included by applications that call PCRE2 via the
POSIX wrapper interface.

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
/* 
   为了确保 size_t 被定义，必须包含 stdlib.h
*/
#include <stdlib.h>

/* 
   允许 C++ 用户
*/
#ifdef __cplusplus
extern "C" {
#endif

/* 
   选项，大部分由 POSIX 定义，但也包含一些额外的选项
*/
#define REG_ICASE     0x0001  /* 映射到 PCRE2_CASELESS */
#define REG_NEWLINE   0x0002  /* 映射到 PCRE2_MULTILINE */
#define REG_NOTBOL    0x0004  /* 映射到 PCRE2_NOTBOL */
#define REG_NOTEOL    0x0008  /* 映射到 PCRE2_NOTEOL */
#define REG_DOTALL    0x0010  /* POSIX 未定义；映射到 PCRE2_DOTALL */
#define REG_NOSUB     0x0020  /* 不报告匹配的内容 */
#define REG_UTF       0x0040  /* POSIX 未定义；映射到 PCRE2_UTF */
#define REG_STARTEND  0x0080  /* BSD 特性：通过 so,eo 传递主题字符串 */
#define REG_NOTEMPTY  0x0100  /* POSIX 未定义；映射到 PCRE2_NOTEMPTY */
#define REG_UNGREEDY  0x0200  /* POSIX 未定义；映射到 PCRE2_UNGREEDY */
#define REG_UCP       0x0400  /* POSIX 未定义；映射到 PCRE2_UCP */
#define REG_PEND      0x0800  /* GNU 特性：通过 re_endp 传递结束模式 */
#define REG_NOSPEC    0x1000  /* 映射到 PCRE2_LITERAL */

/* 
   这个选项并不被 PCRE2 使用，但通过定义它，我们可以更容易地将 PCRE2 插入到现有的进行 POSIX 调用的程序中
*/
#define REG_EXTENDED  0

/* 
   错误值。并非所有这些都与包装器相关或被使用
*/
# 定义枚举类型，表示正则表达式编译过程中可能出现的错误
enum {
  REG_ASSERT = 1,  /* internal error ? */  // 内部错误？
  REG_BADBR,       /* invalid repeat counts in {} */  // {} 中的重复计数无效
  REG_BADPAT,      /* pattern error */  // 模式错误
  REG_BADRPT,      /* ? * + invalid */  // ? * + 无效
  REG_EBRACE,      /* unbalanced {} */  // {} 不平衡
  REG_EBRACK,      /* unbalanced [] */  // [] 不平衡
  REG_ECOLLATE,    /* collation error - not relevant */  // 整理错误 - 不相关
  REG_ECTYPE,      /* bad class */  // 错误的类
  REG_EESCAPE,     /* bad escape sequence */  // 错误的转义序列
  REG_EMPTY,       /* empty expression */  // 空表达式
  REG_EPAREN,      /* unbalanced () */  // () 不平衡
  REG_ERANGE,      /* bad range inside [] */  // [] 内的范围错误
  REG_ESIZE,       /* expression too big */  // 表达式太大
  REG_ESPACE,      /* failed to get memory */  // 获取内存失败
  REG_ESUBREG,     /* bad back reference */  // 错误的后向引用
  REG_INVARG,      /* bad argument */  // 错误的参数
  REG_NOMATCH      /* match failed */  // 匹配失败
};

/* 表示编译后的正则表达式的结构。同时也用于在设置 REG_PEND 时传递模式结束指针。 */
typedef struct {
  void *re_pcre2_code;
  void *re_match_data;
  const char *re_endp;
  size_t re_nsub;
  size_t re_erroffset;
  int re_cflags;
} regex_t;

/* 表示捕获偏移量的结构。 */
typedef int regoff_t;

typedef struct {
  regoff_t rm_so;
  regoff_t rm_eo;
} regmatch_t;

/* 在使用 MSVC 编译器时，有时需要在导出的函数名前包含 "calling convention"。
   为了方便起见，所有导出的函数在其名称前都有 PCRE2_CALL_CONVENTION。
   这很少需要；如果没有设置，我们确保它没有影响。 */
#ifndef PCRE2_CALL_CONVENTION
#define PCRE2_CALL_CONVENTION
#endif

/* 当应用程序在 Windows 中链接到 PCRE2 DLL 时，需要标识为导入的符号。
   在构建 PCRE2 时，需要设置适当的导出设置，并在包含此文件之前在 pcre2posix.c 中设置。 */
#if defined(_WIN32) && !defined(PCRE2_STATIC) && !defined(PCRE2POSIX_EXP_DECL)
#  define PCRE2POSIX_EXP_DECL  extern __declspec(dllimport)
#  define PCRE2POSIX_EXP_DEFN  __declspec(dllimport)
#endif

/* 如果在 Windows 平台，并且没有定义 PCRE2_STATIC 和 PCRE2POSIX_EXP_DECL，则进行以下定义 */
#ifndef PCRE2POSIX_EXP_DECL
#  ifdef __cplusplus
#    define PCRE2POSIX_EXP_DECL  extern "C"
#    define PCRE2POSIX_EXP_DEFN  extern "C"
#  else
#    define PCRE2POSIX_EXP_DECL  extern
#    define PCRE2POSIX_EXP_DEFN  extern
#  endif
#endif

/* 定义函数。实际代码在具有 pcre2_xxx 名称的函数中。为了确保它们始终从 PCRE2 库链接，而不是意外地从其他地方链接（在其他地方，regex_t 的大小不同），提供了 POSIX 名称作为宏。 */
PCRE2POSIX_EXP_DECL int PCRE2_CALL_CONVENTION pcre2_regcomp(regex_t *, const char *, int);
PCRE2POSIX_EXP_DECL int PCRE2_CALL_CONVENTION pcre2_regexec(const regex_t *, const char *, size_t,
                     regmatch_t *, int);
PCRE2POSIX_EXP_DECL size_t PCRE2_CALL_CONVENTION pcre2_regerror(int, const regex_t *, char *, size_t);
PCRE2POSIX_EXP_DECL void PCRE2_CALL_CONVENTION pcre2_regfree(regex_t *);

#define regcomp  pcre2_regcomp
#define regexec  pcre2_regexec
#define regerror pcre2_regerror
#define regfree  pcre2_regfree

/* Debian 有一个使用不同名称的补丁。这里现在保存它们，以免它们不得不维护自己的补丁，但 PCRE2 没有对它们进行文档化。 */
#define PCRE2regcomp  pcre2_regcomp
#define PCRE2regexec  pcre2_regexec
#define PCRE2regerror pcre2_regerror
#define PCRE2regfree  pcre2_regfree

#ifdef __cplusplus
}   /* extern "C" */
#endif

/* pcre2posix.h 结尾 */
```