# `xmrig\src\3rdparty\hwloc\include\hwloc\diff.h`

```cpp
/*
 * 版权所有 © 2013-2020 Inria。
 * 请查看顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 拓扑差异。
 */

#ifndef HWLOC_DIFF_H
#define HWLOC_DIFF_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h
#endif


#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_diff 拓扑差异
 *
 * 操作许多类似的拓扑的应用程序，例如一个均匀集群中的每个节点，可能希望压缩拓扑以减少内存占用。
 *
 * 该文件提供了一种处理拓扑之间差异的方法，并将其导出/导入到XML。
 * 可以通过仅使用与前者的差异来存储一个完整的拓扑，而其他拓扑仅通过其与前者的差异来描述。
 * 当实际需要时，可以通过将预先计算的差异应用于参考拓扑来重建实际拓扑。
 *
 * 该接口针对非常相似的节点。
 * 实际上只支持拓扑之间非常简单的差异，例如内存大小的变化，对象的名称或一些信息属性。
 * 无法表示更复杂的差异，例如添加或删除对象，因此会返回错误。
 * 也无法表示对象集或整个拓扑的差异。
 *
 * 这意味着在查看树结构（多少级别，每个级别多少对象，对象的类型，CPU和节点集等）和绑定到对象时，不需要应用差异。
 * 但是，在查看对象属性（例如名称，内存大小或信息属性）时，必须应用差异。
 *
 * @{
 */


/** \brief 一个对象属性差异的类型。
 */
# 定义枚举类型 hwloc_topology_diff_obj_attr_type_e，表示拓扑结构对象属性的不同类型
typedef enum hwloc_topology_diff_obj_attr_type_e {
  # 对象的本地内存被修改，联合体是 hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_uint64_s（索引字段被忽略）
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE,

  # 对象的名称被修改，联合体是 hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_string_s（名称字段被忽略）
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME,

  # 信息属性的值被修改，联合体是 hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_string_s
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO
} hwloc_topology_diff_obj_attr_type_t;

# 定义拓扑结构对象属性的联合体，用于表示对象属性的不同类型
union hwloc_topology_diff_obj_attr_u {
  # 通用对象属性差异
  struct hwloc_topology_diff_obj_attr_generic_s {
    # 联合体的每个部分都必须以这些字段开始
    hwloc_topology_diff_obj_attr_type_t type;
  } generic;

  # 整数属性修改，可选索引
  struct hwloc_topology_diff_obj_attr_uint64_s {
    # 用于存储整数属性
    hwloc_topology_diff_obj_attr_type_t type;
    hwloc_uint64_t index; # 对于 SIZE 不使用索引
    hwloc_uint64_t oldvalue;
    hwloc_uint64_t newvalue;
  } uint64;

  # 字符串属性修改，可选名称
  struct hwloc_topology_diff_obj_attr_string_s {
    # 用于存储名称和信息对
    hwloc_topology_diff_obj_attr_type_t type;
    char *name; # 对于 NAME 不使用名称
    char *oldvalue;
    char *newvalue;
  } string;
};

# 定义差异列表中一个元素的类型
typedef enum hwloc_topology_diff_type_e {
  /** \brief An object attribute was changed.
   * The union is a hwloc_topology_diff_u::hwloc_topology_diff_obj_attr_s.
   */
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR,

  /** \brief The difference is too complex,
   * it cannot be represented. The difference below
   * this object has not been checked.
   * hwloc_topology_diff_build() will return 1.
   *
   * The union is a hwloc_topology_diff_u::hwloc_topology_diff_too_complex_s.
   */
  HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX
} hwloc_topology_diff_type_t;

/** \brief One element of a difference list between two topologies.
 */
typedef union hwloc_topology_diff_u {
  struct hwloc_topology_diff_generic_s {
    /* each part of the union must start with these */
    hwloc_topology_diff_type_t type;
    union hwloc_topology_diff_u * next; /* pointer to the next element of the list, or NULL */
  } generic;

  /* A difference in an object attribute. */
  struct hwloc_topology_diff_obj_attr_s {
    hwloc_topology_diff_type_t type; /* must be ::HWLOC_TOPOLOGY_DIFF_OBJ_ATTR */
    union hwloc_topology_diff_u * next;
    /* List of attribute differences for a single object */
    int obj_depth;
    unsigned obj_index;
    union hwloc_topology_diff_obj_attr_u diff;
  } obj_attr;

  /* A difference that is too complex. */
  struct hwloc_topology_diff_too_complex_s {
    hwloc_topology_diff_type_t type; /* must be ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX */
    union hwloc_topology_diff_u * next;
    /* Where we had to stop computing the diff in the first topology */
    int obj_depth;
    unsigned obj_index;
  } too_complex;
} * hwloc_topology_diff_t;
/**
 * \brief 计算两个拓扑结构之间的差异。
 *
 * 差异以 ::hwloc_topology_diff_t 条目的列表形式存储在 \p diff 中。
 * 通过同时对两个拓扑树进行深度优先遍历来计算差异。
 *
 * 如果两个对象之间的差异太复杂无法表示（例如，某些对象具有不同类型或不同数量的子对象），则会排队一个特殊的差异条目，类型为 ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX。
 * 差异的计算不会继续到这些对象以下。因此，每个这样的差异条目表示两个子树之间的差异无法计算。
 *
 * \return 如果差异可以正确表示，则返回 0。
 *
 * \return 如果拓扑结构之间没有差异，则返回 0，\p diff 指向 NULL。
 *
 * \return 如果差异太复杂（参见上文），则返回 1。列表中的某些条目将是 ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX 类型。
 *
 * \return 在任何其他错误情况下返回 -1。
 *
 * \note \p flags 目前未使用，应为 0。
 *
 * \note 输出的差异必须使用 hwloc_topology_diff_destroy() 进行释放。
 *
 * \note 只有在返回 0 时（即列表中没有 ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX 类型的条目时），输出的差异才能导出到 XML 或传递给 hwloc_topology_diff_apply()。
 *
 * \note 输出的差异可能会被修改，通过从列表中移除一些条目。应通过将这些条目传递给 hwloc_topology_diff_destroy()（可能作为另一个列表）来释放已移除的条目。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_build(hwloc_topology_t topology, hwloc_topology_t newtopology, unsigned long flags, hwloc_topology_diff_t *diff);

/**
 * \brief 传递给 hwloc_topology_diff_apply() 的标志。
 */
enum hwloc_topology_diff_apply_flags_e {
  /** \brief 在相反方向应用拓扑差异。
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE = (1UL<<0)
};
/**
 * \brief 将拓扑差异应用于现有拓扑。
 *
 * \p flags 是 ::hwloc_topology_diff_apply_flags_e 的 OR 运算集合。
 *
 * 新拓扑将直接修改。可以使用 hwloc_topology_dup() 在修补之前进行复制。
 *
 * 如果差异无法完全应用，则在返回之前将取消应用所有先前应用的元素。
 *
 * \return 成功返回 0。
 *
 * \return 如果在尝试应用第 N 部分差异时应用差异失败，则返回 -N。例如，如果无法应用第一个差异元素，则返回 -1。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_apply(hwloc_topology_t topology, hwloc_topology_diff_t diff, unsigned long flags);

/**
 * \brief 销毁拓扑差异列表。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_destroy(hwloc_topology_diff_t diff);

/**
 * \brief 从 XML 文件加载拓扑差异列表。
 *
 * 如果不是 \c NULL，则 \p refname 将填充差异文件的参考拓扑的标识符字符串，如果在 XML 文件中指定了的话。
 * 此标识符通常是包含参考拓扑的其他 XML 文件的名称。
 *
 * \note refname 中返回的指针应该由调用者稍后释放。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_load_xml(const char *xmlpath, hwloc_topology_diff_t *diff, char **refname);

/**
 * \brief 将拓扑差异列表导出到 XML 文件。
 *
 * 如果不是 \c NULL，则 \p refname 定义了在计算此差异时作为基础使用的参考拓扑的标识符字符串。
 * 此标识符通常是包含参考拓扑的其他 XML 文件的名称。
 * 从 XML 中读取差异时会返回此属性。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_export_xml(hwloc_topology_diff_t diff, const char *refname, const char *xmlpath);
/**
 * \brief 从 XML 缓冲区加载拓扑差异列表。
 *
 * 如果 \c refname 不是 \c NULL，则 \p refname 将被填充为 XML 文件中指定的参考拓扑的标识符字符串。
 * 如果在 XML 文件中指定了参考拓扑，则该标识符通常是包含参考拓扑的另一个 XML 文件的名称。
 *
 * \note refname 中返回的指针应该由调用者在后续进行释放。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_load_xmlbuffer(const char *xmlbuffer, int buflen, hwloc_topology_diff_t *diff, char **refname);

/**
 * \brief 将拓扑差异列表导出到 XML 缓冲区。
 *
 * 如果 \c refname 不是 \c NULL，则 \p refname 定义了用作基础的参考拓扑的标识符字符串。
 * 当计算此差异时，通常是指包含参考拓扑的另一个 XML 文件的名称。
 * 在从 XML 中读取差异时，将返回此属性。
 *
 * 返回的缓冲区以包含在返回长度中的 \0 结尾。
 *
 * \note XML 缓冲区应该在后续使用 hwloc_free_xmlbuffer() 进行释放。
 */
HWLOC_DECLSPEC int hwloc_topology_diff_export_xmlbuffer(hwloc_topology_diff_t diff, const char *refname, char **xmlbuffer, int *buflen);

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_DIFF_H */
```