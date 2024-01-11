# `xmrig\src\crypto\common\LinuxMemory.h`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 它，无论是许可证的第3版还是
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_LINUXMEMORY_H
#define XMRIG_LINUXMEMORY_H

#include <cstdint>
#include <cstddef>

namespace xmrig {

class LinuxMemory
{
public:
    // 静态方法，用于在 Linux 上保留内存
    static bool reserve(size_t size, uint32_t node, size_t hugePageSize);

    // 静态方法，用于写入值到指定路径
    static bool write(const char *path, uint64_t value);
    
    // 静态方法，用于从指定路径读取值
    static int64_t read(const char *path);
};

} /* namespace xmrig */

#endif /* XMRIG_LINUXMEMORY_H */
```