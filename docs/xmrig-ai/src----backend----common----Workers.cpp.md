# `xmrig\src\backend\common\Workers.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证。
 *   有关更多详细信息，请参见GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/common/Workers.h"  // 引入Workers类的头文件
#include "backend/common/Hashrate.h"  // 引入Hashrate类的头文件
#include "backend/common/interfaces/IBackend.h"  // 引入IBackend接口的头文件
#include "backend/cpu/CpuWorker.h"  // 引入CpuWorker类的头文件
#include "base/io/log/Log.h"  // 引入Log类的头文件
#include "base/io/log/Tags.h"  // 引入Tags类的头文件
#include "base/tools/Chrono.h"  // 引入Chrono类的头文件

#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/OclWorker.h"  // 如果定义了XMRIG_FEATURE_OPENCL，则引入OclWorker类的头文件
#endif

#ifdef XMRIG_FEATURE_CUDA
#   include "backend/cuda/CudaWorker.h"  // 如果定义了XMRIG_FEATURE_CUDA，则引入CudaWorker类的头文件
#endif

#ifdef XMRIG_FEATURE_BENCHMARK
#   include "backend/common/benchmark/Benchmark.h"  // 如果定义了XMRIG_FEATURE_BENCHMARK，则引入Benchmark类的头文件
#endif

namespace xmrig {

class WorkersPrivate
{
public:
    XMRIG_DISABLE_COPY_MOVE(WorkersPrivate)  // 禁止复制和移动构造函数

    WorkersPrivate()    = default;  // 默认构造函数
    ~WorkersPrivate()   = default;  // 默认析构函数

    IBackend *backend   = nullptr;  // 初始化IBackend指针为空
    std::shared_ptr<Benchmark> benchmark;  // 初始化Benchmark智能指针为空
    std::shared_ptr<Hashrate> hashrate;  // 初始化Hashrate智能指针为空
};

} // namespace xmrig

template<class T>
xmrig::Workers<T>::Workers() :  // Workers类的构造函数
    d_ptr(new WorkersPrivate())  // 使用WorkersPrivate类的默认构造函数创建d_ptr
{

}

template<class T>
xmrig::Workers<T>::~Workers()  // Workers类的析构函数
{
    delete d_ptr;  // 释放d_ptr指针的内存
}

template<class T>
bool xmrig::Workers<T>::tick(uint64_t)  // Workers类的tick方法
{
    if (!d_ptr->hashrate) {  // 如果hashrate为空
        return true;  // 返回true
    }

    uint64_t ts             = Chrono::steadyMSecs();  // 获取当前时间戳（毫秒）
    # 声明并初始化一个布尔变量，表示总体是否可用
    bool totalAvailable     = true;
    # 声明并初始化一个无符号64位整数变量，表示总哈希数
    uint64_t totalHashCount = 0;
    # 声明并初始化一个无符号64位整数变量，表示哈希数
    uint64_t hashCount      = 0;
    # 声明并初始化一个无符号64位整数变量，表示原始哈希数
    uint64_t rawHashes      = 0;

    # 遍历线程池中的每个线程
    for (Thread<T> *handle : m_workers) {
        # 获取当前线程的工作对象
        IWorker *worker = handle->worker();
        # 如果工作对象存在
        if (worker) {
            # 获取工作对象的哈希率数据
            worker->hashrateData(hashCount, ts, rawHashes);
            # 将哈希率数据添加到哈希率统计中
            d_ptr->hashrate->add(handle->id(), hashCount, ts);

            # 如果原始哈希数为0
            if (rawHashes == 0) {
                # 将总体可用性标记为false
                totalAvailable = false;
            }

            # 累加原始哈希数到总哈希数
            totalHashCount += rawHashes;
        }
    }

    # 如果总体可用性为true
    if (totalAvailable) {
        # 将总哈希数添加到哈希率统计中
        d_ptr->hashrate->add(totalHashCount, Chrono::steadyMSecs());
    }
// 如果定义了 XMRIG_FEATURE_BENCHMARK，则返回 !d_ptr->benchmark || !d_ptr->benchmark->finish(totalHashCount)，否则返回 true
return !d_ptr->benchmark || !d_ptr->benchmark->finish(totalHashCount);

// 返回当前哈希速率
template<class T>
const xmrig::Hashrate *xmrig::Workers<T>::hashrate() const
{
    return d_ptr->hashrate.get();
}

// 设置后端
template<class T>
void xmrig::Workers<T>::setBackend(IBackend *backend)
{
    d_ptr->backend = backend;
}

// 停止工作
template<class T>
void xmrig::Workers<T>::stop()
{
    // 如果定义了 XMRIG_MINER_PROJECT，则停止 Nonce::stop(T::backend())
    #ifdef XMRIG_MINER_PROJECT
    Nonce::stop(T::backend());
    #endif

    // 删除所有工作线程
    for (Thread<T> *worker : m_workers) {
        delete worker;
    }

    // 清空工作线程列表
    m_workers.clear();

    // 如果定义了 XMRIG_MINER_PROJECT，则触发 Nonce::touch(T::backend())
    #ifdef XMRIG_MINER_PROJECT
    Nonce::touch(T::backend());
    #endif

    // 重置哈希速率
    d_ptr->hashrate.reset();
}

// 如果定义了 XMRIG_FEATURE_BENCHMARK，则启动基准测试
template<class T>
void xmrig::Workers<T>::start(const std::vector<T> &data, const std::shared_ptr<Benchmark> &benchmark)
{
    if (!benchmark) {
        return start(data, true);
    }

    start(data, false);

    d_ptr->benchmark = benchmark;
    d_ptr->benchmark->start();
}

// 创建工作线程
template<class T>
xmrig::IWorker *xmrig::Workers<T>::create(Thread<T> *)
{
    return nullptr;
}

// 当准备就绪时的回调函数
template<class T>
void *xmrig::Workers<T>::onReady(void *arg)
{
    auto handle = static_cast<Thread<T>* >(arg);

    // 创建工作线程
    IWorker *worker = create(handle);
    assert(worker != nullptr);

    if (!worker || !worker->selfTest()) {
        // 如果工作线程创建失败或自检失败，则记录错误日志并启动后端
        LOG_ERR("%s " RED("thread ") RED_BOLD("#%zu") RED(" self-test failed"), T::tag(), worker ? worker->id() : 0);
        handle->backend()->start(worker, false);
        delete worker;
        return nullptr;
    }

    assert(handle->backend() != nullptr);

    // 设置工作线程并启动后端
    handle->setWorker(worker);
    handle->backend()->start(worker, true);

    return nullptr;
}

// 启动工作线程
template<class T>
void xmrig::Workers<T>::start(const std::vector<T> &data, bool /*sleep*/)
{
    // 根据数据创建工作线程并添加到工作线程列表
    for (const auto &item : data) {
        m_workers.push_back(new Thread<T>(d_ptr->backend, m_workers.size(), item));
    }

    // 创建哈希速率对象
    d_ptr->hashrate = std::make_shared<Hashrate>(m_workers.size());
}
// 如果定义了 XMRIG_MINER_PROJECT，则调用 Nonce::touch() 方法，传入 T::backend() 作为参数
#ifdef XMRIG_MINER_PROJECT
    Nonce::touch(T::backend());
#endif

// 遍历 m_workers 容器中的每个元素，调用其 start() 方法，传入 Workers<T>::onReady 作为参数
for (auto worker : m_workers) {
    worker->start(Workers<T>::onReady);
}
}


// 命名空间 xmrig
namespace xmrig {


// 模板特化，创建 CpuLaunchData 类型的 Worker 对象
template<>
xmrig::IWorker *xmrig::Workers<CpuLaunchData>::create(Thread<CpuLaunchData> *handle)
{
    // 如果定义了 XMRIG_MINER_PROJECT，则根据 handle->config().intensity 的值创建不同强度的 CpuWorker 对象
    #ifdef XMRIG_MINER_PROJECT
    switch (handle->config().intensity) {
    case 1:
        return new CpuWorker<1>(handle->id(), handle->config());

    case 2:
        return new CpuWorker<2>(handle->id(), handle->config());

    case 3:
        return new CpuWorker<3>(handle->id(), handle->config());

    case 4:
        return new CpuWorker<4>(handle->id(), handle->config());

    case 5:
        return new CpuWorker<5>(handle->id(), handle->config());

    case 8:
        return new CpuWorker<8>(handle->id(), handle->config());
    }

    return nullptr;
    // 如果未定义 XMRIG_MINER_PROJECT，则断言 handle->config().intensity 的值为 1，然后创建 CpuWorker<1> 对象
    #else
    assert(handle->config().intensity == 1);

    return new CpuWorker<1>(handle->id(), handle->config());
    #endif
}


// 实例化 Workers<CpuLaunchData> 模板类
template class Workers<CpuLaunchData>;


// 如果定义了 XMRIG_FEATURE_OPENCL，则创建 OclWorker 对象
#ifdef XMRIG_FEATURE_OPENCL
template<>
xmrig::IWorker *xmrig::Workers<OclLaunchData>::create(Thread<OclLaunchData> *handle)
{
    return new OclWorker(handle->id(), handle->config());
}


// 实例化 Workers<OclLaunchData> 模板类
template class Workers<OclLaunchData>;
#endif


// 如果定义了 XMRIG_FEATURE_CUDA，则创建 CudaWorker 对象
#ifdef XMRIG_FEATURE_CUDA
template<>
xmrig::IWorker *xmrig::Workers<CudaLaunchData>::create(Thread<CudaLaunchData> *handle)
{
    return new CudaWorker(handle->id(), handle->config());
}


// 实例化 Workers<CudaLaunchData> 模板类
template class Workers<CudaLaunchData>;
#endif


} // namespace xmrig
```