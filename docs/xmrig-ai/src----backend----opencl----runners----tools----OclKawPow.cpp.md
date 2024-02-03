# `xmrig\src\backend\opencl\runners\tools\OclKawPow.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/tools/OclKawPow.h"  // 引入OclKawPow.h头文件
#include "3rdparty/libethash/data_sizes.h"  // 引入data_sizes.h头文件
#include "3rdparty/libethash/ethash_internal.h"  // 引入ethash_internal.h头文件
#include "backend/opencl/cl/kawpow/kawpow_cl.h"  // 引入kawpow_cl.h头文件
#include "backend/opencl/interfaces/IOclRunner.h"  // 引入IOclRunner.h头文件
#include "backend/opencl/OclCache.h"  // 引入OclCache.h头文件
#include "backend/opencl/OclLaunchData.h"  // 引入OclLaunchData.h头文件
#include "backend/opencl/OclThread.h"  // 引入OclThread.h头文件
#include "backend/opencl/wrappers/OclError.h"  // 引入OclError.h头文件
#include "backend/opencl/wrappers/OclLib.h"  // 引入OclLib.h头文件
#include "base/io/log/Log.h"  // 引入Log.h头文件
#include "base/io/log/Tags.h"  // 引入Tags.h头文件
#include "base/tools/Baton.h"  // 引入Baton.h头文件
#include "base/tools/Chrono.h"  // 引入Chrono.h头文件
#include "crypto/kawpow/KPHash.h"  // 引入KPHash.h头文件

#include <cstring>  // 引入cstring头文件
#include <mutex>  // 引入mutex头文件
#include <regex>  // 引入regex头文件
#include <sstream>  // 引入sstream头文件
#include <string>  // 引入string头文件
#include <thread>  // 引入thread头文件
#include <uv.h>  // 引入uv.h头文件

namespace xmrig {  // 命名空间xmrig

class KawPowCacheEntry  // 定义KawPowCacheEntry类
{
public:  // 公共成员函数和变量
    inline KawPowCacheEntry(const Algorithm &algo, uint64_t period, uint32_t worksize, uint32_t index, cl_program program, cl_kernel kernel) :  // 构造函数
        program(program),  // 初始化program成员变量
        kernel(kernel),  // 初始化kernel成员变量
        m_algo(algo),  // 初始化m_algo成员变量
        m_index(index),  // 初始化m_index成员变量
        m_period(period),  // 初始化m_period成员变量
        m_worksize(worksize)  // 初始化m_worksize成员变量
    {}
    // 检查当前对象的周期是否已经过期
    inline bool isExpired(uint64_t period) const                                                       { return m_period + 1 < period; }
    // 检查当前对象是否与给定的算法、周期、工作大小和索引匹配
    inline bool match(const Algorithm &algo, uint64_t period, uint32_t worksize, uint32_t index) const { return m_algo == algo && m_period == period && m_worksize == worksize && m_index == index; }
    // 检查当前对象是否与给定的运行器、周期、工作大小匹配
    inline bool match(const IOclRunner &runner, uint64_t period, uint32_t worksize) const              { return match(runner.algorithm(), period, worksize, runner.deviceIndex()); }
    // 释放当前对象的内存资源
    inline void release() const                                                                        { OclLib::release(kernel); OclLib::release(program); }

    // OpenCL 程序对象
    cl_program program;
    // OpenCL 内核对象
    cl_kernel kernel;
// 定义私有成员变量
private:
    Algorithm m_algo; // 存储算法
    uint32_t m_index; // 存储索引
    uint64_t m_period; // 存储周期
    uint32_t m_worksize; // 存储工作大小
};


// 定义 KawPowCache 类
class KawPowCache
{
public:
    KawPowCache() = default; // 默认构造函数

    // 在缓存中搜索指定条件的内核
    inline cl_kernel search(const IOclRunner &runner, uint64_t period, uint32_t worksize) { return search(runner.algorithm(), period, worksize, runner.deviceIndex()); }

    // 在缓存中搜索指定条件的内核
    inline cl_kernel search(const Algorithm &algo, uint64_t period, uint32_t worksize, uint32_t index)
    {
        std::lock_guard<std::mutex> lock(m_mutex); // 加锁

        // 遍历缓存数据，查找匹配条件的内核
        for (const auto &entry : m_data) {
            if (entry.match(algo, period, worksize, index)) {
                return entry.kernel; // 返回匹配的内核
            }
        }

        return nullptr; // 如果没有匹配的内核，则返回空指针
    }

    // 向缓存中添加内核
    void add(const Algorithm &algo, uint64_t period, uint32_t worksize, uint32_t index, cl_program program, cl_kernel kernel)
    {
        if (search(algo, period, worksize, index)) { // 如果已经存在匹配的内核
            OclLib::release(kernel); // 释放内核资源
            OclLib::release(program); // 释放程序资源
            return; // 返回
        }

        std::lock_guard<std::mutex> lock(m_mutex); // 加锁

        gc(period); // 执行垃圾回收
        m_data.emplace_back(algo, period, worksize, index, program, kernel); // 向缓存中添加新的内核数据
    }

    // 清空缓存
    void clear()
    {
        std::lock_guard<std::mutex> lock(m_mutex); // 加锁

        for (auto &entry : m_data) {
            entry.release(); // 释放内核资源
        }

        m_data.clear(); // 清空缓存数据
    }

private:
    // 执行垃圾回收
    void gc(uint64_t period)
    {
        for (size_t i = 0; i < m_data.size();) {
            auto& entry = m_data[i];

            if (entry.isExpired(period)) { // 如果内核已过期
                entry.release(); // 释放内核资源
                entry = m_data.back(); // 将最后一个内核数据移动到当前位置
                m_data.pop_back(); // 移除最后一个内核数据
            }
            else {
                ++i; // 继续下一个内核数据
            }
        }
    }

    std::mutex m_mutex; // 互斥锁
    std::vector<KawPowCacheEntry> m_data; // 存储 KawPowCacheEntry 对象的容器
};

static KawPowCache cache; // 静态的 KawPowCache 对象

// 定义宏函数 rnd()
#define rnd()       (kiss99(rnd_state))
// 定义宏函数 mix_src()
#define mix_src()   ("mix[" + std::to_string(rnd() % KPHash::REGS) + "]")
// 定义一个宏函数，用于生成 mix_dst 字符串
#define mix_dst()   ("mix[" + std::to_string(mix_seq_dst[(mix_seq_dst_cnt++) % KPHash::REGS]) + "]")
// 定义一个宏函数，用于生成 mix_cache 字符串
#define mix_cache() ("mix[" + std::to_string(mix_seq_cache[(mix_seq_cache_cnt++) % KPHash::REGS]) + "]")

// 定义 KawPowBaton 类，继承自 Baton<uv_work_t>
class KawPowBaton : public Baton<uv_work_t>
{
public:
    // 构造函数，初始化成员变量
    inline KawPowBaton(const IOclRunner& runner, uint64_t period, uint32_t worksize) :
        runner(runner),
        period(period),
        worksize(worksize)
    {}

    // 成员变量，用于存储 IOclRunner 对象的引用、周期和工作大小
    const IOclRunner& runner;
    const uint64_t period;
    const uint32_t worksize;
};

// 定义 KawPowBuilder 类
class KawPowBuilder
{
public:
    // 析构函数，用于释放资源
    ~KawPowBuilder()
    {
        // 如果 m_loop 不为空，则发送关闭异步消息，等待线程结束，然后释放 m_loop
        if (m_loop) {
            uv_async_send(&m_shutdownAsync);
            uv_thread_join(&m_loopThread);
            delete m_loop;
        }
    }

    // 异步构建函数，接受 runner、period 和 worksize 作为参数
    void build_async(const IOclRunner& runner, uint64_t period, uint32_t worksize);

    // 同步构建函数，接受 runner、period 和 worksize 作为参数
    cl_kernel build(const IOclRunner &runner, uint64_t period, uint32_t worksize)
    # 使用互斥锁保护临界区，确保线程安全
    {
        std::lock_guard<std::mutex> lock(m_mutex);

        # 获取当前时间戳
        const uint64_t ts = Chrono::steadyMSecs();

        # 在缓存中查找是否已存在指定条件下的内核，如果存在则直接返回
        cl_kernel kernel = cache.search(runner, period, worksize);
        if (kernel) {
            return kernel;
        }

        # 定义 OpenCL 函数返回值
        cl_int ret = 0;
        # 获取指定周期的源代码
        const std::string source = getSource(period);
        # 获取设备 ID
        cl_device_id device      = runner.data().device.id();
        # 获取源代码的 C 字符串表示
        const char *s            = source.c_str();

        # 使用源代码创建 OpenCL 程序对象
        cl_program program = OclLib::createProgramWithSource(runner.ctx(), 1, &s, nullptr, &ret);
        # 如果创建程序失败，则返回空指针
        if (ret != CL_SUCCESS) {
            return nullptr;
        }

        # 设置编译选项
        std::string options = " -DPROGPOW_DAG_ELEMENTS=";
        # 计算当前周期的 DAG 元素数量
        const uint64_t epoch = (period * KPHash::PERIOD_LENGTH) / KPHash::EPOCH_LENGTH;
        const uint64_t dag_elements = dag_sizes[epoch] / 256;
        options += std::to_string(dag_elements);
        options += " -DGROUP_SIZE=";
        options += std::to_string(worksize);
        options += runner.buildOptions();

        # 使用指定选项编译程序
        if (OclLib::buildProgram(program, 1, &device, options.c_str()) != CL_SUCCESS) {
            # 如果编译失败，则打印编译日志，释放程序对象，并返回空指针
            printf("BUILD LOG:\n%s\n", OclLib::getProgramBuildLog(program, device).data());
            OclLib::release(program);
            return nullptr;
        }

        # 创建内核对象
        kernel = OclLib::createKernel(program, "progpow_search", &ret);
        # 如果创建内核失败，则释放程序对象，并返回空指针
        if (ret != CL_SUCCESS) {
            OclLib::release(program);
            return nullptr;
        }

        # 打印编译成功的信息，并记录编译耗时
        LOG_INFO("%s " YELLOW("KawPow") " program for period " WHITE_BOLD("%" PRIu64) " compiled " BLACK_BOLD("(%" PRIu64 "ms)"), Tags::opencl(), period, Chrono::steadyMSecs() - ts);

        # 将编译成功的程序和内核添加到缓存中
        cache.add(runner.algorithm(), period, worksize, runner.deviceIndex(), program, kernel);

        # 返回创建的内核对象
        return kernel;
    }
    // 声明私有互斥量
    std::mutex m_mutex;

    // 定义一个名为 kiss99_t 的结构体，包含四个无符号整型成员变量
    typedef struct {
        uint32_t z, w, jsr, jcong;
    } kiss99_t;

    // 获取源字符串的函数，参数为程序种子，返回字符串
    static std::string getSource(uint64_t prog_seed)
    }

    // 合并字符串的函数，参数为两个字符串和一个无符号整型数，返回合并后的字符串
    static std::string merge(const std::string& a, const std::string& b, uint32_t r)
    {
        // 根据 r 取模后的结果选择不同的合并方式
        switch (r % 4)
        {
        case 0:
            return a + " = (" + a + " * 33) + " + b + ";\n";
        case 1:
            return a + " = (" + a + " ^ " + b + ") * 33;\n";
        case 2:
            return a + " = ROTL32(" + a + ", " + std::to_string(((r >> 16) % 31) + 1) + ") ^ " + b + ";\n";
        case 3:
            return a + " = ROTR32(" + a + ", " + std::to_string(((r >> 16) % 31) + 1) + ") ^ " + b + ";\n";
        }
        return "#error\n";
    }

    // 执行数学运算的函数，参数为三个字符串和一个无符号整型数，返回执行运算后的字符串
    static std::string math(const std::string& d, const std::string& a, const std::string& b, uint32_t r)
    {
        // 根据 r 取模后的结果选择不同的数学运算方式
        switch (r % 11)
        {
        case 0:
            return d + " = " + a + " + " + b + ";\n";
        case 1:
            return d + " = " + a + " * " + b + ";\n";
        case 2:
            return d + " = mul_hi(" + a + ", " + b + ");\n";
        case 3:
            return d + " = min(" + a + ", " + b + ");\n";
        case 4:
            return d + " = ROTL32(" + a + ", " + b + " % 32);\n";
        case 5:
            return d + " = ROTR32(" + a + ", " + b + " % 32);\n";
        case 6:
            return d + " = " + a + " & " + b + ";\n";
        case 7:
            return d + " = " + a + " | " + b + ";\n";
        case 8:
            return d + " = " + a + " ^ " + b + ";\n";
        case 9:
            return d + " = clz(" + a + ") + clz(" + b + ");\n";
        case 10:
            return d + " = popcount(" + a + ") + popcount(" + b + ");\n";
        }
        return "#error\n";
    }

    // 执行 FNV-1a 哈希算法的函数，参数为两个无符号整型数的引用，返回哈希值
    static uint32_t fnv1a(uint32_t& h, uint32_t d)
    {
        return h = (h ^ d) * 0x1000193;
    }

    // 执行 KISS99 伪随机数生成算法的函数，参数为一个 kiss99_t 结构体的引用，返回一个无符号整型数
    static uint32_t kiss99(kiss99_t& st)
    {
        # 更新状态变量 z
        st.z = 36969 * (st.z & 65535) + (st.z >> 16);
        # 更新状态变量 w
        st.w = 18000 * (st.w & 65535) + (st.w >> 16);
        # 计算组合值 MWC
        uint32_t MWC = ((st.z << 16) + st.w);
        # 更新状态变量 jsr
        st.jsr ^= (st.jsr << 17);
        st.jsr ^= (st.jsr >> 13);
        st.jsr ^= (st.jsr << 5);
        # 更新状态变量 jcong
        st.jcong = 69069 * st.jcong + 1234567;
        # 返回最终结果
        return ((MWC ^ st.jcong) + st.jsr);
    }
// 声明私有成员变量
private:
    uv_loop_t* m_loop          = nullptr; // UV 循环对象指针
    uv_thread_t m_loopThread   = {}; // UV 线程对象
    uv_async_t m_shutdownAsync = {}; // UV 异步对象，用于关闭操作
    uv_async_t m_batonAsync    = {}; // UV 异步对象，用于处理任务

    std::vector<KawPowBaton> m_batons; // 存储 KawPowBaton 对象的向量

    // 静态方法，用于运行 UV 循环
    static void loop(void* data)
    {
        KawPowBuilder* builder = static_cast<KawPowBuilder*>(data);
        uv_run(builder->m_loop, UV_RUN_DEFAULT); // 运行 UV 循环
        uv_loop_close(builder->m_loop); // 关闭 UV 循环
    }
};


static KawPowBuilder builder; // 创建 KawPowBuilder 对象


// 异步构建方法
void KawPowBuilder::build_async(const IOclRunner& runner, uint64_t period, uint32_t worksize)
{
    std::lock_guard<std::mutex> lock(m_mutex); // 使用互斥锁保护临界区

    if (!m_loop) {
        m_loop = new uv_loop_t{}; // 初始化 UV 循环对象
        uv_loop_init(m_loop); // 初始化 UV 循环

        // 初始化关闭异步对象
        uv_async_init(m_loop, &m_shutdownAsync, [](uv_async_t* handle)
            {
                KawPowBuilder* builder = reinterpret_cast<KawPowBuilder*>(handle->data);
                uv_close(reinterpret_cast<uv_handle_t*>(&builder->m_shutdownAsync), nullptr); // 关闭关闭异步对象
                uv_close(reinterpret_cast<uv_handle_t*>(&builder->m_batonAsync), nullptr); // 关闭任务异步对象
            });

        // 初始化任务异步对象
        uv_async_init(m_loop, &m_batonAsync, [](uv_async_t* handle)
            {
                std::vector<KawPowBaton> batons;
                {
                    KawPowBuilder* b = reinterpret_cast<KawPowBuilder*>(handle->data);

                    std::lock_guard<std::mutex> lock(b->m_mutex); // 使用互斥锁保护临界区
                    batons = std::move(b->m_batons); // 移动存储的任务到局部变量
                }

                for (const KawPowBaton& baton : batons) {
                    builder.build(baton.runner, baton.period, baton.worksize); // 调用 build 方法处理任务
                }
            });

        m_shutdownAsync.data = this; // 设置关闭异步对象的数据指针
        m_batonAsync.data = this; // 设置任务异步对象的数据指针

        uv_thread_create(&m_loopThread, loop, this); // 创建并运行 UV 线程
    }

    m_batons.emplace_back(runner, period, worksize); // 将任务添加到任务列表
    uv_async_send(&m_batonAsync); // 发送任务异步信号
}


cl_kernel OclKawPow::get(const IOclRunner &runner, uint64_t height, uint32_t worksize)
{
    const uint64_t period = height / KPHash::PERIOD_LENGTH; // 计算周期
    # 如果缓存中不存在指定参数的内核，则执行以下代码块
    if (!cache.search(runner, period + 1, worksize)) {
        # 使用 builder 对象异步构建指定参数的内核
        builder.build_async(runner, period + 1, worksize);
    }

    # 从缓存中搜索指定参数的内核
    cl_kernel kernel = cache.search(runner, period, worksize);
    # 如果找到了内核，则返回该内核
    if (kernel) {
        return kernel;
    }

    # 如果缓存中不存在指定参数的内核，则使用 builder 对象构建内核并返回
    return builder.build(runner, period, worksize);
// 清空 OclKawPow 对象的缓存
void OclKawPow::clear()
{
    // 清空缓存
    cache.clear();
}
// 结束 xmrig 命名空间
} // namespace xmrig
```