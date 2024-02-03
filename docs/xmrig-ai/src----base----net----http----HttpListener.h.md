# `xmrig\src\base\net\http\HttpListener.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPLISTENER_H
#define XMRIG_HTTPLISTENER_H


#include "base/kernel/interfaces/IHttpListener.h"


namespace xmrig {


class HttpListener : public IHttpListener
{
public:
    // 构造函数，初始化HttpListener对象
    inline HttpListener(IHttpListener *listener, const char *tag = nullptr) :
#       ifdef APP_DEBUG
        m_tag(tag),
#       endif
        m_listener(listener)
    {}

protected:
    // 重写基类的onHttpData方法
    void onHttpData(const HttpData &data) override;

private:
#   ifdef APP_DEBUG
    const char *m_tag;
#   endif
    IHttpListener *m_listener;
};


} /* namespace xmrig */


#endif // XMRIG_HTTPLISTENER_H
```