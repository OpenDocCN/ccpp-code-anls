# `nmap\liblua\ldo.c`

```cpp
/*
** $Id: ldo.c $
** Stack and Call structure of Lua
** See Copyright Notice in lua.h
*/

// 定义 ldo.c，表示该文件是 Lua 核心的一部分
#define ldo_c
#define LUA_CORE

// 包含必要的头文件
#include "lprefix.h"
#include <setjmp.h>
#include <stdlib.h>
#include <string.h>
#include "lua.h"
#include "lapi.h"
#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lgc.h"
#include "lmem.h"
#include "lobject.h"
#include "lopcodes.h"
#include "lparser.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "ltm.h"
#include "lundump.h"
#include "lvm.h"
#include "lzio.h"

// 定义 errorstatus 函数，用于判断状态是否为错误状态
#define errorstatus(s)    ((s) > LUA_YIELD)

/*
** {======================================================
** Error-recovery functions
** =======================================================
*/

/*
** LUAI_THROW/LUAI_TRY define how Lua does exception handling. By
** default, Lua handles errors with exceptions when compiling as
** C++ code, with _longjmp/_setjmp when asked to use them, and with
** longjmp/setjmp otherwise.
*/
#if !defined(LUAI_THROW)                /* { */

#if defined(__cplusplus) && !defined(LUA_USE_LONGJMP)    /* { */

/* C++ exceptions */
#define LUAI_THROW(L,c)        throw(c)
#define LUAI_TRY(L,c,a) \
    try { a } catch(...) { if ((c)->status == 0) (c)->status = -1; }
#define luai_jmpbuf        int  /* dummy variable */

#elif defined(LUA_USE_POSIX)                /* }{ */

/* in POSIX, try _longjmp/_setjmp (more efficient) */
#define LUAI_THROW(L,c)        _longjmp((c)->b, 1)
#define LUAI_TRY(L,c,a)        if (_setjmp((c)->b) == 0) { a }
#define luai_jmpbuf        jmp_buf

#else                            /* }{ */

/* ISO C handling with long jumps */
#define LUAI_THROW(L,c)        longjmp((c)->b, 1)
#define LUAI_TRY(L,c,a)        if (setjmp((c)->b) == 0) { a }
#define luai_jmpbuf        jmp_buf

#endif                            /* } */

#endif                            /* } */

/* chain list of long jump buffers */
/* 定义了一个结构体 lua_longjmp，用于保存 longjmp 的信息 */
struct lua_longjmp {
  struct lua_longjmp *previous;  /* 指向上一个 lua_longjmp 结构体的指针 */
  luai_jmpbuf b;  /* 保存 longjmp 的缓冲区 */
  volatile int status;  /* 错误代码，使用 volatile 修饰表示可能会被异步修改 */
};


/* 设置错误对象 */
void luaD_seterrorobj (lua_State *L, int errcode, StkId oldtop) {
  switch (errcode) {
    case LUA_ERRMEM: {  /* 内存错误？ */
      setsvalue2s(L, oldtop, G(L)->memerrmsg); /* 重用预注册的消息 */
      break;
    }
    case LUA_ERRERR: {
      setsvalue2s(L, oldtop, luaS_newliteral(L, "error in error handling"));  /* 设置错误消息为 "error in error handling" */
      break;
    }
    case LUA_OK: {  /* 仅用于关闭 upvalues 的特殊情况 */
      setnilvalue(s2v(oldtop));  /* 没有错误消息 */
      break;
    }
    default: {
      lua_assert(errorstatus(errcode));  /* 真正的错误 */
      setobjs2s(L, oldtop, L->top - 1);  /* 当前栈顶的错误消息 */
      break;
    }
  }
  L->top = oldtop + 1;  /* 设置栈顶指针 */
}


/* 抛出异常 */
l_noret luaD_throw (lua_State *L, int errcode) {
  if (L->errorJmp) {  /* 线程有错误处理程序？ */
    L->errorJmp->status = errcode;  /* 设置状态 */
    LUAI_THROW(L, L->errorJmp);  /* 跳转到错误处理程序 */
  }
  else {  /* 线程没有错误处理程序 */
    global_State *g = G(L);
    errcode = luaE_resetthread(L, errcode);  /* 关闭所有 upvalues */
    if (g->mainthread->errorJmp) {  /* 主线程有处理程序？ */
      setobjs2s(L, g->mainthread->top++, L->top - 1);  /* 复制错误对象 */
      luaD_throw(g->mainthread, errcode);  /* 在主线程中重新抛出 */
    }
    else {  /* 没有任何处理程序；中止 */
      if (g->panic) {  /* panic 函数？ */
        lua_unlock(L);
        g->panic(L);  /* 调用 panic 函数（最后一次跳出的机会） */
      }
      abort();  /* 中止程序 */
    }
  }
}


/* 运行受保护的代码 */
int luaD_rawrunprotected (lua_State *L, Pfunc f, void *ud) {
  l_uint32 oldnCcalls = L->nCcalls;  /* 保存旧的 C 调用次数 */
  struct lua_longjmp lj;  /* 定义 lua_longjmp 结构体 */
  lj.status = LUA_OK;  /* 设置状态为 OK */
  lj.previous = L->errorJmp;  /* 链接新的错误处理程序 */
  L->errorJmp = &lj;  /* 设置当前错误处理程序为 lj */
  LUAI_TRY(L, &lj,  /* 开始 try 块 */
    (*f)(L, ud);  /* 调用函数指针 f */
  );
  L->errorJmp = lj.previous;  /* 恢复旧的错误处理程序 */
  L->nCcalls = oldnCcalls;  /* 恢复旧的 C 调用次数 */
  return lj.status;  /* 返回状态 */
}
/* }====================================================== */

/*
** {==================================================================
** Stack reallocation
** ===================================================================
*/
// 重新分配栈空间，修正所有指向栈的指针
static void correctstack (lua_State *L, StkId oldstack, StkId newstack) {
  CallInfo *ci;
  UpVal *up;
  // 修正栈顶指针
  L->top = (L->top - oldstack) + newstack;
  // 修正线程的待关闭的 upvalue 列表指针
  L->tbclist = (L->tbclist - oldstack) + newstack;
  // 遍历待关闭的 upvalue 列表，修正指向栈的指针
  for (up = L->openupval; up != NULL; up = up->u.open.next)
    up->v = s2v((uplevel(up) - oldstack) + newstack);
  // 遍历调用信息列表，修正指向栈的指针
  for (ci = L->ci; ci != NULL; ci = ci->previous) {
    ci->top = (ci->top - oldstack) + newstack;
    ci->func = (ci->func - oldstack) + newstack;
    if (isLua(ci))
      ci->u.l.trap = 1;  /* signal to update 'trap' in 'luaV_execute' */
  }
}

// 为错误处理预留一些空间
#define ERRORSTACKSIZE    (LUAI_MAXSTACK + 200)

/*
** Reallocate the stack to a new size, correcting all pointers into
** it. (There are pointers to a stack from its upvalues, from its list
** of call infos, plus a few individual pointers.) The reallocation is
** done in two steps (allocation + free) because the correction must be
** done while both addresses (the old stack and the new one) are valid.
** (In ISO C, any pointer use after the pointer has been deallocated is
** undefined behavior.)
** In case of allocation error, raise an error or return false according
** to 'raiseerror'.
*/
// 重新分配栈空间到新的大小，修正所有指向栈的指针
int luaD_reallocstack (lua_State *L, int newsize, int raiseerror) {
  int oldsize = stacksize(L);
  int i;
  // 为新栈空间分配内存
  StkId newstack = luaM_reallocvector(L, NULL, 0,
                                      newsize + EXTRA_STACK, StackValue);
  // 断言新栈大小不超过最大栈大小，或者等于错误栈大小
  lua_assert(newsize <= LUAI_MAXSTACK || newsize == ERRORSTACKSIZE);
  // 如果分配失败
  if (l_unlikely(newstack == NULL)) {  /* reallocation failed? */
    // 根据 'raiseerror' 抛出错误或返回 false
    if (raiseerror)
      luaM_error(L);
    else return 0;  /* 如果条件不满足，则返回0，不触发错误 */
  }
  /* 计算需要复制到新栈的元素数量 */
  i = ((oldsize <= newsize) ? oldsize : newsize) + EXTRA_STACK;
  // 将旧栈中的元素复制到新栈中
  memcpy(newstack, L->stack, i * sizeof(StackValue));
  // 将新栈中多余的部分置为nil，擦除新段
  for (; i < newsize + EXTRA_STACK; i++)
    setnilvalue(s2v(newstack + i)); /* erase new segment */
  // 调整栈的指针和大小
  correctstack(L, L->stack, newstack);
  // 释放旧栈的内存空间
  luaM_freearray(L, L->stack, oldsize + EXTRA_STACK);
  // 更新栈指针和栈的最后一个元素指针
  L->stack = newstack;
  L->stack_last = L->stack + newsize;
  // 返回1表示操作成功
  return 1;
/*
** 尝试至少扩展 'n' 个元素的堆栈。当 'raiseerror' 为真时，引发任何错误；否则，在出现错误时返回 0。
*/
int luaD_growstack (lua_State *L, int n, int raiseerror) {
  int size = stacksize(L);  // 获取当前堆栈大小
  if (l_unlikely(size > LUAI_MAXSTACK)) {  // 如果堆栈大于最大值
    /* 如果堆栈大于最大值，线程已经使用额外空间来处理错误，也就是线程正在处理堆栈错误；不能再进一步扩展。 */
    lua_assert(stacksize(L) == ERRORSTACKSIZE);  // 断言当前堆栈大小等于错误堆栈大小
    if (raiseerror)
      luaD_throw(L, LUA_ERRERR);  /* 在消息处理程序内部发生错误 */
    return 0;  /* 如果不 'raiseerror'，只是发出信号 */
  }
  else {
    int newsize = 2 * size;  /* 试探性的新大小 */
    int needed = cast_int(L->top - L->stack) + n;  // 需要的大小
    if (newsize > LUAI_MAXSTACK)  /* 不能超过限制 */
      newsize = LUAI_MAXSTACK;
    if (newsize < needed)  /* 但必须遵守要求 */
      newsize = needed;
    if (l_likely(newsize <= LUAI_MAXSTACK))
      return luaD_reallocstack(L, newsize, raiseerror);
    else {  /* 堆栈溢出 */
      /* 添加额外大小以处理错误消息 */
      luaD_reallocstack(L, ERRORSTACKSIZE, raiseerror);
      if (raiseerror)
        luaG_runerror(L, "stack overflow");
      return 0;
    }
  }
}


static int stackinuse (lua_State *L) {
  CallInfo *ci;
  int res;
  StkId lim = L->top;
  for (ci = L->ci; ci != NULL; ci = ci->previous) {
    if (lim < ci->top) lim = ci->top;
  }
  lua_assert(lim <= L->stack_last);
  res = cast_int(lim - L->stack) + 1;  /* 使用中的堆栈部分 */
  if (res < LUA_MINSTACK)
    res = LUA_MINSTACK;  /* 确保最小大小 */
  return res;
}


/*
** 如果堆栈大小超过当前使用量的 3 倍，则将该大小减少到当前使用量的两倍。（因此，最终堆栈大小最多是先前大小的 2/3，其一半条目为空。）
** 作为特殊情况，如果堆栈正在处理堆栈溢出，现在
*/
/*
** 如果不是，'max'（受到 LUAI_MAXSTACK 限制）将会比 stacksize（在这种情况下等于 ERRORSTACKSIZE）小，因此栈将被缩减到一个“常规”大小。
*/
void luaD_shrinkstack (lua_State *L) {
  // 计算当前栈中使用的空间大小
  int inuse = stackinuse(L);
  // 计算建议的新栈大小
  int nsize = inuse * 2;
  // 计算最大“合理”大小
  int max = inuse * 3;
  if (max > LUAI_MAXSTACK) {
    max = LUAI_MAXSTACK;  // 限制栈的大小
    if (nsize > LUAI_MAXSTACK)
      nsize = LUAI_MAXSTACK;
  }
  /* 如果线程当前没有处理栈溢出，并且其大小大于最大“合理”大小，则缩小它 */
  if (inuse <= LUAI_MAXSTACK && stacksize(L) > max)
    luaD_reallocstack(L, nsize, 0);  // 如果失败也没关系
  else  // 不改变栈
    condmovestack(L,{},{});  // （仅用于调试时改变）
  luaE_shrinkCI(L);  // 缩小 CI 列表
}


void luaD_inctop (lua_State *L) {
  luaD_checkstack(L, 1);
  L->top++;
}

/* }================================================================== */


/*
** 调用给定事件的钩子。确保有要调用的钩子。（'L->hook' 和 'L->hookmask' 都可以被信号异步更改，触发这个函数。）
*/
void luaD_hook (lua_State *L, int event, int line,
                              int ftransfer, int ntransfer) {
  lua_Hook hook = L->hook;
  if (hook && L->allowhook) {  // 确保有钩子
    int mask = CIST_HOOKED;
    CallInfo *ci = L->ci;
    ptrdiff_t top = savestack(L, L->top);  // 保存原始的 'top'
    ptrdiff_t ci_top = savestack(L, ci->top);  // 同样保存 'ci->top'
    lua_Debug ar;
    ar.event = event;
    ar.currentline = line;
    ar.i_ci = ci;
    if (ntransfer != 0) {
      mask |= CIST_TRAN;  // 'ci' 有传输信息
      ci->u2.transferinfo.ftransfer = ftransfer;
      ci->u2.transferinfo.ntransfer = ntransfer;
    }
    # 如果当前调用的函数是 Lua 函数，并且栈顶位置小于当前调用信息中的栈顶位置
    if (isLua(ci) && L->top < ci->top)
      # 将栈顶位置设置为当前调用信息中的栈顶位置，保护整个激活寄存器
      L->top = ci->top;  /* protect entire activation register */
    # 确保栈的最小大小
    luaD_checkstack(L, LUA_MINSTACK);  /* ensure minimum stack size */
    # 如果当前调用信息中的栈顶位置小于 Lua 状态机中的栈顶位置加上最小栈大小
    if (ci->top < L->top + LUA_MINSTACK)
      # 将当前调用信息中的栈顶位置设置为 Lua 状态机中的栈顶位置加上最小栈大小
      ci->top = L->top + LUA_MINSTACK;
    # 禁止在钩子函数内部调用钩子函数
    L->allowhook = 0;  /* cannot call hooks inside a hook */
    # 将当前调用信息中的调用状态按位或上给定的掩码
    ci->callstatus |= mask;
    # 解锁 Lua 状态机
    lua_unlock(L);
    # 调用给定的钩子函数
    (*hook)(L, &ar);
    # 锁定 Lua 状态机
    lua_lock(L);
    # 断言当前状态下不允许调用钩子函数
    lua_assert(!L->allowhook);
    # 允许调用钩子函数
    L->allowhook = 1;
    # 将当前调用信息中的栈顶位置设置为恢复后的栈顶位置
    ci->top = restorestack(L, ci_top);
    # 将 Lua 状态机中的栈顶位置设置为恢复后的栈顶位置
    L->top = restorestack(L, top);
    # 将当前调用信息中的调用状态按位与上给定的掩码的补码
    ci->callstatus &= ~mask;
  }
/*
** 执行 Lua 函数的调用钩子。每当 'hookmask' 不为零时，都会调用此函数，因此它会检查调用钩子是否激活。
*/
void luaD_hookcall (lua_State *L, CallInfo *ci) {
  L->oldpc = 0;  /* 为新函数设置 'oldpc' */
  if (L->hookmask & LUA_MASKCALL) {  /* 调用钩子是否开启？ */
    int event = (ci->callstatus & CIST_TAIL) ? LUA_HOOKTAILCALL
                                             : LUA_HOOKCALL;
    Proto *p = ci_func(ci)->p;
    ci->u.l.savedpc++;  /* 钩子假设 'pc' 已经增加 */
    luaD_hook(L, event, -1, 1, p->numparams);
    ci->u.l.savedpc--;  /* 修正 'pc' */
  }
}


/*
** 执行 Lua 和 C 函数的返回钩子，并设置/修正 'oldpc'。（请注意，即使返回钩子关闭，由于行钩子需要，因此仍然需要进行此修正。）
*/
static void rethook (lua_State *L, CallInfo *ci, int nres) {
  if (L->hookmask & LUA_MASKRET) {  /* 返回钩子是否开启？ */
    StkId firstres = L->top - nres;  /* 第一个结果的索引 */
    int delta = 0;  /* 可变参数函数的修正 */
    int ftransfer;
    if (isLua(ci)) {
      Proto *p = ci_func(ci)->p;
      if (p->is_vararg)
        delta = ci->u.l.nextraargs + p->numparams + 1;
    }
    ci->func += delta;  /* 如果是可变参数，回到虚拟的 'func' */
    ftransfer = cast(unsigned short, firstres - ci->func);
    luaD_hook(L, LUA_HOOKRET, -1, ftransfer, nres);  /* 调用它 */
    ci->func -= delta;
  }
  if (isLua(ci = ci->previous))
    L->oldpc = pcRel(ci->u.l.savedpc, ci_func(ci)->p);  /* 设置 'oldpc' */
}


/*
** 检查 'func' 是否有 '__call' 元方法。如果有，将其放在原始 'func' 下面的堆栈中，以便 'luaD_precall' 可以调用它。如果没有 '__call' 元方法，则引发错误。
*/
StkId luaD_tryfuncTM (lua_State *L, StkId func) {
  const TValue *tm;  // 定义一个指向 TValue 类型的常量指针 tm
  StkId p;  // 定义一个类型为 StkId 的变量 p
  checkstackGCp(L, 1, func);  // 检查栈空间是否足够，为元方法预留空间
  tm = luaT_gettmbyobj(L, s2v(func), TM_CALL);  // 获取对象的元方法
  if (l_unlikely(ttisnil(tm)))  // 如果元方法为空
    luaG_callerror(L, s2v(func));  // 报错，没有可调用的函数
  for (p = L->top; p > func; p--)  // 为元方法预留空间
    setobjs2s(L, p, p-1);  // 将 p-1 处的值复制到 p 处
  L->top++;  // 栈顶指针上移，为调用者预分配的栈空间
  setobj2s(L, func, tm);  // 将元方法设置为新的要调用的函数
  return func;  // 返回函数
}

/*
** Given 'nres' results at 'firstResult', move 'wanted' of them to 'res'.
** Handle most typical cases (zero results for commands, one result for
** expressions, multiple results for tail calls/single parameters)
** separated.
*/
l_sinline void moveresults (lua_State *L, StkId res, int nres, int wanted) {
  StkId firstresult;  // 定义一个类型为 StkId 的变量 firstresult
  int i;  // 定义一个整型变量 i
  switch (wanted) {  // 根据 wanted 的值进行不同的处理
    case 0:  // 不需要值
      L->top = res;  // 栈顶指针指向 res
      return;  // 返回
    case 1:  // 需要一个值
      if (nres == 0)   // 没有结果？
        setnilvalue(s2v(res));  // 将 res 处的值设为 nil
      else  // 至少有一个结果
        setobjs2s(L, res, L->top - nres);  // 将结果移动到正确的位置
      L->top = res + 1;  // 栈顶指针上移一个位置
      return;  // 返回
    case LUA_MULTRET:
      wanted = nres;  // 我们需要所有的结果
      break;
    default:  /* 处理默认情况 */
      if (hastocloseCfunc(wanted)) {  /* 是否有需要关闭的变量？ */
        ptrdiff_t savedres = savestack(L, res);  /* 保存当前栈顶位置 */
        L->ci->callstatus |= CIST_CLSRET;  /* 设置调用状态为需要关闭返回值 */
        L->ci->u2.nres = nres;  /* 设置调用信息中返回值的数量 */
        luaF_close(L, res, CLOSEKTOP, 1);  /* 关闭需要关闭的变量 */
        L->ci->callstatus &= ~CIST_CLSRET;  /* 清除关闭返回值的状态 */
        if (L->hookmask)  /* 如果需要，调用钩子函数处理 '__close' 之后的操作 */
          rethook(L, L->ci, nres);
        res = restorestack(L, savedres);  /* 恢复栈顶位置，因为关闭和钩子函数可能会移动栈 */
        wanted = decodeNresults(wanted);  /* 解码期望的返回值数量 */
        if (wanted == LUA_MULTRET)  /* 如果期望所有返回值 */
          wanted = nres;  /* 设置期望返回值数量为实际返回值数量 */
      }
      break;  /* 结束 switch 语句块 */
  }
  /* 通用情况 */
  firstresult = L->top - nres;  /* 第一个返回值的索引 */
  if (nres > wanted)  /* 多余的返回值？ */
    nres = wanted;  /* 不需要多余的返回值 */
  for (i = 0; i < nres; i++)  /* 将所有返回值移动到正确的位置 */
    setobjs2s(L, res + i, firstresult + i);
  for (; i < wanted; i++)  /* 补全期望的返回值数量 */
    setnilvalue(s2v(res + i));
  L->top = res + wanted;  /* 栈顶指针指向最后一个返回值之后的位置 */
** 结束函数调用：如果必要，调用钩子函数，将当前结果移动到适当位置，并返回到先前的调用信息。如果函数必须关闭变量，则必须在此之后调用钩子函数。
*/
void luaD_poscall (lua_State *L, CallInfo *ci, int nres) {
  int wanted = ci->nresults;
  // 如果钩子函数存在并且不需要关闭 C 函数，则调用 rethook 函数
  if (l_unlikely(L->hookmask && !hastocloseCfunc(wanted)))
    rethook(L, ci, nres);
  /* 将结果移动到适当位置 */
  moveresults(L, ci->func, nres, wanted);
  /* 返回时函数不能处于以下任何情况 */
  lua_assert(!(ci->callstatus &
        (CIST_HOOKED | CIST_YPCALL | CIST_FIN | CIST_TRAN | CIST_CLSRET)));
  L->ci = ci->previous;  /* 返回到调用者（在关闭变量之后） */
}



#define next_ci(L)  (L->ci->next ? L->ci->next : luaE_extendCI(L))


l_sinline CallInfo *prepCallInfo (lua_State *L, StkId func, int nret,
                                                int mask, StkId top) {
  CallInfo *ci = L->ci = next_ci(L);  /* 新的帧 */
  ci->func = func;
  ci->nresults = nret;
  ci->callstatus = mask;
  ci->top = top;
  return ci;
}


/*
** C 函数的预调用
*/
l_sinline int precallC (lua_State *L, StkId func, int nresults,
                                            lua_CFunction f) {
  int n;  /* 返回值的数量 */
  CallInfo *ci;
  checkstackGCp(L, LUA_MINSTACK, func);  /* 确保堆栈的最小大小 */
  L->ci = ci = prepCallInfo(L, func, nresults, CIST_C,
                               L->top + LUA_MINSTACK);
  lua_assert(ci->top <= L->stack_last);
  if (l_unlikely(L->hookmask & LUA_MASKCALL)) {
    int narg = cast_int(L->top - func) - 1;
    luaD_hook(L, LUA_HOOKCALL, -1, 1, narg);
  }
  lua_unlock(L);
  n = (*f)(L);  /* 进行实际调用 */
  lua_lock(L);
  api_checknelems(L, n);
  luaD_poscall(L, ci, n);
  return n;
}


/*
** 为尾调用准备函数，构建其调用信息并置于当前调用信息之上。'narg1' 是参数的数量加 1
/*
** 准备调用函数（C 函数或 Lua 函数）。对于 C 函数，还要执行调用。
** 要调用的函数在 '*func' 中。参数在函数后的堆栈上。如果是 Lua 函数，
** 返回要执行的 CallInfo。否则（C 函数），返回 NULL，结果都在堆栈上，
** 从原始函数位置开始。
*/
int luaD_pretailcall (lua_State *L, CallInfo *ci, StkId func,
                                    int narg1, int delta) {
 retry:  // 重试标签
  switch (ttypetag(s2v(func))) {  // 根据函数类型标签进行判断
    case LUA_VCCL:  /* C 闭包 */
      return precallC(L, func, LUA_MULTRET, clCvalue(s2v(func))->f);  // 调用 C 函数
    case LUA_VLCF:  /* 轻量级 C 函数 */
      return precallC(L, func, LUA_MULTRET, fvalue(s2v(func)));  // 调用轻量级 C 函数
    case LUA_VLCL: {  /* Lua 函数 */
      Proto *p = clLvalue(s2v(func))->p;  // 获取 Lua 函数的原型
      int fsize = p->maxstacksize;  /* 帧大小 */
      int nfixparams = p->numparams;  // 固定参数个数
      int i;
      checkstackGCp(L, fsize - delta, func);  // 检查堆栈大小
      ci->func -= delta;  /* 恢复 'func'（如果是可变参数）*/
      for (i = 0; i < narg1; i++)  /* 移动函数和参数 */
        setobjs2s(L, ci->func + i, func + i);
      func = ci->func;  /* 移动后的函数 */
      for (; narg1 <= nfixparams; narg1++)
        setnilvalue(s2v(func + narg1));  /* 补全缺失的参数 */
      ci->top = func + 1 + fsize;  /* 新函数的栈顶 */
      lua_assert(ci->top <= L->stack_last);  // 断言栈顶不超过最大值
      ci->u.l.savedpc = p->code;  /* 起始点 */
      ci->callstatus |= CIST_TAIL;  // 设置尾调用状态
      L->top = func + narg1;  /* 设置栈顶 */
      return -1;  // 返回 -1
    }
    default: {  /* 不是函数 */
      func = luaD_tryfuncTM(L, func);  /* 尝试获取 '__call' 元方法 */
      /* return luaD_pretailcall(L, ci, func, narg1 + 1, delta); */
      narg1++;  // 参数加一
      goto retry;  // 重试
    }
  }
}
# 调用 Lua 函数之前的准备工作，包括判断函数类型和准备调用信息
CallInfo *luaD_precall (lua_State *L, StkId func, int nresults) {
 retry:  # 重试标签，用于跳转到指定位置
  switch (ttypetag(s2v(func))) {  # 根据函数类型标签进行判断
    case LUA_VCCL:  /* C closure */  # 如果是 C 闭包
      precallC(L, func, nresults, clCvalue(s2v(func))->f);  # 调用 C 闭包函数
      return NULL;  # 返回空指针
    case LUA_VLCF:  /* light C function */  # 如果是轻量级 C 函数
      precallC(L, func, nresults, fvalue(s2v(func)));  # 调用轻量级 C 函数
      return NULL;  # 返回空指针
    case LUA_VLCL: {  /* Lua function */  # 如果是 Lua 函数
      CallInfo *ci;  # 定义调用信息结构体指针
      Proto *p = clLvalue(s2v(func))->p;  # 获取 Lua 函数对应的原型
      int narg = cast_int(L->top - func) - 1;  # 计算实际参数个数
      int nfixparams = p->numparams;  # 获取固定参数个数
      int fsize = p->maxstacksize;  # 获取帧大小
      checkstackGCp(L, fsize, func);  # 检查栈空间是否足够
      L->ci = ci = prepCallInfo(L, func, nresults, 0, func + 1 + fsize);  # 准备调用信息
      ci->u.l.savedpc = p->code;  # 设置起始指令
      for (; narg < nfixparams; narg++)  # 遍历补全缺失的参数
        setnilvalue(s2v(L->top++));  # 设置为 nil 值
      lua_assert(ci->top <= L->stack_last);  # 断言栈顶位置不超过栈的最后位置
      return ci;  # 返回调用信息
    }
    default: {  /* not a function */  # 如果不是函数
      func = luaD_tryfuncTM(L, func);  # 尝试获取 '__call' 元方法
      /* return luaD_precall(L, func, nresults); */  # 返回调用预处理结果
      goto retry;  # 再次尝试使用元方法
    }
  }
}

'''
** 通过 C 调用函数（C 或 Lua）。'inc' 可以是 1（增加 C 栈中递归调用的次数）或 nyci（相同加上增加不可中断调用的次数）。
'''
l_sinline void ccall (lua_State *L, StkId func, int nResults, int inc) {
  CallInfo *ci;  # 定义调用信息结构体指针
  L->nCcalls += inc;  # 增加 C 调用次数
  if (l_unlikely(getCcalls(L) >= LUAI_MAXCCALLS))  # 如果 C 调用次数超过最大限制
    luaE_checkcstack(L);  # 检查 C 栈空间
  if ((ci = luaD_precall(L, func, nResults)) != NULL) {  /* Lua function? */  # 如果是 Lua 函数
    ci->callstatus = CIST_FRESH;  # 标记为“新鲜”执行
    luaV_execute(L, ci);  # 执行 Lua 函数
  }
  L->nCcalls -= inc;  # 减少 C 调用次数
}

'''
** 'ccall' 的外部接口
'''
void luaD_call (lua_State *L, StkId func, int nResults) {
  ccall(L, func, nResults, 1);  # 调用 ccall 函数
}

'''
** 类似于 'luaD_call'，但在调用期间不允许产生 yield
'''
void luaD_callnoyield (lua_State *L, StkId func, int nResults) {
  // 调用 C 函数，不允许产生 yield
  ccall(L, func, nResults, nyci);
}


/*
** 当 lua_pcallk 被 yield 中断后，完成其工作的函数
** （调用者 finishCcall 调用 adjustresults 完成最终调用）
** 主要工作是完成由 lua_pcallk 调用的 luaD_pcall
** 如果 '__close' 方法在这里产生 yield，最终控制权将回到 finishCcall
** （当 '__close' 方法最终返回时），finishpcallk 将再次运行并关闭任何仍未完成的 '__close' 方法
** 类似地，如果 '__close' 方法出错，'precover' 调用 'unroll'，它调用 'finishCcall'，我们再次回到这里，关闭任何未完成的 '__close' 方法
** 注意，在调用 'luaF_close' 之前，相应的 'CallInfo' 没有被修改，因此这个重复运行的过程就像第一次运行一样（除了至少少做一个 '__close'）
** 特别地，字段 CIST_RECST 保留了这些多次运行中的错误状态，只有在有新错误时才会改变
*/
static int finishpcallk (lua_State *L,  CallInfo *ci) {
  int status = getcistrecst(ci);  /* 获取原始状态 */
  if (l_likely(status == LUA_OK))  /* 没有错误？ */
    status = LUA_YIELD;  /* 被 yield 中断 */
  else {  /* 出错 */
    StkId func = restorestack(L, ci->u2.funcidx);
    L->allowhook = getoah(ci->callstatus);  /* 恢复 'allowhook' */
    luaF_close(L, func, status, 1);  /* 可能产生 yield 或引发错误 */
    func = restorestack(L, ci->u2.funcidx);  /* 栈可能已移动 */
    luaD_seterrorobj(L, status, func);
    luaD_shrinkstack(L);   /* 恢复栈大小以防溢出 */
    setcistrecst(ci, LUA_OK);  /* 清除原始状态 */
  }
  ci->callstatus &= ~CIST_YPCALL;
  L->errfunc = ci->u.c.old_errfunc;
  /* 如果到了这里，说明有错误或 yield；与 'lua_pcallk' 不同，不改变状态 */
  return status;
}


/*
** 完成被 yield 中断的 C 函数的执行
*/
/*
** finishCcall函数用于完成C函数调用的中断处理
** 如果中断发生在'moveresults'关闭其tbc变量或执行'lua_callk'/'lua_pcallk'时
** 在第一种情况下，它只是重新执行'luaD_poscall'
** 在第二种情况下，对'finishpcallk'的调用完成了'lua_pcallk'的中断执行
** 然后调用中断函数的继续执行，最后完成了调用该函数的'luaD_call'的工作
** 在调用'adjustresults'时，我们不知道'lua_callk'/'lua_pcallk'调用的函数结果的数量，所以我们保守地使用LUA_MULTRET（始终调整）
*/
static void finishCcall (lua_State *L, CallInfo *ci) {
  int n;  /* C函数实际返回的结果数量 */
  if (ci->callstatus & CIST_CLSRET) {  /* 是否正在返回？ */
    lua_assert(hastocloseCfunc(ci->nresults));
    n = ci->u2.nres;  /* 重新执行'luaD_poscall' */
    /* 不需要重置CIST_CLSRET，因为它无论如何都会被设置 */
  }
  else {
    int status = LUA_YIELD;  /* 如果没有错误，默认为LUA_YIELD */
    /* 必须有一个继续执行，并且必须能够调用它 */
    lua_assert(ci->u.c.k != NULL && yieldable(L));
    if (ci->callstatus & CIST_YPCALL)   /* 是否在'lua_pcallk'内部？ */
      status = finishpcallk(L, ci);  /* 完成'lua_pcallk' */
    adjustresults(L, LUA_MULTRET);  /* 完成'lua_callk' */
    lua_unlock(L);
    n = (*ci->u.c.k)(L, status, ci->u.c.ctx);  /* 调用继续执行 */
    lua_lock(L);
    api_checknelems(L, n);
  }
  luaD_poscall(L, ci, n);  /* 完成'luaD_call' */
}


/*
** 执行先前中断的协程的“完整继续执行”（堆栈中的所有内容），直到堆栈为空（或另一个中断长跳出循环）
*/
static void unroll (lua_State *L, void *ud) {
  CallInfo *ci;
  UNUSED(ud);
  while ((ci = L->ci) != &L->base_ci) {  /* 堆栈中有内容 */
    # 如果当前调用栈帧不是Lua函数，则表示是C函数
    if (!isLua(ci))  /* C function? */
      # 完成C函数的执行
      finishCcall(L, ci);  /* complete its execution */
    else {  /* Lua function */
      # 完成被中断的指令
      luaV_finishOp(L);  /* finish interrupted instruction */
      # 执行Lua函数直到达到更高级的C函数边界
      luaV_execute(L, ci);  /* execute down to higher C 'boundary' */
    }
  }
/*
** Try to find a suspended protected call (a "recover point") for the
** given thread.
*/
static CallInfo *findpcall (lua_State *L) {
  CallInfo *ci;
  for (ci = L->ci; ci != NULL; ci = ci->previous) {  /* search for a pcall */
    if (ci->callstatus & CIST_YPCALL)
      return ci;
  }
  return NULL;  /* no pending pcall */
}


/*
** Signal an error in the call to 'lua_resume', not in the execution
** of the coroutine itself. (Such errors should not be handled by any
** coroutine error handler and should not kill the coroutine.)
*/
static int resume_error (lua_State *L, const char *msg, int narg) {
  L->top -= narg;  /* remove args from the stack */
  setsvalue2s(L, L->top, luaS_new(L, msg));  /* push error message */
  api_incr_top(L);
  lua_unlock(L);
  return LUA_ERRRUN;
}


/*
** Do the work for 'lua_resume' in protected mode. Most of the work
** depends on the status of the coroutine: initial state, suspended
** inside a hook, or regularly suspended (optionally with a continuation
** function), plus erroneous cases: non-suspended coroutine or dead
** coroutine.
*/
static void resume (lua_State *L, void *ud) {
  int n = *(cast(int*, ud));  /* number of arguments */
  StkId firstArg = L->top - n;  /* first argument */
  CallInfo *ci = L->ci;
  if (L->status == LUA_OK)  /* starting a coroutine? */
    ccall(L, firstArg - 1, LUA_MULTRET, 0);  /* just call its body */
  else {  /* resuming from previous yield */
    lua_assert(L->status == LUA_YIELD);
    L->status = LUA_OK;  /* mark that it is running (again) */
    if (isLua(ci)) {  /* yielded inside a hook? */
      L->top = firstArg;  /* discard arguments */
      luaV_execute(L, ci);  /* just continue running Lua code */
    }
    else {  /* 'common' yield */  # 如果不是特殊情况下的 yield，则执行以下代码
      if (ci->u.c.k != NULL) {  /* does it have a continuation function? */  # 检查是否有一个继续函数
        lua_unlock(L);  # 解锁 Lua 状态机
        n = (*ci->u.c.k)(L, LUA_YIELD, ci->u.c.ctx); /* call continuation */  # 调用继续函数
        lua_lock(L);  # 锁定 Lua 状态机
        api_checknelems(L, n);  # 检查栈中元素的数量
      }
      luaD_poscall(L, ci, n);  /* finish 'luaD_call' */  # 完成 'luaD_call'
    }
    unroll(L, NULL);  /* run continuation */  # 运行继续函数
  }
/*
** 在受保护模式下展开一个协程，当存在可恢复的错误时，即受保护调用内部的错误。
** （任何错误都会中断'展开'，这个循环再次保护它，以便它可以继续。）
** 以正常结束（status == LUA_OK）、一个 yield（status == LUA_YIELD）或一个不受保护的错误（'findpcall' 找不到恢复点）停止。
*/
static int precover (lua_State *L, int status) {
  CallInfo *ci;
  while (errorstatus(status) && (ci = findpcall(L)) != NULL) {
    L->ci = ci;  /* 下降到恢复函数 */
    setcistrecst(ci, status);  /* 设置完成'pcall'的状态 */
    status = luaD_rawrunprotected(L, unroll, NULL);
  }
  return status;
}

LUA_API int lua_resume (lua_State *L, lua_State *from, int nargs,
                                      int *nresults) {
  int status;
  lua_lock(L);
  if (L->status == LUA_OK) {  /* 可能是启动一个协程 */
    if (L->ci != &L->base_ci)  /* 不在基本级别？ */
      return resume_error(L, "cannot resume non-suspended coroutine", nargs);
    else if (L->top - (L->ci->func + 1) == nargs)  /* 没有函数？ */
      return resume_error(L, "cannot resume dead coroutine", nargs);
  }
  else if (L->status != LUA_YIELD)  /* 以错误结束？ */
    return resume_error(L, "cannot resume dead coroutine", nargs);
  L->nCcalls = (from) ? getCcalls(from) : 0;
  if (getCcalls(L) >= LUAI_MAXCCALLS)
    return resume_error(L, "C stack overflow", nargs);
  L->nCcalls++;
  luai_userstateresume(L, nargs);
  api_checknelems(L, (L->status == LUA_OK) ? nargs + 1 : nargs);
  status = luaD_rawrunprotected(L, resume, &nargs);
   /* 在可恢复错误后继续运行 */
  status = precover(L, status);
  if (l_likely(!errorstatus(status)))
    lua_assert(status == L->status);  /* 正常结束或 yield */
  else {  /* 无法恢复的错误 */
    L->status = cast_byte(status);  /* 将线程标记为'死' */
    luaD_seterrorobj(L, status, L->top);  /* 推送错误消息 */
}
    # 将当前线程的栈顶指针赋值给当前执行的线程的栈顶指针
    L->ci->top = L->top;
  }
  # 如果状态是 LUA_YIELD，则将结果数量设置为当前执行的线程的 u2.nyield 值，否则设置为当前执行的线程的栈顶指针减去当前执行的线程的函数指针加一的结果
  *nresults = (status == LUA_YIELD) ? L->ci->u2.nyield
                                    : cast_int(L->top - (L->ci->func + 1));
  # 解锁当前执行的线程
  lua_unlock(L);
  # 返回状态
  return status;
}

// 检查当前线程是否可以被挂起
LUA_API int lua_isyieldable (lua_State *L) {
  return yieldable(L);
}

// 执行一个带有回调函数的 yield 操作
LUA_API int lua_yieldk (lua_State *L, int nresults, lua_KContext ctx,
                        lua_KFunction k) {
  CallInfo *ci;
  luai_userstateyield(L, nresults);  // 记录当前状态
  lua_lock(L);  // 锁定 Lua 状态
  ci = L->ci;  // 获取当前调用信息
  api_checknelems(L, nresults);  // 检查栈中是否有足够的结果
  if (l_unlikely(!yieldable(L))) {  // 如果当前线程不可被挂起
    if (L != G(L)->mainthread)
      luaG_runerror(L, "attempt to yield across a C-call boundary");  // 报错：尝试在 C 调用边界上挂起
    else
      luaG_runerror(L, "attempt to yield from outside a coroutine");  // 报错：尝试在协程外部挂起
  }
  L->status = LUA_YIELD;  // 设置当前状态为挂起
  ci->u2.nyield = nresults;  // 保存结果数量
  if (isLua(ci)) {  // 如果在钩子函数中
    lua_assert(!isLuacode(ci));
    api_check(L, nresults == 0, "hooks cannot yield values");  // 检查：钩子函数不能返回值
    api_check(L, k == NULL, "hooks cannot continue after yielding");  // 检查：钩子函数不能在挂起后继续执行
  }
  else {
    if ((ci->u.c.k = k) != NULL)  // 如果有回调函数
      ci->u.c.ctx = ctx;  // 保存上下文
    luaD_throw(L, LUA_YIELD);  // 执行挂起操作
  }
  lua_assert(ci->callstatus & CIST_HOOKED);  // 必须在钩子函数中
  lua_unlock(L);  // 解锁 Lua 状态
  return 0;  // 返回到 'luaD_hook'
}

// 用于在受保护模式下调用 'luaF_close' 的辅助结构
struct CloseP {
  StkId level;
  int status;
};

// 在受保护模式下调用 'luaF_close' 的辅助函数
static void closepaux (lua_State *L, void *ud) {
  struct CloseP *pcl = cast(struct CloseP *, ud);
  luaF_close(L, pcl->level, pcl->status, 0);
}

// 在受保护模式下调用 'luaF_close'，返回原始状态或新状态（如果出现错误）
int luaD_closeprotected (lua_State *L, ptrdiff_t level, int status) {
  CallInfo *old_ci = L->ci;
  lu_byte old_allowhooks = L->allowhook;
  for (;;) {  // 循环直到没有更多错误
    struct CloseP pcl;
    pcl.level = restorestack(L, level); pcl.status = status;
    status = luaD_rawrunprotected(L, &closepaux, &pcl);
    if (l_likely(status == LUA_OK))  // 没有更多错误？
      return pcl.status;
    else {  /* 如果发生错误，则恢复先前保存的状态并重复 */
      L->ci = old_ci;  // 将当前执行的线程状态恢复为先前保存的线程状态
      L->allowhook = old_allowhooks;  // 将允许钩子的状态恢复为先前保存的状态
    }
  }
/*
** 调用 C 函数 'func' 以受保护模式运行，恢复基本线程信息（'allowhook' 等），
** 特别是在发生错误时恢复其堆栈级别。
*/
int luaD_pcall (lua_State *L, Pfunc func, void *u,
                ptrdiff_t old_top, ptrdiff_t ef) {
  int status;  // 用于存储函数调用的状态
  CallInfo *old_ci = L->ci;  // 保存当前调用信息
  lu_byte old_allowhooks = L->allowhook;  // 保存当前的 allowhook 值
  ptrdiff_t old_errfunc = L->errfunc;  // 保存当前的 errfunc 值
  L->errfunc = ef;  // 设置新的 errfunc 值
  status = luaD_rawrunprotected(L, func, u);  // 在受保护模式下运行函数 'func'
  if (l_unlikely(status != LUA_OK)) {  /* 发生错误？ */
    L->ci = old_ci;  // 恢复旧的调用信息
    L->allowhook = old_allowhooks;  // 恢复旧的 allowhook 值
    status = luaD_closeprotected(L, old_top, status);  // 关闭受保护模式下的执行
    luaD_seterrorobj(L, status, restorestack(L, old_top));  // 设置错误对象
    luaD_shrinkstack(L);   /* 恢复堆栈大小以防溢出 */
  }
  L->errfunc = old_errfunc;  // 恢复旧的 errfunc 值
  return status;  // 返回函数调用的状态
}



/*
** 执行受保护的解析器。
*/
struct SParser {  /* 传递给 'f_parser' 的数据 */
  ZIO *z;  // 输入流
  Mbuffer buff;  /* 扫描器使用的动态结构 */
  Dyndata dyd;  /* 解析器使用的动态结构 */
  const char *mode;  // 模式
  const char *name;  // 名称
};


static void checkmode (lua_State *L, const char *mode, const char *x) {
  if (mode && strchr(mode, x[0]) == NULL) {  // 检查模式是否匹配
    luaO_pushfstring(L,
       "attempt to load a %s chunk (mode is '%s')", x, mode);  // 推入格式化的错误消息
    luaD_throw(L, LUA_ERRSYNTAX);  // 抛出语法错误
  }
}


static void f_parser (lua_State *L, void *ud) {
  LClosure *cl;  // 闭包
  struct SParser *p = cast(struct SParser *, ud);  // 将数据转换为 SParser 结构
  int c = zgetc(p->z);  /* 读取第一个字符 */
  if (c == LUA_SIGNATURE[0]) {  // 如果第一个字符是 Lua 签名
    checkmode(L, p->mode, "binary");  // 检查模式
    cl = luaU_undump(L, p->z, p->name);  // 从输入流中加载二进制代码
  }
  else {
    checkmode(L, p->mode, "text");  // 检查模式
    cl = luaY_parser(L, p->z, &p->buff, &p->dyd, p->name, c);  // 解析文本代码
  }
  lua_assert(cl->nupvalues == cl->p->sizeupvalues);  // 断言检查闭包的 upvalues 数量
  luaF_initupvals(L, cl);  // 初始化闭包的 upvalues
}
int luaD_protectedparser (lua_State *L, ZIO *z, const char *name,
                                        const char *mode) {
  // 定义一个结构体变量 p，用于存储解析器的相关信息
  struct SParser p;
  // 定义一个整型变量 status，用于存储解析的状态
  int status;
  // 禁止在解析期间进行中断
  incnny(L);  /* cannot yield during parsing */
  // 设置结构体 p 的成员变量
  p.z = z; p.name = name; p.mode = mode;
  // 初始化动态数组，用于存储活跃变量
  p.dyd.actvar.arr = NULL; p.dyd.actvar.size = 0;
  // 初始化动态数组，用于存储全局变量
  p.dyd.gt.arr = NULL; p.dyd.gt.size = 0;
  // 初始化动态数组，用于存储标签
  p.dyd.label.arr = NULL; p.dyd.label.size = 0;
  // 初始化缓冲区
  luaZ_initbuffer(L, &p.buff);
  // 调用 luaD_pcall 函数，执行解析器的解析过程
  status = luaD_pcall(L, f_parser, &p, savestack(L, L->top), L->errfunc);
  // 释放缓冲区
  luaZ_freebuffer(L, &p.buff);
  // 释放动态数组内存
  luaM_freearray(L, p.dyd.actvar.arr, p.dyd.actvar.size);
  luaM_freearray(L, p.dyd.gt.arr, p.dyd.gt.size);
  luaM_freearray(L, p.dyd.label.arr, p.dyd.label.size);
  // 恢复中断状态
  decnny(L);
  // 返回解析的状态
  return status;
}
```