# `xmrig\src\backend\common\Threads.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/common/Threads.h"  // 引入Threads.h头文件
#include "3rdparty/rapidjson/document.h"  // 引入rapidjson库的document头文件
#include "backend/cpu/CpuThreads.h"  // 引入CpuThreads.h头文件
#include "crypto/cn/CnAlgo.h"  // 引入CnAlgo.h头文件

#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/OclThreads.h"  // 如果定义了XMRIG_FEATURE_OPENCL，则引入OclThreads.h头文件
#endif

#ifdef XMRIG_FEATURE_CUDA
#   include "backend/cuda/CudaThreads.h"  // 如果定义了XMRIG_FEATURE_CUDA，则引入CudaThreads.h头文件
#endif

namespace xmrig {  // 命名空间xmrig

static const char *kAsterisk = "*";  // 定义静态常量kAsterisk为"*"

} // 命名空间xmrig结束

template <class T>
const T &xmrig::Threads<T>::get(const String &profileName) const  // 模板函数get的定义，返回类型为T的引用，参数为String类型的引用profileName
{
    static T empty;  // 定义静态变量empty为T类型
    if (profileName.isNull() || !has(profileName)) {  // 如果profileName为空或者不存在于m_profiles中
        return empty;  // 返回empty
    }

    return m_profiles.at(profileName);  // 返回m_profiles中profileName对应的值
}

template <class T>
size_t xmrig::Threads<T>::read(const rapidjson::Value &value)  // 模板函数read的定义，返回类型为size_t，参数为rapidjson库中Value类型的引用value
{
    using namespace rapidjson;  // 使用rapidjson命名空间

    for (auto &member : value.GetObject()) {  // 遍历value对象的成员
        if (member.value.IsArray() || member.value.IsObject()) {  // 如果成员的值是数组或对象
            T threads(member.value);  // 创建T类型的threads对象，参数为成员的值

            if (!threads.isEmpty()) {  // 如果threads不为空
                move(member.name.GetString(), std::move(threads));  // 移动成员的名称和threads对象
            }
        }
    }
}
    // 遍历 JSON 对象的每个成员
    for (auto &member : value.GetObject()) {
        // 如果成员的值是数组或对象，则跳过本次循环
        if (member.value.IsArray() || member.value.IsObject()) {
            continue;
        }

        // 创建算法对象，并检查其有效性
        const Algorithm algo(member.name.GetString());
        if (!algo.isValid()) {
            continue;
        }

        // 如果成员的值是布尔类型且为假，则禁用对应的算法并跳过本次循环
        if (member.value.IsBool() && member.value.IsFalse()) {
            disable(algo);
            continue;
        }

        // 如果成员的值是字符串类型
        if (member.value.IsString()) {
            // 如果已经存在该字符串对应的别名，则将算法和别名插入到别名集合中
            if (has(member.value.GetString())) {
                m_aliases.insert({ algo, member.value.GetString() });
            }
            // 否则将算法插入到禁用集合中
            else {
                m_disabled.insert(algo);
            }
        }
    }

    // 返回配置文件中算法配置的数量
    return m_profiles.size();
// 返回指定算法的配置文件名称，如果严格模式下禁用了该算法，则返回空字符串
template <class T>
xmrig::String xmrig::Threads<T>::profileName(const Algorithm &algorithm, bool strict) const
{
    // 如果算法被禁用，则返回空字符串
    if (isDisabled(algorithm)) {
        return String();
    }

    // 获取算法的名称
    String name = algorithm.name();
    // 如果已经存在该名称的配置文件，则返回该名称
    if (has(name)) {
        return name;
    }

    // 如果存在该算法的别名，则返回该别名对应的配置文件名称
    if (m_aliases.count(algorithm) > 0) {
        return m_aliases.at(algorithm);
    }

    // 如果严格模式下，返回空字符串
    if (strict) {
        return String();
    }

    // 如果算法是 CN 算法且基础算法是 CN_2 且存在 CN_2 算法的配置文件，则返回 CN_2 算法的配置文件名称
    if (algorithm.family() == Algorithm::CN && algorithm.base() == Algorithm::CN_2 && has(Algorithm::kCN_2)) {
        return Algorithm::kCN_2;
    }

    // 如果名称包含 "/"，则获取第一个 "/" 前的名称，如果存在该名称的配置文件，则返回该名称
    if (name.contains("/")) {
        String base = name.split('/').at(0);
        if (has(base)) {
            return base;
        }
    }

    // 如果存在 "*" 算法的配置文件，则返回 "*" 算法的配置文件名称
    if (has(kAsterisk)) {
        return kAsterisk;
    }

    // 返回空字符串
    return String();
}

// 将线程配置转换为 JSON 格式
template <class T>
void xmrig::Threads<T>::toJSON(rapidjson::Value &out, rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 遍历所有配置文件，将其添加到 JSON 对象中
    for (const auto &kv : m_profiles) {
        out.AddMember(kv.first.toJSON(), kv.second.toJSON(doc), allocator);
    }

    // 将禁用的算法添加到 JSON 对象中
    for (const Algorithm &algo : m_disabled) {
        out.AddMember(StringRef(algo.name()), false, allocator);
    }

    // 将算法的别名添加到 JSON 对象中
    for (const auto &kv : m_aliases) {
        out.AddMember(StringRef(kv.first.name()), kv.second.toJSON(), allocator);
    }
}

// 实例化 CpuThreads 类的模板
template class Threads<CpuThreads>;

// 如果开启了 OpenCL 特性，则实例化 OclThreads 类的模板
#ifdef XMRIG_FEATURE_OPENCL
template class Threads<OclThreads>;
#endif

// 如果开启了 CUDA 特性，则实例化 CudaThreads 类的模板
#ifdef XMRIG_FEATURE_CUDA
template class Threads<CudaThreads>;
#endif

} // namespace xmrig
```