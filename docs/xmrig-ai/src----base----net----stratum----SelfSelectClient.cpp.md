# `xmrig\src\base\net\stratum\SelfSelectClient.cpp`

```
/* XMRig
 * 版权所有（c）2019       jtgrassie       <https://github.com/jtgrassie>
 * 版权所有（c）2021       Hansie Odendaal <https://github.com/hansieodendaal>
 * 版权所有（c）2018-2021  SChernykh       <https://github.com/SChernykh>
 * 版权所有（c）2016-2021  XMRig           <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/net/stratum/SelfSelectClient.h"  // 导入SelfSelectClient头文件
#include "3rdparty/rapidjson/document.h"  // 导入rapidjson库的document头文件
#include "3rdparty/rapidjson/error/en.h"  // 导入rapidjson库的错误处理头文件
#include "base/io/json/Json.h"  // 导入Json库的头文件
#include "base/io/json/JsonRequest.h"  // 导入JsonRequest头文件
#include "base/io/log/Log.h"  // 导入Log库的头文件
#include "base/io/log/Tags.h"  // 导入Tags头文件
#include "base/net/http/Fetch.h"  // 导入Fetch头文件
#include "base/net/http/HttpData.h"  // 导入HttpData头文件
#include "base/net/stratum/Client.h"  // 导入Client头文件
#include "net/JobResult.h"  // 导入JobResult头文件
#include "base/tools/Cvt.h"  // 导入Cvt头文件


namespace xmrig {

static const char *kBlob                = "blob";  // 定义常量kBlob并赋值为"blob"
static const char *kBlockhashingBlob    = "blockhashing_blob";  // 定义常量kBlockhashingBlob并赋值为"blockhashing_blob"
static const char *kBlocktemplateBlob   = "blocktemplate_blob";  // 定义常量kBlocktemplateBlob并赋值为"blocktemplate_blob"
static const char *kDifficulty          = "difficulty";  // 定义常量kDifficulty并赋值为"difficulty"
static const char *kHeight              = "height";  // 定义常量kHeight并赋值为"height"
static const char *kId                  = "id";  // 定义常量kId并赋值为"id"
static const char *kJobId               = "job_id";  // 定义常量kJobId并赋值为"job_id"
static const char *kNextSeedHash        = "next_seed_hash";  // 定义常量kNextSeedHash并赋值为"next_seed_hash"
static const char *kPrevHash            = "prev_hash";  // 定义常量kPrevHash并赋值为"prev_hash"
static const char *kSeedHash            = "seed_hash";  // 定义常量kSeedHash并赋值为"seed_hash"
// 定义必需的字段数组
static const char * const required_fields[] = { kBlocktemplateBlob, kBlockhashingBlob, kHeight, kDifficulty, kPrevHash };

} /* namespace xmrig */

// SelfSelectClient 类的构造函数
xmrig::SelfSelectClient::SelfSelectClient(int id, const char *agent, IClientListener *listener, bool submitToOrigin) :
    m_submitToOrigin(submitToOrigin),  // 初始化成员变量 m_submitToOrigin
    m_listener(listener)  // 初始化成员变量 m_listener
{
    m_httpListener  = std::make_shared<HttpListener>(this);  // 创建并初始化 m_httpListener
    m_client        = new Client(id, agent, this);  // 创建并初始化 m_client
}

// SelfSelectClient 类的析构函数
xmrig::SelfSelectClient::~SelfSelectClient()
{
    delete m_client;  // 释放 m_client 的内存
}

// 提交作业结果
int64_t xmrig::SelfSelectClient::submit(const JobResult &result)
{
    if (m_submitToOrigin) {
        submitOriginDaemon(result);  // 如果需要提交到原始守护进程，则调用 submitOriginDaemon 方法
    }

    uint64_t submit_result = m_client->submit(result);  // 调用 m_client 的 submit 方法，并将结果保存到 submit_result

    if (m_submitToOrigin) {
        // 确保在块提交后最新的块模板可用
        getBlockTemplate();  // 调用 getBlockTemplate 方法
    }

    return submit_result;  // 返回提交结果
}

// 定时器触发的方法
void xmrig::SelfSelectClient::tick(uint64_t now)
{
    m_client->tick(now);  // 调用 m_client 的 tick 方法，传入当前时间

    if (m_state == RetryState) {
        if (Chrono::steadyMSecs() - m_timestamp < m_retryPause) {
            return;  // 如果重试状态下，且当前时间与时间戳的差小于重试暂停时间，则直接返回
        }

        getBlockTemplate();  // 否则调用 getBlockTemplate 方法
    }
}

// 当接收到作业时触发的方法
void xmrig::SelfSelectClient::onJobReceived(IClient *, const Job &job, const rapidjson::Value &)
{
    m_job = job;  // 将接收到的作业保存到 m_job

    getBlockTemplate();  // 调用 getBlockTemplate 方法
}

// 当登录时触发的方法
void xmrig::SelfSelectClient::onLogin(IClient *, rapidjson::Document &doc, rapidjson::Value &params)
{
    params.AddMember("mode", "self-select", doc.GetAllocator());  // 向参数中添加 "mode" 字段为 "self-select"

    m_listener->onLogin(this, doc, params);  // 调用监听器的 onLogin 方法
}

// 解析响应的方法
bool xmrig::SelfSelectClient::parseResponse(int64_t id, rapidjson::Value &result, const rapidjson::Value &error)
{
    if (id == -1) {
        return false;  // 如果 id 为 -1，则返回 false
    }

    if (error.IsObject()) {
        LOG_ERR("[%s] error: " RED_BOLD("\"%s\"") RED_S ", code: %d", pool().daemon().url().data(), Json::getString(error, "message"), Json::getInt(error, "code"));
        // 如果 error 是对象，则记录错误信息并返回 false
        return false;
    }

    if (!result.IsObject()) {
        return false;  // 如果 result 不是对象，则返回 false
    }
}
    // 遍历必需字段列表，检查结果中是否存在这些字段
    for (auto field : required_fields) {
        // 如果结果中缺少必需字段，则记录错误信息并返回 false
        if (!result.HasMember(field)) {
            LOG_ERR("[%s] required field " RED_BOLD("\"%s\"") RED_S " not found", pool().daemon().url().data(), field);
            return false;
        }
    }

    // 从结果中获取指定字段的字符串值
    const char *blobData = Json::getString(result, kBlockhashingBlob);
    // 如果矿池币种有效
    if (pool().coin().isValid()) {
        // 初始化块数据版本号为 0
        uint8_t blobVersion = 0;
        // 如果块数据存在
        if (blobData) {
            // 将块数据转换为十六进制，并存储到 blobVersion 中
            Cvt::fromHex(&blobVersion, 1, blobData, 2);
        }
        // 设置作业的算法类型
        m_job.setAlgorithm(pool().coin().algorithm(blobVersion));
    }

    // 设置作业的块数据
    if (!m_job.setBlob(blobData)) {
        return false;
    }

    // 设置作业的高度
    m_job.setHeight(Json::getUint64(result, kHeight));
    // 设置作业的种子哈希
    m_job.setSeedHash(Json::getString(result, kSeedHash));

    // 提交块模板
    submitBlockTemplate(result);

    // 返回 true
    return true;
// 设置客户端状态为等待状态
void xmrig::SelfSelectClient::getBlockTemplate()
{
    setState(WaitState); // 设置状态为等待状态

    using namespace rapidjson; // 使用 rapidjson 命名空间
    Document doc(kObjectType); // 创建一个 rapidjson 文档对象
    auto &allocator = doc.GetAllocator(); // 获取文档对象的分配器

    Value params(kObjectType); // 创建一个 rapidjson 值对象
    params.AddMember("wallet_address",  m_job.poolWallet().toJSON(), allocator); // 添加钱包地址到参数中
    params.AddMember("extra_nonce",     m_job.extraNonce().toJSON(), allocator); // 添加额外的 nonce 到参数中

    JsonRequest::create(doc, m_sequence++, "getblocktemplate", params); // 创建一个 JSON 请求

    FetchRequest req(HTTP_POST, pool().daemon().host(), pool().daemon().port(), "/json_rpc", doc, pool().daemon().isTLS(), isQuiet()); // 创建一个 HTTP 请求
    fetch(tag(), std::move(req), m_httpListener); // 发起 HTTP 请求
}


// 重试函数
void xmrig::SelfSelectClient::retry()
{
    setState(RetryState); // 设置状态为重试状态
}


// 设置客户端状态
void xmrig::SelfSelectClient::setState(State state)
{
    if (m_state == state) { // 如果当前状态和要设置的状态相同，则直接返回
        return;
    }

    switch (state) { // 根据要设置的状态进行处理
    case IdleState: // 如果是空闲状态
        m_timestamp = 0; // 时间戳清零
        m_failures  = 0; // 失败次数清零
        break;

    case WaitState: // 如果是等待状态
        m_timestamp = Chrono::steadyMSecs(); // 获取当前时间戳
        break;

    case RetryState: // 如果是重试状态
        m_timestamp = Chrono::steadyMSecs(); // 获取当前时间戳

        if (m_failures > m_retries) { // 如果失败次数超过重试次数
            m_listener->onClose(this, static_cast<int>(m_failures)); // 调用监听器的关闭函数
        }

        m_failures++; // 失败次数加一
        break;
    }

    m_state = state; // 设置客户端状态为指定状态
}


// 提交区块模板
void xmrig::SelfSelectClient::submitBlockTemplate(rapidjson::Value &result)
{
    using namespace rapidjson; // 使用 rapidjson 命名空间
    Document doc(kObjectType); // 创建一个 rapidjson 文档对象
    auto &allocator = doc.GetAllocator(); // 获取文档对象的分配器

    m_blocktemplate = Json::getString(result,kBlocktemplateBlob); // 获取区块模板
    m_blockDiff     = Json::getUint64(result, kDifficulty); // 获取区块难度

    Value params(kObjectType); // 创建一个 rapidjson 值对象
    params.AddMember(StringRef(kId),            m_job.clientId().toJSON(), allocator); // 添加客户端 ID 到参数中
    params.AddMember(StringRef(kJobId),         m_job.id().toJSON(), allocator); // 添加作业 ID 到参数中
    params.AddMember(StringRef(kBlob),          result[kBlocktemplateBlob], allocator); // 添加区块模板到参数中
    params.AddMember(StringRef(kHeight),        m_job.height(), allocator); // 添加高度到参数中
    # 将 kDifficulty 对应的值添加到 params 对象中
    params.AddMember(StringRef(kDifficulty),    result[kDifficulty], allocator);
    # 将 kPrevHash 对应的值添加到 params 对象中
    params.AddMember(StringRef(kPrevHash),      result[kPrevHash], allocator);
    # 将 kSeedHash 对应的值添加到 params 对象中
    params.AddMember(StringRef(kSeedHash),      result[kSeedHash], allocator);
    # 将 kNextSeedHash 对应的值添加到 params 对象中
    params.AddMember(StringRef(kNextSeedHash),  result[kNextSeedHash], allocator);

    # 创建一个 JSON 请求对象，使用 doc 和 sequence() 作为参数，"block_template" 作为请求类型，params 作为请求参数
    JsonRequest::create(doc, sequence(), "block_template", params);

    # 发送请求，并在回调函数中处理返回结果
    send(doc, [this](const rapidjson::Value &result, bool success, uint64_t) {
        # 如果请求失败，则记录错误信息并重试
        if (!success) {
            if (!isQuiet()) {
                LOG_ERR("[%s] error: " RED_BOLD("\"%s\"") RED_S ", code: %d", pool().daemon().url().data(), Json::getString(result, "message"), Json::getInt(result, "code"));
            }
            return retry();
        }

        # 如果 m_active 为假，则直接返回
        if (!m_active) {
            return;
        }

        # 如果 m_failures 大于 m_retries，则调用 m_listener 的 onLoginSuccess 方法
        if (m_failures > m_retries) {
            m_listener->onLoginSuccess(this);
        }

        # 设置状态为 IdleState
        setState(IdleState);
        # 调用 m_listener 的 onJobReceived 方法，传入 this, m_job, 以及一个空的 rapidjson::Value 对象
        m_listener->onJobReceived(this, m_job, rapidjson::Value{});
    });
}

// 提交结果到原始守护进程
void xmrig::SelfSelectClient::submitOriginDaemon(const JobResult& result)
{
    // 如果结果难度为0或者块难度为0，则返回
    if (result.diff == 0 || m_blockDiff == 0) {
        return;
    }

    // 如果实际难度小于块难度，则增加未提交计数，记录日志，然后返回
    if (result.actualDiff() < m_blockDiff) {
        m_originNotSubmitted++;
        LOG_DEBUG("%s " RED_BOLD("not submitted to origin daemon, difficulty too low") " (%" PRId64 "/%" PRId64 ") "
            BLACK_BOLD(" diff ") BLACK_BOLD("%" PRIu64) BLACK_BOLD(" vs. ") BLACK_BOLD("%" PRIu64),
            Tags::origin(), m_originSubmitted, m_originNotSubmitted, m_blockDiff, result.actualDiff());
        return;
    }

    // 获取块模板数据
    char *data = m_blocktemplate.data();
    // 将结果的nonce转换为16进制并写入块模板数据
    Cvt::toHex(data + 78, 8, reinterpret_cast<const uint8_t*>(&result.nonce), 4);

    // 创建JSON对象
    using namespace rapidjson;
    Document doc(kObjectType);

    // 创建参数数组
    Value params(kArrayType);
    // 将块模板转换为JSON并添加到参数数组
    params.PushBack(m_blocktemplate.toJSON(), doc.GetAllocator());

    // 创建JSON-RPC请求
    JsonRequest::create(doc, m_sequence, "submitblock", params);
    // 将请求结果存储到结果映射中
    m_results[m_sequence] = SubmitResult(m_sequence, result.diff, result.actualDiff(), 0, result.backend);

    // 创建HTTP请求
    FetchRequest req(HTTP_POST, pool().daemon().host(), pool().daemon().port(), "/json_rpc", doc, pool().daemon().isTLS(), isQuiet());
    // 发送HTTP请求
    fetch(tag(), std::move(req), m_httpListener);

    // 增加已提交计数，记录日志
    m_originSubmitted++;
    LOG_INFO("%s " GREEN_BOLD("submitted to origin daemon") " (%" PRId64 "/%" PRId64 ") "
        " diff " WHITE("%" PRIu64) " vs. " WHITE("%" PRIu64),
        Tags::origin(), m_originSubmitted, m_originNotSubmitted, m_blockDiff, result.actualDiff(), result.diff);
}

// 处理HTTP数据
void xmrig::SelfSelectClient::onHttpData(const HttpData &data)
{
    // 如果HTTP状态不是200，则重试
    if (data.status != 200) {
        return retry();
    }

    // 解析JSON数据
    rapidjson::Document doc;
    if (doc.Parse(data.body.c_str()).HasParseError()) {
        // 如果解析失败，记录错误日志，然后重试
        if (!isQuiet()) {
            LOG_ERR("[%s] JSON decode failed: \"%s\"",  pool().daemon().url().data(), rapidjson::GetParseError_En(doc.GetParseError()));
        }
        return retry();
    }

    // 获取JSON中的id字段
    const int64_t id = Json::getInt64(doc, "id", -1);
}
    # 如果 id 大于 0 并且 m_sequence 减去 id 不等于 1，则返回
    if (id > 0 && m_sequence - id != 1) {
        return;
    }

    # 如果解析响应失败，则重试
    if (!parseResponse(id, doc["result"], Json::getObject(doc, "error"))) {
        retry();
    }
# 闭合前面的函数定义
```