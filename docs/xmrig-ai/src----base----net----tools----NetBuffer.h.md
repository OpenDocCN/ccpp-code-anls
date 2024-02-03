# `xmrig\src\base\net\tools\NetBuffer.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NETBUFFER_H
#define XMRIG_NETBUFFER_H


struct uv_buf_t;
using uv_handle_t = struct uv_handle_s;


#include <cstddef>


namespace xmrig {


class NetBuffer
{
public:
    static char *allocate();
    static void destroy();
    static void onAlloc(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf);
    static void release(const char *buf);
    static void release(const uv_buf_t *buf);
};


} /* namespace xmrig */


#endif /* XMRIG_NETBUFFER_H */
```