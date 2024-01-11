# `xmrig\src\base\net\stratum\EthStratumClient.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cinttypes>
#include <iomanip>
#include <sstream>
#include <stdexcept>


#include "base/net/stratum/EthStratumClient.h"
#include "3rdparty/libethash/endian.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/error/en.h"
#include "3rdparty/rapidjson/stringbuffer.h"
#include "3rdparty/rapidjson/writer.h"
#include "base/io/json/Json.h"
#include "base/io/json/JsonRequest.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IClientListener.h"
#include "net/JobResult.h"

#ifdef XMRIG_ALGO_GHOSTRIDER
#include <cmath>

extern "C" {
#include "crypto/ghostrider/sph_sha2.h"
}

#include "base/tools/Cvt.h"
#endif

// 创建EthStratumClient类的构造函数，初始化id、agent和listener
xmrig::EthStratumClient::EthStratumClient(int id, const char *agent, IClientListener *listener) :
    Client(id, agent, listener)
{
}

// 提交作业结果的函数
int64_t xmrig::EthStratumClient::submit(const JobResult& result)
{
#   ifndef XMRIG_PROXY_PROJECT
    // 如果当前状态不是已连接状态，或者未经授权，则返回-1
    if ((m_state != ConnectedState) || !m_authorized) {
        return -1;
    }
#   endif

    // 如果结果的难度为0，则记录错误信息，关闭连接，并返回-1
    if (result.diff == 0) {
        LOG_ERR("%s " RED("result.diff is 0"), tag());
        close();

        return -1;
    }

    // 使用rapidjson命名空间
    using namespace rapidjson;
    // 创建一个 JSON 文档对象
    Document doc(kObjectType);
    // 获取 JSON 文档对象的分配器
    auto& allocator = doc.GetAllocator();

    // 创建一个 JSON 数组对象
    Value params(kArrayType);
    // 将用户对象转换为 JSON 格式并添加到参数数组中
    params.PushBack(m_user.toJSON(), allocator);
    // 将工作 ID 对象转换为 JSON 格式并添加到参数数组中
    params.PushBack(result.jobId.toJSON(), allocator);
# 如果定义了 XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
if (m_pool.algorithm().id() == Algorithm::GHOSTRIDER_RTM) {
    # 将固定字符串和 m_extraNonce2Size * 2 长度的 0 添加到参数列表中
    params.PushBack(Value("00000000000000000000000000000000", static_cast<uint32_t>(m_extraNonce2Size * 2)), allocator);
    # 将 m_ntime 的数据添加到参数列表中
    params.PushBack(Value(m_ntime.data(), allocator), allocator);

    # 创建一个 stringstream 对象 s，将 result.nonce 转换为 8 位的十六进制数，并添加到参数列表中
    std::stringstream s;
    s << std::hex << std::setw(8) << std::setfill('0') << result.nonce;
    params.PushBack(Value(s.str().c_str(), allocator), allocator);
}
# 如果没有定义 XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
else {
    # 创建一个 stringstream 对象 s，将 result.nonce 转换为 16 位的十六进制数，并添加到参数列表中
    std::stringstream s;
    s << "0x" << std::hex << std::setw(16) << std::setfill('0') << result.nonce;
    params.PushBack(Value(s.str().c_str(), allocator), allocator);

    # 清空 stringstream 对象 s，并将 result.headerHash() 转换为 32 位的十六进制数，并添加到参数列表中
    s.str(std::string());
    s << "0x";
    for (size_t i = 0; i < 32; ++i) {
        const uint32_t k = result.headerHash()[i];
        s << std::hex << std::setw(2) << std::setfill('0') << k;
    }
    params.PushBack(Value(s.str().c_str(), allocator), allocator);

    # 清空 stringstream 对象 s，并将 result.mixHash() 转换为 32 位的十六进制数，并添加到参数列表中
    s.str(std::string());
    s << "0x";
    for (size_t i = 0; i < 32; ++i) {
        const uint32_t k = result.mixHash()[i];
        s << std::hex << std::setw(2) << std::setfill('0') << k;
    }
    params.PushBack(Value(s.str().c_str(), allocator), allocator);
}

# 创建一个 JSON 请求对象，包含指定的参数列表
JsonRequest::create(doc, m_sequence, "mining.submit", params);

# 定义变量 actual_diff
uint64_t actual_diff;

# 如果定义了 XMRIG_ALGO_GHOSTRIDER，并且 result.algorithm 为 Algorithm::GHOSTRIDER_RTM，则执行以下代码块
if (result.algorithm == Algorithm::GHOSTRIDER_RTM) {
    # 将 result.result() 的第 4 个元素解释为 uint64_t 类型，并赋值给 actual_diff
    actual_diff = reinterpret_cast<const uint64_t*>(result.result())[3];
}
# 如果没有定义 XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
else {
    # 将 result.result() 转换为 uint64_t 类型，并赋值给 actual_diff
    actual_diff = ethash_swap_u64(*((uint64_t*)result.result()));
}

# 如果 actual_diff 不为 0，则将 (uint64_t(-1) / actual_diff) 赋值给 actual_diff，否则将 0 赋值给 actual_diff
actual_diff = actual_diff ? (uint64_t(-1) / actual_diff) : 0;

# 如果定义了 XMRIG_PROXY_PROJECT，则将 SubmitResult 对象添加到 m_results 中，否则将另一个 SubmitResult 对象添加到 m_results 中
# SubmitResult(m_sequence, result.diff, actual_diff, result.id, 0) 或 SubmitResult(m_sequence, result.diff, actual_diff, 0, result.backend)
# 具体参数根据是否定义了 XMRIG_PROXY_PROJECT 而定
# 最终将结果发送出去
# 返回结果
// 登录函数，清空结果，订阅和授权
void xmrig::EthStratumClient::login()
{
    m_results.clear(); // 清空结果

    subscribe(); // 订阅
    authorize(); // 授权
}


// 关闭函数，取消授权，调用父类的关闭函数
void xmrig::EthStratumClient::onClose()
{
    m_authorized = false; // 取消授权
    Client::onClose(); // 调用父类的关闭函数
}


// 处理响应函数，根据响应的id查找回调函数，处理响应结果
bool xmrig::EthStratumClient::handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error)
{
    auto it = m_callbacks.find(id); // 查找id对应的回调函数
    if (it != m_callbacks.end()) { // 如果找到了对应的回调函数
        const uint64_t elapsed = Chrono::steadyMSecs() - it->second.ts; // 计算时间间隔

        if (error.IsArray() || error.IsObject() || error.IsString()) { // 如果错误是数组、对象或字符串
            it->second.callback(error, false, elapsed); // 调用回调函数处理错误
        }
        else {
            it->second.callback(result, true, elapsed); // 调用回调函数处理结果
        }

        m_callbacks.erase(it); // 删除已处理的回调函数

        return true; // 返回处理成功
    }

    return handleSubmitResponse(id, errorMessage(error)); // 处理提交的响应
}


// 解析通知函数，根据方法名和参数解析通知内容
void xmrig::EthStratumClient::parseNotification(const char *method, const rapidjson::Value &params, const rapidjson::Value &)
{
    if (strcmp(method, "mining.set_target") == 0) { // 如果方法名是 "mining.set_target"
        return; // 返回
    }

    if (strcmp(method, "mining.set_extranonce") == 0) { // 如果方法名是 "mining.set_extranonce"
        if (!params.IsArray()) { // 如果参数不是数组
            LOG_ERR("%s " RED("invalid mining.set_extranonce notification: params is not an array"), tag()); // 记录错误日志
            return; // 返回
        }

        auto arr = params.GetArray(); // 获取参数数组

        if (arr.Empty()) { // 如果数组为空
            LOG_ERR("%s " RED("invalid mining.set_extranonce notification: params array is empty"), tag()); // 记录错误日志
            return; // 返回
        }

        setExtraNonce(arr[0]); // 设置额外的随机数
    }

#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果收到的通知是 "mining.set_difficulty"
    if (strcmp(method, "mining.set_difficulty") == 0) {
        # 如果参数不是数组，记录错误并返回
        if (!params.IsArray()) {
            LOG_ERR("%s " RED("invalid mining.set_difficulty notification: params is not an array"), tag());
            return;
        }

        # 如果矿池算法不是GHOSTRIDER_RTM，直接返回
        if (m_pool.algorithm().id() != Algorithm::GHOSTRIDER_RTM) {
            return;
        }

        # 获取参数数组
        auto arr = params.GetArray();

        # 如果参数数组为空，记录错误并返回
        if (arr.Empty()) {
            LOG_ERR("%s " RED("invalid mining.set_difficulty notification: params array is empty"), tag());
            return;
        }

        # 如果参数不是双精度浮点数或无符号64位整数，记录错误并返回
        if (!arr[0].IsDouble() && !arr[0].IsUint64()) {
            LOG_ERR("%s " RED("invalid mining.set_difficulty notification: difficulty is not a number"), tag());
            return;
        }

        # 获取难度值并转换为64位无符号整数
        const double diff = arr[0].IsDouble() ? arr[0].GetDouble() : arr[0].GetUint64();
        m_nextDifficulty = static_cast<uint64_t>(ceil(diff * 65536.0));
    }
# 如果条件满足，则执行以下代码块
if (strcmp(method, "mining.notify") == 0) {
    # 如果参数不是数组，则记录错误信息并返回
    if (!params.IsArray()) {
        LOG_ERR("%s " RED("invalid mining.notify notification: params is not an array"), tag());
        return;
    }

    # 获取参数数组
    auto arr = params.GetArray();

    # 获取矿池的算法，如果不存在则使用默认算法
    auto algo = m_pool.algorithm();
    if (!algo.isValid()) {
        algo = m_pool.coin().algorithm();
    }

    # 根据算法类型确定最小数组大小
    const size_t min_arr_size = (algo.id() == Algorithm::GHOSTRIDER_RTM) ? 8 : 6;

    # 如果参数数组大小小于最小数组大小，则记录错误信息并返回
    if (arr.Size() < min_arr_size) {
        LOG_ERR("%s " RED("invalid mining.notify notification: params array has wrong size"), tag());
        return;
    }

    # 如果第一个元素不是字符串，则记录错误信息并返回
    if (!arr[0].IsString()) {
        LOG_ERR("%s " RED("invalid mining.notify notification: invalid job id"), tag());
        return;
    }

    # 创建 Job 对象，并设置其 ID
    Job job;
    job.setId(arr[0].GetString());

    # 设置 Job 对象的算法和额外的 nonce
    job.setAlgorithm(algo);
    job.setExtraNonce(m_extraNonce.second);

    # 创建字符串流对象
    std::stringstream s;
// 结束if语句
{
    // 计算并添加头部哈希值（32字节）
    s << arr[1].GetString();

    // 生成并添加8字节的nonce模板
    for (uint64_t i = 0, k = m_extraNonce.first; i < sizeof(m_extraNonce.first); ++i, k >>= 8) {
        s << std::hex << std::setw(2) << std::setfill('0') << (k & 0xFF);
    }

    // 将stringstream转换为字符串
    std::string blob = s.str();

    // 在字符串末尾添加0，使其长度达到76*2字节
    blob.resize(76 * 2, '0');
    job.setBlob(blob.c_str());

    // 从JSON中获取目标值字符串，并将其长度调整为16字节
    std::string target_str = arr[3].GetString();
    target_str.resize(16, '0');
    // 将目标值字符串转换为uint64_t类型
    const uint64_t target = strtoull(target_str.c_str(), nullptr, 16);
    // 将目标值设置到job对象中
    job.setDiff(Job::toDiff(target));

    // 从JSON中获取高度值，并设置到job对象中
    job.setHeight(arr[5].GetUint64());
}

// 调用监听器的onVerifyAlgorithm方法，验证算法是否兼容
bool ok = true;
m_listener->onVerifyAlgorithm(this, algo, &ok);

// 如果算法不兼容，则输出错误信息并关闭连接
if (!ok) {
    if (!isQuiet()) {
        LOG_ERR("[%s] incompatible/disabled algorithm \"%s\" detected, reconnect", url(), algo.name());
    }
    close();
    return;
}

// 如果收到的job对象与当前的job对象不同，则更新当前的job对象，并触发相应事件
if (m_job != job) {
    m_job = std::move(job);

    // 对于nanopool.org的问题，如果未授权，则设置为已授权，并触发登录成功事件
    if (!m_authorized) {
        m_authorized = true;
        m_listener->onLoginSuccess(this);
    }

    // 触发job接收事件
    m_listener->onJobReceived(this, m_job, params);
}
// 如果收到的job对象与当前的job对象相同，则输出警告信息并关闭连接
else {
    if (!isQuiet()) {
        LOG_WARN("%s " YELLOW("duplicate job received, reconnect"), tag());
    }
    close();
}
}

// 设置额外的nonce值
void xmrig::EthStratumClient::setExtraNonce(const rapidjson::Value &nonce)
{
    // 如果nonce不是字符串类型，则抛出运行时错误
    if (!nonce.IsString()) {
        throw std::runtime_error("invalid mining.subscribe response: extra nonce is not a string");
    }

    // 获取nonce的字符串和长度
    const char *s = nonce.GetString();
    size_t len    = nonce.GetStringLength();

    // 跳过"0x"
    # 如果字符串长度大于等于2，并且以'0x'开头，则去掉开头的'0x'
    if ((len >= 2) && (s[0] == '0') && (s[1] == 'x')) {
        s += 2;
        len -= 2;
    }

    # 如果字符串长度为奇数，则抛出异常
    if (len & 1) {
        throw std::runtime_error("invalid mining.subscribe response: extra nonce has an odd number of hex chars");
    }

    # 如果字符串长度大于8，则抛出异常
    if (len > 8) {
        throw std::runtime_error("Invalid mining.subscribe response: extra nonce is too long");
    }

    # 将字符串转换为std::string类型，并将其长度调整为16，不足部分用'0'填充
    std::string extra_nonce_str(s);
    extra_nonce_str.resize(16, '0');

    # 记录调试信息，输出URL和字符串s
    LOG_DEBUG("[%s] extra nonce set to %s", url(), s);

    # 将字符串s转换为无符号长整型，并存储在m_extraNonce中
    m_extraNonce = { std::stoull(extra_nonce_str, nullptr, 16), s };
// 返回错误消息，优先返回数组第二个元素，然后返回字符串，最后返回对象中的 "message" 字段
const char *xmrig::EthStratumClient::errorMessage(const rapidjson::Value &error)
{
    // 如果错误是数组并且数组大小大于1
    if (error.IsArray() && error.GetArray().Size() > 1) {
        // 获取数组第二个元素
        auto &value = error.GetArray()[1];
        // 如果第二个元素是字符串，返回该字符串
        if (value.IsString()) {
            return value.GetString();
        }
    }

    // 如果错误是字符串，返回该字符串
    if (error.IsString()) {
        return error.GetString();
    }

    // 如果错误是对象，返回对象中的 "message" 字段
    if (error.IsObject()) {
        return Json::getString(error, "message");
    }

    // 如果以上情况都不符合，返回空指针
    return nullptr;
}


// 发送授权请求
void xmrig::EthStratumClient::authorize()
{
    using namespace rapidjson;

    // 创建 JSON 文档
    Document doc(kObjectType);
    auto &allocator = doc.GetAllocator();

    // 创建参数数组，并添加用户和密码
    Value params(kArrayType);
    params.PushBack(m_user.toJSON(), allocator);
    params.PushBack(m_password.toJSON(), allocator);

    // 创建授权请求的 JSON
    JsonRequest::create(doc, m_sequence, "mining.authorize", params);

    // 发送请求，并设置回调函数处理响应
    send(doc, [this](const rapidjson::Value& result, bool success, uint64_t elapsed) { onAuthorizeResponse(result, success, elapsed); });
}


// 处理授权响应
void xmrig::EthStratumClient::onAuthorizeResponse(const rapidjson::Value &result, bool success, uint64_t)
{
    try {
        // 如果请求失败，抛出异常
        if (!success) {
            const auto message = errorMessage(result);
            if (message) {
                throw std::runtime_error(message);
            }

            throw std::runtime_error("mining.authorize call failed");
        }

        // 如果响应不是布尔值，抛出异常
        if (!result.IsBool()) {
            throw std::runtime_error("invalid mining.authorize response: result is not a boolean");
        }

        // 如果响应是 false，抛出异常
        if (!result.GetBool()) {
            throw std::runtime_error("login failed");
        }
    } catch (const std::exception &ex) {
        // 捕获异常，记录错误日志，关闭连接并返回
        LOG_ERR("%s " RED_BOLD("%s"), tag(), ex.what());

        close();
        return;
    }

    // 记录调试日志，授权成功
    LOG_DEBUG("[%s] login succeeded", url());

    // 如果未授权，设置为已授权，并调用登录成功的回调函数
    if (!m_authorized) {
        m_authorized = true;
        m_listener->onLoginSuccess(this);
    }
}


// 处理订阅响应
void xmrig::EthStratumClient::onSubscribeResponse(const rapidjson::Value &result, bool success, uint64_t)
{
    // 如果请求失败，直接返回
    if (!success) {
        return;
    }
}
    }
    # 尝试捕获异常
    try {
        # 如果结果不是数组，则抛出运行时错误
        if (!result.IsArray()) {
            throw std::runtime_error("invalid mining.subscribe response: result is not an array");
        }
        # 获取结果数组
        auto arr = result.GetArray();
        # 如果数组大小小于等于1，则抛出运行时错误
        if (arr.Size() <= 1) {
            throw std::runtime_error("invalid mining.subscribe response: result array is too short");
        }
        # 设置额外的随机数
        setExtraNonce(arr[1]);
// 如果定义了 XMRIG_ALGO_GHOSTRIDER
#ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果数组大小大于2并且第三个元素是无符号整数，设置m_extraNonce2Size为第三个元素的值
    if ((arr.Size() > 2) && (arr[2].IsUint())) {
        m_extraNonce2Size = arr[2].GetUint();
    }
#endif

// 如果连接的矿池是Nicehash
if (m_pool.isNicehash()) {
    // 使用rapidjson命名空间
    using namespace rapidjson;
    // 创建一个JSON文档对象
    Document doc(kObjectType);
    // 创建一个JSON数组对象
    Value params(kArrayType);
    // 创建一个mining.extranonce.subscribe的JSON请求
    JsonRequest::create(doc, m_sequence, "mining.extranonce.subscribe", params);
    // 发送JSON请求
    send(doc);
}
// 捕获异常并记录错误信息
} catch (const std::exception &ex) {
    // 记录错误信息
    LOG_ERR("%s " RED("%s"), tag(), ex.what());
    // 重置m_extraNonce为默认值
    m_extraNonce = { 0, {} };
}
}

// 订阅函数
void xmrig::EthStratumClient::subscribe()
{
    // 使用rapidjson命名空间
    using namespace rapidjson;

    // 创建一个JSON文档对象
    Document doc(kObjectType);
    // 获取文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个JSON数组对象
    Value params(kArrayType);
    // 将代理信息添加到参数数组中
    params.PushBack(StringRef(agent()), allocator);

    // 创建一个mining.subscribe的JSON请求
    JsonRequest::create(doc, m_sequence, "mining.subscribe", params);

    // 发送JSON请求，并在回调函数中处理响应
    send(doc, [this](const rapidjson::Value& result, bool success, uint64_t elapsed) { onSubscribeResponse(result, success, elapsed); });
}
```