# `xmrig\src\crypto\cn\CryptoNight_x86_vaes.h`

```cpp
// 包含版权声明和许可信息
/* XMRig
 * 版权声明
 */

// 防止头文件重复包含
#ifndef XMRIG_CRYPTONIGHT_X86_VAES_H
#define XMRIG_CRYPTONIGHT_X86_VAES_H

// 包含CnAlgo.h头文件
#include "crypto/cn/CnAlgo.h"

// 声明cryptonight_ctx结构体
struct cryptonight_ctx;

// 声明xmrig命名空间
namespace xmrig {

// 声明使用VAES指令集的内存爆炸和内存压缩函数
void cn_explode_scratchpad_vaes(cryptonight_ctx* ctx, size_t memory, bool half_mem);
void cn_explode_scratchpad_vaes_double(cryptonight_ctx* ctx1, cryptonight_ctx* ctx2, size_t memory, bool half_mem);
void cn_implode_scratchpad_vaes(cryptonight_ctx* ctx, size_t memory, bool half_mem);
void cn_implode_scratchpad_vaes_double(cryptonight_ctx* ctx1, cryptonight_ctx* ctx2, size_t memory, bool half_mem);

} // xmrig

// 结束头文件防护
#endif /* XMRIG_CRYPTONIGHT_X86_VAES_H */
```