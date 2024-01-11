# `xmrig\src\backend\cuda\wrappers\NvmlHealth.h`

```
// XMRig程序的版权声明
// 定义XMRig_NVMLHEALTH_H的宏，用于条件编译
#ifndef XMRIG_NVMLHEALTH_H
#define XMRIG_NVMLHEALTH_H

// 包含必要的头文件
#include <cstdint>  // 包含对应的C++标准库头文件
#include <vector>   // 包含对应的C++标准库头文件

// 定义NvmlHealth结构体，包含风扇速度、时钟、内存时钟、功率和温度
struct NvmlHealth
{
    std::vector<uint32_t> fanSpeed;  // 风扇速度的向量
    uint32_t clock          = 0;     // 时钟
    uint32_t memClock       = 0;     // 内存时钟
    uint32_t power          = 0;     // 功率
    uint32_t temperature    = 0;     // 温度
};

// 结束条件编译
#endif /* XMRIG_NVMLHEALTH_H */
```