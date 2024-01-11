# `xmrig\src\crypto\common\LinuxMemory.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "crypto/common/LinuxMemory.h"  // 导入LinuxMemory.h文件
#include "3rdparty/fmt/core.h"           // 导入fmt/core.h文件
#include "crypto/common/VirtualMemory.h" // 导入VirtualMemory.h文件

#include <algorithm>                    // 导入algorithm标准库
#include <fstream>                      // 导入fstream标准库
#include <mutex>                        // 导入mutex标准库
#include <string>                       // 导入string标准库

namespace xmrig {

static std::mutex mutex;               // 创建静态互斥锁对象mutex
constexpr size_t twoMiB = 2U * 1024U * 1024U;  // 创建常量twoMiB并赋值
constexpr size_t oneGiB = 1024U * 1024U * 1024U;  // 创建常量oneGiB并赋值

static inline std::string sysfs_path(uint32_t node, size_t hugePageSize, bool nr) {  // 创建内联函数sysfs_path
    return fmt::format("/sys/devices/system/node/node{}/hugepages/hugepages-{}kB/{}_hugepages", node, hugePageSize / 1024, nr ? "nr" : "free");  // 返回格式化后的字符串
}

static inline bool write_nr_hugepages(uint32_t node, size_t hugePageSize, uint64_t count) {  // 创建内联函数write_nr_hugepages
    return LinuxMemory::write(sysfs_path(node, hugePageSize, true).c_str(), count);  // 调用LinuxMemory的write方法
}
static inline int64_t free_hugepages(uint32_t node, size_t hugePageSize) {  // 创建内联函数free_hugepages
    return LinuxMemory::read(sysfs_path(node, hugePageSize, false).c_str());  // 调用LinuxMemory的read方法
}
static inline int64_t nr_hugepages(uint32_t node, size_t hugePageSize) {  // 创建内联函数nr_hugepages
    return LinuxMemory::read(sysfs_path(node, hugePageSize, true).c_str());  // 调用LinuxMemory的read方法
}

} // namespace xmrig
# 在 LinuxMemory 类中定义了一个名为 reserve 的方法，用于分配内存
bool xmrig::LinuxMemory::reserve(size_t size, uint32_t node, size_t hugePageSize)
{
    # 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(mutex);

    # 计算所需内存大小，并按照 hugePageSize 对齐
    const size_t required = VirtualMemory::align(size, hugePageSize) / hugePageSize;

    # 获取指定节点上可用的巨大页数量
    const auto available = free_hugepages(node, hugePageSize);
    # 如果可用数量小于 0 或者大于等于所需数量，则返回 false
    if (available < 0 || static_cast<size_t>(available) >= required) {
        return false;
    }

    # 写入新的巨大页数量，并返回 true
    return write_nr_hugepages(node, hugePageSize, std::max<size_t>(nr_hugepages(node, hugePageSize), 0) + (required - available));
}

# 在 LinuxMemory 类中定义了一个名为 write 的方法，用于向文件中写入数据
bool xmrig::LinuxMemory::write(const char *path, uint64_t value)
{
    # 以输出、二进制、截断的方式打开文件
    std::ofstream file(path, std::ios::out | std::ios::binary | std::ios::trunc);
    # 如果文件打开失败，则返回 false
    if (!file.is_open()) {
        return false;
    }

    # 向文件中写入数据并刷新缓冲区
    file << value;
    file.flush();

    # 返回 true
    return true;
}

# 在 LinuxMemory 类中定义了一个名为 read 的方法，用于从文件中读取数据
int64_t xmrig::LinuxMemory::read(const char *path)
{
    # 以默认方式打开文件
    std::ifstream file(path);
    # 如果文件打开失败，则返回 -1
    if (!file.is_open()) {
        return -1;
    }

    # 从文件中读取数据到 value 变量中
    uint64_t value = 0;
    file >> value;

    # 返回读取到的数据
    return value;
}
```