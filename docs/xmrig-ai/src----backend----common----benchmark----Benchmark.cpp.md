# `xmrig\src\backend\common\benchmark\Benchmark.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/common/benchmark/Benchmark.h"  // 引入Benchmark类的头文件
#include "backend/common/benchmark/BenchState.h"  // 引入BenchState类的头文件
#include "base/io/log/Log.h"  // 引入Log类的头文件
#include "base/io/log/Tags.h"  // 引入Tags类的头文件
#include "base/tools/Chrono.h"  // 引入Chrono类的头文件


#include <cinttypes>  // 引入cinttypes标准库


xmrig::Benchmark::Benchmark(size_t workers, const IBackend *backend) :  // Benchmark类的构造函数，接受工作线程数和后端对象指针作为参数
    m_backend(backend),  // 初始化成员变量m_backend
    m_workers(workers)  // 初始化成员变量m_workers
{
}


bool xmrig::Benchmark::finish(uint64_t totalHashCount)  // 完成基准测试，接受总哈希计数作为参数
{
    m_current = totalHashCount;  // 将当前哈希计数设置为传入的总哈希计数

    return BenchState::isDone();  // 返回基准状态是否完成的布尔值
}


void xmrig::Benchmark::start()  // 开始基准测试
{
    m_startTime = BenchState::start(m_workers, m_backend);  // 将开始时间设置为基准状态的开始时间
}


void xmrig::Benchmark::printProgress() const  // 打印进度的函数
{
    if (!m_startTime || !m_current) {  // 如果开始时间或当前哈希计数为0，则返回
        return;
    }

    const double dt      = static_cast<double>(Chrono::steadyMSecs() - m_startTime) / 1000.0;  // 计算时间间隔
    const double percent = static_cast<double>(m_current) / BenchState::size() * 100.0;  // 计算完成百分比

    LOG_NOTICE("%s " MAGENTA_BOLD("%5.2f%% ") CYAN_BOLD("%" PRIu64) CYAN("/%u") BLACK_BOLD(" (%.3fs)"), Tags::bench(), percent, m_current, BenchState::size(), dt);  // 打印进度信息
}
```