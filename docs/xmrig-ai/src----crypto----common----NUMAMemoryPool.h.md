# `xmrig\src\crypto\common\NUMAMemoryPool.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用：
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NUMAMEMORYPOOL_H
#define XMRIG_NUMAMEMORYPOOL_H

#include "backend/common/interfaces/IMemoryPool.h"  // 包含 IMemoryPool 接口
#include "base/tools/Object.h"  // 包含 Object 工具

#include <map>  // 包含 map 头文件

namespace xmrig {

class IMemoryPool;  // 声明 IMemoryPool 类

class NUMAMemoryPool : public IMemoryPool  // 定义 NUMAMemoryPool 类，继承自 IMemoryPool
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(NUMAMemoryPool)  // 禁用默认的拷贝和移动构造函数

    NUMAMemoryPool(size_t size, bool hugePages);  // 构造函数，接受 size 和 hugePages 参数
    ~NUMAMemoryPool() override;  // 虚析构函数

protected:
    bool isHugePages(uint32_t node) const override;  // 重写 IMemoryPool 接口的 isHugePages 方法
    uint8_t *get(size_t size, uint32_t node) override;  // 重写 IMemoryPool 接口的 get 方法
    void release(uint32_t node) override;  // 重写 IMemoryPool 接口的 release 方法

private:
    IMemoryPool *get(uint32_t node) const;  // 声明私有的 get 方法，接受 node 参数并返回 IMemoryPool 指针
    # 声明一个返回 IMemoryPool 指针的函数，如果不存在则创建
    IMemoryPool *getOrCreate(uint32_t node) const;
    
    # 声明并初始化一个布尔类型的变量，表示是否使用大页
    bool m_hugePages        = true;
    
    # 声明并初始化一个 size_t 类型的变量，表示节点大小
    size_t m_nodeSize       = 0;
    
    # 声明并初始化一个 size_t 类型的变量，表示总大小
    size_t m_size           = 0;
    
    # 声明一个可变的映射，将节点号映射到对应的 IMemoryPool 指针
    mutable std::map<uint32_t, IMemoryPool *> m_map;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明 xmrig 命名空间的结束

#endif /* XMRIG_NUMAMEMORYPOOL_H */
// 结束了对 XMRIG_NUMAMEMORYPOOL_H 头文件的 ifndef 检查
```