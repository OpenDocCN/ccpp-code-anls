# `xmrig\src\3rdparty\hwloc\include\hwloc\export.h`

```
/*
 * 版权声明，版权归 Inria 公司所有，保留所有权利
 * 版权归 Université Bordeaux 大学所有
 * 版权归 Cisco Systems, Inc. 公司所有，保留所有权利
 * 请查看顶层目录下的 COPYING 文件
 */

/** \file
 * \brief 将拓扑导出为 XML 或合成字符串
 */

#ifndef HWLOC_EXPORT_H
#define HWLOC_EXPORT_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h 文件
#endif


#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_xmlexport Exporting Topologies to XML
 * @{
 */

/** \brief 导出 XML 拓扑的标志位
 *
 * 作为 OR 运算的一部分传递给 hwloc_topology_export_xml() 函数
 */
enum hwloc_topology_export_xml_flags_e {
 /** \brief 导出可被 hwloc v1.x 加载的 XML
  * 但是，导出可能会丢失有关拓扑的一些细节
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 = (1UL<<0)
};
/**
 * \brief 将拓扑结构导出为 XML 文件。
 *
 * 可以通过 hwloc_topology_set_xml() 后续加载此文件。
 *
 * 默认情况下，使用最新的导出格式，这意味着旧版本的 hwloc
 * （例如 v1.x）将无法导入它。
 * 使用标志 ::HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 可以将导出格式
 * 为 v1.x 特定的 XML 格式，但可能会丢失有关拓扑的一些细节。
 * 如果有任何可能导出的文件可能被使用 hwloc 1.x 的进程导入，
 * 应该考虑在运行时检测并使用相应的导出格式。
 *
 * \p flags 是 ::hwloc_topology_export_xml_flags_e 的 OR'ed 集合。
 *
 * \return 如果发生失败，则返回 -1。
 *
 * \note 参见 hwloc_topology_set_userdata_export_callback()
 * 用于导出特定于应用程序的对象用户数据。
 *
 * \note 导出到 XML 时，拓扑特定的用户数据指针将被忽略。
 *
 * \note 只有可打印字符可以导出为 XML 字符串属性。
 * 任何其他字符，特别是任何非 ASCII 字符，都将被静默丢弃。
 *
 * \note 如果 \p name 为“-”，则将 XML 输出发送到标准输出。
 */
HWLOC_DECLSPEC int hwloc_topology_export_xml(hwloc_topology_t topology, const char *xmlpath, unsigned long flags);
# 将拓扑结构导出到新分配的 XML 内存缓冲区
#
# xmlbuffer 由调用者分配，并应在调用者中稍后使用 hwloc_free_xmlbuffer() 释放
# 此内存缓冲区可以稍后通过 hwloc_topology_set_xmlbuffer() 加载
#
# 默认情况下，使用最新的导出格式，这意味着旧的 hwloc 发行版（例如 v1.x）将无法导入它
# 使用标志 ::HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 可以导出到 v1.x 特定的 XML 格式，但可能会丢失有关拓扑结构的一些细节
# 如果有任何可能导出的缓冲区可能会被使用 hwloc 1.x 的进程导入，应考虑在运行时检测并使用相应的导出格式
#
# 返回的缓冲区以 \0 结尾，该 \0 包含在返回的长度中
#
# flags 是 ::hwloc_topology_export_xml_flags_e 的 OR 运算集
#
# 如果发生失败，则返回 -1
#
# 请参阅 hwloc_topology_set_userdata_export_callback() 以导出特定于应用程序的对象用户数据
#
# 导出到 XML 时，拓扑结构特定的用户数据指针将被忽略
#
# 只有可打印字符可以导出到 XML 字符串属性
# 任何其他字符，特别是任何非 ASCII 字符，将被静默丢弃
#
HWLOC_DECLSPEC int hwloc_topology_export_xmlbuffer(hwloc_topology_t topology, char **xmlbuffer, int *buflen, unsigned long flags);

# 释放由 hwloc_topology_export_xmlbuffer() 分配的缓冲区
HWLOC_DECLSPEC void hwloc_free_xmlbuffer(hwloc_topology_t topology, char *xmlbuffer);
/**
 * \brief 设置用于导出对象用户数据的应用程序特定回调函数
 *
 * 默认情况下，hwloc 不导出对象用户数据指针到 XML，因为它不知道它包含什么内容。
 *
 * 此函数允许应用程序将 \p export_cb 设置为回调函数，将这个不透明的用户数据转换为可导出的字符串。
 *
 * 在 XML 导出期间，对每个具有 \p userdata 指针不为 \c NULL 的对象调用 \p export_cb。
 * 回调应该使用 hwloc_export_obj_userdata() 或 hwloc_export_obj_userdata_base64() 来实际导出
 * 一些东西到 XML（可能每个对象多次）。
 *
 * 如果不希望将用户数据导出到 XML，则可以将 \p export_cb 设置为 \c NULL。
 *
 * \note 在导出到 XML 时，拓扑特定的用户数据指针会被忽略。
 */
HWLOC_DECLSPEC void hwloc_topology_set_userdata_export_callback(hwloc_topology_t topology,
                                void (*export_cb)(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj));
# 将一些对象的用户数据导出到 XML
#
# 只能在传递给 hwloc_topology_set_userdata_export_callback() 的 export() 回调内部调用此函数。
# 可能会被调用多次以将一些用户数据导出到 XML。
# 长度为 length 的缓冲区内容存储在可选的名称 name 下。
#
# 在导入此 XML 文件时，如果设置了 import() 回调，它将被调用的次数与在 export() 中调用 hwloc_export_obj_userdata() 的次数完全相同。
# 它将接收相应的 name、buffer 和 length 参数。
#
# reserved、topology 和 obj 必须是传递给导出回调的前三个参数。
#
# 只能导出可打印字符到 XML 字符串属性。
# 如果在 name 或 buffer 中传递了不可打印的字符，则该函数返回 -1，并将 errno 设置为 EINVAL。
#
# 如果要导出二进制数据，应用程序应首先编码为仅包含可打印字符（或使用 hwloc_export_obj_userdata_base64()）。
# 它还应注意可移植性问题，如果导出可能在不同体系结构上重新导入。
HWLOC_DECLSPEC int hwloc_export_obj_userdata(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);

# 编码并将一些对象的用户数据导出到 XML
#
# 此函数类似于 hwloc_export_obj_userdata()，但它在导出之前将输入缓冲区编码为可打印字符。
# 在导入时，如果设置了 import() 回调，则会自动执行解码，然后将数据传递给 import() 回调（如果有）。
#
# 只能在传递给 hwloc_topology_set_userdata_export_callback() 的 export() 回调内部调用此函数。
#
# 如果导出可能在不同体系结构上重新导入，则此函数不会处理可移植性问题。
# 导出对象的用户数据到 base64 编码的字符串
HWLOC_DECLSPEC int hwloc_export_obj_userdata_base64(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);

/** \brief 设置导入用户数据的应用程序特定回调函数
 *
 * 在 XML 导入时，默认情况下会忽略用户数据，因为 hwloc 不知道如何将其存储在内存中。
 *
 * 此函数允许应用程序将 \p import_cb 设置为回调函数，该函数将获取 XML 存储的用户数据，并将其存储在对象中，以便应用程序使用。
 *
 * 在 hwloc_topology_load() 期间，\p import_cb 会被调用多次，就像在导出期间调用了多次 hwloc_export_obj_userdata() 一样。拓扑结构还没有完全设置好。对象属性已准备好供查询，但对象之间的链接还没有建立。
 *
 * 如果在导入期间应该忽略用户数据，则 \p import_cb 可以是 \c NULL。
 *
 * \note \p buffer 包含 \p length 个字符，后面跟着一个空字符 ('\0')。
 *
 * \note 在调用 hwloc_topology_load() 之前应调用此函数。
 *
 * \note 从 XML 导入时，特定于拓扑结构的用户数据指针将被忽略。
 */
HWLOC_DECLSPEC void hwloc_topology_set_userdata_import_callback(hwloc_topology_t topology,
                                void (*import_cb)(hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length));

/** @} */


/** \defgroup hwlocality_syntheticexport Exporting Topologies to Synthetic
 * @{
 */

/** \brief 用于导出合成拓扑的标志
 *
 * 作为 OR 运算集合传递给 hwloc_topology_export_synthetic()。
 */
# 定义枚举类型 hwloc_topology_export_synthetic_flags_e
enum hwloc_topology_export_synthetic_flags_e {
    # 导出扩展类型，如 L2dcache 作为基本类型，如 Cache
    # 如果使用 hwloc < 1.9 加载合成描述，则需要此选项
    # 隐藏初始化
    HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES = (1UL<<0),

    # 不导出级别属性
    # 忽略级别属性，如内存/缓存大小或 PU 索引
    # 如果使用 hwloc < 1.10 加载合成描述，则需要此选项
    # 隐藏初始化
    HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS = (1UL<<1),

    # 导出内存层次结构，如在 hwloc 1.x 中预期的那样
    # 在可能的情况下，将单个 NUMA 节点子级别导出为正常的中间级别
    # 如果使用 hwloc 1.x 加载合成描述，则需要此选项
    # 但是，如果某些对象具有多个本地 NUMA 节点，则可能会失败
    # 隐藏初始化
    HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1 = (1UL<<2),

    # 不导出内存信息
    # 仅导出正常 CPU 端对象的实际层次结构，并忽略内存的附加位置
    # 这对于 CPU 层次结构才是真正重要的情况很有用
    # 但它的行为就像有一个单一的机器范围内的 NUMA 节点
    # 隐藏初始化
    HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY = (1UL<<3)
};
/**
 * \brief 将拓扑结构导出为合成字符串。
 *
 * 最多将在 \p buffer 中写入 \p buflen 个字符，包括终止的 \0。
 *
 * 此导出的字符串可以传递给 hwloc_topology_set_synthetic()。
 *
 * \p flags 是 ::hwloc_topology_export_synthetic_flags_e 的 OR 运算集合。
 *
 * \return 写入的字符数，不包括终止的 \0。
 *
 * \return 如果无法导出拓扑结构，例如如果它不对称，则返回 -1。
 *
 * \note 忽略 I/O 和 Misc 子节点，合成字符串只描述正常子节点。
 *
 * \note 1024 字节的缓冲区应该足够大，以导出绝大多数情况下的拓扑结构。
 */
  HWLOC_DECLSPEC int hwloc_topology_export_synthetic(hwloc_topology_t topology, char *buffer, size_t buflen, unsigned long flags);

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_EXPORT_H */
```