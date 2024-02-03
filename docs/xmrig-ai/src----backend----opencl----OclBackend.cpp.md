# `xmrig\src\backend\opencl\OclBackend.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <mutex>
#include <string>


#include "backend/opencl/OclBackend.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Hashrate.h"
#include "backend/common/interfaces/IWorker.h"
#include "backend/common/Tags.h"
#include "backend/common/Workers.h"
#include "backend/opencl/OclConfig.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/OclWorker.h"
#include "backend/opencl/runners/tools/OclSharedState.h"
#include "backend/opencl/wrappers/OclContext.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/net/stratum/Job.h"
#include "base/tools/Chrono.h"
#include "base/tools/String.h"
#include "core/config/Config.h"
#include "core/Controller.h"


#ifdef XMRIG_ALGO_KAWPOW
#   include "crypto/kawpow/KPCache.h"
#   include "crypto/kawpow/KPHash.h"
#endif


#ifdef XMRIG_FEATURE_API
#   include "base/api/interfaces/IApiRequest.h"
#endif


#ifdef XMRIG_FEATURE_ADL
#include "backend/opencl/wrappers/AdlLib.h"

namespace xmrig { static const char *kAdlLabel = "ADL"; }
#endif


namespace xmrig {


extern template class Threads<OclThreads>;
# 定义常量，表示1MB的大小
constexpr const size_t oneMiB   = 1024U * 1024U;
# 定义静态常量字符串，表示OPENCL
static const char *kLabel       = "OPENCL";
# 定义静态常量字符串，表示opencl
static const String kType       = "opencl";
# 定义互斥锁对象
static std::mutex mutex;

# 定义函数，用于打印禁用信息
static void printDisabled(const char *label, const char *reason)
{
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") RED_BOLD("disabled") "%s", label, reason);
}

# 定义结构体，表示OpenCL启动状态
struct OclLaunchStatus
{
public:
    # 返回线程数
    inline size_t threads() const { return m_threads; }

    # 判断是否启动，根据ready参数增加启动次数或错误次数，并返回是否所有线程都已启动或出错
    inline bool started(bool ready)
    {
        ready ? m_started++ : m_errors++;

        return (m_started + m_errors) == m_threads;
    }

    # 初始化启动状态，设置线程数，启动次数和错误次数为0，记录启动时间，设置OclWorker的ready状态为false
    inline void start(size_t threads)
    {
        m_started        = 0;
        m_errors         = 0;
        m_threads        = threads;
        m_ts             = Chrono::steadyMSecs();
        OclWorker::ready = false;
    }

    # 打印启动状态信息，如果启动次数为0，则打印禁用信息，否则打印就绪信息
    inline void print() const
    {
        if (m_started == 0) {
            LOG_ERR("%s " RED_BOLD("disabled") YELLOW(" (failed to start threads)"), Tags::opencl());

            return;
        }

        LOG_INFO("%s" GREEN_BOLD(" READY") " threads " "%s%zu/%zu" BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::opencl(),
                 m_errors == 0 ? CYAN_BOLD_S : YELLOW_BOLD_S,
                 m_started,
                 m_threads,
                 Chrono::steadyMSecs() - m_ts
                 );
    }

private:
    size_t m_errors     = 0;
    size_t m_started    = 0;
    size_t m_threads    = 0;
    uint64_t m_ts       = 0;
};

# 定义私有类，表示OpenCL后端
class OclBackendPrivate
{
public:
    # 构造函数，初始化控制器对象，并调用init函数进行初始化
    inline explicit OclBackendPrivate(Controller *controller) :
        controller(controller)
    {
        init(controller->config()->cl());
    }

    # 初始化函数，根据OclConfig对象进行初始化
    void init(const OclConfig &cl)
    # 如果 OpenCL 禁用，则返回禁用信息
    if (!cl.isEnabled()) {
        return printDisabled(kLabel, "");
    }

    # 初始化 OpenCL 库，如果失败则返回禁用信息
    if (!OclLib::init(cl.loader())) {
        return printDisabled(kLabel, RED_S " (failed to load OpenCL runtime)");
    }

    # 如果已经选择了平台，则直接返回
    if (platform.isValid()) {
        return;
    }

    # 获取 OpenCL 平台，如果无效则返回禁用信息
    platform = cl.platform();
    if (!platform.isValid()) {
        return printDisabled(kLabel, RED_S " (selected OpenCL platform NOT found)");
    }

    # 获取平台上的设备列表，如果为空则返回禁用信息
    devices = platform.devices();
    if (devices.empty()) {
        return printDisabled(kLabel, RED_S " (no devices)");
    }
# 如果定义了 XMRIG_FEATURE_ADL，则执行以下代码块
        if (cl.isAdlEnabled()) {
            # 如果 ADL 库初始化成功
            if (AdlLib::init()) {
                # 打印带有颜色的信息，提示用户按下 "e" 键获取健康报告
                Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") "press " MAGENTA_BG(WHITE_BOLD_S "e") " for health report",
                           kAdlLabel
                           );
            }
            # 如果 ADL 库初始化失败
            else {
                # 打印带有颜色的信息，提示用户 ADL 加载失败
                printDisabled(kAdlLabel, RED_S " (failed to load ADL)");
            }
        }
        # 如果未启用 ADL
        else {
            # 打印带有颜色的信息，提示用户 ADL 已禁用
            printDisabled(kAdlLabel, "");
        }

        # 打印带有颜色的信息，显示 OPENCL 平台的索引、名称和版本
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CYAN_BOLD("#%zu ") WHITE_BOLD("%s") "/" WHITE_BOLD("%s"), "OPENCL", platform.index(), platform.name().data(), platform.version().data());

        # 遍历设备列表，打印带有颜色的信息，显示每个 OPENCL GPU 的索引、拓扑结构、可打印名称、时钟频率、计算单元数量、可用内存和全局内存大小
        for (const OclDevice &device : devices) {
            Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CYAN_BOLD("#%zu") YELLOW(" %s") " %s " WHITE_BOLD("%u MHz") " cu:" WHITE_BOLD("%u") " mem:" CYAN("%zu/%zu") " MB",
                       "OPENCL GPU",
                       device.index(),
                       device.topology().toString().data(),
                       device.printableName().data(),
                       device.clock(),
                       device.computeUnits(),
                       device.freeMemSize() / oneMiB,
                       device.globalMemSize() / oneMiB);
        }
    }

    # 定义一个内联函数 start，参数为 Job 对象
    inline void start(const Job &job)
    {
        // 输出信息日志，包括使用的配置文件名、线程数、scratchpad大小等信息
        LOG_INFO("%s use profile " BLUE_BG(WHITE_BOLD_S " %s ") WHITE_BOLD_S " (" CYAN_BOLD("%zu") WHITE_BOLD(" thread%s)") " scratchpad " CYAN_BOLD("%zu KB"),
                 Tags::opencl(),
                 profileName.data(),
                 threads.size(),
                 threads.size() > 1 ? "s" : "",
                 algo.l3() / 1024
                 );
    
        // 打印表头
        Log::print(WHITE_BOLD("|  # | GPU |  BUS ID | INTENSITY | WSIZE | MEMORY | NAME"));
    
        // 获取算法的L3缓存大小
        size_t algo_l3 = algo.l3();
    
        // 初始化循环变量
        size_t i = 0;
        // 遍历线程数据
        for (const auto &data : threads) {
            // 计算内存使用量
            size_t mem_used = data.thread.intensity() * algo_l3 / oneMiB;
// 如果定义了 XMRIG_ALGO_KAWPOW
if (algo.family() == Algorithm::KAWPOW) {
    // 计算当前作业高度对应的纪元数
    const uint32_t epoch = job.height() / KPHash::EPOCH_LENGTH;
    // 计算内存使用量，单位为兆字节
    mem_used = (KPCache::cache_size(epoch) + KPCache::dag_size(epoch)) / oneMiB;
}

// 打印线程、设备和内存使用情况
Log::print("|" CYAN_BOLD("%3zu") " |" CYAN_BOLD("%4u") " |" YELLOW(" %7s") " |" CYAN_BOLD("%10u") " |" CYAN_BOLD("%6u") " |"
           CYAN("%7zu") " | %s",
           i,
           data.thread.index(),
           data.device.topology().toString().data(),
           data.thread.intensity(),
           data.thread.worksize(),
           mem_used,
           data.device.printableName().data()
           );

// 增加计数器
i++;
}

// 启动 OpenCL 共享状态
OclSharedState::start(threads, job);

// 启动线程状态
status.start(threads.size());
workers.start(threads);
}

// 如果定义了 XMRIG_FEATURE_ADL
void printHealth()
{
    // 如果 ADL 库未准备好，则返回
    if (!AdlLib::isReady()) {
        return;
    }

    // 打印设备健康信息
    for (const auto &device : devices) {
        const auto health = AdlLib::health(device);

        LOG_INFO("%s" CYAN_BOLD(" #%u") YELLOW(" %s") MAGENTA_BOLD("%4uW") CSI "1;%um %2uC" CYAN_BOLD(" %4u") CYAN("RPM") WHITE_BOLD(" %u/%u") "MHz",
                 Tags::opencl(),
                 device.index(),
                 device.topology().toString().data(),
                 health.power,
                 health.temperature < 60 ? 32 : (health.temperature > 85 ? 31 : 33),
                 health.temperature,
                 health.rpm,
                 health.clock,
                 health.memClock
                 );
    }
}

// 声明变量
Algorithm algo;
Controller *controller;
OclContext context;
OclLaunchStatus status;
OclPlatform platform;
std::vector<OclDevice> devices;
    # 创建一个存储OclLaunchData对象的向量
    std::vector<OclLaunchData> threads;
    # 创建一个存储字符串的变量，用于存储profile的名称
    String profileName;
    # 创建一个Workers对象，用于存储OclLaunchData对象
    Workers<OclLaunchData> workers;
}; // 结束命名空间 xmrig

} // namespace xmrig

// 返回 OpenCL 标签
const char *xmrig::ocl_tag()
{
    return Tags::opencl();
}

// 构造函数，初始化 OclBackendPrivate 对象
xmrig::OclBackend::OclBackend(Controller *controller) :
    d_ptr(new OclBackendPrivate(controller))
{
    d_ptr->workers.setBackend(this);
}

// 析构函数，释放 OclBackendPrivate 对象
xmrig::OclBackend::~OclBackend()
{
    delete d_ptr;

    OclLib::close();

#   ifdef XMRIG_FEATURE_ADL
    AdlLib::close();
#   endif
}

// 检查 OpenCL 后端是否启用
bool xmrig::OclBackend::isEnabled() const
{
    return d_ptr->controller->config()->cl().isEnabled() && OclLib::isInitialized() && d_ptr->platform.isValid() && !d_ptr->devices.empty();
}

// 检查特定算法的 OpenCL 后端是否启用
bool xmrig::OclBackend::isEnabled(const Algorithm &algorithm) const
{
    return !d_ptr->controller->config()->cl().threads().get(algorithm).isEmpty();
}

// 返回哈希率
const xmrig::Hashrate *xmrig::OclBackend::hashrate() const
{
    return d_ptr->workers.hashrate();
}

// 返回配置文件中的配置文件名
const xmrig::String &xmrig::OclBackend::profileName() const
{
    return d_ptr->profileName;
}

// 返回类型
const xmrig::String &xmrig::OclBackend::type() const
{
    return kType;
}

// 执行命令
void xmrig::OclBackend::execCommand(char)
{
}

// 准备工作
void xmrig::OclBackend::prepare(const Job &job)
{
    if (d_ptr) {
        d_ptr->workers.jobEarlyNotification(job);
    }
}

// 打印哈希率
void xmrig::OclBackend::printHashrate(bool details)
{
    if (!details || !hashrate()) {
        return;
    }

    char num[16 * 3] = { 0 };

    const double hashrate_short  = hashrate()->calc(Hashrate::ShortInterval);
    const double hashrate_medium = hashrate()->calc(Hashrate::MediumInterval);
    const double hashrate_large  = hashrate()->calc(Hashrate::LargeInterval);

    double scale = 1.0;
    const char* h = " H/s";

    if ((hashrate_short >= 1e6) || (hashrate_medium >= 1e6) || (hashrate_large >= 1e6)) {
        scale = 1e-6;
        h = "MH/s";
    }

    Log::print(WHITE_BOLD_S "| OPENCL # | AFFINITY | 10s %s | 60s %s | 15m %s |", h, h, h);

    size_t i = 0;
}
    // 遍历线程数据指针中的每个数据
    for (const auto& data : d_ptr->threads) {
         // 打印线程信息，包括线程编号、亲和性、短、中、长时间内的哈希率，设备索引，设备拓扑结构和可打印名称
         Log::print("| %8zu | %8" PRId64 " | %8s | %8s | %8s |" CYAN_BOLD(" #%u") YELLOW(" %s") " %s",
                    i,
                    data.affinity,
                    Hashrate::format(hashrate()->calc(i, Hashrate::ShortInterval)  * scale, num,          sizeof num / 3),
                    Hashrate::format(hashrate()->calc(i, Hashrate::MediumInterval) * scale, num + 16,     sizeof num / 3),
                    Hashrate::format(hashrate()->calc(i, Hashrate::LargeInterval)  * scale, num + 16 * 2, sizeof num / 3),
                    data.device.index(),
                    data.device.topology().toString().data(),
                    data.device.printableName().data()
                    );

         // 递增线程编号
         i++;
    }

    // 打印总哈希率，包括短、中、长时间内的哈希率
    Log::print(WHITE_BOLD_S "|        - |        - | %8s | %8s | %8s |",
               Hashrate::format(hashrate_short  * scale, num,          sizeof num / 3),
               Hashrate::format(hashrate_medium * scale, num + 16,     sizeof num / 3),
               Hashrate::format(hashrate_large  * scale, num + 16 * 2, sizeof num / 3)
               );
# 打印 OpenCL 后端的健康状态
void xmrig::OclBackend::printHealth()
{
#   ifdef XMRIG_FEATURE_ADL
    # 如果支持 ADL 特性，则打印健康状态
    d_ptr->printHealth();
#   endif
}

# 设置作业
void xmrig::OclBackend::setJob(const Job &job)
{
    # 获取 OpenCL 配置
    const auto &cl = d_ptr->controller->config()->cl();
    # 如果 OpenCL 已启用，则初始化
    if (cl.isEnabled()) {
        d_ptr->init(cl);
    }

    # 如果未启用 OpenCL，则停止
    if (!isEnabled()) {
        return stop();
    }

    # 获取线程数
    auto threads = cl.get(d_ptr->controller->miner(), job.algorithm(), d_ptr->platform, d_ptr->devices);
    # 如果线程数与当前线程数相同，则返回
    if (!d_ptr->threads.empty() && d_ptr->threads.size() == threads.size() && std::equal(d_ptr->threads.begin(), d_ptr->threads.end(), threads.begin())) {
        return;
    }

    # 设置算法和配置文件名
    d_ptr->algo         = job.algorithm();
    d_ptr->profileName  = cl.threads().profileName(job.algorithm());

    # 如果配置文件名为空或线程数为空，则记录警告并停止
    if (d_ptr->profileName.isNull() || threads.empty()) {
        LOG_WARN("%s " RED_BOLD("disabled") YELLOW(" (no suitable configuration found)"), Tags::opencl());
        return stop();
    }

    # 如果上下文初始化失败，则记录警告并停止
    if (!d_ptr->context.init(d_ptr->devices, threads)) {
        LOG_WARN("%s " RED_BOLD("disabled") YELLOW(" (OpenCL context unavailable)"), Tags::opencl());
        return stop();
    }

    # 停止当前任务
    stop();

    # 移动线程数并开始新的作业
    d_ptr->threads = std::move(threads);
    d_ptr->start(job);
}

# 开始任务
void xmrig::OclBackend::start(IWorker *worker, bool ready)
{
    # 加锁
    mutex.lock();

    # 如果状态已经开始，则打印状态
    if (d_ptr->status.started(ready)) {
        d_ptr->status.print();
        OclWorker::ready = true;
    }

    # 解锁
    mutex.unlock();

    # 如果准备好，则开始任务
    if (ready) {
        worker->start();
    }
}

# 停止任务
void xmrig::OclBackend::stop()
{
    # 如果线程数为空，则返回
    if (d_ptr->threads.empty()) {
        return;
    }

    # 记录停止时间
    const uint64_t ts = Chrono::steadyMSecs();

    # 停止工作线程和清空线程数
    d_ptr->workers.stop();
    d_ptr->threads.clear();

    # 释放共享状态
    OclSharedState::release();

    # 记录停止信息
    LOG_INFO("%s" YELLOW(" stopped") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::opencl(), Chrono::steadyMSecs() - ts);
}

# 执行任务
bool xmrig::OclBackend::tick(uint64_t ticks)
{
    return d_ptr->workers.tick(ticks);
}

# 转换为 JSON 格式
#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::OclBackend::toJSON(rapidjson::Document &doc) const
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取 JSON 文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个 JSON 对象
    Value out(kObjectType);
    // 添加 "type" 属性到 JSON 对象
    out.AddMember("type",       type().toJSON(), allocator);
    // 添加 "enabled" 属性到 JSON 对象
    out.AddMember("enabled",    isEnabled(), allocator);
    // 添加 "algo" 属性到 JSON 对象
    out.AddMember("algo",       d_ptr->algo.toJSON(), allocator);
    // 添加 "profile" 属性到 JSON 对象
    out.AddMember("profile",    profileName().toJSON(), allocator);
    // 添加 "platform" 属性到 JSON 对象
    out.AddMember("platform",   d_ptr->platform.toJSON(doc), allocator);

    // 如果线程为空或者 hash 率为 0，则返回 JSON 对象
    if (d_ptr->threads.empty() || !hashrate()) {
        return out;
    }

    // 添加 "hashrate" 属性到 JSON 对象
    out.AddMember("hashrate", hashrate()->toJSON(doc), allocator);

    // 创建一个 JSON 数组
    Value threads(kArrayType);

    // 初始化索引 i
    size_t i = 0;
    // 遍历线程数据
    for (const auto &data : d_ptr->threads) {
        // 创建一个线程 JSON 对象
        Value thread = data.thread.toJSON(doc);
        // 添加 "affinity" 属性到线程 JSON 对象
        thread.AddMember("affinity", data.affinity, allocator);
        // 添加 "hashrate" 属性到线程 JSON 对象
        thread.AddMember("hashrate", hashrate()->toJSON(i, doc), allocator);
        // 将设备数据添加到线程 JSON 对象
        data.device.toJSON(thread, doc);

        // 增加索引 i
        i++;
        // 将线程 JSON 对象添加到线程数组
        threads.PushBack(thread, allocator);
    }

    // 添加 "threads" 属性到 JSON 对象
    out.AddMember("threads", threads, allocator);

    // 返回 JSON 对象
    return out;
# 结束 OclBackend 类的定义
}

# 定义 OclBackend 类的 handleRequest 方法，接受一个名为 IApiRequest 的参数
void xmrig::OclBackend::handleRequest(IApiRequest &)
{
}
# 结束条件编译指令，结束对 OclBackend 类的定义
#endif
```