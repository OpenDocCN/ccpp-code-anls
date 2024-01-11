# `xmrig\src\3rdparty\hwloc\include\private\autogen\config.h`

```
/* 
 * 版权声明
 * 版权声明
 * 版权声明
 * 版权声明
 * 版权声明
 *
 * 可能存在其他版权声明
 *
 * 头文件保护宏，防止重复包含
 */
#ifndef HWLOC_CONFIGURE_H
#define HWLOC_CONFIGURE_H

// 导出符号声明
#define DECLSPEC_EXPORTS

// 定义是否有 MSVC CPUIDEX
#define HWLOC_HAVE_MSVC_CPUIDEX 1

// 如果系统有类型 `CACHE_DESCRIPTOR'，定义为 1
#define HAVE_CACHE_DESCRIPTOR 0

// 如果系统有类型 `CACHE_RELATIONSHIP'，定义为 1
#define HAVE_CACHE_RELATIONSHIP 0

// 如果有 `clz' 函数，定义为 1
/* #undef HAVE_CLZ */

// 如果有 `clzl' 函数，定义为 1
/* #undef HAVE_CLZL */

// 如果有 <CL/cl_ext.h> 头文件，定义为 1
/* #undef HAVE_CL_CL_EXT_H */

// 如果有 `cpuset_setaffinity' 函数，定义为 1
/* #undef HAVE_CPUSET_SETAFFINITY */

// 如果有 `cpuset_setid' 函数，定义为 1
/* #undef HAVE_CPUSET_SETID */

// 如果有 -lcuda，定义为 1
/* #undef HAVE_CUDA */

// 如果有 <cuda.h> 头文件，定义为 1
/* #undef HAVE_CUDA_H */

// 如果有 <cuda_runtime_api.h> 头文件，定义为 1
/* #undef HAVE_CUDA_RUNTIME_API_H */

// 如果有 `CL_DEVICE_TOPOLOGY_AMD' 的声明，定义为 1，否则为 0
/* #undef HAVE_DECL_CL_DEVICE_TOPOLOGY_AMD */

// 如果有 `CTL_HW' 的声明，定义为 1，否则为 0
/* #undef HAVE_DECL_CTL_HW */

// 如果有 `fabsf' 的声明，定义为 1，否则为 0
#define HAVE_DECL_FABSF 1

// 如果有 `modff' 的声明，定义为 1，否则为 0
#define HAVE_DECL_MODFF 1

// 如果有 `HW_NCPU' 的声明，定义为 1，否则为 0
/* #undef HAVE_DECL_HW_NCPU */
/* 定义为1，如果你有`nvmlDeviceGetMaxPcieLinkGeneration`的声明，为0则没有 */
/* #undef HAVE_DECL_NVMLDEVICEGETMAXPCIELINKGENERATION */

/* 定义为1，如果你有`pthread_getaffinity_np`的声明，为0则没有 */
#define HAVE_DECL_PTHREAD_GETAFFINITY_NP 0

/* 定义为1，如果你有`pthread_setaffinity_np`的声明，为0则没有 */
#define HAVE_DECL_PTHREAD_SETAFFINITY_NP 0

/* 定义为1，如果你有`strtoull`的声明，为0则没有 */
#define HAVE_DECL_STRTOULL 0

/* 定义为1，如果你有`strcasecmp`的声明，为0则没有 */
/* #undef HWLOC_HAVE_DECL_STRCASECMP */

/* 定义为1，如果你有`snprintf`的声明，为0则没有 */
#define HAVE_DECL_SNPRINTF 0

/* 定义为1，如果你有`_strdup`的声明，为0则没有 */
#define HAVE_DECL__STRDUP 1

/* 定义为1，如果你有`_putenv`的声明，为0则没有 */
#define HAVE_DECL__PUTENV 1

/* 定义为1，如果你有`_SC_LARGE_PAGESIZE`的声明，为0则没有 */
#define HAVE_DECL__SC_LARGE_PAGESIZE 0

/* 定义为1，如果你有`_SC_NPROCESSORS_CONF`的声明，为0则没有 */
#define HAVE_DECL__SC_NPROCESSORS_CONF 0

/* 定义为1，如果你有`_SC_NPROCESSORS_ONLN`的声明，为0则没有 */
#define HAVE_DECL__SC_NPROCESSORS_ONLN 0

/* 定义为1，如果你有`_SC_NPROC_CONF`的声明，为0则没有 */
#define HAVE_DECL__SC_NPROC_CONF 0

/* 定义为1，如果你有`_SC_NPROC_ONLN`的声明，为0则没有 */
#define HAVE_DECL__SC_NPROC_ONLN 0

/* 定义为1，如果你有`_SC_PAGESIZE`的声明，为0则没有 */
#define HAVE_DECL__SC_PAGESIZE 0

/* 定义为1，如果你有`_SC_PAGE_SIZE`的声明，为0则没有 */
#define HAVE_DECL__SC_PAGE_SIZE 0
/* Define to 1 if you have the <dirent.h> header file. */
/* #define HAVE_DIRENT_H 1 */
// 如果有 <dirent.h> 头文件，则定义 HAVE_DIRENT_H 为 1，否则注释掉

#undef HAVE_DIRENT_H
// 取消定义 HAVE_DIRENT_H

/* Define to 1 if you have the <dlfcn.h> header file. */
/* #undef HAVE_DLFCN_H */
// 如果有 <dlfcn.h> 头文件，则定义 HAVE_DLFCN_H 为 1，否则注释掉

/* Define to 1 if you have the `ffs' function. */
/* #undef HAVE_FFS */
// 如果有 `ffs` 函数，则定义 HAVE_FFS 为 1，否则注释掉

/* Define to 1 if you have the `ffsl' function. */
/* #undef HAVE_FFSL */
// 如果有 `ffsl` 函数，则定义 HAVE_FFSL 为 1，否则注释掉

/* Define to 1 if you have the `fls' function. */
/* #undef HAVE_FLS */
// 如果有 `fls` 函数，则定义 HAVE_FLS 为 1，否则注释掉

/* Define to 1 if you have the `flsl' function. */
/* #undef HAVE_FLSL */
// 如果有 `flsl` 函数，则定义 HAVE_FLSL 为 1，否则注释掉

/* Define to 1 if you have the `getpagesize' function. */
#define HAVE_GETPAGESIZE 1
// 如果有 `getpagesize` 函数，则定义 HAVE_GETPAGESIZE 为 1

/* Define to 1 if the system has the type `GROUP_AFFINITY'. */
#define HAVE_GROUP_AFFINITY 1
// 如果系统有 `GROUP_AFFINITY` 类型，则定义 HAVE_GROUP_AFFINITY 为 1

/* Define to 1 if the system has the type `GROUP_RELATIONSHIP'. */
#define HAVE_GROUP_RELATIONSHIP 1
// 如果系统有 `GROUP_RELATIONSHIP` 类型，则定义 HAVE_GROUP_RELATIONSHIP 为 1

/* Define to 1 if you have the `host_info' function. */
/* #undef HAVE_HOST_INFO */
// 如果有 `host_info` 函数，则定义 HAVE_HOST_INFO 为 1，否则注释掉

/* Define to 1 if you have the <infiniband/verbs.h> header file. */
/* #undef HAVE_INFINIBAND_VERBS_H */
// 如果有 <infiniband/verbs.h> 头文件，则定义 HAVE_INFINIBAND_VERBS_H 为 1，否则注释掉

/* Define to 1 if you have the <inttypes.h> header file. */
#define HAVE_INTTYPES_H 1
// 如果有 <inttypes.h> 头文件，则定义 HAVE_INTTYPES_H 为 1

/* Define to 1 if the system has the type `KAFFINITY'. */
#define HAVE_KAFFINITY 1
// 如果系统有 `KAFFINITY` 类型，则定义 HAVE_KAFFINITY 为 1

/* Define to 1 if you have the <kstat.h> header file. */
/* #undef HAVE_KSTAT_H */
// 如果有 <kstat.h> 头文件，则定义 HAVE_KSTAT_H 为 1，否则注释掉

/* Define to 1 if you have the <langinfo.h> header file. */
/* #undef HAVE_LANGINFO_H */
// 如果有 <langinfo.h> 头文件，则定义 HAVE_LANGINFO_H 为 1，否则注释掉

/* Define to 1 if we have -lgdi32 */
#define HAVE_LIBGDI32 1
// 如果有 -lgdi32，则定义 HAVE_LIBGDI32 为 1

/* Define to 1 if we have -libverbs */
/* #undef HAVE_LIBIBVERBS */
// 如果有 -libverbs，则定义 HAVE_LIBIBVERBS 为 1，否则注释掉

/* Define to 1 if we have -lkstat */
/* #undef HAVE_LIBKSTAT */
// 如果有 -lkstat，则定义 HAVE_LIBKSTAT 为 1，否则注释掉

/* Define to 1 if we have -llgrp */
/* #undef HAVE_LIBLGRP */
// 如果有 -llgrp，则定义 HAVE_LIBLGRP 为 1，否则注释掉

/* Define to 1 if you have the <locale.h> header file. */
#define HAVE_LOCALE_H 1
// 如果有 <locale.h> 头文件，则定义 HAVE_LOCALE_H 为 1

/* Define to 1 if the system has the type `LOGICAL_PROCESSOR_RELATIONSHIP'. */
#define HAVE_LOGICAL_PROCESSOR_RELATIONSHIP 1
// 如果系统有 `LOGICAL_PROCESSOR_RELATIONSHIP` 类型，则定义 HAVE_LOGICAL_PROCESSOR_RELATIONSHIP 为 1

/* Define to 1 if you have the <mach/mach_host.h> header file. */
/* #undef HAVE_MACH_MACH_HOST_H */
// 如果有 <mach/mach_host.h> 头文件，则定义 HAVE_MACH_MACH_HOST_H 为 1，否则注释掉

/* Define to 1 if you have the <mach/mach_init.h> header file. */
/* #undef HAVE_MACH_MACH_INIT_H */
// 如果有 <mach/mach_init.h> 头文件，则定义 HAVE_MACH_MACH_INIT_H 为 1，否则注释掉
/* Define to 1 if you have the <malloc.h> header file. */
#define HAVE_MALLOC_H 1

/* Define to 1 if you have the `memalign' function. */
/* #undef HAVE_MEMALIGN */

/* Define to 1 if you have the <memory.h> header file. */
#define HAVE_MEMORY_H 1

/* Define to 1 if you have the `nl_langinfo' function. */
/* #undef HAVE_NL_LANGINFO */

/* Define to 1 if you have the <numaif.h> header file. */
/* #undef HAVE_NUMAIF_H */

/* Define to 1 if the system has the type `NUMA_NODE_RELATIONSHIP'. */
#define HAVE_NUMA_NODE_RELATIONSHIP 1

/* Define to 1 if you have the <NVCtrl/NVCtrl.h> header file. */
/* #undef HAVE_NVCTRL_NVCTRL_H */

/* Define to 1 if you have the <nvml.h> header file. */
/* #undef HAVE_NVML_H */

/* Define to 1 if you have the `openat' function. */
/* #undef HAVE_OPENAT */

/* Define to 1 if you have the <picl.h> header file. */
/* #undef HAVE_PICL_H */

/* Define to 1 if you have the `posix_memalign' function. */
/* #undef HAVE_POSIX_MEMALIGN */

/* Define to 1 if the system has the type `PROCESSOR_CACHE_TYPE'. */
#define HAVE_PROCESSOR_CACHE_TYPE 1

/* Define to 1 if the system has the type `PROCESSOR_GROUP_INFO'. */
#define HAVE_PROCESSOR_GROUP_INFO 1

/* Define to 1 if the system has the type `PROCESSOR_RELATIONSHIP'. */
#define HAVE_PROCESSOR_RELATIONSHIP 1

/* Define to 1 if the system has the type `PSAPI_WORKING_SET_EX_BLOCK'. */
/* #undef HAVE_PSAPI_WORKING_SET_EX_BLOCK */

/* Define to 1 if the system has the type `PSAPI_WORKING_SET_EX_INFORMATION'. */
/* #undef HAVE_PSAPI_WORKING_SET_EX_INFORMATION */

/* Define to 1 if the system has the type `PROCESSOR_NUMBER'. */
#define HAVE_PROCESSOR_NUMBER 1

/* Define to 1 if you have the <pthread_np.h> header file. */
/* #undef HAVE_PTHREAD_NP_H */

/* Define to 1 if the system has the type `pthread_t'. */
/* #undef HAVE_PTHREAD_T */

/* Define to 1 if you have the `putwc' function. */
#define HAVE_PUTWC 1

/* Define to 1 if the system has the type `RelationProcessorPackage'. */
/* #undef HAVE_RELATIONPROCESSORPACKAGE */

/* 定义为1，表示有`setlocale'函数 */
#define HAVE_SETLOCALE 1

/* 定义为1，表示有<stdint.h>头文件 */
#define HAVE_STDINT_H 1

/* 定义为1，表示有<stdlib.h>头文件 */
#define HAVE_STDLIB_H 1

/* 定义为1，表示有`strftime'函数 */
#define HAVE_STRFTIME 1

/* 定义为1，表示有<string.h>头文件 */
#define HAVE_STRING_H 1

/* 定义为1，表示有`strncasecmp'函数 */
#define HAVE_STRNCASECMP 1

/* 定义为1，表示有`SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX'类型 */
#define HAVE_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX 1

/* 定义为1，表示有<sys/stat.h>头文件 */
#define HAVE_SYS_STAT_H 1

/* 定义为1，表示有<sys/types.h>头文件 */
#define HAVE_SYS_TYPES_H 1
/* #undef HAVE_USELOCALE */  // 如果系统支持 uselocale，则未定义

/* Define to 1 if the system has the type `wchar_t'. */
#define HAVE_WCHAR_T 1  // 如果系统支持 wchar_t 类型，则定义为 1

/* Define to 1 if you have the <X11/keysym.h> header file. */
/* #undef HAVE_X11_KEYSYM_H */  // 如果有 X11/keysym.h 头文件，则未定义

/* Define to 1 if you have the <X11/Xlib.h> header file. */
/* #undef HAVE_X11_XLIB_H */  // 如果有 X11/Xlib.h 头文件，则未定义

/* Define to 1 if you have the <X11/Xutil.h> header file. */
/* #undef HAVE_X11_XUTIL_H */  // 如果有 X11/Xutil.h 头文件，则未定义

/* Define to 1 if you have the <xlocale.h> header file. */
/* #undef HAVE_XLOCALE_H */  // 如果有 xlocale.h 头文件，则未定义

/* Define to 1 on AIX */
/* #undef HWLOC_AIX_SYS */  // 如果在 AIX 系统上，则未定义

/* Define to 1 on BlueGene/Q */
/* #undef HWLOC_BGQ_SYS */  // 如果在 BlueGene/Q 系统上，则未定义

/* Whether C compiler supports symbol visibility or not */
#define HWLOC_C_HAVE_VISIBILITY 0  // C 编译器是否支持符号可见性，定义为 0

/* Define to 1 on Darwin */
/* #undef HWLOC_DARWIN_SYS */  // 如果在 Darwin 系统上，则未定义

/* Whether we are in debugging mode or not */
/* #undef HWLOC_DEBUG */  // 是否处于调试模式，未定义

/* Define to 1 on *FREEBSD */
/* #undef HWLOC_FREEBSD_SYS */  // 如果在 *FREEBSD 系统上，则未定义

/* Whether your compiler has __attribute__ or not */
/* #define HWLOC_HAVE_ATTRIBUTE 1 */  // 编译器是否有 __attribute__，定义为 1
#undef HWLOC_HAVE_ATTRIBUTE  // 取消定义 HWLOC_HAVE_ATTRIBUTE

/* Whether your compiler has __attribute__ aligned or not */
/* #define HWLOC_HAVE_ATTRIBUTE_ALIGNED 1 */  // 编译器是否有 __attribute__ aligned，定义为 1

/* Whether your compiler has __attribute__ always_inline or not */
/* #define HWLOC_HAVE_ATTRIBUTE_ALWAYS_INLINE 1 */  // 编译器是否有 __attribute__ always_inline，定义为 1

/* Whether your compiler has __attribute__ cold or not */
/* #define HWLOC_HAVE_ATTRIBUTE_COLD 1 */  // 编译器是否有 __attribute__ cold，定义为 1

/* Whether your compiler has __attribute__ const or not */
/* #define HWLOC_HAVE_ATTRIBUTE_CONST 1 */  // 编译器是否有 __attribute__ const，定义为 1

/* Whether your compiler has __attribute__ deprecated or not */
/* #define HWLOC_HAVE_ATTRIBUTE_DEPRECATED 1 */  // 编译器是否有 __attribute__ deprecated，定义为 1

/* Whether your compiler has __attribute__ format or not */
/* #define HWLOC_HAVE_ATTRIBUTE_FORMAT 1 */  // 编译器是否有 __attribute__ format，定义为 1

/* Whether your compiler has __attribute__ hot or not */
/* #define HWLOC_HAVE_ATTRIBUTE_HOT 1 */  // 编译器是否有 __attribute__ hot，定义为 1

/* Whether your compiler has __attribute__ malloc or not */
/* #define HWLOC_HAVE_ATTRIBUTE_MALLOC 1 */  // 编译器是否有 __attribute__ malloc，定义为 1

/* Whether your compiler has __attribute__ may_alias or not */
/* #define HWLOC_HAVE_ATTRIBUTE_MAY_ALIAS 1 */  // 编译器是否有 __attribute__ may_alias，定义为 1
/* 是否你的编译器具有__attribute__ nonnull */
/* #define HWLOC_HAVE_ATTRIBUTE_NONNULL 1 */

/* 是否你的编译器具有__attribute__ noreturn */
/* #define HWLOC_HAVE_ATTRIBUTE_NORETURN 1 */

/* 是否你的编译器具有__attribute__ no_instrument_function */
/* #define HWLOC_HAVE_ATTRIBUTE_NO_INSTRUMENT_FUNCTION 1 */

/* 是否你的编译器具有__attribute__ packed */
/* #define HWLOC_HAVE_ATTRIBUTE_PACKED 1 */

/* 是否你的编译器具有__attribute__ pure */
/* #define HWLOC_HAVE_ATTRIBUTE_PURE 1 */

/* 是否你的编译器具有__attribute__ sentinel */
/* #define HWLOC_HAVE_ATTRIBUTE_SENTINEL 1 */

/* 是否你的编译器具有__attribute__ unused */
/* #define HWLOC_HAVE_ATTRIBUTE_UNUSED 1 */

/* 是否你的编译器具有__attribute__ warn unused result */
/* #define HWLOC_HAVE_ATTRIBUTE_WARN_UNUSED_RESULT 1 */

/* 是否你的编译器具有__attribute__ weak alias */
/* #define HWLOC_HAVE_ATTRIBUTE_WEAK_ALIAS 1 */

/* 如果你的 `ffs' 函数已知存在问题，则定义为1 */
/* #undef HWLOC_HAVE_BROKEN_FFS */

/* 如果你有 `cairo' 库，则定义为1 */
/* #undef HWLOC_HAVE_CAIRO */

/* 如果你有 `clz' 函数，则定义为1 */
/* #undef HWLOC_HAVE_CLZ */

/* 如果你有 `clzl' 函数，则定义为1 */
/* #undef HWLOC_HAVE_CLZL */

/* 如果你有 cpuid，则定义为1 */
/* #undef HWLOC_HAVE_CPUID */

/* 如果 CPU_SET 宏可用，则定义为1 */
/* #undef HWLOC_HAVE_CPU_SET */

/* 如果 CPU_SET_S 宏可用，则定义为1 */
/* #undef HWLOC_HAVE_CPU_SET_S */

/* 如果你有 `cudart' SDK，则定义为1 */
/* #undef HWLOC_HAVE_CUDART */

/* 如果函数 `clz' 在系统头文件中声明，则定义为1 */
/* #undef HWLOC_HAVE_DECL_CLZ */

/* 如果函数 `clzl' 在系统头文件中声明，则定义为1 */
/* #undef HWLOC_HAVE_DECL_CLZL */

/* 如果函数 `ffs' 在系统头文件中声明，则定义为1 */
/* #undef HWLOC_HAVE_DECL_FFS */
/* 如果系统头文件声明了函数`ffsl`，则定义为1 */
/* #undef HWLOC_HAVE_DECL_FFSL */

/* 如果系统头文件声明了函数`fls`，则定义为1 */
/* #undef HWLOC_HAVE_DECL_FLS */

/* 如果系统头文件声明了函数`flsl`，则定义为1 */
/* #undef HWLOC_HAVE_DECL_FLSL */

/* 如果有`ffs`函数，则定义为1 */
/* #undef HWLOC_HAVE_FFS */

/* 如果有`ffsl`函数，则定义为1 */
/* #undef HWLOC_HAVE_FFSL */

/* 如果有`fls`函数，则定义为1 */
/* #undef HWLOC_HAVE_FLS */

/* 如果有`flsl`函数，则定义为1 */
/* #undef HWLOC_HAVE_FLSL */

/* 如果有GL模块组件，则定义为1 */
/* #undef HWLOC_HAVE_GL */

/* 如果有提供termcap接口的库，则定义为1 */
/* #undef HWLOC_HAVE_LIBTERMCAP */

/* 如果有`libxml2`库，则定义为1 */
/* #undef HWLOC_HAVE_LIBXML2 */

/* 如果正在构建Linux PCI组件，则定义为1 */
/* #undef HWLOC_HAVE_LINUXPCI */

/* 如果有`NVML`库，则定义为1 */
/* #undef HWLOC_HAVE_NVML */

/* 如果glibc提供了旧的sched_setaffinity()原型（不带长度），则定义为1 */
/* #undef HWLOC_HAVE_OLD_SCHED_SETAFFINITY */

/* 如果有`OpenCL`库，则定义为1 */
/* #undef HWLOC_HAVE_OPENCL */

/* 如果hwloc库应该支持动态加载的插件，则定义为1 */
/* #undef HWLOC_HAVE_PLUGINS */

/* 如果有`pthread_getthrds_np`函数，则定义为1 */
/* #undef HWLOC_HAVE_PTHREAD_GETTHRDS_NP */

/* 如果pthread互斥锁可用，则定义为1 */
/* #undef HWLOC_HAVE_PTHREAD_MUTEX */

/* 如果glibc提供了sched_setaffinity()的原型，则定义为1 */
#define HWLOC_HAVE_SCHED_SETAFFINITY 1

/* 如果有<stdint.h>头文件，则定义为1 */
#define HWLOC_HAVE_STDINT_H 1

/* 如果有`windows.h`头文件，则定义为1 */
#define HWLOC_HAVE_WINDOWS_H 1

/* 如果X11头文件包括Xutil.h和keysym.h可用，则定义为1 */
/* #undef HWLOC_HAVE_X11_KEYSYM */
// 如果定义了 HWLOC_HAVE_X11_KEYSYM，则表示具有 X11 KeySym 功能

/* Define to 1 if function `syscall' is available */
// 如果定义了 HWLOC_HAVE_SYSCALL，则表示具有 syscall 函数

/* Define to 1 on HP-UX */
// 如果定义了 HWLOC_HPUX_SYS，则表示在 HP-UX 系统上

/* Define to 1 on Linux */
// 如果定义了 HWLOC_LINUX_SYS，则表示在 Linux 系统上

/* Define to 1 on *NETBSD */
// 如果定义了 HWLOC_NETBSD_SYS，则表示在 *NETBSD 系统上

/* The size of `unsigned int', as computed by sizeof */
// 通过 sizeof 计算得到的 `unsigned int` 的大小
#define HWLOC_SIZEOF_UNSIGNED_INT 4

/* The size of `unsigned long', as computed by sizeof */
// 通过 sizeof 计算得到的 `unsigned long` 的大小
#define HWLOC_SIZEOF_UNSIGNED_LONG 4

/* Define to 1 on Solaris */
// 如果定义了 HWLOC_SOLARIS_SYS，则表示在 Solaris 系统上

/* The hwloc symbol prefix */
// hwloc 符号的前缀
#define HWLOC_SYM_PREFIX hwloc_

/* The hwloc symbol prefix in all caps */
// 所有大写的 hwloc 符号前缀
#define HWLOC_SYM_PREFIX_CAPS HWLOC_

/* Whether we need to re-define all the hwloc public symbols or not */
// 是否需要重新定义所有 hwloc 公共符号

#define HWLOC_SYM_TRANSFORM 0

/* Define to 1 on unsupported systems */
// 如果定义了 HWLOC_UNSUPPORTED_SYS，则表示在不支持的系统上

/* Define to 1 if ncurses works, preferred over curses */
// 如果定义了 HWLOC_USE_NCURSES，则表示 ncurses 可用，优先于 curses

/* Define to 1 on WINDOWS */
// 如果定义了 HWLOC_WIN_SYS，则表示在 Windows 系统上

/* Define to 1 on x86_32 */
// 如果定义了 HWLOC_X86_32_ARCH，则表示在 x86_32 架构上

/* Define to 1 on x86_64 */
// 如果定义了 HWLOC_X86_64_ARCH，则表示在 x86_64 架构上

/* Define to the sub-directory in which libtool stores uninstalled libraries. */
// 定义 libtool 存储未安装库的子目录

#define LT_OBJDIR ".libs/"

/* Name of package */
// 软件包的名称
#define PACKAGE "hwloc"

/* Define to the address where bug reports for this package should be sent. */
// 定义应发送给该软件包的错误报告的地址
#define PACKAGE_BUGREPORT "https://www.open-mpi.org/projects/hwloc/"

/* Define to the full name of this package. */
// 定义该软件包的完整名称
#define PACKAGE_NAME "hwloc"

/* Define to the full name and version of this package. */
// 定义该软件包的完整名称和版本
#define PACKAGE_STRING "hwloc"

/* Define to the one symbol short name of this package. */
// 定义该软件包的一个符号短名称
#define PACKAGE_TARNAME "hwloc"

/* Define to the home page for this package. */
// 定义该软件包的主页
#define PACKAGE_URL ""

/* Define to the version of this package. */
// 定义该软件包的版本
#define PACKAGE_VERSION HWLOC_VERSION

/* The size of `unsigned int', as computed by sizeof. */
// 通过 sizeof 计算得到的 `unsigned int` 的大小
#define SIZEOF_UNSIGNED_INT 4

/* The size of `unsigned long', as computed by sizeof. */
// 通过 sizeof 计算得到的 `unsigned long` 的大小
/* The size of `unsigned long`, as computed by sizeof. */
#define SIZEOF_UNSIGNED_LONG 4

/* The size of `void *`, as computed by sizeof. */
#define SIZEOF_VOID_P 8

/* Define to 1 if you have the ANSI C header files. */
#define STDC_HEADERS 1

/* Enable extensions on HP-UX. */
#ifndef _HPUX_SOURCE
# define _HPUX_SOURCE 1
#endif

/* Enable extensions on AIX 3, Interix.  */
/*
#ifndef _ALL_SOURCE
# define _ALL_SOURCE 1
#endif
*/

/* Enable GNU extensions on systems that have them.  */
/*
#ifndef _GNU_SOURCE
# define _GNU_SOURCE 1
#endif
*/

/* Enable threading extensions on Solaris.  */
/*
#ifndef _POSIX_PTHREAD_SEMANTICS
# define _POSIX_PTHREAD_SEMANTICS 1
#endif
*/

/* Enable extensions on HP NonStop.  */
/*
#ifndef _TANDEM_SOURCE
# define _TANDEM_SOURCE 1
#endif
*/

/* Enable general extensions on Solaris.  */
/*
#ifndef __EXTENSIONS__
# define __EXTENSIONS__ 1
#endif
*/

/* Version number of package */
#define VERSION HWLOC_VERSION

/* Define to 1 if the X Window System is missing or not being used. */
#define X_DISPLAY_MISSING 1

/* Define to 1 if on MINIX. */
/* #undef _MINIX */

/* Define to 2 if the system does not provide POSIX.1 features except with
   this defined. */
/* #undef _POSIX_1_SOURCE */

/* Define to 1 if you need to in order for `stat' and other things to work. */
/* #undef _POSIX_SOURCE */

/* Define this to the process ID type */
#define hwloc_pid_t HANDLE

/* Define this to either strncasecmp or strncmp */
#define hwloc_strncasecmp strncasecmp

/* Define this to the thread ID type */
#define hwloc_thread_t HANDLE

/* Define to 1 if you have the declaration of `GetModuleFileName', and to 0 if
   you don't. */
#define HAVE_DECL_GETMODULEFILENAME 1

#endif /* HWLOC_CONFIGURE_H */
```