# `nmap\libpcre\src\pcre2_study.c`

```cpp
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
/* This module contains functions for scanning a compiled pattern and
collecting data (e.g. minimum matching length). */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/* The maximum remembered capturing brackets minimum. */
#define MAX_CACHE_BACKREF 128

/* Set a bit in the starting code unit bit map. */
#define SET_BIT(c) re->start_bitmap[(c)/8] |= (1u << ((c)&7))

/* Returns from set_start_bits() */
enum { SSB_FAIL, SSB_DONE, SSB_CONTINUE, SSB_UNKNOWN, SSB_TOODEEP };

/*************************************************
*   Find the minimum subject length for a group  *
*************************************************/

/* Scan a parenthesized group and compute the minimum length of subject that
is needed to match it. This is a lower bound; it does not mean there is a
string of that length that matches. In UTF mode, the result is in characters
rather than code units. The field in a compiled pattern for storing the minimum
length is 16-bits long (on the grounds that anything longer than that is
pathological), so we give up when we reach that amount. This also means that
integer overflow for really crazy patterns cannot happen.

Backreference minimum lengths are cached to speed up multiple references. This
function is called only when the highest back reference in the pattern is less
than or equal to MAX_CACHE_BACKREF, which is one less than the size of the
caching vector. The zeroth element contains the number of the highest set
value.
/*
  查找给定正则表达式的最小长度
  re: 编译后的模式块
  code: 指向组的起始位置（括号）
  startcode: 整个模式代码的起始位置指针
  utf: UTF 标志
  recurses: 用于捕获相互递归的递归检查链
  countptr: 调用计数的指针（用于捕获过于复杂的情况）
  backref_cache: 用于缓存反向引用的向量

  当模式包含 (*ACCEPT) 时，不再调用此函数；但是保留了返回 -1 的旧代码，以防万一。

  返回值：
  最小长度
  -1：在 UTF-8 模式下的 \C 或 (*ACCEPT) 或模式过于复杂
  -2：内部错误（缺少捕获括号）
  -3：内部错误（未列出的操作码）
*/

static int
find_minlength(const pcre2_real_code *re, PCRE2_SPTR code,
  PCRE2_SPTR startcode, BOOL utf, recurse_check *recurses, int *countptr,
  int *backref_cache)
{
  int length = -1;
  int branchlength = 0;
  int prev_cap_recno = -1;
  int prev_cap_d = 0;
  int prev_recurse_recno = -1;
  int prev_recurse_d = 0;
  uint32_t once_fudge = 0;
  BOOL had_recurse = FALSE;
  BOOL dupcapused = (re->flags & PCRE2_DUPCAPUSED) != 0;
  PCRE2_SPTR nextbranch = code + GET(code, 1);
  PCRE2_UCHAR *cc = (PCRE2_UCHAR *)code + 1 + LINK_SIZE;
  recurse_check this_recurse;

  // 如果这是一个“可能为空”的组，其最小长度为 0
  if (*code >= OP_SBRA && *code <= OP_SCOND) return 0;

  // 跳过捕获括号编号
  if (*code == OP_CBRA || *code == OP_CBRAPOS) cc += IMM2_SIZE;

  // 大型和/或复杂的正则表达式可能处理时间过长
  if ((*countptr)++ > 1000) return -1;

  // 遍历此分支的操作码。如果到达分支的末尾，检查累积长度是否超过 16 位，如果超过，则重置为该值并跳过分支的剩余部分
  for (;;)
  {
    int d, min, recno;
    PCRE2_UCHAR op, *cs, *ce;

    if (branchlength >= UINT16_MAX)
    {
    # 设置分支长度的最大值为 UINT16_MAX
    branchlength = UINT16_MAX;
    # 将 nextbranch 转换为 PCRE2_UCHAR 类型的指针
    cc = (PCRE2_UCHAR *)nextbranch;
    }

  # 获取下一个操作码
  op = *cc;
  # 根据操作码进行不同的处理
  switch (op)
    {
    # 如果操作码是 OP_COND 或者 OP_SCOND
    case OP_COND:
    case OP_SCOND:

    /* 如果条件中只有一个分支，则隐含的分支长度为零，所以我们不需要添加任何内容。
    这自动处理了 DEFINE "condition"。如果有两个分支，我们可以将其视为任何其他非捕获子模式。 */

    # 如果条件中只有一个分支，则隐含的分支长度为零，所以我们不需要添加任何内容。
    # 这自动处理了 DEFINE "condition"。如果有两个分支，我们可以将其视为任何其他非捕获子模式。
    cs = cc + GET(cc, 1);
    if (*cs != OP_ALT)
      {
      cc = cs + 1 + LINK_SIZE;
      break;
      }
    # 跳转到处理非捕获子模式的部分
    goto PROCESS_NON_CAPTURE;

    # 如果操作码是 OP_BRA
    case OP_BRA:
    # 如果 OP_BRA 包裹着一个重复的 OP_RECURSE 的特殊情况
    if (cc[1+LINK_SIZE] == OP_RECURSE && cc[2*(1+LINK_SIZE)] == OP_KET)
      {
      # 设置一个虚拟值以跳过 OP_KET 后的 OP_RECURSE
      once_fudge = 1 + LINK_SIZE;
      cc += 1 + LINK_SIZE;
      break;
      }
    # 如果不是特殊情况，继续执行下面的操作
    # 继续执行下面的操作
    # 如果操作码是 OP_ONCE、OP_SCRIPT_RUN、OP_SBRA、OP_BRAPOS、OP_SBRAPOS
    PROCESS_NON_CAPTURE:
    # 查找当前子模式的最小长度
    d = find_minlength(re, cc, startcode, utf, recurses, countptr,
      backref_cache);
    # 如果查找失败，返回错误码
    if (d < 0) return d;
    # 更新分支长度
    branchlength += d;
    # 跳过当前分支的所有操作码
    do cc += GET(cc, 1); while (*cc == OP_ALT);
    cc += 1 + LINK_SIZE;
    break;

    # 如果操作码是 OP_CBRA、OP_SCBRA、OP_CBRAPOS、OP_SCBRAPOS
    # 获取捕获子模式的编号
    recno = (int)GET2(cc, 1+LINK_SIZE);
    # 如果重复捕获被使用，或者记录号不等于前一个捕获的记录号
    if (dupcapused || recno != prev_cap_recno)
      {
      # 将前一个捕获的记录号设置为当前记录号
      prev_cap_recno = recno;
      # 查找最小长度的子模式
      prev_cap_d = find_minlength(re, cc, startcode, utf, recurses, countptr,
        backref_cache);
      # 如果找不到最小长度，返回错误
      if (prev_cap_d < 0) return prev_cap_d;
      }
    # 分支长度增加前一个捕获的长度
    branchlength += prev_cap_d;
    # 跳过当前分支的所有操作码
    do cc += GET(cc, 1); while (*cc == OP_ALT);
    # 移动到下一个操作码
    cc += 1 + LINK_SIZE;
    # 跳出循环

    /* ACCEPT makes things far too complicated; we have to give up. In fact,
    from 10.34 onwards, if a pattern contains (*ACCEPT), this function is not
    used. However, leave the code in place, just in case. */

    # 如果遇到 ACCEPT 操作码，或者 ASSERT_ACCEPT 操作码，返回错误
    case OP_ACCEPT:
    case OP_ASSERT_ACCEPT:
    return -1;

    /* Reached end of a branch; if it's a ket it is the end of a nested
    call. If it's ALT it is an alternation in a nested call. If it is END it's
    the end of the outer call. All can be handled by the same code. If the
    length of any branch is zero, there is no need to scan any subsequent
    branches. */

    # 如果遇到 ALT、KET、KETRMAX、KETRMIN、KETRPOS、END 操作码
    case OP_ALT:
    case OP_KET:
    case OP_KETRMAX:
    case OP_KETRMIN:
    case OP_KETRPOS:
    case OP_END:
    # 如果长度小于0，或者（没有递归且分支长度小于长度）
    if (length < 0 || (!had_recurse && branchlength < length))
      # 将长度设置为分支长度
      length = branchlength;
    # 如果不是 ALT 操作码，或者长度为0，返回长度
    if (op != OP_ALT || length == 0) return length;
    # 移动到下一个分支
    nextbranch = cc + GET(cc, 1);
    cc += 1 + LINK_SIZE;
    # 重置分支长度和递归标志
    branchlength = 0;
    had_recurse = FALSE;
    # 跳出循环

    /* Skip over assertive subpatterns */

    # 跳过断言子模式
    case OP_ASSERT:
    case OP_ASSERT_NOT:
    case OP_ASSERTBACK:
    case OP_ASSERTBACK_NOT:
    case OP_ASSERT_NA:
    case OP_ASSERTBACK_NA:
    do cc += GET(cc, 1); while (*cc == OP_ALT);
    /* Fall through */

    /* Skip over things that don't match chars */

    # 跳过不匹配字符的操作码
    case OP_REVERSE:
    case OP_CREF:
    case OP_DNCREF:
    case OP_RREF:
    case OP_DNRREF:
    case OP_FALSE:
    case OP_TRUE:
    case OP_CALLOUT:
    case OP_SOD:
    case OP_SOM:
    case OP_EOD:
    case OP_EODN:
    case OP_CIRC:
    case OP_CIRCM:
    case OP_DOLL:
    case OP_DOLLM:
    case OP_NOT_WORD_BOUNDARY:
    case OP_WORD_BOUNDARY:
    # 将当前位置的指针移动到下一个操作码的位置
    cc += PRIV(OP_lengths)[*cc];
    # 跳出当前的 switch 语句
    break;

    # 处理 OP_CALLOUT_STR 操作码
    case OP_CALLOUT_STR:
    # 将当前位置的指针移动到下一个操作码的位置
    cc += GET(cc, 1 + 2*LINK_SIZE);
    # 跳出当前的 switch 语句
    break;

    # 处理带有 {0} 或 {0,x} 量词的子模式
    case OP_BRAZERO:
    case OP_BRAMINZERO:
    case OP_BRAPOSZERO:
    case OP_SKIPZERO:
    # 将当前位置的指针移动到下一个操作码的位置
    cc += PRIV(OP_lengths)[*cc];
    # 当前位置的指针移动到下一个操作码的位置，直到遇到 OP_ALT 操作码
    do cc += GET(cc, 1); while (*cc == OP_ALT);
    # 将当前位置的指针移动到下一个操作码的位置
    cc += 1 + LINK_SIZE;
    # 跳出当前的 switch 语句
    break;

    # 处理字面字符和 + 重复
    case OP_CHAR:
    case OP_CHARI:
    case OP_NOT:
    case OP_NOTI:
    case OP_PLUS:
    case OP_PLUSI:
    case OP_MINPLUS:
    case OP_MINPLUSI:
    case OP_POSPLUS:
    case OP_POSPLUSI:
    case OP_NOTPLUS:
    case OP_NOTPLUSI:
    case OP_NOTMINPLUS:
    case OP_NOTMINPLUSI:
    case OP_NOTPOSPLUS:
    case OP_NOTPOSPLUSI:
    # 增加分支长度
    branchlength++;
    # 将当前位置的指针移动到下一个操作码的位置
    cc += 2;
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，并且当前字符是多字节字符的一部分，跳过该字符
    if (utf && HAS_EXTRALEN(cc[-1])) cc += GET_EXTRALEN(cc[-1]);
#endif
    # 跳出当前的 switch 语句
    break;

    case OP_TYPEPLUS:
    case OP_TYPEMINPLUS:
    case OP_TYPEPOSPLUS:
    # 分支长度加一
    branchlength++;
    # 如果下一个字符是 OP_PROP 或者 OP_NOTPROP，则跳过 4 个字符，否则跳过 2 个字符
    cc += (cc[1] == OP_PROP || cc[1] == OP_NOTPROP)? 4 : 2;
    # 跳出当前的 switch 语句
    break;

    /* 处理精确重复。计数已经是字符数，但是在 UTF 模式下可能需要跳过多字节字符。 */

    case OP_EXACT:
    case OP_EXACTI:
    case OP_NOTEXACT:
    case OP_NOTEXACTI:
    # 分支长度加上指定的字符数
    branchlength += GET2(cc,1);
    # 跳过 2 个字符和 IMM2_SIZE 个字符
    cc += 2 + IMM2_SIZE;
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，并且当前字符是多字节字符的一部分，跳过该字符
    if (utf && HAS_EXTRALEN(cc[-1])) cc += GET_EXTRALEN(cc[-1]);
#endif
    # 跳出当前的 switch 语句
    break;

    case OP_TYPEEXACT:
    # 分支长度加上指定的字符数
    branchlength += GET2(cc,1);
    # 跳过 2 个字符、IMM2_SIZE 个字符以及额外的 2 个字符（如果下一个字符是 OP_PROP 或者 OP_NOTPROP）
    cc += 2 + IMM2_SIZE + ((cc[1 + IMM2_SIZE] == OP_PROP
      || cc[1 + IMM2_SIZE] == OP_NOTPROP)? 2 : 0);
    # 跳出当前的 switch 语句
    break;

    /* 处理单字符非字面匹配器 */

    case OP_PROP:
    case OP_NOTPROP:
    # 跳过 2 个字符
    cc += 2;
    # 继续执行下面的 case 语句
    /* Fall through */

    case OP_NOT_DIGIT:
    case OP_DIGIT:
    case OP_NOT_WHITESPACE:
    case OP_WHITESPACE:
    case OP_NOT_WORDCHAR:
    case OP_WORDCHAR:
    case OP_ANY:
    case OP_ALLANY:
    case OP_EXTUNI:
    case OP_HSPACE:
    case OP_NOT_HSPACE:
    case OP_VSPACE:
    case OP_NOT_VSPACE:
    # 分支长度加一
    branchlength++;
    # 跳过一个字符
    cc++;
    # 跳出当前的 switch 语句
    break;

    /* "Any newline" 可能匹配两个字符，但也可能只匹配一个。 */

    case OP_ANYNL:
    # 分支长度加一
    branchlength += 1;
    # 跳过一个字符
    cc++;
    # 跳出当前的 switch 语句
    break;

    /* 单字节匹配器意味着我们不能在 UTF 模式下继续。 (在非 UTF 模式下，\C 实际上会被转换为 OP_ALLANY，所以不会出现，但是留下代码，以防万一。) */

    case OP_ANYBYTE:
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode 并且当前是 UTF 模式，则返回 -1
    if (utf) return -1;
#endif
    # 分支长度加一
    branchlength++;
    # 跳过一个字符
    cc++;
    # 跳出当前的 switch 语句
    break;

    /* 对于重复的字符类型，我们必须测试 \p 和 \P，它们有额外的两个参数字节。 */

    case OP_TYPESTAR:
    case OP_TYPEMINSTAR:
    case OP_TYPEQUERY:
    case OP_TYPEMINQUERY:
    # 如果操作码是 OP_TYPEPOSSTAR 或者 OP_TYPEPOSQUERY，则跳过下一个字符，如果是 OP_PROP 或者 OP_NOTPROP，则再跳过一个字符
    case OP_TYPEPOSSTAR:
    case OP_TYPEPOSQUERY:
    if (cc[1] == OP_PROP || cc[1] == OP_NOTPROP) cc += 2;
    # 根据操作码的长度移动指针
    cc += PRIV(OP_lengths)[op];
    break;

    # 如果操作码是 OP_TYPEUPTO 或者 OP_TYPEMINUPTO 或者 OP_TYPEPOSUPTO，则跳过下一个字符，如果是 OP_PROP 或者 OP_NOTPROP，则再跳过一个字符
    case OP_TYPEUPTO:
    case OP_TYPEMINUPTO:
    case OP_TYPEPOSUPTO:
    if (cc[1 + IMM2_SIZE] == OP_PROP
      || cc[1 + IMM2_SIZE] == OP_NOTPROP) cc += 2;
    # 根据操作码的长度移动指针
    cc += PRIV(OP_lengths)[op];
    break;

    # 检查类是否具有可变量化
    case OP_CLASS:
    case OP_NCLASS:
#ifdef SUPPORT_WIDE_CHARS
    # 如果支持宽字符
    case OP_XCLASS:
    # 如果操作码是 OP_XCLASS
    /* The original code caused an unsigned overflow in 64 bit systems,
    so now we use a conditional statement. */
    # 原始代码在 64 位系统中导致无符号溢出，所以现在我们使用条件语句
    if (op == OP_XCLASS)
      # 如果操作码是 OP_XCLASS
      cc += GET(cc, 1);
      # cc 值加上 GET(cc, 1) 的值
    else
      # 否则
      cc += PRIV(OP_lengths)[OP_CLASS];
      # cc 值加上 PRIV(OP_lengths)[OP_CLASS] 的值
#else
    # 如果不支持宽字符
    cc += PRIV(OP_lengths)[OP_CLASS];
    # cc 值加上 PRIV(OP_lengths)[OP_CLASS] 的值
#endif

    switch (*cc)
      # 根据 *cc 的值进行切换
      {
      case OP_CRPLUS:
      case OP_CRMINPLUS:
      case OP_CRPOSPLUS:
      branchlength++;
      # branchlength 值加一
      /* Fall through */
      # 继续执行下面的 case

      case OP_CRSTAR:
      case OP_CRMINSTAR:
      case OP_CRQUERY:
      case OP_CRMINQUERY:
      case OP_CRPOSSTAR:
      case OP_CRPOSQUERY:
      cc++;
      # cc 值加一
      break;

      case OP_CRRANGE:
      case OP_CRMINRANGE:
      case OP_CRPOSRANGE:
      branchlength += GET2(cc,1);
      # branchlength 值加上 GET2(cc,1) 的值
      cc += 1 + 2 * IMM2_SIZE;
      # cc 值加上 1 加上 2 乘以 IMM2_SIZE 的值
      break;

      default:
      branchlength++;
      # branchlength 值加一
      break;
      }
    break;

    /* Backreferences and subroutine calls (OP_RECURSE) are treated in the same
    way: we find the minimum length for the subpattern. A recursion
    (backreference or subroutine) causes an a flag to be set that causes the
    length of this branch to be ignored. The logic is that a recursion can only
    make sense if there is another alternative that stops the recursing. That
    will provide the minimum length (when no recursion happens).

    If PCRE2_MATCH_UNSET_BACKREF is set, a backreference to an unset bracket
    matches an empty string (by default it causes a matching failure), so in
    that case we must set the minimum length to zero.

    For backreferenes, if duplicate numbers are present in the pattern we check
    for a reference to a duplicate. If it is, we don't know which version will
    be referenced, so we have to set the minimum length to zero. */

    /* Duplicate named pattern back reference. */

    case OP_DNREF:
    case OP_DNREFI:
    # 如果操作码是 OP_DNREF 或者 OP_DNREFI
    # 如果未使用重复捕获并且未设置 PCRE2_MATCH_UNSET_BACKREF 标志
    if (!dupcapused && (re->overall_options & PCRE2_MATCH_UNSET_BACKREF) == 0)
      {
      # 从字节码中获取计数值
      int count = GET2(cc, 1+IMM2_SIZE);
      # 获取指向组名的指针
      PCRE2_UCHAR *slot =
        (PCRE2_UCHAR *)((uint8_t *)re + sizeof(pcre2_real_code)) +
          GET2(cc, 1) * re->name_entry_size;

      # 初始化最短长度为最大整数
      d = INT_MAX;

      # 扫描所有具有相同名称的组；找到最短的
      while (count-- > 0)
        {
        int dd, i;
        # 获取记录号
        recno = GET2(slot, 0);

        # 如果记录号小于等于 backref_cache[0] 并且 backref_cache[recno] 大于等于 0
        if (recno <= backref_cache[0] && backref_cache[recno] >= 0)
          dd = backref_cache[recno];
        else
          {
          # 获取指向组的指针
          ce = cs = (PCRE2_UCHAR *)PRIV(find_bracket)(startcode, utf, recno);
          # 如果指针为空，则返回 -2
          if (cs == NULL) return -2;
          # 移动指针直到遇到 OP_ALT 操作码
          do ce += GET(ce, 1); while (*ce == OP_ALT);

          # 初始化 dd 为 0
          dd = 0;
          # 如果未使用重复捕获或者在 ce 中未找到组
          if (!dupcapused ||
              (PCRE2_UCHAR *)PRIV(find_bracket)(ce, utf, recno) == NULL)
            {
            # 如果 cc 在 cs 和 ce 之间，表示简单递归
            if (cc > cs && cc < ce)
              {
              had_recurse = TRUE;
              }
            else
              {
              # 检查是否存在相互递归
              recurse_check *r = recurses;
              for (r = recurses; r != NULL; r = r->prev)
                if (r->group == cs) break;
              if (r != NULL)           
                {
                had_recurse = TRUE;
                }
              else
                {
                # 递归调用 find_minlength 函数，获取最小长度
                this_recurse.prev = recurses;  
                this_recurse.group = cs;
                dd = find_minlength(re, cs, startcode, utf, &this_recurse,
                  countptr, backref_cache);
                if (dd < 0) return dd;
                }
              }
            }

          # 缓存最小长度
          backref_cache[recno] = dd;
          for (i = backref_cache[0] + 1; i < recno; i++) backref_cache[i] = -1;
          backref_cache[0] = recno;
          }

        # 如果 dd 小于 d，则将 d 赋值为 dd
        if (dd < d) d = dd;
        # 如果 d 小于等于 0，则跳出循环
        if (d <= 0) break;    
        # 移动指针到下一个组名
        slot += re->name_entry_size;
        }
      }
    else d = 0;
    # 增加 cc 的值，用于跳过当前 back reference 的位置
    cc += 1 + 2*IMM2_SIZE;
    # 跳转到 REPEAT_BACK_REFERENCE 标签处

    # 单个通过编号进行反向引用。当没有重复时，通过名称进行的引用会转换为通过编号进行的引用。
    case OP_REF:
    case OP_REFI:
    # 获取反向引用的编号
    recno = GET2(cc, 1);
    # 如果反向引用编号小于等于 backref_cache[0] 并且 backref_cache[recno] 大于等于 0
    if (recno <= backref_cache[0] && backref_cache[recno] >= 0)
      # 将 d 设置为 backref_cache[recno]
      d = backref_cache[recno];
    else
      {
      int i;
      d = 0;

      # 如果未设置 PCRE2_MATCH_UNSET_BACKREF 选项
      if ((re->overall_options & PCRE2_MATCH_UNSET_BACKREF) == 0)
        {
        # 在 startcode 中查找括号，返回括号的起始位置
        ce = cs = (PCRE2_UCHAR *)PRIV(find_bracket)(startcode, utf, recno);
        # 如果未找到括号，返回 -2
        if (cs == NULL) return -2;
        # 循环直到找到下一个分支操作符
        do ce += GET(ce, 1); while (*ce == OP_ALT);

        # 如果未使用重复捕获组，或者在 ce 中未找到括号
        if (!dupcapused ||
            (PCRE2_UCHAR *)PRIV(find_bracket)(ce, utf, recno) == NULL)
          {
          # 如果 cc 大于 cs 并且小于 ce，表示简单递归
          if (cc > cs && cc < ce)
            {
            had_recurse = TRUE;
            }
          else
            {
            # 检查是否存在相互递归
            recurse_check *r = recurses;
            for (r = recurses; r != NULL; r = r->prev) if (r->group == cs) break;
            if (r != NULL)
              {
              had_recurse = TRUE;
              }
            else
              {
              # 记录当前递归信息，计算最小长度
              this_recurse.prev = recurses;
              this_recurse.group = cs;
              d = find_minlength(re, cs, startcode, utf, &this_recurse, countptr,
                backref_cache);
              if (d < 0) return d;
              }
            }
          }

      # 将 d 存入 backref_cache[recno]
      backref_cache[recno] = d;
      # 将 backref_cache 中 recno 之后的元素设置为 -1
      for (i = backref_cache[0] + 1; i < recno; i++) backref_cache[i] = -1;
      # 更新 backref_cache[0] 为 recno
      backref_cache[0] = recno;
      }

    # 增加 cc 的值，用于跳过当前 back reference 的位置

    # 处理重复的反向引用
    REPEAT_BACK_REFERENCE:
    # 根据当前字符指针指向的字符进行不同的操作
    switch (*cc)
      {
      # 对于不同的操作码进行不同的处理
      case OP_CRSTAR:
      case OP_CRMINSTAR:
      case OP_CRQUERY:
      case OP_CRMINQUERY:
      case OP_CRPOSSTAR:
      case OP_CRPOSQUERY:
      # 对于这些操作码，将最小匹配长度设为0，并移动字符指针到下一个字符
      min = 0;
      cc++;
      break;

      case OP_CRPLUS:
      case OP_CRMINPLUS:
      case OP_CRPOSPLUS:
      # 对于这些操作码，将最小匹配长度设为1，并移动字符指针到下一个字符
      min = 1;
      cc++;
      break;

      case OP_CRRANGE:
      case OP_CRMINRANGE:
      case OP_CRPOSRANGE:
      # 对于这些操作码，从字符指针获取最小匹配长度，并移动字符指针到下一个字符
      min = GET2(cc, 1);
      cc += 1 + 2 * IMM2_SIZE;
      break;

      default:
      # 默认情况下，将最小匹配长度设为1
      min = 1;
      break;
      }

     /* Take care not to overflow: (1) min and d are ints, so check that their
     product is not greater than INT_MAX. (2) branchlength is limited to
     UINT16_MAX (checked at the top of the loop). */

    # 确保不会溢出：(1) min 和 d 都是整数，因此需要检查它们的乘积是否大于 INT_MAX。 (2) branchlength 限制在 UINT16_MAX 以内（在循环顶部进行了检查）。

    if ((d > 0 && (INT_MAX/d) < min) || UINT16_MAX - branchlength < min*d)
      # 如果满足条件，将 branchlength 设为 UINT16_MAX，否则将其增加 min * d
      branchlength = UINT16_MAX;
    else branchlength += min * d;
    break;

    /* Recursion always refers to the first occurrence of a subpattern with a
    given number. Therefore, we can always make use of caching, even when the
    pattern contains multiple subpatterns with the same number. */

    # 递归总是指向具有给定编号的子模式的第一次出现。因此，即使模式包含多个具有相同编号的子模式，我们也可以始终使用缓存。
    case OP_RECURSE:
    # 设置 cs 和 ce 指针指向 startcode 中的位置，并获取递归编号
    cs = ce = (PCRE2_UCHAR *)startcode + GET(cc, 1);
    recno = GET2(cs, 1+LINK_SIZE);
    # 如果递归编号与前一个递归编号相同，则将 branchlength 增加 prev_recurse_d
    if (recno == prev_recurse_recno)
      {
      branchlength += prev_recurse_d;
      }
    else
      {
      # 如果不是 OP_ALT，则进行简单递归
      do ce += GET(ce, 1); while (*ce == OP_ALT);
      # 如果当前位置在起始位置和结束位置之间，表示简单递归
      if (cc > cs && cc < ce)    /* Simple recursion */
        had_recurse = TRUE;
      else
        {
        # 检查是否存在相互递归
        recurse_check *r = recurses;
        for (r = recurses; r != NULL; r = r->prev) if (r->group == cs) break;
        if (r != NULL)          /* Mutual recursion */
          had_recurse = TRUE;
        else
          {
          # 记录当前递归的位置和相关信息
          this_recurse.prev = recurses;
          this_recurse.group = cs;
          prev_recurse_d = find_minlength(re, cs, startcode, utf, &this_recurse,
            countptr, backref_cache);
          # 如果查找最小长度失败，则返回错误码
          if (prev_recurse_d < 0) return prev_recurse_d;
          prev_recurse_recno = recno;
          branchlength += prev_recurse_d;
          }
        }
      }
    # 移动到下一个位置
    cc += 1 + LINK_SIZE + once_fudge;
    once_fudge = 0;
    break;

    /* Anything else does not or need not match a character. We can get the
    item's length from the table, but for those that can match zero occurrences
    of a character, we must take special action for UTF-8 characters. As it
    happens, the "NOT" versions of these opcodes are used at present only for
    ASCII characters, so they could be omitted from this list. However, in
    future that may change, so we include them here so as not to leave a
    gotcha for a future maintainer. */

    # 处理各种操作码
    case OP_UPTO:
    case OP_UPTOI:
    case OP_NOTUPTO:
    case OP_NOTUPTOI:
    case OP_MINUPTO:
    case OP_MINUPTOI:
    case OP_NOTMINUPTO:
    case OP_NOTMINUPTOI:
    case OP_POSUPTO:
    case OP_POSUPTOI:
    case OP_NOTPOSUPTO:
    case OP_NOTPOSUPTOI:

    case OP_STAR:
    case OP_STARI:
    case OP_NOTSTAR:
    case OP_NOTSTARI:
    case OP_MINSTAR:
    case OP_MINSTARI:
    case OP_NOTMINSTAR:
    case OP_NOTMINSTARI:
    case OP_POSSTAR:
    case OP_POSSTARI:
    case OP_NOTPOSSTAR:
    case OP_NOTPOSSTARI:

    case OP_QUERY:
    case OP_QUERYI:
    case OP_NOTQUERY:
    case OP_NOTQUERYI:
    case OP_MINQUERY:
    case OP_MINQUERYI:
    # 如果操作码是OP_NOTMINQUERY、OP_NOTMINQUERYI、OP_POSQUERY、OP_POSQUERYI、OP_NOTPOSQUERY、OP_NOTPOSQUERYI中的任意一个，则执行下面的代码
    cc += PRIV(OP_lengths)[op];
    # 将cc增加OP_lengths[op]的值
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，则执行以下代码
    if (utf && HAS_EXTRALEN(cc[-1])) cc += GET_EXTRALEN(cc[-1]);
#endif
    # 跳出当前循环
    break;

    /* 跳过以下操作码，但需要加上名称长度。 */

    case OP_MARK:
    case OP_COMMIT_ARG:
    case OP_PRUNE_ARG:
    case OP_SKIP_ARG:
    case OP_THEN_ARG:
    # 根据操作码长度和下一个字符的长度，移动指针
    cc += PRIV(OP_lengths)[op] + cc[1];
    # 跳出当前循环
    break;

    /* 其余操作码直接跳过。 */

    case OP_CLOSE:
    case OP_COMMIT:
    case OP_FAIL:
    case OP_PRUNE:
    case OP_SET_SOM:
    case OP_SKIP:
    case OP_THEN:
    # 根据操作码长度，移动指针
    cc += PRIV(OP_lengths)[op];
    # 跳出当前循环
    break;

    /* 不应该出现这种情况：我们明确列出所有操作码，以便在添加新操作码时正确考虑。 */

    default:
    # 返回错误码 -3
    return -3;
    }
  }
/* 控制永远不会到达这里 */
}



/*************************************************
*      Set a bit and maybe its alternate case    *
*************************************************/

/* 给定一个字符，设置表中第一个代码单元的位，如果是大小写不敏感，则也设置字母的另一个版本的对应位。

参数：
  re            指向正则表达式块
  p             指向字符的第一个代码单元
  caseless      如果大小写不敏感则为 TRUE
  utf           如果为 UTF 模式则为 TRUE
  ucp           如果为 UCP 模式则为 TRUE

返回：        字符后面的指针
*/

static PCRE2_SPTR
set_table_bit(pcre2_real_code *re, PCRE2_SPTR p, BOOL caseless, BOOL utf,
  BOOL ucp)
{
uint32_t c = *p++;   /* 第一个代码单元 */

(void)utf;           /* 当不支持 UTF 时停止编译器警告 */
(void)ucp;

/* 在 16 位和 32 位模式中，大于 0xff 的代码单元会设置 0xff 的位。 */

#if PCRE2_CODE_UNIT_WIDTH != 8
if (c > 0xff) SET_BIT(0xff); else
#endif

SET_BIT(c);

/* 在 UTF-8 或 UTF-16 模式中，即使大小写不敏感，也要按顺序获取剩余的代码单元以找到字符的结尾。 */

#ifdef SUPPORT_UNICODE
if (utf)
  {
#if PCRE2_CODE_UNIT_WIDTH == 8
  if (c >= 0xc0) GETUTF8INC(c, p);
#elif PCRE2_CODE_UNIT_WIDTH == 16
  # 如果 PCRE2_CODE_UNIT_WIDTH 等于 16
  if ((c & 0xfc00) == 0xd800) GETUTF16INC(c, p);
#endif
  }
#endif  /* SUPPORT_UNICODE */

/* 如果忽略大小写，处理字符的另一种情况。 */

if (caseless)
  {
#ifdef SUPPORT_UNICODE
  if (utf || ucp)
    {
    c = UCD_OTHERCASE(c);
    #if PCRE2_CODE_UNIT_WIDTH == 8
    if (utf)
      {
      PCRE2_UCHAR buff[6];
      (void)PRIV(ord2utf)(c, buff);
      SET_BIT(buff[0]);
      }
    else if (c < 256) SET_BIT(c);
    #else  /* 16-bit or 32-bit mode */
    if (c > 0xff) SET_BIT(0xff); else SET_BIT(c);
    #endif
    }

  else
#endif  /* SUPPORT_UNICODE */

  /* 非 UTF 或 UCP */

  if (MAX_255(c)) SET_BIT(re->tables[fcc_offset + c]);
  }

return p;
}



/*************************************************
*     Set bits for a positive character type     *
*************************************************/

/* 为正字符类型设置起始位。在 UTF-8 模式下，我们只能直接设置小于 128 的字节，否则可能会与 UTF-8 字符中间的字节混淆。在“传统”环境中，表只会识别 ASCII 字符，但在至少一个 Windows 环境中，表中的一些高字节位被设置了。因此，我们通过考虑 UTF-8 编码来处理这种情况。

参数：
  re             正则表达式块
  cbit type      所需字符类型
  table_limit    非 UTF-8 为 32；UTF-8 为 16

返回：         无
*/

static void
set_type_bits(pcre2_real_code *re, int cbit_type, unsigned int table_limit)
{
uint32_t c;
for (c = 0; c < table_limit; c++)
  re->start_bitmap[c] |= re->tables[c+cbits_offset+cbit_type];
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
if (table_limit == 32) return;
for (c = 128; c < 256; c++)
  {
  if ((re->tables[cbits_offset + c/8] & (1u << (c&7))) != 0)
    {
    PCRE2_UCHAR buff[6];
    (void)PRIV(ord2utf)(c, buff);
    SET_BIT(buff[0]);
    }
  }
#endif  /* UTF-8 */
}
/* 设置负字符类型的起始位，比如 \D。
在 UTF-8 模式下，我们只能直接设置小于 128 的字节，否则可能会与 UTF-8 字符中间的字节混淆。
与正字符类型不同，我们可以为特定高值的 UTF-8 字符设置适当的起始位，但在这种情况下，我们必须为所有高值字符设置位。最低位是 0xc2，但为简单起见，我们从 0xc0 (192) 开始。

参数：
  re             正则表达式块
  cbit type      所需字符类型
  table_limit    非 UTF-8 为 32；UTF-8 为 16

返回值：         无
*/

static void
set_nottype_bits(pcre2_real_code *re, int cbit_type, unsigned int table_limit)
{
uint32_t c;
for (c = 0; c < table_limit; c++)
  re->start_bitmap[c] |= (uint8_t)(~(re->tables[c+cbits_offset+cbit_type]));
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
if (table_limit != 32) for (c = 24; c < 32; c++) re->start_bitmap[c] = 0xff;
#endif
}



/* 创建起始码元的位图 */

/* 此函数递归扫描编译的非锚定表达式，并尝试构建一个小于 256 的可能起始码元集的位图。在 16 位和 32 位模式下，大于 255 的值都会导致设置 255 位。在 UTF-8 模式下调用 set[_not]_type_bits() 时，我们传递的最后一个参数值为 16 而不是 32。 (请参阅这些函数的注释原因。)

SSB_CONTINUE 返回值对于模式中的括号组非常有用，比如 (a*)b，其中组提供了一些可选的起始码元，但扫描必须继续在外部级别找到至少一个必需的码元。在
/*
  在最外层级别，除非结果是 SSB_DONE，否则此函数将失败。

  我们限制递归（用于嵌套组）为1000，以避免堆栈溢出问题。

  参数：
    re           指向编译后的正则表达式块
    code         指向一个表达式
    utf          如果处于 UTF 模式，则为 TRUE
    ucp          如果处于 UCP 模式，则为 TRUE
    depthptr     递归深度的指针

  返回值：       
               SSB_FAIL     => 未找到任何起始代码单元
               SSB_DONE     => 找到了必需的起始代码单元
               SSB_CONTINUE => 找到了可选的起始代码单元
               SSB_UNKNOWN  => 遇到了无法识别的操作码
               SSB_TOODEEP  => 递归太深
*/

static int
set_start_bits(pcre2_real_code *re, PCRE2_SPTR code, BOOL utf, BOOL ucp,
  int *depthptr)
{
  uint32_t c;
  int yield = SSB_DONE;

  #if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
  int table_limit = utf? 16:32;
  #else
  int table_limit = 32;
  #endif

  *depthptr += 1;
  if (*depthptr > 1000) return SSB_TOODEEP;

  do
    {
    BOOL try_next = TRUE;
    PCRE2_SPTR tcode = code + 1 + LINK_SIZE;

    if (*code == OP_CBRA || *code == OP_SCBRA ||
        *code == OP_CBRAPOS || *code == OP_SCBRAPOS) tcode += IMM2_SIZE;

    while (try_next)    /* 循环处理此分支中的项目 */
      {
      int rc;
      uint8_t *classmap = NULL;
      #ifdef SUPPORT_WIDE_CHARS
      PCRE2_UCHAR xclassflags;
      #endif

      #if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
            if (utf)
              {
              PCRE2_UCHAR buff[6];
              (void)PRIV(ord2utf)(c, buff);
              c = buff[0];
              }
      #if PCRE2_CODE_UNIT_WIDTH != 8
        SET_BIT(0xA0);
        SET_BIT(0xFF);
      #else
        /* 对于 UTF-8 模式下的 8 位库，设置水平空格字符的第一个代码单元的位。 */
      ```
     
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则设置特定的位
      if (utf)
        {
        SET_BIT(0xC2);  /* For U+00A0 */
        SET_BIT(0xE1);  /* For U+1680, U+180E */
        SET_BIT(0xE2);  /* For U+2000 - U+200A, U+202F, U+205F */
        SET_BIT(0xE3);  /* For U+3000 */
        }
      else
#endif
      /* 对于不在 UTF-8 模式下的 8 位库，设置 0xA0 的位，除非代码是 EBCDIC。 */
        {
#ifndef EBCDIC
        SET_BIT(0xA0);
#endif  /* Not EBCDIC */
        }
#endif  /* 8-bit support */

      # 设置 try_next 为 FALSE，并跳出循环
      try_next = FALSE;
      break;

      case OP_ANYNL:
      case OP_VSPACE:
      # 设置特定的位
      SET_BIT(CHAR_LF);
      SET_BIT(CHAR_VT);
      SET_BIT(CHAR_FF);
      SET_BIT(CHAR_CR);

      /* 对于 16 位和 32 位库（永远不可能是 EBCDIC），设置 NEL 的位和大于等于 255 的代码单元的位，与 UTF 无关。 */
#if PCRE2_CODE_UNIT_WIDTH != 8
      SET_BIT(CHAR_NEL);
      SET_BIT(0xFF);
#else
      /* 对于 UTF-8 模式下的 8 位库，设置垂直空格字符的第一个代码单元的位。 */
#ifdef SUPPORT_UNICODE
      if (utf)
        {
        SET_BIT(0xC2);  /* For U+0085 (NEL) */
        SET_BIT(0xE2);  /* For U+2028, U+2029 */
        }
      else
#endif
      /* 对于不在 UTF-8 模式下的 8 位库，设置 NEL 的位。 */
        {
        SET_BIT(CHAR_NEL);
        }
#if PCRE2_CODE_UNIT_WIDTH != 8
        SET_BIT(0xA0);
        SET_BIT(0xFF);
#else
        /* 对于 UTF-8 模式下的 8 位库，设置水平空格字符的第一个代码单元的位。 */
#ifdef SUPPORT_UNICODE
        if (utf)
          {
          SET_BIT(0xC2);  /* For U+00A0 */
          SET_BIT(0xE1);  /* For U+1680, U+180E */
          SET_BIT(0xE2);  /* For U+2000 - U+200A, U+202F, U+205F */
          SET_BIT(0xE3);  /* For U+3000 */
          }
        else
#endif
        /* 对于不在 UTF-8 模式下的 8 位库，设置 0xA0 的位，除非代码是 EBCDIC。 */
          {
#ifndef EBCDIC
          SET_BIT(0xA0);
#else  /* Not EBCDIC */
          }
#endif  /* 8-bit support */
        break;

        case OP_ANYNL:
        case OP_VSPACE:
        SET_BIT(CHAR_LF);
        SET_BIT(CHAR_VT);
        SET_BIT(CHAR_FF);
        SET_BIT(CHAR_CR);

        /* 对于16位和32位库（永远不可能是EBCDIC），独立于UTF设置NEL和大于等于255的代码单元的位。 */
#if PCRE2_CODE_UNIT_WIDTH != 8
        SET_BIT(CHAR_NEL);
        SET_BIT(0xFF);
#else
        /* 对于UTF-8模式下的8位库，设置垂直空格字符的第一个代码单元的位。 */
#ifdef SUPPORT_UNICODE
        if (utf)
          {
          SET_BIT(0xC2);  /* 对于U+0085（NEL） */
          SET_BIT(0xE2);  /* 对于U+2028，U+2029 */
          }
        else
#endif
        /* 对于非UTF-8模式下的8位库，设置NEL的位。 */
          {
          SET_BIT(CHAR_NEL);
          }
#endif  /* 8-bit support */
        break;

        case OP_NOT_DIGIT:
        # 设置不是数字的位
        set_nottype_bits(re, cbit_digit, table_limit);
        break;

        case OP_DIGIT:
        # 设置数字的位
        set_type_bits(re, cbit_digit, table_limit);
        break;

        case OP_NOT_WHITESPACE:
        # 设置不是空白字符的位
        set_nottype_bits(re, cbit_space, table_limit);
        break;

        case OP_WHITESPACE:
        # 设置空白字符的位
        set_type_bits(re, cbit_space, table_limit);
        break;

        case OP_NOT_WORDCHAR:
        # 设置不是单词字符的位
        set_nottype_bits(re, cbit_word, table_limit);
        break;

        case OP_WORDCHAR:
        # 设置单词字符的位
        set_type_bits(re, cbit_word, table_limit);
        break;
        }

      tcode += 2;
      break;

      /* Extended class: if there are any property checks, or if this is a
      negative XCLASS without a map, give up. If there are no property checks,
      there must be wide characters on the XCLASS list, because otherwise an
      XCLASS would not have been created. This means that code points >= 255
      are potential starters. In the UTF-8 case we can scan them and set bits
      for the relevant leading bytes. */

#ifdef SUPPORT_WIDE_CHARS
      case OP_XCLASS:
      xclassflags = tcode[1 + LINK_SIZE];
      if ((xclassflags & XCL_HASPROP) != 0 ||
          (xclassflags & (XCL_MAP|XCL_NOT)) == XCL_NOT)
        return SSB_FAIL;

      /* We have a positive XCLASS or a negative one without a map. Set up the
      map pointer if there is one, and fall through. */

      classmap = ((xclassflags & XCL_MAP) == 0)? NULL :
        (uint8_t *)(tcode + 1 + LINK_SIZE + 1);

      /* In UTF-8 mode, scan the character list and set bits for leading bytes,
      then jump to handle the map. */
#if PCRE2_CODE_UNIT_WIDTH == 8
      // 如果 PCRE2_CODE_UNIT_WIDTH 等于 8
      if (utf && (xclassflags & XCL_NOT) == 0)
        {
        // 如果启用了 UTF 模式，并且 xclassflags 的 XCL_NOT 标志为 0
        PCRE2_UCHAR b, e;
        // 定义 PCRE2_UCHAR 类型的变量 b 和 e
        PCRE2_SPTR p = tcode + 1 + LINK_SIZE + 1 + ((classmap == NULL)? 0:32);
        // 定义 PCRE2_SPTR 类型的指针变量 p，并初始化为 tcode + 1 + LINK_SIZE + 1 + ((classmap == NULL)? 0:32)
        tcode += GET(tcode, 1);
        // tcode 增加 GET(tcode, 1) 的值

        for (;;) switch (*p++)
          {
          // 无限循环，根据 p 指向的值进行判断
          case XCL_SINGLE:
          // 如果 p 指向的值为 XCL_SINGLE
          b = *p++;
          // 将 p 指向的值赋给 b，然后 p 自增
          while ((*p & 0xc0) == 0x80) p++;
          // 当 p 指向的值的前两位为 10 时，p 自增
          re->start_bitmap[b/8] |= (1u << (b&7));
          // 将 re->start_bitmap[b/8] 的第 (b&7) 位置为 1
          break;

          case XCL_RANGE:
          // 如果 p 指向的值为 XCL_RANGE
          b = *p++;
          // 将 p 指向的值赋给 b，然后 p 自增
          while ((*p & 0xc0) == 0x80) p++;
          // 当 p 指向的值的前两位为 10 时，p 自增
          e = *p++;
          // 将 p 指向的值赋给 e，然后 p 自增
          while ((*p & 0xc0) == 0x80) p++;
          // 当 p 指向的值的前两位为 10 时，p 自增
          for (; b <= e; b++)
            // 循环，从 b 到 e
            re->start_bitmap[b/8] |= (1u << (b&7));
            // 将 re->start_bitmap[b/8] 的第 (b&7) 位置为 1
          break;

          case XCL_END:
          // 如果 p 指向的值为 XCL_END
          goto HANDLE_CLASSMAP;
          // 跳转到 HANDLE_CLASSMAP 标签处

          default:
          return SSB_UNKNOWN;   /* Internal error, should not occur */
          // 默认情况下返回 SSB_UNKNOWN，表示内部错误，不应该发生
          }
        }
#endif  /* SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8 */
#endif  /* SUPPORT_WIDE_CHARS */

      /* It seems that the fall through comment must be outside the #ifdef if
      it is to avoid the gcc compiler warning. */

      /* Fall through */

      /* Enter here for a negative non-XCLASS. In the 8-bit library, if we are
      in UTF mode, any byte with a value >= 0xc4 is a potentially valid starter
      because it starts a character with a value > 255. In 8-bit non-UTF mode,
      there is no difference between CLASS and NCLASS. In all other wide
      character modes, set the 0xFF bit to indicate code units >= 255. */

      case OP_NCLASS:
      // 当操作码为 OP_NCLASS 时执行以下操作
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
      // 如果定义了 SUPPORT_UNICODE 并且 PCRE2_CODE_UNIT_WIDTH 等于 8
      if (utf)
        {
        // 如果启用了 UTF 模式
        re->start_bitmap[24] |= 0xf0;            /* Bits for 0xc4 - 0xc8 */
        // 将 re->start_bitmap[24] 的值的高 4 位设置为 1，表示 0xc4 - 0xc8
        memset(re->start_bitmap+25, 0xff, 7);    /* Bits for 0xc9 - 0xff */
        // 将 re->start_bitmap+25 到 re->start_bitmap+31 的值都设置为 0xff，表示 0xc9 - 0xff
        }
#elif PCRE2_CODE_UNIT_WIDTH != 8
      SET_BIT(0xFF);                             /* For characters >= 255 */
      // 设置 0xFF 位，表示字符大于等于 255
#endif
      /* 如果前面是一个 XCLASS，那么直接跳到这里。如果是一个正常的非 XCLASS，那么设置 classmap 并且移动 tcode 指针。 */

      case OP_CLASS:
      if (*tcode == OP_XCLASS) tcode += GET(tcode, 1); else
        {
        classmap = (uint8_t *)(++tcode);
        tcode += 32 / sizeof(PCRE2_UCHAR);
        }

      /* 当支持宽字符时，classmap 可能为 NULL。在 UTF-8 模式下，类位图中的位对应于字符值，而不是字节值。然而，我们正在构造的位图是针对字节值的。因此，对于代码点大于 127 的字符，我们必须进行转换。实际上，范围为 128 - 255 的字符只有两个可能的起始字节。 */

#if defined SUPPORT_WIDE_CHARS && PCRE2_CODE_UNIT_WIDTH == 8
      HANDLE_CLASSMAP:
#endif
      if (classmap != NULL)
        {
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH == 8
        if (utf)
          {
          for (c = 0; c < 16; c++) re->start_bitmap[c] |= classmap[c];
          for (c = 128; c < 256; c++)
            {
            if ((classmap[c/8] & (1u << (c&7))) != 0)
              {
              int d = (c >> 6) | 0xc0;                 /* 为这个起始字节设置位 */
              re->start_bitmap[d/8] |= (1u << (d&7));  /* 然后跳到下一个相关的字符 */
              c = (c & 0xc0) + 0x40 - 1;
              }
            }
          }
        else
``` 
#endif
        /* 如果不是 UTF-8 模式，两个位图是兼容的。 */

          {
          for (c = 0; c < 32; c++) re->start_bitmap[c] |= classmap[c];
          }
        }

      /* 处理类之后的操作。对于最小重复次数为零的情况，继续处理；否则停止处理。 */

      switch (*tcode)
        {
        case OP_CRSTAR:
        case OP_CRMINSTAR:
        case OP_CRQUERY:
        case OP_CRMINQUERY:
        case OP_CRPOSSTAR:
        case OP_CRPOSQUERY:
        tcode++;
        break;

        case OP_CRRANGE:
        case OP_CRMINRANGE:
        case OP_CRPOSRANGE:
        if (GET2(tcode, 1) == 0) tcode += 1 + 2 * IMM2_SIZE;
          else try_next = FALSE;
        break;

        default:
        try_next = FALSE;
        break;
        }
      break; /* 类处理结束 */
      }      /* 操作码的 switch 结束 */
    }        /* try_next 循环结束 */

  code += GET(code, 1);   /* 跳转到下一个分支 */
  }
while (*code == OP_ALT);

return yield;
}



/*************************************************
*          Study a compiled expression           *
*************************************************/

/* 这个函数接收一个已编译的表达式，必须对其进行研究以产生可以加快匹配速度的信息。

参数：
  re       指向已编译的表达式

返回值：   通常为 0；非零值通常不应该发生
           1 在 set_start_bits 中出现未知操作码
           2 缺少捕获括号
           3 在 find_minlength 中出现未知操作码
*/

int
PRIV(study)(pcre2_real_code *re)
{
int count = 0;
PCRE2_UCHAR *code;
BOOL utf = (re->overall_options & PCRE2_UTF) != 0;
BOOL ucp = (re->overall_options & PCRE2_UCP) != 0;

/* 找到已编译代码的起始位置 */

code = (PCRE2_UCHAR *)((uint8_t *)re + sizeof(pcre2_real_code)) +
  re->name_entry_size * re->name_count;

/* 对于具有第一个代码单元的模式，或多行模式，utf 和 ucp 可能为真 */
# 检查正则表达式的标志，如果没有设置 PCRE2_FIRSTSET 和 PCRE2_STARTLINE 标志，则无需寻找起始码元列表
if ((re->flags & (PCRE2_FIRSTSET|PCRE2_STARTLINE)) == 0)
  {
  # 初始化深度为0
  int depth = 0;
  # 调用 set_start_bits 函数设置起始码元列表
  int rc = set_start_bits(re, code, utf, ucp, &depth);
  # 如果返回 SSB_UNKNOWN，则返回1
  if (rc == SSB_UNKNOWN) return 1;

  # 如果起始码元列表已设置，则扫描列表，看是否只有一个或两个码元
  if (rc == SSB_DONE)
    {
    int i;
    int a = -1;
    int b = -1;
    uint8_t *p = re->start_bitmap;
    uint32_t flags = PCRE2_FIRSTMAPSET;

    # 遍历起始码元列表
    for (i = 0; i < 256; p++, i += 8)
      {
      uint8_t x = *p;
      # 如果码元不为0
      if (x != 0)
        {
        int c;
        uint8_t y = x & (~x + 1);   # 获取最低位的1
        # 如果最低位的1不等于码元本身，则跳到DONE
        if (y != x) goto DONE;      # More than one bit set

        # 在16位和32位库中，0xff位表示"0xff和所有宽字符"，所以这里不能使用它
        # 如果码元为248且x为0x80，则跳到DONE
#if PCRE2_CODE_UNIT_WIDTH != 8
        if (i == 248 && x == 0x80) goto DONE;
#endif

        # 计算字符值
        c = i;
        switch (x)
          {
          case 1:   break;
          case 2:   c += 1; break;  case 4:  c += 2; break;
          case 8:   c += 3; break;  case 16: c += 4; break;
          case 32:  c += 5; break;  case 64: c += 6; break;
          case 128: c += 7; break;
          }

        # c 包含码元值，范围为0-255，在8位UTF模式下，只能使用小于128的值
#if PCRE2_CODE_UNIT_WIDTH == 8
        if (utf && c > 127) goto DONE;
#endif
        // 如果 a 小于 0，则将其赋值为 c，表示找到第一个字符
        if (a < 0) a = c;   /* First one found, save in a */
        // 如果 a 大于等于 0 且 b 小于 0，则表示找到第二个字符
        else if (b < 0)     /* Second one found */
          {
          // 从 re->tables + fcc_offset 中获取 c 对应的值，保存在 d 中
          int d = TABLE_GET((unsigned int)c, re->tables + fcc_offset, c);

#ifdef SUPPORT_UNICODE
          // 如果 utf 或 ucp 为真
          if (utf || ucp)
            {
            // 如果 UCD_CASESET(c) 不等于 0，则跳转到 DONE 标签，表示多个大小写集
            if (UCD_CASESET(c) != 0) goto DONE;     /* Multiple case set */
            // 如果 c 大于 127，则将 d 赋值为 UCD_OTHERCASE(c)
            if (c > 127) d = UCD_OTHERCASE(c);
            }
#endif  /* SUPPORT_UNICODE */

          // 如果 d 不等于 a，则跳转到 DONE 标签，表示不是 a 的另一个大小写形式
          if (d != a) goto DONE;   /* Not the other case of a */
          // 将 c 保存在 b 中，表示找到第二个字符
          b = c;                   /* Save second in b */
          }
        // 如果 a 和 b 都大于等于 0，则跳转到 DONE 标签，表示找到两个以上的字符
        else goto DONE;   /* More than two characters found */
        }
      }

    /* Replace the start code unit bits with a first code unit, but only if it
    is not the same as a required later code unit. This is because a search for
    a required code unit starts after an explicit first code unit, but at a
    code unit found from the bitmap. Patterns such as /a*a/ don't work
    if both the start unit and required unit are the same. */

    // 如果 a 大于等于 0 并且满足一定条件
    if (a >= 0 &&
        (
        (re->flags & PCRE2_LASTSET) == 0 ||
          (
          re->last_codeunit != (uint32_t)a &&
          (b < 0 || re->last_codeunit != (uint32_t)b)
          )
        ))
      {
      // 将 a 赋值给 re->first_codeunit
      re->first_codeunit = a;
      // 设置 flags 为 PCRE2_FIRSTSET
      flags = PCRE2_FIRSTSET;
      // 如果 b 大于等于 0，则设置 flags 为 PCRE2_FIRSTCASELESS
      if (b >= 0) flags |= PCRE2_FIRSTCASELESS;
      }

    // 跳转到 DONE 标签
    DONE:
    // 将 flags 添加到 re->flags 中
    re->flags |= flags;
    }
  }

/* Find the minimum length of subject string. If the pattern can match an empty
string, the minimum length is already known. If the pattern contains (*ACCEPT)
all bets are off, and we don't even try to find a minimum length. If there are
more back references than the size of the vector we are going to cache them in,
do nothing. A pattern that complicated will probably take a long time to
analyze and may in any case turn out to be too complicated. Note that back
reference minima are held as 16-bit numbers. */
# 如果正则表达式的标志不包含 PCRE2_MATCH_EMPTY 和 PCRE2_HASACCEPT，并且顶部反向引用小于等于最大缓存反向引用
if ((re->flags & (PCRE2_MATCH_EMPTY|PCRE2_HASACCEPT)) == 0 &&
     re->top_backref <= MAX_CACHE_BACKREF)
  {
  # 定义最小值和反向引用缓存数组
  int min;
  int backref_cache[MAX_CACHE_BACKREF+1];
  # 设置反向引用缓存数组的第一个元素为0
  backref_cache[0] = 0;    /* Highest one that is set */
  # 调用函数计算最小长度
  min = find_minlength(re, code, code, utf, NULL, &count, backref_cache);
  # 根据最小长度的不同情况进行处理
  switch(min)
    {
    case -1:  /* \C in UTF mode or over-complex regex */
    break;    /* Leave minlength unchanged (will be zero) */

    case -2:
    return 2; /* missing capturing bracket */

    case -3:
    return 3; /* unrecognized opcode */

    default:
    # 如果最小长度大于 UINT16_MAX，则将最小长度设置为 UINT16_MAX，否则设置为 min
    re->minlength = (min > UINT16_MAX)? UINT16_MAX : min;
    break;
    }
  }
# 返回0
return 0;
}

/* End of pcre2_study.c */
```