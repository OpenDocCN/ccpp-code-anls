# `stable-diffusion.cpp\thirdparty\zip.h`

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

/*
 * 防止头文件被重复包含
 */
#pragma once
#ifndef ZIP_H
#define ZIP_H

/*
 * 包含必要的头文件
 */
#include <stdint.h>
#include <string.h>
#include <sys/types.h>

/*
 * 定义 ZIP_EXPORT 宏
 */
#ifndef ZIP_SHARED
#define ZIP_EXPORT
#else
#ifdef _WIN32
#ifdef ZIP_BUILD_SHARED
#define ZIP_EXPORT __declspec(dllexport)
#else
#define ZIP_EXPORT __declspec(dllimport)
#endif
#else
#define ZIP_EXPORT __attribute__((visibility("default")))
#endif
#endif

#ifdef __cplusplus
extern "C" {
#endif

#if !defined(_POSIX_C_SOURCE) && defined(_MSC_VER)
// 64-bit Windows is the only mainstream platform
// where sizeof(long) != sizeof(void*)
#ifdef _WIN64
typedef long long ssize_t; /* byte count or error */
#else
typedef long ssize_t; /* byte count or error */
#endif
#endif

/**
 * @mainpage
 *
 * Documentation for @ref zip.
 */

/**
 * @addtogroup zip
 * @{
 */

/**
 * 默认的 ZIP 压缩级别
 */
#define ZIP_DEFAULT_COMPRESSION_LEVEL 6

/**
 * 错误代码
 */
#define ZIP_ENOINIT -1      // not initialized
#define ZIP_EINVENTNAME -2  // invalid entry name
#define ZIP_ENOENT -3       // entry not found
#define ZIP_EINVMODE -4     // invalid zip mode
#define ZIP_EINVLVL -5      // invalid compression level
#define ZIP_ENOSUP64 -6     // no zip 64 support
#define ZIP_EMEMSET -7      // memset error
#define ZIP_EWRTENT -8      // cannot write data to entry
#define ZIP_ETDEFLINIT -9   // cannot initialize tdefl compressor
#define ZIP_EINVIDX -10     // invalid index
#define ZIP_ENOHDR -11      // header not found
#define ZIP_ETDEFLBUF -12   // cannot flush tdefl buffer
// 定义 ZIP 错误码
#define ZIP_ECRTHDR -13     // 无法创建条目头部
#define ZIP_EWRTHDR -14     // 无法写入条目头部
#define ZIP_EWRTDIR -15     // 无法写入中央目录
#define ZIP_EOPNFILE -16    // 无法打开文件
#define ZIP_EINVENTTYPE -17 // 无效的条目类型
#define ZIP_EMEMNOALLOC -18 // 使用无内存分配提取数据
#define ZIP_ENOFILE -19     // 文件未找到
#define ZIP_ENOPERM -20     // 没有权限
#define ZIP_EOOMEM -21      // 内存不足
#define ZIP_EINVZIPNAME -22 // 无效的 ZIP 存档名称
#define ZIP_EMKDIR -23      // 创建目录错误
#define ZIP_ESYMLINK -24    // 符号链接错误
#define ZIP_ECLSZIP -25     // 关闭存档错误
#define ZIP_ECAPSIZE -26    // 容量大小太小
#define ZIP_EFSEEK -27      // fseek 错误
#define ZIP_EFREAD -28      // fread 错误
#define ZIP_EFWRITE -29     // fwrite 错误
#define ZIP_ERINIT -30      // 无法初始化读取器
#define ZIP_EWINIT -31      // 无法初始化写入器
#define ZIP_EWRINIT -32     // 无法从读取器初始化写入器

/**
 * 查找与错误号对应的错误消息字符串。
 * @param errnum 错误号
 * @return 与 errnum 对应的错误消息字符串，如果找不到错误则返回 NULL。
 */
extern ZIP_EXPORT const char *zip_strerror(int errnum);

/**
 * @struct zip_t
 *
 * 该数据结构在整个库中用于表示 zip 存档 - 前向声明。
 */
struct zip_t;

/**
 * 使用给定模式和压缩级别打开 zip 存档。
 *
 * @param zipname zip 存档文件名。
 * @param level 压缩级别（0-9 是标准的 zlib 风格级别）。
 * @param mode 文件访问模式。
 *        - 'r': 打开文件进行读取/提取（文件必须存在）。
 *        - 'w': 创建一个空文件进行写入。
 *        - 'a': 追加到现有存档。
 *
 * @return zip 存档处理程序，如果出现错误则返回 NULL
 */
/**
 * 打开具有给定压缩级别和模式的 zip 存档。
 * 该函数还返回 @param errnum -
 *
 * @param zipname zip 存档文件名。
 * @param level 压缩级别（0-9 是标准的 zlib 风格级别）。
 * @param mode 文件访问模式。
 *        - 'r'：打开文件进行读取/提取（文件必须存在）。
 *        - 'w'：创建一个空文件进行写入。
 *        - 'a'：追加到现有存档。
 * @param errnum 成功返回 0，错误返回负数（< 0）。
 *
 * @return zip 存档处理程序，错误时返回 NULL
 */
extern ZIP_EXPORT struct zip_t *zip_open(const char *zipname, int level, char mode);

/**
 * 关闭 zip 存档，释放资源 - 总是要完成。
 *
 * @param zip zip 存档处理程序。
 */
extern ZIP_EXPORT void zip_close(struct zip_t *zip);

/**
 * 确定存档是否具有 zip64 中央目录尾部头。
 *
 * @param zip zip 存档处理程序。
 *
 * @return 返回代码 - 1（true），0（false），错误时返回负数（< 0）。
 */
extern ZIP_EXPORT int zip_is64(struct zip_t *zip);

/**
 * 通过名称在 zip 存档中打开条目。
 *
 * 对于以 'w' 或 'a' 模式打开的 zip 存档，该函数将追加一个新条目。
 * 在只读模式下，函数尝试在全局字典中定位条目。
 *
 * @param zip zip 存档处理程序。
 * @param entryname 本地字典中的条目名称。
 *
 * @return 返回代码 - 成功返回 0，错误返回负数（< 0）。
 */
extern ZIP_EXPORT int zip_entry_open(struct zip_t *zip, const char *entryname);
/**
 * 通过名称在zip存档中打开一个条目。
 *
 * 对于以'w'或'a'模式打开的zip存档，该函数将追加一个新条目。在只读模式下，函数尝试在全局字典中定位条目（区分大小写）。
 *
 * @param zip zip存档处理程序。
 * @param entryname 本地字典中的条目名称（区分大小写）。
 *
 * @return 返回代码 - 成功时为0，错误时为负数（<0）。
 */
extern ZIP_EXPORT int zip_entry_opencasesensitive(struct zip_t *zip,
                                                  const char *entryname);

/**
 * 通过索引在zip存档中打开一个新条目。
 *
 * 仅当zip存档以'r'（只读）模式打开时，此函数才有效。
 *
 * @param zip zip存档处理程序。
 * @param index 本地字典中的索引。
 *
 * @return 返回代码 - 成功时为0，错误时为负数（<0）。
 */
extern ZIP_EXPORT int zip_entry_openbyindex(struct zip_t *zip, size_t index);

/**
 * 关闭zip条目，刷新缓冲区并释放资源。
 *
 * @param zip zip存档处理程序。
 *
 * @return 返回代码 - 成功时为0，错误时为负数（<0）。
 */
extern ZIP_EXPORT int zip_entry_close(struct zip_t *zip);

/**
 * 返回当前zip条目的本地名称。
 *
 * 用户条目名称和本地条目名称之间的主要区别是可选的相对路径。
 * 根据.ZIP文件格式规范 - 存储的路径不得包含驱动器或设备字母，也不得包含前导斜杠。
 * 所有斜杠必须是正斜杠'/'，而不是反斜杠'\'，以便与Amiga和UNIX文件系统等兼容。
 *
 * @param zip zip存档处理程序。
 *
 * @return 指向当前zip条目名称的指针，错误时返回NULL。
 */
extern ZIP_EXPORT const char *zip_entry_name(struct zip_t *zip);

/**
 * 返回当前zip条目的索引。
 *
 * @param zip zip存档处理程序。
 *
 * @return 成功时返回索引，错误时返回负数（<0）。
 */
/**
 * 获取当前 ZIP 条目的索引。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 返回代码 - 1（true），0（false），负数（< 0）表示错误。
 */
extern ZIP_EXPORT ssize_t zip_entry_index(struct zip_t *zip);

/**
 * 确定当前 ZIP 条目是否为目录条目。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 返回代码 - 1（true），0（false），负数（< 0）表示错误。
 */
extern ZIP_EXPORT int zip_entry_isdir(struct zip_t *zip);

/**
 * 返回当前 ZIP 条目的未压缩大小。
 * 为了向后兼容，别名为 zip_entry_uncomp_size。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 未压缩大小（以字节为单位）。
 */
extern ZIP_EXPORT unsigned long long zip_entry_size(struct zip_t *zip);

/**
 * 返回当前 ZIP 条目的未压缩大小。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 未压缩大小（以字节为单位）。
 */
extern ZIP_EXPORT unsigned long long zip_entry_uncomp_size(struct zip_t *zip);

/**
 * 返回当前 ZIP 条目的压缩大小。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 压缩大小（以字节为单位）。
 */
extern ZIP_EXPORT unsigned long long zip_entry_comp_size(struct zip_t *zip);

/**
 * 返回当前 ZIP 条目的 CRC-32 校验和。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return CRC-32 校验和。
 */
extern ZIP_EXPORT unsigned int zip_entry_crc32(struct zip_t *zip);

/**
 * 压缩输入缓冲区以用于当前 ZIP 条目。
 *
 * @param zip ZIP 存档处理程序。
 * @param buf 输入缓冲区。
 * @param bufsize 输入缓冲区大小（以字节为单位）。
 *
 * @return 返回代码 - 成功为 0，错误为负数（< 0）。
 */
extern ZIP_EXPORT int zip_entry_write(struct zip_t *zip, const void *buf,
                                      size_t bufsize);

/**
 * 压缩文件以用于当前 ZIP 条目。
 *
 * @param zip ZIP 存档处理程序。
 * @param filename 输入文件。
 *
 * @return 返回代码 - 成功为 0，错误为负数（< 0）。
 */
extern ZIP_EXPORT int zip_entry_fwrite(struct zip_t *zip, const char *filename);
/**
 * 从当前 ZIP 条目中提取数据到输出缓冲区。
 *
 * 该函数为输出缓冲区分配足够的内存。
 *
 * @param zip ZIP 存档处理程序。
 * @param buf 输出缓冲区。
 * @param bufsize 输出缓冲区大小（以字节为单位）。
 *
 * @note 记得释放为输出缓冲区分配的内存。
 *       对于大条目，请查看 zip_entry_extract 函数。
 *
 * @return 返回代码 - 成功时实际读取的字节数。
 *         否则在错误时返回负数（< 0）。
 */
extern ZIP_EXPORT ssize_t zip_entry_read(struct zip_t *zip, void **buf,
                                         size_t *bufsize);

/**
 * 从当前 ZIP 条目中提取数据到内存缓冲区，不使用内存分配。
 *
 * @param zip ZIP 存档处理程序。
 * @param buf 预分配的输出缓冲区。
 * @param bufsize 输出缓冲区大小（以字节为单位）。
 *
 * @note 确保提供的输出缓冲区足够大。
 *       zip_entry_size 函数（返回当前条目的未压缩大小）可用于估计所需的缓冲区大小。
 *       对于大条目，请查看 zip_entry_extract 函数。
 *
 * @return 返回代码 - 成功时实际读取的字节数。
 *         否则在错误时返回负数（< 0）（例如 bufsize 不够大）。
 */
extern ZIP_EXPORT ssize_t zip_entry_noallocread(struct zip_t *zip, void *buf,
                                                size_t bufsize);

/**
 * 将当前 ZIP 条目提取到输出文件中。
 *
 * @param zip ZIP 存档处理程序。
 * @param filename 输出文件。
 *
 * @return 返回代码 - 成功时为 0，错误时为负数（< 0）。
 */
extern ZIP_EXPORT int zip_entry_fread(struct zip_t *zip, const char *filename);
/**
 * 使用回调函数（on_extract）提取当前的 ZIP 条目。
 *
 * @param zip ZIP 存档处理程序。
 * @param on_extract 回调函数。
 * @param arg 不透明指针（可选参数，可以传递给 on_extract 回调）
 *
 * @return 返回代码 - 成功时为 0，错误时为负数（< 0）。
 */
extern ZIP_EXPORT int
zip_entry_extract(struct zip_t *zip,
                  size_t (*on_extract)(void *arg, uint64_t offset,
                                       const void *data, size_t size),
                  void *arg);

/**
 * 返回 ZIP 存档中所有条目（文件和目录）的数量。
 *
 * @param zip ZIP 存档处理程序。
 *
 * @return 返回代码 - 成功时为条目数量，错误时为负数（< 0）。
 */
extern ZIP_EXPORT ssize_t zip_entries_total(struct zip_t *zip);

/**
 * 删除 ZIP 存档条目。
 *
 * @param zip ZIP 存档处理程序。
 * @param entries 要删除的 ZIP 存档条目数组。
 * @param len 要删除的条目数量。
 * @return 删除的条目数量，或错误时为负数（< 0）。
 */
extern ZIP_EXPORT ssize_t zip_entries_delete(struct zip_t *zip,
                                             char *const entries[], size_t len);

/**
 * 将 ZIP 存档流提取到目录中。
 *
 * 如果 on_extract 不为 NULL，则在成功提取每个 ZIP 条目后将调用回调函数。
 * 从回调函数返回负值将导致中止并返回错误。最后一个参数（void *arg）是可选的，您可以使用它将数据传递给 on_extract 回调。
 *
 * @param stream ZIP 存档流。
 * @param size 流大小。
 * @param dir 输出目录。
 * @param on_extract 提取回调。
 * @param arg 不透明指针。
 *
 * @return 返回代码 - 成功时为 0，错误时为负数（< 0）。
 */
extern ZIP_EXPORT int
/**
 * 从内存中提取 ZIP 流，并解压到指定目录
 *
 * @param stream ZIP 流
 * @param size 流大小
 * @param dir 目标目录
 * @param on_extract 解压回调函数
 * @param arg 回调函数参数
 */
zip_stream_extract(const char *stream, size_t size, const char *dir,
                   int (*on_extract)(const char *filename, void *arg),
                   void *arg);

/**
 * 打开内存中的 ZIP 存档流
 *
 * @param stream ZIP 存档流
 * @param size 流大小
 * @param level 压缩级别 (0-9 是标准的 zlib 风格级别)
 * @param mode 文件访问模式
 *        - 'r': 打开文件进行读取/提取 (文件必须存在)
 *        - 'w': 创建一个空文件进行写入
 *        - 'a': 追加到现有存档
 *
 * @return ZIP 存档处理器或错误时返回 NULL
 */
extern ZIP_EXPORT struct zip_t *zip_stream_open(const char *stream, size_t size,
                                                int level, char mode);

/**
 * 打开内存中的 ZIP 存档流
 * 该函数额外返回 @param errnum -
 *
 * @param stream ZIP 存档流
 * @param size 流大小
 * @param level 压缩级别 (0-9 是标准的 zlib 风格级别)
 * @param mode 文件访问模式
 *        - 'r': 打开文件进行读取/提取 (文件必须存在)
 *        - 'w': 创建一个空文件进行写入
 *        - 'a': 追加到现有存档
 * @param errnum 成功时返回 0，错误时返回负数 (< 0)
 *
 * @return ZIP 存档处理器或错误时返回 NULL
 */
extern ZIP_EXPORT struct zip_t *zip_stream_openwitherror(const char *stream,
                                                         size_t size, int level,
                                                         char mode,
                                                         int *errnum);

/**
 * 复制 ZIP 存档流输出缓冲区
 *
 * @param zip ZIP 存档处理器
 * @param buf 输出缓冲区。用户应该释放 buf
 * @param bufsize 输出缓冲区大小 (字节)
 *
 * @return 复制大小
 */
/**
 * 复制 ZIP 流中的数据到缓冲区中
 *
 * @param zip ZIP 对象指针
 * @param buf 缓冲区指针
 * @param bufsize 缓冲区大小
 *
 * @return 返回复制的字节数，负数表示错误
 */
extern ZIP_EXPORT ssize_t zip_stream_copy(struct zip_t *zip, void **buf,
                                          size_t *bufsize);

/**
 * 关闭 ZIP 归档文件，释放资源
 *
 * @param zip ZIP 归档文件处理器
 */
extern ZIP_EXPORT void zip_stream_close(struct zip_t *zip);

/**
 * 创建一个新的归档文件，并将文件放入单个 ZIP 归档文件中
 *
 * @param zipname ZIP 归档文件名
 * @param filenames 输入文件名数组
 * @param len 输入文件数量
 *
 * @return 返回代码 - 成功为 0，错误为负数
 */
extern ZIP_EXPORT int zip_create(const char *zipname, const char *filenames[],
                                 size_t len);

/**
 * 将 ZIP 归档文件解压到目录中
 *
 * 如果 on_extract_entry 不为 NULL，则在成功提取每个 ZIP 条目后将调用回调函数
 * 从回调函数返回负值将导致中止并返回错误。最后一个参数 (void *arg) 是可选的，您可以使用它将数据传递给 on_extract_entry 回调函数。
 *
 * @param zipname ZIP 归档文件名
 * @param dir 输出目录
 * @param on_extract_entry 提取回调函数
 * @param arg 透明指针
 *
 * @return 返回代码 - 成功为 0，错误为负数
 */
extern ZIP_EXPORT int zip_extract(const char *zipname, const char *dir,
                                  int (*on_extract_entry)(const char *filename,
                                                          void *arg),
                                  void *arg);
/** @} */
#ifdef __cplusplus
}
#endif

#endif
```