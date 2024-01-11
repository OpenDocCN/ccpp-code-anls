# `xmrig\src\3rdparty\hwloc\src\bind.c`

```
/*
 * 版权声明
 * 2009年CNRS版权所有
 * 2009-2020年Inria版权所有
 * 2009-2010年，2012年Université Bordeaux版权所有
 * 2011-2015年Cisco Systems, Inc.版权所有
 * 请参阅顶层目录中的COPYING文件。
 */

#include "private/autogen/config.h" // 包含自动生成的配置文件
#include "hwloc.h" // 包含hwloc头文件
#include "private/private.h" // 包含私有头文件
#include "hwloc/helper.h" // 包含辅助函数头文件

#ifdef HAVE_SYS_MMAN_H
#  include <sys/mman.h> // 如果有sys/mman.h头文件，则包含该头文件
#endif
/* <malloc.h> 仅在没有posix_memalign()的情况下需要 */
#if defined(hwloc_getpagesize) && !defined(HAVE_POSIX_MEMALIGN) && defined(HAVE_MEMALIGN) && defined(HAVE_MALLOC_H)
#include <malloc.h> // 如果满足条件，则包含<malloc.h>头文件
#endif
#ifdef HAVE_UNISTD_H
#include <unistd.h> // 如果有unistd.h头文件，则包含该头文件
#endif
#include <stdlib.h> // 包含标准库头文件
#include <errno.h> // 包含错误处理头文件

/* TODO: HWLOC_GNU_SYS,
 *
 * 当可用时，我们可以通用地使用glibc的sched_setaffinity
 *
 * Darwin和OpenBSD似乎没有绑定功能。
 */

#define HWLOC_CPUBIND_ALLFLAGS (HWLOC_CPUBIND_PROCESS|HWLOC_CPUBIND_THREAD|HWLOC_CPUBIND_STRICT|HWLOC_CPUBIND_NOMEMBIND) // 定义CPU绑定的所有标志

static hwloc_const_bitmap_t
hwloc_fix_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t set)
{
  hwloc_const_bitmap_t topology_set = hwloc_topology_get_topology_cpuset(topology); // 获取拓扑结构中的CPU集合
  hwloc_const_bitmap_t complete_set = hwloc_topology_get_complete_cpuset(topology); // 获取完整的CPU集合

  if (hwloc_bitmap_iszero(set)) { // 如果输入的CPU集合为空
    errno = EINVAL; // 设置错误码为无效参数
    return NULL; // 返回空指针
  }

  if (!hwloc_bitmap_isincluded(set, complete_set)) { // 如果输入的CPU集合不包含在完整的CPU集合中
    errno = EINVAL; // 设置错误码为无效参数
    return NULL; // 返回空指针
  }

  if (hwloc_bitmap_isincluded(topology_set, set)) // 如果拓扑结构中的CPU集合包含输入的CPU集合
    set = complete_set; // 将输入的CPU集合设置为完整的CPU集合

  return set; // 返回CPU集合
}

int
hwloc_set_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) { // 如果标志位超出CPU绑定的所有标志
    errno = EINVAL; // 设置错误码为无效参数
    return -1; // 返回-1
  }

  set = hwloc_fix_cpubind(topology, set); // 修正CPU绑定
  if (!set) // 如果CPU集合为空
    return -1; // 返回-1

  if (flags & HWLOC_CPUBIND_PROCESS) { // 如果标志位包含进程级CPU绑定
    if (topology->binding_hooks.set_thisproc_cpubind) // 如果拓扑结构中的绑定钩子包含设置当前进程CPU绑定的函数指针
      return topology->binding_hooks.set_thisproc_cpubind(topology, set, flags); // 调用设置当前进程CPU绑定的函数
  } else if (flags & HWLOC_CPUBIND_THREAD) { // 如果标志位包含线程级CPU绑定
    # 如果拓扑结构中存在设置本线程 CPU 亲和力的钩子函数，则调用该函数并返回结果
    if (topology->binding_hooks.set_thisthread_cpubind)
      return topology->binding_hooks.set_thisthread_cpubind(topology, set, flags);
  } else {
    # 如果拓扑结构中不存在设置本线程 CPU 亲和力的钩子函数
    if (topology->binding_hooks.set_thisproc_cpubind) {
      # 调用设置本进程 CPU 亲和力的钩子函数，并获取返回值
      int err = topology->binding_hooks.set_thisproc_cpubind(topology, set, flags);
      # 如果返回值大于等于 0 或者错误码不是 ENOSYS，则返回返回值
      if (err >= 0 || errno != ENOSYS)
        return err;
      # 如果返回值小于 0 且错误码为 ENOSYS，则执行回退操作
      /* ENOSYS, fallback */
    }
    # 如果拓扑结构中存在设置本线程 CPU 亲和力的钩子函数，则调用该函数并返回结果
    if (topology->binding_hooks.set_thisthread_cpubind)
      return topology->binding_hooks.set_thisthread_cpubind(topology, set, flags);
  }

  # 设置错误码为 ENOSYS
  errno = ENOSYS;
  # 返回 -1
  return -1;
}  // 结束函数定义

int  // 定义返回类型为整型的函数

hwloc_get_cpubind(hwloc_topology_t topology, hwloc_bitmap_t set, int flags)  // 函数名及参数列表

{  // 函数体开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 检查 flags 是否包含未知的标志位
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  if (flags & HWLOC_CPUBIND_PROCESS) {  // 如果 flags 包含进程绑定标志
    if (topology->binding_hooks.get_thisproc_cpubind)  // 如果拓扑结构中存在获取当前进程 CPU 绑定的钩子函数
      return topology->binding_hooks.get_thisproc_cpubind(topology, set, flags);  // 调用获取当前进程 CPU 绑定的钩子函数
  } else if (flags & HWLOC_CPUBIND_THREAD) {  // 如果 flags 包含线程绑定标志
    if (topology->binding_hooks.get_thisthread_cpubind)  // 如果拓扑结构中存在获取当前线程 CPU 绑定的钩子函数
      return topology->binding_hooks.get_thisthread_cpubind(topology, set, flags);  // 调用获取当前线程 CPU 绑定的钩子函数
  } else {  // 如果 flags 既不包含进程绑定标志也不包含线程绑定标志
    if (topology->binding_hooks.get_thisproc_cpubind) {  // 如果拓扑结构中存在获取当前进程 CPU 绑定的钩子函数
      int err = topology->binding_hooks.get_thisproc_cpubind(topology, set, flags);  // 调用获取当前进程 CPU 绑定的钩子函数
      if (err >= 0 || errno != ENOSYS)  // 如果返回值大于等于 0 或者错误码不是不支持的系统调用
        return err;  // 返回结果
      /* ENOSYS, fallback */  // 注释说明发生了不支持的系统调用，需要回退处理
    }
    if (topology->binding_hooks.get_thisthread_cpubind)  // 如果拓扑结构中存在获取当前线程 CPU 绑定的钩子函数
      return topology->binding_hooks.get_thisthread_cpubind(topology, set, flags);  // 调用获取当前线程 CPU 绑定的钩子函数
  }

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回错误
}

int  // 定义返回类型为整型的函数

hwloc_set_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, int flags)  // 函数名及参数列表

{  // 函数体开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 检查 flags 是否包含未知的标志位
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  set = hwloc_fix_cpubind(topology, set);  // 调用修复 CPU 绑定的函数，返回修复后的 CPU 绑定
  if (!set)  // 如果修复后的 CPU 绑定为空
    return -1;  // 返回错误

  if (topology->binding_hooks.set_proc_cpubind)  // 如果拓扑结构中存在设置进程 CPU 绑定的钩子函数
    return topology->binding_hooks.set_proc_cpubind(topology, pid, set, flags);  // 调用设置进程 CPU 绑定的钩子函数

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回错误
}

int  // 定义返回类型为整型的函数

hwloc_get_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, int flags)  // 函数名及参数列表

{  // 函数体开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 检查 flags 是否包含未知的标志位
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  if (topology->binding_hooks.get_proc_cpubind)  // 如果拓扑结构中存在获取进程 CPU 绑定的钩子函数
    return topology->binding_hooks.get_proc_cpubind(topology, pid, set, flags);  // 调用获取进程 CPU 绑定的钩子函数

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回错误
}

#ifdef hwloc_thread_t  // 如果定义了线程类型

int  // 定义返回类型为整型的函数

hwloc_set_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_const_bitmap_t set, int flags)  // 函数名及参数列表

{  // 函数体开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 检查 flags 是否包含未知的标志位
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  set = hwloc_fix_cpubind(topology, set);  // 调用修复 CPU 绑定的函数，返回修复后的 CPU 绑定
  if (!set)  // 如果修复后的 CPU 绑定为空
    # 返回-1，表示出现错误
    return -1;

  # 如果topology->binding_hooks.set_thread_cpubind存在，则调用该函数设置线程的CPU绑定
  if (topology->binding_hooks.set_thread_cpubind)
    return topology->binding_hooks.set_thread_cpubind(topology, tid, set, flags);

  # 如果topology->binding_hooks.set_thread_cpubind不存在，则设置errno为ENOSYS，表示函数不可用
  errno = ENOSYS;
  # 返回-1，表示出现错误
  return -1;
}  // 结束函数定义

int  // 定义返回类型为整型的函数

hwloc_get_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_bitmap_t set, int flags)  // 函数签名，获取线程的 CPU 绑定信息

{  // 函数开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 如果 flags 中包含除了 HWLOC_CPUBIND_ALLFLAGS 之外的其他标志
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回 -1
  }

  if (topology->binding_hooks.get_thread_cpubind)  // 如果拓扑结构中的绑定钩子包含获取线程 CPU 绑定的函数
    return topology->binding_hooks.get_thread_cpubind(topology, tid, set, flags);  // 调用获取线程 CPU 绑定的函数并返回结果

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回 -1
}  // 函数结束

#endif  // 结束条件编译

int  // 定义返回类型为整型的函数

hwloc_get_last_cpu_location(hwloc_topology_t topology, hwloc_bitmap_t set, int flags)  // 函数签名，获取最后一个 CPU 的位置

{  // 函数开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 如果 flags 中包含除了 HWLOC_CPUBIND_ALLFLAGS 之外的其他标志
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回 -1
  }

  if (flags & HWLOC_CPUBIND_PROCESS) {  // 如果 flags 中包含 HWLOC_CPUBIND_PROCESS 标志
    if (topology->binding_hooks.get_thisproc_last_cpu_location)  // 如果拓扑结构中的绑定钩子包含获取当前进程最后一个 CPU 位置的函数
      return topology->binding_hooks.get_thisproc_last_cpu_location(topology, set, flags);  // 调用获取当前进程最后一个 CPU 位置的函数并返回结果
  } else if (flags & HWLOC_CPUBIND_THREAD) {  // 如果 flags 中包含 HWLOC_CPUBIND_THREAD 标志
    if (topology->binding_hooks.get_thisthread_last_cpu_location)  // 如果拓扑结构中的绑定钩子包含获取当前线程最后一个 CPU 位置的函数
      return topology->binding_hooks.get_thisthread_last_cpu_location(topology, set, flags);  // 调用获取当前线程最后一个 CPU 位置的函数并返回结果
  } else {  // 如果 flags 中不包含 HWLOC_CPUBIND_PROCESS 或 HWLOC_CPUBIND_THREAD 标志
    if (topology->binding_hooks.get_thisproc_last_cpu_location) {  // 如果拓扑结构中的绑定钩子包含获取当前进程最后一个 CPU 位置的函数
      int err = topology->binding_hooks.get_thisproc_last_cpu_location(topology, set, flags);  // 调用获取当前进程最后一个 CPU 位置的函数
      if (err >= 0 || errno != ENOSYS)  // 如果返回值大于等于 0 或者错误码不是不支持的系统调用
        return err;  // 返回结果
      /* ENOSYS, fallback */  // 注释说明
    }
    if (topology->binding_hooks.get_thisthread_last_cpu_location)  // 如果拓扑结构中的绑定钩子包含获取当前线程最后一个 CPU 位置的函数
      return topology->binding_hooks.get_thisthread_last_cpu_location(topology, set, flags);  // 调用获取当前线程最后一个 CPU 位置的函数并返回结果
  }

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回 -1
}  // 函数结束

int  // 定义返回类型为整型的函数

hwloc_get_proc_last_cpu_location(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, int flags)  // 函数签名，获取进程最后一个 CPU 的位置

{  // 函数开始

  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {  // 如果 flags 中包含除了 HWLOC_CPUBIND_ALLFLAGS 之外的其他标志
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回 -1
  }

  if (topology->binding_hooks.get_proc_last_cpu_location)  // 如果拓扑结构中的绑定钩子包含获取进程最后一个 CPU 位置的函数
    return topology->binding_hooks.get_proc_last_cpu_location(topology, pid, set, flags);  // 调用获取进程最后一个 CPU 位置的函数并返回结果

  errno = ENOSYS;  // 设置错误码为不支持的系统调用
  return -1;  // 返回 -1
}  // 函数结束

#define HWLOC_MEMBIND_ALLFLAGS (HWLOC_MEMBIND_PROCESS|HWLOC_MEMBIND_THREAD|HWLOC_MEMBIND_STRICT|HWLOC_MEMBIND_MIGRATE|HWLOC_MEMBIND_NOCPUBIND|HWLOC_MEMBIND_BYNODESET)  // 定义内存绑定的所有标志

static hwloc_const_nodeset_t  // 定义静态的节点集合
# 修复内存绑定，将给定的节点集合绑定到指定的拓扑结构上
hwloc_fix_membind(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset)
{
  # 获取拓扑结构的节点集合
  hwloc_const_bitmap_t topology_nodeset = hwloc_topology_get_topology_nodeset(topology);
  # 获取完整的节点集合
  hwloc_const_bitmap_t complete_nodeset = hwloc_topology_get_complete_nodeset(topology);

  # 如果给定的节点集合为空，则设置错误码并返回空指针
  if (hwloc_bitmap_iszero(nodeset)) {
    errno = EINVAL;
    return NULL;
  }

  # 如果给定的节点集合不包含在完整的节点集合中，则设置错误码并返回空指针
  if (!hwloc_bitmap_isincluded(nodeset, complete_nodeset)) {
    errno = EINVAL;
    return NULL;
  }

  # 如果拓扑结构的节点集合包含在给定的节点集合中，则返回完整的节点集合
  if (hwloc_bitmap_isincluded(topology_nodeset, nodeset))
    return complete_nodeset;

  # 否则返回给定的节点集合
  return nodeset;
}

# 修复内存绑定的 CPU 集合，将给定的节点集合绑定到指定的拓扑结构上
static int
hwloc_fix_membind_cpuset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_const_cpuset_t cpuset)
{
  # 获取拓扑结构的 CPU 集合
  hwloc_const_bitmap_t topology_set = hwloc_topology_get_topology_cpuset(topology);
  # 获取完整的 CPU 集合
  hwloc_const_bitmap_t complete_set = hwloc_topology_get_complete_cpuset(topology);
  # 获取完整的节点集合
  hwloc_const_bitmap_t complete_nodeset = hwloc_topology_get_complete_nodeset(topology);

  # 如果给定的 CPU 集合为空，则设置错误码并返回 -1
  if (hwloc_bitmap_iszero(cpuset)) {
    errno = EINVAL;
    return -1;
  }

  # 如果给定的 CPU 集合不包含在完整的 CPU 集合中，则设置错误码并返回 -1
  if (!hwloc_bitmap_isincluded(cpuset, complete_set)) {
    errno = EINVAL;
    return -1;
  }

  # 如果拓扑结构的 CPU 集合包含在给定的 CPU 集合中，则将完整的节点集合复制到给定的节点集合中，并返回 0
  if (hwloc_bitmap_isincluded(topology_set, cpuset)) {
    hwloc_bitmap_copy(nodeset, complete_nodeset);
    return 0;
  }

  # 否则将给定的 CPU 集合转换为节点集合，并返回 0
  hwloc_cpuset_to_nodeset(topology, cpuset, nodeset);
  return 0;
}

# 检查内存绑定策略是否合法
static __hwloc_inline int hwloc__check_membind_policy(hwloc_membind_policy_t policy)
{
  # 如果内存绑定策略是默认、首次访问、绑定、交错或下次访问，则返回 0，否则返回 -1
  if (policy == HWLOC_MEMBIND_DEFAULT
      || policy == HWLOC_MEMBIND_FIRSTTOUCH
      || policy == HWLOC_MEMBIND_BIND
      || policy == HWLOC_MEMBIND_INTERLEAVE
      || policy == HWLOC_MEMBIND_NEXTTOUCH)
    return 0;
  return -1;
}

# 根据节点集合设置内存绑定策略
static int
hwloc_set_membind_by_nodeset(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 如果标志位包含非法标志，或者内存绑定策略不合法，则设置错误码并返回 -1
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  # 修复给定的节点集合
  nodeset = hwloc_fix_membind(topology, nodeset);
  # 如果节点集合为空，则
  if (!nodeset)
    # 返回错误代码 -1
    return -1;

  # 如果标志包含 HWLOC_MEMBIND_PROCESS
  if (flags & HWLOC_MEMBIND_PROCESS) {
    # 如果拓扑结构中的绑定钩子包含设置该进程内存绑定的函数
    if (topology->binding_hooks.set_thisproc_membind)
      # 调用设置该进程内存绑定的函数，并返回结果
      return topology->binding_hooks.set_thisproc_membind(topology, nodeset, policy, flags);
  } else if (flags & HWLOC_MEMBIND_THREAD) {
    # 如果标志包含 HWLOC_MEMBIND_THREAD
    if (topology->binding_hooks.set_thisthread_membind)
      # 如果拓扑结构中的绑定钩子包含设置该线程内存绑定的函数
      return topology->binding_hooks.set_thisthread_membind(topology, nodeset, policy, flags);
  } else {
    # 如果拓扑结构中的绑定钩子包含设置该进程内存绑定的函数
    if (topology->binding_hooks.set_thisproc_membind) {
      # 调用设置该进程内存绑定的函数，并返回结果
      int err = topology->binding_hooks.set_thisproc_membind(topology, nodeset, policy, flags);
      # 如果返回值大于等于0或者错误码不是 ENOSYS，则返回结果
      if (err >= 0 || errno != ENOSYS)
        return err;
      # 如果错误码是 ENOSYS，则使用回退方案
      /* ENOSYS, fallback */
    }
    # 如果拓扑结构中的绑定钩子包含设置该线程内存绑定的函数
    if (topology->binding_hooks.set_thisthread_membind)
      # 调用设置该线程内存绑定的函数，并返回结果
      return topology->binding_hooks.set_thisthread_membind(topology, nodeset, policy, flags);
  }

  # 设置错误码为 ENOSYS
  errno = ENOSYS;
  # 返回错误代码 -1
  return -1;
}

int
hwloc_set_membind(hwloc_topology_t topology, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  // 检查是否使用节点集进行内存绑定
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    // 使用节点集进行内存绑定
    ret = hwloc_set_membind_by_nodeset(topology, set, policy, flags);
  } else {
    // 分配一个节点集
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    // 修复内存绑定的 CPU 集合
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      // 使用节点集进行内存绑定
      ret = hwloc_set_membind_by_nodeset(topology, nodeset, policy, flags);
    // 释放节点集
    hwloc_bitmap_free(nodeset);
  }
  return ret;
}

static int
hwloc_get_membind_by_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  // 检查标志位是否超出 HWLOC_MEMBIND_ALLFLAGS
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_MEMBIND_PROCESS) {
    // 如果是进程级内存绑定
    if (topology->binding_hooks.get_thisproc_membind)
      return topology->binding_hooks.get_thisproc_membind(topology, nodeset, policy, flags);
  } else if (flags & HWLOC_MEMBIND_THREAD) {
    // 如果是线程级内存绑定
    if (topology->binding_hooks.get_thisthread_membind)
      return topology->binding_hooks.get_thisthread_membind(topology, nodeset, policy, flags);
  } else {
    if (topology->binding_hooks.get_thisproc_membind) {
      // 获取进程级内存绑定
      int err = topology->binding_hooks.get_thisproc_membind(topology, nodeset, policy, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.get_thisthread_membind)
      // 获取线程级内存绑定
      return topology->binding_hooks.get_thisthread_membind(topology, nodeset, policy, flags);
  }

  errno = ENOSYS;
  return -1;
}

int
hwloc_get_membind(hwloc_topology_t topology, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    // 获取节点集的内存绑定
    ret = hwloc_get_membind_by_nodeset(topology, set, policy, flags);
  } else {
    // 分配一个节点集
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    // 获取节点集的内存绑定
    ret = hwloc_get_membind_by_nodeset(topology, nodeset, policy, flags);
    # 如果 ret 为假（0），则将 nodeset 转换为对应的 cpuset
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    # 释放 nodeset 占用的内存
    hwloc_bitmap_free(nodeset);
  }
  # 返回 ret
  return ret;
}

static int
hwloc_set_proc_membind_by_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 检查标志位和内存绑定策略是否合法，如果不合法则设置 errno 并返回 -1
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  # 修正节点集合，确保其有效性
  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    return -1;

  # 如果拥有设置进程内存绑定的钩子函数，则调用该函数
  if (topology->binding_hooks.set_proc_membind)
    return topology->binding_hooks.set_proc_membind(topology, pid, nodeset, policy, flags);

  # 如果没有设置进程内存绑定的钩子函数，则设置 errno 并返回 -1
  errno = ENOSYS;
  return -1;
}


int
hwloc_set_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  # 如果标志位包含 HWLOC_MEMBIND_BYNODESET，则调用 hwloc_set_proc_membind_by_nodeset 函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_set_proc_membind_by_nodeset(topology, pid, set, policy, flags);
  } else {
    # 否则，分配一个节点集合，修正 CPU 集合，然后调用 hwloc_set_proc_membind_by_nodeset 函数
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      ret = hwloc_set_proc_membind_by_nodeset(topology, pid, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

static int
hwloc_get_proc_membind_by_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  # 检查标志位是否合法，如果不合法则设置 errno 并返回 -1
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  # 如果拥有获取进程内存绑定的钩子函数，则调用该函数
  if (topology->binding_hooks.get_proc_membind)
    return topology->binding_hooks.get_proc_membind(topology, pid, nodeset, policy, flags);

  # 如果没有获取进程内存绑定的钩子函数，则设置 errno 并返回 -1
  errno = ENOSYS;
  return -1;
}

int
hwloc_get_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  # 如果标志位包含 HWLOC_MEMBIND_BYNODESET，则调用 hwloc_get_proc_membind_by_nodeset 函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_proc_membind_by_nodeset(topology, pid, set, policy, flags);
  } else {
    # 否则，分配一个节点集合，然后调用 hwloc_get_proc_membind_by_nodeset 函数
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_proc_membind_by_nodeset(topology, pid, nodeset, policy, flags);
    # 如果 ret 为假（0），则将 nodeset 转换为对应的 cpuset
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    # 释放 nodeset 占用的内存
    hwloc_bitmap_free(nodeset);
  }
  # 返回 ret
  return ret;
# 设置给定内存区域的内存绑定，根据节点集合
static int
hwloc_set_area_membind_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 检查标志位和内存绑定策略是否合法，如果不合法则设置 errno 并返回 -1
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  # 如果 len 为 0，则无需执行任何操作，直接返回 0
  if (!len)
    /* nothing to do */
    return 0;

  # 修正节点集合，确保其合法性
  nodeset = hwloc_fix_membind(topology, nodeset);
  # 如果节点集合为空，则返回 -1
  if (!nodeset)
    return -1;

  # 如果存在设置区域内存绑定的钩子函数，则调用该函数并返回结果
  if (topology->binding_hooks.set_area_membind)
    return topology->binding_hooks.set_area_membind(topology, addr, len, nodeset, policy, flags);

  # 如果不存在设置区域内存绑定的钩子函数，则设置 errno 并返回 -1
  errno = ENOSYS;
  return -1;
}

# 设置给定内存区域的内存绑定，根据位图
int
hwloc_set_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  # 如果标志位包含 HWLOC_MEMBIND_BYNODESET，则调用 hwloc_set_area_membind_by_nodeset 函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_set_area_membind_by_nodeset(topology, addr, len, set, policy, flags);
  } else {
    # 否则，分配一个节点集合，修正绑定的 CPU 集合，然后调用 hwloc_set_area_membind_by_nodeset 函数
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      ret = hwloc_set_area_membind_by_nodeset(topology, addr, len, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  # 返回结果
  return ret;
}

# 获取给定内存区域的内存绑定，根据节点集合
static int
hwloc_get_area_membind_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  # 检查标志位是否合法，如果不合法则设置 errno 并返回 -1
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  # 如果 len 为 0，则设置 errno 并返回 -1
  if (!len) {
    /* nothing to query */
    errno = EINVAL;
    return -1;
  }

  # 如果存在获取区域内存绑定的钩子函数，则调用该函数并返回结果
  if (topology->binding_hooks.get_area_membind)
    return topology->binding_hooks.get_area_membind(topology, addr, len, nodeset, policy, flags);

  # 如果不存在获取区域内存绑定的钩子函数，则设置 errno 并返回 -1
  errno = ENOSYS;
  return -1;
}

# 获取给定内存区域的内存绑定，根据位图
int
hwloc_get_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  # 如果标志位包含 HWLOC_MEMBIND_BYNODESET，则调用 hwloc_get_area_membind_by_nodeset 函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    # 如果节点集合不为空，则根据节点集合获取内存绑定信息
    ret = hwloc_get_area_membind_by_nodeset(topology, addr, len, set, policy, flags);
  } else {
    # 如果节点集合为空，则创建一个节点集合
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    # 根据节点集合获取内存绑定信息
    ret = hwloc_get_area_membind_by_nodeset(topology, addr, len, nodeset, policy, flags);
    # 如果返回值为空，则将节点集合转换为 CPU 集合
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    # 释放节点集合的内存
    hwloc_bitmap_free(nodeset);
  }
  # 返回获取的内存绑定信息
  return ret;
}

static int
hwloc_get_area_memlocation_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, int flags)
{
  # 检查 flags 是否包含除 HWLOC_MEMBIND_ALLFLAGS 之外的其他标志
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  # 如果 len 为 0，则无需执行任何操作，直接返回
  if (!len)
    /* nothing to do */
    return 0;

  # 如果 topology 对象的 binding_hooks 中包含 get_area_memlocation 函数，则调用该函数
  if (topology->binding_hooks.get_area_memlocation)
    return topology->binding_hooks.get_area_memlocation(topology, addr, len, nodeset, flags);

  # 如果没有 get_area_memlocation 函数，则设置 errno 为 ENOSYS，并返回 -1
  errno = ENOSYS;
  return -1;
}

int
hwloc_get_area_memlocation(hwloc_topology_t topology, const void *addr, size_t len, hwloc_cpuset_t set, int flags)
{
  int ret;

  # 如果 flags 包含 HWLOC_MEMBIND_BYNODESET 标志，则调用 hwloc_get_area_memlocation_by_nodeset 函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_area_memlocation_by_nodeset(topology, addr, len, set, flags);
  } else {
    # 否则，分配一个 nodeset 对象，调用 hwloc_get_area_memlocation_by_nodeset 函数，并根据返回结果设置 cpuset 对象
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_area_memlocation_by_nodeset(topology, addr, len, nodeset, flags);
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

void *
hwloc_alloc_heap(hwloc_topology_t topology __hwloc_attribute_unused, size_t len)
{
  void *p = NULL;
  # 根据系统支持的内存对齐方式分配内存
  # 如果支持 posix_memalign，则使用 posix_memalign 分配内存
  # 如果支持 memalign，则使用 memalign 分配内存
  # 否则，使用 malloc 分配内存
  # 返回分配的内存指针
#if defined(hwloc_getpagesize) && defined(HAVE_POSIX_MEMALIGN)
  errno = posix_memalign(&p, hwloc_getpagesize(), len);
  if (errno)
    p = NULL;
#elif defined(hwloc_getpagesize) && defined(HAVE_MEMALIGN)
  p = memalign(hwloc_getpagesize(), len);
#else
  p = malloc(len);
#endif
  return p;
}

#ifdef MAP_ANONYMOUS
void *
hwloc_alloc_mmap(hwloc_topology_t topology __hwloc_attribute_unused, size_t len)
{
  # 使用 mmap 分配匿名内存映射
  void * buffer = mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
  return buffer == MAP_FAILED ? NULL : buffer;
}
#endif

int
hwloc_free_heap(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len __hwloc_attribute_unused)
{
  # 释放通过 malloc、posix_memalign 或 memalign 分配的内存
  free(addr);
  return 0;
}

#ifdef MAP_ANONYMOUS
int
hwloc_free_mmap(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len)
{
  # 释放通过 mmap 分配的匿名内存映射
  if (!addr)
    return 0;
  return munmap(addr, len);
}
#endif

void *
# 分配内存并绑定到指定的拓扑结构
hwloc_alloc(hwloc_topology_t topology, size_t len)
{
  # 如果拓扑结构中有绑定钩子，则使用绑定钩子进行内存分配
  if (topology->binding_hooks.alloc)
    return topology->binding_hooks.alloc(topology, len);
  # 否则调用默认的内存分配函数
  return hwloc_alloc_heap(topology, len);
}

# 根据节点集分配内存并进行内存绑定
static void *
hwloc_alloc_membind_by_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  void *p;

  # 检查标志位和内存绑定策略是否合法，若不合法则返回错误
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return NULL;
  }

  # 修正节点集的内存绑定
  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    goto fallback;
  # 如果标志位包含 HWLOC_MEMBIND_MIGRATE，则返回错误
  if (flags & HWLOC_MEMBIND_MIGRATE) {
    errno = EINVAL;
    goto fallback;
  }

  # 如果拓扑结构中有内存绑定钩子，则使用内存绑定钩子进行内存分配
  if (topology->binding_hooks.alloc_membind)
    return topology->binding_hooks.alloc_membind(topology, len, nodeset, policy, flags);
  # 如果拓扑结构中有设置区域内存绑定钩子，则使用设置区域内存绑定钩子进行内存分配
  else if (topology->binding_hooks.set_area_membind) {
    p = hwloc_alloc(topology, len);
    if (!p)
      return NULL;
    if (topology->binding_hooks.set_area_membind(topology, p, len, nodeset, policy, flags) && flags & HWLOC_MEMBIND_STRICT) {
      int error = errno;
      free(p);
      errno = error;
      return NULL;
    }
    return p;
  } else {
    errno = ENOSYS;
  }

fallback:
  # 如果标志位包含 HWLOC_MEMBIND_STRICT，则报告错误
  if (flags & HWLOC_MEMBIND_STRICT)
    /* Report error */
    return NULL;
  # 否则进行普通内存分配
  /* Never mind, allocate anyway */
  return hwloc_alloc(topology, len);
}

# 根据位图分配内存并进行内存绑定
void *
hwloc_alloc_membind(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  void *ret;

  # 如果标志位包含 HWLOC_MEMBIND_BYNODESET，则调用根据节点集分配内存的函数
  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_alloc_membind_by_nodeset(topology, len, set, policy, flags);
  } else {
    # 否则，根据位图创建节点集
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    # 修正节点集的内存绑定
    if (hwloc_fix_membind_cpuset(topology, nodeset, set)) {
      # 如果标志位包含 HWLOC_MEMBIND_STRICT，则返回错误
      if (flags & HWLOC_MEMBIND_STRICT)
    ret = NULL;
      else
    ret = hwloc_alloc(topology, len);
    } else
      ret = hwloc_alloc_membind_by_nodeset(topology, len, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

int
# 释放由 hwloc_alloc_membind() 分配的内存
def hwloc_free(hwloc_topology_t topology, void *addr, size_t len):
    # 如果绑定钩子中包含 free_membind 方法，则调用该方法释放内存
    if (topology->binding_hooks.free_membind)
        return topology->binding_hooks.free_membind(topology, addr, len)
    # 否则调用 hwloc_free_heap 方法释放内存
    return hwloc_free_heap(topology, addr, len)

# 空绑定钩子，始终返回成功
static int dontset_return_complete_cpuset(hwloc_topology_t topology, hwloc_cpuset_t set):
    # 将完整的 CPU 集合复制到给定的集合中
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology))
    return 0

static int dontset_thisthread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused):
    return 0

static int dontget_thisthread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused):
    # 调用 dontset_return_complete_cpuset 方法，将完整的 CPU 集合复制到给定的集合中
    return dontset_return_complete_cpuset(topology, set)

static int dontset_thisproc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused):
    return 0

static int dontget_thisproc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused):
    # 调用 dontset_return_complete_cpuset 方法，将完整的 CPU 集合复制到给定的集合中
    return dontset_return_complete_cpuset(topology, set)

static int dontset_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused):
    return 0

static int dontget_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_bitmap_t cpuset, int flags __hwloc_attribute_unused):
    # 调用 dontset_return_complete_cpuset 方法，将完整的 CPU 集合复制到给定的集合中
    return dontset_return_complete_cpuset(topology, cpuset)

#ifdef hwloc_thread_t
static int dontset_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t tid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused):
    return 0
static int dontget_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t tid __hwloc_attribute_unused, hwloc_bitmap_t cpuset, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_cpuset(topology, cpuset);
}
#endif

static int dontset_return_complete_nodeset(hwloc_topology_t topology, hwloc_nodeset_t set, hwloc_membind_policy_t *policy)
{
  // 将系统中所有可用的 NUMA 节点复制到指定的节点集合中
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_nodeset(topology));
  // 设置内存绑定策略为混合模式
  *policy = HWLOC_MEMBIND_MIXED;
  // 返回成功
  return 0;
}

static int dontset_thisproc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  // 不设置当前进程的内存绑定
  return 0;
}
static int dontget_thisproc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  // 获取当前进程的完整 NUMA 节点集合和内存绑定策略
  return dontset_return_complete_nodeset(topology, set, policy);
}

static int dontset_thisthread_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  // 不设置当前线程的内存绑定
  return 0;
}
static int dontget_thisthread_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  // 获取当前线程的完整 NUMA 节点集合和内存绑定策略
  return dontset_return_complete_nodeset(topology, set, policy);
}

static int dontset_proc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  // 不设置指定进程的内存绑定
  return 0;
}
static int dontget_proc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  // 返回不设置完整节点集的结果
  return dontset_return_complete_nodeset(topology, set, policy);
}

static int dontset_area_membind(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  // 返回 0，表示不设置区域内存绑定
  return 0;
}
static int dontget_area_membind(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  // 返回不设置完整节点集的结果
  return dontset_return_complete_nodeset(topology, set, policy);
}
static int dontget_area_memlocation(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused)
{
  hwloc_membind_policy_t policy;
  // 返回不设置完整节点集的结果
  return dontset_return_complete_nodeset(topology, set, &policy);
}

static void * dontalloc_membind(hwloc_topology_t topology __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  // 分配内存并返回
  return malloc(size);
}
static int dontfree_membind(hwloc_topology_t topology __hwloc_attribute_unused, void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused)
{
  // 释放内存
  free(addr);
  // 返回 0，表示释放内存成功
  return 0;
}

static void hwloc_set_dummy_hooks(struct hwloc_binding_hooks *hooks,
                  struct hwloc_topology_support *support __hwloc_attribute_unused)
{
  // 设置当前进程的 CPU 亲和性绑定函数为 dontset_thisproc_cpubind
  hooks->set_thisproc_cpubind = dontset_thisproc_cpubind;
  // 获取当前进程的 CPU 亲和性绑定函数为 dontget_thisproc_cpubind
  hooks->get_thisproc_cpubind = dontget_thisproc_cpubind;
  // 设置当前线程的 CPU 亲和性绑定函数为 dontset_thisthread_cpubind
  hooks->set_thisthread_cpubind = dontset_thisthread_cpubind;
  // 获取当前线程的 CPU 亲和性绑定函数为 dontget_thisthread_cpubind
  hooks->get_thisthread_cpubind = dontget_thisthread_cpubind;
  // 设置进程的 CPU 亲和性绑定函数为 dontset_proc_cpubind
  hooks->set_proc_cpubind = dontset_proc_cpubind;
  // 获取进程的 CPU 亲和性绑定函数为 dontget_proc_cpubind
  hooks->get_proc_cpubind = dontget_proc_cpubind;
#ifdef hwloc_thread_t
  // 设置线程的 CPU 亲和性绑定函数为 dontset_thread_cpubind
  hooks->set_thread_cpubind = dontset_thread_cpubind;
  // 获取线程的 CPU 亲和性绑定函数为 dontget_thread_cpubind
  hooks->get_thread_cpubind = dontget_thread_cpubind;
#endif
  // 获取当前进程的最后 CPU 位置函数为 dontget_thisproc_cpubind
  hooks->get_thisproc_last_cpu_location = dontget_thisproc_cpubind; /* cpubind instead of last_cpu_location is ok */
  // 获取当前线程的最后 CPU 位置函数为 dontget_thisthread_cpubind
  hooks->get_thisthread_last_cpu_location = dontget_thisthread_cpubind; /* cpubind instead of last_cpu_location is ok */
  // 获取进程的最后 CPU 位置函数为 dontget_proc_cpubind
  hooks->get_proc_last_cpu_location = dontget_proc_cpubind; /* cpubind instead of last_cpu_location is ok */
  // 设置当前进程的内存绑定函数为 dontset_thisproc_membind
  hooks->set_thisproc_membind = dontset_thisproc_membind;
  // 获取当前进程的内存绑定函数为 dontget_thisproc_membind
  hooks->get_thisproc_membind = dontget_thisproc_membind;
  // 设置当前线程的内存绑定函数为 dontset_thisthread_membind
  hooks->set_thisthread_membind = dontset_thisthread_membind;
  // 获取当前线程的内存绑定函数为 dontget_thisthread_membind
  hooks->get_thisthread_membind = dontget_thisthread_membind;
  // 设置进程的内存绑定函数为 dontset_proc_membind
  hooks->set_proc_membind = dontset_proc_membind;
  // 获取进程的内存绑定函数为 dontget_proc_membind
  hooks->get_proc_membind = dontget_proc_membind;
  // 设置区域的内存绑定函数为 dontset_area_membind
  hooks->set_area_membind = dontset_area_membind;
  // 获取区域的内存绑定函数为 dontget_area_membind
  hooks->get_area_membind = dontget_area_membind;
  // 获取区域的内存位置函数为 dontget_area_memlocation
  hooks->get_area_memlocation = dontget_area_memlocation;
  // 分配内存绑定函数为 dontalloc_membind
  hooks->alloc_membind = dontalloc_membind;
  // 释放内存绑定函数为 dontfree_membind
  hooks->free_membind = dontfree_membind;
}

void
hwloc_set_native_binding_hooks(struct hwloc_binding_hooks *hooks, struct hwloc_topology_support *support)
{
#    ifdef HWLOC_LINUX_SYS
    // 设置 Linux 系统的钩子函数
    hwloc_set_linuxfs_hooks(hooks, support);
#    endif /* HWLOC_LINUX_SYS */

#    ifdef HWLOC_BGQ_SYS
    // 设置 BGQ 系统的钩子函数
    hwloc_set_bgq_hooks(hooks, support);
#    endif /* HWLOC_BGQ_SYS */

#    ifdef HWLOC_AIX_SYS
    // 设置 AIX 系统的钩子函数
    hwloc_set_aix_hooks(hooks, support);
#    endif /* HWLOC_AIX_SYS */

#    ifdef HWLOC_SOLARIS_SYS
    // 设置 Solaris 系统的钩子函数
    hwloc_set_solaris_hooks(hooks, support);
#    endif /* HWLOC_SOLARIS_SYS */
# 如果是 Windows 系统，设置相应的钩子函数
    hwloc_set_windows_hooks(hooks, support);
# 如果是 Darwin 系统，设置相应的钩子函数
    hwloc_set_darwin_hooks(hooks, support);
# 如果是 FreeBSD 系统，设置相应的钩子函数
    hwloc_set_freebsd_hooks(hooks, support);
# 如果是 NetBSD 系统，设置相应的钩子函数
    hwloc_set_netbsd_hooks(hooks, support);
# 如果是 HPUX 系统，设置相应的钩子函数
    hwloc_set_hpux_hooks(hooks, support);
}

# 如果所代表的系统实际上不是这个系统，则使用虚拟的绑定钩子函数
void
hwloc_set_binding_hooks(struct hwloc_topology *topology)
{
  # 如果是这个系统
  if (topology->is_thissystem) {
    使用本地的绑定钩子函数
    hwloc_set_native_binding_hooks(&topology->binding_hooks, &topology->support);
    /* 每个未设置的钩子函数都会返回 ENOSYS */
  } else {
    /* 不是这个系统，使用虚拟的绑定钩子函数，不执行任何操作（但不返回 ENOSYS） */
    使用虚拟的钩子函数
    hwloc_set_dummy_hooks(&topology->binding_hooks, &topology->support);

    /* Linux 有一些钩子函数在这种情况下也可以工作，但目前并不是必需的。 */
  }

  /* 如果不是这个系统，set_cpubind 是虚假的
   * get_cpubind 返回整个系统的 CPU 集合，
   * 所以不要报告 set/get_cpubind 为支持的
   */
  如果是这个系统
  if (topology->is_thissystem) {
#define DO(which,kind) \
    if (topology->binding_hooks.kind) \
      topology->support.which##bind->kind = 1;
    DO(cpu,set_thisproc_cpubind);
    DO(cpu,get_thisproc_cpubind);
    DO(cpu,set_proc_cpubind);
    DO(cpu,get_proc_cpubind);
    DO(cpu,set_thisthread_cpubind);
    DO(cpu,get_thisthread_cpubind);
#ifdef hwloc_thread_t
    DO(cpu,set_thread_cpubind);
    DO(cpu,get_thread_cpubind);
#endif
    DO(cpu,get_thisproc_last_cpu_location);
    DO(cpu,get_proc_last_cpu_location);
    DO(cpu,get_thisthread_last_cpu_location);
    DO(mem,set_thisproc_membind);
    DO(mem,get_thisproc_membind);
    DO(mem,set_thisthread_membind);
    DO(mem,get_thisthread_membind);
    # 调用set_proc_membind函数，用于将进程绑定到特定的内存节点
    DO(mem,set_proc_membind);
    # 调用get_proc_membind函数，用于获取进程当前绑定的内存节点
    DO(mem,get_proc_membind);
    # 调用set_area_membind函数，用于将内存区域绑定到特定的内存节点
    DO(mem,set_area_membind);
    # 调用get_area_membind函数，用于获取内存区域当前绑定的内存节点
    DO(mem,get_area_membind);
    # 调用get_area_memlocation函数，用于获取内存区域所在的内存节点
    DO(mem,get_area_memlocation);
    # 调用alloc_membind函数，用于在特定的内存节点上分配内存
    DO(mem,alloc_membind);
# 取消定义 DO 宏
#undef DO
# 结束当前的代码块
}
# 结束当前的代码块
}
```