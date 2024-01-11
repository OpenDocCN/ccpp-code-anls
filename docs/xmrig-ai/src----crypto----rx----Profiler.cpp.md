# `xmrig\src\crypto\rx\Profiler.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "crypto/rx/Profiler.h"  // 导入Profiler.h文件
#include "base/io/log/Log.h"  // 导入Log.h文件
#include "base/io/log/Tags.h"  // 导入Tags.h文件


#include <cstring>  // 导入cstring标准库
#include <sstream>  // 导入sstream标准库
#include <thread>  // 导入thread标准库
#include <chrono>  // 导入chrono标准库
#include <algorithm>  // 导入algorithm标准库


#ifdef XMRIG_FEATURE_PROFILING  // 如果定义了XMRIG_FEATURE_PROFILING


ProfileScopeData* ProfileScopeData::s_data[MAX_DATA_COUNT] = {};  // 初始化ProfileScopeData类的s_data数组
volatile long ProfileScopeData::s_dataCount = 0;  // 初始化ProfileScopeData类的s_dataCount变量
double ProfileScopeData::s_tscSpeed = 0.0;  // 初始化ProfileScopeData类的s_tscSpeed变量


#ifndef NOINLINE  // 如果没有定义NOINLINE
#ifdef __GNUC__  // 如果定义了__GNUC__
#define NOINLINE __attribute__ ((noinline))  // 定义NOINLINE为__attribute__ ((noinline))
#elif _MSC_VER  // 如果定义了_MSC_VER
#define NOINLINE __declspec(noinline)  // 定义NOINLINE为__declspec(noinline)
#else  // 如果都没有定义
#define NOINLINE  // 空定义NOINLINE
#endif  // 结束条件编译
#endif  // 结束条件编译


static std::string get_thread_id()  // 定义get_thread_id函数，返回类型为std::string
{
    std::stringstream ss;  // 创建一个字符串流对象ss
    ss << std::this_thread::get_id();  // 将当前线程的ID写入字符串流

    std::string s = ss.str();  // 将字符串流转换为字符串s
    if (s.length() > ProfileScopeData::MAX_THREAD_ID_LENGTH) {  // 如果字符串长度大于MAX_THREAD_ID_LENGTH
        s.resize(ProfileScopeData::MAX_THREAD_ID_LENGTH);  // 调整字符串长度为MAX_THREAD_ID_LENGTH
    }

    return s;  // 返回字符串s
}


NOINLINE void ProfileScopeData::Register(ProfileScopeData* data)  // 定义Register函数，返回类型为void，参数为ProfileScopeData指针
{
#ifdef _MSC_VER  // 如果定义了_MSC_VER
    const long id = _InterlockedIncrement(&s_dataCount) - 1;  // 使用原子操作递增s_dataCount，并将结果减1赋值给id
#else  // 如果没有定义_MSC_VER
    const long id = __sync_fetch_and_add(&s_dataCount, 1);  // 使用原子操作递增s_dataCount，并将结果赋值给id
#endif  // 结束条件编译
    # 检查 id 是否小于 MAX_DATA_COUNT，并进行类型转换为无符号长整型
    if (static_cast<unsigned long>(id) < MAX_DATA_COUNT) {
        # 将数据存储到 s_data 数组中的指定位置
        s_data[id] = data;

        # 获取当前线程的 ID，并将其拷贝到数据对象的 m_threadId 字段中
        const std::string s = get_thread_id();
        memcpy(data->m_threadId, s.c_str(), s.length() + 1);
    }
# 定义一个名为ProfileScopeData的类的Init方法，该方法不会被内联
NOINLINE void ProfileScopeData::Init()
{
    # 引入std::chrono命名空间
    using namespace std::chrono;

    # 获取当前时间的纳秒数，并将其转换为uint64_t类型
    const uint64_t t1 = static_cast<uint64_t>(time_point_cast<nanoseconds>(high_resolution_clock::now()).time_since_epoch().count());
    # 调用ReadTSC函数获取时间戳计数，并将其转换为uint64_t类型
    const uint64_t count1 = ReadTSC();

    # 无限循环
    for (;;)
    {
        # 获取当前时间的纳秒数，并将其转换为uint64_t类型
        const uint64_t t2 = static_cast<uint64_t>(time_point_cast<nanoseconds>(high_resolution_clock::now()).time_since_epoch().count());
        # 调用ReadTSC函数获取时间戳计数，并将其转换为uint64_t类型
        const uint64_t count2 = ReadTSC();

        # 如果t2减去t1大于1000000000
        if (t2 - t1 > 1000000000) {
            # 计算TSC速度，并将结果赋值给s_tscSpeed
            s_tscSpeed = (count2 - count1) * 1e9 / (t2 - t1);
            # 输出日志信息，包括标签和TSC速度
            LOG_INFO("%s TSC speed = %.3f GHz", xmrig::Tags::profiler(), s_tscSpeed / 1e9);
            # 返回
            return;
        }
    }
}


#endif /* XMRIG_FEATURE_PROFILING */
```