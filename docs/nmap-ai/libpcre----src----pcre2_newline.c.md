# `nmap\libpcre\src\pcre2_newline.c`

```
# PCRE 是一个支持正则表达式的函数库，其语法和语义尽可能接近 Perl 5 语言
# 作者：Philip Hazel
# 原始 API 代码版权所有 (c) 1997-2012 剑桥大学
# 新 API 代码版权所有 (c) 2016 剑桥大学
# 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都必须满足以下条件：
#   * 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明
#   * 以二进制形式重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
#   * 未经特定事先书面许可，不得使用剑桥大学的名称或其贡献者的名称来认可或推广从本软件派生的产品
# 本软件由版权所有者和贡献者 "按原样" 提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的暗示担保
# 在任何情况下，无论是合同责任、严格责任还是侵权 (包括疏忽或其他情况)，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害承担责任
# （包括但不限于替代商品或服务的采购；使用、数据或利润的损失；或业务中断）造成的任何损害，无论是何种责任理论
/* This module contains internal functions for testing newlines when more than
one kind of newline is to be recognized. When a newline is found, its length is
returned. In principle, we could implement several newline "types", each
referring to a different set of newline characters. At present, PCRE2 supports
only NLTYPE_FIXED, which gets handled without these functions, NLTYPE_ANYCRLF,
and NLTYPE_ANY. The full list of Unicode newline characters is taken from
http://unicode.org/unicode/reports/tr18/. */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/*************************************************
*      Check for newline at given position       *
*************************************************/

/* This function is called only via the IS_NEWLINE macro, which does so only
when the newline type is NLTYPE_ANY or NLTYPE_ANYCRLF. The case of a fixed
newline (NLTYPE_FIXED) is handled inline. It is guaranteed that the code unit
pointed to by ptr is less than the end of the string.

Arguments:
  ptr          pointer to possible newline
  type         the newline type
  endptr       pointer to the end of the string
  lenptr       where to return the length
  utf          TRUE if in utf mode

Returns:       TRUE or FALSE
*/

BOOL
PRIV(is_newline)(PCRE2_SPTR ptr, uint32_t type, PCRE2_SPTR endptr,
  uint32_t *lenptr, BOOL utf)
{
uint32_t c;

#ifdef SUPPORT_UNICODE
if (utf) { GETCHAR(c, ptr); } else c = *ptr;
#else
(void)utf;
c = *ptr;
#endif  /* SUPPORT_UNICODE */

if (type == NLTYPE_ANYCRLF) switch(c)
  {
  case CHAR_LF:
  *lenptr = 1;
  return TRUE;

  case CHAR_CR:
  *lenptr = (ptr < endptr - 1 && ptr[1] == CHAR_LF)? 2 : 1;
  return TRUE;

  default:
  return FALSE;
  }

/* NLTYPE_ANY */

else switch(c)
  {
#ifdef EBCDIC
  case CHAR_NEL:
#endif
  # 如果遇到换行符 LF、VT、FF，则设置长度为 1，返回 TRUE
  case CHAR_LF:
  case CHAR_VT:
  case CHAR_FF:
  *lenptr = 1;
  return TRUE;

  # 如果遇到回车符 CR，则根据下一个字符是否为 LF 设置长度为 1 或 2，返回 TRUE
  case CHAR_CR:
  *lenptr = (ptr < endptr - 1 && ptr[1] == CHAR_LF)? 2 : 1;
  return TRUE;

#ifndef EBCDIC
#if PCRE2_CODE_UNIT_WIDTH == 8
  # 如果遇到 NEL，则根据 utf 模式设置长度为 1 或 2，返回 TRUE
  case CHAR_NEL:
  *lenptr = utf? 2 : 1;
  return TRUE;

  # 如果遇到 LS 或 PS，则设置长度为 3，返回 TRUE
  case 0x2028:   /* LS */
  case 0x2029:   /* PS */
  *lenptr = 3;
  return TRUE;

#else  /* 16-bit or 32-bit code units */
  # 如果遇到 NEL、LS 或 PS，则设置长度为 1，返回 TRUE
  case CHAR_NEL:
  case 0x2028:   /* LS */
  case 0x2029:   /* PS */
  *lenptr = 1;
  return TRUE;
#endif
#endif /* Not EBCDIC */

  # 默认情况下返回 FALSE
  default:
  return FALSE;
  }
}



/*************************************************
*     Check for newline at previous position     *
*************************************************/

/* This function is called only via the WAS_NEWLINE macro, which does so only
when the newline type is NLTYPE_ANY or NLTYPE_ANYCRLF. The case of a fixed
newline (NLTYPE_FIXED) is handled inline. It is guaranteed that the initial
value of ptr is greater than the start of the string that is being processed.

Arguments:
  ptr          pointer to possible newline
  type         the newline type
  startptr     pointer to the start of the string
  lenptr       where to return the length
  utf          TRUE if in utf mode

Returns:       TRUE or FALSE
*/

# 检查前一个位置是否为换行符
BOOL
PRIV(was_newline)(PCRE2_SPTR ptr, uint32_t type, PCRE2_SPTR startptr,
  uint32_t *lenptr, BOOL utf)
{
uint32_t c;
ptr--;

#ifdef SUPPORT_UNICODE
if (utf)
  {
  BACKCHAR(ptr);
  GETCHAR(c, ptr);
  }
else c = *ptr;
#else
(void)utf;
c = *ptr;
#endif  /* SUPPORT_UNICODE */

# 如果换行类型为 NLTYPE_ANYCRLF，则根据前一个字符判断是否为 CRLF 或 LF，设置长度并返回 TRUE
if (type == NLTYPE_ANYCRLF) switch(c)
  {
  case CHAR_LF:
  *lenptr = (ptr > startptr && ptr[-1] == CHAR_CR)? 2 : 1;
  return TRUE;

  case CHAR_CR:
  *lenptr = 1;
  return TRUE;

  default:
  return FALSE;
  }

# 如果换行类型为 NLTYPE_ANY，则根据前一个字符判断是否为 CRLF 或 LF，设置长度并返回 TRUE
else switch(c)
  {
  case CHAR_LF:
  *lenptr = (ptr > startptr && ptr[-1] == CHAR_CR)? 2 : 1;
  return TRUE;

#ifdef EBCDIC
  case CHAR_NEL:
#endif
  case CHAR_VT:  // 如果是垂直制表符
  case CHAR_FF:  // 如果是换页符
  case CHAR_CR:  // 如果是回车符
  *lenptr = 1;   // 设置长度指针为1
  return TRUE;   // 返回真值

#ifndef EBCDIC
#if PCRE2_CODE_UNIT_WIDTH == 8
  case CHAR_NEL:  // 如果是垂直制表符
  *lenptr = utf? 2 : 1;  // 如果是 UTF 编码则长度为2，否则为1
  return TRUE;   // 返回真值

  case 0x2028:   /* LS */  // 如果是行分隔符
  case 0x2029:   /* PS */  // 如果是段分隔符
  *lenptr = 3;   // 设置长度指针为3
  return TRUE;   // 返回真值

#else /* 16-bit or 32-bit code units */
  case CHAR_NEL:  // 如果是垂直制表符
  case 0x2028:   /* LS */  // 如果是行分隔符
  case 0x2029:   /* PS */  // 如果是段分隔符
  *lenptr = 1;   // 设置长度指针为1
  return TRUE;   // 返回真值
#endif
#endif /* Not EBCDIC */

  default:  // 默认情况
  return FALSE;  // 返回假值
  }
}

/* End of pcre2_newline.c */
```