# `nmap\liblua\lapi.c`

```
/*
** $Id: lapi.c $
** Lua API
** See Copyright Notice in lua.h
*/

// 定义宏，标识 lapi.c 文件
#define lapi_c
// 定义宏，标识 Lua 核心
#define LUA_CORE

// 包含 Lua 头文件
#include "lprefix.h"

// 包含标准库头文件
#include <limits.h>
#include <stdarg.h>
#include <string.h>

// 包含 Lua 头文件
#include "lua.h"

// 包含 Lua API 头文件
#include "lapi.h"
// 包含调试相关头文件
#include "ldebug.h"
// 包含执行相关头文件
#include "ldo.h"
// 包含函数相关头文件
#include "lfunc.h"
// 包含垃圾回收相关头文件
#include "lgc.h"
// 包含内存相关头文件
#include "lmem.h"
// 包含对象相关头文件
#include "lobject.h"
// 包含状态相关头文件
#include "lstate.h"
// 包含字符串相关头文件
#include "lstring.h"
// 包含表相关头文件
#include "ltable.h"
// 包含元方法相关头文件
#include "ltm.h"
// 包含二进制转储相关头文件
#include "lundump.h"
// 包含虚拟机相关头文件
#include "lvm.h"

// Lua 版本信息
const char lua_ident[] =
  "$LuaVersion: " LUA_COPYRIGHT " $"
  "$LuaAuthors: " LUA_AUTHORS " $";

// 测试索引是否有效（不是 'nilvalue'）
#define isvalid(L, o)    (!ttisnil(o) || o != &G(L)->nilvalue)

// 测试是否伪索引
#define ispseudo(i)        ((i) <= LUA_REGISTRYINDEX)

// 测试是否为上值
#define isupvalue(i)        ((i) < LUA_REGISTRYINDEX)

// 将可接受的索引转换为指向其相应值的指针
static TValue *index2value (lua_State *L, int idx) {
  CallInfo *ci = L->ci;
  if (idx > 0) {
    StkId o = ci->func + idx;
    api_check(L, idx <= L->ci->top - (ci->func + 1), "unacceptable index");
    if (o >= L->top) return &G(L)->nilvalue;
    else return s2v(o);
  }
  else if (!ispseudo(idx)) {  /* 负索引 */
    api_check(L, idx != 0 && -idx <= L->top - (ci->func + 1), "invalid index");
    return s2v(L->top + idx);
  }
  else if (idx == LUA_REGISTRYINDEX)
    return &G(L)->l_registry;
  else {  /* 上值 */
    idx = LUA_REGISTRYINDEX - idx;
    api_check(L, idx <= MAXUPVAL + 1, "upvalue index too large");
    if (ttisCclosure(s2v(ci->func))) {  /* C 闭包？ */
      CClosure *func = clCvalue(s2v(ci->func));
      return (idx <= func->nupvalues) ? &func->upvalue[idx-1]
                                      : &G(L)->nilvalue;
    }
    else {  /* light C function or Lua function (through a hook)?) */
      // 检查当前调用的函数是否为轻量级 C 函数或 Lua 函数（通过钩子调用）
      api_check(L, ttislcf(s2v(ci->func)), "caller not a C function");
      // 返回空值，表示没有闭包
      return &G(L)->nilvalue;  /* no upvalues */
    }
  }
}

/*
** Convert a valid actual index (not a pseudo-index) to its address.
*/
l_sinline StkId index2stack (lua_State *L, int idx) {
  CallInfo *ci = L->ci;  // 获取当前调用信息
  if (idx > 0) {  // 如果索引大于0
    StkId o = ci->func + idx;  // 计算索引对应的地址
    api_check(L, o < L->top, "invalid index");  // 检查索引是否有效
    return o;  // 返回地址
  }
  else {    /* non-positive index */
    api_check(L, idx != 0 && -idx <= L->top - (ci->func + 1), "invalid index");  // 检查索引是否有效
    api_check(L, !ispseudo(idx), "invalid index");  // 检查索引是否为伪索引
    return L->top + idx;  // 返回地址
  }
}

LUA_API int lua_checkstack (lua_State *L, int n) {
  int res;
  CallInfo *ci;
  lua_lock(L);  // 锁定 Lua 状态
  ci = L->ci;  // 获取当前调用信息
  api_check(L, n >= 0, "negative 'n'");  // 检查参数是否为负数
  if (L->stack_last - L->top > n)  /* stack large enough? */  // 如果栈空间足够
    res = 1;  /* yes; check is OK */  // 设置结果为1
  else {  /* no; need to grow stack */  // 如果栈空间不够
    int inuse = cast_int(L->top - L->stack) + EXTRA_STACK;  // 计算已使用的栈空间
    if (inuse > LUAI_MAXSTACK - n)  /* can grow without overflow? */  // 如果可以扩展栈空间
      res = 0;  /* no */  // 设置结果为0
    else  /* try to grow stack */  // 尝试扩展栈空间
      res = luaD_growstack(L, n, 0);  // 调用扩展栈空间的函数
  }
  if (res && ci->top < L->top + n)  // 如果结果为真且调用信息的栈顶小于栈顶加上n
    ci->top = L->top + n;  /* adjust frame top */  // 调整帧的栈顶
  lua_unlock(L);  // 解锁 Lua 状态
  return res;  // 返回结果
}

LUA_API void lua_xmove (lua_State *from, lua_State *to, int n) {
  int i;
  if (from == to) return;  // 如果源状态和目标状态相同，则返回
  lua_lock(to);  // 锁定目标 Lua 状态
  api_checknelems(from, n);  // 检查源状态的栈是否有足够的元素
  api_check(from, G(from) == G(to), "moving among independent states");  // 检查源状态和目标状态是否独立
  api_check(from, to->ci->top - to->top >= n, "stack overflow");  // 检查目标状态的栈是否会溢出
  from->top -= n;  // 调整源状态的栈顶
  for (i = 0; i < n; i++) {
    setobjs2s(to, to->top, from->top + i);  // 将源状态的栈元素复制到目标状态的栈
    to->top++;  /* stack already checked by previous 'api_check' */  // 栈顶指针后移
  }
  lua_unlock(to);  // 解锁目标 Lua 状态
}

LUA_API lua_CFunction lua_atpanic (lua_State *L, lua_CFunction panicf) {
  lua_CFunction old;
  lua_lock(L);  // 锁定 Lua 状态
  old = G(L)->panic;  // 保存旧的 panic 函数
  G(L)->panic = panicf;  // 设置新的 panic 函数
  lua_unlock(L);  // 解锁 Lua 状态
  return old;  // 返回旧的 panic 函数
}

LUA_API lua_Number lua_version (lua_State *L) {
  UNUSED(L);  // 忽略参数
  return LUA_VERSION_NUM;  // 返回 Lua 版本号
}

/*
** basic stack manipulation
*/

/*
** convert an acceptable stack index into an absolute index
*/
# 返回绝对索引
LUA_API int lua_absindex (lua_State *L, int idx) {
  return (idx > 0 || ispseudo(idx))  # 如果索引大于 0 或者是伪索引
         ? idx  # 返回索引
         : cast_int(L->top - L->ci->func) + idx;  # 返回计算后的索引
}


# 获取栈顶索引
LUA_API int lua_gettop (lua_State *L) {
  return cast_int(L->top - (L->ci->func + 1));  # 返回计算后的栈顶索引
}


# 设置栈顶索引
LUA_API void lua_settop (lua_State *L, int idx) {
  CallInfo *ci;  # 调用信息
  StkId func, newtop;  # 函数和新栈顶
  ptrdiff_t diff;  # 新栈顶的差异
  lua_lock(L);  # 锁定 Lua
  ci = L->ci;  # 调用信息
  func = ci->func;  # 函数
  if (idx >= 0) {  # 如果索引大于等于 0
    api_check(L, idx <= ci->top - (func + 1), "new top too large");  # 检查新栈顶是否太大
    diff = ((func + 1) + idx) - L->top;  # 计算差异
    for (; diff > 0; diff--)  # 循环直到差异为 0
      setnilvalue(s2v(L->top++));  # 设置新的空值
  }
  else {  # 否则
    api_check(L, -(idx+1) <= (L->top - (func + 1)), "invalid new top");  # 检查新栈顶是否有效
    diff = idx + 1;  # 计算差异
  }
  api_check(L, L->tbclist < L->top, "previous pop of an unclosed slot");  # 检查未关闭槽的弹出
  newtop = L->top + diff;  # 新栈顶
  if (diff < 0 && L->tbclist >= newtop) {  # 如果差异小于 0 并且未关闭槽大于等于新栈顶
    lua_assert(hastocloseCfunc(ci->nresults));  # 断言是否需要关闭 C 函数
    luaF_close(L, newtop, CLOSEKTOP, 0);  # 关闭函数
  }
  L->top = newtop;  # 设置新栈顶
  lua_unlock(L);  # 解锁 Lua
}


# 关闭槽
LUA_API void lua_closeslot (lua_State *L, int idx) {
  StkId level;  # 级别
  lua_lock(L);  # 锁定 Lua
  level = index2stack(L, idx);  # 级别等于索引到栈
  api_check(L, hastocloseCfunc(L->ci->nresults) && L->tbclist == level,
     "no variable to close at given level");  # 检查给定级别是否有要关闭的变量
  luaF_close(L, level, CLOSEKTOP, 0);  # 关闭函数
  level = index2stack(L, idx);  # 级别等于索引到栈
  setnilvalue(s2v(level));  # 设置空值
  lua_unlock(L);  # 解锁 Lua
}


# 反转栈段
# 注意：我们只移动栈内的值。
# （我们不移动可能存在的其他字段。）
l_sinline void reverse (lua_State *L, StkId from, StkId to) {
  for (; from < to; from++, to--) {  # 循环直到 from 小于 to
    TValue temp;  # 临时值
    setobj(L, &temp, s2v(from));  # 设置临时值
    setobjs2s(L, from, to);  # 设置 from 的值为 to 的值
    setobj2s(L, to, &temp);  # 设置 to 的值为临时值
  }
}
# 旋转栈中索引为 idx 的元素及其后的 n 个元素
LUA_API void lua_rotate (lua_State *L, int idx, int n) {
  StkId p, t, m;
  lua_lock(L);
  t = L->top - 1;  /* end of stack segment being rotated */  # 标记旋转的栈段的结束位置
  p = index2stack(L, idx);  /* start of segment */  # 标记旋转的栈段的开始位置
  api_check(L, (n >= 0 ? n : -n) <= (t - p + 1), "invalid 'n'");  # 检查 n 是否合法
  m = (n >= 0 ? t - n : p - n - 1);  /* end of prefix */  # 计算前缀的结束位置
  reverse(L, p, m);  /* reverse the prefix with length 'n' */  # 反转长度为 n 的前缀
  reverse(L, m + 1, t);  /* reverse the suffix */  # 反转后缀
  reverse(L, p, t);  /* reverse the entire segment */  # 反转整个栈段
  lua_unlock(L);  # 解锁 Lua 状态机
}


# 将栈中索引为 fromidx 的元素复制到栈中索引为 toidx 的位置
LUA_API void lua_copy (lua_State *L, int fromidx, int toidx) {
  TValue *fr, *to;
  lua_lock(L);  # 锁定 Lua 状态机
  fr = index2value(L, fromidx);  # 获取源索引对应的值
  to = index2value(L, toidx);  # 获取目标索引对应的值
  api_check(L, isvalid(L, to), "invalid index");  # 检查目标索引是否合法
  setobj(L, to, fr);  # 将源值复制到目标位置
  if (isupvalue(toidx))  /* function upvalue? */  # 如果是函数的 upvalue
    luaC_barrier(L, clCvalue(s2v(L->ci->func)), fr);  # 执行垃圾回收屏障
  /* LUA_REGISTRYINDEX does not need gc barrier
     (collector revisits it before finishing collection) */  # LUA_REGISTRYINDEX 不需要垃圾回收屏障
  lua_unlock(L);  # 解锁 Lua 状态机
}


# 将栈中索引为 idx 的元素的值压入栈顶
LUA_API void lua_pushvalue (lua_State *L, int idx) {
  lua_lock(L);  # 锁定 Lua 状态机
  setobj2s(L, L->top, index2value(L, idx));  # 将索引为 idx 的元素的值复制到栈顶
  api_incr_top(L);  # 增加栈顶指针
  lua_unlock(L);  # 解锁 Lua 状态机
}


# 获取栈中索引为 idx 的元素的类型
LUA_API int lua_type (lua_State *L, int idx) {
  const TValue *o = index2value(L, idx);  # 获取索引为 idx 的元素的值
  return (isvalid(L, o) ? ttype(o) : LUA_TNONE);  # 如果元素合法，则返回其类型，否则返回 LUA_TNONE
}


# 获取类型为 t 的类型名
LUA_API const char *lua_typename (lua_State *L, int t) {
  UNUSED(L);  # 忽略 Lua 状态机
  api_check(L, LUA_TNONE <= t && t < LUA_NUMTYPES, "invalid type");  # 检查类型是否合法
  return ttypename(t);  # 返回类型名
}


# 判断栈中索引为 idx 的元素是否为 C 函数
LUA_API int lua_iscfunction (lua_State *L, int idx) {
  const TValue *o = index2value(L, idx);  # 获取索引为 idx 的元素的值
  return (ttislcf(o) || (ttisCclosure(o)));  # 判断是否为 C 函数
}


# 判断栈中索引为 idx 的元素是否为整数
LUA_API int lua_isinteger (lua_State *L, int idx) {
  const TValue *o = index2value(L, idx);  # 获取索引为 idx 的元素的值
  return ttisinteger(o);  # 判断是否为整数
}


# 判断栈中索引为 idx 的元素是否为数字
LUA_API int lua_isnumber (lua_State *L, int idx) {
  lua_Number n;  # 定义一个 lua_Number 类型的变量
  const TValue *o = index2value(L, idx);  # 获取索引为 idx 的元素的值
  return tonumber(o, &n);  # 尝试将其转换为数字
}
# 检查给定索引处的值是否为字符串类型
LUA_API int lua_isstring (lua_State *L, int idx) {
  const TValue *o = index2value(L, idx);  # 获取给定索引处的值
  return (ttisstring(o) || cvt2str(o));  # 返回该值是否为字符串类型的布尔值
}


# 检查给定索引处的值是否为用户数据类型
LUA_API int lua_isuserdata (lua_State *L, int idx) {
  const TValue *o = index2value(L, idx);  # 获取给定索引处的值
  return (ttisfulluserdata(o) || ttislightuserdata(o));  # 返回该值是否为用户数据类型的布尔值
}


# 比较给定两个索引处的值是否相等
LUA_API int lua_rawequal (lua_State *L, int index1, int index2) {
  const TValue *o1 = index2value(L, index1);  # 获取第一个索引处的值
  const TValue *o2 = index2value(L, index2);  # 获取第二个索引处的值
  return (isvalid(L, o1) && isvalid(L, o2)) ? luaV_rawequalobj(o1, o2) : 0;  # 如果两个值都有效，则调用 luaV_rawequalobj 进行比较，否则返回 0
}


# 执行 Lua 中的算术运算
LUA_API void lua_arith (lua_State *L, int op) {
  lua_lock(L);  # 锁定 Lua 状态
  if (op != LUA_OPUNM && op != LUA_OPBNOT)
    api_checknelems(L, 2);  # 检查堆栈中是否有两个操作数
  else {  # 对于一元操作，添加一个虚拟的第二个操作数
    api_checknelems(L, 1);  # 检查堆栈中是否有一个操作数
    setobjs2s(L, L->top, L->top - 1);  # 设置第二个操作数为堆栈顶部的值
    api_incr_top(L);  # 增加堆栈顶部指针
  }
  # 第一个操作数在 top - 2，第二个在 top - 1；结果放在 top - 2
  luaO_arith(L, op, s2v(L->top - 2), s2v(L->top - 1), L->top - 2);  # 执行算术运算
  L->top--;  # 移除第二个操作数
  lua_unlock(L);  # 解锁 Lua 状态
}


# 比较给定两个索引处的值
LUA_API int lua_compare (lua_State *L, int index1, int index2, int op) {
  const TValue *o1;
  const TValue *o2;
  int i = 0;
  lua_lock(L);  # 锁定 Lua 状态，可能会调用标签方法
  o1 = index2value(L, index1);  # 获取第一个索引处的值
  o2 = index2value(L, index2);  # 获取第二个索引处的值
  if (isvalid(L, o1) && isvalid(L, o2)) {  # 如果两个值都有效
    switch (op) {
      case LUA_OPEQ: i = luaV_equalobj(L, o1, o2); break;  # 如果操作为相等比较，则调用 luaV_equalobj
      case LUA_OPLT: i = luaV_lessthan(L, o1, o2); break;  # 如果操作为小于比较，则调用 luaV_lessthan
      case LUA_OPLE: i = luaV_lessequal(L, o1, o2); break;  # 如果操作为小于等于比较，则调用 luaV_lessequal
      default: api_check(L, 0, "invalid option");  # 其他情况下，抛出无效选项的错误
    }
  }
  lua_unlock(L);  # 解锁 Lua 状态
  return i;  # 返回比较结果
}


# 将字符串转换为数字
LUA_API size_t lua_stringtonumber (lua_State *L, const char *s) {
  size_t sz = luaO_str2num(s, s2v(L->top));  # 将字符串转换为数字
  if (sz != 0)
    api_incr_top(L);  # 如果转换成功，增加堆栈顶部指针
  return sz;  # 返回转换后的数字
}


# 将给定索引处的值转换为数字
LUA_API lua_Number lua_tonumberx (lua_State *L, int idx, int *pisnum) {
  lua_Number n = 0;  # 初始化数字为 0
  const TValue *o = index2value(L, idx);  # 获取给定索引处的值
  int isnum = tonumber(o, &n);  # 将值转换为数字
  if (pisnum)
    *pisnum = isnum;  # 如果传入了 pisnum 指针，则将转换结果存入其中
  return n;  # 返回转换后的数字
}
LUA_API lua_Integer lua_tointegerx (lua_State *L, int idx, int *pisnum) {
  // 初始化结果为0
  lua_Integer res = 0;
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 尝试将值转换为整数，返回是否成功的标志，并将结果保存在res中
  int isnum = tointeger(o, &res);
  // 如果传入的pisnum不为空，则将isnum的值保存在pisnum指向的位置
  if (pisnum)
    *pisnum = isnum;
  // 返回转换后的整数值
  return res;
}


LUA_API int lua_toboolean (lua_State *L, int idx) {
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 返回值是否为真的标志
  return !l_isfalse(o);
}


LUA_API const char *lua_tolstring (lua_State *L, int idx, size_t *len) {
  TValue *o;
  // 锁定Lua状态机
  lua_lock(L);
  // 获取栈上指定索引位置的值
  o = index2value(L, idx);
  // 如果值不是字符串类型
  if (!ttisstring(o)) {
    // 如果无法转换为字符串，则返回空指针
    if (!cvt2str(o)) {  /* not convertible? */
      if (len != NULL) *len = 0;
      // 解锁Lua状态机并返回空指针
      lua_unlock(L);
      return NULL;
    }
    // 将值转换为字符串
    luaO_tostring(L, o);
    // 检查垃圾回收
    luaC_checkGC(L);
    // 重新获取栈上指定索引位置的值，因为前面的操作可能导致栈的重新分配
    o = index2value(L, idx);  /* previous call may reallocate the stack */
  }
  // 如果len不为空，则将字符串长度保存在len指向的位置
  if (len != NULL)
    *len = vslen(o);
  // 解锁Lua状态机并返回字符串值
  lua_unlock(L);
  return svalue(o);
}


LUA_API lua_Unsigned lua_rawlen (lua_State *L, int idx) {
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 根据值的类型返回对应的长度
  switch (ttypetag(o)) {
    case LUA_VSHRSTR: return tsvalue(o)->shrlen;
    case LUA_VLNGSTR: return tsvalue(o)->u.lnglen;
    case LUA_VUSERDATA: return uvalue(o)->len;
    case LUA_VTABLE: return luaH_getn(hvalue(o));
    default: return 0;
  }
}


LUA_API lua_CFunction lua_tocfunction (lua_State *L, int idx) {
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 如果值是C函数，则返回其指针
  if (ttislcf(o)) return fvalue(o);
  // 如果值是闭包函数，则返回其函数指针
  else if (ttisCclosure(o))
    return clCvalue(o)->f;
  else return NULL;  /* not a C function */
}


l_sinline void *touserdata (const TValue *o) {
  // 根据值的类型返回对应的用户数据指针
  switch (ttype(o)) {
    case LUA_TUSERDATA: return getudatamem(uvalue(o));
    case LUA_TLIGHTUSERDATA: return pvalue(o);
    default: return NULL;
  }
}


LUA_API void *lua_touserdata (lua_State *L, int idx) {
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 返回对应的用户数据指针
  return touserdata(o);
}


LUA_API lua_State *lua_tothread (lua_State *L, int idx) {
  // 获取栈上指定索引位置的值
  const TValue *o = index2value(L, idx);
  // 如果值不是线程类型，则返回空指针，否则返回线程指针
  return (!ttisthread(o)) ? NULL : thvalue(o);
}


/*
** Returns a pointer to the internal representation of an object.
# 将 Lua 栈中指定索引处的值转换为指针类型并返回
LUA_API const void *lua_topointer (lua_State *L, int idx) {
  # 获取指定索引处的值
  const TValue *o = index2value(L, idx);
  # 根据值的类型进行不同的处理
  switch (ttypetag(o)) {
    # 如果是浮点数类型，将其转换为 void* 类型并返回
    case LUA_VLCF: return cast_voidp(cast_sizet(fvalue(o)));
    # 如果是用户数据类型或轻量用户数据类型，直接返回其指针
    case LUA_VUSERDATA: case LUA_VLIGHTUSERDATA:
      return touserdata(o);
    # 其他类型的值，如果是可回收的，返回其垃圾收集对象的指针，否则返回 NULL
    default: {
      if (iscollectable(o))
        return gcvalue(o);
      else
        return NULL;
    }
  }
}

# 将 nil 值推入 Lua 栈中
LUA_API void lua_pushnil (lua_State *L) {
  lua_lock(L);
  setnilvalue(s2v(L->top));
  api_incr_top(L);
  lua_unlock(L);
}

# 将数字推入 Lua 栈中
LUA_API void lua_pushnumber (lua_State *L, lua_Number n) {
  lua_lock(L);
  setfltvalue(s2v(L->top), n);
  api_incr_top(L);
  lua_unlock(L);
}

# 将整数推入 Lua 栈中
LUA_API void lua_pushinteger (lua_State *L, lua_Integer n) {
  lua_lock(L);
  setivalue(s2v(L->top), n);
  api_incr_top(L);
  lua_unlock(L);
}

# 将指定长度的字符串推入 Lua 栈中
LUA_API const char *lua_pushlstring (lua_State *L, const char *s, size_t len) {
  TString *ts;
  lua_lock(L);
  # 如果长度为 0，则创建一个空字符串，否则创建指定长度的字符串
  ts = (len == 0) ? luaS_new(L, "") : luaS_newlstr(L, s, len);
  setsvalue2s(L, L->top, ts);
  api_incr_top(L);
  luaC_checkGC(L);
  lua_unlock(L);
  return getstr(ts);
}

# 将以空字符结尾的字符串推入 Lua 栈中
LUA_API const char *lua_pushstring (lua_State *L, const char *s) {
  lua_lock(L);
  # 如果字符串为空，则将 nil 值推入栈中，否则将字符串复制到 Lua 内部并推入栈中
  if (s == NULL)
    setnilvalue(s2v(L->top));
  else {
    TString *ts;
    ts = luaS_new(L, s);
    setsvalue2s(L, L->top, ts);
    s = getstr(ts);  # 内部复制的地址
  }
  api_incr_top(L);
  luaC_checkGC(L);
  lua_unlock(L);
  return s;
}
# 将格式化字符串推入栈中，使用可变参数列表
LUA_API const char *lua_pushvfstring (lua_State *L, const char *fmt, va_list argp) {
  const char *ret;
  lua_lock(L);  # 加锁
  ret = luaO_pushvfstring(L, fmt, argp);  # 调用内部函数将格式化字符串推入栈中
  luaC_checkGC(L);  # 检查垃圾回收
  lua_unlock(L);  # 解锁
  return ret;  # 返回结果
}

# 将格式化字符串推入栈中，使用可变参数列表
LUA_API const char *lua_pushfstring (lua_State *L, const char *fmt, ...) {
  const char *ret;
  va_list argp;
  lua_lock(L);  # 加锁
  va_start(argp, fmt);  # 初始化可变参数列表
  ret = luaO_pushvfstring(L, fmt, argp);  # 调用内部函数将格式化字符串推入栈中
  va_end(argp);  # 结束可变参数列表
  luaC_checkGC(L);  # 检查垃圾回收
  lua_unlock(L);  # 解锁
  return ret;  # 返回结果
}

# 将闭包推入栈中
LUA_API void lua_pushcclosure (lua_State *L, lua_CFunction fn, int n) {
  lua_lock(L);  # 加锁
  if (n == 0) {
    setfvalue(s2v(L->top), fn);  # 设置闭包值
    api_incr_top(L);  # 增加栈顶指针
  }
  else {
    CClosure *cl;
    api_checknelems(L, n);  # 检查栈中元素数量
    api_check(L, n <= MAXUPVAL, "upvalue index too large");  # 检查上值索引是否过大
    cl = luaF_newCclosure(L, n);  # 创建新的 C 闭包
    cl->f = fn;  # 设置闭包函数
    L->top -= n;  # 减少栈顶指针
    while (n--) {
      setobj2n(L, &cl->upvalue[n], s2v(L->top + n));  # 设置闭包的上值
      # 不需要屏障，因为闭包是白色的
      lua_assert(iswhite(cl));  # 断言闭包是白色的
    }
    setclCvalue(L, s2v(L->top), cl);  # 设置闭包的 C 值
    api_incr_top(L);  # 增加栈顶指针
    luaC_checkGC(L);  # 检查垃圾回收
  }
  lua_unlock(L);  # 解锁
}

# 将布尔值推入栈中
LUA_API void lua_pushboolean (lua_State *L, int b) {
  lua_lock(L);  # 加锁
  if (b)
    setbtvalue(s2v(L->top));  # 设置为真值
  else
    setbfvalue(s2v(L->top));  # 设置为假值
  api_incr_top(L);  # 增加栈顶指针
  lua_unlock(L);  # 解锁
}

# 将轻量级用户数据推入栈中
LUA_API void lua_pushlightuserdata (lua_State *L, void *p) {
  lua_lock(L);  # 加锁
  setpvalue(s2v(L->top), p);  # 设置指针值
  api_incr_top(L);  # 增加栈顶指针
  lua_unlock(L);  # 解锁
}

# 将线程推入栈中
LUA_API int lua_pushthread (lua_State *L) {
  lua_lock(L);  # 加锁
  setthvalue(L, s2v(L->top), L);  # 设置线程值
  api_incr_top(L);  # 增加栈顶指针
  lua_unlock(L);  # 解锁
  return (G(L)->mainthread == L);  # 返回是否为主线程
}

# 获取字符串（Lua -> 栈）
l_sinline int auxgetstr (lua_State *L, const TValue *t, const char *k) {
  const TValue *slot;
  TString *str = luaS_new(L, k);  # 创建新的字符串
  if (luaV_fastget(L, t, str, slot, luaH_getstr)) {  # 快速获取字符串
    setobj2s(L, L->top, slot);  # 设置对象到栈顶
    api_incr_top(L);  # 增加栈顶指针
  }
  else {
    setsvalue2s(L, L->top, str);  # 设置字符串到栈顶
    api_incr_top(L);  # 增加栈顶指针
  }
  # 调用 luaV_finishget 函数，完成从表 t 中获取键为 L->top - 1 的值，并将结果存储到 slot 中
  luaV_finishget(L, t, s2v(L->top - 1), L->top - 1, slot);
  # 结束 if 语句块
  }
  # 解锁 Lua 状态机
  lua_unlock(L);
  # 返回栈顶元素的类型
  return ttype(s2v(L->top - 1));
/*
** 获取注册表中的全局表。由于注册表中的所有预定义索引都是在创建注册表时插入的，并且从未被移除，它们必须始终存在于注册表的数组部分中。
*/
#define getGtable(L)  \
    (&hvalue(&G(L)->l_registry)->array[LUA_RIDX_GLOBALS - 1])

/*
** 获取全局变量的值并将其压入堆栈顶部
*/
LUA_API int lua_getglobal (lua_State *L, const char *name) {
  const TValue *G;
  lua_lock(L);
  G = getGtable(L);
  return auxgetstr(L, G, name);
}

/*
** 获取表中指定键的值并将其压入堆栈顶部
*/
LUA_API int lua_gettable (lua_State *L, int idx) {
  const TValue *slot;
  TValue *t;
  lua_lock(L);
  t = index2value(L, idx);
  if (luaV_fastget(L, t, s2v(L->top - 1), slot, luaH_get)) {
    setobj2s(L, L->top - 1, slot);
  }
  else
    luaV_finishget(L, t, s2v(L->top - 1), L->top - 1, slot);
  lua_unlock(L);
  return ttype(s2v(L->top - 1));
}

/*
** 获取表中指定字段的值并将其压入堆栈顶部
*/
LUA_API int lua_getfield (lua_State *L, int idx, const char *k) {
  lua_lock(L);
  return auxgetstr(L, index2value(L, idx), k);
}

/*
** 获取表中指定索引的值并将其压入堆栈顶部
*/
LUA_API int lua_geti (lua_State *L, int idx, lua_Integer n) {
  TValue *t;
  const TValue *slot;
  lua_lock(L);
  t = index2value(L, idx);
  if (luaV_fastgeti(L, t, n, slot)) {
    setobj2s(L, L->top, slot);
  }
  else {
    TValue aux;
    setivalue(&aux, n);
    luaV_finishget(L, t, &aux, L->top, slot);
  }
  api_incr_top(L);
  lua_unlock(L);
  return ttype(s2v(L->top - 1));
}

/*
** 完成原始获取操作并将结果压入堆栈顶部
*/
l_sinline int finishrawget (lua_State *L, const TValue *val) {
  if (isempty(val))  /* 避免将空项目复制到堆栈中 */
    setnilvalue(s2v(L->top));
  else
    setobj2s(L, L->top, val);
  api_incr_top(L);
  lua_unlock(L);
  return ttype(s2v(L->top - 1));
}

/*
** 获取指定索引处的表
*/
static Table *gettable (lua_State *L, int idx) {
  TValue *t = index2value(L, idx);
  api_check(L, ttistable(t), "table expected");
  return hvalue(t);
}
LUA_API int lua_rawget (lua_State *L, int idx) {
  Table *t;  // 声明 Table 类型的指针变量 t
  const TValue *val;  // 声明指向 TValue 类型常量的指针变量 val
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  t = gettable(L, idx);  // 获取栈中索引为 idx 的元素，并将其赋值给 t
  val = luaH_get(t, s2v(L->top - 1));  // 从表 t 中获取键为栈顶元素的值，并将其赋值给 val
  L->top--;  /* remove key */  // 栈顶指针减一，移除键
  return finishrawget(L, val);  // 调用 finishrawget 函数，返回结果
}


LUA_API int lua_rawgeti (lua_State *L, int idx, lua_Integer n) {
  Table *t;  // 声明 Table 类型的指针变量 t
  lua_lock(L);  // 锁定 Lua 状态
  t = gettable(L, idx);  // 获取栈中索引为 idx 的元素，并将其赋值给 t
  return finishrawget(L, luaH_getint(t, n));  // 调用 finishrawget 函数，返回结果
}


LUA_API int lua_rawgetp (lua_State *L, int idx, const void *p) {
  Table *t;  // 声明 Table 类型的指针变量 t
  TValue k;  // 声明 TValue 类型的变量 k
  lua_lock(L);  // 锁定 Lua 状态
  t = gettable(L, idx);  // 获取栈中索引为 idx 的元素，并将其赋值给 t
  setpvalue(&k, cast_voidp(p));  // 将指针 p 转换为 TValue 类型，并赋值给 k
  return finishrawget(L, luaH_get(t, &k));  // 调用 finishrawget 函数，返回结果
}


LUA_API void lua_createtable (lua_State *L, int narray, int nrec) {
  Table *t;  // 声明 Table 类型的指针变量 t
  lua_lock(L);  // 锁定 Lua 状态
  t = luaH_new(L);  // 创建一个新的表，并将其赋值给 t
  sethvalue2s(L, L->top, t);  // 将 t 转换为 TValue 类型，并放入栈顶
  api_incr_top(L);  // 栈顶指针加一
  if (narray > 0 || nrec > 0)
    luaH_resize(L, t, narray, nrec);  // 调整表的大小
  luaC_checkGC(L);  // 检查垃圾回收
  lua_unlock(L);  // 解锁 Lua 状态
}


LUA_API int lua_getmetatable (lua_State *L, int objindex) {
  const TValue *obj;  // 声明指向 TValue 类型常量的指针变量 obj
  Table *mt;  // 声明 Table 类型的指针变量 mt
  int res = 0;  // 声明并初始化整型变量 res 为 0
  lua_lock(L);  // 锁定 Lua 状态
  obj = index2value(L, objindex);  // 获取栈中索引为 objindex 的元素，并将其赋值给 obj
  switch (ttype(obj)) {  // 根据 obj 的类型进行判断
    case LUA_TTABLE:
      mt = hvalue(obj)->metatable;  // 如果 obj 类型为 LUA_TTABLE，则将其元表赋值给 mt
      break;
    case LUA_TUSERDATA:
      mt = uvalue(obj)->metatable;  // 如果 obj 类型为 LUA_TUSERDATA，则将其元表赋值给 mt
      break;
    default:
      mt = G(L)->mt[ttype(obj)];  // 其他情况下，将全局元表中对应类型的元表赋值给 mt
      break;
  }
  if (mt != NULL) {  // 如果 mt 不为空
    sethvalue2s(L, L->top, mt);  // 将 mt 转换为 TValue 类型，并放入栈顶
    api_incr_top(L);  // 栈顶指针加一
    res = 1;  // 将 res 设置为 1
  }
  lua_unlock(L);  // 解锁 Lua 状态
  return res;  // 返回 res
}


LUA_API int lua_getiuservalue (lua_State *L, int idx, int n) {
  TValue *o;  // 声明指向 TValue 类型的指针变量 o
  int t;  // 声明整型变量 t
  lua_lock(L);  // 锁定 Lua 状态
  o = index2value(L, idx);  // 获取栈中索引为 idx 的元素，并将其赋值给 o
  api_check(L, ttisfulluserdata(o), "full userdata expected");  // 检查 o 是否为完整的用户数据
  if (n <= 0 || n > uvalue(o)->nuvalue) {  // 如果 n 小于等于 0 或者大于用户数据的 nuvalue
    setnilvalue(s2v(L->top));  // 将空值放入栈顶
    t = LUA_TNONE;  // 将 t 设置为 LUA_TNONE
  }
  else {
    setobj2s(L, L->top, &uvalue(o)->uv[n - 1].uv);  // 将用户数据的第 n 个值放入栈顶
    t = ttype(s2v(L->top));  // 将栈顶元素的类型赋值给 t
  }
  api_incr_top(L);  // 栈顶指针加一
  lua_unlock(L);  // 解锁 Lua 状态
  return t;  // 返回 t
}


/*
** set functions (stack -> Lua)
*/

/*
** t[k] = value at the top of the stack (where 'k' is a string)
*/
// 辅助函数，用于设置字符串键值对到 Lua 表中
static void auxsetstr (lua_State *L, const TValue *t, const char *k) {
  const TValue *slot;  // 用于存储获取到的槽位
  TString *str = luaS_new(L, k);  // 创建一个新的字符串对象
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  if (luaV_fastget(L, t, str, slot, luaH_getstr)) {  // 尝试快速获取键对应的值
    luaV_finishfastset(L, t, slot, s2v(L->top - 1));  // 完成快速设置操作
    L->top--;  /* pop value */  // 弹出值
  }
  else {
    setsvalue2s(L, L->top, str);  /* push 'str' (to make it a TValue) */  // 将字符串推入栈顶
    api_incr_top(L);  // 增加栈顶指针
    luaV_finishset(L, t, s2v(L->top - 1), s2v(L->top - 2), slot);  // 完成设置操作
    L->top -= 2;  /* pop value and key */  // 弹出值和键
  }
  lua_unlock(L);  /* lock done by caller */  // 解锁，由调用者负责加锁
}

// 设置全局变量
LUA_API void lua_setglobal (lua_State *L, const char *name) {
  const TValue *G;  // 全局表
  lua_lock(L);  /* unlock done in 'auxsetstr' */  // 加锁，'auxsetstr' 函数中解锁
  G = getGtable(L);  // 获取全局表
  auxsetstr(L, G, name);  // 调用辅助函数设置字符串键值对到全局表中
}

// 设置表中的键值对
LUA_API void lua_settable (lua_State *L, int idx) {
  TValue *t;  // 表
  const TValue *slot;  // 用于存储获取到的槽位
  lua_lock(L);  // 加锁
  api_checknelems(L, 2);  // 检查栈中是否有足够的元素
  t = index2value(L, idx);  // 获取表
  if (luaV_fastget(L, t, s2v(L->top - 2), slot, luaH_get)) {  // 尝试快速获取键对应的值
    luaV_finishfastset(L, t, slot, s2v(L->top - 1));  // 完成快速设置操作
  }
  else
    luaV_finishset(L, t, s2v(L->top - 2), s2v(L->top - 1), slot);  // 完成设置操作
  L->top -= 2;  /* pop index and value */  // 弹出索引和值
  lua_unlock(L);  // 解锁
}

// 设置表中的字段
LUA_API void lua_setfield (lua_State *L, int idx, const char *k) {
  lua_lock(L);  /* unlock done in 'auxsetstr' */  // 加锁，'auxsetstr' 函数中解锁
  auxsetstr(L, index2value(L, idx), k);  // 调用辅助函数设置字符串键值对到表中
}

// 设置表中的整数键值对
LUA_API void lua_seti (lua_State *L, int idx, lua_Integer n) {
  TValue *t;  // 表
  const TValue *slot;  // 用于存储获取到的槽位
  lua_lock(L);  // 加锁
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  t = index2value(L, idx);  // 获取表
  if (luaV_fastgeti(L, t, n, slot)) {  // 尝试快速获取键对应的值
    luaV_finishfastset(L, t, slot, s2v(L->top - 1));  // 完成快速设置操作
  }
  else {
    TValue aux;
    setivalue(&aux, n);  // 设置整数值
    luaV_finishset(L, t, &aux, s2v(L->top - 1), slot);  // 完成设置操作
  }
  L->top--;  /* pop value */  // 弹出值
  lua_unlock(L);  // 解锁
}
static void aux_rawset (lua_State *L, int idx, TValue *key, int n) {
  Table *t;  // 创建一个 Table 指针 t
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, n);  // 检查栈中是否有足够的元素
  t = gettable(L, idx);  // 获取栈中索引为 idx 的表
  luaH_set(L, t, key, s2v(L->top - 1));  // 将键值对 key 和值 s2v(L->top - 1) 插入表 t 中
  invalidateTMcache(t);  // 使表 t 的 TM 缓存无效
  luaC_barrierback(L, obj2gco(t), s2v(L->top - 1));  // 在表 t 和值 s2v(L->top - 1) 之间建立一个弱引用
  L->top -= n;  // 减少栈顶元素数量
  lua_unlock(L);  // 解锁 Lua 状态
}

LUA_API void lua_rawset (lua_State *L, int idx) {
  aux_rawset(L, idx, s2v(L->top - 2), 2);  // 调用 aux_rawset 函数，将栈顶第二个元素作为键，栈顶元素作为值，插入到索引为 idx 的表中
}

LUA_API void lua_rawsetp (lua_State *L, int idx, const void *p) {
  TValue k;  // 创建一个 TValue 结构体 k
  setpvalue(&k, cast_voidp(p));  // 将指针 p 转换为 TValue 类型的值，并赋给 k
  aux_rawset(L, idx, &k, 1);  // 调用 aux_rawset 函数，将 k 作为键，栈顶元素作为值，插入到索引为 idx 的表中
}

LUA_API void lua_rawseti (lua_State *L, int idx, lua_Integer n) {
  Table *t;  // 创建一个 Table 指针 t
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  t = gettable(L, idx);  // 获取栈中索引为 idx 的表
  luaH_setint(L, t, n, s2v(L->top - 1));  // 将整数键 n 和值 s2v(L->top - 1) 插入表 t 中
  luaC_barrierback(L, obj2gco(t), s2v(L->top - 1));  // 在表 t 和值 s2v(L->top - 1) 之间建立一个弱引用
  L->top--;  // 减少栈顶元素数量
  lua_unlock(L);  // 解锁 Lua 状态
}

LUA_API int lua_setmetatable (lua_State *L, int objindex) {
  TValue *obj;  // 创建一个 TValue 指针 obj
  Table *mt;  // 创建一个 Table 指针 mt
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  obj = index2value(L, objindex);  // 获取栈中索引为 objindex 的值
  if (ttisnil(s2v(L->top - 1)))  // 如果栈顶元素为 nil
    mt = NULL;  // 将 mt 设为 NULL
  else {
    api_check(L, ttistable(s2v(L->top - 1)), "table expected");  // 检查栈顶元素是否为表
    mt = hvalue(s2v(L->top - 1));  // 获取栈顶元素对应的表
  }
  switch (ttype(obj)) {  // 根据 obj 的类型进行不同的处理
    case LUA_TTABLE: {  // 如果 obj 是表
      hvalue(obj)->metatable = mt;  // 将表的元表设为 mt
      if (mt) {
        luaC_objbarrier(L, gcvalue(obj), mt);  // 在 obj 和 mt 之间建立一个弱引用
        luaC_checkfinalizer(L, gcvalue(obj), mt);  // 检查 obj 和 mt 是否有 finalizer
      }
      break;
    }
    case LUA_TUSERDATA: {  // 如果 obj 是用户数据
      uvalue(obj)->metatable = mt;  // 将用户数据的元表设为 mt
      if (mt) {
        luaC_objbarrier(L, uvalue(obj), mt);  // 在 obj 和 mt 之间建立一个弱引用
        luaC_checkfinalizer(L, gcvalue(obj), mt);  // 检查 obj 和 mt 是否有 finalizer
      }
      break;
    }
    default: {  // 其它情况
      G(L)->mt[ttype(obj)] = mt;  // 将全局元表数组中对应类型的元表设为 mt
      break;
    }
  }
  L->top--;  // 减少栈顶元素数量
  lua_unlock(L);  // 解锁 Lua 状态
  return 1;  // 返回 1
}

LUA_API int lua_setiuservalue (lua_State *L, int idx, int n) {
  TValue *o;  // 创建一个 TValue 指针 o
  int res;  // 创建一个整型变量 res
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, 1);  // 检查栈中是否有足够的元素
  o = index2value(L, idx);  // 获取栈中索引为 idx 的值
  api_check(L, ttisfulluserdata(o), "full userdata expected");  // 检查值是否为完整的用户数据
  if (!(cast_uint(n) - 1u < cast_uint(uvalue(o)->nuvalue)))  // 如果 n 不在合法范围内
    res = 0;  /* 如果'n'不在[1, uvalue(o)->nuvalue]范围内，则设置res为0 */
  else {
    setobj(L, &uvalue(o)->uv[n - 1].uv, s2v(L->top - 1));  /* 否则，将栈顶元素设置为uvalue(o)->uv[n - 1].uv */
    luaC_barrierback(L, gcvalue(o), s2v(L->top - 1));  /* 执行垃圾回收的后向栅栏 */
    res = 1;  /* 设置res为1 */
  }
  L->top--;  /* 栈顶指针减一 */
  lua_unlock(L);  /* 解锁Lua状态机 */
  return res;  /* 返回res的值 */
/*
** 'load' and 'call' functions (run Lua code)
*/

// 定义宏，用于检查函数调用结果是否溢出当前堆栈大小
#define checkresults(L,na,nr) \
     api_check(L, (nr) == LUA_MULTRET || (L->ci->top - L->top >= (nr) - (na)), \
    "results from function overflow current stack size")

// 调用 Lua 函数
LUA_API void lua_callk (lua_State *L, int nargs, int nresults,
                        lua_KContext ctx, lua_KFunction k) {
  StkId func;
  lua_lock(L);
  // 检查是否在钩子函数内部使用了延续
  api_check(L, k == NULL || !isLua(L->ci),
    "cannot use continuations inside hooks");
  // 检查参数数量是否合法
  api_checknelems(L, nargs+1);
  // 检查线程状态是否正常
  api_check(L, L->status == LUA_OK, "cannot do calls on non-normal thread");
  // 检查函数调用结果是否溢出当前堆栈大小
  checkresults(L, nargs, nresults);
  // 获取函数栈指针
  func = L->top - (nargs+1);
  // 需要准备延续吗？
  if (k != NULL && yieldable(L)) {
    L->ci->u.c.k = k;  /* save continuation */
    L->ci->u.c.ctx = ctx;  /* save context */
    luaD_call(L, func, nresults);  /* do the call */
  }
  else  /* no continuation or no yieldable */
    luaD_callnoyield(L, func, nresults);  /* just do the call */
  // 调整结果
  adjustresults(L, nresults);
  lua_unlock(L);
}

/*
** Execute a protected call.
*/
// 'f_call' 函数的数据
struct CallS {  
  StkId func;
  int nresults;
};

// 执行 'f_call' 函数
static void f_call (lua_State *L, void *ud) {
  struct CallS *c = cast(struct CallS *, ud);
  luaD_callnoyield(L, c->func, c->nresults);
}

// 保护模式下的函数调用
LUA_API int lua_pcallk (lua_State *L, int nargs, int nresults, int errfunc,
                        lua_KContext ctx, lua_KFunction k) {
  struct CallS c;
  int status;
  ptrdiff_t func;
  lua_lock(L);
  // 检查是否在钩子函数内部使用了延续
  api_check(L, k == NULL || !isLua(L->ci),
    "cannot use continuations inside hooks");
  // 检查参数数量是否合法
  api_checknelems(L, nargs+1);
  // 检查线程状态是否正常
  api_check(L, L->status == LUA_OK, "cannot do calls on non-normal thread");
  // 检查函数调用结果是否溢出当前堆栈大小
  checkresults(L, nargs, nresults);
  // 如果错误处理函数为 0，则将其置为 0
  if (errfunc == 0)
    func = 0;
  else {
    StkId o = index2stack(L, errfunc);
    // 检查错误处理函数是否为函数类型
    api_check(L, ttisfunction(s2v(o)), "error handler must be a function");
  func = savestack(L, o);  # 保存栈顶指针，用于后续调用
}
c.func = L->top - (nargs+1);  # 设置要调用的函数
if (k == NULL || !yieldable(L)) {  # 如果没有继续或者不可暂停？
  c.nresults = nresults;  # 执行一个常规的受保护调用
  status = luaD_pcall(L, f_call, &c, savestack(L, c.func), func);  # 执行受保护的函数调用
}
else {  # 准备继续（调用已经由'resume'保护）
  CallInfo *ci = L->ci;  # 获取当前调用信息
  ci->u.c.k = k;  # 保存继续
  ci->u.c.ctx = ctx;  # 保存上下文
  /* 保存错误恢复信息 */
  ci->u2.funcidx = cast_int(savestack(L, c.func));  # 保存函数索引
  ci->u.c.old_errfunc = L->errfunc;  # 保存旧的错误处理函数
  L->errfunc = func;  # 设置新的错误处理函数
  setoah(ci->callstatus, L->allowhook);  # 保存'allowhook'的值
  ci->callstatus |= CIST_YPCALL;  # 函数可以进行错误恢复
  luaD_call(L, c.func, nresults);  # 执行调用
  ci->callstatus &= ~CIST_YPCALL;  # 清除错误恢复标志
  L->errfunc = ci->u.c.old_errfunc;  # 恢复旧的错误处理函数
  status = LUA_OK;  # 如果执行到这里，说明没有错误
}
adjustresults(L, nresults);  # 调整结果
lua_unlock(L);  # 解锁 Lua
return status;  # 返回状态
}

/*
** 从给定的读取器中加载 Lua 代码块
** 参数：
**     L：Lua 状态机
**     reader：读取器函数指针
**     data：读取器的数据指针
**     chunkname：代码块的名称
**     mode：模式
** 返回值：
**     加载状态
*/
LUA_API int lua_load (lua_State *L, lua_Reader reader, void *data,
                      const char *chunkname, const char *mode) {
  ZIO z;  // 创建 ZIO 结构体
  int status;  // 状态
  lua_lock(L);  // 锁定 Lua 状态机
  if (!chunkname) chunkname = "?";  // 如果代码块名称为空，则设置为 "?"
  luaZ_init(L, &z, reader, data);  // 初始化 ZIO 结构体
  status = luaD_protectedparser(L, &z, chunkname, mode);  // 保护模式下解析 Lua 代码
  if (status == LUA_OK) {  /* 没有错误？ */
    LClosure *f = clLvalue(s2v(L->top - 1));  /* 获取新创建的函数 */
    if (f->nupvalues >= 1) {  /* 是否有上值？ */
      /* 从注册表中获取全局表 */
      const TValue *gt = getGtable(L);
      /* 将全局表设置为 'f' 的第一个上值（可能是 LUA_ENV） */
      setobj(L, f->upvals[0]->v, gt);
      luaC_barrier(L, f->upvals[0], gt);
    }
  }
  lua_unlock(L);  // 解锁 Lua 状态机
  return status;  // 返回加载状态
}

/*
** 将 Lua 函数对象转换为二进制数据
** 参数：
**     L：Lua 状态机
**     writer：写入器函数指针
**     data：写入器的数据指针
**     strip：是否剥离调试信息
** 返回值：
**     转换状态
*/
LUA_API int lua_dump (lua_State *L, lua_Writer writer, void *data, int strip) {
  int status;  // 状态
  TValue *o;  // 值
  lua_lock(L);  // 锁定 Lua 状态机
  api_checknelems(L, 1);  // 检查堆栈中是否有足够的元素
  o = s2v(L->top - 1);  // 获取栈顶元素
  if (isLfunction(o))  // 如果是 Lua 函数对象
    status = luaU_dump(L, getproto(o), writer, data, strip);  // 转换为二进制数据
  else
    status = 1;  // 状态设置为 1
  lua_unlock(L);  // 解锁 Lua 状态机
  return status;  // 返回转换状态
}

/*
** 获取 Lua 状态机的状态
** 参数：
**     L：Lua 状态机
** 返回值：
**     状态
*/
LUA_API int lua_status (lua_State *L) {
  return L->status;  // 返回状态
}

/*
** 垃圾回收函数
** 参数：
**     L：Lua 状态机
**     what：操作类型
**     ...：可变参数
** 返回值：
**     操作结果
*/
LUA_API int lua_gc (lua_State *L, int what, ...) {
  va_list argp;  // 可变参数列表
  int res = 0;  // 结果
  global_State *g = G(L);  // 获取全局状态
  if (g->gcstp & GCSTPGC)  /* 内部停止？ */
    return -1;  /* 停止时所有选项都无效 */
  lua_lock(L);  // 锁定 Lua 状态机
  va_start(argp, what);  // 开始处理可变参数
  switch (what) {
    case LUA_GCSTOP: {
      g->gcstp = GCSTPUSR;  /* 用户停止 */
      break;
    }
    case LUA_GCRESTART: {
      luaE_setdebt(g, 0);
      g->gcstp = 0;  /* (GCSTPGC 必须已经为零) */
      break;
    }
    case LUA_GCCOLLECT: {
      luaC_fullgc(L, 0);  // 执行完整的垃圾回收
      break;
    }
    case LUA_GCCOUNT: {
      /* GC 值以 Kbytes 表示：#bytes/2^10 */
      res = cast_int(gettotalbytes(g) >> 10);  // 获取总字节数并转换为 Kbytes
      break;
    }
    case LUA_GCCOUNTB: {
      res = cast_int(gettotalbytes(g) & 0x3ff);  // 获取总字节数的低 10 位
      break;
    }
    case LUA_GCSTEP: {  # 处理 Lua 垃圾回收的步骤
      int data = va_arg(argp, int);  # 从参数列表中获取整型数据
      l_mem debt = 1;  /* =1 to signal that it did an actual step */  # 设置初始债务为1，表示实际执行了一步
      lu_byte oldstp = g->gcstp;  # 保存旧的垃圾回收步骤状态
      g->gcstp = 0;  /* allow GC to run (GCSTPGC must be zero here) */  # 设置垃圾回收步骤状态为0，允许垃圾回收运行
      if (data == 0) {  # 如果数据为0
        luaE_setdebt(g, 0);  /* do a basic step */  # 设置债务为0，执行基本步骤
        luaC_step(L);  # 执行 Lua 垃圾回收的一步
      }
      else {  /* add 'data' to total debt */  # 否则，将数据添加到总债务中
        debt = cast(l_mem, data) * 1024 + g->GCdebt;  # 计算新的债务
        luaE_setdebt(g, debt);  # 设置新的债务
        luaC_checkGC(L);  # 检查是否需要执行垃圾回收
      }
      g->gcstp = oldstp;  /* restore previous state */  # 恢复之前的垃圾回收步骤状态
      if (debt > 0 && g->gcstate == GCSpause)  /* end of cycle? */  # 如果债务大于0且垃圾回收状态为暂停
        res = 1;  /* signal it */  # 设置结果为1，表示结束循环
      break;  # 结束 case 语句块
    }
    case LUA_GCSETPAUSE: {  # 处理设置 Lua 垃圾回收暂停时间
      int data = va_arg(argp, int);  # 从参数列表中获取整型数据
      res = getgcparam(g->gcpause);  # 获取当前的垃圾回收暂停时间
      setgcparam(g->gcpause, data);  # 设置新的垃圾回收暂停时间
      break;  # 结束 case 语句块
    }
    case LUA_GCSETSTEPMUL: {  # 处理设置 Lua 垃圾回收步骤乘数
      int data = va_arg(argp, int);  # 从参数列表中获取整型数据
      res = getgcparam(g->gcstepmul);  # 获取当前的垃圾回收步骤乘数
      setgcparam(g->gcstepmul, data);  # 设置新的垃圾回收步骤乘数
      break;  # 结束 case 语句块
    }
    case LUA_GCISRUNNING: {  # 处理 Lua 垃圾回收是否正在运行
      res = gcrunning(g);  # 获取 Lua 垃圾回收是否正在运行的状态
      break;  # 结束 case 语句块
    }
    case LUA_GCGEN: {  # 处理 Lua 垃圾回收的生成模式
      int minormul = va_arg(argp, int);  # 从参数列表中获取整型数据
      int majormul = va_arg(argp, int);  # 从参数列表中获取整型数据
      res = isdecGCmodegen(g) ? LUA_GCGEN : LUA_GCINC;  # 根据当前的垃圾回收模式设置结果
      if (minormul != 0)  # 如果次要乘数不为0
        g->genminormul = minormul;  # 设置次要乘数
      if (majormul != 0)  # 如果主要乘数不为0
        setgcparam(g->genmajormul, majormul);  # 设置主要乘数
      luaC_changemode(L, KGC_GEN);  # 改变 Lua 垃圾回收的模式为生成模式
      break;  # 结束 case 语句块
    }
    case LUA_GCINC: {  # 处理 Lua 垃圾回收的增量模式
      int pause = va_arg(argp, int);  # 从参数列表中获取整型数据
      int stepmul = va_arg(argp, int);  # 从参数列表中获取整型数据
      int stepsize = va_arg(argp, int);  # 从参数列表中获取整型数据
      res = isdecGCmodegen(g) ? LUA_GCGEN : LUA_GCINC;  # 根据当前的垃圾回收模式设置结果
      if (pause != 0)  # 如果暂停时间不为0
        setgcparam(g->gcpause, pause);  # 设置暂停时间
      if (stepmul != 0)  # 如果步骤乘数不为0
        setgcparam(g->gcstepmul, stepmul);  # 设置步骤乘数
      if (stepsize != 0)  # 如果步骤大小不为0
        g->gcstepsize = stepsize;  # 设置步骤大小
      luaC_changemode(L, KGC_INC);  # 改变 Lua 垃圾回收的模式为增量模式
      break;  # 结束 case 语句块
    }
    default: res = -1;  /* invalid option */  # 默认情况下，设置结果为-1，表示无效选项
  }
  va_end(argp);  # 结束可变参数的使用
  lua_unlock(L);  # 解锁 Lua 状态
  return res;  # 返回结果
/*
** miscellaneous functions
*/

// 报错函数，接收一个 lua_State 指针，返回一个整型值
LUA_API int lua_error (lua_State *L) {
  TValue *errobj;  // 定义一个 TValue 指针 errobj
  lua_lock(L);  // 锁定 Lua 状态
  errobj = s2v(L->top - 1);  // 将栈顶元素的值转换为 TValue 类型，赋给 errobj
  api_checknelems(L, 1);  // 检查栈中元素数量是否为 1
  /* error object is the memory error message? */
  if (ttisshrstring(errobj) && eqshrstr(tsvalue(errobj), G(L)->memerrmsg))  // 如果错误对象是内存错误消息
    luaM_error(L);  // 抛出内存错误
  else
    luaG_errormsg(L);  // 抛出常规错误
  /* code unreachable; will unlock when control actually leaves the kernel */
  return 0;  // 返回 0，避免警告
}

// 获取表中的下一个键值对，接收一个 lua_State 指针和一个整型值，返回一个整型值
LUA_API int lua_next (lua_State *L, int idx) {
  Table *t;  // 定义一个 Table 指针 t
  int more;  // 定义一个整型变量 more
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, 1);  // 检查栈中元素数量是否为 1
  t = gettable(L, idx);  // 获取表
  more = luaH_next(L, t, L->top - 1);  // 获取下一个键值对
  if (more) {
    api_incr_top(L);  // 栈顶指针向上移动
  }
  else  // 没有更多元素
    L->top -= 1;  // 移除键
  lua_unlock(L);  // 解锁 Lua 状态
  return more;  // 返回 more
}

// 标记要关闭的 upvalue，接收一个 lua_State 指针和一个整型值
LUA_API void lua_toclose (lua_State *L, int idx) {
  int nresults;  // 定义一个整型变量 nresults
  StkId o;  // 定义一个 StkId 变量 o
  lua_lock(L);  // 锁定 Lua 状态
  o = index2stack(L, idx);  // 将索引转换为栈指针
  nresults = L->ci->nresults;  // 获取结果数量
  api_check(L, L->tbclist < o, "given index below or equal a marked one");  // 检查索引是否小于标记的索引
  luaF_newtbcupval(L, o);  // 创建新的待关闭的 upvalue
  if (!hastocloseCfunc(nresults))  // 函数还未标记？
    L->ci->nresults = codeNresults(nresults);  // 标记函数
  lua_assert(hastocloseCfunc(L->ci->nresults));  // 断言函数已标记
  lua_unlock(L);  // 解锁 Lua 状态
}

// 连接栈顶的 n 个值，接收一个 lua_State 指针和一个整型值
LUA_API void lua_concat (lua_State *L, int n) {
  lua_lock(L);  // 锁定 Lua 状态
  api_checknelems(L, n);  // 检查栈中元素数量是否为 n
  if (n > 0)
    luaV_concat(L, n);  // 连接 n 个值
  else {  // 没有要连接的值
    setsvalue2s(L, L->top, luaS_newlstr(L, "", 0));  // 推入空字符串
    api_incr_top(L);  // 栈顶指针向上移动
  }
  luaC_checkGC(L);  // 检查垃圾回收
  lua_unlock(L);  // 解锁 Lua 状态
}

// 获取栈中值的长度，接收一个 lua_State 指针和一个整型值
LUA_API void lua_len (lua_State *L, int idx) {
  TValue *t;  // 定义一个 TValue 指针 t
  lua_lock(L);  // 锁定 Lua 状态
  t = index2value(L, idx);  // 获取索引对应的值
  luaV_objlen(L, L->top, t);  // 获取对象的长度
  api_incr_top(L);  // 栈顶指针向上移动
  lua_unlock(L);  // 解锁 Lua 状态
}

// 获取内存分配器函数，接收一个 lua_State 指针和一个指向指针的指针，返回一个 lua_Alloc 函数指针
LUA_API lua_Alloc lua_getallocf (lua_State *L, void **ud) {
  lua_Alloc f;  // 定义一个 lua_Alloc 函数指针 f
  lua_lock(L);  // 锁定 Lua 状态
  if (ud) *ud = G(L)->ud;  // 如果 ud 不为空，将 G(L)->ud 赋给它
  f = G(L)->frealloc;  // 获取内存分配器函数
  lua_unlock(L);  // 解锁 Lua 状态
  return f;  // 返回内存分配器函数指针
}
# 设置 Lua 的内存分配函数
LUA_API void lua_setallocf (lua_State *L, lua_Alloc f, void *ud) {
  # 加锁，确保线程安全
  lua_lock(L);
  # 设置全局状态机的用户数据
  G(L)->ud = ud;
  # 设置全局状态机的内存分配函数
  G(L)->frealloc = f;
  # 解锁
  lua_unlock(L);
}

# 设置 Lua 的警告函数
void lua_setwarnf (lua_State *L, lua_WarnFunction f, void *ud) {
  # 加锁，确保线程安全
  lua_lock(L);
  # 设置全局状态机的警告函数的用户数据
  G(L)->ud_warn = ud;
  # 设置全局状态机的警告函数
  G(L)->warnf = f;
  # 解锁
  lua_unlock(L);
}

# 发出 Lua 警告
void lua_warning (lua_State *L, const char *msg, int tocont) {
  # 加锁，确保线程安全
  lua_lock(L);
  # 调用 luaE_warning 函数发出警告
  luaE_warning(L, msg, tocont);
  # 解锁
  lua_unlock(L);
}

# 创建一个带有用户数据的新对象
LUA_API void *lua_newuserdatauv (lua_State *L, size_t size, int nuvalue) {
  Udata *u;
  # 加锁，确保线程安全
  lua_lock(L);
  # 检查 nuvalue 是否在有效范围内
  api_check(L, 0 <= nuvalue && nuvalue < USHRT_MAX, "invalid value");
  # 创建新的用户数据对象
  u = luaS_newudata(L, size, nuvalue);
  # 设置栈顶元素为新创建的用户数据对象
  setuvalue(L, s2v(L->top), u);
  # 增加栈顶指针
  api_incr_top(L);
  # 检查垃圾回收
  luaC_checkGC(L);
  # 解锁
  lua_unlock(L);
  # 返回用户数据对象的内存地址
  return getudatamem(u);
}

# 辅助函数，用于获取闭包的上值
static const char *aux_upvalue (TValue *fi, int n, TValue **val, GCObject **owner) {
  switch (ttypetag(fi)) {
    case LUA_VCCL: {  /* C closure */
      CClosure *f = clCvalue(fi);
      if (!(cast_uint(n) - 1u < cast_uint(f->nupvalues)))
        return NULL;  /* 'n' not in [1, f->nupvalues] */
      *val = &f->upvalue[n-1];
      if (owner) *owner = obj2gco(f);
      return "";
    }
    case LUA_VLCL: {  /* Lua closure */
      LClosure *f = clLvalue(fi);
      TString *name;
      Proto *p = f->p;
      if (!(cast_uint(n) - 1u  < cast_uint(p->sizeupvalues)))
        return NULL;  /* 'n' not in [1, p->sizeupvalues] */
      *val = f->upvals[n-1]->v;
      if (owner) *owner = obj2gco(f->upvals[n - 1]);
      name = p->upvalues[n-1].name;
      return (name == NULL) ? "(no name)" : getstr(name);
    }
    default: return NULL;  /* not a closure */
  }
}

# 获取闭包的上值
LUA_API const char *lua_getupvalue (lua_State *L, int funcindex, int n) {
  const char *name;
  TValue *val = NULL;  /* to avoid warnings */
  # 加锁，确保线程安全
  lua_lock(L);
  # 调用辅助函数获取闭包的上值
  name = aux_upvalue(index2value(L, funcindex), n, &val, NULL);
  # 如果获取成功，将值设置为栈顶元素
  if (name) {
    setobj2s(L, L->top, val);
    api_incr_top(L);
  }
  # 解锁
  lua_unlock(L);
  # 返回闭包的上值名称
  return name;
}
# 设置 Lua 函数的 upvalue
LUA_API const char *lua_setupvalue (lua_State *L, int funcindex, int n) {
  const char *name;  # 声明一个字符串指针变量 name
  TValue *val = NULL;  /* to avoid warnings */  # 声明一个 TValue 指针变量 val，并初始化为 NULL
  GCObject *owner = NULL;  /* to avoid warnings */  # 声明一个 GCObject 指针变量 owner，并初始化为 NULL
  TValue *fi;  # 声明一个 TValue 指针变量 fi
  lua_lock(L);  # 锁定 Lua 状态
  fi = index2value(L, funcindex);  # 获取 funcindex 对应的值，并赋给 fi
  api_checknelems(L, 1);  # 检查 Lua 栈中是否有足够的元素
  name = aux_upvalue(fi, n, &val, &owner);  # 调用 aux_upvalue 函数，获取 upvalue 的名称，并更新 val 和 owner
  if (name) {  # 如果 name 存在
    L->top--;  # 减少栈顶指针
    setobj(L, val, s2v(L->top));  # 设置 val 的值为栈顶的值
    luaC_barrier(L, owner, val);  # 执行垃圾回收屏障
  }
  lua_unlock(L);  # 解锁 Lua 状态
  return name;  # 返回 upvalue 的名称
}


# 获取 Lua 函数的 upvalue 引用
static UpVal **getupvalref (lua_State *L, int fidx, int n, LClosure **pf) {
  static const UpVal *const nullup = NULL;  # 声明一个静态的 UpVal 指针变量 nullup，并初始化为 NULL
  LClosure *f;  # 声明一个 LClosure 指针变量 f
  TValue *fi = index2value(L, fidx);  # 获取 fidx 对应的值，并赋给 fi
  api_check(L, ttisLclosure(fi), "Lua function expected");  # 检查 fi 是否为 Lua 函数
  f = clLvalue(fi);  # 获取 fi 对应的 LClosure，并赋给 f
  if (pf) *pf = f;  # 如果 pf 存在，则将 f 赋给 *pf
  if (1 <= n && n <= f->p->sizeupvalues)
    return &f->upvals[n - 1];  /* get its upvalue pointer */  # 如果 n 在有效范围内，则返回对应的 upvalue 指针
  else
    return (UpVal**)&nullup;  # 否则返回 nullup 的地址
}


# 获取 Lua 函数的 upvalue 的 ID
LUA_API void *lua_upvalueid (lua_State *L, int fidx, int n) {
  TValue *fi = index2value(L, fidx);  # 获取 fidx 对应的值，并赋给 fi
  switch (ttypetag(fi)) {  # 根据 fi 的类型标签进行判断
    case LUA_VLCL: {  /* lua closure */  # 如果是 Lua 闭包
      return *getupvalref(L, fidx, n, NULL);  # 返回对应的 upvalue 引用
    }
    case LUA_VCCL: {  /* C closure */  # 如果是 C 闭包
      CClosure *f = clCvalue(fi);  # 获取 fi 对应的 CClosure，并赋给 f
      if (1 <= n && n <= f->nupvalues)
        return &f->upvalue[n - 1];  # 如果 n 在有效范围内，则返回对应的 upvalue 的地址
      /* else */
    }  /* FALLTHROUGH */
    case LUA_VLCF:
      return NULL;  /* light C functions have no upvalues */  # 如果是轻量级的 C 函数，则返回 NULL
    default: {
      api_check(L, 0, "function expected");  # 检查是否为函数
      return NULL;  # 返回 NULL
    }
  }
}


# 合并两个 Lua 函数的 upvalue
LUA_API void lua_upvaluejoin (lua_State *L, int fidx1, int n1,
                                            int fidx2, int n2) {
  LClosure *f1;  # 声明一个 LClosure 指针变量 f1
  UpVal **up1 = getupvalref(L, fidx1, n1, &f1);  # 获取 fidx1 对应的 upvalue 引用，并赋给 up1
  UpVal **up2 = getupvalref(L, fidx2, n2, NULL);  # 获取 fidx2 对应的 upvalue 引用，并赋给 up2
  api_check(L, *up1 != NULL && *up2 != NULL, "invalid upvalue index");  # 检查 up1 和 up2 是否有效
  *up1 = *up2;  # 将 up2 的值赋给 up1
  luaC_objbarrier(L, f1, *up1);  # 执行对象屏障
}
```