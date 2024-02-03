# `xmrig\src\base\tools\Timer.h`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   它，无论是许可证的第 3 版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TIMER_H
#define XMRIG_TIMER_H


使用 uv_timer_t = struct uv_timer_s;


#include "base/tools/Object.h"


#include <cstdint>


namespace xmrig {


class ITimerListener;


class Timer
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Timer);

    Timer(ITimerListener *listener);
    Timer(ITimerListener *listener, uint64_t timeout, uint64_t repeat);
    ~Timer();

    inline int id() const { return m_id; }

    uint64_t repeat() const;
    void setRepeat(uint64_t repeat);
    void singleShot(uint64_t timeout, int id = 0);
    void start(uint64_t timeout, uint64_t repeat);
    void stop();

private:
    void init();

    static void onTimer(uv_timer_t *handle);

    int m_id                    = 0;
    ITimerListener *m_listener;
    uv_timer_t *m_timer         = nullptr;
};


} /* namespace xmrig */


#endif /* XMRIG_TIMER_H */
```