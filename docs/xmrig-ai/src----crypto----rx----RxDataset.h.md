# `xmrig\src\crypto\rx\RxDataset.h`

```
// 包含版权声明和许可信息
/* XMRig
 * Copyright (c) 2018-2019 tevador     <tevador@gmail.com>
 * Copyright (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * Copyright (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 防止头文件重复包含
#ifndef XMRIG_RX_DATASET_H
#define XMRIG_RX_DATASET_H

// 包含所需的头文件
#include "base/tools/Buffer.h"
#include "base/tools/Object.h"
#include "crypto/common/HugePagesInfo.h"
#include "crypto/randomx/configuration.h"
#include "crypto/rx/RxConfig.h"

#include <atomic>

// 定义 randomx_dataset 结构体
struct randomx_dataset;

// 命名空间 xmrig
namespace xmrig
{

// 声明 RxCache 和 VirtualMemory 类
class RxCache;
class VirtualMemory;

// 定义 RxDataset 类
class RxDataset
{
public:
    // 禁用默认的拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(RxDataset)

    // 构造函数，接受巨大页面标志、1GB 页面标志、缓存标志、模式和节点编号作为参数
    RxDataset(bool hugePages, bool oneGbPages, bool cache, RxConfig::Mode mode, uint32_t node);
    // 构造函数，接受 RxCache 指针作为参数
    RxDataset(RxCache *cache);
    // 析构函数
    ~RxDataset();

    // 返回 randomx_dataset 指针
    inline randomx_dataset *get() const     { return m_dataset; }
    // 返回 RxCache 指针
    inline RxCache *cache() const           { return m_cache; }
    // 设置 RxCache 指针
    inline void setCache(RxCache *cache)    { m_cache = cache; }

    // 初始化数据集，接受种子、线程数和优先级作为参数
    bool init(const Buffer &seed, uint32_t numThreads, int priority);
    // 检查是否使用了巨大页面
    bool isHugePages() const;
    // 检查是否使用了 1GB 页面
    bool isOneGbPages() const;
    // 返回巨大页面信息
    HugePagesInfo hugePages(bool cache = true) const;
    // 返回数据集大小
    size_t size(bool cache = true) const;
    // 尝试分配内存作为 scratchpad
    uint8_t *tryAllocateScrathpad();
    // 返回原始数据集指针
    void *raw() const;
    // 设置原始数据集指针
    void setRaw(const void *raw);
    # 定义一个静态内联函数，返回RANDOMX_DATASET_MAX_SIZE的值作为最大大小的常量。
// 声明一个私有函数 allocate，该函数用于分配内存
private:
    void allocate(bool hugePages, bool oneGbPages);

// 声明一个私有成员变量 m_mode，类型为 RxConfig::Mode，初始值为 RxConfig::FastMode
    const RxConfig::Mode m_mode = RxConfig::FastMode;
// 声明一个私有成员变量 m_node，类型为 uint32_t，用于存储节点信息
    const uint32_t m_node;
// 声明一个指向 randomx_dataset 结构体的指针 m_dataset，初始值为 nullptr
    randomx_dataset *m_dataset  = nullptr;
// 声明一个指向 RxCache 类的指针 m_cache，初始值为 nullptr
    RxCache *m_cache            = nullptr;
// 声明一个变量 m_scratchpadLimit，类型为 size_t，用于存储 scratchpad 的限制大小
    size_t m_scratchpadLimit    = 0;
// 声明一个原子变量 m_scratchpadOffset，类型为 std::atomic<size_t>，用于存储 scratchpad 的偏移量
    std::atomic<size_t> m_scratchpadOffset{};
// 声明一个指向 VirtualMemory 类的指针 m_memory，初始值为 nullptr
    VirtualMemory *m_memory     = nullptr;
};


} /* namespace xmrig */


#endif /* XMRIG_RX_DATASET_H */
```