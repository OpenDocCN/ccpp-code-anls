# `nmap\nsock\src\gh_list.h`

```cpp
/* $Id$ */

#ifndef GH_LIST_H
#define GH_LIST_H

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

#ifdef WIN32
#include "nbase_winconfig.h"
#endif

#include "error.h"
#include <assert.h>

#define GH_LIST_MAGIC       0xBADFACE
#ifndef GH_LIST_PARANOID
#define GH_LIST_PARANOID    0
#endif

// 定义链表节点结构
typedef struct gh_list_node {
  struct gh_list_node *next;  // 下一个节点指针
  struct gh_list_node *prev;  // 上一个节点指针
} gh_lnode_t;

// 定义链表结构
typedef struct gh_list {
  /* Number of elements in the list */
  unsigned int count;  // 链表中元素的数量
  gh_lnode_t *first;   // 链表中第一个节点的指针
  gh_lnode_t *last;    // 链表中最后一个节点的指针
} gh_list_t;

/* That one's an efficiency killer but it should reveal
 * any inconsistency in nsock's lists management. To be
 * called on every list we get and return. */
// 检查链表的一致性，用于调试目的
static inline void paranoid_list_check(gh_list_t *list) {
#if GH_LIST_PARANOID
  switch (list->count) {
    case 0:
      assert(list->first == NULL);
      assert(list->last == NULL);
      break;

    case 1:
      assert(list->first);
      assert(list->last);
      assert(list->first == list->last);
      break;

    default:
      assert(list->first);
      assert(list->last);
      assert(list->first != list->last);
      break;
  }
#endif
}

// 初始化链表
static inline int gh_list_init(gh_list_t *newlist) {
  newlist->count = 0;
  newlist->first = NULL;
  newlist->last  = NULL;
  return 0;
}

// 在链表末尾添加节点
static inline int gh_list_append(gh_list_t *list, gh_lnode_t *lnode) {
  gh_lnode_t *oldlast;

  paranoid_list_check(list);  // 检查链表一致性

  oldlast = list->last;
  if (oldlast)
    oldlast->next = lnode;

  lnode->prev = oldlast;
  lnode->next = NULL;

  list->count++;  // 链表元素数量加一
  list->last = lnode;  // 更新链表最后一个节点指针

  if (list->count == 1)
    list->first = lnode;  // 如果链表中只有一个节点，更新链表第一个节点指针

  paranoid_list_check(list);  // 再次检查链表一致性
  return 0;
}

// 在链表开头添加节点
static inline int gh_list_prepend(gh_list_t *list, gh_lnode_t *lnode) {
  gh_lnode_t *oldfirst;

  paranoid_list_check(list);  // 检查链表一致性

  oldfirst = list->first;
  if (oldfirst)
    # 将旧的第一个节点的前一个节点指向新节点
    oldfirst->prev = lnode;

  # 将新节点的下一个节点指向旧的第一个节点
  lnode->next = oldfirst;
  # 将新节点的前一个节点指向空
  lnode->prev = NULL;

  # 链表节点数量加一
  list->count++;
  # 将链表的第一个节点指向新节点
  list->first = lnode;

  # 如果链表中只有一个节点，则将最后一个节点指向新节点
  if (list->count == 1)
    list->last = lnode;

  # 检查链表是否符合预期的结构
  paranoid_list_check(list);
  # 返回 0 表示成功
  return 0;
# 在给定节点之前插入新节点到链表中
static inline int gh_list_insert_before(gh_list_t *list, gh_lnode_t *before,
                                        gh_lnode_t *lnode) {
  # 检查链表的完整性
  paranoid_list_check(list);

  # 设置新节点的前驱和后继节点
  lnode->prev = before->prev;
  lnode->next = before;

  # 更新前驱节点的后继指针
  if (before->prev)
    before->prev->next = lnode;
  else
    list->first = lnode;

  # 更新给定节点的前驱指针
  before->prev = lnode;
  # 增加链表节点计数
  list->count++;

  # 再次检查链表的完整性
  paranoid_list_check(list);
  return 0;
}

# 从链表中弹出第一个节点
static inline gh_lnode_t *gh_list_pop(gh_list_t *list) {
  gh_lnode_t *elem;

  # 检查链表的完整性
  paranoid_list_check(list);

  # 获取第一个节点
  elem = list->first;
  if (!elem) {
    # 如果链表为空，返回空指针
    paranoid_list_check(list);
    return NULL;
  }

  # 更新链表的第一个节点
  list->first = list->first->next;
  if (list->first)
    list->first->prev = NULL;

  # 减少链表节点计数
  list->count--;

  # 如果链表节点数小于2，更新链表的最后一个节点
  if (list->count < 2)
    list->last = list->first;

  # 重置节点的前驱和后继指针
  elem->prev = NULL;
  elem->next = NULL;

  # 再次检查链表的完整性
  paranoid_list_check(list);
  return elem;
}

# 从链表中移除指定节点
static inline int gh_list_remove(gh_list_t *list, gh_lnode_t *lnode) {
  # 检查链表的完整性
  paranoid_list_check(list);

  # 更新前驱节点的后继指针
  if (lnode->prev) {
    lnode->prev->next = lnode->next;
  } else {
    assert(list->first == lnode);
    list->first = lnode->next;
  }

  # 更新后继节点的前驱指针
  if (lnode->next) {
    lnode->next->prev = lnode->prev;
  } else {
    assert(list->last == lnode);
    list->last = lnode->prev;
  }

  # 重置节点的前驱和后继指针
  lnode->prev = NULL;
  lnode->next = NULL;

  # 减少链表节点计数
  list->count--;

  # 再次检查链表的完整性
  paranoid_list_check(list);
  return 0;
}

# 释放链表的所有节点
static inline int gh_list_free(gh_list_t *list) {
  # 检查链表的完整性
  paranoid_list_check(list);

  # 循环弹出链表的所有节点
  while (list->count > 0)
    gh_list_pop(list);

  # 再次检查链表的完整性
  paranoid_list_check(list);
  # 重置链表的内存内容
  memset(list, 0, sizeof(gh_list_t));
  return 0;
}

# 将指定节点移动到链表的最前面
static inline int gh_list_move_front(gh_list_t *list, gh_lnode_t *lnode) {
  # 检查链表的完整性
  paranoid_list_check(list);
  # 如果节点已经在链表的最前面，直接返回
  if (list->first == lnode)
    return 0;

  # 从当前位置移除节点
  lnode->prev->next = lnode->next;

  # 更新后继节点的前驱指针
  if (lnode->next) {
    lnode->next->prev = lnode->prev;
  } else {
    assert(list->last == lnode);
    # 将链表的最后一个元素指向新节点
    list->last = lnode->prev;
  }

  /* 将元素添加到链表的开头 */
  # 将链表的第一个元素的前驱指向新节点
  list->first->prev = lnode;
  # 将新节点的后继指向链表的第一个元素
  lnode->next = list->first;
  # 将新节点的前驱指向空
  lnode->prev = NULL;
  # 将链表的第一个元素指向新节点
  list->first = lnode;

  # 检查链表是否符合预期的结构
  paranoid_list_check(list);
  # 返回成功
  return 0;
/* 返回给定列表元素的下一个元素 */
static inline gh_lnode_t *gh_lnode_next(gh_lnode_t *elem) {
  return elem->next;
}

/* 返回给定列表元素的上一个元素 */
static inline gh_lnode_t *gh_lnode_prev(gh_lnode_t *elem) {
  return elem->prev;
}

/* 返回给定列表的第一个元素 */
static inline gh_lnode_t *gh_list_first_elem(gh_list_t *list) {
  return list->first;
}

/* 返回给定列表的最后一个元素 */
static inline gh_lnode_t *gh_list_last_elem(gh_list_t *list) {
  return list->last;
}

/* 返回给定列表中元素的数量 */
static inline unsigned int gh_list_count(gh_list_t *list) {
  return list->count;
}

#endif /* GH_LIST_H */
```