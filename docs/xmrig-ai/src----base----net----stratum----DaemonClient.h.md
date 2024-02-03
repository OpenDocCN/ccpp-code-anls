# `xmrig\src\base\net\stratum\DaemonClient.h`

```cpp
/* XMRig
 * 版权所有（c）2019      Howard Chu  <https://github.com/hyc>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DAEMONCLIENT_H
#define XMRIG_DAEMONCLIENT_H


#include "base/kernel/interfaces/IDnsListener.h"
#include "base/kernel/interfaces/IHttpListener.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/net/stratum/BaseClient.h"
#include "base/net/tools/Storage.h"
#include "base/tools/cryptonote/BlockTemplate.h"
#include "base/tools/cryptonote/WalletAddress.h"


#include <memory>


using uv_buf_t      = struct uv_buf_t;
using uv_connect_t  = struct uv_connect_s;
using uv_handle_t   = struct uv_handle_s;
using uv_stream_t   = struct uv_stream_s;
using uv_tcp_t      = struct uv_tcp_s;

#ifdef XMRIG_FEATURE_TLS
using BIO           = struct bio_st;
using SSL           = struct ssl_st;
using SSL_CTX       = struct ssl_ctx_st;
#endif


namespace xmrig {


class DnsRequest;


class DaemonClient : public BaseClient, public IDnsListener, public ITimerListener, public IHttpListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(DaemonClient)

    // 构造函数，初始化DaemonClient对象
    DaemonClient(int id, IClientListener *listener);
    // 析构函数，释放DaemonClient对象
    ~DaemonClient() override;

protected:
    // 断开连接的方法，覆盖基类的方法
    bool disconnect() override;
    # 检查是否使用了 TLS 协议
    bool isTLS() const override;
    
    # 提交作业结果并返回结果的整数值
    int64_t submit(const JobResult &result) override;
    
    # 连接到服务器
    void connect() override;
    
    # 连接到指定的服务器池
    void connect(const Pool &pool) override;
    
    # 设置连接的服务器池
    void setPool(const Pool &pool) override;
    
    # 处理 HTTP 数据
    void onHttpData(const HttpData &data) override;
    
    # 处理定时器事件
    void onTimer(const Timer *timer) override;
    
    # 处理 DNS 解析结果
    void onResolved(const DnsRecords &records, int status, const char* error) override;
    
    # 检查是否具有指定的扩展
    inline bool hasExtension(Extension) const noexcept override         { return false; }
    
    # 返回模式
    inline const char *mode() const override                            { return "daemon"; }
    
    # 返回 TLS 指纹
    inline const char *tlsFingerprint() const override                  { return m_tlsFingerprint; }
    
    # 返回 TLS 版本
    inline const char *tlsVersion() const override                      { return m_tlsVersion; }
    
    # 发送数据并返回发送结果的整数值
    inline int64_t send(const rapidjson::Value &, Callback) override    { return -1; }
    
    # 发送数据并返回发送结果的整数值
    inline int64_t send(const rapidjson::Value &) override              { return -1; }
    
    # 延迟删除当前对象
    void deleteLater() override;
    
    # 执行空操作
    inline void tick(uint64_t) override                                 {}
// 检查给定高度和哈希是否过时
bool isOutdated(uint64_t height, const char *hash) const;

// 解析作业参数
bool parseJob(const rapidjson::Value &params, int *code);

// 解析响应结果
bool parseResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error);

// 获取区块模板
int64_t getBlockTemplate();

// 发送 RPC 请求
int64_t rpcSend(const rapidjson::Document &doc, const std::map<std::string, std::string> &headers = {});

// 重试操作
void retry();

// 发送请求
void send(const char *path);

// 设置套接字状态
void setState(SocketState state);

// API 版本枚举
enum {
    API_CRYPTONOTE_DEFAULT,
    API_MONERO,
} m_apiVersion = API_MONERO;

// 区块模板
BlockTemplate m_blocktemplate;

// 加密货币
Coin m_coin;

// HTTP 监听器
std::shared_ptr<IHttpListener> m_httpListener;

// 区块哈希数据
String m_blockhashingblob;

// 区块模板请求哈希
String m_blocktemplateRequestHash;

// 区块模板字符串
String m_blocktemplateStr;

// 当前作业 ID
String m_currentJobId;

// 上一个哈希
String m_prevHash;

// 作业稳定时间
uint64_t m_jobSteadyMs = 0;

// TLS 指纹
String m_tlsFingerprint;

// TLS 版本
String m_tlsVersion;

// 定时器
Timer *m_timer;

// 区块模板请求高度
uint64_t m_blocktemplateRequestHeight = 0;

// 钱包地址
WalletAddress m_walletAddress;

// 获取客户端
static inline DaemonClient* getClient(void* data) { return m_storage.get(data); }

// 客户端存储
uintptr_t m_key = 0;
static Storage<DaemonClient> m_storage;

// ZMQ 连接回调
static void onZMQConnect(uv_connect_t* req, int status);

// ZMQ 读取回调
static void onZMQRead(uv_stream_t* stream, ssize_t nread, const uv_buf_t* buf);

// ZMQ 关闭回调
static void onZMQClose(uv_handle_t* handle);

// ZMQ 关闭连接回调
static void onZMQShutdown(uv_handle_t* handle);

// ZMQ 连接成功
void ZMQConnected();

// 写入 ZMQ 数据
bool ZMQWrite(const char* data, size_t size);

// 读取 ZMQ 数据
void ZMQRead(ssize_t nread, const uv_buf_t* buf);

// 解析 ZMQ 数据
void ZMQParse();

// 关闭 ZMQ 连接
bool ZMQClose(bool shutdown = false);

// DNS 请求
std::shared_ptr<DnsRequest> m_dns;

// ZMQ 套接字
uv_tcp_t* m_ZMQSocket = nullptr;

// ZMQ 连接状态枚举
enum {
    ZMQ_NOT_CONNECTED,
    ZMQ_GREETING_1,
    ZMQ_GREETING_2,
    ZMQ_HANDSHAKE,
    ZMQ_CONNECTED,
    ZMQ_DISCONNECTING,
} m_ZMQConnectionState = ZMQ_NOT_CONNECTED;

// ZMQ 发送缓冲区
std::vector<char> m_ZMQSendBuf;

// ZMQ 接收缓冲区
std::vector<char> m_ZMQRecvBuf;
#endif /* XMRIG_DAEMONCLIENT_H */

这行代码是 C/C++ 中的预处理指令，用于结束一个条件编译块。在这里，它表示结束对 XMRIG_DAEMONCLIENT_H 宏定义的条件编译。
```