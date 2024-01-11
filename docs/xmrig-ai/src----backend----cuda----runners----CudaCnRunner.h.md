# `xmrig\src\backend\cuda\runners\CudaCnRunner.h`

```
/* XMRig
 * 版权声明，列出了程序的各个作者和他们的联系方式
 */

#ifndef XMRIG_CUDACNRUNNER_H
#define XMRIG_CUDACNRUNNER_H
// 包含头文件 CudaBaseRunner.h
#include "backend/cuda/runners/CudaBaseRunner.h"

// 命名空间 xmrig
namespace xmrig {

// 定义类 CudaCnRunner，继承自 CudaBaseRunner
class CudaCnRunner : public CudaBaseRunner
{
public:
    // 构造函数，接受索引和 CudaLaunchData 对象
    CudaCnRunner(size_t index, const CudaLaunchData &data);

protected:
    // 重写基类的 run 方法，接受起始 nonce、结果计数和结果 nonce
    bool run(uint32_t startNonce, uint32_t *rescount, uint32_t *resnonce) override;
};

} /* namespace xmrig */

#endif // XMRIG_CUDACNRUNNER_H
```