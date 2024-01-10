# `nmap\nsock\src\filespace.c`

```
/* $Id$ */

#include "nsock_internal.h"
#include "filespace.h"

#include <string.h>

#define FS_INITSIZE_DEFAULT 1024


/* Assumes space for fs has already been allocated */
// 初始化文件空间结构体
int filespace_init(struct filespace *fs, int initial_size) {
  // 将文件空间结构体清零
  memset(fs, 0, sizeof(struct filespace));
  // 如果初始大小为0，则使用默认大小
  if (initial_size == 0)
    initial_size = FS_INITSIZE_DEFAULT;

  // 设置当前分配大小
  fs->current_alloc = initial_size;
  // 分配内存空间
  fs->str = (char *)safe_malloc(fs->current_alloc);
  // 设置字符串起始位置
  fs->str[0] = '\0';
  fs->pos = fs->str;
  return 0;
}

// 释放文件空间
int fs_free(struct filespace *fs) {
  // 如果文件空间字符串存在，则释放内存
  if (fs->str)
    free(fs->str);

  // 重置当前分配大小和当前大小
  fs->current_alloc = fs->current_size = 0;
  // 重置字符串起始位置和当前位置
  fs->pos = fs->str = NULL;
  return 0;
}

/* Concatenate a string to the end of a filespace */
// 将字符串连接到文件空间的末尾
int fs_cat(struct filespace *fs, const char *str, int len) {
  // 如果长度小于0，则返回错误
  if (len < 0)
    return -1;

  // 如果长度为0，则直接返回
  if (len == 0)
    return 0;

  // 如果当前分配大小不足以容纳新字符串
  if (fs->current_alloc - fs->current_size < len + 2) {
    char *tmpstr;

    // 计算新的分配大小
    fs->current_alloc = (int)(fs->current_alloc * 1.4 + 1);
    fs->current_alloc += 100 + len;

    // 分配新的内存空间
    tmpstr = (char *)safe_malloc(fs->current_alloc);
    // 将原字符串内容复制到新的内存空间
    memcpy(tmpstr, fs->str, fs->current_size);

    // 更新当前位置指针
    fs->pos = (fs->pos - fs->str) + tmpstr;

    // 释放原字符串内存空间
    if (fs->str)
      free(fs->str);

    // 更新文件空间字符串指针
    fs->str = tmpstr;
  }
  // 将新字符串内容拼接到文件空间字符串末尾
  memcpy(fs->str + fs->current_size, str, len);

  // 更新当前大小
  fs->current_size += len;
  // 添加字符串结束符
  fs->str[fs->current_size] = '\0';
  return 0;
}
```