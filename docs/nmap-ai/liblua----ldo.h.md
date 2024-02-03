# `nmap\liblua\ldo.h`

```cpp
/*
** $Id: ldo.h $
** Lua的堆栈和调用结构
** 请参阅lua.h中的版权声明
*/

#ifndef ldo_h
#define ldo_h

#include "lobject.h"  // 引入lobject.h头文件
#include "lstate.h"   // 引入lstate.h头文件
#include "lzio.h"     // 引入lzio.h头文件

/*
** 用于检查堆栈大小并在需要时扩展堆栈的宏。参数'pre'/'pos'允许宏在重新分配时保留对堆栈的指针，
** 仅在需要时执行工作。它还允许在重新分配堆栈时运行一次GC步骤。
** 'condmovestack'在重型测试中用于强制在每次检查时重新分配堆栈。
*/
#define luaD_checkstackaux(L,n,pre,pos)  \
    if (l_unlikely(L->stack_last - L->top <= (n))) \
      { pre; luaD_growstack(L, n, 1); pos; } \
        else { condmovestack(L,pre,pos); }

/* 一般情况下，'pre'/'pos'为空（没有需要保存的内容） */
#define luaD_checkstack(L,n)    luaD_checkstackaux(L,n,(void)0,(void)0)

#define savestack(L,p)        ((char *)(p) - (char *)L->stack)  // 保存堆栈指针
#define restorestack(L,n)    ((StkId)((char *)L->stack + (n)))  // 恢复堆栈指针

/* 用于检查堆栈大小，保留'p'的宏 */
#define checkstackGCp(L,n,p)  \
  luaD_checkstackaux(L, n, \
    ptrdiff_t t__ = savestack(L, p);  /* 保存'p' */ \
    luaC_checkGC(L),  /* 堆栈增长使用内存 */ \
    p = restorestack(L, t__))  /* 'pos'部分：恢复'p' */

/* 用于检查堆栈大小和GC的宏 */
#define checkstackGC(L,fsize)  \
    luaD_checkstackaux(L, (fsize), luaC_checkGC(L), (void)0)

/* 保护函数的类型，由'runprotected'运行 */
typedef void (*Pfunc) (lua_State *L, void *ud);

LUAI_FUNC void luaD_seterrorobj (lua_State *L, int errcode, StkId oldtop);  // 设置错误对象
LUAI_FUNC int luaD_protectedparser (lua_State *L, ZIO *z, const char *name,
                                                  const char *mode);  // 保护解析器
LUAI_FUNC void luaD_hook (lua_State *L, int event, int line,
                                        int fTransfer, int nTransfer);  // 钩子函数
LUAI_FUNC void luaD_hookcall (lua_State *L, CallInfo *ci);  // 钩子调用
// 在 Lua 状态机 L 上执行一个预处理的函数调用，返回调用结果
LUAI_FUNC int luaD_pretailcall (lua_State *L, CallInfo *ci, StkId func, int narg1, int delta);

// 在 Lua 状态机 L 上执行一个预处理的函数调用，返回调用信息
LUAI_FUNC CallInfo *luaD_precall (lua_State *L, StkId func, int nResults);

// 在 Lua 状态机 L 上执行一个函数调用
LUAI_FUNC void luaD_call (lua_State *L, StkId func, int nResults);

// 在 Lua 状态机 L 上执行一个函数调用，不进行协程切换
LUAI_FUNC void luaD_callnoyield (lua_State *L, StkId func, int nResults);

// 在 Lua 状态机 L 上尝试执行一个函数元方法
LUAI_FUNC StkId luaD_tryfuncTM (lua_State *L, StkId func);

// 在 Lua 状态机 L 上执行一个受保护的函数调用
LUAI_FUNC int luaD_closeprotected (lua_State *L, ptrdiff_t level, int status);

// 在 Lua 状态机 L 上执行一个受保护的函数调用
LUAI_FUNC int luaD_pcall (lua_State *L, Pfunc func, void *u, ptrdiff_t oldtop, ptrdiff_t ef);

// 在 Lua 状态机 L 上执行一个函数调用后的处理
LUAI_FUNC void luaD_poscall (lua_State *L, CallInfo *ci, int nres);

// 在 Lua 状态机 L 上重新分配栈空间
LUAI_FUNC int luaD_reallocstack (lua_State *L, int newsize, int raiseerror);

// 在 Lua 状态机 L 上扩展栈空间
LUAI_FUNC int luaD_growstack (lua_State *L, int n, int raiseerror);

// 在 Lua 状态机 L 上收缩栈空间
LUAI_FUNC void luaD_shrinkstack (lua_State *L);

// 在 Lua 状态机 L 上增加栈顶指针
LUAI_FUNC void luaD_inctop (lua_State *L);

// 在 Lua 状态机 L 上抛出一个错误
LUAI_FUNC l_noret luaD_throw (lua_State *L, int errcode);

// 在 Lua 状态机 L 上执行一个受保护的函数
LUAI_FUNC int luaD_rawrunprotected (lua_State *L, Pfunc f, void *ud);
```