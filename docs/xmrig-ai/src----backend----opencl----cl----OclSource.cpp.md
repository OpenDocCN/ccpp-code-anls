# `xmrig\src\backend\opencl\cl\OclSource.cpp`

```
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/opencl/cl/OclSource.h"
#include "backend/opencl/cl/cn/cryptonight_cl.h"
#include "base/crypto/Algorithm.h"


#ifdef XMRIG_ALGO_RANDOMX
#   include "backend/opencl/cl/rx/randomx_cl.h"
#endif

#ifdef XMRIG_ALGO_KAWPOW
#   include "backend/opencl/cl/kawpow/kawpow_cl.h"
#   include "backend/opencl/cl/kawpow/kawpow_dag_cl.h"
#endif


// 获取指定算法的 OpenCL 源代码
const char *xmrig::OclSource::get(const Algorithm &algorithm)
{
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果算法家族是 RANDOM_X，则返回 randomx_cl
    if (algorithm.family() == Algorithm::RANDOM_X) {
        return randomx_cl;
    }
#   endif

#   ifdef XMRIG_ALGO_KAWPOW
    // 如果算法家族是 KAWPOW，则返回 kawpow_dag_cl
    if (algorithm.family() == Algorithm::KAWPOW) {
        return kawpow_dag_cl;
    }
#   endif

    // 默认返回 cryptonight_cl
    return cryptonight_cl;
}
```