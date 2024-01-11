# `xmrig\src\net\JobResults.cpp`

```
/* XMRig
 * 版权声明，列出了程序的各个版本的版权所有者和联系方式
 */


#include "net/JobResults.h"
#include "backend/common/Tags.h"
#include "base/io/Async.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IAsyncListener.h"
#include "base/tools/Object.h"
#include "net/interfaces/IJobResultListener.h"
#include "net/JobResult.h"
// 引入所需的头文件


#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/randomx/randomx.h"
#   include "crypto/rx/Rx.h"
#   include "crypto/rx/RxVm.h"
#endif
// 如果定义了 XMRIG_ALGO_RANDOMX，则引入相关的头文件


#ifdef XMRIG_ALGO_KAWPOW
#   include "crypto/kawpow/KPCache.h"
#   include "crypto/kawpow/KPHash.h"
#endif
// 如果定义了 XMRIG_ALGO_KAWPOW，则引入相关的头文件


#if defined(XMRIG_FEATURE_OPENCL) || defined(XMRIG_FEATURE_CUDA)
#   include "base/tools/Baton.h"
#   include "crypto/cn/CnCtx.h"
#   include "crypto/cn/CnHash.h"
#   include "crypto/cn/CryptoNight.h"
#endif
// 如果定义了 XMRIG_FEATURE_OPENCL 或 XMRIG_FEATURE_CUDA，则引入相关的头文件
// 包含头文件 "crypto/common/VirtualMemory.h"
#include "crypto/common/VirtualMemory.h"
#endif

#include <cassert>
#include <list>
#include <memory>
#include <mutex>
#include <uv.h>

// 命名空间 xmrig
namespace xmrig {

// 如果定义了 XMRIG_FEATURE_OPENCL 或 XMRIG_FEATURE_CUDA，则定义 JobBundle 类
#if defined(XMRIG_FEATURE_OPENCL) || defined(XMRIG_FEATURE_CUDA)
class JobBundle
{
public:
    // 构造函数，初始化 JobBundle 对象
    inline JobBundle(const Job &job, uint32_t *results, size_t count, uint32_t device_index) :
        job(job),
        nonces(count),
        device_index(device_index)
    {
        // 将结果复制到 nonces 中
        memcpy(nonces.data(), results, sizeof(uint32_t) * count);
    }

    Job job; // Job 对象
    std::vector<uint32_t> nonces; // 随机数向量
    uint32_t device_index; // 设备索引
};

// JobBaton 类，继承自 Baton<uv_work_t>
class JobBaton : public Baton<uv_work_t>
{
public:
    // 构造函数，初始化 JobBaton 对象
    inline JobBaton(std::list<JobBundle> &&bundles, IJobResultListener *listener, bool hwAES) :
        hwAES(hwAES),
        listener(listener),
        bundles(std::move(bundles))
    {}

    const bool hwAES; // 是否使用硬件加密
    IJobResultListener *listener; // Job 结果监听器
    std::list<JobBundle> bundles; // JobBundle 列表
    std::vector<JobResult> results; // Job 结果向量
    uint32_t errors = 0; // 错误计数
};

// 检查哈希值是否符合条件
static inline void checkHash(const JobBundle &bundle, std::vector<JobResult> &results, uint32_t nonce, uint8_t hash[32], uint32_t &errors)
{
    if (*reinterpret_cast<uint64_t*>(hash + 24) < bundle.job.target()) {
        results.emplace_back(bundle.job, nonce, hash);
    }
    else {
        LOG_ERR("%s " RED_S "GPU #%u COMPUTE ERROR", backend_tag(bundle.job.backend()), bundle.device_index);
        errors++;
    }
}

// 获取结果
static void getResults(JobBundle &bundle, std::vector<JobResult> &results, uint32_t &errors, bool hwAES)
{
    const auto &algorithm = bundle.job.algorithm(); // 获取算法
    auto memory           = new VirtualMemory(algorithm.l3(), false, false, false); // 创建虚拟内存对象
    alignas(16) uint8_t hash[32]{ 0 }; // 哈希值数组，初始化为 0

    if (algorithm.family() == Algorithm::RANDOM_X) {
# 如果定义了XMRIG_ALGO_RANDOMX，则执行以下代码块
        # 从bundle.job中获取RxDataset对象，索引为0
        RxDataset *dataset = Rx::dataset(bundle.job, 0);
        # 如果获取的dataset为空指针，则将bundle.nonces的大小加到errors上，删除memory对象，然后返回
        if (dataset == nullptr) {
            errors += bundle.nonces.size();
            delete memory;
            return;
        }
        # 根据dataset和memory的scratchpad创建RxVm对象，根据条件判断是否使用硬件AES指令集，汇编指令为NONE，索引为0
        auto vm = RxVm::create(dataset, memory->scratchpad(), !hwAES, Assembly::NONE, 0);
        # 遍历bundle.nonces中的每个nonce
        for (uint32_t nonce : bundle.nonces) {
            # 将当前nonce赋值给bundle.job.nonce()
            *bundle.job.nonce() = nonce;
            # 使用randomx_calculate_hash计算哈希值，结果存储在hash中
            randomx_calculate_hash(vm, bundle.job.blob(), bundle.job.size(), hash);
            # 检查哈希值，将结果存储在results中，同时更新errors
            checkHash(bundle, results, nonce, hash, errors);
        }
        # 销毁RxVm对象
        RxVm::destroy(vm);
#       endif
    }
    # 如果算法家族为ARGON2，则将bundle.nonces的大小加到errors上，同时添加TODO注释
    else if (algorithm.family() == Algorithm::ARGON2) {
        errors += bundle.nonces.size(); // TODO ARGON2
    }
    # 如果算法家族为KAWPOW，则执行以下代码块
    else if (algorithm.family() == Algorithm::KAWPOW) {
#ifdef XMRIG_ALGO_KAWPOW
    // 如果定义了 XMRIG_ALGO_KAWPOW，则执行以下代码块
    for (uint32_t nonce : bundle.nonces) {
        // 遍历 bundle.nonces 列表，将每个 nonce 赋值给 bundle.job.nonce()
        *bundle.job.nonce() = nonce;

        // 定义一个长度为 32 的 uint8_t 数组 header_hash，并将 bundle.job.blob() 的前 32 个字节复制到 header_hash 中
        uint8_t header_hash[32];
        memcpy(header_hash, bundle.job.blob(), sizeof(header_hash));

        // 定义一个 uint64_t 类型的变量 full_nonce，并将 bundle.job.blob() 的第 32 个字节开始的 8 个字节复制到 full_nonce 中
        uint64_t full_nonce;
        memcpy(&full_nonce, bundle.job.blob() + sizeof(header_hash), sizeof(full_nonce));

        // 定义两个长度为 8 的 uint32_t 数组 output 和 mix_hash
        uint32_t output[8];
        uint32_t mix_hash[8];
        {
            // 创建一个互斥锁对象，用于保护 KPCache::s_cache
            std::lock_guard<std::mutex> lock(KPCache::s_cacheMutex);

            // 根据 bundle.job.height() 和其他参数计算 output 和 mix_hash
            KPCache::s_cache.init(bundle.job.height() / KPHash::EPOCH_LENGTH);
            KPHash::calculate(KPCache::s_cache, bundle.job.height(), header_hash, full_nonce, output, mix_hash);
        }

        // 将 output 数组的内容按逆序复制到 hash 数组中
        for (size_t i = 0; i < sizeof(hash); ++i) {
            hash[i] = ((uint8_t*)output)[sizeof(hash) - 1 - i];
        }

        // 如果 hash 数组的最后 8 个字节转换为 uint64_t 类型的值小于 bundle.job.target()，则将结果添加到 results 列表中
        if (*reinterpret_cast<uint64_t*>(hash + 24) < bundle.job.target()) {
            results.emplace_back(bundle.job, full_nonce, (uint8_t*)output, bundle.job.blob(), (uint8_t*)mix_hash);
        }
        else {
            // 否则，输出错误信息
            LOG_ERR("%s " RED_S "GPU #%u COMPUTE ERROR", backend_tag(bundle.job.backend()), bundle.device_index);
            // 错误计数加一
            ++errors;
        }
    }
#endif
}
else {
    // 如果没有定义 XMRIG_ALGO_KAWPOW，则执行以下代码块
    // 创建一个 cryptonight_ctx 对象数组 ctx，数组大小为 1
    cryptonight_ctx *ctx[1];
    // 使用 memory->scratchpad()、memory->size() 和 1 创建 CnCtx 对象，并将其赋值给 ctx
    CnCtx::create(ctx, memory->scratchpad(), memory->size(), 1);

    // 遍历 bundle.nonces 列表
    for (uint32_t nonce : bundle.nonces) {
        // 将每个 nonce 赋值给 bundle.job.nonce()
        *bundle.job.nonce() = nonce;

        // 根据 algorithm、hwAES、Assembly::NONE 和其他参数计算 hash
        CnHash::fn(algorithm, hwAES ? CnHash::AV_SINGLE : CnHash::AV_SINGLE_SOFT, Assembly::NONE)(bundle.job.blob(), bundle.job.size(), hash, ctx, bundle.job.height());

        // 检查 hash 是否满足条件，并将结果添加到 results 列表中
        checkHash(bundle, results, nonce, hash, errors);
    }
}

// 释放 memory 对象的内存
delete memory;
#endif


// 定义一个名为 JobResultsPrivate 的类，继承自 IAsyncListener
class JobResultsPrivate : public IAsyncListener
{
public:
    // 禁用默认的拷贝构造函数和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(JobResultsPrivate)
    # 定义一个内联类 JobResultsPrivate，继承自 IJobResultListener 接口
    inline JobResultsPrivate(IJobResultListener *listener, bool hwAES) :
        # 初始化成员变量 m_hwAES，用于指示是否使用硬件 AES
        m_hwAES(hwAES),
        # 初始化成员变量 m_listener，用于保存结果监听器
        m_listener(listener)
    {
        # 使用 std::make_shared 创建一个共享指针，指向 Async 类的实例，并将当前对象作为参数传入
        m_async = std::make_shared<Async>(this);
    }
    
    
    # 定义析构函数，使用默认实现
    ~JobResultsPrivate() override = default;
    
    
    # 定义一个内联函数 submit，用于提交任务结果
    inline void submit(const JobResult &result)
    {
        # 使用互斥锁保护共享资源
        std::lock_guard<std::mutex> lock(m_mutex);
        # 将任务结果添加到结果列表中
        m_results.push_back(result);
    
        # 调用异步对象的 send 方法，通知结果监听器有新的结果可用
        m_async->send();
    }
# 如果定义了XMRIG_FEATURE_OPENCL或XMRIG_FEATURE_CUDA，则执行以下代码块
inline void submit(const Job &job, uint32_t *results, size_t count, uint32_t device_index)
{
    # 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(m_mutex);
    # 将任务、结果、计数和设备索引封装成JobBundle对象，并添加到m_bundles列表中
    m_bundles.emplace_back(job, results, count, device_index);

    # 发送异步信号
    m_async->send();
}
# 结束if语句


protected:
# 重写onAsync()方法
inline void onAsync() override  { submit(); }


private:
# 如果定义了XMRIG_FEATURE_OPENCL或XMRIG_FEATURE_CUDA，则执行以下代码块
inline void submit()
{
    # 创建JobBundle列表和JobResult列表
    std::list<JobBundle> bundles;
    std::list<JobResult> results;

    # 获取互斥锁
    m_mutex.lock();
    # 交换m_bundles和bundles列表
    m_bundles.swap(bundles);
    # 交换m_results和results列表
    m_results.swap(results);
    # 释放互斥锁
    m_mutex.unlock();

    # 遍历results列表，将结果传递给监听器
    for (const auto &result : results) {
        m_listener->onJobResult(result);
    }

    # 如果bundles列表为空，则返回
    if (bundles.empty()) {
        return;
    }

    # 创建JobBaton对象，将bundles列表、监听器和m_hwAES传递给它
    auto baton = new JobBaton(std::move(bundles), m_listener, m_hwAES);

    # 将任务提交给libuv的工作队列
    uv_queue_work(uv_default_loop(), &baton->req,
        [](uv_work_t *req) {
            auto baton = static_cast<JobBaton*>(req->data);

            # 遍历bundles列表，获取结果并存储在baton的results和errors列表中
            for (JobBundle &bundle : baton->bundles) {
                getResults(bundle, baton->results, baton->errors, baton->hwAES);
            }
        },
        [](uv_work_t *req, int) {
            auto baton = static_cast<JobBaton*>(req->data);

            # 遍历baton的results列表，将结果传递给监听器
            for (const auto &result : baton->results) {
                baton->listener->onJobResult(result);
            }

            # 删除baton对象
            delete baton;
        }
    );
}
# 结束if语句

# 如果未定义XMRIG_FEATURE_OPENCL和XMRIG_FEATURE_CUDA，则执行以下代码块
inline void submit()
{
    # 创建JobResult列表
    std::list<JobResult> results;

    # 获取互斥锁
    m_mutex.lock();
    # 交换m_results和results列表
    m_results.swap(results);
    # 释放互斥锁
    m_mutex.unlock();

    # 遍历results列表，将结果传递给监听器
    for (const auto &result : results) {
        m_listener->onJobResult(result);
    }
}
# 结束if语句

# 声明私有成员变量
const bool m_hwAES;
IJobResultListener *m_listener;
std::list<JobResult> m_results;
std::mutex m_mutex;
    # 创建一个指向Async对象的shared_ptr，用于异步操作的管理
    std::shared_ptr<Async> m_async;
# 定义了一个名为JobBundle的结构体，用于存储作业的相关信息
struct JobBundle {
    Job job;  // 作业对象
    std::vector<uint32_t> results;  // 存储作业结果的向量
    uint32_t deviceIndex;  // 设备索引
};


namespace xmrig {
    // 定义了一个名为JobResultsPrivate的类，用于处理作业结果
    class JobResultsPrivate {
    public:
        // 构造函数，接受一个IJobResultListener对象和一个布尔值作为参数
        JobResultsPrivate(IJobResultListener *listener, bool hwAES);
        // 析构函数
        ~JobResultsPrivate();
        // 提交作业结果的函数，接受一个JobResult对象作为参数
        void submit(const JobResult &result);
        // 提交作业结果的函数，接受一个Job对象、一个无符号32位整数和一个指向无符号8位整数的指针作为参数
        void submit(const Job &job, uint32_t nonce, const uint8_t *result);
        // 提交作业结果的函数，接受一个Job对象、一个无符号32位整数、一个指向无符号8位整数的指针和一个指向无符号8位整数的指针作为参数
        void submit(const Job& job, uint32_t nonce, const uint8_t* result, const uint8_t* miner_signature);
        // 提交作业结果的函数，接受一个Job对象、一个指向无符号32位整数的指针、一个size_t类型的值和一个无符号32位整数作为参数
        void submit(const Job &job, uint32_t *results, size_t count, uint32_t device_index);
        // 设置作业结果监听器的函数，接受一个IJobResultListener对象和一个布尔值作为参数
        void setListener(IJobResultListener *listener, bool hwAES);
        // 停止作业结果处理的函数
        void stop();
    private:
        std::list<JobBundle> m_bundles;  // 存储作业信息的链表
    };


    static JobResultsPrivate *handler = nullptr;  // 作业结果处理对象的指针


} // namespace xmrig


// 当作业完成时调用的函数，接受一个Job对象作为参数
void xmrig::JobResults::done(const Job &job)
{
    submit(JobResult(job));  // 提交作业结果
}


// 设置作业结果监听器的函数，接受一个IJobResultListener对象和一个布尔值作为参数
void xmrig::JobResults::setListener(IJobResultListener *listener, bool hwAES)
{
    assert(handler == nullptr);  // 断言handler为空

    handler = new JobResultsPrivate(listener, hwAES);  // 创建JobResultsPrivate对象并赋值给handler
}


// 停止作业结果处理的函数
void xmrig::JobResults::stop()
{
    assert(handler != nullptr);  // 断言handler不为空

    delete handler;  // 删除handler指向的对象
    handler = nullptr;  // 将handler置为空指针
}


// 提交作业结果的函数，接受一个Job对象、一个无符号32位整数和一个指向无符号8位整数的指针作为参数
void xmrig::JobResults::submit(const Job &job, uint32_t nonce, const uint8_t *result)
{
    submit(JobResult(job, nonce, result));  // 提交作业结果
}


// 提交作业结果的函数，接受一个Job对象、一个无符号32位整数、一个指向无符号8位整数的指针和一个指向无符号8位整数的指针作为参数
void xmrig::JobResults::submit(const Job& job, uint32_t nonce, const uint8_t* result, const uint8_t* miner_signature)
{
    submit(JobResult(job, nonce, result, nullptr, nullptr, miner_signature));  // 提交作业结果
}


// 提交作业结果的函数，接受一个JobResult对象作为参数
void xmrig::JobResults::submit(const JobResult &result)
{
    assert(handler != nullptr);  // 断言handler不为空

    if (handler) {
        handler->submit(result);  // 提交作业结果
    }
}


#if defined(XMRIG_FEATURE_OPENCL) || defined(XMRIG_FEATURE_CUDA)
// 提交作业结果的函数，接受一个Job对象、一个指向无符号32位整数的指针、一个size_t类型的值和一个无符号32位整数作为参数
void xmrig::JobResults::submit(const Job &job, uint32_t *results, size_t count, uint32_t device_index)
{
    if (handler) {
        handler->submit(job, results, count, device_index);  // 提交作业结果
    }
}
#endif
```