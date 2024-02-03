# `nmap\nping\utils.h`

```cpp
#ifndef UTILS_H
#define UTILS_H 1

#include "common.h"  // 包含 common.h 头文件

#include <stdlib.h>  // 包含标准库头文件

#include <stdarg.h>  // 包含可变参数列表头文件
#include <stdio.h>   // 包含标准输入输出头文件

#if HAVE_UNISTD_H  // 如果定义了 HAVE_UNISTD_H
#include <unistd.h>  // 包含 unistd.h 头文件
#endif

#include "global_structures.h"  // 包含 global_structures.h 头文件

/* Function prototypes */  // 函数原型声明
bool contains(const char *source, const char *substring);  // 包含函数原型声明
bool meansRandom(const char *source);  // 包含函数原型声明
bool isNumber_u8(const char *source, int base = 10);  // 包含函数原型声明
bool isNumber_u16(const char *source, int base = 10);  // 包含函数原型声明
bool isNumber_u32(const char *source, int base = 10);  // 包含函数原型声明
u8 *parseBufferSpec(char *str, size_t *outlen);  // 包含函数原型声明
int bitcmp(u8 *a, u8*b, int len);  // 包含函数原型声明
int removechar(char *string, char c);  // 包含函数原型声明
int removecolon(char *string);  // 包含函数原型声明
void luis_hdump(char *cp, unsigned int length);  // 包含函数原型声明
int validate_number_spec(const char *str);  // 包含函数原型声明
int parse_u8(const char *str, u8 *dstbuff);  // 包含函数原型声明
int parse_u16(const char *str, u16 *dstbuff);  // 包含函数原型声明
int parse_u32(const char *str, u32 *dstbuff);  // 包含函数原型声明
int print_hexdump(int level, const u8 *cp, u32 length);  // 包含函数原型声明

#endif /* UTILS_H */  // 结束条件编译指令
```