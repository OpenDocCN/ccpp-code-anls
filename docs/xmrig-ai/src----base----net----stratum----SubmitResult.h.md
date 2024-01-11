# `xmrig\src\base\net\stratum\SubmitResult.h`

```
#ifndef XMRIG_SUBMITRESULT_H
#define XMRIG_SUBMITRESULT_H

#include "base/tools/Chrono.h"

// 命名空间 xmrig
namespace xmrig {

// SubmitResult 类的定义
class SubmitResult
{
public:
    // 默认构造函数
    SubmitResult() = default;

    // 构造函数，初始化各个成员变量
    inline SubmitResult(int64_t seq, uint64_t diff, uint64_t actualDiff, int64_t reqId, uint32_t backend) :
        reqId(reqId),
        seq(seq),
        backend(backend),
        actualDiff(actualDiff),
        diff(diff),
        m_start(Chrono::steadyMSecs())
    {}

    // 完成函数，计算经过的时间
    inline void done() { elapsed = Chrono::steadyMSecs() - m_start; }

    // 成员变量
    int64_t reqId           = 0;
    int64_t seq             = 0;
    uint32_t backend        = 0;
    uint64_t actualDiff     = 0;
    uint64_t diff           = 0;
    uint64_t elapsed        = 0;

private:
    uint64_t m_start        = 0;
}; 
// 结束匿名命名空间

} /* namespace xmrig */
// 结束 xmrig 命名空间

#endif /* XMRIG_SUBMITRESULT_H */
// 结束 XMRIG_SUBMITRESULT_H 头文件的条件编译
```