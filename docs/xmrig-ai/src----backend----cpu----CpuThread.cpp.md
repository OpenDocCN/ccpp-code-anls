# `xmrig\src\backend\cpu\CpuThread.cpp`

```
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
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/cpu/CpuThread.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


xmrig::CpuThread::CpuThread(const rapidjson::Value &value)
{
    // 如果值是数组并且大小至少为2
    if (value.IsArray() && value.Size() >= 2) {
        // 获取数组中的第一个元素作为强度
        m_intensity = value[0].GetUint();
        // 获取数组中的第二个元素作为亲和性
        m_affinity  = value[1].GetInt();
    }
    // 如果值是整数
    else if (value.IsInt()) {
        // 强度为0
        m_intensity = 0;
        // 亲和性为值本身
        m_affinity  = value.GetInt();
    }
}


rapidjson::Value xmrig::CpuThread::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    // 如果强度为0
    if (m_intensity == 0) {
        // 返回亲和性值
        return Value(m_affinity);
    }

    auto &allocator = doc.GetAllocator();

    // 创建一个数组值
    Value out(kArrayType);
    // 将强度和亲和性添加到数组中
    out.PushBack(m_intensity, allocator);
    out.PushBack(m_affinity, allocator);

    return out;
}
```