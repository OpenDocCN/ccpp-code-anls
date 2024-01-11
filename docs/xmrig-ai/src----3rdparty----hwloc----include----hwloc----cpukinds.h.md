# `xmrig\src\3rdparty\hwloc\include\hwloc\cpukinds.h`

```
/*
 * 版权所有 © 2020-2021 Inria。保留所有权利。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief CPU核心的种类。
 */

#ifndef HWLOC_CPUKINDS_H
#define HWLOC_CPUKINDS_H

#include "hwloc.h"

#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif

/** \brief 获取拓扑结构中不同种类的CPU核心数量。
 *
 * \p flags 目前必须为 \c 0。
 *
 * \return 成功时CPU种类的数量（正整数）。
 * \return 如果没有关于种类的信息，则返回 \c 0。
 * \return 如果 \p flags 无效，则返回 \c -1，并将 \p errno 设置为 \c EINVAL。
 */
HWLOC_DECLSPEC int
hwloc_cpukinds_get_nr(hwloc_topology_t topology,
                      unsigned long flags);

/** \brief 获取包含在 \p cpuset 中列出的CPU的CPU种类的索引。
 *
 * \p flags 目前必须为 \c 0。
 *
 * \return 成功时CPU种类的索引（正整数或 0）。
 * \return 如果 \p cpuset 仅部分包含在某种种类中，则返回 \c -1，并将 \p errno 设置为 \c EXDEV。
 * \return 如果 \p cpuset 完全不包含在任何种类中，则返回 \c -1，并将 \p errno 设置为 \c ENOENT。
 * \return 如果参数无效，则返回 \c -1，并将 \p errno 设置为 \c EINVAL。
 */
HWLOC_DECLSPEC int
hwloc_cpukinds_get_by_cpuset(hwloc_topology_t topology,
                             hwloc_const_bitmap_t cpuset,
                             unsigned long flags);
/**
 * \brief 获取拓扑结构中某种 CPU 类型的 CPU 集合和信息
 *
 * \p kind_index 标识拓扑结构中的一种 CPU 类型，范围在 0 到 hwloc_cpukinds_get_nr() 返回的种类数量减 1 之间
 *
 * 如果不为 \c NULL，\p cpuset 将被填充为该类型的处理器集合
 *
 * 如果不为 \c NULL，\p efficiency 指向的整数将被填充为该 CPU 类型的效率排名（见上文）
 * 它的范围是从 \c 0 到种类数量（由 hwloc_cpukinds_get_nr() 报告）减 1
 *
 * 效率较低的种类将被优先报告
 *
 * 如果拓扑结构中只有一种类型，其效率为 \c 0
 * 如果某些核心类型的效率未知，则所有类型的效率都设置为 \c -1，并且类型不按特定顺序报告
 *
 * 信息属性数组（例如 "CoreType"、"FrequencyMaxMHz" 或 "FrequencyBaseMHz"，参见 \ref topoattrs_cpukinds）及其长度将在 \p infos 或 \p nr_infos 中返回
 * 该数组属于拓扑结构，不应该被释放或修改
 *
 * 如果 \p nr_infos 或 \p infos 为 \c NULL，则不返回任何信息
 *
 * \p flags 目前必须为 \c 0
 *
 * \return 成功时返回 \c 0
 * \return 如果 \p kind_index 不匹配任何 CPU 类型，则返回 \c -1，并将 \p errno 设置为 \c ENOENT
 * \return 如果参数无效，则返回 \c -1，并将 \p errno 设置为 \c EINVAL
 */
HWLOC_DECLSPEC int
hwloc_cpukinds_get_info(hwloc_topology_t topology,
                        unsigned kind_index,
                        hwloc_bitmap_t cpuset,
                        int *efficiency,
                        unsigned *nr_infos, struct hwloc_info_s **infos,
                        unsigned long flags);
/**
 * \brief 在拓扑结构中注册一种 CPU 类型
 *
 * 标记在 \p cpuset 中列出的 PUs 为具有相同类型，与给定属性相关
 *
 * 如果 \p forced_efficiency 为 \c -1，则表示未知。否则，它是一个抽象的效率值，用于强制所有类型的排名，如果它们都具有有效（且不同）的效率值。
 *
 * 大小为 \p nr_infos 的数组 \p infos 可用于提供描述这种 PUs 的信息名称和值。
 *
 * \p flags 现在必须为 \c 0。
 *
 * 参数 \p cpuset 和 \p infos 将在内部进行复制，调用者负责释放它们。
 *
 * 如果 \p cpuset 与某些现有类型重叠，则这些类型可能会被修改或拆分。例如，如果现有类型 A 包含 PUs 0 和 1，并且为 PU 1 和 2 注册了另一种类型，则将产生 3 种结果类型：
 * 现有类型 A 仅限于 PU 0；
 * 新类型 B 仅包含 PU 1，并结合了来自 A 和新注册类型的信息；
 * 新类型 C 仅包含 PU 2，并且仅获取来自新注册类型的信息。
 *
 * \note 提供给此函数的效率 \p forced_efficiency 可能与后来由 hwloc_cpukinds_get_info() 报告的效率不同，因为 hwloc 将效率值缩小到 0 到类型数减 1 之间。
 *
 * \return 成功时返回 \c 0。
 * \return 如果某些参数无效，则返回 \c -1，并将 \p errno 设置为 \c EINVAL，例如如果 \p cpuset 为 \c NULL 或为空。
 */
HWLOC_DECLSPEC int
hwloc_cpukinds_register(hwloc_topology_t topology,
                        hwloc_bitmap_t cpuset,
                        int forced_efficiency,
                        unsigned nr_infos, struct hwloc_info_s *infos,
                        unsigned long flags);

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_CPUKINDS_H */
```