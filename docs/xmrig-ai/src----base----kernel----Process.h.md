# `xmrig\src\base\kernel\Process.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_PROCESS_H
#define XMRIG_PROCESS_H


#include "base/tools/Arguments.h"


#ifdef WIN32
#   define XMRIG_DIR_SEPARATOR "\\"
#else
#   define XMRIG_DIR_SEPARATOR "/"
#endif


namespace xmrig {


class Process
{
public:
    enum Location {
        ExeLocation,  // 可执行文件位置
        CwdLocation,  // 当前工作目录位置
        DataLocation,  // 数据位置
        HomeLocation,  // 主目录位置
        TempLocation  // 临时目录位置
    };

    Process(int argc, char **argv);  // 构造函数，接受命令行参数的数量和数组

    static int pid();  // 返回当前进程的ID
    static int ppid();  // 返回父进程的ID
    static String exepath();  // 返回可执行文件的路径
    static String location(Location location, const char *fileName = nullptr);  // 返回指定位置的文件路径

    inline const Arguments &arguments() const { return m_arguments; }  // 返回参数对象的引用

private:
    Arguments m_arguments;  // 参数对象
};


} /* namespace xmrig */


#endif /* XMRIG_PROCESS_H */
```