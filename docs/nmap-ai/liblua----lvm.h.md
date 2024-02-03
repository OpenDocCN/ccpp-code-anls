# `nmap\liblua\lvm.h`

```cpp
# 定义了 Lua 虚拟机的头文件
# 包含了一些必要的头文件
#ifndef lvm_h
#define lvm_h

#include "ldo.h"
#include "lobject.h"
#include "ltm.h"

# 如果没有定义 LUA_NOCVTN2S，则定义 cvt2str(o) 为 ttisnumber(o)
# 否则定义 cvt2str(o) 为 0
#if !defined(LUA_NOCVTN2S)
#define cvt2str(o)    ttisnumber(o)
#else
#define cvt2str(o)    0    /* no conversion from numbers to strings */
#endif

# 如果没有定义 LUA_NOCVTS2N，则定义 cvt2num(o) 为 ttisstring(o)
# 否则定义 cvt2num(o) 为 0
#if !defined(LUA_NOCVTS2N)
#define cvt2num(o)    ttisstring(o)
#else
#define cvt2num(o)    0    /* no conversion from strings to numbers */
#endif

# 如果没有定义 LUA_FLOORN2I，则定义 LUA_FLOORN2I 为 F2Ieq
# F2Ieq 表示不进行四舍五入，只接受整数值
#if !defined(LUA_FLOORN2I)
#define LUA_FLOORN2I        F2Ieq
#endif

# 定义了浮点数到整数的四舍五入模式
typedef enum {
  F2Ieq,     /* no rounding; accepts only integral values */
  F2Ifloor,  /* takes the floor of the number */
  F2Iceil    /* takes the ceil of the number */
} F2Imod;

# 将对象转换为浮点数（包括字符串转换）
#define tonumber(o,n) \
    (ttisfloat(o) ? (*(n) = fltvalue(o), 1) : luaV_tonumber_(o,n))

# 将对象转换为浮点数（不包括字符串转换）
#define tonumberns(o,n) \
    (ttisfloat(o) ? ((n) = fltvalue(o), 1) : \
    (ttisinteger(o) ? ((n) = cast_num(ivalue(o)), 1) : 0))

# 将对象转换为整数（包括字符串转换）
#define tointeger(o,i) \
  (l_likely(ttisinteger(o)) ? (*(i) = ivalue(o), 1) \
                          : luaV_tointeger(o,i,LUA_FLOORN2I))

# 将对象转换为整数（不包括字符串转换）
#define tointegerns(o,i) \
  (l_likely(ttisinteger(o)) ? (*(i) = ivalue(o), 1) \
                          : luaV_tointegerns(o,i,LUA_FLOORN2I))

# 定义了整数操作
#define intop(op,v1,v2) l_castU2S(l_castS2U(v1) op l_castS2U(v2))

# 定义了 luaV_rawequalobj(t1,t2) 为 luaV_equalobj(NULL,t1,t2)

# 'gettable' 的快速通道：如果 't' 是一个表并且 't[k]' 存在，则返回 1，'slot' 指向 't[k]'（最终结果的位置）
/*
** 否则，返回 0（表示需要检查元方法），'slot' 指向空的 't[k]'（如果 't' 是表）或 NULL（否则）。'f' 是要使用的原始获取函数。
*/
#define luaV_fastget(L,t,k,slot,f) \
  (!ttistable(t)  \
   ? (slot = NULL, 0)  /* 不是表；'slot' 为 NULL，结果为 0 */  \
   : (slot = f(hvalue(t), k),  /* 否则，进行原始访问 */  \
      !isempty(slot)))  /* 结果不为空？ */


/*
** 'luaV_fastget' 的特殊情况，用于整数，在其中内联 'luaH_getint' 的快速情况。
*/
#define luaV_fastgeti(L,t,k,slot) \
  (!ttistable(t)  \
   ? (slot = NULL, 0)  /* 不是表；'slot' 为 NULL，结果为 0 */  \
   : (slot = (l_castS2U(k) - 1u < hvalue(t)->alimit) \
              ? &hvalue(t)->array[k - 1] : luaH_getint(hvalue(t), k), \
      !isempty(slot)))  /* 结果不为空？ */


/*
** 完成快速设置操作（当快速获取成功时）。在这种情况下，'slot' 指向要放置值的位置。
*/
#define luaV_finishfastset(L,t,slot,v) \
    { setobj2t(L, cast(TValue *,slot), v); \
      luaC_barrierback(L, gcvalue(t), v); }




LUAI_FUNC int luaV_equalobj (lua_State *L, const TValue *t1, const TValue *t2);
LUAI_FUNC int luaV_lessthan (lua_State *L, const TValue *l, const TValue *r);
LUAI_FUNC int luaV_lessequal (lua_State *L, const TValue *l, const TValue *r);
LUAI_FUNC int luaV_tonumber_ (const TValue *obj, lua_Number *n);
LUAI_FUNC int luaV_tointeger (const TValue *obj, lua_Integer *p, F2Imod mode);
LUAI_FUNC int luaV_tointegerns (const TValue *obj, lua_Integer *p,
                                F2Imod mode);
LUAI_FUNC int luaV_flttointeger (lua_Number n, lua_Integer *p, F2Imod mode);
LUAI_FUNC void luaV_finishget (lua_State *L, const TValue *t, TValue *key,
                               StkId val, const TValue *slot);
LUAI_FUNC void luaV_finishset (lua_State *L, const TValue *t, TValue *key,
                               TValue *val, const TValue *slot);
LUAI_FUNC void luaV_finishOp (lua_State *L);
// 执行 Lua 代码
LUAI_FUNC void luaV_execute (lua_State *L, CallInfo *ci);

// 连接 Lua 值
LUAI_FUNC void luaV_concat (lua_State *L, int total);

// 对 Lua 整数进行整数除法
LUAI_FUNC lua_Integer luaV_idiv (lua_State *L, lua_Integer x, lua_Integer y);

// 对 Lua 整数进行取模运算
LUAI_FUNC lua_Integer luaV_mod (lua_State *L, lua_Integer x, lua_Integer y);

// 对 Lua 浮点数进行取模运算
LUAI_FUNC lua_Number luaV_modf (lua_State *L, lua_Number x, lua_Number y);

// 对 Lua 整数进行左移操作
LUAI_FUNC lua_Integer luaV_shiftl (lua_Integer x, lua_Integer y);

// 获取 Lua 对象的长度
LUAI_FUNC void luaV_objlen (lua_State *L, StkId ra, const TValue *rb);
```