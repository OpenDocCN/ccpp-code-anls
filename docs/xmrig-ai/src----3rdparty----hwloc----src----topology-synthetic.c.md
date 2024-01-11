# `xmrig\src\3rdparty\hwloc\src\topology-synthetic.c`

```
/*
 * 版权声明
 */
#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含硬件拓扑结构的头文件
#include "private/private.h"  // 包含私有数据结构的头文件
#include "private/misc.h"  // 包含私有杂项函数的头文件
#include "private/debug.h"  // 包含私有调试函数的头文件

#include <limits.h>  // 包含整数类型的最大值和最小值的头文件
#include <assert.h>  // 包含断言宏的头文件
#ifdef HAVE_STRINGS_H
#include <strings.h>  // 包含字符串操作函数的头文件
#endif

// 定义合成对象的属性结构体
struct hwloc_synthetic_attr_s {
  hwloc_obj_type_t type;  // 对象类型
  unsigned depth;  // 对象深度（用于缓存/组）
  hwloc_obj_cache_type_t cachetype;  // 缓存类型（用于缓存）
  hwloc_uint64_t memorysize;  // 内存大小（用于缓存/内存）
};

// 定义合成对象的索引结构体
struct hwloc_synthetic_indexes_s {
  const char *string;  // 解析前的索引字符串
  unsigned long string_length;  // 索引字符串的长度
  unsigned *array;  // 解析后的索引数组

  // 用于填充拓扑结构
  unsigned next;  // 下一个对象的ID
};

// 定义合成层级数据结构
struct hwloc_synthetic_level_data_s {
  unsigned arity;  // 子对象数量
  unsigned long totalwidth;  // 总宽度

  struct hwloc_synthetic_attr_s attr;  // 对象属性
  struct hwloc_synthetic_indexes_s indexes;  // 对象索引

  // 附加的合成属性
  struct hwloc_synthetic_attached_s {
    struct hwloc_synthetic_attr_s attr;  // 附加属性

    struct hwloc_synthetic_attached_s *next;  // 下一个附加属性
  } *attached;
};

// 定义合成后端数据结构
struct hwloc_synthetic_backend_data_s {
  char *string;  // 合成参数字符串

  unsigned long numa_attached_nr;  // NUMA附加数量
  struct hwloc_synthetic_indexes_s numa_attached_indexes;  // NUMA附加索引

#define HWLOC_SYNTHETIC_MAX_DEPTH 128
  struct hwloc_synthetic_level_data_s level[HWLOC_SYNTHETIC_MAX_DEPTH];  // 合成层级数据
};

// 定义合成交错循环结构
struct hwloc_synthetic_intlv_loop_s {
  unsigned step;  // 步长
  unsigned nb;  // 数量
  unsigned level_depth;  // 层级深度
};

// 处理索引的函数
static void
hwloc_synthetic_process_indexes(struct hwloc_synthetic_backend_data_s *data,
                struct hwloc_synthetic_indexes_s *indexes,
                unsigned long total,
                int verbose)
{
  // 定义指向字符串的指针，指向索引的字符串
  const char *attr = indexes->string;
  // 定义无符号长整型变量，表示索引字符串的长度
  unsigned long length = indexes->string_length;
  // 定义指向无符号整型的指针，初始化为空
  unsigned *array = NULL;
  // 定义变量 i，用于循环计数
  size_t i;

  // 如果索引字符串为空，则直接返回
  if (!attr)
    return;

  // 分配 total 个无符号整型的内存空间，初始化为 0
  array = calloc(total, sizeof(*array));
  // 如果分配内存失败
  if (!array) {
    // 如果 verbose 为真，则输出错误信息到标准错误流
    if (verbose)
      fprintf(stderr, "Failed to allocate synthetic index array of size %lu\n", total);
    // 跳转到 out 标签处
    goto out;
  }

  // 计算字符串 attr 中连续包含的数字和逗号的长度
  i = strspn(attr, "0123456789,");
  // 如果连续包含的长度等于字符串长度
  if (i == length) {
    /* explicit array of indexes */

    // 遍历 total 次
    for(i=0; i<total; i++) {
      // 定义指向字符串的指针 next
      const char *next;
      // 将字符串 attr 转换为无符号整型，存储在 idx 中
      unsigned idx = strtoul(attr, (char **) &next, 10);
      // 如果转换失败
      if (next == attr) {
        // 如果 verbose 为真，则输出错误信息到标准错误流
        if (verbose)
          fprintf(stderr, "Failed to read synthetic index #%lu at '%s'\n", (unsigned long) i, attr);
        // 跳转到 out_with_array 标签处
        goto out_with_array;
      }

      // 将 idx 存储在数组中
      array[i] = idx;
      // 如果不是最后一个元素
      if (i != total-1) {
        // 如果下一个字符不是逗号
        if (*next != ',') {
          // 如果 verbose 为真，则输出错误信息到标准错误流
          if (verbose)
            fprintf(stderr, "Missing comma after synthetic index #%lu at '%s'\n", (unsigned long) i, attr);
          // 跳转到 out_with_array 标签处
          goto out_with_array;
        }
        // 将指针 attr 指向下一个字符
        attr = next+1;
      } else {
        // 将指针 attr 指向下一个字符
        attr = next;
      }
    }
    // 将数组赋值给 indexes->array
    indexes->array = array;

  } else {
    /* interleaving */
    // 定义无符号整型变量，表示循环次数
    unsigned nr_loops = 1, cur_loop;
    // 定义无符号整型变量，表示最小步长
    unsigned minstep = total;
    // 定义无符号长整型变量，表示 nbs
    unsigned long nbs = 1;
    // 定义无符号整型变量，表示循环次数
    unsigned j, mul;
    // 定义指向字符串的指针 tmp
    const char *tmp;
    // 定义结构体数组，表示循环
    struct hwloc_synthetic_intlv_loop_s *loops;

    // 将指针 tmp 指向字符串 attr
    tmp = attr;
    // 循环直到 tmp 为 NULL
    while (tmp) {
      // 在 tmp 中查找冒号
      tmp = strchr(tmp, ':');
      // 如果找不到或者超出长度，则跳出循环
      if (!tmp || tmp >= attr+length)
        break;
      // 循环次数加一
      nr_loops++;
      // 指针 tmp 指向下一个字符
      tmp++;
    }

    // 分配 nr_loops+1 个结构体的内存空间
    loops = malloc((nr_loops+1) * sizeof(*loops));
    // 如果分配内存失败，则跳转到 out_with_array 标签处
    if (!loops)
      goto out_with_array;

    // 如果 attr 的第一个字符是数字
    if (*attr >= '0' && *attr <= '9') {
      /* interleaving as x*y:z*t:... */
      // 定义无符号整型变量，表示步长和数量
      unsigned step, nb;

      // 将指针 tmp 指向字符串 attr
      tmp = attr;
      // 当 tmp 不为 NULL 时循环
      cur_loop = 0;
      while (tmp) {
        // 定义指向字符的指针 tmp2 和 tmp3
        char *tmp2, *tmp3;
        // 将 tmp 转换为无符号整型，存储在 step 中
        step = (unsigned) strtol(tmp, &tmp2, 0);
    # 如果 tmp2 等于 tmp，或者 tmp2 指向的字符不是 '*'，则执行以下操作
    if (tmp2 == tmp || *tmp2 != '*') {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Failed to read synthetic index interleaving loop '%s' without number before '*'\n", tmp);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # 如果 step 为假值，则执行以下操作
    if (!step) {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Invalid interleaving loop with step 0 at '%s'\n", tmp);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # tmp2 指针向后移动一个位置
    tmp2++;
    # 将 tmp2 指向的字符串转换为无符号整数，存储到 nb 中
    nb = (unsigned) strtol(tmp2, &tmp3, 0);
    # 如果转换失败，或者 tmp3 指向的字符不是 ':'、')' 或者空格，则执行以下操作
    if (tmp3 == tmp2 || (*tmp3 && *tmp3 != ':' && *tmp3 != ')' && *tmp3 != ' ')) {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Failed to read synthetic index interleaving loop '%s' without number between '*' and ':'\n", tmp);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # 如果 nb 为假值，则执行以下操作
    if (!nb) {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Invalid interleaving loop with number 0 at '%s'\n", tmp2);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # 将当前循环的步长设置为 step
    loops[cur_loop].step = step;
    # 将当前循环的数量设置为 nb
    loops[cur_loop].nb = nb;
    # 如果 step 小于 minstep，则将 minstep 设置为 step
    if (step < minstep)
      minstep = step;
    # 将 nbs 乘以 nb
    nbs *= nb;
    # 当前循环数加一
    cur_loop++;
    # 如果 tmp3 指向的字符是 ')' 或者空格，则跳出循环
    if (*tmp3 == ')' || *tmp3 == ' ')
      break;
    # 将 tmp 指针设置为 tmp3+1
    tmp = (const char*) (tmp3+1);
      }

    } else {
      # 插入类型为 type1:type2:... 的交错
      hwloc_obj_type_t type;
      union hwloc_obj_attr_u attrs;
      int err;

      # 对每个交错循环找到级别深度
      tmp = attr;
      cur_loop = 0;
      # 当 tmp 不为空时执行以下操作
      while (tmp) {
    # 从 tmp 中读取类型和属性，存储到 type 和 attrs 中
    err = hwloc_type_sscanf(tmp, &type, &attrs, sizeof(attrs));
    # 如果读取失败，则执行以下操作
    if (err < 0) {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Failed to read synthetic index interleaving loop type '%s'\n", tmp);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # 如果类型为 Misc、Bridge、PCI 设备或者 OS 设备，则执行以下操作
    if (type == HWLOC_OBJ_MISC || type == HWLOC_OBJ_BRIDGE || type == HWLOC_OBJ_PCI_DEVICE || type == HWLOC_OBJ_OS_DEVICE) {
      # 如果 verbose 为真，则输出错误信息到标准错误流
      if (verbose)
        fprintf(stderr, "Misc object type disallowed in synthetic index interleaving loop type '%s'\n", tmp);
      # 释放 loops 指向的内存
      free(loops);
      # 跳转到标签 out_with_array 处
      goto out_with_array;
    }
    # 循环遍历数据的层级
    for(i=0; ; i++) {
      # 如果当前层级的arity为0，则将当前循环的level_depth设置为-1，并跳出循环
      if (!data->level[i].arity) {
        loops[cur_loop].level_depth = (unsigned)-1;
        break;
      }
      # 如果当前层级的类型不等于给定的类型，则继续下一次循环
      if (type != data->level[i].attr.type)
        continue;
      # 如果当前层级的类型为HWLOC_OBJ_GROUP，并且attrs.group.depth不为-1，并且不等于当前层级的深度，则继续下一次循环
      if (type == HWLOC_OBJ_GROUP
          && attrs.group.depth != (unsigned) -1
          && attrs.group.depth != data->level[i].attr.depth)
        continue;
      # 将当前循环的level_depth设置为当前层级的深度，并跳出循环
      loops[cur_loop].level_depth = (unsigned)i;
      break;
    }
    # 如果当前循环的level_depth为-1
    if (loops[cur_loop].level_depth == (unsigned)-1) {
      # 如果verbose为真，则在标准错误流中打印错误信息
      if (verbose)
        fprintf(stderr, "Failed to find level for synthetic index interleaving loop type '%s'\n",
            tmp);
      # 释放内存并跳转到标签out_with_array
      free(loops);
      goto out_with_array;
    }
    # 在tmp中查找冒号字符
    tmp = strchr(tmp, ':');
    # 如果tmp为空或者tmp大于attr+length，则跳出循环
    if (!tmp || tmp > attr+length)
      break;
    # tmp向后移动一个字符
    tmp++;
    # 当前循环加1
    cur_loop++;
      }

      # 计算实际循环步长/数量
      for(cur_loop=0; cur_loop<nr_loops; cur_loop++) {
    # 获取当前循环的level_depth
    unsigned mydepth = loops[cur_loop].level_depth;
    unsigned prevdepth = 0;
    unsigned step, nb;
    # 遍历循环
    for(i=0; i<nr_loops; i++) {
      # 如果循环i的level_depth等于当前循环的level_depth并且i不等于当前循环，则在标准错误流中打印错误信息，释放内存并跳转到标签out_with_array
      if (loops[i].level_depth == mydepth && i != cur_loop) {
        if (verbose)
          fprintf(stderr, "Invalid duplicate interleaving loop type in synthetic index '%s'\n", attr);
        free(loops);
        goto out_with_array;
      }
      # 如果循环i的level_depth小于当前循环的level_depth并且大于prevdepth，则将prevdepth设置为循环i的level_depth
      if (loops[i].level_depth < mydepth
          && loops[i].level_depth > prevdepth)
        prevdepth = loops[i].level_depth;
    }
    # 计算当前循环的step和nb
    step = total / data->level[mydepth].totalwidth; /* number of objects below us */
    nb = data->level[mydepth].totalwidth / data->level[prevdepth].totalwidth; /* number of us within parent */

    # 将当前循环的step和nb设置为计算得到的值
    loops[cur_loop].step = step;
    loops[cur_loop].nb = nb;
    # 断言nb和step不为0
    assert(nb);
    assert(step);
    # 如果step小于minstep，则将minstep设置为step
    if (step < minstep)
      minstep = step;
    # 计算nbs的乘积
    nbs *= nb;
      }
    }
    # 断言nbs不为0
    assert(nbs);

    # 如果nbs不等于total
    if (nbs != total) {
      # 如果total/nbs等于minstep
      if (minstep == total/nbs) {
    # 将新的循环的step设置为1，nb设置为total/nbs
    loops[nr_loops].step = 1;
    loops[nr_loops].nb = total/nbs;
    // 增加循环计数器
    nr_loops++;
      } else {
    // 如果索引交错总宽度无效，则输出错误信息并释放内存后跳转到标签out_with_array
    if (verbose)
      fprintf(stderr, "Invalid index interleaving total width %lu instead of %lu\n", nbs, total);
    free(loops);
    goto out_with_array;
      }
    }

    /* 生成索引数组 */
    mul = 1;
    for(i=0; i<nr_loops; i++) {
      unsigned step = loops[i].step;
      unsigned nb = loops[i].nb;
      for(j=0; j<total; j++)
    // 根据步长和数量计算索引数组的值
    array[j] += ((j / step) % nb) * mul;
      mul *= nb;
    }

    free(loops);

    /* 检查是否有正确的值（不能传递total，不能给出重复的0） */
    for(j=0; j<total; j++) {
      // 如果索引超出范围，则输出错误信息并跳转到标签out_with_array
      if (array[j] >= total) {
    if (verbose)
      fprintf(stderr, "Invalid index interleaving generates out-of-range index %u\n", array[j]);
    goto out_with_array;
      }
      // 如果生成了重复的索引值，则输出错误信息并跳转到标签out_with_array
      if (!array[j] && j) {
    if (verbose)
      fprintf(stderr, "Invalid index interleaving generates duplicate index values\n");
    goto out_with_array;
      }
    }

    // 将数组指针赋值给indexes->array
    indexes->array = array;
  }

  return;

 out_with_array:
  // 释放数组内存后跳转到标签out
  free(array);
 out:
  return;
# 解析内存属性字符串，返回内存大小
static hwloc_uint64_t
hwloc_synthetic_parse_memory_attr(const char *attr, const char **endp)
{
  const char *endptr;
  hwloc_uint64_t size;
  size = strtoull(attr, (char **) &endptr, 0);  # 将字符串转换为无符号长长整型数
  if (!hwloc_strncasecmp(endptr, "TB", 2)) {  # 如果单位是 TB
    size *= 1000ULL*1000ULL*1000ULL*1000ULL;  # 将大小转换为字节
    endptr += 2;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "TiB", 3)) {  # 如果单位是 TiB
    size <<= 40;  # 将大小左移 40 位，转换为字节
    endptr += 3;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "GB", 2)) {  # 如果单位是 GB
    size *= 1000ULL*1000ULL*1000ULL;  # 将大小转换为字节
    endptr += 2;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "GiB", 3)) {  # 如果单位是 GiB
    size <<= 30;  # 将大小左移 30 位，转换为字节
    endptr += 3;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "MB", 2)) {  # 如果单位是 MB
    size *= 1000ULL*1000ULL;  # 将大小转换为字节
    endptr += 2;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "MiB", 3)) {  # 如果单位是 MiB
    size <<= 20;  # 将大小左移 20 位，转换为字节
    endptr += 3;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "kB", 2)) {  # 如果单位是 kB
    size *= 1000ULL;  # 将大小转换为字节
    endptr += 2;  # 移动指针到单位之后
  } else if (!hwloc_strncasecmp(endptr, "kiB", 3)) {  # 如果单位是 kiB
    size <<= 10;  # 将大小左移 10 位，转换为字节
    endptr += 3;  # 移动指针到单位之后
  }
  *endp = endptr;  # 将解析后的指针位置保存到 endp
  return size;  # 返回解析后的内存大小
}

# 解析属性字符串，返回解析结果
static int
hwloc_synthetic_parse_attrs(const char *attrs, const char **next_posp,
                struct hwloc_synthetic_attr_s *sattr,
                struct hwloc_synthetic_indexes_s *sind,
                int verbose)
{
  hwloc_obj_type_t type = sattr->type;  # 获取对象类型
  const char *next_pos;  # 下一个位置的指针
  hwloc_uint64_t memorysize = 0;  # 内存大小
  const char *index_string = NULL;  # 索引字符串
  size_t index_string_length = 0;  # 索引字符串长度

  next_pos = (const char *) strchr(attrs, ')');  # 查找下一个位置的指针
  if (!next_pos) {  # 如果没有找到下一个位置
    if (verbose)  # 如果需要输出详细信息
      fprintf(stderr, "Missing attribute closing bracket in synthetic string doesn't have a number of objects at '%s'\n", attrs);  # 输出错误信息
    errno = EINVAL;  # 设置错误码为无效参数
    return -1;  # 返回错误
  }

  while (')' != *attrs) {  # 循环直到找到属性结束符
    int iscache = hwloc__obj_type_is_cache(type);  # 判断是否为缓存对象

    if (iscache && !strncmp("size=", attrs, 5)) {  # 如果是缓存对象且属性是 size
      memorysize = hwloc_synthetic_parse_memory_attr(attrs+5, &attrs);  # 解析内存属性

    } else if (!iscache && !strncmp("memory=", attrs, 7)) {  # 如果不是缓存对象且属性是 memory
      memorysize = hwloc_synthetic_parse_memory_attr(attrs+7, &attrs);  # 解析内存属性
    // 如果属性以"indexes="开头，则提取索引字符串并更新属性指针
    } else if (!strncmp("indexes=", attrs, 8)) {
      index_string = attrs+8;
      attrs += 8;
      index_string_length = strcspn(attrs, " )");
      attrs += index_string_length;

    // 如果属性不以"indexes="开头，则输出未知属性的错误信息并返回错误值
    } else {
      if (verbose)
    fprintf(stderr, "Unknown attribute at '%s'\n", attrs);
      errno = EINVAL;
      return -1;
    }

    // 如果属性指针指向空格，则移动到下一个字符；否则检查是否缺少参数分隔符并输出错误信息
    if (' ' == *attrs)
      attrs++;
    else if (')' != *attrs) {
      if (verbose)
    fprintf(stderr, "Missing parameter separator at '%s'\n", attrs);
      errno = EINVAL;
      return -1;
    }
  }

  // 更新sattr结构体中的memorysize属性
  sattr->memorysize = memorysize;

  // 如果存在索引字符串，则将其存储到sind结构体中，并输出覆盖重复索引属性的警告信息
  if (index_string) {
    if (sind->string && verbose)
      fprintf(stderr, "Overwriting duplicate indexes attribute with last occurence\n");
    sind->string = index_string;
    sind->string_length = (unsigned long)index_string_length;
  }

  // 更新next_posp指针的值，并返回成功值
  *next_posp = next_pos+1;
  return 0;
/* 释放层级直到 arity = 0 */
static void
hwloc_synthetic_free_levels(struct hwloc_synthetic_backend_data_s *data)
{
  unsigned i;
  for(i=0; i<HWLOC_SYNTHETIC_MAX_DEPTH; i++) {
    // 获取当前层级的数据
    struct hwloc_synthetic_level_data_s *curlevel = &data->level[i];
    // 获取当前层级的附加数据的指针的指针
    struct hwloc_synthetic_attached_s **pprev = &curlevel->attached;
    // 释放当前层级的附加数据
    while (*pprev) {
      struct hwloc_synthetic_attached_s *cur = *pprev;
      *pprev = cur->next;
      free(cur);
    }
    // 释放当前层级的索引数组
    free(curlevel->indexes.array);
    // 如果 arity 为 0，则跳出循环
    if (!curlevel->arity)
      break;
  }
  // 释放 NUMA 附加索引数组
  free(data->numa_attached_indexes.array);
}

/* 从描述中读取一系列描述对称拓扑的整数，并相应地更新 hwloc_synthetic_backend_data_s。成功时返回零。 */
static int
hwloc_backend_synthetic_init(struct hwloc_synthetic_backend_data_s *data,
                 const char *description)
{
  const char *pos, *next_pos;
  unsigned long item, count;
  unsigned i;
  int type_count[HWLOC_OBJ_TYPE_MAX];
  unsigned unset;
  int verbose = 0;
  const char *env = getenv("HWLOC_SYNTHETIC_VERBOSE");
  int err;
  unsigned long totalarity = 1;

  // 如果存在环境变量 HWLOC_SYNTHETIC_VERBOSE，则设置 verbose 为其值
  if (env)
    verbose = atoi(env);

  // 初始化 NUMA 附加数量和索引数组
  data->numa_attached_nr = 0;
  data->numa_attached_indexes.array = NULL;

  // 在添加根属性之前的默认值
  data->level[0].totalwidth = 1;
  data->level[0].attr.type = HWLOC_OBJ_MACHINE;
  data->level[0].indexes.string = NULL;
  data->level[0].indexes.array = NULL;
  data->level[0].attr.memorysize = 0;
  data->level[0].attached = NULL;
  type_count[HWLOC_OBJ_MACHINE] = 1;
  // 如果描述以 '(' 开头，则解析属性
  if (*description == '(') {
    err = hwloc_synthetic_parse_attrs(description+1, &description, &data->level[0].attr, &data->level[0].indexes, verbose);
    if (err < 0)
      return err;
  }

  // 初始化 NUMA 附加索引字符串和数组
  data->numa_attached_indexes.string = NULL;
  data->numa_attached_indexes.array = NULL;

  // 遍历描述字符串，解析拓扑类型和属性
  for (pos = description, count = 1; *pos; pos = next_pos) {
    hwloc_obj_type_t type = HWLOC_OBJ_TYPE_NONE;
    union hwloc_obj_attr_u attrs;
    /* 将父级arity初始化为0，以防止级别无限增加 */
    data->level[count-1].arity = 0;

    /* 跳过空格和换行符 */
    while (*pos == ' ' || *pos == '\n')
      pos++;

    /* 如果pos为空，则跳出循环 */
    if (!*pos)
      break;

    /* 如果pos指向'['，则处理attached对象 */
    if (*pos == '[') {
      /* attached对象 */
      struct hwloc_synthetic_attached_s *attached, **pprev;
      char *attr;

      pos++;

      /* 从pos中解析出attached对象的类型和属性 */
      if (hwloc_type_sscanf(pos, &type, &attrs, sizeof(attrs)) < 0) {
        if (verbose)
          fprintf(stderr, "Synthetic string with unknown attached object type at '%s'\n", pos);
        errno = EINVAL;
        goto error;
      }
      /* 如果attached对象类型不是NUMANODE，则报错 */
      if (type != HWLOC_OBJ_NUMANODE) {
        if (verbose)
          fprintf(stderr, "Synthetic string with disallowed attached object type at '%s'\n", pos);
        errno = EINVAL;
        goto error;
      }
      /* 更新numa_attached_nr的值 */
      data->numa_attached_nr += data->level[count-1].totalwidth;

      /* 分配内存并初始化attached对象 */
      attached = malloc(sizeof(*attached));
      if (attached) {
        attached->attr.type = type;
        attached->attr.memorysize = 0;
        /* attached->attr.depth and .cachetype unused */
        attached->next = NULL;
        pprev = &data->level[count-1].attached;
        while (*pprev)
          pprev = &((*pprev)->next);
        *pprev = attached;
      }

      /* 查找']'的位置，如果找不到则报错 */
      next_pos = strchr(pos, ']');
      if (!next_pos) {
        if (verbose)
          fprintf(stderr,"Synthetic string doesn't have a closing `]' after attached object type at '%s'\n", pos);
        errno = EINVAL;
        goto error;
      }

      /* 查找'('的位置，如果找到则解析属性 */
      attr = strchr(pos, '(');
      if (attr && attr < next_pos && attached) {
        const char *dummy;
        err = hwloc_synthetic_parse_attrs(attr+1, &dummy, &attached->attr, &data->numa_attached_indexes, verbose);
        if (err < 0)
          goto error;
      }

      /* 继续处理下一个对象 */
      next_pos++;
      continue;
    }

    /* 处理普通级别 */

    /* 重置默认值 */
    data->level[count].indexes.string = NULL;
    data->level[count].indexes.array = NULL;
    data->level[count].attached = NULL;

    /* 如果pos指向的字符不是数字，则解析类型和属性 */
    if (*pos < '0' || *pos > '9') {
      if (hwloc_type_sscanf(pos, &type, &attrs, sizeof(attrs)) < 0) {
    if (!strncmp(pos, "Tile", 4) || !strncmp(pos, "Module", 6)) {
      /* 如果字符串以"Tile"开头的4个字符或者"Module"开头的6个字符，则类型为HWLOC_OBJ_GROUP */
      type = HWLOC_OBJ_GROUP;
    } else {
      /* 否则，打印错误信息并跳转到错误处理部分 */
      if (verbose)
        fprintf(stderr, "Synthetic string with unknown object type at '%s'\n", pos);
      errno = EINVAL;
      goto error;
    }
      }
      if (type == HWLOC_OBJ_MACHINE || type == HWLOC_OBJ_MISC || type == HWLOC_OBJ_BRIDGE || type == HWLOC_OBJ_PCI_DEVICE || type == HWLOC_OBJ_OS_DEVICE) {
    /* 如果类型为HWLOC_OBJ_MACHINE、HWLOC_OBJ_MISC、HWLOC_OBJ_BRIDGE、HWLOC_OBJ_PCI_DEVICE或HWLOC_OBJ_OS_DEVICE，则打印错误信息并跳转到错误处理部分 */
    if (verbose)
      fprintf(stderr, "Synthetic string with disallowed object type at '%s'\n", pos);
    errno = EINVAL;
    goto error;
      }

      next_pos = strchr(pos, ':');
      if (!next_pos) {
    /* 如果字符串中不包含冒号，则打印错误信息并跳转到错误处理部分 */
    if (verbose)
      fprintf(stderr,"Synthetic string doesn't have a `:' after object type at '%s'\n", pos);
    errno = EINVAL;
    goto error;
      }
      pos = next_pos + 1;
    }

    data->level[count].attr.type = type;
    data->level[count].attr.depth = (unsigned) -1;
    data->level[count].attr.cachetype = (hwloc_obj_cache_type_t) -1;
    if (hwloc__obj_type_is_cache(type)) {
      /* 如果类型为缓存，则设置深度和缓存类型 */
      data->level[count].attr.depth = attrs.cache.depth;
      data->level[count].attr.cachetype = attrs.cache.type;
    } else if (type == HWLOC_OBJ_GROUP) {
      /* 如果类型为HWLOC_OBJ_GROUP，则设置深度 */
      data->level[count].attr.depth = attrs.group.depth;
    }

    /* 子对象的数量 */
    item = strtoul(pos, (char **)&next_pos, 0);
    if (next_pos == pos) {
      /* 如果字符串中不包含数字，则打印错误信息并跳转到错误处理部分 */
      if (verbose)
    fprintf(stderr,"Synthetic string doesn't have a number of objects at '%s'\n", pos);
      errno = EINVAL;
      goto error;
    }
    if (!item) {
      /* 如果子对象数量为0，则打印错误信息并跳转到错误处理部分 */
      if (verbose)
    fprintf(stderr,"Synthetic string with disallow 0 number of objects at '%s'\n", pos);
      errno = EINVAL;
      goto error;
    }

    totalarity *= item;
    data->level[count].totalwidth = totalarity;
    # 将指向字符串的指针设为NULL
    data->level[count].indexes.string = NULL;
    # 将指向数组的指针设为NULL
    data->level[count].indexes.array = NULL;
    # 将内存大小设为0
    data->level[count].attr.memorysize = 0;
    # 如果下一个位置是'('，则解析属性并更新下一个位置
    if (*next_pos == '(') {
      err = hwloc_synthetic_parse_attrs(next_pos+1, &next_pos, &data->level[count].attr, &data->level[count].indexes, verbose);
      # 如果解析出错，则跳转到错误处理部分
      if (err < 0)
        goto error;
    }
    # 如果计数超过最大深度，则输出错误信息并跳转到错误处理部分
    if (count + 1 >= HWLOC_SYNTHETIC_MAX_DEPTH) {
      if (verbose)
        fprintf(stderr,"Too many synthetic levels, max %d\n", HWLOC_SYNTHETIC_MAX_DEPTH);
      errno = EINVAL;
      goto error;
    }
    # 如果项目大于UINT_MAX，则输出错误信息并跳转到错误处理部分
    if (item > UINT_MAX) {
      if (verbose)
        fprintf(stderr,"Too big arity, max %u\n", UINT_MAX);
      errno = EINVAL;
      goto error;
    }
    # 更新上一级的度数
    data->level[count-1].arity = (unsigned)item;
    # 计数加一
    count++;
  }

  # 如果最后一级的类型不是HWLOC_OBJ_TYPE_NONE且不是HWLOC_OBJ_PU，则输出错误信息并返回-1
  if (data->level[count-1].attr.type != HWLOC_OBJ_TYPE_NONE && data->level[count-1].attr.type != HWLOC_OBJ_PU) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot use non-PU type for last level\n");
    errno = EINVAL;
    return -1;
  }
  # 将最后一级的类型设为HWLOC_OBJ_PU
  data->level[count-1].attr.type = HWLOC_OBJ_PU;

  # 将type_count数组中的每个元素初始化为0
  for(i=HWLOC_OBJ_TYPE_MIN; i<HWLOC_OBJ_TYPE_MAX; i++) {
    type_count[i] = 0;
  }
  # 从后往前遍历每一级，统计每种类型的数量
  for(i=count-1; i>0; i--) {
    hwloc_obj_type_t type = data->level[i].attr.type;
    if (type != HWLOC_OBJ_TYPE_NONE) {
      type_count[type]++;
    }
  }

  # 进行一些合法性检查
  if (!type_count[HWLOC_OBJ_PU]) {
    if (verbose)
      fprintf(stderr, "Synthetic string missing ending number of PUs\n");
    errno = EINVAL;
    return -1;
  } else if (type_count[HWLOC_OBJ_PU] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several PU levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_PACKAGE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several package levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_DIE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several die levels\n");
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }
  # 如果 NUMA 节点的数量大于 1，则报错
  if (type_count[HWLOC_OBJ_NUMANODE] > 1) {
    if (verbose)
      # 输出错误信息到标准错误流
      fprintf(stderr, "Synthetic string cannot have several NUMA node levels\n");
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }
  # 如果存在 NUMA 节点，并且已经附加了 NUMA 节点，则报错
  if (type_count[HWLOC_OBJ_NUMANODE] && data->numa_attached_nr) {
    if (verbose)
      # 输出错误信息到标准错误流
      fprintf(stderr,"Synthetic string cannot have NUMA nodes both as a level and attached\n");
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }
  # 如果核心的数量大于 1，则报错
  if (type_count[HWLOC_OBJ_CORE] > 1) {
    if (verbose)
      # 输出错误信息到标准错误流
      fprintf(stderr, "Synthetic string cannot have several core levels\n");
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }

  # 处理缺失的中间级别
  unset = 0;
  for(i=1; i<count-1; i++) {
    # 如果当前级别的类型为 NONE，则计数未设置的级别
    if (data->level[i].attr.type == HWLOC_OBJ_TYPE_NONE)
      unset++;
  }
  # 如果存在未设置的级别，并且未设置的级别数量不等于总级别数减去 2，则报错
  if (unset && unset != count-2) {
    if (verbose)
      # 输出错误信息到标准错误流
      fprintf(stderr, "Synthetic string cannot mix unspecified and specified types for levels\n");
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }
  if (unset) {
    # 我们优先考虑：NUMA、package、core、最多 3 个缓存、groups
    unsigned _count = count;
    unsigned neednuma = 0;
    unsigned needpack = 0;
    unsigned needcore = 0;
    unsigned needcaches = 0;
    unsigned needgroups = 0;
    # 为机器和处理器单元保留 2 个级别
    _count -= 2;

    neednuma = (_count >= 1 && !data->numa_attached_nr);
    _count -= neednuma;

    needpack = (_count >= 1);
    _count -= needpack;

    needcore = (_count >= 1);
    _count -= needcore;

    needcaches = (_count > 4 ? 4 : _count);
    _count -= needcaches;

    needgroups = _count;

    # 按顺序设置级别类型：groups、package、numa、caches、core
    for(i = 0; i < needgroups; i++) {
      unsigned depth = 1 + i;
      data->level[depth].attr.type = HWLOC_OBJ_GROUP;
      type_count[HWLOC_OBJ_GROUP]++;
    }
    if (needpack) {
      unsigned depth = 1 + needgroups;
      data->level[depth].attr.type = HWLOC_OBJ_PACKAGE;
      type_count[HWLOC_OBJ_PACKAGE] = 1;
    }
    # 如果需要 NUMA 节点
    if (neednuma) {
      # 计算深度，需要考虑组、包和 NUMA 节点
      unsigned depth = 1 + needgroups + needpack;
      # 设置 NUMA 节点的属性
      data->level[depth].attr.type = HWLOC_OBJ_NUMANODE;
      # 统计 NUMA 节点的数量
      type_count[HWLOC_OBJ_NUMANODE] = 1;
    }
    # 如果需要缓存
    if (needcaches) {
      # 设置 L3 缓存的深度
      unsigned l3depth = 1 + needgroups + needpack + neednuma;
      # 设置 L2 缓存的深度
      unsigned l2depth = l3depth + (needcaches >= 3);
      # 设置 L1 缓存的深度
      unsigned l1depth = l2depth + 1;
      # 设置 L1I 缓存的深度
      unsigned l1idepth = l1depth + 1;
      # 如果需要 3 级缓存
      if (needcaches >= 3) {
        # 设置 L3 缓存的属性
        data->level[l3depth].attr.type = HWLOC_OBJ_L3CACHE;
        data->level[l3depth].attr.depth = 3;
        data->level[l3depth].attr.cachetype = HWLOC_OBJ_CACHE_UNIFIED;
        # 统计 L3 缓存的数量
        type_count[HWLOC_OBJ_L3CACHE] = 1;
      }
      # 设置 L2 缓存的属性
      data->level[l2depth].attr.type = HWLOC_OBJ_L2CACHE;
      data->level[l2depth].attr.depth = 2;
      data->level[l2depth].attr.cachetype = HWLOC_OBJ_CACHE_UNIFIED;
      # 统计 L2 缓存的数量
      type_count[HWLOC_OBJ_L2CACHE] = 1;
      # 如果需要 2 级缓存
      if (needcaches >= 2) {
        # 设置 L1 缓存的属性
        data->level[l1depth].attr.type = HWLOC_OBJ_L1CACHE;
        data->level[l1depth].attr.depth = 1;
        data->level[l1depth].attr.cachetype = HWLOC_OBJ_CACHE_DATA;
        # 统计 L1 缓存的数量
        type_count[HWLOC_OBJ_L1CACHE] = 1;
      }
      # 如果需要 4 级缓存
      if (needcaches >= 4) {
        # 设置 L1I 缓存的属性
        data->level[l1idepth].attr.type = HWLOC_OBJ_L1ICACHE;
        data->level[l1idepth].attr.depth = 1;
        data->level[l1idepth].attr.cachetype = HWLOC_OBJ_CACHE_INSTRUCTION;
        # 统计 L1I 缓存的数量
        type_count[HWLOC_OBJ_L1ICACHE] = 1;
      }
    }
    # 如果需要核心
    if (needcore) {
      # 计算深度，需要考虑组、包、NUMA 和缓存
      unsigned depth = 1 + needgroups + needpack + neednuma + needcaches;
      # 设置核心的属性
      data->level[depth].attr.type = HWLOC_OBJ_CORE;
      # 统计核心的数量
      type_count[HWLOC_OBJ_CORE] = 1;
    }
  }

  # 强制添加一个 NUMA 级别
  if (!type_count[HWLOC_OBJ_NUMANODE] && !data->numa_attached_nr) {
    # 在自动机器根节点下插入一个 NUMA 级别
    if (verbose)
      # 输出信息到标准错误流
      fprintf(stderr, "Inserting a NUMA level with a single object at depth 1\n");
    # 将现有级别向后移动一个位置
    memmove(&data->level[2], &data->level[1], count*sizeof(struct hwloc_synthetic_level_data_s));
    # 设置第一层级的属性为NUMANODE
    data->level[1].attr.type = HWLOC_OBJ_NUMANODE;
    # 清空索引字符串
    data->level[1].indexes.string = NULL;
    # 清空索引数组
    data->level[1].indexes.array = NULL;
    # 设置内存大小为0
    data->level[1].attr.memorysize = 0;
    # 设置总宽度为第0层级的总宽度
    data->level[1].totalwidth = data->level[0].totalwidth;
    # 更新层级的度以插入单个NUMA节点
    data->level[1].arity = data->level[0].arity;
    # 将第0层级的度设置为1
    data->level[0].arity = 1;
    # 计数加一
    count++;
  }

  # 遍历每个层级
  for (i=0; i<count; i++) {
    # 获取当前层级的数据
    struct hwloc_synthetic_level_data_s *curlevel = &data->level[i];
    # 获取当前层级的类型
    hwloc_obj_type_t type = curlevel->attr.type;

    # 如果类型是GROUP
    if (type == HWLOC_OBJ_GROUP) {
      # 如果当前层级的深度为-1，则将其设置为HWLOC_OBJ_GROUP的计数值
      if (curlevel->attr.depth == (unsigned)-1)
    curlevel->attr.depth = type_count[HWLOC_OBJ_GROUP]--;

    } else if (hwloc__obj_type_is_cache(type)) {
      # 如果内存大小为0
      if (!curlevel->attr.memorysize) {
    # 如果深度为1
    if (1 == curlevel->attr.depth)
      # 在L1缓存中为32KiB
      curlevel->attr.memorysize = 32*1024;
    else
      # 每个层级*4，从L2开始为1MiB，统一的
      curlevel->attr.memorysize = 256ULL*1024 << (2*curlevel->attr.depth);
      }

    } else if (type == HWLOC_OBJ_NUMANODE && !curlevel->attr.memorysize) {
      # 在内存节点中为1GiB
      curlevel->attr.memorysize = 1024*1024*1024;
    }

    # 处理层级的索引
    hwloc_synthetic_process_indexes(data, &data->level[i].indexes, data->level[i].totalwidth, verbose);
  }

  # 处理NUMA节点附加的索引
  hwloc_synthetic_process_indexes(data, &data->numa_attached_indexes, data->numa_attached_nr, verbose);

  # 复制描述字符串
  data->string = strdup(description);
  # 将最后一个层级的度设置为0
  data->level[count-1].arity = 0;
  # 返回0表示成功
  return 0;

 error:
  # 释放层级数据
  hwloc_synthetic_free_levels(data);
  # 返回-1表示出错
  return -1;
# 设置合成属性，根据对象类型进行不同的处理
static void
hwloc_synthetic_set_attr(struct hwloc_synthetic_attr_s *sattr,
             hwloc_obj_t obj)
{
  # 根据对象类型进行不同的处理
  switch (obj->type) {
  case HWLOC_OBJ_GROUP:
    # 设置组对象的属性
    obj->attr->group.kind = HWLOC_GROUP_KIND_SYNTHETIC;
    obj->attr->group.subkind = sattr->depth-1;
    break;
  case HWLOC_OBJ_MACHINE:
    # 机器对象不做任何处理
    break;
  case HWLOC_OBJ_NUMANODE:
    # 设置NUMA节点对象的本地内存和页面类型
    obj->attr->numanode.local_memory = sattr->memorysize;
    obj->attr->numanode.page_types_len = 1;
    obj->attr->numanode.page_types = malloc(sizeof(*obj->attr->numanode.page_types));
    memset(obj->attr->numanode.page_types, 0, sizeof(*obj->attr->numanode.page_types));
    obj->attr->numanode.page_types[0].size = 4096;
    obj->attr->numanode.page_types[0].count = sattr->memorysize / 4096;
    break;
  case HWLOC_OBJ_PACKAGE:
  case HWLOC_OBJ_DIE:
    # 处理处理器包和DIE对象
    break;
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
    # 设置缓存对象的深度、行大小、类型和大小
    obj->attr->cache.depth = sattr->depth;
    obj->attr->cache.linesize = 64;
    obj->attr->cache.type = sattr->cachetype;
    obj->attr->cache.size = sattr->memorysize;
    break;
  case HWLOC_OBJ_CORE:
    # 处理核心对象
    break;
  case HWLOC_OBJ_PU:
    # 处理处理器对象
    break;
  default:
    # 默认情况下，断言不应该发生
    assert(0);
    break;
  }
}

# 获取下一个索引值
static unsigned
hwloc_synthetic_next_index(struct hwloc_synthetic_indexes_s *indexes, hwloc_obj_type_t type)
{
  # 获取下一个索引值
  unsigned os_index = indexes->next++;

  # 如果存在数组，则使用数组中的索引值
  if (indexes->array)
    os_index = indexes->array[os_index];
  # 对于缓存和组对象，不强制使用无用的os_indexes
  else if (hwloc__obj_type_is_cache(type) || type == HWLOC_OBJ_GROUP)
    os_index = HWLOC_UNKNOWN_INDEX;

  return os_index;
}

# 插入附加对象
static void
hwloc_synthetic_insert_attached(struct hwloc_topology *topology,
                struct hwloc_synthetic_backend_data_s *data,
                struct hwloc_synthetic_attached_s *attached,
                hwloc_bitmap_t set)
  # 定义变量 child，表示当前节点
  hwloc_obj_t child;
  # 定义变量 attached_os_index，表示附加的操作系统索引
  unsigned attached_os_index;

  # 如果没有附加节点，则直接返回
  if (!attached)
    return;

  # 断言附加节点的类型为 NUMA 节点
  assert(attached->attr.type == HWLOC_OBJ_NUMANODE);

  # 获取下一个可用的 NUMA 节点索引
  attached_os_index = hwloc_synthetic_next_index(&data->numa_attached_indexes, HWLOC_OBJ_NUMANODE);

  # 为当前节点分配并设置对象
  child = hwloc_alloc_setup_object(topology, attached->attr.type, attached_os_index);
  # 复制并设置当前节点的 cpuset
  child->cpuset = hwloc_bitmap_dup(set);

  # 分配并设置当前节点的 nodeset
  child->nodeset = hwloc_bitmap_alloc();
  hwloc_bitmap_set(child->nodeset, attached_os_index);

  # 设置附加节点的属性
  hwloc_synthetic_set_attr(&attached->attr, child);

  # 将当前节点插入拓扑结构中
  hwloc__insert_object_by_cpuset(topology, NULL, child, "synthetic:attached");

  # 递归插入附加节点
  hwloc_synthetic_insert_attached(topology, data, attached->next, set);
}

# 递归构建从指定 CPU 开始的对象
# - level 表示在 type、arity 和 id 数组中查找的位置
# - id 数组用作变量，用于为给定级别获取唯一的 ID
# - 生成的内存应添加到 *memory_kB
# - 生成的 CPU 应添加到 parent_cpuset
# - 返回下一个要使用的 CPU 编号
static void
hwloc__look_synthetic(struct hwloc_topology *topology,
              struct hwloc_synthetic_backend_data_s *data,
              int level,
              hwloc_bitmap_t parent_cpuset)
{
  hwloc_obj_t obj;
  unsigned i;
  struct hwloc_synthetic_level_data_s *curlevel = &data->level[level];
  hwloc_obj_type_t type = curlevel->attr.type;
  hwloc_bitmap_t set;
  unsigned os_index;

  # 断言对象类型为正常类型或 NUMA 节点
  assert(hwloc__obj_type_is_normal(type) || type == HWLOC_OBJ_NUMANODE);
  # 断言对象类型不为 MACHINE
  assert(type != HWLOC_OBJ_MACHINE);

  # 获取下一个可用的对象索引
  os_index = hwloc_synthetic_next_index(&curlevel->indexes, type);

  # 分配并设置当前节点的 cpuset
  set = hwloc_bitmap_alloc();
  if (!curlevel->arity) {
    hwloc_bitmap_set(set, os_index);
  } else {
    for (i = 0; i < curlevel->arity; i++)
      hwloc__look_synthetic(topology, data, level + 1, set);
  }

  # 将当前节点的 cpuset 与父节点的 cpuset合并
  hwloc_bitmap_or(parent_cpuset, parent_cpuset, set);

  # 检查并保留对象类型
  if (hwloc_filter_check_keep_object_type(topology, type)) {
    # 为当前节点分配并设置对象
    obj = hwloc_alloc_setup_object(topology, type, os_index);
    # 复制给定的 CPU 集合到对象的 cpuset 中
    obj->cpuset = hwloc_bitmap_dup(set);

    # 如果对象类型是 NUMA 节点，为对象的 nodeset 分配内存，并设置对应的 NUMA 节点索引
    if (type == HWLOC_OBJ_NUMANODE) {
      obj->nodeset = hwloc_bitmap_alloc();
      hwloc_bitmap_set(obj->nodeset, os_index);
    }

    # 设置对象的属性
    hwloc_synthetic_set_attr(&curlevel->attr, obj);

    # 将对象插入拓扑结构中
    hwloc__insert_object_by_cpuset(topology, NULL, obj, "synthetic");
  }

  # 在拓扑结构中插入附加的合成数据
  hwloc_synthetic_insert_attached(topology, data, curlevel->attached, set);

  # 释放 CPU 集合的内存
  hwloc_bitmap_free(set);
# 查找合成拓扑结构的函数
static int
hwloc_look_synthetic(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  # 获取拓扑结构和私有数据
  struct hwloc_topology *topology = backend->topology;
  struct hwloc_synthetic_backend_data_s *data = backend->private_data;
  # 分配一个位图
  hwloc_bitmap_t cpuset = hwloc_bitmap_alloc();
  unsigned i;

  # 确保处于全局发现阶段
  assert(dstatus->phase == HWLOC_DISC_PHASE_GLOBAL);

  # 确保顶层对象没有 cpuset
  assert(!topology->levels[0][0]->cpuset);

  # 为顶层对象分配根集合
  hwloc_alloc_root_sets(topology->levels[0][0]);

  # 设置支持的发现功能
  topology->support.discovery->pu = 1;
  topology->support.discovery->numa = 1;  # 如果没有给定 NUMA 节点，则添加一个单独的 NUMA 节点
  topology->support.discovery->numa_memory = 1;  # 指定或使用默认大小

  # 对每个层级的 os_index 进行初始化
  for (i = 0; data->level[i].arity > 0; i++)
    data->level[i].indexes.next = 0;
  data->numa_attached_indexes.next = 0;
  # 包括最后一个层级
  data->level[i].indexes.next = 0;

  # 根据合成类型数组更新第一个层级的类型
  topology->levels[0][0]->type = data->level[0].attr.type;
  hwloc_synthetic_set_attr(&data->level[0].attr, topology->levels[0][0]);

  # 对第一个层级的每个对象进行合成
  for (i = 0; i < data->level[0].arity; i++)
    hwloc__look_synthetic(topology, data, 1, cpuset);

  # 插入附加的对象
  hwloc_synthetic_insert_attached(topology, data, data->level[0].attached, cpuset);

  # 释放位图
  hwloc_bitmap_free(cpuset);

  # 为顶层对象添加信息
  hwloc_obj_add_info(topology->levels[0][0], "Backend", "Synthetic");
  hwloc_obj_add_info(topology->levels[0][0], "SyntheticDescription", data->string);
  return 0;
}

# 禁用合成拓扑结构的函数
static void
hwloc_synthetic_backend_disable(struct hwloc_backend *backend)
{
  # 释放层级和私有数据
  struct hwloc_synthetic_backend_data_s *data = backend->private_data;
  hwloc_synthetic_free_levels(data);
  free(data->string);
  free(data);
}
# 定义一个函数，用于实例化一个合成的组件
hwloc_synthetic_component_instantiate(struct hwloc_topology *topology,
                      struct hwloc_disc_component *component,
                      unsigned excluded_phases __hwloc_attribute_unused,
                      const void *_data1,
                      const void *_data2 __hwloc_attribute_unused,
                      const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;  // 定义一个指向 hwloc_backend 结构的指针
  struct hwloc_synthetic_backend_data_s *data;  // 定义一个指向 hwloc_synthetic_backend_data_s 结构的指针
  int err;  // 定义一个整型变量用于存储错误码

  if (!_data1) {  // 如果 _data1 为空
    const char *env = getenv("HWLOC_SYNTHETIC");  // 从环境变量中获取 HWLOC_SYNTHETIC 的值
    if (env) {  // 如果环境变量中有值
      /* 'synthetic' was given in HWLOC_COMPONENTS without a description */
      _data1 = env;  // 将环境变量的值赋给 _data1
    } else {  // 如果环境变量中没有值
      errno = EINVAL;  // 设置错误码为无效参数
      goto out;  // 跳转到 out 标签处
    }
  }

  backend = hwloc_backend_alloc(topology, component);  // 分配并初始化一个 hwloc_backend 结构
  if (!backend)  // 如果分配失败
    goto out;  // 跳转到 out 标签处

  data = malloc(sizeof(*data));  // 分配内存用于存储合成后端数据
  if (!data) {  // 如果分配失败
    errno = ENOMEM;  // 设置错误码为内存不足
    goto out_with_backend;  // 跳转到 out_with_backend 标签处
  }

  err = hwloc_backend_synthetic_init(data, (const char *) _data1);  // 初始化合成后端数据
  if (err < 0)  // 如果初始化失败
    goto out_with_data;  // 跳转到 out_with_data 标签处

  backend->private_data = data;  // 将合成后端数据赋给 backend 的 private_data 成员
  backend->discover = hwloc_look_synthetic;  // 设置 discover 函数为 hwloc_look_synthetic
  backend->disable = hwloc_synthetic_backend_disable;  // 设置 disable 函数为 hwloc_synthetic_backend_disable
  backend->is_thissystem = 0;  // 设置 is_thissystem 为 0

  return backend;  // 返回 backend 结构

 out_with_data:
  free(data);  // 释放合成后端数据的内存
 out_with_backend:
  free(backend);  // 释放 backend 结构的内存
 out:
  return NULL;  // 返回空指针
}

static struct hwloc_disc_component hwloc_synthetic_disc_component = {
  "synthetic",  // 组件名称为 "synthetic"
  HWLOC_DISC_PHASE_GLOBAL,  // 全局发现阶段
  ~0,  // 排除所有位
  hwloc_synthetic_component_instantiate,  // 实例化合成组件的函数
  30,  // 优先级为 30
  1,  // 只能有一个实例
  NULL  // 没有额外数据
};

const struct hwloc_component hwloc_synthetic_component = {
  HWLOC_COMPONENT_ABI,  // 组件的 ABI 版本
  NULL, NULL,  // 没有初始化和销毁函数
  HWLOC_COMPONENT_TYPE_DISC,  // 组件类型为发现
  0,  // 没有特定的数据
  &hwloc_synthetic_disc_component  // 指向合成组件的指针
};

static __hwloc_inline int
hwloc__export_synthetic_update_status(int *ret, char **tmp, ssize_t *tmplen, int res)
{
  if (res < 0)  // 如果 res 小于 0
    return -1;  // 返回 -1
  *ret += res;  // 将 res 加到 ret 上
  if (res >= *tmplen)  // 如果 res 大于等于 tmplen
    res = *tmplen>0 ? (int)(*tmplen) - 1 : 0;  // 如果 tmplen 大于 0，将 res 设置为 tmplen - 1，否则设置为 0
  *tmp += res;  // 将 tmp 增加 res
  *tmplen -= res;  // 将 tmplen 减去 res
  return 0;  // 返回 0
}

static __hwloc_inline void
# 向字符指针中添加一个字符，并更新指针位置和长度
hwloc__export_synthetic_add_char(int *ret, char **tmp, ssize_t *tmplen, char c)
{
  if (*tmplen > 1):  # 检查缓冲区长度是否大于1
    (*tmp)[0] = c;  # 将字符添加到缓冲区中
    (*tmp)[1] = '\0';  # 添加字符串结束符
    (*tmp)++;  # 更新指针位置
    (*tmplen)--;  # 更新长度
  (*ret)++;  # 更新返回值
}

# 导出合成索引
static int
hwloc__export_synthetic_indexes(hwloc_obj_t *level, unsigned total,
                char *buffer, size_t buflen)
{
  unsigned step = 1;  # 初始化步长
  unsigned nr_loops = 0;  # 初始化循环次数
  struct hwloc_synthetic_intlv_loop_s *loops = NULL, *tmploops;  # 初始化循环结构体指针
  hwloc_obj_t cur;  # 初始化对象指针
  unsigned i, j;  # 初始化循环变量
  ssize_t tmplen = buflen;  # 初始化缓冲区长度
  char *tmp = buffer;  # 初始化缓冲区指针
  int res, ret = 0;  # 初始化结果和返回值

  # 必须以0开始
  if (level[0]->os_index)  # 检查第一个对象的索引是否为0
    goto exportall;  # 转到导出所有的标签

  while (step != total):  # 循环直到步长等于总数
    # 必须是总数的除数
    if (total % step)  # 检查总数是否能被步长整除
      goto exportall;  # 转到导出所有的标签

    # 查找 os_index == step
    for(i=1; i<total; i++):  # 遍历对象数组
      if (level[i]->os_index == step)  # 检查对象的索引是否等于步长
        break;  # 退出循环
    if (i == total)  # 如果没有找到匹配的索引
      goto exportall;  # 转到导出所有的标签
    for(j=2; j<total/i; j++):  # 遍历对象数组
      if (level[i*j]->os_index != step*j)  # 检查对象的索引是否符合条件
        break;  # 退出循环

    nr_loops++;  # 更新循环次数
    tmploops = realloc(loops, nr_loops*sizeof(*loops));  # 重新分配循环结构体内存
    if (!tmploops)  # 检查内存分配是否成功
      goto exportall;  # 转到导出所有的标签
    loops = tmploops;  # 更新循环结构体指针
    loops[nr_loops-1].step = i;  # 更新循环步长
    loops[nr_loops-1].nb = j;  # 更新循环次数
    step *= j;  # 更新步长

  # 检查这种交错
  for(i=0; i<total; i++):  # 遍历对象数组
    unsigned ind = 0;  # 初始化索引
    unsigned mul = 1;  # 初始化乘数
    for(j=0; j<nr_loops; j++):  # 遍历循环结构体数组
      ind += (i / loops[j].step) % loops[j].nb * mul;  # 计算索引
      mul *= loops[j].nb;  # 更新乘数
    if (level[i]->os_index != ind)  # 检查对象的索引是否符合条件
      goto exportall;  # 转到导出所有的标签

  # 成功，打印它
  for(j=0; j<nr_loops; j++):  # 遍历循环结构体数组
    res = hwloc_snprintf(tmp, tmplen, "%u*%u%s", loops[j].step, loops[j].nb,
             j == nr_loops-1 ? ")" : ":");  # 格式化字符串
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0):  # 更新状态
      free(loops);  # 释放内存
      return -1;  # 返回错误
  free(loops);  # 释放内存
  return ret;  # 返回结果

 exportall:  # 导出所有的标签
  free(loops);  # 释放内存

  # 转储所有索引
  cur = level[0];  # 初始化当前对象
  while (cur):  # 循环直到当前对象为空
    res = hwloc_snprintf(tmp, tmplen, "%u%s", cur->os_index,
             cur->next_cousin ? "," : ")");  # 格式化字符串
    # 如果导出合成更新状态失败，则返回-1
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
    # 将当前节点指向下一个兄弟节点
    cur = cur->next_cousin;
  }
  # 返回结果
  return ret;
}
# 导出合成对象属性
static int
hwloc__export_synthetic_obj_attr(struct hwloc_topology * topology,
                 hwloc_obj_t obj,
                 char *buffer, size_t buflen)
{
  const char * separator = " ";  # 定义分隔符
  const char * prefix = "(";  # 定义前缀
  char cachesize[64] = "";  # 缓存大小
  char memsize[64] = "";  # 内存大小
  int needindexes = 0;  # 是否需要索引

  if (hwloc__obj_type_is_cache(obj->type) && obj->attr->cache.size) {  # 如果是缓存对象且有缓存大小
    snprintf(cachesize, sizeof(cachesize), "%ssize=%llu",  # 格式化缓存大小
         prefix, (unsigned long long) obj->attr->cache.size);
    prefix = separator;  # 更新前缀为分隔符
  }
  if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory) {  # 如果是NUMA节点且有本地内存大小
    snprintf(memsize, sizeof(memsize), "%smemory=%llu",  # 格式化内存大小
         prefix, (unsigned long long) obj->attr->numanode.local_memory);
    prefix = separator;  # 更新前缀为分隔符
  }
  if (!obj->logical_index /* only display indexes once per level (not for non-first NUMA children, etc.) */
      && (obj->type == HWLOC_OBJ_PU || obj->type == HWLOC_OBJ_NUMANODE)) {  # 如果不是逻辑索引且是处理器或NUMA节点
    hwloc_obj_t cur = obj;  # 当前对象为obj
    while (cur) {  # 循环直到cur为NULL
      if (cur->os_index != cur->logical_index) {  # 如果操作系统索引不等于逻辑索引
    needindexes = 1;  # 设置需要索引为1
    break;  # 跳出循环
      }
      cur = cur->next_cousin;  # 更新cur为下一个兄弟
    }
  }
  if (*cachesize || *memsize || needindexes) {  # 如果缓存大小、内存大小或需要索引
    ssize_t tmplen = buflen;  # 设置tmplen为buflen
    char *tmp = buffer;  # 设置tmp为buffer
    int res, ret = 0;  # 定义res和ret

    res = hwloc_snprintf(tmp, tmplen, "%s%s%s", cachesize, memsize, needindexes ? "" : ")");  # 格式化缓存大小、内存大小和是否需要索引
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)  # 更新合成状态
      return -1;  # 返回-1

    if (needindexes) {  # 如果需要索引
      unsigned total;  # 总数
      hwloc_obj_t *level;  # 层级对象数组

      if (obj->depth < 0) {  # 如果对象深度小于0
    assert(obj->depth == HWLOC_TYPE_DEPTH_NUMANODE);  # 断言对象深度为NUMA节点
    total = topology->slevels[HWLOC_SLEVEL_NUMANODE].nbobjs;  # 设置总数为NUMA节点的对象数
    level = topology->slevels[HWLOC_SLEVEL_NUMANODE].objs;  # 设置层级对象数组为NUMA节点的对象数组
      } else {  # 否则
    total = topology->level_nbobjects[obj->depth];  # 设置总数为对象深度的对象数
    level = topology->levels[obj->depth];  # 设置层级对象数组为对象深度的对象数组
      }

      res = hwloc_snprintf(tmp, tmplen, "%sindexes=", prefix);  # 格式化索引
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)  # 更新合成状态
    # 返回值为-1
    return -1;

      # 调用hwloc__export_synthetic_indexes函数，并将结果赋给res
      res = hwloc__export_synthetic_indexes(level, total, tmp, tmplen);
      # 调用hwloc__export_synthetic_update_status函数更新状态，并检查是否小于0
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
    # 返回值为-1
    return -1;
    }
    # 返回ret的值
    return ret;
  } else {
    # 返回值为0
    return 0;
  }
# 导出合成对象到字符串，包括对象类型和属性
static int
hwloc__export_synthetic_obj(struct hwloc_topology * topology, unsigned long flags,
                hwloc_obj_t obj, unsigned arity,
                char *buffer, size_t buflen)
{
  char aritys[12] = "";  # 初始化一个空字符串，用于存储对象的度
  ssize_t tmplen = buflen;  # 初始化一个变量，用于存储缓冲区的长度
  char *tmp = buffer;  # 初始化一个指针，指向缓冲区的起始位置
  int res, ret = 0;  # 初始化两个整型变量，用于存储结果和返回值

  # <type>:<arity>, except for root，根据对象的类型和度生成字符串，除了根节点
  if (arity != (unsigned)-1)
    snprintf(aritys, sizeof(aritys), ":%u", arity);  # 如果对象的度不是-1，则将其转换为字符串格式并存储到 aritys 中
  if (hwloc__obj_type_is_cache(obj->type)
      && (flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES)) {
    # v1 uses generic "Cache" for non-extended type name，如果对象是缓存类型且不包含扩展类型，则使用通用的“Cache”名称
    res = hwloc_snprintf(tmp, tmplen, "Cache%s", aritys);

  } else if (obj->type == HWLOC_OBJ_PACKAGE
         && (flags & (HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
              |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1))) {
    # if exporting to v1 or without extended-types, use all-v1-compatible Socket name，如果导出到v1或者不包含扩展类型，则使用兼容v1的Socket名称

    res = hwloc_snprintf(tmp, tmplen, "Socket%s", aritys);

  } else if (obj->type == HWLOC_OBJ_DIE
         && (flags & (HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
              |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1))) {
    # if exporting to v1 or without extended-types, use all-v1-compatible Group name，如果导出到v1或者不包含扩展类型，则使用兼容v1的Group名称

    res = hwloc_snprintf(tmp, tmplen, "Group%s", aritys);

  } else if (obj->type == HWLOC_OBJ_GROUP /* don't export group depth */
      || flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES) {
    res = hwloc_snprintf(tmp, tmplen, "%s%s", hwloc_obj_type_string(obj->type), aritys);  # 如果对象是组类型或者不包含扩展类型，则使用对象类型和度生成字符串
  } else {
    char types[64];
    hwloc_obj_type_snprintf(types, sizeof(types), obj, 1);
    res = hwloc_snprintf(tmp, tmplen, "%s%s", types, aritys);  # 否则，使用对象类型和度生成字符串
  }
  if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
    return -1;  # 如果更新状态失败，则返回-1

  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)) {
    # obj attributes，导出对象的属性
    res = hwloc__export_synthetic_obj_attr(topology, obj, tmp, tmplen);
    # 如果调用hwloc__export_synthetic_update_status函数返回值小于0，则返回-1
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }
  # 返回ret的值
  return ret;
  # 导出合成内存子节点
static int
hwloc__export_synthetic_memory_children(struct hwloc_topology * topology, unsigned long flags,
                    hwloc_obj_t parent,
                    char *buffer, size_t buflen,
                    int needprefix, int verbose)
{
  hwloc_obj_t mchild;  # 定义内存子节点对象
  ssize_t tmplen = buflen;  # 定义缓冲区长度
  char *tmp = buffer;  # 定义缓冲区指针
  int res, ret = 0;  # 定义结果和返回值

  mchild = parent->memory_first_child;  # 获取父节点的第一个内存子节点
  if (!mchild)  # 如果没有内存子节点
    return 0;  # 返回0

  if (flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1) {  # 如果标志包含v1
    /* v1: export a single NUMA child */
    if (parent->memory_arity > 1 || mchild->type != HWLOC_OBJ_NUMANODE) {  # 如果父节点的内存度大于1或者内存子节点的类型不是NUMA节点
      /* not supported */
      if (verbose)  # 如果详细模式开启
    fprintf(stderr, "Cannot export to synthetic v1 if multiple memory children are attached to the same location.\n");  # 输出错误信息
      errno = EINVAL;  # 设置错误码为无效参数
      return -1;  # 返回-1
    }

    if (needprefix)  # 如果需要前缀
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');  # 添加空格字符

    res = hwloc__export_synthetic_obj(topology, flags, mchild, 1, tmp, tmplen);  # 导出对象
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)  # 更新状态
      return -1;  # 返回-1
    return ret;  # 返回结果
  }

  while (mchild) {  # 循环遍历内存子节点
    /* FIXME: really recurse to export memcaches and numanode,
     * but it requires clever parsing of [ memcache [numa] [numa] ] during import,
     * better attaching of things to describe the hierarchy.
     */
    hwloc_obj_t numanode = mchild;  # 定义NUMA节点对象
    /* only export the first NUMA node leaf of each memory child
     * FIXME: This assumes mscache aren't shared between nodes, that's true in current platforms
     */
    while (numanode && numanode->type != HWLOC_OBJ_NUMANODE) {  # 循环遍历直到找到NUMA节点
      assert(numanode->arity == 1);  # 断言NUMA节点的度为1
      numanode = numanode->memory_first_child;  # 获取内存子节点的第一个子节点
    }
    assert(numanode); /* there's always a numanode at the bottom of the memory tree */  # 断言总是存在NUMA节点在内存树的底部

    if (needprefix)  # 如果需要前缀
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');  # 添加空格字符

    hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, '[');  # 添加左括号字符

    res = hwloc__export_synthetic_obj(topology, flags, numanode, (unsigned)-1, tmp, tmplen);  # 导出对象
    # 如果导出合成更新状态失败，则返回-1
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;

    # 向导出合成中添加字符']'
    hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ']');

    # 设置需要前缀为1
    needprefix = 1;
    # 移动到下一个兄弟节点
    mchild = mchild->next_sibling;
  }

  # 返回结果
  return ret;
}

static int
hwloc_check_memory_symmetric(struct hwloc_topology * topology)
{
  // 创建一个剩余节点的位图，初始值为整个系统的节点集合
  hwloc_bitmap_t remaining_nodes;
  remaining_nodes = hwloc_bitmap_dup(hwloc_get_root_obj(topology)->nodeset);
  // 如果剩余节点为空，说明内存不对称，返回-1
  if (!remaining_nodes)
    /* assume asymmetric */
    return -1;

  // 当剩余节点不为空时，进行循环
  while (!hwloc_bitmap_iszero(remaining_nodes)) {
    unsigned idx;
    hwloc_obj_t node;
    hwloc_obj_t first_parent;
    unsigned i;

    // 获取剩余节点中的第一个节点的索引
    idx = hwloc_bitmap_first(remaining_nodes);
    // 根据索引获取对应的 NUMA 节点对象
    node = hwloc_get_numanode_obj_by_os_index(topology, idx);
    assert(node);

    // 获取该节点的父节点
    first_parent = node->parent;

    // 检查父节点的所有对象是否具有相同数量的 NUMA 位
    for(i=0; i<hwloc_get_nbobjs_by_depth(topology, first_parent->depth); i++) {
      hwloc_obj_t parent, mchild;

      // 获取指定深度和索引的对象
      parent = hwloc_get_obj_by_depth(topology, first_parent->depth, i);
      assert(parent);

      // 必须具有相同的内存度
      if (parent->memory_arity != first_parent->memory_arity)
    // 如果不相同，跳转到释放位图的部分
    goto out_with_bitmap;

      // 从剩余节点中清除子节点的 NUMA 位
      mchild = parent->memory_first_child;
      while (mchild) {
    hwloc_bitmap_clr(remaining_nodes, mchild->os_index); /* cannot use parent->nodeset, some normal children may have other NUMA nodes */
    mchild = mchild->next_sibling;
      }
    }
  }

  // 释放剩余节点的位图
  hwloc_bitmap_free(remaining_nodes);
  return 0;

 out_with_bitmap:
  // 释放剩余节点的位图
  hwloc_bitmap_free(remaining_nodes);
  return -1;
}

int
hwloc_topology_export_synthetic(struct hwloc_topology * topology,
                char *buffer, size_t buflen,
                unsigned long flags)
{
  // 获取根对象
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  ssize_t tmplen = buflen;
  char *tmp = buffer;
  int res, ret = 0;
  unsigned arity;
  int needprefix = 0;
  int verbose = 0;
  const char *env = getenv("HWLOC_SYNTHETIC_VERBOSE");

  // 如果环境变量中存在 HWLOC_SYNTHETIC_VERBOSE，则将其转换为整数并赋值给 verbose
  if (env)
    verbose = atoi(env);

  // 如果拓扑结构未加载，设置错误码为 EINVAL
  if (!topology->is_loaded) {
    errno = EINVAL;
    # 返回错误代码 -1
    return -1;
  }

  # 检查 flags 是否包含非法标志位，如果有则设置 errno 为 EINVAL 并返回错误代码 -1
  if (flags & ~(HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
        |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS
        |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1
        |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
    errno = EINVAL;
    return -1;
  }

  # TODO: 添加一个标志位来忽略 symmetric_subtree 和 I/Os。
  # 假设树的左分支是对称的。
  # 但每个层级的对象数量可能是错误的，这种情况下怎么处理 OS 索引数组？
  # 只有在层级宽度保持正常的情况下才允许忽略 symmetric_subtree？
  
  # TODO: 默认添加一个根对象，带有类似 tree= 的前缀，以便我们可以向后兼容地识别是否有根对象。
  # 并添加一个标志位来禁用它。
  
  # TODO: 强制所有索引都生效的标志位，不仅仅是 PU 和 NUMA？
  
  # 如果对象的 symmetric_subtree 属性为假，则打印错误信息并返回错误代码 -1
  if (!obj->symmetric_subtree) {
    if (verbose)
      fprintf(stderr, "Cannot export to synthetic unless topology is symmetric (root->symmetric_subtree must be set).\n");
    errno = EINVAL;
    return -1;
  }

  # 如果 flags 不包含 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY
  # 并且内存不对称，则打印错误信息并返回错误代码 -1
  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)
      && hwloc_check_memory_symmetric(topology) < 0) {
    if (verbose)
      fprintf(stderr, "Cannot export to synthetic unless memory is attached symmetrically.\n");
    errno = EINVAL;
    return -1;
  }

  # 如果 flags 包含 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1
  # 则要求所有 NUMA 在同一层级
  hwloc_obj_t node;
  signed pdepth;

  node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, 0);
  assert(node);
  assert(hwloc__obj_type_is_normal(node->parent->type)); # 只有深度为 1 的内存子对象
  pdepth = node->parent->depth;

  while ((node = node->next_cousin) != NULL) {
    assert(hwloc__obj_type_is_normal(node->parent->type)); # 只有深度为 1 的内存子对象
    if (node->parent->depth != pdepth) {
    # 如果 verbose 为真，则向标准错误流输出一条消息
    if (verbose)
      fprintf(stderr, "Cannot export to synthetic v1 if memory is attached to parents at different depths.\n");
    # 设置 errno 为无效参数错误代码
    errno = EINVAL;
    # 返回 -1
    return -1;
      }
    }
  }

  /* 开始导出 */

  # 如果标志中不包含 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS
  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)) {
    # 导出对象属性
    res = hwloc__export_synthetic_obj_attr(topology, obj, tmp, tmplen);
    # 如果结果大于 0，则需要添加前缀
    if (res > 0)
      needprefix = 1;
    # 更新导出状态
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }

  # 如果标志中不包含 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY
  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
    # 导出内存子节点
    res = hwloc__export_synthetic_memory_children(topology, flags, obj, tmp, tmplen, needprefix, verbose);
    # 如果结果大于 0，则需要添加前缀
    if (res > 0)
      needprefix = 1;
    # 更新导出状态
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }

  # 获取对象的度
  arity = obj->arity;
  # 循环处理每个级别
  while (arity) {
    # 对于每个级别
    obj = obj->first_child;

    # 如果需要添加前缀
    if (needprefix)
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');

    # 导出对象
    res = hwloc__export_synthetic_obj(topology, flags, obj, arity, tmp, tmplen);
    # 更新导出状态
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;

    # 如果标志中不包含 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY
    if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
      # 导出内存子节点
      res = hwloc__export_synthetic_memory_children(topology, flags, obj, tmp, tmplen, 1, verbose);
      # 更新导出状态
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
    return -1;
    }

    # 下一个级别
    needprefix = 1;
    arity = obj->arity;
  }

  # 返回结果
  return ret;
# 闭合前面的函数定义
```