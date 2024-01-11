# `xmrig\src\crypto\cn\CnHash.h`

```
// XMRig版权声明
/* XMRig
 * Copyright 2018      Lee Clagett <https://github.com/vtnerd>
 * Copyright 2018-2021 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它，遵循GNU通用公共许可证的条款，由自由软件基金会发布，可以选择遵循许可证的第3版或（自行选择）任何更新版本。
 *
 *   本程序是基于希望它能有用的前提下发布的，但没有任何担保；甚至没有适销性或特定用途的暗示担保。更多详情请参阅GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本。如果没有，请参阅<http://www.gnu.org/licenses/>。
 */

// 如果未定义XMRig_CN_HASH_H，则定义XMRig_CN_HASH_H
#ifndef XMRIG_CN_HASH_H
#define XMRIG_CN_HASH_H

// 包含标准库头文件
#include <cstddef>
#include <cstdint>
#include <map>

// 包含自定义头文件
#include "crypto/cn/CnAlgo.h"
#include "crypto/common/Assembly.h"

// 定义cryptonight_ctx结构体
struct cryptonight_ctx;

// 命名空间xmrig
namespace xmrig
{

// 定义cn_hash_fun和cn_mainloop_fun函数指针类型
using cn_hash_fun     = void (*)(const uint8_t *, size_t, uint8_t *, cryptonight_ctx **, uint64_t);
using cn_mainloop_fun = void (*)(cryptonight_ctx **);

// 定义CnHash类
class CnHash
{
public:
    // 定义AlgoVariant枚举类型
    enum AlgoVariant {
        AV_AUTO,        // 自动模式
        AV_SINGLE,      // 单哈希模式
        AV_DOUBLE,      // 双哈希模式
        AV_SINGLE_SOFT, // 单哈希模式（软件AES）
        AV_DOUBLE_SOFT, // 双哈希模式（软件AES）
        AV_TRIPLE,      // 三重哈希模式
        AV_QUAD,        // 四重哈希模式
        AV_PENTA,       // 五重哈希模式
        AV_TRIPLE_SOFT, // 三重哈希模式（软件AES）
        AV_QUAD_SOFT,   // 四重哈希模式（软件AES）
        AV_PENTA_SOFT,  // 五重哈希模式（软件AES）
        AV_MAX          // 最大值
    };

    // 构造函数
    CnHash();
    # 虚拟析构函数，用于释放对象的资源
    virtual ~CnHash();

    # 静态方法，用于获取哈希函数
    # 参数：
    #   algorithm: 哈希算法
    #   av: 算法变体
    #   assembly: 汇编程序的ID
    # 返回值：哈希函数
    static cn_hash_fun fn(const Algorithm &algorithm, AlgoVariant av, Assembly::Id assembly);
// 声明私有结构体 cn_hash_fun_array，包含一个二维数组，数组大小为 AV_MAX * Assembly::MAX
private:
    struct cn_hash_fun_array {
        cn_hash_fun data[AV_MAX][Assembly::MAX];
    };

    // 声明一个 Algorithm 到 cn_hash_fun_array* 的映射的 map
    std::map<Algorithm, cn_hash_fun_array*> m_map;
};
// 结束命名空间 xmrig
} /* namespace xmrig */

// 结束条件编译指令，关闭 XMRIG_CN_HASH_H 宏定义
#endif /* XMRIG_CN_HASH_H */
```