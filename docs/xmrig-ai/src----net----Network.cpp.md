# `xmrig\src\net\Network.cpp`

```cpp
/* XMRig
 * 版权所有（c）2019      Howard Chu  <https://github.com/hyc>
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifdef _MSC_VER
#pragma warning(disable:4244)
#endif

#include "net/Network.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Tags.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/net/stratum/Client.h"
#include "base/net/stratum/NetworkState.h"
#include "base/net/stratum/SubmitResult.h"
#include "base/tools/Chrono.h"
#include "base/tools/Timer.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "core/Miner.h"
#include "net/JobResult.h"
#include "net/JobResults.h"
#include "net/strategies/DonateStrategy.h"


#ifdef XMRIG_FEATURE_API
#   include "base/api/Api.h"
#   include "base/api/interfaces/IApiRequest.h"
#endif


#ifdef XMRIG_FEATURE_BENCHMARK
#   include "backend/common/benchmark/BenchState.h"
#endif


#include <algorithm>
#include <cinttypes>
#include <ctime>
#include <iterator>
#include <memory>


// 构造函数，初始化网络对象
xmrig::Network::Network(Controller *controller) :
    m_controller(controller)
{
    // 设置作业结果监听器
    JobResults::setListener(this, controller->config()->cpu().isHwAES());
    // 添加控制器监听器
    controller->addListener(this);
# 如果定义了 XMRIG_FEATURE_API，则添加监听器
#ifdef XMRIG_FEATURE_API
    controller->api()->addListener(this);
#endif

# 创建 NetworkState 对象
m_state = new NetworkState(this);

# 获取配置文件中的矿池信息
const Pools &pools = controller->config()->pools();
# 根据矿池信息创建策略对象
m_strategy = pools.createStrategy(m_state);

# 如果矿池的捐赠等级大于0，则创建 DonateStrategy 对象
if (pools.donateLevel() > 0) {
    m_donate = new DonateStrategy(controller, this);
}

# 创建 Timer 对象
m_timer = new Timer(this, kTickInterval, kTickInterval);
}


# Network 类的析构函数
xmrig::Network::~Network()
{
    # 停止 JobResults
    JobResults::stop();

    # 释放内存
    delete m_timer;
    delete m_donate;
    delete m_strategy;
    delete m_state;
}


# 连接到矿池
void xmrig::Network::connect()
{
    # 调用策略对象的 connect 方法
    m_strategy->connect();
}


# 执行命令
void xmrig::Network::execCommand(char command)
{
    # 根据命令执行相应的操作
    switch (command) {
    case 's':
    case 'S':
        # 打印结果
        m_state->printResults();
        break;

    case 'c':
    case 'C':
        # 打印连接信息
        m_state->printConnection();
        break;

    default:
        break;
    }
}


# 当策略对象和客户端对象处于活动状态时调用
void xmrig::Network::onActive(IStrategy *strategy, IClient *client)
{
    # 如果当前策略对象是 DonateStrategy，则打印日志并返回
    if (m_donate && m_donate == strategy) {
        LOG_NOTICE("%s " WHITE_BOLD("dev donate started"), Tags::network());
        return;
    }

    # 获取客户端所属的矿池信息
    const auto &pool = client->pool();

    # 如果矿池的模式是 MODE_BENCHMARK，则返回
#ifdef XMRIG_FEATURE_BENCHMARK
    if (pool.mode() == Pool::MODE_BENCHMARK) {
        return;
    }
#endif

    # 创建一个缓冲区用于存储 ZMQ 端口信息
    char zmq_buf[32] = {};
    # 如果矿池的 ZMQ 端口大于等于0，则将端口信息存储到缓冲区中
    if (client->pool().zmq_port() >= 0) {
        snprintf(zmq_buf, sizeof(zmq_buf), " (ZMQ:%d)", client->pool().zmq_port());
    }

    # 获取客户端的 TLS 版本信息
    const char *tlsVersion = client->tlsVersion();
    # 打印连接信息
    LOG_INFO("%s " WHITE_BOLD("use %s ") CYAN_BOLD("%s:%d%s ") GREEN_BOLD("%s") " " BLACK_BOLD("%s"),
             Tags::network(), client->mode(), pool.host().data(), pool.port(), zmq_buf, tlsVersion ? tlsVersion : "", client->ip().data());

    # 获取客户端的 TLS 指纹信息
    const char *fingerprint = client->tlsFingerprint();
    # 如果指纹信息不为空，则打印指纹信息
    if (fingerprint != nullptr) {
        LOG_INFO("%s " BLACK_BOLD("fingerprint (SHA-256): \"%s\""), Tags::network(), fingerprint);
    }
}


# 当配置文件发生变化时调用
void xmrig::Network::onConfigChanged(Config *config, Config *previousConfig)
{
    # 如果当前配置的连接池数与之前配置的连接池数相同，或者当前配置的连接池数为0，则直接返回，不进行后续操作
    if (config->pools() == previousConfig->pools() || !config->pools().active()) {
        return;
    }

    # 停止当前的策略
    m_strategy->stop();

    # 打印当前连接池的信息
    config->pools().print();

    # 删除当前的策略对象
    delete m_strategy;
    # 根据当前配置的连接池创建新的策略对象
    m_strategy = config->pools().createStrategy(m_state);
    # 连接到新的策略
    connect();
# 处理网络接收到的工作信息
void xmrig::Network::onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &)
{
    # 如果捐赠活动处于激活状态且不是当前策略，则返回
    if (m_donate && m_donate->isActive() && m_donate != strategy) {
        return;
    }
    
    # 设置工作信息给客户端，并指定是否为捐赠策略
    setJob(client, job, m_donate == strategy);
}


# 处理网络接收到的工作结果
void xmrig::Network::onJobResult(const JobResult &result)
{
    # 如果结果索引为1且存在捐赠策略，则提交结果并返回
    if (result.index == 1 && m_donate) {
        m_donate->submit(result);
        return;
    }
    
    # 否则，提交结果给当前策略
    m_strategy->submit(result);
}


# 处理网络登录事件
void xmrig::Network::onLogin(IStrategy *, IClient *client, rapidjson::Document &doc, rapidjson::Value &params)
{
    # 使用 rapidjson 命名空间
    using namespace rapidjson;
    # 获取文档的分配器
    auto &allocator = doc.GetAllocator();
    
    # 获取矿工的算法列表和当前矿池的算法
    Algorithms algorithms     = m_controller->miner()->algorithms();
    const Algorithm algorithm = client->pool().algorithm();
    
    # 如果算法有效，则重新排序算法列表，将当前算法置于首位
    if (algorithm.isValid()) {
        const size_t index = static_cast<size_t>(std::distance(algorithms.begin(), std::find(algorithms.begin(), algorithms.end(), algorithm)));
        if (index > 0 && index < algorithms.size()) {
            std::swap(algorithms[0], algorithms[index]);
        }
    }
    
    # 创建算法数组并添加到参数中
    Value algo(kArrayType);
    for (const auto &a : algorithms) {
        algo.PushBack(StringRef(a.name()), allocator);
    }
    params.AddMember("algo", algo, allocator);
}


# 处理网络暂停事件
void xmrig::Network::onPause(IStrategy *strategy)
{
    # 如果存在捐赠策略且当前策略为捐赠策略，则记录日志并恢复当前策略
    if (m_donate && m_donate == strategy) {
        LOG_NOTICE("%s " WHITE_BOLD("dev donate finished"), Tags::network());
        m_strategy->resume();
    }
    
    # 如果当前策略不活跃，则记录错误日志并暂停挖矿
    if (!m_strategy->isActive()) {
        LOG_ERR("%s " RED("no active pools, stop mining"), Tags::network());
        return m_controller->miner()->pause();
    }
}


# 处理网络接收到的结果被接受事件
void xmrig::Network::onResultAccepted(IStrategy *, IClient *, const SubmitResult &result, const char *error)
{
    # 获取结果的难度和难度的规模
    uint64_t diff     = result.diff;
    const char *scale = NetworkState::scaleDiff(diff);
}
    # 如果存在错误
    if (error) {
        # 记录信息：后端标签 + "rejected" + 当前已接受数量/总数量 + 差异值 + 时间单位 + 错误信息 + 耗时
        LOG_INFO("%s " RED_BOLD("rejected") " (%" PRId64 "/%" PRId64 ") diff " WHITE_BOLD("%" PRIu64 "%s") " " RED("\"%s\"") " " BLACK_BOLD("(%" PRIu64 " ms)"),
                 backend_tag(result.backend), m_state->accepted(), m_state->rejected(), diff, scale, error, result.elapsed);
    }
    # 如果没有错误
    else {
        # 记录信息：后端标签 + "accepted" + 当前已接受数量/总数量 + 差异值 + 时间单位 + 耗时
        LOG_INFO("%s " GREEN_BOLD("accepted") " (%" PRId64 "/%" PRId64 ") diff " WHITE_BOLD("%" PRIu64 "%s") " " BLACK_BOLD("(%" PRIu64 " ms)"),
                 backend_tag(result.backend), m_state->accepted(), m_state->rejected(), diff, scale, result.elapsed);
    }
}

# 当验证算法时，检查矿工是否启用了该算法，如果没有启用，则将 ok 设置为 false
void xmrig::Network::onVerifyAlgorithm(IStrategy *, const IClient *, const Algorithm &algorithm, bool *ok)
{
    if (!m_controller->miner()->isEnabled(algorithm)) {
        *ok = false;

        return;
    }
}

#ifdef XMRIG_FEATURE_API
# 当收到 API 请求时，根据请求类型进行相应的处理
void xmrig::Network::onRequest(IApiRequest &request)
{
    # 如果请求类型是 REQ_SUMMARY，则接受请求，并获取结果和连接信息
    if (request.type() == IApiRequest::REQ_SUMMARY) {
        request.accept();

        getResults(request.reply(), request.doc(), request.version());
        getConnection(request.reply(), request.doc(), request.version());
    }
}
#endif

# 设置矿工的工作任务
void xmrig::Network::setJob(IClient *client, const Job &job, bool donate)
{
#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果没有进行基准测试，则执行以下代码块
    if (!BenchState::size())
#   endif
    {
        # 获取工作任务的难度和缩放比例
        uint64_t diff       = job.diff();
        const char *scale   = NetworkState::scaleDiff(diff);

        # 初始化 zmq_buf 字符数组
        char zmq_buf[32] = {};
        # 如果矿池的 zmq 端口大于等于 0，则将 zmq_buf 设置为 "(ZMQ:端口号)"
        if (client->pool().zmq_port() >= 0) {
            snprintf(zmq_buf, sizeof(zmq_buf), " (ZMQ:%d)", client->pool().zmq_port());
        }

        # 初始化 tx_buf 字符数组
        char tx_buf[32] = {};
        # 获取工作任务的交易数量，如果大于 0，则将 tx_buf 设置为 "(交易数量 tx)"
        const uint32_t num_transactions = job.getNumTransactions();
        if (num_transactions > 0) {
            snprintf(tx_buf, sizeof(tx_buf), " (%u tx)", num_transactions);
        }

        # 初始化 height_buf 字符数组
        char height_buf[64] = {};
        # 如果工作任务的高度大于 0，则将 height_buf 设置为 " height 高度"
        if (job.height() > 0) {
            snprintf(height_buf, sizeof(height_buf), " height " WHITE_BOLD("%" PRIu64), job.height());
        }

        # 打印日志，显示新的工作任务的相关信息
        LOG_INFO("%s " MAGENTA_BOLD("new job") " from " WHITE_BOLD("%s:%d%s") " diff " WHITE_BOLD("%" PRIu64 "%s") " algo " WHITE_BOLD("%s") "%s%s",
                 Tags::network(), client->pool().host().data(), client->pool().port(), zmq_buf, diff, scale, job.algorithm().name(), height_buf, tx_buf);
    }

    # 如果不是捐赠任务，并且捐赠开关打开，则更新捐赠策略
    if (!donate && m_donate) {
        static_cast<DonateStrategy *>(m_donate)->update(client, job);
    }

    # 设置矿工的工作任务
    m_controller->miner()->setJob(job, donate);
}

# 执行网络的 tick 操作
void xmrig::Network::tick()
{
    # 获取当前时间戳
    const uint64_t now = Chrono::steadyMSecs();

    # 执行策略的 tick 操作
    m_strategy->tick(now);
}
    # 如果 m_donate 对象存在，则调用其 tick 方法，并传入当前时间参数
    if (m_donate) {
        m_donate->tick(now);
    }
#ifdef XMRIG_FEATURE_API
// 如果定义了 XMRIG_FEATURE_API 宏，则执行以下代码块
// 调用 m_controller 对象的 api() 方法，并调用其 tick() 方法
m_controller->api()->tick();
#endif
}


#ifdef XMRIG_FEATURE_API
// 如果定义了 XMRIG_FEATURE_API 宏，则执行以下代码块
// 定义一个名为 getConnection 的函数，参数为 reply、doc 和 version
void xmrig::Network::getConnection(rapidjson::Value &reply, rapidjson::Document &doc, int version) const
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取 doc 对象的分配器
    auto &allocator = doc.GetAllocator();

    // 向 reply 对象添加一个名为 "algo" 的成员，值为 m_state 对象的 algorithm() 方法返回的 JSON 对象，使用 allocator 进行分配
    reply.AddMember("algo",         m_state->algorithm().toJSON(), allocator);
    // 向 reply 对象添加一个名为 "connection" 的成员，值为 m_state 对象的 getConnection 方法返回的 JSON 对象，使用 allocator 进行分配
    reply.AddMember("connection",   m_state->getConnection(doc, version), allocator);
}


// 定义一个名为 getResults 的函数，参数为 reply、doc 和 version
void xmrig::Network::getResults(rapidjson::Value &reply, rapidjson::Document &doc, int version) const
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取 doc 对象的分配器
    auto &allocator = doc.GetAllocator();

    // 向 reply 对象添加一个名为 "results" 的成员，值为 m_state 对象的 getResults 方法返回的 JSON 对象，使用 allocator 进行分配
    reply.AddMember("results", m_state->getResults(doc, version), allocator);
}
#endif
```