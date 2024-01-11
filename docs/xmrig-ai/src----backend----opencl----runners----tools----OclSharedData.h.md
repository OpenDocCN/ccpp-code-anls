# `xmrig\src\backend\opencl\runners\tools\OclSharedData.h`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLSHAREDDATA_H
#define XMRIG_OCLSHAREDDATA_H


#include <memory>
#include <mutex>


using cl_context = struct _cl_context *;
using cl_mem     = struct _cl_mem *;


namespace xmrig {


class Job;


class OclSharedData
{
public:
    // 默认构造函数
    OclSharedData() = default;

    // 创建缓冲区
    cl_mem createBuffer(cl_context context, size_t size, size_t &offset, size_t limit);
    // 调整延迟
    uint64_t adjustDelay(size_t id);
    // 恢复延迟
    uint64_t resumeDelay(size_t id);
    // 释放资源
    void release();
    // 设置恢复计数器
    void setResumeCounter(uint32_t value);
    // 设置运行时间
    void setRunTime(uint64_t time);

    // 返回线程数
    inline size_t threads() const       { return m_threads; }

    // 重载运算符++
    inline OclSharedData &operator++()  { ++m_threads; return *this; }

#   ifdef XMRIG_ALGO_RANDOMX
    // 返回数据集
    cl_mem dataset() const;
    // 创建数据集
    void createDataset(cl_context ctx, const Job &job, bool host);
#   endif

private:
    // 缓冲区
    cl_mem m_buffer           = nullptr;
    // 平均运行时间
    double m_averageRunTime   = 0.0;
    // 阈值
    double m_threshold        = 0.95;
    // 偏移量
    size_t m_offset           = 0;
    // 线程数
    size_t m_threads          = 0;
    // 互斥锁
    std::mutex m_mutex;
    // 恢复计数器
    uint32_t m_resumeCounter  = 0;
    // 时间戳
    uint64_t m_timestamp      = 0;

#   ifdef XMRIG_ALGO_RANDOMX
    # 声明一个名为m_dataset的OpenCL内存对象，并初始化为nullptr
    cl_mem m_dataset          = nullptr;
// 结束if条件判断
#   endif
};

// 结束命名空间xmrig
} /* namespace xmrig */

// 结束if条件判断
#endif /* XMRIG_OCLSHAREDDATA_H */
```