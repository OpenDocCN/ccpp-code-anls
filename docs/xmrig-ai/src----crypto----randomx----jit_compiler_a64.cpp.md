# `xmrig\src\crypto\randomx\jit_compiler_a64.cpp`

```cpp
/*
版权声明：
版权所有 (c) 2018-2020, tevador    <tevador@gmail.com>
版权所有 (c) 2019-2020, SChernykh  <https://github.com/SChernykh>
版权所有 (c) 2019-2020, XMRig      <https://github.com/xmrig>, <support@xmrig.com>

保留所有权利。

在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在文档和/或其他提供的材料中，必须复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或其贡献者的名称来认可或推广从本软件派生的产品。

本软件按“原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他）的任何理论，即使事先已被告知此类损害的可能性。
*/


#include "crypto/randomx/jit_compiler_a64.hpp"
#include "crypto/common/VirtualMemory.h"
#include "crypto/randomx/program.hpp"
#include "crypto/randomx/reciprocal.h"
#include "crypto/randomx/superscalar.hpp"
#include "crypto/randomx/virtual_memory.hpp"


// 是否使用大页内存进行 JIT 编译
static bool hugePagesJIT = false;
# 初始化全局变量 optimizedDatasetInit，初始值为 -1
static int optimizedDatasetInit = -1;

# 设置是否使用大页内存来进行即时编译
void randomx_set_huge_pages_jit(bool hugePages)
{
    # 将传入的 hugePages 值赋给全局变量 hugePagesJIT
    hugePagesJIT = hugePages;
}

# 设置优化数据集初始化的值
void randomx_set_optimized_dataset_init(int value)
{
    # 将传入的 value 值赋给全局变量 optimizedDatasetInit
    optimizedDatasetInit = value;
}

# 定义 ARMV8A 命名空间
namespace ARMV8A {

# 定义一系列 ARMV8A 指令的常量
constexpr uint32_t B           = 0x14000000;
constexpr uint32_t EOR         = 0xCA000000;
constexpr uint32_t EOR32       = 0x4A000000;
constexpr uint32_t ADD         = 0x8B000000;
constexpr uint32_t SUB         = 0xCB000000;
constexpr uint32_t MUL         = 0x9B007C00;
constexpr uint32_t UMULH       = 0x9BC07C00;
constexpr uint32_t SMULH       = 0x9B407C00;
constexpr uint32_t MOVZ        = 0xD2800000;
constexpr uint32_t MOVN        = 0x92800000;
constexpr uint32_t MOVK        = 0xF2800000;
constexpr uint32_t ADD_IMM_LO  = 0x91000000;
constexpr uint32_t ADD_IMM_HI  = 0x91400000;
constexpr uint32_t LDR_LITERAL = 0x58000000;
constexpr uint32_t ROR         = 0x9AC02C00;
constexpr uint32_t ROR_IMM     = 0x93C00000;
constexpr uint32_t MOV_REG     = 0xAA0003E0;
constexpr uint32_t MOV_VREG_EL = 0x6E080400;
constexpr uint32_t FADD        = 0x4E60D400;
constexpr uint32_t FSUB        = 0x4EE0D400;
constexpr uint32_t FEOR        = 0x6E201C00;
constexpr uint32_t FMUL        = 0x6E60DC00;
constexpr uint32_t FDIV        = 0x6E60FC00;
constexpr uint32_t FSQRT       = 0x6EE1F800;

}

# 定义 randomx 命名空间
namespace randomx {

# 定义一些常量
static const size_t CodeSize = ((uint8_t*)randomx_init_dataset_aarch64_end) - ((uint8_t*)randomx_program_aarch64);
static const size_t MainLoopBegin = ((uint8_t*)randomx_program_aarch64_main_loop) - ((uint8_t*)randomx_program_aarch64);
static const size_t PrologueSize = ((uint8_t*)randomx_program_aarch64_vm_instructions) - ((uint8_t*)randomx_program_aarch64);
static const size_t ImulRcpLiteralsEnd = ((uint8_t*)randomx_program_aarch64_imul_rcp_literals_end) - ((uint8_t*)randomx_program_aarch64);

# 计算数据集项的大小
static size_t CalcDatasetItemSize()
{
    # 返回以下代码块的结果
    return
    # Prologue
    # 计算数据集项的大小，包括预取、混合、存储结果等操作
    ((uint8_t*)randomx_calc_dataset_item_aarch64_prefetch - (uint8_t*)randomx_calc_dataset_item_aarch64) +
    # 主循环
    RandomX_ConfigurationBase::CacheAccesses * (
        # 主循环序言
        ((uint8_t*)randomx_calc_dataset_item_aarch64_mix - ((uint8_t*)randomx_calc_dataset_item_aarch64_prefetch)) + 4 +
        # 内部主循环（指令）
        ((RandomX_ConfigurationBase::SuperscalarLatency * 3) + 2) * 16 +
        # 主循环尾声
        ((uint8_t*)randomx_calc_dataset_item_aarch64_store_result - (uint8_t*)randomx_calc_dataset_item_aarch64_mix) + 4
    ) +
    # 尾声
    ((uint8_t*)randomx_calc_dataset_item_aarch64_end - (uint8_t*)randomx_calc_dataset_item_aarch64_store_result);
}

# 定义一个包含8个元素的常量数组，表示整数寄存器映射关系
constexpr uint32_t IntRegMap[8] = { 4, 5, 6, 7, 12, 13, 14, 15 };

# JitCompilerA64 类的构造函数
JitCompilerA64::JitCompilerA64(bool hugePagesEnable, bool) :
    # 根据 hugePagesJIT 和 hugePagesEnable 的值确定是否启用 hugePages
    hugePages(hugePagesJIT && hugePagesEnable),
    # 初始化 literalPos 变量为 ImulRcpLiteralsEnd 的值
    literalPos(ImulRcpLiteralsEnd)
{
}

# JitCompilerA64 类的析构函数
JitCompilerA64::~JitCompilerA64()
{
    # 释放分配的内存空间
    freePagedMemory(code, allocatedSize);
}

# 生成程序的函数
void JitCompilerA64::generateProgram(Program& program, ProgramConfiguration& config, uint32_t)
{
    # 如果 allocatedSize 为 0，则分配 CodeSize 大小的内存空间
    if (!allocatedSize) {
        allocate(CodeSize);
    }
    # 如果启用了 XMRIG_SECURE_JIT，则允许写入内存
#ifdef XMRIG_SECURE_JIT
    else {
        enableWriting();
    }
#endif

    # 设置 codePos 变量的值为 MainLoopBegin + 4
    uint32_t codePos = MainLoopBegin + 4;

    # 生成指令：and w16, w10, ScratchpadL3Mask64
    emit32(0x121A0000 | 16 | (10 << 5) | ((RandomX_CurrentConfig.Log2_ScratchpadL3 - 7) << 10), code, codePos);

    # 生成指令：and w17, w20, ScratchpadL3Mask64
    emit32(0x121A0000 | 17 | (20 << 5) | ((RandomX_CurrentConfig.Log2_ScratchpadL3 - 7) << 10), code, codePos);

    # 设置 codePos 变量的值为 PrologueSize
    codePos = PrologueSize;
    # 设置 literalPos 变量的值为 ImulRcpLiteralsEnd
    literalPos = ImulRcpLiteralsEnd;
    # 设置 num32bitLiterals 变量的值为 0
    num32bitLiterals = 0;

    # 遍历寄存器数量
    for (uint32_t i = 0; i < RegistersCount; ++i)
        # 设置 reg_changed_offset 数组的第 i 个元素的值为 codePos
        reg_changed_offset[i] = codePos;

    # 遍历程序的指令
    for (uint32_t i = 0; i < program.getSize(); ++i)
    {
        # 获取第 i 条指令
        Instruction& instr = program(i);
        # 根据指令的操作码调用相应的函数处理指令
        (this->*engine[instr.opcode])(instr, codePos);
    }

    # 生成指令：eor w20, config.readReg2, config.readReg3
    emit32(ARMV8A::EOR32 | 20 | (IntRegMap[config.readReg2] << 5) | (IntRegMap[config.readReg3] << 16), code, codePos);

    # 生成跳转指令，跳转到主循环的开始位置
    const uint32_t offset = (((uint8_t*)randomx_program_aarch64_vm_instructions_end) - ((uint8_t*)randomx_program_aarch64)) - codePos;
    emit32(ARMV8A::B | (offset / 4), code, codePos);

    # 设置 codePos 变量的值为 (((uint8_t*)randomx_program_aarch64_cacheline_align_mask1) - ((uint8_t*)randomx_program_aarch64))
    codePos = (((uint8_t*)randomx_program_aarch64_cacheline_align_mask1) - ((uint8_t*)randomx_program_aarch64));
    # 生成指令：and w20, w20, CacheLineAlignMask
    emit32(0x121A0000 | 20 | (20 << 5) | ((RandomX_CurrentConfig.Log2_DatasetBaseSize - 7) << 10), code, codePos);

    # 生成指令：and w10, w10, CacheLineAlignMask
    # 计算 codePos 的值，即 randomx_program_aarch64_cacheline_align_mask2 和 randomx_program_aarch64 之间的偏移量
    codePos = (((uint8_t*)randomx_program_aarch64_cacheline_align_mask2) - ((uint8_t*)randomx_program_aarch64));
    # 在指定位置插入指令，将 RandomX_CurrentConfig.Log2_DatasetBaseSize 的值左移 10 位，并与其他位进行或运算，然后写入 code 中
    emit32(0x121A0000 | 10 | (10 << 5) | ((RandomX_CurrentConfig.Log2_DatasetBaseSize - 7) << 10), code, codePos);
    
    # 更新 spMix1
    # 使用异或运算符对 config.readReg0 和 config.readReg1 进行异或操作，并将结果写入 x10 寄存器
    codePos = ((uint8_t*)randomx_program_aarch64_update_spMix1) - ((uint8_t*)randomx_program_aarch64);
    emit32(ARMV8A::EOR | 10 | (IntRegMap[config.readReg0] << 5) | (IntRegMap[config.readReg1] << 16), code, codePos);
# 如果不是苹果操作系统
ifndef XMRIG_OS_APPLE
    # 刷新指令缓存
    xmrig::VirtualMemory::flushInstructionCache(reinterpret_cast<char*>(code + MainLoopBegin), codePos - MainLoopBegin);
endif



# 生成轻量级程序
void JitCompilerA64::generateProgramLight(Program& program, ProgramConfiguration& config, uint32_t datasetOffset)
{
    # 如果没有分配内存空间
    if (!allocatedSize) {
        # 分配内存空间
        allocate(CodeSize);
    }
    # 如果启用了安全 JIT
    ifdef XMRIG_SECURE_JIT
        else {
            # 启用写入权限
            enableWriting();
        }
    endif

    # 设置代码位置
    uint32_t codePos = MainLoopBegin + 4;

    # and w16, w10, ScratchpadL3Mask64
    # 将 w10 和 ScratchpadL3Mask64 进行按位与运算，结果存入 w16
    emit32(0x121A0000 | 16 | (10 << 5) | ((RandomX_CurrentConfig.Log2_ScratchpadL3 - 7) << 10), code, codePos);

    # and w17, w20, ScratchpadL3Mask64
    # 将 w20 和 ScratchpadL3Mask64 进行按位与运算，结果存入 w17
    emit32(0x121A0000 | 17 | (20 << 5) | ((RandomX_CurrentConfig.Log2_ScratchpadL3 - 7) << 10), code, codePos);

    # 设置代码位置为 PrologueSize
    codePos = PrologueSize;
    # 设置 literalPos 为 ImulRcpLiteralsEnd
    literalPos = ImulRcpLiteralsEnd;
    # 设置 num32bitLiterals 为 0
    num32bitLiterals = 0;

    # 遍历寄存器数量
    for (uint32_t i = 0; i < RegistersCount; ++i)
        # 设置 reg_changed_offset[i] 为 codePos
        reg_changed_offset[i] = codePos;

    # 遍历程序的指令
    for (uint32_t i = 0; i < program.getSize(); ++i)
    {
        # 获取指令
        Instruction& instr = program(i);
        # 根据指令的操作码调用相应的引擎函数
        (this->*engine[instr.opcode])(instr, codePos);
    }

    # 更新 spMix2
    # eor w20, config.readReg2, config.readReg3
    # 将 config.readReg2 和 config.readReg3 进行按位异或运算，结果存入 w20
    emit32(ARMV8A::EOR32 | 20 | (IntRegMap[config.readReg2] << 5) | (IntRegMap[config.readReg3] << 16), code, codePos);

    # 跳回到主循环
    # 计算偏移量
    const uint32_t offset = (((uint8_t*)randomx_program_aarch64_vm_instructions_end_light) - ((uint8_t*)randomx_program_aarch64)) - codePos;
    # 跳转到指定偏移量处
    emit32(ARMV8A::B | (offset / 4), code, codePos);

    # and w2, w9, CacheLineAlignMask
    # 将 w9 和 CacheLineAlignMask 进行按位与运算，结果存入 w2
    codePos = (((uint8_t*)randomx_program_aarch64_light_cacheline_align_mask) - ((uint8_t*)randomx_program_aarch64));
    emit32(0x121A0000 | 2 | (9 << 5) | ((RandomX_CurrentConfig.Log2_DatasetBaseSize - 7) << 10), code, codePos);

    # 更新 spMix1
    # eor x10, config.readReg0, config.readReg1
    codePos = ((uint8_t*)randomx_program_aarch64_update_spMix1) - ((uint8_t*)randomx_program_aarch64);
}
    # 在代码中生成一个32位的指令，进行异或操作
    emit32(ARMV8A::EOR | 10 | (IntRegMap[config.readReg0] << 5) | (IntRegMap[config.readReg1] << 16), code, codePos);

    # 应用数据集偏移量
    codePos = ((uint8_t*)randomx_program_aarch64_light_dataset_offset) - ((uint8_t*)randomx_program_aarch64);

    # 将数据集偏移量除以缓存行大小
    datasetOffset /= CacheLineSize;
    # 计算偏移量的低12位
    const uint32_t imm_lo = datasetOffset & ((1 << 12) - 1);
    # 计算偏移量的高位
    const uint32_t imm_hi = datasetOffset >> 12;

    # 生成一个32位的指令，进行带有低位立即数的加法操作
    emit32(ARMV8A::ADD_IMM_LO | 2 | (2 << 5) | (imm_lo << 10), code, codePos);
    # 生成一个32位的指令，进行带有高位立即数的加法操作
    emit32(ARMV8A::ADD_IMM_HI | 2 | (2 << 5) | (imm_hi << 10), code, codePos);
# 如果不是苹果操作系统
xmrig::VirtualMemory::flushInstructionCache(reinterpret_cast<char*>(code + MainLoopBegin), codePos - MainLoopBegin);



# 生成超标量哈希函数
template<size_t N>
void JitCompilerA64::generateSuperscalarHash(SuperscalarProgram(&programs)[N])
{
    # 如果没有分配内存空间
    if (!allocatedSize) {
        # 分配内存空间
        allocate(CodeSize + CalcDatasetItemSize());
    }
    # 如果启用了安全 JIT
    # 则允许写入内存
    else {
        enableWriting();
    }

    # 设置代码位置
    uint32_t codePos = CodeSize;

    # 将 randomx_calc_dataset_item_aarch64_prefetch 函数的代码复制到 code 数组中
    uint8_t* p1 = (uint8_t*)randomx_calc_dataset_item_aarch64;
    uint8_t* p2 = (uint8_t*)randomx_calc_dataset_item_aarch64_prefetch;
    memcpy(code + codePos, p1, p2 - p1);
    codePos += p2 - p1;

    # 设置 32 位字面量的数量为 64
    num32bitLiterals = 64;
    # 设置临时寄存器的编号为 12
    constexpr uint32_t tmp_reg = 12;

    # 循环执行 CacheAccesses 次
    for (size_t i = 0; i < RandomX_ConfigurationBase::CacheAccesses; ++i)
    }

    # 将 randomx_calc_dataset_item_aarch64_end 函数的代码复制到 code 数组中
    p1 = (uint8_t*)randomx_calc_dataset_item_aarch64_store_result;
    p2 = (uint8_t*)randomx_calc_dataset_item_aarch64_end;
    memcpy(code + codePos, p1, p2 - p1);
    codePos += p2 - p1;

    # 如果不是苹果操作系统
    xmrig::VirtualMemory::flushInstructionCache(reinterpret_cast<char*>(code + CodeSize), codePos - MainLoopBegin);
}

# 实例化 generateSuperscalarHash 函数
template void JitCompilerA64::generateSuperscalarHash(SuperscalarProgram(&programs)[RANDOMX_CACHE_MAX_ACCESSES]);

# 获取数据集初始化函数的指针
DatasetInitFunc* JitCompilerA64::getDatasetInitFunc() const
{
    # 如果启用了安全 JIT
    # 则允许执行内存中的代码
    enableExecution();

    # 返回数据集初始化函数的指针
    return (DatasetInitFunc*)(code + (((uint8_t*)randomx_init_dataset_aarch64) - ((uint8_t*)randomx_program_aarch64)));
}

# 获取代码的大小
size_t JitCompilerA64::getCodeSize()
{
    # 返回代码的大小
    return CodeSize;
}

# 允许写入内存
void JitCompilerA64::enableWriting() const
{
    xmrig::VirtualMemory::protectRW(code, allocatedSize);
}

# 允许执行内存中的代码
void JitCompilerA64::enableExecution() const
{
    xmrig::VirtualMemory::protectRX(code, allocatedSize);
}

# 分配内存空间
void JitCompilerA64::allocate(size_t size)
{
    # 设置已分配内存空间的大小
    allocatedSize = size;
    # 分配可执行内存空间，并将其指针赋值给 code 数组
    code = static_cast<uint8_t*>(allocExecutableMemory(allocatedSize, hugePages));
}
    // 使用 memcpy 函数将 randomx_program_aarch64 指向的内存内容复制到 code 指向的内存中
    memcpy(code, reinterpret_cast<const void *>(randomx_program_aarch64), CodeSize);
// 如果不是苹果系统
    xmrig::VirtualMemory::flushInstructionCache(reinterpret_cast<char*>(code), CodeSize);
// 结束条件

void JitCompilerA64::emitMovImmediate(uint32_t dst, uint32_t imm, uint8_t* code, uint32_t& codePos)
{
    uint32_t k = codePos;

    if (imm < (1 << 16))
    {
        // 将 imm32（低16位）移动到 tmp_reg
        emit32(ARMV8A::MOVZ | dst | (imm << 5), code, k);
    }
    else
    {
        if (num32bitLiterals < 64)
        {
            if (static_cast<int32_t>(imm) < 0)
            {
                // 将 imm32（低16位）的补码移动到 dst 寄存器
                emit32(0x4E042C00 | dst | ((num32bitLiterals / 4) << 5) | ((num32bitLiterals % 4) << 19), code, k);
            }
            else
            {
                // 将 imm32（低16位）移动到 dst 寄存器
                emit32(0x0E043C00 | dst | ((num32bitLiterals / 4) << 5) | ((num32bitLiterals % 4) << 19), code, k);
            }

            ((uint32_t*)(code + ImulRcpLiteralsEnd))[num32bitLiterals] = imm;
            ++num32bitLiterals;
        }
        else
        {
            if (static_cast<int32_t>(imm) < 0)
            {
                // 将 ~imm32（高16位）的补码移动到 tmp_reg
                emit32(ARMV8A::MOVN | dst | (1 << 21) | ((~imm >> 16) << 5), code, k);
            }
            else
            {
                // 将 imm32（高16位）移动到 tmp_reg
                emit32(ARMV8A::MOVZ | dst | (1 << 21) | ((imm >> 16) << 5), code, k);
            }

            // 将 imm32（低16位）移动到 tmp_reg
            emit32(ARMV8A::MOVK | dst | ((imm & 0xFFFF) << 5), code, k);
        }
    }

    codePos = k;
}

void JitCompilerA64::emitAddImmediate(uint32_t dst, uint32_t src, uint32_t imm, uint8_t* code, uint32_t& codePos)
{
    uint32_t k = codePos;

    if (imm < (1 << 24))
    {
        // 将立即数的低12位提取出来
        const uint32_t imm_lo = imm & ((1 << 12) - 1);
        // 将立即数的高位（除去低12位）提取出来
        const uint32_t imm_hi = imm >> 12;

        // 如果立即数既有低12位又有高位
        if (imm_lo && imm_hi)
        {
            // 发射指令，将低12位加法立即数存入目标寄存器
            emit32(ARMV8A::ADD_IMM_LO | dst | (src << 5) | (imm_lo << 10), code, k);
            // 发射指令，将高位加法立即数存入目标寄存器
            emit32(ARMV8A::ADD_IMM_HI | dst | (dst << 5) | (imm_hi << 10), code, k);
        }
        // 如果只有低12位
        else if (imm_lo)
        {
            // 发射指令，将低12位加法立即数存入目标寄存器
            emit32(ARMV8A::ADD_IMM_LO | dst | (src << 5) | (imm_lo << 10), code, k);
        }
        // 如果只有高位
        else
        {
            // 发射指令，将高位加法立即数存入目标寄存器
            emit32(ARMV8A::ADD_IMM_HI | dst | (src << 5) | (imm_hi << 10), code, k);
        }
    }
    // 如果立即数不需要分解成高低位
    else
    {
        // 临时寄存器的编号
        constexpr uint32_t tmp_reg = 20;
        // 发射指令，将立即数存入临时寄存器
        emitMovImmediate(tmp_reg, imm, code, k);

        // 发射指令，将临时寄存器的值加到目标寄存器中
        emit32(ARMV8A::ADD | dst | (src << 5) | (tmp_reg << 16), code, k);
    }

    // 更新代码位置
    codePos = k;
// 定义一个模板函数，用于加载内存数据到寄存器
template<uint32_t tmp_reg>
void JitCompilerA64::emitMemLoad(uint32_t dst, uint32_t src, Instruction& instr, uint8_t* code, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取立即数
    uint32_t imm = instr.getImm32();

    // 如果源寄存器和目标寄存器不相同
    if (src != dst)
    {
        // 根据指令的内存模式，对立即数进行掩码操作
        imm &= instr.getModMem() ? (RandomX_CurrentConfig.ScratchpadL1_Size - 1) : (RandomX_CurrentConfig.ScratchpadL2_Size - 1);
        // 将立即数加到源寄存器上，并将结果保存到临时寄存器tmp_reg中
        emitAddImmediate(tmp_reg, src, imm, code, k);

        // 构造与指令，用于对临时寄存器tmp_reg进行掩码操作
        constexpr uint32_t t = 0x927d0000 | tmp_reg | (tmp_reg << 5);
        const uint32_t andInstrL1 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL1 - 4) << 10);
        const uint32_t andInstrL2 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL2 - 4) << 10);

        // 根据内存模式选择与指令，并将结果保存到代码中
        emit32(instr.getModMem() ? andInstrL1 : andInstrL2, code, k);

        // ldr tmp_reg, [x2, tmp_reg]
        // 从内存中加载数据到临时寄存器tmp_reg中
        emit32(0xf8606840 | tmp_reg | (tmp_reg << 16), code, k);
    }
    else
    {
        // 对立即数进行掩码操作
        imm = (imm & ScratchpadL3Mask) >> 3;
        // 将立即数移动到临时寄存器tmp_reg中
        emitMovImmediate(tmp_reg, imm, code, k);

        // ldr tmp_reg, [x2, tmp_reg, lsl 3]
        // 从内存中加载数据到临时寄存器tmp_reg中
        emit32(0xf8607840 | tmp_reg | (tmp_reg << 16), code, k);
    }

    // 更新代码位置
    codePos = k;
}

// 定义一个模板函数，用于加载浮点数数据到寄存器
template<uint32_t tmp_reg_fp>
void JitCompilerA64::emitMemLoadFP(uint32_t src, Instruction& instr, uint8_t* code, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取立即数
    uint32_t imm = instr.getImm32();
    // 定义临时寄存器tmp_reg
    constexpr uint32_t tmp_reg = 19;

    // 根据指令的内存模式，对立即数进行掩码操作
    imm &= instr.getModMem() ? (RandomX_CurrentConfig.ScratchpadL1_Size - 1) : (RandomX_CurrentConfig.ScratchpadL2_Size - 1);
    // 将立即数加到源寄存器上，并将结果保存到临时寄存器tmp_reg中
    emitAddImmediate(tmp_reg, src, imm, code, k);

    // 构造与指令，用于对临时寄存器tmp_reg进行掩码操作
    constexpr uint32_t t = 0x927d0000 | tmp_reg | (tmp_reg << 5);
    const uint32_t andInstrL1 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL1 - 4) << 10);
    const uint32_t andInstrL2 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL2 - 4) << 10);

    // 根据内存模式选择与指令，并将结果保存到代码中
    emit32(instr.getModMem() ? andInstrL1 : andInstrL2, code, k);

    // add tmp_reg, x2, tmp_reg
    // 将临时寄存器tmp_reg加到寄存器x2上，并将结果保存到临时寄存器tmp_reg中
    emit32(ARMV8A::ADD | tmp_reg | (2 << 5) | (tmp_reg << 16), code, k);

    // ldpsw tmp_reg, tmp_reg + 1, [tmp_reg]
    // 从内存中加载数据到临时寄存器tmp_reg中
    # 将指定的值转换为32位二进制并写入指定的代码数组中
    emit32(0x69400000 | tmp_reg | (tmp_reg << 5) | ((tmp_reg + 1) << 10), code, k);
    
    # 将tmp_reg的值写入tmp_reg_fp.d[0]寄存器
    emit32(0x4E081C00 | tmp_reg_fp | (tmp_reg << 5), code, k);
    
    # 将tmp_reg + 1的值写入tmp_reg_fp.d[1]寄存器
    emit32(0x4E181C00 | tmp_reg_fp | ((tmp_reg + 1) << 5), code, k);
    
    # 将tmp_reg_fp.2d寄存器的值转换为浮点数，并写入tmp_reg_fp.2d寄存器
    emit32(0x4E61D800 | tmp_reg_fp | (tmp_reg_fp << 5), code, k);
    
    # 更新代码位置
    codePos = k;
}

// 处理 IADD_RS 指令
void JitCompilerA64::h_IADD_RS(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];
    // 获取移位值
    const uint32_t shift = instr.getModShift();

    // 生成 add 指令，实现 dst = dst + (src << shift)
    emit32(ARMV8A::ADD | dst | (dst << 5) | (shift << 10) | (src << 16), code, k);

    // 如果目的操作数需要位移
    if (instr.dst == RegisterNeedsDisplacement)
        // 生成立即数加法指令
        emitAddImmediate(dst, dst, instr.getImm32(), code, k);

    // 记录目的操作数的寄存器变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 处理 IADD_M 指令
void JitCompilerA64::h_IADD_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 临时寄存器编号
    constexpr uint32_t tmp_reg = 20;
    // 生成内存加载指令
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 生成 add 指令，实现 dst = dst + tmp_reg
    emit32(ARMV8A::ADD | dst | (dst << 5) | (tmp_reg << 16), code, k);

    // 记录目的操作数的寄存器变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 处理 ISUB_R 指令
void JitCompilerA64::h_ISUB_R(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源操作数和目的操作数不相同
    if (src != dst)
    {
        // 生成 sub 指令，实现 dst = dst - src
        emit32(ARMV8A::SUB | dst | (dst << 5) | (src << 16), code, k);
    }
    else
    {
        // 生成立即数减法指令
        emitAddImmediate(dst, dst, -instr.getImm32(), code, k);
    }

    // 记录目的操作数的寄存器变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 处理 ISUB_M 指令
void JitCompilerA64::h_ISUB_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 临时寄存器编号
    constexpr uint32_t tmp_reg = 20;
    // 生成内存加载指令
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 生成 sub 指令，实现 dst = dst - tmp_reg
    emit32(ARMV8A::SUB | dst | (dst << 5) | (tmp_reg << 16), code, k);

    // 记录目的操作数的寄存器变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 处理 IMUL_R 指令
void JitCompilerA64::h_IMUL_R(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;
    // 将指令中的源操作数映射为对应的整数寄存器
    uint32_t src = IntRegMap[instr.src];
    // 将指令中的目的操作数映射为对应的整数寄存器，并且不可更改
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源操作数等于目的操作数
    if (src == dst)
    {
        // 将源操作数设置为20
        src = 20;
        // 发出一条立即数移动指令，将20移动到目的操作数中
        emitMovImmediate(src, instr.getImm32(), code, k);
    }

    // 发出一条32位的乘法指令，将源操作数和目的操作数相乘，结果存储在目的操作数中
    emit32(ARMV8A::MUL | dst | (dst << 5) | (src << 16), code, k);

    // 记录目的操作数的寄存器变化偏移量
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
// 定义 JitCompilerA64 类的 h_IMUL_M 方法，用于处理指令的整数乘法操作
void JitCompilerA64::h_IMUL_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 定义临时寄存器
    constexpr uint32_t tmp_reg = 20;
    // 调用 emitMemLoad 方法，加载内存数据到临时寄存器
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 生成 MUL 指令，实现目的寄存器和临时寄存器的乘法操作
    emit32(ARMV8A::MUL | dst | (dst << 5) | (tmp_reg << 16), code, k);

    // 更新目的寄存器的修改偏移量
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_IMULH_R 方法，用于处理指令的整数乘法高位操作
void JitCompilerA64::h_IMULH_R(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 生成 UMULH 指令，实现目的寄存器和源寄存器的无符号乘法高位操作
    emit32(ARMV8A::UMULH | dst | (dst << 5) | (src << 16), code, k);

    // 更新目的寄存器的修改偏移量
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_IMULH_M 方法，用于处理指令的整数乘法高位操作
void JitCompilerA64::h_IMULH_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 定义临时寄存器
    constexpr uint32_t tmp_reg = 20;
    // 调用 emitMemLoad 方法，加载内存数据到临时寄存器
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 生成 UMULH 指令，实现目的寄存器和临时寄存器的无符号乘法高位操作
    emit32(ARMV8A::UMULH | dst | (dst << 5) | (tmp_reg << 16), code, k);

    // 更新目的寄存器的修改偏移量
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_ISMULH_R 方法，用于处理指令的整数乘法高位操作
void JitCompilerA64::h_ISMULH_R(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 生成 SMULH 指令，实现目的寄存器和源寄存器的有符号乘法高位操作
    emit32(ARMV8A::SMULH | dst | (dst << 5) | (src << 16), code, k);

    // 更新目的寄存器的修改偏移量
    reg_changed_offset[instr.dst] = k;
    // 更新代码位置
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_ISMULH_M 方法，用于处理指令的整数乘法高位操作
void JitCompilerA64::h_ISMULH_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数和目的操作数的寄存器映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 定义临时寄存器
    constexpr uint32_t tmp_reg = 20;
    // 调用 emitMemLoad 方法，加载内存数据到临时寄存器
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 生成 SMULH 指令，实现目的寄存器和临时寄存器的有符号乘法高位操作
    emit32(ARMV8A::SMULH | dst | (dst << 5) | (tmp_reg << 16), code, k);
    # 将 instr.dst 对应的值 k 存入 reg_changed_offset 字典中
    reg_changed_offset[instr.dst] = k;
    # 将 codePos 的值设为 k
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_IMUL_RCP 方法，接收 Instruction 对象和 codePos 变量作为参数
void JitCompilerA64::h_IMUL_RCP(Instruction& instr, uint32_t& codePos)
{
    // 获取除数
    const uint64_t divisor = instr.getImm32();
    // 如果除数为0或者是2的幂次方，则返回
    if (isZeroOrPowerOf2(divisor))
        return;

    // 记录当前的 codePos 值
    uint32_t k = codePos;

    // 定义临时寄存器的编号
    constexpr uint32_t tmp_reg = 20;
    // 获取目标寄存器的编号
    const uint32_t dst = IntRegMap[instr.dst];

    // 定义常量 N，值为 2^63
    constexpr uint64_t N = 1ULL << 63;
    // 计算商 q 和余数 r
    const uint64_t q = N / divisor;
    const uint64_t r = N % divisor;
    // 使用内置函数 __builtin_clzll 计算除数的二进制表示中前导零的个数，得到右移的位数
    #ifdef __GNUC__
    const uint64_t shift = 64 - __builtin_clzll(divisor);
    #else
    // 如果编译器不支持 __builtin_clzll 函数，则使用循环计算右移的位数
    uint64_t shift = 32;
    for (uint64_t k = 1U << 31; (k & divisor) == 0; k >>= 1)
        --shift;
    #endif

    // 计算 literal_id，即 reciprocal 的索引
    const uint32_t literal_id = (ImulRcpLiteralsEnd - literalPos) / sizeof(uint64_t);

    // 更新 literalPos 的值
    literalPos -= sizeof(uint64_t);
    // 将 (q << shift) + ((r << shift) / divisor) 存储到 code + literalPos 的位置
    *(uint64_t*)(code + literalPos) = (q << shift) + ((r << shift) / divisor);

    // 如果 literal_id 小于 12
    if (literal_id < 12)
    {
        // 定义 literal_regs 数组，存储 literal 寄存器的编号
        static constexpr uint32_t literal_regs[12] = { 30 << 16, 29 << 16, 28 << 16, 27 << 16, 26 << 16, 25 << 16, 24 << 16, 23 << 16, 22 << 16, 21 << 16, 11 << 16, 0 };

        // 生成 mul 指令，将 dst 寄存器与 literal_reg 寄存器相乘，并将结果存储到 dst 寄存器中
        emit32(ARMV8A::MUL | dst | (dst << 5) | literal_regs[literal_id], code, k);
    }
    else
    {
        // 计算 offset，即 reciprocal 在 code 中的偏移量
        const uint32_t offset = (literalPos - k) / 4;
        // 生成 ldr 指令，将 reciprocal 加载到 tmp_reg 寄存器中
        emit32(ARMV8A::LDR_LITERAL | tmp_reg | (offset << 5), code, k);

        // 生成 mul 指令，将 dst 寄存器与 tmp_reg 寄存器相乘，并将结果存储到 dst 寄存器中
        emit32(ARMV8A::MUL | dst | (dst << 5) | (tmp_reg << 16), code, k);
    }

    // 更新 reg_changed_offset 数组中 dst 寄存器的值为 k
    reg_changed_offset[instr.dst] = k;
    // 更新 codePos 的值为 k
    codePos = k;
}

// 定义 JitCompilerA64 类的 h_INEG_R 方法，接收 Instruction 对象和 codePos 变量作为参数
void JitCompilerA64::h_INEG_R(Instruction& instr, uint32_t& codePos)
{
    // 获取目标寄存器的编号
    const uint32_t dst = IntRegMap[instr.dst];

    // 生成 sub 指令，将 xzr 寄存器与 dst 寄存器相减，并将结果存储到 dst 寄存器中
    emit32(ARMV8A::SUB | dst | (31 << 5) | (dst << 16), code, codePos);

    // 更新 reg_changed_offset 数组中 dst 寄存器的值为 codePos
    reg_changed_offset[instr.dst] = codePos;
}

// 定义 JitCompilerA64 类的 h_IXOR_R 方法，接收 Instruction 对象和 codePos 变量作为参数
void JitCompilerA64::h_IXOR_R(Instruction& instr, uint32_t& codePos)
{
    // 记录当前的 codePos 值
    uint32_t k = codePos;

    // 获取源寄存器和目标寄存器的编号
    uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源寄存器和目标寄存器相同
    if (src == dst)
    {
        # 将20赋值给src变量
        src = 20;
        # 调用emitMovImmediate函数，将src的值作为立即数传递给instr.getImm32()，并将结果存储在code的k位置
        emitMovImmediate(src, instr.getImm32(), code, k);
    }
    
    # 执行eor指令，将dst与src进行异或操作，并将结果存储在dst中
    emit32(ARMV8A::EOR | dst | (dst << 5) | (src << 16), code, k);
    
    # 更新reg_changed_offset字典中instr.dst对应的值为k
    reg_changed_offset[instr.dst] = k;
    # 更新codePos的值为k
    codePos = k;
}

// 处理 IXOR_M 指令
void JitCompilerA64::h_IXOR_M(Instruction& instr, uint32_t& codePos)
{
    // 记录当前指令的位置
    uint32_t k = codePos;

    // 获取源寄存器和目标寄存器的映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 定义一个临时寄存器
    constexpr uint32_t tmp_reg = 20;
    // 加载内存中的数据到临时寄存器
    emitMemLoad<tmp_reg>(dst, src, instr, code, k);

    // 执行异或操作 dst = dst ^ tmp_reg
    emit32(ARMV8A::EOR | dst | (dst << 5) | (tmp_reg << 16), code, k);

    // 记录目标寄存器的变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新指令位置
    codePos = k;
}

// 处理 IROR_R 指令
void JitCompilerA64::h_IROR_R(Instruction& instr, uint32_t& codePos)
{
    // 获取源寄存器和目标寄存器的映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源寄存器和目标寄存器不相同
    if (src != dst)
    {
        // 执行循环右移操作 dst = dst >> src
        emit32(ARMV8A::ROR | dst | (dst << 5) | (src << 16), code, codePos);
    }
    else
    {
        // 执行循环右移操作 dst = dst >> imm
        emit32(ARMV8A::ROR_IMM | dst | (dst << 5) | ((instr.getImm32() & 63) << 10) | (dst << 16), code, codePos);
    }

    // 记录目标寄存器的变化位置
    reg_changed_offset[instr.dst] = codePos;
}

// 处理 IROL_R 指令
void JitCompilerA64::h_IROL_R(Instruction& instr, uint32_t& codePos)
{
    // 记录当前指令的位置
    uint32_t k = codePos;

    // 获取源寄存器和目标寄存器的映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源寄存器和目标寄存器不相同
    if (src != dst)
    {
        // 定义一个临时寄存器
        constexpr uint32_t tmp_reg = 20;

        // 执行减法操作 tmp_reg = xzr - src
        emit32(ARMV8A::SUB | tmp_reg | (31 << 5) | (src << 16), code, k);

        // 执行循环右移操作 dst = dst >> tmp_reg
        emit32(ARMV8A::ROR | dst | (dst << 5) | (tmp_reg << 16), code, k);
    }
    else
    {
        // 执行循环右移操作 dst = dst >> imm
        emit32(ARMV8A::ROR_IMM | dst | (dst << 5) | ((-instr.getImm32() & 63) << 10) | (dst << 16), code, k);
    }

    // 记录目标寄存器的变化位置
    reg_changed_offset[instr.dst] = k;
    // 更新指令位置
    codePos = k;
}

// 处理 ISWAP_R 指令
void JitCompilerA64::h_ISWAP_R(Instruction& instr, uint32_t& codePos)
{
    // 获取源寄存器和目标寄存器的映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];

    // 如果源寄存器和目标寄存器相同，则直接返回
    if (src == dst)
        return;

    // 记录当前指令的位置
    uint32_t k = codePos;

    // 定义一个临时寄存器
    constexpr uint32_t tmp_reg = 20;
    // 将目标寄存器的值复制到临时寄存器
    emit32(ARMV8A::MOV_REG | tmp_reg | (dst << 16), code, k);
    # 在代码中生成一个32位的ARMV8A指令，将src寄存器的值移动到dst寄存器中，并将结果写入到code数组的第k个位置
    emit32(ARMV8A::MOV_REG | dst | (src << 16), code, k);
    # 在代码中生成一个32位的ARMV8A指令，将tmp_reg寄存器的值移动到src寄存器中，并将结果写入到code数组的第k个位置
    emit32(ARMV8A::MOV_REG | src | (tmp_reg << 16), code, k);
    
    # 更新指令中src寄存器对应的寄存器变化偏移量为k
    reg_changed_offset[instr.src] = k;
    # 更新指令中dst寄存器对应的寄存器变化偏移量为k
    reg_changed_offset[instr.dst] = k;
    # 更新代码位置为k
    codePos = k;
// 反转浮点寄存器中的字节顺序
void JitCompilerA64::h_FSWAP_R(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 计算目标寄存器的位置
    const uint32_t dst = instr.dst + 16;

    // 定义临时浮点寄存器和索引值
    constexpr uint32_t tmp_reg_fp = 28;
    constexpr uint32_t src_index1 = 1 << 14;
    constexpr uint32_t dst_index1 = 1 << 20;

    // 发射指令，交换字节顺序
    emit32(ARMV8A::MOV_VREG_EL | tmp_reg_fp | (dst << 5) | src_index1, code, k);
    emit32(ARMV8A::MOV_VREG_EL | dst | (dst << 5) | dst_index1, code, k);
    emit32(ARMV8A::MOV_VREG_EL | dst | (tmp_reg_fp << 5), code, k);

    // 更新代码位置
    codePos = k;
}

// 浮点寄存器相加
void JitCompilerA64::h_FADD_R(Instruction& instr, uint32_t& codePos)
{
    // 计算源寄存器和目标寄存器的位置
    const uint32_t src = (instr.src % 4) + 24;
    const uint32_t dst = (instr.dst % 4) + 16;

    // 发射指令，执行浮点相加
    emit32(ARMV8A::FADD | dst | (dst << 5) | (src << 16), code, codePos);
}

// 内存中的浮点数和浮点寄存器相加
void JitCompilerA64::h_FADD_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 计算源寄存器和目标寄存器的位置
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = (instr.dst % 4) + 16;

    // 定义临时浮点寄存器
    constexpr uint32_t tmp_reg_fp = 28;

    // 加载内存中的浮点数到临时浮点寄存器，然后执行浮点相加
    emitMemLoadFP<tmp_reg_fp>(src, instr, code, k);
    emit32(ARMV8A::FADD | dst | (dst << 5) | (tmp_reg_fp << 16), code, k);

    // 更新代码位置
    codePos = k;
}

// 浮点寄存器相减
void JitCompilerA64::h_FSUB_R(Instruction& instr, uint32_t& codePos)
{
    // 计算源寄存器和目标寄存器的位置
    const uint32_t src = (instr.src % 4) + 24;
    const uint32_t dst = (instr.dst % 4) + 16;

    // 发射指令，执行浮点相减
    emit32(ARMV8A::FSUB | dst | (dst << 5) | (src << 16), code, codePos);
}

// 内存中的浮点数和浮点寄存器相减
void JitCompilerA64::h_FSUB_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 计算源寄存器和目标寄存器的位置
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = (instr.dst % 4) + 16;

    // 定义临时浮点寄存器
    constexpr uint32_t tmp_reg_fp = 28;

    // 加载内存中的浮点数到临时浮点寄存器，然后执行浮点相减
    emitMemLoadFP<tmp_reg_fp>(src, instr, code, k);
    emit32(ARMV8A::FSUB | dst | (dst << 5) | (tmp_reg_fp << 16), code, k);

    // 更新代码位置
    codePos = k;
}

// 浮点寄存器清零
void JitCompilerA64::h_FSCAL_R(Instruction& instr, uint32_t& codePos)
{
    // 计算目标寄存器的位置
    const uint32_t dst = (instr.dst % 4) + 16;

    // 发射指令，执行浮点寄存器清零
    emit32(ARMV8A::FEOR | dst | (dst << 5) | (31 << 16), code, codePos);
}
// 实现 A64 架构的 FMUL 指令
void JitCompilerA64::h_FMUL_R(Instruction& instr, uint32_t& codePos)
{
    // 计算源操作数的寄存器编号
    const uint32_t src = (instr.src % 4) + 24;
    // 计算目的操作数的寄存器编号
    const uint32_t dst = (instr.dst % 4) + 20;

    // 生成 FMUL 指令并写入代码
    emit32(ARMV8A::FMUL | dst | (dst << 5) | (src << 16), code, codePos);
}

// 实现 A64 架构的 FDIV 指令
void JitCompilerA64::h_FDIV_M(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    // 计算目的操作数的寄存器编号
    const uint32_t dst = (instr.dst % 4) + 20;

    // 定义临时浮点寄存器编号
    constexpr uint32_t tmp_reg_fp = 28;
    // 生成加载浮点数内存操作指令并写入代码
    emitMemLoadFP<tmp_reg_fp>(src, instr, code, k);

    // 生成与操作指令并写入代码
    emit32(0x4E201C00 | tmp_reg_fp | (tmp_reg_fp << 5) | (29 << 16), code, k);

    // 生成或操作指令并写入代码
    emit32(0x4EA01C00 | tmp_reg_fp | (tmp_reg_fp << 5) | (30 << 16), code, k);

    // 生成 FDIV 指令并写入代码
    emit32(ARMV8A::FDIV | dst | (dst << 5) | (tmp_reg_fp << 16), code, k);

    // 更新代码位置
    codePos = k;
}

// 实现 A64 架构的 FSQRT 指令
void JitCompilerA64::h_FSQRT_R(Instruction& instr, uint32_t& codePos)
{
    // 计算目的操作数的寄存器编号
    const uint32_t dst = (instr.dst % 4) + 20;

    // 生成 FSQRT 指令并写入代码
    emit32(ARMV8A::FSQRT | dst | (dst << 5), code, codePos);
}

// 实现 A64 架构的 CBRANCH 指令
void JitCompilerA64::h_CBRANCH(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取目的操作数的寄存器编号
    const uint32_t dst = IntRegMap[instr.dst];
    // 获取条件码修饰值
    const uint32_t modCond = instr.getModCond();
    // 计算偏移量
    const uint32_t shift = modCond + RandomX_ConfigurationBase::JumpOffset;
    // 计算立即数
    const uint32_t imm = (instr.getImm32() | (1U << shift)) & ~(1U << (shift - 1));

    // 生成加法立即数指令并写入代码
    emitAddImmediate(dst, dst, imm, code, k);

    // 生成测试指令并写入代码
    emit32((0xF2781C1F - (modCond << 16)) | (dst << 5), code, k);

    // 计算跳转偏移量
    int32_t offset = reg_changed_offset[instr.dst];
    offset = ((offset - k) >> 2) & ((1 << 19) - 1);

    // 生成条件跳转指令并写入代码
    emit32(0x54000000 | (offset << 5), code, k);

    // 更新寄存器变化偏移量
    for (uint32_t i = 0; i < RegistersCount; ++i)
        reg_changed_offset[i] = k;

    // 更新代码位置
    codePos = k;
}

// 实现 A64 架构的 CFROUND 指令
void JitCompilerA64::h_CFROUND(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源操作数的寄存器编号
    const uint32_t src = IntRegMap[instr.src];
    # 定义一个常量 tmp_reg，值为 20
    constexpr uint32_t tmp_reg = 20;
    # 定义一个常量 fpcr_tmp_reg，值为 8
    constexpr uint32_t fpcr_tmp_reg = 8;
    
    # 将 src 按照 imm 的值进行循环右移，并将结果存入 tmp_reg
    emit32(ARMV8A::ROR_IMM | tmp_reg | (src << 5) | ((instr.getImm32() & 63) << 10) | (src << 16), code, k);
    
    # 将 tmp_reg 的第 40 位到第 41 位的值存入 fpcr_tmp_reg
    emit32(0xB3580400 | fpcr_tmp_reg | (tmp_reg << 5), code, k);
    
    # 将 fpcr_tmp_reg 的位反转，并将结果存入 tmp_reg
    emit32(0xDAC00000 | tmp_reg | (fpcr_tmp_reg << 5), code, k);
    
    # 将 tmp_reg 的值存入 fpcr 寄存器
    emit32(0xD51B4400 | tmp_reg, code, k);
    
    # 更新 codePos 的值为 k
    codePos = k;
// 存储指令的实现，将指令中的数据存储到内存中
void JitCompilerA64::h_ISTORE(Instruction& instr, uint32_t& codePos)
{
    // 保存当前代码位置
    uint32_t k = codePos;

    // 获取源寄存器和目标寄存器的映射
    const uint32_t src = IntRegMap[instr.src];
    const uint32_t dst = IntRegMap[instr.dst];
    // 临时寄存器的编号
    constexpr uint32_t tmp_reg = 20;

    // 获取指令中的立即数
    uint32_t imm = instr.getImm32();

    // 根据条件和内存修改标志位对立即数进行处理
    if (instr.getModCond() < StoreL3Condition)
        imm &= instr.getModMem() ? (RandomX_CurrentConfig.ScratchpadL1_Size - 1) : (RandomX_CurrentConfig.ScratchpadL2_Size - 1);
    else
        imm &= RandomX_CurrentConfig.ScratchpadL3_Size - 1;

    // 生成加法指令，将立即数加到目标寄存器中
    emitAddImmediate(tmp_reg, dst, imm, code, k);

    // 生成与操作指令，根据不同的条件和内存修改标志位选择不同的操作数
    constexpr uint32_t t = 0x927d0000 | tmp_reg | (tmp_reg << 5);
    const uint32_t andInstrL1 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL1 - 4) << 10);
    const uint32_t andInstrL2 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL2 - 4) << 10);
    const uint32_t andInstrL3 = t | ((RandomX_CurrentConfig.Log2_ScratchpadL3 - 4) << 10);

    // 生成32位指令，根据条件和内存修改标志位选择不同的与操作指令
    emit32((instr.getModCond() < StoreL3Condition) ? (instr.getModMem() ? andInstrL1 : andInstrL2) : andInstrL3, code, k);

    // 生成存储指令，将源寄存器的值存储到内存中
    emit32(0xF8206840 | src | (tmp_reg << 16), code, k);

    // 更新代码位置
    codePos = k;
}

// 空操作指令的实现
void JitCompilerA64::h_NOP(Instruction& instr, uint32_t& codePos)
{
}

// 指令生成器的初始化
InstructionGeneratorA64 JitCompilerA64::engine[256] = {};

}
```