# `xmrig\src\backend\cpu\CpuThreads.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <algorithm>


#include "backend/cpu/CpuThreads.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


namespace xmrig {


static const char *kAffinity    = "affinity";
static const char *kIntensity   = "intensity";
static const char *kThreads     = "threads";


static inline int64_t getAffinityMask(const rapidjson::Value &value)
{
    if (value.IsInt64()) {
        return value.GetInt64();
    }

    if (value.IsString()) {
        const char *arg = value.GetString();
        const char *p   = strstr(arg, "0x");

        return p ? strtoll(p, nullptr, 16) : strtoll(arg, nullptr, 10);
    }

    return -1L;
}


static inline int64_t getAffinity(uint64_t index, int64_t affinity)
{
    if (affinity == -1L) {
        return -1L;
    }

    size_t idx = 0;

    for (size_t i = 0; i < 64; i++) {
        if (!(static_cast<uint64_t>(affinity) & (1ULL << i))) {
            continue;
        }

        if (idx == index) {
            return static_cast<int64_t>(i);
        }

        idx++;
    }

    return -1L;
}


} // namespace xmrig


xmrig::CpuThreads::CpuThreads(const rapidjson::Value &value)
{
    # 如果值是一个数组
    if (value.IsArray()) {
        # 遍历数组中的每个元素
        for (auto &v : value.GetArray()) {
            # 创建一个 CpuThread 对象
            CpuThread thread(v);
            # 如果线程对象有效
            if (thread.isValid()) {
                # 将线程对象添加到列表中
                add(thread);
            }
        }
    }
    # 如果值是一个对象
    else if (value.IsObject()) {
        # 从 JSON 对象中获取强度值，默认为 1
        uint32_t intensity   = Json::getUint(value, kIntensity, 1);
        # 从 JSON 对象中获取线程数，最大不超过 1024
        const size_t threads = std::min<unsigned>(Json::getUint(value, kThreads), 1024);
        # 获取线程亲和性掩码
        m_affinity           = getAffinityMask(Json::getValue(value, kAffinity));
        # 设置对象格式
        m_format             = ObjectFormat;

        # 如果强度值小于 1 或大于 5，则将其设置为 1
        if (intensity < 1 || intensity > 5) {
            intensity = 1;
        }

        # 根据线程数和亲和性添加线程
        for (size_t i = 0; i < threads; ++i) {
            add(getAffinity(i, m_affinity), intensity);
        }
    }
# 定义了一个名为CpuThreads的类，构造函数接受线程数量和强度参数
xmrig::CpuThreads::CpuThreads(size_t count, uint32_t intensity)
{
    # 为数据成员m_data预留count个空间
    m_data.reserve(count);

    # 循环count次，向m_data中添加CpuThread对象，其中线程号为-1，强度为intensity
    for (size_t i = 0; i < count; ++i) {
        add(-1, intensity);
    }
}

# 比较当前CpuThreads对象和另一个CpuThreads对象是否相等
bool xmrig::CpuThreads::isEqual(const CpuThreads &other) const
{
    # 如果当前对象和另一个对象都为空，则返回true
    if (isEmpty() && other.isEmpty()) {
        return true;
    }

    # 返回当前对象的线程数量和另一个对象的线程数量是否相等，以及m_data和other.m_data是否相等
    return count() == other.count() && std::equal(m_data.begin(), m_data.end(), other.m_data.begin());
}

# 将CpuThreads对象转换为JSON格式
rapidjson::Value xmrig::CpuThreads::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    # 创建一个JSON对象
    Value out;

    # 如果m_format为ArrayFormat，则设置out为数组类型，并将m_data中的每个线程对象转换为JSON并添加到out中
    if (m_format == ArrayFormat) {
        out.SetArray();

        for (const CpuThread &thread : m_data) {
            out.PushBack(thread.toJSON(doc), allocator);
        }
    }
    # 如果m_format不为ArrayFormat，则设置out为对象类型，并添加强度、线程数量和亲和性信息到out中
    else {
        out.SetObject();

        out.AddMember(StringRef(kIntensity), m_data.empty() ? 1 : m_data.front().intensity(), allocator);
        out.AddMember(StringRef(kThreads), static_cast<unsigned>(m_data.size()), allocator);
        out.AddMember(StringRef(kAffinity), m_affinity, allocator);
    }

    # 返回生成的JSON对象
    return out;
}
```