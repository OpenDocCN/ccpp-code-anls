# `xmrig\src\3rdparty\hwloc\src\topology.c`

```
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2022 Inria
 * 版权所有 © 2009-2012, 2020 Université Bordeaux
 * 版权所有 © 2009-2011 Cisco Systems, Inc.
 * 版权所有 © 2022 IBM Corporation
 * 请参阅顶层目录中的 COPYING 文件
 */

#include "private/autogen/config.h"

#define _ATFILE_SOURCE
#include <assert.h>
#include <sys/types.h>
#ifdef HAVE_DIRENT_H
#include <dirent.h>
#endif
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <limits.h>
#include <float.h>

#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"
#include "private/misc.h"

#ifdef HAVE_MACH_MACH_INIT_H
#include <mach/mach_init.h>
#endif
#ifdef HAVE_MACH_INIT_H
#include <mach_init.h>
#endif
#ifdef HAVE_MACH_MACH_HOST_H
#include <mach/mach_host.h>
#endif

#ifdef HAVE_SYS_PARAM_H
#include <sys/param.h>
#endif

#ifdef HAVE_SYS_SYSCTL_H
#include <sys/sysctl.h>
#endif

#ifdef HWLOC_WIN_SYS
#include <windows.h>
#endif

#ifdef HWLOC_HAVE_LEVELZERO
#if HWLOC_HAVE_ATTRIBUTE_CONSTRUCTOR
// 如果支持构造函数属性，则定义构造函数
static void hwloc_constructor(void) __attribute__((constructor));
static void hwloc_constructor(void)
{
  // 如果环境变量 ZES_ENABLE_SYSMAN 不存在，则设置为 1
  if (!getenv("ZES_ENABLE_SYSMAN"))
#ifdef HWLOC_WIN_SYS
    // 在 Windows 平台使用 putenv() 设置环境变量
    putenv("ZES_ENABLE_SYSMAN=1");
#else
    // 在其他平台使用 setenv() 设置环境变量
    setenv("ZES_ENABLE_SYSMAN", "1", 1);
#endif
}
#endif
#ifdef HWLOC_WIN_SYS
// Windows 平台的 DLL 入口函数
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved)
{
  // 如果是 DLL 进程附加事件
  if (fdwReason == DLL_PROCESS_ATTACH) {
    // 如果环境变量 ZES_ENABLE_SYSMAN 不存在，则使用 putenv() 设置
    if (!getenv("ZES_ENABLE_SYSMAN"))
      putenv((char *) "ZES_ENABLE_SYSMAN=1");
  }
  return TRUE;
}
#endif
#endif /* HWLOC_HAVE_LEVELZERO */
unsigned hwloc_get_api_version(void)
{
  // 返回 HWLOC_API_VERSION，即 hwloc 库的 API 版本号
  return HWLOC_API_VERSION;
}

int hwloc_topology_abi_check(hwloc_topology_t topology)
{
  // 检查拓扑对象的 ABI 版本是否与当前 hwloc 库的 ABI 版本一致，一致返回 0，不一致返回 -1
  return topology->topology_abi != HWLOC_TOPOLOGY_ABI ? -1 : 0;
}

/* callers should rather use wrappers HWLOC_SHOW_ALL_ERRORS() and HWLOC_SHOW_CRITICAL_ERRORS() for clarity */
int hwloc_hide_errors(void)
{
  // 隐藏错误信息，默认只显示关键错误。lstopo 将显示其他错误
  static int hide = 1;
  static int checked = 0;
  if (!checked) {
    const char *envvar = getenv("HWLOC_HIDE_ERRORS");
    if (envvar) {
      hide = atoi(envvar);
#ifdef HWLOC_DEBUG
    } else {
      /* if debug is enabled and HWLOC_DEBUG_VERBOSE isn't forced to 0,
       * show all errors jus like we show all debug messages.
       */
      envvar = getenv("HWLOC_DEBUG_VERBOSE");
      if (!envvar || atoi(envvar))
        hide = 0;
#endif
    }
    checked = 1;
  }
  return hide;
}

/* format the obj info to print in error messages */
static void
report_insert_error_format_obj(char *buf, size_t buflen, hwloc_obj_t obj)
{
  // 格式化对象信息，用于在错误消息中打印
  char typestr[64];
  char *cpusetstr;
  char *nodesetstr = NULL;

  hwloc_obj_type_snprintf(typestr, sizeof(typestr), obj, 0);
  hwloc_bitmap_asprintf(&cpusetstr, obj->cpuset);
  if (obj->nodeset) /* may be missing during insert */
    hwloc_bitmap_asprintf(&nodesetstr, obj->nodeset);
  if (obj->os_index != HWLOC_UNKNOWN_INDEX)
    snprintf(buf, buflen, "%s (P#%u cpuset %s%s%s)",
             typestr, obj->os_index, cpusetstr,
             nodesetstr ? " nodeset " : "",
             nodesetstr ? nodesetstr : "");
  else
    snprintf(buf, buflen, "%s (cpuset %s%s%s)",
             typestr, cpusetstr,
             nodesetstr ? " nodeset " : "",
             nodesetstr ? nodesetstr : "");
  free(cpusetstr);
  free(nodesetstr);
}

static void report_insert_error(hwloc_obj_t new, hwloc_obj_t old, const char *msg, const char *reason)
{
  static int reported = 0;

  if (reason && !reported && HWLOC_SHOW_CRITICAL_ERRORS()) {
    char newstr[512];
    // 声明一个长度为512的字符数组oldstr
    char oldstr[512];
    // 将newstr的格式化错误信息插入到newstr中
    report_insert_error_format_obj(newstr, sizeof(newstr), new);
    // 将oldstr的格式化错误信息插入到oldstr中
    report_insert_error_format_obj(oldstr, sizeof(oldstr), old);

    // 输出错误信息到标准错误流
    fprintf(stderr, "****************************************************************************\n");
    fprintf(stderr, "* hwloc %s received invalid information from the operating system.\n", HWLOC_VERSION);
    fprintf(stderr, "*\n");
    fprintf(stderr, "* Failed with: %s\n", msg);
    fprintf(stderr, "* while inserting %s at %s\n", newstr, oldstr);
    fprintf(stderr, "* coming from: %s\n", reason);
    fprintf(stderr, "*\n");
    fprintf(stderr, "* The following FAQ entry in the hwloc documentation may help:\n");
    fprintf(stderr, "*   What should I do when hwloc reports \"operating system\" warnings?\n");
    fprintf(stderr, "* Otherwise please report this error message to the hwloc user's mailing list,\n");
#ifdef HWLOC_LINUX_SYS
    // 如果是在 Linux 系统下，输出特定的错误信息
    fprintf(stderr, "* along with the files generated by the hwloc-gather-topology script.\n");
#else
    // 如果不是在 Linux 系统下，输出另一种特定的错误信息
    fprintf(stderr, "* along with any relevant topology information from your platform.\n");
#endif
    // 输出通用的错误信息
    fprintf(stderr, "* \n");
    fprintf(stderr, "* hwloc will now ignore this invalid topology information and continue.\n");
    fprintf(stderr, "****************************************************************************\n");
    // 标记已经输出过错误信息
    reported = 1;
  }
}

#if defined(HAVE_SYSCTLBYNAME)
// 获取指定名称的系统控制变量的值
int hwloc_get_sysctlbyname(const char *name, int64_t *ret)
{
  union {
    int32_t i32;
    int64_t i64;
  } n;
  size_t size = sizeof(n);
  // 通过名称获取系统控制变量的值
  if (sysctlbyname(name, &n, &size, NULL, 0))
    return -1;
  // 根据返回的值大小，将值赋给 ret
  switch (size) {
    case sizeof(n.i32):
      *ret = n.i32;
      break;
    case sizeof(n.i64):
      *ret = n.i64;
      break;
    default:
      return -1;
  }
  return 0;
}
#endif

#if defined(HAVE_SYSCTL)
// 获取指定名称的系统控制变量的值
int hwloc_get_sysctl(int name[], unsigned namelen, int64_t *ret)
{
  union {
    int32_t i32;
    int64_t i64;
  } n;
  size_t size = sizeof(n);
  // 通过名称获取系统控制变量的值
  if (sysctl(name, namelen, &n, &size, NULL, 0))
    return -1;
  // 根据返回的值大小，将值赋给 ret
  switch (size) {
    case sizeof(n.i32):
      *ret = n.i32;
      break;
    case sizeof(n.i64):
      *ret = n.i64;
      break;
    default:
      return -1;
  }
  return 0;
}
#endif

/* Return the OS-provided number of processors.
 * Assumes topology->is_thissystem is true.
 */
#ifndef HWLOC_WIN_SYS /* The windows implementation is in topology-windows.c */
// 获取系统提供的处理器数量
int
hwloc_fallback_nbprocessors(unsigned flags) {
  int n;

  if (flags & HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE) {
    // 尝试获取所有可以处理离线处理器的 Linux 和 Solaris 系统的 CPU 数量
#if HAVE_DECL__SC_NPROCESSORS_CONF
    n = sysconf(_SC_NPROCESSORS_CONF);
#elif HAVE_DECL__SC_NPROC_CONF
    n = sysconf(_SC_NPROC_CONF);
#else
    n = -1;
#endif
    if (n != -1)
      return n;
  }

  // 尝试获取在线 CPU 的数量，或者获取任何可以获取的 CPU 数量
# 如果系统支持获取在线处理器数量
if HAVE_DECL__SC_NPROCESSORS_ONLN:
  # 获取在线处理器数量
  n = sysconf(_SC_NPROCESSORS_ONLN);
# 如果系统支持获取在线进程数量
elif HAVE_DECL__SC_NPROC_ONLN:
  # 获取在线进程数量
  n = sysconf(_SC_NPROC_ONLN);
# 如果系统支持获取配置的处理器数量
elif HAVE_DECL__SC_NPROCESSORS_CONF:
  # 获取配置的处理器数量
  n = sysconf(_SC_NPROCESSORS_CONF);
# 如果系统支持获取配置的进程数量
elif HAVE_DECL__SC_NPROC_CONF:
  # 获取配置的进程数量
  n = sysconf(_SC_NPROC_CONF);
# 如果系统支持获取主机基本信息
elif defined(HAVE_HOST_INFO) && HAVE_HOST_INFO:
  # 定义主机基本信息结构体和变量
  struct host_basic_info info;
  mach_msg_type_number_t count = HOST_BASIC_INFO_COUNT;
  # 获取主机基本信息
  host_info(mach_host_self(), HOST_BASIC_INFO, (integer_t*) &info, &count);
  # 获取可用的 CPU 数量
  n = info.avail_cpus;
# 如果系统支持通过名称获取系统信息
elif defined(HAVE_SYSCTLBYNAME):
  # 定义变量 nn
  int64_t nn;
  # 如果无法通过名称获取 CPU 数量，则设置为 -1
  if (hwloc_get_sysctlbyname("hw.ncpu", &nn))
    nn = -1;
  n = nn;
# 如果系统支持通过 sysctl 获取 CPU 数量
elif defined(HAVE_SYSCTL) && HAVE_DECL_CTL_HW && HAVE_DECL_HW_NCPU:
  # 定义名称数组和变量 nn
  static int name[2] = {CTL_HW, HW_NCPU};
  int64_t nn;
  # 如果无法通过 sysctl 获取 CPU 数量，则设置为 -1
  if (hwloc_get_sysctl(name, sizeof(name)/sizeof(*name), &nn))
    n = -1;
  n = nn;
# 如果以上条件都不满足
else:
  # 如果是 GNU 编译器，则输出警告信息
  #warning No known way to discover number of available processors on this system
  # 设置 CPU 数量为 -1
  n = -1;
# 返回 CPU 数量
return n;
}

# 获取系统内存大小的回退函数
int64_t
hwloc_fallback_memsize(void) {
  int64_t size;
# 如果系统支持获取主机基本信息
#if defined(HAVE_HOST_INFO) && HAVE_HOST_INFO:
  # 定义主机基本信息结构体和变量
  struct host_basic_info info;
  mach_msg_type_number_t count = HOST_BASIC_INFO_COUNT;
  # 获取主机基本信息
  host_info(mach_host_self(), HOST_BASIC_INFO, (integer_t*) &info, &count);
  # 获取内存大小
  size = info.memory_size;
# 如果系统支持通过 sysctl 获取内存大小
elif defined(HAVE_SYSCTL) && HAVE_DECL_CTL_HW && (HAVE_DECL_HW_REALMEM64 || HAVE_DECL_HW_MEMSIZE64 || HAVE_DECL_HW_PHYSMEM64 || HAVE_DECL_HW_USERMEM64 || HAVE_DECL_HW_REALMEM || HAVE_DECL_HW_MEMSIZE || HAVE_DECL_HW_PHYSMEM || HAVE_DECL_HW_USERMEM)
# 根据不同的宏定义获取内存大小
#if HAVE_DECL_HW_MEMSIZE64
  static int name[2] = {CTL_HW, HW_MEMSIZE64};
#elif HAVE_DECL_HW_REALMEM64
  static int name[2] = {CTL_HW, HW_REALMEM64};
#elif HAVE_DECL_HW_PHYSMEM64
  static int name[2] = {CTL_HW, HW_PHYSMEM64};
#elif HAVE_DECL_HW_USERMEM64
  static int name[2] = {CTL_HW, HW_USERMEM64};
#elif HAVE_DECL_HW_MEMSIZE
  static int name[2] = {CTL_HW, HW_MEMSIZE};
#elif HAVE_DECL_HW_REALMEM
  static int name[2] = {CTL_HW, HW_REALMEM};
#elif HAVE_DECL_HW_PHYSMEM
  // 如果定义了 HAVE_DECL_HW_PHYSMEM，则使用 HW_PHYSMEM 作为参数创建一个静态整型数组 name
  static int name[2] = {CTL_HW, HW_PHYSMEM};
#elif HAVE_DECL_HW_USERMEM
  // 如果定义了 HAVE_DECL_HW_USERMEM，则使用 HW_USERMEM 作为参数创建一个静态整型数组 name
  static int name[2] = {CTL_HW, HW_USERMEM};
#endif
  // 如果 hwloc_get_sysctl 函数返回非零值，则将 size 设置为 -1
  if (hwloc_get_sysctl(name, sizeof(name)/sizeof(*name), &size))
    size = -1;
#elif defined(HAVE_SYSCTLBYNAME)
  // 如果定义了 HAVE_SYSCTLBYNAME，则尝试使用 hw.memsize、hw.realmem、hw.physmem、hw.usermem 获取系统内存大小
  if (hwloc_get_sysctlbyname("hw.memsize", &size) &&
      hwloc_get_sysctlbyname("hw.realmem", &size) &&
      hwloc_get_sysctlbyname("hw.physmem", &size) &&
      hwloc_get_sysctlbyname("hw.usermem", &size))
      size = -1;
#else
  // 否则将 size 设置为 -1
  size = -1;
#endif
  // 返回 size
  return size;
}
#endif /* !HWLOC_WIN_SYS */

/*
 * Use the given number of processors to set a PU level.
 */
void
hwloc_setup_pu_level(struct hwloc_topology *topology,
             unsigned nb_pus)
{
  struct hwloc_obj *obj;
  unsigned oscpu,cpu;

  // 输出调试信息
  hwloc_debug("%s", "\n\n * CPU cpusets *\n\n");
  // 遍历处理器核心
  for (cpu=0,oscpu=0; cpu<nb_pus; oscpu++)
    {
      // 分配并设置处理器核心对象
      obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, oscpu);
      obj->cpuset = hwloc_bitmap_alloc();
      // 将处理器核心的位图设置为只包含当前处理器核心
      hwloc_bitmap_only(obj->cpuset, oscpu);

      // 输出调试信息
      hwloc_debug_2args_bitmap("cpu %u (os %u) has cpuset %s\n",
         cpu, oscpu, obj->cpuset);
      // 将处理器核心对象插入拓扑结构中
      hwloc__insert_object_by_cpuset(topology, NULL, obj, "core:pulevel");

      cpu++;
    }
}

/* Traverse children of a parent in a safe way: reread the next pointer as
 * appropriate to prevent crash on child deletion:  */
// 定义一个宏，用于安全地遍历父对象的子对象
#define for_each_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->first_child, child = *pchild; \
       child; \
       /* 检查当前子对象是否已被删除 */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* 获取下一个子对象的指针 */ \
        child = *pchild)
# 定义一个宏，用于安全地遍历内存子节点
#define for_each_memory_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->memory_first_child, child = *pchild; \
       child; \
       /* 检查当前子节点是否已被删除。 */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* 获取下一个子节点的指针。 */ \
        child = *pchild)
# 定义一个宏，用于安全地遍历 I/O 子节点
#define for_each_io_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->io_first_child, child = *pchild; \
       child; \
       /* 检查当前子节点是否已被删除。 */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* 获取下一个子节点的指针。 */ \
        child = *pchild)
# 定义一个宏，用于安全地遍历其他子节点
#define for_each_misc_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->misc_first_child, child = *pchild; \
       child; \
       /* 检查当前子节点是否已被删除。 */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* 获取下一个子节点的指针。 */ \
        child = *pchild)

#ifdef HWLOC_DEBUG
/* 仅用于调试。 */
static void
hwloc_debug_print_object(int indent __hwloc_attribute_unused, hwloc_obj_t obj)
{
  char type[64], idx[12], attr[1024], *cpuset = NULL;
  hwloc_debug("%*s", 2*indent, "");
  hwloc_obj_type_snprintf(type, sizeof(type), obj, 1);
  if (obj->os_index != HWLOC_UNKNOWN_INDEX)
    snprintf(idx, sizeof(idx), "#%u", obj->os_index);
  else
    *idx = '\0';
  hwloc_obj_attr_snprintf(attr, sizeof(attr), obj, " ", 1);
  hwloc_debug("%s%s%s%s%s", type, idx, *attr ? "(" : "", attr, *attr ? ")" : "");
  if (obj->name)
    hwloc_debug(" name \"%s\"", obj->name);
  if (obj->subtype)
    hwloc_debug(" subtype \"%s\"", obj->subtype);
  if (obj->cpuset) {
    hwloc_bitmap_asprintf(&cpuset, obj->cpuset);
    hwloc_debug(" cpuset %s", cpuset);
    free(cpuset);
  }
  if (obj->complete_cpuset) {
    hwloc_bitmap_asprintf(&cpuset, obj->complete_cpuset);
    hwloc_debug(" complete %s", cpuset);
    # 释放 cpuset 变量所指向的内存空间
    free(cpuset);
  }
  # 如果 obj 的 nodeset 不为空
  if (obj->nodeset) {
    # 将 obj 的 nodeset 转换为字符串格式，并赋值给 cpuset
    hwloc_bitmap_asprintf(&cpuset, obj->nodeset);
    # 打印 nodeset 的值
    hwloc_debug(" nodeset %s", cpuset);
    # 释放 cpuset 变量所指向的内存空间
    free(cpuset);
  }
  # 如果 obj 的 complete_nodeset 不为空
  if (obj->complete_nodeset) {
    # 将 obj 的 complete_nodeset 转换为字符串格式，并赋值给 cpuset
    hwloc_bitmap_asprintf(&cpuset, obj->complete_nodeset);
    # 打印 complete_nodeset 的值
    hwloc_debug(" completeN %s", cpuset);
    # 释放 cpuset 变量所指向的内存空间
    free(cpuset);
  }
  # 如果 obj 的 arity 不为 0
  if (obj->arity)
    # 打印 obj 的 arity 值
    hwloc_debug(" arity %u", obj->arity);
  # 打印换行符
  hwloc_debug("%s", "\n");
static void
hwloc_debug_print_objects(int indent __hwloc_attribute_unused, hwloc_obj_t obj)
{
  // 如果调试级别大于等于2，打印对象信息
  if (hwloc_debug_enabled() >= 2) {
    hwloc_obj_t child;
    hwloc_debug_print_object(indent, obj);
    // 遍历子对象并打印信息
    for_each_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    // 遍历内存子对象并打印信息
    for_each_memory_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    // 遍历IO子对象并打印信息
    for_each_io_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    // 遍历其他子对象并打印信息
    for_each_misc_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
  }
}
#else /* !HWLOC_DEBUG */
// 如果未开启调试，定义为空操作
#define hwloc_debug_print_object(indent, obj) do { /* nothing */ } while (0)
#define hwloc_debug_print_objects(indent, obj) do { /* nothing */ } while (0)
#endif /* !HWLOC_DEBUG */

// 释放信息数组
void hwloc__free_infos(struct hwloc_info_s *infos, unsigned count)
{
  unsigned i;
  // 遍历信息数组，释放name和value
  for(i=0; i<count; i++) {
    free(infos[i].name);
    free(infos[i].value);
  }
  // 释放信息数组
  free(infos);
}

// 添加信息到信息数组
int hwloc__add_info(struct hwloc_info_s **infosp, unsigned *countp, const char *name, const char *value)
{
  unsigned count = *countp;
  struct hwloc_info_s *infos = *infosp;
#define OBJECT_INFO_ALLOC 8
  // 初始未分配内存，按8的倍数重新分配
  unsigned alloccount = (count + 1 + (OBJECT_INFO_ALLOC-1)) & ~(OBJECT_INFO_ALLOC-1);
  if (count != alloccount) {
    struct hwloc_info_s *tmpinfos = realloc(infos, alloccount*sizeof(*infos));
    if (!tmpinfos)
      // 分配失败，忽略此信息
      goto out_with_array;
    *infosp = infos = tmpinfos;
  }
  // 复制name到infos数组
  infos[count].name = strdup(name);
  if (!infos[count].name)
    goto out_with_array;
  // 复制value到infos数组
  infos[count].value = strdup(value);
  if (!infos[count].value)
    goto out_with_name;
  *countp = count+1;
  return 0;

 out_with_name:
  free(infos[count].name);
 out_with_array:
  // 不减少数组大小
  return -1;
}
# 向信息列表中添加信息，如果已存在同名信息则根据标志位决定是否替换
int hwloc__add_info_nodup(struct hwloc_info_s **infosp, unsigned *countp,
              const char *name, const char *value,
              int replace)
{
  # 获取信息列表和信息数量
  struct hwloc_info_s *infos = *infosp;
  unsigned count = *countp;
  unsigned i;
  # 遍历信息列表
  for(i=0; i<count; i++) {
    # 如果存在同名信息
    if (!strcmp(infos[i].name, name)) {
      # 如果需要替换
      if (replace) {
        # 分配新的值并替换原有值
        char *new = strdup(value);
        if (!new)
          return -1;
        free(infos[i].value);
        infos[i].value = new;
      }
      # 返回成功
      return 0;
    }
  }
  # 如果不存在同名信息，则调用添加信息的函数
  return hwloc__add_info(infosp, countp, name, value);
}

# 将源信息列表中的信息移动到目标信息列表中
int hwloc__move_infos(struct hwloc_info_s **dst_infosp, unsigned *dst_countp,
              struct hwloc_info_s **src_infosp, unsigned *src_countp)
{
  # 获取目标信息列表和数量
  unsigned dst_count = *dst_countp;
  struct hwloc_info_s *dst_infos = *dst_infosp;
  # 获取源信息列表和数量
  unsigned src_count = *src_countp;
  struct hwloc_info_s *src_infos = *src_infosp;
  unsigned i;
  # 定义每次分配的信息数量
#define OBJECT_INFO_ALLOC 8
  /* nothing allocated initially, (re-)allocate by multiple of 8 */
  # 计算需要分配的信息数量
  unsigned alloccount = (dst_count + src_count + (OBJECT_INFO_ALLOC-1)) & ~(OBJECT_INFO_ALLOC-1);
  # 如果目标信息列表的大小不够，则重新分配内存
  if (dst_count != alloccount) {
    struct hwloc_info_s *tmp_infos = realloc(dst_infos, alloccount*sizeof(*dst_infos));
    if (!tmp_infos)
      /* Failed to realloc, ignore the appended infos */
      goto drop;
    dst_infos = tmp_infos;
  }
  # 将源信息列表中的信息移动到目标信息列表中
  for(i=0; i<src_count; i++, dst_count++) {
    dst_infos[dst_count].name = src_infos[i].name;
    dst_infos[dst_count].value = src_infos[i].value;
  }
  *dst_infosp = dst_infos;
  *dst_countp = dst_count;
  free(src_infos);
  *src_infosp = NULL;
  *src_countp = 0;
  return 0;

 drop:
  /* drop src infos, don't modify dst_infos at all */
  # 如果重新分配内存失败，则释放源信息列表的内存
  for(i=0; i<src_count; i++) {
    free(src_infos[i].name);
    free(src_infos[i].value);
  }
  free(src_infos);
  *src_infosp = NULL;
  *src_countp = 0;
  return -1;
}

# 向对象的信息列表中添加信息
int hwloc_obj_add_info(hwloc_obj_t obj, const char *name, const char *value)
{
  return hwloc__add_info(&obj->infos, &obj->infos_count, name, value);
}
/* This function duplicates the information array of an object, allocating new memory from the given tma */
int hwloc__tma_dup_infos(struct hwloc_tma *tma,
                         struct hwloc_info_s **newip, unsigned *newcp,
                         struct hwloc_info_s *oldi, unsigned oldc)
{
  struct hwloc_info_s *newi; // 定义新的信息数组指针
  unsigned i, j; // 定义循环变量
  newi = hwloc_tma_calloc(tma, oldc * sizeof(*newi)); // 从tma中分配新的信息数组内存
  if (!newi) // 如果分配失败
    return -1; // 返回错误
  for(i=0; i<oldc; i++) { // 遍历旧的信息数组
    newi[i].name = hwloc_tma_strdup(tma, oldi[i].name); // 从tma中分配新的字符串内存，并复制旧的名称
    newi[i].value = hwloc_tma_strdup(tma, oldi[i].value); // 从tma中分配新的字符串内存，并复制旧的值
    if (!newi[i].name || !newi[i].value) // 如果分配失败
      goto failed; // 跳转到失败标签
  }
  *newip = newi; // 将新的信息数组指针赋值给传入的指针
  *newcp = oldc; // 将旧的信息数组长度赋值给传入的长度指针
  return 0; // 返回成功

 failed:
  assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
  for(j=0; j<=i; j++) { // 遍历失败前已经处理的信息
    free(newi[i].name); // 释放新的名称字符串内存
    free(newi[i].value); // 释放新的值字符串内存
  }
  free(newi); // 释放新的信息数组内存
  *newip = NULL; // 将传入的指针置空
  return -1; // 返回错误
}

static void
hwloc__free_object_contents(hwloc_obj_t obj)
{
  switch (obj->type) { // 根据对象类型进行处理
  case HWLOC_OBJ_NUMANODE:
    free(obj->attr->numanode.page_types); // 释放NUMANODE对象的页面类型数组内存
    break;
  default:
    break;
  }
  hwloc__free_infos(obj->infos, obj->infos_count); // 释放对象的信息数组内存
  free(obj->attr); // 释放对象的属性内存
  free(obj->children); // 释放对象的子对象数组内存
  free(obj->subtype); // 释放对象的子类型字符串内存
  free(obj->name); // 释放对象的名称字符串内存
  hwloc_bitmap_free(obj->cpuset); // 释放对象的cpuset位图内存
  hwloc_bitmap_free(obj->complete_cpuset); // 释放对象的complete_cpuset位图内存
  hwloc_bitmap_free(obj->nodeset); // 释放对象的nodeset位图内存
  hwloc_bitmap_free(obj->complete_nodeset); // 释放对象的complete_nodeset位图内存
}

/* Free an object and all its content.  */
void
hwloc_free_unlinked_object(hwloc_obj_t obj)
{
  hwloc__free_object_contents(obj); // 释放对象及其所有内容
  free(obj); // 释放对象内存
}

/* Replace old with contents of new object, and make new freeable by the caller.
 * Requires reconnect (for siblings pointers and group depth),
 * fixup of sets (only the main cpuset was likely compared before merging),
 * and update of total_memory and group depth.
 */
static void
hwloc_replace_linked_object(hwloc_obj_t old, hwloc_obj_t new)
{
  /* 丢弃旧字段 */
  hwloc__free_object_contents(old);
  /* 将旧树指针复制到新树 */
  new->parent = old->parent;
  new->next_sibling = old->next_sibling;
  new->first_child = old->first_child;
  new->memory_first_child = old->memory_first_child;
  new->io_first_child = old->io_first_child;
  new->misc_first_child = old->misc_first_child;
  /* 现在树指针已经正确，将新内容复制到旧内容 */
  memcpy(old, new, sizeof(*old));
  /* 清空新内容以便释放 */
  memset(new, 0,sizeof(*new));
}

/* 从其父对象中移除对象及其子对象并释放它们。
 * 只更新 next_sibling/first_child 指针，
 * 因此只能在早期发现或销毁期间使用。
 */
static void
unlink_and_free_object_and_children(hwloc_obj_t *pobj)
{
  hwloc_obj_t obj = *pobj, child, *pchild;

  for_each_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_memory_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_io_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_misc_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);

  *pobj = obj->next_sibling;
  hwloc_free_unlinked_object(obj);
}

/* 释放对象及其子对象，但不从父对象中断开链接。
 */
void
hwloc_free_object_and_children(hwloc_obj_t obj)
{
  unlink_and_free_object_and_children(&obj);
}

/* 释放对象、其下一个兄弟对象及其子对象，但不从父对象中断开链接。
 */
void
hwloc_free_object_siblings_and_children(hwloc_obj_t obj)
{
  while (obj)
    unlink_and_free_object_and_children(&obj);
}

/* 将从 firstnew 开始的兄弟对象列表插入为 newparent 的新子对象，
 * 并返回指向下一个对象指针的地址
 */
static hwloc_obj_t *
insert_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t tmp;  // 声明一个名为tmp的hwloc_obj_t类型的变量
  assert(firstnew);  // 确保firstnew不为空，如果为空则触发断言错误
  *firstp = tmp = firstnew;  // 将firstnew赋值给*firstp，并将tmp也赋值为firstnew
  tmp->parent = newparent;  // 将tmp的parent指针指向newparent
  while (tmp->next_sibling) {  // 当tmp的next_sibling不为空时循环
    tmp = tmp->next_sibling;  // 将tmp指向下一个兄弟节点
    tmp->parent = newparent;  // 将tmp的parent指针指向newparent
  }
  return &tmp->next_sibling;  // 返回指向tmp的下一个兄弟节点的指针
}

/* Take the new list starting at firstnew and prepend it to the old list starting at *firstp,
 * and mark the new children as children of newparent.
 * May be used during early or late discovery (updates prev_sibling and sibling_rank).
 * List firstnew must be non-NULL.
 */
static void
prepend_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t *tmpp, tmp, last;  // 声明指针tmpp和变量tmp、last
  unsigned length;  // 声明无符号整型变量length

  /* update parent pointers and find the length and end of the new list */
  for(length = 0, tmpp = &firstnew, last = NULL ; *tmpp; length++, last = *tmpp, tmpp = &((*tmpp)->next_sibling))
    (*tmpp)->parent = newparent;  // 更新parent指针为newparent，并找到新列表的长度和结尾

  /* update sibling_rank */
  for(tmp = *firstp; tmp; tmp = tmp->next_sibling)
    tmp->sibling_rank += length; /* if it wasn't initialized yet, it'll be overwritten later */  // 更新sibling_rank

  /* place the existing list at the end of the new one */
  *tmpp = *firstp;  // 将现有列表放在新列表的末尾
  if (*firstp)
    (*firstp)->prev_sibling = last;  // 如果*firstp不为空，则将其prev_sibling指向last

  /* use the beginning of the new list now */
  *firstp = firstnew;  // 现在使用新列表的开头
}

/* Take the new list starting at firstnew and append it to the old list starting at *firstp,
 * and mark the new children as children of newparent.
 * May be used during early or late discovery (updates prev_sibling and sibling_rank).
 */
static void
append_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t *tmpp, tmp, last;  // 声明指针tmpp和变量tmp、last
  unsigned length;  // 声明无符号整型变量length

  /* find the length and end of the existing list */
  for(length = 0, tmpp = firstp, last = NULL ; *tmpp; length++, last = *tmpp, tmpp = &((*tmpp)->next_sibling));

  /* update parent pointers and sibling_rank */
  for(tmp = firstnew; tmp; tmp = tmp->next_sibling) {
    tmp->parent = newparent;  // 更新parent指针为newparent
    # 将当前节点的兄弟节点排名增加指定长度
    tmp->sibling_rank += length; /* 如果它还没有设置，它将在后面被覆盖 */
  }

  /* 将新列表放在旧列表的末尾 */
  *tmpp = firstnew;
  # 如果存在新列表
  if (firstnew)
    # 设置新列表的前一个兄弟节点为旧列表的最后一个节点
    firstnew->prev_sibling = last;
/* 从其父对象中移除一个对象并释放它。
 * 只更新 next_sibling/first_child 指针，
 * 因此只能在早期发现期间使用。
 *
 * 子对象被插入到父对象中。
 * 如果子对象应该被插入到其他地方（例如与子对象合并时），
 * 调用者在调用此函数之前应该移动它们。
 */
static void
unlink_and_free_single_object(hwloc_obj_t *pparent)
{
  hwloc_obj_t old = *pparent;
  hwloc_obj_t *lastp;

  if (old->type == HWLOC_OBJ_MISC) {
    /* 杂项对象 */

    /* 没有普通子对象 */
    assert(!old->first_child);
    /* 没有内存子对象 */
    assert(!old->memory_first_child);
    /* 没有 I/O 子对象 */
    assert(!old->io_first_child);

    if (old->misc_first_child)
      /* 将旧杂项对象的子对象作为新的兄弟对象插入到父对象下而不是旧对象下 */
      lastp = insert_siblings_list(pparent, old->misc_first_child, old->parent);
    else
      lastp = pparent;
    /* 将旧兄弟对象重新添加回去 */
    *lastp = old->next_sibling;

  } else if (hwloc__obj_type_is_io(old->type)) {
    /* I/O 对象 */

    /* 没有普通子对象 */
    assert(!old->first_child);
    /* 没有内存子对象 */
    assert(!old->memory_first_child);

    if (old->io_first_child)
      /* 将旧 I/O 对象的子对象作为新的兄弟对象插入到父对象下而不是旧对象下 */
      lastp = insert_siblings_list(pparent, old->io_first_child, old->parent);
    else
      lastp = pparent;
    /* 将旧兄弟对象重新添加回去 */
    *lastp = old->next_sibling;

    /* 将旧杂项子对象追加到父对象 */
    if (old->misc_first_child)
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);

  } else if (hwloc__obj_type_is_memory(old->type)) {
    /* 内存对象 */

    /* 没有普通子对象 */
    assert(!old->first_child);
    /* 没有 I/O 子对象 */
    assert(!old->io_first_child);
    if (old->memory_first_child)
      /* 如果旧对象有内存子对象，则将其插入到父对象下面作为新的兄弟对象，而不是旧对象的位置 */
      lastp = insert_siblings_list(pparent, old->memory_first_child, old->parent);
    else
      // 否则，将父对象作为最后一个兄弟对象
      lastp = pparent;
    /* 将旧对象的兄弟对象重新添加回去 */
    *lastp = old->next_sibling;

    /* 将旧的 Misc 子对象追加到父对象 */
    if (old->misc_first_child)
      // 如果旧对象有 Misc 子对象，则将其追加到父对象的 Misc 子对象列表中
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);

  } else {
    /* 普通对象 */

    if (old->first_child)
      /* 如果旧对象有子对象，则将其插入到父对象下面作为新的兄弟对象，而不是旧对象的位置 */
      lastp = insert_siblings_list(pparent, old->first_child, old->parent);
    else
      // 否则，将父对象作为最后一个兄弟对象
      lastp = pparent;
    /* 将旧对象的兄弟对象重新添加回去 */
    *lastp = old->next_sibling;

    /* 将旧的内存、I/O 和 Misc 子对象追加到父对象
     * old->parent 不能为 NULL（移除根对象），Misc 子对象应该在之前由调用者移动过了
     */
    if (old->memory_first_child)
      // 如果旧对象有内存子对象，则将其追加到父对象的内存子对象列表中
      append_siblings_list(&old->parent->memory_first_child, old->memory_first_child, old->parent);
    if (old->io_first_child)
      // 如果旧对象有 I/O 子对象，则将其追加到父对象的 I/O 子对象列表中
      append_siblings_list(&old->parent->io_first_child, old->io_first_child, old->parent);
    if (old->misc_first_child)
      // 如果旧对象有 Misc 子对象，则将其追加到父对象的 Misc 子对象列表中
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);
  }

  // 释放未连接的对象
  hwloc_free_unlinked_object(old);
/* 
   这个函数可能使用一个 tma，不能使用 free() 或 realloc()
   这个函数用于复制一个 hwloc_obj 对象
*/
static int
hwloc__duplicate_object(struct hwloc_topology *newtopology,
            struct hwloc_obj *newparent,
            struct hwloc_obj *newobj,
            struct hwloc_obj *src)
{
  // 获取新拓扑结构体的 tma
  struct hwloc_tma *tma = newtopology->tma;
  hwloc_obj_t *level;
  unsigned level_width;
  size_t len;
  unsigned i;
  hwloc_obj_t child, prev;
  int err = 0;

  /* 要么我们正在复制到已经分配的新根对象，没有新的父对象，
   * 要么我们正在复制到尚未分配的新非根对象，它将有一个新的父对象。
   */
  assert(!newparent == !!newobj);

  if (!newobj) {
    // 如果没有新对象，分配并设置新对象
    newobj = hwloc_alloc_setup_object(newtopology, src->type, src->os_index);
    if (!newobj)
      return -1;
  }

  /* 复制所有非对象指针字段 */
  newobj->logical_index = src->logical_index;
  newobj->depth = src->depth;
  newobj->sibling_rank = src->sibling_rank;

  newobj->type = src->type;
  newobj->os_index = src->os_index;
  newobj->gp_index = src->gp_index;
  newobj->symmetric_subtree = src->symmetric_subtree;

  if (src->name)
    // 复制名称
    newobj->name = hwloc_tma_strdup(tma, src->name);
  if (src->subtype)
    // 复制子类型
    newobj->subtype = hwloc_tma_strdup(tma, src->subtype);
  newobj->userdata = src->userdata;

  newobj->total_memory = src->total_memory;

  // 复制属性
  memcpy(newobj->attr, src->attr, sizeof(*newobj->attr));

  if (src->type == HWLOC_OBJ_NUMANODE && src->attr->numanode.page_types_len) {
    len = src->attr->numanode.page_types_len * sizeof(struct hwloc_memory_page_type_s);
    // 分配内存并复制页面类型
    newobj->attr->numanode.page_types = hwloc_tma_malloc(tma, len);
  // 复制源对象的numanode.page_types到新对象的numanode.page_types
  memcpy(newobj->attr->numanode.page_types, src->attr->numanode.page_types, len);
  // 复制源对象的cpuset到新对象的cpuset
  newobj->cpuset = hwloc_bitmap_tma_dup(tma, src->cpuset);
  // 复制源对象的complete_cpuset到新对象的complete_cpuset
  newobj->complete_cpuset = hwloc_bitmap_tma_dup(tma, src->complete_cpuset);
  // 复制源对象的nodeset到新对象的nodeset
  newobj->nodeset = hwloc_bitmap_tma_dup(tma, src->nodeset);
  // 复制源对象的complete_nodeset到新对象的complete_nodeset
  newobj->complete_nodeset = hwloc_bitmap_tma_dup(tma, src->complete_nodeset);

  // 复制源对象的infos到新对象的infos
  hwloc__tma_dup_infos(tma, &newobj->infos, &newobj->infos_count, src->infos, src->infos_count);

  /* 找到新对象所在的层级 */
  if (src->depth < 0) {
    i = HWLOC_SLEVEL_FROM_DEPTH(src->depth);
    level = newtopology->slevels[i].objs;
    level_width = newtopology->slevels[i].nbobjs;
    /* 处理特殊层级的first/last指针，即使实际上不需要 */
    if (!newobj->logical_index)
      newtopology->slevels[i].first = newobj;
    if (newobj->logical_index == newtopology->slevels[i].nbobjs - 1)
      newtopology->slevels[i].last = newobj;
  } else {
    level = newtopology->levels[src->depth];
    level_width = newtopology->level_nbobjects[src->depth];
  }
  /* 将新对象放置到层级中 */
  assert(newobj->logical_index < level_width);
  level[newobj->logical_index] = newobj;
  /* 链接到已插入的兄弟节点 */
  if (newobj->logical_index > 0 && level[newobj->logical_index-1]) {
    newobj->prev_cousin = level[newobj->logical_index-1];
    level[newobj->logical_index-1]->next_cousin = newobj;
  }
  if (newobj->logical_index < level_width-1 && level[newobj->logical_index+1]) {
    newobj->next_cousin = level[newobj->logical_index+1];
    level[newobj->logical_index+1]->prev_cousin = newobj;
  }

  /* 为子节点做准备 */
  if (src->arity) {
    newobj->children = hwloc_tma_malloc(tma, src->arity * sizeof(*newobj->children));
    if (!newobj->children)
      return -1;
  }
  newobj->arity = src->arity;
  newobj->memory_arity = src->memory_arity;
  newobj->io_arity = src->io_arity;
  newobj->misc_arity = src->misc_arity;

  /* 现在实际插入子节点 */
  for_each_child(child, src) {
    # 复制对象的子对象到新拓扑结构中
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    # 如果复制失败，则跳转到释放子对象资源的标签
    if (err < 0)
      goto out_with_children;
  }
  # 遍历内存子对象，将其复制到新拓扑结构中
  for_each_memory_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    # 如果复制失败，则直接返回错误
    if (err < 0)
      return err;
  }
  # 遍历 I/O 子对象，将其复制到新拓扑结构中
  for_each_io_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    # 如果复制失败，则跳转到释放子对象资源的标签
    if (err < 0)
      goto out_with_children;
  }
  # 遍历其他子对象，将其复制到新拓扑结构中
  for_each_misc_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    # 如果复制失败，则跳转到释放子对象资源的标签
    if (err < 0)
      goto out_with_children;
  }

 out_with_children:

  # 如果所有子对象都被成功插入，则链接它们
  if (!err) {
    # 设置子对象的兄弟关系
    if (newobj->arity) {
      newobj->children[0]->prev_sibling = NULL;
      for(i=1; i<newobj->arity; i++)
    newobj->children[i]->prev_sibling = newobj->children[i-1];
      newobj->last_child = newobj->children[newobj->arity-1];
    }
    # 设置内存子对象的兄弟关系
    if (newobj->memory_arity) {
      child = newobj->memory_first_child;
      prev = NULL;
      while (child) {
    child->prev_sibling = prev;
    prev = child;
    child = child->next_sibling;
      }
    }
    # 设置 I/O 子对象的兄弟关系
    if (newobj->io_arity) {
      child = newobj->io_first_child;
      prev = NULL;
      while (child) {
    child->prev_sibling = prev;
    prev = child;
    child = child->next_sibling;
      }
    }
    # 设置其他子对象的兄弟关系
    if (newobj->misc_arity) {
      child = newobj->misc_first_child;
      prev = NULL;
      while (child) {
    child->prev_sibling = prev;
    prev = child;
    child = child->next_sibling;
      }
    }
  }

  # 如果存在父对象，则继续插入自身并让调用者清理整个树结构
  if (newparent) {
    /* 不需要在这里检查子节点的插入顺序，源拓扑结构应该已经是正确的，而且我们有调试断言。 */
    hwloc_insert_object_by_parent(newtopology, newparent, newobj);

    /* 将我们放置在父节点的子节点数组中 */
    if (hwloc__obj_type_is_normal(newobj->type))
      newparent->children[newobj->sibling_rank] = newobj;
  }

  return err;
# 静态函数，用于初始化拓扑结构
static int
hwloc__topology_init (struct hwloc_topology **topologyp, unsigned nblevels, struct hwloc_tma *tma);

# 复制拓扑结构
int
hwloc__topology_dup(hwloc_topology_t *newp,
            hwloc_topology_t old,
            struct hwloc_tma *tma)
{
  # 定义新拓扑结构
  hwloc_topology_t new;
  # 定义新拓扑结构的根节点
  hwloc_obj_t newroot;
  # 获取旧拓扑结构的根节点
  hwloc_obj_t oldroot = hwloc_get_root_obj(old);
  # 定义变量
  unsigned i;
  int err;

  # 如果旧拓扑结构未加载，则返回错误
  if (!old->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  # 初始化新拓扑结构
  err = hwloc__topology_init(&new, old->nb_levels_allocated, tma);
  if (err < 0)
    goto out;

  # 复制旧拓扑结构的属性到新拓扑结构
  new->flags = old->flags;
  memcpy(new->type_filter, old->type_filter, sizeof(old->type_filter));
  new->is_thissystem = old->is_thissystem;
  new->is_loaded = 1;
  new->pid = old->pid;
  new->next_gp_index = old->next_gp_index;

  memcpy(&new->binding_hooks, &old->binding_hooks, sizeof(old->binding_hooks));

  memcpy(new->support.discovery, old->support.discovery, sizeof(*old->support.discovery));
  memcpy(new->support.cpubind, old->support.cpubind, sizeof(*old->support.cpubind));
  memcpy(new->support.membind, old->support.membind, sizeof(*old->support.membind));
  memcpy(new->support.misc, old->support.misc, sizeof(*old->support.misc));

  # 复制旧拓扑结构的允许CPU集合和节点集合到新拓扑结构
  new->allowed_cpuset = hwloc_bitmap_tma_dup(tma, old->allowed_cpuset);
  new->allowed_nodeset = hwloc_bitmap_tma_dup(tma, old->allowed_nodeset);

  # 复制旧拓扑结构的用户数据导出和导入回调函数到新拓扑结构
  new->userdata_export_cb = old->userdata_export_cb;
  new->userdata_import_cb = old->userdata_import_cb;
  new->userdata_not_decoded = old->userdata_not_decoded;

  # 断言旧拓扑结构的机器内存属性为空
  assert(!old->machine_memory.local_memory);
  assert(!old->machine_memory.page_types_len);
  assert(!old->machine_memory.page_types);

  # 遍历硬件对象类型
  for(i = HWLOC_OBJ_TYPE_MIN; i < HWLOC_OBJ_TYPE_MAX; i++)
    // 将旧对象的类型深度复制给新对象
    new->type_depth[i] = old->type_depth[i];

  /* 复制层级，当复制对象时，我们将把对象放在这些层级上 */
  // 复制旧对象的层级数，并确保新对象分配的层级数大于等于实际层级数
  new->nb_levels = old->nb_levels;
  assert(new->nb_levels_allocated >= new->nb_levels);
  // 复制每个层级的对象数量，并为新对象的每个层级分配内存
  for(i=1 /* root level already allocated */ ; i<new->nb_levels; i++) {
    new->level_nbobjects[i] = old->level_nbobjects[i];
    new->levels[i] = hwloc_tma_calloc(tma, new->level_nbobjects[i] * sizeof(*new->levels[i]));
  }
  // 复制静态层级的对象数量，并为新对象的每个静态层级分配内存
  for(i=0; i<HWLOC_NR_SLEVELS; i++) {
    new->slevels[i].nbobjs = old->slevels[i].nbobjs;
    if (new->slevels[i].nbobjs)
      new->slevels[i].objs = hwloc_tma_calloc(tma, new->slevels[i].nbobjs * sizeof(*new->slevels[i].objs));
  }

  /* 递归地复制对象的子对象 */
  // 获取新对象的根对象
  newroot = hwloc_get_root_obj(new);
  // 复制对象及其子对象
  err = hwloc__duplicate_object(new, NULL, newroot, oldroot);
  if (err < 0)
    goto out_with_topology;

  // 复制内部距离信息
  err = hwloc_internal_distances_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  // 复制内存属性信息
  err = hwloc_internal_memattrs_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  // 复制 CPU 类型信息
  err = hwloc_internal_cpukinds_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  /* 我们在复制过程中已经连接了所有对象 */
  // 复制完成后，将 modified 标记设为 0
  new->modified = 0;

  /* 不需要复制后端，拓扑结构已经加载 */
  // 不需要复制后端信息，将其设为 NULL
  new->backends = NULL;
  new->get_pci_busid_cpuset_backend = NULL;
#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG 宏，则检查环境变量 HWLOC_DEBUG_CHECK 是否存在
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 对新拓扑进行检查
    hwloc_topology_check(new);

  // 将新拓扑指针指向新拓扑对象
  *newp = new;
  // 返回 0 表示成功
  return 0;

 out_with_topology:
  // 断言 tma 不存在或者 tma->dontfree 为假，即 tma 不能分配失败
  assert(!tma || !tma->dontfree); 
  // 销毁新拓扑对象
  hwloc_topology_destroy(new);
 out:
  // 返回 -1 表示失败
  return -1;
}

// 复制拓扑对象
int
hwloc_topology_dup(hwloc_topology_t *newp,
           hwloc_topology_t old)
{
  // 调用内部函数 hwloc__topology_dup 进行拓扑复制
  return hwloc__topology_dup(newp, old, NULL);
}

/* 警告：此数组的索引必须与下面的 obj_order_type[] 数组的顺序相匹配。
   具体来说，值必须按照以下方式布局：

       obj_order_type[obj_type_order[N]] = N

   对于所有的 HWLOC_OBJ_* 的 N 值。换句话说：

       obj_type_order[A] = B

   其中 A 值按照 hwloc_obj_type_t 枚举的顺序排列，B 值是 obj_order_type 的相应索引。

   我们无法使用 C99 语法以稍微安全的方式初始化这个数组 -- 遗憾。:-(

   当启用调试时，在 hwloc_topology_init() 中会断言正确性。
   */
/***** 确保你也更新了下面的 obj_type_priority[]。*****/
static const unsigned obj_type_order[] = {
    /* 第一个条目是 HWLOC_OBJ_MACHINE */  0,
    /* 下一个条目是 HWLOC_OBJ_PACKAGE */  4,
    /* 下一个条目是 HWLOC_OBJ_CORE */     14,
    /* 下一个条目是 HWLOC_OBJ_PU */       18,
    /* 下一个条目是 HWLOC_OBJ_L1CACHE */  12,
    /* 下一个条目是 HWLOC_OBJ_L2CACHE */  10,
    /* 下一个条目是 HWLOC_OBJ_L3CACHE */  8,
    /* 下一个条目是 HWLOC_OBJ_L4CACHE */  7,
    /* 下一个条目是 HWLOC_OBJ_L5CACHE */  6,
    /* 下一个条目是 HWLOC_OBJ_L1ICACHE */ 13,
    /* 下一个条目是 HWLOC_OBJ_L2ICACHE */ 11,
    /* 下一个条目是 HWLOC_OBJ_L3ICACHE */ 9,
    /* 下一个条目是 HWLOC_OBJ_GROUP */    1,
    /* 下一个条目是 HWLOC_OBJ_NUMANODE */ 3,
    /* 下一个条目是 HWLOC_OBJ_BRIDGE */   15,
    /* 下一个条目是 HWLOC_OBJ_PCI_DEVICE */  16,
    /* 下一个条目是 HWLOC_OBJ_OS_DEVICE */   17,
    /* 下一个条目是 HWLOC_OBJ_MISC */     19,
    # 下一个条目是 HWLOC_OBJ_MEMCACHE，值为 2
    2,
    # 下一个条目是 HWLOC_OBJ_DIE，值为 5
    5
};

#ifndef NDEBUG /* only used in debug check assert if !NDEBUG */
static const hwloc_obj_type_t obj_order_type[] = {
  HWLOC_OBJ_MACHINE,  // 机器对象类型
  HWLOC_OBJ_GROUP,  // 组对象类型
  HWLOC_OBJ_MEMCACHE,  // 内存缓存对象类型
  HWLOC_OBJ_NUMANODE,  // NUMA 节点对象类型
  HWLOC_OBJ_PACKAGE,  // 处理器包对象类型
  HWLOC_OBJ_DIE,  // 处理器核心组对象类型
  HWLOC_OBJ_L5CACHE,  // 第 5 级缓存对象类型
  HWLOC_OBJ_L4CACHE,  // 第 4 级缓存对象类型
  HWLOC_OBJ_L3CACHE,  // 第 3 级缓存对象类型
  HWLOC_OBJ_L3ICACHE,  // 第 3 级指令缓存对象类型
  HWLOC_OBJ_L2CACHE,  // 第 2 级缓存对象类型
  HWLOC_OBJ_L2ICACHE,  // 第 2 级指令缓存对象类型
  HWLOC_OBJ_L1CACHE,  // 第 1 级缓存对象类型
  HWLOC_OBJ_L1ICACHE,  // 第 1 级指令缓存对象类型
  HWLOC_OBJ_CORE,  // 处理器核心对象类型
  HWLOC_OBJ_BRIDGE,  // 桥接对象类型
  HWLOC_OBJ_PCI_DEVICE,  // PCI 设备对象类型
  HWLOC_OBJ_OS_DEVICE,  // 操作系统设备对象类型
  HWLOC_OBJ_PU,  // 处理器单元对象类型
  HWLOC_OBJ_MISC  // 其他对象类型，总是叶子节点
};
#endif
/***** Make sure you update obj_type_priority[] below as well. *****/

/* priority to be used when merging identical parent/children object
 * (in merge_useless_child), keep the highest priority one.
 *
 * Always keep Machine/NUMANode/PU/PCIDev/OSDev
 * then Core
 * then Package
 * then Die
 * then Cache,
 * then Instruction Caches
 * then always drop Group/Misc/Bridge.
 *
 * Some type won't actually ever be involved in such merging.
 */
/***** Make sure you update this array when changing the list of types. *****/
# 定义一个静态的常量数组，表示不同对象类型的优先级
static const int obj_type_priority[] = {
  /* first entry is HWLOC_OBJ_MACHINE */     90,  # 机器类型的优先级为90
  /* next entry is HWLOC_OBJ_PACKAGE */     40,   # 包类型的优先级为40
  /* next entry is HWLOC_OBJ_CORE */        60,   # 核心类型的优先级为60
  /* next entry is HWLOC_OBJ_PU */          100,  # 处理器类型的优先级为100
  /* next entry is HWLOC_OBJ_L1CACHE */     20,   # 一级缓存类型的优先级为20
  /* next entry is HWLOC_OBJ_L2CACHE */     20,   # 二级缓存类型的优先级为20
  /* next entry is HWLOC_OBJ_L3CACHE */     20,   # 三级缓存类型的优先级为20
  /* next entry is HWLOC_OBJ_L4CACHE */     20,   # 四级缓存类型的优先级为20
  /* next entry is HWLOC_OBJ_L5CACHE */     20,   # 五级缓存类型的优先级为20
  /* next entry is HWLOC_OBJ_L1ICACHE */    19,   # 一级指令缓存类型的优先级为19
  /* next entry is HWLOC_OBJ_L2ICACHE */    19,   # 二级指令缓存类型的优先级为19
  /* next entry is HWLOC_OBJ_L3ICACHE */    19,   # 三级指令缓存类型的优先级为19
  /* next entry is HWLOC_OBJ_GROUP */       0,    # 组类型的优先级为0
  /* next entry is HWLOC_OBJ_NUMANODE */    100,  # NUMA节点类型的优先级为100
  /* next entry is HWLOC_OBJ_BRIDGE */      0,    # 桥接类型的优先级为0
  /* next entry is HWLOC_OBJ_PCI_DEVICE */  100,  # PCI设备类型的优先级为100
  /* next entry is HWLOC_OBJ_OS_DEVICE */   100,  # 操作系统设备类型的优先级为100
  /* next entry is HWLOC_OBJ_MISC */        0,    # 杂项类型的优先级为0
  /* next entry is HWLOC_OBJ_MEMCACHE */    19,   # 内存缓存类型的优先级为19
  /* next entry is HWLOC_OBJ_DIE */         30    # DIE类型的优先级为30
};

# 比较两个对象类型的优先级
int hwloc_compare_types (hwloc_obj_type_t type1, hwloc_obj_type_t type2)
{
  unsigned order1 = obj_type_order[type1];  # 获取第一个对象类型的优先级
  unsigned order2 = obj_type_order[type2];  # 获取第二个对象类型的优先级

  # 只有普通对象可以进行比较，其他对象只能与机器类型进行比较
  if (!hwloc__obj_type_is_normal(type1)
      && hwloc__obj_type_is_normal(type2) && type2 != HWLOC_OBJ_MACHINE)
    return HWLOC_TYPE_UNORDERED;  # 如果第一个对象类型不是普通类型，而第二个对象类型是普通类型且不是机器类型，则返回无序
  if (!hwloc__obj_type_is_normal(type2)
      && hwloc__obj_type_is_normal(type1) && type1 != HWLOC_OBJ_MACHINE)
    return HWLOC_TYPE_UNORDERED;  # 如果第二个对象类型不是普通类型，而第一个对象类型是普通类型且不是机器类型，则返回无序

  return order1 - order2;  # 返回两个对象类型的优先级差值
}
# 定义枚举类型 hwloc_obj_cmp_e，用于表示不同的对象比较结果
enum hwloc_obj_cmp_e {
  HWLOC_OBJ_EQUAL = HWLOC_BITMAP_EQUAL,            /**< \brief Equal */  # 表示相等
  HWLOC_OBJ_INCLUDED = HWLOC_BITMAP_INCLUDED,        /**< \brief Strictly included into */  # 表示严格包含在内
  HWLOC_OBJ_CONTAINS = HWLOC_BITMAP_CONTAINS,        /**< \brief Strictly contains */  # 表示严格包含
  HWLOC_OBJ_INTERSECTS = HWLOC_BITMAP_INTERSECTS,    /**< \brief Intersects, but no inclusion! */  # 表示相交，但不包含
  HWLOC_OBJ_DIFFERENT = HWLOC_BITMAP_DIFFERENT        /**< \brief No intersection */  # 表示没有交集
};

# 定义静态函数 hwloc_type_cmp，用于比较两个对象的类型
static enum hwloc_obj_cmp_e
hwloc_type_cmp(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  hwloc_obj_type_t type1 = obj1->type;  # 获取对象1的类型
  hwloc_obj_type_t type2 = obj2->type;  # 获取对象2的类型
  int compare;  # 定义比较结果

  compare = hwloc_compare_types(type1, type2);  # 比较两个类型
  if (compare == HWLOC_TYPE_UNORDERED)  # 如果比较结果为无序
    return HWLOC_OBJ_DIFFERENT; /* we cannot do better */  # 返回无法更好的比较结果
  if (compare > 0)  # 如果比较结果大于0
    return HWLOC_OBJ_INCLUDED;  # 返回严格包含在内
  if (compare < 0)  # 如果比较结果小于0
    return HWLOC_OBJ_CONTAINS;  # 返回严格包含

  if (obj1->type == HWLOC_OBJ_GROUP  # 如果对象1的类型是组
      && (obj1->attr->group.kind != obj2->attr->group.kind  # 并且组的种类不同
      || obj1->attr->group.subkind != obj2->attr->group.subkind))  # 或者组的子种类不同
    return HWLOC_OBJ_DIFFERENT; /* we cannot do better */  # 返回无法更好的比较结果

  return HWLOC_OBJ_EQUAL;  # 返回相等
}

/*
 * How to compare objects based on cpusets.
 */
# 定义静态函数 hwloc_obj_cmp_sets，用于基于 cpusets 比较对象
static int
hwloc_obj_cmp_sets(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  hwloc_bitmap_t set1, set2;  # 定义两个 cpuset

  assert(!hwloc__obj_type_is_special(obj1->type));  # 断言对象1的类型不是特殊类型
  assert(!hwloc__obj_type_is_special(obj2->type));  # 断言对象2的类型不是特殊类型

  /* compare cpusets first */
  if (obj1->complete_cpuset && obj2->complete_cpuset) {  # 如果对象1和对象2都有完整的 cpuset
    set1 = obj1->complete_cpuset;  # 使用完整的 cpuset
    set2 = obj2->complete_cpuset;  # 使用完整的 cpuset
  } else {
    set1 = obj1->cpuset;  # 否则使用普通的 cpuset
    set2 = obj2->cpuset;  # 否则使用普通的 cpuset
  }
  if (set1 && set2 && !hwloc_bitmap_iszero(set1) && !hwloc_bitmap_iszero(set2))  # 如果两个 cpuset 都存在且不全为0
    return hwloc_bitmap_compare_inclusion(set1, set2);  # 比较两个 cpuset 的包含关系

  return HWLOC_OBJ_DIFFERENT;  # 返回无交集
}
/* Compare object cpusets based on complete_cpuset if defined (always correctly ordered),
 * or fallback to the main cpusets (only correctly ordered during early insert before disallowed bits are cleared).
 *
 * This is the sane way to compare object among a horizontal level.
 */
int
hwloc__object_cpusets_compare_first(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  // 如果 obj1 和 obj2 都有 complete_cpuset 属性，则比较它们的 complete_cpuset
  if (obj1->complete_cpuset && obj2->complete_cpuset)
    return hwloc_bitmap_compare_first(obj1->complete_cpuset, obj2->complete_cpuset);
  // 如果 obj1 和 obj2 都有 cpuset 属性，则比较它们的 cpuset
  else if (obj1->cpuset && obj2->cpuset)
    return hwloc_bitmap_compare_first(obj1->cpuset, obj2->cpuset);
  // 其他情况返回 0
  return 0;
}

/*
 * How to insert objects into the topology.
 *
 * Note: during detection, only the first_child and next_sibling pointers are
 * kept up to date.  Others are computed only once topology detection is
 * complete.
 */

/* merge new object attributes in old.
 * use old if defined, otherwise use new.
 */
static void
merge_insert_equal(hwloc_obj_t new, hwloc_obj_t old)
{
  // 如果 old 的 os_index 为未知，则使用 new 的 os_index
  if (old->os_index == HWLOC_UNKNOWN_INDEX)
    old->os_index = new->os_index;

  // 如果 new 有 infos，则将其合并到 old 的 infos 中
  if (new->infos_count) {
    /* FIXME: dedup */
    hwloc__move_infos(&old->infos, &old->infos_count,
              &new->infos, &new->infos_count);
  }

  // 如果 new 有 name 且 old 没有，则将 new 的 name 赋值给 old，并将 new 的 name 置为 NULL
  if (new->name && !old->name) {
    old->name = new->name;
    new->name = NULL;
  }
  // 如果 new 有 subtype 且 old 没有，则将 new 的 subtype 赋值给 old，并将 new 的 subtype 置为 NULL
  if (new->subtype && !old->subtype) {
    old->subtype = new->subtype;
    new->subtype = NULL;
  }

  /* Ignore userdata. It will be NULL before load().
   * It may be non-NULL if alloc+insert_group() after load().
   */

  // 根据 new 的类型进行不同的处理
  switch(new->type) {
  case HWLOC_OBJ_NUMANODE:
    if (new->attr->numanode.local_memory && !old->attr->numanode.local_memory) {
      /* 如果新节点有本地内存而旧节点没有，则使用新节点的内存 */
      old->attr->numanode.local_memory = new->attr->numanode.local_memory;
      // 释放旧节点的页类型数组
      free(old->attr->numanode.page_types);
      // 更新旧节点的页类型数组长度为新节点的长度
      old->attr->numanode.page_types_len = new->attr->numanode.page_types_len;
      // 将旧节点的页类型数组指向新节点的页类型数组
      old->attr->numanode.page_types = new->attr->numanode.page_types;
      // 将新节点的页类型数组指针置为空
      new->attr->numanode.page_types = NULL;
      // 将新节点的页类型数组长度置为0
      new->attr->numanode.page_types_len = 0;
    }
    /* old->attr->numanode.total_memory 将由 propagate_total_memory() 更新 */
    break;
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
    // 如果旧节点的缓存大小为0，则使用新节点的缓存大小
    if (!old->attr->cache.size)
      old->attr->cache.size = new->attr->cache.size;
    // 如果旧节点的缓存行大小为0，则使用新节点的缓存行大小
    if (!old->attr->cache.linesize)
      old->attr->cache.size = new->attr->cache.linesize;
    // 如果旧节点的缓存关联度为0，则使用新节点的缓存关联度
    if (!old->attr->cache.associativity)
      old->attr->cache.size = new->attr->cache.linesize;
    break;
  default:
    break;
  }
/* 返回合并的结果，如果没有合并则返回 NULL */
static __hwloc_inline hwloc_obj_t
hwloc__insert_try_merge_group(hwloc_topology_t topology, hwloc_obj_t old, hwloc_obj_t new)
{
  if (new->type == HWLOC_OBJ_GROUP && old->type == HWLOC_OBJ_GROUP) {
    /* 判断要保留哪个组 */
    if (new->attr->group.dont_merge) {
      if (old->attr->group.dont_merge)
        /* 没有一个想要被合并 */
        return NULL;

      /* 保留新的，它不想被合并 */
      hwloc_replace_linked_object(old, new);
      topology->modified = 1;
      return new;

    } else {
      if (old->attr->group.dont_merge)
        /* 保留旧的，它不想被合并 */
        return old;

      /* 比较子类型以决定要保留哪个组 */
      if (new->attr->group.kind < old->attr->group.kind) {
        /* 保留较小的类型 */
        hwloc_replace_linked_object(old, new);
        topology->modified = 1;
      }
      return old;
    }
  }

  if (new->type == HWLOC_OBJ_GROUP && !new->attr->group.dont_merge) {

    if (old->type == HWLOC_OBJ_PU && new->attr->group.kind == HWLOC_GROUP_KIND_MEMORY)
      /* 不要将 Memory 组与 PU 合并，我们不想将 Memory 附加到 PU 下 */
      return NULL;

    /* 立即移除组。正常的忽略代码路径不会告诉我们组是否已被移除，而某些调用者需要知道（至少是 hwloc_topology_insert_group()） */
    return old;

  } else if (old->type == HWLOC_OBJ_GROUP && !old->attr->group.dont_merge) {

    if (new->type == HWLOC_OBJ_PU && old->attr->group.kind == HWLOC_GROUP_KIND_MEMORY)
      /* 不要将 Memory 组与 PU 合并，我们不想将 Memory 附加到 PU 下 */
      return NULL;

    /* 用新对象内容替换组，并让调用者释放新对象 */
    hwloc_replace_linked_object(old, new);
    topology->modified = 1;
    return old;

  } else {
    /* 无法合并 */
    return NULL;
  }
}
/*
 * 主要的插入例程，仅用于 CPU 端对象（普通类型），只使用 cpuset 或 complete_cpuset。
 *
 * 尝试将 OBJ 插入 CUR，如果需要则递归。
 * 如果成功插入，则返回对象，
 * 如果合并了剩余对象，则返回合并后的对象，
 * 如果插入失败则返回 NULL。
 */
static struct hwloc_obj *
hwloc___insert_object_by_cpuset(struct hwloc_topology *topology, hwloc_obj_t cur, hwloc_obj_t obj,
                    const char *reason)
{
  hwloc_obj_t child, next_child = NULL, tmp;
  /* 这些将始终指向它们的下一个最后子对象的指针。 */
  hwloc_obj_t *cur_children = &cur->first_child;
  hwloc_obj_t *obj_children = &obj->first_child;
  /* OBJ 应该放置的指针位置 */
  hwloc_obj_t *putp = NULL; /* 尚未找到 OBJ 的位置 */

  assert(!hwloc__obj_type_is_memory(obj->type));

  /* 迭代并进行预取以完全安全地防止子对象被移除。
   * 列表已经按 cpuset 排序，并且兄弟节点之间没有交集。
   */
  for (child = cur->first_child, child ? next_child = child->next_sibling : NULL;
       child;
       child = next_child, child ? next_child = child->next_sibling : NULL) {

    int res = hwloc_obj_cmp_sets(obj, child);
    int setres = res;

    if (res == HWLOC_OBJ_EQUAL) {
      hwloc_obj_t merged = hwloc__insert_try_merge_group(topology, child, obj);
      if (merged)
    return merged;
      /* 否则比较实际类型以决定是否包含 */
      res = hwloc_type_cmp(obj, child);
    }

    switch (res) {
      case HWLOC_OBJ_EQUAL:
    /* 两个具有相同类型的对象。
     * 组在上面已经处理过了。
     */
    merge_insert_equal(obj, child);
    /* 已经存在，无需插入。 */
    return child;

      case HWLOC_OBJ_INCLUDED:
    /* OBJ 严格包含在 CUR 的某个子对象中，继续深入。 */
    # 调用 hwloc___insert_object_by_cpuset 函数，将 obj 插入到 topology 中
    return hwloc___insert_object_by_cpuset(topology, child, obj, reason);

      # 处理 HWLOC_OBJ_INTERSECTS 情况
      case HWLOC_OBJ_INTERSECTS:
        # 报告插入错误，obj 和 child 有交集但不包含关系
        report_insert_error(obj, child, "intersection without inclusion", reason);
    # 转到 putback 标签处
    goto putback;

      # 处理 HWLOC_OBJ_DIFFERENT 情况
      case HWLOC_OBJ_DIFFERENT:
        # 如果 OBJ 应该在 CHILD 之前成为 CUR 的子对象，标记其位置
    if (!putp && hwloc__object_cpusets_compare_first(obj, child) < 0)
      # 暂时不插入，因为后面可能会有交集错误
      putp = cur_children;
    # 移动 cur_children 指针
    cur_children = &child->next_sibling;
    break;

      # 处理 HWLOC_OBJ_CONTAINS 情况
      case HWLOC_OBJ_CONTAINS:
    # OBJ 包含 CHILD，从 CUR 中移除 CHILD
    *cur_children = child->next_sibling;
    child->next_sibling = NULL;
    # 将 CHILD 放入 OBJ
    *obj_children = child;
    obj_children = &child->next_sibling;
    child->parent = obj;
    if (setres == HWLOC_OBJ_EQUAL) {
      obj->memory_first_child = child->memory_first_child;
      child->memory_first_child = NULL;
      for(tmp=obj->memory_first_child; tmp; tmp = tmp->next_sibling)
        tmp->parent = obj;
    }
    break;
    }
  }
  # cur/obj_children 指向最后的 CUR/OBJ 子对象的 next_sibling 指针，必须为 NULL
  assert(!*obj_children);
  assert(!*cur_children);

  # 将 OBJ 放入其应该在的位置，或者放在 CUR 的子对象的最后
  if (!putp)
    putp = cur_children;
  obj->next_sibling = *putp;
  *putp = obj;
  obj->parent = cur;

  # 标记 topology 已修改
  topology->modified = 1;
  return obj;

 putback:
  # OBJ 无法插入
  # 将 OBJ 的子对象放回 CUR 中，并返回错误
  if (putp)
    cur_children = putp; # 不需要尝试在 OBJ 应该在的位置之前插入
  else
    cur_children = &cur->first_child; # 从头开始
  # 可以按顺序插入，但中间可能有空隙
  while ((child = obj->first_child) != NULL) {
    # 从 OBJ 中移除
    obj->first_child = child->next_sibling;
    /* 在CUR中查找子节点的位置，并重新插入 */
    while (*cur_children && hwloc__object_cpusets_compare_first(*cur_children, child) < 0)
      cur_children = &(*cur_children)->next_sibling;
    // 将子节点插入到CUR的子节点链表中
    child->next_sibling = *cur_children;
    *cur_children = child;
    // 设置子节点的父节点为CUR
    child->parent = cur;
  }
  // 返回空值
  return NULL;
/* 根据内存 cpuset 查找覆盖该 cpuset 的对象，返回找到的对象 */
static struct hwloc_obj *
hwloc__find_obj_covering_memory_cpuset(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_bitmap_t cpuset)
{
  // 根据 cpuset 查找覆盖该 cpuset 的子对象
  hwloc_obj_t child = hwloc_get_child_covering_cpuset(topology, cpuset, parent);
  // 如果没有找到子对象，则返回父对象
  if (!child)
    return parent;
  // 如果找到子对象，并且子对象的 cpuset 与给定 cpuset 完全相等，则返回子对象
  if (child && hwloc_bitmap_isequal(child->cpuset, cpuset))
    return child;
  // 递归查找覆盖该 cpuset 的对象
  return hwloc__find_obj_covering_memory_cpuset(topology, child, cpuset);
}

/* 查找插入内存的父对象 */
static struct hwloc_obj *
hwloc__find_insert_memory_parent(struct hwloc_topology *topology, hwloc_obj_t obj,
                                 const char *reason)
{
  hwloc_obj_t parent, group, result;

  // 如果对象的 cpuset 为空，则将其放入根节点下的专用组中
  if (hwloc_bitmap_iszero(obj->cpuset)) {
    parent = topology->levels[0][0];

  } else {
    // 查找覆盖该 cpuset 的最高级别对象
    parent = hwloc__find_obj_covering_memory_cpuset(topology, topology->levels[0][0], obj->cpuset);
    // 如果没有找到覆盖该 cpuset 的对象，则回退到根节点
    if (!parent) {
      parent = hwloc_get_root_obj(topology);
    }

    // 如果父对象是 PU，则向上查找父对象
    if (parent->type == HWLOC_OBJ_PU) {
      parent = parent->parent;
      assert(parent);
    }

    // 如果根节点的 cpuset 已经更新，则确定组对象是否与根节点保持一致
    if (parent != topology->levels[0][0] && hwloc_bitmap_isequal(parent->cpuset, obj->cpuset))
      return parent;
  }

  // 如果不允许保留组对象类型，则不创建中间组对象
  if (!hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP))
    /* even if parent isn't perfect, we don't want an intermediate group */
    return parent;

  /* 需要插入一个中间组来附加 NUMA 节点 */
  group = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
  if (!group)
    /* 创建组失败，回退到更大的父对象 */
    return parent;

  group->attr->group.kind = HWLOC_GROUP_KIND_MEMORY;
  group->cpuset = hwloc_bitmap_dup(obj->cpuset);
  group->complete_cpuset = hwloc_bitmap_dup(obj->complete_cpuset);
  /* 我们也可以复制 nodesets，但是 hwloc__insert_object_by_cpuset() 实际上不需要它。
   * 而且它可能会阻止未来的调用重用该组来处理其他 NUMA 节点。
   */
  if (!group->cpuset != !obj->cpuset
      || !group->complete_cpuset != !obj->complete_cpuset) {
    /* 创建组失败，回退到更大的父对象 */
    hwloc_free_unlinked_object(group);
    return parent;
  }

  result = hwloc__insert_object_by_cpuset(topology, parent, group, reason);
  if (!result) {
    /* 插入失败，回退到更大的父对象 */
    return parent;
  }

  assert(result == group);
  return group;
}

/* 仅适用于具有单个节点集中的 MEMCACHE 和 NUMAnode */
static hwloc_obj_t
hwloc___attach_memory_object_by_nodeset(struct hwloc_topology *topology, hwloc_obj_t parent,
                    hwloc_obj_t obj, const char *reason)
{
  hwloc_obj_t *curp = &parent->memory_first_child;  // 指向父对象的第一个内存子对象的指针
  unsigned first = hwloc_bitmap_first(obj->nodeset);  // 获取节点集中的第一个位的索引

  while (*curp) {
    hwloc_obj_t cur = *curp;  // 获取当前内存子对象
    unsigned curfirst = hwloc_bitmap_first(cur->nodeset);  // 获取当前内存子对象节点集中的第一个位的索引

    if (first < curfirst) {
      /* 在当前对象之前插入 */
      obj->next_sibling = cur;  // 将当前对象的下一个兄弟设置为当前对象
      *curp = obj;  // 将当前指针指向新对象
      obj->memory_first_child = NULL;  // 将新对象的第一个内存子对象设置为NULL
      obj->parent = parent;  // 将新对象的父对象设置为当前对象
      topology->modified = 1;  // 设置拓扑结构已修改标志
      return obj;  // 返回新对象
    }

    if (first == curfirst) {
      /* 相同的节点集 */
      if (obj->type == HWLOC_OBJ_NUMANODE) {
    if (cur->type == HWLOC_OBJ_NUMANODE) {
      /* 相同的 NUMA 节点？忽略新的 */
          report_insert_error(obj, cur, "NUMAnodes with identical nodesets", reason);  // 报告错误
      return NULL;  // 返回空
    }
    assert(cur->type == HWLOC_OBJ_MEMCACHE);
    /* 在现有 memcache 下面插入新的 NUMA 节点 */
    return hwloc___attach_memory_object_by_nodeset(topology, cur, obj, reason);  // 递归调用自身

      } else {
    assert(obj->type == HWLOC_OBJ_MEMCACHE);
    if (cur->type == HWLOC_OBJ_MEMCACHE) {
      if (cur->attr->cache.depth == obj->attr->cache.depth)
        /* 具有相同节点集和深度的 memcache，忽略新的 */
        return NULL;  // 返回空
      if (cur->attr->cache.depth > obj->attr->cache.depth)
        /* 具有更高缓存深度的 memcache 实际上在层次结构中更高
         * （深度从 NUMA 节点开始）。
         * 在现有 memcache 下面插入新的 memcache
         */
        return hwloc___attach_memory_object_by_nodeset(topology, cur, obj, reason);  // 递归调用自身
    }
    /* 在现有 memcache 或 numa 节点上面插入 memcache */
    obj->next_sibling = cur->next_sibling;  // 将新对象的下一个兄弟设置为当前对象的下一个兄弟
    cur->next_sibling = NULL;  // 将当前对象的下一个兄弟设置为NULL
    obj->memory_first_child = cur;  // 将新对象的第一个内存子对象设置为当前对象
    cur->parent = obj;  // 将当前对象的父对象设置为新对象
    # 将指针 curp 指向 obj
    *curp = obj;
    # 设置 obj 的父节点为 parent
    obj->parent = parent;
    # 设置 topology 的 modified 属性为 1
    topology->modified = 1;
    # 返回 obj
    return obj;
      }
    }

    # 将指针 curp 指向 cur 的下一个兄弟节点
    curp = &cur->next_sibling;
  }

  # 在列表末尾追加节点
  obj->next_sibling = NULL;
  # 将指针 curp 指向 obj
  *curp = obj;
  # 将 obj 的 memory_first_child 属性设置为 NULL
  obj->memory_first_child = NULL;
  # 设置 obj 的父节点为 parent
  obj->parent = parent;
  # 设置 topology 的 modified 属性为 1
  topology->modified = 1;
  # 返回 obj
  return obj;
/* Attach the given memory object below the given normal parent.
 *
 * Only the nodeset is used to find the location inside memory children below parent.
 *
 * Nodeset inclusion inside the given memory hierarchy is guaranteed by this function,
 * but nodesets are not propagated to CPU-side parent yet. It will be done by
 * propagate_nodeset() later.
 */
// 在给定的普通父对象下附加给定的内存对象。
// 只使用节点集来找到父对象下面内存子对象的位置。
// 此函数保证节点集包含在给定的内存层次结构中，但节点集尚未传播到 CPU 端的父对象。这将由 propagate_nodeset() 在后面完成。

struct hwloc_obj *
hwloc__attach_memory_object(struct hwloc_topology *topology, hwloc_obj_t parent,
                hwloc_obj_t obj, const char *reason)
{
  hwloc_obj_t result;

  assert(parent);
  assert(hwloc__obj_type_is_normal(parent->type));

  /* Check the nodeset */
  // 检查节点集
  if (!obj->nodeset || hwloc_bitmap_iszero(obj->nodeset))
    return NULL;
  // 如果节点集为空或全零，则返回空

  /* Initialize or check the complete nodeset */
  // 初始化或检查完整的节点集
  if (!obj->complete_nodeset) {
    obj->complete_nodeset = hwloc_bitmap_dup(obj->nodeset);
  } else if (!hwloc_bitmap_isincluded(obj->nodeset, obj->complete_nodeset)) {
    return NULL;
  }
  // 如果完整节点集不存在，则复制节点集；否则，检查节点集是否包含在完整节点集中

  /* Neither ACPI nor Linux support multinode mscache */
  // ACPI 和 Linux 都不支持多节点 mscache
  assert(hwloc_bitmap_weight(obj->nodeset) == 1);
  // 断言节点集的权重为 1

#if 0
  /* TODO: enable this instead of hack in fixup_sets once NUMA nodes are inserted late */
  // TODO: 一旦 NUMA 节点被晚插入，启用这个而不是在 fixup_sets 中进行 hack
  /* copy the parent cpuset in case it's larger than expected.
   * we could also keep the cpuset smaller than the parent and say that a normal-parent
   * can have multiple memory children with smaller cpusets.
   * However, the user decided the ignore Groups, so hierarchy/locality loss is expected.
   */
  // 如果父对象的 cpuset 大于预期，则复制父对象的 cpuset。
  // 我们也可以保持 cpuset 小于父对象，并说普通父对象可以有多个具有较小 cpuset 的内存子对象。
  // 但是，用户决定忽略组，因此预计会丢失层次结构/局部性。
  hwloc_bitmap_copy(obj->cpuset, parent->cpuset);
  hwloc_bitmap_copy(obj->complete_cpuset, parent->complete_cpuset);
#endif

  result = hwloc___attach_memory_object_by_nodeset(topology, parent, obj, reason);
  if (result == obj) {
    /* Add the bit to the top sets, and to the parent CPU-side object */
    // 将位添加到顶层集合，并添加到父 CPU 端对象
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      hwloc_bitmap_set(topology->levels[0][0]->nodeset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_nodeset, obj->os_index);
    }
  }
  if (result != obj) {
    /* 如果插入失败或者被合并了，释放原始对象 */
    hwloc_free_unlinked_object(obj);
  }
  // 返回结果
  return result;
/* 插入例程，允许更改错误报告回调 */
struct hwloc_obj *
hwloc__insert_object_by_cpuset(struct hwloc_topology *topology, hwloc_obj_t root,
                   hwloc_obj_t obj, const char *reason)
{
  struct hwloc_obj *result;

#ifdef HWLOC_DEBUG
  assert(!hwloc__obj_type_is_special(obj->type));

  /* 我们至少需要一个非空集合（正常或完整，cpuset或nodeset） */
  assert(obj->cpuset || obj->complete_cpuset || obj->nodeset || obj->complete_nodeset);
  /* 我们支持它们全部为空的情况。
   * 当hwloc__find_insert_memory_parent()为没有CPU的NUMA节点插入Group时可能会发生。
   */
#endif

  if (hwloc__obj_type_is_memory(obj->type)) {
    if (!root) {
      root = hwloc__find_insert_memory_parent(topology, obj, reason);
      if (!root) {
    hwloc_free_unlinked_object(obj);
    return NULL;
      }
    }
    return hwloc__attach_memory_object(topology, root, obj, reason);
  }

  if (!root)
    /* 从顶部开始。 */
    root = topology->levels[0][0];

  result = hwloc___insert_object_by_cpuset(topology, root, obj, reason);
  if (result && result->type == HWLOC_OBJ_PU) {
      /* 将位添加到顶部集合 */
      if (hwloc_bitmap_isset(result->cpuset, result->os_index))
    hwloc_bitmap_set(topology->levels[0][0]->cpuset, result->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_cpuset, result->os_index);
  }
  if (result != obj) {
    /* 要么插入失败，要么被合并，释放原始对象 */
    hwloc_free_unlinked_object(obj);
  }
  return result;
}

/* 默认的插入例程在出现错误时发出警告。
 * 大多数后端使用它 */
void
hwloc_insert_object_by_parent(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_obj_t obj)
{
  hwloc_obj_t *current;

  if (obj->type == HWLOC_OBJ_MISC) {
    /* 追加到Misc列表的末尾 */
    for (current = &parent->misc_first_child; *current; current = &(*current)->next_sibling);
  } else if (hwloc__obj_type_is_io(obj->type)) {
    /* 如果对象类型是 I/O，则将其追加到 I/O 列表的末尾 */
    for (current = &parent->io_first_child; *current; current = &(*current)->next_sibling);
  } else if (hwloc__obj_type_is_memory(obj->type)) {
    /* 如果对象类型是内存，则将其追加到内存列表的末尾 */
    for (current = &parent->memory_first_child; *current; current = &(*current)->next_sibling);
    /* 将位添加到顶层集合 */
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      if (hwloc_bitmap_isset(obj->nodeset, obj->os_index))
    hwloc_bitmap_set(topology->levels[0][0]->nodeset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_nodeset, obj->os_index);
    }
  } else {
    /* 将对象追加到列表的末尾。
     * 调用者负责按正确的 cpuset 顺序插入子对象，不会相互交叉。
     * 复制不需要检查顺序，因为源拓扑应该已经是正确的。
     * XML 会重新排序，如果需要的话，并且在交叉的兄弟节点上失败。
     * 其他调用者只是插入随机对象，比如 I/O 或 Misc，那里没有 cpuset 问题。
     */
    for (current = &parent->first_child; *current; current = &(*current)->next_sibling);
    /* 将位添加到顶层集合 */
    if (obj->type == HWLOC_OBJ_PU) {
      if (hwloc_bitmap_isset(obj->cpuset, obj->os_index))
    hwloc_bitmap_set(topology->levels[0][0]->cpuset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_cpuset, obj->os_index);
    }
  }

  *current = obj;
  obj->parent = parent;
  obj->next_sibling = NULL;
  topology->modified = 1;
}

# 分配并设置一个新的硬件对象
hwloc_obj_t
hwloc_alloc_setup_object(hwloc_topology_t topology,
             hwloc_obj_type_t type, unsigned os_index)
{
  # 使用 topology 的 tma 分配内存来创建一个新的硬件对象
  struct hwloc_obj *obj = hwloc_tma_malloc(topology->tma, sizeof(*obj));
  if (!obj)
    return NULL;
  # 将新对象的内存清零
  memset(obj, 0, sizeof(*obj));
  # 设置对象的类型、操作系统索引和全局索引
  obj->type = type;
  obj->os_index = os_index;
  obj->gp_index = topology->next_gp_index++;
  # 使用 topology 的 tma 分配内存来创建对象的属性
  obj->attr = hwloc_tma_malloc(topology->tma, sizeof(*obj->attr));
  if (!obj->attr) {
    assert(!topology->tma || !topology->tma->dontfree); # 这个 tma 不能失败分配
    free(obj);
    return NULL;
  }
  # 将对象的属性内存清零
  memset(obj->attr, 0, sizeof(*obj->attr));
  # 不在这里分配 cpuset，让调用者来做
  return obj;
}

# 分配并设置一个新的组对象
hwloc_obj_t
hwloc_topology_alloc_group_object(struct hwloc_topology *topology)
{
  if (!topology->is_loaded) {
    # 这实际上可能会起作用，参见下面的 insert()
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }
  return hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
}

# 传播对称子树
static void hwloc_propagate_symmetric_subtree(hwloc_topology_t topology, hwloc_obj_t root);
# 传播总内存
static void propagate_total_memory(hwloc_obj_t obj);
# 设置组深度
static void hwloc_set_group_depth(hwloc_topology_t topology);
# 连接子对象
static void hwloc_connect_children(hwloc_obj_t parent);
# 连接层级
static int hwloc_connect_levels(hwloc_topology_t topology);
# 连接特殊层级
static int hwloc_connect_special_levels(hwloc_topology_t topology);

# 插入组对象到拓扑结构中
hwloc_obj_t
hwloc_topology_insert_group_object(struct hwloc_topology *topology, hwloc_obj_t obj)
{
  hwloc_obj_t res, root;
  int cmp;

  if (!topology->is_loaded) {
    # 这实际上可能会起作用，我们只需要在下面禁用 connect_children/levels
    hwloc_free_unlinked_object(obj);
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }

  if (topology->type_filter[HWLOC_OBJ_GROUP] == HWLOC_TYPE_FILTER_KEEP_NONE) {
    hwloc_free_unlinked_object(obj);
  # 设置错误码为无效参数
  errno = EINVAL;
  # 返回空指针
  return NULL;
  }

  # 获取拓扑结构的根对象
  root = hwloc_get_root_obj(topology);
  # 如果对象有 cpuset，则对其进行与操作，与根对象的 cpuset 进行交集
  if (obj->cpuset)
    hwloc_bitmap_and(obj->cpuset, obj->cpuset, root->cpuset);
  # 如果对象有 complete_cpuset，则对其进行与操作，与根对象的 complete_cpuset 进行交集
  if (obj->complete_cpuset)
    hwloc_bitmap_and(obj->complete_cpuset, obj->complete_cpuset, root->complete_cpuset);
  # 如果对象有 nodeset，则对其进行与操作，与根对象的 nodeset 进行交集
  if (obj->nodeset)
    hwloc_bitmap_and(obj->nodeset, obj->nodeset, root->nodeset);
  # 如果对象有 complete_nodeset，则对其进行与操作，与根对象的 complete_nodeset 进行交集
  if (obj->complete_nodeset)
    hwloc_bitmap_and(obj->complete_nodeset, obj->complete_nodeset, root->complete_nodeset);

  # 如果对象没有 cpuset 或者其 cpuset 为空
  if ((!obj->cpuset || hwloc_bitmap_iszero(obj->cpuset))
      && (!obj->complete_cpuset || hwloc_bitmap_iszero(obj->complete_cpuset))) {
    # 如果没有 nodeset 或者其 nodeset 为空
    if ((!obj->nodeset || hwloc_bitmap_iszero(obj->nodeset))
    && (!obj->complete_nodeset || hwloc_bitmap_iszero(obj->complete_nodeset))) {
      # 释放对象并设置错误码为无效参数，返回空指针
      hwloc_free_unlinked_object(obj);
      errno = EINVAL;
      return NULL;
    }

    # 如果对象没有 cpuset
    if (!obj->cpuset) {
      # 分配一个新的 cpuset
      obj->cpuset = hwloc_bitmap_alloc();
      # 如果分配失败，释放对象并返回空指针
      if (!obj->cpuset) {
    hwloc_free_unlinked_object(obj);
    return NULL;
      }
    }

    # 初始化 numa 为 NULL
    numa = NULL;
    # 遍历 NUMA 节点
    while ((numa = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_NUMANODE, numa)) != NULL)
      # 如果节点在 nodeset 中，则将其 cpuset 与对象的 cpuset进行或操作
    hwloc_bitmap_or(obj->cpuset, obj->cpuset, numa->cpuset);
  }
  # FIXME 插入 nodeset 以便即使没有 CPU 也能分组 NUMA？

  # 比较对象和根对象的 cpuset 和 nodeset
  cmp = hwloc_obj_cmp_sets(obj, root);
  # 如果对象包含在根对象中
  if (cmp == HWLOC_OBJ_INCLUDED) {
    # 通过 cpuset 插入对象到拓扑结构中
    res = hwloc__insert_object_by_cpuset(topology, NULL, obj, NULL /* do not show errors on stdout */);
  } else {
    # 否则，将根对象作为结果
    res = root;
  }

  # 如果结果为空，则返回空指针
  if (!res)
    return NULL;

  # 如果结果不等于对象且结果的类型不是组
  if (res != obj && res->type != HWLOC_OBJ_GROUP)
    # 合并了，不是一个组，没有需要更新的内容
    # 返回结果
    return res;

  # 如果 res == obj 表示对象已插入。
  # 需要重新连接级别，填充所有的 CPU/节点集，计算总内存，组深度等。
  #
  # 如果 res != obj 通常表示我们的新组已合并到现有对象中，无需重新计算任何内容。
  # 但是，如果与现有组合并，取决于它们的类型，obj 的内容可能会覆盖旧组的内容。
  # 这需要重新连接级别，填充集合，重新计算总内存等。

  # 适当插入
  # 为 res 添加子集
  hwloc_obj_add_children_sets(res);
  # 如果重新连接拓扑失败，则返回空
  if (hwloc_topology_reconnect(topology, 0) < 0)
    return NULL;
  # 传播对称子树
  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);
  # 设置组深度
  hwloc_set_group_depth(topology);
#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG，则执行以下代码块
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 检查拓扑结构的完整性
    hwloc_topology_check(topology);

  // 返回结果
  return res;
}

// 在拓扑结构中插入一个杂项对象
hwloc_obj_t
hwloc_topology_insert_misc_object(struct hwloc_topology *topology, hwloc_obj_t parent, const char *name)
{
  hwloc_obj_t obj;

  // 如果杂项对象的类型过滤器设置为不保留，则返回错误
  if (topology->type_filter[HWLOC_OBJ_MISC] == HWLOC_TYPE_FILTER_KEEP_NONE) {
    errno = EINVAL;
    return NULL;
  }

  // 如果拓扑结构未加载，则返回错误
  if (!topology->is_loaded) {
    errno = EINVAL;
    return NULL;
  }
  // 如果采用了共享内存地址，则返回错误
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }

  // 分配并设置杂项对象
  obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_MISC, HWLOC_UNKNOWN_INDEX);
  if (name)
    obj->name = strdup(name);

  // 将杂项对象插入到父对象中
  hwloc_insert_object_by_parent(topology, parent, obj);

  /* FIXME: only connect misc parent children and misc level,
   * but this API is likely not performance critical anyway
   */
  // 重新连接拓扑结构
  hwloc_topology_reconnect(topology, 0);

#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG，则执行以下代码块
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 检查拓扑结构的完整性
    hwloc_topology_check(topology);

  // 返回插入的对象
  return obj;
}

/* assuming set is included in the topology complete_cpuset
 * and all objects have a proper complete_cpuset,
 * return the best one containing set.
 * if some object are equivalent (same complete_cpuset), return the highest one.
 */
// 获取包含完整 CPU 集合的最高对象
static hwloc_obj_t
hwloc_get_highest_obj_covering_complete_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  hwloc_obj_t current = hwloc_get_root_obj(topology);
  hwloc_obj_t child;

  // 如果当前对象的完整 CPU 集合与给定集合相等，则返回当前对象
  if (hwloc_bitmap_isequal(set, current->complete_cpuset))
    /* root cpuset is exactly what we want, no need to look at children, we want the highest */
    return current;

 recurse:
  // 查找正确的子对象
  for_each_child(child, current) {
    if (hwloc_bitmap_isequal(set, child->complete_cpuset))
      /* child puset is exactly what we want, no need to look at children, we want the highest */
      return child;
    if (!hwloc_bitmap_iszero(child->complete_cpuset) && hwloc_bitmap_isincluded(set, child->complete_cpuset))
      break;
  }

  if (child) {
    # 将当前节点赋值给child
    current = child;
    # 跳转到递归标签处
    goto recurse;
  }

  # 如果没有更好的子节点，则返回当前节点
  /* no better child */
  return current;
}

hwloc_obj_t
hwloc_find_insert_io_parent_by_complete_cpuset(struct hwloc_topology *topology, hwloc_cpuset_t cpuset)
{
  hwloc_obj_t group_obj, largeparent, parent;

  /* 限制到现有的完整 cpuset，以避免后续出现错误 */
  hwloc_bitmap_and(cpuset, cpuset, hwloc_topology_get_complete_cpuset(topology));
  if (hwloc_bitmap_iszero(cpuset))
    /* 剩余的 cpuset 为空，无效 */
    return NULL;

  largeparent = hwloc_get_highest_obj_covering_complete_cpuset(topology, cpuset);
  if (hwloc_bitmap_isequal(largeparent->complete_cpuset, cpuset)
      || !hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP))
    /* 找到一个有效的对象（正常情况） */
    return largeparent;

  /* 我们需要插入一个中间的组 */
  group_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
  if (!group_obj)
    /* 无法插入确切的组，回退到 largeparent */
    return largeparent;

  group_obj->complete_cpuset = hwloc_bitmap_dup(cpuset);
  hwloc_bitmap_and(cpuset, cpuset, hwloc_topology_get_topology_cpuset(topology));
  group_obj->cpuset = hwloc_bitmap_dup(cpuset);
  group_obj->attr->group.kind = HWLOC_GROUP_KIND_IO;
  parent = hwloc__insert_object_by_cpuset(topology, largeparent, group_obj, "topology:io_parent");
  if (!parent)
    /* 无法插入组，可能是冲突的 cpuset */
    return largeparent;

  /* 组无法合并，否则我们之前就会得到正确的 largeparent */
  assert(parent == group_obj);

  /* 组插入而没有被合并，一切正常，设置其集合 */
  hwloc_obj_add_children_sets(group_obj);

  return parent;
}

static int hwloc_memory_page_type_compare(const void *_a, const void *_b)
{
  const struct hwloc_memory_page_type_s *a = _a;
  const struct hwloc_memory_page_type_s *b = _b;
  /* 将 0 视为更大，以便将大小为 0 的页面类型放在最后 */
  if (!b->size)
    # 返回 -1，表示出现错误
    return -1;
  # 不要将 a-b 强制转换为 int 类型，因为 a 和 b 是无符号长长整型
  /* don't cast a-b in int since those are ullongs */
  # 如果 a 和 b 的大小相等，则返回 0
  if (b->size == a->size)
    return 0;
  # 如果 a 的大小小于 b 的大小，则返回 -1；否则返回 1
  return a->size < b->size ? -1 : 1;
}

/* 传播内存计数 */
static void
propagate_total_memory(hwloc_obj_t obj)
{
  hwloc_obj_t child;
  unsigned i;

  /* 在计算本地和子节点内存之前重置总内存 */
  obj->total_memory = 0;

  /* 向上传播内存 */
  for_each_child(child, obj) {
    propagate_total_memory(child);
    obj->total_memory += child->total_memory;
  }
  for_each_memory_child(child, obj) {
    propagate_total_memory(child);
    obj->total_memory += child->total_memory;
  }
  /* I/O 或 Misc 下没有内存 */

  if (obj->type == HWLOC_OBJ_NUMANODE) {
    obj->total_memory += obj->attr->numanode.local_memory;

    if (obj->attr->numanode.page_types_len) {
      /* 顺便对 page_type 数组进行排序。
       * 不能在插入时进行排序，因为一些后端（例如 XML）在插入对象后添加 page_types。
       */
      qsort(obj->attr->numanode.page_types, obj->attr->numanode.page_types_len, sizeof(*obj->attr->numanode.page_types), hwloc_memory_page_type_compare);
      /* 忽略大小为 0 的 page_types，它们在最后 */
      for(i=obj->attr->numanode.page_types_len; i>=1; i--)
        if (obj->attr->numanode.page_types[i-1].size)
          break;
      obj->attr->numanode.page_types_len = i;
    }
  }
}

/* 现在根集准备好了，将它们传播给子节点
 * 通过分配缺失的集合和限制现有的集合。
 */
static void
fixup_sets(hwloc_obj_t obj)
{
  int in_memory_list;
  hwloc_obj_t child;

  child = obj->first_child;
  in_memory_list = 0;
  /* 首先迭代正常子节点，稍后我们会回来处理内存子节点 */

  /* 注意：如果内存对象是后来插入的，我们应该在插入时更新它们的 cpuset 和 complete_cpuset，而不是在这里 */
 iterate:
  while (child) {
    /* 我们的 cpuset 必须包含在父节点的 cpuset 中 */
    hwloc_bitmap_and(child->cpuset, child->cpuset, obj->cpuset);
    hwloc_bitmap_and(child->nodeset, child->nodeset, obj->nodeset);
    /* our complete_cpuset must be included in our parent's one, but can be larger than our cpuset */
    // 如果子对象的 complete_cpuset 存在，则取子对象的 complete_cpuset 与父对象的 complete_cpuset 的交集
    if (child->complete_cpuset) {
      hwloc_bitmap_and(child->complete_cpuset, child->complete_cpuset, obj->complete_cpuset);
    } else {
      // 如果子对象的 complete_cpuset 不存在，则将子对象的 cpuset 复制给 complete_cpuset
      child->complete_cpuset = hwloc_bitmap_dup(child->cpuset);
    }
    // 如果子对象的 complete_nodeset 存在，则取子对象的 complete_nodeset 与父对象的 complete_nodeset 的交集
    if (child->complete_nodeset) {
      hwloc_bitmap_and(child->complete_nodeset, child->complete_nodeset, obj->complete_nodeset);
    } else {
      // 如果子对象的 complete_nodeset 不存在，则将子对象的 nodeset 复制给 complete_nodeset
      child->complete_nodeset = hwloc_bitmap_dup(child->nodeset);
    }

    // 如果子对象是内存类型，则更新内存子对象的 cpuset 和 complete_cpuset
    if (hwloc_obj_type_is_memory(child->type)) {
      hwloc_bitmap_copy(child->cpuset, obj->cpuset);
      hwloc_bitmap_copy(child->complete_cpuset, obj->complete_cpuset);
    }

    // 修正子对象的集合
    fixup_sets(child);
    // 移动到下一个兄弟节点
    child = child->next_sibling;
  }

  /* switch to memory children list if any */
  // 如果不在内存子对象列表中，并且父对象有内存子对象，则切换到内存子对象列表
  if (!in_memory_list && obj->memory_first_child) {
    child = obj->memory_first_child;
    in_memory_list = 1;
    // 跳转到迭代标签处
    goto iterate;
  }

  /* No sets in I/O or Misc */
  // I/O 或 Misc 类型没有集合
}

/* 通过对其子对象进行OR运算来设置cpusets/nodesets对象。 */
int
hwloc_obj_add_other_obj_sets(hwloc_obj_t dst, hwloc_obj_t src)
{
#define ADD_OTHER_OBJ_SET(_dst, _src, _set)            \
  if ((_src)->_set) {                        \
    if (!(_dst)->_set)                        \
      (_dst)->_set = hwloc_bitmap_alloc();            \
    hwloc_bitmap_or((_dst)->_set, (_dst)->_set, (_src)->_set);    \
  }
  ADD_OTHER_OBJ_SET(dst, src, cpuset);
  ADD_OTHER_OBJ_SET(dst, src, complete_cpuset);
  ADD_OTHER_OBJ_SET(dst, src, nodeset);
  ADD_OTHER_OBJ_SET(dst, src, complete_nodeset);
  return 0;
}

int
hwloc_obj_add_children_sets(hwloc_obj_t obj)
{
  hwloc_obj_t child;
  for_each_child(child, obj) {
    hwloc_obj_add_other_obj_sets(obj, child);
  }
  /* 不需要查看Misc子对象，因为它们不包含PU。 */
  return 0;
}

/* CPU对象是通过cpusets插入的，我们知道它们的cpusets已经正确包含。
 * 我们只需要fixup_sets()来确保它们不会太宽。
 *
 * 在每个内存层次结构中，nodeset也是一致的。
 * 但是它们必须传播到它们的CPU端父对象。
 *
 * 内存对象的nodeset由其下面的NUMA节点组成。
 * 普通对象的nodeset由其任何子对象或父对象附加的NUMA节点组成。
 */
static void
propagate_nodeset(hwloc_obj_t obj)
{
  hwloc_obj_t child;

  /* 从父对象的nodeset开始。
   * 在根节点处为空，随着我们向下递归，它将填充该树枝中的本地节点。
   */
  if (!obj->nodeset)
    obj->nodeset = hwloc_bitmap_alloc();
  if (obj->parent)
    hwloc_bitmap_copy(obj->nodeset, obj->parent->nodeset);
  else
    hwloc_bitmap_zero(obj->nodeset);

  /* 不清除complete_nodeset，只需确保它包含nodeset。
   * 我们不能在根节点清除complete_nodeset并向下重建，因为一些位可能对应于拓扑中缺失的离线/不允许的NUMA节点。
   */
  if (!obj->complete_nodeset)
    # 如果对象的完整节点集合为空，则复制节点集合
    obj->complete_nodeset = hwloc_bitmap_dup(obj->nodeset);
  else
    # 否则，将对象的完整节点集合与节点集合进行按位或操作
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, obj->nodeset);

  # 现在添加我们的本地节点集合
  for_each_memory_child(child, obj) {
    # 将内存子节点的节点集合添加到我们的节点集合中
    hwloc_bitmap_or(obj->nodeset, obj->nodeset, child->nodeset);
    # 将内存子节点的完整节点集合添加到我们的完整节点集合中
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, child->complete_nodeset);
    # 不需要递归，因为hwloc__attach_memory_object()确保每个内存层次结构内的节点集合是一致的。
  }

  # 将我们的节点集合传播给CPU子节点。
  for_each_child(child, obj) {
    propagate_nodeset(child);
  }

  # 将CPU子节点的特定节点集合传播回给我们。
  # 不能合并这两个循环，因为我们不希望首先将子节点的节点集合传播回给我们，然后再传播给第二个子节点。
  # 每个子节点可能有自己的本地节点集合，每个节点集合都传播给我们，但不传播给其他子节点。
  for_each_child(child, obj) {
    # 将子节点的节点集合与我们的节点集合进行按位或操作
    hwloc_bitmap_or(obj->nodeset, obj->nodeset, child->nodeset);
    # 将子节点的完整节点集合与我们的完整节点集合进行按位或操作
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, child->complete_nodeset);
  }

  # I/O或Misc下没有节点集合
}

static void
remove_unused_sets(hwloc_topology_t topology, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  // 对象的 cpuset 与 topology 允许的 cpuset 取交集
  hwloc_bitmap_and(obj->cpuset, obj->cpuset, topology->allowed_cpuset);
  // 对象的 nodeset 与 topology 允许的 nodeset 取交集
  hwloc_bitmap_and(obj->nodeset, obj->nodeset, topology->allowed_nodeset);

  // 遍历对象的每个子对象，递归调用 remove_unused_sets
  for_each_child(child, obj)
    remove_unused_sets(topology, child);
  // 遍历对象的每个内存子对象，递归调用 remove_unused_sets
  for_each_memory_child(child, obj)
    remove_unused_sets(topology, child);
  /* I/O 或 Misc 下没有 cpuset */
}

static void
hwloc__filter_bridges(hwloc_topology_t topology, hwloc_obj_t root, unsigned depth)
{
  hwloc_obj_t child, *pchild;

  /* 过滤 I/O 子对象并递归 */
  for_each_io_child_safe(child, root, pchild) {
    enum hwloc_type_filter_e filter = topology->type_filter[child->type];

    /* 递归进入孙子对象 */
    hwloc__filter_bridges(topology, child, depth+1);

    child->attr->bridge.depth = depth;

    /* 移除没有子对象的桥接对象，
     * 以及没有子对象的 pci-to-non-pci 桥接对象（pcidev）。
     * 保留 NVSwitch，因为它们可能在 NVLink 矩阵中使用。
     */
    if (filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT
    && !child->io_first_child
        && (child->type == HWLOC_OBJ_BRIDGE
            || (child->type == HWLOC_OBJ_PCI_DEVICE && (child->attr->pcidev.class_id >> 8) == 0x06
                && (!child->subtype || strcmp(child->subtype, "NVSwitch"))))) {
      unlink_and_free_single_object(pchild);
      topology->modified = 1;
    }
  }
}

static void
hwloc_filter_bridges(hwloc_topology_t topology, hwloc_obj_t parent)
{
  hwloc_obj_t child = parent->first_child;
  while (child) {
    hwloc_filter_bridges(topology, child);
    child = child->next_sibling;
  }

  hwloc__filter_bridges(topology, parent, 0);
}

void
hwloc__reorder_children(hwloc_obj_t parent)
{
  /* 将子对象列表移至一侧 */
  hwloc_obj_t *prev, child, children = parent->first_child;
  parent->first_child = NULL;
  while (children) {
    /* 出队子对象 */
    child = children;
    children = child->next_sibling;
    /* 找到要入队的位置 */
    prev = &parent->first_child;  // 找到父节点的第一个子节点的指针地址
    while (*prev && hwloc__object_cpusets_compare_first(child, *prev) > 0)  // 当前节点不为空且子节点的第一个 CPU 集合比当前节点的第一个 CPU 集合大
      prev = &((*prev)->next_sibling);  // 更新 prev 指向下一个兄弟节点的指针地址
    /* 入队 */
    child->next_sibling = *prev;  // 将当前节点的下一个兄弟节点指向 prev 指向的节点
    *prev = child;  // 更新 prev 指向的节点为当前节点
  }
  /* 对于 Misc 或 I/O 子节点没有需要强制执行的顺序。 */
}

/* 移除所有 cpuset 为空的普通子对象，以及 nodeset 为空的内存子对象。
 * 同时不移除具有 I/O 子对象的对象，但忽略 Misc。
 */
static void
remove_empty(hwloc_topology_t topology, hwloc_obj_t *pobj)
{
  hwloc_obj_t obj = *pobj, child, *pchild;

  // 递归遍历所有子对象，移除空对象
  for_each_child_safe(child, obj, pchild)
    remove_empty(topology, pchild);
  // 递归遍历所有内存子对象，移除空对象
  for_each_memory_child_safe(child, obj, pchild)
    remove_empty(topology, pchild);
  /* I/O 或 Misc 下没有 cpuset */

  // 如果对象有普通子对象、内存子对象或 I/O 子对象，则不移除
  if (obj->first_child /* 只有在上面所有子对象都被移除后才移除，以免移除 NUMAnode 的父对象 */
      || obj->memory_first_child /* 只有在没有内存附加时才移除 */
      || obj->io_first_child /* 只有在没有 I/O 附加时才移除 */)
    /* 忽略 Misc */
    return;

  // 如果对象是普通对象且 cpuset 不为空，则不移除
  if (hwloc__obj_type_is_normal(obj->type)) {
    if (!hwloc_bitmap_iszero(obj->cpuset))
      return;
  } else {
    assert(hwloc__obj_type_is_memory(obj->type));
    // 如果对象是内存对象且 nodeset 不为空，则不移除
    if (!hwloc_bitmap_iszero(obj->nodeset))
      return;
  }

  hwloc_debug("%s", "\nRemoving empty object ");
  hwloc_debug_print_object(0, obj);
  unlink_and_free_single_object(pobj);
  topology->modified = 1;
}

/* 在修改层级结构之前重置类型深度 */
static void
hwloc_reset_normal_type_depths(hwloc_topology_t topology)
{
  unsigned i;
  for (i=HWLOC_OBJ_TYPE_MIN; i<=HWLOC_OBJ_GROUP; i++)
    topology->type_depth[i] = HWLOC_TYPE_DEPTH_UNKNOWN;
  /* 类型连续性在 topology_check() 中被断言 */
  topology->type_depth[HWLOC_OBJ_DIE] = HWLOC_TYPE_DEPTH_UNKNOWN;
}

static int
hwloc_dont_merge_group_level(hwloc_topology_t topology, unsigned i)
{
  unsigned j;

  /* 在该层级中不要合并某些组？ */
  for(j=0; j<topology->level_nbobjects[i]; j++)
    if (topology->levels[i][j]->attr->group.dont_merge)
      return 1;

  return 0;
}

/* 比较第 i 和第 i-1 层级的结构 */
static int
# 比较给定层级的结构是否符合要求
def hwloc_compare_levels_structure(hwloc_topology_t topology, unsigned i):
    # 检查是否需要检查内存
    checkmemory = (topology->levels[i][0]->type == HWLOC_OBJ_PU)
    # 初始化变量 j
    unsigned j;

    # 如果前一层级的对象数量不等于当前层级的对象数量，则返回-1
    if (topology->level_nbobjects[i-1] != topology->level_nbobjects[i])
        return -1;

    # 遍历当前层级的对象
    for j in range(topology->level_nbobjects[i]):
        # 如果前一层级的对象不等于当前层级对象的父对象，则返回-1
        if (topology->levels[i-1][j] != topology->levels[i][j]->parent)
            return -1;
        # 如果前一层级的对象的度数不为1，则返回-1
        if (topology->levels[i-1][j]->arity != 1)
            return -1;
        # 如果需要检查内存且前一层级的对象的内存度数不为0，则返回-1
        if (checkmemory and topology->levels[i-1][j]->memory_arity)
            # 如果上方有内存，则不合并处理器
            return -1;
    # 如果以上条件都不满足，则返回0
    return 0;

# 如果有任何层级被移除，则返回大于0的值
# 如果需要，内部执行重新连接
static int
hwloc_filter_levels_keep_structure(hwloc_topology_t topology):
    # 初始化变量 i, j, res
    unsigned i, j;
    int res = 0;

    # 如果拓扑结构被修改
    if (topology->modified):
        # 警告：hwloc_topology_reconnect() 在这里部分重复
        # - 我们需要在合并之前正常的层级
        # - 并且在合并之后需要更新特殊的层级
        hwloc_connect_children(topology->levels[0][0])
        if (hwloc_connect_levels(topology) < 0)
            return -1;

    # 从底部开始，因为我们将移除中间层级
    for i in range(topology->nb_levels-1, 0, -1):
        replacechild = 0
        replaceparent = 0
        obj1 = topology->levels[i-1][0]
        obj2 = topology->levels[i][0]
        type1 = obj1->type
        type2 = obj2->type

        # 检查是否可以替换父对象和/或子对象
        if (topology->type_filter[type1] == HWLOC_TYPE_FILTER_KEEP_STRUCTURE):
            # 父对象可以被忽略，以 favor 子对象
            replaceparent = 1
            if (type1 == HWLOC_OBJ_GROUP and hwloc_dont_merge_group_level(topology, i-1)):
                replaceparent = 0;
    # 如果 type2 对应的过滤器类型为 HWLOC_TYPE_FILTER_KEEP_STRUCTURE
    if (topology->type_filter[type2] == HWLOC_TYPE_FILTER_KEEP_STRUCTURE) {
      # 子对象可以被忽略，而选择父对象
      replacechild = 1;
      # 如果 type1 为 HWLOC_OBJ_GROUP 并且不应合并组级别
      if (type1 == HWLOC_OBJ_GROUP && hwloc_dont_merge_group_level(topology, i))
    replacechild = 0;
    }
    # 如果既不替换子对象也不替换父对象，则继续下一次循环
    if (!replacechild && !replaceparent)
      # 没有忽略的情况
      continue;
    # 决定实际要替换的对象
    if (replaceparent && replacechild) {
      # 如果两者都可以替换，查看 obj_type_priority
      if (obj_type_priority[type1] >= obj_type_priority[type2])
    replaceparent = 0;
      else
    replacechild = 0;
    }
    # 这些级别实际上是相同的吗？
    if (hwloc_compare_levels_structure(topology, i) < 0)
      continue;
    # 输出可以合并的级别信息
    hwloc_debug("may merge levels #%u=%s and #%u=%s\n",
        i-1, hwloc_obj_type_string(type1), i, hwloc_obj_type_string(type2));

    # 从树中移除中间对象
    for(j=0; j<topology->level_nbobjects[i]; j++) {
      # 获取父对象和子对象
      hwloc_obj_t parent = topology->levels[i-1][j];
      hwloc_obj_t child = topology->levels[i][j];
      unsigned k;
      if (replacechild) {
    # 将子对象的子对象移动到父对象
    parent->first_child = child->first_child;
    parent->last_child = child->last_child;
    parent->arity = child->arity;
    free(parent->children);
    parent->children = child->children;
    child->children = NULL;
    # 更新子对象的父对象
    for(k=0; k<parent->arity; k++)
      parent->children[k]->parent = parent;
    # 将子对象的内存/IO/其他子对象追加到父对象
    if (child->memory_first_child) {
      append_siblings_list(&parent->memory_first_child, child->memory_first_child, parent);
      parent->memory_arity += child->memory_arity;
    }
    if (child->io_first_child) {
      append_siblings_list(&parent->io_first_child, child->io_first_child, parent);
      parent->io_arity += child->io_arity;
    }
    # 如果子节点存在
    if (child->misc_first_child) {
      # 将子节点的兄弟节点追加到父节点的兄弟节点列表中，并更新父节点的 misc_arity
      append_siblings_list(&parent->misc_first_child, child->misc_first_child, parent);
      parent->misc_arity += child->misc_arity;
    }
    # 释放未连接的对象 child
    hwloc_free_unlinked_object(child);
      } else {
    /* 在爷爷节点中用子节点替换父节点 */
    if (parent->parent) {
      # 将父节点替换为子节点，并更新 sibling_rank
      parent->parent->children[parent->sibling_rank] = child;
      child->sibling_rank = parent->sibling_rank;
      # 如果父节点是第一个孩子
      if (!parent->sibling_rank) {
        parent->parent->first_child = child;
        /* child->prev_sibling 已经是 NULL，child 是单独的 */
      } else {
        # 更新子节点的前一个兄弟节点和后一个兄弟节点
        child->prev_sibling = parent->parent->children[parent->sibling_rank-1];
        child->prev_sibling->next_sibling = child;
      }
      # 如果父节点是最后一个孩子
      if (parent->sibling_rank == parent->parent->arity-1) {
        parent->parent->last_child = child;
        /* child->next_sibling 已经是 NULL，child 是单独的 */
      } else {
        # 更新子节点的前一个兄弟节点和后一个兄弟节点
        child->next_sibling = parent->parent->children[parent->sibling_rank+1];
        child->next_sibling->prev_sibling = child;
      }
      # 更新子节点的父节点
      child->parent = parent->parent;
    } else {
      # 将子节点作为新的根节点
      topology->levels[0][0] = child;
      child->parent = NULL;
    }
    # 将父节点的内存/IO/misc子节点放在子节点之前
    if (parent->memory_first_child) {
      prepend_siblings_list(&child->memory_first_child, parent->memory_first_child, child);
      child->memory_arity += parent->memory_arity;
    }
    if (parent->io_first_child) {
      prepend_siblings_list(&child->io_first_child, parent->io_first_child, child);
      child->io_arity += parent->io_arity;
    }
    if (parent->misc_first_child) {
      prepend_siblings_list(&child->misc_first_child, parent->misc_first_child, child);
      child->misc_arity += parent->misc_arity;
    }
    # 释放未连接的对象 parent
    hwloc_free_unlinked_object(parent);
    /* prev/next_sibling 将在下面的另一个循环中更新 */
      }
    }
    if (replaceparent && i>1) {
      /* 如果需要替换父节点并且当前节点的深度大于1，则执行以下操作 */
      for(j=0; j<topology->level_nbobjects[i]; j++) {
        // 遍历当前深度下的所有对象
        hwloc_obj_t child = topology->levels[i][j];
        // 获取当前对象
        unsigned rank = child->sibling_rank;
        // 获取当前对象在兄弟节点中的排名
        child->prev_sibling = rank > 0 ? child->parent->children[rank-1] : NULL;
        // 设置当前对象的前一个兄弟节点
        child->next_sibling = rank < child->parent->arity-1 ? child->parent->children[rank+1] : NULL;
        // 设置当前对象的后一个兄弟节点
      }
    }

    /* 更新层级以避免下一次重新连接时混淆 */
    if (replaceparent) {
      /* 移除第 i-1 层级，将第 i 层级及之后的层级移动到第 i-1 层级及之后 */
      free(topology->levels[i-1]);
      // 释放第 i-1 层级的内存
      memmove(&topology->levels[i-1],
          &topology->levels[i],
          (topology->nb_levels-i)*sizeof(topology->levels[i]));
      // 移动层级数组
      memmove(&topology->level_nbobjects[i-1],
          &topology->level_nbobjects[i],
          (topology->nb_levels-i)*sizeof(topology->level_nbobjects[i]));
      // 移动层级对象数量数组
      hwloc_debug("removed parent level %s at depth %u\n",
          hwloc_obj_type_string(type1), i-1);
      // 输出调试信息，表示移除了父节点层级
    } else {
      /* 移除第 i 层级，将第 i+1 层级及之后的层级移动到第 i 层级及之后 */
      free(topology->levels[i]);
      // 释放第 i 层级的内存
      memmove(&topology->levels[i],
          &topology->levels[i+1],
          (topology->nb_levels-1-i)*sizeof(topology->levels[i]));
      // 移动层级数组
      memmove(&topology->level_nbobjects[i],
          &topology->level_nbobjects[i+1],
          (topology->nb_levels-1-i)*sizeof(topology->level_nbobjects[i]));
      // 移动层级对象数量数组
      hwloc_debug("removed child level %s at depth %u\n",
          hwloc_obj_type_string(type2), i);
      // 输出调试信息，表示移除了子节点层级
    }
    topology->level_nbobjects[topology->nb_levels-1] = 0;
    // 将最后一个层级的对象数量设置为0
    topology->levels[topology->nb_levels-1] = NULL;
    // 将最后一个层级的对象数组设置为NULL
    topology->nb_levels--;
    // 层级数量减一

    res++;
    // 结果加一
  }

  if (res > 0) {
    /* 如果移除了一些层级，则更新对象和类型的深度 */
    hwloc_reset_normal_type_depths(topology);
    // 重置对象和类型的深度
    # 遍历拓扑结构的层级
    for(i=0; i<topology->nb_levels; i++) {
      # 获取当前层级的第一个对象的类型
      hwloc_obj_type_t type = topology->levels[i][0]->type;
      # 遍历当前层级的对象
      for(j=0; j<topology->level_nbobjects[i]; j++)
        # 将对象的深度设置为当前层级的索引值
        topology->levels[i][j]->depth = (int)i;
      # 如果当前对象类型的深度未知
      if (topology->type_depth[type] == HWLOC_TYPE_DEPTH_UNKNOWN)
        # 将当前对象类型的深度设置为当前层级的索引值
        topology->type_depth[type] = (int)i;
      else
        # 将当前对象类型的深度设置为多个层级
        topology->type_depth[type] = HWLOC_TYPE_DEPTH_MULTIPLE;
    }
  }


  # 如果结果大于0或者拓扑结构被修改过
  if (res > 0 || topology-> modified) {
    # 警告：hwloc_topology_reconnect() 在这里部分重复
    # 和在函数开头部分重复。
    # 如果我们合并了一些层级，一些子+父特殊子列表
    # 可能已经被合并，因此特殊层级可能需要重新排序，
    # 所以只在这里重新连接特殊层级
    # (在函数开头部分不需要)。
    # 如果连接特殊层级失败
    if (hwloc_connect_special_levels(topology) < 0)
      # 返回-1
      return -1;
    # 将拓扑结构的修改标志设置为0
    topology->modified = 0;
  }

  # 返回0
  return 0;
# 传播对称子树属性到整个拓扑结构
static void
hwloc_propagate_symmetric_subtree(hwloc_topology_t topology, hwloc_obj_t root)
{
  hwloc_obj_t child;
  unsigned arity = root->arity;
  hwloc_obj_t *array;
  int ok;

  # 默认情况下假设不对称
  root->symmetric_subtree = 0;

  # 如果没有子节点，则认为是对称的
  if (!arity)
    goto good;

  # 忽略内存，就像忽略 I/O 和 Misc 一样？
  
  # 只查看普通子节点，忽略 I/O 和 Misc。如果任何子节点不对称，则返回
  ok = 1;
  for_each_child(child, root) {
    hwloc_propagate_symmetric_subtree(topology, child);
    if (!child->symmetric_subtree)
      ok = 0;
  }
  if (!ok)
    return;
  # Misc 和 I/O 子节点不关心 symmetric_subtree

  # 如果只有一个子节点是对称的，那就很好
  if (arity == 1)
    goto good;

  # 现在检查子树是否相同。遍历每个树的第一个子节点，比较它们的深度和度
  array = malloc(arity * sizeof(*array));
  if (!array)
    return;
  memcpy(array, root->children, arity * sizeof(*array));
  while (1) {
    unsigned i;
    # 检查当前级别的度和深度
    for(i=1; i<arity; i++)
      if (array[i]->depth != array[0]->depth
      || array[i]->arity != array[0]->arity) {
    free(array);
    return;
      }
    if (!array[0]->arity)
      # 没有更多子节点级别，我们很好
      break;
    # 现在查看每个元素的第一个子节点
    for(i=0; i<arity; i++)
      array[i] = array[i]->first_child;
  }
  free(array);

  # 一切顺利，我们是对称的
 good:
  root->symmetric_subtree = 1;
}

# 设置组的深度
static void hwloc_set_group_depth(hwloc_topology_t topology)
{
  unsigned groupdepth = 0;
  unsigned i, j;
  for(i=0; i<topology->nb_levels; i++)
    if (topology->levels[i][0]->type == HWLOC_OBJ_GROUP) {
      for (j = 0; j < topology->level_nbobjects[i]; j++)
    topology->levels[i][j]->attr->group.depth = groupdepth;
      groupdepth++;
    }
}
/*
 * 初始化整个拓扑结构中的便捷指针。
 * 拓扑结构只有 first_child 和 next_sibling 指针。
 * 当此函数返回时，所有父/子指针都被初始化。
 * 剩余的字段（levels、cousins、logical_index、depth 等）将在 hwloc_connect_levels() 中设置。
 *
 * 可以被多次调用，因此可能需要更新数组。
 */
static void
hwloc_connect_children(hwloc_obj_t parent)
{
  unsigned n, oldn = parent->arity;
  hwloc_obj_t child, prev_child;
  int ok;

  /* 主要子节点列表 */

  ok = 1;
  prev_child = NULL;
  for (n = 0, child = parent->first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
    /* 数组中已经是正确的了吗？ */
    if (n >= oldn || parent->children[n] != child)
      ok = 0;
    /* 递归 */
    hwloc_connect_children(child);
  }
  parent->last_child = prev_child;
  parent->arity = n;
  if (!n) {
    /* 不再需要数组 */
    free(parent->children);
    parent->children = NULL;
    goto memory;
  }
  if (ok)
    /* 数组已经是正确的了（即使太大） */
    goto memory;

  /* 如果需要，分配一个更大的数组 */
  if (oldn < n) {
    free(parent->children);
    parent->children = malloc(n * sizeof(*parent->children));
  }
  /* 填充 */
  for (n = 0, child = parent->first_child;
       child;
       n++,   child = child->next_sibling) {
    parent->children[n] = child;
  }



 memory:
  /* 内存子节点列表 */

  prev_child = NULL;
  for (n = 0, child = parent->memory_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->parent = parent;
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
  # 连接内存子节点列表
  prev_child = NULL;
  for (n = 0, child = parent->memory_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    # 设置子节点的父节点为当前节点
    child->parent = parent;
    # 设置子节点在兄弟节点中的顺序
    child->sibling_rank = n;
    # 设置子节点的前一个兄弟节点
    child->prev_sibling = prev_child;
    # 连接子节点的子节点
    hwloc_connect_children(child);
  }
  # 设置内存子节点的数量
  parent->memory_arity = n;

  # I/O子节点列表
  prev_child = NULL;
  for (n = 0, child = parent->io_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    # 设置子节点的父节点为当前节点
    child->parent = parent;
    # 设置子节点在兄弟节点中的顺序
    child->sibling_rank = n;
    # 设置子节点的前一个兄弟节点
    child->prev_sibling = prev_child;
    # 连接子节点的子节点
    hwloc_connect_children(child);
  }
  # 设置I/O子节点的数量
  parent->io_arity = n;

  # 其他子节点列表
  prev_child = NULL;
  for (n = 0, child = parent->misc_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    # 设置子节点的父节点为当前节点
    child->parent = parent;
    # 设置子节点在兄弟节点中的顺序
    child->sibling_rank = n;
    # 设置子节点的前一个兄弟节点
    child->prev_sibling = prev_child;
    # 连接子节点的子节点
    hwloc_connect_children(child);
  }
  # 设置其他子节点的数量
  parent->misc_arity = n;
}
/*
 * 检查是否在 ROOT 下有一个对象严格位于 OBJ 下，且类型与 OBJ 相同
 */
static int
find_same_type(hwloc_obj_t root, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  for_each_child (child, root) {
    if (hwloc_type_cmp(child, obj) == HWLOC_OBJ_EQUAL)
      return 1;
    if (find_same_type(child, obj))
      return 1;
  }

  return 0;
}

static int
hwloc_build_level_from_list(struct hwloc_special_level_s *slevel)
{
  unsigned i, nb;
  struct hwloc_obj * obj;

  /* 计数 */
  obj = slevel->first;
  i = 0;
  while (obj) {
    i++;
    obj = obj->next_cousin;
  }
  nb = i;

  if (nb) {
    /* 分配并填充级别 */
    slevel->objs = malloc(nb * sizeof(struct hwloc_obj *));
    if (!slevel->objs)
      return -1;

    obj = slevel->first;
    i = 0;
    while (obj) {
      obj->logical_index = i;
      slevel->objs[i] = obj;
      i++;
      obj = obj->next_cousin;
    }
  }

  slevel->nbobjs = nb;
  return 0;
}

static void
hwloc_append_special_object(struct hwloc_special_level_s *level, hwloc_obj_t obj)
{
  if (level->first) {
    obj->prev_cousin = level->last;
    obj->prev_cousin->next_cousin = obj;
    level->last = obj;
  } else {
    obj->prev_cousin = NULL;
    level->first = level->last = obj;
  }
}

/* 将特殊对象追加到它们的列表中 */
static void
hwloc_list_special_objects(hwloc_topology_t topology, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  if (obj->type == HWLOC_OBJ_NUMANODE) {
    obj->next_cousin = NULL;
    obj->depth = HWLOC_TYPE_DEPTH_NUMANODE;
    /* 插入主 NUMA 节点列表 */
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_NUMANODE], obj);

    /* 递归，NUMA 节点只有 Misc 子节点 */
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (obj->type == HWLOC_OBJ_MEMCACHE) {
    obj->next_cousin = NULL;
    obj->depth = HWLOC_TYPE_DEPTH_MEMCACHE;
    /* 插入主 MemCache 列表 */
    # 将特殊对象添加到相应的层级中
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_MEMCACHE], obj);

    # 递归处理，MemCaches 可能有 NUMA 节点或 Misc 子对象
    for_each_memory_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (obj->type == HWLOC_OBJ_MISC) {
    # 设置下一个兄弟节点为空
    obj->next_cousin = NULL;
    # 设置深度为 Misc 类型的深度
    obj->depth = HWLOC_TYPE_DEPTH_MISC;
    # 将主 Misc 列表中插入对象
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_MISC], obj);
    # 递归处理，Misc 只有 Misc 子对象
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (hwloc__obj_type_is_io(obj->type)) {
    # 设置下一个兄弟节点为空
    obj->next_cousin = NULL;

    if (obj->type == HWLOC_OBJ_BRIDGE) {
      # 设置深度为 Bridge 类型的深度
      obj->depth = HWLOC_TYPE_DEPTH_BRIDGE;
      # 将对象插入主桥列表中
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_BRIDGE], obj);

    } else if (obj->type == HWLOC_OBJ_PCI_DEVICE) {
      # 设置深度为 PCI 设备类型的深度
      obj->depth = HWLOC_TYPE_DEPTH_PCI_DEVICE;
      # 将对象插入主 PCI 设备列表中
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_PCIDEV], obj);

    } else if (obj->type == HWLOC_OBJ_OS_DEVICE) {
      # 设置深度为 OS 设备类型的深度
      obj->depth = HWLOC_TYPE_DEPTH_OS_DEVICE;
      # 将对象插入主 OS 设备列表中
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_OSDEV], obj);
    }

    # 递归处理，I/O 只有 I/O 和 Misc 子对象
    for_each_io_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else {
    # 递归处理
    for_each_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_memory_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_io_child(child, obj)
      hwloc_list_special_objects(topology, child);
    # 对每个子节点执行指定的函数，并传入额外的参数
    for_each_misc_child(child, obj)
      # 调用 hwloc_list_special_objects 函数，列出拓扑结构中特殊的对象
      hwloc_list_special_objects(topology, child);
  }
# 构建内存、I/O和其他特殊级别
static int
hwloc_connect_special_levels(hwloc_topology_t topology)
{
  unsigned i;

  # 释放特殊级别对象
  for(i=0; i<HWLOC_NR_SLEVELS; i++)
    free(topology->slevels[i].objs);
  # 将特殊级别对象清零
  memset(&topology->slevels, 0, sizeof(topology->slevels));

  # 列出特殊对象
  hwloc_list_special_objects(topology, topology->levels[0][0]);

  # 构建特殊级别对象
  for(i=0; i<HWLOC_NR_SLEVELS; i++) {
    if (hwloc_build_level_from_list(&topology->slevels[i]) < 0)
      return -1;
  }

  return 0;
}

'''
'''
# 执行hwloc_connect_children()未完成的剩余工作
# 需要对象度和子对象列表被正确初始化（由hwloc_connect_children()完成）
static int
hwloc_connect_levels(hwloc_topology_t topology)
{
  unsigned l, i=0;
  hwloc_obj_t *objs, *taken_objs, *new_objs, top_obj, root;
  unsigned n_objs, n_taken_objs, n_new_objs;

  # 重置非根级别（根在初始化时已经完成，这里不会改变）
  for(l=1; l<topology->nb_levels; l++)
    free(topology->levels[l]);
  memset(topology->levels+1, 0, (topology->nb_levels-1)*sizeof(*topology->levels));
  memset(topology->level_nbobjects+1, 0, (topology->nb_levels-1)*sizeof(*topology->level_nbobjects));
  topology->nb_levels = 1;

  # 初始化所有非I/O/非其他级别的深度为未知
  hwloc_reset_normal_type_depths(topology);

  # 初始化根类型深度
  root = topology->levels[0][0];
  root->depth = 0;
  topology->type_depth[root->type] = 0;
  # 根级别
  root->logical_index = 0;
  root->prev_cousin = NULL;
  root->next_cousin = NULL;
  # 根作为没有父级的子对象
  root->parent = NULL;
  root->sibling_rank = 0;
  root->prev_sibling = NULL;
  root->next_sibling = NULL;

  # 从整个系统的子对象开始
  n_objs = topology->levels[0][0]->arity;
  objs = malloc(n_objs * sizeof(objs[0]));
  if (!objs) {
    errno = ENOMEM;
    # 返回错误代码 -1
    return -1;
  }
  # 将 topology->levels[0][0]->children 的数据复制到 objs 数组中
  memcpy(objs, topology->levels[0][0]->children, n_objs*sizeof(objs[0]));

  # 当 OBJS 中还有对象时，继续构建层级结构
  while (n_objs) {
    # 在这一点上，objs 数组仅包含可能进入层级的对象

    # 首先找到顶层对象的类型
    # 如果还有其他类型的对象，则不使用 PU，因为我们希望将 PU 保留在底部
    for (i = 0; i < n_objs; i++)
      if (objs[i]->type != HWLOC_OBJ_PU)
        break;
    top_obj = i == n_objs ? objs[0] : objs[i];

    # 查看这实际上是否是顶层对象
    for (i = 0; i < n_objs; i++) {
      if (hwloc_type_cmp(top_obj, objs[i]) != HWLOC_OBJ_EQUAL) {
    if (find_same_type(objs[i], top_obj)) {
      # OBJS[i] 严格位于与 TOP_OBJ 相同类型的对象的上方，因此它位于 TOP_OBJ 的上方
      top_obj = objs[i];
    }
      }
    }

    # 现在查看所有相同类型的对象，构建一个具有该类型的层级，并用它们的子对象替换它们

    # 分配足够的空间以容纳所有当前对象和一个结束的 NULL
    taken_objs = malloc((n_objs+1) * sizeof(taken_objs[0]));
    if (!taken_objs) {
      free(objs);
      errno = ENOMEM;
      return -1;
    }

    # 分配足够的空间以保留所有当前对象或它们的子对象
    n_new_objs = 0;
    for (i = 0; i < n_objs; i++) {
      if (objs[i]->arity)
    n_new_objs += objs[i]->arity;
      else
    n_new_objs++;
    }
    new_objs = malloc(n_new_objs * sizeof(new_objs[0]));
    if (!new_objs) {
      free(objs);
      free(taken_objs);
      errno = ENOMEM;
      return -1;
    }

    # 现在实际上获取这些对象
    n_new_objs = 0;
    n_taken_objs = 0;
    for (i = 0; i < n_objs; i++)
      if (hwloc_type_cmp(top_obj, objs[i]) == HWLOC_OBJ_EQUAL) {
    # 获取它，添加主要子对象
    taken_objs[n_taken_objs++] = objs[i];
    // 如果对象的 arity 不为 0，则将对象的 children 复制到新对象数组中
    if (objs[i]->arity)
      memcpy(&new_objs[n_new_objs], objs[i]->children, objs[i]->arity * sizeof(new_objs[0]));
    // 更新新对象数组的计数
    n_new_objs += objs[i]->arity;
      } else {
    /* Leave it.  */
    // 否则将对象直接放入新对象数组中
    new_objs[n_new_objs++] = objs[i];
      }

    // 如果新对象数组为空，则释放内存并将指针置为空
    if (!n_new_objs) {
      free(new_objs);
      new_objs = NULL;
    }

    /* Ok, put numbers in the level and link cousins.  */
    // 为每个被选中的对象设置深度和逻辑索引，并链接兄弟节点
    for (i = 0; i < n_taken_objs; i++) {
      taken_objs[i]->depth = (int) topology->nb_levels;
      taken_objs[i]->logical_index = i;
      if (i) {
    taken_objs[i]->prev_cousin = taken_objs[i-1];
    taken_objs[i-1]->next_cousin = taken_objs[i];
      }
    }
    // 设置第一个和最后一个被选中对象的兄弟节点为 NULL
    taken_objs[0]->prev_cousin = NULL;
    taken_objs[n_taken_objs-1]->next_cousin = NULL;

    /* One more level!  */
    // 输出调试信息，表示增加了一个新的层级
    hwloc_debug("--- %s level", hwloc_obj_type_string(top_obj->type));
    hwloc_debug(" has number %u\n\n", topology->nb_levels);

    // 如果顶层对象的类型深度为未知，则设置其类型深度为当前层级数，否则标记为多个深度
    if (topology->type_depth[top_obj->type] == HWLOC_TYPE_DEPTH_UNKNOWN)
      topology->type_depth[top_obj->type] = (int) topology->nb_levels;
    else
      topology->type_depth[top_obj->type] = HWLOC_TYPE_DEPTH_MULTIPLE; /* mark as unknown */

    // 将最后一个被选中对象的指针置为 NULL
    taken_objs[n_taken_objs] = NULL;

    // 如果当前层级数等于已分配的层级数，则扩展层级数组
    if (topology->nb_levels == topology->nb_levels_allocated) {
      /* extend the arrays of levels */
      void *tmplevels, *tmpnbobjs;
      tmplevels = realloc(topology->levels,
              2 * topology->nb_levels_allocated * sizeof(*topology->levels));
      tmpnbobjs = realloc(topology->level_nbobjects,
              2 * topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));
      if (!tmplevels || !tmpnbobjs) {
        if (HWLOC_SHOW_CRITICAL_ERRORS())
          fprintf(stderr, "hwloc: failed to realloc level arrays to %u\n", topology->nb_levels_allocated * 2);

    /* if one realloc succeeded, make sure the caller will free the new buffer */
    // 如果扩展成功，则更新层级数组的指针
    if (tmplevels)
      topology->levels = tmplevels;
    if (tmpnbobjs)
      topology->level_nbobjects = tmpnbobjs;
    /* 重新分配内存失败后，topology->level_foo保持不变，将由调用者释放 */

    // 释放内存
    free(objs);
    // 释放内存
    free(taken_objs);
    // 释放内存
    free(new_objs);
    // 设置错误码为内存不足
    errno = ENOMEM;
    // 返回-1
    return -1;
      }
      // 重新分配更多的内存
      topology->levels = tmplevels;
      // 重新分配更多的内存
      topology->level_nbobjects = tmpnbobjs;
      // 将新分配的内存初始化为0
      memset(topology->levels + topology->nb_levels_allocated,
         0, topology->nb_levels_allocated * sizeof(*topology->levels));
      // 将新分配的内存初始化为0
      memset(topology->level_nbobjects + topology->nb_levels_allocated,
         0, topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));
      // 扩大已分配的内存空间
      topology->nb_levels_allocated *= 2;
    }
    /* 添加新的层级 */
    topology->level_nbobjects[topology->nb_levels] = n_taken_objs;
    topology->levels[topology->nb_levels] = taken_objs;

    // 层级数加一
    topology->nb_levels++;

    // 释放内存
    free(objs);

    /* 切换到新的对象 */
    objs = new_objs;
    n_objs = n_new_objs;
  }

  /* 现在是空的。 */
  // 释放内存
  free(objs);

  // 返回0
  return 0;
}

int
hwloc_topology_reconnect(struct hwloc_topology *topology, unsigned long flags)
{
  /* WARNING: when updating this function, the replicated code must
   * also be updated inside hwloc_filter_levels_keep_structure()
   */

  // 如果传入的标志不为0，设置错误码并返回-1
  if (flags) {
    errno = EINVAL;
    return -1;
  }
  // 如果拓扑结构未被修改，直接返回0
  if (!topology->modified)
    return 0;

  // 连接拓扑结构的子节点
  hwloc_connect_children(topology->levels[0][0]);

  // 连接拓扑结构的各个层级
  if (hwloc_connect_levels(topology) < 0)
    return -1;

  // 连接拓扑结构的特殊层级
  if (hwloc_connect_special_levels(topology) < 0)
    return -1;

  // 将拓扑结构的修改标志位设为0
  topology->modified = 0;

  // 返回0表示成功
  return 0;
}

/* for regression testing, make sure the order of io devices
 * doesn't change with the dentry order in the filesystem
 *
 * Only needed for OSDev for now.
 */
static hwloc_obj_t
hwloc_debug_insert_osdev_sorted(hwloc_obj_t queue, hwloc_obj_t obj)
{
  hwloc_obj_t *pcur = &queue;
  while (*pcur && strcmp((*pcur)->name, obj->name) < 0)
    pcur = &((*pcur)->next_sibling);
  obj->next_sibling = *pcur;
  *pcur = obj;
  return queue;
}

static void
hwloc_debug_sort_children(hwloc_obj_t root)
{
  hwloc_obj_t child;

  if (root->io_first_child) {
    hwloc_obj_t osdevqueue, *pchild;

    pchild = &root->io_first_child;
    osdevqueue = NULL;
    while ((child = *pchild) != NULL) {
      if (child->type != HWLOC_OBJ_OS_DEVICE) {
    /* keep non-osdev untouched */
    pchild = &child->next_sibling;
    continue;
      }

      /* dequeue this child */
      *pchild = child->next_sibling;
      child->next_sibling = NULL;

      /* insert in osdev queue in order */
      osdevqueue = hwloc_debug_insert_osdev_sorted(osdevqueue, child);
    }

    /* requeue the now-sorted osdev queue */
    *pchild = osdevqueue;
  }

  /* Recurse */
  for_each_child(child, root)
    hwloc_debug_sort_children(child);
  for_each_memory_child(child, root)
    hwloc_debug_sort_children(child);
  for_each_io_child(child, root)
    hwloc_debug_sort_children(child);
  /* no I/O under Misc */
}

void hwloc_alloc_root_sets(hwloc_obj_t root)
{
  /*
   * 所有集合最初都是 NULL。
   *
   * 至少一个后端应该调用此函数一次性初始化所有集合。
   * XML 在需要时才会使用它，以防只有一些集合在 XML 导入中给出。
   *
   * 其他后端可以检查 root->cpuset != NULL 来查看是否有人在他们之前发现了一些东西。
   */
  if (!root->cpuset)
     root->cpuset = hwloc_bitmap_alloc();
  if (!root->complete_cpuset)
     root->complete_cpuset = hwloc_bitmap_alloc();
  if (!root->nodeset)
    root->nodeset = hwloc_bitmap_alloc();
  if (!root->complete_nodeset)
    root->complete_nodeset = hwloc_bitmap_alloc();
}

static void
hwloc_discover_by_phase(struct hwloc_topology *topology,
            struct hwloc_disc_status *dstatus,
            const char *phasename __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  hwloc_debug("%s phase discovery...\n", phasename);
  for(backend = topology->backends; backend; backend = backend->next) {
    if (dstatus->phase & dstatus->excluded_phases)
      break;
    if (!(backend->phases & dstatus->phase))
      continue;
    if (!backend->discover)
      continue;
    hwloc_debug("%s phase discovery in component %s...\n", phasename, backend->component->name);
    backend->discover(backend, dstatus);
    hwloc_debug_print_objects(0, topology->levels[0][0]);
  }
}

/* 主发现循环 */
static int
hwloc_discover(struct hwloc_topology *topology,
           struct hwloc_disc_status *dstatus)
  // 设置拓扑结构的修改标志为0，表示暂时不需要重新连接
  topology->modified = 0; /* no need to reconnect yet */

  // 分配一个完整的允许的 CPU 集合和节点集合
  topology->allowed_cpuset = hwloc_bitmap_alloc_full();
  topology->allowed_nodeset = hwloc_bitmap_alloc_full();

  /* discover() 回调应该使用 hwloc_insert 来添加通过 hwloc_alloc_setup_object 初始化的对象
   * 对于节点级别，需要初始化节点集合和内存
   * 对于缓存级别，需要初始化内存和类型/深度
   * 对于组级别，需要初始化深度
   */

  /* 每个逻辑处理器至少应该有一个 PU 对象，最差情况下由 hwloc_setup_pu_level() 产生
   */

  /* 为了能够使用 hwloc__insert_object_by_cpuset 根据 cpuset 将对象插入拓扑结构中，cpuset 字段必须初始化
   */

  /* 一般来说，所有处理器在拓扑结构中都是可见的，并且对应应用程序是允许的
   *
   * - 如果存在一些处理器，但是对它们的拓扑信息是未知的（因此后端无法为它们创建对象），它们应该被添加到对象可能存在的最低对象的 complete_cpuset 字段中
   *
   * - 如果一些处理器对应应用程序是不允许的（例如出于管理原因），它们应该从 allowed_cpuset 字段中删除
   *
   * 对于节点集合 complete_nodeset 和 allowed_cpuset 也是一样的
   *
   * 如果这样的字段还不存在，可以分配并初始化为零（对于 complete），或者为全满（对于 allowed）。这些值在检测后会自动传播到整个树中
   */

  if (topology->backend_phases & HWLOC_DISC_PHASE_GLOBAL) {
    /* 通常，GLOBAL 是独立的
     * 但是 HWLOC_ANNOTATE_GLOBAL_COMPONENTS=1 允许可选的 ANNOTATE 步骤
     */
    struct hwloc_backend *global_backend = topology->backends;
    assert(global_backend);
    assert(global_backend->phases == HWLOC_DISC_PHASE_GLOBAL);
  }
    # 执行基于单个组件的全局发现
    hwloc_debug("GLOBAL phase discovery...\n");
    hwloc_debug("GLOBAL phase discovery with component %s...\n", global_backend->component->name);
    # 设置发现状态为全局阶段
    dstatus->phase = HWLOC_DISC_PHASE_GLOBAL;
    # 调用全局组件的发现函数
    global_backend->discover(global_backend, dstatus);
    # 打印顶层对象
    hwloc_debug_print_objects(0, topology->levels[0][0])

  # 不显式忽略其他阶段，以防将来需要重新引入
  # 大多数情况下，组件默认会排除它们
  # 除非显式请求注释全局组件
  if (topology->backend_phases & HWLOC_DISC_PHASE_CPU):
    # 首先发现 CPU
    dstatus->phase = HWLOC_DISC_PHASE_CPU
    hwloc_discover_by_phase(topology, dstatus, "CPU")

  if not (topology->backend_phases & (HWLOC_DISC_PHASE_GLOBAL|HWLOC_DISC_PHASE_CPU)):
    hwloc_debug("No GLOBAL or CPU component phase found\n")
    # 在下面会失败

  # 一个后端应该在 PU 和 NUMA 插入期间调用 hwloc_alloc_root_sets() 并设置位
  if not topology->levels[0][0]->cpuset or hwloc_bitmap_iszero(topology->levels[0][0]->cpuset):
    hwloc_debug("%s", "No PU added by any CPU or GLOBAL component phase\n")
    errno = EINVAL
    return -1

  # 内存特定的发现
  if (topology->backend_phases & HWLOC_DISC_PHASE_MEMORY):
    dstatus->phase = HWLOC_DISC_PHASE_MEMORY
  # 通过阶段性地发现内存拓扑，更新拓扑结构
  hwloc_discover_by_phase(topology, dstatus, "MEMORY");
  }

  # 检查是否可以获取本地允许资源的集合
  if (topology->binding_hooks.get_allowed_resources
      && topology->is_thissystem
      && !(dstatus->flags & HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES)
      && ((topology->flags & HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES) != 0
      || ((env = getenv("HWLOC_THISSYSTEM_ALLOWED_RESOURCES")) != NULL && atoi(env)))) {
    # 获取本地允许资源的集合
    topology->binding_hooks.get_allowed_resources(topology);
    dstatus->flags |= HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES;
  }

  # 如果没有 NUMA 节点，则添加一个包含所有内存的节点
  if (hwloc_bitmap_iszero(topology->levels[0][0]->complete_nodeset)) {
    # 分配并设置一个 NUMA 节点对象
    hwloc_obj_t node;
    hwloc_debug("%s", "\nAdd missing single NUMA node\n");
    node = hwloc_alloc_setup_object(topology, HWLOC_OBJ_NUMANODE, 0);
    node->cpuset = hwloc_bitmap_dup(topology->levels[0][0]->cpuset);
    node->nodeset = hwloc_bitmap_alloc();
    # 设置节点集合
    hwloc_bitmap_set(node->nodeset, 0);
    # 复制内存信息并清空机器内存信息
    memcpy(&node->attr->numanode, &topology->machine_memory, sizeof(topology->machine_memory));
    memset(&topology->machine_memory, 0, sizeof(topology->machine_memory));
    # 根据 cpuset 插入对象
    hwloc__insert_object_by_cpuset(topology, NULL, node, "core:defaultnumanode");
  } else {
    # 如果确定已经找到所有 NUMA 节点但没有它们的大小（x86 后端？），则可以将总内存分配给它们
    free(topology->machine_memory.page_types);
  }
    # 将 topology->machine_memory 的内存清零，大小为 sizeof(topology->machine_memory)
    memset(&topology->machine_memory, 0, sizeof(topology->machine_memory));
  }

  # 打印调试信息
  hwloc_debug("%s", "\nFixup root sets\n");
  # 对 topology->levels[0][0]->cpuset 和 topology->levels[0][0]->complete_cpuset 进行按位与操作
  hwloc_bitmap_and(topology->levels[0][0]->cpuset, topology->levels[0][0]->cpuset, topology->levels[0][0]->complete_cpuset);
  # 对 topology->levels[0][0]->nodeset 和 topology->levels[0][0]->complete_nodeset 进行按位与操作
  hwloc_bitmap_and(topology->levels[0][0]->nodeset, topology->levels[0][0]->nodeset, topology->levels[0][0]->complete_nodeset);

  # 对 topology->allowed_cpuset 和 topology->levels[0][0]->cpuset 进行按位与操作
  hwloc_bitmap_and(topology->allowed_cpuset, topology->allowed_cpuset, topology->levels[0][0]->cpuset);
  # 对 topology->allowed_nodeset 和 topology->levels[0][0]->nodeset 进行按位与操作
  hwloc_bitmap_and(topology->allowed_nodeset, topology->allowed_nodeset, topology->levels[0][0]->nodeset);

  # 打印调试信息
  hwloc_debug("%s", "\nPropagate sets\n");
  # 传播 nodeset 到 NUMA 节点的上下级
  propagate_nodeset(topology->levels[0][0]);
  # 修正父/子集合
  fixup_sets(topology->levels[0][0]);

  # 打印调试信息
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  # 如果标记中不包括 HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
  if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
    # 打印调试信息
    hwloc_debug("%s", "\nRemoving unauthorized sets from all sets\n");
    # 从所有集合中移除未授权的集合
    remove_unused_sets(topology, topology->levels[0][0]);
    # 打印调试信息
    hwloc_debug_print_objects(0, topology->levels[0][0]);
  }

  # 检查是否应该忽略根节点，现在我们知道它有多少子节点
  if (!hwloc_filter_check_keep_object(topology, topology->levels[0][0])
      && topology->levels[0][0]->first_child && !topology->levels[0][0]->first_child->next_sibling) {
    hwloc_obj_t oldroot = topology->levels[0][0];
    hwloc_obj_t newroot = oldroot->first_child;
    # 切换到新的根节点
    newroot->parent = NULL;
    topology->levels[0][0] = newroot;
    # 将旧根节点的内存/IO/其他子节点移动到新根节点的子节点之前
    if (oldroot->memory_first_child)
      prepend_siblings_list(&newroot->memory_first_child, oldroot->memory_first_child, newroot);
    if (oldroot->io_first_child)
      prepend_siblings_list(&newroot->io_first_child, oldroot->io_first_child, newroot);
    # 如果旧根节点有杂项子节点，则将其添加到新根节点的杂项子节点列表中
    if (oldroot->misc_first_child)
      prepend_siblings_list(&newroot->misc_first_child, oldroot->misc_first_child, newroot);
    # 销毁旧根节点并使用新根节点
    hwloc_free_unlinked_object(oldroot);
  }

  """
   * 所有对象的 cpuset 和 nodeset 现在都已经正确设置。
   """

  # 现在连接方便的指针，以使剩余的发现更容易。
  hwloc_debug("%s", "\nOk, finished tweaking, now connect\n");
  # 如果重新连接拓扑失败，则返回 -1
  if (hwloc_topology_reconnect(topology, 0) < 0)
    return -1;
  # 打印对象信息
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  """
   * 附加发现
   """
  # 准备 PCI 发现
  hwloc_pci_discovery_prepare(topology);
  # 如果后端阶段包括 PCI，则进行 PCI 发现
  if (topology->backend_phases & HWLOC_DISC_PHASE_PCI) {
    dstatus->phase = HWLOC_DISC_PHASE_PCI;
    hwloc_discover_by_phase(topology, dstatus, "PCI");
  }
  # 如果后端阶段包括 IO，则进行 IO 发现
  if (topology->backend_phases & HWLOC_DISC_PHASE_IO) {
    dstatus->phase = HWLOC_DISC_PHASE_IO;
    hwloc_discover_by_phase(topology, dstatus, "IO");
  }
  # 如果后端阶段包括 MISC，则进行 MISC 发现
  if (topology->backend_phases & HWLOC_DISC_PHASE_MISC) {
    dstatus->phase = HWLOC_DISC_PHASE_MISC;
    hwloc_discover_by_phase(topology, dstatus, "MISC");
  }
  # 如果后端阶段包括 ANNOTATE，则进行 ANNOTATE 发现
  if (topology->backend_phases & HWLOC_DISC_PHASE_ANNOTATE) {
    dstatus->phase = HWLOC_DISC_PHASE_ANNOTATE;
    hwloc_discover_by_phase(topology, dstatus, "ANNOTATE");
  }

  # 退出 PCI 发现（需要直到注释）
  hwloc_pci_discovery_exit(topology);

  # 如果环境变量中包含 HWLOC_DEBUG_SORT_CHILDREN，则对子节点进行排序
  if (getenv("HWLOC_DEBUG_SORT_CHILDREN"))
    hwloc_debug_sort_children(topology->levels[0][0]);

  # 移除一些内容

  hwloc_debug("%s", "\nRemoving bridge objects if needed\n");
  # 如果需要，移除桥接对象
  hwloc_filter_bridges(topology, topology->levels[0][0]);
  # 打印对象信息
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  hwloc_debug("%s", "\nRemoving empty objects\n");
  # 移除空对象
  remove_empty(topology, &topology->levels[0][0]);
  # 如果顶层对象为空
  if (!topology->levels[0][0]) {
    # 如果显示关键错误，则打印错误信息并返回 -1
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology became empty, aborting!\n");
    return -1;
  }
  # 如果顶层对象的 cpuset 为空
  if (hwloc_bitmap_iszero(topology->levels[0][0]->cpuset)) {
    # 如果存在关键错误，则在标准错误流中打印错误信息并返回-1
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology does not contain any PU, aborting!\n");
    return -1;
  }
  # 如果拓扑结构中不包含任何NUMA节点，则在标准错误流中打印错误信息并返回-1
  if (hwloc_bitmap_iszero(topology->levels[0][0]->nodeset)) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology does not contain any NUMA node, aborting!\n");
    return -1;
  }
  # 打印拓扑结构中的对象信息
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  # 打印信息，表示正在移除包含HWLOC_TYPE_FILTER_KEEP_STRUCTURE的层级
  hwloc_debug("%s", "\nRemoving levels with HWLOC_TYPE_FILTER_KEEP_STRUCTURE\n");
  # 如果移除包含HWLOC_TYPE_FILTER_KEEP_STRUCTURE的层级失败，则返回-1
  if (hwloc_filter_levels_keep_structure(topology) < 0)
    return -1;
  # 打印拓扑结构中的对象信息
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  # 累积子节点的内存到总内存字段中（只有在父节点设置后才能进行）
  hwloc_debug("%s", "\nPropagate total memory up\n");
  propagate_total_memory(topology->levels[0][0]);

  # 设置对称子树属性
  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);

  # 应用组深度
  hwloc_set_group_depth(topology);

  # 如果不是从XML加载拓扑结构，则添加一些识别属性
  if (topology->backends
      && strcmp(topology->backends->component->name, "xml")
      && !getenv("HWLOC_DONT_ADD_VERSION_INFO")) {
    char *value;
    # 添加hwlocVersion属性
    hwloc_obj_add_info(topology->levels[0][0], "hwlocVersion", HWLOC_VERSION);
    # 添加ProcessName属性
    value = hwloc_progname(topology);
    if (value) {
      hwloc_obj_add_info(topology->levels[0][0], "ProcessName", value);
      free(value);
    }
  }

  # 返回0表示成功
  return 0;
}

/* 在实际启动发现之前调用，
 * 重置所有内容，以防先前的加载初始化了一些内容。
 */
void
hwloc_topology_setup_defaults(struct hwloc_topology *topology)
}

static void hwloc__topology_filter_init(struct hwloc_topology *topology);

/* 此函数可能使用 tma，不能使用 free() 或 realloc() */
static int
hwloc__topology_init (struct hwloc_topology **topologyp,
              unsigned nblevels,
              struct hwloc_tma *tma)
{
  struct hwloc_topology *topology;

  // 使用 tma 分配内存来创建 topology 结构
  topology = hwloc_tma_malloc (tma, sizeof (struct hwloc_topology));
  if(!topology)
    # 返回错误代码 -1
    return -1;

  # 将 tma 赋值给 topology->tma
  topology->tma = tma;

  # 初始化硬件拓扑组件
  hwloc_components_init(); /* uses malloc without tma, but won't need it since dup() caller already took a reference */
  # 初始化硬件拓扑组件
  hwloc_topology_components_init(topology);
  # 初始化 PCI 设备发现
  hwloc_pci_discovery_init(topology); /* make sure both dup() and load() get sane variables */

  # 设置拓扑上下文
  topology->is_loaded = 0;
  topology->flags = 0;
  topology->is_thissystem = 1;
  topology->pid = 0;
  topology->userdata = NULL;
  topology->topology_abi = HWLOC_TOPOLOGY_ABI;
  topology->adopted_shmem_addr = NULL;
  topology->adopted_shmem_length = 0;

  # 为拓扑支持分配内存
  topology->support.discovery = hwloc_tma_malloc(tma, sizeof(*topology->support.discovery));
  topology->support.cpubind = hwloc_tma_malloc(tma, sizeof(*topology->support.cpubind));
  topology->support.membind = hwloc_tma_malloc(tma, sizeof(*topology->support.membind));
  topology->support.misc = hwloc_tma_malloc(tma, sizeof(*topology->support.misc));

  # 分配拓扑层级和对象数量的内存
  topology->nb_levels_allocated = nblevels; /* enough for default 10 levels = Mach+Pack+Die+NUMA+L3+L2+L1d+L1i+Co+PU */
  topology->levels = hwloc_tma_calloc(tma, topology->nb_levels_allocated * sizeof(*topology->levels));
  topology->level_nbobjects = hwloc_tma_calloc(tma, topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));

  # 初始化拓扑过滤器
  hwloc__topology_filter_init(topology);

  # 初始化内部距离
  hwloc_internal_distances_init(topology);
  # 初始化内部内存属性
  hwloc_internal_memattrs_init(topology);
  # 初始化内部 CPU 类型
  hwloc_internal_cpukinds_init(topology);

  # 设置拓扑的用户数据导出和导入回调函数
  topology->userdata_export_cb = NULL;
  topology->userdata_import_cb = NULL;
  topology->userdata_not_decoded = 0;

  # 设置拓扑的默认值
  hwloc_topology_setup_defaults(topology);

  # 将 topology 指针赋值给 topologyp，并返回成功代码 0
  *topologyp = topology;
  return 0;
}

int
hwloc_topology_init (struct hwloc_topology **topologyp)
{
  // 初始化拓扑结构
  return hwloc__topology_init(topologyp,
                  16, /* 16 is enough for default 10 levels = Mach+Pack+Die+NUMA+L3+L2+L1d+L1i+Co+PU */
                  NULL); /* no TMA for normal topologies, too many allocations to fix */
}

int
hwloc_topology_set_pid(struct hwloc_topology *topology __hwloc_attribute_unused,
                       hwloc_pid_t pid __hwloc_attribute_unused)
{
  // 如果拓扑结构已经加载，则返回错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  /* this does *not* change the backend */
#ifdef HWLOC_LINUX_SYS
  // 设置拓扑结构的进程 ID
  topology->pid = pid;
  return 0;
#else /* HWLOC_LINUX_SYS */
  // 如果不是 Linux 系统，则返回不支持的错误
  errno = ENOSYS;
  return -1;
#endif /* HWLOC_LINUX_SYS */
}

int
hwloc_topology_set_synthetic(struct hwloc_topology *topology, const char *description)
{
  // 如果拓扑结构已经加载，则返回错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  // 强制启用合成拓扑
  return hwloc_disc_component_force_enable(topology,
                       0 /* api */,
                       "synthetic",
                       description, NULL, NULL);
}

int
hwloc_topology_set_xml(struct hwloc_topology *topology,
               const char *xmlpath)
{
  // 如果拓扑结构已经加载，则返回错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  // 强制启用 XML 文件拓扑
  return hwloc_disc_component_force_enable(topology,
                       0 /* api */,
                       "xml",
                       xmlpath, NULL, NULL);
}

int
hwloc_topology_set_xmlbuffer(struct hwloc_topology *topology,
                             const char *xmlbuffer,
                             int size)
{
  // 如果拓扑结构已经加载，则返回错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  // 强制启用 XML 缓冲区拓扑
  return hwloc_disc_component_force_enable(topology,
                       0 /* api */,
                       "xml", NULL,
                       xmlbuffer, (void*) (uintptr_t) size);
}

int
hwloc_topology_set_flags (struct hwloc_topology *topology, unsigned long flags)
{
  // 如果拓扑结构已经加载，则返回错误
  if (topology->is_loaded) {
    /* actually harmless */
    errno = EBUSY;
    # 返回错误代码 -1
    return -1;
  }

  # 检查 flags 是否包含非法标志位，如果有则返回错误代码 -1
  if (flags & ~(HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
                |HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM
                |HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES
                |HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT
                |HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING
                |HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING
                |HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING
                |HWLOC_TOPOLOGY_FLAG_NO_DISTANCES
                |HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS
                |HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS)) {
    # 设置 errno 为无效参数错误代码
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }

  # 检查 flags 是否同时包含 RESTRICT_TO_CPUBINDING 和 IS_THISSYSTEM 标志位，如果是则返回错误代码 -1
  if ((flags & (HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING|HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)) == HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    # 设置 errno 为无效参数错误代码
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }
  # 检查 flags 是否同时包含 RESTRICT_TO_MEMBINDING 和 IS_THISSYSTEM 标志位，如果是则返回错误代码 -1
  if ((flags & (HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING|HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)) == HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING) {
    # 设置 errno 为无效参数错误代码
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }

  # 将 topology 对象的标志位设置为传入的 flags
  topology->flags = flags;
  # 返回成功代码 0
  return 0;
}

unsigned long
hwloc_topology_get_flags (struct hwloc_topology *topology)
{
  // 返回拓扑结构的标志位
  return topology->flags;
}

static void
hwloc__topology_filter_init(struct hwloc_topology *topology)
{
  hwloc_obj_type_t type;
  /* 只有默认情况下忽略无用的杂物 */
  for(type = HWLOC_OBJ_TYPE_MIN; type < HWLOC_OBJ_TYPE_MAX; type++)
    // 将所有对象类型的过滤器设置为保留全部
    topology->type_filter[type] = HWLOC_TYPE_FILTER_KEEP_ALL;
  // 将特定对象类型的过滤器设置为保留空
  topology->type_filter[HWLOC_OBJ_L1ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_L2ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_L3ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_MEMCACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_GROUP] = HWLOC_TYPE_FILTER_KEEP_STRUCTURE;
  topology->type_filter[HWLOC_OBJ_MISC] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_BRIDGE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_PCI_DEVICE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_OS_DEVICE] = HWLOC_TYPE_FILTER_KEEP_NONE;
}

static int
hwloc__topology_set_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter)
{
  if (type == HWLOC_OBJ_PU || type == HWLOC_OBJ_NUMANODE || type == HWLOC_OBJ_MACHINE) {
    if (filter != HWLOC_TYPE_FILTER_KEEP_ALL) {
      /* 我们需要 Machine、PU 和 NUMA 级别 */
      errno = EINVAL;
      return -1;
    }
  } else if (hwloc__obj_type_is_special(type)) {
    if (filter == HWLOC_TYPE_FILTER_KEEP_STRUCTURE) {
      /* I/O 和 Misc 不在主拓扑结构之内，没有意义 */
      errno = EINVAL;
      return -1;
    }
  } else if (type == HWLOC_OBJ_GROUP) {
    if (filter == HWLOC_TYPE_FILTER_KEEP_ALL) {
      /* 组始终被忽略，至少保留结构 */
      errno = EINVAL;
      return -1;
    }
  }

  /* 如果类型不是特殊类型，并且过滤器为保留重要类型，则将过滤器设置为保留所有类型 */
  if (!hwloc__obj_type_is_special(type) && filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT)
    filter = HWLOC_TYPE_FILTER_KEEP_ALL;

  // 将给定类型的过滤器设置为指定的过滤器
  topology->type_filter[type] = filter;
  // 返回成功
  return 0;
}

int
hwloc_topology_set_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter)
{
  // 确保 HWLOC_OBJ_TYPE_MIN 的值为 0
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  // 如果 type 超出范围，设置 errno 为 EINVAL，返回 -1
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX) {
    errno = EINVAL;
    return -1;
  }
  // 如果拓扑已加载，设置 errno 为 EBUSY，返回 -1
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }
  // 调用内部函数设置指定类型的过滤器
  return hwloc__topology_set_type_filter(topology, type, filter);
}

int
hwloc_topology_set_all_types_filter(struct hwloc_topology *topology, enum hwloc_type_filter_e filter)
{
  hwloc_obj_type_t type;
  // 如果拓扑已加载，设置 errno 为 EBUSY，返回 -1
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }
  // 遍历所有对象类型，调用内部函数设置过滤器
  for(type = HWLOC_OBJ_TYPE_MIN; type < HWLOC_OBJ_TYPE_MAX; type++)
    hwloc__topology_set_type_filter(topology, type, filter);
  return 0;
}

int
hwloc_topology_set_cache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  unsigned i;
  // 遍历缓存对象类型，调用内部函数设置过滤器
  for(i=HWLOC_OBJ_L1CACHE; i<HWLOC_OBJ_L3ICACHE; i++)
    hwloc_topology_set_type_filter(topology, (hwloc_obj_type_t) i, filter);
  return 0;
}

int
hwloc_topology_set_icache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  unsigned i;
  // 遍历指令缓存对象类型，调用内部函数设置过滤器
  for(i=HWLOC_OBJ_L1ICACHE; i<HWLOC_OBJ_L3ICACHE; i++)
    hwloc_topology_set_type_filter(topology, (hwloc_obj_type_t) i, filter);
  return 0;
}

int
hwloc_topology_set_io_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  // 设置桥、PCI 设备和操作系统设备的过滤器
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_BRIDGE, filter);
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_PCI_DEVICE, filter);
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_OS_DEVICE, filter);
  return 0;
}

int
hwloc_topology_get_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e *filterp)
{
  // 确保 HWLOC_OBJ_TYPE_MIN 的值为 0
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  // 如果 type 超出范围，设置 errno 为 EINVAL，返回 -1
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX) {
    errno = EINVAL;
    return -1;
  }
  // 获取指定类型的过滤器
  *filterp = topology->type_filter[type];
  return 0;
}

void
hwloc_topology_clear (struct hwloc_topology *topology)
  /* 释放拓扑结构中的 CPU 种类信息 */
  unsigned l;
  hwloc_internal_cpukinds_destroy(topology);
  /* 释放拓扑结构中的距离信息 */
  hwloc_internal_distances_destroy(topology);
  /* 释放拓扑结构中的内存属性信息 */
  hwloc_internal_memattrs_destroy(topology);
  /* 释放拓扑结构中的第一层对象及其子对象 */
  hwloc_free_object_and_children(topology->levels[0][0]);
  /* 释放拓扑结构中的允许的 CPU 集合 */
  hwloc_bitmap_free(topology->allowed_cpuset);
  /* 释放拓扑结构中的允许的节点集合 */
  hwloc_bitmap_free(topology->allowed_nodeset);
  /* 释放拓扑结构中每个层级的对象数组 */
  for (l=0; l<topology->nb_levels; l++)
    free(topology->levels[l]);
  /* 释放拓扑结构中每个系统层级的对象数组 */
  for(l=0; l<HWLOC_NR_SLEVELS; l++)
    free(topology->slevels[l].objs);
  /* 释放拓扑结构中的页面类型数组 */
  free(topology->machine_memory.page_types);
}

/* 销毁拓扑结构 */
void
hwloc_topology_destroy (struct hwloc_topology *topology)
{
  /* 如果拓扑结构采用了共享内存地址，则取消采用并返回 */
  if (topology->adopted_shmem_addr) {
    hwloc__topology_disadopt(topology);
    return;
  }

  /* 禁用所有后端 */
  hwloc_backends_disable_all(topology);
  /* 结束拓扑结构的组件 */
  hwloc_topology_components_fini(topology);
  /* 结束所有组件 */
  hwloc_components_fini();

  /* 清空拓扑结构 */
  hwloc_topology_clear(topology);

  /* 释放拓扑结构中的层级数组 */
  free(topology->levels);
  /* 释放拓扑结构中的每个层级对象数量数组 */
  free(topology->level_nbobjects);

  /* 释放拓扑结构中的发现支持信息 */
  free(topology->support.discovery);
  /* 释放拓扑结构中的 CPU 绑定支持信息 */
  free(topology->support.cpubind);
  /* 释放拓扑结构中的内存绑定支持信息 */
  free(topology->support.membind);
  /* 释放拓扑结构中的其他支持信息 */
  free(topology->support.misc);
  /* 释放拓扑结构 */
  free(topology);
}

/* 加载拓扑结构 */
int
hwloc_topology_load (struct hwloc_topology *topology)
{
  /* 初始化发现状态 */
  struct hwloc_disc_status dstatus;
  const char *env;
  int err;

  /* 如果拓扑结构已经加载，则返回错误 */
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  /* 初始化与环境变量相关的内容 */
  hwloc_internal_distances_prepare(topology);
  hwloc_internal_memattrs_prepare(topology);

  /* 如果设置了 HWLOC_XML_USERDATA_NOT_DECODED 环境变量，则不解码用户数据 */
  if (getenv("HWLOC_XML_USERDATA_NOT_DECODED"))
    topology->userdata_not_decoded = 1;

  /* 如果未设置 HWLOC_COMPONENTS 环境变量，则忽略变量，稍后会处理 */
  if (!getenv("HWLOC_COMPONENTS")) {
    /* 只有在尚未更改后端的情况下才应用变量。
     * 只会保留第一个变量。
     * 首先检查 FSROOT，因为它用于调试，所以可能需要覆盖其他所有内容。
     * 最后检查 XML（这可能是由管理员在系统范围内设置的）
     * 这样只有在其他变量未设置时才会使用它，以便用户可以轻松覆盖。
     */
    if (!topology->backends) {
      const char *fsroot_path_env = getenv("HWLOC_FSROOT");
      if (fsroot_path_env)
    hwloc_disc_component_force_enable(topology,
                      1 /* env force */,
                      "linux",
                      NULL /* backend will getenv again */, NULL, NULL);
    }
    if (!topology->backends) {
      const char *cpuid_path_env = getenv("HWLOC_CPUID_PATH");
      if (cpuid_path_env)
    hwloc_disc_component_force_enable(topology,
                      1 /* env force */,
                      "x86",
                      NULL /* backend will getenv again */, NULL, NULL);
    }
    if (!topology->backends) {
      const char *synthetic_env = getenv("HWLOC_SYNTHETIC");
      if (synthetic_env)
    hwloc_disc_component_force_enable(topology,
                      1 /* env force */,
                      "synthetic",
                      synthetic_env, NULL, NULL);
    }
    if (!topology->backends) {
      const char *xmlpath_env = getenv("HWLOC_XMLFILE");
      if (xmlpath_env)
    hwloc_disc_component_force_enable(topology,
                      1 /* env force */,
                      "xml",
                      xmlpath_env, NULL, NULL);
    }
  }

  dstatus.excluded_phases = 0;
  dstatus.flags = 0; /* 还没有做任何事情 */

  env = getenv("HWLOC_ALLOW");
  if (env && !strcmp(env, "all"))
    /* 不检索允许资源的集合 */
    # 将 HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES 标志位设置为 dstatus.flags 的值
    dstatus.flags |= HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES;

    # 实例化所有可能的其他后端
    hwloc_disc_components_enable_others(topology);
    # 现在后端已启用，更新 thissystem 标志和一些回调
    hwloc_backends_is_thissystem(topology);
    hwloc_backends_find_callbacks(topology);
    '''
    现在根据 topology->is_thissystem 和本地操作系统后端提供的内容设置绑定钩子
    '''
    hwloc_set_binding_hooks(topology);

    # 实际的拓扑发现
    err = hwloc_discover(topology, &dstatus);
    if (err < 0)
        goto out;
#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG，则检查环境变量 HWLOC_DEBUG_CHECK 是否存在
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 检查拓扑结构的一致性
    hwloc_topology_check(topology);

  /* Rank cpukinds */
  // 对 CPU 种类进行排序
  hwloc_internal_cpukinds_rank(topology);

  /* Mark distances objs arrays as invalid since we may have removed objects
   * from the topology after adding the distances (remove_empty, etc).
   * It would be hard to actually verify whether it's needed.
   */
  // 标记距离对象数组为无效，因为在添加距离后可能已经从拓扑结构中移除了对象
  hwloc_internal_distances_invalidate_cached_objs(topology);
  /* And refresh distances so that multithreaded concurrent distances_get()
   * don't refresh() concurrently (disallowed).
   */
  // 刷新距离信息，以防止多线程并发获取距离信息时出现并发刷新
  hwloc_internal_distances_refresh(topology);

  /* Same for memattrs */
  // 对内存属性进行刷新
  hwloc_internal_memattrs_need_refresh(topology);
  hwloc_internal_memattrs_refresh(topology);
  hwloc_internal_memattrs_guess_memory_tiers(topology);

  // 标记拓扑结构已加载
  topology->is_loaded = 1;

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    /* FIXME: filter directly in backends during the discovery.
     * Only x86 does it because binding may cause issues on Windows.
     */
    // 如果拓扑结构标记需要限制到 CPU 绑定
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    if (set) {
      // 获取当前 CPU 绑定
      err = hwloc_get_cpubind(topology, set, HWLOC_CPUBIND_STRICT);
      if (!err)
        // 限制拓扑结构到指定的 CPU 绑定
        hwloc_topology_restrict(topology, set, 0);
      hwloc_bitmap_free(set);
    }
  }
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING) {
    /* FIXME: filter directly in backends during the discovery.
     */
    // 如果拓扑结构标记需要限制到内存绑定
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    hwloc_membind_policy_t policy;
    if (set) {
      // 获取当前内存绑定
      err = hwloc_get_membind(topology, set, &policy, HWLOC_MEMBIND_STRICT | HWLOC_MEMBIND_BYNODESET);
      if (!err)
        // 限制拓扑结构到指定的内存绑定
        hwloc_topology_restrict(topology, set, HWLOC_RESTRICT_FLAG_BYNODESET);
      hwloc_bitmap_free(set);
    }
  }

  // 如果拓扑结构的后端阶段包含 TWEAK 阶段
  if (topology->backend_phases & HWLOC_DISC_PHASE_TWEAK) {
    // 设置发现状态为 TWEAK 阶段
    dstatus.phase = HWLOC_DISC_PHASE_TWEAK;
    # 通过指定的阶段来发现硬件拓扑结构，并将结果存储在topology中
    hwloc_discover_by_phase(topology, &dstatus, "TWEAK");
  }

  # 返回0，表示成功执行
  return 0;

 out:
  # 退出PCI设备的发现模式
  hwloc_pci_discovery_exit(topology);
  # 清除拓扑结构
  hwloc_topology_clear(topology);
  # 设置默认的拓扑结构
  hwloc_topology_setup_defaults(topology);
  # 禁用所有的后端
  hwloc_backends_disable_all(topology);
  # 返回-1，表示执行失败
  return -1;
/* 根据给定的 droppedcpuset 调整对象 cpusets，
 * 删除 cpuset 变为空并且没有子对象的对象，
 * 并且将 NUMA 节点的移除传播为父对象中 nodeset 的变化。
 */
static void
restrict_object_by_cpuset(hwloc_topology_t topology, unsigned long flags, hwloc_obj_t *pobj,
              hwloc_bitmap_t droppedcpuset, hwloc_bitmap_t droppednodeset)
{
  hwloc_obj_t obj = *pobj, child, *pchild;
  int modified = 0;

  if (hwloc_bitmap_intersects(obj->complete_cpuset, droppedcpuset)) {
    hwloc_bitmap_andnot(obj->cpuset, obj->cpuset, droppedcpuset);
    hwloc_bitmap_andnot(obj->complete_cpuset, obj->complete_cpuset, droppedcpuset);
    modified = 1;
  } else {
    if ((flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS)
    && hwloc_bitmap_iszero(obj->complete_cpuset)) {
      /* 我们是空的，我们下面有一个 NUMAnode，这次它将被移除 */
      modified = 1;
    }
    /* nodeset 不能相交，除非 cpuset 相交或为空 */
    if (droppednodeset)
      assert(!hwloc_bitmap_intersects(obj->complete_nodeset, droppednodeset)
         || hwloc_bitmap_iszero(obj->complete_cpuset));
  }
  if (droppednodeset) {
    hwloc_bitmap_andnot(obj->nodeset, obj->nodeset, droppednodeset);
    hwloc_bitmap_andnot(obj->complete_nodeset, obj->complete_nodeset, droppednodeset);
  }

  if (modified) {
    for_each_child_safe(child, obj, pchild)
      restrict_object_by_cpuset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* 如果一些 hwloc_bitmap_first(child->complete_cpuset) 改变了，子对象可能需要重新排序 */
    hwloc__reorder_children(obj);

    for_each_memory_child_safe(child, obj, pchild)
      restrict_object_by_cpuset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* 本地 NUMA 节点具有相同的 cpusets，不需要重新排序它们 */
    /* Nothing to restrict under I/O or Misc */
  }

  # 如果对象没有子对象，并且内存中也没有子对象，且 cpuset 为空
  # 或者对象类型不是 NUMANODE，或者标志包含 HWLOC_RESTRICT_FLAG_REMOVE_CPULESS
  if (!obj->first_child && !obj->memory_first_child /* arity not updated before connect_children() */
      && hwloc_bitmap_iszero(obj->cpuset)
      && (obj->type != HWLOC_OBJ_NUMANODE || (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS))) {
    # 移除对象
    hwloc_debug("%s", "\nRemoving object during restrict by cpuset");
    hwloc_debug_print_object(0, obj);

    # 如果标志不包含 HWLOC_RESTRICT_FLAG_ADAPT_IO
    # 释放 I/O 子对象及其子对象
    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_IO)) {
      hwloc_free_object_siblings_and_children(obj->io_first_child);
      obj->io_first_child = NULL;
    }
    # 如果标志不包含 HWLOC_RESTRICT_FLAG_ADAPT_MISC
    # 释放 Misc 子对象及其子对象
    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_MISC)) {
      hwloc_free_object_siblings_and_children(obj->misc_first_child);
      obj->misc_first_child = NULL;
    }
    # 断言对象没有子对象
    assert(!obj->first_child);
    # 断言对象的内存中没有子对象
    assert(!obj->memory_first_child);
    # 断开并释放单个对象
    unlink_and_free_single_object(pobj);
    # 标记拓扑结构已修改
    topology->modified = 1;
  }
/* 根据给定的 droppednodeset 调整对象节点集，
 * 删除节点集变为空且没有子节点的对象，
 * 并根据父节点的 cpuset 更改传播 PU 的移除。
 */
static void
restrict_object_by_nodeset(hwloc_topology_t topology, unsigned long flags, hwloc_obj_t *pobj,
               hwloc_bitmap_t droppedcpuset, hwloc_bitmap_t droppednodeset)
{
  hwloc_obj_t obj = *pobj, child, *pchild;
  int modified = 0;

  if (hwloc_bitmap_intersects(obj->complete_nodeset, droppednodeset)) {
    hwloc_bitmap_andnot(obj->nodeset, obj->nodeset, droppednodeset);
    hwloc_bitmap_andnot(obj->complete_nodeset, obj->complete_nodeset, droppednodeset);
    modified = 1;
  } else {
    if ((flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
    && hwloc_bitmap_iszero(obj->complete_nodeset)) {
      /* 我们是空的，下面有一个 PU，这次它将被移除 */
      modified = 1;
    }
    /* cpuset 不能相交，除非 nodeset 相交或为空 */
    if (droppedcpuset)
      assert(!hwloc_bitmap_intersects(obj->complete_cpuset, droppedcpuset)
         || hwloc_bitmap_iszero(obj->complete_nodeset));
  }
  if (droppedcpuset) {
    hwloc_bitmap_andnot(obj->cpuset, obj->cpuset, droppedcpuset);
    hwloc_bitmap_andnot(obj->complete_cpuset, obj->complete_cpuset, droppedcpuset);
  }

  if (modified) {
    for_each_child_safe(child, obj, pchild)
      restrict_object_by_nodeset(topology, flags, pchild, droppedcpuset, droppednodeset);
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
      /* 在上面可能已经改变了 cpuset，一些 NUMA 节点被移除。
       * 如果一些 hwloc_bitmap_first(child->complete_cpuset) 改变了，子节点可能需要重新排序 */
      hwloc__reorder_children(obj);

    for_each_memory_child_safe(child, obj, pchild)
      restrict_object_by_nodeset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* FIXME: 如果其中一些节点被移除，我们可能需要重新排序没有 CPU 的 NUMA 节点组 */
    /* Nothing to restrict under I/O or Misc */
  }

  if (!obj->first_child && !obj->memory_first_child /* arity not updated before connect_children() */
      && hwloc_bitmap_iszero(obj->nodeset)
      && (obj->type != HWLOC_OBJ_PU || (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS))) {
    /* remove object */
    hwloc_debug("%s", "\nRemoving object during restrict by nodeset");
    hwloc_debug_print_object(0, obj);

    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_IO)) {
      /* 如果不需要调整 I/O，释放对象的 I/O 子对象和孩子 */
      hwloc_free_object_siblings_and_children(obj->io_first_child);
      obj->io_first_child = NULL;
    }
    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_MISC)) {
      /* 如果不需要调整 Misc，释放对象的 Misc 子对象和孩子 */
      hwloc_free_object_siblings_and_children(obj->misc_first_child);
      obj->misc_first_child = NULL;
    }
    assert(!obj->first_child);
    assert(!obj->memory_first_child);
    /* 断开并释放单个对象 */
    unlink_and_free_single_object(pobj);
    topology->modified = 1;
  }
  // 限制拓扑结构，根据给定的位图和标志
int
hwloc_topology_restrict(struct hwloc_topology *topology, hwloc_const_bitmap_t set, unsigned long flags)
{
  hwloc_bitmap_t droppedcpuset, droppednodeset;  // 定义两个位图指针

  if (!topology->is_loaded) {  // 如果拓扑结构未加载
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }
  if (topology->adopted_shmem_addr) {  // 如果拓扑结构采用了共享内存地址
    errno = EPERM;  // 设置错误码为操作不允许
    return -1;  // 返回错误
  }

  if (flags & ~(HWLOC_RESTRICT_FLAG_REMOVE_CPULESS
        |HWLOC_RESTRICT_FLAG_ADAPT_MISC|HWLOC_RESTRICT_FLAG_ADAPT_IO
        |HWLOC_RESTRICT_FLAG_BYNODESET|HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)) {
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  if (flags & HWLOC_RESTRICT_FLAG_BYNODESET) {  // 如果标志包含 BYNODESET
    /* cannot use CPULESS with BYNODESET */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS) {  // 如果标志包含 REMOVE_CPULESS
      errno = EINVAL;  // 设置错误码为无效参数
      return -1;  // 返回错误
    }
  } else {
    /* cannot use MEMLESS without BYNODESET */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS) {  // 如果标志包含 REMOVE_MEMLESS
      errno = EINVAL;  // 设置错误码为无效参数
      return -1;  // 返回错误
    }
  }

  /* make sure we'll keep something in the topology */
  if (((flags & HWLOC_RESTRICT_FLAG_BYNODESET) && !hwloc_bitmap_intersects(set, topology->allowed_nodeset))
      || (!(flags & HWLOC_RESTRICT_FLAG_BYNODESET) && !hwloc_bitmap_intersects(set, topology->allowed_cpuset))) {
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  droppedcpuset = hwloc_bitmap_alloc();  // 分配一个位图用于存储被移除的 cpuset
  droppednodeset = hwloc_bitmap_alloc();  // 分配一个位图用于存储被移除的 nodeset
  if (!droppedcpuset || !droppednodeset) {  // 如果分配失败
    hwloc_bitmap_free(droppedcpuset);  // 释放已分配的位图
    hwloc_bitmap_free(droppednodeset);  // 释放已分配的位图
    return -1;  // 返回错误
  }

  if (flags & HWLOC_RESTRICT_FLAG_BYNODESET) {  // 如果标志包含 BYNODESET
    /* nodeset to clear */
    hwloc_bitmap_not(droppednodeset, set);  // 计算需要清除的 nodeset
    /* cpuset to clear */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS) {  // 如果标志包含 REMOVE_MEMLESS
      hwloc_obj_t pu = hwloc_get_obj_by_type(topology, HWLOC_OBJ_PU, 0);  // 获取处理器对象
      assert(pu);  // 断言处理器对象存在
      do {
    /* PU will be removed if cpuset gets or was empty */
    if (hwloc_bitmap_iszero(pu->cpuset)  // 如果处理器的 cpuset 为空
        || hwloc_bitmap_isincluded(pu->nodeset, droppednodeset))  // 或者处理器的 nodeset 包含在需要清除的 nodeset 中
      hwloc_bitmap_set(droppedcpuset, pu->os_index);  // 设置需要清除的 cpuset
    pu = pu->next_cousin;  # 将指针移动到下一个同级对象
  } while (pu);  # 循环直到指针为空

  /* 检查是否移除了所有的处理器单元 */
  if (hwloc_bitmap_isincluded(topology->allowed_cpuset, droppedcpuset)) {  # 检查是否移除了所有允许的处理器单元
    errno = EINVAL;  # 设置错误码为无效参数，不修改拓扑结构
    hwloc_bitmap_free(droppedcpuset);  # 释放 droppedcpuset 的内存
    hwloc_bitmap_free(droppednodeset);  # 释放 droppednodeset 的内存
    return -1;  # 返回错误码
  }
}

/* 如果标志中包含 HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS 或者 droppedcpuset 为空，则移除 cpuset */
if (!(flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
|| hwloc_bitmap_iszero(droppedcpuset)) {
  hwloc_bitmap_free(droppedcpuset);  # 释放 droppedcpuset 的内存
  droppedcpuset = NULL;  # 将 droppedcpuset 设置为空
}

/* 现在递归过滤集合并丢弃内容 */
restrict_object_by_nodeset(topology, flags, &topology->levels[0][0], droppedcpuset, droppednodeset);  # 递归过滤集合并丢弃内容
hwloc_bitmap_andnot(topology->allowed_nodeset, topology->allowed_nodeset, droppednodeset);  # 对 topology->allowed_nodeset 进行按位非操作
if (droppedcpuset)
  hwloc_bitmap_andnot(topology->allowed_cpuset, topology->allowed_cpuset, droppedcpuset);  # 对 topology->allowed_cpuset 进行按位非操作

} else {
  /* 要清除的 cpuset */
  hwloc_bitmap_not(droppedcpuset, set);  # 对 droppedcpuset 进行按位非操作
  /* 要清除的 nodeset */
  if (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS) {  # 如果标志中包含 HWLOC_RESTRICT_FLAG_REMOVE_CPULESS
    hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, 0);  # 获取拓扑结构中的 NUMA 节点对象
    assert(node);  # 断言 node 不为空
    do {
      /* 如果 nodeset 为空或者包含在 droppedcpuset 中，则将节点移除 */
      if (hwloc_bitmap_iszero(node->cpuset)
          || hwloc_bitmap_isincluded(node->cpuset, droppedcpuset))
        hwloc_bitmap_set(droppednodeset, node->os_index);  # 将节点的索引设置到 droppednodeset 中
      node = node->next_cousin;  # 将指针移动到下一个同级对象
    } while (node);

    /* 检查是否移除了所有的 NUMA 节点 */
    if (hwloc_bitmap_isincluded(topology->allowed_nodeset, droppednodeset)) {  # 检查是否移除了所有允许的 NUMA 节点
      errno = EINVAL;  # 设置错误码为无效参数，不修改拓扑结构
      hwloc_bitmap_free(droppedcpuset);  # 释放 droppedcpuset 的内存
      hwloc_bitmap_free(droppednodeset);  # 释放 droppednodeset 的内存
      return -1;  # 返回错误码
    }
  }
  /* 如果标志中包含 HWLOC_RESTRICT_FLAG_REMOVE_CPULESS 或者 droppednodeset 为空，则移除 nodeset */
  if (!(flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS)
  || hwloc_bitmap_iszero(droppednodeset)) {
    hwloc_bitmap_free(droppednodeset);  # 释放 droppednodeset 的内存
    droppednodeset = NULL;  # 将 droppednodeset 设置为空
  }
    /* 现在递归过滤集合并丢弃内容 */
    // 通过 CPU 集合限制对象
    restrict_object_by_cpuset(topology, flags, &topology->levels[0][0], droppedcpuset, droppednodeset);
    // 从允许的 CPU 集合中去除被丢弃的 CPU
    hwloc_bitmap_andnot(topology->allowed_cpuset, topology->allowed_cpuset, droppedcpuset);
    // 如果存在被丢弃的节点集合，则从允许的节点集合中去除
    if (droppednodeset)
      hwloc_bitmap_andnot(topology->allowed_nodeset, topology->allowed_nodeset, droppednodeset);
  }

  // 释放被丢弃的 CPU 集合
  hwloc_bitmap_free(droppedcpuset);
  // 释放被丢弃的节点集合
  hwloc_bitmap_free(droppednodeset);

  // 如果保持结构的过滤级别小于 0，则跳转到 out 标签
  if (hwloc_filter_levels_keep_structure(topology) < 0) /* takes care of reconnecting internally */
    goto out;

  /* 一些对象可能已经消失，我们需要更新距离对象数组 */
  // 使内部缓存的距离对象无效
  hwloc_internal_distances_invalidate_cached_objs(topology);
  // 需要刷新内存属性
  hwloc_internal_memattrs_need_refresh(topology);

  // 传播对称子树
  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);
  // 传播总内存
  propagate_total_memory(topology->levels[0][0]);
  // 限制 CPU 种类
  hwloc_internal_cpukinds_restrict(topology);
#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG，则检查环境变量 HWLOC_DEBUG_CHECK
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 检查拓扑结构的完整性
    hwloc_topology_check(topology);

  // 返回成功
  return 0;

 out:
  /* unrecoverable failure, re-init the topology */
   // 不可恢复的失败，重新初始化拓扑结构
   hwloc_topology_clear(topology);
   hwloc_topology_setup_defaults(topology);
   // 返回失败
   return -1;
}

int
hwloc_topology_allow(struct hwloc_topology *topology,
             hwloc_const_cpuset_t cpuset, hwloc_const_nodeset_t nodeset,
             unsigned long flags)
{
  // 如果拓扑结构未加载，则跳转到错误处理
  if (!topology->is_loaded)
    goto einval;

  // 如果拓扑结构已采用共享内存地址，则设置错误码并跳转到错误处理
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    goto error;
  }

  // 如果未设置 HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED 标志，则跳转到错误处理
  if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED))
    goto einval;

  // 如果 flags 中包含未知标志位，则跳转到错误处理
  if (flags & ~(HWLOC_ALLOW_FLAG_ALL|HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS|HWLOC_ALLOW_FLAG_CUSTOM))
    goto einval;

  // 根据 flags 的不同取值进行处理
  switch (flags) {
  case HWLOC_ALLOW_FLAG_ALL: {
    // 如果 cpuset 或 nodeset 不为空，则跳转到错误处理
    if (cpuset || nodeset)
      goto einval;
    // 将完整的 CPU 和 NUMA 节点集合复制到允许集合中
    hwloc_bitmap_copy(topology->allowed_cpuset, hwloc_get_root_obj(topology)->complete_cpuset);
    hwloc_bitmap_copy(topology->allowed_nodeset, hwloc_get_root_obj(topology)->complete_nodeset);
    break;
  }
  case HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS: {
    // 如果 cpuset 或 nodeset 不为空，则跳转到错误处理
    if (cpuset || nodeset)
      goto einval;
    // 如果不是本地系统，则跳转到错误处理
    if (!topology->is_thissystem)
      goto einval;
    // 如果没有获取允许资源的钩子函数，则设置错误码并跳转到错误处理
    if (!topology->binding_hooks.get_allowed_resources) {
      errno = ENOSYS;
      goto error;
    }
    // 调用获取允许资源的钩子函数
    topology->binding_hooks.get_allowed_resources(topology);
    /* make sure the backend returned something sane (Linux cpusets may return offline PUs in some cases) */
    // 确保后端返回了合理的内容（在某些情况下，Linux cpusets 可能返回离线的处理器）
    hwloc_bitmap_and(topology->allowed_cpuset, topology->allowed_cpuset, hwloc_get_root_obj(topology)->cpuset);
    hwloc_bitmap_and(topology->allowed_nodeset, topology->allowed_nodeset, hwloc_get_root_obj(topology)->nodeset);
    break;
  }
  case HWLOC_ALLOW_FLAG_CUSTOM: {
    if (cpuset) {
      /* keep the intersection with the full topology cpuset, if not empty */
      // 如果不为空，则保留与完整拓扑结构 cpuset 的交集
      if (!hwloc_bitmap_intersects(hwloc_get_root_obj(topology)->cpuset, cpuset))
    # 跳转到错误处理标签 einval
    goto einval;
      # 将 cpuset 与根对象的 cpuset 进行与操作，并将结果存储到 topology->allowed_cpuset 中
      hwloc_bitmap_and(topology->allowed_cpuset, hwloc_get_root_obj(topology)->cpuset, cpuset);
    }
    if (nodeset) {
      # 如果 nodeset 不为空，则保留与完整拓扑结构的 nodeset 的交集
      if (!hwloc_bitmap_intersects(hwloc_get_root_obj(topology)->nodeset, nodeset))
    # 跳转到错误处理标签 einval
    goto einval;
      # 将 nodeset 与根对象的 nodeset 进行与操作，并将结果存储到 topology->allowed_nodeset 中
      hwloc_bitmap_and(topology->allowed_nodeset, hwloc_get_root_obj(topology)->nodeset, nodeset);
    }
    break;
  }
  default:
    # 跳转到错误处理标签 einval
    goto einval;
  }

  # 返回成功
  return 0;

 einval:
  # 设置错误码为 EINVAL
  errno = EINVAL;
 error:
  # 返回失败
  return -1;
}

int
hwloc_topology_refresh(struct hwloc_topology *topology)
{
  // 刷新拓扑结构中的 CPU 种类信息
  hwloc_internal_cpukinds_rank(topology);
  // 刷新拓扑结构中的距离信息
  hwloc_internal_distances_refresh(topology);
  // 刷新拓扑结构中的内存属性信息
  hwloc_internal_memattrs_refresh(topology);
  // 返回成功
  return 0;
}

int
hwloc_topology_is_thissystem(struct hwloc_topology *topology)
{
  // 返回拓扑结构中是否代表当前系统
  return topology->is_thissystem;
}

int
hwloc_topology_get_depth(struct hwloc_topology *topology)
{
  // 返回拓扑结构中的层级深度
  return (int) topology->nb_levels;
}

const struct hwloc_topology_support *
hwloc_topology_get_support(struct hwloc_topology * topology)
{
  // 返回拓扑结构的支持信息
  return &topology->support;
}

void hwloc_topology_set_userdata(struct hwloc_topology * topology, const void *userdata)
{
  // 设置拓扑结构的用户数据
  topology->userdata = (void *) userdata;
}

void * hwloc_topology_get_userdata(struct hwloc_topology * topology)
{
  // 获取拓扑结构的用户数据
  return topology->userdata;
}

hwloc_const_cpuset_t
hwloc_topology_get_complete_cpuset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的完整 CPU 集合
  return hwloc_get_root_obj(topology)->complete_cpuset;
}

hwloc_const_cpuset_t
hwloc_topology_get_topology_cpuset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的拓扑 CPU 集合
  return hwloc_get_root_obj(topology)->cpuset;
}

hwloc_const_cpuset_t
hwloc_topology_get_allowed_cpuset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的允许使用的 CPU 集合
  return topology->allowed_cpuset;
}

hwloc_const_nodeset_t
hwloc_topology_get_complete_nodeset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的完整节点集合
  return hwloc_get_root_obj(topology)->complete_nodeset;
}

hwloc_const_nodeset_t
hwloc_topology_get_topology_nodeset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的拓扑节点集合
  return hwloc_get_root_obj(topology)->nodeset;
}

hwloc_const_nodeset_t
hwloc_topology_get_allowed_nodeset(hwloc_topology_t topology)
{
  // 返回拓扑结构中的允许使用的节点集合
  return topology->allowed_nodeset;
}


/****************
 * Debug Checks *
 ****************/

#ifndef NDEBUG /* assert only enabled if !NDEBUG */

static void
hwloc__check_child_siblings(hwloc_obj_t parent, hwloc_obj_t *array,
                unsigned arity, unsigned i,
                hwloc_obj_t child, hwloc_obj_t prev)
{
  // 检查子对象的父对象是否正确
  assert(child->parent == parent);

  // 检查子对象的兄弟排名是否正确
  assert(child->sibling_rank == i);
  if (array)
    # 检查子节点是否等于数组中对应位置的元素
    assert(child == array[i]);

  # 如果存在前一个节点
    assert(prev)
    # 检查前一个节点的下一个兄弟节点是否等于当前子节点
    assert(prev->next_sibling == child);
  # 检查当前子节点的前一个兄弟节点是否等于前一个节点
    assert(child->prev_sibling == prev);

  # 如果当前位置为0
  if (!i)
    # 检查当前子节点的前一个兄弟节点是否为空
    assert(child->prev_sibling == NULL);
  else
    # 检查当前子节点的前一个兄弟节点是否不为空
    assert(child->prev_sibling != NULL);

  # 如果当前位置为数组长度减1
  if (i == arity-1)
    # 检查当前子节点的下一个兄弟节点是否为空
    assert(child->next_sibling == NULL);
  else
    # 检查当前子节点的下一个兄弟节点是否不为空
    assert(child->next_sibling != NULL);
}
# 检查对象的孩子节点
static void
hwloc__check_object(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t obj);

/* 检查父对象之间的孩子节点 */
static void
hwloc__check_normal_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  hwloc_obj_t child, prev;
  unsigned j;

  if (!parent->arity) {
    /* 检查父对象是否真的没有孩子节点 */
    assert(!parent->children);
    assert(!parent->first_child);
    assert(!parent->last_child);
    return;
  }
  /* 检查父对象是否真的有孩子节点 */
  assert(parent->children);
  assert(parent->first_child);
  assert(parent->last_child);

  /* 兄弟节点检查 */
  for(prev = NULL, child = parent->first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    /* 普通孩子节点 */
    assert(hwloc__obj_type_is_normal(child->type));
    /* 检查深度 */
    assert(child->depth > parent->depth);
    /* 检查兄弟节点 */
    hwloc__check_child_siblings(parent, parent->children, parent->arity, j, child, prev);
    /* 递归 */
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* 检查度 */
  assert(j == parent->arity);

  assert(parent->first_child == parent->children[0]);
  assert(parent->last_child == parent->children[parent->arity-1]);

  /* 在 PU 下没有普通孩子节点 */
  if (parent->type == HWLOC_OBJ_PU)
    assert(!parent->arity);
}

static void
hwloc__check_children_cpusets(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj)
{
  /* 我们已经在调用者中检查了对象是否有全部或没有集合 */
  hwloc_obj_t child;
  int prev_first, prev_empty;

  if (obj->type == HWLOC_OBJ_PU) {
    /* PU cpuset 只是它自己，没有普通孩子节点 */
    assert(hwloc_bitmap_weight(obj->cpuset) == 1);
    assert(hwloc_bitmap_first(obj->cpuset) == (int) obj->os_index);
    assert(hwloc_bitmap_weight(obj->complete_cpuset) == 1);
    # 断言对象的完整 CPU 集合中的第一个位与对象的操作系统索引相等
    assert(hwloc_bitmap_first(obj->complete_cpuset) == (int) obj->os_index);
    # 如果拓扑结构的标志不包括禁止的标志
    if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
      # 断言拓扑结构中允许的 CPU 集合中包含对象的操作系统索引
      assert(hwloc_bitmap_isset(topology->allowed_cpuset, (int) obj->os_index));
    }
    # 断言对象的度为 0
    assert(!obj->arity);
  } else if (hwloc__obj_type_is_memory(obj->type)) {
    # 内存对象的 CPU 集合等于其父对象的 CPU 集合
    assert(hwloc_bitmap_isequal(obj->parent->cpuset, obj->cpuset));
    # 断言对象的度为 0
    assert(!obj->arity);
  } else if (!hwloc__obj_type_is_special(obj->type)) {
    hwloc_bitmap_t set;
    # 其他对象的 CPU 集合是正常子对象的异或运算结果，除了处理器单元
    set = hwloc_bitmap_alloc();
    for_each_child(child, obj) {
      # 断言集合与子对象的 CPU 集合没有交集
      assert(!hwloc_bitmap_intersects(set, child->cpuset));
      # 对集合和子对象的 CPU 集合进行或运算
      hwloc_bitmap_or(set, set, child->cpuset);
    }
    # 断言集合与对象的 CPU 集合相等
    assert(hwloc_bitmap_isequal(set, obj->cpuset));
    # 释放集合内存
    hwloc_bitmap_free(set);
  }

  # 检查内存子对象的 CPU 集合是否相等
  for_each_memory_child(child, obj)
    assert(hwloc_bitmap_isequal(obj->cpuset, child->cpuset));

  # 检查子对象的完整 CPU 集合是否正确排序，空集合可以在任何位置
  prev_first = -1; # -1 在下面的比较中可以正常工作
  prev_empty = 0; # 前面的子对象中没有空的 CPU 集合
  for_each_child(child, obj) {
    int first = hwloc_bitmap_first(child->complete_cpuset);
    if (first >= 0) {
      # 断言前面的子对象中没有空的 CPU 集合
      assert(!prev_empty);
      # 断言前一个子对象的第一个位小于当前子对象的第一个位
      assert(prev_first < first);
    } else {
      prev_empty = 1;
    }
    prev_first = first;
  }
# 检查内存子节点，确保其正确性
static void
hwloc__check_memory_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  # 如果父节点没有内存子节点
  if (!parent->memory_arity) {
    # 检查父节点是否真的没有子节点
    assert(!parent->memory_first_child);
    return;
  }
  # 检查父节点是否有内存子节点
  assert(parent->memory_first_child);

  # 遍历内存子节点
  for(prev = NULL, child = parent->memory_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    # 确保子节点类型为内存
    assert(hwloc__obj_type_is_memory(child->type));
    # 检查兄弟节点
    hwloc__check_child_siblings(parent, NULL, parent->memory_arity, j, child, prev);
    # 只有内存和杂项子节点，递归检查
    assert(!child->first_child);
    assert(!child->io_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  # 检查内存子节点的数量
  assert(j == parent->memory_arity);

  # NUMA 节点下没有内存子节点
  if (parent->type == HWLOC_OBJ_NUMANODE)
    assert(!parent->memory_arity);
}

# 检查 I/O 子节点，确保其正确性
static void
hwloc__check_io_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  # 如果父节点没有 I/O 子节点
  if (!parent->io_arity) {
    # 检查父节点是否真的没有子节点
    assert(!parent->io_first_child);
    return;
  }
  # 检查父节点是否有 I/O 子节点
  assert(parent->io_first_child);

  # 遍历 I/O 子节点
  for(prev = NULL, child = parent->io_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    # 确保所有子节点都是 I/O 类型
    assert(hwloc__obj_type_is_io(child->type));
    # 检查兄弟节点
    hwloc__check_child_siblings(parent, NULL, parent->io_arity, j, child, prev);
    # 只有 I/O 和杂项子节点，递归检查
    assert(!child->first_child);
    assert(!child->memory_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  # 检查 I/O 子节点的数量
  assert(j == parent->io_arity);
}
static void
hwloc__check_misc_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  if (!parent->misc_arity) {
    /* 如果父对象的 misc_arity 为 0，则表示父对象没有子对象 */
    assert(!parent->misc_first_child);
    return;
  }
  /* 如果父对象的 misc_arity 不为 0，则表示父对象有子对象 */
  assert(parent->misc_first_child);

  for(prev = NULL, child = parent->misc_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    /* 所有子对象必须是 Misc 类型 */
    assert(child->type == HWLOC_OBJ_MISC);
    /* 检查兄弟对象 */
    hwloc__check_child_siblings(parent, NULL, parent->misc_arity, j, child, prev);
    /* 只能是 Misc 类型的子对象，递归检查 */
    assert(!child->first_child);
    assert(!child->memory_first_child);
    assert(!child->io_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* 检查 misc_arity 是否正确 */
  assert(j == parent->misc_arity);
}

static void
hwloc__check_object(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t obj)
{
  hwloc_uint64_t total_memory;
  hwloc_obj_t child;

  /* 确保 gp_index 在位图中没有被设置过 */
  assert(!hwloc_bitmap_isset(gp_indexes, obj->gp_index));
  hwloc_bitmap_set(gp_indexes, obj->gp_index);

  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  assert((unsigned) obj->type < HWLOC_OBJ_TYPE_MAX);

  assert(hwloc_filter_check_keep_object(topology, obj));

  /* 检查对象的集合和深度 */
  if (hwloc__obj_type_is_special(obj->type)) {
    assert(!obj->cpuset);
    if (obj->type == HWLOC_OBJ_BRIDGE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_BRIDGE);
    else if (obj->type == HWLOC_OBJ_PCI_DEVICE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_PCI_DEVICE);
    else if (obj->type == HWLOC_OBJ_OS_DEVICE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_OS_DEVICE);
    else if (obj->type == HWLOC_OBJ_MISC)
      assert(obj->depth == HWLOC_TYPE_DEPTH_MISC);
  } else {
    assert(obj->cpuset);
  }
}
  # 如果对象类型是 NUMANODE，则断言其深度为 NUMANODE 类型的深度
  if (obj->type == HWLOC_OBJ_NUMANODE)
    assert(obj->depth == HWLOC_TYPE_DEPTH_NUMANODE);
  # 如果对象类型是 MEMCACHE，则断言其深度为 MEMCACHE 类型的深度
  else if (obj->type == HWLOC_OBJ_MEMCACHE)
    assert(obj->depth == HWLOC_TYPE_DEPTH_MEMCACHE);
  # 否则断言对象的深度大于等于 0
  else
    assert(obj->depth >= 0);
  }

  # 在 v2.0+ 版本中，组对象的深度不能再是 -1
  if (obj->type == HWLOC_OBJ_GROUP) {
    assert(obj->attr->group.depth != (unsigned) -1);
  }

  # 只有当存在主 cpuset 时，才会存在其他 cpusets 和 nodesets
  assert(!!obj->cpuset == !!obj->complete_cpuset);
  assert(!!obj->cpuset == !!obj->nodeset);
  assert(!!obj->nodeset == !!obj->complete_nodeset);

  # 检查完整/内联集合是否比主集合大
  if (obj->cpuset) {
    assert(hwloc_bitmap_isincluded(obj->cpuset, obj->complete_cpuset));
    assert(hwloc_bitmap_isincluded(obj->nodeset, obj->complete_nodeset));
  }

  # 检查缓存类型/深度与类型是否匹配
  if (hwloc__obj_type_is_cache(obj->type)) {
    if (hwloc__obj_type_is_icache(obj->type))
      assert(obj->attr->cache.type == HWLOC_OBJ_CACHE_INSTRUCTION);
    else if (hwloc__obj_type_is_dcache(obj->type))
      assert(obj->attr->cache.type == HWLOC_OBJ_CACHE_DATA
         || obj->attr->cache.type == HWLOC_OBJ_CACHE_UNIFIED);
    else
      assert(0);
    assert(hwloc_cache_type_by_depth_type(obj->attr->cache.depth, obj->attr->cache.type) == obj->type);
  }

  # 检查总内存
  total_memory = 0;
  if (obj->type == HWLOC_OBJ_NUMANODE)
    total_memory += obj->attr->numanode.local_memory;
  for_each_child(child, obj) {
    total_memory += child->total_memory;
  }
  for_each_memory_child(child, obj) {
    // 将子对象的总内存大小累加到父对象的总内存大小中
    total_memory += child->total_memory;
  }
  // 断言父对象的总内存大小等于累加的子对象的总内存大小
  assert(total_memory == obj->total_memory);

  /* 检查普通子对象 */
  hwloc__check_normal_children(topology, gp_indexes, obj);
  /* 检查内存子对象 */
  hwloc__check_memory_children(topology, gp_indexes, obj);
  /* 检查 I/O 子对象 */
  hwloc__check_io_children(topology, gp_indexes, obj);
  /* 检查杂项子对象 */
  hwloc__check_misc_children(topology, gp_indexes, obj);
  /* 检查子对象的 CPU 集合 */
  hwloc__check_children_cpusets(topology, obj);
  /* 节点集在另一个递归状态下进行检查 */
    # 检查节点集合，确保节点对象的节点集合和完整节点集合的正确性
    static void
    hwloc__check_nodesets(hwloc_topology_t topology, hwloc_obj_t obj, hwloc_bitmap_t parentset)
    {
      hwloc_obj_t child;
      int prev_first;

      # 如果对象类型是 NUMANODE
      if (obj->type == HWLOC_OBJ_NUMANODE) {
        # NUMANODE 节点集合只包含自身，没有内存/普通子节点
        assert(hwloc_bitmap_weight(obj->nodeset) == 1);
        assert(hwloc_bitmap_first(obj->nodeset) == (int) obj->os_index);
        assert(hwloc_bitmap_weight(obj->complete_nodeset) == 1);
        assert(hwloc_bitmap_first(obj->complete_nodeset) == (int) obj->os_index);
        # 如果拓扑结构标记不包含不允许的节点，则断言节点在允许的节点集合中
        if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
          assert(hwloc_bitmap_isset(topology->allowed_nodeset, (int) obj->os_index));
        }
        assert(!obj->arity);
        assert(!obj->memory_arity);
        # 断言节点集合包含在父节点集合中
        assert(hwloc_bitmap_isincluded(obj->nodeset, parentset));
      } else {
        hwloc_bitmap_t myset;
        hwloc_bitmap_t childset;

        # 本地节点集合是内存子节点的异或运算结果
        myset = hwloc_bitmap_alloc();
        for_each_memory_child(child, obj) {
          assert(!hwloc_bitmap_intersects(myset, child->nodeset));
          hwloc_bitmap_or(myset, myset, child->nodeset);
        }
        # 本地节点集合不能与父节点的本地节点集合相交
        assert(!hwloc_bitmap_intersects(myset, parentset));
        hwloc_bitmap_or(parentset, parentset, myset);
        hwloc_bitmap_free(myset);
        # 父节点集合现在包含父节点和本地节点的贡献

        # 对于每个子节点，递归检查/获取其贡献
        childset = hwloc_bitmap_alloc();
        for_each_child(child, obj) {
          hwloc_bitmap_t set = hwloc_bitmap_dup(parentset); # 不要触碰 parentset，我们不希望将第一个子节点的贡献传播给其他子节点
          hwloc__check_nodesets(topology, child, set);
          # 提取这个子节点的贡献
          hwloc_bitmap_andnot(set, set, parentset);
          # 保存它
          assert(!hwloc_bitmap_intersects(childset, set));
          hwloc_bitmap_or(childset, childset, set);
          hwloc_bitmap_free(set);
        }
    /* 将子节点的贡献合并到父节点集合中 */
    assert(!hwloc_bitmap_intersects(parentset, childset));  // 确保父节点集合和子节点集合没有交集
    hwloc_bitmap_or(parentset, parentset, childset);  // 将子节点集合合并到父节点集合中
    hwloc_bitmap_free(childset);  // 释放子节点集合的内存
    /* 现在检查我们的节点集合是否由父节点、本地节点和子节点组成 */
    assert(hwloc_bitmap_isequal(obj->nodeset, parentset));  // 确保节点集合由父节点、本地节点和子节点组成

  }

  /* 检查子节点的完整节点集合是否正确排序，空节点可以出现在任何位置
   * （对于主节点集合可能不正确，因为移除的处理器单元可能会破坏排序）。
   */
  prev_first = -1; /* -1 在下面的比较中可以正常工作 */
  for_each_memory_child(child, obj) {
    int first = hwloc_bitmap_first(child->complete_nodeset);  // 获取子节点的完整节点集合中第一个位的索引
    assert(prev_first < first);  // 确保前一个子节点的完整节点集合中第一个位的索引小于当前子节点的完整节点集合中第一个位的索引
    prev_first = first;  // 更新前一个子节点的完整节点集合中第一个位的索引
  }
# 检查给定层级的对象是否正确放置和配置
static void
hwloc__check_level(struct hwloc_topology *topology, int depth,
           hwloc_obj_t first, hwloc_obj_t last)
{
  # 获取给定层级的对象数量
  unsigned width = hwloc_get_nbobjs_by_depth(topology, depth);
  # 初始化前一个对象为 NULL
  struct hwloc_obj *prev = NULL;
  # 初始化对象
  hwloc_obj_t obj;
  unsigned j;

  # 检查该层级的每个对象
  for(j=0; j<width; j++) {
    # 获取给定层级和索引的对象
    obj = hwloc_get_obj_by_depth(topology, depth, j);
    # 检查对象是否存在
    assert(obj);
    # 检查对象的深度是否正确
    assert(obj->depth == depth);
    # 检查对象的逻辑索引是否正确
    assert(obj->logical_index == j);
    # 检查该层级的所有对象是否具有相同的类型
    if (prev) {
      assert(hwloc_type_cmp(obj, prev) == HWLOC_OBJ_EQUAL);
      assert(prev->next_cousin == obj);
    }
    assert(obj->prev_cousin == prev);

    # 检查 PUs 和 NUMA 节点是否具有正确的 cpuset/nodeset
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      assert(hwloc_bitmap_weight(obj->complete_nodeset) == 1);
      assert(hwloc_bitmap_first(obj->complete_nodeset) == (int) obj->os_index);
    }
    prev = obj;
  }
  if (prev)
    assert(prev->next_cousin == NULL);

  if (width) {
    # 检查该层级的第一个对象
    obj = hwloc_get_obj_by_depth(topology, depth, 0);
    assert(obj);
    assert(!obj->prev_cousin);
    # 检查类型
    assert(hwloc_get_depth_type(topology, depth) == obj->type);
    assert(depth == hwloc_get_type_depth(topology, obj->type)
       || HWLOC_TYPE_DEPTH_MULTIPLE == hwloc_get_type_depth(topology, obj->type));
    # 检查该层级的最后一个对象
    obj = hwloc_get_obj_by_depth(topology, depth, width-1);
    assert(obj);
    assert(!obj->next_cousin);
  }

  if (depth < 0) {
    assert(first == hwloc_get_obj_by_depth(topology, depth, 0));
    assert(last == hwloc_get_obj_by_depth(topology, depth, width-1));
  } else {
    assert(!first);
    assert(!last);
  }

  # 检查该层级的最后一个对象+1
  obj = hwloc_get_obj_by_depth(topology, depth, width);
  assert(!obj);
}
/* 检查整个拓扑结构 */
void
hwloc_topology_check(struct hwloc_topology *topology)
    // 确保对象类型的顺序是正确的
    assert(obj_order_type[obj_type_order[type]] == type);
  for(i=HWLOC_OBJ_TYPE_MIN; i<HWLOC_OBJ_TYPE_MAX; i++)
    // 确保对象类型的顺序是正确的
    assert(obj_type_order[obj_order_type[i]] == i);

  // 获取拓扑结构的深度
  depth = hwloc_topology_get_depth(topology);

  // 确保拓扑结构没有被修改
  assert(!topology->modified);

  /* 检查第一层是否是 Machine。
   * 根对象不能被忽略。Machine 只能合并到 PU 中，
   * 但是 Machine 下必须有 NUMA 节点，并且不能在 PU 下面。
   */
  assert(hwloc_get_depth_type(topology, 0) == HWLOC_OBJ_MACHINE);

  /* 检查最后一层是否是 PU，并且它没有内存 */
  assert(hwloc_get_depth_type(topology, depth-1) == HWLOC_OBJ_PU);
  assert(hwloc_get_nbobjs_by_depth(topology, depth-1) > 0);
  for(i=0; i<hwloc_get_nbobjs_by_depth(topology, depth-1); i++) {
    obj = hwloc_get_obj_by_depth(topology, depth-1, i);
    assert(obj);
    assert(obj->type == HWLOC_OBJ_PU);
    assert(!obj->memory_first_child);
  }
  /* 检查其他层是否不是 PU 或 Machine */
  for(j=1; j<depth-1; j++) {
    assert(hwloc_get_depth_type(topology, j) != HWLOC_OBJ_PU);
    assert(hwloc_get_depth_type(topology, j) != HWLOC_OBJ_MACHINE);
  }

  /* 检查正常的层 */
  for(j=0; j<depth; j++) {
    int d;
    type = hwloc_get_depth_type(topology, j);
    assert(type != HWLOC_OBJ_NUMANODE);
    assert(type != HWLOC_OBJ_MEMCACHE);
    assert(type != HWLOC_OBJ_PCI_DEVICE);
    assert(type != HWLOC_OBJ_BRIDGE);
    assert(type != HWLOC_OBJ_OS_DEVICE);
    assert(type != HWLOC_OBJ_MISC);
    d = hwloc_get_type_depth(topology, type);
    assert(d == j || d == HWLOC_TYPE_DEPTH_MULTIPLE);
  }

  /* 检查类型深度，即使没有这样的层 */
  for(type=HWLOC_OBJ_TYPE_MIN; type<HWLOC_OBJ_TYPE_MAX; type++) {
    int d;
    d = hwloc_get_type_depth(topology, type);
    # 如果类型是 NUMANODE
    if (type == HWLOC_OBJ_NUMANODE) {
      # 确保深度等于 NUMANODE
      assert(d == HWLOC_TYPE_DEPTH_NUMANODE);
      # 确保获取的深度类型是 NUMANODE
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_NUMANODE);
    } 
    # 如果类型是 MEMCACHE
    else if (type == HWLOC_OBJ_MEMCACHE) {
      # 确保深度等于 MEMCACHE
      assert(d == HWLOC_TYPE_DEPTH_MEMCACHE);
      # 确保获取的深度类型是 MEMCACHE
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_MEMCACHE);
    } 
    # 如果类型是 BRIDGE
    else if (type == HWLOC_OBJ_BRIDGE) {
      # 确保深度等于 BRIDGE
      assert(d == HWLOC_TYPE_DEPTH_BRIDGE);
      # 确保获取的深度类型是 BRIDGE
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_BRIDGE);
    } 
    # 如果类型是 PCI_DEVICE
    else if (type == HWLOC_OBJ_PCI_DEVICE) {
      # 确保深度等于 PCI_DEVICE
      assert(d == HWLOC_TYPE_DEPTH_PCI_DEVICE);
      # 确保获取的深度类型是 PCI_DEVICE
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_PCI_DEVICE);
    } 
    # 如果类型是 OS_DEVICE
    else if (type == HWLOC_OBJ_OS_DEVICE) {
      # 确保深度等于 OS_DEVICE
      assert(d == HWLOC_TYPE_DEPTH_OS_DEVICE);
      # 确保获取的深度类型是 OS_DEVICE
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_OS_DEVICE);
    } 
    # 如果类型是 MISC
    else if (type == HWLOC_OBJ_MISC) {
      # 确保深度等于 MISC
      assert(d == HWLOC_TYPE_DEPTH_MISC);
      # 确保获取的深度类型是 MISC
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_MISC);
    } 
    # 如果类型不是以上任何一种
    else {
      # 确保深度大于等于 0 或者等于 UNKNOWN 或者等于 MULTIPLE
      assert(d >=0 || d == HWLOC_TYPE_DEPTH_UNKNOWN || d == HWLOC_TYPE_DEPTH_MULTIPLE);
    }
  }

  # 顶层特定检查
  # 确保深度为 0 的对象数量为 1
  assert(hwloc_get_nbobjs_by_depth(topology, 0) == 1);
  # 获取根对象
  obj = hwloc_get_root_obj(topology);
  # 确保根对象存在
  assert(obj);
  # 确保根对象没有父对象
  assert(!obj->parent);
  # 确保根对象有 cpuset
  assert(obj->cpuset);
  # 确保根对象的深度为 0
  assert(!obj->depth);

  # 检查允许集合是否大于主集合
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED) {
    # 如果包含不允许的标志，则确保允许的 cpuset 包含根对象的 cpuset
    assert(hwloc_bitmap_isincluded(topology->allowed_cpuset, obj->cpuset));
    # 如果包含不允许的标志，则确保允许的 nodeset 包含根对象的 nodeset
    assert(hwloc_bitmap_isincluded(topology->allowed_nodeset, obj->nodeset));
  } else {
    # 如果不包含不允许的标志，则确保允许的 cpuset 等于根对象的 cpuset
    assert(hwloc_bitmap_isequal(topology->allowed_cpuset, obj->cpuset));
    # 如果不包含不允许的标志，则确保允许的 nodeset 等于根对象的 nodeset
    assert(hwloc_bitmap_isequal(topology->allowed_nodeset, obj->nodeset));
  }

  # 检查每个层级
  for(j=0; j<depth; j++)
    hwloc__check_level(topology, j, NULL, NULL);
  for(j=0; j<HWLOC_NR_SLEVELS; j++)
    # 检查给定层级的对象，包括其子对象的层级
    hwloc__check_level(topology, HWLOC_SLEVEL_TO_DEPTH(j), topology->slevels[j].first, topology->slevels[j].last);

  # 递归检查子对象的树和特定类型的检查
  gp_indexes = hwloc_bitmap_alloc(); # 分配内存给 gp_indexes，用于存储索引
  hwloc__check_object(topology, gp_indexes, obj); # 检查对象的类型和索引
  hwloc_bitmap_free(gp_indexes); # 释放 gp_indexes 占用的内存

  # 递归检查子对象的节点集
  set = hwloc_bitmap_alloc(); # 分配内存给 set，用于存储节点集
  hwloc__check_nodesets(topology, obj, set); # 检查对象的节点集
  hwloc_bitmap_free(set); # 释放 set 占用的内存
#else /* NDEBUG */

// 如果未定义 NDEBUG 宏，则执行以下代码

void
hwloc_topology_check(struct hwloc_topology *topology __hwloc_attribute_unused)
{
    // 空函数，用于在非调试模式下检查拓扑结构
}

#endif /* NDEBUG */
```