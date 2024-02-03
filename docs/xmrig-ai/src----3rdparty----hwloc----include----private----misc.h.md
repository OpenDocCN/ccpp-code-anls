# `xmrig\src\3rdparty\hwloc\include\private\misc.h`

```cpp
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2019 Inria
 * 版权所有 © 2009-2012 Université Bordeaux
 * 版权所有 © 2011 Cisco Systems, Inc.
 * 请查看顶层目录中的 COPYING 文件。
 */

/* 杂项宏和内联函数 */

#ifndef HWLOC_PRIVATE_MISC_H
#define HWLOC_PRIVATE_MISC_H

#include "hwloc/autogen/config.h"
#include "private/autogen/config.h"
#include "hwloc.h"

#ifdef HWLOC_HAVE_DECL_STRNCASECMP
#ifdef HAVE_STRINGS_H
#include <strings.h>
#endif
#else
#ifdef HAVE_CTYPE_H
#include <ctype.h>
#endif
#endif

#define HWLOC_BITS_PER_LONG (HWLOC_SIZEOF_UNSIGNED_LONG * 8)
#define HWLOC_BITS_PER_INT (HWLOC_SIZEOF_UNSIGNED_INT * 8)

#if (HWLOC_BITS_PER_LONG != 32) && (HWLOC_BITS_PER_LONG != 64)
#error "未知的 unsigned long 大小。"
#endif

#if (HWLOC_BITS_PER_INT != 16) && (HWLOC_BITS_PER_INT != 32) && (HWLOC_BITS_PER_INT != 64)
#error "未知的 unsigned int 大小。"
#endif

/* 仅内部使用的值，当我们不知道类型或没有任何值时使用 */
#define HWLOC_OBJ_TYPE_NONE ((hwloc_obj_type_t) -1)

/**
 * ffsl 辅助函数
 */

#if defined(HWLOC_HAVE_BROKEN_FFS)

/* 系统具有损坏的 ffs()。
 * 我们必须在 __GNUC__ 或 HWLOC_HAVE_FFSL 之前检查
 */
#    define HWLOC_NO_FFS

#elif defined(__GNUC__)

#  if (__GNUC__ >= 4) || ((__GNUC__ == 3) && (__GNUC_MINOR__ >= 4))
     /* 从 3.4 开始，gcc 有一个长整型变体。 */
#    define hwloc_ffsl(x) __builtin_ffsl(x)
#  else
#    define hwloc_ffs(x) __builtin_ffs(x)
#    define HWLOC_NEED_FFSL
#  endif

#elif defined(HWLOC_HAVE_FFSL)

#  ifndef HWLOC_HAVE_DECL_FFSL
extern int ffsl(long) __hwloc_attribute_const;
#  endif

#  define hwloc_ffsl(x) ffsl(x)

#elif defined(HWLOC_HAVE_FFS)

#  ifndef HWLOC_HAVE_DECL_FFS
extern int ffs(int) __hwloc_attribute_const;
#  endif

#  define hwloc_ffs(x) ffs(x)
#  define HWLOC_NEED_FFSL

#else /* 没有 ffs 实现 */

#    define HWLOC_NO_FFS

#endif

#ifdef HWLOC_NO_FFS

/* 没有 ffs 或者已知它是损坏的 */
# 定义一个内联函数，计算一个无符号长整型数的最低有效位的位置
static __hwloc_inline int
hwloc_ffsl_manual(unsigned long x) __hwloc_attribute_const;
static __hwloc_inline int
hwloc_ffsl_manual(unsigned long x)
{
    int i;

    # 如果 x 为 0，则返回 0
    if (!x)
        return 0;

    # 初始化 i 为 1
    i = 1;
    # 如果长整型数的位数大于等于 64
    if (HWLOC_BITS_PER_LONG >= 64)
    {
        # 如果 x 的低 32 位全为 0
        if (!(x & 0xfffffffful)) {
            x >>= 32;
            i += 32;
        }
    }
    # 如果 x 的低 16 位全为 0
    if (!(x & 0xffffu)) {
        x >>= 16;
        i += 16;
    }
    # 如果 x 的低 8 位全为 0
    if (!(x & 0xff)) {
        x >>= 8;
        i += 8;
    }
    # 如果 x 的低 4 位全为 0
    if (!(x & 0xf)) {
        x >>= 4;
        i += 4;
    }
    # 如果 x 的低 2 位全为 0
    if (!(x & 0x3)) {
        x >>= 2;
        i += 2;
    }
    # 如果 x 的最低位为 0
    if (!(x & 0x1)) {
        x >>= 1;
        i += 1;
    }

    # 返回最低有效位的位置
    return i;
}
# 总是将 hwloc_ffsl 定义为宏，以避免重命名破坏
#define hwloc_ffsl hwloc_ffsl_manual

#elif defined(HWLOC_NEED_FFSL)

# 如果只有一个 int ffs(int) 实现，构建一个 long 类型的实现
# 首先将其转换为 32 位
static __hwloc_inline int
hwloc_ffs32(unsigned long x) __hwloc_attribute_const;
static __hwloc_inline int
hwloc_ffs32(unsigned long x)
{
    # 如果 int 类型为 16 位
    # 计算低 16 位和高 16 位的最低有效位位置
    # 如果低 16 位的最低有效位位置不为 0，则返回该位置
    # 否则，计算高 16 位的最低有效位位置，返回该位置加上 16
    # 如果都为 0，则返回 0
#if HWLOC_BITS_PER_INT == 16
    int low_ffs, hi_ffs;

    low_ffs = hwloc_ffs(x & 0xfffful);
    if (low_ffs)
        return low_ffs;

    hi_ffs = hwloc_ffs(x >> 16);
    if (hi_ffs)
        return hi_ffs + 16;

    return 0;
#else
    return hwloc_ffs(x);
#endif
}

# 然后将其转换为 64 位
static __hwloc_inline int
hwloc_ffsl_from_ffs32(unsigned long x) __hwloc_attribute_const;
static __hwloc_inline int
hwloc_ffsl_from_ffs32(unsigned long x)
{
    # 如果 long 类型为 64 位
    # 计算低 32 位和高 32 位的最低有效位位置
    # 如果低 32 位的最低有效位位置不为 0，则返回该位置
    # 否则，计算高 32 位的最低有效位位置，返回该位置加上 32
    # 如果都为 0，则返回 0
#if HWLOC_BITS_PER_LONG == 64
    int low_ffs, hi_ffs;

    low_ffs = hwloc_ffs32(x & 0xfffffffful);
    if (low_ffs)
        return low_ffs;

    hi_ffs = hwloc_ffs32(x >> 32);
    if (hi_ffs)
        return hi_ffs + 32;

    return 0;
#else
    return hwloc_ffs32(x);
#endif
}
# 总是将 hwloc_ffsl 定义为宏，以避免重命名破坏
#define hwloc_ffsl hwloc_ffsl_from_ffs32

#endif

/**
 * flsl helpers.
 */
# 如果是 GNU 编译器
#ifdef __GNUC_____
# 如果编译器版本大于等于4，或者版本为3且次版本号大于等于4，则定义 hwloc_flsl 为一个宏，返回参数 x 的最高位的位数
# 否则，定义 hwloc_fls 为一个宏，返回参数 x 的最高位的位数，并定义 HWLOC_NEED_FLSL
#elif defined(HWLOC_HAVE_FLSL)
# 如果没有定义 HWLOC_HAVE_DECL_FLSL，则声明一个返回参数为 long 类型的函数 flsl
extern int flsl(long) __hwloc_attribute_const;
# 将 hwloc_flsl 定义为调用 flsl 函数
# 如果没有定义 HWLOC_HAVE_DECL_FLSL，则声明一个返回参数为 long 类型的函数 flsl
#elif defined(HWLOC_HAVE_CLZL)
extern int clzl(long) __hwloc_attribute_const;
# 将 hwloc_flsl 定义为一个宏，返回参数 x 的最高位的位数
# 如果没有定义 HWLOC_HAVE_DECL_CLZL，则声明一个返回参数为 long 类型的函数 clzl
#elif defined(HWLOC_HAVE_FLS)
extern int fls(int) __hwloc_attribute_const;
# 将 hwloc_fls 定义为调用 fls 函数
# 定义 HWLOC_NEED_FLSL
# 如果没有定义 HWLOC_HAVE_DECL_FLS，则声明一个返回参数为 int 类型的函数 fls
#elif defined(HWLOC_HAVE_CLZ)
extern int clz(int) __hwloc_attribute_const;
# 将 hwloc_fls 定义为一个宏，返回参数 x 的最高位的位数
# 定义 HWLOC_NEED_FLSL
# 如果没有定义 HWLOC_HAVE_DECL_CLZ，则声明一个返回参数为 int 类型的函数 clz
# 如果没有找到 fls 的实现，则定义一个内联函数 hwloc_flsl_manual，返回参数 x 的最高位的位数
# 将 hwloc_flsl 定义为 hwloc_flsl_manual 的宏
#else /* no fls implementation */
# 如果需要定义 HWLOC_NEED_FLSL
# 如果只有一个 int fls(int) 实现，构建一个 long 类型的实现
# 如果原本是 16 位，将其扩展为 32 位
static __hwloc_inline int
hwloc_fls32(unsigned long x) __hwloc_attribute_const;
# 定义一个内联函数，用于返回32位整数的最高位索引

static __hwloc_inline int
hwloc_fls32(unsigned long x)
{
#if HWLOC_BITS_PER_INT == 16
    int low_fls, hi_fls;
    # 定义两个变量用于存储低16位和高16位的最高位索引

    hi_fls = hwloc_fls(x >> 16);
    # 计算高16位的最高位索引
    if (hi_fls)
        return hi_fls + 16;
    # 如果高16位的最高位索引存在，则返回索引值加16

    low_fls = hwloc_fls(x & 0xfffful);
    # 计算低16位的最高位索引
    if (low_fls)
        return low_fls;
    # 如果低16位的最高位索引存在，则返回索引值

    return 0;
    # 如果都不存在最高位索引，则返回0
#else
    return hwloc_fls(x);
    # 如果不是16位整数，则调用hwloc_fls函数返回最高位索引
#endif
}

/* Then make it 64 bit if longs are.  */
# 如果长整型是64位，则将其扩展为64位

static __hwloc_inline int
hwloc_flsl_from_fls32(unsigned long x) __hwloc_attribute_const;
# 定义一个内联函数，用于返回64位长整型的最高位索引

static __hwloc_inline int
hwloc_flsl_from_fls32(unsigned long x)
{
#if HWLOC_BITS_PER_LONG == 64
    int low_fls, hi_fls;
    # 定义两个变量用于存储低32位和高32位的最高位索引

    hi_fls = hwloc_fls32(x >> 32);
    # 计算高32位的最高位索引
    if (hi_fls)
        return hi_fls + 32;
    # 如果高32位的最高位索引存在，则返回索引值加32

    low_fls = hwloc_fls32(x & 0xfffffffful);
    # 计算低32位的最高位索引
    if (low_fls)
        return low_fls;
    # 如果低32位的最高位索引存在，则返回索引值

    return 0;
    # 如果都不存在最高位索引，则返回0
#else
    return hwloc_fls32(x);
    # 如果不是64位长整型，则调用hwloc_fls32函数返回最高位索引
#endif
}
/* always define hwloc_flsl as a macro, to avoid renaming breakage */
# 始终将hwloc_flsl定义为宏，以避免重命名破坏
#define hwloc_flsl hwloc_flsl_from_fls32
# 将hwloc_flsl定义为hwloc_flsl_from_fls32

#endif

static __hwloc_inline int
hwloc_weight_long(unsigned long w) __hwloc_attribute_const;
# 定义一个内联函数，用于返回长整型的权重

static __hwloc_inline int
hwloc_weight_long(unsigned long w)
{
#if HWLOC_BITS_PER_LONG == 32
#if (__GNUC__ >= 4) || ((__GNUC__ == 3) && (__GNUC_MINOR__) >= 4)
    return __builtin_popcount(w);
    # 如果长整型是32位，并且编译器版本大于等于4.3，则使用内置函数__builtin_popcount计算权重
#else
    unsigned int res = (w & 0x55555555) + ((w >> 1) & 0x55555555);
    res = (res & 0x33333333) + ((res >> 2) & 0x33333333);
    res = (res & 0x0F0F0F0F) + ((res >> 4) & 0x0F0F0F0F);
    res = (res & 0x00FF00FF) + ((res >> 8) & 0x00FF00FF);
    return (res & 0x0000FFFF) + ((res >> 16) & 0x0000FFFF);
    # 如果编译器版本不满足要求，则手动计算权重
#endif
#else /* HWLOC_BITS_PER_LONG == 32 */
#if (__GNUC__ >= 4) || ((__GNUC__ == 3) && (__GNUC_MINOR__) >= 4)
    return __builtin_popcountll(w);
    # 如果长整型不是32位，并且编译器版本大于等于4.3，则使用内置函数__builtin_popcountll计算权重
#else
    unsigned long res;
    res = (w & 0x5555555555555555ul) + ((w >> 1) & 0x5555555555555555ul);
    res = (res & 0x3333333333333333ul) + ((res >> 2) & 0x3333333333333333ul);
    res = (res & 0x0F0F0F0F0F0F0F0Ful) + ((res >> 4) & 0x0F0F0F0F0F0F0F0Ful);
    # 如果编译器版本不满足要求，则手动计算权重
    # 将 res 的每个相邻的 8 位相加，并将结果存储回 res
    res = (res & 0x00FF00FF00FF00FFul) + ((res >> 8) & 0x00FF00FF00FF00FFul);
    # 将 res 的每个相邻的 16 位相加，并将结果存储回 res
    res = (res & 0x0000FFFF0000FFFFul) + ((res >> 16) & 0x0000FFFF0000FFFFul);
    # 将 res 的每个相邻的 32 位相加，并将结果返回
    return (res & 0x00000000FFFFFFFFul) + ((res >> 32) & 0x00000000FFFFFFFFul);
#endif
#endif /* HWLOC_BITS_PER_LONG == 64 */
}

#if !HAVE_DECL_STRTOULL && defined(HAVE_STRTOULL)
unsigned long long int strtoull(const char *nptr, char **endptr, int base);
#endif

static __hwloc_inline int hwloc_strncasecmp(const char *s1, const char *s2, size_t n)
{
#ifdef HWLOC_HAVE_DECL_STRNCASECMP
  return strncasecmp(s1, s2, n);  // 如果系统支持 strncasecmp 函数，则直接调用该函数进行字符串比较
#else
  while (n) {
    char c1 = tolower(*s1), c2 = tolower(*s2);  // 将字符串转换为小写字符
    if (!c1 || !c2 || c1 != c2)  // 如果两个字符不相等或者其中一个字符为空
      return c1-c2;  // 返回两个字符的差值
    n--; s1++; s2++;  // 继续比较下一个字符
  }
  return 0;  // 字符串相等，返回 0
#endif
}

static __hwloc_inline hwloc_obj_type_t hwloc_cache_type_by_depth_type(unsigned depth, hwloc_obj_cache_type_t type)
{
  if (type == HWLOC_OBJ_CACHE_INSTRUCTION) {
    if (depth >= 1 && depth <= 3)
      return HWLOC_OBJ_L1ICACHE + depth-1;  // 返回对应的指令缓存类型
    else
      return HWLOC_OBJ_TYPE_NONE;  // 返回无效的缓存类型
  } else {
    if (depth >= 1 && depth <= 5)
      return HWLOC_OBJ_L1CACHE + depth-1;  // 返回对应的缓存类型
    else
      return HWLOC_OBJ_TYPE_NONE;  // 返回无效的缓存类型
  }
}

#define HWLOC_BITMAP_EQUAL 0       /* Bitmaps are equal */
#define HWLOC_BITMAP_INCLUDED 1    /* First bitmap included in second */
#define HWLOC_BITMAP_CONTAINS 2    /* First bitmap contains second */
#define HWLOC_BITMAP_INTERSECTS 3  /* Bitmaps intersect without any inclusion */
#define HWLOC_BITMAP_DIFFERENT  4  /* Bitmaps do not intersect */

/* Compare bitmaps \p bitmap1 and \p bitmap2 from an inclusion point of view. */
HWLOC_DECLSPEC int hwloc_bitmap_compare_inclusion(hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;

/* Return a stringified PCI class. */
HWLOC_DECLSPEC extern const char * hwloc_pci_class_string(unsigned short class_id);

/* Parse a PCI link speed (GT/s) string from Linux sysfs */
#ifdef HWLOC_LINUX_SYS
#include <stdlib.h> /* for atof() */
static __hwloc_inline float
hwloc_linux_pci_link_speed_from_string(const char *string)
{
  /* don't parse Gen1 with atof() since it expects a localized string
   * while the kernel sysfs files aren't.
   */
  if (!strncmp(string, "2.5 ", 4))  // 如果字符串以 "2.5 " 开头
    # 返回 Gen1 的数据传输速率，使用 8/10 编码
    return 2.5 * .8;

  # 如果字符串以 "5 " 开头，则表示 Gen2，也使用特定的编码
  if (!strncmp(string, "5 ", 2))
    # 返回 Gen2 的数据传输速率，使用 8/10 编码
    return 5 * .8;

  # 以通用方式处理 Gen3+ 的情况
  # 将字符串转换为浮点数，然后使用 128/130 编码计算数据传输速率
  return atof(string) * 128./130; # Gen3+ 编码是 128/130
/* Traverse children of a parent */
#define for_each_child(child, parent) for(child = parent->first_child; child; child = child->next_sibling)
#define for_each_memory_child(child, parent) for(child = parent->memory_first_child; child; child = child->next_sibling)
#define for_each_io_child(child, parent) for(child = parent->io_first_child; child; child = child->next_sibling)
#define for_each_misc_child(child, parent) for(child = parent->misc_first_child; child; child = child->next_sibling)

/* 定义宏，用于遍历父对象的子对象，包括所有类型的子对象 */
/* 定义宏，用于遍历父对象的内存子对象 */
/* 定义宏，用于遍历父对象的 I/O 子对象 */
/* 定义宏，用于遍历父对象的其他类型子对象 */

/* Any object attached to normal children */
static __hwloc_inline int hwloc__obj_type_is_normal (hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return type <= HWLOC_OBJ_GROUP || type == HWLOC_OBJ_DIE;
}

/* 判断对象类型是否为普通类型，即不包含内存、I/O、特殊类型的对象 */
/* 通过检查对象类型的范围来判断 */

/* Any object attached to memory children, currently NUMA nodes or Memory-side caches */
static __hwloc_inline int hwloc__obj_type_is_memory (hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return type == HWLOC_OBJ_NUMANODE || type == HWLOC_OBJ_MEMCACHE;
}

/* 判断对象类型是否为内存类型，目前包括 NUMA 节点和内存侧缓存 */
/* 通过检查对象类型的范围来判断 */

/* I/O or Misc object, without cpusets or nodesets. */
static __hwloc_inline int hwloc__obj_type_is_special (hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return type >= HWLOC_OBJ_BRIDGE && type <= HWLOC_OBJ_MISC;
}

/* 判断对象类型是否为特殊类型，即 I/O 或其他类型，不包含 CPU 集或节点集 */
/* 通过检查对象类型的范围来判断 */

/* Any object attached to io children */
static __hwloc_inline int hwloc__obj_type_is_io (hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return type >= HWLOC_OBJ_BRIDGE && type <= HWLOC_OBJ_OS_DEVICE;
}

/* 判断对象类型是否为 I/O 类型，包括桥接设备到操作系统设备 */
/* 通过检查对象类型的范围来判断 */

/* Any CPU caches (not Memory-side caches) */
static __hwloc_inline int
hwloc__obj_type_is_cache(hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return (type >= HWLOC_OBJ_L1CACHE && type <= HWLOC_OBJ_L3ICACHE);
}

/* 判断对象类型是否为 CPU 缓存（不包括内存侧缓存） */
/* 通过检查对象类型的范围来判断 */

static __hwloc_inline int
hwloc__obj_type_is_dcache(hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  return (type >= HWLOC_OBJ_L1CACHE && type <= HWLOC_OBJ_L5CACHE);
}

/* 判断对象类型是否为数据缓存 */
/* 通过检查对象类型的范围来判断 */
/** \brief Check whether an object is a Instruction Cache. */
static __hwloc_inline int
hwloc__obj_type_is_icache(hwloc_obj_type_t type)
{
  /* type contiguity is asserted in topology_check() */
  // 检查对象类型是否为指令缓存
  return (type >= HWLOC_OBJ_L1ICACHE && type <= HWLOC_OBJ_L3ICACHE);
}

#ifdef HAVE_USELOCALE
#include "locale.h"
#ifdef HAVE_XLOCALE_H
#include "xlocale.h"
#endif
// 声明本地化切换变量
#define hwloc_localeswitch_declare locale_t __old_locale = (locale_t)0, __new_locale
// 初始化本地化切换
#define hwloc_localeswitch_init() do {                     \
  __new_locale = newlocale(LC_ALL_MASK, "C", (locale_t)0); \
  if (__new_locale != (locale_t)0)                         \
    __old_locale = uselocale(__new_locale);                \
} while (0)
// 结束本地化切换
#define hwloc_localeswitch_fini() do { \
  if (__new_locale != (locale_t)0) {   \
    uselocale(__old_locale);           \
    freelocale(__new_locale);          \
  }                                    \
} while(0)
#else /* HAVE_USELOCALE */
#if HWLOC_HAVE_ATTRIBUTE_UNUSED
#define hwloc_localeswitch_declare int __dummy_nolocale __hwloc_attribute_unused
#define hwloc_localeswitch_init()
#else
#define hwloc_localeswitch_declare int __dummy_nolocale
#define hwloc_localeswitch_init() (void)__dummy_nolocale
#endif
#define hwloc_localeswitch_fini()
#endif /* HAVE_USELOCALE */

#if !HAVE_DECL_FABSF
#define fabsf(f) fabs((double)(f))
#endif

#if !HAVE_DECL_MODFF
#define modff(x,iptr) (float)modf((double)x,(double *)iptr)
#endif

#if HAVE_DECL__SC_PAGE_SIZE
#define hwloc_getpagesize() sysconf(_SC_PAGE_SIZE)
#elif HAVE_DECL__SC_PAGESIZE
#define hwloc_getpagesize() sysconf(_SC_PAGESIZE)
#elif defined HAVE_GETPAGESIZE
#define hwloc_getpagesize() getpagesize()
#else
#undef hwloc_getpagesize
#endif

#if HWLOC_HAVE_ATTRIBUTE_FORMAT
#  define __hwloc_attribute_format(type, str, arg)  __attribute__((__format__(type, str, arg)))
#else
#  define __hwloc_attribute_format(type, str, arg)
#endif
#define hwloc_memory_size_printf_value(_size, _verbose) \ 
  // 根据内存大小和是否详细输出，返回适合打印的值
  ((_size) < (10ULL<<20) || (_verbose) ? (((_size)>>9)+1)>>1 : (_size) < (10ULL<<30) ? (((_size)>>19)+1)>>1 : (_size) < (10ULL<<40) ? (((_size)>>29)+1)>>1 : (((_size)>>39)+1)>>1)
#define hwloc_memory_size_printf_unit(_size, _verbose) \
  // 根据内存大小和是否详细输出，返回适合打印的单位
  ((_size) < (10ULL<<20) || (_verbose) ? "KB" : (_size) < (10ULL<<30) ? "MB" : (_size) < (10ULL<<40) ? "GB" : "TB")

#ifdef HWLOC_WIN_SYS
#  ifndef HAVE_SSIZE_T
typedef SSIZE_T ssize_t;
#  endif
#  if !HAVE_DECL_STRTOULL && !defined(HAVE_STRTOULL)
#    define strtoull _strtoui64
#  endif
#  ifndef S_ISREG
#    define S_ISREG(m) ((m) & S_IFREG)
#  endif
#  ifndef S_ISDIR
#    define S_ISDIR(m) (((m) & S_IFMT) == S_IFDIR)
#  endif
#  ifndef S_IRWXU
#    define S_IRWXU 00700
#  endif
#  ifndef HWLOC_HAVE_DECL_STRCASECMP
#    define strcasecmp _stricmp
#  endif
#  if !HAVE_DECL_SNPRINTF
#    define snprintf _snprintf
#  endif
#  if HAVE_DECL__STRDUP
#    define strdup _strdup
#  endif
#  if HAVE_DECL__PUTENV
#    define putenv _putenv
#  endif
#endif

#endif /* HWLOC_PRIVATE_MISC_H */
```