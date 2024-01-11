# `xmrig\src\backend\opencl\runners\OclBaseRunner.h`

```
/* XMRig
 * 版权声明
 * 该程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，可以选择遵循许可证的第三版或者之后的版本。
 * 该程序是基于希望它能有用的目的分发的，但没有任何保证；甚至没有暗示的保证，包括适销性或特定用途的保证。更多细节请参阅 GNU 通用公共许可证。
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请访问 http://www.gnu.org/licenses/。
 */

#ifndef XMRIG_OCLBASERUNNER_H
#define XMRIG_OCLBASERUNNER_H

#include <string>

#include "3rdparty/cl.h"
#include "backend/opencl/interfaces/IOclRunner.h"
#include "base/crypto/Algorithm.h"

namespace xmrig {

class OclLaunchData;

class OclBaseRunner : public IOclRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclBaseRunner)

    // 构造函数，初始化 OclBaseRunner 对象
    OclBaseRunner(size_t id, const OclLaunchData &data);
    // 析构函数，释放 OclBaseRunner 对象
    ~OclBaseRunner() override;

protected:
    // 获取 OpenCL 上下文
    inline cl_context ctx() const override                { return m_ctx; }
    // 获取算法
    inline const Algorithm &algorithm() const override    { return m_algorithm; }
    // 获取构建选项
    inline const char *buildOptions() const override      { return m_options.c_str(); }
    // 返回设备密钥
    inline const char *deviceKey() const override         { return m_deviceKey.c_str(); }
    // 返回来源
    inline const char *source() const override            { return m_source; }
    // 返回 OclLaunchData 对象的引用
    inline const OclLaunchData &data() const override     { return m_data; }
    // 返回强度
    inline size_t intensity() const override              { return m_intensity; }
    // 返回线程 ID
    inline size_t threadId() const override               { return m_threadId; }
    // 返回轮大小
    inline uint32_t roundSize() const override            { return m_intensity; }
    // 返回已处理的哈希数量
    inline uint32_t processedHashes() const override      { return m_intensity; }
    // 通知作业提前完成
    inline void jobEarlyNotification(const Job&) override {}

    // 返回缓冲区大小
    size_t bufferSize() const override;
    // 返回设备索引
    uint32_t deviceIndex() const override;
    // 构建
    void build() override;
    // 初始化
    void init() override;
protected:
    // 创建子缓冲区，使用给定的标志和大小
    cl_mem createSubBuffer(cl_mem_flags flags, size_t size);
    // 对齐给定大小的内存
    size_t align(size_t size) const;
    // 将缓冲区的数据读取到指定的内存指针中
    void enqueueReadBuffer(cl_mem buffer, cl_bool blocking_read, size_t offset, size_t size, void *ptr);
    // 将数据写入缓冲区
    void enqueueWriteBuffer(cl_mem buffer, cl_bool blocking_write, size_t offset, size_t size, const void *ptr);
    // 完成并输出哈希输出
    void finalize(uint32_t *hashOutput);

    cl_command_queue m_queue    = nullptr;  // OpenCL 命令队列
    cl_context m_ctx;  // OpenCL 上下文
    cl_mem m_buffer             = nullptr;  // OpenCL 缓冲区
    cl_mem m_input              = nullptr;  // OpenCL 输入缓冲区
    cl_mem m_output             = nullptr;  // OpenCL 输出缓冲区
    cl_program m_program        = nullptr;  // OpenCL 程序
    const Algorithm m_algorithm;  // 算法类型
    const char *m_source;  // OpenCL 源码
    const OclLaunchData &m_data;  // OpenCL 启动数据
    const size_t m_align;  // 内存对齐大小
    const size_t m_threadId;  // 线程 ID
    const uint32_t m_intensity;  // 强度
    size_t m_offset             = 0;  // 偏移量
    std::string m_deviceKey;  // 设备密钥
    std::string m_options;  // 选项
};


} /* namespace xmrig */


#endif // XMRIG_OCLBASERUNNER_H
```