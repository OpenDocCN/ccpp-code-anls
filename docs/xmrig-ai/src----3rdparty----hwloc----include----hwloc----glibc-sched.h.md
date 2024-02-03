# `xmrig\src\3rdparty\hwloc\include\hwloc\glibc-sched.h`

```cpp
/*
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2020 Inria
 * 版权所有 © 2009-2011 Université Bordeaux
 * 版权所有 © 2011 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 glibc 调度例程之间交互的宏。
 *
 * 使用同时使用 hwloc 和 glibc 调度例程（如 sched_getaffinity() 或 pthread_attr_setaffinity_np()）的应用程序可能希望包含此文件，以便简化它们之间类型的转换。
 */

#ifndef HWLOC_GLIBC_SCHED_H
#define HWLOC_GLIBC_SCHED_H

#include "hwloc.h"
#include "hwloc/helper.h"

#include <assert.h>

#if !defined _GNU_SOURCE || (!defined _SCHED_H && !defined _SCHED_H_) || (!defined CPU_SETSIZE && !defined sched_priority)
#error 请确保在包含 glibc-sched.h 之前包含 sched.h，并在任何包含 sched.h 之前定义 _GNU_SOURCE
#endif


#ifdef __cplusplus
extern "C" {
#endif


#ifdef HWLOC_HAVE_CPU_SET


/** \defgroup hwlocality_glibc_sched 与 glibc 调度亲和性的互操作性
 *
 * 此接口提供了一种在 hwloc cpusets 和 glibc cpusets 之间转换的方式，例如 sched_getaffinity() 或 pthread_attr_setaffinity_np() 操作的 cpusets。
 *
 * \note 拓扑 \p topology 必须与当前机器匹配。
 *
 * @{
 */


/** \brief 将 hwloc CPU 集 \p toposet 转换为 glibc 调度亲和性 CPU 集 \p schedset
 *
 * 在调用 sched_setaffinity 或任何以 cpu_set_t 作为输入参数的其他函数之前，可以使用此函数。
 *
 * \p schedsetsize 应该是 sizeof(cpu_set_t)，除非 \p schedset 是使用 CPU_ALLOC 动态分配的。
 */
static __hwloc_inline int
hwloc_cpuset_to_glibc_sched_affinity(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t hwlocset,
                    cpu_set_t *schedset, size_t schedsetsize)
{
#ifdef CPU_ZERO_S
  // 定义一个无符号整数变量 cpu
  unsigned cpu;
  // 清空调度器集合 schedset
  CPU_ZERO_S(schedsetsize, schedset);
  // 遍历 hwlocset 中的每个 CPU
  hwloc_bitmap_foreach_begin(cpu, hwlocset)
    // 将 CPU 加入调度器集合 schedset
    CPU_SET_S(cpu, schedsetsize, schedset);
  hwloc_bitmap_foreach_end();
#else /* !CPU_ZERO_S */
  // 定义一个无符号整数变量 cpu
  unsigned cpu;
  // 清空调度器集合 schedset
  CPU_ZERO(schedset);
  // 断言调度器集合的大小为 cpu_set_t 的大小
  assert(schedsetsize == sizeof(cpu_set_t));
  // 遍历 hwlocset 中的每个 CPU
  hwloc_bitmap_foreach_begin(cpu, hwlocset)
    // 将 CPU 加入调度器集合 schedset
    CPU_SET(cpu, schedset);
  hwloc_bitmap_foreach_end();
#endif /* !CPU_ZERO_S */
  // 返回 0 表示成功
  return 0;
}

/** \brief Convert glibc sched affinity CPU set \p schedset into hwloc CPU set
 *
 * This function may be used before calling sched_setaffinity  or any other function
 * that takes a cpu_set_t  as input parameter.
 *
 * \p schedsetsize should be sizeof(cpu_set_t) unless \p schedset was dynamically allocated with CPU_ALLOC
 */
static __hwloc_inline int
hwloc_cpuset_from_glibc_sched_affinity(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_cpuset_t hwlocset,
                                       const cpu_set_t *schedset, size_t schedsetsize)
{
  int cpu;
#ifdef CPU_ZERO_S
  int count;
#endif
  // 清空 hwlocset
  hwloc_bitmap_zero(hwlocset);
#ifdef CPU_ZERO_S
  // 计算 schedset 中设置的 CPU 数量
  count = CPU_COUNT_S(schedsetsize, schedset);
  cpu = 0;
  // 遍历 schedset 中的每个 CPU
  while (count) {
    // 如果 CPU 在 schedset 中被设置
    if (CPU_ISSET_S(cpu, schedsetsize, schedset)) {
      // 将 CPU 加入 hwlocset
      hwloc_bitmap_set(hwlocset, cpu);
      count--;
    }
    cpu++;
  }
#else /* !CPU_ZERO_S */
  /* sched.h does not support dynamic cpu_set_t (introduced in glibc 2.7),
   * assume we have a very old interface without CPU_COUNT (added in 2.6)
   */
  // 断言调度器集合的大小为 cpu_set_t 的大小
  assert(schedsetsize == sizeof(cpu_set_t));
  // 遍历 schedset 中的每个 CPU
  for(cpu=0; cpu<CPU_SETSIZE; cpu++)
    // 如果 CPU 在 schedset 中被设置
    if (CPU_ISSET(cpu, schedset))
      // 将 CPU 加入 hwlocset
      hwloc_bitmap_set(hwlocset, cpu);
#endif /* !CPU_ZERO_S */
  // 返回 0 表示成功
  return 0;
}

/** @} */


#endif /* CPU_SET */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_GLIBC_SCHED_H */
```