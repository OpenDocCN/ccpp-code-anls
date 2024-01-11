# `xmrig\src\base\kernel\Platform_mac.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <IOKit/IOKitLib.h>
#include <IOKit/ps/IOPowerSources.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/resource.h>
#include <uv.h>
#include <thread>
#include <fstream>


#include "base/kernel/Platform.h"
#include "version.h"


char *xmrig::Platform::createUserAgent()
{
    // 创建用户代理字符串
    constexpr const size_t max = 256;

    char *buf = new char[max]();
    // 格式化用户代理字符串
    int length = snprintf(buf, max,
                          "%s/%s (Macintosh; macOS"
#                         ifdef XMRIG_ARM
                          "; arm64"
#                         else
                          "; x86_64"
#                         endif
                          ") libuv/%s", APP_NAME, APP_VERSION, uv_version_string());

#   ifdef __clang__
    // 添加编译器信息到用户代理字符串
    length += snprintf(buf + length, max - length, " clang/%d.%d.%d", __clang_major__, __clang_minor__, __clang_patchlevel__);
#   elif defined(__GNUC__)
    // 添加编译器信息到用户代理字符串
    length += snprintf(buf + length, max - length, " gcc/%d.%d.%d", __GNUC__, __GNUC_MINOR__, __GNUC_PATCHLEVEL__);
#   endif

    return buf;
}


bool xmrig::Platform::setThreadAffinity(uint64_t cpu_id)
{
    // 设置线程亲和性
    return true;
}


void xmrig::Platform::setProcessPriority(int)
{
    // 设置进程优先级
# 设置线程优先级的函数
void xmrig::Platform::setThreadPriority(int priority)
{
    # 如果优先级为-1，则直接返回
    if (priority == -1) {
        return;
    }

    # 默认优先级为19
    int prio = 19;
    # 根据传入的优先级设置不同的值
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

    # 设置进程的优先级
    setpriority(PRIO_PROCESS, 0, prio);
}

# 检查系统是否在使用电池供电
bool xmrig::Platform::isOnBatteryPower()
{
    # 返回系统估计的剩余时间是否不是无限的
    return IOPSGetTimeRemainingEstimate() != kIOPSTimeRemainingUnlimited;
}

# 获取系统的空闲时间
uint64_t xmrig::Platform::idleTime()
{
    # 初始化空闲时间为0
    uint64_t idle_time  = 0;
    # 获取IOHIDSystem服务
    const auto service  = IOServiceGetMatchingService(kIOMasterPortDefault, IOServiceMatching("IOHIDSystem"));
    # 获取HIDIdleTime属性
    const auto property = IORegistryEntryCreateCFProperty(service, CFSTR("HIDIdleTime"), kCFAllocatorDefault, 0);

    # 从属性中获取空闲时间值
    CFNumberGetValue((CFNumberRef)property, kCFNumberSInt64Type, &idle_time);

    # 释放属性和服务
    CFRelease(property);
    IOObjectRelease(service);

    # 返回空闲时间（以毫秒为单位）
    return idle_time / 1000000U;
}
```