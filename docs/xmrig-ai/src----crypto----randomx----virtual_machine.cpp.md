# `xmrig\src\crypto\randomx\virtual_machine.cpp`

```cpp
/*
版权声明：
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>
保留所有权利。
无论是源代码还是二进制形式的再分发和使用，都需要满足以下条件：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式的再分发中，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，
包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害
（包括但不限于替代商品或服务的采购、使用、数据或利润损失、业务中断等）承担任何责任，
无论是因合同、严格责任还是侵权行为（包括疏忽或其他）而引起的，即使事先已被告知此类损害的可能性。
*/
#include <cstring>  // 包含cstring头文件，提供字符串操作函数
#include <iomanip>  // 包含iomanip头文件，提供格式化输出函数
#include <stdexcept>  // 包含stdexcept头文件，提供异常处理类
#include "crypto/randomx/virtual_machine.hpp"  // 包含虚拟机类头文件
#include "crypto/randomx/aes_hash.hpp"  // 包含AES哈希类头文件
#include "crypto/randomx/allocator.hpp"  // 包含内存分配器类头文件
#include "crypto/randomx/blake2/blake2.h"  // 包含BLAKE2哈希函数头文件
#include "crypto/randomx/common.hpp"  // 包含通用函数头文件
#include "crypto/randomx/intrin_portable.h"  // 包含可移植的内联汇编函数头文件
#include "crypto/randomx/soft_aes.h"  // 包含软件AES加密函数头文件
#include "crypto/rx/Profiler.h"  // 包含性能分析器类头文件

randomx_vm::~randomx_vm() {
    // 虚拟机析构函数的实现
}

void randomx_vm::resetRoundingMode() {
    // 重置舍入模式的函数实现
    # 调用函数 rx_reset_float_state() 重置浮点状态
    rx_reset_float_state();
}

namespace randomx {

    // 获取小正浮点数的位表示
    static inline uint64_t getSmallPositiveFloatBits(uint64_t entropy) {
        // 将熵右移59位，得到指数部分
        auto exponent = entropy >> 59; //0..31
        // 将熵与尾数掩码进行与运算，得到尾数部分
        auto mantissa = entropy & mantissaMask;
        // 指数部分加上偏移量
        exponent += exponentBias;
        // 对指数部分进行掩码操作
        exponent &= exponentMask;
        // 将指数部分左移尾数大小位数
        exponent <<= mantissaSize;
        // 返回指数部分和尾数部分的组合
        return exponent | mantissa;
    }

    // 获取静态指数的位表示
    static inline uint64_t getStaticExponent(uint64_t entropy) {
        // 将常量指数位与熵进行或运算
        auto exponent = constExponentBits;
        // 将熵右移(64 - 静态指数位数)位，得到动态指数部分
        exponent |= (entropy >> (64 - staticExponentBits)) << dynamicExponentBits;
        // 将指数部分左移尾数大小位数
        exponent <<= mantissaSize;
        // 返回指数部分和尾数部分的组合
        return exponent;
    }

    // 获取浮点数掩码
    static inline uint64_t getFloatMask(uint64_t entropy) {
        // 定义一个22位的掩码
        constexpr uint64_t mask22bit = (1ULL << 22) - 1;
        // 将熵与22位掩码进行与运算，得到尾数部分
        return (entropy & mask22bit) | getStaticExponent(entropy);
    }

}

// 初始化虚拟机
void randomx_vm::initialize() {
    // 将第0个熵值转换为小正浮点数位表示，并存储到寄存器a[0]的低位
    store64(&reg.a[0].lo, randomx::getSmallPositiveFloatBits(program.getEntropy(0)));
    // 将第1个熵值转换为小正浮点数位表示，并存储到寄存器a[0]的高位
    store64(&reg.a[0].hi, randomx::getSmallPositiveFloatBits(program.getEntropy(1)));
    // 将第2个熵值转换为小正浮点数位表示，并存储到寄存器a[1]的低位
    store64(&reg.a[1].lo, randomx::getSmallPositiveFloatBits(program.getEntropy(2)));
    // 将第3个熵值转换为小正浮点数位表示，并存储到寄存器a[1]的高位
    store64(&reg.a[1].hi, randomx::getSmallPositiveFloatBits(program.getEntropy(3)));
    // 将第4个熵值转换为小正浮点数位表示，并存储到寄存器a[2]的低位
    store64(&reg.a[2].lo, randomx::getSmallPositiveFloatBits(program.getEntropy(4)));
    // 将第5个熵值转换为小正浮点数位表示，并存储到寄存器a[2]的高位
    store64(&reg.a[2].hi, randomx::getSmallPositiveFloatBits(program.getEntropy(5)));
    // 将第6个熵值转换为小正浮点数位表示，并存储到寄存器a[3]的低位
    store64(&reg.a[3].lo, randomx::getSmallPositiveFloatBits(program.getEntropy(6)));
    // 将第7个熵值转换为小正浮点数位表示，并存储到寄存器a[3]的高位
    store64(&reg.a[3].hi, randomx::getSmallPositiveFloatBits(program.getEntropy(7)));
    // 将第8个熵值与缓存行对齐掩码进行与运算，并存储到内存ma
    mem.ma = program.getEntropy(8) & CacheLineAlignMask;
    // 将第10个熵值存储到内存mx
    mem.mx = program.getEntropy(10);
    // 获取第12个熵值的地址寄存器
    auto addressRegisters = program.getEntropy(12);
    // 将地址寄存器的最低位存储到配置的读取寄存器0
    config.readReg0 = 0 + (addressRegisters & 1);
    // 将地址寄存器右移1位，将最低位存储到配置的读取寄存器1
    addressRegisters >>= 1;
    config.readReg1 = 2 + (addressRegisters & 1);
    addressRegisters >>= 1;
    config.readReg2 = 4 + (addressRegisters & 1);
    addressRegisters >>= 1;
    config.readReg3 = 6 + (addressRegisters & 1);
    # 根据程序的熵值计算数据集的偏移量，使用随机数生成器生成一个介于0和DatasetExtraItems之间的随机数，乘以CacheLineSize得到偏移量
    datasetOffset = (program.getEntropy(13) % (DatasetExtraItems + 1)) * randomx::CacheLineSize;
    
    # 使用程序的熵值生成一个浮点数掩码，并将其存储到config.eMask的第一个位置
    store64(&config.eMask[0], randomx::getFloatMask(program.getEntropy(14)));
    
    # 使用程序的熵值生成一个浮点数掩码，并将其存储到config.eMask的第二个位置
    store64(&config.eMask[1], randomx::getFloatMask(program.getEntropy(15)));
}

namespace randomx {

    template<int softAes>
    VmBase<softAes>::~VmBase() {
    }

    template<int softAes>
    void VmBase<softAes>::setScratchpad(uint8_t *scratchpad) {
        // 如果数据集指针为空，抛出无效参数异常
        if (datasetPtr == nullptr) {
            throw std::invalid_argument("Cache/Dataset not set");
        }

        // 设置虚拟机的scratchpad
        this->scratchpad = scratchpad;
    }

    template<int softAes>
    void VmBase<softAes>::getFinalResult(void* out) {
        // 使用hashAes1Rx4函数计算scratchpad的哈希值，存储在reg.a中
        hashAes1Rx4<softAes>(scratchpad, ScratchpadSize, &reg.a);
        // 调用rx_blake2b_wrapper的run函数，计算最终结果，存储在out中
        rx_blake2b_wrapper::run(out, RANDOMX_HASH_SIZE, &reg, sizeof(RegisterFile));
    }

    template<int softAes>
    void VmBase<softAes>::hashAndFill(void* out, uint64_t (&fill_state)[8]) {
        // 如果softAes为false，调用hashAndFillAes1Rx4函数计算哈希值并填充fill_state
        if (!softAes) {
            hashAndFillAes1Rx4<0, 2>(scratchpad, ScratchpadSize, &reg.a, fill_state);
        }
        // 如果softAes为true，调用GetSoftAESImpl函数获取软AES实现，计算哈希值并填充fill_state
        else {
            (*GetSoftAESImpl())(scratchpad, ScratchpadSize, &reg.a, fill_state);
        }

        // 调用rx_blake2b_wrapper的run函数，计算最终结果，存储在out中
        rx_blake2b_wrapper::run(out, RANDOMX_HASH_SIZE, &reg, sizeof(RegisterFile));
    }

    template<int softAes>
    void VmBase<softAes>::initScratchpad(void* seed) {
        // 使用fillAes1Rx4函数初始化scratchpad
        fillAes1Rx4<softAes>(seed, ScratchpadSize, scratchpad);
    }

    template<int softAes>
    void VmBase<softAes>::generateProgram(void* seed) {
        // 使用fillAes4Rx4函数生成程序
        PROFILE_SCOPE(RandomX_generate_program);
        fillAes4Rx4<softAes>(seed, 128 + RandomX_CurrentConfig.ProgramSize * 8, &program);
    }

    // 实例化VmBase类模板，分别传入false和true作为模板参数
    template class VmBase<false>;
    template class VmBase<true>;
}
```