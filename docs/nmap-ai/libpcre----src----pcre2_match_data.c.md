# `nmap\libpcre\src\pcre2_match_data.c`

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
/*
*  Create a match data block given ovector size
*/

/* A minimum of 1 is imposed on the number of ovector pairs. A maximum is also
imposed because the oveccount field in a match data block is uintt6_t. */

PCRE2_EXP_DEFN pcre2_match_data * PCRE2_CALL_CONVENTION
pcre2_match_data_create(uint32_t oveccount, pcre2_general_context *gcontext)
{
    pcre2_match_data *yield;
    if (oveccount < 1) oveccount = 1;  // 如果oveccount小于1，则设为1
    if (oveccount > UINT16_MAX) oveccount = UINT16_MAX;  // 如果oveccount大于UINT16_MAX，则设为UINT16_MAX
    yield = PRIV(memctl_malloc)(
        offsetof(pcre2_match_data, ovector) + 2*oveccount*sizeof(PCRE2_SIZE),  // 分配内存空间
        (pcre2_memctl *)gcontext);
    if (yield == NULL) return NULL;  // 如果分配内存失败，则返回NULL
    yield->oveccount = oveccount;  // 设置oveccount
    yield->flags = 0;  // 设置flags
    yield->heapframes = NULL;  // 初始化heapframes
    yield->heapframes_size = 0;  // 初始化heapframes_size
    return yield;  // 返回创建的match data block
}

/*************************************************
*  Create a match data block using pattern data  *
*************************************************/

/* If no context is supplied, use the memory allocator from the code. */

PCRE2_EXP_DEFN pcre2_match_data * PCRE2_CALL_CONVENTION
pcre2_match_data_create_from_pattern(const pcre2_code *code,
  pcre2_general_context *gcontext)
{
    if (gcontext == NULL) gcontext = (pcre2_general_context *)code;  // 如果没有提供context，则使用code中的内存分配器
    return pcre2_match_data_create(((pcre2_real_code *)code)->top_bracket + 1,  // 创建match data block，oveccount为top_bracket+1
        gcontext);
}

/*************************************************
*            Free a match data block             *
*************************************************/

PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_match_data_free(pcre2_match_data *match_data)
{
    if (match_data != NULL)  // 如果match_data不为空
    {
        if (match_data->heapframes != NULL)  // 如果heapframes不为空
  # 释放匹配数据结构中的堆帧内存
  match_data->memctl.free(match_data->heapframes, match_data->memctl.memory_data);
  # 如果标志中包含 PCRE2_MD_COPIED_SUBJECT，则释放匹配数据结构中的主题内存
  if ((match_data->flags & PCRE2_MD_COPIED_SUBJECT) != 0)
    match_data->memctl.free((void *)match_data->subject, match_data->memctl.memory_data);
  # 释放匹配数据结构本身的内存
  match_data->memctl.free(match_data, match_data->memctl.memory_data);
# 获取匹配中的最后一个标记
PCRE2_EXP_DEFN PCRE2_SPTR PCRE2_CALL_CONVENTION
pcre2_get_mark(pcre2_match_data *match_data)
{
    return match_data->mark;
}

# 获取指向 ovector 的指针
PCRE2_EXP_DEFN PCRE2_SIZE * PCRE2_CALL_CONVENTION
pcre2_get_ovector_pointer(pcre2_match_data *match_data)
{
    return match_data->ovector;
}

# 获取 ovector 插槽的数量
PCRE2_EXP_DEFN uint32_t PCRE2_CALL_CONVENTION
pcre2_get_ovector_count(pcre2_match_data *match_data)
{
    return match_data->oveccount;
}

# 获取匹配中的起始代码单元
PCRE2_EXP_DEFN PCRE2_SIZE PCRE2_CALL_CONVENTION
pcre2_get_startchar(pcre2_match_data *match_data)
{
    return match_data->startchar;
}

# 获取匹配数据块的大小
PCRE2_EXP_DEFN PCRE2_SIZE PCRE2_CALL_CONVENTION
pcre2_get_match_data_size(pcre2_match_data *match_data)
{
    return offsetof(pcre2_match_data, ovector) +
      2 * (match_data->oveccount) * sizeof(PCRE2_SIZE);
}

/* pcre2_match_data.c 的结束 */
```