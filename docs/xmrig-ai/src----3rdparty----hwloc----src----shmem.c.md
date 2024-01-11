# `xmrig\src\3rdparty\hwloc\src\shmem.c`

```
/*
 * 版权所有 © 2017-2020 Inria。
 * 保留所有权利。请参阅顶层目录中的COPYING文件。
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "hwloc/shmem.h"
#include "private/private.h"

#ifndef HWLOC_WIN_SYS

#include <sys/mman.h>
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <assert.h>

#define HWLOC_SHMEM_HEADER_VERSION 1

struct hwloc_shmem_header {
  uint32_t header_version; /* 检查版本 */
  uint32_t header_length; /* 文件/映射中实际拓扑开始的位置 */
  uint64_t mmap_address; /* 传递给mmap的虚拟地址 */
  uint64_t mmap_length; /* 传递给mmap的长度（包括头部） */
};

#define HWLOC_SHMEM_MALLOC_ALIGN 8UL

static void *
tma_shmem_malloc(struct hwloc_tma * tma,
         size_t length)
{
  void *current = tma->data;
  tma->data = (char*)tma->data  + ((length + HWLOC_SHMEM_MALLOC_ALIGN - 1) & ~(HWLOC_SHMEM_MALLOC_ALIGN - 1));
  return current;

}

static void *
tma_get_length_malloc(struct hwloc_tma * tma,
              size_t length)
{
  size_t *tma_length = tma->data;
  *tma_length += (length + HWLOC_SHMEM_MALLOC_ALIGN - 1) & ~(HWLOC_SHMEM_MALLOC_ALIGN - 1);
  return malloc(length);

}

int
hwloc_shmem_topology_get_length(hwloc_topology_t topology,
                size_t *lengthp,
                unsigned long flags)
{
  hwloc_topology_t new;
  struct hwloc_tma tma;
  size_t length = 0;
  unsigned long pagesize = hwloc_getpagesize(); /* 对mmap()进行全页舍入 */
  int err;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  tma.malloc = tma_get_length_malloc;
  tma.dontfree = 0;
  tma.data = &length;

  err = hwloc__topology_dup(&new, topology, &tma);
  if (err < 0)
    return err;
  hwloc_topology_destroy(new);

  *lengthp = (sizeof(struct hwloc_shmem_header) + length + pagesize - 1) & ~(pagesize - 1);
  return 0;
}

int
# 将拓扑结构写入共享内存
hwloc_shmem_topology_write(hwloc_topology_t topology,
               int fd, hwloc_uint64_t fileoffset,
               void *mmap_address, size_t length,
               unsigned long flags)
{
  hwloc_topology_t new;  # 新的拓扑结构
  struct hwloc_tma tma;  # 内存分配器
  struct hwloc_shmem_header header;  # 共享内存头部信息
  void *mmap_res;  # 映射的内存地址
  int err;  # 错误码

  if (flags) {  # 如果标志不为0
    errno = EINVAL;  # 设置错误码为无效参数
    return -1;  # 返回-1
  }

  # 刷新旧拓扑结构的距离，以防止无效距离的重复分配
  hwloc_internal_distances_refresh(topology);
  hwloc_internal_memattrs_refresh(topology);

  header.header_version = HWLOC_SHMEM_HEADER_VERSION;  # 设置共享内存头部版本号
  header.header_length = sizeof(header);  # 设置头部长度
  header.mmap_address = (uintptr_t) mmap_address;  # 设置映射地址
  header.mmap_length = length;  # 设置映射长度

  err = lseek(fd, fileoffset, SEEK_SET);  # 移动文件偏移量
  if (err < 0)
    return -1;

  err = write(fd, &header, sizeof(header));  # 写入头部信息到文件
  if (err != sizeof(header))
    return -1;

  err = ftruncate(fd, fileoffset + length);  # 调整文件大小
  if (err < 0)
    return -1;

  mmap_res = mmap(mmap_address, length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, fileoffset);  # 映射共享内存
  if (mmap_res == MAP_FAILED)
    return -1;
  if (mmap_res != mmap_address) {  # 如果映射地址不符合预期
    munmap(mmap_res, length);  # 解除映射
    errno = EBUSY;  # 设置错误码为设备忙
    return -1;
  }

  tma.malloc = tma_shmem_malloc;  # 设置内存分配器的malloc函数
  tma.dontfree = 1;  # 设置不释放标志
  tma.data = (char *)mmap_res + sizeof(header);  # 设置数据指针
  err = hwloc__topology_dup(&new, topology, &tma);  # 复制拓扑结构
  if (err < 0)
    return err;
  assert((char*)new == (char*)mmap_address + sizeof(header));  # 断言新拓扑结构的地址符合预期

  assert((char *)mmap_res <= (char *)mmap_address + length);  # 断言映射地址范围符合预期

  # 刷新新的距离/内存属性，以便采用者可以在不刷新只读共享内存映射的情况下使用它们
  hwloc_internal_distances_refresh(new);
  hwloc_internal_memattrs_refresh(topology);

  # 保存拓扑结构后，释放资源
  munmap(mmap_address, length);  # 解除映射
  hwloc_components_fini();  # 组件结束

  return 0;  # 返回成功
}

int
// 采用共享内存方式采用给定的拓扑结构
hwloc_shmem_topology_adopt(hwloc_topology_t *topologyp,
               int fd, hwloc_uint64_t fileoffset,
               void *mmap_address, size_t length,
               unsigned long flags)
{
  hwloc_topology_t new, old;  // 定义两个拓扑结构指针变量
  struct hwloc_shmem_header header;  // 定义共享内存头部结构变量
  void *mmap_res;  // 定义内存映射结果变量
  int err;  // 定义错误码变量

  if (flags) {  // 如果标志位不为0
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回-1
  }

  err = lseek(fd, fileoffset, SEEK_SET);  // 移动文件描述符到指定位置
  if (err < 0)  // 如果移动失败
    return -1;  // 返回-1

  err = read(fd, &header, sizeof(header));  // 从文件中读取共享内存头部结构
  if (err != sizeof(header))  // 如果读取长度不等于头部结构长度
    return -1;  // 返回-1

  if (header.header_version != HWLOC_SHMEM_HEADER_VERSION  // 如果头部版本不匹配
      || header.header_length != sizeof(header)  // 或者头部长度不匹配
      || header.mmap_address != (uintptr_t) mmap_address  // 或者内存映射地址不匹配
      || header.mmap_length != length) {  // 或者内存映射长度不匹配
    errno = EINVAL;  // 设置错误码为无效参数
    return -1;  // 返回-1
  }

  mmap_res = mmap(mmap_address, length, PROT_READ, MAP_SHARED, fd, fileoffset);  // 创建共享内存映射
  if (mmap_res == MAP_FAILED)  // 如果映射失败
    return -1;  // 返回-1
  if (mmap_res != mmap_address) {  // 如果映射结果不等于指定地址
    errno = EBUSY;  // 设置错误码为设备忙
    goto out_with_mmap;  // 跳转到释放内存映射的标签
  }

  old = (hwloc_topology_t)((char*)mmap_address + sizeof(header));  // 计算旧拓扑结构的地址
  if (hwloc_topology_abi_check(old) < 0) {  // 如果ABI检查失败
    errno = EINVAL;  // 设置错误码为无效参数
    goto out_with_mmap;  // 跳转到释放内存映射的标签
  }

  /* enforced by dup() inside shmem_topology_write() */
  assert(old->is_loaded);  // 断言旧拓扑结构已加载
  assert(old->backends == NULL);  // 断言旧拓扑结构的后端为空
  assert(old->get_pci_busid_cpuset_backend == NULL);  // 断言旧拓扑结构的PCI总线ID对应的CPU集合后端为空

  hwloc_components_init();  // 初始化组件

  /* duplicate the topology object so that we ca change use local binding_hooks
   * (those are likely not mapped at the same location in both processes).
   */
  new = malloc(sizeof(struct hwloc_topology));  // 分配新拓扑结构的内存空间
  if (!new)  // 如果分配失败
    # 跳转到组件释放的标签，执行释放操作
    goto out_with_components;
    # 复制旧的数据到新的内存空间
    memcpy(new, old, sizeof(*old));
    # 将新的 tma 指针设置为 NULL
    new->tma = NULL;
    # 将新的 adopted_shmem_addr 设置为 mmap_address
    new->adopted_shmem_addr = mmap_address;
    # 将新的 adopted_shmem_length 设置为 length
    new->adopted_shmem_length = length;
    # 将新的 topology_abi 设置为 HWLOC_TOPOLOGY_ABI
    new->topology_abi = HWLOC_TOPOLOGY_ABI;
    # 设置绑定钩子将触及支持数组，因此也需要复制它们
    new->support.discovery = malloc(sizeof(*new->support.discovery));
    new->support.cpubind = malloc(sizeof(*new->support.cpubind));
    new->support.membind = malloc(sizeof(*new->support.membind));
    new->support.misc = malloc(sizeof(*new->support.misc));
    # 检查内存分配是否成功，如果有一个失败则跳转到释放支持数组的标签
    if (!new->support.discovery || !new->support.cpubind || !new->support.membind || !new->support.misc)
        goto out_with_support;
    # 将旧的支持数组数据复制到新的支持数组
    memcpy(new->support.discovery, old->support.discovery, sizeof(*new->support.discovery));
    memcpy(new->support.cpubind, old->support.cpubind, sizeof(*new->support.cpubind));
    memcpy(new->support.membind, old->support.membind, sizeof(*new->support.membind));
    memcpy(new->support.misc, old->support.misc, sizeof(*new->support.misc));
    # 设置新的绑定钩子
    hwloc_set_binding_hooks(new);
    # 清空指向写入进程函数的用户数据回调
    new->userdata_export_cb = NULL;
    new->userdata_import_cb = NULL;
#ifndef HWLOC_DEBUG
  // 如果未定义 HWLOC_DEBUG 宏，则执行以下代码
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    // 检查新拓扑结构的一致性
    hwloc_topology_check(new);

  // 将新拓扑结构指针赋给传入指针
  *topologyp = new;
  // 返回成功
  return 0;

 out_with_support:
  // 释放内存并返回错误
  free(new->support.discovery);
  free(new->support.cpubind);
  free(new->support.membind);
  free(new->support.misc);
  free(new);
 out_with_components:
  // 释放组件
  hwloc_components_fini();
 out_with_mmap:
  // 解除内存映射并返回错误
  munmap(mmap_res, length);
  return -1;
}

void
hwloc__topology_disadopt(hwloc_topology_t topology)
{
  // 释放组件
  hwloc_components_fini();
  // 解除内存映射
  munmap(topology->adopted_shmem_addr, topology->adopted_shmem_length);
  // 释放内存
  free(topology->support.discovery);
  free(topology->support.cpubind);
  free(topology->support.membind);
  free(topology->support.misc);
  free(topology);
}

#else /* HWLOC_WIN_SYS */

int
hwloc_shmem_topology_get_length(hwloc_topology_t topology __hwloc_attribute_unused,
                size_t *lengthp __hwloc_attribute_unused,
                unsigned long flags __hwloc_attribute_unused)
{
  // 设置错误码并返回错误
  errno = ENOSYS;
  return -1;
}

int
hwloc_shmem_topology_write(hwloc_topology_t topology __hwloc_attribute_unused,
               int fd __hwloc_attribute_unused, hwloc_uint64_t fileoffset __hwloc_attribute_unused,
               void *mmap_address __hwloc_attribute_unused, size_t length __hwloc_attribute_unused,
               unsigned long flags __hwloc_attribute_unused)
{
  // 设置错误码并返回错误
  errno = ENOSYS;
  return -1;
}

int
hwloc_shmem_topology_adopt(hwloc_topology_t *topologyp __hwloc_attribute_unused,
               int fd __hwloc_attribute_unused, hwloc_uint64_t fileoffset __hwloc_attribute_unused,
               void *mmap_address __hwloc_attribute_unused, size_t length __hwloc_attribute_unused,
               unsigned long flags __hwloc_attribute_unused)
{
  // 设置错误码并返回错误
  errno = ENOSYS;
  return -1;
}

void
hwloc__topology_disadopt(hwloc_topology_t topology __hwloc_attribute_unused)
{
  // 空函数
}

#endif /* HWLOC_WIN_SYS */
```