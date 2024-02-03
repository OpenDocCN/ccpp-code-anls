# `xmrig\src\backend\cuda\wrappers\NvmlLib.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NVMLLIB_H
#define XMRIG_NVMLLIB_H

#include "backend/cuda/wrappers/CudaDevice.h"
#include "backend/cuda/wrappers/NvmlHealth.h"

namespace xmrig {

class NvmlLib
{
public:
    static bool init(const char *fileName = nullptr);
    static const char *lastError() noexcept;
    static void close();

    static bool assign(std::vector<CudaDevice> &devices);
    static NvmlHealth health(nvmlDevice_t device);

    static inline bool isInitialized() noexcept         { return m_initialized; }
    static inline bool isReady() noexcept               { return m_ready; }
    static inline const char *driverVersion() noexcept  { return m_driverVersion; }
    static inline const char *version() noexcept        { return m_nvmlVersion; }

private:
    static bool dlopen();
    static bool load();

    static bool m_initialized;
    static bool m_ready;
    static char m_driverVersion[80];
    static char m_nvmlVersion[80];
    static String m_loader;
};

} // namespace xmrig

#endif /* XMRIG_NVMLLIB_H */
```