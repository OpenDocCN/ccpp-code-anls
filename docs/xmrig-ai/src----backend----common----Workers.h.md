# `xmrig\src\backend\common\Workers.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_WORKERS_H
#define XMRIG_WORKERS_H


#include <memory>


#include "backend/common/Thread.h"
#include "backend/cpu/CpuLaunchData.h"


#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/OclLaunchData.h"
#endif


#ifdef XMRIG_FEATURE_CUDA
#   include "backend/cuda/CudaLaunchData.h"
#endif


namespace xmrig {


class Benchmark;
class Hashrate;
class WorkersPrivate;


template<class T>
class Workers
{
public:
    XMRIG_DISABLE_COPY_MOVE(Workers)

    // 构造函数
    Workers();
    // 析构函数
    ~Workers();

    // 启动工作线程
    inline void start(const std::vector<T> &data)   { start(data, true); }

    // 执行工作线程的工作
    bool tick(uint64_t ticks);
    // 获取哈希率
    const Hashrate *hashrate() const;
    // 通知工作线程新的工作
    void jobEarlyNotification(const Job &job);
    // 设置后端
    void setBackend(IBackend *backend);
    // 停止工作线程
    void stop();

#   ifdef XMRIG_FEATURE_BENCHMARK
    // 启动工作线程，并进行基准测试
    void start(const std::vector<T> &data, const std::shared_ptr<Benchmark> &benchmark);
#   endif

private:
    // 创建工作线程
    static IWorker *create(Thread<T> *handle);
    // 工作线程准备就绪时的回调函数
    static void *onReady(void *arg);

    // 启动工作线程
    void start(const std::vector<T> &data, bool sleep);

    // 工作线程列表
    std::vector<Thread<T> *> m_workers;
    WorkersPrivate *d_ptr;
};


template<class T>
// 定义 Workers 类模板的 jobEarlyNotification 方法，接受一个 Job 对象作为参数
void xmrig::Workers<T>::jobEarlyNotification(const Job &job)
{
    // 遍历 m_workers 中的每个线程指针
    for (Thread<T>* t : m_workers) {
        // 如果线程指针指向的对象存在
        if (t->worker()) {
            // 调用该对象的 jobEarlyNotification 方法，传入参数 job
            t->worker()->jobEarlyNotification(job);
        }
    }
}

// 实例化 Workers 类模板的 create 方法，传入 CpuLaunchData 类型的模板参数
template<>
IWorker *Workers<CpuLaunchData>::create(Thread<CpuLaunchData> *handle);

// 声明 Workers 类模板的实例化，传入 CpuLaunchData 类型的模板参数
extern template class Workers<CpuLaunchData>;

#ifdef XMRIG_FEATURE_OPENCL
// 实例化 Workers 类模板的 create 方法，传入 OclLaunchData 类型的模板参数
template<>
IWorker *Workers<OclLaunchData>::create(Thread<OclLaunchData> *handle);
// 声明 Workers 类模板的实例化，传入 OclLaunchData 类型的模板参数
extern template class Workers<OclLaunchData>;
#endif

#ifdef XMRIG_FEATURE_CUDA
// 实例化 Workers 类模板的 create 方法，传入 CudaLaunchData 类型的模板参数
template<>
IWorker *Workers<CudaLaunchData>::create(Thread<CudaLaunchData> *handle);
// 声明 Workers 类模板的实例化，传入 CudaLaunchData 类型的模板参数
extern template class Workers<CudaLaunchData>;
#endif

// 结束 xmrig 命名空间
} // namespace xmrig

// 结束头文件的条件编译
#endif /* XMRIG_WORKERS_H */
```