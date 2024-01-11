# `xmrig\src\crypto\randomx\randomx.cpp`

```
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，就可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。无论在任何情况下，版权持有人或贡献者对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他），即使事先已被告知此类损害的可能性。
*/



#include "crypto/randomx/common.hpp"
#include "crypto/randomx/randomx.h"
#include "crypto/randomx/dataset.hpp"
#include "crypto/randomx/vm_interpreted.hpp"
#include "crypto/randomx/vm_interpreted_light.hpp"
#include "crypto/randomx/vm_compiled.hpp"
#include "crypto/randomx/vm_compiled_light.hpp"
#include "crypto/randomx/blake2/blake2.h"


注释：导入所需的头文件


#if defined(_M_X64) || defined(__x86_64__)
#include "crypto/randomx/jit_compiler_x86_static.hpp"


注释：如果定义了_M_X64或__x86_64__，则导入x86静态JIT编译器的头文件。
#elif (XMRIG_ARM == 8)
#include "crypto/randomx/jit_compiler_a64_static.hpp"
#endif

#include "backend/cpu/Cpu.h"
#include "crypto/common/VirtualMemory.h"
#include <mutex>

#include <cassert>

#include "crypto/rx/Profiler.h"

// RandomX_ConfigurationWownero 类的构造函数
RandomX_ConfigurationWownero::RandomX_ConfigurationWownero()
{
    // 设置 ArgonSalt 字符串
    ArgonSalt = "RandomWOW\x01";
    // 设置 ProgramIterations 变量
    ProgramIterations = 1024;
    // 设置 ProgramCount 变量
    ProgramCount = 16;
    // 设置 ScratchpadL2_Size 变量
    ScratchpadL2_Size = 131072;
    // 设置 ScratchpadL3_Size 变量
    ScratchpadL3_Size = 1048576;

    // 设置 RANDOMX_FREQ_IADD_RS 变量
    RANDOMX_FREQ_IADD_RS = 25;
    // 设置 RANDOMX_FREQ_IROR_R 变量
    RANDOMX_FREQ_IROR_R = 10;
    // 设置 RANDOMX_FREQ_IROL_R 变量
    RANDOMX_FREQ_IROL_R = 0;
    // 设置 RANDOMX_FREQ_FSWAP_R 变量
    RANDOMX_FREQ_FSWAP_R = 8;
    // 设置 RANDOMX_FREQ_FADD_R 变量
    RANDOMX_FREQ_FADD_R = 20;
    // 设置 RANDOMX_FREQ_FSUB_R 变量
    RANDOMX_FREQ_FSUB_R = 20;
    // 设置 RANDOMX_FREQ_FMUL_R 变量
    RANDOMX_FREQ_FMUL_R = 20;
    // 设置 RANDOMX_FREQ_CBRANCH 变量
    RANDOMX_FREQ_CBRANCH = 16;

    // 设置 fillAes4Rx4_Key 数组的元素
    fillAes4Rx4_Key[0] = rx_set_int_vec_i128(0xcf359e95, 0x141f82b7, 0x7ffbe4a6, 0xf890465d);
    fillAes4Rx4_Key[1] = rx_set_int_vec_i128(0x6741ffdc, 0xbd5c5ac3, 0xfee8278a, 0x6a55c450);
    fillAes4Rx4_Key[2] = rx_set_int_vec_i128(0x3d324aac, 0xa7279ad2, 0xd524fde4, 0x114c47a4);
    fillAes4Rx4_Key[3] = rx_set_int_vec_i128(0x76f6db08, 0x42d3dbd9, 0x99a9aeff, 0x810c3a2a);
    fillAes4Rx4_Key[4] = fillAes4Rx4_Key[0];
    fillAes4Rx4_Key[5] = fillAes4Rx4_Key[1];
    fillAes4Rx4_Key[6] = fillAes4Rx4_Key[2];
    fillAes4Rx4_Key[7] = fillAes4Rx4_Key[3];
}

// RandomX_ConfigurationArqma 类的构造函数
RandomX_ConfigurationArqma::RandomX_ConfigurationArqma()
{
    // 设置 ArgonIterations 变量
    ArgonIterations = 1;
    // 设置 ArgonSalt 字符串
    ArgonSalt = "RandomARQ\x01";
    // 设置 ProgramIterations 变量
    ProgramIterations = 1024;
    // 设置 ProgramCount 变量
    ProgramCount = 4;
    // 设置 ScratchpadL2_Size 变量
    ScratchpadL2_Size = 131072;
    // 设置 ScratchpadL3_Size 变量
    ScratchpadL3_Size = 262144;
}

// RandomX_ConfigurationGraft 类的构造函数
RandomX_ConfigurationGraft::RandomX_ConfigurationGraft()
{
    // 设置 ArgonLanes 变量
    ArgonLanes = 2;
    // 设置 ArgonSalt 字符串
    ArgonSalt = "RandomX-Graft\x01";
    // 设置 ProgramSize 变量
    ProgramSize = 280;
    // 设置 RANDOMX_FREQ_IROR_R 变量
    RANDOMX_FREQ_IROR_R = 7;
    // 设置 RANDOMX_FREQ_IROL_R 变量
    RANDOMX_FREQ_IROL_R = 3;
}

// RandomX_ConfigurationSafex 类的构造函数
RandomX_ConfigurationSafex::RandomX_ConfigurationSafex()
{
    // 设置 ArgonSalt 字符串
    ArgonSalt = "RandomSFX\x01";
}

// RandomX_ConfigurationKeva 类的构造函数
RandomX_ConfigurationKeva::RandomX_ConfigurationKeva()
{
    // 设置 ArgonSalt 字符串
    ArgonSalt = "RandomKV\x01";
    // 设置 ScratchpadL2_Size 变量
    ScratchpadL2_Size = 131072;
    // 设置 ScratchpadL3_Size 变量
    ScratchpadL3_Size = 1048576;
}

// RandomX_ConfigurationBase 类的构造函数
RandomX_ConfigurationBase::RandomX_ConfigurationBase()
    # 设置 Argon2 算法的迭代次数为 3
    ArgonIterations(3)
    # 设置 Argon2 算法的并行度为 1
    ArgonLanes(1)
    # 设置 Argon2 算法的盐值为 "RandomX\x03"
    ArgonSalt("RandomX\x03")
    # 设置第一级缓存的大小为 16384 字节
    ScratchpadL1_Size(16384)
    # 设置第二级缓存的大小为 262144 字节
    ScratchpadL2_Size(262144)
    # 设置第三级缓存的大小为 2097152 字节
    ScratchpadL3_Size(2097152)
    # 设置程序的大小为 256 字节
    ProgramSize(256)
    # 设置程序的迭代次数为 2048
    ProgramIterations(2048)
    # 设置程序的数量为 8
    ProgramCount(8)
    # 设置 RANDOMX_FREQ_IADD_RS 的频率为 16
    RANDOMX_FREQ_IADD_RS(16)
    # 设置 RANDOMX_FREQ_IADD_M 的频率为 7
    RANDOMX_FREQ_IADD_M(7)
    # 设置 RANDOMX_FREQ_ISUB_R 的频率为 16
    RANDOMX_FREQ_ISUB_R(16)
    # 设置 RANDOMX_FREQ_ISUB_M 的频率为 7
    RANDOMX_FREQ_ISUB_M(7)
    # 设置 RANDOMX_FREQ_IMUL_R 的频率为 16
    RANDOMX_FREQ_IMUL_R(16)
    # 设置 RANDOMX_FREQ_IMUL_M 的频率为 4
    RANDOMX_FREQ_IMUL_M(4)
    # 设置 RANDOMX_FREQ_IMULH_R 的频率为 4
    RANDOMX_FREQ_IMULH_R(4)
    # 设置 RANDOMX_FREQ_IMULH_M 的频率为 1
    RANDOMX_FREQ_IMULH_M(1)
    # 设置 RANDOMX_FREQ_ISMULH_R 的频率为 4
    RANDOMX_FREQ_ISMULH_R(4)
    # 设置 RANDOMX_FREQ_ISMULH_M 的频率为 1
    RANDOMX_FREQ_ISMULH_M(1)
    # 设置 RANDOMX_FREQ_IMUL_RCP 的频率为 8
    RANDOMX_FREQ_IMUL_RCP(8)
    # 设置 RANDOMX_FREQ_INEG_R 的频率为 2
    RANDOMX_FREQ_INEG_R(2)
    # 设置 RANDOMX_FREQ_IXOR_R 的频率为 15
    RANDOMX_FREQ_IXOR_R(15)
    # 设置 RANDOMX_FREQ_IXOR_M 的频率为 5
    RANDOMX_FREQ_IXOR_M(5)
    # 设置 RANDOMX_FREQ_IROR_R 的频率为 8
    RANDOMX_FREQ_IROR_R(8)
    # 设置 RANDOMX_FREQ_IROL_R 的频率为 2
    RANDOMX_FREQ_IROL_R(2)
    # 设置 RANDOMX_FREQ_ISWAP_R 的频率为 4
    RANDOMX_FREQ_ISWAP_R(4)
    # 设置 RANDOMX_FREQ_FSWAP_R 的频率为 4
    RANDOMX_FREQ_FSWAP_R(4)
    # 设置 RANDOMX_FREQ_FADD_R 的频率为 16
    RANDOMX_FREQ_FADD_R(16)
    # 设置 RANDOMX_FREQ_FADD_M 的频率为 5
    RANDOMX_FREQ_FADD_M(5)
    # 设置 RANDOMX_FREQ_FSUB_R 的频率为 16
    RANDOMX_FREQ_FSUB_R(16)
    # 设置 RANDOMX_FREQ_FSUB_M 的频率为 5
    RANDOMX_FREQ_FSUB_M(5)
    # 设置 RANDOMX_FREQ_FSCAL_R 的频率为 6
    RANDOMX_FREQ_FSCAL_R(6)
    # 设置 RANDOMX_FREQ_FMUL_R 的频率为 32
    RANDOMX_FREQ_FMUL_R(32)
    # 设置 RANDOMX_FREQ_FDIV_M 的频率为 4
    RANDOMX_FREQ_FDIV_M(4)
    # 设置 RANDOMX_FREQ_FSQRT_R 的频率为 6
    RANDOMX_FREQ_FSQRT_R(6)
    # 设置 RANDOMX_FREQ_CBRANCH 的频率为 25
    RANDOMX_FREQ_CBRANCH(25)
    # 设置 RANDOMX_FREQ_CFROUND 的频率为 1
    RANDOMX_FREQ_CFROUND(1)
    # 设置 RANDOMX_FREQ_ISTORE 的频率为 16
    RANDOMX_FREQ_ISTORE(16)
    # 设置 RANDOMX_FREQ_NOP 的频率为 0
    RANDOMX_FREQ_NOP(0)
{
    # 设置 fillAes4Rx4_Key 数组的值
    fillAes4Rx4_Key[0] = rx_set_int_vec_i128(0x99e5d23f, 0x2f546d2b, 0xd1833ddb, 0x6421aadd);
    fillAes4Rx4_Key[1] = rx_set_int_vec_i128(0xa5dfcde5, 0x06f79d53, 0xb6913f55, 0xb20e3450);
    fillAes4Rx4_Key[2] = rx_set_int_vec_i128(0x171c02bf, 0x0aa4679f, 0x515e7baf, 0x5c3ed904);
    fillAes4Rx4_Key[3] = rx_set_int_vec_i128(0xd8ded291, 0xcd673785, 0xe78f5d08, 0x85623763);
    fillAes4Rx4_Key[4] = rx_set_int_vec_i128(0x229effb4, 0x3d518b6d, 0xe3d6a7a6, 0xb5826f73);
    fillAes4Rx4_Key[5] = rx_set_int_vec_i128(0xb272b7d2, 0xe9024d4e, 0x9c10b3d9, 0xc7566bf3);
    fillAes4Rx4_Key[6] = rx_set_int_vec_i128(0xf63befa7, 0x2ba9660a, 0xf765a38b, 0xf273c9e7);
    fillAes4Rx4_Key[7] = rx_set_int_vec_i128(0xc0b0762d, 0x0c06d1fd, 0x915839de, 0x7a7cd609);

#    if defined(XMRIG_FEATURE_ASM) && (defined(_M_X64) || defined(__x86_64__))
    # 定义一个 lambda 函数 addr，用于获取函数的地址
    auto addr = [](void (*func)()) {
        const uint8_t* p = reinterpret_cast<const uint8_t*>(func);
#        if defined(_MSC_VER)
        # 如果是 Visual Studio 编译器，检查是否需要修复地址
        if (p[0] == 0xE9) {
            p += *(const int32_t*)(p + 1) + 5;
        }
#        endif
        return p;
    };

    {
        # 获取 randomx_sshash_prefetch 函数的地址
        const uint8_t* a = addr(randomx_sshash_prefetch);
        # 获取 randomx_sshash_end 函数的地址
        const uint8_t* b = addr(randomx_sshash_end);
        # 将 randomx_sshash_prefetch 到 randomx_sshash_end 之间的代码复制到 codeSshPrefetchTweaked 数组中
        memcpy(codeSshPrefetchTweaked, a, b - a);
    }
    # 如果 CPU 支持 BMI2 指令集
    if (xmrig::Cpu::info()->hasBMI2()) {
        # 获取 randomx_prefetch_scratchpad_bmi2 函数的地址
        const uint8_t* a = addr(randomx_prefetch_scratchpad_bmi2);
        # 获取 randomx_prefetch_scratchpad_end 函数的地址
        const uint8_t* b = addr(randomx_prefetch_scratchpad_end);
        # 将 randomx_prefetch_scratchpad_bmi2 到 randomx_prefetch_scratchpad_end 之间的代码复制到 codePrefetchScratchpadTweaked 数组中
        memcpy(codePrefetchScratchpadTweaked, a, b - a);
        # 设置 codePrefetchScratchpadTweakedSize 的值为复制的代码的大小
        codePrefetchScratchpadTweakedSize = b - a;
    }
    else {
        # 获取 randomx_prefetch_scratchpad 函数的地址
        const uint8_t* a = addr(randomx_prefetch_scratchpad);
        # 获取 randomx_prefetch_scratchpad_bmi2 函数的地址
        const uint8_t* b = addr(randomx_prefetch_scratchpad_bmi2);
        # 将 randomx_prefetch_scratchpad 到 randomx_prefetch_scratchpad_bmi2 之间的代码复制到 codePrefetchScratchpadTweaked 数组中
        memcpy(codePrefetchScratchpadTweaked, a, b - a);
        # 设置 codePrefetchScratchpadTweakedSize 的值为复制的代码的大小
        codePrefetchScratchpadTweakedSize = b - a;
    }
#    endif
}

#if (XMRIG_ARM == 8)
# 定义一个静态函数，用于计算一个数的对数，返回结果为 uint32_t 类型
static uint32_t Log2(size_t value) { return (value > 1) ? (Log2(value / 2) + 1) : 0; }
#endif

# 定义一个全局变量 scratchpadPrefetchMode，并初始化为 1
static int scratchpadPrefetchMode = 1;

# 定义一个函数，用于设置 scratchpadPrefetchMode 的值
void randomx_set_scratchpad_prefetch_mode(int mode)
{
    # 将传入的 mode 赋值给 scratchpadPrefetchMode
    scratchpadPrefetchMode = mode;
}

# 定义一个类 RandomX_ConfigurationBase，并实现其 Apply 方法
void RandomX_ConfigurationBase::Apply()
{
    # 计算 ScratchpadL1Mask_Calculated 的值
    const uint32_t ScratchpadL1Mask_Calculated = (ScratchpadL1_Size / sizeof(uint64_t) - 1) * 8;
    # 计算 ScratchpadL2Mask_Calculated 的值
    const uint32_t ScratchpadL2Mask_Calculated = (ScratchpadL2_Size / sizeof(uint64_t) - 1) * 8;

    # 将 ScratchpadL2Mask_Calculated 的值赋给 AddressMask_Calculated[0]
    AddressMask_Calculated[0] = ScratchpadL2Mask_Calculated;
    # 将 ScratchpadL1Mask_Calculated 的值赋给 AddressMask_Calculated[1]
    AddressMask_Calculated[1] = ScratchpadL1Mask_Calculated;
    # 将 ScratchpadL1Mask_Calculated 的值赋给 AddressMask_Calculated[2]
    AddressMask_Calculated[2] = ScratchpadL1Mask_Calculated;
    # 将 ScratchpadL1Mask_Calculated 的值赋给 AddressMask_Calculated[3]
    AddressMask_Calculated[3] = ScratchpadL1Mask_Calculated;

    # 计算 ScratchpadL3Mask_Calculated 的值
    ScratchpadL3Mask_Calculated = (((ScratchpadL3_Size / sizeof(uint64_t)) - 1) * 8);
    # 计算 ScratchpadL3Mask64_Calculated 的值
    ScratchpadL3Mask64_Calculated = ((ScratchpadL3_Size / sizeof(uint64_t)) / 8 - 1) * 64;

    # 如果定义了 XMRIG_FEATURE_ASM，并且定义了 _M_X64 或者 __x86_64__
    if defined(XMRIG_FEATURE_ASM) && (defined(_M_X64) || defined(__x86_64__))
        # 将 ArgonMemory * 16 - 1 的值赋给 codeSshPrefetchTweaked + 3
        *(uint32_t*)(codeSshPrefetchTweaked + 3) = ArgonMemory * 16 - 1;
        # 不需要在此处使用，因为所有变体都使用默认的数据集基础大小
        # const uint32_t DatasetBaseMask = DatasetBaseSize - RANDOMX_DATASET_ITEM_SIZE;
        # *(uint32_t*)(codeReadDatasetTweaked + 9) = DatasetBaseMask;
        # *(uint32_t*)(codeReadDatasetTweaked + 24) = DatasetBaseMask;
        # *(uint32_t*)(codeReadDatasetLightSshInitTweaked + 59) = DatasetBaseMask;

        # 检查 CPU 是否支持 BMI2 指令集，并将结果赋给 hasBMI2
        const bool hasBMI2 = xmrig::Cpu::info()->hasBMI2();

        # 将 ScratchpadL3Mask64_Calculated 的值赋给 codePrefetchScratchpadTweaked + (hasBMI2 ? 7 : 4)
        *(uint32_t*)(codePrefetchScratchpadTweaked + (hasBMI2 ? 7 : 4)) = ScratchpadL3Mask64_Calculated;
        # 将 ScratchpadL3Mask64_Calculated 的值赋给 codePrefetchScratchpadTweaked + (hasBMI2 ? 17 : 18)
        *(uint32_t*)(codePrefetchScratchpadTweaked + (hasBMI2 ? 17 : 18)) = ScratchpadL3Mask64_Calculated;

        # 应用 scratchpadPrefetchMode 的值
    {
        // 定义指向32位无符号整数的指针a，指向codePrefetchScratchpadTweaked数组的偏移量（根据hasBMI2的值选择偏移量）
        uint32_t* a = (uint32_t*)(codePrefetchScratchpadTweaked + (hasBMI2 ? 11 : 8));
        // 定义指向32位无符号整数的指针b，指向codePrefetchScratchpadTweaked数组的偏移量（根据hasBMI2的值选择偏移量）
        uint32_t* b = (uint32_t*)(codePrefetchScratchpadTweaked + (hasBMI2 ? 21 : 22));

        // 根据scratchpadPrefetchMode的值进行不同的操作
        switch (scratchpadPrefetchMode)
        {
        case 0:
            // 如果scratchpadPrefetchMode为0，将a指向的内存设置为4字节nop指令
            *a = 0x00401F0FUL; // 4-byte nop
            // 如果scratchpadPrefetchMode为0，将b指向的内存设置为4字节nop指令
            *b = 0x00401F0FUL; // 4-byte nop
            break;

        case 1:
        default:
            // 如果scratchpadPrefetchMode为1或默认值，将a指向的内存设置为prefetcht0指令
            *a = 0x060C180FUL; // prefetcht0 [rsi+rax]
            // 如果scratchpadPrefetchMode为1或默认值，将b指向的内存设置为prefetcht0指令
            *b = 0x160C180FUL; // prefetcht0 [rsi+rdx]
            break;

        case 2:
            // 如果scratchpadPrefetchMode为2，将a指向的内存设置为prefetchnta指令
            *a = 0x0604180FUL; // prefetchnta [rsi+rax]
            // 如果scratchpadPrefetchMode为2，将b指向的内存设置为prefetchnta指令
            *b = 0x1604180FUL; // prefetchnta [rsi+rdx]
            break;

        case 3:
            // 如果scratchpadPrefetchMode为3，将a指向的内存设置为mov指令
            *a = 0x060C8B48UL; // mov rcx, [rsi+rax]
            // 如果scratchpadPrefetchMode为3，将b指向的内存设置为mov指令
            *b = 0x160C8B48UL; // mov rcx, [rsi+rdx]
            break;
        }
    }
// 定义一个指向 randomx::JitCompilerX86 类中特定函数的指针类型 InstructionGeneratorX86_2
typedef void(randomx::JitCompilerX86::* InstructionGeneratorX86_2)(const randomx::Instruction&);

// 定义宏 JIT_HANDLE，用于处理指令 x，prev 是前一个指令
#define JIT_HANDLE(x, prev) do { \
        // 创建指向 h_##x 函数的指针 p
        const InstructionGeneratorX86_2 p = &randomx::JitCompilerX86::h_##x; \
        // 将指针 p 的内容复制到 randomx::JitCompilerX86::engine + k 处
        memcpy(randomx::JitCompilerX86::engine + k, &p, sizeof(p)); \
    } while (0)

// 如果 XMRIG_ARM 等于 8
#elif (XMRIG_ARM == 8)

    // 计算 Log2_ScratchpadL1 的值
    Log2_ScratchpadL1 = Log2(ScratchpadL1_Size);
    // 计算 Log2_ScratchpadL2 的值
    Log2_ScratchpadL2 = Log2(ScratchpadL2_Size);
    // 计算 Log2_ScratchpadL3 的值
    Log2_ScratchpadL3 = Log2(ScratchpadL3_Size);
    // 计算 Log2_DatasetBaseSize 的值
    Log2_DatasetBaseSize = Log2(DatasetBaseSize);
    // 计算 Log2_CacheSize 的值
    Log2_CacheSize = Log2((ArgonMemory * randomx::ArgonBlockSize) / randomx::CacheLineSize);

// 定义宏 JIT_HANDLE，用于处理指令 x，prev 是前一个指令
#define JIT_HANDLE(x, prev) randomx::JitCompilerA64::engine[k] = &randomx::JitCompilerA64::h_##x

// 如果不满足上述条件
#else
// 定义宏 JIT_HANDLE，不做任何操作
#define JIT_HANDLE(x, prev)
#endif

// 定义变量 k 和 freq_sum
    uint32_t k = 0;
    uint32_t freq_sum = 0;

// 定义宏 INST_HANDLE，用于处理指令 x，prev 是前一个指令
#define INST_HANDLE(x, prev) \
    // 将 RANDOMX_FREQ_##x 加到 freq_sum 上
    freq_sum += RANDOMX_FREQ_##x; \
    // 循环处理指令，直到 k 小于 freq_sum
    for (; k < freq_sum; ++k) { JIT_HANDLE(x, prev); }

// 定义宏 INST_HANDLE2，用于处理指令 x，func_name 是函数名，prev 是前一个指令
#define INST_HANDLE2(x, func_name, prev) \
    // 将 RANDOMX_FREQ_##x 加到 freq_sum 上
    freq_sum += RANDOMX_FREQ_##x; \
    // 循环处理指令，直到 k 小于 freq_sum
    for (; k < freq_sum; ++k) { JIT_HANDLE(func_name, prev); }

// 处理指令 IADD_RS
    INST_HANDLE(IADD_RS, NULL);
// 处理指令 IADD_M
    INST_HANDLE(IADD_M, IADD_RS);
// 处理指令 ISUB_R
    INST_HANDLE(ISUB_R, IADD_M);
// 处理指令 ISUB_M
    INST_HANDLE(ISUB_M, ISUB_R);
// 处理指令 IMUL_R
    INST_HANDLE(IMUL_R, ISUB_M);
// 处理指令 IMUL_M
    INST_HANDLE(IMUL_M, IMUL_R);

// 如果定义了 XMRIG_FEATURE_ASM 并且是 x64 架构
#if defined(XMRIG_FEATURE_ASM) && (defined(_M_X64) || defined(__x86_64__))
    // 如果支持 BMI2 指令集
    if (hasBMI2) {
        // 处理指令 IMULH_R
        INST_HANDLE2(IMULH_R, IMULH_R_BMI2, IMUL_M);
        // 处理指令 IMULH_M
        INST_HANDLE2(IMULH_M, IMULH_M_BMI2, IMULH_R);
    }
    else
#endif
    {
        // 处理指令 IMULH_R
        INST_HANDLE(IMULH_R, IMUL_M);
        // 处理指令 IMULH_M
        INST_HANDLE(IMULH_M, IMULH_R);
    }

// 处理指令 ISMULH_R
    INST_HANDLE(ISMULH_R, IMULH_M);
// 处理指令 ISMULH_M
    INST_HANDLE(ISMULH_M, ISMULH_R);
// 处理指令 IMUL_RCP
    INST_HANDLE(IMUL_RCP, ISMULH_M);
// 处理指令 INEG_R
    INST_HANDLE(INEG_R, IMUL_RCP);
// 处理指令 IXOR_R
    INST_HANDLE(IXOR_R, INEG_R);
// 处理指令 IXOR_M
    INST_HANDLE(IXOR_M, IXOR_R);
// 处理指令 IROR_R
    INST_HANDLE(IROR_R, IXOR_M);
// 处理指令 IROL_R
    INST_HANDLE(IROL_R, IROR_R);
// 处理指令 ISWAP_R
    INST_HANDLE(ISWAP_R, IROL_R);
// 处理指令 FSWAP_R
    INST_HANDLE(FSWAP_R, ISWAP_R);
// 处理指令 FADD_R
    INST_HANDLE(FADD_R, FSWAP_R);
// 处理指令 FADD_M
    INST_HANDLE(FADD_M, FADD_R);
    # 处理指令FSUB_R，执行浮点数减法操作
    INST_HANDLE(FSUB_R, FADD_M);
    
    # 处理指令FSUB_M，执行浮点数减法操作
    INST_HANDLE(FSUB_M, FSUB_R);
    
    # 处理指令FSCAL_R，执行浮点数缩放操作
    INST_HANDLE(FSCAL_R, FSUB_M);
    
    # 处理指令FMUL_R，执行浮点数乘法操作
    INST_HANDLE(FMUL_R, FSCAL_R);
    
    # 处理指令FDIV_M，执行浮点数除法操作
    INST_HANDLE(FDIV_M, FMUL_R);
    
    # 处理指令FSQRT_R，执行浮点数平方根操作
    INST_HANDLE(FSQRT_R, FDIV_M);
#if defined(_M_X64) || defined(__x86_64__)
    # 如果是 64 位系统
    if (xmrig::Cpu::info()->jccErratum()) {
        # 如果 CPU 支持 jccErratum，则处理条件分支指令
        INST_HANDLE2(CBRANCH, CBRANCH<true>, FSQRT_R);
    }
    else {
        # 如果 CPU 不支持 jccErratum，则处理条件分支指令
        INST_HANDLE2(CBRANCH, CBRANCH<false>, FSQRT_R);
    }
#else
    # 如果不是 64 位系统，则处理条件分支指令
    INST_HANDLE(CBRANCH, FSQRT_R);
#endif

#if defined(XMRIG_FEATURE_ASM) && (defined(_M_X64) || defined(__x86_64__))
    # 如果定义了 XMRIG_FEATURE_ASM 并且是 64 位系统
    if (hasBMI2) {
        # 如果 CPU 支持 BMI2 指令集，则处理 CFROUND 指令
        INST_HANDLE2(CFROUND, CFROUND_BMI2, CBRANCH);
    }
    else
#endif
    {
        # 如果不支持 BMI2 指令集，则处理 CFROUND 指令
        INST_HANDLE(CFROUND, CBRANCH);
    }

    # 处理 ISTORE 指令
    INST_HANDLE(ISTORE, CFROUND);
    # 处理 NOP 指令
    INST_HANDLE(NOP, ISTORE);
#undef INST_HANDLE
}

# 定义 Monero 的 RandomX 配置
RandomX_ConfigurationMonero RandomX_MoneroConfig;
# 定义 Wownero 的 RandomX 配置
RandomX_ConfigurationWownero RandomX_WowneroConfig;
# 定义 Arqma 的 RandomX 配置
RandomX_ConfigurationArqma RandomX_ArqmaConfig;
# 定义 Graft 的 RandomX 配置
RandomX_ConfigurationGraft RandomX_GraftConfig;
# 定义 Safex 的 RandomX 配置
RandomX_ConfigurationSafex RandomX_SafexConfig;
# 定义 Keva 的 RandomX 配置
RandomX_ConfigurationKeva RandomX_KevaConfig;

# 定义当前的 RandomX 配置
alignas(64) RandomX_ConfigurationBase RandomX_CurrentConfig;

# 定义虚拟机池的互斥锁
static std::mutex vm_pool_mutex;

# 定义 C 语言的外部函数
extern "C" {
    # 创建一个随机X缓存对象
    def randomx_create_cache(flags, memory):
        # 如果内存为空，则返回空指针
        if not memory:
            return nullptr
    
        # 初始化缓存对象为空指针
        cache = nullptr
    
        try:
            # 创建一个随机X缓存对象
            cache = new randomx_cache()
            # 根据flags的值选择不同的操作
            switch (flags & RANDOMX_FLAG_JIT):
                # 如果flags的值为RANDOMX_FLAG_DEFAULT
                case RANDOMX_FLAG_DEFAULT:
                    # 设置缓存对象的jit属性为空指针
                    cache->jit          = nullptr
                    # 设置缓存对象的initialize属性为randomx::initCache函数
                    cache->initialize   = &randomx::initCache
                    # 设置缓存对象的datasetInit属性为randomx::initDataset函数
                    cache->datasetInit  = &randomx::initDataset
                    # 设置缓存对象的memory属性为传入的memory参数
                    cache->memory       = memory
                    break
    
                # 如果flags的值为RANDOMX_FLAG_JIT
                case RANDOMX_FLAG_JIT:
                    # 创建一个JitCompiler对象，并将其赋值给缓存对象的jit属性
                    cache->jit          = new randomx::JitCompiler(false, true)
                    # 设置缓存对象的initialize属性为randomx::initCacheCompile函数
                    cache->initialize   = &randomx::initCacheCompile
                    # 设置缓存对象的datasetInit属性为空指针
                    cache->datasetInit  = nullptr
                    # 设置缓存对象的memory属性为传入的memory参数
                    cache->memory       = memory
                    break
    
                # 如果flags的值不是RANDOMX_FLAG_DEFAULT也不是RANDOMX_FLAG_JIT
                default:
                    # 不可达到的代码
                    UNREACHABLE
        # 捕获异常
        except (std::exception &ex):
            # 如果缓存对象不为空指针
            if cache != nullptr:
                # 释放缓存对象
                randomx_release_cache(cache)
                # 将缓存对象置为空指针
    
        # 返回缓存对象
        return cache
    
    # 初始化缓存对象
    def randomx_init_cache(cache, key, keySize):
        # 断言缓存对象不为空指针
        assert(cache != nullptr)
        # 断言keySize为0或者key不为空指针
        assert(keySize == 0 or key != nullptr)
        # 调用缓存对象的initialize方法，传入key和keySize参数
        cache->initialize(cache, key, keySize)
    
    # 释放缓存对象
    def randomx_release_cache(cache):
        # 释放缓存对象的jit属性
        delete cache->jit
        # 释放缓存对象
        delete cache
    
    # 创建一个随机X数据集对象
    def randomx_create_dataset(memory):
        # 如果内存为空，则返回空指针
        if not memory:
            return nullptr
    
        # 创建一个随机X数据集对象
        dataset = new randomx_dataset()
        # 设置数据集对象的memory属性为传入的memory参数
    
        # 返回数据集对象
        return dataset
    
    # 定义数据集项的数量
    #define DatasetItemCount ((RandomX_CurrentConfig.DatasetBaseSize + RandomX_CurrentConfig.DatasetExtraSize) / RANDOMX_DATASET_ITEM_SIZE)
    
    # 获取数据集项的数量
    def randomx_dataset_item_count():
        # 返回数据集项的数量
        return DatasetItemCount
    # 初始化数据集，将缓存中的数据复制到数据集中的指定位置
    void randomx_init_dataset(randomx_dataset *dataset, randomx_cache *cache, unsigned long startItem, unsigned long itemCount) {
        # 断言数据集和缓存不为空
        assert(dataset != nullptr);
        assert(cache != nullptr);
        # 断言起始项小于数据集项数，且项数小于等于数据集项数
        assert(startItem < DatasetItemCount && itemCount <= DatasetItemCount);
        # 断言起始项加项数小于等于数据集项数
        assert(startItem + itemCount <= DatasetItemCount);
        # 初始化缓存中的数据到数据集中的指定位置
        cache->datasetInit(cache, dataset->memory + startItem * randomx::CacheLineSize, startItem, startItem + itemCount);
    }
    
    # 获取数据集的内存指针
    void *randomx_get_dataset_memory(randomx_dataset *dataset) {
        # 断言数据集不为空
        assert(dataset != nullptr);
        # 返回数据集的内存指针
        return dataset->memory;
    }
    
    # 释放数据集的内存
    void randomx_release_dataset(randomx_dataset *dataset) {
        # 删除数据集对象
        delete dataset;
    }
    
    # 设置虚拟机的缓存
    void randomx_vm_set_cache(randomx_vm *machine, randomx_cache* cache) {
        # 断言虚拟机和缓存不为空，且缓存已初始化
        assert(machine != nullptr);
        assert(cache != nullptr && cache->isInitialized());
        # 设置虚拟机的缓存
        machine->setCache(cache);
    }
    
    # 设置虚拟机的数据集
    void randomx_vm_set_dataset(randomx_vm *machine, randomx_dataset *dataset) {
        # 断言虚拟机和数据集不为空
        assert(machine != nullptr);
        assert(dataset != nullptr);
        # 设置虚拟机的数据集
        machine->setDataset(dataset);
    }
    
    # 销毁虚拟机对象
    void randomx_destroy_vm(randomx_vm* vm) {
        # 调用虚拟机的析构函数
        vm->~randomx_vm();
    }
    
    # 计算哈希值
    void randomx_calculate_hash(randomx_vm *machine, const void *input, size_t inputSize, void *output) {
        # 断言虚拟机不为空，输入大小为0或输入不为空，输出不为空
        assert(machine != nullptr);
        assert(inputSize == 0 || input != nullptr);
        assert(output != nullptr);
        # 创建临时哈希数组
        alignas(16) uint64_t tempHash[8];
        # 运行 BLAKE2b 哈希算法，将结果存储在临时哈希数组中
        rx_blake2b_wrapper::run(tempHash, sizeof(tempHash), input, inputSize);
        # 初始化虚拟机的缓存
        machine->initScratchpad(&tempHash);
        # 重置虚拟机的舍入模式
        machine->resetRoundingMode();
        # 循环运行虚拟机，更新临时哈希数组
        for (uint32_t chain = 0; chain < RandomX_CurrentConfig.ProgramCount - 1; ++chain) {
            machine->run(&tempHash);
            rx_blake2b_wrapper::run(tempHash, sizeof(tempHash), machine->getRegisterFile(), sizeof(randomx::RegisterFile));
        }
        # 运行虚拟机，获取最终结果
        machine->run(&tempHash);
        machine->getFinalResult(output);
    }
    // 计算第一个输入的哈希值并初始化随机X虚拟机的内存
    void randomx_calculate_hash_first(randomx_vm* machine, uint64_t (&tempHash)[8], const void* input, size_t inputSize) {
        // 使用BLAKE2b算法计算输入的哈希值
        rx_blake2b_wrapper::run(tempHash, sizeof(tempHash), input, inputSize);
        // 初始化随机X虚拟机的内存
        machine->initScratchpad(tempHash);
    }
    
    // 计算下一个输入的哈希值
    void randomx_calculate_hash_next(randomx_vm* machine, uint64_t (&tempHash)[8], const void* nextInput, size_t nextInputSize, void* output) {
        // 用于性能分析的作用域
        PROFILE_SCOPE(RandomX_hash);
    
        // 重置舍入模式
        machine->resetRoundingMode();
        // 循环执行随机X虚拟机的程序
        for (uint32_t chain = 0; chain < RandomX_CurrentConfig.ProgramCount - 1; ++chain) {
            machine->run(&tempHash);
            // 使用BLAKE2b算法计算哈希值
            rx_blake2b_wrapper::run(tempHash, sizeof(tempHash), machine->getRegisterFile(), sizeof(randomx::RegisterFile));
        }
        machine->run(&tempHash);
    
        // 完成当前哈希计算并填充下一个哈希计算所需的内存
        rx_blake2b_wrapper::run(tempHash, sizeof(tempHash), nextInput, nextInputSize);
        machine->hashAndFill(output, tempHash);
    }
# 闭合前面的函数定义
```