# `nmap\libpcre\src\pcre2_auto_possess.c`

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
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
-----------------------------------------------------------------------------
*/

/* This module contains functions that scan a compiled pattern and change
repeats into possessive repeats where possible. */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/*************************************************
*        Tables for auto-possessification        *
*************************************************/

/* This table is used to check whether auto-possessification is possible
between adjacent character-type opcodes. The left-hand (repeated) opcode is
used to select the row, and the right-hand opcode is use to select the column.
A value of 1 means that auto-possessification is OK. For example, the second
value in the first row means that \D+\d can be turned into \D++\d.

The Unicode property types (\P and \p) have to be present to fill out the table
because of what their opcode values are, but the table values should always be
zero because property types are handled separately in the code. The last four
columns apply to items that cannot be repeated, so there is no need to have
rows for them. Note that OP_DIGIT etc. are generated only when PCRE_UCP is
*not* set. When it is set, \d etc. are converted into OP_(NOT_)PROP codes. */

#define APTROWS (LAST_AUTOTAB_LEFT_OP - FIRST_AUTOTAB_OP + 1)
#define APTCOLS (LAST_AUTOTAB_RIGHT_OP - FIRST_AUTOTAB_OP + 1)

static const uint8_t autoposstab[APTROWS][APTCOLS] = {
# 此表用于检查相邻的 Unicode 属性操作码（OP_PROP 和 OP_NOTPROP）之间是否可能进行自动占有化。
# 左侧（重复）操作码用于选择行，右侧操作码用于选择列。
# 表中的值表示是否可能进行自动占有化，1 为可能，0 为不可能。
# opcode 用于选择列。其取值如下：

# 0   始终返回 FALSE（永不自动占有）
# 1   字符组是不同的（如果两者都是 OP_PROP，则占有）
# 2   检查相同组中的字符类别（通用或特定）
# 3   如果两个操作码不相同，则为 TRUE（PROP vs NOTPROP）

# 4   检查左侧通用类别 vs 右侧特定类别
# 5   检查右侧通用类别 vs 左侧特定类别

# 6   左侧字母数字 vs 右侧通用类别
# 7   左侧空格 vs 右侧通用类别
# 8   左侧单词 vs 右侧通用类别

# 9   右侧字母数字 vs 左侧通用类别
# 10  右侧空格 vs 左侧通用类别
# 11  右侧单词 vs 左侧通用类别

# 12  左侧字母数字 vs 右侧特定类别
# 13  左侧空格 vs 右侧特定类别
# 14  左侧单词 vs 右侧特定类别

# 15  右侧字母数字 vs 左侧特定类别
# 16  右侧空格 vs 左侧特定类别
# 17  右侧单词 vs 左侧特定类别
*/

# 定义一个二维数组，用于存储 propposstab 的值
static const uint8_t propposstab[PT_TABSIZE][PT_TABSIZE] = {
/* 定义一个二维数组，用于检查在指定一般类别和指定特定类别时，相邻的 Unicode 属性操作码（OP_PROP 和 OP_NOTPROP）之间是否可以进行自动占有。行由一般类别选择，列由特定类别选择。如果特定类别不属于一般类别，则值为1。 */
static const uint8_t catposstab[7][30] = {
/* Cc Cf Cn Co Cs Ll Lm Lo Lt Lu Mc Me Mn Nd Nl No Pc Pd Pe Pf Pi Po Ps Sc Sk Sm So Zl Zp Zs */
  { 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 },  /* C */
  { 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 },  /* L */
  { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 },  /* M */
  { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1,
/*
  检查字符属性，确定是否可以自动转换为 possessive
  参数：
    c            字符
    ptype        属性类型
    pdata        类型的数据
    negated      如果是否定属性（\P 或 \p{^}）则为 TRUE
  返回值：       如果可以自动转换为 possessive 则为 TRUE
*/

static BOOL
check_char_prop(uint32_t c, unsigned int ptype, unsigned int pdata,
  BOOL negated)
{
BOOL ok;
const uint32_t *p;
const ucd_record *prop = GET_UCD(c);

switch(ptype)
  {
  case PT_LAMP:
  return (prop->chartype == ucp_Lu ||
          prop->chartype == ucp_Ll ||
          prop->chartype == ucp_Lt) == negated;

  case PT_GC:
  return (pdata == PRIV(ucp_gentype)[prop->chartype]) == negated;

  case PT_PC:
  return (pdata == prop->chartype) == negated;

  case PT_SC:
  return (pdata == prop->script) == negated;

  case PT_SCX:
  ok = (pdata == prop->script
        || MAPBIT(PRIV(ucd_script_sets) + UCD_SCRIPTX_PROP(prop), pdata) != 0);
  return ok == negated;

  /* 这些是特殊情况 */

  case PT_ALNUM:
  return (PRIV(ucp_gentype)[prop->chartype] == ucp_L ||
          PRIV(ucp_gentype)[prop->chartype] == ucp_N) == negated;

  /* Perl 空格用于排除 VT，但从 Perl 5.18 开始，它被包括在内，这意味着 Perl 空格和 POSIX 空格现在是相同的。PCRE 在 8.34 版本中进行了更改。 */

  case PT_SPACE:    /* Perl 空格 */
  case PT_PXSPACE:  /* POSIX 空格 */
  switch(c)
    {
    HSPACE_CASES:
    VSPACE_CASES:
    return negated;

    default:
    return (PRIV(ucp_gentype)[prop->chartype] == ucp_Z) == negated;
    }
  break;  /* 控制永远不会到达这里 */

  case PT_WORD:
  return (PRIV(ucp_gentype)[prop->chartype] == ucp_L ||
          PRIV(ucp_gentype)[prop->chartype] == ucp_N ||
          c == CHAR_UNDERSCORE) == negated;

  case PT_CLIST:
  p = PRIV(ucd_caseless_sets) + prop->caseset;
  for (;;)
    {
    if (c < *p) return !negated;
    if (c == *p++) return negated;
    }
  }
}
    }
  break;  /* Control never reaches here */  # 当前情况下不会执行到这里

  /* Haven't yet thought these through. */  # 这些情况还没有想清楚

  case PT_BIDICL:  # 当前情况为双向字典
  return FALSE;  # 返回假

  case PT_BOOL:  # 当前情况为布尔类型
  return FALSE;  # 返回假
  }
# 返回 FALSE
return FALSE;
}
#endif  /* SUPPORT_UNICODE */



/*************************************************
*        Base opcode of repeated opcodes         *
*************************************************/

/* 返回重复操作码的基本操作码。如果操作码不是重复字符类型，则返回原始值。

参数：c 操作码
返回：该类型的基本操作码
*/

static PCRE2_UCHAR
get_repeat_base(PCRE2_UCHAR c)
{
# 如果操作码不是重复字符类型，则返回原始值
return (c > OP_TYPEPOSUPTO)? c :
       (c >= OP_TYPESTAR)?   OP_TYPESTAR :
       (c >= OP_NOTSTARI)?   OP_NOTSTARI :
       (c >= OP_NOTSTAR)?    OP_NOTSTAR :
       (c >= OP_STARI)?      OP_STARI :
                             OP_STAR;
}


/*************************************************
*        Fill the character property list        *
*************************************************/

/* 检查代码点是否指向可以参与自动占有化的操作码，如果是，则填充列表中的属性。

参数：
  code        指向表达式开头
  utf         如果处于 UTF 模式，则为 TRUE
  ucp         如果处于 UCP 模式，则为 TRUE
  fcc         指向大小写翻转表
  list        指向输出列表
              list[0] 将填充操作码
              list[1] 如果此操作码可以匹配空字符，则为非零值
              list[2..7] 取决于操作码

返回：如果 *code 被接受，则指向下一个操作码的开始
      如果 *code 不被接受，则返回 NULL
*/

static PCRE2_SPTR
get_chr_property_list(PCRE2_SPTR code, BOOL utf, BOOL ucp, const uint8_t *fcc,
  uint32_t *list)
{
PCRE2_UCHAR c = *code;
PCRE2_UCHAR base;
PCRE2_SPTR end;
uint32_t chr;

#ifdef SUPPORT_UNICODE
uint32_t *clist_dest;
const uint32_t *clist_src;
#else
(void)utf    # 抑制“未使用参数”编译器警告
(void)ucp;
#endif

list[0] = c;
list[1] = FALSE;
code++;
# 如果字符 c 在 OP_STAR 和 OP_TYPEPOSUPTO 之间
if (c >= OP_STAR && c <= OP_TYPEPOSUPTO)
  {
  # 获取重复基数
  base = get_repeat_base(c);
  # 重新计算 c 的值
  c -= (base - OP_STAR);

  # 如果 c 是 OP_UPTO、OP_MINUPTO、OP_EXACT 或 OP_POSUPTO，则增加 IMM2_SIZE
  if (c == OP_UPTO || c == OP_MINUPTO || c == OP_EXACT || c == OP_POSUPTO)
    code += IMM2_SIZE;

  # 设置 list[1] 的值
  list[1] = (c != OP_PLUS && c != OP_MINPLUS && c != OP_EXACT &&
             c != OP_POSPLUS);

  # 根据 base 的值进行不同的操作
  switch(base)
    {
    case OP_STAR:
    list[0] = OP_CHAR;
    break;

    case OP_STARI:
    list[0] = OP_CHARI;
    break;

    case OP_NOTSTAR:
    list[0] = OP_NOT;
    break;

    case OP_NOTSTARI:
    list[0] = OP_NOTI;
    break;

    case OP_TYPESTAR:
    list[0] = *code;
    code++;
    break;
    }
  # 更新 c 的值
  c = list[0];
  }

# 根据 c 的值进行不同的操作
switch(c)
  {
  # 如果 c 是以下情况之一，则直接返回 code
  case OP_NOT_DIGIT:
  case OP_DIGIT:
  case OP_NOT_WHITESPACE:
  case OP_WHITESPACE:
  case OP_NOT_WORDCHAR:
  case OP_WORDCHAR:
  case OP_ANY:
  case OP_ALLANY:
  case OP_ANYNL:
  case OP_NOT_HSPACE:
  case OP_HSPACE:
  case OP_NOT_VSPACE:
  case OP_VSPACE:
  case OP_EXTUNI:
  case OP_EODN:
  case OP_EOD:
  case OP_DOLL:
  case OP_DOLLM:
  return code;

  # 如果 c 是 OP_CHAR 或 OP_NOT，则获取下一个字符并返回相应的值
  case OP_CHAR:
  case OP_NOT:
  GETCHARINCTEST(chr, code);
  list[2] = chr;
  list[3] = NOTACHAR;
  return code;

  # 如果 c 是 OP_CHARI 或 OP_NOTI，则根据条件设置 list 的值，并返回相应的值
  case OP_CHARI:
  case OP_NOTI:
  list[0] = (c == OP_CHARI) ? OP_CHAR : OP_NOT;
  GETCHARINCTEST(chr, code);
  list[2] = chr;

  # 根据条件设置 list[3] 的值
#ifdef SUPPORT_UNICODE
  if (chr < 128 || (chr < 256 && !utf && !ucp))
    list[3] = fcc[chr];
  else
    list[3] = UCD_OTHERCASE(chr);
#elif defined SUPPORT_WIDE_CHARS
  list[3] = (chr < 256) ? fcc[chr] : chr;
#else
  list[3] = fcc[chr];
#endif

  # 如果 chr 和 list[3] 相等，则设置 list[3] 为 NOTACHAR，否则设置 list[4] 为 NOTACHAR
  if (chr == list[3])
    list[3] = NOTACHAR;
  else
    list[4] = NOTACHAR;
  return code;

  # 如果 c 是 OP_PROP 或 OP_NOTPROP，则根据条件设置 list 的值，并返回相应的值
#ifdef SUPPORT_UNICODE
  case OP_PROP:
  case OP_NOTPROP:
  if (code[0] != PT_CLIST)
    {
    list[2] = code[0];
    list[3] = code[1];
    return code + 2;
    }

  /* 只有当我们有足够的空间时才进行转换。 */

  clist_src = PRIV(ucd_caseless_sets) + code[1];  // 设置源字符列表为ucd_caseless_sets中的偏移量加上code的第二个元素
  clist_dest = list + 2;  // 设置目标字符列表为list的第三个元素
  code += 2;  // code指针向后移动两个位置

  do {
     if (clist_dest >= list + 8)  // 如果目标字符列表超过了8个元素
       {
       /* 如果没有足够的空间，就提前返回。这不应该发生，因为现在所有的字符列表都比5个字符短。 */
       list[2] = code[0];  // 将code的第一个元素赋值给list的第三个元素
       list[3] = code[1];  // 将code的第二个元素赋值给list的第四个元素
       return code;  // 返回code指针的值
       }
     *clist_dest++ = *clist_src;  // 将源字符列表的值复制到目标字符列表，并将指针向后移动
     }
  while(*clist_src++ != NOTACHAR);  // 当源字符列表的值不是NOTACHAR时继续循环

  /* 所有字符都已经存储。终止的NOTACHAR从clist本身复制。 */

  list[0] = (c == OP_PROP) ? OP_CHAR : OP_NOT;  // 如果c等于OP_PROP，则将list的第一个元素设置为OP_CHAR，否则设置为OP_NOT
  return code;  // 返回code指针的值
#endif

这是一个条件编译指令，用于结束一个条件编译块。


  case OP_NCLASS:
  case OP_CLASS:
#ifdef SUPPORT_WIDE_CHARS
  case OP_XCLASS:
  if (c == OP_XCLASS)
    end = code + GET(code, 0) - 1;
  else
#endif
    end = code + 32 / sizeof(PCRE2_UCHAR);

根据不同的操作码，设置变量end的值。


  switch(*end)
    {
    case OP_CRSTAR:
    case OP_CRMINSTAR:
    case OP_CRQUERY:
    case OP_CRMINQUERY:
    case OP_CRPOSSTAR:
    case OP_CRPOSQUERY:
    list[1] = TRUE;
    end++;
    break;

    case OP_CRPLUS:
    case OP_CRMINPLUS:
    case OP_CRPOSPLUS:
    end++;
    break;

    case OP_CRRANGE:
    case OP_CRMINRANGE:
    case OP_CRPOSRANGE:
    list[1] = (GET2(end, 1) == 0);
    end += 1 + 2 * IMM2_SIZE;
    break;
    }

根据end指向的操作码，进行不同的操作。


  list[2] = (uint32_t)(end - code);
  return end;
  }

设置list[2]的值，并返回end的值。


return NULL;    /* Opcode not accepted */

如果没有匹配的操作码，则返回NULL。


/*************************************************
*    Scan further character sets for match       *
*************************************************/

这是一个注释，说明下面的代码是用于扫描更多的字符集以进行匹配。


/* Checks whether the base and the current opcode have a common character, in
which case the base cannot be possessified.

Arguments:
  code        points to the byte code
  utf         TRUE in UTF mode
  ucp         TRUE in UCP mode
  cb          compile data block
  base_list   the data list of the base opcode
  base_end    the end of the base opcode
  rec_limit   points to recursion depth counter

Returns:      TRUE if the auto-possessification is possible
*/

这是一个函数的注释，说明了函数的作用、参数和返回值。


static BOOL
compare_opcodes(PCRE2_SPTR code, BOOL utf, BOOL ucp, const compile_block *cb,
  const uint32_t *base_list, PCRE2_SPTR base_end, int *rec_limit)
{

定义了一个名为compare_opcodes的静态函数，接受多个参数。


PCRE2_UCHAR c;
uint32_t list[8];
const uint32_t *chr_ptr;
const uint32_t *ochr_ptr;
const uint32_t *list_ptr;
PCRE2_SPTR next_code;
#ifdef SUPPORT_WIDE_CHARS
PCRE2_SPTR xclass_flags;
#endif
const uint8_t *class_bitset;
const uint8_t *set1, *set2, *set_end;
uint32_t chr;
BOOL accepted, invert_bits;
BOOL entered_a_group = FALSE;

定义了一系列变量。


if (--(*rec_limit) <= 0) return FALSE;  /* Recursion has gone too deep */

如果递归深度超过限制，则返回FALSE。

以上是对给定代码的每个语句添加注释后的结果。
/* Note: the base_list[1] contains whether the current opcode has a greedy
(represented by a non-zero value) quantifier. This is a different from
other character type lists, which store here that the character iterator
matches to an empty string (also represented by a non-zero value). */

for(;;)
  {
  /* All operations move the code pointer forward.
  Therefore infinite recursions are not possible. */

  c = *code;

  /* Skip over callouts */

  if (c == OP_CALLOUT)
    {
    code += PRIV(OP_lengths)[c];
    continue;
    }

  if (c == OP_CALLOUT_STR)
    {
    code += GET(code, 1 + 2*LINK_SIZE);
    continue;
    }

  /* At the end of a branch, skip to the end of the group. */

  if (c == OP_ALT)
    {
    do code += GET(code, 1); while (*code == OP_ALT);
    c = *code;
    }

  /* Inspect the next opcode. */

  switch(c)
    {
    /* We can always possessify a greedy iterator at the end of the pattern,
    which is reached after skipping over the final OP_KET. A non-greedy
    iterator must never be possessified. */

    case OP_END:
    return base_list[1] != 0;

    /* When an iterator is at the end of certain kinds of group we can inspect
    what follows the group by skipping over the closing ket. Note that this
    does not apply to OP_KETRMAX or OP_KETRMIN because what follows any given
    iteration is variable (could be another iteration or could be the next
    item). As these two opcodes are not listed in the next switch, they will
    end up as the next code to inspect, and return FALSE by virtue of being
    unsupported. */

    case OP_KET:
    case OP_KETRPOS:
    /* The non-greedy case cannot be converted to a possessive form. */

    if (base_list[1] == 0) return FALSE;

    /* If the bracket is capturing it might be referenced by an OP_RECURSE
    so its last iterator can never be possessified if the pattern contains
    recursions. (This could be improved by keeping a list of group numbers that
    are called by recursion.) */
    # 根据给定的操作码执行不同的操作
    switch(*(code - GET(code, 1)))
      {
      # 如果操作码是 OP_CBRA、OP_SCBRA、OP_CBRAPOS、OP_SCBRAPOS 中的一个，且已经有递归，则返回 FALSE
      case OP_CBRA:
      case OP_SCBRA:
      case OP_CBRAPOS:
      case OP_SCBRAPOS:
      if (cb->had_recurse) return FALSE;
      break;

      # 如果操作码是 OP_SCRIPT_RUN，且 base_list 的第一个元素不是 OP_CHAR 或 OP_CHARI，则返回 FALSE
      # 否则继续执行
      case OP_SCRIPT_RUN:
      if (base_list[0] != OP_CHAR && base_list[0] != OP_CHARI)
        return FALSE;
      break;

      # 如果操作码是 OP_ASSERT、OP_ASSERT_NOT、OP_ASSERTBACK、OP_ASSERTBACK_NOT、OP_ONCE，则根据 entered_a_group 的值返回结果
      case OP_ASSERT:
      case OP_ASSERT_NOT:
      case OP_ASSERTBACK:
      case OP_ASSERTBACK_NOT:
      case OP_ONCE:
      return !entered_a_group;

      # 如果操作码是 OP_ASSERT_NA 或 OP_ASSERTBACK_NA，则返回 FALSE
      case OP_ASSERT_NA:
      case OP_ASSERTBACK_NA:
      return FALSE;
      }

    # 跳过括号并检查接下来的内容
    code += PRIV(OP_lengths)[c];
    continue;

    # 处理下一个项目是一个组的情况
    case OP_ONCE:
    case OP_BRA:
    case OP_CBRA:
    next_code = code + GET(code, 1);
    code += PRIV(OP_lengths)[c];

    # 检查每个分支，除了最后一个分支，都需要递归一层
    while (*next_code == OP_ALT)
      {
      if (!compare_opcodes(code, utf, ucp, cb, base_list, base_end, rec_limit))
        return FALSE;
      code = next_code + 1 + LINK_SIZE;
      next_code += GET(next_code, 1);
      }

    entered_a_group = TRUE;
    continue;

    case OP_BRAZERO:
    case OP_BRAMINZERO:

    next_code = code + 1;
    if (*next_code != OP_BRA && *next_code != OP_CBRA &&
        *next_code != OP_ONCE) return FALSE;

    do next_code += GET(next_code, 1); while (*next_code == OP_ALT);
    /* The bracket content will be checked by the OP_BRA/OP_CBRA case above. */
    // 括号内容将由上面的 OP_BRA/OP_CBRA 情况进行检查

    next_code += 1 + LINK_SIZE;
    // 增加下一个代码的偏移量，包括 1 和 LINK_SIZE 的大小
    if (!compare_opcodes(next_code, utf, ucp, cb, base_list, base_end,
         rec_limit))
      return FALSE;
    // 如果下一个代码与给定条件不匹配，则返回 FALSE

    code += PRIV(OP_lengths)[c];
    // 将代码指针移动到下一个操作码的位置
    continue;
    // 继续执行下一次循环

    /* The next opcode does not need special handling; fall through and use it
    to see if the base can be possessified. */
    // 下一个操作码不需要特殊处理；继续执行并使用它来判断基础是否可以被 possessified

    default:
    break;
    // 默认情况下，跳出 switch 语句
    }

  /* We now have the next appropriate opcode to compare with the base. Check
  for a supported opcode, and load its properties. */
  // 现在我们有了下一个适当的操作码来与基础进行比较。检查支持的操作码，并加载其属性。

  code = get_chr_property_list(code, utf, ucp, cb->fcc, list);
  // 获取字符属性列表的代码，并更新代码指针
  if (code == NULL) return FALSE;    /* Unsupported */
  // 如果代码为空，则返回 FALSE（不支持）

  /* If either opcode is a small character list, set pointers for comparing
  characters from that list with another list, or with a property. */
  // 如果任一操作码是小字符列表，则设置指针，用于将该列表中的字符与另一个列表或属性进行比较。

  if (base_list[0] == OP_CHAR)
    {
    chr_ptr = base_list + 2;
    list_ptr = list;
    }
  else if (list[0] == OP_CHAR)
    {
    chr_ptr = list + 2;
    list_ptr = base_list;
    }

  /* Character bitsets can also be compared to certain opcodes. */
  // 字符位集也可以与某些操作码进行比较。

  else if (base_list[0] == OP_CLASS || list[0] == OP_CLASS
#if PCRE2_CODE_UNIT_WIDTH == 8
      /* 如果 PCRE2_CODE_UNIT_WIDTH 等于 8，则在 8 位、非 UTF 模式下，OP_CLASS 和 OP_NCLASS 是相同的。 */
      || (!utf && (base_list[0] == OP_NCLASS || list[0] == OP_NCLASS))
#endif
      )
    {
#if PCRE2_CODE_UNIT_WIDTH == 8
    if (base_list[0] == OP_CLASS || (!utf && base_list[0] == OP_NCLASS))
#else
    if (base_list[0] == OP_CLASS)
#endif
      {
      set1 = (uint8_t *)(base_end - base_list[2]);
      list_ptr = list;
      }
    else
      {
      set1 = (uint8_t *)(code - list[2]);
      list_ptr = base_list;
      }

    invert_bits = FALSE;
    switch(list_ptr[0])
      {
      case OP_CLASS:
      case OP_NCLASS:
      set2 = (uint8_t *)
        ((list_ptr == list ? code : base_end) - list_ptr[2]);
      break;

#ifdef SUPPORT_WIDE_CHARS
      case OP_XCLASS:
      xclass_flags = (list_ptr == list ? code : base_end) - list_ptr[2] + LINK_SIZE;
      if ((*xclass_flags & XCL_HASPROP) != 0) return FALSE;
      if ((*xclass_flags & XCL_MAP) == 0)
        {
        /* 对于字符 < 256，没有设置任何位。 */
        if (list[1] == 0) return (*xclass_flags & XCL_NOT) == 0;
        /* 可能是一个空的重复。 */
        continue;
        }
      set2 = (uint8_t *)(xclass_flags + 1);
      break;
#endif

      case OP_NOT_DIGIT:
      invert_bits = TRUE;
      /* 继续执行下面的代码 */
      case OP_DIGIT:
      set2 = (uint8_t *)(cb->cbits + cbit_digit);
      break;

      case OP_NOT_WHITESPACE:
      invert_bits = TRUE;
      /* 继续执行下面的代码 */
      case OP_WHITESPACE:
      set2 = (uint8_t *)(cb->cbits + cbit_space);
      break;

      case OP_NOT_WORDCHAR:
      invert_bits = TRUE;
      /* 继续执行下面的代码 */
      case OP_WORDCHAR:
      set2 = (uint8_t *)(cb->cbits + cbit_word);
      break;

      default:
      return FALSE;
      }

    /* 因为位集是不对齐的字节，所以我们需要在这里执行字节比较。 */

    set_end = set1 + 32;
    # 如果需要反转比特
    if (invert_bits)
      {
      # 循环遍历两个集合，如果存在任何一个比特不匹配，则返回 FALSE
      do
        {
        if ((*set1++ & ~(*set2++)) != 0) return FALSE;
        }
      while (set1 < set_end);
      }
    # 如果不需要反转比特
    else
      {
      # 循环遍历两个集合，如果存在任何一个比特匹配，则返回 FALSE
      do
        {
        if ((*set1++ & *set2++) != 0) return FALSE;
        }
      while (set1 < set_end);
      }

    # 如果列表中第二个元素为 0，则返回 TRUE
    if (list[1] == 0) return TRUE;
    # 可能是一个空的重复
    /* Might be an empty repeat. */
    continue;
    }

  # 一些属性组合也是可以接受的。Unicode 属性操作码会被特殊处理；其余的可以用查找表处理。
  else
    {
    # 定义两个操作数
    uint32_t leftop, rightop;

    # 将基本列表的第一个元素赋值给左操作数
    leftop = base_list[0];
    # 将列表的第一个元素赋值给右操作数
    rightop = list[0];
#ifdef SUPPORT_UNICODE
    // 如果支持 Unicode，则始终设置为 FALSE
    accepted = FALSE; /* Always set in non-unicode case. */
    else
#endif  /* SUPPORT_UNICODE */

    // 检查是否接受当前操作符
    accepted = leftop >= FIRST_AUTOTAB_OP && leftop <= LAST_AUTOTAB_LEFT_OP &&
           rightop >= FIRST_AUTOTAB_OP && rightop <= LAST_AUTOTAB_RIGHT_OP &&
           autoposstab[leftop - FIRST_AUTOTAB_OP][rightop - FIRST_AUTOTAB_OP];

    // 如果不接受，则返回 FALSE
    if (!accepted) return FALSE;

    // 如果列表中的第一个元素为 0，则返回 TRUE
    if (list[1] == 0) return TRUE;
    /* 可能是一个空的重复。继续循环 */
    continue;
    }

  /* 控制只有在其中一个项目是小字符列表时才会到达这里。所有字符都与另一侧进行检查。 */

  do
    {
    chr = *chr_ptr;

#ifndef EBCDIC
        // 如果不是 EBCDIC 编码，则返回 FALSE
        case 0x2028:
        case 0x2029:
#endif  /* Not EBCDIC */
        return FALSE;
        }
      break;

      // 如果是 OP_EOD，则可以在 \z 之前始终进行占有
      case OP_EOD:    /* Can always possessify before \z */
      break;

#ifdef SUPPORT_UNICODE
      // 如果是 OP_PROP 或 OP_NOTPROP，则检查字符属性是否匹配
      case OP_PROP:
      case OP_NOTPROP:
      if (!check_char_prop(chr, list_ptr[2], list_ptr[3],
            list_ptr[0] == OP_NOTPROP))
        return FALSE;
      break;
#endif

      // 如果是 OP_NCLASS，则检查字符是否大于 255，如果是则返回 FALSE
      case OP_NCLASS:
      if (chr > 255) return FALSE;
      /* 继续执行 */

      // 如果是 OP_CLASS，则检查字符是否大于 255
      case OP_CLASS:
      if (chr > 255) break;
      class_bitset = (uint8_t *)
        ((list_ptr == list ? code : base_end) - list_ptr[2]);
      // 检查字符是否在位集中，如果是则返回 FALSE
      if ((class_bitset[chr >> 3] & (1u << (chr & 7))) != 0) return FALSE;
      break;

#ifdef SUPPORT_WIDE_CHARS
      // 如果是 OP_XCLASS，则调用 PRIV(xclass) 函数进行检查
      case OP_XCLASS:
      if (PRIV(xclass)(chr, (list_ptr == list ? code : base_end) -
          list_ptr[2] + LINK_SIZE, utf)) return FALSE;
      break;
#endif

      // 默认情况下返回 FALSE
      default:
      return FALSE;
      }

    // 指针指向下一个字符
    chr_ptr++;
    }
  while(*chr_ptr != NOTACHAR);

  // 这个操作码至少必须匹配一个字符
  if (list[1] == 0) return TRUE;
  }

/* 控制永远不会到达这里。这里曾经有一个失败安全的返回 FALSE；但是一些编译器会抱怨有不可达的语句。 */
}
/* 
   扫描编译后的正则表达式，为自动占有替换单个字符迭代为适当的占有替代品。
   这个函数修改了编译后的操作码！遇到不存在的操作码可能表明 PCRE2 中存在 bug，但也可能是由于使用 PCRE2_NO_UTF_CHECK 编译了错误的 UTF 字符串导致的。
   rec_limit 用于捕捉过于复杂或大型的模式。在这些情况下，检查会停止，剩余的模式将保持未被占有化。

   参数：
   code        指向字节码起始位置
   cb          编译数据块

   返回值：      
   0 表示成功
   -1 表示遇到不存在的操作码
*/

int
PRIV(auto_possessify)(PCRE2_UCHAR *code, const compile_block *cb)
{
    PCRE2_UCHAR c;
    PCRE2_SPTR end;
    PCRE2_UCHAR *repeat_opcode;
    uint32_t list[8];
    int rec_limit = 1000;  /* 原先是 10,000，但是 clang+ASAN 使用了大量的堆栈。*/
    BOOL utf = (cb->external_options & PCRE2_UTF) != 0;
    BOOL ucp = (cb->external_options & PCRE2_UCP) != 0;

    for (;;)
    {
        c = *code;

        if (c >= OP_TABLE_LENGTH) return -1;   /* 出现问题 */

        if (c >= OP_STAR && c <= OP_TYPEPOSUPTO)
        {
            c -= get_repeat_base(c) - OP_STAR;
            end = (c <= OP_MINUPTO) ?
                get_chr_property_list(code, utf, ucp, cb->fcc, list) : NULL;
            list[1] = c == OP_STAR || c == OP_PLUS || c == OP_QUERY || c == OP_UPTO;
    # 如果结束符不为空，并且比较操作码返回真值，执行以下操作
    if (end != NULL && compare_opcodes(end, utf, ucp, cb, list, end, &rec_limit))
      {
      # 根据操作码进行不同的处理
      switch(c)
        {
        # 如果操作码为 OP_STAR，执行以下操作
        case OP_STAR:
        *code += OP_POSSTAR - OP_STAR;
        break;

        # 如果操作码为 OP_MINSTAR，执行以下操作
        case OP_MINSTAR:
        *code += OP_POSSTAR - OP_MINSTAR;
        break;

        # 如果操作码为 OP_PLUS，执行以下操作
        case OP_PLUS:
        *code += OP_POSPLUS - OP_PLUS;
        break;

        # 如果操作码为 OP_MINPLUS，执行以下操作
        case OP_MINPLUS:
        *code += OP_POSPLUS - OP_MINPLUS;
        break;

        # 如果操作码为 OP_QUERY，执行以下操作
        case OP_QUERY:
        *code += OP_POSQUERY - OP_QUERY;
        break;

        # 如果操作码为 OP_MINQUERY，执行以下操作
        case OP_MINQUERY:
        *code += OP_POSQUERY - OP_MINQUERY;
        break;

        # 如果操作码为 OP_UPTO，执行以下操作
        case OP_UPTO:
        *code += OP_POSUPTO - OP_UPTO;
        break;

        # 如果操作码为 OP_MINUPTO，执行以下操作
        case OP_MINUPTO:
        *code += OP_POSUPTO - OP_MINUPTO;
        break;
        }
      }
    # 将操作码赋值给变量 c
    c = *code;
    }
  # 如果操作码为 OP_CLASS 或者 OP_NCLASS 或者 OP_XCLASS，执行以下操作
  else if (c == OP_CLASS || c == OP_NCLASS || c == OP_XCLASS)
    {
#ifdef SUPPORT_WIDE_CHARS
    # 如果支持宽字符，则执行以下代码
    if (c == OP_XCLASS)
      # 如果当前字符为 OP_XCLASS，则将 repeat_opcode 设置为 code + GET(code, 1)
      repeat_opcode = code + GET(code, 1);
    else
#endif
      # 如果不支持宽字符，则将 repeat_opcode 设置为 code + 1 + (32 / sizeof(PCRE2_UCHAR))
      repeat_opcode = code + 1 + (32 / sizeof(PCRE2_UCHAR));

    # 将 c 设置为 repeat_opcode 指向的值
    c = *repeat_opcode;
    # 如果 c 大于等于 OP_CRSTAR 并且小于等于 OP_CRMINRANGE
    if (c >= OP_CRSTAR && c <= OP_CRMINRANGE)
      {
      /* The return from get_chr_property_list() will never be NULL when
      *code (aka c) is one of the three class opcodes. However, gcc with
      -fanalyzer notes that a NULL return is possible, and grumbles. Hence we
      put in a check. */
      # 从 get_chr_property_list() 返回的值在 code（也就是 c）是三个类操作码之一时永远不会为 NULL。然而，带有 -fanalyzer 的 gcc 注意到可能会返回 NULL，并且会抱怨。因此我们进行了检查。

      # 将 end 设置为 get_chr_property_list() 的返回值
      end = get_chr_property_list(code, utf, ucp, cb->fcc, list);
      # 将 list[1] 设置为 (c & 1) == 0
      list[1] = (c & 1) == 0;

      # 如果 end 不为 NULL 并且 compare_opcodes() 返回真
      if (end != NULL &&
          compare_opcodes(end, utf, ucp, cb, list, end, &rec_limit))
        {
        # 根据不同的 c 值进行不同的操作
        switch (c)
          {
          case OP_CRSTAR:
          case OP_CRMINSTAR:
          *repeat_opcode = OP_CRPOSSTAR;
          break;

          case OP_CRPLUS:
          case OP_CRMINPLUS:
          *repeat_opcode = OP_CRPOSPLUS;
          break;

          case OP_CRQUERY:
          case OP_CRMINQUERY:
          *repeat_opcode = OP_CRPOSQUERY;
          break;

          case OP_CRRANGE:
          case OP_CRMINRANGE:
          *repeat_opcode = OP_CRPOSRANGE;
          break;
          }
        }
      }
    # 将 c 设置为 *code
    c = *code;
    }

  # 根据 c 的不同值进行不同的操作
  switch(c)
    {
    case OP_END:
    return 0;

    case OP_TYPESTAR:
    case OP_TYPEMINSTAR:
    case OP_TYPEPLUS:
    case OP_TYPEMINPLUS:
    case OP_TYPEQUERY:
    case OP_TYPEMINQUERY:
    case OP_TYPEPOSSTAR:
    case OP_TYPEPOSPLUS:
    case OP_TYPEPOSQUERY:
    # 如果 code[1] 等于 OP_PROP 或者 code[1] 等于 OP_NOTPROP，则将 code 加 2
    if (code[1] == OP_PROP || code[1] == OP_NOTPROP) code += 2;
    break;

    case OP_TYPEUPTO:
    case OP_TYPEMINUPTO:
    case OP_TYPEEXACT:
    case OP_TYPEPOSUPTO:
    # 如果 code[1 + IMM2_SIZE] 等于 OP_PROP 或者 code[1 + IMM2_SIZE] 等于 OP_NOTPROP，则将 code 加 2
    if (code[1 + IMM2_SIZE] == OP_PROP || code[1 + IMM2_SIZE] == OP_NOTPROP)
      code += 2;
    break;

    case OP_CALLOUT_STR:
    # 将 code 加上 GET(code, 1 + 2*LINK_SIZE) 的值
    code += GET(code, 1 + 2*LINK_SIZE);
    break;

#ifdef SUPPORT_WIDE_CHARS
    case OP_XCLASS:
    # 将 code 加上 GET(code, 1) 的值
    code += GET(code, 1);
    break;
#endif

    case OP_MARK:
    case OP_COMMIT_ARG:
    # 其他情况不做处理
    # 如果操作码是 OP_PRUNE_ARG 或者 OP_SKIP_ARG 或者 OP_THEN_ARG，则将代码加上第二个元素的值
    case OP_PRUNE_ARG:
    case OP_SKIP_ARG:
    case OP_THEN_ARG:
    code += code[1];
    break;
    }

  /* 从表中添加固定长度 */
  code += PRIV(OP_lengths)[c];

  /* 在 UTF-8 和 UTF-16 模式下，后面跟着字符的操作码可能后面跟着多字节字符。表中的长度是一个最小值，所以我们必须安排跳过额外的代码单元。 */
#ifdef MAYBE_UTF_MULTI
  # 如果定义了 MAYBE_UTF_MULTI，则根据 utf 变量的值进行条件判断
  if (utf) switch(c)
    {
    # 根据不同的操作码进行条件判断
    case OP_CHAR:
    case OP_CHARI:
    case OP_NOT:
    ...
    # 对于有额外长度的操作码，跳过额外长度的字节
    if (HAS_EXTRALEN(code[-1])) code += GET_EXTRALEN(code[-1]);
    break;
    }
#else
  # 如果没有定义 MAYBE_UTF_MULTI，则忽略 utf 变量，保持编译器的正常运行
  (void)(utf);  /* Keep compiler happy by referencing function argument */
#endif  /* SUPPORT_WIDE_CHARS */
  }
}

/* End of pcre2_auto_possess.c */
```