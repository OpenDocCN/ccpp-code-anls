# `xmrig\src\hw\msr\Msr_linux.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "hw/msr/Msr.h"  // 导入 Msr 类的头文件
#include "3rdparty/fmt/core.h"  // 导入 fmt 库的核心头文件
#include "backend/cpu/Cpu.h"  // 导入 Cpu 类的头文件
#include "base/io/log/Log.h"  // 导入 Log 类的头文件


#include <array>  // 导入数组类的头文件
#include <cctype>  // 导入字符处理函数的头文件
#include <cinttypes>  // 导入整数格式转换的头文件
#include <cstdio>  // 导入 C 标准输入输出函数的头文件
#include <dirent.h>  // 导入目录操作函数的头文件
#include <fcntl.h>  // 导入文件控制函数的头文件
#include <fstream>  // 导入文件流类的头文件
#include <sys/stat.h>  // 导入文件状态函数的头文件
#include <sys/types.h>  // 导入系统数据类型的头文件
#include <unistd.h>  // 导入 POSIX 标准函数的头文件


namespace xmrig {


static int msr_open(int32_t cpu, int flags)
{
    const auto name = fmt::format("/dev/cpu/{}/msr", cpu < 0 ? Cpu::info()->units().front() : cpu);  // 根据 CPU 编号生成 MSR 文件名

    return open(name.c_str(), flags);  // 打开 MSR 文件
}


class MsrPrivate
{
public:
    inline MsrPrivate() : m_available(msr_allow_writes() || msr_modprobe()) {}  // 构造函数，初始化 m_available 成员变量

    inline bool isAvailable() const { return m_available; }  // 返回 m_available 成员变量的值

private:
    inline bool msr_allow_writes()  // 允许写入 MSR 寄存器
    {
        std::ofstream file("/sys/module/msr/parameters/allow_writes", std::ios::out | std::ios::binary | std::ios::trunc);  // 打开 allow_writes 文件
        if (file.is_open()) {  // 如果文件成功打开
            file << "on";  // 写入 "on"
        }

        return file.good();  // 返回文件状态
    }

    inline bool msr_modprobe()  // 装载 MSR 模块
    {
        return system("/sbin/modprobe msr allow_writes=on > /dev/null 2>&1") == 0;  // 执行 modprobe 命令
    }

    const bool m_available;  // 可用性标志
};


} // namespace xmrig
# 创建 Msr 类的构造函数
xmrig::Msr::Msr() : d_ptr(new MsrPrivate())
{
    # 如果 MSR 内核模块不可用，则输出警告信息
    if (!isAvailable()) {
        LOG_WARN("%s " YELLOW_BOLD("msr kernel module is not available"), tag());
    }
}

# 创建 Msr 类的析构函数
xmrig::Msr::~Msr()
{
    # 删除私有成员指针
    delete d_ptr;
}

# 判断 MSR 是否可用的函数
bool xmrig::Msr::isAvailable() const
{
    # 调用私有成员的 isAvailable 函数判断 MSR 是否可用
    return d_ptr->isAvailable();
}

# 写入 MSR 寄存器的函数
bool xmrig::Msr::write(Callback &&callback)
{
    # 获取 CPU 信息中的单元列表
    const auto &units = Cpu::info()->units();

    # 遍历单元列表
    for (int32_t pu : units) {
        # 如果回调函数返回 false，则返回 false
        if (!callback(pu)) {
            return false;
        }
    }

    # 返回 true
    return true;
}

# 读取 MSR 寄存器的函数
bool xmrig::Msr::rdmsr(uint32_t reg, int32_t cpu, uint64_t &value) const
{
    # 打开 MSR 设备文件
    const int fd = msr_open(cpu, O_RDONLY);

    # 如果打开失败，则返回 false
    if (fd < 0) {
        return false;
    }

    # 从设备文件中读取指定寄存器的值，并将其存储到 value 变量中
    const bool success = pread(fd, &value, sizeof value, reg) == sizeof value;

    # 关闭设备文件
    close(fd);

    # 返回读取是否成功
    return success;
}

# 写入 MSR 寄存器的函数
bool xmrig::Msr::wrmsr(uint32_t reg, uint64_t value, int32_t cpu)
{
    # 打开 MSR 设备文件
    const int fd = msr_open(cpu, O_WRONLY);

    # 如果打开失败，则返回 false
    if (fd < 0) {
        return false;
    }

    # 将指定寄存器的值写入设备文件
    const bool success = pwrite(fd, &value, sizeof value, reg) == sizeof value;

    # 关闭设备文件
    close(fd);

    # 返回写入是否成功
    return success;
}
```