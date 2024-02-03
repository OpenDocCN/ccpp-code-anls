# `xmrig\src\base\tools\Chrono.cpp`

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
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "Chrono.h"


#ifdef XMRIG_OS_WIN
#   include <Windows.h>
#endif


namespace xmrig {


// 返回高分辨率的毫秒数
double Chrono::highResolutionMSecs()
{
#   ifdef XMRIG_OS_WIN
    LARGE_INTEGER f, t;
    // 获取性能计数器的频率
    QueryPerformanceFrequency(&f);
    // 获取性能计数器的当前值
    QueryPerformanceCounter(&t);
    // 返回毫秒数
    return static_cast<double>(t.QuadPart) * 1e3 / f.QuadPart;
#   else
    using namespace std::chrono;
    // 返回高分辨率时钟的当前时间，转换为毫秒
    return static_cast<uint64_t>(duration_cast<nanoseconds>(high_resolution_clock::now().time_since_epoch()).count()) / 1e6;
#   endif
}


} /* namespace xmrig */
```