# `xmrig\src\base\net\stratum\Pools.h`

```
/* XMRig
 * 版权声明
 */

#ifndef XMRIG_POOLS_H
#define XMRIG_POOLS_H

#include <vector>  // 包含 vector 头文件

#include "base/net/stratum/Pool.h"  // 包含自定义的 Pool 类头文件

namespace xmrig {

class IJsonReader;  // 声明 IJsonReader 类
class IStrategy;  // 声明 IStrategy 类
class IStrategyListener;  // 声明 IStrategyListener 类

class Pools  // 定义 Pools 类
{
public:
    static const char *kDonateLevel;  // 定义静态成员变量 kDonateLevel
    static const char *kDonateOverProxy;  // 定义静态成员变量 kDonateOverProxy
    static const char *kPools;  // 定义静态成员变量 kPools
    static const char *kRetries;  // 定义静态成员变量 kRetries
    static const char *kRetryPause;  // 定义静态成员变量 kRetryPause

    enum ProxyDonate {  // 定义枚举类型 ProxyDonate
        PROXY_DONATE_NONE,  // 枚举值 PROXY_DONATE_NONE
        PROXY_DONATE_AUTO,  // 枚举值 PROXY_DONATE_AUTO
        PROXY_DONATE_ALWAYS  // 枚举值 PROXY_DONATE_ALWAYS
    };

    Pools();  // Pools 类的构造函数

#   ifdef XMRIG_FEATURE_BENCHMARK
    inline bool isBenchmark() const                     { return !!m_benchmark; }  // 如果 m_benchmark 不为空，则返回 true
#   else
    inline constexpr static bool isBenchmark()          { return false; }  // 否则返回 false
#   endif
    # 返回私有成员变量 m_data 的引用
    inline const std::vector<Pool> &data() const        { return m_data; }
    # 返回私有成员变量 m_retries 的值
    inline int retries() const                          { return m_retries; }
    # 返回私有成员变量 m_retryPause 的值
    inline int retryPause() const                       { return m_retryPause; }
    # 返回私有成员变量 m_proxyDonate 的值
    inline ProxyDonate proxyDonate() const              { return m_proxyDonate; }
    
    # 重载不等于运算符，判断当前对象是否不等于另一个 Pools 对象
    inline bool operator!=(const Pools &other) const    { return !isEqual(other); }
    # 重载等于运算符，判断当前对象是否等于另一个 Pools 对象
    inline bool operator==(const Pools &other) const    { return isEqual(other); }
    
    # 判断当前对象是否与另一个 Pools 对象相等
    bool isEqual(const Pools &other) const;
    # 返回私有成员变量 m_donateLevel 的值
    int donateLevel() const;
    # 根据监听器创建策略对象
    IStrategy *createStrategy(IStrategyListener *listener) const;
    # 将对象转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    # 返回当前活跃的对象数量
    size_t active() const;
    # 返回私有成员变量 m_benchSize 的值
    uint32_t benchSize() const;
    # 从 JSON 格式的数据中加载对象
    void load(const IJsonReader &reader);
    # 打印对象的信息
    void print() const;
    # 将对象转换为 JSON 格式
    void toJSON(rapidjson::Value &out, rapidjson::Document &doc) const;
// 设置捐赠级别
private:
    void setDonateLevel(int level);
    // 设置代理捐赠值
    void setProxyDonate(int value);
    // 设置重试次数
    void setRetries(int retries);
    // 设置重试暂停时间
    void setRetryPause(int retryPause);

    // 捐赠级别
    int m_donateLevel;
    // 重试次数，默认为5
    int m_retries               = 5;
    // 重试暂停时间，默认为5
    int m_retryPause            = 5;
    // 代理捐赠值，默认为PROXY_DONATE_AUTO
    ProxyDonate m_proxyDonate   = PROXY_DONATE_AUTO;
    // 数据池
    std::vector<Pool> m_data;

    // 如果定义了XMRIG_FEATURE_BENCHMARK，则使用BenchmarkConfig
#   ifdef XMRIG_FEATURE_BENCHMARK
    // 基准配置
    std::shared_ptr<BenchConfig> m_benchmark;
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_POOLS_H */
```