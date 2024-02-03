# `nmap\libpcre\src\pcre2_substitute.c`

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

#ifdef HAVE_CONFIG_H：如果定义了HAVE_CONFIG_H，则包含config.h文件
#include "config.h"：包含config.h文件


#include "pcre2_internal.h"

#include "pcre2_internal.h"：包含pcre2_internal.h文件


#define PTR_STACK_SIZE 20

#define PTR_STACK_SIZE 20：定义PTR_STACK_SIZE为20


#define SUBSTITUTE_OPTIONS \
  (PCRE2_SUBSTITUTE_EXTENDED|PCRE2_SUBSTITUTE_GLOBAL| \
   PCRE2_SUBSTITUTE_LITERAL|PCRE2_SUBSTITUTE_MATCHED| \
   PCRE2_SUBSTITUTE_OVERFLOW_LENGTH|PCRE2_SUBSTITUTE_REPLACEMENT_ONLY| \
   PCRE2_SUBSTITUTE_UNKNOWN_UNSET|PCRE2_SUBSTITUTE_UNSET_EMPTY)

#define SUBSTITUTE_OPTIONS \：定义SUBSTITUTE_OPTIONS为一系列PCRE2_SUBSTITUTE开头的选项的组合


static int
find_text_end(const pcre2_code *code, PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend,
  BOOL last)

static int find_text_end(const pcre2_code *code, PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend, BOOL last)：定义一个静态函数find_text_end，接受pcre2_code指针、PCRE2_SPTR指针、BOOL类型的last参数，并返回int类型的值


int rc = 0;
uint32_t nestlevel = 0;
BOOL literal = FALSE;
PCRE2_SPTR ptr = *ptrptr;

int rc = 0;：定义int类型的变量rc并初始化为0
uint32_t nestlevel = 0;：定义uint32_t类型的变量nestlevel并初始化为0
BOOL literal = FALSE;：定义BOOL类型的变量literal并初始化为FALSE
PCRE2_SPTR ptr = *ptrptr;：定义PCRE2_SPTR类型的变量ptr并初始化为*ptrptr的值


for (; ptr < ptrend; ptr++)
  {
  if (literal)
    {
    if (ptr[0] == CHAR_BACKSLASH && ptr < ptrend - 1 && ptr[1] == CHAR_E)
      {
      literal = FALSE;
      ptr += 1;
      }
    }

for (; ptr < ptrend; ptr++)：循环，当ptr小于ptrend时执行循环，每次循环ptr自增1
if (literal)：如果literal为真
if (ptr[0] == CHAR_BACKSLASH && ptr < ptrend - 1 && ptr[1] == CHAR_E)：如果ptr[0]等于CHAR_BACKSLASH且ptr小于ptrend-1且ptr[1]等于CHAR_E
literal = FALSE;：将literal设置为假
ptr += 1;：ptr自增1


  else if (*ptr == CHAR_RIGHT_CURLY_BRACKET)
    {
    if (nestlevel == 0) goto EXIT;
    nestlevel--;
    }

else if (*ptr == CHAR_RIGHT_CURLY_BRACKET)：否则如果*ptr等于CHAR_RIGHT_CURLY_BRACKET
if (nestlevel == 0) goto EXIT;：如果nestlevel等于0，跳转到EXIT
nestlevel--;：nestlevel自减1


  else if (*ptr == CHAR_COLON && !last && nestlevel == 0) goto EXIT;

else if (*ptr == CHAR_COLON && !last && nestlevel == 0)：否则如果*ptr等于CHAR_COLON且last为假且nestlevel等于0，跳转到EXIT


  else if (*ptr == CHAR_DOLLAR_SIGN)
    {

else if (*ptr == CHAR_DOLLAR_SIGN)：否则如果*ptr等于CHAR_DOLLAR_SIGN
    # 如果指针指向的字符不是最后一个字符，并且下一个字符是左花括号
    if (ptr < ptrend - 1 && ptr[1] == CHAR_LEFT_CURLY_BRACKET)
      {
      # 嵌套级别加一
      nestlevel++;
      # 指针向后移动一位
      ptr += 1;
      }
    }

  # 如果指针指向的字符是反斜杠
  else if (*ptr == CHAR_BACKSLASH)
    {
    int erc;
    int errorcode;
    uint32_t ch;

    # 如果指针指向的字符不是倒数第二个字符
    if (ptr < ptrend - 1) switch (ptr[1])
      {
      # 根据下一个字符的不同情况进行处理
      case CHAR_L:
      case CHAR_l:
      case CHAR_U:
      case CHAR_u:
      # 指针向后移动一位
      ptr += 1;
      # 继续循环
      continue;
      }

    # 指针向后移动一位，指向反斜杠后的字符
    ptr += 1;  /* Must point after \ */
    # 检查转义字符
    erc = PRIV(check_escape)(&ptr, ptrend, &ch, &errorcode,
      code->overall_options, code->extra_options, FALSE, NULL);
    # 指针回退一位，指向转义字符的最后一个字符
    ptr -= 1;  /* Back to last code unit of escape */
    # 如果出现错误
    if (errorcode != 0)
      {
      # 返回错误码
      rc = errorcode;
      # 跳转到退出标签
      goto EXIT;
      }

    # 根据转义字符的不同情况进行处理
    switch(erc)
      {
      # 数据字符或者孤立的\E被忽略
      case 0:      
      case ESC_E:  
      break;

      # 设置literal为真
      case ESC_Q:
      literal = TRUE;
      break;

      # 返回错误码
      default:
      rc = PCRE2_ERROR_BADREPESCAPE;
      # 跳转到退出标签
      goto EXIT;
      }
    }
  }
rc = PCRE2_ERROR_REPMISSINGBRACE;   /* Terminator not found */

EXIT:
*ptrptr = ptr;
return rc;



/*************************************************
*              Match and substitute              *
*************************************************/

/* This function applies a compiled re to a subject string and creates a new
string with substitutions. The first 7 arguments are the same as for
pcre2_match(). Either string length may be PCRE2_ZERO_TERMINATED.

Arguments:
  code            points to the compiled expression
  subject         points to the subject string
  length          length of subject string (may contain binary zeros)
  start_offset    where to start in the subject string
  options         option bits
  match_data      points to a match_data block, or is NULL
  context         points a PCRE2 context
  replacement     points to the replacement string
  rlength         length of replacement string
  buffer          where to put the substituted string
  blength         points to length of buffer; updated to length of string

Returns:          >= 0 number of substitutions made
                  < 0 an error code
                  PCRE2_ERROR_BADREPLACEMENT means invalid use of $
*/

/* This macro checks for space in the buffer before copying into it. On
overflow, either give an error immediately, or keep on, accumulating the
length. */

#define CHECKMEMCPY(from,length) \
  { \
  if (!overflowed && lengthleft < length) \
    { \
    if ((suboptions & PCRE2_SUBSTITUTE_OVERFLOW_LENGTH) == 0) goto NOROOM; \
    overflowed = TRUE; \
    extra_needed = length - lengthleft; \
    } \
  else if (overflowed) \
    { \
    extra_needed += length; \
    }  \
  else \
    {  \
    memcpy(buffer + buff_offset, from, CU2BYTES(length)); \
    buff_offset += length; \
    lengthleft -= length; \
    } \
  }

/* Here's the function */

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
# 定义 pcre2_substitute 函数，接受多个参数
pcre2_substitute(const pcre2_code *code, PCRE2_SPTR subject, PCRE2_SIZE length,
  PCRE2_SIZE start_offset, uint32_t options, pcre2_match_data *match_data,
  pcre2_match_context *mcontext, PCRE2_SPTR replacement, PCRE2_SIZE rlength,
  PCRE2_UCHAR *buffer, PCRE2_SIZE *blength)
{
# 定义变量 rc，subs，forcecase，forcecasereset，ovector_count，goptions，suboptions，internal_match_data，escaped_literal，overflowed，use_existing_match，replacement_only
int rc;
int subs;
int forcecase = 0;
int forcecasereset = 0;
uint32_t ovector_count;
uint32_t goptions = 0;
uint32_t suboptions;
pcre2_match_data *internal_match_data = NULL;
BOOL escaped_literal = FALSE;
BOOL overflowed = FALSE;
BOOL use_existing_match;
BOOL replacement_only;
#ifdef SUPPORT_UNICODE
BOOL utf = (code->overall_options & PCRE2_UTF) != 0;
BOOL ucp = (code->overall_options & PCRE2_UCP) != 0;
#endif
PCRE2_UCHAR temp[6];
PCRE2_SPTR ptr;
PCRE2_SPTR repend;
PCRE2_SIZE extra_needed = 0;
PCRE2_SIZE buff_offset, buff_length, lengthleft, fraglength;
PCRE2_SIZE *ovector;
PCRE2_SIZE ovecsave[3];
pcre2_substitute_callout_block scb;

/* General initialization */

buff_offset = 0;
lengthleft = buff_length = *blength;
*blength = PCRE2_UNSET;
ovecsave[0] = ovecsave[1] = ovecsave[2] = PCRE2_UNSET;

/* Partial matching is not valid. This must come after setting *blength to
PCRE2_UNSET, so as not to imply an offset in the replacement. */

# 如果选项中包含 PCRE2_PARTIAL_HARD 或 PCRE2_PARTIAL_SOFT，返回错误码 PCRE2_ERROR_BADOPTION
if ((options & (PCRE2_PARTIAL_HARD|PCRE2_PARTIAL_SOFT)) != 0)
  return PCRE2_ERROR_BADOPTION;

/* Validate length and find the end of the replacement. A NULL replacement of
zero length is interpreted as an empty string. */

# 验证长度并找到替换的结尾。长度为零的空替换被解释为空字符串
if (replacement == NULL)
  {
  if (rlength != 0) return PCRE2_ERROR_NULL;
  replacement = (PCRE2_SPTR)"";
  }

if (rlength == PCRE2_ZERO_TERMINATED) rlength = PRIV(strlen)(replacement);
repend = replacement + rlength;

/* Check for using a match that has already happened. Note that the subject
pointer in the match data may be NULL after a no-match. */

# 检查是否使用已经发生的匹配。注意，在无匹配后，匹配数据中的主题指针可能为空
use_existing_match = ((options & PCRE2_SUBSTITUTE_MATCHED) != 0);
replacement_only = ((options & PCRE2_SUBSTITUTE_REPLACEMENT_ONLY) != 0);
# 如果从现有匹配开始，必须有外部提供的匹配数据块。我们在两种情况下创建内部匹配数据块：
# (a) 未提供外部匹配数据块（并且我们不是从现有匹配开始）；
# (b) 要对第一个替换使用现有匹配。在后一种情况下，我们将现有匹配复制到内部块中，除了任何缓存的堆栈帧大小和指针。这确保不对外部匹配数据块进行任何更改。

if (match_data == NULL)
  {
  pcre2_general_context *gcontext;
  if (use_existing_match) return PCRE2_ERROR_NULL;
  gcontext = (mcontext == NULL)?
    (pcre2_general_context *)code :
    (pcre2_general_context *)mcontext;
  match_data = internal_match_data =
    pcre2_match_data_create_from_pattern(code, gcontext);
  if (internal_match_data == NULL) return PCRE2_ERROR_NOMEMORY;
  }

else if (use_existing_match)
  {
  pcre2_general_context *gcontext = (mcontext == NULL)?
    (pcre2_general_context *)code :
    (pcre2_general_context *)mcontext;
  int pairs = (code->top_bracket + 1 < match_data->oveccount)?
    code->top_bracket + 1 : match_data->oveccount;
  internal_match_data = pcre2_match_data_create(match_data->oveccount,
    gcontext);
  if (internal_match_data == NULL) return PCRE2_ERROR_NOMEMORY;
  memcpy(internal_match_data, match_data, offsetof(pcre2_match_data, ovector)
    + 2*pairs*sizeof(PCRE2_SIZE));
  internal_match_data->heapframes = NULL;
  internal_match_data->heapframes_size = 0;
  match_data = internal_match_data;
  }

# 记住 ovector 的细节
ovector = pcre2_get_ovector_pointer(match_data);
ovector_count = pcre2_get_ovector_count(match_data);

# 在调用块中固定的内容
scb.version = 0;
scb.input = subject;
scb.output = (PCRE2_SPTR)buffer;
scb.ovector = ovector;

# NULL 主题的长度为零被视为空字符串。
if (subject == NULL)
  {
  if (length != 0) return PCRE2_ERROR_NULL;
  subject = (PCRE2_SPTR)"";
  }
# 如果长度为 PCRE2_ZERO_TERMINATED，则计算字符串长度
if (length == PCRE2_ZERO_TERMINATED)
  length = subject? PRIV(strlen)(subject) : 0;

# 如果需要检查 UTF 替换字符串
#ifdef SUPPORT_UNICODE
if (utf && (options & PCRE2_NO_UTF_CHECK) == 0)
  {
  # 检查 UTF 替换字符串是否有效
  rc = PRIV(valid_utf)(replacement, rlength, &(match_data->startchar));
  if (rc != 0)
    {
    match_data->leftchar = 0;
    goto EXIT;
    }
  }
#endif  /* SUPPORT_UNICODE */

# 保存替换选项并从匹配选项中移除它们
suboptions = options & SUBSTITUTE_OPTIONS;
options &= ~SUBSTITUTE_OPTIONS;

# 如果起始匹配偏移大于主题长度，则报错
if (start_offset > length)
  {
  match_data->leftchar = 0;
  rc = PCRE2_ERROR_BADOFFSET;
  goto EXIT;
  }

# 复制直到起始偏移，除非只需要替换
if (!replacement_only) CHECKMEMCPY(subject, start_offset);

# 全局替换循环。如果设置了 PCRE2_SUBSTITUTE_MATCHED，则从传入的 match_data 中获取第一个匹配
subs = 0;
do
  {
  PCRE2_SPTR ptrstack[PTR_STACK_SIZE];
  uint32_t ptrstackptr = 0;

  if (use_existing_match)
    {
    rc = match_data->rc;
    use_existing_match = FALSE;
    }
  else rc = pcre2_match(code, subject, length, start_offset, options|goptions,
    match_data, mcontext);

#ifdef SUPPORT_UNICODE
  if (utf) options |= PCRE2_NO_UTF_CHECK;  /* 只需要检查一次 */
#endif

  # 除了无匹配之外的任何错误都返回错误代码。在不进行特殊的空匹配全局重新匹配或者在主题末尾时，无匹配会中断全局循环。否则，将起始点向前移动一个字符，将其复制到输出，然后再次尝试。
  if (rc < 0)
    {
    PCRE2_SIZE save_start;

    if (rc != PCRE2_ERROR_NOMATCH) goto EXIT;
    if (goptions == 0 || start_offset >= length) break;

    # 向前移动一个代码点。然后，如果 CRLF 是有效的换行序列且
    # 如果已经进入中间部分，再向前进一个代码点
    # 换句话说，即使CR和LF本身是有效的换行符，也不要从CRLF的中间开始
    save_start = start_offset++;
    # 如果前一个字符是CR，并且换行约定不是PCRE2_NEWLINE_CR或PCRE2_NEWLINE_LF，
    # 并且下一个字符是LF，并且还未到达字符串末尾，则向前进一位
    if (subject[start_offset-1] == CHAR_CR &&
        code->newline_convention != PCRE2_NEWLINE_CR &&
        code->newline_convention != PCRE2_NEWLINE_LF &&
        start_offset < length &&
        subject[start_offset] == CHAR_LF)
      start_offset++;
    # 否则，在UTF模式下，向前进入任何次要代码点
    else if ((code->overall_options & PCRE2_UTF) != 0)
      {
#if PCRE2_CODE_UNIT_WIDTH == 8
      // 如果 PCRE2_CODE_UNIT_WIDTH 等于 8，则执行以下代码块
      while (start_offset < length && (subject[start_offset] & 0xc0) == 0x80)
        // 当 start_offset 小于 length 且 subject[start_offset] 的前两位为 10，则增加 start_offset
        start_offset++;
#elif PCRE2_CODE_UNIT_WIDTH == 16
      // 如果 PCRE2_CODE_UNIT_WIDTH 等于 16，则执行以下代码块
      while (start_offset < length &&
            (subject[start_offset] & 0xfc00) == 0xdc00)
        // 当 start_offset 小于 length 且 subject[start_offset] 的前五位为 11011，则增加 start_offset
        start_offset++;
#endif
      }

    /* Copy what we have advanced past (unless not required), reset the special
    global options, and continue to the next match. */

    // 复制我们已经跳过的内容（除非不需要），重置特殊的全局选项，并继续下一个匹配
    fraglength = start_offset - save_start;
    if (!replacement_only) CHECKMEMCPY(subject + save_start, fraglength);
    goptions = 0;
    continue;
    }

  /* Handle a successful match. Matches that use \K to end before they start
  or start before the current point in the subject are not supported. */

  // 处理成功的匹配。使用 \K 结束在开始之前或开始在主题的当前点之前的匹配不受支持。
  if (ovector[1] < ovector[0] || ovector[0] < start_offset)
    {
    rc = PCRE2_ERROR_BADSUBSPATTERN;
    goto EXIT;
    }

  /* Check for the same match as previous. This is legitimate after matching an
  empty string that starts after the initial match offset. We have tried again
  at the match point in case the pattern is one like /(?<=\G.)/ which can never
  match at its starting point, so running the match achieves the bumpalong. If
  we do get the same (null) match at the original match point, it isn't such a
  pattern, so we now do the empty string magic. In all other cases, a repeat
  match should never occur. */

  // 检查与之前相同的匹配。在匹配初始匹配偏移量之后开始的空字符串后，这是合法的。我们在匹配点再次尝试，以防模式像 /(?<=\G.)/ 这样的模式永远无法在其起始点匹配，因此运行匹配实现了 bumpalong。如果我们在原始匹配点得到相同的（空）匹配，那么它不是这样的模式，所以现在我们做空字符串魔术。在所有其他情况下，不应发生重复匹配。

  if (ovecsave[0] == ovector[0] && ovecsave[1] == ovector[1])
    {
    if (ovector[0] == ovector[1] && ovecsave[2] != start_offset)
      {
      goptions = PCRE2_NOTEMPTY_ATSTART | PCRE2_ANCHORED;
      ovecsave[2] = start_offset;
      continue;    /* Back to the top of the loop */
      }
    rc = PCRE2_ERROR_INTERNAL_DUPMATCH;
    goto EXIT;
    }

  /* Count substitutions with a paranoid check for integer overflow; surely no
  real call to this function would ever hit this! */

  // 使用对整数溢出的偏执检查计算替换次数；当然，真正调用此函数的情况永远不会发生这种情况！
  if (subs == INT_MAX)
    {
    rc = PCRE2_ERROR_TOOMANYREPLACE;
    goto EXIT;
    }
  subs++;

  /* 增加替换次数计数 */

  /* 复制匹配之前的文本（除非不需要），并记住插入的起始位置和设置的匹配偏移量对数。 */

  if (rc == 0) rc = ovector_count;
  fraglength = ovector[0] - start_offset;
  if (!replacement_only) CHECKMEMCPY(subject + start_offset, fraglength);
  scb.output_offsets[0] = buff_offset;
  scb.oveccount = rc;

  /* 处理替换字符串。如果整个替换字符串是字面值，只需复制它并检查长度。 */

  ptr = replacement;
  if ((suboptions & PCRE2_SUBSTITUTE_LITERAL) != 0)
    {
    CHECKMEMCPY(ptr, rlength);
    }

  /* 在非字面值替换中，必须逐个字符扫描，可以通过\Q设置本地字面值模式，但只能在扩展模式下解释反斜杠。在扩展模式下，我们必须处理需要重新处理的嵌套子字符串。 */

  else for (;;)
    {
    uint32_t ch;
    unsigned int chlen;

    /* 如果在嵌套子字符串的末尾，弹出堆栈。 */

    if (ptr >= repend)
      {
      if (ptrstackptr == 0) break;       /* 替换字符串结束 */
      repend = ptrstack[--ptrstackptr];
      ptr = ptrstack[--ptrstackptr];
      continue;
      }

    /* 处理下一个字符 */

    if (escaped_literal)
      {
      if (ptr[0] == CHAR_BACKSLASH && ptr < repend - 1 && ptr[1] == CHAR_E)
        {
        escaped_literal = FALSE;
        ptr += 2;
        continue;
        }
      goto LOADLITERAL;
      }

    /* 非字面值模式。 */
#ifdef SUPPORT_UNICODE
            # 如果支持 Unicode，则进行以下操作
            if (utf || ucp)
              {
              # 获取字符的类型
              uint32_t type = UCD_CHARTYPE(ch);
              # 如果字符是小写字母，并且不是强制转换为大写或小写，则转换为另一个大小写
              if (PRIV(ucp_gentype)[type] == ucp_L &&
                  type != ((forcecase > 0)? ucp_Lu : ucp_Ll))
                ch = UCD_OTHERCASE(ch);
              }
            else
#endif
              {
              # 如果不支持 Unicode，则根据条件进行大小写转换
              if (((code->tables + cbits_offset +
                  ((forcecase > 0)? cbit_upper:cbit_lower)
                  )[ch/8] & (1u << (ch%8))) == 0)
                ch = (code->tables + fcc_offset)[ch];
              }
            # 重置大小写转换标志
            forcecase = forcecasereset;
            }

#ifdef SUPPORT_UNICODE
          # 如果支持 Unicode，则根据条件设置字符长度
          if (utf) chlen = PRIV(ord2utf)(ch, temp); else
#endif
            {
            # 如果不支持 Unicode，则将字符放入临时数组，并设置字符长度为1
            temp[0] = ch;
            chlen = 1;
            }
          # 检查并复制字符到临时数组
          CHECKMEMCPY(temp, chlen);
          }
        }
      }

    /* 处理扩展模式中的转义序列。我们可以使用check_escape()来处理\Q、\E、\c、\o、\x和后面跟着非字母数字的\，但是在pcre2_compile()中不支持大小写转换转义，因此必须在这里识别。 */
    else if ((suboptions & PCRE2_SUBSTITUTE_EXTENDED) != 0 &&  # 如果替换选项中包含 PCRE2_SUBSTITUTE_EXTENDED 并且指针指向的字符是反斜杠
              *ptr == CHAR_BACKSLASH)
      {
      int errorcode;  # 定义错误码变量

      if (ptr < repend - 1) switch (ptr[1])  # 如果指针小于 repend 减 1，根据 ptr[1] 的值进行判断
        {
        case CHAR_L:  # 如果 ptr[1] 的值是 CHAR_L
        forcecase = forcecasereset = -1;  # 设置 forcecase 和 forcecasereset 的值为 -1
        ptr += 2;  # 指针向后移动两位
        continue;  # 继续下一次循环

        case CHAR_l:  # 如果 ptr[1] 的值是 CHAR_l
        forcecase = -1;  # 设置 forcecase 的值为 -1
        forcecasereset = 0;  # 设置 forcecasereset 的值为 0
        ptr += 2;  # 指针向后移动两位
        continue;  # 继续下一次循环

        case CHAR_U:  # 如果 ptr[1] 的值是 CHAR_U
        forcecase = forcecasereset = 1;  # 设置 forcecase 和 forcecasereset 的值为 1
        ptr += 2;  # 指针向后移动两位
        continue;  # 继续下一次循环

        case CHAR_u:  # 如果 ptr[1] 的值是 CHAR_u
        forcecase = 1;  # 设置 forcecase 的值为 1
        forcecasereset = 0;  # 设置 forcecasereset 的值为 0
        ptr += 2;  # 指针向后移动两位
        continue;  # 继续下一次循环

        default:  # 默认情况
        break;  # 结束 switch 语句
        }

      ptr++;  /* Point after \ */  # 指针向后移动一位，指向反斜杠后的字符
      rc = PRIV(check_escape)(&ptr, repend, &ch, &errorcode,  # 调用 PRIV(check_escape) 函数，检查转义字符
        code->overall_options, code->extra_options, FALSE, NULL);
      if (errorcode != 0) goto BADESCAPE;  # 如果错误码不为 0，跳转到 BADESCAPE 标签

      switch(rc)  # 根据 rc 的值进行判断
        {
        case ESC_E:  # 如果 rc 的值是 ESC_E
        forcecase = forcecasereset = 0;  # 设置 forcecase 和 forcecasereset 的值为 0
        continue;  # 继续下一次循环

        case ESC_Q:  # 如果 rc 的值是 ESC_Q
        escaped_literal = TRUE;  # 设置 escaped_literal 的值为 TRUE
        continue;  # 继续下一次循环

        case 0:      /* Data character */  # 如果 rc 的值是 0，表示数据字符
        goto LITERAL;  # 跳转到 LITERAL 标签

        default:  # 默认情况
        goto BADESCAPE;  # 跳转到 BADESCAPE 标签
        }
      }

    /* Handle a literal code unit */

    else  # 如果不满足上述条件
      {
      LOADLITERAL:  # 定义 LOADLITERAL 标签
      GETCHARINCTEST(ch, ptr);    /* Get character value, increment pointer */  # 获取字符值，增加指针位置

      LITERAL:  # 定义 LITERAL 标签
      if (forcecase != 0)  # 如果 forcecase 不等于 0
        {
#ifdef SUPPORT_UNICODE
        # 如果支持 Unicode，则进行以下操作
        if (utf || ucp)
          {
          # 获取字符的类型
          uint32_t type = UCD_CHARTYPE(ch);
          # 如果字符是小写字母，并且不是强制转换为大写或小写，则转换为另一个大小写形式
          if (PRIV(ucp_gentype)[type] == ucp_L &&
              type != ((forcecase > 0)? ucp_Lu : ucp_Ll))
            ch = UCD_OTHERCASE(ch);
          }
        else
#endif
          {
          # 如果不支持 Unicode，则根据 forcecase 的值进行大小写转换
          if (((code->tables + cbits_offset +
              ((forcecase > 0)? cbit_upper:cbit_lower)
              )[ch/8] & (1u << (ch%8))) == 0)
            ch = (code->tables + fcc_offset)[ch];
          }
        # 重置 forcecase 的值
        forcecase = forcecasereset;
        }

#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则根据 utf 的值确定 chlen 的长度，否则直接将 ch 存入 temp 中
      if (utf) chlen = PRIV(ord2utf)(ch, temp); else
#endif
        {
        temp[0] = ch;
        chlen = 1;
        }
      # 检查并复制 temp 中的内容到指定位置
      CHECKMEMCPY(temp, chlen);
      } /* End handling a literal code unit */
    }   /* End of loop for scanning the replacement. */

  /* The replacement has been copied to the output, or its size has been
  remembered. Do the callout if there is one and we have done an actual
  replacement. */

  # 如果没有溢出，并且存在 callout 并且已经进行了实际替换，则进行 callout
  if (!overflowed && mcontext != NULL && mcontext->substitute_callout != NULL)
    {
    # 设置替换的数量和输出偏移量
    scb.subscount = subs;
    scb.output_offsets[1] = buff_offset;
    # 调用 substitute_callout 函数
    rc = mcontext->substitute_callout(&scb, mcontext->substitute_callout_data);

    /* A non-zero return means cancel this substitution. Instead, copy the
    matched string fragment. */

    # 如果返回值不为零，则取消此次替换，复制匹配的字符串片段
    if (rc != 0)
      {
      # 计算新旧长度
      PCRE2_SIZE newlength = scb.output_offsets[1] - scb.output_offsets[0];
      PCRE2_SIZE oldlength = ovector[1] - ovector[0];

      # 调整偏移量和剩余长度
      buff_offset -= newlength;
      lengthleft += newlength;
      # 如果不仅仅是替换，则复制匹配的字符串片段
      if (!replacement_only) CHECKMEMCPY(subject + ovector[0], oldlength);

      /* A negative return means do not do any more. */

      # 如果返回值为负数，则不再进行替换
      if (rc < 0) suboptions &= (~PCRE2_SUBSTITUTE_GLOBAL);
      }
    }

  /* 保存此匹配的详细信息。参见上文中数据如何使用。如果我们匹配了空字符串，执行全局匹配的特殊处理。更新起始偏移量指向主题字符串的剩余部分。如果我们重用了现有的匹配作为第一个匹配，切换到内部匹配数据块。 */

  // 保存匹配的起始和结束位置，以及起始偏移量
  ovecsave[0] = ovector[0];
  ovecsave[1] = ovector[1];
  ovecsave[2] = start_offset;

  // 设置匹配选项，用于全局匹配
  goptions = (ovector[0] != ovector[1] || ovector[0] > start_offset)? 0 :
    PCRE2_ANCHORED|PCRE2_NOTEMPTY_ATSTART;
  // 更新起始偏移量
  start_offset = ovector[1];
  } while ((suboptions & PCRE2_SUBSTITUTE_GLOBAL) != 0);  /* 重复“do”循环 */
/* 如果不仅仅是替换操作，复制主题的剩余部分，然后以二进制零终止输出 */

if (!replacement_only)
  {
  // 计算剩余部分的长度
  fraglength = length - start_offset;
  // 复制剩余部分到输出
  CHECKMEMCPY(subject + start_offset, fraglength);
  }

// 在临时缓冲区中添加一个二进制零
temp[0] = 0;
CHECKMEMCPY(temp, 1);

/* 如果设置了溢出标志，意味着设置了 PCRE2_SUBSTITUTE_OVERFLOW_LENGTH，
并且在完整缓冲区之后匹配已经继续，以计算所需的长度。
否则，溢出会立即生成错误返回。 */

if (overflowed)
  {
  rc = PCRE2_ERROR_NOMEMORY;
  *blength = buff_length + extra_needed;
  }

/* 在成功执行后，返回替换的次数，并设置使用的缓冲区长度，不包括尾随的零。 */

else
  {
  rc = subs;
  *blength = buff_offset - 1;
  }

EXIT:
// 释放内部匹配数据
if (internal_match_data != NULL) pcre2_match_data_free(internal_match_data);
  else match_data->rc = rc;
return rc;

NOROOM:
// 内存不足错误
rc = PCRE2_ERROR_NOMEMORY;
goto EXIT;

BAD:
// 替换字符串错误
rc = PCRE2_ERROR_BADREPLACEMENT;
goto PTREXIT;

BADESCAPE:
// 替换转义字符错误
rc = PCRE2_ERROR_BADREPESCAPE;

PTREXIT:
// 设置替换后的长度
*blength = (PCRE2_SIZE)(ptr - replacement);
goto EXIT;
}

/* pcre2_substitute.c 结束 */
```