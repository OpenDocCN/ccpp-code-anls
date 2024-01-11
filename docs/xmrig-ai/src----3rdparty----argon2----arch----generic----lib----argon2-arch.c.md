# `xmrig\src\3rdparty\argon2\arch\generic\lib\argon2-arch.c`

```
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#include "impl-select.h"

// 定义一个64位整数的循环右移宏
#define rotr64(x, n) (((x) >> (n)) | ((x) << (64 - (n))))

#include "argon2-template-64.h"

// 填充默认段的函数，使用64位实现
void fill_segment_default(const argon2_instance_t *instance,
                          argon2_position_t position)
{
    fill_segment_64(instance, position);
}

// 获取实现列表的函数，暂时没有实现
void argon2_get_impl_list(argon2_impl_list *list)
{
    list->count = 0;
}
```