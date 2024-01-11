# `xmrig\src\backend\cuda\runners\CudaRxRunner.h`

```
/* XMRig
 * 版权声明，列出了程序的各个作者和他们的联系方式
 */

#ifndef XMRIG_CUDARXRUNNER_H
#define XMRIG_CUDARXRUNNER_H
// 包含头文件 CudaBaseRunner.h
#include "backend/cuda/runners/CudaBaseRunner.h"

// 命名空间 xmrig
namespace xmrig {

// 类 CudaRxRunner 继承自 CudaBaseRunner
class CudaRxRunner : public CudaBaseRunner
{
public:
    // 构造函数，接受索引和 CudaLaunchData 对象
    CudaRxRunner(size_t index, const CudaLaunchData &data);

protected:
    // 重写基类的 intensity 方法，返回 m_intensity 的值
    inline size_t intensity() const override { return m_intensity; }

    // 重写基类的 run 方法，接受起始 nonce、结果计数和结果 nonce，返回是否运行成功
    bool run(uint32_t startNonce, uint32_t *rescount, uint32_t *resnonce) override;
    // 重写基类的 set 方法，接受作业和数据块，返回是否设置成功
    bool set(const Job &job, uint8_t *blob) override;

private:
    // 成员变量，表示是否准备就绪
    bool m_ready             = false;
    // 常量成员变量，表示数据集是否在主机上
    const bool m_datasetHost = false;
    // 成员变量，表示强度
    size_t m_intensity       = 0;
};

} /* namespace xmrig */

#endif // XMRIG_CUDARXRUNNER_H
```