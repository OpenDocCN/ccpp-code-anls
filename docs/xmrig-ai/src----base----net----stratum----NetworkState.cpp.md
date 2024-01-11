# `xmrig\src\base\net\stratum\NetworkState.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/NetworkState.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IClient.h"
#include "base/kernel/interfaces/IStrategy.h"
#include "base/net/stratum/Job.h"
#include "base/net/stratum/Pool.h"
#include "base/net/stratum/SubmitResult.h"
#include "base/tools/Chrono.h"


#include <algorithm>
#include <cstdio>
#include <cstring>
#include <uv.h>



namespace xmrig {


inline static void printCount(uint64_t accepted, uint64_t rejected)
{
    float percent   = 100.0;  // 初始化百分比为100.0
    int color       = 2;       // 初始化颜色为2

    if (!accepted) {  // 如果接受的数量为0
        percent     = 0.0;     // 百分比为0.0
        color       = 1;       // 颜色为1
    }
    else if (rejected) {  // 如果拒绝的数量不为0
        percent     = static_cast<float>(accepted) / (accepted + rejected) * 100.0;  // 计算接受数量占总数的百分比
        color       = 3;       // 颜色为3
    }

    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CSI "1;3%dm%" PRIu64 CLEAR CSI "0;3%dm (%1.1f%%)", "accepted", color, accepted, color, percent);  // 打印接受数量和百分比

    if (rejected) {  // 如果拒绝的数量不为0
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") RED_BOLD("%" PRIu64), "rejected", rejected);  // 打印拒绝数量
    }
}


inline static void printHashes(uint64_t accepted, uint64_t hashes)
{
    # 使用 Log 类的 print 方法打印带有颜色和格式的信息
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CYAN_BOLD("%" PRIu64) " avg " CYAN("%1.0f"),
               # 打印字符串 "pool-side hashes"，使用绿色粗体显示
               "pool-side hashes", 
               # 打印无符号整数，使用白色粗体显示
               hashes, 
               # 打印平均值，使用青色粗体显示
               static_cast<double>(hashes) / accepted);
}

// 打印平均结果时间
inline static void printAvgTime(uint64_t time)
{
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CSI "1;3%dm%1.1fs", "avg result time", (time < 10000 ? 3 : 2), time / 1000.0);
}

// 打印难度
static void printDiff(uint64_t diff)
{
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CYAN_BOLD("%s"), "difficulty", NetworkState::humanDiff(diff).c_str());
}

// 打印难度和工作量
inline static void printDiff(size_t i, uint64_t diff, uint64_t hashes)
{
    if (!diff) {
        return;
    }

    const double effort = static_cast<double>(hashes) / diff * 100.0;
    const double target = (i + 1) * 100.0;
    const int color     = effort > (target + 100.0) ? 1 : (effort > target ? 3 : 2);

    Log::print("%3zu | %10s | " CSI "0;3%dm%8.2f" CLEAR " |", i + 1, NetworkState::humanDiff(diff).c_str(), color, effort);
}

// 打印延迟
inline static void printLatency(uint32_t latency)
{
    if (!latency) {
        return;
    }

    const int color = latency < 100 ? 2 : (latency > 500 ? 1 : 3);

    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CSI "1;3%dm%ums", "ping time", color, latency);
}

} // namespace xmrig

// 构造函数，初始化网络状态
xmrig::NetworkState::NetworkState(IStrategyListener *listener) : StrategyProxy(listener)
{
}

// 获取连接信息
#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::NetworkState::getConnection(rapidjson::Document &doc, int version) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    Value connection(kObjectType);
    connection.AddMember("pool",            StringRef(m_pool), allocator);
    connection.AddMember("ip",              m_ip.toJSON(), allocator);
    connection.AddMember("uptime",          connectionTime() / 1000, allocator);
    connection.AddMember("uptime_ms",       connectionTime(), allocator);
    connection.AddMember("ping",            latency(), allocator);
    connection.AddMember("failures",        m_failures, allocator);
    connection.AddMember("tls",             m_tls.toJSON(), allocator);
    # 向连接对象添加 TLS 指纹信息
    connection.AddMember("tls-fingerprint", m_fingerprint.toJSON(), allocator);

    # 向连接对象添加算法信息
    connection.AddMember("algo",            m_algorithm.toJSON(), allocator);
    # 向连接对象添加差异信息
    connection.AddMember("diff",            m_diff, allocator);
    # 向连接对象添加接受的信息
    connection.AddMember("accepted",        m_accepted, allocator);
    # 向连接对象添加拒绝的信息
    connection.AddMember("rejected",        m_rejected, allocator);
    # 向连接对象添加平均时间（以秒为单位）
    connection.AddMember("avg_time",        avgTime() / 1000, allocator);
    # 向连接对象添加平均时间（以毫秒为单位）
    connection.AddMember("avg_time_ms",     avgTime(), allocator);
    # 向连接对象添加总哈希数
    connection.AddMember("hashes_total",    m_hashes, allocator);

    # 如果版本为1，则向连接对象添加错误日志信息
    if (version == 1) {
        connection.AddMember("error_log", Value(kArrayType), allocator);
    }

    # 返回连接对象
    return connection;
rapidjson::Value xmrig::NetworkState::getResults(rapidjson::Document &doc, int version) const
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个对象类型的值
    Value results(kObjectType);

    // 向结果对象中添加成员
    results.AddMember("diff_current",  m_diff, allocator);
    results.AddMember("shares_good",   m_accepted, allocator);
    results.AddMember("shares_total",  m_accepted + m_rejected, allocator);
    results.AddMember("avg_time",      avgTime() / 1000, allocator);
    results.AddMember("avg_time_ms",   avgTime(), allocator);
    results.AddMember("hashes_total",  m_hashes, allocator);

    // 创建一个数组类型的值
    Value best(kArrayType);
    // 预留空间
    best.Reserve(m_topDiff.size(), allocator);

    // 遍历 m_topDiff 数组，将元素添加到 best 数组中
    for (uint64_t i : m_topDiff) {
        best.PushBack(i, allocator);
    }

    // 向结果对象中添加 best 数组
    results.AddMember("best", best, allocator);

    // 如果版本为1，则向结果对象中添加 error_log 数组
    if (version == 1) {
        results.AddMember("error_log", Value(kArrayType), allocator);
    }

    // 返回结果对象
    return results;
}
#endif


void xmrig::NetworkState::printConnection() const
{
    // 如果连接不活跃，则打印提示信息并返回
    if (!m_active) {
        LOG_NOTICE(YELLOW_BOLD_S "no active connection");
        return;
    }

    // 打印连接信息
    Log::print(MAGENTA_BOLD_S " - CONNECTION");
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CYAN_BOLD("%s ") BLACK_BOLD("(%s) ") GREEN_BOLD("%s"),
               "pool address", m_pool, m_ip.data(), m_tls.isNull() ? "" : m_tls.data());

    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") WHITE_BOLD("%s"), "algorithm", m_algorithm.name());
    printDiff(m_diff);
    printLatency(latency());
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-17s") CYAN_BOLD("%" PRIu64 "s"), "connection time", connectionTime() / 1000);
}


void xmrig::NetworkState::printResults() const
{
    // 如果没有哈希值，则打印提示信息并返回
    if (!m_hashes) {
        LOG_NOTICE(YELLOW_BOLD_S "no results yet");
        return;
    }

    // 打印结果信息
    Log::print(MAGENTA_BOLD_S " - RESULTS");

    printCount(m_accepted, m_rejected);
    printHashes(m_accepted, m_hashes);
    printDiff(m_diff);

    // 如果连接活跃且延迟不为空，则打印平均时间
    if (m_active && !m_latency.empty()) {
        printAvgTime(avgTime());
    }
}
    # 调用Log类的print方法，打印带有紫色加粗样式的字符串“ - TOP 10”
    Log::print(MAGENTA_BOLD_S " - TOP 10");
    # 调用Log类的print方法，打印带有白色加粗样式的字符串“  # | DIFFICULTY | EFFORT %% |”
    Log::print(WHITE_BOLD_S "  # | DIFFICULTY | EFFORT %% |");

    # 遍历m_topDiff数组，打印每个元素的信息
    for (size_t i = 0; i < m_topDiff.size(); ++i) {
        # 调用printDiff方法，打印第i个元素的信息，包括索引i，m_topDiff[i]的值，m_hashes的值
        printDiff(i, m_topDiff[i], m_hashes);
    }
# 将难度值按照不同的量级进行缩放，并返回对应的量级标识
const char *xmrig::NetworkState::scaleDiff(uint64_t &diff)
{
    # 如果难度值大于等于100000000000，将难度值除以1000000000，并返回"G"
    if (diff >= 100000000000) {
        diff /= 1000000000;
        return "G";
    }
    # 如果难度值大于等于100000000，将难度值除以1000000，并返回"M"
    if (diff >= 100000000) {
        diff /= 1000000;
        return "M";
    }
    # 如果难度值大于等于1000000，将难度值除以1000，并返回"K"
    if (diff >= 1000000) {
        diff /= 1000;
        return "K";
    }
    # 如果难度值小于1000000，返回空字符串
    return "";
}

# 将难度值转换为带有量级标识的字符串
std::string xmrig::NetworkState::humanDiff(uint64_t diff)
{
    # 调用scaleDiff函数，获取量级标识
    const char *scale = scaleDiff(diff);
    # 将难度值和量级标识转换为字符串并返回
    return std::to_string(diff) + scale;
}

# 当连接处于活跃状态时调用的函数
void xmrig::NetworkState::onActive(IStrategy *strategy, IClient *client)
{
    # 格式化连接池的地址和端口号
    snprintf(m_pool, sizeof(m_pool) - 1, "%s:%d", client->pool().host().data(), client->pool().port());
    # 获取客户端的IP地址、TLS版本和TLS指纹
    m_ip             = client->ip();
    m_tls            = client->tlsVersion();
    m_fingerprint    = client->tlsFingerprint();
    # 设置连接状态为活跃
    m_active         = true;
    # 记录连接时间
    m_connectionTime = Chrono::steadyMSecs();
    # 调用父类的onActive函数
    StrategyProxy::onActive(strategy, client);
}

# 当接收到新的工作任务时调用的函数
void xmrig::NetworkState::onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &params)
{
    # 获取工作任务的算法和难度值
    m_algorithm = job.algorithm();
    m_diff      = job.diff();
    # 调用父类的onJob函数
    StrategyProxy::onJob(strategy, client, job, params);
}

# 当连接暂停时调用的函数
void xmrig::NetworkState::onPause(IStrategy *strategy)
{
    # 如果策略不处于活跃状态，则停止连接
    if (!strategy->isActive()) {
        stop();
    }
    # 调用父类的onPause函数
    StrategyProxy::onPause(strategy);
}

# 当接收到提交结果时调用的函数
void xmrig::NetworkState::onResultAccepted(IStrategy *strategy, IClient *client, const SubmitResult &result, const char *error)
{
    # 将提交结果和错误信息添加到记录中
    add(result, error);
    # 调用父类的onResultAccepted函数
    StrategyProxy::onResultAccepted(strategy, client, result, error);
}

# 获取连接的延迟时间
uint32_t xmrig::NetworkState::latency() const
{
    # 获取延迟时间记录的数量
    const size_t calls = m_latency.size();
    # 如果记录数量为0，则返回0
    if (calls == 0) {
        return 0;
    }
    # 对延迟时间记录进行排序，并返回中间值
    auto v = m_latency;
    std::nth_element(v.begin(), v.begin() + calls / 2, v.end());
    return v[calls / 2];
}

# 获取平均连接时间
uint64_t xmrig::NetworkState::avgTime() const
{
    # 如果延迟时间记录为空，则返回0
    if (m_latency.empty()) {
        return 0;
    }
    # 返回连接时间除以延迟时间记录数量的结果
    return connectionTime() / m_latency.size();
}
// 返回连接时间，如果活动状态为真，则返回当前时间减去连接时间，否则返回0
uint64_t xmrig::NetworkState::connectionTime() const
{
    return m_active ? ((Chrono::steadyMSecs() - m_connectionTime)) : 0;
}

// 添加提交结果和错误信息
void xmrig::NetworkState::add(const SubmitResult &result, const char *error)
{
    // 如果有错误信息，则增加被拒绝的计数并返回
    if (error) {
        m_rejected++;
        return;
    }

    // 增加被接受的计数
    m_accepted++;
    // 增加哈希数
    m_hashes += result.diff;

    // 获取顶部难度数组的长度
    const size_t ln = m_topDiff.size() - 1;
    // 如果实际难度大于顶部难度数组的最后一个元素，则更新顶部难度数组并排序
    if (result.actualDiff > m_topDiff[ln]) {
        m_topDiff[ln] = result.actualDiff;
        std::sort(m_topDiff.rbegin(), m_topDiff.rend());
    }

    // 将延迟时间添加到延迟数组中，如果超过最大值则设置为最大值
    m_latency.push_back(result.elapsed > 0xFFFF ? 0xFFFF : static_cast<uint16_t>(result.elapsed));
}

// 停止网络状态
void xmrig::NetworkState::stop()
{
    // 设置活动状态为假，重置难度、IP、TLS和指纹
    m_active      = false;
    m_diff        = 0;
    m_ip          = nullptr;
    m_tls         = nullptr;
    m_fingerprint = nullptr;

    // 增加失败计数，清空延迟数组
    m_failures++;
    m_latency.clear();
}
```