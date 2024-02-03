# `nmap\liblua\lauxlib.h`

```cpp
/*
** $Id: lauxlib.h $
** 辅助函数，用于构建 Lua 库
** 请参阅 lua.h 中的版权声明
*/

#ifndef lauxlib_h
#define lauxlib_h

#include <stddef.h>
#include <stdio.h>

#include "luaconf.h"
#include "lua.h"

/* 全局表 */
#define LUA_GNAME    "_G"

typedef struct luaL_Buffer luaL_Buffer;

/* 'luaL_loadfilex' 的额外错误代码 */
#define LUA_ERRFILE     (LUA_ERRERR+1)

/* 注册表中已加载模块的表的键 */
#define LUA_LOADED_TABLE    "_LOADED"

/* 注册表中预加载加载器的表的键 */
#define LUA_PRELOAD_TABLE    "_PRELOAD"

typedef struct luaL_Reg {
  const char *name;
  lua_CFunction func;
} luaL_Reg;

#define LUAL_NUMSIZES    (sizeof(lua_Integer)*16 + sizeof(lua_Number))

LUALIB_API void (luaL_checkversion_) (lua_State *L, lua_Number ver, size_t sz);
#define luaL_checkversion(L)  \
      luaL_checkversion_(L, LUA_VERSION_NUM, LUAL_NUMSIZES)

LUALIB_API int (luaL_getmetafield) (lua_State *L, int obj, const char *e);
LUALIB_API int (luaL_callmeta) (lua_State *L, int obj, const char *e);
LUALIB_API const char *(luaL_tolstring) (lua_State *L, int idx, size_t *len);
LUALIB_API int (luaL_argerror) (lua_State *L, int arg, const char *extramsg);
LUALIB_API int (luaL_typeerror) (lua_State *L, int arg, const char *tname);
LUALIB_API const char *(luaL_checklstring) (lua_State *L, int arg, size_t *l);
LUALIB_API const char *(luaL_optlstring) (lua_State *L, int arg, const char *def, size_t *l);
LUALIB_API lua_Number (luaL_checknumber) (lua_State *L, int arg);
LUALIB_API lua_Number (luaL_optnumber) (lua_State *L, int arg, lua_Number def);
LUALIB_API lua_Integer (luaL_checkinteger) (lua_State *L, int arg);
LUALIB_API lua_Integer (luaL_optinteger) (lua_State *L, int arg, lua_Integer def);
LUALIB_API void (luaL_checkstack) (lua_State *L, int sz, const char *msg);
# 检查 Lua 值的类型是否符合预期
LUALIB_API void (luaL_checktype) (lua_State *L, int arg, int t);
# 检查 Lua 值是否为任意类型
LUALIB_API void (luaL_checkany) (lua_State *L, int arg);

# 创建新的元表并将其压入堆栈
LUALIB_API int   (luaL_newmetatable) (lua_State *L, const char *tname);
# 将指定名称的元表压入堆栈
LUALIB_API void  (luaL_setmetatable) (lua_State *L, const char *tname);
# 检查指定位置的值是否为指定类型的用户数据，并返回其指针
LUALIB_API void *(luaL_testudata) (lua_State *L, int ud, const char *tname);
# 检查指定位置的值是否为指定类型的用户数据，并返回其指针
LUALIB_API void *(luaL_checkudata) (lua_State *L, int ud, const char *tname);

# 将调用栈的信息压入堆栈
LUALIB_API void (luaL_where) (lua_State *L, int lvl);
# 抛出一个带有指定格式的错误
LUALIB_API int (luaL_error) (lua_State *L, const char *fmt, ...);

# 检查指定位置的值是否为指定列表中的一个，返回其在列表中的索引
LUALIB_API int (luaL_checkoption) (lua_State *L, int arg, const char *def,
                                   const char *const lst[]);

# 将文件操作的结果压入堆栈
LUALIB_API int (luaL_fileresult) (lua_State *L, int stat, const char *fname);
# 将执行操作的结果压入堆栈
LUALIB_API int (luaL_execresult) (lua_State *L, int stat);

# 预定义的引用
#define LUA_NOREF       (-2)
#define LUA_REFNIL      (-1)

# 创建一个指定位置的值的引用，并将其压入堆栈
LUALIB_API int (luaL_ref) (lua_State *L, int t);
# 释放指定位置的值的引用
LUALIB_API void (luaL_unref) (lua_State *L, int t, int ref);

# 加载文件作为 Lua 代码块，并将其压入堆栈
LUALIB_API int (luaL_loadfilex) (lua_State *L, const char *filename,
                                               const char *mode);
# 宏定义，加载文件作为 Lua 代码块，并将其压入堆栈
#define luaL_loadfile(L,f)    luaL_loadfilex(L,f,NULL)

# 加载缓冲区中的内容作为 Lua 代码块，并将其压入堆栈
LUALIB_API int (luaL_loadbufferx) (lua_State *L, const char *buff, size_t sz,
                                   const char *name, const char *mode);
# 加载字符串作为 Lua 代码块，并将其压入堆栈
LUALIB_API int (luaL_loadstring) (lua_State *L, const char *s);

# 创建一个新的 Lua 状态机
LUALIB_API lua_State *(luaL_newstate) (void);

# 返回指定位置的值的长度
LUALIB_API lua_Integer (luaL_len) (lua_State *L, int idx);

# 在缓冲区中添加字符串，并进行全局替换
LUALIB_API void (luaL_addgsub) (luaL_Buffer *b, const char *s,
                                     const char *p, const char *r);
# 在字符串中进行全局替换
LUALIB_API const char *(luaL_gsub) (lua_State *L, const char *s,
                                    const char *p, const char *r);

# 将指定的函数注册到指定的 Lua 状态机中
LUALIB_API void (luaL_setfuncs) (lua_State *L, const luaL_Reg *l, int nup);

# 获取指定位置的表中指定字段的子表
LUALIB_API int (luaL_getsubtable) (lua_State *L, int idx, const char *fname);
# 定义了一个名为 luaL_traceback 的函数，用于在 Lua 状态机中生成一个回溯信息
LUALIB_API void (luaL_traceback) (lua_State *L, lua_State *L1,
                                  const char *msg, int level);

# 定义了一个名为 luaL_requiref 的函数，用于在 Lua 状态机中加载指定的模块
LUALIB_API void (luaL_requiref) (lua_State *L, const char *modname,
                                 lua_CFunction openf, int glb);

# 定义了一些有用的宏

# 创建一个新的表格，并将其推入 Lua 栈中
#define luaL_newlibtable(L,l)    \
  lua_createtable(L, 0, sizeof(l)/sizeof((l)[0]) - 1)

# 创建一个新的表格，并将其推入 Lua 栈中，然后将指定的函数注册到表格中
#define luaL_newlib(L,l)  \
  (luaL_checkversion(L), luaL_newlibtable(L,l), luaL_setfuncs(L,l,0))

# 检查参数是否满足条件，如果不满足则抛出错误
#define luaL_argcheck(L, cond,arg,extramsg)    \
    ((void)(luai_likely(cond) || luaL_argerror(L, (arg), (extramsg))))

# 检查参数类型是否符合预期，如果不符合则抛出类型错误
#define luaL_argexpected(L,cond,arg,tname)    \
    ((void)(luai_likely(cond) || luaL_typeerror(L, (arg), (tname)))

# 检查指定位置的参数是否为字符串，如果是则返回其值，否则抛出错误
#define luaL_checkstring(L,n)    (luaL_checklstring(L, (n), NULL))

# 检查指定位置的参数是否为字符串，如果是则返回其值，否则返回默认值
#define luaL_optstring(L,n,d)    (luaL_optlstring(L, (n), (d), NULL))

# 返回指定位置的参数的类型名称
#define luaL_typename(L,i)    lua_typename(L, lua_type(L,(i)))

# 加载并执行指定文件中的 Lua 代码
#define luaL_dofile(L, fn) \
    (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))

# 加载并执行指定字符串中的 Lua 代码
#define luaL_dostring(L, s) \
    (luaL_loadstring(L, s) || lua_pcall(L, 0, LUA_MULTRET, 0))

# 获取指定名称的元表，并将其推入 Lua 栈中
#define luaL_getmetatable(L,n)    (lua_getfield(L, LUA_REGISTRYINDEX, (n)))

# 如果指定位置的参数为 nil，则返回默认值，否则返回参数值
#define luaL_opt(L,f,n,d)    (lua_isnoneornil(L,(n)) ? (d) : f(L,(n)))

# 加载指定大小和内容的缓冲区中的 Lua 代码
#define luaL_loadbuffer(L,s,sz,n)    luaL_loadbufferx(L,s,sz,n,NULL)

# 对 lua_Integer 类型的值执行算术操作，具有环绕运算的语义
#define luaL_intop(op,v1,v2)  \
    ((lua_Integer)((lua_Unsigned)(v1) op (lua_Unsigned)(v2)))

# 将表示失败/错误的值推入 Lua 栈中
#define luaL_pushfail(L)    lua_pushnil(L)

# 用于内部调试的断言
#if !defined(lua_assert)

#if defined LUAI_ASSERT
  #include <assert.h>
  #define lua_assert(c)        assert(c)
#else
  #define lua_assert(c)        ((void)0)
#endif

#endif
/*
** {======================================================
** Generic Buffer manipulation
** =======================================================
*/

// 定义了一个结构体 luaL_Buffer，用于通用的缓冲区操作
struct luaL_Buffer {
  char *b;  /* buffer address */  // 缓冲区地址
  size_t size;  /* buffer size */  // 缓冲区大小
  size_t n;  /* number of characters in buffer */  // 缓冲区中字符的数量
  lua_State *L;  // Lua 状态
  union {
    LUAI_MAXALIGN;  /* ensure maximum alignment for buffer */  // 确保缓冲区的最大对齐
    char b[LUAL_BUFFERSIZE];  /* initial buffer */  // 初始缓冲区
  } init;
};

// 定义了一些宏用于操作缓冲区
#define luaL_bufflen(bf)    ((bf)->n)  // 返回缓冲区中字符的数量
#define luaL_buffaddr(bf)    ((bf)->b)  // 返回缓冲区地址

#define luaL_addchar(B,c) \
  ((void)((B)->n < (B)->size || luaL_prepbuffsize((B), 1)), \
   ((B)->b[(B)->n++] = (c)))  // 向缓冲区中添加一个字符

#define luaL_addsize(B,s)    ((B)->n += (s))  // 向缓冲区中添加指定数量的字符

#define luaL_buffsub(B,s)    ((B)->n -= (s))  // 从缓冲区中减去指定数量的字符

// 定义了一些函数用于操作缓冲区
LUALIB_API void (luaL_buffinit) (lua_State *L, luaL_Buffer *B);  // 初始化缓冲区
LUALIB_API char *(luaL_prepbuffsize) (luaL_Buffer *B, size_t sz);  // 准备指定大小的缓冲区
LUALIB_API void (luaL_addlstring) (luaL_Buffer *B, const char *s, size_t l);  // 向缓冲区中添加指定长度的字符串
LUALIB_API void (luaL_addstring) (luaL_Buffer *B, const char *s);  // 向缓冲区中添加字符串
LUALIB_API void (luaL_addvalue) (luaL_Buffer *B);  // 向缓冲区中添加值
LUALIB_API void (luaL_pushresult) (luaL_Buffer *B);  // 将缓冲区中的内容推入栈中
LUALIB_API void (luaL_pushresultsize) (luaL_Buffer *B, size_t sz);  // 将缓冲区中指定大小的内容推入栈中
LUALIB_API char *(luaL_buffinitsize) (lua_State *L, luaL_Buffer *B, size_t sz);  // 初始化指定大小的缓冲区

#define luaL_prepbuffer(B)    luaL_prepbuffsize(B, LUAL_BUFFERSIZE)  // 准备缓冲区

/* }====================================================== */



/*
** {======================================================
** File handles for IO library
** =======================================================
*/

/*
** A file handle is a userdata with metatable 'LUA_FILEHANDLE' and
** initial structure 'luaL_Stream' (it may contain other fields
** after that initial structure).
*/

// 定义了文件句柄的结构体 luaL_Stream
#define LUA_FILEHANDLE          "FILE*"

typedef struct luaL_Stream {
  FILE *f;  /* stream (NULL for incompletely created streams) */  // 流（对于未完全创建的流为 NULL）
  lua_CFunction closef;  /* to close stream (NULL for closed streams) */  // 关闭流的函数（对于已关闭的流为 NULL）
} luaL_Stream;
/* }====================================================== */
/* 闭合注释，表示上一个注释块的结束 */

/*
** {==================================================================
** "Abstraction Layer" for basic report of messages and errors
** ===================================================================
*/
/* 开始一个新的注释块，用于解释下面的代码是关于基本消息和错误报告的抽象层 */

/* print a string */
#if !defined(lua_writestring)
#define lua_writestring(s,l)   fwrite((s), sizeof(char), (l), stdout)
#endif
/* 如果未定义宏lua_writestring，则定义宏lua_writestring，用于将字符串打印到标准输出 */

/* print a newline and flush the output */
#if !defined(lua_writeline)
#define lua_writeline()        (lua_writestring("\n", 1), fflush(stdout))
#endif
/* 如果未定义宏lua_writeline，则定义宏lua_writeline，用于打印换行并刷新输出 */

/* print an error message */
#if !defined(lua_writestringerror)
#define lua_writestringerror(s,p) \
        (fprintf(stderr, (s), (p)), fflush(stderr))
#endif
/* 如果未定义宏lua_writestringerror，则定义宏lua_writestringerror，用于打印错误消息 */

/* }================================================================== */
/* 闭合注释，表示上一个注释块的结束 */

/*
** {============================================================
** Compatibility with deprecated conversions
** =============================================================
*/
/* 开始一个新的注释块，用于解释下面的代码是关于与弃用转换的兼容性 */

#if defined(LUA_COMPAT_APIINTCASTS)
/* 如果定义了宏LUA_COMPAT_APIINTCASTS */

#define luaL_checkunsigned(L,a)    ((lua_Unsigned)luaL_checkinteger(L,a))
#define luaL_optunsigned(L,a,d)    \
    ((lua_Unsigned)luaL_optinteger(L,a,(lua_Integer)(d)))
/* 定义了一些宏，用于进行无符号整数的检查和可选检查 */

#define luaL_checkint(L,n)    ((int)luaL_checkinteger(L, (n)))
#define luaL_optint(L,n,d)    ((int)luaL_optinteger(L, (n), (d)))
/* 定义了一些宏，用于进行整数的检查和可选检查 */

#define luaL_checklong(L,n)    ((long)luaL_checkinteger(L, (n)))
#define luaL_optlong(L,n,d)    ((long)luaL_optinteger(L, (n), (d)))
/* 定义了一些宏，用于进行长整数的检查和可选检查 */

#endif
/* 结束条件编译 */

/* }============================================================ */
/* 闭合注释，表示上一个注释块的结束 */

#endif
/* 结束条件编译 */
```