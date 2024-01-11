# `xmrig\src\crypto\ghostrider\ghostrider.h`

```
/* XMRig
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_GR_HASH_H
#define XMRIG_GR_HASH_H

#include <cstddef>
#include <cstdint>
#include <vector>

// 定义 cryptonight_ctx 结构体
struct cryptonight_ctx;

// 命名空间 xmrig
namespace xmrig
{
    // 命名空间 ghostrider
    namespace ghostrider
    {
        // 定义 HelperThread 结构体
        struct HelperThread;

        // 声明 benchmark 函数
        void benchmark();
        // 声明 create_helper_thread 函数
        HelperThread* create_helper_thread(int64_t cpu_index, int priority, const std::vector<int64_t>& affinities);
        // 声明 destroy_helper_thread 函数
        void destroy_helper_thread(HelperThread* t);
        // 声明 hash_octa 函数
        void hash_octa(const uint8_t* data, size_t size, uint8_t* output, cryptonight_ctx** ctx, HelperThread* helper, bool verbose = true);
    } // namespace ghostrider
} // namespace xmrig

#endif // XMRIG_GR_HASH_H
```