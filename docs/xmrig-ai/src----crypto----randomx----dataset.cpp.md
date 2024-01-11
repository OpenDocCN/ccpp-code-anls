# `xmrig\src\crypto\randomx\dataset.cpp`

```
/*
版权声明：
版权所有 (c) 2018-2020, tevador    <tevador@gmail.com>
版权所有 (c) 2019-2020, SChernykh  <https://github.com/SChernykh>
版权所有 (c) 2019-2020, XMRig      <https://github.com/xmrig>, <support@xmrig.com>

保留所有权利。

在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
    * 必须保留源代码中的上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式中，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或其贡献者的名称来认可或推广从本软件派生的产品。

本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他）的任何理论，即使事先已被告知可能发生此类损害。
*/

/* 原始代码来自于 Argon2 参考源代码包，使用 CC0 许可证
 * https://github.com/P-H-C/phc-winner-argon2
 * 版权所有 2015
 * Daniel Dinu, Dmitry Khovratovich, Jean-Philippe Aumasson 和 Samuel Neves
*/

#include <new> // 包含 new 头文件，用于动态内存分配
#include <algorithm> // 包含 algorithm 头文件，用于提供各种算法操作
#include <stdexcept> // 包含 stdexcept 头文件，用于提供异常类
*/
#include <cstring>  // 包含cstring头文件，提供字符串操作函数
#include <limits>  // 包含limits头文件，提供数值范围的定义
#include <cstring>  // 包含cstring头文件，提供字符串操作函数

#include "crypto/randomx/common.hpp"  // 包含common.hpp头文件，提供随机数生成相关的函数和结构体
#include "crypto/randomx/dataset.hpp"  // 包含dataset.hpp头文件，提供数据集生成相关的函数和结构体
#include "crypto/randomx/virtual_memory.hpp"  // 包含virtual_memory.hpp头文件，提供虚拟内存管理相关的函数和结构体
#include "crypto/randomx/superscalar.hpp"  // 包含superscalar.hpp头文件，提供超标量相关的函数和结构体
#include "crypto/randomx/blake2_generator.hpp"  // 包含blake2_generator.hpp头文件，提供blake2哈希生成相关的函数和结构体
#include "crypto/randomx/reciprocal.h"  // 包含reciprocal.h头文件，提供倒数计算相关的函数和结构体
#include "crypto/randomx/blake2/endian.h"  // 包含endian.h头文件，提供字节序转换相关的函数和结构体
#include "crypto/randomx/jit_compiler.hpp"  // 包含jit_compiler.hpp头文件，提供即时编译相关的函数和结构体
#include "crypto/randomx/intrin_portable.h"  // 包含intrin_portable.h头文件，提供可移植的内联汇编函数

#include "3rdparty/argon2/include/argon2.h"  // 包含argon2.h头文件，提供argon2哈希函数
#include "3rdparty/argon2/lib/core.h"  // 包含core.h头文件，提供argon2哈希函数的核心实现

//static_assert(RANDOMX_ARGON_MEMORY % (RANDOMX_ARGON_LANES * ARGON2_SYNC_POINTS) == 0, "RANDOMX_ARGON_MEMORY - invalid value");
// 静态断言，判断RANDOMX_ARGON_MEMORY是否是RANDOMX_ARGON_LANES * ARGON2_SYNC_POINTS的倍数，如果不是则输出错误信息

static_assert(ARGON2_BLOCK_SIZE == randomx::ArgonBlockSize, "Unexpected value of ARGON2_BLOCK_SIZE");
// 静态断言，判断ARGON2_BLOCK_SIZE是否等于randomx::ArgonBlockSize，如果不是则输出错误信息

namespace randomx {

    template<class Allocator>
    void deallocCache(randomx_cache* cache) {
        // 如果cache的memory成员不为空指针
        if (cache->memory != nullptr) {
            // 释放cache的memory成员指向的内存块
            Allocator::freeMemory(cache->memory, RANDOMX_CACHE_MAX_SIZE);
        }

        // 删除cache的jit成员指向的对象
        delete cache->jit;
    }

    // 实例化deallocCache函数模板，使用DefaultAllocator作为模板参数
    template void deallocCache<DefaultAllocator>(randomx_cache* cache);
    // 实例化deallocCache函数模板，使用LargePageAllocator作为模板参数
    template void deallocCache<LargePageAllocator>(randomx_cache* cache);
}


注释：
    # 初始化缓存，使用给定的密钥和密钥大小
    void initCache(randomx_cache* cache, const void* key, size_t keySize) {
        # 创建 Argon2 上下文对象
        argon2_context context;
    
        # 初始化 Argon2 上下文对象的各个属性
        # 输出指针为空
        context.out = nullptr;
        # 输出长度为0
        context.outlen = 0;
        # 密码为给定的密钥
        context.pwd = CONST_CAST(uint8_t *)key;
        # 密码长度为给定的密钥大小
        context.pwdlen = (uint32_t)keySize;
        # 盐值为预定义的 ArgonSalt
        context.salt = CONST_CAST(uint8_t *)RandomX_CurrentConfig.ArgonSalt;
        # 盐值长度为预定义 ArgonSalt 的长度
        context.saltlen = (uint32_t)strlen(RandomX_CurrentConfig.ArgonSalt);
        # 密钥为 nullptr
        context.secret = nullptr;
        # 密钥长度为0
        context.secretlen = 0;
        # 附加数据为 nullptr
        context.ad = nullptr;
        # 附加数据长度为0
        context.adlen = 0;
        # 迭代次数为预定义的 ArgonIterations
        context.t_cost = RandomX_CurrentConfig.ArgonIterations;
        # 内存大小为预定义的 ArgonMemory
        context.m_cost = RandomX_CurrentConfig.ArgonMemory;
        # 并行线程数为预定义的 ArgonLanes
        context.lanes = RandomX_CurrentConfig.ArgonLanes;
        # 线程数为1
        context.threads = 1;
        # 分配内存的回调函数为空
        context.allocate_cbk = nullptr;
        # 释放内存的回调函数为空
        context.free_cbk = nullptr;
        # 标志位为 ARGON2_DEFAULT_FLAGS
        context.flags = ARGON2_DEFAULT_FLAGS;
        # 版本号为 ARGON2_VERSION_NUMBER
        context.version = ARGON2_VERSION_NUMBER;
    
        # 使用 Argon2 上下文对象生成缓存的内存
        argon2_ctx_mem(&context, Argon2_d, cache->memory, RandomX_CurrentConfig.ArgonMemory * 1024);
    
        # 使用给定的密钥和密钥大小创建 Blake2 生成器对象
        randomx::Blake2Generator gen(key, keySize);
        # 生成缓存中的超标量程序
        for (uint32_t i = 0; i < RandomX_CurrentConfig.CacheAccesses; ++i) {
            randomx::generateSuperscalar(cache->programs[i], gen);
        }
    }
    
    # 初始化缓存并编译
    void initCacheCompile(randomx_cache* cache, const void* key, size_t keySize) {
        # 调用 initCache 函数初始化缓存
        initCache(cache, key, keySize);
# 如果定义了 XMRIG_SECURE_JIT，则启用 JIT 缓存的写入功能
ifdef XMRIG_SECURE_JIT
    cache->jit->enableWriting();
endif

# 生成超标量哈希函数的 JIT 代码
cache->jit->generateSuperscalarHash(cache->programs);
# 生成数据集初始化代码的 JIT 代码
cache->jit->generateDatasetInitCode();
# 获取数据集初始化函数的指针
cache->datasetInit  = cache->jit->getDatasetInitFunc();

# 如果定义了 XMRIG_SECURE_JIT，则启用 JIT 缓存的执行功能
ifdef XMRIG_SECURE_JIT
    cache->jit->enableExecution();
endif
}

# 定义超标量哈希函数的乘法常量
constexpr uint64_t superscalarMul0 = 6364136223846793005ULL;
# 定义超标量哈希函数的加法常量
constexpr uint64_t superscalarAdd1 = 9298411001130361340ULL;
constexpr uint64_t superscalarAdd2 = 12065312585734608966ULL;
constexpr uint64_t superscalarAdd3 = 9306329213124626780ULL;
constexpr uint64_t superscalarAdd4 = 5281919268842080866ULL;
constexpr uint64_t superscalarAdd5 = 10536153434571861004ULL;
constexpr uint64_t superscalarAdd6 = 3398623926847679864ULL;
constexpr uint64_t superscalarAdd7 = 9549104520008361294ULL;

# 定义获取混合块的函数，根据寄存器值和内存地址计算得到
static inline uint8_t* getMixBlock(uint64_t registerValue, uint8_t *memory) {
    # 计算掩码，用于限制内存地址范围
    const uint32_t mask = (RandomX_CurrentConfig.ArgonMemory * randomx::ArgonBlockSize) / CacheLineSize - 1;
    # 根据寄存器值和掩码计算得到混合块的内存地址
    return memory + (registerValue & mask) * CacheLineSize;
}
    // 初始化数据集中的一个项目
    void initDatasetItem(randomx_cache* cache, uint8_t* out, uint64_t itemNumber) {
        int_reg_t rl[8];  // 定义一个包含8个寄存器的数组
        uint8_t* mixBlock;  // 定义一个指向uint8_t类型的指针
        uint64_t registerValue = itemNumber;  // 将itemNumber赋给registerValue
        rl[0] = (itemNumber + 1) * superscalarMul0;  // 计算并赋值给rl[0]
        rl[1] = rl[0] ^ superscalarAdd1;  // 计算并赋值给rl[1]
        rl[2] = rl[0] ^ superscalarAdd2;  // 计算并赋值给rl[2]
        rl[3] = rl[0] ^ superscalarAdd3;  // 计算并赋值给rl[3]
        rl[4] = rl[0] ^ superscalarAdd4;  // 计算并赋值给rl[4]
        rl[5] = rl[0] ^ superscalarAdd5;  // 计算并赋值给rl[5]
        rl[6] = rl[0] ^ superscalarAdd6;  // 计算并赋值给rl[6]
        rl[7] = rl[0] ^ superscalarAdd7;  // 计算并赋值给rl[7]
        for (unsigned i = 0; i < RandomX_CurrentConfig.CacheAccesses; ++i) {  // 循环执行CacheAccesses次
            mixBlock = getMixBlock(registerValue, cache->memory);  // 调用getMixBlock函数并将结果赋给mixBlock
            rx_prefetch_nta(mixBlock);  // 执行数据预取操作
            SuperscalarProgram& prog = cache->programs[i];  // 获取cache中的程序

            executeSuperscalar(rl, prog);  // 执行超标量程序

            for (unsigned q = 0; q < 8; ++q)  // 循环8次
                rl[q] ^= load64_native(mixBlock + 8 * q);  // 对rl[q]进行异或操作

            registerValue = rl[prog.getAddressRegister()];  // 获取地址寄存器的值赋给registerValue
        }

        memcpy(out, &rl, CacheLineSize);  // 将rl的内容复制到out中，长度为CacheLineSize
    }

    // 初始化数据集
    void initDataset(randomx_cache* cache, uint8_t* dataset, uint32_t startItem, uint32_t endItem) {
        for (uint32_t itemNumber = startItem; itemNumber < endItem; ++itemNumber, dataset += CacheLineSize)  // 循环处理数据集中的项目
            initDatasetItem(cache, dataset, itemNumber);  // 调用initDatasetItem函数
    }
# 闭合前面的函数定义
```