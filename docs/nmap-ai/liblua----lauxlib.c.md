# `nmap\liblua\lauxlib.c`

```
/*
** $Id: lauxlib.c $
** Auxiliary functions for building Lua libraries
** See Copyright Notice in lua.h
*/

// 定义宏，标识 lauxlib.c 文件
#define lauxlib_c
// 定义宏，标识为 Lua 库
#define LUA_LIB

// 包含 Lua 的前缀文件
#include "lprefix.h"

// 包含标准库头文件
#include <errno.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 包含 Lua 核心头文件
#include "lua.h"

// 包含 lauxlib 头文件
#include "lauxlib.h"

// 如果未定义 MAX_SIZET，则定义最大的 size_t 值
#if !defined(MAX_SIZET)
/* maximum value for size_t */
#define MAX_SIZET    ((size_t)(~(size_t)0))
#endif

/*
** This file uses only the official API of Lua.
** Any function declared here could be written as an application function.
*/

// 包含 Lua 核心头文件
#include "lua.h"

// 包含 lauxlib 头文件
#include "lauxlib.h"

// 定义常量，表示堆栈的第一部分大小
#define LEVELS1    10    /* size of the first part of the stack */
// 定义常量，表示堆栈的第二部分大小
#define LEVELS2    11    /* size of the second part of the stack */

/*
** {======================================================
** Traceback
** =======================================================
*/

// 定义函数，用于在表中查找指定索引的对象
static int findfield (lua_State *L, int objidx, int level) {
  // 如果递归层级为 0 或者栈顶不是表，则返回 0
  if (level == 0 || !lua_istable(L, -1))
    return 0;  /* not found */
  // 将 nil 压入栈顶，开始遍历表
  lua_pushnil(L);  /* start 'next' loop */
  // 遍历表中的每对键值对
  while (lua_next(L, -2)) {  /* for each pair in table */
    // 如果键的类型是字符串
    if (lua_type(L, -2) == LUA_TSTRING) {  /* ignore non-string keys */
      // 如果找到了指定的对象
      if (lua_rawequal(L, objidx, -1)) {  /* found object? */
        // 弹出值，保留键，返回 1
        lua_pop(L, 1);  /* remove value (but keep name) */
        return 1;
      }
      // 否则尝试递归查找
      else if (findfield(L, objidx, level - 1)) {  /* try recursively */
        /* stack: lib_name, lib_table, field_name (top) */
        // 在两个名称之间放置 '.'，拼接成完整的名称
        lua_pushliteral(L, ".");  /* place '.' between the two names */
        // 替换表中的占位符，拼接成完整的名称
        lua_replace(L, -3);  /* (in the slot occupied by table) */
        // 连接字符串，得到完整的名称
        lua_concat(L, 3);  /* lib_name.field_name */
        return 1;
      }
    }
    // 弹出值
    lua_pop(L, 1);  /* remove value */
  }
  return 0;  /* not found */
}

/*
** Search for a name for a function in all loaded modules
*/
static int pushglobalfuncname (lua_State *L, lua_Debug *ar) {
  int top = lua_gettop(L);  /* 获取栈顶索引 */
  lua_getinfo(L, "f", ar);  /* 获取函数信息并压入栈顶 */
  lua_getfield(L, LUA_REGISTRYINDEX, LUA_LOADED_TABLE);  /* 获取全局表 */
  if (findfield(L, top + 1, 2)) {  /* 在全局表中查找函数名 */
    const char *name = lua_tostring(L, -1);  /* 将函数名转换为字符串 */
    if (strncmp(name, LUA_GNAME ".", 3) == 0) {  /* 判断函数名是否以'_G.'开头 */
      lua_pushstring(L, name + 3);  /* 将去除前缀后的函数名压入栈顶 */
      lua_remove(L, -2);  /* 移除原始函数名 */
    }
    lua_copy(L, -1, top + 1);  /* 将函数名复制到正确的位置 */
    lua_settop(L, top + 1);  /* 移除表"loaded"和函数名的副本 */
    return 1;  /* 返回1表示成功 */
  }
  else {
    lua_settop(L, top);  /* 移除函数和全局表 */
    return 0;  /* 返回0表示失败 */
  }
}


static void pushfuncname (lua_State *L, lua_Debug *ar) {
  if (pushglobalfuncname(L, ar)) {  /* 尝试获取全局函数名 */
    lua_pushfstring(L, "function '%s'", lua_tostring(L, -1));  /* 将函数名格式化并压入栈顶 */
    lua_remove(L, -2);  /* 移除函数名 */
  }
  else if (*ar->namewhat != '\0')  /* 是否有代码中的名称？ */
    lua_pushfstring(L, "%s '%s'", ar->namewhat, ar->name);  /* 使用代码中的名称 */
  else if (*ar->what == 'm')  /* 主函数？ */
      lua_pushliteral(L, "main chunk");  /* 压入"main chunk" */
  else if (*ar->what != 'C')  /* 对于 Lua 函数，使用<file:line>格式 */
    lua_pushfstring(L, "function <%s:%d>", ar->short_src, ar->linedefined);  /* 格式化并压入函数位置信息 */
  else  /* 没有其他信息了... */
    lua_pushliteral(L, "?");  /* 压入"?" */
}


static int lastlevel (lua_State *L) {
  lua_Debug ar;
  int li = 1, le = 1;
  /* 找到一个上限 */
  while (lua_getstack(L, le, &ar)) { li = le; le *= 2; }  /* 通过二进制搜索找到上限 */
  /* 进行二分搜索 */
  while (li < le) {
    int m = (li + le)/2;
    if (lua_getstack(L, m, &ar)) li = m + 1;
    else le = m;
  }
  return le - 1;  /* 返回上限减一 */
}


LUALIB_API void luaL_traceback (lua_State *L, lua_State *L1,
                                const char *msg, int level) {
  luaL_Buffer b;
  lua_Debug ar;
  int last = lastlevel(L1);  /* 获取调用栈的最大深度 */
  int limit2show = (last - level > LEVELS1 + LEVELS2) ? LEVELS1 : -1;  /* 计算要显示的调用栈层级 */
  luaL_buffinit(L, &b);  /* 初始化缓冲区 */
  if (msg) {
  # 将消息添加到缓冲区中
  luaL_addstring(&b, msg);
  # 将换行符添加到缓冲区中
  luaL_addchar(&b, '\n');
  # 添加"stack traceback:"到缓冲区中
  luaL_addstring(&b, "stack traceback:");
  # 循环获取调用栈信息
  while (lua_getstack(L1, level++, &ar)) {
    # 如果超出限制的调用栈层数
    if (limit2show-- == 0) {  
      # 计算需要跳过的层数
      int n = last - level - LEVELS2 + 1;  
      # 添加关于跳过层数的警告信息到缓冲区中
      lua_pushfstring(L, "\n\t...\t(skipping %d levels)", n);
      luaL_addvalue(&b);  
      # 跳过指定层数
      level += n;  
    }
    else {
      # 获取调用栈信息
      lua_getinfo(L1, "Slnt", &ar);
      # 如果当前行号小于等于0
      if (ar.currentline <= 0)
        # 添加源文件信息到缓冲区中
        lua_pushfstring(L, "\n\t%s: in ", ar.short_src);
      else
        # 添加源文件和行号信息到缓冲区中
        lua_pushfstring(L, "\n\t%s:%d: in ", ar.short_src, ar.currentline);
      luaL_addvalue(&b);
      # 添加函数名到缓冲区中
      pushfuncname(L, &ar);
      luaL_addvalue(&b);
      # 如果是尾调用，添加尾调用信息到缓冲区中
      if (ar.istailcall)
        luaL_addstring(&b, "\n\t(...tail calls...)");
    }
  }
  # 将缓冲区中的内容推入栈中
  luaL_pushresult(&b);
/* }====================================================== */
/* 结束符号，表示某一部分的结束 */

/*
** {======================================================
** Error-report functions
** =======================================================
*/
/* 错误报告函数 */

LUALIB_API int luaL_argerror (lua_State *L, int arg, const char *extramsg) {
  /* 报告参数错误，如果没有堆栈帧，则报错 */
  lua_Debug ar;
  if (!lua_getstack(L, 0, &ar))
    return luaL_error(L, "bad argument #%d (%s)", arg, extramsg);
  lua_getinfo(L, "n", &ar);
  if (strcmp(ar.namewhat, "method") == 0) {
    arg--;  /* 不计算 'self' 参数 */
    if (arg == 0)  /* 错误在 self 参数本身？ */
      return luaL_error(L, "calling '%s' on bad self (%s)",
                           ar.name, extramsg);
  }
  if (ar.name == NULL)
    ar.name = (pushglobalfuncname(L, &ar)) ? lua_tostring(L, -1) : "?";
  return luaL_error(L, "bad argument #%d to '%s' (%s)",
                        arg, ar.name, extramsg);
}

/* 报告类型错误 */
LUALIB_API int luaL_typeerror (lua_State *L, int arg, const char *tname) {
  const char *msg;
  const char *typearg;  /* 实际参数的类型名称 */
  if (luaL_getmetafield(L, arg, "__name") == LUA_TSTRING)
    typearg = lua_tostring(L, -1);  /* 使用给定的类型名称 */
  else if (lua_type(L, arg) == LUA_TLIGHTUSERDATA)
    typearg = "light userdata";  /* 特殊的消息类型名称 */
  else
    typearg = luaL_typename(L, arg);  /* 标准的类型名称 */
  msg = lua_pushfstring(L, "%s expected, got %s", tname, typearg);
  return luaL_argerror(L, arg, msg);
}

/* 标签错误 */
static void tag_error (lua_State *L, int arg, int tag) {
  luaL_typeerror(L, arg, lua_typename(L, tag));
}

/*
** 使用 'lua_pushfstring' 确保在调用时不需要保留堆栈空间。
*/
LUALIB_API void luaL_where (lua_State *L, int level) {
  lua_Debug ar;
  if (lua_getstack(L, level, &ar)) {  /* 检查指定层级的函数 */
    lua_getinfo(L, "Sl", &ar);  /* 获取有关函数的信息 */
    # 如果当前行号大于0，表示有信息
    if (ar.currentline > 0) {  /* is there info? */
      # 将文件名和行号格式化成字符串，压入栈中
      lua_pushfstring(L, "%s:%d: ", ar.short_src, ar.currentline);
      # 返回
      return;
    }
  }
  # 如果没有信息，则压入空字符串
  lua_pushfstring(L, "");  /* else, no information available... */
/*
** Again, the use of 'lua_pushvfstring' ensures this function does
** not need reserved stack space when called. (At worst, it generates
** an error with "stack overflow" instead of the given message.)
*/
LUALIB_API int luaL_error (lua_State *L, const char *fmt, ...) {
  va_list argp;  // 创建一个变量参数列表
  va_start(argp, fmt);  // 初始化变量参数列表
  luaL_where(L, 1);  // 获取错误位置信息
  lua_pushvfstring(L, fmt, argp);  // 将格式化的字符串推入栈中
  va_end(argp);  // 结束变量参数列表
  lua_concat(L, 2);  // 连接栈顶两个值
  return lua_error(L);  // 抛出 Lua 错误
}


LUALIB_API int luaL_fileresult (lua_State *L, int stat, const char *fname) {
  int en = errno;  /* calls to Lua API may change this value */  // 保存当前的错误码
  if (stat) {  // 如果状态为真
    lua_pushboolean(L, 1);  // 将布尔值推入栈中
    return 1;  // 返回 1
  }
  else {  // 否则
    luaL_pushfail(L);  // 将失败信息推入栈中
    if (fname)  // 如果文件名存在
      lua_pushfstring(L, "%s: %s", fname, strerror(en));  // 将格式化的字符串推入栈中
    else
      lua_pushstring(L, strerror(en));  // 将错误信息推入栈中
    lua_pushinteger(L, en);  // 将错误码推入栈中
    return 3;  // 返回 3
  }
}


#if !defined(l_inspectstat)    /* { */

#if defined(LUA_USE_POSIX)

#include <sys/wait.h>

/*
** use appropriate macros to interpret 'pclose' return status
*/
#define l_inspectstat(stat,what)  \  // 定义宏来解释 'pclose' 的返回状态
   if (WIFEXITED(stat)) { stat = WEXITSTATUS(stat); } \  // 如果正常退出，获取退出状态
   else if (WIFSIGNALED(stat)) { stat = WTERMSIG(stat); what = "signal"; }  // 如果被信号终止，获取信号值

#else

#define l_inspectstat(stat,what)  /* no op */  // 如果不是 POSIX 系统，不执行任何操作

#endif

#endif                /* } */


LUALIB_API int luaL_execresult (lua_State *L, int stat) {
  if (stat != 0 && errno != 0)  /* error with an 'errno'? */  // 如果状态不为 0 并且错误码不为 0
    return luaL_fileresult(L, 0, NULL);  // 返回文件操作结果
  else {
    const char *what = "exit";  /* type of termination */  // 终止类型
    l_inspectstat(stat, what);  /* interpret result */  // 解释结果
    if (*what == 'e' && stat == 0)  /* successful termination? */  // 如果是正常终止
      lua_pushboolean(L, 1);  // 将布尔值推入栈中
    else
      luaL_pushfail(L);  // 将失败信息推入栈中
    lua_pushstring(L, what);  // 将终止类型推入栈中
    lua_pushinteger(L, stat);  // 将状态值推入栈中
    return 3;  /* return true/fail,what,code */  // 返回 3
  }
}

/* }====================================================== */



/*
** {======================================================
** Userdata's metatable manipulation
/*
** =======================================================
*/

// 创建一个新的元表，如果元表名已经存在则返回 0
LUALIB_API int luaL_newmetatable (lua_State *L, const char *tname) {
  if (luaL_getmetatable(L, tname) != LUA_TNIL)  /* name already in use? */
    return 0;  /* leave previous value on top, but return 0 */
  lua_pop(L, 1);
  lua_createtable(L, 0, 2);  /* create metatable */
  lua_pushstring(L, tname);
  lua_setfield(L, -2, "__name");  /* metatable.__name = tname */
  lua_pushvalue(L, -1);
  lua_setfield(L, LUA_REGISTRYINDEX, tname);  /* registry.name = metatable */
  return 1;
}


// 设置指定类型的元表
LUALIB_API void luaL_setmetatable (lua_State *L, const char *tname) {
  luaL_getmetatable(L, tname);
  lua_setmetatable(L, -2);
}


// 检查用户数据类型是否符合指定的元表
LUALIB_API void *luaL_testudata (lua_State *L, int ud, const char *tname) {
  void *p = lua_touserdata(L, ud);
  if (p != NULL) {  /* value is a userdata? */
    if (lua_getmetatable(L, ud)) {  /* does it have a metatable? */
      luaL_getmetatable(L, tname);  /* get correct metatable */
      if (!lua_rawequal(L, -1, -2))  /* not the same? */
        p = NULL;  /* value is a userdata with wrong metatable */
      lua_pop(L, 2);  /* remove both metatables */
      return p;
    }
  }
  return NULL;  /* value is not a userdata with a metatable */
}


// 检查用户数据类型是否符合指定的元表，如果不符合则抛出错误
LUALIB_API void *luaL_checkudata (lua_State *L, int ud, const char *tname) {
  void *p = luaL_testudata(L, ud, tname);
  luaL_argexpected(L, p != NULL, ud, tname);
  return p;
}

/* }====================================================== */


/*
** {======================================================
** Argument check functions
** =======================================================
*/

// 检查参数是否在指定的选项列表中
LUALIB_API int luaL_checkoption (lua_State *L, int arg, const char *def,
                                 const char *const lst[]) {
  const char *name = (def) ? luaL_optstring(L, arg, def) :
                             luaL_checkstring(L, arg);
  int i;
  for (i=0; lst[i]; i++)
    # 如果列表中的第i个元素与给定的name相等，则返回i
    if (strcmp(lst[i], name) == 0)
      return i;
  # 如果未找到匹配的选项，则抛出Lua错误，指定参数错误，并推送格式化的错误消息到栈顶
  return luaL_argerror(L, arg,
                       lua_pushfstring(L, "invalid option '%s'", name));
/*
** 确保堆栈至少有 'space' 个额外的插槽，如果无法满足请求则引发错误
** （错误处理需要一些额外的插槽来格式化错误消息。如果发生错误而没有这些额外空间，Lua 将生成相同的 'stack overflow' 错误，但没有 'msg'。）
*/
LUALIB_API void luaL_checkstack (lua_State *L, int space, const char *msg) {
  if (l_unlikely(!lua_checkstack(L, space))) {
    if (msg)
      luaL_error(L, "stack overflow (%s)", msg);
    else
      luaL_error(L, "stack overflow");
  }
}


LUALIB_API void luaL_checktype (lua_State *L, int arg, int t) {
  if (l_unlikely(lua_type(L, arg) != t))
    tag_error(L, arg, t);
}


LUALIB_API void luaL_checkany (lua_State *L, int arg) {
  if (l_unlikely(lua_type(L, arg) == LUA_TNONE))
    luaL_argerror(L, arg, "value expected");
}


LUALIB_API const char *luaL_checklstring (lua_State *L, int arg, size_t *len) {
  const char *s = lua_tolstring(L, arg, len);
  if (l_unlikely(!s)) tag_error(L, arg, LUA_TSTRING);
  return s;
}


LUALIB_API const char *luaL_optlstring (lua_State *L, int arg,
                                        const char *def, size_t *len) {
  if (lua_isnoneornil(L, arg)) {
    if (len)
      *len = (def ? strlen(def) : 0);
    return def;
  }
  else return luaL_checklstring(L, arg, len);
}


LUALIB_API lua_Number luaL_checknumber (lua_State *L, int arg) {
  int isnum;
  lua_Number d = lua_tonumberx(L, arg, &isnum);
  if (l_unlikely(!isnum))
    tag_error(L, arg, LUA_TNUMBER);
  return d;
}


LUALIB_API lua_Number luaL_optnumber (lua_State *L, int arg, lua_Number def) {
  return luaL_opt(L, luaL_checknumber, arg, def);
}


static void interror (lua_State *L, int arg) {
  if (lua_isnumber(L, arg))
    luaL_argerror(L, arg, "number has no integer representation");
  else
    tag_error(L, arg, LUA_TNUMBER);
}
# 检查参数中的值是否为整数，如果是则返回整数值，否则抛出错误
LUALIB_API lua_Integer luaL_checkinteger (lua_State *L, int arg) {
  int isnum;
  lua_Integer d = lua_tointegerx(L, arg, &isnum);
  if (l_unlikely(!isnum)) {
    interror(L, arg);
  }
  return d;
}


# 检查参数中的值是否为整数，如果是则返回整数值，否则返回默认值
LUALIB_API lua_Integer luaL_optinteger (lua_State *L, int arg,
                                                      lua_Integer def) {
  return luaL_opt(L, luaL_checkinteger, arg, def);
}

/* }====================================================== */


'''
** {======================================================
** 通用缓冲区操作
** =======================================================
'''

# 封装任意数据的用户数据
typedef struct UBox {
  void *box;
  size_t bsize;
} UBox;


# 调整用户数据的大小
static void *resizebox (lua_State *L, int idx, size_t newsize) {
  void *ud;
  lua_Alloc allocf = lua_getallocf(L, &ud);
  UBox *box = (UBox *)lua_touserdata(L, idx);
  void *temp = allocf(ud, box->box, box->bsize, newsize);
  if (l_unlikely(temp == NULL && newsize > 0)) {  /* 分配错误？ */
    lua_pushliteral(L, "not enough memory");
    lua_error(L);  /* 抛出内存错误 */
  }
  box->box = temp;
  box->bsize = newsize;
  return temp;
}


# 用户数据的垃圾回收函数
static int boxgc (lua_State *L) {
  resizebox(L, 1, 0);
  return 0;
}


# 用户数据的元方法
static const luaL_Reg boxmt[] = {  /* box metamethods */
  {"__gc", boxgc},
  {"__close", boxgc},
  {NULL, NULL}
};


# 创建新的用户数据
static void newbox (lua_State *L) {
  UBox *box = (UBox *)lua_newuserdatauv(L, sizeof(UBox), 0);
  box->box = NULL;
  box->bsize = 0;
  if (luaL_newmetatable(L, "_UBOX*"))  /* 创建元表？ */
    luaL_setfuncs(L, boxmt, 0);  /* 设置元方法 */
  lua_setmetatable(L, -2);
}


# 检查缓冲区是否使用堆栈上的用户数据作为临时缓冲区
#define buffonstack(B)    ((B)->b != (B)->init.b)


# 每当访问缓冲区时，槽 'idx' 必须是一个用户数据（不能为NULL），或者是缓冲区的占位符
#define checkbufferlevel(B,idx)  \  # 定义宏，用于检查缓冲区的级别
  lua_assert(buffonstack(B) ? lua_touserdata(B->L, idx) != NULL  \  # 使用lua_assert宏检查缓冲区是否在堆栈上，如果是，则检查缓冲区中的数据是否为空
                            : lua_touserdata(B->L, idx) == (void*)B)  # 如果不在堆栈上，则检查缓冲区中的数据是否等于B


/*
** 计算缓冲区'B'的新大小，足够容纳额外的'sz'字节。
*/
static size_t newbuffsize (luaL_Buffer *B, size_t sz) {
  size_t newsize = B->size * 2;  /* 将缓冲区大小加倍 */
  if (l_unlikely(MAX_SIZET - sz < B->n))  /* (B->n + sz)是否会溢出？ */
    return luaL_error(B->L, "buffer too large");  # 如果溢出，则返回错误
  if (newsize < B->n + sz)  /* 加倍后仍然不够大？ */
    newsize = B->n + sz;  # 将新大小设置为(B->n + sz)
  return newsize;  # 返回新的缓冲区大小
}


/*
** 返回指向缓冲区'B'中至少有'sz'字节的空闲区域的指针。'boxidx'是堆栈中缓冲区的盒子或其占位符的相对位置。
*/
static char *prepbuffsize (luaL_Buffer *B, size_t sz, int boxidx) {
  checkbufferlevel(B, boxidx);  # 检查缓冲区级别
  if (B->size - B->n >= sz)  /* 空间足够？ */
    return B->b + B->n;  # 返回空闲区域的指针
  else {
    lua_State *L = B->L;
    char *newbuff;
    size_t newsize = newbuffsize(B, sz);  # 计算新的缓冲区大小
    /* 创建更大的缓冲区 */
    if (buffonstack(B))  /* 缓冲区已经有一个盒子？ */
      newbuff = (char *)resizebox(L, boxidx, newsize);  /* 调整大小 */
    else {  /* 还没有盒子 */
      lua_remove(L, boxidx);  /* 移除占位符 */
      newbox(L);  /* 创建一个新的盒子 */
      lua_insert(L, boxidx);  /* 将盒子移动到其预期位置 */
      lua_toclose(L, boxidx);
      newbuff = (char *)resizebox(L, boxidx, newsize);
      memcpy(newbuff, B->b, B->n * sizeof(char));  /* 复制原始内容 */
    }
    B->b = newbuff;
    B->size = newsize;
    return newbuff + B->n;
  }
}

/*
** 返回指向至少有'sz'字节的空闲区域的指针
*/
LUALIB_API char *luaL_prepbuffsize (luaL_Buffer *B, size_t sz) {
  return prepbuffsize(B, sz, -1);  # 返回指向空闲区域的指针
}


LUALIB_API void luaL_addlstring (luaL_Buffer *B, const char *s, size_t l) {
  if (l > 0) {  /* 当's'可能为NULL时，避免'memcpy' */
    char *b = prepbuffsize(B, l, -1);  # 返回指向空闲区域的指针
    # 将源地址 s 指向的数据拷贝到目标地址 b 指向的位置，拷贝长度为 l 个字符的大小
    memcpy(b, s, l * sizeof(char));
    # 将 l 个字符的大小添加到 luaL_Buffer 对象 B 中
    luaL_addsize(B, l);
  }
/* }====================================================== */

LUALIB_API void luaL_addstring (luaL_Buffer *B, const char *s) {
  // 将字符串 s 添加到缓冲区 B 中
  luaL_addlstring(B, s, strlen(s));
}

LUALIB_API void luaL_pushresult (luaL_Buffer *B) {
  // 获取 Lua 状态机
  lua_State *L = B->L;
  // 检查缓冲区 B 的级别
  checkbufferlevel(B, -1);
  // 将缓冲区 B 中的内容作为字符串推入栈中
  lua_pushlstring(L, B->b, B->n);
  // 如果缓冲区 B 在栈上
  if (buffonstack(B))
    // 关闭盒子
    lua_closeslot(L, -2);
  // 从栈中移除盒子或占位符
  lua_remove(L, -2);
}

LUALIB_API void luaL_pushresultsize (luaL_Buffer *B, size_t sz) {
  // 向缓冲区 B 中添加指定大小的内容
  luaL_addsize(B, sz);
  // 将缓冲区 B 中的内容作为字符串推入栈中
  luaL_pushresult(B);
}

/*
** 'luaL_addvalue' is the only function in the Buffer system where the
** box (if existent) is not on the top of the stack. So, instead of
** calling 'luaL_addlstring', it replicates the code using -2 as the
** last argument to 'prepbuffsize', signaling that the box is (or will
** be) bellow the string being added to the buffer. (Box creation can
** trigger an emergency GC, so we should not remove the string from the
** stack before we have the space guaranteed.)
*/
LUALIB_API void luaL_addvalue (luaL_Buffer *B) {
  // 获取 Lua 状态机
  lua_State *L = B->L;
  // 获取栈顶字符串的长度和内容
  size_t len;
  const char *s = lua_tolstring(L, -1, &len);
  // 准备缓冲区的大小
  char *b = prepbuffsize(B, len, -2);
  // 将字符串复制到缓冲区
  memcpy(b, s, len * sizeof(char));
  // 向缓冲区 B 中添加指定大小的内容
  luaL_addsize(B, len);
  // 从栈中弹出字符串
  lua_pop(L, 1);
}

LUALIB_API void luaL_buffinit (lua_State *L, luaL_Buffer *B) {
  // 设置缓冲区 B 的 Lua 状态机
  B->L = L;
  // 设置缓冲区 B 的内容
  B->b = B->init.b;
  // 设置缓冲区 B 的大小
  B->n = 0;
  // 设置缓冲区 B 的初始大小
  B->size = LUAL_BUFFERSIZE;
  // 将缓冲区 B 的占位符推入栈中
  lua_pushlightuserdata(L, (void*)B);
}

LUALIB_API char *luaL_buffinitsize (lua_State *L, luaL_Buffer *B, size_t sz) {
  // 初始化缓冲区 B
  luaL_buffinit(L, B);
  // 准备缓冲区的大小
  return prepbuffsize(B, sz, -1);
}

/* }====================================================== */

/*
** {======================================================
** Reference system
** =======================================================
*/

/* index of free-list header (after the predefined values) */
// 自由列表头的索引（在预定义值之后）
#define freelist    (LUA_RIDX_LAST + 1)

/*
** The previously freed references form a linked list:
** 先前释放的引用形成一个链表：
# t[freelist] 是第一个空闲索引的索引，如果列表为空则为零；t[t[freelist]] 是第二个元素的索引，依此类推。
LUALIB_API int luaL_ref (lua_State *L, int t) {
  int ref;
  if (lua_isnil(L, -1)) {
    lua_pop(L, 1);  # 从堆栈中移除
    return LUA_REFNIL;  # 'nil' 有一个唯一的固定引用
  }
  t = lua_absindex(L, t);
  if (lua_rawgeti(L, t, freelist) == LUA_TNIL) {  # 第一次访问？
    ref = 0;  # 列表为空
    lua_pushinteger(L, 0);  # 初始化为空列表
    lua_rawseti(L, t, freelist);  # ref = t[freelist] = 0
  }
  else {  # 已经初始化
    lua_assert(lua_isinteger(L, -1));
    ref = (int)lua_tointeger(L, -1);  # ref = t[freelist]
  }
  lua_pop(L, 1);  # 从堆栈中移除元素
  if (ref != 0) {  # 有空闲元素？
    lua_rawgeti(L, t, ref);  # 从列表中移除
    lua_rawseti(L, t, freelist);  # (t[freelist] = t[ref])
  }
  else:  # 没有空闲元素
    ref = (int)lua_rawlen(L, t) + 1;  # 获取一个新的引用
  lua_rawseti(L, t, ref);
  return ref;
}

LUALIB_API void luaL_unref (lua_State *L, int t, int ref) {
  if (ref >= 0) {
    t = lua_absindex(L, t);
    lua_rawgeti(L, t, freelist);
    lua_assert(lua_isinteger(L, -1));
    lua_rawseti(L, t, ref);  # t[ref] = t[freelist]
    lua_pushinteger(L, ref);
    lua_rawseti(L, t, freelist);  # t[freelist] = ref
  }
}

# }======================================================

# {======================================================
# 加载函数
# =======================================================
typedef struct LoadF {
  int n;  # 预读字符的数量
  FILE *f;  # 正在读取的文件
  char buff[BUFSIZ];  # 用于读取文件的区域
} LoadF;

static const char *getF (lua_State *L, void *ud, size_t *size) {
  LoadF *lf = (LoadF *)ud;
  (void)L;  # 未使用
  if (lf->n > 0) {  # 是否有预读字符需要读取？
    *size = lf->n;  /* 将lf->n的值赋给size，即返回已经在缓冲区中的字符数量 */
    lf->n = 0;  /* 清空预读字符数量，表示没有预读字符了 */
  }
  else {  /* 从文件中读取一个块 */
    /* 'fread' 可能返回 > 0 且设置了 EOF 标志。如果下一次 'getF' 调用 'fread'，它可能仍然等待用户输入。
       下一个检查避免了这个问题。 */
    if (feof(lf->f)) return NULL;  /* 如果文件结束，返回空指针 */
    *size = fread(lf->buff, 1, sizeof(lf->buff), lf->f);  /* 读取一个块 */
  }
  return lf->buff;  /* 返回读取的数据块 */
  // 如果发生错误，返回错误信息
static int errfile (lua_State *L, const char *what, int fnameindex) {
  const char *serr = strerror(errno);  // 获取错误信息
  const char *filename = lua_tostring(L, fnameindex) + 1;  // 获取文件名
  lua_pushfstring(L, "cannot %s %s: %s", what, filename, serr);  // 将错误信息推入栈
  lua_remove(L, fnameindex);  // 移除文件名
  return LUA_ERRFILE;  // 返回文件错误
}

// 跳过文件的 UTF-8 BOM 标记
static int skipBOM (LoadF *lf) {
  const char *p = "\xEF\xBB\xBF";  /* UTF-8 BOM mark */
  int c;
  lf->n = 0;
  do {
    c = getc(lf->f);  // 读取文件字符
    if (c == EOF || c != *(const unsigned char *)p++) return c;  // 如果到达文件末尾或者不是 BOM 标记，则返回字符
    lf->buff[lf->n++] = c;  /* to be read by the parser */  // 将字符存入缓冲区
  } while (*p != '\0');
  lf->n = 0;  /* prefix matched; discard it */  // 匹配前缀，丢弃它
  return getc(lf->f);  /* return next character */  // 返回下一个字符
}

/*
** 读取文件 'f' 的第一个字符，并跳过开头的可选 BOM 标记以及如果以 '#' 开头的第一行。如果跳过了第一行，则返回 true。无论如何，'*cp' 都有文件的第一个“有效”字符（在可选 BOM 和第一行注释之后）。
*/
static int skipcomment (LoadF *lf, int *cp) {
  int c = *cp = skipBOM(lf);  // 跳过 BOM 标记
  if (c == '#') {  /* 第一行是注释（Unix 可执行文件）？ */
    do {  /* 跳过第一行 */
      c = getc(lf->f);  // 读取字符
    } while (c != EOF && c != '\n');  // 直到文件末尾或者换行符
    *cp = getc(lf->f);  /* 跳过换行符，如果存在 */
    return 1;  /* 存在注释 */
  }
  else return 0;  /* 不存在注释 */
}

// 从文件加载 Lua 代码
LUALIB_API int luaL_loadfilex (lua_State *L, const char *filename,
                                             const char *mode) {
  LoadF lf;
  int status, readstatus;
  int c;
  int fnameindex = lua_gettop(L) + 1;  /* 文件名在栈中的索引 */
  if (filename == NULL) {
    lua_pushliteral(L, "=stdin");  // 将 "=stdin" 推入栈
    lf.f = stdin;  // 设置文件指针为标准输入
  }
  else {
    lua_pushfstring(L, "@%s", filename);  // 将文件名推入栈
    lf.f = fopen(filename, "r");  // 以只读方式打开文件
    if (lf.f == NULL) return errfile(L, "open", fnameindex);  // 如果打开文件失败，返回错误信息
  }
  if (skipcomment(&lf, &c))  /* 读取初始部分 */
    lf.buff[lf.n++] = '\n';  /* 将换行符添加到正确的行号 */
  if (c == LUA_SIGNATURE[0] && filename) {  /* 是否为二进制文件？ */
    lf.f = freopen(filename, "rb", lf.f);  /* 以二进制模式重新打开文件 */
    if (lf.f == NULL) return errfile(L, "reopen", fnameindex);
    skipcomment(&lf, &c);  /* 重新读取初始部分 */
  }
  if (c != EOF)
    lf.buff[lf.n++] = c;  /* 'c' 是流的第一个字符 */
  status = lua_load(L, getF, &lf, lua_tostring(L, -1), mode);
  readstatus = ferror(lf.f);
  if (filename) fclose(lf.f);  /* 关闭文件（即使出现错误） */
  if (readstatus) {
    lua_settop(L, fnameindex);  /* 忽略 'lua_load' 的结果 */
    return errfile(L, "read", fnameindex);
  }
  lua_remove(L, fnameindex);
  return status;
/* 结构体定义，用于加载字符串 */
typedef struct LoadS {
  const char *s;  // 字符串指针
  size_t size;    // 字符串大小
} LoadS;


/* 从自定义数据源中获取字符串 */
static const char *getS (lua_State *L, void *ud, size_t *size) {
  LoadS *ls = (LoadS *)ud;  // 将自定义数据源转换为LoadS结构体指针
  (void)L;  /* not used */  // 参数未使用
  if (ls->size == 0) return NULL;  // 如果字符串大小为0，返回空指针
  *size = ls->size;  // 设置字符串大小
  ls->size = 0;  // 将字符串大小置为0
  return ls->s;  // 返回字符串指针
}


/* 加载缓冲区中的代码，并将其作为函数压入栈中 */
LUALIB_API int luaL_loadbufferx (lua_State *L, const char *buff, size_t size,
                                 const char *name, const char *mode) {
  LoadS ls;  // 创建LoadS结构体对象
  ls.s = buff;  // 设置LoadS结构体对象的字符串指针
  ls.size = size;  // 设置LoadS结构体对象的字符串大小
  return lua_load(L, getS, &ls, name, mode);  // 调用lua_load函数加载代码
}


/* 加载字符串中的代码，并将其作为函数压入栈中 */
LUALIB_API int luaL_loadstring (lua_State *L, const char *s) {
  return luaL_loadbuffer(L, s, strlen(s), s);  // 调用luaL_loadbuffer函数加载字符串中的代码
}

/* }====================================================== */


/* 获取对象的元表字段 */
LUALIB_API int luaL_getmetafield (lua_State *L, int obj, const char *event) {
  if (!lua_getmetatable(L, obj))  /* no metatable? */  // 如果没有元表，返回LUA_TNIL
    return LUA_TNIL;
  else {
    int tt;
    lua_pushstring(L, event);  // 将事件字符串压入栈中
    tt = lua_rawget(L, -2);  // 获取元表中的字段
    if (tt == LUA_TNIL)  /* is metafield nil? */  // 如果元表字段为nil
      lua_pop(L, 2);  /* remove metatable and metafield */  // 移除元表和元表字段
    else
      lua_remove(L, -2);  /* remove only metatable */  // 只移除元表
    return tt;  /* return metafield type */  // 返回元表字段类型
  }
}


/* 调用对象的元方法 */
LUALIB_API int luaL_callmeta (lua_State *L, int obj, const char *event) {
  obj = lua_absindex(L, obj);  // 获取绝对索引
  if (luaL_getmetafield(L, obj, event) == LUA_TNIL)  /* no metafield? */  // 如果没有元字段
    return 0;  // 返回0
  lua_pushvalue(L, obj);  // 将对象压入栈中
  lua_call(L, 1, 1);  // 调用函数
  return 1;  // 返回1
}


/* 获取对象的长度 */
LUALIB_API lua_Integer luaL_len (lua_State *L, int idx) {
  lua_Integer l;  // 定义整数变量
  int isnum;  // 定义整数标志
  lua_len(L, idx);  // 获取对象的长度
  l = lua_tointegerx(L, -1, &isnum);  // 将栈顶的值转换为整数
  if (l_unlikely(!isnum))  // 如果不是整数
    luaL_error(L, "object length is not an integer");  // 抛出错误
  lua_pop(L, 1);  /* remove object */  // 移除对象
  return l;  // 返回长度
}


/* 将对象转换为字符串 */
LUALIB_API const char *luaL_tolstring (lua_State *L, int idx, size_t *len) {
  idx = lua_absindex(L,idx);  // 获取绝对索引
  if (luaL_callmeta(L, idx, "__tostring")) {  /* metafield? */  // 调用元方法
    if (!lua_isstring(L, -1))  // 如果不是字符串
      luaL_error(L, "'__tostring' must return a string");  // 抛出错误
  }
  else {
    # 根据 Lua 值的类型进行不同的处理
    switch (lua_type(L, idx)) {
      # 如果是数字类型
      case LUA_TNUMBER: {
        # 如果是整数，将其格式化为整数字符串
        if (lua_isinteger(L, idx))
          lua_pushfstring(L, "%I", (LUAI_UACINT)lua_tointeger(L, idx));
        # 如果是浮点数，将其格式化为浮点数字符串
        else
          lua_pushfstring(L, "%f", (LUAI_UACNUMBER)lua_tonumber(L, idx));
        break;
      }
      # 如果是字符串类型
      case LUA_TSTRING:
        # 将字符串值推入栈顶
        lua_pushvalue(L, idx);
        break;
      # 如果是布尔类型
      case LUA_TBOOLEAN:
        # 将布尔值转换为字符串推入栈顶
        lua_pushstring(L, (lua_toboolean(L, idx) ? "true" : "false"));
        break;
      # 如果是空值类型
      case LUA_TNIL:
        # 将字符串 "nil" 推入栈顶
        lua_pushliteral(L, "nil");
        break;
      # 其他类型
      default: {
        # 尝试获取元表中的 "__name" 字段
        int tt = luaL_getmetafield(L, idx, "__name");  /* try name */
        # 如果存在 "__name" 字段，获取其值作为类型名称，否则使用默认类型名称
        const char *kind = (tt == LUA_TSTRING) ? lua_tostring(L, -1) :
                                                 luaL_typename(L, idx);
        # 将类型名称和指针地址格式化为字符串推入栈顶
        lua_pushfstring(L, "%s: %p", kind, lua_topointer(L, idx));
        # 如果存在 "__name" 字段，移除栈顶的类型名称
        if (tt != LUA_TNIL)
          lua_remove(L, -2);  /* remove '__name' */
        break;
      }
    }
  }
  # 将栈顶的值转换为字符串并返回
  return lua_tolstring(L, -1, len);
# 从列表 'l' 中的函数设置到栈顶的表中，每个函数都会获取栈顶的 'nup' 个元素作为 upvalues
# 返回时栈中只剩下表
LUALIB_API void luaL_setfuncs (lua_State *L, const luaL_Reg *l, int nup) {
  # 检查栈是否有足够的空间来存放 upvalues
  luaL_checkstack(L, nup, "too many upvalues");
  # 遍历函数列表，将函数填充到给定的表中
  for (; l->name != NULL; l++) {
    # 如果是占位符，则压入一个布尔值
    if (l->func == NULL)
      lua_pushboolean(L, 0);
    else {
      int i;
      # 将 upvalues 复制到栈顶
      for (i = 0; i < nup; i++)
        lua_pushvalue(L, -nup);
      # 创建一个闭包，带有这些 upvalues
      lua_pushcclosure(L, l->func, nup);
    }
    # 设置表中的字段
    lua_setfield(L, -(nup + 2), l->name);
  }
  # 移除 upvalues
  lua_pop(L, nup);
}


# 确保栈中的 stack[idx][fname] 是一个表，并将该表压入栈中
LUALIB_API int luaL_getsubtable (lua_State *L, int idx, const char *fname) {
  # 如果栈中的字段是一个表，则返回 1
  if (lua_getfield(L, idx, fname) == LUA_TTABLE)
    return 1;  # 表已经存在
  else {
    lua_pop(L, 1);  # 移除之前的结果
    idx = lua_absindex(L, idx);
    lua_newtable(L);
    lua_pushvalue(L, -1);  # 复制到栈顶
    lua_setfield(L, idx, fname);  # 将新表分配给字段
    return 0;  # 返回 false，因为没有在那里找到表
  }
}


# 精简的 'require'：在检查 "loaded" 表之后，调用 'openf' 打开一个模块，将结果注册到 'package.loaded' 表中，
# 如果 'glb' 为 true，还将结果注册到全局表中。将生成的模块留在栈顶
LUALIB_API void luaL_requiref (lua_State *L, const char *modname,
                               lua_CFunction openf, int glb) {
  luaL_getsubtable(L, LUA_REGISTRYINDEX, LUA_LOADED_TABLE);
  lua_getfield(L, -1, modname);  # LOADED[modname]
  if (!lua_toboolean(L, -1)) {  # 包未加载？
    lua_pop(L, 1);  # 移除字段
    lua_pushcfunction(L, openf);
  }
}
    # 将 modname 压入 Lua 栈中，作为打开函数的参数
    lua_pushstring(L, modname);  /* argument to open function */
    # 调用 'openf' 函数来打开模块
    lua_call(L, 1, 1);  /* call 'openf' to open module */
    # 复制模块（调用结果）并压入 Lua 栈中
    lua_pushvalue(L, -1);  /* make copy of module (call result) */
    # 设置 LOADED[modname] = module
    lua_setfield(L, -3, modname);  /* LOADED[modname] = module */
  }
  # 从 Lua 栈中移除 LOADED 表
  lua_remove(L, -2);  /* remove LOADED table */
  # 如果 glb 为真
  if (glb) {
    # 复制模块并压入 Lua 栈中
    lua_pushvalue(L, -1);  /* copy of module */
    # 设置 _G[modname] = module
    lua_setglobal(L, modname);  /* _G[modname] = module */
  }
}

LUALIB_API void luaL_addgsub (luaL_Buffer *b, const char *s,
                                     const char *p, const char *r) {
  const char *wild;
  size_t l = strlen(p);
  while ((wild = strstr(s, p)) != NULL) {
    luaL_addlstring(b, s, wild - s);  /* 将前缀推入缓冲区 */
    luaL_addstring(b, r);  /* 将替换内容推入缓冲区以替换模式 */
    s = wild + l;  /* 继续处理 'p' 之后的内容 */
  }
  luaL_addstring(b, s);  /* 将最后的后缀推入缓冲区 */
}

LUALIB_API const char *luaL_gsub (lua_State *L, const char *s,
                                  const char *p, const char *r) {
  luaL_Buffer b;
  luaL_buffinit(L, &b);
  luaL_addgsub(&b, s, p, r);
  luaL_pushresult(&b);
  return lua_tostring(L, -1);
}

static void *l_alloc (void *ud, void *ptr, size_t osize, size_t nsize) {
  (void)ud; (void)osize;  /* 未使用 */
  if (nsize == 0) {
    free(ptr);
    return NULL;
  }
  else
    return realloc(ptr, nsize);
}

static int panic (lua_State *L) {
  const char *msg = lua_tostring(L, -1);
  if (msg == NULL) msg = "error object is not a string";
  lua_writestringerror("PANIC: unprotected error in call to Lua API (%s)\n",
                        msg);
  return 0;  /* 返回 Lua 以中止 */
}

/*
** 警告函数:
** warnfoff: 警告系统关闭
** warnfon: 准备开始新消息
** warnfcont: 上一条消息将继续
*/
static void warnfoff (void *ud, const char *message, int tocont);
static void warnfon (void *ud, const char *message, int tocont);
static void warnfcont (void *ud, const char *message, int tocont);

/*
** 检查消息是否为控制消息。如果是，执行控制，如果未知则忽略。
*/
static int checkcontrol (lua_State *L, const char *message, int tocont) {
  if (tocont || *(message++) != '@')  /* 不是控制消息? */
    return 0;
  else {
    if (strcmp(message, "off") == 0)
      lua_setwarnf(L, warnfoff, L);  /* 关闭警告 */
    else if (strcmp(message, "on") == 0)
      lua_setwarnf(L, warnfon, L);   /* 如果消息是"on"，则打开警告 */
    return 1;  /* 这是一个控制消息 */
  }
/* 
** 写入消息并处理'tocont'，如果需要的话完成消息并设置下一个警告函数。
*/
static void warnfcont (void *ud, const char *message, int tocont) {
  lua_State *L = (lua_State *)ud;
  lua_writestringerror("%s", message);  /* 写入消息 */
  if (tocont)  /* 不是最后一部分？ */
    lua_setwarnf(L, warnfcont, L);  /* 继续 */
  else {  /* 最后一部分 */
    lua_writestringerror("%s", "\n");  /* 以换行符结束消息 */
    lua_setwarnf(L, warnfon, L);  /* 下一次调用是一个新消息 */
  }
}

/*
** 警告函数，关闭警告
*/
static void warnfoff (void *ud, const char *message, int tocont) {
  checkcontrol((lua_State *)ud, message, tocont);
}

/*
** 警告函数，打开警告
*/
static void warnfon (void *ud, const char *message, int tocont) {
  if (checkcontrol((lua_State *)ud, message, tocont))  /* 控制消息？ */
    return;  /* 没有其他事情要做 */
  lua_writestringerror("%s", "Lua warning: ");  /* 开始一个新的警告 */
  warnfcont(ud, message, tocont);  /* 完成处理 */
}

/*
** 创建一个新的 Lua 状态机
*/
LUALIB_API lua_State *luaL_newstate (void) {
  lua_State *L = lua_newstate(l_alloc, NULL);
  if (l_likely(L)) {
    lua_atpanic(L, &panic);
    lua_setwarnf(L, warnfoff, L);  /* 默认关闭警告 */
  }
  return L;
}

/*
** 检查 Lua 版本
*/
LUALIB_API void luaL_checkversion_ (lua_State *L, lua_Number ver, size_t sz) {
  lua_Number v = lua_version(L);
  if (sz != LUAL_NUMSIZES)  /* 检查数字类型 */
    luaL_error(L, "core and library have incompatible numeric types");
  else if (v != ver)
    luaL_error(L, "version mismatch: app. needs %f, Lua core provides %f",
                  (LUAI_UACNUMBER)ver, (LUAI_UACNUMBER)v);
}
```