# `xmrig\src\crypto\randomx\vm_compiled.cpp`

```cpp
/*
版权声明，声明了对代码的使用和再分发的条件
*/
Copyright (c) 2018-2020, tevador    <tevador@gmail.com>
Copyright (c) 2019-2020, SChernykh  <https://github.com/SChernykh>
Copyright (c) 2019-2020, XMRig      <https://github.com/xmrig>, <support@xmrig.com>

保留所有权利。

在源代码和二进制形式中重新分发和使用，无论是否经过修改，都必须满足以下条件：
    * 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
    * 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有者或其贡献者的名称来认可或推广从本软件衍生的产品。

本软件由版权持有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权持有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。

*/

#include "crypto/randomx/vm_compiled.hpp"  // 引入 vm_compiled.hpp 文件
#include "crypto/randomx/common.hpp"  // 引入 common.hpp 文件
#include "crypto/rx/Profiler.h"  // 引入 Profiler.h 文件

namespace randomx {

    static_assert(sizeof(MemoryRegisters) == 2 * sizeof(addr_t) + sizeof(uintptr_t), "Invalid alignment of struct randomx::MemoryRegisters");  // 静态断言，检查 MemoryRegisters 结构体的对齐是否正确
    // 确保 RegisterFile 结构体的大小为 256，如果不是则输出错误信息
    static_assert(sizeof(RegisterFile) == 256, "Invalid alignment of struct randomx::RegisterFile");

    // 设置数据集指针
    template<int softAes>
    void CompiledVm<softAes>::setDataset(randomx_dataset* dataset) {
        datasetPtr = dataset;
    }

    // 运行虚拟机
    template<int softAes>
    void CompiledVm<softAes>::run(void* seed) {
        PROFILE_SCOPE(RandomX_run);

        // 准备编译器
        compiler.prepare();
        // 生成程序
        VmBase<softAes>::generateProgram(seed);
        // 初始化随机数虚拟机
        randomx_vm::initialize();
        // 生成程序
        compiler.generateProgram(program, config, randomx_vm::getFlags());
        // 设置内存
        mem.memory = datasetPtr->memory + datasetOffset;
        // 执行程序
        execute();
    }

    // 执行程序
    template<int softAes>
    void CompiledVm<softAes>::execute() {
        PROFILE_SCOPE(RandomX_JIT_execute);
    }
# 如果定义了 XMRIG_ARM，则将 config.eMask 的内容复制到 reg.f 中
#ifdef XMRIG_ARM
    memcpy(reg.f, config.eMask, sizeof(config.eMask));
#endif
    使用编译器获取程序函数，并传入参数 reg, mem, scratchpad, RandomX_CurrentConfig.ProgramIterations
    compiler.getProgramFunc()(reg, mem, scratchpad, RandomX_CurrentConfig.ProgramIterations);
}

# 实例化编译后的虚拟机类，分别传入 false 和 true 作为模板参数
template class CompiledVm<false>;
template class CompiledVm<true>;
```