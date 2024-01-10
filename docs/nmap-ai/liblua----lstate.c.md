# `nmap\liblua\lstate.c`

```
/*
** $Id: lstate.c $
** 全局状态
** 请参阅 lua.h 中的版权声明
*/

#define lstate_c
#define LUA_CORE

#include "lprefix.h"


#include <stddef.h>
#include <string.h>

#include "lua.h"

#include "lapi.h"
#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lgc.h"
#include "llex.h"
#include "lmem.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "ltm.h"



/*
** 线程状态 + 额外空间
*/
typedef struct LX {
  lu_byte extra_[LUA_EXTRASPACE];
  lua_State l;
} LX;


/*
** 主线程结合了线程状态和全局状态
*/
typedef struct LG {
  LX l;
  global_State g;
} LG;



#define fromstate(L)    (cast(LX *, cast(lu_byte *, (L)) - offsetof(LX, l)))


/*
** 用于在创建状态时创建一个“随机”种子的宏；
** 该种子用于随机化字符串哈希值。
*/
#if !defined(luai_makeseed)

#include <time.h>

/*
** 使用一定程度的随机性计算初始种子。
** 依赖于地址空间布局随机化（如果存在）和当前时间。
*/
#define addbuff(b,p,e) \
  { size_t t = cast_sizet(e); \
    memcpy(b + p, &t, sizeof(t)); p += sizeof(t); }

static unsigned int luai_makeseed (lua_State *L) {
  char buff[3 * sizeof(size_t)];
  unsigned int h = cast_uint(time(NULL));
  int p = 0;
  addbuff(buff, p, L);  /* 堆变量 */
  addbuff(buff, p, &h);  /* 局部变量 */
  addbuff(buff, p, &lua_newstate);  /* 公共函数 */
  lua_assert(p == sizeof(buff));
  return luaS_hash(buff, p, h);
}

#endif


/*
** 将 GCdebt 设置为新值，保持值（totalbytes + GCdebt）
** 不变（并避免 'totalbytes' 下溢）
*/
void luaE_setdebt (global_State *g, l_mem debt) {
  l_mem tb = gettotalbytes(g);
  lua_assert(tb > 0);
  if (debt < tb - MAX_LMEM)
    debt = tb - MAX_LMEM;  /* 将使 'totalbytes == MAX_LMEM' */
  g->totalbytes = tb - debt;
  g->GCdebt = debt;
}
# 设置 Lua 调用栈的大小限制
LUA_API int lua_setcstacklimit (lua_State *L, unsigned int limit) {
  UNUSED(L); UNUSED(limit);
  return LUAI_MAXCCALLS;  /* 返回最大调用次数 */
}


# 扩展调用信息结构
CallInfo *luaE_extendCI (lua_State *L) {
  CallInfo *ci;  # 声明调用信息结构
  lua_assert(L->ci->next == NULL);  # 断言当前调用信息结构的下一个为空
  ci = luaM_new(L, CallInfo);  # 为调用信息结构分配内存
  lua_assert(L->ci->next == NULL);  # 断言当前调用信息结构的下一个为空
  L->ci->next = ci;  # 设置当前调用信息结构的下一个为新分配的调用信息结构
  ci->previous = L->ci;  # 设置新分配的调用信息结构的前一个为当前调用信息结构
  ci->next = NULL;  # 设置新分配的调用信息结构的下一个为空
  ci->u.l.trap = 0;  # 设置新分配的调用信息结构的陷阱标志为 0
  L->nci++;  # 调用信息结构计数加一
  return ci;  # 返回新分配的调用信息结构
}


# 释放未使用的调用信息结构
void luaE_freeCI (lua_State *L) {
  CallInfo *ci = L->ci;  # 获取当前调用信息结构
  CallInfo *next = ci->next;  # 获取当前调用信息结构的下一个
  ci->next = NULL;  # 设置当前调用信息结构的下一个为空
  while ((ci = next) != NULL) {  # 循环直到当前调用信息结构为空
    next = ci->next;  # 获取下一个调用信息结构
    luaM_free(L, ci);  # 释放当前调用信息结构的内存
    L->nci--;  # 调用信息结构计数减一
  }
}


# 缩小未使用的调用信息结构
void luaE_shrinkCI (lua_State *L) {
  CallInfo *ci = L->ci->next;  # 获取第一个空闲的调用信息结构
  CallInfo *next;  # 声明下一个调用信息结构
  if (ci == NULL)
    return;  # 如果没有额外的元素，则返回
  while ((next = ci->next) != NULL) {  # 循环直到下一个调用信息结构为空
    CallInfo *next2 = next->next;  # 获取下一个调用信息结构的下一个
    ci->next = next2;  # 从列表中移除下一个调用信息结构
    L->nci--;  # 调用信息结构计数减一
    luaM_free(L, next);  # 释放下一个调用信息结构的内存
    if (next2 == NULL)
      break;  # 如果没有更多的元素，则跳出循环
    else {
      next2->previous = ci;  # 设置下一个调用信息结构的前一个为当前调用信息结构
      ci = next2;  # 继续循环
    }
  }
}


# 检查 Lua 调用栈是否溢出
void luaE_checkcstack (lua_State *L) {
  if (getCcalls(L) == LUAI_MAXCCALLS)
    luaG_runerror(L, "C stack overflow");  # 如果调用次数达到最大值，则抛出栈溢出错误
  else if (getCcalls(L) >= (LUAI_MAXCCALLS / 10 * 11))
    luaD_throw(L, LUA_ERRERR);  # 如果调用次数大于最大值的十分之一乘以十一，则抛出错误
}


# 增加 Lua 调用栈的调用次数
LUAI_FUNC void luaE_incCstack (lua_State *L) {
  L->nCcalls++;  # 调用次数加一
  if (l_unlikely(getCcalls(L) >= LUAI_MAXCCALLS))
    luaE_checkcstack(L);  # 如果调用次数大于等于最大值，则检查调用栈
}
static void stack_init (lua_State *L1, lua_State *L) {
  int i; CallInfo *ci;
  /* initialize stack array */
  // 为堆栈数组分配内存空间
  L1->stack = luaM_newvector(L, BASIC_STACK_SIZE + EXTRA_STACK, StackValue);
  // 设置 tbclist 指向堆栈数组
  L1->tbclist = L1->stack;
  // 遍历堆栈数组，将每个元素设置为 nil
  for (i = 0; i < BASIC_STACK_SIZE + EXTRA_STACK; i++)
    setnilvalue(s2v(L1->stack + i));  /* erase new stack */
  // 设置 top 指向堆栈数组的起始位置
  L1->top = L1->stack;
  // 设置 stack_last 指向堆栈数组的末尾位置
  L1->stack_last = L1->stack + BASIC_STACK_SIZE;
  // 初始化第一个 CallInfo 结构
  ci = &L1->base_ci;
  ci->next = ci->previous = NULL;
  ci->callstatus = CIST_C;
  ci->func = L1->top;
  ci->u.c.k = NULL;
  ci->nresults = 0;
  // 设置 top 指向的值为 nil
  setnilvalue(s2v(L1->top));  /* 'function' entry for this 'ci' */
  L1->top++;
  // 设置 ci->top 指向 top 之后的位置
  ci->top = L1->top + LUA_MINSTACK;
  // 设置当前线程的 ci 指向刚初始化的 CallInfo 结构
  L1->ci = ci;
}

static void freestack (lua_State *L) {
  if (L->stack == NULL)
    return;  /* stack not completely built yet */
  // 释放整个 'ci' 列表
  L->ci = &L->base_ci;
  luaE_freeCI(L);
  // 断言当前 'ci' 列表为空
  lua_assert(L->nci == 0);
  // 释放堆栈数组的内存空间
  luaM_freearray(L, L->stack, stacksize(L) + EXTRA_STACK);  /* free stack */
}

/*
** Create registry table and its predefined values
*/
static void init_registry (lua_State *L, global_State *g) {
  /* create registry */
  // 创建注册表
  Table *registry = luaH_new(L);
  // 设置全局状态中的 l_registry 指向注册表
  sethvalue(L, &g->l_registry, registry);
  // 调整注册表的大小
  luaH_resize(L, registry, LUA_RIDX_LAST, 0);
  // 设置 registry[LUA_RIDX_MAINTHREAD] = L
  setthvalue(L, &registry->array[LUA_RIDX_MAINTHREAD - 1], L);
  // 设置 registry[LUA_RIDX_GLOBALS] = new table (table of globals)
  sethvalue(L, &registry->array[LUA_RIDX_GLOBALS - 1], luaH_new(L));
}

/*
** open parts of the state that may cause memory-allocation errors.
*/
static void f_luaopen (lua_State *L, void *ud) {
  global_State *g = G(L);
  UNUSED(ud);
  stack_init(L, L);  /* init stack */
  init_registry(L, g);
  luaS_init(L);
  luaT_init(L);
  luaX_init(L);
  g->gcstp = 0;  /* allow gc */
  setnilvalue(&g->nilvalue);  /* now state is complete */
  luai_userstateopen(L);
}
/*
** 为了避免错误，初始化线程之前需要分配任意内存
*/
static void preinit_thread (lua_State *L, global_State *g) {
  G(L) = g;  /* 设置当前线程的全局状态 */
  L->stack = NULL;  /* 初始化线程的栈为空 */
  L->ci = NULL;  /* 初始化当前调用信息为空 */
  L->nci = 0;  /* 初始化当前调用信息数量为0 */
  L->twups = L;  /* 线程没有上值 */
  L->nCcalls = 0;  /* 初始化 C 调用数量为0 */
  L->errorJmp = NULL;  /* 初始化错误跳转为空 */
  L->hook = NULL;  /* 初始化钩子为空 */
  L->hookmask = 0;  /* 初始化钩子掩码为0 */
  L->basehookcount = 0;  /* 初始化基础钩子计数为0 */
  L->allowhook = 1;  /* 允许钩子 */
  resethookcount(L);  /* 重置钩子计数 */
  L->openupval = NULL;  /* 初始化打开的上值为空 */
  L->status = LUA_OK;  /* 初始化状态为 OK */
  L->errfunc = 0;  /* 初始化错误处理函数为0 */
  L->oldpc = 0;  /* 初始化旧的程序计数器为0 */
}

static void close_state (lua_State *L) {
  global_State *g = G(L);
  if (!completestate(g))  /* 是否关闭了部分构建的状态？ */
    luaC_freeallobjects(L);  /* 仅收集其对象 */
  else {  /* 关闭完全构建的状态 */
    L->ci = &L->base_ci;  /* 展开调用信息列表 */
    luaD_closeprotected(L, 1, LUA_OK);  /* 关闭所有上值 */
    luaC_freeallobjects(L);  /* 收集所有对象 */
    luai_userstateclose(L);  /* 关闭用户状态 */
  }
  luaM_freearray(L, G(L)->strt.hash, G(L)->strt.size);  /* 释放字符串表 */
  freestack(L);  /* 释放栈 */
  lua_assert(gettotalbytes(g) == sizeof(LG));  /* 断言总字节数等于 LG 结构体的大小 */
  (*g->frealloc)(g->ud, fromstate(L), sizeof(LG), 0);  /* 释放主块 */
}

LUA_API lua_State *lua_newthread (lua_State *L) {
  global_State *g;
  lua_State *L1;
  lua_lock(L);
  g = G(L);
  luaC_checkGC(L);  /* 检查垃圾回收 */
  /* 创建新线程 */
  L1 = &cast(LX *, luaM_newobject(L, LUA_TTHREAD, sizeof(LX)))->l;
  L1->marked = luaC_white(g);  /* 标记为白色 */
  L1->tt = LUA_VTHREAD;  /* 设置类型为线程 */
  /* 将其链接到 'allgc' 列表上 */
  L1->next = g->allgc;
  g->allgc = obj2gco(L1);
  /* 将其锚定在 L 栈上 */
  setthvalue2s(L, L->top, L1);
  api_incr_top(L);
  preinit_thread(L1, g);  /* 初始化新线程 */
  L1->hookmask = L->hookmask;  /* 设置钩子掩码 */
  L1->basehookcount = L->basehookcount;  /* 设置基础钩子计数 */
  L1->hook = L->hook;  /* 设置钩子 */
  resethookcount(L1);  /* 重置钩子计数 */
  /* 初始化 L1 的额外空间 */
  memcpy(lua_getextraspace(L1), lua_getextraspace(g->mainthread), LUA_EXTRASPACE);
  luai_userstatethread(L, L1);  /* 初始化用户状态线程 */
  stack_init(L1, L);  /* 初始化栈 */
  lua_unlock(L);
  return L1;  /* 返回新线程 */
}
void luaE_freethread (lua_State *L, lua_State *L1) {
  LX *l = fromstate(L1);  /* 获取线程 L1 对应的 LX 结构体指针 */
  luaF_closeupval(L1, L1->stack);  /* 关闭所有的上值 */
  lua_assert(L1->openupval == NULL);  /* 断言确保没有打开的上值 */
  luai_userstatefree(L, L1);  /* 释放用户状态 */
  freestack(L1);  /* 释放线程 L1 的栈空间 */
  luaM_free(L, l);  /* 释放 LX 结构体指针 */
}


int luaE_resetthread (lua_State *L, int status) {
  CallInfo *ci = L->ci = &L->base_ci;  /* 重置 CallInfo 列表 */
  setnilvalue(s2v(L->stack));  /* 为基本的 'ci' 设置 'function' 入口 */
  ci->func = L->stack;  /* 设置 'ci' 的函数指针为栈顶 */
  ci->callstatus = CIST_C;  /* 设置 'ci' 的调用状态为 CIST_C */
  if (status == LUA_YIELD)
    status = LUA_OK;  /* 如果状态为 LUA_YIELD，则将状态设置为 LUA_OK */
  L->status = LUA_OK;  /* 设置线程状态为 LUA_OK，以便运行 __close 元方法 */
  status = luaD_closeprotected(L, 1, status);  /* 关闭保护模式下的执行 */
  if (status != LUA_OK)  /* 如果有错误 */
    luaD_seterrorobj(L, status, L->stack + 1);  /* 设置错误对象 */
  else
    L->top = L->stack + 1;  /* 否则，设置栈顶指针为栈底 + 1 */
  ci->top = L->top + LUA_MINSTACK;  /* 设置 'ci' 的栈顶指针为栈顶 + 最小栈空间 */
  luaD_reallocstack(L, cast_int(ci->top - L->stack), 0);  /* 重新分配栈空间 */
  return status;  /* 返回状态 */
}


LUA_API int lua_resetthread (lua_State *L) {
  int status;
  lua_lock(L);  /* 加锁 */
  status = luaE_resetthread(L, L->status);  /* 重置线程状态 */
  lua_unlock(L);  /* 解锁 */
  return status;  /* 返回状态 */
}
# 创建一个新的 Lua 状态机
LUA_API lua_State *lua_newstate (lua_Alloc f, void *ud) {
  int i;  # 声明一个整型变量 i
  lua_State *L;  # 声明一个 Lua 状态机指针变量 L
  global_State *g;  # 声明一个全局状态机指针变量 g
  LG *l = cast(LG *, (*f)(ud, NULL, LUA_TTHREAD, sizeof(LG)));  # 调用分配器函数 f 来分配内存
  if (l == NULL) return NULL;  # 如果分配失败，则返回空指针
  L = &l->l.l;  # 获取 l 中的 Lua 状态机指针
  g = &l->g;  # 获取 l 中的全局状态机指针
  L->tt = LUA_VTHREAD;  # 设置 Lua 状态机类型为线程
  g->currentwhite = bitmask(WHITE0BIT);  # 设置当前白色位
  L->marked = luaC_white(g);  # 标记为白色
  preinit_thread(L, g);  # 初始化线程
  g->allgc = obj2gco(L);  # 将 L 转换为 GCObject，并赋值给全局状态机的 allgc
  L->next = NULL;  # 下一个线程为空
  incnny(L);  # 增加线程的计数
  g->frealloc = f;  # 设置全局状态机的内存分配器
  g->ud = ud;  # 设置全局状态机的用户数据
  g->warnf = NULL;  # 设置警告函数为空
  g->ud_warn = NULL;  # 设置警告函数的用户数据为空
  g->mainthread = L;  # 设置全局状态机的主线程为 L
  g->seed = luai_makeseed(L);  # 生成种子
  g->gcstp = GCSTPGC;  # 设置 GC 状态为 GCSTPGC
  g->strt.size = g->strt.nuse = 0;  # 设置字符串表的大小和使用量为 0
  g->strt.hash = NULL;  # 字符串表的哈希表为空
  setnilvalue(&g->l_registry);  # 设置全局状态机的注册表为 nil
  g->panic = NULL;  # 设置全局状态机的 panic 函数为空
  g->gcstate = GCSpause;  # 设置 GC 状态为暂停
  g->gckind = KGC_INC;  # 设置 GC 类型为增量 GC
  g->gcstopem = 0;  # 设置 GC 停止标志为 0
  g->gcemergency = 0;  # 设置 GC 紧急标志为 0
  g->finobj = g->tobefnz = g->fixedgc = NULL;  # 设置各种 GC 相关的指针为空
  g->firstold1 = g->survival = g->old1 = g->reallyold = NULL;  # 设置各种老对象指针为空
  g->finobjsur = g->finobjold1 = g->finobjrold = NULL;  # 设置各种 GC 相关的指针为空
  g->sweepgc = NULL;  # 设置 sweepgc 为空
  g->gray = g->grayagain = NULL;  # 设置灰色链表为空
  g->weak = g->ephemeron = g->allweak = NULL;  # 设置弱表链表为空
  g->twups = NULL;  # 设置线程链表为空
  g->totalbytes = sizeof(LG);  # 设置总字节数为 LG 的大小
  g->GCdebt = 0;  # 设置 GC 债务为 0
  g->lastatomic = 0;  # 设置最后原子操作为 0
  setivalue(&g->nilvalue, 0);  # 设置 nil 值为 0
  setgcparam(g->gcpause, LUAI_GCPAUSE);  # 设置 GC 暂停参数
  setgcparam(g->gcstepmul, LUAI_GCMUL);  # 设置 GC 步长参数
  g->gcstepsize = LUAI_GCSTEPSIZE;  # 设置 GC 步长大小
  setgcparam(g->genmajormul, LUAI_GENMAJORMUL);  # 设置主要 GC 倍数参数
  g->genminormul = LUAI_GENMINORMUL;  # 设置次要 GC 倍数参数
  for (i=0; i < LUA_NUMTAGS; i++) g->mt[i] = NULL;  # 初始化元表数组
  if (luaD_rawrunprotected(L, f_luaopen, NULL) != LUA_OK) {  # 调用 luaD_rawrunprotected 函数
    /* 内存分配错误：释放部分状态 */
    close_state(L);  # 关闭状态机
    L = NULL;  # 将状态机指针置为空
  }
  return L;  # 返回状态机指针
}


# 关闭 Lua 状态机
LUA_API void lua_close (lua_State *L) {
  lua_lock(L);  # 锁定状态机
  L = G(L)->mainthread;  # 获取全局状态机的主线程
  close_state(L);  # 关闭状态机
}


# Lua 警告函数
void luaE_warning (lua_State *L, const char *msg, int tocont) {
  lua_WarnFunction wf = G(L)->warnf;  # 获取全局状态机的警告函数
  if (wf != NULL)  # 如果警告函数不为空
    # 调用 wf 函数，传入参数 G(L)->ud_warn, msg, tocont
    wf(G(L)->ud_warn, msg, tocont);
/*
** 从错误消息生成一个警告
*/
void luaE_warnerror (lua_State *L, const char *where) {
  // 获取错误对象
  TValue *errobj = s2v(L->top - 1);  /* error object */
  // 如果错误对象是字符串，则使用错误消息，否则使用默认消息
  const char *msg = (ttisstring(errobj))
                  ? svalue(errobj)
                  : "error object is not a string";
  // 生成警告 "error in %s (%s)" (where, msg)
  luaE_warning(L, "error in ", 1);
  luaE_warning(L, where, 1);
  luaE_warning(L, " (", 1);
  luaE_warning(L, msg, 1);
  luaE_warning(L, ")", 0);
}
```