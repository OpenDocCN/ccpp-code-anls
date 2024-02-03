# `xmrig\src\base\io\Watcher.cpp`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   该许可证由自由软件基金会发布，无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何担保；甚至没有暗示的担保适用于特定目的。有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */


#include <uv.h>


#include "base/kernel/interfaces/IWatcherListener.h"
#include "base/io/Watcher.h"
#include "base/tools/Handle.h"
#include "base/tools/Timer.h"


xmrig::Watcher::Watcher(const String &path, IWatcherListener *listener) :
    m_listener(listener),  // 初始化监听器
    m_path(path)  // 初始化路径
{
    m_timer = new Timer(this);  // 创建一个计时器对象

    m_fsEvent = new uv_fs_event_t;  // 创建一个文件系统事件对象
    m_fsEvent->data = this;  // 将当前对象作为数据存储在文件系统事件对象中
    uv_fs_event_init(uv_default_loop(), m_fsEvent);  // 初始化文件系统事件对象

    start();  // 启动监视
}


xmrig::Watcher::~Watcher()
{
    delete m_timer;  // 删除计时器对象

    Handle::close(m_fsEvent);  // 关闭文件系统事件对象
}


void xmrig::Watcher::onFsEvent(uv_fs_event_t *handle, const char *filename, int, int)
{
    if (!filename) {  // 如果文件名为空
        return;  // 退出函数
    }

    static_cast<Watcher *>(handle->data)->queueUpdate();  // 将文件系统事件对象中的数据转换为监视对象，并调用更新队列函数
}


void xmrig::Watcher::queueUpdate()
{
    # 停止定时器
    m_timer->stop();
    # 重新启动定时器，设置延迟时间为kDelay，重复次数为0（表示无限重复）
    m_timer->start(kDelay, 0);
# 重新加载Watcher对象
void xmrig::Watcher::reload()
{
    # 调用监听器的onFileChanged方法，传入文件路径参数
    m_listener->onFileChanged(m_path);
    
    # 如果不是在Windows平台
    # 停止文件系统事件监听
    uv_fs_event_stop(m_fsEvent);
    # 重新开始文件系统事件监听
    start();
}

# 开始文件系统事件监听
void xmrig::Watcher::start()
{
    # 启动文件系统事件监听，传入回调函数onFsEvent、文件路径和标志参数
    uv_fs_event_start(m_fsEvent, xmrig::Watcher::onFsEvent, m_path, 0);
}
```