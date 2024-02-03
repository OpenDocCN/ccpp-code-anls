# `xmrig\src\base\tools\Arguments.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ARGUMENTS_H
#define XMRIG_ARGUMENTS_H


#include <vector>


#include "base/tools/String.h"


namespace xmrig {


class Arguments
{
public:
    Arguments(int argc, char **argv);

    // 检查参数中是否存在指定的参数名
    bool hasArg(const char *name) const;
    // 获取指定参数名的值
    const char *value(const char *key1, const char *key2 = nullptr) const;

    // 返回参数列表
    inline char **argv() const                     { return m_argv; }
    // 返回参数数据
    inline const std::vector<String> &data() const { return m_data; }
    // 返回参数个数
    inline int argc() const                        { return m_argc; }

private:
    // 添加参数
    void add(const char *arg);

    char **m_argv;
    int m_argc;
    std::vector<String> m_data;
};


} /* namespace xmrig */


#endif /* XMRIG_ARGUMENTS_H */
```