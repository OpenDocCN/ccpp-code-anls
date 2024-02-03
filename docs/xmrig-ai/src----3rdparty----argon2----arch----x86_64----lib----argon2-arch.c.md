# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-arch.c`

```cpp
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#include "impl-select.h"

#include "argon2-sse2.h"
#include "argon2-ssse3.h"
#include "argon2-xop.h"
#include "argon2-avx2.h"
#include "argon2-avx512f.h"

/* NOTE: there is no portable intrinsic for 64-bit rotate, but any
 * sane compiler should be able to compile this into a ROR instruction: */
#define rotr64(x, n) ((x) >> (n)) | ((x) << (64 - (n)))

#include "argon2-template-64.h"

void fill_segment_default(const argon2_instance_t *instance,
                          argon2_position_t position)
{
    fill_segment_64(instance, position);
}

void argon2_get_impl_list(argon2_impl_list *list)
{
    static const argon2_impl IMPLS[] = {
        { "x86_64",     NULL,                     fill_segment_default },  // 定义 x86_64 平台的默认实现
        { "SSE2",       xmrig_ar2_check_sse2,     xmrig_ar2_fill_segment_sse2 },  // 定义 SSE2 平台的实现
        { "SSSE3",      xmrig_ar2_check_ssse3,    xmrig_ar2_fill_segment_ssse3 },  // 定义 SSSE3 平台的实现
        { "XOP",        xmrig_ar2_check_xop,      xmrig_ar2_fill_segment_xop },  // 定义 XOP 平台的实现
        { "AVX2",       xmrig_ar2_check_avx2,     xmrig_ar2_fill_segment_avx2 },  // 定义 AVX2 平台的实现
        { "AVX-512F",   xmrig_ar2_check_avx512f,  xmrig_ar2_fill_segment_avx512f },  // 定义 AVX-512F 平台的实现
    };

    list->count = sizeof(IMPLS) / sizeof(IMPLS[0]);  // 计算实现列表的数量
    list->entries = IMPLS;  // 将实现列表赋值给传入的列表参数
}
```