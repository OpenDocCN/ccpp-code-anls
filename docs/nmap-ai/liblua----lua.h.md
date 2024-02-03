# `nmap\liblua\lua.h`

```cpp
/*
** $Id: lua.h $
** Lua - A Scripting Language
** Lua.org, PUC-Rio, Brazil (http://www.lua.org)
** See Copyright Notice at the end of this file
*/

#ifndef lua_h
#define lua_h

#include <stdarg.h>
#include <stddef.h>

#include "luaconf.h"

#define LUA_VERSION_MAJOR    "5"  // 定义 Lua 主版本号
#define LUA_VERSION_MINOR    "4"  // 定义 Lua 次版本号
#define LUA_VERSION_RELEASE    "4"  // 定义 Lua 发布版本号

#define LUA_VERSION_NUM            504  // 定义 Lua 版本号
#define LUA_VERSION_RELEASE_NUM        (LUA_VERSION_NUM * 100 + 4)  // 定义 Lua 发布版本号

#define LUA_VERSION    "Lua " LUA_VERSION_MAJOR "." LUA_VERSION_MINOR  // 定义 Lua 版本信息
#define LUA_RELEASE    LUA_VERSION "." LUA_VERSION_RELEASE  // 定义 Lua 发布信息
#define LUA_COPYRIGHT    LUA_RELEASE "  Copyright (C) 1994-2022 Lua.org, PUC-Rio"  // 定义 Lua 版权信息
#define LUA_AUTHORS    "R. Ierusalimschy, L. H. de Figueiredo, W. Celes"  // 定义 Lua 作者信息

/* mark for precompiled code ('<esc>Lua') */
#define LUA_SIGNATURE    "\x1bLua"  // 用于预编译代码的标记

/* option for multiple returns in 'lua_pcall' and 'lua_call' */
#define LUA_MULTRET    (-1)  // 'lua_pcall' 和 'lua_call' 中多返回值的选项

/*
** Pseudo-indices
** (-LUAI_MAXSTACK is the minimum valid index; we keep some free empty
** space after that to help overflow detection)
*/
#define LUA_REGISTRYINDEX    (-LUAI_MAXSTACK - 1000)  // 伪索引，用于访问注册表
#define lua_upvalueindex(i)    (LUA_REGISTRYINDEX - (i))  // 用于访问上值的伪索引

/* thread status */
#define LUA_OK        0  // 线程状态：成功
#define LUA_YIELD    1  // 线程状态：挂起
#define LUA_ERRRUN    2  // 线程状态：运行错误
#define LUA_ERRSYNTAX    3  // 线程状态：语法错误
#define LUA_ERRMEM    4  // 线程状态：内存错误
#define LUA_ERRERR    5  // 线程状态：错误处理错误

typedef struct lua_State lua_State;  // 定义 Lua 状态结构体

/*
** basic types
*/
#define LUA_TNONE        (-1)  // 基本类型：无

#define LUA_TNIL        0  // 基本类型：nil
#define LUA_TBOOLEAN        1  // 基本类型：布尔值
#define LUA_TLIGHTUSERDATA    2  // 基本类型：轻量用户数据
#define LUA_TNUMBER        3  // 基本类型：数字
#define LUA_TSTRING        4  // 基本类型：字符串
#define LUA_TTABLE        5  // 基本类型：表
#define LUA_TFUNCTION        6  // 基本类型：函数
#define LUA_TUSERDATA        7  // 基本类型：用户数据
#define LUA_TTHREAD        8  // 基本类型：线程

#define LUA_NUMTYPES        9  // 基本类型数量

/* minimum Lua stack available to a C function */
#define LUA_MINSTACK    20  // C 函数可用的最小 Lua 栈空间

/* predefined values in the registry */
#define LUA_RIDX_MAINTHREAD    1  // 注册表中预定义值：主线程
#define LUA_RIDX_GLOBALS    2  // 注册表中预定义值：全局变量
#define LUA_RIDX_LAST        LUA_RIDX_GLOBALS  // 注册表中预定义值：最后一个

/* type of numbers in Lua */
# 定义 Lua 中的数字类型
typedef LUA_NUMBER lua_Number;

# 定义 Lua 中的整数函数类型
typedef LUA_INTEGER lua_Integer;

# 定义 Lua 中的无符号整数类型
typedef LUA_UNSIGNED lua_Unsigned;

# 定义 Lua 中的续传函数上下文类型
typedef LUA_KCONTEXT lua_KContext;

# 定义 Lua 中注册的 C 函数类型
typedef int (*lua_CFunction) (lua_State *L);

# 定义 Lua 中的续传函数类型
typedef int (*lua_KFunction) (lua_State *L, int status, lua_KContext ctx);

# 定义 Lua 中用于加载/转储 Lua 代码块时读取/写入块的函数类型
typedef const char * (*lua_Reader) (lua_State *L, void *ud, size_t *sz);
typedef int (*lua_Writer) (lua_State *L, const void *p, size_t sz, void *ud);

# 定义 Lua 中的内存分配函数类型
typedef void * (*lua_Alloc) (void *ud, void *ptr, size_t osize, size_t nsize);

# 定义 Lua 中的警告函数类型
typedef void (*lua_WarnFunction) (void *ud, const char *msg, int tocont);

# 如果定义了 LUA_USER_H，则包含额外的用户头文件
#if defined(LUA_USER_H)
#include LUA_USER_H
#endif

# 定义 RCS 标识字符串
extern const char lua_ident[];

# Lua 状态操作
LUA_API lua_State *(lua_newstate) (lua_Alloc f, void *ud);
LUA_API void       (lua_close) (lua_State *L);
LUA_API lua_State *(lua_newthread) (lua_State *L);
LUA_API int        (lua_resetthread) (lua_State *L);
LUA_API lua_CFunction (lua_atpanic) (lua_State *L, lua_CFunction panicf);
LUA_API lua_Number (lua_version) (lua_State *L);

# 基本的堆栈操作
LUA_API int   (lua_absindex) (lua_State *L, int idx);
LUA_API int   (lua_gettop) (lua_State *L);
LUA_API void  (lua_settop) (lua_State *L, int idx);
LUA_API void  (lua_pushvalue) (lua_State *L, int idx);
LUA_API void  (lua_rotate) (lua_State *L, int idx, int n);
LUA_API void  (lua_copy) (lua_State *L, int fromidx, int toidx);
LUA_API int   (lua_checkstack) (lua_State *L, int n);
LUA_API void  (lua_xmove) (lua_State *from, lua_State *to, int n);
# 检查栈上指定索引处的值是否为数字
LUA_API int             (lua_isnumber) (lua_State *L, int idx);
# 检查栈上指定索引处的值是否为字符串
LUA_API int             (lua_isstring) (lua_State *L, int idx);
# 检查栈上指定索引处的值是否为 C 函数
LUA_API int             (lua_iscfunction) (lua_State *L, int idx);
# 检查栈上指定索引处的值是否为整数
LUA_API int             (lua_isinteger) (lua_State *L, int idx);
# 检查栈上指定索引处的值是否为用户数据
LUA_API int             (lua_isuserdata) (lua_State *L, int idx);
# 返回栈上指定索引处的值的类型
LUA_API int             (lua_type) (lua_State *L, int idx);
# 返回指定类型的类型名
LUA_API const char     *(lua_typename) (lua_State *L, int tp);

# 将栈上指定索引处的值转换为 lua_Number 类型的数值
LUA_API lua_Number      (lua_tonumberx) (lua_State *L, int idx, int *isnum);
# 将栈上指定索引处的值转换为 lua_Integer 类型的整数
LUA_API lua_Integer     (lua_tointegerx) (lua_State *L, int idx, int *isnum);
# 将栈上指定索引处的值转换为布尔值
LUA_API int             (lua_toboolean) (lua_State *L, int idx);
# 将栈上指定索引处的值转换为字符串
LUA_API const char     *(lua_tolstring) (lua_State *L, int idx, size_t *len);
# 返回栈上指定索引处的值的长度
LUA_API lua_Unsigned    (lua_rawlen) (lua_State *L, int idx);
# 将栈上指定索引处的值转换为 C 函数
LUA_API lua_CFunction   (lua_tocfunction) (lua_State *L, int idx);
# 将栈上指定索引处的值转换为用户数据
LUA_API void           *(lua_touserdata) (lua_State *L, int idx);
# 将栈上指定索引处的值转换为线程
LUA_API lua_State      *(lua_tothread) (lua_State *L, int idx);
# 将栈上指定索引处的值转换为指针
LUA_API const void     *(lua_topointer) (lua_State *L, int idx);

/*
** Comparison and arithmetic functions
*/

# 定义加法操作的标识
#define LUA_OPADD    0    /* ORDER TM, ORDER OP */
# 定义减法操作的标识
#define LUA_OPSUB    1
# 定义乘法操作的标识
#define LUA_OPMUL    2
# 定义取模操作的标识
#define LUA_OPMOD    3
# 定义乘方操作的标识
#define LUA_OPPOW    4
# 定义除法操作的标识
#define LUA_OPDIV    5
# 定义整数除法操作的标识
#define LUA_OPIDIV    6
# 定义按位与操作的标识
#define LUA_OPBAND    7
# 定义按位或操作的标识
#define LUA_OPBOR    8
# 定义按位异或操作的标识
#define LUA_OPBXOR    9
# 定义左移操作的标识
#define LUA_OPSHL    10
# 定义右移操作的标识
#define LUA_OPSHR    11
# 定义取负操作的标识
#define LUA_OPUNM    12
# 定义按位取反操作的标识
#define LUA_OPBNOT    13

# 执行指定操作
LUA_API void  (lua_arith) (lua_State *L, int op);

# 定义相等比较操作的标识
#define LUA_OPEQ    0
# 定义小于比较操作的标识
#define LUA_OPLT    1
# 定义小于等于比较操作的标识
#define LUA_OPLE    2

# 比较两个值是否相等
LUA_API int   (lua_rawequal) (lua_State *L, int idx1, int idx2);
# 比较两个值的大小
LUA_API int   (lua_compare) (lua_State *L, int idx1, int idx2, int op);

/*
** push functions (C -> stack)
*/
# 将空值推入栈中
LUA_API void        (lua_pushnil) (lua_State *L);
# 将数字推入栈中
LUA_API void        (lua_pushnumber) (lua_State *L, lua_Number n);
# 将整数推入栈中
LUA_API void        (lua_pushinteger) (lua_State *L, lua_Integer n);
# 将指定长度的字符串压入 Lua 栈
LUA_API const char *(lua_pushlstring) (lua_State *L, const char *s, size_t len);
# 将以空字符结尾的字符串压入 Lua 栈
LUA_API const char *(lua_pushstring) (lua_State *L, const char *s);
# 将格式化字符串压入 Lua 栈
LUA_API const char *(lua_pushvfstring) (lua_State *L, const char *fmt, va_list argp);
# 将格式化字符串压入 Lua 栈
LUA_API const char *(lua_pushfstring) (lua_State *L, const char *fmt, ...);
# 将闭包函数压入 Lua 栈
LUA_API void  (lua_pushcclosure) (lua_State *L, lua_CFunction fn, int n);
# 将布尔值压入 Lua 栈
LUA_API void  (lua_pushboolean) (lua_State *L, int b);
# 将轻量级用户数据压入 Lua 栈
LUA_API void  (lua_pushlightuserdata) (lua_State *L, void *p);
# 将线程压入 Lua 栈
LUA_API int   (lua_pushthread) (lua_State *L);

# 获取全局变量
LUA_API int (lua_getglobal) (lua_State *L, const char *name);
# 获取表中指定索引处的值
LUA_API int (lua_gettable) (lua_State *L, int idx);
# 获取表中指定索引处指定键的值
LUA_API int (lua_getfield) (lua_State *L, int idx, const char *k);
# 获取表中指定索引处指定整数键的值
LUA_API int (lua_geti) (lua_State *L, int idx, lua_Integer n);
# 获取表中指定索引处的值（不触发元方法）
LUA_API int (lua_rawget) (lua_State *L, int idx);
# 获取表中指定索引处指定整数键的值（不触发元方法）
LUA_API int (lua_rawgeti) (lua_State *L, int idx, lua_Integer n);
# 获取表中指定索引处指定指针键的值（不触发元方法）
LUA_API int (lua_rawgetp) (lua_State *L, int idx, const void *p);
# 创建一个新的空表，并将其压入 Lua 栈
LUA_API void  (lua_createtable) (lua_State *L, int narr, int nrec);
# 创建一个新的 userdata，并将其压入 Lua 栈
LUA_API void *(lua_newuserdatauv) (lua_State *L, size_t sz, int nuvalue);
# 获取指定索引处值的元表
LUA_API int   (lua_getmetatable) (lua_State *L, int objindex);
# 获取指定索引处值的用户值
LUA_API int  (lua_getiuservalue) (lua_State *L, int idx, int n);

# 设置全局变量
LUA_API void  (lua_setglobal) (lua_State *L, const char *name);
# 设置表中指定索引处的值
LUA_API void  (lua_settable) (lua_State *L, int idx);
# 设置表中指定索引处指定键的值
LUA_API void  (lua_setfield) (lua_State *L, int idx, const char *k);
# 设置表中指定索引处指定整数键的值
LUA_API void  (lua_seti) (lua_State *L, int idx, lua_Integer n);
# 设置表中指定索引处的值（不触发元方法）
LUA_API void  (lua_rawset) (lua_State *L, int idx);
# 设置表中指定索引处指定整数键的值（不触发元方法）
LUA_API void  (lua_rawseti) (lua_State *L, int idx, lua_Integer n);
# 设置表中指定索引处指定指针键的值（不触发元方法）
LUA_API void  (lua_rawsetp) (lua_State *L, int idx, const void *p);
# 设置指定索引处值的元表
LUA_API int   (lua_setmetatable) (lua_State *L, int objindex);
# 设置指定索引处值的用户值
LUA_API int   (lua_setiuservalue) (lua_State *L, int idx, int n);
/*
** 'load' and 'call' functions (load and run Lua code)
*/
// 调用 Lua 代码的函数，可以传入参数和返回结果
LUA_API void  (lua_callk) (lua_State *L, int nargs, int nresults,
                           lua_KContext ctx, lua_KFunction k);
// 定义宏，简化调用 lua_callk 函数的方式
#define lua_call(L,n,r)        lua_callk(L, (n), (r), 0, NULL)

// 调用 Lua 代码的函数，可以传入参数和返回结果，并处理错误
LUA_API int   (lua_pcallk) (lua_State *L, int nargs, int nresults, int errfunc,
                            lua_KContext ctx, lua_KFunction k);
// 定义宏，简化调用 lua_pcallk 函数的方式
#define lua_pcall(L,n,r,f)    lua_pcallk(L, (n), (r), (f), 0, NULL)

// 加载 Lua 代码的函数
LUA_API int   (lua_load) (lua_State *L, lua_Reader reader, void *dt,
                          const char *chunkname, const char *mode);

// 将 Lua 代码转换成二进制数据的函数
LUA_API int (lua_dump) (lua_State *L, lua_Writer writer, void *data, int strip);


/*
** coroutine functions
*/
// 挂起当前协程的函数
LUA_API int  (lua_yieldk)     (lua_State *L, int nresults, lua_KContext ctx,
                               lua_KFunction k);
// 恢复被挂起的协程的函数
LUA_API int  (lua_resume)     (lua_State *L, lua_State *from, int narg,
                               int *nres);
// 获取协程状态的函数
LUA_API int  (lua_status)     (lua_State *L);
// 判断当前协程是否可以被挂起的函数
LUA_API int (lua_isyieldable) (lua_State *L);
// 定义宏，简化调用 lua_yieldk 函数的方式
#define lua_yield(L,n)        lua_yieldk(L, (n), 0, NULL)


/*
** Warning-related functions
*/
// 设置警告函数的函数
LUA_API void (lua_setwarnf) (lua_State *L, lua_WarnFunction f, void *ud);
// 发出警告的函数
LUA_API void (lua_warning)  (lua_State *L, const char *msg, int tocont);


/*
** garbage-collection function and options
*/
// 垃圾回收函数及选项
#define LUA_GCSTOP        0
#define LUA_GCRESTART        1
#define LUA_GCCOLLECT        2
#define LUA_GCCOUNT        3
#define LUA_GCCOUNTB        4
#define LUA_GCSTEP        5
#define LUA_GCSETPAUSE        6
#define LUA_GCSETSTEPMUL    7
#define LUA_GCISRUNNING        9
#define LUA_GCGEN        10
#define LUA_GCINC        11
// 执行垃圾回收的函数
LUA_API int (lua_gc) (lua_State *L, int what, ...);


/*
** miscellaneous functions
*/
// 抛出 Lua 错误的函数
LUA_API int   (lua_error) (lua_State *L);
// 遍历表的函数
LUA_API int   (lua_next) (lua_State *L, int idx);
// 连接栈顶若干个值的函数
LUA_API void  (lua_concat) (lua_State *L, int n);
// 获取表或字符串长度的函数
LUA_API void  (lua_len)    (lua_State *L, int idx);
# 将字符串转换为数字，返回转换后的数字的长度
LUA_API size_t   (lua_stringtonumber) (lua_State *L, const char *s);

# 获取当前分配器函数，返回当前分配器函数和用户数据
LUA_API lua_Alloc (lua_getallocf) (lua_State *L, void **ud);
# 设置新的分配器函数，传入新的分配器函数和用户数据
LUA_API void      (lua_setallocf) (lua_State *L, lua_Alloc f, void *ud);

# 关闭指定索引处的对象
LUA_API void (lua_toclose) (lua_State *L, int idx);
# 关闭指定索引处的对象，并将其从栈中移除
LUA_API void (lua_closeslot) (lua_State *L, int idx);

# 一些有用的宏定义
# 获取 Lua 状态机的额外空间
#define lua_getextraspace(L)    ((void *)((char *)(L) - LUA_EXTRASPACE))

# 将指定索引处的值转换为数字，返回转换后的数字
#define lua_tonumber(L,i)    lua_tonumberx(L,(i),NULL)
# 将指定索引处的值转换为整数，返回转换后的整数
#define lua_tointeger(L,i)    lua_tointegerx(L,(i),NULL)

# 弹出栈顶的 n 个元素
#define lua_pop(L,n)        lua_settop(L, -(n)-1)

# 创建一个新的空表
#define lua_newtable(L)        lua_createtable(L, 0, 0)

# 注册一个新的全局函数
#define lua_register(L,n,f) (lua_pushcfunction(L, (f)), lua_setglobal(L, (n)))

# 将 C 函数压入栈中
#define lua_pushcfunction(L,f)    lua_pushcclosure(L, (f), 0)

# 判断指定索引处的值是否为函数
#define lua_isfunction(L,n)    (lua_type(L, (n)) == LUA_TFUNCTION)
# 判断指定索引处的值是否为表
#define lua_istable(L,n)    (lua_type(L, (n)) == LUA_TTABLE)
# 判断指定索引处的值是否为轻量用户数据
#define lua_islightuserdata(L,n)    (lua_type(L, (n)) == LUA_TLIGHTUSERDATA)
# 判断指定索引处的值是否为 nil
#define lua_isnil(L,n)        (lua_type(L, (n)) == LUA_TNIL)
# 判断指定索引处的值是否为布尔值
#define lua_isboolean(L,n)    (lua_type(L, (n)) == LUA_TBOOLEAN)
# 判断指定索引处的值是否为线程
#define lua_isthread(L,n)    (lua_type(L, (n)) == LUA_TTHREAD)
# 判断指定索引处的值是否为空
#define lua_isnone(L,n)        (lua_type(L, (n)) == LUA_TNONE)
# 判断指定索引处的值是否为空或者为 nil
#define lua_isnoneornil(L, n)    (lua_type(L, (n)) <= 0)

# 将字符串常量压入栈中
#define lua_pushliteral(L, s)    lua_pushstring(L, "" s)

# 将全局环境表压入栈中
#define lua_pushglobaltable(L)  \
    ((void)lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_GLOBALS))

# 将指定索引处的值转换为字符串，返回转换后的字符串
#define lua_tostring(L,i)    lua_tolstring(L, (i), NULL)

# 将指定索引处的值插入到指定位置，其上面的值依次向上移动
#define lua_insert(L,idx)    lua_rotate(L, (idx), 1)

# 移除指定索引处的值，并将其上面的值依次向下移动
#define lua_remove(L,idx)    (lua_rotate(L, (idx), -1), lua_pop(L, 1))

# 替换指定索引处的值为栈顶的值，并将栈顶的值移除
#define lua_replace(L,idx)    (lua_copy(L, -1, (idx)), lua_pop(L, 1)

/* }============================================================== */
/*
** compatibility macros
** ===============================================================
*/
#if defined(LUA_COMPAT_APIINTCASTS)

#define lua_pushunsigned(L,n)    lua_pushinteger(L, (lua_Integer)(n))  // 将无符号整数推入栈
#define lua_tounsignedx(L,i,is)    ((lua_Unsigned)lua_tointegerx(L,i,is))  // 将栈中索引为 i 的值转换为无符号整数
#define lua_tounsigned(L,i)    lua_tounsignedx(L,(i),NULL)  // 将栈中索引为 i 的值转换为无符号整数

#endif

#define lua_newuserdata(L,s)    lua_newuserdatauv(L,s,1)  // 创建一个新的 userdata 对象
#define lua_getuservalue(L,idx)    lua_getiuservalue(L,idx,1)  // 获取 userdata 对象的值
#define lua_setuservalue(L,idx)    lua_setiuservalue(L,idx,1)  // 设置 userdata 对象的值

#define LUA_NUMTAGS        LUA_NUMTYPES  // 定义标签数量

/* }============================================================== */

/*
** {======================================================================
** Debug API
** =======================================================================
*/


/*
** Event codes
*/
#define LUA_HOOKCALL    0  // 调用事件
#define LUA_HOOKRET    1  // 返回事件
#define LUA_HOOKLINE    2  // 行事件
#define LUA_HOOKCOUNT    3  // 计数事件
#define LUA_HOOKTAILCALL 4  // 尾调用事件


/*
** Event masks
*/
#define LUA_MASKCALL    (1 << LUA_HOOKCALL)  // 调用事件掩码
#define LUA_MASKRET    (1 << LUA_HOOKRET)  // 返回事件掩码
#define LUA_MASKLINE    (1 << LUA_HOOKLINE)  // 行事件掩码
#define LUA_MASKCOUNT    (1 << LUA_HOOKCOUNT)  // 计数事件掩码

typedef struct lua_Debug lua_Debug;  /* activation record */  // 调试信息结构体


/* Functions to be called by the debugger in specific events */
typedef void (*lua_Hook) (lua_State *L, lua_Debug *ar);  // 调试器在特定事件中调用的函数

LUA_API int (lua_getstack) (lua_State *L, int level, lua_Debug *ar);  // 获取调用栈信息
LUA_API int (lua_getinfo) (lua_State *L, const char *what, lua_Debug *ar);  // 获取特定信息
LUA_API const char *(lua_getlocal) (lua_State *L, const lua_Debug *ar, int n);  // 获取本地变量
LUA_API const char *(lua_setlocal) (lua_State *L, const lua_Debug *ar, int n);  // 设置本地变量
LUA_API const char *(lua_getupvalue) (lua_State *L, int funcindex, int n);  // 获取上值
LUA_API const char *(lua_setupvalue) (lua_State *L, int funcindex, int n);  // 设置上值

LUA_API void *(lua_upvalueid) (lua_State *L, int fidx, int n);  // 获取上值的唯一标识
LUA_API void  (lua_upvaluejoin) (lua_State *L, int fidx1, int n1,
                                               int fidx2, int n2);  // 合并上值
// 设置一个新的钩子函数，用于调试
LUA_API void (lua_sethook) (lua_State *L, lua_Hook func, int mask, int count);
// 获取当前的钩子函数
LUA_API lua_Hook (lua_gethook) (lua_State *L);
// 获取当前的钩子掩码
LUA_API int (lua_gethookmask) (lua_State *L);
// 获取当前的钩子计数
LUA_API int (lua_gethookcount) (lua_State *L);

// 设置 Lua 堆栈的最大限制
LUA_API int (lua_setcstacklimit) (lua_State *L, unsigned int limit);

// 定义 Lua 调试信息的结构体
struct lua_Debug {
  int event;  // 事件类型
  const char *name;    /* (n) */  // 函数或变量的名称
  const char *namewhat;    /* (n) 'global', 'local', 'field', 'method' */  // 名称的类型
  const char *what;    /* (S) 'Lua', 'C', 'main', 'tail' */  // 函数类型
  const char *source;    /* (S) */  // 函数所在的源文件
  size_t srclen;    /* (S) */  // 源文件名的长度
  int currentline;    /* (l) */  // 当前行号
  int linedefined;    /* (S) */  // 函数起始行号
  int lastlinedefined;    /* (S) */  // 函数结束行号
  unsigned char nups;    /* (u) number of upvalues */  // 上值的数量
  unsigned char nparams;/* (u) number of parameters */  // 参数的数量
  char isvararg;        /* (u) */  // 是否是变长参数
  char istailcall;    /* (t) */  // 是否是尾调用
  unsigned short ftransfer;   /* (r) index of first value transferred */  // 第一个传递值的索引
  unsigned short ntransfer;   /* (r) number of transferred values */  // 传递值的数量
  char short_src[LUA_IDSIZE]; /* (S) */  // 源文件名的缩写
  /* private part */
  struct CallInfo *i_ci;  /* active function */  // 活动函数的调用信息
};

/* }====================================================================== */


/******************************************************************************
* 版权所有 (C) 1994-2022 Lua.org, PUC-Rio.
*
* 特此免费授予任何获得本软件及相关文档文件（以下简称"软件"）的人
* 无限制地处理本软件的权利，包括但不限于使用、复制、修改、合并、发布、
* 分发、再许可和/或销售本软件的副本，并允许被授予本软件的人员这样做，
* 但须符合以下条件：
*
* 上述版权声明和本许可声明应包含在所有副本或重要部分的软件中。
*
* 本软件按"原样"提供，不提供任何形式的担保或条件，
/*
* 表示代码的版权声明和许可证信息
* 包括对软件的使用和免责声明
* 作者或版权持有人不对任何索赔、损害或其他责任负责
* 无论是合同诉讼、侵权行为还是其他情况，都不承担责任
* 与软件或软件使用或其他软件相关的索赔、损害或其他责任
******************************************************************************/
#endif
*/
```