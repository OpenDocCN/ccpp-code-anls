# `xmrig\src\backend\opencl\OclThread.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/OclThread.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


#include <algorithm>


namespace xmrig {

static const char *kIndex        = "index";
static const char *kIntensity    = "intensity";
static const char *kStridedIndex = "strided_index";
static const char *kThreads      = "threads";
static const char *kUnroll       = "unroll";
static const char *kWorksize     = "worksize";

#ifdef XMRIG_ALGO_RANDOMX
static const char *kBFactor      = "bfactor";
static const char *kGCNAsm       = "gcn_asm";
static const char* kDatasetHost  = "dataset_host";
#endif

} // namespace xmrig


xmrig::OclThread::OclThread(const rapidjson::Value &value)
{
    // 如果值不是对象，则返回
    if (!value.IsObject()) {
        return;
    }

    // 从JSON对象中获取index值并赋给m_index
    m_index         = Json::getUint(value, kIndex);
    // 从JSON对象中获取worksize值，并限制在1到512之间，赋给m_worksize
    m_worksize      = std::max(std::min(Json::getUint(value, kWorksize), 512U), 1U);
    // 从JSON对象中获取unroll值，并限制在1到128之间，赋给m_unrollFactor
    m_unrollFactor  = std::max(std::min(Json::getUint(value, kUnroll, m_unrollFactor), 128U), 1U);

    // 设置强度值
    setIntensity(Json::getUint(value, kIntensity));

    // 从JSON对象中获取strided_index数组
    const auto &si = Json::getArray(value, kStridedIndex);
    # 检查si是否为数组并且大小大于等于2
    if (si.IsArray() && si.Size() >= 2) {
        # 如果满足条件，将si数组的第一个元素和第二个元素的最小值赋给m_stridedIndex和m_memChunk
        m_stridedIndex = std::min(si[0].GetUint(), 2U);
        m_memChunk     = std::min(si[1].GetUint(), 18U);
    }
    else {
        # 如果不满足条件，将m_stridedIndex和m_memChunk都赋值为0，并将STRIDED_INDEX_FIELD字段设置为false
        m_stridedIndex = 0;
        m_memChunk     = 0;
        m_fields.set(STRIDED_INDEX_FIELD, false);
    }

    # 获取value中名为kThreads的数组
    const auto &threads = Json::getArray(value, kThreads);
    # 检查threads是否为数组
    if (threads.IsArray()) {
        # 如果是数组，预留空间给m_threads
        m_threads.reserve(threads.Size());

        # 遍历threads数组中的元素，将每个元素转换为int64并添加到m_threads中
        for (const auto &affinity : threads.GetArray()) {
            m_threads.emplace_back(affinity.GetInt64());
        }
    }

    # 如果m_threads为空，添加一个值为-1的元素
    if (m_threads.empty()) {
        m_threads.emplace_back(-1);
    }
// 如果定义了 XMRIG_ALGO_RANDOMX，则执行以下代码块
#ifdef XMRIG_ALGO_RANDOMX
    // 获取配置中的 GCNAsm 值
    const auto &gcnAsm = Json::getValue(value, kGCNAsm);
    // 如果 GCNAsm 是布尔类型
    if (gcnAsm.IsBool()) {
        // 设置 RANDOMX_FIELDS 为 true
        m_fields.set(RANDOMX_FIELDS, true);

        // 将 GCNAsm 转换为布尔值并赋给 m_gcnAsm
        m_gcnAsm      = gcnAsm.GetBool();
        // 获取配置中的 BFactor 值，如果不存在则使用默认值 m_bfactor
        m_bfactor     = Json::getUint(value, kBFactor, m_bfactor);
        // 获取配置中的 DatasetHost 值，如果不存在则使用默认值 m_datasetHost
        m_datasetHost = Json::getBool(value, kDatasetHost, m_datasetHost);
    }
#endif
}


// 比较当前 OclThread 对象与另一个 OclThread 对象是否相等
bool xmrig::OclThread::isEqual(const OclThread &other) const
{
    // 检查线程数是否相等，并且每个线程的值也相等
    return other.m_threads.size() == m_threads.size() &&
           std::equal(m_threads.begin(), m_threads.end(), other.m_threads.begin()) &&
           other.m_bfactor      == m_bfactor &&
           other.m_datasetHost  == m_datasetHost &&
           other.m_gcnAsm       == m_gcnAsm &&
           other.m_index        == m_index &&
           other.m_intensity    == m_intensity &&
           other.m_memChunk     == m_memChunk &&
           other.m_stridedIndex == m_stridedIndex &&
           other.m_unrollFactor == m_unrollFactor &&
           other.m_worksize     == m_worksize;
}


// 将当前 OclThread 对象转换为 JSON 格式
rapidjson::Value xmrig::OclThread::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    // 获取 JSON 文档的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个空的 JSON 对象
    Value out(kObjectType);

    // 添加索引、强度和工作大小到 JSON 对象中
    out.AddMember(StringRef(kIndex),        index(), allocator);
    out.AddMember(StringRef(kIntensity),    intensity(), allocator);
    out.AddMember(StringRef(kWorksize),     worksize(), allocator);

    // 如果包含 STRIDED_INDEX_FIELD
    if (m_fields.test(STRIDED_INDEX_FIELD)) {
        // 创建一个包含 stridedIndex 和 memChunk 的数组，并添加到 JSON 对象中
        Value si(kArrayType);
        si.Reserve(2, allocator);
        si.PushBack(stridedIndex(), allocator);
        si.PushBack(memChunk(), allocator);
        out.AddMember(StringRef(kStridedIndex), si, allocator);
    }

    // 创建一个线程数组，并将每个线程的值添加到数组中
    Value threads(kArrayType);
    threads.Reserve(m_threads.size(), allocator);

    for (auto thread : m_threads) {
        threads.PushBack(thread, allocator);
    }

    // 将线程数组添加到 JSON 对象中
    out.AddMember(StringRef(kThreads), threads, allocator);

    // 如果包含 RANDOMX_FIELDS
    if (m_fields.test(RANDOMX_FIELDS)) {
# 如果定义了 XMRIG_ALGO_RANDOMX，则执行以下代码块
        # 将 bfactor() 的返回值添加到 JSON 对象 out 中
        out.AddMember(StringRef(kBFactor),      bfactor(), allocator);
        # 将 isAsm() 的返回值添加到 JSON 对象 out 中
        out.AddMember(StringRef(kGCNAsm),       isAsm(), allocator);
        # 将 isDatasetHost() 的返回值添加到 JSON 对象 out 中
        out.AddMember(StringRef(kDatasetHost),  isDatasetHost(), allocator);
# 结束 ifdef XMRIG_ALGO_RANDOMX 的条件判断

# 如果不满足条件 m_fields.test(KAWPOW_FIELDS)，则执行以下代码块
    # 将 unrollFactor() 的返回值添加到 JSON 对象 out 中
    out.AddMember(StringRef(kUnroll), unrollFactor(), allocator);

# 返回 JSON 对象 out
return out;
```