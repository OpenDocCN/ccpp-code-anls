# `xmrig\src\base\net\tools\Storage.h`

```cpp
/* XMRig
 * Copyright 2018-2023 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
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
#ifndef XMRIG_STORAGE_H
#define XMRIG_STORAGE_H

// 包含断言和映射容器的头文件
#include <cassert>
#include <map>

// 命名空间定义
namespace xmrig {

// 模板类定义
template <class TYPE>
class Storage
{
public:
    // 默认构造函数
    inline Storage() = default;

    // 添加指针到数据映射中
    inline uintptr_t add(TYPE *ptr)
    {
        m_data[m_counter] = ptr;

        return m_counter++;
    }

    // 根据ID获取指针
    inline TYPE *ptr(uintptr_t id)          { return reinterpret_cast<TYPE *>(id); }

    // 根据ID获取指针
    inline TYPE *get(const void *id) const  { return get(reinterpret_cast<uintptr_t>(id)); }
    inline TYPE *get(uintptr_t id) const
    {
        // 断言ID存在于映射中
        assert(m_data.count(id) > 0);
        if (m_data.count(id) == 0) {
            return nullptr;
        }

        return m_data.at(id);
    }

    // 判断映射是否为空
    inline bool isEmpty() const             { return m_data.empty(); }
    // 获取映射大小
    inline size_t size() const              { return m_data.size(); }

    // 根据ID删除指针
    inline void remove(const void *id)      { delete release(reinterpret_cast<uintptr_t>(id)); }
    inline void remove(uintptr_t id)        { delete release(id); }

    // 根据ID释放指针
    inline TYPE *release(const void *id)    { return release(reinterpret_cast<uintptr_t>(id)); }
    inline TYPE *release(uintptr_t id)
    # 通过id获取对象
    auto obj = get(id);
    # 如果对象不为空
    if (obj != nullptr) {
        # 从数据中删除该对象
        m_data.erase(id);
    }
    # 返回获取到的对象
    return obj;
# 声明私有成员变量，使用 std::map 存储键为 uintptr_t 类型，值为 TYPE* 类型的数据
private:
    std::map<uintptr_t, TYPE *> m_data;
    # 声明私有成员变量，初始化为 0，用于计数
    uintptr_t m_counter  = 0;
};


# 结束命名空间 xmrig
} /* namespace xmrig */


# 结束条件编译指令，关闭 XMRIG_STORAGE_H 文件的定义
#endif /* XMRIG_STORAGE_H */
```