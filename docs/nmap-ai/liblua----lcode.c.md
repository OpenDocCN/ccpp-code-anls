# `nmap\liblua\lcode.c`

```
/*
** $Id: lcode.c $
** Lua的代码生成器
** 请参见lua.h中的版权声明
*/

#define lcode_c
#define LUA_CORE

#include "lprefix.h"


#include <float.h>
#include <limits.h>
#include <math.h>
#include <stdlib.h>

#include "lua.h"

#include "lcode.h"
#include "ldebug.h"
#include "ldo.h"
#include "lgc.h"
#include "llex.h"
#include "lmem.h"
#include "lobject.h"
#include "lopcodes.h"
#include "lparser.h"
#include "lstring.h"
#include "ltable.h"
#include "lvm.h"


/* Lua函数中寄存器的最大数量（必须适合8位） */
#define MAXREGS        255


#define hasjumps(e)    ((e)->t != (e)->f)


static int codesJ (FuncState *fs, OpCode o, int sj, int k);



/* 语义错误 */
l_noret luaK_semerror (LexState *ls, const char *msg) {
  ls->t.token = 0;  /* 从最终消息中删除"near <token>" */
  luaX_syntaxerror(ls, msg);
}


/*
** 如果表达式是一个数字常量，则用它的值填充'v'并返回1。否则返回0。
*/
static int tonumeral (const expdesc *e, TValue *v) {
  if (hasjumps(e))
    return 0;  /* 不是数字 */
  switch (e->k) {
    case VKINT:
      if (v) setivalue(v, e->u.ival);
      return 1;
    case VKFLT:
      if (v) setfltvalue(v, e->u.nval);
      return 1;
    default: return 0;
  }
}


/*
** 从常量表达式中获取常量值
*/
static TValue *const2val (FuncState *fs, const expdesc *e) {
  lua_assert(e->k == VCONST);
  return &fs->ls->dyd->actvar.arr[e->u.info].k;
}


/*
** 如果表达式是一个常量，则用它的值填充'v'并返回1。否则返回0。
*/
int luaK_exp2const (FuncState *fs, const expdesc *e, TValue *v) {
  if (hasjumps(e))
    return 0;  /* 不是常量 */
  switch (e->k) {
    case VFALSE:
      setbfvalue(v);
      return 1;
    case VTRUE:
      setbtvalue(v);
      return 1;
    case VNIL:
      setnilvalue(v);
      return 1;
    case VKSTR: {
      setsvalue(fs->ls->L, v, e->u.strval);
      return 1;
    }
    # 如果表达式类型是常量
    case VCONST: {
      # 将常量转换为值，并设置给目标变量
      setobj(fs->ls->L, v, const2val(fs, e));
      # 返回设置成功
      return 1;
    }
    # 如果表达式类型不是常量，则将其转换为数值
    default: return tonumeral(e, v);
  }
/*
** 返回当前代码的前一条指令。如果当前指令和前一条指令之间可能有一个跳转目标，
** 则返回一个无效指令（以避免错误的优化）。
*/
static Instruction *previousinstruction (FuncState *fs) {
  static const Instruction invalidinstruction = ~(Instruction)0;
  if (fs->pc > fs->lasttarget)
    return &fs->f->code[fs->pc - 1];  /* 返回前一条指令 */
  else
    return cast(Instruction*, &invalidinstruction);
}


/*
** 创建一个 OP_LOADNIL 指令，但尝试进行优化：如果前一条指令也是 OP_LOADNIL 并且范围兼容，
** 则调整前一条指令的范围而不是生成新的指令。（例如，'local a; local b' 将生成一个单独的操作码。）
*/
void luaK_nil (FuncState *fs, int from, int n) {
  int l = from + n - 1;  /* 最后一个要设置为 nil 的寄存器 */
  Instruction *previous = previousinstruction(fs);
  if (GET_OPCODE(*previous) == OP_LOADNIL) {  /* 前一条指令是 LOADNIL 吗？ */
    int pfrom = GETARG_A(*previous);  /* 获取前一条指令的范围 */
    int pl = pfrom + GETARG_B(*previous);
    if ((pfrom <= from && from <= pl + 1) ||
        (from <= pfrom && pfrom <= l + 1)) {  /* 两者可以连接吗？ */
      if (pfrom < from) from = pfrom;  /* from = min(from, pfrom) */
      if (pl > l) l = pl;  /* l = max(l, pl) */
      SETARG_A(*previous, from);
      SETARG_B(*previous, l - from);
      return;
    }  /* 否则继续 */
  }
  luaK_codeABC(fs, OP_LOADNIL, from, n - 1, 0);  /* 否则不进行优化 */
}


/*
** 获取跳转指令的目标地址。用于遍历跳转列表。
*/
static int getjump (FuncState *fs, int pc) {
  int offset = GETARG_sJ(fs->f->code[pc]);
  if (offset == NO_JUMP)  /* 指向自身表示列表结束 */
    return NO_JUMP;  /* 列表结束 */
  else
    return (pc+1)+offset;  /* 将偏移转换为绝对位置 */
}


/*
** 修复位置 'pc' 处的跳转指令，使其跳转到 'dest'。
*/
/*
** 修正跳转地址
*/
static void fixjump (FuncState *fs, int pc, int dest) {
  // 获取跳转指令
  Instruction *jmp = &fs->f->code[pc];
  // 计算跳转偏移量
  int offset = dest - (pc + 1);
  // 确保目标地址不为空
  lua_assert(dest != NO_JUMP);
  // 如果偏移量超出范围，报错
  if (!(-OFFSET_sJ <= offset && offset <= MAXARG_sJ - OFFSET_sJ))
    luaX_syntaxerror(fs->ls, "control structure too long");
  // 确保跳转指令为OP_JMP
  lua_assert(GET_OPCODE(*jmp) == OP_JMP);
  // 设置跳转偏移量
  SETARG_sJ(*jmp, offset);
}


/*
** 将跳转列表'l2'连接到跳转列表'l1'中
*/
void luaK_concat (FuncState *fs, int *l1, int l2) {
  // 如果'l2'为空，直接返回
  if (l2 == NO_JUMP) return;  /* nothing to concatenate? */
  // 如果'l1'为空，直接将'l1'指向'l2'
  else if (*l1 == NO_JUMP)  /* no original list? */
    *l1 = l2;  /* 'l1' points to 'l2' */
  // 否则，找到'l1'的最后一个元素，将其指向'l2'
  else {
    int list = *l1;
    int next;
    while ((next = getjump(fs, list)) != NO_JUMP)  /* find last element */
      list = next;
    fixjump(fs, list, l2);  /* last element links to 'l2' */
  }
}


/*
** 创建一个跳转指令并返回其位置，以便稍后修正其目标地址
*/
int luaK_jump (FuncState *fs) {
  return codesJ(fs, OP_JMP, NO_JUMP, 0);
}


/*
** 编码'return'指令
*/
void luaK_ret (FuncState *fs, int first, int nret) {
  OpCode op;
  switch (nret) {
    case 0: op = OP_RETURN0; break;
    case 1: op = OP_RETURN1; break;
    default: op = OP_RETURN; break;
  }
  luaK_codeABC(fs, op, first, nret + 1, 0);
}


/*
** 编码"条件跳转"，即测试或比较操作码后面跟着一个跳转。返回跳转位置
*/
static int condjump (FuncState *fs, OpCode op, int A, int B, int C, int k) {
  // 编码条件跳转指令
  luaK_codeABCk(fs, op, A, B, C, k);
  // 返回跳转位置
  return luaK_jump(fs);
}


/*
** 返回当前'pc'并标记为跳转目标（以避免不在同一基本块中的连续指令出现错误的优化）
*/
int luaK_getlabel (FuncState *fs) {
  fs->lasttarget = fs->pc;
  return fs->pc;
}


/*
** 返回控制给定跳转（即其条件）的指令的位置，如果是无条件跳转，则返回跳转本身
*/
# 获取跳转指令的控制指令
static Instruction *getjumpcontrol (FuncState *fs, int pc) {
  # 获取当前指令
  Instruction *pi = &fs->f->code[pc];
  # 如果当前指令前面有一条指令，并且前一条指令是测试指令，返回前一条指令的地址
  if (pc >= 1 && testTMode(GET_OPCODE(*(pi-1))))
    return pi-1;
  else
    return pi;
}

/*
** 修补 TESTSET 指令的目标寄存器
** 如果 'node' 位置的指令不是 TESTSET，则返回 0（"失败"）
** 否则，如果 'reg' 不是 'NO_REG'，将其设置为目标寄存器
** 否则，将指令更改为简单的 'TEST'（不产生寄存器值）
*/
static int patchtestreg (FuncState *fs, int node, int reg) {
  # 获取控制指令
  Instruction *i = getjumpcontrol(fs, node);
  # 如果控制指令不是 TESTSET，则无法修补其他指令
  if (GET_OPCODE(*i) != OP_TESTSET)
    return 0;
  # 如果寄存器不是 NO_REG 并且不等于指令的 B 参数，则将 A 参数设置为 reg
  if (reg != NO_REG && reg != GETARG_B(*i))
    SETARG_A(*i, reg);
  else {
     /* 没有寄存器放置值或寄存器已经有值；
        将指令更改为简单的测试 */
    *i = CREATE_ABCk(OP_TEST, GETARG_B(*i), 0, 0, GETARG_k(*i));
  }
  return 1;
}

/*
** 遍历测试列表，确保没有一个产生值
*/
static void removevalues (FuncState *fs, int list) {
  # 遍历测试列表，将其目标寄存器修补为 NO_REG
  for (; list != NO_JUMP; list = getjump(fs, list))
      patchtestreg(fs, list, NO_REG);
}

/*
** 遍历测试列表，修补它们的目标地址和寄存器：
** 产生值的测试跳转到 'vtarget'（并将它们的值放在 'reg' 中），其他测试跳转到 'dtarget'
*/
static void patchlistaux (FuncState *fs, int list, int vtarget, int reg,
                          int dtarget) {
  while (list != NO_JUMP) {
    int next = getjump(fs, list);
    # 如果修补了测试指令的目标寄存器，则修复跳转到 vtarget
    if (patchtestreg(fs, list, reg))
      fixjump(fs, list, vtarget);
    else
      fixjump(fs, list, dtarget);  /* 跳转到默认目标 */
    list = next;
  }
}

/*
** 将 'list' 中的所有跳转路径修补为跳转到 'target'
** （断言意味着我们不能修复到一个前向地址的跳转，因为只有在生成代码时才知道地址）
*/
void luaK_patchlist (FuncState *fs, int list, int target) {
  // 断言目标位置在当前函数状态的指令计数范围内
  lua_assert(target <= fs->pc);
  // 调用辅助函数，对指定列表中的跳转目标进行修补
  patchlistaux(fs, list, target, NO_REG, target);
}


void luaK_patchtohere (FuncState *fs, int list) {
  // 获取当前位置的标签作为跳转目标
  int hr = luaK_getlabel(fs);  /* mark "here" as a jump target */
  // 对指定列表中的跳转目标进行修补，使其跳转到当前位置
  luaK_patchlist(fs, list, hr);
}


/* limit for difference between lines in relative line info. */
#define LIMLINEDIFF    0x80


/*
** Save line info for a new instruction. If difference from last line
** does not fit in a byte, of after that many instructions, save a new
** absolute line info; (in that case, the special value 'ABSLINEINFO'
** in 'lineinfo' signals the existence of this absolute information.)
** Otherwise, store the difference from last line in 'lineinfo'.
*/
static void savelineinfo (FuncState *fs, Proto *f, int line) {
  // 计算当前行号与上一条指令行号的差值
  int linedif = line - fs->previousline;
  int pc = fs->pc - 1;  /* last instruction coded */
  // 如果行号差值超过阈值或者累计的绝对行号信息数量达到上限
  if (abs(linedif) >= LIMLINEDIFF || fs->iwthabs++ >= MAXIWTHABS) {
    // 扩展函数原型中的绝对行号信息数组
    luaM_growvector(fs->ls->L, f->abslineinfo, fs->nabslineinfo,
                    f->sizeabslineinfo, AbsLineInfo, MAX_INT, "lines");
    // 将当前指令位置和行号信息添加到绝对行号信息数组中
    f->abslineinfo[fs->nabslineinfo].pc = pc;
    f->abslineinfo[fs->nabslineinfo++].line = line;
    // 将行号差值设为特殊值，表示存在绝对行号信息
    linedif = ABSLINEINFO;  /* signal that there is absolute information */
    // 重置累计的绝对行号信息数量
    fs->iwthabs = 1;  /* restart counter */
  }
  // 扩展函数原型中的相对行号信息数组，并存储行号差值
  luaM_growvector(fs->ls->L, f->lineinfo, pc, f->sizelineinfo, ls_byte,
                  MAX_INT, "opcodes");
  f->lineinfo[pc] = linedif;
  // 更新上一条指令的行号信息
  fs->previousline = line;  /* last line saved */
}


/*
** Remove line information from the last instruction.
** If line information for that instruction is absolute, set 'iwthabs'
** above its max to force the new (replacing) instruction to have
** absolute line info, too.
*/
static void removelastlineinfo (FuncState *fs) {
  Proto *f = fs->f;
  int pc = fs->pc - 1;  /* last instruction coded */
  // 如果最后一条指令的行号信息是相对行号信息
  if (f->lineinfo[pc] != ABSLINEINFO) {  /* relative line info? */
    # 减去上一个保存的行数，以纠正上一行的信息
    fs->previousline -= f->lineinfo[pc];  
    # 撤销先前的增量
    fs->iwthabs--;  
  }
  else {  # 绝对行信息
    # 断言，确保绝对行信息数组中最后一个元素的程序计数器值等于当前程序计数器值
    lua_assert(f->abslineinfo[fs->nabslineinfo - 1].pc == pc);
    # 移除最后一个绝对行信息
    fs->nabslineinfo--;  
    # 强制下一个行信息为绝对行信息
    fs->iwthabs = MAXIWTHABS + 1;  
  }
/*
** Remove the last instruction created, correcting line information
** accordingly.
*/
static void removelastinstruction (FuncState *fs) {
  // 移除最后创建的指令，并相应地修正行信息
  removelastlineinfo(fs);
  fs->pc--;
}


/*
** Emit instruction 'i', checking for array sizes and saving also its
** line information. Return 'i' position.
*/
int luaK_code (FuncState *fs, Instruction i) {
  Proto *f = fs->f;
  /* put new instruction in code array */
  // 将新指令放入代码数组中
  luaM_growvector(fs->ls->L, f->code, fs->pc, f->sizecode, Instruction,
                  MAX_INT, "opcodes");
  f->code[fs->pc++] = i;
  // 保存指令的行信息
  savelineinfo(fs, f, fs->ls->lastline);
  return fs->pc - 1;  /* index of new instruction */
}


/*
** Format and emit an 'iABC' instruction. (Assertions check consistency
** of parameters versus opcode.)
*/
int luaK_codeABCk (FuncState *fs, OpCode o, int a, int b, int c, int k) {
  lua_assert(getOpMode(o) == iABC);
  lua_assert(a <= MAXARG_A && b <= MAXARG_B &&
             c <= MAXARG_C && (k & ~1) == 0);
  return luaK_code(fs, CREATE_ABCk(o, a, b, c, k));
}


/*
** Format and emit an 'iABx' instruction.
*/
int luaK_codeABx (FuncState *fs, OpCode o, int a, unsigned int bc) {
  lua_assert(getOpMode(o) == iABx);
  lua_assert(a <= MAXARG_A && bc <= MAXARG_Bx);
  return luaK_code(fs, CREATE_ABx(o, a, bc));
}


/*
** Format and emit an 'iAsBx' instruction.
*/
int luaK_codeAsBx (FuncState *fs, OpCode o, int a, int bc) {
  unsigned int b = bc + OFFSET_sBx;
  lua_assert(getOpMode(o) == iAsBx);
  lua_assert(a <= MAXARG_A && b <= MAXARG_Bx);
  return luaK_code(fs, CREATE_ABx(o, a, b));
}


/*
** Format and emit an 'isJ' instruction.
*/
static int codesJ (FuncState *fs, OpCode o, int sj, int k) {
  unsigned int j = sj + OFFSET_sJ;
  lua_assert(getOpMode(o) == isJ);
  lua_assert(j <= MAXARG_sJ && (k & ~1) == 0);
  return luaK_code(fs, CREATE_sJ(o, j, k));
}


/*
** Emit an "extra argument" instruction (format 'iAx')
*/
/*
** 根据给定的参数a生成一个指令，用于扩展参数
** 参数a必须小于等于MAXARG_Ax
*/
static int codeextraarg (FuncState *fs, int a) {
  lua_assert(a <= MAXARG_Ax);
  return luaK_code(fs, CREATE_Ax(OP_EXTRAARG, a));
}


/*
** 发出一个“加载常量”指令，使用'OP_LOADK'（如果常量索引'k'适合18位），
** 或者使用带有“额外参数”的'OP_LOADKX'指令
*/
static int luaK_codek (FuncState *fs, int reg, int k) {
  if (k <= MAXARG_Bx)
    return luaK_codeABx(fs, OP_LOADK, reg, k);
  else {
    int p = luaK_codeABx(fs, OP_LOADKX, reg, 0);
    codeextraarg(fs, k);
    return p;
  }
}


/*
** 检查寄存器堆栈级别，并在字段'maxstacksize'中跟踪其最大大小
*/
void luaK_checkstack (FuncState *fs, int n) {
  int newstack = fs->freereg + n;
  if (newstack > fs->f->maxstacksize) {
    if (newstack >= MAXREGS)
      luaX_syntaxerror(fs->ls,
        "function or expression needs too many registers");
    fs->f->maxstacksize = cast_byte(newstack);
  }
}


/*
** 在寄存器堆栈中保留'n'个寄存器
*/
void luaK_reserveregs (FuncState *fs, int n) {
  luaK_checkstack(fs, n);
  fs->freereg += n;
}


/*
** 释放寄存器'reg'，如果它既不是常量索引也不是局部变量
*/
static void freereg (FuncState *fs, int reg) {
  if (reg >= luaY_nvarstack(fs)) {
    fs->freereg--;
    lua_assert(reg == fs->freereg);
  }
}


/*
** 以正确的顺序释放两个寄存器
*/
static void freeregs (FuncState *fs, int r1, int r2) {
  if (r1 > r2) {
    freereg(fs, r1);
    freereg(fs, r2);
  }
  else {
    freereg(fs, r2);
    freereg(fs, r1);
  }
}


/*
** 释放表达式'e'使用的寄存器（如果有的话）
*/
static void freeexp (FuncState *fs, expdesc *e) {
  if (e->k == VNONRELOC)
    freereg(fs, e->u.info);
}


/*
** 以正确的顺序释放表达式'e1'和'e2'使用的寄存器（如果有的话）
*/
static void freeexps (FuncState *fs, expdesc *e1, expdesc *e2) {
  int r1 = (e1->k == VNONRELOC) ? e1->u.info : -1;
  int r2 = (e2->k == VNONRELOC) ? e2->u.info : -1;
  freeregs(fs, r1, r2);
}
/*
** 将常量 'v' 添加到原型的常量列表中（字段 'k'）。
** 使用扫描器的表来缓存常量在常量列表中的位置，并尝试重用常量。
** 因为某些值不应该被用作键（nil 不能作为键，整数键可能与浮点键重叠），
** 调用者必须提供一个有用的 'key' 来索引缓存。
** 注意，所有函数共享同一个表，因此进入或退出函数可能会使一些索引错误。
*/
static int addk (FuncState *fs, TValue *key, TValue *v) {
  TValue val;
  lua_State *L = fs->ls->L;
  Proto *f = fs->f;
  const TValue *idx = luaH_get(fs->ls->h, key);  /* 查询扫描器表 */
  int k, oldsize;
  if (ttisinteger(idx)) {  /* 是否有索引？ */
    k = cast_int(ivalue(idx));
    /* 正确的值？（警告：必须区分浮点数和整数！） */
    if (k < fs->nk && ttypetag(&f->k[k]) == ttypetag(v) &&
                      luaV_rawequalobj(&f->k[k], v))
      return k;  /* 重用索引 */
  }
  /* 未找到常量；创建新条目 */
  oldsize = f->sizek;
  k = fs->nk;
  /* 数值不需要 GC 屏障；
     表没有元表，因此不需要使缓存无效 */
  setivalue(&val, k);
  luaH_finishset(L, fs->ls->h, key, idx, &val);
  luaM_growvector(L, f->k, k, f->sizek, TValue, MAXARG_Ax, "constants");
  while (oldsize < f->sizek) setnilvalue(&f->k[oldsize++]);
  setobj(L, &f->k[k], v);
  fs->nk++;
  luaC_barrier(L, f, v);
  return k;
}


/*
** 将字符串添加到常量列表中并返回其索引。
*/
static int stringK (FuncState *fs, TString *s) {
  TValue o;
  setsvalue(fs->ls->L, &o, s);
  return addk(fs, &o, &o);  /* 使用字符串本身作为键 */
}


/*
** 将整数添加到常量列表中并返回其索引。
*/
static int luaK_intK (FuncState *fs, lua_Integer n) {
  TValue o;
  setivalue(&o, n);
  return addk(fs, &o, &o);  /* 使用整数本身作为键 */
}

/*
** 将浮点数添加到常量列表中并返回其索引。浮点数
*/
/*
** with integral values need a different key, to avoid collision
** with actual integers. To that, we add to the number its smaller
** power-of-two fraction that is still significant in its scale.
** For doubles, that would be 1/2^52.
** (This method is not bulletproof: there may be another float
** with that value, and for floats larger than 2^53 the result is
** still an integer. At worst, this only wastes an entry with
** a duplicate.)
*/
static int luaK_numberK (FuncState *fs, lua_Number r) {
  TValue o;  // 创建一个 TValue 类型的变量 o
  lua_Integer ik;  // 创建一个 lua_Integer 类型的变量 ik
  setfltvalue(&o, r);  // 将 r 转换为浮点数类型的值，并赋给 o
  if (!luaV_flttointeger(r, &ik, F2Ieq))  /* not an integral value? */  // 如果 r 不是整数值
    return addk(fs, &o, &o);  /* use number itself as key */  // 将 o 添加到常量表中，并返回其索引
  else {  /* must build an alternative key */  // 否则，需要构建一个替代键
    const int nbm = l_floatatt(MANT_DIG);  // 获取浮点数的尾数位数
    const lua_Number q = l_mathop(ldexp)(l_mathop(1.0), -nbm + 1);  // 计算 1/2^52
    const lua_Number k = (ik == 0) ? q : r + r*q;  /* new key */  // 计算新的键值
    TValue kv;  // 创建一个 TValue 类型的变量 kv
    setfltvalue(&kv, k);  // 将 k 转换为浮点数类型的值，并赋给 kv
    /* result is not an integral value, unless value is too large */
    lua_assert(!luaV_flttointeger(k, &ik, F2Ieq) || l_mathop(fabs)(r) >= l_mathop(1e6));  // 断言，除非值太大，否则结果不是整数值
    return addk(fs, &kv, &o);  // 将 kv 添加到常量表中，并返回其索引
  }
}


/*
** Add a false to list of constants and return its index.
*/
static int boolF (FuncState *fs) {
  TValue o;  // 创建一个 TValue 类型的变量 o
  setbfvalue(&o);  // 将 o 设置为布尔值 false
  return addk(fs, &o, &o);  /* use boolean itself as key */  // 将 o 添加到常量表中，并返回其索引
}


/*
** Add a true to list of constants and return its index.
*/
static int boolT (FuncState *fs) {
  TValue o;  // 创建一个 TValue 类型的变量 o
  setbtvalue(&o);  // 将 o 设置为布尔值 true
  return addk(fs, &o, &o);  /* use boolean itself as key */  // 将 o 添加到常量表中，并返回其索引
}


/*
** Add nil to list of constants and return its index.
*/
static int nilK (FuncState *fs) {
  TValue k, v;  // 创建两个 TValue 类型的变量 k 和 v
  setnilvalue(&v);  // 将 v 设置为 nil
  /* cannot use nil as key; instead use table itself to represent nil */
  sethvalue(fs->ls->L, &k, fs->ls->h);  // 将表 fs->ls->h 设置为键 k
  return addk(fs, &k, &v);  // 将 k 和 v 添加到常量表中，并返回其索引
}


/*
** Check whether 'i' can be stored in an 'sC' operand. Equivalent to
** (0 <= int2sC(i) && int2sC(i) <= MAXARG_C) but without risk of
/*
** 检查 'i' 是否可以存储在 'sBx' 操作数中
*/
static int fitsBx (lua_Integer i) {
  return (-OFFSET_sBx <= i && i <= MAXARG_Bx - OFFSET_sBx);
}


/*
** 将常量 'v' 转换为表达式描述 'e'
*/
static void const2exp (TValue *v, expdesc *e) {
  switch (ttypetag(v)) {
    case LUA_VNUMINT:
      e->k = VKINT; e->u.ival = ivalue(v);
      break;
    case LUA_VNUMFLT:
      e->k = VKFLT; e->u.nval = fltvalue(v);
      break;
    case LUA_VFALSE:
      e->k = VFALSE;
      break;
    case LUA_VTRUE:
      e->k = VTRUE;
      break;
    case LUA_VNIL:
      e->k = VNIL;
      break;
    case LUA_VSHRSTR:  case LUA_VLNGSTR:
      e->k = VKSTR; e->u.strval = tsvalue(v);
      break;
    default: lua_assert(0);
  }
}


/*
** 修正表达式以返回结果数量 'nresults'。
** 'e' 必须是多返回值表达式（函数调用或可变参数）。
*/
void luaK_setreturns (FuncState *fs, expdesc *e, int nresults) {
  Instruction *pc = &getinstruction(fs, e);
  if (e->k == VCALL)  /* 表达式是一个开放的函数调用？ */
    SETARG_C(*pc, nresults + 1);
  else {
    lua_assert(e->k == VVARARG);
    SETARG_C(*pc, nresults + 1);
    SETARG_A(*pc, fs->freereg);
    luaK_reserveregs(fs, 1);
  }
}


/*
** 将 VKSTR 转换为 VK
*/
static void str2K (FuncState *fs, expdesc *e) {
  lua_assert(e->k == VKSTR);
  e->u.info = stringK(fs, e->u.strval);
  e->k = VK;
}
/*
** 修复表达式以返回一个结果。
** 如果表达式不是多返回表达式（函数调用或可变参数），它已经返回一个结果，所以不需要做任何事情。
** 函数调用变成 VNONRELOC 表达式（因为其结果固定在调用的基本寄存器中），而可变参数表达式变成 VRELOC（因为 OP_VARARG 将其结果放在它想要的位置）。
** （调用被创建为返回一个结果，所以不需要修复。）
*/
void luaK_setoneret (FuncState *fs, expdesc *e) {
  if (e->k == VCALL) {  /* 表达式是一个开放的函数调用？ */
    /* 已经返回 1 个值 */
    lua_assert(GETARG_C(getinstruction(fs, e)) == 2);
    e->k = VNONRELOC;  /* 结果有固定位置 */
    e->u.info = GETARG_A(getinstruction(fs, e));
  }
  else if (e->k == VVARARG) {
    SETARG_C(getinstruction(fs, e), 2);
    e->k = VRELOC;  /* 可以重新定位其简单结果 */
  }
}


/*
** 确保表达式 'e' 不是一个变量（也不是 <const>）。
** （表达式仍然可能有跳转列表。）
*/
void luaK_dischargevars (FuncState *fs, expdesc *e) {
  switch (e->k) {
    case VCONST: {
      const2exp(const2val(fs, e), e);
      break;
    }
    case VLOCAL: {  /* 已经在一个寄存器中 */
      e->u.info = e->u.var.ridx;
      e->k = VNONRELOC;  /* 变成一个不可重定位的值 */
      break;
    }
    case VUPVAL: {  /* 将值移动到某个（待定的）寄存器中 */
      e->u.info = luaK_codeABC(fs, OP_GETUPVAL, 0, e->u.info, 0);
      e->k = VRELOC;
      break;
    }
    case VINDEXUP: {
      e->u.info = luaK_codeABC(fs, OP_GETTABUP, 0, e->u.ind.t, e->u.ind.idx);
      e->k = VRELOC;
      break;
    }
    case VINDEXI: {
      freereg(fs, e->u.ind.t);
      e->u.info = luaK_codeABC(fs, OP_GETI, 0, e->u.ind.t, e->u.ind.idx);
      e->k = VRELOC;
      break;
    }
    case VINDEXSTR: {
      freereg(fs, e->u.ind.t);
      e->u.info = luaK_codeABC(fs, OP_GETFIELD, 0, e->u.ind.t, e->u.ind.idx);
      e->k = VRELOC;
      break;
    }
    # 如果表达式类型为索引表达式
    case VINDEXED: {
      # 释放表达式中使用的寄存器
      freeregs(fs, e->u.ind.t, e->u.ind.idx);
      # 生成获取表中元素的指令，并将结果保存在表达式中
      e->u.info = luaK_codeABC(fs, OP_GETTABLE, 0, e->u.ind.t, e->u.ind.idx);
      # 将表达式类型标记为可重定位
      e->k = VRELOC;
      # 跳出 switch 语句
      break;
    }
    # 如果表达式类型为可变参数或函数调用
    case VVARARG: case VCALL: {
      # 生成设置单返回值的指令
      luaK_setoneret(fs, e);
      # 跳出 switch 语句
      break;
    }
    # 如果表达式类型为其他类型
    default: break;  /* there is one value available (somewhere) */
  }
# 确保表达式值在寄存器'reg'中，使'e'成为不可重定位的表达式
# （表达式仍可能有跳转列表）
static void discharge2reg (FuncState *fs, expdesc *e, int reg) {
  luaK_dischargevars(fs, e);  # 释放变量
  switch (e->k) {
    case VNIL: {
      luaK_nil(fs, reg, 1);  # 将寄存器中的值设为nil
      break;
    }
    case VFALSE: {
      luaK_codeABC(fs, OP_LOADFALSE, reg, 0, 0);  # 将false加载到寄存器中
      break;
    }
    case VTRUE: {
      luaK_codeABC(fs, OP_LOADTRUE, reg, 0, 0);  # 将true加载到寄存器中
      break;
    }
    case VKSTR: {
      str2K(fs, e);  # 将字符串转换为常量
    }  # 继续执行下面的代码
    case VK: {
      luaK_codek(fs, reg, e->u.info);  # 将常量加载到寄存器中
      break;
    }
    case VKFLT: {
      luaK_float(fs, reg, e->u.nval);  # 将浮点数加载到寄存器中
      break;
    }
    case VKINT: {
      luaK_int(fs, reg, e->u.ival);  # 将整数加载到寄存器中
      break;
    }
    case VRELOC: {
      Instruction *pc = &getinstruction(fs, e);
      SETARG_A(*pc, reg);  # 指令将结果放入'reg'寄存器中
      break;
    }
    case VNONRELOC: {
      if (reg != e->u.info)
        luaK_codeABC(fs, OP_MOVE, reg, e->u.info, 0);  # 将值从一个寄存器移动到另一个寄存器
      break;
    }
    default: {
      lua_assert(e->k == VJMP);
      return;  # 无需执行任何操作
    }
  }
  e->u.info = reg;  # 将'reg'寄存器中的值赋给'e'的信息字段
  e->k = VNONRELOC;  # 将'e'标记为非重定位
}


# 确保表达式值在寄存器中，使'e'成为不可重定位的表达式
# （表达式仍可能有跳转列表）
static void discharge2anyreg (FuncState *fs, expdesc *e) {
  if (e->k != VNONRELOC) {  # 还没有固定寄存器？
    luaK_reserveregs(fs, 1);  # 获取一个寄存器
    discharge2reg(fs, e, fs->freereg-1);  # 将值放入寄存器中
  }
}


static int code_loadbool (FuncState *fs, int A, OpCode op) {
  luaK_getlabel(fs);  # 这些指令可能是跳转目标
  return luaK_codeABC(fs, op, A, 0, 0);  # 生成ABC模式的指令
}


# 检查列表中是否有任何不产生值或产生反转值的跳转
static int need_value (FuncState *fs, int list) {
  for (; list != NO_JUMP; list = getjump(fs, list)) {
    Instruction i = *getjumpcontrol(fs, list);
    # 如果指令 i 的操作码不等于 OP_TESTSET，则返回 1
    if (GET_OPCODE(i) != OP_TESTSET) return 1;
  }
  # 如果没有找到符合条件的情况，则返回 0
  return 0;  /* not found */
/*
** 确保最终表达式结果（包括其跳转列表的结果）在寄存器'reg'中。
** 如果表达式有跳转，需要将这些跳转修补到其最终位置或“load”指令（对于那些不产生值的测试）。
*/
static void exp2reg (FuncState *fs, expdesc *e, int reg) {
  discharge2reg(fs, e, reg);  // 将表达式的值放入寄存器'reg'中
  if (e->k == VJMP)  /* 表达式本身是一个测试吗？ */
    luaK_concat(fs, &e->t, e->u.info);  /* 将这个跳转放入't'列表中 */
  if (hasjumps(e)) {
    int final;  /* 整个表达式结束后的位置 */
    int p_f = NO_JUMP;  /* 一个可能的LOAD false的位置 */
    int p_t = NO_JUMP;  /* 一个可能的LOAD true的位置 */
    if (need_value(fs, e->t) || need_value(fs, e->f)) {
      int fj = (e->k == VJMP) ? NO_JUMP : luaK_jump(fs);  // 如果'e'不是一个测试，创建一个跳转
      p_f = code_loadbool(fs, reg, OP_LFALSESKIP);  /* 跳过下一条指令 */
      p_t = code_loadbool(fs, reg, OP_LOADTRUE);
      /* 如果'e'不是一个测试，跳过这些布尔值 */
      luaK_patchtohere(fs, fj);
    }
    final = luaK_getlabel(fs);
    patchlistaux(fs, e->f, final, reg, p_f);  // 修补假跳转列表
    patchlistaux(fs, e->t, final, reg, p_t);  // 修补真跳转列表
  }
  e->f = e->t = NO_JUMP;
  e->u.info = reg;
  e->k = VNONRELOC;
}


/*
** 确保最终表达式结果在下一个可用的寄存器中。
*/
void luaK_exp2nextreg (FuncState *fs, expdesc *e) {
  luaK_dischargevars(fs, e);  // 释放变量
  freeexp(fs, e);  // 释放表达式
  luaK_reserveregs(fs, 1);  // 预留一个寄存器
  exp2reg(fs, e, fs->freereg - 1);  // 将表达式结果放入下一个寄存器中
}


/*
** 确保最终表达式结果在某个（任意）寄存器中，并返回该寄存器。
*/
int luaK_exp2anyreg (FuncState *fs, expdesc *e) {
  luaK_dischargevars(fs, e);  // 释放变量
  if (e->k == VNONRELOC) {  /* 表达式已经有寄存器了？ */
    if (!hasjumps(e))  /* 没有跳转？ */
      return e->u.info;  /* 结果已经在寄存器中 */
    if (e->u.info >= luaY_nvarstack(fs)) {  /* 寄存器不是本地的？ */
      exp2reg(fs, e, e->u.info);  // 将最终结果放入寄存器中
      return e->u.info;
    }
    /* 如果 else 表达式有跳转，并且不能更改其寄存器来保存跳转值，因为它是一个局部变量。
       继续执行默认情况。 */
  }
  luaK_exp2nextreg(fs, e);  /* 默认情况：使用下一个可用的寄存器 */
  return e->u.info;
/*
** 确保最终表达式结果要么在寄存器中，要么在 upvalue 中
*/
void luaK_exp2anyregup (FuncState *fs, expdesc *e) {
  if (e->k != VUPVAL || hasjumps(e))
    luaK_exp2anyreg(fs, e);
}


/*
** 确保最终表达式结果要么在寄存器中，要么是一个常量
*/
void luaK_exp2val (FuncState *fs, expdesc *e) {
  if (hasjumps(e))
    luaK_exp2anyreg(fs, e);
  else
    luaK_dischargevars(fs, e);
}


/*
** 尝试将 'e' 变成一个带有 R/K 索引范围内的 K 表达式。如果成功则返回 true。
*/
static int luaK_exp2K (FuncState *fs, expdesc *e) {
  if (!hasjumps(e)) {
    int info;
    switch (e->k) {  /* 将常量移动到 'k' 中 */
      case VTRUE: info = boolT(fs); break;
      case VFALSE: info = boolF(fs); break;
      case VNIL: info = nilK(fs); break;
      case VKINT: info = luaK_intK(fs, e->u.ival); break;
      case VKFLT: info = luaK_numberK(fs, e->u.nval); break;
      case VKSTR: info = stringK(fs, e->u.strval); break;
      case VK: info = e->u.info; break;
      default: return 0;  /* 不是常量 */
    }
    if (info <= MAXINDEXRK) {  /* 常量是否适合 'argC' 中？ */
      e->k = VK;  /* 使表达式成为 'K' 表达式 */
      e->u.info = info;
      return 1;
    }
  }
  /* 否则，表达式不适合；保持不变 */
  return 0;
}


/*
** 确保最终表达式结果在有效的 R/K 索引中
** （即，要么在寄存器中，要么在 'k' 中并且具有 R/K 索引范围内的索引）。
** 如果表达式是 K，则返回 1。
*/
int luaK_exp2RK (FuncState *fs, expdesc *e) {
  if (luaK_exp2K(fs, e))
    return 1;
  else {  /* 不是正确范围内的常量：将其放入寄存器中 */
    luaK_exp2anyreg(fs, e);
    return 0;
  }
}


static void codeABRK (FuncState *fs, OpCode o, int a, int b,
                      expdesc *ec) {
  int k = luaK_exp2RK(fs, ec);
  luaK_codeABCk(fs, o, a, b, ec->u.info, k);
}


/*
** 生成代码，将表达式 'ex' 的结果存储到变量 'var' 中
*/
void luaK_storevar (FuncState *fs, expdesc *var, expdesc *ex) {
  switch (var->k) {
    case VLOCAL: {
      // 释放表达式占用的寄存器
      freeexp(fs, ex);
      // 将表达式计算结果移动到正确的位置
      exp2reg(fs, ex, var->u.var.ridx);  /* compute 'ex' into proper place */
      return;
    }
    case VUPVAL: {
      // 将表达式计算结果移动到任意寄存器
      int e = luaK_exp2anyreg(fs, ex);
      // 生成设置 UPVAL 指令
      luaK_codeABC(fs, OP_SETUPVAL, e, var->u.info, 0);
      break;
    }
    case VINDEXUP: {
      // 生成设置 UPVAL 表元素指令
      codeABRK(fs, OP_SETTABUP, var->u.ind.t, var->u.ind.idx, ex);
      break;
    }
    case VINDEXI: {
      // 生成设置整数表元素指令
      codeABRK(fs, OP_SETI, var->u.ind.t, var->u.ind.idx, ex);
      break;
    }
    case VINDEXSTR: {
      // 生成设置字符串表元素指令
      codeABRK(fs, OP_SETFIELD, var->u.ind.t, var->u.ind.idx, ex);
      break;
    }
    case VINDEXED: {
      // 生成设置表元素指令
      codeABRK(fs, OP_SETTABLE, var->u.ind.t, var->u.ind.idx, ex);
      break;
    }
    default: lua_assert(0);  /* invalid var kind to store */
  }
  // 释放表达式占用的寄存器
  freeexp(fs, ex);
}


/*
** Emit SELF instruction (convert expression 'e' into 'e:key(e,').
*/
void luaK_self (FuncState *fs, expdesc *e, expdesc *key) {
  int ereg;
  // 将表达式计算结果移动到任意寄存器
  luaK_exp2anyreg(fs, e);
  ereg = e->u.info;  /* register where 'e' was placed */
  // 释放表达式占用的寄存器
  freeexp(fs, e);
  // 设置基础寄存器为 op_self
  e->u.info = fs->freereg;  /* base register for op_self */
  // self 表达式有一个固定的寄存器
  e->k = VNONRELOC;  /* self expression has a fixed register */
  // 为 op_self 产生函数和 'self'
  luaK_reserveregs(fs, 2);  /* function and 'self' produced by op_self */
  // 生成 SELF 指令
  codeABRK(fs, OP_SELF, e->u.info, ereg, key);
  // 释放表达式占用的寄存器
  freeexp(fs, key);
}


/*
** Negate condition 'e' (where 'e' is a comparison).
*/
static void negatecondition (FuncState *fs, expdesc *e) {
  // 获取跳转指令的控制指令
  Instruction *pc = getjumpcontrol(fs, e->u.info);
  // 断言跳转指令是测试模式，并且不是 OP_TESTSET 和 OP_TEST
  lua_assert(testTMode(GET_OPCODE(*pc)) && GET_OPCODE(*pc) != OP_TESTSET &&
                                           GET_OPCODE(*pc) != OP_TEST);
  // 取反条件
  SETARG_k(*pc, (GETARG_k(*pc) ^ 1));
}


/*
** Emit instruction to jump if 'e' is 'cond' (that is, if 'cond'
** is true, code will jump if 'e' is true.) Return jump position.
** Optimize when 'e' is 'not' something, inverting the condition
** and removing the 'not'.
*/
static int jumponcond (FuncState *fs, expdesc *e, int cond) {
  // 如果表达式类型为 VRELOC
  if (e->k == VRELOC) {
    // 获取表达式对应的指令
    Instruction ie = getinstruction(fs, e);
    // 如果指令为 OP_NOT
    if (GET_OPCODE(ie) == OP_NOT) {
      // 移除之前的 OP_NOT 操作
      removelastinstruction(fs);
      // 返回条件跳转的指令
      return condjump(fs, OP_TEST, GETARG_B(ie), 0, 0, !cond);
    }
    /* else go through */
  }
  // 将表达式加载到寄存器中
  discharge2anyreg(fs, e);
  // 释放表达式的寄存器
  freeexp(fs, e);
  // 返回条件跳转的指令
  return condjump(fs, OP_TESTSET, NO_REG, e->u.info, 0, cond);
}


/*
** Emit code to go through if 'e' is true, jump otherwise.
*/
void luaK_goiftrue (FuncState *fs, expdesc *e) {
  int pc;  /* pc of new jump */
  // 释放表达式的变量
  luaK_dischargevars(fs, e);
  switch (e->k) {
    case VJMP: {  /* condition? */
      // 反转条件，跳转到假的位置
      negatecondition(fs, e);  /* jump when it is false */
      // 保存跳转位置
      pc = e->u.info;  /* save jump position */
      break;
    }
    case VK: case VKFLT: case VKINT: case VKSTR: case VTRUE: {
      pc = NO_JUMP;  /* always true; do nothing */
      break;
    }
    default: {
      // 跳转到假的位置
      pc = jumponcond(fs, e, 0);  /* jump when false */
      break;
    }
  }
  // 将新的跳转位置插入到假列表中
  luaK_concat(fs, &e->f, pc);  /* insert new jump in false list */
  // 将真列表跳转到当前位置
  luaK_patchtohere(fs, e->t);  /* true list jumps to here (to go through) */
  e->t = NO_JUMP;
}


/*
** Emit code to go through if 'e' is false, jump otherwise.
*/
void luaK_goiffalse (FuncState *fs, expdesc *e) {
  int pc;  /* pc of new jump */
  // 释放表达式的变量
  luaK_dischargevars(fs, e);
  switch (e->k) {
    case VJMP: {
      pc = e->u.info;  /* already jump if true */
      break;
    }
    case VNIL: case VFALSE: {
      pc = NO_JUMP;  /* always false; do nothing */
      break;
    }
    default: {
      // 跳转到真的位置
      pc = jumponcond(fs, e, 1);  /* jump if true */
      break;
    }
  }
  // 将新的跳转位置插入到真列表中
  luaK_concat(fs, &e->t, pc);  /* insert new jump in 't' list */
  // 将假列表跳转到当前位置
  luaK_patchtohere(fs, e->f);  /* false list jumps to here (to go through) */
  e->f = NO_JUMP;
}


/*
** Code 'not e', doing constant folding.
*/
static void codenot (FuncState *fs, expdesc *e) {
  switch (e->k) {
    # 如果表达式的值为 nil 或 false，则将其改为 true，因为 true == not nil == not false
    case VNIL: case VFALSE: {
      e->k = VTRUE;  /* true == not nil == not false */
      break;
    }
    # 如果表达式的值为常量或 true，则将其改为 false，因为 false == not "x" == not 0.5 == not 1 == not true
    case VK: case VKFLT: case VKINT: case VKSTR: case VTRUE: {
      e->k = VFALSE;  /* false == not "x" == not 0.5 == not 1 == not true */
      break;
    }
    # 如果表达式的类型为跳转指令，则对条件进行取反
    case VJMP: {
      negatecondition(fs, e);
      break;
    }
    # 如果表达式的类型为可重定位或不可重定位，则将其转换为任意寄存器，释放表达式，生成取反指令，然后将类型改为可重定位
    case VRELOC:
    case VNONRELOC: {
      discharge2anyreg(fs, e);
      freeexp(fs, e);
      e->u.info = luaK_codeABC(fs, OP_NOT, 0, e->u.info, 0);
      e->k = VRELOC;
      break;
    }
    # 默认情况下，断言不会发生
    default: lua_assert(0);  /* cannot happen */
  }
  # 交换 true 和 false 列表
  { int temp = e->f; e->f = e->t; e->t = temp; }
  # 移除取反后无用的值
  removevalues(fs, e->f);  /* values are useless when negated */
  removevalues(fs, e->t);
/*
** 检查表达式 'e' 是否为一个小的字面字符串
*/
static int isKstr (FuncState *fs, expdesc *e) {
  return (e->k == VK && !hasjumps(e) && e->u.info <= MAXARG_B &&
          ttisshrstring(&fs->f->k[e->u.info]));
}

/*
** 检查表达式 'e' 是否为字面整数。
*/
int luaK_isKint (expdesc *e) {
  return (e->k == VKINT && !hasjumps(e));
}


/*
** 检查表达式 'e' 是否为字面整数，并且在适合放入寄存器 C 的范围内
*/
static int isCint (expdesc *e) {
  return luaK_isKint(e) && (l_castS2U(e->u.ival) <= l_castS2U(MAXARG_C));
}


/*
** 检查表达式 'e' 是否为字面整数，并且在适合放入寄存器 sC 的范围内
*/
static int isSCint (expdesc *e) {
  return luaK_isKint(e) && fitsC(e->u.ival);
}


/*
** 检查表达式 'e' 是否为字面整数或浮点数，并且在适合放入寄存器（sB 或 sC）的范围内
*/
static int isSCnumber (expdesc *e, int *pi, int *isfloat) {
  lua_Integer i;
  if (e->k == VKINT)
    i = e->u.ival;
  else if (e->k == VKFLT && luaV_flttointeger(e->u.nval, &i, F2Ieq))
    *isfloat = 1;
  else
    return 0;  /* 不是一个数字 */
  if (!hasjumps(e) && fitsC(i)) {
    *pi = int2sC(cast_int(i));
    return 1;
  }
  else
    return 0;
}


/*
** 创建表达式 't[k]'。't' 必须已经在寄存器或上值中有其最终结果。
** 上值只能由字面字符串索引。
** 键可以是常量表中的字面字符串，也可以是寄存器中的任意值。
*/
void luaK_indexed (FuncState *fs, expdesc *t, expdesc *k) {
  if (k->k == VKSTR)
    str2K(fs, k);
  lua_assert(!hasjumps(t) &&
             (t->k == VLOCAL || t->k == VNONRELOC || t->k == VUPVAL));
  if (t->k == VUPVAL && !isKstr(fs, k))  /* 上值被非 'Kstr' 索引？ */
    luaK_exp2anyreg(fs, t);  /* 将其放入寄存器 */
  if (t->k == VUPVAL) {
    t->u.ind.t = t->u.info;  /* 上值索引 */
    t->u.ind.idx = k->u.info;  /* 字面字符串 */
    t->k = VINDEXUP;
  }
  else {
    /* 将表的索引注册到表中 */
    t->u.ind.t = (t->k == VLOCAL) ? t->u.var.ridx: t->u.info;
    // 如果常量是字符串
    if (isKstr(fs, k)) {
      // 将常量的索引注册到表中
      t->u.ind.idx = k->u.info;  /* literal string */
      t->k = VINDEXSTR;
    }
    // 如果常量是整数
    else if (isCint(k)) {
      // 将常量的整数值注册到表中
      t->u.ind.idx = cast_int(k->u.ival);  /* int. constant in proper range */
      t->k = VINDEXI;
    }
    // 如果常量是其他类型
    else {
      // 将常量的值注册到表中
      t->u.ind.idx = luaK_exp2anyreg(fs, k);  /* register */
      t->k = VINDEXED;
    }
  }
*/


/*
** Return false if folding can raise an error.
** Bitwise operations need operands convertible to integers; division
** operations cannot have 0 as divisor.
*/
static int validop (int op, TValue *v1, TValue *v2) {
  switch (op) {
    case LUA_OPBAND: case LUA_OPBOR: case LUA_OPBXOR:
    case LUA_OPSHL: case LUA_OPSHR: case LUA_OPBNOT: {  /* conversion errors */
      lua_Integer i;
      return (luaV_tointegerns(v1, &i, LUA_FLOORN2I) &&
              luaV_tointegerns(v2, &i, LUA_FLOORN2I));
    }
    case LUA_OPDIV: case LUA_OPIDIV: case LUA_OPMOD:  /* division by 0 */
      return (nvalue(v2) != 0);
    default: return 1;  /* everything else is valid */
  }
}


/*
** Try to "constant-fold" an operation; return 1 iff successful.
** (In this case, 'e1' has the final result.)
*/
static int constfolding (FuncState *fs, int op, expdesc *e1,
                                        const expdesc *e2) {
  TValue v1, v2, res;
  if (!tonumeral(e1, &v1) || !tonumeral(e2, &v2) || !validop(op, &v1, &v2))
    return 0;  /* non-numeric operands or not safe to fold */
  luaO_rawarith(fs->ls->L, op, &v1, &v2, &res);  /* does operation */
  if (ttisinteger(&res)) {
    e1->k = VKINT;
    e1->u.ival = ivalue(&res);
  }
  else {  /* folds neither NaN nor 0.0 (to avoid problems with -0.0) */
    lua_Number n = fltvalue(&res);
    if (luai_numisnan(n) || n == 0)
      return 0;
    e1->k = VKFLT;
    e1->u.nval = n;
  }
  return 1;
}


/*
** Emit code for unary expressions that "produce values"
** (everything but 'not').
** Expression to produce final result will be encoded in 'e'.
*/
static void codeunexpval (FuncState *fs, OpCode op, expdesc *e, int line) {
  int r = luaK_exp2anyreg(fs, e);  /* opcodes operate only on registers */
  freeexp(fs, e);
  e->u.info = luaK_codeABC(fs, op, 0, r, 0);  /* generate opcode */
  e->k = VRELOC;  /* all those operations are relocatable */
  luaK_fixline(fs, line);
}


/*
** Emit code for binary expressions that "produce values"
*/
/*
** 完成二进制表达式的值计算，不包括逻辑运算符 'and'/'or' 和比较运算符
** 表达式的最终结果将被编码在 'e1' 中。
*/
static void finishbinexpval (FuncState *fs, expdesc *e1, expdesc *e2,
                             OpCode op, int v2, int flip, int line,
                             OpCode mmop, TMS event) {
  int v1 = luaK_exp2anyreg(fs, e1);  // 将表达式 e1 转换为寄存器索引
  int pc = luaK_codeABCk(fs, op, 0, v1, v2, 0);  // 生成二进制操作的指令，并返回指令索引
  freeexps(fs, e1, e2);  // 释放表达式 e1 和 e2 占用的寄存器
  e1->u.info = pc;  // 将表达式 e1 的寄存器索引设置为指令索引
  e1->k = VRELOC;  /* 所有这些操作都是可重定位的 */
  luaK_fixline(fs, line);  // 设置当前行号
  luaK_codeABCk(fs, mmop, v1, v2, event, flip);  // 调用元方法
  luaK_fixline(fs, line);  // 设置当前行号
}


/*
** 为“产生值”的二进制表达式生成代码，使用两个寄存器。
*/
static void codebinexpval (FuncState *fs, OpCode op,
                           expdesc *e1, expdesc *e2, int line) {
  int v2 = luaK_exp2anyreg(fs, e2);  // 将表达式 e2 转换为寄存器索引
  lua_assert(OP_ADD <= op && op <= OP_SHR);  // 断言，确保 op 在指定范围内
  finishbinexpval(fs, e1, e2, op, v2, 0, line, OP_MMBIN,
                  cast(TMS, (op - OP_ADD) + TM_ADD));  // 完成二进制表达式的值计算
}


/*
** 用立即数操作数生成二进制运算符的代码。
*/
static void codebini (FuncState *fs, OpCode op,
                       expdesc *e1, expdesc *e2, int flip, int line,
                       TMS event) {
  int v2 = int2sC(cast_int(e2->u.ival));  // 将整数转换为 sC 值（带符号的 C 值）
  lua_assert(e2->k == VKINT);  // 断言，确保表达式 e2 是整数常量
  finishbinexpval(fs, e1, e2, op, v2, flip, line, OP_MMBINI, event);  // 完成二进制表达式的值计算
}


/* 尝试对二元运算符进行编码，对第二个操作数取负。
** 对于元方法，第二个操作数必须保持其原始值。
*/
static int finishbinexpneg (FuncState *fs, expdesc *e1, expdesc *e2,
                             OpCode op, int line, TMS event) {
  if (!luaK_isKint(e2))
    return 0;  // 不是整数常量
  else {
    lua_Integer i2 = e2->u.ival;
    if (!(fitsC(i2) && fitsC(-i2)))
      return 0;  // 不在适当的范围内
    else {  /* operating a small integer constant */
      // 如果操作的是一个小整数常量，则执行以下操作
      int v2 = cast_int(i2);
      // 将 i2 转换为整数类型并赋值给 v2
      finishbinexpval(fs, e1, e2, op, int2sC(-v2), 0, line, OP_MMBINI, event);
      // 调用 finishbinexpval 函数，传入参数 fs, e1, e2, op, int2sC(-v2), 0, line, OP_MMBINI, event
      /* correct metamethod argument */
      // 修正元方法参数
      SETARG_B(fs->f->code[fs->pc - 1], int2sC(v2));
      // 设置指令参数 B 为整数 v2
      return 1;  /* successfully coded */
      // 返回 1，表示成功编码
    }
  }
# 交换两个表达式的值
static void swapexps (expdesc *e1, expdesc *e2) {
  expdesc temp = *e1; *e1 = *e2; *e2 = temp;  /* 交换 'e1' 和 'e2' 的值 */
}


/*
** 生成算术运算符代码（'+'，'-'等）。如果第二个操作数是适当范围内的常量，则使用带有 K 操作数的变体操作码。
*/
static void codearith (FuncState *fs, BinOpr opr,
                       expdesc *e1, expdesc *e2, int flip, int line) {
  TMS event = cast(TMS, opr + TM_ADD);
  if (tonumeral(e2, NULL) && luaK_exp2K(fs, e2)) {  /* K 操作数？ */
    int v2 = e2->u.info;  /* K 索引 */
    OpCode op = cast(OpCode, opr + OP_ADDK);
    finishbinexpval(fs, e1, e2, op, v2, flip, line, OP_MMBINK, event);
  }
  else {  /* 'e2' 既不是立即数也不是 K 操作数 */
    OpCode op = cast(OpCode, opr + OP_ADD);
    if (flip)
      swapexps(e1, e2);  /* 恢复原始顺序 */
    codebinexpval(fs, op, e1, e2, line);  /* 使用标准操作符 */
  }
}


/*
** 生成可交换的运算符代码（'+'，'*'）。如果第一个操作数是数值常量，则更改操作数的顺序以尝试使用立即数或 K 操作符。
*/
static void codecommutative (FuncState *fs, BinOpr op,
                             expdesc *e1, expdesc *e2, int line) {
  int flip = 0;
  if (tonumeral(e1, NULL)) {  /* 第一个操作数是数值常量？ */
    swapexps(e1, e2);  /* 更改顺序 */
    flip = 1;
  }
  if (op == OPR_ADD && isSCint(e2))  /* 立即数操作数？ */
    codebini(fs, cast(OpCode, OP_ADDI), e1, e2, flip, line, TM_ADD);
  else
    codearith(fs, op, e1, e2, flip, line);
}


/*
** 生成位运算代码；它们都是可交换的，因此该函数尝试将整数常量放在第二个操作数（K 操作数）。
*/
static void codebitwise (FuncState *fs, BinOpr opr,
                         expdesc *e1, expdesc *e2, int line) {
  int flip = 0;
  int v2;
  OpCode op;
  if (e1->k == VKINT && luaK_exp2RK(fs, e1)) {
    swapexps(e1, e2);  /* 'e2' 将成为常量操作数 */
    flip = 1;  # 设置 flip 变量的值为 1
  }  # 结束 if 语句块
  else if (!(e2->k == VKINT && luaK_exp2RK(fs, e2))) {  # 如果 e2 不是整数常量
    op = cast(OpCode, opr + OP_ADD);  # 设置 op 变量的值为 opr + OP_ADD
    codebinexpval(fs, op, e1, e2, line);  # 调用 codebinexpval 函数，传入参数 fs, op, e1, e2, line
    return;  # 返回
  }  # 结束 else if 语句块
  v2 = e2->u.info;  # 将 e2->u.info 的值赋给 v2
  op = cast(OpCode, opr + OP_ADDK);  # 设置 op 变量的值为 opr + OP_ADDK
  lua_assert(ttisinteger(&fs->f->k[v2]));  # 断言 fs->f->k[v2] 是整数类型
  finishbinexpval(fs, e1, e2, op, v2, flip, line, OP_MMBINK,  # 调用 finishbinexpval 函数，传入参数 fs, e1, e2, op, v2, flip, line, OP_MMBINK, cast(TMS, opr + TM_ADD)
                  cast(TMS, opr + TM_ADD));  # 传入参数 cast(TMS, opr + TM_ADD)
/*
** Emit code for order comparisons. When using an immediate operand,
** 'isfloat' tells whether the original value was a float.
*/
static void codeorder (FuncState *fs, OpCode op, expdesc *e1, expdesc *e2) {
  int r1, r2;
  int im;
  int isfloat = 0;
  if (isSCnumber(e2, &im, &isfloat)) {
    /* 如果 e2 是立即数，使用立即数操作数 */
    r1 = luaK_exp2anyreg(fs, e1);
    r2 = im;
    op = cast(OpCode, (op - OP_LT) + OP_LTI);
  }
  else if (isSCnumber(e1, &im, &isfloat)) {
    /* 将 (A < B) 转换为 (B > A)，将 (A <= B) 转换为 (B >= A) */
    r1 = luaK_exp2anyreg(fs, e2);
    r2 = im;
    op = (op == OP_LT) ? OP_GTI : OP_GEI;
  }
  else {  /* 常规情况，比较两个寄存器的值 */
    r1 = luaK_exp2anyreg(fs, e1);
    r2 = luaK_exp2anyreg(fs, e2);
  }
  freeexps(fs, e1, e2);
  e1->u.info = condjump(fs, op, r1, r2, isfloat, 1);
  e1->k = VJMP;
}


/*
** Emit code for equality comparisons ('==', '~=').
** 'e1' was already put as RK by 'luaK_infix'.
*/
static void codeeq (FuncState *fs, BinOpr opr, expdesc *e1, expdesc *e2) {
  int r1, r2;
  int im;
  int isfloat = 0;  /* 这里不需要，但为了对称性而保留 */
  OpCode op;
  if (e1->k != VNONRELOC) {
    lua_assert(e1->k == VK || e1->k == VKINT || e1->k == VKFLT);
    swapexps(e1, e2);
  }
  r1 = luaK_exp2anyreg(fs, e1);  /* 第一个表达式必须在寄存器中 */
  if (isSCnumber(e2, &im, &isfloat)) {
    op = OP_EQI;
    r2 = im;  /* 立即操作数 */
  }
  else if (luaK_exp2RK(fs, e2)) {  /* 第一个表达式是常量？ */
    op = OP_EQK;
    r2 = e2->u.info;  /* 常量索引 */
  }
  else {
    op = OP_EQ;  /* 将比较两个寄存器的值 */
    r2 = luaK_exp2anyreg(fs, e2);
  }
  freeexps(fs, e1, e2);
  e1->u.info = condjump(fs, op, r1, r2, isfloat, (opr == OPR_EQ));
  e1->k = VJMP;
}


/*
** Apply prefix operation 'op' to expression 'e'.
*/
void luaK_prefix (FuncState *fs, UnOpr op, expdesc *e, int line) {
  static const expdesc ef = {VKINT, {0}, NO_JUMP, NO_JUMP};
  luaK_dischargevars(fs, e);
  switch (op) {
    # 如果操作符是减号或按位取反，则使用'ef'作为假的第二个操作数
    if (constfolding(fs, op + LUA_OPUNM, e, &ef))
        break;
    # 如果操作符是取长度，则调用codeunexpval函数，生成对应的指令
    # 否则，继续执行下面的操作
    case OPR_LEN:
        codeunexpval(fs, cast(OpCode, op + OP_UNM), e, line);
        break;
    # 如果操作符是非，则调用codenot函数，生成对应的指令
    case OPR_NOT: 
        codenot(fs, e); 
        break;
    # 默认情况下，断言为假
    default: 
        lua_assert(0);
/*
** Process 1st operand 'v' of binary operation 'op' before reading
** 2nd operand.
*/
void luaK_infix (FuncState *fs, BinOpr op, expdesc *v) {
  // 处理二元操作符 'op' 的第一个操作数 'v'，确保其为变量
  luaK_dischargevars(fs, v);
  switch (op) {
    case OPR_AND: {
      luaK_goiftrue(fs, v);  /* 只有当 'v' 为真时才继续执行 */
      break;
    }
    case OPR_OR: {
      luaK_goiffalse(fs, v);  /* 只有当 'v' 为假时才继续执行 */
      break;
    }
    case OPR_CONCAT: {
      luaK_exp2nextreg(fs, v);  /* 操作数必须在栈上 */
      break;
    }
    case OPR_ADD: case OPR_SUB:
    case OPR_MUL: case OPR_DIV: case OPR_IDIV:
    case OPR_MOD: case OPR_POW:
    case OPR_BAND: case OPR_BOR: case OPR_BXOR:
    case OPR_SHL: case OPR_SHR: {
      if (!tonumeral(v, NULL))
        luaK_exp2anyreg(fs, v);
      /* 否则保持数字，可能会与第二个操作数合并 */
      break;
    }
    case OPR_EQ: case OPR_NE: {
      if (!tonumeral(v, NULL))
        luaK_exp2RK(fs, v);
      /* 否则保持数字，可能会成为立即数操作数 */
      break;
    }
    case OPR_LT: case OPR_LE:
    case OPR_GT: case OPR_GE: {
      int dummy, dummy2;
      if (!isSCnumber(v, &dummy, &dummy2))
        luaK_exp2anyreg(fs, v);
      /* 否则保持数字，可能会成为立即数操作数 */
      break;
    }
    default: lua_assert(0);
  }
}

/*
** Create code for '(e1 .. e2)'.
** For '(e1 .. e2.1 .. e2.2)' (which is '(e1 .. (e2.1 .. e2.2))',
** because concatenation is right associative), merge both CONCATs.
*/
static void codeconcat (FuncState *fs, expdesc *e1, expdesc *e2, int line) {
  Instruction *ie2 = previousinstruction(fs);
  if (GET_OPCODE(*ie2) == OP_CONCAT) {  /* 'e2' 是否为一个连接操作？ */
    int n = GETARG_B(*ie2);  /* 'e2' 中连接的元素数量 */
    lua_assert(e1->u.info + 1 == GETARG_A(*ie2));
    freeexp(fs, e2);
    SETARG_A(*ie2, e1->u.info);  /* 修正第一个元素 ('e1') */
    SETARG_B(*ie2, n + 1);  /* 设置指令参数B为n+1，将会连接一个更多的元素 */
  }
  else {  /* 'e2' 不是一个连接操作 */
    luaK_codeABC(fs, OP_CONCAT, e1->u.info, 2, 0);  /* 生成新的连接操作码 */
    freeexp(fs, e2);  /* 释放表达式e2 */
    luaK_fixline(fs, line);  /* 调整代码行号 */
  }
/*
** Finalize code for binary operation, after reading 2nd operand.
*/
void luaK_posfix (FuncState *fs, BinOpr opr,
                  expdesc *e1, expdesc *e2, int line) {
  // 释放第二个操作数的变量
  luaK_dischargevars(fs, e2);
  // 如果可以通过折叠来完成操作，则返回
  if (foldbinop(opr) && constfolding(fs, opr + LUA_OPADD, e1, e2))
    return;  /* done by folding */
  // 根据操作符进行不同的处理
  switch (opr) {
    case OPR_AND: {
      // 断言 e1 的跳转列表为空，由 'luaK_infix' 关闭
      lua_assert(e1->t == NO_JUMP);  
      // 合并 e2 的假跳转列表到 e1 的假跳转列表
      luaK_concat(fs, &e2->f, e1->f);
      // 将 e2 的信息复制给 e1
      *e1 = *e2;
      break;
    }
    case OPR_OR: {
      // 断言 e1 的假跳转列表为空，由 'luaK_infix' 关闭
      lua_assert(e1->f == NO_JUMP);  
      // 合并 e2 的真跳转列表到 e1 的真跳转列表
      luaK_concat(fs, &e2->t, e1->t);
      // 将 e2 的信息复制给 e1
      *e1 = *e2;
      break;
    }
    case OPR_CONCAT: {  /* e1 .. e2 */
      // 将 e2 的值存储到下一个寄存器
      luaK_exp2nextreg(fs, e2);
      // 生成连接操作的指令
      codeconcat(fs, e1, e2, line);
      break;
    }
    case OPR_ADD: case OPR_MUL: {
      // 生成可交换的操作指令
      codecommutative(fs, opr, e1, e2, line);
      break;
    }
    case OPR_SUB: {
      // 如果可以完成负数表达式的操作，则返回
      if (finishbinexpneg(fs, e1, e2, OP_ADDI, line, TM_SUB))
        break; /* coded as (r1 + -I) */
      /* ELSE */
    }  /* FALLTHROUGH */
    case OPR_DIV: case OPR_IDIV: case OPR_MOD: case OPR_POW: {
      // 生成算术操作的指令
      codearith(fs, opr, e1, e2, 0, line);
      break;
    }
    case OPR_BAND: case OPR_BOR: case OPR_BXOR: {
      // 生成位操作的指令
      codebitwise(fs, opr, e1, e2, line);
      break;
    }
    case OPR_SHL: {
      // 如果 e1 是常量整数，则交换 e1 和 e2，并生成左移指令
      if (isSCint(e1)) {
        swapexps(e1, e2);
        codebini(fs, OP_SHLI, e1, e2, 1, line, TM_SHL);  /* I << r2 */
      }
      // 如果可以完成负数表达式的操作，则返回
      else if (finishbinexpneg(fs, e1, e2, OP_SHRI, line, TM_SHL)) {
        /* coded as (r1 >> -I) */;
      }
      else  /* regular case (two registers) */
       // 生成左移操作的指令
       codebinexpval(fs, OP_SHL, e1, e2, line);
      break;
    }
    case OPR_SHR: {
      // 如果 e2 是常量整数，则生成右移指令
      if (isSCint(e2))
        codebini(fs, OP_SHRI, e1, e2, 0, line, TM_SHR);  /* r1 >> I */
      else  /* regular case (two registers) */
        // 生成右移操作的指令
        codebinexpval(fs, OP_SHR, e1, e2, line);
      break;
    }
    case OPR_EQ: case OPR_NE: {
      // 生成相等或不相等比较的指令
      codeeq(fs, opr, e1, e2);
      break;
    }
    # 如果操作符是小于或小于等于，则将操作符转换为对应的等于或大于等于
    case OPR_LT: case OPR_LE: {
      # 根据操作符计算出对应的等于或大于等于的操作码
      OpCode op = cast(OpCode, (opr - OPR_EQ) + OP_EQ);
      # 生成比较操作的指令
      codeorder(fs, op, e1, e2);
      # 跳出当前 case
      break;
    }
    # 如果操作符是大于或大于等于
    case OPR_GT: case OPR_GE: {
      # 交换表达式的顺序，将大于或大于等于转换为小于或小于等于
      /* '(a > b)' <=> '(b < a)';  '(a >= b)' <=> '(b <= a)' */
      OpCode op = cast(OpCode, (opr - OPR_NE) + OP_EQ);
      swapexps(e1, e2);
      # 生成比较操作的指令
      codeorder(fs, op, e1, e2);
      # 跳出当前 case
      break;
    }
    # 默认情况下，断言为假
    default: lua_assert(0);
  }
/*
** Change line information associated with current position, by removing
** previous info and adding it again with new line.
*/
void luaK_fixline (FuncState *fs, int line) {
  // 移除之前的行信息，并使用新行重新添加
  removelastlineinfo(fs);
  savelineinfo(fs, fs->f, line);
}


void luaK_settablesize (FuncState *fs, int pc, int ra, int asize, int hsize) {
  // 获取指令地址
  Instruction *inst = &fs->f->code[pc];
  // 计算哈希表大小
  int rb = (hsize != 0) ? luaO_ceillog2(hsize) + 1 : 0;  /* hash size */
  // 计算数组大小的高位和低位
  int extra = asize / (MAXARG_C + 1);  /* higher bits of array size */
  int rc = asize % (MAXARG_C + 1);  /* lower bits of array size */
  int k = (extra > 0);  /* true iff needs extra argument */
  // 创建新表指令
  *inst = CREATE_ABCk(OP_NEWTABLE, ra, rb, rc, k);
  // 添加额外参数
  *(inst + 1) = CREATE_Ax(OP_EXTRAARG, extra);
}


/*
** Emit a SETLIST instruction.
** 'base' is register that keeps table;
** 'nelems' is #table plus those to be stored now;
** 'tostore' is number of values (in registers 'base + 1',...) to add to
** table (or LUA_MULTRET to add up to stack top).
*/
void luaK_setlist (FuncState *fs, int base, int nelems, int tostore) {
  // 断言条件
  lua_assert(tostore != 0 && tostore <= LFIELDS_PER_FLUSH);
  // 如果tostore为LUA_MULTRET，则置为0
  if (tostore == LUA_MULTRET)
    tostore = 0;
  // 根据nelems的大小选择不同的指令
  if (nelems <= MAXARG_C)
    luaK_codeABC(fs, OP_SETLIST, base, tostore, nelems);
  else {
    int extra = nelems / (MAXARG_C + 1);
    nelems %= (MAXARG_C + 1);
    luaK_codeABCk(fs, OP_SETLIST, base, tostore, nelems, 1);
    codeextraarg(fs, extra);
  }
  fs->freereg = base + 1;  /* free registers with list values */
}


/*
** return the final target of a jump (skipping jumps to jumps)
*/
static int finaltarget (Instruction *code, int i) {
  int count;
  for (count = 0; count < 100; count++) {  /* avoid infinite loops */
    Instruction pc = code[i];
    if (GET_OPCODE(pc) != OP_JMP)
      break;
     else
       i += GETARG_sJ(pc) + 1;
  }
  return i;
}


/*
** Do a final pass over the code of a function, doing small peephole
** optimizations and adjustments.
*/
void luaK_finish (FuncState *fs) {
  // 定义变量 i
  int i;
  // 获取当前函数状态的函数原型
  Proto *p = fs->f;
  // 遍历函数状态的指令数组
  for (i = 0; i < fs->pc; i++) {
    // 获取当前指令的指针
    Instruction *pc = &p->code[i];
    // 断言当前指令的前一条指令是否为相同类型
    lua_assert(i == 0 || isOT(*(pc - 1)) == isIT(*pc));
    // 根据指令类型进行不同的操作
    switch (GET_OPCODE(*pc)) {
      // 处理返回0或返回1的情况
      case OP_RETURN0: case OP_RETURN1: {
        // 如果不需要关闭或者函数原型不是变长参数函数，则跳过
        if (!(fs->needclose || p->is_vararg))
          break;  /* no extra work */
        // 否则使用 OP_RETURN 来进行额外的工作
        SET_OPCODE(*pc, OP_RETURN);
      }  /* FALLTHROUGH */
      // 处理返回或尾调用的情况
      case OP_RETURN: case OP_TAILCALL: {
        // 如果需要关闭，则设置参数 k 为 1，表示需要关闭
        if (fs->needclose)
          SETARG_k(*pc, 1);  /* signal that it needs to close */
        // 如果函数原型是变长参数函数，则设置参数 C 为 numparams + 1，表示是变长参数
        if (p->is_vararg)
          SETARG_C(*pc, p->numparams + 1);  /* signal that it is vararg */
        break;
      }
      // 处理跳转指令的情况
      case OP_JMP: {
        // 获取跳转目标的位置
        int target = finaltarget(p->code, i);
        // 修正跳转指令的目标位置
        fixjump(fs, i, target);
        break;
      }
      // 默认情况下不做任何操作
      default: break;
    }
  }
}
```