# `nmap\liblua\lopcodes.h`

```
/*
** $Id: lopcodes.h $
** Lua虚拟机的操作码
** 请参见lua.h中的版权声明
*/

#ifndef lopcodes_h
#define lopcodes_h

#include "llimits.h"


/*===========================================================================
  我们假设指令是无符号32位整数。
  所有指令的前7位是操作码。
  指令可以有以下格式：

        3 3 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0
        1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
iABC          C(8)     |      B(8)     |k|     A(8)      |   Op(7)     |
iABx                Bx(17)               |     A(8)      |   Op(7)     |
iAsBx              sBx (signed)(17)      |     A(8)      |   Op(7)     |
iAx                           Ax(25)                     |   Op(7)     |
isJ                           sJ(25)                     |   Op(7)     |

  有符号参数用过量K表示：表示值是写入的无符号值减去K，其中K是相应无符号参数的最大值的一半。
===========================================================================*/


enum OpMode {iABC, iABx, iAsBx, iAx, isJ};  /* 基本指令格式 */


/*
** 操作码参数的大小和位置。
*/
#define SIZE_C        8
#define SIZE_B        8
#define SIZE_Bx        (SIZE_C + SIZE_B + 1)
#define SIZE_A        8
#define SIZE_Ax        (SIZE_Bx + SIZE_A)
#define SIZE_sJ        (SIZE_Bx + SIZE_A)

#define SIZE_OP        7

#define POS_OP        0

#define POS_A        (POS_OP + SIZE_OP)
#define POS_k        (POS_A + SIZE_A)
#define POS_B        (POS_k + 1)
#define POS_C        (POS_B + SIZE_B)

#define POS_Bx        POS_k

#define POS_Ax        POS_A

#define POS_sJ        POS_A


/*
** 操作码参数的限制。
** 我们使用（有符号）'int'来处理大多数参数，所以它们必须适合于int。
*/

/* 检查类型'int'是否至少有'b'位（'b' < 32） */
#define L_INTHASBITS(b)        ((UINT_MAX >> ((b) - 1)) >= 1)  // 检查整数是否有足够的位数来表示给定的位数

#if L_INTHASBITS(SIZE_Bx)
#define MAXARG_Bx    ((1<<SIZE_Bx)-1)  // 如果整数有足够的位数，计算最大的 Bx 参数值
#else
#define MAXARG_Bx    MAX_INT  // 如果整数没有足够的位数，将最大的 Bx 参数值设置为最大整数值
#endif

#define OFFSET_sBx    (MAXARG_Bx>>1)         /* 'sBx' is signed */  // 计算 sBx 参数的偏移量

#if L_INTHASBITS(SIZE_Ax)
#define MAXARG_Ax    ((1<<SIZE_Ax)-1)  // 如果整数有足够的位数，计算最大的 Ax 参数值
#else
#define MAXARG_Ax    MAX_INT  // 如果整数没有足够的位数，将最大的 Ax 参数值设置为最大整数值
#endif

#if L_INTHASBITS(SIZE_sJ)
#define MAXARG_sJ    ((1 << SIZE_sJ) - 1)  // 如果整数有足够的位数，计算最大的 sJ 参数值
#else
#define MAXARG_sJ    MAX_INT  // 如果整数没有足够的位数，将最大的 sJ 参数值设置为最大整数值
#endif

#define OFFSET_sJ    (MAXARG_sJ >> 1)  // 计算 sJ 参数的偏移量

#define MAXARG_A    ((1<<SIZE_A)-1)  // 计算最大的 A 参数值
#define MAXARG_B    ((1<<SIZE_B)-1)  // 计算最大的 B 参数值
#define MAXARG_C    ((1<<SIZE_C)-1)  // 计算最大的 C 参数值
#define OFFSET_sC    (MAXARG_C >> 1)  // 计算 sC 参数的偏移量

#define int2sC(i)    ((i) + OFFSET_sC)  // 将整数转换为 sC 参数
#define sC2int(i)    ((i) - OFFSET_sC)  // 将 sC 参数转换为整数

/* creates a mask with 'n' 1 bits at position 'p' */
#define MASK1(n,p)    ((~((~(Instruction)0)<<(n)))<<(p))  // 创建一个在位置 p 处有 n 个 1 的掩码

/* creates a mask with 'n' 0 bits at position 'p' */
#define MASK0(n,p)    (~MASK1(n,p))  // 创建一个在位置 p 处有 n 个 0 的掩码

/*
** the following macros help to manipulate instructions
*/

#define GET_OPCODE(i)    (cast(OpCode, ((i)>>POS_OP) & MASK1(SIZE_OP,0)))  // 获取指令中的操作码
#define SET_OPCODE(i,o)    ((i) = (((i)&MASK0(SIZE_OP,POS_OP)) | \
        ((cast(Instruction, o)<<POS_OP)&MASK1(SIZE_OP,POS_OP))))  // 设置指令中的操作码

#define checkopm(i,m)    (getOpMode(GET_OPCODE(i)) == m)  // 检查指令的操作模式

#define getarg(i,pos,size)    (cast_int(((i)>>(pos)) & MASK1(size,0)))  // 获取指令中的参数
#define setarg(i,v,pos,size)    ((i) = (((i)&MASK0(size,pos)) | \
                ((cast(Instruction, v)<<pos)&MASK1(size,pos))))  // 设置指令中的参数

#define GETARG_A(i)    getarg(i, POS_A, SIZE_A)  // 获取指令中的 A 参数
#define SETARG_A(i,v)    setarg(i, v, POS_A, SIZE_A)  // 设置指令中的 A 参数

#define GETARG_B(i)    check_exp(checkopm(i, iABC), getarg(i, POS_B, SIZE_B))  // 获取指令中的 B 参数
#define GETARG_sB(i)    sC2int(GETARG_B(i))  // 获取指令中的 sB 参数
#define SETARG_B(i,v)    setarg(i, v, POS_B, SIZE_B)  // 设置指令中的 B 参数

#define GETARG_C(i)    check_exp(checkopm(i, iABC), getarg(i, POS_C, SIZE_C))  // 获取指令中的 C 参数
#define GETARG_sC(i)    sC2int(GETARG_C(i))  // 获取指令中的 sC 参数
#define SETARG_C(i,v)    setarg(i, v, POS_C, SIZE_C)  // 设置指令中的 C 参数

#define TESTARG_k(i)    check_exp(checkopm(i, iABC), (cast_int(((i) & (1u << POS_k)))))  // 测试指令中的 k 参数
#define GETARG_k(i)    check_exp(checkopm(i, iABC), getarg(i, POS_k, 1))
# 获取指令 i 中的 k 参数

#define SETARG_k(i,v)    setarg(i, v, POS_k, 1)
# 设置指令 i 中的 k 参数为 v

#define GETARG_Bx(i)    check_exp(checkopm(i, iABx), getarg(i, POS_Bx, SIZE_Bx))
# 获取指令 i 中的 Bx 参数

#define SETARG_Bx(i,v)    setarg(i, v, POS_Bx, SIZE_Bx)
# 设置指令 i 中的 Bx 参数为 v

#define GETARG_Ax(i)    check_exp(checkopm(i, iAx), getarg(i, POS_Ax, SIZE_Ax))
# 获取指令 i 中的 Ax 参数

#define SETARG_Ax(i,v)    setarg(i, v, POS_Ax, SIZE_Ax)
# 设置指令 i 中的 Ax 参数为 v

#define GETARG_sBx(i)  \
    check_exp(checkopm(i, iAsBx), getarg(i, POS_Bx, SIZE_Bx) - OFFSET_sBx)
# 获取指令 i 中的 sBx 参数

#define SETARG_sBx(i,b)    SETARG_Bx((i),cast_uint((b)+OFFSET_sBx))
# 设置指令 i 中的 sBx 参数为 b

#define GETARG_sJ(i)  \
    check_exp(checkopm(i, isJ), getarg(i, POS_sJ, SIZE_sJ) - OFFSET_sJ)
# 获取指令 i 中的 sJ 参数

#define SETARG_sJ(i,j) \
    setarg(i, cast_uint((j)+OFFSET_sJ), POS_sJ, SIZE_sJ)
# 设置指令 i 中的 sJ 参数为 j

#define CREATE_ABCk(o,a,b,c,k)    ((cast(Instruction, o)<<POS_OP) \
            | (cast(Instruction, a)<<POS_A) \
            | (cast(Instruction, b)<<POS_B) \
            | (cast(Instruction, c)<<POS_C) \
            | (cast(Instruction, k)<<POS_k))
# 创建一个包含 ABCk 参数的指令

#define CREATE_ABx(o,a,bc)    ((cast(Instruction, o)<<POS_OP) \
            | (cast(Instruction, a)<<POS_A) \
            | (cast(Instruction, bc)<<POS_Bx))
# 创建一个包含 ABx 参数的指令

#define CREATE_Ax(o,a)        ((cast(Instruction, o)<<POS_OP) \
            | (cast(Instruction, a)<<POS_Ax))
# 创建一个包含 Ax 参数的指令

#define CREATE_sJ(o,j,k)    ((cast(Instruction, o) << POS_OP) \
            | (cast(Instruction, j) << POS_sJ) \
            | (cast(Instruction, k) << POS_k))
# 创建一个包含 sJ 参数的指令

#if !defined(MAXINDEXRK)  /* (for debugging only) */
#define MAXINDEXRK    MAXARG_B
#endif
# 如果 MAXINDEXRK 未定义，则将其定义为 MAXARG_B

/*
** invalid register that fits in 8 bits
*/
#define NO_REG        MAXARG_A
# 适合 8 位的无效寄存器

/*
** R[x] - register
** K[x] - constant (in constant table)
** RK(x) == if k(i) then K[x] else R[x]
*/
# 注释说明了 R[x]、K[x] 和 RK(x) 的含义

/*
** Grep "ORDER OP" if you change these enums. Opcodes marked with a (*)
** has extra descriptions in the notes after the enumeration.
*/
# 如果更改这些枚举值，请搜索 "ORDER OP"。带有 (*) 的操作码在枚举后的注释中有额外的描述

typedef enum {
/*----------------------------------------------------------------------
  name        args    description
# 枚举类型的注释说明
# 将寄存器中的值从寄存器B移动到寄存器A
OP_MOVE,/*    A B    R[A] := R[B]                    */

# 将常量sBx加载到寄存器A中
OP_LOADI,/*    A sBx    R[A] := sBx                    */

# 将常量sBx转换为lua_Number类型加载到寄存器A中
OP_LOADF,/*    A sBx    R[A] := (lua_Number)sBx                */

# 将常量表中索引为Bx的常量加载到寄存器A中
OP_LOADK,/*    A Bx    R[A] := K[Bx]                    */

# 将额外参数的常量加载到寄存器A中
OP_LOADKX,/*    A    R[A] := K[extra arg]                */

# 将false加载到寄存器A中
OP_LOADFALSE,/*    A    R[A] := false                    */

# 将false加载到寄存器A中，并跳过下一条指令
OP_LFALSESKIP,/*A    R[A] := false; pc++    (*)            */

# 将true加载到寄存器A中
OP_LOADTRUE,/*    A    R[A] := true                    */

# 将nil加载到寄存器A及其后续B个寄存器中
OP_LOADNIL,/*    A B    R[A], R[A+1], ..., R[A+B] := nil        */

# 获取UpValue[B]的值加载到寄存器A中
OP_GETUPVAL,/*    A B    R[A] := UpValue[B]                */

# 将寄存器A的值设置为UpValue[B]的值
OP_SETUPVAL,/*    A B    UpValue[B] := R[A]                */

# 获取UpValue[B]中键为K[C]的值加载到寄存器A中
OP_GETTABUP,/*    A B C    R[A] := UpValue[B][K[C]:string]            */

# 获取寄存器B中的表中寄存器C的值加载到寄存器A中
OP_GETTABLE,/*    A B C    R[A] := R[B][R[C]]                */

# 获取寄存器B中的表中常量C的值加载到寄存器A中
OP_GETI,/*    A B C    R[A] := R[B][C]                    */

# 获取寄存器B中的表中键为K[C]的值加载到寄存器A中
OP_GETFIELD,/*    A B C    R[A] := R[B][K[C]:string]            */

# 设置UpValue[A]中键为K[B]的值为寄存器C的值
OP_SETTABUP,/*    A B C    UpValue[A][K[B]:string] := RK(C)        */

# 设置寄存器A中的表中寄存器B的值为寄存器C的值
OP_SETTABLE,/*    A B C    R[A][R[B]] := RK(C)                */

# 设置寄存器A中的表中索引为B的值为寄存器C的值
OP_SETI,/*    A B C    R[A][B] := RK(C)                */

# 设置寄存器A中的表中键为K[B]的值为寄存器C的值
OP_SETFIELD,/*    A B C    R[A][K[B]:string] := RK(C)            */

# 创建一个新的空表加载到寄存器A中
OP_NEWTABLE,/*    A B C k    R[A] := {}                    */

# 将寄存器B的值作为表，寄存器C的值作为键，加载到寄存器A中
OP_SELF,/*    A B C    R[A+1] := R[B]; R[A] := R[B][RK(C):string]    */

# 将寄存器B的值加上常量sC的值，结果加载到寄存器A中
OP_ADDI,/*    A B sC    R[A] := R[B] + sC                */

# 将寄存器B的值加上常量表中索引为C的值，结果加载到寄存器A中
OP_ADDK,/*    A B C    R[A] := R[B] + K[C]:number            */

# 将寄存器B的值减去常量表中索引为C的值，结果加载到寄存器A中
OP_SUBK,/*    A B C    R[A] := R[B] - K[C]:number            */

# 将寄存器B的值乘以常量表中索引为C的值，结果加载到寄存器A中
OP_MULK,/*    A B C    R[A] := R[B] * K[C]:number            */

# 将寄存器B的值取模常量表中索引为C的值，结果加载到寄存器A中
OP_MODK,/*    A B C    R[A] := R[B] % K[C]:number            */

# 将寄存器B的值的C次方加载到寄存器A中
OP_POWK,/*    A B C    R[A] := R[B] ^ K[C]:number            */

# 将寄存器B的值除以常量表中索引为C的值，结果加载到寄存器A中
OP_DIVK,/*    A B C    R[A] := R[B] / K[C]:number            */

# 将寄存器B的值整除以常量表中索引为C的值，结果加载到寄存器A中
OP_IDIVK,/*    A B C    R[A] := R[B] // K[C]:number            */

# 将寄存器B的值按位与常量表中索引为C的值，结果加载到寄存器A中
OP_BANDK,/*    A B C    R[A] := R[B] & K[C]:integer            */
OP_BORK,/*    A B C    R[A] := R[B] | K[C]:integer            */  # 将寄存器 R[B] 和常量 K[C] 进行按位或操作，结果存入 R[A]
OP_BXORK,/*    A B C    R[A] := R[B] ~ K[C]:integer            */  # 将寄存器 R[B] 和常量 K[C] 进行按位异或操作，结果存入 R[A]

OP_SHRI,/*    A B sC    R[A] := R[B] >> sC                */  # 将寄存器 R[B] 右移 sC 位，结果存入 R[A]
OP_SHLI,/*    A B sC    R[A] := sC << R[B]                */  # 将寄存器 R[B] 左移 sC 位，结果存入 R[A]

OP_ADD,/*    A B C    R[A] := R[B] + R[C]                */  # 将寄存器 R[B] 和 R[C] 相加，结果存入 R[A]
OP_SUB,/*    A B C    R[A] := R[B] - R[C]                */  # 将寄存器 R[B] 减去 R[C]，结果存入 R[A]
OP_MUL,/*    A B C    R[A] := R[B] * R[C]                */  # 将寄存器 R[B] 和 R[C] 相乘，结果存入 R[A]
OP_MOD,/*    A B C    R[A] := R[B] % R[C]                */  # 将寄存器 R[B] 对 R[C] 取模，结果存入 R[A]
OP_POW,/*    A B C    R[A] := R[B] ^ R[C]                */  # 将寄存器 R[B] 的 R[C] 次方，结果存入 R[A]
OP_DIV,/*    A B C    R[A] := R[B] / R[C]                */  # 将寄存器 R[B] 除以 R[C]，结果存入 R[A]
OP_IDIV,/*    A B C    R[A] := R[B] // R[C]                */  # 将寄存器 R[B] 整除以 R[C]，结果存入 R[A]

OP_BAND,/*    A B C    R[A] := R[B] & R[C]                */  # 将寄存器 R[B] 和 R[C] 进行按位与操作，结果存入 R[A]
OP_BOR,/*    A B C    R[A] := R[B] | R[C]                */  # 将寄存器 R[B] 和 R[C] 进行按位或操作，结果存入 R[A]
OP_BXOR,/*    A B C    R[A] := R[B] ~ R[C]                */  # 将寄存器 R[B] 和 R[C] 进行按位异或操作，结果存入 R[A]
OP_SHL,/*    A B C    R[A] := R[B] << R[C]                */  # 将寄存器 R[B] 左移 R[C] 位，结果存入 R[A]
OP_SHR,/*    A B C    R[A] := R[B] >> R[C]                */  # 将寄存器 R[B] 右移 R[C] 位，结果存入 R[A]

OP_MMBIN,/*    A B C    call C metamethod over R[A] and R[B]    (*)    */  # 调用 R[A] 和 R[B] 上的元方法 C
OP_MMBINI,/*    A sB C k    call C metamethod over R[A] and sB    */  # 调用 R[A] 和常量 sB 上的元方法 C
OP_MMBINK,/*    A B C k        call C metamethod over R[A] and K[B]    */  # 调用 R[A] 和常量 K[B] 上的元方法 C

OP_UNM,/*    A B    R[A] := -R[B]                    */  # 将寄存器 R[B] 取负，结果存入 R[A]
OP_BNOT,/*    A B    R[A] := ~R[B]                    */  # 将寄存器 R[B] 按位取反，结果存入 R[A]
OP_NOT,/*    A B    R[A] := not R[B]                */  # 将寄存器 R[B] 取非，结果存入 R[A]
OP_LEN,/*    A B    R[A] := #R[B] (length operator)            */  # 获取寄存器 R[B] 的长度，结果存入 R[A]

OP_CONCAT,/*    A B    R[A] := R[A].. ... ..R[A + B - 1]        */  # 将寄存器 R[A] 到 R[A+B-1] 的值连接起来，结果存入 R[A]

OP_CLOSE,/*    A    close all upvalues >= R[A]            */  # 关闭所有大于等于 R[A] 的上值
OP_TBC,/*    A    mark variable A "to be closed"            */  # 标记变量 A 为“待关闭”
OP_JMP,/*    sJ    pc += sJ                    */  # 跳转到当前指令地址加上 sJ 的位置

OP_EQ,/*    A B k    if ((R[A] == R[B]) ~= k) then pc++        */  # 如果 R[A] 等于 R[B] 并且 k 为真，那么跳转到下一条指令
OP_LT,/*    A B k    if ((R[A] <  R[B]) ~= k) then pc++        */  # 如果 R[A] 小于 R[B] 并且 k 为真，那么跳转到下一条指令
OP_LE,/*    A B k    if ((R[A] <= R[B]) ~= k) then pc++        */  # 如果 R[A] 小于等于 R[B] 并且 k 为真，那么跳转到下一条指令

OP_EQK,/*    A B k    if ((R[A] == K[B]) ~= k) then pc++        */  # 如果 R[A] 等于常量 K[B] 并且 k 为真，那么跳转到下一条指令
/* 指令属性的掩码。格式如下：
   位 0-2: 操作模式
   位 3: 指令设置寄存器 A
   位 4: 操作符是一个测试（下一条指令必须是跳转）
   位 5: 指令使用上一个指令设置的 'L->top'（当 B == 0 时）
*/
# 定义了一些宏和函数，用于解析和操作 Lua 字节码指令的操作模式
** bit 6: instruction sets 'L->top' for next instruction (when C == 0)
** bit 7: instruction is an MM instruction (call a metamethod)
*/

# 定义了 Lua 字节码指令的操作模式数组
LUAI_DDEC(const lu_byte luaP_opmodes[NUM_OPCODES];)

# 获取指定指令的操作模式
#define getOpMode(m)    (cast(enum OpMode, luaP_opmodes[m] & 7))
# 检查指定指令是否使用 A 寄存器模式
#define testAMode(m)    (luaP_opmodes[m] & (1 << 3))
# 检查指定指令是否使用 T 寄存器模式
#define testTMode(m)    (luaP_opmodes[m] & (1 << 4))
# 检查指定指令是否使用 IT 寄存器模式
#define testITMode(m)    (luaP_opmodes[m] & (1 << 5))
# 检查指定指令是否使用 OT 寄存器模式
#define testOTMode(m)    (luaP_opmodes[m] & (1 << 6))
# 检查指定指令是否是 MM 指令
#define testMMMode(m)    (luaP_opmodes[m] & (1 << 7))

# 检查指定指令是否是 "out top" 指令
#define isOT(i)  \
    ((testOTMode(GET_OPCODE(i)) && GETARG_C(i) == 0) || \
          GET_OPCODE(i) == OP_TAILCALL)

# 检查指定指令是否是 "in top" 指令
#define isIT(i)        (testITMode(GET_OPCODE(i)) && GETARG_B(i) == 0)

# 通过给定的参数构造操作模式
#define opmode(mm,ot,it,t,a,m)  \
    (((mm) << 7) | ((ot) << 6) | ((it) << 5) | ((t) << 4) | ((a) << 3) | (m)

# 每个 SETLIST 指令累积的列表项数目
#define LFIELDS_PER_FLUSH    50

# 结束条件
#endif
```