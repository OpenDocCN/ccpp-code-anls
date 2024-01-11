# `xmrig\src\3rdparty\hwloc\include\hwloc\memattrs.h`

```
/*
 * 版权所有 © 2019-2022 Inria。
 * 保留所有权利。请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 内存节点属性。
 */

#ifndef HWLOC_MEMATTR_H
#define HWLOC_MEMATTR_H

#include "hwloc.h"

#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif

/** \brief 内存节点属性。 */
};

/** \brief 内存属性标识符。
 * 可能是 ::hwloc_memattr_id_e 中的一个，也可能是由 hwloc_memattr_register() 返回的新标识符。
 */
typedef unsigned hwloc_memattr_id_t;

/** \brief 根据给定名称返回内存属性的标识符。
 */
HWLOC_DECLSPEC int
hwloc_memattr_get_by_name(hwloc_topology_t topology,
                          const char *name,
                          hwloc_memattr_id_t *id);


/** \brief 位置类型。 */
enum hwloc_location_type_e {
  /** \brief 位置以 cpuset 形式给出，在位置 cpuset 联合字段中。 \hideinitializer */
  HWLOC_LOCATION_TYPE_CPUSET = 1,
  /** \brief 位置以对象形式给出，在位置对象联合字段中。 \hideinitializer */
  HWLOC_LOCATION_TYPE_OBJECT = 0
};

/** \brief 从哪里测量属性。 */
struct hwloc_location {
  /** \brief 位置类型。 */
  enum hwloc_location_type_e type;
  /** \brief 实际位置。 */
  union hwloc_location_u {
    /** \brief 位置作为 cpuset，当位置类型为 ::HWLOC_LOCATION_TYPE_CPUSET 时。 */
    hwloc_cpuset_t cpuset;
    /** \brief 位置作为对象，当位置类型为 ::HWLOC_LOCATION_TYPE_OBJECT 时。 */
    hwloc_obj_t object;
  } location;
};


/** \brief 用于选择目标 NUMA 节点的标志。 */
# 定义枚举类型 hwloc_local_numanode_flag_e
enum hwloc_local_numanode_flag_e {
  # 选择本地性大于给定 cpuset 的 NUMA 节点
  HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY = (1UL<<0),

  # 选择本地性小于给定 cpuset 的 NUMA 节点
  HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY = (1UL<<1),

  # 选择拓扑中的所有 NUMA 节点
  HWLOC_LOCAL_NUMANODE_FLAG_ALL = (1UL<<2)
};
/**
 * \brief 返回本地 NUMA 节点的数组。
 *
 * 默认情况下，仅选择本地性完全符合给定位置的 NUMA 节点。如果附加标志作为 ::hwloc_local_numanode_flag_e 的 OR 运算集合给出，则可能选择更多节点。
 *
 * 如果位置作为显式对象给出，则使用其 CPU 集合来查找具有相应本地性的 NUMA 节点。
 * 如果对象没有 CPU 集合（例如 I/O 对象），则使用 CPU 父对象（连接到 I/O 对象的位置）。
 *
 * 在输入时，\p nr 指向可能存储在 \p nodes 数组中的节点数。
 * 在输出时，\p nr 将被更改为存储的节点数，或者如果有足够的空间，则将存储的节点数。
 *
 * \note 这些 NUMA 节点中的一些可能没有任何内存属性值，因此在其他函数中可能不会报告为实际目标。
 *
 * \note 可以使用拓扑中 NUMA 节点的数量（通过对根对象节点集执行 hwloc_bitmap_weight() 获得）来分配 \p nodes 数组。
 *
 * \note 当给定对象 CPU 集合作为本地性时，例如一个 Package，并且标志同时包含 ::HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY 和 ::HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY 时，返回的数组对应于该对象的节点集。
 */
HWLOC_DECLSPEC int
hwloc_get_local_numanode_objs(hwloc_topology_t topology,
                              struct hwloc_location *location,
                              unsigned *nr,
                              hwloc_obj_t *nodes,
                              unsigned long flags);
/**
 * \brief 返回特定目标 NUMA 节点的属性值。
 *
 * 如果属性与特定的发起者无关（没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR），
 * 则位置 \p initiator 将被忽略，可以是 \c NULL。
 *
 * \p flags 目前必须为 \c 0。
 *
 * \note 当涉及到 CPU 核心执行的访问时，发起者 \p initiator 应该是 ::HWLOC_LOCATION_TYPE_CPUSET 类型。
 * ::HWLOC_LOCATION_TYPE_OBJECT 目前在 hwloc 内部未被使用，
 * 但用户可以使用它来提供有关 GPU 执行的主机内存访问的自定义信息。
 */
HWLOC_DECLSPEC int
hwloc_memattr_get_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t attribute,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t *value);
/**
 * \brief 返回给定属性和发起者的最佳目标 NUMA 节点。
 *
 * 如果属性与特定发起者无关（没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR），
 * 则位置 \p initiator 将被忽略，可以为 \c NULL。
 *
 * 如果 \p value 非 \c NULL，则相应的值将返回到那里。
 *
 * 如果多个目标具有相同的属性值，只返回一个（并且无法澄清如何选择那个）。
 * 想要检测具有相同/相似值的目标的应用程序，或者想要查看多个属性的值，
 * 应该使用 hwloc_memattr_get_value() 获取所有值，并手动选择他们认为最佳的目标。
 *
 * \p flags 目前必须为 \c 0。
 *
 * 如果没有匹配的目标，则返回 \c -1，并将 \p errno 设置为 \c ENOENT；
 *
 * \note 当涉及 CPU 核心执行的访问时，发起者 \p initiator 应该是 ::HWLOC_LOCATION_TYPE_CPUSET 类型。
 * ::HWLOC_LOCATION_TYPE_OBJECT 目前在 hwloc 内部未使用，但用户可以例如使用它来提供有关 GPU 执行的主机内存访问的自定义信息。
 */
HWLOC_DECLSPEC int
hwloc_memattr_get_best_target(hwloc_topology_t topology,
                              hwloc_memattr_id_t attribute,
                              struct hwloc_location *initiator,
                              unsigned long flags,
                              hwloc_obj_t *best_target, hwloc_uint64_t *value);
# 返回给定属性和目标 NUMA 节点的最佳初始化器
#
# 如果属性与特定初始化器无关（没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR），则返回 -1，并将 errno 设置为 EINVAL。
#
# 如果 value 非空，则将相应的值返回到那里。
#
# 如果多个初始化器具有相同的属性值，只返回一个（并且无法澄清选择了哪一个）。
# 希望检测具有相同/相似值的初始化器的应用程序，或者希望查看多个属性的值，应该使用 hwloc_memattr_get_value() 获取所有值，并手动选择他们认为最佳的初始化器。
#
# 返回的初始化器不应该被修改或释放，它属于拓扑结构。
#
# flags 现在必须为 0。
#
# 如果没有匹配的初始化器，则返回 -1，并将 errno 设置为 ENOENT。
HWLOC_DECLSPEC int
hwloc_memattr_get_best_initiator(hwloc_topology_t topology,
                                 hwloc_memattr_id_t attribute,
                                 hwloc_obj_t target,
                                 unsigned long flags,
                                 struct hwloc_location *best_initiator, hwloc_uint64_t *value);

/** @} */

# 返回内存属性的名称
HWLOC_DECLSPEC int
hwloc_memattr_get_name(hwloc_topology_t topology,
                       hwloc_memattr_id_t attribute,
                       const char **name);

# 返回给定属性的标志
#
# 标志是 ::hwloc_memattr_flag_e 的 OR 运算集。
HWLOC_DECLSPEC int
hwloc_memattr_get_flags(hwloc_topology_t topology,
                        hwloc_memattr_id_t attribute,
                        unsigned long *flags);
/** \brief 内存属性标志。
 * 由 hwloc_memattr_register() 给出，并由 hwloc_memattr_get_flags() 返回。
 */
enum hwloc_memattr_flag_e {
  /** \brief 此内存属性的最佳节点是具有较高值的节点。
   * 例如带宽。
   */
  HWLOC_MEMATTR_FLAG_HIGHER_FIRST = (1UL<<0),
  /** \brief 此内存属性的最佳节点是具有较低值的节点。
   * 例如延迟。
   */
  HWLOC_MEMATTR_FLAG_LOWER_FIRST = (1UL<<1),
  /** \brief 返回的值取决于给定的发起者。
   * 例如带宽和延迟，但不包括容量。
   */
  HWLOC_MEMATTR_FLAG_NEED_INITIATOR = (1UL<<2)
};

/** \brief 注册新的内存属性。
 *
 * 添加一个在 ::hwloc_memattr_id_e 中未定义的特定内存属性。
 * 标志是 ::hwloc_memattr_flag_e 的 OR 运算集。它必须至少包含
 * ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST 或 ::HWLOC_MEMATTR_FLAG_LOWER_FIRST 中的一个。
 */
HWLOC_DECLSPEC int
hwloc_memattr_register(hwloc_topology_t topology,
                       const char *name,
                       unsigned long flags,
                       hwloc_memattr_id_t *id);

/** \brief 为特定目标 NUMA 节点设置属性值。
 *
 * 如果属性与特定发起者无关
 * (它没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR)，
 * 位置 \p initiator 将被忽略，可以是 \c NULL。
 *
 * 发起者将被复制到拓扑结构中，
 * 调用者应该释放为存储发起者分配的任何内容，
 * 例如 cpuset。
 *
 * \p flags 现在必须为 \c 0。
 *
 * \note 当涉及到 CPU 核心执行的访问时，发起者 \p initiator 应该是 ::HWLOC_LOCATION_TYPE_CPUSET 类型。
 * ::HWLOC_LOCATION_TYPE_OBJECT 目前在 hwloc 内部未使用，
 * 但用户可以使用它来提供有关 GPU 执行的主机内存访问的自定义信息。
 */
HWLOC_DECLSPEC int
# 设置指定拓扑结构中的内存属性值
hwloc_memattr_set_value(
    hwloc_topology_t topology,  # 指向拓扑结构的指针
    hwloc_memattr_id_t attribute,  # 内存属性的标识符
    hwloc_obj_t target_node,  # 目标节点的对象
    struct hwloc_location *initiator,  # 初始化者的位置信息
    unsigned long flags,  # 标志位
    hwloc_uint64_t value  # 属性值
);
/**
 * \brief 返回具有给定属性值的目标 NUMA 节点。
 *
 * 在 \p targets 数组中返回给定属性的目标（如果有指定发起者）。
 * 如果 \p values 不是 \c NULL，则相应的属性值存储在其指向的数组中。
 *
 * 在输入时，\p nr 指向可能存储在 \p targets（和 \p values）数组中的目标数量。
 * 在输出时，\p nr 指向实际找到的目标（和值）的数量，即使其中一些无法存储在数组中。
 * 无法存储的目标将被忽略，但函数仍然返回成功（\c 0）。调用者可以通过比较函数调用前后 \p nr 指向的值来找出。
 *
 * 返回的目标不应该被修改或释放，它们属于拓扑结构。
 *
 * 如果属性与特定发起者无关（没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR），则忽略参数 \p initiator。
 * 否则，如果 \p initiator 可能非 \c NULL，则报告仅对该发起者具有值的目标。
 *
 * \p flags 目前必须为 \c 0。
 *
 * \note 此函数用于工具和调试（列出内部信息），而不是用于应用程序查询。应用程序应该使用 hwloc_get_local_numanode_objs() 选择有用的 NUMA 节点，然后查看它们的属性值。
 *
 * \note 当涉及 CPU 核心执行的访问时，\p initiator 应该是 ::HWLOC_LOCATION_TYPE_CPUSET 类型。
 * ::HWLOC_LOCATION_TYPE_OBJECT 目前在 hwloc 内部未使用，但用户可以例如使用它来提供有关 GPU 执行的主机内存访问的自定义信息。
 */
HWLOC_DECLSPEC int
# 返回具有给定属性和特定目标 NUMA 节点的发起者
# 在 initiators 数组中返回给定属性和目标节点的发起者
# 如果 values 不为 NULL，则将相应的属性值存储在它指向的数组中
# 在输入时，nr 指向可能存储在数组 initiators（和 values）中的发起者数量
# 在输出时，nr 指向实际找到的发起者（和值）的数量，即使其中一些无法存储在数组中
# 无法存储的发起者将被忽略，但函数仍然返回成功（0）。调用者可以通过比较函数调用前后 nr 指向的值来找出这一点
# 返回的发起者不应被修改或释放，它们属于拓扑结构
# flags 目前必须为 0
# 如果属性与特定发起者无关（它没有标志 ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR），则不返回发起者
# 注意：此函数用于工具和调试（列出内部信息），而不是用于应用程序查询。应用程序应该使用 hwloc_get_local_numanode_objs() 选择有用的 NUMA 节点，然后查看它们的一些相关发起者的属性值
HWLOC_DECLSPEC int
# 获取内存属性的初始化器
hwloc_memattr_get_initiators(hwloc_topology_t topology,  # 获取内存属性的初始化器，传入拓扑结构体和内存属性ID
                             hwloc_memattr_id_t attribute,  # 内存属性ID
                             hwloc_obj_t target_node,  # 目标节点对象
                             unsigned long flags,  # 标志位
                             unsigned *nr, struct hwloc_location *initiators, hwloc_uint64_t *values);  # 初始化器数量、初始化器位置、初始化器值

/** @} */  # 结束注释块

#ifdef __cplusplus  # 如果是 C++ 代码
} /* extern "C" */  # 使用 C 语言进行编译
#endif  # 结束条件编译

#endif /* HWLOC_MEMATTR_H */  # 结束头文件的条件编译
```