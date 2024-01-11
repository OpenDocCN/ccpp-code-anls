# `xmrig\src\backend\cpu\platform\HwlocCpuInfo.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifdef XMRIG_HWLOC_DEBUG
#   include <uv.h>
#endif


#include <algorithm>
#include <cmath>
#include <hwloc.h>


#if HWLOC_API_VERSION < 0x00010b00
#   define HWLOC_OBJ_PACKAGE HWLOC_OBJ_SOCKET
#   define HWLOC_OBJ_NUMANODE HWLOC_OBJ_NODE
#endif


#include "backend/cpu/platform/HwlocCpuInfo.h"
#include "base/io/log/Log.h"


#if HWLOC_API_VERSION < 0x20000
static inline int hwloc_obj_type_is_cache(hwloc_obj_type_t type)
{
    return type == HWLOC_OBJ_CACHE;
}
#endif


namespace xmrig {


template <typename func>
static inline void findCache(hwloc_obj_t obj, unsigned min, unsigned max, func lambda)
{
    for (size_t i = 0; i < obj->arity; i++) {
        if (hwloc_obj_type_is_cache(obj->children[i]->type)) {
            const unsigned depth = obj->children[i]->attr->cache.depth;
            if (depth < min || depth > max) {
                continue;
            }

            lambda(obj->children[i]);
        }

        findCache(obj->children[i], min, max, lambda);
    }
}


template <typename func>
static inline void findByType(hwloc_obj_t obj, hwloc_obj_type_t type, func lambda)
{
    # 遍历对象的子元素
    for (size_t i = 0; i < obj->arity; i++) {
        # 检查子元素的类型是否与指定类型相同
        if (obj->children[i]->type == type) {
            # 如果相同，调用 lambda 函数处理该子元素
            lambda(obj->children[i]);
        }
        # 如果类型不相同
        else {
            # 递归调用自身，查找符合指定类型的子元素
            findByType(obj->children[i], type, lambda);
        }
    }
} // 结束命名空间 xmrig

// 根据给定的拓扑和对象类型计算对象数量
static inline size_t countByType(hwloc_topology_t topology, hwloc_obj_type_t type)
{
    // 调用 hwloc_get_nbobjs_by_type 函数获取指定类型的对象数量
    const int count = hwloc_get_nbobjs_by_type(topology, type);
    // 如果数量大于0，则返回数量，否则返回0
    return count > 0 ? static_cast<size_t>(count) : 0;
}

// 根据给定的对象和类型查找对象
static inline std::vector<hwloc_obj_t> findByType(hwloc_obj_t obj, hwloc_obj_type_t type)
{
    // 创建一个空的对象向量
    std::vector<hwloc_obj_t> out;
    // 调用 findByType 函数查找指定类型的对象，并将其添加到向量中
    findByType(obj, type, [&out](hwloc_obj_t found) { out.emplace_back(found); });
    // 返回包含找到的对象的向量
    return out;
}

// 根据给定的对象和类型计算对象数量
static inline size_t countByType(hwloc_obj_t obj, hwloc_obj_type_t type)
{
    // 初始化对象数量为0
    size_t count = 0;
    // 调用 findByType 函数查找指定类型的对象，并递增数量
    findByType(obj, type, [&count](hwloc_obj_t) { count++; });
    // 返回对象数量
    return count;
}

// 检查对象是否具有排他性缓存
static inline bool isCacheExclusive(hwloc_obj_t obj)
{
    // 获取对象的 "Inclusive" 信息
    const char *value = hwloc_obj_get_info_by_name(obj, "Inclusive");
    // 如果信息为空或者第一个字符不是 '1'，则返回 true，否则返回 false
    return value == nullptr || value[0] != '1';
}

#endif // 结束条件编译指令

// 构造函数 HwlocCpuInfo 的实现
xmrig::HwlocCpuInfo::HwlocCpuInfo()
{
    // 初始化拓扑对象
    hwloc_topology_init(&m_topology);
    // 加载拓扑
    hwloc_topology_load(m_topology);

#   ifdef XMRIG_HWLOC_DEBUG
#   if defined(UV_VERSION_HEX) && UV_VERSION_HEX >= 0x010c00
    {
        // 创建一个大小为520的环境变量数组
        char env[520] = { 0 };
        size_t size   = sizeof(env);
        // 如果能够获取到环境变量 "HWLOC_XMLFILE"，则打印使用的 HWLOC XML 文件
        if (uv_os_getenv("HWLOC_XMLFILE", env, &size) == 0) {
            printf("use HWLOC XML file: \"%s\"\n", env);
        }
    }
#   endif

    // 查找根对象下的所有包对象
    const std::vector<hwloc_obj_t> packages = findByType(hwloc_get_root_obj(m_topology), HWLOC_OBJ_PACKAGE);
    // 如果包对象不为空，则获取第一个包对象的 CPUModel 信息并复制到 m_brand 中
    if (!packages.empty()) {
        const char *value = hwloc_obj_get_info_by_name(packages[0], "CPUModel");
        if (value) {
            strncpy(m_brand, value, 64);
        }
    }
#   endif

    // 获取根对象
    hwloc_obj_t root = hwloc_get_root_obj(m_topology);

#   if HWLOC_API_VERSION >= 0x00010b00
    // 获取根对象的 hwlocVersion 信息，并将其格式化到 m_backend 中
    const char *version = hwloc_obj_get_info_by_name(root, "hwlocVersion");
    if (version) {
        snprintf(m_backend, sizeof m_backend, "hwloc/%s", version);
    }
    else
#   endif
    {
        // 使用 snprintf 格式化字符串，将 hwloc 版本信息写入 m_backend 中
        snprintf(m_backend, sizeof m_backend, "hwloc/%d.%d.%d",
                       (HWLOC_API_VERSION>>16)&0x000000ff,
                       (HWLOC_API_VERSION>>8 )&0x000000ff,
                       (HWLOC_API_VERSION    )&0x000000ff
               );
    }

    // 在根节点下查找缓存，深度为2，类型为3，使用 lambda 表达式更新 m_cache
    findCache(root, 2, 3, [this](hwloc_obj_t found) { this->m_cache[found->attr->cache.depth] += found->attr->cache.size; });

    // 设置线程数为 CPU 核心的数量
    setThreads(countByType(m_topology, HWLOC_OBJ_PU));

    // 统计 CPU 核心的数量，存储在 m_cores 中
    m_cores     = countByType(m_topology, HWLOC_OBJ_CORE);
    // 统计 NUMA 节点的数量，如果数量小于1，则设置为1
    m_nodes     = std::max(hwloc_bitmap_weight(hwloc_topology_get_complete_nodeset(m_topology)), 1);
    // 统计 CPU 包的数量，存储在 m_packages 中
    m_packages  = countByType(m_topology, HWLOC_OBJ_PACKAGE);

    // 如果 NUMA 节点数量大于1，则初始化 m_nodeset，并遍历 NUMA 节点，将其索引存储在 m_nodeset 中
    if (m_nodes > 1) {
        m_nodeset.reserve(m_nodes);
        hwloc_obj_t node = nullptr;

        while ((node = hwloc_get_next_obj_by_type(m_topology, HWLOC_OBJ_NUMANODE, node)) != nullptr) {
            m_nodeset.emplace_back(node->os_index);
        }
    }
#   if defined(XMRIG_OS_MACOS) && defined(XMRIG_ARM)
    # 如果定义了 XMRIG_OS_MACOS 和 XMRIG_ARM，则执行以下代码块
    if (L2() == 33554432U && m_cores == 8 && m_cores == m_threads) {
        # 如果 L2 缓存大小为 33554432U，核心数为 8，并且核心数等于线程数，则执行以下代码
        m_cache[2] = 16777216U;
        # 将 m_cache 数组的第三个元素设置为 16777216U
    }
#   endif
}
# 结束 if 语句块

xmrig::HwlocCpuInfo::~HwlocCpuInfo()
{
    # 析构函数，销毁 HwlocCpuInfo 对象时执行以下代码
    hwloc_topology_destroy(m_topology);
    # 销毁 m_topology 对象
}

bool xmrig::HwlocCpuInfo::membind(hwloc_const_bitmap_t nodeset)
{
    # 设置内存绑定，传入节点集合参数
    if (!hwloc_topology_get_support(m_topology)->membind->set_thisthread_membind) {
        # 如果当前系统不支持设置本线程的内存绑定，则返回 false
        return false;
    }

#   if HWLOC_API_VERSION >= 0x20000
    # 如果 HWLOC_API_VERSION 大于等于 0x20000，则执行以下代码块
    return hwloc_set_membind(m_topology, nodeset, HWLOC_MEMBIND_BIND, HWLOC_MEMBIND_THREAD | HWLOC_MEMBIND_BYNODESET) >= 0;
    # 设置内存绑定，返回设置结果
#   else
    # 如果 HWLOC_API_VERSION 小于 0x20000，则执行以下代码块
    return hwloc_set_membind_nodeset(m_topology, nodeset, HWLOC_MEMBIND_BIND, HWLOC_MEMBIND_THREAD) >= 0;
    # 设置节点集合的内存绑定，返回设置结果
#   endif
}

xmrig::CpuThreads xmrig::HwlocCpuInfo::threads(const Algorithm &algorithm, uint32_t limit) const
{
#   ifndef XMRIG_ARM
    # 如果未定义 XMRIG_ARM，则执行以下代码块
    if (L2() == 0 && L3() == 0) {
        # 如果 L2 缓存大小和 L3 缓存大小都为 0，则执行以下代码
        return BasicCpuInfo::threads(algorithm, limit);
        # 返回 BasicCpuInfo 类的 threads 方法的结果
    }

    const unsigned depth = L3() > 0 ? 3 : 2;
    # 根据 L3 缓存大小是否大于 0，确定深度为 3 或 2

    CpuThreads threads;
    # 创建 CpuThreads 对象
    threads.reserve(m_threads);
    # 预留 m_threads 大小的空间

    std::vector<hwloc_obj_t> caches;
    # 创建 hwloc_obj_t 类型的向量 caches
    caches.reserve(16);
    # 预留 16 个空间

    findCache(hwloc_get_root_obj(m_topology), depth, depth, [&caches](hwloc_obj_t found) { caches.emplace_back(found); });
    # 查找缓存，将找到的缓存对象添加到 caches 中

    if (limit > 0 && limit < 100 && !caches.empty()) {
        # 如果限制大于 0 且小于 100 且 caches 不为空，则执行以下代码块
        const double maxTotalThreads = round(m_threads * (limit / 100.0));
        # 计算最大总线程数
        const auto maxPerCache       = std::max(static_cast<int>(round(maxTotalThreads / caches.size())), 1);
        # 计算每个缓存的最大线程数
        int remaining                = std::max(static_cast<int>(maxTotalThreads), 1);
        # 计算剩余线程数

        for (hwloc_obj_t cache : caches) {
            # 遍历缓存对象
            processTopLevelCache(cache, algorithm, threads, std::min(maxPerCache, remaining));
            # 处理顶层缓存
            remaining -= maxPerCache;
            # 减去每个缓存的最大线程数
            if (remaining <= 0) {
                # 如果剩余线程数小于等于 0，则跳出循环
                break;
            }
        }
    }
    else {
        # 如果不满足上述条件，则执行以下代码块
        for (hwloc_obj_t cache : caches) {
            # 遍历缓存对象
            processTopLevelCache(cache, algorithm, threads, 0);
            # 处理顶层缓存
        }
    }
}
    # 如果线程列表为空
    if (threads.isEmpty()) {
        # 记录警告日志，指出算法“algorithm.name()”的硬件位置自动配置失败
        LOG_WARN("hwloc auto configuration for algorithm \"%s\" failed.", algorithm.name());
        # 返回使用BasicCpuInfo类的threads方法获取的线程列表
        return BasicCpuInfo::threads(algorithm, limit);
    }
    # 返回线程列表
    return threads;
# 如果条件不满足，则返回使用指定算法和限制的所有线程
    return allThreads(algorithm, limit);
# endif
}


# 返回所有线程
xmrig::CpuThreads xmrig::HwlocCpuInfo::allThreads(const Algorithm &algorithm, uint32_t limit) const
{
    # 为线程预留空间
    CpuThreads threads;
    threads.reserve(m_threads)

    # 根据算法家族确定强度
    const uint32_t intensity = (algorithm.family() == Algorithm::GHOSTRIDER) ? 8 : 0;

    # 遍历单元并添加线程
    for (const int32_t pu : m_units) {
        threads.add(pu, intensity);
    }

    # 如果线程为空，则返回基本CPU信息的线程
    if (threads.isEmpty()) {
        return BasicCpuInfo::threads(algorithm, limit);
    }

    # 返回线程
    return threads;
}



# 处理顶层缓存
void xmrig::HwlocCpuInfo::processTopLevelCache(hwloc_obj_t cache, const Algorithm &algorithm, CpuThreads &threads, size_t limit) const
{
# ifndef XMRIG_ARM
    # 定义1MB
    constexpr size_t oneMiB = 1024U * 1024U;

    # 计算PU的数量，如果为0则返回
    size_t PUs = countByType(cache, HWLOC_OBJ_PU);
    if (PUs == 0) {
        return;
    }

    # 创建核心向量并根据类型查找核心
    std::vector<hwloc_obj_t> cores;
    cores.reserve(m_cores);
    findByType(cache, HWLOC_OBJ_CORE, [&cores](hwloc_obj_t found) { cores.emplace_back(found); });

    # 判断L3缓存是否独占
    const bool L3_exclusive = isCacheExclusive(cache);

# ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果算法为GHOSTRIDER_RTM，并且L3缓存独占，并且PU数量大于核心数量，并且PU数量小于核心数量的两倍
    if ((algorithm == Algorithm::GHOSTRIDER_RTM) && L3_exclusive && (PUs > cores.size()) && (PUs < cores.size() * 2)) {
        # 在Alder Lake上不使用E-cores
        cores.erase(std::remove_if(cores.begin(), cores.end(), [](hwloc_obj_t c) { return hwloc_bitmap_weight(c->cpuset) == 1; }), cores.end());

        # 如果核心为空，则重新查找核心
        if (cores.empty()) {
            findByType(cache, HWLOC_OBJ_CORE, [&cores](hwloc_obj_t found) { cores.emplace_back(found); });
        }
    }
# endif

    # 获取L3和L2缓存的大小和关联性
    size_t L3               = cache->attr->cache.size;
    size_t L2               = 0;
    int L2_associativity    = 0;
    size_t extra            = 0;
    size_t scratchpad       = algorithm.l3();
    uint32_t intensity      = algorithm.maxIntensity() == 1 ? 0 : 1;
    # 如果缓存深度为3
    if (cache->attr->cache.depth == 3) {
        # 遍历缓存的子节点
        for (size_t i = 0; i < cache->arity; ++i) {
            # 获取第二级缓存对象
            hwloc_obj_t l2 = cache->children[i];
            # 如果不是缓存类型或者属性为空，则跳过当前循环
            if (!hwloc_obj_type_is_cache(l2->type) || l2->attr == nullptr) {
                continue;
            }

            # 累加第二级缓存大小
            L2 += l2->attr->cache.size;
            # 获取第二级缓存的关联度
            L2_associativity = l2->attr->cache.associativity;

            # 如果L3是独占的并且第二级缓存大小大于等于scratchpad
            if (L3_exclusive && l2->attr->cache.size >= scratchpad) {
                # 累加额外的scratchpad大小
                extra += scratchpad;
            }
        }
    }

    # 如果scratchpad等于2 * oneMiB
    if (scratchpad == 2 * oneMiB) {
        # 如果L2存在且核心数乘以oneMiB等于L2并且L2的关联度为16并且L3大于等于L2
        if (L2 && (cores.size() * oneMiB) == L2 && L2_associativity == 16 && L3 >= L2) {
            # 将L3设置为L2
            L3    = L2;
            # 将extra设置为L2
            extra = L2;
        }
    }

    # 计算缓存哈希值
    size_t cacheHashes = ((L3 + extra) + (scratchpad / 2)) / scratchpad;

    # 获取算法的族
    const auto family = algorithm.family();
    # 如果强度大于0并且算法族为CN_PICO或CN_FEMTO并且缓存哈希值除以处理器数大于等于2
    if (intensity && ((family == Algorithm::CN_PICO) || (family == Algorithm::CN_FEMTO)) && (cacheHashes / PUs) >= 2) {
        # 将强度设置为2
        intensity = 2;
    }
#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX
    if ((algorithm.family() == Algorithm::RANDOM_X) && L3_exclusive && (PUs > cores.size()) && (PUs < cores.size() * 2)) {
        # 如果算法家族是 RANDOM_X，并且 L3_exclusive 为真，并且 PUs 大于核心数，并且 PUs 小于核心数的两倍
        # 使用最新的英特尔 CPU 上的所有 L3+L2，P-cores、E-cores 和独占 L3 缓存
        cacheHashes = (L3 + L2) / scratchpad;
    }
    if (extra == 0 && algorithm.l2() > 0) {
        # 如果额外值为 0，并且算法的 L2 大于 0
        cacheHashes = std::min<size_t>(std::max<size_t>(L2 / algorithm.l2(), cores.size()), cacheHashes);
    }
#   endif

    if (limit > 0) {
        # 如果限制大于 0
        cacheHashes = std::min(cacheHashes, limit);
    }

#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果定义了 XMRIG_ALGO_GHOSTRIDER，并且算法是 GHOSTRIDER_RTM
    if (algorithm == Algorithm::GHOSTRIDER_RTM) {
        # GhostRider 实现一次运行 8 个哈希
        intensity = 8;
        # 每个核心始终使用 1 个线程（在可能的情况下使用额外的辅助线程）
        cacheHashes = std::min(cacheHashes, cores.size());
    }
#   endif

    if (cacheHashes >= PUs) {
        # 如果缓存哈希数大于等于处理器单元数
        for (hwloc_obj_t core : cores) {
            const std::vector<hwloc_obj_t> units = findByType(core, HWLOC_OBJ_PU);
            for (hwloc_obj_t pu : units) {
                threads.add(pu->os_index, intensity);
            }
        }
        return;
    }

    # 创建一个存储线程数据的向量，并预留空间
    std::vector<std::pair<int64_t, int32_t>> threads_data;
    threads_data.reserve(cores.size());

    # 初始化处理器单元 ID
    size_t pu_id = 0;
    # 当缓存哈希数大于0且处理器单元数大于0时执行循环
    while (cacheHashes > 0 && PUs > 0) {
        # 初始化标记，表示是否已分配处理器单元
        bool allocated_pu = false;

        # 清空线程数据
        threads_data.clear();
        # 遍历核心列表
        for (hwloc_obj_t core : cores) {
            # 查找当前核心下的处理器单元
            const std::vector<hwloc_obj_t> units = findByType(core, HWLOC_OBJ_PU);
            # 如果处理器单元数量小于等于pu_id，则跳过当前核心
            if (units.size() <= pu_id) {
                continue;
            }

            # 缓存哈希数和处理器单元数减一
            cacheHashes--;
            PUs--;

            # 设置已分配处理器单元的标记为true
            allocated_pu = true;
            # 将处理器单元的os_index和intensity添加到线程数据中
            threads_data.emplace_back(units[pu_id]->os_index, intensity);

            # 如果缓存哈希数为0，则跳出循环
            if (cacheHashes == 0) {
                break;
            }
        }

        # 对"threads_data"和"cores"进行反转，以便从最后一个虚拟核心开始填充，但仍然按顺序
        # 例如，在6核Zen2/Zen3上的cn-heavy线程将具有亲和性[0,2,4,6,8,10,9,11]
        # 这对于Zen3 cn-heavy优化很重要
        if (pu_id & 1) {
            std::reverse(threads_data.begin(), threads_data.end());
        }

        # 将线程数据中的处理器单元添加到线程中
        for (const auto& t : threads_data) {
            threads.add(t.first, t.second);
        }

        # 如果未分配处理器单元，则跳出循环
        if (!allocated_pu) {
            break;
        }

        # 处理器单元id加一
        pu_id++;
        # 反转核心列表
        std::reverse(cores.begin(), cores.end());
    }
// 结束 if 语句
}

// 设置线程数
void xmrig::HwlocCpuInfo::setThreads(size_t threads)
{
    // 如果线程数为 0，则直接返回
    if (!threads) {
        return;
    }

    // 将线程数赋给成员变量 m_threads
    m_threads = threads;

    // 如果 m_units 的大小不等于线程数，则重新调整 m_units 的大小为线程数
    if (m_units.size() != m_threads) {
        m_units.resize(m_threads);
    }

    // 初始化变量 pu 和 i
    hwloc_obj_t pu = nullptr;
    size_t i       = 0;

    // 遍历获取每个处理单元的 os_index，并存储到 m_units 中
    while ((pu = hwloc_get_next_obj_by_type(m_topology, HWLOC_OBJ_PU, pu)) != nullptr) {
        m_units[i++] = static_cast<int32_t>(pu->os_index);
    }
}
```