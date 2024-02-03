# `xmrig\src\base\net\tools\MemPool.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MEMPOOL_H
#define XMRIG_MEMPOOL_H


#include <array>
#include <cassert>
#include <cstddef>
#include <map>
#include <set>


namespace xmrig {


template<size_t CHUNK_SIZE, size_t INIT_SIZE>
class MemPool
{
public:
    // 默认构造函数
    MemPool() = default;


    // 返回块大小
    constexpr size_t chunkSize() const  { return CHUNK_SIZE; }
    // 返回空闲空间大小
    inline size_t freeSize() const      { return m_free.size() * CHUNK_SIZE; }
    // 返回总大小
    inline size_t size() const          { return m_data.size() * CHUNK_SIZE * INIT_SIZE; }


    // 分配内存
    inline char *allocate()
    {
        // 如果没有空闲空间，则调整大小
        if (m_free.empty()) {
            resize();
        }

        const size_t i = *m_free.begin();
        const size_t r = i / INIT_SIZE;

        char *ptr = m_data[r].data() + (i - r * INIT_SIZE) * CHUNK_SIZE;

        m_used.insert({ ptr, i });
        m_free.erase(i);

        return ptr;
    }


    // 释放内存
    inline void deallocate(const char *ptr)
    {
        if (ptr == nullptr) {
            return;
        }

        assert(m_used.count(ptr));

        m_free.emplace(m_used[ptr]);
        m_used.erase(ptr);
    }


private:
    // 调整大小函数
    inline void resize()
    # 定义一个常量 index，其值为 m_data 的大小
    const size_t index = m_data.size();
    # 访问 m_data 中索引为 index 的元素，但是没有对其进行任何操作
    m_data[index];

    # 遍历 INIT_SIZE 次，将 (index * INIT_SIZE) + i 的值添加到 m_free 中
    for (size_t i = 0; i < INIT_SIZE; ++i) {
        m_free.emplace((index * INIT_SIZE) + i);
    }


    # 定义一个映射，将字符指针映射到大小为 size_t 的值
    std::map<const char *, size_t> m_used;
    # 定义一个映射，将大小为 size_t 的值映射到大小为 CHUNK_SIZE * INIT_SIZE 的字符数组
    std::map<size_t, std::array<char, CHUNK_SIZE * INIT_SIZE> > m_data;
    # 定义一个集合，用于存储大小为 size_t 的值
    std::set<size_t> m_free;
}; 
// 结束了xmrig命名空间的定义

} /* namespace xmrig */
// 结束了xmrig命名空间的声明

#endif /* XMRIG_MEMPOOL_H */
// 结束了XMRIG_MEMPOOL_H的条件编译
```