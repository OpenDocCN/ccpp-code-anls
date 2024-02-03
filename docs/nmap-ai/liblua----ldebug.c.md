# `nmap\liblua\ldebug.c`

```cpp
/*
** $Id: ldebug.c $
** Debug Interface
** See Copyright Notice in lua.h
*/

#define ldebug_c  // 定义 ldebug_c 宏
#define LUA_CORE  // 定义 LUA_CORE 宏

#include "lprefix.h"  // 包含 lprefix.h 文件


#include <stdarg.h>  // 包含标准参数头文件
#include <stddef.h>  // 包含标准定义头文件
#include <string.h>  // 包含字符串操作头文件

#include "lua.h"  // 包含 Lua 头文件

#include "lapi.h"  // 包含 lapi.h 文件
#include "lcode.h"  // 包含 lcode.h 文件
#include "ldebug.h"  // 包含 ldebug.h 文件
#include "ldo.h"  // 包含 ldo.h 文件
#include "lfunc.h"  // 包含 lfunc.h 文件
#include "lobject.h"  // 包含 lobject.h 文件
#include "lopcodes.h"  // 包含 lopcodes.h 文件
#include "lstate.h"  // 包含 lstate.h 文件
#include "lstring.h"  // 包含 lstring.h 文件
#include "ltable.h"  // 包含 ltable.h 文件
#include "ltm.h"  // 包含 ltm.h 文件
#include "lvm.h"  // 包含 lvm.h 文件


#define noLuaClosure(f)        ((f) == NULL || (f)->c.tt == LUA_VCCL)  // 定义宏 noLuaClosure


static const char *funcnamefromcall (lua_State *L, CallInfo *ci,
                                                   const char **name);  // 声明函数 funcnamefromcall


static int currentpc (CallInfo *ci) {  // 定义函数 currentpc
  lua_assert(isLua(ci));  // 断言函数 isLua 返回真
  return pcRel(ci->u.l.savedpc, ci_func(ci)->p);  // 返回当前指令的偏移量
}


/*
** Get a "base line" to find the line corresponding to an instruction.
** Base lines are regularly placed at MAXIWTHABS intervals, so usually
** an integer division gets the right place. When the source file has
** large sequences of empty/comment lines, it may need extra entries,
** so the original estimate needs a correction.
** If the original estimate is -1, the initial 'if' ensures that the
** 'while' will run at least once.
** The assertion that the estimate is a lower bound for the correct base
** is valid as long as the debug info has been generated with the same
** value for MAXIWTHABS or smaller. (Previous releases use a little
** smaller value.)
*/
static int getbaseline (const Proto *f, int pc, int *basepc) {  // 定义函数 getbaseline
  if (f->sizeabslineinfo == 0 || pc < f->abslineinfo[0].pc) {  // 如果绝对行信息的大小为0或者当前指令偏移量小于第一个绝对行信息的偏移量
    *basepc = -1;  // 将 basepc 设置为-1，从头开始
    return f->linedefined;  // 返回函数定义的行号
  }
  else {
    int i = cast_uint(pc) / MAXIWTHABS - 1;  // 获取一个估计值
    /* estimate must be a lower bound of the correct base */
    lua_assert(i < 0 ||
              (i < f->sizeabslineinfo && f->abslineinfo[i].pc <= pc));  // 断言估计值是正确基数的下界
    while (i + 1 < f->sizeabslineinfo && pc >= f->abslineinfo[i + 1].pc)  // 当估计值加1小于绝对行信息的大小并且当前指令偏移量大于等于下一个绝对行信息的偏移量时
      i++;  // 调整估计值
    # 将指针 basepc 指向 f->abslineinfo[i].pc 所指向的内存地址
    *basepc = f->abslineinfo[i].pc;
    # 返回 f->abslineinfo[i].line 的值
    return f->abslineinfo[i].line;
  }
/*
** 获取函数 'f' 中指令 'pc' 对应的行号；
** 首先获取基准行号，然后进行增量直到达到所需的指令。
*/
int luaG_getfuncline (const Proto *f, int pc) {
  // 没有调试信息？
  if (f->lineinfo == NULL)  
    return -1;
  else {
    int basepc;
    // 获取基准行号
    int baseline = getbaseline(f, pc, &basepc);
    // 循环直到达到给定指令
    while (basepc++ < pc) {  
      // 断言基准行号对应的调试信息不是绝对行号
      lua_assert(f->lineinfo[basepc] != ABSLINEINFO);
      // 纠正行号
      baseline += f->lineinfo[basepc];  
    }
    return baseline;
  }
}


// 获取当前行号
static int getcurrentline (CallInfo *ci) {
  return luaG_getfuncline(ci_func(ci)->p, currentpc(ci));
}


/*
** 为所有活动的 Lua 帧设置 'trap'。
** 可以在信号期间调用此函数，在“合理”的假设下。
** 新的 'ci' 在成为“活动”列表的一部分之前完全链接到列表中，并且我们假设指针是原子的；参见下一个函数中的注释。
** （进行过程序间优化的编译器理论上可以重新排列内存写入，以使列表在插入新元素时暂时中断。我们只是假设它没有充分的理由这样做。）
*/
static void settraps (CallInfo *ci) {
  for (; ci != NULL; ci = ci->previous)
    if (isLua(ci))
      ci->u.l.trap = 1;
}


/*
** 可以在信号期间调用此函数，在“合理”的假设下。
** 'basehookcount' 和 'hookcount' 字段（由 'resethookcount' 设置）仅用于调试，如果它们获得任意值（最多导致一个错误的钩子调用）也没有问题。
** 'hookmask' 是原子值。我们假设指针也是原子的（例如，gcc 确保在其运行的所有平台上都是这样）。此外，在调用之前始终检查 'hook'（参见 'luaD_hook'）。
*/
LUA_API void lua_sethook (lua_State *L, lua_Hook func, int mask, int count) {
  // 关闭钩子？
  if (func == NULL || mask == 0) {  
    mask = 0;
    # 将 func 变量设置为 NULL
    func = NULL;
  }
  # 设置虚拟机的钩子函数
  L->hook = func;
  # 设置虚拟机的基础钩子计数
  L->basehookcount = count;
  # 重置钩子计数
  resethookcount(L);
  # 设置虚拟机的钩子掩码
  L->hookmask = cast_byte(mask);
  # 如果钩子掩码不为 0，则设置陷阱以跟踪 'luaV_execute' 函数内部
  if (mask)
    settraps(L->ci);  /* to trace inside 'luaV_execute' */
// 返回当前状态机的钩子函数
LUA_API lua_Hook lua_gethook (lua_State *L) {
  return L->hook;
}

// 返回当前状态机的钩子掩码
LUA_API int lua_gethookmask (lua_State *L) {
  return L->hookmask;
}

// 返回当前状态机的基础钩子计数
LUA_API int lua_gethookcount (lua_State *L) {
  return L->basehookcount;
}

// 获取调用栈信息
LUA_API int lua_getstack (lua_State *L, int level, lua_Debug *ar) {
  int status;
  CallInfo *ci;
  if (level < 0) return 0;  /* invalid (negative) level */
  lua_lock(L);
  for (ci = L->ci; level > 0 && ci != &L->base_ci; ci = ci->previous)
    level--;
  if (level == 0 && ci != &L->base_ci) {  /* level found? */
    status = 1;
    ar->i_ci = ci;
  }
  else status = 0;  /* no such level */
  lua_unlock(L);
  return status;
}

// 返回指定函数的上值名称
static const char *upvalname (const Proto *p, int uv) {
  TString *s = check_exp(uv < p->sizeupvalues, p->upvalues[uv].name);
  if (s == NULL) return "?";
  else return getstr(s);
}

// 查找可变参数
static const char *findvararg (CallInfo *ci, int n, StkId *pos) {
  if (clLvalue(s2v(ci->func))->p->is_vararg) {
    int nextra = ci->u.l.nextraargs;
    if (n >= -nextra) {  /* 'n' is negative */
      *pos = ci->func - nextra - (n + 1);
      return "(vararg)";  /* generic name for any vararg */
    }
  }
  return NULL;  /* no such vararg */
}

// 查找本地变量
const char *luaG_findlocal (lua_State *L, CallInfo *ci, int n, StkId *pos) {
  StkId base = ci->func + 1;
  const char *name = NULL;
  if (isLua(ci)) {
    if (n < 0)  /* access to vararg values? */
      return findvararg(ci, n, pos);
    else
      name = luaF_getlocalname(ci_func(ci)->p, n, currentpc(ci));
  }
  if (name == NULL) {  /* no 'standard' name? */
    StkId limit = (ci == L->ci) ? L->top : ci->next->func;
    if (limit - base >= n && n > 0) {  /* is 'n' inside 'ci' stack? */
      /* generic name for any valid slot */
      name = isLua(ci) ? "(temporary)" : "(C temporary)";
    }
    else
      return NULL;  /* no name */
  }
  if (pos)
    *pos = base + (n - 1);
  return name;
}
# 获取当前函数的局部变量名
LUA_API const char *lua_getlocal (lua_State *L, const lua_Debug *ar, int n) {
  const char *name;  # 声明一个指向字符常量的指针变量
  lua_lock(L);  # 加锁，确保线程安全
  if (ar == NULL) {  # 如果传入的调试信息为空
    if (!isLfunction(s2v(L->top - 1)))  # 如果栈顶不是 Lua 函数
      name = NULL;  # 将 name 设置为 NULL
    else  # 如果是 Lua 函数
      name = luaF_getlocalname(clLvalue(s2v(L->top - 1))->p, n, 0);  # 获取函数参数的局部变量名
  }
  else {  # 如果传入的调试信息不为空
    StkId pos = NULL;  # 声明一个栈指针变量，避免警告
    name = luaG_findlocal(L, ar->i_ci, n, &pos);  # 通过调试信息获取局部变量名
    if (name) {  # 如果获取到了局部变量名
      setobjs2s(L, L->top, pos);  # 将局部变量的值设置到栈顶
      api_incr_top(L);  # 增加栈顶指针
    }
  }
  lua_unlock(L);  # 解锁
  return name;  # 返回局部变量名
}


# 设置当前函数的局部变量
LUA_API const char *lua_setlocal (lua_State *L, const lua_Debug *ar, int n) {
  StkId pos = NULL;  # 声明一个栈指针变量，避免警告
  const char *name;  # 声明一个指向字符常量的指针变量
  lua_lock(L);  # 加锁，确保线程安全
  name = luaG_findlocal(L, ar->i_ci, n, &pos);  # 通过调试信息获取局部变量名
  if (name) {  # 如果获取到了局部变量名
    setobjs2s(L, pos, L->top - 1);  # 将栈顶的值设置到局部变量
    L->top--;  # 减少栈顶指针，弹出值
  }
  lua_unlock(L);  # 解锁
  return name;  # 返回局部变量名
}


# 获取函数信息
static void funcinfo (lua_Debug *ar, Closure *cl) {
  if (noLuaClosure(cl)) {  # 如果不是 Lua 函数
    ar->source = "=[C]";  # 设置源文件名为 =[C]
    ar->srclen = LL("=[C]");  # 设置源文件名长度
    ar->linedefined = -1;  # 设置起始行号为 -1
    ar->lastlinedefined = -1;  # 设置结束行号为 -1
    ar->what = "C";  # 设置函数类型为 C
  }
  else {  # 如果是 Lua 函数
    const Proto *p = cl->l.p;  # 获取函数的原型
    if (p->source) {  # 如果函数有源文件名
      ar->source = getstr(p->source);  # 获取源文件名
      ar->srclen = tsslen(p->source);  # 获取源文件名长度
    }
    else {  # 如果函数没有源文件名
      ar->source = "=?";  # 设置源文件名为 =?
      ar->srclen = LL("=?");  # 设置源文件名长度
    }
    ar->linedefined = p->linedefined;  # 设置起始行号
    ar->lastlinedefined = p->lastlinedefined;  # 设置结束行号
    ar->what = (ar->linedefined == 0) ? "main" : "Lua";  # 根据起始行号设置函数类型
  }
  luaO_chunkid(ar->short_src, ar->source, ar->srclen);  # 生成函数的短源文件名
}


# 获取下一行的行号
static int nextline (const Proto *p, int currentline, int pc) {
  if (p->lineinfo[pc] != ABSLINEINFO)  # 如果指令对应的行号不是绝对行号
    return currentline + p->lineinfo[pc];  # 返回当前行号加上指令对应的行号
  else
    return luaG_getfuncline(p, pc);  # 返回指令对应的行号
}


# 收集有效的行号
static void collectvalidlines (lua_State *L, Closure *f) {
  if (noLuaClosure(f)) {  # 如果不是 Lua 函数
    setnilvalue(s2v(L->top));  # 设置栈顶为 nil
    api_incr_top(L);  # 增加栈顶指针
  }
  else {  # 如果是 Lua 函数
    int i;  # 声明一个整型变量
    TValue v;  # 声明一个值变量
    // 从函数对象中获取其原型指针
    const Proto *p = f->l.p;
    // 获取当前行号
    int currentline = p->linedefined;
    // 创建一个新的表来存储活动行
    Table *t = luaH_new(L);  /* new table to store active lines */
    // 将新创建的表推入栈顶
    sethvalue2s(L, L->top, t);  /* push it on stack */
    // 栈顶指针向上移动
    api_incr_top(L);
    // 设置布尔值'true'为所有索引的值
    setbtvalue(&v);  /* boolean 'true' to be the value of all indices */
    // 如果不是可变参数函数
    if (!p->is_vararg)  /* regular function? */
      // 将i设置为0，考虑所有指令
      i = 0;  /* consider all instructions */
    // 如果是可变参数函数
    else {  /* vararg function */
      // 断言第一条指令是OP_VARARGPREP
      lua_assert(GET_OPCODE(p->code[0]) == OP_VARARGPREP);
      // 获取下一行的行号
      currentline = nextline(p, currentline, 0);
      // 跳过第一条指令（OP_VARARGPREP）
      i = 1;  /* skip first instruction (OP_VARARGPREP) */
    }
    // 对于每一条指令
    for (; i < p->sizelineinfo; i++) {  /* for each instruction */
      // 获取指令所在的行号
      currentline = nextline(p, currentline, i);  /* get its line */
      // 将true值存入表中对应的行号位置
      luaH_setint(L, t, currentline, &v);  /* table[line] = true */
    }
}

static const char *getfuncname (lua_State *L, CallInfo *ci, const char **name) {
  /* 获取函数名 */
  if (ci != NULL && !(ci->callstatus & CIST_TAIL))
    return funcnamefromcall(L, ci->previous, name);
  else return NULL;  /* 无法找到函数名 */
}

static int auxgetinfo (lua_State *L, const char *what, lua_Debug *ar,
                       Closure *f, CallInfo *ci) {
  int status = 1;
  for (; *what; what++) {
    switch (*what) {
      case 'S': {
        funcinfo(ar, f);
        break;
      }
      case 'l': {
        ar->currentline = (ci && isLua(ci)) ? getcurrentline(ci) : -1;
        break;
      }
      case 'u': {
        ar->nups = (f == NULL) ? 0 : f->c.nupvalues;
        if (noLuaClosure(f)) {
          ar->isvararg = 1;
          ar->nparams = 0;
        }
        else {
          ar->isvararg = f->l.p->is_vararg;
          ar->nparams = f->l.p->numparams;
        }
        break;
      }
      case 't': {
        ar->istailcall = (ci) ? ci->callstatus & CIST_TAIL : 0;
        break;
      }
      case 'n': {
        ar->namewhat = getfuncname(L, ci, &ar->name);
        if (ar->namewhat == NULL) {
          ar->namewhat = "";  /* 未找到 */
          ar->name = NULL;
        }
        break;
      }
      case 'r': {
        if (ci == NULL || !(ci->callstatus & CIST_TRAN))
          ar->ftransfer = ar->ntransfer = 0;
        else {
          ar->ftransfer = ci->u2.transferinfo.ftransfer;
          ar->ntransfer = ci->u2.transferinfo.ntransfer;
        }
        break;
      }
      case 'L':
      case 'f':  /* 由 lua_getinfo 处理 */
        break;
      default: status = 0;  /* 无效选项 */
    }
  }
  return status;
}

LUA_API int lua_getinfo (lua_State *L, const char *what, lua_Debug *ar) {
  int status;
  Closure *cl;
  CallInfo *ci;
  TValue *func;
  lua_lock(L);
  if (*what == '>') {
    ci = NULL;
    func = s2v(L->top - 1);
    api_check(L, ttisfunction(func), "function expected");
    what++;  /* 跳过'>' */
    L->top--;  /* 弹出函数 */
  }
  else {
    ci = ar->i_ci;
    func = s2v(ci->func);
    lua_assert(ttisfunction(func));
  }
  cl = ttisclosure(func) ? clvalue(func) : NULL;  /* 如果func是闭包，则获取闭包的值，否则为NULL */
  status = auxgetinfo(L, what, ar, cl, ci);  /* 获取函数信息 */
  if (strchr(what, 'f')) {  /* 如果what包含'f' */
    setobj2s(L, L->top, func);  /* 设置栈顶元素为func */
    api_incr_top(L);  /* 栈顶指针上移 */
  }
  if (strchr(what, 'L'))  /* 如果what包含'L' */
    collectvalidlines(L, cl);  /* 收集有效行号 */
  lua_unlock(L);  /* 解锁Lua状态 */
  return status;  /* 返回状态 */
/*
** {======================================================
** Symbolic Execution
** =======================================================
*/
*/

# 定义一个静态函数，用于获取常量 'c' 的名称
static const char *getobjname (const Proto *p, int lastpc, int reg,
                               const char **name);


# 查找常量 'c' 的名称
static void kname (const Proto *p, int c, const char **name) {
  TValue *kvalue = &p->k[c];
  *name = (ttisstring(kvalue)) ? svalue(kvalue) : "?";
}


# 查找寄存器 'c' 的名称
static void rname (const Proto *p, int pc, int c, const char **name) {
  const char *what = getobjname(p, pc, c, name); /* search for 'c' */
  if (!(what && *what == 'c'))  /* did not find a constant name? */
    *name = "?";
}


# 查找在 RK 指令中 'C' 值的名称
static void rkname (const Proto *p, int pc, Instruction i, const char **name) {
  int c = GETARG_C(i);  /* key index */
  if (GETARG_k(i))  /* is 'c' a constant? */
    kname(p, c, name);
  else  /* 'c' is a register */
    rname(p, pc, c, name);
}


# 过滤指令位置，返回最后一个修改寄存器 'reg' 的指令位置
static int filterpc (int pc, int jmptarget) {
  if (pc < jmptarget)  /* is code conditional (inside a jump)? */
    return -1;  /* cannot know who sets that register */
  else return pc;  /* current position sets that register */
}


# 尝试找到在 'lastpc' 之前修改寄存器 'reg' 的最后一条指令
static int findsetreg (const Proto *p, int lastpc, int reg) {
  int pc;
  int setreg = -1;  /* keep last instruction that changed 'reg' */
  int jmptarget = 0;  /* any code before this address is conditional */
  if (testMMMode(GET_OPCODE(p->code[lastpc])))
    lastpc--;  /* previous instruction was not actually executed */
  for (pc = 0; pc < lastpc; pc++) {
    Instruction i = p->code[pc];
    OpCode op = GET_OPCODE(i);
    int a = GETARG_A(i);
    int change;  /* true if current instruction changed 'reg' */
    switch (op) {
      case OP_LOADNIL: {  /* set registers from 'a' to 'a+b' */
        int b = GETARG_B(i);
        // 获取指令中的参数 B
        change = (a <= reg && reg <= a + b);
        // 判断寄存器是否在指定范围内
        break;
      }
      case OP_TFORCALL: {  /* affect all regs above its base */
        change = (reg >= a + 2);
        // 判断寄存器是否在指定范围内
        break;
      }
      case OP_CALL:
      case OP_TAILCALL: {  /* affect all registers above base */
        change = (reg >= a);
        // 判断寄存器是否在指定范围内
        break;
      }
      case OP_JMP: {  /* doesn't change registers, but changes 'jmptarget' */
        int b = GETARG_sJ(i);
        int dest = pc + 1 + b;
        // 获取指令中的参数 sJ，计算跳转目标地址
        /* jump does not skip 'lastpc' and is larger than current one? */
        if (dest <= lastpc && dest > jmptarget)
          jmptarget = dest;  /* update 'jmptarget' */
        // 如果跳转目标地址在合理范围内，则更新跳转目标地址
        change = 0;
        break;
      }
      default:  /* any instruction that sets A */
        change = (testAMode(op) && reg == a);
        // 判断指令是否设置 A 寄存器
        break;
    }
    if (change)
      setreg = filterpc(pc, jmptarget);
  }
  return setreg;
/*
** 检查指令'i'索引的表是否是环境'_ENV'
*/
static const char *gxf (const Proto *p, int pc, Instruction i, int isup) {
  int t = GETARG_B(i);  /* 表的索引 */
  const char *name;  /* 被索引变量的名称 */
  if (isup)  /* 是否是上值？ */
    name = upvalname(p, t);
  else
    getobjname(p, pc, t, &name);
  return (name && strcmp(name, LUA_ENV) == 0) ? "global" : "field";
}


static const char *getobjname (const Proto *p, int lastpc, int reg,
                               const char **name) {
  int pc;
  *name = luaF_getlocalname(p, reg + 1, lastpc);
  if (*name)  /* 是否是局部变量？ */
    return "local";
  /* 否则尝试符号执行 */
  pc = findsetreg(p, lastpc, reg);
  if (pc != -1) {  /* 能找到指令？ */
    Instruction i = p->code[pc];
    OpCode op = GET_OPCODE(i);
    # 根据操作码执行不同的操作
    switch (op) {
      case OP_MOVE: {
        # 获取指令中的参数 B，表示从位置 b 移动到位置 a
        int b = GETARG_B(i);  /* move from 'b' to 'a' */
        # 如果 b 小于 a，则返回位置 b 对应的对象的名称
        if (b < GETARG_A(i))
          return getobjname(p, pc, b, name);  /* get name for 'b' */
        # 否则跳出当前 case
        break;
      }
      case OP_GETTABUP: {
        # 获取指令中的参数 C，表示键的索引
        int k = GETARG_C(i);  /* key index */
        # 根据键的索引获取键的名称
        kname(p, k, name);
        # 返回全局表中指定键的值
        return gxf(p, pc, i, 1);
      }
      case OP_GETTABLE: {
        # 获取指令中的参数 C，表示键的索引
        int k = GETARG_C(i);  /* key index */
        # 根据键的索引获取键的名称
        rname(p, pc, k, name);
        # 返回表中指定键的值
        return gxf(p, pc, i, 0);
      }
      case OP_GETI: {
        # 将名称设置为 "integer index"
        *name = "integer index";
        # 返回字段类型为 "field"
        return "field";
      }
      case OP_GETFIELD: {
        # 获取指令中的参数 C，表示键的索引
        int k = GETARG_C(i);  /* key index */
        # 根据键的索引获取键的名称
        kname(p, k, name);
        # 返回表中指定键的值
        return gxf(p, pc, i, 0);
      }
      case OP_GETUPVAL: {
        # 将名称设置为指定 upvalue 的名称
        *name = upvalname(p, GETARG_B(i));
        # 返回字段类型为 "upvalue"
        return "upvalue";
      }
      case OP_LOADK:
      case OP_LOADKX: {
        # 如果操作码为 OP_LOADK，则获取参数 Bx，否则获取下一条指令的参数 Ax
        int b = (op == OP_LOADK) ? GETARG_Bx(i)
                                 : GETARG_Ax(p->code[pc + 1]);
        # 如果常量表中对应位置的值为字符串类型，则将名称设置为该字符串值，返回字段类型为 "constant"
        if (ttisstring(&p->k[b])) {
          *name = svalue(&p->k[b]);
          return "constant";
        }
        # 否则跳出当前 case
        break;
      }
      case OP_SELF: {
        # 获取指令中的参数 C，表示键的索引
        rkname(p, pc, i, name);
        # 返回字段类型为 "method"
        return "method";
      }
      default: break;  /* 继续执行，返回 NULL */
    }
  }
  # 未找到合理的名称，返回 NULL
  return NULL;  /* could not find reasonable name */
/*
** 根据调用它的代码尝试为函数找到一个名称。
** （仅在函数被 Lua 函数调用时有效。）
** 返回名称是什么（例如，“for iterator”，“方法”，“元方法”）并将 '*name' 设置为指向名称的指针。
*/
static const char *funcnamefromcode (lua_State *L, const Proto *p,
                                     int pc, const char **name) {
  TMS tm = (TMS)0;  /* （初始值避免警告） */
  Instruction i = p->code[pc];  /* 调用指令 */
  switch (GET_OPCODE(i)) {
    case OP_CALL:
    case OP_TAILCALL:
      return getobjname(p, pc, GETARG_A(i), name);  /* 获取函数名称 */
    case OP_TFORCALL: {  /* for 循环迭代器 */
      *name = "for iterator";
       return "for iterator";
    }
    /* 其他指令可以通过元方法进行调用 */
    case OP_SELF: case OP_GETTABUP: case OP_GETTABLE:
    case OP_GETI: case OP_GETFIELD:
      tm = TM_INDEX;
      break;
    case OP_SETTABUP: case OP_SETTABLE: case OP_SETI: case OP_SETFIELD:
      tm = TM_NEWINDEX;
      break;
    case OP_MMBIN: case OP_MMBINI: case OP_MMBINK: {
      tm = cast(TMS, GETARG_C(i));
      break;
    }
    case OP_UNM: tm = TM_UNM; break;
    case OP_BNOT: tm = TM_BNOT; break;
    case OP_LEN: tm = TM_LEN; break;
    case OP_CONCAT: tm = TM_CONCAT; break;
    case OP_EQ: tm = TM_EQ; break;
    /* 对于 OP_EQI 和 OP_EQK 没有情况，因为它们不调用元方法 */
    case OP_LT: case OP_LTI: case OP_GTI: tm = TM_LT; break;
    case OP_LE: case OP_LEI: case OP_GEI: tm = TM_LE; break;
    case OP_CLOSE: case OP_RETURN: tm = TM_CLOSE; break;
    default:
      return NULL;  /* 无法找到合理的名称 */
  }
  *name = getstr(G(L)->tmname[tm]) + 2;
  return "metamethod";
}


/*
** 尝试根据调用方式为函数找到一个名称。
*/
/*
** 从调用中获取函数名
** 如果在钩子函数中调用，则返回"hook"
** 如果作为终结器调用，则返回"metamethod"
** 如果是 Lua 函数，则调用 funcnamefromcode 函数获取函数名
** 否则返回 NULL
*/
static const char *funcnamefromcall (lua_State *L, CallInfo *ci,
                                                   const char **name) {
  if (ci->callstatus & CIST_HOOKED) {  /* 是否在钩子函数中调用？ */
    *name = "?";
    return "hook";
  }
  else if (ci->callstatus & CIST_FIN) {  /* 是否作为终结器调用？ */
    *name = "__gc";
    return "metamethod";  /* 作为元方法报告 */
  }
  else if (isLua(ci))
    return funcnamefromcode(L, ci_func(ci)->p, currentpc(ci), name);
  else
    return NULL;
}

/* }====================================================== */



/*
** 检查指针'o'是否指向当前函数堆栈帧中的某个值
** 因为'o'可能不指向此堆栈中的值，所以不能将其与区域边界进行比较（在 ISO C 中是未定义行为）
*/
static int isinstack (CallInfo *ci, const TValue *o) {
  StkId pos;
  for (pos = ci->func + 1; pos < ci->top; pos++) {
    if (o == s2v(pos))
      return 1;
  }
  return 0;  /* 未找到 */
}


/*
** 检查值'o'是否来自闭包
** （这只能发生在指令 OP_GETTABUP/OP_SETTABUP 中，这些指令直接操作闭包）
*/
static const char *getupvalname (CallInfo *ci, const TValue *o,
                                 const char **name) {
  LClosure *c = ci_func(ci);
  int i;
  for (i = 0; i < c->nupvalues; i++) {
    if (c->upvals[i]->v == o) {
      *name = upvalname(c->p, i);
      return "upvalue";
    }
  }
  return NULL;
}


/*
** 格式化变量信息
** 如果 kind 为 NULL，则返回空字符串
** 否则返回格式化后的字符串，格式为 "(kind 'name')"
*/
static const char *formatvarinfo (lua_State *L, const char *kind,
                                                const char *name) {
  if (kind == NULL)
    return "";  /* 没有信息 */
  else
    return luaO_pushfstring(L, " (%s '%s')", kind, name);
}

/*
** 构建一个描述值'o'的字符串，例如"变量'x'"或"闭包'y'"
*/
/*
** 返回变量信息
*/
static const char *varinfo (lua_State *L, const TValue *o) {
  // 获取当前调用信息
  CallInfo *ci = L->ci;
  // 为了避免警告，初始化变量名和类型
  const char *name = NULL;
  const char *kind = NULL;
  // 如果当前调用是 Lua 函数
  if (isLua(ci)) {
    // 获取 upvalue 的名字
    kind = getupvalname(ci, o, &name);
    // 如果没有找到 upvalue，尝试在寄存器中查找
    if (!kind && isinstack(ci, o))
      kind = getobjname(ci_func(ci)->p, currentpc(ci),
                        cast_int(cast(StkId, o) - (ci->func + 1)), &name);
  }
  // 返回格式化后的变量信息
  return formatvarinfo(L, kind, name);
}


/*
** 抛出类型错误
*/
static l_noret typeerror (lua_State *L, const TValue *o, const char *op,
                          const char *extra) {
  // 获取对象的类型
  const char *t = luaT_objtypename(L, o);
  // 抛出错误信息
  luaG_runerror(L, "attempt to %s a %s value%s", op, t, extra);
}


/*
** 抛出带有关于错误对象的“标准”信息的类型错误（使用 'varinfo'）
*/
l_noret luaG_typeerror (lua_State *L, const TValue *o, const char *op) {
  // 抛出类型错误
  typeerror(L, o, op, varinfo(L, o));
}


/*
** 抛出一个错误，表示调用了一个不可调用的对象。尝试根据调用方式找到对象的名称（'funcnamefromcall'）；
** 如果无法在那里找到名称，则尝试 'varinfo'。
*/
l_noret luaG_callerror (lua_State *L, const TValue *o) {
  // 获取当前调用信息
  CallInfo *ci = L->ci;
  // 为了避免警告，初始化变量名和类型
  const char *name = NULL;
  const char *kind = funcnamefromcall(L, ci, &name);
  // 根据调用方式获取额外信息
  const char *extra = kind ? formatvarinfo(L, kind, name) : varinfo(L, o);
  // 抛出类型错误
  typeerror(L, o, "call", extra);
}


/*
** 抛出一个 'for' 循环错误（期望数字，得到特定类型）
*/
l_noret luaG_forerror (lua_State *L, const TValue *o, const char *what) {
  // 抛出错误信息
  luaG_runerror(L, "bad 'for' %s (number expected, got %s)",
                   what, luaT_objtypename(L, o));
}


/*
** 抛出一个连接错误（期望字符串，得到特定类型）
*/
l_noret luaG_concaterror (lua_State *L, const TValue *p1, const TValue *p2) {
  // 如果第一个参数是字符串或者可以转换为字符串，则使用第二个参数
  if (ttisstring(p1) || cvt2str(p1)) p1 = p2;
  // 抛出类型错误
  luaG_typeerror(L, p1, "concatenate");
}
# 当两个值都可以转换为数字，但不能转换为整数时出错
l_noret luaG_opinterror (lua_State *L, const TValue *p1,
                         const TValue *p2, const char *msg) {
  # 如果第一个操作数不是数字类型
  if (!ttisnumber(p1))  /* first operand is wrong? */
    # 将第二个操作数设置为错误的类型
    p2 = p1;  /* now second is wrong */
  # 抛出类型错误异常
  luaG_typeerror(L, p2, msg);
}


/*
** 当两个值都可以转换为数字，但不能转换为整数时出错
*/
l_noret luaG_tointerror (lua_State *L, const TValue *p1, const TValue *p2) {
  # 临时整数变量
  lua_Integer temp;
  # 如果第一个值不能转换为整数
  if (!luaV_tointegerns(p1, &temp, LUA_FLOORN2I))
    # 将第二个操作数设置为错误的类型
    p2 = p1;
  # 抛出运行时错误异常
  luaG_runerror(L, "number%s has no integer representation", varinfo(L, p2));
}


l_noret luaG_ordererror (lua_State *L, const TValue *p1, const TValue *p2) {
  # 获取第一个值的类型名称
  const char *t1 = luaT_objtypename(L, p1);
  # 获取第二个值的类型名称
  const char *t2 = luaT_objtypename(L, p2);
  # 如果两个类型相同
  if (strcmp(t1, t2) == 0)
    # 抛出运行时错误异常
    luaG_runerror(L, "attempt to compare two %s values", t1);
  else
    # 抛出运行时错误异常
    luaG_runerror(L, "attempt to compare %s with %s", t1, t2);
}


/* add src:line information to 'msg' */
const char *luaG_addinfo (lua_State *L, const char *msg, TString *src,
                                        int line) {
  # 创建一个缓冲区
  char buff[LUA_IDSIZE];
  # 如果有源信息
  if (src)
    # 将源信息添加到缓冲区
    luaO_chunkid(buff, getstr(src), tsslen(src));
  else {  /* no source available; use "?" instead */
    # 如果没有源信息，使用 "?" 替代
    buff[0] = '?'; buff[1] = '\0';
  }
  # 返回带有源信息的消息
  return luaO_pushfstring(L, "%s:%d: %s", buff, line, msg);
}


l_noret luaG_errormsg (lua_State *L) {
  # 如果存在错误处理函数
  if (L->errfunc != 0) {  /* is there an error handling function? */
    # 恢复错误处理函数的栈位置
    StkId errfunc = restorestack(L, L->errfunc);
    # 断言错误处理函数是一个函数类型
    lua_assert(ttisfunction(s2v(errfunc)));
    # 移动参数
    setobjs2s(L, L->top, L->top - 1);  /* move argument */
    # 推入函数
    setobjs2s(L, L->top - 1, errfunc);  /* push function */
    # 增加栈顶指针
    L->top++;  /* assume EXTRA_STACK */
    # 调用错误处理函数
    luaD_callnoyield(L, L->top - 2, 1);  /* call it */
  }
  # 抛出运行时错误异常
  luaD_throw(L, LUA_ERRRUN);
}
/*
** 当出现运行时错误时，生成错误信息并抛出异常
** 参数：
**     L：Lua 状态机
**     fmt：错误信息格式字符串
** 返回：
**     无
*/
l_noret luaG_runerror (lua_State *L, const char *fmt, ...) {
  // 获取当前调用信息
  CallInfo *ci = L->ci;
  // 错误信息
  const char *msg;
  // 可变参数列表
  va_list argp;
  // 检查垃圾回收
  luaC_checkGC(L);  /* error message uses memory */
  // 开始处理可变参数
  va_start(argp, fmt);
  // 格式化错误信息
  msg = luaO_pushvfstring(L, fmt, argp);  /* format message */
  // 结束处理可变参数
  va_end(argp);
  // 如果是 Lua 函数，添加源码和行号信息
  if (isLua(ci))  /* if Lua function, add source:line information */
    luaG_addinfo(L, msg, ci_func(ci)->p->source, getcurrentline(ci));
  // 抛出错误信息
  luaG_errormsg(L);
}


/*
** 检查新指令 'newpc' 是否与上一个指令 'oldpc' 在不同的行
** 大多数情况下，'newpc' 仅在 'oldpc' 之后一两个指令（必须在之后，参见调用者），
** 因此尽量避免调用 'luaG_getfuncline'。如果它们相隔太远，很可能有 ABSLINEINFO，因此直接调用 'luaG_getfuncline'。
*/
static int changedline (const Proto *p, int oldpc, int newpc) {
  // 没有调试信息？
  if (p->lineinfo == NULL)
    return 0;
  // 不相隔太远？
  if (newpc - oldpc < MAXIWTHABS / 2) {
    // 行差
    int delta = 0;
    int pc = oldpc;
    for (;;) {
      int lineinfo = p->lineinfo[++pc];
      if (lineinfo == ABSLINEINFO)
        break;  /* cannot compute delta; fall through */
      delta += lineinfo;
      if (pc == newpc)
        return (delta != 0);  /* delta computed successfully */
    }
  }
  // 指令相隔太远或中间有绝对行信息；显式计算行差
  return (luaG_getfuncline(p, oldpc) != luaG_getfuncline(p, newpc));
}


/*
** 追踪 Lua 函数的执行。在执行每个操作码之前调用，当调试开启时。
** 'L->oldpc' 存储上一个追踪的指令，用于检测行号变化。
** 进入新函数时，'npci' 将为零，并且无论 'oldpc' 的值如何，都将测试为新行。
** 一些特殊情况可能在不设置 'oldpc' 的情况下返回到函数。在这种情况下，'oldpc' 可能是
*/
/* 
** invalid; if so, use zero as a valid value. (A wrong but valid 'oldpc'
** at most causes an extra call to a line hook.)
** This function is not "Protected" when called, so it should correct
** 'L->top' before calling anything that can run the GC.
*/
int luaG_traceexec (lua_State *L, const Instruction *pc) {
  // 获取当前调用信息
  CallInfo *ci = L->ci;
  // 获取当前的钩子掩码
  lu_byte mask = L->hookmask;
  // 获取当前函数的原型
  const Proto *p = ci_func(ci)->p;
  int counthook;
  // 如果没有设置行钩子和计数钩子，则直接返回
  if (!(mask & (LUA_MASKLINE | LUA_MASKCOUNT))) {  /* no hooks? */
    ci->u.l.trap = 0;  /* don't need to stop again */
    return 0;  /* turn off 'trap' */
  }
  pc++;  /* reference is always next instruction */
  ci->u.l.savedpc = pc;  /* save 'pc' */
  counthook = (--L->hookcount == 0 && (mask & LUA_MASKCOUNT));
  // 如果计数钩子触发，则重置计数
  if (counthook)
    resethookcount(L);  /* reset count */
  // 如果没有设置行钩子，则直接返回
  else if (!(mask & LUA_MASKLINE))
    return 1;  /* no line hook and count != 0; nothing to be done now */
  // 如果上次调用的是钩子函数，则不再调用
  if (ci->callstatus & CIST_HOOKYIELD) {  /* called hook last time? */
    ci->callstatus &= ~CIST_HOOKYIELD;  /* erase mark */
    return 1;  /* do not call hook again (VM yielded, so it did not move) */
  }
  // 如果顶部未被使用，则修正顶部
  if (!isIT(*(ci->u.l.savedpc - 1)))  /* top not being used? */
    L->top = ci->top;  /* correct top */
  // 如果计数钩子触发，则调用计数钩子
  if (counthook)
    luaD_hook(L, LUA_HOOKCOUNT, -1, 0, 0);  /* call count hook */
  // 如果设置了行钩子
  if (mask & LUA_MASKLINE) {
    /* 'L->oldpc' may be invalid; use zero in this case */
    // 获取旧的程序计数器值
    int oldpc = (L->oldpc < p->sizecode) ? L->oldpc : 0;
    // 获取相对于函数原型的程序计数器值
    int npci = pcRel(pc, p);
    // 如果跳转回去或者进入新的行，则调用行钩子
    if (npci <= oldpc ||  /* call hook when jump back (loop), */
        changedline(p, oldpc, npci)) {  /* or when enter new line */
      int newline = luaG_getfuncline(p, npci);
      luaD_hook(L, LUA_HOOKLINE, newline, 0, 0);  /* call line hook */
    }
    L->oldpc = npci;  /* 'pc' of last call to line hook */
  }
  // 如果钩子函数导致了暂停
  if (L->status == LUA_YIELD) {  /* did hook yield? */
    // 如果计数钩子触发，则撤销减量操作
    if (counthook)
      L->hookcount = 1;  /* undo decrement to zero */
    ci->u.l.savedpc--;  /* undo increment (resume will increment it again) */
    # 将调用状态的标志位设置为CIST_HOOKYIELD，表示已经产生了yield
    ci->callstatus |= CIST_HOOKYIELD;  
    # 抛出一个LUA_YIELD异常，表示产生了yield
    luaD_throw(L, LUA_YIELD);
  }
  # 返回1，保持'trap'状态
  return 1;  /* keep 'trap' on */
# 代码块结束
```