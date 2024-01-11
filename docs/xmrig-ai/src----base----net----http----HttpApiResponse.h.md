# `xmrig\src\base\net\http\HttpApiResponse.h`

```
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用而分发的，但没有任何担保；甚至没有适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPAPIRESPONSE_H
#define XMRIG_HTTPAPIRESPONSE_H


#include "3rdparty/rapidjson/document.h"
#include "base/net/http/HttpResponse.h"


namespace xmrig {


class HttpApiResponse : public HttpResponse
{
public:
    // 构造函数，传入 id
    HttpApiResponse(uint64_t id);
    // 构造函数，传入 id 和状态
    HttpApiResponse(uint64_t id, int status);

    // 返回 rapidjson 文档对象的引用
    inline rapidjson::Document &doc() { return m_doc; }

    // 结束响应
    void end();

private:
    // rapidjson 文档对象
    rapidjson::Document m_doc;
};


} // namespace xmrig


#endif // XMRIG_HTTPAPIRESPONSE_H
```