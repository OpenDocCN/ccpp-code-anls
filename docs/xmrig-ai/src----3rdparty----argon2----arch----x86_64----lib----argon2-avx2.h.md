# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-avx2.h`

```
// 如果 ARGON2_AVX2_H 未定义，则定义 ARGON2_AVX2_H
#ifndef ARGON2_AVX2_H
#define ARGON2_AVX2_H

// 包含 core.h 文件
#include "core.h"

// 声明函数 xmrig_ar2_fill_segment_avx2，接受 argon2_instance_t 类型指针和 argon2_position_t 类型参数
void xmrig_ar2_fill_segment_avx2(const argon2_instance_t *instance, argon2_position_t position);
// 声明函数 xmrig_ar2_check_avx2，不接受任何参数
int xmrig_ar2_check_avx2(void);

// 结束条件编译指令，结束定义 ARGON2_AVX2_H
#endif // ARGON2_AVX2_H
```