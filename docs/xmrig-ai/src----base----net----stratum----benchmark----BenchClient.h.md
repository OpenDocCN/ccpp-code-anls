# `xmrig\src\base\net\stratum\benchmark\BenchClient.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或者（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BENCHCLIENT_H
#define XMRIG_BENCHCLIENT_H

#include "backend/common/interfaces/IBenchListener.h"
#include "base/kernel/interfaces/IDnsListener.h"
#include "base/kernel/interfaces/IHttpListener.h"
#include "base/net/stratum/Client.h"

namespace xmrig {

class BenchClient : public IClient, public IHttpListener, public IBenchListener, public IDnsListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(BenchClient)

    // BenchClient构造函数，接受benchmark配置和监听器作为参数
    BenchClient(const std::shared_ptr<BenchConfig> &benchmark, IClientListener* listener);
    // BenchClient析构函数
    ~BenchClient() override;

    // 断开连接的内联函数
    inline bool disconnect() override                                               { return true; }
    // 检查是否具有指定扩展的内联函数
    inline bool hasExtension(Extension) const noexcept override                     { return false; }
    // 检查是否启用的内联函数
    inline bool isEnabled() const override                                          { return true; }
    // 检查是否使用TLS的内联函数
    inline bool isTLS() const override                                              { return false; }
    // 返回模式的内联函数
    inline const char *mode() const override                                        { return "benchmark"; }
    // 返回空指针，表示不支持 TLS 指纹
    inline const char *tlsFingerprint() const override                              { return nullptr; }
    // 返回空指针，表示不支持 TLS 版本
    inline const char *tlsVersion() const override                                  { return nullptr; }
    // 返回当前任务对象的引用
    inline const Job &job() const override                                          { return m_job; }
    // 返回当前连接的矿池对象的引用
    inline const Pool &pool() const override                                        { return m_pool; }
    // 返回当前连接的 IP 地址
    inline const String &ip() const override                                        { return m_ip; }
    // 返回连接的 ID，这里返回固定值 0
    inline int id() const override                                                  { return 0; }
    // 发送数据到服务器，并指定回调函数
    inline int64_t send(const rapidjson::Value &, Callback) override                { return 0; }
    // 发送数据到服务器
    inline int64_t send(const rapidjson::Value &) override                          { return 0; }
    // 返回当前连接的序列号，这里返回固定值 0
    inline int64_t sequence() const override                                        { return 0; }
    // 提交作业结果到服务器
    inline int64_t submit(const JobResult &) override                               { return 0; }
    // 连接到指定的矿池
    inline void connect(const Pool &pool) override                                  { setPool(pool); }
    // 延迟删除当前对象
    inline void deleteLater() override                                              { delete this; }
    // 设置算法
    inline void setAlgo(const Algorithm &algo) override                             {}
    // 设置是否启用连接
    inline void setEnabled(bool enabled) override                                   {}
    // 设置代理
    inline void setProxy(const ProxyUrl &proxy) override                            {}
    // 设置是否安静模式
    inline void setQuiet(bool quiet) override                                       {}
    // 设置重试次数
    inline void setRetries(int retries) override                                    {}
    // 设置重试间隔时间
    inline void setRetryPause(uint64_t ms) override                                 {}
    // 执行定时操作
    inline void tick(uint64_t now) override                                         {}

    // 返回连接的标签
    const char *tag() const override;
    // 连接到服务器
    void connect() override;
    // 设置连接的矿池
    void setPool(const Pool &pool) override;
protected:
    // 当基准测试完成时调用，传入结果、时间差和时间戳
    void onBenchDone(uint64_t result, uint64_t diff, uint64_t ts) override;
    // 当基准测试准备就绪时调用，传入时间戳、线程数和后端信息
    void onBenchReady(uint64_t ts, uint32_t threads, const IBackend *backend) override;
    // 当接收到 HTTP 数据时调用，传入 HTTP 数据对象
    void onHttpData(const HttpData &data) override;
    // 当 DNS 解析完成时调用，传入解析结果、状态和错误信息
    void onResolved(const DnsRecords &records, int status, const char *error) override;

private:
    // 模式枚举类型，包括静态基准测试、在线基准测试、静态验证和在线验证
    enum Mode : uint32_t {
        STATIC_BENCH,
        ONLINE_BENCH,
        STATIC_VERIFY,
        ONLINE_VERIFY
    };

    // 请求枚举类型，包括无请求、获取基准测试、创建基准测试、开始基准测试和完成基准测试
    enum Request : uint32_t {
        NO_REQUEST,
        GET_BENCH,
        CREATE_BENCH,
        START_BENCH,
        DONE_BENCH
    };

    // 设置种子，传入种子字符串，返回设置是否成功
    bool setSeed(const char *seed);
    // 返回参考哈希值
    uint64_t referenceHash() const;
    // 打印退出信息
    void printExit() const;
    // 启动函数
    void start();

#   ifdef XMRIG_FEATURE_HTTP
    // 创建回复处理函数，传入 rapidjson::Value 对象
    void onCreateReply(const rapidjson::Value &value);
    // 完成回复处理函数，传入 rapidjson::Value 对象
    void onDoneReply(const rapidjson::Value &value);
    // 获取回复处理函数，传入 rapidjson::Value 对象
    void onGetReply(const rapidjson::Value &value);
    // 解析函数
    void resolve();
    // 发送请求，传入请求类型
    void send(Request request);
    // 设置错误信息，传入错误消息和标签
    void setError(const char *message, const char *label = nullptr);
    // 更新函数，传入 rapidjson::Value 对象
    void update(const rapidjson::Value &body);
#   endif

    const IBackend *m_backend   = nullptr;
    IClientListener* m_listener;
    Job m_job;
    Mode m_mode                 = STATIC_BENCH;
    Pool m_pool;
    Request m_request           = NO_REQUEST;
    std::shared_ptr<BenchConfig> m_benchmark;
    std::shared_ptr<DnsRequest> m_dns;
    std::shared_ptr<IHttpListener> m_httpListener;
    String m_ip;
    String m_token;
    uint32_t m_threads          = 0;
    uint64_t m_diff             = 0;
    uint64_t m_doneTime         = 0;
    uint64_t m_hash             = 0;
    uint64_t m_readyTime        = 0;
    uint64_t m_result           = 0;
    uint64_t m_startTime        = 0;
};


} /* namespace xmrig */


#endif /* XMRIG_BENCHCLIENT_H */
```