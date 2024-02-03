# `xmrig\src\backend\common\interfaces\IRxStorage.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发和/或修改它，
 *   其版本由自由软件基金会发布，可以选择使用版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有适用于特定目的的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IRXSTORAGE_H
#define XMRIG_IRXSTORAGE_H


#include "base/tools/Object.h"
#include "crypto/common/HugePagesInfo.h"
#include "crypto/rx/RxConfig.h"


#include <cstdint>
#include <utility>


namespace xmrig {


class Job;
class RxDataset;
class RxSeed;


class IRxStorage
{
public:
    XMRIG_DISABLE_COPY_MOVE(IRxStorage)

    IRxStorage()            = default;
    virtual ~IRxStorage()   = default;

    virtual bool isAllocated() const                                                                                            = 0;  // 检查存储是否已分配
    virtual HugePagesInfo hugePages() const                                                                                     = 0;  // 获取大页信息
    virtual RxDataset *dataset(const Job &job, uint32_t nodeId) const                                                           = 0;  // 获取数据集
    virtual void init(const RxSeed &seed, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority) = 0;  // 初始化存储
};


} /* namespace xmrig */


#endif // XMRIG_IRXSTORAGE_H
```