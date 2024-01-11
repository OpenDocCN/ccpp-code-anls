# `xmrig\src\3rdparty\hwloc\src\topology-x86.c`

```
/*
 * 版权声明
 * 本后端仅在操作系统未将必要的硬件拓扑信息导出到用户空间应用程序时使用。
 * 目前，FreeBSD 和 NetBSD 仅添加处理器核心（PU），然后回退到此后端以进行 CPU/Cache 的发现。
 * 其他后端（如 Linux）有它们自己的方式从操作系统中检索各种硬件拓扑信息，而无需使用这个特定于 x86 的代码。
 * 但是在它们之后仍然使用此后端来为一些对象添加附加细节（Package 中的 CPU 信息，Cache 中的包含性）。
 */

#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含 hwloc 头文件
#include "private/private.h"  // 包含私有头文件
#include "private/debug.h"  // 包含调试相关的私有头文件
#include "private/misc.h"  // 包含杂项的私有头文件
#include "private/cpuid-x86.h"  // 包含 x86 CPUID 相关的私有头文件

#include <sys/types.h>  // 包含系统类型的头文件
#ifdef HAVE_DIRENT_H
#include <dirent.h>  // 包含目录项的头文件
#endif
#ifdef HAVE_VALGRIND_VALGRIND_H
#include <valgrind/valgrind.h>  // 包含 Valgrind 相关的头文件
#endif

// x86 后端的数据结构
struct hwloc_x86_backend_data_s {
  unsigned nbprocs;  // 处理器核心数量
  hwloc_bitmap_t apicid_set;  // APIC ID 集合
  int apicid_unique;  // APIC ID 是否唯一
  char *src_cpuiddump_path;  // CPUID dump 路径
  int is_knl;  // 是否为 KNL 架构
};

/************************************
 * 管理 CPUID dump 作为输入
 */

// CPUID dump 结构
struct cpuiddump {
  unsigned nr;  // 数量
  struct cpuiddump_entry {
    unsigned inmask;  // 输入掩码
    unsigned ineax;  // 输入 EAX 寄存器值
    unsigned inebx;  // 输入 EBX 寄存器值
    unsigned inecx;  // 输入 ECX 寄存器值
    unsigned inedx;  // 输入 EDX 寄存器值
    unsigned outeax;  // 输出 EAX 寄存器值
    unsigned outebx;  // 输出 EBX 寄存器值
    unsigned outecx;  // 输出 ECX 寄存器值
    unsigned outedx;  // 输出 EDX 寄存器值
  } *entries;  // 条目
};

// 释放 CPUID dump 结构
static void
cpuiddump_free(struct cpuiddump *cpuiddump) {
  if (cpuiddump->nr)
    free(cpuiddump->entries);
  free(cpuiddump);
}

// 读取 CPUID dump 结构
static struct cpuiddump *
cpuiddump_read(const char *dirpath, unsigned idx)
{
  // 声明一个指向结构体cpuiddump的指针cpuiddump
  struct cpuiddump *cpuiddump;
  // 声明一个指向结构体cpuiddump_entry的指针cur
  struct cpuiddump_entry *cur;
  // 声明一个变量filenamelen，用于存储文件名的长度
  size_t filenamelen;
  // 声明一个指向字符的指针filename，用于存储文件名
  char *filename;
  // 声明一个指向文件的指针file
  FILE *file;
  // 壿声明一个字符数组line，用于存储文件中的一行内容
  char line[128];
  // 声明一个无符号整数nr，用于存储行数
  unsigned nr;

  // 分配内存给cpuiddump指针，大小为结构体cpuiddump的大小
  cpuiddump = malloc(sizeof(*cpuiddump));
  // 如果分配内存失败，则输出错误信息并跳转到out标签
  if (!cpuiddump) {
    fprintf(stderr, "Failed to allocate cpuiddump for PU #%u, ignoring cpuiddump.\n", idx);
    goto out;
  }

  // 计算文件名的长度
  filenamelen = strlen(dirpath) + 15;
  // 分配内存给filename指针，大小为文件名长度
  filename = malloc(filenamelen);
  // 如果分配内存失败，则跳转到out_with_dump标签
  if (!filename)
    goto out_with_dump;
  // 将文件名格式化为"%s/pu%u"，存储到filename中
  snprintf(filename, filenamelen, "%s/pu%u", dirpath, idx);
  // 打开文件，以只读方式
  file = fopen(filename, "r");
  // 如果打开文件失败，则输出错误信息并跳转到out_with_filename标签
  if (!file) {
    fprintf(stderr, "Could not read dumped cpuid file %s, ignoring cpuiddump.\n", filename);
    goto out_with_filename;
  }

  // 初始化nr为0
  nr = 0;
  // 读取文件的每一行，计算行数
  while (fgets(line, sizeof(line), file))
    nr++;
  // 分配内存给cpuiddump->entries指针，大小为nr乘以结构体cpuiddump_entry的大小
  cpuiddump->entries = malloc(nr * sizeof(struct cpuiddump_entry));
  // 如果分配内存失败，则输出错误信息并跳转到out_with_file标签
  if (!cpuiddump->entries) {
    fprintf(stderr, "Failed to allocate %u cpuiddump entries for PU #%u, ignoring cpuiddump.\n", nr, idx);
    goto out_with_file;
  }

  // 将文件指针重新定位到文件开头
  fseek(file, 0, SEEK_SET);
  // 将cur指针指向cpuiddump->entries[0]
  cur = &cpuiddump->entries[0];
  // 重新初始化nr为0
  nr = 0;
  // 读取文件的每一行
  while (fgets(line, sizeof(line), file)) {
    // 如果行以#开头，则跳过
    if (*line == '#')
      continue;
    // 使用sscanf函数解析行中的内容，并存储到cur指向的结构体中
    if (sscanf(line, "%x %x %x %x %x => %x %x %x %x",
          &cur->inmask,
          &cur->ineax, &cur->inebx, &cur->inecx, &cur->inedx,
          &cur->outeax, &cur->outebx, &cur->outecx, &cur->outedx) == 9) {
      // cur指针向后移动一位
      cur++;
      // nr加1
      nr++;
    }
  }

  // 将cpuiddump->nr设置为nr
  cpuiddump->nr = nr;
  // 关闭文件
  fclose(file);
  // 释放filename指针所指向的内存
  free(filename);
  // 返回cpuiddump指针
  return cpuiddump;

  // 如果分配内存失败，则关闭文件并释放filename指针所指向的内存
 out_with_file:
  fclose(file);
 out_with_filename:
  free(filename);
  // 释放cpuiddump指针所指向的内存
 out_with_dump:
  free(cpuiddump);
  // 返回NULL
 out:
  return NULL;
}

// 根据输入查找cpuiddump
static void
cpuiddump_find_by_input(unsigned *eax, unsigned *ebx, unsigned *ecx, unsigned *edx, struct cpuiddump *cpuiddump)
{
  // 声明一个无符号整数i
  unsigned i;

  // 遍历cpuiddump->nr次
  for(i=0; i<cpuiddump->nr; i++) {
    // 声明一个指向结构体cpuiddump_entry的指针entry，指向cpuiddump->entries[i]
    struct cpuiddump_entry *entry = &cpuiddump->entries[i];
    // 如果(entry->inmask & 0x1)为真，并且*eax不等于entry->ineax，则继续下一次循环
    if ((entry->inmask & 0x1) && *eax != entry->ineax)
      continue;
    // 如果(entry->inmask & 0x2)为真，并且*ebx不等于entry->inebx，则继续下一次循环
    if ((entry->inmask & 0x2) && *ebx != entry->inebx)
      continue;
    # 如果入口的掩码中包含 0x4，并且 ecx 指针指向的值不等于入口的 inecx 值，则跳过当前循环
    if ((entry->inmask & 0x4) && *ecx != entry->inecx)
      continue;
    # 如果入口的掩码中包含 0x8，并且 edx 指针指向的值不等于入口的 inedx 值，则跳过当前循环
    if ((entry->inmask & 0x8) && *edx != entry->inedx)
      continue;
    # 将 eax 指针指向的值设置为入口的 outeax 值
    *eax = entry->outeax;
    # 将 ebx 指针指向的值设置为入口的 outebx 值
    *ebx = entry->outebx;
    # 将 ecx 指针指向的值设置为入口的 outecx 值
    *ecx = entry->outecx;
    # 将 edx 指针指向的值设置为入口的 outedx 值
    *edx = entry->outedx;
    # 返回结果
    return;
  }
  # 如果无法在转储的 cpuid 中找到 *eax, *ebx, *ecx, *edx 对应的值，则输出错误信息
  fprintf(stderr, "Couldn't find %x,%x,%x,%x in dumped cpuid, returning 0s.\n",
      *eax, *ebx, *ecx, *edx);
  # 将 *eax, *ebx, *ecx, *edx 的值都设置为 0
  *eax = 0;
  *ebx = 0;
  *ecx = 0;
  *edx = 0;
}
# 定义一个静态函数，用于根据给定的 CPUID 数据结构或者 CPUID 数据转储结构来获取 CPUID 信息
static void cpuid_or_from_dump(unsigned *eax, unsigned *ebx, unsigned *ecx, unsigned *edx, struct cpuiddump *src_cpuiddump)
{
  # 如果给定了 CPUID 数据转储结构，则根据输入的 CPUID 查找对应的信息
  if (src_cpuiddump) {
    cpuiddump_find_by_input(eax, ebx, ecx, edx, src_cpuiddump);
  } else {
    # 否则，调用 hwloc_x86_cpuid 函数获取 CPUID 信息
    hwloc_x86_cpuid(eax, ebx, ecx, edx);
  }
}

/*******************************
 * Core detection routines and structures
 */

# 定义枚举类型 hwloc_x86_disc_flags，用于表示 x86 架构的发现标志
enum hwloc_x86_disc_flags {
  HWLOC_X86_DISC_FLAG_FULL = (1<<0), /* discover everything instead of only annotating */
  HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES = (1<<1) /* use AMD topoext numanode information */
};

# 定义宏函数 has_topoext，用于判断给定的 CPU 特性是否支持 AMD topoext
#define has_topoext(features) ((features)[6] & (1 << 22))
# 定义宏函数 has_x2apic，用于判断给定的 CPU 特性是否支持 x2APIC
#define has_x2apic(features) ((features)[4] & (1 << 21))
# 定义宏函数 has_hybrid，用于判断给定的 CPU 特性是否支持混合模式
#define has_hybrid(features) ((features)[18] & (1 << 15))

# 定义缓存信息结构体 cacheinfo
struct cacheinfo {
  hwloc_obj_cache_type_t type;  # 缓存类型
  unsigned level;  # 缓存层级
  unsigned nbthreads_sharing;  # 共享线程数
  unsigned cacheid;  # 缓存 ID

  unsigned linesize;  # 缓存行大小
  unsigned linepart;  # 缓存行分区
  int inclusive;  # 是否包容性缓存
  int ways;  # 关联度
  unsigned sets;  # 组数
  unsigned long size;  # 缓存大小
};

# 定义处理器信息结构体 procinfo
struct procinfo {
  unsigned present;  # 是否存在
  unsigned apicid;  # APIC ID
  # 定义处理器 ID
  # 0: 包 ID
  # 1: 核心 ID
  # 2: NUMA 节点 ID
  # 3: 单元 ID
  # 4: TILE ID
  # 5: 模块 ID
  # 6: DIE ID
  unsigned ids[HWLOC_X86_PROCINFO_ID_NR];
  unsigned *otherids;  # 其他 ID
  unsigned levels;  # 缓存层级数
  unsigned numcaches;  # 缓存数量
  struct cacheinfo *cache;  # 缓存信息数组
  char cpuvendor[13];  # CPU 厂商
  char cpumodel[3*4*4+1];  # CPU 型号
  unsigned cpustepping;  # CPU 步进
  unsigned cpumodelnumber;  # CPU 型号编号
  unsigned cpufamilynumber;  # CPU 家族编号

  unsigned hybridcoretype;  # 混合核心类型
  unsigned hybridnativemodel;  # 混合本地模型
};

# 定义 CPUID 类型枚举
enum cpuid_type {
  intel,  # 英特尔
  amd,  # AMD
  zhaoxin,  # 中兴
  hygon,  # 海光
  unknown  # 未知
};

# 定义一个静态函数，用于设置 AMD CPU 的遗留缓存信息
static void setup__amd_cache_legacy(struct procinfo *infos, unsigned level, hwloc_obj_cache_type_t type, unsigned nbthreads_sharing, unsigned cpuid)
{
  struct cacheinfo *cache, *tmpcaches;
  unsigned cachenum;
  unsigned long size = 0;

  # 如果缓存层级为 1，则计算缓存大小
  if (level == 1)
    size = ((cpuid >> 24)) << 10;
  # 如果缓存层级为 2
  else if (level == 2)
    # 根据 CPUID 和 level 计算缓存大小
    size = ((cpuid >> 16)) << 10;
  else if (level == 3)
    # 根据 CPUID 和 level 计算缓存大小
    size = ((cpuid >> 18)) << 19;
  # 如果缓存大小为 0，则返回
  if (!size)
    return;

  # 重新分配内存给 tmpcaches
  tmpcaches = realloc(infos->cache, (infos->numcaches+1)*sizeof(*infos->cache));
  # 如果分配失败，则忽略该缓存
  if (!tmpcaches)
    return;
  # 将 tmpcaches 赋值给 infos->cache
  infos->cache = tmpcaches;
  # 将缓存编号赋值给 cachenum，并递增 numcaches
  cachenum = infos->numcaches++;

  # 获取当前缓存的指针
  cache = &infos->cache[cachenum];

  # 设置缓存的类型、级别、共享线程数、行大小、线部分、是否包容性
  cache->type = type;
  cache->level = level;
  cache->nbthreads_sharing = nbthreads_sharing;
  cache->linesize = cpuid & 0xff;
  cache->linepart = 0;
  cache->inclusive = 0; # 旧的 AMD (K8-K10) 假定具有排他性缓存

  # 如果级别为 1，则设置缓存的路数
  if (level == 1) {
    cache->ways = (cpuid >> 16) & 0xff;
    # 如果路数为 0xff，则设置为全关联
    if (cache->ways == 0xff)
      cache->ways = -1;
  } else {
    # 根据 CPUID 计算路数
    static const unsigned ways_tab[] = { 0, 1, 2, 0, 4, 0, 8, 0, 16, 0, 32, 48, 64, 96, 128, -1 };
    unsigned ways = (cpuid >> 12) & 0xf;
    cache->ways = ways_tab[ways];
  }
  # 设置缓存的大小和集合数
  cache->size = size;
  cache->sets = 0;

  # 输出缓存的调试信息
  hwloc_debug("cache L%u t%u linesize %u ways %d size %luKB\n", cache->level, cache->nbthreads_sharing, cache->linesize, cache->ways, cache->size >> 10);
}
/* 从 CPUID 0x80000005-6 leaves 读取 AMD 老版本的缓存信息 */
static void read_amd_caches_legacy(struct procinfo *infos, struct cpuiddump *src_cpuiddump, unsigned legacy_max_log_proc)
{
  unsigned eax, ebx, ecx, edx;

  eax = 0x80000005;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_DATA, 1, ecx); /* 设置私有 L1d 缓存 */
  setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_INSTRUCTION, 1, edx); /* 设置私有 L1i 缓存 */

  eax = 0x80000006;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  if (ecx & 0xf000)
    /* 实际上这也支持在 Intel 上，但 LinePerTag 不在 8-11 位返回。
     * 如果一些 Intel（至少在 Core 微架构之前）支持此 leaf 而不需要 leaf 0x4，这可能会有用。
     */
    setup__amd_cache_legacy(infos, 2, HWLOC_OBJ_CACHE_UNIFIED, 1, ecx); /* 设置私有 L2u 缓存 */
  if (edx & 0xf000)
    setup__amd_cache_legacy(infos, 3, HWLOC_OBJ_CACHE_UNIFIED, legacy_max_log_proc, edx); /* 设置整个 package 的 L3u 缓存 */
}

/* 从 CPUID 0x8000001d leaf (topoext) 读取 AMD 缓存信息 */
static void read_amd_caches_topoext(struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned eax, ebx, ecx, edx;
  unsigned cachenum;
  struct cacheinfo *cache;

  /* 以下代码暂时不需要其他缓存 */
  assert(!infos->numcaches);

  for (cachenum = 0; ; cachenum++) {
    eax = 0x8000001d;
    ecx = cachenum;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if ((eax & 0x1f) == 0)
      break;
    infos->numcaches++;
  }

  cache = infos->cache = malloc(infos->numcaches * sizeof(*infos->cache));
  if (cache) {
    for (cachenum = 0; ; cachenum++) {
      unsigned long linesize, linepart, ways, sets;
      eax = 0x8000001d;
      ecx = cachenum;
      cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

      if ((eax & 0x1f) == 0)
    # 结束当前的循环
    break;
      # 根据 eax 的低 5 位确定缓存类型
      switch (eax & 0x1f) {
      # 如果为 1，表示数据缓存
      case 1: cache->type = HWLOC_OBJ_CACHE_DATA; break;
      # 如果为 2，表示指令缓存
      case 2: cache->type = HWLOC_OBJ_CACHE_INSTRUCTION; break;
      # 其他情况，表示统一缓存
      default: cache->type = HWLOC_OBJ_CACHE_UNIFIED; break;
      }

      # 获取缓存的层级
      cache->level = (eax >> 5) & 0x7;
      # 获取共享缓存的线程数
      /* 注意：实际上是核心数 */
      cache->nbthreads_sharing = ((eax >> 14) &  0xfff) + 1;

      # 获取缓存行的大小
      cache->linesize = linesize = (ebx & 0xfff) + 1;
      # 获取缓存行的部分
      cache->linepart = linepart = ((ebx >> 12) & 0x3ff) + 1;
      # 获取缓存的路数
      ways = ((ebx >> 22) & 0x3ff) + 1;

      # 如果 eax 的第 9 位为 1，表示完全关联
      if (eax & (1 << 9))
    /* 完全关联 */
    cache->ways = -1;
      # 否则，表示路数为 ways
      else
    cache->ways = ways;
      # 获取缓存的组数
      cache->sets = sets = ecx + 1;
      # 计算缓存的大小
      cache->size = linesize * linepart * ways * sets;
      # 判断缓存是否包容
      cache->inclusive = edx & 0x2;

      # 调试信息，打印缓存的各项参数
      hwloc_debug("cache %u L%u%c t%u linesize %lu linepart %lu ways %lu sets %lu, size %luKB\n",
          cachenum, cache->level,
          cache->type == HWLOC_OBJ_CACHE_DATA ? 'd' : cache->type == HWLOC_OBJ_CACHE_INSTRUCTION ? 'i' : 'u',
          cache->nbthreads_sharing, linesize, linepart, ways, sets, cache->size >> 10);

      # 指针移动到下一个缓存结构
      cache++;
    }
  # 如果没有缓存信息
  } else {
    # 缓存数置为 0
    infos->numcaches = 0;
  }
  /* 从 CPUID 0x04 leaf 读取 Intel 缓存信息 */
static void read_intel_caches(struct hwloc_x86_backend_data_s *data, struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned level;
  struct cacheinfo *tmpcaches;
  unsigned eax, ebx, ecx, edx;
  unsigned oldnumcaches = infos->numcaches; /* 保存之前的缓存数量，以防我们得到了更多的缓存 */
  unsigned cachenum;
  struct cacheinfo *cache;

  for (cachenum = 0; ; cachenum++) {
    eax = 0x04;
    ecx = cachenum;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

    hwloc_debug("cache %u type %u\n", cachenum, eax & 0x1f);  // 打印缓存类型和编号
    if ((eax & 0x1f) == 0)  // 如果缓存类型为 0，表示没有更多缓存了，退出循环
      break;
    level = (eax >> 5) & 0x7;  // 获取缓存级别
    if (data->is_knl && level == 3)
      /* KNL 报告了错误的 L3 信息（大小始终为 0，cpuset 始终是整个机器），忽略它 */
      break;
    infos->numcaches++;  // 缓存数量加一
  }

  tmpcaches = realloc(infos->cache, infos->numcaches * sizeof(*infos->cache));  // 重新分配缓存数组的内存空间
  if (!tmpcaches) {
    infos->numcaches = oldnumcaches;  // 如果内存分配失败，恢复之前的缓存数量
  } else {
    infos->cache = tmpcaches;  // 更新缓存数组
    cache = &infos->cache[oldnumcaches];  // 获取新缓存的起始位置

    for (cachenum = 0; ; cachenum++) {
      unsigned long linesize, linepart, ways, sets;
      eax = 0x04;
      ecx = cachenum;
      cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

      if ((eax & 0x1f) == 0)  // 如果缓存类型为 0，表示没有更多缓存了，退出循环
    break;
      level = (eax >> 5) & 0x7;  // 获取缓存级别
      if (data->is_knl && level == 3)
    /* KNL 报告了错误的 L3 信息（大小始终为 0，cpuset 始终是整个机器），忽略它 */
    # 结束当前的循环
    break;
      # 根据 eax 的低 5 位进行分支
      switch (eax & 0x1f) {
      # 如果低 5 位为 1，表示缓存类型为数据缓存
      case 1: cache->type = HWLOC_OBJ_CACHE_DATA; break;
      # 如果低 5 位为 2，表示缓存类型为指令缓存
      case 2: cache->type = HWLOC_OBJ_CACHE_INSTRUCTION; break;
      # 其他情况，表示缓存类型为统一缓存
      default: cache->type = HWLOC_OBJ_CACHE_UNIFIED; break;
      }

      # 缓存层级为给定的 level
      cache->level = level;
      # 可共享的线程数为 eax 的第 15 至 28 位加 1
      cache->nbthreads_sharing = ((eax >> 14) & 0xfff) + 1;

      # 缓存行大小为 ebx 的低 12 位加 1
      cache->linesize = linesize = (ebx & 0xfff) + 1;
      # 缓存行的部分数为 ebx 的第 13 至 24 位加 1
      cache->linepart = linepart = ((ebx >> 12) & 0x3ff) + 1;
      # 关联度为 ebx 的第 23 至 32 位加 1
      ways = ((ebx >> 22) & 0x3ff) + 1;
      # 如果 eax 的第 9 位为 1，表示完全关联
      if (eax & (1 << 9))
        # 设置关联度为 -1
        cache->ways = -1;
      else
        cache->ways = ways;
      # 集合数为 ecx 加 1
      cache->sets = sets = ecx + 1;
      # 缓存大小为行大小 * 行部分 * 关联度 * 集合数
      cache->size = linesize * linepart * ways * sets;
      # 是否包含其他缓存
      cache->inclusive = edx & 0x2;

      # 调试信息，打印缓存的各项参数
      hwloc_debug("cache %u L%u%c t%u linesize %lu linepart %lu ways %lu sets %lu, size %luKB\n",
          cachenum, cache->level,
          cache->type == HWLOC_OBJ_CACHE_DATA ? 'd' : cache->type == HWLOC_OBJ_CACHE_INSTRUCTION ? 'i' : 'u',
          cache->nbthreads_sharing, linesize, linepart, ways, sets, cache->size >> 10);
      # 指针指向下一个缓存结构
      cache++;
    }
  }
}

/* 从 CPUID 0x80000008 叶子节点读取 AMD 核心/线程信息 */
static void read_amd_cores_legacy(struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned eax, ebx, ecx, edx;
  unsigned max_nbcores;
  unsigned max_nbthreads;
  unsigned coreidsize;
  unsigned logprocid;
  unsigned threadid __hwloc_attribute_unused;

  eax = 0x80000008;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

  coreidsize = (ecx >> 12) & 0xf;
  hwloc_debug("core ID size: %u\n", coreidsize);
  if (!coreidsize) {
    max_nbcores = (ecx & 0xff) + 1;
  } else
    max_nbcores = 1 << coreidsize;
  hwloc_debug("Thus max # of cores: %u\n", max_nbcores);

  /* 旧的 CPUID 叶子节点不支持多线程的 AMD 处理器 */
  max_nbthreads = 1 ;
  hwloc_debug("and max # of threads: %u\n", max_nbthreads);

  /* legacy_max_log_proc 已经废弃，它可能比 max_nbcores 小，
   * max_nbcores 是处理器理论上支持的最大核心数
   * (参见 AMD CPUID 规范中的 "Multiple Core Calculation")。
   * 因此需要重新计算 packageid/coreid。
   */
  infos->ids[PKG] = infos->apicid / max_nbcores;
  logprocid = infos->apicid % max_nbcores;
  infos->ids[CORE] = logprocid / max_nbthreads;
  threadid = logprocid % max_nbthreads;
  hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
}

/* 从 CPUID 0x8000001e 叶子节点 (topoext) 读取 AMD 单元/节点信息 */
static void read_amd_cores_topoext(struct procinfo *infos, unsigned long flags, struct cpuiddump *src_cpuiddump)
{
  unsigned apic_id, nodes_per_proc = 0;
  unsigned eax, ebx, ecx, edx;

  eax = 0x8000001e;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  infos->apicid = apic_id = eax;

  if (flags & HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES) {
    if (infos->cpufamilynumber == 0x16) {
      /* ecx is reserved */
      infos->ids[NODE] = 0;
      nodes_per_proc = 1;
  } else {
    /* 如果不是 AMD 或者 Hygon 18h 家族的处理器，则执行以下操作 */
    infos->ids[NODE] = ecx & 0xff;  // 将 ecx 寄存器的低 8 位赋值给节点 ID
    nodes_per_proc = ((ecx >> 8) & 7) + 1;  // 计算每个处理器的节点数，并赋值给 nodes_per_proc
  }
  if ((infos->cpufamilynumber == 0x15 && nodes_per_proc > 2)
  || ((infos->cpufamilynumber == 0x17 || infos->cpufamilynumber == 0x18) && nodes_per_proc > 4)
      || (infos->cpufamilynumber == 0x19 && nodes_per_proc > 1)) {
    hwloc_debug("warning: undefined nodes_per_proc value %u, assuming it means %u\n", nodes_per_proc, nodes_per_proc);  // 输出警告信息，提示节点数值未定义
  }
}

if (infos->cpufamilynumber <= 0x16) { /* topoext appeared in 0x15 and compute-units were only used in 0x15 and 0x16 */
  unsigned cores_per_unit;
  /* coreid was obtained from read_amd_cores_legacy() earlier */
  infos->ids[UNIT] = ebx & 0xff;  // 将 ebx 寄存器的低 8 位赋值给单元 ID
  cores_per_unit = ((ebx >> 8) & 0xff) + 1;  // 计算每个单元的核心数，并赋值给 cores_per_unit
  hwloc_debug("topoext %08x, %u nodes, node %u, %u cores in unit %u\n", apic_id, nodes_per_proc, infos->ids[NODE], cores_per_unit, infos->ids[UNIT]);  // 输出调试信息，显示拓扑扩展信息
  /* coreid and unitid are package-wide (core 0-15 and unit 0-7 on 16-core 2-NUMAnode processor).
   * The Linux kernel reduces theses to NUMA-node-wide (by applying %core_per_node and %unit_per node respectively).
   * It's not clear if we should do this as well.
   */
} else {
  unsigned threads_per_core;
  infos->ids[CORE] = ebx & 0xff;  // 将 ebx 寄存器的低 8 位赋值给核心 ID
  threads_per_core = ((ebx >> 8) & 0xff) + 1;  // 计算每个核心的线程数，并赋值给 threads_per_core
  hwloc_debug("topoext %08x, %u nodes, node %u, %u threads in core %u\n", apic_id, nodes_per_proc, infos->ids[NODE], threads_per_core, infos->ids[CORE]);  // 输出调试信息，显示拓扑扩展信息
}
/* 从 CPUID 0x0b 或 0x1f 离开（v1 和 v2 扩展拓扑枚举）中读取 Intel 核心/线程或甚至从 CPUID 0x0b 或 0x1f 离开（v1 和 v2 扩展拓扑枚举）中读取模块/瓦片 */
static void read_intel_cores_exttopoenum(struct procinfo *infos, unsigned leaf, struct cpuiddump *src_cpuiddump)
{
  unsigned level, apic_nextshift, apic_number, apic_type, apic_id = 0, apic_shift = 0, id;
  unsigned threadid __hwloc_attribute_unused = 0; /* 关闭编译器警告 */
  unsigned eax, ebx, ecx = 0, edx;
  int apic_packageshift = 0;

  for (level = 0; ; level++) {
    ecx = level;
    eax = leaf;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if (!eax && !ebx)
      break;
    apic_packageshift = eax & 0x1f;
  }

  if (level) {
    infos->otherids = malloc(level * sizeof(*infos->otherids));  // 分配内存给其他 ID
    if (infos->otherids) {
      infos->levels = level;  // 设置层级
      for (level = 0; ; level++) {
    ecx = level;
    eax = leaf;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if (!eax && !ebx)
      break;
    apic_nextshift = eax & 0x1f;
    apic_number = ebx & 0xffff;
    apic_type = (ecx & 0xff00) >> 8;
    apic_id = edx;
    id = (apic_id >> apic_shift) & ((1 << (apic_packageshift - apic_shift)) - 1);
    hwloc_debug("x2APIC %08x %u: nextshift %u num %2u type %u id %2u\n", apic_id, level, apic_nextshift, apic_number, apic_type, id);  // 输出调试信息
    infos->apicid = apic_id;  // 设置 apicid
    infos->otherids[level] = UINT_MAX;  // 设置其他 ID
    switch (apic_type) {
    case 1:
      threadid = id;  // 设置线程 ID
      /* apic_number is the actual number of threads per core */
      break;
    case 2:
      infos->ids[CORE] = id;  // 设置核心 ID
      /* apic_number is the actual number of threads per die */
      break;
    case 3:
      infos->ids[MODULE] = id;  // 设置模块 ID
      /* apic_number is the actual number of threads per tile */
      break;
    case 4:
      infos->ids[TILE] = id;  // 设置瓦片 ID
      /* apic_number is the actual number of threads per die */
      break;
    case 5:
      infos->ids[DIE] = id;  // 设置 die ID
      /* apic_number is the actual number of threads per package */
      break;
    # 默认情况下，输出 x2APIC 的类型和级别
    default:
      hwloc_debug("x2APIC %u: unknown type %u\n", level, apic_type);
      # 将 x2APIC ID 右移 apic_shift 位，存储到其他 ID 数组中
      infos->otherids[level] = apic_id >> apic_shift;
      # 跳出 switch 语句
      break;
    }
    # 更新 apic_shift 的值为 apic_nextshift
    apic_shift = apic_nextshift;
      }
      # 将 apic_id 存储到 apicid 中
      infos->apicid = apic_id;
      # 将 apic_id 右移 apic_shift 位，存储到 ids 数组的 PKG 位置
      infos->ids[PKG] = apic_id >> apic_shift;
      # 输出 x2APIC 的余数
      hwloc_debug("x2APIC remainder: %u\n", infos->ids[PKG]);
      # 输出当前线程所在的核心和线程 ID
      hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
    }
  }
/* 从处理器本身获取信息，使用 cpuid 并将其存储在 infos 中，以便供 summarize 全局分析 */
static void look_proc(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags, unsigned highest_cpuid, unsigned highest_ext_cpuid, unsigned *features, enum cpuid_type cpuid_type, struct cpuiddump *src_cpuiddump)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  unsigned eax, ebx, ecx = 0, edx;
  unsigned cachenum;
  struct cacheinfo *cache;
  unsigned regs[4];
  unsigned legacy_max_log_proc; /* 在 Intel 处理器中，当线程数 > 256 或支持 cpuid 0x80000008 时，此值无效 */
  unsigned legacy_log_proc_id;
  unsigned _model, _extendedmodel, _family, _extendedfamily;

  infos->present = 1;

  /* 从 cpuid 0x01 获取 apicid、legacy_max_log_proc、packageid、legacy_log_proc_id */
  eax = 0x01;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  infos->apicid = ebx >> 24;
  if (edx & (1 << 28)) {
    legacy_max_log_proc = 1 << hwloc_flsl(((ebx >> 16) & 0xff) - 1);
  } else {
    hwloc_debug("CPUID 0x01.edx 中未设置 HTT 位，假设 legacy_max_log_proc = 1\n");
    legacy_max_log_proc = 1;
  }

  hwloc_debug("APIC ID 0x%02x legacy_max_log_proc %u\n", infos->apicid, legacy_max_log_proc);
  infos->ids[PKG] = infos->apicid / legacy_max_log_proc;
  legacy_log_proc_id = infos->apicid % legacy_max_log_proc;
  hwloc_debug("物理 %u 旧线程 %u\n", infos->ids[PKG], legacy_log_proc_id);

  /* 从相同的 cpuid 获取 cpu 型号/家族/步进数 */
  _model          = (eax>>4) & 0xf;
  _extendedmodel  = (eax>>16) & 0xf;
  _family         = (eax>>8) & 0xf;
  _extendedfamily = (eax>>20) & 0xff;
  if ((cpuid_type == intel || cpuid_type == amd || cpuid_type == hygon) && _family == 0xf) {
    infos->cpufamilynumber = _family + _extendedfamily;
  } else {
  // 将 CPU 族系列号赋值给 infos 结构体中的 cpufamilynumber 字段
  infos->cpufamilynumber = _family;
  // 根据不同的 CPU 类型和系列号，确定 CPU 型号号码并赋值给 infos 结构体中的 cpumodelnumber 字段
  if ((cpuid_type == intel && (_family == 0x6 || _family == 0xf))
      || ((cpuid_type == amd || cpuid_type == hygon) && _family == 0xf)
      || (cpuid_type == zhaoxin && (_family == 0x6 || _family == 0x7))) {
    infos->cpumodelnumber = _model + (_extendedmodel << 4);
  } else {
    infos->cpumodelnumber = _model;
  }
  // 从寄存器中获取 CPU 的步进信息，并赋值给 infos 结构体中的 cpustepping 字段
  infos->cpustepping = eax & 0xf;
  // 如果是英特尔 CPU 且 CPU 族系列号为 0x6，且 CPU 型号号码为 0x57 或 0x85，则将 data 结构体中的 is_knl 字段设为 1
  if (cpuid_type == intel && infos->cpufamilynumber == 0x6 &&
      (infos->cpumodelnumber == 0x57 || infos->cpumodelnumber == 0x85))
    data->is_knl = 1; /* KNM is the same as KNL */

  // 从 cpuid 0x00 中获取 CPU 供应商信息，并存储到 infos 结构体中的 cpuvendor 字段
  memset(regs, 0, sizeof(regs));
  regs[0] = 0;
  cpuid_or_from_dump(&regs[0], &regs[1], &regs[3], &regs[2], src_cpuiddump);
  memcpy(infos->cpuvendor, regs+1, 4*3);
  /* infos was calloc'ed, already ends with \0 */

  // 如果支持扩展的 cpuid 0x80000002-4，则从中获取 CPU 型号信息，并存储到 infos 结构体中的 cpumodel 字段
  if (highest_ext_cpuid >= 0x80000004) {
    memset(regs, 0, sizeof(regs));
    regs[0] = 0x80000002;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel, regs, 4*4);
    regs[0] = 0x80000003;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel + 4*4, regs, 4*4);
    regs[0] = 0x80000004;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel + 4*4*2, regs, 4*4);
    /* infos was calloc'ed, already ends with \0 */
  }

  // 如果不是 AMD 或 Hygon CPU，并且支持 cpuid 0x04，则从中获取核心/线程信息
  if ((cpuid_type != amd && cpuid_type != hygon) && highest_cpuid >= 0x04) {
    /* Get core/thread information from first cache reported by cpuid 0x04
     * (not supported on AMD)
     */
    eax = 0x04;
    ecx = 0;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  }
    # 如果 eax 和 0x1f 的按位与结果不为 0
    if ((eax & 0x1f) != 0) {
      # 缓存看起来有效
      unsigned max_nbcores;
      unsigned max_nbthreads;
      unsigned threadid __hwloc_attribute_unused;
      hwloc_debug("Trying to get core/thread IDs from 0x04...\n");
      # 从 0x04 获取核心/线程 ID
      max_nbcores = ((eax >> 26) & 0x3f) + 1;
      hwloc_debug("found %u cores max\n", max_nbcores);
      # 一些虚拟机（例如 issue#525）未报告有效信息，在除以 0 之前检查一下
      if (!max_nbcores) {
        hwloc_debug("cannot detect core/thread IDs from 0x04 without a valid max of cores\n");
      } else {
        max_nbthreads = legacy_max_log_proc / max_nbcores;
        hwloc_debug("found %u threads max\n", max_nbthreads);
        if (!max_nbthreads) {
          hwloc_debug("cannot detect core/thread IDs from 0x04 without a valid max of threads\n");
        } else {
          threadid = legacy_log_proc_id % max_nbthreads;
          infos->ids[CORE] = legacy_log_proc_id / max_nbthreads;
          hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
        }
      }
    }
  }

  # 如果 highest_cpuid 大于等于 0x1a 并且具有混合特性
  if (highest_cpuid >= 0x1a && has_hybrid(features)) {
    # 从 cpuid 0x1a 获取混合 CPU 信息
    eax = 0x1a;
    ecx = 0;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    infos->hybridcoretype = eax >> 24;
    infos->hybridnativemodel = eax & 0xffffff;
  }

  /*********************************************************************************
   * 从 CPU 特定的叶子节点获取线程、核心、die、package 等的层次结构
   */

  # 如果 cpuid_type 不是 intel 且 cpuid_type 不是 zhaoxin 且 highest_ext_cpuid 大于等于 0x80000008 且没有 x2apic 特性
  if (cpuid_type != intel && cpuid_type != zhaoxin && highest_ext_cpuid >= 0x80000008 && !has_x2apic(features)) {
    # 从 cpuid 0x80000008 获取核心/线程信息（在 Intel 上不支持）
    # 当 x2apic 受支持时，我们可以忽略此代码路径，但如果设置了 HWLOC_X86_TOPOEXT_NUMANODES，我们可能需要 nodeids。
  // 调用函数读取 AMD 平台旧版本的处理器核心信息
  read_amd_cores_legacy(infos, src_cpuiddump);
}

if (cpuid_type != intel && cpuid_type != zhaoxin && has_topoext(features)) {
  /* 从 cpuid 0x8000001e (AMD topology extension) 获取 apicid、nodeid、unitid/coreid 信息。
   * 在 0x15-16 家族上需要 read_amd_cores_legacy() 来获取 coreid 信息。
   *
   * 仅在需要 NUMA 节点时才需要 x2apic 支持。
   */
  read_amd_cores_topoext(infos, flags, src_cpuiddump);
}

if ((cpuid_type == intel) && highest_cpuid >= 0x1f) {
  /* 从 cpuid 0x1f (Intel v2 Extended Topology Enumeration) 获取 package/die/module/tile/core/thread 信息 */
  read_intel_cores_exttopoenum(infos, 0x1f, src_cpuiddump);

} else if ((cpuid_type == intel || cpuid_type == amd || cpuid_type == zhaoxin)
       && highest_cpuid >= 0x0b && has_x2apic(features)) {
  /* 从 cpuid 0x0b (Intel v1 Extended Topology Enumeration) 获取 package/core/thread 信息 */
  read_intel_cores_exttopoenum(infos, 0x0b, src_cpuiddump);
}

/**************************************
 * 从 CPU 特定的 leaves 获取缓存信息
 */

infos->numcaches = 0;
infos->cache = NULL;

if (cpuid_type != intel && cpuid_type != zhaoxin && has_topoext(features)) {
  /* 从 cpuid 0x8000001d (AMD topology extension) 获取缓存信息 */
  read_amd_caches_topoext(infos, src_cpuiddump);

} else if (cpuid_type != intel && cpuid_type != zhaoxin && highest_ext_cpuid >= 0x80000006) {
  /* 如果没有 topoext，从 cpuid 0x80000005 和 0x80000006 获取缓存信息。
   * (Intel 不支持)
   * 看起来我们不能单独使用 0x80000005，必须和 0x80000006 一起使用。
   */
  read_amd_caches_legacy(infos, src_cpuiddump, legacy_max_log_proc);
}

if ((cpuid_type != amd && cpuid_type != hygon) && highest_cpuid >= 0x04) {
  /* 从 cpuid 0x04 获取缓存信息
   * (AMD 不支持)
   */
  read_intel_caches(data, infos, src_cpuiddump);
  }

  /* 现在我们有了所有的信息，计算缓存ID并应用特殊处理 */
  for (cachenum = 0; cachenum < infos->numcaches; cachenum++) {
    cache = &infos->cache[cachenum];

    /* 默认的缓存ID值 */
    cache->cacheid = infos->apicid / cache->nbthreads_sharing;

    if (cpuid_type == intel) {
      /* 将nbthreads_sharing四舍五入到最接近的2的幂，以构建一个掩码（用于清除低位） */
      unsigned bits = hwloc_flsl(cache->nbthreads_sharing-1);
      unsigned mask = ~((1U<<bits) - 1);
      cache->cacheid = infos->apicid & mask;

    } else if (cpuid_type == amd) {
      /* AMD特殊处理 */
      if (infos->cpufamilynumber >= 0x17 && cache->level == 3) {
    /* AMD家族0x19始终在16个APIC ID（8个HT核心）之间共享L3。
         * 而家族0x17在8个APIC ID（4个HT核心）之间共享。
         * 但许多型号启用的APIC ID较少并在nbthreads_sharing中报告。
         * 这意味着我们必须将nbthreads_sharing四舍五入到最接近的2的幂
         * 然后再计算cacheid。
     */
        unsigned nbapics_sharing = cache->nbthreads_sharing;
        if (nbapics_sharing & (nbapics_sharing-1))
          /* 不是2的幂，四舍五入 */
          nbapics_sharing = 1U<<(1+hwloc_ffsl(nbapics_sharing));

    cache->cacheid = infos->apicid / nbapics_sharing;

      } else if (infos->cpufamilynumber== 0x10 && infos->cpumodelnumber == 0x9
      && cache->level == 3
      && (cache->ways == -1 || (cache->ways % 2 == 0)) && cache->nbthreads_sharing >= 8) {
    /* 修复AMD家族0x10型号0x9（Magny-Cours）具有8或12个核心。
     * L3（及其关联性）实际上分为两半。
     */
    if (cache->nbthreads_sharing == 16)
      cache->nbthreads_sharing = 12; /* nbthreads_sharing是2的幂，但处理器实际上有8或12个核心 */
    cache->nbthreads_sharing /= 2;
    cache->size /= 2;
    if (cache->ways != -1)
      cache->ways /= 2;
    /* AMD Magny-Cours 12-cores processor reserve APIC ids as AAAAAABBBBBB....
     * among first L3 (A), second L3 (B), and unexisting cores (.).
     * On multi-socket servers, L3 in non-first sockets may have APIC id ranges
     * such as [16-21] that are not aligned on multiple of nbthreads_sharing (6).
     * That means, we can't just compare apicid/nbthreads_sharing to identify siblings.
     */
    // 设置缓存的 cacheid 属性，用于标识在同一包中的缓存
    cache->cacheid = (infos->apicid % legacy_max_log_proc) / cache->nbthreads_sharing /* cacheid within the package */
      + 2 * (infos->apicid / legacy_max_log_proc); /* add 2 caches per previous package */

      } else if (infos->cpufamilynumber == 0x15
         && (infos->cpumodelnumber == 0x1 /* Bulldozer */ || infos->cpumodelnumber == 0x2 /* Piledriver */)
         && cache->level == 3 && cache->nbthreads_sharing == 6) {
    /* AMD Bulldozer and Piledriver 12-core processors have same APIC ids as Magny-Cours above,
     * but we can't merge the checks because the original nbthreads_sharing must be exactly 6 here.
     */
    // 设置缓存的 cacheid 属性，用于标识在同一包中的缓存
    cache->cacheid = (infos->apicid % legacy_max_log_proc) / cache->nbthreads_sharing /* cacheid within the package */
      + 2 * (infos->apicid / legacy_max_log_proc); /* add 2 cache per previous package */
      }
    } else if (cpuid_type == hygon) {
      if (infos->cpufamilynumber == 0x18
      && cache->level == 3 && cache->nbthreads_sharing == 6) {
        /* Hygon family 0x18 always shares L3 between 8 APIC ids,
         * even when only 6 APIC ids are enabled and reported in nbthreads_sharing
         * (on 24-core CPUs).
         */
        // 设置缓存的 cacheid 属性，用于标识在同一包中的缓存
        cache->cacheid = infos->apicid / 8;
      }
    }
  }

  // 检查 apicid 是否在 apicid_set 中，如果是则将 apicid_unique 设置为 0，否则将 apicid 加入 apicid_set
  if (hwloc_bitmap_isset(data->apicid_set, infos->apicid))
    data->apicid_unique = 0;
  else
    hwloc_bitmap_set(data->apicid_set, infos->apicid);
    # 添加 CPU 信息到给定的对象中
    static void
    hwloc_x86_add_cpuinfos(hwloc_obj_t obj, struct procinfo *info, int replace)
    {
      # 创建一个字符串数组，用于存储数字
      char number[12];
      # 如果 CPU 厂商信息存在，则添加到对象的信息中
      if (info->cpuvendor[0])
        hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUVendor", info->cpuvendor, replace);
      # 将 CPU 家族编号转换为字符串，并添加到对象的信息中
      snprintf(number, sizeof(number), "%u", info->cpufamilynumber);
      hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUFamilyNumber", number, replace);
      # 将 CPU 型号编号转换为字符串，并添加到对象的信息中
      snprintf(number, sizeof(number), "%u", info->cpumodelnumber);
      hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUModelNumber", number, replace);
      # 如果 CPU 型号信息存在，则添加到对象的信息中
      if (info->cpumodel[0]) {
        const char *c = info->cpumodel;
        while (*c == ' ')
          c++;
        hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUModel", c, replace);
      }
      # 将 CPU 步进信息转换为字符串，并添加到对象的信息中
      snprintf(number, sizeof(number), "%u", info->cpustepping);
      hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUStepping", number, replace);
    }

    # 添加组信息到拓扑结构中
    static void
    hwloc_x86_add_groups(hwloc_topology_t topology,
                 struct procinfo *infos,
                 unsigned nbprocs,
                 hwloc_bitmap_t remaining_cpuset,
                 unsigned type,
                 const char *subtype,
                 unsigned kind,
                 int dont_merge)
    {
      # 创建对象的 CPU 集合
      hwloc_bitmap_t obj_cpuset;
      # 创建对象
      hwloc_obj_t obj;
      unsigned i, j;

      # 遍历剩余 CPU 集合
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
        # 获取包 ID 和类型 ID
        unsigned packageid = infos[i].ids[PKG];
        unsigned id = infos[i].ids[type];

        # 如果类型 ID 为 -1，则清除 CPU 集合中的对应位，并继续下一次循环
        if (id == (unsigned)-1) {
          hwloc_bitmap_clr(remaining_cpuset, i);
          continue;
        }

        # 分配对象的 CPU 集合
        obj_cpuset = hwloc_bitmap_alloc();
        # 遍历处理相同包 ID 和类型 ID 的 CPU
        for (j = i; j < nbprocs; j++) {
          if (infos[j].ids[type] == (unsigned) -1) {
        hwloc_bitmap_clr(remaining_cpuset, j);
        continue;
          }

          if (infos[j].ids[PKG] == packageid && infos[j].ids[type] == id) {
        hwloc_bitmap_set(obj_cpuset, j);
        hwloc_bitmap_clr(remaining_cpuset, j);
          }
        }

        # 分配并设置对象
        obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, id);
        obj->cpuset = obj_cpuset;
        obj->subtype = strdup(subtype);
    # 设置对象的属性中的组的类型
    obj->attr->group.kind = kind;
    # 设置对象的属性中的组的合并标志
    obj->attr->group.dont_merge = dont_merge;
    # 打印调试信息，显示操作系统、ID和对象的 CPU 集合
    hwloc_debug_2args_bitmap("os %s %u has cpuset %s\n",
                 subtype, id, obj_cpuset);
    # 根据 CPU 集合将对象插入拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, obj, "x86:group");
  /* 分析存储在 infos 中的信息，并相应地构建/注释拓扑级别 */
static void summarize(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags)
{
  struct hwloc_topology *topology = backend->topology;  // 获取拓扑结构
  struct hwloc_x86_backend_data_s *data = backend->private_data;  // 获取私有数据
  unsigned nbprocs = data->nbprocs;  // 获取处理器数量
  hwloc_bitmap_t complete_cpuset = hwloc_bitmap_alloc();  // 分配一个完整的 CPU 集合
  unsigned i, j, l, level;  // 定义循环变量
  int one = -1;  // 初始化一个变量
  hwloc_bitmap_t remaining_cpuset;  // 定义剩余的 CPU 集合
  int gotnuma = 0;  // 初始化一个变量
  int fulldiscovery = (flags & HWLOC_X86_DISC_FLAG_FULL);  // 根据标志位设置 fulldiscovery 变量

#ifdef HWLOC_DEBUG
  hwloc_debug("\nSummary of x86 CPUID topology:\n");  // 输出调试信息
  for(i=0; i<nbprocs; i++) {
    hwloc_debug("PU %u present=%u apicid=%u on PKG %d CORE %d DIE %d NODE %d\n",
                i, infos[i].present, infos[i].apicid,
                infos[i].ids[PKG], infos[i].ids[CORE], infos[i].ids[DIE], infos[i].ids[NODE]);  // 输出处理器的相关信息
  }
  hwloc_debug("\n");
#endif

  for (i = 0; i < nbprocs; i++)
    if (infos[i].present) {  // 如果处理器存在
      hwloc_bitmap_set(complete_cpuset, i);  // 设置完整的 CPU 集合
      one = i;  // 更新 one 变量
    }

  if (one == -1) {  // 如果 one 仍然为 -1
    hwloc_bitmap_free(complete_cpuset);  // 释放完整的 CPU 集合
    return;  // 返回
  }

  remaining_cpuset = hwloc_bitmap_alloc();  // 分配剩余的 CPU 集合

  /* 理想情况下，当 fulldiscovery=0 时，我们可以添加任何尚不存在的对象。
   * 但是如果 x86 和本地后端由于一个有 bug 而不一致呢？该信任哪一个？
   * 我们目前只添加缺失的缓存，并为其他现有对象添加注释。
   */

  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_PACKAGE)) {  // 检查是否保留对象类型
    /* 寻找 package */
    hwloc_obj_t package;  // 定义 package 对象

    hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);  // 复制完整的 CPU 集合到剩余的 CPU 集合
    while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {  // 循环处理剩余的 CPU 集合
      if (fulldiscovery) {  // 如果 fulldiscovery 为真
    unsigned packageid = infos[i].ids[PKG];  // 获取 package 的 ID
    hwloc_bitmap_t package_cpuset = hwloc_bitmap_alloc();  // 分配 package 的 CPU 集合
    # 遍历从 i 到 nbprocs 的索引
    for (j = i; j < nbprocs; j++) {
      # 如果第 j 个处理器的包 ID 等于给定的 packageid
      if (infos[j].ids[PKG] == packageid) {
        # 将第 j 个处理器的位图设置到 package_cpuset 中
        hwloc_bitmap_set(package_cpuset, j);
        # 将第 j 个处理器的位图从 remaining_cpuset 中清除
        hwloc_bitmap_clr(remaining_cpuset, j);
      }
    }
    # 分配并设置 package 对象
    package = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PACKAGE, packageid);
    package->cpuset = package_cpuset;

    # 添加处理器信息到 package 对象
    hwloc_x86_add_cpuinfos(package, &infos[i], 0);

    # 打印调试信息
    hwloc_debug_1arg_bitmap("os package %u has cpuset %s\n",
                packageid, package_cpuset);
    # 根据位图插入对象到拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, package, "x86:package");

      } else {
    # 为先前存在的包添加注释
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    hwloc_bitmap_set(set, i);
    # 获取覆盖给定类型的下一个对象
    package = hwloc_get_next_obj_covering_cpuset_by_type(topology, set, HWLOC_OBJ_PACKAGE, NULL);
    hwloc_bitmap_free(set);
    if (package) {
      # 如果找到了包，添加处理器信息并从 remaining_cpuset 中清除 package 的位图
      hwloc_x86_add_cpuinfos(package, &infos[i], 1);
      hwloc_bitmap_andnot(remaining_cpuset, remaining_cpuset, package->cpuset);
    } else {
      # 如果没有找到包，添加处理器信息到根对象并跳出循环
      hwloc_x86_add_cpuinfos(hwloc_get_root_obj(topology), &infos[i], 1);
      break;
    }
      }
    }
  }

  # 在包内查找 NUMA 节点（无法被过滤）
  if (fulldiscovery && (flags & HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES)) {
    hwloc_bitmap_t node_cpuset;
    hwloc_obj_t node;

    # FIXME: 如果根对象内有内存，将其分成 NUMA 节点？

    # 复制 remaining_cpuset 到 complete_cpuset
    hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
    # 当 remaining_cpuset 中还有位图时循环
    while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
      # 获取处理器所在的包 ID 和 NUMA 节点 ID
      unsigned packageid = infos[i].ids[PKG];
      unsigned nodeid = infos[i].ids[NODE];

      # 如果 NUMA 节点 ID 为 -1，从 remaining_cpuset 中清除该位图并继续下一次循环
      if (nodeid == (unsigned)-1) {
        hwloc_bitmap_clr(remaining_cpuset, i);
        continue;
      }

      # 分配一个新的位图给 node_cpuset
      node_cpuset = hwloc_bitmap_alloc();
      # 遍历从 i 到 nbprocs 的索引
      for (j = i; j < nbprocs; j++) {
    # 如果节点的 ID 为 -1，则将其从 remaining_cpuset 中清除，并继续下一个节点
    if (infos[j].ids[NODE] == (unsigned) -1) {
      hwloc_bitmap_clr(remaining_cpuset, j);
      continue;
    }

        # 如果节点的包 ID 和节点 ID 与给定的 packageid 和 nodeid 相匹配
        if (infos[j].ids[PKG] == packageid && infos[j].ids[NODE] == nodeid) {
          # 设置节点的 CPU 集合
          hwloc_bitmap_set(node_cpuset, j);
          # 从 remaining_cpuset 中清除节点
          hwloc_bitmap_clr(remaining_cpuset, j);
        }
      }
      # 分配并设置 NUMA 节点对象
      node = hwloc_alloc_setup_object(topology, HWLOC_OBJ_NUMANODE, nodeid);
      node->cpuset = node_cpuset;
      node->nodeset = hwloc_bitmap_alloc();
      hwloc_bitmap_set(node->nodeset, nodeid);
      # 打印调试信息
      hwloc_debug_1arg_bitmap("os node %u has cpuset %s\n",
          nodeid, node_cpuset);
      # 将节点对象插入拓扑结构
      hwloc__insert_object_by_cpuset(topology, NULL, node, "x86:numa");
      # 增加 NUMA 节点计数
      gotnuma++;
    }
  }

  # 检查是否需要保留对象类型为 HWLOC_OBJ_GROUP
  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP)) {
    if (fulldiscovery) {
      # 在包内查找 AMD Compute 单元
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
               UNIT, "Compute Unit",
               HWLOC_GROUP_KIND_AMD_COMPUTE_UNIT, 0);
      # 在包内查找 Intel 模块
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
               MODULE, "Module",
               HWLOC_GROUP_KIND_INTEL_MODULE, 0);
      # 在包内查找 Intel Tile
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
               TILE, "Tile",
               HWLOC_GROUP_KIND_INTEL_TILE, 0);

      # 查找未知对象
      if (infos[one].otherids) {
    # 遍历infos数组中的元素，从levels-1开始，递减直到0
    for (level = infos[one].levels-1; level <= infos[one].levels-1; level--) {
      # 如果当前元素的otherids[level]不是UINT_MAX
      if (infos[one].otherids[level] != UINT_MAX) {
        # 创建一个未知的cpuset和对象
        hwloc_bitmap_t unknown_cpuset;
        hwloc_obj_t unknown_obj;

        # 复制complete_cpuset到remaining_cpuset
        hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
        # 当remaining_cpuset中还有元素时
        while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
          # 获取当前元素的otherids[level]
          unsigned unknownid = infos[i].otherids[level];

          # 分配一个未知的cpuset
          unknown_cpuset = hwloc_bitmap_alloc();
          # 遍历infos数组，找到具有相同unknownid的元素，设置其cpuset并清除remaining_cpuset中的对应位
          for (j = i; j < nbprocs; j++) {
            if (infos[j].otherids[level] == unknownid) {
              hwloc_bitmap_set(unknown_cpuset, j);
              hwloc_bitmap_clr(remaining_cpuset, j);
            }
          }
          # 分配一个未知的对象，并设置其cpuset、kind和subkind
          unknown_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, unknownid);
          unknown_obj->cpuset = unknown_cpuset;
          unknown_obj->attr->group.kind = HWLOC_GROUP_KIND_INTEL_EXTTOPOENUM_UNKNOWN;
          unknown_obj->attr->group.subkind = level;
          # 输出未知对象的信息
          hwloc_debug_2args_bitmap("os unknown%u %u has cpuset %s\n",
                       level, unknownid, unknown_cpuset);
          # 将未知对象插入拓扑结构中
          hwloc__insert_object_by_cpuset(topology, NULL, unknown_obj, "x86:group:unknown");
        }
      }
    }
  }

  # 如果拓扑结构中包含HWLOC_OBJ_DIE类型的对象
  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_DIE)) {
    # 在package内部查找Intel Dies
    if (fulldiscovery) {
      # 分配一个die_cpuset和对象
      hwloc_bitmap_t die_cpuset;
      hwloc_obj_t die;

      # 复制complete_cpuset到remaining_cpuset
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      # 当remaining_cpuset中还有元素时
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
        # 获取当前元素的packageid和dieid
        unsigned packageid = infos[i].ids[PKG];
        unsigned dieid = infos[i].ids[DIE];

        # 如果dieid为-1，则清除remaining_cpuset中的对应位并继续下一次循环
        if (dieid == (unsigned) -1) {
          hwloc_bitmap_clr(remaining_cpuset, i);
          continue;
        }

        # 分配一个die_cpuset
        die_cpuset = hwloc_bitmap_alloc();
    # 遍历从 i 到 nbprocs 的索引
    for (j = i; j < nbprocs; j++) {
      # 如果 infos[j] 的 DIE ID 为 -1，则将 remaining_cpuset 中对应位置清零，并继续下一次循环
      if (infos[j].ids[DIE] == (unsigned) -1) {
        hwloc_bitmap_clr(remaining_cpuset, j);
        continue;
      }

      # 如果 infos[j] 的 PKG ID 和 DIE ID 与给定的 packageid 和 dieid 相同
      if (infos[j].ids[PKG] == packageid && infos[j].ids[DIE] == dieid) {
        # 将 die_cpuset 中对应位置设置为 1，remaining_cpuset 中对应位置清零
        hwloc_bitmap_set(die_cpuset, j);
        hwloc_bitmap_clr(remaining_cpuset, j);
      }
    }
    # 分配并设置一个 DIE 对象，将其 cpuset 设置为 die_cpuset
    die = hwloc_alloc_setup_object(topology, HWLOC_OBJ_DIE, dieid);
    die->cpuset = die_cpuset;
    # 打印调试信息，显示 DIE 对象的 cpuset
    hwloc_debug_1arg_bitmap("os die %u has cpuset %s\n",
                dieid, die_cpuset);
    # 将 DIE 对象插入拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, die, "x86:die");
      }
    }
  }

  # 检查是否需要保留 CORE 对象类型
  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_CORE)) {
    # 寻找 CORE 对象
    if (fulldiscovery) {
      # 复制 remaining_cpuset 到 complete_cpuset
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      # 当 remaining_cpuset 中还有未处理的位时
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
    # 获取当前位对应的 packageid、nodeid 和 coreid
    unsigned packageid = infos[i].ids[PKG];
    unsigned nodeid = infos[i].ids[NODE];
    unsigned coreid = infos[i].ids[CORE];

    # 如果 coreid 为 -1，则将 remaining_cpuset 中对应位置清零，并继续下一次循环
    if (coreid == (unsigned) -1) {
      hwloc_bitmap_clr(remaining_cpuset, i);
      continue;
    }

    # 分配并设置一个 CORE 对象，将其 cpuset 设置为 core_cpuset
    core = hwloc_alloc_setup_object(topology, HWLOC_OBJ_CORE, coreid);
    core->cpuset = core_cpuset;
    # 打印调试信息，显示 CORE 对象的 cpuset
    hwloc_debug_1arg_bitmap("os core %u has cpuset %s\n",
                coreid, core_cpuset);
    # 将 CORE 对象插入拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, core, "x86:core");
      }
    }
  }

  # 寻找 PU 对象（无法被过滤掉）
  if (fulldiscovery) {
    # 打印调试信息，显示 CPU cpusets
    hwloc_debug("%s", "\n\n * CPU cpusets *\n\n");
    for (i=0; i<nbprocs; i++)
      if(infos[i].present) { /* Only add present PU. We don't know if others actually exist */
       // 如果处理器单元存在，则添加到拓扑结构中，不确定其他处理器单元是否存在
       struct hwloc_obj *obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, i);
       // 为处理器单元对象分配并设置相关信息
       obj->cpuset = hwloc_bitmap_alloc();
       // 将处理器单元的编号加入到 cpuset 中
       hwloc_bitmap_only(obj->cpuset, i);
       // 输出调试信息，显示处理器单元的 cpuset
       hwloc_debug_1arg_bitmap("PU %u has cpuset %s\n", i, obj->cpuset);
       // 将处理器单元对象插入到拓扑结构中
       hwloc__insert_object_by_cpuset(topology, NULL, obj, "x86:pu");
     }
  }

  /* Look for caches */
  /* First find max level */
  level = 0;
  for (i = 0; i < nbprocs; i++)
    for (j = 0; j < infos[i].numcaches; j++)
      if (infos[i].cache[j].level > level)
        level = infos[i].cache[j].level;
  // 找到缓存的最大级别
  while (level > 0) {
    hwloc_obj_cache_type_t type;
    HWLOC_BUILD_ASSERT(HWLOC_OBJ_CACHE_DATA == HWLOC_OBJ_CACHE_UNIFIED+1);
    HWLOC_BUILD_ASSERT(HWLOC_OBJ_CACHE_INSTRUCTION == HWLOC_OBJ_CACHE_DATA+1);
    for (type = HWLOC_OBJ_CACHE_UNIFIED; type <= HWLOC_OBJ_CACHE_INSTRUCTION; type++) {
      /* Look for caches of that type at level level */
      // 在指定级别查找特定类型的缓存
      hwloc_obj_type_t otype;
      hwloc_obj_t cache;

      otype = hwloc_cache_type_by_depth_type(level, type);
      if (otype == HWLOC_OBJ_TYPE_NONE)
    continue;
      if (!hwloc_filter_check_keep_object_type(topology, otype))
    continue;

      // 复制完整的 cpuset 到剩余 cpuset
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
    hwloc_bitmap_t puset;

    for (l = 0; l < infos[i].numcaches; l++) {
      if (infos[i].cache[l].level == level && infos[i].cache[l].type == type)
        break;
    }
    if (l == infos[i].numcaches) {
      /* no cache Llevel of that type in i */
      // 如果处理器 i 中没有该类型的缓存，则从剩余 cpuset 中移除 i
      hwloc_bitmap_clr(remaining_cpuset, i);
      continue;
    }

    // 创建一个新的 cpuset
    puset = hwloc_bitmap_alloc();
    // 将处理器 i 加入到新的 cpuset 中
    hwloc_bitmap_set(puset, i);
    // 根据指定类型和 cpuset 查找下一个对象
    cache = hwloc_get_next_obj_covering_cpuset_by_type(topology, puset, otype, NULL);
    // 释放新的 cpuset
    hwloc_bitmap_free(puset);
    # 如果存在缓存
    if (cache) {
      # 在PU上方找到缓存，如果还没有"Inclusive"属性，则添加该属性
      if (!hwloc_obj_get_info_by_name(cache, "Inclusive"))
        hwloc_obj_add_info(cache, "Inclusive", infos[i].cache[l].inclusive ? "1" : "0");
      # 从剩余的CPU集中排除缓存的CPU集
      hwloc_bitmap_andnot(remaining_cpuset, remaining_cpuset, cache->cpuset);
    }
      }
    }
    # 减少层级
    level--;
  }

  # 如果是KNL并且L2被禁用，添加tiles而不是L2（待解决的问题）

  # 释放剩余CPU集和完整CPU集的内存
  hwloc_bitmap_free(remaining_cpuset);
  hwloc_bitmap_free(complete_cpuset);

  # 如果存在NUMA节点，则设置拓扑结构的发现支持中的NUMA为1
  if (gotnuma)
    topology->support.discovery->numa = 1;
}

static int
look_procs(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags,
       unsigned highest_cpuid, unsigned highest_ext_cpuid, unsigned *features, enum cpuid_type cpuid_type,
       int (*get_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags),
       int (*set_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags),
           hwloc_bitmap_t restrict_set)
{
  // 获取后端私有数据
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  // 获取拓扑结构
  struct hwloc_topology *topology = backend->topology;
  // 获取处理器数量
  unsigned nbprocs = data->nbprocs;
  // 原始 CPU 集合
  hwloc_bitmap_t orig_cpuset = NULL;
  // CPU 集合
  hwloc_bitmap_t set = NULL;
  unsigned i;

  // 如果没有 CPUID dump 路径
  if (!data->src_cpuiddump_path) {
    // 分配原始 CPU 集合
    orig_cpuset = hwloc_bitmap_alloc();
    // 获取 CPU 绑定
    if (get_cpubind(topology, orig_cpuset, HWLOC_CPUBIND_STRICT)) {
      // 释放原始 CPU 集合
      hwloc_bitmap_free(orig_cpuset);
      return -1;
    }
    // 分配 CPU 集合
    set = hwloc_bitmap_alloc();
  }

  // 遍历处理器
  for (i = 0; i < nbprocs; i++) {
    struct cpuiddump *src_cpuiddump = NULL;

    // 如果限制集合存在且当前处理器不在绑定掩码内，则跳过
    if (restrict_set && !hwloc_bitmap_isset(restrict_set, i)) {
      /* skip this CPU outside of the binding mask */
      continue;
    }

    // 如果存在 CPUID dump 路径
    if (data->src_cpuiddump_path) {
      // 读取 CPUID dump 数据
      src_cpuiddump = cpuiddump_read(data->src_cpuiddump_path, i);
      if (!src_cpuiddump)
    continue;
    } else {
      // 设置 CPU 集合
      hwloc_bitmap_only(set, i);
      // 调试信息
      hwloc_debug("binding to CPU%u\n", i);
      // 设置 CPU 绑定
      if (set_cpubind(topology, set, HWLOC_CPUBIND_STRICT)) {
    // 调试信息
    hwloc_debug("could not bind to CPU%u: %s\n", i, strerror(errno));
    continue;
      }
    }

    // 查看处理器信息
    look_proc(backend, &infos[i], flags, highest_cpuid, highest_ext_cpuid, features, cpuid_type, src_cpuiddump);

    // 如果存在 CPUID dump 路径，则释放 CPUID dump 数据
    if (data->src_cpuiddump_path) {
      cpuiddump_free(src_cpuiddump);
    }
  }

  // 如果没有 CPUID dump 路径
  if (!data->src_cpuiddump_path) {
    // 恢复 CPU 绑定
    set_cpubind(topology, orig_cpuset, 0);
    // 释放 CPU 集合
    hwloc_bitmap_free(set);
    // 释放原始 CPU 集合
    hwloc_bitmap_free(orig_cpuset);
  }

  // 如果 APICID 唯一，则总结信息
  if (data->apicid_unique) {
    summarize(backend, infos, flags);
    # 如果具有混合特性并且拥有 CPU 种类标志，则执行以下操作
    if (has_hybrid(features) && !(topology->flags & HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS)) {
      # 为 cpukinds 使用混合信息
      hwloc_bitmap_t atomset = hwloc_bitmap_alloc();  # 分配一个新的位图用于存储 IntelAtom 的集合
      hwloc_bitmap_t coreset = hwloc_bitmap_alloc();  # 分配一个新的位图用于存储 IntelCore 的集合
      for(i=0; i<nbprocs; i++) {
        # 如果处理器 i 的混合核心类型为 0x20，则将其添加到 atomset 中
        if (infos[i].hybridcoretype == 0x20)
          hwloc_bitmap_set(atomset, i);
        # 如果处理器 i 的混合核心类型为 0x40，则将其添加到 coreset 中
        else if (infos[i].hybridcoretype == 0x40)
          hwloc_bitmap_set(coreset, i);
      }
      # 如果 IntelAtom 集合不为空，则注册 IntelAtom 集合
      if (!hwloc_bitmap_iszero(atomset)) {
        struct hwloc_info_s infoattr;  # 创建一个 hwloc_info_s 结构体
        infoattr.name = (char *) "CoreType";  # 设置名称为 "CoreType"
        infoattr.value = (char *) "IntelAtom";  # 设置值为 "IntelAtom"
        hwloc_internal_cpukinds_register(topology, atomset, HWLOC_CPUKIND_EFFICIENCY_UNKNOWN, &infoattr, 1, 0);  # 注册 IntelAtom 集合
        # 将 cpuset 给予调用者
      } else {
        hwloc_bitmap_free(atomset);  # 如果 IntelAtom 集合为空，则释放内存
      }
      # 如果 IntelCore 集合不为空，则注册 IntelCore 集合
      if (!hwloc_bitmap_iszero(coreset)) {
        struct hwloc_info_s infoattr;  # 创建一个 hwloc_info_s 结构体
        infoattr.name = (char *) "CoreType";  # 设置名称为 "CoreType"
        infoattr.value = (char *) "IntelCore";  # 设置值为 "IntelCore"
        hwloc_internal_cpukinds_register(topology, coreset, HWLOC_CPUKIND_EFFICIENCY_UNKNOWN, &infoattr, 1, 0);  # 注册 IntelCore 集合
        # 将 cpuset 给予调用者
      } else {
        hwloc_bitmap_free(coreset);  # 如果 IntelCore 集合为空，则释放内存
      }
    }
  }
  # 如果 !data->apicid_unique，则不执行任何操作并返回成功，以便调用者也不执行任何操作
  return 0;
# 如果定义了 HWLOC_FREEBSD_SYS 并且定义了 HAVE_CPUSET_SETID
#if defined HWLOC_FREEBSD_SYS && defined HAVE_CPUSET_SETID
#include <sys/param.h>
#include <sys/cpuset.h>
typedef cpusetid_t hwloc_x86_os_state_t;
static void hwloc_x86_os_state_save(hwloc_x86_os_state_t *state, struct cpuiddump *src_cpuiddump)
{
  if (!src_cpuiddump) {
    /* 临时使所有 CPU 在发现期间可用 */
    cpuset_getid(CPU_LEVEL_CPUSET, CPU_WHICH_PID, -1, state);
    cpuset_setid(CPU_WHICH_PID, -1, 0);
  }
}
static void hwloc_x86_os_state_restore(hwloc_x86_os_state_t *state, struct cpuiddump *src_cpuiddump)
{
  if (!src_cpuiddump) {
    /* 恢复初始 CPU 集合 */
    cpuset_setid(CPU_WHICH_PID, -1, *state);
  }
}
#else /* !defined HWLOC_FREEBSD_SYS || !defined HAVE_CPUSET_SETID */
typedef void * hwloc_x86_os_state_t;
static void hwloc_x86_os_state_save(hwloc_x86_os_state_t *state __hwloc_attribute_unused, struct cpuiddump *src_cpuiddump __hwloc_attribute_unused) { }
static void hwloc_x86_os_state_restore(hwloc_x86_os_state_t *state __hwloc_attribute_unused, struct cpuiddump *src_cpuiddump __hwloc_attribute_unused) { }
#endif /* !defined HWLOC_FREEBSD_SYS || !defined HAVE_CPUSET_SETID */

/* GenuineIntel */
#define INTEL_EBX ('G' | ('e'<<8) | ('n'<<16) | ('u'<<24))
#define INTEL_EDX ('i' | ('n'<<8) | ('e'<<16) | ('I'<<24))
#define INTEL_ECX ('n' | ('t'<<8) | ('e'<<16) | ('l'<<24))

/* AuthenticAMD */
#define AMD_EBX ('A' | ('u'<<8) | ('t'<<16) | ('h'<<24))
#define AMD_EDX ('e' | ('n'<<8) | ('t'<<16) | ('i'<<24))
#define AMD_ECX ('c' | ('A'<<8) | ('M'<<16) | ('D'<<24))

/* HYGON "HygonGenuine" */
#define HYGON_EBX ('H' | ('y'<<8) | ('g'<<16) | ('o'<<24))
#define HYGON_EDX ('n' | ('G'<<8) | ('e'<<16) | ('n'<<24))
#define HYGON_ECX ('u' | ('i'<<8) | ('n'<<16) | ('e'<<24))

/* (Zhaoxin) CentaurHauls */
#define ZX_EBX ('C' | ('e'<<8) | ('n'<<16) | ('t'<<24))
#define ZX_EDX ('a' | ('u'<<8) | ('r'<<16) | ('H'<<24))
#define ZX_ECX ('a' | ('u'<<8) | ('l'<<16) | ('s'<<24))
/* (Zhaoxin) Shanghai */
# 定义 SH_EBX 值为 ASCII 码空格和 'Sh' 字符串的组合
#define SH_EBX (' ' | (' '<<8) | ('S'<<16) | ('h'<<24))
# 定义 SH_EDX 值为 ASCII 码 'angh' 字符串的组合
#define SH_EDX ('a' | ('n'<<8) | ('g'<<16) | ('h'<<24))
# 定义 SH_ECX 值为 ASCII 码 'ai  ' 字符串的组合
#define SH_ECX ('a' | ('i'<<8) | (' '<<16) | (' '<<24))

# 当 nbprocs=1 且不支持绑定时，用于模拟 cpubind 的函数
static int fake_get_cpubind(hwloc_topology_t topology __hwloc_attribute_unused,
                hwloc_cpuset_t set __hwloc_attribute_unused,
                int flags __hwloc_attribute_unused)
{
  return 0;
}
# 当 nbprocs=1 且不支持绑定时，用于模拟设置 cpubind 的函数
static int fake_set_cpubind(hwloc_topology_t topology __hwloc_attribute_unused,
                hwloc_const_cpuset_t set __hwloc_attribute_unused,
                int flags __hwloc_attribute_unused)
{
  return 0;
}

# x86 架构的查找函数，接受一个 hwloc_backend 结构和标志参数
static
int hwloc_look_x86(struct hwloc_backend *backend, unsigned long flags)
  # 获取指向 x86 后端数据的指针
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  # 获取指向拓扑结构的指针
  struct hwloc_topology *topology = backend->topology;
  # 获取处理器数量
  unsigned nbprocs = data->nbprocs;
  # 定义寄存器值
  unsigned eax, ebx, ecx = 0, edx;
  # 定义循环变量
  unsigned i;
  # 定义最高 cpuid 值
  unsigned highest_cpuid;
  # 定义最高扩展 cpuid 值
  unsigned highest_ext_cpuid;
  # 存储 cpuid 特性的数组
  unsigned features[19] = { 0 };
  # 进程信息指针
  struct procinfo *infos = NULL;
  # 定义 cpuid 类型
  enum cpuid_type cpuid_type = unknown;
  # x86 操作系统状态
  hwloc_x86_os_state_t os_state;
  # 绑定钩子
  struct hwloc_binding_hooks hooks;
  # 拓扑支持
  struct hwloc_topology_support support;
  # 内存绑定支持
  struct hwloc_topology_membind_support memsupport __hwloc_attribute_unused;
  # 获取 CPU 绑定的函数指针
  int (*get_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags) = NULL;
  # 设置 CPU 绑定的函数指针
  int (*set_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags) = NULL;
  # 限制集合
  hwloc_bitmap_t restrict_set = NULL;
  # CPUID 数据转储
  struct cpuiddump *src_cpuiddump = NULL;
  # 返回值
  int ret = -1;

  # 检查绑定是否有效
  memset(&hooks, 0, sizeof(hooks));
  support.membind = &memsupport;
  # 设置本地绑定钩子
  hwloc_set_native_binding_hooks(&hooks, &support);
  # 如果存在 CPUID 转储路径
  if (data->src_cpuiddump_path) {
    # 从转储中读取 CPUID
    src_cpuiddump = cpuiddump_read(data->src_cpuiddump_path, 0);
    if (!src_cpuiddump)
      # 转储读取失败，跳转到结束
      goto out;
  } else {
    # 使用真实硬件
    if (hooks.get_thisthread_cpubind && hooks.set_thisthread_cpubind) {
      # 获取当前线程的 CPU 绑定函数
      get_cpubind = hooks.get_thisthread_cpubind;
      # 设置当前线程的 CPU 绑定函数
      set_cpubind = hooks.set_thisthread_cpubind;
    } else if (hooks.get_thisproc_cpubind && hooks.set_thisproc_cpubind) {
      /* 如果被多线程程序调用，我们将为每个线程恢复原始的进程绑定，而不是它们自己的原始线程绑定。
       * 参见问题＃158。
       */
      get_cpubind = hooks.get_thisproc_cpubind;
      set_cpubind = hooks.set_thisproc_cpubind;
    } else {
      /* 如果存在多个处理器单元，则需要绑定支持 */
      if (nbprocs > 1)
        goto out;
      get_cpubind = fake_get_cpubind;
      set_cpubind = fake_set_cpubind;
    }
  }

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    restrict_set = hwloc_bitmap_alloc();
    if (!restrict_set)
      goto out;
    if (hooks.get_thisproc_cpubind)
      hooks.get_thisproc_cpubind(topology, restrict_set, 0);
    else if (hooks.get_thisthread_cpubind)
      hooks.get_thisthread_cpubind(topology, restrict_set, 0);
    if (hwloc_bitmap_iszero(restrict_set)) {
      hwloc_bitmap_free(restrict_set);
      restrict_set = NULL;
    }
  }

  if (!src_cpuiddump && !hwloc_have_x86_cpuid())
    goto out;

  infos = calloc(nbprocs, sizeof(struct procinfo));
  if (NULL == infos)
    goto out;
  for (i = 0; i < nbprocs; i++) {
    infos[i].ids[PKG] = (unsigned) -1;
    infos[i].ids[CORE] = (unsigned) -1;
    infos[i].ids[NODE] = (unsigned) -1;
    infos[i].ids[UNIT] = (unsigned) -1;
    infos[i].ids[TILE] = (unsigned) -1;
    infos[i].ids[MODULE] = (unsigned) -1;
    infos[i].ids[DIE] = (unsigned) -1;
  }

  eax = 0x00;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  highest_cpuid = eax;
  if (ebx == INTEL_EBX && ecx == INTEL_ECX && edx == INTEL_EDX)
    cpuid_type = intel;
  else if (ebx == AMD_EBX && ecx == AMD_ECX && edx == AMD_EDX)
    cpuid_type = amd;
  else if ((ebx == ZX_EBX && ecx == ZX_ECX && edx == ZX_EDX)
       || (ebx == SH_EBX && ecx == SH_ECX && edx == SH_EDX))
    # 设置 CPU 型号为兆芯
    cpuid_type = zhaoxin;
  # 如果 CPUID 的值符合海光处理器的标识，则设置 CPU 型号为海光
  else if (ebx == HYGON_EBX && ecx == HYGON_ECX && edx == HYGON_EDX)
    cpuid_type = hygon;

  # 输出最高 CPUID 值和 CPUID 类型
  hwloc_debug("highest cpuid %x, cpuid type %u\n", highest_cpuid, cpuid_type);
  # 如果最高 CPUID 值小于 0x01，则跳转到 out_with_infos 标签处
  if (highest_cpuid < 0x01) {
      goto out_with_infos;
  }

  # 设置 EAX 寄存器的值为 0x01
  eax = 0x01;
  # 从 src_cpuiddump 中获取 CPUID 信息，并存入相应寄存器
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  # 将 EDX 寄存器的值存入 features 数组的第一个元素，将 ECX 寄存器的值存入 features 数组的第五个元素
  features[0] = edx;
  features[4] = ecx;

  # 设置 EAX 寄存器的值为 0x80000000
  eax = 0x80000000;
  # 从 src_cpuiddump 中获取 CPUID 信息，并存入相应寄存器
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  # 将 EAX 寄存器的值存入 highest_ext_cpuid 变量
  highest_ext_cpuid = eax;

  # 输出最高扩展 CPUID 值
  hwloc_debug("highest extended cpuid %x\n", highest_ext_cpuid);

  # 如果最高 CPUID 值大于等于 0x7
  if (highest_cpuid >= 0x7) {
    # 设置 EAX 寄存器的值为 0x7，ECX 寄存器的值为 0
    eax = 0x7;
    ecx = 0;
    # 从 src_cpuiddump 中获取 CPUID 信息，并存入相应寄存器
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    # 将 EBX 寄存器的值存入 features 数组的第九个元素，将 EDX 寄存器的值存入 features 数组的第十九个元素
    features[9] = ebx;
    features[18] = edx;
  }

  # 如果 CPU 型号不是英特尔，并且最高扩展 CPUID 值大于等于 0x80000001
  if (cpuid_type != intel && highest_ext_cpuid >= 0x80000001) {
    # 设置 EAX 寄存器的值为 0x80000001
    eax = 0x80000001;
    # 从 src_cpuiddump 中获取 CPUID 信息，并存入相应寄存器
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    # 将 EDX 寄存器的值存入 features 数组的第一个元素，将 ECX 寄存器的值存入 features 数组的第六个元素
    features[1] = edx;
    features[6] = ecx;
  }

  # 保存 x86 操作系统状态
  hwloc_x86_os_state_save(&os_state, src_cpuiddump);

  # 调用 look_procs 函数，传入相应参数
  ret = look_procs(backend, infos, flags,
           highest_cpuid, highest_ext_cpuid, features, cpuid_type,
           get_cpubind, set_cpubind, restrict_set);
  # 如果返回值为假，则跳转到 out_with_os_state 标签处
  if (!ret)
    /* success, we're done */
    goto out_with_os_state;

  # 如果处理器数量为 1
  if (nbprocs == 1) {
    /* only one processor, no need to bind */
    # 调用 look_proc 函数，传入相应参数
    look_proc(backend, &infos[0], flags, highest_cpuid, highest_ext_cpuid, features, cpuid_type, src_cpuiddump);
    # 对处理器信息进行总结
    summarize(backend, infos, flags);
    # 返回值设为 0
    ret = 0;
  }
out_with_os_state:
  # 保存当前操作系统状态
  hwloc_x86_os_state_restore(&os_state, src_cpuiddump);

out_with_infos:
  # 如果 infos 不为空，则释放内存
  if (NULL != infos) {
    for (i = 0; i < nbprocs; i++) {
      free(infos[i].cache);
      free(infos[i].otherids);
    }
    free(infos);
  }

out:
  # 释放 restrict_set 占用的内存
  hwloc_bitmap_free(restrict_set);
  # 如果 src_cpuiddump 不为空，则释放其占用的内存
  if (src_cpuiddump)
    cpuiddump_free(src_cpuiddump);
  # 返回 ret 变量的值
  return ret;
}

static int
hwloc_x86_discover(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  # 获取指向 backend 的私有数据的指针
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  # 获取指向 topology 的指针
  struct hwloc_topology *topology = backend->topology;
  # 初始化 flags 变量
  unsigned long flags = 0;
  # 初始化 alreadypus 变量
  int alreadypus = 0;
  # 初始化 ret 变量
  int ret;

  # 断言当前的探测阶段是 CPU
  assert(dstatus->phase == HWLOC_DISC_PHASE_CPU);

  # 如果拓扑标记包含 HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING) {
    /* TODO: Things would work if there's a single PU, no need to rebind */
    # 返回 0
    return 0;
  }

  # 如果环境变量中包含 HWLOC_X86_TOPOEXT_NUMANODES
  if (getenv("HWLOC_X86_TOPOEXT_NUMANODES")) {
    # 设置 flags 变量的相应位
    flags |= HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES;
  }

  # 如果定义了 RUNNING_ON_VALGRIND 并且 data->src_cpuiddump_path 为空
  if (RUNNING_ON_VALGRIND && !data->src_cpuiddump_path) {
    # 输出错误信息并返回 0
    fprintf(stderr, "hwloc x86 backend cannot work under Valgrind, disabling.\n"
        "May be reenabled by dumping CPUIDs with hwloc-gather-cpuid\n"
        "and reloading them under Valgrind with HWLOC_CPUID_PATH.\n");
    return 0;
  }

  # 如果 data->src_cpuiddump_path 不为空
  if (data->src_cpuiddump_path) {
    # 断言 data->nbprocs 大于 0
    assert(data->nbprocs > 0); /* enforced by hwloc_x86_component_instantiate() */
    # 设置拓扑支持的发现 PU 标记为 1
    topology->support.discovery->pu = 1;
  } else {
    # 获取处理器数量
    int nbprocs = hwloc_fallback_nbprocessors(HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE);
    # 如果处理器数量大于等于 1
    if (nbprocs >= 1)
      # 设置拓扑支持的发现 PU 标记为 1
      topology->support.discovery->pu = 1;
    else
      # 否则设置处理器数量为 1
      nbprocs = 1;
    # 将处理器数量转换为无符号整数并赋值给 data->nbprocs
    data->nbprocs = (unsigned) nbprocs;
  }

  # 如果拓扑的第一层第一个对象的 cpuset 不为空
  if (topology->levels[0][0]->cpuset) {
    /* somebody else discovered things, reconnect levels so that we can look at them */
    # 重新连接拓扑层级
    hwloc_topology_reconnect(topology, 0);
    # 如果拓扑结构的层级数为2，并且第二层的对象数等于数据中的处理器数
    if (topology->nb_levels == 2 && topology->level_nbobjects[1] == data->nbprocs) {
      # 如果只有处理器被发现，尽可能完整地使用其他内容完成拓扑结构
      alreadypus = 1;
      # 跳转到完整发现的部分
      goto fulldiscovery;
    }

    # 如果添加了多种对象类型，无法轻松完成，只进行部分发现
    ret = hwloc_look_x86(backend, flags);
    if (ret)
      # 向拓扑结构的第一层第一个对象添加信息，标记为“Backend”、“x86”
      hwloc_obj_add_info(topology->levels[0][0], "Backend", "x86");
    # 返回0，表示成功
    return 0;
  } else {
    # 拓扑结构为空，进行初始化
    hwloc_alloc_root_sets(topology->levels[0][0]);
  }
# 定义函数fulldiscovery
fulldiscovery:
  # 如果hwloc_look_x86函数返回值小于0
  if (hwloc_look_x86(backend, flags | HWLOC_X86_DISC_FLAG_FULL) < 0) {
    # 如果失败，创建处理器单元
    if (!alreadypus)
      hwloc_setup_pu_level(topology, data->nbprocs);
  }
  # 在topology的第一级添加信息"Backend"，值为"x86"
  hwloc_obj_add_info(topology->levels[0][0], "Backend", "x86");
  # 如果src_cpuiddump_path为空
  if (!data->src_cpuiddump_path) { /* CPUID dump works for both x86 and x86_64 */
    # 如果系统支持uname函数
    # 添加uname信息到topology
    hwloc_add_uname_info(topology, NULL); /* we already know is_thissystem() is true */
  } else {
    # 如果系统不支持uname函数
    # 手动设置"Architecture"信息
    # 如果系统支持x86_64架构
    # 在topology的第一级添加信息"Architecture"，值为"x86_64"
    # 如果系统不支持x86_64架构
    # 在topology的第一级添加信息"Architecture"，值为"x86"
  }
  # 返回1
  return 1;
}

# 定义函数hwloc_x86_check_cpuiddump_input
static int
hwloc_x86_check_cpuiddump_input(const char *src_cpuiddump_path, hwloc_bitmap_t set) {
  # 如果系统不是Windows系统或者不是MinGW32或者不是Cygwin
  # 定义dirent结构体指针dirent
  # 定义DIR指针dir
  # 定义字符串指针path
  # 定义文件指针file
  # 定义字符数组line，大小为32
  # 打开src_cpuiddump_path目录
  dir = opendir(src_cpuiddump_path);
  # 如果打开失败，返回-1
  if (!dir) 
    return -1;
  # 为path分配内存
  path = malloc(strlen(src_cpuiddump_path) + strlen("/hwloc-cpuid-info") + 1);
  # 如果分配失败，释放dir并返回
  if (!path)
    goto out_with_dir;
  # 将路径拼接成path
  sprintf(path, "%s/hwloc-cpuid-info", src_cpuiddump_path);
  # 以只读方式打开文件
  file = fopen(path, "r");
  # 如果打开失败，输出错误信息并释放path
  if (!file) {
    fprintf(stderr, "Couldn't open dumped cpuid summary %s\n", path);
    goto out_with_path;
  }
  # 读取文件的一行
  if (!fgets(line, sizeof(line), file)) {
    fprintf(stderr, "Found read dumped cpuid summary in %s\n", path);
    fclose(file);
    goto out_with_path;
  }
  # 关闭文件
  fclose(file);
  # 如果读取的行不是"Architecture: x86\n"，输出错误信息并释放path
  if (strcmp(line, "Architecture: x86\n")) {
    fprintf(stderr, "Found non-x86 dumped cpuid summary in %s: %s\n", path, line);
    goto out_with_path;
  }
  # 释放path
  free(path);
  # 遍历目录中的文件
  while ((dirent = readdir(dir)) != NULL) {
    # 如果文件名以"pu"开头
    if (!strncmp(dirent->d_name, "pu", 2)) {
      # 解析出PU的索引
      char *end;
      unsigned long idx = strtoul(dirent->d_name+2, &end, 10);
      # 如果解析成功，设置对应的位
      if (!*end)
        hwloc_bitmap_set(set, idx);
      else
    # 打印错误信息，忽略在指定的 CPUID 目录中发现的无效目录项
    fprintf(stderr, "Ignoring invalid dirent `%s' in dumped cpuid directory `%s'\n",
        dirent->d_name, src_cpuiddump_path);
    }
  }
  # 关闭 CPUID 目录
  closedir(dir);

  # 如果找不到任何有效的处理器单元条目，则打印错误信息并返回-1
  if (hwloc_bitmap_iszero(set)) {
    fprintf(stderr, "Did not find any valid pu%%u entry in dumped cpuid directory `%s'\n",
        src_cpuiddump_path);
    return -1;
  } else if (hwloc_bitmap_last(set) != hwloc_bitmap_weight(set) - 1) {
    # 如果找到的处理器单元范围不是连续的，则打印错误信息并返回-1
    /* The x86 backends enforces contigous set of PUs starting at 0 so far */
    fprintf(stderr, "Found non-contigous pu%%u range in dumped cpuid directory `%s'\n",
        src_cpuiddump_path);
    return -1;
  }

  # 返回成功
  return 0;

 out_with_path:
  # 释放路径内存
  free(path);
 out_with_dir:
  # 关闭 CPUID 目录
  closedir(dir);
#endif /* HWLOC_WIN_SYS & !__MINGW32__ needs a lot of work */
  return -1;
}

static void
hwloc_x86_backend_disable(struct hwloc_backend *backend)
{
  // 释放 APICID 集合
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  hwloc_bitmap_free(data->apicid_set);
  // 释放 CPUID 数据路径
  free(data->src_cpuiddump_path);
  // 释放私有数据
  free(data);
}

static struct hwloc_backend *
hwloc_x86_component_instantiate(struct hwloc_topology *topology,
                struct hwloc_disc_component *component,
                unsigned excluded_phases __hwloc_attribute_unused,
                const void *_data1 __hwloc_attribute_unused,
                const void *_data2 __hwloc_attribute_unused,
                const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  struct hwloc_x86_backend_data_s *data;
  const char *src_cpuiddump_path;

  // 分配并初始化后端对象
  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    goto out;

  // 分配并初始化后端私有数据
  data = malloc(sizeof(*data));
  if (!data) {
    errno = ENOMEM;
    goto out_with_backend;
  }

  backend->private_data = data;
  backend->discover = hwloc_x86_discover;
  backend->disable = hwloc_x86_backend_disable;

  /* default values */
  // 设置默认值
  data->is_knl = 0;
  data->apicid_set = hwloc_bitmap_alloc();
  data->apicid_unique = 1;
  data->src_cpuiddump_path = NULL;

  // 获取环境变量 HWLOC_CPUID_PATH 的值
  src_cpuiddump_path = getenv("HWLOC_CPUID_PATH");
  if (src_cpuiddump_path) {
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    // 检查 CPUID 数据路径是否有效
    if (!hwloc_x86_check_cpuiddump_input(src_cpuiddump_path, set)) {
      backend->is_thissystem = 0;
      data->src_cpuiddump_path = strdup(src_cpuiddump_path);
      assert(!hwloc_bitmap_iszero(set)); /* enforced by hwloc_x86_check_cpuiddump_input() */
      data->nbprocs = hwloc_bitmap_weight(set);
    } else {
      fprintf(stderr, "Ignoring dumped cpuid directory.\n");
    }
    hwloc_bitmap_free(set);
  }

  return backend;

 out_with_backend:
  free(backend);
 out:
  return NULL;
}
# 定义一个静态结构体，表示 x86 架构的发现组件
static struct hwloc_disc_component hwloc_x86_disc_component = {
  "x86",  # 组件名称为 x86
  HWLOC_DISC_PHASE_CPU,  # 发现阶段为 CPU
  HWLOC_DISC_PHASE_GLOBAL,  # 全局发现阶段
  hwloc_x86_component_instantiate,  # 实例化函数为 hwloc_x86_component_instantiate
  45,  # 优先级为 45，介于 native 和 no_os 之间
  1,  # 可见性为 1
  NULL  # 无额外数据
};

# 定义一个常量结构体，表示 x86 架构的组件
const struct hwloc_component hwloc_x86_component = {
  HWLOC_COMPONENT_ABI,  # 组件 ABI 版本
  NULL, NULL,  # 无需初始化和销毁函数
  HWLOC_COMPONENT_TYPE_DISC,  # 组件类型为发现组件
  0,  # 无需额外数据
  &hwloc_x86_disc_component  # 指向发现组件的指针
};
```