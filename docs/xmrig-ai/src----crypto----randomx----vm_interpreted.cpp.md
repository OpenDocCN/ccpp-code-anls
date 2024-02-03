# `xmrig\src\crypto\randomx\vm_interpreted.cpp`

```cpp
/*
版权声明，允许在源代码和二进制形式下重新分发和使用，但需要满足以下条件：
- 在源代码的重新分发中需要保留上述版权声明、条件列表和以下免责声明
- 在二进制形式的重新分发中需要在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
- 不得使用版权持有者的名称或其贡献者的名称，未经特定事先书面许可，来认可或推广从本软件衍生的产品

本软件由版权持有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的情况下，版权持有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害的可能性。
*/

#include "crypto/randomx/vm_interpreted.hpp"  // 包含虚拟机解释器头文件
#include "crypto/randomx/dataset.hpp"  // 包含数据集头文件
#include "crypto/randomx/intrin_portable.h"  // 包含可移植指令头文件
#include "crypto/randomx/reciprocal.h"  // 包含倒数头文件

namespace randomx {

    template<int softAes>
    void InterpretedVm<softAes>::setDataset(randomx_dataset* dataset) {  // 设置数据集的方法
        datasetPtr = dataset;  // 将数据集指针设置为传入的数据集指针
        mem.memory = dataset->memory;  // 将内存设置为数据集的内存
    }

    template<int softAes>
    // 运行解释虚拟机，生成程序并执行
    void InterpretedVm<softAes>::run(void* seed) {
        // 生成程序
        VmBase<softAes>::generateProgram(seed);
        // 初始化随机数生成器
        randomx_vm::initialize();
        // 执行程序
        execute();
    }
    
    // 读取数据集中的数据到寄存器数组中
    template<int softAes>
    void InterpretedVm<softAes>::datasetRead(uint64_t address, int_reg_t(&r)[RegistersCount]) {
        // 获取数据集中的一行数据
        uint64_t* datasetLine = (uint64_t*)(mem.memory + address);
        // 将数据集中的数据与寄存器数组中的数据进行异或操作
        for (int i = 0; i < RegistersCount; ++i)
            r[i] ^= datasetLine[i];
    }
    
    // 预取数据集中的数据
    template<int softAes>
    void InterpretedVm<softAes>::datasetPrefetch(uint64_t address) {
        // 使用非临近预取指令预取数据集中的数据
        rx_prefetch_nta(mem.memory + address);
    }
    
    // 实例化InterpretedVm类模板，分别使用false和true作为模板参数
    template class InterpretedVm<false>;
    template class InterpretedVm<true>;
这是一个闭合的大括号，可能是代码块的结束。
```