# `nmap\liblua\lcode.h`

```cpp
/*
** $Id: lcode.h $
** Code generator for Lua
** See Copyright Notice in lua.h
*/

#ifndef lcode_h
#define lcode_h

#include "llex.h"  // 包含词法分析器的头文件
#include "lobject.h"  // 包含对象定义的头文件
#include "lopcodes.h"  // 包含操作码定义的头文件
#include "lparser.h"  // 包含解析器的头文件


/*
** Marks the end of a patch list. It is an invalid value both as an absolute
** address, and as a list link (would link an element to itself).
*/
#define NO_JUMP (-1)  // 定义一个表示结束的标记


/*
** grep "ORDER OPR" if you change these enums  (ORDER OP)
*/
typedef enum BinOpr {
  /* arithmetic operators */
  OPR_ADD, OPR_SUB, OPR_MUL, OPR_MOD, OPR_POW,  // 算术运算符
  OPR_DIV, OPR_IDIV,
  /* bitwise operators */
  OPR_BAND, OPR_BOR, OPR_BXOR,  // 位运算符
  OPR_SHL, OPR_SHR,
  /* string operator */
  OPR_CONCAT,  // 字符串连接运算符
  /* comparison operators */
  OPR_EQ, OPR_LT, OPR_LE,  // 比较运算符
  OPR_NE, OPR_GT, OPR_GE,
  /* logical operators */
  OPR_AND, OPR_OR,  // 逻辑运算符
  OPR_NOBINOPR
} BinOpr;


/* true if operation is foldable (that is, it is arithmetic or bitwise) */
#define foldbinop(op)    ((op) <= OPR_SHR)  // 判断操作是否可折叠（即，它是算术或位运算）


#define luaK_codeABC(fs,o,a,b,c)    luaK_codeABCk(fs,o,a,b,c,0)  // 生成指令


typedef enum UnOpr { OPR_MINUS, OPR_BNOT, OPR_NOT, OPR_LEN, OPR_NOUNOPR } UnOpr;  // 枚举一元操作符


/* get (pointer to) instruction of given 'expdesc' */
#define getinstruction(fs,e)    ((fs)->f->code[(e)->u.info])  // 获取给定 'expdesc' 的指令


#define luaK_setmultret(fs,e)    luaK_setreturns(fs, e, LUA_MULTRET)  // 设置多返回值


#define luaK_jumpto(fs,t)    luaK_patchlist(fs, luaK_jump(fs), t)  // 跳转到指定位置


LUAI_FUNC int luaK_code (FuncState *fs, Instruction i);  // 生成指令
LUAI_FUNC int luaK_codeABx (FuncState *fs, OpCode o, int A, unsigned int Bx);  // 生成指令
LUAI_FUNC int luaK_codeAsBx (FuncState *fs, OpCode o, int A, int Bx);  // 生成指令
LUAI_FUNC int luaK_codeABCk (FuncState *fs, OpCode o, int A,
                                            int B, int C, int k);  // 生成指令
LUAI_FUNC int luaK_isKint (expdesc *e);  // 判断是否为整数
LUAI_FUNC int luaK_exp2const (FuncState *fs, const expdesc *e, TValue *v);  // 表达式转换为常量
LUAI_FUNC void luaK_fixline (FuncState *fs, int line);  // 修正行号
LUAI_FUNC void luaK_nil (FuncState *fs, int from, int n);  // 将寄存器置空
LUAI_FUNC void luaK_reserveregs (FuncState *fs, int n);  // 预留寄存器
# 检查函数状态中的栈空间是否足够，如果不够则扩展
LUAI_FUNC void luaK_checkstack (FuncState *fs, int n);

# 将整数 n 存储到寄存器 reg 中
LUAI_FUNC void luaK_int (FuncState *fs, int reg, lua_Integer n);

# 释放变量的值
LUAI_FUNC void luaK_dischargevars (FuncState *fs, expdesc *e);

# 将表达式 e 移动到任意寄存器中
LUAI_FUNC int luaK_exp2anyreg (FuncState *fs, expdesc *e);

# 将表达式 e 移动到下一个寄存器中
LUAI_FUNC void luaK_exp2anyregup (FuncState *fs, expdesc *e);

# 将表达式 e 移动到下一个寄存器中
LUAI_FUNC void luaK_exp2nextreg (FuncState *fs, expdesc *e);

# 将表达式 e 移动到值寄存器中
LUAI_FUNC void luaK_exp2val (FuncState *fs, expdesc *e);

# 将表达式 e 移动到常量寄存器中
LUAI_FUNC int luaK_exp2RK (FuncState *fs, expdesc *e);

# 生成 self 表达式
LUAI_FUNC void luaK_self (FuncState *fs, expdesc *e, expdesc *key);

# 生成索引表达式
LUAI_FUNC void luaK_indexed (FuncState *fs, expdesc *t, expdesc *k);

# 如果表达式 e 为真，则跳转到指定位置
LUAI_FUNC void luaK_goiftrue (FuncState *fs, expdesc *e);

# 如果表达式 e 为假，则跳转到指定位置
LUAI_FUNC void luaK_goiffalse (FuncState *fs, expdesc *e);

# 存储表达式 e 到变量 var 中
LUAI_FUNC void luaK_storevar (FuncState *fs, expdesc *var, expdesc *e);

# 设置返回值的数量
LUAI_FUNC void luaK_setreturns (FuncState *fs, expdesc *e, int nresults);

# 设置返回值的数量为 1
LUAI_FUNC void luaK_setoneret (FuncState *fs, expdesc *e);

# 生成一个跳转指令
LUAI_FUNC int luaK_jump (FuncState *fs);

# 生成一个返回指令
LUAI_FUNC void luaK_ret (FuncState *fs, int first, int nret);

# 将列表中的跳转目标修改为指定位置
LUAI_FUNC void luaK_patchlist (FuncState *fs, int list, int target);

# 将列表中的跳转目标修改为当前位置
LUAI_FUNC void luaK_patchtohere (FuncState *fs, int list);

# 连接两个列表
LUAI_FUNC void luaK_concat (FuncState *fs, int *l1, int l2);

# 获取一个新的标签
LUAI_FUNC int luaK_getlabel (FuncState *fs);

# 生成一元操作符表达式
LUAI_FUNC void luaK_prefix (FuncState *fs, UnOpr op, expdesc *v, int line);

# 生成二元操作符表达式
LUAI_FUNC void luaK_infix (FuncState *fs, BinOpr op, expdesc *v);

# 生成后缀操作符表达式
LUAI_FUNC void luaK_posfix (FuncState *fs, BinOpr op, expdesc *v1,
                            expdesc *v2, int line);

# 设置表的大小
LUAI_FUNC void luaK_settablesize (FuncState *fs, int pc,
                                  int ra, int asize, int hsize);

# 设置列表的大小
LUAI_FUNC void luaK_setlist (FuncState *fs, int base, int nelems, int tostore);

# 结束函数的编译
LUAI_FUNC void luaK_finish (FuncState *fs);

# 报告语法错误
LUAI_FUNC l_noret luaK_semerror (LexState *ls, const char *msg);
```