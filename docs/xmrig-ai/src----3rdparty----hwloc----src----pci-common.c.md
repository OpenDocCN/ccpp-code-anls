# `xmrig\src\3rdparty\hwloc\src\pci-common.c`

```cpp
/*
 * 版权所有 © 2009-2022 Inria。
 * 保留所有权利。请参阅顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "hwloc/plugins.h"
#include "private/private.h"
#include "private/debug.h"
#include "private/misc.h"

#include <fcntl.h>
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <sys/stat.h>

#if defined(HWLOC_WIN_SYS) && !defined(__CYGWIN__)
#include <io.h>
#define open _open
#define read _read
#define close _close
#endif


/**************************************
 * Init/Exit and Forced PCI localities
 */

static void
hwloc_pci_forced_locality_parse_one(struct hwloc_topology *topology,
                    const char *string /* must contain a ' ' */,
                    unsigned *allocated)
{
  unsigned nr = topology->pci_forced_locality_nr; // 初始化 nr 变量为拓扑结构体中的 pci_forced_locality_nr 成员
  unsigned domain, bus_first, bus_last, dummy; // 声明无符号整数变量
  hwloc_bitmap_t set; // 声明 hwloc_bitmap_t 类型的变量
  char *tmp; // 声明字符指针变量

  // 从输入字符串中解析出域、起始总线、结束总线和虚拟设备号
  if (sscanf(string, "%x:%x-%x %x", &domain, &bus_first, &bus_last, &dummy) == 4) {
    /* fine */
  } else if (sscanf(string, "%x:%x %x", &domain, &bus_first, &dummy) == 3) {
    bus_last = bus_first;
  } else if (sscanf(string, "%x %x", &domain, &dummy) == 2) {
    bus_first = 0;
    bus_last = 255;
  } else
    return;

  // 在输入字符串中查找空格，并将 tmp 指向空格后的字符串
  tmp = strchr(string, ' ');
  if (!tmp)
    return;
  tmp++;

  // 分配一个位图并从字符串中读取位图
  set = hwloc_bitmap_alloc();
  hwloc_bitmap_sscanf(set, tmp);

  // 如果 allocated 为 0，则分配一个新的 pci_forced_locality 结构体
  if (!*allocated) {
    topology->pci_forced_locality = malloc(sizeof(*topology->pci_forced_locality));
    if (!topology->pci_forced_locality)
      goto out_with_set; /* failed to allocate, ignore this forced locality */
    *allocated = 1;
  } else if (nr >= *allocated) {
    // 如果 nr 大于或等于 allocated，则重新分配内存
    struct hwloc_pci_forced_locality_s *tmplocs;
    tmplocs = realloc(topology->pci_forced_locality,
              2 * *allocated * sizeof(*topology->pci_forced_locality));
    if (!tmplocs)
      goto out_with_set; /* failed to allocate, ignore this forced locality */
    topology->pci_forced_locality = tmplocs;
    # 将指针 *allocated 指向的值乘以2
    *allocated *= 2;
  }

  # 设置强制本地性的PCI域、总线范围和CPU集合
  topology->pci_forced_locality[nr].domain = domain;
  topology->pci_forced_locality[nr].bus_first = bus_first;
  topology->pci_forced_locality[nr].bus_last = bus_last;
  topology->pci_forced_locality[nr].cpuset = set;
  # 增加强制本地性的PCI域数量
  topology->pci_forced_locality_nr++;
  # 返回
  return;

 out_with_set:
  # 释放CPU集合的内存
  hwloc_bitmap_free(set);
  # 返回
  return;
}

static void
hwloc_pci_forced_locality_parse(struct hwloc_topology *topology, const char *_env)
{
  // 复制环境变量字符串
  char *env = strdup(_env);
  unsigned allocated = 0;
  char *tmp = env;

  // 循环解析环境变量字符串
  while (1) {
    // 获取下一个分号或换行符之前的子字符串长度
    size_t len = strcspn(tmp, ";\r\n");
    char *next = NULL;

    // 如果子字符串不是最后一个字符
    if (tmp[len] != '\0') {
      tmp[len] = '\0';
      // 如果子字符串后面还有字符，则指向下一个字符
      if (tmp[len+1] != '\0')
    next = &tmp[len]+1;
    }

    // 解析单个环境变量字符串
    hwloc_pci_forced_locality_parse_one(topology, tmp, &allocated);

    // 如果有下一个子字符串，则继续解析
    if (next)
      tmp = next;
    else
      break;
  }

  // 释放复制的环境变量字符串
  free(env);
}

void
hwloc_pci_discovery_init(struct hwloc_topology *topology)
{
  // 初始化 PCI 强制本地性相关的变量
  topology->pci_has_forced_locality = 0;
  topology->pci_forced_locality_nr = 0;
  topology->pci_forced_locality = NULL;

  topology->first_pci_locality = topology->last_pci_locality = NULL;

  // 定义 PCI 本地性的特殊情况
#define HWLOC_PCI_LOCALITY_QUIRK_CRAY_EX235A (1ULL<<0)
#define HWLOC_PCI_LOCALITY_QUIRK_FAKE (1ULL<<62)
  // 初始化 PCI 本地性的特殊情况
  topology->pci_locality_quirks = (uint64_t) -1;
  /* -1 is unknown, 0 is disabled, >0 is bitmask of enabled quirks.
   * bit 63 should remain unused so that -1 is unaccessible as a bitmask.
   */
}

void
hwloc_pci_discovery_prepare(struct hwloc_topology *topology)
{
  char *env;

  // 获取环境变量 HWLOC_PCI_LOCALITY 的值
  env = getenv("HWLOC_PCI_LOCALITY");
  if (env) {
    int fd;

    // 设置 PCI 强制本地性标志
    topology->pci_has_forced_locality = 1;

    // 打开环境变量指定的文件
    fd = open(env, O_RDONLY);
    if (fd >= 0) {
      struct stat st;
      char *buffer;
      int err = fstat(fd, &st);
      // 获取文件状态成功
      if (!err) {
    // 如果文件大小不超过64KB
    if (st.st_size <= 64*1024) { /* random limit large enough to store multiple cpusets for thousands of PUs */
      // 分配缓冲区
      buffer = malloc(st.st_size+1);
      // 读取文件内容到缓冲区
      if (buffer && read(fd, buffer, st.st_size) == st.st_size) {
        buffer[st.st_size] = '\0';
        // 解析文件内容
        hwloc_pci_forced_locality_parse(topology, buffer);
      }
      // 释放缓冲区
      free(buffer);
    } else {
          // 如果文件过大，则忽略并输出错误信息
          if (HWLOC_SHOW_CRITICAL_ERRORS())
            fprintf(stderr, "hwloc/pci: Ignoring HWLOC_PCI_LOCALITY file `%s' too large (%lu bytes)\n",
                    env, (unsigned long) st.st_size);
    }
      }
      // 关闭文件
      close(fd);
    } else
      hwloc_pci_forced_locality_parse(topology, env);
  }



# 如果条件不满足，则执行以下代码
hwloc_pci_forced_locality_parse(topology, env);
}
// 退出 PCI 设备发现模块
void
hwloc_pci_discovery_exit(struct hwloc_topology *topology)
{
  struct hwloc_pci_locality_s *cur; // 定义指向 hwloc_pci_locality_s 结构体的指针 cur
  unsigned i; // 定义无符号整型变量 i

  // 释放强制本地性数组中每个元素的 cpuset
  for(i=0; i<topology->pci_forced_locality_nr; i++)
    hwloc_bitmap_free(topology->pci_forced_locality[i].cpuset);
  // 释放强制本地性数组
  free(topology->pci_forced_locality);

  // 释放 PCI 本地性链表中每个元素的 cpuset 和内存
  cur = topology->first_pci_locality;
  while (cur) {
    struct hwloc_pci_locality_s *next = cur->next;
    hwloc_bitmap_free(cur->cpuset);
    free(cur);
    cur = next;
  }

  // 重新初始化 PCI 设备发现模块
  hwloc_pci_discovery_init(topology);
}

/******************************
 * Inserting in Tree by Bus ID
 */

#ifdef HWLOC_DEBUG
static void
hwloc_pci_traverse_print_cb(void * cbdata __hwloc_attribute_unused,
                struct hwloc_obj *pcidev)
{
  char busid[14]; // 定义长度为 14 的字符数组 busid
  hwloc_obj_t parent; // 定义指向 hwloc_obj 结构体的指针 parent

  /* indent */
  parent = pcidev->parent; // 获取 pcidev 的父对象
  while (parent) { // 循环直到 parent 为空
    hwloc_debug("%s", "  "); // 打印两个空格
    parent = parent->parent; // 获取 parent 的父对象
  }

  // 格式化总线 ID 字符串
  snprintf(busid, sizeof(busid), "%04x:%02x:%02x.%01x",
           pcidev->attr->pcidev.domain, pcidev->attr->pcidev.bus, pcidev->attr->pcidev.dev, pcidev->attr->pcidev.func);

  if (pcidev->type == HWLOC_OBJ_BRIDGE) { // 如果 pcidev 的类型是桥
    if (pcidev->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_HOST) // 如果上游类型是主机
      hwloc_debug("HostBridge"); // 打印 HostBridge
    else
      hwloc_debug("%s Bridge [%04x:%04x]", busid,
          pcidev->attr->pcidev.vendor_id, pcidev->attr->pcidev.device_id); // 打印桥的总线 ID 和厂商 ID、设备 ID
    if (pcidev->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) // 如果下游类型是 PCI
      hwloc_debug(" to %04x:[%02x:%02x]\n",
                  pcidev->attr->bridge.downstream.pci.domain, pcidev->attr->bridge.downstream.pci.secondary_bus, pcidev->attr->bridge.downstream.pci.subordinate_bus); // 打印下游的总线 ID 和二级总线号、子总线号
    else
      assert(0); // 断言失败
  } else
    hwloc_debug("%s Device [%04x:%04x (%04x:%04x) rev=%02x class=%04x]\n", busid,
        pcidev->attr->pcidev.vendor_id, pcidev->attr->pcidev.device_id,
        pcidev->attr->pcidev.subvendor_id, pcidev->attr->pcidev.subdevice_id,
        pcidev->attr->pcidev.revision, pcidev->attr->pcidev.class_id); // 打印设备的总线 ID 和厂商 ID、设备 ID、子厂商 ID、子设备 ID、修订号、类别 ID
}

static void
# 遍历 PCI 设备树，对每个节点执行回调函数
hwloc_pci_traverse(void * cbdata, struct hwloc_obj *tree,
           void (*cb)(void * cbdata, struct hwloc_obj *))
{
  # 获取当前节点的子节点
  hwloc_obj_t child;
  # 执行回调函数
  cb(cbdata, tree);
  # 遍历当前节点的每个 IO 子节点
  for_each_io_child(child, tree) {
    # 如果子节点是桥接设备，则递归遍历其子节点
    if (child->type == HWLOC_OBJ_BRIDGE)
      hwloc_pci_traverse(cbdata, child, cb);
  }
}
#endif /* HWLOC_DEBUG */

# 定义 PCI 总线 ID 比较的枚举类型
enum hwloc_pci_busid_comparison_e {
  HWLOC_PCI_BUSID_LOWER,
  HWLOC_PCI_BUSID_HIGHER,
  HWLOC_PCI_BUSID_INCLUDED,
  HWLOC_PCI_BUSID_SUPERSET,
  HWLOC_PCI_BUSID_EQUAL
};

# 比较两个 PCI 设备的总线 ID
static enum hwloc_pci_busid_comparison_e
hwloc_pci_compare_busids(struct hwloc_obj *a, struct hwloc_obj *b)
{
#ifdef HWLOC_DEBUG
  # 如果节点 a 是桥接设备，则断言其上游类型为 PCI
  if (a->type == HWLOC_OBJ_BRIDGE)
    assert(a->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI);
  # 如果节点 b 是桥接设备，则断言其上游类型为 PCI
  if (b->type == HWLOC_OBJ_BRIDGE)
    assert(b->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI);
#endif

  # 比较两个节点的域 ID
  if (a->attr->pcidev.domain < b->attr->pcidev.domain)
    return HWLOC_PCI_BUSID_LOWER;
  if (a->attr->pcidev.domain > b->attr->pcidev.domain)
    return HWLOC_PCI_BUSID_HIGHER;

  # 比较桥接设备的下游总线范围
  if (a->type == HWLOC_OBJ_BRIDGE && a->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI
      && b->attr->pcidev.bus >= a->attr->bridge.downstream.pci.secondary_bus
      && b->attr->pcidev.bus <= a->attr->bridge.downstream.pci.subordinate_bus)
    return HWLOC_PCI_BUSID_SUPERSET;
  if (b->type == HWLOC_OBJ_BRIDGE && b->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI
      && a->attr->pcidev.bus >= b->attr->bridge.downstream.pci.secondary_bus
      && a->attr->pcidev.bus <= b->attr->bridge.downstream.pci.subordinate_bus)
    return HWLOC_PCI_BUSID_INCLUDED;

  # 比较节点的总线 ID
  if (a->attr->pcidev.bus < b->attr->pcidev.bus)
    return HWLOC_PCI_BUSID_LOWER;
  if (a->attr->pcidev.bus > b->attr->pcidev.bus)
    return HWLOC_PCI_BUSID_HIGHER;

  # 比较节点的设备号
  if (a->attr->pcidev.dev < b->attr->pcidev.dev)
    return HWLOC_PCI_BUSID_LOWER;
  if (a->attr->pcidev.dev > b->attr->pcidev.dev)
    return HWLOC_PCI_BUSID_HIGHER;

  # 比较节点的函数号
  if (a->attr->pcidev.func < b->attr->pcidev.func)
  # 如果 a 的 PCI 设备函数号小于 b 的 PCI 设备函数号，则返回较小的值
  return HWLOC_PCI_BUSID_LOWER;
  # 如果 a 的 PCI 设备函数号大于 b 的 PCI 设备函数号，则返回较大的值
  if (a->attr->pcidev.func > b->attr->pcidev.func)
    return HWLOC_PCI_BUSID_HIGHER;

  # 如果以上条件都不满足，则返回相等的值，理论上不应该执行到这里
  /* Should never reach here. */
  return HWLOC_PCI_BUSID_EQUAL;
}
static void
hwloc_pci_add_object(struct hwloc_obj *parent, struct hwloc_obj **parent_io_first_child_p, struct hwloc_obj *new)
{
  struct hwloc_obj **curp, **childp;

  curp = parent_io_first_child_p;
  while (*curp) {
    enum hwloc_pci_busid_comparison_e comp = hwloc_pci_compare_busids(new, *curp);
    switch (comp) {
    case HWLOC_PCI_BUSID_HIGHER:
      /* 如果新对象的总线 ID 比当前对象的总线 ID 更高，继续向下查找 */
      curp = &(*curp)->next_sibling;
      continue;
    case HWLOC_PCI_BUSID_INCLUDED:
      /* 如果新对象的总线 ID 包含在当前对象的总线 ID 中，将新对象插入到当前对象的下方 */
      hwloc_pci_add_object(*curp, &(*curp)->io_first_child, new);
      return;
    case HWLOC_PCI_BUSID_LOWER:
    case HWLOC_PCI_BUSID_SUPERSET: {
      /* 如果新对象的总线 ID 比当前对象的总线 ID 低或者是当前对象的总线 ID 的超集，将新对象插入到当前对象的前方 */
      new->next_sibling = *curp;
      *curp = new;
      new->parent = parent;
      if (new->type == HWLOC_OBJ_BRIDGE && new->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) {
    /* 查看剩余的兄弟节点，并将其中一些移动到新对象的下方 */
    childp = &new->io_first_child;
    curp = &new->next_sibling;
    while (*curp) {
      hwloc_obj_t cur = *curp;
      if (hwloc_pci_compare_busids(new, cur) == HWLOC_PCI_BUSID_LOWER) {
        /* 这个兄弟节点仍然在根节点下，放在新对象后面 */
        if (cur->attr->pcidev.domain > new->attr->pcidev.domain
        || cur->attr->pcidev.bus > new->attr->bridge.downstream.pci.subordinate_bus)
          /* 这个兄弟节点甚至在新对象的下级总线之上，没有其他兄弟节点可以放在新对象下方 */
          return;
        curp = &cur->next_sibling;
      } else {
        /* 这个兄弟节点放在新对象下方 */
        *childp = cur;
        *curp = cur->next_sibling;
        (*childp)->parent = new;
        (*childp)->next_sibling = NULL;
        childp = &(*childp)->next_sibling;
      }
    }
      }
      return;
    }
    # 如果 PCI 总线 ID 相等的情况
    case HWLOC_PCI_BUSID_EQUAL: {
      # 静态变量，用于标记是否已经报告过错误
      static int reported = 0;
      # 如果未报告过错误且允许显示关键错误
      if (!reported && HWLOC_SHOW_CRITICAL_ERRORS()) {
        # 输出错误信息
        fprintf(stderr, "*********************************************************\n");
        fprintf(stderr, "* hwloc %s received invalid PCI information.\n", HWLOC_VERSION);
        fprintf(stderr, "*\n");
        fprintf(stderr, "* Trying to insert PCI object %04x:%02x:%02x.%01x at %04x:%02x:%02x.%01x\n",
                new->attr->pcidev.domain, new->attr->pcidev.bus, new->attr->pcidev.dev, new->attr->pcidev.func,
                (*curp)->attr->pcidev.domain, (*curp)->attr->pcidev.bus, (*curp)->attr->pcidev.dev, (*curp)->attr->pcidev.func);
        fprintf(stderr, "*\n");
        fprintf(stderr, "* hwloc will now ignore this object and continue.\n");
        fprintf(stderr, "*********************************************************\n");
        # 标记已经报告过错误
        reported = 1;
      }
      # 释放未链接的对象
      hwloc_free_unlinked_object(new);
      # 返回
      return;
    }
    }
  }
  # 如果新对象的深度比所有对象都大，则添加到列表末尾
  new->parent = parent;
  new->next_sibling = NULL;
  *curp = new;
}
// 插入一个 PCI 设备对象到树中
void
hwloc_pcidisc_tree_insert_by_busid(struct hwloc_obj **treep,
                   struct hwloc_obj *obj)
{
  // 调用 hwloc_pci_add_object 函数，将 obj 插入到 treep 指向的树中
  hwloc_pci_add_object(NULL /* no parent on top of tree */, treep, obj);
}


/**********************
 * Attaching PCI Trees
 */

// 添加主机桥对象到树中
static struct hwloc_obj *
hwloc_pcidisc_add_hostbridges(struct hwloc_topology *topology,
                  struct hwloc_obj *old_tree)
{
  // 定义新的主机桥对象指针和指向该指针的指针
  struct hwloc_obj * new = NULL, **newp = &new;

  /*
   * tree points to all objects connected to any upstream bus in the machine.
   * We now create one real hostbridge object per upstream bus.
   * It's not actually a PCI device so we have to create it.
   */
  while (old_tree) {
    /* start a new host bridge */
    // 定义主机桥对象和指向其子对象的指针
    struct hwloc_obj *hostbridge;
    struct hwloc_obj **dstnextp;
    struct hwloc_obj **srcnextp;
    struct hwloc_obj *child;
    unsigned current_domain;
    unsigned char current_bus;
    unsigned char current_subordinate;

    // 为主机桥对象分配内存并初始化
    hostbridge = hwloc_alloc_setup_object(topology, HWLOC_OBJ_BRIDGE, HWLOC_UNKNOWN_INDEX);
    if (!hostbridge) {
      // 如果分配失败，将剩余的对象挂在树上并返回
      *newp = old_tree;
      return new;
    }
    dstnextp = &hostbridge->io_first_child;

    srcnextp = &old_tree;
    child = *srcnextp;
    current_domain = child->attr->pcidev.domain;
    current_bus = child->attr->pcidev.bus;
    current_subordinate = current_bus;

    hwloc_debug("Adding new PCI hostbridge %04x:%02x\n", current_domain, current_bus);

  next_child:
    /* remove next child from tree */
    *srcnextp = child->next_sibling;
    /* append it to hostbridge */
    *dstnextp = child;
    child->parent = hostbridge;
    child->next_sibling = NULL;
    dstnextp = &child->next_sibling;

    /* compute hostbridge secondary/subordinate buses */
    if (child->type == HWLOC_OBJ_BRIDGE && child->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI
    // 如果子节点的下游 PCI 总线大于当前下游总线，则更新当前下游总线
    && child->attr->bridge.downstream.pci.subordinate_bus > current_subordinate)
      current_subordinate = child->attr->bridge.downstream.pci.subordinate_bus;

    /* 如果下一个子节点具有相同的域/总线，则使用下一个子节点 */
    child = *srcnextp;
    if (child
    && child->attr->pcidev.domain == current_domain
    && child->attr->pcidev.bus == current_bus)
      goto next_child;

    /* 完成设置此主机桥 */
    hostbridge->attr->bridge.upstream_type = HWLOC_OBJ_BRIDGE_HOST;
    hostbridge->attr->bridge.downstream_type = HWLOC_OBJ_BRIDGE_PCI;
    hostbridge->attr->bridge.downstream.pci.domain = current_domain;
    hostbridge->attr->bridge.downstream.pci.secondary_bus = current_bus;
    hostbridge->attr->bridge.downstream.pci.subordinate_bus = current_subordinate;
    hwloc_debug("  new PCI hostbridge covers %04x:[%02x-%02x]\n",
        current_domain, current_bus, current_subordinate);

    *newp = hostbridge;
    newp = &hostbridge->next_sibling;
  }

  return new;
/* 如果应用了一个怪癖，则返回1 */
static int
hwloc__pci_find_busid_parent_quirk(struct hwloc_topology *topology,
                                   struct hwloc_pcidev_attr_s *busid,
                                   hwloc_cpuset_t cpuset)
{
  // 如果 PCI 本地性怪癖未知
  if (topology->pci_locality_quirks == (uint64_t)-1 /* unknown */) {
    const char *dmi_board_name, *env;

    /* 第一次调用，检测需要哪些怪癖 */
    topology->pci_locality_quirks = 0; /* 尚无怪癖 */

    // 获取 DMIBoardName 信息
    dmi_board_name = hwloc_obj_get_info_by_name(hwloc_get_root_obj(topology), "DMIBoardName");
    // 如果 DMIBoardName 存在且等于 "HPE CRAY EX235A"
    if (dmi_board_name && !strcmp(dmi_board_name, "HPE CRAY EX235A")) {
      hwloc_debug("enabling for PCI locality quirk for HPE Cray EX235A\n");
      topology->pci_locality_quirks |= HWLOC_PCI_LOCALITY_QUIRK_CRAY_EX235A;
    }

    // 获取环境变量 HWLOC_PCI_LOCALITY_QUIRK_FAKE
    env = getenv("HWLOC_PCI_LOCALITY_QUIRK_FAKE");
    // 如果环境变量存在且为非零值
    if (env && atoi(env)) {
      hwloc_debug("enabling for PCI locality fake quirk (attaching everything to last PU)\n");
      topology->pci_locality_quirks |= HWLOC_PCI_LOCALITY_QUIRK_FAKE;
    }
  }

  // 如果存在 HWLOC_PCI_LOCALITY_QUIRK_FAKE 怪癖
  if (topology->pci_locality_quirks & HWLOC_PCI_LOCALITY_QUIRK_FAKE) {
    // 获取最后一个处理器编号
    unsigned last = hwloc_bitmap_last(hwloc_topology_get_topology_cpuset(topology));
    // 将最后一个处理器编号添加到 cpuset 中
    hwloc_bitmap_set(cpuset, last);
    return 1;
  }

  // 如果存在 HWLOC_PCI_LOCALITY_QUIRK_CRAY_EX235A 怪癖
  if (topology->pci_locality_quirks & HWLOC_PCI_LOCALITY_QUIRK_CRAY_EX235A) {
    /* AMD Trento has xGMI ports connected to individual CCDs (8 cores + L3)
     * instead of NUMA nodes (pairs of CCDs within Trento) as is usual in AMD EPYC CPUs.
     * This is not described by the ACPI tables, hence we need to manually hardwire
     * the xGMI locality for the (currently single) server that currently uses that CPU.
     * It's not clear if ACPI tables can/will ever be fixed (would require one initiator
     * proximity domain per CCD), or if Linux can/will work around the issue.
     */
    # 如果总线 ID 的域为 0
    if (busid->domain == 0) {
      # 如果总线 ID 的总线号在 0xd0 和 0xd1 之间
      if (busid->bus >= 0xd0 && busid->bus <= 0xd1) {
        # 设置 CPU 核心集合的范围为 0 到 7
        hwloc_bitmap_set_range(cpuset, 0, 7);
        # 设置 CPU 核心集合的范围为 64 到 71
        hwloc_bitmap_set_range(cpuset, 64, 71);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xd4 和 0xd6 之间
      if (busid->bus >= 0xd4 && busid->bus <= 0xd6) {
        # 设置 CPU 核心集合的范围为 8 到 15
        hwloc_bitmap_set_range(cpuset, 8, 15);
        # 设置 CPU 核心集合的范围为 72 到 79
        hwloc_bitmap_set_range(cpuset, 72, 79);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xc8 和 0xc9 之间
      if (busid->bus >= 0xc8 && busid->bus <= 0xc9) {
        # 设置 CPU 核心集合的范围为 16 到 23
        hwloc_bitmap_set_range(cpuset, 16, 23);
        # 设置 CPU 核心集合的范围为 80 到 87
        hwloc_bitmap_set_range(cpuset, 80, 87);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xcc 和 0xce 之间
      if (busid->bus >= 0xcc && busid->bus <= 0xce) {
        # 设置 CPU 核心集合的范围为 24 到 31
        hwloc_bitmap_set_range(cpuset, 24, 31);
        # 设置 CPU 核心集合的范围为 88 到 95
        hwloc_bitmap_set_range(cpuset, 88, 95);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xd8 和 0xd9 之间
      if (busid->bus >= 0xd8 && busid->bus <= 0xd9) {
        # 设置 CPU 核心集合的范围为 32 到 39
        hwloc_bitmap_set_range(cpuset, 32, 39);
        # 设置 CPU 核心集合的范围为 96 到 103
        hwloc_bitmap_set_range(cpuset, 96, 103);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xdc 和 0xde 之间
      if (busid->bus >= 0xdc && busid->bus <= 0xde) {
        # 设置 CPU 核心集合的范围为 40 到 47
        hwloc_bitmap_set_range(cpuset, 40, 47);
        # 设置 CPU 核心集合的范围为 104 到 111
        hwloc_bitmap_set_range(cpuset, 104, 111);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xc0 和 0xc1 之间
      if (busid->bus >= 0xc0 && busid->bus <= 0xc1) {
        # 设置 CPU 核心集合的范围为 48 到 55
        hwloc_bitmap_set_range(cpuset, 48, 55);
        # 设置 CPU 核心集合的范围为 112 到 119
        hwloc_bitmap_set_range(cpuset, 112, 119);
        # 返回 1，表示设置成功
        return 1;
      }
      # 如果总线 ID 的总线号在 0xc4 和 0xc6 之间
      if (busid->bus >= 0xc4 && busid->bus <= 0xc6) {
        # 设置 CPU 核心集合的范围为 56 到 63
        hwloc_bitmap_set_range(cpuset, 56, 63);
        # 设置 CPU 核心集合的范围为 120 到 127
        hwloc_bitmap_set_range(cpuset, 120, 127);
        # 返回 1，表示设置成功
        return 1;
      }
    }
  }
  # 如果以上条件都不满足，则返回 0，表示设置失败
  return 0;
}

static struct hwloc_obj *
hwloc__pci_find_busid_parent(struct hwloc_topology *topology, struct hwloc_pcidev_attr_s *busid)
{
  // 分配一个位图用于存储 CPU 核心的集合
  hwloc_bitmap_t cpuset = hwloc_bitmap_alloc();
  // 定义一个指向父对象的指针
  hwloc_obj_t parent;
  // 定义一个变量用于标记是否强制匹配
  int forced = 0;
  // 定义一个变量用于标记是否禁用 quirks
  int noquirks = 0, got_quirked = 0;
  // 定义一个无符号整数变量
  unsigned i;
  // 定义一个整数变量用于存储错误码
  int err;

  // 输出调试信息，查找给定 PCI 总线标识的父对象
  hwloc_debug("Looking for parent of PCI busid %04x:%02x:%02x.%01x\n",
          busid->domain, busid->bus, busid->dev, busid->func);

  /* try to match a forced locality */
  // 如果拓扑结构中存在强制的本地性
  if (topology->pci_has_forced_locality) {
    for(i=0; i<topology->pci_forced_locality_nr; i++) {
      // 如果给定的总线标识在强制本地性范围内
      if (busid->domain == topology->pci_forced_locality[i].domain
      && busid->bus >= topology->pci_forced_locality[i].bus_first
      && busid->bus <= topology->pci_forced_locality[i].bus_last) {
    // 复制强制本地性的 CPU 核心集合
    hwloc_bitmap_copy(cpuset, topology->pci_forced_locality[i].cpuset);
    // 标记为强制匹配
    forced = 1;
    break;
      }
    }
    // 如果 PCI 本地性被强制，即使为空，也不让 quirks 改变操作系统报告的内容
    noquirks = 1;
  }

  /* deprecated force locality variables */
  // 如果没有强制匹配
  if (!forced) {
    const char *env;
    char envname[256];
    // 如果环境变量中有给定 PCI 总线标识的本地 CPU 集合
    snprintf(envname, sizeof(envname), "HWLOC_PCI_%04x_%02x_LOCALCPUS",
         busid->domain, busid->bus);
    env = getenv(envname);
    if (env) {
      static int reported = 0;
      // 如果拓扑结构中没有强制本地性，并且还没有报告过
      if (!topology->pci_has_forced_locality && !reported) {
        if (HWLOC_SHOW_ALL_ERRORS())
          // 输出警告信息，提示环境变量已经过时
          fprintf(stderr, "hwloc/pci: Environment variable %s is deprecated, please use HWLOC_PCI_LOCALITY instead.\n", env);
    reported = 1;
      }
      // 如果环境变量不为空
      if (*env) {
    // 强制使用环境变量中的 CPU 核心集合
    hwloc_debug("Overriding PCI locality using %s in the environment\n", envname);
    hwloc_bitmap_sscanf(cpuset, env);
    // 标记为强制匹配
    forced = 1;
      }
      // 如果环境变量存在，即使为空，也不让 quirks 改变操作系统报告的内容
      noquirks = 1;
  }
}

// 如果不是强制获取并且没有使用特殊规则，并且PCI位置规则未知或者已启用
if (!forced && !noquirks && topology->pci_locality_quirks /* either quirks are unknown yet, or some are enabled */) {
  // 在PCI拓扑中查找总线ID的父级位置规则
  err = hwloc__pci_find_busid_parent_quirk(topology, busid, cpuset);
  if (err > 0)
    got_quirked = 1;
}

// 如果不是强制获取并且没有使用特殊规则
if (!forced && !got_quirked) {
  /* 通过询问提供相关钩子的后端来获取cpuset。 */
  struct hwloc_backend *backend = topology->get_pci_busid_cpuset_backend;
  if (backend)
    err = backend->get_pci_busid_cpuset(backend, busid, cpuset);
  else
    err = -1;
  if (err < 0)
    /* 如果没有获取到任何信息，则假设此PCI总线连接到层次结构的顶部 */
    hwloc_bitmap_copy(cpuset, hwloc_topology_get_topology_cpuset(topology));
}

// 调试信息，显示将PCI总线连接到cpuset
hwloc_debug_bitmap("  will attach PCI bus to cpuset %s\n", cpuset);

// 通过完整cpuset查找并插入IO父级位置
parent = hwloc_find_insert_io_parent_by_complete_cpuset(topology, cpuset);
if (!parent) {
  /* 回退到根节点 */
  parent = hwloc_get_root_obj(topology);
}

// 释放cpuset内存
hwloc_bitmap_free(cpuset);
// 返回父级位置
return parent;
}
// 定义函数 hwloc_pcidisc_tree_attach，用于将 PCI 设备树附加到拓扑结构中
int
hwloc_pcidisc_tree_attach(struct hwloc_topology *topology, struct hwloc_obj *tree)
{
  enum hwloc_type_filter_e bfilter; // 声明枚举类型变量 bfilter

  if (!tree)
    /* 如果树为空，则找不到任何内容，退出 */
    return 0;

#ifdef HWLOC_DEBUG
  hwloc_debug("%s", "\nPCI hierarchy:\n"); // 输出调试信息
  hwloc_pci_traverse(NULL, tree, hwloc_pci_traverse_print_cb); // 遍历 PCI 设备树并打印信息
  hwloc_debug("%s", "\n"); // 输出调试信息
#endif

  bfilter = topology->type_filter[HWLOC_OBJ_BRIDGE]; // 获取拓扑结构中桥接设备的过滤器
  if (bfilter != HWLOC_TYPE_FILTER_KEEP_NONE) {
    tree = hwloc_pcidisc_add_hostbridges(topology, tree); // 如果过滤器不是保留空的，则添加主机桥接设备到树中
  }

  while (tree) {
    struct hwloc_obj *obj, *pciobj; // 声明对象指针
    struct hwloc_obj *parent; // 声明父对象指针
    struct hwloc_pci_locality_s *loc; // 声明 PCI 位置信息结构指针
    unsigned domain, bus_min, bus_max; // 声明无符号整型变量

    obj = tree; // 将树赋值给对象指针

    /* 主机桥接设备没有用于查找位置信息的 PCI 总线 ID，使用它们的第一个子设备 */
    if (obj->type == HWLOC_OBJ_BRIDGE && obj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_HOST)
      pciobj = obj->io_first_child; // 如果是主机桥接设备，则使用其第一个子设备
    else
      pciobj = obj; // 否则使用当前对象
    /* 现在我们有一个 PCI 设备或 PCI 桥接设备 */
    assert(pciobj->type == HWLOC_OBJ_PCI_DEVICE
       || (pciobj->type == HWLOC_OBJ_BRIDGE && pciobj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI)); // 断言，确保是 PCI 设备或 PCI 桥接设备

    if (obj->type == HWLOC_OBJ_BRIDGE && obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) {
      domain = obj->attr->bridge.downstream.pci.domain; // 获取桥接设备的域
      bus_min = obj->attr->bridge.downstream.pci.secondary_bus; // 获取桥接设备的次要总线
      bus_max = obj->attr->bridge.downstream.pci.subordinate_bus; // 获取桥接设备的从属总线
    } else {
      domain = pciobj->attr->pcidev.domain; // 获取 PCI 设备的域
      bus_min = pciobj->attr->pcidev.bus; // 获取 PCI 设备的总线
      bus_max = pciobj->attr->pcidev.bus; // 获取 PCI 设备的总线
    }

    /* 找到要附加 PCI 总线的位置 */
    parent = hwloc__pci_find_busid_parent(topology, &pciobj->attr->pcidev); // 查找要附加 PCI 总线的位置

    /* 如果可能，重用之前的位置信息 */
    if (topology->last_pci_locality
    && parent == topology->last_pci_locality->parent
    && domain == topology->last_pci_locality->domain
    && (bus_min == topology->last_pci_locality->bus_max
        || bus_min == topology->last_pci_locality->bus_max+1)) {
      // 如果 bus_min 等于最后一个 PCI 位置的最大总线号，或者等于最大总线号加一，则重用 PCI 位置直到总线号 %04x:%02x
      hwloc_debug("  Reusing PCI locality up to bus %04x:%02x\n",
          domain, bus_max);
      // 更新最后一个 PCI 位置的最大总线号
      topology->last_pci_locality->bus_max = bus_max;
      // 跳转到 done 标签处
      goto done;
    }

    // 分配内存给 loc
    loc = malloc(sizeof(*loc));
    if (!loc) {
      /* 回退到附加到根节点 */
      // 获取根节点对象
      parent = hwloc_get_root_obj(topology);
      // 跳转到 done 标签处
      goto done;
    }

    // 设置 loc 的域、最小总线号、最大总线号、父对象、cpuset
    loc->domain = domain;
    loc->bus_min = bus_min;
    loc->bus_max = bus_max;
    loc->parent = parent;
    loc->cpuset = hwloc_bitmap_dup(parent->cpuset);
    if (!loc->cpuset) {
      /* 回退到附加到根节点 */
      // 释放 loc 的内存
      free(loc);
      // 获取根节点对象
      parent = hwloc_get_root_obj(topology);
      // 跳转到 done 标签处
      goto done;
    }

    // 输出添加的 PCI 位置信息
    hwloc_debug("Adding PCI locality %s P#%u for bus %04x:[%02x:%02x]\n",
        hwloc_obj_type_string(parent->type), parent->os_index, loc->domain, loc->bus_min, loc->bus_max);
    if (topology->last_pci_locality) {
      // 设置 loc 的前一个和后一个指针，更新最后一个 PCI 位置的指针
      loc->prev = topology->last_pci_locality;
      loc->next = NULL;
      topology->last_pci_locality->next = loc;
      topology->last_pci_locality = loc;
    } else {
      // 设置 loc 的前一个和后一个指针，更新第一个和最后一个 PCI 位置的指针
      loc->prev = NULL;
      loc->next = NULL;
      topology->first_pci_locality = loc;
      topology->last_pci_locality = loc;
    }

  done:
    /* 出队该对象 */
    // 将 obj 从树中移除
    tree = obj->next_sibling;
    obj->next_sibling = NULL;
    // 将 obj 插入到 parent 对象下
    hwloc_insert_object_by_parent(topology, parent, obj);
  }

  // 返回 0
  return 0;
/*********************************
 * Finding PCI objects or parents
 */

# 通过总线ID查找父对象
struct hwloc_obj *
hwloc_pci_find_parent_by_busid(struct hwloc_topology *topology,
                   unsigned domain, unsigned bus, unsigned dev, unsigned func)
{
  struct hwloc_pcidev_attr_s busid;  # 定义总线ID属性结构体
  hwloc_obj_t parent;  # 定义父对象

  /* 尝试查找确切的总线ID */
  parent = hwloc_pci_find_by_busid(topology, domain, bus, dev, func);  # 调用函数查找总线ID
  if (parent)
    return parent;  # 如果找到则返回父对象

  /* 尝试查找该总线的位置 */
  busid.domain = domain;  # 设置总线ID的域
  busid.bus = bus;  # 设置总线ID的总线号
  busid.dev = dev;  # 设置总线ID的设备号
  busid.func = func;  # 设置总线ID的函数号
  return hwloc__pci_find_busid_parent(topology, &busid);  # 调用函数查找总线ID的父对象
}

/* 返回包含所需总线ID的最小对象 */
static struct hwloc_obj *
hwloc__pci_find_by_busid(hwloc_obj_t parent,
             unsigned domain, unsigned bus, unsigned dev, unsigned func)
{
  hwloc_obj_t child;  # 定义子对象

  for_each_io_child(child, parent) {  # 遍历父对象的每个IO子对象
    if (child->type == HWLOC_OBJ_PCI_DEVICE  # 如果子对象是PCI设备
    || (child->type == HWLOC_OBJ_BRIDGE  # 或者子对象是桥
        && child->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI)) {  # 且上游类型是PCI桥
      if (child->attr->pcidev.domain == domain  # 如果子对象的PCI域等于给定的域
      && child->attr->pcidev.bus == bus  # 且PCI总线等于给定的总线
      && child->attr->pcidev.dev == dev  # 且PCI设备等于给定的设备
      && child->attr->pcidev.func == func)  # 且PCI函数等于给定的函数
    /* 这就是正确的总线ID */
    return child;  # 返回子对象
      if (child->attr->pcidev.domain > domain  # 如果子对象的PCI域大于给定的域
      || (child->attr->pcidev.domain == domain  # 或者PCI域等于给定的域
          && child->attr->pcidev.bus > bus))  # 且PCI总线大于给定的总线
    /* 总线ID太高，之后找不到任何东西，返回父对象 */
    return parent;  # 返回父对象
      if (child->type == HWLOC_OBJ_BRIDGE  # 如果子对象是桥
      && child->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI  # 且下游类型是PCI桥
      && child->attr->bridge.downstream.pci.domain == domain  # 且下游PCI域等于给定的域
      && child->attr->bridge.downstream.pci.secondary_bus <= bus  # 且下游PCI次要总线小于等于给定的总线
      && child->attr->bridge.downstream.pci.subordinate_bus >= bus)  # 且下游PCI从属总线大于等于给定的总线
    /* 不是正确的总线ID，但它包含在该桥下面的总线中 */
    return hwloc__pci_find_by_busid(child, domain, bus, dev, func);  # 递归调用函数查找总线ID
    } else if (child->type == HWLOC_OBJ_BRIDGE  # 如果子节点类型是桥接设备
           && child->attr->bridge.upstream_type != HWLOC_OBJ_BRIDGE_PCI  # 并且上游类型不是 PCI
           && child->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI  # 并且下游类型是 PCI
           /* 非 PCI 到 PCI 桥接，只需查看从属总线 */
           && child->attr->bridge.downstream.pci.domain == domain  # 并且下游 PCI 设备的域与给定的域相匹配
           && child->attr->bridge.downstream.pci.secondary_bus <= bus  # 并且下游 PCI 设备的次要总线小于等于给定的总线
           && child->attr->bridge.downstream.pci.subordinate_bus >= bus) {  # 并且下游 PCI 设备的从属总线大于等于给定的总线
      /* 包含我们的总线，递归 */
      return hwloc__pci_find_by_busid(child, domain, bus, dev, func);  # 包含我们的总线，递归查找
    }
  }
  /* 没有找到任何内容，返回父节点 */
  return parent;  # 没有找到任何内容，返回父节点
# 根据总线 ID 查找对应的 PCI 设备对象
struct hwloc_obj *hwloc_pci_find_by_busid(struct hwloc_topology *topology,
            unsigned domain, unsigned bus, unsigned dev, unsigned func)
{
  struct hwloc_pci_locality_s *loc;  # 定义一个指向 hwloc_pci_locality_s 结构体的指针 loc
  hwloc_obj_t root = hwloc_get_root_obj(topology);  # 获取拓扑结构的根对象
  hwloc_obj_t parent = NULL;  # 定义一个指向 hwloc_obj 结构体的指针 parent，初始化为 NULL

  hwloc_debug("pcidisc looking for bus id %04x:%02x:%02x.%01x\n", domain, bus, dev, func);  # 打印调试信息，查找对应总线 ID 的 PCI 设备
  loc = topology->first_pci_locality;  # 将拓扑结构中的第一个 PCI 本地性赋值给 loc
  while (loc) {  # 进入循环，遍历 PCI 本地性
    if (loc->domain == domain && loc->bus_min <= bus && loc->bus_max >= bus) {  # 如果找到对应的域和总线范围
      parent = loc->parent;  # 将当前 PCI 本地性的父对象赋值给 parent
      assert(parent);  # 断言 parent 不为空
      hwloc_debug("  found pci locality for %04x:[%02x:%02x]\n",
          loc->domain, loc->bus_min, loc->bus_max);  # 打印调试信息，找到对应的 PCI 本地性
      break;  # 跳出循环
    }
    loc = loc->next;  # 将下一个 PCI 本地性赋值给 loc
  }
  # 如果未能插入本地性，也查看根对象
  if (!parent)  # 如果 parent 为空
    parent = root;  # 将根对象赋值给 parent

  hwloc_debug("  looking for bus %04x:%02x:%02x.%01x below %s P#%u\n",
          domain, bus, dev, func,
          hwloc_obj_type_string(parent->type), parent->os_index);  # 打印调试信息，查找对应总线 ID 的 PCI 设备在指定父对象下
  parent = hwloc__pci_find_by_busid(parent, domain, bus, dev, func);  # 根据总线 ID 在指定父对象下查找对应的 PCI 设备
  if (parent == root) {  # 如果找到的父对象是根对象
    hwloc_debug("  found nothing better than root object, ignoring\n");  # 打印调试信息，找不到比根对象更好的对象，忽略
    return NULL;  # 返回空指针
  } else {  # 否则
    if (parent->type == HWLOC_OBJ_PCI_DEVICE  # 如果父对象类型是 PCI 设备
    || (parent->type == HWLOC_OBJ_BRIDGE && parent->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI))  # 或者是桥接设备且上游类型是 PCI
      hwloc_debug("  found busid %04x:%02x:%02x.%01x\n",
          parent->attr->pcidev.domain, parent->attr->pcidev.bus,
          parent->attr->pcidev.dev, parent->attr->pcidev.func);  # 打印调试信息，找到对应总线 ID 的 PCI 设备
    else  # 否则
      hwloc_debug("  found parent %s P#%u\n",
          hwloc_obj_type_string(parent->type), parent->os_index);  # 打印调试信息，找到对应总线 ID 的 PCI 设备的父对象
    return parent;  # 返回找到的 PCI 设备对象
  }
}


/*******************************
 * Parsing the PCI Config Space
 */

#define HWLOC_PCI_STATUS 0x06  # 定义 PCI 配置空间中状态寄存器的偏移量
#define HWLOC_PCI_STATUS_CAP_LIST 0x10  # 定义 PCI 配置空间中状态寄存器中是否有能力列表的位
#define HWLOC_PCI_CAPABILITY_LIST 0x34  # 定义 PCI 配置空间中能力列表指针寄存器的偏移量
#define HWLOC_PCI_CAP_LIST_ID 0  # 定义能力列表项中 ID 的偏移量
#define HWLOC_PCI_CAP_LIST_NEXT 1  # 定义能力列表项中下一个指针的偏移量

unsigned
hwloc_pcidisc_find_cap(const unsigned char *config, unsigned cap)  # 定义函数，查找 PCI 配置空间中的能力
{
  // 创建一个长度为256的无符号字符数组，用于标记已经遍历过的指针
  unsigned char seen[256] = { 0 };
  // 声明一个无符号字符指针，用于遍历配置空间的指针，使用无符号字符以确保不超出256字节的配置空间
  unsigned char ptr; 

  // 如果配置中的PCI状态不包含CAP_LIST，则返回0
  if (!(config[HWLOC_PCI_STATUS] & HWLOC_PCI_STATUS_CAP_LIST))
    return 0;

  // 遍历配置中的PCI能力列表指针，直到下一个指针为0时退出循环
  for (ptr = config[HWLOC_PCI_CAPABILITY_LIST] & ~3;
       ptr; 
       ptr = config[ptr + HWLOC_PCI_CAP_LIST_NEXT] & ~3) {
    unsigned char id;

    // 如果指针已经遍历过，则跳出循环
    if (seen[ptr])
      break;
    seen[ptr] = 1;

    // 获取指针处的能力ID
    id = config[ptr + HWLOC_PCI_CAP_LIST_ID];
    // 如果能力ID等于cap，则返回指针
    if (id == cap)
      return ptr;
    // 如果能力ID为0或0xff，则跳出循环
    if (id == 0xff) 
      break;
  }
  // 如果没有找到匹配的能力ID，则返回0
  return 0;
}

// 定义PCI扩展链接状态寄存器的偏移量和速度、宽度的掩码
#define HWLOC_PCI_EXP_LNKSTA 0x12
#define HWLOC_PCI_EXP_LNKSTA_SPEED 0x000f
#define HWLOC_PCI_EXP_LNKSTA_WIDTH 0x03f0

// 查找PCI设备链接速度的函数
int
hwloc_pcidisc_find_linkspeed(const unsigned char *config,
                 unsigned offset, float *linkspeed)
{
  // 定义无符号整型变量linksta, speed, width
  unsigned linksta, speed, width;
  // 定义浮点型变量lanespeed

  // 从config数组的偏移量offset处复制4个字节到linksta变量中
  memcpy(&linksta, &config[offset + HWLOC_PCI_EXP_LNKSTA], 4);
  // 从linksta中获取PCIe的速度
  speed = linksta & HWLOC_PCI_EXP_LNKSTA_SPEED; /* PCIe generation */
  // 从linksta中获取PCIe的通道宽度
  width = (linksta & HWLOC_PCI_EXP_LNKSTA_WIDTH) >> 4; /* how many lanes */
  /*
   * 这些只是单向带宽。
   *
   * Gen1使用NRZ和8/10编码。
   * PCIe Gen1 = 每通道2.5GT/s信号速率 x 8/10    =  0.25GB/s数据速率
   * PCIe Gen2 = 每通道5GT/s信号速率 x 8/10    =  0.5 GB/s数据速率
   * Gen3切换到NRZ和128/130编码。
   * PCIe Gen3 = 每通道8GT/s信号速率 x 128/130 =  1   GB/s数据速率
   * PCIe Gen4 = 每通道16GT/s信号速率 x 128/130 =  2   GB/s数据速率
   * PCIe Gen5 = 每通道32GT/s信号速率 x 128/130 =  4   GB/s数据速率
   * Gen6切换到PAM和242/256 FLIT (242B有效载荷由8B CRC + 6B FEC保护)。
   * PCIe Gen6 = 每通道64GT/s信号速率 x 242/256 =  8   GB/s数据速率
   * PCIe Gen7 = 每通道128GT/s信号速率 x 242/256 = 16   GB/s数据速率
   */

  /* lanespeed以Gbit/s为单位 */
  // 如果速度小于等于2，则lanespeed为2.5乘以速度乘以0.8
  if (speed <= 2)
    lanespeed = 2.5f * speed * 0.8f;
  // 如果速度小于等于5，则lanespeed为8乘以2的（速度-3）次方乘以128/130
  else if (speed <= 5)
    lanespeed = 8.0f * (1<<(speed-3)) * 128/130;
  // 否则lanespeed为8乘以2的（速度-3）次方乘以242/256，假设Gen8将是256GT/s等等
  else
    lanespeed = 8.0f * (1<<(speed-3)) * 242/256; /* assume Gen8 will be 256 GT/s and so on */

  /* linkspeed以GB/s为单位 */
  // 计算linkspeed，lanespeed乘以宽度再除以8
  *linkspeed = lanespeed * width / 8;
  // 返回0表示成功
  return 0;
}

// 定义常量HWLOC_PCI_HEADER_TYPE为0x0e
#define HWLOC_PCI_HEADER_TYPE 0x0e
// 定义常量HWLOC_PCI_HEADER_TYPE_BRIDGE为1
#define HWLOC_PCI_HEADER_TYPE_BRIDGE 1
// 定义常量HWLOC_PCI_CLASS_BRIDGE_PCI为0x0604
#define HWLOC_PCI_CLASS_BRIDGE_PCI 0x0604

// 检查PCI设备类型
hwloc_obj_type_t
hwloc_pcidisc_check_bridge_type(unsigned device_class, const unsigned char *config)
{
  unsigned char headertype;

  // 如果设备类别不是PCI桥接设备，则返回PCI设备类型
  if (device_class != HWLOC_PCI_CLASS_BRIDGE_PCI)
    return HWLOC_OBJ_PCI_DEVICE;
  // 从config数组的HWLOC_PCI_HEADER_TYPE处获取头类型
  headertype = config[HWLOC_PCI_HEADER_TYPE] & 0x7f;
  // 如果头类型是PCI桥接类型，则返回桥接类型，否则返回PCI设备类型
  return (headertype == HWLOC_PCI_HEADER_TYPE_BRIDGE)
    ? HWLOC_OBJ_BRIDGE : HWLOC_OBJ_PCI_DEVICE;
}

// 定义常量HWLOC_PCI_PRIMARY_BUS为0x18
#define HWLOC_PCI_PRIMARY_BUS 0x18
#define HWLOC_PCI_SECONDARY_BUS 0x19
#define HWLOC_PCI_SUBORDINATE_BUS 0x1a

int
hwloc_pcidisc_find_bridge_buses(unsigned domain, unsigned bus, unsigned dev, unsigned func,
                unsigned *secondary_busp, unsigned *subordinate_busp,
                const unsigned char *config)
{
  unsigned secondary_bus, subordinate_bus;

  if (config[HWLOC_PCI_PRIMARY_BUS] != bus) {
    /* 如果配置空间中的 PCI_PRIMARY_BUS 不等于总线号，则输出调试信息 */
    hwloc_debug("  %04x:%02x:%02x.%01x bridge with (ignored) invalid PCI_PRIMARY_BUS %02x\n",
        domain, bus, dev, func, config[HWLOC_PCI_PRIMARY_BUS]);
  }

  secondary_bus = config[HWLOC_PCI_SECONDARY_BUS];
  subordinate_bus = config[HWLOC_PCI_SUBORDINATE_BUS];

  if (secondary_bus <= bus
      || subordinate_bus <= bus
      || secondary_bus > subordinate_bus) {
    /* 如果次级总线号小于等于总线号，或者从属总线号小于等于总线号，或者次级总线号大于从属总线号，则输出调试信息并返回-1 */
    hwloc_debug("  %04x:%02x:%02x.%01x bridge has invalid secondary-subordinate buses [%02x-%02x]\n",
        domain, bus, dev, func,
        secondary_bus, subordinate_bus);
    return -1;
  }

  *secondary_busp = secondary_bus;
  *subordinate_busp = subordinate_bus;
  return 0;
}


/****************
 * Class Strings
 */

const char *
hwloc_pci_class_string(unsigned short class_id)
{
  /* 参考 https://pci-ids.ucw.cz/read/PD/ */
  switch ((class_id & 0xff00) >> 8) {
    case 0x00:
      switch (class_id) {
    case 0x0001: return "VGA";
      }
      break;
    case 0x01:
      switch (class_id) {
    # 根据不同的 case 值返回对应的设备类型字符串
    case 0x0100: return "SCSI";
    case 0x0101: return "IDE";
    case 0x0102: return "Floppy";
    case 0x0103: return "IPI";
    case 0x0104: return "RAID";
    case 0x0105: return "ATA";
    case 0x0106: return "SATA";
    case 0x0107: return "SAS";
    case 0x0108: return "NVMExp";
      }
      return "Storage";
    case 0x02:
      switch (class_id) {
    case 0x0200: return "Ethernet";
    case 0x0201: return "TokenRing";
    case 0x0202: return "FDDI";
    case 0x0203: return "ATM";
    case 0x0204: return "ISDN";
    case 0x0205: return "WorldFip";
    case 0x0206: return "PICMG";
    case 0x0207: return "InfiniBand";
    case 0x0208: return "Fabric";
      }
      return "Network";
    case 0x03:
      switch (class_id) {
    case 0x0300: return "VGA";
    case 0x0301: return "XGA";
    case 0x0302: return "3D";
      }
      return "Display";
    case 0x04:
      switch (class_id) {
    case 0x0400: return "MultimediaVideo";
    case 0x0401: return "MultimediaAudio";
    case 0x0402: return "Telephony";
    case 0x0403: return "AudioDevice";
      }
      return "Multimedia";
    case 0x05:
      switch (class_id) {
    case 0x0500: return "RAM";
    case 0x0501: return "Flash";
        case 0x0502: return "CXLMem";
      }
      return "Memory";
    case 0x06:
      switch (class_id) {
    case 0x0600: return "HostBridge";
    case 0x0601: return "ISABridge";
    case 0x0602: return "EISABridge";
    case 0x0603: return "MicroChannelBridge";
    case 0x0604: return "PCIBridge";
    case 0x0605: return "PCMCIABridge";
    case 0x0606: return "NubusBridge";
    case 0x0607: return "CardBusBridge";
    case 0x0608: return "RACEwayBridge";
    case 0x0609: return "SemiTransparentPCIBridge";
    case 0x060a: return "InfiniBandPCIHostBridge";
      }
      return "Bridge";
    case 0x07:
      switch (class_id) {
    case 0x0700: return "Serial";
    case 0x0701: return "Parallel";
    case 0x0702: return "MultiportSerial";
    case 0x0703: return "Model";
    # 其他情况返回默认值
    # 根据不同的 case 值返回对应的设备类型
    case 0x0704: return "GPIB";  # 如果 case 值为 0x0704，则返回 "GPIB"
    case 0x0705: return "SmartCard";  # 如果 case 值为 0x0705，则返回 "SmartCard"
      }
      return "Communication";  # 如果以上 case 值都不匹配，则返回 "Communication"
    case 0x08:
      switch (class_id) {
    case 0x0800: return "PIC";  # 如果 case 值为 0x0800，则返回 "PIC"
    case 0x0801: return "DMA";  # 如果 case 值为 0x0801，则返回 "DMA"
    case 0x0802: return "Timer";  # 如果 case 值为 0x0802，则返回 "Timer"
    case 0x0803: return "RTC";  # 如果 case 值为 0x0803，则返回 "RTC"
    case 0x0804: return "PCIHotPlug";  # 如果 case 值为 0x0804，则返回 "PCIHotPlug"
    case 0x0805: return "SDHost";  # 如果 case 值为 0x0805，则返回 "SDHost"
    case 0x0806: return "IOMMU";  # 如果 case 值为 0x0806，则返回 "IOMMU"
      }
      return "SystemPeripheral";  # 如果以上 case 值都不匹配，则返回 "SystemPeripheral"
    case 0x09:
      switch (class_id) {
    case 0x0900: return "Keyboard";  # 如果 case 值为 0x0900，则返回 "Keyboard"
    case 0x0901: return "DigitizerPen";  # 如果 case 值为 0x0901，则返回 "DigitizerPen"
    case 0x0902: return "Mouse";  # 如果 case 值为 0x0902，则返回 "Mouse"
    case 0x0903: return "Scanern";  # 如果 case 值为 0x0903，则返回 "Scanern"
    case 0x0904: return "Gameport";  # 如果 case 值为 0x0904，则返回 "Gameport"
      }
      return "Input";  # 如果以上 case 值都不匹配，则返回 "Input"
    case 0x0a:
      return "DockingStation";  # 如果 case 值为 0x0a，则返回 "DockingStation"
    case 0x0b:
      switch (class_id) {
    case 0x0b00: return "386";  # 如果 case 值为 0x0b00，则返回 "386"
    case 0x0b01: return "486";  # 如果 case 值为 0x0b01，则返回 "486"
    case 0x0b02: return "Pentium";  # 如果 case 值为 0x0b02，则返回 "Pentium"
/* 0x0b03 and 0x0b04 might be Pentium and P6 ? */
    case 0x0b10: return "Alpha";  // 如果 class_id 为 0x0b10，返回 "Alpha"
    case 0x0b20: return "PowerPC";  // 如果 class_id 为 0x0b20，返回 "PowerPC"
    case 0x0b30: return "MIPS";  // 如果 class_id 为 0x0b30，返回 "MIPS"
    case 0x0b40: return "Co-Processor";  // 如果 class_id 为 0x0b40，返回 "Co-Processor"
      }
      return "Processor";  // 如果不匹配任何 case，返回 "Processor"
    case 0x0c:
      switch (class_id) {
    case 0x0c00: return "FireWire";  // 如果 class_id 为 0x0c00，返回 "FireWire"
    case 0x0c01: return "ACCESS";  // 如果 class_id 为 0x0c01，返回 "ACCESS"
    case 0x0c02: return "SSA";  // 如果 class_id 为 0x0c02，返回 "SSA"
    case 0x0c03: return "USB";  // 如果 class_id 为 0x0c03，返回 "USB"
    case 0x0c04: return "FibreChannel";  // 如果 class_id 为 0x0c04，返回 "FibreChannel"
    case 0x0c05: return "SMBus";  // 如果 class_id 为 0x0c05，返回 "SMBus"
    case 0x0c06: return "InfiniBand";  // 如果 class_id 为 0x0c06，返回 "InfiniBand"
    case 0x0c07: return "IPMI-SMIC";  // 如果 class_id 为 0x0c07，返回 "IPMI-SMIC"
    case 0x0c08: return "SERCOS";  // 如果 class_id 为 0x0c08，返回 "SERCOS"
    case 0x0c09: return "CANBUS";  // 如果 class_id 为 0x0c09，返回 "CANBUS"
      }
      return "SerialBus";  // 如果不匹配任何 case，返回 "SerialBus"
    case 0x0d:
      switch (class_id) {
    case 0x0d00: return "IRDA";  // 如果 class_id 为 0x0d00，返回 "IRDA"
    case 0x0d01: return "ConsumerIR";  // 如果 class_id 为 0x0d01，返回 "ConsumerIR"
    case 0x0d10: return "RF";  // 如果 class_id 为 0x0d10，返回 "RF"
    case 0x0d11: return "Bluetooth";  // 如果 class_id 为 0x0d11，返回 "Bluetooth"
    case 0x0d12: return "Broadband";  // 如果 class_id 为 0x0d12，返回 "Broadband"
    case 0x0d20: return "802.1a";  // 如果 class_id 为 0x0d20，返回 "802.1a"
    case 0x0d21: return "802.1b";  // 如果 class_id 为 0x0d21，返回 "802.1b"
      }
      return "Wireless";  // 如果不匹配任何 case，返回 "Wireless"
    case 0x0e:
      switch (class_id) {
    case 0x0e00: return "I2O";  // 如果 class_id 为 0x0e00，返回 "I2O"
      }
      return "Intelligent";  // 如果不匹配任何 case，返回 "Intelligent"
    case 0x0f:
      return "Satellite";  // 如果 class_id 为 0x0f，返回 "Satellite"
    case 0x10:
      return "Encryption";  // 如果 class_id 为 0x10，返回 "Encryption"
    case 0x11:
      return "SignalProcessing";  // 如果 class_id 为 0x11，返回 "SignalProcessing"
    case 0x12:
      return "ProcessingAccelerator";  // 如果 class_id 为 0x12，返回 "ProcessingAccelerator"
    case 0x13:
      return "Instrumentation";  // 如果 class_id 为 0x13，返回 "Instrumentation"
    case 0x40:
      return "Co-Processor";  // 如果 class_id 为 0x40，返回 "Co-Processor"
  }
  return "Other";  // 如果不匹配任何 case，返回 "Other"
}
```