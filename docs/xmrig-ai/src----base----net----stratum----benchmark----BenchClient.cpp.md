# `xmrig\src\base\net\stratum\benchmark\BenchClient.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/benchmark/BenchClient.h"
#include "3rdparty/fmt/core.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/benchmark/BenchState.h"
#include "backend/common/interfaces/IBackend.h"
#include "backend/cpu/Cpu.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/kernel/interfaces/IClientListener.h"
#include "base/net/dns/Dns.h"
#include "base/net/dns/DnsRecords.h"
#include "base/net/http/Fetch.h"
#include "base/net/http/HttpData.h"
#include "base/net/http/HttpListener.h"
#include "base/net/stratum/benchmark/BenchConfig.h"
#include "base/tools/Cvt.h"
#include "version.h"

#ifdef XMRIG_FEATURE_DMI
#   include "hw/dmi/DmiReader.h"
#endif

// BenchClient类的构造函数，接受一个BenchConfig类型的智能指针和一个IClientListener指针
xmrig::BenchClient::BenchClient(const std::shared_ptr<BenchConfig> &benchmark, IClientListener* listener) :
    m_listener(listener), // 初始化m_listener成员变量
    m_benchmark(benchmark), // 初始化m_benchmark成员变量
    m_hash(benchmark->hash()) // 初始化m_hash成员变量为benchmark的哈希值
{
    // 创建一个包含112*2+1个'0'字符的向量，并将最后一个字符设置为'\0'
    std::vector<char> blob(112 * 2 + 1, '0');
    blob.back() = '\0';

#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果 benchmark 对象的算法是 GHOSTRIDER_RTM
    if (m_benchmark->algorithm() == Algorithm::GHOSTRIDER_RTM) {
        # 计算旋转值除以20的商，并对1取余
        const uint32_t q = (benchmark->rotation() / 20) & 1;
        # 计算旋转值对20取余
        const uint32_t r = benchmark->rotation() % 20;

        # 静态常量数组，包含20个长度为3的数组
        static constexpr uint32_t indices[20][3] = {
             { 0, 1, 2 },
             { 0, 1, 3 },
             { 0, 1, 4 },
             { 0, 1, 5 },
             { 0, 2, 3 },
             { 0, 2, 4 },
             { 0, 2, 5 },
             { 0, 3, 4 },
             { 0, 3, 5 },
             { 0, 4, 5 },
             { 1, 2, 3 },
             { 1, 2, 4 },
             { 1, 2, 5 },
             { 1, 3, 4 },
             { 1, 3, 5 },
             { 1, 4, 5 },
             { 2, 3, 4 },
             { 2, 3, 5 },
             { 2, 4, 5 },
             { 3, 4, 5 },
        };

        # 根据计算得到的索引值更新 blob 数组的值
        blob[ 8] = '0' + indices[r][q ? 2 : 1];
        blob[ 9] = '0' + indices[r][0];
        blob[11] = '0' + indices[r][q ? 1 : 2];
    }
#   endif
#   endif 语句的结束

    # 设置作业的算法
    m_job.setAlgorithm(m_benchmark->algorithm());
    # 设置作业的数据
    m_job.setBlob(blob.data());
    # 设置作业的难度
    m_job.setDiff(std::numeric_limits<uint64_t>::max());
    # 设置作业的高度
    m_job.setHeight(1);
    # 设置作业的ID
    m_job.setId("00000000");

    # 将种子哈希的末尾设置为'\0'
    blob[Job::kMaxSeedSize * 2] = '\0';
    # 设置作业的种子哈希
    m_job.setSeedHash(blob.data());

    # 初始化BenchState对象
    BenchState::init(this, m_benchmark->size());

#   ifdef XMRIG_FEATURE_HTTP
    # 如果是提交且算法家族为RANDOM_X，则设置模式为ONLINE_BENCH，并设置token
    if (m_benchmark->isSubmit() && (m_benchmark->algorithm().family() == Algorithm::RANDOM_X)) {
        m_mode  = ONLINE_BENCH;
        m_token = m_benchmark->token();

        return;
    }

    # 如果benchmark的ID不为空，则设置作业的ID和token，并设置模式为ONLINE_VERIFY
    if (!m_benchmark->id().isEmpty()) {
        m_job.setId(m_benchmark->id());
        m_token = m_benchmark->token();
        m_mode  = ONLINE_VERIFY;

        return;
    }
#   endif

    # 如果hash不为空且设置种子成功，则设置模式为STATIC_VERIFY
    if (m_hash && setSeed(m_benchmark->seed())) {
        m_mode = STATIC_VERIFY;

        return;
    }

    # 设置作业的基准大小
    m_job.setBenchSize(m_benchmark->size());

}


# BenchClient类的析构函数
xmrig::BenchClient::~BenchClient()
{
    # 销毁BenchState对象
    BenchState::destroy();
}


# 返回BenchClient类的标签
const char *xmrig::BenchClient::tag() const
{
    return Tags::bench();
}


# 连接函数
void xmrig::BenchClient::connect()
{
#   ifdef XMRIG_FEATURE_HTTP
    # 如果模式为ONLINE_BENCH或ONLINE_VERIFY，则解析
    if (m_mode == ONLINE_BENCH || m_mode == ONLINE_VERIFY) {
        return resolve();
    }
#   endif

    # 启动连接
    start();
}


# 设置矿池
void xmrig::BenchClient::setPool(const Pool &pool)
{
    m_pool = pool;
}


# 当基准测试完成时的回调函数
void xmrig::BenchClient::onBenchDone(uint64_t result, uint64_t diff, uint64_t ts)
{
    # 设置结果、难度和完成时间
    m_result    = result;
    m_diff      = diff;
    m_doneTime  = ts;

#   ifdef XMRIG_FEATURE_HTTP
    # 如果token不为空，则发送DONE_BENCH
    if (!m_token.isEmpty()) {
        send(DONE_BENCH);
    }
#   endif

    # 获取参考哈希值
    const uint64_t ref = referenceHash();
    # 根据参考哈希值的情况设置颜色
    const char *color  = ref ? ((result == ref) ? GREEN_BOLD_S : RED_BOLD_S) : BLACK_BOLD_S;

    # 计算完成时间，并打印日志
    const double dt = static_cast<int64_t>(ts - m_readyTime) / 1000.0;
    LOG_NOTICE("%s " WHITE_BOLD("benchmark finished in ") CYAN_BOLD("%.3f seconds (%.1f h/s)") WHITE_BOLD_S " hash sum = " CLEAR "%s%016" PRIX64 CLEAR, tag(), dt, BenchState::size() / dt, color, result);
    # 如果 m_token 为空，执行 printExit() 函数
    if (m_token.isEmpty()) {
        printExit();
    }
}

// 当基准测试准备就绪时调用，记录准备就绪的时间戳、线程数和后端信息
void xmrig::BenchClient::onBenchReady(uint64_t ts, uint32_t threads, const IBackend *backend)
{
    m_readyTime = ts; // 记录准备就绪的时间戳
    m_threads   = threads; // 记录线程数
    m_backend   = backend; // 记录后端信息

#   ifdef XMRIG_FEATURE_HTTP
    if (m_mode == ONLINE_BENCH) { // 如果模式为在线基准测试
        send(CREATE_BENCH); // 发送创建基准测试的请求
    }
#   endif
}

// 当收到 HTTP 数据时调用
void xmrig::BenchClient::onHttpData(const HttpData &data)
{
#   ifdef XMRIG_FEATURE_HTTP
    rapidjson::Document doc;

    try {
        doc = data.json(); // 尝试解析收到的 JSON 数据
    } catch (const std::exception &ex) {
        return setError(ex.what()); // 如果解析失败，设置错误信息并返回
    }

    if (data.status != 200) { // 如果 HTTP 状态码不为 200
        return setError(data.statusName()); // 设置错误信息并返回
    }

    switch (m_request) {
    case GET_BENCH:
        return onGetReply(doc); // 处理获取基准测试的回复

    case CREATE_BENCH:
        return onCreateReply(doc); // 处理创建基准测试的回复

    case DONE_BENCH:
        return onDoneReply(doc); // 处理完成基准测试的回复

    default:
        break;
    }
#   endif
}

// 当 DNS 解析完成时调用
void xmrig::BenchClient::onResolved(const DnsRecords &records, int status, const char *error)
{
#   ifdef XMRIG_FEATURE_HTTP
    assert(!m_httpListener); // 断言 HTTP 监听器不存在

    m_dns.reset(); // 重置 DNS 解析器

    if (status < 0) { // 如果 DNS 解析状态小于 0
        return setError(error, "DNS error"); // 设置 DNS 错误信息并返回
    }

    m_ip            = records.get().ip(); // 获取解析到的 IP 地址
    m_httpListener  = std::make_shared<HttpListener>(this, tag()); // 创建 HTTP 监听器

    if (m_mode == ONLINE_BENCH) { // 如果模式为在线基准测试
        start(); // 开始基准测试
    }
    else {
        send(GET_BENCH); // 发送获取基准测试的请求
    }
#   endif
}

// 设置种子值
bool xmrig::BenchClient::setSeed(const char *seed)
{
    if (!seed) {
        return false; // 如果种子值为空，返回 false
    }

    size_t size = strlen(seed); // 获取种子值的长度
    if (size % 2 != 0) {
        return false; // 如果长度为奇数，返回 false
    }

    size /= 2; // 将长度除以 2
    if (size < 4 || size >= m_job.size()) {
        return false; // 如果长度小于 4 或大于等于作业大小，返回 false
    }

    if (!Cvt::fromHex(m_job.blob(), m_job.size(), seed, size * 2)) {
        return false; // 如果十六进制转换失败，返回 false
    }

    m_job.setBenchSize(BenchState::size()); // 设置基准测试大小

    LOG_NOTICE("%s " WHITE_BOLD("seed ") BLACK_BOLD("%s"), tag(), seed); // 记录种子值

    return true; // 返回 true
}

// 获取参考哈希值
uint64_t xmrig::BenchClient::referenceHash() const
{
    if (m_hash || m_mode == ONLINE_BENCH) {
        return m_hash; // 如果哈希值存在或模式为在线基准测试，返回哈希值
    }
    # 调用BenchState类的referenceHash方法，传入m_job.algorithm()、BenchState::size()和m_threads作为参数，并返回结果
    return BenchState::referenceHash(m_job.algorithm(), BenchState::size(), m_threads);
}

// 打印退出信息
void xmrig::BenchClient::printExit() const
{
    LOG_INFO("%s " WHITE_BOLD("press ") MAGENTA_BOLD("Ctrl+C") WHITE_BOLD(" to exit"), tag());
}

// 开始基准测试
void xmrig::BenchClient::start()
{
    const uint32_t size = BenchState::size();

    // 打印开始基准测试的信息
    LOG_NOTICE("%s " MAGENTA_BOLD("start benchmark ") "hashes " CYAN_BOLD("%u%s") " algo " WHITE_BOLD("%s"),
               tag(),
               size < 1000000 ? size / 1000 : size / 1000000,
               size < 1000000 ? "K" : "M",
               m_job.algorithm().name());

    // 通知监听器登录成功
    m_listener->onLoginSuccess(this);
    // 通知监听器接收到作业
    m_listener->onJobReceived(this, m_job, rapidjson::Value());
}

// 处理创建回复
#ifdef XMRIG_FEATURE_HTTP
void xmrig::BenchClient::onCreateReply(const rapidjson::Value &value)
{
    // 设置开始时间
    m_startTime = Chrono::steadyMSecs();
    // 获取令牌
    m_token     = Json::getString(value, BenchConfig::kToken);

    // 设置作业ID和种子
    m_job.setId(Json::getString(value, BenchConfig::kId));
    setSeed(Json::getString(value, BenchConfig::kSeed));

    // 通知监听器接收到作业
    m_listener->onJobReceived(this, m_job, rapidjson::Value());

    // 发送开始基准测试请求
    send(START_BENCH);
}

// 处理完成回复
void xmrig::BenchClient::onDoneReply(const rapidjson::Value &)
{
    // 打印基准测试提交信息
    LOG_NOTICE("%s " WHITE_BOLD("benchmark submitted ") CYAN_BOLD("https://xmrig.com/benchmark/%s"), tag(), m_job.id().data());
    // 打印退出信息
    printExit();
}

// 处理获取回复
void xmrig::BenchClient::onGetReply(const rapidjson::Value &value)
{
    // 获取哈希值并转换为无符号长长整型
    const char *hash = Json::getString(value, BenchConfig::kHash);
    if (hash) {
        m_hash = strtoull(hash, nullptr, 16);
    }

    // 设置基准测试大小和算法
    BenchState::setSize(Json::getUint(value, BenchConfig::kSize));
    m_job.setAlgorithm(Json::getString(value, BenchConfig::kAlgo));
    setSeed(Json::getString(value, BenchConfig::kSeed));

    // 开始基准测试
    start();
}

// 解析API主机名
void xmrig::BenchClient::resolve()
{
    m_dns = Dns::resolve(BenchConfig::kApiHost, this);
}

// 发送请求
void xmrig::BenchClient::send(Request request)
{
    using namespace rapidjson;

    // 创建JSON文档和分配器
    Document doc(kObjectType);
    auto &allocator = doc.GetAllocator();
    m_request       = request;

    // 根据请求类型执行相应操作
    switch (m_request) {
    # 当接收到 GET_BENCH 消息时执行以下代码块
    case GET_BENCH:
        # 创建一个 HTTP GET 请求对象，请求地址为 m_ip 和 BenchConfig::kApiPort 拼接而成的地址，路径为 "/1/benchmark/" 加上 m_job.id() 的结果，使用 BenchConfig::kApiTLS，启用日志
        FetchRequest req(HTTP_GET, m_ip, BenchConfig::kApiPort, fmt::format("/1/benchmark/{}", m_job.id()).c_str(), BenchConfig::kApiTLS, true);
        # 发起请求并将结果传递给 m_httpListener
        fetch(tag(), std::move(req), m_httpListener);
        # 跳出 switch 语句
        break;

    # 当接收到 CREATE_BENCH 消息时执行以下代码块
    case CREATE_BENCH:
        # 向 JSON 文档中添加键值对，键为 BenchConfig::kSize，值为 m_benchmark->size()，使用 allocator 分配内存
        doc.AddMember(StringRef(BenchConfig::kSize),    m_benchmark->size(), allocator);
        # 向 JSON 文档中添加键值对，键为 BenchConfig::kAlgo，值为 m_benchmark->algorithm().toJSON()，使用 allocator 分配内存
        doc.AddMember(StringRef(BenchConfig::kAlgo),    m_benchmark->algorithm().toJSON(), allocator);
        # 向 JSON 文档中添加键值对，键为 BenchConfig::kUser，值为 m_benchmark->user().toJSON()，使用 allocator 分配内存
        doc.AddMember(StringRef(BenchConfig::kUser),    m_benchmark->user().toJSON(), allocator);
        # 向 JSON 文档中添加键值对，键为 "version"，值为 APP_VERSION，使用 allocator 分配内存
        doc.AddMember("version",                        APP_VERSION, allocator);
        # 向 JSON 文档中添加键值对，键为 "threads"，值为 m_threads，使用 allocator 分配内存
        doc.AddMember("threads",                        m_threads, allocator);
        # 向 JSON 文档中添加键值对，键为 "steady_ready_ts"，值为 m_readyTime，使用 allocator 分配内存
        doc.AddMember("steady_ready_ts",                m_readyTime, allocator);
        # 向 JSON 文档中添加键值对，键为 "cpu"，值为 Cpu::toJSON(doc)，使用 allocator 分配内存
        doc.AddMember("cpu",                            Cpu::toJSON(doc), allocator);
// 如果定义了 XMRIG_FEATURE_DMI，则执行以下代码块
#ifdef XMRIG_FEATURE_DMI
    // 如果 benchmark 对象包含 DMI 信息
    if (m_benchmark->isDMI()) {
        // 创建 DmiReader 对象
        DmiReader reader;
        // 读取 DMI 信息
        if (reader.read()) {
            // 将 DMI 信息转换为 JSON 格式，并添加到 doc 对象中
            doc.AddMember("dmi", reader.toJSON(doc), allocator);
        }
    }
#endif

    // 创建 HTTP POST 请求对象
    FetchRequest req(HTTP_POST, m_ip, BenchConfig::kApiPort, "/1/benchmark", doc, BenchConfig::kApiTLS, true);

    // 如果 token 不为空
    if (!m_token.isEmpty()) {
        // 在请求头中插入 Authorization 字段
        req.headers.insert({ "Authorization", fmt::format("Bearer {}", m_token)});
    }

    // 发起 HTTP 请求
    fetch(tag(), std::move(req), m_httpListener);
    // 结束 switch 语句块
    }
    break;

// 当 case 为 START_BENCH 时执行以下代码块
case START_BENCH:
    // 添加 steady_start_ts 字段到 doc 对象中
    doc.AddMember("steady_start_ts",    m_startTime, allocator);
    // 更新请求
    update(doc);
    break;

// 当 case 为 DONE_BENCH 时执行以下代码块
case DONE_BENCH:
    // 添加 steady_done_ts 字段到 doc 对象中
    doc.AddMember("steady_done_ts",     m_doneTime, allocator);
    // 添加 hash、diff 和 backend 字段到 doc 对象中
    doc.AddMember("hash",               Value(fmt::format("{:016X}", m_result).c_str(), allocator), allocator);
    doc.AddMember("diff",               m_diff, allocator);
    doc.AddMember("backend",            m_backend->toJSON(doc), allocator);
    // 更新请求
    update(doc);
    break;

// 当 case 为 NO_REQUEST 时执行以下代码块
case NO_REQUEST:
    // 什么也不做
    break;
}

// 设置错误信息
void xmrig::BenchClient::setError(const char *message, const char *label)
{
    // 输出错误信息
    LOG_ERR("%s " RED("%s: ") RED_BOLD("\"%s\""), tag(), label ? label : "benchmark failed", message);
    // 打印退出信息
    printExit();
    // 销毁 BenchState 对象
    BenchState::destroy();
}

// 更新请求
void xmrig::BenchClient::update(const rapidjson::Value &body)
{
    // 断言 token 不为空
    assert(!m_token.isEmpty());
    // 创建 HTTP PATCH 请求对象
    FetchRequest req(HTTP_PATCH, m_ip, BenchConfig::kApiPort, fmt::format("/1/benchmark/{}", m_job.id()).c_str(), body, BenchConfig::kApiTLS, true);
    // 在请求头中插入 Authorization 字段
    req.headers.insert({ "Authorization", fmt::format("Bearer {}", m_token)});
    // 发起 HTTP 请求
    fetch(tag(), std::move(req), m_httpListener);
}
#endif
```