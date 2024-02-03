# `xmrig\src\crypto\rx\RxVm.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   该许可证由自由软件基金会发布，无论是许可证的第3版还是
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。
 *   有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。
 *   如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RX_VM_H
#define XMRIG_RX_VM_H


#include <cstdint>


class randomx_vm;


namespace xmrig
{


class Assembly;
class RxDataset;


class RxVm
{
public:
    static randomx_vm *create(RxDataset *dataset, uint8_t *scratchpad, bool softAes, const Assembly &assembly, uint32_t node);
    static void destroy(randomx_vm *vm);
};


} /* namespace xmrig */


#endif /* XMRIG_RX_VM_H */
```