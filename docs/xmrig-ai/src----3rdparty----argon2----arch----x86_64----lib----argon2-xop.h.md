# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-xop.h`

```
// 如果 ARGON2_XOP_H 未定义，则定义 ARGON2_XOP_H
#ifndef ARGON2_XOP_H
#define ARGON2_XOP_H

// 包含 "core.h" 文件
#include "core.h"

// 声明函数 xmrig_ar2_fill_segment_xop，接受 argon2_instance_t 类型指针参数 instance 和 argon2_position_t 类型参数 position，无返回值
void xmrig_ar2_fill_segment_xop(const argon2_instance_t *instance, argon2_position_t position);
// 声明函数 xmrig_ar2_check_xop，无参数，返回整型
int xmrig_ar2_check_xop(void);

// 结束条件编译指令 ARGON2_XOP_H
#endif // ARGON2_XOP_H
```