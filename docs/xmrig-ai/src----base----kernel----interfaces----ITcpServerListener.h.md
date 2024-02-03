# `xmrig\src\base\kernel\interfaces\ITcpServerListener.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   该许可证由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何担保；甚至没有暗示的担保适用于特定目的。请参阅 GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ITCPSERVERLISTENER_H
#define XMRIG_ITCPSERVERLISTENER_H


#include <stdint.h>


typedef struct uv_stream_s uv_stream_t;


namespace xmrig {


class String;


class ITcpServerListener
{
public:
    virtual ~ITcpServerListener() = default;

    virtual void onConnection(uv_stream_t *stream, uint16_t port) = 0;
};


} /* namespace xmrig */


#endif // XMRIG_ITCPSERVERLISTENER_H
```