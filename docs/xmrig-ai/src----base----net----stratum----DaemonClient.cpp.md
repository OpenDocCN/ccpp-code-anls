# `xmrig\src\base\net\stratum\DaemonClient.cpp`

```
/*
 * XMRig
 * 版权所有（c）2010 Jeff Garzik <jgarzik@pobox.com>
 * 版权所有（c）2012-2014 pooler <pooler@litecoinpool.org>
 * 版权所有（c）2014 Lucas Jones <https://github.com/lucasjones>
 * 版权所有（c）2014-2016 Wolf9466 <https://github.com/OhGodAPet>
 * 版权所有（c）2016 Jay D Dee <jayddee246@gmail.com>
 * 版权所有（c）2017-2018 XMR-Stak <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有（c）2019 Howard Chu <https://github.com/hyc>
 * 版权所有（c）2018-2023 SChernykh <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <uv.h>

#include "base/net/stratum/DaemonClient.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/error/en.h"
#include "base/io/json/Json.h"
#include "base/io/json/JsonRequest.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IClientListener.h"
#include "base/kernel/Platform.h"
#include "base/net/dns/Dns.h"
#include "base/net/dns/DnsRecords.h"
#include "base/net/http/Fetch.h"
#include "base/net/http/HttpData.h"
#include "base/net/http/HttpListener.h"
#include "base/net/stratum/SubmitResult.h"
#include "base/net/tools/NetBuffer.h"
#include "base/tools/bswap_64.h"
#include "base/tools/cryptonote/Signatures.h"
*/
#include "base/tools/Cvt.h"
#include "base/tools/Timer.h"
#include "net/JobResult.h"

#ifdef XMRIG_FEATURE_TLS
#include <openssl/ssl.h>
#endif

#include <algorithm>
#include <cassert>
#include <random>

// 命名空间定义
namespace xmrig {

// DaemonClient 类的静态成员变量
Storage<DaemonClient> DaemonClient::m_storage;

// 定义常量字符串
static const char* kBlocktemplateBlob       = "blocktemplate_blob";
static const char* kBlockhashingBlob        = "blockhashing_blob";
static const char *kGetHeight               = "/getheight";
static const char *kGetInfo                 = "/getinfo";
static const char *kHash                    = "hash";
static const char *kHeight                  = "height";
static const char *kJsonRPC                 = "/json_rpc";

// 定义常量大小
static constexpr size_t kBlobReserveSize    = 8;

// 定义常量字符数组
static const char kZMQGreeting[64] = { static_cast<char>(-1), 0, 0, 0, 0, 0, 0, 0, 0, 127, 3, 0, 'N', 'U', 'L', 'L' };
static constexpr size_t kZMQGreetingSize1 = 11;

// 定义常量字符数组
static const char kZMQHandshake[] = "\4\x19\5READY\xbSocket-Type\0\0\0\3SUB";
static const char kZMQSubscribe[] = "\0\x18\1json-minimal-chain_main";

} // namespace xmrig

// DaemonClient 类的构造函数
xmrig::DaemonClient::DaemonClient(int id, IClientListener *listener) :
    BaseClient(id, listener)
{
    // 初始化成员变量
    m_httpListener  = std::make_shared<HttpListener>(this);
    m_timer         = new Timer(this);
    m_key           = m_storage.add(this);
}

// DaemonClient 类的析构函数
xmrig::DaemonClient::~DaemonClient()
{
    // 释放内存
    delete m_timer;
    delete m_ZMQSocket;
}

// 延迟删除函数
void xmrig::DaemonClient::deleteLater()
{
    // 根据条件关闭 ZMQ 连接或者删除对象
    if (m_pool.zmq_port() >= 0) {
        ZMQClose(true);
    }
    else {
        delete this;
    }
}

// 断开连接函数
bool xmrig::DaemonClient::disconnect()
{
    // 如果当前状态不是未连接状态，则设置为未连接状态
    if (m_state != UnconnectedState) {
        setState(UnconnectedState);
    }
    return true;
}

// 判断是否使用 TLS 函数
bool xmrig::DaemonClient::isTLS() const
{
#   ifdef XMRIG_FEATURE_TLS
    return m_pool.isTLS();
#   else
    return false;
#   endif
}

// 提交作业结果函数
int64_t xmrig::DaemonClient::submit(const JobResult &result)
{
    // 如果作业 ID 不匹配，则返回 -1
    if (result.jobId != m_currentJobId) {
        return -1;
    }
}
    // 将 m_blocktemplateStr 转换为字符指针并赋值给 data
    char *data = m_blocktemplateStr.data();

    // 计算签名偏移量，即 m_job.nonceOffset() + m_job.nonceSize()
    const size_t sig_offset = m_job.nonceOffset() + m_job.nonceSize();
# ifdef XMRIG_PROXY_PROJECT
#   如果定义了 XMRIG_PROXY_PROJECT，则执行以下代码块

    // 将 result.nonce 的内容复制到 data 中的 m_job.nonceOffset() * 2 处，长度为 8
    memcpy(data + m_job.nonceOffset() * 2, result.nonce, 8);

    // 如果区块模板包含矿工签名并且 result.sig 存在
    if (m_blocktemplate.hasMinerSignature() && result.sig) {
        // 将 result.sig 的内容复制到 data 中的 sig_offset * 2 处，长度为 64 * 2
        memcpy(data + sig_offset * 2, result.sig, 64 * 2);
        // 将 result.sig_data 的前 32 * 2 个字节复制到 data 中的 TX_PUBKEY_OFFSET 处，长度为 32 * 2
        memcpy(data + m_blocktemplate.offset(BlockTemplate::TX_PUBKEY_OFFSET) * 2, result.sig_data, 32 * 2);
        // 将 result.sig_data 的后 32 * 2 个字节复制到 data 中的 EPH_PUBLIC_KEY_OFFSET 处，长度为 32 * 2
        memcpy(data + m_blocktemplate.offset(BlockTemplate::EPH_PUBLIC_KEY_OFFSET) * 2, result.sig_data + 32 * 2, 32 * 2);

        // 处理 txout_to_tagged_key 输出的 view tag
        if (m_blocktemplate.outputType() == 3) {
            // 将 result.view_tag 转换为十六进制，并复制到 data 中的 EPH_PUBLIC_KEY_OFFSET 处的后 32 * 2 个字节，长度为 2
            Cvt::toHex(data + m_blocktemplate.offset(BlockTemplate::EPH_PUBLIC_KEY_OFFSET) * 2 + 32 * 2, 2, &result.view_tag, 1);
        }
    }

    // 如果 result.extra_nonce 大于等于 0
    if (result.extra_nonce >= 0) {
        // 将 result.extra_nonce 转换为十六进制，并复制到 data 中的 TX_EXTRA_NONCE_OFFSET 处，长度为 8
        Cvt::toHex(data + m_blocktemplate.offset(BlockTemplate::TX_EXTRA_NONCE_OFFSET) * 2, 8, reinterpret_cast<const uint8_t*>(&result.extra_nonce), 4);
    }

# else
#   如果未定义 XMRIG_PROXY_PROJECT，则执行以下代码块

    // 将 result.nonce 的内容转换为十六进制，并复制到 data 中的 m_job.nonceOffset() * 2 处，长度为 8
    Cvt::toHex(data + m_job.nonceOffset() * 2, 8, reinterpret_cast<const uint8_t*>(&result.nonce), 4);

    // 如果区块模板包含矿工签名
    if (m_blocktemplate.hasMinerSignature()) {
        // 将 result.minerSignature() 的内容转换为十六进制，并复制到 data 中的 sig_offset * 2 处，长度为 128
        Cvt::toHex(data + sig_offset * 2, 128, result.minerSignature(), 64);
    }

# endif
# 结束 ifdef XMRIG_PROXY_PROJECT

    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 创建一个空的 JSON 文档
    Document doc(kObjectType);

    // 创建一个空的 JSON 数组
    Value params(kArrayType);
    // 将 m_blocktemplateStr 转换为 JSON 格式，并添加到 params 数组中
    params.PushBack(m_blocktemplateStr.toJSON(), doc.GetAllocator());

    // 创建一个 JSON 请求
    JsonRequest::create(doc, m_sequence, "submitblock", params);

# ifdef XMRIG_PROXY_PROJECT
    // 如果定义了 XMRIG_PROXY_PROJECT，则将结果存储到 m_results 中
    m_results[m_sequence] = SubmitResult(m_sequence, result.diff, result.actualDiff(), result.id, 0);
# else
    // 如果未定义 XMRIG_PROXY_PROJECT，则将结果存储到 m_results 中
    m_results[m_sequence] = SubmitResult(m_sequence, result.diff, result.actualDiff(), 0, result.backend);
# endif

    // 创建一个存储请求头信息的 map
    std::map<std::string, std::string> headers;
    // 将 X-Hash-Difficulty 和 result.actualDiff() 转换为字符串，并添加到 headers 中
    headers.insert({"X-Hash-Difficulty", std::to_string(result.actualDiff())});

    // 发送 RPC 请求，并返回结果
    return rpcSend(doc, headers);
}

// 连接到 Daemon
void xmrig::DaemonClient::connect()
{
    # 定义一个 lambda 函数，用于处理连接错误，接受一个消息参数
    auto connectError = [this](const char *message) {
        # 如果不是静默模式，记录连接错误消息
        if (!isQuiet()) {
            LOG_ERR("%s " RED("connect error: ") RED_BOLD("\"%s\""), tag(), message);
        }
        # 重试连接
        retry();
    };

    # 设置状态为连接中
    setState(ConnectingState);

    # 如果货币无效且矿池算法无效，则返回连接错误
    if (!m_coin.isValid() && !m_pool.algorithm().isValid()) {
        return connectError("Invalid algorithm.");
    }

    # 如果矿池算法无效，则设置矿池算法为货币算法
    if (!m_pool.algorithm().isValid()) {
        m_pool.setAlgo(m_coin.algorithm());
    }

    # 如果 API 版本为 API_MONERO 且钱包地址无效，则返回连接错误
    if ((m_apiVersion == API_MONERO) && !m_walletAddress.isValid()) {
        return connectError("Invalid wallet address.");
    }

    # 如果矿池的 ZMQ 端口号大于等于 0，则解析矿池的主机名
    if (m_pool.zmq_port() >= 0) {
        m_dns = Dns::resolve(m_pool.host(), this);
    }
    # 否则，获取区块模板
    else {
        getBlockTemplate();
    }
# 结束前一个函数的定义
}

# 连接到指定的矿池
void xmrig::DaemonClient::connect(const Pool &pool)
{
    # 设置连接的矿池
    setPool(pool);
    # 连接到矿池
    connect();
}

# 设置连接的矿池
void xmrig::DaemonClient::setPool(const Pool &pool)
{
    # 调用基类的设置矿池方法
    BaseClient::setPool(pool);

    # 解析钱包地址
    m_walletAddress.decode(m_user);

    # 设置币种
    m_coin = pool.coin().isValid() ?  pool.coin() : m_walletAddress.coin();

    # 如果币种无效且算法为RX_WOW，则设置币种为WOWNERO
    if (!m_coin.isValid() && pool.algorithm() == Algorithm::RX_WOW) {
        m_coin = Coin::WOWNERO;
    }
}

# 处理HTTP数据
void xmrig::DaemonClient::onHttpData(const HttpData &data)
{
    # 如果HTTP状态不是200，则重试连接
    if (data.status != 200) {
        return retry();
    }

    # 获取IP地址
    m_ip = data.ip().c_str();

#   ifdef XMRIG_FEATURE_TLS
    # 获取TLS版本和指纹
    m_tlsVersion     = data.tlsVersion();
    m_tlsFingerprint = data.tlsFingerprint();
#   endif

    # 解析JSON数据
    rapidjson::Document doc;
    if (doc.Parse(data.body.c_str()).HasParseError()) {
        # 如果解析失败，记录错误并重试连接
        if (!isQuiet()) {
            LOG_ERR("%s " RED("JSON decode failed: ") RED_BOLD("\"%s\""), tag(), rapidjson::GetParseError_En(doc.GetParseError()));
        }

        return retry();
    }
}
    # 如果请求方法为 HTTP_GET
    if (data.method == HTTP_GET) {
        # 如果请求的 URL 为获取高度的 URL
        if (data.url == kGetHeight) {
            # 如果文档中没有包含 kHash 字段
            if (!doc.HasMember(kHash)) {
                # 设置 API 版本为默认值
                m_apiVersion = API_CRYPTONOTE_DEFAULT;
                # 发送获取信息的请求
                return send(kGetInfo);
            }

            # 从文档中获取高度并转换为 uint64_t 类型
            const uint64_t height = Json::getUint64(doc, kHeight);
            # 从文档中获取哈希值并转换为 String 类型
            const String hash = Json::getString(doc, kHash);

            # 如果高度和哈希值已经过时
            if (isOutdated(height, hash)) {
                # 多个 /getheight 响应可能同时到达，导致多次调用 getBlockTemplate()
                if ((height != m_blocktemplateRequestHeight) || (hash != m_blocktemplateRequestHash)) {
                    # 更新 m_blocktemplateRequestHeight 和 m_blocktemplateRequestHash
                    m_blocktemplateRequestHeight = height;
                    m_blocktemplateRequestHash = hash;
                    # 调用 getBlockTemplate() 方法
                    getBlockTemplate();
                }
            }
        }
        # 如果请求的 URL 为获取信息的 URL
        else if (data.url == kGetInfo) {
            # 从文档中获取高度并转换为 uint64_t 类型
            const uint64_t height = Json::getUint64(doc, kHeight);
            # 从文档中获取顶部块的哈希值并转换为 String 类型
            const String hash = Json::getString(doc, "top_block_hash");

            # 如果高度和哈希值已经过时
            if (isOutdated(height, hash)) {
                # 多个 /getinfo 响应可能同时到达，导致多次调用 getBlockTemplate()
                if ((height != m_blocktemplateRequestHeight) || (hash != m_blocktemplateRequestHash)) {
                    # 更新 m_blocktemplateRequestHeight 和 m_blocktemplateRequestHash
                    m_blocktemplateRequestHeight = height;
                    m_blocktemplateRequestHash = hash;
                    # 调用 getBlockTemplate() 方法
                    getBlockTemplate();
                }
            }
        }

        # 结束当前函数
        return;
    }

    # 如果解析响应失败
    if (!parseResponse(Json::getInt64(doc, "id", -1), Json::getObject(doc, "result"), Json::getObject(doc, "error"))) {
        # 重试请求
        retry();
    }
// 定义 DaemonClient 类的 onTimer 方法
void xmrig::DaemonClient::onTimer(const Timer *)
{
    // 如果连接的池的 ZeroMQ 端口大于等于 0
    if (m_pool.zmq_port() >= 0) {
        // 重置 m_prevHash 和 m_blocktemplateRequestHash
        m_prevHash = nullptr;
        m_blocktemplateRequestHash = nullptr;
        // 发送 kGetHeight 指令
        send(kGetHeight);
        // 返回
        return;
    }

    // 如果当前时间超过上一个作业的稳定时间加上池的作业超时时间
    if (Chrono::steadyMSecs() >= m_jobSteadyMs + m_pool.jobTimeout()) {
        // 重置 m_prevHash 和 m_blocktemplateRequestHash
        m_prevHash = nullptr;
        m_blocktemplateRequestHash = nullptr;
    }

    // 如果当前状态为连接状态
    if (m_state == ConnectingState) {
        // 连接
        connect();
    }
    // 如果当前状态为已连接状态
    else if (m_state == ConnectedState) {
        // 发送 kGetHeight 或 kGetInfo 指令
        send((m_apiVersion == API_MONERO) ? kGetHeight : kGetInfo);
    }
}

// 定义 DaemonClient 类的 onResolved 方法
void xmrig::DaemonClient::onResolved(const DnsRecords &records, int status, const char* error)
{
    // 重置 m_dns
    m_dns.reset();

    // 如果状态小于 0 并且记录为空
    if (status < 0 && records.isEmpty()) {
        // 如果不是安静模式，输出 DNS 错误信息
        if (!isQuiet()) {
            LOG_ERR("%s " RED("DNS error: ") RED_BOLD("\"%s\""), tag(), error);
        }
        // 重试连接
        retry();
        // 返回
        return;
    }

    // 获取记录
    const auto &record = records.get();
    // 获取 IP 地址
    m_ip = record.ip();

    // 创建 uv_connect_t 对象
    auto req = new uv_connect_t;
    req->data = m_storage.ptr(m_key);

    // 创建 uv_tcp_t 对象
    uv_tcp_t* s = new uv_tcp_t;
    s->data = m_storage.ptr(m_key);

    // 初始化 TCP 对象
    uv_tcp_init(uv_default_loop(), s);
    // 设置 TCP 为无延迟模式
    uv_tcp_nodelay(s, 1);

    // 如果平台支持保持连接
    if (Platform::hasKeepalive()) {
        // 设置 TCP 保持连接
        uv_tcp_keepalive(s, 1, 60);
    }

    // 如果池的 ZeroMQ 端口大于 0
    if (m_pool.zmq_port() > 0) {
        // 删除 m_ZMQSocket
        delete m_ZMQSocket;
        // 设置 m_ZMQSocket 为 s
        m_ZMQSocket = s;
        // 连接到 ZeroMQ 端口
        uv_tcp_connect(req, s, record.addr(m_pool.zmq_port()), onZMQConnect);
    }
}

// 定义 DaemonClient 类的 isOutdated 方法
bool xmrig::DaemonClient::isOutdated(uint64_t height, const char *hash) const
{
    // 如果当前作业的高度不等于给定高度，或者上一个哈希不等于给定哈希，或者当前时间超过上一个作业的稳定时间加上池的作业超时时间
    return m_job.height() != height || m_prevHash != hash || Chrono::steadyMSecs() >= m_jobSteadyMs + m_pool.jobTimeout();
}

// 定义 DaemonClient 类的 parseJob 方法
bool xmrig::DaemonClient::parseJob(const rapidjson::Value &params, int *code)
{
    // 定义 jobError 函数
    auto jobError = [this, code](const char *message) {
        // 如果不是安静模式，输出作业错误信息
        if (!isQuiet()) {
            LOG_ERR("%s " RED("job error: ") RED_BOLD("\"%s\""), tag(), message);
        }
        // 设置 code 为 1
        *code = 1;
        // 返回 false
        return false;
    };

    // 创建 Job 对象
    Job job(false, m_pool.algorithm(), String());
}
    # 从参数中获取区块模板的字符串
    String blocktemplate = Json::getString(params, kBlocktemplateBlob);

    # 如果获取到的区块模板字符串为空
    if (blocktemplate.isNull()) {
        # 返回一个错误信息
        return jobError("Empty block template received from daemon."); // FIXME
    }

    # 如果无法解析获取到的区块模板字符串
    if (!m_blocktemplate.parse(blocktemplate, m_coin)) {
        # 返回一个错误信息
        return jobError("Invalid block template received from daemon.");
    }
# 如果定义了XMRIG_PROXY_PROJECT，则执行以下代码块
    const size_t k = m_blocktemplate.offset(BlockTemplate::MINER_TX_PREFIX_OFFSET);
    # 设置工作的矿工交易
    job.setMinerTx(
        m_blocktemplate.blob() + k,  # 设置矿工交易的起始位置
        m_blocktemplate.blob() + m_blocktemplate.offset(BlockTemplate::MINER_TX_PREFIX_END_OFFSET),  # 设置矿工交易的结束位置
        m_blocktemplate.offset(BlockTemplate::EPH_PUBLIC_KEY_OFFSET) - k,  # 设置临时公钥的偏移量
        m_blocktemplate.offset(BlockTemplate::TX_PUBKEY_OFFSET) - k,  # 设置交易公钥的偏移量
        m_blocktemplate.offset(BlockTemplate::TX_EXTRA_NONCE_OFFSET) - k,  # 设置交易额外随机数的偏移量
        m_blocktemplate.txExtraNonce().size(),  # 设置交易额外随机数的大小
        m_blocktemplate.minerTxMerkleTreeBranch(),  # 设置矿工交易的Merkle树分支
        m_blocktemplate.outputType() == 3  # 设置输出类型是否为3
    );
#   endif

    m_blockhashingblob = Json::getString(params, kBlockhashingBlob);  # 从参数中获取块哈希数据

    if (m_blocktemplate.hasMinerSignature()) {  # 如果块模板有矿工签名
        if (m_pool.spendSecretKey().isEmpty()) {  # 如果矿池的花费秘钥为空
            return jobError("Secret spend key is not set.");  # 返回错误信息
        }

        if (m_pool.spendSecretKey().size() != 64) {  # 如果矿池的花费秘钥长度不为64
            return jobError("Secret spend key has invalid length. It must be 64 hex characters.");  # 返回错误信息
        }

        uint8_t secret_spendkey[32];  # 定义32字节的秘密花费秘钥数组
        if (!Cvt::fromHex(secret_spendkey, 32, m_pool.spendSecretKey(), 64)) {  # 如果无法将16进制数据转换为字节流
            return jobError("Secret spend key is not a valid hex data.");  # 返回错误信息
        }

        uint8_t public_spendkey[32];  # 定义32字节的公共花费秘钥数组
        if (!secret_key_to_public_key(secret_spendkey, public_spendkey)) {  # 如果无法从秘密花费秘钥生成公共花费秘钥
            return jobError("Secret spend key is invalid.");  # 返回错误信息
        }

#       ifdef XMRIG_PROXY_PROJECT  # 如果定义了XMRIG_PROXY_PROJECT
        job.setSpendSecretKey(secret_spendkey);  # 设置矿工的花费秘钥
        # 如果不是新的区块模板，则执行以下操作
        uint8_t secret_viewkey[32];
        # 根据秘密花费密钥派生出视图密钥
        derive_view_secret_key(secret_spendkey, secret_viewkey);

        uint8_t public_viewkey[32];
        # 如果无法将视图密钥转换为公钥，则返回错误
        if (!secret_key_to_public_key(secret_viewkey, public_viewkey)) {
            return jobError("Secret view key is invalid.");
        }

        uint8_t derivation[32];
        # 如果无法为矿工签名生成密钥派生，则返回错误
        if (!generate_key_derivation(m_blocktemplate.blob(BlockTemplate::TX_PUBKEY_OFFSET), secret_viewkey, derivation, nullptr)) {
            return jobError("Failed to generate key derivation for miner signature.");
        }

        # 如果无法解码钱包地址，则返回错误
        if (!m_walletAddress.decode(m_pool.user())) {
            return jobError("Invalid wallet address.");
        }

        # 如果钱包地址的花费密钥与公共花费密钥不匹配，则返回错误
        if (memcmp(m_walletAddress.spendKey(), public_spendkey, sizeof(public_spendkey)) != 0) {
            return jobError("Wallet address and spend key don't match.");
        }

        # 如果钱包地址的视图密钥与公共视图密钥不匹配，则返回错误
        if (memcmp(m_walletAddress.viewKey(), public_viewkey, sizeof(public_viewkey)) != 0) {
            return jobError("Wallet address and view key don't match.");
        }

        uint8_t eph_secret_key[32];
        # 根据密钥派生生成临时密钥
        derive_secret_key(derivation, 0, secret_spendkey, eph_secret_key);

        # 设置临时密钥和公共密钥
        job.setEphemeralKeys(m_blocktemplate.blob(BlockTemplate::EPH_PUBLIC_KEY_OFFSET), eph_secret_key);
#       endif
    }

    # 如果币种有效，则设置算法
    if (m_coin.isValid()) {
        job.setAlgorithm(m_coin.algorithm(m_blocktemplate.majorVersion()));
    }

    # 如果无法设置区块数据，则返回错误
    if (!job.setBlob(m_blockhashingblob)) {
        *code = 3;
        return false;
    }

    # 设置种子哈希、高度和难度
    job.setSeedHash(Json::getString(params, "seed_hash"));
    job.setHeight(Json::getUint64(params, kHeight));
    job.setDiff(Json::getUint64(params, "difficulty"));

    # 生成当前作业的ID
    m_currentJobId = Cvt::toHex(Cvt::randomBytes(4));
    job.setId(m_currentJobId);

    # 移动作业、区块模板和前一个哈希
    m_job              = std::move(job);
    m_blocktemplateStr = std::move(blocktemplate);
    m_prevHash         = Json::getString(params, "prev_hash");
    m_jobSteadyMs      = Chrono::steadyMSecs();
    # 如果当前状态为连接状态
    if (m_state == ConnectingState) {
        # 设置状态为已连接状态
        setState(ConnectedState);
    }
    
    # 通过监听器通知作业已接收，并传递作业、参数
    m_listener->onJobReceived(this, m_job, params);
    # 返回true
    return true;
// 解析响应结果，根据 id、result 和 error 进行处理
bool xmrig::DaemonClient::parseResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error)
{
    // 如果 id 为 -1，则返回 false
    if (id == -1) {
        return false;
    }

    // 如果 error 是一个对象
    if (error.IsObject()) {
        // 获取 error 中的 message 字段
        const char *message = error["message"].GetString();

        // 如果处理响应失败并且不是静默模式，则记录错误日志
        if (!handleSubmitResponse(id, message) && !isQuiet()) {
            LOG_ERR("[%s:%d] error: " RED_BOLD("\"%s\"") RED_S ", code: %d", m_pool.host().data(), m_pool.port(), message, error["code"].GetInt());
        }

        return false;
    }

    // 如果 result 不是一个对象，则返回 false
    if (!result.IsObject()) {
        return false;
    }

    // 如果 result 中包含 top_block_hash 字段
    if (result.HasMember("top_block_hash")) {
        // 如果当前哈希值不等于 top_block_hash 字段的值，则调用 getBlockTemplate() 方法
        if (m_prevHash != Json::getString(result, "top_block_hash")) {
            getBlockTemplate();
        }
        return true;
    }

    // 初始化 code 为 -1
    int code = -1;
    // 如果 result 中包含 kBlocktemplateBlob 字段并且成功解析作业，则返回 true
    if (result.HasMember(kBlocktemplateBlob) && parseJob(result, &code)) {
        return true;
    }

    // 初始化 error_msg 为 nullptr
    const char* error_msg = nullptr;

    // 如果处理响应成功并且 error_msg 不为空，或者 m_pool.zmq_port() 小于 0，则调用 getBlockTemplate() 方法
    if (handleSubmitResponse(id, error_msg)) {
        if (error_msg || (m_pool.zmq_port() < 0)) {
            getBlockTemplate();
        }
        return true;
    }

    // 其他情况返回 false
    return false;
}

// 获取区块模板
int64_t xmrig::DaemonClient::getBlockTemplate()
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 创建一个空的 JSON 文档
    Document doc(kObjectType);
    // 获取文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建 params 对象，并添加 wallet_address 和 extra_nonce 字段
    Value params(kObjectType);
    params.AddMember("wallet_address", m_user.toJSON(), allocator);
    params.AddMember("extra_nonce", Cvt::toHex(Cvt::randomBytes(kBlobReserveSize)).toJSON(doc), allocator);

    // 创建 JSON-RPC 请求
    JsonRequest::create(doc, m_sequence, "getblocktemplate", params);

    // 发送 RPC 请求
    return rpcSend(doc);
}

// 发送 RPC 请求
int64_t xmrig::DaemonClient::rpcSend(const rapidjson::Document &doc, const std::map<std::string, std::string> &headers)
{
    // 创建一个 HTTP 请求
    FetchRequest req(HTTP_POST, m_pool.host(), m_pool.port(), kJsonRPC, doc, m_pool.isTLS(), isQuiet());
    // 添加自定义的请求头
    for (const auto &header : headers) {
        req.headers.insert(header);
    }

    // 发送请求并返回序列号
    fetch(tag(), std::move(req), m_httpListener);

    return m_sequence++;
}

// 重试方法
void xmrig::DaemonClient::retry()
{
    // 空方法，无需注释
}
    # 增加失败计数
    m_failures++;

    # 调用监听器的关闭方法，传入失败次数
    m_listener->onClose(this, static_cast<int>(m_failures));

    # 如果失败次数为-1，则直接返回，不执行后续操作
    if (m_failures == -1) {
        return;
    }

    # 如果当前状态为连接状态，则将状态设置为连接中状态
    if (m_state == ConnectedState) {
        setState(ConnectingState);
    }

    # 如果 ZMQ 连接状态不是未连接且不是正在断开连接
    if ((m_ZMQConnectionState != ZMQ_NOT_CONNECTED) && (m_ZMQConnectionState != ZMQ_DISCONNECTING)) {
        # 如果平台支持保持连接，则设置 ZMQ 套接字的 TCP 保持连接参数
        if (Platform::hasKeepalive()) {
            uv_tcp_keepalive(m_ZMQSocket, 0, 60);
        }
        # 关闭 ZMQ 套接字
        uv_close(reinterpret_cast<uv_handle_t*>(m_ZMQSocket), onZMQClose);
    }

    # 停止定时器
    m_timer->stop();
    # 重新启动定时器，设置重试暂停时间为 m_retryPause
    m_timer->start(m_retryPause, 0);
# 发送请求到指定路径
void xmrig::DaemonClient::send(const char *path)
{
    # 创建一个Fetch请求对象
    FetchRequest req(HTTP_GET, m_pool.host(), m_pool.port(), path, m_pool.isTLS(), isQuiet());
    # 发送请求并设置回调函数
    fetch(tag(), std::move(req), m_httpListener);
}

# 设置客户端状态
void xmrig::DaemonClient::setState(SocketState state)
{
    # 如果状态相同则直接返回
    if (m_state == state) {
        return;
    }

    # 更新状态
    m_state = state;

    # 根据状态进行不同的操作
    switch (state) {
    case ConnectedState:
        {
            # 重置失败次数
            m_failures = 0;
            # 调用登录成功的回调函数
            m_listener->onLoginSuccess(this);

            # 根据是否有ZMQ端口来设置定时器
            if (m_pool.zmq_port() < 0) {
                const uint64_t interval = std::max<uint64_t>(20, m_pool.pollInterval());
                m_timer->start(interval, interval);
            }
            else {
                const uint64_t t = m_pool.jobTimeout();
                m_timer->start(t, t);
            }
        }
        break;

    case UnconnectedState:
        # 设置失败次数为-1
        m_failures = -1;
        # 停止定时器
        m_timer->stop();
        break;

    default:
        break;
    }
}

# ZMQ连接成功的回调函数
void xmrig::DaemonClient::onZMQConnect(uv_connect_t* req, int status)
{
    # 获取客户端对象
    DaemonClient* client = getClient(req->data);
    # 释放请求对象
    delete req;

    # 如果客户端对象不存在则直接返回
    if (!client) {
        return;
    }

    # 如果连接状态小于0，则输出错误信息并重试连接
    if (status < 0) {
        LOG_ERR("%s " RED("ZMQ connect error: ") RED_BOLD("\"%s\""), client->tag(), uv_strerror(status));
        client->retry();
        return;
    }

    # 调用ZMQ连接成功的函数
    client->ZMQConnected();
}

# ZMQ读取数据的回调函数
void xmrig::DaemonClient::onZMQRead(uv_stream_t* stream, ssize_t nread, const uv_buf_t* buf)
{
    # 获取客户端对象
    DaemonClient* client = getClient(stream->data);
    # 如果客户端对象存在则调用ZMQ读取函数
    if (client) {
        client->ZMQRead(nread, buf);
    }

    # 释放缓冲区
    NetBuffer::release(buf);
}

# ZMQ关闭连接的回调函数
void xmrig::DaemonClient::onZMQClose(uv_handle_t* handle)
{
    # 获取客户端对象
    DaemonClient* client = getClient(handle->data);
    # 如果客户端对象存在则执行相应操作
    if (client) {
#       ifdef APP_DEBUG
        LOG_DEBUG(CYAN("tcp-zmq://%s:%u") BLACK_BOLD(" disconnected"), client->m_pool.host().data(), client->m_pool.zmq_port());
#       endif
        client->m_ZMQConnectionState = ZMQ_NOT_CONNECTED;
    }
}

# ZMQ关闭连接的回调函数
void xmrig::DaemonClient::onZMQShutdown(uv_handle_t* handle)
{
    // 获取与给定句柄相关的守护进程客户端
    DaemonClient* client = getClient(handle->data);
    // 如果客户端存在
    if (client) {
#       ifdef APP_DEBUG
        // 如果是调试模式，记录客户端的连接信息
        LOG_DEBUG(CYAN("tcp-zmq://%s:%u") BLACK_BOLD(" shutdown"), client->m_pool.host().data(), client->m_pool.zmq_port());
#       endif
        // 设置客户端的 ZMQ 连接状态为未连接
        client->m_ZMQConnectionState = ZMQ_NOT_CONNECTED;
        // 从存储中移除客户端的键
        m_storage.remove(client->m_key);
    }
}


void xmrig::DaemonClient::ZMQConnected()
{
#   ifdef APP_DEBUG
    // 如果是调试模式，记录客户端的连接信息
    LOG_DEBUG(CYAN("tcp-zmq://%s:%u") BLACK_BOLD(" connected"), m_pool.host().data(), m_pool.zmq_port());
#   endif
    // 设置客户端的 ZMQ 连接状态为 ZMQ_GREETING_1
    m_ZMQConnectionState = ZMQ_GREETING_1;
    // 预留发送缓冲区大小为256
    m_ZMQSendBuf.reserve(256);
    // 预留接收缓冲区大小为256
    m_ZMQRecvBuf.reserve(256);
    // 如果成功写入 ZMQ 问候消息
    if (ZMQWrite(kZMQGreeting, kZMQGreetingSize1)) {
        // 开始从 ZMQ 套接字读取数据
        uv_read_start(reinterpret_cast<uv_stream_t*>(m_ZMQSocket), NetBuffer::onAlloc, onZMQRead);
    }
}


bool xmrig::DaemonClient::ZMQWrite(const char* data, size_t size)
{
    // 将数据复制到发送缓冲区
    m_ZMQSendBuf.assign(data, data + size);
    // 设置缓冲区
    uv_buf_t buf;
    buf.base = m_ZMQSendBuf.data();
    buf.len = static_cast<uint32_t>(m_ZMQSendBuf.size());
    // 尝试写入数据到 ZMQ 套接字
    const int rc = uv_try_write(reinterpret_cast<uv_stream_t*>(m_ZMQSocket), &buf, 1);
    // 如果成功写入所有数据
    if (static_cast<size_t>(rc) == buf.len) {
        return true;
    }
    // 如果写入失败，记录错误信息，关闭 ZMQ 连接并返回 false
    LOG_ERR("%s " RED("ZMQ write failed, rc = %d"), tag(), rc);
    ZMQClose();
    return false;
}


void xmrig::DaemonClient::ZMQRead(ssize_t nread, const uv_buf_t* buf)
{
    // 如果读取失败，记录错误信息，关闭 ZMQ 连接并返回
    if (nread <= 0) {
        LOG_ERR("%s " RED("ZMQ read failed, nread = %" PRId64), tag(), nread);
        ZMQClose();
        return;
    }
    // 将接收到的数据添加到接收缓冲区
    m_ZMQRecvBuf.insert(m_ZMQRecvBuf.end(), buf->base, buf->base + nread);
    // 无限循环，需要注意可能的逻辑错误
    } while (true);
}


void xmrig::DaemonClient::ZMQParse()
{
#   ifdef APP_DEBUG
    // 如果是调试模式，创建消息向量
    std::vector<char> msg;
#   endif
    // 初始化消息大小
    size_t msg_size = 0;
    // 获取接收缓冲区的数据和可用大小
    char *data   = m_ZMQRecvBuf.data();
    size_t avail = m_ZMQRecvBuf.size();
    bool more    = false;
}
    // 如果可用数据小于1，则返回
    do {
        if (avail < 1) {
            return;
        }

        // 检查第一个字节的最低位，判断是否还有更多数据
        more                 = (data[0] & 1) != 0;
        // 检查第一个字节的次低位，判断是否使用长尺寸
        const bool long_size = (data[0] & 2) != 0;
        // 检查第一个字节的次高位，判断是否是命令
        const bool command   = (data[0] & 4) != 0;

        // 移动数据指针和减少可用数据大小
        ++data;
        --avail;

        uint64_t size = 0;
        // 如果使用长尺寸
        if (long_size)
        {
            // 如果可用数据小于uint64_t的大小，则返回
            if (avail < sizeof(uint64_t)) {
                return;
            }
            // 读取uint64_t大小的数据，并转换为主机字节顺序
            size = bswap_64(*((uint64_t*)data));
            data += sizeof(uint64_t);
            avail -= sizeof(uint64_t);
        }
        else
        {
            // 如果可用数据小于uint8_t的大小，则返回
            if (avail < sizeof(uint8_t)) {
                return;
            }
            // 读取uint8_t大小的数据
            size = static_cast<uint8_t>(*data);
            ++data;
            --avail;
        }

        // 如果消息大小超过1024字节减去当前消息大小，则记录错误并返回
        if (size > 1024U - msg_size)
        {
            LOG_ERR("%s " RED("ZMQ message is too large, size = %" PRIu64 " bytes"), tag(), size);
            ZMQClose();
            return;
        }

        // 如果可用数据小于消息大小，则返回
        if (avail < size) {
            return;
        }

        // 如果不是命令
        if (!command) {
// 如果处于调试模式，将数据从data指针开始的size大小的内容插入到msg末尾
#ifdef APP_DEBUG
    msg.insert(msg.end(), data, data + size);
#endif

// 累加消息大小
msg_size += size;
}

// 更新data指针和avail大小，直到avail为0
data += size;
avail -= size;
} while (more);

// 从m_ZMQRecvBuf中擦除data指针之前的数据
m_ZMQRecvBuf.erase(m_ZMQRecvBuf.begin(), m_ZMQRecvBuf.begin() + (data - m_ZMQRecvBuf.data()));

#ifdef APP_DEBUG
// 在调试模式下，在msg末尾添加空字符，并记录日志
msg.push_back('\0');
LOG_DEBUG(CYAN("tcp-zmq://%s:%u") BLACK_BOLD(" read ") CYAN_BOLD("%zu") BLACK_BOLD(" bytes") " %s", m_pool.host().data(), m_pool.zmq_port(), msg.size() - 1, msg.data());
#endif

// 清除先前的哈希并检查守护进程高度，以确保xmrig稍后会调用get_block_template RPC
// 不能直接调用get_block_template，因为守护进程还没有准备好
m_prevHash = nullptr;
m_blocktemplateRequestHash = nullptr;
send(kGetHeight);

// 获取作业超时时间，并重置定时器
const uint64_t t = m_pool.jobTimeout();
m_timer->stop();
m_timer->start(t, t);
}

// 关闭ZMQ连接
bool xmrig::DaemonClient::ZMQClose(bool shutdown)
{
    // 如果ZMQ连接状态为未连接或正在断开连接，则返回false
    if ((m_ZMQConnectionState == ZMQ_NOT_CONNECTED) || (m_ZMQConnectionState == ZMQ_DISCONNECTING)) {
        // 如果需要关闭，从存储中移除键
        if (shutdown) {
            m_storage.remove(m_key);
        }
        return false;
    }

    // 设置ZMQ连接状态为正在断开连接
    m_ZMQConnectionState = ZMQ_DISCONNECTING;

    // 如果ZMQ套接字没有关闭，并且支持保持活动状态
    if (uv_is_closing(reinterpret_cast<uv_handle_t*>(m_ZMQSocket)) == 0) {
        if (Platform::hasKeepalive()) {
            // 设置TCP套接字的保持活动状态
            uv_tcp_keepalive(m_ZMQSocket, 0, 60);
        }
        // 关闭ZMQ套接字，并根据shutdown参数选择回调函数
        uv_close(reinterpret_cast<uv_handle_t*>(m_ZMQSocket), shutdown ? onZMQShutdown : onZMQClose);
        // 如果不是shutdown，重试连接
        if (!shutdown) {
            retry();
        }
        return true;
    }

    return false;
}
```