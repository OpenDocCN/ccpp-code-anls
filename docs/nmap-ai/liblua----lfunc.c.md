# `nmap\liblua\lfunc.c`

```cpp
/*
** $Id: lfunc.c $
** 辅助函数，用于操作原型和闭包
** 请参阅 lua.h 中的版权声明
*/

#define lfunc_c
#define LUA_CORE

#include "lprefix.h"


#include <stddef.h>

#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lgc.h"
#include "lmem.h"
#include "lobject.h"
#include "lstate.h"



CClosure *luaF_newCclosure (lua_State *L, int nupvals) {
  // 创建一个新的闭包对象，类型为 C 闭包，包含 nupvals 个上值
  GCObject *o = luaC_newobj(L, LUA_VCCL, sizeCclosure(nupvals));
  CClosure *c = gco2ccl(o);
  c->nupvalues = cast_byte(nupvals);
  return c;
}


LClosure *luaF_newLclosure (lua_State *L, int nupvals) {
  // 创建一个新的闭包对象，类型为 Lua 闭包，包含 nupvals 个上值
  GCObject *o = luaC_newobj(L, LUA_VLCL, sizeLclosure(nupvals));
  LClosure *c = gco2lcl(o);
  c->p = NULL;
  c->nupvalues = cast_byte(nupvals);
  while (nupvals--) c->upvals[nupvals] = NULL;
  return c;
}


/*
** 用新的闭合上值填充闭包
*/
void luaF_initupvals (lua_State *L, LClosure *cl) {
  int i;
  for (i = 0; i < cl->nupvalues; i++) {
    // 创建一个新的上值对象，类型为上值，大小为 UpVal 结构的大小
    GCObject *o = luaC_newobj(L, LUA_VUPVAL, sizeof(UpVal));
    UpVal *uv = gco2upv(o);
    uv->v = &uv->u.value;  /* 使其封闭 */
    setnilvalue(uv->v);
    cl->upvals[i] = uv;
    luaC_objbarrier(L, cl, uv);
  }
}


/*
** 在给定级别创建一个新的上值，并将其链接到 'L' 的打开上值列表中的 'prev' 条目之后
**/
static UpVal *newupval (lua_State *L, int tbc, StkId level, UpVal **prev) {
  // 创建一个新的上值对象，类型为上值，大小为 UpVal 结构的大小
  GCObject *o = luaC_newobj(L, LUA_VUPVAL, sizeof(UpVal));
  UpVal *uv = gco2upv(o);
  UpVal *next = *prev;
  uv->v = s2v(level);  /* 当前值存在于堆栈中 */
  uv->tbc = tbc;
  uv->u.open.next = next;  /* 将其链接到打开上值列表 */
  uv->u.open.previous = prev;
  if (next)
    next->u.open.previous = &uv->u.open.next;
  *prev = uv;
  if (!isintwups(L)) {  /* 线程不在具有上值的线程列表中？ */
    L->twups = G(L)->twups;  /* 将其链接到列表中 */
    G(L)->twups = L;
  }
  return uv;
}
/*
** 在给定级别查找上值。
*/
UpVal *luaF_findupval (lua_State *L, StkId level) {
  UpVal **pp = &L->openupval;  /* 指向开放的上值链表的指针 */
  UpVal *p;  /* 上值指针 */
  lua_assert(isintwups(L) || L->openupval == NULL);  /* 断言，确保在正确的线程中或者开放的上值链表为空 */
  while ((p = *pp) != NULL && uplevel(p) >= level) {  /* 循环查找上值 */
    lua_assert(!isdead(G(L), p));  /* 断言，确保上值不是死的 */
    if (uplevel(p) == level)  /* 是否是对应的上值？ */
      return p;  /* 返回上值 */
    pp = &p->u.open.next;  /* 更新指向下一个上值的指针 */
  }
  /* 没有找到：在 'pp' 之后创建一个新的上值 */
  return newupval(L, 0, level, pp);  /* 创建新的上值 */
}


/*
** 调用对象 'obj' 的关闭方法，使用错误消息 'err'。布尔值 'yy' 控制调用是否可中断。
** (此函数假设有额外的堆栈空间。)
*/
static void callclosemethod (lua_State *L, TValue *obj, TValue *err, int yy) {
  StkId top = L->top;  /* 保存当前栈顶位置 */
  const TValue *tm = luaT_gettmbyobj(L, obj, TM_CLOSE);  /* 获取对象的关闭元方法 */
  setobj2s(L, top, tm);  /* 将元方法设置到栈顶，将调用元方法 */
  setobj2s(L, top + 1, obj);  /* 将 'self' 作为第一个参数设置到栈顶+1 */
  setobj2s(L, top + 2, err);  /* 将错误消息作为第二个参数设置到栈顶+2 */
  L->top = top + 3;  /* 添加函数和参数 */
  if (yy)
    luaD_call(L, top, 0);  /* 调用函数，可中断 */
  else
    luaD_callnoyield(L, top, 0);  /* 调用函数，不可中断 */
}


/*
** 检查给定级别的对象是否有关闭元方法，如果没有则报错。
*/
static void checkclosemth (lua_State *L, StkId level) {
  const TValue *tm = luaT_gettmbyobj(L, s2v(level), TM_CLOSE);  /* 获取对象的关闭元方法 */
  if (ttisnil(tm)) {  /* 没有元方法？ */
    int idx = cast_int(level - L->ci->func);  /* 变量索引 */
    const char *vname = luaG_findlocal(L, L->ci, idx, NULL);  /* 查找局部变量的名称 */
    if (vname == NULL) vname = "?";
    luaG_runerror(L, "variable '%s' got a non-closable value", vname);  /* 报错，变量得到了一个不可关闭的值 */
  }
}


/*
** 准备并调用一个关闭方法。
** 如果状态是 CLOSEKTOP，则调用关闭方法将被推送到堆栈的顶部。否则，值可以被推送到关闭上值的 'level' 之后，因为之后的值不会再被使用。
*/
static void prepcallclosemth (lua_State *L, StkId level, int status, int yy) {
  TValue *uv = s2v(level);  /* 获取将要关闭的值 */
  TValue *errobj;
  if (status == CLOSEKTOP)
    errobj = &G(L)->nilvalue;  /* 错误对象为空 */
  else {  /* 'luaD_seterrorobj' 将把栈顶设置为 level + 2 */
    errobj = s2v(level + 1);  /* 错误对象在 'uv' 之后 */
    luaD_seterrorobj(L, status, level + 1);  /* 设置错误对象 */
  }
  callclosemethod(L, uv, errobj, yy);
}


/*
** 'tbclist' 中增量的最大值，取决于增量的类型。
** (这个宏假设在使用时有一个 'L' 在作用域中。)
*/
#define MAXDELTA  \
    ((256ul << ((sizeof(L->stack->tbclist.delta) - 1) * 8)) - 1)


/*
** 在将要关闭的变量列表中插入一个变量。
*/
void luaF_newtbcupval (lua_State *L, StkId level) {
  lua_assert(level > L->tbclist);
  if (l_isfalse(s2v(level)))
    return;  /* false 不需要被关闭 */
  checkclosemth(L, level);  /* 值必须有一个关闭方法 */
  while (cast_uint(level - L->tbclist) > MAXDELTA) {
    L->tbclist += MAXDELTA;  /* 在最大增量处创建一个虚拟节点 */
    L->tbclist->tbclist.delta = 0;
  }
  level->tbclist.delta = cast(unsigned short, level - L->tbclist);
  L->tbclist = level;
}


void luaF_unlinkupval (UpVal *uv) {
  lua_assert(upisopen(uv));
  *uv->u.open.previous = uv->u.open.next;
  if (uv->u.open.next)
    uv->u.open.next->u.open.previous = uv->u.open.previous;
}


/*
** 关闭直到给定栈级别的所有 upvalue。
*/
void luaF_closeupval (lua_State *L, StkId level) {
  UpVal *uv;
  StkId upl;  /* 'uv' 指向的栈索引 */
  while ((uv = L->openupval) != NULL && (upl = uplevel(uv)) >= level) {
    TValue *slot = &uv->u.value;  /* 值的新位置 */
    lua_assert(uplevel(uv) < L->top);
    luaF_unlinkupval(uv);  /* 从 'openupval' 列表中移除 upvalue */
    setobj(L, slot, uv->v);  /* 将值移动到 upvalue 位置 */
    uv->v = slot;  /* 现在当前值存在于这里 */
    # 如果上值既不是白色也不是死亡状态
    if (!iswhite(uv)) {  /* neither white nor dead? */
      # 将上值标记为黑色，表示已经关闭的上值不能是灰色
      nw2black(uv);  /* closed upvalues cannot be gray */
      # 在 Lua 垃圾回收器中设置屏障，确保上值不会被回收
      luaC_barrier(L, uv, slot);
    }
  }
/*
** Remove firt element from the tbclist plus its dummy nodes.
*/
static void poptbclist (lua_State *L) {
  // 从 tbclist 中移除第一个元素以及其虚拟节点
  StkId tbc = L->tbclist;
  lua_assert(tbc->tbclist.delta > 0);  /* first element cannot be dummy */  // 断言第一个元素不能是虚拟节点
  tbc -= tbc->tbclist.delta;
  while (tbc > L->stack && tbc->tbclist.delta == 0)
    tbc -= MAXDELTA;  /* remove dummy nodes */  // 移除虚拟节点
  L->tbclist = tbc;
}


/*
** Close all upvalues and to-be-closed variables up to the given stack
** level.
*/
void luaF_close (lua_State *L, StkId level, int status, int yy) {
  ptrdiff_t levelrel = savestack(L, level);  // 保存栈的状态
  luaF_closeupval(L, level);  /* first, close the upvalues */  // 首先关闭 upvalues
  while (L->tbclist >= level) {  /* traverse tbc's down to that level */  // 遍历 tbc 直到达到指定的栈级别
    StkId tbc = L->tbclist;  /* get variable index */  // 获取变量索引
    poptbclist(L);  /* remove it from list */  // 从列表中移除变量
    prepcallclosemth(L, tbc, status, yy);  /* close variable */  // 准备关闭变量
    level = restorestack(L, levelrel);  // 恢复栈的状态
  }
}


Proto *luaF_newproto (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_VPROTO, sizeof(Proto));  // 创建新的 Proto 对象
  Proto *f = gco2p(o);  // 将 GCObject 转换为 Proto
  // 初始化 Proto 对象的各个字段
  f->k = NULL;
  f->sizek = 0;
  f->p = NULL;
  f->sizep = 0;
  f->code = NULL;
  f->sizecode = 0;
  f->lineinfo = NULL;
  f->sizelineinfo = 0;
  f->abslineinfo = NULL;
  f->sizeabslineinfo = 0;
  f->upvalues = NULL;
  f->sizeupvalues = 0;
  f->numparams = 0;
  f->is_vararg = 0;
  f->maxstacksize = 0;
  f->locvars = NULL;
  f->sizelocvars = 0;
  f->linedefined = 0;
  f->lastlinedefined = 0;
  f->source = NULL;
  return f;  // 返回 Proto 对象
}


void luaF_freeproto (lua_State *L, Proto *f) {
  // 释放 Proto 对象的各个字段所占用的内存
  luaM_freearray(L, f->code, f->sizecode);
  luaM_freearray(L, f->p, f->sizep);
  luaM_freearray(L, f->k, f->sizek);
  luaM_freearray(L, f->lineinfo, f->sizelineinfo);
  luaM_freearray(L, f->abslineinfo, f->sizeabslineinfo);
  luaM_freearray(L, f->locvars, f->sizelocvars);
  luaM_freearray(L, f->upvalues, f->sizeupvalues);
  luaM_free(L, f);  // 释放 Proto 对象
}


/*
** Look for n-th local variable at line 'line' in function 'func'.
** Returns NULL if not found.
*/
// 获取指定函数中指定局部变量的名称
const char *luaF_getlocalname (const Proto *f, int local_number, int pc) {
  int i;
  // 遍历局部变量数组，查找指定局部变量的名称
  for (i = 0; i<f->sizelocvars && f->locvars[i].startpc <= pc; i++) {
    // 判断变量是否在活动状态
    if (pc < f->locvars[i].endpc) {  /* is variable active? */
      // 减少局部变量编号，直到找到指定编号的局部变量
      local_number--;
      if (local_number == 0)
        // 返回局部变量的名称
        return getstr(f->locvars[i].varname);
    }
  }
  // 未找到指定编号的局部变量，返回空指针
  return NULL;  /* not found */
}
```