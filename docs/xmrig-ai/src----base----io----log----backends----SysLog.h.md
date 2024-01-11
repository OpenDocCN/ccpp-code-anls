# `xmrig\src\base\io\log\backends\SysLog.h`

```
/* XMRig
 * 版权所有 (c) 2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它
 *   其中规定了由自由软件基金会发布的许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SYSLOG_H
#define XMRIG_SYSLOG_H


#include "base/kernel/interfaces/ILogBackend.h"


namespace xmrig {


class SysLog : public ILogBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE(SysLog)

    SysLog();
    ~SysLog() override;

protected:
    void print(uint64_t timestamp, int level, const char *line, size_t offset, size_t size, bool colors) override;
};


} /* namespace xmrig */


#endif /* XMRIG_SYSLOG_H */
```