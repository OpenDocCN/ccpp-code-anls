# `xmrig\src\3rdparty\hwloc\src\diff.c`

```
/*
 * 版权所有 © 2013-2022 Inria。
 * 请查看顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h"
#include "private/private.h"
#include "private/misc.h"

// 销毁拓扑差异对象
int hwloc_topology_diff_destroy(hwloc_topology_diff_t diff)
{
    hwloc_topology_diff_t next;
    // 遍历拓扑差异对象链表
    while (diff) {
        // 保存下一个拓扑差异对象
        next = diff->generic.next;
        // 根据拓扑差异对象的类型进行处理
        switch (diff->generic.type) {
        default:
            break;
        case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR:
            // 根据拓扑差异对象属性的类型进行处理
            switch (diff->obj_attr.diff.generic.type) {
            default:
                break;
            case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME:
            case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO:
                // 释放拓扑差异对象属性的字符串内存
                free(diff->obj_attr.diff.string.name);
                free(diff->obj_attr.diff.string.oldvalue);
                free(diff->obj_attr.diff.string.newvalue);
                break;
            }
            break;
        }
        // 释放拓扑差异对象内存
        free(diff);
        // 移动到下一个拓扑差异对象
        diff = next;
    }
    return 0;
}

/************************
 * 计算差异
 */

// 追加拓扑差异对象到链表中
static void hwloc_append_diff(hwloc_topology_diff_t newdiff,
                  hwloc_topology_diff_t *firstdiffp,
                  hwloc_topology_diff_t *lastdiffp)
{
    if (*firstdiffp)
        (*lastdiffp)->generic.next = newdiff;
    else
        *firstdiffp = newdiff;
    *lastdiffp = newdiff;
    newdiff->generic.next = NULL;
}

// 将拓扑对象的差异追加到链表中，如果太复杂则返回错误
static int hwloc_append_diff_too_complex(hwloc_obj_t obj1,
                     hwloc_topology_diff_t *firstdiffp,
                     hwloc_topology_diff_t *lastdiffp)
{
    hwloc_topology_diff_t newdiff;
    newdiff = malloc(sizeof(*newdiff));
    if (!newdiff)
        return -1;

    newdiff->too_complex.type = HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX;
    newdiff->too_complex.obj_depth = obj1->depth;
    newdiff->too_complex.obj_index = obj1->logical_index;
    // 追加拓扑差异对象到链表中
    hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
    return 0;
}
# 向拓扑结构差异对象中添加字符串类型的属性差异
static int hwloc_append_diff_obj_attr_string(hwloc_obj_t obj,
                         hwloc_topology_diff_obj_attr_type_t type,
                         const char *name,
                         const char *oldvalue,
                         const char *newvalue,
                         hwloc_topology_diff_t *firstdiffp,
                         hwloc_topology_diff_t *lastdiffp)
{
    # 分配新的拓扑结构差异对象
    hwloc_topology_diff_t newdiff;
    newdiff = malloc(sizeof(*newdiff));
    if (!newdiff)
        return -1;

    # 设置拓扑结构差异对象的属性
    newdiff->obj_attr.type = HWLOC_TOPOLOGY_DIFF_OBJ_ATTR;
    newdiff->obj_attr.obj_depth = obj->depth;
    newdiff->obj_attr.obj_index = obj->logical_index;
    newdiff->obj_attr.diff.string.type = type;
    newdiff->obj_attr.diff.string.name = name ? strdup(name) : NULL;
    newdiff->obj_attr.diff.string.oldvalue = oldvalue ? strdup(oldvalue) : NULL;
    newdiff->obj_attr.diff.string.newvalue = newvalue ? strdup(newvalue) : NULL;
    # 将新的拓扑结构差异对象添加到链表中
    hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
    return 0;
}

# 向拓扑结构差异对象中添加无符号64位整数类型的属性差异
static int hwloc_append_diff_obj_attr_uint64(hwloc_obj_t obj,
                         hwloc_topology_diff_obj_attr_type_t type,
                         hwloc_uint64_t idx,
                         hwloc_uint64_t oldvalue,
                         hwloc_uint64_t newvalue,
                         hwloc_topology_diff_t *firstdiffp,
                         hwloc_topology_diff_t *lastdiffp)
{
    # 分配新的拓扑结构差异对象
    hwloc_topology_diff_t newdiff;
    newdiff = malloc(sizeof(*newdiff));
    if (!newdiff)
        return -1;

    # 设置拓扑结构差异对象的属性
    newdiff->obj_attr.type = HWLOC_TOPOLOGY_DIFF_OBJ_ATTR;
    newdiff->obj_attr.obj_depth = obj->depth;
    newdiff->obj_attr.obj_index = obj->logical_index;
    newdiff->obj_attr.diff.uint64.type = type;
    newdiff->obj_attr.diff.uint64.index = idx;
    newdiff->obj_attr.diff.uint64.oldvalue = oldvalue;
    newdiff->obj_attr.diff.uint64.newvalue = newvalue;
    # 将新的拓扑结构差异对象添加到链表中
    hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
    return 0;
}

static int
# 比较两个给定的拓扑结构对象及其子对象之间的差异，并返回差异信息
def hwloc_diff_trees(hwloc_topology_t topo1, hwloc_obj_t obj1,
         hwloc_topology_t topo2, hwloc_obj_t obj2,
         unsigned flags,
         hwloc_topology_diff_t *firstdiffp, hwloc_topology_diff_t *lastdiffp)
{
    unsigned i;
    int err;
    hwloc_obj_t child1, child2;

    # 如果两个对象的深度不同，则跳转到标签 out_too_complex
    if (obj1->depth != obj2->depth)
        goto out_too_complex;

    # 如果两个对象的类型不同，则跳转到标签 out_too_complex
    if (obj1->type != obj2->type)
        goto out_too_complex;
    # 如果两个对象的子类型不同，则跳转到标签 out_too_complex
    if ((!obj1->subtype) != (!obj2->subtype)
        || (obj1->subtype && strcmp(obj1->subtype, obj2->subtype)))
        goto out_too_complex;

    # 如果两个对象的 os_index 不同，则跳转到标签 out_too_complex
    if (obj1->os_index != obj2->os_index)
        /* we could allow different os_index for non-PU non-NUMAnode objects
         * but it's likely useless anyway */
        goto out_too_complex;

    # 定义宏，用于比较两个集合是否不同
#define _SETS_DIFFERENT(_set1, _set2) \
 (   ( !(_set1) != !(_set2) ) \
  || ( (_set1) && !hwloc_bitmap_isequal(_set1, _set2) ) )
#define SETS_DIFFERENT(_set, _obj1, _obj2) _SETS_DIFFERENT((_obj1)->_set, (_obj2)->_set)
    # 如果两个对象的 cpuset、complete_cpuset、nodeset、complete_nodeset 中任意一个不同，则跳转到标签 out_too_complex
    if (SETS_DIFFERENT(cpuset, obj1, obj2)
        || SETS_DIFFERENT(complete_cpuset, obj1, obj2)
        || SETS_DIFFERENT(nodeset, obj1, obj2)
        || SETS_DIFFERENT(complete_nodeset, obj1, obj2))
        goto out_too_complex;

    # 不需要检查 logical_index、sibling_rank、symmetric_subtree，因为父对象已经检查过了

    # gp_index 不需要严格相同

    # 如果两个对象的名称不同，则将差异信息添加到 firstdiffp 和 lastdiffp 中
    if ((!obj1->name) != (!obj2->name)
        || (obj1->name && strcmp(obj1->name, obj2->name))) {
        err = hwloc_append_diff_obj_attr_string(obj1,
                               HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME,
                               NULL,
                               obj1->name,
                               obj2->name,
                               firstdiffp, lastdiffp);
        if (err < 0)
            return err;
    }

    # 根据对象的类型进行特定属性的处理
    switch (obj1->type) {
    default:
        break;
    # 如果对象类型是NUMANODE
    case HWLOC_OBJ_NUMANODE:
        # 如果两个对象的本地内存不相等
        if (obj1->attr->numanode.local_memory != obj2->attr->numanode.local_memory) {
            # 将本地内存大小差异添加到对象属性差异列表中
            err = hwloc_append_diff_obj_attr_uint64(obj1,
                                HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE,
                                0,
                                obj1->attr->numanode.local_memory,
                                obj2->attr->numanode.local_memory,
                                firstdiffp, lastdiffp);
            # 如果出现错误则返回错误码
            if (err < 0)
                return err;
        }
        # 忽略内存页类型
        /* ignore memory page_types */
        break;
    # 如果对象类型是缓存（L1/L2/L3/L4/L5 Cache，Instruction Cache）
    case HWLOC_OBJ_L1CACHE:
    case HWLOC_OBJ_L2CACHE:
    case HWLOC_OBJ_L3CACHE:
    case HWLOC_OBJ_L4CACHE:
    case HWLOC_OBJ_L5CACHE:
    case HWLOC_OBJ_L1ICACHE:
    case HWLOC_OBJ_L2ICACHE:
    case HWLOC_OBJ_L3ICACHE:
        # 如果两个对象的缓存属性不相等，则跳转到复杂处理
        if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->cache)))
            goto out_too_complex;
        break;
    # 如果对象类型是组
    case HWLOC_OBJ_GROUP:
        # 如果两个对象的组属性不相等，则跳转到复杂处理
        if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->group)))
            goto out_too_complex;
        break;
    # 如果对象类型是PCI设备
    case HWLOC_OBJ_PCI_DEVICE:
        # 如果两个对象的PCI设备属性不相等，则跳转到复杂处理
        if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->pcidev)))
            goto out_too_complex;
        break;
    # 如果对象类型是桥接设备
    case HWLOC_OBJ_BRIDGE:
        # 如果两个对象的桥接设备属性不相等，则跳转到复杂处理
        if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->bridge)))
            goto out_too_complex;
        break;
    # 如果对象类型是操作系统设备
    case HWLOC_OBJ_OS_DEVICE:
        # 如果两个对象的操作系统设备属性不相等，则跳转到复杂处理
        if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->osdev)))
            goto out_too_complex;
        break;
    }

    # 如果对象的信息数量不相等，则跳转到复杂处理
    /* infos */
    if (obj1->infos_count != obj2->infos_count)
        goto out_too_complex;
    # 遍历obj1的infos_count次
    for(i=0; i<obj1->infos_count; i++) {
        # 获取obj1和obj2的第i个info结构体
        struct hwloc_info_s *info1 = &obj1->infos[i], *info2 = &obj2->infos[i];
        # 比较info1和info2的name，如果不相同则跳转到out_too_complex标签
        if (strcmp(info1->name, info2->name))
            goto out_too_complex;
        # 比较info1和info2的value，如果不相同则执行以下操作
        if (strcmp(info1->value, info2->value)) {
            # 将info1和info2的差异添加到obj1的属性中
            err = hwloc_append_diff_obj_attr_string(obj1,
                                HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO,
                                info1->name,
                                info1->value,
                                info2->value,
                                firstdiffp, lastdiffp);
            # 如果出错则返回错误码
            if (err < 0)
                return err;
        }
    }

    # 忽略userdata

    # 遍历obj1和obj2的子节点，比较它们的拓扑结构
    for(child1 = obj1->first_child, child2 = obj2->first_child;
        child1 != NULL && child2 != NULL;
        child1 = child1->next_sibling, child2 = child2->next_sibling) {
        # 比较obj1和obj2的子节点的拓扑结构
        err = hwloc_diff_trees(topo1, child1,
                       topo2, child2,
                       flags,
                       firstdiffp, lastdiffp);
        # 如果出错则返回错误码
        if (err < 0)
            return err;
    }
    # 如果obj1或obj2有多余的子节点，则跳转到out_too_complex标签
    if (child1 || child2)
        goto out_too_complex;

    # 遍历obj1和obj2的内存子节点，比较它们的拓扑结构
    for(child1 = obj1->memory_first_child, child2 = obj2->memory_first_child;
        child1 != NULL && child2 != NULL;
        child1 = child1->next_sibling, child2 = child2->next_sibling) {
        # 比较obj1和obj2的内存子节点的拓扑结构
        err = hwloc_diff_trees(topo1, child1,
                       topo2, child2,
                       flags,
                       firstdiffp, lastdiffp);
        # 如果出错则返回错误码
        if (err < 0)
            return err;
    }
    # 如果obj1或obj2有多余的内存子节点，则跳转到out_too_complex标签
    if (child1 || child2)
        goto out_too_complex;

    # I/O children
    # 遍历两个对象的 I/O 子节点，比较它们的拓扑结构
    for(child1 = obj1->io_first_child, child2 = obj2->io_first_child;
        child1 != NULL && child2 != NULL;
        child1 = child1->next_sibling, child2 = child2->next_sibling) {
        # 对比两个子节点的拓扑结构差异
        err = hwloc_diff_trees(topo1, child1,
                       topo2, child2,
                       flags,
                       firstdiffp, lastdiffp);
        # 如果出现错误，直接返回错误码
        if (err < 0)
            return err;
    }
    # 如果其中一个子节点不为空，说明拓扑结构过于复杂，直接跳转到错误处理
    if (child1 || child2)
        goto out_too_complex;

    # 遍历两个对象的其他子节点
    for(child1 = obj1->misc_first_child, child2 = obj2->misc_first_child;
        child1 != NULL && child2 != NULL;
        child1 = child1->next_sibling, child2 = child2->next_sibling) {
        # 对比两个子节点的拓扑结构差异
        err = hwloc_diff_trees(topo1, child1,
                       topo2, child2,
                       flags,
                       firstdiffp, lastdiffp);
        # 如果出现错误，直接返回错误码
        if (err < 0)
            return err;
    }
    # 如果其中一个子节点不为空，说明拓扑结构过于复杂，直接跳转到错误处理
    if (child1 || child2)
        goto out_too_complex;

    # 如果以上都没有问题，返回 0 表示两个对象的拓扑结构相同
    return 0;
# 如果拓扑差异过于复杂，则将其添加到太复杂的差异列表中，并返回0
out_too_complex:
    hwloc_append_diff_too_complex(obj1, firstdiffp, lastdiffp);
    return 0;
}

# 构建拓扑差异
int hwloc_topology_diff_build(hwloc_topology_t topo1,
                  hwloc_topology_t topo2,
                  unsigned long flags,
                  hwloc_topology_diff_t *diffp)
{
    hwloc_topology_diff_t lastdiff, tmpdiff;  # 定义拓扑差异对象
    struct hwloc_internal_distances_s *dist1, *dist2;  # 定义内部距离对象
    unsigned i;  # 定义无符号整数i
    int err;  # 定义错误码

    # 如果拓扑1或拓扑2未加载，则返回错误
    if (!topo1->is_loaded || !topo2->is_loaded) {
      errno = EINVAL;
      return -1;
    }

    # 如果flags不为0，则返回错误
    if (flags != 0) {
        errno = EINVAL;
        return -1;
    }

    # 初始化差异指针
    *diffp = NULL;
    # 计算拓扑差异
    err = hwloc_diff_trees(topo1, hwloc_get_root_obj(topo1),
                   topo2, hwloc_get_root_obj(topo2),
                   flags,
                   diffp, &lastdiff);
    # 如果没有错误，则继续处理
    if (!err) {
        tmpdiff = *diffp;
        while (tmpdiff) {
            # 如果差异类型为太复杂，则设置错误码为1
            if (tmpdiff->generic.type == HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX) {
                err = 1;
                break;
            }
            tmpdiff = tmpdiff->generic.next;
        }
    }

    # 如果没有错误，则继续处理
    if (!err) {
        # 如果允许的cpuset或nodeset不同，则跳转到roottoocomplex标签处
        if (SETS_DIFFERENT(allowed_cpuset, topo1, topo2)
            || SETS_DIFFERENT(allowed_nodeset, topo1, topo2))
                  goto roottoocomplex;
    }
    if (!err) {
        /* 如果没有错误发生，则执行以下操作 */
        hwloc_internal_distances_refresh(topo1);
        // 刷新topo1中的距离信息
        hwloc_internal_distances_refresh(topo2);
        // 刷新topo2中的距离信息
        dist1 = topo1->first_dist;
        // 获取topo1中的第一个距离对象
        dist2 = topo2->first_dist;
        // 获取topo2中的第一个距离对象
        while (dist1 || dist2) {
            // 当dist1或dist2存在时执行循环
            if (!!dist1 != !!dist2)
                          goto roottoocomplex;
            // 如果dist1和dist2的存在性不一致，则跳转到roottoocomplex标签处
            if (dist1->unique_type != dist2->unique_type
                || dist1->different_types || dist2->different_types /* too lazy to support this case */
                || dist1->nbobjs != dist2->nbobjs
                || dist1->kind != dist2->kind
                || memcmp(dist1->values, dist2->values, dist1->nbobjs * dist1->nbobjs * sizeof(*dist1->values)))
                          goto roottoocomplex;
            // 如果距离对象的属性不一致，则跳转到roottoocomplex标签处
            for(i=0; i<dist1->nbobjs; i++)
                /* gp_index isn't enforced above. so compare logical_index instead, which is enforced. requires distances refresh() above */
                if (dist1->objs[i]->logical_index != dist2->objs[i]->logical_index)
                                  goto roottoocomplex;
                // 如果距离对象中的对象的逻辑索引不一致，则跳转到roottoocomplex标签处
            dist1 = dist1->next;
            // 获取下一个距离对象
            dist2 = dist2->next;
            // 获取下一个距离对象
        }
    return err;
    // 返回错误码

 roottoocomplex:
  hwloc_append_diff_too_complex(hwloc_get_root_obj(topo1), diffp, &lastdiff);
  // 将topo1的根对象添加到diffp中，表示太复杂
  return 1;
  // 返回错误码1
}

/********************
 * Applying diffs
 */

static int
hwloc_apply_diff_one(hwloc_topology_t topology,
             hwloc_topology_diff_t diff,
             unsigned long flags)
{
    int reverse = !!(flags & HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE);

    switch (diff->generic.type) {
    }
    default:
        return -1;
    }

    return 0;
}

int hwloc_topology_diff_apply(hwloc_topology_t topology,
                  hwloc_topology_diff_t diff,
                  unsigned long flags)
{
    hwloc_topology_diff_t tmpdiff, tmpdiff2;
    int err, nr;

    if (!topology->is_loaded) {
      errno = EINVAL;
      return -1;
    }
    if (topology->adopted_shmem_addr) {
      errno = EPERM;
      return -1;
    }

    if (flags & ~HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE) {
        errno = EINVAL;
        return -1;
    }

    tmpdiff = diff;
    nr = 0;
    while (tmpdiff) {
        nr++;
        err = hwloc_apply_diff_one(topology, tmpdiff, flags);
        if (err < 0)
            goto cancel;
        tmpdiff = tmpdiff->generic.next;
    }
    return 0;

cancel:
    tmpdiff2 = tmpdiff;
    tmpdiff = diff;
    while (tmpdiff != tmpdiff2) {
        hwloc_apply_diff_one(topology, tmpdiff, flags ^ HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE);
        tmpdiff = tmpdiff->generic.next;
    }
    errno = EINVAL;
    return -nr; /* return the index (starting at 1) of the first element that couldn't be applied */
}
```