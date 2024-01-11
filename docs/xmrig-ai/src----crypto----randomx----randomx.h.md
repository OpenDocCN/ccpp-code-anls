# `xmrig\src\crypto\randomx\randomx.h`

```
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，都可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，
包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，无论是合同、严格责任还是侵权行为（包括疏忽或其他原因），
都不承担任何直接、间接、偶然、特殊、惩罚性或后果性的损害赔偿责任，
包括但不限于替代商品或服务的采购、使用、数据或利润损失或业务中断，
即使事先被告知此类损害的可能性。
*/

#ifndef RANDOMX_H
#define RANDOMX_H

#include <cstddef>
#include <cstdint>
#include <type_traits>
#include "crypto/randomx/intrin_portable.h"

#define RANDOMX_HASH_SIZE 32
#define RANDOMX_DATASET_ITEM_SIZE 64

#ifndef RANDOMX_EXPORT
#define RANDOMX_EXPORT
#endif


注释：这段代码是一个头文件的开头，包含了版权声明、宏定义和一些头文件的引用。
# 定义一个枚举类型 randomx_flags，包含不同的标志位
enum randomx_flags {
  RANDOMX_FLAG_DEFAULT = 0,  # 默认标志位
  RANDOMX_FLAG_LARGE_PAGES = 1,  # 大页标志位
  RANDOMX_FLAG_HARD_AES = 2,  # 强AES标志位
  RANDOMX_FLAG_FULL_MEM = 4,  # 全内存标志位
  RANDOMX_FLAG_JIT = 8,  # JIT标志位
  RANDOMX_FLAG_1GB_PAGES = 16,  # 1GB页标志位
  RANDOMX_FLAG_AMD = 64,  # AMD标志位
};

# 定义 randomx_dataset 结构体
struct randomx_dataset;

# 定义 randomx_cache 结构体
struct randomx_cache;

# 定义 randomx_vm 类
class randomx_vm;

# 定义 RandomX_ConfigurationBase 结构体
struct RandomX_ConfigurationBase
{
    # 构造函数
    RandomX_ConfigurationBase();

    # 应用配置
    void Apply();

    # 所有 RandomX 变体的公共参数
    enum Params : uint64_t
    {
        ArgonMemory = 262144,  # Argon 内存大小
        CacheAccesses = 8,  # 缓存访问次数
        SuperscalarLatency = 170,  # 超标量延迟
        DatasetBaseSize = 2147483648,  # 数据集基础大小
        DatasetExtraSize = 33554368,  # 数据集额外大小
        JumpBits = 8,  # 跳转位数
        JumpOffset = 8,  # 跳转偏移
        CacheLineAlignMask_Calculated = (DatasetBaseSize - 1) & ~(RANDOMX_DATASET_ITEM_SIZE - 1),  # 缓存行对齐掩码
        DatasetExtraItems_Calculated = DatasetExtraSize / RANDOMX_DATASET_ITEM_SIZE,  # 数据集额外项数
        ConditionMask_Calculated = ((1 << JumpBits) - 1) << JumpOffset,  # 条件掩码
    };

    uint32_t ArgonIterations;  # Argon 迭代次数
    uint32_t ArgonLanes;  # Argon 轨道数
    const char* ArgonSalt;  # Argon 盐值

    uint32_t ScratchpadL1_Size;  # Scratchpad L1 大小
    uint32_t ScratchpadL2_Size;  # Scratchpad L2 大小
    uint32_t ScratchpadL3_Size;  # Scratchpad L3 大小

    uint32_t ProgramSize;  # 程序大小
    uint32_t ProgramIterations;  # 程序迭代次数
    uint32_t ProgramCount;  # 程序计数

    uint32_t RANDOMX_FREQ_IADD_RS;  # IADD_RS 指令频率
    uint32_t RANDOMX_FREQ_IADD_M;  # IADD_M 指令频率
    uint32_t RANDOMX_FREQ_ISUB_R;  # ISUB_R 指令频率
    uint32_t RANDOMX_FREQ_ISUB_M;  # ISUB_M 指令频率
    uint32_t RANDOMX_FREQ_IMUL_R;  # IMUL_R 指令频率
    uint32_t RANDOMX_FREQ_IMUL_M;  # IMUL_M 指令频率
    uint32_t RANDOMX_FREQ_IMULH_R;  # IMULH_R 指令频率
    uint32_t RANDOMX_FREQ_IMULH_M;  # IMULH_M 指令频率
    uint32_t RANDOMX_FREQ_ISMULH_R;  # ISMULH_R 指令频率
    uint32_t RANDOMX_FREQ_ISMULH_M;  # ISMULH_M 指令频率
    uint32_t RANDOMX_FREQ_IMUL_RCP;  # IMUL_RCP 指令频率
    uint32_t RANDOMX_FREQ_INEG_R;  # INEG_R 指令频率
    uint32_t RANDOMX_FREQ_IXOR_R;  # IXOR_R 指令频率
    uint32_t RANDOMX_FREQ_IXOR_M;  # IXOR_M 指令频率
    uint32_t RANDOMX_FREQ_IROR_R;  # IROR_R 指令频率
    uint32_t RANDOMX_FREQ_IROL_R;  # IROL_R 指令频率
    uint32_t RANDOMX_FREQ_ISWAP_R;  # ISWAP_R 指令频率
    uint32_t RANDOMX_FREQ_FSWAP_R;  # FSWAP_R 指令频率
    uint32_t RANDOMX_FREQ_FADD_R;  # FADD_R 指令频率
    uint32_t RANDOMX_FREQ_FADD_M;  # FADD_M 指令频率
    uint32_t RANDOMX_FREQ_FSUB_R;  # FSUB_R 指令频率
    uint32_t RANDOMX_FREQ_FSUB_M;  # FSUB_M 指令频率
    uint32_t RANDOMX_FREQ_FSCAL_R;  # FSCAL_R 指令频率
    # 定义随机数频率常量，用于指示随机数操作的频率
    uint32_t RANDOMX_FREQ_FMUL_R;
    uint32_t RANDOMX_FREQ_FDIV_M;
    uint32_t RANDOMX_FREQ_FSQRT_R;
    uint32_t RANDOMX_FREQ_CBRANCH;
    uint32_t RANDOMX_FREQ_CFROUND;
    uint32_t RANDOMX_FREQ_ISTORE;
    uint32_t RANDOMX_FREQ_NOP;
    
    # 定义用于AES加密的128位向量数组
    rx_vec_i128 fillAes4Rx4_Key[8];
    
    # 定义用于SSH预取的字节数组和大小
    uint8_t codeSshPrefetchTweaked[20];
    uint8_t codePrefetchScratchpadTweaked[28];
    uint32_t codePrefetchScratchpadTweakedSize;
    
    # 定义用于地址掩码计算的32位整数数组和单个32位整数
    uint32_t AddressMask_Calculated[4];
    uint32_t ScratchpadL3Mask_Calculated;
    uint32_t ScratchpadL3Mask64_Calculated;
# 定义了一系列 uint32_t 类型的变量，用于存储不同配置参数的对数值
uint32_t Log2_ScratchpadL1;
uint32_t Log2_ScratchpadL2;
uint32_t Log2_ScratchpadL3;
uint32_t Log2_DatasetBaseSize;
uint32_t Log2_CacheSize;

# 定义了一系列结构体，分别用于不同的加密货币配置，继承自 RandomX_ConfigurationBase
struct RandomX_ConfigurationMonero : public RandomX_ConfigurationBase {};
struct RandomX_ConfigurationWownero : public RandomX_ConfigurationBase { RandomX_ConfigurationWownero(); };
struct RandomX_ConfigurationArqma : public RandomX_ConfigurationBase { RandomX_ConfigurationArqma(); };
struct RandomX_ConfigurationGraft : public RandomX_ConfigurationBase { RandomX_ConfigurationGraft(); };
struct RandomX_ConfigurationSafex : public RandomX_ConfigurationBase { RandomX_ConfigurationSafex(); };
struct RandomX_ConfigurationKeva : public RandomX_ConfigurationBase { RandomX_ConfigurationKeva(); };

# 定义了一系列全局变量，分别用于不同加密货币的配置
extern RandomX_ConfigurationMonero RandomX_MoneroConfig;
extern RandomX_ConfigurationWownero RandomX_WowneroConfig;
extern RandomX_ConfigurationArqma RandomX_ArqmaConfig;
extern RandomX_ConfigurationGraft RandomX_GraftConfig;
extern RandomX_ConfigurationSafex RandomX_SafexConfig;
extern RandomX_ConfigurationKeva RandomX_KevaConfig;

# 定义了一个模板函数 randomx_apply_config，用于应用配置
template<typename T>
void randomx_apply_config(const T& config)
{
    # 静态断言，确保传入的配置结构体大小与 RandomX_ConfigurationBase 相同
    static_assert(sizeof(T) == sizeof(RandomX_ConfigurationBase), "Invalid RandomX configuration struct size");
    # 静态断言，确保传入的配置结构体是 RandomX_ConfigurationBase 的子类
    static_assert(std::is_base_of<RandomX_ConfigurationBase, T>::value, "Incompatible RandomX configuration struct");
    # 将传入的配置应用到全局变量 RandomX_CurrentConfig
    RandomX_CurrentConfig = config;
    # 调用配置的 Apply 方法
    RandomX_CurrentConfig.Apply();
}

# 定义了一系列函数，用于设置不同的参数
void randomx_set_scratchpad_prefetch_mode(int mode);
void randomx_set_huge_pages_jit(bool hugePages);
void randomx_set_optimized_dataset_init(int value);

# 如果是 C++ 环境，则使用 extern "C" 包裹以下代码
#if defined(__cplusplus)
extern "C" {
#endif
/**
 * 创建一个 randomx_cache 结构并为 RandomX 缓存分配内存。
 *
 * @param flags 是以下两个标志的任意组合（每个标志可以设置或不设置）：
 *        RANDOMX_FLAG_LARGE_PAGES - 在大页中分配内存
 *        RANDOMX_FLAG_JIT - 创建具有 JIT 编译支持的缓存结构；这使得后续的数据集初始化更快
 *
 * @return 指向已分配的 randomx_cache 结构的指针。
 *         如果内存分配失败或者设置了 RANDOMX_FLAG_JIT 并且当前平台不支持 JIT 编译，则返回 NULL。
 */
RANDOMX_EXPORT randomx_cache *randomx_create_cache(randomx_flags flags, uint8_t *memory);

/**
 * 使用提供的密钥值初始化缓存内存和 SuperscalarHash。
 *
 * @param cache 是指向先前分配的 randomx_cache 结构的指针。必须不为 NULL。
 * @param key 是指向包含密钥值的内存的指针。必须不为 NULL。
 * @param keySize 是密钥的字节数。
*/
RANDOMX_EXPORT void randomx_init_cache(randomx_cache *cache, const void *key, size_t keySize);

/**
 * 释放 randomx_cache 结构占用的所有内存。
 *
 * @param cache 是指向先前分配的 randomx_cache 结构的指针。
*/
RANDOMX_EXPORT void randomx_release_cache(randomx_cache* cache);

/**
 * 创建一个 randomx_dataset 结构并为 RandomX 数据集分配内存。
 *
 * @param flags 是初始化标志。只支持一个标志（可以设置或不设置）：
 *        RANDOMX_FLAG_LARGE_PAGES - 在大页中分配内存
 *
 * @return 指向已分配的 randomx_dataset 结构的指针。
 *         如果内存分配失败，则返回 NULL。
 */
RANDOMX_EXPORT randomx_dataset *randomx_create_dataset(uint8_t *memory);

/**
 * 获取数据集中包含的项目数。
 *
 * @return 数据集中包含的项目数。
*/
# 返回数据集中的项目数量
RANDOMX_EXPORT unsigned long randomx_dataset_item_count(void);

/**
 * 初始化数据集项目。
 *
 * 注意：为了使用数据集，必须初始化从0到(randomx_dataset_item_count() - 1)的所有项目。
 * 可以通过多次调用此函数来使用非重叠的项目序列进行初始化。
 *
 * @param dataset 是一个指向先前分配的randomx_dataset结构的指针。不能为NULL。
 * @param cache 是一个指向先前分配和初始化的randomx_cache结构的指针。不能为NULL。
 * @param startItem 是初始化应该开始的项目编号。
 * @param itemCount 是应该初始化的项目数量。
*/
RANDOMX_EXPORT void randomx_init_dataset(randomx_dataset *dataset, randomx_cache *cache, unsigned long startItem, unsigned long itemCount);

/**
 * 返回数据集结构的内部内存缓冲区的指针。内部内存缓冲区的大小为randomx_dataset_item_count() * RANDOMX_DATASET_ITEM_SIZE。
 *
 * @param dataset 是一个指向先前分配的randomx_dataset结构的指针。不能为NULL。
 *
 * @return 数据集结构的内部内存缓冲区的指针。
*/
RANDOMX_EXPORT void *randomx_get_dataset_memory(randomx_dataset *dataset);

/**
 * 释放randomx_dataset结构占用的所有内存。
 *
 * @param dataset 是一个指向先前分配的randomx_dataset结构的指针。
*/
RANDOMX_EXPORT void randomx_release_dataset(randomx_dataset *dataset);
/**
 * 创建并初始化一个 RandomX 虚拟机。
 *
 * @param flags 是以下 4 个标志的任意组合（每个标志可以设置或不设置）：
 *        RANDOMX_FLAG_LARGE_PAGES - 在大页中分配临时存储器
 *        RANDOMX_FLAG_HARD_AES - 虚拟机将使用硬件加速的 AES
 *        RANDOMX_FLAG_FULL_MEM - 虚拟机将使用完整的数据集
 *        RANDOMX_FLAG_JIT - 虚拟机将使用 JIT 编译器
 *        标志的数值被排序，使得数值越高提供更快的哈希计算，数值越低提供更高的可移植性。
 *        使用 RANDOMX_FLAG_DEFAULT（所有标志都未设置）可以在所有平台上运行，但速度最慢。
 * @param cache 是指向已初始化的 randomx_cache 结构的指针。如果设置了 RANDOMX_FLAG_FULL_MEM，则可以为 NULL。
 * @param dataset 是指向 randomx_dataset 结构的指针。如果未设置 RANDOMX_FLAG_FULL_MEM，则可以为 NULL。
 *
 * @return 指向已初始化的 randomx_vm 结构的指针。
 *         如果：
 *         （1）临时存储器分配失败。
 *         （2）当前平台不支持请求的初始化标志。
 *         （3）cache 参数为 NULL 并且未设置 RANDOMX_FLAG_FULL_MEM。
 *         （4）dataset 参数为 NULL 并且设置了 RANDOMX_FLAG_FULL_MEM。
 *         则返回 NULL。
*/
RANDOMX_EXPORT randomx_vm *randomx_create_vm(randomx_flags flags, randomx_cache *cache, randomx_dataset *dataset, uint8_t *scratchpad, uint32_t node);

/**
 * 使用新的 Cache 重新初始化虚拟机。每当 Cache 使用新的密钥重新初始化时，应调用此函数。
 *
 * @param machine 是指向已初始化的 randomx_vm 结构的指针，该结构是在没有设置 RANDOMX_FLAG_FULL_MEM 的情况下初始化的。不能为空。
 * @param cache 是指向已初始化的 randomx_cache 结构的指针。不能为空。
*/
RANDOMX_EXPORT void randomx_vm_set_cache(randomx_vm *machine, randomx_cache* cache);
/**
 * 重新初始化虚拟机，使用新的数据集
 *
 * @param machine 是一个指向 randomx_vm 结构的指针，该结构使用 RANDOMX_FLAG_FULL_MEM 进行初始化。不得为 NULL。
 * @param dataset 是一个指向已初始化的 randomx_dataset 结构的指针。不得为 NULL。
*/
RANDOMX_EXPORT void randomx_vm_set_dataset(randomx_vm *machine, randomx_dataset *dataset);

/**
 * 释放 randomx_vm 结构占用的所有内存
 *
 * @param machine 是一个指向先前创建的 randomx_vm 结构的指针。
*/
RANDOMX_EXPORT void randomx_destroy_vm(randomx_vm *machine);

/**
 * 计算 RandomX 哈希值
 *
 * @param machine 是一个指向 randomx_vm 结构的指针。不得为 NULL。
 * @param input 是要进行哈希的内存的指针。不得为 NULL。
 * @param inputSize 是要进行哈希的字节数。
 * @param output 是哈希将被存储的内存的指针。不得为 NULL，并且至少必须有 RANDOMX_HASH_SIZE 字节可用于写入。
*/
RANDOMX_EXPORT void randomx_calculate_hash(randomx_vm *machine, const void *input, size_t inputSize, void *output);

RANDOMX_EXPORT void randomx_calculate_hash_first(randomx_vm* machine, uint64_t (&tempHash)[8], const void* input, size_t inputSize);
RANDOMX_EXPORT void randomx_calculate_hash_next(randomx_vm* machine, uint64_t (&tempHash)[8], const void* nextInput, size_t nextInputSize, void* output);

#if defined(__cplusplus)
}
#endif

#endif
```