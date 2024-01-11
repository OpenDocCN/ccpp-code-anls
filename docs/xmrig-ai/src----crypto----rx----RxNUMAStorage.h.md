# `xmrig\src\crypto\rx\RxNUMAStorage.h`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布本程序的副本
 *   或者（根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RX_NUMASTORAGE_H
#define XMRIG_RX_NUMASTORAGE_H


#include "backend/common/interfaces/IRxStorage.h"


#include <vector>


namespace xmrig
{


class RxNUMAStoragePrivate;


class RxNUMAStorage : public IRxStorage
{
public:
    XMRIG_DISABLE_COPY_MOVE(RxNUMAStorage);

    RxNUMAStorage(const std::vector<uint32_t> &nodeset);
    ~RxNUMAStorage() override;

protected:
    bool isAllocated() const override;
    HugePagesInfo hugePages() const override;
    RxDataset *dataset(const Job &job, uint32_t nodeId) const override;
    void init(const RxSeed &seed, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority) override;

private:
    RxNUMAStoragePrivate *d_ptr;
};


} /* namespace xmrig */


#endif /* XMRIG_RX_NUMASTORAGE_H */


注释：这是一个C++头文件，定义了`RxNUMAStorage`类，该类是`IRxStorage`接口的一个实现。它包含了一些成员函数和私有变量。
```