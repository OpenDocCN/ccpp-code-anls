# `xmrig\src\core\config\ConfigTransform.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONFIGTRANSFORM_H
#define XMRIG_CONFIGTRANSFORM_H


#include "base/kernel/config/BaseTransform.h"


namespace xmrig {


class ConfigTransform : public BaseTransform
{
protected:
    void finalize(rapidjson::Document &doc) override;
    void transform(rapidjson::Document &doc, int key, const char *arg) override;

private:
    void transformBoolean(rapidjson::Document &doc, int key, bool enable);
    void transformUint64(rapidjson::Document &doc, int key, uint64_t arg);

#   ifdef XMRIG_FEATURE_BENCHMARK
    void transformBenchmark(rapidjson::Document &doc, int key, const char *arg);
#   endif

    bool m_opencl           = false;
    int64_t m_affinity      = -1;
    uint64_t m_intensity    = 1;
    uint64_t m_threads      = 0;
};


} // namespace xmrig


#endif /* XMRIG_CONFIGTRANSFORM_H */
```