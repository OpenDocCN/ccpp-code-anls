# `xmrig\src\base\net\stratum\Client.cpp`

```
/*
 * XMRig
 * 版权所有（c）2019      jtgrassie   <https://github.com/jtgrassie>
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，许可证的第3版
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cassert>  // 断言库
#include <cinttypes>  // C语言整数类型库
#include <iterator>  // 迭代器库
#include <cstdio>  // C标准输入输出库
#include <cstring>  // C字符串库
#include <utility>  // 实用程序库
#include <sstream>  // 字符串流库

#ifdef XMRIG_FEATURE_TLS
#   include <openssl/ssl.h>  // OpenSSL安全套接字层协议库
#   include <openssl/err.h>  // OpenSSL错误处理库
#   include "base/net/stratum/Tls.h"  // TLS库
#endif

#include "base/net/stratum/Client.h"  // 客户端库
#include "3rdparty/rapidjson/document.h"  // RapidJSON文档库
#include "3rdparty/rapidjson/error/en.h"  // RapidJSON错误库
#include "3rdparty/rapidjson/stringbuffer.h"  // RapidJSON字符串缓冲库
#include "3rdparty/rapidjson/writer.h"  // RapidJSON写入库
#include "base/io/json/Json.h"  // JSON库
#include "base/io/json/JsonRequest.h"  // JSON请求库
#include "base/io/log/Log.h"  // 日志库
#include "base/kernel/interfaces/IClientListener.h"  // 客户端监听器接口库
#include "base/kernel/Platform.h"  // 平台库
#include "base/net/dns/Dns.h"  // DNS库
#include "base/net/dns/DnsRecords.h"  // DNS记录库
#include "base/net/stratum/Socks5.h"  // Socks5库
#include "base/net/tools/NetBuffer.h"  // 网络缓冲库
#include "base/tools/Chrono.h"  // 时间库
#include "base/tools/cryptonote/BlobReader.h"  // 加密货币笔记库
#include "base/tools/Cvt.h"  // 转换库
#include "net/JobResult.h"  // 作业结果库

#ifdef _MSC_VER
#   define strncasecmp(x,y,z) _strnicmp(x,y,z)  // 定义不区分大小写的字符串比较
#endif

namespace xmrig {
    Storage<Client> Client::m_storage;  // 客户端存储
} /* namespace xmrig */
#ifdef APP_DEBUG
// 定义调试模式下的状态字符串数组
static const char *states[] = {
    "unconnected",
    "host-lookup",
    "connecting",
    "connected",
    "closing",
    "reconnecting"
};
#endif


// 客户端构造函数
xmrig::Client::Client(int id, const char *agent, IClientListener *listener) :
    BaseClient(id, listener),  // 调用基类构造函数
    m_agent(agent),  // 初始化代理
    m_sendBuf(1024),  // 初始化发送缓冲区大小
    m_tempBuf(256)  // 初始化临时缓冲区大小
{
    m_reader.setListener(this);  // 设置读取器监听器为当前客户端
    m_key = m_storage.add(this);  // 将当前客户端添加到存储中
}


// 客户端析构函数
xmrig::Client::~Client()
{
    delete m_socket;  // 释放套接字内存
}


// 断开连接
bool xmrig::Client::disconnect()
{
    m_keepAlive = 0;  // 设置保持连接为0
    m_expire    = 0;  // 设置过期时间为0
    m_failures  = -1;  // 设置失败次数为-1

    return close();  // 调用关闭连接函数
}


// 判断是否使用TLS
bool xmrig::Client::isTLS() const
{
#   ifdef XMRIG_FEATURE_TLS
    return m_pool.isTLS() && m_tls;  // 如果连接池使用TLS并且TLS存在，则返回true
#   else
    return false;  // 否则返回false
#   endif
}


// 获取TLS指纹
const char *xmrig::Client::tlsFingerprint() const
{
#   ifdef XMRIG_FEATURE_TLS
    if (isTLS() && m_pool.fingerprint() == nullptr) {
        return m_tls->fingerprint();  // 如果使用TLS并且连接池指纹为空，则返回TLS指纹
    }
#   endif

    return nullptr;  // 否则返回空指针
}


// 获取TLS版本
const char *xmrig::Client::tlsVersion() const
{
#   ifdef XMRIG_FEATURE_TLS
    if (isTLS()) {
        return m_tls->version();  // 如果使用TLS，则返回TLS版本
    }
#   endif

    return nullptr;  // 否则返回空指针
}


// 发送JSON数据
int64_t xmrig::Client::send(const rapidjson::Value &obj, Callback callback)
{
    assert(obj["id"] == sequence());  // 断言确保JSON对象中包含"id"字段，并且值等于当前序列号

    m_callbacks.insert({ sequence(), std::move(callback) });  // 插入回调函数到回调映射中

    return send(obj);  // 调用发送函数
}


// 发送JSON数据
int64_t xmrig::Client::send(const rapidjson::Value &obj)
{
    using namespace rapidjson;

    StringBuffer buffer(nullptr, 512);  // 创建字符串缓冲区
    Writer<StringBuffer> writer(buffer);  // 创建JSON写入器
    obj.Accept(writer);  // 将JSON对象写入缓冲区

    const size_t size = buffer.GetSize();  // 获取缓冲区大小
    if (size > kMaxSendBufferSize) {
        LOG_ERR("%s " RED("send failed: ") RED_BOLD("\"max send buffer size exceeded: %zu\""), tag(), size);  // 记录错误日志
        close();  // 关闭连接

        return -1;  // 返回错误码
    }

    if (size > (m_sendBuf.size() - 2)) {
        m_sendBuf.resize(((size + 1) / 1024 + 1) * 1024);  // 调整发送缓冲区大小
    }

    memcpy(m_sendBuf.data(), buffer.GetString(), size);  // 将缓冲区内容复制到发送缓冲区
    m_sendBuf[size]     = '\n';  // 在发送缓冲区末尾添加换行符
    m_sendBuf[size + 1] = '\0';  // 在发送缓冲区末尾添加空字符

    return send(size + 1);  // 调用发送函数
}
// 如果不是代理项目，并且结果的客户端ID不等于RPC ID，或者RPC ID为空，或者状态不是连接状态，则返回-1
#ifndef XMRIG_PROXY_PROJECT
    if (result.clientId != m_rpcId || m_rpcId.isNull() || m_state != ConnectedState) {
        return -1;
    }
#endif

// 如果结果的难度为0，则关闭连接并返回-1
    if (result.diff == 0) {
        close();
        return -1;
    }

// 使用rapidjson命名空间
    using namespace rapidjson;

// 如果是代理项目，则使用结果的nonce和result；否则使用临时缓冲区的nonce、data和signature
#ifdef XMRIG_PROXY_PROJECT
    const char *nonce = result.nonce;
    const char *data  = result.result;
#else
    char *nonce = m_tempBuf.data();
    char *data  = m_tempBuf.data() + 16;
    char *signature = m_tempBuf.data() + 88;

    // 将结果的nonce和data转换为十六进制字符串
    Cvt::toHex(nonce, sizeof(uint32_t) * 2 + 1, reinterpret_cast<const uint8_t *>(&result.nonce), sizeof(uint32_t));
    Cvt::toHex(data, 65, result.result(), 32);

    // 如果结果有矿工签名，则将其转换为十六进制字符串
    if (result.minerSignature()) {
        Cvt::toHex(signature, 129, result.minerSignature(), 64);
    }
#endif

// 创建rapidjson的Document对象和分配器
    Document doc(kObjectType);
    auto &allocator = doc.GetAllocator();

// 创建params对象，并添加id、job_id、nonce和result成员
    Value params(kObjectType);
    params.AddMember("id",     StringRef(m_rpcId.data()), allocator);
    params.AddMember("job_id", StringRef(result.jobId.data()), allocator);
    params.AddMember("nonce",  StringRef(nonce), allocator);
    params.AddMember("result", StringRef(data), allocator);

// 如果不是代理项目，并且结果有矿工签名，则添加sig成员
#ifndef XMRIG_PROXY_PROJECT
    if (result.minerSignature()) {
        params.AddMember("sig", StringRef(signature), allocator);
    }
// 如果是代理项目，并且结果有sig，则添加sig成员
#else
    if (result.sig) {
        params.AddMember("sig", StringRef(result.sig), allocator);
    }
#endif

// 如果具有EXT_ALGO，并且结果的算法有效，则添加algo成员
    if (has<EXT_ALGO>() && result.algorithm.isValid()) {
        params.AddMember("algo", StringRef(result.algorithm.name()), allocator);
    }

// 创建JSON-RPC请求
    JsonRequest::create(doc, m_sequence, "submit", params);

// 如果是代理项目，则将结果添加到m_results中；否则将结果添加到m_results中
#ifdef XMRIG_PROXY_PROJECT
    m_results[m_sequence] = SubmitResult(m_sequence, result.diff, result.actualDiff(), result.id, 0);
#else
    m_results[m_sequence] = SubmitResult(m_sequence, result.diff, result.actualDiff(), 0, result.backend);
#endif

// 发送JSON-RPC请求并返回结果
    return send(doc);
}
    # 如果代理对象有效
    if (m_pool.proxy().isValid()) {
        # 创建一个 Socks5 对象
        m_socks5 = new Socks5(this);
        # 解析代理主机的地址
        resolve(m_pool.proxy().host());
        # 返回
        return;
    }
// 如果支持 TLS 特性，并且连接的矿池使用了 TLS，则创建一个 Tls 对象
    if (m_pool.isTLS()) {
        m_tls = new Tls(this);
    }

// 解析矿池的主机地址
    resolve(m_pool.host());
}


// 连接矿池
void xmrig::Client::connect(const Pool &pool)
{
    setPool(pool);
    connect();
}


// 延迟删除客户端
void xmrig::Client::deleteLater()
{
    // 如果监听器为空，则直接返回
    if (!m_listener) {
        return;
    }

    m_listener = nullptr;

    // 如果无法断开连接，则从存储中移除键
    if (!disconnect()) {
        m_storage.remove(m_key);
    }
}


// 客户端心跳
void xmrig::Client::tick(uint64_t now)
{
    // 如果状态为已连接
    if (m_state == ConnectedState) {
        // 如果超时并且当前时间大于超时时间，则记录错误日志并关闭连接
        if (m_expire && now > m_expire) {
            LOG_DEBUG_ERR("[%s] timeout", url());
            close();
        }
        // 如果保持连接并且当前时间大于保持连接时间，则发送 ping
        else if (m_keepAlive && now > m_keepAlive) {
            ping();
        }

        return;
    }

    // 如果状态为重新连接中，并且超时时间已到，则重新连接
    if (m_state == ReconnectingState && m_expire && now > m_expire) {
        return connect();
    }

    // 如果状态为连接中，并且超时时间已到，则关闭连接
    if (m_state == ConnectingState && m_expire && now > m_expire) {
        close();
    }
}


// DNS 解析完成后的回调函数
void xmrig::Client::onResolved(const DnsRecords &records, int status, const char *error)
{
    m_dns.reset();

    // 断言监听器不为空，为空则重新连接
    assert(m_listener != nullptr);
    if (!m_listener) {
        return reconnect();
    }

    // 如果 DNS 解析出错或者返回的记录为空，则记录错误日志并重新连接
    if (status < 0 && records.isEmpty()) {
        if (!isQuiet()) {
            LOG_ERR("%s " RED("DNS error: ") RED_BOLD("\"%s\""), tag(), error);
        }

        return reconnect();
    }

    // 获取记录的 IP 地址，并连接到该地址
    const auto &record = records.get();
    m_ip = record.ip();

    connect(record.addr(m_socks5 ? m_pool.proxy().port() : m_pool.port()));
}


// 关闭连接
bool xmrig::Client::close()
{
    // 如果状态为正在关闭中，则返回连接是否为空
    if (m_state == ClosingState) {
        return m_socket != nullptr;
    }

    // 如果状态为未连接或者连接为空，则返回 false
    if (m_state == UnconnectedState || m_socket == nullptr) {
        return false;
    }

    // 设置状态为正在关闭中
    setState(ClosingState);

    // 如果连接未关闭，则设置 TCP 保持连接并关闭连接
    if (uv_is_closing(reinterpret_cast<uv_handle_t*>(m_socket)) == 0) {
        if (Platform::hasKeepalive()) {
            uv_tcp_keepalive(m_socket, 0, 60);
        }
        uv_close(reinterpret_cast<uv_handle_t*>(m_socket), Client::onClose);
    }

    return true;
}
bool xmrig::Client::parseJob(const rapidjson::Value &params, int *code)
{
    // 检查参数是否为对象，如果不是则返回错误码2
    if (!params.IsObject()) {
        *code = 2;
        return false;
    }

    // 创建一个新的作业对象，包括是否为NiceHash、矿池算法和RPC ID
    Job job(has<EXT_NICEHASH>(), m_pool.algorithm(), m_rpcId);

    // 设置作业的ID，如果失败则返回错误码3
    if (!job.setId(params["job_id"].GetString())) {
        *code = 3;
        return false;
    }

    // 获取算法和数据块，并设置作业的算法
    const char *algo = Json::getString(params, "algo");
    const char *blobData = Json::getString(params, "blob");
    if (algo) {
        job.setAlgorithm(algo);
    }
    else if (m_pool.coin().isValid()) {
        uint8_t blobVersion = 0;
        if (blobData) {
            Cvt::fromHex(&blobVersion, 1, blobData, 2);
        }
        job.setAlgorithm(m_pool.coin().algorithm(blobVersion));
    }

#   ifdef XMRIG_FEATURE_HTTP
    // 如果矿池模式为自选，则设置额外的nonce和矿池钱包地址
    if (m_pool.mode() == Pool::MODE_SELF_SELECT) {
        job.setExtraNonce(Json::getString(params, "extra_nonce"));
        job.setPoolWallet(Json::getString(params, "pool_wallet"));

        // 如果额外的nonce或矿池钱包地址为空，则返回错误码4
        if (job.extraNonce().isNull() || job.poolWallet().isNull()) {
            *code = 4;
            return false;
        }
    }
    else
#   endif
    {
        // 如果不是自选模式，则设置作业的数据块
        if (!job.setBlob(blobData)) {
            *code = 4;
            return false;
        }
    }

    // 设置作业的目标值，如果失败则返回错误码5
    if (!job.setTarget(params["target"].GetString())) {
        *code = 5;
        return false;
    }

    // 设置作业的高度
    job.setHeight(Json::getUint64(params, "height"));

    // 验证算法是否有效，如果不是则返回错误码6
    if (!verifyAlgorithm(job.algorithm(), algo)) {
        *code = 6;
        return false;
    }

    // 如果矿池模式不是自选，并且算法是RANDOM_X，且设置种子哈希失败，则返回错误码7
    if (m_pool.mode() != Pool::MODE_SELF_SELECT && job.algorithm().family() == Algorithm::RANDOM_X && !job.setSeedHash(Json::getString(params, "seed_hash"))) {
        *code = 7;
        return false;
    }

    // 设置作业的签名密钥
    job.setSigKey(Json::getString(params, "sig_key"));

    // 设置作业的客户端ID
    m_job.setClientId(m_rpcId);

    // 如果当前作业与新作业不同，则更新作业并返回true
    if (m_job != job) {
        m_jobs++;
        m_job = std::move(job);
        return true;
    }

    // 如果作业数为0，则返回false
    if (m_jobs == 0) { // https://github.com/xmrig/xmrig/issues/459
        return false;
    }
}
    # 如果不是静默模式，则记录警告信息，包括标签和重复作业的提示
    if (!isQuiet()) {
        LOG_WARN("%s " YELLOW("duplicate job received, reconnect"), tag());
    }
    
    # 关闭连接
    close();
    
    # 返回 false，表示处理失败
    return false;
# 定义 send 函数，接收一个 BIO 对象作为参数
bool xmrig::Client::send(BIO *bio)
{
#   ifdef XMRIG_FEATURE_TLS
    // 定义一个缓冲区对象 buf
    uv_buf_t buf;
    // 从 BIO 对象中获取数据并存入 buf 中，同时返回数据长度
    buf.len = BIO_get_mem_data(bio, &buf.base); // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)

    // 如果数据长度为 0，则返回 true
    if (buf.len == 0) {
        return true;
    }

    // 打印调试信息，包括 URL 和数据长度
    LOG_DEBUG("[%s] TLS send     (%d bytes)", url(), static_cast<int>(buf.len));

    // 定义一个布尔值 result，并初始化为 false
    bool result = false;
    // 如果状态为 ConnectedState 并且流可写，则调用 write 函数，并将结果赋给 result
    if (state() == ConnectedState && uv_is_writable(stream())) {
        result = write(buf);
    }
    // 否则打印错误信息
    else {
        LOG_DEBUG_ERR("[%s] send failed, invalid state: %d", url(), m_state);
    }

    // 重置 BIO 对象
    (void) BIO_reset(bio);

    // 返回 result
    return result;
#   else
    // 如果没有启用 TLS 特性，则返回 false
    return false;
#   endif
}

# 验证算法是否有效
bool xmrig::Client::verifyAlgorithm(const Algorithm &algorithm, const char *algo) const
{
    // 如果算法无效，则打印相应错误信息，并返回 false
    if (!algorithm.isValid()) {
        // 如果不是静默模式，则打印错误信息
        if (!isQuiet()) {
            if (algo == nullptr) {
                LOG_ERR("%s " RED("unknown algorithm, make sure you set \"algo\" or \"coin\" option"), tag(), algo);
            }
            else {
                LOG_ERR("%s " RED("unsupported algorithm ") RED_BOLD("\"%s\" ") RED("detected, reconnect"), tag(), algo);
            }
        }
        return false;
    }

    // 调用监听器的 onVerifyAlgorithm 函数，并将结果存入 ok
    bool ok = true;
    m_listener->onVerifyAlgorithm(this, algorithm, &ok);

    // 如果不是静默模式且 ok 为 false，则打印错误信息
    if (!ok && !isQuiet()) {
        LOG_ERR("%s " RED("incompatible/disabled algorithm ") RED_BOLD("\"%s\" ") RED("detected, reconnect"), tag(), algorithm.name());
    }

    // 返回 ok
    return ok;
}

# 写入数据到流
bool xmrig::Client::write(const uv_buf_t &buf)
{
    // 尝试将数据写入流，返回写入的字节数
    const int rc = uv_try_write(stream(), &buf, 1);
    // 如果写入的字节数等于数据长度，则返回 true
    if (static_cast<size_t>(rc) == buf.len) {
        return true;
    }

    // 如果不是静默模式，则打印错误信息
    if (!isQuiet()) {
        LOG_ERR("%s " RED("write error: ") RED_BOLD("\"%s\""), tag(), uv_strerror(rc));
    }

    // 关闭连接
    close();

    // 返回 false
    return false;
}

# 解析主机名
int xmrig::Client::resolve(const String &host)
{
    // 设置状态为 HostLookupState
    setState(HostLookupState);

    // 重置读取器
    m_reader.reset();

    // 如果失败次数为 -1，则将其重置为 0
    if (m_failures == -1) {
        m_failures = 0;
    }

    // 解析主机名并返回结果
    m_dns = Dns::resolve(host, this);

    return 0;
}

# 发送数据
int64_t xmrig::Client::send(size_t size)
{
    # 记录调试信息，包括 URL、发送字节数和发送的数据
    LOG_DEBUG("[%s] send (%d bytes): \"%.*s\"", url(), size, static_cast<int>(size) - 1, m_sendBuf.data());
// 如果支持 TLS，则进行 TLS 发送
#ifdef XMRIG_FEATURE_TLS
    if (isTLS()) {
        // 如果 TLS 发送失败，则返回 -1
        if (!m_tls->send(m_sendBuf.data(), size)) {
            return -1;
        }
    }
    else
#endif
    {
        // 如果状态不是已连接状态，或者流不可写，则记录错误日志并返回 -1
        if (state() != ConnectedState || !uv_is_writable(stream())) {
            LOG_DEBUG_ERR("[%s] send failed, invalid state: %d", url(), m_state);
            return -1;
        }

        // 初始化 uv_buf_t 对象，用于发送数据
        uv_buf_t buf = uv_buf_init(m_sendBuf.data(), (unsigned int) size);

        // 如果发送失败，则返回 -1
        if (!write(buf)) {
            return -1;
        }
    }

    // 设置超时时间，并返回递增后的序列号
    m_expire = Chrono::steadyMSecs() + kResponseTimeout;
    return m_sequence++;
}


// 连接到指定地址
void xmrig::Client::connect(const sockaddr *addr)
{
    // 设置连接状态为连接中
    setState(ConnectingState);

    // 初始化 uv_connect_t 对象，并设置其数据
    auto req = new uv_connect_t;
    req->data = m_storage.ptr(m_key);

    // 初始化 uv_tcp_t 对象，并设置其数据
    m_socket = new uv_tcp_t;
    m_socket->data = m_storage.ptr(m_key);

    // 初始化 TCP 对象，并设置 TCP 无延迟
    uv_tcp_init(uv_default_loop(), m_socket);
    uv_tcp_nodelay(m_socket, 1);

    // 如果平台支持保持连接，则设置 TCP 保持连接
    if (Platform::hasKeepalive()) {
        uv_tcp_keepalive(m_socket, 1, 60);
    }

    // 发起 TCP 连接请求
    uv_tcp_connect(req, m_socket, addr, onConnect);
}


// 进行握手操作
void xmrig::Client::handshake()
{
    // 如果使用 SOCKS5 代理，则进行 SOCKS5 握手
    if (m_socks5) {
        return m_socks5->handshake();
    }

    // 如果支持 TLS，则进行 TLS 握手
#ifdef XMRIG_FEATURE_TLS
    if (isTLS()) {
        // 设置超时时间，并进行 TLS 握手
        m_expire = Chrono::steadyMSecs() + kResponseTimeout;
        m_tls->handshake(m_pool.isSNI() ? m_pool.host().data() : nullptr);
    }
    else
#endif
    {
        // 否则进行登录操作
        login();
    }
}


// 解析登录结果
bool xmrig::Client::parseLogin(const rapidjson::Value &result, int *code)
{
    // 设置 RPC ID，并检查是否为空
    setRpcId(Json::getString(result, "id"));
    if (rpcId().isNull()) {
        *code = 1;
        return false;
    }

    // 解析扩展信息
    parseExtensions(result);

    // 解析工作信息，并重置工作数量
    const bool rc = parseJob(result["job"], code);
    m_jobs = 0;

    return rc;
}


// 发起登录操作
void xmrig::Client::login()
{
    using namespace rapidjson;
    // 清空结果集
    m_results.clear();

    // 创建 JSON 文档，并获取分配器
    Document doc(kObjectType);
    auto &allocator = doc.GetAllocator();

    // 创建参数对象，并添加登录信息
    Value params(kObjectType);
    params.AddMember("login", m_user.toJSON(),     allocator);
    params.AddMember("pass",  m_password.toJSON(), allocator);
}
    # 向参数中添加代理信息，使用给定的代理字符串和分配器
    params.AddMember("agent", StringRef(m_agent),  allocator);

    # 如果刚体ID不为空
    if (!m_rigId.isNull()) {
        # 向参数中添加刚体ID信息，使用刚体ID的JSON表示和分配器
        params.AddMember("rigid", m_rigId.toJSON(), allocator);
    }

    # 调用监听器的登录方法，传入当前对象、文档和参数
    m_listener->onLogin(this, doc, params);

    # 创建一个JSON请求，传入文档、请求ID、请求类型和参数
    JsonRequest::create(doc, 1, "login", params);

    # 发送请求
    send(doc);
# 关闭客户端连接时的操作
void xmrig::Client::onClose()
{
    # 删除套接字对象
    delete m_socket;

    # 将套接字指针置为空
    m_socket = nullptr;
    # 设置客户端状态为未连接状态
    setState(UnconnectedState);

    #ifdef XMRIG_FEATURE_TLS
    # 如果启用了 TLS 功能
    if (m_tls) {
        # 删除 TLS 对象
        delete m_tls;
        # 将 TLS 指针置为空
        m_tls = nullptr;
    }
    #endif

    # 重新连接
    reconnect();
}

# 解析收到的数据
void xmrig::Client::parse(char *line, size_t len)
{
    # 开始计时
    startTimeout();

    # 记录调试日志，包括收到的数据内容和长度
    LOG_DEBUG("[%s] received (%d bytes): \"%.*s\"", url(), len, static_cast<int>(len), line);

    # 如果收到的数据长度小于22或者第一个字符不是 '{'，则返回
    if (len < 22 || line[0] != '{') {
        if (!isQuiet()) {
            # 如果不是静默模式，则记录错误日志
            LOG_ERR("%s " RED("JSON decode failed"), tag());
        }
        return;
    }

    # 创建 JSON 文档对象
    rapidjson::Document doc;
    # 尝试解析收到的数据，如果解析出错，则返回
    if (doc.ParseInsitu(line).HasParseError()) {
        if (!isQuiet()) {
            # 如果不是静默模式，则记录错误日志
            LOG_ERR("%s " RED("JSON decode failed: ") RED_BOLD("\"%s\""), tag(), rapidjson::GetParseError_En(doc.GetParseError()));
        }
        return;
    }

    # 如果解析出的文档不是对象类型，则返回
    if (!doc.IsObject()) {
        return;
    }

    # 获取文档中的 id、error 和 method 字段的值
    const auto &id    = Json::getValue(doc, "id");
    const auto &error = Json::getValue(doc, "error");
    const char *method = Json::getString(doc, "method");
}
    // 检查方法是否存在且为"client.reconnect"
    if (method && strcmp(method, "client.reconnect") == 0) {
        // 获取参数
        const auto &params = Json::getValue(doc, "params");
        // 如果参数不是数组，记录错误并返回
        if (!params.IsArray()) {
            LOG_ERR("%s " RED("invalid client.reconnect notification: params is not an array"), tag());
            return;
        }

        // 获取参数数组
        auto arr = params.GetArray();

        // 如果参数数组为空，记录错误并返回
        if (arr.Empty()) {
            LOG_ERR("%s " RED("invalid client.reconnect notification: params array is empty"), tag());
            return;
        }

        // 如果参数数组大小不为2，记录错误并返回
        if (arr.Size() != 2) {
            LOG_ERR("%s " RED("invalid client.reconnect notification: params array has wrong size"), tag());
            return;
        }

        // 如果第一个参数不是字符串，记录错误并返回
        if (!arr[0].IsString()) {
            LOG_ERR("%s " RED("invalid client.reconnect notification: host is not a string"), tag());
            return;
        }

        // 如果第二个参数不是字符串，记录错误并返回
        if (!arr[1].IsString()) {
            LOG_ERR("%s " RED("invalid client.reconnect notification: port is not a string"), tag());
            return;
        }

        // 拼接主机和端口信息，记录警告日志
        std::stringstream s;
        s << arr[0].GetString() << ":" << arr[1].GetString();
        LOG_WARN("%s " YELLOW("client.reconnect to %s"), tag(), s.str().c_str());
        // 设置连接池 URL
        setPoolUrl(s.str().c_str());
        // 重新连接
        return reconnect();
    }

    // 如果 ID 是 Int64 类型，解析响应
    if (id.IsInt64()) {
        return parseResponse(id.GetInt64(), Json::getValue(doc, "result"), error);
    }

    // 如果方法不存在，返回
    if (!method) {
        return;
    }

    // 如果存在错误对象，记录错误信息
    if (error.IsObject()) {
        if (!isQuiet()) {
            LOG_ERR("%s " RED("error: ") RED_BOLD("\"%s\"") RED(", code: ") RED_BOLD("%d"),
                    tag(), Json::getString(error, "message"), Json::getInt(error, "code"));
        }

        return;
    }

    // 解析通知
    parseNotification(method, Json::getValue(doc, "params"), error);
}

// 解析挖矿客户端的扩展信息
void xmrig::Client::parseExtensions(const rapidjson::Value &result)
{
    // 重置扩展信息
    m_extensions.reset();

    // 如果结果中没有"extensions"字段，则返回
    if (!result.HasMember("extensions")) {
        return;
    }

    // 获取"extensions"字段的值
    const rapidjson::Value &extensions = result["extensions"];
    // 如果值不是数组，则返回
    if (!extensions.IsArray()) {
        return;
    }

    // 遍历扩展数组
    for (const rapidjson::Value &ext : extensions.GetArray()) {
        // 如果不是字符串类型，则继续下一次循环
        if (!ext.IsString()) {
            continue;
        }

        // 获取扩展名
        const char *name = ext.GetString();

        // 根据扩展名设置对应的扩展标志位
        if (strcmp(name, "algo") == 0) {
            setExtension(EXT_ALGO, true);
        }
        else if (strcmp(name, "nicehash") == 0) {
            setExtension(EXT_NICEHASH, true);
        }
        else if (strcmp(name, "connect") == 0) {
            setExtension(EXT_CONNECT, true);
        }
        else if (strcmp(name, "keepalive") == 0) {
            setExtension(EXT_KEEPALIVE, true);
            startTimeout();
        }
#       ifdef XMRIG_FEATURE_TLS
        else if (strcmp(name, "tls") == 0) {
            setExtension(EXT_TLS, true);
        }
#       endif
    }
}

// 解析通知信息
void xmrig::Client::parseNotification(const char *method, const rapidjson::Value &params, const rapidjson::Value &)
{
    // 如果通知方法是"job"
    if (strcmp(method, "job") == 0) {
        int code = -1;
        // 解析工作参数
        if (parseJob(params, &code)) {
            // 调用监听器的工作接收回调函数
            m_listener->onJobReceived(this, m_job, params);
        }
        else {
            // 关闭客户端
            close();
        }

        return;
    }
}

// 解析响应信息
void xmrig::Client::parseResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error)
{
    // 如果处理响应成功，则返回
    if (handleResponse(id, result, error)) {
        return;
    }

    // 如果错误信息是对象类型
    if (error.IsObject()) {
        // 获取错误消息
        const char *message = error["message"].GetString();

        // 如果处理响应失败且不是静默模式，则记录错误日志
        if (!handleSubmitResponse(id, message) && !isQuiet()) {
            LOG_ERR("%s " RED("error: ") RED_BOLD("\"%s\"") RED(", code: ") RED_BOLD("%d"), tag(), message, Json::getInt(error, "code"));
        }

        // 如果是第一个请求或者是关键错误消息，则关闭客户端
        if (m_id == 1 || isCriticalError(message)) {
            close();
        }

        return;
    }
}
    # 如果返回结果不是一个对象，则直接返回
    if (!result.IsObject()) {
        return;
    }

    # 如果 id 等于 1，则执行以下操作
    if (id == 1) {
        # 初始化 code 为 -1
        int code = -1;
        # 如果解析登录结果失败，则记录错误信息并关闭连接
        if (!parseLogin(result, &code)) {
            if (!isQuiet()) {
                LOG_ERR("%s " RED("login error code: ") RED_BOLD("%d"), tag(), code);
            }

            close();
            return;
        }

        # 重置失败次数为 0，并通知监听器登录成功
        m_failures = 0;
        m_listener->onLoginSuccess(this);

        # 如果存在有效的作业，则通知监听器作业接收成功
        if (m_job.isValid()) {
            m_listener->onJobReceived(this, m_job, result["job"]);
        }

        # 返回
        return;
    }

    # 处理提交响应
    handleSubmitResponse(id);
}

// 发送 ping 消息
void xmrig::Client::ping()
{
    // 格式化 ping 消息并发送
    send(snprintf(m_sendBuf.data(), m_sendBuf.size(), "{\"id\":%" PRId64 ",\"jsonrpc\":\"2.0\",\"method\":\"keepalived\",\"params\":{\"id\":\"%s\"}}\n", m_sequence, m_rpcId.data()));

    // 重置 keepAlive 计数
    m_keepAlive = 0;
}

// 读取数据
void xmrig::Client::read(ssize_t nread, const uv_buf_t *buf)
{
    // 转换读取的字节数为无符号整数
    const auto size = static_cast<size_t>(nread);
    // 处理读取错误
    if (nread < 0) {
        if (!isQuiet()) {
            // 输出错误信息
            LOG_ERR("%s " RED("read error: ") RED_BOLD("\"%s\""), tag(), uv_strerror(static_cast<int>(nread)));
        }
        // 关闭连接
        close();
        return;
    }

    // 断言监听器不为空
    assert(m_listener != nullptr);
    if (!m_listener) {
        // 重新连接
        return reconnect();
    }

    // 处理 SOCKS5 代理
    if (m_socks5) {
        // 读取数据并检查是否准备就绪
        m_socks5->read(buf->base, size);
        if (m_socks5->isReady()) {
            delete m_socks5;
            m_socks5 = nullptr;
#           ifdef XMRIG_FEATURE_TLS
            // 如果使用 TLS，则创建 TLS 对象
            if (m_pool.isTLS() && !m_tls) {
                m_tls = new Tls(this);
            }
#           endif
            // 发起握手
            handshake();
        }
        return;
    }

#   ifdef XMRIG_FEATURE_TLS
    // 如果使用 TLS，则读取数据
    if (isTLS()) {
        LOG_DEBUG("[%s] TLS received (%d bytes)", url(), static_cast<int>(nread));
        m_tls->read(buf->base, size);
    }
    else
#   endif
    {
        // 否则解析数据
        m_reader.parse(buf->base, size);
    }
}

// 重新连接
void xmrig::Client::reconnect()
{
    if (!m_listener) {
        // 如果监听器为空，则移除键
        m_storage.remove(m_key);
        return;
    }

    // 重置 keepAlive 计数
    m_keepAlive = 0;

    if (m_failures == -1) {
        // 如果失败次数为 -1，则关闭连接
        return m_listener->onClose(this, -1);
    }

    // 设置状态为重新连接
    setState(ReconnectingState);

    // 增加失败次数并通知监听器
    m_failures++;
    m_listener->onClose(this, static_cast<int>(m_failures));
}

// 设置连接状态
void xmrig::Client::setState(SocketState state)
{
    // 输出状态变化信息
    LOG_DEBUG("[%s] state: \"%s\" -> \"%s\"", url(), states[m_state], states[state]);

    // 如果状态未变化，则返回
    if (m_state == state) {
        return;
    }

    switch (state) {
    case HostLookupState:
        // 如果是主机查找状态，则重置过期时间
        m_expire = 0;
        break;

    case ConnectingState:
        // 如果是连接状态，则设置过期时间
        m_expire = Chrono::steadyMSecs() + kConnectTimeout;
        break;
    # 如果状态为重新连接状态
    case ReconnectingState:
        # 设置过期时间为当前时间加上重试暂停时间
        m_expire = Chrono::steadyMSecs() + m_retryPause;
        # 跳出 switch 语句
        break;

    # 如果状态为其他情况
    default:
        # 什么也不做，直接跳出 switch 语句
        break;
    }

    # 将状态设置为给定的状态
    m_state = state;
// 开始超时计时
void xmrig::Client::startTimeout()
{
    // 重置超时时间
    m_expire = 0;

    // 如果支持保持连接功能
    if (has<EXT_KEEPALIVE>()) {
        // 计算保持连接的超时时间（毫秒）
        const uint64_t ms = static_cast<uint64_t>(m_pool.keepAlive() > 0 ? m_pool.keepAlive() : Pool::kKeepAliveTimeout) * 1000;
        // 设置保持连接的超时时间点
        m_keepAlive = Chrono::steadyMSecs() + ms;
    }
}

// 判断是否为关键错误
bool xmrig::Client::isCriticalError(const char *message)
{
    // 如果消息为空，则不是关键错误
    if (!message) {
        return false;
    }

    // 判断消息是否包含特定关键错误信息
    if (strncasecmp(message, "Unauthenticated", 15) == 0) {
        return true;
    }

    if (strncasecmp(message, "your IP is banned", 17) == 0) {
        return true;
    }

    if (strncasecmp(message, "IP Address currently banned", 27) == 0) {
        return true;
    }

    if (strncasecmp(message, "Invalid job id", 14) == 0) {
        return true;
    }

    // 默认不是关键错误
    return false;
}

// 关闭处理函数
void xmrig::Client::onClose(uv_handle_t *handle)
{
    // 获取对应的客户端对象
    auto client = getClient(handle->data);
    // 如果客户端对象不存在，则返回
    if (!client) {
        return;
    }
    // 调用客户端对象的关闭处理函数
    client->onClose();
}

// 连接处理函数
void xmrig::Client::onConnect(uv_connect_t *req, int status)
{
    // 获取对应的客户端对象
    auto client = getClient(req->data);
    // 释放连接请求对象
    delete req;
    // 如果客户端对象不存在，则返回
    if (!client) {
        return;
    }
    // 如果连接出现错误
    if (status < 0) {
        // 如果不是静默模式，则记录连接错误日志
        if (!client->isQuiet()) {
            LOG_ERR("%s %s " RED("connect error: ") RED_BOLD("\"%s\""), client->tag(), client->ip().data(), uv_strerror(status));
        }
        // 如果处于重新连接或关闭状态，则返回
        if (client->state() == ReconnectingState || client->state() == ClosingState) {
            return;
        }
        // 如果不是连接状态，则返回
        if (client->state() != ConnectingState) {
            return;
        }
        // 关闭客户端连接
        client->close();
        return;
    }
    // 如果客户端已连接，则返回
    if (client->state() == ConnectedState) {
        return;
    }
    // 设置客户端状态为已连接
    client->setState(ConnectedState);
    // 开始读取数据
    uv_read_start(client->stream(), NetBuffer::onAlloc, onRead);
    // 发起握手
    client->handshake();
}

// 读取数据处理函数
void xmrig::Client::onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf)
{
    // 获取对应的客户端对象
    auto client = getClient(stream->data);
    // 如果客户端对象存在，则调用读取数据处理函数
    if (client) {
        client->read(nread, buf);
    }
    // 释放缓冲区
    NetBuffer::release(buf);
}
```