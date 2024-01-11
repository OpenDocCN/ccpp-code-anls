# `xmrig\src\backend\opencl\wrappers\OclPlatform.h`

```
// 定义了 XMRig 的版权信息
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018-2020 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 防止头文件重复包含
#ifndef XMRIG_OCLPLATFORM_H
#define XMRIG_OCLPLATFORM_H

// 包含必要的头文件
#include <vector>
#include "backend/opencl/wrappers/OclDevice.h"
#include "base/tools/String.h"

// 使用声明
using cl_platform_id = struct _cl_platform_id *;

// 命名空间
namespace xmrig {

// 定义 OclPlatform 类
class OclPlatform
{
public:
    // 默认构造函数
    OclPlatform() = default;
    // 带参数的构造函数
    OclPlatform(size_t index, cl_platform_id id) : m_id(id), m_index(index) {}

    // 静态成员函数，获取 OpenCL 平台列表
    static std::vector<OclPlatform> get();
    // 静态成员函数，打印 OpenCL 平台信息
    static void print();

    // 内联函数，检查平台是否有效
    inline bool isValid() const      { return m_id != nullptr; }
    // 内联函数，返回平台 ID
    inline cl_platform_id id() const { return m_id; }
    // 内联函数，返回平台索引
    inline size_t index() const      { return m_index; }

    // 将平台信息转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 获取平台的设备列表
    std::vector<OclDevice> devices() const;
    // 返回平台的扩展信息
    String extensions() const;
    // 返回设备的名称
    String name() const;
    // 返回设备的配置文件
    String profile() const;
    // 返回设备的供应商信息
    String vendor() const;
    // 返回设备的版本信息
    String version() const;
# 私有成员变量，用于存储平台的ID
cl_platform_id m_id = nullptr;
# 私有成员变量，用于存储平台的索引
size_t m_index      = 0;
# 结束命名空间声明
} // namespace xmrig
# 结束头文件声明
#endif /* XMRIG_OCLPLATFORM_H */
```