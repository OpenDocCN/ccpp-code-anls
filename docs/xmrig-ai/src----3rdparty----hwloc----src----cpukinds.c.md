# `xmrig\src\3rdparty\hwloc\src\cpukinds.c`

```cpp
/*
 * 版权所有 © 2020-2022 Inria。
 * 保留所有权利。请参阅顶层目录中的COPYING文件。
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"


/*****************
 * Basics
 */

// 初始化拓扑结构中的 cpukinds 相关字段
void
hwloc_internal_cpukinds_init(struct hwloc_topology *topology)
{
  topology->cpukinds = NULL; // 将 cpukinds 字段置为空
  topology->nr_cpukinds = 0; // 将 nr_cpukinds 字段置为0
  topology->nr_cpukinds_allocated = 0; // 将 nr_cpukinds_allocated 字段置为0
}

// 销毁拓扑结构中的 cpukinds 相关字段
void
hwloc_internal_cpukinds_destroy(struct hwloc_topology *topology)
{
  unsigned i;
  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    hwloc_bitmap_free(kind->cpuset); // 释放 cpuset 字段所指向的位图
    hwloc__free_infos(kind->infos, kind->nr_infos); // 释放 infos 字段所指向的信息数组
  }
  free(topology->cpukinds); // 释放 cpukinds 字段所指向的内存
  topology->cpukinds = NULL; // 将 cpukinds 字段置为空
  topology->nr_cpukinds = 0; // 将 nr_cpukinds 字段置为0
}

// 复制拓扑结构中的 cpukinds 相关字段
int
hwloc_internal_cpukinds_dup(hwloc_topology_t new, hwloc_topology_t old)
{
  struct hwloc_tma *tma = new->tma; // 获取新拓扑结构中的 tma 字段
  struct hwloc_internal_cpukind_s *kinds; // 定义 cpukind 结构体指针
  unsigned i;

  if (!old->nr_cpukinds) // 如果旧拓扑结构中的 nr_cpukinds 字段为0，则直接返回
    return 0;

  kinds = hwloc_tma_malloc(tma, old->nr_cpukinds * sizeof(*kinds)); // 分配新拓扑结构中的 cpukinds 字段内存
  if (!kinds) // 如果分配失败，则返回-1
    return -1;
  new->cpukinds = kinds; // 将新拓扑结构中的 cpukinds 字段指向新分配的内存
  new->nr_cpukinds = old->nr_cpukinds; // 将新拓扑结构中的 nr_cpukinds 字段设置为旧拓扑结构中的值
  memcpy(kinds, old->cpukinds, old->nr_cpukinds * sizeof(*kinds)); // 复制旧拓扑结构中的 cpukinds 字段内容到新拓扑结构中

  for(i=0;i<old->nr_cpukinds; i++) {
    kinds[i].cpuset = hwloc_bitmap_tma_dup(tma, old->cpukinds[i].cpuset); // 复制旧拓扑结构中的 cpuset 字段内容到新拓扑结构中
    if (!kinds[i].cpuset) { // 如果复制失败，则释放已分配的内存并返回-1
      new->nr_cpukinds = i;
      goto failed;
    }
    if (hwloc__tma_dup_infos(tma,
                             &kinds[i].infos, &kinds[i].nr_infos,
                             old->cpukinds[i].infos, old->cpukinds[i].nr_infos) < 0) {
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      hwloc_bitmap_free(kinds[i].cpuset); // 释放已分配的内存
      new->nr_cpukinds = i;
      goto failed;
    }
  }

  return 0;

 failed:
  hwloc_internal_cpukinds_destroy(new); // 失败时销毁新拓扑结构中的 cpukinds 相关字段
  return -1;
}

// 限制拓扑结构中的 cpukinds 相关字段
void
hwloc_internal_cpukinds_restrict(hwloc_topology_t topology)
{
  // 声明一个无符号整数变量 i
  unsigned i;
  // 声明一个整数变量 removed，并初始化为 0
  int removed = 0;
  // 循环遍历 topology->nr_cpukinds 次
  for(i=0; i<topology->nr_cpukinds; i++) {
    // 获取 topology->cpukinds[i] 的地址，并赋值给指针 kind
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    // 对 kind->cpuset 和 topology 的根对象的 cpuset 进行按位与操作
    hwloc_bitmap_and(kind->cpuset, kind->cpuset, hwloc_get_root_obj(topology)->cpuset);
    // 如果 kind->cpuset 全为 0
    if (hwloc_bitmap_iszero(kind->cpuset)) {
      // 释放 kind->cpuset
      hwloc_bitmap_free(kind->cpuset);
      // 释放 kind->infos，并将后面的元素向前移动
      hwloc__free_infos(kind->infos, kind->nr_infos);
      memmove(kind, kind+1, (topology->nr_cpukinds - i - 1)*sizeof(*kind));
      // 减小 i 和 topology->nr_cpukinds
      i--;
      topology->nr_cpukinds--;
      // 设置 removed 为 1
      removed = 1;
    }
  }
  // 如果有移除操作，则重新排列 cpukinds
  if (removed)
    hwloc_internal_cpukinds_rank(topology);
}

/********************
 * Registering
 */

// 检查是否有重复的信息
static __hwloc_inline int
hwloc__cpukind_check_duplicate_info(struct hwloc_internal_cpukind_s *kind,
                                    const char *name, const char *value)
{
  unsigned i;
  // 遍历 kind->nr_infos 次
  for(i=0; i<kind->nr_infos; i++)
    // 如果 name 和 value 都相同，则返回 1
    if (!strcmp(kind->infos[i].name, name)
        && !strcmp(kind->infos[i].value, value))
      return 1;
  // 否则返回 0
  return 0;
}

// 添加信息到 cpukind
static __hwloc_inline void
hwloc__cpukind_add_infos(struct hwloc_internal_cpukind_s *kind,
                         const struct hwloc_info_s *infos, unsigned nr_infos)
{
  unsigned i;
  // 遍历 nr_infos 次
  for(i=0; i<nr_infos; i++) {
    // 如果已经存在相同的信息，则跳过
    if (hwloc__cpukind_check_duplicate_info(kind, infos[i].name, infos[i].value))
      continue;
    // 否则添加信息到 cpukind
    hwloc__add_info(&kind->infos, &kind->nr_infos, infos[i].name, infos[i].value);
  }
}

// 注册 cpukind
int
hwloc_internal_cpukinds_register(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
                                 int forced_efficiency,
                                 const struct hwloc_info_s *infos, unsigned nr_infos,
                                 unsigned long flags)
{
  struct hwloc_internal_cpukind_s *kinds;
  unsigned i, max, bits, oldnr, newnr;

  // 如果 cpuset 全为 0，则释放 cpuset，并返回错误
  if (hwloc_bitmap_iszero(cpuset)) {
    hwloc_bitmap_free(cpuset);
    errno = EINVAL;
    return -1;
  }

  // 如果 flags 中包含未知的标志位，则返回错误
  if (flags & ~HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY) {
    errno = EINVAL;
  // 返回错误代码 -1
  return -1;
}

/* TODO: for now, only windows provides a forced efficiency.
 * if another backend ever provides a conflicting value, the first backend value will be kept.
 * (user-provided values are not an issue, they are meant to overwrite)
 */
/* 目前只有 Windows 提供了强制效率。
 * 如果另一个后端提供了冲突的值，将保留第一个后端的值。
 * （用户提供的值不是问题，它们意味着要覆盖）
 */

/* 如果我们当前有 N 种类型，插入新类型后可能需要 2N+1 种类型：
 * - 每种现有类型可能会分成哪些处理器在新类型中，哪些不在。
 * - 一些处理器可能还没有分配到任何类型。
 */
max = 2 * topology->nr_cpukinds + 1;
/* 分配大于 2N+1 的最小的 2 的幂。 */
bits = hwloc_flsl(max-1) + 1;
max = 1U<<bits;
/* 分配至少 8 个以避免多次重新分配 */
if (max < 8)
  max = 8;

/* 如果需要，创建或扩大类型数组 */
kinds = topology->cpukinds;
if (max > topology->nr_cpukinds_allocated) {
  kinds = realloc(kinds, max * sizeof(*kinds));
  if (!kinds) {
    hwloc_bitmap_free(cpuset);
    return -1;
  }
  memset(&kinds[topology->nr_cpukinds_allocated], 0, (max - topology->nr_cpukinds_allocated) * sizeof(*kinds));
  topology->nr_cpukinds_allocated = max;
  topology->cpukinds = kinds;
}

newnr = oldnr = topology->nr_cpukinds;
for(i=0; i<oldnr; i++) {
  int res = hwloc_bitmap_compare_inclusion(cpuset, kinds[i].cpuset);
    if (res == HWLOC_BITMAP_INTERSECTS || res == HWLOC_BITMAP_INCLUDED) {
      /* 如果结果是相交或者包含关系，则创建新的种类，其 cpuset 是两者的交集，效率未知，强制效率为指定值 */
      kinds[newnr].cpuset = hwloc_bitmap_alloc();
      kinds[newnr].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
      kinds[newnr].forced_efficiency = forced_efficiency;
      hwloc_bitmap_and(kinds[newnr].cpuset, cpuset, kinds[i].cpuset);
      hwloc__cpukind_add_infos(&kinds[newnr], kinds[i].infos, kinds[i].nr_infos);
      hwloc__cpukind_add_infos(&kinds[newnr], infos, nr_infos);
      /* 从刚刚分割的现有种类中移除 cpuset PUs */
      hwloc_bitmap_andnot(kinds[i].cpuset, kinds[i].cpuset, kinds[newnr].cpuset);
      /* 清除已处理的 cpuset PUs */
      hwloc_bitmap_andnot(cpuset, cpuset, kinds[newnr].cpuset);

      newnr++;

    } else if (res == HWLOC_BITMAP_CONTAINS
               || res == HWLOC_BITMAP_EQUAL) {
      /* 将新的信息追加到现有较小（或相等）的种类中 */
      hwloc__cpukind_add_infos(&kinds[i], infos, nr_infos);
      if ((flags & HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY)
          || kinds[i].forced_efficiency == HWLOC_CPUKIND_EFFICIENCY_UNKNOWN)
        kinds[i].forced_efficiency = forced_efficiency;
      /* 清除已处理的 cpuset PUs */
      hwloc_bitmap_andnot(cpuset, cpuset, kinds[i].cpuset);

    } else {
      assert(res == HWLOC_BITMAP_DIFFERENT);
      /* 无需执行任何操作 */
    }

    /* 如果 cpuset 已经为空，则不再与其他内容比较 */
    if (hwloc_bitmap_iszero(cpuset))
      break;
  }

  /* 如果还有剩余的 PUs，则添加一个最终的种类 */
  if (!hwloc_bitmap_iszero(cpuset)) {
    kinds[newnr].cpuset = cpuset;
    kinds[newnr].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
    kinds[newnr].forced_efficiency = forced_efficiency;
    hwloc__cpukind_add_infos(&kinds[newnr], infos, nr_infos);
    newnr++;
  } else {
    hwloc_bitmap_free(cpuset);
  }

  topology->nr_cpukinds = newnr;
  return 0;
}

int
hwloc_cpukinds_register(hwloc_topology_t topology, hwloc_cpuset_t _cpuset,
                        int forced_efficiency,
                        unsigned nr_infos, struct hwloc_info_s *infos,
                        unsigned long flags)
{
  hwloc_bitmap_t cpuset; // 声明一个位图变量
  int err; // 声明一个整型变量用于存储错误码

  if (flags) { // 如果标志位不为0
    errno = EINVAL; // 设置错误码为无效参数
    return -1; // 返回错误
  }

  if (!_cpuset || hwloc_bitmap_iszero(_cpuset)) { // 如果输入的位图为空
    errno = EINVAL; // 设置错误码为无效参数
    return -1; // 返回错误
  }

  cpuset = hwloc_bitmap_dup(_cpuset); // 复制输入的位图
  if (!cpuset) // 如果复制失败
    return -1; // 返回错误

  if (forced_efficiency < 0) // 如果强制效率小于0
    forced_efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN; // 设置为未知效率

  err = hwloc_internal_cpukinds_register(topology, cpuset, forced_efficiency, infos, nr_infos, HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY); // 调用内部函数进行 CPU 种类注册
  if (err < 0) // 如果注册失败
    return err; // 返回错误

  hwloc_internal_cpukinds_rank(topology); // 对 CPU 种类进行排序
  return 0; // 返回成功
}


/*********************
 * Ranking
 */

static int
hwloc__cpukinds_check_duplicate_rankings(struct hwloc_topology *topology)
{
  unsigned i,j;
  for(i=0; i<topology->nr_cpukinds; i++) // 遍历 CPU 种类
    for(j=i+1; j<topology->nr_cpukinds; j++) // 再次遍历 CPU 种类
      if (topology->cpukinds[i].ranking_value == topology->cpukinds[j].ranking_value) // 如果存在重复的排序值
        /* if any duplicate, fail */
        return -1; // 返回错误
  return 0; // 返回成功
}

static int
hwloc__cpukinds_try_rank_by_forced_efficiency(struct hwloc_topology *topology)
{
  unsigned i;

  hwloc_debug("Trying to rank cpukinds by forced efficiency...\n"); // 输出调试信息
  for(i=0; i<topology->nr_cpukinds; i++) { // 遍历 CPU 种类
    if (topology->cpukinds[i].forced_efficiency == HWLOC_CPUKIND_EFFICIENCY_UNKNOWN) // 如果存在未知效率
      /* if any unknown, fail */
      return -1; // 返回错误
    topology->cpukinds[i].ranking_value = topology->cpukinds[i].forced_efficiency; // 设置排序值为强制效率
  }

  return hwloc__cpukinds_check_duplicate_rankings(topology); // 检查是否存在重复排序值
}

struct hwloc_cpukinds_info_summary {
  int have_max_freq; // 是否有最大频率
  int have_base_freq; // 是否有基础频率
  int have_intel_core_type; // 是否有英特尔核心类型
  struct hwloc_cpukind_info_summary {
    unsigned intel_core_type; /* 1 for atom, 2 for core */ // 英特尔核心类型，1代表原子核心，2代表核心
    # 定义结构体成员变量，表示最大频率和基本频率，单位为 MHz，因此小于 100000
    unsigned max_freq, base_freq; /* MHz, hence < 100000 */
  } * summaries;
};

static void
hwloc__cpukinds_summarize_info(struct hwloc_topology *topology,
                               struct hwloc_cpukinds_info_summary *summary)
{
  unsigned i, j;

  // 初始化标记，表示已经获取了最大频率、基础频率和英特尔核心类型信息
  summary->have_max_freq = 1;
  summary->have_base_freq = 1;
  summary->have_intel_core_type = 1;

  // 遍历每个 CPU 种类
  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    // 遍历每个 CPU 种类的信息
    for(j=0; j<kind->nr_infos; j++) {
      struct hwloc_info_s *info = &kind->infos[j];
      // 如果信息名称是 "FrequencyMaxMHz"，则将其值转换为整数并存储到 summary 结构中
      if (!strcmp(info->name, "FrequencyMaxMHz")) {
        summary->summaries[i].max_freq = atoi(info->value);
      } else if (!strcmp(info->name, "FrequencyBaseMHz")) {
        // 如果信息名称是 "FrequencyBaseMHz"，则将其值转换为整数并存储到 summary 结构中
        summary->summaries[i].base_freq = atoi(info->value);
      } else if (!strcmp(info->name, "CoreType")) {
        // 如果信息名称是 "CoreType"，根据值设置英特尔核心类型
        if (!strcmp(info->value, "IntelAtom"))
          summary->summaries[i].intel_core_type = 1;
        else if (!strcmp(info->value, "IntelCore"))
          summary->summaries[i].intel_core_type = 2;
      }
    }
    // 输出调试信息，显示每个 CPU 种类的英特尔核心类型、最大频率和基础频率
    hwloc_debug("cpukind #%u has intel_core_type %u max_freq %u base_freq %u\n",
                i, summary->summaries[i].intel_core_type,
                summary->summaries[i].max_freq, summary->summaries[i].base_freq);
    // 检查是否存在基础频率、最大频率和英特尔核心类型信息
    if (!summary->summaries[i].base_freq)
      summary->have_base_freq = 0;
    if (!summary->summaries[i].max_freq)
      summary->have_max_freq = 0;
    if (!summary->summaries[i].intel_core_type)
      summary->have_intel_core_type = 0;
  }
}
enum hwloc_cpukinds_ranking {
  HWLOC_CPUKINDS_RANKING_DEFAULT, /* 默认的排名策略，根据 ARM 平台的强制性和频率，或者其他平台的强制性和核心类型频率 */
  HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY, /* 没有强制效率的默认排名策略 */
  HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY, /* 有强制效率的排名策略 */
  HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY, /* 核心类型或频率或两者都有的排名策略 */
  HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT, /* 严格要求核心类型和频率都有的排名策略 */
  HWLOC_CPUKINDS_RANKING_CORETYPE, /* 只有核心类型的排名策略 */
  HWLOC_CPUKINDS_RANKING_FREQUENCY, /* 只有频率的排名策略 */
  HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX, /* 最大频率的排名策略 */
  HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE, /* 基本频率的排名策略 */
  HWLOC_CPUKINDS_RANKING_NONE /* 没有排名策略 */
};

static int
hwloc__cpukinds_try_rank_by_info(struct hwloc_topology *topology,
                                 enum hwloc_cpukinds_ranking heuristics,
                                 struct hwloc_cpukinds_info_summary *summary)
{
  unsigned i;

  if (HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype+frequency_strict...\n");
    /* 尝试通过核心类型+频率严格排名cpukinds... */
    /* 我们需要 intel_core_type 和 (基本频率或最大频率) 对于所有种类 */
    if (!summary->have_intel_core_type
        || (!summary->have_max_freq && !summary->have_base_freq))
      return -1;
    /* 首先按核心类型 (Core>>Atom) 排序，然后按频率排序，如果有基本频率则使用基本频率，否则使用最大频率 */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      if (summary->have_base_freq)
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].base_freq;
      else
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype+frequency...\n");
    /* 尝试通过核心类型+频率排名cpukinds... */
    /* 我们需要 intel_core_type 或 (基本频率或最大频率) 对于所有种类 */
    // 如果没有检测到 Intel 核心类型，并且没有最大频率和基本频率，则返回 -1
    if (!summary->have_intel_core_type
        && (!summary->have_max_freq && !summary->have_base_freq))
      return -1;
    /* 首先按核心类型排名（Core>>Atom），然后按频率排名，如果有基本频率则使用基本频率，否则使用最大频率 */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      // 如果有基本频率，则使用基本频率进行排名
      if (summary->have_base_freq)
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].base_freq;
      // 否则使用最大频率进行排名
      else
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_CORETYPE == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype...\n");
    // 需要 Intel 核心类型
    if (!summary->have_intel_core_type)
      return -1;
    // 按核心类型（Core>>Atom）进行排名
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      kind->ranking_value = (summary->summaries[i].intel_core_type << 20);
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY == heuristics) {
    hwloc_debug("Trying to rank cpukinds by frequency...\n");
    // 需要所有种类的基本频率或最大频率
    if (!summary->have_max_freq && !summary->have_base_freq)
      return -1;
    // 首先按频率排名，如果有基本频率则使用基本频率，否则使用最大频率
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      // 如果有基本频率，则使用基本频率进行排名
      if (summary->have_base_freq)
        kind->ranking_value = summary->summaries[i].base_freq;
      // 否则使用最大频率进行排名
      else
        kind->ranking_value = summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX == heuristics) {
    hwloc_debug("Trying to rank cpukinds by frequency max...\n");
    // 需要所有种类的最大频率
    if (!summary->have_max_freq)
      return -1;
    // 首先按频率排名，如果有基本频率则使用基本频率，否则使用最大频率
    # 遍历 CPU 种类的数量
    for(i=0; i<topology->nr_cpukinds; i++) {
      # 获取当前 CPU 种类的指针
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      # 将当前 CPU 种类的排名值设置为频率的最大值
      kind->ranking_value = summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE == heuristics) {
    # 如果启用了基于频率基准的排名启发式算法，则输出调试信息
    hwloc_debug("Trying to rank cpukinds by frequency base...\n");
    # 如果没有基准频率信息，则返回错误
    if (!summary->have_base_freq)
      return -1;
    # 首先按照频率排名，如果有基准频率则使用基准频率，否则使用最大频率
    for(i=0; i<topology->nr_cpukinds; i++) {
      # 获取当前 CPU 种类的指针
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      # 将当前 CPU 种类的排名值设置为基准频率
      kind->ranking_value = summary->summaries[i].base_freq;
    }

  } else assert(0);

  # 检查是否存在重复的排名值，并返回结果
  return hwloc__cpukinds_check_duplicate_rankings(topology);
}

static int hwloc__cpukinds_compare_ranking_values(const void *_a, const void *_b)
{
  // 定义比较函数，用于排序 CPU 种类的排名值
  const struct hwloc_internal_cpukind_s *a = _a;
  const struct hwloc_internal_cpukind_s *b = _b;
  uint64_t arv = a->ranking_value;
  uint64_t brv = b->ranking_value;
  return arv < brv ? -1 : arv > brv ? 1 : 0;
}

/* this function requires ranking values to be unique */
static void
hwloc__cpukinds_finalize_ranking(struct hwloc_topology *topology)
{
  unsigned i;
  /* 对 CPU 种类进行排序 */
  qsort(topology->cpukinds, topology->nr_cpukinds, sizeof(*topology->cpukinds), hwloc__cpukinds_compare_ranking_values);
  /* 定义 CPU 种类的效率值在 0 到 N-1 之间 */
  for(i=0; i<topology->nr_cpukinds; i++)
    topology->cpukinds[i].efficiency = i;
}

int
hwloc_internal_cpukinds_rank(struct hwloc_topology *topology)
{
  enum hwloc_cpukinds_ranking heuristics;
  char *env;
  unsigned i;
  int err;

  if (!topology->nr_cpukinds)
    return 0;

  if (topology->nr_cpukinds == 1) {
    topology->cpukinds[0].efficiency = 0;
    return 0;
  }

  heuristics = HWLOC_CPUKINDS_RANKING_DEFAULT;
  env = getenv("HWLOC_CPUKINDS_RANKING");
  if (env) {
    // 根据环境变量设置 CPU 种类的排名策略
    if (!strcmp(env, "default"))
      heuristics = HWLOC_CPUKINDS_RANKING_DEFAULT;
    else if (!strcmp(env, "none"))
      heuristics = HWLOC_CPUKINDS_RANKING_NONE;
    else if (!strcmp(env, "coretype+frequency"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY;
    else if (!strcmp(env, "coretype+frequency_strict"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT;
    else if (!strcmp(env, "coretype"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE;
    else if (!strcmp(env, "frequency"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY;
    else if (!strcmp(env, "frequency_max"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX;
    else if (!strcmp(env, "frequency_base"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE;
    else if (!strcmp(env, "forced_efficiency"))
      heuristics = HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY;  # 如果环境变量为"forced_efficiency"，则设置heuristics为HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY
    else if (!strcmp(env, "no_forced_efficiency"))
      heuristics = HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY;  # 如果环境变量为"no_forced_efficiency"，则设置heuristics为HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY
    else if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Failed to recognize HWLOC_CPUKINDS_RANKING value %s\n", env);  # 如果环境变量不匹配任何预定义的值，输出错误信息到标准错误流
  }

  if (heuristics == HWLOC_CPUKINDS_RANKING_DEFAULT
      || heuristics == HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY) {
    /* default is forced_efficiency first */
    struct hwloc_cpukinds_info_summary summary;

    if (heuristics == HWLOC_CPUKINDS_RANKING_DEFAULT)
      hwloc_debug("Using default ranking strategy...\n");  # 如果heuristics为HWLOC_CPUKINDS_RANKING_DEFAULT，输出默认的排列策略信息
    else
      hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);  # 如果heuristics为HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY，输出自定义排列策略信息

    if (heuristics != HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY) {
      err = hwloc__cpukinds_try_rank_by_forced_efficiency(topology);  # 尝试按照强制效率排列CPU种类
      if (!err)
        goto ready;  # 如果成功，跳转到ready标签
    }

    summary.summaries = calloc(topology->nr_cpukinds, sizeof(*summary.summaries));  # 分配内存给summary.summaries
    if (!summary.summaries)
      goto failed;  # 如果分配内存失败，跳转到failed标签
    hwloc__cpukinds_summarize_info(topology, &summary);  # 对topology中的CPU种类信息进行总结

    err = hwloc__cpukinds_try_rank_by_info(topology, HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY, &summary);  # 尝试按照信息排列CPU种类
    free(summary.summaries);  # 释放summary.summaries的内存
    if (!err)
      goto ready;  # 如果成功，跳转到ready标签

  } else if (heuristics == HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY) {
    hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);  # 输出自定义排列策略信息

    err = hwloc__cpukinds_try_rank_by_forced_efficiency(topology);  # 尝试按照强制效率排列CPU种类
    if (!err)
      goto ready;  # 如果成功，跳转到ready标签

  } else if (heuristics != HWLOC_CPUKINDS_RANKING_NONE) {
    /* custom heuristics */
    struct hwloc_cpukinds_info_summary summary;

    hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);  # 输出自定义排列策略信息

    summary.summaries = calloc(topology->nr_cpukinds, sizeof(*summary.summaries));  # 分配内存给summary.summaries
    if (!summary.summaries)
      goto failed;  # 如果分配内存失败，跳转到failed标签
    hwloc__cpukinds_summarize_info(topology, &summary);  # 对topology中的CPU种类信息进行总结
    # 尝试根据信息对 CPU 种类进行排名
    err = hwloc__cpukinds_try_rank_by_info(topology, heuristics, &summary);
    # 释放 summary 结构体中的 summaries 数组
    free(summary.summaries);
    # 如果没有错误，则跳转到 ready 标签
    if (!err)
      goto ready;
  }

 failed:
  /* 排名失败，清除效率 */
  # 遍历 CPU 种类，将效率设置为未知
  for(i=0; i<topology->nr_cpukinds; i++)
    topology->cpukinds[i].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
  # 输出调试信息
  hwloc_debug("Failed to rank cpukinds.\n\n");
  # 返回 0
  return 0;

 ready:
  # 输出调试信息，显示每个 CPU 种类的排名值
  for(i=0; i<topology->nr_cpukinds; i++)
    hwloc_debug("cpukind #%u got ranking value %llu\n", i, (unsigned long long) topology->cpukinds[i].ranking_value);
  # 完成 CPU 种类的排名
  hwloc__cpukinds_finalize_ranking(topology);
#ifdef HWLOC_DEBUG
  // 如果定义了 HWLOC_DEBUG 宏，则进行以下断言检查
  for(i=0; i<topology->nr_cpukinds; i++)
    // 断言每个 cpukind 的效率等于其索引值
    assert(topology->cpukinds[i].efficiency == (int) i);
#endif
  // 输出调试信息
  hwloc_debug("\n");
  // 返回 0 表示成功
  return 0;
}


/*****************
 * Consulting
 */

// 获取 CPU 种类的数量
int
hwloc_cpukinds_get_nr(hwloc_topology_t topology, unsigned long flags)
{
  // 如果 flags 不为 0，则设置错误码并返回 -1
  if (flags) {
    errno = EINVAL;
    return -1;
  }
  // 返回 topology 中 CPU 种类的数量
  return topology->nr_cpukinds;
}

// 获取 CPU 种类的信息
int
hwloc_cpukinds_get_info(hwloc_topology_t topology,
                        unsigned id,
                        hwloc_bitmap_t cpuset,
                        int *efficiencyp,
                        unsigned *nr_infosp, struct hwloc_info_s **infosp,
                        unsigned long flags)
{
  struct hwloc_internal_cpukind_s *kind;

  // 如果 flags 不为 0，则设置错误码并返回 -1
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  // 如果 id 超出了 CPU 种类的数量范围，则设置错误码并返回 -1
  if (id >= topology->nr_cpukinds) {
    errno = ENOENT;
    return -1;
  }

  // 获取指定 id 的 CPU 种类
  kind = &topology->cpukinds[id];

  // 如果 cpuset 不为空，则将 kind 的 cpuset 复制给 cpuset
  if (cpuset)
    hwloc_bitmap_copy(cpuset, kind->cpuset);

  // 如果 efficiencyp 不为空，则将 kind 的 efficiency 赋值给 efficiencyp
  if (efficiencyp)
    *efficiencyp = kind->efficiency;

  // 如果 nr_infosp 和 infosp 都不为空，则将 kind 的 nr_infos 和 infos 赋值给它们
  if (nr_infosp && infosp) {
    *nr_infosp = kind->nr_infos;
    *infosp = kind->infos;
  }
  // 返回 0 表示成功
  return 0;
}

// 根据 cpuset 获取 CPU 种类的 id
int
hwloc_cpukinds_get_by_cpuset(hwloc_topology_t topology,
                             hwloc_const_bitmap_t cpuset,
                             unsigned long flags)
{
  unsigned id;

  // 如果 flags 不为 0，则设置错误码并返回 -1
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  // 如果 cpuset 为空或者为零，则设置错误码并返回 -1
  if (!cpuset || hwloc_bitmap_iszero(cpuset)) {
    errno = EINVAL;
    return -1;
  }

  // 遍历 CPU 种类，根据 cpuset 找到对应的 id
  for(id=0; id<topology->nr_cpukinds; id++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[id];
    int res = hwloc_bitmap_compare_inclusion(cpuset, kind->cpuset);
    if (res == HWLOC_BITMAP_EQUAL || res == HWLOC_BITMAP_INCLUDED) {
      return (int) id;
    } else if (res == HWLOC_BITMAP_INTERSECTS || res == HWLOC_BITMAP_CONTAINS) {
      errno = EXDEV;
      return -1;
    }
  }

  // 如果未找到对应的 id，则设置错误码并返回 -1
  errno = ENOENT;
  return -1;
}
```