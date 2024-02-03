# `xmrig\src\backend\opencl\wrappers\AdlHealth.h`

```cpp
// 定义了 XMRig_AdlHealth 结构体，包含了 GPU 的健康信息
#ifndef XMRIG_ADLHEALTH_H
#define XMRIG_ADLHEALTH_H

// 包含了必要的头文件
#include <cstdint>
#include <vector>

// 定义了 AdlHealth 结构体，包含了 GPU 的健康信息
struct AdlHealth
{
    uint32_t clock          = 0;  // GPU 的时钟频率
    uint32_t memClock       = 0;  // GPU 内存的时钟频率
    uint32_t power          = 0;  // GPU 的功耗
    uint32_t rpm            = 0;  // GPU 风扇的转速
    uint32_t temperature    = 0;  // GPU 的温度
};

#endif /* XMRIG_ADLHEALTH_H */
```