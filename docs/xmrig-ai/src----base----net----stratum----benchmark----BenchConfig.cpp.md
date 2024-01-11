# `xmrig\src\base\net\stratum\benchmark\BenchConfig.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/benchmark/BenchConfig.h"
#include "3rdparty/fmt/core.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


#include <string>


#ifdef _MSC_VER
#   define strcasecmp  _stricmp
#endif


namespace xmrig {


const char *BenchConfig::kAlgo      = "algo";
const char *BenchConfig::kBenchmark = "benchmark";
const char *BenchConfig::kHash      = "hash";
const char *BenchConfig::kId        = "id";
const char *BenchConfig::kSeed      = "seed";
const char *BenchConfig::kSize      = "size";
const char *BenchConfig::kRotation  = "rotation";
const char *BenchConfig::kSubmit    = "submit";
const char *BenchConfig::kToken     = "token";
const char *BenchConfig::kUser      = "user";
const char *BenchConfig::kVerify    = "verify";

#ifndef XMRIG_DEBUG_BENCHMARK_API
const char *BenchConfig::kApiHost   = "api.xmrig.com";
#else
const char *BenchConfig::kApiHost   = "127.0.0.1";
#endif

} // namespace xmrig


xmrig::BenchConfig::BenchConfig(uint32_t size, const String &id, const rapidjson::Value &object, bool dmi, uint32_t rotation) :
    m_algorithm(Json::getString(object, kAlgo)),
    m_dmi(dmi),

- 该部分代码是C++程序的头文件，包含了一些版权声明和宏定义
- 定义了xmrig命名空间
- 定义了BenchConfig类的一些常量成员变量
- 根据条件定义了kApiHost的值
- 实现了BenchConfig类的构造函数，初始化了m_algorithm和m_dmi成员变量
    # 使用 Json 对象中的键 kSubmit 获取布尔值，并赋值给成员变量 m_submit
    m_submit(Json::getBool(object, kSubmit)),
    # 将 id 赋值给成员变量 m_id
    m_id(id),
    # 使用 Json 对象中的键 kSeed 获取字符串，并赋值给成员变量 m_seed
    m_seed(Json::getString(object, kSeed)),
    # 使用 Json 对象中的键 kToken 获取字符串，并赋值给成员变量 m_token
    m_token(Json::getString(object, kToken)),
    # 使用 Json 对象中的键 kUser 获取字符串，并赋值给成员变量 m_user
    m_user(Json::getString(object, kUser)),
    # 将 size 赋值给成员变量 m_size
    m_size(size),
    # 将 rotation 赋值给成员变量 m_rotation
    m_rotation(rotation)
{
    // 获取算法族
    auto f = m_algorithm.family();
    // 如果算法无效或者不是随机算法
    if (!m_algorithm.isValid() || (f != Algorithm::RANDOM_X
#       ifdef XMRIG_ALGO_GHOSTRIDER
        && f != Algorithm::GHOSTRIDER
#       endif
        )) {
        // 将算法设置为 RX_0
        m_algorithm = Algorithm::RX_0;
    }

    // 获取 JSON 对象中的哈希值
    const char *hash = Json::getString(object, kHash);
    // 如果哈希值存在
    if (hash) {
        // 将哈希值转换为无符号长长整型
        m_hash = strtoull(hash, nullptr, 16);
    }
}


// 创建 BenchConfig 对象的静态方法
xmrig::BenchConfig *xmrig::BenchConfig::create(const rapidjson::Value &object, bool dmi)
{
    // 如果对象不是对象类型或者为空对象
    if (!object.IsObject() || object.ObjectEmpty()) {
        // 返回空指针
        return nullptr;
    }

    // 获取尺寸和验证 ID
    const uint32_t size     = getSize(Json::getString(object, kSize));
    const String id         = Json::getString(object, kVerify);

    // 获取旋转值
    const char* rotation_str = Json::getString(object, kRotation);
    const uint32_t rotation = rotation_str ? strtoul(rotation_str, nullptr, 10) : 0;

    // 如果尺寸为0且验证 ID 为空
    if (size == 0 && id.isEmpty()) {
        // 返回空指针
        return nullptr;
    }

    // 创建 BenchConfig 对象并返回
    return new BenchConfig(size, id, object, dmi, rotation);
}


// 将 BenchConfig 对象转换为 JSON 对象
rapidjson::Value xmrig::BenchConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    // 创建一个对象类型的 JSON 值
    Value out(kObjectType);
    // 获取分配器
    auto &allocator = doc.GetAllocator();

    // 如果尺寸为0
    if (m_size == 0) {
        // 添加尺寸为0的成员
        out.AddMember(StringRef(kSize), 0U, allocator);
    }
    // 如果尺寸小于1000000
    else if (m_size < 1000000) {
        // 添加尺寸为 K 的成员
        out.AddMember(StringRef(kSize), Value(fmt::format("{}K", m_size / 1000).c_str(), allocator), allocator);
    }
    // 否则
    else {
        // 添加尺寸为 M 的成员
        out.AddMember(StringRef(kSize), Value(fmt::format("{}M", m_size / 1000000).c_str(), allocator), allocator);
    }

    // 添加算法成员
    out.AddMember(StringRef(kAlgo),     m_algorithm.toJSON(), allocator);
    // 添加提交成员
    out.AddMember(StringRef(kSubmit),   m_submit, allocator);
    // 添加验证成员
    out.AddMember(StringRef(kVerify),   m_id.toJSON(), allocator);
    // 添加令牌成员
    out.AddMember(StringRef(kToken),    m_token.toJSON(), allocator);
    // 添加种子成员
    out.AddMember(StringRef(kSeed),     m_seed.toJSON(), allocator);
    // 添加用户成员
    out.AddMember(StringRef(kUser),     m_user.toJSON(), allocator);
    # 如果存在哈希值
    if (m_hash) {
        # 将十六进制格式化的哈希值添加到输出对象中
        out.AddMember(StringRef(kHash), Value(fmt::format("{:016X}", m_hash).c_str(), allocator), allocator);
    }
    # 如果不存在哈希值
    else {
        # 将空值添加到输出对象中
        out.AddMember(StringRef(kHash), kNullType, allocator);
    }
    
    # 返回输出对象
    return out;
# 返回给定 benchmark 的大小，单位为字节
uint32_t xmrig::BenchConfig::getSize(const char *benchmark)
{
    # 如果 benchmark 为空，则返回 0
    if (!benchmark) {
        return 0;
    }
    
    # 将 benchmark 转换为无符号长整型数
    const auto size = strtoul(benchmark, nullptr, 10);
    
    # 如果 size 在 1 到 10 之间
    if (size >= 1 && size <= 10) {
        # 如果 benchmark 与格式化后的字符串相等，则返回对应的大小，否则返回 0
        return strcasecmp(benchmark, fmt::format("{}M", size).c_str()) == 0 ? size * 1000000 : 0;
    }
    
    # 如果 size 为 250 或 500
    if (size == 250 || size == 500) {
        # 如果 benchmark 与格式化后的字符串相等，则返回对应的大小，否则返回 0
        return strcasecmp(benchmark, fmt::format("{}K", size).c_str()) == 0 ? size * 1000 : 0;
    }
    
    # 如果定义了 NDEBUG
    # 返回 size 大于等于 10000 则返回 size，否则返回 0
    # 否则返回 0
#   ifndef NDEBUG
    return size >= 10000 ? size : 0;
#   else
    return 0;
#   endif
}
```