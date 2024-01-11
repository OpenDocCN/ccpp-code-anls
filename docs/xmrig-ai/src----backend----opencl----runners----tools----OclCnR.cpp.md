# `xmrig\src\backend\opencl\runners\tools\OclCnR.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/tools/OclCnR.h"  // 引入OclCnR工具类的头文件
#include "backend/opencl/cl/cn/cryptonight_r_cl.h"  // 引入cryptonight_r_cl的头文件
#include "backend/opencl/interfaces/IOclRunner.h"  // 引入IOclRunner接口的头文件
#include "backend/opencl/OclCache.h"  // 引入OclCache的头文件
#include "backend/opencl/OclLaunchData.h"  // 引入OclLaunchData的头文件
#include "backend/opencl/OclThread.h"  // 引入OclThread的头文件
#include "backend/opencl/wrappers/OclError.h"  // 引入OclError的头文件
#include "backend/opencl/wrappers/OclLib.h"  // 引入OclLib的头文件
#include "base/io/log/Log.h"  // 引入Log的头文件
#include "base/tools/Baton.h"  // 引入Baton的头文件
#include "base/tools/Chrono.h"  // 引入Chrono的头文件
#include "crypto/cn/CryptoNight_monero.h"  // 引入CryptoNight_monero的头文件

#include <cstring>  // 引入cstring标准库
#include <mutex>  // 引入mutex标准库
#include <regex>  // 引入regex标准库
#include <sstream>  // 引入sstream标准库
#include <string>  // 引入string标准库
#include <thread>  // 引入thread标准库
#include <uv.h>  // 引入uv.h头文件

namespace xmrig {

// CnrCacheEntry类的定义
class CnrCacheEntry
{
public:
    // 构造函数，初始化成员变量
    inline CnrCacheEntry(const Algorithm &algo, uint64_t offset, uint32_t index, cl_program program) :
        program(program),
        m_algo(algo),
        m_index(index),
        m_offset(offset)
    {}

    // 判断缓存是否过期
    inline bool isExpired(uint64_t offset) const                                    { return m_offset + OclCnR::kHeightChunkSize < offset; }
    // 检查算法、偏移量和索引是否匹配
    inline bool match(const Algorithm &algo, uint64_t offset, uint32_t index) const { return m_algo == algo && m_offset == offset && m_index == index; }
    // 检查运行器、偏移量是否匹配
    inline bool match(const IOclRunner &runner, uint64_t offset) const              { return match(runner.algorithm(), offset, runner.deviceIndex()); }
    // 释放程序资源
    inline void release() const                                                     { OclLib::release(program); }

    // OpenCL 程序对象
    cl_program program;
private:
    // 声明一个枚举类型的成员变量 m_algo
    Algorithm m_algo;
    // 声明一个 32 位无符号整数类型的成员变量 m_index
    uint32_t m_index;
    // 声明一个 64 位无符号整数类型的成员变量 m_offset
    uint64_t m_offset;
};


class CnrCache
{
public:
    // 默认构造函数
    CnrCache() = default;

    // 根据给定的 IOclRunner 对象和偏移量搜索缓存中的程序
    inline cl_program search(const IOclRunner &runner, uint64_t offset) { return search(runner.algorithm(), offset, runner.deviceIndex()); }

    // 根据给定的算法、偏移量和索引搜索缓存中的程序
    inline cl_program search(const Algorithm &algo, uint64_t offset, uint32_t index)
    {
        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 遍历缓存数据，查找匹配的条目
        for (const auto &entry : m_data) {
            if (entry.match(algo, offset, index)) {
                return entry.program;
            }
        }

        return nullptr;
    }

    // 向缓存中添加新的条目
    void add(const Algorithm &algo, uint64_t offset, uint32_t index, cl_program program)
    {
        // 如果已存在相同的条目，则释放新的程序对象并返回
        if (search(algo, offset, index)) {
            OclLib::release(program);
            return;
        }

        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 执行垃圾回收，清理过期的条目
        gc(offset);
        // 向缓存中添加新的条目
        m_data.emplace_back(algo, offset, index, program);
    }

    // 清空缓存
    void clear()
    {
        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 释放所有条目中的程序对象
        for (auto &entry : m_data) {
            entry.release();
        }

        // 清空缓存数据
        m_data.clear();
    }

private:
    // 执行垃圾回收，清理过期的条目
    void gc(uint64_t offset)
    {
        for (size_t i = 0; i < m_data.size();) {
            auto &entry = m_data[i];

            // 如果条目过期，则释放条目中的程序对象，并用最后一个条目替换当前条目
            if (entry.isExpired(offset)) {
                entry.release();
                entry = m_data.back();
                m_data.pop_back();
            }
            else {
                ++i;
            }
        }
    }

    // 互斥锁对象，用于保护临界区
    std::mutex m_mutex;
    // 存储缓存数据的向量
    std::vector<CnrCacheEntry> m_data;
};

// 静态全局缓存对象
static CnrCache cache;

class CnrBuilder
{
public:
    // 默认构造函数
    CnrBuilder() = default;

    // 根据给定的 IOclRunner 对象和偏移量构建程序
    cl_program build(const IOclRunner &runner, uint64_t offset)
    {
    #   ifdef APP_DEBUG
        // 获取当前时间戳
        const uint64_t ts = Chrono::steadyMSecs();
    // 结束 if 语句块

        // 使用互斥锁保护临界区，确保线程安全
        std::lock_guard<std::mutex> lock(m_mutex);
        // 在缓存中查找指定的程序对象
        cl_program program = cache.search(runner, offset);
        // 如果程序对象存在，则直接返回
        if (program) {
            return program;
        }

        // 定义变量
        cl_int ret = 0;
        // 获取指定偏移量处的源代码
        const std::string source = getSource(offset);
        // 获取 OpenCL 设备 ID
        cl_device_id device      = runner.data().device.id();
        // 获取源代码的 C 字符串表示
        const char *s            = source.c_str();

        // 使用源代码创建程序对象
        program = OclLib::createProgramWithSource(runner.ctx(), 1, &s, nullptr, &ret);
        // 如果创建程序对象失败，则返回空指针
        if (ret != CL_SUCCESS) {
            return nullptr;
        }

        // 构建程序对象
        if (OclLib::buildProgram(program, 1, &device, runner.buildOptions()) != CL_SUCCESS) {
            // 打印构建日志
            printf("BUILD LOG:\n%s\n", OclLib::getProgramBuildLog(program, device).data());
            // 释放程序对象并返回空指针
            OclLib::release(program);
            return nullptr;
        }

        // 打印调试信息
        LOG_DEBUG(GREEN_BOLD("[ocl]") " programs for heights %" PRIu64 " - %" PRIu64 " compiled. (%" PRIu64 "ms)", offset, offset + OclCnR::kHeightChunkSize - 1, Chrono::steadyMSecs() - ts);

        // 将程序对象添加到缓存中
        cache.add(runner.algorithm(), offset, runner.deviceIndex(), program);

        // 返回程序对象
        return program;
    }
private:
    // 获取指令代码的字符串表示
    static std::string getCode(const V4_Instruction *code, int code_size)
    {
        // 创建一个字符串流
        std::stringstream s;

        // 遍历指令代码数组
        for (int i = 0; i < code_size; ++i) {
            // 获取当前指令
            const V4_Instruction inst = code[i];

            // 获取指令中的目标和源操作数
            const uint32_t a = inst.dst_index;
            const uint32_t b = inst.src_index;

            // 根据指令类型进行不同的操作
            switch (inst.opcode)
            {
            case MUL:
                s << 'r' << a << "*=r" << b << ';';
                break;

            case ADD:
                s << 'r' << a << "+=r" << b << '+' << inst.C << "U;";
                break;

            case SUB:
                s << 'r' << a << "-=r" << b << ';';
                break;

            case ROR:
            case ROL:
                s << 'r' << a << "=rotate(r" << a << ((inst.opcode == ROR) ? ",ROT_BITS-r" : ",r") << b << ");";
                break;

            case XOR:
                s << 'r' << a << "^=r" << b << ';';
                break;
            }

            // 添加换行符
            s << '\n';
        }

        // 返回指令代码的字符串表示
        return s.str();
    }

    // 获取源代码
    static std::string getSource(uint64_t offset)
    {
        // 初始化源代码
        std::string source(cryptonight_r_defines_cl);

        // 遍历高度块大小
        for (size_t i = 0; i < OclCnR::kHeightChunkSize; ++i) {
            // 创建指令数组并获取指令数量
            V4_Instruction code[256];
            const int code_size      = v4_random_math_init<Algorithm::CN_R>(code, offset + i);
            // 替换源代码中的随机数学函数
            const std::string kernel = std::regex_replace(std::string(cryptonight_r_cl), std::regex("XMRIG_INCLUDE_RANDOM_MATH"), getCode(code, code_size));

            // 替换内核名称并添加到源代码中
            source += std::regex_replace(kernel, std::regex("KERNEL_NAME"), "cn1_" + std::to_string(offset + i));
        }

        // 返回源代码
        return source;
    }

    // 互斥锁
    std::mutex m_mutex;
};


class CnrBaton : public Baton<uv_work_t>
{
public:
    // 构造函数
    inline CnrBaton(const IOclRunner &runner, uint64_t offset) :
        runner(runner),
        offset(offset)
    {}

    // OpenCL 运行器和偏移量
    const IOclRunner &runner;
    const uint64_t offset;
};

// 静态构建器和后台线程互斥锁
static CnrBuilder builder;
static std::mutex bg_mutex;

} // namespace xmrig
// 根据给定的运行器和高度获取 OpenCL 程序
cl_program xmrig::OclCnR::get(const IOclRunner &runner, uint64_t height)
{
    // 计算当前高度所在的块的起始高度
    const uint64_t offset = (height / kHeightChunkSize) * kHeightChunkSize;

    // 如果当前高度所在的块只有一个高度，创建一个新的 CnrBaton 对象并进行异步处理
    if (offset + kHeightChunkSize - height == 1) {
        auto baton = new CnrBaton(runner, offset + kHeightChunkSize);

        // 将异步任务加入 libuv 的工作队列
        uv_queue_work(uv_default_loop(), &baton->req,
            [](uv_work_t *req) {
                auto baton = static_cast<CnrBaton*>(req->data);

                // 使用互斥锁保护临界区
                std::lock_guard<std::mutex> lock(bg_mutex);

                // 使用 builder 对象构建程序
                builder.build(baton->runner, baton->offset);
            },
            [](uv_work_t *req, int) { delete static_cast<CnrBaton*>(req->data); }
        );
    }

    // 在缓存中查找对应的程序
    cl_program program = cache.search(runner, offset);
    // 如果找到了程序，直接返回
    if (program) {
        return program;
    }

    // 如果缓存中没有对应的程序，使用 builder 对象构建程序并返回
    return builder.build(runner, offset);;
}

// 清空缓存
void xmrig::OclCnR::clear()
{
    // 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(bg_mutex);

    // 清空缓存
    cache.clear();
}
```