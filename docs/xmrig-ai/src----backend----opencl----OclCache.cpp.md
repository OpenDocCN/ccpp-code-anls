# `xmrig\src\backend\opencl\OclCache.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   许可证的第3版或（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <fstream>
#include <map>
#include <mutex>
#include <sstream>


#include "backend/opencl/OclCache.h"
#include "3rdparty/base32/base32.h"
#include "backend/common/Tags.h"
#include "backend/opencl/interfaces/IOclRunner.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/crypto/keccak.h"
#include "base/io/log/Log.h"
#include "base/tools/Chrono.h"


namespace xmrig {


static std::mutex mutex;


static cl_program createFromSource(const IOclRunner *runner)
{
    // 记录信息到日志，指示GPU编译
    LOG_INFO("%s GPU " WHITE_BOLD("#%zu") " " YELLOW_BOLD("compiling..."), ocl_tag(), runner->data().device.index());

    cl_int ret = 0;
    cl_device_id device = runner->data().device.id();
    const char *source  = runner->source();
    const uint64_t ts   = Chrono::steadyMSecs();

    // 使用源代码创建OpenCL程序
    cl_program program = OclLib::createProgramWithSource(runner->ctx(), 1, &source, nullptr, &ret);
    if (ret != CL_SUCCESS) {
        return nullptr;
    }
    # 如果使用 OclLib 构建程序失败
    if (OclLib::buildProgram(program, 1, &device, runner->buildOptions()) != CL_SUCCESS) {
        # 打印构建日志
        printf("BUILD LOG:\n%s\n", OclLib::getProgramBuildLog(program, device).data());
        # 释放程序资源
        OclLib::release(program);
        # 返回空指针
        return nullptr;
    }

    # 打印编译完成的信息，包括标签、设备索引和编译耗时
    LOG_INFO("%s GPU " WHITE_BOLD("#%zu") " " GREEN_BOLD("compilation completed") BLACK_BOLD(" (%" PRIu64 " ms)"),
             ocl_tag(), runner->data().device.index(), Chrono::steadyMSecs() - ts);

    # 返回程序
    return program;
} // 结束命名空间 xmrig

// 从二进制文件创建 OpenCL 程序
static cl_program createFromBinary(const IOclRunner *runner, const std::string &fileName)
{
    // 以二进制只读模式打开文件
    std::ifstream file(fileName, std::ofstream::in | std::ofstream::binary);
    // 如果文件打开失败，返回空指针
    if (!file.good()) {
        return nullptr;
    }

    // 读取文件内容到字符串流
    std::ostringstream ss;
    ss << file.rdbuf();

    // 将字符串流转换为字符串
    const std::string s = ss.str();
    // 获取字符串长度
    const size_t bin_size = s.size();
    // 获取字符串数据指针
    auto data_ptr = s.data();
    // 获取 OpenCL 设备 ID
    cl_device_id device = runner->data().device.id();

    cl_int clStatus = 0;
    cl_int ret = 0;
    // 使用二进制数据创建 OpenCL 程序
    cl_program program = OclLib::createProgramWithBinary(runner->ctx(), 1, &device, &bin_size, reinterpret_cast<const unsigned char **>(&data_ptr), &clStatus, &ret);
    // 如果创建失败，返回空指针
    if (ret != CL_SUCCESS) {
        return nullptr;
    }

    // 如果编译失败，释放程序并返回空指针
    if (OclLib::buildProgram(program, 1, &device) != CL_SUCCESS) {
        OclLib::release(program);
        return nullptr;
    }

    // 返回创建的程序
    return program;
}

// 构建 OpenCL 程序
cl_program xmrig::OclCache::build(const IOclRunner *runner)
{
    // 加锁
    std::lock_guard<std::mutex> lock(mutex);

    // 如果 Nonce 序列为 0，返回空指针
    if (Nonce::sequence(Nonce::OPENCL) == 0) {
        return nullptr;
    }

    std::string fileName;
    // 如果 runner 的缓存存在
    if (runner->data().cache) {
        // 根据操作系统设置文件名
#       ifdef _WIN32
        fileName = prefix() + "\\xmrig\\.cache\\" + cacheKey(runner) + ".bin";
#       else
        fileName = prefix() + "/.cache/" + cacheKey(runner) + ".bin";
#       endif

        // 从二进制文件创建程序
        cl_program program = createFromBinary(runner, fileName);
        // 如果创建成功，返回程序
        if (program) {
            return program;
        }
    }

    // 从源代码创建程序
    cl_program program = createFromSource(runner);
    // 如果 runner 的缓存存在且程序创建成功，保存程序到文件
    if (runner->data().cache && program) {
        save(program, fileName);
    }

    // 返回程序
    return program;
}

// 生成缓存键
std::string xmrig::OclCache::cacheKey(const char *deviceKey, const char *options, const char *source)
{
    // 将源代码、选项和设备键连接成一个字符串
    std::string in(source);
    in += options;
    in += deviceKey;

    // 计算哈希值
    uint8_t hash[200];
    keccak(in.c_str(), in.size(), hash);

    // 对哈希值进行 base32 编码
    uint8_t result[32] = { 0 };
    base32_encode(hash, 12, result, sizeof(result));
}
    // 将result重新解释为char指针类型，并返回
    return reinterpret_cast<char *>(result);
// 生成用于缓存的键值，由设备键、构建选项和源代码组成
std::string xmrig::OclCache::cacheKey(const IOclRunner *runner)
{
    return cacheKey(runner->deviceKey(), runner->buildOptions(), runner->source());
}

// 将 OpenCL 程序保存到文件
void xmrig::OclCache::save(cl_program program, const std::string &fileName)
{
    // 获取程序的二进制大小
    size_t size = 0;
    if (OclLib::getProgramInfo(program, CL_PROGRAM_BINARY_SIZES, sizeof(size), &size) != CL_SUCCESS) {
        return;
    }

    // 创建二进制数据的缓冲区
    std::vector<char> binary(size);

    // 获取程序的二进制数据
    char *data = binary.data();
    if (OclLib::getProgramInfo(program, CL_PROGRAM_BINARIES, sizeof(char *), &data) != CL_SUCCESS) {
        return;
    }

    // 创建保存文件的目录
    createDirectory();

    // 打开文件流，写入二进制数据，并关闭文件流
    std::ofstream file_stream;
    file_stream.open(fileName, std::ofstream::out | std::ofstream::binary);
    file_stream.write(binary.data(), static_cast<int64_t>(binary.size()));
    file_stream.close();
}
```