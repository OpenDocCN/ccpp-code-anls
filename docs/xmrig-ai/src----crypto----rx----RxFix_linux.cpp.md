# `xmrig\src\crypto\rx\RxFix_linux.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，版本 3 或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。详见
 *   GNU 通用公共许可证获取更多详细信息。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "crypto/rx/RxFix.h"
#include "base/io/log/Log.h"


#include <csignal>
#include <cstdlib>
#include <ucontext.h>


namespace xmrig {


static thread_local std::pair<const void*, const void*> mainLoopBounds = { nullptr, nullptr };


static void MainLoopHandler(int sig, siginfo_t *info, void *ucontext)
{
    // 将 void* 类型的 ucontext 转换为 ucontext_t* 类型
    ucontext_t *ucp = (ucontext_t*) ucontext;

    // 打印日志，输出信号名称和触发地址
    LOG_VERBOSE(YELLOW_BOLD("%s at %p"), (sig == SIGSEGV) ? "SIGSEGV" : "SIGILL", ucp->uc_mcontext.gregs[REG_RIP]);

    // 将 void* 类型的指针转换为 void* 类型
    void* p = reinterpret_cast<void*>(ucp->uc_mcontext.gregs[REG_RIP]);
    // 获取 mainLoopBounds 的引用
    const std::pair<const void*, const void*>& loopBounds = mainLoopBounds;

    // 如果触发地址在 mainLoopBounds 范围内，则将 RIP 寄存器设置为 mainLoopBounds 的结束地址
    if ((loopBounds.first <= p) && (p < loopBounds.second)) {
        ucp->uc_mcontext.gregs[REG_RIP] = reinterpret_cast<size_t>(loopBounds.second);
    }
    // 否则，终止程序
    else {
        abort();
    }
}


} // namespace xmrig



void xmrig::RxFix::setMainLoopBounds(const std::pair<const void *, const void *> &bounds)
{
    // 设置 mainLoopBounds 的值为给定的范围
    mainLoopBounds = bounds;
}


void xmrig::RxFix::setupMainLoopExceptionFrame()
{
    // 创建一个 sigaction 结构体对象 act
    struct sigaction act = {};
    // 将 MainLoopHandler 函数赋值给 sa_sigaction 字段
    act.sa_sigaction = MainLoopHandler;
    # 设置信号处理器的标志，指定当信号处理器执行时是否自动重启系统调用，并且提供有关信号的附加信息
    act.sa_flags = SA_RESTART | SA_SIGINFO;
    # 设置对 SIGSEGV 信号的处理方式为 act 指定的处理方式
    sigaction(SIGSEGV, &act, nullptr);
    # 设置对 SIGILL 信号的处理方式为 act 指定的处理方式
    sigaction(SIGILL, &act, nullptr);
这是一个闭合的大括号，可能是代码块的结束。
```