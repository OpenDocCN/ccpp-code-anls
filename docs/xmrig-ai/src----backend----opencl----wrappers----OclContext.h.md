# `xmrig\src\backend\opencl\wrappers\OclContext.h`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 许可证的版本为 3 或（根据您的选择）更高版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLCONTEXT_H
#define XMRIG_OCLCONTEXT_H


#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/wrappers/OclDevice.h"
#include "base/tools/Object.h"


using cl_context = struct _cl_context *;


namespace xmrig {


class Job;


class OclContext
{
public:
    XMRIG_DISABLE_COPY_MOVE(OclContext)

    OclContext() = default;
    OclContext(const OclDevice &device);
    ~OclContext();

    bool init(const std::vector<OclDevice> &devices, std::vector<OclLaunchData> &threads);

    inline bool isValid() const     { return m_ctx != nullptr; }  // 检查上下文是否有效
    inline cl_context ctx() const   { return m_ctx; }  // 返回上下文

private:
    cl_context m_ctx = nullptr;  // 上下文对象
};


} // namespace xmrig


#endif /* XMRIG_OCLCONTEXT_H */
```