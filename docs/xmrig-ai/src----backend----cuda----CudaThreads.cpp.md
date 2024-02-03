# `xmrig\src\backend\cuda\CudaThreads.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的第3版或（在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参阅
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/CudaThreads.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


#include <algorithm>


// 根据给定的rapidjson::Value对象构造CudaThreads对象
xmrig::CudaThreads::CudaThreads(const rapidjson::Value &value)
{
    // 如果value是数组
    if (value.IsArray()) {
        // 遍历数组中的每个元素
        for (auto &v : value.GetArray()) {
            // 根据数组元素构造CudaThread对象
            CudaThread thread(v);
            // 如果CudaThread对象有效
            if (thread.isValid()) {
                // 将CudaThread对象添加到CudaThreads对象中
                add(thread);
            }
        }
    }
}

// 根据给定的CudaDevice对象和Algorithm对象构造CudaThreads对象
xmrig::CudaThreads::CudaThreads(const std::vector<CudaDevice> &devices, const Algorithm &algorithm)
{
    // 遍历设备列表中的每个设备
    for (const auto &device : devices) {
        // 根据设备和算法生成CudaThread对象，并添加到CudaThreads对象中
        device.generate(algorithm, *this);
    }
}

// 判断当前CudaThreads对象是否与另一个CudaThreads对象相等
bool xmrig::CudaThreads::isEqual(const CudaThreads &other) const
{
    // 如果当前对象和另一个对象都为空
    if (isEmpty() && other.isEmpty()) {
        return true;
    }

    // 返回当前对象和另一个对象的CudaThread对象数量是否相等，并且每个CudaThread对象是否相等
    return count() == other.count() && std::equal(m_data.begin(), m_data.end(), other.m_data.begin());
}

// 将CudaThreads对象转换为JSON格式
rapidjson::Value xmrig::CudaThreads::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建一个JSON数组对象
    Value out(kArrayType);

    // 设置数组类型
    out.SetArray();

    // 遍历CudaThreads对象中的每个CudaThread对象
    for (const CudaThread &thread : m_data) {
        // 将每个CudaThread对象转换为JSON格式，并添加到数组中
        out.PushBack(thread.toJSON(doc), allocator);
    }
}
    # 返回变量 out 的值
    return out;
# 闭合前面的函数定义
```