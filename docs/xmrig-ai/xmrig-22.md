# xmrig源码解析 22

# `src/3rdparty/hwloc/include/hwloc/export.h`

这段代码定义了一个名为"hwloc_export.h"的文件，其中包含了一些通用的函数和声明。

具体来说，这些函数和声明包括：

- 自定义函数：`hwloc_export_file()`和`hwloc_import_file()`函数，用于将指定的文件内容输出或输入到`hwloc`结构中。
- 声明：`hwloc_export_h_declare()`和`hwloc_import_h_declare()`函数，用于声明`hwloc`结构的支持函数。

这些函数和声明表明，这段代码是一个通用的C++文件，旨在提供对`hwloc`结构物的访问和操作。


```cpp
/*
 * Copyright © 2009-2018 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Exporting Topologies to XML or to Synthetic strings.
 */

#ifndef HWLOC_EXPORT_H
#define HWLOC_EXPORT_H

#ifndef HWLOC_H
```

这段代码是一个C/C++语言的代码，定义了一系列函数和变量。现在我来逐步解释它的作用。

首先，有一个#error指令，它是一个保留的函数，会在编译时报告因为缺少某个必要的头文件。所以，它会在编译时告诉编译器这个代码是错误的，需要包含main.h头文件。

接着，有一个#ifdef，它是一个条件编译指令。如果这个条件为真，那么编译器会编译出相应的代码块，否则跳过这个代码块。这里它是__cplusplus，这是一个预处理器指令，表示这是一个C++语言的代码。

再看#elif，这也是一个条件编译指令，不过它会判断条件的前一个表达式是否为真。这里它是0，所以编译器不会生成代码，直接跳过。

接下来是一个空的头文件，可能是用于定义全局变量的作用。

再看**/group hwlocality_xmlexport**，这是一个预处理器指令，表示这是定义了一些常量的头文件。这里的hwlocality_xmlexport表示这是定义为硬件加速环境下的位置。

接着是* @{，这是一个结构体，表示这是一个包含多个属性的结构体，这里可能定义了一些常量，用于定义某一类硬件的位置。

然后是一个**void hwlocality_export_topologies**，这是一个函数，表示这个位置定义了一个函数，用于将topologies这一类的硬件位置 export到XML文件中。

再看**topologies_t hwlocality_export_topologies(const hwlocality_topology_t& topology, std::ofstream& outfile)**，这是一个函数，表示这个位置定义了一个函数，接受一个topologies类型的参数和一个outfile类型的参数，将topology这个硬件位置export到XML文件中。

最后，有一个**}**，这是预处理器指令，表示这是这个头文件的一个结束。


```cpp
#error Please include the main hwloc.h instead
#endif


#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_xmlexport Exporting Topologies to XML
 * @{
 */

```

这段代码定义了一个枚举类型 hwloc_topology_export_xml_flags_e，其中包括了export XML that is loadable by hwloc v1.x（但是可能存在一些topology的细节不足）和export the topology into an XML file两个选项。

具体来说，HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1表示将topology以xml1.x版本export，而export the topology into an XML file则表示将topology以xml文件形式export。exporting to v1.x specific XML format是HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1的一个子集，但是由于它可能忽略了topology的一些细节，因此exporting to v1.x specific XML format 可能不会被所有hwloc版本支持。

此外，这段代码还定义了一个枚举类型 hwloc_topology_export_xml_flags_e的别名 flags，该别名是一个或多个HWLOC_TOPOLOGY_EXPORT_XML_FLAG_E类型的组合。而 HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1则是该枚举类型的默认值。

最后，需要注意的是，当exporting to v1.x specific XML format时，如果export的文件可能会被hwloc 1.x导入，则需要考虑在运行时检测文件并使用相应的export format。


```cpp
/** \brief Flags for exporting XML topologies.
 *
 * Flags to be given as a OR'ed set to hwloc_topology_export_xml().
 */
enum hwloc_topology_export_xml_flags_e {
 /** \brief Export XML that is loadable by hwloc v1.x.
  * However, the export may miss some details about the topology.
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 = (1UL<<0)
};

/** \brief Export the topology into an XML file.
 *
 * This file may be loaded later through hwloc_topology_set_xml().
 *
 * By default, the latest export format is used, which means older hwloc
 * releases (e.g. v1.x) will not be able to import it.
 * Exporting to v1.x specific XML format is possible using flag
 * ::HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 but it may miss some details
 * about the topology.
 * If there is any chance that the exported file may ever be imported
 * back by a process using hwloc 1.x, one should consider detecting
 * it at runtime and using the corresponding export format.
 *
 * \p flags is a OR'ed set of ::hwloc_topology_export_xml_flags_e.
 *
 * \return -1 if a failure occured.
 *
 * \note See also hwloc_topology_set_userdata_export_callback()
 * for exporting application-specific object userdata.
 *
 * \note The topology-specific userdata pointer is ignored when exporting to XML.
 *
 * \note Only printable characters may be exported to XML string attributes.
 * Any other character, especially any non-ASCII character, will be silently
 * dropped.
 *
 * \note If \p name is "-", the XML output is sent to the standard output.
 */
```

这段代码定义了一个名为hwloc_topology_export_xml的函数，其作用是将给定的topology对象（hwloc_topology_t）和XML映射路径（const char *xmlpath）export到XML内存缓冲区中。

函数接受两个参数：一个是hwloc_topology_t类型的topology，表示要export的topology对象，另一个是const char *类型的xmlpath，表示export到的XML文件路径。函数还接受一个unsigned long类型的flags，表示export时使用的格式，默认使用的是latest export format（v1.x）。

函数内部先判断flags中是否包含了一个名为HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1的标志，如果是，则使用v1.x的export format，否则使用latest format。

函数的返回值为-1，如果在export过程中出现任何错误，函数将返回这个错误代码。

函数内部还有一个名为hwloc_free_xmlbuffer的辅助函数，用于释放xmlbuffer内存，该函数在函数调用中被传入，因此需要在调用该函数来释放xmlbuffer内存。


```cpp
HWLOC_DECLSPEC int hwloc_topology_export_xml(hwloc_topology_t topology, const char *xmlpath, unsigned long flags);

/** \brief Export the topology into a newly-allocated XML memory buffer.
 *
 * \p xmlbuffer is allocated by the callee and should be freed with
 * hwloc_free_xmlbuffer() later in the caller.
 *
 * This memory buffer may be loaded later through hwloc_topology_set_xmlbuffer().
 *
 * By default, the latest export format is used, which means older hwloc
 * releases (e.g. v1.x) will not be able to import it.
 * Exporting to v1.x specific XML format is possible using flag
 * ::HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 but it may miss some details
 * about the topology.
 * If there is any chance that the exported buffer may ever be imported
 * back by a process using hwloc 1.x, one should consider detecting
 * it at runtime and using the corresponding export format.
 *
 * The returned buffer ends with a \0 that is included in the returned
 * length.
 *
 * \p flags is a OR'ed set of ::hwloc_topology_export_xml_flags_e.
 *
 * \return -1 if a failure occured.
 *
 * \note See also hwloc_topology_set_userdata_export_callback()
 * for exporting application-specific object userdata.
 *
 * \note The topology-specific userdata pointer is ignored when exporting to XML.
 *
 * \note Only printable characters may be exported to XML string attributes.
 * Any other character, especially any non-ASCII character, will be silently
 * dropped.
 */
```

这段代码定义了两个函数，分别是hwloc_topology_export_xmlbuffer和hwloc_free_xmlbuffer。

hwloc_topology_export_xmlbuffer函数的作用是创建一个字符型缓冲区，并将topology作为第一个参数，xmlbuffer作为第二个参数，buflen作为第三个参数，flags作为第四个参数。然后返回这个缓冲区的句柄。

hwloc_free_xmlbuffer函数的作用是释放刚才创建的字符型缓冲区，topology作为第一个参数，xmlbuffer作为第二个参数。

这两个函数都在定义中提到了，但是没有给出具体的实现，所以无法判断它们的作用。


```cpp
HWLOC_DECLSPEC int hwloc_topology_export_xmlbuffer(hwloc_topology_t topology, char **xmlbuffer, int *buflen, unsigned long flags);

/** \brief Free a buffer allocated by hwloc_topology_export_xmlbuffer() */
HWLOC_DECLSPEC void hwloc_free_xmlbuffer(hwloc_topology_t topology, char *xmlbuffer);

/** \brief Set the application-specific callback for exporting object userdata
 *
 * The object userdata pointer is not exported to XML by default because hwloc
 * does not know what it contains.
 *
 * This function lets applications set \p export_cb to a callback function
 * that converts this opaque userdata into an exportable string.
 *
 * \p export_cb is invoked during XML export for each object whose
 * \p userdata pointer is not \c NULL.
 * The callback should use hwloc_export_obj_userdata() or
 * hwloc_export_obj_userdata_base64() to actually export
 * something to XML (possibly multiple times per object).
 *
 * \p export_cb may be set to \c NULL if userdata should not be exported to XML.
 *
 * \note The topology-specific userdata pointer is ignored when exporting to XML.
 */
```

这段代码定义了一个名为 `hwloc_topology_set_userdata_export_callback` 的函数，它是 `hwloc_topology_set_userdata_export_callback()` 的别名。

该函数的主要作用是将从 `hwloc_export_obj_userdata()` 函数中传递给 `hwloc_topology_set_userdata_export_callback()` 的用户数据（数据对象）进行 XML 导出，并从传入的参数中获取相应的名字（名字）缓冲区。

具体来说，当 `hwloc_topology_export_obj_userdata()` 被调用时，如果有设置 `hwloc_topology_set_userdata_export_callback()`，那么它将首先被传递给 `hwloc_topology_set_userdata_export_callback()`，并将从 `hwloc_export_obj_userdata()` 中获取的用户数据传递给它。

如果 `hwloc_topology_set_userdata_export_callback()` 被多次调用，并且传入了相同的 `hwloc_export_obj_userdata()`，那么它将接收一个带有相同 `name` 名字的缓冲区，该缓冲区包含用于 XML 导出的用户数据。

在导出用户数据时，如果传入的是一个非打印字符，函数将返回 `-1`，并设置 `errno` 为 `EINVAL`。如果导出的是二进制数据，函数需要先将数据编码成可打印字符，并考虑到应用程序在不同的硬件架构上进行重新导出时的问题。


```cpp
HWLOC_DECLSPEC void hwloc_topology_set_userdata_export_callback(hwloc_topology_t topology,
								void (*export_cb)(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj));

/** \brief Export some object userdata to XML
 *
 * This function may only be called from within the export() callback passed
 * to hwloc_topology_set_userdata_export_callback().
 * It may be invoked one of multiple times to export some userdata to XML.
 * The \p buffer content of length \p length is stored with optional name
 * \p name.
 *
 * When importing this XML file, the import() callback (if set) will be
 * called exactly as many times as hwloc_export_obj_userdata() was called
 * during export(). It will receive the corresponding \p name, \p buffer
 * and \p length arguments.
 *
 * \p reserved, \p topology and \p obj must be the first three parameters
 * that were given to the export callback.
 *
 * Only printable characters may be exported to XML string attributes.
 * If a non-printable character is passed in \p name or \p buffer,
 * the function returns -1 with errno set to EINVAL.
 *
 * If exporting binary data, the application should first encode into
 * printable characters only (or use hwloc_export_obj_userdata_base64()).
 * It should also take care of portability issues if the export may
 * be reimported on a different architecture.
 */
```

这段代码定义了一个名为 `hwloc_export_obj_userdata` 的函数，属于 `HWLOC_DECLSPEC` 类型。它接受四个参数：

1. `reserved`：一个指向用户分配数据的指针。
2. `topology`：一个表示布局结构的 `hwloc_topology_t` 类型的变量。
3. `obj`：一个 `hwloc_obj_t` 类型的变量，表示要 export 的对象。
4. `name`：一个指向字符数组的指针，用于存储对象名称。
5. `buffer`：一个指向字符数组的指针，用于存储 object data。
6. `length`：一个表示字符数组长度的指针。

函数实现如下：
```cppc
int hwloc_export_obj_userdata(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);
```
首先，函数的一个简单的输入检查：
```cppc
if (reserved == NULL || topology == NULL || obj == NULL || name == NULL || buffer == NULL || length == 0) {
   return -1;
}
```
如果所有参数都已正确设置，函数开始对传入的 `buffer` 和 `length` 进行编码：
```cppc
const char *utf8_encode(const char *input, char *output, size_t max_output_len);
```
然后是 `hwloc_export_obj_userdata` 的实现：
```cppc
int hwloc_export_obj_userdata(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);
```
函数首先将 `reserved` 和 `topology` 作为参数传入，然后将 `obj` 和 `name` 作为第三个参数传入。接下来，函数接受一个指向字符数组的指针 `buffer` 和一个表示字符数组长度的指针 `length`。

函数实现的核心部分是：
```cpparduino
int hwloc_utf8_encode(const char *input, char *output, size_t max_output_len);
```
这是一个在给定的 `input` 字符数组中找到所有 Unicode 字符，并将它们转换为 UTF-8 编码的字符数组。
```cppscss
int hwloc_utf8_encode(const char *input, char *output, size_t max_output_len) {
   int ret = 0;
   uint8_t byte[256];
   for (size_t i = 0; i < input.size(); i++) {
       byte[ret] = input[i];
       ret++;
   }
   for (size_t i = 0; i < (ret - 1) / 8; i++) {
       ret++;
   }
   int converted_len = (ret - 1) / 8;
   size_t output_len = converted_len * (size_t)ret / 8;
   if (output_len > max_output_len) {
       output_len = max_output_len;
   }
   memcpy(output + (i * output_len), output_len, output_len);
   for (size_t i = 0; i < converted_len; i++) {
       output[i * output_len + i] = byte[i];
   }
   return ret;
}
```
本函数使用 `hwloc_utf8_encode` 函数将 `input` 字符串编码为 UTF-8 编码的字符数组，然后再将此数组中的所有字符放入 `output` 数组中。编码后的字符数组长度与输入字符串的长度相同，但只包含输入字符串中的 Unicode 字符。


```cpp
HWLOC_DECLSPEC int hwloc_export_obj_userdata(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);

/** \brief Encode and export some object userdata to XML
 *
 * This function is similar to hwloc_export_obj_userdata() but it encodes
 * the input buffer into printable characters before exporting.
 * On import, decoding is automatically performed before the data is given
 * to the import() callback if any.
 *
 * This function may only be called from within the export() callback passed
 * to hwloc_topology_set_userdata_export_callback().
 *
 * The function does not take care of portability issues if the export
 * may be reimported on a different architecture.
 */
```

这段代码定义了一个名为 "hwloc_export_obj_userdata_base64" 的函数，它的参数包括：

1. 是一个指向void类型对象的指针，这个对象将存储用户数据。
2. 是一个名为 "topology" 的hwloc_topology_t类型的参数，用于设置在从XML中导入数据时的顶级拓扑结构。
3. 是一个名为 "obj" 的hwloc_obj_t类型的参数，用于存储用户数据的目标对象。
4. 是一个名为 "name" 的字符数组，用于指定存储用户数据的名称。
5. 是一个名为 "buffer" 的void类型指针，用于存储用户数据。
6. 是一个名为 "length" 的size_t类型的参数，用于指定用户数据的长度。

函数的作用是在从XML中导入数据时设置应用特定的回调函数，将用户数据存储在给定的对象中。当topology尚未设置完成时，可以先调用函数，这样就可以在使用hwloc_topology_load()函数加载topology之后，在当前对象上设置用户数据。函数可以在hwloc_export_obj_userdata()函数中被调用，因为在这个函数中，topology和对象都被设置好了，可以调用函数来设置用户数据。


```cpp
HWLOC_DECLSPEC int hwloc_export_obj_userdata_base64(void *reserved, hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length);

/** \brief Set the application-specific callback for importing userdata
 *
 * On XML import, userdata is ignored by default because hwloc does not know
 * how to store it in memory.
 *
 * This function lets applications set \p import_cb to a callback function
 * that will get the XML-stored userdata and store it in the object as expected
 * by the application.
 *
 * \p import_cb is called during hwloc_topology_load() as many times as
 * hwloc_export_obj_userdata() was called during export. The topology
 * is not entirely setup yet. Object attributes are ready to consult,
 * but links between objects are not.
 *
 * \p import_cb may be \c NULL if userdata should be ignored during import.
 *
 * \note \p buffer contains \p length characters followed by a null byte ('\0').
 *
 * \note This function should be called before hwloc_topology_load().
 *
 * \note The topology-specific userdata pointer is ignored when importing from XML.
 */
```

The hwloc\_topology\_export\_synthetic\_flags\_e enum defines a set of flags for exporting synthetic topologies in hwloc. These flags are OR'ed together to form a single value that can be passed to hwloc\_topology\_export\_synthetic() function.

The possible values for the enum are:

* HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_NO\_EXTENDED\_TYPES: This flag prevents the export of extended topologies such as L2dcache.
* HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_NO\_ATTRS: This flag prevents the export of level attributes such as memory/cache sizes or PU indexes.
* HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_V1: This flag enables version 1 of the synthetic topology export, where memory hierarchy is exported as expected in hwloc 1.x.
* HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_V1\*: This flag is the same as HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_V1, but it is the OR of all other HWLOC\_TOPOLOGY\_EXPORT\_SYNTHETIC\_FLAG\_V1 values.

The hwloc\_topology\_export\_synthetic\_flags\_e enum can be useful for debugging and understanding the behavior of the hwloc\_topology\_export\_synthetic() function.


```cpp
HWLOC_DECLSPEC void hwloc_topology_set_userdata_import_callback(hwloc_topology_t topology,
								void (*import_cb)(hwloc_topology_t topology, hwloc_obj_t obj, const char *name, const void *buffer, size_t length));

/** @} */


/** \defgroup hwlocality_syntheticexport Exporting Topologies to Synthetic
 * @{
 */

/** \brief Flags for exporting synthetic topologies.
 *
 * Flags to be given as a OR'ed set to hwloc_topology_export_synthetic().
 */
enum hwloc_topology_export_synthetic_flags_e {
 /** \brief Export extended types such as L2dcache as basic types such as Cache.
  *
  * This is required if loading the synthetic description with hwloc < 1.9.
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES = (1UL<<0),

 /** \brief Do not export level attributes.
  *
  * Ignore level attributes such as memory/cache sizes or PU indexes.
  * This is required if loading the synthetic description with hwloc < 1.10.
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS = (1UL<<1),

 /** \brief Export the memory hierarchy as expected in hwloc 1.x.
  *
  * Instead of attaching memory children to levels, export single NUMA node child
  * as normal intermediate levels, when possible.
  * This is required if loading the synthetic description with hwloc 1.x.
  * However this may fail if some objects have multiple local NUMA nodes.
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1 = (1UL<<2),

 /** \brief Do not export memory information.
  *
  * Only export the actual hierarchy of normal CPU-side objects and ignore
  * where memory is attached.
  * This is useful for when the hierarchy of CPUs is what really matters,
  * but it behaves as if there was a single machine-wide NUMA node.
  * \hideinitializer
  */
 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY = (1UL<<3)
};

```

这段代码定义了一个名为 `hwloc_topology_export_synthetic` 的函数，它的作用是输出指定拓扑结构的拓扑信息到一个字符数组中。

函数接受三个参数：一个 `hwloc_topology_t` 的结构体表示拓扑结构，一个指向字符数组的指针 `buffer`，和一个字符数组长度 `buflen`。函数返回字符数组中实际存储的字符数，不包括末尾的 `0`。

函数内部先定义了一个名为 `hwloc_topology_export_synthetic_flags_e` 的 OR 变量，它包含了一个由多个标志组成的集合。这些标志用于指示拓扑结构是否对称。

函数实现中，首先判断输入的拓扑结构是否对称，如果不对称，就无法正常 export。如果能够正常 export，函数会将拓扑结构中的 topology 和 flags 存储到一个字符数组中，并返回字符数组长度。注意，仅输出拓扑结构中的 normal 子拓扑，而忽略其他子拓扑。


```cpp
/** \brief Export the topology as a synthetic string.
 *
 * At most \p buflen characters will be written in \p buffer,
 * including the terminating \0.
 *
 * This exported string may be given back to hwloc_topology_set_synthetic().
 *
 * \p flags is a OR'ed set of ::hwloc_topology_export_synthetic_flags_e.
 *
 * \return The number of characters that were written,
 * not including the terminating \0.
 *
 * \return -1 if the topology could not be exported,
 * for instance if it is not symmetric.
 *
 * \note I/O and Misc children are ignored, the synthetic string only
 * describes normal children.
 *
 * \note A 1024-byte buffer should be large enough for exporting
 * topologies in the vast majority of cases.
 */
  HWLOC_DECLSPEC int hwloc_topology_export_synthetic(hwloc_topology_t topology, char *buffer, size_t buflen, unsigned long flags);

```

这段代码是一个C语言中的一个头文件，它定义了一个名为`__attribute__((constructor))`的修饰符。这个修饰符可以用来告诉编译器在定义函数时不要输出该函数的定义，从而使编译器更关注函数的实际代码实现。

更具体地说，这段代码定义了一个函数`constructor`，该函数没有定义。如果包含`__attribute__((constructor))`修饰符，那么编译器会在编译时将其定义成函数，并在函数定义时忽略它的定义。这样可以避免在编译时产生无意义的错误。

例如，如果在实际使用中没有定义`constructor`函数，而尝试编译时将其定义成函数，编译器会忽略它，不会生成任何错误。但是如果定义了这个函数，则会生成错误，因为编译器不知道该函数的实现。


```cpp
/** @} */



#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_EXPORT_H */

```

# `src/3rdparty/hwloc/include/hwloc/gl.h`

这段代码定义了一个名为"HWLOC_GL_H"的文件，旨在为使用硬件布局(hwloc)和OpenGL显示器的应用程序提供帮助。该文件包含了一些使用宏定义(macros)的函数，以帮助开发人员更轻松地与硬件和OpenGL之间进行交互。

具体来说，这段代码定义了一个名为"__hwloc_init"的宏，用于初始化hwloc设备。此外，还定义了几个其他的宏，如"__hwloc_get_物理_device"和"__hwloc_set_transform"等，用于在hwloc设备上进行更高级别的操作。

这些宏定义允许开发人员使用通用的函数来与硬件和OpenGL之间进行交互，而不必为每个具体的硬件设备编写自己的代码。这使得开发人员可以更轻松地开发具有良好用户界面和更高级别的硬件布局的应用程序。


```cpp
/*
 * Copyright © 2012 Blue Brain Project, EPFL. All rights reserved.
 * Copyright © 2012-2021 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and OpenGL displays.
 *
 * Applications that use both hwloc and OpenGL may want to include
 * this file so as to get topology information for OpenGL displays.
 */

#ifndef HWLOC_GL_H
#define HWLOC_GL_H

```

这段代码是一个C语言程序，它包括了OpenGL显示器驱动程序的头部声明。

这个头部声明中包含两个部分：

1. 引入了hwloc.h和stdio.h两个头文件。
2. 在ifdef __cplusplus; 后面声明了一个extern "C"整型变量x11，接着在#include <stdio.h>和#include <string.h>中用#ifdef __cplusplus；进行了判断，如果是C++编译器，那么就会编译出x11；否则不做任何处理。最后在#include <hwloc.h>中引入了hwlocality_gl头文件。

因此，这段代码的作用是定义了一个C++程序，在编译时需要使用C++编译器，并在运行时需要使用C++编译器。通过检查当前系统是否支持NVIDIA GPU，并引入了仅限于NVIDIA GPU的OpenGL头文件，实现了一个接口，用于从NVIDIA GPU中检索图形硬件顶点数据。


```cpp
#include "hwloc.h"

#include <stdio.h>
#include <string.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_gl Interoperability with OpenGL displays
 *
 * This interface offers ways to retrieve topology information about
 * OpenGL displays.
 *
 * Only the NVIDIA display locality information is currently available,
 * using the NV-CONTROL X11 extension and the NVCtrl library.
 *
 * @{
 */

```

这段代码定义了一个名为`hwloc_get_opengl_device`的函数，它的作用是获取给定端口和设备索引的OpenGL设备的`hwloc` OS设备对象。如果没有找到相应的设备对象，函数返回`NULL`。

函数的实现中，首先定义了一个`hwloc_obj_t`类型的变量`device_obj`，用于存储OpenGL设备的`hwloc`设备对象。然后通过调用`hwloc_find_device`函数，获取OpenGL设备的`hwloc`设备对象。如果`hwloc`设备对象被找到，将其存储在`device_obj`中，否则返回`NULL`。

函数中还定义了一个`topology_t`类型的变量`topology`，用于存储当前机器的顶图。该变量可以使用`xml_import`函数从指定端口的XML文件中加载顶图，因此可以在不匹配当前机器的情况下使用。

最后，函数中还定义了一个`__hwloc_inline`类型的声明，用于将`hwloc_get_opengl_device`函数作为`hwloc_obj_t`类型的别名。


```cpp
/** \brief Get the hwloc OS device object corresponding to the
 * OpenGL display given by port and device index.
 *
 * \return The hwloc OS device object describing the OpenGL display
 * whose port (server) is \p port and device (screen) is \p device.
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the GL component must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 */
static __hwloc_inline hwloc_obj_t
```

这段代码的作用是获取通过指定端口和设备的 GPU 显示 OSD 设备。它首先定义了两个变量 `x` 和 `y`，分别代表水平和垂直方向的初始位置。接下来，它创建了一个名为 `osdev` 的指针变量，用于存储当前已知的 OSD 设备。

在循环中，它首先使用 `hwloc_topology_t` 结构体类型的 `topology` 来获取输入的拓扑结构。接着，对于每个已知的 OSD 设备，它首先检查设备是否为 GPU，如果是，就检查其是否与指定端口和设备匹配。如果是，就返回该 OSD 设备。在循环结束后，如果找不到匹配的 OSD 设备，就返回一个 `NULL` 表示出错。

代码中使用了一个名为 `EINVAL` 的错误码，它表示发生了不可重复的错误。如果发生错误，请检查你的输入参数是否正确。


```cpp
hwloc_gl_get_display_osdev_by_port_device(hwloc_topology_t topology,
					  unsigned port, unsigned device)
{
        unsigned x = (unsigned) -1, y = (unsigned) -1;
        hwloc_obj_t osdev = NULL;
        while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
                    && sscanf(osdev->name, ":%u.%u", &x, &y) == 2
                    && port == x && device == y)
                        return osdev;
        }
	errno = EINVAL;
        return NULL;
}

```

这段代码定义了一个名为`get_opengl_display`的函数，用于获取一个OpenGL显示设备对象，该函数接受一个以`name`为参数的OpenGL显示设备名称，返回描述该名称OpenGL显示设备的`hwloc_obj_t`类型的对象，如果无法找到相应的OpenGL显示设备，则返回`NULL`。

该函数返回类型为`hwloc_obj_t`，定义了一个`hwloc_obj_t`类型的函数指针，指针变量类型需要手动指定。函数指针中包含两个参数，第一个参数是`name`，第二个参数是`topology`，用于指定`get_opengl_display`函数需要查询的OpenGL显示设备的信息。

函数实现中，首先判断`topology`是否与当前机器的硬件平台相匹配，如果不匹配，则需要通过查询`OS device parent object`来获取相应的硬件平台。然后，通过对`name`的解析，获取到对应的OpenGL设备名称，最后返回对应的`hwloc_obj_t`类型的对象。如果通过上述步骤仍然无法找到对应的OpenGL显示设备，则返回`NULL`。


```cpp
/** \brief Get the hwloc OS device object corresponding to the
 * OpenGL display given by name.
 *
 * \return The hwloc OS device object describing the OpenGL display
 * whose name is \p name, built as ":port.device" such as ":0.0" .
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the GL component must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 */
static __hwloc_inline hwloc_obj_t
```

这段代码的作用是获取给定硬件定位（hwloc）中的OpenGL显示端口和设备的名称。

在函数中，首先定义了一个名为osdev的硬件设备对象，并将其初始化为NULL。然后使用一个while循环，在循环体内使用hwloc_get_next_osdev函数来获取第一个具有GPU类型的OS设备对象，并将其存储在osdev中。

接下来，使用if语句检查给定的名字是否与osdev的名称匹配。如果匹配，就返回osdev，否则会输出错误码并返回NULL。


```cpp
hwloc_gl_get_display_osdev_by_name(hwloc_topology_t topology,
				   const char *name)
{
        hwloc_obj_t osdev = NULL;
        while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
                    && !strcmp(name, osdev->name))
                        return osdev;
        }
	errno = EINVAL;
        return NULL;
}

/** \brief Get the OpenGL display port and device corresponding
 * to the given hwloc OS object.
 *
 * Retrieves the OpenGL display port (server) in \p port and device (screen)
 * in \p screen that correspond to the given hwloc OS device object.
 *
 * \return \c -1 if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the GL component must be enabled in the topology.
 */
```

该函数的作用是获取通过操作系统设备的显示设备（例如GPU）并返回其硬件设备（例如显示卡）的ID，以便在hwloc_topology_t结构中使用。函数首先定义了x和y变量，初始化为-1。然后，函数遍历osdev->name数组，查找与osdev->attr->osdev.type相同且包含"GPU"的元素，如果找到，则函数将x和y的值存储到port和device变量中，并返回0。否则，函数将errno设置为EINVAL并返回-1。


```cpp
static __hwloc_inline int
hwloc_gl_get_display_by_osdev(hwloc_topology_t topology __hwloc_attribute_unused,
			      hwloc_obj_t osdev,
			      unsigned *port, unsigned *device)
{
	unsigned x = -1, y = -1;
	if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
	    && sscanf(osdev->name, ":%u.%u", &x, &y) == 2) {
		*port = x;
		*device = y;
		return 0;
	}
	errno = EINVAL;
	return -1;
}

```

这段代码是一个C/C++语言的预处理指令，用于定义一个C或C++程序中的符号常量。符号常量是在编译时定义的，而不是在程序运行时定义的。

具体来说，这段代码定义了一个名为"__cplusplus"的符号常量，其值为1。这意味着，在任何使用"__cplusplus"的的地方，其值都被视为1。

由于符号常量是在编译时定义的，因此，如果在这个符号常量之前定义了一个相同的符号常量，那么它们的值将相等，但不会被自动激活。也就是说，如果定义了多次"__cplusplus"，它们将始终被视为同一个常量，但只有最后定义的那一个将被视为有效常量。

此外，由于这段代码使用了"extern "C""，因此它告诉编译器这个符号常量是C语言中的一个符号常量。


```cpp
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_GL_H */


```

# `src/3rdparty/hwloc/include/hwloc/glibc-sched.h`

这段代码定义了一些 macro，用于帮助在 hwloc 和 glibc 调度程序之间进行交互。它包含以下几行：

```cpp
#include <sys/types.h>
#include <sys/syscall.h>
#include <signal.h>
#include <pthread.h>

#define __FILE__ "hwloc_sched.h"
```

首先，它定义了一个名为“hwloc_sched.h”的头文件，其中包含了一些与 hwloc 和 glibc 相关的定义。

```cpp
#define MAX_JOB_NAME_LENGTH                       100
#define MAX_NUMS_IN_JOB                          50
#define MAX_PRIORITY_IN_JOB                    30
#define MAX_SCHED_IN_JOB                         10
```

接下来，定义了一些宏，用于帮助转换 hwloc 和 glibc 调度程序之间的数据类型。

```cpp
#define MAX_ACTIVE_JOB_COUNT                     1024
#define MAX_SURPLUS_JOB_COUNT                   1024
#define MAX_SCHEDULED_JOB_COUNT                   1024
```

接着，定义了一些信号变量，以便在 hwloc 和 glibc 调度程序之间传递信号。

```cpp
#define JOIN_COMMAND                             0
#define PASS_COMMAND                             1
#define WAIT_COMMAND                             2
#define做一些协商工作...
```

然后，定义了一个名为“sched_control_dependencies”的结构体，用于记录在哪个 glibc 函数中，以便在 hwloc 调度程序中进行查找。

```cpp
typedef struct {
   int fd;
   int rev;
} sched_control_dependencies;
```

接下来，定义了一些函数，用于将给定的 hwloc 作业设置为当前调度程序。

```cpp
void sched_set_job_control_dependencies(int fd, int rev);
```

```cpp
void sched_set_job_dependencies(int fd, int sched_line);
```

这两个函数分别用于设置给定作业的调度程序和依赖关系。

```cpp
void sched_set_active_job_count(int job_fd, int job_count);
```

```cpp
void sched_set_surplus_job_count(int job_fd, int job_count);
```

这两个函数分别用于设置给定作业的当前和剩余数量。

```cpp
void sched_set_max_active_job_count(int max_active_count, int job_fd, int sched_line);
```

```cpp
void sched_set_max_surplus_job_count(int max_surplus_count, int job_fd, int sched_line);
```

这两个函数分别用于设置给定作业的最大当前和剩余数量。

```cpp
void sched_set_max_scheduled_job_count(int max_scheduled_count, int job_fd, int sched_line);
```

```cpp
void sched_set_dependencies_for_job(int job_fd, int sched_line, int dependency);
```

这个函数用于设置给定作业的调度程序和依赖关系。

```cpp
void sched_set_joins_for_job(int job_fd, int sched_line, int join_command);
```

```cpp
void sched_set_passes_for_job(int job_fd, int sched_line, int pass_command);
```

这两个函数分别用于设置给定作业的调度程序。

```cpp
void sched_set_max_active_passes(int max_active_passes, int job_fd, int sched_line);
```

```cpp
void sched_set_max_surplus_passes(int max_surplus_passes, int job_fd, int sched_line);
```

这两个函数分别用于设置给定作业的最大剩余数量。

```cpp
void sched_set_start_time_for_job(int job_fd, int sched_line, struct timespec start_time);
```

```cpp
void sched_set_end_time_for_job(int job_fd, int sched_line, struct timespec end_time);
```

这两个函数分别用于设置给定作业的起始和结束时间。

```cpp
void sched_set_hwloc_scheduler(int sched_line, struct hwloc_scheduler *scheduler);
```

```cpp
void sched_set_glibc_scheduler(int sched_line, struct hwloc_scheduler *scheduler);
```

这两个函数分别用于设置给定作业的 hwloc 和 glibc 调度程序。

```cpp
void sched_scheduler_interrupt(int sched_line);
```

```cpp
void sched_scheduler_signal(int sched_line);
```

这两个函数分别用于在 hwloc 和 glibc 调度程序之间传递信号。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2020 Inria.  All rights reserved.
 * Copyright © 2009-2011 Université Bordeaux
 * Copyright © 2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and glibc scheduling routines.
 *
 * Applications that use both hwloc and glibc scheduling routines such as
 * sched_getaffinity() or pthread_attr_setaffinity_np() may want to include
 * this file so as to ease conversion between their respective types.
 */

```



这段代码定义了一个名为 "HWLOC_GLIBC_SCHED_H" 的头文件，其中包含了一些与 Sched 相关的函数和声明。

具体来说，这段代码实现了一些必不可少的内容，以满足 Linux 系统的需求。其中包括：

- 包含 "hwloc.h" 头文件，它是 HWLOC 库的源代码文件，定义了一些常用的宏和函数，与这段代码的作用无关。
- 包含 "helper.h" 头文件，它是 HWLOC 库的帮助文档，定义了一些常用的函数和声明，与这段代码的作用无关。
- 包含了一些与 sched 相关的函数和声明，包括 assert.h，以及 sched.h，定义了一些与操作系统 调度相关的函数，可以实现排他性，确保同一时间只有一个进程运行。
- 定义了一些常量和宏，包括 CPU_SETSIZE，它定义了 CPU 缓存的容量，以及 sched_priority，它定义了调度程序的优先级，用于确定调度程序的执行顺序。

由于这段代码定义了一些与操作系统调度相关的函数和常量，因此它必须在任何包含这些函数或常量的项目中才能被编译。


```cpp
#ifndef HWLOC_GLIBC_SCHED_H
#define HWLOC_GLIBC_SCHED_H

#include "hwloc.h"
#include "hwloc/helper.h"

#include <assert.h>

#if !defined _GNU_SOURCE || (!defined _SCHED_H && !defined _SCHED_H_) || (!defined CPU_SETSIZE && !defined sched_priority)
#error Please make sure to include sched.h before including glibc-sched.h, and define _GNU_SOURCE before any inclusion of sched.h
#endif


#ifdef __cplusplus
extern "C" {
```

这段代码是一个C语言中的预处理指令，它的作用是检查当前系统是否支持硬件资源调度（hwlocality）和/或协作进程调度（glibc scheduler）。如果系统支持，则该接口可以用于将hwloc CPU集合和glibc CPU集合进行转换，例如使用sched_getaffinity()或pthread_attr_setaffinity_np()函数。

具体来说，这段代码定义了一个名为`hwlocality_glibc_sched`的函数接口，它提供了将hwloc CPU集合和glibc CPU集合相互转换的方法。此外，该接口还定义了一个预处理器指令`#ifdef`，用于在代码块之前检查特定预处理指令是否定义。如果该预处理器指令被定义，则编译器将编译该代码块。

如果`#ifdef HWLOC_HAVE_CPU_SET`预处理器指令被定义，则该代码块编译为可执行文件。否则，编译器将其视为普通C代码。


```cpp
#endif


#ifdef HWLOC_HAVE_CPU_SET


/** \defgroup hwlocality_glibc_sched Interoperability with glibc sched affinity
 *
 * This interface offers ways to convert between hwloc cpusets and glibc cpusets
 * such as those manipulated by sched_getaffinity() or pthread_attr_setaffinity_np().
 *
 * \note Topology \p topology must match the current machine.
 *
 * @{
 */


```

这段代码是一个静态函数，名为`hwloc_cpuset_to_glibc_sched_affinity`，它的输入参数是一个`hwloc_topology_t`类型的`topology`和一个`hwloc_const_cpuset_t`类型的`hwlocset`，输出参数是一个`size_t`类型的整数，表示`schedset`可以被接受的最大的`cpu_set_t`集合。

该函数的作用是将一个`hwloc_const_cpuset_t`类型的`hwlocset`设置为与操作系统中的`cpu_set_t`数据类型对应的`schedset`，该函数可以被用于在调用`sched_setaffinity`函数之前设置一个`cpu_set_t`类型的`schedset`，该函数的输入参数为`schedset`，大小为`size_t`类型的`schedsetsize`。


```cpp
/** \brief Convert hwloc CPU set \p toposet into glibc sched affinity CPU set \p schedset
 *
 * This function may be used before calling sched_setaffinity or any other function
 * that takes a cpu_set_t as input parameter.
 *
 * \p schedsetsize should be sizeof(cpu_set_t) unless \p schedset was dynamically allocated with CPU_ALLOC
 */
static __hwloc_inline int
hwloc_cpuset_to_glibc_sched_affinity(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t hwlocset,
				    cpu_set_t *schedset, size_t schedsetsize)
{
#ifdef CPU_ZERO_S
  unsigned cpu;
  CPU_ZERO_S(schedsetsize, schedset);
  hwloc_bitmap_foreach_begin(cpu, hwlocset)
    CPU_SET_S(cpu, schedsetsize, schedset);
  hwloc_bitmap_foreach_end();
```

这段代码是一个 C 语言函数，名为“convert_sched_affinity”，它将一个 glibc（GNU C库）中的 CPU 设置（schedset）映射到硬件平台上的 CPU 设置（hwlocset）。

首先，函数检查是否已经使用过 `CPU_ZERO` 和 `CPU_SET` 函数。如果没有，那么函数将创建一个新的 CPU 设置并将其设置为当前调度程序（schedset）的 CPU 设置。这样，就可以确保在将来的调度程序中，不会出现由于现有 CPU 设置导致的竞争条件。

如果 `CPU_ZERO` 和 `CPU_SET` 函数已经被使用过了，那么函数将直接使用给定的 `schedset` 和 `schedsetsize`（当前 CPU 设置的大小）创建一个新的 CPU 设置，并将其设置为当前调度程序的 CPU 设置。

最后，函数返回 0，表示成功地将 glibc 的调度程序映射到硬件平台上的 CPU 设置。


```cpp
#else /* !CPU_ZERO_S */
  unsigned cpu;
  CPU_ZERO(schedset);
  assert(schedsetsize == sizeof(cpu_set_t));
  hwloc_bitmap_foreach_begin(cpu, hwlocset)
    CPU_SET(cpu, schedset);
  hwloc_bitmap_foreach_end();
#endif /* !CPU_ZERO_S */
  return 0;
}

/** \brief Convert glibc sched affinity CPU set \p schedset into hwloc CPU set
 *
 * This function may be used before calling sched_setaffinity  or any other function
 * that takes a cpu_set_t  as input parameter.
 *
 * \p schedsetsize should be sizeof(cpu_set_t) unless \p schedset was dynamically allocated with CPU_ALLOC
 */
```

这段代码是一个静态函数，名为`hwloc_cpuset_from_glibc_sched_affinity`，属于`hwloc_topology_t`类型的`hwloc_attribute_unused`修饰。它的作用是返回一个整数，表示当前CPU属于哪个`hwloc_cpuset_t`。

函数接收三个参数：

1. `topology`：属于`hwloc_topology_t`类型的整数，用于表示输入的topology结构。但是，这个参数没有被使用。
2. `hwlocset`：属于`hwloc_cpuset_t`类型的整数，用于表示需要返回的CPU集合。
3. `schedset`：属于`cpu_set_t`类型的整数，用于指定当前CPU属于哪个schedset。
4. `schedsetsize`：属于`size_t`类型的整数，用于指定schedset的大小。

函数内部首先执行一个循环，遍历schedsetsize个schedset，检查当前CPU是否属于schedset。如果是，就执行以下操作：

1. 设置hwlocset中cpu指向的schedset的hwloc，然后将schedsetsize减1。
2. 将当前CPU的计数器减1，用于统计当前CPU是否已经遍历完schedsetsize个schedset。

最终，函数返回当前CPU属于哪个`hwloc_cpuset_t`。


```cpp
static __hwloc_inline int
hwloc_cpuset_from_glibc_sched_affinity(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_cpuset_t hwlocset,
                                       const cpu_set_t *schedset, size_t schedsetsize)
{
  int cpu;
#ifdef CPU_ZERO_S
  int count;
#endif
  hwloc_bitmap_zero(hwlocset);
#ifdef CPU_ZERO_S
  count = CPU_COUNT_S(schedsetsize, schedset);
  cpu = 0;
  while (count) {
    if (CPU_ISSET_S(cpu, schedsetsize, schedset)) {
      hwloc_bitmap_set(hwlocset, cpu);
      count--;
    }
    cpu++;
  }
```

这段代码是一个C语言函数，它检查在schedsetsize变量所表示的内存区域中，是否包含某个特定内联的CPU设置（CPU_SETT）。如果是，则执行该内联设置，并使用hwloc_bitmap_set函数将该CPU设置映射到硬件位置。如果该内联设置不存在，则表示schedsetsize变量所表示的内存区域与sched严厉打击无关，不需要执行任何操作。

注意，该函数使用了几个宏定义，包括sched、schedsetsize和CPU_SETT。此外，该函数使用了if语句和assert语句，if语句用于检查schedsetsize变量所表示的内存区域是否包含某个特定内联的CPU设置，assert语句用于确保该内存区域包含CPU_SETT（即使是在很老的接口中）。


```cpp
#else /* !CPU_ZERO_S */
  /* sched.h does not support dynamic cpu_set_t (introduced in glibc 2.7),
   * assume we have a very old interface without CPU_COUNT (added in 2.6)
   */
  assert(schedsetsize == sizeof(cpu_set_t));
  for(cpu=0; cpu<CPU_SETSIZE; cpu++)
    if (CPU_ISSET(cpu, schedset))
      hwloc_bitmap_set(hwlocset, cpu);
#endif /* !CPU_ZERO_S */
  return 0;
}

/** @} */


```

这段代码是一个C语言中的预处理指令。作用是检查当前源文件是否定义了`CPU_SET`变量。如果是，则定义了该变量，代码将跳过下面的注释。如果不是，则执行下面注释中的代码。

具体来说，代码分为两部分。第一部分是一个标识符，表示当前文件是否为C++语言编写的。如果是，则预处理指令将编译为可执行文件。第二部分是一个注释，表示如果当前文件不是C++语言编写的，则不需要执行任何操作。

在第一部分中，如果当前文件是C++语言编写的，则定义了一个名为`CPU_SET`的变量，并将其设置为真。这意味着编译器将把一些C++语言特有的预处理指令编译到可执行文件中。

如果当前文件不是C++语言编写的，则直接跳过第一部分的注释，并继续执行下一部分的代码。


```cpp
#endif /* CPU_SET */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_GLIBC_SCHED_H */

```

# `src/3rdparty/hwloc/include/hwloc/helper.h`

此代码定义了一个名为"hwloc_helper.h"的文件，其中包括一些定义和函数，用于提供高性能的硬件抽象层(HAL)路径定位器和查找器。

具体来说，这些函数和定义提供了以下功能：

- "hwloc_is_hwloc_device()"：用于检查给定的硬件设备是否为硬件定位设备(如CPU、GPU等)。
- "hwloc_get_device_tree_root()"：用于获取给定硬件设备的设备树根节点，以便进一步定位该设备。
- "hwloc_get_device_node()"：用于获取给定硬件设备的设备节点，进一步了解该硬件设备的信息。
- "hwloc_get_device_ properties()"：用于获取给定硬件设备的属性，例如电压、时钟频率、架构等。
- "hwloc_set_device_tree_root()"：用于设置给定硬件设备的设备树根节点。
- "hwloc_set_device_node()"：用于设置给定硬件设备的设备节点。
- "hwloc_set_device_properties()"：用于设置给定硬件设备的属性，例如电压、时钟频率、架构等。

这些函数和定义都可以被用于开发支持多种硬件设备的驱动程序或应用程序。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2010 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief High-level hwloc traversal helpers.
 */

#ifndef HWLOC_HELPER_H
#define HWLOC_HELPER_H

```

这段代码是一个C语言的预处理指令，它主要作用是检查源文件中是否定义了`hwlocality_helper_find_inside`函数。如果没有定义该函数，则程序会报错，并提示用户包含`main.h`文件才是正确的。如果已经定义了这个函数，那么这段代码就无实际意义了，因为它已经代表着这是一个无效的代码。

这里给出的是一个C语言编译器（如gcc）使用的预处理指令，当程序在编译之前遇到包含`hwlocality_helper_find_inside`这个词汇时，会检查`main.h`文件中是否已经定义了这个函数。这个函数可能是用于在CPU设置中查找特定对象的。如果这个函数尚未定义，则编译器无法识别编译，并报错。


```cpp
#ifndef HWLOC_H
#error Please include the main hwloc.h instead
#endif

#include <stdlib.h>
#include <errno.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_helper_find_inside Finding Objects inside a CPU set
 * @{
 */

```

这段代码定义了一个名为 `hwloc_get_first_largest_obj_inside_cpuset` 的函数，它接受一个 `hwloc_topology_t` 和一个 `hwloc_const_cpuset_t` 类型的参数。函数返回第一个包含在给定的 CPU 设置中的最大对象（也称为“根对象”），并且该对象的父节点在给定的 CPU 设置中不是。

该函数首先定义了一个名为 `obj` 的变量，该变量代表根对象。接下来，函数判断根对象是否属于给定的 CPU 设置。如果是，函数将返回该根对象。如果不是，函数将遍历根对象的第一个子对象，并在子对象中查找第一个与给定 CPU 设置中的元素相交的元素。如果找到了相交的元素，函数将返回该子对象。如果遍历完所有的子对象都没有找到与给定 CPU 设置中的元素相交的元素，函数将返回根对象（因为根对象本身就是包含在给定 CPU 设置中的）。

该函数可以通过递归调用来遍历整个 CPU 设置，从而方便地迭代所有最大的对象。


```cpp
/** \brief Get the first largest object included in the given cpuset \p set.
 *
 * \return the first object that is included in \p set and whose parent is not.
 *
 * This is convenient for iterating over all largest objects within a CPU set
 * by doing a loop getting the first largest object and clearing its CPU set
 * from the remaining CPU set.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_first_largest_obj_inside_cpuset(hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  if (!hwloc_bitmap_intersects(obj->cpuset, set))
    return NULL;
  while (!hwloc_bitmap_isincluded(obj->cpuset, set)) {
    /* while the object intersects without being included, look at its children */
    hwloc_obj_t child = obj->first_child;
    while (child) {
      if (hwloc_bitmap_intersects(child->cpuset, set))
	break;
      child = child->next_sibling;
    }
    if (!child)
      /* no child intersects, return their father */
      return obj;
    /* found one intersecting child, look at its children */
    obj = child;
  }
  /* obj is included, return it */
  return obj;
}

```

这段代码定义了一个名为 `hwloc_get_largest_objs_inside_cpuset` 的函数，它的实现在于输入参数上：

1. 首先，定义了一个名为 `topology` 的输入参数，是一个 `hwloc_topology_t` 类型的数据结构，它表示要操作的图形环境。
2. 接下来，定义了一个名为 `set` 的输入参数，是一个 `hwloc_const_cpuset_t` 类型的数据结构，表示要包含在计算中的 CPU 集合。
3. 接着，定义了一个名为 `objs` 的输入参数，是一个 `hwloc_obj_t` 类型的数据结构，它表示要返回的一组图形对象。
4. 定义了一个名为 `max` 的输入参数，是一个整数类型的数据结构，表示要在 CPU 集合中找到的最大深度。

函数实现如下：
```cppc
int hwloc_get_largest_objs_inside_cpuset(hwloc_topology_t topology, hwloc_const_cpuset_t set,
                                            hwloc_obj_t * __hwloc_restrict objs, int max)
```
首先，函数接收两个输入参数：`topology` 和 `set`。
```cpp


```
/** \brief Get the set of largest objects covering exactly a given cpuset \p set
 *
 * \return the number of objects returned in \p objs.
 */
HWLOC_DECLSPEC int hwloc_get_largest_objs_inside_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set,
						 hwloc_obj_t * __hwloc_restrict objs, int max);

/** \brief Return the next object at depth \p depth included in CPU set \p set.
 *
 * If \p prev is \c NULL, return the first object at depth \p depth
 * included in \p set.  The next invokation should pass the previous
 * return value in \p prev so as to obtain the next object in \p set.
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects at the given depth do
 * not have CPU sets (I/O or Misc objects).
 */
```cpp

这段代码定义了一个名为`hwloc_get_next_obj_inside_cpuset_by_depth`的函数，它接受一个`hwloc_topology_t`类型的顶图、一个`hwloc_const_cpuset_t`类型的设置和一个`hwloc_obj_t`类型的前一个对象作为参数。

该函数的作用是返回一个包含在给定设置中的`hwloc_obj_t`类型的下一个对象的引用，如果当前对象存在于设置中，否则返回`NULL`。该函数会在`hwloc_topology_t`的树中搜索对象，并沿着`hwloc_get_next_obj_by_depth`函数返回的链向上递归搜索，直到找到包含给定对象的CPU集合或者到达顶级。

该函数仅在给定类型对象的CPU集合中有效，如果该对象没有CPU集合，则返回`NULL`。如果给定的对象是`hwloc_iore鹏口`（可能是I/O设备或服务），则返回`hwloc_get_next_obj_inside_cpuset_by_depth`函数的返回值。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					   int depth, hwloc_obj_t prev)
{
  hwloc_obj_t next = hwloc_get_next_obj_by_depth(topology, depth, prev);
  if (!next)
    return NULL;
  while (next && (hwloc_bitmap_iszero(next->cpuset) || !hwloc_bitmap_isincluded(next->cpuset, set)))
    next = next->next_cousin;
  return next;
}

/** \brief Return the next object of type \p type included in CPU set \p set.
 *
 * If there are multiple or no depth for given type, return \c NULL
 * and let the caller fallback to
 * hwloc_get_next_obj_inside_cpuset_by_depth().
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects of the given type do
 * not have CPU sets (I/O or Misc objects).
 */
```cpp

这段代码定义了一个名为`hwloc_get_next_obj_inside_cpuset_by_type`的函数，属于`hwloc_loc`系列的函数。它接受一个`hwloc_topology_t`类型的顶类图结构，一个`hwloc_const_cpuset_t`类型的设置，一个`hwloc_obj_type_t`类型的类型参数和一个`hwloc_obj_t`类型的预先值。

该函数的作用是返回在给定的CPU集合中，类型为给定顶类图结构中指定的对象的项（或称为索引）的逻辑索引。如果类型参数或顶类图结构中指定的对象在CPU集合中没有相应的项，那么函数将返回`NULL`。

具体来说，函数首先通过`hwloc_get_type_depth`函数获取给定的顶类图结构和类型之间的深度。如果深度为`HWLOC_TYPE_DEPTH_UNKNOWN`或`HWLOC_TYPE_DEPTH_MULTIPLE`，那么函数将返回`NULL`，否则它将返回`hwloc_get_next_obj_inside_cpuset_by_depth`函数，这个函数将在给定的CPU集合中返回深度为`depth`的下一个对象的逻辑索引，直到返回到`HWLOC_TYPE_DEPTH_END`。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					  hwloc_obj_type_t type, hwloc_obj_t prev)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_next_obj_inside_cpuset_by_depth(topology, set, depth, prev);
}

/** \brief Return the (logically) \p idx -th object at depth \p depth included in CPU set \p set.
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects at the given depth do
 * not have CPU sets (I/O or Misc objects).
 */
```cpp

该代码定义了一个名为 `hwloc_get_obj_inside_cpuset_by_depth` 的函数，它接受一个 `hwloc_topology_t` 和一个 `hwloc_const_cpuset_t` 作为参数。该函数返回一个在 `hwloc_topology_t` 中从 `hwloc_const_cpuset_t` 中选择深度为 `depth` 的 `hwloc_obj_t` 对象的引用，如果选择了对象失败则返回 `NULL`。

该函数的核心思想是遍历 `hwloc_topology_t` 中的所有对象，对于每个对象，首先检查它是否属于指定的 `cpuset`，如果是，并且它在当前 `cpuset` 中，则返回该对象。如果当前对象无法确定是否属于指定的 `cpuset`，则遍历它的所有相邻对象，直到找到匹配的 `cpuset` 或者遍历完所有对象为止。如果所有的对象都未找到匹配的 `cpuset`，则返回 `NULL`。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
				      int depth, unsigned idx) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
				      int depth, unsigned idx)
{
  hwloc_obj_t obj = hwloc_get_obj_by_depth (topology, depth, 0);
  unsigned count = 0;
  if (!obj)
    return NULL;
  while (obj) {
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set)) {
      if (count == idx)
	return obj;
      count++;
    }
    obj = obj->next_cousin;
  }
  return NULL;
}

```cpp

这段代码定义了一个名为 `hwloc_get_obj_inside_cpuset_by_type` 的函数，它用于返回在给定拓扑结构中属于指定类型的对象中，第 `idx` 个对象的 CPU 集。

函数的实现基于两个条件：给定的拓扑结构和要查找的对象的类型。如果给定的拓扑结构中不存在指定类型的对象，函数将返回 `NULL`，否则它将返回指定拓扑结构中与指定类型具有相同深度（通过 `hwloc_get_obj_inside_cpuset_by_depth()` 函数可以获取）的对象的 CPU 集。

函数的实现还包含一个限制条件：如果给定的对象类型为空闲的（没有 CPU 集或 I/O 对象），则函数不会将其包含在给定的拓扑结构中。


```
/** \brief Return the \p idx -th object of type \p type included in CPU set \p set.
 *
 * If there are multiple or no depth for given type, return \c NULL
 * and let the caller fallback to
 * hwloc_get_obj_inside_cpuset_by_depth().
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects of the given type do
 * not have CPU sets (I/O or Misc objects).
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
				     hwloc_obj_type_t type, unsigned idx) __hwloc_attribute_pure;
```cpp

这段代码定义了一个名为 `hwloc_get_obj_inside_cpuset_by_type` 的函数，属于 `hwloc_topology_ex` 库。它的作用是返回在指定的 CPU 集合中，与给定类型相同的对象的深度。

具体来说，函数的第一个参数是一个 `hwloc_topology_t` 类型的顶片（Topology）结构，第二个参数是一个 `hwloc_const_cpuset_t` 类型的设置（Set），第三个参数是一个 `hwloc_obj_type_t` 类型的整数，最后一个参数是一个 `unsigned int` 类型的整数（Index）。函数首先检查给定类型的对象的深度是否为 "unknown" 或 "multiple"，如果是，则返回 `NULL`，否则进入循环。在循环中，函数接收一个指向 `hwloc_get_obj_inside_cpuset_by_depth` 函数的指针，该函数将在给定的顶片和设置上递归查找与指定类型相同的对象的深度。然后，函数使用给定的索引和深度，返回在给定设置中与指定类型相同的对象的个数。如果给定对象没有设置，函数将忽略它们。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
				     hwloc_obj_type_t type, unsigned idx)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_obj_inside_cpuset_by_depth(topology, set, depth, idx);
}

/** \brief Return the number of objects at depth \p depth included in CPU set \p set.
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects at the given depth do
 * not have CPU sets (I/O or Misc objects).
 */
```cpp

这两段代码定义了一个名为 `hwloc_get_nbobjs_inside_cpuset_by_depth` 的函数，它接受两个参数：顶上结构体 `hwloc_topology_t` 和一个指向 `hwloc_const_cpuset_t` 的引用 `set`，以及一个整数参数 `depth`。函数返回一个整数，表示在给定深度内，包含在给定 `set` 中的 `hwloc_obj_t` 对象的个数。

函数首先定义了一个名为 `hwloc_get_obj_by_depth` 的函数，它接受两个参数：顶上结构体 `hwloc_topology_t` 和一个整数参数 `depth`。函数返回一个指向 `hwloc_obj_t` 对象的指针，该对象在给定深度内与给定 `set` 中的 `hwloc_obj_t` 对象的索引。

然后定义了 `hwloc_get_nbobjs_inside_cpuset_by_depth` 函数，与之前的同名函数不同之处在于它使用了 `hwloc_topology_t` 类型的 `topology` 参数和 `hwloc_const_cpuset_t` 类型的 `set` 参数。这两个参数在函数体中都被当作整数来处理。函数首先获取给定深度内包含在给定 `set` 中的 `hwloc_obj_t` 对象的个数，然后计算这个个数与给定 `set` 中 `hwloc_bitmap_iszero` 函数的返回值的关系，这个函数返回一个布尔值，表示给定 `set` 中是否有对象。最后，函数使用给定的 `topology` 和 `set` 参数，计算给定深度内包含在给定 `set` 中的 `hwloc_obj_t` 对象的个数，并将其返回。


```
static __hwloc_inline unsigned
hwloc_get_nbobjs_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					 int depth) __hwloc_attribute_pure;
static __hwloc_inline unsigned
hwloc_get_nbobjs_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					 int depth)
{
  hwloc_obj_t obj = hwloc_get_obj_by_depth (topology, depth, 0);
  unsigned count = 0;
  if (!obj)
    return 0;
  while (obj) {
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set))
      count++;
    obj = obj->next_cousin;
  }
  return count;
}

```cpp

这段代码定义了一个名为 `hwloc_get_nbobjs_inside_cpuset_by_type` 的函数，它接受两个参数：`topology` 和 `set`，分别表示 CPU 集和设置，以及要查找的物体类型。函数返回一个整数，表示在指定的 CPU 集和设置中，指定类型对象的个数。

函数的实现基于两个假设：

1. 如果指定类型对象不存在于 CPU 集中，那么返回 0。
2. 如果指定类型对象存在于多个分层中，那么返回 -1。

对于这个问题，该函数可以确保在给定的设置中，指定类型对象的个数不会被忽略或计算错误。


```
/** \brief Return the number of objects of type \p type included in CPU set \p set.
 *
 * If no object for that type exists inside CPU set \p set, 0 is
 * returned.  If there are several levels with objects of that type
 * inside CPU set \p set, -1 is returned.
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if objects of the given type do
 * not have CPU sets (I/O objects).
 */
static __hwloc_inline int
hwloc_get_nbobjs_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					hwloc_obj_type_t type) __hwloc_attribute_pure;
```cpp

这段代码定义了一个名为 `hwloc_get_nbobjs_inside_cpuset_by_type` 的函数，属于 `hwloc_topology_t` 系列的函数，用于计算指定类型在给定 CPU 集合中的对象数量。

函数的第一个参数是一个指向 `hwloc_topology_t` 类型的整型变量 `topology`，第二个参数是一个指向 `hwloc_const_cpuset_t` 类型的整型变量 `set`，第三个参数是一个指向 `hwloc_obj_type_t` 类型的整型变量 `type`。

函数内部先调用 `hwloc_get_type_depth` 函数来获取指定类型在给定 topology 中的深度，如果深度未知或者为多重深度，则返回 0。接着，调用 `hwloc_get_nbobjs_inside_cpuset_by_depth` 函数来计算指定类型在给定 depth 的 CPU 集合中的对象数量，并返回该数量。

函数的第二个注释是一个备注，指出该函数在某些情况下可能无法正确工作，例如对象没有 CPU 设置或者在 topology 中具有多个深度时。


```
static __hwloc_inline int
hwloc_get_nbobjs_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
					hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return 0;
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return -1; /* FIXME: agregate nbobjs from different levels? */
  return (int) hwloc_get_nbobjs_inside_cpuset_by_depth(topology, set, depth);
}

/** \brief Return the logical index among the objects included in CPU set \p set.
 *
 * Consult all objects in the same level as \p obj and inside CPU set \p set
 * in the logical order, and return the index of \p obj within them.
 * If \p set covers the entire topology, this is the logical index of \p obj.
 * Otherwise, this is similar to a logical index within the part of the topology
 * defined by CPU set \p set.
 *
 * \note Objects with empty CPU sets are ignored
 * (otherwise they would be considered included in any given set).
 *
 * \note This function cannot work if obj does not have CPU sets (I/O objects).
 */
```cpp

这段代码定义了一个名为 `hwloc_get_obj_index_inside_cpuset` 的函数，它接受三个参数：`topology`、`set` 和 `obj`。函数实现如下：

1. 首先定义了一个内部函数，名为 `hwloc_get_obj_index_inside_cpuset`，它具有 `__hwloc_attribute_pure` 签声明，这意味着它不能被任何其他源代码文件或全局变量访问。
2. 在内部函数中，定义了一个整型变量 `idx`，用于记录从当前节点到要查询的节点在层次结构中传输的路径上包含当前节点的数量。
3. 在函数中，首先判断给定的 `obj` 是否属于给定的 `set`。如果是，则执行以下操作：
  a. 如果 `obj` 的 `cpuset` 集合中包含给定的 `set`，则返回 `idx`。
  b. 否则，递归地遍历 `obj` 的前一个兄弟节点，如果兄弟节点的 `cpuset` 中包含给定的 `set`，则记录 `idx`。
4. 最后，返回 `idx` 的值，作为函数的返回值。

该函数的作用是获取给定的 `obj` 属于给定的 `set` 时的索引值，用于在树状结构中定位包含 `obj` 的节点。`hwloc_topology_t` 和 `hwloc_const_cpuset_t` 信号用于定义输入参数的类型。


```
static __hwloc_inline int
hwloc_get_obj_index_inside_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
				   hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline int
hwloc_get_obj_index_inside_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
				   hwloc_obj_t obj)
{
  int idx = 0;
  if (!hwloc_bitmap_isincluded(obj->cpuset, set))
    return -1;
  /* count how many objects are inside the cpuset on the way from us to the beginning of the level */
  while ((obj = obj->prev_cousin) != NULL)
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set))
      idx++;
  return idx;
}

```cpp

这段代码定义了一个名为 `hwlocality_helper_find_covering` 的函数组，其中包括以下成员函数：

1. `__hwloc_inline` 修饰符表示该函数可以被任何 HWLOCOLIB 库中的实现称为 "hwlocality_helper_find_covering" 的函数指针或对象调用。

2. `hwlocality_helper_find_covering` 函数本身没有函数体，它是一个函数指针类型。

3. `static __hwloc_inline hwloc_obj_t` 定义了一个名为 `hwlocality_helper_find_covering` 的函数组，这个函数组可以被任何 HWLOCOLIB 库中的实现称为 "hwlocality_helper_find_covering" 的函数指针或对象调用，而函数内部执行的代码是在 `__hwloc_inline` 修饰符下定义的。

4. `static` 修饰符表示该函数组中的所有函数都是不可变形的，即它们不能被修改为不同的函数。

5. `__hwloc_inline` 修饰符表示该函数可以被任何 HWLOCOLIB 库中的实现称为 "hwlocality_helper_find_covering" 的函数指针或对象调用。

6. `hwlocality_helper_find_covering` 函数内部执行的代码是固定的，无法被修改。


```
/** @} */



/** \defgroup hwlocality_helper_find_covering Finding Objects covering at least CPU set
 * @{
 */

/** \brief Get the child covering at least CPU set \p set.
 *
 * \return \c NULL if no child matches or if \p set is empty.
 *
 * \note This function cannot work if parent does not have a CPU set (I/O or Misc objects).
 */
static __hwloc_inline hwloc_obj_t
```cpp

这段代码定义了一个名为 `hwloc_get_child_covering_cpuset` 的函数，属于 `hwloc_topology_t` 类型的辅助成员函数。

它的实现在 `hwloc_get_child_covering_cpuset` 函数内部，使用 `hwloc_topology_t` 的辅助成员函数 `hwloc_get_root_cpuset` 计算出当前物理机的 CPU 集合，然后使用 `hwloc_const_cpuset_t` 类型的辅助成员函数 `hwloc_bitmap_iszero` 来判断当前物理机是否包含指定的 CPU 集合，如果是，就返回指定的 CPU 集合的第一个元素，否则继续判断下一个元素。

具体地，如果当前物理机包含指定的 CPU 集合，就直接返回指定的 CPU 集合的第一个元素；否则，就遍历当前物理机的所有子节点，并将子节点的 CPU 集合也传递给 `hwloc_get_child_covering_cpuset` 函数，继续向下查找指定的 CPU 集合。最终，如果找到了指定的 CPU 集合，函数返回该 CPU 集合的第一个元素，否则返回 `NULL`。


```
hwloc_get_child_covering_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
				hwloc_obj_t parent) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_child_covering_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
				hwloc_obj_t parent)
{
  hwloc_obj_t child;
  if (hwloc_bitmap_iszero(set))
    return NULL;
  child = parent->first_child;
  while (child) {
    if (child->cpuset && hwloc_bitmap_isincluded(set, child->cpuset))
      return child;
    child = child->next_sibling;
  }
  return NULL;
}

```cpp

该代码定义了一个名为 `hwloc_get_obj_covering_cpuset` 的函数，用于获取在给定的拓扑结构中，至少包含给定 CPU 集的最低对象。

函数首先定义了一个名为 `hwloc_topology_t` 的枚举类型 `hwloc_topology_t`，该类型用于表示拓扑结构的信息。接下来定义了一个名为 `hwloc_const_cpuset_t` 的枚举类型 `hwloc_const_cpuset_t`，该类型用于表示CPU集的标识符。

函数的核心部分定义了一个名为 `hwloc_get_root_obj` 的函数，该函数从拓扑结构的根对象开始递归搜索，以找到至少包含给定 CPU 集的对象。如果找到或者给定 CPU 集是空集，函数返回 NULL。

在函数中还有一个名为 `hwloc_bitmap_iszero` 的函数，该函数用于检查给定的 CPU 集是否为空集。如果空集，函数立即返回 NULL，否则函数继续递归寻找对象。

函数最后一个参数是一个指向 `hwloc_obj_t` 类型的指针 `current`，用于存储当前递归的层次结构中的对象。在函数内部，递归地遍历从根对象到子对象的路径，如果找到或者子对象包含给定 CPU 集，就返回子对象，否则继续递归。


```
/** \brief Get the lowest object covering at least CPU set \p set
 *
 * \return \c NULL if no object matches or if \p set is empty.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  struct hwloc_obj *current = hwloc_get_root_obj(topology);
  if (hwloc_bitmap_iszero(set) || !hwloc_bitmap_isincluded(set, current->cpuset))
    return NULL;
  while (1) {
    hwloc_obj_t child = hwloc_get_child_covering_cpuset(topology, set, current);
    if (!child)
      return current;
    current = child;
  }
}

```cpp

这段代码定义了一个名为 `hwloc_get_next_obj_covering_cpuset_by_depth` 的函数，用于 iterate through 相同深度的对象，前提是这些对象覆盖了 CPU 集合。

函数接受三个参数：`topology` 是输入的 topology 结构体，包含用于输入的顶级结构体和用于输出连接的指针；`set` 是输入的 CPU 集合，传递给 `hwloc_get_next_obj_by_depth` 的第二个参数；`depth` 是输入参数，指定要遍历的深度；`prev` 是输入参数，用于在每次迭代中传递前一个返回值，以便在需要时获得前一个返回值。

函数首先定义了一个下一层的下一个对象，如果在当前层中找到了对象，则从下一层开始递归寻找覆盖了 CPU 集合的下一个对象。如果当前层中找不到对象，函数返回 NULL。在每一次迭代中，函数首先检查给定的前一个对象是否与当前对象的 CPU 集合相交，如果是，则递归调用下一层函数。如果当前对象和前一个对象之间不存在相交的 CPU 集合，则继续递归下一层函数。

该函数的实现基于 `hwloc_get_next_obj_by_depth` 函数，用于从指定深度开始遍历同一拓扑结构中的对象，这个函数会尝试从同一拓扑结构中的最底层开始递归，直到找到符合条件的对象。如果底层对象无法找到符合条件的对象，则返回 NULL。


```
/** \brief Iterate through same-depth objects covering at least CPU set \p set
 *
 * If object \p prev is \c NULL, return the first object at depth \p
 * depth covering at least part of CPU set \p set.  The next
 * invokation should pass the previous return value in \p prev so as
 * to obtain the next object covering at least another part of \p set.
 *
 * \note This function cannot work if objects at the given depth do
 * not have CPU sets (I/O or Misc objects).
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_covering_cpuset_by_depth(hwloc_topology_t topology, hwloc_const_cpuset_t set,
					    int depth, hwloc_obj_t prev)
{
  hwloc_obj_t next = hwloc_get_next_obj_by_depth(topology, depth, prev);
  if (!next)
    return NULL;
  while (next && !hwloc_bitmap_intersects(set, next->cpuset))
    next = next->next_cousin;
  return next;
}

```cpp

这段代码定义了一个名为`hwloc_get_next_obj_covering_cpuset_by_depth`的函数，它的作用是遍历相同类型的对象，覆盖至少一个CPU设置（包括虚拟CPU设置）。如果当前对象`prev`为`NULL`，则返回至少一个CPU设置的对象。接下来，如果`prev`仍然为`NULL`，则返回上一个返回值，以便在下一次调用时获得下一个对象。

如果同一类型对象存在多个深度，则返回`NULL`。如果调用者需要为每个深度调用该函数，则可以考虑使用`hwloc_get_next_obj_covering_cpuset_by_depth`函数。


```
/** \brief Iterate through same-type objects covering at least CPU set \p set
 *
 * If object \p prev is \c NULL, return the first object of type \p
 * type covering at least part of CPU set \p set.  The next invokation
 * should pass the previous return value in \p prev so as to obtain
 * the next object of type \p type covering at least another part of
 * \p set.
 *
 * If there are no or multiple depths for type \p type, \c NULL is returned.
 * The caller may fallback to hwloc_get_next_obj_covering_cpuset_by_depth()
 * for each depth.
 *
 * \note This function cannot work if objects of the given type do
 * not have CPU sets (I/O or Misc objects).
 */
```cpp

这段代码定义了一个名为`hwloc_get_next_obj_covering_cpuset_by_type`的函数，属于`hwloc_topology_ex`系列的帮助函数。

这个函数接受三个参数：`topology`是一个表示拓扑结构的有向无环图（hwloc_topology_t）结构体；`set`是一个表示CPU集的整数数组（hwloc_const_cpuset_t）结构体；`type`是一个表示要查找的拓扑对象的类型（hwloc_obj_type_t）整数；`prev`是一个指向上一个返回对象的指针（hwloc_obj_t）。

函数的作用是返回`hwloc_get_next_obj_covering_cpuset_by_depth`的函数的递归调用，其中`hwloc_get_next_obj_covering_cpuset_by_depth`的参数包括`topology`、`set`和`type`，返回值为在`hwloc_get_next_obj_covering_cpuset_by_depth`递归调用中返回的值，如果`topology`所表示的拓扑结构中包含具有`type`类型的对象，该函数将返回`NULL`，否则返回`hwloc_get_next_obj_covering_cpuset_by_depth`的返回值。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_covering_cpuset_by_type(hwloc_topology_t topology, hwloc_const_cpuset_t set,
					   hwloc_obj_type_t type, hwloc_obj_t prev)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_next_obj_covering_cpuset_by_depth(topology, set, depth, prev);
}

/** @} */



/** \defgroup hwlocality_helper_ancestors Looking at Ancestor and Child Objects
 * @{
 *
 * Be sure to see the figure in \ref termsanddefs that shows a
 * complete topology tree, including depths, child/sibling/cousin
 * relationships, and an example of an asymmetric topology where one
 * package has fewer caches than its peers.
 */

```cpp

这两段代码是用来获取一个对象在深度为 depth 的层次结构中的祖先对象。

第一段代码定义了一个名为 hwloc_get_ancestor_obj_by_depth 的函数，它接收两个参数：一个 hwloc_topology_t 类型的 topology 对象和 depth 参数。函数内部先定义了一个名为 ancestor 的变量，该变量初始化为传入的 obj 对象，然后通过 while 循环从 depth 0 开始，逐层向上寻找 depth 比 obj 对象的 depth 小于 depth 的祖先对象，并将它们存储在 ancestor 变量中。最后，函数返回祖先对象的 hwloc_obj_t 类型。

第二段代码在函数内部使用 hwloc_topology_t 和 hwloc_obj_t 定义了两个函数，将 topology 和 obj 对象传递给第一个函数，将 depth 和 depth 参数传递给第二个函数。这两个函数在调用时会对传入的参数进行处理，然后返回一个名为 ancestor 的 hwloc_obj_t 类型的对象。


```
/** \brief Returns the ancestor object of \p obj at depth \p depth.
 *
 * \note \p depth should not be the depth of PU or NUMA objects
 * since they are ancestors of no objects (except Misc or I/O).
 * This function rather expects an intermediate level depth,
 * such as the depth of Packages, Cores, or Caches.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_depth (hwloc_topology_t topology __hwloc_attribute_unused, int depth, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_depth (hwloc_topology_t topology __hwloc_attribute_unused, int depth, hwloc_obj_t obj)
{
  hwloc_obj_t ancestor = obj;
  if (obj->depth < depth)
    return NULL;
  while (ancestor && ancestor->depth > depth)
    ancestor = ancestor->parent;
  return ancestor;
}

```cpp

这两函数定义在名为 `hwloc_get_ancestor_obj_by_type` 的函数内。它们的作用是返回一个对象类型为 `type` 的对象的祖先对象，即使 `type` 是 `::HWLOC_OBJ_PU` 或 `::HWLOC_OBJ_NUMANODE`，只要它是一个中间的包或节点对象。

函数首先定义了一个静态函数实参 `topology` 和两个形参 `type` 和 `obj`，其中 `topology` 和 `type` 是输入参数，而 `obj` 是输出参数。函数实现了一个 While 循环，该循环在 `topology` 和 `obj` 对象之间进行逐步向上遍历，直到找到一个名为 `type` 的对象的路径，该对象的类型是 `topology` 的父类型，否则继续遍历。

在函数体中，首先定义了一个名为 `ancestor` 的临时变量，用于存储 `obj` 的祖先对象。然后，通过 `topology` 和 `ancestor` 之间的 `parent` 指针，不断将 `ancestor` 对象指向前面的 `topology` 对象。当遍历完成后，返回 `ancestor` 对象。

这两个函数定义了一种查找对象类型为 `type` 的对象的祖先对象的方法，即使这种对象不是显式的父对象。这种方法在某些情况下很有用，例如当需要在继承关系中查找一个对象的祖先对象时。


```
/** \brief Returns the ancestor object of \p obj with type \p type.
 *
 * \note \p type should not be ::HWLOC_OBJ_PU or ::HWLOC_OBJ_NUMANODE
 * since these objects are ancestors of no objects (except Misc or I/O).
 * This function rather expects an intermediate object type,
 * such as ::HWLOC_OBJ_PACKAGE, ::HWLOC_OBJ_CORE, etc.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_type (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_type_t type, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_type (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_type_t type, hwloc_obj_t obj)
{
  hwloc_obj_t ancestor = obj->parent;
  while (ancestor && ancestor->type != type)
    ancestor = ancestor->parent;
  return ancestor;
}

```cpp

该代码定义了一个名为 `hwloc_get_common_ancestor_obj` 的函数，用于获取两个给定对象的常见祖先对象。该函数返回两个参数中的第一个对象的父对象，如果两个对象都有共同祖先，则返回该共同祖先对象，否则返回第一个对象。

该函数的核心部分是一个循环，该循环遍历两个传入的对象，比较它们在深度上的深度。如果两个对象的深度相同，它们将交换它们在深度上的数据，并将循环条件设置为 `obj1 == obj2`，这将循环一直保持在同一深度上。在循环的过程中，如果两个对象中有一个对象的深度大于另一个对象的深度，则交换它们在深度上的数据，并将循环条件设置为 `obj1 == obj2`。

该函数的实现依赖于传递给它的 `topology` 参数，该参数是一个表示底层物理设备的 `hwloc_topology_t` 类型，以及两个输入参数 `obj1` 和 `obj2`，它们都是 `hwloc_obj_t` 类型。


```
/** \brief Returns the common parent object to objects \p obj1 and \p obj2 */
static __hwloc_inline hwloc_obj_t
hwloc_get_common_ancestor_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj1, hwloc_obj_t obj2) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_common_ancestor_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  /* the loop isn't so easy since intermediate ancestors may have
   * different depth, causing us to alternate between using obj1->parent
   * and obj2->parent. Also, even if at some point we find ancestors of
   * of the same depth, their ancestors may have different depth again.
   */
  while (obj1 != obj2) {
    while (obj1->depth > obj2->depth)
      obj1 = obj1->parent;
    while (obj2->depth > obj1->depth)
      obj2 = obj2->parent;
    if (obj1 != obj2 && obj1->depth == obj2->depth) {
      obj1 = obj1->parent;
      obj2 = obj2->parent;
    }
  }
  return obj1;
}

```cpp

这段代码定义了两个函数，第一个函数`hwloc_obj_is_in_subtree`和第二个函数`hwloc_obj_get_next_child`。

第一个函数`hwloc_obj_is_in_subtree`接收三个参数，一个是`hwloc_topology_t`类型的`topology`，一个是`hwloc_obj_t`类型的`obj`，另一个是`hwloc_obj_t`类型的`subtree_root`。函数返回`true`，如果`obj`在子树中（包含`subtree_root`）并且`topology`和`subtree_root`的`cpuset`包含在`obj`的`cpuset`中，否则返回`false`。

第二个函数`hwloc_obj_get_next_child`与第一个函数类似，但允许在`hwloc_topology_t`和`hwloc_obj_t`之间插入`NULL`。它的接收参数同样包括`topology`、`obj`和`subtree_root`，但第一个参数是`NULL`。函数返回`NULL`，表示当前没有下一个子节点。

这两个函数共同组成了 `hwloc_obj_topology_backend` 实现，用于管理在给定树中的对象，可以用来实现更高级别的并查集操作。


```
/** \brief Returns true if \p obj is inside the subtree beginning with ancestor object \p subtree_root.
 *
 * \note This function cannot work if \p obj and \p subtree_root objects do
 * not have CPU sets (I/O or Misc objects).
 */
static __hwloc_inline int
hwloc_obj_is_in_subtree (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj, hwloc_obj_t subtree_root) __hwloc_attribute_pure;
static __hwloc_inline int
hwloc_obj_is_in_subtree (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj, hwloc_obj_t subtree_root)
{
  return obj->cpuset && subtree_root->cpuset && hwloc_bitmap_isincluded(obj->cpuset, subtree_root->cpuset);
}

/** \brief Return the next child.
 *
 * Return the next child among the normal children list,
 * then among the memory children list, then among the I/O
 * children list, then among the Misc children list.
 *
 * If \p prev is \c NULL, return the first child.
 *
 * Return \c NULL when there is no next child.
 */
```cpp

该函数为 `hwloc_get_next_child` 函数，属于 `hwloc_topology_t` 结构体的成员函数。它的作用是返回 `hwloc_obj_t` 类型的指针，指向其父级的下一个子对象。

函数的实现中，首先定义了一个名为 `obj` 的 `hwloc_obj_t` 类型的变量，用于存储当前要查找的子对象。接着定义了一个名为 `state` 的整型变量，用于跟踪当前所处的查找状态。然后分别对 `prev`、`parent` 和 `topology` 三个参数进行了判断，以确定当前的查找状态。接着，在判断 `prev` 是否为 `hwloc_obj_misc` 时，如果为真，则执行特定的逻辑，否则执行默认逻辑。接着，对 `parent` 和 `topology` 进行了相同的判断，以确定父级和当前顶层的子对象类型。最后，根据 `state` 的值确定下一层子对象的查找方式，并返回找到的子对象。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_next_child (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t parent, hwloc_obj_t prev)
{
  hwloc_obj_t obj;
  int state = 0;
  if (prev) {
    if (prev->type == HWLOC_OBJ_MISC)
      state = 3;
    else if (prev->type == HWLOC_OBJ_BRIDGE || prev->type == HWLOC_OBJ_PCI_DEVICE || prev->type == HWLOC_OBJ_OS_DEVICE)
      state = 2;
    else if (prev->type == HWLOC_OBJ_NUMANODE)
      state = 1;
    obj = prev->next_sibling;
  } else {
    obj = parent->first_child;
  }
  if (!obj && state == 0) {
    obj = parent->memory_first_child;
    state = 1;
  }
  if (!obj && state == 1) {
    obj = parent->io_first_child;
    state = 2;
  }
  if (!obj && state == 2) {
    obj = parent->misc_first_child;
    state = 3;
  }
  return obj;
}

```cpp

这段代码定义了一个名为 `hwlocality_helper_types` 的 `Kinds` 类型，它包含了四种不同的对象类型：Normal、Memory、I/O 和 Misc。这些类型可以使用 `hwloc_obj_type_is_normal()`、`hwloc_obj_type_is_memory()` 和 `hwloc_obj_type_is_io()` 函数来判断是否属于相应的类型。如果属于其中一种或多种类型，则返回真，否则返回假。

由于所有对象类型都只能属于其中一种或几种，因此这个 `Kinds` 类型可以被用来定义对象类型的枚举类型，例如：
```c
typedef enum hwlocality_helper_types {
 Normal,
 Memory,
 I/O,
 Misc
} ObjectType;
```cpp
这个枚举类型包含了四种不同的对象类型，分别对应 `hwlocality_helper_types.Normal`、`hwlocality_helper_types.Memory`、`hwlocality_helper_types.I/O` 和 `hwlocality_helper_types.Misc` 类型的变量可以分别使用 `ObjectType` 类型的变量进行访问。例如，如果你在使用这个枚举类型时需要定义一个对象类型的变量，你可以将其定义为：
```java
ObjectType objType = ObjectType::I/O;
```cpp
这个代码片段定义了一个 `ObjectType` 类型的变量 `objType`，并且它的值为 `ObjectType::I/O`。


```
/** @} */



/** \defgroup hwlocality_helper_types Kinds of object Type
 * @{
 *
 * Each object type is
 * either Normal (i.e. hwloc_obj_type_is_normal() returns 1),
 * or Memory (i.e. hwloc_obj_type_is_memory() returns 1)
 * or I/O (i.e. hwloc_obj_type_is_io() returns 1)
 * or Misc (i.e. equal to ::HWLOC_OBJ_MISC).
 * It cannot be of more than one of these kinds.
 */

```cpp

这段代码定义了两个函数，分别检查一个对象类型是否为“Normal”和“I/O”。

函数名为“hwloc_obj_type_is_normal”，接收一个“hwloc_obj_type_t”类型的参数，返回值为1（真）或0（假）。函数判断一个对象类型是否属于“Normal”类别，即CPU主层 hierarchy中的对象，但不包括NUMA节点、I/O设备或内存设备。

函数名为“hwloc_obj_type_is_i_o”，同样接收一个“hwloc_obj_type_t”类型的参数，但用途是检查对象类型是否属于“I/O”类别。这里的“I/O”指的是所有连接到父母设备上的设备，包括桥接设备、PCI设备（如PCIe设备）、操作系统设备等。


```
/** \brief Check whether an object type is Normal.
 *
 * Normal objects are objects of the main CPU hierarchy
 * (Machine, Package, Core, PU, CPU caches, etc.),
 * but they are not NUMA nodes, I/O devices or Misc objects.
 *
 * They are attached to parent as Normal children,
 * not as Memory, I/O or Misc children.
 *
 * \return 1 if an object of type \p type is a Normal object, 0 otherwise.
 */
HWLOC_DECLSPEC int
hwloc_obj_type_is_normal(hwloc_obj_type_t type);

/** \brief Check whether an object type is I/O.
 *
 * I/O objects are objects attached to their parents
 * in the I/O children list.
 * This current includes Bridges, PCI and OS devices.
 *
 * \return 1 if an object of type \p type is a I/O object, 0 otherwise.
 */
```cpp

这段代码定义了两个名为 `hwloc_obj_type_is_io` 和 `hwloc_obj_type_is_memory` 的函数，用于检查给定的对象类型是否为内存或 CPU 缓存。

函数 `hwloc_obj_type_is_io` 接收一个 `hwloc_obj_type_t` 类型的参数，返回值为 1 如果给定的对象类型是内存，否则返回 0。内存对象包括 NUMA 节点和内存- sidescaches。

函数 `hwloc_obj_type_is_memory` 同样接收一个 `hwloc_obj_type_t` 类型的参数，但返回值为 1 如果给定的对象类型是 CPU 缓存(包括数据缓存、统一缓存和指令缓存)，否则返回 0。这个函数排除了内存- sidescaches 和 NUMA 节点。


```
HWLOC_DECLSPEC int
hwloc_obj_type_is_io(hwloc_obj_type_t type);

/** \brief Check whether an object type is Memory.
 *
 * Memory objects are objects attached to their parents
 * in the Memory children list.
 * This current includes NUMA nodes and Memory-side caches.
 *
 * \return 1 if an object of type \p type is a Memory object, 0 otherwise.
 */
HWLOC_DECLSPEC int
hwloc_obj_type_is_memory(hwloc_obj_type_t type);

/** \brief Check whether an object type is a CPU Cache (Data, Unified or Instruction).
 *
 * Memory-side caches are not CPU caches.
 *
 * \return 1 if an object of type \p type is a Cache, 0 otherwise.
 */
```cpp

这段代码定义了两个名为 `hwloc_obj_type_is_cache` 和 `hwloc_obj_type_is_dcache` 的函数，它们的返回类型为整数类型。函数使用了 `HWLOC_DECLSPEC` 标记，这意味着它们是 Cache-side Cache 函数，而不是用户态函数。

这两个函数的目的是检查给定的对象类型是否为 CPU Data 缓存或 Unified Cache。如果对象类型是一个 CPU Data 缓存或 Unified Cache，函数将返回 1，否则将返回 0。


```
HWLOC_DECLSPEC int
hwloc_obj_type_is_cache(hwloc_obj_type_t type);

/** \brief Check whether an object type is a CPU Data or Unified Cache.
 *
 * Memory-side caches are not CPU caches.
 *
 * \return 1 if an object of type \p type is a CPU Data or Unified Cache, 0 otherwise.
 */
HWLOC_DECLSPEC int
hwloc_obj_type_is_dcache(hwloc_obj_type_t type);

/** \brief Check whether an object type is a CPU Instruction Cache,
 *
 * Memory-side caches are not CPU caches.
 *
 * \return 1 if an object of type \p type is a CPU Instruction Cache, 0 otherwise.
 */
```cpp

这段代码定义了一个名为HWLOC_DECLSPEC的int类型，用于表示缓存对象的类型。接着定义了一个名为hwloc_obj_type_is_icache的函数，该函数接受一个缓存对象的类型参数，并返回该缓存对象的深度。该函数使用了hwloc_get_type_depth函数作为其底层实现，但对其进行了增强，能够处理多种类型的缓存对象。

具体来说，该函数会根据缓存对象的类型，尝试查找与缓存级别和类型相匹配的缓存对象。如果找到了匹配的缓存对象，该函数将返回匹配到的深度。如果找不到匹配的缓存对象，或者缓存对象类型为unified cache，该函数将返回该缓存对象的深度。如果缓存对象类型为invalid，该函数将忽略该缓存对象，并返回multiply作为结果。

该函数的作用是帮助用户查找具有特定缓存级别和类型的缓存对象的深度，从而更好地理解其系统中的缓存情况。


```
HWLOC_DECLSPEC int
hwloc_obj_type_is_icache(hwloc_obj_type_t type);

/** @} */



/** \defgroup hwlocality_helper_find_cache Looking at Cache Objects
 * @{
 */

/** \brief Find the depth of cache objects matching cache level and type.
 *
 * Return the depth of the topology level that contains cache objects
 * whose attributes match \p cachelevel and \p cachetype.

 * This function is identical to calling hwloc_get_type_depth() with the
 * corresponding type such as ::HWLOC_OBJ_L1ICACHE, except that it may
 * also return a Unified cache when looking for an instruction cache.
 *
 * If no cache level matches, ::HWLOC_TYPE_DEPTH_UNKNOWN is returned.
 *
 * If \p cachetype is ::HWLOC_OBJ_CACHE_UNIFIED, the depth of the
 * unique matching unified cache level is returned.
 *
 * If \p cachetype is ::HWLOC_OBJ_CACHE_DATA or ::HWLOC_OBJ_CACHE_INSTRUCTION,
 * either a matching cache, or a unified cache is returned.
 *
 * If \p cachetype is \c -1, it is ignored and multiple levels may
 * match. The function returns either the depth of a uniquely matching
 * level or ::HWLOC_TYPE_DEPTH_MULTIPLE.
 */
```cpp

该函数计算缓存器类型和当前深度之间的最大匹配深度。它通过以下步骤来查找缓存器类型和当前深度之间的最大匹配度：

1. 首先检查缓存器类型是否为 (hwloc_obj_cache_type_t) -1。如果是，函数将返回当前深度，因为两个负数之间的相等关系始终指向深度较小的那个数。
2. 如果缓存器类型不是 (hwloc_obj_cache_type_t) -1，函数将遍历当前深度以下的所有缓存器类型，直到找到与当前深度相匹配的缓存器类型或者找到一个匹配的缓存器类型并返回当前深度。
3. 如果函数在遍历过程中找到一个匹配的缓存器类型（无论是数据缓存还是指令缓存），函数将返回当前深度，因为两个缓存器类型始终指向同一个深度。
4. 如果函数在遍历过程中没有找到匹配的缓存器类型，或者已经遍历了所有深度，函数将返回当前深度，因为该缓存器类型未找到匹配的深度。


```
static __hwloc_inline int
hwloc_get_cache_type_depth (hwloc_topology_t topology,
			    unsigned cachelevel, hwloc_obj_cache_type_t cachetype)
{
  int depth;
  int found = HWLOC_TYPE_DEPTH_UNKNOWN;
  for (depth=0; ; depth++) {
    hwloc_obj_t obj = hwloc_get_obj_by_depth(topology, depth, 0);
    if (!obj)
      break;
    if (!hwloc_obj_type_is_dcache(obj->type) || obj->attr->cache.depth != cachelevel)
      /* doesn't match, try next depth */
      continue;
    if (cachetype == (hwloc_obj_cache_type_t) -1) {
      if (found != HWLOC_TYPE_DEPTH_UNKNOWN) {
	/* second match, return MULTIPLE */
        return HWLOC_TYPE_DEPTH_MULTIPLE;
      }
      /* first match, mark it as found */
      found = depth;
      continue;
    }
    if (obj->attr->cache.type == cachetype || obj->attr->cache.type == HWLOC_OBJ_CACHE_UNIFIED)
      /* exact match (either unified is alone, or we match instruction or data), return immediately */
      return depth;
  }
  /* went to the bottom, return what we found */
  return found;
}

```cpp

这段代码定义了一个名为 `hwloc_get_cache_covering_cpuset` 的函数，用于获取一个 cpuset 对应的第一个数据(或统一)缓存。

函数接受两个参数：顶片类型 `hwloc_topology_t` 和要设置的cpuset `hwloc_const_cpuset_t`。函数返回一个指向缓存对象的 `hwloc_obj_t` 类型的变量，如果缓存不存在，则返回 `NULL`。

函数内部，首先调用 `hwloc_get_obj_covering_cpuset` 函数，该函数将 `topology` 和 `set` 作为参数，返回一个指向以 `set` 为唯一cpuset 的缓存对象的 `hwloc_obj_t` 类型的变量。然后，在内部循环中，如果缓存对象是数据缓存，就返回它，否则递归到下一层。如果最后一层递归完成后仍然没有找到缓存，则返回 `NULL`。


```
/** \brief Get the first data (or unified) cache covering a cpuset \p set
 *
 * \return \c NULL if no cache matches.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_cache_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_cache_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  hwloc_obj_t current = hwloc_get_obj_covering_cpuset(topology, set);
  while (current) {
    if (hwloc_obj_type_is_dcache(current->type))
      return current;
    current = current->parent;
  }
  return NULL;
}

```cpp

这段代码定义了一个名为 `hwloc_get_shared_cache_covering_obj` 的函数，用于获取一个对象与其它的对象之间共享的第一级缓存。

函数接受两个参数：一个 `hwloc_topology_t` 类型的上下文对象 `topology`，和一个 `hwloc_obj_t` 类型的要获取缓存的对象 `obj`。函数内部使用 `hwloc_attribute_pure` 修饰函数，但仍然被允许输出 `NULL` 表示函数没有返回任何值。

函数首先检查对象 `obj` 是否拥有其父对象的 CPU 集。如果是，则直接返回 `obj`。如果不是，则遍历所有父对象，并在找到第一个具有相同 CPU 集的对象时返回该对象。如果在遍历过程中，函数发现了一个与目标对象共享的缓存，则返回该缓存。否则，函数返回 `NULL`。


```
/** \brief Get the first data (or unified) cache shared between an object and somebody else.
 *
 * \return \c NULL if no cache matches or if an invalid object is given.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_shared_cache_covering_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_shared_cache_covering_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj)
{
  hwloc_obj_t current = obj->parent;
  if (!obj->cpuset)
    return NULL;
  while (current) {
    if (!hwloc_bitmap_isequal(current->cpuset, obj->cpuset)
        && hwloc_obj_type_is_dcache(current->type))
      return current;
    current = current->parent;
  }
  return NULL;
}

```cpp

这段代码定义了一个名为 `hwlocality_helper_find_misc` 的函数组，包括用于查找硬件定位物（miscellaneous helpers）的函数。函数组包含两个函数：`remove_multithreading_pus` 和 `remove_puid`。

1. `remove_multithreading_pus` 函数的作用是移除同时包含在 `topology` 对象中的多个 PUI。函数会遍历 `topology` 对象中的每个核心，检查是否包含有多个与该核心 PUI 同时在物理索引中的 PUI。如果是，函数会修改 `topology` 对象中的 PUI，仅保留一个与该核心 PUI 在物理索引中的 PUI。如果不是，函数不会保留任何 PUI。

2. `remove_puid` 函数的作用是移除指定 `which` 参数中大于指定 `p` 且在 `topology` 对象中存在的 PUI。如果 `p` 大于指定 `topology` 对象中存在的 PUI 数量，则该 PUI 和它的所有后代 PUI 都不会被保留。

注意：`topology` 对象在这段代码之外，具体实现可能会涉及其他文件和数据结构，这里仅提供了一个大致的框架。


```
/** @} */



/** \defgroup hwlocality_helper_find_misc Finding objects, miscellaneous helpers
 * @{
 *
 * Be sure to see the figure in \ref termsanddefs that shows a
 * complete topology tree, including depths, child/sibling/cousin
 * relationships, and an example of an asymmetric topology where one
 * package has fewer caches than its peers.
 */

/** \brief Remove simultaneous multithreading PUs from a CPU set.
 *
 * For each core in \p topology, if \p cpuset contains some PUs of that core,
 * modify \p cpuset to only keep a single PU for that core.
 *
 * \p which specifies which PU will be kept.
 * PU are considered in physical index order.
 * If 0, for each core, the function keeps the first PU that was originally set in \p cpuset.
 *
 * If \p which is larger than the number of PUs in a core there were originally set in \p cpuset,
 * no PU is kept for that core.
 *
 * \note PUs that are not below a Core object are ignored
 * (for instance if the topology does not contain any Core object).
 * None of them is removed from \p cpuset.
 */
```cpp

这段代码定义了一个名为 `hwloc_bitmap_singlify_per_core` 的函数，属于 `HWLOC_DECLSPEC` 类。该函数接受两个参数：一个 `hwloc_topology_t` 类型的顶类和一个 `hwloc_bitmap_t` 类型的 `cpu_set` 参数。函数返回一个指向 `HWLOC_OBJ_PU` 类型对象的指针，这个对象表示包含 `os_index` 等于输入的 `os_index` 的 `CPU` 集合。

该函数的作用是将一个 `CPU` 集合转换为 `PU` 对象，其中 `os_index` 参数表示要返回的 CPU 集合的索引。在函数内部，首先定义了一个名为 `hwloc_get_pu_obj_by_os_index` 的函数，也是一个内部函数，这个函数使用 `hwloc_get_next_obj_by_type` 函数获取顶类的 `HWLOC_OBJ_PU` 对象，并使用 `hwloc_bitmap_foreach_begin` 和 `hwloc_bitmap_foreach_end` 函数遍历 CPU 集合，然后找到与输入的 `os_index` 相等的 CPU 集合，最后返回这个 `HWLOC_OBJ_PU` 类型的对象。

该函数在 `hwloc_topology_t` 和 `hwloc_bitmap_t` 类型定义的开源代码中没有提供实现，因此无法提供更多细节。


```
HWLOC_DECLSPEC int hwloc_bitmap_singlify_per_core(hwloc_topology_t topology, hwloc_bitmap_t cpuset, unsigned which);

/** \brief Returns the object of type ::HWLOC_OBJ_PU with \p os_index.
 *
 * This function is useful for converting a CPU set into the PU
 * objects it contains.
 * When retrieving the current binding (e.g. with hwloc_get_cpubind()),
 * one may iterate over the bits of the resulting CPU set with
 * hwloc_bitmap_foreach_begin(), and find the corresponding PUs
 * with this function.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pu_obj_by_os_index(hwloc_topology_t topology, unsigned os_index) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_pu_obj_by_os_index(hwloc_topology_t topology, unsigned os_index)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_PU, obj)) != NULL)
    if (obj->os_index == os_index)
      return obj;
  return NULL;
}

```cpp

这段代码定义了一个名为 `hwloc_get_numanode_obj_by_os_index` 的函数，它接受一个 `hwloc_topology_t` 类型的参数 `topology` 和一个 `unsigned` 类型的参数 `os_index`。

这个函数的作用是返回一个对象类型为 `HWLOC_OBJ_NUMANODE` 的 `hwloc_obj_t` 类型的对象，其中 `os_index` 是传递给函数的操作系统索引。

函数内部首先定义了一个名为 `obj` 的空对象变量，然后使用 `hwloc_get_next_obj_by_type` 函数尝试从 `topology` 中获取一个 `HWLOC_OBJ_NUMANODE` 类型的对象。如果找到这个对象，函数将检查其 `os_index` 是否与 `os_index` 相等。如果是，函数将返回该对象。否则，函数返回一个空对象。

最后，函数将返回一个指向 `NULL` 的指针，表示没有找到匹配 `os_index` 的 `HWLOC_OBJ_NUMANODE` 对象。


```
/** \brief Returns the object of type ::HWLOC_OBJ_NUMANODE with \p os_index.
 *
 * This function is useful for converting a nodeset into the NUMA node
 * objects it contains.
 * When retrieving the current binding (e.g. with hwloc_get_membind() with HWLOC_MEMBIND_BYNODESET),
 * one may iterate over the bits of the resulting nodeset with
 * hwloc_bitmap_foreach_begin(), and find the corresponding NUMA nodes
 * with this function.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_numanode_obj_by_os_index(hwloc_topology_t topology, unsigned os_index) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_numanode_obj_by_os_index(hwloc_topology_t topology, unsigned os_index)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_NUMANODE, obj)) != NULL)
    if (obj->os_index == os_index)
      return obj;
  return NULL;
}

```cpp

这两段代码定义了一个名为 `hwloc_get_closest_objs` 的函数，它的作用是深度优先遍历顶端图，找到与给定源对象 `src` 在同一深度层的所有对象，并将它们按距离排序并返回数量。如果给定的源对象 `src` 是一个 I/O 对象，则该函数将返回 0。此外，该函数还允许在调用时指定对象数量 `max`，以便在遍历时只返回距离源对象最近的 `max` 个对象。

函数的实现包括两个部分：

1. 函数参数部分：定义了三个参数：`topology`，`src` 和 `max`，分别表示顶端图类型、给定源对象和对象数量。还有一个参数 `objs`，用于保存返回的对象列表。

2. 函数实现部分：在函数实现中，首先定义了一个名为 `hwloc_topology_t` 的枚举类型，用于表示顶端图的顶部对象。然后，通过 `hwloc_get_closest_objs` 函数，递归地获取当前对象在给定深度层中的信息，包括其父对象和子对象。最后，将这些信息存储到 `objs` 参数中，并返回参数 `max`。如果给定的源对象 `src` 是一个 I/O 对象，该函数将返回 0。


```
/** \brief Do a depth-first traversal of the topology to find and sort
 *
 * all objects that are at the same depth than \p src.
 * Report in \p objs up to \p max physically closest ones to \p src.
 *
 * \return the number of objects returned in \p objs.
 *
 * \return 0 if \p src is an I/O object.
 *
 * \note This function requires the \p src object to have a CPU set.
 */
/* TODO: rather provide an iterator? Provide a way to know how much should be allocated? By returning the total number of objects instead? */
HWLOC_DECLSPEC unsigned hwloc_get_closest_objs (hwloc_topology_t topology, hwloc_obj_t src, hwloc_obj_t * __hwloc_restrict objs, unsigned max);

/** \brief Find an object below another object, both specified by types and indexes.
 *
 * Start from the top system object and find object of type \p type1
 * and logical index \p idx1.  Then look below this object and find another
 * object of type \p type2 and logical index \p idx2.  Indexes are specified
 * within the parent, not withing the entire system.
 *
 * For instance, if type1 is PACKAGE, idx1 is 2, type2 is CORE and idx2
 * is 3, return the fourth core object below the third package.
 *
 * \note This function requires these objects to have a CPU set.
 */
```cpp

这段代码定义了一个名为 `hwloc_get_obj_below_by_type` 的函数，它接受两个参数：`topology` 和 `type1` 和 `idx1` 和 `type2` 和 `idx2`，它是 `hwloc_topology_t` 和 `hwloc_obj_type_t` 类的成员。

该函数返回一个名为 `obj` 的 `hwloc_obj_t` 类型的对象，如果没有传入 `topology`、`type1`、`idx1` 和 `type2` 和 `idx2` 中的任何一个，函数将返回 `NULL`。

如果 `obj` 对象存在，该函数将尝试从 `topology` 对象的 `cpuset` 属性中获取 `type2` 类型的 `hwloc_obj_t` 对象，并返回它。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_by_type (hwloc_topology_t topology,
			     hwloc_obj_type_t type1, unsigned idx1,
			     hwloc_obj_type_t type2, unsigned idx2) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_by_type (hwloc_topology_t topology,
			     hwloc_obj_type_t type1, unsigned idx1,
			     hwloc_obj_type_t type2, unsigned idx2)
{
  hwloc_obj_t obj;
  obj = hwloc_get_obj_by_type (topology, type1, idx1);
  if (!obj)
    return NULL;
  return hwloc_get_obj_inside_cpuset_by_type(topology, obj->cpuset, type2, idx2);
}

```cpp

这段代码定义了一个名为`find_obj_below_by_type`的函数，它的作用是找到一个链式下标的对象，该对象基于一个指定类型的数组和索引。

具体来说，这个函数需要一个指定数组`typev`和一个指定索引的整数`idxv`。它从系统的最高级对象开始，沿着这两个数组遍历。对于每个数组中的类型和逻辑索引对，它会在之前找到的对象中查找该类型的索引。

例如，如果`nr`是3，`typev`包含`NODE`、`PACKAGE`和`CORE`，而`idxv`包含0、1和2，那么该函数将返回第二个包装程序对象（即第二个NUMA节点）下面的第三个核心对象。

需要注意的是，这个函数需要所有这些对象和根对象具有CPU设置。


```
/** \brief Find an object below a chain of objects specified by types and indexes.
 *
 * This is a generalized version of hwloc_get_obj_below_by_type().
 *
 * Arrays \p typev and \p idxv must contain \p nr types and indexes.
 *
 * Start from the top system object and walk the arrays \p typev and \p idxv.
 * For each type and logical index couple in the arrays, look under the previously found
 * object to find the index-th object of the given type.
 * Indexes are specified within the parent, not withing the entire system.
 *
 * For instance, if nr is 3, typev contains NODE, PACKAGE and CORE,
 * and idxv contains 0, 1 and 2, return the third core object below
 * the second package below the first NUMA node.
 *
 * \note This function requires all these objects and the root object
 * to have a CPU set.
 */
```cpp

这两段代码定义了一个名为`hwloc_get_obj_below_array_by_type`的函数，它接受一个`hwloc_topology_t`类型的顶上`hwloc_obj_type_t`类型变量和一个最大数量`int`类型的整数参数`nr`。

函数首先获取一个根对象`obj`，然后使用一个循环遍历`nr`次。在循环中，函数首先检查`obj`是否为空。如果是，函数返回`NULL`。否则，函数使用`hwloc_get_obj_inside_cpuset_by_type`函数获取`topology`的`cpuset`属性的值，然后使用它调用`hwloc_get_obj_by_type`函数获取指定类型的对象。最后，函数返回找到的`hwloc_obj_t`对象。

这两段代码定义了一个名为`hwloc_get_obj_below_array_by_type`的函数，它接受一个`hwloc_topology_t`类型的顶上`hwloc_obj_type_t`类型变量和一个最大数量`int`类型的整数参数`nr`。函数返回一个指向指定类型的对象的`hwloc_obj_t`对象的指针。


```
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_array_by_type (hwloc_topology_t topology, int nr, hwloc_obj_type_t *typev, unsigned *idxv) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_array_by_type (hwloc_topology_t topology, int nr, hwloc_obj_type_t *typev, unsigned *idxv)
{
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  int i;
  for(i=0; i<nr; i++) {
    if (!obj)
      return NULL;
    obj = hwloc_get_obj_inside_cpuset_by_type(topology, obj->cpuset, typev[i], idxv[i]);
  }
  return obj;
}

```cpp

这段代码定义了一个名为 `get_device` 的函数，其作用是返回一个与输入参数 `src` 具有相同布局（即相同 CPU 和节点设置）的异质类型对象。函数的具体实现根据输入参数 `src` 的类型进行不同的处理。

如果 `src` 是一个正常或内存类型，函数将返回一个与 `src` 具有相同布局的异质类型对象。如果 `src` 是一个 PCI 或一个在 PCI 设备内的 OS 设备，函数将返回一个与 `src` 具有相同布局的 OS 设备，或者返回一个 PCI 设备（如果 `src` 是一个 OS 设备）。

如果 `src` 是一个 CUDA 或 OpenCL 设备，函数将根据 `src` 的设备类型选择相应的 OS 设备或 CUDA/OpenCL 设备。如果 `src` 是一个 PCI 设备，函数将返回该 PCI 设备。

如果 `src` 是一个普通对象，函数将直接返回一个与 `src` 具有相同布局的异质类型对象。

函数的参数包括两个：`src` 参数和一个选择参数 `subtype`，用于指定返回的设备类型子集。另一个参数是一个字符串 `nameprefix`，用于指定返回设备名称前缀。

如果多个设备与 `src` 匹配，函数将返回第一个匹配的设备。函数不会跨越桥接，也不能将 CPU 和 I/O 设备之间进行转换。


```
/** \brief Return an object of a different type with same locality.
 *
 * If the source object \p src is a normal or memory type,
 * this function returns an object of type \p type with same
 * CPU and node sets, either below or above in the hierarchy.
 *
 * If the source object \p src is a PCI or an OS device within a PCI
 * device, the function may either return that PCI device, or another
 * OS device in the same PCI parent.
 * This may for instance be useful for converting between OS devices
 * such as "nvml0" or "rsmi1" used in distance structures into the
 * the PCI device, or the CUDA or OpenCL OS device that correspond
 * to the same physical card.
 *
 * If not \c NULL, parameter \p subtype only select objects whose
 * subtype attribute exists and is \p subtype (case-insensitively),
 * for instance "OpenCL" or "CUDA".
 *
 * If not \c NULL, parameter \p nameprefix only selects objects whose
 * name attribute exists and starts with \p nameprefix (case-insensitively),
 * for instance "rsmi" for matching "rsmi0".
 *
 * If multiple objects match, the first one is returned.
 *
 * This function will not walk the hierarchy across bridges since
 * the PCI locality may become different.
 * This function cannot also convert between normal/memory objects
 * and I/O or Misc objects.
 *
 * \p flags must be \c 0 for now.
 *
 * \return An object with identical locality,
 * matching \p subtype and \p nameprefix if any.
 *
 * \return \c NULL if no matching object could be found,
 * or if the source object and target type are incompatible,
 * for instance if converting between CPU and I/O objects.
 */
```cpp

这段代码定义了一个名为 `hwlocality_helper_distribute` 的函数，属于 `hwlocality_helper` 组。这个函数的作用是分布具有相同局部性的 `hwloc_obj_t` 对象，它将 `hwloc_topology_t`、`hwloc_obj_t`、`hwloc_obj_type_t` 和一个子类型 `const char *` 作为输入参数，并将输入的 `hwloc_obj_t` 的类型设置为 `hwloc_obj_type_t_FLAG`，同时传递给 `hwloc_distrib` 的标志有 `__EXCEPTION_技术创新` 和 `__EXCEPTION_OVERLOAD`。


```
HWLOC_DECLSPEC hwloc_obj_t
hwloc_get_obj_with_same_locality(hwloc_topology_t topology, hwloc_obj_t src,
                                 hwloc_obj_type_t type, const char *subtype, const char *nameprefix,
                                 unsigned long flags);

/** @} */



/** \defgroup hwlocality_helper_distribute Distributing items over a topology
 * @{
 */

/** \brief Flags to be given to hwloc_distrib().
 */
```cpp

这段代码定义了一个枚举类型 hwloc_distrib_flags_e，其中包括了两个成员变量：HWLOC_DISTRIB_FLAG_REVERSE 和 ```std::vector<hwloc_distrib_flags_e>`类型的 obj。函数原型定义了一个枚举类型，其值为 HWLOC_DISTRIB_FLAG_REVERSE，同时定义了一个函数 `distribute_n<T>`，其参数包括 `T` 类型的变量 `n` 和 `std::vector<hwloc_distrib_flags_e>`类型的变量 `obj`。函数 `distribute_n<T>` 的实现包括对 `obj` 进行递归遍历，将 `T` 类型的每个元素 `i` 和 `hwloc_distrib_flags_e` 的 `i` 值合并，最后输出合并后的 `hwloc_distrib_flags_e` 类型的变量。


```cpp
enum hwloc_distrib_flags_e {
  /** \brief Distrib in reverse order, starting from the last objects.
   * \hideinitializer
   */
  HWLOC_DISTRIB_FLAG_REVERSE = (1UL<<0)
};

/** \brief Distribute \p n items over the topology under \p roots
 *
 * Array \p set will be filled with \p n cpusets recursively distributed
 * linearly over the topology under objects \p roots, down to depth \p until
 * (which can be INT_MAX to distribute down to the finest level).
 *
 * \p n_roots is usually 1 and \p roots only contains the topology root object
 * so as to distribute over the entire topology.
 *
 * This is typically useful when an application wants to distribute \p n
 * threads over a machine, giving each of them as much private cache as
 * possible and keeping them locally in number order.
 *
 * The caller may typically want to also call hwloc_bitmap_singlify()
 * before binding a thread so that it does not move at all.
 *
 * \p flags should be 0 or a OR'ed set of ::hwloc_distrib_flags_e.
 *
 * \note This function requires the \p roots objects to have a CPU set.
 */
```



This function is a part of the Linux kernel's hwloc module, which is used to allocate hardware resources like CPUs and memory domains. The function takes in a root node with a specified flags and creates a node that is then added to the root object.

The function starts by setting up a hierarchy of nodes, starting from the root node, and then calculates the weight of each node based on the number of CPUs and the amount of memory it requires. The weight is then assigned to the root node.

If the root node is a simple memory node, the function calculates the chunk of memory to give to the root node, and分配s that chunk to the root node.

If the root node is a CPU node, the function starts by giving the root node a chunk of memory proportional to its weight, and then recursively allocating memory chunks to the children nodes until the entire tree is filled.

The function uses various flags to specify the behavior of the allocation process, such as whether to distribute the memory in chunks or not, and whether to consider the parent node when allocating memory.

The function returns 0 if the allocation was successful, or an error code if there was an issue with the allocation process.


```cpp
static __hwloc_inline int
hwloc_distrib(hwloc_topology_t topology,
	      hwloc_obj_t *roots, unsigned n_roots,
	      hwloc_cpuset_t *set,
	      unsigned n,
	      int until, unsigned long flags)
{
  unsigned i;
  unsigned tot_weight;
  unsigned given, givenweight;
  hwloc_cpuset_t *cpusetp = set;

  if (flags & ~HWLOC_DISTRIB_FLAG_REVERSE) {
    errno = EINVAL;
    return -1;
  }

  tot_weight = 0;
  for (i = 0; i < n_roots; i++)
    tot_weight += (unsigned) hwloc_bitmap_weight(roots[i]->cpuset);

  for (i = 0, given = 0, givenweight = 0; i < n_roots; i++) {
    unsigned chunk, weight;
    hwloc_obj_t root = roots[flags & HWLOC_DISTRIB_FLAG_REVERSE ? n_roots-1-i : i];
    hwloc_cpuset_t cpuset = root->cpuset;
    while (!hwloc_obj_type_is_normal(root->type))
      /* If memory/io/misc, walk up to normal parent */
      root = root->parent;
    weight = (unsigned) hwloc_bitmap_weight(cpuset);
    if (!weight)
      continue;
    /* Give to root a chunk proportional to its weight.
     * If previous chunks got rounded-up, we may get a bit less. */
    chunk = (( (givenweight+weight) * n  + tot_weight-1) / tot_weight)
          - ((  givenweight         * n  + tot_weight-1) / tot_weight);
    if (!root->arity || chunk <= 1 || root->depth >= until) {
      /* We can't split any more, put everything there.  */
      if (chunk) {
	/* Fill cpusets with ours */
	unsigned j;
	for (j=0; j < chunk; j++)
	  cpusetp[j] = hwloc_bitmap_dup(cpuset);
      } else {
	/* We got no chunk, just merge our cpuset to a previous one
	 * (the first chunk cannot be empty)
	 * so that this root doesn't get ignored.
	 */
	assert(given);
	hwloc_bitmap_or(cpusetp[-1], cpusetp[-1], cpuset);
      }
    } else {
      /* Still more to distribute, recurse into children */
      hwloc_distrib(topology, root->children, root->arity, cpusetp, chunk, until, flags);
    }
    cpusetp += chunk;
    given += chunk;
    givenweight += weight;
  }

  return 0;
}

```

这段代码定义了一个名为`hwlocality_helper_topology_sets`的函数组，用于获取整个系统的CPU和节点设置。该函数组包含两个函数，分别为`get_cpu_set`和`get_complete_cpu_set`。函数组的说明中包含如下描述：

"Get complete CPU set

This function returns the complete CPU set of processors of the system."

"The returned cpuset is not newly allocated and should thus not be changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy."

"This is equivalent to retrieving the root object complete CPU-set."

因此，这个函数组的作用是用于获取整个系统的CPU和节点设置，并返回一个完整的CPU设置，但这个设置不会被修改或者被释放。如果需要获取根对象的CPU设置，可以使用`hwloc_bitmap_dup()`函数来获取一个本地 copy。


```cpp
/** @} */



/** \defgroup hwlocality_helper_topology_sets CPU and node sets of entire topologies
 * @{
 */

/** \brief Get complete CPU set
 *
 * \return the complete CPU set of processors of the system.
 *
 * \note The returned cpuset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * \note This is equivalent to retrieving the root object complete CPU-set.
 */
```

这段代码定义了一个名为 `hwloc_const_cpuset_t` 的变量，它是一个 `hwloc_topology_t` 类型的 `HWLOC_DECLSPEC` 修饰的 `hwloc_const_cpuset_t` 类型。它的作用是获取一个系统中的所有处理器（也就是 CPU 集合）的集合，这个集合是由 `hwloc_topology_get_complete_cpuset` 函数计算出来的。

由于这个函数返回的集合不是 newly allocated 的，因此可以通过调用 `hwloc_bitmap_dup` 函数来获取一个 local copy。同时，这个函数也确保了这个集合不会被修改或释放，因此可以安全地使用它来获取系统对象的 CPU 集合。


```cpp
HWLOC_DECLSPEC hwloc_const_cpuset_t
hwloc_topology_get_complete_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief Get topology CPU set
 *
 * \return the CPU set of processors of the system for which hwloc
 * provides topology information. This is equivalent to the cpuset of the
 * system object.
 *
 * \note The returned cpuset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * \note This is equivalent to retrieving the root object CPU-set.
 */
HWLOC_DECLSPEC hwloc_const_cpuset_t
```

这段代码是一个C语言函数，名为`hwloc_topology_get_topology_cpuset`，属于`hwloc_topology_t`类型的`hwloc_attribute_pure`修饰。它返回系统允许的CPU设置。

函数首先检查一个名为`topology`的topology结构体是否设置了`::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED`标志。如果是，则函数的实现与`hwloc_topology_get_topology_cpuset()`相同，即允许所有PU。否则，函数尝试通过`hwloc_bitmap_intersects()`函数来获取当前CPU集，并检查其是否包含被禁用的CPU。如果是，则函数返回允许的CPU设置；否则，使用`hwloc_bitmap_and()`函数获取允许的CPU设置。最后，函数返回一个`hwloc_cpuset`类型的结果，这个结果不应该被修改或释放，应该使用`hwloc_bitmap_dup()`获取一个本地副本。


```cpp
hwloc_topology_get_topology_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief Get allowed CPU set
 *
 * \return the CPU set of allowed processors of the system.
 *
 * \note If the topology flag ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was not set,
 * this is identical to hwloc_topology_get_topology_cpuset(), which means
 * all PUs are allowed.
 *
 * \note If ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was set, applying
 * hwloc_bitmap_intersects() on the result of this function and on an object
 * cpuset checks whether there are allowed PUs inside that object.
 * Applying hwloc_bitmap_and() returns the list of these allowed PUs.
 *
 * \note The returned cpuset is not newly allocated and should thus not be
 * changed or freed, hwloc_bitmap_dup() must be used to obtain a local copy.
 */
```

这两行代码是用来获取系统的完整节点集的。这个函数是`hwloc_topology_get_allowed_cpuset`的别体，而`hwloc_topology_get_complete_nodeset`则是它的导出函数。

这两行代码的作用是获取指定拓扑结构中所有允许的节点集，然后将该节点集返回。这个函数可以在需要时动态地获取系统中的所有节点，而不需要在函数内部保存或复制节点。

注意，这个函数没有实现。它只是一个头文件，需要在实现时进行定义和调用的封装。


```cpp
HWLOC_DECLSPEC hwloc_const_cpuset_t
hwloc_topology_get_allowed_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief Get complete node set
 *
 * \return the complete node set of memory of the system.
 *
 * \note The returned nodeset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * \note This is equivalent to retrieving the root object complete nodeset.
 */
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_complete_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

```

这两段代码是用来获取系统的拓扑节点集合的。第一段代码定义了一个名为 `hwloc_topology_get_topology_nodeset` 的函数，它接受一个 `hwloc_topology_t` 的参数，这个参数表示系统的拓扑结构。函数返回的是系统的内存中所有允许的节点集合，也就是系统对象的节点集合。如果 `hwloc_topology_topology_get_topology_nodeset` 这个函数被调用，并且 `::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED` 没有被设置，那么函数将返回系统对象中所有允许的节点集合。如果 `::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED` 被设置，那么函数将返回系统对象中所有允许的节点集合，并且将首先使用 `hwloc_bitmap_intersects()` 函数检查返回的节点集合中是否有允许的节点，如果有，则使用 `hwloc_bitmap_and()` 函数返回允许的节点集合。最后，函数返回允许的节点集合。


```cpp
/** \brief Get topology node set
 *
 * \return the node set of memory of the system for which hwloc
 * provides topology information. This is equivalent to the nodeset of the
 * system object.
 *
 * \note The returned nodeset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * \note This is equivalent to retrieving the root object nodeset.
 */
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_topology_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief Get allowed node set
 *
 * \return the node set of allowed memory of the system.
 *
 * \note If the topology flag ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was not set,
 * this is identical to hwloc_topology_get_topology_nodeset(), which means
 * all NUMA nodes are allowed.
 *
 * \note If ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was set, applying
 * hwloc_bitmap_intersects() on the result of this function and on an object
 * nodeset checks whether there are allowed NUMA nodes inside that object.
 * Applying hwloc_bitmap_and() returns the list of these allowed NUMA nodes.
 *
 * \note The returned nodeset is not newly allocated and should thus not be
 * changed or freed, hwloc_bitmap_dup() must be used to obtain a local copy.
 */
```

这段代码定义了一个名为 `hwlocality_helper_nodeset_convert` 的函数，它用于将 CPU 集转换为 NUMA 节点集。

函数接受一个 `hwloc_topology_t` 的输入，表示要转换的顶级拓扑结构，然后返回一个 `hwloc_const_nodeset_t` 的输出，表示转换后的节点集。

函数的核心部分是在 `hwloc_topology_get_allowed_nodeset` 函数中实现的，这个函数接收一个 `hwloc_topology_t` 的输入，返回允许节点的最小集合。通过遍历输入的 CPU 集，函数会检查每个节点是否存在本地 CPU，如果存在，则将该节点添加到输出节点集中。如果输入的 CPU 集中有节点没有本地 CPU，函数不会设置该节点在输出节点集中的索引。

整个函数的目的是将输入的顶级拓扑结构转换为包含所有有本地 CPU 的 NUMA 节点集，然后返回该节点集。


```cpp
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_allowed_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** @} */



/** \defgroup hwlocality_helper_nodeset_convert Converting between CPU sets and node sets
 *
 * @{
 */

/** \brief Convert a CPU set into a NUMA node set
 *
 * For each PU included in the input \p _cpuset, set the corresponding
 * local NUMA node(s) in the output \p nodeset.
 *
 * If some NUMA nodes have no CPUs at all, this function never sets their
 * indexes in the output node set, even if a full CPU set is given in input.
 *
 * Hence the entire topology CPU set is converted into the set of all nodes
 * that have some local CPUs.
 */
```

这段代码的作用是实现将一个NUMA节点集转换为CPU集的功能。具体来说，它会遍历输入节点集中的每个节点，并检查其是否有本地NUMA节点。如果有，它会将该节点对应的本地CPU加入到输出CPU集中。在函数内部，首先获取输入节点集的深度，然后使用一个位图来记录每个节点的本地CPU。接下来，使用一个循环遍历每个节点，并检查其对应的本地CPU是否存在于输出CPU集中。如果是，就返回0；如果不是，就返回-1。最后，如果所有节点都没有本地CPU，则函数不会对输出做任何处理。


```cpp
static __hwloc_inline int
hwloc_cpuset_to_nodeset(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset)
{
	int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
	hwloc_obj_t obj = NULL;
	assert(depth != HWLOC_TYPE_DEPTH_UNKNOWN);
	hwloc_bitmap_zero(nodeset);
	while ((obj = hwloc_get_next_obj_covering_cpuset_by_depth(topology, _cpuset, depth, obj)) != NULL)
		if (hwloc_bitmap_set(nodeset, obj->os_index) < 0)
			return -1;
	return 0;
}

/** \brief Convert a NUMA node set into a CPU set
 *
 * For each NUMA node included in the input \p nodeset, set the corresponding
 * local PUs in the output \p _cpuset.
 *
 * If some CPUs have no local NUMA nodes, this function never sets their
 * indexes in the output CPU set, even if a full node set is given in input.
 *
 * Hence the entire topology node set is converted into the set of all CPUs
 * that have some local NUMA nodes.
 */
```

这段代码定义了一个名为 `hwloc_cpuset_from_nodeset` 的函数，它是 `hwloc_topology_t` 和 `hwloc_cpuset_t` 类的静态成员函数。

该函数的作用是：从指定的 `nodeset` 中选择所有属于 `_cpuset` 的 CPU 设置，使得选出的 CPU 设置与 `_cpuset` 中的所有元素都或操作后，剩余的元素都为 0。然后返回选出的 CPU 设置的数量。

代码中首先定义了一个名为 `depth` 的整数变量，用于获取 topology 对象中指定节点序列的深度。接着定义了一个名为 `obj` 的空指针变量，用于存储当前正在遍历的节点序列中的下一个节点。然后定义了一个名为 `_cpuset` 的 bitmap 变量，用于存储当前 CPU 设置。接着使用 while 循环，从 topology 对象中获取一个深度为 `depth` 的节点序列，并将其存储到 `_cpuset` 中。然后使用 another while 循环，遍历 `_cpuset` 中的所有元素，检查当前节点序列是否与 `nodeset` 中的元素相同。如果是，则说明当前节点序列中已经存在与 `nodeset` 中的元素相同的 CPU 设置，因此不需要再检查当前节点序列中的 CPU 设置，直接返回 0。如果当前节点序列与 `nodeset` 中的元素相同，但是已经遍历完 `_cpuset` 中的所有元素，那么就需要检查当前节点序列中的 CPU 设置是否与 `nodeset` 中的元素相同。如果相同，则说明当前节点序列中的 CPU 设置与 `nodeset` 中的元素相同，返回 0。


```cpp
static __hwloc_inline int
hwloc_cpuset_from_nodeset(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset)
{
	int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
	hwloc_obj_t obj = NULL;
	assert(depth != HWLOC_TYPE_DEPTH_UNKNOWN);
	hwloc_bitmap_zero(_cpuset);
	while ((obj = hwloc_get_next_obj_by_depth(topology, depth, obj)) != NULL) {
		if (hwloc_bitmap_isset(nodeset, obj->os_index))
			/* no need to check obj->cpuset because objects in levels always have a cpuset */
			if (hwloc_bitmap_or(_cpuset, _cpuset, obj->cpuset) < 0)
				return -1;
	}
	return 0;
}

```

这段代码定义了一个名为 `hwlocality_advanced_io` 的头文件，其中包含了一些与 I/O 相关的内容。

在该头文件中，定义了一个名为 `hwlocality_find_iio_ancestor` 的函数，该函数接受一个名为 `ioobj` 的 I/O 对象作为参数，并返回一个指向其最接近的非 I/O 祖先对象的指针。

函数的实现包括以下几个步骤：

1. 定义了一个名为 `hwlocality_find_iio_ancestor` 的头文件，其中包含了一个名为 `find_iio_ancestor` 的函数，该函数接受一个名为 `ioobj` 的 I/O 对象作为参数，并返回一个指向其最接近的非 I/O 祖先对象的指针。

2. 在 `find_iio_ancestor` 函数中，定义了一个名为 `hwlocality_ancestor` 的变量，用于存储输入参数 `ioobj` 中的对象的最小非 I/O 祖先对象。

3. 在 `find_iio_ancestor` 函数中，定义了一个名为 `hwlocality_normal_iio` 的变量，用于存储找到的最小非 I/O 祖先对象，该对象可以是正常的 I/O 对象或内存对象。

4. 在 `hwlocality_find_iio_ancestor` 函数中，由于已经找到了最小非 I/O 祖先对象，所以可以返回该对象的指针。

5. 在 `hwlocality_normal_iio` 的定义中，说明它可以是正常的 I/O 对象或内存对象，因此可以用于绑定的目的，因为它的 `cpu` 和 `node` 设置与输入参数 `ioobj` 相同，而 `locality` 设置也相同。


```cpp
/** @} */



/** \defgroup hwlocality_advanced_io Finding I/O objects
 * @{
 */

/** \brief Get the first non-I/O ancestor object.
 *
 * Given the I/O object \p ioobj, find the smallest non-I/O ancestor
 * object. This object (normal or memory) may then be used for binding
 * because it has non-NULL CPU and node sets
 * and because its locality is the same as \p ioobj.
 *
 * \note The resulting object is usually a normal object but it could also
 * be a memory object (e.g. NUMA node) in future platforms if I/O objects
 * ever get attached to memory instead of CPUs.
 */
```

这段代码定义了一个名为 `hwloc_get_non_io_ancestor_obj` 的函数，它接受两个参数：`topology` 和 `ioobj`，然后返回一个名为 `obj` 的 `hwloc_obj_t` 类型的变量。函数的具体实现如下：

1. 首先，将 `ioobj` 赋值给参数 `obj`，因为 `ioobj` 也被认为是 `hwloc_obj_t` 类型的变量。
2. 接下来，进入一个循环，只要 `obj` 存在并且 `obj` 的 `cpuset` 字段为 `NULL`，就继续执行循环操作。
3. 在循环中，将 `obj` 指向其父对象，因为 `obj` 的父对象也可能是 `hwloc_topology_t` 类型的变量。
4. 循环结束后，返回 `obj`，它现在代表了一个 `hwloc_obj_t` 类型的对象，代表了一个 `PCI` 设备。

函数的实现主要作用是为了解决在 `hwloc_topology_t` 类型的拓扑结构中，如何根据 `ioobj` 的类型找到系统的第一个 `PCI` 设备。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_get_non_io_ancestor_obj(hwloc_topology_t topology __hwloc_attribute_unused,
			      hwloc_obj_t ioobj)
{
  hwloc_obj_t obj = ioobj;
  while (obj && !obj->cpuset) {
    obj = obj->parent;
  }
  return obj;
}

/** \brief Get the next PCI device in the system.
 *
 * \return the first PCI device if \p prev is \c NULL.
 */
```

这两段代码定义了两个函数，名为`hwloc_get_next_pcidev`和`hwloc_get_pcidev_by_busid`，它们都接受一个`hwloc_topology_t`类型的参数，以及一个或多个`hwloc_obj_t`类型的参数。

`hwloc_get_next_pcidev`函数接收两个`hwloc_obj_t`类型的参数，然后使用`hwloc_get_next_obj_by_type`函数，在给定的拓扑结构中查找具有指定类型（`HWLOC_OBJ_PCI_DEVICE`）的`hwloc_obj_t`对象。如果找到了匹配的`hwloc_obj_t`，函数将返回该对象。如果没有找到匹配的`hwloc_obj_t`，函数将返回`NULL`。

`hwloc_get_pcidev_by_busid`函数也接受一个`hwloc_topology_t`类型的参数，以及一个或多个`hwloc_obj_t`类型的参数。函数使用`hwloc_get_next_pcidev`函数查找具有指定PCI总线ID的`hwloc_obj_t`对象。如果找到了匹配的`hwloc_obj_t`，函数将返回该对象。如果没有找到匹配的`hwloc_obj_t`，函数将返回`NULL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_get_next_pcidev(hwloc_topology_t topology, hwloc_obj_t prev)
{
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_PCI_DEVICE, prev);
}

/** \brief Find the PCI device object matching the PCI bus id
 * given domain, bus device and function PCI bus id.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pcidev_by_busid(hwloc_topology_t topology,
			  unsigned domain, unsigned bus, unsigned dev, unsigned func)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_pcidev(topology, obj)) != NULL) {
    if (obj->attr->pcidev.domain == domain
	&& obj->attr->pcidev.bus == bus
	&& obj->attr->pcidev.dev == dev
	&& obj->attr->pcidev.func == func)
      return obj;
  }
  return NULL;
}

```

这段代码定义了一个名为 `hwloc_get_pcidev_by_busidstring` 的函数，用于根据传入的 PCI 总线 ID 字符串查找与该 ID 相匹配的 PCI 设备对象。函数的实现包括以下几个步骤：

1. 初始化变量：将 `domain` 变量初始化为 0，用于存储默认的 PCI 设备对象；将 `bus`、`dev` 和 `func` 变量分别用于存储输入的 PCI 总线 ID 的三个部分。

2. 解析输入：使用 `sscanf` 函数将输入的 PCI 总线 ID 字符串转换为三个整数，分别存储为 `bus`、`dev` 和 `func` 变量。如果转换失败或者输入的 ID 不是三个数字，函数将返回 `NULL`，以避免出错。

3. 查找匹配的设备对象：使用 `hwloc_get_pcidev` 函数，将输入的 PCI 总线 ID 转换为相应的领域、总线和功能，然后传递给 `hwloc_get_pcidev_by_busid` 函数进行查找。如果找到匹配的设备对象，将返回该设备对象。

4. 返回结果：如果查找成功，函数返回与输入 ID 相匹配的 PCI 设备对象；如果查找失败，函数返回 `NULL`。


```cpp
/** \brief Find the PCI device object matching the PCI bus id
 * given as a string xxxx:yy:zz.t or yy:zz.t.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pcidev_by_busidstring(hwloc_topology_t topology, const char *busid)
{
  unsigned domain = 0; /* default */
  unsigned bus, dev, func;

  if (sscanf(busid, "%x:%x.%x", &bus, &dev, &func) != 3
      && sscanf(busid, "%x:%x:%x.%x", &domain, &bus, &dev, &func) != 4) {
    errno = EINVAL;
    return NULL;
  }

  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, func);
}

```

hwloc_get_next_bridge(hwloc_topology_t topology, hwloc_obj_t prev)
{
return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_BRIDGE, prev);
}


```cpp
/** \brief Get the next OS device in the system.
 *
 * \return the first OS device if \p prev is \c NULL.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_osdev(hwloc_topology_t topology, hwloc_obj_t prev)
{
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_OS_DEVICE, prev);
}

/** \brief Get the next bridge in the system.
 *
 * \return the first bridge if \p prev is \c NULL.
 */
static __hwloc_inline hwloc_obj_t
```

该代码是一个 C 语言函数，名为 `hwloc_get_next_bridge`，属于 `hwloc_topology_t` 和 `hwloc_obj_t` 类型的数据成员。

函数的作用是返回一个 `hwloc_obj_t` 类型的下一个桥，它可以通过以下步骤获取：

1. 通过 `hwloc_topology_t` 类型的 `topology` 参数，获取一个桥的当前位置。
2. 通过 `hwloc_get_next_obj_by_type` 函数，获取桥对象，该函数根据要获取的类型从当前位置的所有对象中获取下一个对象。
3. 通过 `hwloc_bridge_covers_pcibus` 函数，检查给定的桥是否覆盖了给定的 PCI 总线。该函数需要以下参数：桥对象，当前域（unsigned domain）和总线号（unsigned bus）。
4. 返回 `true`，如果桥对象覆盖了给定的 PCI 总线，否则返回 `false`。

该函数可以用于在 `hwloc_topology_t` 类型的数据结构中获取桥对象，并检查它们是否覆盖了给定的 PCI 总线。


```cpp
hwloc_get_next_bridge(hwloc_topology_t topology, hwloc_obj_t prev)
{
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_BRIDGE, prev);
}

/* \brief Checks whether a given bridge covers a given PCI bus.
 */
static __hwloc_inline int
hwloc_bridge_covers_pcibus(hwloc_obj_t bridge,
			   unsigned domain, unsigned bus)
{
  return bridge->type == HWLOC_OBJ_BRIDGE
    && bridge->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI
    && bridge->attr->bridge.downstream.pci.domain == domain
    && bridge->attr->bridge.downstream.pci.secondary_bus <= bus
    && bridge->attr->bridge.downstream.pci.subordinate_bus >= bus;
}

```

这段代码是一个C语言的预处理指令，它主要作用是在编译之前对代码进行处理。具体来说，它包括以下几个部分：

1. `/** @}`：这是一个表示作用于当前声明的所有外部声明的结束的标识符。这个标识符告诉编译器不要在此处继续添加新的声明。

2. `#ifdef __cplusplus`：这是一个条件编译语句，用于检查是否支持C++ Plus的编译选项。如果这个条件为真，那么编译器会开启C++ Plus的特性，包括编译历史的支持。

3. `#endif`：这是一个条件编译语句，用于检查是否支持某个标识符。这个标识符可以是任何C或C++标准中的标识符，如`__attribute__((__ overload__))`。如果这个标识符为真，那么编译器会执行相应的方法或设置相应的选项。

4. `#include "hwloc_helper.h"`：这是一个预处理指令，用于包含C或C++标准中定义的`hwloc_helper.h`头文件。这个头文件可能包含一些预定义的宏和函数，以帮助开发人员更轻松地使用C或C++的HAL库。

总的来说，这段代码的作用是帮助开发人员更轻松地使用C或C++的HAL库，并提供了一些预处理指令以简化代码的编译过程。


```cpp
/** @} */



#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_HELPER_H */

```