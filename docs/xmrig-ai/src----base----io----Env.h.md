# `xmrig\src\base\io\Env.h`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 要么是许可证的第3版，要么是（在您的选择下）任何更高版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ENV_H
#define XMRIG_ENV_H

#include "base/tools/String.h"

#include <map>

namespace xmrig {

class Env
{
public:
    // 根据给定的环境变量和额外的映射，扩展输入字符串
    static String expand(const char *in, const std::map<String, String> &extra = {});
    // 获取指定环境变量的值，可提供额外的映射
    static String get(const String &name, const std::map<String, String> &extra = {});
    // 获取主机名
    static String hostname();
};

} /* namespace xmrig */

#endif /* XMRIG_ENV_H */
```