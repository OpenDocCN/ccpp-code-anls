# `xmrig\src\crypto\rx\RxMsr.h`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布本程序的副本
 *   或者（根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RXMSR_H
#define XMRIG_RXMSR_H


#include <vector>


namespace xmrig
{


class CpuThread;
class RxConfig;


class RxMsr
{
public:
    // 返回 MSR 功能是否启用
    static inline bool isEnabled()      { return m_enabled; }
    // 返回 MSR 功能是否已初始化
    static inline bool isInitialized()  { return m_initialized; }

    // 初始化 MSR 功能
    static bool init(const RxConfig &config, const std::vector<CpuThread> &threads);
    // 销毁 MSR 功能
    static void destroy();

private:
    static bool m_cacheQoS;
    static bool m_enabled;
    static bool m_initialized;
};


} /* namespace xmrig */


#endif /* XMRIG_RXMSR_H */


注释：
```