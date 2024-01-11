# `xmrig\src\xmrig.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发和/或修改
 *   它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "App.h"  // 包含自定义的App头文件
#include "base/kernel/Entry.h"  // 包含自定义的Entry头文件
#include "base/kernel/Process.h"  // 包含自定义的Process头文件


int main(int argc, char **argv)  // 主函数，接受命令行参数
{
    using namespace xmrig;  // 使用xmrig命名空间

    Process process(argc, argv);  // 创建Process对象，传入命令行参数
    const Entry::Id entry = Entry::get(process);  // 获取Entry的ID
    if (entry) {  // 如果存在Entry
        return Entry::exec(process, entry);  // 执行Entry
    }

    App app(&process);  // 创建App对象，传入Process对象

    return app.exec();  // 执行App对象的exec方法
}
```