# `nmap\nsock\src\filespace.h`

```cpp
/* $Id$ */

// 如果 FILESPACE_H 未定义，则定义 FILESPACE_H
#ifndef FILESPACE_H
#define FILESPACE_H

// 如果 HAVE_CONFIG_H 定义了，则包含 nsock_config.h 和 nbase_config.h
#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

// 如果 WIN32 定义了，则包含 nbase_winconfig.h
#ifdef WIN32
#include "nbase_winconfig.h"
#endif

// 包含标准库头文件
#include <stdlib.h>
#include <stdarg.h>
// 如果定义了 HAVE_STRING_H，则包含 string.h
#if HAVE_STRING_H
#include <string.h>
#endif
// 如果定义了 HAVE_STRINGS_H，则包含 strings.h
#if HAVE_STRINGS_H
#include <strings.h>
#endif

// 定义文件空间结构体
struct filespace {
  int current_size; // 当前大小
  int current_alloc; // 当前分配

  /* Current position in the filespace */
  char *pos; // 文件空间中的当前位置
  char *str; // 文件空间中的字符串
};

// 内联函数，返回文件空间的长度
static inline int fs_length(const struct filespace *fs) {
  return fs->current_size;
}

// 内联函数，返回文件空间的字符串
static inline char *fs_str(const struct filespace *fs) {
  return fs->str;
}

// 初始化文件空间
int filespace_init(struct filespace *fs, int initial_size);

// 释放文件空间
int fs_free(struct filespace *fs);

// 将字符串追加到文件空间中
int fs_cat(struct filespace *fs, const char *str, int len);

#endif /* FILESPACE_H */
```