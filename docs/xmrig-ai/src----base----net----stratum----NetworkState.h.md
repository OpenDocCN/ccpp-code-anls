# `xmrig\src\base\net\stratum\NetworkState.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NETWORKSTATE_H
#define XMRIG_NETWORKSTATE_H


#include "base/crypto/Algorithm.h"
#include "base/net/stratum/strategies/StrategyProxy.h"
#include "base/tools/String.h"


#include <array>
#include <string>
#include <vector>


namespace xmrig {


class NetworkState : public StrategyProxy
{
public:
    NetworkState(IStrategyListener *listener);

    // 返回算法
    inline const Algorithm &algorithm() const   { return m_algorithm; }
    // 返回已接受的数量
    inline uint64_t accepted() const            { return m_accepted; }
    // 返回已拒绝的数量
    inline uint64_t rejected() const            { return m_rejected; }

#   ifdef XMRIG_FEATURE_API
    // 获取连接信息
    rapidjson::Value getConnection(rapidjson::Document &doc, int version) const;
    // 获取结果信息
    rapidjson::Value getResults(rapidjson::Document &doc, int version) const;
#   endif

    // 打印连接信息
    void printConnection() const;
    // 打印结果信息
    void printResults() const;

    // 缩放难度
    static const char *scaleDiff(uint64_t &diff);
    // 人类可读的难度
    static std::string humanDiff(uint64_t diff);

protected:
    // 活跃时的回调函数
    void onActive(IStrategy *strategy, IClient *client) override;
    // 作业时的回调函数
    void onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &params) override;
    # 当策略暂停时调用的函数
    void onPause(IStrategy *strategy) override;
    # 当结果被接受时调用的函数，包括策略、客户端、提交结果和错误信息
    void onResultAccepted(IStrategy *strategy, IClient *client, const SubmitResult &result, const char *error) override;
// 声明私有成员函数，返回值为 uint32_t 类型的延迟
uint32_t latency() const;
// 声明私有成员函数，返回值为 uint64_t 类型的平均时间
uint64_t avgTime() const;
// 声明私有成员函数，返回值为 uint64_t 类型的连接时间
uint64_t connectionTime() const;
// 声明私有成员函数，参数为 SubmitResult 类型的 result 和 char 类型的 error
void add(const SubmitResult &result, const char *error);
// 声明私有成员函数，无返回值
void stop();

// 声明私有成员变量
Algorithm m_algorithm;
bool m_active = false;
char m_pool[256]{};
std::array<uint64_t, 10> m_topDiff { { } };
std::vector<uint16_t> m_latency;
String m_fingerprint;
String m_ip;
String m_tls;
uint64_t m_accepted = 0;
uint64_t m_connectionTime = 0;
uint64_t m_diff = 0;
uint64_t m_failures = 0;
uint64_t m_hashes = 0;
uint64_t m_rejected = 0;
```