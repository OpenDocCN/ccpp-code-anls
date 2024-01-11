# `xmrig\src\backend\opencl\wrappers\OclContext.cpp`

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
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/opencl/wrappers/OclContext.h"
#include "backend/opencl/runners/tools/OclSharedState.h"
#include "backend/opencl/wrappers/OclLib.h"


xmrig::OclContext::OclContext(const OclDevice &device)
{
    std::vector<cl_device_id> ids = { device.id() };  // 创建包含设备ID的向量
    m_ctx = OclLib::createContext(ids);  // 使用设备ID创建上下文
}


xmrig::OclContext::~OclContext()
{
    if (m_ctx) {  // 如果上下文存在
        OclLib::release(m_ctx);  // 释放上下文
    }
}


bool xmrig::OclContext::init(const std::vector<OclDevice> &devices, std::vector<OclLaunchData> &threads)
{
    if (!m_ctx) {  // 如果上下文不存在
        std::vector<cl_device_id> ids(devices.size());  // 创建包含设备ID的向量
        for (size_t i = 0; i < devices.size(); ++i) {  // 遍历设备
            ids[i] = devices[i].id();  // 将设备ID存入向量
        }

        m_ctx = OclLib::createContext(ids);  // 使用设备ID创建上下文
    }
    # 如果上下文对象为空，则返回 false
    if (!m_ctx) {
        return false;
    }

    # 遍历线程列表，为每个线程设置上下文对象
    for (OclLaunchData &data : threads) {
        data.ctx = m_ctx;
    }

    # 返回 true，表示设置上下文对象成功
    return true;
# 闭合前面的函数定义
```