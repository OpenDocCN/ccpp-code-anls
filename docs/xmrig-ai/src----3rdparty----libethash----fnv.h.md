# `xmrig\src\3rdparty\libethash\fnv.h`

```cpp
/*
  这个文件是 cpp-ethereum 的一部分。

  cpp-ethereum 是自由软件：您可以根据自由软件基金会发布的 GNU 通用公共许可证的第3版或（根据您的选择）更高版本的条款重新分发和/或修改它。

  cpp-ethereum 希望能够提供帮助，但没有任何担保；甚至没有对特定目的的适销性或适用性的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。

  您应该已经收到了 GNU 通用公共许可证的副本，连同 cpp-ethereum 一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
*/
/** @file fnv.h
* @author Matthew Wampler-Doty <negacthulhu@gmail.com>
* @date 2015
*/

#pragma once
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

#define FNV_PRIME 0x01000193

/* FNV-1规范将素数与输入的每个字节（八位）依次相乘。
   我们改为将其与完整的32位输入相乘。
   这与规范的 FNV-1 实现相比会得到不同的结果。
*/
static inline uint32_t fnv_hash(uint32_t const x, uint32_t const y)
{
    return x * FNV_PRIME ^ y;
}

#ifdef __cplusplus
}
#endif
```