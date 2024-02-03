# `xmrig\src\core\Taskbar.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的适用性。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TASKBAR_H
#define XMRIG_TASKBAR_H

namespace xmrig {

struct TaskbarPrivate;

class Taskbar
{
public:
    Taskbar();  // 构造函数
    ~Taskbar();  // 析构函数

    void setActive(bool active);  // 设置任务栏是否激活
    void setEnabled(bool enabled);  // 设置任务栏是否启用

private:
    bool m_active = false;  // 任务栏是否激活，默认为false
    bool m_enabled = true;  // 任务栏是否启用，默认为true

    TaskbarPrivate* d_ptr = nullptr;  // 任务栏私有指针

    void updateTaskbarColor();  // 更新任务栏颜色
};

} // namespace xmrig

#endif /* XMRIG_TASKBAR_H */
```