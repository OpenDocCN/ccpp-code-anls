# `xmrig\src\base\kernel\constants.h`

```
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONSTANTS_H
#define XMRIG_CONSTANTS_H


#include <cstddef>
#include <cstdint>


// 定义网络缓冲区块大小为 64KB
constexpr size_t      XMRIG_NET_BUFFER_CHUNK_SIZE           = 64 * 1024;
// 定义网络缓冲区初始块数为 4
constexpr size_t      XMRIG_NET_BUFFER_INIT_CHUNKS          = 4;


#endif /* XMRIG_CONSTANTS_H */
```