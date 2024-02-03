# `xmrig\src\crypto\cn\CnHash.cpp`

```cpp
/*
 * XMRig
 * 版权 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，版本为 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有适用于特定目的的隐含保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "crypto/cn/CnHash.h"  // 导入 CnHash 模块
#include "backend/cpu/Cpu.h"  // 导入 Cpu 模块
#include "base/tools/cryptonote/umul128.h"  // 导入 umul128 模块
#include "crypto/common/VirtualMemory.h"  // 导入 VirtualMemory 模块

#if defined(XMRIG_ARM)
#   include "crypto/cn/CryptoNight_arm.h"  // 如果是 ARM 架构，导入 CryptoNight_arm 模块
#else
#   include "crypto/cn/CryptoNight_x86.h"  // 如果不是 ARM 架构，导入 CryptoNight_x86 模块
#endif

#ifdef XMRIG_ALGO_ARGON2
#   include "crypto/argon2/Hash.h"  // 如果定义了 XMRIG_ALGO_ARGON2，导入 Hash 模块
#endif
// 定义宏，用于添加算法函数到映射表中
#define ADD_FN(algo) do {                                                                            \
        // 为指定算法创建新的哈希函数数组对象
        m_map[algo] = new cn_hash_fun_array{};                                                       \
        // 设置单线程无软件优化的哈希函数
        m_map[algo]->data[AV_SINGLE][Assembly::NONE]      = cryptonight_single_hash<algo, false, 0>; \
        // 设置单线程软件优化的哈希函数
        m_map[algo]->data[AV_SINGLE_SOFT][Assembly::NONE] = cryptonight_single_hash<algo, true,  0>; \
        // 设置双线程无软件优化的哈希函数
        m_map[algo]->data[AV_DOUBLE][Assembly::NONE]      = cryptonight_double_hash<algo, false>;    \
        // 设置双线程软件优化的哈希函数
        m_map[algo]->data[AV_DOUBLE_SOFT][Assembly::NONE] = cryptonight_double_hash<algo, true>;     \
        // 设置三线程无软件优化的哈希函数
        m_map[algo]->data[AV_TRIPLE][Assembly::NONE]      = cryptonight_triple_hash<algo, false>;    \
        // 设置三线程软件优化的哈希函数
        m_map[algo]->data[AV_TRIPLE_SOFT][Assembly::NONE] = cryptonight_triple_hash<algo, true>;     \
        // 设置四线程无软件优化的哈希函数
        m_map[algo]->data[AV_QUAD][Assembly::NONE]        = cryptonight_quad_hash<algo,   false>;    \
        // 设置四线程软件优化的哈希函数
        m_map[algo]->data[AV_QUAD_SOFT][Assembly::NONE]   = cryptonight_quad_hash<algo,   true>;     \
        // 设置五线程无软件优化的哈希函数
        m_map[algo]->data[AV_PENTA][Assembly::NONE]       = cryptonight_penta_hash<algo,  false>;    \
        // 设置五线程软件优化的哈希函数
        m_map[algo]->data[AV_PENTA_SOFT][Assembly::NONE]  = cryptonight_penta_hash<algo,  true>;     \
    } while (0)

// 初始化 SSE41 和 VAES 的状态为禁用
bool cn_sse41_enabled = false;
bool cn_vaes_enabled = false;

// 如果定义了 XMRIG_FEATURE_ASM，则执行以下代码
#ifdef XMRIG_FEATURE_ASM
# 定义宏，用于添加特定算法的汇编语言实现到哈希函数映射表中
#define ADD_FN_ASM(algo) do {
    # 为特定算法的单线程、Intel架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_SINGLE][Assembly::INTEL]     = cryptonight_single_hash_asm<algo, Assembly::INTEL>;
    # 为特定算法的单线程、Ryzen架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_SINGLE][Assembly::RYZEN]     = cryptonight_single_hash_asm<algo, Assembly::RYZEN>;
    # 为特定算法的单线程、Bulldozer架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_SINGLE][Assembly::BULLDOZER] = cryptonight_single_hash_asm<algo, Assembly::BULLDOZER>;
    # 为特定算法的双线程、Intel架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_DOUBLE][Assembly::INTEL]     = cryptonight_double_hash_asm<algo, Assembly::INTEL>;
    # 为特定算法的双线程、Ryzen架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_DOUBLE][Assembly::RYZEN]     = cryptonight_double_hash_asm<algo, Assembly::RYZEN>;
    # 为特定算法的双线程、Bulldozer架构的哈希函数添加汇编语言实现
    m_map[algo]->data[AV_DOUBLE][Assembly::BULLDOZER] = cryptonight_double_hash_asm<algo, Assembly::BULLDOZER>;
} while (0)

# 命名空间声明
namespace xmrig {

# 初始化各种算法在不同架构下的汇编语言实现为nullptr
cn_mainloop_fun        cn_half_mainloop_ivybridge_asm             = nullptr;
cn_mainloop_fun        cn_half_mainloop_ryzen_asm                 = nullptr;
cn_mainloop_fun        cn_half_mainloop_bulldozer_asm             = nullptr;
cn_mainloop_fun        cn_half_double_mainloop_sandybridge_asm    = nullptr;

cn_mainloop_fun        cn_trtl_mainloop_ivybridge_asm             = nullptr;
cn_mainloop_fun        cn_trtl_mainloop_ryzen_asm                 = nullptr;
cn_mainloop_fun        cn_trtl_mainloop_bulldozer_asm             = nullptr;
cn_mainloop_fun        cn_trtl_double_mainloop_sandybridge_asm    = nullptr;

cn_mainloop_fun        cn_tlo_mainloop_ivybridge_asm              = nullptr;
cn_mainloop_fun        cn_tlo_mainloop_ryzen_asm                  = nullptr;
cn_mainloop_fun        cn_tlo_mainloop_bulldozer_asm              = nullptr;
cn_mainloop_fun        cn_tlo_double_mainloop_sandybridge_asm     = nullptr;

cn_mainloop_fun        cn_zls_mainloop_ivybridge_asm              = nullptr;
cn_mainloop_fun        cn_zls_mainloop_ryzen_asm                  = nullptr;
# 定义指向函数的指针，初始化为 nullptr
cn_mainloop_fun        cn_zls_mainloop_bulldozer_asm              = nullptr;
cn_mainloop_fun        cn_zls_double_mainloop_sandybridge_asm     = nullptr;

cn_mainloop_fun        cn_double_mainloop_ivybridge_asm           = nullptr;
cn_mainloop_fun        cn_double_mainloop_ryzen_asm               = nullptr;
cn_mainloop_fun        cn_double_mainloop_bulldozer_asm           = nullptr;
cn_mainloop_fun        cn_double_double_mainloop_sandybridge_asm  = nullptr;

cn_mainloop_fun        cn_upx2_mainloop_asm                       = nullptr;
cn_mainloop_fun        cn_upx2_double_mainloop_asm                = nullptr;

cn_mainloop_fun        cn_gr0_single_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr1_single_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr2_single_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr3_single_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr4_single_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr5_single_mainloop_asm                 = nullptr;

cn_mainloop_fun        cn_gr0_double_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr1_double_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr2_double_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr3_double_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr4_double_mainloop_asm                 = nullptr;
cn_mainloop_fun        cn_gr5_double_mainloop_asm                 = nullptr;

cn_mainloop_fun        cn_gr0_quad_mainloop_asm                   = nullptr;
cn_mainloop_fun        cn_gr1_quad_mainloop_asm                   = nullptr;
cn_mainloop_fun        cn_gr2_quad_mainloop_asm                   = nullptr;
cn_mainloop_fun        cn_gr3_quad_mainloop_asm                   = nullptr;
cn_mainloop_fun        cn_gr4_quad_mainloop_asm                   = nullptr;
// 定义指向 cn_gr5_quad_mainloop_asm 函数的指针，初始值为空指针
cn_mainloop_fun        cn_gr5_quad_mainloop_asm                   = nullptr;

// 定义模板函数 patchCode，用于修改代码
template<Algorithm::Id SOURCE_ALGO = Algorithm::CN_2, typename T, typename U>
static void patchCode(T dst, U src, const uint32_t iterations, const uint32_t mask = CnAlgo<Algorithm::CN_HALF>().mask())
{
    auto p = reinterpret_cast<const uint8_t*>(src);

    // 在 Visual Studio 调试构建中，解决跳板代码的问题
#   if defined(_MSC_VER)
    if (p[0] == 0xE9) {
        p += *(int32_t*)(p + 1) + 5;
    }
#   endif

    // 计算要修改的代码块的大小
    size_t size = 0;
    while (*(uint32_t*)(p + size) != 0xDEADC0DE) {
        ++size;
    }

    size += sizeof(uint32_t);

    // 将源代码块的内容复制到目标地址
    memcpy((void*) dst, (const void*) src, size);

    // 对修改后的代码进行进一步处理
    auto patched_data = reinterpret_cast<uint8_t*>(dst);
    for (size_t i = 0; i + sizeof(uint32_t) <= size; ++i) {
        switch (*(uint32_t*)(patched_data + i)) {
        case CnAlgo<SOURCE_ALGO>().iterations():
            *(uint32_t*)(patched_data + i) = iterations;
            break;

        case CnAlgo<SOURCE_ALGO>().mask():
            *(uint32_t*)(patched_data + i) = mask;
            break;
        }
    }
}

// 对汇编代码变体进行修补
static void patchAsmVariants()
{
    // 分配可执行内存空间
    constexpr size_t allocation_size = 0x20000;
    auto base = static_cast<uint8_t *>(VirtualMemory::allocateExecutableMemory(allocation_size, false));

    // 将不同的汇编代码函数指针指向不同的内存地址
    cn_half_mainloop_ivybridge_asm              = reinterpret_cast<cn_mainloop_fun>         (base + 0x0000);
    cn_half_mainloop_ryzen_asm                  = reinterpret_cast<cn_mainloop_fun>         (base + 0x1000);
    cn_half_mainloop_bulldozer_asm              = reinterpret_cast<cn_mainloop_fun>         (base + 0x2000);
    cn_half_double_mainloop_sandybridge_asm     = reinterpret_cast<cn_mainloop_fun>         (base + 0x3000);

#   ifdef XMRIG_ALGO_CN_PICO
    cn_trtl_mainloop_ivybridge_asm              = reinterpret_cast<cn_mainloop_fun>         (base + 0x4000);
    cn_trtl_mainloop_ryzen_asm                  = reinterpret_cast<cn_mainloop_fun>         (base + 0x5000);
    // 将地址偏移量加上0x6000，然后将结果转换为 cn_mainloop_fun 类型的函数指针，赋值给 cn_trtl_mainloop_bulldozer_asm
    cn_trtl_mainloop_bulldozer_asm              = reinterpret_cast<cn_mainloop_fun>         (base + 0x6000);
    // 将地址偏移量加上0x7000，然后将结果转换为 cn_mainloop_fun 类型的函数指针，赋值给 cn_trtl_double_mainloop_sandybridge_asm
    cn_trtl_double_mainloop_sandybridge_asm     = reinterpret_cast<cn_mainloop_fun>         (base + 0x7000);
#   endif
#   ifdef XMRIG_ALGO_CN_PICO
    # 根据条件编译选择性添加代码
    cn_tlo_mainloop_ivybridge_asm               = reinterpret_cast<cn_mainloop_fun>         (base + 0x10000);
    cn_tlo_mainloop_ryzen_asm                   = reinterpret_cast<cn_mainloop_fun>         (base + 0x11000);
    cn_tlo_mainloop_bulldozer_asm               = reinterpret_cast<cn_mainloop_fun>         (base + 0x12000);
    cn_tlo_double_mainloop_sandybridge_asm      = reinterpret_cast<cn_mainloop_fun>         (base + 0x13000);
#   endif
#   ifdef XMRIG_ALGO_CN_FEMTO
    # 根据条件编译选择性添加代码
    cn_upx2_mainloop_asm                        = reinterpret_cast<cn_mainloop_fun>         (base + 0x14000);
    cn_upx2_double_mainloop_asm                 = reinterpret_cast<cn_mainloop_fun>         (base + 0x15000);
#   endif
#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 根据条件编译选择性添加代码
    cn_gr0_single_mainloop_asm                  = reinterpret_cast<cn_mainloop_fun>         (base + 0x16000);
    cn_gr1_single_mainloop_asm                  = reinterpret_cast<cn_mainloop_fun>         (base + 0x16800);
    cn_gr2_single_mainloop_asm                  = reinterpret_cast<cn_mainloop_fun>         (base + 0x17000);
    # 将地址偏移量加上0x17800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr3_single_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x17800);
    # 将地址偏移量加上0x18000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr4_single_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x18000);
    # 将地址偏移量加上0x18800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr5_single_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x18800);
    
    # 将地址偏移量加上0x19000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr0_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x19000);
    # 将地址偏移量加上0x19800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr1_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x19800);
    # 将地址偏移量加上0x1A000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr2_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1A000);
    # 将地址偏移量加上0x1A800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr3_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1A800);
    # 将地址偏移量加上0x1B000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr4_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1B000);
    # 将地址偏移量加上0x1B800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr5_double_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1B800);
    
    # 将地址偏移量加上0x1C000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr0_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1C000);
    # 将地址偏移量加上0x1C800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr1_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1C800);
    # 将地址偏移量加上0x1D000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr2_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1D000);
    # 将地址偏移量加上0x1D800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr3_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1D800);
    # 将地址偏移量加上0x1E000，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr4_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1E000);
    # 将地址偏移量加上0x1E800，然后将结果转换为cn_mainloop_fun类型的函数指针
    cn_gr5_quad_mainloop_asm = reinterpret_cast<cn_mainloop_fun>(base + 0x1E800);
#   endif
{
    # 定义常量 ITER 为 CN_HALF 算法的迭代次数
    constexpr uint32_t ITER = CnAlgo<Algorithm::CN_HALF>().iterations();
    # 对 cn_half_mainloop_ivybridge_asm 进行补丁，替换为 cnv2_mainloop_ivybridge_asm，迭代次数为 ITER
    patchCode(cn_half_mainloop_ivybridge_asm, cnv2_mainloop_ivybridge_asm, ITER);
    # 对 cn_half_mainloop_ryzen_asm 进行补丁，替换为 cnv2_mainloop_ryzen_asm，迭代次数为 ITER
    patchCode(cn_half_mainloop_ryzen_asm, cnv2_mainloop_ryzen_asm, ITER);
    # 对 cn_half_mainloop_bulldozer_asm 进行补丁，替换为 cnv2_mainloop_bulldozer_asm，迭代次数为 ITER
    patchCode(cn_half_mainloop_bulldozer_asm, cnv2_mainloop_bulldozer_asm, ITER);
    # 对 cn_half_double_mainloop_sandybridge_asm 进行补丁，替换为 cnv2_double_mainloop_sandybridge_asm，迭代次数为 ITER
    patchCode(cn_half_double_mainloop_sandybridge_asm, cnv2_double_mainloop_sandybridge_asm, ITER);
}

#   ifdef XMRIG_ALGO_CN_PICO
{
    # 定义常量 ITER 为 CN_PICO_0 算法的迭代次数
    constexpr uint32_t ITER = CnAlgo<Algorithm::CN_PICO_0>().iterations();
    # 定义常量 MASK 为 CN_PICO_0 算法的掩码
    constexpr uint32_t MASK = CnAlgo<Algorithm::CN_PICO_0>().mask();
    # 对 cn_trtl_mainloop_ivybridge_asm 进行补丁，替换为 cnv2_mainloop_ivybridge_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_trtl_mainloop_ivybridge_asm, cnv2_mainloop_ivybridge_asm, ITER, MASK);
    # 对 cn_trtl_mainloop_ryzen_asm 进行补丁，替换为 cnv2_mainloop_ryzen_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_trtl_mainloop_ryzen_asm, cnv2_mainloop_ryzen_asm, ITER, MASK);
    # 对 cn_trtl_mainloop_bulldozer_asm 进行补丁，替换为 cnv2_mainloop_bulldozer_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_trtl_mainloop_bulldozer_asm, cnv2_mainloop_bulldozer_asm, ITER, MASK);
    # 对 cn_trtl_double_mainloop_sandybridge_asm 进行补丁，替换为 cnv2_double_mainloop_sandybridge_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_trtl_double_mainloop_sandybridge_asm, cnv2_double_mainloop_sandybridge_asm, ITER, MASK);
}

{
    # 定义常量 ITER 为 CN_PICO_TLO 算法的迭代次数
    constexpr uint32_t ITER = CnAlgo<Algorithm::CN_PICO_TLO>().iterations();
    # 定义常量 MASK 为 CN_PICO_TLO 算法的掩码
    constexpr uint32_t MASK = CnAlgo<Algorithm::CN_PICO_TLO>().mask();
    # 对 cn_tlo_mainloop_ivybridge_asm 进行补丁，替换为 cnv2_mainloop_ivybridge_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_tlo_mainloop_ivybridge_asm, cnv2_mainloop_ivybridge_asm, ITER, MASK);
    # 对 cn_tlo_mainloop_ryzen_asm 进行补丁，替换为 cnv2_mainloop_ryzen_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_tlo_mainloop_ryzen_asm, cnv2_mainloop_ryzen_asm, ITER, MASK);
    # 对 cn_tlo_mainloop_bulldozer_asm 进行补丁，替换为 cnv2_mainloop_bulldozer_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_tlo_mainloop_bulldozer_asm, cnv2_mainloop_bulldozer_asm, ITER, MASK);
    # 对 cn_tlo_double_mainloop_sandybridge_asm 进行补丁，替换为 cnv2_double_mainloop_sandybridge_asm，迭代次数为 ITER，掩码为 MASK
    patchCode(cn_tlo_double_mainloop_sandybridge_asm, cnv2_double_mainloop_sandybridge_asm, ITER, MASK);
}
#   endif
    {
        // 获取CN_ZLS算法的迭代次数
        constexpr uint32_t ITER = CnAlgo<Algorithm::CN_ZLS>().iterations();

        // 对ivybridge架构的汇编代码进行补丁，替换为cnv2_mainloop_ivybridge_asm，迭代次数为ITER
        patchCode(cn_zls_mainloop_ivybridge_asm,             cnv2_mainloop_ivybridge_asm,           ITER);
        // 对ryzen架构的汇编代码进行补丁，替换为cnv2_mainloop_ryzen_asm，迭代次数为ITER
        patchCode(cn_zls_mainloop_ryzen_asm,                 cnv2_mainloop_ryzen_asm,               ITER);
        // 对bulldozer架构的汇编代码进行补丁，替换为cnv2_mainloop_bulldozer_asm，迭代次数为ITER
        patchCode(cn_zls_mainloop_bulldozer_asm,             cnv2_mainloop_bulldozer_asm,           ITER);
        // 对sandybridge架构的双重主循环汇编代码进行补丁，替换为cnv2_double_mainloop_sandybridge_asm，迭代次数为ITER
        patchCode(cn_zls_double_mainloop_sandybridge_asm,    cnv2_double_mainloop_sandybridge_asm,  ITER);
    }

    {
        // 获取CN_DOUBLE算法的迭代次数
        constexpr uint32_t ITER = CnAlgo<Algorithm::CN_DOUBLE>().iterations();

        // 对ivybridge架构的汇编代码进行补丁，替换为cnv2_mainloop_ivybridge_asm，迭代次数为ITER
        patchCode(cn_double_mainloop_ivybridge_asm,          cnv2_mainloop_ivybridge_asm,           ITER);
        // 对ryzen架构的汇编代码进行补丁，替换为cnv2_mainloop_ryzen_asm，迭代次数为ITER
        patchCode(cn_double_mainloop_ryzen_asm,              cnv2_mainloop_ryzen_asm,               ITER);
        // 对bulldozer架构的汇编代码进行补丁，替换为cnv2_mainloop_bulldozer_asm，迭代次数为ITER
        patchCode(cn_double_mainloop_bulldozer_asm,          cnv2_mainloop_bulldozer_asm,           ITER);
        // 对sandybridge架构的双重主循环汇编代码进行补丁，替换为cnv2_double_mainloop_sandybridge_asm，迭代次数为ITER
        patchCode(cn_double_double_mainloop_sandybridge_asm, cnv2_double_mainloop_sandybridge_asm,  ITER);
    }
# 如果定义了 XMRIG_ALGO_CN_FEMTO，则执行以下代码块
{
    # 使用 CN_UPX2 算法的迭代次数和掩码来修补代码
    patchCode<Algorithm::CN_RWZ>(cn_upx2_mainloop_asm, cnv2_rwz_mainloop_asm, ITER, MASK);
    patchCode<Algorithm::CN_RWZ>(cn_upx2_double_mainloop_asm, cnv2_rwz_double_mainloop_asm, ITER, MASK);
}
# 结束 XMRIG_ALGO_CN_FEMTO 代码块

# 如果定义了 XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
patchCode<Algorithm::CN_1>(cn_gr0_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_0>().iterations(), CnAlgo<Algorithm::CN_GR_0>().mask());
patchCode<Algorithm::CN_1>(cn_gr1_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_1>().iterations(), CnAlgo<Algorithm::CN_GR_1>().mask());
patchCode<Algorithm::CN_1>(cn_gr2_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_2>().iterations(), CnAlgo<Algorithm::CN_GR_2>().mask());
patchCode<Algorithm::CN_1>(cn_gr3_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_3>().iterations(), CnAlgo<Algorithm::CN_GR_3>().mask());
patchCode<Algorithm::CN_1>(cn_gr4_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_4>().iterations(), CnAlgo<Algorithm::CN_GR_4>().mask());
patchCode<Algorithm::CN_1>(cn_gr5_single_mainloop_asm, cnv1_single_mainloop_asm, CnAlgo<Algorithm::CN_GR_5>().iterations(), CnAlgo<Algorithm::CN_GR_5>().mask());

patchCode<Algorithm::CN_1>(cn_gr0_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_0>().iterations(), CnAlgo<Algorithm::CN_GR_0>().mask());
patchCode<Algorithm::CN_1>(cn_gr1_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_1>().iterations(), CnAlgo<Algorithm::CN_GR_1>().mask());
patchCode<Algorithm::CN_1>(cn_gr2_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_2>().iterations(), CnAlgo<Algorithm::CN_GR_2>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr3_double_mainloop_asm、cnv1_double_mainloop_asm、CN_GR_3算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr3_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_3>().iterations(), CnAlgo<Algorithm::CN_GR_3>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr4_double_mainloop_asm、cnv1_double_mainloop_asm、CN_GR_4算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr4_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_4>().iterations(), CnAlgo<Algorithm::CN_GR_4>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr5_double_mainloop_asm、cnv1_double_mainloop_asm、CN_GR_5算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr5_double_mainloop_asm, cnv1_double_mainloop_asm, CnAlgo<Algorithm::CN_GR_5>().iterations(), CnAlgo<Algorithm::CN_GR_5>().mask());
    
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr0_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_0算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr0_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_0>().iterations(), CnAlgo<Algorithm::CN_GR_0>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr1_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_1算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr1_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_1>().iterations(), CnAlgo<Algorithm::CN_GR_1>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr2_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_2算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr2_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_2>().iterations(), CnAlgo<Algorithm::CN_GR_2>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr3_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_3算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr3_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_3>().iterations(), CnAlgo<Algorithm::CN_GR_3>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr4_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_4算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr4_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_4>().iterations(), CnAlgo<Algorithm::CN_GR_4>().mask());
    # 使用patchCode函数对CN_1算法进行代码修补，传入cn_gr5_quad_mainloop_asm、cnv1_quad_mainloop_asm、CN_GR_5算法的迭代次数和掩码
    patchCode<Algorithm::CN_1>(cn_gr5_quad_mainloop_asm, cnv1_quad_mainloop_asm, CnAlgo<Algorithm::CN_GR_5>().iterations(), CnAlgo<Algorithm::CN_GR_5>().mask());
// 结束条件判断
#   endif

// 设置内存区域为可执行和可读
VirtualMemory::protectRX(base, allocation_size);
// 刷新指令缓存
VirtualMemory::flushInstructionCache(base, allocation_size);
}
} // namespace xmrig

// 定义静态常量 xmrig::CnHash 类
static const xmrig::CnHash cnHash;

// xmrig::CnHash 类的构造函数
xmrig::CnHash::CnHash()
{
    // 添加算法函数
    ADD_FN(Algorithm::CN_0);
    ADD_FN(Algorithm::CN_1);
    ADD_FN(Algorithm::CN_2);
    ADD_FN(Algorithm::CN_R);
    ADD_FN(Algorithm::CN_FAST);
    ADD_FN(Algorithm::CN_HALF);
    ADD_FN(Algorithm::CN_XAO);
    ADD_FN(Algorithm::CN_RTO);
    ADD_FN(Algorithm::CN_RWZ);
    ADD_FN(Algorithm::CN_ZLS);
    ADD_FN(Algorithm::CN_DOUBLE);

    // 添加算法函数（汇编版本）
    ADD_FN_ASM(Algorithm::CN_2);
    ADD_FN_ASM(Algorithm::CN_HALF);
    ADD_FN_ASM(Algorithm::CN_R);
    ADD_FN_ASM(Algorithm::CN_RWZ);
    ADD_FN_ASM(Algorithm::CN_ZLS);
    ADD_FN_ASM(Algorithm::CN_DOUBLE);

    // 根据条件编译添加算法函数
#   ifdef XMRIG_ALGO_CN_LITE
    ADD_FN(Algorithm::CN_LITE_0);
    ADD_FN(Algorithm::CN_LITE_1);
#   endif

#   ifdef XMRIG_ALGO_CN_HEAVY
    ADD_FN(Algorithm::CN_HEAVY_0);
    ADD_FN(Algorithm::CN_HEAVY_TUBE);
    ADD_FN(Algorithm::CN_HEAVY_XHV);
#   endif

#   ifdef XMRIG_ALGO_CN_PICO
    ADD_FN(Algorithm::CN_PICO_0);
    ADD_FN_ASM(Algorithm::CN_PICO_0);
    ADD_FN(Algorithm::CN_PICO_TLO);
    ADD_FN_ASM(Algorithm::CN_PICO_TLO);
#   endif

    // 添加算法函数
    ADD_FN(Algorithm::CN_CCX);

#   ifdef XMRIG_ALGO_CN_FEMTO
    ADD_FN(Algorithm::CN_UPX2);
    ADD_FN_ASM(Algorithm::CN_UPX2);
#   endif

#   ifdef XMRIG_ALGO_ARGON2
    // 初始化算法函数映射
    m_map[Algorithm::AR2_CHUKWA] = new cn_hash_fun_array{};
    m_map[Algorithm::AR2_CHUKWA]->data[AV_SINGLE][Assembly::NONE]         = argon2::single_hash<Algorithm::AR2_CHUKWA>;
    m_map[Algorithm::AR2_CHUKWA]->data[AV_SINGLE_SOFT][Assembly::NONE]    = argon2::single_hash<Algorithm::AR2_CHUKWA>;

    m_map[Algorithm::AR2_CHUKWA_V2] = new cn_hash_fun_array{};
    m_map[Algorithm::AR2_CHUKWA_V2]->data[AV_SINGLE][Assembly::NONE]      = argon2::single_hash<Algorithm::AR2_CHUKWA_V2>;
    # 将 AR2_CHUKWA_V2 算法的单一哈希函数存储到数据结构中
    m_map[Algorithm::AR2_CHUKWA_V2]->data[AV_SINGLE_SOFT][Assembly::NONE] = argon2::single_hash<Algorithm::AR2_CHUKWA_V2>;

    # 创建 AR2_WRKZ 算法的哈希函数数组对象
    m_map[Algorithm::AR2_WRKZ] = new cn_hash_fun_array{};
    # 将 AR2_WRKZ 算法的单一哈希函数存储到数据结构中
    m_map[Algorithm::AR2_WRKZ]->data[AV_SINGLE][Assembly::NONE]           = argon2::single_hash<Algorithm::AR2_WRKZ>;
    # 将 AR2_WRKZ 算法的单一软件哈希函数存储到数据结构中
    m_map[Algorithm::AR2_WRKZ]->data[AV_SINGLE_SOFT][Assembly::NONE]      = argon2::single_hash<Algorithm::AR2_WRKZ>;
#   endif
#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果定义了 XMRIG_ALGO_GHOSTRIDER，则执行以下代码
    ADD_FN(Algorithm::CN_GR_0);
    ADD_FN(Algorithm::CN_GR_1);
    ADD_FN(Algorithm::CN_GR_2);
    ADD_FN(Algorithm::CN_GR_3);
    ADD_FN(Algorithm::CN_GR_4);
    ADD_FN(Algorithm::CN_GR_5);
    # 添加指定的算法函数到函数列表中
#   endif
#   ifdef XMRIG_FEATURE_ASM
    # 如果定义了 XMRIG_FEATURE_ASM，则执行以下代码
    # 对汇编变体进行修补
    patchAsmVariants();
#   endif
}
# 析构函数，用于释放内存
xmrig::CnHash::~CnHash()
{
    # 遍历 m_map 中的元素
    for (auto const& x : m_map) {
      # 释放 m_map 中的指针所指向的内存
      delete m_map[x.first];
    }
}
# 返回指定算法的哈希函数
xmrig::cn_hash_fun xmrig::CnHash::fn(const Algorithm &algorithm, AlgoVariant av, Assembly::Id assembly)
{
    # 断言 m_map 中包含指定的算法
    assert(cnHash.m_map.count(algorithm));
    # 如果算法无效，则返回空指针
    if (!algorithm.isValid()) {
        return nullptr;
    }
    # 在 m_map 中查找指定算法
    const auto it = cnHash.m_map.find(algorithm);
    # 如果未找到指定算法，则返回空指针
    if (it == cnHash.m_map.end()) {
        return nullptr;
    }
#   ifdef XMRIG_ALGO_CN_HEAVY
    # 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码
    # 对 Zen3/Zen4 CPU 进行 cn-heavy 优化
    const auto arch = Cpu::info()->arch();
    const uint32_t model = Cpu::info()->model();
    const bool is_vermeer = (arch == ICpuInfo::ARCH_ZEN3) && (model == 0x21);
    const bool is_raphael = (arch == ICpuInfo::ARCH_ZEN4) && (model == 0x61);
    # 如果满足条件，则返回对应的哈希函数
    if ((av == AV_SINGLE) && (assembly != Assembly::NONE) && (is_vermeer || is_raphael)) {
        switch (algorithm.id()) {
        case Algorithm::CN_HEAVY_0:
            return cryptonight_single_hash<Algorithm::CN_HEAVY_0, false, 3>;
        case Algorithm::CN_HEAVY_TUBE:
            return cryptonight_single_hash<Algorithm::CN_HEAVY_TUBE, false, 3>;
        case Algorithm::CN_HEAVY_XHV:
            return cryptonight_single_hash<Algorithm::CN_HEAVY_XHV, false, 3>;
        default:
            break;
        }
    }
#   endif
#   ifdef XMRIG_FEATURE_ASM
    # 如果定义了 XMRIG_FEATURE_ASM，则执行以下代码
    # 获取指定算法和汇编类型对应的哈希函数
    cn_hash_fun fun = it->second->data[av][Cpu::assembly(assembly)];
    # 如果哈希函数存在，则返回该函数
    if (fun) {
        return fun;
    }
#   endif
    # 返回指定算法和汇编类型对应的哈希函数，如果不存在则返回空指针
    return it->second->data[av][Assembly::NONE];
}
```