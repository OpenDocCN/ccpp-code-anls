# `xmrig\src\crypto\ghostrider\ghostrider.cpp`

```cpp
/* XMRig
 * 版权所有 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "ghostrider.h"
#include "sph_blake.h"
#include "sph_bmw.h"
#include "sph_groestl.h"
#include "sph_jh.h"
#include "sph_keccak.h"
#include "sph_skein.h"
#include "sph_luffa.h"
#include "sph_cubehash.h"
#include "sph_shavite.h"
#include "sph_simd.h"
#include "sph_echo.h"
#include "sph_hamsi.h"
#include "sph_fugue.h"
#include "sph_shabal.h"
#include "sph_whirlpool.h"

#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/tools/Chrono.h"
#include "backend/cpu/Cpu.h"
#include "crypto/cn/CnHash.h"
#include "crypto/cn/CnCtx.h"
#include "crypto/cn/CryptoNight.h"
#include "crypto/common/VirtualMemory.h"

#include <thread>
#include <atomic>
#include <uv.h>

#ifdef XMRIG_FEATURE_HWLOC
#   include "base/kernel/Platform.h"
#   include <hwloc.h>

#   if HWLOC_API_VERSION < 0x20000
#       define HWLOC_OBJ_L3CACHE HWLOC_OBJ_CACHE
#   endif
#endif

#if defined(XMRIG_ARM)
#   include "crypto/cn/sse2neon.h"
#elif defined(__GNUC__)
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define CORE_HASH(i, x) static void h##i(const uint8_t* data, size_t size, uint8_t* output) \
{ \
    // 定义哈希函数，参数为输入数据、大小和输出
    sph_##x##_context ctx; \
    # 调用特定哈希算法的初始化函数，初始化哈希上下文
    sph_##x##_init(&ctx); \
    # 使用特定哈希算法处理数据，更新哈希上下文
    sph_##x(&ctx, data, size); \
    # 完成特定哈希算法的计算，输出哈希值到指定的输出数组
    sph_##x##_close(&ctx, output); \
// 定义宏，用于生成一系列的核心哈希函数
CORE_HASH( 0, blake512   );
CORE_HASH( 1, bmw512     );
CORE_HASH( 2, groestl512 );
CORE_HASH( 3, jh512      );
CORE_HASH( 4, keccak512  );
CORE_HASH( 5, skein512   );
CORE_HASH( 6, luffa512   );
CORE_HASH( 7, cubehash512);
CORE_HASH( 8, shavite512 );
CORE_HASH( 9, simd512    );
CORE_HASH(10, echo512    );
CORE_HASH(11, hamsi512   );
CORE_HASH(12, fugue512   );
CORE_HASH(13, shabal512  );
CORE_HASH(14, whirlpool  );

// 取消之前定义的宏
#undef CORE_HASH

// 定义一个函数指针数组，用于存储核心哈希函数
using core_hash_func = void (*)(const uint8_t* data, size_t size, uint8_t* output);
static const core_hash_func core_hash[15] = { h0, h1, h2, h3, h4, h5, h6, h7, h8, h9, h10, h11, h12, h13, h14 };

// 命名空间定义
namespace xmrig
{
    // 定义一个包含6个元素的常量数组，表示不同的哈希算法
    static constexpr Algorithm::Id cn_hash[6] = {
        Algorithm::CN_GR_0,
        Algorithm::CN_GR_1,
        Algorithm::CN_GR_2,
        Algorithm::CN_GR_3,
        Algorithm::CN_GR_4,
        Algorithm::CN_GR_5,
    };

    // 定义一个包含6个元素的常量字符串数组，表示不同哈希算法的名称
    static constexpr const char* cn_names[6] = {
        "cn/dark (512 KB)",
        "cn/dark-lite (256 KB)",
        "cn/fast (2 MB)",
        "cn/lite (1 MB)",
        "cn/turtle (256 KB)",
        "cn/turtle-lite (128 KB)",
    };

    // 定义一个包含6个元素的常量大小数组，表示不同哈希算法的大小
    static constexpr size_t cn_sizes[6] = {
        Algorithm::l3(Algorithm::CN_GR_0),     // 512 KB
        Algorithm::l3(Algorithm::CN_GR_1) / 2, // 256 KB
        Algorithm::l3(Algorithm::CN_GR_2),     // 2 MB
        Algorithm::l3(Algorithm::CN_GR_3),     // 1 MB
        Algorithm::l3(Algorithm::CN_GR_4),     // 256 KB
        Algorithm::l3(Algorithm::CN_GR_5) / 2, // 128 KB
    };

    // 定义一个包含5个元素的常量数组，表示硬件AES加速的哈希算法变体
    static constexpr CnHash::AlgoVariant av_hw_aes[5] = { CnHash::AV_SINGLE, CnHash::AV_SINGLE, CnHash::AV_DOUBLE, CnHash::AV_TRIPLE, CnHash::AV_QUAD };
    // 定义一个包含5个元素的常量数组，表示软件AES加速的哈希算法变体
    static constexpr CnHash::AlgoVariant av_soft_aes[5] = { CnHash::AV_SINGLE_SOFT, CnHash::AV_SINGLE_SOFT, CnHash::AV_DOUBLE_SOFT, CnHash::AV_TRIPLE_SOFT, CnHash::AV_QUAD_SOFT };

    // 模板函数，用于选择一定数量的索引
    template<size_t N>
    static inline void select_indices(uint32_t (&indices)[N], const uint8_t* seed)
    {
        // 布尔数组，用于标记索引是否被选择
        bool selected[N] = {};

        // 初始化计数器
        uint32_t k = 0;
    # 遍历64次，生成索引值，确保不重复
    for (uint32_t i = 0; i < 64; ++i) {
        # 计算索引值，使用种子数组中的值进行位运算和取模运算
        const uint8_t index = ((seed[i / 2] >> ((i & 1) * 4)) & 0xF) % N;
        # 如果索引值对应的元素未被选中，则选中该元素，并将其索引值存入indices数组中
        if (!selected[index]) {
            selected[index] = true;
            indices[k++] = index;
            # 如果已经选中了N个元素，则结束循环
            if (k >= N) {
                return;
            }
        }
    }

    # 遍历N次，将未被选中的元素的索引值存入indices数组中
    for (uint32_t i = 0; i < N; ++i) {
        if (!selected[i]) {
            indices[k++] = i;
        }
    }
}
// 结束 ghostrider 命名空间

#ifdef XMRIG_FEATURE_HWLOC
// 如果定义了 XMRIG_FEATURE_HWLOC

static struct AlgoTune
{
    double hashrate = 0.0;
    uint32_t step = 1;
    uint32_t threads = 1;
} tuneDefault[6], tune8MB[6];
// 定义了一个静态结构体 AlgoTune，包含 hashrate、step 和 threads 三个成员变量，以及两个数组 tuneDefault 和 tune8MB

struct HelperThread
{
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HelperThread)
    // 禁用 HelperThread 的拷贝和移动构造函数

    HelperThread(hwloc_bitmap_t cpu_set, int priority, bool is8MB) : m_cpuSet(cpu_set), m_priority(priority), m_is8MB(is8MB)
    {
        // HelperThread 的构造函数，初始化成员变量
        uv_mutex_init(&m_mutex);
        uv_cond_init(&m_cond);

        m_thread = new std::thread(&HelperThread::run, this);
        // 创建一个新的线程，并调用 HelperThread 的 run 方法
        do {
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        } while (!m_ready);
        // 在线程准备好之前一直等待
    }

    ~HelperThread()
    {
        // HelperThread 的析构函数
        uv_mutex_lock(&m_mutex);
        m_finished = true;
        uv_cond_signal(&m_cond);
        uv_mutex_unlock(&m_mutex);
        // 设置标志位，发送条件信号，解锁互斥量

        m_thread->join();
        delete m_thread;
        // 等待线程结束并释放资源

        uv_mutex_destroy(&m_mutex);
        uv_cond_destroy(&m_cond);
        // 销毁互斥量和条件变量

        hwloc_bitmap_free(m_cpuSet);
        // 释放 CPU 核心集合
    }

    struct TaskBase
    {
        XMRIG_DISABLE_COPY_MOVE(TaskBase)
        // 禁用 TaskBase 的拷贝和移动构造函数

        TaskBase()          = default;
        virtual ~TaskBase() = default;
        virtual void run()  = 0;
        // 定义 TaskBase 结构体，包含虚析构函数和纯虚函数 run
    };

    template<typename T>
    struct Task : TaskBase
    {
        explicit inline Task(T&& task) : m_task(std::move(task))
        {
            static_assert(sizeof(Task) <= 128, "Task struct is too large");
        }
        // 定义 Task 结构体，包含一个带参构造函数和一个静态断言

        void run() override
        {
            m_task();
            this->~Task();
        }
        // 重写虚函数 run，执行任务并销毁 Task 对象

        T m_task;
    };

    template<typename T>
    inline void launch_task(T&& task)
    {
        uv_mutex_lock(&m_mutex);
        new (&m_tasks[m_numTasks++]) Task<T>(std::forward<T>(task));
        uv_cond_signal(&m_cond);
        uv_mutex_unlock(&m_mutex);
    }
    // 启动任务的函数，将任务添加到任务队列中并发送条件信号

    inline void wait() const
    {
        while (m_numTasks) {
            _mm_pause();
        }
    }
    // 等待任务队列为空

    void run()
    {
        // 如果 CPU 核心集合中的核心数量大于 0
        if (hwloc_bitmap_weight(m_cpuSet) > 0) {
            // 获取当前 CPU 拓扑结构
            hwloc_topology_t topology = Cpu::info()->topology();
            // 将当前线程绑定到指定的 CPU 核心集合上
            if (hwloc_set_cpubind(topology, m_cpuSet, HWLOC_CPUBIND_THREAD | HWLOC_CPUBIND_STRICT) < 0) {
                // 如果绑定失败，则使用非严格模式进行绑定
                hwloc_set_cpubind(topology, m_cpuSet, HWLOC_CPUBIND_THREAD);
            }
        }
    
        // 设置当前线程的优先级
        Platform::setThreadPriority(m_priority);
    
        // 获取互斥锁
        uv_mutex_lock(&m_mutex);
        // 设置标志位为 true，表示线程已经准备好
        m_ready = true;
    
        // 循环执行以下操作，直到线程结束
        do {
            // 等待条件变量的信号
            uv_cond_wait(&m_cond, &m_mutex);
    
            // 获取当前任务数量
            const uint32_t n = m_numTasks;
            // 如果任务数量大于 0
            if (n > 0) {
                // 遍历任务数组，依次执行任务
                for (uint32_t i = 0; i < n; ++i) {
                    // 执行任务
                    reinterpret_cast<TaskBase*>(&m_tasks[i])->run();
                }
                // 添加内存屏障，保证任务执行顺序
                std::atomic_thread_fence(std::memory_order_seq_cst);
                // 重置任务数量为 0
                m_numTasks = 0;
            }
        } while (!m_finished); // 循环直到线程结束
    
        // 释放互斥锁
        uv_mutex_unlock(&m_mutex);
    }
    
    // 定义互斥锁和条件变量
    uv_mutex_t m_mutex;
    uv_cond_t m_cond;
    
    // 用于存储任务的数组，每个任务大小为 128 字节，共有 4 个任务
    alignas(16) uint8_t m_tasks[4][128] = {};
    // 当前任务数量
    volatile uint32_t m_numTasks = 0;
    // 标志位，表示线程是否准备好
    volatile bool m_ready = false;
    // 标志位，表示线程是否结束
    volatile bool m_finished = false;
    // CPU 核心集合
    hwloc_bitmap_t m_cpuSet = {};
    // 线程优先级
    int m_priority = -1;
    // 是否为 8MB
    bool m_is8MB = false;
    
    // 线程指针
    std::thread* m_thread = nullptr;
};
// 结束 benchmark 函数

void benchmark()
{
#ifndef XMRIG_ARM
    // 定义静态原子变量 done，用于标记是否已经执行过 benchmark
    static std::atomic<int> done{ 0 };
    // 如果 done 的值为 1，则表示已经执行过 benchmark，直接返回
    if (done.exchange(1)) {
        return;
    }

    });

    // 等待辅助线程结束
    t.join();

    // 输出 GhostRider 调优结果的分隔线和标题
    LOG_VERBOSE("---------------------------------------------");
    LOG_VERBOSE("|         GhostRider tuning results         |");
    LOG_VERBOSE("---------------------------------------------");

    // 遍历算法，输出调优结果
    for (int algo = 0; algo < 6; ++algo) {
        LOG_VERBOSE("%24s | %ux%u | %.2f h/s", cn_names[algo], tuneDefault[algo].step, tuneDefault[algo].threads, tuneDefault[algo].hashrate);
        // 如果 8MB 调优结果与默认结果不同，输出 8MB 调优结果
        if ((tune8MB[algo].step != tuneDefault[algo].step) || (tune8MB[algo].threads != tuneDefault[algo].threads)) {
            LOG_VERBOSE("%24s | %ux%u | %.2f h/s", cn_names[algo], tune8MB[algo].step, tune8MB[algo].threads, tune8MB[algo].hashrate);
        }
    }
#endif
}

// 根据类型查找对象
template <typename func>
static inline bool findByType(hwloc_obj_t obj, hwloc_obj_type_t type, func lambda)
{
    for (size_t i = 0; i < obj->arity; i++) {
        // 如果子对象的类型与指定类型相同，执行 lambda 函数
        if (obj->children[i]->type == type) {
            if (lambda(obj->children[i])) {
                return true;
            }
        }
        else {
            // 递归查找指定类型的子对象
            if (findByType(obj->children[i], type, lambda)) {
                return true;
            }
        }
    }
    return false;
}

// 创建辅助线程
HelperThread* create_helper_thread(int64_t cpu_index, int priority, const std::vector<int64_t>& affinities)
{
#ifndef XMRIG_ARM
    // 分配 CPU 亲和性位图
    hwloc_bitmap_t helper_cpu_set = hwloc_bitmap_alloc();
    hwloc_bitmap_t main_threads_set = hwloc_bitmap_alloc();

    // 设置主线程的亲和性
    for (int64_t i : affinities) {
        if (i >= 0) {
            hwloc_bitmap_set(main_threads_set, i);
        }
    }

    // 如果 CPU 索引大于等于 0
    if (cpu_index >= 0) {
        // 获取系统拓扑结构的根节点
        hwloc_obj_t root = hwloc_get_root_obj(Cpu::info()->topology());

        // 是否为 8MB 缓存
        bool is8MB = false;

        // 根据类型查找 L3CACHE 对象
        findByType(root, HWLOC_OBJ_L3CACHE, [cpu_index, &is8MB](hwloc_obj_t obj) {
#           if HWLOC_API_VERSION < 0x20000
            // 如果缓存深度不为 3，返回 false
            if (obj->attr->cache.depth != 3) {
                return false;
            }
# 检查是否定义了 endif，如果没有则执行下面的代码
            if (!hwloc_bitmap_isset(obj->cpuset, cpu_index)) {
                # 如果当前 CPU 不在对象的 cpuset 中，则返回 false
                return false;
            }

            # 初始化变量 num_cores 为 0
            uint32_t num_cores = 0;
            # 通过 findByType 函数查找 HWLOC_OBJ_CORE 类型的对象，并对 num_cores 进行计数
            findByType(obj, HWLOC_OBJ_CORE, [&num_cores](hwloc_obj_t) { ++num_cores; return false; });

            # 如果缓存大小右移 22 位后大于 num_cores，则执行下面的代码
            if ((obj->attr->cache.size >> 22) > num_cores) {
                # 计算 8MB 缓存的核心数
                uint32_t num_8MB_cores = (obj->attr->cache.size >> 22) - num_cores;

                # 通过 findByType 函数查找 HWLOC_OBJ_CORE 类型的对象，并对 num_8MB_cores 进行计数
                is8MB = findByType(obj, HWLOC_OBJ_CORE, [cpu_index, &num_8MB_cores](hwloc_obj_t obj2) {
                    # 如果 num_8MB_cores 大于 0，则执行下面的代码
                    if (num_8MB_cores > 0) {
                        # 减少 num_8MB_cores 的值
                        --num_8MB_cores;
                        # 如果当前对象的 cpuset 中包含 cpu_index，则返回 true
                        if (hwloc_bitmap_isset(obj2->cpuset, cpu_index)) {
                            return true;
                        }
                    }
                    return false;
                });
            }
            # 返回 true
            return true;
        });

#       if HWLOC_API_VERSION >= 0x20000
        # 遍历 HWLOC_OBJ_CORE、HWLOC_OBJ_L1CACHE、HWLOC_OBJ_L2CACHE、HWLOC_OBJ_L3CACHE 四种类型的对象
        for (auto obj_type : { HWLOC_OBJ_CORE, HWLOC_OBJ_L1CACHE, HWLOC_OBJ_L2CACHE, HWLOC_OBJ_L3CACHE }) {
#       else
        # 遍历 HWLOC_OBJ_CORE、HWLOC_OBJ_CACHE 两种类型的对象
        for (auto obj_type : { HWLOC_OBJ_CORE, HWLOC_OBJ_CACHE }) {
#       endif
            # 通过 findByType 函数查找指定类型的对象，并执行下面的代码
            findByType(root, obj_type, [cpu_index, helper_cpu_set, main_threads_set](hwloc_obj_t obj) {
                # 获取对象的 cpuset
                const hwloc_cpuset_t& s = obj->cpuset;
                # 如果当前 CPU 在对象的 cpuset 中，则执行下面的代码
                if (hwloc_bitmap_isset(s, cpu_index)) {
                    # 对 helper_cpu_set 和 main_threads_set 进行位操作，并判断结果是否大于 0
                    hwloc_bitmap_andnot(helper_cpu_set, s, main_threads_set);
                    if (hwloc_bitmap_weight(helper_cpu_set) > 0) {
                        return true;
                    }
                }
                return false;
            });

            # 如果 helper_cpu_set 的位数大于 0，则执行下面的代码
            if (hwloc_bitmap_weight(helper_cpu_set) > 0) {
                # 返回一个新的 HelperThread 对象
                return new HelperThread(helper_cpu_set, priority, is8MB);
            }
        }
    }
#endif

    # 返回空指针
    return nullptr;
}


# 销毁 HelperThread 对象
void destroy_helper_thread(HelperThread* t)
{
    delete t;
}


# 对数据进行哈希处理
void hash_octa(const uint8_t* data, size_t size, uint8_t* output, cryptonight_ctx** ctx, HelperThread* helper, bool verbose)
{
    // 定义枚举常量 N 的值为 8
    enum { N = 8 };

    // 创建指向 uint8_t 类型的指针数组，数组长度为 N
    uint8_t* ctx_memory[N];
    // 遍历 ctx 数组，将每个元素的 memory 属性赋值给 ctx_memory 数组对应位置的指针
    for (size_t i = 0; i < N; ++i) {
        ctx_memory[i] = ctx[i]->memory;
    }

    // 从数据中选择核心索引，存储在 core_indices 数组中
    uint32_t core_indices[15];
    select_indices(core_indices, data + 4);

    // 从数据中选择 cn 索引，存储在 cn_indices 数组中
    uint32_t cn_indices[6];
    select_indices(cn_indices, data + 4);

    // 如果 verbose 为真，则执行以下代码块
    if (verbose) {
        // 静态声明 prev_indices 数组，存储上一次的 cn 索引
        static uint32_t prev_indices[3];
        // 如果 cn_indices 与 prev_indices 不相等，则执行以下代码块
        if (memcmp(cn_indices, prev_indices, sizeof(prev_indices)) != 0) {
            // 将 cn_indices 复制给 prev_indices
            memcpy(prev_indices, cn_indices, sizeof(prev_indices));
            // 遍历输出 cn_indices 中的索引对应的 GhostRider 算法名称
            for (int i = 0; i < 3; ++i) {
                LOG_INFO("%s GhostRider algo %d: %s", Tags::cpu(), i + 1, cn_names[cn_indices[i]]);
            }
        }
    }

    // 根据 CPU 是否支持 AES，选择相应的算法变体
    const CnHash::AlgoVariant* av = Cpu::info()->hasAES() ? av_hw_aes : av_soft_aes;
    // 根据 helper 是否存在以及其属性，选择相应的算法调优
    const AlgoTune* tune = (helper && helper->m_is8MB) ? tune8MB : tuneDefault;

    // 创建临时数组 tmp，长度为 64*N
    uint8_t tmp[64 * N];

    // 如果 helper 存在且 cn_indices 中的前三个索引对应的线程数都为 2，则执行以下代码块
    if (helper && (tune[cn_indices[0]].threads == 2) && (tune[cn_indices[1]].threads == 2) && (tune[cn_indices[2]].threads == 2)) {
        // 定义常量 n 为 N 的一半
        constexpr size_t n = N / 2;
        // 启动任务，使用 av、data、size、ctx_memory、ctx、cn_indices、core_indices、tmp、output、tune 作为参数
        helper->launch_task([av, data, size, &ctx_memory, ctx, &cn_indices, &core_indices, &tmp, output, tune]() {
// 如果定义了 _MSC_VER，则将 N 除以 2，并赋值给 n
#ifdef _MSC_VER
constexpr size_t n = N / 2;
}

}

// 将 ctx_memory 数组中的每个元素赋值为对应 ctx 数组中元素的 memory 成员
for (size_t i = 0; i < N; ++i) {
    ctx[i]->memory = ctx_memory[i];
}
#else // XMRIG_FEATURE_HWLOC

// 如果未定义 _MSC_VER，则定义 benchmark 函数为空函数
void benchmark() {}
// 创建一个帮助线程，参数为 int64_t、int 和 std::vector<int64_t>，返回空指针
HelperThread* create_helper_thread(int64_t, int, const std::vector<int64_t>&) { return nullptr; }
// 销毁帮助线程，参数为 HelperThread 指针，返回空指针
void destroy_helper_thread(HelperThread*) {}

// 对输入数据进行哈希计算，输出结果到 output，使用 ctx 数组中的上下文，不使用帮助线程，verbose 为是否冗长输出
void hash_octa(const uint8_t* data, size_t size, uint8_t* output, cryptonight_ctx** ctx, HelperThread*, bool verbose)
{
    // 定义常量 N 为 8
    constexpr uint32_t N = 8;

    // 定义长度为 N 的 ctx_memory 数组，将 ctx 数组中每个元素的 memory 成员赋值给对应的 ctx_memory 元素
    uint8_t* ctx_memory[N];
    for (size_t i = 0; i < N; ++i) {
        ctx_memory[i] = ctx[i]->memory;
    }

    // 从输入数据中提取 PrevBlockHash（GhostRider 的种子），存储在 seed 中
    const uint8_t* seed = data + 4;

    // 定义长度为 15 的 core_indices 数组，并根据 seed 选择索引值
    uint32_t core_indices[15];
    select_indices(core_indices, seed);

    // 定义长度为 6 的 cn_indices 数组，并根据 seed 选择索引值
    uint32_t cn_indices[6];
    select_indices(cn_indices, seed);

    // 根据平台选择步长数组 step
#ifdef XMRIG_ARM
    uint32_t step[6] = { 1, 1, 1, 1, 1, 1 };
#else
    uint32_t step[6] = { 4, 4, 1, 2, 4, 4 };
#endif

    // 如果 verbose 为真，则输出 GhostRider 算法的信息
    if (verbose) {
        static uint32_t prev_indices[3];
        if (memcmp(cn_indices, prev_indices, sizeof(prev_indices)) != 0) {
            memcpy(prev_indices, cn_indices, sizeof(prev_indices));
            for (int i = 0; i < 3; ++i) {
                LOG_INFO("%s GhostRider algo %d: %s", Tags::cpu(), i + 1, cn_names[cn_indices[i]]);
            }
        }
    }

    // 根据 CPU 支持的情况选择 AES 算法变体
    const CnHash::AlgoVariant* av = Cpu::info()->hasAES() ? av_hw_aes : av_soft_aes;

    // 定义长度为 64*N 的临时数组 tmp
    uint8_t tmp[64 * N];
}
    // 遍历3个部分的循环
    for (size_t part = 0; part < 3; ++part) {

        // 分配临时内存空间
        {
            // 从上下文内存中获取指针
            uint8_t* p = ctx_memory[0];

            // 遍历N个元素
            for (size_t i = 0, k = 0; i < N; ++i) {
                // 如果i是step[cn_indices[part]]的倍数
                if ((i % step[cn_indices[part]]) == 0) {
                    k = 0;
                    p = ctx_memory[0];
                }
                // 如果p - ctx_memory[k]大于等于2的21次方
                else if (p - ctx_memory[k] >= (1 << 21)) {
                    ++k;
                    p = ctx_memory[k];
                }
                // 设置ctx[i]的内存指针为p
                ctx[i]->memory = p;
                // p指针向后移动cn_sizes[cn_indices[part]]个字节
                p += cn_sizes[cn_indices[part]];
            }
        }

        // 循环5次
        for (size_t i = 0; i < 5; ++i) {
            // 循环N次
            for (size_t j = 0; j < N; ++j) {
                // 调用core_hash[core_indices[part * 5 + i]]函数
                core_hash[core_indices[part * 5 + i]](data + j * size, size, tmp + j * 64);
            }
            // 将data指向tmp
            data = tmp;
            // 将size设置为64
            size = 64;
        }

        // 调用CnHash::fn函数
        auto f = CnHash::fn(cn_hash[cn_indices[part]], av[step[cn_indices[part]]], Assembly::AUTO);
        // 循环N/step[cn_indices[part]]次
        for (size_t j = 0; j < N; j += step[cn_indices[part]]) {
            // 调用f函数
            f(tmp + j * 64, 64, output + j * 32, ctx, 0);
        }

        // 循环N次
        for (size_t j = 0; j < N; ++j) {
            // 将output + j * 32的32个字节复制到tmp + j * 64
            memcpy(tmp + j * 64, output + j * 32, 32);
            // 将tmp + j * 64 + 32的32个字节设置为0
            memset(tmp + j * 64 + 32, 0, 32);
        }
    }

    // 循环N次
    for (size_t i = 0; i < N; ++i) {
        // 设置ctx[i]的内存指针为ctx_memory[i]
        ctx[i]->memory = ctx_memory[i];
    }
} // 结束 #ifdef XMRIG_FEATURE_HWLOC

} // 结束命名空间 ghostrider

} // 结束命名空间 xmrig
```