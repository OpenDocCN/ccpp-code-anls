# `xmrig\src\crypto\randomx\blake2_generator.cpp`

```
/*
版权声明，版权所有，保留所有权利
在源代码和二进制形式下重新分发和使用，无论是否经过修改，都必须满足以下条件：
* 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
* 二进制形式的再分发必须在提供的文档和/或其他材料中复制上述版权声明、条件列表和以下免责声明。
* 不得使用版权持有人或其贡献者的名称来认可或推广从本软件衍生的产品，除非事先得到特定的书面许可。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权持有人或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害。
*/
#include <stddef.h>
#include "crypto/randomx/blake2/blake2.h"
#include "crypto/randomx/blake2/endian.h"
#include "crypto/randomx/blake2_generator.hpp"

namespace randomx {

    // 定义最大种子大小为60
    constexpr int maxSeedSize = 60;
    # 定义 Blake2Generator 类的构造函数，接受 seed、seedSize 和 nonce 作为参数
    Blake2Generator::Blake2Generator(const void* seed, size_t seedSize, int nonce) : dataIndex(sizeof(data)) {
        # 将 data 数组的所有元素初始化为 0
        memset(data, 0, sizeof(data));
        # 将 seed 数组的内容复制到 data 数组中，最多复制 maxSeedSize 个字节
        memcpy(data, seed, seedSize > maxSeedSize ? maxSeedSize : seedSize);
        # 将 nonce 存储到 data 数组的 maxSeedSize 位置
        store32(&data[maxSeedSize], nonce);
    }
    
    # 定义 getByte 方法，返回一个字节
    uint8_t Blake2Generator::getByte() {
        # 检查是否还有足够的数据可用，如果不够则重新生成数据
        checkData(1);
        # 返回 data 数组中的一个字节，并将 dataIndex 加一
        return data[dataIndex++];
    }
    
    # 定义 getUInt32 方法，返回一个 32 位无符号整数
    uint32_t Blake2Generator::getUInt32() {
        # 检查是否还有足够的数据可用，如果不够则重新生成数据
        checkData(4);
        # 从 data 数组中加载 32 位无符号整数，并将 dataIndex 加四
        auto ret = load32(&data[dataIndex]);
        dataIndex += 4;
        # 返回加载的整数
        return ret;
    }
    
    # 定义 checkData 方法，检查是否还有足够的数据可用
    void Blake2Generator::checkData(const size_t bytesNeeded) {
        # 如果 dataIndex 加上需要的字节数大于 data 数组的大小
        if (dataIndex + bytesNeeded > sizeof(data)) {
            # 使用 rx_blake2b 函数重新生成数据，并将 dataIndex 重置为 0
            rx_blake2b(data, sizeof(data), data, sizeof(data));
            dataIndex = 0;
        }
    }
# 闭合前面的函数定义
```