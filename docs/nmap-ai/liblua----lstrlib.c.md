# `nmap\liblua\lstrlib.c`

```cpp
/*
** $Id: lstrlib.c $
** 字符串操作和模式匹配的标准库
** 请参阅 lua.h 中的版权声明
*/

#define lstrlib_c
#define LUA_LIB

#include "lprefix.h"


#include <ctype.h>
#include <float.h>
#include <limits.h>
#include <locale.h>
#include <math.h>
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"


/*
** 模式匹配期间模式可以执行的最大捕获次数
** 此限制是任意的，但必须适合无符号字符
*/
#if !defined(LUA_MAXCAPTURES)
#define LUA_MAXCAPTURES        32
#endif


/* 宏用于将字符转换为无符号字符 */
#define uchar(c)    ((unsigned char)(c))


/*
** 一些大小最好限制在 'int' 中，但也必须适合 'size_t'。
** （我们假设 'lua_Integer' 不能比 'int' 更小。）
*/
#define MAX_SIZET    ((size_t)(~(size_t)0))

#define MAXSIZE  \
    (sizeof(size_t) < sizeof(int) ? MAX_SIZET : (size_t)(INT_MAX))




static int str_len (lua_State *L) {
  size_t l;
  luaL_checklstring(L, 1, &l);
  lua_pushinteger(L, (lua_Integer)l);
  return 1;
}


/*
** 转换相对初始字符串位置
** （负数表示从末尾开始）：将结果裁剪为 [1, 无穷大]。
** Lua 中任何字符串的长度都必须适合 lua_Integer，
** 因此在转换中不会发生溢出。
** 反转的比较避免了可能的溢出计算 '-pos'。
*/
static size_t posrelatI (lua_Integer pos, size_t len) {
  if (pos > 0)
    return (size_t)pos;
  else if (pos == 0)
    return 1;
  else if (pos < -(lua_Integer)len)  /* 反转的比较 */
    return 1;  /* 裁剪为 1 */
  else return len + (size_t)pos + 1;
}


/*
** 从参数 'arg' 获取可选的结束字符串位置，
** 默认值为 'def'。
** 负数表示从末尾开始：将结果裁剪为 [0, len]
*/
// 获取指定位置的结束位置
static size_t getendpos (lua_State *L, int arg, lua_Integer def,
                         size_t len) {
  // 获取参数中的整数值，如果没有则使用默认值
  lua_Integer pos = luaL_optinteger(L, arg, def);
  // 如果位置大于字符串长度，则返回字符串长度
  if (pos > (lua_Integer)len)
    return len;
  // 如果位置在字符串长度范围内，则返回该位置
  else if (pos >= 0)
    return (size_t)pos;
  // 如果位置为负数且绝对值大于字符串长度，则返回0
  else if (pos < -(lua_Integer)len)
    return 0;
  // 其他情况返回计算后的位置
  else return len + (size_t)pos + 1;
}


// 截取字符串
static int str_sub (lua_State *L) {
  size_t l;
  const char *s = luaL_checklstring(L, 1, &l);
  size_t start = posrelatI(luaL_checkinteger(L, 2), l);
  size_t end = getendpos(L, 3, -1, l);
  // 如果起始位置小于等于结束位置，则将截取的子串压入栈中
  if (start <= end)
    lua_pushlstring(L, s + start - 1, (end - start) + 1);
  // 否则压入空字符串
  else lua_pushliteral(L, "");
  return 1;
}


// 反转字符串
static int str_reverse (lua_State *L) {
  size_t l, i;
  luaL_Buffer b;
  const char *s = luaL_checklstring(L, 1, &l);
  char *p = luaL_buffinitsize(L, &b, l);
  // 循环将字符串反转后存入缓冲区
  for (i = 0; i < l; i++)
    p[i] = s[l - i - 1];
  // 将反转后的字符串压入栈中
  luaL_pushresultsize(&b, l);
  return 1;
}


// 将字符串转换为小写
static int str_lower (lua_State *L) {
  size_t l;
  size_t i;
  luaL_Buffer b;
  const char *s = luaL_checklstring(L, 1, &l);
  char *p = luaL_buffinitsize(L, &b, l);
  // 循环将字符串中的字符转换为小写后存入缓冲区
  for (i=0; i<l; i++)
    p[i] = tolower(uchar(s[i]));
  // 将转换为小写后的字符串压入栈中
  luaL_pushresultsize(&b, l);
  return 1;
}


// 将字符串转换为大写
static int str_upper (lua_State *L) {
  size_t l;
  size_t i;
  luaL_Buffer b;
  const char *s = luaL_checklstring(L, 1, &l);
  char *p = luaL_buffinitsize(L, &b, l);
  // 循环将字符串中的字符转换为大写后存入缓冲区
  for (i=0; i<l; i++)
    p[i] = toupper(uchar(s[i]));
  // 将转换为大写后的字符串压入栈中
  luaL_pushresultsize(&b, l);
  return 1;
}


// 重复字符串
static int str_rep (lua_State *L) {
  size_t l, lsep;
  const char *s = luaL_checklstring(L, 1, &l);
  lua_Integer n = luaL_checkinteger(L, 2);
  const char *sep = luaL_optlstring(L, 3, "", &lsep);
  // 如果重复次数小于等于0，则压入空字符串
  if (n <= 0)
    lua_pushliteral(L, "");
  // 如果重复后的字符串长度超过最大限制，则报错
  else if (l_unlikely(l + lsep < l || l + lsep > MAXSIZE / n))
    return luaL_error(L, "resulting string too large");
  // 否则进行字符串重复操作
  else {
    size_t totallen = (size_t)n * l + (size_t)(n - 1) * lsep;
    luaL_Buffer b;
    char *p = luaL_buffinitsize(L, &b, totallen);
    while (n-- > 1) {  /* 当 n 大于 1 时执行循环，执行 n-1 次拷贝操作（后面跟着分隔符） */
      memcpy(p, s, l * sizeof(char)); p += l;  // 将 s 指向的内容拷贝到 p 指向的位置，拷贝长度为 l 个字符，然后移动 p 指针
      if (lsep > 0) {  /* 如果分隔符长度大于 0 */
        memcpy(p, sep, lsep * sizeof(char));  // 将 sep 指向的内容拷贝到 p 指向的位置，拷贝长度为 lsep 个字符
        p += lsep;  // 移动 p 指针
      }
    }
    memcpy(p, s, l * sizeof(char));  /* 最后一次拷贝（后面没有分隔符） */
    luaL_pushresultsize(&b, totallen);  // 将 b 中的内容推入 Lua 栈，推入的内容长度为 totallen
  }
  return 1;  // 返回结果为 1
static int str_byte (lua_State *L) {
  size_t l;  // 用于存储字符串长度
  const char *s = luaL_checklstring(L, 1, &l);  // 获取函数参数中的字符串
  lua_Integer pi = luaL_optinteger(L, 2, 1);  // 获取函数参数中的整数，如果参数不存在则默认为1
  size_t posi = posrelatI(pi, l);  // 获取字符串的起始位置
  size_t pose = getendpos(L, 3, pi, l);  // 获取字符串的结束位置
  int n, i;  // 用于循环计数
  if (posi > pose) return 0;  // 如果起始位置大于结束位置，则返回0
  if (l_unlikely(pose - posi >= (size_t)INT_MAX))  // 如果字符串切片长度超过INT_MAX，则报错
    return luaL_error(L, "string slice too long");
  n = (int)(pose -  posi) + 1;  // 计算切片长度
  luaL_checkstack(L, n, "string slice too long");  // 检查栈空间是否足够
  for (i=0; i<n; i++)  // 遍历切片
    lua_pushinteger(L, uchar(s[posi+i-1]));  // 将切片中的字符转换为整数并压入栈中
  return n;  // 返回切片长度
}

static int str_char (lua_State *L) {
  int n = lua_gettop(L);  // 获取参数个数
  int i;  // 用于循环计数
  luaL_Buffer b;  // 创建缓冲区
  char *p = luaL_buffinitsize(L, &b, n);  // 初始化缓冲区
  for (i=1; i<=n; i++) {  // 遍历参数
    lua_Unsigned c = (lua_Unsigned)luaL_checkinteger(L, i);  // 获取参数中的整数
    luaL_argcheck(L, c <= (lua_Unsigned)UCHAR_MAX, i, "value out of range");  // 检查整数是否在合法范围内
    p[i - 1] = uchar(c);  // 将整数转换为字符存入缓冲区
  }
  luaL_pushresultsize(&b, n);  // 将缓冲区中的内容推入栈中
  return 1;  // 返回1
}

/*
** Buffer to store the result of 'string.dump'. It must be initialized
** after the call to 'lua_dump', to ensure that the function is on the
** top of the stack when 'lua_dump' is called. ('luaL_buffinit' might
** push stuff.)
*/
struct str_Writer {
  int init;  // 标记缓冲区是否已经初始化
  luaL_Buffer B;  // 缓冲区
};

static int writer (lua_State *L, const void *b, size_t size, void *ud) {
  struct str_Writer *state = (struct str_Writer *)ud;  // 获取缓冲区状态
  if (!state->init) {  // 如果缓冲区未初始化
    state->init = 1;  // 标记缓冲区已初始化
    luaL_buffinit(L, &state->B);  // 初始化缓冲区
  }
  luaL_addlstring(&state->B, (const char *)b, size);  // 将字符串添加到缓冲区
  return 0;  // 返回0
}

static int str_dump (lua_State *L) {
  struct str_Writer state;  // 创建缓冲区状态
  int strip = lua_toboolean(L, 2);  // 获取布尔值参数
  luaL_checktype(L, 1, LUA_TFUNCTION);  // 检查第一个参数是否为函数
  lua_settop(L, 1);  // 确保函数在栈顶
  state.init = 0;  // 初始化缓冲区状态
  if (l_unlikely(lua_dump(L, writer, &state, strip) != 0))  // 如果lua_dump函数执行失败
    return luaL_error(L, "unable to dump given function");  // 报错
  luaL_pushresult(&state.B);  // 将缓冲区中的内容推入栈中
  return 1;  // 返回1
}
/* {======================================================
** METAMETHODS
** =======================================================
*/

#if defined(LUA_NOCVTS2N)    /* { */

/* 如果定义了 LUA_NOCVTS2N，则不进行从字符串到数字的强制转换 */

static const luaL_Reg stringmetamethods[] = {
  {"__index", NULL},  /* 占位符 */
  {NULL, NULL}
};

#else        /* }{ */

/* 如果没有定义 LUA_NOCVTS2N，则进行从字符串到数字的强制转换 */

static int tonum (lua_State *L, int arg) {
  if (lua_type(L, arg) == LUA_TNUMBER) {  /* 已经是数字？ */
    lua_pushvalue(L, arg);
    return 1;
  }
  else {  /* 检查是否是一个数字字符串 */
    size_t len;
    const char *s = lua_tolstring(L, arg, &len);
    return (s != NULL && lua_stringtonumber(L, s) == len + 1);
  }
}

/* 尝试调用元方法 */
static void trymt (lua_State *L, const char *mtname) {
  lua_settop(L, 2);  /* 回到原始参数 */
  if (l_unlikely(lua_type(L, 2) == LUA_TSTRING ||
                 !luaL_getmetafield(L, 2, mtname)))
    luaL_error(L, "attempt to %s a '%s' with a '%s'", mtname + 2,
                  luaL_typename(L, -2), luaL_typename(L, -1));
  lua_insert(L, -3);  /* 将元方法放在参数之前 */
  lua_call(L, 2, 1);  /* 调用元方法 */
}

/* 进行算术运算 */
static int arith (lua_State *L, int op, const char *mtname) {
  if (tonum(L, 1) && tonum(L, 2))
    lua_arith(L, op);  /* 结果将会在栈顶 */
  else
    trymt(L, mtname);
  return 1;
}

/* 加法 */
static int arith_add (lua_State *L) {
  return arith(L, LUA_OPADD, "__add");
}

/* 减法 */
static int arith_sub (lua_State *L) {
  return arith(L, LUA_OPSUB, "__sub");
}

/* 乘法 */
static int arith_mul (lua_State *L) {
  return arith(L, LUA_OPMUL, "__mul");
}

/* 取模 */
static int arith_mod (lua_State *L) {
  return arith(L, LUA_OPMOD, "__mod");
}

/* 求幂 */
static int arith_pow (lua_State *L) {
  return arith(L, LUA_OPPOW, "__pow");
}

/* 除法 */
static int arith_div (lua_State *L) {
  return arith(L, LUA_OPDIV, "__div");
}

/* 整数除法 */
static int arith_idiv (lua_State *L) {
  return arith(L, LUA_OPIDIV, "__idiv");
}

/* 取负 */
static int arith_unm (lua_State *L) {
  return arith(L, LUA_OPUNM, "__unm");
}
static const luaL_Reg stringmetamethods[] = {
  {"__add", arith_add},  // 定义字符串元方法 __add，对应的函数为 arith_add
  {"__sub", arith_sub},  // 定义字符串元方法 __sub，对应的函数为 arith_sub
  {"__mul", arith_mul},  // 定义字符串元方法 __mul，对应的函数为 arith_mul
  {"__mod", arith_mod},  // 定义字符串元方法 __mod，对应的函数为 arith_mod
  {"__pow", arith_pow},  // 定义字符串元方法 __pow，对应的函数为 arith_pow
  {"__div", arith_div},  // 定义字符串元方法 __div，对应的函数为 arith_div
  {"__idiv", arith_idiv},  // 定义字符串元方法 __idiv，对应的函数为 arith_idiv
  {"__unm", arith_unm},  // 定义字符串元方法 __unm，对应的函数为 arith_unm
  {"__index", NULL},  /* placeholder */  // 定义字符串元方法 __index，占位符为 NULL
  {NULL, NULL}  // 结束标记
};

#endif        /* } */

/* }====================================================== */

/*
** {======================================================
** PATTERN MATCHING
** =======================================================
*/

#define CAP_UNFINISHED    (-1)  // 定义未完成的捕获标志
#define CAP_POSITION    (-2)  // 定义位置捕获标志

typedef struct MatchState {
  const char *src_init;  /* init of source string */  // 源字符串的起始位置
  const char *src_end;  /* end ('\0') of source string */  // 源字符串的结束位置
  const char *p_end;  /* end ('\0') of pattern */  // 模式的结束位置
  lua_State *L;  // Lua 状态
  int matchdepth;  /* control for recursive depth (to avoid C stack overflow) */  // 递归深度控制
  unsigned char level;  /* total number of captures (finished or unfinished) */  // 捕获的总数（已完成或未完成）
  struct {
    const char *init;
    ptrdiff_t len;
  } capture[LUA_MAXCAPTURES];  // 捕获的初始化位置和长度
} MatchState;

/* recursive function */
static const char *match (MatchState *ms, const char *s, const char *p);  // 递归函数 match

/* maximum recursion depth for 'match' */
#if !defined(MAXCCALLS)
#define MAXCCALLS    200  // 'match' 的最大递归深度
#endif

#define L_ESC        '%'  // 转义字符
#define SPECIALS    "^$*+?.([%-"  // 特殊字符集合

static int check_capture (MatchState *ms, int l) {  // 检查捕获
  l -= '1';
  if (l_unlikely(l < 0 || l >= ms->level ||
                 ms->capture[l].len == CAP_UNFINISHED))
    return luaL_error(ms->L, "invalid capture index %%%d", l + 1);  // 报错：无效的捕获索引
  return l;
}

static int capture_to_close (MatchState *ms) {  // 获取待关闭的捕获
  int level = ms->level;
  for (level--; level>=0; level--)
    if (ms->capture[level].len == CAP_UNFINISHED) return level;
  return luaL_error(ms->L, "invalid pattern capture");  // 报错：无效的模式捕获
}

static const char *classend (MatchState *ms, const char *p) {  // 获取字符类的结束位置
  switch (*p++) {
    # 处理转义字符
    case L_ESC: {
      # 如果指针指向结尾，则抛出异常
      if (l_unlikely(p == ms->p_end))
        luaL_error(ms->L, "malformed pattern (ends with '%%')");
      # 返回指针后移一位
      return p+1;
    }
    # 处理字符集
    case '[': {
      # 如果下一个字符是'^'，则指针后移一位
      if (*p == '^') p++;
      # 寻找']'字符
      do {  /* look for a ']' */
        # 如果指针指向结尾，则抛出异常
        if (l_unlikely(p == ms->p_end))
          luaL_error(ms->L, "malformed pattern (missing ']')");
        # 如果当前字符是转义字符并且下一个字符不是结尾，则指针后移一位（例如'%]'）
        if (*(p++) == L_ESC && p < ms->p_end)
          p++;  /* skip escapes (e.g. '%]') */
      } while (*p != ']');
      # 返回指针后移一位
      return p+1;
    }
    # 默认情况
    default: {
      # 返回指针
      return p;
    }
  }
# 匹配指定字符和字符类的函数
static int match_class (int c, int cl) {
  int res;
  # 将字符类转换为小写字母
  switch (tolower(cl)) {
    # 如果字符类为 'a'，则判断字符是否为字母
    case 'a' : res = isalpha(c); break;
    # 如果字符类为 'c'，则判断字符是否为控制字符
    case 'c' : res = iscntrl(c); break;
    # 如果字符类为 'd'，则判断字符是否为数字
    case 'd' : res = isdigit(c); break;
    # 如果字符类为 'g'，则判断字符是否为图形字符
    case 'g' : res = isgraph(c); break;
    # 如果字符类为 'l'，则判断字符是否为小写字母
    case 'l' : res = islower(c); break;
    # 如果字符类为 'p'，则判断字符是否为标点字符
    case 'p' : res = ispunct(c); break;
    # 如果字符类为 's'，则判断字符是否为空白字符
    case 's' : res = isspace(c); break;
    # 如果字符类为 'u'，则判断字符是否为大写字母
    case 'u' : res = isupper(c); break;
    # 如果字符类为 'w'，则判断字符是否为字母或数字
    case 'w' : res = isalnum(c); break;
    # 如果字符类为 'x'，则判断字符是否为十六进制数字
    case 'x' : res = isxdigit(c); break;
    # 如果字符类为 'z'，则判断字符是否为零（已废弃选项）
    case 'z' : res = (c == 0); break;  /* deprecated option */
    # 默认情况下，判断字符是否与字符类相等
    default: return (cl == c);
  }
  # 如果字符类为小写字母，则返回 res，否则返回非 res
  return (islower(cl) ? res : !res);
}

# 匹配括号字符类的函数
static int matchbracketclass (int c, const char *p, const char *ec) {
  int sig = 1;
  # 如果下一个字符是 '^'，则将 sig 置为 0，并跳过 '^'
  if (*(p+1) == '^') {
    sig = 0;
    p++;  /* skip the '^' */
  }
  # 遍历括号字符类
  while (++p < ec) {
    # 如果遇到转义字符
    if (*p == L_ESC) {
      p++;
      # 判断字符是否匹配字符类
      if (match_class(c, uchar(*p)))
        return sig;
    }
    # 如果遇到范围表示符 '-'，并且后面还有字符
    else if ((*(p+1) == '-') && (p+2 < ec)) {
      p+=2;
      # 判断字符是否在范围内
      if (uchar(*(p-2)) <= c && c <= uchar(*p))
        return sig;
    }
    # 如果字符与括号字符类相等，则返回 sig
    else if (uchar(*p) == c) return sig;
  }
  # 如果字符不在括号字符类中，则返回非 sig
  return !sig;
}

# 匹配单个字符的函数
static int singlematch (MatchState *ms, const char *s, const char *p,
                        const char *ep) {
  # 如果字符串已经结束，则返回 0
  if (s >= ms->src_end)
    return 0;
  else {
    int c = uchar(*s);
    # 根据模式字符进行匹配
    switch (*p) {
      # 如果模式字符为 '.'，则匹配任意字符
      case '.': return 1;  /* matches any char */
      # 如果模式字符为转义字符，根据后面的字符类进行匹配
      case L_ESC: return match_class(c, uchar(*(p+1)));
      # 如果模式字符为 '['，则匹配括号字符类
      case '[': return matchbracketclass(c, p, ep-1);
      # 默认情况下，判断字符是否与模式字符相等
      default:  return (uchar(*p) == c);
    }
  }
}

# 匹配平衡括号的函数
static const char *matchbalance (MatchState *ms, const char *s,
                                   const char *p) {
  # 如果模式字符已经超出范围，则抛出错误
  if (l_unlikely(p >= ms->p_end - 1))
    luaL_error(ms->L, "malformed pattern (missing arguments to '%%b')");
  # 如果字符串与模式字符相等，则返回空指针
  if (*s != *p) return NULL;
  else {
    int b = *p;
    int e = *(p+1);
    int cont = 1;
    # 遍历字符串
    while (++s < ms->src_end) {
      # 如果遇到结束括号
      if (*s == e) {
        # 如果括号匹配完成，则返回结束括号的下一个位置
        if (--cont == 0) return s+1;
      }
      # 如果遇到起始括号
      else if (*s == b) cont++;
    }
  }
  # 如果括号不匹配，则返回空指针
  return NULL;
}
    }  // 结束内部循环
  }  // 结束外部循环
  return NULL;  // 字符串不平衡，返回空指针
# 返回最大匹配的字符串
static const char *max_expand (MatchState *ms, const char *s,
                                 const char *p, const char *ep) {
  ptrdiff_t i = 0;  /* 记录项目的最大扩展次数 */
  while (singlematch(ms, s + i, p, ep))
    i++;
  /* 不断尝试使用最大重复次数进行匹配 */
  while (i>=0) {
    const char *res = match(ms, (s+i), ep+1);
    if (res) return res;
    i--;  /* 否则没有匹配; 减少1次重复以再次尝试 */
  }
  return NULL;
}


# 返回最小匹配的字符串
static const char *min_expand (MatchState *ms, const char *s,
                                 const char *p, const char *ep) {
  for (;;) {
    const char *res = match(ms, s, ep+1);
    if (res != NULL)
      return res;
    else if (singlematch(ms, s, p, ep))
      s++;  /* 尝试增加一次重复 */
    else return NULL;
  }
}


# 开始捕获匹配的字符串
static const char *start_capture (MatchState *ms, const char *s,
                                    const char *p, int what) {
  const char *res;
  int level = ms->level;
  if (level >= LUA_MAXCAPTURES) luaL_error(ms->L, "too many captures");
  ms->capture[level].init = s;
  ms->capture[level].len = what;
  ms->level = level+1;
  if ((res=match(ms, s, p)) == NULL)  /* 匹配失败? */
    ms->level--;  /* 撤销捕获 */
  return res;
}


# 结束捕获匹配的字符串
static const char *end_capture (MatchState *ms, const char *s,
                                  const char *p) {
  int l = capture_to_close(ms);
  const char *res;
  ms->capture[l].len = s - ms->capture[l].init;  /* 结束捕获 */
  if ((res = match(ms, s, p)) == NULL)  /* 匹配失败? */
    ms->capture[l].len = CAP_UNFINISHED;  /* 撤销捕获 */
  return res;
}


# 匹配捕获的字符串
static const char *match_capture (MatchState *ms, const char *s, int l) {
  size_t len;
  l = check_capture(ms, l);
  len = ms->capture[l].len;
  if ((size_t)(ms->src_end-s) >= len &&
      memcmp(ms->capture[l].init, s, len) == 0)
    return s+len;
  else return NULL;
}
static const char *match (MatchState *ms, const char *s, const char *p) {
  // 如果匹配深度减少到0，抛出异常
  if (l_unlikely(ms->matchdepth-- == 0))
    luaL_error(ms->L, "pattern too complex");
  init: /* using goto's to optimize tail recursion */
  // 如果模式未结束
  if (p != ms->p_end) {  /* end of pattern? */
    }
  }
  // 增加匹配深度
  ms->matchdepth++;
  // 返回字符串
  return s;
}



static const char *lmemfind (const char *s1, size_t l1,
                               const char *s2, size_t l2) {
  // 如果s2为空字符串，返回s1
  if (l2 == 0) return s1;  /* empty strings are everywhere */
  // 如果s2长度大于s1，返回空
  else if (l2 > l1) return NULL;  /* avoids a negative 'l1' */
  else {
    const char *init;  /* to search for a '*s2' inside 's1' */
    // 减少l2，因为第一个字符将由'memchr'检查
    l2--;  /* 1st char will be checked by 'memchr' */
    // l1减去l2，因为's2'不能在此之后找到
    l1 = l1-l2;  /* 's2' cannot be found after that */
    // 当l1大于0且init不为空时，循环
    while (l1 > 0 && (init = (const char *)memchr(s1, *s2, l1)) != NULL) {
      init++;   /* 1st char is already checked */
      // 如果init和s2+1的内容相等，返回init-1
      if (memcmp(init, s2+1, l2) == 0)
        return init-1;
      else {  /* correct 'l1' and 's1' to try again */
        // 修正'l1'和's1'以再次尝试
        l1 -= init-s1;
        s1 = init;
      }
    }
    // 未找到，返回空
    return NULL;  /* not found */
  }
}


/*
** get information about the i-th capture. If there are no captures
** and 'i==0', return information about the whole match, which
** is the range 's'..'e'. If the capture is a string, return
** its length and put its address in '*cap'. If it is an integer
** (a position), push it on the stack and return CAP_POSITION.
*/
static size_t get_onecapture (MatchState *ms, int i, const char *s,
                              const char *e, const char **cap) {
  // 如果i大于等于level
  if (i >= ms->level) {
    // 如果i不等于0，抛出异常
    if (l_unlikely(i != 0))
      luaL_error(ms->L, "invalid capture index %%%d", i + 1);
    // 将s的地址赋给*cap，返回e-s的长度
    *cap = s;
    return e - s;
  }
  else {
    ptrdiff_t capl = ms->capture[i].len;
    *cap = ms->capture[i].init;
    // 如果capl为CAP_UNFINISHED，抛出异常
    if (l_unlikely(capl == CAP_UNFINISHED))
      luaL_error(ms->L, "unfinished capture");
    // 如果capl为CAP_POSITION，将(ms->capture[i].init - ms->src_init) + 1推入栈中，返回capl
    else if (capl == CAP_POSITION)
      lua_pushinteger(ms->L, (ms->capture[i].init - ms->src_init) + 1);
    return capl;
  }
}


/*
/* 将第 i 个捕获结果压入栈中 */
static void push_onecapture (MatchState *ms, int i, const char *s,
                                                    const char *e) {
  const char *cap;
  // 获取第 i 个捕获结果的长度和内容
  ptrdiff_t l = get_onecapture(ms, i, s, e, &cap);
  // 如果长度不是位置信息，则将捕获结果压入栈中
  if (l != CAP_POSITION)
    lua_pushlstring(ms->L, cap, l);
  /* else position was already pushed */
}

// 将捕获结果压入栈中
static int push_captures (MatchState *ms, const char *s, const char *e) {
  int i;
  // 计算需要压入栈的捕获结果数量
  int nlevels = (ms->level == 0 && s) ? 1 : ms->level;
  // 检查栈空间是否足够
  luaL_checkstack(ms->L, nlevels, "too many captures");
  // 遍历每个捕获结果，将其压入栈中
  for (i = 0; i < nlevels; i++)
    push_onecapture(ms, i, s, e);
  return nlevels;  /* 压入栈的字符串数量 */
}

/* 检查模式字符串是否不包含特殊字符 */
static int nospecials (const char *p, size_t l) {
  size_t upto = 0;
  do {
    // 如果模式字符串中包含特殊字符，则返回 0
    if (strpbrk(p + upto, SPECIALS))
      return 0;  /* 模式字符串包含特殊字符 */
    upto += strlen(p + upto) + 1;  /* 可能在 \0 后还有特殊字符 */
  } while (upto <= l);
  return 1;  /* 未发现特殊字符 */
}

// 初始化匹配状态
static void prepstate (MatchState *ms, lua_State *L,
                       const char *s, size_t ls, const char *p, size_t lp) {
  ms->L = L;
  ms->matchdepth = MAXCCALLS;
  ms->src_init = s;
  ms->src_end = s + ls;
  ms->p_end = p + lp;
}

// 重新初始化匹配状态
static void reprepstate (MatchState *ms) {
  ms->level = 0;
  lua_assert(ms->matchdepth == MAXCCALLS);
}

// 辅助函数，用于字符串查找操作
static int str_find_aux (lua_State *L, int find) {
  size_t ls, lp;
  const char *s = luaL_checklstring(L, 1, &ls);
  const char *p = luaL_checklstring(L, 2, &lp);
  // 获取起始位置
  size_t init = posrelatI(luaL_optinteger(L, 3, 1), ls) - 1;
  if (init > ls) {  /* 起始位置超出字符串末尾? */
    luaL_pushfail(L);  /* 无法找到任何内容 */
    return 1;
  }
  /* 显式请求或者模式字符串没有特殊字符? */
  if (find && (lua_toboolean(L, 4) || nospecials(p, lp))) {
    /* 进行普通搜索 */
    const char *s2 = lmemfind(s + init, ls - init, p, lp);
    // 如果 s2 存在
    if (s2) {
      // 将匹配的子串长度压入栈
      lua_pushinteger(L, (s2 - s) + 1);
      // 将匹配的子串结束位置压入栈
      lua_pushinteger(L, (s2 - s) + lp);
      // 返回 2，表示找到匹配
      return 2;
    }
  }
  else {
    // 创建 MatchState 对象
    MatchState ms;
    // 指向初始位置的指针
    const char *s1 = s + init;
    // 判断是否以 '^' 开头
    int anchor = (*p == '^');
    if (anchor) {
      // 跳过锚定字符
      p++; lp--;
    }
    // 准备 MatchState 对象
    prepstate(&ms, L, s, ls, p, lp);
    do {
      const char *res;
      // 重新准备 MatchState 对象
      reprepstate(&ms);
      // 如果匹配成功
      if ((res=match(&ms, s1, p)) != NULL) {
        // 如果需要找到所有匹配
        if (find) {
          // 压入匹配的子串起始位置
          lua_pushinteger(L, (s1 - s) + 1);
          // 压入匹配的子串结束位置
          lua_pushinteger(L, res - s);
          // 返回匹配的捕获内容数量加 2
          return push_captures(&ms, NULL, 0) + 2;
        }
        else
          // 返回匹配的捕获内容数量
          return push_captures(&ms, s1, res);
      }
    } while (s1++ < ms.src_end && !anchor);
  }
  // 压入失败标志
  luaL_pushfail(L);  /* not found */
  // 返回 1，表示未找到匹配
  return 1;
}

// 在 Lua 中查找字符串的辅助函数
static int str_find (lua_State *L) {
  return str_find_aux(L, 1);
}

// 在 Lua 中匹配字符串的辅助函数
static int str_match (lua_State *L) {
  return str_find_aux(L, 0);
}

// 'gmatch' 的状态
typedef struct GMatchState {
  const char *src;  // 当前位置
  const char *p;  // 模式
  const char *lastmatch;  // 上次匹配的结束位置
  MatchState ms;  // 匹配状态
} GMatchState;

// 'gmatch' 的辅助函数
static int gmatch_aux (lua_State *L) {
  GMatchState *gm = (GMatchState *)lua_touserdata(L, lua_upvalueindex(3));
  const char *src;
  gm->ms.L = L;
  for (src = gm->src; src <= gm->ms.src_end; src++) {
    const char *e;
    reprepstate(&gm->ms);
    if ((e = match(&gm->ms, src, gm->p)) != NULL && e != gm->lastmatch) {
      gm->src = gm->lastmatch = e;
      return push_captures(&gm->ms, src, e);
    }
  }
  return 0;  // 没有找到
}

// 'gmatch' 函数
static int gmatch (lua_State *L) {
  size_t ls, lp;
  const char *s = luaL_checklstring(L, 1, &ls);
  const char *p = luaL_checklstring(L, 2, &lp);
  size_t init = posrelatI(luaL_optinteger(L, 3, 1), ls) - 1;
  GMatchState *gm;
  lua_settop(L, 2);  // 保持字符串在闭包中，避免被回收
  gm = (GMatchState *)lua_newuserdatauv(L, sizeof(GMatchState), 0);
  if (init > ls)  // 起始位置超过字符串末尾？
    init = ls + 1;  // 避免在 's + init' 中溢出
  prepstate(&gm->ms, L, s, ls, p, lp);
  gm->src = s + init; gm->p = p; gm->lastmatch = NULL;
  lua_pushcclosure(L, gmatch_aux, 3);
  return 1;
}

// 向 MatchState 中添加字符串
static void add_s (MatchState *ms, luaL_Buffer *b, const char *s,
                                                   const char *e) {
  size_t l;
  lua_State *L = ms->L;
  const char *news = lua_tolstring(L, 3, &l);
  const char *p;
  while ((p = (char *)memchr(news, L_ESC, l)) != NULL) {
    luaL_addlstring(b, news, p - news);
    p++;  // 跳过 ESC
    if (*p == L_ESC)  // '%%'
      luaL_addchar(b, *p);
    else if (*p == '0')  // '%0'
        luaL_addlstring(b, s, e - s);
    else if (isdigit(uchar(*p))) {  /* 如果当前字符是数字，表示 '%n' */
      const char *cap;  // 定义一个指向字符常量的指针 cap
      ptrdiff_t resl = get_onecapture(ms, *p - '1', s, e, &cap);  // 调用 get_onecapture 函数获取匹配的子串
      if (resl == CAP_POSITION)  // 如果返回的结果是 CAP_POSITION
        luaL_addvalue(b);  /* 将位置添加到累积结果中 */
      else
        luaL_addlstring(b, cap, resl);  // 将匹配的子串添加到累积结果中
    }
    else
      luaL_error(L, "invalid use of '%c' in replacement string", L_ESC);  // 如果不是数字，则在 Lua 环境中抛出错误
    l -= p + 1 - news;  // 更新剩余字符串的长度
    news = p + 1;  // 更新 news 指针的位置
  }
  luaL_addlstring(b, news, l);  // 将剩余的字符串添加到累积结果中
/*
** 将替换值添加到字符串缓冲区 'b' 中。
** 如果原始字符串被更改，则返回 true。（函数调用和表索引导致 nil 或 false 不会更改主题。）
*/
static int add_value (MatchState *ms, luaL_Buffer *b, const char *s,
                                      const char *e, int tr) {
  lua_State *L = ms->L;
  switch (tr) {
    case LUA_TFUNCTION: {  /* 调用函数 */
      int n;
      lua_pushvalue(L, 3);  /* 压入函数 */
      n = push_captures(ms, s, e);  /* 将所有捕获作为参数 */
      lua_call(L, n, 1);  /* 调用函数 */
      break;
    }
    case LUA_TTABLE: {  /* 索引表 */
      push_onecapture(ms, 0, s, e);  /* 第一个捕获是索引 */
      lua_gettable(L, 3);
      break;
    }
    default: {  /* LUA_TNUMBER 或 LUA_TSTRING */
      add_s(ms, b, s, e);  /* 将值添加到缓冲区 */
      return 1;  /* 有变化 */
    }
  }
  if (!lua_toboolean(L, -1)) {  /* 是 nil 或 false 吗？ */
    lua_pop(L, 1);  /* 移除值 */
    luaL_addlstring(b, s, e - s);  /* 保持原始文本 */
    return 0;  /* 没有变化 */
  }
  else if (l_unlikely(!lua_isstring(L, -1)))
    return luaL_error(L, "invalid replacement value (a %s)",
                         luaL_typename(L, -1));
  else {
    luaL_addvalue(b);  /* 将结果添加到累加器 */
    return 1;  /* 有变化 */
  }
}
static int str_gsub (lua_State *L) {
  size_t srcl, lp;  // 定义变量 srcl 和 lp，用于存储字符串长度
  const char *src = luaL_checklstring(L, 1, &srcl);  // 获取第一个参数作为字符串，并获取其长度
  const char *p = luaL_checklstring(L, 2, &lp);  // 获取第二个参数作为字符串，并获取其长度
  const char *lastmatch = NULL;  // 定义变量 lastmatch，用于存储上一次匹配的结束位置
  int tr = lua_type(L, 3);  // 获取第三个参数的类型
  lua_Integer max_s = luaL_optinteger(L, 4, srcl + 1);  // 获取第四个参数作为整数，如果不存在则使用默认值
  int anchor = (*p == '^');  // 判断模式是否以 '^' 开头
  lua_Integer n = 0;  // 定义变量 n，用于存储替换次数
  int changed = 0;  // 定义变量 changed，用于标记是否有改变
  MatchState ms;  // 定义 MatchState 结构体变量 ms
  luaL_Buffer b;  // 定义 luaL_Buffer 结构体变量 b
  luaL_argexpected(L, tr == LUA_TNUMBER || tr == LUA_TSTRING ||
                   tr == LUA_TFUNCTION || tr == LUA_TTABLE, 3,
                      "string/function/table");  // 检查第三个参数的类型是否符合要求
  luaL_buffinit(L, &b);  // 初始化缓冲区
  if (anchor) {
    p++; lp--;  // 如果模式以 '^' 开头，则跳过 '^'
  }
  prepstate(&ms, L, src, srcl, p, lp);  // 准备匹配状态
  while (n < max_s) {
    const char *e;  // 定义变量 e，用于存储匹配结束位置
    reprepstate(&ms);  // 重新准备匹配状态
    if ((e = match(&ms, src, p)) != NULL && e != lastmatch) {  // 如果匹配成功且不是上一次的匹配结果
      n++;  // 替换次数加一
      changed = add_value(&ms, &b, src, e, tr) | changed;  // 添加匹配结果到缓冲区，并更新 changed 标记
      src = lastmatch = e;  // 更新源字符串和上一次匹配结束位置
    }
    else if (src < ms.src_end)  // 否则，跳过一个字符
      luaL_addchar(&b, *src++);
    else break;  // 源字符串结束
    if (anchor) break;  // 如果模式以 '^' 开头，则跳出循环
  }
  if (!changed)  // 如果没有改变
    lua_pushvalue(L, 1);  // 返回原始字符串
  else {  // 否则，有改变
    luaL_addlstring(&b, src, ms.src_end-src);  // 添加剩余的源字符串到缓冲区
    luaL_pushresult(&b);  // 创建并返回新字符串
  }
  lua_pushinteger(L, n);  // 压入替换次数
  return 2;  // 返回结果
}

/* }====================================================== */



/*
** {======================================================
** STRING FORMAT
** =======================================================
*/

#if !defined(lua_number2strx)    /* { */

/*
** Hexadecimal floating-point formatter
*/

#define SIZELENMOD    (sizeof(LUA_NUMBER_FRMLEN)/sizeof(char))
/* 
** Number of bits that goes into the first digit. It can be any value
** between 1 and 4; the following definition tries to align the number
** to nibble boundaries by making what is left after that first digit a
** multiple of 4.
*/
#define L_NBFD        ((l_floatatt(MANT_DIG) - 1)%4 + 1)


/*
** Add integer part of 'x' to buffer and return new 'x'
*/
static lua_Number adddigit (char *buff, int n, lua_Number x) {
  lua_Number dd = l_mathop(floor)(x);  /* get integer part from 'x' */
  int d = (int)dd;
  buff[n] = (d < 10 ? d + '0' : d - 10 + 'a');  /* add to buffer */
  return x - dd;  /* return what is left */
}


static int num2straux (char *buff, int sz, lua_Number x) {
  /* if 'inf' or 'NaN', format it like '%g' */
  if (x != x || x == (lua_Number)HUGE_VAL || x == -(lua_Number)HUGE_VAL)
    return l_sprintf(buff, sz, LUA_NUMBER_FMT, (LUAI_UACNUMBER)x);
  else if (x == 0) {  /* can be -0... */
    /* create "0" or "-0" followed by exponent */
    return l_sprintf(buff, sz, LUA_NUMBER_FMT "x0p+0", (LUAI_UACNUMBER)x);
  }
  else {
    int e;
    lua_Number m = l_mathop(frexp)(x, &e);  /* 'x' fraction and exponent */
    int n = 0;  /* character count */
    if (m < 0) {  /* is number negative? */
      buff[n++] = '-';  /* add sign */
      m = -m;  /* make it positive */
    }
    buff[n++] = '0'; buff[n++] = 'x';  /* add "0x" */
    m = adddigit(buff, n++, m * (1 << L_NBFD));  /* add first digit */
    e -= L_NBFD;  /* this digit goes before the radix point */
    if (m > 0) {  /* more digits? */
      buff[n++] = lua_getlocaledecpoint();  /* add radix point */
      do {  /* add as many digits as needed */
        m = adddigit(buff, n++, m * 16);
      } while (m > 0);
    }
    n += l_sprintf(buff + n, sz - n, "p%+d", e);  /* add exponent */
    lua_assert(n < sz);
    return n;
  }
}
static int lua_number2strx (lua_State *L, char *buff, int sz,
                            const char *fmt, lua_Number x) {
  // 将 lua_Number 转换为字符串，并将结果存储到 buff 中，返回转换后的字符串长度
  int n = num2straux(buff, sz, x);
  // 如果格式化字符串中的大小写转换标志为 'A'
  if (fmt[SIZELENMOD] == 'A') {
    int i;
    // 将转换后的字符串中的字符全部转换为大写
    for (i = 0; i < n; i++)
      buff[i] = toupper(uchar(buff[i]));
  }
  // 如果格式化字符串中的大小写转换标志不是 'a' 或 'A'，则抛出错误
  else if (l_unlikely(fmt[SIZELENMOD] != 'a'))
    return luaL_error(L, "modifiers for format '%%a'/'%%A' not implemented");
  // 返回转换后的字符串长度
  return n;
}

#endif                /* } */


/*
** Maximum size for items formatted with '%f'. This size is produced
** by format('%.99f', -maxfloat), and is equal to 99 + 3 ('-', '.',
** and '\0') + number of decimal digits to represent maxfloat (which
** is maximum exponent + 1). (99+3+1, adding some extra, 110)
*/
// 定义使用 '%f' 格式化的项目的最大大小
#define MAX_ITEMF    (110 + l_floatatt(MAX_10_EXP))


/*
** All formats except '%f' do not need that large limit.  The other
** float formats use exponents, so that they fit in the 99 limit for
** significant digits; 's' for large strings and 'q' add items directly
** to the buffer; all integer formats also fit in the 99 limit.  The
** worst case are floats: they may need 99 significant digits, plus
** '0x', '-', '.', 'e+XXXX', and '\0'. Adding some extra, 120.
*/
// 定义除 '%f' 之外的所有格式的项目的最大大小
#define MAX_ITEM    120


/* valid flags in a format specification */
#if !defined(L_FMTFLAGSF)

/* valid flags for a, A, e, E, f, F, g, and G conversions */
#define L_FMTFLAGSF    "-+#0 "

/* valid flags for o, x, and X conversions */
#define L_FMTFLAGSX    "-#0"

/* valid flags for d and i conversions */
#define L_FMTFLAGSI    "-+0 "

/* valid flags for u conversions */
#define L_FMTFLAGSU    "-0"

/* valid flags for c, p, and s conversions */
#define L_FMTFLAGSC    "-"
#endif


/*
** Maximum size of each format specification (such as "%-099.99d"):
** Initial '%', flags (up to 5), width (2), period, precision (2),
** length modifier (8), conversion specifier, and final '\0', plus some
** extra.
*/
// 定义每个格式规范的最大大小
#define MAX_FORMAT    32
static void addquoted (luaL_Buffer *b, const char *s, size_t len) {
  // 在缓冲区中添加双引号
  luaL_addchar(b, '"');
  // 遍历字符串
  while (len--) {
    // 如果字符是双引号、反斜杠或换行符，则在缓冲区中添加转义字符和原字符
    if (*s == '"' || *s == '\\' || *s == '\n') {
      luaL_addchar(b, '\\');
      luaL_addchar(b, *s);
    }
    // 如果字符是控制字符，则格式化成转义字符序列
    else if (iscntrl(uchar(*s))) {
      char buff[10];
      if (!isdigit(uchar(*(s+1))))
        l_sprintf(buff, sizeof(buff), "\\%d", (int)uchar(*s));
      else
        l_sprintf(buff, sizeof(buff), "\\%03d", (int)uchar(*s));
      luaL_addstring(b, buff);
    }
    // 否则直接添加字符到缓冲区
    else
      luaL_addchar(b, *s);
    s++;
  }
  // 在缓冲区中添加双引号
  luaL_addchar(b, '"');
}


/*
** Serialize a floating-point number in such a way that it can be
** scanned back by Lua. Use hexadecimal format for "common" numbers
** (to preserve precision); inf, -inf, and NaN are handled separately.
** (NaN cannot be expressed as a numeral, so we write '(0/0)' for it.)
*/
static int quotefloat (lua_State *L, char *buff, lua_Number n) {
  const char *s;  /* for the fixed representations */
  // 如果是无穷大，则使用固定表示形式
  if (n == (lua_Number)HUGE_VAL)  /* inf? */
    s = "1e9999";
  // 如果是负无穷大，则使用固定表示形式
  else if (n == -(lua_Number)HUGE_VAL)  /* -inf? */
    s = "-1e9999";
  // 如果是 NaN，则使用固定表示形式
  else if (n != n)  /* NaN? */
    s = "(0/0)";
  else {  /* format number as hexadecimal */
    // 将数字格式化为十六进制
    int  nb = lua_number2strx(L, buff, MAX_ITEM,
                                 "%" LUA_NUMBER_FRMLEN "a", n);
    /* ensures that 'buff' string uses a dot as the radix character */
    // 确保 'buff' 字符串使用点作为基数字符
    if (memchr(buff, '.', nb) == NULL) {  /* no dot? */
      char point = lua_getlocaledecpoint();  /* try locale point */
      char *ppoint = (char *)memchr(buff, point, nb);
      if (ppoint) *ppoint = '.';  /* change it to a dot */
    }
    return nb;
  }
  // 返回固定表示形式的长度
  return l_sprintf(buff, MAX_ITEM, "%s", s);
}


static void addliteral (lua_State *L, luaL_Buffer *b, int arg) {
  switch (lua_type(L, arg)) {
    case LUA_TSTRING: {
      size_t len;
      const char *s = lua_tolstring(L, arg, &len);
      // 将字符串添加到缓冲区中，并进行转义处理
      addquoted(b, s, len);
      break;
    }
    # 如果值的类型是数字
    case LUA_TNUMBER: {
      # 为缓冲区分配一定大小的空间
      char *buff = luaL_prepbuffsize(b, MAX_ITEM);
      int nb;
      # 如果不是整数，则是浮点数
      if (!lua_isinteger(L, arg))  /* float? */
        # 将浮点数格式化为字符串
        nb = quotefloat(L, buff, lua_tonumber(L, arg));
      else {  /* integers */
        # 如果是整数，获取整数值
        lua_Integer n = lua_tointeger(L, arg);
        # 判断是否是最小整数值
        const char *format = (n == LUA_MININTEGER)  /* corner case? */
                           ? "0x%" LUA_INTEGER_FRMLEN "x"  /* use hex */
                           : LUA_INTEGER_FMT;  /* else use default format */
        # 格式化整数值为字符串
        nb = l_sprintf(buff, MAX_ITEM, format, (LUAI_UACINT)n);
      }
      # 将格式化后的字符串添加到缓冲区
      luaL_addsize(b, nb);
      # 结束当前 case
      break;
    }
    # 如果值的类型是空值或布尔值
    case LUA_TNIL: case LUA_TBOOLEAN: {
      # 将值转换为字符串并添加到缓冲区
      luaL_tolstring(L, arg, NULL);
      luaL_addvalue(b);
      # 结束当前 case
      break;
    }
    # 如果值的类型是其他类型
    default: {
      # 抛出错误，表示该值没有字面形式
      luaL_argerror(L, arg, "value has no literal form");
    }
  }
# 返回一个指向两位数字的指针
static const char *get2digits (const char *s) {
  # 如果第一个字符是数字，则指针向后移动一位
  if (isdigit(uchar(*s))) {
    s++;
    # 如果下一个字符也是数字，则指针再向后移动一位（最多两位数字）
    if (isdigit(uchar(*s))) s++;  /* (2 digits at most) */
  }
  return s;
}


/*
** 检查转换规范是否有效。在调用时，'form'中的第一个字符必须是'%'，最后一个字符必须是有效的转换规范。
** 'flags'是接受的标志；'precision'表示是否接受精度。
*/
static void checkformat (lua_State *L, const char *form, const char *flags,
                                       int precision) {
  # 转换规范的指针从'%'后开始
  const char *spec = form + 1;  /* skip '%' */
  # 跳过标志
  spec += strspn(spec, flags);  /* skip flags */
  # 如果规范不以'0'开头，则跳过宽度
  if (*spec != '0') {
    spec = get2digits(spec);  /* skip width */
    # 如果下一个字符是'.'并且precision为真，则跳过精度
    if (*spec == '.' && precision) {
      spec++;
      spec = get2digits(spec);  /* skip precision */
    }
  }
  # 如果规范没有到达结尾，则抛出错误
  if (!isalpha(uchar(*spec)))  /* did not go to the end? */
    luaL_error(L, "invalid conversion specification: '%s'", form);
}


/*
** 获取一个转换规范并将其复制到'form'中。
** 返回其最后一个字符的地址。
*/
static const char *getformat (lua_State *L, const char *strfrmt,
                                            char *form) {
  /* 跨越标志、宽度和精度（'0'被包括在标志中） */
  size_t len = strspn(strfrmt, L_FMTFLAGSF "123456789.");
  len++;  /* 添加后面的字符（应该是转换规范） */
  /* 仍然需要空间来存放'%', '\0'，以及长度修饰符 */
  if (len >= MAX_FORMAT - 10)
    luaL_error(L, "invalid format (too long)");
  *(form++) = '%';
  memcpy(form, strfrmt, len * sizeof(char));
  *(form + len) = '\0';
  return strfrmt + len - 1;
}


/*
** 将长度修饰符添加到格式中
*/
static void addlenmod (char *form, const char *lenmod) {
  size_t l = strlen(form);
  size_t lm = strlen(lenmod);
  char spec = form[l - 1];
  strcpy(form + l - 1, lenmod);
  form[l + lm - 1] = spec;
  form[l + lm] = '\0';
}
static int str_format (lua_State *L) {
  // 获取栈顶索引
  int top = lua_gettop(L);
  // 参数索引
  int arg = 1;
  // 字符串长度
  size_t sfl;
  // 获取参数中的字符串和长度
  const char *strfrmt = luaL_checklstring(L, arg, &sfl);
  // 字符串结束位置
  const char *strfrmt_end = strfrmt+sfl;
  const char *flags;
  // 初始化缓冲区
  luaL_Buffer b;
  luaL_buffinit(L, &b);
  // 遍历字符串
  while (strfrmt < strfrmt_end) {
    // 如果不是转义字符，则添加到缓冲区
    if (*strfrmt != L_ESC)
      luaL_addchar(&b, *strfrmt++);
    // 如果是转义字符
    else if (*++strfrmt == L_ESC)
      luaL_addchar(&b, *strfrmt++);  /* %% */
    }
  }
  // 将缓冲区内容推入栈中
  luaL_pushresult(&b);
  // 返回结果数量
  return 1;
}

/* }====================================================== */


/*
** {======================================================
** PACK/UNPACK
** =======================================================
*/


/* value used for padding */
#if !defined(LUAL_PACKPADBYTE)
#define LUAL_PACKPADBYTE        0x00
#endif

/* maximum size for the binary representation of an integer */
#define MAXINTSIZE    16

/* number of bits in a character */
#define NB    CHAR_BIT

/* mask for one character (NB 1's) */
#define MC    ((1 << NB) - 1)

/* size of a lua_Integer */
#define SZINT    ((int)sizeof(lua_Integer))


/* dummy union to get native endianness */
static const union {
  int dummy;
  char little;  /* true iff machine is little endian */
} nativeendian = {1};


/*
** information to pack/unpack stuff
*/
typedef struct Header {
  lua_State *L;
  int islittle;
  int maxalign;
} Header;


/*
** options for pack/unpack
*/
typedef enum KOption {
  Kint,        /* signed integers */
  Kuint,    /* unsigned integers */
  Kfloat,    /* single-precision floating-point numbers */
  Knumber,    /* Lua "native" floating-point numbers */
  Kdouble,    /* double-precision floating-point numbers */
  Kchar,    /* fixed-length strings */
  Kstring,    /* strings with prefixed length */
  Kzstr,    /* zero-terminated strings */
  Kpadding,    /* padding */
  Kpaddalign,    /* padding for alignment */
  Knop        /* no-op (configuration or spaces) */
} KOption;
/*
** 从字符串'fmt'中读取一个整数数字，如果没有数字则返回'df'
*/
static int digit (int c) { return '0' <= c && c <= '9'; }

static int getnum (const char **fmt, int df) {
  if (!digit(**fmt))  /* 没有数字？ */
    return df;  /* 返回默认值 */
  else {
    int a = 0;
    do {
      a = a*10 + (*((*fmt)++) - '0');
    } while (digit(**fmt) && a <= ((int)MAXSIZE - 9)/10);
    return a;
  }
}


/*
** 读取一个整数数字，如果大于整数的最大大小则引发错误
*/
static int getnumlimit (Header *h, const char **fmt, int df) {
  int sz = getnum(fmt, df);
  if (l_unlikely(sz > MAXINTSIZE || sz <= 0))
    return luaL_error(h->L, "integral size (%d) out of limits [1,%d]",
                            sz, MAXINTSIZE);
  return sz;
}


/*
** 初始化Header
*/
static void initheader (lua_State *L, Header *h) {
  h->L = L;
  h->islittle = nativeendian.little;
  h->maxalign = 1;
}


/*
** 读取并分类下一个选项。'size'被填充为选项的大小。
*/
static KOption getoption (Header *h, const char **fmt, int *size) {
  /* 用于获取本机对齐要求的虚拟结构 */
  struct cD { char c; union { LUAI_MAXALIGN; } u; };
  int opt = *((*fmt)++);
  *size = 0;  /* 默认 */
  switch (opt) {
    case 'b': *size = sizeof(char); return Kint;
    case 'B': *size = sizeof(char); return Kuint;
    case 'h': *size = sizeof(short); return Kint;
    case 'H': *size = sizeof(short); return Kuint;
    case 'l': *size = sizeof(long); return Kint;
    case 'L': *size = sizeof(long); return Kuint;
    case 'j': *size = sizeof(lua_Integer); return Kint;
    case 'J': *size = sizeof(lua_Integer); return Kuint;
    case 'T': *size = sizeof(size_t); return Kuint;
    case 'f': *size = sizeof(float); return Kfloat;
    case 'n': *size = sizeof(lua_Number); return Knumber;
    case 'd': *size = sizeof(double); return Kdouble;
    case 'i': *size = getnumlimit(h, fmt, sizeof(int)); return Kint;
    # 如果格式为'I'，则获取限制范围内的整数大小，并返回Kuint
    case 'I': *size = getnumlimit(h, fmt, sizeof(int)); return Kuint;
    # 如果格式为's'，则获取限制范围内的字符串大小，并返回Kstring
    case 's': *size = getnumlimit(h, fmt, sizeof(size_t)); return Kstring;
    # 如果格式为'c'，则获取字符大小，并返回Kchar
    case 'c':
      *size = getnum(fmt, -1);
      # 如果大小为-1，则抛出错误
      if (l_unlikely(*size == -1))
        luaL_error(h->L, "missing size for format option 'c'");
      return Kchar;
    # 如果格式为'z'，则返回Kzstr
    case 'z': return Kzstr;
    # 如果格式为'x'，则大小为1，返回Kpadding
    case 'x': *size = 1; return Kpadding;
    # 如果格式为'X'，则返回Kpaddalign
    case 'X': return Kpaddalign;
    # 如果格式为' '，则跳过
    case ' ': break;
    # 如果格式为'<'，则设置islittle为1
    case '<': h->islittle = 1; break;
    # 如果格式为'>'，则设置islittle为0
    case '>': h->islittle = 0; break;
    # 如果格式为'='，则设置islittle为nativeendian.little
    case '=': h->islittle = nativeendian.little; break;
    # 如果格式为'!'，则设置maxalign为最大对齐值
    case '!': {
      const int maxalign = offsetof(struct cD, u);
      h->maxalign = getnumlimit(h, fmt, maxalign);
      break;
    }
    # 默认情况下，抛出无效格式选项错误
    default: luaL_error(h->L, "invalid format option '%c'", opt);
  }
  # 返回Knop
  return Knop;
# 读取、分类和填充下一个选项的其他细节。
# 'psize' 填充选项的大小，'notoalign' 填充其对齐要求。
# 本地变量 'size' 获取要对齐的大小。（Kpadal 选项始终获得其完整对齐，其他选项受最大对齐（'maxalign'）的限制。Kchar 选项不需要对齐，尽管其大小。）
static KOption getdetails (Header *h, size_t totalsize,
                           const char **fmt, int *psize, int *ntoalign) {
  # 获取选项的详细信息
  KOption opt = getoption(h, fmt, psize);
  int align = *psize;  /* 通常，对齐遵循大小 */
  if (opt == Kpaddalign) {  /* 'X' 从后续选项获取对齐 */
    if (**fmt == '\0' || getoption(h, fmt, &align) == Kchar || align == 0)
      luaL_argerror(h->L, 1, "invalid next option for option 'X'");
  }
  if (align <= 1 || opt == Kchar)  /* 不需要对齐？ */
    *ntoalign = 0;
  else {
    if (align > h->maxalign)  /* 强制最大对齐 */
      align = h->maxalign;
    if (l_unlikely((align & (align - 1)) != 0))  /* 不是2的幂？ */
      luaL_argerror(h->L, 1, "format asks for alignment not power of 2");
    *ntoalign = (align - (int)(totalsize & (align - 1))) & (align - 1);
  }
  return opt;
}

# 用 'size' 字节和 'islittle' 字节顺序打包整数 'n'。
# 最后的 'if' 处理 'size' 大于 Lua 整数大小的情况，必要时纠正额外的符号扩展字节（默认情况下，它们将是零）。
static void packint (luaL_Buffer *b, lua_Unsigned n,
                     int islittle, int size, int neg) {
  char *buff = luaL_prepbuffsize(b, size);
  int i;
  buff[islittle ? 0 : size - 1] = (char)(n & MC);  /* 第一个字节 */
  for (i = 1; i < size; i++) {
    n >>= NB;
    buff[islittle ? i : size - 1 - i] = (char)(n & MC);
  }
  if (neg && size > SZINT) {  /* 负数需要符号扩展？ */
    for (i = SZINT; i < size; i++)  /* 从 SZINT 开始循环直到 size，用于修正额外的字节 */
      buff[islittle ? i : size - 1 - i] = (char)MC;  // 根据字节序将 MC 赋值给 buff 中的对应位置
  }
  luaL_addsize(b, size);  /* 将 size 添加到缓冲区中 */ 
/*
** Copy 'size' bytes from 'src' to 'dest', correcting endianness if
** given 'islittle' is different from native endianness.
*/
static void copywithendian (char *dest, const char *src,
                            int size, int islittle) {
  // 如果给定的 'islittle' 与本机字节序不同，则校正字节序后，从 'src' 复制 'size' 字节到 'dest'
  if (islittle == nativeendian.little)
    memcpy(dest, src, size);
  else {
    dest += size - 1;
    while (size-- != 0)
      *(dest--) = *(src++);
  }
}


static int str_pack (lua_State *L) {
  luaL_Buffer b;
  Header h;
  const char *fmt = luaL_checkstring(L, 1);  /* format string */
  int arg = 1;  /* current argument to pack */
  size_t totalsize = 0;  /* accumulate total size of result */
  initheader(L, &h);
  lua_pushnil(L);  /* mark to separate arguments from string buffer */
  luaL_buffinit(L, &b);
  while (*fmt != '\0') {
    int size, ntoalign;
    KOption opt = getdetails(&h, totalsize, &fmt, &size, &ntoalign);
    totalsize += ntoalign + size;
    while (ntoalign-- > 0)
     luaL_addchar(&b, LUAL_PACKPADBYTE);  /* fill alignment */
    arg++;
    }
  }
  luaL_pushresult(&b);
  return 1;
}


static int str_packsize (lua_State *L) {
  Header h;
  const char *fmt = luaL_checkstring(L, 1);  /* format string */
  size_t totalsize = 0;  /* accumulate total size of result */
  initheader(L, &h);
  while (*fmt != '\0') {
    int size, ntoalign;
    KOption opt = getdetails(&h, totalsize, &fmt, &size, &ntoalign);
    luaL_argcheck(L, opt != Kstring && opt != Kzstr, 1,
                     "variable-length format");
    size += ntoalign;  /* total space used by option */
    luaL_argcheck(L, totalsize <= MAXSIZE - size, 1,
                     "format result too large");
    totalsize += size;
  }
  lua_pushinteger(L, (lua_Integer)totalsize);
  return 1;
}


/*
** Unpack an integer with 'size' bytes and 'islittle' endianness.
** If size is smaller than the size of a Lua integer and integer
** is signed, must do sign extension (propagating the sign to the
*/
/*
** 根据给定的字符串和参数解析出一个整数值，返回解析结果
** 参数：
** L - Lua 状态机
** str - 待解析的字符串
** islittle - 是否小端序
** size - 整数的字节数
** issigned - 是否有符号
*/
static lua_Integer unpackint (lua_State *L, const char *str,
                              int islittle, int size, int issigned) {
  lua_Unsigned res = 0;  // 用于存储解析结果的无符号整数
  int i;
  int limit = (size  <= SZINT) ? size : SZINT;  // 计算循环次数的上限
  for (i = limit - 1; i >= 0; i--) {  // 从高位到低位解析整数
    res <<= NB;  // 左移 NB 位
    res |= (lua_Unsigned)(unsigned char)str[islittle ? i : size - 1 - i];  // 按照大小端序解析每个字节
  }
  if (size < SZINT) {  // 实际大小小于 Lua 整数的大小？
    if (issigned) {  // 需要符号扩展？
      lua_Unsigned mask = (lua_Unsigned)1 << (size*NB - 1);  // 计算符号扩展的掩码
      res = ((res ^ mask) - mask);  // 进行符号扩展
    }
  }
  else if (size > SZINT) {  // 必须检查未读取的字节？
    int mask = (!issigned || (lua_Integer)res >= 0) ? 0 : MC;  // 计算未读取字节的掩码
    for (i = limit; i < size; i++) {  // 遍历未读取的字节
      if (l_unlikely((unsigned char)str[islittle ? i : size - 1 - i] != mask))  // 检查未读取的字节是否符合掩码
        luaL_error(L, "%d-byte integer does not fit into Lua Integer", size);  // 抛出错误
    }
  }
  return (lua_Integer)res;  // 返回解析结果
}


static int str_unpack (lua_State *L) {
  Header h;  // 头部信息
  const char *fmt = luaL_checkstring(L, 1);  // 解析格式字符串
  size_t ld;
  const char *data = luaL_checklstring(L, 2, &ld);  // 获取待解析的数据
  size_t pos = posrelatI(luaL_optinteger(L, 3, 1), ld) - 1;  // 获取起始解析位置
  int n = 0;  /* 结果数量 */
  luaL_argcheck(L, pos <= ld, 3, "initial position out of string");  // 检查起始位置是否合法
  initheader(L, &h);  // 初始化头部信息
  while (*fmt != '\0') {  // 遍历格式字符串
    int size, ntoalign;
    KOption opt = getdetails(&h, pos, &fmt, &size, &ntoalign);  // 获取格式字符串中的详细信息
    luaL_argcheck(L, (size_t)ntoalign + size <= ld - pos, 2,
                    "data string too short");  // 检查数据字符串是否足够长
    pos += ntoalign;  /* 跳过对齐字节 */
    /* 为解析结果和下一个位置预留栈空间 */
    luaL_checkstack(L, 2, "too many results");
    n++;
    # 根据不同的选项进行不同的处理
    switch (opt) {
      case Kint:
      case Kuint: {
        # 根据选项解析整数，并压入栈中
        lua_Integer res = unpackint(L, data + pos, h.islittle, size,
                                       (opt == Kint));
        lua_pushinteger(L, res);
        break;
      }
      case Kfloat: {
        # 解析浮点数，并压入栈中
        float f;
        copywithendian((char *)&f, data + pos, sizeof(f), h.islittle);
        lua_pushnumber(L, (lua_Number)f);
        break;
      }
      case Knumber: {
        # 解析数字，并压入栈中
        lua_Number f;
        copywithendian((char *)&f, data + pos, sizeof(f), h.islittle);
        lua_pushnumber(L, f);
        break;
      }
      case Kdouble: {
        # 解析双精度浮点数，并压入栈中
        double f;
        copywithendian((char *)&f, data + pos, sizeof(f), h.islittle);
        lua_pushnumber(L, (lua_Number)f);
        break;
      }
      case Kchar: {
        # 将字符数组压入栈中
        lua_pushlstring(L, data + pos, size);
        break;
      }
      case Kstring: {
        # 解析字符串，并压入栈中
        size_t len = (size_t)unpackint(L, data + pos, h.islittle, size, 0);
        luaL_argcheck(L, len <= ld - pos - size, 2, "data string too short");
        lua_pushlstring(L, data + pos + size, len);
        pos += len;  /* skip string */
        break;
      }
      case Kzstr: {
        # 解析以'\0'结尾的字符串，并压入栈中
        size_t len = strlen(data + pos);
        luaL_argcheck(L, pos + len < ld, 2,
                         "unfinished string for format 'z'");
        lua_pushlstring(L, data + pos, len);
        pos += len + 1;  /* skip string plus final '\0' */
        break;
      }
      case Kpaddalign: case Kpadding: case Knop:
        n--;  /* undo increment */
        break;
    }
    # 更新位置
    pos += size;
  }
  # 压入下一个位置
  lua_pushinteger(L, pos + 1);  /* next position */
  return n + 1;
/* }====================================================== */

// 定义字符串库的函数列表
static const luaL_Reg strlib[] = {
  {"byte", str_byte},  // 返回字符的整数表示
  {"char", str_char},  // 接受一系列整数，返回对应的字符
  {"dump", str_dump},  // 返回字符串的十六进制表示
  {"find", str_find},  // 在字符串中查找指定模式的匹配
  {"format", str_format},  // 返回格式化后的字符串
  {"gmatch", gmatch},  // 返回一个迭代器函数，用于遍历字符串中的所有匹配
  {"gsub", str_gsub},  // 替换字符串中的匹配
  {"len", str_len},  // 返回字符串的长度
  {"lower", str_lower},  // 返回字符串的小写形式
  {"match", str_match},  // 在字符串中查找指定模式的匹配
  {"rep", str_rep},  // 返回重复指定次数的字符串
  {"reverse", str_reverse},  // 返回字符串的逆序
  {"sub", str_sub},  // 返回字符串的子串
  {"upper", str_upper},  // 返回字符串的大写形式
  {"pack", str_pack},  // 返回按指定格式打包后的字符串
  {"packsize", str_packsize},  // 返回按指定格式打包后的字符串的长度
  {"unpack", str_unpack},  // 返回按指定格式解包后的值
  {NULL, NULL}  // 结束标记
};

// 创建字符串元表
static void createmetatable (lua_State *L) {
  /* table to be metatable for strings */
  luaL_newlibtable(L, stringmetamethods);  // 创建一个新的表
  luaL_setfuncs(L, stringmetamethods, 0);  // 将字符串元方法添加到表中
  lua_pushliteral(L, "");  /* dummy string */  // 将一个空字符串压入栈中
  lua_pushvalue(L, -2);  /* copy table */  // 复制表
  lua_setmetatable(L, -2);  /* set table as metatable for strings */  // 将表设置为字符串的元表
  lua_pop(L, 1);  /* pop dummy string */  // 弹出栈顶的元素
  lua_pushvalue(L, -2);  /* get string library */  // 获取字符串库
  lua_setfield(L, -2, "__index");  /* metatable.__index = string */  // 设置元表的__index字段为字符串库
  lua_pop(L, 1);  /* pop metatable */  // 弹出栈顶的元素
}

/*
** Open string library
*/
// 打开字符串库
LUAMOD_API int luaopen_string (lua_State *L) {
  luaL_newlib(L, strlib);  // 创建新的库
  createmetatable(L);  // 创建字符串元表
  return 1;  // 返回结果
}
```