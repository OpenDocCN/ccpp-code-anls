# `xmrig\src\net\strategies\DonateStrategy.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布本程序的副本
 *   或者（根据您的选择）任何更高版本。
 *
 *   本程序发布的目的是希望它对您有用，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#include <algorithm>
#include <cassert>
#include <iterator>


#include "net/strategies/DonateStrategy.h"
#include "3rdparty/rapidjson/document.h"
#include "base/crypto/keccak.h"
#include "base/kernel/Platform.h"
#include "base/net/stratum/Client.h"
#include "base/net/stratum/Job.h"
#include "base/net/stratum/strategies/FailoverStrategy.h"
#include "base/net/stratum/strategies/SinglePoolStrategy.h"
#include "base/tools/Buffer.h"
#include "base/tools/Cvt.h"
#include "base/tools/Timer.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "core/Miner.h"
#include "net/Network.h"


namespace xmrig {

// 定义一个内联函数，用于生成指定范围内的随机浮点数
static inline double randomf(double min, double max)                 { return (max - min) * (((static_cast<double>(rand())) / static_cast<double>(RAND_MAX))) + min; }
// 定义一个内联函数，用于生成指定范围内的随机整数
static inline uint64_t random(uint64_t base, double min, double max) { return static_cast<uint64_t>(base * randomf(min, max)); }

// 定义一个常量字符串，表示捐赠主机的地址
static const char *kDonateHost = "donate.v2.xmrig.com";
#ifdef XMRIG_FEATURE_TLS
// 定义一个常量字符串，表示使用 TLS 的捐赠主机的地址
static const char *kDonateHostTls = "donate.ssl.xmrig.com";
#endif

} // namespace xmrig
// DonateStrategy 类的构造函数，接受一个 Controller 指针和一个 IStrategyListener 指针作为参数
xmrig::DonateStrategy::DonateStrategy(Controller *controller, IStrategyListener *listener) :
    // 根据配置中的 donateLevel 计算出 donateTime，单位为毫秒
    m_donateTime(static_cast<uint64_t>(controller->config()->pools().donateLevel()) * 60 * 1000),
    // 根据配置中的 donateLevel 计算出 idleTime，单位为毫秒
    m_idleTime((100 - static_cast<uint64_t>(controller->config()->pools().donateLevel())) * 60 * 1000),
    // 保存传入的 Controller 指针
    m_controller(controller),
    // 保存传入的 IStrategyListener 指针
    m_listener(listener)
{
    // 创建一个长度为 200 的 uint8_t 数组
    uint8_t hash[200];

    // 获取配置中第一个矿池的用户信息
    const auto &user = controller->config()->pools().data().front().user();
    // 对用户信息进行 keccak 哈希计算，结果保存在 hash 数组中
    keccak(reinterpret_cast<const uint8_t *>(user.data()), user.size(), hash);
    // 将 hash 数组中的内容转换为十六进制字符串，保存在 m_userId 中
    Cvt::toHex(m_userId, sizeof(m_userId), hash, 32);

    // 根据条件编译选择不同的 Pool 模式
#   ifdef XMRIG_ALGO_KAWPOW
    constexpr Pool::Mode mode = Pool::MODE_AUTO_ETH;
#   else
    constexpr Pool::Mode mode = Pool::MODE_POOL;
#   endif

    // 根据条件编译选择是否启用 TLS，创建对应的矿池对象并添加到 m_pools 中
#   ifdef XMRIG_FEATURE_TLS
    m_pools.emplace_back(kDonateHostTls, 443, m_userId, nullptr, nullptr, 0, true, true, mode);
#   endif
    // 创建矿池对象并添加到 m_pools 中
    m_pools.emplace_back(kDonateHost, 3333, m_userId, nullptr, nullptr, 0, true, false, mode);

    // 根据 m_pools 的大小选择使用 FailoverStrategy 或 SinglePoolStrategy，并保存在 m_strategy 中
    if (m_pools.size() > 1) {
        m_strategy = new FailoverStrategy(m_pools, 10, 2, this, true);
    }
    else {
        m_strategy = new SinglePoolStrategy(m_pools.front(), 10, 2, this, true);
    }

    // 创建一个 Timer 对象，保存在 m_timer 中
    m_timer = new Timer(this);

    // 设置状态为 STATE_IDLE
    setState(STATE_IDLE);
}

// DonateStrategy 类的析构函数
xmrig::DonateStrategy::~DonateStrategy()
{
    // 释放 m_timer 指针指向的内存
    delete m_timer;
    // 释放 m_strategy 指针指向的内存
    delete m_strategy;

    // 如果 m_proxy 不为空，则延迟释放 m_proxy
    if (m_proxy) {
        m_proxy->deleteLater();
    }
}

// 更新策略，接受一个 IClient 指针和一个 Job 对象作为参数
void xmrig::DonateStrategy::update(IClient *client, const Job &job)
{
    // 设置当前算法为 job 中的算法
    setAlgo(job.algorithm());
    // 设置代理为 client 指向的矿池的代理信息
    setProxy(client->pool().proxy());

    // 更新难度、高度和种子信息
    m_diff   = job.diff();
    m_height = job.height();
    m_seed   = job.seed();
}

// 提交任务结果，接受一个 JobResult 对象作为参数
int64_t xmrig::DonateStrategy::submit(const JobResult &result)
{
    // 如果 m_proxy 不为空，则调用 m_proxy 的 submit 方法，否则调用 m_strategy 的 submit 方法
    return m_proxy ? m_proxy->submit(result) : m_strategy->submit(result);
}

// 连接矿池
void xmrig::DonateStrategy::connect()
{
    // 创建代理对象并连接
    m_proxy = createProxy();
    if (m_proxy) {
        m_proxy->connect();
    }
    // 如果创建代理失败，则连接策略对象
    else {
        m_strategy->connect();
    }
}

// 设置算法，接受一个 Algorithm 对象作为参数
void xmrig::DonateStrategy::setAlgo(const xmrig::Algorithm &algo)
{
    # 将传入的算法赋值给成员变量 m_algorithm
    m_algorithm = algo;
    
    # 调用 m_strategy 对象的 setAlgo 方法，将算法作为参数传入
    m_strategy->setAlgo(algo);
# 设置代理
void xmrig::DonateStrategy::setProxy(const ProxyUrl &proxy)
{
    # 调用策略对象的setProxy方法，设置代理
    m_strategy->setProxy(proxy);
}


# 停止策略
void xmrig::DonateStrategy::stop()
{
    # 停止定时器
    m_timer->stop();
    # 停止策略对象
    m_strategy->stop();
}


# 策略的tick方法
void xmrig::DonateStrategy::tick(uint64_t now)
{
    # 更新当前时间
    m_now = now;

    # 调用策略对象的tick方法
    m_strategy->tick(now);

    # 如果代理存在，则调用代理的tick方法
    if (m_proxy) {
        m_proxy->tick(now);
    }

    # 如果状态为等待状态，并且当前时间大于时间戳，则将状态设置为空闲状态
    if (state() == STATE_WAIT && now > m_timestamp) {
        setState(STATE_IDLE);
    }
}


# 当策略被激活时调用的方法
void xmrig::DonateStrategy::onActive(IStrategy *, IClient *client)
{
    # 如果策略已经激活，则直接返回
    if (isActive()) {
        return;
    }

    # 设置状态为激活状态
    setState(STATE_ACTIVE);
    # 调用监听器的onActive方法
    m_listener->onActive(this, client);
}


# 当策略被暂停时调用的方法
void xmrig::DonateStrategy::onPause(IStrategy *)
{
}


# 当连接关闭时调用的方法
void xmrig::DonateStrategy::onClose(IClient *, int failures)
{
    # 如果失败次数为2，并且配置中的代理捐赠设置为自动，则删除代理对象，并将其设置为nullptr
    if (failures == 2 && m_controller->config()->pools().proxyDonate() == Pools::PROXY_DONATE_AUTO) {
        m_proxy->deleteLater();
        m_proxy = nullptr;

        # 连接策略对象
        m_strategy->connect();
    }
}


# 当登录成功时调用的方法
void xmrig::DonateStrategy::onLoginSuccess(IClient *client)
{
    # 如果策略已经激活，则直接返回
    if (isActive()) {
        return;
    }

    # 设置状态为激活状态
    setState(STATE_ACTIVE);
    # 调用监听器的onActive方法
    m_listener->onActive(this, client);
}


# 当验证算法时调用的方法
void xmrig::DonateStrategy::onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok)
{
    # 调用 m_listener 对象的 onVerifyAlgorithm 方法，传入当前对象的指针 this，client 对象，algorithm 对象和 ok 布尔值作为参数
    m_listener->onVerifyAlgorithm(this, client, algorithm, ok);
}

// 当验证算法时调用，通知监听器进行验证
void xmrig::DonateStrategy::onVerifyAlgorithm(IStrategy *, const  IClient *client, const Algorithm &algorithm, bool *ok)
{
    m_listener->onVerifyAlgorithm(this, client, algorithm, ok);
}

// 定时器触发时调用，设置状态为等待或连接
void xmrig::DonateStrategy::onTimer(const Timer *)
{
    setState(isActive() ? STATE_WAIT : STATE_CONNECT);
}

// 创建代理客户端
xmrig::IClient *xmrig::DonateStrategy::createProxy()
{
    // 如果不需要代理捐赠，则返回空指针
    if (m_controller->config()->pools().proxyDonate() == Pools::PROXY_DONATE_NONE) {
        return nullptr;
    }

    // 获取当前策略和客户端
    IStrategy *strategy = m_controller->network()->strategy();
    if (!strategy->isActive() || !strategy->client()->hasExtension(IClient::EXT_CONNECT)) {
        return nullptr;
    }

    const IClient *client = strategy->client();
    m_tls                 = client->hasExtension(IClient::EXT_TLS);

    // 创建代理池
    Pool pool(client->pool().proxy().isValid() ? client->pool().host() : client->ip(), client->pool().port(), m_userId, client->pool().password(), client->pool().spendSecretKey(), 0, true, client->isTLS(), Pool::MODE_POOL);
    pool.setAlgo(client->pool().algorithm());
    pool.setProxy(client->pool().proxy());

    // 创建代理客户端
    IClient *proxy = new Client(-1, Platform::userAgent(), this);
    proxy->setPool(pool);
    proxy->setQuiet(true);

    return proxy;
}

// 设置空闲状态
void xmrig::DonateStrategy::idle(double min, double max)
{
    m_timer->start(random(m_idleTime, min, max), 0);
}

// 设置工作参数
void xmrig::DonateStrategy::setJob(IClient *client, const Job &job, const rapidjson::Value &params)
{
    if (isActive()) {
        m_listener->onJob(this, client, job, params);
    }
}

// 设置参数
void xmrig::DonateStrategy::setParams(rapidjson::Document &doc, rapidjson::Value &params)
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();
    auto algorithms = m_controller->miner()->algorithms();

    const size_t index = static_cast<size_t>(std::distance(algorithms.begin(), std::find(algorithms.begin(), algorithms.end(), m_algorithm)));
}
    # 如果 index 大于 0 且小于 algorithms 的大小
    if (index > 0 && index < algorithms.size()) {
        # 交换 algorithms 中的第一个元素和指定索引位置的元素
        std::swap(algorithms[0], algorithms[index]);
    }

    # 创建一个 JSON 数组对象 algo
    Value algo(kArrayType);

    # 遍历 algorithms 中的每个元素，将其名称添加到 algo 数组中
    for (const auto &a : algorithms) {
        algo.PushBack(StringRef(a.name()), allocator);
    }

    # 将 algo 数组作为成员 "algo" 添加到 params 对象中
    params.AddMember("algo",    algo, allocator);
    # 将 m_diff 添加为成员 "diff" 到 params 对象中
    params.AddMember("diff",    m_diff, allocator);
    # 将 m_height 添加为成员 "height" 到 params 对象中
    params.AddMember("height",  m_height, allocator);

    # 如果 m_seed 不为空
    if (!m_seed.empty()) {
       # 将 m_seed 转换为十六进制字符串，并将其添加为成员 "seed_hash" 到 params 对象中
       params.AddMember("seed_hash", Cvt::toHex(m_seed, doc), allocator);
    }
}

// 设置捐赠策略的结果
void xmrig::DonateStrategy::setResult(IClient *client, const SubmitResult &result, const char *error)
{
    // 调用监听器的 onResultAccepted 方法，传入捐赠策略、客户端、结果和错误信息
    m_listener->onResultAccepted(this, client, result, error);
}

// 设置捐赠策略的状态
void xmrig::DonateStrategy::setState(State state)
{
    // 定义等待时间为 3000 毫秒
    constexpr const uint64_t waitTime = 3000;

    // 断言当前状态不等于新状态，并且新状态不等于 STATE_NEW
    assert(m_state != state && state != STATE_NEW);
    // 如果当前状态等于新状态，则直接返回
    if (m_state == state) {
        return;
    }

    // 保存当前状态，设置新状态
    const State prev = m_state;
    m_state = state;

    // 根据新状态进行不同的操作
    switch (state) {
    case STATE_NEW:
        break;

    case STATE_IDLE:
        // 如果前一个状态是 STATE_NEW，则调用 idle 方法，传入参数 0.5 和 1.5
        if (prev == STATE_NEW) {
            idle(0.5, 1.5);
        }
        // 如果前一个状态是 STATE_CONNECT，则启动定时器，设置定时器间隔为 20000 毫秒
        else if (prev == STATE_CONNECT) {
            m_timer->start(20000, 0);
        }
        // 其他情况下，停止策略，删除代理对象，调用 idle 方法，传入参数 0.8 和 1.2
        else {
            m_strategy->stop();
            if (m_proxy) {
                m_proxy->deleteLater();
                m_proxy = nullptr;
            }
            idle(0.8, 1.2);
        }
        break;

    case STATE_CONNECT:
        // 调用 connect 方法
        connect();
        break;

    case STATE_ACTIVE:
        // 启动定时器，设置定时器间隔为捐赠时间，0 表示不重复
        m_timer->start(m_donateTime, 0);
        break;

    case STATE_WAIT:
        // 设置时间戳为当前时间加上等待时间，调用监听器的 onPause 方法
        m_timestamp = m_now + waitTime;
        m_listener->onPause(this);
        break;
    }
}
```