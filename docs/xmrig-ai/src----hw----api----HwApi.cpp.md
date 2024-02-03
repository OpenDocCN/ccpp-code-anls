# `xmrig\src\hw\api\HwApi.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布
 *   版本 3 或 (根据您的选择) 任何更高版本。
 *
 *   本程序以希望它有用的方式分发，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "hw/api/HwApi.h"
#include "base/api/interfaces/IApiRequest.h"
#include "base/tools/String.h"


#ifdef XMRIG_FEATURE_DMI
#   include "hw/dmi/DmiReader.h"
#endif


// 处理 API 请求的回调函数
void xmrig::HwApi::onRequest(IApiRequest &request)
{
    // 如果请求方法为 GET
    if (request.method() == IApiRequest::METHOD_GET) {
        // 如果请求的 URL 为 "/2/dmi"
#       ifdef XMRIG_FEATURE_DMI
        if (request.url() == "/2/dmi") {
            // 如果 DMI 读取器为空，则创建并读取 DMI 信息
            if (!m_dmi) {
                m_dmi = std::make_shared<DmiReader>();
                m_dmi->read();
            }

            // 接受请求
            request.accept();
            // 将 DMI 信息转换为 JSON 格式，并添加到请求的回复中
            m_dmi->toJSON(request.reply(), request.doc());
        }
#       endif
    }
}
```