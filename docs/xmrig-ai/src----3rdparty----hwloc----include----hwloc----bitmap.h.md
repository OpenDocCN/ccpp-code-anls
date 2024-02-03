# `xmrig\src\3rdparty\hwloc\include\hwloc\bitmap.h`

```cpp
/*
 * 版权声明，版权归 CNRS 所有
 * 版权声明，版权归 Inria 所有，2009-2022年
 * 版权声明，版权归 Université Bordeaux 所有，2009-2012年
 * 版权声明，版权归 Cisco Systems, Inc. 所有，2009-2011年
 * 请查看顶层目录下的 COPYING 文件
 */

/** \file
 * \brief 位图 API，用于在 hwloc 中使用
 */

#ifndef HWLOC_BITMAP_H
#define HWLOC_BITMAP_H

#include "hwloc/autogen/config.h"

#include <assert.h>


#ifdef __cplusplus
extern "C" {
#endif
/**
 * \defgroup hwlocality_bitmap The bitmap API
 *
 * The ::hwloc_bitmap_t type represents a set of integers (positive or null).
 * A bitmap may be of infinite size (all bits are set after some point).
 * A bitmap may even be full if all bits are set.
 *
 * Bitmaps are used by hwloc for sets of OS processors
 * (which may actually be hardware threads) as by ::hwloc_cpuset_t
 * (a typedef for ::hwloc_bitmap_t), or sets of NUMA memory nodes
 * as ::hwloc_nodeset_t (also a typedef for ::hwloc_bitmap_t).
 * Those are used for cpuset and nodeset fields in the ::hwloc_obj structure,
 * see \ref hwlocality_object_sets.
 *
 * <em>Both CPU and node sets are always indexed by OS physical number.</em>
 * However users should usually not build CPU and node sets manually
 * (e.g. with hwloc_bitmap_set()).
 * One should rather use existing object sets and combine them with
 * hwloc_bitmap_or(), etc.
 * For instance, binding the current thread on a pair of cores may be performed with:
 * \code
 * hwloc_obj_t core1 = ... , core2 = ... ;
 * hwloc_bitmap_t set = hwloc_bitmap_alloc();
 * hwloc_bitmap_or(set, core1->cpuset, core2->cpuset);
 * hwloc_set_cpubind(topology, set, HWLOC_CPUBIND_THREAD);
 * hwloc_bitmap_free(set);
 * \endcode
 *
 * \note Most functions below return an int that may be negative in case of
 * error. The usual error case would be an internal failure to realloc/extend
 * the storage of the bitmap (\p errno would be set to \c ENOMEM).
 *
 * \note Several examples of using the bitmap API are available under the
 * doc/examples/ directory in the source tree.
 * Regression tests such as tests/hwloc/hwloc_bitmap*.c also make intensive use
 * of this API.
 * @{
 */


/** \brief
 * Set of bits represented as an opaque pointer to an internal bitmap.
 */
typedef struct hwloc_bitmap_s * hwloc_bitmap_t;
/** \brief a non-modifiable ::hwloc_bitmap_t */
typedef const struct hwloc_bitmap_s * hwloc_const_bitmap_t;


/*
 * Bitmap allocation, freeing and copying.
 */
# 分配一个新的空位图
# 返回一个有效的位图或 NULL
# 位图应该通过调用 hwloc_bitmap_free() 来释放
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_alloc(void) __hwloc_attribute_malloc;

# 分配一个新的完整位图
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_alloc_full(void) __hwloc_attribute_malloc;

# 释放位图 bitmap
# 如果 bitmap 是 NULL，则不执行任何操作
HWLOC_DECLSPEC void hwloc_bitmap_free(hwloc_bitmap_t bitmap);

# 通过分配一个新的位图并复制 bitmap 的内容来复制位图 bitmap
# 如果 bitmap 是 NULL，则返回 NULL
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_dup(hwloc_const_bitmap_t bitmap) __hwloc_attribute_malloc;

# 将位图 src 的内容复制到已分配的位图 dst
HWLOC_DECLSPEC int hwloc_bitmap_copy(hwloc_bitmap_t dst, hwloc_const_bitmap_t src);

# 将位图转换为字符串
# 最多可以写入 buflen 个字符到缓冲区 buf
# 如果 buflen 为 0，则 buf 可以安全地为 NULL
# 如果没有截断，则返回实际写入的字符数，不包括结尾的 \0
HWLOC_DECLSPEC int hwloc_bitmap_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

# 将位图转换为新分配的字符串
# 出错时返回 -1
HWLOC_DECLSPEC int hwloc_bitmap_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

# 解析位图字符串并将其存储在位图 bitmap 中
HWLOC_DECLSPEC int hwloc_bitmap_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);
/** \brief 将位图转换为列表格式的字符串。
 *
 * 列表是逗号分隔的索引或范围。
 * 范围是用破折号分隔的索引。
 * 如果位图是无限设置的，则最后一个范围可能没有结束索引。
 *
 * 最多可以在缓冲区 \p buf 中写入 \p buflen 个字符。
 *
 * 如果 \p buflen 为 0，则 \p buf 可以安全地为 \c NULL。
 *
 * \return 如果没有截断，则实际写入的字符数，或者将要写入的字符数（不包括结尾的 \\0）。
 */
HWLOC_DECLSPEC int hwloc_bitmap_list_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

/** \brief 将位图转换为新分配的列表字符串。
 *
 * \return 出错时返回 -1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_list_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

/** \brief 解析列表字符串并将其存储在位图 \p bitmap 中。
 */
HWLOC_DECLSPEC int hwloc_bitmap_list_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);

/** \brief 将位图转换为特定于 taskset 的格式的字符串。
 *
 * taskset 命令操作包含以 0x 开头的单个（可能非常长的）十六进制数字的位图字符串。
 *
 * 最多可以在缓冲区 \p buf 中写入 \p buflen 个字符。
 *
 * 如果 \p buflen 为 0，则 \p buf 可以安全地为 \c NULL。
 *
 * \return 如果没有截断，则实际写入的字符数，或者将要写入的字符数（不包括结尾的 \\0）。
 */
HWLOC_DECLSPEC int hwloc_bitmap_taskset_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

/** \brief 将位图转换为新分配的特定于 taskset 的字符串。
 *
 * \return 出错时返回 -1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_taskset_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

/** \brief 解析特定于 taskset 的位图字符串并将其存储在位图 \p bitmap 中。
 */
HWLOC_DECLSPEC int hwloc_bitmap_taskset_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);
/*
 * 构建位图。
 */

/** \brief 清空位图 \p bitmap */
HWLOC_DECLSPEC void hwloc_bitmap_zero(hwloc_bitmap_t bitmap);

/** \brief 用所有可能的索引填充位图 \p bitmap（即使这些对象不存在或不可用） */
HWLOC_DECLSPEC void hwloc_bitmap_fill(hwloc_bitmap_t bitmap);

/** \brief 清空位图 \p bitmap 并添加位 \p id */
HWLOC_DECLSPEC int hwloc_bitmap_only(hwloc_bitmap_t bitmap, unsigned id);

/** \brief 填充位图 \p bitmap 并清除索引 \p id */
HWLOC_DECLSPEC int hwloc_bitmap_allbut(hwloc_bitmap_t bitmap, unsigned id);

/** \brief 从无符号长整型 \p mask 设置位图 \p bitmap */
HWLOC_DECLSPEC int hwloc_bitmap_from_ulong(hwloc_bitmap_t bitmap, unsigned long mask);

/** \brief 从无符号长整型 \p mask 设置位图 \p bitmap 作为第 \p i 个子集 */
HWLOC_DECLSPEC int hwloc_bitmap_from_ith_ulong(hwloc_bitmap_t bitmap, unsigned i, unsigned long mask);

/** \brief 从无符号长整型数组 \p masks 设置位图 \p bitmap 作为前 \p nr 个子集 */
HWLOC_DECLSPEC int hwloc_bitmap_from_ulongs(hwloc_bitmap_t bitmap, unsigned nr, const unsigned long *masks);


/*
 * 修改位图。
 */

/** \brief 在位图 \p bitmap 中添加索引 \p id */
HWLOC_DECLSPEC int hwloc_bitmap_set(hwloc_bitmap_t bitmap, unsigned id);

/** \brief 在位图 \p bitmap 中添加从 \p begin 到 \p end 的索引。
 *
 * 如果 \p end 是 \c -1，则范围是无限的。
 */
HWLOC_DECLSPEC int hwloc_bitmap_set_range(hwloc_bitmap_t bitmap, unsigned begin, int end);

/** \brief 用无符号长整型 \p mask 替换位图 \p bitmap 的第 \p i 个子集 */
HWLOC_DECLSPEC int hwloc_bitmap_set_ith_ulong(hwloc_bitmap_t bitmap, unsigned i, unsigned long mask);

/** \brief 从位图 \p bitmap 中移除索引 \p id */
HWLOC_DECLSPEC int hwloc_bitmap_clr(hwloc_bitmap_t bitmap, unsigned id);

/** \brief 从位图 \p bitmap 中移除从 \p begin 到 \p end 的索引。
 *
 * 如果 \p end 是 \c -1，则范围是无限的。
 */
# 清除位图中指定范围内的位
HWLOC_DECLSPEC int hwloc_bitmap_clr_range(hwloc_bitmap_t bitmap, unsigned begin, int end);

/** \brief 保留位图 \p bitmap 中设置的单个索引
 *
 * 在绑定之前可能很有用，这样进程就没有机会在原始掩码中的多个处理器之间迁移。
 * 而不是在给定的 CPU 集合中的任何处理器上运行任务，操作系统调度程序将被强制在这些处理器中的一个上运行它。
 * 它避免了迁移开销和处理器之间的缓存行来回移动。
 *
 * \note 此函数不适用于在单个 CPU 集合中分发多个进程。当在相同的输入集上多次调用时，它总是返回相同的单个位。
 * 可以使用 hwloc_distrib() 生成用于在单个多处理器对象下分发多个任务的 CPU 集合。
 *
 * \note 此函数不能直接应用于对象集。它应该应用于副本（可以使用 hwloc_bitmap_dup() 获得）。
 */
HWLOC_DECLSPEC int hwloc_bitmap_singlify(hwloc_bitmap_t bitmap);


/*
 * 咨询位图。
 */

/** \brief 将位图 \p bitmap 的开始部分转换为无符号长整型 \p mask */
HWLOC_DECLSPEC unsigned long hwloc_bitmap_to_ulong(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 将位图 \p bitmap 的第 \p i 个子集转换为无符号长整型掩码 */
HWLOC_DECLSPEC unsigned long hwloc_bitmap_to_ith_ulong(hwloc_const_bitmap_t bitmap, unsigned i) __hwloc_attribute_pure;

/** \brief 将位图 \p bitmap 的前 \p nr 个子集转换为 \p nr 个无符号长整型 \p masks 的数组
 *
 * \p nr 可能会在之前使用 hwloc_bitmap_nr_ulongs() 确定。
 *
 * \return 0
 */
HWLOC_DECLSPEC int hwloc_bitmap_to_ulongs(hwloc_const_bitmap_t bitmap, unsigned nr, unsigned long *masks);
/** \brief 返回存储位图 \p bitmap 所需的无符号长整型数的数量
 *
 * 这是从位图的第一个位（即使未设置）到最后一个设置位的连续无符号长整型数的数量。
 * 这对于知道要传递给 hwloc_bitmap_to_ulongs() 的 \p nr 参数（或者需要调用 hwloc_bitmap_to_ith_ulong() 的调用）
 * 来完全将位图转换为多个无符号长整型数非常有用。
 *
 * 当在 hwloc_topology_get_topology_cpuset() 的输出上调用时，返回的数字足够大，可以容纳拓扑的所有 cpuset。
 *
 * \return 如果 \p bitmap 是无限的，则返回 -1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_nr_ulongs(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 测试索引 \p id 是否属于位图 \p bitmap。
 *
 * \return 如果位图 \p bitmap 中的索引 \p id 处的位被设置，则返回 1，否则返回 0。
 */
HWLOC_DECLSPEC int hwloc_bitmap_isset(hwloc_const_bitmap_t bitmap, unsigned id) __hwloc_attribute_pure;

/** \brief 测试位图 \p bitmap 是否为空
 *
 * \return 如果位图为空，则返回 1，否则返回 0。
 */
HWLOC_DECLSPEC int hwloc_bitmap_iszero(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 测试位图 \p bitmap 是否完全满
 *
 * \return 如果位图是满的，则返回 1，否则返回 0。
 *
 * \note 满位图总是无限设置的。
 */
HWLOC_DECLSPEC int hwloc_bitmap_isfull(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 中的第一个索引（最低有效位）
 *
 * \return 如果 \p bitmap 中没有设置索引，则返回 -1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_first(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 中在索引 \p prev 之后的下一个索引
 *
 * 如果 \p prev 为 -1，则返回第一个索引。
 *
 * \return 如果在 \p bitmap 中没有设置更高索引的索引，则返回 -1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_next(hwloc_const_bitmap_t bitmap, int prev) __hwloc_attribute_pure;
/** \brief 计算位图 \p bitmap 中最后一个索引（最高有效位）
 *
 * \return 如果 \p bitmap 中没有设置索引，或者 \p bitmap 无限设置，则返回-1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_last(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 的“权重”（即位图中设置的索引数量）
 *
 * \return 位图中设置的索引数量。
 *
 * \return 如果 \p bitmap 无限设置，则返回-1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_weight(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 中第一个未设置的索引（最低有效位）
 *
 * \return 如果 \p bitmap 中没有未设置的索引，则返回-1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_first_unset(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 中在索引 \p prev 之后的下一个未设置的索引
 *
 * 如果 \p prev 为-1，则返回第一个未设置的索引。
 *
 * \return 如果 \p bitmap 中没有更高索引的未设置索引，则返回-1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_next_unset(hwloc_const_bitmap_t bitmap, int prev) __hwloc_attribute_pure;

/** \brief 计算位图 \p bitmap 中最后一个未设置的索引（最高有效位）
 *
 * \return 如果 \p bitmap 中没有未设置的索引，或者 \p bitmap 无限设置，则返回-1。
 */
HWLOC_DECLSPEC int hwloc_bitmap_last_unset(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;
/** \brief Loop macro iterating on bitmap \p bitmap
 *
 * The loop must start with hwloc_bitmap_foreach_begin() and end
 * with hwloc_bitmap_foreach_end() followed by a terminating ';'.
 *
 * \p id is the loop variable; it should be an unsigned int.  The
 * first iteration will set \p id to the lowest index in the bitmap.
 * Successive iterations will iterate through, in order, all remaining
 * indexes set in the bitmap.  To be specific: each iteration will return a
 * value for \p id such that hwloc_bitmap_isset(bitmap, id) is true.
 *
 * The assert prevents the loop from being infinite if the bitmap is infinitely set.
 *
 * \hideinitializer
 */
#define hwloc_bitmap_foreach_begin(id, bitmap) \
do { \
        assert(hwloc_bitmap_weight(bitmap) != -1); \  // 断言，防止位图无限设置导致循环无限
        for (id = hwloc_bitmap_first(bitmap); \  // 从位图中获取第一个设置的索引，作为循环变量的初始值
             (unsigned) id != (unsigned) -1; \  // 循环条件，直到遍历完所有设置的索引
             id = hwloc_bitmap_next(bitmap, id)) {  // 获取下一个设置的索引作为下一次循环的值

/** \brief End of loop macro iterating on a bitmap.
 *
 * Needs a terminating ';'.
 *
 * \sa hwloc_bitmap_foreach_begin()
 * \hideinitializer
 */
#define hwloc_bitmap_foreach_end()        \  // 循环结束

/*
 * Combining bitmaps.
 */

/** \brief Or bitmaps \p bitmap1 and \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_or (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);  // 对位图进行或操作，并将结果存储在 res 中

/** \brief And bitmaps \p bitmap1 and \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_and (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);  // 对位图进行与操作，并将结果存储在 res 中

/** \brief And bitmap \p bitmap1 and the negation of \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_andnot (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);  // 对位图1和位图2的取反进行与操作，并将结果存储在 res 中
/** \brief 对位图 \p bitmap1 和 \p bitmap2 进行异或操作，并将结果存储在位图 \p res 中
 *
 * \p res 可以与 \p bitmap1 或 \p bitmap2 相同
 */
HWLOC_DECLSPEC int hwloc_bitmap_xor (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);

/** \brief 对位图 \p bitmap 进行取反操作，并将结果存储在位图 \p res 中
 *
 * \p res 可以与 \p bitmap 相同
 */
HWLOC_DECLSPEC int hwloc_bitmap_not (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap);


/*
 * 比较位图。
 */

/** \brief 测试位图 \p bitmap1 和 \p bitmap2 是否相交。
 *
 * \return 如果位图相交则返回1，否则返回0。
 */
HWLOC_DECLSPEC int hwloc_bitmap_intersects (hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;

/** \brief 测试位图 \p sub_bitmap 是否是位图 \p super_bitmap 的一部分。
 *
 * \return 如果 \p sub_bitmap 包含在 \p super_bitmap 中则返回1，否则返回0。
 *
 * \note 空位图被认为包含在任何其他位图中。
 */
HWLOC_DECLSPEC int hwloc_bitmap_isincluded (hwloc_const_bitmap_t sub_bitmap, hwloc_const_bitmap_t super_bitmap) __hwloc_attribute_pure;

/** \brief 测试位图 \p bitmap1 是否等于位图 \p bitmap2。
 *
 * \return 如果位图相等则返回1，否则返回0。
 */
HWLOC_DECLSPEC int hwloc_bitmap_isequal (hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;
/** \brief Compare bitmaps \p bitmap1 and \p bitmap2 using their lowest index.
 *
 * A bitmap is considered smaller if its least significant bit is smaller.
 * The empty bitmap is considered higher than anything (because its least significant bit does not exist).
 *
 * \return -1 if \p bitmap1 is considered smaller than \p bitmap2.
 * \return 1 if \p bitmap1 is considered larger than \p bitmap2.
 *
 * For instance comparing binary bitmaps 0011 and 0110 returns -1
 * (hence 0011 is considered smaller than 0110)
 * because least significant bit of 0011 (0001) is smaller than least significant bit of 0110 (0010).
 * Comparing 01001 and 00110 would also return -1 for the same reason.
 *
 * \return 0 if bitmaps are considered equal, even if they are not strictly equal.
 * They just need to have the same least significant bit.
 * For instance, comparing binary bitmaps 0010 and 0110 returns 0 because they have the same least significant bit.
 */
HWLOC_DECLSPEC int hwloc_bitmap_compare_first(hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;

/** \brief Compare bitmaps \p bitmap1 and \p bitmap2 in lexicographic order.
 *
 * Lexicographic comparison of bitmaps, starting for their highest indexes.
 * Compare last indexes first, then second, etc.
 * The empty bitmap is considered lower than anything.
 *
 * \return -1 if \p bitmap1 is considered smaller than \p bitmap2.
 * \return 1 if \p bitmap1 is considered larger than \p bitmap2.
 * \return 0 if bitmaps are equal (contrary to hwloc_bitmap_compare_first()).
 *
 * For instance comparing binary bitmaps 0011 and 0110 returns -1
 * (hence 0011 is considered smaller than 0110).
 * Comparing 00101 and 01010 returns -1 too.
 *
 * \note This is different from the non-existing hwloc_bitmap_compare_last()
 * which would only compare the highest index of each bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_compare(hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;
/** @} */
// 结束一个代码块或者文档块

#ifdef __cplusplus
} /* extern "C" */
#endif
// 如果是 C++ 环境，使用 extern "C" 包裹代码

#endif /* HWLOC_BITMAP_H */
// 结束条件编译指令，确保 HWLOC_BITMAP_H 只被包含一次
```