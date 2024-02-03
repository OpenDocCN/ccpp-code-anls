# `nmap\libpcre\src\pcre2_context.c`

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
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

// 如果有配置文件，则包含配置文件


#include "pcre2_internal.h"

// 包含 pcre2_internal.h 文件


static void *default_malloc(size_t size, void *data)
{
(void)data;
return malloc(size);
}

// 默认的分配内存函数，忽略用户数据参数


static void default_free(void *block, void *data)
{
(void)data;
free(block);
}

// 默认的释放内存函数，忽略用户数据参数


extern void *PRIV(memctl_malloc)(size_t size, pcre2_memctl *memctl)
{
pcre2_memctl *newmemctl;
void *yield = (memctl == NULL)? malloc(size) :
  memctl->malloc(size, memctl->memory_data);
if (yield == NULL) return NULL;
newmemctl = (pcre2_memctl *)yield;
if (memctl == NULL)
  {
  newmemctl->malloc = default_malloc;
  newmemctl->free = default_free;
  newmemctl->memory_data = NULL;
  }
else *newmemctl = *memctl;
return yield;
}

// 获取内存块并保存内存控制数据的内部函数


/* Initializing for compile and match contexts is done in separate, private
functions so that these can be called from functions such as pcre2_compile()
when an external context is not supplied. The initializing functions have an
option to set up default memory management. */

// 初始化编译和匹配上下文是在单独的私有函数中完成的，这样可以在没有提供外部上下文的情况下从诸如 pcre2_compile() 等函数中调用这些函数。初始化函数有一个选项来设置默认的内存管理。
# 创建一个通用的正则表达式编译上下文
PCRE2_EXP_DEFN pcre2_general_context * PCRE2_CALL_CONVENTION
pcre2_general_context_create(void *(*private_malloc)(size_t, void *),
  void (*private_free)(void *, void *), void *memory_data)
{
# 声明一个指向 pcre2_general_context 结构体的指针 gcontext
pcre2_general_context *gcontext;
# 如果 private_malloc 为空，则使用默认的内存分配函数 default_malloc
if (private_malloc == NULL) private_malloc = default_malloc;
# 如果 private_free 为空，则使用默认的内存释放函数 default_free
if (private_free == NULL) private_free = default_free;
# 使用 private_malloc 分配内存来创建一个 pcre2_real_general_context 结构体，并将其赋值给 gcontext
gcontext = private_malloc(sizeof(pcre2_real_general_context), memory_data);
# 如果分配内存失败，则返回空指针
if (gcontext == NULL) return NULL;
# 设置 gcontext 的内存控制器的 malloc 和 free 函数，以及 memory_data
gcontext->memctl.malloc = private_malloc;
gcontext->memctl.free = private_free;
gcontext->memctl.memory_data = memory_data;
# 返回创建的通用上下文
return gcontext;
}

# 默认的编译上下文被设置为在运行时不需要初始化，当编译函数没有提供上下文时使用
const pcre2_compile_context PRIV(default_compile_context) = {
  { default_malloc, default_free, NULL },    /* Default memory handling */
  NULL,                                      /* Stack guard */
  NULL,                                      /* Stack guard data */
  PRIV(default_tables),                      /* Character tables */
  PCRE2_UNSET,                               /* Max pattern length */
  BSR_DEFAULT,                               /* Backslash R default */
  NEWLINE_DEFAULT,                           /* Newline convention */
  PARENS_NEST_LIMIT,                         /* As it says */
  0 };                                       /* Extra options */

# 创建函数将默认上下文复制到新的内存中，但如果提供了 gcontext，则必须覆盖默认的内存处理函数
PCRE2_EXP_DEFN pcre2_compile_context * PCRE2_CALL_CONVENTION
pcre2_compile_context_create(pcre2_general_context *gcontext)
{
# 使用 gcontext 的内存控制器的 malloc 函数来分配内存来创建一个 pcre2_real_compile_context 结构体，并将其赋值给 ccontext
pcre2_compile_context *ccontext = PRIV(memctl_malloc)(
  sizeof(pcre2_real_compile_context), (pcre2_memctl *)gcontext);
# 如果分配内存失败，则返回空指针
if (ccontext == NULL) return NULL;
# 将默认的编译上下文复制到 ccontext
*ccontext = PRIV(default_compile_context);
# 如果 gcontext 不为空，则将其内存控制器的函数复制到 ccontext
if (gcontext != NULL)
  *((pcre2_memctl *)ccontext) = *((pcre2_memctl *)gcontext);
# 返回创建的编译上下文
return ccontext;
}
/* 设置默认的匹配上下文，以便在没有提供上下文的情况下初始化运行时不必进行初始化 */

const pcre2_match_context PRIV(default_match_context) = {
  { default_malloc, default_free, NULL },  /* 默认内存分配和释放函数 */
#ifdef SUPPORT_JIT
  NULL,          /* JIT 回调 */
  NULL,          /* JIT 回调数据 */
#endif
  NULL,          /* 调用函数 */
  NULL,          /* 调用数据 */
  NULL,          /* 替换调用函数 */
  NULL,          /* 替换调用数据 */
  PCRE2_UNSET,   /* 偏移限制 */
  HEAP_LIMIT,    /* 堆限制 */
  MATCH_LIMIT,   /* 匹配限制 */
  MATCH_LIMIT_DEPTH  /* 匹配深度限制 */
};

/* 创建函数将默认值复制到新的内存中，但如果提供了 gcontext，则必须覆盖默认的内存处理函数 */

PCRE2_EXP_DEFN pcre2_match_context * PCRE2_CALL_CONVENTION
pcre2_match_context_create(pcre2_general_context *gcontext)
{
pcre2_match_context *mcontext = PRIV(memctl_malloc)(
  sizeof(pcre2_real_match_context), (pcre2_memctl *)gcontext);
if (mcontext == NULL) return NULL;
*mcontext = PRIV(default_match_context);
if (gcontext != NULL)
  *((pcre2_memctl *)mcontext) = *((pcre2_memctl *)gcontext);
return mcontext;
}

/* 设置默认的转换上下文，以便在没有提供上下文的情况下初始化运行时不必进行初始化 */

const pcre2_convert_context PRIV(default_convert_context) = {
  { default_malloc, default_free, NULL },    /* 默认内存处理 */
#ifdef _WIN32
  CHAR_BACKSLASH,                            /* 默认路径分隔符 */
  CHAR_GRAVE_ACCENT                          /* 默认转义字符 */
#else  /* Not Windows */
  CHAR_SLASH,                                /* 默认路径分隔符 */
  CHAR_BACKSLASH                             /* 默认转义字符 */
#endif
};

/* 创建函数将默认值复制到新的内存中，但如果提供了 gcontext，则必须覆盖默认的内存处理函数。 */
# 创建一个新的转换上下文，用于转换操作
PCRE2_EXP_DEFN pcre2_convert_context * PCRE2_CALL_CONVENTION
pcre2_convert_context_create(pcre2_general_context *gcontext)
{
# 分配内存以存储转换上下文对象
pcre2_convert_context *ccontext = PRIV(memctl_malloc)(
  sizeof(pcre2_real_convert_context), (pcre2_memctl *)gcontext);
# 如果内存分配失败，则返回空指针
if (ccontext == NULL) return NULL;
# 将默认的转换上下文复制到新创建的转换上下文中
*ccontext = PRIV(default_convert_context);
# 如果通用上下文不为空，则将通用上下文的内存控制器复制到新创建的转换上下文中
if (gcontext != NULL)
  *((pcre2_memctl *)ccontext) = *((pcre2_memctl *)gcontext);
# 返回新创建的转换上下文对象
return ccontext;
}


/*************************************************
*              Context copy functions            *
*************************************************/

# 复制通用上下文对象
PCRE2_EXP_DEFN pcre2_general_context * PCRE2_CALL_CONVENTION
pcre2_general_context_copy(pcre2_general_context *gcontext)
{
# 分配内存以存储新的通用上下文对象
pcre2_general_context *new =
  gcontext->memctl.malloc(sizeof(pcre2_real_general_context),
  gcontext->memctl.memory_data);
# 如果内存分配失败，则返回空指针
if (new == NULL) return NULL;
# 将原通用上下文对象的内容复制到新创建的通用上下文对象中
memcpy(new, gcontext, sizeof(pcre2_real_general_context));
# 返回新创建的通用上下文对象
return new;
}

# 复制编译上下文对象
PCRE2_EXP_DEFN pcre2_compile_context * PCRE2_CALL_CONVENTION
pcre2_compile_context_copy(pcre2_compile_context *ccontext)
{
# 分配内存以存储新的编译上下文对象
pcre2_compile_context *new =
  ccontext->memctl.malloc(sizeof(pcre2_real_compile_context),
  ccontext->memctl.memory_data);
# 如果内存分配失败，则返回空指针
if (new == NULL) return NULL;
# 将原编译上下文对象的内容复制到新创建的编译上下文对象中
memcpy(new, ccontext, sizeof(pcre2_real_compile_context));
# 返回新创建的编译上下文对象
return new;
}

# 复制匹配上下文对象
PCRE2_EXP_DEFN pcre2_match_context * PCRE2_CALL_CONVENTION
pcre2_match_context_copy(pcre2_match_context *mcontext)
{
# 分配内存以存储新的匹配上下文对象
pcre2_match_context *new =
  mcontext->memctl.malloc(sizeof(pcre2_real_match_context),
  mcontext->memctl.memory_data);
# 如果内存分配失败，则返回空指针
if (new == NULL) return NULL;
# 将原匹配上下文对象的内容复制到新创建的匹配上下文对象中
memcpy(new, mcontext, sizeof(pcre2_real_match_context));
# 返回新创建的匹配上下文对象
return new;
}

# 复制转换上下文对象
PCRE2_EXP_DEFN pcre2_convert_context * PCRE2_CALL_CONVENTION
pcre2_convert_context_copy(pcre2_convert_context *ccontext)
{
# 分配内存以存储新的转换上下文对象
pcre2_convert_context *new =
  ccontext->memctl.malloc(sizeof(pcre2_real_convert_context),
  ccontext->memctl.memory_data);
# 如果内存分配失败，则返回空指针
if (new == NULL) return NULL;
# 将原转换上下文对象的内容复制到新创建的转换上下文对象中
memcpy(new, ccontext, sizeof(pcre2_real_convert_context));
# 返回新创建的转换上下文对象
return new;
}
/*************************************************
*              Context free functions            *
*************************************************/

# 释放通用上下文内存
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_general_context_free(pcre2_general_context *gcontext)
{
# 如果通用上下文不为空，则释放内存
if (gcontext != NULL)
  gcontext->memctl.free(gcontext, gcontext->memctl.memory_data);
}


# 释放编译上下文内存
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_compile_context_free(pcre2_compile_context *ccontext)
{
# 如果编译上下文不为空，则释放内存
if (ccontext != NULL)
  ccontext->memctl.free(ccontext, ccontext->memctl.memory_data);
}


# 释放匹配上下文内存
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_match_context_free(pcre2_match_context *mcontext)
{
# 如果匹配上下文不为空，则释放内存
if (mcontext != NULL)
  mcontext->memctl.free(mcontext, mcontext->memctl.memory_data);
}


# 释放转换上下文内存
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_convert_context_free(pcre2_convert_context *ccontext)
{
# 如果转换上下文不为空，则释放内存
if (ccontext != NULL)
  ccontext->memctl.free(ccontext, ccontext->memctl.memory_data);
}


/*************************************************
*             Set values in contexts             *
*************************************************/

# 所有这些函数在成功时返回0，如果给定无效数据则返回PCRE2_ERROR_BADDATA。只有一些函数能够测试数据的有效性。

# ------------ 编译上下文 ------------

# 设置字符表
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_character_tables(pcre2_compile_context *ccontext,
  const uint8_t *tables)
{
ccontext->tables = tables;
return 0;
}

# 设置反斜杠转义字符的约定
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_bsr(pcre2_compile_context *ccontext, uint32_t value)
{
switch(value)
  {
  case PCRE2_BSR_ANYCRLF:
  case PCRE2_BSR_UNICODE:
  ccontext->bsr_convention = value;
  return 0;

  default:
  return PCRE2_ERROR_BADDATA;
  }
}

# 设置最大模式长度
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_max_pattern_length(pcre2_compile_context *ccontext, PCRE2_SIZE length)
{
ccontext->max_pattern_length = length;
return 0;
}

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
# 设置 PCRE2 编译上下文的换行符约定
pcre2_set_newline(pcre2_compile_context *ccontext, uint32_t newline)
{
    switch(newline)
    {
        case PCRE2_NEWLINE_CR:
        case PCRE2_NEWLINE_LF:
        case PCRE2_NEWLINE_CRLF:
        case PCRE2_NEWLINE_ANY:
        case PCRE2_NEWLINE_ANYCRLF:
        case PCRE2_NEWLINE_NUL:
            ccontext->newline_convention = newline;
            return 0;
        default:
            return PCRE2_ERROR_BADDATA;
    }
}

# 设置 PCRE2 编译上下文的括号嵌套限制
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_parens_nest_limit(pcre2_compile_context *ccontext, uint32_t limit)
{
    ccontext->parens_nest_limit = limit;
    return 0;
}

# 设置 PCRE2 编译上下文的额外选项
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_compile_extra_options(pcre2_compile_context *ccontext, uint32_t options)
{
    ccontext->extra_options = options;
    return 0;
}

# 设置 PCRE2 编译上下文的递归保护
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_compile_recursion_guard(pcre2_compile_context *ccontext,
  int (*guard)(uint32_t, void *), void *user_data)
{
    ccontext->stack_guard = guard;
    ccontext->stack_guard_data = user_data;
    return 0;
}

# 设置 PCRE2 匹配上下文的回调函数
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_callout(pcre2_match_context *mcontext,
  int (*callout)(pcre2_callout_block *, void *), void *callout_data)
{
    mcontext->callout = callout;
    mcontext->callout_data = callout_data;
    return 0;
}

# 设置 PCRE2 匹配上下文的替换回调函数
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_substitute_callout(pcre2_match_context *mcontext,
  int (*substitute_callout)(pcre2_substitute_callout_block *, void *),
    void *substitute_callout_data)
{
    mcontext->substitute_callout = substitute_callout;
    mcontext->substitute_callout_data = substitute_callout_data;
    return 0;
}

# 设置 PCRE2 匹配上下文的堆限制
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_heap_limit(pcre2_match_context *mcontext, uint32_t limit)
{
    mcontext->heap_limit = limit;
    return 0;
}

# 设置 PCRE2 匹配上下文的匹配限制
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_match_limit(pcre2_match_context *mcontext, uint32_t limit)
{
    mcontext->match_limit = limit;
    return 0;
}

# 设置 PCRE2 匹配上下文的深度限制
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_depth_limit(pcre2_match_context *mcontext, uint32_t limit)
{
# 设置匹配上下文的深度限制，并返回成功标志
mcontext->depth_limit = limit;
return 0;
}

# 设置匹配上下文的偏移限制，并返回成功标志
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_offset_limit(pcre2_match_context *mcontext, PCRE2_SIZE limit)
{
mcontext->offset_limit = limit;
return 0;
}

# 在 10.30 版本中已经过时的函数，第一个函数为了向后兼容性而保留。第二个函数现在不起作用。从覆盖报告中排除这两个函数。

# LCOV_EXCL_START

# 设置匹配上下文的递归限制，并返回成功标志
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_recursion_limit(pcre2_match_context *mcontext, uint32_t limit)
{
return pcre2_set_depth_limit(mcontext, limit);
}

# 设置匹配上下文的递归内存管理，并返回成功标志
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_recursion_memory_management(pcre2_match_context *mcontext,
  void *(*mymalloc)(size_t, void *), void (*myfree)(void *, void *),
  void *mydata)
{
(void)mcontext;
(void)mymalloc;
(void)myfree;
(void)mydata;
return 0;
}

# LCOV_EXCL_STOP

# 设置转换上下文的全局分隔符，并返回成功标志
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_glob_separator(pcre2_convert_context *ccontext, uint32_t separator)
{
if (separator != CHAR_SLASH && separator != CHAR_BACKSLASH &&
    separator != CHAR_DOT) return PCRE2_ERROR_BADDATA;
ccontext->glob_separator = separator;
return 0;
}

# 设置转换上下文的全局转义符，并返回成功标志
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_set_glob_escape(pcre2_convert_context *ccontext, uint32_t escape)
{
if (escape > 255 || (escape != 0 && !ispunct(escape)))
  return PCRE2_ERROR_BADDATA;
ccontext->glob_escape = escape;
return 0;
}

# pcre2_context.c 结束
```