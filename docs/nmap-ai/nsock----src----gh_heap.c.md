# `nmap\nsock\src\gh_heap.c`

```cpp
/* $Id$ */

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

#ifdef WIN32
#include "nbase_winconfig.h"
#endif

#include <nbase.h>
#include "gh_heap.h"

#define GH_SLOTS   128


static gh_hnode_t **hnode_ptr(gh_heap_t *heap, unsigned int index) {
  // 确保索引不超出堆的范围
  assert(index <= heap->count);
  // 获取指向指定索引位置的节点指针
  gh_hnode_t **ptr = &(heap->slots[index]);
  // 确保索引位置正确或者是堆的最后一个节点
  assert(index == heap->count || (*ptr)->index == index);
  return ptr;
}

gh_hnode_t *gh_heap_find(gh_heap_t *heap, unsigned int index) {
  // 如果索引超出堆的范围，则返回空指针
  if (index >= heap->count)
    return NULL;

  return *hnode_ptr(heap, index);
}

static int hnode_up(gh_heap_t *heap, gh_hnode_t *hnode)
{
  unsigned int cur_idx = hnode->index;
  gh_hnode_t **cur_ptr = hnode_ptr(heap, cur_idx);
  unsigned int parent_idx;
  gh_hnode_t **parent_ptr;
  int action = 0;

  // 确保当前节点指针指向正确的节点
  assert(*cur_ptr == hnode);

  // 向上调整节点位置，使其满足堆的性质
  while (cur_idx > 0) {
    parent_idx = (cur_idx - 1) >> 1;

    parent_ptr = hnode_ptr(heap, parent_idx);

    if (heap->cmp_op(*parent_ptr, hnode))
      break;

    (*parent_ptr)->index = cur_idx;
    *cur_ptr = *parent_ptr;
    cur_ptr = parent_ptr;
    cur_idx = parent_idx;
    action = 1;
  }

  hnode->index = cur_idx;
  *cur_ptr = hnode;

  return action;
}

static int hnode_down(gh_heap_t *heap, gh_hnode_t *hnode)
{
  unsigned int count = heap->count;
  unsigned int ch1_idx, ch2_idx, cur_idx;
  gh_hnode_t **ch1_ptr, **ch2_ptr, **cur_ptr;
  gh_hnode_t  *ch1, *ch2;
  int action = 0;

  cur_idx = hnode->index;
  cur_ptr = hnode_ptr(heap, cur_idx);
  // 确保当前节点指针指向正确的节点
  assert(*cur_ptr == hnode);

  // 向下调整节点位置，使其满足堆的性质
  while (cur_idx < count) {
    ch1_idx = (cur_idx << 1) + 1;
    if (ch1_idx >= count)
      break;

    ch1_ptr = hnode_ptr(heap, ch1_idx);
    ch1 = *ch1_ptr;

    ch2_idx = ch1_idx + 1;
    if (ch2_idx < count) {
      ch2_ptr = hnode_ptr(heap, ch2_idx);
      ch2 = *ch2_ptr;

      if (heap->cmp_op(ch2, ch1)) {
        ch1_idx = ch2_idx;
        ch1_ptr = ch2_ptr;
        ch1 = ch2;
      }
    }

    // 确保子节点指针指向正确的节点
    assert(ch1->index == ch1_idx);
    # 如果堆的比较操作返回 true，跳出循环
    if (heap->cmp_op(hnode, ch1))
      break;

    # 将当前节点的索引设置为 cur_idx
    ch1->index = cur_idx;
    # 将当前指针指向 ch1
    *cur_ptr = ch1;
    # 更新当前指针为 ch1_ptr
    cur_ptr = ch1_ptr;
    # 更新当前索引为 ch1_idx
    cur_idx = ch1_idx;
    # 设置动作为 1
    action = 1;
  }

  # 将堆节点的索引设置为 cur_idx
  hnode->index = cur_idx;
  # 将当前指针指向堆节点
  *cur_ptr = hnode;

  # 返回动作值
  return action;
# 堆增长函数，用于增加堆的容量
static int heap_grow(gh_heap_t *heap) {
  int newsize;

  # 检查是否真的需要增长
  assert(heap->count == heap->highwm);

  # 计算新的大小
  newsize = heap->count + GH_SLOTS;
  # 重新分配堆的空间
  heap->slots = (gh_hnode_t **)safe_realloc(heap->slots,
                                            newsize * sizeof(gh_hnode_t *));
  # 更新高水位
  heap->highwm += GH_SLOTS;
  # 将新增的空间初始化为0
  memset(heap->slots + heap->count, 0, GH_SLOTS * sizeof(gh_hnode_t *));
  return 0;
}

# 初始化堆
int gh_heap_init(gh_heap_t *heap, gh_heap_cmp_t cmp_op) {
  int rc;

  # 如果比较操作为空，则返回错误
  if (!cmp_op)
      return -1;

  # 设置比较操作
  heap->cmp_op = cmp_op;
  # 初始化计数、高水位和空间
  heap->count  = 0;
  heap->highwm = 0;
  heap->slots  = NULL;

  # 调用堆增长函数
  rc = heap_grow(heap);
  # 如果增长失败，则释放堆
  if (rc)
    gh_heap_free(heap);

  return rc;
}

# 释放堆
void gh_heap_free(gh_heap_t *heap) {
  # 如果高水位不为0，则释放空间
  if (heap->highwm) {
    assert(heap->slots);
    free(heap->slots);
  }
  # 将堆的内容清零
  memset(heap, 0, sizeof(gh_heap_t));
}

# 向堆中添加节点
int gh_heap_push(gh_heap_t *heap, gh_hnode_t *hnode) {
  gh_hnode_t **new_ptr;
  unsigned int new_index = heap->count;

  # 检查节点是否有效
  assert(!gh_hnode_is_valid(hnode));

  # 如果索引等于高水位，则增长堆
  if (new_index == heap->highwm)
    heap_grow(heap);

  # 设置节点的索引
  hnode->index = new_index;
  new_ptr = hnode_ptr(heap, new_index);
  assert(*new_ptr == NULL);
  heap->count++;
  *new_ptr = hnode;

  # 调整堆中节点的位置
  hnode_up(heap, hnode);
  return 0;
}

# 从堆中移除节点
int gh_heap_remove(gh_heap_t *heap, gh_hnode_t *hnode)
{
  unsigned int count = heap->count;
  unsigned int cur_idx = hnode->index;
  gh_hnode_t **cur_ptr;
  gh_hnode_t *last;

  # 检查节点是否有效
  assert(gh_hnode_is_valid(hnode));
  assert(cur_idx < count);

  cur_ptr = hnode_ptr(heap, cur_idx);
  assert(*cur_ptr == hnode);

  count--;
  last = *hnode_ptr(heap, count);
  heap->count = count;
  if (last != hnode)
  {
    last->index = cur_idx;
    *cur_ptr = last;
    if (!hnode_up(heap, *cur_ptr))
      hnode_down(heap, *cur_ptr);
  }

  # 使节点无效
  gh_hnode_invalidate(hnode);
  cur_ptr = hnode_ptr(heap, count);
  *cur_ptr = NULL;
  return 0;
}
```