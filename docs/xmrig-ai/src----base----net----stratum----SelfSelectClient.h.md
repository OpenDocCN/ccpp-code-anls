# `xmrig\src\base\net\stratum\SelfSelectClient.h`

```cpp
/* XMRig
 * 版权所有（c）2019       jtgrassie       <https://github.com/jtgrassie>
 * 版权所有（c）2021       Hansie Odendaal <https://github.com/hansieodendaal>
 * 版权所有（c）2018-2021  SChernykh       <https://github.com/SChernykh>
 * 版权所有（c）2016-2021  XMRig           <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SELFSELECTCLIENT_H
#define XMRIG_SELFSELECTCLIENT_H


#include "base/kernel/interfaces/IClient.h"
#include "base/kernel/interfaces/IClientListener.h"
#include "base/net/http/HttpListener.h"
#include "base/net/stratum/Job.h"


#include <map>
#include <memory>


namespace xmrig {


class SelfSelectClient : public IClient, public IClientListener, public IHttpListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(SelfSelectClient)

    SelfSelectClient(int id, const char *agent, IClientListener *listener, bool submitToOrigin);
    ~SelfSelectClient() override;

protected:
    // IClient
    // 断开连接
    inline bool disconnect() override                                               { return m_client->disconnect(); }
    // 检查是否具有扩展
    inline bool hasExtension(Extension extension) const noexcept override           { return m_client->hasExtension(extension); }
    // 检查是否启用
    inline bool isEnabled() const override                                          { return m_client->isEnabled(); }
    // 检查客户端是否使用了 TLS
    inline bool isTLS() const override                                              { return m_client->isTLS(); }
    // 返回客户端的模式
    inline const char *mode() const override                                        { return m_client->mode(); }
    // 返回客户端的标签
    inline const char *tag() const override                                         { return m_client->tag(); }
    // 返回客户端的 TLS 指纹
    inline const char *tlsFingerprint() const override                              { return m_client->tlsFingerprint(); }
    // 返回客户端的 TLS 版本
    inline const char *tlsVersion() const override                                  { return m_client->tlsVersion(); }
    // 返回当前任务
    inline const Job &job() const override                                          { return m_job; }
    // 返回客户端所在的连接池
    inline const Pool &pool() const override                                        { return m_client->pool(); }
    // 返回客户端的 IP 地址
    inline const String &ip() const override                                        { return m_client->ip(); }
    // 返回客户端的 ID
    inline int id() const override                                                  { return m_client->id(); }
    // 发送数据给客户端，并指定回调函数
    inline int64_t send(const rapidjson::Value &obj, Callback callback) override    { return m_client->send(obj, callback); }
    // 发送数据给客户端
    inline int64_t send(const rapidjson::Value &obj) override                       { return m_client->send(obj); }
    // 返回客户端的序列号
    inline int64_t sequence() const override                                        { return m_client->sequence(); }
    // 连接客户端
    inline void connect() override                                                  { m_client->connect(); }
    // 连接客户端到指定的连接池
    inline void connect(const Pool &pool) override                                  { m_client->connect(pool); }
    // 延迟删除客户端
    inline void deleteLater() override                                              { m_client->deleteLater(); }
    // 设置客户端的算法
    inline void setAlgo(const Algorithm &algo) override                             { m_client->setAlgo(algo); }
    // 启用或禁用客户端
    inline void setEnabled(bool enabled) override                                   { m_client->setEnabled(enabled); }
    // 设置连接池
    inline void setPool(const Pool &pool) override                                  { m_client->setPool(pool); }
    // 设置代理
    inline void setProxy(const ProxyUrl &proxy) override                            { m_client->setProxy(proxy); }
    // 设置是否安静模式
    inline void setQuiet(bool quiet) override                                       { m_client->setQuiet(quiet); m_quiet = quiet;  }
    // 设置重试次数
    inline void setRetries(int retries) override                                    { m_client->setRetries(retries); m_retries = retries; }
    // 设置重试暂停时间
    inline void setRetryPause(uint64_t ms) override                                 { m_client->setRetryPause(ms); m_retryPause = ms; }

    // 提交作业结果
    int64_t submit(const JobResult &result) override;
    // 定时器触发
    void tick(uint64_t now) override;

    // IClientListener
    // 关闭连接回调
    inline void onClose(IClient *, int failures) override                                           { m_listener->onClose(this, failures); setState(IdleState); m_active = false; }
    // 登录成功回调
    inline void onLoginSuccess(IClient *) override                                                  { m_listener->onLoginSuccess(this); setState(IdleState); m_active = true; }
    // 接受结果回调
    inline void onResultAccepted(IClient *, const SubmitResult &result, const char *error) override { m_listener->onResultAccepted(this, result, error); }
    // 验证算法回调
    inline void onVerifyAlgorithm(const IClient *, const Algorithm &algorithm, bool *ok) override   { m_listener->onVerifyAlgorithm(this, algorithm, ok); }

    // 收到作业回调
    void onJobReceived(IClient *, const Job &job, const rapidjson::Value &params) override;
    // 登录回调
    void onLogin(IClient *, rapidjson::Document &doc, rapidjson::Value &params) override;

    // IHttpListener
    // HTTP 数据回调
    void onHttpData(const HttpData &data) override;
// 定义私有成员变量
private:
    // 定义枚举类型 State，并初始化为 IdleState
    enum State {
        IdleState,
        WaitState,
        RetryState
    };

    // 定义内联函数 isQuiet，用于判断是否为静默状态
    inline bool isQuiet() const { return m_quiet || m_failures >= m_retries; }

    // 声明私有成员函数 parseResponse，用于解析响应
    bool parseResponse(int64_t id, rapidjson::Value &result, const rapidjson::Value &error);
    // 声明私有成员函数 getBlockTemplate，用于获取区块模板
    void getBlockTemplate();
    // 声明私有成员函数 retry，用于重试
    void retry();
    // 声明私有成员函数 setState，用于设置状态
    void setState(State state);
    // 声明私有成员函数 submitBlockTemplate，用于提交区块模板
    void submitBlockTemplate(rapidjson::Value &result);
    // 声明私有成员函数 submitOriginDaemon，用于提交原始守护程序
    void submitOriginDaemon(const JobResult &result);

    // 定义私有成员变量
    bool m_active                   = false;
    bool m_quiet                    = false;
    const bool m_submitToOrigin;
    IClient *m_client;
    IClientListener *m_listener;
    int m_retries                   = 5;
    int64_t m_failures              = 0;
    int64_t m_sequence              = 1;
    Job m_job;
    State m_state                   = IdleState;
    std::map<int64_t, SubmitResult> m_results;
    std::shared_ptr<IHttpListener> m_httpListener;
    String m_blocktemplate;
    uint64_t m_blockDiff            = 0;
    uint64_t m_originNotSubmitted   = 0;
    uint64_t m_originSubmitted      = 0;
    uint64_t m_retryPause           = 5000;
    uint64_t m_timestamp            = 0;
};


} /* namespace xmrig */

// 结束定义自选客户端类
#endif /* XMRIG_SELFSELECTCLIENT_H */
```