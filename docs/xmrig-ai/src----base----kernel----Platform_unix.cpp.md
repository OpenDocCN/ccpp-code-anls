# `xmrig\src\base\kernel\Platform_unix.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifdef XMRIG_OS_FREEBSD
#   include <sys/types.h>
#   include <sys/param.h>
#   ifndef __DragonFly__
#       include <sys/cpuset.h>
#   endif
#   include <pthread_np.h>
#endif


#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/resource.h>
#include <unistd.h>
#include <uv.h>
#include <thread>
#include <fstream>
#include <limits>


#include "base/kernel/Platform.h"
#include "version.h"


char *xmrig::Platform::createUserAgent()
{
    // 定义最大长度为256
    constexpr const size_t max = 256;

    // 创建一个长度为max的字符数组，并初始化为0
    char *buf = new char[max]();
    // 格式化字符串到buf中，返回格式化后的长度
    int length = snprintf(buf, max, "%s/%s (Linux ", APP_NAME, APP_VERSION);

#   if defined(__x86_64__)
    // 根据不同的架构格式化字符串并更新长度
    length += snprintf(buf + length, max - length, "x86_64) libuv/%s", uv_version_string());
#   elif defined(__aarch64__)
    length += snprintf(buf + length, max - length, "aarch64) libuv/%s", uv_version_string());
#   elif defined(__arm__)
    length += snprintf(buf + length, max - length, "arm) libuv/%s", uv_version_string());
#   else
    length += snprintf(buf + length, max - length, "i686) libuv/%s", uv_version_string());
#   endif

#   ifdef __clang__
    # 将 " clang/" 后面的内容格式化并添加到 buf 中，返回添加后的总长度
    length += snprintf(buf + length, max - length, " clang/%d.%d.%d", __clang_major__, __clang_minor__, __clang_patchlevel__);
#   elif defined(__GNUC__)
    # 如果定义了 __GNUC__，则执行以下代码
    length += snprintf(buf + length, max - length, " gcc/%d.%d.%d", __GNUC__, __GNUC_MINOR__, __GNUC_PATCHLEVEL__);
    # 将 gcc 的版本信息格式化后添加到 buf 中

#   endif
    # 结束条件编译指令

    return buf;
    # 返回 buf


#ifndef XMRIG_FEATURE_HWLOC
    # 如果未定义 XMRIG_FEATURE_HWLOC，则执行以下代码
#ifdef __DragonFly__
    # 如果定义了 __DragonFly__，则执行以下代码
bool xmrig::Platform::setThreadAffinity(uint64_t cpu_id)
{
    return true;
}
    # 返回 true

#else
    # 如果未定义 __DragonFly__，则执行以下代码
#ifdef XMRIG_OS_FREEBSD
    # 如果定义了 XMRIG_OS_FREEBSD，则执行以下代码
typedef cpuset_t cpu_set_t;
    # 定义 cpuset_t 为 cpu_set_t
#endif
    # 结束条件编译指令

bool xmrig::Platform::setThreadAffinity(uint64_t cpu_id)
{
    # 设置线程亲和性
    cpu_set_t mn;
    CPU_ZERO(&mn);
    CPU_SET(cpu_id, &mn);

#   ifndef __ANDROID__
    # 如果未定义 __ANDROID__，则执行以下代码
    const bool result = (pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &mn) == 0);
    # 设置当前线程的 CPU 亲和性
#   else
    # 如果定义了 __ANDROID__，则执行以下代码
    const bool result = (sched_setaffinity(gettid(), sizeof(cpu_set_t), &mn) == 0);
    # 设置当前线程的 CPU 亲和性
#   endif
    # 结束条件编译指令

    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    # 当前线程休眠 1 毫秒
    return result;
    # 返回结果
}

#endif // __DragonFly__
#endif // XMRIG_FEATURE_HWLOC
    # 结束条件编译指令


void xmrig::Platform::setProcessPriority(int)
{
    # 设置进程优先级
}


void xmrig::Platform::setThreadPriority(int priority)
{
    # 设置线程优先级
    if (priority == -1) {
        return;
    }

    int prio = 19;
    switch (priority)
    {
    case 1:
        prio = 5;
        break;

    case 2:
        prio = 0;
        break;

    case 3:
        prio = -5;
        break;

    case 4:
        prio = -10;
        break;

    case 5:
        prio = -15;
        break;

    default:
        break;
    }

    setpriority(PRIO_PROCESS, 0, prio);
    # 设置进程的优先级

#   ifdef SCHED_IDLE
    # 如果定义了 SCHED_IDLE，则执行以下代码
    if (priority == 0) {
        sched_param param;
        param.sched_priority = 0;

        if (sched_setscheduler(0, SCHED_IDLE, &param) != 0) {
            sched_setscheduler(0, SCHED_BATCH, &param);
        }
    }
    # 设置调度策略为 SCHED_IDLE 或 SCHED_BATCH
#   endif
}


bool xmrig::Platform::isOnBatteryPower()
{
    # 判断是否在使用电池供电
    for (int i = 0; i <= 1; ++i) {
        char buf[64];
        snprintf(buf, 64, "/sys/class/power_supply/BAT%d/status", i);
        std::ifstream f(buf);
        if (f.is_open()) {
            std::string status;
            f >> status;
            return (status == "Discharging");
        }
    }
    return false;
}
# 返回一个最大值，表示系统空闲时间
uint64_t xmrig::Platform::idleTime()
{
    return std::numeric_limits<uint64_t>::max();
}
```