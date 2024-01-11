# `xmrig\src\backend\opencl\wrappers\AdlLib_linux.cpp`

```
/*
 * XMRig
 * 版权所有 (c) 2008-2018 高级微处理器公司
 * 版权所有 (c) 2020      Patrick Bollinger            <https://github.com/pjbollinger>
 * 版权所有 (c) 2018-2021 SChernykh                    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig                        <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，无论是许可证的第3版
 * （在您的选择）任何后续版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/wrappers/AdlLib.h"
#include "3rdparty/fmt/core.h"
#include "backend/opencl/wrappers/OclDevice.h"


#include <limits.h>
#include <dirent.h>
#include <fstream>
#include <map>
#include <string>
#include <sys/stat.h>
#include <sys/types.h>


namespace xmrig {


// 初始化静态成员变量
bool AdlLib::m_initialized          = false;
bool AdlLib::m_ready                = false;
static const std::string kPrefix    = "/sys/bus/pci/drivers/amdgpu/";
static std::map<PciTopology, std::string> hwmon_cache;


// 检查文件是否存在
static inline bool sysfs_is_file(const char *path)
{
    struct stat sb;

    return stat(path, &sb) == 0 && ((sb.st_mode & S_IFMT) == S_IFREG);
}


// 过滤目录中的文件
static inline int dir_filter(const struct dirent *dirp)
{
    return strlen(dirp->d_name) > 5 ? 1 : 0;
}


// 检查是否为amdgpu文件
static bool sysfs_is_amdgpu(const char *path, char *buf, const char *filename)
{
    strcpy(buf, filename);

    if (!sysfs_is_file(path)) {
        return false;
    }

    std::ifstream file(path);
    # 如果文件未成功打开，则返回 false
    if (!file.is_open()) {
        return false;
    }

    # 声明一个字符串变量 name
    std::string name;
    # 从文件中读取一行内容到 name 变量中
    std::getline(file, name);

    # 返回 name 变量是否等于 "amdgpu" 的布尔值
    return name == "amdgpu";
} // 结束命名空间 xmrig

// 从指定路径读取文件内容到缓冲区，返回读取的字节数
static uint32_t sysfs_read(const char *path, char *buf, const char *filename)
{
    // 将文件名复制到缓冲区
    strcpy(buf, filename);

    // 打开指定路径的文件
    std::ifstream file(path);
    // 如果文件未成功打开，则返回 0
    if (!file.is_open()) {
        return 0;
    }

    // 从文件中读取一个 32 位无符号整数
    uint32_t value = 0;
    file >> value;

    return value;
}

// 在路径中添加前缀，返回添加前缀后的路径长度
static size_t sysfs_prefix(char path[PATH_MAX], const PciTopology &topology)
{
    // 在缓存中查找拓扑信息
    const auto it = hwmon_cache.find(topology);
    // 如果找到了缓存中的拓扑信息
    if (it != hwmon_cache.end()) {
        // 将缓存中的路径复制到指定路径中
        strcpy(path, it->second.data());
        // 返回缓存中路径的长度
        return it->second.size();
    }

    // 格式化路径，添加前缀
    char *base = fmt::format_to(path, "{}0000:{}/hwmon/", kPrefix, topology.toString());
    *base      = '\0';
    char *end  = nullptr;

    // 获取目录中的文件列表
    struct dirent **namelist;
    int n = scandir(path, &namelist, dir_filter, nullptr);
    // 如果获取文件列表失败，则返回空值
    if (n < 0) {
        return {};
    }

    // 遍历文件列表
    while (n--) {
        // 如果 end 为空
        if (!end) {
            // 格式化临时路径
            char *tmp = fmt::format_to(base, "{}/", namelist[n]->d_name);
            // 判断是否是 amdgpu 设备，并且读取温度或功耗数据
            end       = (sysfs_is_amdgpu(path, tmp, "name") && (sysfs_read(path, tmp, "temp1_input") || sysfs_read(path, tmp, "power1_average"))) ? tmp : nullptr;
        }

        // 释放文件列表中的每个元素
        free(namelist[n]);
    }

    // 释放文件列表
    free(namelist);

    // 如果 end 不为空
    if (end) {
        // 将 end 位置设为字符串结束，并将路径信息插入缓存
        *end = '\0';
        hwmon_cache.insert({ topology, path });

        // 返回添加前缀后的路径长度
        return end - path;
    }

    return 0;
}

// 初始化 AdlLib 对象
bool xmrig::AdlLib::init()
{
    // 如果未初始化
    if (!m_initialized) {
        // 动态加载库并加载函数
        m_ready       = dlopen() && load();
        m_initialized = true;
    }

    return m_ready;
}

// 返回最后的错误信息
const char *xmrig::AdlLib::lastError() noexcept
{
    return nullptr;
}

// 关闭 AdlLib 对象
void xmrig::AdlLib::close()
{
}

// 获取 AMD GPU 设备的健康信息
AdlHealth xmrig::AdlLib::health(const OclDevice &device)
{
    // 如果未准备好或设备不是 AMD 厂商的
    if (!isReady() || device.vendorId() != OCL_VENDOR_AMD) {
        return {};
    }

    // 静态字符数组，用于存储路径信息
    static char path[PATH_MAX]{};

    // 在路径中添加前缀，并将结果赋给 buf
    char *buf = path + sysfs_prefix(path, device.topology());
    // 如果 buf 等于 path，则返回空值
    if (buf == path) {
        return {};
    }

    AdlHealth health;
    # 设置健康数据中的时钟频率，将读取的数值除以1000000
    health.clock        = sysfs_read(path, buf, "freq1_input") / 1000000;
    # 设置健康数据中的内存时钟频率，将读取的数值除以1000000
    health.memClock     = sysfs_read(path, buf, "freq2_input") / 1000000;
    # 设置健康数据中的功率，将读取的数值除以1000000
    health.power        = sysfs_read(path, buf, "power1_average") / 1000000;
    # 设置健康数据中的转速，直接读取数值
    health.rpm          = sysfs_read(path, buf, "fan1_input");
    # 设置健康数据中的温度，将读取的数值除以1000
    health.temperature  = sysfs_read(path, buf, "temp2_input") / 1000;

    # 如果温度为0，则尝试读取另一个温度传感器的数值并除以1000
    if (!health.temperature) {
        health.temperature = sysfs_read(path, buf, "temp1_input") / 1000;
    }

    # 返回健康数据
    return health;
# 结束 xmrig::AdlLib 类的定义
}

# 检查指定路径的文件状态，如果不存在则返回 false
bool xmrig::AdlLib::dlopen()
{
    // 定义用于存储文件状态信息的结构体
    struct stat sb;
    // 使用 stat 函数获取指定路径的文件状态信息，并将结果存储在 sb 中
    if (stat(kPrefix.c_str(), &sb) == -1) {
        // 如果获取失败，则返回 false
        return false;
    }
    // 检查文件状态的 st_mode 属性，判断是否为目录
    return (sb.st_mode & S_IFMT) == S_IFDIR;
}

# 加载 AdlLib 类
bool xmrig::AdlLib::load()
{
    // 直接返回 true，表示加载成功
    return true;
}
```