# `xmrig\src\base\kernel\Process_win.cpp`

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
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <uv.h>

#include "base/kernel/Process.h"

// 返回当前进程的进程ID
int xmrig::Process::pid()
{
#   if UV_VERSION_HEX >= 0x011200
    // 如果 libuv 版本大于等于 1.18.0，则使用 libuv 提供的函数获取进程ID
    return uv_os_getpid();
#   else
    // 否则使用操作系统提供的函数获取进程ID
    return GetCurrentProcessId();
#   endif
}
```