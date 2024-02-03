# `nmap\nsock\tests\ghlists.c`

```cpp
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"
/* Additional checks enabled */
#define GH_LIST_PARANOID 1
#include "../src/gh_list.h"
/* For container_of */
#include "../src/gh_heap.h"
#include <stdint.h>
#include <time.h>


#define LIST_COUNT  16
#define ELT_COUNT   2000


struct testlist {
  unsigned int val;
  gh_lnode_t lnode;
};

static unsigned int nodeval(gh_lnode_t *lnode) {
  struct testlist *tl;

  tl = container_of(lnode, struct testlist, lnode);
  return tl->val;
}

static gh_lnode_t *mknode(unsigned int val) {
  struct testlist *tl;

  tl = calloc(1, sizeof(struct testlist)); // 分配内存空间，大小为结构体 testlist 的大小
  tl->val = val; // 将传入的值赋给结构体的 val 成员
  return &tl->lnode; // 返回结构体中的 lnode 成员的地址
}

static void delnode(gh_lnode_t *lnode) {
  if (lnode)
    free(container_of(lnode, struct testlist, lnode)); // 释放结构体 testlist 的内存空间
}

static int ghlist_stress(void *tdata) {
  gh_list_t lists[LIST_COUNT]; // 创建长度为 LIST_COUNT 的 gh_list_t 数组
  gh_lnode_t *current, *next; // 定义指向 gh_lnode_t 结构体的指针
  int num = 0; // 初始化变量 num 为 0
  int ret; // 定义变量 ret
  int i; // 定义变量 i

  for (i = 0; i < LIST_COUNT; i++) // 循环初始化 gh_list_t 数组
    gh_list_init(&lists[i]); // 初始化 gh_list_t 数组中的每个元素

  for (num = ELT_COUNT/2; num < ELT_COUNT; num++) { // 循环从 ELT_COUNT/2 到 ELT_COUNT
    for (i = 0; i < LIST_COUNT; i++) { // 循环遍历 gh_list_t 数组
      gh_list_append(&lists[i], mknode(num)); // 将 mknode(num) 的返回值添加到 gh_list_t 数组中
    }
  }

  for (num = (ELT_COUNT/2 - 1); num >= 0; num--) { // 逆序循环从 ELT_COUNT/2 - 1 到 0
    for (i = 0; i < LIST_COUNT; i++) { // 循环遍历 gh_list_t 数组
      gh_list_prepend(&lists[i], mknode(num)); // 将 mknode(num) 的返回值添加到 gh_list_t 数组中
    }
  }

  for (num = 0; num < ELT_COUNT; num++) { // 循环从 0 到 ELT_COUNT
    for (i = 0; i < LIST_COUNT; i++) { // 循环遍历 gh_list_t 数组
      current = gh_list_pop(&lists[i]); // 弹出 gh_list_t 数组中的元素
      ret = nodeval(current); // 获取当前元素的值
      if (ret != num) { // 如果值不等于 num
    fprintf(stderr, "prepend_test: Bogus return value %d when expected %d\n",
                    ret, num); // 输出错误信息
    return -EINVAL; // 返回错误值
      }
      delnode(current); // 释放当前元素的内存空间
    }
  }
  for (i = 0; i < LIST_COUNT; i++) { // 循环遍历 gh_list_t 数组
    current = gh_list_pop(&lists[i]); // 弹出 gh_list_t 数组中的元素
    if (current) { // 如果当前元素存在
      fprintf(stderr, "Ret is bogus for list %d", i); // 输出错误信息
      return -EINVAL; // 返回错误值
    }
  }

  for (num = (ELT_COUNT/2 - 1); num >= 0; num--) { // 逆序循环从 ELT_COUNT/2 - 1 到 0
    for (i = 0; i < LIST_COUNT; i++) { // 循环遍历 gh_list_t 数组
      gh_list_prepend(&lists[i], mknode(num)); // 将 mknode(num) 的返回值添加到 gh_list_t 数组中
  }
}

for (num = ELT_COUNT/2; num < ELT_COUNT; num++) {
  for (i = 0; i < LIST_COUNT; i++) {
    // 将新节点添加到列表末尾
    gh_list_append(&lists[i], mknode(num));
  }
}

for (num = 0; num < ELT_COUNT; num++) {
  for (i=0; i < LIST_COUNT; i++) {
    // 从列表中弹出当前节点
    current = gh_list_pop(&lists[i]);
    // 获取节点值
    ret = nodeval(current);
    // 检查节点值是否符合预期
    if (ret != num) {
      fprintf(stderr, "prepend_test: Bogus return value %d when expected %d\n",
              ret, num);
      return -EINVAL;
    }
    // 删除当前节点
    delnode(current);
  }
}

for (num = ELT_COUNT/2; num < ELT_COUNT; num++) {
  for (i = 0; i < LIST_COUNT; i++)
    // 将新节点添加到列表末尾
    gh_list_append(&lists[i], mknode(num));
}

for (num = ELT_COUNT/2 - 1; num >= 0; num--) {
  for (i = 0; i < LIST_COUNT; i++)
    // 将新节点添加到列表开头
    gh_list_prepend(&lists[i], mknode(num));
}

for (num = 0; num < ELT_COUNT; num++) {
  for (i = 0; i < LIST_COUNT; i++) {
    // 从列表中弹出当前节点
    current = gh_list_pop(&lists[i]);
    // 获取节点值
    ret = nodeval(current);
    // 检查节点值是否符合预期
    if (ret != num) {
      fprintf(stderr, "prepend_test: Bogus return value %d when expected %d\n",
              ret, num);
      return -EINVAL;
    }
    // 删除当前节点
    delnode(current);
  }
}

for (num = ELT_COUNT/2 - 1; num >= 0; num--) {
  for (i = 0; i < LIST_COUNT; i++)
    // 将新节点添加到列表开头
    gh_list_prepend(&lists[i], mknode(num));
}

for (num = ELT_COUNT/2; num < ELT_COUNT; num++) {
  for (i=0; i < LIST_COUNT; i++)
    // 将新节点添加到列表末尾
    gh_list_append(&lists[i], mknode(num));
}

for (i = 0; i < LIST_COUNT; i++) {
  num = 0;

  for (current = gh_list_first_elem(&lists[i]); current; current = next) {
    int k;

    next = gh_lnode_next(current);
    k = nodeval(current);
    // 检查节点值是否符合预期
    if (k != num) {
      fprintf(stderr, "Got %d when I expected %d\n", k, num);
      return -EINVAL;
    }
    // 从列表中移除当前节点
    gh_list_remove(&lists[i], current);
    // 删除当前节点
    delnode(current);
    num++;
  }
  // 检查节点数量是否符合预期
  if (num != ELT_COUNT) {
    fprintf(stderr, "Number is %d, even though %d was expected", num, ELT_COUNT);
    return -EINVAL;
  }
}
    # 检查列表是否为空，如果不为空则输出错误信息并返回无效参数错误码
    if (gh_list_count(&lists[i]) != 0) {
      fprintf(stderr, "List should be empty, but instead it has %d members!\n",
              gh_list_count(&lists[i]));
      return -EINVAL;
    }
  }

  # 遍历列表数组，依次清空每个列表
  for (i = 0; i < LIST_COUNT; i++) {
    # 循环弹出列表中的节点，并删除节点
    while ((current = gh_list_pop(&lists[i])) != NULL)
      delnode(current);

    # 释放列表的内存空间
    gh_list_free(&lists[i]);
  }

  # 返回成功执行的标志
  return 0;
# 定义一个名为TestGHLists的测试用例结构体
const struct test_case TestGHLists = {
  # 设置测试用例的名称为"test nsock internal ghlists"
  .t_name     = "test nsock internal ghlists",
  # 设置测试用例的初始化函数为空
  .t_setup    = NULL,
  # 设置测试用例的运行函数为ghlist_stress
  .t_run      = ghlist_stress,
  # 设置测试用例的清理函数为空
  .t_teardown = NULL
};
```