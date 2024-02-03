# `nmap\liblua\loslib.c`

```cpp
/*
** $Id: loslib.c $
** Standard Operating System library
** See Copyright Notice in lua.h
*/

// 定义 loslib.c，标准操作系统库
#define loslib_c
// 定义 LUA_LIB
#define LUA_LIB

// 包含 lprefix.h 文件
#include "lprefix.h"

// 包含标准库头文件
#include <errno.h>
#include <locale.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// 包含 lua.h 头文件
#include "lua.h"

// 包含 lauxlib.h 头文件
#include "lauxlib.h"
// 包含 lualib.h 头文件
#include "lualib.h"


/*
** {==================================================================
** List of valid conversion specifiers for the 'strftime' function;
** options are grouped by length; group of length 2 start with '||'.
** ===================================================================
*/
#if !defined(LUA_STRFTIMEOPTIONS)    /* { */

/* options for ANSI C 89 (only 1-char options) */
// ANSI C 89 标准的 'strftime' 函数的有效转换说明符列表
#define L_STRFTIMEC89        "aAbBcdHIjmMpSUwWxXyYZ%"

/* options for ISO C 99 and POSIX */
// ISO C 99 和 POSIX 标准的 'strftime' 函数的有效转换说明符列表
#define L_STRFTIMEC99 "aAbBcCdDeFgGhHIjmMnprRStTuUVwWxXyYzZ%" \
    "||" "EcECExEXEyEY" "OdOeOHOIOmOMOSOuOUOVOwOWOy"  /* two-char options */

/* options for Windows */
// Windows 系统的 'strftime' 函数的有效转换说明符列表
#define L_STRFTIMEWIN "aAbBcdHIjmMpSUwWxXyYzZ%" \
    "||" "#c#x#d#H#I#j#m#M#S#U#w#W#y#Y"  /* two-char options */

#if defined(LUA_USE_WINDOWS)
// 如果是 Windows 系统，则使用 Windows 的转换说明符列表
#define LUA_STRFTIMEOPTIONS    L_STRFTIMEWIN
#elif defined(LUA_USE_C89)
// 如果是 ANSI C 89 标准，则使用 ANSI C 89 的转换说明符列表
#define LUA_STRFTIMEOPTIONS    L_STRFTIMEC89
#else  /* C99 specification */
// 否则使用 ISO C 99 和 POSIX 标准的转换说明符列表
#define LUA_STRFTIMEOPTIONS    L_STRFTIMEC99
#endif

#endif                    /* } */
/* }================================================================== */


/*
** {==================================================================
** Configuration for time-related stuff
** ===================================================================
*/

/*
** type to represent time_t in Lua
*/
#if !defined(LUA_NUMTIME)    /* { */

// 用于在 Lua 中表示 time_t 的类型
#define l_timet            lua_Integer
// 将 time_t 压入 Lua 栈中
#define l_pushtime(L,t)        lua_pushinteger(L,(lua_Integer)(t))
// 从 Lua 栈中获取 time_t
#define l_gettime(L,arg)    luaL_checkinteger(L, arg)

#else                /* }{ */

// 用于在 Lua 中表示 time_t 的类型
#define l_timet            lua_Number
// 将 time_t 压入 Lua 栈中
#define l_pushtime(L,t)        lua_pushnumber(L,(lua_Number)(t))
#define l_gettime(L,arg)    luaL_checknumber(L, arg)
// 宏定义，用于获取 Lua 参数中的时间值

#endif                /* } */


#if !defined(l_gmtime)        /* { */
/*
** By default, Lua uses gmtime/localtime, except when POSIX is available,
** where it uses gmtime_r/localtime_r
*/

#if defined(LUA_USE_POSIX)    /* { */

#define l_gmtime(t,r)        gmtime_r(t,r)
#define l_localtime(t,r)    localtime_r(t,r)
// 如果支持 POSIX，使用 gmtime_r 和 localtime_r 函数

#else                /* }{ */

/* ISO C definitions */
#define l_gmtime(t,r)        ((void)(r)->tm_sec, gmtime(t))
#define l_localtime(t,r)    ((void)(r)->tm_sec, localtime(t))
// 否则使用 ISO C 定义的 gmtime 和 localtime 函数

#endif                /* } */

#endif                /* } */

/* }================================================================== */


/*
** {==================================================================
** Configuration for 'tmpnam':
** By default, Lua uses tmpnam except when POSIX is available, where
** it uses mkstemp.
** ===================================================================
*/
#if !defined(lua_tmpnam)    /* { */

#if defined(LUA_USE_POSIX)    /* { */

#include <unistd.h>

#define LUA_TMPNAMBUFSIZE    32
// 定义临时文件名缓冲区大小

#if !defined(LUA_TMPNAMTEMPLATE)
#define LUA_TMPNAMTEMPLATE    "/tmp/lua_XXXXXX"
#endif
// 如果未定义临时文件名模板，则使用默认模板

#define lua_tmpnam(b,e) { \
        strcpy(b, LUA_TMPNAMTEMPLATE); \
        e = mkstemp(b); \
        if (e != -1) close(e); \
        e = (e == -1); }
// 如果支持 POSIX，使用 mkstemp 函数生成临时文件名

#else                /* }{ */

/* ISO C definitions */
#define LUA_TMPNAMBUFSIZE    L_tmpnam
#define lua_tmpnam(b,e)        { e = (tmpnam(b) == NULL); }
// 否则使用 ISO C 定义的 tmpnam 函数生成临时文件名

#endif                /* } */

#endif                /* } */
/* }================================================================== */



static int os_execute (lua_State *L) {
  const char *cmd = luaL_optstring(L, 1, NULL);
  int stat;
  errno = 0;
  stat = system(cmd);
  if (cmd != NULL)
    return luaL_execresult(L, stat);
  else {
    lua_pushboolean(L, stat);  /* true if there is a shell */
    return 1;
  }
}
// 定义 os_execute 函数，用于执行系统命令
  # 从 Lua 栈中获取第一个参数作为文件名，检查其类型是否为字符串
  const char *filename = luaL_checkstring(L, 1);
  # 调用 remove 函数删除指定文件，并将结果作为布尔值返回到 Lua 栈
  return luaL_fileresult(L, remove(filename) == 0, filename);
}


static int os_rename (lua_State *L) {
  # 从 Lua 栈中获取前两个参数作为原文件名和目标文件名，检查它们的类型是否为字符串
  const char *fromname = luaL_checkstring(L, 1);
  const char *toname = luaL_checkstring(L, 2);
  # 调用 rename 函数重命名文件，并将结果作为布尔值返回到 Lua 栈
  return luaL_fileresult(L, rename(fromname, toname) == 0, NULL);
}


static int os_tmpname (lua_State *L) {
  # 创建一个缓冲区用于存储临时文件名
  char buff[LUA_TMPNAMBUFSIZE];
  int err;
  # 生成一个唯一的临时文件名，并将结果存储在 buff 中
  lua_tmpnam(buff, err);
  # 如果生成临时文件名出错，则返回错误信息到 Lua 栈
  if (l_unlikely(err))
    return luaL_error(L, "unable to generate a unique filename");
  # 将生成的临时文件名压入 Lua 栈
  lua_pushstring(L, buff);
  return 1;
}


static int os_getenv (lua_State *L) {
  # 从 Lua 栈中获取参数作为环境变量名，获取对应的环境变量值并压入 Lua 栈
  lua_pushstring(L, getenv(luaL_checkstring(L, 1)));  /* if NULL push nil */
  return 1;
}


static int os_clock (lua_State *L) {
  # 获取当前时钟时间并将其转换为 Lua 数字类型，然后压入 Lua 栈
  lua_pushnumber(L, ((lua_Number)clock())/(lua_Number)CLOCKS_PER_SEC);
  return 1;
}


/*
** {======================================================
** Time/Date operations
** { year=%Y, month=%m, day=%d, hour=%H, min=%M, sec=%S,
**   wday=%w+1, yday=%j, isdst=? }
** =======================================================
*/

/*
** 关于溢出检查：当时间由 lua_Integer 表示时，不会发生溢出，因为 lua_Integer 足够大，可以表示所有 int 字段，
** 或者它不足以表示导致字段溢出的时间。然而，如果时间以双精度表示，并且 lua_Integer 是 int 类型，
** 那么当将 0x1.e1853b0d184f6p+55 加上 1900 以计算年份时，会导致溢出。
*/
static void setfield (lua_State *L, const char *key, int value, int delta) {
  #if (defined(LUA_NUMTIME) && LUA_MAXINTEGER <= INT_MAX)
    # 如果 value + delta 大于 LUA_MAXINTEGER，则抛出错误
    if (l_unlikely(value > LUA_MAXINTEGER - delta))
      luaL_error(L, "field '%s' is out-of-bound", key);
  #endif
  # 将 value + delta 压入 Lua 栈
  lua_pushinteger(L, (lua_Integer)value + delta);
  # 设置 Lua 表中指定 key 的字段值
  lua_setfield(L, -2, key);
}


static void setboolfield (lua_State *L, const char *key, int value) {
  # 如果 value 小于 0，则表示未定义
    # 返回空值，不设置字段
    return;  
    # 将布尔值压入 Lua 栈
    lua_pushboolean(L, value);
    # 设置 Lua 表中指定索引处的字段值
    lua_setfield(L, -2, key);
/*
** 将结构体'tm'中的所有字段设置为栈顶的表中的值
*/
static void setallfields (lua_State *L, struct tm *stm) {
  // 设置表中的字段'year'为tm结构体中的tm_year字段的值加上1900
  setfield(L, "year", stm->tm_year, 1900);
  // 设置表中的字段'month'为tm结构体中的tm_mon字段的值加上1
  setfield(L, "month", stm->tm_mon, 1);
  // 设置表中的字段'day'为tm结构体中的tm_mday字段的值
  setfield(L, "day", stm->tm_mday, 0);
  // 设置表中的字段'hour'为tm结构体中的tm_hour字段的值
  setfield(L, "hour", stm->tm_hour, 0);
  // 设置表中的字段'min'为tm结构体中的tm_min字段的值
  setfield(L, "min", stm->tm_min, 0);
  // 设置表中的字段'sec'为tm结构体中的tm_sec字段的值
  setfield(L, "sec", stm->tm_sec, 0);
  // 设置表中的字段'yday'为tm结构体中的tm_yday字段的值加上1
  setfield(L, "yday", stm->tm_yday, 1);
  // 设置表中的字段'wday'为tm结构体中的tm_wday字段的值加上1
  setfield(L, "wday", stm->tm_wday, 1);
  // 设置表中的字段'isdst'为tm结构体中的tm_isdst字段的值
  setboolfield(L, "isdst", stm->tm_isdst);
}


// 从表中获取布尔类型字段的值
static int getboolfield (lua_State *L, const char *key) {
  int res;
  // 获取表中指定字段的值，如果为nil则返回-1，否则返回其布尔值
  res = (lua_getfield(L, -1, key) == LUA_TNIL) ? -1 : lua_toboolean(L, -1);
  // 弹出栈顶的值
  lua_pop(L, 1);
  return res;
}


// 从表中获取整数类型字段的值
static int getfield (lua_State *L, const char *key, int d, int delta) {
  int isnum;
  int t = lua_getfield(L, -1, key);  /* 获取字段及其类型 */
  lua_Integer res = lua_tointegerx(L, -1, &isnum);
  if (!isnum) {  /* 字段不是整数? */
    if (l_unlikely(t != LUA_TNIL))  /* 其他值? */
      return luaL_error(L, "field '%s' is not an integer", key);
    else if (l_unlikely(d < 0))  /* 缺少字段; 没有默认值? */
      return luaL_error(L, "field '%s' missing in date table", key);
    res = d;
  }
  else {
    /* 无符号避免lua_Integer为32位时的溢出 */
    if (!(res >= 0 ? (lua_Unsigned)res <= (lua_Unsigned)INT_MAX + delta
                   : (lua_Integer)INT_MIN + delta <= res))
      return luaL_error(L, "field '%s' is out-of-bound", key);
    res -= delta;
  }
  lua_pop(L, 1);
  return (int)res;
}


// 检查选项是否存在
static const char *checkoption (lua_State *L, const char *conv,
                                ptrdiff_t convlen, char *buff) {
  const char *option = LUA_STRFTIMEOPTIONS;
  int oplen = 1;  /* 正在检查的选项的长度 */
  for (; *option != '\0' && oplen <= convlen; option += oplen) {
    if (*option == '|')  /* 下一个块? */
      oplen++;  /* 将检查下一个长度的选项 (+1) */
    else if (memcmp(conv, option, oplen) == 0) {  /* 如果匹配成功 */
      memcpy(buff, conv, oplen);  /* 将有效的选项复制到缓冲区 */
      buff[oplen] = '\0';
      return conv + oplen;  /* 返回下一个项目 */
    }
  }
  luaL_argerror(L, 1,
    lua_pushfstring(L, "invalid conversion specifier '%%%s'", conv));  /* 抛出 Lua 错误，指示无效的转换说明符 */
  return conv;  /* 为了避免警告而返回 conv */
/* 返回 Lua 状态机中指定位置的时间值 */
static time_t l_checktime (lua_State *L, int arg) {
  // 从 Lua 栈中获取时间值
  l_timet t = l_gettime(L, arg);
  // 检查时间值是否在合理范围内
  luaL_argcheck(L, (time_t)t == t, arg, "time out-of-bounds");
  // 返回时间值
  return (time_t)t;
}

/* 用于 'strftime' 项目的最大大小 */
#define SIZETIMEFMT    250

/* Lua 中的 os.date 函数 */
static int os_date (lua_State *L) {
  // 获取格式字符串
  size_t slen;
  const char *s = luaL_optlstring(L, 1, "%c", &slen);
  // 获取时间值
  time_t t = luaL_opt(L, l_checktime, 2, time(NULL));
  const char *se = s + slen;  /* 's' end */
  struct tm tmr, *stm;
  // 判断是否为 UTC 时间
  if (*s == '!') {  /* UTC? */
    // 获取 UTC 时间
    stm = l_gmtime(&t, &tmr);
    s++;  /* skip '!' */
  }
  else
    // 获取本地时间
    stm = l_localtime(&t, &tmr);
  // 判断日期是否有效
  if (stm == NULL)  /* invalid date? */
    return luaL_error(L,
                 "date result cannot be represented in this installation");
  // 判断是否为 "*t" 格式
  if (strcmp(s, "*t") == 0) {
    // 创建 Lua 表
    lua_createtable(L, 0, 9);  /* 9 = number of fields */
    // 设置表中的字段
    setallfields(L, stm);
  }
  else {
    char cc[4];  /* buffer for individual conversion specifiers */
    luaL_Buffer b;
    cc[0] = '%';
    luaL_buffinit(L, &b);
    // 遍历格式字符串
    while (s < se) {
      // 如果不是转换说明符，则直接添加到缓冲区
      if (*s != '%')  /* not a conversion specifier? */
        luaL_addchar(&b, *s++);
      else {
        size_t reslen;
        char *buff = luaL_prepbuffsize(&b, SIZETIMEFMT);
        s++;  /* skip '%' */
        // 复制转换说明符到 'cc'
        s = checkoption(L, s, se - s, cc + 1);  /* copy specifier to 'cc' */
        // 根据转换说明符格式化时间
        reslen = strftime(buff, SIZETIMEFMT, cc, stm);
        luaL_addsize(&b, reslen);
      }
    }
    // 将结果推入 Lua 栈
    luaL_pushresult(&b);
  }
  // 返回结果数量
  return 1;
}

/* Lua 中的 os.time 函数 */
static int os_time (lua_State *L) {
  time_t t;
  // 如果没有参数，则获取当前时间
  if (lua_isnoneornil(L, 1))  /* called without args? */
    t = time(NULL);  /* get current time */
  else {
    struct tm ts;
    luaL_checktype(L, 1, LUA_TTABLE);
    lua_settop(L, 1);  /* make sure table is at the top */
    // 获取年份字段
    ts.tm_year = getfield(L, "year", -1, 1900);
    // 获取月份字段
    ts.tm_mon = getfield(L, "month", -1, 1);
    // 获取日期字段
    ts.tm_mday = getfield(L, "day", -1, 0);
    // 获取小时字段
    ts.tm_hour = getfield(L, "hour", 12, 0);
    // 获取分钟字段
    ts.tm_min = getfield(L, "min", 0, 0);
    # 设置时间结构体的秒字段为从 Lua 表中获取的值，如果获取失败则设置为 0
    ts.tm_sec = getfield(L, "sec", 0, 0);
    # 设置时间结构体的夏令时标志为从 Lua 表中获取的布尔值
    ts.tm_isdst = getboolfield(L, "isdst");
    # 根据时间结构体计算时间的秒数，并赋值给 t
    t = mktime(&ts);
    # 使用规范化后的值更新时间结构体的所有字段
    setallfields(L, &ts);  /* update fields with normalized values */
  }
  # 如果 t 不等于 time_t 类型的 t，或者 t 等于 -1，则返回错误信息
  if (t != (time_t)(l_timet)t || t == (time_t)(-1))
    return luaL_error(L,
                  "time result cannot be represented in this installation");
  # 将时间 t 压入 Lua 栈中
  l_pushtime(L, t);
  # 返回 1 表示成功
  return 1;
/* }====================================================== */

// 定义一个静态函数，计算两个时间之间的差值
static int os_difftime (lua_State *L) {
  // 获取第一个时间参数
  time_t t1 = l_checktime(L, 1);
  // 获取第二个时间参数
  time_t t2 = l_checktime(L, 2);
  // 将时间差值压入栈中
  lua_pushnumber(L, (lua_Number)difftime(t1, t2));
  // 返回1表示有一个返回值
  return 1;
}

// 设置本地化信息
static int os_setlocale (lua_State *L) {
  // 定义本地化类别
  static const int cat[] = {LC_ALL, LC_COLLATE, LC_CTYPE, LC_MONETARY,
                      LC_NUMERIC, LC_TIME};
  // 定义本地化类别名称
  static const char *const catnames[] = {"all", "collate", "ctype", "monetary",
     "numeric", "time", NULL};
  // 获取第一个参数，本地化信息
  const char *l = luaL_optstring(L, 1, NULL);
  // 获取第二个参数，本地化类别
  int op = luaL_checkoption(L, 2, "all", catnames);
  // 将设置的本地化信息压入栈中
  lua_pushstring(L, setlocale(cat[op], l));
  // 返回1表示有一个返回值
  return 1;
}

// 退出程序
static int os_exit (lua_State *L) {
  int status;
  // 判断第一个参数是否为布尔类型
  if (lua_isboolean(L, 1))
    // 如果是布尔类型，根据布尔值设置退出状态
    status = (lua_toboolean(L, 1) ? EXIT_SUCCESS : EXIT_FAILURE);
  else
    // 如果不是布尔类型，获取整数类型的退出状态
    status = (int)luaL_optinteger(L, 1, EXIT_SUCCESS);
  // 判断第二个参数是否为布尔类型
  if (lua_toboolean(L, 2))
    // 如果是布尔类型，关闭 Lua 环境
    lua_close(L);
  // 如果 Lua 环境存在，根据状态退出程序
  if (L) exit(status);  /* 'if' to avoid warnings for unreachable 'return' */
  // 返回0表示没有返回值
  return 0;
}

// 定义系统库函数表
static const luaL_Reg syslib[] = {
  {"clock",     os_clock},
  {"date",      os_date},
  {"difftime",  os_difftime},
  {"execute",   os_execute},
  {"exit",      os_exit},
  {"getenv",    os_getenv},
  {"remove",    os_remove},
  {"rename",    os_rename},
  {"setlocale", os_setlocale},
  {"time",      os_time},
  {"tmpname",   os_tmpname},
  {NULL, NULL}
};

/* }====================================================== */

// 打开 os 模块
LUAMOD_API int luaopen_os (lua_State *L) {
  // 创建新的库
  luaL_newlib(L, syslib);
  // 返回1表示有一个返回值
  return 1;
}
```