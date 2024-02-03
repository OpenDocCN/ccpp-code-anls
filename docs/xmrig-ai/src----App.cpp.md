# `xmrig\src\App.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <cstdlib>
#include <uv.h>


#include "App.h"
#include "backend/cpu/Cpu.h"
#include "base/io/Console.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/io/Signals.h"
#include "base/kernel/Platform.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "Summary.h"
#include "version.h"


xmrig::App::App(Process *process)
{
    // 使用进程创建一个控制器对象
    m_controller = std::make_shared<Controller>(process);
}


xmrig::App::~App()
{
    // 释放 CPU 资源
    Cpu::release();
}


int xmrig::App::exec()
{
    // 如果控制器不准备好，则记录错误信息并返回 2
    if (!m_controller->isReady()) {
        LOG_EMERG("no valid configuration found, try https://xmrig.com/wizard");

        return 2;
    }
    # 创建一个指向 Signals 对象的 shared_ptr，该对象是当前对象的成员
    m_signals = std::make_shared<Signals>(this);

    # 定义一个整型变量 rc，并初始化为 0
    int rc = 0;
    # 如果在后台运行，则直接返回 rc
    if (background(rc)) {
        return rc;
    }

    # 初始化控制器，将返回值赋给 rc
    rc = m_controller->init();
    # 如果初始化失败，则直接返回 rc
    if (rc != 0) {
        return rc;
    }

    # 如果控制器不在后台运行，则创建一个指向 Console 对象的 shared_ptr，该对象是当前对象的成员
    if (!m_controller->isBackground()) {
        m_console = std::make_shared<Console>(this);
    }

    # 调用 Summary 类的 print 方法，传入控制器对象的指针作为参数
    Summary::print(m_controller.get());

    # 如果配置为模拟运行，则记录日志并返回 0
    if (m_controller->config()->isDryRun()) {
        LOG_NOTICE("%s " WHITE_BOLD("OK"), Tags::config());
        return 0;
    }

    # 启动控制器
    m_controller->start();

    # 运行 libuv 默认事件循环，将返回值赋给 rc
    rc = uv_run(uv_default_loop(), UV_RUN_DEFAULT);
    # 关闭 libuv 默认事件循环
    uv_loop_close(uv_default_loop());

    # 返回 rc
    return rc;
// 定义 xmrig::App 类的 onConsoleCommand 方法，处理控制台命令
void xmrig::App::onConsoleCommand(char command)
{
    // 如果接收到 Ctrl+C 命令
    if (command == 3) {
        // 输出警告日志，表示接收到 Ctrl+C 命令，然后退出程序
        LOG_WARN("%s " YELLOW("Ctrl+C received, exiting"), Tags::signal());
        close();
    }
    // 如果接收到其他命令
    else {
        // 调用 m_controller 对象的 execCommand 方法，执行命令
        m_controller->execCommand(command);
    }
}

// 定义 xmrig::App 类的 onSignal 方法，处理信号
void xmrig::App::onSignal(int signum)
{
    // 根据不同的信号类型进行处理
    switch (signum)
    {
    // 如果是 SIGHUP、SIGTERM 或 SIGINT 信号
    case SIGHUP:
    case SIGTERM:
    case SIGINT:
        // 调用 close 方法，关闭程序
        return close();

    // 如果是其他信号
    default:
        // 什么也不做
        break;
    }
}

// 定义 xmrig::App 类的 close 方法，关闭程序
void xmrig::App::close()
{
    // 重置 m_signals 对象
    m_signals.reset();
    // 重置 m_console 对象
    m_console.reset();

    // 调用 m_controller 对象的 stop 方法，停止控制器
    m_controller->stop();

    // 销毁日志对象
    Log::destroy();
}
```