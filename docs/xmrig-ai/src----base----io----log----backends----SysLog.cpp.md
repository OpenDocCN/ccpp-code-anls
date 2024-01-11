# `xmrig\src\base\io\log\backends\SysLog.cpp`

```
/*
 * XMRig
 * 版权所有 (c) 2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本3或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <syslog>


#include "base/io/log/backends/SysLog.h"
#include "version.h"


// SysLog类的构造函数
xmrig::SysLog::SysLog()
{
    // 打开系统日志，使用应用程序ID，记录进程ID，记录到LOG_USER
    openlog(APP_ID, LOG_PID, LOG_USER);
}

// SysLog类的析构函数
xmrig::SysLog::~SysLog()
{
    // 关闭系统日志
    closelog();
}

// 打印日志
void xmrig::SysLog::print(uint64_t, int level, const char *line, size_t offset, size_t, bool colors)
{
    // 如果启用了颜色，则返回
    if (colors) {
        return;
    }
    // 否则，根据日志级别记录日志
    syslog(level == -1 ? LOG_INFO : level, "%s", line + offset);
}
```