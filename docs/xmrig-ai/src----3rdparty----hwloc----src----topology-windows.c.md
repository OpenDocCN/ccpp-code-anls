# `xmrig\src\3rdparty\hwloc\src\topology-windows.c`

```
/*
 * 版权声明
 */
/* 尝试在下面复制所有声明 */

/* 定义 _WIN32_WINNT 为 0x0601 */
#define _WIN32_WINNT 0x0601

/* 包含自动生成的配置文件 */
#include "private/autogen/config.h"
/* 包含 hwloc.h 头文件 */
#include "hwloc.h"
/* 包含 windows.h 头文件 */
#include "hwloc/windows.h"
/* 包含私有头文件 */
#include "private/private.h"
/* 包含 windows.h 头文件之前必须包含的私有 windows.h 头文件 */
#include "private/windows.h"
/* 包含调试相关的私有头文件 */
#include "private/debug.h"

/* 包含 Windows.h 头文件 */
#include <windows.h>

/* 如果没有定义 HAVE_KAFFINITY，则定义 KAFFINITY 为 ULONG_PTR 类型 */
#ifndef HAVE_KAFFINITY
typedef ULONG_PTR KAFFINITY, *PKAFFINITY;
#endif

/* 如果没有定义 HAVE_PROCESSOR_CACHE_TYPE，则定义 PROCESSOR_CACHE_TYPE 枚举类型 */
#ifndef HAVE_PROCESSOR_CACHE_TYPE
typedef enum _PROCESSOR_CACHE_TYPE {
  CacheUnified,
  CacheInstruction,
  CacheData,
  CacheTrace
} PROCESSOR_CACHE_TYPE;
#endif

/* 如果没有定义 CACHE_FULLY_ASSOCIATIVE，则定义 CACHE_FULLY_ASSOCIATIVE 为 0xFF */
#ifndef CACHE_FULLY_ASSOCIATIVE
#define CACHE_FULLY_ASSOCIATIVE 0xFF
#endif

/* 如果没有定义 MAXIMUM_PROC_PER_GROUP，则定义 MAXIMUM_PROC_PER_GROUP 为 64 */
#ifndef MAXIMUM_PROC_PER_GROUP /* 在 MinGW 中缺失 */
#define MAXIMUM_PROC_PER_GROUP 64
#endif

/* 如果没有定义 HAVE_CACHE_DESCRIPTOR，则定义 CACHE_DESCRIPTOR 结构体 */
#ifndef HAVE_CACHE_DESCRIPTOR
typedef struct _CACHE_DESCRIPTOR {
  BYTE Level;
  BYTE Associativity;
  WORD LineSize;
  DWORD Size; /* 以字节为单位 */
  PROCESSOR_CACHE_TYPE Type;
} CACHE_DESCRIPTOR, *PCACHE_DESCRIPTOR;
#endif

/* 如果没有定义 HAVE_LOGICAL_PROCESSOR_RELATIONSHIP，则定义 LOGICAL_PROCESSOR_RELATIONSHIP 枚举类型 */
#ifndef HAVE_LOGICAL_PROCESSOR_RELATIONSHIP
typedef enum _LOGICAL_PROCESSOR_RELATIONSHIP {
  RelationProcessorCore,
  RelationNumaNode,
  RelationCache,
  RelationProcessorPackage,
  RelationGroup,
  RelationAll = 0xffff
} LOGICAL_PROCESSOR_RELATIONSHIP;
#else /* 如果定义了 HAVE_LOGICAL_PROCESSOR_RELATIONSHIP */
#  ifndef HAVE_RELATIONPROCESSORPACKAGE
#    define RelationProcessorPackage 3
#    define RelationGroup 4
#    define RelationAll 0xffff
#  endif /* 如果没有定义 HAVE_RELATIONPROCESSORPACKAGE */
#endif /* 如果没有定义 HAVE_LOGICAL_PROCESSOR_RELATIONSHIP */

/* 如果没有定义 HAVE_GROUP_AFFINITY，则定义 GROUP_AFFINITY 结构体 */
#ifndef HAVE_GROUP_AFFINITY
typedef struct _GROUP_AFFINITY {
  KAFFINITY Mask;
  WORD Group;
  WORD Reserved[3];
} GROUP_AFFINITY, *PGROUP_AFFINITY;
#endif

/* 始终使用自己的结构，因为 EfficiencyClass 字段在 Win10 之前不存在 */
/* 定义处理器关系结构体，用于描述处理器之间的关系 */
typedef struct HWLOC_PROCESSOR_RELATIONSHIP {
  BYTE Flags;  // 标志位
  BYTE EfficiencyClass; /* for RelationProcessorCore, higher means greater performance but less efficiency */  // 效率等级，对于 RelationProcessorCore，值越高表示性能越好但效率越低
  BYTE Reserved[20];  // 保留字段
  WORD GroupCount;  // 组数量
  GROUP_AFFINITY GroupMask[ANYSIZE_ARRAY];  // 组掩码数组
} HWLOC_PROCESSOR_RELATIONSHIP;

/* 总是使用自己的结构，因为在某些 Win10 中 GroupCount 和 GroupMasks 字段不存在 */
typedef struct HWLOC_NUMA_NODE_RELATIONSHIP {
  DWORD NodeNumber;  // 节点编号
  BYTE Reserved[18];  // 保留字段
  WORD GroupCount;  // 组数量
  _ANONYMOUS_UNION
  union {
    GROUP_AFFINITY GroupMask;  // 组掩码
    GROUP_AFFINITY GroupMasks[ANYSIZE_ARRAY];  // 组掩码数组
  } DUMMYUNIONNAME;
} HWLOC_NUMA_NODE_RELATIONSHIP;

typedef struct HWLOC_CACHE_RELATIONSHIP {
  BYTE Level;  // 缓存级别
  BYTE Associativity;  // 关联度
  WORD LineSize;  // 行大小
  DWORD CacheSize;  // 缓存大小
  PROCESSOR_CACHE_TYPE Type;  // 缓存类型
  BYTE Reserved[18];  // 保留字段
  WORD GroupCount;  // 组数量
  union {
    GROUP_AFFINITY GroupMask;  // 组掩码
    GROUP_AFFINITY GroupMasks[ANYSIZE_ARRAY];  // 组掩码数组
  } DUMMYUNIONNAME;
} HWLOC_CACHE_RELATIONSHIP;

#ifndef HAVE_PROCESSOR_GROUP_INFO
typedef struct _PROCESSOR_GROUP_INFO {
  BYTE MaximumProcessorCount;  // 最大处理器数量
  BYTE ActiveProcessorCount;  // 活动处理器数量
  BYTE Reserved[38];  // 保留字段
  KAFFINITY ActiveProcessorMask;  // 活动处理器掩码
} PROCESSOR_GROUP_INFO, *PPROCESSOR_GROUP_INFO;
#endif

#ifndef HAVE_GROUP_RELATIONSHIP
typedef struct _GROUP_RELATIONSHIP {
  WORD MaximumGroupCount;  // 最大组数量
  WORD ActiveGroupCount;  // 活动组数量
  ULONGLONG Reserved[2];  // 保留字段
  PROCESSOR_GROUP_INFO GroupInfo[ANYSIZE_ARRAY];  // 处理器组信息数组
} GROUP_RELATIONSHIP, *PGROUP_RELATIONSHIP;
#endif

/* 总是使用自己的结构，因为我们需要自己的 HWLOC_PROCESSOR/CACHE/NUMA_NODE_RELATIONSHIP */
typedef struct HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX {
  LOGICAL_PROCESSOR_RELATIONSHIP Relationship;  // 逻辑处理器关系
  DWORD Size;  // 大小
  _ANONYMOUS_UNION
  union {
    HWLOC_PROCESSOR_RELATIONSHIP Processor;  // 处理器关系
    HWLOC_NUMA_NODE_RELATIONSHIP NumaNode;  // NUMA 节点关系
    HWLOC_CACHE_RELATIONSHIP Cache;  // 缓存关系
    GROUP_RELATIONSHIP Group;  // 组关系
    /* 奇怪：没有成员来告诉包的 CPU 掩码... */
  } DUMMYUNIONNAME;
# 定义 HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX 结构体
} HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX;

# 如果没有定义 PSAPI_WORKING_SET_EX_BLOCK，则定义该结构体
#ifndef HAVE_PSAPI_WORKING_SET_EX_BLOCK
typedef union _PSAPI_WORKING_SET_EX_BLOCK {
  ULONG_PTR Flags;
  struct {
    unsigned Valid  :1;
    unsigned ShareCount  :3;
    unsigned Win32Protection  :11;
    unsigned Shared  :1;
    unsigned Node  :6;
    unsigned Locked  :1;
    unsigned LargePage  :1;
  };
} PSAPI_WORKING_SET_EX_BLOCK;
#endif

# 如果没有定义 PSAPI_WORKING_SET_EX_INFORMATION，则定义该结构体
#ifndef HAVE_PSAPI_WORKING_SET_EX_INFORMATION
typedef struct _PSAPI_WORKING_SET_EX_INFORMATION {
  PVOID VirtualAddress;
  PSAPI_WORKING_SET_EX_BLOCK VirtualAttributes;
} PSAPI_WORKING_SET_EX_INFORMATION;
#endif

# 如果没有定义 PROCESSOR_NUMBER，则定义该结构体
#ifndef HAVE_PROCESSOR_NUMBER
typedef struct _PROCESSOR_NUMBER {
  WORD Group;
  BYTE Number;
  BYTE Reserved;
} PROCESSOR_NUMBER, *PPROCESSOR_NUMBER;
#endif

# 定义函数指针类型
# 获取活动处理器组数量的函数指针
typedef WORD (WINAPI *PFN_GETACTIVEPROCESSORGROUPCOUNT)(void);
static PFN_GETACTIVEPROCESSORGROUPCOUNT GetActiveProcessorGroupCountProc;

# 获取活动处理器数量的函数指针
typedef WORD (WINAPI *PFN_GETACTIVEPROCESSORCOUNT)(WORD);
static PFN_GETACTIVEPROCESSORCOUNT GetActiveProcessorCountProc;

# 获取当前处理器编号的函数指针
typedef DWORD (WINAPI *PFN_GETCURRENTPROCESSORNUMBER)(void);
static PFN_GETCURRENTPROCESSORNUMBER GetCurrentProcessorNumberProc;

# 获取当前处理器编号的扩展函数指针
typedef VOID (WINAPI *PFN_GETCURRENTPROCESSORNUMBEREX)(PPROCESSOR_NUMBER);
static PFN_GETCURRENTPROCESSORNUMBEREX GetCurrentProcessorNumberExProc;

# 获取逻辑处理器信息的函数指针
typedef BOOL (WINAPI *PFN_GETLOGICALPROCESSORINFORMATIONEX)(LOGICAL_PROCESSOR_RELATIONSHIP relationship, HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX *Buffer, PDWORD ReturnLength);
static PFN_GETLOGICALPROCESSORINFORMATIONEX GetLogicalProcessorInformationExProc;

# 设置线程组亲和力的函数指针
typedef BOOL (WINAPI *PFN_SETTHREADGROUPAFFINITY)(HANDLE hThread, const GROUP_AFFINITY *GroupAffinity, PGROUP_AFFINITY PreviousGroupAffinity);
static PFN_SETTHREADGROUPAFFINITY SetThreadGroupAffinityProc;

# 获取线程组亲和力的函数指针
typedef BOOL (WINAPI *PFN_GETTHREADGROUPAFFINITY)(HANDLE hThread, PGROUP_AFFINITY GroupAffinity);
static PFN_GETTHREADGROUPAFFINITY GetThreadGroupAffinityProc;
    // 定义一个函数指针类型，用于获取指定 NUMA 节点的可用内存大小
    typedef BOOL (WINAPI *PFN_GETNUMAAVAILABLEMEMORYNODE)(UCHAR Node, PULONGLONG AvailableBytes);
    // 声明一个函数指针变量，用于指向获取指定 NUMA 节点的可用内存大小的函数
    static PFN_GETNUMAAVAILABLEMEMORYNODE GetNumaAvailableMemoryNodeProc;

    // 定义一个函数指针类型，用于获取指定 NUMA 节点的可用内存大小（扩展版）
    typedef BOOL (WINAPI *PFN_GETNUMAAVAILABLEMEMORYNODEEX)(USHORT Node, PULONGLONG AvailableBytes);
    // 声明一个函数指针变量，用于指向获取指定 NUMA 节点的可用内存大小（扩展版）的函数
    static PFN_GETNUMAAVAILABLEMEMORYNODEEX GetNumaAvailableMemoryNodeExProc;

    // 定义一个函数指针类型，用于在指定 NUMA 节点上为进程分配虚拟内存
    typedef LPVOID (WINAPI *PFN_VIRTUALALLOCEXNUMA)(HANDLE hProcess, LPVOID lpAddress, SIZE_T dwSize, DWORD flAllocationType, DWORD flProtect, DWORD nndPreferred);
    // 声明一个函数指针变量，用于指向在指定 NUMA 节点上为进程分配虚拟内存的函数
    static PFN_VIRTUALALLOCEXNUMA VirtualAllocExNumaProc;

    // 定义一个函数指针类型，用于在指定 NUMA 节点上释放虚拟内存
    typedef BOOL (WINAPI *PFN_VIRTUALFREEEX)(HANDLE hProcess, LPVOID lpAddress, SIZE_T dwSize, DWORD dwFreeType);
    // 声明一个函数指针变量，用于指向在指定 NUMA 节点上释放虚拟内存的函数
    static PFN_VIRTUALFREEEX VirtualFreeExProc;

    // 定义一个函数指针类型，用于查询工作集信息
    typedef BOOL (WINAPI *PFN_QUERYWORKINGSETEX)(HANDLE hProcess, PVOID pv, DWORD cb);
    // 声明一个函数指针变量，用于指向查询工作集信息的函数
    static PFN_QUERYWORKINGSETEX QueryWorkingSetExProc;

    // 定义一个函数指针类型，用于获取操作系统版本信息
    typedef NTSTATUS (WINAPI *PFN_RTLGETVERSION)(OSVERSIONINFOEX*);
    // 声明一个函数指针变量，用于指向获取操作系统版本信息的函数
    PFN_RTLGETVERSION RtlGetVersionProc;

    // 定义一个函数，用于获取 Windows API 函数的地址
    static void hwloc_win_get_function_ptrs(void)
    {
      HMODULE kernel32, ntdll;

    #if HWLOC_HAVE_GCC_W_CAST_FUNCTION_TYPE
    #pragma GCC diagnostic ignored "-Wcast-function-type"
    #endif

        // 加载 kernel32.dll 动态链接库
        kernel32 = LoadLibrary("kernel32.dll");
        // 如果加载成功
        if (kernel32) {
          // 获取 GetActiveProcessorGroupCount 函数的地址
          GetActiveProcessorGroupCountProc =
        (PFN_GETACTIVEPROCESSORGROUPCOUNT) GetProcAddress(kernel32, "GetActiveProcessorGroupCount");
          // 获取 GetActiveProcessorCount 函数的地址
          GetActiveProcessorCountProc =
        (PFN_GETACTIVEPROCESSORCOUNT) GetProcAddress(kernel32, "GetActiveProcessorCount");
          // 获取 GetCurrentProcessorNumber 函数的地址
          GetCurrentProcessorNumberProc =
        (PFN_GETCURRENTPROCESSORNUMBER) GetProcAddress(kernel32, "GetCurrentProcessorNumber");
          // 获取 GetCurrentProcessorNumberEx 函数的地址
          GetCurrentProcessorNumberExProc =
        (PFN_GETCURRENTPROCESSORNUMBEREX) GetProcAddress(kernel32, "GetCurrentProcessorNumberEx");
          // 获取 SetThreadGroupAffinity 函数的地址
          SetThreadGroupAffinityProc =
        (PFN_SETTHREADGROUPAFFINITY) GetProcAddress(kernel32, "SetThreadGroupAffinity");
          // 获取 GetThreadGroupAffinity 函数的地址
          GetThreadGroupAffinityProc =
        (PFN_GETTHREADGROUPAFFINITY) GetProcAddress(kernel32, "GetThreadGroupAffinity");
          // 获取 GetNumaAvailableMemoryNode 函数的地址
          GetNumaAvailableMemoryNodeProc =
    # 获取 kernel32.dll 中 GetNumaAvailableMemoryNode 函数的地址，并赋值给 GetNumaAvailableMemoryNodeProc
    (PFN_GETNUMAAVAILABLEMEMORYNODE) GetProcAddress(kernel32, "GetNumaAvailableMemoryNode");
    # 获取 kernel32.dll 中 GetNumaAvailableMemoryNodeEx 函数的地址，并赋值给 GetNumaAvailableMemoryNodeExProc
    GetNumaAvailableMemoryNodeExProc =
    (PFN_GETNUMAAVAILABLEMEMORYNODEEX) GetProcAddress(kernel32, "GetNumaAvailableMemoryNodeEx");
    # 获取 kernel32.dll 中 GetLogicalProcessorInformationEx 函数的地址，并赋值给 GetLogicalProcessorInformationExProc
    GetLogicalProcessorInformationExProc =
    (PFN_GETLOGICALPROCESSORINFORMATIONEX)GetProcAddress(kernel32, "GetLogicalProcessorInformationEx");
    # 获取 kernel32.dll 中 K32QueryWorkingSetEx 函数的地址，并赋值给 QueryWorkingSetExProc
    QueryWorkingSetExProc =
    (PFN_QUERYWORKINGSETEX) GetProcAddress(kernel32, "K32QueryWorkingSetEx");
    # 获取 kernel32.dll 中 VirtualAllocExNuma 函数的地址，并赋值给 VirtualAllocExNumaProc
    VirtualAllocExNumaProc =
    (PFN_VIRTUALALLOCEXNUMA) GetProcAddress(kernel32, "VirtualAllocExNuma");
    # 获取 kernel32.dll 中 VirtualFreeEx 函数的地址，并赋值给 VirtualFreeExProc
    VirtualFreeExProc =
    (PFN_VIRTUALFREEEX) GetProcAddress(kernel32, "VirtualFreeEx");
    }

    # 如果 QueryWorkingSetExProc 为空
    if (!QueryWorkingSetExProc) {
      # 加载 psapi.dll 库
      HMODULE psapi = LoadLibrary("psapi.dll");
      # 如果加载成功
      if (psapi)
        # 获取 psapi.dll 中 QueryWorkingSetEx 函数的地址，并赋值给 QueryWorkingSetExProc
        QueryWorkingSetExProc = (PFN_QUERYWORKINGSETEX) GetProcAddress(psapi, "QueryWorkingSetEx");
    }

    # 获取 ntdll.dll 模块的句柄
    ntdll = GetModuleHandle("ntdll");
    # 获取 ntdll.dll 中 RtlGetVersion 函数的地址，并赋值给 RtlGetVersionProc
    RtlGetVersionProc = (PFN_RTLGETVERSION) GetProcAddress(ntdll, "RtlGetVersion");
#if HWLOC_HAVE_GCC_W_CAST_FUNCTION_TYPE
#pragma GCC diagnostic warning "-Wcast-function-type"
#endif
}

/*
 * ULONG_PTR and DWORD_PTR are 64/32bits depending on the arch
 * while bitmaps use unsigned long (always 32bits)
 */

static void hwloc_bitmap_from_ULONG_PTR(hwloc_bitmap_t set, ULONG_PTR mask)
{
#if SIZEOF_VOID_P == 8
  // 如果指针大小为 8 字节，则将 ULONG_PTR 的低 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_from_ulong(set, mask & 0xffffffff);
  // 将 ULONG_PTR 的高 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_set_ith_ulong(set, 1, mask >> 32);
#else
  // 如果指针大小不为 8 字节，则直接将 ULONG_PTR 转换为无符号长整型，并设置到位图中
  hwloc_bitmap_from_ulong(set, mask);
#endif
}

static void hwloc_bitmap_from_ith_ULONG_PTR(hwloc_bitmap_t set, unsigned i, ULONG_PTR mask)
{
#if SIZEOF_VOID_P == 8
  // 如果指针大小为 8 字节，则将 ULONG_PTR 的低 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_from_ith_ulong(set, 2*i, mask & 0xffffffff);
  // 将 ULONG_PTR 的高 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_set_ith_ulong(set, 2*i+1, mask >> 32);
#else
  // 如果指针大小不为 8 字节，则直接将 ULONG_PTR 转换为无符号长整型，并设置到位图中
  hwloc_bitmap_from_ith_ulong(set, i, mask);
#endif
}

static void hwloc_bitmap_set_ith_ULONG_PTR(hwloc_bitmap_t set, unsigned i, ULONG_PTR mask)
{
#if SIZEOF_VOID_P == 8
  // 如果指针大小为 8 字节，则将 ULONG_PTR 的低 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_set_ith_ulong(set, 2*i, mask & 0xffffffff);
  // 将 ULONG_PTR 的高 32 位转换为无符号长整型，并设置到位图中
  hwloc_bitmap_set_ith_ulong(set, 2*i+1, mask >> 32);
#else
  // 如果指针大小不为 8 字节，则直接将 ULONG_PTR 转换为无符号长整型，并设置到位图中
  hwloc_bitmap_set_ith_ulong(set, i, mask);
#endif
}

static ULONG_PTR hwloc_bitmap_to_ULONG_PTR(hwloc_const_bitmap_t set)
{
#if SIZEOF_VOID_P == 8
  // 如果指针大小为 8 字节，则将位图中的第一个无符号长整型转换为 ULONG_PTR 的高 32 位
  ULONG_PTR up = hwloc_bitmap_to_ith_ulong(set, 1);
  // 将位图中的第二个无符号长整型转换为 ULONG_PTR 的低 32 位
  up <<= 32;
  up |= hwloc_bitmap_to_ulong(set);
  return up;
#else
  // 如果指针大小不为 8 字节，则直接将位图中的无符号长整型转换为 ULONG_PTR
  return hwloc_bitmap_to_ulong(set);
#endif
}

static ULONG_PTR hwloc_bitmap_to_ith_ULONG_PTR(hwloc_const_bitmap_t set, unsigned i)
{
#if SIZEOF_VOID_P == 8
  // 如果指针大小为 8 字节，则将位图中的第 2*i+1 个无符号长整型转换为 ULONG_PTR 的高 32 位
  ULONG_PTR up = hwloc_bitmap_to_ith_ulong(set, 2*i+1);
  // 将位图中的第 2*i 个无符号长整型转换为 ULONG_PTR 的低 32 位
  up <<= 32;
  up |= hwloc_bitmap_to_ith_ulong(set, 2*i);
  return up;
#else
  // 如果指针大小不为 8 字节，则直接将位图中的第 i 个无符号长整型转换为 ULONG_PTR
  return hwloc_bitmap_to_ith_ulong(set, i);
#endif
}

/* convert set into index+mask if all set bits are in the same ULONG.
 * otherwise return -1.
 */
static int hwloc_bitmap_to_single_ULONG_PTR(hwloc_const_bitmap_t set, unsigned *index, ULONG_PTR *mask)
{
  // 获取位图中第一个和最后一个非零位所在的无符号长整型的索引
  unsigned first_ulp, last_ulp;
  if (hwloc_bitmap_weight(set) == -1)
    # 返回错误代码 -1
    return -1;
  # 计算集合中第一个位的位置并转换为 ULONG_PTR 的大小
  first_ulp = hwloc_bitmap_first(set) / (sizeof(ULONG_PTR)*8);
  # 计算集合中最后一个位的位置并转换为 ULONG_PTR 的大小
  last_ulp = hwloc_bitmap_last(set) / (sizeof(ULONG_PTR)*8);
  # 如果第一个位和最后一个位所在的 ULONG_PTR 不相等，则返回错误代码 -1
  if (first_ulp != last_ulp)
    return -1;
  # 将集合中第一个位所在的 ULONG_PTR 转换为 mask，并赋值给指针变量 mask
  *mask = hwloc_bitmap_to_ith_ULONG_PTR(set, first_ulp);
  # 将第一个位所在的 ULONG_PTR 赋值给指针变量 index
  *index = first_ulp;
  # 返回成功代码 0
  return 0;
}

/**********************
 * Processor Groups
 */

static unsigned long max_numanode_index = 0;  // 初始化最大 NUMA 节点索引为 0

static unsigned long nr_processor_groups = 1;  // 初始化处理器组数量为 1
static hwloc_cpuset_t * processor_group_cpusets = NULL;  // 初始化处理器组的 CPU 集合为空

static void
hwloc_win_get_processor_groups(void)  // 定义获取处理器组信息的函数
{
  HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX *procInfoTotal, *tmpprocInfoTotal, *procInfo;  // 定义处理器信息结构体指针
  DWORD length;  // 定义长度变量
  unsigned i;  // 定义循环变量

  hwloc_debug("querying windows processor groups\n");  // 输出调试信息

  if (!GetLogicalProcessorInformationExProc)  // 如果获取处理器信息函数不存在
    goto error;  // 跳转到错误处理

  nr_processor_groups = GetActiveProcessorGroupCountProc();  // 获取活动处理器组数量
  if (!nr_processor_groups)  // 如果处理器组数量为 0
    goto error;  // 跳转到错误处理

  hwloc_debug("found %lu windows processor groups\n", nr_processor_groups);  // 输出调试信息

  if (nr_processor_groups > 1 && SIZEOF_VOID_P == 4) {  // 如果处理器组数量大于 1 且指针大小为 4
    if (HWLOC_SHOW_ALL_ERRORS())  // 如果显示所有错误
      fprintf(stderr, "hwloc: multiple processor groups found on 32bits Windows, topology may be invalid/incomplete.\n");  // 输出错误信息
  }

  length = 0;  // 初始化长度为 0
  procInfoTotal = NULL;  // 初始化处理器信息总指针为空

  while (1) {  // 进入循环
    if (GetLogicalProcessorInformationExProc(RelationGroup, procInfoTotal, &length))  // 获取逻辑处理器信息
      break;  // 如果成功则跳出循环
    if (GetLastError() != ERROR_INSUFFICIENT_BUFFER)  // 如果获取错误不是缓冲区不足
      goto error;  // 跳转到错误处理
    tmpprocInfoTotal = realloc(procInfoTotal, length);  // 重新分配处理器信息总指针的内存空间
    if (!tmpprocInfoTotal)  // 如果重新分配失败
      goto error_with_procinfo;  // 跳转到带处理器信息错误处理
    procInfoTotal = tmpprocInfoTotal;  // 更新处理器信息总指针
  }

  processor_group_cpusets = calloc(nr_processor_groups, sizeof(*processor_group_cpusets));  // 分配处理器组的 CPU 集合内存空间
  if (!processor_group_cpusets)  // 如果分配失败
    goto error_with_procinfo;  // 跳转到带处理器信息错误处理

  for (procInfo = procInfoTotal;  // 遍历处理器信息
       (void*) procInfo < (void*) ((uintptr_t) procInfoTotal + length);  // 直到处理器信息结束
       procInfo = (void*) ((uintptr_t) procInfo + procInfo->Size)) {  // 更新处理器信息指针
    unsigned id;  // 定义 ID 变量

    assert(procInfo->Relationship == RelationGroup);  // 断言处理器信息关系为处理器组

    hwloc_debug("Found %u active windows processor groups\n",
                (unsigned) procInfo->Group.ActiveGroupCount);  // 输出调试信息
    # 遍历处理器组的活动组数
    for (id = 0; id < procInfo->Group.ActiveGroupCount; id++) {
      # 定义处理器亲和性掩码和硬件位置集
      KAFFINITY mask;
      hwloc_bitmap_t set;

      # 分配硬件位置集
      set = hwloc_bitmap_alloc();
      # 如果分配失败，跳转到错误处理
      if (!set)
        goto error_with_cpusets;

      # 获取处理器组的活动处理器掩码
      mask = procInfo->Group.GroupInfo[id].ActiveProcessorMask;
      # 输出调试信息，显示处理器组的编号、活动处理器数量和掩码值
      hwloc_debug("group %u with %u cpus mask 0x%llx\n", id,
                  (unsigned) procInfo->Group.GroupInfo[id].ActiveProcessorCount, (unsigned long long) mask);
      /* KAFFINITY is ULONG_PTR */
      # 在硬件位置集中设置指定位置的 ULONG_PTR 值
      hwloc_bitmap_set_ith_ULONG_PTR(set, id, mask);
      /* FIXME: what if running 32bits on a 64bits windows with 64-processor groups?
       * ULONG_PTR is 32bits, so half the group is invisible?
       * maybe scale id to id*8/sizeof(ULONG_PTR) so that groups are 64-PU aligned?
       */
      # 输出调试信息，显示处理器组的编号、活动处理器数量和硬件位置集的位图
      hwloc_debug_2args_bitmap("group %u %d bitmap %s\n", id, procInfo->Group.GroupInfo[id].ActiveProcessorCount, set);
      # 将硬件位置集存储到处理器组硬件位置集数组中
      processor_group_cpusets[id] = set;
    }
  }

  # 释放总处理器信息的内存
  free(procInfoTotal);
  # 返回

 error_with_cpusets:
  # 在处理器组硬件位置集数组中释放已分配的硬件位置集
  for(i=0; i<nr_processor_groups; i++) {
    if (processor_group_cpusets[i])
      hwloc_bitmap_free(processor_group_cpusets[i]);
  }
  # 释放处理器组硬件位置集数组的内存
  free(processor_group_cpusets);
  processor_group_cpusets = NULL;
 error_with_procinfo:
  # 释放总处理器信息的内存
  free(procInfoTotal);
 error:
  /* on error set nr to 1 and keep cpusets NULL. We'll use the topology cpuset whenever needed */
  # 如果出现错误，将处理器组数量设置为1，并保持处理器组硬件位置集为NULL。在需要时，将使用拓扑硬件位置集
  nr_processor_groups = 1;
# 释放 Windows 系统中处理器组的资源
static void
hwloc_win_free_processor_groups(void)
{
  # 遍历处理器组，释放每个处理器组的位图资源
  unsigned i;
  for(i=0; i<nr_processor_groups; i++) {
    if (processor_group_cpusets[i])
      hwloc_bitmap_free(processor_group_cpusets[i]);
  }
  # 释放处理器组位图数组的内存
  free(processor_group_cpusets);
  processor_group_cpusets = NULL;
  # 重置处理器组数量为 1
  nr_processor_groups = 1;
}

# 获取 Windows 系统中的处理器组数量
int
hwloc_windows_get_nr_processor_groups(hwloc_topology_t topology, unsigned long flags)
{
  # 检查拓扑是否已加载且为当前系统
  if (!topology->is_loaded || !topology->is_thissystem) {
    errno = EINVAL;
    return -1;
  }

  # 检查标志是否为 0
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  # 返回处理器组数量
  return nr_processor_groups;
}

# 获取指定处理器组的位图
int
hwloc_windows_get_processor_group_cpuset(hwloc_topology_t topology, unsigned pg_index, hwloc_cpuset_t cpuset, unsigned long flags)
{
  # 检查拓扑是否已加载且为当前系统
  if (!topology->is_loaded || !topology->is_thissystem) {
    errno = EINVAL;
    return -1;
  }

  # 检查位图是否为空
  if (!cpuset) {
    errno = EINVAL;
    return -1;
  }

  # 检查标志是否为 0
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  # 检查处理器组索引是否超出范围
  if (pg_index >= nr_processor_groups) {
    errno = ENOENT;
    return -1;
  }

  # 如果处理器组位图数组为空
  if (!processor_group_cpusets) {
    assert(nr_processor_groups == 1);
    # 将整个拓扑的位图复制给传入的位图
    hwloc_bitmap_copy(cpuset, topology->levels[0][0]->cpuset);
    return 0;
  }

  # 如果指定处理器组的位图为空
  if (!processor_group_cpusets[pg_index]) {
    errno = ENOENT;
    return -1;
  }

  # 将指定处理器组的位图复制给传入的位图
  hwloc_bitmap_copy(cpuset, processor_group_cpusets[pg_index]);
  return 0;
}
/**************************************************************
 * hwloc PU numbering with respect to Windows processor groups
 *
 * Everywhere below we reserve 64 physical indexes per processor groups because that's the maximum (MAXIMUM_PROC_PER_GROUP). Windows may actually use less bits than that in some groups (either to avoid splitting NUMA nodes across groups, or because of OS tweaks such as "bcdedit /set groupsize 8") but we keep some unused indexes for simplicity.
 * That means PU physical indexes and cpusets may be non-contigous.
 * That also means hwloc_fallback_nbprocessors() below must return the last PU index + 1 instead the actual number of processors.
 */

/********************
 * last_cpu_location
 */

static int
hwloc_win_get_thisthread_last_cpu_location(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_cpuset_t set, int flags __hwloc_attribute_unused)
{
  assert(GetCurrentProcessorNumberExProc || (GetCurrentProcessorNumberProc && nr_processor_groups == 1));

  if (nr_processor_groups > 1 || !GetCurrentProcessorNumberProc) {
    PROCESSOR_NUMBER num;
    GetCurrentProcessorNumberExProc(&num);
    hwloc_bitmap_from_ith_ULONG_PTR(set, num.Group, ((ULONG_PTR)1) << num.Number);
    return 0;
  }

  hwloc_bitmap_from_ith_ULONG_PTR(set, 0, ((ULONG_PTR)1) << GetCurrentProcessorNumberProc());
  return 0;
}

/* TODO: hwloc_win_get_thisproc_last_cpu_location() using
 * CreateToolhelp32Snapshot(), Thread32First/Next()
 * th.th32OwnerProcessID == GetCurrentProcessId() for filtering within process
 * OpenThread(THREAD_SET_INFORMATION|THREAD_QUERY_INFORMATION, FALSE, te32.th32ThreadID) to get a handle.
 */


/******************************
 * set cpu/membind for threads
 */

/* TODO: SetThreadIdealProcessor{,Ex} */

static int
hwloc_win_set_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t thread, hwloc_const_bitmap_t hwloc_set, int flags)
{
  // 定义 DWORD_PTR 类型的 mask 变量
  DWORD_PTR mask;
  // 定义无符号整型的 group 变量
  unsigned group;

  // 如果 flags 中包含 HWLOC_CPUBIND_NOMEMBIND 标志位
  if (flags & HWLOC_CPUBIND_NOMEMBIND) {
    // 设置 errno 为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return -1;
  }

  // 将 hwloc_set 转换为单个 ULONG_PTR，并存储在 group 和 mask 中
  if (hwloc_bitmap_to_single_ULONG_PTR(hwloc_set, &group, &mask) < 0) {
    // 设置 errno 为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return -1;
  }

  // 断言 nr_processor_groups 等于 1 或者 SetThreadGroupAffinityProc 存在
  assert(nr_processor_groups == 1 || SetThreadGroupAffinityProc);

  // 如果 nr_processor_groups 大于 1
  if (nr_processor_groups > 1) {
    // 定义 GROUP_AFFINITY 类型的 aff 变量，并初始化为 0
    GROUP_AFFINITY aff;
    memset(&aff, 0, sizeof(aff)); /* we get Invalid Parameter error if Reserved field isn't cleared */
    // 设置 aff 的 Group 和 Mask 属性
    aff.Group = group;
    aff.Mask = mask;
    // 如果设置线程组亲和力成功，则返回 -1
    if (!SetThreadGroupAffinityProc(thread, &aff, NULL))
      return -1;

  } else {
    // SetThreadAffinityMask() 只会改变当前处理器组内的掩码
    // 结果绑定总是严格的
    // 如果设置线程亲和力掩码成功，则返回 -1
    if (!SetThreadAffinityMask(thread, mask))
      return -1;
  }
  // 返回 0
  return 0;
}

// 设置当前线程的 CPU 亲和力
static int
hwloc_win_set_thisthread_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t hwloc_set, int flags)
{
  // 调用 hwloc_win_set_thread_cpubind 函数设置当前线程的 CPU 亲和力
  return hwloc_win_set_thread_cpubind(topology, GetCurrentThread(), hwloc_set, flags);
}

// 设置当前线程的内存绑定
static int
hwloc_win_set_thisthread_membind(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  int ret;
  hwloc_const_cpuset_t cpuset;
  hwloc_cpuset_t _cpuset = NULL;

  // 如果 policy 不是 HWLOC_MEMBIND_DEFAULT 或 HWLOC_MEMBIND_BIND，或者 flags 中包含 HWLOC_MEMBIND_NOCPUBIND 标志位
  if ((policy != HWLOC_MEMBIND_DEFAULT && policy != HWLOC_MEMBIND_BIND)
      || flags & HWLOC_MEMBIND_NOCPUBIND) {
    // 设置 errno 为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return -1;
  }

  // 如果 policy 是 HWLOC_MEMBIND_DEFAULT
  if (policy == HWLOC_MEMBIND_DEFAULT) {
    // 获取 topology 的完整 CPU 集合
    cpuset = hwloc_topology_get_complete_cpuset(topology);
  } else {
    // 分配一个新的 cpuset，并根据 nodeset 设置其值
    cpuset = _cpuset = hwloc_bitmap_alloc();
    hwloc_cpuset_from_nodeset(topology, _cpuset, nodeset);
  }

  // 调用 hwloc_win_set_thisthread_cpubind 函数设置当前线程的 CPU 亲和力
  ret = hwloc_win_set_thisthread_cpubind(topology, cpuset,
                     (flags & HWLOC_MEMBIND_STRICT) ? HWLOC_CPUBIND_STRICT : 0);
  // 释放 _cpuset
  hwloc_bitmap_free(_cpuset);
  // 返回 ret
  return ret;
}


/******************************
 * get cpu/membind for threads
 */

static int
# 获取当前线程的 CPU 亲和性绑定
hwloc_win_get_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t thread, hwloc_cpuset_t set, int flags __hwloc_attribute_unused)
{
  # 定义 GROUP_AFFINITY 结构体
  GROUP_AFFINITY aff;

  # 断言 GetThreadGroupAffinityProc 函数存在
  assert(GetThreadGroupAffinityProc);

  # 如果 GetThreadGroupAffinityProc 函数成功获取当前线程的亲和性信息
  if (!GetThreadGroupAffinityProc(thread, &aff))
    # 返回 -1
    return -1;
  # 将 GROUP_AFFINITY 结构体中的 Group 和 Mask 转换为 hwloc_cpuset_t 类型的 set
  hwloc_bitmap_from_ith_ULONG_PTR(set, aff.Group, aff.Mask);
  # 返回 0
  return 0;
}

# 获取当前线程的 CPU 亲和性绑定
static int
hwloc_win_get_thisthread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_cpuset_t set, int flags __hwloc_attribute_unused)
{
  # 调用 hwloc_win_get_thread_cpubind 函数获取当前线程的 CPU 亲和性绑定
  return hwloc_win_get_thread_cpubind(topology, GetCurrentThread(), set, flags);
}

# 获取当前线程的内存绑定
static int
hwloc_win_get_thisthread_membind(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  # 定义变量 ret 用于存储返回值
  int ret;
  # 分配一个 hwloc_cpuset_t 类型的 cpuset
  hwloc_cpuset_t cpuset = hwloc_bitmap_alloc();
  # 调用 hwloc_win_get_thread_cpubind 函数获取当前线程的 CPU 亲和性绑定
  ret = hwloc_win_get_thread_cpubind(topology, GetCurrentThread(), cpuset, flags);
  # 如果成功获取 CPU 亲和性绑定
  if (!ret) {
    # 设置内存绑定策略为 HWLOC_MEMBIND_BIND
    *policy = HWLOC_MEMBIND_BIND;
    # 将 cpuset 转换为 nodeset
    hwloc_cpuset_to_nodeset(topology, cpuset, nodeset);
  }
  # 释放 cpuset
  hwloc_bitmap_free(cpuset);
  # 返回 ret
  return ret;
}

/********************************
 * 为进程设置 CPU/内存绑定
 */

# 为进程设置 CPU 亲和性绑定
static int
hwloc_win_set_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t proc, hwloc_const_bitmap_t hwloc_set, int flags)
{
  # 定义变量 mask 用于存储 CPU 亲和性掩码
  DWORD_PTR mask;

  # 断言 nr_processor_groups 等于 1
  assert(nr_processor_groups == 1);

  # 如果 flags 包含 HWLOC_CPUBIND_NOMEMBIND
  if (flags & HWLOC_CPUBIND_NOMEMBIND) {
    # 设置 errno 为 ENOSYS
    errno = ENOSYS;
    // 返回-1表示出错
    return -1;
  }

  /* TODO: SetThreadGroupAffinity() for all threads doesn't enforce the whole process affinity,
   * maybe because of process-specific resource locality */
  /* TODO: if we are in a single group (check with GetProcessGroupAffinity()),
   * SetProcessAffinityMask() changes the binding within that same group.
   */
  /* TODO: NtSetInformationProcess() works very well for binding to any mask in a single group,
   * but it's an internal routine.
   */
  /* TODO: checks whether hwloc-bind.c needs to pass INHERIT_PARENT_AFFINITY to CreateProcess() instead of execvp(). */

  /* The resulting binding is always strict */
  // 将 hwloc_set 中的位图转换为 ULONG_PTR 类型的掩码
  mask = hwloc_bitmap_to_ULONG_PTR(hwloc_set);
  // 如果设置进程的亲和性掩码失败，则返回-1
  if (!SetProcessAffinityMask(proc, mask))
    return -1;
  // 设置成功，返回0
  return 0;
# 设置当前进程的 CPU 亲和性
static int
hwloc_win_set_thisproc_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t hwloc_set, int flags)
{
  # 调用 hwloc_win_set_proc_cpubind 函数设置当前进程的 CPU 亲和性
  return hwloc_win_set_proc_cpubind(topology, GetCurrentProcess(), hwloc_set, flags);
}

# 设置指定进程的内存绑定
static int
hwloc_win_set_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  int ret;
  hwloc_const_cpuset_t cpuset;
  hwloc_cpuset_t _cpuset = NULL;

  # 如果策略不是默认或者绑定，或者标志包含 HWLOC_MEMBIND_NOCPUBIND
  if ((policy != HWLOC_MEMBIND_DEFAULT && policy != HWLOC_MEMBIND_BIND)
      || flags & HWLOC_MEMBIND_NOCPUBIND) {
    # 设置错误码为不支持的系统调用
    errno = ENOSYS;
    return -1;
  }

  # 如果策略是默认
  if (policy == HWLOC_MEMBIND_DEFAULT) {
    # 获取拓扑结构中的完整 CPU 集合
    cpuset = hwloc_topology_get_complete_cpuset(topology);
  } else {
    # 否则，从节点集合创建 CPU 集合
    cpuset = _cpuset = hwloc_bitmap_alloc();
    hwloc_cpuset_from_nodeset(topology, _cpuset, nodeset);
  }

  # 调用 hwloc_win_set_proc_cpubind 函数设置指定进程的 CPU 亲和性
  ret = hwloc_win_set_proc_cpubind(topology, pid, cpuset,
                   (flags & HWLOC_MEMBIND_STRICT) ? HWLOC_CPUBIND_STRICT : 0);
  # 释放 CPU 集合
  hwloc_bitmap_free(_cpuset);
  return ret;
}

# 设置当前进程的内存绑定
static int
hwloc_win_set_thisproc_membind(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 调用 hwloc_win_set_proc_membind 函数设置当前进程的内存绑定
  return hwloc_win_set_proc_membind(topology, GetCurrentProcess(), nodeset, policy, flags);
}


/********************************
 * 获取进程的 CPU/内存绑定
 */

# 获取指定进程的 CPU 亲和性
static int
hwloc_win_get_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t proc, hwloc_bitmap_t hwloc_set, int flags)
{
  # 获取进程掩码和系统掩码
  DWORD_PTR proc_mask, sys_mask;

  # 断言处理器组数为 1
  assert(nr_processor_groups == 1);

  # 如果标志包含 HWLOC_CPUBIND_NOMEMBIND
  if (flags & HWLOC_CPUBIND_NOMEMBIND) {
    # 设置错误码为不支持的系统调用
    errno = ENOSYS;
    // 返回-1，表示出错
    return -1;
  }

  /* TODO: 如果我们在一个单一的组中（使用GetProcessGroupAffinity()检查），
   * GetProcessAffinityMask()会给出该组内的掩码。
   */
  /* TODO: 如果我们在多个组中，GetProcessGroupAffinity()会给出它们的ID，
   * 但我们不知道它们的掩码。
   */
  /* TODO: 对于所有线程的GetThreadGroupAffinity()可能比整个进程的亲和力小，
   * 可能是因为进程特定的资源局部性。
   */

  // 如果无法获取进程的亲和力掩码，则返回-1
  if (!GetProcessAffinityMask(proc, &proc_mask, &sys_mask))
    return -1;
  // 将proc_mask转换为hwloc_set
  hwloc_bitmap_from_ULONG_PTR(hwloc_set, proc_mask);
  // 返回0，表示成功
  return 0;
}

static int
hwloc_win_get_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  int ret;
  hwloc_cpuset_t cpuset = hwloc_bitmap_alloc();  // 分配一个位图用于表示 CPU 核心集合
  ret = hwloc_win_get_proc_cpubind(topology, pid, cpuset,  // 获取指定进程的 CPU 绑定情况
                   (flags & HWLOC_MEMBIND_STRICT) ? HWLOC_CPUBIND_STRICT : 0);
  if (!ret) {
    *policy = HWLOC_MEMBIND_BIND;  // 如果成功获取 CPU 绑定情况，则设置内存绑定策略为 BIND
    hwloc_cpuset_to_nodeset(topology, cpuset, nodeset);  // 将 CPU 核心集合转换为 NUMA 节点集合
  }
  hwloc_bitmap_free(cpuset);  // 释放之前分配的 CPU 核心集合位图
  return ret;  // 返回获取内存绑定情况的结果
}

static int
hwloc_win_get_thisproc_cpubind(hwloc_topology_t topology, hwloc_bitmap_t hwloc_cpuset, int flags)
{
  return hwloc_win_get_proc_cpubind(topology, GetCurrentProcess(), hwloc_cpuset, flags);  // 获取当前进程的 CPU 绑定情况
}

static int
hwloc_win_get_thisproc_membind(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_win_get_proc_membind(topology, GetCurrentProcess(), nodeset, policy, flags);  // 获取当前进程的内存绑定情况
}


/************************
 * membind alloc/free
 */

static void *
hwloc_win_alloc(hwloc_topology_t topology __hwloc_attribute_unused, size_t len) {
  return VirtualAlloc(NULL, len, MEM_COMMIT|MEM_RESERVE, PAGE_EXECUTE_READWRITE);  // 在 Windows 上分配内存
}

static void *
hwloc_win_alloc_membind(hwloc_topology_t topology __hwloc_attribute_unused, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) {
  int node;

  switch (policy) {
    case HWLOC_MEMBIND_DEFAULT:
    case HWLOC_MEMBIND_BIND:
      break;
    default:
      errno = ENOSYS;  // 设置错误码为不支持的系统调用
      return hwloc_alloc_or_fail(topology, len, flags);  // 分配内存或者返回错误
  }

  if (flags & HWLOC_MEMBIND_STRICT) {
    errno = ENOSYS;  // 设置错误码为不支持的系统调用
    return NULL;  // 返回空指针
  }

  if (policy == HWLOC_MEMBIND_DEFAULT
      || hwloc_bitmap_isequal(nodeset, hwloc_topology_get_complete_nodeset(topology)))
    return hwloc_win_alloc(topology, len);  // 如果内存绑定策略为默认或者节点集合与整个系统节点集合相等，则在 Windows 上分配内存

  if (hwloc_bitmap_weight(nodeset) != 1) {
    /* Not a single node, can't do this */
    errno = EXDEV;  // 设置错误码为跨设备链接
  # 调用 hwloc_alloc_or_fail 函数分配内存，并返回分配的内存地址
  return hwloc_alloc_or_fail(topology, len, flags);
}

# 从 NUMA 节点集中获取第一个节点的编号
node = hwloc_bitmap_first(nodeset);
# 在指定的 NUMA 节点上为当前进程分配指定大小的虚拟内存
return VirtualAllocExNumaProc(GetCurrentProcess(), NULL, len, MEM_COMMIT|MEM_RESERVE, PAGE_EXECUTE_READWRITE, node);
# 释放 Windows 平台上的内存绑定
static int
hwloc_win_free_membind(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len __hwloc_attribute_unused) {
  # 如果地址为空，则直接返回 0
  if (!addr)
    return 0;
  # 调用 VirtualFreeExProc 函数释放内存
  if (!VirtualFreeExProc(GetCurrentProcess(), addr, 0, MEM_RELEASE))
    return -1;
  # 返回 0 表示成功
  return 0;
}

# 获取内存区域的内存位置
static int
hwloc_win_get_area_memlocation(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr, size_t len, hwloc_nodeset_t nodeset, int flags __hwloc_attribute_unused)
{
  # 定义变量
  SYSTEM_INFO SystemInfo;
  DWORD page_size;
  uintptr_t start;
  unsigned nb;
  PSAPI_WORKING_SET_EX_INFORMATION *pv;
  unsigned i;

  # 获取系统信息
  GetSystemInfo(&SystemInfo);
  page_size = SystemInfo.dwPageSize;

  # 计算起始地址和页数
  start = (((uintptr_t) addr) / page_size) * page_size;
  nb = (unsigned)((((uintptr_t) addr + len - start) + page_size - 1) / page_size);

  # 如果页数为 0，则设置为 1
  if (!nb)
    nb = 1;

  # 分配内存
  pv = calloc(nb, sizeof(*pv));
  # 如果分配失败，则返回 -1
  if (!pv)
    return -1;

  # 循环设置虚拟地址
  for (i = 0; i < nb; i++)
    pv[i].VirtualAddress = (void*) (start + i * page_size);
  # 查询工作集
  if (!QueryWorkingSetExProc(GetCurrentProcess(), pv, nb * sizeof(*pv))) {
    free(pv);
    return -1;
  }

  # 遍历工作集，设置节点集合
  for (i = 0; i < nb; i++) {
    if (pv[i].VirtualAttributes.Valid)
      hwloc_bitmap_set(nodeset, pv[i].VirtualAttributes.Node);
  }

  # 释放内存
  free(pv);
  # 返回 0 表示成功
  return 0;
}

# Windows 平台效率类
struct hwloc_win_efficiency_classes {
  unsigned nr_classes;
  unsigned nr_classes_allocated;
  struct hwloc_win_efficiency_class {
    unsigned value;
    hwloc_bitmap_t cpuset;
  } *classes;
};

# 初始化效率类
static void
hwloc_win_efficiency_classes_init(struct hwloc_win_efficiency_classes *classes)
{
  classes->classes = NULL;
  classes->nr_classes_allocated = 0;
  classes->nr_classes = 0;
}

# 添加效率类
static int
hwloc_win_efficiency_classes_add(struct hwloc_win_efficiency_classes *classes,
                                 hwloc_const_bitmap_t cpuset,
                                 unsigned value)
{
  unsigned i;

  /* 查找具有该效率值的现有类 */
  for(i=0; i<classes->nr_classes; i++) {
    if (classes->classes[i].value == value) {
      hwloc_bitmap_or(classes->classes[i].cpuset, classes->classes[i].cpuset, cpuset);
      return 0;
    }
  }

  /* 如果需要，扩展数组 */
  if (classes->nr_classes == classes->nr_classes_allocated) {
    struct hwloc_win_efficiency_class *tmp;
    unsigned new_nr_allocated = 2*classes->nr_classes_allocated;
    if (!new_nr_allocated) {
#define HWLOC_WIN_EFFICIENCY_CLASSES_DEFAULT_MAX 4 /* 在大多数情况下，2应该足够了 */
      new_nr_allocated = HWLOC_WIN_EFFICIENCY_CLASSES_DEFAULT_MAX;
    }
    tmp = realloc(classes->classes, new_nr_allocated * sizeof(*classes->classes));
    if (!tmp)
      return -1;
    classes->classes = tmp;
    classes->nr_classes_allocated = new_nr_allocated;
  }

  /* 添加新类 */
  classes->classes[classes->nr_classes].cpuset = hwloc_bitmap_alloc();
  if (!classes->classes[classes->nr_classes].cpuset)
    return -1;
  classes->classes[classes->nr_classes].value = value;
  hwloc_bitmap_copy(classes->classes[classes->nr_classes].cpuset, cpuset);
  classes->nr_classes++;
  return 0;
}

static void
hwloc_win_efficiency_classes_register(hwloc_topology_t topology,
                                      struct hwloc_win_efficiency_classes *classes)
{
  unsigned i;
  for(i=0; i<classes->nr_classes; i++) {
    hwloc_internal_cpukinds_register(topology, classes->classes[i].cpuset, classes->classes[i].value, NULL, 0, 0);
    classes->classes[i].cpuset = NULL; /* 给予 cpukinds */
  }
}

static void
hwloc_win_efficiency_classes_destroy(struct hwloc_win_efficiency_classes *classes)
{
  unsigned i;
  for(i=0; i<classes->nr_classes; i++)
    hwloc_bitmap_free(classes->classes[i].cpuset);
  free(classes->classes);
}

/*************************
 * discovery
 */

static int
hwloc_look_windows(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  /*
   * This backend uses the underlying OS.
   * However we don't enforce topology->is_thissystem so that
   * we may still force use this backend when debugging with !thissystem.
   */

  // 定义指向拓扑结构的指针
  struct hwloc_topology *topology = backend->topology;
  // 定义位图集合指针
  hwloc_bitmap_t groups_pu_set = NULL;
  // 定义系统信息结构体
  SYSTEM_INFO SystemInfo;
  // 定义DWORD类型的变量
  DWORD length;
  // 初始化变量
  int gotnuma = 0;
  int gotnumamemory = 0;
  // 定义操作系统版本信息结构体
  OSVERSIONINFOEX osvi;
  // 定义字符数组
  char versionstr[20];
  char hostname[122] = "";
  // 定义无符号整型变量
  unsigned hostname_size = sizeof(hostname);
  // 初始化变量
  int has_efficiencyclass = 0;
  // 定义 Windows 系统效率类结构体
  struct hwloc_win_efficiency_classes eclasses;
  // 获取环境变量
  char *env = getenv("HWLOC_WINDOWS_PROCESSOR_GROUP_OBJS");
  // 根据环境变量判断是否保留处理器组对象
  int keep_pgroup_objs = (env && atoi(env));

  // 断言当前状态为 CPU 相关的发现阶段
  assert(dstatus->phase == HWLOC_DISC_PHASE_CPU);

  // 如果拓扑结构的第一层第一个对象的 cpuset 不为空，则返回错误
  if (topology->levels[0][0]->cpuset)
    /* somebody discovered things */
    return -1;

  // 清空操作系统版本信息结构体
  ZeroMemory(&osvi, sizeof(OSVERSIONINFOEX));
  osvi.dwOSVersionInfoSize = sizeof(OSVERSIONINFOEX);

  // 如果存在 RtlGetVersionProc 函数，则调用该函数获取当前运行的 Windows 版本
  if (RtlGetVersionProc) {
    /* RtlGetVersion() returns the currently-running Windows version */
    RtlGetVersionProc(&osvi);
  } else {
    /* GetVersionEx() and isWindows10OrGreater() depend on what the manifest says
     * (manifest of the program, not of libhwloc.dll), they may return old versions
     * if the currently-running Windows is not listed in the manifest.
     */
    // 否则调用 GetVersionEx() 函数获取当前运行的 Windows 版本
    GetVersionEx((LPOSVERSIONINFO)&osvi);
  }

  // 如果操作系统主版本号大于等于 10，则设置效率类标志为 1，并初始化 Windows 系统效率类
  if (osvi.dwMajorVersion >= 10) {
    has_efficiencyclass = 1;
    hwloc_win_efficiency_classes_init(&eclasses);
  }

  // 分配根集合
  hwloc_alloc_root_sets(topology->levels[0][0]);

  // 获取系统信息
  GetSystemInfo(&SystemInfo);

  // 如果存在 GetLogicalProcessorInformationExProc 函数
  if (GetLogicalProcessorInformationExProc) {
      HWLOC_SYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX *procInfoTotal, *tmpprocInfoTotal, *procInfo;
      unsigned id;
      struct hwloc_obj *obj;
      hwloc_obj_type_t type;

      length = 0;
      procInfoTotal = NULL;

      // 循环获取逻辑处理器信息
      while (1) {
    if (GetLogicalProcessorInformationExProc(RelationAll, procInfoTotal, &length))
      break;
    # 如果上一个系统调用返回的错误码不是 ERROR_INSUFFICIENT_BUFFER，则返回 -1
    if (GetLastError() != ERROR_INSUFFICIENT_BUFFER)
      return -1;
    # 重新分配内存以扩展 procInfoTotal 数组的大小
    tmpprocInfoTotal = realloc(procInfoTotal, length);
    # 如果分配失败，则释放原有内存，跳转到 out 标签处
    if (!tmpprocInfoTotal) {
      free(procInfoTotal);
      goto out;
    }
    # 更新 procInfoTotal 指向新分配的内存
    procInfoTotal = tmpprocInfoTotal;

    # 遍历 procInfoTotal 数组
    for (procInfo = procInfoTotal;
       (void*) procInfo < (void*) ((uintptr_t) procInfoTotal + length);
       procInfo = (void*) ((uintptr_t) procInfo + procInfo->Size)) {
        unsigned num, i;
        unsigned efficiency_class = 0;
        GROUP_AFFINITY *GroupMask;

        # 如果 procInfo 的关系是 RelationCache，且缓存类型不是 CacheUnified、CacheData、CacheInstruction，则跳过本次循环
        if (procInfo->Relationship == RelationCache
            && procInfo->Cache.Type != CacheUnified
            && procInfo->Cache.Type != CacheData
            && procInfo->Cache.Type != CacheInstruction)
          continue;

        id = HWLOC_UNKNOWN_INDEX;

        # 如果不符合过滤器要求，则跳过本次循环
        if (!hwloc_filter_check_keep_object_type(topology, type))
          continue;

        # 分配并初始化一个 hwloc 对象
        obj = hwloc_alloc_setup_object(topology, type, id);
        # 分配一个位图用于表示对象的 cpuset
        obj->cpuset = hwloc_bitmap_alloc();
        # 遍历 GroupMask 数组
        for (i = 0; i < num; i++) {
          # 打印调试信息
          hwloc_debug("%s#%u %d: mask %d:%lx\n", hwloc_obj_type_string(type), id, i, GroupMask[i].Group, GroupMask[i].Mask);
          # 设置 cpuset 的第 i 位为 GroupMask[i].Mask
          hwloc_bitmap_set_ith_ULONG_PTR(obj->cpuset, GroupMask[i].Group, GroupMask[i].Mask);
          # FIXME: 将 id 缩放为 id*8/sizeof(ULONG_PTR)？
        }
        # 打印调试信息
        hwloc_debug_2args_bitmap("%s#%u bitmap %s\n", hwloc_obj_type_string(type), id, obj->cpuset);
    }
    # 根据类型进行不同的操作
    switch (type) {
        # 如果是核心对象
        case HWLOC_OBJ_CORE: {
          # 如果有效率类别，将对象的 cpuset 和效率类别添加到效率类别集合中
          if (has_efficiencyclass)
            hwloc_win_efficiency_classes_add(&eclasses, obj->cpuset, efficiency_class);
          # 结束当前 case
          break;
        }
        # 如果是 NUMA 节点对象
        case HWLOC_OBJ_NUMANODE:
        {
          # 定义 avail 变量
          ULONGLONG avail;
          # 为对象的 nodeset 分配内存
          obj->nodeset = hwloc_bitmap_alloc();
          # 将 id 添加到对象的 nodeset 中
          hwloc_bitmap_set(obj->nodeset, id);
          # 如果获取 NUMA 节点可用内存成功
          if ((GetNumaAvailableMemoryNodeExProc && GetNumaAvailableMemoryNodeExProc(id, &avail))
          || (GetNumaAvailableMemoryNodeProc && GetNumaAvailableMemoryNodeProc(id, &avail))) {
            # 设置对象的本地内存属性
            obj->attr->numanode.local_memory = avail;
            # 增加获取到 NUMA 内存的计数
            gotnumamemory++;
          }
          # 为对象的 page_types 分配内存
          obj->attr->numanode.page_types = malloc(2 * sizeof(*obj->attr->numanode.page_types));
          # 将分配的内存初始化为 0
          memset(obj->attr->numanode.page_types, 0, 2 * sizeof(*obj->attr->numanode.page_types));
          # 设置 page_types_len 为 1
          obj->attr->numanode.page_types_len = 1;
          # 设置 page_types[0] 的大小为系统页面大小
          obj->attr->numanode.page_types[0].size = SystemInfo.dwPageSize;
#if HAVE_DECL__SC_LARGE_PAGESIZE
          // 如果定义了 _SC_LARGE_PAGESIZE，增加 NUMA 节点的页类型长度
          obj->attr->numanode.page_types_len++;
          // 设置第二个页类型的大小为系统定义的大页大小
          obj->attr->numanode.page_types[1].size = sysconf(_SC_LARGE_PAGESIZE);
#endif
          // 结束条件
          break;
        }
      case HWLOC_OBJ_L1CACHE:
      case HWLOC_OBJ_L2CACHE:
      case HWLOC_OBJ_L3CACHE:
      case HWLOC_OBJ_L4CACHE:
      case HWLOC_OBJ_L5CACHE:
      case HWLOC_OBJ_L1ICACHE:
      case HWLOC_OBJ_L2ICACHE:
      case HWLOC_OBJ_L3ICACHE:
        // 设置缓存大小
        obj->attr->cache.size = procInfo->Cache.CacheSize;
        // 设置缓存关联度
        obj->attr->cache.associativity = procInfo->Cache.Associativity == CACHE_FULLY_ASSOCIATIVE ? -1 : procInfo->Cache.Associativity ;
        // 设置缓存行大小
        obj->attr->cache.linesize = procInfo->Cache.LineSize;
        // 设置缓存深度
        obj->attr->cache.depth = procInfo->Cache.Level;
        switch (procInfo->Cache.Type) {
          case CacheUnified:
        // 设置缓存类型为统一缓存
        obj->attr->cache.type = HWLOC_OBJ_CACHE_UNIFIED;
        break;
          case CacheData:
        // 设置缓存类型为数据缓存
        obj->attr->cache.type = HWLOC_OBJ_CACHE_DATA;
        break;
          case CacheInstruction:
        // 设置缓存类型为指令缓存
        obj->attr->cache.type = HWLOC_OBJ_CACHE_INSTRUCTION;
        break;
          default:
        // 释放未连接的对象并继续循环
        hwloc_free_unlinked_object(obj);
        continue;
        }
        // 结束条件
        break;
      default:
        // 结束条件
        break;
    }
    // 将对象插入拓扑结构
    hwloc__insert_object_by_cpuset(topology, NULL, obj, "windows:GetLogicalProcessorInformationEx");
      }
      // 释放 procInfoTotal 内存
      free(procInfoTotal);
  }

  // 设置拓扑结构支持的发现类型
  topology->support.discovery->pu = 1;
  topology->support.discovery->numa = gotnuma;
  topology->support.discovery->numa_memory = gotnumamemory;

  if (groups_pu_set) {
    /* the system supports multiple Groups.
     * PU indexes may be discontiguous, especially if Groups contain less than 64 procs.
     */
    // 声明对象和索引
    hwloc_obj_t obj;
    unsigned idx;
    # 遍历处理器组的每个处理器
    hwloc_bitmap_foreach_begin(idx, groups_pu_set) {
      # 为处理器创建 PU 对象
      obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, idx);
      # 为 PU 对象分配位图
      obj->cpuset = hwloc_bitmap_alloc();
      # 将处理器的索引添加到 PU 对象的位图中
      hwloc_bitmap_only(obj->cpuset, idx);
      # 输出调试信息，显示处理器的位图
      hwloc_debug_1arg_bitmap("cpu %u has cpuset %s\n",
                  idx, obj->cpuset);
      # 将 PU 对象插入到拓扑结构中
      hwloc__insert_object_by_cpuset(topology, NULL, obj, "windows:ProcessorGroup:pu");
    } hwloc_bitmap_foreach_end();
    # 释放处理器组的位图
    hwloc_bitmap_free(groups_pu_set);
  } else {
    # 如果没有处理器组
    hwloc_obj_t obj;
    unsigned idx;
    # 遍历每个处理器
    for(idx=0; idx<32; idx++)
      # 如果处理器在活动处理器掩码中
      if (SystemInfo.dwActiveProcessorMask & (((DWORD_PTR)1)<<idx)) {
    # 为处理器创建 PU 对象
    obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, idx);
    # 为 PU 对象分配位图
    obj->cpuset = hwloc_bitmap_alloc();
    # 将处理器的索引添加到 PU 对象的位图中
    hwloc_bitmap_only(obj->cpuset, idx);
    # 输出调试信息，显示处理器的位图
    hwloc_debug_1arg_bitmap("cpu %u has cpuset %s\n",
                idx, obj->cpuset);
    # 将 PU 对象插入到拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, obj, "windows:pu");
      }
  }

  # 如果有效率类
  if (has_efficiencyclass) {
    # 设置拓扑结构支持发现的 CPU 类型为有效率类
    topology->support.discovery->cpukind_efficiency = 1;
    # 注册有效率类
    hwloc_win_efficiency_classes_register(topology, &eclasses);
  }

  # 释放有效率类
  out:
  if (has_efficiencyclass)
    hwloc_win_efficiency_classes_destroy(&eclasses);

  # 模拟 uname 而不是调用 hwloc_add_uname_info()
  # 添加 Windows 操作系统信息到拓扑结构的根对象
  hwloc_obj_add_info(topology->levels[0][0], "Backend", "Windows");
  hwloc_obj_add_info(topology->levels[0][0], "OSName", "Windows");
#if defined(__CYGWIN__)
  # 如果是在 CYGWIN 环境下编译，添加 WindowsBuildEnvironment 信息为 Cygwin
  hwloc_obj_add_info(topology->levels[0][0], "WindowsBuildEnvironment", "Cygwin");
#elif defined(__MINGW32__)
  # 如果是在 MINGW32 环境下编译，添加 WindowsBuildEnvironment 信息为 MinGW
  hwloc_obj_add_info(topology->levels[0][0], "WindowsBuildEnvironment", "MinGW");
#endif

  /* 查看操作系统版本信息，根据不同版本添加对应的 OSRelease 信息 */
  if (osvi.dwMajorVersion == 10) {
    if (osvi.dwMinorVersion == 0)
      hwloc_obj_add_info(topology->levels[0][0], "OSRelease", "10");
  } else if (osvi.dwMajorVersion == 6) {
    if (osvi.dwMinorVersion == 3)
      hwloc_obj_add_info(topology->levels[0][0], "OSRelease", "8.1"); /* 或者 "Server 2012 R2" */
    else if (osvi.dwMinorVersion == 2)
      hwloc_obj_add_info(topology->levels[0][0], "OSRelease", "8"); /* 或者 "Server 2012" */
    else if (osvi.dwMinorVersion == 1)
      hwloc_obj_add_info(topology->levels[0][0], "OSRelease", "7"); /* 或者 "Server 2008 R2" */
    else if (osvi.dwMinorVersion == 0)
      hwloc_obj_add_info(topology->levels[0][0], "OSRelease", "Vista"); /* 或者 "Server 2008" */
  } /* 忽略更早的版本 */

  // 将操作系统版本号转换为字符串，添加 OSVersion 信息
  snprintf(versionstr, sizeof(versionstr), "%u.%u.%u", osvi.dwMajorVersion, osvi.dwMinorVersion, osvi.dwBuildNumber);
  hwloc_obj_add_info(topology->levels[0][0], "OSVersion", versionstr);

#if !defined(__CYGWIN__)
  // 如果不是在 CYGWIN 环境下，获取计算机名并添加 Hostname 信息
  GetComputerName(hostname, &hostname_size);
#else
  // 如果是在 CYGWIN 环境下，获取主机名并添加 Hostname 信息
  gethostname(hostname, hostname_size);
#endif
  // 如果主机名不为空，添加 Hostname 信息
  if (*hostname)
    hwloc_obj_add_info(topology->levels[0][0], "Hostname", hostname);

  /* 将系统架构转换为类 Unix 的架构字符串，并添加 Architecture 信息 */
  switch (SystemInfo.wProcessorArchitecture) {
  case 0:
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "i686");
    break;
  case 9:
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "x86_64");
    break;
  case 5:
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "arm");
    break;
  case 12:
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "arm64");
    break;
  case 6:
  // 添加其他架构信息
    # 给定的拓扑结构中的第一级第一个对象添加信息，指定架构为ia64
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "ia64");
    # 结束switch语句
    break;
  }
  # 返回0，表示执行成功
  return 0;
# 设置 Windows 特定的绑定钩子
void
hwloc_set_windows_hooks(struct hwloc_binding_hooks *hooks,
            struct hwloc_topology_support *support)
{
  # 如果存在 GetCurrentProcessorNumberExProc 函数或者 (GetCurrentProcessorNumberProc 函数和 nr_processor_groups 等于 1)
  if (GetCurrentProcessorNumberExProc || (GetCurrentProcessorNumberProc && nr_processor_groups == 1))
    # 设置获取当前线程最后 CPU 位置的钩子函数
    hooks->get_thisthread_last_cpu_location = hwloc_win_get_thisthread_last_cpu_location;

  # 如果 nr_processor_groups 等于 1
  if (nr_processor_groups == 1) {
    # 设置进程 CPU 绑定的钩子函数
    hooks->set_proc_cpubind = hwloc_win_set_proc_cpubind;
    # 设置获取进程 CPU 绑定的钩子函数
    hooks->get_proc_cpubind = hwloc_win_get_proc_cpubind;
    # 设置当前进程 CPU 绑定的钩子函数
    hooks->set_thisproc_cpubind = hwloc_win_set_thisproc_cpubind;
    # 设置获取当前进程 CPU 绑定的钩子函数
    hooks->get_thisproc_cpubind = hwloc_win_get_thisproc_cpubind;
    # 设置进程内存绑定的钩子函数
    hooks->set_proc_membind = hwloc_win_set_proc_membind;
    # 设置获取进程内存绑定的钩子函数
    hooks->get_proc_membind = hwloc_win_get_proc_membind;
    # 设置当前进程内存绑定的钩子函数
    hooks->set_thisproc_membind = hwloc_win_set_thisproc_membind;
    # 设置获取当前进程内存绑定的钩子函数
    hooks->get_thisproc_membind = hwloc_win_get_thisproc_membind;
  }
  # 如果 nr_processor_groups 等于 1 或者存在 SetThreadGroupAffinityProc 函数
  if (nr_processor_groups == 1 || SetThreadGroupAffinityProc) {
    # 设置线程 CPU 绑定的钩子函数
    hooks->set_thread_cpubind = hwloc_win_set_thread_cpubind;
    # 设置当前线程 CPU 绑定的钩子函数
    hooks->set_thisthread_cpubind = hwloc_win_set_thisthread_cpubind;
    # 设置当前线程内存绑定的钩子函数
    hooks->set_thisthread_membind = hwloc_win_set_thisthread_membind;
  }
  # 如果存在 GetThreadGroupAffinityProc 函数
  if (GetThreadGroupAffinityProc) {
    # 设置获取线程 CPU 绑定的钩子函数
    hooks->get_thread_cpubind = hwloc_win_get_thread_cpubind;
    # 设置获取当前线程 CPU 绑定的钩子函数
    hooks->get_thisthread_cpubind = hwloc_win_get_thisthread_cpubind;
    # 设置获取当前线程内存绑定的钩子函数
    hooks->get_thisthread_membind = hwloc_win_get_thisthread_membind;
  }

  # 如果存在 VirtualAllocExNumaProc 函数
  if (VirtualAllocExNumaProc) {
    # 设置内存绑定的分配函数
    hooks->alloc_membind = hwloc_win_alloc_membind;
    # 设置内存分配函数
    hooks->alloc = hwloc_win_alloc;
    # 设置内存绑定的释放函数
    hooks->free_membind = hwloc_win_free_membind;
    # 设置支持内存绑定
    support->membind->bind_membind = 1;
  }

  # 如果存在 QueryWorkingSetExProc 函数并且 max_numanode_index 小于等于 63
  if (QueryWorkingSetExProc && max_numanode_index <= 63 /* PSAPI_WORKING_SET_EX_BLOCK.Node is 6 bits only */)
    # 设置获取区域内存位置的钩子函数
    hooks->get_area_memlocation = hwloc_win_get_area_memlocation;
}

# Windows 组件初始化函数
static int hwloc_windows_component_init(unsigned long flags __hwloc_attribute_unused)
{
  # 获取 Windows 相关函数指针
  hwloc_win_get_function_ptrs();
  # 获取处理器组信息
  hwloc_win_get_processor_groups();
  # 返回 0 表示初始化成功
  return 0;
}
static void hwloc_windows_component_finalize(unsigned long flags __hwloc_attribute_unused)
{
  // 释放处理器组资源
  hwloc_win_free_processor_groups();
}

static struct hwloc_backend *
hwloc_windows_component_instantiate(struct hwloc_topology *topology,
                    struct hwloc_disc_component *component,
                    unsigned excluded_phases __hwloc_attribute_unused,
                    const void *_data1 __hwloc_attribute_unused,
                    const void *_data2 __hwloc_attribute_unused,
                    const void *_data3 __hwloc_attribute_unused)
{
  // 分配一个新的后端对象
  struct hwloc_backend *backend;
  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    return NULL;
  // 设置后端对象的发现函数为 hwloc_look_windows
  backend->discover = hwloc_look_windows;
  return backend;
}

// 定义 Windows 系统发现组件
static struct hwloc_disc_component hwloc_windows_disc_component = {
  "windows", // 组件名称
  HWLOC_DISC_PHASE_CPU, // CPU 相关发现阶段
  HWLOC_DISC_PHASE_GLOBAL, // 全局相关发现阶段
  hwloc_windows_component_instantiate, // 实例化函数
  50, // 优先级
  1, // 是否可重复
  NULL // 额外数据
};

// 定义 Windows 系统组件
const struct hwloc_component hwloc_windows_component = {
  HWLOC_COMPONENT_ABI, // 组件 ABI 版本
  hwloc_windows_component_init, // 初始化函数
  hwloc_windows_component_finalize, // 终结函数
  HWLOC_COMPONENT_TYPE_DISC, // 组件类型为发现
  0, // 不需要额外数据
  &hwloc_windows_disc_component // 指向发现组件的指针
};

// 获取处理器数量的回退函数
int
hwloc_fallback_nbprocessors(unsigned flags __hwloc_attribute_unused) {
  int n;
  SYSTEM_INFO sysinfo;

  // TODO 处理 flags & HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE

  // 默认情况下，忽略处理器组（仅返回当前组中的处理器数量）
  GetSystemInfo(&sysinfo);
  n = sysinfo.dwNumberOfProcessors; // 可能是非连续的，最好从 dwActiveProcessorMask 返回掩码？
  
  if (nr_processor_groups > 1) {
    // 假设 n-1 组是完整的，因为这是我们在 cpusets 中存储的方式
    if (GetActiveProcessorCountProc)
      n = MAXIMUM_PROC_PER_GROUP*(nr_processor_groups-1)
    + GetActiveProcessorCountProc((WORD)nr_processor_groups-1);
    else
      n = MAXIMUM_PROC_PER_GROUP*nr_processor_groups;
  }

  return n;
}

// 获取内存大小的回退函数
int64_t
hwloc_fallback_memsize(void) {
  // 未使用
  return -1;
}
```