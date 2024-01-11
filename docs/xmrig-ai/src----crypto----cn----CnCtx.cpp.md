# `xmrig\src\crypto\cn\CnCtx.cpp`

```
/*
 * XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <limits>


#include "crypto/cn/CnCtx.h"
#include "base/crypto/Algorithm.h"
#include "crypto/cn/CryptoNight.h"
#include "crypto/common/portable/mm_malloc.h"
#include "crypto/common/VirtualMemory.h"


void xmrig::CnCtx::create(cryptonight_ctx **ctx, uint8_t *memory, size_t size, size_t count)
{
    for (size_t i = 0; i < count; ++i) {
        auto *c     = static_cast<cryptonight_ctx *>(_mm_malloc(sizeof(cryptonight_ctx), 4096)); // 分配内存空间
        c->memory   = memory + (i * size); // 设置内存地址

        c->generated_code              = reinterpret_cast<cn_mainloop_fun_ms_abi>(VirtualMemory::allocateExecutableMemory(0x4000, false)); // 生成可执行内存
        c->generated_code_data.algo    = Algorithm::INVALID; // 设置算法为无效
        c->generated_code_data.height  = std::numeric_limits<uint64_t>::max(); // 设置高度为最大值

        ctx[i] = c; // 将c赋值给ctx[i]
    }
}


void xmrig::CnCtx::release(cryptonight_ctx **ctx, size_t count)
{
    if (ctx[0] == nullptr) { // 如果ctx[0]为空指针，则返回
        return;
    }

    for (size_t i = 0; i < count; ++i) {
        _mm_free(ctx[i]); // 释放内存空间
    }
}
```