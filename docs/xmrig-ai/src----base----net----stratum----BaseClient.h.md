# `xmrig\src\base\net\stratum\BaseClient.h`

```cpp
/* XMRig
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   它，无论是许可证的第3版，还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是在希望它有用的情况下分发的
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BASECLIENT_H
#define XMRIG_BASECLIENT_H


#include <map>


#include "base/kernel/interfaces/IClient.h"
#include "base/net/stratum/Job.h"
#include "base/net/stratum/Pool.h"
#include "base/tools/Chrono.h"


namespace xmrig {


class IClientListener;
class SubmitResult;


class BaseClient : public IClient
{
public:
    // 构造函数，初始化基本客户端
    BaseClient(int id, IClientListener *listener);

protected:
    // 检查客户端是否启用
    inline bool isEnabled() const override                     { return m_enabled; }
    // 返回客户端标签
    inline const char *tag() const override                    { return m_tag.c_str(); }
    // 返回当前工作
    inline const Job &job() const override                     { return m_job; }
    // 返回连接的矿池
    inline const Pool &pool() const override                   { return m_pool; }
    // 返回客户端IP地址
    inline const String &ip() const override                   { return m_ip; }
    // 返回客户端ID
    inline int id() const override                             { return m_id; }
    // 返回序列号
    inline int64_t sequence() const override                   { return m_sequence; }
    // 设置算法
    inline void setAlgo(const Algorithm &algo) override        { m_pool.setAlgo(algo); }
    // 设置客户端是否启用
    inline void setEnabled(bool enabled) override              { m_enabled = enabled; }
    # 设置代理服务器的地址和端口
    inline void setProxy(const ProxyUrl &proxy) override       { m_pool.setProxy(proxy); }
    # 设置是否静默模式，即是否输出详细信息
    inline void setQuiet(bool quiet) override                  { m_quiet = quiet; }
    # 设置重试次数
    inline void setRetries(int retries) override               { m_retries = retries; }
    # 设置重试间隔时间，单位为毫秒
    inline void setRetryPause(uint64_t ms) override            { m_retryPause = ms; }
    
    # 设置连接池
    void setPool(const Pool &pool) override;
protected:
    // 定义枚举类型，表示套接字的不同状态
    enum SocketState {
        UnconnectedState,
        HostLookupState,
        ConnectingState,
        ConnectedState,
        ClosingState,
        ReconnectingState
    };

    // 定义发送结果结构体
    struct SendResult
    {
        // 构造函数，初始化回调函数和时间戳
        inline SendResult(Callback &&callback) : callback(callback), ts(Chrono::steadyMSecs()) {}

        Callback callback;  // 回调函数
        const uint64_t ts;  // 时间戳
    };

    // 判断是否为静默状态
    inline bool isQuiet() const { return m_quiet || m_failures >= m_retries; }

    // 处理响应的虚拟函数
    virtual bool handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error);
    // 处理提交响应的函数
    bool handleSubmitResponse(int64_t id, const char *error = nullptr);

    bool m_quiet                    = false;  // 是否为静默状态
    IClientListener *m_listener;  // 客户端监听器
    int m_id;  // ID
    int m_retries                   = 5;  // 重试次数
    int64_t m_failures              = 0;  // 失败次数
    Job m_job;  // 作业
    Pool m_pool;  // 矿池
    SocketState m_state             = UnconnectedState;  // 套接字状态
    std::map<int64_t, SendResult> m_callbacks;  // 发送结果回调映射
    std::map<int64_t, SubmitResult> m_results;  // 提交结果映射
    std::string m_tag;  // 标签
    String m_ip;  // IP 地址
    String m_password;  // 密码
    String m_rigId;  // 机器 ID
    String m_user;  // 用户名
    uint64_t m_retryPause           = 5000;  // 重试暂停时间

    static int64_t m_sequence;  // 静态成员变量，序列号

private:
    bool m_enabled = true;  // 是否启用
};


} /* namespace xmrig */


#endif /* XMRIG_BASECLIENT_H */
```