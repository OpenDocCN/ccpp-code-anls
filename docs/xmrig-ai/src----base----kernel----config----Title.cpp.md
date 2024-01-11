# `xmrig\src\base\kernel\config\Title.cpp`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/kernel/config/Title.h"  // 引入Title类的头文件
#include "3rdparty/rapidjson/document.h"  // 引入rapidjson库的document头文件
#include "base/io/Env.h"  // 引入Env类的头文件
#include "version.h"  // 引入版本信息的头文件


xmrig::Title::Title(const rapidjson::Value &value)  // Title类的构造函数，参数为rapidjson::Value类型的引用
{
    if (value.IsBool()) {  // 如果value是布尔类型
        m_enabled = value.GetBool();  // 将value的布尔值赋给m_enabled
    }
    else if (value.IsString()) {  // 如果value是字符串类型
        m_value = value.GetString();  // 将value的字符串值赋给m_value
    }
}


rapidjson::Value xmrig::Title::toJSON() const  // 返回类型为rapidjson::Value的toJSON方法
{
    if (isEnabled() && !m_value.isNull()) {  // 如果启用并且m_value不为空
        return m_value.toJSON();  // 返回m_value的JSON表示
    }

    return rapidjson::Value(m_enabled);  // 否则返回m_enabled的JSON表示
}


xmrig::String xmrig::Title::value() const  // 返回类型为String的value方法
{
    if (!isEnabled()) {  // 如果未启用
        return {};  // 返回空字符串
    }

    if (m_value.isNull()) {  // 如果m_value为空
        return APP_NAME " " APP_VERSION;  // 返回应用程序名称和版本号
    }

    return Env::expand(m_value);  // 否则返回扩展后的m_value
}
```