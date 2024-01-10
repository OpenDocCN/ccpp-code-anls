# `nmap\liblua\lobject.c`

```
/*
** $Id: lobject.c $
** Some generic functions over Lua objects
** See Copyright Notice in lua.h
*/

#define lobject_c
#define LUA_CORE

#include "lprefix.h"


#include <locale.h>
#include <math.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "lctype.h"
#include "ldebug.h"
#include "ldo.h"
#include "lmem.h"
#include "lobject.h"
#include "lstate.h"
#include "lstring.h"
#include "lvm.h"


/*
** Computes ceil(log2(x))
*/
// 计算 x 的对数的向上取整
int luaO_ceillog2 (unsigned int x) {
  static const lu_byte log_2[256] = {  /* log_2[i] = ceil(log2(i - 1)) */
    // 预先计算 0 到 255 的对数的向上取整值
    0,1,2,2,3,3,3,3,4,4,4,4,4,4,4,4,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
    7,7,7,7,7,7,7,7,7,7,7,7,7,7
}

static lua_Number numarith (lua_State *L, int op, lua_Number v1,
                                                  lua_Number v2) {
  switch (op) {
    case LUA_OPADD: return luai_numadd(L, v1, v2);  // 执行加法操作
    case LUA_OPSUB: return luai_numsub(L, v1, v2);  // 执行减法操作
    case LUA_OPMUL: return luai_nummul(L, v1, v2);  // 执行乘法操作
    case LUA_OPDIV: return luai_numdiv(L, v1, v2);  // 执行除法操作
    case LUA_OPPOW: return luai_numpow(L, v1, v2);  // 执行幂运算操作
    case LUA_OPIDIV: return luai_numidiv(L, v1, v2);  // 执行整除操作
    case LUA_OPUNM: return luai_numunm(L, v1);  // 执行负数操作
    case LUA_OPMOD: return luaV_modf(L, v1, v2);  // 执行取模操作
    default: lua_assert(0); return 0;  // 默认情况下断言失败
  }
}


int luaO_rawarith (lua_State *L, int op, const TValue *p1, const TValue *p2,
                   TValue *res) {
  switch (op) {
    case LUA_OPBAND: case LUA_OPBOR: case LUA_OPBXOR:
    case LUA_OPSHL: case LUA_OPSHR:
    case LUA_OPBNOT: {  /* 只对整数进行操作 */
      lua_Integer i1; lua_Integer i2;
      if (tointegerns(p1, &i1) && tointegerns(p2, &i2)) {
        setivalue(res, intarith(L, op, i1, i2));  // 设置结果为整数运算的值
        return 1;
      }
      else return 0;  /* 失败 */
    }
    case LUA_OPDIV: case LUA_OPPOW: {  /* 只对浮点数进行操作 */
      lua_Number n1; lua_Number n2;
      if (tonumberns(p1, &n1) && tonumberns(p2, &n2)) {
        setfltvalue(res, numarith(L, op, n1, n2));  // 设置结果为浮点数运算的值
        return 1;
      }
      else return 0;  /* 失败 */
    }
    default: {  /* 其他操作 */
      lua_Number n1; lua_Number n2;
      if (ttisinteger(p1) && ttisinteger(p2)) {
        setivalue(res, intarith(L, op, ivalue(p1), ivalue(p2));  // 设置结果为整数运算的值
        return 1;
      }
      else if (tonumberns(p1, &n1) && tonumberns(p2, &n2)) {
        setfltvalue(res, numarith(L, op, n1, n2));  // 设置结果为浮点数运算的值
        return 1;
      }
      else return 0;  /* 失败 */
    }
  }
}


void luaO_arith (lua_State *L, int op, const TValue *p1, const TValue *p2,
                 StkId res) {
  if (!luaO_rawarith(L, op, p1, p2, s2v(res))) {
    /* 无法执行原始操作；尝试元方法 */
    # 调用 luaT_trybinTM 函数，传入参数 L, p1, p2, res, 以及计算得到的特殊方法号
    luaT_trybinTM(L, p1, p2, res, cast(TMS, (op - LUA_OPADD) + TM_ADD));
  }
/* 
** 函数：luaO_hexavalue
** 作用：将十六进制字符转换为对应的数值
*/
int luaO_hexavalue (int c) {
  // 如果是数字字符，则直接返回对应的数值
  if (lisdigit(c)) return c - '0';
  // 如果是字母字符，则返回对应的数值
  else return (ltolower(c) - 'a') + 10;
}

/* 
** 函数：isneg
** 作用：判断字符串是否为负数
*/
static int isneg (const char **s) {
  // 如果字符串以负号开头，则将指针向后移动一位并返回1
  if (**s == '-') { (*s)++; return 1; }
  // 如果字符串以正号开头，则将指针向后移动一位并返回0
  else if (**s == '+') (*s)++;
  return 0;
}

/* 
** 函数：lua_strx2number
** 作用：将十六进制字符串转换为数字
*/
#if !defined(lua_strx2number)

/* 最大有效数字位数（避免即使是单精度浮点数也会溢出） */
#define MAXSIGDIG    30

/* 将十六进制字符串转换为数字，遵循 C99 规范的 'strtod' */
static lua_Number lua_strx2number (const char *s, char **endptr) {
  int dot = lua_getlocaledecpoint();
  lua_Number r = l_mathop(0.0);  /* 结果（累加器） */
  int sigdig = 0;  /* 有效数字位数 */
  int nosigdig = 0;  /* 无效数字位数 */
  int e = 0;  /* 指数修正 */
  int neg;  /* 如果数字为负数则为1 */
  int hasdot = 0;  /* 是否已经遇到小数点 */
  *endptr = cast_charp(s);  /* 目前还没有有效字符 */
  while (lisspace(cast_uchar(*s))) s++;  /* 跳过开头的空格 */
  neg = isneg(&s);  /* 检查符号 */
  if (!(*s == '0' && (*(s + 1) == 'x' || *(s + 1) == 'X')))  /* 检查是否以 '0x' 开头 */
    return l_mathop(0.0);  /* 格式无效（没有 '0x'） */
  for (s += 2; ; s++) {  /* 跳过 '0x' 并读取数字 */
    if (*s == dot) {
      if (hasdot) break;  /* 第二个小数点？停止循环 */
      else hasdot = 1;
    }
    else if (lisxdigit(cast_uchar(*s))) {
      if (sigdig == 0 && *s == '0')  /* 无效数字位（零）？ */
        nosigdig++;
      else if (++sigdig <= MAXSIGDIG)  /* 可以读取而不会溢出？ */
          r = (r * l_mathop(16.0)) + luaO_hexavalue(*s);
      else e++; /* 数字位数过多；忽略，但仍然计入指数 */
      if (hasdot) e--;  /* 小数位？修正指数 */
    }
    else break;  /* 如果既不是小数点也不是数字，则跳出循环 */
  }
  if (nosigdig + sigdig == 0)  /* 没有数字？ */
    return l_mathop(0.0);  /* 无效的格式 */
  *endptr = cast_charp(s);  /* 到这里是有效的 */
  e *= 4;  /* 每个数字乘以/除以2的4次方 */
  if (*s == 'p' || *s == 'P') {  /* 指数部分？ */
    int exp1 = 0;  /* 指数值 */
    int neg1;  /* 指数符号 */
    s++;  /* 跳过 'p' */
    neg1 = isneg(&s);  /* 符号 */
    if (!lisdigit(cast_uchar(*s)))
      return l_mathop(0.0);  /* 无效；必须至少有一个数字 */
    while (lisdigit(cast_uchar(*s)))  /* 读取指数 */
      exp1 = exp1 * 10 + *(s++) - '0';
    if (neg1) exp1 = -exp1;
    e += exp1;
    *endptr = cast_charp(s);  /* 到这里是有效的 */
  }
  if (neg) r = -r;
  return l_mathop(ldexp)(r, e);
/* }====================================================== */

/* maximum length of a numeral to be converted to a number */
#if !defined (L_MAXLENNUM)
#define L_MAXLENNUM    200
#endif

/*
** Convert string 's' to a Lua number (put in 'result'). Return NULL on
** fail or the address of the ending '\0' on success. ('mode' == 'x')
** means a hexadecimal numeral.
*/
static const char *l_str2dloc (const char *s, lua_Number *result, int mode) {
  char *endptr;
  *result = (mode == 'x') ? lua_strx2number(s, &endptr)  /* try to convert */
                          : lua_str2number(s, &endptr);
  if (endptr == s) return NULL;  /* nothing recognized? */
  while (lisspace(cast_uchar(*endptr))) endptr++;  /* skip trailing spaces */
  return (*endptr == '\0') ? endptr : NULL;  /* OK iff no trailing chars */
}


/*
** Convert string 's' to a Lua number (put in 'result') handling the
** current locale.
** This function accepts both the current locale or a dot as the radix
** mark. If the conversion fails, it may mean number has a dot but
** locale accepts something else. In that case, the code copies 's'
** to a buffer (because 's' is read-only), changes the dot to the
** current locale radix mark, and tries to convert again.
** The variable 'mode' checks for special characters in the string:
** - 'n' means 'inf' or 'nan' (which should be rejected)
** - 'x' means a hexadecimal numeral
** - '.' just optimizes the search for the common case (no special chars)
*/
static const char *l_str2d (const char *s, lua_Number *result) {
  const char *endptr;
  const char *pmode = strpbrk(s, ".xXnN");  /* look for special chars */
  int mode = pmode ? ltolower(cast_uchar(*pmode)) : 0;
  if (mode == 'n')  /* reject 'inf' and 'nan' */
    return NULL;
  endptr = l_str2dloc(s, result, mode);  /* try to convert */
  if (endptr == NULL) {  /* failed? may be a different locale */
    char buff[L_MAXLENNUM + 1];
    const char *pdot = strchr(s, '.');
    # 如果传入的指针为 NULL，或者字符串长度超过最大长度限制，则返回 NULL，表示失败
    if (pdot == NULL || strlen(s) > L_MAXLENNUM)
      return NULL;  /* string too long or no dot; fail */
    # 将字符串复制到缓冲区中
    strcpy(buff, s);  /* copy string to buffer */
    # 将缓冲区中小数点位置的字符替换为当前环境的小数点字符
    buff[pdot - s] = lua_getlocaledecpoint();  /* correct decimal point */
    # 尝试再次将字符串转换为数字，将结果存入 result 中，并返回转换结束的位置指针
    endptr = l_str2dloc(buff, result, mode);  /* try again */
    # 如果转换结束的位置指针不为 NULL，则将其相对于原始字符串 's' 的位置计算出来
    if (endptr != NULL)
      endptr = s + (endptr - buff);  /* make relative to 's' */
  }
  # 返回转换结束的位置指针
  return endptr;
# 定义一个宏，将 LUA_MAXINTEGER 除以 10 的结果转换为 lua_Unsigned 类型
#define MAXBY10        cast(lua_Unsigned, LUA_MAXINTEGER / 10)
# 定义一个宏，将 LUA_MAXINTEGER 取模 10 的结果转换为 int 类型
#define MAXLASTD    cast_int(LUA_MAXINTEGER % 10)

# 定义一个静态函数，将字符串转换为整数
static const char *l_str2int (const char *s, lua_Integer *result) {
  lua_Unsigned a = 0;  # 初始化一个无符号整数 a
  int empty = 1;  # 初始化一个标志位，表示字符串是否为空
  int neg;  # 初始化一个标志位，表示是否为负数
  while (lisspace(cast_uchar(*s))) s++;  # 跳过字符串开头的空格
  neg = isneg(&s);  # 判断是否为负数
  if (s[0] == '0' && (s[1] == 'x' || s[1] == 'X')) {  # 判断是否为十六进制
    s += 2;  # 跳过 '0x'
    for (; lisxdigit(cast_uchar(*s)); s++) {  # 遍历十六进制数字
      a = a * 16 + luaO_hexavalue(*s);  # 将字符转换为十六进制数并累加到 a 中
      empty = 0;  # 标志位置为非空
    }
  }
  else {  # 如果不是十六进制，则为十进制
    for (; lisdigit(cast_uchar(*s)); s++) {  # 遍历十进制数字
      int d = *s - '0';  # 将字符转换为数字
      if (a >= MAXBY10 && (a > MAXBY10 || d > MAXLASTD + neg))  # 判断是否溢出
        return NULL;  # 如果溢出则返回空指针
      a = a * 10 + d;  # 将数字累加到 a 中
      empty = 0;  # 标志位置为非空
    }
  }
  while (lisspace(cast_uchar(*s))) s++;  # 跳过字符串末尾的空格
  if (empty || *s != '\0') return NULL;  # 如果字符串为空或者不是以空字符结尾，则返回空指针
  else {
    *result = l_castU2S((neg) ? 0u - a : a);  # 将结果赋值给 result
    return s;  # 返回字符串指针
  }
}

# 定义一个函数，将字符串转换为数字
size_t luaO_str2num (const char *s, TValue *o) {
  lua_Integer i; lua_Number n;  # 定义一个整数和一个浮点数
  const char *e;  # 定义一个字符串指针
  if ((e = l_str2int(s, &i)) != NULL) {  # 尝试将字符串转换为整数
    setivalue(o, i);  # 将整数赋值给 o
  }
  else if ((e = l_str2d(s, &n)) != NULL) {  # 否则尝试将字符串转换为浮点数
    setfltvalue(o, n);  # 将浮点数赋值给 o
  }
  else
    return 0;  # 转换失败，返回 0
  return (e - s) + 1;  # 转换成功，返回字符串大小
}

# 定义一个函数，将 Unicode 编码转换为 UTF-8 编码
int luaO_utf8esc (char *buff, unsigned long x) {
  int n = 1;  # 初始化一个变量，表示放入缓冲区的字节数（倒序）
  lua_assert(x <= 0x7FFFFFFFu);  # 断言 x 的值不大于 0x7FFFFFFF
  if (x < 0x80)  # 如果是 ASCII 码
    buff[UTF8BUFFSZ - 1] = cast_char(x);  # 将 x 转换为字符并放入缓冲区
  else {  # 否则需要继续字节
    unsigned int mfb = 0x3f;  # 最大适合放入第一个字节的值
    do {  # 添加继续字节
      buff[UTF8BUFFSZ - (n++)] = cast_char(0x80 | (x & 0x3f));  # 将 x 的低 6 位加上 0x80 放入缓冲区
      x >>= 6;  # 移除已添加的位
      mfb >>= 1;  # 第一个字节中的可用位减一
    } while (x > mfb);  /* 如果 x 大于 mfb，说明仍然需要继续读取后续字节 */
    buff[UTF8BUFFSZ - n] = cast_char((~mfb << 1) | x);  /* 将第一个字节添加到缓冲区 */
  }
  return n;  /* 返回读取的字节数 */
/*
** Maximum length of the conversion of a number to a string. Must be
** enough to accommodate both LUA_INTEGER_FMT and LUA_NUMBER_FMT.
** (For a long long int, this is 19 digits plus a sign and a final '\0',
** adding to 21. For a long double, it can go to a sign, 33 digits,
** the dot, an exponent letter, an exponent sign, 5 exponent digits,
** and a final '\0', adding to 43.)
*/
#define MAXNUMBER2STR    44


/*
** Convert a number object to a string, adding it to a buffer
*/
static int tostringbuff (TValue *obj, char *buff) {
  int len;
  lua_assert(ttisnumber(obj));  // 检查对象是否为数字类型
  if (ttisinteger(obj))  // 如果是整数类型
    len = lua_integer2str(buff, MAXNUMBER2STR, ivalue(obj));  // 将整数转换为字符串
  else {
    len = lua_number2str(buff, MAXNUMBER2STR, fltvalue(obj));  // 将浮点数转换为字符串
    if (buff[strspn(buff, "-0123456789")] == '\0') {  /* looks like an int? */  // 如果看起来像是整数
      buff[len++] = lua_getlocaledecpoint();  // 添加小数点
      buff[len++] = '0';  /* adds '.0' to result */  // 添加 '.0' 到结果
    }
  }
  return len;  // 返回字符串长度
}


/*
** Convert a number object to a Lua string, replacing the value at 'obj'
*/
void luaO_tostring (lua_State *L, TValue *obj) {
  char buff[MAXNUMBER2STR];  // 创建一个缓冲区
  int len = tostringbuff(obj, buff);  // 调用tostringbuff函数将对象转换为字符串
  setsvalue(L, obj, luaS_newlstr(L, buff, len));  // 将转换后的字符串设置为对象的值
}




/*
** {==================================================================
** 'luaO_pushvfstring'
** ===================================================================
*/

/* size for buffer space used by 'luaO_pushvfstring' */
#define BUFVFS        200

/* buffer used by 'luaO_pushvfstring' */
typedef struct BuffFS {
  lua_State *L;
  int pushed;  /* number of string pieces already on the stack */  // 已经在堆栈上的字符串片段数量
  int blen;  /* length of partial string in 'space' */  // 'space'中部分字符串的长度
  char space[BUFVFS];  /* holds last part of the result */  // 保存结果的最后部分
} BuffFS;


/*
** Push given string to the stack, as part of the buffer, and
** join the partial strings in the stack into one.
*/
# 将字符串推入 Lua 栈中
static void pushstr (BuffFS *buff, const char *str, size_t l) {
  lua_State *L = buff->L;  # 获取 Lua 状态机
  setsvalue2s(L, L->top, luaS_newlstr(L, str, l));  # 将字符串转换为 Lua 字符串对象，并推入栈顶
  L->top++;  # 栈顶指针向上移动一个位置，可能会使用一个额外的槽位
  buff->pushed++;  # 记录推入栈中的字符串数量
  luaV_concat(L, buff->pushed);  # 将部分结果连接成一个
  buff->pushed = 1;  # 重置推入栈中的字符串数量
}

'''
** 清空缓冲区空间到栈中
'''
static void clearbuff (BuffFS *buff) {
  pushstr(buff, buff->space, buff->blen);  # 将缓冲区内容推入栈中
  buff->blen = 0;  # 现在空间是空的
}

'''
** 在缓冲区中获取大小为 'sz' 的空间。如果缓冲区空间不足，清空它。'sz' 必须适合一个空缓冲区。
'''
static char *getbuff (BuffFS *buff, int sz) {
  lua_assert(buff->blen <= BUFVFS); lua_assert(sz <= BUFVFS);  # 断言缓冲区大小和 sz 的合法性
  if (sz > BUFVFS - buff->blen)  # 空间不足？
    clearbuff(buff)  # 清空缓冲区
  return buff->space + buff->blen;  # 返回缓冲区中的空间
}

#define addsize(b,sz)    ((b)->blen += (sz))  # 增加缓冲区大小

'''
** 将 'str' 添加到缓冲区中。如果字符串大于缓冲区空间，直接推送字符串到栈中。
'''
static void addstr2buff (BuffFS *buff, const char *str, size_t slen) {
  if (slen <= BUFVFS) {  # 字符串是否适合缓冲区？
    char *bf = getbuff(buff, cast_int(slen));  # 获取缓冲区空间
    memcpy(bf, str, slen);  # 将字符串添加到缓冲区
    addsize(buff, cast_int(slen));  # 增加缓冲区大小
  }
  else {  # 字符串大于缓冲区
    clearbuff(buff);  # 字符串在缓冲区内容之后
    pushstr(buff, str, slen);  # 推送字符串
  }
}

'''
** 将数字添加到缓冲区中
'''
static void addnum2buff (BuffFS *buff, TValue *num) {
  char *numbuff = getbuff(buff, MAXNUMBER2STR);  # 获取缓冲区空间
  int len = tostringbuff(num, numbuff);  # 将数字格式化为 'numbuff'
  addsize(buff, len);  # 增加缓冲区大小
}

'''
** 此函数仅处理 '%d', '%c', '%f', '%p', '%s', 和 '%%' 传统格式，以及 Lua 特定的 '%I' 和 '%U'
'''
const char *luaO_pushvfstring (lua_State *L, const char *fmt, va_list argp) {
  BuffFS buff;  /* holds last part of the result */  // 定义一个缓冲区结构体变量，用于保存结果的最后部分
  const char *e;  /* points to next '%' */  // 指向下一个 '%' 的指针
  buff.pushed = buff.blen = 0;  // 初始化缓冲区的推送和长度为0
  buff.L = L;  // 将 Lua 状态指针赋值给缓冲区结构体
  while ((e = strchr(fmt, '%')) != NULL) {  // 在格式字符串中查找 '%'，直到没有找到为止
    addstr2buff(&buff, fmt, e - fmt);  /* add 'fmt' up to '%' */  // 将格式字符串中 '%' 之前的部分添加到缓冲区中
    switch (*(e + 1)) {  /* conversion specifier */  // 根据 '%' 后面的字符进行不同的处理
      case 's': {  /* zero-terminated string */  // 处理字符串类型
        const char *s = va_arg(argp, char *);  // 从可变参数列表中获取字符串
        if (s == NULL) s = "(null)";  // 如果字符串为空，则赋值为 "(null)"
        addstr2buff(&buff, s, strlen(s));  // 将字符串添加到缓冲区中
        break;
      }
      case 'c': {  /* an 'int' as a character */  // 处理字符类型
        char c = cast_uchar(va_arg(argp, int));  // 将整数转换为字符
        addstr2buff(&buff, &c, sizeof(char));  // 将字符添加到缓冲区中
        break;
      }
      case 'd': {  /* an 'int' */  // 处理整数类型
        TValue num;  // 定义一个值对象
        setivalue(&num, va_arg(argp, int));  // 将整数转换为值对象
        addnum2buff(&buff, &num);  // 将值对象添加到缓冲区中
        break;
      }
      case 'I': {  /* a 'lua_Integer' */  // 处理 lua_Integer 类型
        TValue num;  // 定义一个值对象
        setivalue(&num, cast(lua_Integer, va_arg(argp, l_uacInt)));  // 将整数转换为值对象
        addnum2buff(&buff, &num);  // 将值对象添加到缓冲区中
        break;
      }
      case 'f': {  /* a 'lua_Number' */  // 处理 lua_Number 类型
        TValue num;  // 定义一个值对象
        setfltvalue(&num, cast_num(va_arg(argp, l_uacNumber)));  // 将数字转换为值对象
        addnum2buff(&buff, &num);  // 将值对象添加到缓冲区中
        break;
      }
      case 'p': {  /* a pointer */  // 处理指针类型
        const int sz = 3 * sizeof(void*) + 8; /* enough space for '%p' */  // 计算足够存放 '%p' 的空间大小
        char *bf = getbuff(&buff, sz);  // 获取缓冲区中的空间
        void *p = va_arg(argp, void *);  // 从可变参数列表中获取指针
        int len = lua_pointer2str(bf, sz, p);  // 将指针转换为字符串
        addsize(&buff, len);  // 将转换后的字符串长度添加到缓冲区中
        break;
      }
      case 'U': {  /* a 'long' as a UTF-8 sequence */  // 处理长整型作为 UTF-8 序列
        char bf[UTF8BUFFSZ];  // 定义一个缓冲区
        int len = luaO_utf8esc(bf, va_arg(argp, long));  // 将长整型转换为 UTF-8 编码的字符串
        addstr2buff(&buff, bf + UTF8BUFFSZ - len, len);  // 将转换后的字符串添加到缓冲区中
        break;
      }
      case '%': {  // 处理百分号
        addstr2buff(&buff, "%", 1);  // 将百分号添加到缓冲区中
        break;
      }
      default: {  // 处理无效选项
        luaG_runerror(L, "invalid option '%%%c' to 'lua_pushfstring'", *(e + 1));  // 抛出运行时错误
      }
    }
    fmt = e + 2;  /* 将 e 的值加 2 赋给 fmt，跳过 '%' 和格式说明符 */
  }
  addstr2buff(&buff, fmt, strlen(fmt));  /* 将 fmt 后面的内容添加到缓冲区中 */
  clearbuff(&buff);  /* 清空缓冲区，将内容放回栈中 */
  lua_assert(buff.pushed == 1);  /* 断言缓冲区中的内容已经被放回栈中 */
  return svalue(s2v(L->top - 1));  /* 返回栈顶元素的值 */
/* luaO_pushfstring 函数，用于将格式化的字符串推入 Lua 栈中 */
const char *luaO_pushfstring (lua_State *L, const char *fmt, ...) {
  const char *msg;  // 用于存储格式化后的字符串
  va_list argp;  // 定义参数列表
  va_start(argp, fmt);  // 初始化参数列表
  msg = luaO_pushvfstring(L, fmt, argp);  // 调用 luaO_pushvfstring 函数进行格式化
  va_end(argp);  // 结束参数列表
  return msg;  // 返回格式化后的字符串
}

/* luaO_chunkid 函数，用于将源码标识符转换为特定格式的字符串 */
void luaO_chunkid (char *out, const char *source, size_t srclen) {
  size_t bufflen = LUA_IDSIZE;  // 缓冲区的剩余空间
  if (*source == '=') {  // 如果是 'literal' 源码
    if (srclen <= bufflen)  // 如果源码长度小于等于缓冲区剩余空间
      memcpy(out, source + 1, srclen * sizeof(char));  // 将源码拷贝到输出缓冲区
    else {  // 如果源码长度大于缓冲区剩余空间
      addstr(out, source + 1, bufflen - 1);  // 将部分源码拷贝到输出缓冲区
      *out = '\0';  // 添加字符串结束符
    }
  }
  else if (*source == '@') {  // 如果是文件名
    if (srclen <= bufflen)  // 如果文件名长度小于等于缓冲区剩余空间
      memcpy(out, source + 1, srclen * sizeof(char));  // 将文件名拷贝到输出缓冲区
    else {  // 如果文件名长度大于缓冲区剩余空间
      addstr(out, RETS, LL(RETS));  // 添加省略号到输出缓冲区
      bufflen -= LL(RETS);  // 更新剩余空间
      memcpy(out, source + 1 + srclen - bufflen, bufflen * sizeof(char));  // 将部分文件名拷贝到输出缓冲区
    }
  }
  else {  // 如果是字符串
    const char *nl = strchr(source, '\n');  // 查找第一个换行符的位置
    addstr(out, PRE, LL(PRE));  // 添加前缀到输出缓冲区
    bufflen -= LL(PRE RETS POS) + 1;  // 更新剩余空间
    if (srclen < bufflen && nl == NULL) {  // 如果源码长度小于剩余空间并且没有换行符
      addstr(out, source, srclen);  // 将源码拷贝到输出缓冲区
    }
    else {
      if (nl != NULL) srclen = nl - source;  // 如果有换行符，截取到第一个换行符位置
      if (srclen > bufflen) srclen = bufflen;  // 如果源码长度大于剩余空间，截取到剩余空间长度
      addstr(out, source, srclen);  // 将部分源码拷贝到输出缓冲区
      addstr(out, RETS, LL(RETS));  // 添加省略号到输出缓冲区
    }
    memcpy(out, POS, (LL(POS) + 1) * sizeof(char));  // 添加后缀到输出缓冲区
  }
}
```