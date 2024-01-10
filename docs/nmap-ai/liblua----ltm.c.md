# `nmap\liblua\ltm.c`

```
/*
** $Id: ltm.c $
** Tag methods
** See Copyright Notice in lua.h
*/

// 定义 ltm_c，表示这是 ltm 模块的源文件
#define ltm_c
// 定义 LUA_CORE，表示这是 Lua 核心模块
#define LUA_CORE

// 包含前缀文件
#include "lprefix.h"

// 包含相关的头文件
#include <string.h>
#include "lua.h"
#include "ldebug.h"
#include "ldo.h"
#include "lgc.h"
#include "lobject.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "ltm.h"
#include "lvm.h"

// 定义 userdata 类型的名称
static const char udatatypename[] = "userdata";

// 定义 Lua 中各种类型的名称
LUAI_DDEF const char *const luaT_typenames_[LUA_TOTALTYPES] = {
  "no value",
  "nil", "boolean", udatatypename, "number",
  "string", "table", "function", udatatypename, "thread",
  "upvalue", "proto" /* these last cases are used for tests only */
};

// 初始化 tag methods
void luaT_init (lua_State *L) {
  static const char *const luaT_eventname[] = {  /* ORDER TM */
    "__index", "__newindex",
    "__gc", "__mode", "__len", "__eq",
    "__add", "__sub", "__mul", "__mod", "__pow",
    "__div", "__idiv",
    "__band", "__bor", "__bxor", "__shl", "__shr",
    "__unm", "__bnot", "__lt", "__le",
    "__concat", "__call", "__close"
  };
  int i;
  for (i=0; i<TM_N; i++) {
    G(L)->tmname[i] = luaS_new(L, luaT_eventname[i]);
    luaC_fix(L, obj2gco(G(L)->tmname[i]));  /* never collect these names */
  }
}

// 获取 tag method
const TValue *luaT_gettm (Table *events, TMS event, TString *ename) {
  const TValue *tm = luaH_getshortstr(events, ename);
  lua_assert(event <= TM_EQ);
  if (notm(tm)) {  /* no tag method? */
    events->flags |= cast_byte(1u<<event);  /* cache this fact */
    return NULL;
  }
  else return tm;
}

// 根据对象和事件获取 tag method
const TValue *luaT_gettmbyobj (lua_State *L, const TValue *o, TMS event) {
  Table *mt;
  switch (ttype(o)) {
    case LUA_TTABLE:
      mt = hvalue(o)->metatable;
      break;
    case LUA_TUSERDATA:
      mt = uvalue(o)->metatable;
      break;
    default:
      mt = G(L)->mt[ttype(o)];
  }
  return (mt ? luaH_getshortstr(mt, G(L)->tmname[event]) : &G(L)->nilvalue);
}
/* 返回对象的类型名称。对于带有元表的表和用户数据，如果存在 '__name' 元字段，则使用它们的类型名称 */
const char *luaT_objtypename (lua_State *L, const TValue *o) {
  Table *mt;
  // 如果对象是表并且具有元表，或者是完整用户数据并且具有元表
  if ((ttistable(o) && (mt = hvalue(o)->metatable) != NULL) ||
      (ttisfulluserdata(o) && (mt = uvalue(o)->metatable) != NULL)) {
    // 获取元表中的 '__name' 字段
    const TValue *name = luaH_getshortstr(mt, luaS_new(L, "__name"));
    // 如果 '__name' 是字符串
    if (ttisstring(name))  
      // 返回 '__name' 字段的值作为类型名称
      return getstr(tsvalue(name));  
  }
  // 否则使用标准类型名称
  return ttypename(ttype(o));  
}


void luaT_callTM (lua_State *L, const TValue *f, const TValue *p1,
                  const TValue *p2, const TValue *p3) {
  StkId func = L->top;
  setobj2s(L, func, f);  /* 推入函数（假设有额外的栈空间） */
  setobj2s(L, func + 1, p1);  /* 第一个参数 */
  setobj2s(L, func + 2, p2);  /* 第二个参数 */
  setobj2s(L, func + 3, p3);  /* 第三个参数 */
  L->top = func + 4;
  /* 当从 Lua 代码中调用时，元方法可能会产生 yield */
  if (isLuacode(L->ci))
    luaD_call(L, func, 0);
  else
    luaD_callnoyield(L, func, 0);
}


void luaT_callTMres (lua_State *L, const TValue *f, const TValue *p1,
                     const TValue *p2, StkId res) {
  ptrdiff_t result = savestack(L, res);
  StkId func = L->top;
  setobj2s(L, func, f);  /* 推入函数（假设有额外的栈空间） */
  setobj2s(L, func + 1, p1);  /* 第一个参数 */
  setobj2s(L, func + 2, p2);  /* 第二个参数 */
  L->top += 3;
  /* 当从 Lua 代码中调用时，元方法可能会产生 yield */
  if (isLuacode(L->ci))
    luaD_call(L, func, 1);
  else
    luaD_callnoyield(L, func, 1);
  res = restorestack(L, result);
  setobjs2s(L, res, --L->top);  /* 将结果移动到其位置 */
}


static int callbinTM (lua_State *L, const TValue *p1, const TValue *p2,
                      StkId res, TMS event) {
  const TValue *tm = luaT_gettmbyobj(L, p1, event);  /* 尝试第一个操作数 */
  if (notm(tm))
    # 通过对象 p2 和事件 event 获取元方法
    tm = luaT_gettmbyobj(L, p2, event);  /* try second operand */
    # 如果没有获取到元方法，则返回 0
    if (notm(tm)) return 0;
    # 调用元方法 tm，传入参数 p1, p2，并将结果保存在 res 中
    luaT_callTMres(L, tm, p1, p2, res);
    # 返回 1，表示成功调用元方法
    return 1;
/* 
   尝试调用二元元方法，用于处理二元操作符的元方法调用
   参数：
   - L: Lua 状态机
   - p1: 第一个操作数
   - p2: 第二个操作数
   - res: 结果栈指针
   - event: 元方法事件
*/
void luaT_trybinTM (lua_State *L, const TValue *p1, const TValue *p2,
                    StkId res, TMS event) {
  // 如果调用二元元方法返回 false
  if (l_unlikely(!callbinTM(L, p1, p2, res, event))) {
    // 根据事件类型进行不同的处理
    switch (event) {
      // 位运算的元方法
      case TM_BAND: case TM_BOR: case TM_BXOR:
      case TM_SHL: case TM_SHR: case TM_BNOT: {
        // 如果操作数都是数字类型，则抛出错误
        if (ttisnumber(p1) && ttisnumber(p2))
          luaG_tointerror(L, p1, p2);
        else
          luaG_opinterror(L, p1, p2, "perform bitwise operation on");
      }
      /* 调用永远不会返回，但为了避免警告：*//* FALLTHROUGH */
      default:
        luaG_opinterror(L, p1, p2, "perform arithmetic on");
    }
  }
}

/* 
   尝试调用连接元方法
   参数：
   - L: Lua 状态机
*/
void luaT_tryconcatTM (lua_State *L) {
  // 获取栈顶指针
  StkId top = L->top;
  // 如果调用连接元方法返回 false，则抛出连接错误
  if (l_unlikely(!callbinTM(L, s2v(top - 2), s2v(top - 1), top - 2, TM_CONCAT)))
    luaG_concaterror(L, s2v(top - 2), s2v(top - 1));
}

/* 
   尝试调用关联的二元元方法
   参数：
   - L: Lua 状态机
   - p1: 第一个操作数
   - p2: 第二个操作数
   - flip: 是否翻转操作数
   - res: 结果栈指针
   - event: 元方法事件
*/
void luaT_trybinassocTM (lua_State *L, const TValue *p1, const TValue *p2,
                                       int flip, StkId res, TMS event) {
  // 如果需要翻转操作数，则调用翻转后的二元元方法，否则调用原始的二元元方法
  if (flip)
    luaT_trybinTM(L, p2, p1, res, event);
  else
    luaT_trybinTM(L, p1, p2, res, event);
}

/* 
   尝试调用整数二元元方法
   参数：
   - L: Lua 状态机
   - p1: 第一个操作数
   - i2: 第二个操作数的整数值
   - flip: 是否翻转操作数
   - res: 结果栈指针
   - event: 元方法事件
*/
void luaT_trybiniTM (lua_State *L, const TValue *p1, lua_Integer i2,
                                   int flip, StkId res, TMS event) {
  // 创建一个整数类型的辅助值
  TValue aux;
  setivalue(&aux, i2);
  // 调用关联的二元元方法
  luaT_trybinassocTM(L, p1, &aux, flip, res, event);
}

/* 
   调用一个顺序标签方法
   参数：
   - L: Lua 状态机
   - p1: 第一个操作数
   - p2: 第二个操作数
   - event: 元方法事件
*/
int luaT_callorderTM (lua_State *L, const TValue *p1, const TValue *p2,
                      TMS event) {
  // 尝试调用原始事件的元方法
  if (callbinTM(L, p1, p2, L->top, event))  
    // 返回结果是否为真值
    return !l_isfalse(s2v(L->top));
}
#if defined(LUA_COMPAT_LT_LE)
  else if (event == TM_LE) {
      /* 如果事件是TM_LE，则尝试使用'!(p2 < p1)'替换'(p1 <= p2)' */
      L->ci->callstatus |= CIST_LEQ;  /* 标记正在执行'lt'操作以替换'le' */
      if (callbinTM(L, p2, p1, L->top, TM_LT)) {
        L->ci->callstatus ^= CIST_LEQ;  /* 清除标记 */
        return l_isfalse(s2v(L->top));
      }
      /* 否则错误将移除此'ci'；不需要清除标记 */
  }
#endif
  luaG_ordererror(L, p1, p2);  /* 未找到元方法 */
  return 0;  /* 避免警告 */
}


int luaT_callorderiTM (lua_State *L, const TValue *p1, int v2,
                       int flip, int isfloat, TMS event) {
  TValue aux; const TValue *p2;
  if (isfloat) {
    setfltvalue(&aux, cast_num(v2));
  }
  else
    setivalue(&aux, v2);
  if (flip) {  /* 参数是否交换？ */
    p2 = p1; p1 = &aux;  /* 纠正参数 */
  }
  else
    p2 = &aux;
  return luaT_callorderTM(L, p1, p2, event);
}


void luaT_adjustvarargs (lua_State *L, int nfixparams, CallInfo *ci,
                         const Proto *p) {
  int i;
  int actual = cast_int(L->top - ci->func) - 1;  /* 参数个数 */
  int nextra = actual - nfixparams;  /* 额外参数个数 */
  ci->u.l.nextraargs = nextra;
  luaD_checkstack(L, p->maxstacksize + 1);
  /* 将函数复制到栈顶 */
  setobjs2s(L, L->top++, ci->func);
  /* 将固定参数移动到栈顶 */
  for (i = 1; i <= nfixparams; i++) {
    setobjs2s(L, L->top++, ci->func + i);
    setnilvalue(s2v(ci->func + i));  /* 擦除原始参数（用于GC） */
  }
  ci->func += actual + 1;
  ci->top += actual + 1;
  lua_assert(L->top <= ci->top && ci->top <= L->stack_last);
}


void luaT_getvarargs (lua_State *L, CallInfo *ci, StkId where, int wanted) {
  int i;
  int nextra = ci->u.l.nextraargs;
  if (wanted < 0) {
    wanted = nextra;  /* 获取所有可用的额外参数 */
    checkstackGCp(L, nextra, where);  /* 确保栈空间 */
    L->top = where + nextra;  /* 设置栈顶指针，下一条指令将需要这个位置 */
  }
  for (i = 0; i < wanted && i < nextra; i++)
    setobjs2s(L, where + i, ci->func - nextra + i);
  for (; i < wanted; i++)   /* 用 nil 值填充剩余的需要的结果 */
    setnilvalue(s2v(where + i));
# 代码块结束
```