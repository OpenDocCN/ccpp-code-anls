# `xmrig\src\base\kernel\interfaces\IClient.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ICLIENT_H
#define XMRIG_ICLIENT_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Object.h"


#include <functional>


namespace xmrig {


class Algorithm;
class Job;
class JobResult;
class Pool;
class ProxyUrl;
class String;


class IClient
{
public:
    XMRIG_DISABLE_COPY_MOVE(IClient)

    enum Extension {
        EXT_ALGO,  // 算法扩展
        EXT_NICEHASH,  // NiceHash扩展
        EXT_CONNECT,  // 连接扩展
        EXT_TLS,  // TLS扩展
        EXT_KEEPALIVE,  // 保持连接扩展
        EXT_MAX  // 最大扩展数
    };

    using Callback = std::function<void(const rapidjson::Value &result, bool success, uint64_t elapsed)>;

    IClient()           = default;
    virtual ~IClient()  = default;

    virtual bool disconnect()                                               = 0;  // 断开连接
    virtual bool hasExtension(Extension extension) const noexcept           = 0;  // 检查是否具有特定扩展
    virtual bool isEnabled() const                                          = 0;  // 检查是否启用
    virtual bool isTLS() const                                              = 0;  // 检查是否使用TLS
    virtual const char *mode() const                                        = 0;  // 获取模式
    virtual const char *tag() const                                         = 0;  // 获取标签
    # 返回 TLS 指纹的字符串常量
    virtual const char *tlsFingerprint() const                              = 0;
    # 返回 TLS 版本的字符串常量
    virtual const char *tlsVersion() const                                  = 0;
    # 返回作业的常量引用
    virtual const Job &job() const                                          = 0;
    # 返回连接池的常量引用
    virtual const Pool &pool() const                                        = 0;
    # 返回 IP 地址的常量引用
    virtual const String &ip() const                                        = 0;
    # 返回连接的 ID
    virtual int id() const                                                  = 0;
    # 发送 JSON 对象并指定回调函数，返回发送的消息 ID
    virtual int64_t send(const rapidjson::Value &obj, Callback callback)    = 0;
    # 发送 JSON 对象，返回发送的消息 ID
    virtual int64_t send(const rapidjson::Value &obj)                       = 0;
    # 返回连接的序列号
    virtual int64_t sequence() const                                        = 0;
    # 提交作业结果，返回提交的消息 ID
    virtual int64_t submit(const JobResult &result)                         = 0;
    # 连接到服务器
    virtual void connect()                                                  = 0;
    # 连接到指定连接池
    virtual void connect(const Pool &pool)                                  = 0;
    # 延迟删除对象
    virtual void deleteLater()                                              = 0;
    # 设置算法
    virtual void setAlgo(const Algorithm &algo)                             = 0;
    # 设置是否启用
    virtual void setEnabled(bool enabled)                                   = 0;
    # 设置连接池
    virtual void setPool(const Pool &pool)                                  = 0;
    # 设置代理
    virtual void setProxy(const ProxyUrl &proxy)                            = 0;
    # 设置是否安静模式
    virtual void setQuiet(bool quiet)                                       = 0;
    # 设置重试次数
    virtual void setRetries(int retries)                                    = 0;
    # 设置重试暂停时间
    virtual void setRetryPause(uint64_t ms)                                 = 0;
    # 执行定时操作
    virtual void tick(uint64_t now)                                         = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明代码结束了 xmrig 命名空间的定义

#endif // XMRIG_ICLIENT_H
// 结束了 XMRIG_ICLIENT_H 的条件编译指令
```