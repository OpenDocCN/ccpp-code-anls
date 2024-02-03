# `xmrig\src\3rdparty\hwloc\include\private\xml.h`

```cpp
/*
 * 版权所有 © 2009-2017 Inria。
 * 保留所有权利。请参阅顶层目录中的 COPYING 文件。
 */

#ifndef PRIVATE_XML_H
#define PRIVATE_XML_H 1

#include "hwloc.h"

#include <sys/types.h>

// 声明函数 hwloc__xml_verbose，返回值为 int
HWLOC_DECLSPEC int hwloc__xml_verbose(void);

/**************
 * XML 导入 *
 **************/

// 定义结构 hwloc__xml_import_state_s
typedef struct hwloc__xml_import_state_s {
  struct hwloc__xml_import_state_s *parent; // 指向父状态的指针

  // 全局变量，在整个导入过程中共享
  struct hwloc_xml_backend_data_s *global;

  // 用于存储后端特定数据的不透明数据。
  // 静态分配以允许常见代码进行堆栈分配，而不知道实际后端的需求。
  char data[32];
} * hwloc__xml_import_state_t; // 定义指向结构的指针类型

// 定义结构 hwloc__xml_imported_v1distances_s
struct hwloc__xml_imported_v1distances_s {
  unsigned long kind; // 类型为 unsigned long 的成员变量
  unsigned nbobjs; // 类型为 unsigned 的成员变量
  float *floats; // 指向 float 类型的指针
  struct hwloc__xml_imported_v1distances_s *prev, *next; // 指向前一个和后一个结构的指针
};

// 声明函数 hwloc__xml_import_diff，参数为 state 和 firstdiffp，返回值为 int
HWLOC_DECLSPEC int hwloc__xml_import_diff(hwloc__xml_import_state_t state, hwloc_topology_diff_t *firstdiffp);
# 定义了一个结构体 hwloc_xml_backend_data_s，用于存储 XML 后端的数据
struct hwloc_xml_backend_data_s {
  /* xml 后端参数 */

  # 初始化函数指针，用于初始化 XML 后端数据和导入状态
  int (*look_init)(struct hwloc_xml_backend_data_s *bdata, struct hwloc__xml_import_state_s *state);

  # 结束函数指针，用于结束 XML 后端数据和导入状态
  void (*look_done)(struct hwloc_xml_backend_data_s *bdata, int result);

  # 后端退出函数指针，用于退出 XML 后端数据
  void (*backend_exit)(struct hwloc_xml_backend_data_s *bdata);

  # 下一个属性函数指针，用于获取下一个属性的名称和值
  int (*next_attr)(struct hwloc__xml_import_state_s * state, char **namep, char **valuep);

  # 查找子节点函数指针，用于查找子节点的导入状态和标签
  int (*find_child)(struct hwloc__xml_import_state_s * state, struct hwloc__xml_import_state_s * childstate, char **tagp);

  # 关闭标签函数指针，用于查找显式的关闭标签 </name>
  int (*close_tag)(struct hwloc__xml_import_state_s * state);

  # 关闭子节点函数指针，用于关闭子节点的导入状态
  void (*close_child)(struct hwloc__xml_import_state_s * state);

  # 获取内容函数指针，用于获取导入状态的内容
  int (*get_content)(struct hwloc__xml_import_state_s * state, const char **beginp, size_t expected_length);

  # 关闭内容函数指针，用于关闭导入状态的内容
  void (*close_content)(struct hwloc__xml_import_state_s * state);

  # 消息前缀
  char * msgprefix;

  # 数据指针，可以是 libxml2 文档或 nolibxml 缓冲区
  void *data;

  # 主版本号和次版本号
  unsigned version_major, version_minor;

  # NUMA 节点数量
  unsigned nbnumanodes;

  # 第一个 NUMA 节点和最后一个 NUMA 节点
  hwloc_obj_t first_numanode, last_numanode; /* 用于处理 v1distances 的临时 cousin-list */

  # 第一个 v1distances 和最后一个 v1distances
  struct hwloc__xml_imported_v1distances_s *first_v1dist, *last_v1dist;
};

/**************
 * XML 导出 *
 **************/

# 定义了一个结构体 hwloc__xml_export_state_s，用于存储 XML 导出的状态
typedef struct hwloc__xml_export_state_s {
  # 父状态指针
  struct hwloc__xml_export_state_s *parent;

  # 新子节点函数指针，用于创建新的子节点状态
  void (*new_child)(struct hwloc__xml_export_state_s *parentstate, struct hwloc__xml_export_state_s *state, const char *name);

  # 新属性函数指针，用于创建新的属性
  void (*new_prop)(struct hwloc__xml_export_state_s *state, const char *name, const char *value);

  # 添加内容函数指针，用于向状态中添加内容
  void (*add_content)(struct hwloc__xml_export_state_s *state, const char *buffer, size_t length);

  # 结束对象函数指针，用于结束对象的状态
  void (*end_object)(struct hwloc__xml_export_state_s *state, const char *name);

  # XML 导出数据结构
  struct hwloc__xml_export_data_s {
    # 定义一个指向 hwloc_obj_t 类型的指针 v1_memory_group，用于在导出到 v1 时需要在内存子节点上方插入中间组
    hwloc_obj_t v1_memory_group; /* if we need to insert intermediate group above memory children when exporting to v1 */
  } *global;

  /* 用于存储后端特定数据的不透明数据。
   * 静态分配以允许常用代码进行堆栈分配，而不需要知道实际后端的需求。
   */
  char data[40];
} * hwloc__xml_export_state_t; 
// 定义指向 hwloc__xml_export_state_t 结构体的指针

HWLOC_DECLSPEC void hwloc__xml_export_topology(hwloc__xml_export_state_t parentstate, hwloc_topology_t topology, unsigned long flags);
// 声明导出拓扑结构的函数，接受父状态、拓扑结构和标志作为参数

HWLOC_DECLSPEC void hwloc__xml_export_diff(hwloc__xml_export_state_t parentstate, hwloc_topology_diff_t diff);
// 声明导出拓扑差异的函数，接受父状态和拓扑差异作为参数

/******************
 * XML components *
 ******************/

struct hwloc_xml_callbacks {
  int (*backend_init)(struct hwloc_xml_backend_data_s *bdata, const char *xmlpath, const char *xmlbuffer, int xmlbuflen);
  // 定义指向 backend_init 函数的指针，接受指向 hwloc_xml_backend_data_s 结构体的指针、XML 路径、XML 缓冲区和缓冲区长度作为参数

  int (*export_file)(struct hwloc_topology *topology, struct hwloc__xml_export_data_s *edata, const char *filename, unsigned long flags);
  // 定义指向 export_file 函数的指针，接受指向 hwloc_topology 结构体、指向 hwloc__xml_export_data_s 结构体的指针、文件名和标志作为参数

  int (*export_buffer)(struct hwloc_topology *topology, struct hwloc__xml_export_data_s *edata, char **xmlbuffer, int *buflen, unsigned long flags);
  // 定义指向 export_buffer 函数的指针，接受指向 hwloc_topology 结构体、指向 hwloc__xml_export_data_s 结构体的指针、XML 缓冲区指针、缓冲区长度指针和标志作为参数

  void (*free_buffer)(void *xmlbuffer);
  // 定义指向 free_buffer 函数的指针，接受 XML 缓冲区指针作为参数

  int (*import_diff)(struct hwloc__xml_import_state_s *state, const char *xmlpath, const char *xmlbuffer, int xmlbuflen, hwloc_topology_diff_t *diff, char **refnamep);
  // 定义指向 import_diff 函数的指针，接受指向 hwloc__xml_import_state_s 结构体的指针、XML 路径、XML 缓冲区、缓冲区长度、拓扑差异指针和引用名称指针作为参数

  int (*export_diff_file)(union hwloc_topology_diff_u *diff, const char *refname, const char *filename);
  // 定义指向 export_diff_file 函数的指针，接受指向 hwloc_topology_diff_u 联合体、引用名称和文件名作为参数

  int (*export_diff_buffer)(union hwloc_topology_diff_u *diff, const char *refname, char **xmlbuffer, int *buflen);
  // 定义指向 export_diff_buffer 函数的指针，接受指向 hwloc_topology_diff_u 联合体、引用名称、XML 缓冲区指针和缓冲区长度指针作为参数
};

struct hwloc_xml_component {
  struct hwloc_xml_callbacks *nolibxml_callbacks;
  // 定义指向 hwloc_xml_callbacks 结构体的指针，用于非 libxml 后端

  struct hwloc_xml_callbacks *libxml_callbacks;
  // 定义指向 hwloc_xml_callbacks 结构体的指针，用于 libxml 后端
};

HWLOC_DECLSPEC void hwloc_xml_callbacks_register(struct hwloc_xml_component *component);
// 声明注册 XML 回调函数的函数，接受指向 hwloc_xml_component 结构体的指针作为参数

HWLOC_DECLSPEC void hwloc_xml_callbacks_reset(void);
// 声明重置 XML 回调函数的函数
#endif /* PRIVATE_XML_H */
```