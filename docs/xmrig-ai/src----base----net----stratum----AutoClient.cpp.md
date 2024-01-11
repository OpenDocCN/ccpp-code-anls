# `xmrig\src\base\net\stratum\AutoClient.cpp`

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
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/AutoClient.h" // 包含自定义的头文件
#include "3rdparty/rapidjson/document.h"  // 包含第三方库的头文件
#include "base/io/json/Json.h"            // 包含自定义的头文件

// 构造函数，初始化AutoClient对象
xmrig::AutoClient::AutoClient(int id, const char *agent, IClientListener *listener) :
    EthStratumClient(id, agent, listener)
{
}

// 处理响应的方法
bool xmrig::AutoClient::handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error)
{
    // 如果模式为默认模式，则调用父类Client的handleResponse方法
    if (m_mode == DEFAULT_MODE) {
        return Client::handleResponse(id, result, error); // NOLINT(bugprone-parent-virtual-call)
    }

    // 否则调用EthStratumClient的handleResponse方法
    return EthStratumClient::handleResponse(id, result, error);
}

// 解析登录信息的方法
bool xmrig::AutoClient::parseLogin(const rapidjson::Value &result, int *code)
{
    // 如果结果中包含"job"成员，则调用父类Client的parseLogin方法
    if (result.HasMember("job")) {
        return Client::parseLogin(result, code);
    }

    // 设置RPC ID
    setRpcId(Json::getString(result, "id"));
    if (rpcId().isNull()) {
        *code = 1;
        return false;
    }

    // 获取算法类型并进行判断
    const Algorithm algo(Json::getString(result, "algo"));
    if (algo.family() != Algorithm::KAWPOW && algo.family() != Algorithm::GHOSTRIDER) {
        *code = 6;
        return false;
    }

    // 尝试设置额外的nonce
    try {
        setExtraNonce(Json::getValue(result, "extra_nonce"));
    } catch (const std::exception &ex) {
        # 捕获异常并处理，将错误代码设置为6，返回false
        *code = 6;
        return false;
    }

    # 设置模式为ETH_MODE
    m_mode = ETH_MODE;
    # 设置算法
    setAlgo(algo);
// 如果定义了 XMRIG_ALGO_GHOSTRIDER
    if (algo.family() == Algorithm::GHOSTRIDER) {
        // 设置额外的Nonce2大小
        setExtraNonce2Size(Json::getUint64(result, "extra_nonce2_size"));
    }
// 返回true
    return true;
}


// 提交作业结果
int64_t xmrig::AutoClient::submit(const JobResult &result)
{
    // 如果模式为默认模式
    if (m_mode == DEFAULT_MODE) {
        // 调用Client类的submit方法，NOLINT(bugprone-parent-virtual-call)表示忽略虚函数调用的警告
        return Client::submit(result); 
    }

    // 返回EthStratumClient类的submit方法
    return EthStratumClient::submit(result);
}


// 解析通知
void xmrig::AutoClient::parseNotification(const char *method, const rapidjson::Value &params, const rapidjson::Value &error)
{
    // 如果模式为默认模式
    if (m_mode == DEFAULT_MODE) {
        // 调用Client类的parseNotification方法，NOLINT(bugprone-parent-virtual-call)表示忽略虚函数调用的警告
        return Client::parseNotification(method, params, error); 
    }

    // 返回EthStratumClient类的parseNotification方法
    return EthStratumClient::parseNotification(method, params, error);
}
```