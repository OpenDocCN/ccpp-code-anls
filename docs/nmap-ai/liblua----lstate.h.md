# `nmap\liblua\lstate.h`

```cpp
/*
** $Id: lstate.h $
** Global State
** See Copyright Notice in lua.h
*/

#ifndef lstate_h
#define lstate_h

#include "lua.h"

#include "lobject.h"
#include "ltm.h"
#include "lzio.h"


/*
** Some notes about garbage-collected objects: All objects in Lua must
** be kept somehow accessible until being freed, so all objects always
** belong to one (and only one) of these lists, using field 'next' of
** the 'CommonHeader' for the link:
**
** 'allgc': all objects not marked for finalization;
** 'finobj': all objects marked for finalization;
** 'tobefnz': all objects ready to be finalized;
** 'fixedgc': all objects that are not to be collected (currently
** only small strings, such as reserved words).
**
** For the generational collector, some of these lists have marks for
** generations. Each mark points to the first element in the list for
** that particular generation; that generation goes until the next mark.
**
** 'allgc' -> 'survival': new objects;
** 'survival' -> 'old': objects that survived one collection;
** 'old1' -> 'reallyold': objects that became old in last collection;
** 'reallyold' -> NULL: objects old for more than one cycle.
**
** 'finobj' -> 'finobjsur': new objects marked for finalization;
** 'finobjsur' -> 'finobjold1': survived   """";
** 'finobjold1' -> 'finobjrold': just old  """";
** 'finobjrold' -> NULL: really old       """".
**
** All lists can contain elements older than their main ages, due
** to 'luaC_checkfinalizer' and 'udata2finalize', which move
** objects between the normal lists and the "marked for finalization"
** lists. Moreover, barriers can age young objects in young lists as
** OLD0, which then become OLD1. However, a list never contains
** elements younger than their main ages.
**
** The generational collector also uses a pointer 'firstold1', which
** points to the first OLD1 object in the list. It is used to optimize
** 'markold'. (Potentially OLD1 objects can be anywhere between 'allgc'
*/
/*
** and 'reallyold', but often the list has no OLD1 objects or they are
** after 'old1'.) Note the difference between it and 'old1':
** 'firstold1': no OLD1 objects before this point; there can be all
**   ages after it.
** 'old1': no objects younger than OLD1 after this point.
*/

/*
** Moreover, there is another set of lists that control gray objects.
** These lists are linked by fields 'gclist'. (All objects that
** can become gray have such a field. The field is not the same
** in all objects, but it always has this name.)  Any gray object
** must belong to one of these lists, and all objects in these lists
** must be gray (with two exceptions explained below):
**
** 'gray': regular gray objects, still waiting to be visited.
** 'grayagain': objects that must be revisited at the atomic phase.
**   That includes
**   - black objects got in a write barrier;
**   - all kinds of weak tables during propagation phase;
**   - all threads.
** 'weak': tables with weak values to be cleared;
** 'ephemeron': ephemeron tables with white->white entries;
** 'allweak': tables with weak keys and/or weak values to be cleared.
**
** The exceptions to that "gray rule" are:
** - TOUCHED2 objects in generational mode stay in a gray list (because
** they must be visited again at the end of the cycle), but they are
** marked black because assignments to them must activate barriers (to
** move them back to TOUCHED1).
** - Open upvales are kept gray to avoid barriers, but they stay out
** of gray lists. (They don't even have a 'gclist' field.)
*/

/*
** About 'nCcalls':  This count has two parts: the lower 16 bits counts
** the number of recursive invocations in the C stack; the higher
** 16 bits counts the number of non-yieldable calls in the stack.
** (They are together so that we can change and save both with one
** instruction.)
*/

/* true if this thread does not have non-yieldable calls in the stack */
#define yieldable(L)        (((L)->nCcalls & 0xffff0000) == 0)
/* 获取实际的 C 调用次数 */
#define getCcalls(L)    ((L)->nCcalls & 0xffff)

/* 增加不可中断调用的次数 */
#define incnny(L)    ((L)->nCcalls += 0x10000)

/* 减少不可中断调用的次数 */
#define decnny(L)    ((L)->nCcalls -= 0x10000)

/* 不可中断调用增量 */
#define nyci    (0x10000 | 1)

/* lua_longjmp 结构体的定义，定义在 ldo.c 中 */
struct lua_longjmp;

/*
** 原子类型（相对于信号）以更好地确保 'lua_sethook' 是线程安全的
*/
#if !defined(l_signalT)
#include <signal.h>
#define l_signalT    sig_atomic_t
#endif

/*
** 用于处理 TM 调用和一些其他额外的堆栈空间。这个空间不包括在 'stack_last' 中。
** 它仅用于避免堆栈检查，要么因为元素将很快被弹出，要么因为在推送后很快会有堆栈检查。
** 函数帧永远不使用这个额外的空间，所以它不需要保持清洁。
*/
#define EXTRA_STACK   5

#define BASIC_STACK_SIZE        (2*LUA_MINSTACK)

#define stacksize(th)    cast_int((th)->stack_last - (th)->stack)

/* 垃圾回收的类型 */
#define KGC_INC        0    /* 增量垃圾回收 */
#define KGC_GEN        1    /* 分代垃圾回收 */

/* 字符串表的结构体 */
typedef struct stringtable {
  TString **hash;
  int nuse;  /* 元素数量 */
  int size;
} stringtable;

/*
** 关于调用的信息。
** 关于联合 'u'：
** - 字段 'l' 仅用于 Lua 函数；
** - 字段 'c' 仅用于 C 函数。
** 关于联合 'u2'：
** - 字段 'funcidx' 仅在 C 函数进行受保护调用时使用；
** - 字段 'nyield' 仅在函数“执行” yield 时使用（从 yield 到下一次 resume）；
** - 字段 'nres' 仅在从函数返回时关闭 tbc 变量时使用；
** - 字段 'transferinfo' 仅在调用/返回钩子期间使用，在函数开始之前或结束之后。
*/
// 定义了 CallInfo 结构体，用于保存函数调用的信息
typedef struct CallInfo {
  StkId func;  /* 函数在栈中的索引 */
  StkId    top;  /* 该函数的栈顶 */
  struct CallInfo *previous, *next;  /* 动态调用链 */
  union {
    struct {  /* 仅用于 Lua 函数 */
      const Instruction *savedpc;
      volatile l_signalT trap;
      int nextraargs;  /* 可变参数函数中的额外参数个数 */
    } l;
    struct {  /* 仅用于 C 函数 */
      lua_KFunction k;  /* 在产生 yield 时的继续执行函数 */
      ptrdiff_t old_errfunc;
      lua_KContext ctx;  /* 在产生 yield 时的上下文信息 */
    } c;
  } u;
  union {
    int funcidx;  /* 被调用的函数索引 */
    int nyield;  /* 产生的值的数量 */
    int nres;  /* 返回的值的数量 */
    struct {  /* 传递值的信息（用于调用/返回钩子） */
      unsigned short ftransfer;  /* 第一个传递的值的偏移量 */
      unsigned short ntransfer;  /* 传递的值的数量 */
    } transferinfo;
  } u2;
  short nresults;  /* 该函数期望的返回结果数量 */
  unsigned short callstatus;
} CallInfo;

/*
** CallInfo 状态的位
*/
#define CIST_OAH    (1<<0)    /* 'allowhook' 的原始值 */
#define CIST_C        (1<<1)    /* 调用正在运行一个 C 函数 */
#define CIST_FRESH    (1<<2)    /* 调用在一个新的 "luaV_execute" 帧上 */
#define CIST_HOOKED    (1<<3)    /* 调用正在运行一个调试钩子 */
#define CIST_YPCALL    (1<<4)    /* 执行一个可产生 yield 的受保护调用 */
#define CIST_TAIL    (1<<5)    /* 调用是尾调用 */
#define CIST_HOOKYIELD    (1<<6)    /* 最后一个调试钩子调用产生了 yield */
#define CIST_FIN    (1<<7)    /* 函数 "调用" 了一个终结器 */
#define CIST_TRAN    (1<<8)    /* 'ci' 包含传递信息 */
#define CIST_CLSRET    (1<<9)  /* 函数正在关闭 tbc 变量 */
/* 位 10-12 用于 CIST_RECST（见下文） */
#define CIST_RECST    10
#if defined(LUA_COMPAT_LT_LE)
#define CIST_LEQ    (1<<13)  /* 使用 __lt 来实现 __le */
#endif
/* Field CIST_RECST stores the "recover status", used to keep the error
** status while closing to-be-closed variables in coroutines, so that
** Lua can correctly resume after an yield from a __close method called
** because of an error.  (Three bits are enough for error status.)
*/
#define getcistrecst(ci)     (((ci)->callstatus >> CIST_RECST) & 7)  // 获取coroutine的recover状态
#define setcistrecst(ci,st)  \  // 设置coroutine的recover状态
  check_exp(((st) & 7) == (st),   /* status must fit in three bits */  \  // 确保状态在三位内
            ((ci)->callstatus = ((ci)->callstatus & ~(7 << CIST_RECST))  \  // 清除原状态，设置新状态
                                                  | ((st) << CIST_RECST)))


/* active function is a Lua function */
#define isLua(ci)    (!((ci)->callstatus & CIST_C))  // 判断是否为Lua函数

/* call is running Lua code (not a hook) */
#define isLuacode(ci)    (!((ci)->callstatus & (CIST_C | CIST_HOOKED)))  // 判断是否正在运行Lua代码（不是hook）

/* assume that CIST_OAH has offset 0 and that 'v' is strictly 0/1 */
#define setoah(st,v)    ((st) = ((st) & ~CIST_OAH) | (v))  // 设置CIST_OAH的值为v
#define getoah(st)    ((st) & CIST_OAH)  // 获取CIST_OAH的值


/*
** 'global state', shared by all threads of this state
*/
} global_State;  // 全局状态


/*
** 'per thread' state
*/
# 定义 Lua 状态结构体
struct lua_State {
  CommonHeader;  # 公共头部
  lu_byte status;  # 状态
  lu_byte allowhook;  # 是否允许钩子
  unsigned short nci;  # 'ci' 列表中的项目数
  StkId top;  # 栈中的第一个空闲槽
  global_State *l_G;  # 全局状态
  CallInfo *ci;  # 当前函数的调用信息
  StkId stack_last;  # 栈的末尾（最后一个元素 + 1）
  StkId stack;  # 栈基址
  UpVal *openupval;  # 此栈中打开的上值列表
  StkId tbclist;  # 待关闭变量列表
  GCObject *gclist;  # 可收集对象列表
  struct lua_State *twups;  # 具有打开上值的线程列表
  struct lua_longjmp *errorJmp;  # 当前错误恢复点
  CallInfo base_ci;  # 第一级（C 调用 Lua）的 CallInfo
  volatile lua_Hook hook;  # 钩子函数
  ptrdiff_t errfunc;  # 当前错误处理函数（栈索引）
  l_uint32 nCcalls;  # 嵌套（不可中断 | C）调用的数量
  int oldpc;  # 最后跟踪的 pc
  int basehookcount;  # 基本钩子计数
  int hookcount;  # 钩子计数
  volatile l_signalT hookmask;  # 钩子掩码
};

# 定义宏，返回 Lua 全局状态
#define G(L)    (L->l_G)

# 定义宏，检查状态是否完全构建
#define completestate(g)    ttisnil(&g->nilvalue)

# 定义联合体，包含所有可收集对象（仅用于转换）
union GCUnion {
  GCObject gc;  # 公共头部
  struct TString ts;  # 字符串
  struct Udata u;  # 用户数据
  union Closure cl;  # 闭包
  struct Table h;  # 表
  struct Proto p;  # 原型
  struct lua_State th;  # 线程
  struct UpVal upv;  # 上值
};

# 定义宏，将对象转换为联合体指针
#define cast_u(o)    cast(union GCUnion *, (o))

# 宏，将 GCObject 转换为特定值的宏
/* 宏定义：将 Lua 对象转换为 GCObject，确保 'v' 实际上是一个 Lua 对象 */
#define obj2gco(v)    check_exp((v)->tt >= LUA_TSTRING, &(cast_u(v)->gc))

/* 宏定义：将 GCObject 转换为 TString */
#define gco2ts(o)  \
    check_exp(novariant((o)->tt) == LUA_TSTRING, &((cast_u(o))->ts))

/* 宏定义：将 GCObject 转换为 Udata */
#define gco2u(o)  check_exp((o)->tt == LUA_VUSERDATA, &((cast_u(o))->u))

/* 宏定义：将 GCObject 转换为 LClosure */
#define gco2lcl(o)  check_exp((o)->tt == LUA_VLCL, &((cast_u(o))->cl.l))

/* 宏定义：将 GCObject 转换为 CClosure */
#define gco2ccl(o)  check_exp((o)->tt == LUA_VCCL, &((cast_u(o))->cl.c))

/* 宏定义：将 GCObject 转换为 Closure */
#define gco2cl(o)  \
    check_exp(novariant((o)->tt) == LUA_TFUNCTION, &((cast_u(o))->cl))

/* 宏定义：将 GCObject 转换为 Table */
#define gco2t(o)  check_exp((o)->tt == LUA_VTABLE, &((cast_u(o))->h))

/* 宏定义：将 GCObject 转换为 Proto */
#define gco2p(o)  check_exp((o)->tt == LUA_VPROTO, &((cast_u(o))->p))

/* 宏定义：将 GCObject 转换为 Thread */
#define gco2th(o)  check_exp((o)->tt == LUA_VTHREAD, &((cast_u(o))->th))

/* 宏定义：将 GCObject 转换为 UpVal */
#define gco2upv(o)    check_exp((o)->tt == LUA_VUPVAL, &((cast_u(o))->upv))

/* 获取实际分配的总字节数 */
#define gettotalbytes(g)    cast(lu_mem, (g)->totalbytes + (g)->GCdebt)

/* 函数声明 */
LUAI_FUNC void luaE_setdebt (global_State *g, l_mem debt);
LUAI_FUNC void luaE_freethread (lua_State *L, lua_State *L1);
LUAI_FUNC CallInfo *luaE_extendCI (lua_State *L);
LUAI_FUNC void luaE_freeCI (lua_State *L);
LUAI_FUNC void luaE_shrinkCI (lua_State *L);
LUAI_FUNC void luaE_checkcstack (lua_State *L);
LUAI_FUNC void luaE_incCstack (lua_State *L);
LUAI_FUNC void luaE_warning (lua_State *L, const char *msg, int tocont);
LUAI_FUNC void luaE_warnerror (lua_State *L, const char *where);
LUAI_FUNC int luaE_resetthread (lua_State *L, int status);
```