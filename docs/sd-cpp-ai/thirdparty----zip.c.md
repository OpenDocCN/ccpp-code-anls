# `stable-diffusion.cpp\thirdparty\zip.c`

```
/*
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR
 * OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
 * ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
 * OTHER DEALINGS IN THE SOFTWARE.
 */
*/

#define __STDC_WANT_LIB_EXT1__ 1

#include <errno.h>
#include <sys/stat.h>
#include <time.h>

#if defined(_WIN32) || defined(__WIN32__) || defined(_MSC_VER) ||              \
    defined(__MINGW32__)
/* Win32, DOS, MSVC, MSVS */
#include <direct.h>

#define STRCLONE(STR) ((STR) ? _strdup(STR) : NULL)
#define HAS_DEVICE(P)                                                          \
  ((((P)[0] >= 'A' && (P)[0] <= 'Z') || ((P)[0] >= 'a' && (P)[0] <= 'z')) &&   \
   (P)[1] == ':')
#define FILESYSTEM_PREFIX_LEN(P) (HAS_DEVICE(P) ? 2 : 0)

#else

#include <unistd.h> // needed for symlink()
#define STRCLONE(STR) ((STR) ? strdup(STR) : NULL)

#endif

#ifdef __MINGW32__
#include <sys/types.h>
#include <unistd.h>
#endif

#include "miniz.h"
#include "zip.h"

#ifdef _MSC_VER
#include <io.h>

#define ftruncate(fd, sz) (-(_chsize_s((fd), (sz)) != 0))
#define fileno _fileno
#endif

#if defined(__TINYC__) && (defined(_WIN32) || defined(_WIN64))
#include <io.h>

#define ftruncate(fd, sz) (-(_chsize_s((fd), (sz)) != 0))
#define fileno _fileno
#endif

#ifndef HAS_DEVICE
#define HAS_DEVICE(P) 0
#endif

#ifndef FILESYSTEM_PREFIX_LEN
#define FILESYSTEM_PREFIX_LEN(P) 0
#endif

#ifndef ISSLASH
#define ISSLASH(C) ((C) == '/' || (C) == '\\')
#endif

#define CLEANUP(ptr)                                                           \
  do {                                                                         \
    // 检查指针是否为空
    if (ptr) {                                                                 \
      // 释放指针指向的内存
      free((void *)ptr);                                                       \
      // 将指针置为 NULL
      ptr = NULL;                                                              \
    }                                                                          \
  } while (0)
# 定义 Unix 目录的权限标志
#define UNX_IFDIR 0040000  /* Unix directory */
# 定义 Unix 普通文件的权限标志
#define UNX_IFREG 0100000  /* Unix regular file */
# 定义 Unix 套接字的权限标志（BSD，非 SysV 或 Amiga）
#define UNX_IFSOCK 0140000 /* Unix socket (BSD, not SysV or Amiga) */
# 定义 Unix 符号链接的权限标志（非 SysV，Amiga）
#define UNX_IFLNK 0120000  /* Unix symbolic link (not SysV, Amiga) */
# 定义 Unix 块设备的权限标志（非 Amiga）
#define UNX_IFBLK 0060000  /* Unix block special       (not Amiga) */
# 定义 Unix 字符设备的权限标志（非 Amiga）
#define UNX_IFCHR 0020000  /* Unix character special   (not Amiga) */
# 定义 Unix FIFO 的权限标志（BCC，非 MSC 或 Amiga）
#define UNX_IFIFO 0010000  /* Unix fifo    (BCC, not MSC or Amiga) */

# 定义 ZIP 文件中每个条目的结构体
struct zip_entry_t {
  ssize_t index;
  char *name;
  mz_uint64 uncomp_size;
  mz_uint64 comp_size;
  mz_uint32 uncomp_crc32;
  mz_uint64 offset;
  mz_uint8 header[MZ_ZIP_LOCAL_DIR_HEADER_SIZE];
  mz_uint64 header_offset;
  mz_uint16 method;
  mz_zip_writer_add_state state;
  tdefl_compressor comp;
  mz_uint32 external_attr;
  time_t m_time;
};

# 定义 ZIP 文件的结构体
struct zip_t {
  mz_zip_archive archive;
  mz_uint level;
  struct zip_entry_t entry;
};

# 定义 ZIP 文件修改类型的枚举
enum zip_modify_t {
  MZ_KEEP = 0,
  MZ_DELETE = 1,
  MZ_MOVE = 2,
};

# 定义 ZIP 文件条目标记的结构体
struct zip_entry_mark_t {
  ssize_t file_index;
  enum zip_modify_t type;
  mz_uint64 m_local_header_ofs;
  size_t lf_length;
};

# 定义 ZIP 错误信息列表
static const char *const zip_errlist[33] = {
    NULL,
    "not initialized\0",
    "invalid entry name\0",
    "entry not found\0",
    "invalid zip mode\0",
    "invalid compression level\0",
    "no zip 64 support\0",
    "memset error\0",
    "cannot write data to entry\0",
    "cannot initialize tdefl compressor\0",
    "invalid index\0",
    "header not found\0",
    "cannot flush tdefl buffer\0",
    "cannot write entry header\0",
    "cannot create entry header\0",
    "cannot write to central dir\0",
    "cannot open file\0",
    "invalid entry type\0",
    "extracting data using no memory allocation\0",
    "file not found\0",
    "no permission\0",
    "out of memory\0",
    "invalid zip archive name\0",
    "make dir error\0",
    "symlink error\0",
    "close archive error\0",
    "capacity size too small\0",
    "fseek error\0",
    "fread error\0",
    "fwrite error\0",
    # 错误消息：无法初始化读取器
    "cannot initialize reader\0",
    # 错误消息：无法初始化写入器
    "cannot initialize writer\0",
    # 错误消息：无法从读取器初始化写入器
    "cannot initialize writer from reader\0",
};

// 返回 ZIP 错误信息
const char *zip_strerror(int errnum) {
  // 将错误号取反
  errnum = -errnum;
  // 如果错误号小于等于0或大于等于33，返回空
  if (errnum <= 0 || errnum >= 33) {
    return NULL;
  }

  // 返回对应错误号的错误信息
  return zip_errlist[errnum];
}

// 返回路径的基本名称
static const char *zip_basename(const char *name) {
  char const *p;
  char const *base = name += FILESYSTEM_PREFIX_LEN(name);
  int all_slashes = 1;

  // 遍历路径，找到最后一个斜杠后的内容作为基本名称
  for (p = name; *p; p++) {
    if (ISSLASH(*p))
      base = p + 1;
    else
      all_slashes = 0;
  }

  // 如果路径全是斜杠，则返回 '/'
  if (*base == '\0' && ISSLASH(*name) && all_slashes)
    --base;

  return base;
}

// 创建路径
static int zip_mkpath(char *path) {
  char *p;
  char npath[MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE + 1];
  int len = 0;
  int has_device = HAS_DEVICE(path);

  // 初始化 npath
  memset(npath, 0, MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE + 1);
  if (has_device) {
    // 只在 Windows 上执行
    npath[0] = path[0];
    npath[1] = path[1];
    len = 2;
  }
  // 遍历路径，创建目录
  for (p = path + len; *p && len < MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE; p++) {
    if (ISSLASH(*p) && ((!has_device && len > 0) || (has_device && len > 2))) {
      // 根据系统不同，将斜杠替换为 '/'
      if (MZ_MKDIR(npath) == -1) {
        if (errno != EEXIST) {
          return ZIP_EMKDIR;
        }
      }
    }
    npath[len++] = *p;
  }

  return 0;
}

// 替换字符串中的字符
static char *zip_strrpl(const char *str, size_t n, char oldchar, char newchar) {
  char c;
  size_t i;
  char *rpl = (char *)calloc((1 + n), sizeof(char));
  char *begin = rpl;
  if (!rpl) {
    return NULL;
  }

  // 遍历字符串，替换指定字符
  for (i = 0; (i < n) && (c = *str++); ++i) {
    if (c == oldchar) {
      c = newchar;
    }
    *rpl++ = c;
  }

  return begin;
}

// 标准化路径名
static char *zip_name_normalize(char *name, char *const nname, size_t len) {
  size_t offn = 0;
  size_t offnn = 0, ncpy = 0;

  if (name == NULL || nname == NULL || len <= 0) {
    return NULL;
  }
  // 跳过末尾的 '/'
  while (ISSLASH(*name))
    name++;

  for (; offn < len; offn++) {
    // 如果当前字符是斜杠，则进行处理
    if (ISSLASH(name[offn])) {
      // 如果当前字符不是"."或者".."，则将其添加到新的文件名中
      if (ncpy > 0 && strcmp(&nname[offnn], ".\0") &&
          strcmp(&nname[offnn], "..\0")) {
        offnn += ncpy;
        nname[offnn++] = name[offn]; // 在新文件名后追加'/'
      }
      ncpy = 0;
    } else {
      // 如果当前字符不是斜杠，则将其添加到新的文件名中
      nname[offnn + ncpy] = name[offn];
      ncpy++;
    }
  }

  // 在结束时，检查我们已经复制的内容
  // 如果最后一部分是"."或者".."，则将其截断
  if (ncpy == 0 || !strcmp(&nname[offnn], ".\0") ||
      !strcmp(&nname[offnn], "..\0")) {
    nname[offnn] = 0;
  }
  // 返回处理后的新文件名
  return nname;
# 检查 ZIP 文件名是否匹配的函数
static mz_bool zip_name_match(const char *name1, const char *name2) {
  char *nname2 = NULL;

  # 如果定义了 ZIP_RAW_ENTRYNAME 宏，则复制 name2 到 nname2
  # 否则，将 name2 中的 '\\' 替换为 '/'
#ifdef ZIP_RAW_ENTRYNAME
  nname2 = STRCLONE(name2);
#else
  nname2 = zip_strrpl(name2, strlen(name2), '\\', '/');
#endif

  # 如果 nname2 为空，则返回假
  if (!nname2) {
    return MZ_FALSE;
  }

  # 比较 name1 和 nname2 是否相等，返回比较结果
  mz_bool res = (strcmp(name1, nname2) == 0) ? MZ_TRUE : MZ_FALSE;
  # 清理 nname2 的内存
  CLEANUP(nname2);
  return res;
}

# 截断 ZIP 存档的函数
static int zip_archive_truncate(mz_zip_archive *pzip) {
  # 获取 ZIP 存档的内部状态
  mz_zip_internal_state *pState = pzip->m_pState;
  # 获取 ZIP 存档的文件大小
  mz_uint64 file_size = pzip->m_archive_size;
  # 如果使用堆写入函数并且存在内存，则返回 0
  if ((pzip->m_pWrite == mz_zip_heap_write_func) && (pState->m_pMem)) {
    return 0;
  }
  # 如果 ZIP 模式为已经完成写入，则执行以下操作
  if (pzip->m_zip_mode == MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED) {
    # 如果存在文件指针，则获取文件描述符并截断文件大小
    if (pState->m_pFile) {
      int fd = fileno(pState->m_pFile);
      return ftruncate(fd, file_size);
    }
  }
  return 0;
}

# 提取 ZIP 存档的函数
static int zip_archive_extract(mz_zip_archive *zip_archive, const char *dir,
                               int (*on_extract)(const char *filename,
                                                 void *arg),
                               void *arg) {
  int err = 0;
  mz_uint i, n;
  char path[MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE + 1];
  char symlink_to[MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE + 1];
  mz_zip_archive_file_stat info;
  size_t dirlen = 0, filename_size = MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE;
  mz_uint32 xattr = 0;

  # 初始化 path 和 symlink_to 数组
  memset(path, 0, sizeof(path));
  memset(symlink_to, 0, sizeof(symlink_to));

  # 获取目录名的长度
  dirlen = strlen(dir);
  # 如果目录名长度超过最大文件名长度，则返回错误
  if (dirlen + 1 > MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE) {
    return ZIP_EINVENTNAME;
  }

  # 初始化 info 结构体
  memset((void *)&info, 0, sizeof(mz_zip_archive_file_stat));

  # 根据操作系统设置路径分隔符
#if defined(_MSC_VER)
  strcpy_s(path, MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE, dir);
#else
  strcpy(path, dir);
#endif

  # 如果路径最后不是斜杠，则添加斜杠
  if (!ISSLASH(path[dirlen - 1])) {
#if defined(_WIN32) || defined(__WIN32__)
    path[dirlen] = '\\';
#else
    path[dirlen] = '/';
#endif
    ++dirlen;
  }

  # 如果文件名大小超过最大文件名长度减去目录名长度，则执行以下操作
    // 计算文件名大小，限制为最大文件名长度减去目录名长度
    filename_size = MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE - dirlen;
  }
  // 获取并打印存档中每个文件的信息
  n = mz_zip_reader_get_num_files(zip_archive);
  for (i = 0; i < n; ++i) {
    // 如果无法获取文件信息
    if (!mz_zip_reader_file_stat(zip_archive, i, &info)) {
      // 无法获取 zip 存档的信息
      err = ZIP_ENOENT;
      // 跳转到 out 标签处
      goto out;
    }

    // 如果无法规范化文件名
    if (!zip_name_normalize(info.m_filename, info.m_filename,
                            strlen(info.m_filename))) {
      // 无法规范化文件名
      err = ZIP_EINVENTNAME;
      // 跳转到 out 标签处
      goto out;
    }
#if defined(_MSC_VER)
    // 如果是在 Windows 平台下编译，使用安全的字符串拷贝函数
    strncpy_s(&path[dirlen], filename_size, info.m_filename, filename_size);
#else
    // 在其他平台下使用普通的字符串拷贝函数
    strncpy(&path[dirlen], info.m_filename, filename_size);
#endif
    // 创建路径
    err = zip_mkpath(path);
    if (err < 0) {
      // 无法创建路径，跳转到结束标签
      goto out;
    }

    if ((((info.m_version_made_by >> 8) == 3) ||
         ((info.m_version_made_by >> 8) ==
          19)) // 如果 ZIP 文件是在 Unix 或 macOS 上生成的
        && info.m_external_attr &
               (0x20 << 24)) { // 并且具有符号链接属性
#if defined(_WIN32) || defined(__WIN32__) || defined(_MSC_VER) ||              \
    defined(__MINGW32__)
#else
      if (info.m_uncomp_size > MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE ||
          !mz_zip_reader_extract_to_mem_no_alloc(
              zip_archive, i, symlink_to, MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE, 0,
              NULL, 0)) {
        err = ZIP_EMEMNOALLOC;
        goto out;
      }
      symlink_to[info.m_uncomp_size] = '\0';
      if (symlink(symlink_to, path) != 0) {
        err = ZIP_ESYMLINK;
        goto out;
      }
#endif
    } else {
      if (!mz_zip_reader_is_file_a_directory(zip_archive, i)) {
        if (!mz_zip_reader_extract_to_file(zip_archive, i, path, 0)) {
          // 无法将 ZIP 存档提取到文件
          err = ZIP_ENOFILE;
          goto out;
        }
      }

#if defined(_MSC_VER) || defined(PS4)
      (void)xattr; // 未使用
#else
      xattr = (info.m_external_attr >> 16) & 0xFFFF;
      if (xattr > 0 && xattr <= MZ_UINT16_MAX) {
        if (CHMOD(path, (mode_t)xattr) < 0) {
          err = ZIP_ENOPERM;
          goto out;
        }
      }
#endif
    }

    if (on_extract) {
      if (on_extract(path, arg) < 0) {
        goto out;
      }
    }
  }

out:
  // 关闭 ZIP 存档，释放任何正在使用的资源
  if (!mz_zip_reader_end(zip_archive)) {
    // 无法结束 ZIP 读取器
    # 将错误码设置为 ZIP_ECLSZIP
    err = ZIP_ECLSZIP;
  }
  # 返回错误码
  return err;
// 定义一个静态内联函数，用于完成 ZIP 归档的最终化操作
static inline void zip_archive_finalize(mz_zip_archive *pzip) {
  // 调用函数完成 ZIP 归档的最终化
  mz_zip_writer_finalize_archive(pzip);
  // 调用函数截断 ZIP 归档
  zip_archive_truncate(pzip);
}

// 标记 ZIP 文件中的条目，以便后续操作
static ssize_t zip_entry_mark(struct zip_t *zip,
                              struct zip_entry_mark_t *entry_mark,
                              const ssize_t n, char *const entries[],
                              const size_t len) {
  ssize_t i = 0;
  ssize_t err = 0;
  // 检查参数是否有效
  if (!zip || !entry_mark || !entries) {
    return ZIP_ENOINIT;
  }

  // 定义文件状态结构体和位置变量
  mz_zip_archive_file_stat file_stat;
  mz_uint64 d_pos = UINT64_MAX;
  // 遍历 ZIP 文件中的条目
  for (i = 0; i < n; ++i) {
    // 打开指定索引的 ZIP 条目
    if ((err = zip_entry_openbyindex(zip, i))) {
      return (ssize_t)err;
    }

    // 检查条目名称是否匹配
    mz_bool name_matches = MZ_FALSE;
    {
      size_t j;
      // 遍历条目名称数组，查找匹配的名称
      for (j = 0; j < len; ++j) {
        if (zip_name_match(zip->entry.name, entries[j])) {
          name_matches = MZ_TRUE;
          break;
        }
      }
    }
    // 根据名称匹配情况标记条目类型
    if (name_matches) {
      entry_mark[i].type = MZ_DELETE;
    } else {
      entry_mark[i].type = MZ_KEEP;
    }

    // 获取文件状态信息
    if (!mz_zip_reader_file_stat(&zip->archive, i, &file_stat)) {
      return ZIP_ENOENT;
    }

    // 关闭 ZIP 条目
    zip_entry_close(zip);

    // 更新条目标记信息
    entry_mark[i].m_local_header_ofs = file_stat.m_local_header_ofs;
    entry_mark[i].file_index = (ssize_t)-1;
    entry_mark[i].lf_length = 0;
    // 如果是删除类型的条目且位置小于当前最小位置，则更新最小位置
    if ((entry_mark[i].type) == MZ_DELETE &&
        (d_pos > entry_mark[i].m_local_header_ofs)) {
      d_pos = entry_mark[i].m_local_header_ofs;
    }
  }

  // 根据最小位置和条目类型更新条目标记信息
  for (i = 0; i < n; ++i) {
    if ((entry_mark[i].m_local_header_ofs > d_pos) &&
        (entry_mark[i].type != MZ_DELETE)) {
      entry_mark[i].type = MZ_MOVE;
    }
  }
  return err;
}

// 获取下一个 ZIP 索引
static ssize_t zip_index_next(mz_uint64 *local_header_ofs_array,
                              ssize_t cur_index) {
  ssize_t new_index = 0, i;
  // 从当前索引向前查找，找到下一个索引
  for (i = cur_index - 1; i >= 0; --i) {
    if (local_header_ofs_array[cur_index] > local_header_ofs_array[i]) {
      new_index = i + 1;
      return new_index;
    }
  }
  return new_index;
}
static ssize_t zip_sort(mz_uint64 *local_header_ofs_array, ssize_t cur_index) {
  // 找到下一个需要排序的索引
  ssize_t nxt_index = zip_index_next(local_header_ofs_array, cur_index);

  // 如果下一个索引不等于当前索引
  if (nxt_index != cur_index) {
    // 交换当前索引和下一个索引的值
    mz_uint64 temp = local_header_ofs_array[cur_index];
    ssize_t i;
    for (i = cur_index; i > nxt_index; i--) {
      local_header_ofs_array[i] = local_header_ofs_array[i - 1];
    }
    local_header_ofs_array[nxt_index] = temp;
  }
  return nxt_index;
}

static int zip_index_update(struct zip_entry_mark_t *entry_mark,
                            ssize_t last_index, ssize_t nxt_index) {
  ssize_t j;
  // 更新文件索引
  for (j = 0; j < last_index; j++) {
    if (entry_mark[j].file_index >= nxt_index) {
      entry_mark[j].file_index += 1;
    }
  }
  entry_mark[nxt_index].file_index = last_index;
  return 0;
}

static int zip_entry_finalize(struct zip_t *zip,
                              struct zip_entry_mark_t *entry_mark,
                              const ssize_t n) {

  ssize_t i = 0;
  // 分配本地头偏移数组的内存
  mz_uint64 *local_header_ofs_array = (mz_uint64 *)calloc(n, sizeof(mz_uint64));
  if (!local_header_ofs_array) {
    return ZIP_EOOMEM;
  }

  // 对每个条目进行排序
  for (i = 0; i < n; ++i) {
    local_header_ofs_array[i] = entry_mark[i].m_local_header_ofs;
    ssize_t index = zip_sort(local_header_ofs_array, i);

    // 如果索引不等于当前索引，更新索引
    if (index != i) {
      zip_index_update(entry_mark, i, index);
    }
    entry_mark[i].file_index = index;
  }

  // 分配长度数组的内存
  size_t *length = (size_t *)calloc(n, sizeof(size_t));
  if (!length) {
    CLEANUP(local_header_ofs_array);
    return ZIP_EOOMEM;
  }
  // 计算每个条目的长度
  for (i = 0; i < n - 1; i++) {
    length[i] =
        (size_t)(local_header_ofs_array[i + 1] - local_header_ofs_array[i]);
  }
  length[n - 1] =
      (size_t)(zip->archive.m_archive_size - local_header_ofs_array[n - 1]);

  // 更新每个条目的长度
  for (i = 0; i < n; i++) {
    entry_mark[i].lf_length = length[entry_mark[i].file_index];
  }

  CLEANUP(length);
  CLEANUP(local_header_ofs_array);
  return 0;
}
// 设置 ZIP 文件中指定条目的内容
static ssize_t zip_entry_set(struct zip_t *zip,
                             struct zip_entry_mark_t *entry_mark, ssize_t n,
                             char *const entries[], const size_t len) {
  ssize_t err = 0;

  // 标记 ZIP 文件中指定条目的位置
  if ((err = zip_entry_mark(zip, entry_mark, n, entries, len)) < 0) {
    return err;
  }
  // 完成 ZIP 文件中指定条目的写入
  if ((err = zip_entry_finalize(zip, entry_mark, n)) < 0) {
    return err;
  }
  return 0;
}

// 移动 ZIP 文件中的数据
static ssize_t zip_file_move(MZ_FILE *m_pFile, const mz_uint64 to,
                             const mz_uint64 from, const size_t length,
                             mz_uint8 *move_buf, const size_t capacity_size) {
  // 如果要移动的数据长度超过缓冲区容量，返回错误
  if (length > capacity_size) {
    return ZIP_ECAPSIZE;
  }
  // 将文件指针移动到指定位置
  if (MZ_FSEEK64(m_pFile, from, SEEK_SET)) {
    return ZIP_EFSEEK;
  }
  // 从文件中读取数据到缓冲区
  if (fread(move_buf, 1, length, m_pFile) != length) {
    return ZIP_EFREAD;
  }
  // 将缓冲区中的数据写入文件指定位置
  if (MZ_FSEEK64(m_pFile, to, SEEK_SET)) {
    return ZIP_EFSEEK;
  }
  // 将缓冲区中的数据写入文件
  if (fwrite(move_buf, 1, length, m_pFile) != length) {
    return ZIP_EFWRITE;
  }
  return (ssize_t)length;
}

// 移动多个 ZIP 文件中的数据
static ssize_t zip_files_move(MZ_FILE *m_pFile, mz_uint64 writen_num,
                              mz_uint64 read_num, size_t length) {
  ssize_t n = 0;
  const size_t page_size = 1 << 12; // 4K
  mz_uint8 *move_buf = (mz_uint8 *)calloc(1, page_size);
  if (!move_buf) {
    return ZIP_EOOMEM;
  }

  ssize_t moved_length = 0;
  ssize_t move_count = 0;
  while ((mz_int64)length > 0) {
    // 计算每次移动的数据长度
    move_count = (length >= page_size) ? page_size : length;
    // 调用单个文件移动函数进行数据移动
    n = zip_file_move(m_pFile, writen_num, read_num, move_count, move_buf,
                      page_size);
    if (n < 0) {
      moved_length = n;
      goto cleanup;
    }

    // 检查实际移动的数据长度是否与预期一致
    if (n != move_count) {
      goto cleanup;
    }

    // 更新写入和读取位置，以及剩余数据长度
    writen_num += move_count;
    read_num += move_count;
    length -= move_count;
    moved_length += move_count;
  }

cleanup:
  CLEANUP(move_buf); // 释放动态分配的内存
  return moved_length; // 返回实际移动的数据长度
}
// 移动 ZIP 中央目录的条目，删除指定索引的条目
static int zip_central_dir_move(mz_zip_internal_state *pState, int begin,
                                int end, int entry_num) {
  // 如果起始索引等于指定索引，则直接返回
  if (begin == entry_num) {
    return 0;
  }

  size_t l_size = 0; // 左侧大小
  size_t r_size = 0; // 右侧大小
  mz_uint32 d_size = 0; // 数据大小
  mz_uint8 *next = NULL; // 下一个指针
  mz_uint8 *deleted = &MZ_ZIP_ARRAY_ELEMENT(
      &pState->m_central_dir, mz_uint8,
      MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32, begin)); // 被删除的条目
  l_size = (size_t)(deleted - (mz_uint8 *)(pState->m_central_dir.m_p)); // 计算左侧大小
  // 如果结束索引等于指定索引，则右侧大小为0
  if (end == entry_num) {
    r_size = 0;
  } else {
    next = &MZ_ZIP_ARRAY_ELEMENT(
        &pState->m_central_dir, mz_uint8,
        MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32, end)); // 下一个指针
    r_size = pState->m_central_dir.m_size -
             (mz_uint32)(next - (mz_uint8 *)(pState->m_central_dir.m_p)); // 计算右侧大小
    d_size = (mz_uint32)(next - deleted); // 计算数据大小
  }

  // 如果下一个指针存在且左侧大小为0
  if (next && l_size == 0) {
    // 移动数据到左侧
    memmove(pState->m_central_dir.m_p, next, r_size);
    // 重新分配内存
    pState->m_central_dir.m_p = MZ_REALLOC(pState->m_central_dir.m_p, r_size);
    {
      int i;
      // 更新后续条目的偏移量
      for (i = end; i < entry_num; i++) {
        MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32, i) -=
            d_size;
      }
    }
  }

  // 如果下一个指针存在且左侧大小和右侧大小都不为0
  if (next && l_size * r_size != 0) {
    // 移动数据到左侧
    memmove(deleted, next, r_size);
    {
      int i;
      // 更新后续条目的偏移量
      for (i = end; i < entry_num; i++) {
        MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32, i) -=
            d_size;
      }
    }
  }

  // 更新 ZIP 中央目录的大小
  pState->m_central_dir.m_size = l_size + r_size;
  return 0;
}

// 删除 ZIP 中央目录的条目
static int zip_central_dir_delete(mz_zip_internal_state *pState,
                                  int *deleted_entry_index_array,
                                  int entry_num) {
  int i = 0;
  int begin = 0;
  int end = 0;
  int d_num = 0;
  // 遍历所有条目
  while (i < entry_num) {
    // 找到第一个未删除的条目
    while ((i < entry_num) && (!deleted_entry_index_array[i])) {
      i++;
    }
    begin = i; // 起始索引

    // 找到第一个被删除的条目
    while ((i < entry_num) && (deleted_entry_index_array[i])) {
      i++;
    }
    end = i; // 结束索引
    # 移动 ZIP 中央目录的位置
    zip_central_dir_move(pState, begin, end, entry_num);
  }

  # 初始化索引变量 i
  i = 0;
  # 循环直到遍历完所有条目
  while (i < entry_num) {
    # 找到第一个未删除的条目
    while ((i < entry_num) && (!deleted_entry_index_array[i])) {
      i++;
    }
    # 记录起始位置
    begin = i;
    # 如果起始位置等于总条目数，跳出循环
    if (begin == entry_num) {
      break;
    }
    # 找到下一个被删除的条目
    while ((i < entry_num) && (deleted_entry_index_array[i])) {
      i++;
    }
    # 记录结束位置
    end = i;
    # 初始化变量 k 和 j
    int k = 0, j;
    # 将被删除的条目后的条目向前移动
    for (j = end; j < entry_num; j++) {
      MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32,
                           begin + k) =
          (mz_uint32)MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets,
                                          mz_uint32, j);
      k++;
    }
    # 更新删除的条目数量
    d_num += end - begin;
  }

  # 更新中央目录偏移数组的大小
  pState->m_central_dir_offsets.m_size =
      sizeof(mz_uint32) * (entry_num - d_num);
  # 返回 0 表示成功
  return 0;
static ssize_t zip_entries_delete_mark(struct zip_t *zip,
                                       struct zip_entry_mark_t *entry_mark,
                                       int entry_num) {
  // 初始化写入字节数和读取字节数
  mz_uint64 writen_num = 0;
  mz_uint64 read_num = 0;
  // 初始化已删除长度和移动长度
  size_t deleted_length = 0;
  size_t move_length = 0;
  int i = 0;
  // 初始化已删除条目数和返回值
  size_t deleted_entry_num = 0;
  ssize_t n = 0;

  // 分配并初始化已删除条目标记数组
  mz_bool *deleted_entry_flag_array =
      (mz_bool *)calloc(entry_num, sizeof(mz_bool));
  if (deleted_entry_flag_array == NULL) {
    return ZIP_EOOMEM;
  }

  // 获取 ZIP 文件的内部状态
  mz_zip_internal_state *pState = zip->archive.m_pState;
  // 设置 ZIP 模式为写入模式
  zip->archive.m_zip_mode = MZ_ZIP_MODE_WRITING;

  // 检查文件指针是否存在，若不存在则返回错误
  if ((!pState->m_pFile) || MZ_FSEEK64(pState->m_pFile, 0, SEEK_SET)) {
    CLEANUP(deleted_entry_flag_array);
    return ZIP_ENOENT;
  }

  // 循环处理每个条目
  while (i < entry_num) {
    // 处理保留类型的条目
    while ((i < entry_num) && (entry_mark[i].type == MZ_KEEP)) {
      writen_num += entry_mark[i].lf_length;
      read_num = writen_num;
      i++;
    }

    // 处理删除类型的条目
    while ((i < entry_num) && (entry_mark[i].type == MZ_DELETE)) {
      deleted_entry_flag_array[i] = MZ_TRUE;
      read_num += entry_mark[i].lf_length;
      deleted_length += entry_mark[i].lf_length;
      i++;
      deleted_entry_num++;
    }

    // 处理移动类型的条目
    while ((i < entry_num) && (entry_mark[i].type == MZ_MOVE)) {
      move_length += entry_mark[i].lf_length;
      mz_uint8 *p = &MZ_ZIP_ARRAY_ELEMENT(
          &pState->m_central_dir, mz_uint8,
          MZ_ZIP_ARRAY_ELEMENT(&pState->m_central_dir_offsets, mz_uint32, i));
      if (!p) {
        CLEANUP(deleted_entry_flag_array);
        return ZIP_ENOENT;
      }
      mz_uint32 offset = MZ_READ_LE32(p + MZ_ZIP_CDH_LOCAL_HEADER_OFS);
      offset -= (mz_uint32)deleted_length;
      MZ_WRITE_LE32(p + MZ_ZIP_CDH_LOCAL_HEADER_OFS, offset);
      i++;
    }

    // 移动文件数据
    n = zip_files_move(pState->m_pFile, writen_num, read_num, move_length);
    if (n != (ssize_t)move_length) {
      CLEANUP(deleted_entry_flag_array);
      return n;
    }
    writen_num += move_length;
    read_num += move_length;
  }



  // 将读取的字节数增加移动长度，用于更新读取位置
  zip->archive.m_archive_size -= (mz_uint64)deleted_length;
  // 减去删除的长度，用于更新 ZIP 存档的大小
  zip->archive.m_total_files =
      (mz_uint32)entry_num - (mz_uint32)deleted_entry_num;
  // 更新 ZIP 存档中文件的总数，减去删除的文件数

  zip_central_dir_delete(pState, deleted_entry_flag_array, entry_num);
  // 调用函数删除 ZIP 中的中央目录项
  CLEANUP(deleted_entry_flag_array);
  // 清理删除的目录项标志数组

  return (ssize_t)deleted_entry_num;
  // 返回删除的目录项数目
// 打开一个 ZIP 文件，返回一个指向 zip_t 结构体的指针
struct zip_t *zip_open(const char *zipname, int level, char mode) {
  int errnum = 0;
  // 调用带错误码参数的 zip_openwitherror 函数
  return zip_openwitherror(zipname, level, mode, &errnum);
}

// 打开一个 ZIP 文件，返回一个指向 zip_t 结构体的指针，并设置错误码
struct zip_t *zip_openwitherror(const char *zipname, int level, char mode,
                                int *errnum) {
  struct zip_t *zip = NULL;
  *errnum = 0;

  // 检查 ZIP 文件名是否为空
  if (!zipname || strlen(zipname) < 1) {
    // ZIP 文件名为空或为 NULL
    *errnum = ZIP_EINVZIPNAME;
    goto cleanup;
  }

  // 检查压缩级别是否合法
  if (level < 0)
    level = MZ_DEFAULT_LEVEL;
  if ((level & 0xF) > MZ_UBER_COMPRESSION) {
    // 压缩级别错误
    *errnum = ZIP_EINVLVL;
    goto cleanup;
  }

  // 分配内存给 zip_t 结构体
  zip = (struct zip_t *)calloc((size_t)1, sizeof(struct zip_t));
  if (!zip) {
    // 内存不足
    *errnum = ZIP_EOOMEM;
    goto cleanup;
  }

  // 设置压缩级别
  zip->level = (mz_uint)level;
  switch (mode) {
  case 'w':
    // 创建一个新的归档文件
    if (!mz_zip_writer_init_file_v2(&(zip->archive), zipname, 0,
                                    MZ_ZIP_FLAG_WRITE_ZIP64)) {
      // 无法初始化 zip_archive 写入器
      *errnum = ZIP_EWINIT;
      goto cleanup;
    }
    break;

  case 'r':
    if (!mz_zip_reader_init_file_v2(
            &(zip->archive), zipname,
            zip->level | MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY, 0, 0)) {
      // 归档文件不存在或无法初始化 zip_archive 读取器
      *errnum = ZIP_ERINIT;
      goto cleanup;
    }
    break;

  case 'a':
  case 'd':
    if (!mz_zip_reader_init_file_v2_rpb(
            &(zip->archive), zipname,
            zip->level | MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY, 0, 0)) {
      // 归档文件不存在或无法初始化 zip_archive 读取器
      *errnum = ZIP_ERINIT;
      goto cleanup;
    }
    # 如果模式为 'a' 或 'd'，则执行以下操作
    if ((mode == 'a' || mode == 'd')) {
        # 从现有的 ZIP 读取器初始化 ZIP 写入器，不重新打开文件
        if (!mz_zip_writer_init_from_reader_v2_noreopen(&(zip->archive), zipname, 0)) {
            # 如果初始化失败，设置错误码为 ZIP_EWRINIT
            *errnum = ZIP_EWRINIT;
            # 结束 ZIP 读取器
            mz_zip_reader_end(&(zip->archive));
            # 跳转到清理代码块
            goto cleanup;
        }
    }
    # 结束当前的 switch 语句
    break;

  # 如果模式不是 'a' 或 'd'，执行以下操作
  default:
    # 设置错误码为 ZIP_EINVMODE
    *errnum = ZIP_EINVMODE;
    # 跳转到清理代码块
    goto cleanup;
  }

  # 返回 ZIP 对象
  return zip;
// 清理函数，释放 ZIP 对象资源并返回 NULL
cleanup:
  // 调用清理宏，释放 ZIP 对象资源
  CLEANUP(zip);
  // 返回 NULL
  return NULL;
}

// 关闭 ZIP 对象
void zip_close(struct zip_t *zip) {
  // 检查 ZIP 对象是否存在
  if (zip) {
    // 获取 ZIP 对象的内部结构指针
    mz_zip_archive *pZip = &(zip->archive);
    // 如果 ZIP 模式为写入，则始终进行最终化以确保有一个有效的中央目录
    if (pZip->m_zip_mode == MZ_ZIP_MODE_WRITING) {
      mz_zip_writer_finalize_archive(pZip);
    }

    // 如果 ZIP 模式为写入或已经完成写入，则截断 ZIP 文件并结束写入
    if (pZip->m_zip_mode == MZ_ZIP_MODE_WRITING ||
        pZip->m_zip_mode == MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED) {
      zip_archive_truncate(pZip);
      mz_zip_writer_end(pZip);
    }
    // 如果 ZIP 模式为读取，则结束读取
    if (pZip->m_zip_mode == MZ_ZIP_MODE_READING) {
      mz_zip_reader_end(pZip);
    }

    // 调用清理宏，释放 ZIP 对象资源
    CLEANUP(zip);
  }
}

// 检查 ZIP 对象是否为 64 位
int zip_is64(struct zip_t *zip) {
  // 如果 ZIP 对象不存在或 ZIP 状态未初始化，则返回错误码
  if (!zip || !zip->archive.m_pState) {
    return ZIP_ENOINIT;
  }

  // 返回 ZIP 对象是否为 64 位的标志
  return (int)zip->archive.m_pState->m_zip64;
}

// 打开 ZIP 条目
static int _zip_entry_open(struct zip_t *zip, const char *entryname,
                           int case_sensitive) {
  size_t entrylen = 0;
  mz_zip_archive *pzip = NULL;
  mz_uint num_alignment_padding_bytes, level;
  mz_zip_archive_file_stat stats;
  int err = 0;
  mz_uint16 dos_time = 0, dos_date = 0;
  mz_uint32 extra_size = 0;
  mz_uint8 extra_data[MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE];
  mz_uint64 local_dir_header_ofs = 0;

  // 如果 ZIP 对象不存在，则返回错误码
  if (!zip) {
    return ZIP_ENOINIT;
  }

  // 记录 ZIP 文件中局部目录头的偏移量
  local_dir_header_ofs = zip->archive.m_archive_size;

  // 如果条目名为空，则返回错误码
  if (!entryname) {
    return ZIP_EINVENTNAME;
  }

  // 获取条目名的长度
  entrylen = strlen(entryname);
  // 如果条目名长度为 0，则返回错误码
  if (entrylen == 0) {
    return ZIP_EINVENTNAME;
  }

  /*
    .ZIP 文件格式规范版本：6.3.3

    4.4.17.1 文件名，带有可选的相对路径。
    存储的路径不得包含驱动器或设备号，或前导斜杠。
    所有斜杠必须是正斜杠 '/'，而不是反斜杠 '\'，以便与 Amiga 和 UNIX 文件系统等兼容。
    如果输入来自标准输入，则没有文件名字段。
  */
  // 如果 ZIP 条目名存在
  if (zip->entry.name) {
    # 清理 ZIP 对象中的条目名称
    CLEANUP(zip->entry.name);
  }
#ifdef ZIP_RAW_ENTRYNAME
  // 如果定义了 ZIP_RAW_ENTRYNAME 宏，则直接复制 entryname 到 zip->entry.name
  zip->entry.name = STRCLONE(entryname);
#else
  // 如果未定义 ZIP_RAW_ENTRYNAME 宏，则将 entryname 中的 '\\' 替换为 '/'
  zip->entry.name = zip_strrpl(entryname, entrylen, '\\', '/');
#endif

  if (!zip->entry.name) {
    // 如果无法解析 ZIP 条目名称，则返回 ZIP_EINVENTNAME 错误
    return ZIP_EINVENTNAME;
  }

  pzip = &(zip->archive);
  if (pzip->m_zip_mode == MZ_ZIP_MODE_READING) {
    // 在 ZIP 读取模式下，定位文件名为 zip->entry.name 的文件
    zip->entry.index = (ssize_t)mz_zip_reader_locate_file(
        pzip, zip->entry.name, NULL,
        case_sensitive ? MZ_ZIP_FLAG_CASE_SENSITIVE : 0);
    if (zip->entry.index < (ssize_t)0) {
      // 如果无法定位文件，则返回 ZIP_ENOENT 错误
      err = ZIP_ENOENT;
      goto cleanup;
    }

    if (!mz_zip_reader_file_stat(pzip, (mz_uint)zip->entry.index, &stats)) {
      // 如果无法获取文件状态，则返回 ZIP_ENOENT 错误
      err = ZIP_ENOENT;
      goto cleanup;
    }

    // 设置 zip->entry 的各项属性
    zip->entry.comp_size = stats.m_comp_size;
    zip->entry.uncomp_size = stats.m_uncomp_size;
    zip->entry.uncomp_crc32 = stats.m_crc32;
    zip->entry.offset = stats.m_central_dir_ofs;
    zip->entry.header_offset = stats.m_local_header_ofs;
    zip->entry.method = stats.m_method;
    zip->entry.external_attr = stats.m_external_attr;
#ifndef MINIZ_NO_TIME
    zip->entry.m_time = stats.m_time;
#endif

    return 0;
  }

  level = zip->level & 0xF;

  // 设置 zip->entry 的各项属性
  zip->entry.index = (ssize_t)zip->archive.m_total_files;
  zip->entry.comp_size = 0;
  zip->entry.uncomp_size = 0;
  zip->entry.uncomp_crc32 = MZ_CRC32_INIT;
  zip->entry.offset = zip->archive.m_archive_size;
  zip->entry.header_offset = zip->archive.m_archive_size;
  memset(zip->entry.header, 0, MZ_ZIP_LOCAL_DIR_HEADER_SIZE * sizeof(mz_uint8));
  zip->entry.method = level ? MZ_DEFLATED : 0;

  // 根据平台设置 zip->entry 的 external_attr 属性
#if MZ_PLATFORM == 3 || MZ_PLATFORM == 19
  // UNIX 或 APPLE 平台下，设置文件权限为 rw-r--r--
  zip->entry.external_attr = (mz_uint32)(0100644) << 16;
#else
  zip->entry.external_attr = 0;
#endif

  // 计算需要的字节对齐填充
  num_alignment_padding_bytes =
      mz_zip_writer_compute_padding_needed_for_file_alignment(pzip);

  if (!pzip->m_pState || (pzip->m_zip_mode != MZ_ZIP_MODE_WRITING)) {
    // 如果 ZIP 模式无效，则返回 ZIP_EINVMODE 错误
    err = ZIP_EINVMODE;
    // 跳转到清理代码的位置
    goto cleanup;
  }
  // 如果 ZIP 对象的压缩级别包含压缩数据标志
  if (zip->level & MZ_ZIP_FLAG_COMPRESSED_DATA) {
    // 无效的 ZIP 压缩级别
    err = ZIP_EINVLVL;
    // 跳转到清理代码的位置
    goto cleanup;
  }

  // 如果无法使用 mz_zip_writer_write_zeros 函数对 ZIP 条目头部进行填充
  if (!mz_zip_writer_write_zeros(pzip, zip->entry.offset,
                                 num_alignment_padding_bytes)) {
    // 无法对 ZIP 条目头部进行填充
    err = ZIP_EMEMSET;
    // 跳转到清理代码的位置
    goto cleanup;
  }
  // 更新本地目录头偏移量
  local_dir_header_ofs += num_alignment_padding_bytes;

  // 设置 ZIP 条目的修改时间为当前时间
  zip->entry.m_time = time(NULL);
#ifndef MINIZ_NO_TIME
  // 如果没有定义 MINIZ_NO_TIME，则将 ZIP 文件的时间转换为 DOS 时间格式
  mz_zip_time_t_to_dos_time(zip->entry.m_time, &dos_time, &dos_date);
#endif

  // 创建 ZIP64 头部，包含 NULL 大小（大小将在数据描述符中，即文件数据后面）
  extra_size = mz_zip_writer_create_zip64_extra_data(
      extra_data, NULL, NULL,
      (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs : NULL);

  // 创建本地目录头部
  if (!mz_zip_writer_create_local_dir_header(
          pzip, zip->entry.header, entrylen, (mz_uint16)extra_size, 0, 0, 0,
          zip->entry.method,
          MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_UTF8 |
              MZ_ZIP_LDH_BIT_FLAG_HAS_LOCATOR,
          dos_time, dos_date)) {
    // 无法创建 ZIP 条目头部
    err = ZIP_EMEMSET;
    goto cleanup;
  }

  // 设置 ZIP 条目头部的偏移量
  zip->entry.header_offset = zip->entry.offset + num_alignment_padding_bytes;

  // 写入 ZIP 条目头部
  if (pzip->m_pWrite(pzip->m_pIO_opaque, zip->entry.header_offset,
                     zip->entry.header,
                     sizeof(zip->entry.header)) != sizeof(zip->entry.header)) {
    // 无法写入 ZIP 条目头部
    err = ZIP_EMEMSET;
    goto cleanup;
  }

  // 检查文件偏移量是否符合对齐要求
  if (pzip->m_file_offset_alignment) {
    MZ_ASSERT(
        (zip->entry.header_offset & (pzip->m_file_offset_alignment - 1)) == 0);
  }
  // 更新 ZIP 条目的偏移量
  zip->entry.offset += num_alignment_padding_bytes + sizeof(zip->entry.header);

  // 写入 ZIP 条目的数据
  if (pzip->m_pWrite(pzip->m_pIO_opaque, zip->entry.offset, zip->entry.name,
                     entrylen) != entrylen) {
    // 无法写入 ZIP 条目的数据
    err = ZIP_EWRTENT;
    goto cleanup;
  }

  // 更新 ZIP 条目的偏移量
  zip->entry.offset += entrylen;

  // 写入 ZIP64 数据到 ZIP 条目
  if (pzip->m_pWrite(pzip->m_pIO_opaque, zip->entry.offset, extra_data,
                     extra_size) != extra_size) {
    // 无法写入 ZIP64 数据到 ZIP 条目
    err = ZIP_EWRTENT;
    goto cleanup;
  }
  // 更新 ZIP 条目的偏移量
  zip->entry.offset += extra_size;

  // 如果级别不为零，则更新 ZIP 条目的状态信息
  if (level) {
    zip->entry.state.m_pZip = pzip;
    zip->entry.state.m_cur_archive_file_ofs = zip->entry.offset;
    zip->entry.state.m_comp_size = 0;
    // 如果无法初始化 ZIP 压缩器，则返回错误码 ZIP_ETDEFLINIT
    if (tdefl_init(&(zip->entry.comp), mz_zip_writer_add_put_buf_callback,
                   &(zip->entry.state),
                   (int)tdefl_create_comp_flags_from_zip_params(
                       (int)level, -15, MZ_DEFAULT_STRATEGY)) !=
        TDEFL_STATUS_OKAY) {
      // 无法初始化 ZIP 压缩器
      err = ZIP_ETDEFLINIT;
      // 跳转到清理代码的部分
      goto cleanup;
    }
  }

  // 返回成功状态码
  return 0;
// 清理操作的标签，用于在函数结束时进行资源清理
cleanup:
  // 调用 CLEANUP 宏清理 zip->entry.name
  CLEANUP(zip->entry.name);
  // 返回错误码
  return err;
}

// 打开指定名称的 ZIP 文件条目
int zip_entry_open(struct zip_t *zip, const char *entryname) {
  // 调用 _zip_entry_open 函数打开 ZIP 文件条目，不区分大小写
  return _zip_entry_open(zip, entryname, 0);
}

// 打开指定名称的 ZIP 文件条目，区分大小写
int zip_entry_opencasesensitive(struct zip_t *zip, const char *entryname) {
  // 调用 _zip_entry_open 函数打开 ZIP 文件条目，区分大小写
  return _zip_entry_open(zip, entryname, 1);
}

// 根据索引打开 ZIP 文件条目
int zip_entry_openbyindex(struct zip_t *zip, size_t index) {
  // 定义变量
  mz_zip_archive *pZip = NULL;
  mz_zip_archive_file_stat stats;
  mz_uint namelen;
  const mz_uint8 *pHeader;
  const char *pFilename;

  // 检查 zip 是否为空
  if (!zip) {
    // ZIP 处理程序未初始化
    return ZIP_ENOINIT;
  }

  // 获取 ZIP 模式，必须为读取模式
  pZip = &(zip->archive);
  if (pZip->m_zip_mode != MZ_ZIP_MODE_READING) {
    // 打开索引需要只读模式
    return ZIP_EINVMODE;
  }

  // 检查索引是否超出范围
  if (index >= (size_t)pZip->m_total_files) {
    // 索引超出范围
    return ZIP_EINVIDX;
  }

  // 获取文件头信息
  if (!(pHeader = &MZ_ZIP_ARRAY_ELEMENT(
            &pZip->m_pState->m_central_dir, mz_uint8,
            MZ_ZIP_ARRAY_ELEMENT(&pZip->m_pState->m_central_dir_offsets,
                                 mz_uint32, index)))) {
    // 在中央目录中找不到文件头
    return ZIP_ENOHDR;
  }

  // 获取文件名长度和文件名
  namelen = MZ_READ_LE16(pHeader + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  pFilename = (const char *)pHeader + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE;

  /*
    .ZIP 文件格式规范版本：6.3.3

    4.4.17.1 文件名，可包含相对路径。
    存储的路径不得包含驱动器或设备号，或前导斜杠。
    所有斜杠必须是正斜杠 '/'，而不是反斜杠 '\'，以便与 Amiga 和 UNIX 文件系统等兼容。
    如果输入来自标准输入，则没有文件名字段。
  */
  // 清理之前的文件名
  if (zip->entry.name) {
    CLEANUP(zip->entry.name);
  }
  // 根据宏定义选择不同的文件名处理方式
#ifdef ZIP_RAW_ENTRYNAME
  zip->entry.name = STRCLONE(pFilename);
#else
  zip->entry.name = zip_strrpl(pFilename, namelen, '\\', '/');
#endif

  // 检查文件名是否为空
  if (!zip->entry.name) {
    // 本地文件名为空
    return ZIP_EINVENTNAME;
  }

  // 如果无法获取指定索引的文件信息，则返回 ZIP_ENOENT
  if (!mz_zip_reader_file_stat(pZip, (mz_uint)index, &stats)) {
    return ZIP_ENOENT;
  }

  // 设置 ZIP 文件条目的索引、压缩大小、未压缩大小、未压缩 CRC32 校验值、偏移量、头部偏移量、压缩方法和外部属性
  zip->entry.index = (ssize_t)index;
  zip->entry.comp_size = stats.m_comp_size;
  zip->entry.uncomp_size = stats.m_uncomp_size;
  zip->entry.uncomp_crc32 = stats.m_crc32;
  zip->entry.offset = stats.m_central_dir_ofs;
  zip->entry.header_offset = stats.m_local_header_ofs;
  zip->entry.method = stats.m_method;
  zip->entry.external_attr = stats.m_external_attr;
#ifndef MINIZ_NO_TIME
  // 如果未定义 MINIZ_NO_TIME 宏，则设置 ZIP 文件条目的修改时间
  zip->entry.m_time = stats.m_time;
#endif

  // 返回成功
  return 0;
}

// 关闭 ZIP 文件条目
int zip_entry_close(struct zip_t *zip) {
  mz_zip_archive *pzip = NULL;
  mz_uint level;
  tdefl_status done;
  mz_uint16 entrylen;
  mz_uint16 dos_time = 0, dos_date = 0;
  int err = 0;
  mz_uint8 *pExtra_data = NULL;
  mz_uint32 extra_size = 0;
  mz_uint8 extra_data[MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE];
  mz_uint8 local_dir_footer[MZ_ZIP_DATA_DESCRIPTER_SIZE64];
  mz_uint32 local_dir_footer_size = MZ_ZIP_DATA_DESCRIPTER_SIZE64;

  if (!zip) {
    // 如果 zip_t 处理程序未初始化，则返回错误
    err = ZIP_ENOINIT;
    goto cleanup;
  }

  pzip = &(zip->archive);
  if (pzip->m_zip_mode == MZ_ZIP_MODE_READING) {
    // 如果 ZIP 模式为读取，则跳过清理
    goto cleanup;
  }

  level = zip->level & 0xF;
  if (level) {
    // 如果压缩级别不为 0，则压缩缓冲区
    done = tdefl_compress_buffer(&(zip->entry.comp), "", 0, TDEFL_FINISH);
    if (done != TDEFL_STATUS_DONE && done != TDEFL_STATUS_OKAY) {
      // 无法刷新压缩缓冲区，返回错误
      err = ZIP_ETDEFLBUF;
      goto cleanup;
    }
    zip->entry.comp_size = zip->entry.state.m_comp_size;
    zip->entry.offset = zip->entry.state.m_cur_archive_file_ofs;
    zip->entry.method = MZ_DEFLATED;
  }

  entrylen = (mz_uint16)strlen(zip->entry.name);
#ifndef MINIZ_NO_TIME
  // 如果未定义 MINIZ_NO_TIME 宏，则将 ZIP 文件条目的修改时间转换为 DOS 时间
  mz_zip_time_t_to_dos_time(zip->entry.m_time, &dos_time, &dos_date);
#endif

  // 写入本地目录尾部
  MZ_WRITE_LE32(local_dir_footer + 0, MZ_ZIP_DATA_DESCRIPTOR_ID);
  MZ_WRITE_LE32(local_dir_footer + 4, zip->entry.uncomp_crc32);
  MZ_WRITE_LE64(local_dir_footer + 8, zip->entry.comp_size);
  MZ_WRITE_LE64(local_dir_footer + 16, zip->entry.uncomp_size);

  if (pzip->m_pWrite(pzip->m_pIO_opaque, zip->entry.offset, local_dir_footer,
                     local_dir_footer_size) != local_dir_footer_size) {
    // 无法写入 ZIP 文件条目头部，返回错误
    err = ZIP_EWRTHDR;
    // 跳转到清理代码的位置
    goto cleanup;
  }
  // 更新 ZIP 文件条目的偏移量
  zip->entry.offset += local_dir_footer_size;

  // 保存额外数据的指针，并创建 ZIP64 额外数据
  pExtra_data = extra_data;
  extra_size = mz_zip_writer_create_zip64_extra_data(
      extra_data,
      (zip->entry.uncomp_size >= MZ_UINT32_MAX) ? &zip->entry.uncomp_size
                                                : NULL,
      (zip->entry.comp_size >= MZ_UINT32_MAX) ? &zip->entry.comp_size : NULL,
      (zip->entry.header_offset >= MZ_UINT32_MAX) ? &zip->entry.header_offset
                                                  : NULL);

  // 如果是目录且未压缩，则设置 DOS 子目录属性位
  if ((entrylen) && (zip->entry.name[entrylen - 1] == '/') &&
      !zip->entry.uncomp_size) {
    /* Set DOS Subdirectory attribute bit. */
    zip->entry.external_attr |= MZ_ZIP_DOS_DIR_ATTRIBUTE_BITFLAG;
  }

  // 将文件信息添加到 ZIP 中央目录
  if (!mz_zip_writer_add_to_central_dir(
          pzip, zip->entry.name, entrylen, pExtra_data, (mz_uint16)extra_size,
          "", 0, zip->entry.uncomp_size, zip->entry.comp_size,
          zip->entry.uncomp_crc32, zip->entry.method,
          MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_UTF8 |
              MZ_ZIP_LDH_BIT_FLAG_HAS_LOCATOR,
          dos_time, dos_date, zip->entry.header_offset,
          zip->entry.external_attr, NULL, 0)) {
    // 无法写入到 ZIP 中央目录
    err = ZIP_EWRTDIR;
    // 跳转到清理代码的位置
    goto cleanup;
  }

  // 更新 ZIP 文件总数和归档大小
  pzip->m_total_files++;
  pzip->m_archive_size = zip->entry.offset;
// 清理函数，用于释放 ZIP 对象的资源
cleanup:
  // 如果 ZIP 对象存在
  if (zip) {
    // 将 ZIP 对象的 entry 的修改时间设置为 0
    zip->entry.m_time = 0;
    // 清理 ZIP 对象的 entry 的名称
    CLEANUP(zip->entry.name);
  }
  // 返回错误码
  return err;
}

// 获取 ZIP 对象中当前 entry 的名称
const char *zip_entry_name(struct zip_t *zip) {
  // 如果 ZIP 对象未初始化
  if (!zip) {
    // 返回空指针
    return NULL;
  }

  // 返回 ZIP 对象中当前 entry 的名称
  return zip->entry.name;
}

// 获取 ZIP 对象中当前 entry 的索引
ssize_t zip_entry_index(struct zip_t *zip) {
  // 如果 ZIP 对象未初始化
  if (!zip) {
    // 返回 ZIP 未初始化的错误码
    return (ssize_t)ZIP_ENOINIT;
  }

  // 返回 ZIP 对象中当前 entry 的索引
  return zip->entry.index;
}

// 检查 ZIP 对象中当前 entry 是否为目录
int zip_entry_isdir(struct zip_t *zip) {
  // 如果 ZIP 对象未初始化
  if (!zip) {
    // 返回 ZIP 未初始化的错误码
    return ZIP_ENOINIT;
  }

  // 如果当前 entry 的索引小于 0
  if (zip->entry.index < (ssize_t)0) {
    // 返回 ZIP entry 未打开的错误码
    return ZIP_EINVIDX;
  }

  // 返回当前 entry 是否为目录的结果
  return (int)mz_zip_reader_is_file_a_directory(&zip->archive,
                                                (mz_uint)zip->entry.index);
}

// 获取 ZIP 对象中当前 entry 的未压缩大小
unsigned long long zip_entry_size(struct zip_t *zip) {
  // 调用获取 ZIP entry 未压缩大小的函数
  return zip_entry_uncomp_size(zip);
}

// 获取 ZIP 对象中当前 entry 的未压缩大小
unsigned long long zip_entry_uncomp_size(struct zip_t *zip) {
  // 如果 ZIP 对象存在，返回当前 entry 的未压缩大小，否则返回 0
  return zip ? zip->entry.uncomp_size : 0;
}

// 获取 ZIP 对象中当前 entry 的压缩大小
unsigned long long zip_entry_comp_size(struct zip_t *zip) {
  // 如果 ZIP 对象存在，返回当前 entry 的压缩大小，否则返回 0
  return zip ? zip->entry.comp_size : 0;
}

// 获取 ZIP 对象中当前 entry 的 CRC32 校验值
unsigned int zip_entry_crc32(struct zip_t *zip) {
  // 如果 ZIP 对象存在，返回当前 entry 的 CRC32 校验值，否则返回 0
  return zip ? zip->entry.uncomp_crc32 : 0;
}

// 将数据写入 ZIP 对象中当前 entry
int zip_entry_write(struct zip_t *zip, const void *buf, size_t bufsize) {
  mz_uint level;
  mz_zip_archive *pzip = NULL;
  tdefl_status status;

  // 如果 ZIP 对象未初始化
  if (!zip) {
    // 返回 ZIP 未初始化的错误码
    return ZIP_ENOINIT;
  }

  // 获取 ZIP 对象的指针
  pzip = &(zip->archive);
  // 如果数据缓冲区和大小有效
  if (buf && bufsize > 0) {
    // 更新当前 entry 的未压缩大小和 CRC32 校验值
    zip->entry.uncomp_size += bufsize;
    zip->entry.uncomp_crc32 = (mz_uint32)mz_crc32(
        zip->entry.uncomp_crc32, (const mz_uint8 *)buf, bufsize);

    // 获取压缩级别
    level = zip->level & 0xF;
    // 如果压缩级别为 0
    if (!level) {
      // 如果写入数据失败
      if ((pzip->m_pWrite(pzip->m_pIO_opaque, zip->entry.offset, buf,
                          bufsize) != bufsize)) {
        // 返回写入 ZIP entry 失败的错误码
        return ZIP_EWRTENT;
      }
      // 更新 entry 的偏移和压缩大小
      zip->entry.offset += bufsize;
      zip->entry.comp_size += bufsize;
    } else {
      // 使用 tdefl_compress_buffer 函数对缓冲区进行压缩
      status = tdefl_compress_buffer(&(zip->entry.comp), buf, bufsize, TDEFL_NO_FLUSH);
      // 如果压缩状态不是完成状态或者正常状态
      if (status != TDEFL_STATUS_DONE && status != TDEFL_STATUS_OKAY) {
        // 无法压缩缓冲区，返回 ZIP_ETDEFLBUF 错误码
        return ZIP_ETDEFLBUF;
      }
    }
  }

  // 返回 0 表示成功
  return 0;
}
// 将 ZIP 文件中的一个条目写入到 zip_t 结构中
int zip_entry_fwrite(struct zip_t *zip, const char *filename) {
  int err = 0;
  size_t n = 0;
  MZ_FILE *stream = NULL;
  mz_uint8 buf[MZ_ZIP_MAX_IO_BUF_SIZE];
  struct MZ_FILE_STAT_STRUCT file_stat;
  mz_uint16 modes;

  if (!zip) {
    // 如果 zip_t 处理程序未初始化，则返回错误代码
    return ZIP_ENOINIT;
  }

  // 将 buf 数组清零
  memset(buf, 0, MZ_ZIP_MAX_IO_BUF_SIZE);
  // 将 file_stat 结构清零
  memset((void *)&file_stat, 0, sizeof(struct MZ_FILE_STAT_STRUCT));
  // 获取文件信息，如果失败则返回错误代码
  if (MZ_FILE_STAT(filename, &file_stat) != 0) {
    // 获取信息出错，检查 errno
    return ZIP_ENOENT;
  }

#if defined(_WIN32) || defined(__WIN32__) || defined(DJGPP)
  (void)modes; // 未使用
#else
  /* 使用权限位初始化 modes 变量--这不是实现可选的 */
  modes = file_stat.st_mode &
          (S_IRWXU | S_IRWXG | S_IRWXO | S_ISUID | S_ISGID | S_ISVTX);
  if (S_ISDIR(file_stat.st_mode))
    modes |= UNX_IFDIR;
  if (S_ISREG(file_stat.st_mode))
    modes |= UNX_IFREG;
  if (S_ISLNK(file_stat.st_mode))
    modes |= UNX_IFLNK;
  if (S_ISBLK(file_stat.st_mode))
    modes |= UNX_IFBLK;
  if (S_ISCHR(file_stat.st_mode))
    modes |= UNX_IFCHR;
  if (S_ISFIFO(file_stat.st_mode))
    modes |= UNX_IFIFO;
  if (S_ISSOCK(file_stat.st_mode))
    modes |= UNX_IFSOCK;
  // 设置 zip->entry.external_attr 属性
  zip->entry.external_attr = (modes << 16) | !(file_stat.st_mode & S_IWUSR);
  if ((file_stat.st_mode & S_IFMT) == S_IFDIR) {
    zip->entry.external_attr |= MZ_ZIP_DOS_DIR_ATTRIBUTE_BITFLAG;
  }
#endif

  // 设置 zip->entry.m_time 属性
  zip->entry.m_time = file_stat.st_mtime;

  // 打开文件，如果失败则返回错误代码
  if (!(stream = MZ_FOPEN(filename, "rb"))) {
    // 无法打开文件
    return ZIP_EOPNFILE;
  }

  // 读取文件内容并写入 zip 结构中
  while ((n = fread(buf, sizeof(mz_uint8), MZ_ZIP_MAX_IO_BUF_SIZE, stream) > 0)) {
    if (zip_entry_write(zip, buf, n) < 0) {
      err = ZIP_EWRTENT;
      break;
    }
  }
  // 关闭文件流
  fclose(stream);

  return err;
}

// 从 zip_t 结构中读取 ZIP 条目的内容
ssize_t zip_entry_read(struct zip_t *zip, void **buf, size_t *bufsize) {
  mz_zip_archive *pzip = NULL;
  mz_uint idx;
  size_t size = 0;

  if (!zip) {
    // 如果 zip_t 处理程序未初始化
    // 返回 ZIP_ENOINIT 的 ssize_t 类型值
    return (ssize_t)ZIP_ENOINIT;
  }

  // 获取 ZIP 对象指针
  pzip = &(zip->archive);
  // 检查 ZIP 模式是否为读取模式，以及索引是否小于0
  if (pzip->m_zip_mode != MZ_ZIP_MODE_READING ||
      zip->entry.index < (ssize_t)0) {
    // 如果条目未找到或者没有读取权限，则返回 ZIP_ENOENT 的 ssize_t 类型值
    return (ssize_t)ZIP_ENOENT;
  }

  // 获取索引值
  idx = (mz_uint)zip->entry.index;
  // 检查条目是否为目录
  if (mz_zip_reader_is_file_a_directory(pzip, idx)) {
    // 如果条目是目录，则返回 ZIP_EINVENTTYPE 的 ssize_t 类型值
    return (ssize_t)ZIP_EINVENTTYPE;
  }

  // 从 ZIP 中提取数据到堆中，并将结果存储在 buf 中
  *buf = mz_zip_reader_extract_to_heap(pzip, idx, &size, 0);
  // 如果 buf 不为空且 bufsize 不为0，则将 size 存储在 bufsize 中
  if (*buf && bufsize) {
    *bufsize = size;
  }
  // 返回 size 的 ssize_t 类型值
  return (ssize_t)size;
// 读取 ZIP 文件中指定索引的条目数据到缓冲区，不进行内存分配
ssize_t zip_entry_noallocread(struct zip_t *zip, void *buf, size_t bufsize) {
  mz_zip_archive *pzip = NULL;

  if (!zip) {
    // zip_t 处理程序未初始化
    return (ssize_t)ZIP_ENOINIT;
  }

  pzip = &(zip->archive);
  if (pzip->m_zip_mode != MZ_ZIP_MODE_READING ||
      zip->entry.index < (ssize_t)0) {
    // 未找到条目或者没有读取权限
    return (ssize_t)ZIP_ENOENT;
  }

  if (!mz_zip_reader_extract_to_mem_no_alloc(pzip, (mz_uint)zip->entry.index,
                                             buf, bufsize, 0, NULL, 0)) {
    return (ssize_t)ZIP_EMEMNOALLOC;
  }

  return (ssize_t)zip->entry.uncomp_size;
}

// 从 ZIP 文件中读取指定文件名的条目
int zip_entry_fread(struct zip_t *zip, const char *filename) {
  mz_zip_archive *pzip = NULL;
  mz_uint idx;
  mz_uint32 xattr = 0;
  mz_zip_archive_file_stat info;

  if (!zip) {
    // zip_t 处理程序未初始化
    return ZIP_ENOINIT;
  }

  // 初始化文件信息结构
  memset((void *)&info, 0, sizeof(mz_zip_archive_file_stat));
  pzip = &(zip->archive);
  if (pzip->m_zip_mode != MZ_ZIP_MODE_READING ||
      zip->entry.index < (ssize_t)0) {
    // 未找到条目或者没有读取权限
    return ZIP_ENOENT;
  }

  idx = (mz_uint)zip->entry.index;
  if (mz_zip_reader_is_file_a_directory(pzip, idx)) {
    // 条目是一个目录
    return ZIP_EINVENTTYPE;
  }

  // 将条目数据提取到文件中
  if (!mz_zip_reader_extract_to_file(pzip, idx, filename, 0)) {
    return ZIP_ENOFILE;
  }

  // 根据操作系统设置文件属性
#if defined(_MSC_VER) || defined(PS4)
  (void)xattr; // 未使用
#else
  if (!mz_zip_reader_file_stat(pzip, idx, &info)) {
    // 无法获取有关 ZIP 存档的信息
    return ZIP_ENOFILE;
  }

  xattr = (info.m_external_attr >> 16) & 0xFFFF;
  if (xattr > 0 && xattr <= MZ_UINT16_MAX) {
    if (CHMOD(filename, (mode_t)xattr) < 0) {
      return ZIP_ENOPERM;
    }
  }
#endif

  return 0;
}
// 从 zip_t 结构中提取 zip 条目
int zip_entry_extract(struct zip_t *zip,
                      size_t (*on_extract)(void *arg, uint64_t offset,
                                           const void *buf, size_t bufsize),
                      void *arg) {
  mz_zip_archive *pzip = NULL;
  mz_uint idx;

  if (!zip) {
    // zip_t 处理程序未初始化
    return ZIP_ENOINIT;
  }

  pzip = &(zip->archive);
  if (pzip->m_zip_mode != MZ_ZIP_MODE_READING ||
      zip->entry.index < (ssize_t)0) {
    // 条目未找到或者没有读取权限
    return ZIP_ENOENT;
  }

  idx = (mz_uint)zip->entry.index;
  return (mz_zip_reader_extract_to_callback(pzip, idx, on_extract, arg, 0))
             ? 0
             : ZIP_EINVIDX;
}

// 返回 zip_t 结构中的总条目数
ssize_t zip_entries_total(struct zip_t *zip) {
  if (!zip) {
    // zip_t 处理程序未初始化
    return ZIP_ENOINIT;
  }

  return (ssize_t)zip->archive.m_total_files;
}

// 删除 zip_t 结构中指定的条目
ssize_t zip_entries_delete(struct zip_t *zip, char *const entries[],
                           size_t len) {
  ssize_t n = 0;
  ssize_t err = 0;
  struct zip_entry_mark_t *entry_mark = NULL;

  if (zip == NULL || (entries == NULL && len != 0)) {
    return ZIP_ENOINIT;
  }

  if (entries == NULL && len == 0) {
    return 0;
  }

  n = zip_entries_total(zip);

  entry_mark = (struct zip_entry_mark_t *)calloc(
      (size_t)n, sizeof(struct zip_entry_mark_t));
  if (!entry_mark) {
    return ZIP_EOOMEM;
  }

  zip->archive.m_zip_mode = MZ_ZIP_MODE_READING;

  err = zip_entry_set(zip, entry_mark, n, entries, len);
  if (err < 0) {
    CLEANUP(entry_mark);
    return err;
  }

  err = zip_entries_delete_mark(zip, entry_mark, (int)n);
  CLEANUP(entry_mark);
  return err;
}

// 从流中提取 zip 条目
int zip_stream_extract(const char *stream, size_t size, const char *dir,
                       int (*on_extract)(const char *filename, void *arg),
                       void *arg) {
  mz_zip_archive zip_archive;
  if (!stream || !dir) {
    // 无法解析 zip 存档流
    // 返回 ZIP_ENOINIT 错误代码
    return ZIP_ENOINIT;
  }
  // 使用 memset 函数将 zip_archive 结构体清零，如果失败则返回 ZIP_EMEMSET 错误代码
  if (!memset(&zip_archive, 0, sizeof(mz_zip_archive))) {
    // 不能清零 zip_archive 结构体
    return ZIP_EMEMSET;
  }
  // 使用 mz_zip_reader_init_mem 函数初始化 zip_archive 读取器，如果失败则返回 ZIP_ENOINIT 错误代码
  if (!mz_zip_reader_init_mem(&zip_archive, stream, size, 0)) {
    // 无法初始化 zip_archive 读取器
    return ZIP_ENOINIT;
  }

  // 调用 zip_archive_extract 函数提取 zip_archive 中的内容到指定目录，执行提取回调函数 on_extract，传入参数 arg
  return zip_archive_extract(&zip_archive, dir, on_extract, arg);
// 打开一个 ZIP 流，返回一个指向 zip_t 结构体的指针
struct zip_t *zip_stream_open(const char *stream, size_t size, int level,
                              char mode) {
  int errnum = 0;
  // 调用带错误码参数的 zip_stream_open 函数
  return zip_stream_openwitherror(stream, size, level, mode, &errnum);
}

// 打开一个 ZIP 流，返回一个指向 zip_t 结构体的指针，并传入错误码参数
struct zip_t *zip_stream_openwitherror(const char *stream, size_t size,
                                       int level, char mode, int *errnum) {
  // 分配一个 zip_t 结构体的内存空间
  struct zip_t *zip = (struct zip_t *)calloc((size_t)1, sizeof(struct zip_t));
  if (!zip) {
    // 内存不足
    *errnum = ZIP_EOOMEM;
    return NULL;
  }

  if (level < 0) {
    level = MZ_DEFAULT_LEVEL;
  }
  if ((level & 0xF) > MZ_UBER_COMPRESSION) {
    // 压缩级别错误
    *errnum = ZIP_EINVLVL;
    goto cleanup;
  }
  zip->level = (mz_uint)level;

  if ((stream != NULL) && (size > 0) && (mode == 'r')) {
    if (!mz_zip_reader_init_mem(&(zip->archive), stream, size, 0)) {
      *errnum = ZIP_ERINIT;
      goto cleanup;
    }
  } else if ((stream == NULL) && (size == 0) && (mode == 'w')) {
    // 创建一个新的存档
    if (!mz_zip_writer_init_heap(&(zip->archive), 0, 1024)) {
      // 无法初始化 zip_archive 写入器
      *errnum = ZIP_EWINIT;
      goto cleanup;
    }
  } else {
    *errnum = ZIP_EINVMODE;
    goto cleanup;
  }

  *errnum = 0;
  return zip;

cleanup:
  // 清理资源
  CLEANUP(zip);
  return NULL;
}

// 复制 ZIP 流的内容到缓冲区
ssize_t zip_stream_copy(struct zip_t *zip, void **buf, size_t *bufsize) {
  size_t n;

  if (!zip) {
    return (ssize_t)ZIP_ENOINIT;
  }
  // 完成 zip_archive 结构体的最终化
  zip_archive_finalize(&(zip->archive));

  n = (size_t)zip->archive.m_archive_size;
  if (bufsize != NULL) {
    *bufsize = n;
  }

  // 分配内存并复制 ZIP 流的内容到缓冲区
  *buf = calloc(sizeof(unsigned char), n);
  memcpy(*buf, zip->archive.m_pState->m_pMem, n);

  return (ssize_t)n;
}

// 关闭 ZIP 流
void zip_stream_close(struct zip_t *zip) {
  if (zip) {
    // 结束 ZIP 写入器和读取器
    mz_zip_writer_end(&(zip->archive));
    mz_zip_reader_end(&(zip->archive));
    // 清理资源
    CLEANUP(zip);
  }
}
// 创建一个 ZIP 文件，包含指定文件名数组中的文件
int zip_create(const char *zipname, const char *filenames[], size_t len) {
  int err = 0; // 初始化错误码为 0
  size_t i; // 定义循环变量 i
  mz_zip_archive zip_archive; // 定义 ZIP 存档对象
  struct MZ_FILE_STAT_STRUCT file_stat; // 定义文件状态结构体
  mz_uint32 ext_attributes = 0; // 初始化扩展属性为 0
  mz_uint16 modes; // 定义文件模式

  if (!zipname || strlen(zipname) < 1) {
    // 如果 ZIP 文件名为空或长度小于 1，则返回错误码 ZIP_EINVZIPNAME
    return ZIP_EINVZIPNAME;
  }

  // 创建一个新的存档
  if (!memset(&(zip_archive), 0, sizeof(zip_archive))) {
    // 无法使用 memset 初始化 zip_archive，返回错误码 ZIP_EMEMSET
    return ZIP_EMEMSET;
  }

  if (!mz_zip_writer_init_file(&zip_archive, zipname, 0)) {
    // 无法初始化 zip_archive 写入器，返回错误码 ZIP_ENOINIT
    return ZIP_ENOINIT;
  }

  if (!memset((void *)&file_stat, 0, sizeof(struct MZ_FILE_STAT_STRUCT))) {
    // 无法使用 memset 初始化 file_stat，返回错误码 ZIP_EMEMSET
    return ZIP_EMEMSET;
  }

  for (i = 0; i < len; ++i) {
    const char *name = filenames[i]; // 获取当前文件名
    if (!name) {
      // 如果文件名为空，设置错误码 ZIP_EINVENTNAME 并跳出循环
      err = ZIP_EINVENTNAME;
      break;
    }

    if (MZ_FILE_STAT(name, &file_stat) != 0) {
      // 获取文件信息失败，检查 errno，设置错误码 ZIP_ENOFILE 并跳出循环
      err = ZIP_ENOFILE;
      break;
    }

#if defined(_WIN32) || defined(__WIN32__) || defined(DJGPP)
    (void)modes; // 未使用的变量
#else

    /* 使用权限位初始化模式--这不是实现可选的 */
    modes = file_stat.st_mode &
            (S_IRWXU | S_IRWXG | S_IRWXO | S_ISUID | S_ISGID | S_ISVTX);
    if (S_ISDIR(file_stat.st_mode))
      modes |= UNX_IFDIR;
    if (S_ISREG(file_stat.st_mode))
      modes |= UNX_IFREG;
    if (S_ISLNK(file_stat.st_mode))
      modes |= UNX_IFLNK;
    if (S_ISBLK(file_stat.st_mode))
      modes |= UNX_IFBLK;
    if (S_ISCHR(file_stat.st_mode))
      modes |= UNX_IFCHR;
    if (S_ISFIFO(file_stat.st_mode))
      modes |= UNX_IFIFO;
    if (S_ISSOCK(file_stat.st_mode))
      modes |= UNX_IFSOCK;
    ext_attributes = (modes << 16) | !(file_stat.st_mode & S_IWUSR);
    if ((file_stat.st_mode & S_IFMT) == S_IFDIR) {
      ext_attributes |= MZ_ZIP_DOS_DIR_ATTRIBUTE_BITFLAG;
    }
#endif
    // 如果无法将文件添加到 zip_archive 中，则返回错误 ZIP_ENOFILE
    if (!mz_zip_writer_add_file(&zip_archive, zip_basename(name), name, "", 0,
                                ZIP_DEFAULT_COMPRESSION_LEVEL,
                                ext_attributes)) {
      // 无法将文件添加到 zip_archive
      err = ZIP_ENOFILE;
      break;
    }
  }

  // 完成 zip_archive 的最终化
  mz_zip_writer_finalize_archive(&zip_archive);
  // 结束 zip_archive 的写入
  mz_zip_writer_end(&zip_archive);
  // 返回错误码
  return err;
// 从 ZIP 文件中提取文件到指定目录，同时可以在提取过程中执行回调函数
int zip_extract(const char *zipname, const char *dir,
                int (*on_extract)(const char *filename, void *arg), void *arg) {
  // 创建一个 mz_zip_archive 结构体用于存储 ZIP 文件的信息
  mz_zip_archive zip_archive;

  // 检查 ZIP 文件名和目录名是否为空
  if (!zipname || !dir) {
    // 无法解析 ZIP 文件名
    return ZIP_EINVZIPNAME;
  }

  // 使用 memset 函数将 zip_archive 结构体清零
  if (!memset(&zip_archive, 0, sizeof(mz_zip_archive))) {
    // 无法清零 zip_archive 结构体
    return ZIP_EMEMSET;
  }

  // 尝试打开 ZIP 文件
  if (!mz_zip_reader_init_file(&zip_archive, zipname, 0)) {
    // 无法初始化 zip_archive 读取器
    return ZIP_ENOINIT;
  }

  // 调用函数来提取 ZIP 文件中的内容到指定目录，并在提取过程中执行回调函数
  return zip_archive_extract(&zip_archive, dir, on_extract, arg);
}
```