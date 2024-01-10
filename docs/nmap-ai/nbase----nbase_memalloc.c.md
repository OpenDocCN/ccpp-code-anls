# `nmap\nbase\nbase_memalloc.c`

```
/* $Id$ */
// 包含头文件 nbase.h 和标准输入输出头文件
#include "nbase.h"
#include <stdio.h>

// 声明一个静态函数，用于输出错误信息并退出程序
NORETURN static void fatal(char *fmt, ...)
  __attribute__ ((format (printf, 1, 2)));

// 实现静态函数 fatal
static void fatal(char *fmt, ...) {
  va_list ap; // 声明一个可变参数列表

  va_start(ap, fmt); // 初始化可变参数列表
  fflush(stdout); // 刷新标准输出流
  vfprintf(stderr, fmt, ap); // 将格式化输出写入标准错误流
  fprintf(stderr, "\nQUITTING!\n"); // 输出退出信息
  va_end(ap); // 结束可变参数列表
  exit(1); // 退出程序
}

// 实现安全分配内存的函数
void *safe_malloc(size_t size) {
  void *mymem; // 声明一个指向内存的指针

  if ((int)size < 0)            /* Catch caller errors */ // 检查调用者传入的参数是否为负数
    fatal("Tried to malloc negative amount of memory!!!"); // 如果是负数，输出错误信息并退出程序
  mymem = malloc(size); // 分配指定大小的内存
  if (mymem == NULL) // 检查内存是否分配成功
    fatal("Malloc Failed! Probably out of space."); // 如果分配失败，输出错误信息并退出程序
  return mymem; // 返回分配的内存指针
}

// 实现安全重新分配内存的函数
void *safe_realloc(void *ptr, size_t size) {
  void *mymem; // 声明一个指向内存的指针

  if ((int)size < 0)            /* Catch caller errors */ // 检查调用者传入的参数是否为负数
    fatal("Tried to realloc negative amount of memory!!!"); // 如果是负数，输出错误信息并退出程序
  mymem = realloc(ptr, size); // 重新分配指定大小的内存
  if (mymem == NULL) // 检查内存是否重新分配成功
    fatal("Realloc Failed! Probably out of space."); // 如果重新分配失败，输出错误信息并退出程序
  return mymem; // 返回重新分配的内存指针
}

// 实现安全分配并初始化为零的内存的函数
/* Zero-initializing version of safe_malloc */
void *safe_zalloc(size_t size) {
  void *mymem; // 声明一个指向内存的指针

  if ((int)size < 0) // 检查调用者传入的参数是否为负数
    fatal("Tried to malloc negative amount of memory!!!"); // 如果是负数，输出错误信息并退出程序
  mymem = calloc(1, size); // 分配指定大小的内存并初始化为零
  if (mymem == NULL) // 检查内存是否分配成功
    fatal("Malloc Failed! Probably out of space."); // 如果分配失败，输出错误信息并退出程序
  return mymem; // 返回分配的内存指针
}
```