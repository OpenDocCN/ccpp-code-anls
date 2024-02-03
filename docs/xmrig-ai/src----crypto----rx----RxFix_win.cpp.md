# `xmrig\src\crypto\rx\RxFix_win.cpp`

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
 *   本程序以希望它有用的方式分发，
 *   但没有任何担保；甚至没有隐含的适销性或特定用途的保证。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/RxFix.h"
#include "base/io/log/Log.h"


#include <windows.h>


namespace xmrig {


// 定义一个线程局部变量，用于存储主循环的边界
static thread_local std::pair<const void*, const void*> mainLoopBounds = { nullptr, nullptr };


// 定义一个异常处理函数，用于处理主循环的异常
static LONG WINAPI MainLoopHandler(_EXCEPTION_POINTERS *ExceptionInfo)
{
    // 如果异常代码为0xC0000005（访问冲突）
    if (ExceptionInfo->ExceptionRecord->ExceptionCode == 0xC0000005) {
        const char* accessType = nullptr;
        // 根据异常信息中的第一个参数确定访问类型
        switch (ExceptionInfo->ExceptionRecord->ExceptionInformation[0]) {
        case 0: accessType = "read"; break;
        case 1: accessType = "write"; break;
        case 8: accessType = "DEP violation"; break;
        default: accessType = "unknown"; break;
        }
        // 输出线程ID、异常地址、访问类型和异常信息中的第二个参数
        LOG_VERBOSE(YELLOW_BOLD("[THREAD %u] Access violation at 0x%p: %s at address 0x%p"), GetCurrentThreadId(), ExceptionInfo->ExceptionRecord->ExceptionAddress, accessType, ExceptionInfo->ExceptionRecord->ExceptionInformation[1]);
    }
    else {
        // 如果发生异常，打印异常信息，包括线程ID、异常代码和异常地址
        LOG_VERBOSE(YELLOW_BOLD("[THREAD %u] Exception 0x%08X at 0x%p"), GetCurrentThreadId(), ExceptionInfo->ExceptionRecord->ExceptionCode, ExceptionInfo->ExceptionRecord->ExceptionAddress);
    }

    // 将异常地址转换为指针类型
    void* p = reinterpret_cast<void*>(ExceptionInfo->ContextRecord->Rip); // NOLINT(performance-no-int-to-ptr)
    // 获取主循环的边界
    const std::pair<const void*, const void*>& loopBounds = mainLoopBounds;

    // 如果异常地址在主循环边界内，则将异常地址设置为主循环的结束地址，并继续执行
    if ((loopBounds.first <= p) && (p < loopBounds.second)) {
        ExceptionInfo->ContextRecord->Rip = reinterpret_cast<DWORD64>(loopBounds.second);
        return EXCEPTION_CONTINUE_EXECUTION;
    }

    // 如果异常地址不在主循环边界内，则继续搜索异常处理程序
    return EXCEPTION_CONTINUE_SEARCH;
} // namespace xmrig

这是一个命名空间的结束标记，表示代码块的结束。


void xmrig::RxFix::setMainLoopBounds(const std::pair<const void *, const void *> &bounds)
{
    mainLoopBounds = bounds;
}

这是一个名为`setMainLoopBounds`的函数，它接受一个类型为`std::pair<const void *, const void *>`的参数`bounds`。函数的作用是将`bounds`赋值给`mainLoopBounds`变量。


void xmrig::RxFix::setupMainLoopExceptionFrame()
{
    AddVectoredExceptionHandler(1, MainLoopHandler);
}

这是一个名为`setupMainLoopExceptionFrame`的函数，它没有参数。函数的作用是调用`AddVectoredExceptionHandler`函数，将参数`1`和`MainLoopHandler`传递给它。
```