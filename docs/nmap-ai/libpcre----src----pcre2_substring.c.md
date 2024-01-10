# `nmap\libpcre\src\pcre2_substring.c`

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

#include "pcre2_internal.h"

/*************************************************
*   Copy named captured string to given buffer   *
*************************************************/

/* This function copies a single captured substring into a given buffer,
identifying it by name. If the regex permits duplicate names, the first
substring that is set is chosen.

Arguments:
  match_data     points to the match data
  stringname     the name of the required substring
  buffer         where to put the substring
  sizeptr        the size of the buffer, updated to the size of the substring

Returns:         if successful: zero
                 if not successful, a negative error code:
                   (1) an error from nametable_scan()
                   (2) an error from copy_bynumber()
                   (3) PCRE2_ERROR_UNAVAILABLE: no group is in ovector
                   (4) PCRE2_ERROR_UNSET: all named groups in ovector are unset
*/
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_copy_byname(pcre2_match_data *match_data, PCRE2_SPTR stringname,
  PCRE2_UCHAR *buffer, PCRE2_SIZE *sizeptr)
{
  PCRE2_SPTR first, last, entry;
  int failrc, entrysize;
  // 如果匹配数据是由 DFA 解释器产生的，则返回错误
  if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
    return PCRE2_ERROR_DFA_UFUNC;
  // 通过名称在 nametable 中查找对应的子字符串
  entrysize = pcre2_substring_nametable_scan(match_data->code, stringname,
    &first, &last);
  if (entrysize < 0) return entrysize;
  failrc = PCRE2_ERROR_UNAVAILABLE;
  // 遍历找到的子字符串
  for (entry = first; entry <= last; entry += entrysize)
    {
    uint32_t n = GET2(entry, 0);
    // 如果子字符串在 ovector 中存在
    if (n < match_data->oveccount)
      {
        // 如果子字符串已经被设置，则通过编号复制子字符串到指定的缓冲区
        if (match_data->ovector[n*2] != PCRE2_UNSET)
          return pcre2_substring_copy_bynumber(match_data, n, buffer, sizeptr);
        // 如果子字符串未被设置，则返回 PCRE2_ERROR_UNSET
        failrc = PCRE2_ERROR_UNSET;
      }
    }
  // 返回失败的错误码
  return failrc;
}
# 将匹配的字符串按编号复制到给定的缓冲区中
# 这个函数将单个捕获的子字符串复制到给定的缓冲区中，通过编号进行标识
# 参数：
#   match_data     指向匹配数据
#   stringnumber   所需子字符串的编号
#   buffer         子字符串的存放位置
#   sizeptr        缓冲区的大小，更新为子字符串的大小
# 返回值：
#   如果成功：0
#   如果不成功，负数错误代码：
#     PCRE2_ERROR_NOMEMORY: 缓冲区太小
#     PCRE2_ERROR_NOSUBSTRING: 没有这样的子字符串
#     PCRE2_ERROR_UNAVAILABLE: ovector太小
#     PCRE2_ERROR_UNSET: 子字符串未设置
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_copy_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_UCHAR *buffer, PCRE2_SIZE *sizeptr)
{
int rc;
PCRE2_SIZE size;
rc = pcre2_substring_length_bynumber(match_data, stringnumber, &size);
if (rc < 0) return rc;
if (size + 1 > *sizeptr) return PCRE2_ERROR_NOMEMORY;
memcpy(buffer, match_data->subject + match_data->ovector[stringnumber*2],
  CU2BYTES(size));
buffer[size] = 0;
*sizeptr = size;
return 0;
}


# 提取命名的捕获字符串
# 这个函数将单个捕获的子字符串，通过名称标识，复制到新的内存中。如果正则表达式允许重复的名称，将选择第一个设置的子字符串。
# 参数：
#   match_data     指向匹配数据的指针
#   stringname     所需子字符串的名称
#   stringptr      存放新内存指针的位置
#   sizeptr        存放子字符串长度的位置
/*
返回：如果成功，返回零
     如果不成功，返回负值：
       (1) 来自 nametable_scan() 的错误
       (2) 来自 get_bynumber() 的错误
       (3) PCRE2_ERROR_UNAVAILABLE: ovector 中没有任何分组
       (4) PCRE2_ERROR_UNSET: ovector 中所有命名分组都未设置
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_get_byname(pcre2_match_data *match_data,
  PCRE2_SPTR stringname, PCRE2_UCHAR **stringptr, PCRE2_SIZE *sizeptr)
{
PCRE2_SPTR first, last, entry;
int failrc, entrysize;
if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
  return PCRE2_ERROR_DFA_UFUNC;
获取命名分组在 nametable 中的大小
entrysize = pcre2_substring_nametable_scan(match_data->code, stringname,
  &first, &last);
如果获取失败，返回错误码
if (entrysize < 0) return entrysize;
设置默认失败错误码
failrc = PCRE2_ERROR_UNAVAILABLE;
遍历命名分组的 nametable
for (entry = first; entry <= last; entry += entrysize)
  {
  从 nametable 中获取分组编号
  uint32_t n = GET2(entry, 0);
  如果分组编号小于 ovector 中的分组数量
  if (n < match_data->oveccount)
    {
    如果 ovector 中的分组未设置，则返回相应编号的分组数据
    if (match_data->ovector[n*2] != PCRE2_UNSET)
      return pcre2_substring_get_bynumber(match_data, n, stringptr, sizeptr);
    设置未设置分组的错误码
    failrc = PCRE2_ERROR_UNSET;
    }
  }
返回失败错误码
return failrc;
}



/*************************************************
*      Extract captured string to new memory     *
*************************************************/

/* 这个函数将单个捕获的子字符串复制到新的内存中。

参数：
  match_data     指向匹配数据的指针
  stringnumber   所需子字符串的编号
  stringptr      指向新内存的指针
  sizeptr        子字符串的大小

返回：如果成功，返回0
     如果不成功，返回负错误码：
       PCRE2_ERROR_NOMEMORY: 获取内存失败
       PCRE2_ERROR_NOSUBSTRING: 没有这样的子字符串
       PCRE2_ERROR_UNAVAILABLE: ovector 太小
       PCRE2_ERROR_UNSET: 子字符串未设置
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
# 根据给定的匹配数据和子串编号，获取对应的子串内容和大小
pcre2_substring_get_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_UCHAR **stringptr, PCRE2_SIZE *sizeptr)
{
# 定义变量
int rc;
PCRE2_SIZE size;
PCRE2_UCHAR *yield;
# 调用函数获取指定编号子串的长度
rc = pcre2_substring_length_bynumber(match_data, stringnumber, &size);
# 如果获取长度失败，返回错误码
if (rc < 0) return rc;
# 分配内存空间用于存储子串内容
yield = PRIV(memctl_malloc)(sizeof(pcre2_memctl) +
  (size + 1)*PCRE2_CODE_UNIT_WIDTH, (pcre2_memctl *)match_data);
# 如果内存分配失败，返回内存不足错误码
if (yield == NULL) return PCRE2_ERROR_NOMEMORY;
# 将指针指向分配的内存空间
yield = (PCRE2_UCHAR *)(((char *)yield) + sizeof(pcre2_memctl));
# 将匹配数据中指定子串的内容拷贝到分配的内存空间中
memcpy(yield, match_data->subject + match_data->ovector[stringnumber*2],
  CU2BYTES(size));
# 在子串内容的末尾添加字符串结束符
yield[size] = 0;
# 将子串内容和大小返回给调用者
*stringptr = yield;
*sizeptr = size;
# 返回成功标识
return 0;
}



/*************************************************
*       Free memory obtained by get_substring    *
*************************************************/

# 释放由 get_substring 函数获取的内存空间
/*
Argument:     the result of a previous pcre2_substring_get_byxxx()
Returns:      nothing
*/

PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_substring_free(PCRE2_UCHAR *string)
{
# 如果传入的字符串指针不为空
if (string != NULL)
  {
  # 获取内存控制块的指针
  pcre2_memctl *memctl = (pcre2_memctl *)((char *)string - sizeof(pcre2_memctl));
  # 调用内存控制块的释放函数释放内存
  memctl->free(memctl, memctl->memory_data);
  }
}



/*************************************************
*         Get length of a named substring        *
*************************************************/

# 获取指定名称的捕获子串的长度
/* This function returns the length of a named captured substring. If the regex
permits duplicate names, the first substring that is set is chosen.

Arguments:
  match_data      pointer to match data
  stringname      the name of the required substring
  sizeptr         where to put the length

Returns:          0 if successful, else a negative error number
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_length_byname(pcre2_match_data *match_data,
  PCRE2_SPTR stringname, PCRE2_SIZE *sizeptr)
{
# 定义变量
PCRE2_SPTR first, last, entry;
int failrc, entrysize;
# 如果匹配数据是由 DFA 解释器产生的，返回 DFA 不支持错误码
if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
  return PCRE2_ERROR_DFA_UFUNC;
# 获取匹配数据中指定编号的子字符串的长度
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_length_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_SIZE *sizeptr)
{
# 定义左右边界
PCRE2_SIZE left, right;
# 获取匹配数据中的子字符串数量
int count = match_data->rc;
# 如果匹配数据为部分匹配，则返回部分匹配错误
if (count == PCRE2_ERROR_PARTIAL)
  {
  if (stringnumber > 0) return PCRE2_ERROR_PARTIAL;
  count = 0;
  }
# 如果匹配数据小于0，则返回匹配失败
else if (count < 0) return count;            /* Match failed */

# 如果匹配数据不是由 DFA 解释器匹配的
if (match_data->matchedby != PCRE2_MATCHEDBY_DFA_INTERPRETER)
  {
  # 如果指定编号超出了括号的数量，则返回无此子字符串错误
  if (stringnumber > match_data->code->top_bracket)
    return PCRE2_ERROR_NOSUBSTRING;
  # 如果指定编号超出了匹配数据的 ovector 数组的范围，则返回 ovector 数组过小错误
  if (stringnumber >= match_data->oveccount)
    return PCRE2_ERROR_UNAVAILABLE;
  # 如果指定编号的子字符串未设置，则返回子字符串未设置错误
  if (match_data->ovector[stringnumber*2] == PCRE2_UNSET)
    return PCRE2_ERROR_UNSET;
  }
else  /* 使用 pcre2_dfa_match() 匹配 */
  {
  // 如果字符串编号大于等于匹配数据的 oveccount，返回 PCRE2_ERROR_UNAVAILABLE
  if (stringnumber >= match_data->oveccount) return PCRE2_ERROR_UNAVAILABLE;
  // 如果 count 不为 0 并且字符串编号大于等于 count，返回 PCRE2_ERROR_UNSET
  if (count != 0 && stringnumber >= (uint32_t)count) return PCRE2_ERROR_UNSET;
  }

// 获取匹配数据中指定字符串编号的左边界和右边界
left = match_data->ovector[stringnumber*2];
right = match_data->ovector[stringnumber*2+1];
// 如果 sizeptr 不为 NULL，将左右边界之间的长度赋值给 sizeptr
if (sizeptr != NULL) *sizeptr = (left > right)? 0 : right - left;
// 返回 0
return 0;
}



/*************************************************
*    提取所有捕获的字符串到新的内存中    *
*************************************************/

/* 该函数获取一块内存并构建一个指针列表，其中包含所有捕获的子字符串。在列表的末尾放置一个空指针。
子字符串以零结尾，但如果最后一个参数非空，则还返回一个长度列表。这允许处理二进制数据。

参数：
  match_data     指向匹配数据的指针
  listptr        设置为指向指针列表的指针
  lengthsptr     设置为指向长度列表的指针（可以为 NULL）

返回值：         如果成功：0
                 如果不成功，一个负的错误代码：
                   PCRE2_ERROR_NOMEMORY：无法获取内存，
                   或匹配失败代码
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_list_get(pcre2_match_data *match_data, PCRE2_UCHAR ***listptr,
  PCRE2_SIZE **lengthsptr)
{
// 定义变量
int i, count, count2;
PCRE2_SIZE size;
PCRE2_SIZE *lensp;
pcre2_memctl *memp;
PCRE2_UCHAR **listp;
PCRE2_UCHAR *sp;
PCRE2_SIZE *ovector;

// 如果匹配失败，返回 count
if ((count = match_data->rc) < 0) return count;   /* 匹配失败 */
// 如果 count 为 0，将 count 设置为匹配数据的 oveccount
if (count == 0) count = match_data->oveccount;    /* ovector 太小 */

count2 = 2*count;
ovector = match_data->ovector;
// 计算所需内存大小
size = sizeof(pcre2_memctl) + sizeof(PCRE2_UCHAR *);      /* 用于最终的 NULL */
if (lengthsptr != NULL) size += sizeof(PCRE2_SIZE) * count;  /* 用于长度 */
for (i = 0; i < count2; i += 2)
  {
  // 计算 size 的值，包括 PCRE2_UCHAR 指针的大小和 CU2BYTES(1) 的大小
  size += sizeof(PCRE2_UCHAR *) + CU2BYTES(1);
  // 如果当前捕获组的结束位置大于开始位置，计算并加上结束位置和开始位置之间的大小
  if (ovector[i+1] > ovector[i]) size += CU2BYTES(ovector[i+1] - ovector[i]);
  }

// 为 match_data 分配内存
memp = PRIV(memctl_malloc)(size, (pcre2_memctl *)match_data);
// 如果内存分配失败，返回 PCRE2_ERROR_NOMEMORY
if (memp == NULL) return PCRE2_ERROR_NOMEMORY;

// 设置 listptr 和 listp 指向内存块的起始位置
*listptr = listp = (PCRE2_UCHAR **)((char *)memp + sizeof(pcre2_memctl));
// 设置 lensp 指向 listp 之后的位置
lensp = (PCRE2_SIZE *)((char *)listp + sizeof(PCRE2_UCHAR *) * (count + 1));

// 如果 lengthsptr 为 NULL，设置 sp 指向 lensp，否则设置 *lengthsptr 指向 lensp
if (lengthsptr == NULL)
  {
  sp = (PCRE2_UCHAR *)lensp;
  lensp = NULL;
  }
else
  {
  *lengthsptr = lensp;
  sp = (PCRE2_UCHAR *)((char *)lensp + sizeof(PCRE2_SIZE) * count);
  }

for (i = 0; i < count2; i += 2)
  {
  // 计算当前捕获组的大小
  size = (ovector[i+1] > ovector[i])? (ovector[i+1] - ovector[i]) : 0;

  /* Size == 0 includes the case when the capture is unset. Avoid adding
  PCRE2_UNSET to match_data->subject because it overflows, even though with
  zero size calling memcpy() is harmless. */

  // 如果大小不为 0，将 match_data->subject 中捕获组的内容拷贝到 sp 中
  if (size != 0) memcpy(sp, match_data->subject + ovector[i], CU2BYTES(size));
  // 设置 *listp 指向 sp
  *listp++ = sp;
  // 如果 lensp 不为 NULL，设置 *lensp 指向 size
  if (lensp != NULL) *lensp++ = size;
  // 将 sp 移动到下一个位置
  sp += size;
  // 在 sp 的末尾添加一个空字符
  *sp++ = 0;
  }

// 设置 listp 的最后一个元素为 NULL
*listp = NULL;
// 返回 0
return 0;
}



/*************************************************
*   Free memory obtained by substring_list_get   *
*************************************************/

/*
Argument:     the result of a previous pcre2_substring_list_get()
Returns:      nothing
*/

// 释放由 pcre2_substring_list_get() 获得的内存
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_substring_list_free(PCRE2_SPTR *list)
{
// 如果 list 不为 NULL
if (list != NULL)
  {
  // 获取内存控制块的指针
  pcre2_memctl *memctl = (pcre2_memctl *)((char *)list - sizeof(pcre2_memctl));
  // 释放内存
  memctl->free(memctl, memctl->memory_data);
  }
}



/*************************************************
*     Find (multiple) entries for named string   *
*************************************************/

/* This function scans the nametable for a given name, using binary chop. It
returns either two pointers to the entries in the table, or, if no pointers are
given, the number of a unique group with the given name. If duplicate names are
/*
  搜索给定名称的字符串在编译后的正则表达式中的位置
  如果名称不唯一或者不允许，将生成错误

  参数：
    code        编译后的正则表达式
    stringname  需要查找的名称
    firstptr    指向第一个条目的指针
    lastptr     指向最后一个条目的指针

  返回值：      
    如果未找到名称，则返回 PCRE2_ERROR_NOSUBSTRING
    否则，如果 firstptr 和 lastptr 为 NULL：
      对于唯一的子字符串，返回一个组号
      否则返回 PCRE2_ERROR_NOUNIQUESUBSTRING
    否则：
      设置 firstptr 和 lastptr 后，返回每个条目的长度
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_nametable_scan(const pcre2_code *code, PCRE2_SPTR stringname,
  PCRE2_SPTR *firstptr, PCRE2_SPTR *lastptr)
{
  uint16_t bot = 0;
  uint16_t top = code->name_count;
  uint16_t entrysize = code->name_entry_size;
  PCRE2_SPTR nametable = (PCRE2_SPTR)((char *)code + sizeof(pcre2_real_code));

  while (top > bot)
  {
    uint16_t mid = (top + bot) / 2;
    PCRE2_SPTR entry = nametable + entrysize*mid;
    int c = PRIV(strcmp)(stringname, entry + IMM2_SIZE);
    if (c == 0)
    {
      PCRE2_SPTR first;
      PCRE2_SPTR last;
      PCRE2_SPTR lastentry;
      lastentry = nametable + entrysize * (code->name_count - 1);
      first = last = entry;
      while (first > nametable)
      {
        if (PRIV(strcmp)(stringname, (first - entrysize + IMM2_SIZE)) != 0) break;
        first -= entrysize;
      }
      while (last < lastentry)
      {
        if (PRIV(strcmp)(stringname, (last + entrysize + IMM2_SIZE)) != 0) break;
        last += entrysize;
      }
      if (firstptr == NULL) return (first == last)? (int)GET2(entry, 0) : PCRE2_ERROR_NOUNIQUESUBSTRING;
      *firstptr = first;
      *lastptr = last;
      return entrysize;
    }
    if (c > 0) bot = mid + 1; else top = mid;
  }

  return PCRE2_ERROR_NOSUBSTRING;
}
# 这个函数是 pcre2_substring_nametable_scan() 的一个便利包装器，当已知名称是唯一的时候使用。如果存在重复的名称，则不定义返回哪个编号。

# 参数：
#   code        编译后的正则表达式
#   stringname  需要获取编号的名称

# 返回值：      名为 stringname 的括号的编号，或者一个负数
#                PCRE2_ERROR_NOSUBSTRING 如果未找到
#                PCRE2_ERROR_NOUNIQUESUBSTRING 如果不唯一
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_number_from_name(const pcre2_code *code,
  PCRE2_SPTR stringname)
{
    # 调用 pcre2_substring_nametable_scan() 函数，获取名称为 stringname 的括号的编号
    return pcre2_substring_nametable_scan(code, stringname, NULL, NULL);
}

# pcre2_substring.c 文件的结束
```