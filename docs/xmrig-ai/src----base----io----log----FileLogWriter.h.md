# `xmrig\src\base\io\log\FileLogWriter.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FILELOGWRITER_H
#define XMRIG_FILELOGWRITER_H


#include <cstddef>
#include <cstdint>


namespace xmrig {


class FileLogWriter
{
public:
    FileLogWriter() = default;
    FileLogWriter(const char *fileName) { open(fileName); }

    // 检查文件是否打开
    inline bool isOpen() const  { return m_file >= 0; }
    // 返回当前文件位置
    inline int64_t pos() const  { return m_pos; }

    // 打开文件
    bool open(const char *fileName);
    // 写入数据到文件
    bool write(const char *data, size_t size);
    // 写入一行数据到文件
    bool writeLine(const char *data, size_t size);

private:
#   ifdef XMRIG_OS_WIN
    const char m_endl[3]  = {'\r', '\n', 0};
#   else
    const char m_endl[2]  = {'\n', 0};
#   endif

    int m_file      = -1;
    int64_t m_pos   = 0;
};


} /* namespace xmrig */


#endif /* XMRIG_FILELOGWRITER_H */
```