# `xmrig\src\base\kernel\interfaces\ILineListener.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ILINELISTENER_H
#define XMRIG_ILINELISTENER_H


#include "base/tools/Object.h"


#include <cstdint>


namespace xmrig {


class ILineListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(ILineListener)

    ILineListener()             = default;
    virtual ~ILineListener()    = default;

    virtual void onLine(char *line, size_t size) = 0;
};


} /* namespace xmrig */


#endif // XMRIG_ILINELISTENER_H
```