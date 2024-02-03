# `xmrig\src\3rdparty\hwloc\src\topology-xml.c`

```cpp
/*
 * 版权声明
 */
#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含硬件拓扑库的头文件
#include "private/xml.h"  // 包含私有的 XML 头文件
#include "private/private.h"  // 包含私有的头文件
#include "private/misc.h"  // 包含私有的杂项头文件
#include "private/debug.h"  // 包含私有的调试头文件

#include <math.h>  // 包含数学库的头文件

// 返回 XML 的详细信息级别
int
hwloc__xml_verbose(void)
{
  static int checked = 0;  // 静态变量，用于标记是否已经检查过
  static int verbose = 0;  // 静态变量，存储详细信息级别
  if (!checked) {
    const char *env = getenv("HWLOC_XML_VERBOSE");  // 获取环境变量 HWLOC_XML_VERBOSE
    if (env)
      verbose = atoi(env);  // 将环境变量转换为整数
    checked = 1;  // 标记已经检查过
  }
  return verbose;  // 返回详细信息级别
}

// 检查是否禁用 libxml 的导入
static int
hwloc_nolibxml_import(void)
{
  static int checked = 0;  // 静态变量，用于标记是否已经检查过
  static int nolibxml = 0;  // 静态变量，存储是否禁用 libxml 的导入
  if (!checked) {
    const char *env = getenv("HWLOC_LIBXML");  // 获取环境变量 HWLOC_LIBXML
    if (env) {
      nolibxml = !atoi(env);  // 将环境变量转换为整数，并取反
    } else {
      env = getenv("HWLOC_LIBXML_IMPORT");  // 获取环境变量 HWLOC_LIBXML_IMPORT
      if (env)
        nolibxml = !atoi(env);  // 将环境变量转换为整数，并取反
    }
    checked = 1;  // 标记已经检查过
  }
  return nolibxml;  // 返回是否禁用 libxml 的导入
}

// 检查是否禁用 libxml 的导出
static int
hwloc_nolibxml_export(void)
{
  static int checked = 0;  // 静态变量，用于标记是否已经检查过
  static int nolibxml = 0;  // 静态变量，存储是否禁用 libxml 的导出
  if (!checked) {
    const char *env = getenv("HWLOC_LIBXML");  // 获取环境变量 HWLOC_LIBXML
    if (env) {
      nolibxml = !atoi(env);  // 将环境变量转换为整数，并取反
    } else {
      env = getenv("HWLOC_LIBXML_EXPORT");  // 获取环境变量 HWLOC_LIBXML_EXPORT
      if (env)
        nolibxml = !atoi(env);  // 将环境变量转换为整数，并取反
    }
    checked = 1;  // 标记已经检查过
  }
  return nolibxml;  // 返回是否禁用 libxml 的导出
}

// 计算 Base64 编码后的长度
#define BASE64_ENCODED_LENGTH(length) (4*(((length)+2)/3))

// XML 回调函数
static struct hwloc_xml_callbacks *hwloc_nolibxml_callbacks = NULL, *hwloc_libxml_callbacks = NULL;

// 注册 XML 回调函数
void
hwloc_xml_callbacks_register(struct hwloc_xml_component *comp)
{
  if (!hwloc_nolibxml_callbacks)
    # 将comp结构体中的nolibxml_callbacks赋值给hwloc_nolibxml_callbacks
    hwloc_nolibxml_callbacks = comp->nolibxml_callbacks;
    # 如果hwloc_libxml_callbacks为空，则将comp结构体中的libxml_callbacks赋值给hwloc_libxml_callbacks
    if (!hwloc_libxml_callbacks)
        hwloc_libxml_callbacks = comp->libxml_callbacks;
void
hwloc_xml_callbacks_reset(void)
{
  // 重置无libxml和libxml的回调函数指针
  hwloc_nolibxml_callbacks = NULL;
  hwloc_libxml_callbacks = NULL;
}

/************************************************
 ********* XML import (common routines) *********
 ************************************************/

// 定义一些常量
#define _HWLOC_OBJ_CACHE_OLD (HWLOC_OBJ_TYPE_MAX+1) /* temporarily used when importing pre-v2.0 attribute-less cache types */
#define _HWLOC_OBJ_FUTURE    (HWLOC_OBJ_TYPE_MAX+2) /* temporarily used when ignoring future types */

// 解析并处理对象的属性
static void
hwloc__xml_import_object_attr(struct hwloc_topology *topology,
                  struct hwloc_xml_backend_data_s *data,
                  struct hwloc_obj *obj,
                  const char *name, const char *value,
                  hwloc__xml_import_state_t state,
                  int *ignore)
{
  // 如果属性名是"type"，则已经处理过，直接返回
  if (!strcmp(name, "type")) {
    /* already handled */
    return;
  }

  // 如果属性名是"os_index"，将属性值转换为无符号长整型并赋值给obj->os_index
  else if (!strcmp(name, "os_index"))
    obj->os_index = strtoul(value, NULL, 10);
  // 如果属性名是"gp_index"，将属性值转换为无符号长长整型并赋值给obj->gp_index
  else if (!strcmp(name, "gp_index")) {
    obj->gp_index = strtoull(value, NULL, 10);
    // 如果obj->gp_index为0且启用了详细输出，打印警告信息
    if (!obj->gp_index && hwloc__xml_verbose())
      fprintf(stderr, "%s: unexpected zero gp_index, topology may be invalid\n", state->global->msgprefix);
    // 如果obj->gp_index大于等于topology->next_gp_index，更新topology->next_gp_index
    if (obj->gp_index >= topology->next_gp_index)
      topology->next_gp_index = obj->gp_index + 1;
  } else if (!strcmp(name, "id")) { /* forward compat */
    // 如果属性名是"id"，根据属性值的不同情况进行处理
    if (!strncmp(value, "obj", 3)) {
      // 如果属性值以"obj"开头，将其后的部分转换为无符号长长整型并赋值给obj->gp_index
      obj->gp_index = strtoull(value+3, NULL, 10);
      // 如果obj->gp_index为0且启用了详细输出，打印警告信息
      if (!obj->gp_index && hwloc__xml_verbose())
        fprintf(stderr, "%s: unexpected zero id, topology may be invalid\n", state->global->msgprefix);
      // 如果obj->gp_index大于等于topology->next_gp_index，更新topology->next_gp_index
      if (obj->gp_index >= topology->next_gp_index)
        topology->next_gp_index = obj->gp_index + 1;
    } else {
      // 如果启用了详细输出，打印警告信息
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: unexpected id `%s' not-starting with `obj', ignoring\n", state->global->msgprefix, value);
    }
  } else if (!strcmp(name, "cpuset")) {
    // 如果obj->cpuset为空，分配一个位图给obj->cpuset
    if (!obj->cpuset)
      obj->cpuset = hwloc_bitmap_alloc();
  # 根据属性名判断是否为 cpuset，如果是则将值转换为位图
  hwloc_bitmap_sscanf(obj->cpuset, value);
  # 如果属性名为 complete_cpuset，且对象的 complete_cpuset 为空，则分配内存
  else if (!strcmp(name, "complete_cpuset")) {
    if (!obj->complete_cpuset)
      obj->complete_cpuset = hwloc_bitmap_alloc();
    # 将值转换为位图
    hwloc_bitmap_sscanf(obj->complete_cpuset, value);
  }
  # 如果属性名为 allowed_cpuset，且对象没有父节点，则将值转换为位图并赋给 topology 的 allowed_cpuset
  else if (!strcmp(name, "allowed_cpuset")) {
    /* ignored except for root */
    if (!obj->parent)
      hwloc_bitmap_sscanf(topology->allowed_cpuset, value);
  }
  # 如果属性名为 nodeset，且对象的 nodeset 为空，则分配内存
  else if (!strcmp(name, "nodeset")) {
    if (!obj->nodeset)
      obj->nodeset = hwloc_bitmap_alloc();
    # 将值转换为位图
    hwloc_bitmap_sscanf(obj->nodeset, value);
  }
  # 如果属性名为 complete_nodeset，且对象的 complete_nodeset 为空，则分配内存
  else if (!strcmp(name, "complete_nodeset")) {
    if (!obj->complete_nodeset)
      obj->complete_nodeset = hwloc_bitmap_alloc();
    # 将值转换为位图
    hwloc_bitmap_sscanf(obj->complete_nodeset, value);
  }
  # 如果属性名为 allowed_nodeset，且对象没有父节点，则将值转换为位图并赋给 topology 的 allowed_nodeset
  else if (!strcmp(name, "allowed_nodeset")) {
    /* ignored except for root */
    if (!obj->parent)
      hwloc_bitmap_sscanf(topology->allowed_nodeset, value);
  }
  # 如果属性名为 name，且对象的 name 不为空，则释放原有内存并赋予新值
  else if (!strcmp(name, "name")) {
    if (obj->name)
      free(obj->name);
    obj->name = strdup(value);
  }
  # 如果属性名为 subtype，且对象的 subtype 不为空，则释放原有内存并赋予新值
  else if (!strcmp(name, "subtype")) {
    if (obj->subtype)
      free(obj->subtype);
    obj->subtype = strdup(value);
  }
  # 如果属性名为 cache_size，将值转换为无符号长整型并根据对象类型赋值给对应的属性
  else if (!strcmp(name, "cache_size")) {
    unsigned long long lvalue = strtoull(value, NULL, 10);
    if (hwloc__obj_type_is_cache(obj->type) || obj->type == _HWLOC_OBJ_CACHE_OLD || obj->type == HWLOC_OBJ_MEMCACHE)
      obj->attr->cache.size = lvalue;
    else if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring cache_size attribute for non-cache object type\n",
          state->global->msgprefix);
  }
  # 如果属性名为 cache_linesize，将值转换为无符号长整型并根据对象类型赋值给对应的属性
  else if (!strcmp(name, "cache_linesize")) {
    unsigned long lvalue = strtoul(value, NULL, 10);
    if (hwloc__obj_type_is_cache(obj->type) || obj->type == _HWLOC_OBJ_CACHE_OLD || obj->type == HWLOC_OBJ_MEMCACHE)
      obj->attr->cache.linesize = lvalue;
  # 如果启用了 XML 详细模式
  else if (hwloc__xml_verbose())
    # 输出忽略缓存行大小属性的警告信息
    fprintf(stderr, "%s: ignoring cache_linesize attribute for non-cache object type\n",
        state->global->msgprefix);
  }

  # 如果属性名为 "cache_associativity"
  else if (!strcmp(name, "cache_associativity")) {
    # 将属性值转换为整数
    int lvalue = atoi(value);
    # 如果对象类型是缓存，或者是旧的缓存对象类型，或者是内存缓存
    if (hwloc__obj_type_is_cache(obj->type) || obj->type == _HWLOC_OBJ_CACHE_OLD || obj->type == HWLOC_OBJ_MEMCACHE)
      # 设置对象的关联度属性
      obj->attr->cache.associativity = lvalue;
    else if (hwloc__xml_verbose())
      # 输出忽略关联度属性的警告信息
      fprintf(stderr, "%s: ignoring cache_associativity attribute for non-cache object type\n",
          state->global->msgprefix);
  }

  # 如果属性名为 "cache_type"
  else if (!strcmp(name, "cache_type")) {
    # 将属性值转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    # 如果对象类型是缓存，或者是旧的缓存对象类型，或者是内存缓存
    if (hwloc__obj_type_is_cache(obj->type) || obj->type == _HWLOC_OBJ_CACHE_OLD || obj->type == HWLOC_OBJ_MEMCACHE) {
      # 如果属性值是合法的缓存类型
      if (lvalue == HWLOC_OBJ_CACHE_UNIFIED
      || lvalue == HWLOC_OBJ_CACHE_DATA
      || lvalue == HWLOC_OBJ_CACHE_INSTRUCTION)
        # 设置对象的缓存类型属性
        obj->attr->cache.type = (hwloc_obj_cache_type_t) lvalue;
      else
        if (hwloc__xml_verbose())
          # 输出忽略无效缓存类型属性的警告信息
          fprintf(stderr, "%s: ignoring invalid cache_type attribute %lu\n",
                  state->global->msgprefix, lvalue);
    } else if (hwloc__xml_verbose())
      # 输出忽略缓存类型属性的警告信息
      fprintf(stderr, "%s: ignoring cache_type attribute for non-cache object type\n",
          state->global->msgprefix);
  }

  # 如果属性名为 "local_memory"
  else if (!strcmp(name, "local_memory")) {
    # 将属性值转换为无符号长长整型
    unsigned long long lvalue = strtoull(value, NULL, 10);
    # 如果对象类型是 NUMA 节点
    if (obj->type == HWLOC_OBJ_NUMANODE)
      # 设置对象的本地内存属性
      obj->attr->numanode.local_memory = lvalue;
    # 如果对象没有父对象
    else if (!obj->parent)
      # 设置拓扑结构的本地内存属性
      topology->machine_memory.local_memory = lvalue;
    else if (hwloc__xml_verbose())
      # 输出忽略本地内存属性的警告信息
      fprintf(stderr, "%s: ignoring local_memory attribute for non-NUMAnode non-root object\n",
          state->global->msgprefix);
  }

  # 如果属性名为 "depth"
  else if (!strcmp(name, "depth")) {
    # 将属性值转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    # 如果对象类型是缓存，或者是旧的缓存对象类型，或者是内存缓存
    if (hwloc__obj_type_is_cache(obj->type) || obj->type == _HWLOC_OBJ_CACHE_OLD || obj->type == HWLOC_OBJ_MEMCACHE) {
    # 设置对象属性的深度值
    obj->attr->cache.depth = lvalue;
     } else if (obj->type == HWLOC_OBJ_GROUP || obj->type == HWLOC_OBJ_BRIDGE) {
       # 对于组对象或桥接对象，将被核心覆盖
     } else if (hwloc__xml_verbose())
       # 如果是非深度对象类型，则忽略深度属性
       fprintf(stderr, "%s: ignoring depth attribute for object type without depth\n",
           state->global->msgprefix);
  }

  else if (!strcmp(name, "kind")) {
    # 将字符串转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    if (obj->type == HWLOC_OBJ_GROUP)
      # 设置组对象的种类属性
      obj->attr->group.kind = lvalue;
    else if (hwloc__xml_verbose())
      # 如果不是组对象类型，则忽略种类属性
      fprintf(stderr, "%s: ignoring kind attribute for non-group object type\n",
          state->global->msgprefix);
  }

  else if (!strcmp(name, "subkind")) {
    # 将字符串转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    if (obj->type == HWLOC_OBJ_GROUP)
      # 设置组对象的子种类属性
      obj->attr->group.subkind = lvalue;
    else if (hwloc__xml_verbose())
      # 如果不是组对象类型，则忽略子种类属性
      fprintf(stderr, "%s: ignoring subkind attribute for non-group object type\n",
          state->global->msgprefix);
  }

  else if (!strcmp(name, "dont_merge")) {
    # 将字符串转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    if (obj->type == HWLOC_OBJ_GROUP)
      # 设置组对象的不合并属性
      obj->attr->group.dont_merge = (unsigned char) lvalue;
    else if (hwloc__xml_verbose())
      # 如果不是组对象类型，则忽略不合并属性
      fprintf(stderr, "%s: ignoring dont_merge attribute for non-group object type\n",
          state->global->msgprefix);
  }

  else if (!strcmp(name, "pci_busid")) {
    switch (obj->type) {
    case HWLOC_OBJ_PCI_DEVICE:
    case HWLOC_OBJ_BRIDGE: {
      unsigned domain, bus, dev, func;
      # 将字符串解析为 PCI 总线 ID
      if (sscanf(value, "%x:%02x:%02x.%01x",
         &domain, &bus, &dev, &func) != 4) {
    if (hwloc__xml_verbose())
      # 如果格式无效，则忽略 PCI 总线 ID
      fprintf(stderr, "%s: ignoring invalid pci_busid format string %s\n",
          state->global->msgprefix, value);
    *ignore = 1;
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
      // 如果不支持 32 位 PCI 域
      } else if (domain > 0xffff) {
    // 如果域大于 0xffff
    static int warned = 0;
    // 静态变量 warned 初始化为 0
    if (!warned && HWLOC_SHOW_ALL_ERRORS())
      // 如果 warned 为假且显示所有错误
      fprintf(stderr, "hwloc/xml: Ignoring PCI device with non-16bit domain.\nPass --enable-32bits-pci-domain to configure to support such devices\n(warning: it would break the library ABI, don't enable unless really needed).\n");
    // 输出警告信息
    warned = 1;
    // warned 置为 1
    *ignore = 1;
    // 忽略该设备
#endif
      } else {
    // 否则
    obj->attr->pcidev.domain = domain;
    // 设置对象的域
    obj->attr->pcidev.bus = bus;
    // 设置对象的总线
    obj->attr->pcidev.dev = dev;
    // 设置对象的设备
    obj->attr->pcidev.func = func;
    // 设置对象的功能
      }
      break;
    }
    default:
      // 默认情况
      if (hwloc__xml_verbose())
    // 如果启用了详细输出
    fprintf(stderr, "%s: ignoring pci_busid attribute for non-PCI object\n",
        state->global->msgprefix);
      // 输出忽略非 PCI 对象的警告信息
      break;
    }
  }

  else if (!strcmp(name, "pci_type")) {
    // 如果属性名为 "pci_type"
    switch (obj->type) {
    // 根据对象类型进行切换
    case HWLOC_OBJ_PCI_DEVICE:
    case HWLOC_OBJ_BRIDGE: {
      // 如果是 PCI 设备或桥接设备
      unsigned classid, vendor, device, subvendor, subdevice, revision;
      // 定义无符号整型变量
      if (sscanf(value, "%x [%04x:%04x] [%04x:%04x] %02x",
         &classid, &vendor, &device, &subvendor, &subdevice, &revision) != 6) {
    // 如果格式化输入失败
    if (hwloc__xml_verbose())
      // 如果启用了详细输出
      fprintf(stderr, "%s: ignoring invalid pci_type format string %s\n",
          state->global->msgprefix, value);
      // 输出忽略无效 pci_type 格式的警告信息
      } else {
    obj->attr->pcidev.class_id = classid;
    // 设置对象的 class_id
    obj->attr->pcidev.vendor_id = vendor;
    // 设置对象的 vendor_id
    obj->attr->pcidev.device_id = device;
    // 设置对象的 device_id
    obj->attr->pcidev.subvendor_id = subvendor;
    // 设置对象的 subvendor_id
    obj->attr->pcidev.subdevice_id = subdevice;
    // 设置对象的 subdevice_id
    obj->attr->pcidev.revision = revision;
    // 设置对象的 revision
      }
      break;
    }
    default:
      // 默认情况
      if (hwloc__xml_verbose())
    // 如果启用了详细输出
    fprintf(stderr, "%s: ignoring pci_type attribute for non-PCI object\n",
        state->global->msgprefix);
      // 输出忽略非 PCI 对象的警告信息
      break;
    }
  }

  else if (!strcmp(name, "pci_link_speed")) {
    // 如果属性名为 "pci_link_speed"
    switch (obj->type) {
    // 根据对象类型进行切换
    case HWLOC_OBJ_PCI_DEVICE:
    case HWLOC_OBJ_BRIDGE: {
      // 如果是 PCI 设备或桥接设备
      obj->attr->pcidev.linkspeed = (float) atof(value);
      // 设置对象的链接速度
      break;
    }
    # 如果标签名为 default
    default:
      # 如果启用了 XML 详细输出，打印忽略非 PCI 对象的 pci_link_speed 属性警告信息
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring pci_link_speed attribute for non-PCI object\n",
        state->global->msgprefix);
      # 结束当前 case
      break;
    }
  }

  # 如果标签名为 bridge_type
  else if (!strcmp(name, "bridge_type")) {
    # 根据对象类型进行判断
    switch (obj->type) {
    # 如果对象类型为 HWLOC_OBJ_BRIDGE
    case HWLOC_OBJ_BRIDGE: {
      # 定义上游类型和下游类型
      unsigned upstream_type, downstream_type;
      # 如果无法按指定格式解析 value 字符串
      if (sscanf(value, "%u-%u", &upstream_type, &downstream_type) != 2) {
    # 如果启用了 XML 详细输出，打印忽略无效 bridge_type 格式字符串的警告信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring invalid bridge_type format string %s\n",
          state->global->msgprefix, value);
      } else {
    # 将解析得到的上游类型和下游类型赋值给对象的桥属性
    obj->attr->bridge.upstream_type = (hwloc_obj_bridge_type_t) upstream_type;
    obj->attr->bridge.downstream_type = (hwloc_obj_bridge_type_t) downstream_type;
        # FIXME 验证上游/下游类型是否有效
      };
      # 结束当前 case
      break;
    }
    # 如果对象类型不为 HWLOC_OBJ_BRIDGE
    default:
      # 如果启用了 XML 详细输出，打印忽略非桥对象的 bridge_type 属性警告信息
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring bridge_type attribute for non-bridge object\n",
        state->global->msgprefix);
      # 结束当前 case
      break;
    }
  }

  # 如果标签名为 bridge_pci
  else if (!strcmp(name, "bridge_pci")) {
    # 根据对象类型进行判断
    switch (obj->type) {
    # 如果对象类型为 HWLOC_OBJ_BRIDGE
    case HWLOC_OBJ_BRIDGE: {
      # 定义域、次级总线和子级总线
      unsigned domain, secbus, subbus;
      # 如果无法按指定格式解析 value 字符串
      if (sscanf(value, "%x:[%02x-%02x]",
         &domain, &secbus, &subbus) != 3) {
    # 如果启用了 XML 详细输出，打印忽略无效 bridge_pci 格式字符串的警告信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring invalid bridge_pci format string %s\n",
          state->global->msgprefix, value);
    # 将 ignore 标志置为 1
    *ignore = 1;
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
      // 如果不支持32位PCI域
      } else if (domain > 0xffff) {
    // 静态变量，用于标记是否已经发出警告
    static int warned = 0;
    // 如果未发出警告且允许显示所有错误
    if (!warned && HWLOC_SHOW_ALL_ERRORS())
      // 输出警告信息
      fprintf(stderr, "hwloc/xml: Ignoring bridge to PCI with non-16bit domain.\nPass --enable-32bits-pci-domain to configure to support such devices\n(warning: it would break the library ABI, don't enable unless really needed).\n");
    // 标记已经发出警告
    warned = 1;
    // 忽略此对象
    *ignore = 1;
#endif
      } else {
        /* FIXME verify that downstream type vs pci info are valid */
    // 设置对象的PCI域
    obj->attr->bridge.downstream.pci.domain = domain;
    // 设置对象的次要总线号
    obj->attr->bridge.downstream.pci.secondary_bus = secbus;
    // 设置对象的从属总线号
    obj->attr->bridge.downstream.pci.subordinate_bus = subbus;
      }
      break;
    }
    default:
      // 如果允许显示详细信息
      if (hwloc__xml_verbose())
    // 输出警告信息
    fprintf(stderr, "%s: ignoring bridge_pci attribute for non-bridge object\n",
        state->global->msgprefix);
      break;
    }
  }

  else if (!strcmp(name, "osdev_type")) {
    switch (obj->type) {
    case HWLOC_OBJ_OS_DEVICE: {
      unsigned osdev_type;
      // 将字符串转换为无符号整数，赋值给osdev_type
      if (sscanf(value, "%u", &osdev_type) != 1) {
    // 如果允许显示详细信息
    if (hwloc__xml_verbose())
      // 输出警告信息
      fprintf(stderr, "%s: ignoring invalid osdev_type format string %s\n",
          state->global->msgprefix, value);
      } else
    // 设置对象的OS设备类型
    obj->attr->osdev.type = (hwloc_obj_osdev_type_t) osdev_type;
      break;
    }
    default:
      // 如果允许显示详细信息
      if (hwloc__xml_verbose())
    // 输出警告信息
    fprintf(stderr, "%s: ignoring osdev_type attribute for non-osdev object\n",
        state->global->msgprefix);
      break;
    }
  }

  else if (data->version_major < 2) {
    /************************
     * deprecated from 1.x
     */
    // 如果属性名为"os_level"或"online_cpuset"
    if (!strcmp(name, "os_level")
    || !strcmp(name, "online_cpuset"))
      { /* ignored */ }

    /*************************
     * deprecated from 1.0
     */
    // 如果属性名为"dmi_board_vendor"
    else if (!strcmp(name, "dmi_board_vendor")) {
      // 如果值不为空
      if (value[0])
    // 添加对象信息
    hwloc_obj_add_info(obj, "DMIBoardVendor", value);
    }
    // 如果属性名为"dmi_board_name"
    else if (!strcmp(name, "dmi_board_name")) {
      // 如果值不为空
      if (value[0])
    // 为给定的 hwloc 对象添加 DMIBoardName 属性和对应的值
    hwloc_obj_add_info(obj, "DMIBoardName", value);
    }

    // 如果版本号小于 1
    else if (data->version_major < 1) {
      /*************************
       * 从 0.9 版本开始已废弃
       */
      // 如果属性名为 memory_kB
      if (!strcmp(name, "memory_kB")) {
    // 将字符串类型的 value 转换为无符号长整型
    unsigned long long lvalue = strtoull(value, NULL, 10);
    // 如果对象类型为 _HWLOC_OBJ_CACHE_OLD
    if (obj->type == _HWLOC_OBJ_CACHE_OLD)
      // 设置缓存大小
      obj->attr->cache.size = lvalue << 10;
    // 如果对象类型为 HWLOC_OBJ_NUMANODE
    else if (obj->type == HWLOC_OBJ_NUMANODE)
      // 设置本地内存大小
      obj->attr->numanode.local_memory = lvalue << 10;
    // 如果没有父对象
    else if (!obj->parent)
      // 设置机器内存的本地内存大小
      topology->machine_memory.local_memory = lvalue << 10;
    // 如果启用了 XML 详细输出
    else if (hwloc__xml_verbose())
      // 输出警告信息
      fprintf(stderr, "%s: ignoring memory_kB attribute for non-NUMAnode non-root object\n",
          state->global->msgprefix);
      }
      // 如果属性名为 huge_page_size_kB
      else if (!strcmp(name, "huge_page_size_kB")) {
    // 将字符串类型的 value 转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    // 如果对象类型为 HWLOC_OBJ_NUMANODE 或者没有父对象
    if (obj->type == HWLOC_OBJ_NUMANODE || !obj->parent) {
      // 获取 NUMAnode 或者机器内存的属性
      struct hwloc_numanode_attr_s *memory = obj->type == HWLOC_OBJ_NUMANODE ? &obj->attr->numanode : &topology->machine_memory;
      // 如果没有设置页类型
      if (!memory->page_types) {
        // 分配内存
        memory->page_types = malloc(sizeof(*memory->page_types));
        memory->page_types_len = 1;
      }
      // 断言内存已分配
      assert(memory->page_types);
      // 设置页大小
      memory->page_types[0].size = lvalue << 10;
    } else if (hwloc__xml_verbose()) {
      // 输出警告信息
      fprintf(stderr, "%s: ignoring huge_page_size_kB attribute for non-NUMAnode non-root object\n",
          state->global->msgprefix);
    }
      }
      // 如果属性名为 huge_page_free
      else if (!strcmp(name, "huge_page_free")) {
    // 将字符串类型的 value 转换为无符号长整型
    unsigned long lvalue = strtoul(value, NULL, 10);
    // 如果对象类型为 HWLOC_OBJ_NUMANODE 或者没有父对象
    if (obj->type == HWLOC_OBJ_NUMANODE || !obj->parent) {
      // 获取 NUMAnode 或者机器内存的属性
      struct hwloc_numanode_attr_s *memory = obj->type == HWLOC_OBJ_NUMANODE ? &obj->attr->numanode : &topology->machine_memory;
      // 如果没有设置页类型
      if (!memory->page_types) {
        // 分配内存
        memory->page_types = malloc(sizeof(*memory->page_types));
        memory->page_types_len = 1;
      }
      // 断言内存已分配
      assert(memory->page_types);
      // 设置页数量
      memory->page_types[0].count = lvalue;
    } else if (hwloc__xml_verbose()) {
      # 如果设置了 XML 详细输出，打印忽略巨大页面释放属性的警告信息
      fprintf(stderr, "%s: ignoring huge_page_free attribute for non-NUMAnode non-root object\n",
          state->global->msgprefix);
    }
      }
      # 从 0.9 版本开始废弃的部分结束
      else goto unknown;
    }
    # 从 1.0 版本开始废弃的部分结束
    else goto unknown;
  }
  else {
  unknown:
    # 如果设置了 XML 详细输出，打印忽略未知对象属性的警告信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring unknown object attribute %s\n",
          state->global->msgprefix, name);
  }
# 解析 XML 中的信息名称和信息值
static int
hwloc___xml_import_info(char **infonamep, char **infovaluep,
                        hwloc__xml_import_state_t state)
{
  char *infoname = NULL;  # 初始化信息名称为空
  char *infovalue = NULL;  # 初始化信息值为空

  while (1):  # 进入循环
    char *attrname, *attrvalue;  # 定义属性名称和属性值
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  # 如果获取下一个属性失败，则退出循环
      break;
    if (!strcmp(attrname, "name"))  # 如果属性名称为"name"
      infoname = attrvalue  # 将属性值赋给信息名称
    else if (!strcmp(attrname, "value"))  # 如果属性名称为"value"
      infovalue = attrvalue  # 将属性值赋给信息值
    else
      return -1  # 否则返回错误
  *infonamep = infoname  # 将信息名称赋给指针
  *infovaluep = infovalue  # 将信息值赋给指针

  return state->global->close_tag(state)  # 返回关闭标签的状态
}

# 解析 XML 中的对象信息
static int
hwloc__xml_import_obj_info(struct hwloc_xml_backend_data_s *data,
                           hwloc_obj_t obj,
                           hwloc__xml_import_state_t state)
{
  char *infoname = NULL;  # 初始化信息名称为空
  char *infovalue = NULL;  # 初始化信息值为空
  int err;  # 定义错误码

  err = hwloc___xml_import_info(&infoname, &infovalue, state)  # 调用解析信息的函数
  if (err < 0)  # 如果解析出错
    return err  # 返回错误码

  if (infoname):  # 如果信息名称存在
    # 空字符串会被 libxml 忽略
    if (data->version_major < 2 and
    (!strcmp(infoname, "Type") or !strcmp(infoname, "CoProcType"))):  # 如果版本小于2且信息名称为"Type"或"CoProcType"
      # 1.x 存储子类型在 Type 或 CoProcType 中
      if (infovalue):  # 如果信息值存在
    if (obj->subtype)  # 如果对象的子类型存在
      free(obj->subtype)  # 释放对象的子类型
    obj->subtype = strdup(infovalue)  # 将信息值赋给对象的子类型
      else:  # 如果信息值不存在
        pass  # 什么也不做
    else:  # 如果信息名称不是"Type"或"CoProcType"
      if (infovalue)  # 如果信息值存在
    hwloc_obj_add_info(obj, infoname, infovalue)  # 将信息名称和信息值添加到对象中

  return err  # 返回错误码

# 解析 XML 中的页面类型
static int
hwloc__xml_import_pagetype(hwloc_topology_t topology __hwloc_attribute_unused, struct hwloc_numanode_attr_s *memory,
               hwloc__xml_import_state_t state)
{
  uint64_t size = 0, count = 0  # 初始化大小和数量为0

  while (1):  # 进入循环
    char *attrname, *attrvalue  # 定义属性名称和属性值
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  # 如果获取下一个属性失败，则退出循环
      break;
    if (!strcmp(attrname, "size"))  # 如果属性名称为"size"
      size = strtoull(attrvalue, NULL, 10)  # 将属性值转换为无符号长长整型
    else if (!strcmp(attrname, "count"))  # 如果属性名称为"count"
      count = strtoull(attrvalue, NULL, 10)  # 将属性值转换为无符号长长整型
    else
      return -1  # 否则返回错误
  if (size):  # 如果大小存在
    unsigned idx = memory->page_types_len  # 获取页面类型长度
    struct hwloc_memory_page_type_s *tmp  # 定义临时页面类型
    # 重新分配内存，用于存储页面类型数据，扩展到 (idx+1)*sizeof(*memory->page_types) 大小
    tmp = realloc(memory->page_types, (idx+1)*sizeof(*memory->page_types));
    # 如果分配失败，则忽略这个页面类型的条目
    if (tmp) { 
      # 将重新分配的内存赋值给 page_types
      memory->page_types = tmp;
      # 更新 page_types_len 的长度
      memory->page_types_len = idx+1;
      # 设置当前页面类型的大小
      memory->page_types[idx].size = size;
      # 设置当前页面类型的数量
      memory->page_types[idx].count = count;
    }
  }
  # 返回全局状态的关闭标签
  return state->global->close_tag(state);
static int
hwloc__xml_v1import_distances(struct hwloc_xml_backend_data_s *data,
                  hwloc_obj_t obj,
                  hwloc__xml_import_state_t state)
{
  unsigned long reldepth = 0, nbobjs = 0;  // 定义相对深度和对象数量变量
  float latbase = 0;  // 定义基础延迟变量
  char *tag;  // 定义标签变量
  int ret;  // 定义返回值变量

  while (1) {  // 进入无限循环
    char *attrname, *attrvalue;  // 定义属性名和属性值变量
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  // 如果获取下一个属性失败，则退出循环
      break;
    if (!strcmp(attrname, "nbobjs"))  // 如果属性名为"nbobjs"
      nbobjs = strtoul(attrvalue, NULL, 10);  // 将属性值转换为无符号长整型
    else if (!strcmp(attrname, "relative_depth"))  // 如果属性名为"relative_depth"
      reldepth = strtoul(attrvalue, NULL, 10);  // 将属性值转换为无符号长整型
    else if (!strcmp(attrname, "latency_base"))  // 如果属性名为"latency_base"
      latbase = (float) atof(attrvalue);  // 将属性值转换为浮点数
    else
      return -1;  // 否则返回错误
  }

  if (nbobjs && reldepth && latbase) {  // 如果对象数量、相对深度和基础延迟都不为0
    unsigned i;  // 定义无符号整型变量i
    float *matrix;  // 定义浮点数指针变量matrix
    struct hwloc__xml_imported_v1distances_s *v1dist;  // 定义导入v1距离结构体指针变量v1dist

    matrix = malloc(nbobjs*nbobjs*sizeof(float));  // 分配存储距离矩阵的内存空间
    v1dist = malloc(sizeof(*v1dist));  // 分配存储v1距离结构体的内存空间
    if (!matrix || !v1dist) {  // 如果内存分配失败
      if (hwloc__xml_verbose())  // 如果启用了详细输出
    fprintf(stderr, "%s: failed to allocate v1distance matrix for %lu objects\n",
        state->global->msgprefix, nbobjs);  // 输出错误信息
      free(v1dist);  // 释放v1dist内存
      free(matrix);  // 释放matrix内存
      return -1;  // 返回错误
    }

    v1dist->kind = HWLOC_DISTANCES_KIND_FROM_OS|HWLOC_DISTANCES_KIND_MEANS_LATENCY;  // 设置v1距离结构体的类型

    v1dist->nbobjs = nbobjs;  // 设置v1距离结构体的对象数量
    v1dist->floats = matrix;  // 设置v1距离结构体的浮点数指针

    for(i=0; i<nbobjs*nbobjs; i++) {  // 进入循环，遍历对象数量*对象数量次
      struct hwloc__xml_import_state_s childstate;  // 定义导入状态结构体变量childstate
      char *attrname, *attrvalue;  // 定义属性名和属性值变量
      float val;  // 定义浮点数变量val

      ret = state->global->find_child(state, &childstate, &tag);  // 查找子节点
      if (ret <= 0 || strcmp(tag, "latency")) {  // 如果查找失败或标签不是"latency"
    /* a latency child is needed */
    free(matrix);  // 释放matrix内存
    free(v1dist);  // 释放v1dist内存
    # 返回错误代码 -1
    return -1;
      }

      # 获取下一个属性的名称和值
      ret = state->global->next_attr(&childstate, &attrname, &attrvalue);
      # 如果获取失败或者属性名不是"value"，释放内存并返回错误代码 -1
      if (ret < 0 || strcmp(attrname, "value")) {
    free(matrix);
    free(v1dist);
    return -1;
      }

      # 将属性值转换为浮点数，并乘以latbase，赋值给matrix[i]
      val = (float) atof((char *) attrvalue);
      matrix[i] = val * latbase;

      # 关闭当前标签
      ret = state->global->close_tag(&childstate);
      # 如果关闭失败，释放内存并返回错误代码 -1
      if (ret < 0) {
    free(matrix);
    free(v1dist);
    return -1;
      }

      # 关闭子标签
      state->global->close_child(&childstate);
    }

    # 如果nbobjs小于2
    if (nbobjs < 2) {
      # 如果只有一个对象，打印警告信息并释放内存
      assert(nbobjs == 1);
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring invalid distance matrix with only 1 object\n",
        state->global->msgprefix);
      free(matrix);
      free(v1dist);

    } else if (obj->parent) {
      # 如果对象有父节点，释放内存
      free(matrix);
      free(v1dist);

    } else {
      # 将距离加入队列
      v1dist->prev = data->last_v1dist;
      v1dist->next = NULL;
      if (data->last_v1dist)
    data->last_v1dist->next = v1dist;
      else
    data->first_v1dist = v1dist;
      data->last_v1dist = v1dist;
    }
  }

  # 关闭当前标签
  return state->global->close_tag(state);
static int
hwloc__xml_import_userdata(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj,
               hwloc__xml_import_state_t state)
{
  size_t length = 0;  // 初始化长度为0
  int encoded = 0;  // 初始化编码标志为0
  char *name = NULL; /* optional */  // 初始化名称为空

  int ret;  // 定义返回值

  while (1) {  // 进入无限循环
    char *attrname, *attrvalue;  // 定义属性名和属性值的指针
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  // 如果获取下一个属性失败，则跳出循环
      break;
    if (!strcmp(attrname, "length"))  // 如果属性名为"length"
      length = strtoul(attrvalue, NULL, 10);  // 将属性值转换为无符号长整型
    else if (!strcmp(attrname, "encoding"))  // 如果属性名为"encoding"
      encoded = !strcmp(attrvalue, "base64");  // 如果属性值为"base64"，则编码标志为1，否则为0
    else if (!strcmp(attrname, "name"))  // 如果属性名为"name"
      name = attrvalue;  // 将属性值赋给名称
    else
      return -1;  // 其他情况返回-1
  }

  if (!topology->userdata_import_cb) {  // 如果用户数据导入回调函数为空
    const char *buffer;  // 定义缓冲区
    size_t reallength = encoded ? BASE64_ENCODED_LENGTH(length) : length;  // 如果编码标志为真，则实际长度为编码后的长度，否则为原始长度
    ret = state->global->get_content(state, &buffer, reallength);  // 获取内容
    if (ret < 0)  // 如果获取内容失败
      return -1;  // 返回-1

  } else if (topology->userdata_not_decoded) {  // 如果用户数据未解码
      const char *buffer;  // 定义缓冲区
      char *fakename;  // 定义假名称
      size_t reallength = encoded ? BASE64_ENCODED_LENGTH(length) : length;  // 如果编码标志为真，则实际长度为编码后的长度，否则为原始长度
      ret = state->global->get_content(state, &buffer, reallength);  // 获取内容
      if (ret < 0)  // 如果获取内容失败
        return -1;  // 返回-1
      fakename = malloc(6 + 1 + (name ? strlen(name) : 4) + 1);  // 分配假名称的内存空间
      if (!fakename)  // 如果分配内存失败
        return -1;  // 返回-1
      sprintf(fakename, encoded ? "base64%c%s" : "normal%c%s", name ? ':' : '-', name ? name : "anon");  // 根据编码标志和名称生成假名称
      topology->userdata_import_cb(topology, obj, fakename, buffer, length);  // 调用用户数据导入回调函数
      free(fakename);  // 释放假名称的内存空间

  } else if (encoded && length) {  // 如果编码标志为真且长度不为0
      const char *encoded_buffer;  // 定义编码缓冲区
      size_t encoded_length = BASE64_ENCODED_LENGTH(length);  // 计算编码后的长度
      ret = state->global->get_content(state, &encoded_buffer, encoded_length);  // 获取编码内容
      if (ret < 0)  // 如果获取内容失败
        return -1;  // 返回-1
      if (ret) {  // 如果获取内容成功
    char *decoded_buffer = malloc(length+1);  // 分配解码缓冲区的内存空间
    if (!decoded_buffer)  // 如果分配内存失败
      return -1;  // 返回-1
    assert(encoded_buffer[encoded_length] == 0);  // 断言编码缓冲区的结尾为0
    ret = hwloc_decode_from_base64(encoded_buffer, decoded_buffer, length+1);  // 解码内容
    # 如果返回值不等于长度，释放解码缓冲区并返回-1
    if (ret != (int) length) {
      free(decoded_buffer);
      return -1;
    }
    # 调用回调函数，传入拓扑结构、对象、名称、解码缓冲区和长度
    topology->userdata_import_cb(topology, obj, name, decoded_buffer, length);
    # 释放解码缓冲区
    free(decoded_buffer);
      }

  } else { /* 在非编码情况下始终处理长度==0 */
      # 定义一个空字符串
      const char *buffer = "";
      # 如果长度不为0，获取内容并存入buffer中
      if (length) {
    ret = state->global->get_content(state, &buffer, length);
    # 如果返回值小于0，返回-1
    if (ret < 0)
      return -1;
      }
      # 调用回调函数，传入拓扑结构、对象、名称、buffer和长度
      topology->userdata_import_cb(topology, obj, name, buffer, length);
  }
  # 关闭内容
  state->global->close_content(state);
  # 关闭标签
  return state->global->close_tag(state);
# 报告 XML 拓扑加载顺序错误的函数
static void hwloc__xml_import_report_outoforder(hwloc_topology_t topology, hwloc_obj_t new, hwloc_obj_t old)
{
  # 获取程序名称
  char *progname = hwloc_progname(topology);
  # 获取原始版本号
  const char *origversion = hwloc_obj_get_info_by_name(topology->levels[0][0], "hwlocVersion");
  # 获取原始程序名称
  const char *origprogname = hwloc_obj_get_info_by_name(topology->levels[0][0], "ProcessName");
  char *c1, *cc1, t1[64];
  char *c2 = NULL, *cc2 = NULL, t2[64];

  # 将新对象的 cpuset 和 complete_cpuset 转换为字符串
  hwloc_bitmap_asprintf(&c1, new->cpuset);
  hwloc_bitmap_asprintf(&cc1, new->complete_cpuset);
  # 将新对象的类型转换为字符串
  hwloc_obj_type_snprintf(t1, sizeof(t1), new, 0);

  # 如果旧对象的 cpuset 存在，则将其转换为字符串
  if (old->cpuset)
    hwloc_bitmap_asprintf(&c2, old->cpuset);
  # 如果旧对象的 complete_cpuset 存在，则将其转换为字符串
  if (old->complete_cpuset)
    hwloc_bitmap_asprintf(&cc2, old->complete_cpuset);
  # 将旧对象的类型转换为字符串
  hwloc_obj_type_snprintf(t2, sizeof(t2), old, 0);

  # 输出错误报告
  fprintf(stderr, "****************************************************************************\n");
  fprintf(stderr, "* hwloc has encountered an out-of-order XML topology load.\n");
  fprintf(stderr, "* Object %s cpuset %s complete %s\n",
      t1, c1, cc1);
  fprintf(stderr, "* was inserted after object %s with %s and %s.\n",
      t2, c2 ? c2 : "none", cc2 ? cc2 : "none");
  fprintf(stderr, "* The error occured in hwloc %s inside process `%s', while\n",
      HWLOC_VERSION,
      progname ? progname : "<unknown>");
  # 如果存在原始版本号或原始程序名称，则输出相应信息
  if (origversion || origprogname)
    fprintf(stderr, "* the input XML was generated by hwloc %s inside process `%s'.\n",
        origversion ? origversion : "(unknown version)",
        origprogname ? origprogname : "<unknown>");
  else
    # 如果不存在原始版本号或原始程序名称，则输出相应信息
    fprintf(stderr, "* the input XML was generated by an unspecified ancient hwloc release.\n");
  fprintf(stderr, "* Please check that your input topology XML file is valid.\n");
  fprintf(stderr, "* Set HWLOC_DEBUG_CHECK=1 in the environment to detect further issues.\n");
  fprintf(stderr, "****************************************************************************\n");

  # 释放内存
  free(c1);
  free(cc1);
  free(c2);
  free(cc2);
  free(progname);
}

static int
# 从 XML 数据导入硬件拓扑对象
def hwloc__xml_import_object(hwloc_topology_t topology,
             struct hwloc_xml_backend_data_s *data,
             hwloc_obj_t parent, hwloc_obj_t obj, int *gotignored,
             hwloc__xml_import_state_t state)
{
  # 初始化被忽略标志
  int ignored = 0;
  int childrengotignored = 0;
  int attribute_less_cache = 0;
  int numa_was_root = 0;
  char *tag;
  struct hwloc__xml_import_state_s childstate;

  # 设置父对象，因为在导入过程中或子函数中会用到
  obj->parent = parent;

  # 处理属性
  while (1) {
    char *attrname, *attrvalue;
    # 获取下一个属性名和属性值
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)
      break;
    # 检查属性名是否为"type"
    if (!strcmp(attrname, "type")) {
      # 尝试解析属性值为硬件对象类型
      if (hwloc_type_sscanf(attrvalue, &obj->type, NULL, 0) < 0) {
    # 如果属性值为"Cache"
    if (!strcasecmp(attrvalue, "Cache")) {
      # 设置对象类型为旧的缓存类型
      obj->type = _HWLOC_OBJ_CACHE_OLD; # 将在下面修复
      attribute_less_cache = 1;
    # 如果属性值为"System"
    else if (!strcasecmp(attrvalue, "System")):
      # 如果父对象为空，则设置对象类型为机器
      if (!parent)
        obj->type = HWLOC_OBJ_MACHINE;
      else:
        # 如果父对象不为空，则报错
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: obsolete System object only allowed at root\n",
              state->global->msgprefix);
        goto error_with_object;
    # 如果属性值为"Tile"
    else if (!strcasecmp(attrvalue, "Tile")):
      # 设置对象类型为组，设置组的种类为Intel Tile
      obj->type = HWLOC_OBJ_GROUP;
      obj->attr->group.kind = HWLOC_GROUP_KIND_INTEL_TILE;
    # 如果属性值为"Module"
    else if (!strcasecmp(attrvalue, "Module")):
      # 设置对象类型为组，设置组的种类为Intel Module
      obj->type = HWLOC_OBJ_GROUP;
      obj->attr->group.kind = HWLOC_GROUP_KIND_INTEL_MODULE;
    # 如果属性值为"MemCache"
    else if (!strcasecmp(attrvalue, "MemCache")):
      # 忽略未来可能的类型
      obj->type = _HWLOC_OBJ_FUTURE;
      ignored = 1;
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: %s object not-supported, will be ignored\n",
            state->global->msgprefix, attrvalue);
    } else {
      # 如果对象类型字符串无法识别，且启用了详细输出，则在标准错误流中打印错误信息
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: unrecognized object type string %s\n",
            state->global->msgprefix, attrvalue);
      # 跳转到错误处理代码块
      goto error_with_object;
    }
      }
    } else {
      /* type needed first */
      # 如果对象的类型是 HWLOC_OBJ_TYPE_NONE，则在标准错误流中打印错误信息
      if (obj->type == HWLOC_OBJ_TYPE_NONE) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: object attribute %s found before type\n",
          state->global->msgprefix,  attrname);
    # 跳转到错误处理代码块
    goto error_with_object;
      }
      # 导入对象属性
      hwloc__xml_import_object_attr(topology, data, obj, attrname, attrvalue, state, &ignored);
    }
  }

  /* process non-object subnodes to get info attrs (as well as page_types, etc) */
  # 处理非对象子节点以获取信息属性（以及 page_types 等）
  while (1) {
    int ret;

    tag = NULL;
    # 查找子节点
    ret = state->global->find_child(state, &childstate, &tag);
    if (ret < 0)
      # 跳转到错误处理代码块
      goto error;
    if (!ret)
      break;

    if (!strcmp(tag, "object")) {
      # 我们稍后会处理子节点
      break;

    } else if (!strcmp(tag, "page_type")) {
      # 如果对象类型是 HWLOC_OBJ_NUMANODE，则导入 page_type
      if (obj->type == HWLOC_OBJ_NUMANODE) {
    ret = hwloc__xml_import_pagetype(topology, &obj->attr->numanode, &childstate);
      } else if (!parent) {
    ret = hwloc__xml_import_pagetype(topology, &topology->machine_memory, &childstate);
      } else {
    # 如果启用了详细输出，则在标准错误流中打印错误信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid non-NUMAnode object child %s\n",
          state->global->msgprefix, tag);
    ret = -1;
      }

    } else if (!strcmp(tag, "info")) {
      # 导入对象信息
      ret = hwloc__xml_import_obj_info(data, obj, &childstate);
    } else if (data->version_major < 2 && !strcmp(tag, "distances")) {
      # 如果主版本号小于 2 且标签为 "distances"，则导入距离信息
      ret = hwloc__xml_v1import_distances(data, obj, &childstate);
    } else if (!strcmp(tag, "userdata")) {
      # 导入用户数据
      ret = hwloc__xml_import_userdata(topology, obj, &childstate);
    } else {
      # 如果启用了详细输出，则在标准错误流中打印错误信息
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: invalid special object child %s\n",
        state->global->msgprefix, tag);
      ret = -1;
    }

    if (ret < 0)
      # 跳转到错误处理代码块
      goto error;
    state->global->close_child(&childstate);
  }

  if (parent && obj->type == HWLOC_OBJ_MACHINE) {
    /* 如果存在父对象并且当前对象类型为机器，则将其替换为组对象 */
    obj->type = HWLOC_OBJ_GROUP;
  }

  if (parent && data->version_major >= 2) {
    /* 对于版本大于等于2.x，检查父/子对象类型 */
    if (hwloc__obj_type_is_normal(obj->type)) {
      if (!hwloc__obj_type_is_normal(parent->type)) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "normal object %s cannot be child of non-normal parent %s\n",
          hwloc_obj_type_string(obj->type), hwloc_obj_type_string(parent->type));
    goto error_with_object;
      }
    } else if (hwloc__obj_type_is_memory(obj->type)) {
      if (hwloc__obj_type_is_io(parent->type) || HWLOC_OBJ_MISC == parent->type) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "Memory object %s cannot be child of non-normal-or-memory parent %s\n",
          hwloc_obj_type_string(obj->type), hwloc_obj_type_string(parent->type));
    goto error_with_object;
      }
    } else if (hwloc__obj_type_is_io(obj->type)) {
      if (hwloc__obj_type_is_memory(parent->type) || HWLOC_OBJ_MISC == parent->type) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "I/O object %s cannot be child of non-normal-or-I/O parent %s\n",
          hwloc_obj_type_string(obj->type), hwloc_obj_type_string(parent->type));
    goto error_with_object;
      }
    }

  } else if (parent && data->version_major < 2) {
    /* 对于版本小于2.0，检查父/子对象类型 */
    if (hwloc__obj_type_is_normal(obj->type) || HWLOC_OBJ_NUMANODE == obj->type) {
      if (hwloc__obj_type_is_special(parent->type)) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "v1.x normal v1.x object %s cannot be child of special parent %s\n",
          hwloc_obj_type_string(obj->type), hwloc_obj_type_string(parent->type));
    goto error_with_object;
      }
    } else if (hwloc__obj_type_is_io(obj->type)) {
      if (HWLOC_OBJ_MISC == parent->type) {
    // 如果父对象类型为MISC，则报错
    # 如果启用了 XML 详细输出，则打印错误信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "I/O object %s cannot be child of Misc parent\n",
          hwloc_obj_type_string(obj->type));
    # 跳转到错误处理标签
    goto error_with_object;
      }
    }
  }

  # 如果数据版本小于 2
  if (data->version_major < 2) {
    /***************************
     * 1.x specific checks
     */

    # 在 v1.x 版本中的特定检查

    # 将 v2.0 之前的 NUMA 节点的子节点附加到正常的父节点上
    if (parent && parent->type == HWLOC_OBJ_NUMANODE) {
      parent = parent->parent;
      assert(parent);
    }

    # 如果对象类型是 NUMA 节点
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      if (!parent) {
    # 特殊情况下的 NUMA 节点根节点，重新插入一个 Machine 对象
    hwloc_obj_t machine = hwloc_alloc_setup_object(topology, HWLOC_OBJ_MACHINE, HWLOC_UNKNOWN_INDEX);
    machine->cpuset = hwloc_bitmap_dup(obj->cpuset);
    machine->complete_cpuset = hwloc_bitmap_dup(obj->cpuset);
    machine->nodeset = hwloc_bitmap_dup(obj->nodeset);
    machine->complete_nodeset = hwloc_bitmap_dup(obj->complete_nodeset);
    topology->levels[0][0] = machine;
    parent = machine;
    numa_was_root = 1;

      } else if (!hwloc_bitmap_isequal(obj->complete_cpuset, parent->complete_cpuset)) {
    # 这个 NUMA 节点与其父节点的本地性不同
    # 不要将其附加到此父节点，否则它将获取其父节点的 CPU 集合
    # 添加一个具有所需本地性的中间 Group
    int needgroup = 1;
    hwloc_obj_t sibling;

    sibling = parent->memory_first_child;
    if (sibling && !sibling->subtype
        && !sibling->next_sibling
        && obj->subtype && !strcmp(obj->subtype, "MCDRAM")
        && hwloc_bitmap_iszero(obj->complete_cpuset)) {
      # 这是 KNL MCDRAM，我们希望将其附加到其 DDR 兄弟节点附近
      needgroup = 0;
    }
    /* 理想情况下，我们还将在未来的非 KNL 平台上检测类似情况，该平台具有多个本地 NUMA 节点。
     * 这在 v1.x 中不太可能发生。
     * 而且我们无法确定是否需要这个无 CPU 的节点。
     */

    // 如果需要分组，并且硬件过滤器检查保留对象类型为组
    if (needgroup
        && hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP)) {
      // 分配并设置组对象
      hwloc_obj_t group = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
      group->gp_index = 0; /* 将在发现结束时初始化，一旦我们知道最大值 */
      group->cpuset = hwloc_bitmap_dup(obj->cpuset);
      group->complete_cpuset = hwloc_bitmap_dup(obj->cpuset);
      group->nodeset = hwloc_bitmap_dup(obj->nodeset);
      group->complete_nodeset = hwloc_bitmap_dup(obj->complete_nodeset);
      group->attr->group.kind = HWLOC_GROUP_KIND_MEMORY;
      // 根据父对象插入组对象
      hwloc_insert_object_by_parent(topology, parent, group);
      parent = group;
    }
      }
    }

    /* 修复从 v2.0 之前的 XML 导入的无属性缓存 */
    if (attribute_less_cache) {
      assert(obj->type == _HWLOC_OBJ_CACHE_OLD);
      obj->type = hwloc_cache_type_by_depth_type(obj->attr->cache.depth, obj->attr->cache.type);
    }

    /* 修复在 v2.0 之前的 XML 中由 cpusets 插入的 Misc 对象 */
    if (obj->type == HWLOC_OBJ_MISC && obj->cpuset)
      obj->type = HWLOC_OBJ_GROUP;

    /* 检查集合的一致性。
     * 1.7.2 及更早版本报告了只有 cpuset 的 I/O 组，我们不想拒绝这些 XML。
     * 忽略这些组，因为修复缺失的集合很困难（需要查看尚不可用的子集合）。
     * 只是中止非组的 XML。
     */
    if (!obj->cpuset != !obj->complete_cpuset) {
      /* 有一些 cpuset 没有其他的 */
      if (obj->type == HWLOC_OBJ_GROUP) {
    ignored = 1;
      } else {
    # 如果启用了 XML 详细输出，则在标准错误流中打印无效对象的信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid object %s P#%u with some missing cpusets\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
      }
    # 如果节点集不为空或者完整节点集不为空，则执行以下代码
    } else if (!obj->nodeset != !obj->complete_nodeset) {
      # 如果有一些节点集而其他节点集缺失，则执行以下代码
      if (obj->type == HWLOC_OBJ_GROUP) {
    # 忽略此情况
    ignored = 1;
      } else {
    # 如果启用了 XML 详细输出，则在标准错误流中打印无效对象的信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid object %s P#%u with some missing nodesets\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
      }
    # 如果节点集不为空且 cpuset 为空，则执行以下代码
    } else if (obj->nodeset && !obj->cpuset) {
      # 如果节点集存在但 cpuset 不存在（在 2.0 之前是允许的），则执行以下代码
      if (obj->type == HWLOC_OBJ_GROUP) {
    # 忽略此情况
    ignored = 1;
      } else {
    # 如果启用了 XML 详细输出，则在标准错误流中打印无效对象的信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid object %s P#%u with either cpuset or nodeset missing\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
      }
    # 1.x 特定检查结束
    /* end of 1.x specific checks */
  }

  # 2.0 版本向后兼容性
  /* 2.0 backward compatibility */
  if (obj->type == HWLOC_OBJ_GROUP) {
    # 如果对象类型是组，并且组的类型是英特尔 DIE 或者子类型不为空且等于 "Die"，则将对象类型设置为 DIE
    if (obj->attr->group.kind == HWLOC_GROUP_KIND_INTEL_DIE
    || (obj->subtype && !strcmp(obj->subtype, "Die")))
      obj->type = HWLOC_OBJ_DIE;
  }

  # 检查缓存属性是否与实际类型一致
  /* check that cache attributes are coherent with the actual type */
  if (hwloc__obj_type_is_cache(obj->type)
      && obj->type != hwloc_cache_type_by_depth_type(obj->attr->cache.depth, obj->attr->cache.type)) {
    # 如果启用了 XML 详细输出，则在标准错误流中打印无效缓存类型的信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid cache type %s with attribute depth %u and type %d\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->attr->cache.depth, (int) obj->attr->cache.type);
    # 跳转到错误处理代码块
    goto error_with_object;
  }

  # 检查特殊类型与 cpuset 的关系
  /* check special types vs cpuset */
  if (!obj->cpuset && !hwloc__obj_type_is_special(obj->type)) {
    # 如果启用了 XML 详细输出，则打印错误消息到标准错误流，指出无效的普通对象或特殊对象
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid normal object %s P#%u without cpuset\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
  }
  # 如果对象有 cpuset 并且对象类型是特殊对象，则打印错误消息到标准错误流，指出无效的特殊对象
  if (obj->cpuset && hwloc__obj_type_is_special(obj->type)) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid special object %s with cpuset\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type));
    # 跳转到错误处理代码块
    goto error_with_object;
  }

  # 检查父对象和子对象的 cpuset
  if (obj->cpuset && parent && !parent->cpuset) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid object %s P#%u with cpuset while parent has none\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
  }
  # 检查父对象和子对象的 nodeset
  if (obj->nodeset && parent && !parent->nodeset) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid object %s P#%u with nodeset while parent has none\n",
          state->global->msgprefix, hwloc_obj_type_string(obj->type), obj->os_index);
    # 跳转到错误处理代码块
    goto error_with_object;
  }

  # 检查 NUMA 节点对象是否有 nodeset
  if (obj->type == HWLOC_OBJ_NUMANODE) {
    if (!obj->nodeset) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: invalid NUMA node object P#%u without nodeset\n",
        state->global->msgprefix, obj->os_index);
      # 跳转到错误处理代码块
      goto error_with_object;
    }
    # 增加 NUMA 节点对象计数
    data->nbnumanodes++;
    # 设置对象的兄弟指针
    obj->prev_cousin = data->last_numanode;
    obj->next_cousin = NULL;
    if (data->last_numanode)
      data->last_numanode->next_cousin = obj;
    else
      data->first_numanode = obj;
    data->last_numanode = obj;
  }

  # 如果对象不符合过滤条件，则忽略该对象
  if (!hwloc_filter_check_keep_object(topology, obj)) {
    # 如果有父对象，则标记为被忽略
    if (parent)
      ignored = 1;
  }

  # 如果有父对象并且未被忽略，则执行以下代码
  if (parent && !ignored) {
    # 将对象插入到指定父对象下
    hwloc_insert_object_by_parent(topology, parent, obj);
    # insert_object_by_parent() 在插入时不会合并，所以 obj 仍然有效
  }

  # 处理对象的子节点，如果在上面的循环中找到了一个
  while (tag):
    ret = 0

    if (not strcmp(tag, "object")):
      # 分配并设置子对象
      childobj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_TYPE_MAX, HWLOC_UNKNOWN_INDEX)
      # 如果被忽略，则子对象的父对象为 parent，否则为 obj
      childobj->parent = ignored ? parent : obj
      # 导入 XML 中的对象
      ret = hwloc__xml_import_object(topology, data, ignored ? parent : obj, childobj,
                     &childrengotignored,
                     &childstate)
    else:
      if (hwloc__xml_verbose()):
        # 输出错误信息
        fprintf(stderr, "%s: invalid special object child %s while looking for objects\n",
        state->global->msgprefix, tag)
      ret = -1

    if (ret < 0):
      if (parent and not ignored):
        # 如果有父对象且没有被忽略，则跳转到错误处理
        goto error
      else:
        # 否则跳转到带有对象的错误处理
        goto error_with_object

    # 关闭子状态
    state->global->close_child(&childstate)

    tag = NULL
    ret = state->global->find_child(state, &childstate, &tag)
    if (ret < 0):
      if (parent and not ignored):
        # 如果有父对象且没有被忽略，则跳转到错误处理
        goto error
      else:
        # 否则跳转到带有对象的错误处理
        goto error_with_object
    if (not ret):
      # 如果没有找到子节点，则跳出循环
      break

  if (numa_was_root):
    # 复制 NUMA 信息到根对象，大部分可能是特定于根对象的
    unsigned i
    for(i=0; i<obj->infos_count; i++):
      info = &obj->infos[i]
      # 向父对象添加信息
      hwloc_obj_add_info(parent, info->name, info->value)
    # TODO 一些信息仅适用于根对象（hwlocVersion，ProcessName 等），从 obj 中删除它们？

  if (ignored):
    # 丢弃该对象，并告诉父对象有一个子对象被忽略
    hwloc_free_unlinked_object(obj)
    *gotignored = 1

  elif (obj->first_child):
    # 现在所有子对象都已插入，确保它们是有序的，这样核心就不必处理混乱的子对象列表
    cur, next = obj->first_child, obj->first_child->next_sibling
    # 遍历对象的子节点，cur指向当前节点，next指向下一个节点
    for(cur = obj->first_child, next = cur->next_sibling;
    next;
    cur = next, next = next->next_sibling) {
      /* 如果需要重新排序，至少有一对连续的子节点将是无序的。
       * 因此只需检查连续子节点对。
       *
       * 我们在上面检查过complete_cpuset总是被设置的。
       */
      if (hwloc_bitmap_compare_first(next->complete_cpuset, cur->complete_cpuset) < 0) {
    /* next应该在cur之前 */
    if (!childrengotignored) {
      static int reported = 0;
      if (!reported && HWLOC_SHOW_CRITICAL_ERRORS()) {
        hwloc__xml_import_report_outoforder(topology, next, cur);
        reported = 1;
      }
    }
    hwloc__reorder_children(obj);
    break;
      }
    }
    /* 只要没有中间的内存对象可能在被过滤掉时导致重新排序，就不需要重新排序内存子节点。 */
  }

  return state->global->close_tag(state);

 error_with_object:
  if (parent)
    /* root->parent为NULL，并且root已经被插入。调用者将清理该root。 */
    hwloc_free_unlinked_object(obj);
 error:
  return -1;
static int
hwloc__xml_v2import_support(hwloc_topology_t topology,
                            hwloc__xml_import_state_t state)
{
  char *name = NULL;  // 初始化 name 变量为 NULL
  int value = 1; // 初始化 value 变量为 1，这是可选的
  while (1) {  // 进入无限循环
    char *attrname, *attrvalue;  // 定义两个指针变量
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  // 调用 next_attr 函数获取下一个属性名和属性值，如果返回值小于 0 则跳出循环
      break;
    if (!strcmp(attrname, "name"))  // 如果属性名为 "name"
      name = attrvalue;  // 将属性值赋给 name
    else if (!strcmp(attrname, "value"))  // 如果属性名为 "value"
      value = atoi(attrvalue);  // 将属性值转换为整数并赋给 value
    else {
      if (hwloc__xml_verbose())  // 如果启用了详细输出
    fprintf(stderr, "%s: ignoring unknown support attribute %s\n",
        state->global->msgprefix, attrname);  // 输出警告信息，忽略未知的支持属性
    }
  }

  if (name && topology->flags & HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT) {  // 如果 name 不为空且拓扑结构的标志包含 HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT
#ifdef HWLOC_DEBUG
    HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_support) == 4*sizeof(void*));  // 断言结构体的大小
    HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_discovery_support) == 6);  // 断言结构体的大小
    HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_cpubind_support) == 11);  // 断言结构体的大小
    HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_membind_support) == 15);  // 断言结构体的大小
    HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_misc_support) == 1);  // 断言结构体的大小
#endif

#define DO(_cat,_name) if (!strcmp(#_cat "." #_name, name)) topology->support._cat->_name = value  // 定义宏，根据属性名和值设置拓扑结构的支持属性
    DO(discovery,pu);
    else DO(discovery,numa);
    else DO(discovery,numa_memory);
    else DO(discovery,disallowed_pu);
    else DO(discovery,disallowed_numa);
    else DO(discovery,cpukind_efficiency);
    else DO(cpubind,set_thisproc_cpubind);
    else DO(cpubind,get_thisproc_cpubind);
    else DO(cpubind,set_proc_cpubind);
    else DO(cpubind,get_proc_cpubind);
    else DO(cpubind,set_thisthread_cpubind);
    else DO(cpubind,get_thisthread_cpubind);
    else DO(cpubind,set_thread_cpubind);
    else DO(cpubind,get_thread_cpubind);
    else DO(cpubind,get_thisproc_last_cpu_location);
    else DO(cpubind,get_proc_last_cpu_location);
    else DO(cpubind,get_thisthread_last_cpu_location);
    else DO(membind,set_thisproc_membind);
    else DO(membind,get_thisproc_membind);
    # 如果条件为真，则执行 DO 函数中的 set_proc_membind
    else DO(membind,set_proc_membind);
    # 如果条件为真，则执行 DO 函数中的 get_proc_membind
    else DO(membind,get_proc_membind);
    # 如果条件为真，则执行 DO 函数中的 set_thisthread_membind
    else DO(membind,set_thisthread_membind);
    # 如果条件为真，则执行 DO 函数中的 get_thisthread_membind
    else DO(membind,get_thisthread_membind);
    # 如果条件为真，则执行 DO 函数中的 set_area_membind
    else DO(membind,set_area_membind);
    # 如果条件为真，则执行 DO 函数中的 get_area_membind
    else DO(membind,get_area_membind);
    # 如果条件为真，则执行 DO 函数中的 alloc_membind
    else DO(membind,alloc_membind);
    # 如果条件为真，则执行 DO 函数中的 firsttouch_membind
    else DO(membind,firsttouch_membind);
    # 如果条件为真，则执行 DO 函数中的 bind_membind
    else DO(membind,bind_membind);
    # 如果条件为真，则执行 DO 函数中的 interleave_membind
    else DO(membind,interleave_membind);
    # 如果条件为真，则执行 DO 函数中的 nexttouch_membind
    else DO(membind,nexttouch_membind);
    # 如果条件为真，则执行 DO 函数中的 migrate_membind
    else DO(membind,migrate_membind);
    # 如果条件为真，则执行 DO 函数中的 get_area_memlocation
    else DO(membind,get_area_memlocation);

    # 如果条件为真，则执行以下代码块
    else if (!strcmp("custom.exported_support", name))
      # 在自定义/虚假字段中导出了支持，将其标记为在此处导入
      topology->support.misc->imported_support = 1;
static int
hwloc__xml_v2import_distances(hwloc_topology_t topology,
                  hwloc__xml_import_state_t state,
                  int heterotypes)
{
  // 初始化变量
  hwloc_obj_type_t unique_type = HWLOC_OBJ_TYPE_NONE;
  hwloc_obj_type_t *different_types = NULL;
  unsigned nbobjs = 0;
  int indexing = heterotypes;
  int os_indexing = 0;
  int gp_indexing = heterotypes;
  char *name = NULL;
  unsigned long kind = 0;
  unsigned nr_indexes, nr_u64values;
  uint64_t *indexes;
  uint64_t *u64values;
  int ret;

#define _TAG_NAME (heterotypes ? "distances2hetero" : "distances2")

  /* process attributes */
  // 处理属性
  while (1) {
    char *attrname, *attrvalue;
    // 获取下一个属性
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)
      break;
    // 解析属性
    if (!strcmp(attrname, "nbobjs"))
      nbobjs = strtoul(attrvalue, NULL, 10);
    else if (!strcmp(attrname, "type")) {
      // 解析类型
      if (hwloc_type_sscanf(attrvalue, &unique_type, NULL, 0) < 0) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: unrecognized %s type %s\n",
          state->global->msgprefix, _TAG_NAME, attrvalue);
    goto out;
      }
    }
    else if (!strcmp(attrname, "indexing")) {
      // 解析索引方式
      indexing = 1;
      if (!strcmp(attrvalue, "os"))
    os_indexing = 1;
      else if (!strcmp(attrvalue, "gp"))
    gp_indexing = 1;
    }
    else if (!strcmp(attrname, "kind")) {
      // 解析种类
      kind = strtoul(attrvalue, NULL, 10);
    }
    else if (!strcmp(attrname, "name")) {
      // 解析名称
      name = attrvalue;
    }
    else {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring unknown %s attribute %s\n",
        state->global->msgprefix, _TAG_NAME, attrname);
    }
  }

  /* abort if missing attribute */
  // 如果缺少属性，则中止
  if (!nbobjs || (!heterotypes && unique_type == HWLOC_OBJ_TYPE_NONE) || !indexing || !kind) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: %s missing some attributes\n",
          state->global->msgprefix, _TAG_NAME);
    # 跳转到标签out处
    goto out;
  }

  # 为索引和值数组分配内存空间
  indexes = malloc(nbobjs*sizeof(*indexes));
  u64values = malloc(nbobjs*nbobjs*sizeof(*u64values));
  # 如果存在不同类型的对象，则为不同类型的对象分配内存空间
  if (heterotypes)
    different_types = malloc(nbobjs*sizeof(*different_types));
  # 如果内存分配失败，则输出错误信息并跳转到标签out_with_arrays处
  if (!indexes || !u64values || (heterotypes && !different_types)) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: failed to allocate %s arrays for %u objects\n",
          state->global->msgprefix, _TAG_NAME, nbobjs);
    goto out_with_arrays;
  }

  # 处理子元素
  nr_indexes = 0;
  nr_u64values = 0;
  while (1) {
    struct hwloc__xml_import_state_s childstate;
    char *attrname, *attrvalue, *tag;
    const char *buffer;
    int length;
    int is_index = 0;
    int is_u64values = 0;

    # 查找子元素
    ret = state->global->find_child(state, &childstate, &tag);
    if (ret <= 0)
      break;

    # 判断子元素类型
    if (!strcmp(tag, "indexes"))
      is_index = 1;
    else if (!strcmp(tag, "u64values"))
      is_u64values = 1;
    # 如果子元素类型不是索引或值，则输出错误信息并跳转到标签out_with_arrays处
    if (!is_index && !is_u64values) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: %s with unrecognized child %s\n",
        state->global->msgprefix, _TAG_NAME, tag);
      goto out_with_arrays;
    }

    # 获取子元素的长度属性
    if (state->global->next_attr(&childstate, &attrname, &attrvalue) < 0
    || strcmp(attrname, "length")) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: %s child must have length attribute\n",
        state->global->msgprefix, _TAG_NAME);
      goto out_with_arrays;
    }
    length = atoi(attrvalue);

    # 获取子元素的内容
    ret = state->global->get_content(&childstate, &buffer, length);
    if (ret < 0) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: %s child needs content of length %d\n",
        state->global->msgprefix, _TAG_NAME, length);
      goto out_with_arrays;
    }

    if (is_index) {
      # 获取索引
      const char *tmp, *tmp2;
      if (nr_indexes >= nbobjs) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: %s with more than %u indexes\n",
          state->global->msgprefix, _TAG_NAME, nbobjs);
    # 跳转到数组处理结束的标签
    goto out_with_arrays;
      }
      # 临时变量指向缓冲区
      tmp = buffer;
      # 进入无限循环
      while (1) {
    # 定义指向字符的指针
    char *next;
    # 定义无符号长整型变量
    unsigned long long u;
    # 如果是异构类型
    if (heterotypes) {
      # 初始化硬件位置对象类型
      hwloc_obj_type_t t = HWLOC_OBJ_TYPE_NONE;
          # 如果遇到空字符，表示到达索引属性的结尾
          if (!*tmp)
            /* reached the end of this indexes attribute */
            break;
      # 如果无法识别异构类型
      if (hwloc_type_sscanf(tmp, &t, NULL, 0) < 0) {
        # 如果启用了 XML 详细输出，则打印错误信息
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: %s with unrecognized heterogeneous type %s\n",
              state->global->msgprefix, _TAG_NAME, tmp);
        # 跳转到数组处理结束的标签
        goto out_with_arrays;
      }
      # 查找冒号的位置
      tmp2 = strchr(tmp, ':');
      # 如果找不到冒号
      if (!tmp2) {
        # 如果启用了 XML 详细输出，则打印错误信息
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: %s with missing colon after heterogeneous type %s\n",
              state->global->msgprefix, _TAG_NAME, tmp);
        # 跳转到数组处理结束的标签
        goto out_with_arrays;
      }
      # 更新临时变量指向冒号后的位置
      tmp = tmp2+1;
      # 将不同类型的索引存入数组
      different_types[nr_indexes] = t;
    }
    # 将字符串转换为无符号长整型
    u = strtoull(tmp, &next, 0);
    # 如果下一个字符和当前字符相同，表示转换失败，退出循环
    if (next == tmp)
      break;
    # 将索引存入数组
    indexes[nr_indexes++] = u;
    # 如果下一个字符不是空格，退出循环
    if (*next != ' ')
      break;
    # 如果索引数量达到指定值，退出循环
    if (nr_indexes == nbobjs)
      break;
    # 更新临时变量指向下一个位置
    tmp = next+1;
      }

    } else if (is_u64values) {
      # 获取 uint64_t 值
      const char *tmp;
      # 如果 uint64_t 值的数量超过指定值，打印错误信息并跳转到数组处理结束的标签
      if (nr_u64values >= nbobjs*nbobjs) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: %s with more than %u u64values\n",
          state->global->msgprefix, _TAG_NAME, nbobjs*nbobjs);
    goto out_with_arrays;
      }
      # 临时变量指向缓冲区
      tmp = buffer;
      # 进入无限循环
      while (1) {
    # 定义指向字符的指针
    char *next;
    # 将字符串转换为无符号长整型
    unsigned long long u = strtoull(tmp, &next, 0);
    # 如果下一个字符和当前字符相同，表示转换失败，退出循环
    if (next == tmp)
      break;
    # 将 uint64_t 值存入数组
    u64values[nr_u64values++] = u;
    # 如果下一个字符不是空格，退出循环
    if (*next != ' ')
      break;
    # 如果 uint64_t 值的数量达到指定值，退出循环
    if (nr_u64values == nbobjs*nbobjs)
      break;
    # 更新临时变量指向下一个位置
    tmp = next+1;
      }
    }

    # 关闭子状态的内容
    state->global->close_content(&childstate);

    # 关闭子状态的标签
    ret = state->global->close_tag(&childstate);
    # 如果关闭标签失败，打印错误信息并跳转到数组处理结束的标签
    if (ret < 0) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: %s with more than %u indexes\n",
        state->global->msgprefix, _TAG_NAME, nbobjs);
      goto out_with_arrays;
    }
    state->global->close_child(&childstate);
  }

  if (nr_indexes != nbobjs) {
    // 如果索引数不等于对象数，则输出错误信息并跳转到out_with_arrays标签
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: %s with less than %u indexes\n",
          state->global->msgprefix, _TAG_NAME, nbobjs);
    goto out_with_arrays;
  }
  if (nr_u64values != nbobjs*nbobjs) {
    // 如果u64值的数量不等于对象数的平方，则输出错误信息并跳转到out_with_arrays标签
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: %s with less than %u u64values\n",
          state->global->msgprefix, _TAG_NAME, nbobjs*nbobjs);
    goto out_with_arrays;
  }

  if (nbobjs < 2) {
    /* distances with a single object are useless, even if the XML isn't invalid */
    // 如果对象数小于2，则输出警告信息并跳转到out_ignore标签
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring %s with only %u objects\n",
          state->global->msgprefix, _TAG_NAME, nbobjs);
    goto out_ignore;
  }
  if (unique_type == HWLOC_OBJ_PU || unique_type == HWLOC_OBJ_NUMANODE) {
    if (!os_indexing) {
      // 如果是PU或NUMA节点，并且没有os_indexing，则输出警告信息并跳转到out_ignore标签
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring PU or NUMA %s without os_indexing\n",
        state->global->msgprefix, _TAG_NAME);
      goto out_ignore;
    }
  } else {
    if (!gp_indexing) {
      // 如果不是PU或NUMA节点，并且没有gp_indexing，则输出警告信息并跳转到out_ignore标签
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring !PU or !NUMA %s without gp_indexing\n",
        state->global->msgprefix, _TAG_NAME);
      goto out_ignore;
    }
  }

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_NO_DISTANCES)
    // 如果拓扑结构标记中包含HWLOC_TOPOLOGY_FLAG_NO_DISTANCES，则跳转到out_ignore标签
    goto out_ignore;

  // 向拓扑结构中添加距离信息
  hwloc_internal_distances_add_by_index(topology, name, unique_type, different_types, nbobjs, indexes, u64values, kind, 0 /* assume grouping was applied when this matrix was discovered before exporting to XML */);

  /* prevent freeing below */
  // 防止下面的内存释放
  indexes = NULL;
  u64values = NULL;
  different_types = NULL;

 out_ignore:
  // 释放内存并返回close_tag函数的结果
  free(different_types);
  free(indexes);
  free(u64values);
  return state->global->close_tag(state);

 out_with_arrays:
  // 释放内存并返回-1
  free(different_types);
  free(indexes);
  free(u64values);
 out:
  // 返回-1
  return -1;
  #undef _TAG_NAME
}

static int
hwloc__xml_import_memattr_value(hwloc_topology_t topology,
                                hwloc_memattr_id_t id,
                                unsigned long flags,
                                hwloc__xml_import_state_t state)
{
  char *target_obj_gp_index_s = NULL;  // 初始化目标对象的全局索引字符串
  char *target_obj_type_s = NULL;  // 初始化目标对象的类型字符串
  hwloc_uint64_t target_obj_gp_index;  // 目标对象的全局索引
  char *value_s = NULL;  // 初始化值的字符串
  hwloc_uint64_t value;  // 值
  char *initiator_cpuset_s = NULL;  // 初始化发起者的 CPU 集合字符串
  char *initiator_obj_gp_index_s = NULL;  // 初始化发起者对象的全局索引字符串
  char *initiator_obj_type_s = NULL;  // 初始化发起者对象的类型字符串
  hwloc_obj_type_t target_obj_type = HWLOC_OBJ_TYPE_NONE;  // 目标对象的类型

  while (1) {
    char *attrname, *attrvalue;
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  // 获取下一个属性名和属性值
      break;
    if (!strcmp(attrname, "target_obj_gp_index"))  // 如果属性名是目标对象的全局索引
      target_obj_gp_index_s = attrvalue;  // 将属性值赋给目标对象的全局索引字符串
    else if (!strcmp(attrname, "target_obj_type"))  // 如果属性名是目标对象的类型
      target_obj_type_s = attrvalue;  // 将属性值赋给目标对象的类型字符串
    else if (!strcmp(attrname, "value"))  // 如果属性名是值
      value_s = attrvalue;  // 将属性值赋给值的字符串
    else if (!strcmp(attrname, "initiator_cpuset"))  // 如果属性名是发起者的 CPU 集合
      initiator_cpuset_s = attrvalue;  // 将属性值赋给发起者的 CPU 集合字符串
    else if (!strcmp(attrname, "initiator_obj_gp_index"))  // 如果属性名是发起者对象的全局索引
      initiator_obj_gp_index_s = attrvalue;  // 将属性值赋给发起者对象的全局索引字符串
    else if (!strcmp(attrname, "initiator_obj_type"))  // 如果属性名是发起者对象的类型
      initiator_obj_type_s = attrvalue;  // 将属性值赋给发起者对象的类型字符串
    else {
      if (hwloc__xml_verbose())  // 如果是详细模式
        fprintf(stderr, "%s: ignoring unknown memattr_value attribute %s\n",
                state->global->msgprefix, attrname);  // 输出忽略未知的内存属性值属性
      return -1;  // 返回错误
    }
  }

  if (!target_obj_type_s) {  // 如果没有目标对象类型
    if (hwloc__xml_verbose())  // 如果是详细模式
      fprintf(stderr, "%s: ignoring memattr_value without target_obj_type.\n",
              state->global->msgprefix);  // 输出忽略没有目标对象类型的内存属性值
    return -1;  // 返回错误
  }
  if (hwloc_type_sscanf(target_obj_type_s, &target_obj_type, NULL, 0) < 0) {  // 将目标对象类型字符串转换为类型
    if (hwloc__xml_verbose())  // 如果是详细模式
      fprintf(stderr, "%s: failed to identify memattr_value target object type %s\n",
              state->global->msgprefix, target_obj_type_s);  // 输出无法识别目标对象类型的内存属性值
    return -1;  // 返回错误
  }

  if (!value_s || !target_obj_gp_index_s) {  // 如果没有值或者没有目标对象的全局索引
    # 如果启用了 XML 详细输出，打印忽略没有值和目标对象 gp 索引的 memattr_value 的警告信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring memattr_value without value and target_obj_gp_index\n",
              state->global->msgprefix);
    # 返回错误代码 -1
    return -1;
  }
  # 将目标对象 gp 索引和值转换为无符号长长整型
  target_obj_gp_index = strtoull(target_obj_gp_index_s, NULL, 10);
  value = strtoull(value_s, NULL, 10);

  # 如果标志包含 HWLOC_MEMATTR_FLAG_NEED_INITIATOR
  if (flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    # 添加带有发起者的值
    struct hwloc_internal_location_s loc;
    # 如果没有发起者 cpuset 或者没有发起者对象 gp 索引和类型
    if (!initiator_cpuset_s && (!initiator_obj_gp_index_s || !initiator_obj_type_s)) {
      # 如果启用了 XML 详细输出，打印忽略没有发起者属性的 memattr_value 的警告信息
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: ignoring memattr_value without initiator attributes\n",
                state->global->msgprefix);
      # 返回错误代码 -1
      return -1;
    }

    # 设置发起者
    if (initiator_cpuset_s) {
      loc.type = HWLOC_LOCATION_TYPE_CPUSET;
      loc.location.cpuset = hwloc_bitmap_alloc();
      # 如果无法分配发起者 cpuset
      if (!loc.location.cpuset) {
        # 如果启用了 XML 详细输出，打印分配 memattr_value 发起者 cpuset 失败的警告信息
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: failed to allocated memattr_value initiator cpuset\n",
                  state->global->msgprefix);
        # 返回错误代码 -1
        return -1;
      }
      # 将发起者 cpuset 字符串转换为位图
      hwloc_bitmap_sscanf(loc.location.cpuset, initiator_cpuset_s);
    } else {
      loc.type = HWLOC_LOCATION_TYPE_OBJECT;
      loc.location.object.gp_index = strtoull(initiator_obj_gp_index_s, NULL, 10);
      # 如果无法识别发起者对象类型
      if (hwloc_type_sscanf(initiator_obj_type_s, &loc.location.object.type, NULL, 0) < 0) {
        # 如果启用了 XML 详细输出，打印无法识别 memattr_value 发起者对象类型的警告信息
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: failed to identify memattr_value initiator object type %s\n",
                  state->global->msgprefix, initiator_obj_type_s);
        # 返回错误代码 -1
        return -1;
      }
    }

    # 设置 memattr_value 的值和发起者
    hwloc_internal_memattr_set_value(topology, id, target_obj_type, target_obj_gp_index, (unsigned)-1, &loc, value);

    # 如果发起者类型为 HWLOC_LOCATION_TYPE_CPUSET，释放发起者 cpuset
    if (loc.type == HWLOC_LOCATION_TYPE_CPUSET)
      hwloc_bitmap_free(loc.location.cpuset);

  } else {
    # 添加没有发起者的值
    hwloc_internal_memattr_set_value(topology, id, target_obj_type, target_obj_gp_index, (unsigned)-1, NULL, value);
  }

  # 返回成功代码 0
  return 0;
static int
hwloc__xml_import_memattr(hwloc_topology_t topology,
                          hwloc__xml_import_state_t state)
{
  char *name = NULL;  // 初始化 name 变量为空
  unsigned long flags = (unsigned long) -1;  // 初始化 flags 变量为 -1
  hwloc_memattr_id_t id = (hwloc_memattr_id_t) -1;  // 初始化 id 变量为 -1
  int ret;  // 定义 ret 变量

  while (1) {  // 进入循环
    char *attrname, *attrvalue;  // 定义 attrname 和 attrvalue 变量
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)  // 如果获取下一个属性失败，则跳出循环
      break;
    if (!strcmp(attrname, "name"))  // 如果属性名为 "name"
      name = attrvalue;  // 将属性值赋给 name
    else if (!strcmp(attrname, "flags"))  // 如果属性名为 "flags"
      flags = strtoul(attrvalue, NULL, 10);  // 将属性值转换为无符号长整型赋给 flags
    else {
      if (hwloc__xml_verbose())  // 如果启用了详细输出
        fprintf(stderr, "%s: ignoring unknown memattr attribute %s\n",
                state->global->msgprefix, attrname);  // 输出忽略未知的 memattr 属性
      return -1;  // 返回错误
    }
  }

  if (name && flags != (unsigned long) -1
      && !(topology->flags & HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS)) {  // 如果 name 和 flags 都不为空且标志位中不包含 HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS
    hwloc_memattr_id_t _id;  // 定义 _id 变量

    ret = hwloc_memattr_get_by_name(topology, name, &_id);  // 获取指定名称的内存属性
    if (ret < 0) {  // 如果获取失败
      /* register a new attribute */
      ret = hwloc_memattr_register(topology, name, flags, &_id);  // 注册一个新的属性
      if (!ret)  // 如果注册成功
        id = _id;  // 将 _id 赋给 id
    } else {
      /* check the flags of the existing attribute  */
      unsigned long mflags;  // 定义 mflags 变量
      ret = hwloc_memattr_get_flags(topology, _id, &mflags);  // 获取现有属性的标志位
      if (!ret && mflags == flags)  // 如果获取成功且标志位相同
        id = _id;  // 将 _id 赋给 id
    }
    /* if there's no matching attribute, id is -1 and values will be ignored below */
  }

  while (1) {  // 进入循环
    struct hwloc__xml_import_state_s childstate;  // 定义 childstate 变量
    char *tag;  // 定义 tag 变量

    ret = state->global->find_child(state, &childstate, &tag);  // 查找子节点
    if (ret <= 0)  // 如果查找失败或没有子节点
      break;

    if (!strcmp(tag, "memattr_value")) {  // 如果子节点标签为 "memattr_value"
      ret = hwloc__xml_import_memattr_value(topology, id, flags, &childstate);  // 导入内存属性值
    } else {
      if (hwloc__xml_verbose())  // 如果启用了详细输出
        fprintf(stderr, "%s: memattr with unrecognized child %s\n",
                state->global->msgprefix, tag);  // 输出未识别的 memattr 子节点
      ret = -1;  // 返回错误
    }

    if (ret < 0)  // 如果导入失败
      goto error;  // 跳转到 error 标签
    # 调用全局状态中的close_child方法，关闭子状态
    state->global->close_child(&childstate);
  }

  # 返回全局状态中的close_tag方法的结果
  return state->global->close_tag(state);

 error:
  # 返回错误标志-1
  return -1;
  # 解析 XML 文件中的 cpukind 节点内容并导入到 hwloc_topology_t 结构中
static int
hwloc__xml_import_cpukind(hwloc_topology_t topology,
                          hwloc__xml_import_state_t state)
{
  # 初始化变量
  hwloc_bitmap_t cpuset = NULL;
  int forced_efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
  unsigned nr_infos = 0;
  struct hwloc_info_s *infos = NULL;
  int ret;

  # 循环解析 cpukind 节点的属性
  while (1) {
    char *attrname, *attrvalue;
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)
      break;
    # 如果属性名为 "cpuset"，则解析属性值并存储到 cpuset 变量中
    if (!strcmp(attrname, "cpuset")) {
      if (!cpuset)
        cpuset = hwloc_bitmap_alloc();
      hwloc_bitmap_sscanf(cpuset, attrvalue);
    } 
    # 如果属性名为 "forced_efficiency"，则将属性值转换为整数并存储到 forced_efficiency 变量中
    else if (!strcmp(attrname, "forced_efficiency")) {
      forced_efficiency = atoi(attrvalue);
    } 
    # 如果属性名不是已知的属性，则输出警告信息并释放资源后返回错误
    else {
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: ignoring unknown cpukind attribute %s\n",
                state->global->msgprefix, attrname);
      hwloc_bitmap_free(cpuset);
      return -1;
    }
  }

  # 循环解析 cpukind 节点的子节点
  while (1) {
    struct hwloc__xml_import_state_s childstate;
    char *tag;

    ret = state->global->find_child(state, &childstate, &tag);
    if (ret <= 0)
      break;

    # 如果子节点为 "info"，则解析其内容并添加到 infos 数组中
    if (!strcmp(tag, "info")) {
      char *infoname = NULL;
      char *infovalue = NULL;
      ret = hwloc___xml_import_info(&infoname, &infovalue, &childstate);
      if (!ret && infoname && infovalue)
        hwloc__add_info(&infos, &nr_infos, infoname, infovalue);
    } 
    # 如果子节点不是已知的子节点，则输出警告信息并返回错误
    else {
      if (hwloc__xml_verbose())
        fprintf(stderr, "%s: cpukind with unrecognized child %s\n",
                state->global->msgprefix, tag);
      ret = -1;
    }

    # 如果解析出错，则跳转到错误处理部分
    if (ret < 0)
      goto error;

    state->global->close_child(&childstate);
  }

  # 如果没有解析到 cpuset，则输出警告信息并返回错误
  if (!cpuset) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: ignoring cpukind without cpuset\n",
              state->global->msgprefix);
    goto error;
  }

  # 如果拓扑结构中设置了不支持 cpukind，则释放资源后返回
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS) {
    hwloc__free_infos(infos, nr_infos);
    hwloc_bitmap_free(cpuset);
  } else {
    # 注册内部 CPU 种类，将其与拓扑结构、CPU 集合、强制效率、信息、信息数量和标志关联起来
    hwloc_internal_cpukinds_register(topology, cpuset, forced_efficiency, infos, nr_infos, HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY);
    # 释放信息数组和信息数量
    hwloc__free_infos(infos, nr_infos);
  }

  # 返回状态的全局关闭标签
  return state->global->close_tag(state);

 error:
  # 释放信息数组和信息数量
  hwloc__free_infos(infos, nr_infos);
  # 释放 CPU 集合
  hwloc_bitmap_free(cpuset);
  # 返回错误状态
  return -1;
# 定义静态函数，用于处理导入的 XML 差异
static int
hwloc__xml_import_diff_one(hwloc__xml_import_state_t state,
               hwloc_topology_diff_t *firstdiffp,
               hwloc_topology_diff_t *lastdiffp)
{
  # 定义变量用于存储不同属性的字符串值
  char *type_s = NULL;
  char *obj_depth_s = NULL;
  char *obj_index_s = NULL;
  char *obj_attr_type_s = NULL;
  # char *obj_attr_index_s = NULL; 目前未使用
  char *obj_attr_name_s = NULL;
  char *obj_attr_oldvalue_s = NULL;
  char *obj_attr_newvalue_s = NULL;

  # 循环处理 XML 中的属性
  while (1) {
    char *attrname, *attrvalue;
    # 从全局状态中获取下一个属性的名称和值
    if (state->global->next_attr(state, &attrname, &attrvalue) < 0)
      break;
    # 根据属性名称进行不同的处理
    if (!strcmp(attrname, "type"))
      type_s = attrvalue;
    else if (!strcmp(attrname, "obj_depth"))
      obj_depth_s = attrvalue;
    else if (!strcmp(attrname, "obj_index"))
      obj_index_s = attrvalue;
    else if (!strcmp(attrname, "obj_attr_type"))
      obj_attr_type_s = attrvalue;
    else if (!strcmp(attrname, "obj_attr_index"))
      { /* obj_attr_index_s = attrvalue; 目前未使用 */ }
    else if (!strcmp(attrname, "obj_attr_name"))
      obj_attr_name_s = attrvalue;
    else if (!strcmp(attrname, "obj_attr_oldvalue"))
      obj_attr_oldvalue_s = attrvalue;
    else if (!strcmp(attrname, "obj_attr_newvalue"))
      obj_attr_newvalue_s = attrvalue;
    else {
      # 如果属性名称未知，则打印警告信息并返回错误
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: ignoring unknown diff attribute %s\n",
        state->global->msgprefix, attrname);
      return -1;
    }
  }

  # 根据 type_s 的值进行不同的处理
  if (type_s) {
    switch (atoi(type_s)) {
    default:
      break;
    case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR: {
      # 对象属性差异
      hwloc_topology_diff_obj_attr_type_t obj_attr_type;
      hwloc_topology_diff_t diff;

      # 检查必需的通用属性是否存在
      if (!obj_depth_s || !obj_index_s || !obj_attr_type_s) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: missing mandatory obj attr generic attributes\n",
          state->global->msgprefix);
    # 结束当前的循环
    break;
      }

      # 检查是否缺少必需的 obj_attr 属性值
      if (!obj_attr_oldvalue_s || !obj_attr_newvalue_s) {
    # 如果启用了 XML 详细输出，则打印错误信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: missing mandatory obj attr value attributes\n",
          state->global->msgprefix);
    # 结束当前的循环
    break;
      }

      # 检查 obj_attr_info 子类型的必需属性
      obj_attr_type = atoi(obj_attr_type_s);
      if (obj_attr_type == HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO && !obj_attr_name_s) {
    # 如果启用了 XML 详细输出，则打印错误信息
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: missing mandatory obj attr info name attribute\n",
          state->global->msgprefix);
    # 结束当前的循环
    break;
      }

      # 分配内存以存储差异对象
      diff = malloc(sizeof(*diff));
      if (!diff)
    return -1;
      # 设置差异对象的类型和索引
      diff->obj_attr.type = HWLOC_TOPOLOGY_DIFF_OBJ_ATTR;
      diff->obj_attr.obj_depth = atoi(obj_depth_s);
      diff->obj_attr.obj_index = atoi(obj_index_s);
      # 初始化差异对象的数据
      memset(&diff->obj_attr.diff, 0, sizeof(diff->obj_attr.diff));
      diff->obj_attr.diff.generic.type = obj_attr_type;

      # 根据不同的 obj_attr_type 进行不同的处理
      switch (obj_attr_type) {
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE:
    # 设置差异对象的大小属性值
    diff->obj_attr.diff.uint64.oldvalue = strtoull(obj_attr_oldvalue_s, NULL, 0);
    diff->obj_attr.diff.uint64.newvalue = strtoull(obj_attr_newvalue_s, NULL, 0);
    break;
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO:
    # 设置差异对象的信息属性值
    diff->obj_attr.diff.string.name = strdup(obj_attr_name_s);
    # 继续执行下一个 case
    /* FALLTHRU */
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME:
    # 设置差异对象的名称属性值
    diff->obj_attr.diff.string.oldvalue = strdup(obj_attr_oldvalue_s);
    diff->obj_attr.diff.string.newvalue = strdup(obj_attr_newvalue_s);
    break;
      }

      # 将当前差异对象添加到链表中
      if (*firstdiffp)
    (*lastdiffp)->generic.next = diff;
      else
        *firstdiffp = diff;
      *lastdiffp = diff;
      diff->generic.next = NULL;
    }
    }
  }

  # 返回全局状态的关闭标签函数的结果
  return state->global->close_tag(state);
}

int
hwloc__xml_import_diff(hwloc__xml_import_state_t state,
               hwloc_topology_diff_t *firstdiffp)
{
  hwloc_topology_diff_t firstdiff = NULL, lastdiff = NULL;
  *firstdiffp = NULL;

  while (1) {
    struct hwloc__xml_import_state_s childstate;
    char *tag;
    int ret;

    ret = state->global->find_child(state, &childstate, &tag);  // 查找子节点
    if (ret < 0)
      return -1;
    if (!ret)
      break;

    if (!strcmp(tag, "diff")) {  // 如果标签是 "diff"
      ret = hwloc__xml_import_diff_one(&childstate, &firstdiff, &lastdiff);  // 导入差异
    } else
      ret = -1;

    if (ret < 0)
      return ret;

    state->global->close_child(&childstate);  // 关闭子节点
  }

  *firstdiffp = firstdiff;  // 将第一个差异指针赋值给传入的指针
  return 0;
}

/***********************************
 ********* main XML import *********
 ***********************************/

static void
hwloc_convert_from_v1dist_floats(hwloc_topology_t topology, unsigned nbobjs, float *floats, uint64_t *u64s)
{
  unsigned i;
  int is_uint;
  char *env;
  float scale = 1000.f;
  char scalestring[20];

  env = getenv("HWLOC_XML_V1DIST_SCALE");  // 获取环境变量
  if (env) {
    scale = (float) atof(env);  // 将环境变量转换为浮点数
    goto scale;
  }

  is_uint = 1;
  /* find out if all values are integers */
  for(i=0; i<nbobjs*nbobjs; i++) {
    float f, iptr, fptr;
    f = floats[i];
    if (f < 0.f) {
      is_uint = 0;
      break;
    }
    fptr = modff(f, &iptr);
    if (fptr > .001f && fptr < .999f) {
      is_uint = 0;
      break;
    }
    u64s[i] = (int)(f+.5f);
  }
  if (is_uint)
    return;

 scale:
  /* TODO heuristic to find a good scale */
  for(i=0; i<nbobjs*nbobjs; i++)
    u64s[i] = (uint64_t)(scale * floats[i]);  // 将浮点数转换为64位无符号整数

  /* save the scale in root info attrs.
   * Not perfect since we may have multiple of them,
   * and some distances might disappear in case of restrict, etc.
   */
  sprintf(scalestring, "%f", scale);  // 将比例转换为字符串
  hwloc_obj_add_info(hwloc_get_root_obj(topology), "xmlv1DistancesScale", scalestring);  // 将比例添加到根信息属性中
}

/* this canNOT be the first XML call */
static int
  /*
   * 这个后端默认情况下强制 !topology->is_thissystem。
   */

  // 获取后端的拓扑结构
  struct hwloc_topology *topology = backend->topology;
  // 获取后端的私有数据
  struct hwloc_xml_backend_data_s *data = backend->private_data;
  // 定义状态和子状态
  struct hwloc__xml_import_state_s state, childstate;
  // 获取拓扑结构的根节点
  struct hwloc_obj *root = topology->levels[0][0];
  // 定义标签
  char *tag;
  // 初始化忽略标记
  int gotignored = 0;
  // 初始化本地化切换
  hwloc_localeswitch_declare;
  // 初始化返回值
  int ret;

  // 断言当前状态为全局发现阶段
  assert(dstatus->phase == HWLOC_DISC_PHASE_GLOBAL);

  // 设置全局状态
  state.global = data;

  // 断言根节点的 cpuset 不存在
  assert(!root->cpuset);

  // 初始化本地化切换
  hwloc_localeswitch_init();

  // 初始化私有数据的 numanodes 和 v1dist
  data->nbnumanodes = 0;
  data->first_numanode = data->last_numanode = NULL;
  data->first_v1dist = data->last_v1dist = NULL;

  // 初始化查找操作
  ret = data->look_init(data, &state);
  if (ret < 0)
    goto failed;

  // 如果版本大于2，则无法导入 XML 版本大于2
  if (data->version_major > 2) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: cannot import XML version %u.%u > 2\n",
          data->msgprefix, data->version_major, data->version_minor);
    goto err;
  }

  /* 找到根对象标签并导入它 */
  ret = state.global->find_child(&state, &childstate, &tag);
  if (ret < 0 || !ret || strcmp(tag, "object"))
    goto failed;
  ret = hwloc__xml_import_object(topology, data, NULL /*  no parent */, root,
                 &gotignored,
                 &childstate);
  if (ret < 0)
    goto failed;
  state.global->close_child(&childstate);
  assert(!gotignored);

  /* 如果我们不得不重新插入 Machine，则根可能已经改变 */
  root = topology->levels[0][0];

  // 如果版本大于等于2，则查找 v2 距离
  if (data->version_major >= 2) {
    /* 查找 v2 距离 */
    while (1) {
      ret = state.global->find_child(&state, &childstate, &tag);
      if (ret < 0)
    goto failed;
      if (!ret)
    break;
      if (!strcmp(tag, "distances2")) {
    ret = hwloc__xml_v2import_distances(topology, &childstate, 0);
    if (ret < 0)
      goto failed;
      } else if (!strcmp(tag, "distances2hetero")) {
    ret = hwloc__xml_v2import_distances(topology, &childstate, 1);
    # 如果返回值小于 0，则跳转到失败标签
    if (ret < 0)
      goto failed;
      # 如果标签与 "support" 不相等
      } else if (!strcmp(tag, "support")) {
    # 调用 hwloc__xml_v2import_support 函数导入支持信息，并更新子状态
    ret = hwloc__xml_v2import_support(topology, &childstate);
    # 如果返回值小于 0，则跳转到失败标签
    if (ret < 0)
      goto failed;
      # 如果标签与 "memattr" 不相等
      } else if (!strcmp(tag, "memattr")) {
        # 调用 hwloc__xml_import_memattr 函数导入内存属性，并更新子状态
        ret = hwloc__xml_import_memattr(topology, &childstate);
        # 如果返回值小于 0，则跳转到失败标签
        if (ret < 0)
          goto failed;
      # 如果标签与 "cpukind" 不相等
      } else if (!strcmp(tag, "cpukind")) {
        # 调用 hwloc__xml_import_cpukind 函数导入 CPU 类型，并更新子状态
        ret = hwloc__xml_import_cpukind(topology, &childstate);
        # 如果返回值小于 0，则跳转到失败标签
        if (ret < 0)
          goto failed;
      # 如果以上条件都不满足
      } else {
        # 如果启用了详细输出，则打印忽略未知标签的警告信息
        if (hwloc__xml_verbose())
          fprintf(stderr, "%s: ignoring unknown tag `%s' after root object.\n",
              data->msgprefix, tag);
        # 跳转到完成标签
        goto done;
      }
      # 调用全局状态的关闭子状态函数
      state.global->close_child(&childstate);
    }
  }

  # 查找拓扑标签的结束位置
  state.global->close_tag(&state);
  # 如果根对象没有 cpuset，则输出错误信息并跳转到错误处理部分
  if (!root->cpuset) {
    if (hwloc__xml_verbose())
      fprintf(stderr, "%s: invalid root object without cpuset\n",
          data->msgprefix);
    goto err;
  }

  # 如果版本号小于 2 且存在第一个 NUMA 节点，则更新先前版本的内存组 gp_index
  if (data->version_major < 2 && data->first_numanode) {
    hwloc_obj_t node = data->first_numanode;
    do {
      # 如果节点的父对象是 GROUP 类型且 gp_index 为 0，则设置 gp_index 为下一个可用的索引值
      if (node->parent->type == HWLOC_OBJ_GROUP
      && !node->parent->gp_index)
    node->parent->gp_index = topology->next_gp_index++;
      node = node->next_cousin;
    } while (node);
  }

  # 如果版本号小于 2 且存在第一个 v1dist，则处理 v1 距离
  if (data->version_major < 2 && data->first_v1dist) {
    # 处理 v1 距离
    struct hwloc__xml_imported_v1distances_s *v1dist, *v1next = data->first_v1dist;
    while ((v1dist = v1next) != NULL) {
      unsigned nbobjs = v1dist->nbobjs;
      v1next = v1dist->next;
      # 如果 nbobjs 与 nbnumanodes 相匹配且拓扑结构不禁用距离信息，则处理距离信息
      if (nbobjs == data->nbnumanodes
          && !(topology->flags & HWLOC_TOPOLOGY_FLAG_NO_DISTANCES)) {
    hwloc_obj_t *objs = malloc(nbobjs*sizeof(hwloc_obj_t));
    uint64_t *values = malloc(nbobjs*nbobjs*sizeof(*values));
        assert(data->nbnumanodes > 0); # 导入后 v1dist->nbobjs 大于 0
        assert(data->first_numanode);
    if (objs && values) {
      hwloc_obj_t node;
      unsigned i;
      for(i=0, node = data->first_numanode;
          i<nbobjs;
          i++, node = node->next_cousin)
        objs[i] = node;
      hwloc_convert_from_v1dist_floats(topology, nbobjs, v1dist->floats, values);
      hwloc_internal_distances_add(topology, NULL, nbobjs, objs, values, v1dist->kind, 0);
    } else {
      free(objs);
      free(values);
    }
      }
      // 释放v1dist结构体中的floats数组内存
      free(v1dist->floats);
      // 释放v1dist结构体内存
      free(v1dist);
    }
    // 将first_v1dist和last_v1dist指针置空
    data->first_v1dist = data->last_v1dist = NULL;
  }

  /* FIXME:
   * 我们应该检查现有对象集是否一致：
   * 同一级别的对象之间没有交集，
   * 对象集包含在父集中。
   * hwloc从未生成过这样有bug的XML，但用户可能会创建这样的XML。
   *
   * 我们希望将这些检查添加到现有的核心代码中，
   * 以添加丢失的集合并传播父/子集
   * （以防其他后端也生成有bug的对象集）。
   */

  if (data->version_major >= 2) {
    /* v2必须具有非空的nodesets，因为至少需要一个NUMA节点 */
    if (!root->nodeset) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: invalid root object without nodeset\n",
        data->msgprefix);
      goto err;
    }
    if (hwloc_bitmap_iszero(root->nodeset)) {
      if (hwloc__xml_verbose())
    fprintf(stderr, "%s: invalid root object with empty nodeset\n",
        data->msgprefix);
      goto err;
    }
  } else {
    /* 如果v1没有nodeset，则核心将添加默认的NUMA节点和nodesets */
  }

  /* 如果缺少默认cpusets和nodesets，则分配它们，核心将限制它们 */
  hwloc_alloc_root_sets(root);

  /* 保持“Backend”信息不变 */
  /* 我们可以添加“BackendSource=XML”来通知实际后端和此处之间使用了XML */

  if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT)) {
    topology->support.discovery->pu = 1;
    topology->support.discovery->disallowed_pu = 1;
    if (data->nbnumanodes) {
      topology->support.discovery->numa = 1;
      topology->support.discovery->numa_memory = 1; // FIXME
      topology->support.discovery->disallowed_numa = 1;
    }
  }

  if (data->look_done)
    data->look_done(data, 0);

  hwloc_localeswitch_fini();
  return 0;

 failed:
  if (data->look_done)
    data->look_done(data, -1);
  if (hwloc__xml_verbose())
    // 输出错误信息到标准错误流，包含消息前缀
    fprintf(stderr, "%s: XML component discovery failed.\n",
        data->msgprefix);
 err:
    // 释放根对象的子对象及其兄弟对象
    hwloc_free_object_siblings_and_children(root->first_child);
    root->first_child = NULL;
    // 释放根对象的内存子对象及其兄弟对象
    hwloc_free_object_siblings_and_children(root->memory_first_child);
    root->memory_first_child = NULL;
    // 释放根对象的 I/O 子对象及其兄弟对象
    hwloc_free_object_siblings_and_children(root->io_first_child);
    root->io_first_child = NULL;
    // 释放根对象的杂项子对象及其兄弟对象
    hwloc_free_object_siblings_and_children(root->misc_first_child);
    root->misc_first_child = NULL;

    /* 确保核心会中止 */
    // 如果根对象的 cpuset 存在，则将其清零
    if (root->cpuset)
        hwloc_bitmap_zero(root->cpuset);
    // 如果根对象的 nodeset 存在，则将其清零
    if (root->nodeset)
        hwloc_bitmap_zero(root->nodeset);

    // 结束本地化开关
    hwloc_localeswitch_fini();
    // 返回错误码 -1
    return -1;
}
/* this can be the first XML call */
int
hwloc_topology_diff_load_xml(const char *xmlpath,
                 hwloc_topology_diff_t *firstdiffp, char **refnamep)
{
  struct hwloc__xml_import_state_s state;
  struct hwloc_xml_backend_data_s fakedata; /* only for storing global info during parsing */
  hwloc_localeswitch_declare;
  const char *local_basename;
  int force_nolibxml;
  int ret;

  state.global = &fakedata;

  local_basename = strrchr(xmlpath, '/');  // 获取文件路径中最后一个 '/' 后的字符串
  if (local_basename)
    local_basename++;
  else
    local_basename = xmlpath;
  fakedata.msgprefix = strdup(local_basename);  // 复制文件路径中最后一个 '/' 后的字符串

  hwloc_components_init();  // 初始化组件
  assert(hwloc_nolibxml_callbacks);  // 断言 nolibxml 回调函数存在

  hwloc_localeswitch_init();  // 初始化本地化开关

  *firstdiffp = NULL;  // 将 firstdiffp 指向的值设为 NULL

  force_nolibxml = hwloc_nolibxml_import();  // 强制使用 nolibxml

retry:
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))  // 如果 libxml 回调函数不存在或者强制使用 nolibxml
    ret = hwloc_nolibxml_callbacks->import_diff(&state, xmlpath, NULL, 0, firstdiffp, refnamep);  // 调用 nolibxml 回调函数的 import_diff 方法
  else {
    ret = hwloc_libxml_callbacks->import_diff(&state, xmlpath, NULL, 0, firstdiffp, refnamep);  // 调用 libxml 回调函数的 import_diff 方法
    if (ret < 0 && errno == ENOSYS) {
      hwloc_libxml_callbacks = NULL;
      goto retry;  // 如果返回值小于 0 并且错误码为 ENOSYS，则将 libxml 回调函数置为 NULL，重新尝试
    }
  }

  hwloc_localeswitch_fini();  // 结束本地化开关
  hwloc_components_fini();  // 结束组件
  free(fakedata.msgprefix);  // 释放 fakedata.msgprefix 的内存
  return ret;  // 返回 ret 的值
}

/* this can be the first XML call */
int
hwloc_topology_diff_load_xmlbuffer(const char *xmlbuffer, int buflen,
                   hwloc_topology_diff_t *firstdiffp, char **refnamep)
{
  struct hwloc__xml_import_state_s state;
  struct hwloc_xml_backend_data_s fakedata; /* only for storing global info during parsing */
  hwloc_localeswitch_declare;
  int force_nolibxml;
  int ret;

  state.global = &fakedata;
  fakedata.msgprefix = strdup("xmldiffbuffer");  // 复制 "xmldiffbuffer" 字符串

  hwloc_components_init();  // 初始化组件
  assert(hwloc_nolibxml_callbacks);  // 断言 nolibxml 回调函数存在

  hwloc_localeswitch_init();  // 初始化本地化开关

  *firstdiffp = NULL;  // 将 firstdiffp 指向的值设为 NULL

  force_nolibxml = hwloc_nolibxml_import();  // 强制使用 nolibxml

 retry:
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    # 使用 hwloc_nolibxml_callbacks 对象调用 import_diff 方法，传入相应参数，并将结果赋给 ret
    ret = hwloc_nolibxml_callbacks->import_diff(&state, NULL, xmlbuffer, buflen, firstdiffp, refnamep);
  else {
    # 否则，使用 hwloc_libxml_callbacks 对象调用 import_diff 方法，传入相应参数，并将结果赋给 ret
    ret = hwloc_libxml_callbacks->import_diff(&state, NULL, xmlbuffer, buflen, firstdiffp, refnamep);
    # 如果返回值小于 0 并且错误码为 ENOSYS
    if (ret < 0 && errno == ENOSYS) {
      # 将 hwloc_libxml_callbacks 对象置为 NULL
      hwloc_libxml_callbacks = NULL;
      # 跳转到 retry 标签处重新执行
      goto retry;
    }
  }

  # 结束本地化环境
  hwloc_localeswitch_fini();
  # 结束组件环境
  hwloc_components_fini();
  # 释放 fakedata.msgprefix 所指向的内存
  free(fakedata.msgprefix);
  # 返回 ret 的值
  return ret;
}
/************************************************
 ********* XML export (common routines) *********
 ************************************************/

// 定义宏，用于检查字符是否为有效的 XML 字符
#define HWLOC_XML_CHAR_VALID(c) (((c) >= 32 && (c) <= 126) || (c) == '\t' || (c) == '\n' || (c) == '\r')

// 检查缓冲区中的字符是否为有效的 XML 字符
static int
hwloc__xml_export_check_buffer(const char *buf, size_t length)
{
  unsigned i;
  for(i=0; i<length; i++)
    if (!HWLOC_XML_CHAR_VALID(buf[i]))
      return -1;
  return 0;
}

// 复制字符串并移除难看的字符
static char*
hwloc__xml_export_safestrdup(const char *old)
{
  char *new = malloc(strlen(old)+1);
  char *dst = new;
  const char *src = old;
  if (!new)
    return NULL;

  while (*src) {
    if (HWLOC_XML_CHAR_VALID(*src))
      *(dst++) = *src;
    src++;
  }
  *dst = '\0';
  return new;
}

// 导出对象的内容到 XML
static void
hwloc__xml_export_object_contents (hwloc__xml_export_state_t state, hwloc_topology_t topology, hwloc_obj_t obj, unsigned long flags)
{
  char *setstring = NULL, *setstring2 = NULL;
  char tmp[255];
  int v1export = flags & HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1;
  unsigned i,j;

  // 根据标志和对象类型设置属性
  if (v1export && obj->type == HWLOC_OBJ_PACKAGE)
    state->new_prop(state, "type", "Socket");
  else if (v1export && obj->type == HWLOC_OBJ_DIE)
    state->new_prop(state, "type", "Group");
  else if (v1export && hwloc__obj_type_is_cache(obj->type))
    state->new_prop(state, "type", "Cache");
  else
    state->new_prop(state, "type", hwloc_obj_type_string(obj->type));

  // 如果对象的 os_index 不是未知的索引，则设置属性
  if (obj->os_index != HWLOC_UNKNOWN_INDEX) {
    sprintf(tmp, "%u", obj->os_index);
    state->new_prop(state, "os_index", tmp);
  }

  // 如果对象有 cpuset，则设置属性
  if (obj->cpuset) {
    int empty_cpusets = 0;

    if (v1export && obj->type == HWLOC_OBJ_NUMANODE) {
      // 遍历内存层次结构，找出是否是第一个 NUMA 节点
      // v1 非第一个 NUMA 节点具有空的 cpuset
      hwloc_obj_t parent = obj;
      while (!hwloc_obj_type_is_normal(parent->type)) {
    # 如果父节点的兄弟节点排名大于0
    if (parent->sibling_rank > 0) {
      # 设置空的 CPU 集合标志为1，并跳出循环
      empty_cpusets = 1;
      break;
    }
    # 将当前节点的父节点设置为父节点的父节点
    parent = parent->parent;
      }
    }

    # 如果存在空的 CPU 集合
    if (empty_cpusets) {
      # 设置新的属性值为 "0x0"
      state->new_prop(state, "cpuset", "0x0");
      state->new_prop(state, "online_cpuset", "0x0");
      state->new_prop(state, "complete_cpuset", "0x0");
      state->new_prop(state, "allowed_cpuset", "0x0");

    } else {
      # 正常情况
      # 将 obj 的 cpuset 转换为字符串，并设置为新的属性值
      hwloc_bitmap_asprintf(&setstring, obj->cpuset);
      state->new_prop(state, "cpuset", setstring);
      # 释放 setstring2 的内存
      hwloc_bitmap_asprintf(&setstring2, obj->complete_cpuset);
      state->new_prop(state, "complete_cpuset", setstring2);
      free(setstring2);

      # 如果是 v1 导出
      if (v1export)
    state->new_prop(state, "online_cpuset", setstring);
      free(setstring);

      # 如果是 v1 导出
      if (v1export) {
    # 复制 obj 的 cpuset，并与 topology->allowed_cpuset 求交集
    hwloc_bitmap_t allowed_cpuset = hwloc_bitmap_dup(obj->cpuset);
    hwloc_bitmap_and(allowed_cpuset, allowed_cpuset, topology->allowed_cpuset);
    # 将 allowed_cpuset 转换为字符串，并设置为新的属性值
    hwloc_bitmap_asprintf(&setstring, allowed_cpuset);
    state->new_prop(state, "allowed_cpuset", setstring);
    free(setstring);
    # 释放 allowed_cpuset 的内存
    hwloc_bitmap_free(allowed_cpuset);
      } else if (!obj->parent) {
    # 将 topology->allowed_cpuset 转换为字符串，并设置为新的属性值
    hwloc_bitmap_asprintf(&setstring, topology->allowed_cpuset);
    state->new_prop(state, "allowed_cpuset", setstring);
    free(setstring);
      }
    }

    # 将 obj 的 nodeset 转换为字符串，并设置为新的属性值
    hwloc_bitmap_asprintf(&setstring, obj->nodeset);
    state->new_prop(state, "nodeset", setstring);
    free(setstring);

    # 将 obj 的 complete_nodeset 转换为字符串，并设置为新的属性值
    hwloc_bitmap_asprintf(&setstring, obj->complete_nodeset);
    state->new_prop(state, "complete_nodeset", setstring);
    free(setstring);
    # 如果 v1export 为真
    if (v1export) {
      # 复制对象的节点集合
      hwloc_bitmap_t allowed_nodeset = hwloc_bitmap_dup(obj->nodeset);
      # 对节点集合进行与操作，得到允许的节点集合
      hwloc_bitmap_and(allowed_nodeset, allowed_nodeset, topology->allowed_nodeset);
      # 将允许的节点集合格式化为字符串
      hwloc_bitmap_asprintf(&setstring, allowed_nodeset);
      # 将允许的节点集合添加到状态对象的属性中
      state->new_prop(state, "allowed_nodeset", setstring);
      # 释放字符串内存
      free(setstring);
      # 释放允许的节点集合内存
      hwloc_bitmap_free(allowed_nodeset);
    } 
    # 如果 v1export 为假且对象没有父对象
    else if (!obj->parent) {
      # 将允许的节点集合格式化为字符串
      hwloc_bitmap_asprintf(&setstring, topology->allowed_nodeset);
      # 将允许的节点集合添加到状态对象的属性中
      state->new_prop(state, "allowed_nodeset", setstring);
      # 释放字符串内存
      free(setstring);
    }
  }

  # 如果 v1export 为假
  if (!v1export) {
    # 将对象的 gp_index 转换为字符串格式
    sprintf(tmp, "%llu", (unsigned long long) obj->gp_index);
    # 将 gp_index 添加到状态对象的属性中
    state->new_prop(state, "gp_index", tmp);
  }

  # 如果对象有名称
  if (obj->name) {
    # 安全地复制对象的名称
    char *name = hwloc__xml_export_safestrdup(obj->name);
    # 如果名称不为空
    if (name) {
      # 将名称添加到状态对象的属性中
      state->new_prop(state, "name", name);
      # 释放名称内存
      free(name);
    }
  }
  # 如果 v1export 为假且对象有子类型
  if (!v1export && obj->subtype) {
    # 安全地复制对象的子类型
    char *subtype = hwloc__xml_export_safestrdup(obj->subtype);
    # 如果子类型不为空
    if (subtype) {
      # 将子类型添加到状态对象的属性中
      state->new_prop(state, "subtype", subtype);
      # 释放子类型内存
      free(subtype);
    }
  }

  # 根据对象的类型进行不同的处理
  switch (obj->type) {
  # 如果对象类型为 NUMANODE
  case HWLOC_OBJ_NUMANODE:
    # 如果对象的 NUMANODE 属性中有本地内存
    if (obj->attr->numanode.local_memory) {
      # 将本地内存转换为字符串格式
      sprintf(tmp, "%llu", (unsigned long long) obj->attr->numanode.local_memory);
      # 将本地内存添加到状态对象的属性中
      state->new_prop(state, "local_memory", tmp);
    }
    # 遍历对象的 NUMANODE 属性中的页面类型
    for(i=0; i<obj->attr->numanode.page_types_len; i++) {
      # 创建一个新的子状态对象
      struct hwloc__xml_export_state_s childstate;
      # 在状态对象中创建一个新的子对象，并将子状态对象传入
      state->new_child(state, &childstate, "page_type");
      # 将页面大小转换为字符串格式
      sprintf(tmp, "%llu", (unsigned long long) obj->attr->numanode.page_types[i].size);
      # 将页面大小添加到子状态对象的属性中
      childstate.new_prop(&childstate, "size", tmp);
      # 将页面数量转换为字符串格式
      sprintf(tmp, "%llu", (unsigned long long) obj->attr->numanode.page_types[i].count);
      # 将页面数量添加到子状态对象的属性中
      childstate.new_prop(&childstate, "count", tmp);
      # 结束子状态对象的创建
      childstate.end_object(&childstate, "page_type");
    }
    # 在此处添加代码，表示在此处有一条break语句
    break;
  # 如果对象是L1缓存、L2缓存、L3缓存、L4缓存、L5缓存、L1指令缓存、L2指令缓存、L3指令缓存或内存缓存
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
  case HWLOC_OBJ_MEMCACHE:
    # 将缓存大小转换为字符串并存储在tmp中
    sprintf(tmp, "%llu", (unsigned long long) obj->attr->cache.size);
    # 将缓存大小添加到状态中
    state->new_prop(state, "cache_size", tmp);
    # 将缓存深度转换为字符串并存储在tmp中
    sprintf(tmp, "%u", obj->attr->cache.depth);
    # 将缓存深度添加到状态中
    state->new_prop(state, "depth", tmp);
    # 将缓存行大小转换为字符串并存储在tmp中
    sprintf(tmp, "%u", (unsigned) obj->attr->cache.linesize);
    # 将缓存行大小添加到状态中
    state->new_prop(state, "cache_linesize", tmp);
    # 将缓存关联度转换为字符串并存储在tmp中
    sprintf(tmp, "%d", obj->attr->cache.associativity);
    # 将缓存关联度添加到状态中
    state->new_prop(state, "cache_associativity", tmp);
    # 将缓存类型转换为字符串并存储在tmp中
    sprintf(tmp, "%d", (int) obj->attr->cache.type);
    # 将缓存类型添加到状态中
    state->new_prop(state, "cache_type", tmp);
    # 结束当前case的处理
    break;
  # 如果对象是组
  case HWLOC_OBJ_GROUP:
    # 如果是v1版本的导出
    if (v1export) {
      # 将组深度转换为字符串并存储在tmp中
      sprintf(tmp, "%u", obj->attr->group.depth);
      # 将组深度添加到状态中
      state->new_prop(state, "depth", tmp);
      # 如果组不合并，则将dont_merge属性添加到状态中
      if (obj->attr->group.dont_merge)
        state->new_prop(state, "dont_merge", "1");
    } else {
      # 将组种类转换为字符串并存储在tmp中
      sprintf(tmp, "%u", obj->attr->group.kind);
      # 将组种类添加到状态中
      state->new_prop(state, "kind", tmp);
      # 将组子种类转换为字符串并存储在tmp中
      sprintf(tmp, "%u", obj->attr->group.subkind);
      # 将组子种类添加到状态中
      state->new_prop(state, "subkind", tmp);
      # 如果组不合并，则将dont_merge属性添加到状态中
      if (obj->attr->group.dont_merge)
        state->new_prop(state, "dont_merge", "1");
    }
    # 结束当前case的处理
    break;
  # 如果对象是桥
  case HWLOC_OBJ_BRIDGE:
    # 将上游类型和下游类型转换为字符串并存储在tmp中
    sprintf(tmp, "%d-%d", (int) obj->attr->bridge.upstream_type, (int) obj->attr->bridge.downstream_type);
    # 将桥类型添加到状态中
    state->new_prop(state, "bridge_type", tmp);
    # 将桥深度转换为字符串并存储在tmp中
    sprintf(tmp, "%u", obj->attr->bridge.depth);
    # 将桥深度添加到状态中
    state->new_prop(state, "depth", tmp);
    # 如果下游类型是PCI桥
    if (obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) {
      # 将PCI桥信息转换为字符串并存储在tmp中
      sprintf(tmp, "%04x:[%02x-%02x]",
          (unsigned) obj->attr->bridge.downstream.pci.domain,
          (unsigned) obj->attr->bridge.downstream.pci.secondary_bus,
          (unsigned) obj->attr->bridge.downstream.pci.subordinate_bus);
      # 将PCI桥信息添加到状态中
      state->new_prop(state, "bridge_pci", tmp);
    }
    // 如果对象的上游类型不是 PCI 桥，则跳出循环
    if (obj->attr->bridge.upstream_type != HWLOC_OBJ_BRIDGE_PCI)
      break;
    // 如果上一个条件不成立，则继续执行下面的代码
    /* FALLTHRU */
  case HWLOC_OBJ_PCI_DEVICE:
    // 格式化输出 PCI 设备的地址信息，并存储到 tmp 变量中
    sprintf(tmp, "%04x:%02x:%02x.%01x",
        (unsigned) obj->attr->pcidev.domain,
        (unsigned) obj->attr->pcidev.bus,
        (unsigned) obj->attr->pcidev.dev,
        (unsigned) obj->attr->pcidev.func);
    // 将 PCI 设备的地址信息存储为属性
    state->new_prop(state, "pci_busid", tmp);
    // 格式化输出 PCI 设备的类型信息，并存储到 tmp 变量中
    sprintf(tmp, "%04x [%04x:%04x] [%04x:%04x] %02x",
        (unsigned) obj->attr->pcidev.class_id,
        (unsigned) obj->attr->pcidev.vendor_id, (unsigned) obj->attr->pcidev.device_id,
        (unsigned) obj->attr->pcidev.subvendor_id, (unsigned) obj->attr->pcidev.subdevice_id,
        (unsigned) obj->attr->pcidev.revision);
    // 将 PCI 设备的类型信息存储为属性
    state->new_prop(state, "pci_type", tmp);
    // 格式化输出 PCI 设备的链接速度信息，并存储到 tmp 变量中
    sprintf(tmp, "%f", obj->attr->pcidev.linkspeed);
    // 将 PCI 设备的链接速度信息存储为属性
    state->new_prop(state, "pci_link_speed", tmp);
    // 跳出 switch 语句
    break;
  // 如果对象类型是 OS 设备
  case HWLOC_OBJ_OS_DEVICE:
    // 格式化输出 OS 设备的类型信息，并存储到 tmp 变量中
    sprintf(tmp, "%d", (int) obj->attr->osdev.type);
    // 将 OS 设备的类型信息存储为属性
    state->new_prop(state, "osdev_type", tmp);
    // 跳出 switch 语句
    break;
  // 如果对象类型不是以上任何一种
  default:
    // 跳出 switch 语句
    break;
  }

  // 遍历对象的信息数量
  for(i=0; i<obj->infos_count; i++) {
    // 复制信息的名称和值到 name 和 value 变量中
    char *name = hwloc__xml_export_safestrdup(obj->infos[i].name);
    char *value = hwloc__xml_export_safestrdup(obj->infos[i].value);
    // 如果 name 和 value 都不为空
    if (name && value) {
      // 创建一个新的子状态对象
      struct hwloc__xml_export_state_s childstate;
      state->new_child(state, &childstate, "info");
      // 将信息的名称和值存储为属性
      childstate.new_prop(&childstate, "name", name);
      childstate.new_prop(&childstate, "value", value);
      // 结束当前子对象的处理
      childstate.end_object(&childstate, "info");
    }
    // 释放 name 和 value 变量的内存
    free(name);
    free(value);
  }
  // 如果 v1export 为真且对象有子类型
  if (v1export && obj->subtype) {
    // 复制子类型到 subtype 变量中
    char *subtype = hwloc__xml_export_safestrdup(obj->subtype);
    # 如果存在子类型
    if (subtype) {
      # 创建子状态对象
      struct hwloc__xml_export_state_s childstate;
      # 判断是否为协处理器类型
      int is_coproctype = (obj->type == HWLOC_OBJ_OS_DEVICE && obj->attr->osdev.type == HWLOC_OBJ_OSDEV_COPROC);
      # 创建子节点
      state->new_child(state, &childstate, "info");
      # 添加属性名和值
      childstate.new_prop(&childstate, "name", is_coproctype ? "CoProcType" : "Type");
      childstate.new_prop(&childstate, "value", subtype);
      # 结束子对象
      childstate.end_object(&childstate, "info");
      # 释放子类型内存
      free(subtype);
    }
  }
  # 如果是 v1 导出并且对象类型为 HWLOC_OBJ_DIE
  if (v1export && obj->type == HWLOC_OBJ_DIE) {
    # 创建子状态对象
    struct hwloc__xml_export_state_s childstate;
    # 创建子节点
    state->new_child(state, &childstate, "info");
    # 添加属性名和值
    childstate.new_prop(&childstate, "name", "Type");
    childstate.new_prop(&childstate, "value", "Die");
    # 结束子对象
    childstate.end_object(&childstate, "info");
  }

  # 如果是 v1 导出并且没有父对象
  if (v1export && !obj->parent) {
    # 只有覆盖整个机器的延迟矩阵才能导出到 v1
    struct hwloc_internal_distances_s *dist;
    # 刷新距离，因为我们需要下面的对象
    hwloc_internal_distances_refresh(topology);
    # 遍历距离对象
    for(dist = topology->first_dist; dist; dist = dist->next) {
      # 创建子状态对象
      struct hwloc__xml_export_state_s childstate;
      # 获取对象数量和逻辑到 v2 数组
      unsigned nbobjs = dist->nbobjs;
      unsigned *logical_to_v2array;
      int depth;

      # 如果对象数量不等于相同类型的对象数量，则跳过
      if (nbobjs != (unsigned) hwloc_get_nbobjs_by_type(topology, dist->unique_type))
    continue;
      # 如果不是延迟的类型，则跳过
      if (!(dist->kind & HWLOC_DISTANCES_KIND_MEANS_LATENCY))
    continue;
      # 如果是异构类型，则跳过
      if (dist->kind & HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES)
    continue;

      # 分配逻辑到 v2 数组的内存
      logical_to_v2array = malloc(nbobjs * sizeof(*logical_to_v2array));
      # 如果分配失败，则打印错误信息并跳过
      if (!logical_to_v2array) {
        if (HWLOC_SHOW_ALL_ERRORS())
          fprintf(stderr, "hwloc/xml/export/v1: failed to allocated logical_to_v2array\n");
    continue;
      }

      # 计算相对深度
      for(i=0; i<nbobjs; i++)
    logical_to_v2array[dist->objs[i]->logical_index] = i;

      # 计算相对深度
      if (dist->unique_type == HWLOC_OBJ_NUMANODE) {
    /* for NUMA nodes, use the highest normal-parent depth + 1 */
    // 对于 NUMA 节点，使用最高正常父级深度 + 1
    depth = -1;
    for(i=0; i<nbobjs; i++) {
      hwloc_obj_t parent = dist->objs[i]->parent;
      while (hwloc__obj_type_is_memory(parent->type))
        parent = parent->parent;
      if (parent->depth+1 > depth)
        depth = parent->depth+1;
    }
      } else {
    /* for non-NUMA nodes, increase the object depth if any of them has memory above */
    // 对于非 NUMA 节点，如果它们中的任何一个具有上面的内存，则增加对象深度
    int parent_with_memory = 0;
    for(i=0; i<nbobjs; i++) {
      hwloc_obj_t parent = dist->objs[i]->parent;
      while (parent) {
        if (parent->memory_first_child) {
          parent_with_memory = 1;
          goto done;
        }
        parent = parent->parent;
      }
    }
      done:
    depth = hwloc_get_type_depth(topology, dist->unique_type) + parent_with_memory;
      }

      state->new_child(state, &childstate, "distances");
      sprintf(tmp, "%u", nbobjs);
      childstate.new_prop(&childstate, "nbobjs", tmp);
      sprintf(tmp, "%d", depth);
      childstate.new_prop(&childstate, "relative_depth", tmp);
      sprintf(tmp, "%f", 1.f);
      childstate.new_prop(&childstate, "latency_base", tmp);
      for(i=0; i<nbobjs; i++) {
        for(j=0; j<nbobjs; j++) {
      /* we should export i*nbobjs+j, we translate using logical_to_v2array[] */
      // 我们应该导出 i*nbobjs+j，我们使用 logical_to_v2array[] 进行转换
      unsigned k = logical_to_v2array[i]*nbobjs+logical_to_v2array[j];
      struct hwloc__xml_export_state_s greatchildstate;
      childstate.new_child(&childstate, &greatchildstate, "latency");
      sprintf(tmp, "%f", (float) dist->values[k]);
      greatchildstate.new_prop(&greatchildstate, "value", tmp);
      greatchildstate.end_object(&greatchildstate, "latency");
    }
      }
      childstate.end_object(&childstate, "distances");
      free(logical_to_v2array);
    }
  }

  if (obj->userdata && topology->userdata_export_cb)
    topology->userdata_export_cb((void*) state, topology, obj);
    // 如果对象具有用户数据并且拓扑结构具有用户数据导出回调函数，则调用该回调函数
}
# 定义一个静态函数，用于导出 XML 格式的对象
static void
hwloc__xml_v2export_object (hwloc__xml_export_state_t parentstate, hwloc_topology_t topology, hwloc_obj_t obj, unsigned long flags)
{
  # 创建一个新的子状态，表示一个对象
  struct hwloc__xml_export_state_s state;
  parentstate->new_child(parentstate, &state, "object");

  # 导出对象的内容
  hwloc__xml_export_object_contents(&state, topology, obj, flags);

  # 遍历对象的内存子对象，导出每个子对象
  for_each_memory_child(child, obj)
    hwloc__xml_v2export_object (&state, topology, child, flags);
  # 遍历对象的普通子对象，导出每个子对象
  for_each_child(child, obj)
    hwloc__xml_v2export_object (&state, topology, child, flags);
  # 遍历对象的 I/O 子对象，导出每个子对象
  for_each_io_child(child, obj)
    hwloc__xml_v2export_object (&state, topology, child, flags);
  # 遍历对象的其他子对象，导出每个子对象
  for_each_misc_child(child, obj)
    hwloc__xml_v2export_object (&state, topology, child, flags);

  # 结束当前对象的导出
  state.end_object(&state, "object");
}

# 定义一个静态函数，用于导出 XML 格式的对象
static void
hwloc__xml_v1export_object (hwloc__xml_export_state_t parentstate, hwloc_topology_t topology, hwloc_obj_t obj, unsigned long flags);

# 导出对象的下一个 NUMA 节点
static hwloc_obj_t
hwloc__xml_v1export_object_next_numanode(hwloc_obj_t obj, hwloc_obj_t cur)
{
  hwloc_obj_t parent;

  if (!cur) {
    # 第一个 NUMA 节点位于最底部的左侧
    cur = obj->memory_first_child;
    goto find_first;
  }

  # 向上遍历，直到找到下一个兄弟节点
  parent = cur;
  while (1) {
    if (parent->next_sibling) {
      # 找到下一个兄弟节点，从那里向下遍历左侧
      cur = parent->next_sibling;
      break;
    }
    parent = parent->parent;
    if (parent == obj)
      return NULL;
  }

 find_first:
  while (cur->type != HWLOC_OBJ_NUMANODE)
    cur = cur->memory_first_child;
  assert(cur);
  return cur;
}

# 列出对象的 NUMA 节点
static unsigned
hwloc__xml_v1export_object_list_numanodes(hwloc_obj_t obj, hwloc_obj_t *first_p, hwloc_obj_t **nodes_p)
{
  hwloc_obj_t *nodes, cur;
  int nr;

  if (!obj->memory_first_child) {
    *first_p = NULL;
    *nodes_p = NULL;
    # 返回值为0
    return 0;
  }
  # 确保至少有一个 NUMA 节点
  /* we're sure there's at least one numa node */
  nr = hwloc_bitmap_weight(obj->nodeset);
  assert(nr > 0);
  /* 这些是本地节点，但其中一些可能是连接在上面而不是在这里 */
  /* these are local nodes, but some of them may be attached above instead of here */
  # 分配内存以存储节点指针
  nodes = calloc(nr, sizeof(*nodes));
  if (!nodes) {
    /* 只返回第一个节点 */
    /* only return the first node */
    cur = hwloc__xml_v1export_object_next_numanode(obj, NULL);
    assert(cur);
    *first_p = cur;
    *nodes_p = NULL;
    return 1;
  }
  # 重置 nr 的值
  nr = 0;
  cur = NULL;
  # 循环遍历节点，直到遍历完所有节点
  while (1) {
    cur = hwloc__xml_v1export_object_next_numanode(obj, cur);
    if (!cur)
      break;
    nodes[nr++] = cur;
  }
  # 将第一个节点的指针赋给 first_p
  *first_p = nodes[0];
  # 将节点数组的指针赋给 nodes_p
  *nodes_p = nodes;
  # 返回节点数量
  return nr;
static void
hwloc__xml_v1export_object_with_memory(hwloc__xml_export_state_t parentstate, hwloc_topology_t topology, hwloc_obj_t obj, unsigned long flags)
{
  struct hwloc__xml_export_state_s gstate, mstate, ostate, *state = parentstate;  // 定义导出状态结构体和指针state
  hwloc_obj_t child;  // 定义对象子节点
  unsigned nr_numanodes;  // 定义NUMA节点数量
  hwloc_obj_t *numanodes, first_numanode;  // 定义NUMA节点指针和第一个NUMA节点
  unsigned i;  // 定义循环变量i

  nr_numanodes = hwloc__xml_v1export_object_list_numanodes(obj, &first_numanode, &numanodes);  // 获取对象的NUMA节点数量和指针

  if (obj->parent->arity > 1 && nr_numanodes > 1 && parentstate->global->v1_memory_group) {  // 判断对象的父节点arity大于1且NUMA节点数量大于1且存在v1_memory_group
    /* child has sibling, we must add a Group around those memory children */
    hwloc_obj_t group = parentstate->global->v1_memory_group;  // 获取v1_memory_group
    parentstate->new_child(parentstate, &gstate, "object");  // 创建新的子节点
    group->parent = obj->parent;  // 设置group的父节点
    group->cpuset = obj->cpuset;  // 设置group的cpuset
    group->complete_cpuset = obj->complete_cpuset;  // 设置group的complete_cpuset
    group->nodeset = obj->nodeset;  // 设置group的nodeset
    group->complete_nodeset = obj->complete_nodeset;  // 设置group的complete_nodeset
    hwloc__xml_export_object_contents (&gstate, topology, group, flags);  // 导出group的内容
    group->cpuset = NULL;  // 清空group的cpuset
    group->complete_cpuset = NULL;  // 清空group的complete_cpuset
    group->nodeset = NULL;  // 清空group的nodeset
    group->complete_nodeset = NULL;  // 清空group的complete_nodeset
    state = &gstate;  // 设置state为gstate
  }

  /* export first memory child */
  state->new_child(state, &mstate, "object");  // 创建新的子节点
  hwloc__xml_export_object_contents (&mstate, topology, first_numanode, flags);  // 导出第一个NUMA节点的内容

  /* then the actual object */
  mstate.new_child(&mstate, &ostate, "object");  // 创建新的子节点
  hwloc__xml_export_object_contents (&ostate, topology, obj, flags);  // 导出对象的内容

  /* then its normal/io/misc children */
  for_each_child(child, obj)  // 遍历对象的普通子节点
    hwloc__xml_v1export_object (&ostate, topology, child, flags);  // 导出普通子节点
  for_each_io_child(child, obj)  // 遍历对象的IO子节点
    hwloc__xml_v1export_object (&ostate, topology, child, flags);  // 导出IO子节点
  for_each_misc_child(child, obj)  // 遍历对象的misc子节点
    hwloc__xml_v1export_object (&ostate, topology, child, flags);  // 导出misc子节点

  /* close object and first memory child */
  ostate.end_object(&ostate, "object");  // 结束对象
  mstate.end_object(&mstate, "object");  // 结束第一个NUMA节点

  /* now other memory children */
  for(i=1; i<nr_numanodes; i++)  // 遍历其他NUMA节点
    # 调用函数将 NUMA 节点信息导出为 XML 格式
    hwloc__xml_v1export_object (state, topology, numanodes[i], flags);

  # 释放 NUMA 节点数组的内存
  free(numanodes);

  # 如果状态指针指向全局状态
  if (state == &gstate) {
    # 如果存在组对象，则关闭组对象
    gstate.end_object(&gstate, "object");
  }
# 定义一个静态函数，用于导出 XML 格式的对象
static void
hwloc__xml_v1export_object (hwloc__xml_export_state_t parentstate, hwloc_topology_t topology, hwloc_obj_t obj, unsigned long flags)
{
  # 定义一个状态结构体
  struct hwloc__xml_export_state_s state;
  # 定义一个对象指针
  hwloc_obj_t child;

  # 在父状态下创建一个新的子状态，类型为"object"
  parentstate->new_child(parentstate, &state, "object");

  # 导出对象的内容
  hwloc__xml_export_object_contents(&state, topology, obj, flags);

  # 遍历对象的每个子对象
  for_each_child(child, obj) {
    if (!child->memory_arity) {
      # 如果没有内存子对象，则正常导出
      hwloc__xml_v1export_object (&state, topology, child, flags);
    } else {
      # 如果有内存子对象，则带有内存信息导出
      hwloc__xml_v1export_object_with_memory(&state, topology, child, flags);
    }
  }

  # 遍历对象的每个 I/O 子对象，并导出
  for_each_io_child(child, obj)
    hwloc__xml_v1export_object (&state, topology, child, flags);
  # 遍历对象的每个杂项子对象，并导出
  for_each_misc_child(child, obj)
    hwloc__xml_v1export_object (&state, topology, child, flags);

  # 结束当前对象的导出
  state.end_object(&state, "object");
}

# 定义一个宏，用于导出数组
#define EXPORT_ARRAY(state, type, nr, values, tagname, format, maxperline) do { \
  unsigned _i = 0; \
  while (_i<(nr)) { \
    char _tmp[255]; /* 足够存放 (snprintf(format)+space) x maxperline 的空间 */ \
    char _tmp2[16]; \
    size_t _len = 0; \
    unsigned _j; \
    struct hwloc__xml_export_state_s _childstate; \
    # 在状态下创建一个新的子状态，类型为给定的标签名
    (state)->new_child(state, &_childstate, tagname); \
    # 遍历数组的每个元素，按照给定的格式导出
    for(_j=0; \
    _i+_j<(nr) && _j<maxperline; \
    _j++) \
      _len += sprintf(_tmp+_len, format " ", (type) (values)[_i+_j]); \
    _i += _j; \
    # 将长度转换为字符串，并添加为子状态的属性
    sprintf(_tmp2, "%lu", (unsigned long) _len); \
    _childstate.new_prop(&_childstate, "length", _tmp2); \
    # 将导出的内容添加到子状态中
    _childstate.add_content(&_childstate, _tmp, _len); \
    # 结束当前子状态的导出
    _childstate.end_object(&_childstate, tagname); \
  } \
} while (0)

# 定义一个宏，用于导出类型为 GPINDEX 的数组
#define EXPORT_TYPE_GPINDEX_ARRAY(state, nr, objs, tagname, maxperline) do { \
  unsigned _i = 0; \
  while (_i<(nr)) { \
    char _tmp[255]; /* 足够存放 (snprintf(type+index)+space) x maxperline 的空间 */ \
    char _tmp2[16]; \
    size_t _len = 0; \
    unsigned _j; \
    struct hwloc__xml_export_state_s _childstate; \
    # 在状态下创建一个新的子状态，类型为给定的标签名
    (state)->new_child(state, &_childstate, tagname); \
    # 循环开始，初始化_j为0
    for(_j=0; \
    # 循环条件：_i+_j小于nr并且_j小于maxperline
    _i+_j<(nr) && _j<maxperline; \
    # 每次循环_j自增，计算_len的值
    _j++) \
      # 将格式化后的字符串追加到_tmp中，包括对象类型和gp_index
      _len += sprintf(_tmp+_len, "%s:%llu ", hwloc_obj_type_string((objs)[_i+_j]->type), (unsigned long long) (objs)[_i+_j]->gp_index); \
    # _i增加_j的值
    _i += _j; \
    # 将_len转换为字符串并存储在_tmp2中
    sprintf(_tmp2, "%lu", (unsigned long) _len); \
    # 调用new_prop方法，添加"length"属性
    _childstate.new_prop(&_childstate, "length", _tmp2); \
    # 调用add_content方法，将_tmp中的内容添加到_childstate中
    _childstate.add_content(&_childstate, _tmp, _len); \
    # 调用end_object方法，结束对象的构建
    _childstate.end_object(&_childstate, tagname); \
  } \
} while (0)

这是一个do-while循环的结束标记，表示循环体结束。


static void
hwloc___xml_v2export_distances(hwloc__xml_export_state_t parentstate, struct hwloc_internal_distances_s *dist)

定义了一个静态函数hwloc___xml_v2export_distances，接受两个参数：一个是hwloc__xml_export_state_t类型的parentstate，另一个是struct hwloc_internal_distances_s类型的指针dist。


{
  char tmp[255];
  unsigned nbobjs = dist->nbobjs;
  struct hwloc__xml_export_state_s state;

声明了一个字符数组tmp，大小为255；声明了一个无符号整数nbobjs，赋值为dist->nbobjs；声明了一个结构体hwloc__xml_export_state_s类型的state。


  if (dist->different_types) {
    parentstate->new_child(parentstate, &state, "distances2hetero");
  } else {
    parentstate->new_child(parentstate, &state, "distances2");
    state.new_prop(&state, "type", hwloc_obj_type_string(dist->unique_type));
  }

根据dist->different_types的值，选择不同的标签名创建新的子节点，并设置属性type为dist->unique_type对应的字符串。


  sprintf(tmp, "%u", nbobjs);
  state.new_prop(&state, "nbobjs", tmp);
  sprintf(tmp, "%lu", dist->kind);
  state.new_prop(&state, "kind", tmp);
  if (dist->name)
    state.new_prop(&state, "name", dist->name);

将nbobjs和dist->kind转换为字符串，分别设置为属性nbobjs和kind的值；如果dist->name存在，则设置为属性name的值。


  if (!dist->different_types) {
    state.new_prop(&state, "indexing",
           HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type) ? "os" : "gp");
  }

如果dist->different_types为假，则根据HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type)的值设置属性indexing的值。


  /* TODO don't hardwire 10 below. either snprintf the max to guess it, or just append until the end of the buffer */
  if (dist->different_types) {
    EXPORT_TYPE_GPINDEX_ARRAY(&state, nbobjs, dist->objs, "indexes", 10);
  } else {
    EXPORT_ARRAY(&state, unsigned long long, nbobjs, dist->indexes, "indexes", "%llu", 10);
  }
  EXPORT_ARRAY(&state, unsigned long long, nbobjs*nbobjs, dist->values, "u64values", "%llu", 10);
  state.end_object(&state, dist->different_types ? "distances2hetero" : "distances2");

根据dist->different_types的值，调用不同的导出函数EXPORT_TYPE_GPINDEX_ARRAY或EXPORT_ARRAY，导出数据到XML文件中。


}

static void
hwloc__xml_v2export_distances(hwloc__xml_export_state_t parentstate, hwloc_topology_t topology)

定义了一个静态函数hwloc__xml_v2export_distances，接受两个参数：一个是hwloc__xml_export_state_t类型的parentstate，另一个是hwloc_topology_t类型的topology。


{
  struct hwloc_internal_distances_s *dist;
  for(dist = topology->first_dist; dist; dist = dist->next)
    if (!dist->different_types)
      hwloc___xml_v2export_distances(parentstate, dist);
  /* export homogeneous distances first in case the importer doesn't support heterogeneous and stops there */
  for(dist = topology->first_dist; dist; dist = dist->next)
    if (dist->different_types)
      hwloc___xml_v2export_distances(parentstate, dist);
}

遍历topology中的距离结构体，根据不同类型的距离调用hwloc___xml_v2export_distances函数导出数据到XML文件中。
# 定义函数 hwloc__xml_v2export_support，接受两个参数：父状态和拓扑结构
hwloc__xml_v2export_support(hwloc__xml_export_state_t parentstate, hwloc_topology_t topology)
{
  # 定义结构体 state 用于保存导出状态
  struct hwloc__xml_export_state_s state;
  # 定义一个长度为11的字符数组 tmp
  char tmp[11];

  # 如果处于调试模式下
#ifdef HWLOC_DEBUG
  # 断言结构体的大小
  HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_support) == 4*sizeof(void*));
  HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_discovery_support) == 6);
  HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_cpubind_support) == 11);
  HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_membind_support) == 15);
  HWLOC_BUILD_ASSERT(sizeof(struct hwloc_topology_misc_support) == 1);
#endif

  # 定义宏，用于生成支持属性的导出
#define DO(_cat,_name) do {                                     \
    # 如果拓扑结构的支持属性为真
    if (topology->support._cat->_name) {                        \
      # 创建新的子状态
      parentstate->new_child(parentstate, &state, "support");   \
      # 添加属性名和值
      state.new_prop(&state, "name", #_cat "." #_name);         \
      # 如果支持属性的值不为1
      if (topology->support._cat->_name != 1) {                 \
        # 将值转换为字符串并保存在 tmp 中
        sprintf(tmp, "%u", topology->support._cat->_name); \
        # 添加属性值
        state.new_prop(&state, "value", tmp);                   \
      }                                                         \
      # 结束当前对象的导出
      state.end_object(&state, "support");                      \
  }                                                           \   # 结束 do-while 循环
  } while (0)                                                  # 使用 do-while 循环来确保宏的完整性

  DO(discovery,pu);                                            # 调用名为 discovery 的宏，传入参数 pu
  DO(discovery,numa);                                          # 调用名为 discovery 的宏，传入参数 numa
  DO(discovery,numa_memory);                                   # 调用名为 discovery 的宏，传入参数 numa_memory
  DO(discovery,disallowed_pu);                                 # 调用名为 discovery 的宏，传入参数 disallowed_pu
  DO(discovery,disallowed_numa);                               # 调用名为 discovery 的宏，传入参数 disallowed_numa
  DO(discovery,cpukind_efficiency);                            # 调用名为 discovery 的宏，传入参数 cpukind_efficiency
  DO(cpubind,set_thisproc_cpubind);                            # 调用名为 cpubind 的宏，传入参数 set_thisproc_cpubind
  DO(cpubind,get_thisproc_cpubind);                            # 调用名为 cpubind 的宏，传入参数 get_thisproc_cpubind
  DO(cpubind,set_proc_cpubind);                                # 调用名为 cpubind 的宏，传入参数 set_proc_cpubind
  DO(cpubind,get_proc_cpubind);                                # 调用名为 cpubind 的宏，传入参数 get_proc_cpubind
  DO(cpubind,set_thisthread_cpubind);                          # 调用名为 cpubind 的宏，传入参数 set_thisthread_cpubind
  DO(cpubind,get_thisthread_cpubind);                          # 调用名为 cpubind 的宏，传入参数 get_thisthread_cpubind
  DO(cpubind,set_thread_cpubind);                              # 调用名为 cpubind 的宏，传入参数 set_thread_cpubind
  DO(cpubind,get_thread_cpubind);                              # 调用名为 cpubind 的宏，传入参数 get_thread_cpubind
  DO(cpubind,get_thisproc_last_cpu_location);                  # 调用名为 cpubind 的宏，传入参数 get_thisproc_last_cpu_location
  DO(cpubind,get_proc_last_cpu_location);                      # 调用名为 cpubind 的宏，传入参数 get_proc_last_cpu_location
  DO(cpubind,get_thisthread_last_cpu_location);                # 调用名为 cpubind 的宏，传入参数 get_thisthread_last_cpu_location
  DO(membind,set_thisproc_membind);                            # 调用名为 membind 的宏，传入参数 set_thisproc_membind
  DO(membind,get_thisproc_membind);                            # 调用名为 membind 的宏，传入参数 get_thisproc_membind
  DO(membind,set_proc_membind);                                # 调用名为 membind 的宏，传入参数 set_proc_membind
  DO(membind,get_proc_membind);                                # 调用名为 membind 的宏，传入参数 get_proc_membind
  DO(membind,set_thisthread_membind);                          # 调用名为 membind 的宏，传入参数 set_thisthread_membind
  DO(membind,get_thisthread_membind);                          # 调用名为 membind 的宏，传入参数 get_thisthread_membind
  DO(membind,set_area_membind);                                # 调用名为 membind 的宏，传入参数 set_area_membind
  DO(membind,get_area_membind);                                # 调用名为 membind 的宏，传入参数 get_area_membind
  DO(membind,alloc_membind);                                   # 调用名为 membind 的宏，传入参数 alloc_membind
  DO(membind,firsttouch_membind);                              # 调用名为 membind 的宏，传入参数 firsttouch_membind
  DO(membind,bind_membind);                                    # 调用名为 membind 的宏，传入参数 bind_membind
  DO(membind,interleave_membind);                              # 调用名为 membind 的宏，传入参数 interleave_membind
  DO(membind,nexttouch_membind);                               # 调用名为 membind 的宏，传入参数 nexttouch_membind
  DO(membind,migrate_membind);                                 # 调用名为 membind 的宏，传入参数 migrate_membind
  DO(membind,get_area_memlocation);                            # 调用名为 membind 的宏，传入参数 get_area_memlocation

  /* misc.imported_support would be meaningless in the remote importer,
   * but the importer needs to know whether we exported support or not
   * (in case there are no support bit set at all),
   * use a custom/fake field to do so.
   */
  parentstate->new_child(parentstate, &state, "support");       # 创建名为 support 的新子对象
  state.new_prop(&state, "name", "custom.exported_support");    # 在状态对象中创建新属性，属性名为 name，值为 custom.exported_support
  state.end_object(&state, "support");                          # 结束名为 support 的对象
# 定义静态函数，用于导出内存属性目标
static void
hwloc__xml_export_memattr_target(hwloc__xml_export_state_t state,
                                 struct hwloc_internal_memattr_s *imattr,
                                 struct hwloc_internal_memattr_target_s *imtg)
{
  # 定义内部状态变量
  struct hwloc__xml_export_state_s vstate;
  # 定义临时字符串数组
  char tmp[255];

  # 如果内存属性标志需要初始化器
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    # 导出所有初始化器
    unsigned k;
    for(k=0; k<imtg->nr_initiators; k++) {
      # 获取当前初始化器
      struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[k];
      # 创建新的子节点
      state->new_child(state, &vstate, "memattr_value");
      # 设置目标对象类型属性
      vstate.new_prop(&vstate, "target_obj_type", hwloc_obj_type_string(imtg->type));
      # 将目标对象的全局索引转换为字符串
      snprintf(tmp, sizeof(tmp), "%llu", (unsigned long long) imtg->gp_index);
      vstate.new_prop(&vstate, "target_obj_gp_index", tmp);
      # 将初始化器的值转换为字符串
      snprintf(tmp, sizeof(tmp), "%llu", (unsigned long long) imi->value);
      vstate.new_prop(&vstate, "value", tmp);
      # 根据初始化器类型进行不同处理
      switch (imi->initiator.type) {
      case HWLOC_LOCATION_TYPE_OBJECT:
        # 如果初始化器类型为对象，则将对象的全局索引转换为字符串，并设置相关属性
        snprintf(tmp, sizeof(tmp), "%llu", (unsigned long long) imi->initiator.location.object.gp_index);
        vstate.new_prop(&vstate, "initiator_obj_gp_index", tmp);
        vstate.new_prop(&vstate, "initiator_obj_type", hwloc_obj_type_string(imi->initiator.location.object.type));
        break;
      case HWLOC_LOCATION_TYPE_CPUSET: {
        # 如果初始化器类型为 cpuset，则将 cpuset 转换为字符串，并设置相关属性
        char *setstring;
        hwloc_bitmap_asprintf(&setstring, imi->initiator.location.cpuset);
        if (setstring)
          vstate.new_prop(&vstate, "initiator_cpuset", setstring);
        free(setstring);
        break;
      }
      default:
        # 默认情况下断言失败
        assert(0);
      }
      # 结束当前对象的导出
      vstate.end_object(&vstate, "memattr_value");
    }
  } else {
    # 如果内存属性标志不需要初始化器，则只导出全局值
    state->new_child(state, &vstate, "memattr_value");
    vstate.new_prop(&vstate, "target_obj_type", hwloc_obj_type_string(imtg->type));
    # 将目标对象的全局索引转换为字符串
    snprintf(tmp, sizeof(tmp), "%llu", (unsigned long long) imtg->gp_index);
    # 为vstate对象添加新属性"target_obj_gp_index"，值为tmp
    vstate.new_prop(&vstate, "target_obj_gp_index", tmp);
    # 将imtg->noinitiator_value的值格式化为字符串，存入tmp中
    snprintf(tmp, sizeof(tmp), "%llu", (unsigned long long) imtg->noinitiator_value);
    # 为vstate对象添加新属性"value"，值为tmp
    vstate.new_prop(&vstate, "value", tmp);
    # 结束当前对象的构建，对象名为"memattr_value"
    vstate.end_object(&vstate, "memattr_value");
# 导出内存属性到 XML
static void
hwloc__xml_export_memattrs(hwloc__xml_export_state_t state, hwloc_topology_t topology)
{
  unsigned id;
  # 遍历内存属性
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr;
    struct hwloc__xml_export_state_s mstate;
    char tmp[255];
    unsigned j;

    # 如果是容量或者局部性内存属性，则不需要导出虚拟内存属性
    if (id == HWLOC_MEMATTR_ID_CAPACITY || id == HWLOC_MEMATTR_ID_LOCALITY)
      continue;

    # 获取内存属性
    imattr = &topology->memattrs[id];
    # 如果是标准属性且没有任何目标，则不需要导出
    if (id < HWLOC_MEMATTR_ID_MAX && !imattr->nr_targets)
      continue;

    # 创建新的内存属性子节点
    state->new_child(state, &mstate, "memattr");
    # 添加属性名和标志
    mstate.new_prop(&mstate, "name", imattr->name);
    snprintf(tmp, sizeof(tmp), "%lu", imattr->flags);
    mstate.new_prop(&mstate, "flags", tmp);

    # 导出内存属性的目标
    for(j=0; j<imattr->nr_targets; j++)
      hwloc__xml_export_memattr_target(&mstate, imattr, &imattr->targets[j]);

    # 结束内存属性子节点
    mstate.end_object(&mstate, "memattr");
  }
}

# 导出 CPU 类型到 XML
static void
hwloc__xml_export_cpukinds(hwloc__xml_export_state_t state, hwloc_topology_t topology)
{
  unsigned i;
  # 遍历 CPU 类型
  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    struct hwloc__xml_export_state_s cstate;
    char *setstring;
    unsigned j;

    # 创建新的 CPU 类型子节点
    state->new_child(state, &cstate, "cpukind");
    # 将 CPU 集合转换为字符串并添加为属性
    hwloc_bitmap_asprintf(&setstring, kind->cpuset);
    cstate.new_prop(&cstate, "cpuset", setstring);
    free(setstring);
    # 如果强制效率不是未知的，则添加为属性
    if (kind->forced_efficiency != HWLOC_CPUKIND_EFFICIENCY_UNKNOWN) {
      char tmp[11];
      snprintf(tmp, sizeof(tmp), "%d", kind->forced_efficiency);
      cstate.new_prop(&cstate, "forced_efficiency", tmp);
    }
    # 遍历kind->nr_infos次，j从0到kind->nr_infos-1
    for(j=0; j<kind->nr_infos; j++) {
      # 为name分配内存并将kind->infos[j].name的安全副本赋值给name
      char *name = hwloc__xml_export_safestrdup(kind->infos[j].name);
      # 为value分配内存并将kind->infos[j].value的安全副本赋值给value
      char *value = hwloc__xml_export_safestrdup(kind->infos[j].value);
      # 创建istate对象
      struct hwloc__xml_export_state_s istate;
      # 在cstate下创建新的子节点，节点名为"info"，并将其状态保存在istate中
      cstate.new_child(&cstate, &istate, "info");
      # 在istate下创建新的属性，属性名为"name"，属性值为name
      istate.new_prop(&istate, "name", name);
      # 在istate下创建新的属性，属性名为"value"，属性值为value
      istate.new_prop(&istate, "value", value);
      # 结束当前对象的创建，节点名为"info"
      istate.end_object(&istate, "info");
      # 释放name所占用的内存
      free(name);
      # 释放value所占用的内存
      free(value);
    }

    # 结束当前对象的创建，节点名为"cpukind"
    cstate.end_object(&cstate, "cpukind");
  }
}

void
hwloc__xml_export_topology(hwloc__xml_export_state_t state, hwloc_topology_t topology, unsigned long flags)
{
  char *env;
  // 获取拓扑结构的根节点
  hwloc_obj_t root = hwloc_get_root_obj(topology);

  if (flags & HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1) {
    hwloc_obj_t *numanodes, first_numanode;
    unsigned nr_numanodes;

    // 导出 NUMA 节点
    nr_numanodes = hwloc__xml_v1export_object_list_numanodes(root, &first_numanode, &numanodes);

    if (nr_numanodes) {
      /* we don't use hwloc__xml_v1export_object_with_memory() because we want/can keep root above the numa node */
      struct hwloc__xml_export_state_s rstate, mstate;
      hwloc_obj_t child;
      unsigned i;
      /* export the root */
      // 导出根节点
      state->new_child(state, &rstate, "object");
      hwloc__xml_export_object_contents (&rstate, topology, root, flags);
      /* export first memory child */
      // 导出第一个内存子节点
      rstate.new_child(&rstate, &mstate, "object");
      hwloc__xml_export_object_contents (&mstate, topology, first_numanode, flags);
      /* then its normal/io/misc children */
      // 然后导出其普通/IO/杂项子节点
      for_each_child(child, root)
        hwloc__xml_v1export_object (&mstate, topology, child, flags);
      for_each_io_child(child, root)
        hwloc__xml_v1export_object (&mstate, topology, child, flags);
      for_each_misc_child(child, root)
        hwloc__xml_v1export_object (&mstate, topology, child, flags);
      /* close first memory child */
      // 关闭第一个内存子节点
      mstate.end_object(&mstate, "object");
      /* now other memory children */
      // 现在导出其他内存子节点
      for(i=1; i<nr_numanodes; i++)
        hwloc__xml_v1export_object (&rstate, topology, numanodes[i], flags);
      /* close the root */
      // 关闭根节点
      rstate.end_object(&rstate, "object");
    } else {
      // 如果没有 NUMA 节点，则导出根节点
      hwloc__xml_v1export_object(state, topology, root, flags);
    }

    free(numanodes);

  } else {
    // 使用 XML V2 导出对象
    hwloc__xml_v2export_object (state, topology, root, flags);
    // 使用 XML V2 导出距离信息
    hwloc__xml_v2export_distances (state, topology);
    env = getenv("HWLOC_XML_EXPORT_SUPPORT");
    if (!env || atoi(env))
      // 如果环境变量中支持 XML 导出，则使用 XML V2 导出支持信息
      hwloc__xml_v2export_support(state, topology);
    # 调用函数将内存属性导出到 XML
    hwloc__xml_export_memattrs(state, topology);
    # 调用函数将 CPU 类型导出到 XML
    hwloc__xml_export_cpukinds(state, topology);
  }
}

void
hwloc__xml_export_diff(hwloc__xml_export_state_t parentstate, hwloc_topology_diff_t diff)
{
  while (diff) {
    // 创建一个新的 XML 导出状态对象
    struct hwloc__xml_export_state_s state;
    char tmp[255];

    // 在父状态下创建一个新的子节点
    parentstate->new_child(parentstate, &state, "diff");

    // 将整数类型的 diff->generic.type 转换为字符串并添加为属性
    sprintf(tmp, "%d", (int) diff->generic.type);
    state.new_prop(&state, "type", tmp);

    switch (diff->generic.type) {
    case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR:
      // 将整数类型的 diff->obj_attr.obj_depth 转换为字符串并添加为属性
      sprintf(tmp, "%d", diff->obj_attr.obj_depth);
      state.new_prop(&state, "obj_depth", tmp);
      // 将无符号整数类型的 diff->obj_attr.obj_index 转换为字符串并添加为属性
      sprintf(tmp, "%u", diff->obj_attr.obj_index);
      state.new_prop(&state, "obj_index", tmp);

      // 将整数类型的 diff->obj_attr.diff.generic.type 转换为字符串并添加为属性
      sprintf(tmp, "%d", (int) diff->obj_attr.diff.generic.type);
      state.new_prop(&state, "obj_attr_type", tmp);

      switch (diff->obj_attr.diff.generic.type) {
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE:
        // 将无符号长整数类型的 diff->obj_attr.diff.uint64.index 转换为字符串并添加为属性
        sprintf(tmp, "%llu", (unsigned long long) diff->obj_attr.diff.uint64.index);
        state.new_prop(&state, "obj_attr_index", tmp);
        // 将无符号长整数类型的 diff->obj_attr.diff.uint64.oldvalue 转换为字符串并添加为属性
        sprintf(tmp, "%llu", (unsigned long long) diff->obj_attr.diff.uint64.oldvalue);
        state.new_prop(&state, "obj_attr_oldvalue", tmp);
        // 将无符号长整数类型的 diff->obj_attr.diff.uint64.newvalue 转换为字符串并添加为属性
        sprintf(tmp, "%llu", (unsigned long long) diff->obj_attr.diff.uint64.newvalue);
        state.new_prop(&state, "obj_attr_newvalue", tmp);
        break;
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME:
      case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO:
        // 如果 diff->obj_attr.diff.string.name 不为空，则添加为属性
        if (diff->obj_attr.diff.string.name)
          state.new_prop(&state, "obj_attr_name", diff->obj_attr.diff.string.name);
        // 添加旧值为属性
        state.new_prop(&state, "obj_attr_oldvalue", diff->obj_attr.diff.string.oldvalue);
        // 添加新值为属性
        state.new_prop(&state, "obj_attr_newvalue", diff->obj_attr.diff.string.newvalue);
        break;
      }

      break;
    default:
      // 断言，如果不是预期的类型则终止程序
      assert(0);
    }
    // 结束当前对象的导出
    state.end_object(&state, "diff");

    // 移动到下一个 diff
    diff = diff->generic.next;
  }
}

/**********************************
 ********* main XML export ********
 **********************************/

/* this can be the first XML call */
# 将硬件拓扑结构导出为 XML 文件
int hwloc_topology_export_xml(hwloc_topology_t topology, const char *filename, unsigned long flags)
{
  hwloc_localeswitch_declare;  // 声明本地化开关
  struct hwloc__xml_export_data_s edata;  // 定义 XML 导出数据结构
  int force_nolibxml;  // 强制使用无 libxml 标志
  int ret;  // 返回值

  if (!topology->is_loaded) {  // 如果拓扑结构未加载
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  assert(hwloc_nolibxml_callbacks); /* the core called components_init() for the topology */  // 断言核心调用了 components_init() 来初始化拓扑结构

  if (flags & ~HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1) {  // 如果标志位包含不支持的标志
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  hwloc_internal_distances_refresh(topology);  // 刷新内部距离信息

  hwloc_localeswitch_init();  // 初始化本地化开关

  edata.v1_memory_group = NULL;  // 初始化 v1 内存组为空
  if (flags & HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1)  // 如果标志位包含 v1 导出标志
    /* temporary group to be used during v1 export of memory children */
    edata.v1_memory_group = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);  // 为 v1 导出内存子对象创建临时组

  force_nolibxml = hwloc_nolibxml_export();  // 获取无 libxml 导出标志
retry:
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))  // 如果没有 libxml 回调或者强制使用无 libxml
    ret = hwloc_nolibxml_callbacks->export_file(topology, &edata, filename, flags);  // 使用无 libxml 回调导出文件
  else {
    ret = hwloc_libxml_callbacks->export_file(topology, &edata, filename, flags);  // 使用 libxml 回调导出文件
    if (ret < 0 && errno == ENOSYS) {  // 如果返回值小于 0 并且错误码为不支持的系统调用
      hwloc_libxml_callbacks = NULL;  // 清空 libxml 回调
      goto retry;  // 重试
    }
  }

  if (edata.v1_memory_group)  // 如果 v1 内存组存在
    hwloc_free_unlinked_object(edata.v1_memory_group);  // 释放未链接的对象

  hwloc_localeswitch_fini();  // 结束本地化开关
  return ret;  // 返回结果
}

/* this can be the first XML call */
int hwloc_topology_export_xmlbuffer(hwloc_topology_t topology, char **xmlbuffer, int *buflen, unsigned long flags)
{
  hwloc_localeswitch_declare;  // 声明本地化开关
  struct hwloc__xml_export_data_s edata;  // 定义 XML 导出数据结构
  int force_nolibxml;  // 强制使用无 libxml 标志
  int ret;  // 返回值

  if (!topology->is_loaded) {  // 如果拓扑结构未加载
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回错误
  }

  assert(hwloc_nolibxml_callbacks); /* the core called components_init() for the topology */  // 断言核心调用了 components_init() 来初始化拓扑结构

  if (flags & ~HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1) {  // 如果标志位包含不支持的标志
    errno = EINVAL;  // 设置错误码为无效参数
    # 返回错误代码 -1
    return -1;
  }

  # 刷新内部距离信息
  hwloc_internal_distances_refresh(topology);

  # 初始化本地化开关
  hwloc_localeswitch_init();

  # 初始化 v1_memory_group 为 NULL
  edata.v1_memory_group = NULL;
  # 如果标志包含 HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1
  if (flags & HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1)
    # 创建一个临时组对象，用于在导出内存子对象时使用
    edata.v1_memory_group = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);

  # 检查是否需要强制禁用 libxml 导出
  force_nolibxml = hwloc_nolibxml_export();
retry:
  # 如果没有设置 hwloc_libxml_callbacks 或者 (hwloc_nolibxml_callbacks 存在并且 force_nolibxml 为真)，则使用 hwloc_nolibxml_callbacks 导出缓冲区
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    ret = hwloc_nolibxml_callbacks->export_buffer(topology, &edata, xmlbuffer, buflen, flags);
  else:
    # 否则，使用 hwloc_libxml_callbacks 导出缓冲区
    ret = hwloc_libxml_callbacks->export_buffer(topology, &edata, xmlbuffer, buflen, flags);
    # 如果返回值小于 0 并且 errno 为 ENOSYS，则将 hwloc_libxml_callbacks 置为 NULL，重新尝试
    if (ret < 0 && errno == ENOSYS):
      hwloc_libxml_callbacks = NULL;
      goto retry;

  # 如果 edata.v1_memory_group 存在，则释放未链接的对象
  if (edata.v1_memory_group)
    hwloc_free_unlinked_object(edata.v1_memory_group);

  # 结束本地化环境
  hwloc_localeswitch_fini();
  # 返回结果
  return ret;
}

# 导出拓扑差异到 XML 文件
int
hwloc_topology_diff_export_xml(hwloc_topology_diff_t diff, const char *refname,
                   const char *filename)
{
  hwloc_localeswitch_declare;
  hwloc_topology_diff_t tmpdiff;
  int force_nolibxml;
  int ret;

  # 初始化 tmpdiff 为 diff
  tmpdiff = diff;
  # 遍历 tmpdiff，如果遇到 HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX 类型的差异，则设置 errno 并返回 -1
  while (tmpdiff):
    if (tmpdiff->generic.type == HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX):
      errno = EINVAL;
      return -1;
    tmpdiff = tmpdiff->generic.next;

  # 初始化组件
  hwloc_components_init();
  # 断言 hwloc_nolibxml_callbacks 存在
  assert(hwloc_nolibxml_callbacks);

  # 初始化本地化环境
  hwloc_localeswitch_init();

  # 获取是否强制使用 nolibxml
  force_nolibxml = hwloc_nolibxml_export();
retry:
  # 如果没有设置 hwloc_libxml_callbacks 或者 (hwloc_nolibxml_callbacks 存在并且 force_nolibxml 为真)，则使用 hwloc_nolibxml_callbacks 导出差异文件
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    ret = hwloc_nolibxml_callbacks->export_diff_file(diff, refname, filename);
  else:
    # 否则，使用 hwloc_libxml_callbacks 导出差异文件
    ret = hwloc_libxml_callbacks->export_diff_file(diff, refname, filename);
    # 如果返回值小于 0 并且 errno 为 ENOSYS，则将 hwloc_libxml_callbacks 置为 NULL，重新尝试
    if (ret < 0 && errno == ENOSYS):
      hwloc_libxml_callbacks = NULL;
      goto retry;

  # 结束本地化环境
  hwloc_localeswitch_fini();
  # 结束组件
  hwloc_components_fini();
  # 返回结果
  return ret;
}

# 导出拓扑差异到 XML 缓冲区
int
hwloc_topology_diff_export_xmlbuffer(hwloc_topology_diff_t diff, const char *refname,
                     char **xmlbuffer, int *buflen)
{
  hwloc_localeswitch_declare;
  hwloc_topology_diff_t tmpdiff;
  int force_nolibxml;
  int ret;

  # 初始化 tmpdiff 为 diff
  tmpdiff = diff;
  # 遍历 tmpdiff，如果遇到 HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX 类型的差异，则设置 errno 并返回 -1
  while (tmpdiff):
    if (tmpdiff->generic.type == HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX):
      errno = EINVAL;
      return -1;
    # 将 tmpdiff 指针指向下一个节点
    tmpdiff = tmpdiff->generic.next;
  }

  # 初始化硬件位置组件
  hwloc_components_init();
  # 断言确保 hwloc_nolibxml_callbacks 存在
  assert(hwloc_nolibxml_callbacks);

  # 初始化本地化环境
  hwloc_localeswitch_init();

  # 强制使用无 libxml 导出
  force_nolibxml = hwloc_nolibxml_export();
# 重试标签，用于循环重试
retry:
  # 如果没有硬件定位库的 XML 回调函数，或者强制使用无 XML 的回调函数
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    # 使用无 XML 的回调函数导出差异数据到缓冲区
    ret = hwloc_nolibxml_callbacks->export_diff_buffer(diff, refname, xmlbuffer, buflen);
  else:
    # 否则，使用硬件定位库的 XML 回调函数导出差异数据到缓冲区
    ret = hwloc_libxml_callbacks->export_diff_buffer(diff, refname, xmlbuffer, buflen);
    # 如果返回值小于 0 并且错误码为 ENOSYS
    if (ret < 0 && errno == ENOSYS):
      # 将硬件定位库的 XML 回调函数置空
      hwloc_libxml_callbacks = NULL;
      # 跳转到重试标签，进行重试
      goto retry;

# 结束本地化切换
  hwloc_localeswitch_fini();
# 结束组件
  hwloc_components_fini();
# 返回结果
  return ret;
}

# 释放 XML 缓冲区
void hwloc_free_xmlbuffer(hwloc_topology_t topology __hwloc_attribute_unused, char *xmlbuffer)
{
  int force_nolibxml;

  # 断言硬件定位库的无 XML 回调函数存在，核心调用了 components_init() 来初始化拓扑
  assert(hwloc_nolibxml_callbacks);
  
  # 获取是否强制使用无 XML 的标志
  force_nolibxml = hwloc_nolibxml_export();
  # 如果没有硬件定位库的 XML 回调函数，或者强制使用无 XML 的回调函数
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    # 使用无 XML 的回调函数释放缓冲区
    hwloc_nolibxml_callbacks->free_buffer(xmlbuffer);
  else
    # 否则，使用硬件定位库的 XML 回调函数释放缓冲区
    hwloc_libxml_callbacks->free_buffer(xmlbuffer);
}

# 设置拓扑数据导出回调函数
void
hwloc_topology_set_userdata_export_callback(hwloc_topology_t topology,
                        void (*export)(void *reserved, struct hwloc_topology *topology, struct hwloc_obj *obj))
{
  # 设置拓扑数据导出回调函数
  topology->userdata_export_cb = export;
}

# 导出对象的用户数据
static void
hwloc__export_obj_userdata(hwloc__xml_export_state_t parentstate, int encoded,
               const char *name, size_t length, const void *buffer, size_t encoded_length)
{
  struct hwloc__xml_export_state_s state;
  char tmp[255];
  # 创建新的子节点
  parentstate->new_child(parentstate, &state, "userdata");
  # 如果名称存在，设置属性名为名称
  if (name)
    state.new_prop(&state, "name", name);
  # 将长度转换为字符串
  sprintf(tmp, "%lu", (unsigned long) length);
  # 设置长度属性
  state.new_prop(&state, "length", tmp);
  # 如果编码标志为真，设置编码属性为 base64
  if (encoded)
    state.new_prop(&state, "encoding", "base64");
  # 如果编码长度存在，添加内容
  if (encoded_length)
    state.add_content(&state, buffer, encoded ? encoded_length : length);
  # 结束对象
  state.end_object(&state, "userdata");
}

# 导出对象的用户数据
int
hwloc_export_obj_userdata(void *reserved,
              struct hwloc_topology *topology, struct hwloc_obj *obj __hwloc_attribute_unused,
              const char *name, const void *buffer, size_t length)
{
  // 初始化 XML 导出状态为保留状态
  hwloc__xml_export_state_t state = reserved;

  // 如果 buffer 为空，则设置 errno 为无效参数并返回 -1
  if (!buffer) {
    errno = EINVAL;
    return -1;
  }

  // 如果 name 不为空且 name 的长度不符合要求，或者 buffer 的长度不符合要求，则设置 errno 为无效参数并返回 -1
  if ((name && hwloc__xml_export_check_buffer(name, strlen(name)) < 0)
      || hwloc__xml_export_check_buffer(buffer, length) < 0) {
    errno = EINVAL;
    return -1;
  }

  // 如果拓扑结构的 userdata_not_decoded 不为空
  if (topology->userdata_not_decoded) {
    int encoded;
    size_t encoded_length;
    const char *realname;
    assert(name);
    // 如果 name 以 "base64" 开头
    if (!strncmp(name, "base64", 6)) {
      encoded = 1;
      encoded_length = BASE64_ENCODED_LENGTH(length);
    } else {
      assert(!strncmp(name, "normal", 6));
      encoded = 0;
      encoded_length = length;
    }
    // 如果 name 的第 7 个字符是 ':'
    if (name[6] == ':')
      realname = name+7;
    else {
      assert(!strcmp(name+6, "-anon"));
      realname = NULL;
    }
    // 导出对象的 userdata
    hwloc__export_obj_userdata(state, encoded, realname, length, buffer, encoded_length);

  } else
    // 导出对象的 userdata
    hwloc__export_obj_userdata(state, 0, name, length, buffer, length);

  return 0;
}

// 导出对象的 userdata，使用 base64 编码
int
hwloc_export_obj_userdata_base64(void *reserved,
                 struct hwloc_topology *topology __hwloc_attribute_unused, struct hwloc_obj *obj __hwloc_attribute_unused,
                 const char *name, const void *buffer, size_t length)
{
  // 初始化 XML 导出状态为保留状态
  hwloc__xml_export_state_t state = reserved;
  size_t encoded_length;
  char *encoded_buffer;
  int ret __hwloc_attribute_unused;

  // 如果 buffer 为空，则设置 errno 为无效参数并返回 -1
  if (!buffer) {
    errno = EINVAL;
    return -1;
  }

  // 断言拓扑结构的 userdata_not_decoded 为空
  assert(!topology->userdata_not_decoded);

  // 如果 name 不为空且 name 的长度不符合要求，则设置 errno 为无效参数并返回 -1
  if (name && hwloc__xml_export_check_buffer(name, strlen(name)) < 0) {
    errno = EINVAL;
    return -1;
  }

  // 计算 base64 编码后的长度
  encoded_length = BASE64_ENCODED_LENGTH(length);
  // 分配内存用于存储 base64 编码后的数据
  encoded_buffer = malloc(encoded_length+1);
  if (!encoded_buffer) {
    errno = ENOMEM;
    return -1;
  }

  // 对 buffer 进行 base64 编码
  ret = hwloc_encode_to_base64(buffer, length, encoded_buffer, encoded_length+1);
  assert(ret == (int) encoded_length);

  // 导出对象的 userdata，使用 base64 编码
  hwloc__export_obj_userdata(state, 1, name, length, encoded_buffer, encoded_length);

  // 释放内存
  free(encoded_buffer);
  return 0;
}

void
# 设置用户数据导入回调函数
hwloc_topology_set_userdata_import_callback(hwloc_topology_t topology,
                        void (*import)(struct hwloc_topology *topology, struct hwloc_obj *obj, const char *name, const void *buffer, size_t length))
{
  # 将传入的回调函数赋值给拓扑结构体中的用户数据导入回调函数指针
  topology->userdata_import_cb = import;
}

/***************************************
 ************ XML component ************
 ***************************************/

# 禁用 XML 组件
static void
hwloc_xml_backend_disable(struct hwloc_backend *backend)
{
  # 获取后端私有数据
  struct hwloc_xml_backend_data_s *data = backend->private_data;
  # 调用后端退出函数
  data->backend_exit(data);
  # 释放消息前缀内存
  free(data->msgprefix);
  # 释放后端私有数据内存
  free(data);
}

# 实例化 XML 组件
static struct hwloc_backend *
hwloc_xml_component_instantiate(struct hwloc_topology *topology,
                struct hwloc_disc_component *component,
                unsigned excluded_phases __hwloc_attribute_unused,
                const void *_data1,
                const void *_data2,
                const void *_data3)
{
  struct hwloc_xml_backend_data_s *data;
  struct hwloc_backend *backend;
  const char *env;
  int force_nolibxml;
  const char * xmlpath = (const char *) _data1;
  const char * xmlbuffer = (const char *) _data2;
  int xmlbuflen = (int)(uintptr_t) _data3;
  const char *local_basename;
  int err;

  # 断言核心调用了组件初始化函数
  assert(hwloc_nolibxml_callbacks); /* the core called components_init() for the component's topology */

  # 如果没有提供 XML 文件路径和缓冲区，则从环境变量中获取
  if (!xmlpath && !xmlbuffer) {
    env = getenv("HWLOC_XMLFILE");
    if (env) {
      # 如果环境变量中存在 XML 文件路径，则使用该路径
      xmlpath = env;
    } else {
      # 否则设置错误码并跳转到结束标签
      errno = EINVAL;
      goto out;
    }
  }

  # 分配后端内存
  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    goto out;

  # 分配后端私有数据内存
  data = malloc(sizeof(*data));
  if (!data) {
    errno = ENOMEM;
    goto out_with_backend;
  }

  # 将后端私有数据赋值给后端
  backend->private_data = data;
  # 设置后端发现函数为 XML 查找函数
  backend->discover = hwloc_look_xml;
  # 设置后端禁用函数为 XML 后端禁用函数
  backend->disable = hwloc_xml_backend_disable;
  # 设置是否为本系统标志为 0
  backend->is_thissystem = 0;

  # 如果提供了 XML 文件路径
  if (xmlpath) {
    # 获取文件名
    local_basename = strrchr(xmlpath, '/');
    # 如果本地基本名称存在，则将其指针向后移动一位
    if (local_basename)
      local_basename++;
    # 如果本地基本名称不存在，则将其设置为xmlpath
    else
      local_basename = xmlpath;
  # 否则，将本地基本名称设置为"xmlbuffer"
  } else {
    local_basename = "xmlbuffer";
  }
  # 将本地基本名称复制到数据结构的消息前缀中
  data->msgprefix = strdup(local_basename);

  # 强制使用nolibxml进行导入
  force_nolibxml = hwloc_nolibxml_import();
# 重试标签，用于循环重试
retry:
  # 如果没有设置 hwloc_libxml_callbacks 或者设置了 hwloc_nolibxml_callbacks 并且强制使用 nolibxml，则执行以下代码
  if (!hwloc_libxml_callbacks || (hwloc_nolibxml_callbacks && force_nolibxml))
    # 调用 hwloc_nolibxml_callbacks 的 backend_init 方法
    err = hwloc_nolibxml_callbacks->backend_init(data, xmlpath, xmlbuffer, xmlbuflen);
  else:
    # 调用 hwloc_libxml_callbacks 的 backend_init 方法
    err = hwloc_libxml_callbacks->backend_init(data, xmlpath, xmlbuffer, xmlbuflen);
    # 如果出现错误并且错误码为 ENOSYS，则执行以下代码
    if (err < 0 && errno == ENOSYS):
      # 将 hwloc_libxml_callbacks 置为 NULL
      hwloc_libxml_callbacks = NULL;
      # 跳转到 retry 标签处，进行重试
      goto retry;
  # 如果出现错误，则跳转到 out_with_data 标签处
  if (err < 0)
    goto out_with_data;

  # 返回 backend 对象
  return backend;

 # 释放 data 对象的 msgprefix 字段内存
 out_with_data:
  free(data->msgprefix);
  # 释放 data 对象内存
  free(data);
 # 释放 backend 对象内存
 out_with_backend:
  free(backend);
 # 返回 NULL
 out:
  return NULL;
}

# 定义 hwloc_xml_disc_component 结构体
static struct hwloc_disc_component hwloc_xml_disc_component = {
  "xml",  # 组件名称
  HWLOC_DISC_PHASE_GLOBAL,  # 组件的全局阶段
  ~0,  # 组件的深度
  hwloc_xml_component_instantiate,  # 实例化组件的方法
  30,  # 组件的权重
  1,  # 组件的可见性
  NULL  # 额外数据
};

# 定义 hwloc_xml_component 结构体
const struct hwloc_component hwloc_xml_component = {
  HWLOC_COMPONENT_ABI,  # 组件的 ABI 版本
  NULL, NULL,  # 初始化和销毁方法
  HWLOC_COMPONENT_TYPE_DISC,  # 组件类型为 DISC
  0,  # 保留字段
  &hwloc_xml_disc_component  # 指向 hwloc_xml_disc_component 结构体的指针
};
```