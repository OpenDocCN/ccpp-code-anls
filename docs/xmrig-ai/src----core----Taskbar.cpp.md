# `xmrig\src\core\Taskbar.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "core/Taskbar.h"

#ifdef _WIN32


#include <Shobjidl.h>
#include <Objbase.h>


namespace xmrig {


struct TaskbarPrivate
{
    TaskbarPrivate()
    {
        // 初始化 COM 线程模型
        HRESULT hr = CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);
        if (hr < 0) {
            return;
        }

        // 创建任务栏列表实例
        hr = CoCreateInstance(CLSID_TaskbarList, NULL, CLSCTX_INPROC_SERVER, IID_PPV_ARGS(&m_taskbar));
        if (hr < 0) {
            return;
        }

        // 初始化任务栏列表
        hr = m_taskbar->HrInit();
        if (hr < 0) {
            m_taskbar->Release();
            m_taskbar = nullptr;
            return;
        }

        // 获取控制台窗口句柄
        m_consoleWnd = GetConsoleWindow();
    }

    ~TaskbarPrivate()
    {
        if (m_taskbar) {
            m_taskbar->Release();
        }
        // 反初始化 COM
        CoUninitialize();
    }

    ITaskbarList3* m_taskbar = nullptr;
    HWND m_consoleWnd = nullptr;
};


Taskbar::Taskbar() : d_ptr(new TaskbarPrivate())
{
}


Taskbar::~Taskbar()
{
    delete d_ptr;
}


void Taskbar::setActive(bool active)
{
    m_active = active;
    updateTaskbarColor();
}


void Taskbar::setEnabled(bool enabled)
{
    m_enabled = enabled;
    updateTaskbarColor();
}
// 更新任务栏颜色的函数
void Taskbar::updateTaskbarColor()
{
    // 如果任务栏存在
    if (d_ptr->m_taskbar) {
        // 如果程序处于活动状态
        if (m_active) {
            // 设置任务栏进度状态为无进度或暂停
            d_ptr->m_taskbar->SetProgressState(d_ptr->m_consoleWnd, m_enabled ? TBPF_NOPROGRESS : TBPF_PAUSED);
            // 设置任务栏进度值为0或1
            d_ptr->m_taskbar->SetProgressValue(d_ptr->m_consoleWnd, m_enabled ? 0 : 1, 1);
        }
        // 如果程序不处于活动状态
        else {
            // 设置任务栏进度状态为错误
            d_ptr->m_taskbar->SetProgressState(d_ptr->m_consoleWnd, TBPF_ERROR);
            // 设置任务栏进度值为1
            d_ptr->m_taskbar->SetProgressValue(d_ptr->m_consoleWnd, 1, 1);
        }
    }
}

// 结束命名空间 xmrig
} // namespace xmrig

// 如果不是 Windows 平台
#else // _WIN32

// 开始命名空间 xmrig
namespace xmrig {

// 空的构造函数
Taskbar::Taskbar() {}
// 空的析构函数
Taskbar::~Taskbar() {}
// 设置活动状态的函数
void Taskbar::setActive(bool) {}
// 设置启用状态的函数
void Taskbar::setEnabled(bool) {}

// 结束命名空间 xmrig
} // namespace xmrig

// 结束条件编译
#endif // _WIN32
```