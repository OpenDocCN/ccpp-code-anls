# `xmrig\src\hw\msr\Msr_win.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 其中包括许可证的第3版或（根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的分发
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "hw/msr/Msr.h"  // 导入 Msr 类的头文件
#include "backend/cpu/Cpu.h"  // 导入 Cpu 类的头文件
#include "base/io/log/Log.h"  // 导入 Log 类的头文件
#include "base/kernel/Platform.h"  // 导入 Platform 类的头文件

#include <string>  // 导入字符串类
#include <thread>  // 导入线程类
#include <vector>  // 导入向量类
#include <windows.h>  // 导入 Windows 头文件

#define SERVICE_NAME    L"WinRing0_1_2_0"  // 定义服务名称为 WinRing0_1_2_0
#define IOCTL_READ_MSR  CTL_CODE(40000, 0x821, METHOD_BUFFERED, FILE_ANY_ACCESS)  // 定义读取 MSR 的控制码
#define IOCTL_WRITE_MSR CTL_CODE(40000, 0x822, METHOD_BUFFERED, FILE_ANY_ACCESS)  // 定义写入 MSR 的控制码

namespace xmrig {  // 命名空间 xmrig

static const wchar_t *kServiceName = SERVICE_NAME;  // 定义常量 kServiceName 为 SERVICE_NAME

class MsrPrivate  // MsrPrivate 类
{
public:  // 公共访问权限

    bool uninstall()  // 卸载函数
    {
        // 如果 driver 不等于 INVALID_HANDLE_VALUE，则关闭 driver 句柄
        if (driver != INVALID_HANDLE_VALUE) {
            CloseHandle(driver);
        }

        // 如果 service 为空，则返回 true
        if (!service) {
            return true;
        }

        // 初始化 result 为 true
        bool result = true;

        // 如果不重用服务，则执行以下操作
        if (!reuse) {
            // 定义 SERVICE_STATUS 结构体
            SERVICE_STATUS serviceStatus;

            // 控制服务停止，并将结果保存在 serviceStatus 中
            if (!ControlService(service, SERVICE_CONTROL_STOP, &serviceStatus)) {
                result = false;
            }

            // 删除服务，如果失败则记录错误信息并将 result 设为 false
            if (!DeleteService(service)) {
                LOG_ERR("%s " RED("failed to remove WinRing0 driver, error %u"), Msr::tag(), GetLastError());
                result = false;
            }
        }

        // 关闭服务句柄
        CloseServiceHandle(service);
        // 将 service 指针置为空
        service = nullptr;

        // 返回 result
        return result;
    }


    // 初始化 reuse 为 false
    bool reuse          = false;
    // 初始化 driver 为 INVALID_HANDLE_VALUE
    HANDLE driver       = INVALID_HANDLE_VALUE;
    // 初始化 manager 为 nullptr
    SC_HANDLE manager   = nullptr;
    // 初始化 service 为 nullptr
    SC_HANDLE service   = nullptr;
// 命名空间 xmrig
namespace xmrig {

// Msr 类的构造函数
xmrig::Msr::Msr() : d_ptr(new MsrPrivate())
{
    DWORD err = 0;

    // 打开服务控制管理器，获取管理器句柄
    d_ptr->manager = OpenSCManager(nullptr, nullptr, SC_MANAGER_ALL_ACCESS);
    if (!d_ptr->manager) {
        // 如果打开服务控制管理器失败，则判断错误码
        if ((err = GetLastError()) == ERROR_ACCESS_DENIED) {
            // 如果错误码为 ERROR_ACCESS_DENIED，则输出警告信息，要求管理员权限
            LOG_WARN("%s " YELLOW_BOLD("to access MSR registers Administrator privileges required."), tag());
        }
        else {
            // 如果错误码不是 ERROR_ACCESS_DENIED，则输出错误信息
            LOG_ERR("%s " RED("failed to open service control manager, error %u"), tag(), err);
        }

        return;
    }

    // 定义一个 wchar_t 类型的动态数组 dir
    std::vector<wchar_t> dir;

    // 循环获取当前模块的文件路径，直到获取成功或者出错
    do {
        // 如果 dir 为空，则设置 dir 的大小为 MAX_PATH，否则将 dir 的大小扩大两倍
        dir.resize(dir.empty() ? MAX_PATH : dir.size() * 2);
        // 获取当前模块的文件路径
        GetModuleFileNameW(nullptr, dir.data(), dir.size());
        // 获取错误码
        err = GetLastError();
    } while (err == ERROR_INSUFFICIENT_BUFFER);

    // 如果获取文件路径出错，则输出错误信息
    if (err != ERROR_SUCCESS) {
        LOG_ERR("%s " RED("failed to get path to driver, error %u"), tag(), err);
        return;
    }

    // 从文件路径中截取出文件所在目录的路径
    for (auto it = dir.end() - 1; it != dir.begin(); --it) {
        if ((*it == L'\\') || (*it == L'/')) {
            ++it;
            *it = L'\0';
            break;
        }
    }

    // 构造驱动文件的完整路径
    const std::wstring path = std::wstring(dir.data()) + L"WinRing0x64.sys";

    // 打开服务，获取服务句柄
    d_ptr->service = OpenServiceW(d_ptr->manager, kServiceName, SERVICE_ALL_ACCESS);
    # 如果服务已经存在
    if (d_ptr->service) {
        # 输出警告信息，指示服务已经存在
        LOG_WARN("%s " YELLOW("service ") YELLOW_BOLD("WinRing0_1_2_0") YELLOW(" already exists"), tag());

        # 查询服务的状态
        SERVICE_STATUS status;
        const auto rc = QueryServiceStatus(d_ptr->service, &status);

        # 如果查询成功
        if (rc) {
            # 定义变量，用于存储查询服务配置所需的缓冲区大小
            DWORD dwBytesNeeded = 0;

            # 查询服务配置，获取所需的缓冲区大小
            QueryServiceConfigA(d_ptr->service, nullptr, 0, &dwBytesNeeded);

            # 如果缓冲区大小不足
            if (GetLastError() == ERROR_INSUFFICIENT_BUFFER) {
                # 创建一个足够大的缓冲区
                std::vector<BYTE> buffer(dwBytesNeeded);
                auto config = reinterpret_cast<LPQUERY_SERVICE_CONFIGA>(buffer.data());

                # 查询服务配置，获取服务的二进制路径名
                if (QueryServiceConfigA(d_ptr->service, config, buffer.size(), &dwBytesNeeded)) {
                    # 输出信息，指示服务的二进制路径名
                    LOG_INFO("%s " YELLOW("service path: ") YELLOW_BOLD("\"%s\""), tag(), config->lpBinaryPathName);
                }
            }
        }

        # 如果查询成功且服务状态为运行中
        if (rc && status.dwCurrentState == SERVICE_RUNNING) {
            # 设置重用标志为真
            d_ptr->reuse = true;
        }
        # 如果服务存在但无法卸载
        else if (!d_ptr->uninstall()) {
            # 返回
            return;
        }
    }

    # 创建一个文件句柄，用于打开驱动程序
    d_ptr->driver = CreateFileW(L"\\\\.\\" SERVICE_NAME, GENERIC_READ | GENERIC_WRITE, 0, nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
    # 如果文件句柄有效，说明服务已经存在但服务名称不同
    if (d_ptr->driver != INVALID_HANDLE_VALUE) {
        # 输出警告信息，指示服务已经存在但服务名称不同
        LOG_WARN("%s " YELLOW("service ") YELLOW_BOLD("WinRing0_1_2_0") YELLOW(" already exists, but with a different service name"), tag());
        # 设置重用标志为真
        d_ptr->reuse = true;
        # 返回
        return;
    }
    # 如果不是重用模式
    if (!d_ptr->reuse) {
        # 创建服务对象，并指定服务的名称、显示名称、访问权限、服务类型、启动类型、错误控制、驱动文件路径等参数
        d_ptr->service = CreateServiceW(d_ptr->manager, kServiceName, kServiceName, SERVICE_ALL_ACCESS, SERVICE_KERNEL_DRIVER, SERVICE_DEMAND_START, SERVICE_ERROR_NORMAL, path.c_str(), nullptr, nullptr, nullptr, nullptr, nullptr);
        # 如果创建服务失败
        if (!d_ptr->service) {
            # 输出错误信息
            LOG_ERR("%s " RED("failed to install WinRing0 driver, error %u"), tag(), GetLastError());
            # 返回

            return;
        }
        # 如果服务未启动
        if (!StartService(d_ptr->service, 0, nullptr)) {
            # 获取错误码
            err = GetLastError();
            # 如果错误码不是服务已经在运行
            if (err != ERROR_SERVICE_ALREADY_RUNNING) {
                # 如果错误码是文件未找到
                if (err == ERROR_FILE_NOT_FOUND) {
                    # 输出错误信息
                    LOG_ERR("%s " RED("failed to start WinRing0 driver: ") RED_BOLD("\"WinRing0x64.sys not found\""), tag());
                }
                # 否则
                else {
                    # 输出错误信息
                    LOG_ERR("%s " RED("failed to start WinRing0 driver, error %u"), tag(), err);
                }
                # 卸载服务
                d_ptr->uninstall();
                # 返回

                return;
            }
        }
    }
    # 创建文件对象，并指定文件路径、访问权限等参数
    d_ptr->driver = CreateFileW(L"\\\\.\\" SERVICE_NAME, GENERIC_READ | GENERIC_WRITE, 0, nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
    # 如果创建文件对象失败
    if (d_ptr->driver == INVALID_HANDLE_VALUE) {
        # 输出错误信息
        LOG_ERR("%s " RED("failed to connect to WinRing0 driver, error %u"), tag(), GetLastError());;
    }
# Msr 类的析构函数
xmrig::Msr::~Msr()
{
    # 卸载驱动
    d_ptr->uninstall();

    # 删除 d_ptr 对象
    delete d_ptr;
}


# 判断 Msr 对象是否可用
bool xmrig::Msr::isAvailable() const
{
    # 判断驱动是否为无效句柄
    return d_ptr->driver != INVALID_HANDLE_VALUE;
}


# 写入 MSR 寄存器
bool xmrig::Msr::write(Callback &&callback)
{
    # 获取 CPU 信息中的单元列表
    const auto &units = Cpu::info()->units();
    # 初始化 success 变量为 false
    bool success      = false;

    # 创建一个线程，用于执行回调函数
    std::thread thread([&callback, &units, &success]() {
        # 遍历单元列表
        for (int32_t pu : units) {
            # 设置线程亲和性
            if (!Platform::setThreadAffinity(pu)) {
                continue;
            }

            # 执行回调函数
            if (!callback(pu)) {
                return;
            }
        }

        # 设置 success 变量为 true
        success = true;
    });

    # 等待线程执行完毕
    thread.join();

    # 返回 success 变量的值
    return success;
}


# 读取 MSR 寄存器
bool xmrig::Msr::rdmsr(uint32_t reg, int32_t cpu, uint64_t &value) const
{
    # 断言 cpu 值小于 0
    assert(cpu < 0);

    # 初始化 size 变量为 0
    DWORD size = 0;

    # 调用 DeviceIoControl 函数读取 MSR 寄存器的值
    return DeviceIoControl(d_ptr->driver, IOCTL_READ_MSR, &reg, sizeof(reg), &value, sizeof(value), &size, nullptr) != 0;
}


# 写入 MSR 寄存器
bool xmrig::Msr::wrmsr(uint32_t reg, uint64_t value, int32_t cpu)
{
    # 断言 cpu 值小于 0
    assert(cpu < 0);

    # 定义 input 结构体，用于传递给驱动的输入参数
    struct {
        uint32_t reg = 0;
        uint32_t value[2]{};
    } input;

    # 静态断言，确保 input 结构体的大小为 12 字节
    static_assert(sizeof(input) == 12, "Invalid struct size for WinRing0 driver");

    # 设置 input 结构体的 reg 和 value 字段的值
    input.reg = reg;
    *(reinterpret_cast<uint64_t*>(input.value)) = value;

    # 定义 output 和 k 变量
    DWORD output;
    DWORD k;

    # 调用 DeviceIoControl 函数写入 MSR 寄存器的值
    return DeviceIoControl(d_ptr->driver, IOCTL_WRITE_MSR, &input, sizeof(input), &output, sizeof(output), &k, nullptr) != 0;
}
```