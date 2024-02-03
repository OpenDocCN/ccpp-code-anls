# `xmrig\src\crypto\argon2\Hash.h`

```cpp
/* XMRig
 * 版权声明
 * 该程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，可以选择遵循许可证的第三版或者之后的版本。
 * 该程序是基于希望它能有用而发布的，但没有任何担保；甚至没有暗示的担保适用于特定目的的保证。更多详情请参阅 GNU 通用公共许可证。
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请访问 http://www.gnu.org/licenses/。
 */

#ifndef XMRIG_ARGON2_HASH_H
#define XMRIG_ARGON2_HASH_H

#include "3rdparty/argon2.h"  // 引入第三方库 argon2
#include "base/crypto/Algorithm.h"  // 引入加密算法
#include "crypto/cn/CryptoNight.h"  // 引入 CryptoNight 加密算法

namespace xmrig { namespace argon2 {

// 定义单次哈希函数，根据不同的算法进行不同的处理
template<Algorithm::Id ALGO>
inline void single_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t)
{
    // 如果算法为 AR2_CHUKWA，则使用 argon2id_hash_raw_ex 函数进行哈希计算
    if (ALGO == Algorithm::AR2_CHUKWA) {
        argon2id_hash_raw_ex(3, 512, 1, input, size, input, 16, output, 32, ctx[0]->memory);
    }
    // 如果算法为 AR2_CHUKWA_V2，则使用 argon2id_hash_raw_ex 函数进行哈希计算
    else if (ALGO == Algorithm::AR2_CHUKWA_V2) {
        argon2id_hash_raw_ex(4, 1024, 1, input, size, input, 16, output, 32, ctx[0]->memory);
    }
    # 如果算法选择为AR2_WRKZ，则执行以下操作
    else if (ALGO == Algorithm::AR2_WRKZ) {
        # 使用argon2id_hash_raw_ex函数进行哈希计算，参数依次为：内存迭代次数、内存大小、线程数、输入数据、输入数据大小、输入数据、盐值大小、输出数据、输出数据大小、上下文内存
        argon2id_hash_raw_ex(4, 256, 1, input, size, input, 16, output, 32, ctx[0]->memory);
    }
# 结束 xmrig::argon2 命名空间
}

# 结束 xmrig 命名空间
}} // namespace xmrig::argon2

# 结束条件编译指令，关闭 XMRIG_ARGON2_HASH_H 头文件
#endif /* XMRIG_ARGON2_HASH_H */
```