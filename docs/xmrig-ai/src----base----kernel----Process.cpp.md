# `xmrig\src\base\kernel\Process.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布，无论许可证的版本是3还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <ctime>
#include <string>
#include <uv.h>


#include "base/kernel/Process.h"
#include "3rdparty/fmt/core.h"
#include "base/tools/Chrono.h"
#include "version.h"


#ifdef XMRIG_OS_WIN
#   ifdef _MSC_VER
#       include <direct.h>
#       define MKDIR(path) _mkdir(path.c_str());
#   else
#       define MKDIR(path) mkdir((path).c_str());
#   endif
#else
#   define MKDIR(path) mkdir(path.c_str(), 0700);
#endif


namespace xmrig {


static char pathBuf[520];
static std::string dataDir;


static std::string getPath(Process::Location location)
{
    size_t size = sizeof(pathBuf);

    if (location == Process::DataLocation) {
        if (!dataDir.empty()) {
            return dataDir;
        }

        location = Process::ExeLocation;
    }

    if (location == Process::HomeLocation) {
#       if UV_VERSION_HEX >= 0x010600
        return uv_os_homedir(pathBuf, &size) < 0 ? "" : std::string(pathBuf, size);
#       else
        location = Process::ExeLocation;
#       endif
    }

    if (location == Process::TempLocation) {
#       if UV_VERSION_HEX >= 0x010900
        return uv_os_tmpdir(pathBuf, &size) < 0 ? "" : std::string(pathBuf, size);
// 如果位置为 Process::ExeLocation，则获取当前可执行文件的路径
if (location == Process::ExeLocation) {
    // 如果获取可执行文件路径失败，则返回空字符串
    if (uv_exepath(pathBuf, &size) < 0) {
        return {};
    }
    // 将路径转换为字符串
    auto path = std::string(pathBuf, size);
    // 查找路径中最后一个目录分隔符的位置
    const auto pos = path.rfind(*XMRIG_DIR_SEPARATOR);
    // 如果找到目录分隔符，则返回分隔符之前的路径
    if (pos != std::string::npos) {
        return path.substr(0, pos);
    }
    // 否则返回整个路径
    return path;
}

// 如果位置为 Process::CwdLocation，则获取当前工作目录的路径
if (location == Process::CwdLocation) {
    // 获取当前工作目录的路径
    return uv_cwd(pathBuf, &size) < 0 ? "" : std::string(pathBuf, size);
}

// 其他情况返回空字符串
return {};
}

// 设置数据目录的路径
static void setDataDir(const char *path)
{
    // 如果路径为空，则直接返回
    if (path == nullptr) {
        return;
    }
    // 将路径转换为字符串
    std::string dir = path;
    // 如果路径不为空且最后一个字符是目录分隔符，则去除最后一个字符
    if (!dir.empty() && (dir.back() == '/' || dir.back() == '\\')) {
        dir.pop_back();
    }
    // 如果路径不为空且成功切换工作目录，则设置数据目录路径
    if (!dir.empty() && uv_chdir(dir.c_str()) == 0) {
        dataDir = dir;
    }
}

// 构造函数，初始化参数并设置数据目录路径
xmrig::Process::Process(int argc, char **argv) :
    m_arguments(argc, argv)
{
    // 设置随机种子
    srand(static_cast<unsigned int>(Chrono::currentMSecsSinceEpoch() ^ reinterpret_cast<uintptr_t>(this)));
    // 设置数据目录路径
    setDataDir(m_arguments.value("--data-dir", "-d"));

    // 如果定义了 XMRIG_SHARED_DATADIR 并且数据目录为空，则设置默认数据目录路径
    #ifdef XMRIG_SHARED_DATADIR
    if (dataDir.empty()) {
        dataDir = fmt::format("{}" XMRIG_DIR_SEPARATOR ".xmrig" XMRIG_DIR_SEPARATOR, location(HomeLocation));
        MKDIR(dataDir);
        dataDir += APP_KIND;
        MKDIR(dataDir);
        uv_chdir(dataDir.c_str());
    }
    #endif
}

// 获取父进程的进程ID
int xmrig::Process::ppid()
{
    // 如果 libuv 版本大于等于 1.17.0，则调用 uv_os_getppid() 获取父进程ID
    #if UV_VERSION_HEX >= 0x011000
    return uv_os_getppid();
    // 否则返回 0
    #else
    return 0;
    #endif
}

// 获取当前可执行文件的路径
xmrig::String xmrig::Process::exepath()
{
    size_t size = sizeof(pathBuf);
    // 获取当前可执行文件的路径
    return uv_exepath(pathBuf, &size) < 0 ? String("") : String(pathBuf, size);
}

// 获取指定位置的路径
xmrig::String xmrig::Process::location(Location location, const char *fileName)
{
    auto path = getPath(location);
    // 如果路径为空或文件名为空，则返回路径
    if (path.empty() || fileName == nullptr) {
        return path.c_str();
    }
    # 使用 fmt 库的 format 函数将 path 和 fileName 拼接成一个路径，并返回其 C 字符串形式
    return fmt::format("{}" XMRIG_DIR_SEPARATOR "{}", path, fileName).c_str();
# 闭合前面的函数定义
```