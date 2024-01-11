# `xmrig\src\backend\common\benchmark\BenchState.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 要么许可证的第3版，要么
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/common/benchmark/BenchState.h"
#include "backend/common/benchmark/BenchState_test.h"
#include "backend/common/interfaces/IBenchListener.h"
#include "base/io/Async.h"
#include "base/tools/Chrono.h"

#include <algorithm>
#include <cassert>
#include <memory>
#include <mutex>

namespace xmrig {

// BenchStatePrivate类的定义
class BenchStatePrivate
{
public:
    // BenchStatePrivate类的构造函数
    BenchStatePrivate(IBenchListener *listener, uint32_t size) :
        listener(listener),
        size(size)
    {}

    // IBenchListener指针
    IBenchListener *listener;
    // 互斥锁
    std::mutex mutex;
    // 异步操作的共享指针
    std::shared_ptr<Async> async;
    // 剩余数量
    uint32_t remaining = 0;
    // 大小
    uint32_t size;
    // 完成时间
    uint64_t doneTime = 0;
};

// BenchStatePrivate指针
static BenchStatePrivate *d_ptr = nullptr;
// 原子类型的BenchState类数据
std::atomic<uint64_t> BenchState::m_data{};

} // namespace xmrig

// 检查BenchState是否完成
bool xmrig::BenchState::isDone()
{
    return d_ptr == nullptr;
}

// 获取BenchState的大小
uint32_t xmrig::BenchState::size()
{
    return d_ptr ? d_ptr->size : 0U;
}

// 获取算法的参考哈希
uint64_t xmrig::BenchState::referenceHash(const Algorithm &algo, uint32_t size, uint32_t threads)
{
    uint64_t hash = 0;

    try {
        const auto &h = (threads == 1) ? hashCheck1T : hashCheck;
        hash = h.at(algo).at(size);
    // 捕获任何标准异常并忽略
    } catch (const std::exception &ex) {}
    
    // 返回计算得到的哈希值
    return hash;
# 开始执行基准测试，设置线程数和后端
uint64_t xmrig::BenchState::start(size_t threads, const IBackend *backend)
{
    # 确保私有指针不为空
    assert(d_ptr != nullptr);

    # 设置剩余线程数为传入的线程数
    d_ptr->remaining = static_cast<uint32_t>(threads);

    # 创建一个异步对象，当异步执行完成时调用onBenchDone方法，并销毁对象
    d_ptr->async = std::make_shared<Async>([] {
        d_ptr->listener->onBenchDone(m_data, 0, d_ptr->doneTime);

        destroy();
    });

    # 获取当前时间戳
    const uint64_t ts = Chrono::steadyMSecs();
    # 调用onBenchReady方法，通知基准测试准备就绪
    d_ptr->listener->onBenchReady(ts, d_ptr->remaining, backend);

    # 返回时间戳
    return ts;
}

# 销毁BenchState对象
void xmrig::BenchState::destroy()
{
    # 删除私有指针指向的对象，并将指针置为空
    delete d_ptr;
    d_ptr = nullptr;
}

# 完成基准测试
void xmrig::BenchState::done()
{
    # 确保私有指针、异步对象和剩余线程数都不为空
    assert(d_ptr != nullptr && d_ptr->async && d_ptr->remaining > 0);

    # 获取当前时间戳
    const uint64_t ts = Chrono::steadyMSecs();

    # 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(d_ptr->mutex);

    # 更新完成时间为最大值，并减少剩余线程数
    d_ptr->doneTime = std::max(d_ptr->doneTime, ts);
    --d_ptr->remaining;

    # 如果剩余线程数为0，则发送异步信号
    if (d_ptr->remaining == 0) {
        d_ptr->async->send();
    }
}

# 初始化BenchState对象
void xmrig::BenchState::init(IBenchListener *listener, uint32_t size)
{
    # 确保私有指针为空
    assert(d_ptr == nullptr);

    # 创建一个新的BenchStatePrivate对象，并赋值给私有指针
    d_ptr = new BenchStatePrivate(listener, size);
}

# 设置基准测试的大小
void xmrig::BenchState::setSize(uint32_t size)
{
    # 确保私有指针不为空
    assert(d_ptr != nullptr);

    # 设置基准测试的大小
    d_ptr->size = size;
}
```