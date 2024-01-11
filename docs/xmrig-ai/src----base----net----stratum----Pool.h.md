# `xmrig\src\base\net\stratum\Pool.h`

```
// XMRig程序的版权声明和许可信息
/* XMRig
 * Copyright (c) 2019      Howard Chu  <https://github.com/hyc>
 * Copyright (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * Copyright (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 防止头文件重复包含
#ifndef XMRIG_POOL_H
#define XMRIG_POOL_H

// 包含必要的头文件
#include <bitset>
#include <vector>
#include <memory>
#include "3rdparty/rapidjson/fwd.h"
#include "base/crypto/Coin.h"
#include "base/net/stratum/ProxyUrl.h"

// 命名空间定义
namespace xmrig {

// 前置声明
class BenchConfig;
class IClient;
class IClientListener;

// 类定义
class Pool
{
public:
    // 枚举类型定义
    enum Mode {
        MODE_POOL,
        MODE_DAEMON,
        MODE_SELF_SELECT,
        MODE_AUTO_ETH,
#       ifdef XMRIG_FEATURE_BENCHMARK
        MODE_BENCHMARK,
#       endif
    };

    // 静态常量定义
    static const String kDefaultPassword;
    static const String kDefaultUser;

    static const char *kAlgo;
    static const char *kCoin;
    static const char *kDaemon;
    static const char *kDaemonPollInterval;
    static const char* kDaemonJobTimeout;
    static const char *kEnabled;
    static const char *kFingerprint;
    static const char *kKeepalive;
    static const char *kNicehash;
    static const char *kPass;
    static const char *kRigId;
    static const char *kSelfSelect;
    static const char *kSOCKS5;
    static const char *kSubmitToOrigin;
    static const char *kTls;
    // 声明静态常量字符串指针 kSni
    static const char* kSni;
    // 声明静态常量字符串指针 kUrl
    static const char *kUrl;
    // 声明静态常量字符串指针 kUser
    static const char *kUser;
    // 声明静态常量字符串指针 kSpendSecretKey
    static const char* kSpendSecretKey;
    // 声明静态常量字符串指针 kDaemonZMQPort
    static const char* kDaemonZMQPort;
    // 声明静态常量字符串指针 kNicehashHost
    static const char *kNicehashHost;

    // 声明静态常量整型变量 kKeepAliveTimeout 并初始化为 60
    constexpr static int kKeepAliveTimeout         = 60;
    // 声明静态常量 16 位无符号整型变量 kDefaultPort 并初始化为 3333
    constexpr static uint16_t kDefaultPort         = 3333;
    // 声明静态常量 64 位无符号整型变量 kDefaultPollInterval 并初始化为 1000
    constexpr static uint64_t kDefaultPollInterval = 1000;
    // 声明静态常量 64 位无符号整型变量 kDefaultJobTimeout 并初始化为 15000
    constexpr static uint64_t kDefaultJobTimeout   = 15000;

    // 默认构造函数
    Pool() = default;
    // 构造函数，接受主机名、端口、用户名、密码、花费密钥、保持连接时间、是否为 Nicehash、是否使用 TLS、模式作为参数
    Pool(const char *host, uint16_t port, const char *user, const char *password, const char* spendSecretKey, int keepAlive, bool nicehash, bool tls, Mode mode);
    // 构造函数，接受 URL 作为参数
    Pool(const char *url);
    // 构造函数，接受 rapidjson::Value 对象作为参数
    Pool(const rapidjson::Value &object);
#ifdef XMRIG_FEATURE_BENCHMARK
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则使用 benchmark 构造函数
    Pool(const std::shared_ptr<BenchConfig> &benchmark);

    // 返回 benchmark 对象的指针
    BenchConfig *benchmark() const;
    // 返回 benchmark 大小
    uint32_t benchSize() const;
#endif

    // 检查是否为 Nicehash 矿池
    inline bool isNicehash() const                      { return m_flags.test(FLAG_NICEHASH); }
    // 检查是否使用 TLS
    inline bool isTLS() const                           { return m_flags.test(FLAG_TLS) || m_url.isTLS(); }
    // 检查是否使用 SNI
    inline bool isSNI() const                           { return m_flags.test(FLAG_SNI); }
    // 检查 URL 是否有效
    inline bool isValid() const                         { return m_url.isValid(); }
    // 返回算法对象的引用
    inline const Algorithm &algorithm() const           { return m_algorithm; }
    // 返回货币对象的引用
    inline const Coin &coin() const                     { return m_coin; }
    // 返回代理 URL 对象的引用
    inline const ProxyUrl &proxy() const                { return m_proxy; }
    // 返回指纹字符串的引用
    inline const String &fingerprint() const            { return m_fingerprint; }
    // 返回主机名字符串的引用
    inline const String &host() const                   { return m_url.host(); }
    // 返回密码字符串的引用，如果为空则返回默认密码
    inline const String &password() const               { return !m_password.isNull() ? m_password : kDefaultPassword; }
    // 返回矿工 ID 字符串的引用
    inline const String &rigId() const                  { return m_rigId; }
    // 返回 URL 字符串的引用
    inline const String &url() const                    { return m_url.url(); }
    // 返回用户名字符串的引用，如果为空则返回默认用户名
    inline const String &user() const                   { return !m_user.isNull() ? m_user : kDefaultUser; }
    // 返回花费密钥字符串的引用
    inline const String &spendSecretKey() const         { return m_spendSecretKey; }
    // 返回守护进程 URL 对象的引用
    inline const Url &daemon() const                    { return m_daemon; }
    // 返回保持连接的时间
    inline int keepAlive() const                        { return m_keepAlive; }
    // 返回模式
    inline Mode mode() const                            { return m_mode; }
    // 返回端口号
    inline uint16_t port() const                        { return m_url.port(); }
    // 返回 ZeroMQ 端口号
    inline int zmq_port() const                         { return m_zmqPort; }
    // 返回轮询间隔
    inline uint64_t pollInterval() const                { return m_pollInterval; }
    // 返回作业超时时间
    inline uint64_t jobTimeout() const                  { return m_jobTimeout; }
    # 设置算法类型
    inline void setAlgo(const Algorithm &algorithm)     { m_algorithm = algorithm; }
    # 设置 URL
    inline void setUrl(const char *url)                 { m_url = Url(url); }
    # 设置密码
    inline void setPassword(const String &password)     { m_password = password; }
    # 设置代理
    inline void setProxy(const ProxyUrl &proxy)         { m_proxy = proxy; }
    # 设置矿工 ID
    inline void setRigId(const String &rigId)           { m_rigId = rigId; }
    # 设置用户
    inline void setUser(const String &user)             { m_user = user; }
    
    # 判断是否不等于另一个矿池对象
    inline bool operator!=(const Pool &other) const     { return !isEqual(other); }
    # 判断是否等于另一个矿池对象
    inline bool operator==(const Pool &other) const     { return isEqual(other); }
    
    # 判断矿池是否启用
    bool isEnabled() const;
    # 判断是否与另一个矿池对象相等
    bool isEqual(const Pool &other) const;
    # 创建客户端
    IClient *createClient(int id, IClientListener *listener) const;
    # 转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    # 获取可打印的矿池名称
    std::string printableName() const;
#ifdef APP_DEBUG
    // 如果是调试模式，定义一个打印函数
    void print() const;
#endif

private:
    // 定义一个枚举类型，表示各种标志位
    enum Flags {
        FLAG_ENABLED,
        FLAG_NICEHASH,
        FLAG_TLS,
        FLAG_SNI,
        FLAG_MAX
    };

    // 设置是否保持连接
    inline void setKeepAlive(bool enable)               { setKeepAlive(enable ? kKeepAliveTimeout : 0); }
    // 设置保持连接的时间
    inline void setKeepAlive(int keepAlive)             { m_keepAlive = keepAlive >= 0 ? keepAlive : 0; }

    // 设置保持连接的时间，根据 JSON 值
    void setKeepAlive(const rapidjson::Value &value);

    // 算法
    Algorithm m_algorithm;
    // 是否提交到原始地址
    bool m_submitToOrigin           = false;
    // 货币
    Coin m_coin;
    // 保持连接的时间
    int m_keepAlive                 = 0;
    // 模式
    Mode m_mode                     = MODE_POOL;
    // 代理地址
    ProxyUrl m_proxy;
    // 标志位
    std::bitset<FLAG_MAX> m_flags   = 0;
    // 指纹
    String m_fingerprint;
    // 密码
    String m_password;
    // 矿工 ID
    String m_rigId;
    // 用户名
    String m_user;
    // 花费密钥
    String m_spendSecretKey;
    // 轮询间隔
    uint64_t m_pollInterval         = kDefaultPollInterval;
    // 作业超时
    uint64_t m_jobTimeout           = kDefaultJobTimeout;
    // 守护进程地址
    Url m_daemon;
    // URL 地址
    Url m_url;
    // ZeroMQ 端口
    int m_zmqPort                   = -1;

#ifdef XMRIG_FEATURE_BENCHMARK
    // 基准配置
    std::shared_ptr<BenchConfig> m_benchmark;
#endif
};


} /* namespace xmrig */


#endif /* XMRIG_POOL_H */
```