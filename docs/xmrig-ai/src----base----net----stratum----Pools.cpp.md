# `xmrig\src\base\net\stratum\Pools.cpp`

```
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/stratum/Pools.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IJsonReader.h"
#include "base/net/stratum/strategies/FailoverStrategy.h"
#include "base/net/stratum/strategies/SinglePoolStrategy.h"
#include "donate.h"


#ifdef XMRIG_FEATURE_BENCHMARK
#   include "base/net/stratum/benchmark/BenchConfig.h"
#endif


namespace xmrig {


const char *Pools::kDonateLevel     = "donate-level";
const char *Pools::kDonateOverProxy = "donate-over-proxy";
const char *Pools::kPools           = "pools";
const char *Pools::kRetries         = "retries";
const char *Pools::kRetryPause      = "retry-pause";


} // namespace xmrig


xmrig::Pools::Pools() :
    m_donateLevel(kDefaultDonateLevel)
// 如果定义了 XMRIG_PROXY_PROJECT，则设置重试次数为2，重试暂停时间为1
#ifdef XMRIG_PROXY_PROJECT
    m_retries    = 2;
    m_retryPause = 1;
#endif
}


// 比较两个 Pools 对象是否相等
bool xmrig::Pools::isEqual(const Pools &other) const
{
    // 如果数据大小不相等，或者重试次数不相等，或者重试暂停时间不相等，则返回false
    if (m_data.size() != other.m_data.size() || m_retries != other.m_retries || m_retryPause != other.m_retryPause) {
        return false;
    }

    // 使用 std::equal 比较两个数据容器是否相等
    return std::equal(m_data.begin(), m_data.end(), other.m_data.begin());
}


// 获取捐赠级别
int xmrig::Pools::donateLevel() const
{
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则返回0或者 m_donateLevel
    #ifdef XMRIG_FEATURE_BENCHMARK
    return benchSize() || (m_benchmark && !m_benchmark->id().isEmpty()) ? 0 : m_donateLevel;
    #else
    // 否则返回 m_donateLevel
    return m_donateLevel;
    #endif
}


// 创建策略
xmrig::IStrategy *xmrig::Pools::createStrategy(IStrategyListener *listener) const
{
    // 如果活跃的池子数量为1
    if (active() == 1) {
        // 遍历池子，如果池子是启用状态，则创建 SinglePoolStrategy 对象
        for (const Pool &pool : m_data) {
            if (pool.isEnabled()) {
                return new SinglePoolStrategy(pool, retryPause(), retries(), listener);
            }
        }
    }

    // 创建 FailoverStrategy 对象
    auto strategy = new FailoverStrategy(retryPause(), retries(), listener);
    // 遍历池子，如果池子是启用状态，则添加到策略中
    for (const Pool &pool : m_data) {
        if (pool.isEnabled()) {
            strategy->add(pool);
        }
    }

    return strategy;
}


// 将 Pools 对象转换为 JSON 格式
rapidjson::Value xmrig::Pools::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建 JSON 数组对象
    Value pools(kArrayType);

    // 遍历池子，将每个池子转换为 JSON 对象并添加到数组中
    for (const Pool &pool : m_data) {
        pools.PushBack(pool.toJSON(doc), allocator);
    }

    return pools;
}


// 获取活跃的池子数量
size_t xmrig::Pools::active() const
{
    size_t count = 0;
    // 遍历池子，如果池子是启用状态，则计数加一
    for (const Pool &pool : m_data) {
        if (pool.isEnabled()) {
            count++;
        }
    }

    return count;
}


// 从 JSON 数据中加载池子信息
void xmrig::Pools::load(const IJsonReader &reader)
{
    // 清空池子数据
    m_data.clear();

    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则加载 Benchmark 配置
    #ifdef XMRIG_FEATURE_BENCHMARK
    m_benchmark = std::shared_ptr<BenchConfig>(BenchConfig::create(reader.getObject(BenchConfig::kBenchmark), reader.getBool("dmi", true)));
    if (m_benchmark) {
        m_data.emplace_back(m_benchmark);

        return;
    }
    #endif
}
    // 从 JSON 对象中获取名为 kPools 的数组，并赋值给变量 pools
    const rapidjson::Value &pools = reader.getArray(kPools);
    // 如果 pools 不是数组类型，则返回
    if (!pools.IsArray()) {
        return;
    }

    // 遍历 pools 数组中的每个值
    for (const rapidjson::Value &value : pools.GetArray()) {
        // 如果值不是对象类型，则继续下一次循环
        if (!value.IsObject()) {
            continue;
        }

        // 用值创建 Pool 对象
        Pool pool(value);
        // 如果 Pool 对象有效，则将其移动到 m_data 容器中
        if (pool.isValid()) {
            m_data.push_back(std::move(pool));
        }
    }

    // 设置捐赠级别
    setDonateLevel(reader.getInt(kDonateLevel, kDefaultDonateLevel));
    // 设置代理捐赠
    setProxyDonate(reader.getInt(kDonateOverProxy, PROXY_DONATE_AUTO));
    // 设置重试次数
    setRetries(reader.getInt(kRetries));
    // 设置重试暂停时间
    setRetryPause(reader.getInt(kRetryPause));
# 返回当前基准测试的大小，如果基准测试不可用则返回0
uint32_t xmrig::Pools::benchSize() const
{
#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果基准测试可用，则返回基准测试的大小，否则返回0
    return m_benchmark ? m_benchmark->size() : 0;
#   else
    # 如果基准测试不可用，则返回0
    return 0;
#   endif
}


# 打印当前所有矿池的信息
void xmrig::Pools::print() const
{
    # 初始化索引值
    size_t i = 1;
    # 遍历所有矿池并打印信息
    for (const Pool &pool : m_data) {
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("POOL #%-7zu") "%s", i, pool.printableName().c_str());

        i++;
    }

#   ifdef APP_DEBUG
    # 在调试模式下打印所有矿池的信息
    LOG_NOTICE("POOLS --------------------------------------------------------------------");
    for (const Pool &pool : m_data) {
        pool.print();
    }
    LOG_NOTICE("--------------------------------------------------------------------------");
#   endif
}


# 将矿池信息转换为JSON格式
void xmrig::Pools::toJSON(rapidjson::Value &out, rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果基准测试可用，则将基准测试信息添加到JSON中并返回
    if (m_benchmark) {
        out.AddMember(StringRef(BenchConfig::kBenchmark), m_benchmark->toJSON(doc), allocator);

        return;
    }
#   endif

    # 将矿池的捐赠等级、代理捐赠、重试次数等信息添加到JSON中
    doc.AddMember(StringRef(kDonateLevel),      m_donateLevel, allocator);
    doc.AddMember(StringRef(kDonateOverProxy),  m_proxyDonate, allocator);
    out.AddMember(StringRef(kPools),            toJSON(doc), allocator);
    doc.AddMember(StringRef(kRetries),          retries(), allocator);
    doc.AddMember(StringRef(kRetryPause),       retryPause(), allocator);
}


# 设置矿池的捐赠等级
void xmrig::Pools::setDonateLevel(int level)
{
    # 如果捐赠等级在合理范围内，则设置捐赠等级
    if (level >= kMinimumDonateLevel && level <= 99) {
        m_donateLevel = level;
    }
}


# 设置矿池的代理捐赠方式
void xmrig::Pools::setProxyDonate(int value)
{
    # 根据传入的值设置矿池的代理捐赠方式
    switch (value) {
    case PROXY_DONATE_NONE:
    case PROXY_DONATE_AUTO:
    case PROXY_DONATE_ALWAYS:
        m_proxyDonate = static_cast<ProxyDonate>(value);

    default:
        break;
    }
}


# 设置矿池的重试次数
void xmrig::Pools::setRetries(int retries)
{
    # 如果重试次数在合理范围内，则设置重试次数
    if (retries > 0 && retries <= 1000) {
        m_retries = retries;
    }
}


# 设置矿池的重试间隔
void xmrig::Pools::setRetryPause(int retryPause)
{
    # 如果重试暂停时间大于0并且小于等于3600秒
    if (retryPause > 0 && retryPause <= 3600) {
        # 将重试暂停时间赋值给m_retryPause
        m_retryPause = retryPause;
    }
# 闭合前面的函数定义
```