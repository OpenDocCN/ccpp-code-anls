# `xmrig\src\3rdparty\hwloc\src\bitmap.c`

```
/*
 * 版权声明
 * 2009年CNRS 版权所有
 * 2009-2020年Inria 版权所有
 * 2009-2011年Université Bordeaux 版权所有
 * 2009-2011年思科系统公司 版权所有
 * 请参阅顶层目录中的COPYING文件。
 */

#include "private/autogen/config.h" // 包含自动生成的配置文件
#include "hwloc/autogen/config.h" // 包含自动生成的配置文件
#include "hwloc.h" // 包含 hwloc 头文件
#include "private/misc.h" // 包含私有的杂项函数
#include "private/private.h" // 包含私有的函数
#include "private/debug.h" // 包含私有的调试函数
#include "hwloc/bitmap.h" // 包含位图操作相关的函数

#include <stdarg.h> // 包含可变参数列表相关的头文件
#include <stdio.h> // 包含标准输入输出相关的头文件
#include <assert.h> // 包含断言相关的头文件
#include <errno.h> // 包含错误码相关的头文件
#include <ctype.h> // 包含字符处理相关的头文件

/*
 * 可能的改进:
 * - 有一种方法可以改变初始分配大小:
 *   添加 hwloc_bitmap_set_foo() 来改变全局变量，
 *   并使 hwloc 核心根据早期处理器数量调用
 * - 使 HWLOC_BITMAP_PREALLOC_BITS 可配置，并在 Linux 上通过解析 /proc/cpuinfo 进行配置时进行检测。
 * - 在位图结构内部进行预分配（以便整个结构都是一个缓存行），并在重新分配更大的时候才分配专用数组。
 * - 添加一个 bitmap->ulongs_empty_first，保证一些最初的 ulongs 是空的，对于大位图来说，使测试速度更快，因为不需要查看最初的 ulongs。
 *   不需要 ulongs_empty_first 精确地是最大数量的空 ulongs，清除之前设置的位不太常见。
 */

/* 魔术数字 */
#define HWLOC_BITMAP_MAGIC 0x20091007

/* 每个位图的预分配位数 */
#define HWLOC_BITMAP_PREALLOC_BITS 512
#define HWLOC_BITMAP_PREALLOC_ULONGS (HWLOC_BITMAP_PREALLOC_BITS/HWLOC_BITS_PER_LONG)

/* 实际不透明类型的内部结构 */
struct hwloc_bitmap_s {
  unsigned ulongs_count; /* 有效的 ulong 位掩码数量，>= 1 */
  unsigned ulongs_allocated; /* 已分配的 ulong 位掩码数量，>= ulongs_count */
  unsigned long *ulongs; /* ulong 位掩码数组 */
  int infinite; /* 如果超出 ulongs 的所有位都设置，则设置为1 */
#ifdef HWLOC_DEBUG
  int magic; /* 魔术数 */
#endif
};
/* 在调试模式下进行过度检查，不如 valgrind 强大但仍然有用 */
#ifdef HWLOC_DEBUG
#define HWLOC__BITMAP_CHECK(set) do {                \
  assert((set)->magic == HWLOC_BITMAP_MAGIC);            \  // 确保位图对象的魔术数正确
  assert((set)->ulongs_count >= 1);                \  // 确保位图对象的 ulong 数量大于等于 1
  assert((set)->ulongs_allocated >= (set)->ulongs_count);    \  // 确保位图对象的分配的 ulong 数量大于等于实际的 ulong 数量
} while (0)
#else
#define HWLOC__BITMAP_CHECK(set)
#endif

/* 从位图中提取子集，使用索引或 CPU 编号 */
#define HWLOC_SUBBITMAP_INDEX(cpu)        ((cpu)/(HWLOC_BITS_PER_LONG))  // 根据 CPU 编号计算所在 ulong 数组的索引
#define HWLOC_SUBBITMAP_CPU_ULBIT(cpu)        ((cpu)%(HWLOC_BITS_PER_LONG))  // 根据 CPU 编号计算在 ulong 中的位偏移
/* 从位图 ulong 中读取，不知道 x 是否有效。
 * 写入者应确保 x 有效，并直接修改 set->ulongs[x]。
 */
#define HWLOC_SUBBITMAP_READULONG(set,x)    ((x) < (set)->ulongs_count ? (set)->ulongs[x] : (set)->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO)  // 从位图 ulong 中读取数据，如果 x 无效则返回全 1 或全 0

/* 预定义的子集数值 */
#define HWLOC_SUBBITMAP_ZERO            0UL  // 子集为全 0
#define HWLOC_SUBBITMAP_FULL            (~0UL)  // 子集为全 1
#define HWLOC_SUBBITMAP_ULBIT(bit)        (1UL<<(bit))  // 生成指定位的子集
#define HWLOC_SUBBITMAP_CPU(cpu)        HWLOC_SUBBITMAP_ULBIT(HWLOC_SUBBITMAP_CPU_ULBIT(cpu))  // 生成指定 CPU 编号的子集
#define HWLOC_SUBBITMAP_ULBIT_TO(bit)        (HWLOC_SUBBITMAP_FULL>>(HWLOC_BITS_PER_LONG-1-(bit)))  // 生成从 0 到指定位的子集
#define HWLOC_SUBBITMAP_ULBIT_FROM(bit)        (HWLOC_SUBBITMAP_FULL<<(bit))  // 生成从指定位到最高位的子集
#define HWLOC_SUBBITMAP_ULBIT_FROMTO(begin,end)    (HWLOC_SUBBITMAP_ULBIT_TO(end) & HWLOC_SUBBITMAP_ULBIT_FROM(begin))  // 生成从 begin 到 end 位的子集

struct hwloc_bitmap_s * hwloc_bitmap_alloc(void)
{
  struct hwloc_bitmap_s * set;

  set = malloc(sizeof(struct hwloc_bitmap_s));  // 分配位图对象内存
  if (!set)
    return NULL;

  set->ulongs_count = 1;  // 初始化 ulong 数量为 1
  set->ulongs_allocated = HWLOC_BITMAP_PREALLOC_ULONGS;  // 初始化分配的 ulong 数量
  set->ulongs = malloc(HWLOC_BITMAP_PREALLOC_ULONGS * sizeof(unsigned long));  // 分配 ulong 数组内存
  if (!set->ulongs) {
    free(set);
    return NULL;
  }

  set->ulongs[0] = HWLOC_SUBBITMAP_ZERO;  // 初始化第一个 ulong 为全 0
  set->infinite = 0;  // 初始化无限标志为 0
#ifdef HWLOC_DEBUG
  set->magic = HWLOC_BITMAP_MAGIC;  // 设置位图对象的魔术数
#endif
  return set;  // 返回初始化后的位图对象
}
# 分配一个包含所有位的位图
struct hwloc_bitmap_s * hwloc_bitmap_alloc_full(void)
{
  # 分配一个位图对象
  struct hwloc_bitmap_s * set = hwloc_bitmap_alloc();
  # 如果分配成功
  if (set) {
    # 设置位图对象的无限标志
    set->infinite = 1;
    # 将位图对象的第一个无符号长整型设置为 HWLOC_SUBBITMAP_FULL
    set->ulongs[0] = HWLOC_SUBBITMAP_FULL;
  }
  # 返回位图对象
  return set;
}

# 释放位图对象
void hwloc_bitmap_free(struct hwloc_bitmap_s * set)
{
  # 如果位图对象为空，则直接返回
  if (!set)
    return;

  # 检查位图对象的有效性
  HWLOC__BITMAP_CHECK(set);
  # 在调试模式下，将位图对象的魔术数设置为0
#ifdef HWLOC_DEBUG
  set->magic = 0;
#endif

  # 释放位图对象的无符号长整型数组
  free(set->ulongs);
  # 释放位图对象
  free(set);
}

# 根据需要的无符号长整型数量扩大位图对象
static int
hwloc_bitmap_enlarge_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_enlarge_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  # 计算需要的无符号长整型数量
  unsigned tmp = 1U << hwloc_flsl((unsigned long) needed_count - 1);
  # 如果需要的数量大于已分配的数量
  if (tmp > set->ulongs_allocated) {
    # 重新分配更大的无符号长整型数组
    unsigned long *tmpulongs;
    tmpulongs = realloc(set->ulongs, tmp * sizeof(unsigned long));
    # 如果分配失败，则返回-1
    if (!tmpulongs)
      return -1;
    # 更新位图对象的无符号长整型数组和已分配数量
    set->ulongs = tmpulongs;
    set->ulongs_allocated = tmp;
  }
  # 返回0表示成功
  return 0;
}

# 根据需要的无符号长整型数量重新分配位图对象，并根据无限标志更新新的无符号长整型数组
static int
hwloc_bitmap_realloc_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_realloc_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  # 定义变量
  unsigned i;

  # 检查位图对象的有效性
  HWLOC__BITMAP_CHECK(set);

  # 如果需要的数量小于等于已有的数量，则直接返回
  if (needed_count <= set->ulongs_count)
    return 0;

  # 根据需要的数量扩大位图对象
  if (hwloc_bitmap_enlarge_by_ulongs(set, needed_count) < 0)
    return -1;

  # 根据无限标志填充新分配的子集
  for(i=set->ulongs_count; i<needed_count; i++)
    set->ulongs[i] = set->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
  # 更新位图对象的无符号长整型数量
  set->ulongs_count = needed_count;
  # 返回0表示成功
  return 0;
}

# 根据需要的CPU索引数量重新分配位图对象
#define hwloc_bitmap_realloc_by_cpu_index(set, cpu) hwloc_bitmap_realloc_by_ulongs(set, ((cpu)/HWLOC_BITS_PER_LONG)+1)
/* 重置位图以确切需要的大小。
 * 调用者必须稍后重新初始化所有 ulongs 和无限标志。
 */
static int
hwloc_bitmap_reset_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_reset_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  if (hwloc_bitmap_enlarge_by_ulongs(set, needed_count))
    return -1;
  set->ulongs_count = needed_count;
  return 0;
}

/* 重置位图，直到它包含确切的 cpu+1 位（向上舍入到一个 ulong）。
 * 调用者必须稍后重新初始化所有 ulongs 和无限标志。
 */
#define hwloc_bitmap_reset_by_cpu_index(set, cpu) hwloc_bitmap_reset_by_ulongs(set, ((cpu)/HWLOC_BITS_PER_LONG)+1)

struct hwloc_bitmap_s * hwloc_bitmap_tma_dup(struct hwloc_tma *tma, const struct hwloc_bitmap_s * old)
{
  struct hwloc_bitmap_s * new;

  if (!old)
    return NULL;

  HWLOC__BITMAP_CHECK(old);

  new = hwloc_tma_malloc(tma, sizeof(struct hwloc_bitmap_s));
  if (!new)
    return NULL;

  new->ulongs = hwloc_tma_malloc(tma, old->ulongs_allocated * sizeof(unsigned long));
  if (!new->ulongs) {
    free(new);
    return NULL;
  }
  new->ulongs_allocated = old->ulongs_allocated;
  new->ulongs_count = old->ulongs_count;
  memcpy(new->ulongs, old->ulongs, new->ulongs_count * sizeof(unsigned long));
  new->infinite = old->infinite;
#ifdef HWLOC_DEBUG
  new->magic = HWLOC_BITMAP_MAGIC;
#endif
  return new;
}

struct hwloc_bitmap_s * hwloc_bitmap_dup(const struct hwloc_bitmap_s * old)
{
  return hwloc_bitmap_tma_dup(NULL, old);
}

int hwloc_bitmap_copy(struct hwloc_bitmap_s * dst, const struct hwloc_bitmap_s * src)
{
  HWLOC__BITMAP_CHECK(dst);
  HWLOC__BITMAP_CHECK(src);

  if (hwloc_bitmap_reset_by_ulongs(dst, src->ulongs_count) < 0)
    return -1;

  memcpy(dst->ulongs, src->ulongs, src->ulongs_count * sizeof(unsigned long));
  dst->infinite = src->infinite;
  return 0;
}

/* 字符串总是使用 32 位组 */
#define HWLOC_PRIxSUBBITMAP        "%08lx"
#define HWLOC_BITMAP_SUBSTRING_SIZE    32
// 定义宏，表示每个子字符串的大小为32位

#define HWLOC_BITMAP_SUBSTRING_LENGTH    (HWLOC_BITMAP_SUBSTRING_SIZE/4)
// 定义宏，表示每个子字符串的长度为32位除以4，即8个字符

#define HWLOC_BITMAP_STRING_PER_LONG    (HWLOC_BITS_PER_LONG/HWLOC_BITMAP_SUBSTRING_SIZE)
// 定义宏，表示每个长整型可以包含的子字符串的个数为64位除以32，即2个子字符串

int hwloc_bitmap_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  ssize_t size = buflen;
  char *tmp = buf;
  int res, ret = 0;
  int needcomma = 0;
  int i;
  unsigned long accum = 0;
  int accumed = 0;
#if HWLOC_BITS_PER_LONG == HWLOC_BITMAP_SUBSTRING_SIZE
  const unsigned long accum_mask = ~0UL;
#else /* HWLOC_BITS_PER_LONG != HWLOC_BITMAP_SUBSTRING_SIZE */
  const unsigned long accum_mask = ((1UL << HWLOC_BITMAP_SUBSTRING_SIZE) - 1) << (HWLOC_BITS_PER_LONG - HWLOC_BITMAP_SUBSTRING_SIZE);
#endif /* HWLOC_BITS_PER_LONG != HWLOC_BITMAP_SUBSTRING_SIZE */
// 根据不同情况定义累加掩码

  HWLOC__BITMAP_CHECK(set);
  // 检查位图是否有效

  /* mark the end in case we do nothing later */
  if (buflen > 0)
    tmp[0] = '\0';
  // 如果缓冲区大小大于0，则在缓冲区末尾标记结束符

  if (set->infinite) {
    res = hwloc_snprintf(tmp, size, "0xf...f");
    needcomma = 1;
    if (res < 0)
      return -1;
    ret += res;
    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;
    tmp += res;
    size -= res;
  }
  // 如果位图为无限，则将字符串"0xf...f"写入缓冲区

  i=(int) set->ulongs_count-1;

  if (set->infinite) {
    /* ignore starting FULL since we have 0xf...f already */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_FULL)
      i--;
  } else {
    /* ignore starting ZERO except the last one */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_ZERO)
      i--;
  }
  // 忽略起始的全满子字符串或零子字符串

  while (i>=0 || accumed) {
    /* Refill accumulator */
    if (!accumed) {
      accum = set->ulongs[i--];
      accumed = HWLOC_BITS_PER_LONG;
    }
    // 重新填充累加器

    if (accum & accum_mask) {
      /* print the whole subset if not empty */
        res = hwloc_snprintf(tmp, size, needcomma ? ",0x" HWLOC_PRIxSUBBITMAP : "0x" HWLOC_PRIxSUBBITMAP,
             (accum & accum_mask) >> (HWLOC_BITS_PER_LONG - HWLOC_BITMAP_SUBSTRING_SIZE));
      needcomma = 1;
      // 如果累加器不为空，则将子字符串打印到缓冲区
    } else if (i == -1 && accumed == HWLOC_BITMAP_SUBSTRING_SIZE) {
      /* 如果 i 等于 -1 并且 accumed 等于 HWLOC_BITMAP_SUBSTRING_SIZE，则打印一个单独的 0 来标记最后的子集 */
      res = hwloc_snprintf(tmp, size, needcomma ? ",0x0" : "0x0");
    } else if (needcomma) {
      /* 如果需要逗号，则打印逗号 */
      res = hwloc_snprintf(tmp, size, ",");
    } else {
      /* 否则，结果为 0 */
      res = 0;
    }
    /* 如果结果小于 0，则返回 -1 */
    if (res < 0)
      return -1;
    /* 结果累加到 ret 变量中 */
    ret += res;
#if HWLOC_BITS_PER_LONG == HWLOC_BITMAP_SUBSTRING_SIZE
    // 如果 HWLOC_BITS_PER_LONG 等于 HWLOC_BITMAP_SUBSTRING_SIZE，则初始化 accum 和 accumed 为 0
    accum = 0;
    accumed = 0;
#else
    // 如果不相等，则将 accum 左移 HWLOC_BITMAP_SUBSTRING_SIZE 位，同时减去 HWLOC_BITMAP_SUBSTRING_SIZE
    accum <<= HWLOC_BITMAP_SUBSTRING_SIZE;
    accumed -= HWLOC_BITMAP_SUBSTRING_SIZE;
#endif

    // 如果 res 大于等于 size，则将 res 设为 size-1 或 0
    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;

    // 将 tmp 加上 res，size 减去 res
    tmp += res;
    size -= res;
  }

  /* 如果没有显示任何内容，则显示 0x0 */
  if (!ret) {
    // 使用 hwloc_snprintf 将 "0x0" 格式化到 tmp 中
    res = hwloc_snprintf(tmp, size, "0x0");
    // 如果格式化失败，则返回 -1
    if (res < 0)
      return -1;
    // ret 加上 res
    ret += res;
  }

  // 返回 ret
  return ret;
}

// 将位图格式化为字符串
int hwloc_bitmap_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;
  char *buf;

  // 检查位图是否合法
  HWLOC__BITMAP_CHECK(set);

  // 获取格式化后的字符串长度
  len = hwloc_bitmap_snprintf(NULL, 0, set);
  // 分配内存
  buf = malloc(len+1);
  // 如果分配失败，则返回 -1
  if (!buf)
    return -1;
  // 将格式化后的字符串赋值给 strp
  *strp = buf;
  // 返回格式化后的字符串长度
  return hwloc_bitmap_snprintf(buf, len+1, set);
}

// 从字符串中读取位图
int hwloc_bitmap_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;
  unsigned long accum = 0;
  int count=0;
  int infinite = 0;

  /* 计算子字符串的数量 */
  count++;
  while ((current = strchr(current+1, ',')) != NULL)
    count++;

  current = string;
  if (!strncmp("0xf...f", current, 7)) {
    current += 7;
    if (*current != ',') {
      /* 无限/全位图的特殊情况 */
      // 将位图设置为全满
      hwloc_bitmap_fill(set);
      return 0;
    }
    current++;
    infinite = 1;
    count--;
  }

  // 重置位图
  if (hwloc_bitmap_reset_by_ulongs(set, (count + HWLOC_BITMAP_STRING_PER_LONG - 1) / HWLOC_BITMAP_STRING_PER_LONG) < 0)
    return -1;
  set->infinite = 0;

  while (*current != '\0') {
    unsigned long val;
    char *next;
    // 将当前字符串转换为无符号长整型
    val = strtoul(current, &next, 16);

    assert(count > 0);
    count--;

    // 将值存入 accum 中
    accum |= (val << ((count * HWLOC_BITMAP_SUBSTRING_SIZE) % HWLOC_BITS_PER_LONG));
    if (!(count % HWLOC_BITMAP_STRING_PER_LONG)) {
      set->ulongs[count / HWLOC_BITMAP_STRING_PER_LONG] = accum;
      accum = 0;
    }

    if (*next != ',') {
      if (*next || count > 0)
    goto failed;
      else
    break;
    }
    current = (const char*) next+1;  # 将指针 next 转换为 const char* 类型，并加1赋值给指针 current
  }

  set->infinite = infinite;  # 将变量 infinite 的值赋给结构体指针 set 的成员变量 infinite，放在最后以避免在填充新的 ulongs 时出现不必要的重新分配内存

  return 0;  # 返回成功的标志

 failed:  # 标签，表示失败的情况
  hwloc_bitmap_zero(set);  # 将结构体指针 set 指向的位图清零
  return -1;  # 返回失败的标志
}

int hwloc_bitmap_list_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int prev = -1;  // 初始化前一个位置为-1
  ssize_t size = buflen;  // 初始化 size 为 buflen
  char *tmp = buf;  // 初始化 tmp 为 buf
  int res, ret = 0;  // 初始化 res 和 ret

  HWLOC__BITMAP_CHECK(set);  // 检查位图是否有效

  /* mark the end in case we do nothing later */
  if (buflen > 0)
    tmp[0] = '\0';  // 如果 buflen 大于 0，则在 tmp[0] 处标记结束

  while (1) {
    int begin, end;  // 初始化 begin 和 end

    begin = hwloc_bitmap_next(set, prev);  // 获取下一个被设置的位的位置
    if (begin == -1)  // 如果没有找到被设置的位，则跳出循环
      break;
    end = hwloc_bitmap_next_unset(set, begin);  // 获取下一个未设置的位的位置

    if (end == begin+1) {  // 如果 end 与 begin 相差 1
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d" : "%d", begin);  // 格式化输出到 tmp 中
    } else if (end == -1) {  // 如果 end 为 -1
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d-" : "%d-", begin);  // 格式化输出到 tmp 中
    } else {
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d-%d" : "%d-%d", begin, end-1);  // 格式化输出到 tmp 中
    }
    if (res < 0)  // 如果格式化输出失败，则返回 -1
      return -1;
    ret += res;  // 累加 res 到 ret

    if (res >= size)  // 如果 res 大于等于 size
      res = size>0 ? (int)size - 1 : 0;  // 将 res 赋值为 size-1 或 0

    tmp += res;  // tmp 向后移动 res 个位置
    size -= res;  // size 减去 res
    needcomma = 1;  // 设置 needcomma 为 1

    if (end == -1)  // 如果 end 为 -1
      break;  // 跳出循环
    else
      prev = end - 1;  // 设置 prev 为 end-1
  }

  return ret;  // 返回 ret
}

int hwloc_bitmap_list_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;  // 初始化 len
  char *buf;  // 初始化 buf

  HWLOC__BITMAP_CHECK(set);  // 检查位图是否有效

  len = hwloc_bitmap_list_snprintf(NULL, 0, set);  // 获取格式化输出的长度
  buf = malloc(len+1);  // 分配内存
  if (!buf)  // 如果分配内存失败
    return -1;  // 返回 -1
  *strp = buf;  // 将 buf 赋值给 strp
  return hwloc_bitmap_list_snprintf(buf, len+1, set);  // 返回格式化输出的长度
}

int hwloc_bitmap_list_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;  // 初始化 current
  char *next;  // 初始化 next
  long begin = -1, val;  // 初始化 begin 和 val

  hwloc_bitmap_zero(set);  // 将位图清零

  while (*current != '\0') {  // 当 current 不为结束符时循环

    /* ignore empty ranges */
    while (*current == ',' || *current == ' ')  // 忽略空范围
      current++;  // current 向后移动

    val = strtoul(current, &next, 0);  // 将字符串转换为无符号长整型数
    /* make sure we got at least one digit */
    if (next == current)  // 如果没有获取到数字
      goto failed;  // 跳转到 failed 标签

    if (begin != -1) {  // 如果 begin 不为 -1
      /* finishing a range */
      if (hwloc_bitmap_set_range(set, begin, val) < 0)  // 设置位图的范围
        goto failed;  // 跳转到 failed 标签
      begin = -1;  // 将 begin 设置为 -1
    }
    } else if (*next == '-') {
      /* 如果下一个字符是减号，表示开始一个新的范围 */
      if (*(next+1) == '\0') {
    /* 如果下一个字符是空字符，表示无限范围 */
    if (hwloc_bitmap_set_range(set, val, -1) < 0)
      goto failed;
        break;
      } else {
    /* 否则表示正常范围 */
    begin = val;
      }

    } else if (*next == ',' || *next == ' ' || *next == '\0') {
      /* 如果下一个字符是逗号、空格或者空字符，表示单个数字 */
      hwloc_bitmap_set(set, val);
    }

    if (*next == '\0')
      break;
    current = next+1;
  }

  return 0;

 failed:
  /* 解析失败，将位图清零 */
  hwloc_bitmap_zero(set);
  return -1;
}
# 定义一个函数，将 hwloc_bitmap_s 结构体转换为字符串
int hwloc_bitmap_taskset_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  ssize_t size = buflen;  # 定义一个 ssize_t 类型的变量 size，用于存储 buflen 的值
  char *tmp = buf;  # 定义一个 char 类型的指针 tmp，指向 buf
  int res, ret = 0;  # 定义两个 int 类型的变量 res 和 ret，ret 初始化为 0
  int started = 0;  # 定义一个 int 类型的变量 started，初始化为 0
  int i;  # 定义一个 int 类型的变量 i

  HWLOC__BITMAP_CHECK(set);  # 检查 set 是否为有效的 hwloc_bitmap_s 结构体

  /* mark the end in case we do nothing later */
  if (buflen > 0)  # 如果 buflen 大于 0
    tmp[0] = '\0';  # 将 tmp 的第一个字符设为字符串结束符

  if (set->infinite) {  # 如果 set 的 infinite 属性为真
    res = hwloc_snprintf(tmp, size, "0xf...f");  # 使用 hwloc_snprintf 将 "0xf...f" 格式化输出到 tmp 中
    started = 1;  # 将 started 设为 1
    if (res < 0)  # 如果 res 小于 0
      return -1;  # 返回 -1
    ret += res;  # 将 res 加到 ret 上
    if (res >= size)  # 如果 res 大于等于 size
      res = size>0 ? (int)size - 1 : 0;  # 将 res 设为 size-1 或 0
    tmp += res;  # 将 tmp 向后移动 res 个位置
    size -= res;  # 将 size 减去 res
  }

  i=set->ulongs_count-1;  # 将 i 设为 set 的 ulongs_count 属性减 1

  if (set->infinite) {  # 如果 set 的 infinite 属性为真
    /* ignore starting FULL since we have 0xf...f already */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_FULL)  # 当 i 大于等于 0 且 set 的 ulongs[i] 等于 HWLOC_SUBBITMAP_FULL
      i--;  # 将 i 减 1
  } else {
    /* ignore starting ZERO except the last one */
    while (i>=1 && set->ulongs[i] == HWLOC_SUBBITMAP_ZERO)  # 当 i 大于等于 1 且 set 的 ulongs[i] 等于 HWLOC_SUBBITMAP_ZERO
      i--;  # 将 i 减 1
  }

  while (i>=0) {  # 当 i 大于等于 0
    unsigned long val = set->ulongs[i--];  # 定义一个 unsigned long 类型的变量 val，赋值为 set 的 ulongs[i]，然后将 i 减 1
    if (started) {  # 如果 started 为真
      /* print the whole subset */
#if HWLOC_BITS_PER_LONG == 64
      res = hwloc_snprintf(tmp, size, "%016lx", val);  # 使用 hwloc_snprintf 将 val 格式化输出到 tmp 中
#else
      res = hwloc_snprintf(tmp, size, "%08lx", val);  # 使用 hwloc_snprintf 将 val 格式化输出到 tmp 中
#endif
    } else if (val || i == -1) {  # 否则如果 val 不为 0 或者 i 为 -1
      res = hwloc_snprintf(tmp, size, "0x%lx", val);  # 使用 hwloc_snprintf 将 val 格式化输出到 tmp 中
      started = 1;  # 将 started 设为 1
    } else {
      res = 0;  # 否则将 res 设为 0
    }
    if (res < 0)  # 如果 res 小于 0
      return -1;  # 返回 -1
    ret += res;  # 将 res 加到 ret 上
    if (res >= size)  # 如果 res 大于等于 size
      res = size>0 ? (int)size - 1 : 0;  # 将 res 设为 size-1 或 0
    tmp += res;  # 将 tmp 向后移动 res 个位置
    size -= res;  # 将 size 减去 res
  }

  /* if didn't display anything, display 0x0 */
  if (!ret) {  # 如果 ret 为 0
    res = hwloc_snprintf(tmp, size, "0x0");  # 使用 hwloc_snprintf 将 "0x0" 格式化输出到 tmp 中
    if (res < 0)  # 如果 res 小于 0
      return -1;  # 返回 -1
    ret += res;  # 将 res 加到 ret 上
  }

  return ret;  # 返回 ret
}

# 定义一个函数，将 hwloc_bitmap_s 结构体转换为字符串，并分配内存
int hwloc_bitmap_taskset_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;  # 定义一个 int 类型的变量 len
  char *buf;  # 定义一个 char 类型的指针 buf

  HWLOC__BITMAP_CHECK(set);  # 检查 set 是否为有效的 hwloc_bitmap_s 结构体

  len = hwloc_bitmap_taskset_snprintf(NULL, 0, set);  # 调用 hwloc_bitmap_taskset_snprintf 函数，获取转换后字符串的长度
  buf = malloc(len+1);  # 分配 len+1 大小的内存空间给 buf
  if (!buf)  # 如果 buf 为空
    return -1;  # 返回 -1
  *strp = buf;  # 将 buf 的地址赋值给 strp
  return hwloc_bitmap_taskset_snprintf(buf, len+1, set);  # 调用 hwloc_bitmap_taskset_snprintf 函数，将转换后的字符串存储到 buf 中
}
# 从字符串中解析出一个 hwloc_bitmap_s 结构
int hwloc_bitmap_taskset_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;  # 定义一个指向字符串的指针
  int chars;  # 字符数
  int count;  # 计数
  int infinite = 0;  # 无限标志位，默认为0

  if (!strncmp("0xf...f", current, 7)) {  # 如果字符串以 "0xf...f" 开头
    /* infinite bitmap */
    infinite = 1;  # 设置无限标志位为1
    current += 7;  # 移动指针到第7个字符
    if (*current == '\0') {  # 如果当前字符为空
      /* special case for infinite/full bitmap */
      hwloc_bitmap_fill(set);  # 填充位图
      return 0;  # 返回0
    }
  } else {
    /* finite bitmap */
    if (!strncmp("0x", current, 2))  # 如果字符串以 "0x" 开头
      current += 2;  # 移动指针到第2个字符
    if (*current == '\0') {  # 如果当前字符为空
      /* special case for empty bitmap */
      hwloc_bitmap_zero(set);  # 清空位图
      return 0;  # 返回0
    }
  }
  /* we know there are other characters now */

  chars = (int)strlen(current);  # 获取当前字符数
  count = (chars * 4 + HWLOC_BITS_PER_LONG - 1) / HWLOC_BITS_PER_LONG;  # 计算需要的长整型数

  if (hwloc_bitmap_reset_by_ulongs(set, count) < 0)  # 重置位图
    return -1;  # 返回-1
  set->infinite = 0;  # 设置无限标志位为0

  while (*current != '\0') {  # 循环直到当前字符为空
    int tmpchars;  # 临时字符数
    char ustr[17];  # 临时字符串
    unsigned long val;  # 无符号长整型值
    char *next;  # 下一个字符

    tmpchars = chars % (HWLOC_BITS_PER_LONG/4);  # 计算临时字符数
    if (!tmpchars)  # 如果临时字符数为0
      tmpchars = (HWLOC_BITS_PER_LONG/4);  # 设置为4

    memcpy(ustr, current, tmpchars);  # 复制临时字符数个字符到临时字符串
    ustr[tmpchars] = '\0';  # 在临时字符串末尾添加空字符
    val = strtoul(ustr, &next, 16);  # 将临时字符串转换为无符号长整型值
    if (*next != '\0')  # 如果下一个字符不为空
      goto failed;  # 跳转到失败标签

    set->ulongs[count-1] = val;  # 设置 ulongs 数组的值

    current += tmpchars;  # 移动当前指针
    chars -= tmpchars;  # 减去已处理的字符数
    count--;  # 计数减一
  }

  set->infinite = infinite;  # 设置无限标志位为无限值

  return 0;  # 返回0

 failed:
  /* failure to parse */
  hwloc_bitmap_zero(set);  # 失败时清空位图
  return -1;  # 返回-1
}

static void hwloc_bitmap__zero(struct hwloc_bitmap_s *set)
{
    unsigned i;
    for(i=0; i<set->ulongs_count; i++)  # 遍历 ulongs 数组
        set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;  # 设置 ulongs 数组的值为 HWLOC_SUBBITMAP_ZERO
    set->infinite = 0;  # 设置无限标志位为0
}

void hwloc_bitmap_zero(struct hwloc_bitmap_s * set)
{
    HWLOC__BITMAP_CHECK(set);  # 检查位图

    HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);  # 构建断言，检查预分配的长整型数是否大于等于1
}
    # 如果重置位图失败
    if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
        # 由于预先分配了一些无符号长整型，所以不应该失败。
        # 如果我们曾经没有预分配任何东西，那么我们将重置为0个无符号长整型。
    }
    # 将位图设置为零
    hwloc_bitmap__zero(set);
# 填充给定的位图集合，使其包含所有的位
static void hwloc_bitmap__fill(struct hwloc_bitmap_s * set)
{
    # 初始化循环变量 i
    unsigned i;
    # 遍历位图集合的每个 ulong，将其设置为 HWLOC_SUBBITMAP_FULL
    for(i=0; i<set->ulongs_count; i++)
        set->ulongs[i] = HWLOC_SUBBITMAP_FULL;
    # 将位图集合标记为无限
    set->infinite = 1;
}

# 将给定的位图集合填充为全 1
void hwloc_bitmap_fill(struct hwloc_bitmap_s * set)
{
    # 检查位图集合是否合法
    HWLOC__BITMAP_CHECK(set);

    # 断言预分配的 ulong 数组长度至少为 1
    HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);
    # 如果重置位图集合失败，则不做任何操作
    if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
        /* cannot fail since we pre-allocate some ulongs.
         * if we ever pre-allocate nothing, we'll reset to 0 ulongs.
         */
    }
    # 调用内部函数填充位图集合
    hwloc_bitmap__fill(set);
}

# 根据给定的 ulong 创建位图集合
int hwloc_bitmap_from_ulong(struct hwloc_bitmap_s *set, unsigned long mask)
{
    # 检查位图集合是否合法
    HWLOC__BITMAP_CHECK(set);

    # 断言预分配的 ulong 数组长度至少为 1
    HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);
    # 如果重置位图集合失败，则不做任何操作
    if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
        /* cannot fail since we pre-allocate some ulongs.
         * if ever pre-allocate nothing, we may have to return a failure.
         */
    }
    # 将第一个 ulong 设置为给定的 mask
    set->ulongs[0] = mask; /* there's always at least one ulong allocated */
    # 将位图集合标记为有限
    set->infinite = 0;
    # 返回成功
    return 0;
}

# 根据给定的索引和 ulong 创建位图集合
int hwloc_bitmap_from_ith_ulong(struct hwloc_bitmap_s *set, unsigned i, unsigned long mask)
{
    # 初始化循环变量 j
    unsigned j;

    # 检查位图集合是否合法
    HWLOC__BITMAP_CHECK(set);

    # 如果重置位图集合失败，则返回失败
    if (hwloc_bitmap_reset_by_ulongs(set, i+1) < 0)
        return -1;

    # 将第 i 个 ulong 设置为给定的 mask
    set->ulongs[i] = mask;
    # 将前面的 ulong 设置为 HWLOC_SUBBITMAP_ZERO
    for(j=0; j<i; j++)
        set->ulongs[j] = HWLOC_SUBBITMAP_ZERO;
    # 将位图集合标记为有限
    set->infinite = 0;
    # 返回成功
    return 0;
}

# 根据给定的 ulong 数组创建位图集合
int hwloc_bitmap_from_ulongs(struct hwloc_bitmap_s *set, unsigned nr, const unsigned long *masks)
{
    # 初始化循环变量 j
    unsigned j;

    # 检查位图集合是否合法
    HWLOC__BITMAP_CHECK(set);

    # 如果重置位图集合失败，则返回失败
    if (hwloc_bitmap_reset_by_ulongs(set, nr) < 0)
        return -1;

    # 将给定的 ulong 数组复制到位图集合中
    for(j=0; j<nr; j++)
        set->ulongs[j] = masks[j];
    # 将位图集合标记为有限
    set->infinite = 0;
    # 返回成功
    return 0;
}

# 将位图集合转换为 ulong
unsigned long hwloc_bitmap_to_ulong(const struct hwloc_bitmap_s *set)
{
    # 检查位图集合是否合法
    HWLOC__BITMAP_CHECK(set);

    # 返回第一个 ulong
    return set->ulongs[0]; /* there's always at least one ulong allocated */
}

# 将位图集合的第 i 个 ulong 转换为 ulong
unsigned long hwloc_bitmap_to_ith_ulong(const struct hwloc_bitmap_s *set, unsigned i)
{
    # 检查位图是否已设置
    HWLOC__BITMAP_CHECK(set);
    
    # 从子位图中读取指定位置的无符号长整型数据
    return HWLOC_SUBBITMAP_READULONG(set, i);
# 将 hwloc_bitmap_s 结构体转换为一组无符号长整型数，存储在 masks 数组中
int hwloc_bitmap_to_ulongs(const struct hwloc_bitmap_s *set, unsigned nr, unsigned long *masks)
{
    unsigned j;

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 遍历位图中的每个位置，将对应位置的数据存储到 masks 数组中
    for(j=0; j<nr; j++)
        masks[j] = HWLOC_SUBBITMAP_READULONG(set, j);
    return 0;
}

# 返回位图中所需的无符号长整型数的数量
int hwloc_bitmap_nr_ulongs(const struct hwloc_bitmap_s *set)
{
    unsigned last;

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果位图是无限的，则返回-1
    if (set->infinite)
        return -1;

    # 计算位图中所需的无符号长整型数的数量
    last = hwloc_bitmap_last(set);
    return (last + HWLOC_BITS_PER_LONG)/HWLOC_BITS_PER_LONG;
}

# 将位图中只包含指定 CPU 的位置设置为 1
int hwloc_bitmap_only(struct hwloc_bitmap_s * set, unsigned cpu)
{
    unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果重置指定 CPU 的位置失败，则返回-1
    if (hwloc_bitmap_reset_by_cpu_index(set, cpu) < 0)
        return -1;

    # 将位图清零，并将指定 CPU 的位置设置为 1
    hwloc_bitmap__zero(set);
    set->ulongs[index_] |= HWLOC_SUBBITMAP_CPU(cpu);
    return 0;
}

# 将位图中除了指定 CPU 外的位置都设置为 1
int hwloc_bitmap_allbut(struct hwloc_bitmap_s * set, unsigned cpu)
{
    unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果重置指定 CPU 的位置失败，则返回-1
    if (hwloc_bitmap_reset_by_cpu_index(set, cpu) < 0)
        return -1;

    # 将位图中所有位置都设置为 1，然后将指定 CPU 的位置设置为 0
    hwloc_bitmap__fill(set);
    set->ulongs[index_] &= ~HWLOC_SUBBITMAP_CPU(cpu);
    return 0;
}

# 将位图中指定 CPU 的位置设置为 1
int hwloc_bitmap_set(struct hwloc_bitmap_s * set, unsigned cpu)
{
    unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果位图是无限的，并且设置的 CPU 位置在无限部分之内，则不做任何操作
    if (set->infinite && cpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
        return 0;

    # 如果重新分配内存失败，则返回-1
    if (hwloc_bitmap_realloc_by_cpu_index(set, cpu) < 0)
        return -1;

    # 将位图中指定 CPU 的位置设置为 1
    set->ulongs[index_] |= HWLOC_SUBBITMAP_CPU(cpu);
    return 0;
}

# 将位图中指定范围内的 CPU 的位置设置为 1
int hwloc_bitmap_set_range(struct hwloc_bitmap_s * set, unsigned begincpu, int _endcpu)
{
    unsigned i;
    unsigned beginset,endset;
    unsigned endcpu = (unsigned) _endcpu;

    # 检查输入的位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果结束 CPU 小于起始 CPU，则直接返回
    if (endcpu < begincpu)
        return 0;
    # 如果设置为无限并且起始CPU位置大于等于设置中的位数*每个长整型的位数，则无需操作，直接返回0
    if (set->infinite && begincpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
        /* setting only in the already-set infinite part, nothing to do */
        return 0;

    # 如果结束CPU位置为-1，则表示范围为无限

        /* 无限范围 */

        /* 确保我们可以处理包含begincpu的ulong */
        if (hwloc_bitmap_realloc_by_cpu_index(set, begincpu) < 0)
            return -1;

        /* 更新包含begincpu的ulong */
        beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
        set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
        /* 如果已经分配了begincpu之后的ulongs，则设置它们 */
        for(i=beginset+1; i<set->ulongs_count; i++)
            set->ulongs[i] = HWLOC_SUBBITMAP_FULL;
        /* 标记为无限设置 */
        set->infinite = 1;
    else:
        /* 有限范围 */

        /* 忽略与已设置的无限范围重叠的部分 */
        if (set->infinite && endcpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
            endcpu = set->ulongs_count * HWLOC_BITS_PER_LONG - 1;
        /* 确保我们可以处理包含begincpu和endcpu的ulongs */
        if (hwloc_bitmap_realloc_by_cpu_index(set, endcpu) < 0)
            return -1;

        /* 更新第一个和最后一个ulongs */
        beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
        endset = HWLOC_SUBBITMAP_INDEX(endcpu);
        if (beginset == endset):
            set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROMTO(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu), HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
        else:
            set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
            set->ulongs[endset] |= HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
        /* 设置范围中间的ulongs */
        for(i=beginset+1; i<endset; i++)
            set->ulongs[i] = HWLOC_SUBBITMAP_FULL;

    # 返回0表示操作成功
    return 0;
# 设置第 i 个位置的 ulong 值为 mask
int hwloc_bitmap_set_ith_ulong(struct hwloc_bitmap_s *set, unsigned i, unsigned long mask)
{
    # 检查输入的位图是否合法
    HWLOC__BITMAP_CHECK(set);

    # 如果需要的话，重新分配内存以容纳 i+1 个 ulong
    if (hwloc_bitmap_realloc_by_ulongs(set, i+1) < 0)
        return -1;

    # 设置第 i 个位置的 ulong 值为 mask
    set->ulongs[i] = mask;
    return 0;
}

# 清除位图中指定位置的位
int hwloc_bitmap_clr(struct hwloc_bitmap_s * set, unsigned cpu)
{
    # 计算指定 CPU 的索引
    unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

    # 检查输入的位图是否合法
    HWLOC__BITMAP_CHECK(set);

    # 如果要清除的位置在位图的无限部分内，则不做任何操作
    if (!set->infinite && cpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
        return 0;

    # 如果需要的话，重新分配内存以容纳指定 CPU 的位置
    if (hwloc_bitmap_realloc_by_cpu_index(set, cpu) < 0)
        return -1;

    # 清除指定位置的位
    set->ulongs[index_] &= ~HWLOC_SUBBITMAP_CPU(cpu);
    return 0;
}

# 清除位图中指定范围内的位
int hwloc_bitmap_clr_range(struct hwloc_bitmap_s * set, unsigned begincpu, int _endcpu)
{
    unsigned i;
    unsigned beginset,endset;
    unsigned endcpu = (unsigned) _endcpu;

    # 检查输入的位图是否合法
    HWLOC__BITMAP_CHECK(set);

    # 如果结束位置小于开始位置，则直接返回
    if (endcpu < begincpu)
        return 0;

    # 如果不是在位图的无限部分内清除，则直接返回
    if (!set->infinite && begincpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
        return 0;

    # 如果结束位置为 -1，则表示清除无限范围
    if (_endcpu == -1) {
        # 确保我们可以处理包含 begincpu 的 ulong
        if (hwloc_bitmap_realloc_by_cpu_index(set, begincpu) < 0)
            return -1;

        # 更新包含 begincpu 的 ulong
        beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
        set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
        # 清除 begincpu 之后的 ulong（如果有的话）
        for(i=beginset+1; i<set->ulongs_count; i++)
            set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
        # 标记无限部分为未设置
        set->infinite = 0;
    } else {
        /* 如果范围是有限的 */

        /* 忽略与已经取消设置的无限部分重叠的范围 */
        if (!set->infinite && endcpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
            endcpu = set->ulongs_count * HWLOC_BITS_PER_LONG - 1;
        /* 确保我们可以处理包含 begincpu 和 endcpu 的 ulongs */
        if (hwloc_bitmap_realloc_by_cpu_index(set, endcpu) < 0)
            return -1;

        /* 更新第一个和最后一个 ulongs */
        beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
        endset = HWLOC_SUBBITMAP_INDEX(endcpu);
        if (beginset == endset) {
            set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROMTO(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu), HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
        } else {
            set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
            set->ulongs[endset] &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
        }
        /* 清除范围中间的 ulongs */
        for(i=beginset+1; i<endset; i++)
            set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
    }

    return 0;
# 检查给定的位图中是否设置了指定的 CPU 位
int hwloc_bitmap_isset(const struct hwloc_bitmap_s * set, unsigned cpu)
{
    # 计算 CPU 位所在的子位图索引
    unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 返回指定 CPU 位是否被设置
    return (HWLOC_SUBBITMAP_READULONG(set, index_) & HWLOC_SUBBITMAP_CPU(cpu)) != 0;
}

# 检查位图是否全为零
int hwloc_bitmap_iszero(const struct hwloc_bitmap_s *set)
{
    unsigned i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果位图为无限大，则不全为零
    if (set->infinite)
        return 0;
    # 遍历位图的每个 unsigned long，检查是否全为零
    for(i=0; i<set->ulongs_count; i++)
        if (set->ulongs[i] != HWLOC_SUBBITMAP_ZERO)
            return 0;
    return 1;
}

# 检查位图是否全为一
int hwloc_bitmap_isfull(const struct hwloc_bitmap_s *set)
{
    unsigned i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果位图不是无限大，则不全为一
    if (!set->infinite)
        return 0;
    # 遍历位图的每个 unsigned long，检查是否全为一
    for(i=0; i<set->ulongs_count; i++)
        if (set->ulongs[i] != HWLOC_SUBBITMAP_FULL)
            return 0;
    return 1;
}

# 检查两个位图是否相等
int hwloc_bitmap_isequal (const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
    # 获取两个位图的 unsigned long 数量
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    unsigned min_count = count1 < count2 ? count1 : count2;
    unsigned i;

    # 检查两个位图是否有效
    HWLOC__BITMAP_CHECK(set1);
    HWLOC__BITMAP_CHECK(set2);

    # 遍历两个位图的 unsigned long，检查是否相等
    for(i=0; i<min_count; i++)
        if (set1->ulongs[i] != set2->ulongs[i];

    # 如果两个位图的 unsigned long 数量不相等，进一步检查剩余部分是否全为零或全为一
    if (count1 != count2) {
        unsigned long w1 = set1->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
        unsigned long w2 = set2->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
        for(i=min_count; i<count1; i++) {
            if (set1->ulongs[i] != w2)
                return 0;
        }
        for(i=min_count; i<count2; i++) {
            if (set2->ulongs[i] != w1)
                return 0;
        }
    }

    # 检查两个位图是否具有相同的无限大属性
    if (set1->infinite != set2->infinite)
        return 0;

    return 1;
}

# 检查两个位图是否有交集
int hwloc_bitmap_intersects (const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
    # 获取两个位图的 unsigned long 数量
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    unsigned min_count = count1 < count2 ? count1 : count2;
    # 声明一个无符号整数变量 i
    unsigned i;

    # 检查 set1 是否为有效的位图
    HWLOC__BITMAP_CHECK(set1);
    # 检查 set2 是否为有效的位图
    HWLOC__BITMAP_CHECK(set2);

    # 遍历 min_count 次，检查 set1 和 set2 对应位置的位是否都为1，是则返回1
    for(i=0; i<min_count; i++)
        if (set1->ulongs[i] & set2->ulongs[i])
            return 1;

    # 如果 count1 不等于 count2
    if (count1 != count2) {
        # 如果 set2 是无限位图
        if (set2->infinite) {
            # 从 min_count 开始遍历 set1 的 ulongs_count 次，如果有非零位则返回1
            for(i=min_count; i<set1->ulongs_count; i++)
                if (set1->ulongs[i])
                    return 1;
        }
        # 如果 set1 是无限位图
        if (set1->infinite) {
            # 从 min_count 开始遍历 set2 的 ulongs_count 次，如果有非零位则返回1
            for(i=min_count; i<set2->ulongs_count; i++)
                if (set2->ulongs[i])
                    return 1;
        }
    }

    # 如果 set1 和 set2 都是无限位图，则返回1
    if (set1->infinite && set2->infinite)
        return 1;

    # 否则返回0
    return 0;
}
# 检查一个位图是否包含在另一个位图中
int hwloc_bitmap_isincluded (const struct hwloc_bitmap_s *sub_set, const struct hwloc_bitmap_s *super_set)
{
    # 获取超集和子集的 ulong 数量
    unsigned super_count = super_set->ulongs_count;
    unsigned sub_count = sub_set->ulongs_count;
    # 获取最小的 ulong 数量
    unsigned min_count = super_count < sub_count ? super_count : sub_count;
    unsigned i;

    # 检查子集和超集是否合法
    HWLOC__BITMAP_CHECK(sub_set);
    HWLOC__BITMAP_CHECK(super_set);

    # 遍历最小数量的 ulong，检查是否超集的 ulong 包含子集的 ulong
    for(i=0; i<min_count; i++)
        if (super_set->ulongs[i] != (super_set->ulongs[i] | sub_set->ulongs[i]))
            return 0;

    # 如果超集和子集的 ulong 数量不相等
    if (super_count != sub_count) {
        # 如果超集不是无限的，检查超集多出来的部分是否为 0
        if (!super_set->infinite)
            for(i=min_count; i<sub_count; i++)
                if (sub_set->ulongs[i])
                    return 0;
        # 如果子集是无限的，检查子集多出来的部分是否为全 1
        if (sub_set->infinite)
            for(i=min_count; i<super_count; i++)
                if (super_set->ulongs[i] != HWLOC_SUBBITMAP_FULL)
                    return 0;
    }

    # 如果子集是无限的而超集不是，返回 0
    if (sub_set->infinite && !super_set->infinite)
        return 0;

    # 否则返回 1
    return 1;
}

# 对两个位图进行按位或操作
int hwloc_bitmap_or (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
    /* cache counts so that we can reset res even if it's also set1 or set2 */
    # 缓存计数，以便即使 res 也是 set1 或 set2，也可以重置 res
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    unsigned max_count = count1 > count2 ? count1 : count2;
    unsigned min_count = count1 + count2 - max_count;
    unsigned i;

    # 检查结果、set1 和 set2 是否合法
    HWLOC__BITMAP_CHECK(res);
    HWLOC__BITMAP_CHECK(set1);
    HWLOC__BITMAP_CHECK(set2);

    # 如果重置 res 失败，返回 -1
    if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
        return -1;

    # 对最小数量的 ulong 进行按位或操作
    for(i=0; i<min_count; i++)
        res->ulongs[i] = set1->ulongs[i] | set2->ulongs[i];
    # 如果两个计数不相等
    if (count1 != count2) {
        # 如果最小计数小于 count1
        if (min_count < count1) {
            # 如果 set2 是无限的，将结果的 ulongs_count 设置为最小计数
            if (set2->infinite) {
                res->ulongs_count = min_count;
            } else {
                # 否则，将 set1 的 ulongs 复制到结果的 ulongs 中
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = set1->ulongs[i];
            }
        } else {
            # 如果 set1 是无限的，将结果的 ulongs_count 设置为最小计数
            if (set1->infinite) {
                res->ulongs_count = min_count;
            } else {
                # 否则，将 set2 的 ulongs 复制到结果的 ulongs 中
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = set2->ulongs[i];
            }
        }
    }

    # 结果的无限属性为 set1 或 set2 的无限属性
    res->infinite = set1->infinite || set2->infinite;
    # 返回 0 表示成功
    return 0;
}
# 与操作，将res设置为set1和set2的交集
int hwloc_bitmap_and (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
    # 缓存计数，以便即使res也是set1或set2，也可以重置res
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    unsigned max_count = count1 > count2 ? count1 : count2;
    unsigned min_count = count1 + count2 - max_count;
    unsigned i;

    # 检查res是否为有效的位图
    HWLOC__BITMAP_CHECK(res);
    # 检查set1是否为有效的位图
    HWLOC__BITMAP_CHECK(set1);
    # 检查set2是否为有效的位图
    HWLOC__BITMAP_CHECK(set2);

    # 如果按照最大计数重置res失败，则返回-1
    if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
        return -1;

    # 对于每个位，计算set1和set2的交集，并存储到res中
    for(i=0; i<min_count; i++)
        res->ulongs[i] = set1->ulongs[i] & set2->ulongs[i];

    # 如果set1和set2的计数不相等
    if (count1 != count2) {
        # 如果min_count小于count1
        if (min_count < count1) {
            # 如果set2是无限的，则将set1的剩余部分复制到res中
            if (set2->infinite) {
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = set1->ulongs[i];
            } else {
                res->ulongs_count = min_count;
            }
        } else {
            # 如果set1是无限的，则将set2的剩余部分复制到res中
            if (set1->infinite) {
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = set2->ulongs[i];
            } else {
                res->ulongs_count = min_count;
            }
        }
    }

    # 设置res的无限属性为set1和set2的无限属性的逻辑与
    res->infinite = set1->infinite && set2->infinite;
    return 0;
}

# 非操作，将res设置为set1和set2的补集
int hwloc_bitmap_andnot (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
    # 缓存计数，以便即使res也是set1或set2，也可以重置res
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    unsigned max_count = count1 > count2 ? count1 : count2;
    unsigned min_count = count1 + count2 - max_count;
    unsigned i;

    # 检查res是否为有效的位图
    HWLOC__BITMAP_CHECK(res);
    # 检查set1是否为有效的位图
    HWLOC__BITMAP_CHECK(set1);
    # 检查set2是否为有效的位图
    HWLOC__BITMAP_CHECK(set2);

    # 如果按照最大计数重置res失败，则返回-1
    if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
        return -1;

    # 对于每个位，计算set1和set2的补集，并存储到res中
    for(i=0; i<min_count; i++)
        res->ulongs[i] = set1->ulongs[i] & ~set2->ulongs[i];
    # 如果两个计数不相等
    if (count1 != count2) {
        # 如果最小计数小于 count1
        if (min_count < count1) {
            # 如果 set2 不是无限的
            if (!set2->infinite) {
                # 从 min_count 到 max_count 复制 set1 的 ulongs 到 res 的 ulongs
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = set1->ulongs[i];
            } else {
                # 设置 res 的 ulongs_count 为 min_count
                res->ulongs_count = min_count;
            }
        } else {
            # 如果 set1 是无限的
            if (set1->infinite) {
                # 从 min_count 到 max_count 对 set2 的 ulongs 取反后复制到 res 的 ulongs
                for(i=min_count; i<max_count; i++)
                    res->ulongs[i] = ~set2->ulongs[i];
            } else {
                # 设置 res 的 ulongs_count 为 min_count
                res->ulongs_count = min_count;
            }
        }
    }

    # 设置 res 的 infinite 为 set1 的 infinite 与 set2 的 infinite 的逻辑与
    res->infinite = set1->infinite && !set2->infinite;
    # 返回 0
    return 0;
}
    # 如果集合是无限的，则返回集合中无符号长整型数的数量乘以每个长整型数的位数
    if (set->infinite)
        return set->ulongs_count * HWLOC_BITS_PER_LONG;
    
    # 如果集合不是无限的，则返回-1
    return -1;
}
# 返回给定位图中第一个未设置位的索引
int hwloc_bitmap_first_unset(const struct hwloc_bitmap_s * set)
{
    unsigned i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    for(i=0; i<set->ulongs_count; i++) {
        # 子集是无符号长整型，使用 ffsl 函数
        unsigned long w = ~set->ulongs[i];
        if (w)
            return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
    }

    if (!set->infinite)
        return set->ulongs_count * HWLOC_BITS_PER_LONG;

    return -1;
}

# 返回给定位图中最后一个设置位的索引
int hwloc_bitmap_last(const struct hwloc_bitmap_s * set)
{
    int i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    if (set->infinite)
        return -1;

    for(i=(int)set->ulongs_count-1; i>=0; i--) {
        # 子集是无符号长整型，使用 flsl 函数
        unsigned long w = set->ulongs[i];
        if (w)
            return hwloc_flsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
    }

    return -1;
}

# 返回给定位图中最后一个未设置位的索引
int hwloc_bitmap_last_unset(const struct hwloc_bitmap_s * set)
{
    int i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    if (!set->infinite)
        return -1;

    for(i=(int)set->ulongs_count-1; i>=0; i--) {
        # 子集是无符号长整型，使用 flsl 函数
        unsigned long w = ~set->ulongs[i];
        if (w)
            return hwloc_flsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
    }

    return -1;
}

# 返回给定位图中给定索引后的下一个设置位
int hwloc_bitmap_next(const struct hwloc_bitmap_s * set, int prev_cpu)
{
    unsigned i = HWLOC_SUBBITMAP_INDEX(prev_cpu + 1);

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    if (i >= set->ulongs_count) {
        if (set->infinite)
            return prev_cpu + 1;
        else
            return -1;
    }
    // 循环遍历集合中的每个 unsigned long
    for(; i<set->ulongs_count; i++) {
        /* subsets are unsigned longs, use ffsl */
        // 使用 ffsl 函数处理 unsigned long 类型的子集
        unsigned long w = set->ulongs[i];

        /* if the prev cpu is in the same word as the possible next one,
           we need to mask out previous cpus */
        // 如果前一个 CPU 在可能的下一个 CPU 所在的相同单词中，需要屏蔽掉前面的 CPU
        if (prev_cpu >= 0 && HWLOC_SUBBITMAP_INDEX((unsigned) prev_cpu) == i)
            w &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(prev_cpu));

        // 如果 w 不为 0，则返回 w 中第一个置位的位置
        if (w)
            return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
    }

    // 如果集合为无限集合，则返回集合中包含的 CPU 数量乘以每个 unsigned long 的位数
    if (set->infinite)
        return set->ulongs_count * HWLOC_BITS_PER_LONG;

    // 如果没有找到符合条件的 CPU，则返回 -1
    return -1;
# 找到下一个未设置的位
int hwloc_bitmap_next_unset(const struct hwloc_bitmap_s * set, int prev_cpu)
{
    # 计算 prev_cpu + 1 对应的子位图索引
    unsigned i = HWLOC_SUBBITMAP_INDEX(prev_cpu + 1);

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果索引超出了位图的范围
    if (i >= set->ulongs_count) {
        # 如果位图不是无限的，则返回下一个未设置的 CPU 编号
        if (!set->infinite)
            return prev_cpu + 1;
        # 如果位图是无限的，则返回 -1
        else
            return -1;
    }

    # 遍历位图的每个子位图
    for(; i<set->ulongs_count; i++) {
        # 使用 ffsl 函数找到子位图中未设置的位
        unsigned long w = ~set->ulongs[i];

        # 如果 prev_cpu 和可能的下一个 CPU 在同一个字中
        if (prev_cpu >= 0 && HWLOC_SUBBITMAP_INDEX((unsigned) prev_cpu) == i)
            # 需要屏蔽掉之前的 CPU
            w &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(prev_cpu));

        # 如果找到未设置的位，则返回对应的 CPU 编号
        if (w)
            return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
    }

    # 如果位图不是无限的，则返回位图中未设置的最大 CPU 编号
    if (!set->infinite)
        return set->ulongs_count * HWLOC_BITS_PER_LONG;

    # 如果位图是无限的，则返回 -1
    return -1;
}

# 将位图中的多个位设置为 0，只保留第一个非零位
int hwloc_bitmap_singlify(struct hwloc_bitmap_s * set)
{
    unsigned i;
    int found = 0;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 遍历位图的每个子位图
    for(i=0; i<set->ulongs_count; i++) {
        # 如果已经找到非零位
        if (found) {
            # 将当前子位图设置为 0
            set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
            continue;
        } else {
            # 使用 ffsl 函数找到子位图中第一个非零位
            unsigned long w = set->ulongs[i];
            if (w) {
                int _ffs = hwloc_ffsl(w);
                # 将子位图中只保留第一个非零位，其他位设置为 0
                set->ulongs[i] = HWLOC_SUBBITMAP_CPU(_ffs-1);
                found = 1;
            }
        }
    }

    # 如果位图是无限的
    if (set->infinite) {
        # 如果找到了非零位
        if (found) {
            # 将位图设置为有限的
            set->infinite = 0;
        } else {
            # 设置位图的第一个未分配的位
            unsigned first = set->ulongs_count * HWLOC_BITS_PER_LONG;
            set->infinite = 0; # 防止重新分配填充新分配的位图
            return hwloc_bitmap_set(set, first);
        }
    }

    return 0;
}

# 比较两个位图中第一个非零位的位置
int hwloc_bitmap_compare_first(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
    # 获取位图 set1 的子位图数量
    unsigned count1 = set1->ulongs_count;
    # 获取set2中的无符号长整型数的数量
    unsigned count2 = set2->ulongs_count;
    # 获取两个集合中无符号长整型数的最大数量
    unsigned max_count = count1 > count2 ? count1 : count2;
    # 获取两个集合中无符号长整型数的最小数量
    unsigned min_count = count1 + count2 - max_count;
    # 定义循环变量i
    unsigned i;

    # 检查set1是否为位图
    HWLOC__BITMAP_CHECK(set1);
    # 检查set2是否为位图
    HWLOC__BITMAP_CHECK(set2);

    # 遍历两个集合中的最小数量的无符号长整型数
    for(i=0; i<min_count; i++) {
        # 获取set1中第i个位置的无符号长整型数
        unsigned long w1 = set1->ulongs[i];
        # 获取set2中第i个位置的无符号长整型数
        unsigned long w2 = set2->ulongs[i];
        # 如果w1或w2不为0
        if (w1 || w2) {
            # 获取w1中第一个非零位的位置
            int _ffs1 = hwloc_ffsl(w1);
            # 获取w2中第一个非零位的位置
            int _ffs2 = hwloc_ffsl(w2);
            # 如果w1和w2都有位被设置，则进行真实比较
            if (_ffs1 && _ffs2)
                return _ffs1-_ffs2;
            # 如果其中一个集合为空，则将其视为更高位，进行反向比较
            return _ffs2-_ffs1;
        }
    }

    # 如果count1和count2不相等
    if (count1 != count2) {
        # 如果最小数量小于count2
        if (min_count < count2) {
            # 遍历set2中从min_count到count2的无符号长整型数
            for(i=min_count; i<count2; i++) {
                # 获取set2中第i个位置的无符号长整型数
                unsigned long w2 = set2->ulongs[i];
                # 如果set1为无限集合，则返回-1
                if (set1->infinite)
                    return -!(w2 & 1);
                # 如果w2不为0，则返回1
                else if (w2)
                    return 1;
            }
        } else {
            # 如果最小数量小于count1
            for(i=min_count; i<count1; i++) {
                # 获取set1中第i个位置的无符号长整型数
                unsigned long w1 = set1->ulongs[i];
                # 如果set2为无限集合，则返回1
                if (set2->infinite)
                    return !(w1 & 1);
                # 如果w1不为0，则返回-1
                else if (w1)
                    return -1;
            }
        }
    }

    # 返回set1是否为无限集合与set2是否为无限集合的布尔值差
    return !!set1->infinite - !!set2->infinite;
}
# 比较两个位图的内容是否相同
int hwloc_bitmap_compare(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
    # 获取两个位图的 ulong 数量
    unsigned count1 = set1->ulongs_count;
    unsigned count2 = set2->ulongs_count;
    # 获取最大和最小的 ulong 数量
    unsigned max_count = count1 > count2 ? count1 : count2;
    unsigned min_count = count1 + count2 - max_count;
    int i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set1);
    HWLOC__BITMAP_CHECK(set2);

    # 如果一个位图是无限的，另一个不是，则返回它们的无限性差异
    if ((!set1->infinite) != (!set2->infinite))
        return !!set1->infinite - !!set2->infinite;

    # 如果两个位图的 ulong 数量不同
    if (count1 != count2) {
        # 如果最小的 ulong 数量小于第二个位图的 ulong 数量
        if (min_count < count2) {
            # 获取第一个位图的值
            unsigned long val1 = set1->infinite ? HWLOC_SUBBITMAP_FULL :  HWLOC_SUBBITMAP_ZERO;
            # 从最大的 ulong 开始比较
            for(i=(int)max_count-1; i>=(int) min_count; i--) {
                unsigned long val2 = set2->ulongs[i];
                # 如果值相同，则继续比较下一个
                if (val1 == val2)
                    continue;
                # 返回比较结果
                return val1 < val2 ? -1 : 1;
            }
        } else {
            # 获取第二个位图的值
            unsigned long val2 = set2->infinite ? HWLOC_SUBBITMAP_FULL :  HWLOC_SUBBITMAP_ZERO;
            # 从最大的 ulong 开始比较
            for(i=(int)max_count-1; i>=(int) min_count; i--) {
                unsigned long val1 = set1->ulongs[i];
                # 如果值相同，则继续比较下一个
                if (val1 == val2)
                    continue;
                # 返回比较结果
                return val1 < val2 ? -1 : 1;
            }
        }
    }

    # 从最小的 ulong 开始比较
    for(i=(int)min_count-1; i>=0; i--) {
        unsigned long val1 = set1->ulongs[i];
        unsigned long val2 = set2->ulongs[i];
        # 如果值相同，则继续比较下一个
        if (val1 == val2)
            continue;
        # 返回比较结果
        return val1 < val2 ? -1 : 1;
    }

    # 位图相同，返回 0
    return 0;
}

# 计算位图中设置为 1 的位数
int hwloc_bitmap_weight(const struct hwloc_bitmap_s * set)
{
    int weight = 0;
    unsigned i;

    # 检查位图是否有效
    HWLOC__BITMAP_CHECK(set);

    # 如果位图是无限的，则返回 -1
    if (set->infinite)
        return -1;

    # 遍历位图中的 ulong，计算设置为 1 的位数
    for(i=0; i<set->ulongs_count; i++)
        weight += hwloc_weight_long(set->ulongs[i]);
    return weight;
}

# 比较两个位图的包含关系
int hwloc_bitmap_compare_inclusion(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
    # 获取两个位图中 ulong 数量较大的值
    unsigned max_count = set1->ulongs_count > set2->ulongs_count ? set1->ulongs_count : set2->ulongs_count;
    // 初始化结果为相等，表示空集合返回相等
    int result = HWLOC_BITMAP_EQUAL; /* means empty sets return equal */
    // 初始化两个集合为空
    int empty1 = 1;
    int empty2 = 1;
    unsigned i;

    // 检查集合1是否为空
    HWLOC__BITMAP_CHECK(set1);
    // 检查集合2是否为空
    HWLOC__BITMAP_CHECK(set2);

    }

    // 如果集合1不是无限的
    if (!set1->infinite) {
      // 如果集合2是无限的
      if (set2->infinite) {
        /* set2 infinite only */
        // 如果结果是包含
        if (result == HWLOC_BITMAP_CONTAINS) {
          // 如果集合2不为空
          if (!empty2)
        return HWLOC_BITMAP_INTERSECTS;
          // 结果变为不同
          result = HWLOC_BITMAP_DIFFERENT;
        } else if (result == HWLOC_BITMAP_EQUAL) {
          // 结果变为包含
          result = HWLOC_BITMAP_INCLUDED;
        }
        /* no change otherwise */
      }
    } else if (!set2->infinite) {
      /* set1 infinite only */
      // 如果结果是包含
      if (result == HWLOC_BITMAP_INCLUDED) {
        // 如果集合1不为空
        if (!empty1)
          return HWLOC_BITMAP_INTERSECTS;
        // 结果变为不同
        result = HWLOC_BITMAP_DIFFERENT;
      } else if (result == HWLOC_BITMAP_EQUAL) {
        // 结果变为包含
        result = HWLOC_BITMAP_CONTAINS;
      }
      /* no change otherwise */
    } else {
      /* both infinite */
      // 如果结果是不同
      if (result == HWLOC_BITMAP_DIFFERENT)
        return HWLOC_BITMAP_INTERSECTS;
      // 相等/包含/被包含不变
    }

    // 返回结果
    return result;
# 闭合前面的函数定义
```