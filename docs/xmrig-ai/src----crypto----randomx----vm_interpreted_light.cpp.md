# `xmrig\src\crypto\randomx\vm_interpreted_light.cpp`

```
/*
版权声明，允许在源代码和二进制形式下重新分发和使用，需满足以下条件：
* 在源代码的重新分发中必须保留上述版权声明、条件列表和以下免责声明。
* 在二进制形式的重新分发中，必须在提供的文档和/或其他材料中复制上述版权声明、条件列表和以下免责声明。
* 不得使用版权持有者的名称或其贡献者的名称，来认可或推广从本软件派生的产品，除非事先得到特定的书面许可。

本软件由版权持有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权持有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）承担任何责任，即使已被告知可能发生此类损害的可能性。
*/
#include "crypto/randomx/vm_interpreted_light.hpp" // 包含解释轻量级虚拟机的头文件
#include "crypto/randomx/dataset.hpp" // 包含数据集的头文件

namespace randomx {

    template<int softAes>
    void InterpretedLightVm<softAes>::setCache(randomx_cache* cache) { // 设置缓存的函数
        cachePtr = cache; // 将缓存指针指向传入的缓存
        mem.memory = cache->memory; // 将内存指针指向缓存的内存
    }

    template<int softAes>
    // 从数据集中读取数据
    void InterpretedLightVm<softAes>::datasetRead(uint64_t address, int_reg_t(&r)[8]) {
        // 计算数据项编号
        uint32_t itemNumber = address / CacheLineSize;
        // 创建一个临时数组
        int_reg_t rl[8];
        
        // 初始化数据集中的数据项
        initDatasetItem(cachePtr, (uint8_t*)rl, itemNumber);

        // 将读取的数据与传入的数组进行异或操作
        for (unsigned q = 0; q < 8; ++q)
            r[q] ^= rl[q];
    }

    // 实例化模板类
    template class InterpretedLightVm<false>;
    template class InterpretedLightVm<true>;
这是一个闭合的大括号，可能是代码块的结束。
```