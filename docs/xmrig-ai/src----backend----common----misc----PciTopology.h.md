# `xmrig\src\backend\common\misc\PciTopology.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_PCITOPOLOGY_H
#define XMRIG_PCITOPOLOGY_H


#include <cstdio>


#include "base/tools/String.h"


namespace xmrig {


class PciTopology
{
public:
    PciTopology() = default;
    PciTopology(uint32_t bus, uint32_t device, uint32_t function) : m_valid(true), m_bus(bus), m_device(device), m_function(function) {}

    inline bool isEqual(const PciTopology &other) const     { return m_valid == other.m_valid && toUint32() == other.toUint32(); }
    inline bool isValid() const                             { return m_valid; }
    inline uint8_t bus() const                              { return m_bus; }
    inline uint8_t device() const                           { return m_device; }
    inline uint8_t function() const                         { return m_function; }

    inline bool operator!=(const PciTopology &other) const  { return !isEqual(other); }
    inline bool operator<(const PciTopology &other) const   { return toUint32() < other.toUint32(); }
    inline bool operator==(const PciTopology &other) const  { return isEqual(other); }

    String toString() const
    # 如果不是有效的设备，返回 "n/a"
    if (!isValid()) {
        return "n/a";
    }

    # 创建一个长度为8的字符数组，并初始化为0
    char *buf = new char[8]();
    
    # 将总线、设备和功能的值格式化成字符串，存储到buf中
    snprintf(buf, 8, "%02hhx:%02hhx.%01hhx", bus(), device(), function());

    # 返回格式化后的字符串
    return buf;
// 私有成员变量和方法声明
private:
    // 将成员变量 m_bus、m_device、m_function 合并为一个 32 位的无符号整数
    inline uint32_t toUint32() const { return m_bus << 16 | m_device << 8 | m_function;  }

    // 标识是否有效的标志位，默认为 false
    bool m_valid         = false;
    // 总线号，默认为 0
    uint8_t m_bus        = 0;
    // 设备号，默认为 0
    uint8_t m_device     = 0;
    // 功能号，默认为 0
    uint8_t m_function   = 0;
};


// 命名空间结束
} // namespace xmrig

// 头文件结束
#endif /* XMRIG_PCITOPOLOGY_H */
```