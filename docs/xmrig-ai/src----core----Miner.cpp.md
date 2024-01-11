# `xmrig\src\core\Miner.cpp`

```
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

#include <algorithm>
#include <mutex>
#include <thread>


#include "core/Miner.h"
#include "core/Taskbar.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Hashrate.h"
#include "backend/cpu/Cpu.h"
#include "backend/cpu/CpuBackend.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/kernel/Platform.h"
#include "base/net/stratum/Job.h"
#include "base/tools/Object.h"
#include "base/tools/Timer.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "crypto/common/Nonce.h"
#include "version.h"


#ifdef XMRIG_FEATURE_API
#   include "base/api/Api.h"
#   include "base/api/interfaces/IApiRequest.h"
#endif


#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/OclBackend.h"
#endif


#ifdef XMRIG_FEATURE_CUDA
#   include "backend/cuda/CudaBackend.h"
#endif


#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/rx/Profiler.h"
#   include "crypto/rx/Rx.h"
#   include "crypto/rx/RxConfig.h"
#endif


#ifdef XMRIG_ALGO_GHOSTRIDER
#   include "crypto/ghostrider/ghostrider.h"
#endif


namespace xmrig {


static std::mutex mutex;


class MinerPrivate
{
public:
    # 禁用默认的拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(MinerPrivate)

    # MinerPrivate 类的构造函数，接受一个 Controller 对象指针作为参数
    inline explicit MinerPrivate(Controller *controller) : controller(controller) {}

    # MinerPrivate 类的析构函数
    inline ~MinerPrivate()
    {
        # 释放定时器对象的内存
        delete timer;

        # 遍历 backends 列表，释放每个 IBackend 对象的内存
        for (IBackend *backend : backends) {
            delete backend;
        }
# 如果定义了 XMRIG_ALGO_RANDOMX，则销毁 Rx 对象
#ifdef XMRIG_ALGO_RANDOMX
    Rx::destroy();
#endif
}


# 检查指定算法是否启用
bool isEnabled(const Algorithm &algorithm) const
{
    for (IBackend *backend : backends) {
        if (backend->isEnabled() && backend->isEnabled(algorithm)) {
            return true;
        }
    }

    return false;
}


# 重新构建算法列表
inline void rebuild()
{
    algorithms = Algorithm::all([this](const Algorithm &algo) { return isEnabled(algo); });
}


# 处理作业变化
inline void handleJobChange()
{
    # 如果未启用，则暂停 Nonce
    if (!enabled) {
        Nonce::pause(true);
    }

    # 如果需要重置，则重置 Nonce
    if (reset) {
        Nonce::reset(job.index());
    }

    # 设置后端作业
    for (IBackend *backend : backends) {
        backend->setJob(job);
    }

    # 更新 Nonce
    Nonce::touch();

    # 如果活跃且启用，则恢复 Nonce
    if (active && enabled) {
        Nonce::pause(false);
    }

    # 如果 ticks 为 0，则增加 ticks，并启动定时器
    if (ticks == 0) {
        ticks++;
        timer->start(500, 500);
    }
}


# 如果定义了 XMRIG_FEATURE_API，则获取矿工信息
#ifdef XMRIG_FEATURE_API
void getMiner(rapidjson::Value &reply, rapidjson::Document &doc, int) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    # 添加矿工信息到 reply
    reply.AddMember("version",      APP_VERSION, allocator);
    reply.AddMember("kind",         APP_KIND, allocator);
    reply.AddMember("ua",           Platform::userAgent().toJSON(), allocator);
    reply.AddMember("cpu",          Cpu::toJSON(doc), allocator);
    reply.AddMember("donate_level", controller->config()->pools().donateLevel(), allocator);
    reply.AddMember("paused",       !enabled, allocator);

    Value algo(kArrayType);

    # 遍历算法列表，添加到 reply
    for (const Algorithm &a : algorithms) {
        algo.PushBack(StringRef(a.name()), allocator);
    }

    reply.AddMember("algorithms", algo, allocator);
}


# 获取哈希率信息
void getHashrate(rapidjson::Value &reply, rapidjson::Document &doc, int version) const
    {
        // 使用 rapidjson 命名空间
        using namespace rapidjson;
        // 获取文档的分配器
        auto &allocator = doc.GetAllocator();
    
        // 创建名为 hashrate 的对象，并指定类型为对象
        Value hashrate(kObjectType);
        // 创建名为 total 的对象，并指定类型为数组
        Value total(kArrayType);
        // 创建名为 threads 的对象，并指定类型为数组
        Value threads(kArrayType);
    
        // 创建长度为 3 的 double 类型数组 t，并初始化为 0.0
        double t[3] = { 0.0 };
    
        // 遍历 backends 中的 IBackend 对象
        for (IBackend *backend : backends) {
            // 获取当前 backend 的 hashrate
            const Hashrate *hr = backend->hashrate();
            // 如果没有 hashrate，则跳过当前循环
            if (!hr) {
                continue;
            }
    
            // 分别累加不同时间间隔的 hashrate 到 t 数组中
            t[0] += hr->calc(Hashrate::ShortInterval);
            t[1] += hr->calc(Hashrate::MediumInterval);
            t[2] += hr->calc(Hashrate::LargeInterval);
    
            // 如果 version 大于 1，则跳过当前循环
            if (version > 1) {
                continue;
            }
    
            // 遍历当前 hashrate 对象的线程数
            for (size_t i = 0; i < hr->threads(); i++) {
                // 创建名为 thread 的对象，并指定类型为数组
                Value thread(kArrayType);
                // 将不同时间间隔的线程 hashrate 标准化后加入 thread 数组中
                thread.PushBack(Hashrate::normalize(hr->calc(i, Hashrate::ShortInterval)),  allocator);
                thread.PushBack(Hashrate::normalize(hr->calc(i, Hashrate::MediumInterval)), allocator);
                thread.PushBack(Hashrate::normalize(hr->calc(i, Hashrate::LargeInterval)),  allocator);
    
                // 将 thread 对象加入 threads 数组中
                threads.PushBack(thread, allocator);
            }
        }
    
        // 将 t 数组中的累加结果标准化后加入 total 数组中
        total.PushBack(Hashrate::normalize(t[0]),  allocator);
        total.PushBack(Hashrate::normalize(t[1]), allocator);
        total.PushBack(Hashrate::normalize(t[2]),  allocator);
    
        // 将 total 和最大 hashrate 加入 hashrate 对象中
        hashrate.AddMember("total",   total, allocator);
        hashrate.AddMember("highest", Hashrate::normalize(maxHashrate[algorithm]), allocator);
    
        // 如果 version 等于 1，则将 threads 加入 hashrate 对象中
        if (version == 1) {
            hashrate.AddMember("threads", threads, allocator);
        }
    
        // 将 hashrate 对象加入 reply 对象中
        reply.AddMember("hashrate", hashrate, allocator);
    }
    
    
    void getBackends(rapidjson::Value &reply, rapidjson::Document &doc) const
    {
        // 使用 rapidjson 命名空间
        using namespace rapidjson;
        // 获取文档的分配器
        auto &allocator = doc.GetAllocator();
    
        // 设置 reply 对象为数组
        reply.SetArray();
    
        // 遍历 backends 中的 IBackend 对象
        for (IBackend *backend : backends) {
            // 将 backend 对象转换为 JSON 格式，并加入 reply 数组中
            reply.PushBack(backend->toJSON(doc), allocator);
        }
    }
// 结束条件判断
    static inline void printProfile()
    {
// 如果定义了 XMRIG_FEATURE_PROFILING 宏
        ProfileScopeData* data[ProfileScopeData::MAX_DATA_COUNT];

        const uint32_t n = std::min<uint32_t>(ProfileScopeData::s_dataCount, ProfileScopeData::MAX_DATA_COUNT);
        // 复制 ProfileScopeData::s_data 数组的前 n 个元素到 data 数组
        memcpy(data, ProfileScopeData::s_data, n * sizeof(ProfileScopeData*));

        // 对 data 数组进行排序，按照线程 ID 的字典序
        std::sort(data, data + n, [](ProfileScopeData* a, ProfileScopeData* b) {
            return strcmp(a->m_threadId, b->m_threadId) < 0;
        });

        // 创建一个映射，用于存储每个函数的平均执行时间和调用次数
        std::map<std::string, std::pair<uint32_t, double>> averageTime;

        // 遍历 data 数组
        for (uint32_t i = 0; i < n;)
        {
            uint32_t n1 = i;
            // 找到相同线程 ID 的范围
            while ((n1 < n) && (strcmp(data[i]->m_threadId, data[n1]->m_threadId) == 0)) {
                ++n1;
            }

            // 对相同线程 ID 的范围内的数据进行排序，按照总周期数降序
            std::sort(data + i, data + n1, [](ProfileScopeData* a, ProfileScopeData* b) {
                return a->m_totalCycles > b->m_totalCycles;
            });

            // 遍历相同线程 ID 的范围内的数据
            for (uint32_t j = i; j < n1; ++j) {
                ProfileScopeData* p = data[j];
                // 计算平均执行时间和调用次数
                const double t = p->m_totalCycles / p->m_totalSamples * 1e9 / ProfileScopeData::s_tscSpeed;
                // 输出日志信息
                LOG_INFO("%s Thread %6s | %-30s | %7.3f%% | %9.0f ns",
                    Tags::profiler(),
                    p->m_threadId,
                    p->m_name,
                    p->m_totalCycles * 100.0 / data[i]->m_totalCycles,
                    t
                );
                auto& value = averageTime[p->m_name];
                ++value.first;
                value.second += t;
            }

            // 输出分隔线
            LOG_INFO("%s --------------|--------------------------------|----------|-------------", Tags::profiler());

            i = n1;
        }

        // 输出每个函数的平均执行时间
        for (auto& data : averageTime) {
            LOG_INFO("%s %-30s %9.1f ns", Tags::profiler(), data.first.c_str(), data.second.second / data.second.first);
        }
    }
        // 定义一个长度为 16*5 的字符数组，初始化为 0
        char num[16 * 5] = { 0 };
        // 定义一个长度为 3 的双精度浮点数数组，初始化为 0.0
        double speed[3]  = { 0.0 };
        // 定义一个无符号 32 位整数变量，初始化为 0
        uint32_t count   = 0;

        // 定义一个双精度浮点数变量，初始化为 0.0，用于存储平均哈希率
        double avg_hashrate = 0.0;

        // 遍历 backends 容器中的每个元素
        for (auto backend : backends) {
            // 获取当前 backend 的哈希率
            const auto hashrate = backend->hashrate();
            // 如果哈希率存在
            if (hashrate) {
                // 增加计数器
                ++count;

                // 分别累加不同时间间隔的哈希率到 speed 数组中
                speed[0] += hashrate->calc(Hashrate::ShortInterval);
                speed[1] += hashrate->calc(Hashrate::MediumInterval);
                speed[2] += hashrate->calc(Hashrate::LargeInterval);

                // 累加平均哈希率
                avg_hashrate += hashrate->average();
            }

            // 打印当前 backend 的哈希率详情
            backend->printHashrate(details);
        }

        // 如果没有有效的哈希率，则直接返回
        if (!count) {
            return;
        }

        // 打印性能概要
        printProfile();

        // 初始化比例为 1.0，单位为 H/s
        double scale  = 1.0;
        const char* h = "H/s";

        // 如果任意速度或最大哈希率超过 1e6，则将比例设置为 1e-6，单位设置为 MH/s
        if ((speed[0] >= 1e6) || (speed[1] >= 1e6) || (speed[2] >= 1e6) || (maxHashrate[algorithm] >= 1e6)) {
            scale = 1e-6;
            h = "MH/s";
        }

        // 初始化存储平均哈希率的字符数组
        char avg_hashrate_buf[64];
        avg_hashrate_buf[0] = '\0';
# ifdef XMRIG_ALGO_GHOSTRIDER
        # 如果定义了XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
        if (algorithm.family() == Algorithm::GHOSTRIDER) {
            # 如果算法族为GHOSTRIDER，则格式化平均哈希率的字符串并存储到avg_hashrate_buf中
            snprintf(avg_hashrate_buf, sizeof(avg_hashrate_buf), " avg " CYAN_BOLD("%s %s"), Hashrate::format(avg_hashrate * scale, num + 16 * 4, 16), h);
        }
# endif

        # 记录矿工的速度和哈希率信息
        LOG_INFO("%s " WHITE_BOLD("speed") " 10s/60s/15m " CYAN_BOLD("%s") CYAN(" %s %s ") CYAN_BOLD("%s") " max " CYAN_BOLD("%s %s") "%s",
                 Tags::miner(),
                 Hashrate::format(speed[0] * scale,                 num,          16),
                 Hashrate::format(speed[1] * scale,                 num + 16,     16),
                 Hashrate::format(speed[2] * scale,                 num + 16 * 2, 16), h,
                 Hashrate::format(maxHashrate[algorithm] * scale,   num + 16 * 3, 16), h,
                 avg_hashrate_buf
                 );

# ifdef XMRIG_FEATURE_BENCHMARK
        # 如果定义了XMRIG_FEATURE_BENCHMARK，则执行以下代码块
        for (auto backend : backends) {
            # 遍历后端列表，打印基准测试进度
            backend->printBenchProgress();
        }
# endif
    }

# ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了XMRIG_ALGO_RANDOMX，则执行以下代码块
    inline bool initRX() const { return Rx::init(job, controller->config()->rx(), controller->config()->cpu()); }
# endif

# ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果定义了XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
    inline void initGhostRider() const { ghostrider::benchmark(); }
# endif

    # 定义矿工类的成员变量
    Algorithm algorithm;
    Algorithms algorithms;
    bool active         = false;
    bool battery_power  = false;
    bool user_active    = false;
    bool enabled        = true;
    int32_t auto_pause = 0;
    bool reset          = true;
    Controller *controller;
    Job job;
    mutable std::map<Algorithm::Id, double> maxHashrate;
    std::vector<IBackend *> backends;
    String userJobId;
    Timer *timer        = nullptr;
    uint64_t ticks      = 0;

    Taskbar m_taskbar;
};

} // namespace xmrig

# 创建矿工类的构造函数，初始化私有成员变量
xmrig::Miner::Miner(Controller *controller)
    : d_ptr(new MinerPrivate(controller))
{
    # 获取CPU优先级
    const int priority = controller->config()->cpu().priority();
    # 如果优先级大于等于0
    if (priority >= 0) {
        # 设置进程优先级
        Platform::setProcessPriority(priority);
        # 设置线程优先级为优先级加1和5中的最小值
        Platform::setThreadPriority(std::min(priority + 1, 5));
    }
#   ifdef XMRIG_FEATURE_PROFILING
    # 如果定义了 XMRIG_FEATURE_PROFILING，则初始化性能分析数据
    ProfileScopeData::Init();
#   endif

#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX，则初始化 RandomX 算法
    Rx::init(this);
#   endif

    # 将当前实例添加为控制器的监听器
    controller->addListener(this);

#   ifdef XMRIG_FEATURE_API
    # 如果定义了 XMRIG_FEATURE_API，则将当前实例添加为控制器 API 的监听器
    controller->api()->addListener(this);
#   endif

    # 创建一个定时器，并将其指针赋给当前实例的 timer 属性
    d_ptr->timer = new Timer(this);

    # 预留 3 个空间给后端，并将 CPU 后端添加到后端列表中
    d_ptr->backends.reserve(3);
    d_ptr->backends.push_back(new CpuBackend(controller));

#   ifdef XMRIG_FEATURE_OPENCL
    # 如果定义了 XMRIG_FEATURE_OPENCL，则将 OpenCL 后端添加到后端列表中
    d_ptr->backends.push_back(new OclBackend(controller));
#   endif

#   ifdef XMRIG_FEATURE_CUDA
    # 如果定义了 XMRIG_FEATURE_CUDA，则将 CUDA 后端添加到后端列表中
    d_ptr->backends.push_back(new CudaBackend(controller));
#   endif

    # 重新构建当前实例
    d_ptr->rebuild();
}


# 析构函数，释放 d_ptr 指向的内存
xmrig::Miner::~Miner()
{
    delete d_ptr;
}


# 返回当前实例的 enabled 属性
bool xmrig::Miner::isEnabled() const
{
    return d_ptr->enabled;
}


# 返回指定算法是否在当前实例的 algorithms 列表中
bool xmrig::Miner::isEnabled(const Algorithm &algorithm) const
{
    return std::find(d_ptr->algorithms.begin(), d_ptr->algorithms.end(), algorithm) != d_ptr->algorithms.end();
}


# 返回当前实例的 algorithms 列表
const xmrig::Algorithms &xmrig::Miner::algorithms() const
{
    return d_ptr->algorithms;
}


# 返回当前实例的 backends 列表
const std::vector<xmrig::IBackend *> &xmrig::Miner::backends() const
{
    return d_ptr->backends;
}


# 返回当前实例的 job 属性
xmrig::Job xmrig::Miner::job() const
{
    # 使用互斥锁保护，返回当前实例的 job 属性
    std::lock_guard<std::mutex> lock(mutex);

    return d_ptr->job;
}


# 执行指定命令
void xmrig::Miner::execCommand(char command)
{
    switch (command) {
    case 'h':
    case 'H':
        # 如果命令是 'h' 或 'H'，则打印哈希率
        d_ptr->printHashrate(true);
        break;

    case 'p':
    case 'P':
        # 如果命令是 'p' 或 'P'，则禁用当前实例
        setEnabled(false);
        break;

    case 'r':
    case 'R':
        # 如果命令是 'r' 或 'R'，则启用当前实例
        setEnabled(true);
        break;

    case 'e':
    case 'E':
        # 如果命令是 'e' 或 'E'，则遍历后端列表，打印健康状态
        for (auto backend : d_ptr->backends) {
            backend->printHealth();
        }
        break;

    default:
        break;
    }

    # 遍历后端列表，执行指定命令
    for (auto backend : d_ptr->backends) {
        backend->execCommand(command);
    }
}


# 暂停当前实例
void xmrig::Miner::pause()
{
    d_ptr->active = false;
    d_ptr->m_taskbar.setActive(false);

    Nonce::pause(true);
    Nonce::touch();
}


# 设置当前实例的 enabled 属性
void xmrig::Miner::setEnabled(bool enabled)
{
    # 如果当前状态与要设置的状态相同，则直接返回，不做任何操作
    if (d_ptr->enabled == enabled) {
        return;
    }

    # 如果配置为在电池模式下暂停，并且当前为电池供电，并且要设置的状态为启用，则记录日志并返回
    if (d_ptr->controller->config()->isPauseOnBattery() && d_ptr->battery_power && enabled) {
        LOG_INFO("%s " YELLOW_BOLD("can't resume while on battery power"), Tags::miner());
        return;
    }

    # 更新状态为要设置的状态
    d_ptr->enabled = enabled;
    # 更新任务栏的状态
    d_ptr->m_taskbar.setEnabled(enabled);

    # 如果状态为启用，则记录日志为已恢复
    if (enabled) {
        LOG_INFO("%s " GREEN_BOLD("resumed"), Tags::miner());
    }
    # 如果状态为禁用，则根据是否为电池供电记录不同的日志
    else {
        if (d_ptr->battery_power) {
            LOG_INFO("%s " YELLOW_BOLD("paused"), Tags::miner());
        }
        else {
            LOG_INFO("%s " YELLOW_BOLD("paused") ", press " MAGENTA_BG_BOLD(" r ") " to resume", Tags::miner());
        }
    }

    # 如果当前状态为非活动状态，则直接返回，不做任何操作
    if (!d_ptr->active) {
        return;
    }

    # 根据要设置的状态暂停或恢复 Nonce
    Nonce::pause(!enabled);
    # 更新 Nonce 的时间戳
    Nonce::touch();
# 设置矿工的工作任务和是否捐赠
void xmrig::Miner::setJob(const Job &job, bool donate)
{
    # 遍历后端，为每个后端准备工作任务
    for (IBackend *backend : d_ptr->backends) {
        backend->prepare(job);
    }

#   ifdef XMRIG_ALGO_RANDOMX
    # 如果工作任务的算法是 RANDOM_X 并且不是准备好的状态
    if (job.algorithm().family() == Algorithm::RANDOM_X && !Rx::isReady(job)) {
        # 如果当前算法不等于工作任务的算法，则停止挖矿
        if (d_ptr->algorithm != job.algorithm()) {
            stop();
        }
        else {
            # 暂停 Nonce，然后重新触发
            Nonce::pause(true);
            Nonce::touch();
        }
    }
#   endif

    # 设置当前算法为工作任务的算法
    d_ptr->algorithm = job.algorithm();

    # 加锁
    mutex.lock();

    # 根据是否捐赠设置索引
    const uint8_t index = donate ? 1 : 0;

    # 重置标志位，如果当前工作任务的索引不等于1且索引不等于0且用户工作任务ID不等于工作任务ID
    d_ptr->reset = !(d_ptr->job.index() == 1 && index == 0 && d_ptr->userJobId == job.id());

    # 如果当前工作任务的哈希值相等，则不重置标志位
    if (d_ptr->job.isEqualBlob(job)) {
        d_ptr->reset = false;
    }

    # 设置当前工作任务为给定的工作任务
    d_ptr->job   = job;
    d_ptr->job.setIndex(index);

    # 如果索引为0，则设置用户工作任务ID为工作任务ID
    if (index == 0) {
        d_ptr->userJobId = job.id();
    }

#   ifdef XMRIG_ALGO_RANDOMX
    # 初始化 RandomX 算法
    const bool ready = d_ptr->initRX();
#   else
    # 默认设置 ready 为 true
    constexpr const bool ready = true;
#   endif

#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果工作任务的算法是 GHOSTRIDER，则初始化 GhostRider 算法
    if (job.algorithm().family() == Algorithm::GHOSTRIDER) {
        d_ptr->initGhostRider();
    }
#   endif

    # 解锁
    mutex.unlock();

    # 设置活跃状态为 true
    d_ptr->active = true;
    d_ptr->m_taskbar.setActive(true);

    # 如果 ready 为 true，则处理工作任务变化
    if (ready) {
        d_ptr->handleJobChange();
    }
}

# 停止挖矿
void xmrig::Miner::stop()
{
    # 停止 Nonce
    Nonce::stop();

    # 遍历后端，停止每个后端的工作
    for (IBackend *backend : d_ptr->backends) {
        backend->stop();
    }
}

# 当配置发生变化时触发
void xmrig::Miner::onConfigChanged(Config *config, Config *previousConfig)
{
    # 重建
    d_ptr->rebuild();

    # 如果新配置的矿池不等于旧配置的矿池且新配置的矿池数量大于0，则返回
    if (config->pools() != previousConfig->pools() && config->pools().active() > 0) {
        return;
    }

    # 获取当前工作任务
    const Job job = this->job();

    # 为每个后端设置工作任务
    for (IBackend *backend : d_ptr->backends) {
        backend->setJob(job);
    }
}

# 定时器触发时执行
void xmrig::Miner::onTimer(const Timer *)
{
    # 最大哈希率设置为0
    double maxHashrate          = 0.0;
    # 获取控制器的配置
    const auto config           = d_ptr->controller->config();
}
    // 从配置中获取健康打印时间
    const auto healthPrintTime  = config->healthPrintTime();

    // 初始化停止矿工标志
    bool stopMiner = false;

    // 遍历后端列表
    for (IBackend *backend : d_ptr->backends) {
        // 如果后端的tick函数返回false，则设置停止矿工标志为true
        if (!backend->tick(d_ptr->ticks)) {
            stopMiner = true;
        }

        // 如果健康打印时间存在且ticks不为0且满足条件，则打印健康信息
        if (healthPrintTime && d_ptr->ticks && (d_ptr->ticks % (healthPrintTime * 2)) == 0 && backend->isEnabled()) {
            backend->printHealth();
        }

        // 如果后端的hashrate存在，则计算最大hashrate
        if (backend->hashrate()) {
            maxHashrate += backend->hashrate()->calc(Hashrate::ShortInterval);
        }
    }

    // 更新最大hashrate
    d_ptr->maxHashrate[d_ptr->algorithm] = std::max(d_ptr->maxHashrate[d_ptr->algorithm], maxHashrate);

    // 从配置中获取打印时间
    const auto printTime = config->printTime();
    // 如果打印时间存在且ticks不为0且满足条件，则打印hashrate
    if (printTime && d_ptr->ticks && (d_ptr->ticks % (printTime * 2)) == 0) {
        d_ptr->printHashrate(false);
    }

    // ticks自增
    d_ptr->ticks++;

    // 定义自动暂停函数
    auto autoPause = [this](bool &state, bool pause, const char *pauseMessage, const char *activeMessage)
    {
        // 如果需要暂停且状态为非暂停，或者不需要暂停且状态为暂停，则更新状态并记录自动暂停次数
        if ((pause && !state) || (!pause && state)) {
            LOG_INFO("%s %s", Tags::miner(), pause ? pauseMessage : activeMessage);

            state = pause;
            d_ptr->auto_pause += pause ? 1 : -1;
            setEnabled(d_ptr->auto_pause == 0);
        }
    };

    // 如果配置中设置了在电池模式下暂停，则调用自动暂停函数
    if (config->isPauseOnBattery()) {
        autoPause(d_ptr->battery_power, Platform::isOnBatteryPower(), YELLOW_BOLD("on battery power"), GREEN_BOLD("on AC power"));
    }

    // 如果配置中设置了在用户活动时暂停，则调用自动暂停函数
    if (config->isPauseOnActive()) {
        autoPause(d_ptr->user_active, Platform::isUserActive(config->idleTime()), YELLOW_BOLD("user active"), GREEN_BOLD("user inactive"));
    }

    // 如果需要停止矿工，则调用停止函数
    if (stopMiner) {
        stop();
    }
#ifdef XMRIG_FEATURE_API
void xmrig::Miner::onRequest(IApiRequest &request)
{
    // 检查请求方法是否为 GET
    if (request.method() == IApiRequest::METHOD_GET) {
        // 如果请求类型为 SUMMARY
        if (request.type() == IApiRequest::REQ_SUMMARY) {
            // 接受请求
            request.accept();

            // 获取矿工信息并添加到回复中
            d_ptr->getMiner(request.reply(), request.doc(), request.version());
            // 获取哈希率并添加到回复中
            d_ptr->getHashrate(request.reply(), request.doc(), request.version());
        }
        // 如果请求的 URL 为 /2/backends
        else if (request.url() == "/2/backends") {
            // 接受请求
            request.accept();

            // 获取后端信息并添加到回复中
            d_ptr->getBackends(request.reply(), request.doc());
        }
    }
    // 如果请求类型为 JSON_RPC
    else if (request.type() == IApiRequest::REQ_JSON_RPC) {
        // 如果 RPC 方法为 pause
        if (request.rpcMethod() == "pause") {
            // 接受请求
            request.accept();

            // 设置矿工为不可用状态
            setEnabled(false);
        }
        // 如果 RPC 方法为 resume
        else if (request.rpcMethod() == "resume") {
            // 接受请求
            request.accept();

            // 设置矿工为可用状态
            setEnabled(true);
        }
        // 如果 RPC 方法为 stop
        else if (request.rpcMethod() == "stop") {
            // 接受请求
            request.accept();

            // 停止矿工
            stop();
        }
    }

    // 遍历后端列表，处理请求
    for (IBackend *backend : d_ptr->backends) {
        backend->handleRequest(request);
    }
}
#endif


#ifdef XMRIG_ALGO_RANDOMX
void xmrig::Miner::onDatasetReady()
{
    // 如果随机 X 数据集未准备好，则返回
    if (!Rx::isReady(job())) {
        return;
    }

    // 处理作业变更
    d_ptr->handleJobChange();
}
#endif
```