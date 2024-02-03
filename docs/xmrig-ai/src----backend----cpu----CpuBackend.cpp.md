# `xmrig\src\backend\cpu\CpuBackend.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <mutex>


#include "backend/cpu/CpuBackend.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Hashrate.h"
#include "backend/common/interfaces/IWorker.h"
#include "backend/common/Tags.h"
#include "backend/common/Workers.h"
#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/net/stratum/Job.h"
#include "base/tools/Chrono.h"
#include "base/tools/String.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "crypto/common/VirtualMemory.h"
#include "crypto/rx/Rx.h"
#include "crypto/rx/RxDataset.h"


#ifdef XMRIG_FEATURE_API
#   include "base/api/interfaces/IApiRequest.h"
#endif


#ifdef XMRIG_ALGO_ARGON2
#   include "crypto/argon2/Impl.h"
#endif


#ifdef XMRIG_FEATURE_BENCHMARK
#   include "backend/common/benchmark/Benchmark.h"
#   include "backend/common/benchmark/BenchState.h"
#endif


namespace xmrig {


extern template class Threads<CpuThreads>;


static const String kType   = "cpu";
static std::mutex mutex;


struct CpuLaunchStatus
{
public:
    inline const HugePagesInfo &hugePages() const   { return m_hugePages; }
*/
    // 返回内存总量
    inline size_t memory() const                    { return m_ways * m_memory; }
    // 返回线程数量
    inline size_t threads() const                   { return m_threads; }
    // 返回并行度
    inline size_t ways() const                      { return m_ways; }

    // 初始化并启动线程
    inline void start(const std::vector<CpuLaunchData> &threads, size_t memory)
    {
        // 清空工作线程内存
        m_workersMemory.clear();
        // 重置巨大页面
        m_hugePages.reset();
        // 设置内存大小
        m_memory       = memory;
        // 初始化已启动线程数量
        m_started      = 0;
        // 初始化总启动线程数量
        m_totalStarted = 0;
        // 初始化错误数量
        m_errors       = 0;
        // 设置线程数量
        m_threads      = threads.size();
        // 初始化并行度
        m_ways         = 0;
        // 记录开始时间
        m_ts           = Chrono::steadyMSecs();
    }

    // 判断线程是否已启动
    inline bool started(IWorker *worker, bool ready)
    {
        if (ready) {
            // 已启动线程数量加一
            m_started++;
            // 总启动线程数量加上当前工作线程的线程数量
            m_totalStarted += worker->threads();

            // 如果工作线程的内存尚未记录，则记录其内存，并更新巨大页面数量
            if (m_workersMemory.insert(worker->memory()).second) {
                m_hugePages += worker->memory()->hugePages();
            }
            // 更新并行度
            m_ways += worker->intensity();
        }
        else {
            // 如果线程未启动，则错误数量加一
            m_errors++;
        }

        // 返回是否所有线程都已启动或出现错误
        return (m_started + m_errors) == m_threads;
    }

    // 打印线程启动情况
    inline void print() const
    {
        if (m_started == 0) {
            // 如果没有线程启动，则打印相应信息并返回
            LOG_ERR("%s " RED_BOLD("disabled") YELLOW(" (failed to start threads)"), Tags::cpu());

            return;
        }

        // 打印线程启动情况、巨大页面情况和内存使用情况
        LOG_INFO("%s" GREEN_BOLD(" READY") " threads %s%zu/%zu (%zu)" CLEAR " huge pages %s%1.0f%% %zu/%zu" CLEAR " memory " CYAN_BOLD("%zu KB") BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::cpu(),
                 m_errors == 0 ? CYAN_BOLD_S : YELLOW_BOLD_S,
                 m_totalStarted, std::max(m_totalStarted, m_threads), m_ways,
                 (m_hugePages.isFullyAllocated() ? GREEN_BOLD_S : (m_hugePages.allocated == 0 ? RED_BOLD_S : YELLOW_BOLD_S)),
                 m_hugePages.percent(),
                 m_hugePages.allocated, m_hugePages.total,
                 memory() / 1024,
                 Chrono::steadyMSecs() - m_ts
                 );
    }
private:
    // 保存指向虚拟内存对象的指针的集合
    std::set<const VirtualMemory*> m_workersMemory;
    // 存储大页信息的对象
    HugePagesInfo m_hugePages;
    // 错误数量
    size_t m_errors       = 0;
    // 内存大小
    size_t m_memory       = 0;
    // 启动数量
    size_t m_started      = 0;
    // 总启动数量
    size_t m_totalStarted = 0;
    // 线程数量
    size_t m_threads      = 0;
    // 方式数量
    size_t m_ways         = 0;
    // 时间戳
    uint64_t m_ts         = 0;
};


class CpuBackendPrivate
{
public:
    // 构造函数，初始化控制器指针
    inline explicit CpuBackendPrivate(Controller *controller) : controller(controller)   {}


    // 启动函数
    inline void start()
    {
        // 输出日志信息，包括使用的配置文件名、线程数量和缓存大小
        LOG_INFO("%s use profile " BLUE_BG(WHITE_BOLD_S " %s ") WHITE_BOLD_S " (" CYAN_BOLD("%zu") WHITE_BOLD(" thread%s)") " scratchpad " CYAN_BOLD("%zu KB"),
                 Tags::cpu(),
                 profileName.data(),
                 threads.size(),
                 threads.size() > 1 ? "s" : "",
                 algo.l3() / 1024
                 );
        // 启动状态对象，传入线程数量和缓存大小
        status.start(threads, algo.l3());

#       ifdef XMRIG_FEATURE_BENCHMARK
        // 如果启用了基准测试，使用基准测试启动工作线程
        workers.start(threads, benchmark);
#       else
        // 否则，正常启动工作线程
        workers.start(threads);
#       endif
    }


    // 获取方式数量
    size_t ways() const
    {
        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(mutex);

        return status.ways();
    }


    // 获取大页信息
    rapidjson::Value hugePages(int version, rapidjson::Document &doc) const
    {
        HugePagesInfo pages;

    #   ifdef XMRIG_ALGO_RANDOMX
        // 如果算法是RandomX，获取RandomX的大页信息
        if (algo.family() == Algorithm::RANDOM_X) {
            pages += Rx::hugePages();
        }
    #   endif

        // 上锁
        mutex.lock();

        // 获取状态对象的大页信息
        pages += status.hugePages();

        // 解锁
        mutex.unlock();

        rapidjson::Value hugepages;

        // 根据版本号不同，构造不同的大页信息对象
        if (version > 1) {
            hugepages.SetArray();
            hugepages.PushBack(static_cast<uint64_t>(pages.allocated), doc.GetAllocator());
            hugepages.PushBack(static_cast<uint64_t>(pages.total), doc.GetAllocator());
        }
        else {
            hugepages = pages.isFullyAllocated();
        }

        return hugepages;
    }


    // 算法对象
    Algorithm algo;
    // 控制器指针
    Controller *controller;
    // CPU启动状态对象
    CpuLaunchStatus status;
    # 创建一个存储CpuLaunchData对象的向量
    std::vector<CpuLaunchData> threads;
    # 创建一个存储字符串的变量，用于存储profile的名称
    String profileName;
    # 创建一个Workers对象，用于存储CpuLaunchData对象
    Workers<CpuLaunchData> workers;
// 如果定义了 XMRIG_FEATURE_BENCHMARK，则创建一个 Benchmark 对象的共享指针
#ifdef XMRIG_FEATURE_BENCHMARK
    std::shared_ptr<Benchmark> benchmark;
#endif
};


} // namespace xmrig


// 返回与给定后端对应的标签
const char *xmrig::backend_tag(uint32_t backend)
{
#ifdef XMRIG_FEATURE_OPENCL
    // 如果后端是 OPENCL，则返回 OPENCL 的标签
    if (backend == Nonce::OPENCL) {
        return ocl_tag();
    }
#endif

#ifdef XMRIG_FEATURE_CUDA
    // 如果后端是 CUDA，则返回 CUDA 的标签
    if (backend == Nonce::CUDA) {
        return cuda_tag();
    }
#endif

    // 否则返回 CPU 的标签
    return Tags::cpu();
}


// 返回 CPU 的标签
const char *xmrig::cpu_tag()
{
    return Tags::cpu();
}


// 构造函数，初始化 CpuBackend 对象
xmrig::CpuBackend::CpuBackend(Controller *controller) :
    d_ptr(new CpuBackendPrivate(controller))
{
    // 设置 workers 的后端为当前对象
    d_ptr->workers.setBackend(this);
}


// 析构函数，释放 CpuBackend 对象的私有指针
xmrig::CpuBackend::~CpuBackend()
{
    delete d_ptr;
}


// 检查 CPU 后端是否启用
bool xmrig::CpuBackend::isEnabled() const
{
    return d_ptr->controller->config()->cpu().isEnabled();
}


// 检查特定算法的 CPU 后端是否启用
bool xmrig::CpuBackend::isEnabled(const Algorithm &algorithm) const
{
    return !d_ptr->controller->config()->cpu().threads().get(algorithm).isEmpty();
}


// 更新 workers 的 tick
bool xmrig::CpuBackend::tick(uint64_t ticks)
{
    return d_ptr->workers.tick(ticks);
}


// 返回 workers 的哈希率
const xmrig::Hashrate *xmrig::CpuBackend::hashrate() const
{
    return d_ptr->workers.hashrate();
}


// 返回 profileName
const xmrig::String &xmrig::CpuBackend::profileName() const
{
    return d_ptr->profileName;
}


// 返回类型
const xmrig::String &xmrig::CpuBackend::type() const
{
    return kType;
}


// 准备工作，根据下一个任务的算法选择是否使用 argon2 实现
void xmrig::CpuBackend::prepare(const Job &nextJob)
{
#ifdef XMRIG_ALGO_ARGON2
    const auto f = nextJob.algorithm().family();
    if ((f == Algorithm::ARGON2) || (f == Algorithm::RANDOM_X)) {
        if (argon2::Impl::select(d_ptr->controller->config()->cpu().argon2Impl())) {
            // 输出使用的 argon2 实现信息
            LOG_INFO("%s use " WHITE_BOLD("argon2") " implementation " CSI "1;%dm" "%s",
                     Tags::cpu(),
                     argon2::Impl::name() == "default" ? 33 : 32,
                     argon2::Impl::name().data()
                     );
        }
    }
#endif
}


// 打印哈希率，如果 details 为假或哈希率为空则返回
void xmrig::CpuBackend::printHashrate(bool details)
{
    if (!details || !hashrate()) {
        return;
    }
    // 创建一个包含24个元素的字符数组，每个元素都初始化为0
    char num[8 * 3] = { 0 };

    // 打印表头信息
    Log::print(WHITE_BOLD_S "|    CPU # | AFFINITY | 10s H/s | 60s H/s | 15m H/s |");

    // 初始化循环计数器
    size_t i = 0;
    // 遍历线程数据
    for (const CpuLaunchData &data : d_ptr->threads) {
         // 打印每个线程的信息
         Log::print("| %8zu | %8" PRId64 " | %7s | %7s | %7s |",
                    i,
                    data.affinity,
                    // 格式化并打印10秒内的哈希率
                    Hashrate::format(hashrate()->calc(i, Hashrate::ShortInterval),  num,         sizeof num / 3),
                    // 格式化并打印60秒内的哈希率
                    Hashrate::format(hashrate()->calc(i, Hashrate::MediumInterval), num + 8,     sizeof num / 3),
                    // 格式化并打印15分钟内的哈希率
                    Hashrate::format(hashrate()->calc(i, Hashrate::LargeInterval),  num + 8 * 2, sizeof num / 3)
                    );

         // 更新循环计数器
         i++;
    }

    // 打印总体哈希率信息
    Log::print(WHITE_BOLD_S "|        - |        - | %7s | %7s | %7s |",
               // 格式化并打印10秒内的总体哈希率
               Hashrate::format(hashrate()->calc(Hashrate::ShortInterval),  num,         sizeof num / 3),
               // 格式化并打印60秒内的总体哈希率
               Hashrate::format(hashrate()->calc(Hashrate::MediumInterval), num + 8,     sizeof num / 3),
               // 格式化并打印15分钟内的总体哈希率
               Hashrate::format(hashrate()->calc(Hashrate::LargeInterval),  num + 8 * 2, sizeof num / 3)
               );
# 打印 CPU 后端的健康状态
void xmrig::CpuBackend::printHealth()
{
}

# 设置作业，根据作业算法和 CPU 配置确定线程数，并启动相应的线程
void xmrig::CpuBackend::setJob(const Job &job)
{
    # 如果 CPU 后端未启用，则停止
    if (!isEnabled()) {
        return stop();
    }

    # 获取 CPU 配置中对应作业算法的线程数
    const auto &cpu = d_ptr->controller->config()->cpu();
    auto threads = cpu.get(d_ptr->controller->miner(), job.algorithm());
    
    # 如果当前线程不为空且与新线程数相等且内容相同，则直接返回
    if (!d_ptr->threads.empty() && d_ptr->threads.size() == threads.size() && std::equal(d_ptr->threads.begin(), d_ptr->threads.end(), threads.begin())) {
        return;
    }

    # 设置作业算法和线程配置文件名
    d_ptr->algo         = job.algorithm();
    d_ptr->profileName  = cpu.threads().profileName(job.algorithm());

    # 如果配置文件名为空或线程数为空，则记录警告信息并停止
    if (d_ptr->profileName.isNull() || threads.empty()) {
        LOG_WARN("%s " RED_BOLD("disabled") YELLOW(" (no suitable configuration found)"), Tags::cpu());
        return stop();
    }

    # 如果启用了基准测试，则创建基准测试对象
    #ifdef XMRIG_FEATURE_BENCHMARK
    if (BenchState::size()) {
        d_ptr->benchmark = std::make_shared<Benchmark>(threads.size(), this);
    }
    #endif

    # 移动线程配置到当前线程，并启动线程
    d_ptr->threads = std::move(threads);
    d_ptr->start();
}

# 启动 CPU 后端，如果 ready 为 true，则启动 worker
void xmrig::CpuBackend::start(IWorker *worker, bool ready)
{
    # 加锁
    mutex.lock();

    # 如果状态已经启动，则打印状态信息
    if (d_ptr->status.started(worker, ready)) {
        d_ptr->status.print();
    }

    # 解锁
    mutex.unlock();

    # 如果 ready 为 true，则启动 worker
    if (ready) {
        worker->start();
    }
}

# 停止 CPU 后端，停止所有线程并记录停止信息
void xmrig::CpuBackend::stop()
{
    # 如果线程为空，则直接返回
    if (d_ptr->threads.empty()) {
        return;
    }

    # 记录停止前的时间戳
    const uint64_t ts = Chrono::steadyMSecs();

    # 停止所有 worker 和清空线程
    d_ptr->workers.stop();
    d_ptr->threads.clear();

    # 记录停止信息
    LOG_INFO("%s" YELLOW(" stopped") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::cpu(), Chrono::steadyMSecs() - ts);
}

# 如果启用了 API 功能，则将 CPU 后端信息转换为 JSON 格式
#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::CpuBackend::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator         = doc.GetAllocator();
    const CpuConfig &cpu    = d_ptr->controller->config()->cpu();

    # 创建 JSON 对象
    Value out(kObjectType);
    out.AddMember("type",       type().toJSON(), allocator);
    out.AddMember("enabled",    isEnabled(), allocator);
    # 将 d_ptr 对象中的 algo 转换为 JSON 格式，并添加到 out 对象中
    out.AddMember("algo",       d_ptr->algo.toJSON(), allocator);
    # 将 profileName 转换为 JSON 格式，并添加到 out 对象中
    out.AddMember("profile",    profileName().toJSON(), allocator);
    # 检查 CPU 是否支持硬件 AES，然后添加到 out 对象中
    out.AddMember("hw-aes",     cpu.isHwAES(), allocator);
    # 获取 CPU 的优先级，并添加到 out 对象中
    out.AddMember("priority",   cpu.priority(), allocator);
    # 检查是否支持 MSR，然后添加到 out 对象中
    out.AddMember("msr",        Rx::isMSR(), allocator);
// 如果定义了 XMRIG_FEATURE_ASM，则使用 CPU 的汇编信息添加到输出中
const Assembly assembly = Cpu::assembly(cpu.assembly());
out.AddMember("asm", assembly.toJSON(), allocator);
// 如果未定义 XMRIG_FEATURE_ASM，则将 false 添加到输出中
out.AddMember("asm", false, allocator);

// 如果定义了 XMRIG_ALGO_ARGON2，则添加 argon2 实现的名称到输出中
out.AddMember("argon2-impl", argon2::Impl::name().toJSON(), allocator);

// 添加系统的巨页设置信息到输出中
out.AddMember("hugepages", d_ptr->hugePages(2, doc), allocator);
// 添加内存信息到输出中
out.AddMember("memory", static_cast<uint64_t>(d_ptr->algo.isValid() ? (d_ptr->ways() * d_ptr->algo.l3()) : 0), allocator);

// 如果线程为空或者哈希率为 0，则直接返回输出
if (d_ptr->threads.empty() || !hashrate()) {
    return out;
}

// 添加哈希率信息到输出中
out.AddMember("hashrate", hashrate()->toJSON(doc), allocator);

// 创建线程数组
Value threads(kArrayType);

// 遍历线程数据，添加到线程数组中
size_t i = 0;
for (const CpuLaunchData &data : d_ptr->threads) {
    Value thread(kObjectType);
    thread.AddMember("intensity", data.intensity, allocator);
    thread.AddMember("affinity", data.affinity, allocator);
    thread.AddMember("av", data.av(), allocator);
    thread.AddMember("hashrate", hashrate()->toJSON(i, doc), allocator);

    i++;
    threads.PushBack(thread, allocator);
}

// 将线程数组添加到输出中
out.AddMember("threads", threads, allocator);

// 返回输出
return out;
}

// 处理 API 请求
void xmrig::CpuBackend::handleRequest(IApiRequest &request)
{
    // 如果请求类型为 REQ_SUMMARY，则添加系统的巨页设置信息到回复中
    if (request.type() == IApiRequest::REQ_SUMMARY) {
        request.reply().AddMember("hugepages", d_ptr->hugePages(request.version(), request.doc()), request.doc().GetAllocator());
    }
}
#endif

#ifdef XMRIG_FEATURE_BENCHMARK
// 返回基准测试指针
xmrig::Benchmark *xmrig::CpuBackend::benchmark() const
{
    return d_ptr->benchmark.get();
}

// 打印基准测试进度
void xmrig::CpuBackend::printBenchProgress() const
{
    // 如果存在基准测试对象，则打印进度信息
    if (d_ptr->benchmark) {
        d_ptr->benchmark->printProgress();
    }
}
#endif
```