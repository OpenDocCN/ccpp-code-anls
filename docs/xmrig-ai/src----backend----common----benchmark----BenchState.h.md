# `xmrig\src\backend\common\benchmark\BenchState.h`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BENCHSTATE_H
#define XMRIG_BENCHSTATE_H


#include <atomic>
#include <cstddef>
#include <cstdint>


namespace xmrig {


class Algorithm;
class IBackend;
class IBenchListener;


class BenchState
{
public:
    static bool isDone(); // 检查基准测试是否完成
    static uint32_t size(); // 返回基准测试的大小
    static uint64_t referenceHash(const Algorithm &algo, uint32_t size, uint32_t threads); // 返回参考哈希值
    static uint64_t start(size_t threads, const IBackend *backend); // 开始基准测试
    static void destroy(); // 销毁基准测试
    static void done(); // 标记基准测试为完成
    static void init(IBenchListener *listener, uint32_t size); // 初始化基准测试
    static void setSize(uint32_t size); // 设置基准测试的大小

    inline static uint64_t data()           { return m_data; } // 返回数据
    inline static void add(uint64_t value)  { m_data.fetch_xor(value, std::memory_order_relaxed); } // 添加值

private:
    static std::atomic<uint64_t> m_data; // 原子类型的数据
};


} // namespace xmrig


#endif /* XMRIG_BENCHSTATE_H */
```