# `nmap\nbase\nbase_str.c`

```cpp
/* $Id$ */
// 定义一个标识符

#include "nbase.h"
// 包含自定义的头文件 nbase.h
#include <assert.h>
// 包含断言处理的头文件
#include <string.h>
// 包含字符串处理的头文件
#include <stdarg.h>
// 包含可变参数列表处理的头文件
#include <stdio.h>
// 包含标准输入输出的头文件

#ifndef HAVE_STRCASESTR
// 如果没有定义 HAVE_STRCASESTR
char *strcasestr(const char *haystack, const char *pneedle) {
  // 定义 strcasestr 函数，用于在 haystack 中查找 pneedle
  char buf[512];
  // 定义一个缓冲区
  unsigned int needlelen;
  // 定义一个无符号整数变量 needlelen
  const char *p;
  // 定义一个指向常量字符的指针 p
  char *needle, *q, *foundto;
  // 定义字符指针 needle, q, foundto

  /* Should crash if !pneedle -- this is OK */
  // 如果 !pneedle 则应该崩溃 -- 这是可以接受的
  if (!*pneedle)
    return (char *)haystack;
  // 如果 pneedle 为空，则返回 haystack
  if (!haystack)
    return NULL;
  // 如果 haystack 为空，则返回空指针

  needlelen = (unsigned int)strlen(pneedle);
  // 计算 pneedle 的长度
  if (needlelen >= sizeof(buf))
    needle = (char *)safe_malloc(needlelen + 1);
  // 如果 pneedle 的长度大于等于缓冲区的大小，则分配内存
  else
    needle = buf;
  // 否则使用缓冲区

  p = pneedle;
  // 将 p 指向 pneedle
  q = needle;
  // 将 q 指向 needle

  while ((*q++ = tolower((int)(unsigned char)*p++)))
    ;
  // 将 pneedle 转换为小写存储到 needle 中

  p = haystack - 1;
  // 将 p 指向 haystack 减 1
  foundto = needle;
  // 将 foundto 指向 needle
  while (*++p) {
    // 循环遍历 haystack
    if (tolower((int)(unsigned char)*p) == *foundto) {
      // 如果 haystack 中的字符等于 foundto 指向的字符
      if (!*++foundto) {
        // 如果 foundto 指向的字符为空
        /* Yeah, we found it */
        // 找到了
        if (needlelen >= sizeof(buf))
          free(needle);
        // 如果分配了内存，则释放内存
        return (char *)(p - needlelen + 1);
        // 返回找到的位置
      }
    } else {
      p -= foundto - needle;
      // 如果不相等，则将 p 移动到 foundto - needle 的位置
      foundto = needle;
      // 将 foundto 指向 needle
    }
  }
  if (needlelen >= sizeof(buf))
    free(needle);
  // 如果分配了内存，则释放内存
  return NULL;
  // 返回空指针
}
#endif

int Strncpy(char *dest, const char *src, size_t n) {
  // 定义 Strncpy 函数，用于将 src 复制到 dest 中
  strncpy(dest, src, n);
  // 调用 strncpy 函数
  if (dest[n - 1] == '\0')
    return 0;
  // 如果 dest 的最后一个字符是空字符，则返回 0
  dest[n - 1] = '\0';
  // 将 dest 的最后一个字符设置为空字符
  return -1;
  // 返回 -1
}

int Vsnprintf(char *s, size_t n, const char *fmt, va_list ap) {
  // 定义 Vsnprintf 函数，用于格式化输出字符串
  int ret;

  ret = vsnprintf(s, n, fmt, ap);
  // 调用 vsnprintf 函数

  if (ret < 0 || (unsigned)ret >= n)
    s[n - 1] = '\0'; /* technically redundant */
  // 如果 ret 小于 0 或者 ret 大于等于 n，则将 s 的倒数第二个字符设置为空字符（在技术上是多余的）

  return ret;
  // 返回 ret
}

int Snprintf(char *s, size_t n, const char *fmt, ...) {
  // 定义 Snprintf 函数，用于格式化输出字符串
  va_list ap;
  // 定义可变参数列表 ap
  int ret;

  va_start(ap, fmt);
  // 初始化可变参数列表
  ret = Vsnprintf(s, n, fmt, ap);
  // 调用 Vsnprintf 函数
  va_end(ap);
  // 结束可变参数列表

  return ret;
  // 返回 ret
}

/* Make a new allocated null-terminated string from the bytes [start, end). */
// 从字节 [start, end) 中创建一个新的分配的以空字符结尾的字符串
char *mkstr(const char *start, const char *end) {
  // 定义 mkstr 函数
  char *s;

  assert(end >= start);
  // 断言 end 大于等于 start
  s = (char *)safe_malloc(end - start + 1);
  // 分配内存
  memcpy(s, start, end - start);
  // 将 start 到 end 的字节复制到 s 中
  s[end - start] = '\0';
  // 将 s 的最后一个字符设置为空字符

  return s;
  // 返回 s
}

/* Like strchr, but don't go past end. Nulls not handled specially. */
// 类似于 strchr，但不会超过 end。空字符不会被特殊处理
/* 在字符串中查找字符 c，返回指向该字符的指针，如果找不到则返回 NULL */
const char *strchr_p(const char *str, const char *end, char c) {
  const char *q=str;
  // 断言字符串指针不为空，并且结束指针大于等于起始指针
  assert(str && end >= str);
  // 遍历字符串，查找字符 c，找到则返回指向该字符的指针
  for (; q < end; q++) {
    if (*q == c)
      return q;
  }
  // 找不到则返回 NULL
  return NULL;
}

/* 动态分配缓冲区进行 vsprintf，类似于 Glibc 中的 asprintf。返回缓冲区的长度，出错时返回 -1 */
int alloc_vsprintf(char **strp, const char *fmt, va_list va) {
  va_list va_tmp;
  char *s;
  int size = 32;
  int n;

  s = NULL;
  size = 32;
  for (;;) {
    s = (char *)safe_realloc(s, size);

#ifdef WIN32
    va_tmp = va;
#else
    va_copy(va_tmp, va);
#endif
    n = vsnprintf(s, size, fmt, va_tmp);
    va_end(va_tmp);

    if (n >= size)
      size = n + 1;
    else if (n < 0)
      size = size * 2;
    else
      break;
  }
  *strp = s;

  return n;
}

/* 用于 escape_windows_command_arg，在给定位置向给定缓冲区追加一个字符，必要时调整缓冲区大小。调用后移动位置指针一个字节 */
static char* safe_append_char(char* buf, char byte, unsigned int *rpos, unsigned int *rsize)
{
    if (*rpos >= *rsize) {
        *rsize += 512;
        buf = (char*) safe_realloc(buf, *rsize);
    }
    buf[(*rpos)++] = byte;
    return buf;
}

/* 转义字符串，使其可以在命令行字符串中往返，并由默认的 C/C++ 命令行解析器检索。可以使用此函数转义字符串列表，用空格连接它们，传递给 CreateProcess，新进程将在其 argv 数组中获得相同的字符串列表。

   返回一个动态分配的字符串。

   此函数在 test/test-escape_windows_command_arg.c 中有一个测试程序。在进行任何更改后运行该程序。 */
char *escape_windows_command_arg(const char *arg)
{
    const char *p;
    # 声明一个指向字符的指针变量
    char *ret;
    # 初始化 rpos 为 0，rsize 为 1
    unsigned int rpos = 0, rsize = 1;

    # 为 ret 分配内存空间
    ret = (char *) safe_malloc(rsize);
    # 在 ret 中添加双引号，并更新 rpos 和 rsize
    ret = safe_append_char(ret, '"', &rpos, &rsize);

    # 遍历参数 arg
    for (p = arg; *p != '\0'; p++) {
        unsigned int num_backslashes;
        unsigned int i;

        # 统计连续的反斜杠数量
        num_backslashes = 0;
        for (; *p == '\\'; p++)
            num_backslashes++;

        # 如果当前字符是字符串结束符
        if (*p == '\0') {
            # 对所有反斜杠进行转义，但保留下面添加的双引号
            for (i = 0; i < num_backslashes*2; i++)
                ret = safe_append_char(ret, '\\', &rpos, &rsize);
            break;
        } else if (*p == '"') {
            # 对所有反斜杠和后面的双引号进行转义
            for (i = 0; i < num_backslashes*2 + 1; i++)
                ret = safe_append_char(ret, '\\', &rpos, &rsize);
            # 将当前字符添加到 ret 中
            ret[rpos++] = *p;
        } else {
            # 在这里反斜杠不是特殊字符
            for (i = 0; i < num_backslashes; i++)
                ret = safe_append_char(ret, '\\', &rpos, &rsize);
            # 将当前字符添加到 ret 中
            ret = safe_append_char(ret, *p, &rpos, &rsize);
        }
    }

    # 在 ret 中添加双引号，并更新 rpos 和 rsize
    ret = safe_append_char(ret, '"', &rpos, &rsize);
    # 在 ret 中添加字符串结束符，并更新 rpos 和 rsize
    ret = safe_append_char(ret, '\0', &rpos, &rsize);

    # 返回处理后的字符串
    return ret;
/* Convert non-printable characters to replchar in the string */
// 将字符串中的非可打印字符替换为指定的字符
void replacenonprintable(char *str, int strlength, char replchar) {
  int i;

  for (i = 0; i < strlength; i++)
    // 检查字符是否为可打印字符，如果不是则替换为指定字符
    if (!isprint((int)(unsigned char)str[i]))
      str[i] = replchar;

  return;
}

/* Returns the position of the last directory separator (slash, also backslash
   on Win32) in a path. Returns -1 if none was found. */
// 返回路径中最后一个目录分隔符（斜杠，在 Win32 上也包括反斜杠）的位置。如果没有找到则返回-1
static int find_last_path_separator(const char *path) {
#ifndef WIN32
  const char *PATH_SEPARATORS = "/";
#else
  const char *PATH_SEPARATORS = "\\/";
#endif
  const char *p;

  p = path + strlen(path) - 1;
  while (p >= path) {
    // 遍历路径，查找最后一个目录分隔符的位置
    if (strchr(PATH_SEPARATORS, *p) != NULL)
      return (int)(p - path);
    p--;
  }

  return -1;
}

/* Returns the directory name part of a path (everything up to the last
   directory separator). If there is no separator, returns ".". If there is only
   one separator and it is the first character, returns "/". Returns NULL on
   error. The returned string must be freed. */
// 返回路径的目录部分（直到最后一个目录分隔符为止）。如果没有分隔符，则返回“.”。如果只有一个分隔符且是第一个字符，则返回“/”。在出错时返回NULL。返回的字符串必须被释放
char *path_get_dirname(const char *path) {
  char *result;
  int i;

  // 获取路径中最后一个目录分隔符的位置
  i = find_last_path_separator(path);
  if (i == -1)
    return strdup(".");
  if (i == 0)
    return strdup("/");

  // 分配内存并拷贝目录部分的字符串
  result = (char *)safe_malloc(i + 1);
  strncpy(result, path, i);
  result[i] = '\0';

  return result;
}

/* Returns the file name part of a path (everything after the last directory
   separator). Returns NULL on error. The returned string must be freed. */
// 返回路径的文件名部分（最后一个目录分隔符之后的部分）。在出错时返回NULL。返回的字符串必须被释放
char *path_get_basename(const char *path) {
  int i;

  // 获取路径中最后一个目录分隔符的位置
  i = find_last_path_separator(path);

  return strdup(path + i + 1);
}
```