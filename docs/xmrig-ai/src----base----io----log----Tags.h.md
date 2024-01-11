# `xmrig\src\base\io\log\Tags.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TAGS_H
#define XMRIG_TAGS_H

#include <cstddef>
#include <cstdint>

namespace xmrig {

class Tags
{
public:
    static const char *config();  // 返回配置标签的指针
    static const char *network();  // 返回网络标签的指针
    static const char *origin();  // 返回原始标签的指针
    static const char *signal();  // 返回信号标签的指针

#   ifdef XMRIG_MINER_PROJECT
    static const char *cpu();  // 如果XMRIG_MINER_PROJECT定义了，返回CPU标签的指针
    static const char *miner();  // 如果XMRIG_MINER_PROJECT定义了，返回矿工标签的指针
#   ifdef XMRIG_ALGO_RANDOMX
    static const char *randomx();  // 如果XMRIG_MINER_PROJECT和XMRIG_ALGO_RANDOMX都定义了，返回RandomX标签的指针
#   endif
#   ifdef XMRIG_FEATURE_BENCHMARK
    static const char *bench();  // 如果XMRIG_MINER_PROJECT和XMRIG_FEATURE_BENCHMARK都定义了，返回基准标签的指针
#   endif
#   endif

#   ifdef XMRIG_PROXY_PROJECT
    static const char *proxy();  // 如果XMRIG_PROXY_PROJECT定义了，返回代理标签的指针
#   endif

#   ifdef XMRIG_FEATURE_CUDA
    static const char *nvidia();  // 如果XMRIG_FEATURE_CUDA定义了，返回NVIDIA标签的指针
#   endif

#   ifdef XMRIG_FEATURE_OPENCL
    static const char *opencl();  // 如果XMRIG_FEATURE_OPENCL定义了，返回OpenCL标签的指针
#   endif

#   ifdef XMRIG_FEATURE_PROFILING
    static const char* profiler();  // 如果XMRIG_FEATURE_PROFILING定义了，返回性能分析标签的指针
#   endif
};

} /* namespace xmrig */

#endif /* XMRIG_TAGS_H */
```