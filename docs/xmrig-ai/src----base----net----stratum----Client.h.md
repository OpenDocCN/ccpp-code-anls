# `xmrig\src\base\net\stratum\Client.h`

```cpp
/* XMRig
 * 版权所有（c）2019      jtgrassie   <https://github.com/jtgrassie>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它将是有用的，但没有任何保证；甚至没有对适销性或特定用途的隐含保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CLIENT_H
#define XMRIG_CLIENT_H

#include <bitset>
#include <map>
#include <uv.h>
#include <vector>

#include "base/kernel/interfaces/IDnsListener.h"
#include "base/kernel/interfaces/ILineListener.h"
#include "base/net/stratum/BaseClient.h"
#include "base/net/stratum/Job.h"
#include "base/net/stratum/Pool.h"
#include "base/net/stratum/SubmitResult.h"
#include "base/net/tools/LineReader.h"
#include "base/net/tools/Storage.h"
#include "base/tools/Object.h"

using BIO = struct bio_st;

namespace xmrig {

class DnsRequest;
class IClientListener;
class JobResult;

class Client : public BaseClient, public IDnsListener, public ILineListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Client)  // 禁用默认的拷贝和移动构造函数

    constexpr static uint64_t kConnectTimeout   = 20 * 1000;  // 连接超时时间
    constexpr static uint64_t kResponseTimeout  = 20 * 1000;  // 响应超时时间
    constexpr static size_t kMaxSendBufferSize  = 1024 * 16;  // 最大发送缓冲区大小

    Client(int id, const char *agent, IClientListener *listener);  // 构造函数
    ~Client() override;  // 析构函数

protected:
    bool disconnect() override;  // 断开连接的方法
    bool isTLS() const override;  // 判断是否使用TLS
    // 返回 TLS 指纹
    const char *tlsFingerprint() const override;
    // 返回 TLS 版本
    const char *tlsVersion() const override;
    // 发送数据并指定回调函数
    int64_t send(const rapidjson::Value &obj, Callback callback) override;
    // 发送数据
    int64_t send(const rapidjson::Value &obj) override;
    // 提交作业结果
    int64_t submit(const JobResult &result) override;
    // 连接
    void connect() override;
    // 连接到指定的连接池
    void connect(const Pool &pool) override;
    // 延迟删除
    void deleteLater() override;
    // 定时器触发
    void tick(uint64_t now) override;

    // 解析 DNS 记录回调
    void onResolved(const DnsRecords &records, int status, const char *error) override;

    // 检查是否具有指定扩展
    inline bool hasExtension(Extension extension) const noexcept override   { return m_extensions.test(extension); }
    // 返回模式
    inline const char *mode() const override                                { return "pool"; }
    // 处理接收到的数据
    inline void onLine(char *line, size_t size) override                    { parse(line, size); }

    // 返回代理信息
    inline const char *agent() const                                        { return m_agent; }
    // 返回连接的 URL
    inline const char *url() const                                          { return m_pool.url(); }
    // 返回 RPC ID
    inline const String &rpcId() const                                      { return m_rpcId; }
    // 设置 RPC ID
    inline void setRpcId(const char *id)                                    { m_rpcId = id; }
    // 设置连接池的 URL
    inline void setPoolUrl(const char *url)                                 { m_pool.setUrl(url); }

    // 解析登录结果
    virtual bool parseLogin(const rapidjson::Value &result, int *code);
    // 登录
    virtual void login();
    // 解析通知
    virtual void parseNotification(const char* method, const rapidjson::Value& params, const rapidjson::Value& error);

    // 关闭连接
    bool close();
    // 关闭连接回调
    virtual void onClose();
private:
    // 声明内部类 Socks5
    class Socks5;
    // 声明内部类 Tls

    // 解析任务参数，并返回解析结果和状态码
    bool parseJob(const rapidjson::Value &params, int *code);
    // 发送数据到 BIO
    bool send(BIO *bio);
    // 验证算法是否匹配
    bool verifyAlgorithm(const Algorithm &algorithm, const char *algo) const;
    // 写入数据到缓冲区
    bool write(const uv_buf_t &buf);
    // 解析主机名
    int resolve(const String &host);
    // 发送指定大小的数据
    int64_t send(size_t size);
    // 连接到指定地址
    void connect(const sockaddr *addr);
    // 握手
    void handshake();
    // 解析数据
    void parse(char *line, size_t len);
    // 解析扩展信息
    void parseExtensions(const rapidjson::Value &result);
    // 解析响应
    void parseResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error);
    // 发送 ping
    void ping();
    // 读取数据
    void read(ssize_t nread, const uv_buf_t *buf);
    // 重新连接
    void reconnect();
    // 设置套接字状态
    void setState(SocketState state);
    // 启动超时计时器
    void startTimeout();

    // 获取套接字状态
    inline SocketState state() const                                { return m_state; }
    // 获取流对象
    inline uv_stream_t *stream() const                              { return reinterpret_cast<uv_stream_t *>(m_socket); }
    // 设置扩展信息
    inline void setExtension(Extension ext, bool enable) noexcept   { m_extensions.set(ext, enable); }
    // 检查是否存在指定扩展
    template<Extension ext> inline bool has() const noexcept        { return m_extensions.test(ext); }

    // 判断是否为关键错误
    static bool isCriticalError(const char *message);
    // 关闭回调函数
    static void onClose(uv_handle_t *handle);
    // 连接回调函数
    static void onConnect(uv_connect_t *req, int status);
    // 读取回调函数
    static void onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf);

    // 获取客户端对象
    static inline Client *getClient(void *data) { return m_storage.get(data); }

    // 客户端代理信息
    const char *m_agent;
    // 行读取器
    LineReader m_reader;
    // Socks5 对象指针
    Socks5 *m_socks5            = nullptr;
    // 扩展信息位集合
    std::bitset<EXT_MAX> m_extensions;
    // DNS 请求对象指针
    std::shared_ptr<DnsRequest> m_dns;
    // 发送缓冲区
    std::vector<char> m_sendBuf;
    // 临时缓冲区
    std::vector<char> m_tempBuf;
    // RPC ID
    String m_rpcId;
    // Tls 对象指针
    Tls *m_tls                  = nullptr;
    // 过期时间
    uint64_t m_expire           = 0;
    // 任务数
    uint64_t m_jobs             = 0;
    // 保持连接时间
    uint64_t m_keepAlive        = 0;
    // 键值
    uintptr_t m_key             = 0;
    // TCP 套接字指针
    uv_tcp_t *m_socket          = nullptr;

    // 客户端存储对象
    static Storage<Client> m_storage;
};
// 检查客户端是否具有 EXT_NICEHASH 扩展，如果 m_extensions 中包含 EXT_NICEHASH 或者 m_pool 是 Nicehash，则返回 true
template<> inline bool Client::has<Client::EXT_NICEHASH>() const noexcept  { return m_extensions.test(EXT_NICEHASH) || m_pool.isNicehash(); }
// 检查客户端是否具有 EXT_KEEPALIVE 扩展，如果 m_extensions 中包含 EXT_KEEPALIVE 或者 m_pool 的 keepAlive() 大于 0，则返回 true
template<> inline bool Client::has<Client::EXT_KEEPALIVE>() const noexcept { return m_extensions.test(EXT_KEEPALIVE) || m_pool.keepAlive() > 0; }
// 命名空间结束
} /* namespace xmrig */
// 结束 if 条件，结束文件
#endif /* XMRIG_CLIENT_H */
```