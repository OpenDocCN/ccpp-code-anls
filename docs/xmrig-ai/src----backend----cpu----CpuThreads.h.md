# `xmrig\src\backend\cpu\CpuThreads.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本3，或者
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */
#ifndef XMRIG_CPUTHREADS_H
#define XMRIG_CPUTHREADS_H

#include <vector>
#include "backend/cpu/CpuThread.h"

namespace xmrig {

class CpuThreads
{
public:
    inline CpuThreads() = default;  // 默认构造函数
    inline CpuThreads(size_t count) : m_data(count) {}  // 带有计数参数的构造函数

    CpuThreads(const rapidjson::Value &value);  // 从rapidjson::Value对象构造CpuThreads对象
    CpuThreads(size_t count, uint32_t intensity);  // 带有计数和强度参数的构造函数

    inline bool isEmpty() const                             { return m_data.empty(); }  // 判断是否为空
    inline const std::vector<CpuThread> &data() const       { return m_data; }  // 返回数据
    inline size_t count() const                             { return m_data.size(); }  // 返回数据数量
    inline void add(const CpuThread &thread)                { m_data.push_back(thread); }  // 添加CpuThread对象
    inline void add(int64_t affinity, uint32_t intensity)   { add(CpuThread(affinity, intensity)); }  // 添加亲和力和强度参数的CpuThread对象
    inline void reserve(size_t capacity)                    { m_data.reserve(capacity); }  // 预留容量

    inline bool operator!=(const CpuThreads &other) const   { return !isEqual(other); }  // 不等于运算符重载
    inline bool operator==(const CpuThreads &other) const   { return isEqual(other); }  // 等于运算符重载

    bool isEqual(const CpuThreads &other) const;  // 判断是否相等
    // 将 rapidjson::Document 转换为 rapidjson::Value 类型并返回
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
# 声明私有成员变量和枚举类型 Format
private:
    enum Format {
        ArrayFormat,  # 数组格式
        ObjectFormat   # 对象格式
    };

    Format m_format     = ArrayFormat;  # 初始化 m_format 为 ArrayFormat
    int64_t m_affinity  = -1;  # 初始化 m_affinity 为 -1
    std::vector<CpuThread> m_data;  # 声明一个存储 CpuThread 对象的向量 m_data
};


} /* namespace xmrig */  # 结束命名空间 xmrig

#endif /* XMRIG_CPUTHREADS_H */  # 结束头文件定义
```