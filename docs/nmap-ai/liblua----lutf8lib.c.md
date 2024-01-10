# `nmap\liblua\lutf8lib.c`

```
# 定义 lutf8lib_c 宏，用于标识 lutf8lib.c 文件
#define lutf8lib_c
# 定义 LUA_LIB 宏，用于标识该文件为 Lua 标准库
#define LUA_LIB

# 包含 lua.h 头文件
#include "lua.h"
# 包含 lauxlib.h 头文件
#include "lauxlib.h"
# 包含 lualib.h 头文件
#include "lualib.h"

# 定义最大 Unicode 值
#define MAXUNICODE    0x10FFFFu
# 定义最大 UTF-8 值
#define MAXUTF        0x7FFFFFFFu

# 如果系统支持 31 位整数，则使用 unsigned int 类型，否则使用 unsigned long 类型
#if (UINT_MAX >> 30) >= 1
typedef unsigned int utfint;
#else
typedef unsigned long utfint;
#endif

# 定义宏函数，用于判断是否为 UTF-8 的 continuation 字节
#define iscont(p)    ((*(p) & 0xC0) == 0x80)

# 定义函数，用于将相对位置转换为绝对位置
static lua_Integer u_posrelat (lua_Integer pos, size_t len) {
  if (pos >= 0) return pos;
  else if (0u - (size_t)pos > len) return 0;
  else return (lua_Integer)len + pos + 1;
}

# 定义函数，用于解码一个 UTF-8 序列
static const char *utf8_decode (const char *s, utfint *val, int strict) {
  # 定义 limits 数组，存储每个序列长度的最小值，用于检查过长的表示
  static const utfint limits[] =
        {~(utfint)0, 0x80, 0x800, 0x10000u, 0x200000u, 0x4000000u};
  # 读取第一个字节
  unsigned int c = (unsigned char)s[0];
  # 最终结果
  utfint res = 0;
  # 如果是 ASCII 字符
  if (c < 0x80)
    res = c;
  else:
    # 计数 continuation 字节的数量
    int count = 0;
    # 当需要 continuation 字节时
    for (; c & 0x40; c <<= 1):
      # 读取下一个字节
      unsigned int cc = (unsigned char)s[++count];
      # 如果不是 continuation 字节，则返回空
      if ((cc & 0xC0) != 0x80)
        return NULL;
      # 将低 6 位添加到结果中
      res = (res << 6) | (cc & 0x3F);
    # 添加第一个字节
    res |= ((utfint)(c & 0x7F) << (count * 5));
    # 如果连续字节大于5个，或者结果大于最大UTF值，或者结果小于限制值，则返回空
    if (count > 5 || res > MAXUTF || res < limits[count])
      return NULL;  /* 无效的字节序列 */
    # 跳过已读取的连续字节
    s += count;  /* 跳过已读取的连续字节 */
  }
  # 如果启用严格模式
  if (strict) {
    # 检查无效的码点；过大或者代理项
    if (res > MAXUNICODE || (0xD800u <= res && res <= 0xDFFFu))
      return NULL;
  }
  # 如果val不为空，则将结果赋值给val
  if (val) *val = res;
  # 返回s加1，包括第一个字节
  return s + 1;  /* +1 to include first byte */
/*
** utf8len(s [, i [, j [, lax]]]) --> number of characters that
** start in the range [i,j], or nil + current position if 's' is not
** well formed in that interval
*/
static int utflen (lua_State *L) {
  lua_Integer n = 0;  /* counter for the number of characters */
  size_t len;  /* string length in bytes */
  const char *s = luaL_checklstring(L, 1, &len);  // 获取 Lua 函数参数中的字符串和其长度
  lua_Integer posi = u_posrelat(luaL_optinteger(L, 2, 1), len);  // 获取起始位置参数，转换为有效的索引值
  lua_Integer posj = u_posrelat(luaL_optinteger(L, 3, -1), len);  // 获取结束位置参数，转换为有效的索引值
  int lax = lua_toboolean(L, 4);  // 获取是否宽松模式的参数
  luaL_argcheck(L, 1 <= posi && --posi <= (lua_Integer)len, 2,
                   "initial position out of bounds");  // 检查起始位置是否在字符串范围内
  luaL_argcheck(L, --posj < (lua_Integer)len, 3,
                   "final position out of bounds");  // 检查结束位置是否在字符串范围内
  while (posi <= posj) {  // 循环遍历起始位置到结束位置的字符
    const char *s1 = utf8_decode(s + posi, NULL, !lax);  // 解码 UTF-8 字符串，获取下一个字符的位置
    if (s1 == NULL) {  /* conversion error? */
      luaL_pushfail(L);  /* return fail ... */
      lua_pushinteger(L, posi + 1);  /* ... and current position */
      return 2;  // 如果解码出错，返回失败和当前位置
    }
    posi = s1 - s;  // 更新当前位置
    n++;  // 统计字符数量
  }
  lua_pushinteger(L, n);  // 将字符数量压入栈
  return 1;  // 返回结果数量
}


/*
** codepoint(s, [i, [j [, lax]]]) -> returns codepoints for all
** characters that start in the range [i,j]
*/
static int codepoint (lua_State *L) {
  size_t len;
  const char *s = luaL_checklstring(L, 1, &len);  // 获取 Lua 函数参数中的字符串和其长度
  lua_Integer posi = u_posrelat(luaL_optinteger(L, 2, 1), len);  // 获取起始位置参数，转换为有效的索引值
  lua_Integer pose = u_posrelat(luaL_optinteger(L, 3, posi), len);  // 获取结束位置参数，转换为有效的索引值
  int lax = lua_toboolean(L, 4);  // 获取是否宽松模式的参数
  int n;
  const char *se;
  luaL_argcheck(L, posi >= 1, 2, "out of bounds");  // 检查起始位置是否在字符串范围内
  luaL_argcheck(L, pose <= (lua_Integer)len, 3, "out of bounds");  // 检查结束位置是否在字符串范围内
  if (posi > pose) return 0;  /* empty interval; return no values */  // 如果起始位置大于结束位置，返回 0
  if (pose - posi >= INT_MAX)  /* (lua_Integer -> int) overflow? */  // 检查索引值是否溢出
    # 如果字符串切片过长，返回错误信息
    return luaL_error(L, "string slice too long");
  # 计算返回值的上限
  n = (int)(pose -  posi) + 1;  /* upper bound for number of returns */
  # 检查 Lua 栈的空间是否足够存放 n 个元素，如果不够则报错
  luaL_checkstack(L, n, "string slice too long");
  # 重置 n 为 0，用于计数返回值的个数
  n = 0;  /* count the number of returns */
  # 计算字符串的结束位置
  se = s + pose;  /* string end */
  # 遍历字符串，将每个 UTF-8 编码的字符转换为整数并压入 Lua 栈
  for (s += posi - 1; s < se;) {
    # 存储 UTF-8 编码的字符
    utfint code;
    # 解码 UTF-8 字符，并将结果存入 code 中
    s = utf8_decode(s, &code, !lax);
    # 如果解码失败，返回错误信息
    if (s == NULL)
      return luaL_error(L, "invalid UTF-8 code");
    # 将解码后的整数压入 Lua 栈
    lua_pushinteger(L, code);
    # 计数返回值的个数
    n++;
  }
  # 返回返回值的个数
  return n;
}

// 将整数参数转换为 UTF-8 字符并压入栈中
static void pushutfchar (lua_State *L, int arg) {
  lua_Unsigned code = (lua_Unsigned)luaL_checkinteger(L, arg);  // 获取整数参数
  luaL_argcheck(L, code <= MAXUTF, arg, "value out of range");  // 检查参数范围
  lua_pushfstring(L, "%U", (long)code);  // 将 UTF-8 字符串压入栈中
}

// 将整数参数转换为 UTF-8 字符串
static int utfchar (lua_State *L) {
  int n = lua_gettop(L);  // 获取参数个数
  if (n == 1)  // 优化常见情况：单个字符
    pushutfchar(L, 1);
  else {
    int i;
    luaL_Buffer b;
    luaL_buffinit(L, &b);  // 初始化缓冲区
    for (i = 1; i <= n; i++) {
      pushutfchar(L, i);  // 将整数参数转换为 UTF-8 字符并压入栈中
      luaL_addvalue(&b);  // 将栈顶元素添加到缓冲区中
    }
    luaL_pushresult(&b);  // 将缓冲区中的内容压入栈中
  }
  return 1;  // 返回结果个数
}

// 返回指定字符的偏移量
static int byteoffset (lua_State *L) {
  size_t len;
  const char *s = luaL_checklstring(L, 1, &len);  // 获取字符串参数及其长度
  lua_Integer n  = luaL_checkinteger(L, 2);  // 获取整数参数
  lua_Integer posi = (n >= 0) ? 1 : len + 1;  // 计算起始位置
  posi = u_posrelat(luaL_optinteger(L, 3, posi), len);  // 计算相对位置
  luaL_argcheck(L, 1 <= posi && --posi <= (lua_Integer)len, 3, "position out of bounds");  // 检查位置是否越界
  if (n == 0) {
    // 查找当前字节序列的起始位置
    while (posi > 0 && iscont(s + posi)) posi--;
  }
  else {
    if (iscont(s + posi))
      return luaL_error(L, "initial position is a continuation byte");  // 报错：初始位置是一个连续字节
    if (n < 0) {
       while (n < 0 && posi > 0) {  // 向前移动
         do {  // 查找前一个字符的起始位置
           posi--;
         } while (posi > 0 && iscont(s + posi));
         n++;
       }
     }
     else {
       n--;  // 第一个字符不移动
       while (n > 0 && posi < (lua_Integer)len) {
         do {  // 查找下一个字符的起始位置
           posi++;
         } while (iscont(s + posi));  // （不能超过最后的 '\0'）
         n--;
       }
     }
  }
  if (n == 0)  // 是否找到指定字符？
    lua_pushinteger(L, posi + 1);  // 将结果压入栈中
  else  // 没有找到指定字符
    # 将失败标记推送到 Lua 栈顶
    luaL_pushfail(L);
    # 返回 1，表示执行失败
    return 1;
static int iter_aux (lua_State *L, int strict) {
  size_t len;
  const char *s = luaL_checklstring(L, 1, &len);  // 检查并获取参数1的字符串，并返回其长度
  lua_Unsigned n = (lua_Unsigned)lua_tointeger(L, 2);  // 将参数2转换为无符号整数
  if (n < len) {
    while (iscont(s + n)) n++;  /* skip continuation bytes */  // 如果n小于长度，则跳过连续字节
  }
  if (n >= len)  /* (also handles original 'n' being negative) */
    return 0;  /* no more codepoints */  // 如果n大于等于长度，表示没有更多的码点，返回0
  else {
    utfint code;
    const char *next = utf8_decode(s + n, &code, strict);  // 解码UTF-8字符，返回下一个字符的指针
    if (next == NULL)
      return luaL_error(L, "invalid UTF-8 code");  // 如果解码失败，返回错误信息
    lua_pushinteger(L, n + 1);  // 将n + 1压入栈
    lua_pushinteger(L, code);  // 将解码后的字符压入栈
    return 2;  // 返回2表示成功
  }
}

static int iter_auxstrict (lua_State *L) {
  return iter_aux(L, 1);  // 调用iter_aux函数，传入参数1
}

static int iter_auxlax (lua_State *L) {
  return iter_aux(L, 0);  // 调用iter_aux函数，传入参数0
}

static int iter_codes (lua_State *L) {
  int lax = lua_toboolean(L, 2);  // 获取参数2的布尔值
  luaL_checkstring(L, 1);  // 检查参数1是否为字符串
  lua_pushcfunction(L, lax ? iter_auxlax : iter_auxstrict);  // 将iter_auxlax或iter_auxstrict函数压入栈
  lua_pushvalue(L, 1);  // 将参数1的值压入栈
  lua_pushinteger(L, 0);  // 将整数0压入栈
  return 3;  // 返回3表示成功
}

/* pattern to match a single UTF-8 character */
#define UTF8PATT    "[\0-\x7F\xC2-\xFD][\x80-\xBF]*"  // 定义用于匹配单个UTF-8字符的模式

static const luaL_Reg funcs[] = {
  {"offset", byteoffset},  // 注册名为"offset"的函数byteoffset
  {"codepoint", codepoint},  // 注册名为"codepoint"的函数codepoint
  {"char", utfchar},  // 注册名为"char"的函数utfchar
  {"len", utflen},  // 注册名为"len"的函数utflen
  {"codes", iter_codes},  // 注册名为"codes"的函数iter_codes
  /* placeholders */
  {"charpattern", NULL},  // 注册名为"charpattern"的占位符
  {NULL, NULL}  // 结束标记
};

LUAMOD_API int luaopen_utf8 (lua_State *L) {
  luaL_newlib(L, funcs);  // 创建新的库
  lua_pushlstring(L, UTF8PATT, sizeof(UTF8PATT)/sizeof(char) - 1);  // 将UTF8PATT字符串压入栈
  lua_setfield(L, -2, "charpattern");  // 设置库中字段"charpattern"的值
  return 1;  // 返回1表示成功
}
```