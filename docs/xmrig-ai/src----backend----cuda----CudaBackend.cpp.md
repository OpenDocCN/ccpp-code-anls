# `xmrig\src\backend\cuda\CudaBackend.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <mutex>
#include <string>


#include "backend/cuda/CudaBackend.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Hashrate.h"
#include "backend/common/interfaces/IWorker.h"
#include "backend/common/Tags.h"
#include "backend/common/Workers.h"
#include "backend/cuda/CudaConfig.h"
#include "backend/cuda/CudaThreads.h"
#include "backend/cuda/CudaWorker.h"
#include "backend/cuda/wrappers/CudaDevice.h"
#include "backend/cuda/wrappers/CudaLib.h"
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


#ifdef XMRIG_FEATURE_NVML
#include "backend/cuda/wrappers/NvmlLib.h"

namespace xmrig { static const char *kNvmlLabel = "NVML"; }
#endif


namespace xmrig {


extern template class Threads<CudaThreads>;


constexpr const size_t oneMiB   = 1024U * 1024U;
# 定义常量字符串指针 kLabel，用于标识 CUDA
static const char *kLabel       = "CUDA";
# 定义常量字符串 kType，表示 CUDA 类型
static const String kType       = "cuda";
# 定义互斥锁 mutex，用于多线程同步

# 定义函数 printDisabled，用于打印禁用信息
static void printDisabled(const char *label, const char *reason)
{
    # 打印禁用信息，包括标签和原因
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") RED_BOLD("disabled") "%s", label, reason);
}

# 定义结构体 CudaLaunchStatus，用于跟踪 CUDA 启动状态
struct CudaLaunchStatus
{
public:
    # 返回线程数
    inline size_t threads() const { return m_threads; }

    # 判断是否已经启动，根据 ready 参数决定增加启动次数或错误次数
    inline bool started(bool ready)
    {
        ready ? m_started++ : m_errors++;

        return (m_started + m_errors) == m_threads;
    }

    # 初始化启动状态，设置线程数，启动时间，以及 CUDAWorker 的就绪状态
    inline void start(size_t threads)
    {
        m_started         = 0;
        m_errors          = 0;
        m_threads         = threads;
        m_ts              = Chrono::steadyMSecs();
        CudaWorker::ready = false;
    }

    # 打印启动状态信息
    inline void print() const
    {
        if (m_started == 0) {
            # 如果没有启动任何线程，打印禁用信息
            LOG_ERR("%s " RED_BOLD("disabled") YELLOW(" (failed to start threads)"), Tags::nvidia());

            return;
        }

        # 打印 CUDA 就绪状态信息，包括已启动线程数、总线程数、以及启动时间
        LOG_INFO("%s" GREEN_BOLD(" READY") " threads " "%s%zu/%zu" BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::nvidia(),
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

# 定义类 CudaBackendPrivate，用于管理 CUDA 后端私有信息
class CudaBackendPrivate
{
public:
    # 构造函数，初始化控制器和 CUDA 配置
    inline explicit CudaBackendPrivate(Controller *controller) :
        controller(controller)
    {
        init(controller->config()->cuda());
    }

    # 初始化 CUDA 配置
    void init(const CudaConfig &cuda)
    # 检查是否启用了 CUDA，如果没有则返回禁用信息
    if (!cuda.isEnabled()) {
        return printDisabled(kLabel, "");
    }

    # 初始化 CUDA 库，如果初始化失败则返回禁用信息
    if (!CudaLib::init(cuda.loader())) {
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") RED_BOLD("disabled ") RED("(%s)"), kLabel, CudaLib::lastError());
        return;
    }

    # 获取 CUDA 运行时版本和驱动版本
    runtimeVersion = CudaLib::runtimeVersion();
    driverVersion  = CudaLib::driverVersion();

    # 如果运行时版本、驱动版本或设备数量为 0，则返回禁用信息
    if (!runtimeVersion || !driverVersion || !CudaLib::deviceCount()) {
        return printDisabled(kLabel, RED_S " (no devices)");
    }

    # 如果已经获取了设备信息，则直接返回
    if (!devices.empty()) {
        return;
    }

    # 获取 CUDA 设备信息，如果获取失败则返回禁用信息
    devices = CudaLib::devices(cuda.bfactor(), cuda.bsleep(), cuda.devicesHint());
    if (devices.empty()) {
        return printDisabled(kLabel, RED_S " (no devices)");
    }

    # 打印 CUDA 相关信息
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") WHITE_BOLD("%s") "/" WHITE_BOLD("%s") BLACK_BOLD("/%s"), kLabel,
               CudaLib::version(runtimeVersion).c_str(), CudaLib::version(driverVersion).c_str(), CudaLib::pluginVersion());
// 如果定义了 XMRIG_FEATURE_NVML
#ifdef XMRIG_FEATURE_NVML
    // 如果启用了 NVML
    if (cuda.isNvmlEnabled()) {
        // 初始化 NVML 库
        if (NvmlLib::init(cuda.nvmlLoader())) {
            // 将设备信息分配给 NVML 库
            NvmlLib::assign(devices);

            // 打印 NVML 版本和驱动版本信息
            Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") WHITE_BOLD("%s") "/" GREEN_BOLD("%s") " press " MAGENTA_BG(WHITE_BOLD_S "e") " for health report",
                       kNvmlLabel,
                       NvmlLib::version(),
                       NvmlLib::driverVersion()
                       );
        }
        else {
            // 打印 NVML 加载失败信息
            printDisabled(kNvmlLabel, RED_S " (failed to load NVML)");
        }
    }
    else {
        // 打印 NVML 禁用信息
        printDisabled(kNvmlLabel, "");
    }
#endif

    // 遍历设备列表，打印 CUDA 设备信息
    for (const CudaDevice &device : devices) {
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CYAN_BOLD("#%zu") YELLOW(" %s") GREEN_BOLD(" %s ") WHITE_BOLD("%u/%u MHz") " smx:" WHITE_BOLD("%u") " arch:" WHITE_BOLD("%u%u") " mem:" CYAN("%zu/%zu") " MB",
                   "CUDA GPU",
                   device.index(),
                   device.topology().toString().data(),
                   device.name().data(),
                   device.clock(),
                   device.memoryClock(),
                   device.smx(),
                   device.computeCapability(true),
                   device.computeCapability(false),
                   device.freeMemSize() / oneMiB,
                   device.globalMemSize() / oneMiB);
    }
}

// 启动函数，接受一个 Job 对象作为参数
inline void start(const Job &job)
        # 打印信息日志，包括使用的配置文件名、线程数、算法内存占用等信息
        LOG_INFO("%s use profile " BLUE_BG(WHITE_BOLD_S " %s ") WHITE_BOLD_S " (" CYAN_BOLD("%zu") WHITE_BOLD(" thread%s)") " scratchpad " CYAN_BOLD("%zu KB"),
                 Tags::nvidia(),
                 profileName.data(),
                 threads.size(),
                 threads.size() > 1 ? "s" : "",
                 algo.l3() / 1024
                 );

        # 打印表头信息
        Log::print(WHITE_BOLD("|  # | GPU |  BUS ID | INTENSITY | THREADS | BLOCKS | BF |  BS | MEMORY | NAME"));

        # 获取算法的 L3 缓存大小
        size_t algo_l3 = algo.l3();

        # 初始化循环变量 i
        size_t i = 0;
        # 遍历线程数据
        for (const auto &data : threads) {
            # 计算内存使用量
            size_t mem_used = (data.thread.threads() * data.thread.blocks()) * algo_l3 / oneMiB;
#           ifdef XMRIG_ALGO_KAWPOW
            // 如果定义了 XMRIG_ALGO_KAWPOW
            if (algo.family() == Algorithm::KAWPOW) {
                // 如果算法族为 KAWPOW，则计算当前高度对应的 epoch
                const uint32_t epoch = job.height() / KPHash::EPOCH_LENGTH;
                // 计算内存使用量，单位为 MiB
                mem_used = (KPCache::dag_size(epoch) + oneMiB - 1) / oneMiB;
            }
#           endif

            // 打印线程、设备和内存使用情况
            Log::print("|" CYAN_BOLD("%3zu") " |" CYAN_BOLD("%4u") " |" YELLOW(" %7s") " |" CYAN_BOLD("%10d") " |" CYAN_BOLD("%8d") " |"
                       CYAN_BOLD("%7d") " |" CYAN_BOLD("%3d") " |" CYAN_BOLD("%4d") " |" CYAN("%7zu") " | " GREEN("%s"),
                       i,
                       data.thread.index(),
                       data.device.topology().toString().data(),
                       data.thread.threads() * data.thread.blocks(),
                       data.thread.threads(),
                       data.thread.blocks(),
                       data.thread.bfactor(),
                       data.thread.bsleep(),
                       mem_used,
                       data.device.name().data()
                       );

                    // 递增计数器
                    i++;
        }

        // 启动状态监控
        status.start(threads.size());
        // 启动工作线程
        workers.start(threads);
    }


#   ifdef XMRIG_FEATURE_NVML
    // 打印健康状态信息
    void printHealth()
    {
        // 遍历设备列表
        for (const auto &device : devices) {
            // 获取设备健康信息
            const auto health = NvmlLib::health(device.nvmlDevice());
    
            // 初始化字符串变量用于存储设备时钟信息
            std::string clocks;
            // 如果设备时钟和内存时钟都存在
            if (health.clock && health.memClock) {
                // 将时钟信息添加到字符串中
                clocks += " " + std::to_string(health.clock) + "/" + std::to_string(health.memClock) + " MHz";
            }
    
            // 初始化字符串变量用于存储风扇信息
            std::string fans;
            // 如果风扇速度信息不为空
            if (!health.fanSpeed.empty()) {
                // 遍历风扇速度信息
                for (size_t i = 0; i < health.fanSpeed.size(); ++i) {
                    // 将风扇速度信息添加到字符串中
                    fans += " fan" + std::to_string(i) + ":" CYAN_BOLD_S + std::to_string(health.fanSpeed[i]) + "%" CLEAR;
                }
            }
    
            // 输出设备信息
            LOG_INFO("%s" CYAN_BOLD(" #%u") YELLOW(" %s") MAGENTA_BOLD("%4uW") CSI "1;%um %2uC" CLEAR WHITE_BOLD("%s") "%s",
                     Tags::nvidia(),
                     device.index(),
                     device.topology().toString().data(),
                     health.power,
                     health.temperature < 60 ? 32 : (health.temperature > 85 ? 31 : 33),
                     health.temperature,
                     clocks.c_str(),
                     fans.c_str()
                     );
        }
    }
// 定义结束标记
#   endif

// 声明变量
    Algorithm algo;
    Controller *controller;
    CudaLaunchStatus status;
    std::vector<CudaDevice> devices;
    std::vector<CudaLaunchData> threads;
    String profileName;
    uint32_t driverVersion      = 0;
    uint32_t runtimeVersion     = 0;
    Workers<CudaLaunchData> workers;
};

// 命名空间结束
} // namespace xmrig

// 返回 NVIDIA 标签
const char *xmrig::cuda_tag()
{
    return Tags::nvidia();
}

// 构造函数，初始化私有指针
xmrig::CudaBackend::CudaBackend(Controller *controller) :
    d_ptr(new CudaBackendPrivate(controller))
{
    d_ptr->workers.setBackend(this);
}

// 析构函数，释放私有指针
xmrig::CudaBackend::~CudaBackend()
{
    delete d_ptr;

    CudaLib::close();

// 如果定义了 XMRIG_FEATURE_NVML，则关闭 NvmlLib
#   ifdef XMRIG_FEATURE_NVML
    NvmlLib::close();
#   endif
}

// 检查 CUDA 是否启用
bool xmrig::CudaBackend::isEnabled() const
{
    return d_ptr->controller->config()->cuda().isEnabled() && CudaLib::isInitialized() && !d_ptr->devices.empty();;
}

// 检查特定算法的 CUDA 是否启用
bool xmrig::CudaBackend::isEnabled(const Algorithm &algorithm) const
{
    return !d_ptr->controller->config()->cuda().threads().get(algorithm).isEmpty();
}

// 返回哈希率
const xmrig::Hashrate *xmrig::CudaBackend::hashrate() const
{
    return d_ptr->workers.hashrate();
}

// 返回配置文件名
const xmrig::String &xmrig::CudaBackend::profileName() const
{
    return d_ptr->profileName;
}

// 返回类型
const xmrig::String &xmrig::CudaBackend::type() const
{
    return kType;
}

// 执行命令
void xmrig::CudaBackend::execCommand(char)
{
}

// 准备作业
void xmrig::CudaBackend::prepare(const Job &job)
{
    if (d_ptr) {
        d_ptr->workers.jobEarlyNotification(job);
    }
}

// 打印哈希率
void xmrig::CudaBackend::printHashrate(bool details)
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
}
    # 如果任一哈希率大于等于1百万，设置缩放因子为1百万分之一，单位为MH/s
    if ((hashrate_short >= 1e6) || (hashrate_medium >= 1e6) || (hashrate_large >= 1e6)) {
        scale = 1e-6;
        h = "MH/s";
    }

    # 打印CUDA相关信息的表头，包括10秒、60秒和15分钟的哈希率
    Log::print(WHITE_BOLD_S "|   CUDA # | AFFINITY | 10s %s | 60s %s | 15m %s |", h, h, h);

    # 初始化循环计数器i为0，遍历d_ptr->threads中的数据
    size_t i = 0;
    for (const auto& data : d_ptr->threads) {
         # 打印每个线程的相关信息，包括线程编号、亲和性、不同时间间隔的哈希率、设备索引、拓扑结构和设备名称
         Log::print("| %8zu | %8" PRId64 " | %8s | %8s | %8s |" CYAN_BOLD(" #%u") YELLOW(" %s") GREEN(" %s"),
                    i,
                    data.thread.affinity(),
                    Hashrate::format(hashrate()->calc(i, Hashrate::ShortInterval)  * scale, num,          sizeof num / 3),
                    Hashrate::format(hashrate()->calc(i, Hashrate::MediumInterval) * scale, num + 16,      sizeof num / 3),
                    Hashrate::format(hashrate()->calc(i, Hashrate::LargeInterval)  * scale, num + 16 * 2, sizeof num / 3),
                    data.device.index(),
                    data.device.topology().toString().data(),
                    data.device.name().data()
                    );

         # 循环计数器i自增1
         i++;
    }

    # 打印总哈希率的表尾，包括10秒、60秒和15分钟的哈希率
    Log::print(WHITE_BOLD_S "|        - |        - | %8s | %8s | %8s |",
               Hashrate::format(hashrate_short  * scale, num,          sizeof num / 3),
               Hashrate::format(hashrate_medium * scale, num + 16,     sizeof num / 3),
               Hashrate::format(hashrate_large  * scale, num + 16 * 2, sizeof num / 3)
               );
# 如果启用了 NVML 特性
void xmrig::CudaBackend::printHealth()
{
#   ifdef XMRIG_FEATURE_NVML
    # 调用私有成员函数打印健康信息
    d_ptr->printHealth();
#   endif
}

# 设置作业
void xmrig::CudaBackend::setJob(const Job &job)
{
    # 获取 CUDA 配置
    const auto &cuda = d_ptr->controller->config()->cuda();
    # 如果 CUDA 被启用，则初始化 CUDA
    if (cuda.isEnabled()) {
        d_ptr->init(cuda);
    }

    # 如果未启用 CUDA，则停止
    if (!isEnabled()) {
        return stop();
    }

    # 获取线程数
    auto threads = cuda.get(d_ptr->controller->miner(), job.algorithm(), d_ptr->devices);
    # 如果线程数不为空且与当前线程数相等，则返回
    if (!d_ptr->threads.empty() && d_ptr->threads.size() == threads.size() && std::equal(d_ptr->threads.begin(), d_ptr->threads.end(), threads.begin())) {
        return;
    }

    # 设置算法和配置文件名
    d_ptr->algo         = job.algorithm();
    d_ptr->profileName  = cuda.threads().profileName(job.algorithm());

    # 如果配置文件名为空或线程数为空，则记录警告并停止
    if (d_ptr->profileName.isNull() || threads.empty()) {
        LOG_WARN("%s " RED_BOLD("disabled") YELLOW(" (no suitable configuration found)"), Tags::nvidia());
        return stop();
    }

    # 停止当前操作
    stop();

    # 移动线程数并开始作业
    d_ptr->threads = std::move(threads);
    d_ptr->start(job);
}

# 开始 CUDA 后端
void xmrig::CudaBackend::start(IWorker *worker, bool ready)
{
    # 加锁
    mutex.lock();

    # 如果状态已经开始，则打印状态并设置为就绪
    if (d_ptr->status.started(ready)) {
        d_ptr->status.print();
        CudaWorker::ready = true;
    }

    # 解锁
    mutex.unlock();

    # 如果就绪，则开始工作
    if (ready) {
        worker->start();
    }
}

# 停止 CUDA 后端
void xmrig::CudaBackend::stop()
{
    # 如果线程数为空，则返回
    if (d_ptr->threads.empty()) {
        return;
    }

    # 记录停止时间并停止工作和清空线程数
    const uint64_t ts = Chrono::steadyMSecs();
    d_ptr->workers.stop();
    d_ptr->threads.clear();
    LOG_INFO("%s" YELLOW(" stopped") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::nvidia(), Chrono::steadyMSecs() - ts);
}

# 执行滴答操作
bool xmrig::CudaBackend::tick(uint64_t ticks)
{
    return d_ptr->workers.tick(ticks);
}

# 如果启用了 API 特性，则转换为 JSON 格式
#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::CudaBackend::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    # 创建 JSON 对象
    Value out(kObjectType);
    out.AddMember("type",       type().toJSON(), allocator);
    out.AddMember("enabled",    isEnabled(), allocator);
    # 将算法对象转换为 JSON 格式，并添加到输出对象中
    out.AddMember("algo",       d_ptr->algo.toJSON(), allocator);
    # 将配置文件名转换为 JSON 格式，并添加到输出对象中
    out.AddMember("profile",    profileName().toJSON(), allocator);

    # 如果 CUDA 库已准备好
    if (CudaLib::isReady()) {
        # 创建存储版本信息的 JSON 对象
        Value versions(kObjectType);
        # 添加 CUDA 运行时版本信息到版本对象中
        versions.AddMember("cuda-runtime",   Value(CudaLib::version(d_ptr->runtimeVersion).c_str(), allocator), allocator);
        # 添加 CUDA 驱动版本信息到版本对象中
        versions.AddMember("cuda-driver",    Value(CudaLib::version(d_ptr->driverVersion).c_str(), allocator), allocator);
        # 添加 CUDA 插件版本信息到版本对象中
        versions.AddMember("plugin",         String(CudaLib::pluginVersion()).toJSON(doc), allocator);
// 如果定义了 XMRIG_FEATURE_NVML
#ifdef XMRIG_FEATURE_NVML
    // 如果 NvmlLib 准备就绪
    if (NvmlLib::isReady()) {
        // 添加 nvml 版本信息到 versions 对象
        versions.AddMember("nvml", StringRef(NvmlLib::version()), allocator);
        // 添加驱动版本信息到 versions 对象
        versions.AddMember("driver", StringRef(NvmlLib::driverVersion()), allocator);
    }
#endif

// 将 versions 对象添加到 out 对象
out.AddMember("versions", versions, allocator);

// 如果线程为空或者 hash 率为 0
if (d_ptr->threads.empty() || !hashrate()) {
    // 返回 out 对象
    return out;
}

// 将 hash 率添加到 out 对象
out.AddMember("hashrate", hashrate()->toJSON(doc), allocator);

// 创建一个数组对象 threads
Value threads(kArrayType);

// 初始化索引 i
size_t i = 0;
// 遍历线程数据
for (const auto &data : d_ptr->threads) {
    // 将线程数据转换为 JSON 对象并添加到 thread 对象
    Value thread = data.thread.toJSON(doc);
    // 将 hash 率添加到 thread 对象
    thread.AddMember("hashrate", hashrate()->toJSON(i, doc), allocator);
    // 将设备数据转换为 JSON 对象并添加到 thread 对象
    data.device.toJSON(thread, doc);
    // 索引递增
    i++;
    // 将 thread 对象添加到数组 threads
    threads.PushBack(thread, allocator);
}

// 将数组 threads 添加到 out 对象
out.AddMember("threads", threads, allocator);

// 返回 out 对象
return out;
}

// 处理 CUDA 后端的请求
void xmrig::CudaBackend::handleRequest(IApiRequest &) {
}
#endif
```