# `xmrig\src\backend\opencl\cl\kawpow\defs.h`

```
#ifdef cl_clang_storage_class_specifiers
#pragma OPENCL EXTENSION cl_clang_storage_class_specifiers : enable
#endif

#ifndef GROUP_SIZE
#define GROUP_SIZE 256
#endif
#define GROUP_SHARE (GROUP_SIZE / 16)

typedef unsigned int       uint32_t;  // 定义无符号 32 位整数类型
typedef unsigned long      uint64_t;  // 定义无符号 64 位长整数类型
#define ROTL32(x, n) rotate((x), (uint32_t)(n))  // 定义 32 位左循环移位操作
#define ROTR32(x, n) rotate((x), (uint32_t)(32-n))  // 定义 32 位右循环移位操作

#define PROGPOW_LANES           16  // 定义 PROGPOW 算法中的并行计算通道数
#define PROGPOW_REGS            32  // 定义 PROGPOW 算法中的寄存器数
#define PROGPOW_DAG_LOADS       4   // 定义 PROGPOW 算法中的 DAG 加载数
#define PROGPOW_CACHE_WORDS     4096  // 定义 PROGPOW 算法中的缓存字数
#define PROGPOW_CNT_DAG         64  // 定义 PROGPOW 算法中的 DAG 计数
#define PROGPOW_CNT_MATH        18  // 定义 PROGPOW 算法中的数学计数

#define OPENCL_PLATFORM_UNKNOWN 0  // 定义未知的 OpenCL 平台
#define OPENCL_PLATFORM_NVIDIA 1  // 定义 NVIDIA 的 OpenCL 平台
#define OPENCL_PLATFORM_AMD 2  // 定义 AMD 的 OpenCL 平台
#define OPENCL_PLATFORM_CLOVER 3  // 定义 CLOVER 的 OpenCL 平台

#ifndef MAX_OUTPUTS
#define MAX_OUTPUTS 63U  // 如果未定义最大输出数，则设置最大输出数为 63
#endif

#ifndef PLATFORM
#ifdef cl_amd_media_ops
#define PLATFORM OPENCL_PLATFORM_AMD  // 如果未定义平台，则根据是否支持 AMD 媒体操作来定义平台
#else
#define PLATFORM OPENCL_PLATFORM_UNKNOWN  // 如果不支持 AMD 媒体操作，则定义平台为未知
#endif
#endif

#define HASHES_PER_GROUP (GROUP_SIZE / PROGPOW_LANES)  // 计算每个组的哈希数
#define FNV_PRIME 0x1000193  // 定义 FNV 哈希算法的质数
#define FNV_OFFSET_BASIS 0x811c9dc5  // 定义 FNV 哈希算法的偏移基数
```