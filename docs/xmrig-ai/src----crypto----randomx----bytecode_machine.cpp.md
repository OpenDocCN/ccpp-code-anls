# `xmrig\src\crypto\randomx\bytecode_machine.cpp`

```
/*
版权声明：
版权所有 (c) 2019, tevador <tevador@gmail.com>
保留所有权利。
在源代码和二进制形式下，无论是否修改，只要满足以下条件，都可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在文档和/或其他提供的材料中，必须复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，
包括但不限于适销性和适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害承担责任，
包括但不限于替代商品或服务的采购、使用、数据或利润损失或业务中断，
无论是合同责任、严格责任还是侵权行为 (包括疏忽或其他) 的任何理论，
即使事先已被告知此类损害的可能性。
*/
#include "crypto/randomx/bytecode_machine.hpp"
#include "crypto/randomx/reciprocal.h"

namespace randomx {

    // 定义 BytecodeMachine 类的静态成员变量 zero，并初始化为 0
    const int_reg_t BytecodeMachine::zero = 0;

    // 定义宏 INSTR_CASE，用于根据指令类型执行相应的指令函数
#define INSTR_CASE(x) case InstructionType::x: \
    exe_ ## x(ibc, pc, scratchpad, config); \
    break;
    # 执行指令的函数，根据指令类型进行不同的操作
    void BytecodeMachine::executeInstruction(RANDOMX_EXE_ARGS) {
        # 根据指令类型进行不同的操作
        switch (ibc.type)
        {
            # 对于指令类型为 IADD_RS 的情况
            INSTR_CASE(IADD_RS)
            # 对于指令类型为 IADD_M 的情况
            INSTR_CASE(IADD_M)
            # 对于指令类型为 ISUB_R 的情况
            INSTR_CASE(ISUB_R)
            # 对于指令类型为 ISUB_M 的情况
            INSTR_CASE(ISUB_M)
            # 对于指令类型为 IMUL_R 的情况
            INSTR_CASE(IMUL_R)
            # 对于指令类型为 IMUL_M 的情况
            INSTR_CASE(IMUL_M)
            # 对于指令类型为 IMULH_R 的情况
            INSTR_CASE(IMULH_R)
            # 对于指令类型为 IMULH_M 的情况
            INSTR_CASE(IMULH_M)
            # 对于指令类型为 ISMULH_R 的情况
            INSTR_CASE(ISMULH_R)
            # 对于指令类型为 ISMULH_M 的情况
            INSTR_CASE(ISMULH_M)
            # 对于指令类型为 INEG_R 的情况
            INSTR_CASE(INEG_R)
            # 对于指令类型为 IXOR_R 的情况
            INSTR_CASE(IXOR_R)
            # 对于指令类型为 IXOR_M 的情况
            INSTR_CASE(IXOR_M)
            # 对于指令类型为 IROR_R 的情况
            INSTR_CASE(IROR_R)
            # 对于指令类型为 IROL_R 的情况
            INSTR_CASE(IROL_R)
            # 对于指令类型为 ISWAP_R 的情况
            INSTR_CASE(ISWAP_R)
            # 对于指令类型为 FSWAP_R 的情况
            INSTR_CASE(FSWAP_R)
            # 对于指令类型为 FADD_R 的情况
            INSTR_CASE(FADD_R)
            # 对于指令类型为 FADD_M 的情况
            INSTR_CASE(FADD_M)
            # 对于指令类型为 FSUB_R 的情况
            INSTR_CASE(FSUB_R)
            # 对于指令类型为 FSUB_M 的情况
            INSTR_CASE(FSUB_M)
            # 对于指令类型为 FSCAL_R 的情况
            INSTR_CASE(FSCAL_R)
            # 对于指令类型为 FMUL_R 的情况
            INSTR_CASE(FMUL_R)
            # 对于指令类型为 FDIV_M 的情况
            INSTR_CASE(FDIV_M)
            # 对于指令类型为 FSQRT_R 的情况
            INSTR_CASE(FSQRT_R)
            # 对于指令类型为 CBRANCH 的情况
            INSTR_CASE(CBRANCH)
            # 对于指令类型为 CFROUND 的情况
            INSTR_CASE(CFROUND)
            # 对于指令类型为 ISTORE 的情况
            INSTR_CASE(ISTORE)
    
        # 对于指令类型为 NOP 的情况
        case InstructionType::NOP:
            # 什么都不做
            break;
    
        # 对于指令类型为 IMUL_RCP 的情况，执行和 IMUL_R 相同的操作
        case InstructionType::IMUL_RCP: //executed as IMUL_R
        # 默认情况，不可达到的代码
        default:
            UNREACHABLE;
        }
    }
# 闭合前面的函数定义
```