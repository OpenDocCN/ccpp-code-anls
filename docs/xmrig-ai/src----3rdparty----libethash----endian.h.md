# `xmrig\src\3rdparty\libethash\endian.h`

```
#pragma once

#include <stdint.h>

#if defined(__MINGW32__) || defined(_WIN32)
  # define LITTLE_ENDIAN 1234  // 定义小端字节序
  # define BYTE_ORDER    LITTLE_ENDIAN  // 定义字节序为小端
#elif defined(__FreeBSD__) || defined(__DragonFly__) || defined(__NetBSD__)
  # include <sys/endian.h>  // 包含系统字节序头文件
#elif defined(__OpenBSD__) || defined(__SVR4)
  # include <sys/types.h>  // 包含系统类型头文件
#elif defined(__APPLE__)
# include <machine/endian.h>  // 包含机器字节序头文件
#elif defined( BSD ) && (BSD >= 199103)
  # include <machine/endian.h>  // 包含机器字节序头文件
#elif defined( __QNXNTO__ ) && defined( __LITTLEENDIAN__ )
  # define LITTLE_ENDIAN 1234  // 定义小端字节序
  # define BYTE_ORDER    LITTLE_ENDIAN  // 定义字节序为小端
#elif defined( __QNXNTO__ ) && defined( __BIGENDIAN__ )
  # define BIG_ENDIAN 1234  // 定义大端字节序
  # define BYTE_ORDER BIG_ENDIAN  // 定义字节序为大端
#else
# include <endian.h>  // 包含字节序头文件
#endif

#if defined(_WIN32)
#include <stdlib.h>
#define ethash_swap_u32(input_) _byteswap_ulong(input_)  // 定义 32 位整数的字节序交换函数
#define ethash_swap_u64(input_) _byteswap_uint64(input_)  // 定义 64 位整数的字节序交换函数
#elif defined(__APPLE__)
#include <libkern/OSByteOrder.h>
#define ethash_swap_u32(input_) OSSwapInt32(input_)  // 定义 32 位整数的字节序交换函数
#define ethash_swap_u64(input_) OSSwapInt64(input_)  // 定义 64 位整数的字节序交换函数
#elif defined(__FreeBSD__) || defined(__DragonFly__) || defined(__NetBSD__)
#define ethash_swap_u32(input_) bswap32(input_)  // 定义 32 位整数的字节序交换函数
#define ethash_swap_u64(input_) bswap64(input_)  // 定义 64 位整数的字节序交换函数
#elif defined(__OpenBSD__)
#include <endian.h>
#define ethash_swap_u32(input_) swap32(input_)  // 定义 32 位整数的字节序交换函数
#define ethash_swap_u64(input_) swap64(input_)  // 定义 64 位整数的字节序交换函数
#else // posix
#include <byteswap.h>
#define ethash_swap_u32(input_) bswap_32(input_)  // 定义 32 位整数的字节序交换函数
#define ethash_swap_u64(input_) bswap_64(input_)  // 定义 64 位整数的字节序交换函数
#endif


#if LITTLE_ENDIAN == BYTE_ORDER

#define fix_endian32(dst_ ,src_) dst_ = src_  // 如果字节序为小端，则不需要进行字节序转换
#define fix_endian32_same(val_)  // 如果字节序为小端，则不需要进行字节序转换
#define fix_endian64(dst_, src_) dst_ = src_  // 如果字节序为小端，则不需要进行字节序转换
#define fix_endian64_same(val_)  // 如果字节序为小端，则不需要进行字节序转换
#define fix_endian_arr32(arr_, size_)  // 如果字节序为小端，则不需要进行字节序转换
#define fix_endian_arr64(arr_, size_)  // 如果字节序为小端，则不需要进行字节序转换

#elif BIG_ENDIAN == BYTE_ORDER

#define fix_endian32(dst_, src_) dst_ = ethash_swap_u32(src_)  // 如果字节序为大端，则进行 32 位整数的字节序转换
#define fix_endian32_same(val_) val_ = ethash_swap_u32(val_)  // 如果字节序为大端，则进行 32 位整数的字节序转换
#define fix_endian64(dst_, src_) dst_ = ethash_swap_u64(src_)  // 如果字节序为大端，则进行 64 位整数的字节序转换
#ifdef BYTE_ORDER
// 如果是大端字节序，定义一个宏用于修复64位整数的字节序
#define fix_endian64_same(val_) val_ = ethash_swap_u64(val_)
// 如果是大端字节序，定义一个宏用于修复32位整数数组的字节序
#define fix_endian_arr32(arr_, size_) \
  do { \
    for (unsigned i_ = 0; i_ < (size_); ++i_) { \
      arr_[i_] = ethash_swap_u32(arr_[i_]); \
    } \
  } while (0)
// 如果是大端字节序，定义一个宏用于修复64位整数数组的字节序
#define fix_endian_arr64(arr_, size_) \
  do { \
    for (unsigned i_ = 0; i_ < (size_); ++i_) { \
      arr_[i_] = ethash_swap_u64(arr_[i_]); \
    } \
  } while (0)
#else
// 如果不支持当前字节序，输出错误信息
# error "endian not supported"
#endif // BYTE_ORDER
```