# `xmrig\src\base\net\stratum\Pool.cpp`

```
/* XMRig
 * 版权所有 (c) 2019      Howard Chu  <https://github.com/hyc>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。详细信息请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <cassert>
#include <cstring>
#include <cstdlib>
#include <cstdio>
#include <string>


#include "base/net/stratum/Pool.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"
#include "base/kernel/Platform.h"
#include "base/net/stratum/Client.h"

#if defined XMRIG_ALGO_KAWPOW || defined XMRIG_ALGO_GHOSTRIDER
#   include "base/net/stratum/AutoClient.h"
#   include "base/net/stratum/EthStratumClient.h"
#endif


#ifdef XMRIG_FEATURE_HTTP
#   include "base/net/stratum/DaemonClient.h"
#   include "base/net/stratum/SelfSelectClient.h"
#endif


#ifdef XMRIG_FEATURE_BENCHMARK
#   include "base/net/stratum/benchmark/BenchClient.h"
#   include "base/net/stratum/benchmark/BenchConfig.h"
#endif


#ifdef _MSC_VER
#   define strcasecmp  _stricmp
#endif


namespace xmrig {


const String Pool::kDefaultPassword       = "x";
const String Pool::kDefaultUser           = "x";


const char *Pool::kAlgo                   = "algo";
const char *Pool::kCoin                   = "coin";
const char *Pool::kDaemon                 = "daemon";
// 定义常量字符串，表示不同的连接池配置项
const char *Pool::kDaemonPollInterval     = "daemon-poll-interval";
const char *Pool::kDaemonJobTimeout       = "daemon-job-timeout";
const char *Pool::kDaemonZMQPort          = "daemon-zmq-port";
const char *Pool::kEnabled                = "enabled";
const char *Pool::kFingerprint            = "tls-fingerprint";
const char *Pool::kKeepalive              = "keepalive";
const char *Pool::kNicehash               = "nicehash";
const char *Pool::kPass                   = "pass";
const char *Pool::kRigId                  = "rig-id";
const char *Pool::kSelfSelect             = "self-select";
const char *Pool::kSOCKS5                 = "socks5";
const char *Pool::kSubmitToOrigin         = "submit-to-origin";
const char *Pool::kTls                    = "tls";
const char *Pool::kSni                    = "sni";
const char *Pool::kUrl                    = "url";
const char *Pool::kUser                   = "user";
const char *Pool::kSpendSecretKey         = "spend-secret-key";
const char *Pool::kNicehashHost           = "nicehash.com";

} // namespace xmrig

// 构造函数，初始化连接池对象
xmrig::Pool::Pool(const char *url) :
    m_flags(1 << FLAG_ENABLED), // 设置标志位，表示连接池已启用
    m_pollInterval(kDefaultPollInterval), // 设置轮询间隔为默认值
    m_jobTimeout(kDefaultJobTimeout), // 设置作业超时时间为默认值
    m_url(url) // 设置连接池的 URL
{
}

// 构造函数，初始化连接池对象
xmrig::Pool::Pool(const char *host, uint16_t port, const char *user, const char *password, const char* spendSecretKey, int keepAlive, bool nicehash, bool tls, Mode mode) :
    m_keepAlive(keepAlive), // 设置保持连接的时间
    m_mode(mode), // 设置连接模式
    m_flags(1 << FLAG_ENABLED), // 设置标志位，表示连接池已启用
    m_password(password), // 设置连接密码
    m_user(user), // 设置连接用户名
    m_spendSecretKey(spendSecretKey), // 设置花费密钥
    m_pollInterval(kDefaultPollInterval), // 设置轮询间隔为默认值
    m_jobTimeout(kDefaultJobTimeout), // 设置作业超时时间为默认值
    m_url(host, port, tls) // 设置连接池的 URL
{
    m_flags.set(FLAG_NICEHASH, nicehash || strstr(host, kNicehashHost)); // 设置标志位，表示是否为 NiceHash 连接池
    m_flags.set(FLAG_TLS,      tls); // 设置标志位，表示是否启用 TLS
}

// 构造函数，初始化连接池对象
xmrig::Pool::Pool(const rapidjson::Value &object) :
    m_flags(1 << FLAG_ENABLED), // 设置标志位，表示连接池已启用
    m_pollInterval(kDefaultPollInterval), // 设置轮询间隔为默认值
    m_jobTimeout(kDefaultJobTimeout), // 设置作业超时时间为默认值
    m_url(Json::getString(object, kUrl)) // 从 JSON 对象中获取连接池的 URL
{
    # 如果 URL 无效，则直接返回，不进行后续操作
    if (!m_url.isValid()) {
        return;
    }

    # 从 JSON 对象中获取用户信息并赋值给 m_user
    m_user           = Json::getString(object, kUser);
    # 从 JSON 对象中获取花费密钥信息并赋值给 m_spendSecretKey
    m_spendSecretKey = Json::getString(object, kSpendSecretKey);
    # 从 JSON 对象中获取密码信息并赋值给 m_password
    m_password       = Json::getString(object, kPass);
    # 从 JSON 对象中获取矿机 ID 信息并赋值给 m_rigId
    m_rigId          = Json::getString(object, kRigId);
    # 从 JSON 对象中获取指纹信息并赋值给 m_fingerprint
    m_fingerprint    = Json::getString(object, kFingerprint);
    # 从 JSON 对象中获取守护进程轮询间隔信息并赋值给 m_pollInterval，如果不存在则使用默认值
    m_pollInterval   = Json::getUint64(object, kDaemonPollInterval, kDefaultPollInterval);
    # 从 JSON 对象中获取作业超时信息并赋值给 m_jobTimeout，如果不存在则使用默认值
    m_jobTimeout     = Json::getUint64(object, kDaemonJobTimeout, kDefaultJobTimeout);
    # 从 JSON 对象中获取算法信息并赋值给 m_algorithm
    m_algorithm      = Json::getString(object, kAlgo);
    # 从 JSON 对象中获取货币信息并赋值给 m_coin
    m_coin           = Json::getString(object, kCoin);
    # 从 JSON 对象中获取守护进程信息并赋值给 m_daemon
    m_daemon         = Json::getString(object, kSelfSelect);
    # 从 JSON 对象中获取代理信息并赋值给 m_proxy
    m_proxy          = Json::getValue(object, kSOCKS5);
    # 从 JSON 对象中获取 ZMQ 端口信息并赋值给 m_zmqPort，如果不存在则使用默认值
    m_zmqPort        = Json::getInt(object, kDaemonZMQPort, m_zmqPort);

    # 设置 FLAG_ENABLED 标志位，根据 JSON 对象中的 kEnabled 值，如果不存在则默认为 true
    m_flags.set(FLAG_ENABLED,  Json::getBool(object, kEnabled, true));
    # 设置 FLAG_NICEHASH 标志位，根据 JSON 对象中的 kNicehash 值或者 URL 中是否包含 kNicehashHost 来确定
    m_flags.set(FLAG_NICEHASH, Json::getBool(object, kNicehash) || m_url.host().contains(kNicehashHost));
    # 设置 FLAG_TLS 标志位，根据 JSON 对象中的 kTls 值或者 URL 是否为 TLS 来确定
    m_flags.set(FLAG_TLS,      Json::getBool(object, kTls) || m_url.isTLS());
    # 设置 FLAG_SNI 标志位，根据 JSON 对象中的 kSni 值来确定
    m_flags.set(FLAG_SNI,      Json::getBool(object, kSni));

    # 设置 keep-alive 参数，根据 JSON 对象中的 kKeepalive 值来确定
    setKeepAlive(Json::getValue(object, kKeepalive));

    # 如果 m_daemon 有效，则设置为 MODE_SELF_SELECT 模式，并根据 JSON 对象中的 kSubmitToOrigin 值来确定是否提交到原始地址
    if (m_daemon.isValid()) {
        m_mode           = MODE_SELF_SELECT;
        m_submitToOrigin = Json::getBool(object, kSubmitToOrigin, m_submitToOrigin);
    }
    # 如果 JSON 对象中存在 kDaemon 值，则设置为 MODE_DAEMON 模式
    else if (Json::getBool(object, kDaemon)) {
        m_mode = MODE_DAEMON;
    }
# 如果定义了 XMRIG_FEATURE_BENCHMARK，则以下代码块为 Pool 类的构造函数
#ifdef XMRIG_FEATURE_BENCHMARK
xmrig::Pool::Pool(const std::shared_ptr<BenchConfig> &benchmark) :
    m_mode(MODE_BENCHMARK),  # 设置模式为 MODE_BENCHMARK
    m_flags(1 << FLAG_ENABLED),  # 设置标志位为 1 左移 FLAG_ENABLED 位
    m_url(BenchConfig::kBenchmark),  # 设置 URL 为 BenchConfig::kBenchmark
    m_benchmark(benchmark)  # 设置 benchmark 对象
{
}
#endif

# 如果定义了 XMRIG_FEATURE_BENCHMARK，则以下代码块为 benchmark 方法
xmrig::BenchConfig *xmrig::Pool::benchmark() const
{
    assert(m_mode == MODE_BENCHMARK && m_benchmark);  # 断言模式为 MODE_BENCHMARK 并且 benchmark 对象存在

    return m_benchmark.get();  # 返回 benchmark 对象
}

# 如果定义了 XMRIG_FEATURE_BENCHMARK，则以下代码块为 benchSize 方法
uint32_t xmrig::Pool::benchSize() const
{
    return benchmark()->size();  # 返回 benchmark 对象的大小
}

# 如果未定义 XMRIG_FEATURE_BENCHMARK，则以下代码块为 isEnabled 方法
bool xmrig::Pool::isEnabled() const
{
#   ifndef XMRIG_FEATURE_TLS
    if (isTLS()) {  # 如果启用了 TLS，则返回 false
        return false;
    }
#   endif

#   ifndef XMRIG_FEATURE_HTTP
    if (m_mode == MODE_DAEMON) {  # 如果模式为 MODE_DAEMON，则返回 false
        return false;
    }
#   endif

#   ifndef XMRIG_FEATURE_HTTP
    if (m_mode == MODE_SELF_SELECT) {  # 如果模式为 MODE_SELF_SELECT，则返回 false
        return false;
    }
#   endif

    return m_flags.test(FLAG_ENABLED) && isValid();  # 返回标志位是否启用并且是否有效
}

# 如果未定义 XMRIG_FEATURE_BENCHMARK，则以下代码块为 isEqual 方法
bool xmrig::Pool::isEqual(const Pool &other) const
{
    return (m_flags           == other.m_flags  # 比较各个属性是否相等
            && m_keepAlive    == other.m_keepAlive
            && m_algorithm    == other.m_algorithm
            && m_coin         == other.m_coin
            && m_mode         == other.m_mode
            && m_fingerprint  == other.m_fingerprint
            && m_password     == other.m_password
            && m_rigId        == other.m_rigId
            && m_url          == other.m_url
            && m_user         == other.m_user
            && m_pollInterval == other.m_pollInterval
            && m_jobTimeout   == other.m_jobTimeout
            && m_daemon       == other.m_daemon
            && m_proxy        == other.m_proxy
            );
}

# 如果未定义 XMRIG_FEATURE_BENCHMARK，则以下代码块为 createClient 方法
xmrig::IClient *xmrig::Pool::createClient(int id, IClientListener *listener) const
{
    IClient *client = nullptr;  # 初始化 client 为 nullptr

    if (m_mode == MODE_POOL) {  # 如果模式为 MODE_POOL
# 如果定义了XMRIG_ALGO_KAWPOW或者XMRIG_ALGO_GHOSTRIDER
const uint32_t f = m_algorithm.family();
# 如果算法族是KAWPOW或者GHOSTRIDER，或者币种是RAVEN
if ((f == Algorithm::KAWPOW) || (f == Algorithm::GHOSTRIDER) || (m_coin == Coin::RAVEN)) {
    # 创建一个EthStratumClient对象
    client = new EthStratumClient(id, Platform::userAgent(), listener);
}
# 否则
else {
    # 创建一个Client对象
    client = new Client(id, Platform::userAgent(), listener);
}

# 如果定义了XMRIG_FEATURE_HTTP
else if (m_mode == MODE_DAEMON) {
    # 创建一个DaemonClient对象
    client = new DaemonClient(id, listener);
}
else if (m_mode == MODE_SELF_SELECT) {
    # 创建一个SelfSelectClient对象
    client = new SelfSelectClient(id, Platform::userAgent(), listener, m_submitToOrigin);
}

# 如果定义了XMRIG_ALGO_KAWPOW或者XMRIG_ALGO_GHOSTRIDER
else if (m_mode == MODE_AUTO_ETH) {
    # 创建一个AutoClient对象
    client = new AutoClient(id, Platform::userAgent(), listener);
}

# 如果定义了XMRIG_FEATURE_BENCHMARK
else if (m_mode == MODE_BENCHMARK) {
    # 创建一个BenchClient对象
    client = new BenchClient(m_benchmark, listener);
}

# 断言client不为空
assert(client != nullptr);

# 如果client不为空
if (client) {
    # 设置client的池
    client->setPool(*this);
}

# 返回client对象
return client;
}


rapidjson::Value xmrig::Pool::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    auto &allocator = doc.GetAllocator();

    # 创建一个空的JSON对象
    Value obj(kObjectType);

    # 向JSON对象中添加成员
    obj.AddMember(StringRef(kAlgo),  m_algorithm.toJSON(), allocator);
    obj.AddMember(StringRef(kCoin),  m_coin.toJSON(), allocator);
    obj.AddMember(StringRef(kUrl),   url().toJSON(), allocator);
    obj.AddMember(StringRef(kUser),  m_user.toJSON(), allocator);

    # 如果spendSecretKey不为空
    if (!m_spendSecretKey.isEmpty()) {
        # 向JSON对象中添加成员
        obj.AddMember(StringRef(kSpendSecretKey), m_spendSecretKey.toJSON(), allocator);
    }

    # 如果模式不是MODE_DAEMON
    if (m_mode != MODE_DAEMON) {
        # 向JSON对象中添加成员
        obj.AddMember(StringRef(kPass),  m_password.toJSON(), allocator);
        obj.AddMember(StringRef(kRigId), m_rigId.toJSON(), allocator);
// 如果不是 XMRIG_PROXY_PROJECT，则向对象添加名为 kNicehash 的成员，其值为 isNicehash() 的结果
        obj.AddMember(StringRef(kNicehash), isNicehash(), allocator);

// 如果 m_keepAlive 为 0 或者 kKeepAliveTimeout，则向对象添加名为 kKeepalive 的成员，其值为 m_keepAlive 是否大于 0
        if (m_keepAlive == 0 || m_keepAlive == kKeepAliveTimeout) {
            obj.AddMember(StringRef(kKeepalive), m_keepAlive > 0, allocator);
        }
// 否则，向对象添加名为 kKeepalive 的成员，其值为 m_keepAlive
        else {
            obj.AddMember(StringRef(kKeepalive), m_keepAlive, allocator);
        }
    }

// 向对象添加名为 kEnabled 的成员，其值为 m_flags 中是否包含 FLAG_ENABLED
    obj.AddMember(StringRef(kEnabled),      m_flags.test(FLAG_ENABLED), allocator);
// 向对象添加名为 kTls 的成员，其值为 isTLS() 的结果
    obj.AddMember(StringRef(kTls),          isTLS(), allocator);
// 向对象添加名为 kSni 的成员，其值为 isSNI() 的结果
    obj.AddMember(StringRef(kSni),          isSNI(), allocator);
// 向对象添加名为 kFingerprint 的成员，其值为 m_fingerprint.toJSON() 的结果
    obj.AddMember(StringRef(kFingerprint),  m_fingerprint.toJSON(), allocator);
// 向对象添加名为 kDaemon 的成员，其值为 m_mode 是否等于 MODE_DAEMON
    obj.AddMember(StringRef(kDaemon),       m_mode == MODE_DAEMON, allocator);
// 向对象添加名为 kSOCKS5 的成员，其值为 m_proxy.toJSON(doc) 的结果
    obj.AddMember(StringRef(kSOCKS5),       m_proxy.toJSON(doc), allocator);

// 如果 m_mode 等于 MODE_DAEMON，则向对象添加名为 kDaemonPollInterval、kDaemonJobTimeout 和 kDaemonZMQPort 的成员，分别为 m_pollInterval、m_jobTimeout 和 m_zmqPort
    if (m_mode == MODE_DAEMON) {
        obj.AddMember(StringRef(kDaemonPollInterval), m_pollInterval, allocator);
        obj.AddMember(StringRef(kDaemonJobTimeout), m_jobTimeout, allocator);
        obj.AddMember(StringRef(kDaemonZMQPort), m_zmqPort, allocator);
    }
// 否则，向对象添加名为 kSelfSelect 和 kSubmitToOrigin 的成员，分别为 m_daemon.url().toJSON() 和 m_submitToOrigin
    else {
        obj.AddMember(StringRef(kSelfSelect),     m_daemon.url().toJSON(), allocator);
        obj.AddMember(StringRef(kSubmitToOrigin), m_submitToOrigin, allocator);
    }

// 返回对象
    return obj;
}

// 返回可打印的池名称
std::string xmrig::Pool::printableName() const
{
// 创建字符串 out，包含 ANSI 转义码和池的 URL
    std::string out(CSI "1;" + std::to_string(isEnabled() ? (isTLS() ? 32 : 36) : 31) + "m" + url().data() + CLEAR);

// 如果 m_coin 有效，则添加 coin 和 m_coin.name() 到 out，否则添加 algo 和 m_algorithm.name() 或 "auto" 到 out
    if (m_coin.isValid()) {
        out += std::string(" coin ") + WHITE_BOLD_S + m_coin.name() + CLEAR;
    }
    else {
        out += std::string(" algo ") + WHITE_BOLD_S + (m_algorithm.isValid() ? m_algorithm.name() : "auto") + CLEAR;
    }

// 如果 m_mode 等于 MODE_SELF_SELECT，则添加 self-select 和 m_daemon.url() 到 out，以及 m_submitToOrigin 是否为真
    if (m_mode == MODE_SELF_SELECT) {
        out += std::string(" self-select ") + CSI "1;" + std::to_string(m_daemon.isTLS() ? 32 : 36) + "m" + m_daemon.url().data() + WHITE_BOLD_S + (m_submitToOrigin ? " submit-to-origin" : "") + CLEAR;
    }

// 返回 out
    return out;
}
#ifdef APP_DEBUG
// 如果是调试模式，打印连接池的相关信息
void xmrig::Pool::print() const
{
    // 打印连接池的 URL
    LOG_NOTICE("url:       %s", url().data());
    // 打印连接池的主机名
    LOG_DEBUG ("host:      %s", host().data());
    // 打印连接池的端口号
    LOG_DEBUG ("port:      %d", static_cast<int>(port()));
    // 如果有 ZMQ 端口，打印 ZMQ 端口号
    if (m_zmqPort >= 0) {
        LOG_DEBUG("zmq-port:  %d", m_zmqPort);
    }
    // 打印连接池的用户名
    LOG_DEBUG ("user:      %s", m_user.data());
    // 打印连接池的密码
    LOG_DEBUG ("pass:      %s", m_password.data());
    // 打印连接池的矿工 ID
    LOG_DEBUG ("rig-id     %s", m_rigId.data());
    // 打印连接池使用的算法
    LOG_DEBUG ("algo:      %s", m_algorithm.name());
    // 打印是否为 NiceHash 模式
    LOG_DEBUG ("nicehash:  %d", static_cast<int>(m_flags.test(FLAG_NICEHASH)));
    // 打印连接池的 keepAlive 参数
    LOG_DEBUG ("keepAlive: %d", m_keepAlive);
}
#endif

// 设置连接池的 keepAlive 参数
void xmrig::Pool::setKeepAlive(const rapidjson::Value &value)
{
    // 如果传入的值是整数，设置 keepAlive 参数
    if (value.IsInt()) {
        setKeepAlive(value.GetInt());
    }
    // 如果传入的值是布尔值，设置 keepAlive 参数
    else if (value.IsBool()) {
        setKeepAlive(value.GetBool());
    }
}
```