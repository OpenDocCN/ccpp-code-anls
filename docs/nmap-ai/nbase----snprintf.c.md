# `nmap\nbase\snprintf.c`

```
/*
 * 从 tcpdump-2000-9-17 CVS 快照 (www.tcpdump.org) 获取的文件
 * 稍作修改以与 libnbase 兼容。修改细节可能在 nbase CHANGELOG 中 - fyodor@nmap.org
 */

/*
 * 版权所有 1995-1999 年 Kungliga Tekniska Högskolan
 * (瑞典斯德哥尔摩皇家理工学院)。
 * 保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，
 * 都是允许的，只要满足以下条件：
 *
 * 1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 *
 * 2. 以二进制形式再分发必须在文档和/或其他提供的材料中重现上述版权声明、
 *    此条件列表和以下免责声明。
 *
 * 3. 未经特定事先书面许可，不得使用学院的名称或其贡献者的名称来认可或推广
 *    从本软件派生的产品。
 *
 * 本软件由学院和贡献者“按原样”提供，任何明示或暗示的保证，
 * 包括但不限于适销性和特定用途适用性的暗示保证，都是被拒绝的。
 * 在任何情况下，无论是在合同、严格责任或侵权 (包括疏忽或其他方式) 的情况下，
 * 学院或贡献者都不对任何直接、间接、偶发、特殊、惩罚性或后果性的损害
 * (包括但不限于替代商品或服务的采购；使用、数据或利润的损失；或业务中断)
 * 负责，即使已被告知可能发生此类损害的可能性。
 */

/* $Id$ */

#if HAVE_CONFIG_H
#include "nbase_config.h"
#else
#ifdef WIN32
#include "nbase_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifndef lint
static const char rcsid[] __attribute__((unused)) =
     "@(#) $Header$";
#endif

#include <stdio.h>  // 包含标准输入输出库
#include <stdarg.h>  // 包含可变参数列表库
#include <stdlib.h>  // 包含标准库
#include <string.h>  // 包含字符串处理库
#include <ctype.h>   // 包含字符处理库
#include <sys/types.h>  // 包含系统类型库

#if HAVE_LIBIBERTY_H
#include <libiberty.h>  // 如果有 libiberty 库，则包含该库
#endif

#include "nbase.h"  // 包含自定义的 nbase 库

#ifndef min
#define min(a,b) ((a)>(b)?(b):(a))  // 定义取最小值的宏
#endif
#ifndef max
#define max(a,b) ((b)>(a)?(b):(a))  // 定义取最大值的宏
#endif

enum format_flags {
    minus_flag     =  1,  // 定义格式化标志位
    plus_flag      =  2,  // 定义格式化标志位
    space_flag     =  4,  // 定义格式化标志位
    alternate_flag =  8,  // 定义格式化标志位
    zero_flag      = 16   // 定义格式化标志位
};

/*
 * Common state
 */

struct state {
  unsigned char *str;  // 无符号字符指针
  unsigned char *s;    // 无符号字符指针
  unsigned char *theend;  // 无符号字符指针
  size_t sz;  // 大小类型
  size_t max_sz;  // 大小类型
  int (*append_char)(struct state *, unsigned char);  // 函数指针，指向接受 state 结构和无符号字符的函数
  int (*reserve)(struct state *, size_t);  // 函数指针，指向接受 state 结构和大小类型的函数
  /* XXX - methods */
};

#ifndef HAVE_VSNPRINTF
static int
sn_reserve (struct state *state, size_t n)  // 静态函数，接受 state 结构和大小类型的参数
{
  return state->s + n > state->theend;  // 返回比较结果
}

static int
sn_append_char (struct state *state, unsigned char c)  // 静态函数，接受 state 结构和无符号字符的参数
{
  if (sn_reserve (state, 1)) {  // 如果满足条件
    return 1;  // 返回 1
  } else {
    *state->s++ = c;  // 否则将字符添加到 state 结构中
    return 0;  // 返回 0
  }
}
#endif

static int
as_reserve (struct state *state, size_t n)  // 静态函数，接受 state 结构和大小类型的参数
{
  if (state->s + n > state->theend) {  // 如果满足条件
    int off = state->s - state->str;  // 计算偏移量
    unsigned char *tmp;  // 无符号字符指针

    if (state->max_sz && state->sz >= state->max_sz)  // 如果满足条件
      return 1;  // 返回 1

    state->sz = max(state->sz * 2, state->sz + n);  // 更新大小类型
    if (state->max_sz)  // 如果满足条件
      state->sz = min(state->sz, state->max_sz);  // 更新大小类型
    tmp = safe_realloc(state->str, state->sz);  // 重新分配内存
    state->str = tmp;  // 更新指针
    state->s = state->str + off;  // 更新指针
    state->theend = state->str + state->sz - 1;  // 更新指针
  }
  return 0;  // 返回 0
}

static int
as_append_char (struct state *state, unsigned char c)  // 静态函数，接受 state 结构和无符号字符的参数
{
  if(as_reserve (state, 1))  // 如果满足条件
    return 1;  // 返回 1
  else {
    *state->s++ = c;  // 否则将字符添加到 state 结构中
    return 0;  // 返回 0
  }
}

static int
# 向状态结构体中追加数字
append_number(struct state *state,
              unsigned long num, unsigned base, char *rep,
              int width, int prec, int flags, int minusp)
{
  int len = 0;  # 初始化长度为0
  int i;

  # 如果有精度，则忽略零标志
  if(prec != -1)
    flags &= ~zero_flag;
  else
    prec = 1;
  # 零值且精度为零 -> ""
  if(prec == 0 && num == 0)
    return 0;
  do{
    if((*state->append_char)(state, rep[num % base]))  # 追加字符到状态结构体
      return 1;
    len++;
    num /= base;
  }while(num);
  prec -= len;
  # 用精度补零
  while(prec-- > 0){
    if((*state->append_char)(state, '0'))
      return 1;
    len++;
  }
  # 添加备用前缀的长度到长度中
  if(flags & alternate_flag && (base == 16 || base == 8))
    len += base / 8;
  # 用零补齐
  if(flags & zero_flag):
    width -= len;
    if(minusp || (flags & space_flag) || (flags & plus_flag))
      width--;
    while(width-- > 0):
      if((*state->append_char)(state, '0'))
        return 1;
      len++;
  # 添加备用前缀
  if(flags & alternate_flag && (base == 16 || base == 8)):
    if(base == 16)
      if((*state->append_char)(state, rep[10] + 23))  # XXX
        return 1;
    if((*state->append_char)(state, '0'))
      return 1;
  # 添加符号
  if(minusp):
    if((*state->append_char)(state, '-'))
      return 1;
    len++;
  else if(flags & plus_flag):
    if((*state->append_char)(state, '+'))
      return 1;
    len++;
  else if(flags & space_flag):
    if((*state->append_char)(state, ' '))
      return 1;
    len++;
  if(flags & minus_flag)
    # 在用空格补齐之前交换
    for(i = 0; i < len / 2; i++):
      char c = state->s[-i-1];
      state->s[-i-1] = state->s[-len+i];
      state->s[-len+i] = c;
  width -= len;
  while(width-- > 0):
    if((*state->append_char)(state,  ' '))
      return 1;
    len++;
  if(!(flags & minus_flag))
    # 在用空格补齐之后交换
    # 从索引 0 开始循环到长度的一半
    for(i = 0; i < len / 2; i++){
      # 交换字符串中对应位置的字符
      char c = state->s[-i-1];
      state->s[-i-1] = state->s[-len+i];
      state->s[-len+i] = c;
    }
  # 返回 0，表示操作成功
  return 0;
static int
append_string (struct state *state,
               unsigned char *arg,
               int width,
               int prec,
               int flags)
{
  // 如果参数为空，则将参数指向字符串 "(null)"
  if(!arg)
    arg = (unsigned char *) "(null)";
  // 如果有指定精度，则宽度减去精度
  if(prec != -1)
    width -= prec;
  else
    // 否则宽度减去字符串长度
    width -= strlen((char *)arg);
  // 如果没有指定左对齐标志，则在宽度范围内填充空格
  if(!(flags & minus_flag))
    while(width-- > 0)
      if((*state->append_char) (state, ' '))
        return 1;
  // 如果有指定精度，则按照精度输出字符
  if (prec != -1) {
    while (*arg && prec--)
      if ((*state->append_char) (state, *arg++))
        return 1;
  } else {
    // 否则输出整个字符串
    while (*arg)
      if ((*state->append_char) (state, *arg++))
        return 1;
  }
  // 如果有指定左对齐标志，则在宽度范围内填充空格
  if(flags & minus_flag)
    while(width-- > 0)
      if((*state->append_char) (state, ' '))
        return 1;
  return 0;
}

static int
append_char(struct state *state,
            unsigned char arg,
            int width,
            int flags)
{
  // 如果没有指定左对齐标志，则在宽度范围内填充空格
  while(!(flags & minus_flag) && --width > 0)
    if((*state->append_char) (state, ' '))
      return 1;

  // 输出字符
  if((*state->append_char) (state, arg))
    return 1;
  // 如果有指定左对齐标志，则在宽度范围内填充空格
  while((flags & minus_flag) && --width > 0)
    if((*state->append_char) (state, ' '))
      return 1;

  return 0;
}

/*
 * This can't be made into a function...
 */

#define PARSE_INT_FORMAT(res, arg, unsig) \
if (long_flag) \
     res = (unsig long)va_arg(arg, unsig long); \
else if (short_flag) \
     res = (unsig short)va_arg(arg, unsig int); \
else \
     res = (unsig int)va_arg(arg, unsig int)

/*
 * zyxprintf - return 0 or -1
 */

static int
xyzprintf (struct state *state, const char *char_format, va_list ap)
{
  // 将格式字符串转换为无符号字符指针
  const unsigned char *format = (const unsigned char *)char_format;
  unsigned char c;

  // 遍历格式字符串
  while((c = *format++)) {
    // 如果遇到 '%' 字符，则调用 append_char 函数输出下一个字符
    if (c == '%') {
      c = *format++;
      if ((*state->append_char) (state, c))
        return -1;
    } else {
      // 否则直接输出字符
      if ((*state->append_char) (state, c))
        return -1;
    }
  }
  return 0;
}

#ifndef HAVE_SNPRINTF
int
snprintf (char *str, size_t sz, const char *format, ...)
{
  va_list args;
  int ret;

  va_start(args, format);
  // 调用 vsnprintf 函数
  ret = vsnprintf (str, sz, format, args);
  va_end(args);

#ifdef PARANOIA
  {
    int ret2;
    // 声明一个指向字符的指针变量tmp
    char *tmp;

    // 使用safe_malloc函数分配内存，并将指针赋值给tmp
    tmp = safe_malloc (sz);

    // 初始化可变参数列表args，以便访问传递给函数的可变参数
    va_start(args, format);

    // 将格式化的数据写入tmp指向的内存中，返回写入的字符数
    ret2 = vsprintf (tmp, format, args);

    // 结束可变参数的访问
    va_end(args);

    // 如果vsprintf返回的字符数与ret不相等，或者str与tmp不相等，则终止程序
    if (ret != ret2 || strcmp(str, tmp))
      abort ();

    // 释放tmp指向的内存
    free (tmp);
  }
#ifndef HAVE_VSNPRINTF
// 如果没有定义 HAVE_VSNPRINTF，则定义函数 vsnprintf
int
vsnprintf (char *str, size_t sz, const char *format, va_list args)
#endif

#ifndef HAVE_ASNPRINTF
// 如果没有定义 HAVE_ASNPRINTF，则定义函数 asnprintf
int
asnprintf (char **ret, size_t max_sz, const char *format, ...)
#endif

#ifndef HAVE_ASPRINTF
// 如果没有定义 HAVE_ASPRINTF，则定义函数 asprintf
int
asprintf (char **ret, const char *format, ...)
#endif

#ifndef HAVE_VASPRINTF
// 如果没有定义 HAVE_VASPRINTF，则定义函数 vasprintf
int
vasprintf (char **ret, const char *format, va_list args)
#endif

#ifndef HAVE_VASNPRINTF
// 如果没有定义 HAVE_VASNPRINTF，则定义函数 vasnprintf
int
vasnprintf (char **ret, size_t max_sz, const char *format, va_list args)
#endif
# 定义结构体 state
struct state state;
# 定义变量 ret，用于存储函数返回值
int ret;
# 将字符串转换为无符号字符指针
unsigned char *ustr = (unsigned char *)str;

# 初始化 state 结构体的各个成员变量
state.max_sz = 0;
state.sz     = sz;
state.str    = ustr;
state.s      = ustr;
state.theend = ustr + sz - 1;
state.append_char = sn_append_char;
state.reserve     = sn_reserve;

# 调用 xyzprintf 函数，将格式化后的字符串写入 state 结构体
ret = xyzprintf (&state, format, args);
# 将 state.s 指向的位置置为字符串结束符'\0'
*state.s = '\0';
# 如果 ret 不为 0，则返回 -1，否则返回 state.s 指针减去 state.str 指针的值
if (ret)
  return -1;
else
  return state.s - state.str;
# 结束 if 语句
# 结束函数
#endif
```