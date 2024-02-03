# `xmrig\src\base\io\Signals.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的适用性。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/kernel/interfaces/ISignalListener.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/io/Signals.h"
#include "base/tools/Handle.h"

#ifdef SIGUSR1
static const int signums[xmrig::Signals::kSignalsCount] = { SIGHUP, SIGINT, SIGTERM, SIGUSR1 };
#else
static const int signums[xmrig::Signals::kSignalsCount] = { SIGHUP, SIGINT, SIGTERM };
#endif

// 信号处理类的构造函数，初始化信号监听器
xmrig::Signals::Signals(ISignalListener *listener)
    : m_listener(listener)
{
#   ifndef XMRIG_OS_WIN
    // 忽略SIGPIPE信号
    signal(SIGPIPE, SIG_IGN);
#   endif

    // 遍历信号数组，为每个信号初始化信号处理器
    for (size_t i = 0; i < kSignalsCount; ++i) {
        auto signal  = new uv_signal_t;
        signal->data = this;

        m_signals[i] = signal;

        uv_signal_init(uv_default_loop(), signal);
        uv_signal_start(signal, Signals::onSignal, signums[i]);
    }
}

// 信号处理类的析构函数，关闭所有信号处理器
xmrig::Signals::~Signals()
{
    for (auto signal : m_signals) {
        Handle::close(signal);
    }
}

// 信号处理函数，根据接收到的信号进行相应的处理
void xmrig::Signals::onSignal(uv_signal_t *handle, int signum)
{
    switch (signum)
    {
    case SIGHUP:
        LOG_WARN("%s " YELLOW("SIGHUP received, exiting"), Tags::signal());
        break;
    # 如果接收到 SIGTERM 信号，则记录警告日志并退出程序
    case SIGTERM:
        LOG_WARN("%s " YELLOW("SIGTERM received, exiting"), Tags::signal());
        break;

    # 如果接收到 SIGINT 信号，则记录警告日志并退出程序
    case SIGINT:
        LOG_WARN("%s " YELLOW("SIGINT received, exiting"), Tags::signal());
        break;
# 如果定义了 SIGUSR1 信号
    case SIGUSR1:
        # 记录日志，表示接收到 SIGUSR1 信号
        LOG_V5("%s " WHITE_BOLD("SIGUSR1 received"), Tags::signal());
        # 跳出 switch 语句
        break;
# 结束 ifdef 块
# 默认情况
    default:
        # 跳出 switch 语句
        break;
    }
    # 转换指针类型，调用 onSignal 方法处理信号
    static_cast<Signals *>(handle->data)->m_listener->onSignal(signum);
}
```