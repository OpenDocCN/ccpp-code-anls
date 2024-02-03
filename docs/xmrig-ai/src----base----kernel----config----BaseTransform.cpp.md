# `xmrig\src\base\kernel\config\BaseTransform.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cstdio>


#ifdef _MSC_VER
#   include "getopt/getopt.h"
#else
#   include <getopt.h>
#endif


#include "base/kernel/config/BaseTransform.h"
#include "base/io/json/JsonChain.h"
#include "base/io/log/Log.h"
#include "base/kernel/config/BaseConfig.h"
#include "base/kernel/interfaces/IConfig.h"
#include "base/kernel/Process.h"
#include "base/net/dns/DnsConfig.h"
#include "base/net/stratum/Pool.h"
#include "base/net/stratum/Pools.h"
#include "core/config/Config_platform.h"


#ifdef XMRIG_FEATURE_TLS
#   include "base/net/tls/TlsConfig.h"
#endif


void xmrig::BaseTransform::load(JsonChain &chain, Process *process, IConfigTransform &transform)
{
    using namespace rapidjson;

    // 初始化变量
    int key     = 0;
    int argc    = process->arguments().argc();
    char **argv = process->arguments().argv();

    // 创建一个空的 JSON 文档对象
    Document doc(kObjectType);
    # 当条件为真时执行循环
    while (true) {
        # 使用 getopt_long 函数获取命令行参数
        key = getopt_long(argc, argv, short_options, options, nullptr); // NOLINT(concurrency-mt-unsafe)
        # 如果 key 小于 0，跳出循环
        if (key < 0) {
            break;
        }

        # 如果 key 等于 IConfig::ConfigKey
        if (key == IConfig::ConfigKey) {
            # 将 doc 移动到 chain 中
            chain.add(std::move(doc));
            # 将文件路径添加到 chain 中
            chain.addFile(optarg);
            # 创建一个新的空文档对象
            doc = Document(kObjectType);
        }
        # 否则
        else {
            # 使用 transform 对象对 doc 进行转换
            transform.transform(doc, key, optarg);
        }
    }

    # 如果还有未处理的非选项参数
    if (optind < argc) {
        # 输出警告信息
        LOG_WARN("%s: unsupported non-option argument '%s'", argv[0], argv[optind]);
    }

    # 对 doc 进行最终处理
    transform.finalize(doc);
    # 将 doc 移动到 chain 中
    chain.add(std::move(doc));
// 定义一个名为 finalize 的函数，参数为一个 rapidjson 文档的引用
void xmrig::BaseTransform::finalize(rapidjson::Document &doc)
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取文档的分配器
    auto &allocator = doc.GetAllocator();

    // 如果算法有效并且文档包含 Pools::kPools 成员
    if (m_algorithm.isValid() && doc.HasMember(Pools::kPools)) {
        // 获取 Pools::kPools 成员的引用
        auto &pools = doc[Pools::kPools];
        // 遍历 pools 数组中的每个元素
        for (Value &pool : pools.GetArray()) {
            // 如果 pool 没有 Pool::kAlgo 成员
            if (!pool.HasMember(Pool::kAlgo)) {
                // 添加 Pool::kAlgo 成员，值为 m_algorithm 的 JSON 表示，使用分配器 allocator
                pool.AddMember(StringRef(Pool::kAlgo), m_algorithm.toJSON(), allocator);
            }
        }
    }

    // 如果币种有效并且文档包含 Pools::kPools 成员
    if (m_coin.isValid() && doc.HasMember(Pools::kPools)) {
        // 获取 Pools::kPools 成员的引用
        auto &pools = doc[Pools::kPools];
        // 遍历 pools 数组中的每个元素
        for (Value &pool : pools.GetArray()) {
            // 如果 pool 没有 Pool::kCoin 成员
            if (!pool.HasMember(Pool::kCoin)) {
                // 添加 Pool::kCoin 成员，值为 m_coin 的 JSON 表示，使用分配器 allocator
                pool.AddMember(StringRef(Pool::kCoin), m_coin.toJSON(), allocator);
            }
        }
    }

    // 如果 m_http 为真
    if (m_http) {
        // 设置文档中 BaseConfig::kHttp 成员的值为 true
        set(doc, BaseConfig::kHttp, Http::kEnabled, true);
    }
}

// 定义一个名为 transform 的函数，参数为一个 rapidjson 文档的引用，一个整数 key，一个指向字符的指针 arg
void xmrig::BaseTransform::transform(rapidjson::Document &doc, int key, const char *arg)
{
    // 根据 key 的值进行不同的操作
    switch (key) {
    case IConfig::AlgorithmKey: /* --algo */
        // 如果文档中不包含 Pools::kPools 成员
        if (!doc.HasMember(Pools::kPools)) {
            // 将 m_algorithm 的值设置为 arg
            m_algorithm = arg;
        }
        else {
            // 调用 add 函数，向文档中的 Pools::kPools 成员添加 Pool::kAlgo 成员，值为 arg
            return add(doc, Pools::kPools, Pool::kAlgo, arg);
        }
        break;

    case IConfig::CoinKey: /* --coin */
        // 如果文档中不包含 Pools::kPools 成员
        if (!doc.HasMember(Pools::kPools)) {
            // 将 m_coin 的值设置为 arg
            m_coin = arg;
        }
        else {
            // 调用 add 函数，向文档中的 Pools::kPools 成员添加 Pool::kCoin 成员，值为 arg
            return add(doc, Pools::kPools, Pool::kCoin, arg);
        }
        break;

    case IConfig::UserpassKey: /* --userpass */
        {
            // 在 arg 中查找最后一个冒号的位置
            const char *p = strrchr(arg, ':');
            // 如果找不到冒号，直接返回
            if (!p) {
                return;
            }

            // 创建一个长度为 p - arg + 1 的字符数组 user，初始化为零
            char *user = new char[p - arg + 1]();
            // 将 arg 中的内容复制到 user 中
            strncpy(user, arg, static_cast<size_t>(p - arg));

            // 调用 add 函数，向文档中的 Pools::kPools 成员添加 Pool::kUser 成员，值为 user
            add<const char *>(doc, Pools::kPools, Pool::kUser, user);
            // 调用 add 函数，向文档中的 Pools::kPools 成员添加 Pool::kPass 成员，值为冒号后的内容
            add(doc, Pools::kPools, Pool::kPass, p + 1);
            // 释放 user 的内存
            delete [] user;
        }
        break;

    case IConfig::UrlKey:    /* --url */
    case IConfig::StressKey: /* --stress */
    # 如果文档中不存在 Pools::kPools 键，则添加一个空的数组
    if (!doc.HasMember(Pools::kPools)) {
        doc.AddMember(rapidjson::StringRef(Pools::kPools), rapidjson::kArrayType, doc.GetAllocator());
    }

    # 获取 Pools::kPools 对应的数组
    rapidjson::Value &array = doc[Pools::kPools];
    
    # 如果数组为空，或者最后一个元素是有效的池对象，则向数组中添加一个空的对象
    if (array.Size() == 0 || Pool(array[array.Size() - 1]).isValid()) {
        array.PushBack(rapidjson::kObjectType, doc.GetAllocator());
    }
# 如果定义了 XMRIG_FEATURE_BENCHMARK
        if (key != IConfig::UrlKey) {
            # 设置 URL
            set(doc, array[array.Size() - 1], Pool::kUrl,
#           ifdef XMRIG_FEATURE_TLS
                "stratum+ssl://randomx.xmrig.com:443"
#           else
                "randomx.xmrig.com:3333"
#           endif
            );
        } else
#       endif
        {
            # 设置 URL
            set(doc, array[array.Size() - 1], Pool::kUrl, arg);
        }
        # 结束当前的 case
        break;
    }

    case IConfig::UserKey: /* --user */
        # 添加用户信息
        return add(doc, Pools::kPools, Pool::kUser, arg);

    case IConfig::PasswordKey: /* --pass */
        # 添加密码信息
        return add(doc, Pools::kPools, Pool::kPass, arg);

    case IConfig::SpendSecretKey: /* --spend-secret-key */
        # 添加 Spend Secret Key 信息
        return add(doc, Pools::kPools, Pool::kSpendSecretKey, arg);

    case IConfig::RigIdKey: /* --rig-id */
        # 添加 Rig ID 信息
        return add(doc, Pools::kPools, Pool::kRigId, arg);

    case IConfig::FingerprintKey: /* --tls-fingerprint */
        # 添加 TLS 指纹信息
        return add(doc, Pools::kPools, Pool::kFingerprint, arg);

    case IConfig::SelfSelectKey: /* --self-select */
        # 添加自选信息
        return add(doc, Pools::kPools, Pool::kSelfSelect, arg);

    case IConfig::ProxyKey: /* --proxy */
        # 添加代理信息
        return add(doc, Pools::kPools, Pool::kSOCKS5, arg);

    case IConfig::LogFileKey: /* --log-file */
        # 设置日志文件路径
        return set(doc, BaseConfig::kLogFile, arg);

    case IConfig::HttpAccessTokenKey: /* --http-access-token */
        # 设置 HTTP 访问令牌
        m_http = true;
        return set(doc, BaseConfig::kHttp, Http::kToken, arg);

    case IConfig::HttpHostKey: /* --http-host */
        # 设置 HTTP 主机
        m_http = true;
        return set(doc, BaseConfig::kHttp, Http::kHost, arg);

    case IConfig::ApiWorkerIdKey: /* --api-worker-id */
        # 设置 API Worker ID
        return set(doc, BaseConfig::kApi, BaseConfig::kApiWorkerId, arg);

    case IConfig::ApiIdKey: /* --api-id */
        # 设置 API ID
        return set(doc, BaseConfig::kApi, BaseConfig::kApiId, arg);

    case IConfig::UserAgentKey: /* --user-agent */
        # 设置用户代理
        return set(doc, BaseConfig::kUserAgent, arg);
    # 如果参数是标题，则设置文档的标题
    case IConfig::TitleKey: /* --title */
        return set(doc, BaseConfig::kTitle, arg);
# 如果启用了 XMRIG_FEATURE_TLS，则执行以下操作

# 处理 --tls-cert 参数，设置证书路径
case IConfig::TlsCertKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kCert, arg);

# 处理 --tls-cert-key 参数，设置证书密钥路径
case IConfig::TlsCertKeyKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kCertKey, arg);

# 处理 --tls-dhparam 参数，设置 DH 参数路径
case IConfig::TlsDHparamKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kDhparam, arg);

# 处理 --tls-ciphers 参数，设置加密算法
case IConfig::TlsCiphersKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kCiphers, arg);

# 处理 --tls-ciphersuites 参数，设置密码套件
case IConfig::TlsCipherSuitesKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kCipherSuites, arg);

# 处理 --tls-protocols 参数，设置协议
case IConfig::TlsProtocolsKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kProtocols, arg);

# 处理 --tls-gen 参数，设置生成证书和密钥
case IConfig::TlsGenKey: 
    return set(doc, BaseConfig::kTls, TlsConfig::kGen, arg);
# 如果未启用 XMRIG_FEATURE_TLS，则执行以下操作

# 处理 --retries、--retry-pause、--print-time、--http-port、--donate-level、--daemon-poll-interval、--daemon-job-timeout、--dns-ttl、--daemon-zmq-port 参数，将参数转换为无符号64位整数
case IConfig::RetriesKey:       
case IConfig::RetryPauseKey:    
case IConfig::PrintTimeKey:     
case IConfig::HttpPort:         
case IConfig::DonateLevelKey:   
case IConfig::DaemonPollKey:    
case IConfig::DaemonJobTimeoutKey: 
case IConfig::DnsTtlKey:        
case IConfig::DaemonZMQPortKey: 
    return transformUint64(doc, key, static_cast<uint64_t>(strtol(arg, nullptr, 10)));

# 处理 --background、--syslog、--keepalive、--nicehash、--tls、--dry-run、--http-enabled、--daemon、--submit-to-origin 参数，将参数转换为布尔值
case IConfig::BackgroundKey:  
case IConfig::SyslogKey:      
case IConfig::KeepAliveKey:   
case IConfig::NicehashKey:    
case IConfig::TlsKey:         
case IConfig::DryRunKey:      
case IConfig::HttpEnabledKey: 
case IConfig::DaemonKey:      
case IConfig::SubmitToOriginKey: 
    # 如果 key 是 --verbose 或 --dns-ipv6，则返回对应的布尔值
    case IConfig::VerboseKey:     /* --verbose */
    case IConfig::DnsIPv6Key:     /* --dns-ipv6 */
        return transformBoolean(doc, key, true);

    # 如果 key 是 --no-color、--http-no-restricted 或 --no-title，则返回对应的布尔值
    case IConfig::ColorKey:          /* --no-color */
    case IConfig::HttpRestrictedKey: /* --http-no-restricted */
    case IConfig::NoTitleKey:        /* --no-title */
        return transformBoolean(doc, key, false);

    # 默认情况下不返回任何值
    default:
        break;
    }
    // 根据传入的 key 值进行不同的布尔类型参数转换操作
    void xmrig::BaseTransform::transformBoolean(rapidjson::Document &doc, int key, bool enable)
    {
        // 根据 key 值进行不同的操作
        switch (key) {
        case IConfig::BackgroundKey: /* --background */
            // 设置 doc 中的 kBackground 字段为 enable
            return set(doc, BaseConfig::kBackground, enable);

        case IConfig::SyslogKey: /* --syslog */
            // 设置 doc 中的 kSyslog 字段为 enable
            return set(doc, BaseConfig::kSyslog, enable);

        case IConfig::KeepAliveKey: /* --keepalive */
            // 将 enable 添加到 doc 中的 Pools::kPools, Pool::kKeepalive 字段
            return add(doc, Pools::kPools, Pool::kKeepalive, enable);

        case IConfig::TlsKey: /* --tls */
            // 将 enable 添加到 doc 中的 Pools::kPools, Pool::kTls 字段
            return add(doc, Pools::kPools, Pool::kTls, enable);

        case IConfig::SubmitToOriginKey: /* --submit-to-origin */
            // 将 enable 添加到 doc 中的 Pools::kPools, Pool::kSubmitToOrigin 字段
            return add(doc, Pools::kPools, Pool::kSubmitToOrigin, enable);
    #   ifdef XMRIG_FEATURE_HTTP
        case IConfig::DaemonKey: /* --daemon */
            // 将 enable 添加到 doc 中的 Pools::kPools, Pool::kDaemon 字段
            return add(doc, Pools::kPools, Pool::kDaemon, enable);
    #   endif

    #   ifndef XMRIG_PROXY_PROJECT
        case IConfig::NicehashKey: /* --nicehash */
            // 将 enable 添加到 doc 中的 Pools::kPools, Pool::kNicehash 字段
            return add<bool>(doc, Pools::kPools, Pool::kNicehash, enable);
    #   endif

        case IConfig::ColorKey: /* --no-color */
            // 设置 doc 中的 kColors 字段为 enable
            return set(doc, BaseConfig::kColors, enable);

        case IConfig::HttpRestrictedKey: /* --http-no-restricted */
            // 设置 m_http 为 true，然后设置 doc 中的 kHttp, Http::kRestricted 字段为 enable
            m_http = true;
            return set(doc, BaseConfig::kHttp, Http::kRestricted, enable);

        case IConfig::HttpEnabledKey: /* --http-enabled */
            // 设置 m_http 为 true
            m_http = true;
            break;

        case IConfig::DryRunKey: /* --dry-run */
            // 设置 doc 中的 kDryRun 字段为 enable
            return set(doc, BaseConfig::kDryRun, enable);

        case IConfig::VerboseKey: /* --verbose */
            // 设置 doc 中的 kVerbose 字段为 enable
            return set(doc, BaseConfig::kVerbose, enable);

        case IConfig::NoTitleKey: /* --no-title */
            // 设置 doc 中的 kTitle 字段为 enable
            return set(doc, BaseConfig::kTitle, enable);

        case IConfig::DnsIPv6Key: /* --dns-ipv6 */
            // 设置 doc 中的 DnsConfig::kField, DnsConfig::kIPv6 字段为 enable
            return set(doc, DnsConfig::kField, DnsConfig::kIPv6, enable);

        default:
            break;
        }
    }

    // 根据传入的 key 值进行不同的 64 位无符号整数参数转换操作
    void xmrig::BaseTransform::transformUint64(rapidjson::Document &doc, int key, uint64_t arg)
    {
        // 根据 key 值进行不同的操作
        switch (key) {
    # 处理配置中的重试次数参数，设置到对应的配置文档中
    case IConfig::RetriesKey: /* --retries */
        return set(doc, Pools::kRetries, arg);

    # 处理配置中的重试暂停时间参数，设置到对应的配置文档中
    case IConfig::RetryPauseKey: /* --retry-pause */
        return set(doc, Pools::kRetryPause, arg);

    # 处理配置中的捐赠级别参数，设置到对应的配置文档中
    case IConfig::DonateLevelKey: /* --donate-level */
        return set(doc, Pools::kDonateLevel, arg);

    # 处理配置中的代理捐赠参数，设置到对应的配置文档中
    case IConfig::ProxyDonateKey: /* --donate-over-proxy */
        return set(doc, Pools::kDonateOverProxy, arg);

    # 处理配置中的 HTTP 端口参数，设置到对应的配置文档中，并标记启用了 HTTP
    case IConfig::HttpPort: /* --http-port */
        m_http = true;
        return set(doc, BaseConfig::kHttp, Http::kPort, arg);

    # 处理配置中的打印时间参数，设置到对应的配置文档中
    case IConfig::PrintTimeKey: /* --print-time */
        return set(doc, BaseConfig::kPrintTime, arg);

    # 处理配置中的 DNS 缓存时间参数，设置到对应的 DNS 配置文档中
    case IConfig::DnsTtlKey: /* --dns-ttl */
        return set(doc, DnsConfig::kField, DnsConfig::kTTL, arg);
# 如果定义了 XMRIG_FEATURE_HTTP，则执行以下操作
    # 当参数为 --daemon-poll-interval 时，将其添加到配置文档中
    case IConfig::DaemonPollKey:  
        return add(doc, Pools::kPools, Pool::kDaemonPollInterval, arg);

    # 当参数为 --daemon-job-timeout 时，将其添加到配置文档中
    case IConfig::DaemonJobTimeoutKey:  
        return add(doc, Pools::kPools, Pool::kDaemonJobTimeout, arg);

    # 当参数为 --daemon-zmq-port 时，将其添加到配置文档中
    case IConfig::DaemonZMQPortKey:  
        return add(doc, Pools::kPools, Pool::kDaemonZMQPort, arg);
# 如果未定义 XMRIG_FEATURE_HTTP，则执行以下操作
    default:
        break;
    }
}
```