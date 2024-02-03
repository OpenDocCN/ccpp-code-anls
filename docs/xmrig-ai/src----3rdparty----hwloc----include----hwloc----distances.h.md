# `xmrig\src\3rdparty\hwloc\include\hwloc\distances.h`

```cpp
/*
 * 版权所有 © 2010-2022 Inria。
 * 保留所有权利。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 对象距离。
 */

#ifndef HWLOC_DISTANCES_H
#define HWLOC_DISTANCES_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h
#endif


#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_distances_get 检索对象之间的距离
 * @{
 */

/** \brief 一组对象之间的距离矩阵。
 *
 * 这个矩阵通常包含 NUMA 节点之间的延迟
 * （如 ACPI 规范中的系统局部性距离信息表 (SLIT) 中所报告的），
 * 这些延迟可能或可能不是物理上准确的。
 * 它对应于从一个节点的核心访问另一个节点的内存的延迟。
 * 相应的类型是 ::HWLOC_DISTANCES_KIND_FROM_OS | ::HWLOC_DISTANCES_KIND_FROM_USER。
 * 此距离结构的名称为 "NUMALatency"。
 * 其他距离结构包括 "XGMIBandwidth"、"XGMIHops"、
 * "XeLinkBandwidth" 和 "NVLinkBandwidth"。
 *
 * 该矩阵还可能包含随机对象集之间的带宽，
 * 可能由用户提供，如 \p kind 属性中所指定的。
 *
 * 指针 \p objs 和 \p values 不应被替换、重新分配、释放等。
 * 但是调用者可以修改 \p kind 以及 \p objs 和 \p values 数组的内容。
 * 例如，如果每个 Package 中只有一个 NUMA 节点，
 * 可以使用 hwloc_get_obj_with_same_locality() 将它们之间进行转换，
 * 并用相应的 Packages 替换 \p objs 数组中的 NUMA 节点。
 * 另请参阅 hwloc_distances_transform() 以对结构应用一些转换。
 */
# 定义了描述距离矩阵的数据结构
struct hwloc_distances_s {
  unsigned nbobjs;        /**< \brief 距离矩阵描述的对象数量。 */
  hwloc_obj_t *objs;        /**< \brief 距离矩阵描述的对象数组。
                 * 这些对象没有特定的顺序，
                 * 可以使用 hwloc_distances_obj_index() 和 hwloc_distances_obj_pair_values()
                 * 来方便地找到数组中的对象及其对应的值。
                 */
  unsigned long kind;        /**< \brief OR'ed 的 ::hwloc_distances_kind_e 集合。 */
  hwloc_uint64_t *values;    /**< \brief 对象之间距离的矩阵，存储为一维数组。
                 *
                 * 从第 i 个到第 j 个对象的距离存储在位置 i*nbobjs+j。
                 * 值的含义取决于 \p kind 属性。
                 */
};

/** \brief 距离矩阵的类型。
 *
 * struct hwloc_distances_s 的 \p kind 属性是一组 OR'ed 的类型。
 *
 * 格式为 HWLOC_DISTANCES_KIND_FROM_* 的类型指定了距离信息的来源，如果已知的话。
 *
 * 格式为 HWLOC_DISTANCES_KIND_MEANS_* 的类型指定了值是延迟还是带宽，如果适用的话。
 */
# 定义枚举类型 hwloc_distances_kind_e
enum hwloc_distances_kind_e {
  /** \brief 这些距离是从操作系统或硬件获取的。
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_FROM_OS = (1UL<<0),
  /** \brief 这些距离是由用户提供的。
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_FROM_USER = (1UL<<1),

  /** \brief 距离值类似于对象之间的延迟。
   * 对于更接近的对象，值较小，因此在矩阵的对角线上最小（对象与自身的距离）。
   * 它也可以是对象之间的网络跳数等。
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_MEANS_LATENCY = (1UL<<2),
  /** \brief 距离值类似于对象之间的带宽。
   * 对于更接近的对象，值较高，因此在矩阵的对角线上最大（对象与自身的距离）。
   * 目前对于基于距离的分组，此类值被忽略。
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH = (1UL<<3),

  /** \brief 此距离结构涵盖不同类型的对象。
   * 在存在 NVSwitch 或 POWER 处理器 NVLink 端口时，这可能适用于 "NVLinkBandwidth" 结构。
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES = (1UL<<4)
};
/**
 * 从拓扑结构中检索距离矩阵。
 *
 * 从拓扑结构中检索距离矩阵，并存储在 \p distances 数组中。
 *
 * \p flags 目前未使用，应该为 \c 0。
 *
 * \p kind 作为过滤器。如果为 \c 0，则返回所有距离矩阵。
 * 如果包含一些 HWLOC_DISTANCES_KIND_FROM_*，则只返回其种类与之匹配的距离矩阵。
 * 如果包含一些 HWLOC_DISTANCES_KIND_MEANS_*，则只返回其种类与之匹配的距离矩阵。
 *
 * 在输入时，\p nr 指向可能存储在 \p distances 中的距离矩阵的数量。
 * 在输出时，\p nr 指向实际找到的距离矩阵的数量，即使其中一些无法存储在 \p distances 中。
 * 无法存储的距离矩阵将被忽略，但函数仍然返回成功（\c 0）。
 * 调用者可以通过比较函数调用前后 \p nr 指向的值来找出这一点。
 *
 * 在 \p distances 数组中返回的每个距离矩阵应该由调用者使用 hwloc_distances_release() 释放。
 */
HWLOC_DECLSPEC int
hwloc_distances_get(hwloc_topology_t topology,
            unsigned *nr, struct hwloc_distances_s **distances,
            unsigned long kind, unsigned long flags);

/**
 * 在拓扑结构中检索特定深度的对象的距离矩阵。
 *
 * 与 hwloc_distances_get() 相同，额外增加了 \p depth 过滤器。
 */
HWLOC_DECLSPEC int
hwloc_distances_get_by_depth(hwloc_topology_t topology, int depth,
                 unsigned *nr, struct hwloc_distances_s **distances,
                 unsigned long kind, unsigned long flags);

/**
 * 检索特定类型对象的距离矩阵。
 *
 * 与 hwloc_distances_get() 相同，额外增加了 \p type 过滤器。
 */
HWLOC_DECLSPEC int
# 通过给定的类型和拓扑结构，获取距离矩阵
hwloc_distances_get_by_type(hwloc_topology_t topology, hwloc_obj_type_t type,
                unsigned *nr, struct hwloc_distances_s **distances,
                unsigned long kind, unsigned long flags);

/** \brief 通过给定的名称检索距离矩阵。
 *
 * 通常情况下，只有一个距离结构与给定的名称匹配。
 *
 * 最常见的结构名称是 "NUMALatency"。
 * 其他包括 "XGMIBandwidth"、"XGMIHops"、"XeLinkBandwidth" 和 "NVLinkBandwidth"。
 */
HWLOC_DECLSPEC int
hwloc_distances_get_by_name(hwloc_topology_t topology, const char *name,
                unsigned *nr, struct hwloc_distances_s **distances,
                unsigned long flags);

/** \brief 获取距离结构包含的描述信息。
 *
 * 例如，对于硬件提供的 NUMA 距离（ACPI SLIT），描述为 "NUMALatency"，如果未知则返回 NULL。
 */
HWLOC_DECLSPEC const char *
hwloc_distances_get_name(hwloc_topology_t topology, struct hwloc_distances_s *distances);

/** \brief 释放先前由 hwloc_distances_get() 返回的距离矩阵结构。
 *
 * \note 如果使用 hwloc_distances_release_remove() 删除结构，则不需要此函数。
 */
HWLOC_DECLSPEC void
hwloc_distances_release(hwloc_topology_t topology, struct hwloc_distances_s *distances);

/** \brief 距离结构的转换。 */
# 定义枚举类型 hwloc_distances_transform_e
enum hwloc_distances_transform_e {
  /** \brief 从距离结构中移除 \c NULL 对象。
   *
   * 在 \p objs 数组中被替换为 \c NULL 的每个对象都将被移除，并相应地更新 \p values 数组。
   *
   * 至少必须保留 \c 2 个对象，否则 hwloc_distances_transform() 将返回 \c -1，并将 \p errno 设置为 \c EINVAL。
   *
   * 根据剩余的对象，\p kind 将会被更新，或者根据 ::HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES 进行更新。
   *
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL = 0,

  /** \brief 用链接数量替换带宽值。
   *
   * 通常所有值都将是 \c 0（无链接）或 \c 1（一个链接）。
   * 但是，如果一些对等体之间通过不同数量的链接连接，则一些矩阵可能会获得较大的值。
   *
   * 对角线上的值设置为 \c 0。
   *
   * 此转换仅适用于带宽矩阵。
   *
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_LINKS = 1,

  /** \brief 将具有多个端口的交换机合并为单个对象。
   * 目前仅适用于 NVSwitch，其中 GPU 似乎连接到 NVLinkBandwidth 矩阵中的不同单独的交换机端口。此转换将用所有 GPU 连接的相同端口替换所有端口。
   * 其他端口通过内部应用 ::HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL 被移除。
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS = 2,

  /** \brief 对矩阵应用传递闭包以连接跨交换机的对象。
   * 目前仅适用于 NVLinkBandwidth 矩阵中的 GPU 和 NVSwitch。
   * 所有 GPU 对将被报告为直接连接。
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE = 3
};
/**
 * \brief 对距离结构应用转换
 *
 * 修改先前使用hwloc_distances_get()或其变体之一获取的距离结构。
 *
 * 这会修改距离结构的本地副本，但不会修改存储在拓扑结构内部的距离信息
 * （通过另一个调用hwloc_distances_get()检索或导出到XML）。
 * 要这样做，应该添加一个新的距离结构，具有相同的名称、类型、对象和值
 * （参见\ref hwlocality_distances_add），然后使用hwloc_distances_release_remove()删除旧的结构。
 *
 * \p transform必须是::hwloc_distances_transform_e中列出的转换之一。
 *
 * 这些转换可能会修改\p objs或\p values数组的内容。
 *
 * \p transform_attr现在必须为\c NULL。
 *
 * \p flags现在必须为\c 0。
 *
 * \note 距离数组\p objs中的对象可能直接就地修改，而无需使用hwloc_distances_transform()。
 * 可以使用hwloc_get_obj_with_same_locality()轻松地在不同类型的相似对象之间进行转换。
 */
HWLOC_DECLSPEC int hwloc_distances_transform(hwloc_topology_t topology, struct hwloc_distances_s *distances,
                                             enum hwloc_distances_transform_e transform,
                                             void *transform_attr,
                                             unsigned long flags);

/** @} */

/** \defgroup hwlocality_distances_consult 用于查询距离矩阵的辅助函数
 * @{
 */

/** \brief 在距离结构中查找对象的索引
 *
 * 如果对象\p obj未涉及结构\p distances，则返回-1。
 */
static __hwloc_inline int
hwloc_distances_obj_index(struct hwloc_distances_s *distances, hwloc_obj_t obj)
{
  unsigned i;
  for(i=0; i<distances->nbobjs; i++)
    if (distances->objs[i] == obj)
      return (int)i;
  return -1;
}
/** \brief Find the values between two objects in a distance matrices.
 *
 * The distance from \p obj1 to \p obj2 is stored in the value pointed by
 * \p value1to2 and reciprocally.
 *
 * \return -1 if object \p obj1 or \p obj2 is not involved in structure \p distances.
 */
static __hwloc_inline int
hwloc_distances_obj_pair_values(struct hwloc_distances_s *distances,
                hwloc_obj_t obj1, hwloc_obj_t obj2,
                hwloc_uint64_t *value1to2, hwloc_uint64_t *value2to1)
{
  // 获取对象 obj1 在 distances 中的索引
  int i1 = hwloc_distances_obj_index(distances, obj1);
  // 获取对象 obj2 在 distances 中的索引
  int i2 = hwloc_distances_obj_index(distances, obj2);
  // 如果 obj1 或 obj2 不在 distances 结构中，则返回 -1
  if (i1 < 0 || i2 < 0)
    return -1;
  // 将 obj1 到 obj2 的距离值存储在 value1to2 指向的位置
  *value1to2 = distances->values[i1 * distances->nbobjs + i2];
  // 将 obj2 到 obj1 的距离值存储在 value2to1 指向的位置
  *value2to1 = distances->values[i2 * distances->nbobjs + i1];
  // 返回 0 表示成功
  return 0;
}

/** @} */



/** \defgroup hwlocality_distances_add Add distances between objects
 *
 * The usual way to add distances is:
 * \code
 * hwloc_distances_add_handle_t handle;
 * int err = -1;
 * handle = hwloc_distances_add_create(topology, "name", kind, 0);
 * if (handle) {
 *   err = hwloc_distances_add_values(topology, handle, nbobjs, objs, values, 0);
 *   if (!err)
 *     err = hwloc_distances_add_commit(topology, handle, flags);
 * }
 * \endcode
 * If \p err is \c 0 at the end, then addition was successful.
 *
 * @{
 */

/** \brief Handle to a new distances structure during its addition to the topology. */
typedef void * hwloc_distances_add_handle_t;
# 创建一个新的空距离结构
#
# 创建一个空的距离结构，用于填充hwloc_distances_add_values()，然后使用hwloc_distances_add_commit()提交。
#
# 参数name是可选的，可以是NULL。否则，它将在内部被复制，稍后可以由调用者释放。
#
# kind参数指定距离的种类，作为::hwloc_distances_kind_e的OR'ed集合。
# 根据hwloc_distances_add_values()中具有不同类型的对象，将自动设置Kind ::HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES。
#
# flags现在必须为0。
#
# 返回一个hwloc_distances_add_handle_t，然后应将其传递给hwloc_distances_add_values()和hwloc_distances_add_commit()。
#
# 在错误时返回NULL。
HWLOC_DECLSPEC hwloc_distances_add_handle_t
hwloc_distances_add_create(hwloc_topology_t topology,
                           const char *name, unsigned long kind,
                           unsigned long flags);

# 指定新的空距离结构中的对象和值
#
# 指定由hwloc_distances_add_create()返回为句柄的新距离结构中的对象和值。
# 然后必须使用hwloc_distances_add_commit()提交该结构。
#
# 对象的数量是nbobjs，对象的数组是objs。
# 距离值存储为一维数组在values中。
# 从对象i到对象j的距离在槽i*nbobjs+j中。
#
# nbobjs必须至少为2。
#
# 数组objs和values将在内部被复制，稍后可以由调用者释放。
#
# 在错误时，临时距离结构及其内容将被销毁。
#
# flags现在必须为0。
#
# 成功时返回0。
# 错误时返回-1。
# 添加新的距离信息到拓扑结构中
HWLOC_DECLSPEC int hwloc_distances_add_values(hwloc_topology_t topology,
                                              hwloc_distances_add_handle_t handle,
                                              unsigned nbobjs, hwloc_obj_t *objs,
                                              hwloc_uint64_t *values,
                                              unsigned long flags);

/** \brief 用于向拓扑结构中添加新距离的标志 */
enum hwloc_distances_add_flag_e {
  /** \brief 尝试根据新提供的距离信息对对象进行分组。
   * 对于不同类型对象之间的距离，此标志将被忽略。
   * \hideinitializer
   */
  HWLOC_DISTANCES_ADD_FLAG_GROUP = (1UL<<0),
  /** \brief 如果进行分组，将距离值视为不准确，并在分组算法中放宽比较。
   * 实际的准确性可以通过 HWLOC_GROUPING_ACCURACY 环境变量进行修改（参见 \ref envvar）。
   * \hideinitializer
   */
  HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE = (1UL<<1)
};

/** \brief 提交新的距离结构。
 *
 * 此函数完成距离结构，并将其插入拓扑结构中。
 *
 * 参数 \p handle 之前由 hwloc_distances_add_create() 返回。
 * 然后使用 hwloc_distances_add_values() 指定对象和值。
 *
 * \p flags 使用可选的 OR 运算设置 ::hwloc_distances_add_flag_e 配置函数的行为。
 * 可以用于请求基于距离对现有对象进行分组。
 *
 * 出现错误时，临时距离结构及其内容将被销毁。
 *
 * \return \c 0 表示成功。
 * \return \c -1 表示错误。
 */
HWLOC_DECLSPEC int hwloc_distances_add_commit(hwloc_topology_t topology,
                                              hwloc_distances_add_handle_t handle,
                                              unsigned long flags);

/** @} */



/** \defgroup hwlocality_distances_remove 移除对象之间的距离
 * @{
 */
/** \brief 从拓扑结构中移除所有距离矩阵。
 *
 * 移除所有距离矩阵，无论是用户提供的还是通过操作系统收集的。
 *
 * 如果这些距离被用于对对象进行分组，那么这些额外的 Group 对象不会从拓扑结构中移除。
 */
HWLOC_DECLSPEC int hwloc_distances_remove(hwloc_topology_t topology);

/** \brief 从拓扑结构中移除特定深度的对象的距离矩阵。
 *
 * 与 hwloc_distances_remove() 相同，但仅适用于拓扑结构中的一个层级。
 */
HWLOC_DECLSPEC int hwloc_distances_remove_by_depth(hwloc_topology_t topology, int depth);

/** \brief 从拓扑结构中移除特定类型的对象的距离矩阵。
 *
 * 与 hwloc_distances_remove() 相同，但仅适用于拓扑结构中的一个层级。
 */
static __hwloc_inline int
hwloc_distances_remove_by_type(hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return 0;
  return hwloc_distances_remove_by_depth(topology, depth);
}

/** \brief 释放并从拓扑结构中移除给定的距离矩阵。
 *
 * 此函数包括对 hwloc_distances_release() 的调用。
 */
HWLOC_DECLSPEC int hwloc_distances_release_remove(hwloc_topology_t topology, struct hwloc_distances_s *distances);

/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_DISTANCES_H */
```