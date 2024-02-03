# `xmrig\src\App_win.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布
 * 由自由软件基金会发布的许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <winsock2.h>
#include <windows.h>


#include "App.h"
#include "core/Controller.h"


// 后台运行函数
bool xmrig::App::background(int &)
{
    // 如果控制器不在后台运行，则返回false
    if (!m_controller->isBackground()) {
        return false;
    }

    // 获取控制台窗口句柄
    HWND hcon = GetConsoleWindow();
    // 如果存在控制台窗口，则隐藏窗口
    if (hcon) {
        ShowWindow(hcon, SW_HIDE);
    } else {
        // 否则，关闭标准输出句柄并释放控制台
        HANDLE h = GetStdHandle(STD_OUTPUT_HANDLE);
        CloseHandle(h);
        FreeConsole();
    }

    return false;
}
```