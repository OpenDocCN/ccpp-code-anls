# `xmrig\src\base\kernel\Entry.cpp`

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
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <cstdio>
#include <uv.h>


#ifdef XMRIG_FEATURE_TLS
#   include <openssl/opensslv.h>
#endif

#ifdef XMRIG_FEATURE_HWLOC
#   include <hwloc.h>
#endif

#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/wrappers/OclLib.h"
#   include "backend/opencl/wrappers/OclPlatform.h"
#endif

#include "base/kernel/Entry.h"
#include "base/kernel/Process.h"
#include "core/config/usage.h"
#include "version.h"


namespace xmrig {


static int showVersion()
{
    // 打印程序名称和版本号，以及构建日期
    printf(APP_NAME " " APP_VERSION "\n built on " __DATE__

#   if defined(__clang__)
    " with clang " __clang_version__);
#   elif defined(__GNUC__)
    " with GCC");
    // 如果是 GCC 编译器，打印 GCC 的版本号
    printf(" %d.%d.%d", __GNUC__, __GNUC_MINOR__, __GNUC_PATCHLEVEL__);
#   elif defined(_MSC_VER)
    " with MSVC");
    # 使用 printf 函数输出 MSVC_VERSION 的值
    printf(" %d", MSVC_VERSION);
#   else
    );
#   endif

    printf("\n features:"
#   if defined(__i386__) || defined(_M_IX86)
    " 32-bit"
#   elif defined(__x86_64__) || defined(_M_AMD64)
    " 64-bit"
#   endif

#   if defined(__AES__) || defined(_MSC_VER)
    " AES"
#   endif
    "\n");
    # 打印系统特性信息，包括32位/64位和AES支持情况

    printf("\nlibuv/%s\n", uv_version_string());
    # 打印libuv版本信息

#   if defined(XMRIG_FEATURE_TLS)
    {
#       if defined(LIBRESSL_VERSION_TEXT)
        printf("LibreSSL/%s\n", LIBRESSL_VERSION_TEXT + 9);
        # 如果支持TLS，则打印LibreSSL版本信息
#       elif defined(OPENSSL_VERSION_TEXT)
        constexpr const char *v = &OPENSSL_VERSION_TEXT[8];
        printf("OpenSSL/%.*s\n", static_cast<int>(strchr(v, ' ') - v), v);
        # 如果支持TLS，则打印OpenSSL版本信息
#       endif
    }
#   endif

#   if defined(XMRIG_FEATURE_HWLOC)
#   if defined(HWLOC_VERSION)
    printf("hwloc/%s\n", HWLOC_VERSION);
    # 如果支持硬件定位，则打印hwloc版本信息
#   elif HWLOC_API_VERSION >= 0x20000
    printf("hwloc/2\n");
    # 如果支持硬件定位，则打印hwloc版本信息
#   else
    printf("hwloc/1\n");
    # 如果支持硬件定位，则打印hwloc版本信息
#   endif
#   endif

    return 0;
    # 返回0，表示执行成功


#ifdef XMRIG_FEATURE_HWLOC
static int exportTopology(const Process &)
{
    const String path = Process::location(Process::ExeLocation, "topology.xml");
    # 定义路径变量path，用于存储topology.xml文件的路径

    hwloc_topology_t topology = nullptr;
    hwloc_topology_init(&topology);
    hwloc_topology_load(topology);
    # 初始化和加载硬件定位拓扑信息

#   if HWLOC_API_VERSION >= 0x20000
    if (hwloc_topology_export_xml(topology, path, 0) == -1) {
    # 如果硬件定位API版本大于等于2.0，则导出拓扑信息到xml文件
#   else
    if (hwloc_topology_export_xml(topology, path) == -1) {
    # 否则，导出拓扑信息到xml文件
#   endif
        printf("failed to export hwloc topology.\n");
        # 打印导出失败信息
    }
    else {
        printf("hwloc topology successfully exported to \"%s\"\n", path.data());
        # 打印成功导出拓扑信息的信息
    }

    hwloc_topology_destroy(topology);
    # 销毁硬件定位拓扑信息

    return 0;
    # 返回0，表示执行成功
}
#endif


} // namespace xmrig


xmrig::Entry::Id xmrig::Entry::get(const Process &process)
{
    const Arguments &args = process.arguments();
    # 获取进程的参数信息

    if (args.hasArg("-h") || args.hasArg("--help")) {
         return Usage;
         # 如果参数中包含-h或--help，则返回Usage
    }

    if (args.hasArg("-V") || args.hasArg("--version") || args.hasArg("--versions")) {
         return Version;
         # 如果参数中包含-V、--version或--versions，则返回Version
    }

#   ifdef XMRIG_FEATURE_HWLOC
    # 如果命令行参数中包含 "--export-topology"，则返回 Topo
    if (args.hasArg("--export-topology")) {
        return Topo;
    }
// 结束条件
#   endif

// 如果编译时开启了 OpenCL 特性，并且命令行参数中包含 --print-platforms，则返回平台信息
#   ifdef XMRIG_FEATURE_OPENCL
    if (args.hasArg("--print-platforms")) {
        return Platforms;
    }
#   endif

    // 默认返回 Default
    return Default;
}


// 执行函数，根据不同的 id 执行不同的操作
int xmrig::Entry::exec(const Process &process, Id id)
{
    switch (id) {
    // 如果 id 为 Usage，则打印用法信息并返回 0
    case Usage:
        printf("%s\n", usage().c_str());
        return 0;

    // 如果 id 为 Version，则显示版本信息
    case Version:
        return showVersion();

    // 如果编译时开启了 HWLOC 特性，并且 id 为 Topo，则导出拓扑信息
#   ifdef XMRIG_FEATURE_HWLOC
    case Topo:
        return exportTopology(process);
#   endif

    // 如果编译时开启了 OpenCL 特性，并且 id 为 Platforms，则初始化 OpenCL 并打印平台信息
#   ifdef XMRIG_FEATURE_OPENCL
    case Platforms:
        if (OclLib::init()) {
            OclPlatform::print();
        }
        return 0;
#   endif

    // 默认情况下返回 1
    default:
        break;
    }

    return 1;
}
```