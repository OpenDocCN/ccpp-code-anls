# `xmrig\src\base\io\log\backends\FileLog.cpp`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用而分发的，但没有任何担保；甚至没有暗示的担保适用于特定目的。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/io/log/backends/FileLog.h"

#include <cassert>
#include <cstring>

// 使用命名空间 xmrig
xmrig::FileLog::FileLog(const char *fileName) :
    m_writer(fileName)  // 使用文件名初始化 m_writer
{
}

// 打印日志
void xmrig::FileLog::print(uint64_t, int, const char *line, size_t, size_t size, bool colors)
{
    // 如果文件未打开或者需要使用颜色，则直接返回
    if (!m_writer.isOpen() || colors) {
        return;
    }

    // 断言行的长度等于指定的大小
    assert(strlen(line) == size);

    // 写入日志内容到文件
    m_writer.write(line, size);
}
```