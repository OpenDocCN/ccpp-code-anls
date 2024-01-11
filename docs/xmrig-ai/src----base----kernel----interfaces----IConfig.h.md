# `xmrig\src\base\kernel\interfaces\IConfig.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ICONFIG_H
#define XMRIG_ICONFIG_H


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class IJsonReader;
class String;


class IConfig
{
public:
    // 默认析构函数
    virtual ~IConfig() = default;

    // 返回是否处于监视模式
    virtual bool isWatch() const                                       = 0;
    // 从JSON读取配置
    virtual bool read(const IJsonReader &reader, const char *fileName) = 0;
    // 保存配置
    virtual bool save()                                                = 0;
    // 返回配置文件名
    virtual const String &fileName() const                             = 0;
    // 获取配置的JSON表示
    virtual void getJSON(rapidjson::Document &doc) const               = 0;
    // 设置配置文件名
    virtual void setFileName(const char *fileName)                     = 0;
};


} /* namespace xmrig */


#endif // XMRIG_ICONFIG_H
```