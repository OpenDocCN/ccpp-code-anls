# `xmrig\src\crypto\common\MemoryPool.h`

```
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
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何更高版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。详细信息请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MEMORYPOOL_H
#define XMRIG_MEMORYPOOL_H


#include "backend/common/interfaces/IMemoryPool.h"
#include "base/tools/Object.h"


namespace xmrig {


class VirtualMemory;


class MemoryPool : public IMemoryPool
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(MemoryPool)

    MemoryPool(size_t size, bool hugePages, uint32_t node = 0);
    ~MemoryPool() override;

protected:
    bool isHugePages(uint32_t node) const override;
    uint8_t *get(size_t size, uint32_t node) override;
    void release(uint32_t node) override;

private:
    size_t m_refs           = 0;
    size_t m_offset         = 0;
    size_t m_alignOffset    = 0;
    // 创建一个指向 VirtualMemory 对象的指针，并初始化为 nullptr
    VirtualMemory *m_memory = nullptr;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明 xmrig 命名空间的结束

#endif /* XMRIG_MEMORYPOOL_H */
// 结束了对 XMRIG_MEMORYPOOL_H 头文件的引用
```