# `xmrig\src\base\io\log\backends\FileLog.h`

```
/* XMRig
 * 版权所有（c）2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FILELOG_H
#define XMRIG_FILELOG_H


#include "base/io/log/FileLogWriter.h"
#include "base/kernel/interfaces/ILogBackend.h"


namespace xmrig {


class FileLog : public ILogBackend
{
public:
    FileLog(const char *fileName);

protected:
    void print(uint64_t timestamp, int level, const char *line, size_t offset, size_t size, bool colors) override;

private:
    FileLogWriter m_writer;
};


} /* namespace xmrig */


#endif /* XMRIG_FILELOG_H */
```