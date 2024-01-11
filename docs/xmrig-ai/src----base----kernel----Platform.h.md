# `xmrig\src\base\kernel\Platform.h`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   许可证的版本为 3 或 (在您的选择下) 任何更高版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_PLATFORM_H
#define XMRIG_PLATFORM_H


#include <cstdint>


#include "base/tools/String.h"


namespace xmrig {


class Platform
{
public:
    static inline bool trySetThreadAffinity(int64_t cpu_id)
    {
        if (cpu_id < 0) {
            return false;
        }

        return setThreadAffinity(static_cast<uint64_t>(cpu_id));
    }

    static bool setThreadAffinity(uint64_t cpu_id);
    static void init(const char *userAgent);
    static void setProcessPriority(int priority);
    static void setThreadPriority(int priority);

    static inline bool isUserActive(uint64_t ms)    { return idleTime() < ms; }
    static inline const String &userAgent()         { return m_userAgent; }

#   ifdef XMRIG_OS_WIN
    static bool hasKeepalive();
#   else
    static constexpr bool hasKeepalive()            { return true; }
#   endif

    static bool isOnBatteryPower();
    static uint64_t idleTime();

private:
    static char *createUserAgent();

    static String m_userAgent;
};


} // namespace xmrig


#endif /* XMRIG_PLATFORM_H */
```