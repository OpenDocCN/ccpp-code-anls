# `xmrig\src\3rdparty\argon2\lib\impl-select.h`

```
#ifndef ARGON2_IMPL_SELECT_H
#define ARGON2_IMPL_SELECT_H
// 如果 ARGON2_IMPL_SELECT_H 未定义，则定义 ARGON2_IMPL_SELECT_H

#include "core.h"
// 包含 core.h 文件

typedef struct Argon2_impl {
    const char *name;
    int (*check)(void);
    void (*fill_segment)(const argon2_instance_t *instance,
                         argon2_position_t position);
} argon2_impl;
// 定义 Argon2_impl 结构体，包含 name、check 和 fill_segment 三个成员

typedef struct Argon2_impl_list {
    const argon2_impl *entries;
    size_t count;
} argon2_impl_list;
// 定义 Argon2_impl_list 结构体，包含 entries 和 count 两个成员

void argon2_get_impl_list(argon2_impl_list *list);
void fill_segment_default(const argon2_instance_t *instance,
                          argon2_position_t position);
// 声明 argon2_get_impl_list 和 fill_segment_default 函数

#endif // ARGON2_IMPL_SELECT_H
// 结束条件编译指令，如果 ARGON2_IMPL_SELECT_H 已定义，则结束定义
```