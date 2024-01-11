# `xmrig\src\base\net\stratum\benchmark\BenchConfig.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BENCHCONFIG_H
#define XMRIG_BENCHCONFIG_H


#include "base/crypto/Algorithm.h"
#include "base/tools/String.h"


namespace xmrig {


class BenchConfig
{
public:
    static const char *kAlgo;  // 算法名称
    static const char *kApiHost;  // API 主机
    static const char *kBenchmark;  // 基准测试
    static const char *kHash;  // 哈希
    static const char *kId;  // ID
    static const char *kSeed;  // 种子
    static const char *kSize;  // 大小
    static const char* kRotation;  // 旋转
    static const char *kSubmit;  // 提交
    static const char *kToken;  // 令牌
    static const char *kUser;  // 用户
    static const char *kVerify;  // 验证

#   ifndef XMRIG_DEBUG_BENCHMARK_API
    static constexpr bool kApiTLS               = true;  // API 是否使用 TLS
    static constexpr const uint16_t kApiPort    = 443;  // API 端口号
#   else
    static constexpr bool kApiTLS               = false;  // API 是否使用 TLS
    static constexpr const uint16_t kApiPort    = 18805;  // API 端口号
#   endif

    BenchConfig(uint32_t size, const String &id, const rapidjson::Value &object, bool dmi, uint32_t rotation);  // 构造函数

    static BenchConfig *create(const rapidjson::Value &object, bool dmi);  // 创建 BenchConfig 对象

    inline bool isDMI() const                   { return m_dmi; }  // 是否为 DMI
    inline bool isSubmit() const                { return m_submit; }  // 是否提交
    # 返回成员变量 m_algorithm 的引用
    inline const Algorithm &algorithm() const   { return m_algorithm; }
    # 返回成员变量 m_id 的引用
    inline const String &id() const             { return m_id; }
    # 返回成员变量 m_seed 的引用
    inline const String &seed() const           { return m_seed; }
    # 返回成员变量 m_token 的引用
    inline const String &token() const          { return m_token; }
    # 返回成员变量 m_user 的引用
    inline const String &user() const           { return m_user; }
    # 返回成员变量 m_size 的值
    inline uint32_t size() const                { return m_size; }
    # 返回成员变量 m_hash 的值
    inline uint64_t hash() const                { return m_hash; }
    # 返回成员变量 m_rotation 的值
    inline uint32_t rotation() const            { return m_rotation; }
    
    # 将对象转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
// 声明私有成员变量和静态方法
private:
    static uint32_t getSize(const char *benchmark); // 声明一个静态方法，用于获取 benchmark 的大小

    // 声明各种成员变量
    Algorithm m_algorithm; // 用于存储算法
    bool m_dmi; // 用于标识是否启用 DMI
    bool m_submit; // 用于标识是否提交
    String m_id; // 用于存储 ID
    String m_seed; // 用于存储种子
    String m_token; // 用于存储令牌
    String m_user; // 用于存储用户
    uint32_t m_size; // 用于存储大小
    uint32_t m_rotation; // 用于存储旋转
    uint64_t m_hash = 0; // 用于存储哈希值
};


} /* namespace xmrig */ // 结束 xmrig 命名空间


#endif /* XMRIG_BENCHCONFIG_H */ // 结束头文件定义
```