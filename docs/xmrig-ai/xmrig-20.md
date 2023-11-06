# xmrig源码解析 20

# `src/3rdparty/hwloc/include/hwloc/bitmap.h`

这段代码定义了一个名为 "hwloc-bitmap.h" 的头文件。从代码中可以看出，这是一份关于 "hwloc"（硬件位置）的库或者框架的相关代码。然而，对于具体实现和作用，我们需要查看该库或框架的源代码。

一般来说，这段代码可能用于定义 "hwloc" 库中的 "bitmap" 类型。这个类型可能是用于在 "hwloc" 中表示和操作计算机硬件位置（如内存、外设等）的映像。通过使用这个 bitmap，用户可以更方便地在 "hwloc" 中进行各种操作，而不需要显式地关心具体的硬件位置信息。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief The bitmap API, for use in hwloc itself.
 */

#ifndef HWLOC_BITMAP_H
#define HWLOC_BITMAP_H

```

Yes, you are correct. The bitmap data structure can have an infinite size if all bits are set, and it may also be full if all bits are set. When using bitmaps, it is recommended to use the existing object sets and combines them with hwloc_bitmap_or() and hwloc_set_cpubind() functions to access the bits of the bitmap. The hwloc\_cpu\_set\_t, hwloc\_node\_set\_t, and hwloc\_object\_set\_t functions can be used to access the current thread on a pair of cores, which may be a hardware thread. Therefore, when using bitmaps, users should generally avoid building the CPU and node sets manually and instead use the existing object sets.


```cpp
#include "hwloc/autogen/config.h"

#include <assert.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_bitmap The bitmap API
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


```

这段代码定义了一个名为`hwloc_bitmap_t`的指针类型，代表一个内部位图，这个指针类型还有一个名为`hwloc_const_bitmap_t`的常量指针类型。

同时，还定义了一个名为`hwloc_bitmap_free()`的函数，用于释放分配的位图，有一个没有实现函数的`hwloc_bitmap_copy()`函数。

此外，在定义内部函数`hwloc_bitmap_allocate()`中，使用了`const`关键字，表示这个函数可以被const修饰，但是没有实现const函数，所以这个函数只能作为const函数调用。

然后定义了名为`hwloc_bitmap_init()`的函数，用于初始化位图，传入一个空位图，函数内部将这个位图赋值为`NULL`。

接着定义了名为`hwloc_bitmap_copy()`的函数，用于复制一个位图，传入两个`hwloc_bitmap_t`类型的参数，一个表示源位图，一个表示目标位图，函数内部将源位图复制到目标位图中，并返回目标位图。

最后定义了名为`hwloc_bitmap_free()`的函数，用于释放分配的位图，传入一个`hwloc_bitmap_t`类型的参数，函数内部将这个位图所占用的内存释放掉。


```cpp
/** \brief
 * Set of bits represented as an opaque pointer to an internal bitmap.
 */
typedef struct hwloc_bitmap_s * hwloc_bitmap_t;
/** \brief a non-modifiable ::hwloc_bitmap_t */
typedef const struct hwloc_bitmap_s * hwloc_const_bitmap_t;


/*
 * Bitmap allocation, freeing and copying.
 */

/** \brief Allocate a new empty bitmap.
 *
 * \returns A valid bitmap or \c NULL.
 *
 * The bitmap should be freed by a corresponding call to
 * hwloc_bitmap_free().
 */
```

这段代码定义了一个名为 hwloc_bitmap_t 的结构体，其中包含了一个名为 hwloc_bitmap_alloc 的函数和一个名为 hwloc_bitmap_free 的函数。

函数 hwloc_bitmap_alloc 是一个 allocator 函数，用于在内存中分配一个新的 bitmap 并返回其内存地址。这个函数接受一个 void 类型的参数，因此在函数内部，可以对 void 类型的变量进行操作，而不会产生任何错误。

函数 hwloc_bitmap_free 是一个 freeler 函数，用于释放之前分配的 bitmap 内存，如果分配的 bitmap 是一个 NULL 指针，则该函数不会执行任何操作，而是直接返回。

函数 hwloc_bitmap_alloc_full 和 hwloc_bitmap_free 可能是其他库函数，在没有上下文的情况下无法确定它们的确切作用。


```cpp
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_alloc(void) __hwloc_attribute_malloc;

/** \brief Allocate a new full bitmap. */
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_alloc_full(void) __hwloc_attribute_malloc;

/** \brief Free bitmap \p bitmap.
 *
 * If \p bitmap is \c NULL, no operation is performed.
 */
HWLOC_DECLSPEC void hwloc_bitmap_free(hwloc_bitmap_t bitmap);

/** \brief Duplicate bitmap \p bitmap by allocating a new bitmap and copying \p bitmap contents.
 *
 * If \p bitmap is \c NULL, \c NULL is returned.
 */
```

这段代码定义了一个名为hwloc_bitmap_t的类型，其中包含一个名为hwloc_const_bitmap_t的公有指针成员hwloc_const_bitmap_t，以及一个名为hwloc_bitmap_dup的私有成员函数，该函数接受一个名为hwloc_const_bitmap_t的参数src，并将其内容复制到名为dst的已经分配的bitmap中。

另外，还定义了一个名为hwloc_bitmap_copy的函数，该函数也接受名为hwloc_const_bitmap_t的src和dst参数，但它的作用是释放src和dst对bitmap的引用，并返回bitmap dst成功复制src的内容后所占用的内存空间大小。


```cpp
HWLOC_DECLSPEC hwloc_bitmap_t hwloc_bitmap_dup(hwloc_const_bitmap_t bitmap) __hwloc_attribute_malloc;

/** \brief Copy the contents of bitmap \p src into the already allocated bitmap \p dst */
HWLOC_DECLSPEC int hwloc_bitmap_copy(hwloc_bitmap_t dst, hwloc_const_bitmap_t src);


/*
 * Bitmap/String Conversion
 */

/** \brief Stringify a bitmap.
 *
 * Up to \p buflen characters may be written in buffer \p buf.
 *
 * If \p buflen is 0, \p buf may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这三段代码定义了一个名为"hwloc_bitmap_snprintf"的函数，名为：`int hwloc_bitmap_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap)`；一个名为"hwloc_bitmap_asprintf"的函数，名为：`int hwloc_bitmap_asprintf(char ** strp, hwloc_const_bitmap_t bitmap)`；一个名为"hwloc_bitmap_sscanf"的函数，名为：`int hwloc_bitmap_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string)`。

`hwloc_bitmap_snprintf`函数的作用是将一个`hwloc_const_bitmap_t`类型的位图中的数据字符串化并存储到一个`char *`类型的缓冲区中。它接受两个参数：要存储在缓冲区中的字符数组`buf`以及一个表示缓冲区最大字符数组大小的整数`buflen`。如果`buf`参数长度小于`buflen`，该函数将导致错误并返回-1。

`hwloc_bitmap_asprintf`函数的作用是将一个`hwloc_const_bitmap_t`类型的位图字符串化并存储到一个指向字符数组`strp`的指针中。它接受一个参数：要将位图字符串存储到`strp`指针中的字符数组，以及一个表示位图最大字符数组大小的整数`buflen`。

`hwloc_bitmap_sscanf`函数的作用是将一个`hwloc_bitmap_t`类型的位图字符串解析为一个整数数组，然后将该数组存储在给定的`const char *`类型的变量`string`中。它接受一个参数：要存储在`string`中的字符数组，以及一个表示位图最大字符数组大小的整数`buflen`。如果`buflen`小于要存储的字符数，该函数将导致错误并返回-1。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

/** \brief Stringify a bitmap into a newly allocated string.
 *
 * \return -1 on error.
 */
HWLOC_DECLSPEC int hwloc_bitmap_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

/** \brief Parse a bitmap string and stores it in bitmap \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);

/** \brief Stringify a bitmap in the list format.
 *
 * Lists are comma-separated indexes or ranges.
 * Ranges are dash separated indexes.
 * The last range may not have an ending indexes if the bitmap is infinitely set.
 *
 * Up to \p buflen characters may be written in buffer \p buf.
 *
 * If \p buflen is 0, \p buf may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这段代码定义了三个函数，用于将不同格式的 bitmap 转化为字符串和从字符串中解析出 bitmap。

第一个函数 `hwloc_bitmap_list_snprintf` 将给定的 bitmap 中的所有元素拼接成一个字符串，使用给定的 `buf` 和 `buflen` 参数。这个函数的实现类似于 `snprintf` 函数，用于从 `buf` 中将元素一个一个地输出，然后将剩余的元素放入 `buflen` 参数指定的字符串中。

第二个函数 `hwloc_bitmap_list_asprintf` 将从给定的字符串中解析出一个 bitmap，使用 `buflen` 参数指定从字符串中读取的最大字符数。这个函数的实现类似于 `asprintf` 函数，用于从字符串中解析出一个 bitmap，并尝试将解析得到的 bitmap 赋值给给定的 `bitmap` 参数。

第三个函数 `hwloc_bitmap_list_sscanf` 将给定的 bitmap 中的所有元素解析成一个字符串，并尝试从给定的字符串中解析出每个元素。这个函数的实现类似于 `sscanf` 函数，用于将给定的 bitmap 中的所有元素解析成一个字符串，并尝试将解析得到的字符串存储在给定的 `strp` 参数指向的指针中。

这三个函数都在 `hwloc_bitmap_list_` 命名空间中定义，说明它们属于该命名空间。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_list_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

/** \brief Stringify a bitmap into a newly allocated list string.
 *
 * \return -1 on error.
 */
HWLOC_DECLSPEC int hwloc_bitmap_list_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

/** \brief Parse a list string and stores it in bitmap \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_list_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);

/** \brief Stringify a bitmap in the taskset-specific format.
 *
 * The taskset command manipulates bitmap strings that contain a single
 * (possible very long) hexadecimal number starting with 0x.
 *
 * Up to \p buflen characters may be written in buffer \p buf.
 *
 * If \p buflen is 0, \p buf may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这段代码定义了三个函数，用于将HWLOC结构体中的bitmap参数输出为字符串，从字符串中解析出bitmap参数，并从字符串中解析出HWLOC结构体。

第一个函数hwloc_bitmap_taskset_snprintf接收一个整型指针buf，表示要存储的字符串长度，以及一个整型参数bitmap，表示要操作的bitmap。该函数将bitmap填充到buf指向的位置，并使用snprintf函数将字节数组buf填充为字符串，如果填充失败或者buf为空，则返回-1。

第二个函数hwloc_bitmap_taskset_asprintf接收一个指向字符型字符p的指针，以及一个整型参数bitmap，表示要解析的字符串长度。该函数使用asprintf函数将字符串缓冲区缓冲为bitmap，如果解析失败或者bitmap为空，则返回-1。

第三个函数hwloc_bitmap_taskset_sscanf接收一个整型参数bitmap，以及一个指向字符型字符p的指针，表示要解析的字符串。该函数使用sscanf函数从字符串中解析出bitmap，并将解析结果存储到bitmap指向的位置。如果解析失败或者bitmap为空，则返回-1。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_taskset_snprintf(char * __hwloc_restrict buf, size_t buflen, hwloc_const_bitmap_t bitmap);

/** \brief Stringify a bitmap into a newly allocated taskset-specific string.
 *
 * \return -1 on error.
 */
HWLOC_DECLSPEC int hwloc_bitmap_taskset_asprintf(char ** strp, hwloc_const_bitmap_t bitmap);

/** \brief Parse a taskset-specific bitmap string and stores it in bitmap \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_taskset_sscanf(hwloc_bitmap_t bitmap, const char * __hwloc_restrict string);


/*
 * Building bitmaps.
 */

```

以下是这些函数的注释：

1. `hwloc_bitmap_zero`：此函数用于将给定的位图清零。它接收一个 `hwloc_bitmap_t` 类型的位图，不输出任何信息，但请注意，在使用此函数之前，确保已使用 `hwloc_bitmap_ Allocator` 函数分配了足够的内存空间以存储位图。

2. `hwloc_bitmap_fill`：此函数用于将给定的位图填充满所有可能的索引（即使那些索引不存在的或者不可用）。它接收一个 `hwloc_bitmap_t` 类型的位图，不输出任何信息，但请注意，在使用此函数之前，确保已使用 `hwloc_bitmap_ Allocator` 函数分配了足够的内存空间以存储位图。

3. `hwloc_bitmap_only`：此函数用于将给定的位图填充为指定索引处的非零值，但不填充其他索引处的值。它接收一个 `hwloc_bitmap_t` 类型的位图和一个 `unsigned int` 类型的索引，不输出任何信息，但请注意，在使用此函数之前，确保已使用 `hwloc_bitmap_ Allocator` 函数分配了足够的内存空间以存储位图。

4. `hwloc_bitmap_allbut`：此函数用于将给定的位图填充为指定索引处的非零值，但不填充其他索引处的值。它接收一个 `hwloc_bitmap_t` 类型的位图和一个 `unsigned int` 类型的索引，不输出任何信息，但请注意，在使用此函数之前，确保已使用 `hwloc_bitmap_ Allocator` 函数分配了足够的内存空间以存储位图。

5. `hwloc_bitmap_from_ulong`：此函数用于从指定 `unsigned long` 类型的掩码创建位图，并将其分配给指定的位图。它接收一个 `hwloc_bitmap_t` 类型的位图和一个 `unsigned long` 类型的掩码，不输出任何信息，但请注意，在使用此函数之前，确保已使用 `hwloc_bitmap_ Allocator` 函数分配了足够的内存空间以存储位图。


```cpp
/** \brief Empty the bitmap \p bitmap */
HWLOC_DECLSPEC void hwloc_bitmap_zero(hwloc_bitmap_t bitmap);

/** \brief Fill bitmap \p bitmap with all possible indexes (even if those objects don't exist or are otherwise unavailable) */
HWLOC_DECLSPEC void hwloc_bitmap_fill(hwloc_bitmap_t bitmap);

/** \brief Empty the bitmap \p bitmap and add bit \p id */
HWLOC_DECLSPEC int hwloc_bitmap_only(hwloc_bitmap_t bitmap, unsigned id);

/** \brief Fill the bitmap \p and clear the index \p id */
HWLOC_DECLSPEC int hwloc_bitmap_allbut(hwloc_bitmap_t bitmap, unsigned id);

/** \brief Setup bitmap \p bitmap from unsigned long \p mask */
HWLOC_DECLSPEC int hwloc_bitmap_from_ulong(hwloc_bitmap_t bitmap, unsigned long mask);

```

这段代码定义了两个名为 `hwloc_bitmap_from_ith_ulong` 和 `hwloc_bitmap_from_ulongs` 的函数，它们的功能是分别从 `unsigned long` 和 `const unsigned long *` 类型的输入中构建位图，并支持对位图进行修改。

具体来说，这两个函数接受一个 `hwloc_bitmap_t` 类型的输入参数 `bitmap`，一个 `unsigned i` 和一个 `unsigned long` 类型的输入参数 `mask`，和一个 `unsigned nr` 类型的输入参数 `masks`。

第一个函数 `hwloc_bitmap_from_ith_ulong` 使用位图的 `i` 索引作为子集，并从 `mask` 中选择适当的子集，然后将结果赋值给 `bitmap`。第二个函数 `hwloc_bitmap_from_ulongs` 则使用 `nr` 作为输入参数，并将每个 `masks` 作为位图的子集。


```cpp
/** \brief Setup bitmap \p bitmap from unsigned long \p mask used as \p i -th subset */
HWLOC_DECLSPEC int hwloc_bitmap_from_ith_ulong(hwloc_bitmap_t bitmap, unsigned i, unsigned long mask);

/** \brief Setup bitmap \p bitmap from unsigned longs \p masks used as first \p nr subsets */
HWLOC_DECLSPEC int hwloc_bitmap_from_ulongs(hwloc_bitmap_t bitmap, unsigned nr, const unsigned long *masks);


/*
 * Modifying bitmaps.
 */

/** \brief Add index \p id in bitmap \p bitmap */
HWLOC_DECLSPEC int hwloc_bitmap_set(hwloc_bitmap_t bitmap, unsigned id);

/** \brief Add indexes from \p begin to \p end in bitmap \p bitmap.
 *
 * If \p end is \c -1, the range is infinite.
 */
```

这段代码定义了三个名为 `hwloc_bitmap_set_range`、`hwloc_bitmap_set_ith_ulong` 和 `hwloc_bitmap_clr` 的函数，以及一个名为 `hwloc_bitmap_clr_range` 的函数。它们都是用于在给定的 `hwloc_bitmap_t` 类型的图像中设置或清除特定索引处的像素值。

具体来说，这些函数可以用于在给定的位置上设置或清除图像中的像素值。通过使用 `unsigned long` 类型的参数，可以确保在图像中设置或清除像素值时使用的是无符号长整数，从而能够正确处理负数和浮点数。

此外，这些函数还提供了一个名为 `hwloc_bitmap_clr_range` 的函数，它可以清除指定范围内的所有像素值。这个函数有一个 `unsigned int` 类型的参数 `end`，如果 `end` 的值是 `-1`，表示该范围是无限的，这可能会在某些情况下有用。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_set_range(hwloc_bitmap_t bitmap, unsigned begin, int end);

/** \brief Replace \p i -th subset of bitmap \p bitmap with unsigned long \p mask */
HWLOC_DECLSPEC int hwloc_bitmap_set_ith_ulong(hwloc_bitmap_t bitmap, unsigned i, unsigned long mask);

/** \brief Remove index \p id from bitmap \p bitmap */
HWLOC_DECLSPEC int hwloc_bitmap_clr(hwloc_bitmap_t bitmap, unsigned id);

/** \brief Remove indexes from \p begin to \p end in bitmap \p bitmap.
 *
 * If \p end is \c -1, the range is infinite.
 */
HWLOC_DECLSPEC int hwloc_bitmap_clr_range(hwloc_bitmap_t bitmap, unsigned begin, int end);

/** \brief Keep a single index among those set in bitmap \p bitmap
 *
 * May be useful before binding so that the process does not
 * have a chance of migrating between multiple processors
 * in the original mask.
 * Instead of running the task on any PU inside the given CPU set,
 * the operating system scheduler will be forced to run it on a single
 * of these PUs.
 * It avoids a migration overhead and cache-line ping-pongs between PUs.
 *
 * \note This function is NOT meant to distribute multiple processes
 * within a single CPU set. It always return the same single bit when
 * called multiple times on the same input set. hwloc_distrib() may
 * be used for generating CPU sets to distribute multiple tasks below
 * a single multi-PU object.
 *
 * \note This function cannot be applied to an object set directly. It
 * should be applied to a copy (which may be obtained with hwloc_bitmap_dup()).
 */
```

这段代码是一个名为 `hwloc_bitmap_singlify` 的函数，它的作用是将给定的 `hwloc_bitmap_t` 类型的位图转换为 `unsigned long` 类型的位图，并返回生成的 `unsigned long` 类型的位图。

通过观察代码可以发现，这个函数内部进行了两次转换，一次是将给定的位图转换为 `unsigned long` 类型的位图，第二次是将给定的 `unsigned long` 类型的位图转换为 `unsigned long` 类型的位图。

此外，函数还使用了两个辅助函数 `hwloc_bitmap_to_ulong` 和 `hwloc_bitmap_to_ith_ulong`，它们的作用是将给定的位图转换为 `unsigned long` 类型的位图，并传入相应的索引 `i` 和 `nr`。这两个函数同样使用了 `__hwloc_attribute_pure` 修饰，表示函数内部的数据不可被修改。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_singlify(hwloc_bitmap_t bitmap);


/*
 * Consulting bitmaps.
 */

/** \brief Convert the beginning part of bitmap \p bitmap into unsigned long \p mask */
HWLOC_DECLSPEC unsigned long hwloc_bitmap_to_ulong(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Convert the \p i -th subset of bitmap \p bitmap into unsigned long mask */
HWLOC_DECLSPEC unsigned long hwloc_bitmap_to_ith_ulong(hwloc_const_bitmap_t bitmap, unsigned i) __hwloc_attribute_pure;

/** \brief Convert the first \p nr subsets of bitmap \p bitmap into the array of \p nr unsigned long \p masks
 *
 * \p nr may be determined earlier with hwloc_bitmap_nr_ulongs().
 *
 * \return 0
 */
```

这段代码定义了一个名为 `hwloc_bitmap_to_ulongs` 的函数，它的参数是一个 `hwloc_const_bitmap_t` 类型的 bitmap，以及一个 `unsigned long *` 类型的整数 `masks`，表示需要存储的 bitmap 的掩码。

这个函数的作用是将给定的 bitmap 转换成多个 unsigned longs，并返回所需的位数数量。这个函数可以用于将 bitmap 转换为不同的布局，如 `hwloc_bitmap_to_ulong` 和 `hwloc_bitmap_to_ith` 函数。

具体来说，这个函数通过统计 bitmap 中所有可设置的 bit 的数量，来确定需要存储的 unsigned longs 的数量。如果 bitmap 无法完全设置，函数将返回 -1，此时需要根据实际需求来选择需要的 unsigned longs 数量。

需要注意的是，这个函数在 `hwloc_topology_get_topology_cpuset` 的输出中使用，返回的结果需要足够大，才能保证所有可设置的 bit 都能够正确地转换成 unsigned longs。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_to_ulongs(hwloc_const_bitmap_t bitmap, unsigned nr, unsigned long *masks);

/** \brief Return the number of unsigned longs required for storing bitmap \p bitmap entirely
 *
 * This is the number of contiguous unsigned longs from the very first bit of the bitmap
 * (even if unset) up to the last set bit.
 * This is useful for knowing the \p nr parameter to pass to hwloc_bitmap_to_ulongs()
 * (or which calls to hwloc_bitmap_to_ith_ulong() are needed)
 * to entirely convert a bitmap into multiple unsigned longs.
 *
 * When called on the output of hwloc_topology_get_topology_cpuset(),
 * the returned number is large enough for all cpusets of the topology.
 *
 * \return -1 if \p bitmap is infinite.
 */
```

这段代码定义了三个名为hwloc_bitmap_nr_ulongs、hwloc_bitmap_isset和hwloc_bitmap_iszero的函数，以及一个名为hwloc_bitmap_is_completely_full的判断函数。这些函数和判断函数都使用了hwloc_const_bitmap_t类型的变量bitmap，并对其进行了修改，以提供对bitmap的测试和判断。

具体来说，hwloc_bitmap_nr_ulongs函数接收一个bitmap和一个整数参数id，并返回该bitmap中包含多少个id为该整数的元素。hwloc_bitmap_isset函数则接收一个bitmap和一个整数参数id，并返回该bitmap中是否包含该id的元素。hwloc_bitmap_iszero函数则接收一个bitmap，并返回其中所有元素都为0。hwloc_bitmap_is_completely_full函数则接收一个bitmap，并返回该bitmap是否已经被完全占用。

由于这些函数和判断函数都使用了pure修饰，因此它们的功能不会被任何元猜测优化器优化，也不会对函数调用者的性能产生任何影响。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_nr_ulongs(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Test whether index \p id is part of bitmap \p bitmap.
 *
 * \return 1 if the bit at index \p id is set in bitmap \p bitmap, 0 otherwise.
 */
HWLOC_DECLSPEC int hwloc_bitmap_isset(hwloc_const_bitmap_t bitmap, unsigned id) __hwloc_attribute_pure;

/** \brief Test whether bitmap \p bitmap is empty
 *
 * \return 1 if bitmap is empty, 0 otherwise.
 */
HWLOC_DECLSPEC int hwloc_bitmap_iszero(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Test whether bitmap \p bitmap is completely full
 *
 * \return 1 if bitmap is full, 0 otherwise.
 *
 * \note A full bitmap is always infinitely set.
 */
```

这两段代码定义了两个名为 "hwloc_bitmap_isfull" 和 "hwloc_bitmap_first" 的函数，以及它们的输入参数和返回值类型。

"hwloc_bitmap_isfull" 函数接收一个名为 "bitmap" 的常引用 bitmap，并计算出 bitmap 中第一个非空元素的索引(即第一个有效位)。如果没有有效的索引，函数将返回 -1。函数的实现为：

```cppc
int hwloc_bitmap_isfull(hwloc_const_bitmap_t bitmap) {
   int index = -1;
   if (bitmap) {
       for (int i = 0; i < bitmap->size; i++) {
           if (bitmap->data[i] != 0) {
               index = i;
               break;
           }
       }
   }
   return index;
}
```

"hwloc_bitmap_first" 函数与 "hwloc_bitmap_isfull" 类似，但它的输入参数中有一个名为 "prev" 的参数，表示要返回的结果 index 所处的比特位在 bitmap 中是第几个。如果 "prev" 的值为 -1，函数将直接返回第一个非空元素的索引。函数的实现为：

```cppc
int hwloc_bitmap_first(hwloc_const_bitmap_t bitmap, int prev) {
   int index = -1;
   if (bitmap) {
       for (int i = 0; i < bitmap->size; i++) {
           if (bitmap->data[i] != 0) {
               index = i;
               break;
           }
       }
       if (prev == -1) {
           index = 0;
       }
   }
   return index;
}
```

这两段代码定义的函数可以用于管理一个 bitmap 中的索引，使得函数可以方便地在不同的上下文中使用。例如，可以在使用 bitmap 的时候使用这些函数来查找特定的元素，或者在需要返回一个 bitmap 中某个元素的时候使用这些函数。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_isfull(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Compute the first index (least significant bit) in bitmap \p bitmap
 *
 * \return -1 if no index is set in \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_first(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Compute the next index in bitmap \p bitmap which is after index \p prev
 *
 * If \p prev is -1, the first index is returned.
 *
 * \return -1 if no index with higher index is set in \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_next(hwloc_const_bitmap_t bitmap, int prev) __hwloc_attribute_pure;

```

这两段代码定义了两个名为"hwloc_bitmap_last"和"hwloc_bitmap_weight"的函数，用于计算位图(bitmap)中Last索引(最显著的比特位)和该位图(bitmap)中所有有效索引的数量。

具体来说，"hwloc_bitmap_last"函数接收一个hwloc_const_bitmap_t类型的位图(bitmap)参数，并返回一个int类型的值，表示位图中最显著的比特位的索引位置。如果位图中没有设置任何索引，函数将返回-1。如果位图无限设置，函数也将返回-1。

而"hwloc_bitmap_weight"函数与"hwloc_bitmap_last"类似，但会计算位图中所有有效索引的数量。这个函数同样接收一个hwloc_const_bitmap_t类型的位图(bitmap)参数，并返回一个int类型的值。如果位图中设置的索引数量是无穷大的，函数将返回这个无穷大的值。否则，函数将返回位图中的有效索引数量。


```cpp
/** \brief Compute the last index (most significant bit) in bitmap \p bitmap
 *
 * \return -1 if no index is set in \p bitmap, or if \p bitmap is infinitely set.
 */
HWLOC_DECLSPEC int hwloc_bitmap_last(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Compute the "weight" of bitmap \p bitmap (i.e., number of
 * indexes that are in the bitmap).
 *
 * \return the number of indexes that are in the bitmap.
 *
 * \return -1 if \p bitmap is infinitely set.
 */
HWLOC_DECLSPEC int hwloc_bitmap_weight(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

```

这段代码定义了两个函数，用于计算位图（bitmap）中第一个未被设置的比特（bit）的索引（index）或下一个未被设置的比特的索引。这两个函数是作为 hwloc_const_bitmap_t 类型的函数，并且使用了 hwloc_attribute_pure 修饰，这意味着这两个函数的实现不会对原始的 bitmap 产生任何副作用，也不会返回任何有用的信息。

第一个函数名为 hwloc_bitmap_first_unset，它的作用是计算位图第一个未被设置的比特的索引。如果位图中的第一个未被设置的比特的索引不存在，那么函数将返回 -1。函数的实现非常简单，直接使用 bitmap 的第一个未被设置的比特的索引，并将其返回。

第二个函数名为 hwloc_bitmap_next_unset，它的作用是计算位图下一个未被设置的比特的索引。如果位图中的第一个未被设置的比特的索引为 -1，那么函数将返回该索引。函数的实现也相对简单，直接使用 bitmap 的最后一个未被设置的比特的索引，并将其返回。


```cpp
/** \brief Compute the first unset index (least significant bit) in bitmap \p bitmap
 *
 * \return -1 if no index is unset in \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_first_unset(hwloc_const_bitmap_t bitmap) __hwloc_attribute_pure;

/** \brief Compute the next unset index in bitmap \p bitmap which is after index \p prev
 *
 * If \p prev is -1, the first unset index is returned.
 *
 * \return -1 if no index with higher index is unset in \p bitmap.
 */
HWLOC_DECLSPEC int hwloc_bitmap_next_unset(hwloc_const_bitmap_t bitmap, int prev) __hwloc_attribute_pure;

/** \brief Compute the last unset index (most significant bit) in bitmap \p bitmap
 *
 * \return -1 if no index is unset in \p bitmap, or if \p bitmap is infinitely set.
 */
```

这段代码定义了一个名为 `hwloc_bitmap_last_unset` 的函数，属于 `hwloc_const_bitmap_t` 类型，声明为 `__hwloc_attribute_pure`。

这个函数接受一个 `hwloc_const_bitmap_t` 的参数 `bitmap`，并使用一个循环来遍历该参数。循环的起始点为 `hwloc_bitmap_foreach_begin()`，结束点为 `hwloc_bitmap_foreach_end()`，并在退出循环后添加一个 `;`。

在循环内部，定义了一个整型变量 `id`，并使用 `hwloc_bitmap_isset` 函数检查当前的 `bitmap` 是否包含 `id`。如果是，返回 `true`，否则返回 `false`。为了防止无限循环，函数使用了 `__assert` 函数。

该函数的作用是，遍历 `bitmap` 中的所有元素，对于每个元素，检查它是否被设置为 `true`，如果是，则返回 `true`，否则返回 `false`。


```cpp
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
```

这段代码定义了一个名为`hwloc_bitmap_foreach_begin`的宏，以及一个名为`hwloc_bitmap_foreach_end`的宏。它们的作用是生成一个循环，该循环会在定义的`id`和`bitmap`中迭代，直到`hwloc_bitmap_weight(bitmap)`不等于-1时停止。

更具体地说，`hwloc_bitmap_foreach_begin`定义了一个`do-while`循环，该循环会首先检查`bitmap`是否具有重量值。如果是，它会尝试使用`hwloc_bitmap_first(bitmap)`和`hwloc_bitmap_next(bitmap, id)`获取第一个元素和后续的元素。然后，它会迭代`id`，直到找到第一个元素的下一个元素，这意味着整个循环将迭代`bitmap`中的所有元素。

`hwloc_bitmap_foreach_end`定义了一个`break`宏，用于在`hwloc_bitmap_weight(bitmap)`不等于-1时跳出`do-while`循环。如果没有这样的循环，该宏将一直运行，不断迭代`id`和`bitmap`中的所有元素。


```cpp
#define hwloc_bitmap_foreach_begin(id, bitmap) \
do { \
        assert(hwloc_bitmap_weight(bitmap) != -1); \
        for (id = hwloc_bitmap_first(bitmap); \
             (unsigned) id != (unsigned) -1; \
             id = hwloc_bitmap_next(bitmap, id)) {

/** \brief End of loop macro iterating on a bitmap.
 *
 * Needs a terminating ';'.
 *
 * \sa hwloc_bitmap_foreach_begin()
 * \hideinitializer
 */
#define hwloc_bitmap_foreach_end()		\
        } \
} while (0)


```

这段代码定义了两个名为`hwloc_bitmap_or`和`hwloc_bitmap_and`的函数，它们用于对两个位图进行异或或与操作。函数接受两个位图`bitmap1`和`bitmap2`，并将它们的与结果存储在参数`res`中。如果两个位图相同，则可以将结果直接存储在`res`中。函数内部定义了位图和二进制字段，以使函数在定义时可以与位图和二进制字段相互替换。


```cpp
/*
 * Combining bitmaps.
 */

/** \brief Or bitmaps \p bitmap1 and \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_or (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);

/** \brief And bitmaps \p bitmap1 and \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_and (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);

```

这段代码定义了三个函数，名为`hwloc_bitmap_andnot`，`hwloc_bitmap_xor`和`hwloc_bitmap_not`，它们都接受两个`hwloc_bitmap_t`类型的参数：`res`，`bitmap1`和`bitmap2`。

`hwloc_bitmap_andnot`函数的作用是执行 bitmap 1 和 bitmap 2的按位与操作，并将结果存储到参数 `res` 中。它需要两个输入 bitmap，然后输出一个和这两个 bitmap 一样的 bitmap。

`hwloc_bitmap_xor`函数的作用是执行 bitmap 1 和 bitmap 2的按位或操作，并将结果存储到参数 `res` 中。它需要两个输入 bitmap，然后输出一个新的 bitmap，其中包含这两个 bitmap 的按位或结果。

`hwloc_bitmap_not`函数的作用是执行 bitmap 1的按位取反操作，并将结果存储到参数 `res` 中。它需要一个输入 bitmap，然后输出一个新的 bitmap，其中包含该 bitmap 的按位取反结果。


```cpp
/** \brief And bitmap \p bitmap1 and the negation of \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_andnot (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);

/** \brief Xor bitmaps \p bitmap1 and \p bitmap2 and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap1 or \p bitmap2
 */
HWLOC_DECLSPEC int hwloc_bitmap_xor (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2);

/** \brief Negate bitmap \p bitmap and store the result in bitmap \p res
 *
 * \p res can be the same as \p bitmap
 */
```

这段代码定义了一个名为 `hwloc_bitmap_not` 的函数，其参数为两个 `hwloc_bitmap_t` 类型的变量 `res` 和 `bitmap`，函数返回一个整数类型。函数的作用是判断两个 `bitmap` 是否 intersect（重叠），即如果两个 `bitmap` 中有公共的元素，则返回 1，否则返回 0。

此外，还定义了一个名为 `hwloc_bitmap_intersects` 的函数，其参数为两个 `hwloc_const_bitmap_t` 类型的变量 `bitmap1` 和 `bitmap2`，函数返回一个整数类型。函数的作用也是判断两个 `bitmap` 是否 intersect，与前者类似，但函数使用了一个名为 `__hwloc_attribute_pure` 的修饰，这个修饰表示该函数的实现不需要考虑指针的正确性，可以对变量 `bitmap1` 和 `bitmap2` 进行修改而不影响函数的输出结果。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_not (hwloc_bitmap_t res, hwloc_const_bitmap_t bitmap);


/*
 * Comparing bitmaps.
 */

/** \brief Test whether bitmaps \p bitmap1 and \p bitmap2 intersects.
 *
 * \return 1 if bitmaps intersect, 0 otherwise.
 */
HWLOC_DECLSPEC int hwloc_bitmap_intersects (hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;

/** \brief Test whether bitmap \p sub_bitmap is part of bitmap \p super_bitmap.
 *
 * \return 1 if \p sub_bitmap is included in \p super_bitmap, 0 otherwise.
 *
 * \note The empty bitmap is considered included in any other bitmap.
 */
```

这两段代码定义了一个名为 `hwloc_bitmap_isincluded` 的函数，该函数接收两个 `hwloc_const_bitmap_t` 类型的参数 `sub_bitmap` 和 `super_bitmap`，并返回一个整数。函数实现了一个比较两个 `hwloc_const_bitmap_t` 是否相等的功能。如果两个 `hwloc_const_bitmap_t` 相等，函数返回 0，否则返回 1。如果两个 `hwloc_const_bitmap_t` 不相等，但它们的元素值相同（注意，元素值相同但排列顺序可能不同，函数仍然会返回 1），函数仍然会返回 0。

另外，这两段代码还定义了一个名为 `hwloc_bitmap_isequal` 的函数，该函数与 `hwloc_bitmap_isincluded` 函数正好相反，该函数接收两个 `hwloc_const_bitmap_t` 类型的参数 `bitmap1` 和 `bitmap2`，并返回一个整数。函数实现了一个比较两个 `hwloc_const_bitmap_t` 是否相等的功能。如果两个 `hwloc_const_bitmap_t` 相等，函数返回 0，否则返回 1。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_isincluded (hwloc_const_bitmap_t sub_bitmap, hwloc_const_bitmap_t super_bitmap) __hwloc_attribute_pure;

/** \brief Test whether bitmap \p bitmap1 is equal to bitmap \p bitmap2.
 *
 * \return 1 if bitmaps are equal, 0 otherwise.
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
```

这段代码定义了一个名为 `hwloc_bitmap_compare_first` 的函数，它比较两个给定的位图 `bitmap1` 和 `bitmap2`，返回比较结果。

这个函数使用了 `__hwloc_attribute_pure` 修饰，表明它是一个纯函数，即不产生任何副作用（返回值可以被忽略）。

函数的实现包括以下几个步骤：

1. 比较两个位图的第一个元素，如果两个位图的第一个元素相同，则返回 0。
2. 如果两个位图的第一个元素不同，将 `bitmap1` 的第一个元素设为高水位，`bitmap2` 的第一个元素设为低水位，然后对两个位图的第一个元素进行比较，根据比较结果更新两个位图的第一个元素。
3. 对于 `bitmap1` 和 `bitmap2`，重复步骤 1 和 2，直到两个位图的最后一个元素相同或者其中一个位图的最后一个元素是 `NULL`。

该函数的作用是提供一个在 `lexicographic` 顺序下比较两个位图的方法，这个顺序按照位图的最高索引排序，也就是说，首先比较两个位图的最高索引，如果相同就比较次高的索引，以此类推。如果两个位图的所有元素都相同，则返回 `0`，否则返回 `-1`。


```cpp
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
```

这段代码定义了一个名为 `hwloc_bitmap_compare` 的函数，它的参数有两个，分别为 `hwloc_const_bitmap_t` 类型的 bitmap1 和 bitmap2。这个函数使用了 `__hwloc_attribute_pure` 修饰，表明它是在一个被称为 `hwloc` 的库中定义的，同时也使用了 `extern "C"` 声明，这意味着这个函数是 C 语言编写的。

这个函数的作用是比较两个 bitmap，返回比较结果，官方文档中没有提供具体的实现，但根据它的名字和文档中没有其他函数和头文件来继承，我们可以推测它可能是一个通用的比较函数，用于比较两个 bitmap 的差异。


```cpp
HWLOC_DECLSPEC int hwloc_bitmap_compare(hwloc_const_bitmap_t bitmap1, hwloc_const_bitmap_t bitmap2) __hwloc_attribute_pure;

/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_BITMAP_H */

```

# `src/3rdparty/hwloc/include/hwloc/cpukinds.h`

这段代码定义了一个名为"HWLOC_CPUKINDS_H"的文件，其中定义了CPU的不同类型。它包含了以下几部分：

1. 引入头文件：包含"hwloc.h"。

2. 使用 extern 关键字定义一个名为"__attribute__((constructor))"的函数，它的参数列表为空，这个函数会在程序运行时自己定义。

3. 定义了 CPU 类型枚举类型 "CPU_KINDS"，列举了以下值：

  - 0: "CPU_KINDS_DEFAULT"
  - 1: "CPU_KINDS_CYCLES"
  - 2: "CPU_KINDS_SSE"
  - 3: "CPU_KINDS_SSE2"
  - 4: "CPU_KINDS_AVX"
  - 5: "CPU_KINDS_AVX2"
  - 6: "CPU_KINDS_KNI"
  - 7: "CPU_KINDS_KNI8"
  - 8: "CPU_KINDS_KNI9"
  - 9: "CPU_KINDS_KNI16"
  - 10: "CPU_KINDS_KNI32"
  - 11: "CPU_KINDS_KNI64"

4. 使用 enum 关键字定义了一个名为 "CPU_KINDS" 的枚举类型，其中包含了上面定义的 11 个枚举值。

5. 使用 include 语句引入了 "hwloc/__hwloc_exports.h" 头文件，可能是用于在编译时检查 HWLOC 是否定义。


```cpp
/*
 * Copyright © 2020-2021 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Kinds of CPU cores.
 */

#ifndef HWLOC_CPUKINDS_H
#define HWLOC_CPUKINDS_H

#include "hwloc.h"

#ifdef __cplusplus
```

This is a function that returns information about a CPU set if the platform or operating system does not expose any information about CPU cores. The function takes a single parameter, `cpu_set`, which is a `hwloc.topology.CPUSet` object. If the `cpu_set` object does not describe any CPU cores, the function returns an empty array `[]` with an `hwloc.topology.CPUKind` index of `-1`.

If the `cpu_set` object describes some CPU cores, the function uses two methods to gather information about the CPU kind. The first method is to compare the CPU cores using frequencies. The second method is to compare the CPU cores using their `CoreType` property. The function returns an abstracted efficiency value, an array of `hwloc.topology.CPUKindInfoAttributes` objects, and an array of `hwloc.topology.CPUKind` objects.

The efficiency value is a measure of the intrinsic performance of the CPU cores. A higher efficiency value means greater performance, but at the cost of lower power consumption or energy efficiency. The efficiency value is gathered from the operating system if it is available. If the operating system does not expose efficiencies and core frequencies, the function uses the second method to compute the efficiency.

If the operating system fails to expose efficiencies and core frequencies, the function returns an array with all kinds having an unknown efficiency value.


```cpp
extern "C" {
#elif 0
}
#endif

/** \defgroup hwlocality_cpukinds Kinds of CPU cores
 *
 * Platforms with heterogeneous CPUs may have some cores with
 * different features or frequencies.
 * This API exposes identical PUs in sets called CPU kinds.
 * Each PU of the topology may only be in a single kind.
 *
 * The number of kinds may be obtained with hwloc_cpukinds_get_nr().
 * If the platform is homogeneous, there may be a single kind
 * with all PUs.
 * If the platform or operating system does not expose any
 * information about CPU cores, there may be no kind at all.
 *
 * The index of the kind that describes a given CPU set
 * (if any, and not partially)
 * may be obtained with hwloc_cpukinds_get_by_cpuset().
 *
 * From the index of a kind, it is possible to retrieve information
 * with hwloc_cpukinds_get_info():
 * an abstracted efficiency value,
 * and an array of info attributes
 * (for instance the "CoreType" and "FrequencyMaxMHz",
 *  see \ref topoattrs_cpukinds).
 *
 * A higher efficiency value means greater intrinsic performance
 * (and possibly less performance/power efficiency).
 * Kinds with lower efficiency values are ranked first:
 * Passing 0 as \p kind_index to hwloc_cpukinds_get_info() will
 * return information about the CPU kind with lower performance
 * but higher energy-efficiency.
 * Higher \p kind_index values would rather return information
 * about power-hungry high-performance cores.
 *
 * When available, efficiency values are gathered from the operating system.
 * If so, \p cpukind_efficiency is set in the struct hwloc_topology_discovery_support array.
 * This is currently available on Windows 10, Mac OS X (Darwin),
 * and on some Linux platforms where core "capacity" is exposed in sysfs.
 *
 * If the operating system does not expose core efficiencies natively,
 * hwloc tries to compute efficiencies by comparing CPU kinds using
 * frequencies (on ARM), or core types and frequencies (on other architectures).
 * The environment variable HWLOC_CPUKINDS_RANKING may be used
 * to change this heuristics, see \ref envvar.
 *
 * If hwloc fails to rank any kind, for instance because the operating
 * system does not expose efficiencies and core frequencies,
 * all kinds will have an unknown efficiency (\c -1),
 * and they are not indexed/ordered in any specific way.
 *
 * @{
 */

```

这段代码定义了一个名为 `hwloc_cpukinds_get_nr` 的函数，它接受一个 `hwloc_topology_t` 的输入参数 `topology`，然后返回不同 CPU 核心类型的数量。

函数有两个参数，一个 `unsigned long flags` 标志，用于指定获取 CPU 核心类型的方式，它的值可以是 0；另一个 `hwloc_topology_t` 的输入参数 `topology`，用于返回 CPU 核心的布局信息。

函数的实现如下：
```cppc
int hwloc_cpukinds_get_nr(hwloc_topology_t topology, unsigned long flags);
```
首先，函数将输入参数 `topology` 传递给 `hwloc_topology_t` 类型的变量 `topology`，然后使用 `unsigned long` 类型的标志 `flags` 获取指定 CPU 核心类型的方式，它的值可以是 0。

函数的返回值类型是一个 `int` 类型，用于表示上一步获取到的 CPU 核心类型数量，它是非负的整数。如果函数成功获取了 CPU 核心类型信息，则返回值为 CPU 核心类型的数量；如果函数遇到错误，则返回值为一个错误码，错误码的值为 `-1`。


```cpp
/** \brief Get the number of different kinds of CPU cores in the topology.
 *
 * \p flags must be \c 0 for now.
 *
 * \return The number of CPU kinds (positive integer) on success.
 * \return \c 0 if no information about kinds was found.
 * \return \c -1 with \p errno set to \c EINVAL if \p flags is invalid.
 */
HWLOC_DECLSPEC int
hwloc_cpukinds_get_nr(hwloc_topology_t topology,
                      unsigned long flags);

/** \brief Get the index of the CPU kind that contains CPUs listed in \p cpuset.
 *
 * \p flags must be \c 0 for now.
 *
 * \return The index of the CPU kind (positive integer or 0) on success.
 * \return \c -1 with \p errno set to \c EXDEV if \p cpuset is
 * only partially included in the some kind.
 * \return \c -1 with \p errno set to \c ENOENT if \p cpuset is
 * not included in any kind, even partially.
 * \return \c -1 with \p errno set to \c EINVAL if parameters are invalid.
 */
```

`hwloc_cpukinds_get_by_cpuset`函数用于获取一个CPU设置中的CPU类型及其相关信息。它接受一个hwloc_topology_t类型的topology作为第一个参数，hwloc_const_bitmap_t类型的cpuset作为第二个参数，以及一个unsigned long类型的flags作为第三个参数。

该函数首先通过`hwloc_cpukinds_get_nr`函数获取指定CPU设置中可用的CPU类型数目，然后根据该数目填充`cpuset`，接着获取指定CPU类型的效率（通过`hwloc_topology_get_efficiency_model`函数获取），最后返回相关信息和排名（通过`hwloc_cpukinds_get_nr`函数获取）。

该函数在`hwloc_cpukinds_get_by_cpuset`中成功执行时返回0，如果遇到任何错误，则返回-1，具体的错误代码可以在函数的实现中通过`errno`变量获取。


```cpp
HWLOC_DECLSPEC int
hwloc_cpukinds_get_by_cpuset(hwloc_topology_t topology,
                             hwloc_const_bitmap_t cpuset,
                             unsigned long flags);

/** \brief Get the CPU set and infos about a CPU kind in the topology.
 *
 * \p kind_index identifies one kind of CPU between 0 and the number
 * of kinds returned by hwloc_cpukinds_get_nr() minus 1.
 *
 * If not \c NULL, the bitmap \p cpuset will be filled with
 * the set of PUs of this kind.
 *
 * The integer pointed by \p efficiency, if not \c NULL will, be filled
 * with the ranking of this kind of CPU in term of efficiency (see above).
 * It ranges from \c 0 to the number of kinds
 * (as reported by hwloc_cpukinds_get_nr()) minus 1.
 *
 * Kinds with lower efficiency are reported first.
 *
 * If there is a single kind in the topology, its efficiency \c 0.
 * If the efficiency of some kinds of cores is unknown,
 * the efficiency of all kinds is set to \c -1,
 * and kinds are reported in no specific order.
 *
 * The array of info attributes (for instance the "CoreType",
 * "FrequencyMaxMHz" or "FrequencyBaseMHz", see \ref topoattrs_cpukinds)
 * and its length are returned in \p infos or \p nr_infos.
 * The array belongs to the topology, it should not be freed or modified.
 *
 * If \p nr_infos or \p infos is \c NULL, no info is returned.
 *
 * \p flags must be \c 0 for now.
 *
 * \return \c 0 on success.
 * \return \c -1 with \p errno set to \c ENOENT if \p kind_index does not match any CPU kind.
 * \return \c -1 with \p errno set to \c EINVAL if parameters are invalid.
 */
```

This function appears to be a part of a higher-level software library that manages hardware resources, such as CPUs and GPUs, on a HWLOC (Highway Quality Log-on) platform. It appears to allow users to register CPUs in a topology, where different CPUs can be grouped together into "kinds", and to provide information about each kind.

The function takes as input:

* a count of the CPUs in the specified "cpuset"
* a bitmap representing the CPUs in the cpuset, with the index of the row and column corresponding to the CPU index in the cpuset
* an optional array of integers with an index of the "kind\_index" of each kind
* an array of integers with an index of the "efficiency" of each kind
* an array of integers with an index of the "nr\_infos" names and values of each kind
* a pointer to an array of information about each kind, of size nr\_infos
* a pointer to the flags that should be set for the registration

It appears to be registering the CPUs in the cpuset, and the "kind\_index" is used to determine the index of the kind in the registration. The "efficiency" is also being set to provide an abstracted efficiency value for the registration. The number of infos that can be provided for each kind is also being set.

It is also seems to handle the case where the registration of multiple CPUs in the same topology may cause conflicts. The function returns 0 on success, and -1 with an error code if some of the parameters are invalid.


```cpp
HWLOC_DECLSPEC int
hwloc_cpukinds_get_info(hwloc_topology_t topology,
                        unsigned kind_index,
                        hwloc_bitmap_t cpuset,
                        int *efficiency,
                        unsigned *nr_infos, struct hwloc_info_s **infos,
                        unsigned long flags);

/** \brief Register a kind of CPU in the topology.
 *
 * Mark the PUs listed in \p cpuset as being of the same kind
 * with respect to the given attributes.
 *
 * \p forced_efficiency should be \c -1 if unknown.
 * Otherwise it is an abstracted efficiency value to enforce
 * the ranking of all kinds if all of them have valid (and
 * different) efficiencies.
 *
 * The array \p infos of size \p nr_infos may be used to provide
 * info names and values describing this kind of PUs.
 *
 * \p flags must be \c 0 for now.
 *
 * Parameters \p cpuset and \p infos will be duplicated internally,
 * the caller is responsible for freeing them.
 *
 * If \p cpuset overlaps with some existing kinds, those might get
 * modified or split. For instance if existing kind A contains
 * PUs 0 and 1, and one registers another kind for PU 1 and 2,
 * there will be 3 resulting kinds:
 * existing kind A is restricted to only PU 0;
 * new kind B contains only PU 1 and combines information from A
 * and from the newly-registered kind;
 * new kind C contains only PU 2 and only gets information from
 * the newly-registered kind.
 *
 * \note The efficiency \p forced_efficiency provided to this function
 * may be different from the one reported later by hwloc_cpukinds_get_info()
 * because hwloc will scale efficiency values down to
 * between 0 and the number of kinds minus 1.
 *
 * \return \c 0 on success.
 * \return \c -1 with \p errno set to \c EINVAL if some parameters are invalid,
 * for instance if \p cpuset is \c NULL or empty.
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的函数，属于hwloc_cpukinds_register类型。它接受4个参数：

1. hwloc_topology_t：表示输入的拓扑结构。
2. hwloc_bitmap_t：表示输入的CPU集合。
3.  forced_efficiency：一个整数，表示强制效率，可以通过set到非0值来强制某些CPU在某些时刻工作。
4. unsigned long flags：表示与输入有关的一组标志，这些标志可以用来组合其他函数的输出。

函数接受4个整数类型的参数，然后将它们存储在一个HWLOC_INFO_S类型的结构体中，最后将这个结构体作为参数返回。

函数的定义没有包含任何注释。


```cpp
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

# `src/3rdparty/hwloc/include/hwloc/cuda.h`

这段代码定义了一系列 macro，它们用于帮助在 hwloc 和 CUDA 驱动程序 API 之间进行交互。具体来说，这些宏可以用于从 hwloc 获取 topology 信息，以便在 CUDA 设备上执行相应的计算。

注意，本文件需要包含在项目的头文件中，否则无法使用 hwloc 和 CUDA 驱动程序 API。


```cpp
/*
 * Copyright © 2010-2021 Inria.  All rights reserved.
 * Copyright © 2010-2011 Université Bordeaux
 * Copyright © 2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and the CUDA Driver API.
 *
 * Applications that use both hwloc and the CUDA Driver API may want to
 * include this file so as to get topology information for CUDA devices.
 *
 */

```



这段代码是一个CUDA头的定义，它包括了对CUDA头文件、autogenerated配置文件、CUDA库以及某些Linux特定头文件的引用。

具体来说，这个代码的作用是定义了一个名为"hwloc_cuda.h"的头文件，其中包含了与CUDA相关的定义、宏和函数。CUDA是一个并行计算库，可以用来进行高性能计算，尤其是在GPU上进行计算。通过引入这个CUDA库，可以使得在使用CUDA时，不需要手动设置环境变量等相关配置。

这个代码还引入了autogenerated配置文件和CUDA库的定义，这些定义会在编译时进行检查和编译。另外，这个代码也包含了两个条件分支，一个是在Linux系统上编译，另一个是在CUDA系统上编译。这两个分支分别引入了与Linux和CUDA相关的头文件和库。


```cpp
#ifndef HWLOC_CUDA_H
#define HWLOC_CUDA_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <cuda.h>


#ifdef __cplusplus
extern "C" {
```

这段代码是一个CUDA程序的头文件，它定义了一个名为“hwlocality_cuda”的接口。这个接口提供了使用CUDA Driver API来检索有关CUDA设备拓扑信息的方法。

具体来说，这个接口接受一个CUDA设备对象（通过`cuda_device_t`结构体）作为参数，然后返回 device 的 domain（内存空间）、bus（总线）和device ID（设备ID）等三个值。使用这个接口，用户可以轻松地获取CUDA设备的拓扑结构信息，以便更好地使用CUDA Driver API。


```cpp
#endif


/** \defgroup hwlocality_cuda Interoperability with the CUDA Driver API
 *
 * This interface offers ways to retrieve topology information about
 * CUDA devices when using the CUDA Driver API.
 *
 * @{
 */

/** \brief Return the domain, bus and device IDs of the CUDA device \p cudevice.
 *
 * Device \p cudevice must match the local machine.
 */
```

这段代码定义了一个名为 `hwloc_cuda_get_device_pci_ids` 的函数，属于 `hwloc_cuda_device_t` 系列的函数。它的作用是获取一块设备的 PCI ID。函数接受四个参数：

- `topology`：一个 `hwloc_topology_t` 类型的参数，用于指定数据根结构。
- `cudevice`：一个 `CUdevice` 类型的参数，用于获取 CUDA 设备的句柄。
- `domain`：一个指向 `int` 类型的参数，用于存储设备的领域（如 0，以便在 `hwloc_topology_cuda_domain_get_nod町` 中使用）。
- `bus`：一个指向 `int` 类型的参数，用于存储设备的总线 ID。
- `dev`：一个指向 `int` 类型的参数，用于存储设备的设备 ID。

函数首先检查 `cudevice` 是否为有效的 CUDA 设备句柄，如果没有错误，则返回领域值，否则输出错误并返回 -1。接下来，函数依次获取总线和设备 ID，并返回 0，如果任何一次获取失败，则输出错误并返回 -1。


```cpp
static __hwloc_inline int
hwloc_cuda_get_device_pci_ids(hwloc_topology_t topology __hwloc_attribute_unused,
			      CUdevice cudevice, int *domain, int *bus, int *dev)
{
  CUresult cres;

#if CUDA_VERSION >= 4000
  cres = cuDeviceGetAttribute(domain, CU_DEVICE_ATTRIBUTE_PCI_DOMAIN_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }
#else
  *domain = 0;
#endif
  cres = cuDeviceGetAttribute(bus, CU_DEVICE_ATTRIBUTE_PCI_BUS_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }
  cres = cuDeviceGetAttribute(dev, CU_DEVICE_ATTRIBUTE_PCI_DEVICE_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }

  return 0;
}

```

这段代码是一个 CUDA 库函数，它的作用是获取一个与设备物理位置相近的 CPU 集，并将该 CPU 集存储在 p set 变量中。这个函数仅返回设备的局部性，如果需要更多的信息，可以使用 OS 对象，请参阅 hwloc\_cuda\_get\_device\_osdev() 和 hwloc\_cuda\_get\_device\_osdev\_by\_index()。

在这段注释中，开发人员解释了函数的用途，说明了该函数仅在 Linux 系统有意义，因为在其他操作系统上，该函数将返回完整的 CPU 集。


```cpp
/** \brief Get the CPU set of processors that are physically
 * close to device \p cudevice.
 *
 * Store in \p set the CPU-set describing the locality of the CUDA device \p cudevice.
 *
 * Topology \p topology and device \p cudevice must match the local machine.
 * I/O devices detection and the CUDA component are not needed in the topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_cuda_get_device_osdev()
 * and hwloc_cuda_get_device_osdev_by_index().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux; other systems will simply get a full cpuset.
 */
```

这段代码的作用是获取一个CUDA设备的设备ID，并将该设备的设备ID存储在一个CUDA设备设置中。如果该设备ID无法从系统中获取，或者系统上存在多个具有相同ID的设备，则抛出错误并返回-1。

具体来说，代码首先通过hwloc_cuda_get_device_pci_ids函数获取一个CUDA设备的设备ID和域ID。接着，代码使用sprintf函数将获取到的设备ID转换为树形路径，并使用hwloc_linux_read_path_as_cpumask函数将该路径作为掩码与设置进行与操作。如果设置中的所有CPU都未被分配到系统中，则将设置结果复制到设置中。最后，如果上述步骤仍无法成功获取设备ID，则抛出错误并返回-1。


```cpp
static __hwloc_inline int
hwloc_cuda_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
			     CUdevice cudevice, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* If we're on Linux, use the sysfs mechanism to get the local cpus */
#define HWLOC_CUDA_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_CUDA_DEVICE_SYSFS_PATH_MAX];
  int domainid, busid, deviceid;

  if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domainid, &busid, &deviceid))
    return -1;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", domainid, busid, deviceid);
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个 if-else 语句，用于在 CUDA 设备存在时获取相应的 hwloc PCI 设备对象。

如果没有 CUDA 设备，那么该函数将返回 NULL。否则，它将使用 hwloc_topology_get_complete_cpuset 函数获取 CUDA 设备的 topology，并将其传递给 hwloc_bitmap_copy 函数，该函数将创建一个与 CUDA 设备对应的 bitmap 映射。

函数的第二个参数是一个指向 hwloc_topology_t 类型的变量 topology，它表示一个topology类型的数据结构，它应该与 local machine 中的I/O 设备检测设置相匹配。如果 topology 变量不匹配 local machine，或者 I/O 设备检测未启用，那么函数的行为将不可预测。


```cpp
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc PCI device object corresponding to the
 * CUDA device \p cudevice.
 *
 * \return The hwloc PCI device object describing the CUDA device \p cudevice.
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p cudevice must match the local machine.
 * I/O devices detection must be enabled in topology \p topology.
 * The CUDA component is not needed in the topology.
 */
```

这段代码定义了一个名为`hwloc_cuda_get_device_pcidev`的函数，它的作用是返回一个设备ID，该设备符合CUDA设备的PCI ID。

该函数的实现依赖于另外两个名为`hwloc_cuda_get_device_pci_ids`和`hwloc_get_pcidev_by_busid`的函数。其中，`hwloc_cuda_get_device_pci_ids`函数用于获取CUDA设备的PCI ID，而`hwloc_get_pcidev_by_busid`函数在获取到PCI ID后，使用它来返回相应的hwloc设备对象。

由于该函数需要与本地机器的topology和device匹配，并且I/O设备检测和CUDA组件都需要在topology中启用，因此如果PCI设备被过滤出，该函数可能无法正常工作。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_pcidev(hwloc_topology_t topology, CUdevice cudevice)
{
  int domain, bus, dev;

  if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domain, &bus, &dev))
    return NULL;

  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, 0);
}

/** \brief Get the hwloc OS device object corresponding to CUDA device \p cudevice.
 *
 * \return The hwloc OS device object that describes the given CUDA device \p cudevice.
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p cudevice must match the local machine.
 * I/O devices detection and the CUDA component must be enabled in the topology.
 * If not, the locality of the object may still be found using
 * hwloc_cuda_get_device_cpuset().
 *
 * \note This function cannot work if PCI devices are filtered out.
 *
 * \note The corresponding hwloc PCI device may be found by looking
 * at the result parent pointer (unless PCI devices are filtered out).
 */
```

这段代码定义了一个名为 `hwloc_cuda_get_device_osdev` 的函数，它的作用是返回一个硬件设备对象，用于在给定的 topology 结构中查找与给定 CUDA 设备的操作系统设备。

函数接收两个参数：`hwloc_topology_t` 和 `CUdevice` 类型的对象，用于表示要查询的 topology 结构和 CUDA 设备。函数内部首先定义了一个名为 `osdev` 的空对象，用于存储找到的操作系统设备。然后函数使用嵌套循环，在 topology 中查找与给定 CUDA 设备匹配的操作系统设备。如果找到匹配的设备，函数将返回该设备对象。否则，函数返回 `NULL`，表示没有找到匹配的设备。

函数中还定义了一个名为 `hwloc_get_next_osdev` 的函数，它的作用是在 topology 中查找下一个操作系统设备。这个函数将返回一个指向下一个设备的指针，如果当前 topology 结构中没有设备，函数将返回 `NULL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_osdev(hwloc_topology_t topology, CUdevice cudevice)
{
	hwloc_obj_t osdev = NULL;
	int domain, bus, dev;

	if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domain, &bus, &dev))
		return NULL;

	osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		hwloc_obj_t pcidev = osdev->parent;
		if (strncmp(osdev->name, "cuda", 4))
			continue;
		if (pcidev
		    && pcidev->type == HWLOC_OBJ_PCI_DEVICE
		    && (int) pcidev->attr->pcidev.domain == domain
		    && (int) pcidev->attr->pcidev.bus == bus
		    && (int) pcidev->attr->pcidev.dev == dev
		    && pcidev->attr->pcidev.func == 0)
			return osdev;
		/* if PCI are filtered out, we need a info attr to match on */
	}

	return NULL;
}

```

这段代码是一个 CUDA 设备对象函数，它的作用是获取与给定 CUDA 设备索引（ ）相对应的 hwloc OS 设备对象。如果没有找到相应的设备对象，函数返回 NULL。这个函数是建立在 topology 参数上的，topology 参数表示了一个 CUDA 设备的 topology，即设备及其对应的 I/O 设备。为了使这个函数有效，需要开启 I/O 设备检测和 CUDA 组件。同时，为了在 topology 中查找相应的 CUDA 设备，需要先获取 CUDA 设备的 topology 后代对象。然而，如果 PCI 设备被过滤了，可能无法通过 topology 获取相应的 CUDA 设备。


```cpp
/** \brief Get the hwloc OS device object corresponding to the
 * CUDA device whose index is \p idx.
 *
 * \return The hwloc OS device object describing the CUDA device whose index is \p idx.
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the CUDA component must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 *
 * \note This function is identical to hwloc_cudart_get_device_osdev_by_index().
 */
```

这段代码定义了一个名为 `hwloc_cuda_get_device_osdev_by_index` 的函数，它接受一个 `hwloc_topology_t` 的输入参数，和一个 `unsigned int` 的输入参数 `idx`。

函数内部首先定义了一个名为 `osdev` 的 `hwloc_obj_t` 类型的变量，用于保存返回的设备对象。

接着，函数使用一个 while 循环，该循环从传递给函数的 `topology` 中的第一个 `osdev` 对象开始，逐个尝试获取下一个 `osdev` 对象。

在循环体内部，函数使用一系列条件判断来筛选出合适的 `osdev` 对象：

1. 如果 `osdev->attr->osdev.type` 等于 `HWLOC_OBJ_OSDEV_COPROC` 并且 `osdev->name` 包含字符串 `"cuda"`，那么就认为这个 `osdev` 对象可能是要找的设备对象。
2. 如果上面两个条件都不符合，再通过 `atoi` 函数将 `osdev->name` 中的字符串转换为整数，并与 `idx` 进行比较。如果 `idx` 与这个结果相等，那么就返回这个 `osdev` 对象。

最后，如果循环结束后仍然没有找到合适的 `osdev` 对象，函数返回一个 `NULL` 类型的变量表示没有找到匹配的设备对象。

该函数的作用是返回一个 `hwloc_obj_t` 类型的设备对象，它可能是基于 CUDA 技术的 GPU 设备对象。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
{
	hwloc_obj_t osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		if (HWLOC_OBJ_OSDEV_COPROC == osdev->attr->osdev.type
		    && osdev->name
		    && !strncmp("cuda", osdev->name, 4)
		    && atoi(osdev->name + 4) == (int) idx)
			return osdev;
	}
	return NULL;
}

/** @} */


```

这段代码是一个C/C++编程语言的预处理指令。其中包含两个条件判断。

第一个条件判断为 #ifdef __cplusplus，如果这个预处理指令的前缀存在，则表示这个文件是使用C++语言编译的，并且编译器支持C++的__cplusplus预处理指令。这个条件判断的作用是用来检查当前源文件是否为使用C++语言编译的。

第二个条件判断为 #endif，如果这个预处理指令的前缀不存在，则表示这个文件不是使用C++语言编译的。这个条件判断的作用是用来检查当前源文件是否为使用C语言编译的。

如果当前源文件是使用C++语言编译的，并且编译器支持C++的__cplusplus预处理指令，则执行第一个条件判断块，输出一个换行符；否则执行第二个条件判断块，不输出任何内容。

最终的作用是，根据源文件的语言编译情况，输出一个换行符。


```cpp
#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_CUDA_H */

```

# `src/3rdparty/hwloc/include/hwloc/cudart.h`

这段代码定义了一系列 macro，它们有助于在 hwloc 和 CUDA 运行时 API 之间进行交互。具体来说，这些宏可以用于从 hwloc 获取 topology 信息，以便在 CUDA 设备上进行 topology 查询。

注意，这段代码中并没有包含 CUDA runtime API 的实现，因为这不是该文件的作用。相反，它定义了一些通用的功能，以便在 hwloc 和 CUDA 运行时 API 之间进行交互。


```cpp
/*
 * Copyright © 2010-2021 Inria.  All rights reserved.
 * Copyright © 2010-2011 Université Bordeaux
 * Copyright © 2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and the CUDA Runtime API.
 *
 * Applications that use both hwloc and the CUDA Runtime API may want to
 * include this file so as to get topology information for CUDA devices.
 *
 */

```

这段代码定义了一个头文件名为 "hwloc_cudart.h"，其中包含了 CUDA 版本的 C 语言代码。这个头文件被包含在了 HWLOC_CUDART_H 函数中，也就是定义了这个头文件的作用。

在头文件中，首先引入了 HWLOC 和 hwloc 头文件，以及自身定义的一个名为 "hwloc_cudart_api"，这个函数会定义 CUDA 版本的 CUDA 函数。

接下来，通过 #ifdef 预处理指令，判断操作系统是否支持 CUDA，如果支持，那么就包含了一个名为 "linux" 的宏，这个宏会包含在头文件中。

在main函数中，首先包含了一个 hwloc 和 linux 头文件，然后通过 非洲 gate 开放头文件，最后定义了这个头文件的作用，也就是 CUDA 版本的 C 语言代码。


```cpp
#ifndef HWLOC_CUDART_H
#define HWLOC_CUDART_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <cuda.h> /* for CUDA_VERSION */
#include <cuda_runtime_api.h>


#ifdef __cplusplus
```

这段代码是一个CUDA C++的函数声明，定义了一个名为"hwlocality_cudart"的函数。该函数声明了从CUDA Runtime API获取CUDA设备局部拓扑信息的方法，并包含在头文件中。

具体来说，该函数通过引用一个名为"__C"的外部符号，告诉编译器可以使用CUDA C++语言中的C函数。函数声明中包含两个 preprocessor 指令，分别是 "#endif" 和 "#define"。其中，#endif 表示这个指令所在的代码块已经结束，可以随时插入其他代码。#define 则表示定义了一个名为 "hwlocality_cudart" 的宏，并在后续代码中使用该宏时将其实际的函数名替换为 "hwlocality_cudart"。

函数体中包含一个函数声明，声明了一个名为 "retainTopology" 的函数，该函数返回了 CUDA 设备的领域(device domain)、总线(device bus)和设备 ID(device ID)，其中领域、总线和设备 ID 都是整数类型。函数声明中还包含一个名为 "deviceIndex" 的参数，类型为 int32_t，表示要检索的 CUDA 设备索引。

函数的作用是返回选定 CUDA 设备的领域、总线和设备 ID，以帮助用户获取设备拓扑信息，从而更好地使用 CUDA 运行时 API。


```cpp
extern "C" {
#endif


/** \defgroup hwlocality_cudart Interoperability with the CUDA Runtime API
 *
 * This interface offers ways to retrieve topology information about
 * CUDA devices when using the CUDA Runtime API.
 *
 * @{
 */

/** \brief Return the domain, bus and device IDs of the CUDA device whose index is \p idx.
 *
 * Device index \p idx must match the local machine.
 */
```

这段代码的作用是获取通过CUDA的CUDART设备从给定的CUDA设备的设备PCI ID中提取出domain ID。然后，通过这些device PCI ID，可以定位到对应的CUDA设备并获取其设备的domain ID和bus ID。

devicePCI ID是通过ROCm（CUDA寄存器访问模型）获取的，它是一个32位的ID，由CUDA硬件制造商预先分配。domain ID和bus ID是通过CUDA的CUDART设备访问器函数获取的，这些函数允许您通过CUDA设备对象访问CUDA设备。

在使用上，devicePCI ID是用来在CUDA设备对象中查找设备的。一旦设备对象被获取，可以通过访问其domain ID和bus ID来获取设备的配置。这些信息对于了解设备如何工作以及如何分配其 resources非常重要。


```cpp
static __hwloc_inline int
hwloc_cudart_get_device_pci_ids(hwloc_topology_t topology __hwloc_attribute_unused,
				int idx, int *domain, int *bus, int *dev)
{
  cudaError_t cerr;
  struct cudaDeviceProp prop;

  cerr = cudaGetDeviceProperties(&prop, idx);
  if (cerr) {
    errno = ENOSYS;
    return -1;
  }

#if CUDA_VERSION >= 4000
  *domain = prop.pciDomainID;
```

这段代码是一个C函数，名为“get_cpu_set”，功能是获取一个与设备ID prop.pciDeviceID物理位置相近的CPU集合。它将被存储在prop.set中，用于描述CUDA设备的局部性。

代码首先包含一个预处理指令#else，如果没有这个指令，则执行下面语句：将domain的值为0。这个值在后面还被使用。

接着，代码定义了两个整型变量bus和dev，分别表示设备ID和物理位置的设备。

然后代码回溯到if语句中，判断了当前函数是否在实现一个更复杂的版本的逻辑。如果不是，则执行简单的操作，否则使用OS对象获取更详细的信息。

最后，函数返回局部性，并作为参数接收一个整型参数。


```cpp
#else
  *domain = 0;
#endif

  *bus = prop.pciBusID;
  *dev = prop.pciDeviceID;

  return 0;
}

/** \brief Get the CPU set of processors that are physically
 * close to device \p idx.
 *
 * Store in \p set the CPU-set describing the locality of the CUDA device
 * whose index is \p idx.
 *
 * Topology \p topology and device \p idx must match the local machine.
 * I/O devices detection and the CUDA component are not needed in the topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_cudart_get_device_osdev_by_index().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux; other systems will simply get a full cpuset.
 */
```

这段代码的作用是获取CUDA设备的设备ID，如果当前系统不是Linux系统，会抛出错误。

具体来说，代码首先通过`hwloc_cudart_get_device_pci_ids`函数获取CUDA设备的PCI ID，然后判断当前系统是否为Linux系统，如果不是，则会抛出错误。接着，代码使用`hwloc_topology_is_thissystem`函数判断当前系统是否为Linux系统，如果是，则会执行后续操作。

接着，代码使用`sprintf`函数将PCI ID转换为字符串，并使用`/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus`格式化字符串，最后在Linux系统中使用`hwloc_linux_read_path_as_cpumask`函数获取本地CPUID，并将获取到的ID添加到设置中，如果获取失败或者设置中没有对应的CPUID，则会将设置覆盖为全置。


```cpp
static __hwloc_inline int
hwloc_cudart_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
			       int idx, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* If we're on Linux, use the sysfs mechanism to get the local cpus */
#define HWLOC_CUDART_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_CUDART_DEVICE_SYSFS_PATH_MAX];
  int domain, bus, dev;

  if (hwloc_cudart_get_device_pci_ids(topology, idx, &domain, &bus, &dev))
    return -1;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", (unsigned) domain, (unsigned) bus, (unsigned) dev);
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个if语句的else部分。if语句的的条件为true或者file||err。如果条件为true，则执行if语句块内的代码，否则跳过if语句块。

if语句块内的代码首先使用hwloc_bitmap_copy函数复制一个名为set的hwloc_topology_get_complete_cpuset(topology)的元组，该元组包含topology所支持的CUDA设备的ID。然后，代码使用return 0；语句返回0，表明成功执行if语句块内的代码。

if语句块内的代码的else部分是一个空的，因此如果if语句块的条件为true，则else部分的代码不会被执行。


```cpp
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc PCI device object corresponding to the
 * CUDA device whose index is \p idx.
 *
 * \return The hwloc PCI device object describing the CUDA device whose index is \p idx.
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p idx must match the local machine.
 * I/O devices detection must be enabled in topology \p topology.
 * The CUDA component is not needed in the topology.
 */
```

这段代码定义了一个名为 `hwloc_cudart_get_device_pcidev` 的函数，它接受一个 `hwloc_topology_t` 的输入参数和一个整数 `idx`，并返回一个指向 CUDA 设备的 `hwloc_obj_t` 类型的对象。

具体来说，函数首先通过 `hwloc_cudart_get_device_pci_ids` 函数获取 CUDA 设备在当前系统上的 PCI ID，然后使用这些 ID 在 topology 中查找对应的 CUDA 设备。如果成功找到了匹配的设备，函数将返回该设备的 `hwloc_obj_t` 类型；否则，函数返回 NULL。

函数的第二个参数 `topology` 是一个表示拓扑结构的对象，它用于指定当前系统的硬件资源。这个参数在调用 `hwloc_cudart_get_device_pci_ids` 函数时用于获取 CUDA 设备在 topology 中的父设备。

函数的第三个参数 `idx` 表示要查找的 CUDA 设备的索引。这个参数在 `hwloc_cudart_get_device_pci_ids` 函数中传递给 `domain`、`bus` 和 `dev` 变量，用于获取设备的索引。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_cudart_get_device_pcidev(hwloc_topology_t topology, int idx)
{
  int domain, bus, dev;

  if (hwloc_cudart_get_device_pci_ids(topology, idx, &domain, &bus, &dev))
    return NULL;

  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, 0);
}

/** \brief Get the hwloc OS device object corresponding to the
 * CUDA device whose index is \p idx.
 *
 * \return The hwloc OS device object describing the CUDA device whose index is \p idx.
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the CUDA component must be enabled in the topology.
 * If not, the locality of the object may still be found using
 * hwloc_cudart_get_device_cpuset().
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 *
 * \note This function is identical to hwloc_cuda_get_device_osdev_by_index().
 */
```

这段代码定义了一个名为 `hwloc_cudart_get_device_osdev_by_index` 的函数，它接受一个 `hwloc_topology_t` 的输入参数，和一个 `unsigned` 类型的索引参数。

函数首先定义了一个名为 `osdev` 的变量，该变量初始化为 `NULL`。

接下来，函数使用一个 `while` 循环，该循环在每次循环中获取一个 `hwloc_obj_t` 类型的变量作为子句。

循环的头部包含一个条件判断，如果当前子句的 `osdev` 对象满足以下两个条件：

1. `HWLOC_OBJ_OSDEV_COPROC` 属性与 `osdev` 对象的属性中的 `osdev.type` 属性相等，并且 `osdev.name` 中的字符串部分等于 `"cuda"` 中的字符串部分。
2. `osdev->name` 中的字符串部分是一个数字，且该数字等于索引参数 `idx`。

如果上述两个条件都满足，则返回 `osdev` 对象。

否则，函数使用 `hwloc_get_next_osdev` 函数获取下一个 `osdev` 对象，并将 `NULL` 赋给 `osdev` 变量。

函数的末尾包含一个返回 `NULL` 的语句，这意味着它不接受任何返回值。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_cudart_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
{
	hwloc_obj_t osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		if (HWLOC_OBJ_OSDEV_COPROC == osdev->attr->osdev.type
		    && osdev->name
		    && !strncmp("cuda", osdev->name, 4)
		    && atoi(osdev->name + 4) == (int) idx)
			return osdev;
	}
	return NULL;
}

/** @} */


```

这段代码是一个C语言中的预处理指令，名为“#ifdef __cplusplus”。它的作用是在编译时检查是否支持C++语言的特性。如果没有这个预处理指令，那么编译器会报错。

具体来说，这段代码包含了一个包含两个预处理指令的段落。第一个预处理指令是“#ifdef __cplusplus”，它会在编译时检查是否支持C++语言的编译器和特性。如果这个指令出现在#include <cstdio>或#include <cstdlib>等标准库头文件包含之前，那么说明编译器支持C++语言，这些标准的C语言库包含了对C++语言的支持。否则，编译器会报错。

第二个预处理指令是空字符串，它告诉编译器在预处理任何输入时不要进行任何操作。这个指令通常在预处理指令的结尾使用，以避免在预处理指令内部定义了其他预处理指令。

总之，这段代码的作用是检查C++语言的特性，并在编译时提供相应的错误提示。


```cpp
#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_CUDART_H */

```