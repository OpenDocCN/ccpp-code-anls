# `nmap\libpcre\src\pcre2_jit_compile.c`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
                    This module by Zoltan Herczeg
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

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

#include "pcre2_internal.h"

#ifdef SUPPORT_JIT
/* NMAP_MODIFICATIONS */
#endif

#ifdef SUPPORT_JIT
/* NMAP_MODIFICATIONS */
#endif

#define PUBLIC_JIT_COMPILE_OPTIONS \
  (PCRE2_JIT_COMPLETE|PCRE2_JIT_PARTIAL_SOFT|PCRE2_JIT_PARTIAL_HARD|PCRE2_JIT_INVALID_UTF)

#define PUBLIC_JIT_COMPILE_OPTIONS \
  (PCRE2_JIT_COMPLETE|PCRE2_JIT_PARTIAL_SOFT|PCRE2_JIT_PARTIAL_HARD|PCRE2_JIT_INVALID_UTF)

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_jit_compile(pcre2_code *code, uint32_t options)
{
pcre2_real_code *re = (pcre2_real_code *)code;
#ifdef SUPPORT_JIT
executable_functions *functions;
static int executable_allocator_is_working = -1;
#endif

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_jit_compile(pcre2_code *code, uint32_t options)
{
pcre2_real_code *re = (pcre2_real_code *)code;
#ifdef SUPPORT_JIT
executable_functions *functions;
static int executable_allocator_is_working = -1;
#endif

if (code == NULL)
  return PCRE2_ERROR_NULL;

if (code == NULL)
  return PCRE2_ERROR_NULL;

if ((options & ~PUBLIC_JIT_COMPILE_OPTIONS) != 0)
  return PCRE2_ERROR_JIT_BADOPTION;

if ((options & ~PUBLIC_JIT_COMPILE_OPTIONS) != 0)
  return PCRE2_ERROR_JIT_BADOPTION;

/* Support for invalid UTF was first introduced in JIT, with the option
PCRE2_JIT_INVALID_UTF. Later, support was added to the interpreter, and the
compile-time option PCRE2_MATCH_INVALID_UTF was created. This is now the
preferred feature, with the earlier option deprecated. However, for backward
compatibility, if the earlier option is set, it forces the new option so that
if JIT matching falls back to the interpreter, there is still support for
invalid UTF. However, if this function has already been successfully called
without PCRE2_JIT_INVALID_UTF and without PCRE2_MATCH_INVALID_UTF (meaning that
non-invalid-supporting JIT code was compiled), give an error.

If in the future support for PCRE2_JIT_INVALID_UTF is withdrawn, the following

/* Support for invalid UTF was first introduced in JIT, with the option
PCRE2_JIT_INVALID_UTF. Later, support was added to the interpreter, and the
compile-time option PCRE2_MATCH_INVALID_UTF was created. This is now the
preferred feature, with the earlier option deprecated. However, for backward
compatibility, if the earlier option is set, it forces the new option so that
if JIT matching falls back to the interpreter, there is still support for
invalid UTF. However, if this function has already been successfully called
without PCRE2_JIT_INVALID_UTF and without PCRE2_MATCH_INVALID_UTF (meaning that
non-invalid-supporting JIT code was compiled), give an error.

If in the future support for PCRE2_JIT_INVALID_UTF is withdrawn, the following
/*
  1. 从 pcre2.h.in 中删除定义，并从 PUBLIC_JIT_COMPILE_OPTIONS 列表中删除。
  2. 在此模块中用本地标志替换 PCRE2_JIT_INVALID_UTF。
  3. 在 pcre2_jit_test.c 中用本地标志替换 PCRE2_JIT_INVALID_UTF。
  4. 删除以下代码块。"re" 和 "functions" 的设置可以移动到下面的 JIT-only 块中，但如果这样做，非 JIT 情况下将需要 (void)re 和 (void)functions，以避免编译器警告。
*/

#ifdef SUPPORT_JIT
functions = (executable_functions *)re->executable_jit;
#endif

if ((options & PCRE2_JIT_INVALID_UTF) != 0)
  {
  if ((re->overall_options & PCRE2_MATCH_INVALID_UTF) == 0)
    {
#ifdef SUPPORT_JIT
    if (functions != NULL) return PCRE2_ERROR_JIT_BADOPTION;
#endif
    re->overall_options |= PCRE2_MATCH_INVALID_UTF;
    }
  }

/* 上述测试在有无 JIT 支持的情况下运行。这意味着即使没有 JIT，PCRE2_JIT_INVALID_UTF 也会传播回正则表达式选项（确保解释器支持）。但是现在，如果没有 JIT 支持，就返回错误。 */

#ifndef SUPPORT_JIT
return PCRE2_ERROR_JIT_BADOPTION;
#else  /* SUPPORT_JIT */
/* NMAP_MODIFICATIONS */
#endif  /* SUPPORT_JIT */
}

/* JIT 编译器采用一体化方法。这提高了安全性，因为代码生成函数不会被导出。 */

#define INCLUDED_FROM_PCRE2_JIT_COMPILE

#if 0
/* NMAP_MODIFICATIONS */
#include "pcre2_jit_match.c"
#include "pcre2_jit_misc.c"
#endif

/* pcre2_jit_compile.c 结尾 */
```