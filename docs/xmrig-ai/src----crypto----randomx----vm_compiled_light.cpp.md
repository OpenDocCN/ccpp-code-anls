# `xmrig\src\crypto\randomx\vm_compiled_light.cpp`

```
/*
版权声明：
版权所有 (c) 2018-2020, tevador    <tevador@gmail.com>
版权所有 (c) 2019-2020, SChernykh  <https://github.com/SChernykh>
版权所有 (c) 2019-2020, XMRig      <https://github.com/xmrig>, <support@xmrig.com>

保留所有权利。

在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
    * 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
    * 以二进制形式重新分发必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或其贡献者的名称来认可或推广从本软件派生的产品。

本软件按“原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他）产生的任何理论，即使事先已被告知可能发生此类损害。
*/

#include "crypto/randomx/vm_compiled_light.hpp"
#include "crypto/randomx/common.hpp"
#include <stdexcept>

namespace randomx {

    // 设置缓存指针
    template<int softAes>
    void CompiledLightVm<softAes>::setCache(randomx_cache* cache) {
        // 将缓存指针赋值给cachePtr
        cachePtr = cache;
        // 将缓存的内存指针赋值给虚拟机的内存指针
        mem.memory = cache->memory;
# 如果定义了 XMRIG_SECURE_JIT，则启用编译器的写入功能
ifdef XMRIG_SECURE_JIT
    compiler.enableWriting();
endif

# 生成超标量哈希
compiler.generateSuperscalarHash(cache->programs);
}

# 运行虚拟机
template<int softAes>
void CompiledLightVm<softAes>::run(void* seed) {
    # 生成程序
    VmBase<softAes>::generateProgram(seed);
    # 初始化随机数生成器
    randomx_vm::initialize();

    # 如果定义了 XMRIG_SECURE_JIT，则启用编译器的写入功能
    ifdef XMRIG_SECURE_JIT
        compiler.enableWriting();
    endif

    # 生成轻量级程序
    compiler.generateProgramLight(program, config, datasetOffset);

    # 执行编译后的虚拟机
    CompiledVm<softAes>::execute();
}

# 实例化编译后的轻量级虚拟机类
template class CompiledLightVm<false>;
template class CompiledLightVm<true>;
}
```