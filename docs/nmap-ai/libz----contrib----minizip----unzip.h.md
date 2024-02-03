# `nmap\libz\contrib\minizip\unzip.h`

```cpp
/* unzip.h -- 使用 zlib 解压缩 .zip 文件的 IO
   Version 1.1, 2010年2月14日
   MiniZip 项目的一部分 - ( http://www.winimage.com/zLibDll/minizip.html )

         版权所有 (C) 1998-2010 Gilles Vollant (minizip) ( http://www.winimage.com/zLibDll/minizip.html )

         Unzip 的 Zip64 修改
         版权所有 (C) 2007-2008 Even Rouault

         用于 zip 和 unzip 的 Zip64 支持的修改
         版权所有 (C) 2009-2010 Mathias Svensson ( http://result42.com )

         更多信息请阅读 MiniZip_info.txt

         ---------------------------------------------------------------------------------

        使用和分发条件与 zlib 相同：

  本软件按原样提供，没有任何明示或暗示的保证。
  作者不对因使用本软件而产生的任何损害负责。

  任何人都可以使用本软件进行任何目的，包括商业应用，并且可以自由修改和重新分发，但必须遵守以下限制：

  1. 本软件的来源不得被歪曲；您不得声称自己编写了原始软件。如果您在产品中使用本软件，产品文档中的致谢将不是必需的，但会受到赞赏。
  2. 修改后的源版本必须明确标记，并且不得被误传为原始软件。
  3. 本通知不得从任何源分发中删除或更改。

  ---------------------------------------------------------------------------------

        变更

        请参阅 unzip64.c 的头部

*/

#ifndef _unz64_H
#define _unz64_H

#ifdef __cplusplus
extern "C" {
#endif

#ifndef _ZLIB_H
#include "zlib.h"
#endif

#ifndef  _ZLIBIOAPI_H
#include "ioapi.h"
#endif

#ifdef HAVE_BZIP2
#include "bzlib.h"
#endif

#define Z_BZIP2ED 12

#if defined(STRICTUNZIP) || defined(STRICTZIPUNZIP)
/* 定义一个不能直接从(void*)转换而需要强制转换的指针，类似于WIN32的STRICT */
typedef struct TagunzFile__ { int unused; } unzFile__;
typedef unzFile__ *unzFile;
#else
typedef voidp unzFile;
#endif

/* 定义一些常量，表示不同的返回状态 */
#define UNZ_OK                          (0)
#define UNZ_END_OF_LIST_OF_FILE         (-100)
#define UNZ_ERRNO                       (Z_ERRNO)
#define UNZ_EOF                         (0)
#define UNZ_PARAMERROR                  (-102)
#define UNZ_BADZIPFILE                  (-103)
#define UNZ_INTERNALERROR               (-104)
#define UNZ_CRCERROR                    (-105)

/* tm_unz 结构包含日期/时间信息 */
typedef struct tm_unz_s
{
    int tm_sec;             /* 分钟之后的秒数 - [0,59] */
    int tm_min;             /* 小时之后的分钟数 - [0,59] */
    int tm_hour;            /* 从午夜开始的小时数 - [0,23] */
    int tm_mday;            /* 月份的日期 - [1,31] */
    int tm_mon;             /* 从一月开始的月数 - [0,11] */
    int tm_year;            /* 年份 - [1980..2044] */
} tm_unz;

/* unz_global_info64 结构包含关于ZIP文件的全局数据
   这些数据来自中央目录的末尾 */
typedef struct unz_global_info64_s
{
    ZPOS64_T number_entry;         /* 该磁盘上中央目录中的条目总数 */
    uLong size_comment;         /* 压缩文件的全局注释的大小 */
} unz_global_info64;

typedef struct unz_global_info_s
{
    uLong number_entry;         /* 该磁盘上中央目录中的条目总数 */
    uLong size_comment;         /* 压缩文件的全局注释的大小 */
} unz_global_info;

/* unz_file_info 结构包含ZIP文件中文件的信息 */
typedef struct unz_file_info64_s
{
    uLong version;              /* 创建版本                 2字节 */
    uLong version_needed;       /* 提取所需的版本           2字节 */
    uLong flag;                 /* 通用位标志，占2个字节 */
    uLong compression_method;   /* 压缩方法，占2个字节 */
    uLong dosDate;              /* 文件的最后修改日期，以Dos格式表示，占4个字节 */
    uLong crc;                  /* crc-32校验值，占4个字节 */
    ZPOS64_T compressed_size;   /* 压缩后的文件大小，占8个字节 */
    ZPOS64_T uncompressed_size; /* 解压后的文件大小，占8个字节 */
    uLong size_filename;        /* 文件名长度，占2个字节 */
    uLong size_file_extra;      /* 额外字段长度，占2个字节 */
    uLong size_file_comment;    /* 文件注释长度，占2个字节 */

    uLong disk_num_start;       /* 起始磁盘号，占2个字节 */
    uLong internal_fa;          /* 内部文件属性，占2个字节 */
    uLong external_fa;          /* 外部文件属性，占4个字节 */

    tm_unz tmu_date;            /* 文件的最后修改日期和时间 */
} unz_file_info64;

typedef struct unz_file_info_s
{
    uLong version;              /* version made by                 2 bytes */
    uLong version_needed;       /* version needed to extract       2 bytes */
    uLong flag;                 /* general purpose bit flag        2 bytes */
    uLong compression_method;   /* compression method              2 bytes */
    uLong dosDate;              /* last mod file date in Dos fmt   4 bytes */
    uLong crc;                  /* crc-32                          4 bytes */
    uLong compressed_size;      /* compressed size                 4 bytes */
    uLong uncompressed_size;    /* uncompressed size               4 bytes */
    uLong size_filename;        /* filename length                 2 bytes */
    uLong size_file_extra;      /* extra field length              2 bytes */
    uLong size_file_comment;    /* file comment length             2 bytes */

    uLong disk_num_start;       /* disk number start               2 bytes */
    uLong internal_fa;          /* internal file attributes        2 bytes */
    uLong external_fa;          /* external file attributes        4 bytes */

    tm_unz tmu_date;
} unz_file_info;

extern int ZEXPORT unzStringFileNameCompare OF ((const char* fileName1,
                                                 const char* fileName2,
                                                 int iCaseSensitivity));
/*
   Compare two filename (fileName1,fileName2).
   If iCaseSenisivity = 1, comparision is case sensitivity (like strcmp)
   If iCaseSenisivity = 2, comparision is not case sensitivity (like strcmpi
                                or strcasecmp)
   If iCaseSenisivity = 0, case sensitivity is defaut of your operating system
    (like 1 on Unix, 2 on Windows)
*/


extern unzFile ZEXPORT unzOpen OF((const char *path));
extern unzFile ZEXPORT unzOpen64 OF((const void *path));
/*
  打开一个 Zip 文件。path 包含完整的路径名（例如，在 Windows XP 计算机上为 "c:\\zlib\\zlib113.zip"，在 Unix 计算机上为 "zlib/zlib113.zip"）。
  如果无法打开 zip 文件（文件不存在或无效），返回值为 NULL。
  否则，返回值是一个 unzFile 句柄，可用于此解压缩包的其他函数。
  "64" 函数接受一个 const void* 指针，因为路径只是传递给 open64_file_func 回调的值。
  在 Windows 下，如果定义了 UNICODE，使用 fill_fopen64_filefunc，路径是一个宽 Unicode 字符串的指针（LPCTSTR 是 LPCWSTR），因此 const char* 不符合实际情况
*/

extern unzFile ZEXPORT unzOpen2 OF((const char *path,
                                    zlib_filefunc_def* pzlib_filefunc_def));
/*
   打开一个 Zip 文件，类似于 unzOpen，但提供一组用于读/写 zip 文件的文件低级 API（参见 ioapi.h）
*/

extern unzFile ZEXPORT unzOpen2_64 OF((const void *path,
                                    zlib_filefunc64_def* pzlib_filefunc_def));
/*
   打开一个 Zip 文件，类似于 unz64Open，但提供一组用于读/写 zip 文件的文件低级 API（参见 ioapi.h）
*/

extern int ZEXPORT unzClose OF((unzFile file));
/*
  关闭用 unzOpen 打开的 ZipFile。
  如果在使用 unzOpenCurrentFile 打开了 .Zip 中的文件（参见后文），在调用 unzClose 之前，这些文件必须使用 unzCloseCurrentFile 关闭。
  如果没有问题，返回 UNZ_OK。
*/

extern int ZEXPORT unzGetGlobalInfo OF((unzFile file,
                                        unz_global_info *pglobal_info));

extern int ZEXPORT unzGetGlobalInfo64 OF((unzFile file,
                                        unz_global_info64 *pglobal_info));
/*
  在 *pglobal_info 结构中写入有关 ZipFile 的信息。
  不需要准备结构
  如果没有问题，返回 UNZ_OK。
*/
# 定义一个外部函数，用于获取 ZipFile 的全局注释
extern int ZEXPORT unzGetGlobalComment OF((unzFile file,
                                           char *szComment,
                                           uLong uSizeBuf));
/*
  获取 ZipFile 的全局注释字符串，存储在 szComment 缓冲区中。
  uSizeBuf 是 szComment 缓冲区的大小。
  返回复制的字节数或小于0的错误代码
*/


/***************************************************************************/
/* Unzip package allow you browse the directory of the zipfile */

# 定义一个外部函数，用于将 zipfile 的当前文件设置为第一个文件
extern int ZEXPORT unzGoToFirstFile OF((unzFile file));
/*
  将 zipfile 的当前文件设置为第一个文件。
  如果没有问题，返回 UNZ_OK
*/

# 定义一个外部函数，用于将 zipfile 的当前文件设置为下一个文件
extern int ZEXPORT unzGoToNextFile OF((unzFile file));
/*
  将 zipfile 的当前文件设置为下一个文件。
  如果没有问题，返回 UNZ_OK
  如果当前文件是最后一个文件，则返回 UNZ_END_OF_LIST_OF_FILE
*/

# 定义一个外部函数，用于尝试在 zipfile 中定位文件
extern int ZEXPORT unzLocateFile OF((unzFile file,
                     const char *szFileName,
                     int iCaseSensitivity));
/*
  尝试在 zipfile 中定位文件 szFileName。
  关于 iCaseSensitivity 的含义，请参见 unzStringFileNameCompare

  返回值：
  如果找到文件，则返回 UNZ_OK。它将成为当前文件。
  如果未找到文件，则返回 UNZ_END_OF_LIST_OF_FILE
*/


/* ****************************************** */
/* Ryan supplied functions */
/* unz_file_info contain information about a file in the zipfile */

# 定义一个结构，包含 zipfile 中文件的信息
typedef struct unz_file_pos_s
{
    uLong pos_in_zip_directory;   /* 偏移量在 zip 文件目录中 */
    uLong num_of_file;            /* 文件数量 */
} unz_file_pos;

# 定义一个外部函数，用于获取文件位置信息
extern int ZEXPORT unzGetFilePos(
    unzFile file,
    unz_file_pos* file_pos);

# 定义一个外部函数，用于将文件位置设置为指定位置
extern int ZEXPORT unzGoToFilePos(
    unzFile file,
    unz_file_pos* file_pos);

# 定义一个结构，包含 zipfile 中文件的信息（64位）
typedef struct unz64_file_pos_s
{
    ZPOS64_T pos_in_zip_directory;   /* 偏移量在 zip 文件目录中 */
    ZPOS64_T num_of_file;            /* 文件数量 */
} unz64_file_pos;

# 定义一个外部函数，用于获取文件位置信息（64位）
extern int ZEXPORT unzGetFilePos64(
    # 定义一个名为file的变量，类型为unzFile，表示一个未压缩的文件
    # 定义一个名为file_pos的指针变量，指向unz64_file_pos类型的数据，用于表示未压缩文件的位置信息
# 定义函数 unzGoToFilePos64，用于将文件指针移动到指定位置
extern int ZEXPORT unzGoToFilePos64(
    unzFile file,
    const unz64_file_pos* file_pos);

/* ****************************************** */

# 定义函数 unzGetCurrentFileInfo64，用于获取当前文件的信息（64位版本）
extern int ZEXPORT unzGetCurrentFileInfo64 OF((unzFile file,
                         unz_file_info64 *pfile_info,
                         char *szFileName,
                         uLong fileNameBufferSize,
                         void *extraField,
                         uLong extraFieldBufferSize,
                         char *szComment,
                         uLong commentBufferSize));

# 定义函数 unzGetCurrentFileInfo，用于获取当前文件的信息
extern int ZEXPORT unzGetCurrentFileInfo OF((unzFile file,
                         unz_file_info *pfile_info,
                         char *szFileName,
                         uLong fileNameBufferSize,
                         void *extraField,
                         uLong extraFieldBufferSize,
                         char *szComment,
                         uLong commentBufferSize));
/*
  获取当前文件的信息
  如果 pfile_info!=NULL，则 *pfile_info 结构将包含有关当前文件的一些信息
  如果 szFileName!=NULL，则文件名字符串将被复制到 szFileName 中（fileNameBufferSize 是缓冲区的大小）
  如果 extraField!=NULL，则额外字段信息将被复制到 extraField 中（extraFieldBufferSize 是缓冲区的大小）
  这是额外字段的中央头版本
  如果 szComment!=NULL，则文件的注释字符串将被复制到 szComment 中（commentBufferSize 是缓冲区的大小）
*/


/** Addition for GDAL : START */

# 定义函数 unzGetCurrentFileZStreamPos64，用于获取当前文件的 ZStream 位置（64位版本）
extern ZPOS64_T ZEXPORT unzGetCurrentFileZStreamPos64 OF((unzFile file));

/** Addition for GDAL : END */


/***************************************************************************/
/* 用于读取当前压缩文件的内容，可以打开它，从中读取数据，然后关闭它（可以在读取完所有文件之前关闭它）
   */
# 定义函数 unzOpenCurrentFile，用于打开当前文件
extern int ZEXPORT unzOpenCurrentFile OF((unzFile file));
/*
  打开 ZIP 文件中当前文件以供读取数据。
  如果没有错误，返回值为 UNZ_OK。
*/

extern int ZEXPORT unzOpenCurrentFilePassword OF((unzFile file,
                                                  const char* password));
/*
  打开 ZIP 文件中当前文件以供读取数据。
  password 是加密密码。
  如果没有错误，返回值为 UNZ_OK。
*/

extern int ZEXPORT unzOpenCurrentFile2 OF((unzFile file,
                                           int* method,
                                           int* level,
                                           int raw));
/*
  与 unzOpenCurrentFile 相同，但如果 raw==1，则以原始格式打开文件以供读取（不解压缩）。
  *method 将接收压缩方法，*level 将接收压缩级别。
  注意：您可以将 level 参数设置为 NULL（如果您不想知道级别），但是您不能将 method 参数设置为 NULL。
*/

extern int ZEXPORT unzOpenCurrentFile3 OF((unzFile file,
                                           int* method,
                                           int* level,
                                           int raw,
                                           const char* password));
/*
  与 unzOpenCurrentFile 相同，但如果 raw==1，则以原始格式打开文件以供读取（不解压缩）。
  *method 将接收压缩方法，*level 将接收压缩级别。
  注意：您可以将 level 参数设置为 NULL（如果您不想知道级别），但是您不能将 method 参数设置为 NULL。
*/

extern int ZEXPORT unzCloseCurrentFile OF((unzFile file));
/*
  关闭使用 unzOpenCurrentFile 打开的 ZIP 文件中的文件。
  如果已读取整个文件但 CRC 不正确，则返回 UNZ_CRCERROR。
*/

extern int ZEXPORT unzReadCurrentFile OF((unzFile file,
                      voidp buf,
                      unsigned len));
/*
  从当前文件（由unzOpenCurrentFile打开）读取字节
  buf包含数据必须被复制的缓冲区
  len是buf的大小。

  如果有字节被复制，则返回复制的字节数
  如果到达文件末尾，则返回0
  如果出现错误，则返回<0的错误代码
    （IO错误为UNZ_ERRNO，解压缩错误为zLib错误）
*/

extern z_off_t ZEXPORT unztell OF((unzFile file));

extern ZPOS64_T ZEXPORT unztell64 OF((unzFile file));
/*
  给出未压缩数据的当前位置
*/

extern int ZEXPORT unzeof OF((unzFile file));
/*
  如果到达文件末尾，则返回1，否则返回0
*/

extern int ZEXPORT unzGetLocalExtrafield OF((unzFile file,
                                             voidp buf,
                                             unsigned len));
/*
  从当前文件（由unzOpenCurrentFile打开）读取额外字段
  这是额外字段的本地头版本（有时，本地头版本中的信息比中央头版本中的信息更多）

  如果buf==NULL，则返回本地额外字段的大小

  如果buf！=NULL，则len是缓冲区的大小，额外头将被复制到buf中。
  返回值是复制到buf中的字节数，或（如果<0）
  错误代码
*/

/***************************************************************************/

/* 获取当前文件偏移量 */
extern ZPOS64_T ZEXPORT unzGetOffset64 (unzFile file);
extern uLong ZEXPORT unzGetOffset (unzFile file);

/* 设置当前文件偏移量 */
extern int ZEXPORT unzSetOffset64 (unzFile file, ZPOS64_T pos);
extern int ZEXPORT unzSetOffset (unzFile file, uLong pos);



#ifdef __cplusplus
}
#endif

#endif /* _unz64_H */
```