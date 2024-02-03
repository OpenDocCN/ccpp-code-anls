# `xmrig\src\net\JobResults.h`

```cpp
/* XMRig
 * 版权声明，列出了程序的各个作者和版权年份
 */

#ifndef XMRIG_JOBRESULTS_H
#define XMRIG_JOBRESULTS_H
// 防止头文件重复包含

#include <cstddef>
#include <cstdint>
// 包含标准库头文件

namespace xmrig {
// 命名空间开始

class IJobResultListener;
class Job;
class JobResult;
// 声明类

class JobResults
{
public:
    static void done(const Job &job);
    // 标记给定的工作任务已完成

    static void setListener(IJobResultListener *listener, bool hwAES);
    // 设置工作结果监听器和硬件AES加速标志

    static void stop();
    // 停止工作结果处理

    static void submit(const Job &job, uint32_t nonce, const uint8_t *result);
    // 提交工作结果，包括工作任务、随机数和计算结果

    static void submit(const Job& job, uint32_t nonce, const uint8_t* result, const uint8_t* miner_signature);
    // 提交带有矿工签名的工作结果

    static void submit(const JobResult &result);
    // 提交工作结果对象

#   if defined(XMRIG_FEATURE_OPENCL) || defined(XMRIG_FEATURE_CUDA)
    // 如果定义了OpenCL或CUDA特性，则编译以下代码
    // 定义一个静态函数，用于提交作业
    static void submit(const Job &job, uint32_t *results, size_t count, uint32_t device_index);
# 结束if条件
};


# 结束命名空间xmrig
} // namespace xmrig

# 结束条件编译指令，关闭XMRIG_JOBRESULTS_H头文件
#endif /* XMRIG_JOBRESULTS_H */
```