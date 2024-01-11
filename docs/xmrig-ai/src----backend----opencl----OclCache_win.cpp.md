# `xmrig\src\backend\opencl\OclCache_win.cpp`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用：
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <direct.h>
#include <shlobj.h>
#include <windows.h>


#include "backend/opencl/OclCache.h"


void xmrig::OclCache::createDirectory()
{
    // 创建目录路径
    std::string path = prefix() + "/xmrig";
    // 创建目录
    _mkdir(path.c_str());

    // 更新路径
    path += "/.cache";
    // 创建目录
    _mkdir(path.c_str());
}


std::string xmrig::OclCache::prefix()
{
    // 获取本地应用程序数据文件夹路径
    char path[MAX_PATH + 1];
    if (SHGetSpecialFolderPathA(HWND_DESKTOP, path, CSIDL_LOCAL_APPDATA, FALSE)) {
        return path;
    }

    // 如果获取失败，返回当前目录
    return ".";
}
```