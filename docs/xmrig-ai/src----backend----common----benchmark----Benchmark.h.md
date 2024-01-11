# `xmrig\src\backend\common\benchmark\Benchmark.h`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BENCHMARK_H
#define XMRIG_BENCHMARK_H


#include "base/tools/Object.h"


namespace xmrig {


class IBackend;


class Benchmark
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Benchmark)

    Benchmark(size_t workers, const IBackend *backend);
    ~Benchmark() = default;

    bool finish(uint64_t totalHashCount);
    void printProgress() const;
    void start();

private:
    const IBackend *m_backend;
    const size_t m_workers;
    uint64_t m_current          = 0;
    uint64_t m_startTime        = 0;
};


} // namespace xmrig


#endif /* XMRIG_BENCHMARK_H */
```