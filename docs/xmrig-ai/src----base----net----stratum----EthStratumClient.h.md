# `xmrig\src\base\net\stratum\EthStratumClient.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ETHSTRATUMCLIENT_H
#define XMRIG_ETHSTRATUMCLIENT_H


#include "base/net/stratum/Client.h"


#include <utility>


namespace xmrig {


class EthStratumClient : public Client
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(EthStratumClient)

    // 构造函数，初始化以太坊 Stratum 客户端
    EthStratumClient(int id, const char *agent, IClientListener *listener);
    // 析构函数
    ~EthStratumClient() override = default;

protected:
    // 提交作业结果
    int64_t submit(const JobResult &result) override;
    // 登录
    void login() override;
    // 关闭连接
    void onClose() override;

    // 处理响应
    bool handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error) override;
    // 解析通知
    void parseNotification(const char *method, const rapidjson::Value &params, const rapidjson::Value &error) override;

    // 设置额外的随机数
    void setExtraNonce(const rapidjson::Value &nonce);

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 设置额外的随机数2的大小
    inline void setExtraNonce2Size(uint64_t size)   { m_extraNonce2Size = size; }
#   endif

private:
    // 获取错误消息
    static const char *errorMessage(const rapidjson::Value &error);

    // 授权
    void authorize();
    // 处理授权响应
    void onAuthorizeResponse(const rapidjson::Value &result, bool success, uint64_t elapsed);
    # 定义一个名为onSubscribeResponse的函数，参数为rapidjson::Value类型的result，bool类型的success，uint64_t类型的elapsed
    void onSubscribeResponse(const rapidjson::Value &result, bool success, uint64_t elapsed);
    
    # 定义一个名为subscribe的函数，无参数
    void subscribe();
    
    # 定义一个名为m_authorized的布尔类型变量，并初始化为false
    bool m_authorized   = false;
    
    # 定义一个名为m_extraNonce的pair类型变量，包含一个uint64_t类型和一个String类型的元素，并初始化为默认值
    std::pair<uint64_t, String> m_extraNonce{};
#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果定义了 XMRIG_ALGO_GHOSTRIDER，则执行以下代码
    uint64_t m_extraNonce2Size = 0;
    // 初始化 m_extraNonce2Size 为 0
    uint64_t m_nextDifficulty = 0;
    // 初始化 m_nextDifficulty 为 0
    String m_ntime;
    // 声明一个 String 类型的变量 m_ntime
#   endif
};


} /* namespace xmrig */
// 结束 xmrig 命名空间


#endif /* XMRIG_ETHSTRATUMCLIENT_H */
// 结束 XMRIG_ETHSTRATUMCLIENT_H 的条件编译
```