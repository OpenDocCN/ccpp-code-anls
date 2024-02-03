# `xmrig\src\crypto\rx\Profiler.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，版本 3 或
 *   (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的保证。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_PROFILER_H
#define XMRIG_PROFILER_H


#ifndef FORCE_INLINE
#if defined(_MSC_VER)
#define FORCE_INLINE __forceinline
#elif defined(__GNUC__)
#define FORCE_INLINE __attribute__((always_inline)) inline
#elif defined(__clang__)
#define FORCE_INLINE __inline__
#else
#define FORCE_INLINE
#endif
#endif


#ifdef XMRIG_FEATURE_PROFILING


#include <cstdint>
#include <cstddef>
#include <type_traits>

#if defined(_MSC_VER)
#include <intrin.h>
#endif


// 定义一个函数，用于读取时间戳计数器的值
static FORCE_INLINE uint64_t ReadTSC()
{
#ifdef _MSC_VER
    return __rdtsc();
#else
    uint32_t hi, lo;
    __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
    return (((uint64_t)hi) << 32) | lo;
#endif
}


// 定义一个结构体，用于存储性能分析的数据
struct ProfileScopeData
{
    const char* m_name; // 分析范围的名称
    uint64_t m_totalCycles; // 总周期数
    uint32_t m_totalSamples; // 总样本数

    enum
    {
        MAX_THREAD_ID_LENGTH = 11, // 线程ID的最大长度
        MAX_SAMPLE_COUNT = 128, // 最大样本数
        MAX_DATA_COUNT = 1024 // 最大数据数
    };

    char m_threadId[MAX_THREAD_ID_LENGTH + 1]; // 线程ID

    static ProfileScopeData* s_data[MAX_DATA_COUNT]; // 存储所有分析数据的数组
    static volatile long s_dataCount; // 分析数据的数量
    static double s_tscSpeed; // 时间戳计数器的速度

    static void Register(ProfileScopeData* data); // 注册分析数据
    static void Init(); // 初始化
};
// 确保ProfileScopeData是一个平凡的结构体
static_assert(std::is_trivial<ProfileScopeData>::value, "ProfileScopeData must be a trivial struct");
// 确保ProfileScopeData结构体的大小不超过32字节
static_assert(sizeof(ProfileScopeData) <= 32, "ProfileScopeData struct is too big");

// 定义ProfileScope类
class ProfileScope
{
public:
    // 构造函数，接收一个ProfileScopeData的引用
    FORCE_INLINE ProfileScope(ProfileScopeData& data)
        : m_data(data)
    {
        // 如果m_data的m_totalCycles为0，则将data注册到ProfileScopeData中
        if (m_data.m_totalCycles == 0) {
            ProfileScopeData::Register(&data);
        }

        // 获取当前时间戳，并赋值给m_startCounter
        m_startCounter = ReadTSC();
    }

    // 析构函数
    FORCE_INLINE ~ProfileScope()
    {
        // 计算时间差，并加到m_data的m_totalCycles中
        m_data.m_totalCycles += ReadTSC() - m_startCounter;
        // m_data的m_totalSamples加1
        ++m_data.m_totalSamples;
    }

private:
    // ProfileScopeData的引用
    ProfileScopeData& m_data;
    // 开始计时的时间戳
    uint64_t m_startCounter;
};

// 定义宏PROFILE_SCOPE，用于创建ProfileScope对象
#define PROFILE_SCOPE(x) static thread_local ProfileScopeData x##_data{#x}; ProfileScope x(x##_data);

// 如果没有定义XMRIG_FEATURE_PROFILING，则定义PROFILE_SCOPE为空
#else /* XMRIG_FEATURE_PROFILING */
#define PROFILE_SCOPE(x)
#endif /* XMRIG_FEATURE_PROFILING */

// 包含blake2.h头文件
#include "crypto/randomx/blake2/blake2.h"

// 定义rx_blake2b_wrapper结构体
struct rx_blake2b_wrapper
{
    // 静态函数run，接收输出指针、输出长度、输入指针和输入长度
    FORCE_INLINE static void run(void* out, size_t outlen, const void* in, size_t inlen)
    {
        // 创建ProfileScope对象，名称为RandomX_Blake2b
        PROFILE_SCOPE(RandomX_Blake2b);
        // 调用rx_blake2b函数，计算哈希值
        rx_blake2b(out, outlen, in, inlen);
    }
};

// 结束XMRIG_PROFILER_H的宏定义
#endif /* XMRIG_PROFILER_H */
```