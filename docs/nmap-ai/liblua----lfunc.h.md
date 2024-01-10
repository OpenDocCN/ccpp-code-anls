# `nmap\liblua\lfunc.h`

```
/*
** $Id: lfunc.h $
** 辅助函数，用于操作原型和闭包
** 请参阅 lua.h 中的版权声明
*/

#ifndef lfunc_h
#define lfunc_h

#include "lobject.h"

// 计算闭包的大小
#define sizeCclosure(n)    (cast_int(offsetof(CClosure, upvalue)) + \
                         cast_int(sizeof(TValue)) * (n))

// 计算 Lua 闭包的大小
#define sizeLclosure(n)    (cast_int(offsetof(LClosure, upvals)) + \
                         cast_int(sizeof(TValue *)) * (n))

// 检测线程是否在 'twups' 列表中
#define isintwups(L)    (L->twups != L)

// 闭包中最大的上值数量（包括 C 和 Lua 闭包）。值必须适合于虚拟机寄存器
#define MAXUPVAL    255

// 检测上值是否打开
#define upisopen(up)    ((up)->v != &(up)->u.value)

// 获取上值的级别
#define uplevel(up)    check_exp(upisopen(up), cast(StkId, (up)->v))

// 在放弃原型中闭包缓存之前的最大缺失次数
#define MAXMISS        10

// 关闭上值并保留栈顶的特殊状态
#define CLOSEKTOP    (-1)

// 创建新的原型
LUAI_FUNC Proto *luaF_newproto (lua_State *L);
// 创建新的 C 闭包
LUAI_FUNC CClosure *luaF_newCclosure (lua_State *L, int nupvals);
// 创建新的 Lua 闭包
LUAI_FUNC LClosure *luaF_newLclosure (lua_State *L, int nupvals);
// 初始化上值
LUAI_FUNC void luaF_initupvals (lua_State *L, LClosure *cl);
// 查找上值
LUAI_FUNC UpVal *luaF_findupval (lua_State *L, StkId level);
// 创建新的 to-be-closed 上值
LUAI_FUNC void luaF_newtbcupval (lua_State *L, StkId level);
// 关闭上值
LUAI_FUNC void luaF_closeupval (lua_State *L, StkId level);
// 关闭
LUAI_FUNC void luaF_close (lua_State *L, StkId level, int status, int yy);
// 解除上值的链接
LUAI_FUNC void luaF_unlinkupval (UpVal *uv);
// 释放原型
LUAI_FUNC void luaF_freeproto (lua_State *L, Proto *f);
// 获取本地名称
LUAI_FUNC const char *luaF_getlocalname (const Proto *func, int local_number,
                                         int pc);

#endif
```