# `xmrig\src\base\tools\Timer.cpp`

```
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   其中包括许可证的第3版或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/tools/Timer.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/tools/Handle.h"


// 定义 Timer 类的构造函数，传入 ITimerListener 指针
xmrig::Timer::Timer(ITimerListener *listener) :
    m_listener(listener)
{
    // 初始化 Timer 对象
    init();
}


// 定义 Timer 类的构造函数，传入 ITimerListener 指针、超时时间和重复时间
xmrig::Timer::Timer(ITimerListener *listener, uint64_t timeout, uint64_t repeat) :
    m_listener(listener)
{
    // 初始化 Timer 对象
    init();
    // 启动定时器
    start(timeout, repeat);
}


// 定义 Timer 类的析构函数
xmrig::Timer::~Timer()
{
    // 关闭定时器
    Handle::close(m_timer);
}


// 返回定时器的重复时间
uint64_t xmrig::Timer::repeat() const
{
    return uv_timer_get_repeat(m_timer);
}


// 设置定时器的重复时间
void xmrig::Timer::setRepeat(uint64_t repeat)
{
    uv_timer_set_repeat(m_timer, repeat);
}


// 单次触发定时器，传入超时时间和ID
void xmrig::Timer::singleShot(uint64_t timeout, int id)
{
    m_id = id;

    // 停止定时器
    stop();
    // 启动定时器，重复时间为0
    start(timeout, 0);
}


// 启动定时器，传入超时时间和重复时间
void xmrig::Timer::start(uint64_t timeout, uint64_t repeat)
{
    uv_timer_start(m_timer, onTimer, timeout, repeat);
}


// 停止定时器
void xmrig::Timer::stop()
{
    // 设置重复时间为0
    setRepeat(0);
    // 停止定时器
    uv_timer_stop(m_timer);
}


// 初始化定时器
void xmrig::Timer::init()
{
    // 创建 uv_timer_t 对象
    m_timer = new uv_timer_t;
    // 设置对象的数据为 this 指针
    m_timer->data = this;
    // 初始化 uv_timer_t 对象
    uv_timer_init(uv_default_loop(), m_timer);
}


// 定时器触发时的回调函数
void xmrig::Timer::onTimer(uv_timer_t *handle)
{
    // 将 handle 的数据转换为 Timer 对象指针
    const auto timer = static_cast<Timer *>(handle->data);
    # 调用 timer 对象的 m_listener 成员变量的 onTimer 方法，并传入 timer 对象作为参数
    timer->m_listener->onTimer(timer);
# 闭合前面的函数定义
```