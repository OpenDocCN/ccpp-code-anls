# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-avx512f.h`

```cpp
#ifndef ARGON2_AVX512F_H
#define ARGON2_AVX512F_H
// 如果 ARGON2_AVX512F_H 未定义，则定义 ARGON2_AVX512F_H

#include "core.h"
// 包含 core.h 文件

void xmrig_ar2_fill_segment_avx512f(const argon2_instance_t *instance, argon2_position_t position);
// 声明一个名为 xmrig_ar2_fill_segment_avx512f 的函数，接受 argon2_instance_t 类型指针和 argon2_position_t 类型参数

int xmrig_ar2_check_avx512f(void);
// 声明一个名为 xmrig_ar2_check_avx512f 的函数，不接受任何参数

#endif // ARGON2_AVX512F_H
// 结束条件编译指令，如果 ARGON2_AVX512F_H 已定义，则结束条件编译指令
```