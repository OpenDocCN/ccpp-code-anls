# `nmap\libpcre\src\pcre2_convert.c`

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

// 包含 pcre2_internal.h 头文件


#define TYPE_OPTIONS (PCRE2_CONVERT_GLOB| \
  PCRE2_CONVERT_POSIX_BASIC|PCRE2_CONVERT_POSIX_EXTENDED)

// 定义 TYPE_OPTIONS 宏，包括 PCRE2_CONVERT_GLOB、PCRE2_CONVERT_POSIX_BASIC 和 PCRE2_CONVERT_POSIX_EXTENDED


#define ALL_OPTIONS (PCRE2_CONVERT_UTF|PCRE2_CONVERT_NO_UTF_CHECK| \
  PCRE2_CONVERT_GLOB_NO_WILD_SEPARATOR| \
  PCRE2_CONVERT_GLOB_NO_STARSTAR| \
  TYPE_OPTIONS)

// 定义 ALL_OPTIONS 宏，包括 PCRE2_CONVERT_UTF、PCRE2_CONVERT_NO_UTF_CHECK、PCRE2_CONVERT_GLOB_NO_WILD_SEPARATOR、PCRE2_CONVERT_GLOB_NO_STARSTAR 和 TYPE_OPTIONS


#define DUMMY_BUFFER_SIZE 100

// 定义 DUMMY_BUFFER_SIZE 宏，值为 100


/* Generated pattern fragments */

// 生成的模式片段


#define STR_BACKSLASH_A STR_BACKSLASH STR_A
#define STR_BACKSLASH_z STR_BACKSLASH STR_z
#define STR_COLON_RIGHT_SQUARE_BRACKET STR_COLON STR_RIGHT_SQUARE_BRACKET
#define STR_DOT_STAR_LOOKBEHIND STR_DOT STR_ASTERISK STR_LEFT_PARENTHESIS STR_QUESTION_MARK STR_LESS_THAN_SIGN STR_EQUALS_SIGN
#define STR_LOOKAHEAD_NOT_DOT STR_LEFT_PARENTHESIS STR_QUESTION_MARK STR_EXCLAMATION_MARK STR_BACKSLASH STR_DOT STR_RIGHT_PARENTHESIS
#define STR_QUERY_s STR_LEFT_PARENTHESIS STR_QUESTION_MARK STR_s STR_RIGHT_PARENTHESIS
#define STR_STAR_NUL STR_LEFT_PARENTHESIS STR_ASTERISK STR_N STR_U STR_L STR_RIGHT_PARENTHESIS

// 定义一系列字符串宏


/* States for POSIX processing */

// POSIX 处理的状态


enum { POSIX_START_REGEX, POSIX_ANCHORED, POSIX_NOT_BRACKET,
       POSIX_CLASS_NOT_STARTED, POSIX_CLASS_STARTING, POSIX_CLASS_STARTED };

// 定义枚举类型，表示 POSIX 处理的不同状态


/* Macro to add a character string to the output buffer, checking for overflow. */

// 宏，用于将字符字符串添加到输出缓冲区，并检查是否溢出


#define PUTCHARS(string) \
  { \
  for (s = (char *)(string); *s != 0; s++) \
    { \
    if (p >= endp) return PCRE2_ERROR_NOMEMORY; \
    *p++ = *s; \
    } \
  }

// 定义 PUTCHARS 宏，用于将字符字符串添加到输出缓冲区，并检查是否溢出


/* Literals that must be escaped: \ ? * + | . ^ $ { } [ ] ( ) */

// 必须转义的字面值：\ ? * + | . ^ $ { } [ ] ( )
/* 定义转义的元字符，用于正则表达式的转义 */
static const char *pcre2_escaped_literals =
  STR_BACKSLASH STR_QUESTION_MARK STR_ASTERISK STR_PLUS
  STR_VERTICAL_LINE STR_DOT STR_CIRCUMFLEX_ACCENT STR_DOLLAR_SIGN
  STR_LEFT_CURLY_BRACKET STR_RIGHT_CURLY_BRACKET
  STR_LEFT_SQUARE_BRACKET STR_RIGHT_SQUARE_BRACKET
  STR_LEFT_PARENTHESIS STR_RIGHT_PARENTHESIS;

/* 在 POSIX 基本模式中识别的转义元字符 */
static const char *posix_meta_escapes =
  STR_LEFT_PARENTHESIS STR_RIGHT_PARENTHESIS
  STR_LEFT_CURLY_BRACKET STR_RIGHT_CURLY_BRACKET
  STR_1 STR_2 STR_3 STR_4 STR_5 STR_6 STR_7 STR_8 STR_9;

/*************************************************
*           转换 POSIX 模式                        *
*************************************************/

/* 此函数处理基本和扩展的 POSIX 模式。

参数：
  pattype        模式类型
  pattern        模式
  plength        代码单元长度
  utf            如果是 UTF 则为 TRUE
  use_buffer     输出位置
  use_length     use_buffer 的长度
  bufflenptr     用于放置使用的长度
  dummyrun       如果是虚拟运行则为 TRUE
  ccontext       转换上下文

返回值：         0 => 成功
                !0 => 错误代码
*/

static int
convert_posix(uint32_t pattype, PCRE2_SPTR pattern, PCRE2_SIZE plength,
  BOOL utf, PCRE2_UCHAR *use_buffer, PCRE2_SIZE use_length,
  PCRE2_SIZE *bufflenptr, BOOL dummyrun, pcre2_convert_context *ccontext)
{
char *s;
PCRE2_SPTR posix = pattern;
PCRE2_UCHAR *p = use_buffer;
PCRE2_UCHAR *pp = p;
PCRE2_UCHAR *endp = p + use_length - 1;  /* 允许有结尾的零 */
PCRE2_SIZE convlength = 0;

uint32_t bracount = 0;
uint32_t posix_state = POSIX_START_REGEX;
uint32_t lastspecial = 0;
BOOL extended = (pattype & PCRE2_CONVERT_POSIX_EXTENDED) != 0;
BOOL nextisliteral = FALSE;

(void)utf;       /* 当不支持 Unicode 时不使用 */
(void)ccontext;  /* 目前未使用 */

/* 初始化错误偏移的默认值为输入的结尾。 */

*bufflenptr = plength;
PUTCHARS(STR_STAR_NUL);
/* 现在扫描输入。 */

while (plength > 0)
  {
  uint32_t c, sc;
  int clength = 1;

  /* 添加上一个项目的长度，然后，如果是虚拟运行，将指针拉回到（临时）缓冲区的开头，然后记住下一个项目的开头。 */

  convlength += p - pp;
  if (dummyrun) p = use_buffer;
  pp = p;

  /* 获取下一个字符 */

#ifndef SUPPORT_UNICODE
  c = *posix;
#else
  GETCHARLENTEST(c, posix, clength);
#endif
  posix += clength;
  plength -= clength;

  sc = nextisliteral? 0 : c;
  nextisliteral = FALSE;

  /* 处理类内的字符。 */

  if (posix_state >= POSIX_CLASS_NOT_STARTED)
    {
    if (c == CHAR_RIGHT_SQUARE_BRACKET)
      {
      PUTCHARS(STR_RIGHT_SQUARE_BRACKET);
      posix_state = POSIX_NOT_BRACKET;
      }

    /* 不是类的结尾 */

    else
      {
      switch (posix_state)
        {
        case POSIX_CLASS_STARTED:
        if (c <= 127 && islower(c)) break;  /* 保持在开始状态 */
        posix_state = POSIX_CLASS_NOT_STARTED;
        if (c == CHAR_COLON  && plength > 0 &&
            *posix == CHAR_RIGHT_SQUARE_BRACKET)
          {
          PUTCHARS(STR_COLON_RIGHT_SQUARE_BRACKET);
          plength--;
          posix++;
          continue;    /* 继续处理 :] 后面的字符 */
          }
        /* 继续执行 */

        case POSIX_CLASS_NOT_STARTED:
        if (c == CHAR_LEFT_SQUARE_BRACKET)
          posix_state = POSIX_CLASS_STARTING;
        break;

        case POSIX_CLASS_STARTING:
        if (c == CHAR_COLON) posix_state = POSIX_CLASS_STARTED;
        break;
        }

      if (c == CHAR_BACKSLASH) PUTCHARS(STR_BACKSLASH);
      if (p + clength > endp) return PCRE2_ERROR_NOMEMORY;
      memcpy(p, posix - clength, CU2BYTES(clength));
      p += clength;
      }
    }

  /* 处理不在类内的字符。 */

  else switch(sc)
    {
    case CHAR_LEFT_SQUARE_BRACKET:
    PUTCHARS(STR_LEFT_SQUARE_BRACKET);

#ifdef NEVER
    /* 如果模式长度大于等于6，检查是否符合特殊情况 [[:<:]] 和 [[:>:]] 的格式（PCRE 支持这种格式），但它们不是 POSIX 1003.1 的一部分。 */
    if (plength >= 6)
      {
      /* 如果符合特殊格式，将其复制到输出缓冲区中，并更新指针和长度 */
      if (posix[0] == CHAR_LEFT_SQUARE_BRACKET &&
          posix[1] == CHAR_COLON &&
          (posix[2] == CHAR_LESS_THAN_SIGN ||
           posix[2] == CHAR_GREATER_THAN_SIGN) &&
          posix[3] == CHAR_COLON &&
          posix[4] == CHAR_RIGHT_SQUARE_BRACKET &&
          posix[5] == CHAR_RIGHT_SQUARE_BRACKET)
        {
        if (p + 6 > endp) return PCRE2_ERROR_NOMEMORY;
        memcpy(p, posix, CU2BYTES(6));
        p += 6;
        posix += 6;
        plength -= 6;
        continue;  /* 继续处理下一个字符 */
        }
      }
#endif

    /* 处理“正常”字符类的开始 */

    posix_state = POSIX_CLASS_NOT_STARTED;

    /* 处理^和]作为第一个字符 */

    if (plength > 0)
      {
      if (*posix == CHAR_CIRCUMFLEX_ACCENT)
        {
        posix++;
        plength--;
        PUTCHARS(STR_CIRCUMFLEX_ACCENT);
        }
      if (plength > 0 && *posix == CHAR_RIGHT_SQUARE_BRACKET)
        {
        posix++;
        plength--;
        PUTCHARS(STR_RIGHT_SQUARE_BRACKET);
        }
      }
    break;

    case CHAR_BACKSLASH:
    if (plength == 0) return PCRE2_ERROR_END_BACKSLASH;
    if (extended) nextisliteral = TRUE; else
      {
      if (*posix < 127 && strchr(posix_meta_escapes, *posix) != NULL)
        {
        if (isdigit(*posix)) PUTCHARS(STR_BACKSLASH);
        if (p + 1 > endp) return PCRE2_ERROR_NOMEMORY;
        lastspecial = *p++ = *posix++;
        plength--;
        }
      else nextisliteral = TRUE;
      }
    break;

    case CHAR_RIGHT_PARENTHESIS:
    if (!extended || bracount == 0) goto ESCAPE_LITERAL;
    bracount--;
    goto COPY_SPECIAL;

    case CHAR_LEFT_PARENTHESIS:
    bracount++;
    /* 继续执行 */

    case CHAR_QUESTION_MARK:
    case CHAR_PLUS:
    case CHAR_LEFT_CURLY_BRACKET:
    case CHAR_RIGHT_CURLY_BRACKET:
    case CHAR_VERTICAL_LINE:
    if (!extended) goto ESCAPE_LITERAL;
    /* 继续执行 */

    case CHAR_DOT:
    case CHAR_DOLLAR_SIGN:
    posix_state = POSIX_NOT_BRACKET;
    COPY_SPECIAL:
    lastspecial = c;
    if (p + 1 > endp) return PCRE2_ERROR_NOMEMORY;
    *p++ = c;
    break;

    case CHAR_ASTERISK:
    if (lastspecial != CHAR_ASTERISK)
      {
      if (!extended && (posix_state < POSIX_NOT_BRACKET ||
          lastspecial == CHAR_LEFT_PARENTHESIS))
        goto ESCAPE_LITERAL;
      goto COPY_SPECIAL;
      }
    break;   /* 忽略第二个及后续的星号 */

    case CHAR_CIRCUMFLEX_ACCENT:
    if (extended) goto COPY_SPECIAL;
    # 如果当前状态为 POSIX_START_REGEX 或者上一个特殊字符为左括号
    if (posix_state == POSIX_START_REGEX ||
        lastspecial == CHAR_LEFT_PARENTHESIS)
      {
      # 设置 POSIX 状态为 POSIX_ANCHORED
      posix_state = POSIX_ANCHORED;
      # 跳转到 COPY_SPECIAL 标签处
      goto COPY_SPECIAL;
      }
    # 如果不符合上述条件，则执行下面的代码

    # 默认情况
    default:
    # 如果字符 c 小于 128 并且在转义字符列表中
    if (c < 128 && strchr(pcre2_escaped_literals, c) != NULL)
      {
      # 转义字符处理
      ESCAPE_LITERAL:
      # 输出反斜杠
      PUTCHARS(STR_BACKSLASH);
      }
    # 上一个特殊字符设置为 0xff，表示没有特殊字符
    lastspecial = 0xff;  /* Indicates nothing special */
    # 如果 p + clength 大于 endp，则返回内存不足错误
    if (p + clength > endp) return PCRE2_ERROR_NOMEMORY;
    # 复制 posix - clength 处的内容到 p 处
    memcpy(p, posix - clength, CU2BYTES(clength));
    # p 向后移动 clength 个位置
    p += clength;
    # 设置 POSIX 状态为 POSIX_NOT_BRACKET
    posix_state = POSIX_NOT_BRACKET;
    # 结束当前的 switch 语句
    break;
    }
  }
# 如果 POSIX 状态大于等于 POSIX_CLASS_NOT_STARTED，则返回错误码 PCRE2_ERROR_MISSING_SQUARE_BRACKET
if (posix_state >= POSIX_CLASS_NOT_STARTED)
  return PCRE2_ERROR_MISSING_SQUARE_BRACKET;
# 将 convlength 增加 p 和 pp 之间的长度，作为最终段的长度
convlength += p - pp;        /* Final segment */
# 将 bufflenptr 指向的值设置为 convlength
*bufflenptr = convlength;
# 将 p 指向的位置设置为 0
*p++ = 0;
# 返回 0，表示成功
return 0;
}


/*************************************************
*           Convert a glob pattern               *
*************************************************/

# 定义写入输出缓冲区的上下文
typedef struct pcre2_output_context {
  PCRE2_UCHAR *output;                  /* current output position */
  PCRE2_SPTR output_end;                /* output end */
  PCRE2_SIZE output_size;               /* size of the output */
  uint8_t out_str[8];                   /* string copied to the output */
} pcre2_output_context;


# 将一个字符写入输出
static void
convert_glob_write(pcre2_output_context *out, PCRE2_UCHAR chr)
{
# 输出大小增加 1
out->output_size++;

# 如果输出位置小于输出结束位置，则将字符写入输出
if (out->output < out->output_end)
  *out->output++ = chr;
}


# 将一个字符串写入输出
static void
convert_glob_write_str(pcre2_output_context *out, PCRE2_SIZE length)
{
# 获取输出字符串和输出位置
uint8_t *out_str = out->out_str;
PCRE2_UCHAR *output = out->output;
PCRE2_SPTR output_end = out->output_end;
PCRE2_SIZE output_size = out->output_size;

# 循环写入字符串中的字符到输出中
do
  {
  output_size++;

  if (output < output_end)
    *output++ = *out_str++;
  }
while (--length != 0);

# 更新输出位置和输出大小
out->output = output;
out->output_size = output_size;
}


# 将分隔符打印到输出中
static void
convert_glob_print_separator(pcre2_output_context *out,
  PCRE2_UCHAR separator, BOOL with_escape)
{
# 如果需要转义，则在分隔符前面加上反斜杠
if (with_escape)
  convert_glob_write(out, CHAR_BACKSLASH);

# 将分隔符写入输出
convert_glob_write(out, separator);
}


# 将通配符打印到输出中
/* 
   将通配符转换为对应的正则表达式字符
   参数：
     out            输出上下文
     separator      通配符分隔符
     with_escape    分隔符前是否需要反斜杠
*/
static void
convert_glob_print_wildcard(pcre2_output_context *out,
  PCRE2_UCHAR separator, BOOL with_escape)
{
  // 将左方括号写入输出字符串
  out->out_str[0] = CHAR_LEFT_SQUARE_BRACKET;
  // 将脱字符写入输出字符串
  out->out_str[1] = CHAR_CIRCUMFLEX_ACCENT;
  // 将字符串写入输出
  convert_glob_write_str(out, 2);

  // 打印分隔符
  convert_glob_print_separator(out, separator, with_escape);

  // 将右方括号写入输出
  convert_glob_write(out, CHAR_RIGHT_SQUARE_BRACKET);
}

/* 解析 POSIX 类
   参数：
     from           扫描范围的起始点
     pattern_end    模式的结束点
     out            输出上下文
   返回值： 
     >0 => 类索引
     0  => 类格式错误
*/
static int
convert_glob_parse_class(PCRE2_SPTR *from, PCRE2_SPTR pattern_end,
  pcre2_output_context *out)
{
  // POSIX 类的字符串表示
  static const char *posix_classes = "alnum:alpha:ascii:blank:cntrl:digit:"
    "graph:lower:print:punct:space:upper:word:xdigit:";
  // 起始点
  PCRE2_SPTR start = *from + 1;
  // 模式
  PCRE2_SPTR pattern = start;
  // 类指针
  const char *class_ptr;
  // 字符
  PCRE2_UCHAR c;
  // 类索引
  int class_index;

  // 循环直到遇到非小写字母
  while (TRUE)
  {
    // 如果超出模式范围，则返回0
    if (pattern >= pattern_end) return 0;

    // 获取字符
    c = *pattern++;

    // 如果字符不是小写字母，则跳出循环
    if (c < CHAR_a || c > CHAR_z) break;
  }

  // 如果字符不是冒号，或者超出模式范围，或者下一个字符不是右方括号，则返回0
  if (c != CHAR_COLON || pattern >= pattern_end ||
      *pattern != CHAR_RIGHT_SQUARE_BRACKET)
    return 0;

  // 初始化类指针和类索引
  class_ptr = posix_classes;
  class_index = 1;

  // 循环直到遇到空字符
  while (TRUE)
  {
    // 如果类指针指向空字符，则返回0
    if (*class_ptr == CHAR_NUL) return 0;

    // 重置模式
    pattern = start;

    // 循环直到模式字符和类指针字符不相等
    while (*pattern == (PCRE2_UCHAR) *class_ptr)
    {
      // 如果模式字符是冒号
      if (*pattern == CHAR_COLON)
      {
        // 移动模式指针，减去2
        pattern += 2;
        start -= 2;

        // 将从起始点到模式指针之间的字符写入输出
        do convert_glob_write(out, *start++); while (start < pattern);

        // 更新 from 指针
        *from = pattern;
        // 返回类索引
        return class_index;
      }
      // 移动模式指针和类指针
      pattern++;
      class_ptr++;
    }

    // 循环直到类指针指向冒号
    while (*class_ptr != CHAR_COLON) class_ptr++;
    // 移动类指针
    class_ptr++;
    // 增加类索引
    class_index++;
  }
}

/* 检查字符是否在类中
   参数：
     class_index    类索引
     c              字符
   返回值：
     !0 => 字符在类中
     0 => 否则
*/
static BOOL
# 根据给定的类别索引和字符判断字符是否符合该类别
convert_glob_char_in_class(int class_index, PCRE2_UCHAR c)
{
    # 根据类别索引进行不同的判断
    switch (class_index)
    {
        # 判断字符是否为字母或数字
        case 1: return isalnum(c);
        # 判断字符是否为字母
        case 2: return isalpha(c);
        # 无条件返回真
        case 3: return 1;
        # 判断字符是否为水平制表符或空格
        case 4: return c == CHAR_HT || c == CHAR_SPACE;
        # 判断字符是否为控制字符
        case 5: return iscntrl(c);
        # 判断字符是否为数字
        case 6: return isdigit(c);
        # 判断字符是否为可打印字符
        case 7: return isgraph(c);
        # 判断字符是否为小写字母
        case 8: return islower(c);
        # 判断字符是否为可打印字符
        case 9: return isprint(c);
        # 判断字符是否为标点符号
        case 10: return ispunct(c);
        # 判断字符是否为空白字符
        case 11: return isspace(c);
        # 判断字符是否为大写字母
        case 12: return isupper(c);
        # 判断字符是否为字母、数字或下划线
        case 13: return isalnum(c) || c == CHAR_UNDERSCORE;
        # 默认情况下判断字符是否为十六进制数字
        default: return isxdigit(c);
    }
}

/* 解析字符范围。

参数：
  from           范围扫描的起始点
  pattern_end    模式的结束点
  out            输出上下文
  separator      通配符分隔符
  with_escape    分隔符前是否需要反斜杠
  escape         转义字符
  no_wildsep     是否允许通配符分隔符

返回值：         0 => 成功
                !0 => 错误代码
*/

static int
convert_glob_parse_range(PCRE2_SPTR *from, PCRE2_SPTR pattern_end,
  pcre2_output_context *out, BOOL utf, PCRE2_UCHAR separator,
  BOOL with_escape, PCRE2_UCHAR escape, BOOL no_wildsep)
{
    BOOL is_negative = FALSE;
    BOOL separator_seen = FALSE;
    BOOL has_prev_c;
    PCRE2_SPTR pattern = *from;
    PCRE2_SPTR char_start = NULL;
    uint32_t c, prev_c;
    int len, class_index;

    (void)utf; # 避免编译器警告

    if (pattern >= pattern_end)
    {
        *from = pattern;
        return PCRE2_ERROR_MISSING_SQUARE_BRACKET;
    }

    if (*pattern == CHAR_EXCLAMATION_MARK
        || *pattern == CHAR_CIRCUMFLEX_ACCENT)
    {
        pattern++;

        if (pattern >= pattern_end)
        {
            *from = pattern;
            return PCRE2_ERROR_MISSING_SQUARE_BRACKET;
        }

        is_negative = TRUE;

        out->out_str[0] = CHAR_LEFT_SQUARE_BRACKET;
        out->out_str[1] = CHAR_CIRCUMFLEX_ACCENT;
        len = 2;

        if (!no_wildsep)
        {
            if (with_escape)
            {
                out->out_str[len] = CHAR_BACKSLASH;
                len++;
            }
            out->out_str[len] = (uint8_t) separator;
        }

        convert_glob_write_str(out, len + 1);
    }
    else
        convert_glob_write(out, CHAR_LEFT_SQUARE_BRACKET);

    has_prev_c = FALSE;
}
# 初始化变量 prev_c 为 0
prev_c = 0;

# 如果模式的第一个字符是右方括号
if (*pattern == CHAR_RIGHT_SQUARE_BRACKET)
  {
  # 将转义字符和右方括号写入输出字符串
  out->out_str[0] = CHAR_BACKSLASH;
  out->out_str[1] = CHAR_RIGHT_SQUARE_BRACKET;
  convert_glob_write_str(out, 2);
  # 设置标志表示前一个字符是右方括号
  has_prev_c = TRUE;
  # 更新 prev_c 为右方括号
  prev_c = CHAR_RIGHT_SQUARE_BRACKET;
  # 移动指针到下一个字符
  pattern++;
  }

# 当模式未结束时循环
while (pattern < pattern_end)
  {
  # 记录当前字符的起始位置
  char_start = pattern;
  # 获取下一个字符并将其存储在 c 中
  GETCHARINCTEST(c, pattern);

  # 如果当前字符是右方括号
  if (c == CHAR_RIGHT_SQUARE_BRACKET)
    {
    # 将右方括号写入输出字符串
    convert_glob_write(out, c);

    # 如果不是负类且不是无通配符分隔符，则执行以下操作
    if (!is_negative && !no_wildsep && separator_seen)
      {
      # 写入左括号、问号、小于号、感叹号到输出字符串
      out->out_str[0] = CHAR_LEFT_PARENTHESIS;
      out->out_str[1] = CHAR_QUESTION_MARK;
      out->out_str[2] = CHAR_LESS_THAN_SIGN;
      out->out_str[3] = CHAR_EXCLAMATION_MARK;
      convert_glob_write_str(out, 4);

      # 打印分隔符到输出字符串
      convert_glob_print_separator(out, separator, with_escape);
      # 写入右括号到输出字符串
      convert_glob_write(out, CHAR_RIGHT_PARENTHESIS);
      }

    # 更新 from 指针并返回 0
    *from = pattern;
    return 0;
    }

  # 如果模式已经结束则跳出循环
  if (pattern >= pattern_end) break;

  # 如果当前字符是左方括号且下一个字符是冒号
  if (c == CHAR_LEFT_SQUARE_BRACKET && *pattern == CHAR_COLON)
    {
    # 更新 from 指针并解析类
    *from = pattern;
    class_index = convert_glob_parse_class(from, pattern_end, out);

    # 如果类索引不为 0
    if (class_index != 0)
      {
      pattern = *from;

      # 重置标志和 prev_c
      has_prev_c = FALSE;
      prev_c = 0;

      # 如果不是负类且字符在类中，则设置分隔符标志
      if (!is_negative &&
          convert_glob_char_in_class (class_index, separator))
        separator_seen = TRUE;
      continue;
      }
    }
  # 如果当前字符是减号且前一个字符存在且下一个字符不是右方括号
  else if (c == CHAR_MINUS && has_prev_c &&
           *pattern != CHAR_RIGHT_SQUARE_BRACKET)
    {
    # 将减号写入输出字符串
    convert_glob_write(out, CHAR_MINUS);

    # 记录当前字符的起始位置
    char_start = pattern;
    # 获取下一个字符并将其存储在 c 中
    GETCHARINCTEST(c, pattern);

    # 如果模式已经结束则跳出循环
    if (pattern >= pattern_end) break;

    # 如果转义字符存在且当前字符等于转义字符
    if (escape != 0 && c == escape)
      {
      # 记录当前字符的起始位置
      char_start = pattern;
      # 获取下一个字符并将其存储在 c 中
      GETCHARINCTEST(c, pattern);
      }
    # 如果当前字符是左方括号且下一个字符是冒号
    else if (c == CHAR_LEFT_SQUARE_BRACKET && *pattern == CHAR_COLON)
      {
      # 更新 from 指针并返回转换语法错误
      *from = pattern;
      return PCRE2_ERROR_CONVERT_SYNTAX;
      }

    # 如果前一个字符大于当前字符
    if (prev_c > c)
      {
      # 更新 from 指针并返回转换语法错误
      *from = pattern;
      return PCRE2_ERROR_CONVERT_SYNTAX;
      }
    # 如果前一个字符小于分隔符并且分隔符小于当前字符，则标记分隔符已见
    if (prev_c < separator && separator < c) separator_seen = TRUE;

    # 标记前一个字符不存在
    has_prev_c = FALSE;
    # 将前一个字符置为0
    prev_c = 0;
    }
  else
    {
    # 如果转义字符不为0并且当前字符等于转义字符
    if (escape != 0 && c == escape)
      {
      # 将字符起始位置置为模式的起始位置
      char_start = pattern;
      # 获取下一个字符并测试是否到达模式的末尾
      GETCHARINCTEST(c, pattern);

      # 如果到达模式的末尾则跳出循环
      if (pattern >= pattern_end) break;
      }

    # 标记前一个字符存在
    has_prev_c = TRUE;
    # 将前一个字符置为当前字符
    prev_c = c;
    }

  # 如果当前字符为左方括号、右方括号、反斜杠或减号，则转义字符写入输出
  if (c == CHAR_LEFT_SQUARE_BRACKET || c == CHAR_RIGHT_SQUARE_BRACKET ||
      c == CHAR_BACKSLASH || c == CHAR_MINUS)
    convert_glob_write(out, CHAR_BACKSLASH);

  # 如果当前字符为分隔符，则标记分隔符已见
  if (c == separator) separator_seen = TRUE;

  # 将字符起始位置到模式结束位置的字符写入输出
  do convert_glob_write(out, *char_start++); while (char_start < pattern);
  }
/* 设置指针 from 指向 pattern */
from = pattern;
/* 返回错误代码 PCRE2_ERROR_MISSING_SQUARE_BRACKET */
return PCRE2_ERROR_MISSING_SQUARE_BRACKET;
}


/* 将 (*COMMIT) 打印到输出中。

参数：
  out            输出上下文
*/
static void
convert_glob_print_commit(pcre2_output_context *out)
{
/* 将 (*COMMIT) 写入输出字符串 */
out->out_str[0] = CHAR_LEFT_PARENTHESIS;
out->out_str[1] = CHAR_ASTERISK;
out->out_str[2] = CHAR_C;
out->out_str[3] = CHAR_O;
out->out_str[4] = CHAR_M;
out->out_str[5] = CHAR_M;
out->out_str[6] = CHAR_I;
out->out_str[7] = CHAR_T;
/* 将字符串写入输出 */
convert_glob_write_str(out, 8);
/* 写入右括号到输出 */
convert_glob_write(out, CHAR_RIGHT_PARENTHESIS);
}


/* Bash 通配符转换器。

参数：
  pattype        模式类型
  pattern        模式
  plength        代码单元长度
  utf            如果是 UTF 则为 TRUE
  use_buffer     输出放置位置
  use_length     use_buffer 的长度
  bufflenptr     输出已使用长度的位置
  dummyrun       如果是虚拟运行则为 TRUE
  ccontext       转换上下文

返回值：         0 => 成功
                !0 => 错误代码
*/
static int
convert_glob(uint32_t options, PCRE2_SPTR pattern, PCRE2_SIZE plength,
  BOOL utf, PCRE2_UCHAR *use_buffer, PCRE2_SIZE use_length,
  PCRE2_SIZE *bufflenptr, BOOL dummyrun, pcre2_convert_context *ccontext)
{
pcre2_output_context out;
PCRE2_SPTR pattern_start = pattern;
PCRE2_SPTR pattern_end = pattern + plength;
PCRE2_UCHAR separator = ccontext->glob_separator;
PCRE2_UCHAR escape = ccontext->glob_escape;
PCRE2_UCHAR c;
BOOL no_wildsep = (options & PCRE2_CONVERT_GLOB_NO_WILD_SEPARATOR) != 0;
BOOL no_starstar = (options & PCRE2_CONVERT_GLOB_NO_STARSTAR) != 0;
BOOL in_atomic = FALSE;
BOOL after_starstar = FALSE;
BOOL no_slash_z = FALSE;
BOOL with_escape, is_start, after_separator;
int result = 0;

(void)utf; /* 避免编译器警告 */

#ifdef SUPPORT_UNICODE
if (utf && (separator >= 128 || escape >= 128))
  {
  /* 目前只支持 ASCII 字符。 */
  *bufflenptr = 0;
  return PCRE2_ERROR_CONVERT_SYNTAX;
  }
#endif

with_escape = strchr(pcre2_escaped_literals, separator) != NULL;
# 初始化错误偏移量的默认值为输入的结尾。
out.output = use_buffer;
out.output_end = use_buffer + use_length;
out.output_size = 0;

# 将字符串 "(?s)" 写入输出缓冲区
out.out_str[0] = CHAR_LEFT_PARENTHESIS;
out.out_str[1] = CHAR_QUESTION_MARK;
out.out_str[2] = CHAR_s;
out.out_str[3] = CHAR_RIGHT_PARENTHESIS;
convert_glob_write_str(&out, 4);

# 设置起始标志为真
is_start = TRUE;

# 如果模式的第一个字符是 '*'，则根据条件设置起始标志
if (pattern < pattern_end && pattern[0] == CHAR_ASTERISK)
  {
  if (no_wildsep)
    is_start = FALSE;
  else if (!no_starstar && pattern + 1 < pattern_end &&
           pattern[1] == CHAR_ASTERISK)
    is_start = FALSE;
  }

# 如果起始标志为真，则将字符串 "\A" 写入输出缓冲区
if (is_start)
  {
  out.out_str[0] = CHAR_BACKSLASH;
  out.out_str[1] = CHAR_A;
  convert_glob_write_str(&out, 2);
  }

# 遍历模式字符串
while (pattern < pattern_end)
  {
  c = *pattern++;

  # 如果当前字符是 '*'，则根据条件设置起始标志
  if (c == CHAR_ASTERISK)
    {
    is_start = pattern == pattern_start + 1;

    # 如果在原子组内，则写入 ')' 并将原子组标志设为假
    if (in_atomic)
      {
      convert_glob_write(&out, CHAR_RIGHT_PARENTHESIS);
      in_atomic = FALSE;
      }
    # 如果不是 no_starstar 并且模式小于模式结束，并且模式指向的字符是星号
    if (!no_starstar && pattern < pattern_end && *pattern == CHAR_ASTERISK)
      {
      # 判断是否在起始位置或者在分隔符之后
      after_separator = is_start || (pattern[-2] == separator);

      # 跳过连续的星号
      do pattern++; while (pattern < pattern_end &&
                           *pattern == CHAR_ASTERISK);

      # 如果已经到达模式结束，则设置 no_slash_z 为真并跳出循环
      if (pattern >= pattern_end)
        {
        no_slash_z = TRUE;
        break;
        }

      # 设置 after_starstar 为真
      after_starstar = TRUE;

      # 如果在分隔符之后，并且转义字符不为0，并且模式指向的字符是转义字符，并且下一个字符是分隔符，则跳过转义字符
      if (after_separator && escape != 0 && *pattern == escape &&
          pattern + 1 < pattern_end && pattern[1] == separator)
        pattern++;

      # 如果在起始位置，则继续下一次循环
      if (is_start)
        {
        if (*pattern != separator) continue;

        # 设置输出字符串为特定字符序列
        out.out_str[0] = CHAR_LEFT_PARENTHESIS;
        out.out_str[1] = CHAR_QUESTION_MARK;
        out.out_str[2] = CHAR_COLON;
        out.out_str[3] = CHAR_BACKSLASH;
        out.out_str[4] = CHAR_A;
        out.out_str[5] = CHAR_VERTICAL_LINE;
        convert_glob_write_str(&out, 6);

        # 打印分隔符
        convert_glob_print_separator(&out, separator, with_escape);
        # 写入右括号
        convert_glob_write(&out, CHAR_RIGHT_PARENTHESIS);

        # 跳过当前字符并继续下一次循环
        pattern++;
        continue;
        }

      # 打印已经匹配的部分
      convert_glob_print_commit(&out);

      # 如果不在分隔符之后或者模式指向的字符不是分隔符
      if (!after_separator || *pattern != separator)
        {
        # 设置输出字符串为特定字符序列
        out.out_str[0] = CHAR_DOT;
        out.out_str[1] = CHAR_ASTERISK;
        out.out_str[2] = CHAR_QUESTION_MARK;
        convert_glob_write_str(&out, 3);
        continue;
        }

      # 设置输出字符串为特定字符序列
      out.out_str[0] = CHAR_LEFT_PARENTHESIS;
      out.out_str[1] = CHAR_QUESTION_MARK;
      out.out_str[2] = CHAR_COLON;
      out.out_str[3] = CHAR_DOT;
      out.out_str[4] = CHAR_ASTERISK;
      out.out_str[5] = CHAR_QUESTION_MARK;

      convert_glob_write_str(&out, 6);

      # 打印分隔符
      convert_glob_print_separator(&out, separator, with_escape);

      # 设置输出字符串为特定字符序列
      out.out_str[0] = CHAR_RIGHT_PARENTHESIS;
      out.out_str[1] = CHAR_QUESTION_MARK;
      out.out_str[2] = CHAR_QUESTION_MARK;
      convert_glob_write_str(&out, 3);

      # 跳过当前字符并继续下一次循环
      pattern++;
      continue;
      }
    # 如果模式小于模式结束并且当前字符为*号
    if (pattern < pattern_end && *pattern == CHAR_ASTERISK)
      {
      # 跳过所有连续的*号
      do pattern++; while (pattern < pattern_end &&
                           *pattern == CHAR_ASTERISK);
      }

    # 如果不允许通配符分隔符
    if (no_wildsep)
      {
      # 如果模式已经结束，则设置no_slash_z为TRUE并跳出循环
      if (pattern >= pattern_end)
        {
        no_slash_z = TRUE;
        break;
        }

      # 如果是起始检查，则继续下一次循环
      # 注意：起始检查必须在结束检查之后进行
      if (is_start) continue;
      }

    # 如果不是起始检查
    if (!is_start)
      {
      # 如果在**之后，则输出"(? >"并设置in_atomic为TRUE
      if (after_starstar)
        {
        out.out_str[0] = CHAR_LEFT_PARENTHESIS;
        out.out_str[1] = CHAR_QUESTION_MARK;
        out.out_str[2] = CHAR_GREATER_THAN_SIGN;
        convert_glob_write_str(&out, 3);
        in_atomic = TRUE;
        }
      # 否则，输出当前结果并提交
      else
        convert_glob_print_commit(&out);
      }

    # 如果不允许通配符分隔符，则输出"."
    if (no_wildsep)
      convert_glob_write(&out, CHAR_DOT);
    # 否则，输出通配符
    else
      convert_glob_print_wildcard(&out, separator, with_escape);

    # 输出"*"和"?"
    out.out_str[0] = CHAR_ASTERISK;
    out.out_str[1] = CHAR_QUESTION_MARK;
    # 如果模式已经结束，则将第二个字符设置为"+"
    if (pattern >= pattern_end)
      out.out_str[1] = CHAR_PLUS;
    convert_glob_write_str(&out, 2);
    # 继续下一次循环
    continue;
    }

  # 如果当前字符为"?"
  if (c == CHAR_QUESTION_MARK)
    {
    # 如果不允许通配符分隔符，则输出"."
    if (no_wildsep)
      convert_glob_write(&out, CHAR_DOT);
    # 否则，输出通配符
    else
      convert_glob_print_wildcard(&out, separator, with_escape);
    # 继续下一次循环
    continue;
    }

  # 如果当前字符为"["
  if (c == CHAR_LEFT_SQUARE_BRACKET)
    {
    # 解析范围并继续下一次循环
    result = convert_glob_parse_range(&pattern, pattern_end,
      &out, utf, separator, with_escape, escape, no_wildsep);
    if (result != 0) break;
    continue;
    }

  # 如果有转义字符并且当前字符为转义字符
  if (escape != 0 && c == escape)
    {
    # 如果模式已经结束，则设置错误代码并跳出循环
    if (pattern >= pattern_end)
      {
      result = PCRE2_ERROR_CONVERT_SYNTAX;
      break;
      }
    # 获取下一个字符
    c = *pattern++;
    }

  # 如果当前字符小于128并且是pcre2_escaped_literals中的转义字符，则输出"\"
  if (c < 128 && strchr(pcre2_escaped_literals, c) != NULL)
    convert_glob_write(&out, CHAR_BACKSLASH);

  # 输出当前字符
  convert_glob_write(&out, c);
  }
if (result == 0)
  {
  // 如果结果为0，则执行以下代码块
  if (!no_slash_z)
    {
    // 如果不是no_slash_z模式，则在输出字符串的开头添加反斜杠和z
    out.out_str[0] = CHAR_BACKSLASH;
    out.out_str[1] = CHAR_z;
    convert_glob_write_str(&out, 2);
    }

  // 如果在原子组内部，则在输出字符串中添加右括号
  if (in_atomic)
    convert_glob_write(&out, CHAR_RIGHT_PARENTHESIS);

  // 在输出字符串中添加空字符
  convert_glob_write(&out, CHAR_NUL);

  // 如果不是虚拟运行，并且输出大小不等于use_buffer的大小，则将结果设置为内存不足错误
  if (!dummyrun && out.output_size != (PCRE2_SIZE) (out.output - use_buffer))
    result = PCRE2_ERROR_NOMEMORY;
  }

if (result != 0)
  {
  // 如果结果不为0，则将bufflenptr设置为pattern与pattern_start之间的偏移量，并返回结果
  *bufflenptr = pattern - pattern_start;
  return result;
  }

// 将bufflenptr设置为out.output_size减1，并返回0
*bufflenptr = out.output_size - 1;
return 0;
}


/*************************************************
*                Convert pattern                 *
*************************************************/

/* This is the external-facing function for converting other forms of pattern
into PCRE2 regular expression patterns. On error, the bufflenptr argument is
used to return an offset in the original pattern.

Arguments:
  pattern     the input pattern
  plength     length of input, or PCRE2_ZERO_TERMINATED
  options     options bits
  buffptr     pointer to pointer to output buffer
  bufflenptr  pointer to length of output buffer
  ccontext    convert context or NULL

Returns:      0 for success, else an error code (+ve or -ve)
*/

// 这是用于将其他形式的模式转换为PCRE2正则表达式模式的外部函数。在出错时，bufflenptr参数用于返回原始模式中的偏移量。
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_pattern_convert(PCRE2_SPTR pattern, PCRE2_SIZE plength, uint32_t options,
  PCRE2_UCHAR **buffptr, PCRE2_SIZE *bufflenptr,
  pcre2_convert_context *ccontext)
{
// 定义变量
int i, rc;
PCRE2_UCHAR dummy_buffer[DUMMY_BUFFER_SIZE];
PCRE2_UCHAR *use_buffer = dummy_buffer;
PCRE2_SIZE use_length = DUMMY_BUFFER_SIZE;
BOOL utf = (options & PCRE2_CONVERT_UTF) != 0;
uint32_t pattype = options & TYPE_OPTIONS;

// 如果pattern为空或bufflenptr为空，则返回空指针错误
if (pattern == NULL || bufflenptr == NULL) return PCRE2_ERROR_NULL;

// 如果选项中包含未定义的位，则返回错误；如果模式类型中设置了多个类型，则返回错误
if ((options & ~ALL_OPTIONS) != 0 ||        
    (pattype & (~pattype+1)) != pattype ||  
    # 如果没有设置类型
    pattype == 0)                           /* No type set */
  {
    # 将缓冲区长度指针指向0，表示错误偏移量
  *bufflenptr = 0;                          /* Error offset */
    # 返回错误代码，表示选项错误
  return PCRE2_ERROR_BADOPTION;
  }
# 如果模式长度为零终止，则将其设置为模式的长度
if (plength == PCRE2_ZERO_TERMINATED) plength = PRIV(strlen)(pattern);
# 如果上下文为空，则使用默认转换上下文
if (ccontext == NULL) ccontext = (pcre2_convert_context *)(&PRIV(default_convert_context));

# 如果需要，检查 UTF
#ifndef SUPPORT_UNICODE
# 如果不支持 UTF，则返回错误
if (utf)
  {
  *bufflenptr = 0;  /* 错误偏移量 */
  return PCRE2_ERROR_UNICODE_NOT_SUPPORTED;
  }
#else
# 如果需要检查 UTF，并且选项中未设置 PCRE2_CONVERT_NO_UTF_CHECK，则进行 UTF 校验
if (utf && (options & PCRE2_CONVERT_NO_UTF_CHECK) == 0)
  {
  PCRE2_SIZE erroroffset;
  rc = PRIV(valid_utf)(pattern, plength, &erroroffset);
  if (rc != 0)
    {
    *bufflenptr = erroroffset;
    return rc;
    }
  }
#endif

# 如果 buffptr 不为空，并且其指向的内容也不为空，则使用提供的缓冲区和长度
if (buffptr != NULL && *buffptr != NULL)
  {
  use_buffer = *buffptr;
  use_length = *bufflenptr;
  }

# 调用单个转换器，如果提供了缓冲区或只需要长度，则只调用一次；如果需要内存分配，则调用两次
for (i = 0; i < 2; i++)
  {
  PCRE2_UCHAR *allocated;
  BOOL dummyrun = buffptr == NULL || *buffptr == NULL;

  switch(pattype)
    {
    case PCRE2_CONVERT_GLOB:
    # 调用转换为 glob 格式的函数
    rc = convert_glob(options & ~PCRE2_CONVERT_GLOB, pattern, plength, utf, use_buffer, use_length, bufflenptr, dummyrun, ccontext);
    break;

    case PCRE2_CONVERT_POSIX_BASIC:
    case PCRE2_CONVERT_POSIX_EXTENDED:
    # 调用转换为 POSIX 格式的函数
    rc = convert_posix(pattype, pattern, plength, utf, use_buffer, use_length, bufflenptr, dummyrun, ccontext);
    break;

    default:
    *bufflenptr = 0;  /* 错误偏移量 */
    return PCRE2_ERROR_INTERNAL;
    }

  # 如果出现错误，或者只需要长度，或者提供了或分配了缓冲区，则返回结果
  if (rc != 0 ||           /* 错误 */
      buffptr == NULL ||   /* 只需要长度 */
      *buffptr != NULL)    /* 提供了或分配了缓冲区 */
    return rc;

  # 为缓冲区分配内存，其中包含一个分配器的隐藏空间。下一次循环将运行真正的转换
  allocated = PRIV(memctl_malloc)(sizeof(pcre2_memctl) +
  ...
  # 分配内存，大小为(bufflenptr + 1)*PCRE2_CODE_UNIT_WIDTH，使用ccontext进行内存控制
  allocated = pcre2_maketables((bufflenptr + 1)*PCRE2_CODE_UNIT_WIDTH, (pcre2_memctl *)ccontext);
  # 如果分配失败，返回内存不足错误
  if (allocated == NULL) return PCRE2_ERROR_NOMEMORY;
  # 将buffptr指向分配的内存起始位置
  *buffptr = (PCRE2_UCHAR *)(((char *)allocated) + sizeof(pcre2_memctl));

  # 将use_buffer指向buffptr
  use_buffer = *buffptr;
  # 将use_length设置为bufflenptr + 1
  use_length = *bufflenptr + 1;
# 控制不应该到达这里，如果到达这里，返回内部错误代码
return PCRE2_ERROR_INTERNAL;
}

# 释放转换后的模式
# 这个函数用于释放在新分配的内存中放置的转换后的模式
# 参数：转换后的模式
# 返回：无
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_converted_pattern_free(PCRE2_UCHAR *converted)
{
if (converted != NULL)
  {
  pcre2_memctl *memctl =
    (pcre2_memctl *)((char *)converted - sizeof(pcre2_memctl));
  memctl->free(memctl, memctl->memory_data);
  }
}

# pcre2_convert.c 文件结束
```