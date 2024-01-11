# `xmrig\src\3rdparty\hwloc\include\private\solaris-chiptype.h`

```
/*
 * 版权声明，版权归 Oracle 及其关联公司所有
 *
 * 版权归 Inria 所有
 * $COPYRIGHT$
 *
 * 可能会有其他版权声明
 *
 * $HEADER$
 */
 
#ifdef HWLOC_INSIDE_PLUGIN
/*
 * 这些声明仅供内部使用，插件无法访问这些声明
 * (下面的函数是内部静态符号)
 */
#error 该文件不应该在插件中使用
#endif

#ifndef HWLOC_PRIVATE_SOLARIS_CHIPTYPE_H
#define HWLOC_PRIVATE_SOLARIS_CHIPTYPE_H

// 定义 Solaris 系统芯片信息的结构体
struct hwloc_solaris_chip_info_s {
  char *model; // 芯片型号
  char *type; // 芯片类型
  /* L1i, L1d, L2, L3 */
#define HWLOC_SOLARIS_CHIP_INFO_L1I 0
#define HWLOC_SOLARIS_CHIP_INFO_L1D 1
#define HWLOC_SOLARIS_CHIP_INFO_L2I 2
#define HWLOC_SOLARIS_CHIP_INFO_L2D 3
#define HWLOC_SOLARIS_CHIP_INFO_L3  4
  long cache_size[5]; // 缓存大小，如果不需要则设置为 -1
  unsigned cache_linesize[5]; // 缓存行大小
  unsigned cache_associativity[5]; // 缓存关联度
  int l2_unified; // L2 缓存是否统一
};

// 获取 Solaris 系统芯片信息，出错时将结构体清零
extern void hwloc_solaris_get_chip_info(struct hwloc_solaris_chip_info_s *info);

#endif /* HWLOC_PRIVATE_SOLARIS_CHIPTYPE_H */
```