# `xmrig\src\crypto\randomx\jit_compiler_x86.cpp`

```
# 版权声明
# 本软件的版权归版权持有人和贡献者所有
# 在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
# * 必须保留上述版权声明、此条件列表和以下免责声明。
# * 在二进制形式的重新分发中，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
# * 未经特定事先书面许可，不得使用版权持有人或贡献者的名称来认可或推广从本软件派生的产品。
# 
# 本软件按“原样”提供，不提供任何明示或暗示的保证，包括但不限于适销性和适用性的暗示保证。
# 在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他）。
# 即使在使用本软件的可能性已被告知的情况下，也不承担任何责任。
# 
# 引入所需的库和头文件
#include <stdexcept>  # 引入标准异常类的头文件
#include <cstring>  # 引入字符串操作函数的头文件
#include <climits>  # 引入整数类型的最大值和最小值的宏定义的头文件
#include <atomic>  # 引入原子操作的头文件
#include "crypto/randomx/jit_compiler_x86.hpp"  # 引入 JIT 编译器的头文件
#include "backend/cpu/Cpu.h"  # 引入 CPU 的头文件
#include "crypto/common/VirtualMemory.h"  # 引入虚拟内存的头文件
#include "crypto/randomx/jit_compiler_x86_static.hpp"  # 引入静态 JIT 编译器的头文件
#include "crypto/randomx/program.hpp"  # 引入程序的头文件
#include "crypto/randomx/reciprocal.h"
#include "crypto/randomx/superscalar.hpp"
#include "crypto/randomx/virtual_memory.hpp"
#include "crypto/rx/Profiler.h"

#ifdef XMRIG_FIX_RYZEN
#   include "crypto/rx/RxFix.h"
#endif

#ifdef _MSC_VER
#   include <intrin.h>
#endif

// 定义静态变量 hugePagesJIT，用于标识是否启用大页 JIT
static bool hugePagesJIT = false;
// 定义静态变量 optimizedDatasetInit，用于存储优化数据集初始化的值
static int optimizedDatasetInit = -1;

// 设置是否启用大页 JIT
void randomx_set_huge_pages_jit(bool hugePages)
{
    hugePagesJIT = hugePages;
}

// 设置优化数据集初始化的值
void randomx_set_optimized_dataset_init(int value)
{
    optimizedDatasetInit = value;
}

namespace randomx {
    /*

    REGISTER ALLOCATION:

    ; rax -> temporary
    ; rbx -> iteration counter "ic"
    ; rcx -> temporary
    ; rdx -> temporary
    ; rsi -> scratchpad pointer
    ; rdi -> dataset pointer
    ; rbp -> memory registers "ma" (high 32 bits), "mx" (low 32 bits)
    ; rsp -> stack pointer
    ; r8  -> "r0"
    ; r9  -> "r1"
    ; r10 -> "r2"
    ; r11 -> "r3"
    ; r12 -> "r4"
    ; r13 -> "r5"
    ; r14 -> "r6"
    ; r15 -> "r7"
    ; xmm0 -> "f0"
    ; xmm1 -> "f1"
    ; xmm2 -> "f2"
    ; xmm3 -> "f3"
    ; xmm4 -> "e0"
    ; xmm5 -> "e1"
    ; xmm6 -> "e2"
    ; xmm7 -> "e3"
    ; xmm8 -> "a0"
    ; xmm9 -> "a1"
    ; xmm10 -> "a2"
    ; xmm11 -> "a3"
    ; xmm12 -> temporary
    ; xmm13 -> E 'and' mask = 0x00ffffffffffffff00ffffffffffffff
    ; xmm14 -> E 'or' mask  = 0x3*00000000******3*00000000******
    ; xmm15 -> scale mask   = 0x81f000000000000081f0000000000000

    */

#    if defined(_MSC_VER) && (defined(_DEBUG) || defined (RELWITHDEBINFO))
    #define ADDR(x) ((((uint8_t*)&x)[0] == 0xE9) ? (((uint8_t*)&x) + *(const int32_t*)(((uint8_t*)&x) + 1) + 5) : ((uint8_t*)&x))
#    else
    #define ADDR(x) ((uint8_t*)&x)
#    endif

    // 定义宏，用于获取指定函数的地址
    #define codePrologue ADDR(randomx_program_prologue)
    #define codeLoopBegin ADDR(randomx_program_loop_begin)
    #define codeLoopLoad ADDR(randomx_program_loop_load)
    #define codeLoopLoadXOP ADDR(randomx_program_loop_load_xop)
    #define codeProgramStart ADDR(randomx_program_start)
    # 定义代码中各个函数的地址
    #define codeReadDataset ADDR(randomx_program_read_dataset)
    #define codeReadDatasetLightSshInit ADDR(randomx_program_read_dataset_sshash_init)
    #define codeReadDatasetLightSshFin ADDR(randomx_program_read_dataset_sshash_fin)
    #define codeDatasetInit ADDR(randomx_dataset_init)
    #define codeDatasetInitAVX2Prologue ADDR(randomx_dataset_init_avx2_prologue)
    #define codeDatasetInitAVX2LoopEnd ADDR(randomx_dataset_init_avx2_loop_end)
    #define codeDatasetInitAVX2Epilogue ADDR(randomx_dataset_init_avx2_epilogue)
    #define codeDatasetInitAVX2SshLoad ADDR(randomx_dataset_init_avx2_ssh_load)
    #define codeDatasetInitAVX2SshPrefetch ADDR(randomx_dataset_init_avx2_ssh_prefetch)
    #define codeLoopStore ADDR(randomx_program_loop_store)
    #define codeLoopEnd ADDR(randomx_program_loop_end)
    #define codeEpilogue ADDR(randomx_program_epilogue)
    #define codeProgramEnd ADDR(randomx_program_end)
    #define codeSshLoad ADDR(randomx_sshash_load)
    #define codeSshPrefetch ADDR(randomx_sshash_prefetch)
    #define codeSshEnd ADDR(randomx_sshash_end)
    #define codeSshInit ADDR(randomx_sshash_init)
    
    # 定义各个代码段的大小
    #define prologueSize (codeLoopBegin - codePrologue)
    #define loopLoadSize (codeLoopLoadXOP - codeLoopLoad)
    #define loopLoadXOPSize (codeProgramStart - codeLoopLoadXOP)
    #define readDatasetSize (codeReadDatasetLightSshInit - codeReadDataset)
    #define readDatasetLightInitSize (codeReadDatasetLightSshFin - codeReadDatasetLightSshInit)
    #define readDatasetLightFinSize (codeLoopStore - codeReadDatasetLightSshFin)
    #define loopStoreSize (codeLoopEnd - codeLoopStore)
    #define datasetInitSize (codeDatasetInitAVX2Prologue - codeDatasetInit)
    #define datasetInitAVX2PrologueSize (codeDatasetInitAVX2LoopEnd - codeDatasetInitAVX2Prologue)
    #define datasetInitAVX2LoopEndSize (codeDatasetInitAVX2Epilogue - codeDatasetInitAVX2LoopEnd)
    #define datasetInitAVX2EpilogueSize (codeDatasetInitAVX2SshLoad - codeDatasetInitAVX2Epilogue)
    # 定义datasetInitAVX2SshLoadSize变量，表示codeDatasetInitAVX2SshPrefetch和codeDatasetInitAVX2SshLoad之间的大小差
    # 定义datasetInitAVX2SshPrefetchSize变量，表示codeEpilogue和codeDatasetInitAVX2SshPrefetch之间的大小差
    # 定义epilogueSize变量，表示codeSshLoad和codeEpilogue之间的大小差
    # 定义codeSshLoadSize变量，表示codeSshPrefetch和codeSshLoad之间的大小差
    # 定义codeSshPrefetchSize变量，表示codeSshEnd和codeSshPrefetch之间的大小差
    # 定义codeSshInitSize变量，表示codeProgramEnd和codeSshInit之间的大小差
    # 定义epilogueOffset变量，表示CodeSize减去epilogueSize再按位与上63的结果
    # 定义superScalarHashOffset变量，值为32768
    # 定义NOP1-NOP9变量，分别表示长度为1-9的NOP指令
    # 定义NOPX变量，为NOP1-NOP9的数组
    # 定义NOP13-NOP26变量，分别表示长度为13-26的NOP指令
    # 定义一个静态常量数组，用于对齐跳转指令的前缀
    static const uint8_t JMP_ALIGN_PREFIX[14][16] = {
        {},
        {0x2E},
        {0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x90, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x66, 0x90, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x66, 0x66, 0x90, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x0F, 0x1F, 0x40, 0x00, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
        {0x0F, 0x1F, 0x44, 0x00, 0x00, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E, 0x2E},
    };
    
    # 定义一个内联函数，用于将指针对齐到页边界
    static inline uint8_t* alignToPage(uint8_t* p, size_t pageSize) {
        size_t k = (size_t) p;
        k -= k % pageSize;
        return (uint8_t*) k;
    }
    
    # 定义一个函数，用于获取代码的大小
    size_t JitCompilerX86::getCodeSize() {
        # 如果代码位置小于前导部分的大小，则返回0，否则返回代码位置减去前导部分的大小
        return codePos < prologueSize ? 0 : codePos - prologueSize;
    }
    
    # 定义一个函数，用于将代码段设置为可写
    void JitCompilerX86::enableWriting() const {
        # 将代码指针对齐到页边界
        uint8_t* p1 = alignToPage(code, 4096);
        # 计算代码结束位置
        uint8_t* p2 = code + CodeSize;
        # 将代码段的内存权限设置为可读写
        xmrig::VirtualMemory::protectRW(p1, p2 - p1);
    }
    
    # 定义一个函数，用于将代码段设置为可执行
    void JitCompilerX86::enableExecution() const {
        # 将代码指针对齐到页边界
        uint8_t* p1 = alignToPage(code, 4096);
        # 计算代码结束位置
        uint8_t* p2 = code + CodeSize;
        # 将代码段的内存权限设置为可读可执行
        xmrig::VirtualMemory::protectRX(p1, p2 - p1);
    }
#    ifdef _MSC_VER
    # 如果是在 Visual Studio 编译环境下
    static FORCE_INLINE uint32_t rotl32(uint32_t a, int shift) { return _rotl(a, shift); }
    # 定义一个静态内联函数，用于将 32 位整数左循环移位
#    else
    # 如果不是在 Visual Studio 编译环境下
    static FORCE_INLINE uint32_t rotl32(uint32_t a, int shift) { return (a << shift) | (a >> (-shift & 31)); }
    # 定义一个静态内联函数，用于将 32 位整数左循环移位
#    endif

    static std::atomic<size_t> codeOffset;
    # 定义一个静态原子类型变量 codeOffset，用于存储偏移量
    constexpr size_t codeOffsetIncrement = 59 * 64;
    # 定义一个常量 codeOffsetIncrement，表示偏移量的增量

#            ifdef XMRIG_SECURE_JIT
            false
#            else
            hugePagesJIT && hugePagesEnable
#            endif
        ));
        # 如果不是在安全 JIT 模式下，判断是否启用大页内存，并将结果作为参数传递
        // Shift code base address to improve caching - all threads will use different L2/L3 cache sets
        # 将代码基地址进行偏移，以改善缓存效果 - 所有线程将使用不同的 L2/L3 缓存集
        code = allocatedCode + (codeOffset.fetch_add(codeOffsetIncrement) % CodeSize);
        # 计算出代码的地址，并将其存储在变量 code 中
        memcpy(code, codePrologue, prologueSize);
        # 将代码前言部分复制到代码地址中
        if (hasXOP) {
            # 如果支持 XOP 指令集
            memcpy(code + prologueSize, codeLoopLoadXOP, loopLoadXOPSize);
            # 将 XOP 循环加载代码复制到代码地址中
        }
        else {
            # 如果不支持 XOP 指令集
            memcpy(code + prologueSize, codeLoopLoad, loopLoadSize);
            # 将非 XOP 循环加载代码复制到代码地址中
        }
        memcpy(code + epilogueOffset, codeEpilogue, epilogueSize);
        # 将代码尾部部分复制到代码地址中

        codePosFirst = prologueSize + (hasXOP ? loopLoadXOPSize : loopLoadSize);
        # 计算出代码的第一个位置，并将其存储在变量 codePosFirst 中

#        ifdef XMRIG_FIX_RYZEN
        mainLoopBounds.first = code + prologueSize;
        mainLoopBounds.second = code + epilogueOffset;
#        endif
        # 如果定义了 XMRIG_FIX_RYZEN 宏，则设置主循环边界的起始和结束位置

    }

    JitCompilerX86::~JitCompilerX86() {
        codeOffset.fetch_sub(codeOffsetIncrement);
        # 减去偏移量的增量
        freePagedMemory(allocatedCode, allocatedSize);
        # 释放分配的内存空间
    }

    template<size_t N>
    static FORCE_INLINE void prefetch_data(const void* data) {
        rx_prefetch_nta(data);
        prefetch_data<N - 1>(reinterpret_cast<const char*>(data) + 64);
    }
    # 定义一个模板函数，用于预取数据

    template<> FORCE_INLINE void prefetch_data<0>(const void*) {}
    # 模板函数的特化，当 N 为 0 时的处理方式

    template<typename T> static FORCE_INLINE void prefetch_data(const T& data) { prefetch_data<(sizeof(T) + 63) / 64>(&data); }
    # 定义一个模板函数，用于预取数据

    void JitCompilerX86::prepare() {
        prefetch_data(engine);
        # 预取 engine 变量的数据
        prefetch_data(RandomX_CurrentConfig);
        # 预取 RandomX_CurrentConfig 变量的数据
    }
    # 准备函数，用于预取数据
    # 生成程序的JIT编译器函数
    # 参数：prog - 程序对象，pcfg - 程序配置对象，flags - 标志位
    # 作用：用于生成给定程序的机器码，以便后续执行
// 如果定义了 XMRIG_SECURE_JIT，则启用写入操作
#ifdef XMRIG_SECURE_JIT
    enableWriting();
#endif

// 将标志位赋值给虚拟机标志
vm_flags = flags;

// 生成程序的前导部分
generateProgramPrologue(prog, pcfg);

// 发出读取数据集的指令，将数据集的大小、代码和代码位置作为参数
emit(codeReadDataset, readDatasetSize, code, codePos);

// 生成程序的结尾部分
generateProgramEpilogue(prog, pcfg);
}

// 生成轻量级程序的函数
void JitCompilerX86::generateProgramLight(Program& prog, ProgramConfiguration& pcfg, uint32_t datasetOffset) {
    // 生成程序的前导部分
    generateProgramPrologue(prog, pcfg);

    // 发出读取轻量级数据集初始化的指令，将初始化大小、代码和代码位置作为参数
    emit(codeReadDatasetLightSshInit, readDatasetLightInitSize, code, codePos);

    // 将值 0xc381 写入代码数组中的指定位置
    *(uint32_t*)(code + codePos) = 0xc381;
    codePos += 2;

    // 发出 32 位数据集偏移量除以 CacheLineSize 的指令
    emit32(datasetOffset / CacheLineSize, code, codePos);

    // 发出字节为 0xe8 的指令
    emitByte(0xe8, code, codePos);

    // 发出 32 位超标量哈希偏移量减去 (代码位置 + 4) 的指令
    emit32(superScalarHashOffset - (codePos + 4), code, codePos);

    // 发出读取轻量级数据集结束的指令，将结束大小、代码和代码位置作为参数
    emit(codeReadDatasetLightSshFin, readDatasetLightFinSize, code, codePos);

    // 生成程序的结尾部分
    generateProgramEpilogue(prog, pcfg);
}

// 生成超标量哈希的模板函数
template<size_t N>
}

// 生成超标量哈希的函数
template
void JitCompilerX86::generateSuperscalarHash(SuperscalarProgram(&programs)[RANDOMX_CACHE_MAX_ACCESSES]);

// 生成数据集初始化代码的函数
void JitCompilerX86::generateDatasetInitCode() {
    // 如果未初始化 AVX2，则将代码数组初始化为数据集初始化代码数组
    if (!initDatasetAVX2) {
        memcpy(code, codeDatasetInit, datasetInitSize);
    }
}

// 生成程序的前导部分
void JitCompilerX86::generateProgramPrologue(Program& prog, ProgramConfiguration& pcfg) {
    // 将代码位置设置为 randomx_program_prologue_first_load 与 randomx_program_prologue 的地址差
    codePos = ADDR(randomx_program_prologue_first_load) - ADDR(randomx_program_prologue);

    // 将 ScratchpadL3Mask64_Calculated 的值写入代码数组中指定位置
    *(uint32_t*)(code + codePos + 4) = RandomX_CurrentConfig.ScratchpadL3Mask64_Calculated;
    *(uint32_t*)(code + codePos + 14) = RandomX_CurrentConfig.ScratchpadL3Mask64_Calculated;

    // 如果支持 AVX，则将指定位置的值进行位操作
    if (hasAVX) {
        uint32_t* p = (uint32_t*)(code + codePos + 61);
        *p = (*p & 0xFF000000U) | 0x0077F8C5U; // vzeroupper
    }

    // 如果定义了 XMRIG_FIX_RYZEN，则设置主循环边界
#ifdef XMRIG_FIX_RYZEN
    xmrig::RxFix::setMainLoopBounds(mainLoopBounds);
# 定义 imul_rcp_storage 变量，其值为 code 加上 randomx_program_imul_rcp_store 和 codePrologue 的差值再加 2
imul_rcp_storage = code + (ADDR(randomx_program_imul_rcp_store) - codePrologue) + 2
# 初始化 imul_rcp_storage_used 变量为 0
imul_rcp_storage_used = 0

# 将 pcfg.eMask 的值拷贝到 imul_rcp_storage - 34 的位置
memcpy(imul_rcp_storage - 34, &pcfg.eMask, sizeof(pcfg.eMask))
# 将 codePosFirst 的值赋给 codePos
codePos = codePosFirst
# 初始化 prevCFROUND 变量为 -1
prevCFROUND = -1
# 初始化 prevFPOperation 变量为 -1
prevFPOperation = -1

# 将 registerUsage 数组中的所有元素都标记为已使用
uint64_t* r = (uint64_t*)registerUsage
uint64_t k = codePos
k |= k << 32
for (unsigned j = 0; j < RegistersCount / 2; ++j) {
    r[j] = k
}

# 遍历 RandomX_CurrentConfig.ProgramSize 次，每次处理四条指令
for (int i = 0, n = static_cast<int>(RandomX_CurrentConfig.ProgramSize); i < n; i += 4) {
    # 获取第 i 条指令、第 i+1 条指令、第 i+2 条指令和第 i+3 条指令
    Instruction& instr1 = prog(i)
    Instruction& instr2 = prog(i + 1)
    Instruction& instr3 = prog(i + 2)
    Instruction& instr4 = prog(i + 3)

    # 根据指令的 opcode 获取对应的指令生成器
    InstructionGeneratorX86 gen1 = engine[instr1.opcode]
    InstructionGeneratorX86 gen2 = engine[instr2.opcode]
    InstructionGeneratorX86 gen3 = engine[instr3.opcode]
    InstructionGeneratorX86 gen4 = engine[instr4.opcode]

    # 调用指令生成器生成指令的机器码
    (*gen1)(this, instr1)
    (*gen2)(this, instr2)
    (*gen3)(this, instr3)
    (*gen4)(this, instr4)
}

# 将 0xc03341c08b41ull 加上 pcfg.readReg2 左移 16 位的结果加上 pcfg.readReg3 左移 40 位的结果赋给 code + codePos 的位置
*(uint64_t*)(code + codePos) = 0xc03341c08b41ull + (static_cast<uint64_t>(pcfg.readReg2) << 16) + (static_cast<uint64_t>(pcfg.readReg3) << 40)
# 将 codePos 的值增加 6
codePos += 6
    # 生成程序的结尾部分
    void JitCompilerX86::generateProgramEpilogue(Program& prog, ProgramConfiguration& pcfg) {
        # 在代码数组中写入一个64位整数，该整数的值为0xc03349c08b49加上pcfg.readReg0和pcfg.readReg1的值的组合
        *(uint64_t*)(code + codePos) = 0xc03349c08b49ull + (static_cast<uint64_t>(pcfg.readReg0) << 16) + (static_cast<uint64_t>(pcfg.readReg1) << 40);
        # 将代码指针向后移动6个字节
        codePos += 6;
        # 将RandomX_CurrentConfig.codePrefetchScratchpadTweaked的内容复制到代码数组中
        emit(RandomX_CurrentConfig.codePrefetchScratchpadTweaked, RandomX_CurrentConfig.codePrefetchScratchpadTweakedSize, code, codePos);
        # 将codeLoopStore的内容复制到代码数组中
        memcpy(code + codePos, codeLoopStore, loopStoreSize);
        # 将代码指针向后移动loopStoreSize个字节
        codePos += loopStoreSize;
    
        # 如果BranchesWithin32B为真
        if (BranchesWithin32B) {
            # 将branch_begin设置为codePos的低32位
            const uint32_t branch_begin = static_cast<uint32_t>(codePos);
            # 将branch_end设置为branch_begin加上9
            const uint32_t branch_end = static_cast<uint32_t>(branch_begin + 9);
    
            # 如果跳转跨越或触及32字节边界，对齐它
            if ((branch_begin ^ branch_end) >= 32) {
                # 计算对齐大小
                uint32_t alignment_size = 32 - (branch_begin & 31);
                # 如果对齐大小大于8，将NOPX[alignment_size - 9]的内容复制到代码数组中，并将代码指针向后移动alignment_size - 8个字节
                if (alignment_size > 8) {
                    emit(NOPX[alignment_size - 9], alignment_size - 8, code, codePos);
                    alignment_size = 8;
                }
                # 将NOPX[alignment_size - 1]的内容复制到代码数组中，并将代码指针向后移动alignment_size个字节
                emit(NOPX[alignment_size - 1], alignment_size, code, codePos);
            }
        }
    
        # 在代码数组中写入一个64位整数，该整数的值为0x850f01eb83
        *(uint64_t*)(code + codePos) = 0x850f01eb83ull;
        # 将代码指针向后移动5个字节
        codePos += 5;
        # 在代码数组中写入一个32位整数，该整数的值为prologueSize - codePos - 4
        emit32(prologueSize - codePos - 4, code, codePos);
        # 在代码数组中写入一个字节，该字节的值为0xe9
        emitByte(0xe9, code, codePos);
        # 在代码数组中写入一个32位整数，该整数的值为epilogueOffset - codePos - 4
        emit32(epilogueOffset - codePos - 4, code, codePos);
    }
    
    # 生成超标量代码
    template<bool AVX2>
    }
    
    # 实例化generateSuperscalarCode函数模板
    template void JitCompilerX86::generateSuperscalarCode<false>(Instruction&, uint8_t*, uint32_t&);
    template void JitCompilerX86::generateSuperscalarCode<true>(Instruction&, uint8_t*, uint32_t&);
    
    # 实例化generateSuperscalarCode函数模板
    template<bool rax>
    // 生成用于寄存器地址的代码，根据指令和源操作数生成代码
    FORCE_INLINE void JitCompilerX86::genAddressReg(const Instruction& instr, const uint32_t src, uint8_t* code, uint32_t& codePos) {
        // 将指令的源操作数左移 16 位，然后根据 rax 寄存器的状态选择不同的操作码，并写入代码中
        *(uint32_t*)(code + codePos) = (rax ? 0x24808d41 : 0x24888d41) + (src << 16);
    
        // 根据 RegisterNeedsSib 的值计算 add_table，然后根据 src 的值更新 codePos
        constexpr uint32_t add_table = 0x33333333u + (1u << (RegisterNeedsSib * 4));
        codePos += (add_table >> (src * 4)) & 0xf;
    
        // 发射 32 位立即数到代码中
        emit32(instr.getImm32(), code, codePos);
        // 如果 rax 寄存器被使用，则发射 0x25 到代码中，否则发射 0xe181 到代码中
        if (rax) {
            emitByte(0x25, code, codePos);
        }
        else {
            *(uint32_t*)(code + codePos) = 0xe181;
            codePos += 2;
        }
        // 根据 instr 的 modMem 值发射地址掩码到代码中
        emit32(AddressMask[instr.getModMem()], code, codePos);
    }
    
    // 生成用于寄存器地址的代码，根据指令和源操作数生成代码的模板实例化
    template void JitCompilerX86::genAddressReg<false>(const Instruction& instr, const uint32_t src, uint8_t* code, uint32_t& codePos);
    template void JitCompilerX86::genAddressReg<true>(const Instruction& instr, const uint32_t src, uint8_t* code, uint32_t& codePos);
    
    // 生成用于目的寄存器地址的代码，根据指令生成代码
    FORCE_INLINE void JitCompilerX86::genAddressRegDst(const Instruction& instr, uint8_t* code, uint32_t& codePos) {
        // 将指令的目的操作数左移 16 位，然后写入代码中
        const uint32_t dst = static_cast<uint32_t>(instr.dst) << 16;
        *(uint32_t*)(code + codePos) = 0x24808d41 + dst;
        // 根据 dst 的值更新 codePos
        codePos += (dst == (RegisterNeedsSib << 16)) ? 4 : 3;
    
        // 发射 32 位立即数到代码中
        emit32(instr.getImm32(), code, codePos);
        // 发射 0x25 到代码中
        emitByte(0x25, code, codePos);
    
        // 根据 instr 的 modMem 值选择地址掩码，然后发射到代码中
        const uint32_t mask1 = AddressMask[instr.getModMem()];
        const uint32_t mask2 = ScratchpadL3Mask;
        emit32((instr.mod < (StoreL3Condition << 4)) ? mask1 : mask2, code, codePos);
    }
    
    // 生成用于立即数地址的代码，根据指令生成代码
    FORCE_INLINE void JitCompilerX86::genAddressImm(const Instruction& instr, uint8_t* code, uint32_t& codePos) {
        // 发射 instr 的 32 位立即数和 ScratchpadL3Mask 的按位与结果到代码中
        emit32(instr.getImm32() & ScratchpadL3Mask, code, codePos);
    }
    // 执行整数加法指令，将两个寄存器或内存中的值相加，并将结果存储在目标寄存器中
    void JitCompilerX86::h_IADD_RS(const Instruction& instr) {
        // 获取当前代码位置
        uint32_t pos = codePos;
        // 获取代码指针
        uint8_t* const p = code + pos;
    
        // 获取目标寄存器和 SIB 字节
        const uint32_t dst = instr.dst;
        const uint32_t sib = (instr.getModShift() << 6) | (instr.src << 3) | dst;
    
        // 计算指令的操作码
        uint32_t k = 0x048d4f + (dst << 19);
        if (dst == RegisterNeedsDisplacement)
            k = 0xac8d4f;
    
        // 将操作码和 SIB 字节写入代码
        *(uint32_t*)(p) = k | (sib << 24);
        *(uint32_t*)(p + 4) = instr.getImm32();
    
        // 更新代码位置
        pos += ((dst == RegisterNeedsDisplacement) ? 8 : 4);
    
        // 更新目标寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    
    // 执行整数加法指令，将寄存器或内存中的值与目标寄存器中的值相加，并将结果存储在目标寄存器中
    void JitCompilerX86::h_IADD_M(const Instruction& instr) {
        // 获取代码指针
        uint8_t* const p = code;
        // 获取当前代码位置
        uint32_t pos = codePos;
    
        // 获取源操作数和目标寄存器
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;
    
        // 如果源操作数和目标寄存器不同
        if (src != dst) {
            // 生成地址寄存器
            genAddressReg<true>(instr, src, p, pos);
            // 发出指令
            emit32(0x0604034c + (dst << 19), p, pos);
        }
        else {
            // 将指令写入代码
            *(uint32_t*)(p + pos) = 0x86034c + (dst << 19);
            pos += 3;
            // 生成立即数
            genAddressImm(instr, p, pos);
        }
    
        // 更新目标寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    
    // 执行整数减法指令，将源寄存器或内存中的值从目标寄存器中的值中减去
    void JitCompilerX86::h_ISUB_R(const Instruction& instr) {
        // 获取代码指针
        uint8_t* const p = code;
        // 获取当前代码位置
        uint32_t pos = codePos;
        
        // 获取源操作数和目标寄存器
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;
    
        // 如果源操作数和目标寄存器不同
        if (src != dst) {
            // 将指令写入代码
            *(uint32_t*)(p + pos) = 0xc02b4d + (dst << 19) + (src << 16);
            pos += 3;
        }
        else {
            // 将指令写入代码
            *(uint32_t*)(p + pos) = 0xe88149 + (dst << 16);
            pos += 3;
            // 发出立即数
            emit32(instr.getImm32(), p, pos);
        }
    
        // 更新目标寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    // 对应指令 ISUB_M 的汇编代码生成函数
    void JitCompilerX86::h_ISUB_M(const Instruction& instr) {
        // 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        // 获取源操作数和目的操作数
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;
    
        // 如果源操作数和目的操作数不相同
        if (src != dst) {
            // 生成源操作数的地址寄存器
            genAddressReg<true>(instr, src, p, pos);
            // 发出指令 0x06042b4c 加上目的操作数的偏移
            emit32(0x06042b4c + (dst << 19), p, pos);
        }
        else {
            // 在代码指针位置处写入指令 0x862b4c 加上目的操作数的偏移
            *(uint32_t*)(p + pos) = 0x862b4c + (dst << 19);
            pos += 3;
            // 生成立即数地址
            genAddressImm(instr, p, pos);
        }
    
        // 更新目的操作数的寄存器使用情况
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    
    // 对应指令 IMUL_R 的汇编代码生成函数
    void JitCompilerX86::h_IMUL_R(const Instruction& instr) {
        // 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        // 获取源操作数和目的操作数
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;
    
        // 如果源操作数和目的操作数不相同
        if (src != dst) {
            // 发出指令 0xc0af0f4d 加上目的操作数和源操作数的偏移
            emit32(0xc0af0f4d + ((dst * 8 + src) << 24), p, pos);
        }
        else {
            // 在代码指针位置处写入指令 0xc0694d 加上目的操作数的偏移
            *(uint32_t*)(p + pos) = 0xc0694d + (((dst << 3) + dst) << 16);
            pos += 3;
            // 发出立即数指令
            emit32(instr.getImm32(), p, pos);
        }
    
        // 更新目的操作数的寄存器使用情况
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    
    // 对应指令 IMUL_M 的汇编代码生成函数
    void JitCompilerX86::h_IMUL_M(const Instruction& instr) {
        // 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        // 获取源操作数和目的操作数
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;
    
        // 如果源操作数和目的操作数不相同
        if (src != dst) {
            // 生成源操作数的地址寄存器
            genAddressReg<true>(instr, src, p, pos);
            // 在代码指针位置处写入指令 0x0604af0f4c 加上目的操作数的偏移
            *(uint64_t*)(p + pos) = 0x0604af0f4cull + (dst << 27);
            pos += 5;
        }
        else {
            // 发出指令 0x86af0f4c 加上目的操作数的偏移
            emit32(0x86af0f4c + (dst << 27), p, pos);
            // 生成立即数地址
            genAddressImm(instr, p, pos);
        }
    
        // 更新目的操作数的寄存器使用情况
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    // 定义函数 h_IMULH_R，接受一个指令参数
    void JitCompilerX86::h_IMULH_R(const Instruction& instr) {
        // 获取代码指针
        uint8_t* const p = code;
        // 获取代码位置
        uint32_t pos = codePos;

        // 获取指令中的源操作数和目的操作数
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;

        // 在代码中写入指令操作码和目的寄存器
        *(uint32_t*)(p + pos) = 0xc08b49 + (dst << 16);
        // 在代码中写入指令操作码和源寄存器
        *(uint32_t*)(p + pos + 3) = 0xe0f749 + (src << 16);
        // 在代码中写入指令操作码和目的寄存器
        *(uint32_t*)(p + pos + 6) = 0xc28b4c + (dst << 19);
        // 更新代码位置
        pos += 9;

        // 更新目的寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }

    // 定义函数 h_IMULH_R_BMI2，接受一个指令参数
    void JitCompilerX86::h_IMULH_R_BMI2(const Instruction& instr) {
        // 获取代码指针
        uint8_t* const p = code;
        // 获取代码位置
        uint32_t pos = codePos;

        // 获取指令中的源操作数和目的操作数
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;

        // 在代码中写入指令操作码和目的寄存器
        *(uint32_t*)(p + pos) = 0xC4D08B49 + (dst << 16);
        // 在代码中写入指令操作码和目的寄存器以及源寄存器
        *(uint32_t*)(p + pos + 4) = 0xC0F6FB42 + (dst << 27) + (src << 24);
        // 更新代码位置
        pos += 8;

        // 更新目的寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }

    // 定义函数 h_IMULH_M，接受一个指令参数
    void JitCompilerX86::h_IMULH_M(const Instruction& instr) {
        // 获取代码指针
        uint8_t* const p = code;
        // 获取代码位置
        uint32_t pos = codePos;

        // 获取指令中的源操作数和目的操作数
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;

        // 如果源操作数不等于目的操作数
        if (src != dst) {
            // 生成地址寄存器
            genAddressReg<false>(instr, src, p, pos);
            // 在代码中写入指令操作码和目的寄存器
            *(uint64_t*)(p + pos) = 0x0e24f748c08b49ull + (dst << 16);
            // 更新代码位置
            pos += 7;
        }
        else {
            // 在代码中写入指令操作码和目的寄存器
            *(uint64_t*)(p + pos) = 0xa6f748c08b49ull + (dst << 16);
            // 更新代码位置
            pos += 6;
            // 生成立即数地址
            genAddressImm(instr, p, pos);
        }
        // 在代码中写入指令操作码和目的寄存器
        *(uint32_t*)(p + pos) = 0xc28b4c + (dst << 19);
        // 更新代码位置
        pos += 3;

        // 更新目的寄存器的使用位置
        registerUsage[dst] = pos;
        // 更新代码位置
        codePos = pos;
    }
    // 实现对应的指令 IMULH_M_BMI2
    void JitCompilerX86::h_IMULH_M_BMI2(const Instruction& instr) {
        uint8_t* const p = code;  // 定义指令代码的指针
        uint32_t pos = codePos;  // 获取当前指令代码的位置
    
        const uint64_t src = instr.src;  // 获取指令的源操作数
        const uint64_t dst = instr.dst;  // 获取指令的目的操作数
    
        if (src != dst) {  // 如果源操作数和目的操作数不相同
            genAddressReg<false>(instr, src, p, pos);  // 生成寄存器地址
            *(uint32_t*)(p + pos) = static_cast<uint32_t>(0xC4D08B49 + (dst << 16));  // 将指定的值写入指定的位置
            *(uint64_t*)(p + pos + 4) = 0x0E04F6FB62ULL + (dst << 27);  // 将指定的值写入指定的位置
            pos += 9;  // 更新指令代码的位置
        }
        else {  // 如果源操作数和目的操作数相同
            *(uint64_t*)(p + pos) = 0x86F6FB62C4D08B49ULL + (dst << 16) + (dst << 59);  // 将指定的值写入指定的位置
            *(uint32_t*)(p + pos + 8) = instr.getImm32() & ScratchpadL3Mask;  // 将指定的值写入指定的位置
            pos += 12;  // 更新指令代码的位置
        }
    
        registerUsage[dst] = pos;  // 更新寄存器使用情况
        codePos = pos;  // 更新指令代码的位置
    }
    
    // 实现对应的指令 ISMULH_R
    void JitCompilerX86::h_ISMULH_R(const Instruction& instr) {
        uint8_t* const p = code;  // 定义指令代码的指针
        uint32_t pos = codePos;  // 获取当前指令代码的位置
    
        const uint64_t src = instr.src;  // 获取指令的源操作数
        const uint64_t dst = instr.dst;  // 获取指令的目的操作数
    
        *(uint64_t*)(p + pos) = 0x8b4ce8f749c08b49ull + (dst << 16) + (src << 40);  // 将指定的值写入指定的位置
        pos += 8;  // 更新指令代码的位置
        emitByte(0xc2 + 8 * dst, p, pos);  // 生成指定的字节
    
        registerUsage[dst] = pos;  // 更新寄存器使用情况
        codePos = pos;  // 更新指令代码的位置
    }
    
    // 实现对应的指令 ISMULH_M
    void JitCompilerX86::h_ISMULH_M(const Instruction& instr) {
        uint8_t* const p = code;  // 定义指令代码的指针
        uint32_t pos = codePos;  // 获取当前指令代码的位置
    
        const uint64_t src = instr.src;  // 获取指令的源操作数
        const uint64_t dst = instr.dst;  // 获取指令的目的操作数
    
        if (src != dst) {  // 如果源操作数和目的操作数不相同
            genAddressReg<false>(instr, src, p, pos);  // 生成寄存器地址
            *(uint64_t*)(p + pos) = 0x0e2cf748c08b49ull + (dst << 16);  // 将指定的值写入指定的位置
            pos += 7;  // 更新指令代码的位置
        }
        else {  // 如果源操作数和目的操作数相同
            *(uint64_t*)(p + pos) = 0xaef748c08b49ull + (dst << 16);  // 将指定的值写入指定的位置
            pos += 6;  // 更新指令代码的位置
            genAddressImm(instr, p, pos);  // 生成指定的地址和立即数
        }
        *(uint32_t*)(p + pos) = 0xc28b4c + (dst << 19);  // 将指定的值写入指定的位置
        pos += 3;  // 更新指令代码的位置
    
        registerUsage[dst] = pos;  // 更新寄存器使用情况
        codePos = pos;  // 更新指令代码的位置
    }
    # 定义函数 h_IMUL_RCP，接收一个指令作为参数
    void JitCompilerX86::h_IMUL_RCP(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
        
        # 从指令中获取除数的值
        uint64_t divisor = instr.getImm32();
        # 判断除数是否为零或者是2的幂次方
        if (!isZeroOrPowerOf2(divisor)) {
            # 获取目标寄存器的编号
            const uint32_t dst = instr.dst;
    
            # 计算除数的倒数
            const uint64_t reciprocal = randomx_reciprocal_fast(divisor);
            # 判断是否还有可用的存储空间
            if (imul_rcp_storage_used < 16) {
                # 将倒数存储到 imul_rcp_storage 数组中
                *(uint64_t*)(imul_rcp_storage) = reciprocal;
                # 将指令写入代码块，将倒数存储到目标寄存器中
                *(uint64_t*)(p + pos) = 0x2444AF0F4Cull + (dst << 27) + (static_cast<uint64_t>(248 - imul_rcp_storage_used * 8) << 40);
                # 更新已使用的存储空间数量
                ++imul_rcp_storage_used;
                # 更新 imul_rcp_storage 指针的位置
                imul_rcp_storage += 11;
                # 更新代码块的位置
                pos += 6;
            }
            else {
                # 将指令写入代码块，将倒数存储到目标寄存器中
                *(uint32_t*)(p + pos) = 0xb848;
                # 更新代码块的位置
                pos += 2;
    
                # 将倒数存储到代码块中
                emit64(reciprocal, p, pos);
    
                # 将指令写入代码块，将倒数存储到目标寄存器中
                emit32(0xc0af0f4c + (dst << 27), p, pos);
            }
    
            # 更新目标寄存器的使用位置
            registerUsage[dst] = pos;
        }
    
        # 更新代码块的位置
        codePos = pos;
    }
    
    # 定义函数 h_INEG_R，接收一个指令作为参数
    void JitCompilerX86::h_INEG_R(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 获取目标寄存器的编号
        const uint32_t dst = instr.dst;
        # 将指令写入代码块，将目标寄存器的值取反
        *(uint32_t*)(p + pos) = 0xd8f749 + (dst << 16);
        # 更新代码块的位置
        pos += 3;
    
        # 更新目标寄存器的使用位置
        registerUsage[dst] = pos;
        # 更新代码块的位置
        codePos = pos;
    }
    
    # 定义函数 h_IXOR_R，接收一个指令作为参数
    void JitCompilerX86::h_IXOR_R(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 获取源寄存器和目标寄存器的编号
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;
    
        # 判断源寄存器和目标寄存器是否相同
        if (src != dst) {
            # 将指令写入代码块，将源寄存器和目标寄存器进行异或运算
            *(uint32_t*)(p + pos) = 0xc0334d + (((dst << 3) + src) << 16);
            # 更新代码块的位置
            pos += 3;
        }
        else {
            # 从指令中获取立即数的值
            const uint64_t imm = instr.getImm32();
            # 将指令写入代码块，将立即数和目标寄存器进行异或运算
            *(uint64_t*)(p + pos) = (imm << 24) + 0xf08149 + (dst << 16);
            # 更新代码块的位置
            pos += 7;
        }
    
        # 更新目标寄存器的使用位置
        registerUsage[dst] = pos;
        # 更新代码块的位置
        codePos = pos;
    }
    # 对应指令 IXOR_M 的处理函数
    void JitCompilerX86::h_IXOR_M(const Instruction& instr) {
        # 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        # 获取源操作数和目标操作数
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;
    
        # 如果源操作数和目标操作数不相同
        if (src != dst) {
            # 生成源操作数的地址寄存器
            genAddressReg<true>(instr, src, p, pos);
            # 生成目标操作数的指令
            emit32(0x0604334c + (dst << 19), p, pos);
        }
        else {
            # 生成目标操作数的指令
            *(uint32_t*)(p + pos) = 0x86334c + (dst << 19);
            pos += 3;
            # 生成立即数操作数的指令
            genAddressImm(instr, p, pos);
        }
    
        # 更新目标操作数的寄存器使用情况
        registerUsage[dst] = pos;
        # 更新代码位置
        codePos = pos;
    }
    
    # 对应指令 IROR_R 的处理函数
    void JitCompilerX86::h_IROR_R(const Instruction& instr) {
        # 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        # 获取源操作数和目标操作数
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;
    
        # 如果源操作数和目标操作数不相同
        if (src != dst) {
            # 生成源操作数和目标操作数的指令
            *(uint64_t*)(p + pos) = 0xc8d349c88b41ull + (src << 16) + (dst << 40);
            pos += 6;
        }
        else {
            # 生成目标操作数的指令
            *(uint32_t*)(p + pos) = 0xc8c149 + (dst << 16);
            pos += 3;
            # 生成立即数操作数的指令
            emitByte(instr.getImm32() & 63, p, pos);
        }
    
        # 更新目标操作数的寄存器使用情况
        registerUsage[dst] = pos;
        # 更新代码位置
        codePos = pos;
    }
    
    # 对应指令 IROL_R 的处理函数
    void JitCompilerX86::h_IROL_R(const Instruction& instr) {
        # 获取代码指针和位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        # 获取源操作数和目标操作数
        const uint64_t src = instr.src;
        const uint64_t dst = instr.dst;
    
        # 如果源操作数和目标操作数不相同
        if (src != dst) {
            # 生成源操作数和目标操作数的指令
            *(uint64_t*)(p + pos) = 0xc0d349c88b41ull + (src << 16) + (dst << 40);
            pos += 6;
        }
        else {
            # 生成目标操作数的指令
            *(uint32_t*)(p + pos) = 0xc0c149 + (dst << 16);
            pos += 3;
            # 生成立即数操作数的指令
            emitByte(instr.getImm32() & 63, p, pos);
        }
    
        # 更新目标操作数的寄存器使用情况
        registerUsage[dst] = pos;
        # 更新代码位置
        codePos = pos;
    }
    # 定义函数 h_ISWAP_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_ISWAP_R(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 pos，表示当前代码位置
        uint32_t pos = codePos;
    
        # 从 instr 中获取源寄存器和目标寄存器的编号
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst;
    
        # 如果源寄存器和目标寄存器不相同
        if (src != dst) {
            # 将指令写入代码数组中，实现寄存器交换操作
            *(uint32_t*)(p + pos) = 0xc0874d + (((dst << 3) + src) << 16);
            pos += 3;
            # 更新目标寄存器和源寄存器在代码中的使用位置
            registerUsage[dst] = pos;
            registerUsage[src] = pos;
        }
    
        # 更新代码位置
        codePos = pos;
    }
    
    # 定义函数 h_FSWAP_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FSWAP_R(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 pos，表示当前代码位置
        uint32_t pos = codePos;
    
        # 从 instr 中获取目标寄存器的编号
        const uint64_t dst = instr.dst;
    
        # 将指令写入代码数组中，实现浮点寄存器交换操作
        *(uint64_t*)(p + pos) = 0x01c0c60f66ull + (((dst << 3) + dst) << 24);
        pos += 5;
    
        # 更新代码位置
        codePos = pos;
    }
    
    # 定义函数 h_FADD_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FADD_R(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 pos，表示当前代码位置
        uint32_t pos = codePos;
    
        # 将当前位置记录为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 从 instr 中获取目标寄存器和源寄存器的编号
        const uint64_t dst = instr.dst % RegisterCountFlt;
        const uint64_t src = instr.src % RegisterCountFlt;
    
        # 将指令写入代码数组中，实现浮点寄存器相加操作
        *(uint64_t*)(p + pos) = 0xc0580f4166ull + (((dst << 3) + src) << 32);
        pos += 5;
    
        # 更新代码位置
        codePos = pos;
    }
    
    # 定义函数 h_FADD_M，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FADD_M(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 pos，表示当前代码位置
        uint32_t pos = codePos;
    
        # 将当前位置记录为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 从 instr 中获取源寄存器和目标寄存器的编号
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst % RegisterCountFlt;
    
        # 生成用于访问内存的地址寄存器指令
        genAddressReg<true>(instr, src, p, pos);
        # 将指令写入代码数组中，实现浮点寄存器和内存相加操作
        *(uint64_t*)(p + pos) = 0x41660624e60f44f3ull;
        *(uint32_t*)(p + pos + 8) = 0xc4580f + (dst << 19);
        pos += 11;
    
        # 更新代码位置
        codePos = pos;
    }
    # 定义函数 h_FSUB_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FSUB_R(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 将当前位置记录为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 获取指令中的目标寄存器和源寄存器
        const uint64_t dst = instr.dst % RegisterCountFlt;
        const uint64_t src = instr.src % RegisterCountFlt;
    
        # 在代码块中写入指令的机器码
        *(uint64_t*)(p + pos) = 0xc05c0f4166ull + (((dst << 3) + src) << 32);
        # 更新代码块的当前位置
        pos += 5;
    
        # 更新代码块的当前位置
        codePos = pos;
    }
    
    # 定义函数 h_FSUB_M，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FSUB_M(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 将当前位置记录为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 获取指令中的源寄存器和目标寄存器
        const uint32_t src = instr.src;
        const uint32_t dst = instr.dst % RegisterCountFlt;
    
        # 在代码块中生成源内存地址的机器码
        genAddressReg<true>(instr, src, p, pos);
        # 在代码块中写入指令的机器码
        *(uint64_t*)(p + pos) = 0x41660624e60f44f3ull;
        *(uint32_t*)(p + pos + 8) = 0xc45c0f + (dst << 19);
        # 更新代码块的当前位置
        pos += 11;
    
        # 更新代码块的当前位置
        codePos = pos;
    }
    
    # 定义函数 h_FSCAL_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FSCAL_R(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 获取指令中的目标寄存器
        const uint32_t dst = instr.dst % RegisterCountFlt;
    
        # 在代码块中写入指令的机器码
        emit32(0xc7570f41 + (dst << 27), p, pos);
    
        # 更新代码块的当前位置
        codePos = pos;
    }
    
    # 定义函数 h_FMUL_R，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_FMUL_R(const Instruction& instr) {
        # 定义指针 p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量 pos，表示代码块的当前位置
        uint32_t pos = codePos;
    
        # 将当前位置记录为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 获取指令中的目标寄存器和源寄存器
        const uint64_t dst = instr.dst % RegisterCountFlt;
        const uint64_t src = instr.src % RegisterCountFlt;
    
        # 在代码块中写入指令的机器码
        *(uint64_t*)(p + pos) = 0xe0590f4166ull + (((dst << 3) + src) << 32);
        # 更新代码块的当前位置
        pos += 5;
    
        # 更新代码块的当前位置
        codePos = pos;
    }
    # 定义函数 h_FDIV_M，处理 FDIV_M 指令
    void JitCompilerX86::h_FDIV_M(const Instruction& instr) {
        # 获取代码指针和代码位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        # 将当前位置设置为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 获取源操作数和目标操作数
        const uint32_t src = instr.src;
        const uint64_t dst = instr.dst % RegisterCountFlt;
    
        # 生成源操作数的地址寄存器
        genAddressReg<true>(instr, src, p, pos);
    
        # 将指令写入代码块
        *(uint64_t*)(p + pos) = 0x0624e60f44f3ull;
        pos += 6;
        if (hasXOP) {
            *(uint64_t*)(p + pos) = 0xd0e6a218488full;
            pos += 6;
        }
        else {
            *(uint64_t*)(p + pos) = 0xe6560f45e5540f45ull;
            pos += 8;
        }
        *(uint64_t*)(p + pos) = 0xe45e0f4166ull + (dst << 35);
        pos += 5;
    
        # 更新代码位置
        codePos = pos;
    }
    
    # 定义函数 h_FSQRT_R，处理 FSQRT_R 指令
    void JitCompilerX86::h_FSQRT_R(const Instruction& instr) {
        # 获取代码指针和代码位置
        uint8_t* const p = code;
        uint32_t pos = codePos;
    
        # 将当前位置设置为前一个浮点操作的位置
        prevFPOperation = pos;
    
        # 获取目标操作数
        const uint32_t dst = instr.dst % RegisterCountFlt;
    
        # 将指令写入代码块
        emit32(0xe4510f66 + (((dst << 3) + dst) << 24), p, pos);
    
        # 更新代码位置
        codePos = pos;
    }
    # 定义函数h_CFROUND，用于处理CFROUND指令
    void JitCompilerX86::h_CFROUND(const Instruction& instr) {
        # 定义指针p，指向代码块的起始位置
        uint8_t* const p = code;
        # 定义变量t，存储prevCFROUND的值
        int32_t t = prevCFROUND;
    
        # 如果t大于prevFPOperation
        if (t > prevFPOperation) {
            # 如果vm_flags的值与RANDOMX_FLAG_AMD相等
            if (vm_flags & RANDOMX_FLAG_AMD) {
                # 将NOP26的内容复制到p + t的位置
                memcpy(p + t, NOP26, 26);
            }
            else {
                # 将NOP14的内容复制到p + t的位置
                memcpy(p + t, NOP14, 14);
            }
        }
    
        # 将codePos的值赋给变量pos
        uint32_t pos = codePos;
        # 将pos的值赋给prevCFROUND
        prevCFROUND = pos;
    
        # 将instr.src左移16位后与0x00C08B49相加的结果存储到p + pos的位置
        *(uint32_t*)(p + pos) = 0x00C08B49 + (src << 16);
        # 将(instr.getImm32() & 63) - 2的结果与63进行按位与运算后的值存储到rotate中
        const int rotate = (static_cast<int>(instr.getImm32() & 63) - 2) & 63;
        # 将rotate左移24位后与0x00C8C148相加的结果存储到p + pos + 3的位置
        *(uint32_t*)(p + pos + 3) = 0x00C8C148 + (rotate << 24);
    
        # 如果vm_flags的值与RANDOMX_FLAG_AMD相等
        if (vm_flags & RANDOMX_FLAG_AMD) {
            # 将0x742024443B0CE083ULL的值存储到p + pos + 7的位置
            *(uint64_t*)(p + pos + 7) = 0x742024443B0CE083ULL;
            # 将0x8900EB0414AE0F0AULL的值存储到p + pos + 15的位置
            *(uint64_t*)(p + pos + 15) = 0x8900EB0414AE0F0AULL;
            # 将0x202444的值存储到p + pos + 23的位置
            *(uint32_t*)(p + pos + 23) = 0x202444;
            # 将pos + 26的值赋给pos
            pos += 26;
        }
        else {
            # 将0x0414AE0F0CE083ULL的值存储到p + pos + 7的位置
            *(uint64_t*)(p + pos + 7) = 0x0414AE0F0CE083ULL;
            # 将pos + 14的值赋给pos
            pos += 14;
        }
    
        # 将pos的值赋给codePos
        codePos = pos;
    }
    # 定义函数 h_CFROUND_BMI2，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_CFROUND_BMI2(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 t，存储 prevCFROUND 的值
        int32_t t = prevCFROUND;
    
        # 如果 t 大于 prevFPOperation
        if (t > prevFPOperation) {
            # 如果 vm_flags 的值与 RANDOMX_FLAG_AMD 相等
            if (vm_flags & RANDOMX_FLAG_AMD) {
                # 将 NOP25 数组的内容复制到 p + t 的位置
                memcpy(p + t, NOP25, 25);
            }
            else {
                # 将 NOP13 数组的内容复制到 p + t 的位置
                memcpy(p + t, NOP13, 13);
            }
        }
    
        # 将 codePos 的值赋给 prevCFROUND
        uint32_t pos = codePos;
        prevCFROUND = pos;
    
        # 将 instr.src 的值赋给 src
        const uint64_t src = instr.src;
    
        # 将 instr.getImm32() & 63 的结果减去 2，并与 63 进行按位与运算，将结果赋给 rotate
        const uint64_t rotate = (static_cast<int>(instr.getImm32() & 63) - 2) & 63;
        # 将 0xC0F0FBC3C4ULL | (src << 32) | (rotate << 40) 的结果存储到 p + pos 的位置
        *(uint64_t*)(p + pos) = 0xC0F0FBC3C4ULL | (src << 32) | (rotate << 40);
    
        # 如果 vm_flags 的值与 RANDOMX_FLAG_AMD 相等
        if (vm_flags & RANDOMX_FLAG_AMD) {
            # 将 0x742024443B0CE083ULL 的值存储到 p + pos + 6 的位置
            *(uint64_t*)(p + pos + 6) = 0x742024443B0CE083ULL;
            # 将 0x8900EB0414AE0F0AULL 的值存储到 p + pos + 14 的位置
            *(uint64_t*)(p + pos + 14) = 0x8900EB0414AE0F0AULL;
            # 将 0x202444 的值存储到 p + pos + 22 的位置
            *(uint32_t*)(p + pos + 22) = 0x202444;
            # pos 增加 25
            pos += 25;
        }
        else {
            # 将 0x0414AE0F0CE083ULL 的值存储到 p + pos + 6 的位置
            *(uint64_t*)(p + pos + 6) = 0x0414AE0F0CE083ULL;
            # pos 增加 13
            pos += 13;
        }
    
        # 将 pos 的值赋给 codePos
        codePos = pos;
    }
    
    # 定义模板函数 h_CBRANCH，接收一个 Instruction 对象作为参数，模板参数为 jccErratum 的值
    template<bool jccErratum>
    }
    
    # 实例化模板函数 h_CBRANCH<false> 和 h_CBRANCH<true>
    template void JitCompilerX86::h_CBRANCH<false>(const Instruction&);
    template void JitCompilerX86::h_CBRANCH<true>(const Instruction&);
    
    # 定义函数 h_ISTORE，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_ISTORE(const Instruction& instr) {
        # 定义指针 p，指向 code 数组
        uint8_t* const p = code;
        # 定义变量 pos，存储 codePos 的值
        uint32_t pos = codePos;
    
        # 调用 genAddressRegDst 函数，传入 instr、p 和 pos 作为参数
        genAddressRegDst(instr, p, pos);
        # 将 0x0604894c + (static_cast<uint32_t>(instr.src) << 19) 的结果存储到 p + pos 的位置
        emit32(0x0604894c + (static_cast<uint32_t>(instr.src) << 19), p, pos);
    
        # 将 pos 的值赋给 codePos
        codePos = pos;
    }
    
    # 定义函数 h_NOP，接收一个 Instruction 对象作为参数
    void JitCompilerX86::h_NOP(const Instruction& instr) {
        # 在 code 数组的 codePos 位置插入字节 0x90
        emitByte(0x90, code, codePos);
    }
    
    # 定义 InstructionGeneratorX86 数组 engine，长度为 256，对齐到 64 字节边界
    alignas(64) InstructionGeneratorX86 JitCompilerX86::engine[256] = {};
这是一个闭合的大括号，可能是代码块的结束。
```