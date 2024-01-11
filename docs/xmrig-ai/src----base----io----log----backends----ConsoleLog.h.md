# `xmrig\src\base\io\log\backends\ConsoleLog.h`

```
/* XMRig
 * 版权所有（c）2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONSOLELOG_H
#define XMRIG_CONSOLELOG_H


使用uv_stream_t = struct uv_stream_s;
使用uv_tty_t    = struct uv_tty_s;


包括 "base/kernel/interfaces/ILogBackend.h"
包括 "base/tools/Object.h"


在xmrig命名空间中


类标题;


类ConsoleLog：公共ILogBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE(ConsoleLog)

    ConsoleLog(const Title &title);
    ~ConsoleLog() override;

protected:
    void print(uint64_t timestamp, int level, const char *line, size_t offset, size_t size, bool colors) override;

private:
    static bool isSupported();

    uv_tty_t *m_tty = nullptr;

#   ifdef XMRIG_OS_WIN
    bool isWritable() const;

    uv_stream_t *m_stream = nullptr;
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_CONSOLELOG_H */
```