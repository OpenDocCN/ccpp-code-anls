# `xmrig\src\base\net\tools\LineReader.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2020      cohcho      <https://github.com/cohcho>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_LINEREADER_H
#define XMRIG_LINEREADER_H


#include "base/tools/Object.h"


#include <cstddef>


namespace xmrig {


class ILineListener;


class LineReader
{
public:
    XMRIG_DISABLE_COPY_MOVE(LineReader)

    LineReader() = default;
    LineReader(ILineListener *listener) : m_listener(listener) {}
    ~LineReader();

    inline void setListener(ILineListener *listener) { m_listener = listener; }

    void parse(char *data, size_t size);
    void reset();

private:
    void add(const char *data, size_t size);
    void getline(char *data, size_t size);

    char *m_buf                 = nullptr;
    ILineListener *m_listener   = nullptr;
    size_t m_pos                = 0;
};


} /* namespace xmrig */


#endif /* XMRIG_NETBUFFER_H */
```