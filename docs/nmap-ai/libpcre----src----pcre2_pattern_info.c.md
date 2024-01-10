# `nmap\libpcre\src\pcre2_pattern_info.c`

```
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

# 如果有配置文件，则包含配置文件

#include "pcre2_internal.h"

# 包含内部 PCRE2 头文件

/*************************************************
*        Return info about compiled pattern      *
*************************************************/

/*
Arguments:
  code          points to compiled code
  what          what information is required
  where         where to put the information; if NULL, return length

Returns:        0 when data returned
                > 0 when length requested
                < 0 on error or unset value
*/

# 返回关于编译模式的信息

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_pattern_info(const pcre2_code *code, uint32_t what, void *where)
{
const pcre2_real_code *re = (pcre2_real_code *)code;

# 定义 pcre2_pattern_info 函数，接收编译后的模式代码、所需信息和存放信息的位置

if (where == NULL)   /* Requests field length */
  {
  switch(what)
    {
    case PCRE2_INFO_ALLOPTIONS:
    case PCRE2_INFO_ARGOPTIONS:
    case PCRE2_INFO_BACKREFMAX:
    case PCRE2_INFO_BSR:
    case PCRE2_INFO_CAPTURECOUNT:
    case PCRE2_INFO_DEPTHLIMIT:
    case PCRE2_INFO_EXTRAOPTIONS:
    case PCRE2_INFO_FIRSTCODETYPE:
    case PCRE2_INFO_FIRSTCODEUNIT:
    case PCRE2_INFO_HASBACKSLASHC:
    case PCRE2_INFO_HASCRORLF:
    case PCRE2_INFO_HEAPLIMIT:
    case PCRE2_INFO_JCHANGED:
    case PCRE2_INFO_LASTCODETYPE:
    case PCRE2_INFO_LASTCODEUNIT:
    case PCRE2_INFO_MATCHEMPTY:
    case PCRE2_INFO_MATCHLIMIT:
    case PCRE2_INFO_MAXLOOKBEHIND:
    case PCRE2_INFO_MINLENGTH:
    case PCRE2_INFO_NAMEENTRYSIZE:
    case PCRE2_INFO_NAMECOUNT:
    case PCRE2_INFO_NEWLINE:
    return sizeof(uint32_t);

    case PCRE2_INFO_FIRSTBITMAP:
    return sizeof(const uint8_t *);

    case PCRE2_INFO_JITSIZE:
    case PCRE2_INFO_SIZE:
    case PCRE2_INFO_FRAMESIZE:
    return sizeof(size_t);

    case PCRE2_INFO_NAMETABLE:
    return sizeof(PCRE2_SPTR);
    }
  }

# 如果存放信息的位置为 NULL，则返回字段长度

if (re == NULL) return PCRE2_ERROR_NULL;

# 如果编译后的模式代码为空，则返回空指针错误
# 检查块中的第一个字段是否是魔术数字，如果不是，则返回 PCRE2_ERROR_BADMAGIC
if (re->magic_number != MAGIC_NUMBER) return PCRE2_ERROR_BADMAGIC;

# 检查此模式是否是在正确的位模式下编译的
if ((re->flags & (PCRE2_CODE_UNIT_WIDTH/8)) == 0) return PCRE2_ERROR_BADMODE;

switch(what)
  {
  case PCRE2_INFO_ALLOPTIONS:
  # 将整体选项存储到指定位置
  *((uint32_t *)where) = re->overall_options;
  break;

  case PCRE2_INFO_ARGOPTIONS:
  # 将编译选项存储到指定位置
  *((uint32_t *)where) = re->compile_options;
  break;

  case PCRE2_INFO_BACKREFMAX:
  # 将最大反向引用存储到指定位置
  *((uint32_t *)where) = re->top_backref;
  break;

  case PCRE2_INFO_BSR:
  # 将 BSR 约定存储到指定位置
  *((uint32_t *)where) = re->bsr_convention;
  break;

  case PCRE2_INFO_CAPTURECOUNT:
  # 将捕获计数存储到指定位置
  *((uint32_t *)where) = re->top_bracket;
  break;

  case PCRE2_INFO_DEPTHLIMIT:
  # 将深度限制存储到指定位置，如果深度限制为 UINT32_MAX，则返回 PCRE2_ERROR_UNSET
  *((uint32_t *)where) = re->limit_depth;
  if (re->limit_depth == UINT32_MAX) return PCRE2_ERROR_UNSET;
  break;

  case PCRE2_INFO_EXTRAOPTIONS:
  # 将额外选项存储到指定位置
  *((uint32_t *)where) = re->extra_options;
  break;

  case PCRE2_INFO_FIRSTCODETYPE:
  # 将第一个代码类型存储到指定位置
  *((uint32_t *)where) = ((re->flags & PCRE2_FIRSTSET) != 0)? 1 :
                         ((re->flags & PCRE2_STARTLINE) != 0)? 2 : 0;
  break;

  case PCRE2_INFO_FIRSTCODEUNIT:
  # 将第一个代码单元存储到指定位置
  *((uint32_t *)where) = ((re->flags & PCRE2_FIRSTSET) != 0)?
    re->first_codeunit : 0;
  break;

  case PCRE2_INFO_FIRSTBITMAP:
  # 将第一个位图存储到指定位置
  *((const uint8_t **)where) = ((re->flags & PCRE2_FIRSTMAPSET) != 0)?
    &(re->start_bitmap[0]) : NULL;
  break;

  case PCRE2_INFO_FRAMESIZE:
  # 将帧大小存储到指定位置
  *((size_t *)where) = offsetof(heapframe, ovector) +
  re->top_bracket * 2 * sizeof(PCRE2_SIZE);
  break;
  # 计算 re->top_bracket * 2 * sizeof(PCRE2_SIZE) 的值

  case PCRE2_INFO_HASBACKSLASHC:
  *((uint32_t *)where) = (re->flags & PCRE2_HASBKC) != 0;
  break;
  # 如果 re->flags 中包含 PCRE2_HASBKC 标志位，则将 where 转换为 uint32_t 指针类型，并赋值为 1，否则赋值为 0

  case PCRE2_INFO_HASCRORLF:
  *((uint32_t *)where) = (re->flags & PCRE2_HASCRORLF) != 0;
  break;
  # 如果 re->flags 中包含 PCRE2_HASCRORLF 标志位，则将 where 转换为 uint32_t 指针类型，并赋值为 1，否则赋值为 0

  case PCRE2_INFO_HEAPLIMIT:
  *((uint32_t *)where) = re->limit_heap;
  if (re->limit_heap == UINT32_MAX) return PCRE2_ERROR_UNSET;
  break;
  # 将 where 转换为 uint32_t 指针类型，并赋值为 re->limit_heap 的值，如果 re->limit_heap 的值为 UINT32_MAX，则返回 PCRE2_ERROR_UNSET

  case PCRE2_INFO_JCHANGED:
  *((uint32_t *)where) = (re->flags & PCRE2_JCHANGED) != 0;
  break;
  # 如果 re->flags 中包含 PCRE2_JCHANGED 标志位，则将 where 转换为 uint32_t 指针类型，并赋值为 1，否则赋值为 0

  case PCRE2_INFO_JITSIZE:
  # 在此处缺少代码，需要补充
#ifdef SUPPORT_JIT
  # 如果支持 JIT 编译，则将可执行的 JIT 大小写入指定位置
  *((size_t *)where) = (re->executable_jit != NULL)?
    PRIV(jit_get_size)(re->executable_jit) : 0;
#else
  # 如果不支持 JIT 编译，则将 0 写入指定位置
  *((size_t *)where) = 0;
#endif
  break;

  case PCRE2_INFO_LASTCODETYPE:
  # 将最后一个代码类型写入指定位置
  *((uint32_t *)where) = ((re->flags & PCRE2_LASTSET) != 0)? 1 : 0;
  break;

  case PCRE2_INFO_LASTCODEUNIT:
  # 将最后一个代码单元写入指定位置
  *((uint32_t *)where) = ((re->flags & PCRE2_LASTSET) != 0)?
    re->last_codeunit : 0;
  break;

  case PCRE2_INFO_MATCHEMPTY:
  # 将匹配空字符串标志写入指定位置
  *((uint32_t *)where) = (re->flags & PCRE2_MATCH_EMPTY) != 0;
  break;

  case PCRE2_INFO_MATCHLIMIT:
  # 将匹配限制写入指定位置
  *((uint32_t *)where) = re->limit_match;
  if (re->limit_match == UINT32_MAX) return PCRE2_ERROR_UNSET;
  break;

  case PCRE2_INFO_MAXLOOKBEHIND:
  # 将最大回溯长度写入指定位置
  *((uint32_t *)where) = re->max_lookbehind;
  break;

  case PCRE2_INFO_MINLENGTH:
  # 将最小匹配长度写入指定位置
  *((uint32_t *)where) = re->minlength;
  break;

  case PCRE2_INFO_NAMEENTRYSIZE:
  # 将命名条目大小写入指定位置
  *((uint32_t *)where) = re->name_entry_size;
  break;

  case PCRE2_INFO_NAMECOUNT:
  # 将命名计数写入指定位置
  *((uint32_t *)where) = re->name_count;
  break;

  case PCRE2_INFO_NAMETABLE:
  # 将命名表写入指定位置
  *((PCRE2_SPTR *)where) = (PCRE2_SPTR)((char *)re + sizeof(pcre2_real_code));
  break;

  case PCRE2_INFO_NEWLINE:
  # 将换行约定写入指定位置
  *((uint32_t *)where) = re->newline_convention;
  break;

  case PCRE2_INFO_SIZE:
  # 将块大小写入指定位置
  *((size_t *)where) = re->blocksize;
  break;

  default: return PCRE2_ERROR_BADOPTION;
  }

return 0;
}



/*************************************************
*              Callout enumerator                *
*************************************************/

/*
Arguments:
  code          points to compiled code
  callback      function called for each callout block
  callout_data  user data passed to the callback

Returns:        0 when successfully completed
                < 0 on local error
               != 0 for callback error
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_callout_enumerate(const pcre2_code *code,
  int (*callback)(pcre2_callout_enumerate_block *, void *), void *callout_data)
{
# 将编译后的代码转换为实际的代码对象
pcre2_real_code *re = (pcre2_real_code *)code;
# 定义 pcre2_callout_enumerate_block 类型的变量 cb
pcre2_callout_enumerate_block cb;
# 定义 PCRE2_SPTR 类型的变量 cc
PCRE2_SPTR cc;
# 如果 re 为空，则返回 PCRE2_ERROR_NULL
if (re == NULL) return PCRE2_ERROR_NULL;
# 如果支持 UNICODE，则设置 utf 为真
#ifdef SUPPORT_UNICODE
utf = (re->overall_options & PCRE2_UTF) != 0;
#endif
# 检查第一个字段是否是魔术数字，如果不是，则返回 PCRE2_ERROR_BADMAGIC
if (re->magic_number != MAGIC_NUMBER) return PCRE2_ERROR_BADMAGIC;
# 检查该模式是否是在正确的位模式下编译的
if ((re->flags & (PCRE2_CODE_UNIT_WIDTH/8)) == 0) return PCRE2_ERROR_BADMODE;
# 将 cb.version 设置为 0
cb.version = 0;
# 将 cc 设置为 re 的起始地址加上 pcre2_real_code 的大小再加上 re->name_count 乘以 re->name_entry_size
cc = (PCRE2_SPTR)((uint8_t *)re + sizeof(pcre2_real_code))
     + re->name_count * re->name_entry_size;
# 进入循环
while (TRUE)
  {
  int rc;
  # 根据 *cc 的值进行不同的操作
  switch (*cc)
    {
    # 如果 *cc 的值是 OP_END，则返回 0
    case OP_END:
    return 0;
    # 如果 *cc 的值是 OP_CHAR、OP_CHARI、OP_NOT、OP_NOTI 等等，则将 cc 增加相应操作码的长度
    case OP_CHAR:
    case OP_CHARI:
    case OP_NOT:
    ...
    cc += PRIV(OP_lengths)[*cc];
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，并且当前字符是 UTF-8 编码的一部分，跳过额外的字节
    if (utf && HAS_EXTRALEN(cc[-1])) cc += GET_EXTRALEN(cc[-1]);
#endif
    # 跳出当前的 switch 语句
    break;

    case OP_TYPESTAR:
    case OP_TYPEMINSTAR:
    case OP_TYPEPLUS:
    case OP_TYPEMINPLUS:
    case OP_TYPEQUERY:
    case OP_TYPEMINQUERY:
    case OP_TYPEUPTO:
    case OP_TYPEMINUPTO:
    case OP_TYPEEXACT:
    case OP_TYPEPOSSTAR:
    case OP_TYPEPOSPLUS:
    case OP_TYPEPOSQUERY:
    case OP_TYPEPOSUPTO:
    # 根据操作码获取相应的长度，然后移动指针
    cc += PRIV(OP_lengths)[*cc];
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode 并且当前字符是属性或非属性操作码，移动指针
    if (cc[-1] == OP_PROP || cc[-1] == OP_NOTPROP) cc += 2;
#endif
    # 跳出当前的 switch 语句
    break;

#if defined SUPPORT_UNICODE || PCRE2_CODE_UNIT_WIDTH != 8
    case OP_XCLASS:
    # 获取下一个操作码的位置，然后移动指针
    cc += GET(cc, 1);
    # 跳出当前的 switch 语句
    break;
#endif

    case OP_MARK:
    case OP_COMMIT_ARG:
    case OP_PRUNE_ARG:
    case OP_SKIP_ARG:
    case OP_THEN_ARG:
    # 根据操作码获取相应的长度和偏移量，然后移动指针
    cc += PRIV(OP_lengths)[*cc] + cc[1];
    # 跳出当前的 switch 语句
    break;

    case OP_CALLOUT:
    # 设置回调结构体的相关属性
    cb.pattern_position = GET(cc, 1);
    cb.next_item_length = GET(cc, 1 + LINK_SIZE);
    cb.callout_number = cc[1 + 2*LINK_SIZE];
    cb.callout_string_offset = 0;
    cb.callout_string_length = 0;
    cb.callout_string = NULL;
    # 调用回调函数，处理回调数据
    rc = callback(&cb, callout_data);
    # 如果回调函数返回非零值，直接返回该值
    if (rc != 0) return rc;
    # 移动指针
    cc += PRIV(OP_lengths)[*cc];
    # 跳出当前的 switch 语句
    break;

    case OP_CALLOUT_STR:
    # 设置回调结构体的相关属性
    cb.pattern_position = GET(cc, 1);
    cb.next_item_length = GET(cc, 1 + LINK_SIZE);
    cb.callout_number = 0;
    cb.callout_string_offset = GET(cc, 1 + 3*LINK_SIZE);
    cb.callout_string_length =
      GET(cc, 1 + 2*LINK_SIZE) - (1 + 4*LINK_SIZE) - 2;
    cb.callout_string = cc + (1 + 4*LINK_SIZE) + 1;
    # 调用回调函数，处理回调数据
    rc = callback(&cb, callout_data);
    # 如果回调函数返回非零值，直接返回该值
    if (rc != 0) return rc;
    # 移动指针
    cc += GET(cc, 1 + 2*LINK_SIZE);
    # 跳出当前的 switch 语句
    break;

    default:
    # 根据操作码获取相应的长度，然后移动指针
    cc += PRIV(OP_lengths)[*cc];
    # 跳出当前的 switch 语句
    break;
    }
  }
}

/* End of pcre2_pattern_info.c */
```