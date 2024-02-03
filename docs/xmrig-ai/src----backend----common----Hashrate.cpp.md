# `xmrig\src\backend\common\Hashrate.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <cassert>  // 引入断言库
#include <memory.h>  // 引入内存操作库
#include <cstdio>    // 引入标准输入输出库


#include "backend/common/Hashrate.h"  // 引入Hashrate类的头文件
#include "3rdparty/rapidjson/document.h"  // 引入rapidjson库的document头文件
#include "base/io/json/Json.h"  // 引入Json类的头文件
#include "base/tools/Chrono.h"  // 引入Chrono类的头文件
#include "base/tools/Handle.h"  // 引入Handle类的头文件


inline static const char *format(double h, char *buf, size_t size)  // 定义一个内联函数format，用于格式化输出
{
    if (std::isnormal(h)) {  // 如果h是正常数
        snprintf(buf, size, (h < 100.0) ? "%04.2f" : "%03.1f", h);  // 格式化输出到buf中
        return buf;  // 返回格式化后的字符串
    }

    return "n/a";  // 如果h不是正常数，返回"n/a"
}


xmrig::Hashrate::Hashrate(size_t threads) :  // Hashrate类的构造函数，参数为线程数
    m_threads(threads + 1)  // 初始化m_threads成员变量
{
    m_counts     = new uint64_t*[m_threads];  // 动态分配m_threads个uint64_t*类型的内存
    m_timestamps = new uint64_t*[m_threads];  // 动态分配m_threads个uint64_t*类型的内存
    m_top        = new uint32_t[m_threads];  // 动态分配m_threads个uint32_t类型的内存

    for (size_t i = 0; i < m_threads; i++) {  // 遍历m_threads次
        m_counts[i]     = new uint64_t[kBucketSize]();  // 动态分配kBucketSize个uint64_t类型的内存，并初始化为0
        m_timestamps[i] = new uint64_t[kBucketSize]();  // 动态分配kBucketSize个uint64_t类型的内存，并初始化为0
        m_top[i]        = 0;  // 初始化m_top数组的值为0
    }

    m_earliestTimestamp = std::numeric_limits<uint64_t>::max();  // 初始化m_earliestTimestamp为uint64_t的最大值
    m_totalCount = 0;  // 初始化m_totalCount为0
}


xmrig::Hashrate::~Hashrate()  // Hashrate类的析构函数
{
    # 遍历线程数，释放每个线程的计数和时间戳数组内存
    for (size_t i = 0; i < m_threads; i++) {
        delete [] m_counts[i];
        delete [] m_timestamps[i];
    }
    # 释放存储线程计数的数组内存
    delete [] m_counts;
    # 释放存储线程时间戳的数组内存
    delete [] m_timestamps;
    # 释放存储线程顶部元素的数组内存
    delete [] m_top;
// 计算平均哈希率
double xmrig::Hashrate::average() const
{
    // 获取当前时间戳
    const uint64_t ts = Chrono::steadyMSecs();
    // 如果当前时间戳大于最早时间戳，则计算平均哈希率，否则返回0
    return (ts > m_earliestTimestamp) ? (m_totalCount * 1e3 / (ts - m_earliestTimestamp)) : 0.0;
}

// 格式化哈希率
const char *xmrig::Hashrate::format(double h, char *buf, size_t size)
{
    // 调用format函数格式化哈希率
    return ::format(h, buf, size);
}

// 标准化哈希率
rapidjson::Value xmrig::Hashrate::normalize(double d)
{
    // 调用Json::normalize函数标准化哈希率
    return Json::normalize(d, false);
}

// 将哈希率转换为JSON格式
#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::Hashrate::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建JSON数组对象
    Value out(kArrayType);
    // 将不同时间间隔的哈希率标准化后添加到JSON数组中
    out.PushBack(normalize(calc(ShortInterval)),  allocator);
    out.PushBack(normalize(calc(MediumInterval)), allocator);
    out.PushBack(normalize(calc(LargeInterval)),  allocator);

    return out;
}

// 将特定线程的哈希率转换为JSON格式
rapidjson::Value xmrig::Hashrate::toJSON(size_t threadId, rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建JSON数组对象
    Value out(kArrayType);
    // 将特定线程不同时间间隔的哈希率标准化后添加到JSON数组中
    out.PushBack(normalize(calc(threadId, ShortInterval)),  allocator);
    out.PushBack(normalize(calc(threadId, MediumInterval)), allocator);
    out.PushBack(normalize(calc(threadId, LargeInterval)),  allocator);

    return out;
}
#endif

// 计算特定线程在特定时间间隔内的哈希率
double xmrig::Hashrate::hashrate(size_t index, size_t ms) const
{
    // 确保线程索引在范围内
    assert(index < m_threads);
    // 如果线程索引超出范围，则返回NaN
    if (index >= m_threads) {
        return nan("");
    }

    // 初始化变量
    uint64_t earliestHashCount = 0;
    uint64_t earliestStamp     = 0;
    bool haveFullSet           = false;

    // 计算时间戳限制
    const uint64_t timeStampLimit = xmrig::Chrono::steadyMSecs() - ms;
    uint64_t* timestamps          = m_timestamps[index];
    uint64_t* counts              = m_counts[index];

    // 计算起始索引
    const size_t idx_start  = (m_top[index] - 1) & kBucketMask;
    size_t idx              = idx_start;

    // 获取最新时间戳和哈希计数
    uint64_t lastestStamp   = timestamps[idx];
    uint64_t lastestHashCnt = counts[idx];
}
    # 使用 do-while 循环来处理时间戳数组
    do {
        # 如果时间戳小于时间戳限制
        if (timestamps[idx] < timeStampLimit) {
            # 判断是否有完整的时间戳集合
            haveFullSet = (timestamps[idx] != 0);
            # 如果索引不是起始索引
            if (idx != idx_start) {
                # 更新索引和最早时间戳、最早哈希计数
                idx = (idx + 1) & kBucketMask;
                earliestStamp = timestamps[idx];
                earliestHashCount = counts[idx];
            }
            # 跳出循环
            break;
        }
        # 更新索引为前一个索引
        idx = (idx - 1) & kBucketMask;
    } while (idx != idx_start);

    # 如果没有完整的时间戳集合或者最早时间戳和最晚时间戳为0
    if (!haveFullSet || earliestStamp == 0 || lastestStamp == 0) {
        # 返回 NaN
        return nan("");
    }

    # 如果最晚时间戳减去最早时间戳为0
    if (lastestStamp - earliestStamp == 0) {
        # 返回 NaN
        return nan("");
    }

    # 计算哈希数和时间差
    const auto hashes = static_cast<double>(lastestHashCnt - earliestHashCount);
    const auto time   = static_cast<double>(lastestStamp - earliestStamp) / 1000.0;

    # 返回哈希速率
    return hashes / time;
# 添加数据到哈希率对象中
void xmrig::Hashrate::addData(size_t index, uint64_t count, uint64_t timestamp)
{
    # 获取当前索引对应的顶部位置
    const size_t top         = m_top[index];
    # 将数据计数添加到对应索引的计数数组中
    m_counts[index][top]     = count;
    # 将时间戳添加到对应索引的时间戳数组中
    m_timestamps[index][top] = timestamp;

    # 更新当前索引对应的顶部位置
    m_top[index] = (top + 1) & kBucketMask;

    # 如果索引为0
    if (index == 0) {
        # 如果最早时间戳为最大值，则更新为当前时间戳
        if (m_earliestTimestamp == std::numeric_limits<uint64_t>::max()) {
            m_earliestTimestamp = timestamp;
        }
        # 更新总计数
        m_totalCount = count;
    }
}
```