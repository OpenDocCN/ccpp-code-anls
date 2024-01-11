# `xmrig\src\crypto\rx\RxQueue.h`

```
# XMRig是一个开源的加密货币挖矿程序，此处为版权声明和许可证信息
/* XMRig
 * Copyright (c) 2018      Lee Clagett <https://github.com/vtnerd>
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

# 定义了XMRig的RxQueue类的头文件宏
#ifndef XMRIG_RX_QUEUE_H
#define XMRIG_RX_QUEUE_H

# 引入所需的头文件
#include "base/kernel/interfaces/IAsyncListener.h"
#include "base/tools/Object.h"
#include "crypto/common/HugePagesInfo.h"
#include "crypto/rx/RxConfig.h"
#include "crypto/rx/RxSeed.h"

# 引入所需的库
#include <condition_variable>
#include <mutex>
#include <thread>

# 声明命名空间xmrig
namespace xmrig
{

# 声明IRxListener和IRxStorage类
class IRxListener;
class IRxStorage;
class RxDataset;

# 定义RxQueueItem类
class RxQueueItem
{
public:
    # 构造函数，接收RxSeed、nodeset、threads、hugePages、oneGbPages、mode和priority参数
    RxQueueItem(const RxSeed &seed, const std::vector<uint32_t> &nodeset, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority) :
        hugePages(hugePages),
        oneGbPages(oneGbPages),
        priority(priority),
        mode(mode),
        seed(seed),
        nodeset(nodeset),
        threads(threads)
    {}

    # 定义成员变量
    const bool hugePages;
    const bool oneGbPages;
    const int priority;
    const RxConfig::Mode mode;
    const RxSeed seed;
    const std::vector<uint32_t> nodeset;
    const uint32_t threads;
};

# 定义RxQueue类，继承自IAsyncListener类
class RxQueue : public IAsyncListener
{
public:
    // 禁用拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE(RxQueue);

    // RxQueue 类的构造函数，接受一个 IRxListener 指针作为参数
    RxQueue(IRxListener *listener);

    // RxQueue 类的析构函数
    ~RxQueue() override;

    // 返回 HugePagesInfo 对象
    HugePagesInfo hugePages();

    // 返回 RxDataset 指针，接受一个 Job 对象和一个 nodeId 作为参数
    RxDataset *dataset(const Job &job, uint32_t nodeId);

    // 模板函数，接受一个 T 类型的参数，返回一个布尔值，判断是否准备就绪
    template<typename T> bool isReady(const T &seed);

    // 将 RxSeed 对象和节点集合、线程数、是否使用大页、是否使用 1GB 页、模式、优先级作为参数，将任务加入队列
    void enqueue(const RxSeed &seed, const std::vector<uint32_t> &nodeset, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority);
protected:
    // 当异步操作完成时调用 onReady() 方法
    inline void onAsync() override  { onReady(); }

private:
    // 定义状态枚举
    enum State {
        STATE_IDLE,
        STATE_PENDING,
        STATE_SHUTDOWN
    };

    // 定义模板函数，用于检查是否准备就绪
    template<typename T> bool isReadyUnsafe(const T &seed) const;
    // 后台初始化方法
    void backgroundInit();
    // 当准备就绪时调用的方法
    void onReady();

    // 监听器指针
    IRxListener *m_listener = nullptr;
    // 存储指针
    IRxStorage *m_storage   = nullptr;
    // 种子对象
    RxSeed m_seed;
    // 状态变量
    State m_state = STATE_IDLE;
    // 条件变量
    std::condition_variable m_cv;
    // 互斥锁
    std::mutex m_mutex;
    // 异步对象指针
    std::shared_ptr<Async> m_async;
    // 线程对象
    std::thread m_thread;
    // 队列对象
    std::vector<RxQueueItem> m_queue;
};


} /* namespace xmrig */


#endif /* XMRIG_RX_QUEUE_H */
```