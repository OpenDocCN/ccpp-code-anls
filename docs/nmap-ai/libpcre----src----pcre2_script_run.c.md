# `nmap\libpcre\src\pcre2_script_run.c`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
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
/*
This module contains the function for checking a script run.
*/
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/*************************************************
*                Check script run                *
*************************************************/

/*
A script run is conceptually a sequence of characters all in the same
Unicode script. However, it isn't quite that simple. There are special rules
for scripts that are commonly used together, and also special rules for digits.
This function implements the appropriate checks, which is possible only when
PCRE2 is compiled with Unicode support. The function returns TRUE if there is
no Unicode support; however, it should never be called in that circumstance
because an error is given by pcre2_compile() if a script run is called for in a
version of PCRE2 compiled without Unicode support.

Arguments:
  pgr       point to the first character
  endptr    point after the last character
  utf       TRUE if in UTF mode

Returns:    TRUE if this is a valid script run
*/

/* These are states in the checking process. */

enum { SCRIPT_UNSET,          /* Requirement as yet unknown */
       SCRIPT_MAP,            /* Bitmap contains acceptable scripts */
       SCRIPT_HANPENDING,     /* Have had only Han characters */
       SCRIPT_HANHIRAKATA,    /* Expect Han or Hirikata */
       SCRIPT_HANBOPOMOFO,    /* Expect Han or Bopomofo */
       SCRIPT_HANHANGUL       /* Expect Han or Hangul */
       };

#define UCD_MAPSIZE (ucp_Unknown/32 + 1)
#define FULL_MAPSIZE (ucp_Script_Count/32 + 1)

BOOL
PRIV(script_run)(PCRE2_SPTR ptr, PCRE2_SPTR endptr, BOOL utf)
{
#ifdef SUPPORT_UNICODE
uint32_t require_state = SCRIPT_UNSET;  /* Initialize the state for script run requirement */
uint32_t require_map[FULL_MAPSIZE];     /* Initialize the map for required scripts */
uint32_t map[FULL_MAPSIZE];              /* Initialize the map for scripts */
// 定义一个需要数字集的变量，初始值为0
uint32_t require_digitset = 0;
// 定义一个字符变量c
uint32_t c;

// 如果PCRE2_CODE_UNIT_WIDTH等于32，则避免编译器警告
#if PCRE2_CODE_UNIT_WIDTH == 32
(void)utf;    /* Avoid compiler warning */
#endif

// 任何包含少于2个字符的字符串都是有效的脚本运行
if (ptr >= endptr) return TRUE;
// 从ptr指向的位置获取一个字符并将其存储在c中
GETCHARINCTEST(c, ptr);
if (ptr >= endptr) return TRUE;

// 初始化需要的映射。这是一个完整大小的位图，对于每个脚本都有一个位，而不是ucd_script_sets中的映射，它只对小于ucp_Unknown的脚本有位 - 这些脚本出现在脚本扩展列表中。
for (int i = 0; i < FULL_MAPSIZE; i++) require_map[i] = 0;

// 扫描两个或更多字符的字符串，检查每个代码点的Unicode特性。对于可以与汉字脚本的字符组合的脚本，有特殊的代码。这可能与这些组合中的其他四个脚本一起使用：
// . 汉字与平假名和片假名允许（用于日语）。
// . 汉字与注音符号允许（用于台湾普通话）。
// . 汉字与韩文允许（用于韩语）。
// 如果第一个显著字符的脚本是其中四个之一，则立即知道所需的脚本类型。但是，如果第一个显著字符的脚本是汉字，则必须继续检查非汉字字符。因此有SCRIPT_HANPENDING状态。
for (;;)
  {
  const ucd_record *ucd = GET_UCD(c);
  uint32_t script = ucd->script;

  // 如果脚本是Unknown，则字符串不是有效的脚本运行。这样的字符只能形成长度为一的脚本运行（见上面的测试）。
  if (script == ucp_Unknown) return FALSE;

  // 没有任何脚本扩展且脚本为Inherited或Common的字符始终接受任何脚本。如果有扩展，则对所有脚本进行以下处理。
  if (UCD_SCRIPTX_PROP(ucd) != 0 || (script != ucp_Inherited && script != ucp_Common))
    {
    BOOL OK;

    // 为此字符设置一个全尺寸的映射，可以包括所有位
    # 复制脚本映射以覆盖此字符的脚本扩展列表中出现的脚本，将剩余值设置为零，然后，除了Common或Inherited之外，将此脚本的位添加到映射中。
    memcpy(map, PRIV(ucd_script_sets) + UCD_SCRIPTX_PROP(ucd), UCD_MAPSIZE * sizeof(uint32_t));
    memset(map + UCD_MAPSIZE, 0, (FULL_MAPSIZE - UCD_MAPSIZE) * sizeof(uint32_t));
    if (script != ucp_Common && script != ucp_Inherited) MAPSET(map, script);

    # 处理不同的检查状态
    switch(require_state)
      {
      # 第一个显著字符 - 它可能跟在没有任何脚本扩展的Common或Inherited字符后面。
      case SCRIPT_UNSET:
      switch(script)
        {
        case ucp_Han:
        require_state = SCRIPT_HANPENDING;
        break;

        case ucp_Hiragana:
        case ucp_Katakana:
        require_state = SCRIPT_HANHIRAKATA;
        break;

        case ucp_Bopomofo:
        require_state = SCRIPT_HANBOPOMOFO;
        break;

        case ucp_Hangul:
        require_state = SCRIPT_HANHANGUL;
        break;

        default:
        memcpy(require_map, map, FULL_MAPSIZE * sizeof(uint32_t));
        require_state = SCRIPT_MAP;
        break;
        }
      break;

      # 第一个显著字符是汉字。检查Unicode 11.0.0文件显示涉及汉字、注音符号、平假名、片假名和韩文脚本的以下类型的脚本扩展列表：
      # . 注音符号 + 汉字
      # . 汉字 + 平假名 + 片假名
      # . 平假名 + 片假名
      # . 注音符号 + 韩文 + 汉字 + 平假名 + 片假名
      # 以下代码试图理解这一点。
#define FOUND_BOPOMOFO 1
#define FOUND_HIRAGANA 2
#define FOUND_KATAKANA 4
    }   /* End checking character's script and extensions. */

  /* The character is in an acceptable script. We must now ensure that all
  decimal digits in the string come from the same set. Some scripts (e.g.
  Common, Arabic) have more than one set of decimal digits. This code does
  not allow mixing sets, even within the same script. The vector called
  PRIV(ucd_digit_sets)[] contains, in its first element, the number of
  following elements, and then, in ascending order, the code points of the
  '9' characters in every set of 10 digits. Each set is identified by the
  offset in the vector of its '9' character. An initial check of the first
  value picks up ASCII digits quickly. Otherwise, a binary chop is used. */

  if (ucd->chartype == ucp_Nd)
    {
    uint32_t digitset;

    if (c <= PRIV(ucd_digit_sets)[1]) digitset = 1; else
      {
      int mid;
      int bot = 1;
      int top = PRIV(ucd_digit_sets)[0];
      for (;;)
        {
        if (top <= bot + 1)    /* <= rather than == is paranoia */
          {
          digitset = top;
          break;
          }
        mid = (top + bot) / 2;
        if (c <= PRIV(ucd_digit_sets)[mid]) top = mid; else bot = mid;
        }
      }

    /* A required value of 0 means "unset". */

    if (require_digitset == 0) require_digitset = digitset;
      else if (digitset != require_digitset) return FALSE;
    }   /* End digit handling */

  /* If we haven't yet got to the end, pick up the next character. */

  if (ptr >= endptr) return TRUE;
  GETCHARINCTEST(c, ptr);
  }  /* End checking loop */

#else   /* NOT SUPPORT_UNICODE */
(void)ptr;
(void)endptr;
(void)utf;
return TRUE;
#endif  /* SUPPORT_UNICODE */
}

/* End of pcre2_script_run.c */
```