# `xmrig\src\hw\dmi\DmiReader.h`

```cpp
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 许可证的版本为 3 或 (在您的选择下) 任何更高版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DMIREADER_H
#define XMRIG_DMIREADER_H


#include "hw/dmi/DmiBoard.h"
#include "hw/dmi/DmiMemory.h"


#include <functional>


namespace xmrig {


class DmiReader
{
public:
    DmiReader() = default;

    // 返回板卡信息
    inline const DmiBoard &board() const                { return m_board; }
    // 返回系统信息
    inline const DmiBoard &system() const               { return m_system; }
    // 返回内存信息
    inline const std::vector<DmiMemory> &memory() const { return m_memory; }
    // 返回大小
    inline uint32_t size() const                        { return m_size; }
    // 返回版本
    inline uint32_t version() const                     { return m_version; }

    // 读取DMI信息
    bool read();

#   ifdef XMRIG_FEATURE_API
    // 转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    void toJSON(rapidjson::Value &out, rapidjson::Document &doc) const;
#   endif

private:
    using Cleanup = std::function<void()>;

    // 解码函数，带清理函数
    bool decode(uint8_t *buf, const Cleanup &cleanup);
    // 解码函数
    bool decode(uint8_t *buf);

    DmiBoard m_board;
    DmiBoard m_system;
    std::vector<DmiMemory> m_memory;
    uint32_t m_size     = 0;
    # 定义一个名为 m_version 的 32 位无符号整数变量，并初始化为 0
    uint32_t m_version  = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 结束了 xmrig 命名空间的声明

#endif /* XMRIG_DMIREADER_H */
// 结束了 XMRIG_DMIREADER_H 文件的条件编译
```