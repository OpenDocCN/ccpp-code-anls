# `xmrig\src\base\io\Signals.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SIGNALS_H
#define XMRIG_SIGNALS_H


#include "base/tools/Object.h"


#include <csignal>
#include <cstddef>


using uv_signal_t = struct uv_signal_s;


namespace xmrig {


class ISignalListener;


class Signals
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Signals)

#   ifdef SIGUSR1
    constexpr static const size_t kSignalsCount = 4;
#   else
    constexpr static const size_t kSignalsCount = 3;
#   endif

    // 构造函数，接受一个 ISignalListener 指针
    Signals(ISignalListener *listener);
    // 析构函数
    ~Signals();

private:
    // 关闭信号处理
    void close(int signum);

    // 处理信号的静态方法
    static void onSignal(uv_signal_t *handle, int signum);

    ISignalListener *m_listener; // 信号监听器指针
    uv_signal_t *m_signals[kSignalsCount]{}; // 信号数组
};


} /* namespace xmrig */


#endif /* XMRIG_SIGNALS_H */
```