# `xmrig\src\crypto\common\portable\mm_malloc.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何担保；甚至没有对特定目的的隐含担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MM_MALLOC_PORTABLE_H
#define XMRIG_MM_MALLOC_PORTABLE_H


#if defined(XMRIG_ARM) && !defined(__clang__)
#include <stdlib.h>


#ifndef __cplusplus
extern
#else
extern "C"
#endif
int posix_memalign(void **__memptr, size_t __alignment, size_t __size);


static __inline__ void *__attribute__((__always_inline__, __malloc__)) _mm_malloc(size_t __size, size_t __align)
{
    if (__align == 1) {
        return malloc(__size);
    }

    if (!(__align & (__align - 1)) && __align < sizeof(void *)) {
        __align = sizeof(void *);
    }

    void *__mallocedMemory;
    if (posix_memalign(&__mallocedMemory, __align, __size)) {
        return nullptr;
    }

    return __mallocedMemory;
}
static __inline__ void __attribute__((__always_inline__)) _mm_free(void *__p)
{
    // 定义一个内联函数，用于释放内存
    free(__p);
}
// 如果是在 Windows 平台，并且不是使用 GCC 编译器
#   include <malloc.h>
// 否则，包含 mm_malloc.h 头文件
#else
#   include <mm_malloc.h>
#endif

// 结束条件编译
#endif /* XMRIG_MM_MALLOC_PORTABLE_H */
```