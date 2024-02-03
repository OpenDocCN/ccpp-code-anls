# `xmrig\src\App_unix.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cstdlib>
#include <csignal>
#include <cerrno>
#include <unistd.h>

#include "App.h"
#include "base/io/log/Log.h"
#include "core/Controller.h"

// 后台化程序
bool xmrig::App::background(int &rc)
{
    // 如果控制器不在后台运行，则返回false
    if (!m_controller->isBackground()) {
        return false;
    }

    // 创建子进程
    int i = fork();
    if (i < 0) {
        rc = 1;
        return true;
    }

    // 如果是父进程，则返回0
    if (i > 0) {
        rc = 0;
        return true;
    }

    // 创建新的会话
    i = setsid();

    // 如果创建会话失败，则记录错误信息
    if (i < 0) {
        LOG_ERR("setsid() failed (errno = %d)", errno);
    }

    // 改变当前工作目录为根目录
    i = chdir("/");
    if (i < 0) {
        LOG_ERR("chdir() failed (errno = %d)", errno);
    }

    return false;
}
```