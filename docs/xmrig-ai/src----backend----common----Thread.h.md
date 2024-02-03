# `xmrig\src\backend\common\Thread.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_THREAD_H
#define XMRIG_THREAD_H


#include "backend/common/interfaces/IWorker.h"


#include <thread>


#ifdef XMRIG_OS_APPLE
#   include <pthread.h>
#   include <mach/thread_act.h>
#endif


namespace xmrig {


class IBackend;


template<class T>
class Thread
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Thread)

    // 构造函数，初始化线程的 ID、配置和后端
    inline Thread(IBackend *backend, size_t id, const T &config) : m_id(id), m_config(config), m_backend(backend) {}

#   ifdef XMRIG_OS_APPLE
    // 析构函数，等待线程结束并删除工作线程
    inline ~Thread() { pthread_join(m_thread, nullptr); delete m_worker; }

    // 启动线程，传入回调函数
    inline void start(void *(*callback)(void *))
    {
        // 如果配置的亲和性大于等于0
        if (m_config.affinity >= 0) {
            // 创建一个挂起的线程
            pthread_create_suspended_np(&m_thread, nullptr, callback, this);

            // 获取线程的 Mach 端口
            mach_port_t mach_thread              = pthread_mach_thread_np(m_thread);
            // 设置线程的亲和性策略
            thread_affinity_policy_data_t policy = { static_cast<integer_t>(m_config.affinity + 1) };
            thread_policy_set(mach_thread, THREAD_AFFINITY_POLICY, reinterpret_cast<thread_policy_t>(&policy), THREAD_AFFINITY_POLICY_COUNT);
            // 恢复线程执行
            thread_resume(mach_thread);
        }
        // 如果配置的亲和性小于0
        else {
            // 创建一个线程
            pthread_create(&m_thread, nullptr, callback, this);
        }
    }
// 线程类的析构函数，当线程对象被销毁时，会等待线程执行完成并删除工作对象
inline ~Thread() { m_thread.join(); delete m_worker; }

// 启动线程，传入回调函数指针作为参数
inline void start(void *(*callback)(void *))    { m_thread = std::thread(callback, this); }

// 返回配置对象的常引用
inline const T &config() const                  { return m_config; }

// 返回后端对象的常指针
inline IBackend *backend() const                { return m_backend; }

// 返回工作对象的常指针
inline IWorker *worker() const                  { return m_worker; }

// 返回线程的ID
inline size_t id() const                        { return m_id; }

// 设置工作对象
inline void setWorker(IWorker *worker)          { m_worker = worker; }

// 线程类的私有成员变量
private:
    const size_t m_id    = 0;  // 线程的ID，默认为0
    const T m_config;  // 配置对象
    IBackend *m_backend;  // 后端对象指针
    IWorker *m_worker       = nullptr;  // 工作对象指针，默认为空指针

    #ifdef XMRIG_OS_APPLE
    pthread_t m_thread{};  // macOS平台的线程对象
#   else
    std::thread m_thread;  // 其他平台的线程对象
#   endif
};


} // namespace xmrig

#endif /* XMRIG_THREAD_H */
```