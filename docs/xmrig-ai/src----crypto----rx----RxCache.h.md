# `xmrig\src\crypto\rx\RxCache.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2019 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何更高版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RX_CACHE_H
#define XMRIG_RX_CACHE_H


#include <cstdint>


#include "base/tools/Buffer.h"
#include "base/tools/Object.h"
#include "crypto/common/HugePagesInfo.h"
#include "crypto/randomx/configuration.h"


struct randomx_cache;


namespace xmrig
{


class RxCache
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(RxCache)

    RxCache(bool hugePages, uint32_t nodeId);
    RxCache(uint8_t *memory);
    ~RxCache();

    inline bool isJIT() const               { return m_jit; }
    inline const Buffer &seed() const       { return m_seed; }
    inline randomx_cache *get() const       { return m_cache; }
    # 返回缓存的最大大小
        inline size_t size() const              { return maxSize(); }
    
        # 使用给定的种子初始化缓存
        bool init(const Buffer &seed);
        # 返回HugePagesInfo对象
        HugePagesInfo hugePages() const;
    
        # 返回缓存的最大大小
        static inline constexpr size_t maxSize() { return RANDOMX_CACHE_MAX_SIZE; }
# 声明私有成员变量和方法
private:
    # 声明一个名为create的私有方法，参数为uint8_t类型的指针memory
    void create(uint8_t *memory);

    # 声明一个名为m_jit的布尔类型成员变量，并初始化为true
    bool m_jit              = true;
    # 声明一个名为m_seed的Buffer类型成员变量
    Buffer m_seed;
    # 声明一个名为m_cache的randomx_cache类型指针成员变量，并初始化为nullptr
    randomx_cache *m_cache  = nullptr;
    # 声明一个名为m_memory的VirtualMemory类型指针成员变量，并初始化为nullptr
    VirtualMemory *m_memory = nullptr;
};


} /* namespace xmrig */
# 结束命名空间xmrig
#endif /* XMRIG_RX_CACHE_H */
# 结束条件编译指令```
```