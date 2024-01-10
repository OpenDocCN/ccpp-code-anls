# `nmap\charpool.h`

```
/* $Id$ */

#ifndef CHARPOOL_H
#define CHARPOOL_H

#include <vector>

/* len does not include null terminator */
// 通过源字符串和长度创建一个新的字符串，不包括空终止符
const char *cp_strndup(const char *src, int len);
// 通过源字符串创建一个新的字符串
const char *cp_strdup(const char *src);
// 返回一个指向包含一个字符的字符串的指针
const char *cp_char2str(char c);

void cp_free(void);

typedef std::vector<char *> BucketList;

class CharPool {
  private:
    BucketList buckets;
    size_t currentbucketsz;
    size_t nexti;
  public:
    // 构造函数，初始化大小默认为256
    CharPool(size_t init_sz=256);
    // 析构函数，清空所有分配的内存
    ~CharPool() { this->clear(); }
    // 清空所有分配的内存
    void clear();
    // 如果长度小于0，将使用strlen确定源字符串的长度
    const char *dup(const char *src, int len=-1);
};

#endif
```