# `xmrig\src\backend\cpu\CpuThread.h`

```
/* XMRig
 * 版权声明
 */

#ifndef XMRIG_CPUTHREAD_H
#define XMRIG_CPUTHREAD_H

#include "3rdparty/rapidjson/fwd.h"
// 引入 rapidjson 库的前置声明

namespace xmrig {
// 命名空间 xmrig

class CpuThread
// 定义类 CpuThread
{
public:
    inline constexpr CpuThread() = default;
    // 默认构造函数
    inline constexpr CpuThread(int64_t affinity, uint32_t intensity) : m_affinity(affinity), m_intensity(intensity) {}
    // 构造函数，接受亲和性和强度参数

    CpuThread(const rapidjson::Value &value);
    // 构造函数，接受 rapidjson::Value 对象作为参数

    inline bool isEqual(const CpuThread &other) const       { return other.m_affinity == m_affinity && other.m_intensity == m_intensity; }
    // 判断两个 CpuThread 对象是否相等的函数
    inline bool isValid() const                             { return m_intensity <= 8; }
    // 判断 CpuThread 对象是否有效的函数
    inline int64_t affinity() const                         { return m_affinity; }
    // 返回亲和性的函数
    # 返回当前线程的强度值，如果强度为0则返回1
    inline uint32_t intensity() const                       { return m_intensity == 0 ? 1 : m_intensity; }

    # 重载不等于运算符，判断当前线程是否不等于另一个线程
    inline bool operator!=(const CpuThread &other) const    { return !isEqual(other); }
    # 重载等于运算符，判断当前线程是否等于另一个线程
    inline bool operator==(const CpuThread &other) const    { return isEqual(other); }

    # 将当前线程转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
# 私有成员变量，表示线程的亲和性，初始值为-1
int64_t m_affinity   = -1;
# 私有成员变量，表示线程的强度，初始值为0
uint32_t m_intensity = 0;
# 结束私有命名空间
} /* namespace xmrig */
# 结束头文件的定义
#endif /* XMRIG_CPUTHREAD_H */
```