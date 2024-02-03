# `xmrig\src\base\io\log\Tags.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   许可证的第3版或（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/io/log/Tags.h"
#include "base/io/log/Log.h"


const char *xmrig::Tags::config()
{
    static const char *tag = CYAN_BG_BOLD(WHITE_BOLD_S " config  ");

    return tag;
}


const char *xmrig::Tags::network()
{
    static const char *tag = BLUE_BG_BOLD(WHITE_BOLD_S " net     ");

    return tag;
}


const char* xmrig::Tags::origin()
{
    static const char* tag = YELLOW_BG_BOLD(WHITE_BOLD_S " origin  ");

    return tag;
}


const char *xmrig::Tags::signal()
{
    static const char *tag = YELLOW_BG_BOLD(WHITE_BOLD_S " signal  ");

    return tag;
}


#ifdef XMRIG_MINER_PROJECT
const char *xmrig::Tags::cpu()
{
    static const char *tag = CYAN_BG_BOLD(WHITE_BOLD_S " cpu     ");

    return tag;
}


const char *xmrig::Tags::miner()
{
    static const char *tag = MAGENTA_BG_BOLD(WHITE_BOLD_S " miner   ");

    return tag;
}


#ifdef XMRIG_ALGO_RANDOMX
const char *xmrig::Tags::randomx()
{
    static const char *tag = BLUE_BG(WHITE_BOLD_S " randomx ") " ";

    return tag;
}
#endif


#ifdef XMRIG_FEATURE_BENCHMARK
const char *xmrig::Tags::bench()
{
    static const char *tag = GREEN_BG_BOLD(WHITE_BOLD_S " bench   ");

    return tag;
#ifdef XMRIG_PROXY_PROJECT
// 如果定义了 XMRIG_PROXY_PROJECT 宏，则编译以下代码
const char *xmrig::Tags::proxy()
{
    // 静态变量，代表带有颜色和样式的字符串 " proxy   "
    static const char *tag = MAGENTA_BG_BOLD(WHITE_BOLD_S " proxy   ");

    // 返回静态变量 tag
    return tag;
}
#endif

#ifdef XMRIG_FEATURE_CUDA
// 如果定义了 XMRIG_FEATURE_CUDA 宏，则编译以下代码
const char *xmrig::Tags::nvidia()
{
    // 静态变量，代表带有颜色和样式的字符串 " nvidia  "
    static const char *tag = GREEN_BG_BOLD(WHITE_BOLD_S " nvidia  ");

    // 返回静态变量 tag
    return tag;
}
#endif

#ifdef XMRIG_FEATURE_OPENCL
// 如果定义了 XMRIG_FEATURE_OPENCL 宏，则编译以下代码
const char *xmrig::Tags::opencl()
{
    // 静态变量，代表带有颜色和样式的字符串 " opencl  "
    static const char *tag = MAGENTA_BG_BOLD(WHITE_BOLD_S " opencl  ");

    // 返回静态变量 tag
    return tag;
}
#endif

#ifdef XMRIG_FEATURE_PROFILING
// 如果定义了 XMRIG_FEATURE_PROFILING 宏，则编译以下代码
const char* xmrig::Tags::profiler()
{
    // 静态变量，代表带有颜色和样式的字符串 " profile "
    static const char* tag = CYAN_BG_BOLD(WHITE_BOLD_S " profile ");

    // 返回静态变量 tag
    return tag;
}
#endif
```