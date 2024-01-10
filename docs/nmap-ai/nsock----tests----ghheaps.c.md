# `nmap\nsock\tests\ghheaps.c`

```
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"
#include "../src/gh_heap.h"
#include <stdint.h>
#include <time.h>


#define HEAP_COUNT  3

struct testitem {
  int val;
  gh_hnode_t  node;
};

static int hnode_int_cmp(gh_hnode_t *n1, gh_hnode_t *n2) {
  // 定义比较函数，用于比较两个节点的值大小
  struct testitem *a;
  struct testitem *b;

  a = container_of(n1, struct testitem, node);
  b = container_of(n2, struct testitem, node);

  return (a->val < b->val);
}

static gh_hnode_t *mknode(int val) {
  // 创建一个新的节点，并初始化其值
  struct testitem *item;

  item = calloc(1, sizeof(struct testitem));
  assert(item != NULL);
  item->val = val;
  gh_hnode_invalidate(&item->node);
  return &item->node;
}

static int node2int(gh_hnode_t *hnode) {
  // 将节点转换为整数值
  struct testitem *item;

  item = container_of(hnode, struct testitem, node);
  return item->val;
}

static int ghheap_ordering(void *tdata) {
  // 对堆进行排序测试
  gh_heap_t heap;
  int i, n, k;

  gh_heap_init(&heap, hnode_int_cmp);

  for (i = 25000; i < 50000; i++)
    gh_heap_push(&heap, mknode(i));

  for (i = 24999; i >= 0; i--)
    gh_heap_push(&heap, mknode(i));

  for (i = 25000; i < 50000; i++)
    gh_heap_push(&heap, mknode(i));

  n = -1;
  do {
    gh_hnode_t *current;

    current = gh_heap_pop(&heap);
    assert(!gh_hnode_is_valid(current));
    k = node2int(current);

    if (k < n)
      return -EINVAL;

    n = k;
    free(container_of(current, struct testitem, node));
  } while (gh_heap_count(&heap) > 0);

  gh_heap_free(&heap);
  return 0;
}

static int ghheap_stress(void *tdata) {
  // 对堆进行压力测试
  gh_heap_t heaps[HEAP_COUNT];
  int i, num;

  for (i = 0; i < HEAP_COUNT; i++)
    gh_heap_init(&heaps[i], hnode_int_cmp);

  for (num = 25000; num < 50000; num++) {
    for (i = 0; i < HEAP_COUNT; i++) {
      gh_heap_push(&heaps[i], mknode(num));
    }
  }

  for (num = 24999; num >= 0; num--) {
    for (i = 0; i < HEAP_COUNT; i++) {
      gh_heap_push(&heaps[i], mknode(num));
    }
  }

  for (num = 0; num < 50000; num++) {
    // 进行一些操作
  }
}
    # 遍历堆数组，从0到HEAP_COUNT-1
    for (i = 0; i < HEAP_COUNT; i++) {
      int r_min, r_pop;
      gh_hnode_t *hnode;

      # 获取堆中最小值并转换为整数
      r_min = node2int(gh_heap_min(&heaps[i]));
      # 从堆中弹出最小节点
      hnode = gh_heap_pop(&heaps[i]);
      # 将弹出的节点转换为整数
      r_pop = node2int(hnode);

      # 检查最小值和弹出值是否相等，如果不相等则输出错误信息并返回错误码
      if (r_min != r_pop) {
        fprintf(stderr, "Bogus min/pop return values (%d != %d)\n", r_min, r_pop);
        return -EINVAL;
      }

      # 检查最小值是否等于num，如果不相等则输出错误信息并返回错误码
      if (r_min != num) {
        fprintf(stderr, "Bogus return value %d when expected %d\n", r_min, num);
        return -EINVAL;
      }

      # 释放弹出的节点所在的内存空间
      free(container_of(hnode, struct testitem, node));
    }
  }

  # 遍历堆数组，从0到HEAP_COUNT-1
  for (i = 0; i < HEAP_COUNT; i++) {
    void *ret;

    # 从堆中弹出节点，并将结果赋给ret
    ret = gh_heap_pop(&heaps[i]);
    # 如果ret不为空，则输出错误信息并返回错误码
    if (ret != NULL) {
      fprintf(stderr, "Ret is bogus for heap %d\n", i);
      return -EINVAL;
    }
  }

  # 遍历堆数组，从0到HEAP_COUNT-1，释放堆所占用的内存空间
  for (i = 0; i < HEAP_COUNT; i++)
    gh_heap_free(&heaps[i]);

  # 返回成功
  return 0;
# 定义名为TestGHHeaps的测试用例结构体
const struct test_case TestGHHeaps = {
  # 测试用例的名称为"test nsock internal ghheaps"
  .t_name     = "test nsock internal ghheaps",
  # 设置测试用例的初始化函数为NULL
  .t_setup    = NULL,
  # 设置测试用例的运行函数为ghheap_stress
  .t_run      = ghheap_stress,
  # 设置测试用例的清理函数为NULL
  .t_teardown = NULL
};

# 定义名为TestHeapOrdering的测试用例结构体
const struct test_case TestHeapOrdering = {
  # 测试用例的名称为"test heaps conditions"
  .t_name     = "test heaps conditions",
  # 设置测试用例的初始化函数为NULL
  .t_setup    = NULL,
  # 设置测试用例的运行函数为ghheap_ordering
  .t_run      = ghheap_ordering,
  # 设置测试用例的清理函数为NULL
  .t_teardown = NULL
};
```