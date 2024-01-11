# `xmrig\src\base\net\http\HttpListener.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/net/http/HttpListener.h"
#include "3rdparty/llhttp/llhttp.h"
#include "base/io/log/Log.h"
#include "base/net/http/HttpData.h"


void xmrig::HttpListener::onHttpData(const HttpData &data)
{
#   ifdef APP_DEBUG
    // 如果数据不是请求，则记录调试信息
    if (!data.isRequest()) {
        LOG_DEBUG("%s " CYAN_BOLD("http%s://%s:%u ") MAGENTA_BOLD("\"%s %s\" ") CSI "1;%dm%d" CLEAR BLACK_BOLD(" received: ") CYAN_BOLD("%zu") BLACK_BOLD(" bytes"),
                  m_tag, data.tlsVersion() ? "s" : "", data.host(), data.port(), llhttp_method_name(static_cast<llhttp_method>(data.method)), data.url.data(),
                  (data.status >= 400 || data.status < 0) ? 31 : 32, data.status, data.body.size());

        // 如果数据体大小小于最大缓冲区大小减去1024，并且是JSON格式，则打印数据体内容
        if (data.body.size() < (Log::kMaxBufferSize - 1024) && data.isJSON()) {
            Log::print(BLUE_BG_BOLD("%s:") BLACK_BOLD_S " %.*s", data.headers.at(HttpData::kContentTypeL).c_str(), static_cast<int>(data.body.size()), data.body.c_str());
        }
    }
#   endif

    // 调用监听器的onHttpData方法
    m_listener->onHttpData(data);
}
```