# `xmrig\src\base\kernel\Platform_win.cpp`

```
/* XMRig
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <algorithm>
#include <winsock2.h>
#include <windows.h>
#include <uv.h>
#include <limits>


#include "base/kernel/Platform.h"
#include "version.h"


static inline OSVERSIONINFOEX winOsVersion()
{
    typedef NTSTATUS (NTAPI *RtlGetVersionFunction)(LPOSVERSIONINFO); // NOLINT(modernize-use-using)
    OSVERSIONINFOEX result = { sizeof(OSVERSIONINFOEX), 0, 0, 0, 0, {'\0'}, 0, 0, 0, 0, 0};

    HMODULE ntdll = GetModuleHandleW(L"ntdll.dll");
    if (ntdll ) {
        auto pRtlGetVersion = reinterpret_cast<RtlGetVersionFunction>(GetProcAddress(ntdll, "RtlGetVersion"));

        if (pRtlGetVersion) {
            pRtlGetVersion(reinterpret_cast<LPOSVERSIONINFO>(&result));
        }
    }

    return result;
}


char *xmrig::Platform::createUserAgent()
{
    const auto osver = winOsVersion();
    constexpr const size_t max = 256;

    char *buf = new char[max](); // 创建一个大小为max的字符数组，并初始化为0
    int length = snprintf(buf, max, "%s/%s (Windows NT %lu.%lu", APP_NAME, APP_VERSION, osver.dwMajorVersion, osver.dwMinorVersion); // 将格式化的字符串写入buf中

#   if defined(__x86_64__) || defined(_M_AMD64)
    length += snprintf(buf + length, max - length, "; Win64; x64) libuv/%s", uv_version_string()); // 将格式化的字符串追加到buf中
#   else
    # 如果不满足上面的条件，执行下面的代码
    length += snprintf(buf + length, max - length, ") libuv/%s", uv_version_string());
#   endif

#   ifdef __GNUC__
    # 如果定义了 __GNUC__，执行下面的代码
    snprintf(buf + length, max - length, " gcc/%d.%d.%d", __GNUC__, __GNUC_MINOR__, __GNUC_PATCHLEVEL__);
#   elif _MSC_VER
    # 如果定义了 _MSC_VER，执行下面的代码
    snprintf(buf + length, max - length, " msvc/%d", MSVC_VERSION);
#   endif

    # 返回 buf
    return buf;
}


bool xmrig::Platform::hasKeepalive()
{
    # 返回当前操作系统版本是否大于等于 6
    return winOsVersion().dwMajorVersion >= 6;
}


#ifndef XMRIG_FEATURE_HWLOC
bool xmrig::Platform::setThreadAffinity(uint64_t cpu_id)
{
    # 设置当前线程的亲和性
    const bool result = (SetThreadAffinityMask(GetCurrentThread(), 1ULL << cpu_id) != 0);
    # 等待 1 毫秒
    Sleep(1);
    return result;
}
#endif


void xmrig::Platform::setProcessPriority(int priority)
{
    # 如果优先级为 -1，直接返回
    if (priority == -1) {
        return;
    }

    # 设置进程的优先级
    DWORD prio = IDLE_PRIORITY_CLASS;
    switch (priority)
    {
    case 1:
        prio = BELOW_NORMAL_PRIORITY_CLASS;
        break;

    case 2:
        prio = NORMAL_PRIORITY_CLASS;
        break;

    case 3:
        prio = ABOVE_NORMAL_PRIORITY_CLASS;
        break;

    case 4:
        prio = HIGH_PRIORITY_CLASS;
        break;

    case 5:
        prio = REALTIME_PRIORITY_CLASS;
        break;

    default:
        break;
    }

    SetPriorityClass(GetCurrentProcess(), prio);
}


void xmrig::Platform::setThreadPriority(int priority)
{
    # 如果优先级为 -1，直接返回
    if (priority == -1) {
        return;
    }

    # 设置线程的优先级
    int prio = THREAD_PRIORITY_IDLE;
    switch (priority)
    {
    case 1:
        prio = THREAD_PRIORITY_BELOW_NORMAL;
        break;

    case 2:
        prio = THREAD_PRIORITY_NORMAL;
        break;

    case 3:
        prio = THREAD_PRIORITY_ABOVE_NORMAL;
        break;

    case 4:
        prio = THREAD_PRIORITY_HIGHEST;
        break;

    case 5:
        prio = THREAD_PRIORITY_TIME_CRITICAL;
        break;

    default:
        break;
    }

    SetThreadPriority(GetCurrentThread(), prio);
}


bool xmrig::Platform::isOnBatteryPower()
{
    # 获取系统电源状态
    SYSTEM_POWER_STATUS st;
    # 如果成功获取系统电源状态
    if (GetSystemPowerStatus(&st)) {
        # 返回是否为电池供电
        return (st.ACLineStatus == 0);
    }
    # 获取系统电源状态失败，返回 false
    return false;
# 返回系统空闲时间的函数，返回一个 64 位无符号整数
uint64_t xmrig::Platform::idleTime()
{
    # 创建一个 LASTINPUTINFO 结构体对象，并设置其大小
    LASTINPUTINFO info{};
    info.cbSize = sizeof(LASTINPUTINFO);

    # 如果无法获取用户最后输入信息，则返回最大的 64 位无符号整数
    if (!GetLastInputInfo(&info)) {
        return std::numeric_limits<uint64_t>::max();
    }

    # 返回系统当前时间与用户最后输入时间的差值，转换为 64 位无符号整数
    return static_cast<uint64_t>(GetTickCount() - info.dwTime);
}
```