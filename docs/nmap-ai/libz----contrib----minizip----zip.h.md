# `nmap\libz\contrib\minizip\zip.h`

```
# 包含有关 .zip 文件的 IO 操作的头文件，使用 zlib 库
# 版本信息和版权声明
# 条款和条件
# 更改信息
# 定义条件编译指令
# 包含必要的头文件
# 定义 Z_BZIP2ED 常量
# 定义 STRICTZIP 或 STRICTZIPUNZIP 常量
/* 定义结构体 zipFile__，用于表示 ZIP 文件 */
typedef struct TagzipFile__ { int unused; } zipFile__;
/* 定义指向 zipFile__ 结构体的指针类型 zipFile */
typedef zipFile__ *zipFile;
#else
/* 如果没有定义 zipFile 类型，则定义为 void 指针类型 */
typedef voidp zipFile;
#endif

/* 定义 ZIP 文件操作返回值常量 */
#define ZIP_OK                          (0)
#define ZIP_EOF                         (0)
#define ZIP_ERRNO                       (Z_ERRNO)
#define ZIP_PARAMERROR                  (-102)
#define ZIP_BADZIPFILE                  (-103)
#define ZIP_INTERNALERROR               (-104)

#ifndef DEF_MEM_LEVEL
/* 如果未定义 DEF_MEM_LEVEL，则根据最大内存级别定义默认内存级别 */
#  if MAX_MEM_LEVEL >= 8
#    define DEF_MEM_LEVEL 8
#  else
#    define DEF_MEM_LEVEL  MAX_MEM_LEVEL
#  endif
#endif
/* 默认内存级别 */

/* 定义 tm_zip 结构体，包含日期/时间信息 */
typedef struct tm_zip_s
{
    int tm_sec;             /* 每分钟的秒数 - [0,59] */
    int tm_min;             /* 每小时的分钟数 - [0,59] */
    int tm_hour;            /* 从午夜开始的小时数 - [0,23] */
    int tm_mday;            /* 月份中的日期 - [1,31] */
    int tm_mon;             /* 从一月开始的月数 - [0,11] */
    int tm_year;            /* 年份 - [1980..2044] */
} tm_zip;

/* 定义 zip_fileinfo 结构体，包含文件信息 */
typedef struct
{
    tm_zip      tmz_date;       /* 可理解的日期格式 */
    uLong       dosDate;       /* 如果 dos_date == 0，则使用 tmu_date */

    uLong       internal_fa;    /* 内部文件属性 */
    uLong       external_fa;    /* 外部文件属性 */
} zip_fileinfo;

/* 定义 zipcharpc 类型，表示指向常量字符的指针 */
typedef const char* zipcharpc;

/* 定义追加文件到 ZIP 的状态常量 */
#define APPEND_STATUS_CREATE        (0)
#define APPEND_STATUS_CREATEAFTER   (1)
#define APPEND_STATUS_ADDINZIP      (2)

/* 声明 zipOpen 函数，用于打开 ZIP 文件并返回 zipFile 对象 */
extern zipFile ZEXPORT zipOpen OF((const char *pathname, int append));
/* 声明 zipOpen64 函数，用于打开 ZIP 文件并返回 zipFile 对象（64位） */
extern zipFile ZEXPORT zipOpen64 OF((const void *pathname, int append));
/*
  创建一个zip文件。
     pathname在Windows XP上包含一个类似"c:\\zlib\\zlib113.zip"的文件名，或者在Unix计算机上是"zlib/zlib113.zip"。
     如果文件pathname存在，并且append==APPEND_STATUS_CREATEAFTER，那么zip文件将被创建在文件的末尾。
       （如果文件包含自解压缩代码，则很有用）
     如果文件pathname存在，并且append==APPEND_STATUS_ADDINZIP，我们将在现有的zip文件中添加文件
       （确保不要添加不存在的文件）
     如果无法打开zip文件，则返回值为NULL。
     否则，返回值是一个zipFile句柄，可用于此zip包的其他函数。
*/

/* 注意：zip文件中没有删除函数。
   如果要在zip文件中删除文件，必须打开一个zip文件，并创建另一个zip文件。
   当然，您可以使用原始读写来复制您不想删除的文件
*/

extern zipFile ZEXPORT zipOpen2 OF((const char *pathname,
                                   int append,
                                   zipcharpc* globalcomment,
                                   zlib_filefunc_def* pzlib_filefunc_def));

extern zipFile ZEXPORT zipOpen2_64 OF((const void *pathname,
                                   int append,
                                   zipcharpc* globalcomment,
                                   zlib_filefunc64_def* pzlib_filefunc_def));

extern zipFile ZEXPORT zipOpen3 OF((const void *pathname,
                                    int append,
                                    zipcharpc* globalcomment,
                                    zlib_filefunc64_32_def* pzlib_filefunc64_32_def));
extern int ZEXPORT zipOpenNewFileInZip OF((zipFile file,
                       const char* filename,
                       const zip_fileinfo* zipfi,
                       const void* extrafield_local,
                       uInt size_extrafield_local,
                       const void* extrafield_global,
                       uInt size_extrafield_global,
                       const char* comment,
                       int method,
                       int level));

extern int ZEXPORT zipOpenNewFileInZip64 OF((zipFile file,
                       const char* filename,
                       const zip_fileinfo* zipfi,
                       const void* extrafield_local,
                       uInt size_extrafield_local,
                       const void* extrafield_global,
                       uInt size_extrafield_global,
                       const char* comment,
                       int method,
                       int level,
                       int zip64));

/*
  在 ZIP 文件中打开一个新文件进行写入。
  filename : ZIP 中的文件名（如果为 NULL，则使用未引用的 '-'）
  *zipfi 包含附加信息
  如果 extrafield_local!=NULL 并且 size_extrafield_local>0，则 extrafield_local 包含本地头部的额外字段数据
  如果 extrafield_global!=NULL 并且 size_extrafield_global>0，则 extrafield_global 包含本地头部的额外字段数据
  如果 comment != NULL，则 comment 包含注释字符串
  method 包含压缩方法（0 表示存储，Z_DEFLATED 表示压缩）
  level 包含压缩级别（可以是 Z_DEFAULT_COMPRESSION）
  如果未压缩大小 >= 0xffffffff，则将 zip64 设置为 1，以便在本地文件头部添加 zip64 扩展信息块。
                    如果未压缩大小 >= 0xffffffff，则这必须是 '1'。
*/
# 定义一个函数用于在 ZIP 文件中打开一个新文件，并写入数据
# 参数包括：zipFile对象，文件名，文件信息，本地额外字段，本地额外字段大小，全局额外字段，全局额外字段大小，注释，压缩方法，压缩级别，是否写入原始文件
extern int ZEXPORT zipOpenNewFileInZip2 OF((zipFile file,
                                            const char* filename,
                                            const zip_fileinfo* zipfi,
                                            const void* extrafield_local,
                                            uInt size_extrafield_local,
                                            const void* extrafield_global,
                                            uInt size_extrafield_global,
                                            const char* comment,
                                            int method,
                                            int level,
                                            int raw));

# 定义一个函数用于在 ZIP 文件中打开一个新文件，并写入数据（支持64位）
# 参数包括：zipFile对象，文件名，文件信息，本地额外字段，本地额外字段大小，全局额外字段，全局额外字段大小，注释，压缩方法，压缩级别，是否写入原始文件，是否使用64位
extern int ZEXPORT zipOpenNewFileInZip2_64 OF((zipFile file,
                                            const char* filename,
                                            const zip_fileinfo* zipfi,
                                            const void* extrafield_local,
                                            uInt size_extrafield_local,
                                            const void* extrafield_global,
                                            uInt size_extrafield_global,
                                            const char* comment,
                                            int method,
                                            int level,
                                            int raw,
                                            int zip64));
/*
  如果 raw=1，则与zipOpenNewFileInZip相同，除非我们写入原始文件
 */
// 声明一个外部的函数 zipOpenNewFileInZip3，接受多个参数
// file: zipFile 类型的文件对象
// filename: 文件名
// zipfi: zip_fileinfo 类型的文件信息
// extrafield_local: 本地额外字段数据
// size_extrafield_local: 本地额外字段数据的大小
// extrafield_global: 全局额外字段数据
// size_extrafield_global: 全局额外字段数据的大小
// comment: 注释
// method: 压缩方法
// level: 压缩级别
// raw: 是否使用原始数据
// windowBits: 窗口位数
// memLevel: 内存级别
// strategy: 压缩策略
// password: 密码
// crcForCrypting: 用于加密的 CRC
extern int ZEXPORT zipOpenNewFileInZip3_64 OF((zipFile file,  // 声明一个外部函数，用于在 ZIP 文件中打开一个新文件
                                            const char* filename,  // 新文件的文件名
                                            const zip_fileinfo* zipfi,  // 新文件的文件信息
                                            const void* extrafield_local,  // 本地额外字段
                                            uInt size_extrafield_local,  // 本地额外字段的大小
                                            const void* extrafield_global,  // 全局额外字段
                                            uInt size_extrafield_global,  // 全局额外字段的大小
                                            const char* comment,  // 注释
                                            int method,  // 压缩方法
                                            int level,  // 压缩级别
                                            int raw,  // 是否原始数据
                                            int windowBits,  // 窗口位数
                                            int memLevel,  // 内存级别
                                            int strategy,  // 策略
                                            const char* password,  // 加密密码
                                            uLong crcForCrypting,  // 用于加密的文件的 CRC
                                            int zip64  // 是否使用 ZIP64
                                            ));

/*
  与 zipOpenNewFileInZip2 相同，除了
    windowBits,memLevel,,strategy : 参见 deflateInit2 中的 strategy 参数
    password : 加密密码（NULL 表示不加密）
    crcForCrypting : 要压缩的文件的 CRC（加密时需要）
 */
// 定义一个外部函数，用于在 ZIP 文件中打开一个新文件并进行写入
extern int ZEXPORT zipOpenNewFileInZip4 OF((zipFile file,  // ZIP 文件对象
                                            const char* filename,  // 文件名
                                            const zip_fileinfo* zipfi,  // 文件信息
                                            const void* extrafield_local,  // 本地额外字段
                                            uInt size_extrafield_local,  // 本地额外字段大小
                                            const void* extrafield_global,  // 全局额外字段
                                            uInt size_extrafield_global,  // 全局额外字段大小
                                            const char* comment,  // 注释
                                            int method,  // 压缩方法
                                            int level,  // 压缩级别
                                            int raw,  // 是否原始数据
                                            int windowBits,  // 窗口位数
                                            int memLevel,  // 内存级别
                                            int strategy,  // 策略
                                            const char* password,  // 密码
                                            uLong crcForCrypting,  // 用于加密的 CRC
                                            uLong versionMadeBy,  // 创建文件的版本
                                            uLong flagBase  // 基本标志
                                            ));
# 定义一个函数，用于在 ZIP 文件中打开一个新文件，并写入数据
extern int ZEXPORT zipOpenNewFileInZip4_64 OF((zipFile file,
                                            const char* filename,
                                            const zip_fileinfo* zipfi,
                                            const void* extrafield_local,
                                            uInt size_extrafield_local,
                                            const void* extrafield_global,
                                            uInt size_extrafield_global,
                                            const char* comment,
                                            int method,
                                            int level,
                                            int raw,
                                            int windowBits,
                                            int memLevel,
                                            int strategy,
                                            const char* password,
                                            uLong crcForCrypting,
                                            uLong versionMadeBy,
                                            uLong flagBase,
                                            int zip64
                                            ));
/*
  与 zipOpenNewFileInZip4 相同，不同之处在于：
    versionMadeBy：Version made by 字段的值
    flag：flag 字段的值（将添加压缩级别信息）
 */

# 定义一个函数，用于在 ZIP 文件中写入数据
extern int ZEXPORT zipWriteInFileInZip OF((zipFile file,
                       const void* buf,
                       unsigned len));
/*
  在 ZIP 文件中写入数据
*/

# 定义一个函数，用于在 ZIP 文件中关闭当前文件
extern int ZEXPORT zipCloseFileInZip OF((zipFile file));
/*
  在 ZIP 文件中关闭当前文件
*/

# 定义一个函数，用于在 ZIP 文件中以原始方式关闭当前文件
extern int ZEXPORT zipCloseFileInZipRaw OF((zipFile file,
                                            uLong uncompressed_size,
                                            uLong crc32));
/*
  声明一个函数，用于在 ZIP 文件中关闭当前文件
  file: ZIP 文件对象
  uncompressed_size: 未压缩大小
  crc32: CRC32 校验值
*/
extern int ZEXPORT zipCloseFileInZipRaw64 OF((zipFile file,
                                            ZPOS64_T uncompressed_size,
                                            uLong crc32));

/*
  关闭 ZIP 文件中当前文件的函数，用于在使用参数 raw=1 的情况下在 zipOpenNewFileInZip2 中打开文件
  uncompressed_size 和 crc32 是未压缩大小和 CRC32 校验值
*/

/*
  声明一个函数，用于关闭 ZIP 文件
  file: ZIP 文件对象
  global_comment: 全局注释
*/
extern int ZEXPORT zipClose OF((zipFile file,
                const char* global_comment));
/*
  关闭 ZIP 文件的函数
*/

/*
  声明一个函数，用于移除额外信息块
  pData: 需要处理的数据
  dataLen: 数据长度
  sHeader: 头部标识
*/
extern int ZEXPORT zipRemoveExtraInfoBlock OF((char* pData, int* dataLen, short sHeader));
/*
  zipRemoveExtraInfoBlock - 由 Mathias Svensson 添加

  从本地文件头或中央目录头的额外信息数据中移除额外信息块

  在使用原始模式时，需要在写入数据之前移除 ZIP64 额外信息块

  0x0001 是 ZIP64 额外信息块的签名头

  用法：
                        从中央目录额外字段数据中移除 ZIP64 额外信息
              zipRemoveExtraInfoBlock(pCenDirExtraFieldData, &nCenDirExtraFieldDataLen, 0x0001);

                        从本地文件头额外字段数据中移除 ZIP64 额外信息
        zipRemoveExtraInfoBlock(pLocalHeaderExtraFieldData, &nLocalHeaderExtraFieldDataLen, 0x0001);
*/

#ifdef __cplusplus
}
#endif

#endif /* _zip64_H */
```