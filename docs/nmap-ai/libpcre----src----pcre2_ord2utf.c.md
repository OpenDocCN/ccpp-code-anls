# `nmap\libpcre\src\pcre2_ord2utf.c`

```cpp
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
/* 
   如果使用此软件，即使已被告知可能会造成此类损害，也不会对任何因使用此软件而引起的任何索赔、损害或其他责任负责。
   -----------------------------------------------------------------------------
*/

/* 
   该文件包含一个将 Unicode 字符代码点转换为 UTF 字符串的函数。对于每个代码单元宽度，其行为是不同的。
*/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/* 
   如果未定义 SUPPORT_UNICODE，将永远不会调用此函数。
   提供一个虚拟函数，因为一些编译器不喜欢空的源模块。
*/
#ifndef SUPPORT_UNICODE
unsigned int
PRIV(ord2utf)(uint32_t cvalue, PCRE2_UCHAR *buffer)
{
(void)(cvalue);
(void)(buffer);
return 0;
}
#else  /* SUPPORT_UNICODE */

/*************************************************
*          Convert code point to UTF             *
*************************************************/

/*
   参数：
     cvalue     字符值
     buffer     指向结果缓冲区的指针

   返回值：     放入缓冲区的代码单元数
*/
unsigned int
PRIV(ord2utf)(uint32_t cvalue, PCRE2_UCHAR *buffer)
{
   /* 转换为 UTF-8 */
#if PCRE2_CODE_UNIT_WIDTH == 8
int i, j;
for (i = 0; i < PRIV(utf8_table1_size); i++)
  if ((int)cvalue <= PRIV(utf8_table1)[i]) break;
buffer += i;
for (j = i; j > 0; j--)
 {
 *buffer-- = 0x80 | (cvalue & 0x3f);
 cvalue >>= 6;
 }
*buffer = PRIV(utf8_table2)[i] | cvalue;
return i + 1;

/* 转换为 UTF-16 */
#elif PCRE2_CODE_UNIT_WIDTH == 16
if (cvalue <= 0xffff)
  {
  *buffer = (PCRE2_UCHAR)cvalue;
  return 1;
  }
cvalue -= 0x10000;
*buffer++ = 0xd800 | (cvalue >> 10);
*buffer = 0xdc00 | (cvalue & 0x3ff);
return 2;

/* 转换为 UTF-32 */
#else
*buffer = (PCRE2_UCHAR)cvalue;
return 1;
#endif
}
#endif  /* SUPPORT_UNICODE */

/* End of pcre_ord2utf.c */
```