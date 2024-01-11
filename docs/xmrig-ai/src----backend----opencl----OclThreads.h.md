# `xmrig\src\backend\opencl\OclThreads.h`

```
// 包含版权声明和许可信息
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
#ifndef XMRIG_OCLTHREADS_H
#define XMRIG_OCLTHREADS_H

// 包含必要的头文件
#include <vector>
#include "backend/opencl/OclThread.h"
#include "backend/opencl/wrappers/OclDevice.h"

// 命名空间定义
namespace xmrig {

// 类定义
class OclThreads
{
public:
    // 默认构造函数
    OclThreads() = default;
    // 从 JSON 值构造函数
    OclThreads(const rapidjson::Value &value);
    // 从 OpenCL 设备和算法构造函数
    OclThreads(const std::vector<OclDevice> &devices, const Algorithm &algorithm);

    // 判断是否为空
    inline bool isEmpty() const                             { return m_data.empty(); }
    // 返回数据
    inline const std::vector<OclThread> &data() const       { return m_data; }
    // 返回数据数量
    inline size_t count() const                             { return m_data.size(); }
    // 添加线程
    inline void add(OclThread &&thread)                     { m_data.push_back(thread); }
    // 为数据预留一定的容量
    inline void reserve(size_t capacity)                    { m_data.reserve(capacity); }

    // 重载不等于运算符，判断当前对象是否不等于另一个对象
    inline bool operator!=(const OclThreads &other) const   { return !isEqual(other); }
    // 重载等于运算符，判断当前对象是否等于另一个对象
    inline bool operator==(const OclThreads &other) const   { return isEqual(other); }

    // 判断当前对象是否等于另一个对象
    bool isEqual(const OclThreads &other) const;
    // 将当前对象转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
# 声明私有成员变量，用于存储 OclThread 对象的向量
private:
    std::vector<OclThread> m_data;
};  # 结束类定义

} /* namespace xmrig */  # 结束命名空间定义

#endif /* XMRIG_OCLTHREADS_H */  # 结束条件编译指令
```