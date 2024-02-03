# `xmrig\src\backend\opencl\OclThreads.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <algorithm>


#include "backend/opencl/OclThreads.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


// 根据 JSON 值构造 OclThreads 对象
xmrig::OclThreads::OclThreads(const rapidjson::Value &value)
{
    // 如果值是数组
    if (value.IsArray()) {
        // 遍历数组中的每个值
        for (auto &v : value.GetArray()) {
            // 根据值构造 OclThread 对象
            OclThread thread(v);
            // 如果线程对象有效
            if (thread.isValid()) {
                // 添加线程对象到 OclThreads 中
                add(std::move(thread));
            }
        }
    }
}


// 根据设备和算法构造 OclThreads 对象
xmrig::OclThreads::OclThreads(const std::vector<OclDevice> &devices, const Algorithm &algorithm)
{
    // 遍历设备列表
    for (const auto &device : devices) {
        // 生成设备对应的线程对象
        device.generate(algorithm, *this);
    }
}


// 判断当前 OclThreads 对象是否与另一个对象相等
bool xmrig::OclThreads::isEqual(const OclThreads &other) const
{
    // 如果当前对象和另一个对象都为空
    if (isEmpty() && other.isEmpty()) {
        // 返回 true
        return true;
    }
    # 返回当前对象的计数是否等于另一个对象的计数，并且当前对象的数据是否与另一个对象的数据相等
    return count() == other.count() && std::equal(m_data.begin(), m_data.end(), other.m_data.begin());
}
// 将 OclThreads 对象转换为 JSON 格式
rapidjson::Value xmrig::OclThreads::toJSON(rapidjson::Document &doc) const
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个 JSON 数组对象
    Value out(kArrayType);

    // 设置对象为数组类型
    out.SetArray();

    // 遍历 OclThreads 对象中的每个 OclThread 对象，将其转换为 JSON 格式并添加到数组中
    for (const OclThread &thread : m_data) {
        out.PushBack(thread.toJSON(doc), allocator);
    }

    // 返回 JSON 数组对象
    return out;
}
```