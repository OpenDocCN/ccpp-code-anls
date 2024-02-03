# `xmrig\src\base\io\Watcher.h`

```cpp
/* XMRig
 * 版权声明
 */

#ifndef XMRIG_WATCHER_H
#define XMRIG_WATCHER_H

#include "base/kernel/interfaces/ITimerListener.h"  // 引入 ITimerListener 接口
#include "base/tools/String.h"  // 引入 String 类

typedef struct uv_fs_event_s uv_fs_event_t;  // 定义 uv_fs_event_t 结构体类型

namespace xmrig {

class IWatcherListener;  // 声明 IWatcherListener 类
class Timer;  // 声明 Timer 类

class Watcher : public ITimerListener  // Watcher 类继承自 ITimerListener 接口
{
public:
    Watcher(const String &path, IWatcherListener *listener);  // Watcher 类的构造函数
    ~Watcher() override;  // Watcher 类的析构函数

protected:
    inline void onTimer(const Timer *) override { reload(); }  // 定义 ITimerListener 接口的 onTimer 方法

private:
    constexpr static int kDelay = 500;  // 定义静态常量 kDelay

    static void onFsEvent(uv_fs_event_t *handle, const char *filename, int events, int status);  // 定义静态方法 onFsEvent

    void queueUpdate();  // 声明 queueUpdate 方法
    void reload();  // 声明 reload 方法
    void start();  // 声明 start 方法

    IWatcherListener *m_listener;  // 声明 IWatcherListener 类型的成员变量 m_listener
    String m_path;  // 声明 String 类型的成员变量 m_path
    Timer *m_timer;  // 声明 Timer 类型的指针成员变量 m_timer
    uv_fs_event_t *m_fsEvent;  // 声明 uv_fs_event_t 类型的指针成员变量 m_fsEvent
}; 
// 结束了匿名命名空间的定义

} /* namespace xmrig */
// 结束了 xmrig 命名空间的定义

#endif /* XMRIG_WATCHER_H */
// 结束了 XMRIG_WATCHER_H 的条件编译
```