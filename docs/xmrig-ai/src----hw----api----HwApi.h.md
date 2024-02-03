# `xmrig\src\hw\api\HwApi.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据GNU通用公共许可证的条款，它是免费软件基金会发布的版本3或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序以希望它有用的方式分发，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细信息请参见
 *   GNU通用公共许可证。
 *
 *   如果没有收到GNU通用公共许可证的副本
 *   请参阅<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HWAPI_H
#define XMRIG_HWAPI_H


#include "base/api/interfaces/IApiListener.h"


#include <memory>


namespace xmrig {


class DmiReader;


class HwApi : public IApiListener
{
public:
    HwApi() = default;

protected:
    void onRequest(IApiRequest &request) override;

private:
#   ifdef XMRIG_FEATURE_DMI
    std::shared_ptr<DmiReader> m_dmi;
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_HWAPI_H */


注释：这段代码是一个头文件，定义了一个名为HwApi的类，该类继承自IApiListener接口。它还包含了一个名为DmiReader的类的共享指针。
```