# `xmrig\src\backend\opencl\wrappers\AdlLib.h`

```
/* XMRig
 * 版权所有 (c) 2008-2018 高级微处理器公司
 * 版权所有 (c) 2018-2021 SChernykh                    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig                        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ADLLIB_H
#define XMRIG_ADLLIB_H


#include "backend/opencl/wrappers/AdlHealth.h"


namespace xmrig {


class OclDevice;


class AdlLib
{
public:
    static bool init();
    static const char *lastError() noexcept;
    static void close();

    static AdlHealth health(const OclDevice &device);

    static inline bool isInitialized() noexcept         { return m_initialized; }
    static inline bool isReady() noexcept               { return m_ready; }

private:
    static bool dlopen();
    static bool load();

    static bool m_initialized;
    static bool m_ready;
};


} // namespace xmrig


#endif /* XMRIG_ADLLIB_H */
```