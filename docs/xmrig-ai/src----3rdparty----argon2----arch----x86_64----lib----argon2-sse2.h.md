# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-sse2.h`

```cpp
// 如果未定义 ARGON2_SSE2_H，则定义 ARGON2_SSE2_H
#ifndef ARGON2_SSE2_H
#define ARGON2_SSE2_H

// 包含 "core.h" 头文件
#include "core.h"

// 声明函数 xmrig_ar2_fill_segment_sse2，接受 argon2_instance_t 类型指针参数和 argon2_position_t 类型参数
void xmrig_ar2_fill_segment_sse2(const argon2_instance_t *instance, argon2_position_t position);
// 声明函数 xmrig_ar2_check_sse2，返回整型
int xmrig_ar2_check_sse2(void);

// 结束条件，如果定义了 ARGON2_SSE2_H，则结束定义
#endif // ARGON2_SSE2_H
```