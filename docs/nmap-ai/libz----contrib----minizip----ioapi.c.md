# `nmap\libz\contrib\minizip\ioapi.c`

```
/* ioapi.h -- IO base function header for compress/uncompress .zip
   part of the MiniZip project - ( http://www.winimage.com/zLibDll/minizip.html )

         Copyright (C) 1998-2010 Gilles Vollant (minizip) ( http://www.winimage.com/zLibDll/minizip.html )

         Modifications for Zip64 support
         Copyright (C) 2009-2010 Mathias Svensson ( http://result42.com )

         For more info read MiniZip_info.txt

*/

#if defined(_WIN32) && (!(defined(_CRT_SECURE_NO_WARNINGS)))
        #define _CRT_SECURE_NO_WARNINGS
#endif

#if defined(__APPLE__) || defined(IOAPI_NO_64)
// In darwin and perhaps other BSD variants off_t is a 64 bit value, hence no need for specific 64 bit functions
#define FOPEN_FUNC(filename, mode) fopen(filename, mode)
#define FTELLO_FUNC(stream) ftello(stream)
#define FSEEKO_FUNC(stream, offset, origin) fseeko(stream, offset, origin)
#else
#define FOPEN_FUNC(filename, mode) fopen64(filename, mode)
#define FTELLO_FUNC(stream) ftello64(stream)
#define FSEEKO_FUNC(stream, offset, origin) fseeko64(stream, offset, origin)
#endif


#include "ioapi.h"

voidpf call_zopen64 (const zlib_filefunc64_32_def* pfilefunc,const void*filename,int mode)
{
    if (pfilefunc->zfile_func64.zopen64_file != NULL)
        return (*(pfilefunc->zfile_func64.zopen64_file)) (pfilefunc->zfile_func64.opaque,filename,mode);
    else
    {
        return (*(pfilefunc->zopen32_file))(pfilefunc->zfile_func64.opaque,(const char*)filename,mode);
    }
}

long call_zseek64 (const zlib_filefunc64_32_def* pfilefunc,voidpf filestream, ZPOS64_T offset, int origin)
{
    if (pfilefunc->zfile_func64.zseek64_file != NULL)
        return (*(pfilefunc->zfile_func64.zseek64_file)) (pfilefunc->zfile_func64.opaque,filestream,offset,origin);
    else
    {
        uLong offsetTruncated = (uLong)offset;
        if (offsetTruncated != offset)
            return -1;
        else
            return (*(pfilefunc->zseek32_file))(pfilefunc->zfile_func64.opaque,filestream,offsetTruncated,origin);
    # 代码块结束
# 定义一个函数，用于获取文件指针的当前位置
ZPOS64_T call_ztell64 (const zlib_filefunc64_32_def* pfilefunc,voidpf filestream)
{
    # 如果存在 64 位的 seek 函数，则调用该函数获取文件指针的当前位置
    if (pfilefunc->zfile_func64.zseek64_file != NULL)
        return (*(pfilefunc->zfile_func64.ztell64_file)) (pfilefunc->zfile_func64.opaque,filestream);
    # 如果不存在 64 位的 seek 函数，则调用 32 位的 tell 函数获取文件指针的当前位置
    else
    {
        uLong tell_uLong = (uLong)(*(pfilefunc->ztell32_file))(pfilefunc->zfile_func64.opaque,filestream);
        # 如果返回的位置是最大的 32 位无符号整数，则返回 -1
        if ((tell_uLong) == MAXU32)
            return (ZPOS64_T)-1;
        # 否则返回 32 位的位置
        else
            return tell_uLong;
    }
}

# 定义一个函数，用于将 32 位的文件操作函数转换为 64 位的文件操作函数
void fill_zlib_filefunc64_32_def_from_filefunc32(zlib_filefunc64_32_def* p_filefunc64_32,const zlib_filefunc_def* p_filefunc32)
{
    # 将 64 位的打开文件函数设置为 NULL
    p_filefunc64_32->zfile_func64.zopen64_file = NULL;
    # 将 32 位的打开文件函数设置为 32 位文件操作函数中的打开文件函数
    p_filefunc64_32->zopen32_file = p_filefunc32->zopen_file;
    # 将 64 位的错误处理函数设置为 32 位文件操作函数中的错误处理函数
    p_filefunc64_32->zfile_func64.zerror_file = p_filefunc32->zerror_file;
    # 将 64 位的读取文件函数设置为 32 位文件操作函数中的读取文件函数
    p_filefunc64_32->zfile_func64.zread_file = p_filefunc32->zread_file;
    # 将 64 位的写入文件函数设置为 32 位文件操作函数中的写入文件函数
    p_filefunc64_32->zfile_func64.zwrite_file = p_filefunc32->zwrite_file;
    # 将 64 位的 64 位 seek 函数设置为 NULL
    p_filefunc64_32->zfile_func64.ztell64_file = NULL;
    p_filefunc64_32->zfile_func64.zseek64_file = NULL;
    # 将 64 位的关闭文件函数设置为 32 位文件操作函数中的关闭文件函数
    p_filefunc64_32->zfile_func64.zclose_file = p_filefunc32->zclose_file;
    # 将 64 位的错误处理函数设置为 32 位文件操作函数中的错误处理函数
    p_filefunc64_32->zfile_func64.zerror_file = p_filefunc32->zerror_file;
    # 将 64 位的 opaque 设置为 32 位文件操作函数中的 opaque
    p_filefunc64_32->zfile_func64.opaque = p_filefunc32->opaque;
    # 将 64 位的 32 位 seek 函数设置为 32 位文件操作函数中的 seek 函数
    p_filefunc64_32->zseek32_file = p_filefunc32->zseek_file;
    # 将 64 位的 32 位 tell 函数设置为 32 位文件操作函数中的 tell 函数
    p_filefunc64_32->ztell32_file = p_filefunc32->ztell_file;
}

# 定义一系列静态函数指针，用于文件操作
static voidpf  ZCALLBACK fopen_file_func OF((voidpf opaque, const char* filename, int mode));
static uLong   ZCALLBACK fread_file_func OF((voidpf opaque, voidpf stream, void* buf, uLong size));
static uLong   ZCALLBACK fwrite_file_func OF((voidpf opaque, voidpf stream, const void* buf,uLong size));
static ZPOS64_T ZCALLBACK ftell64_file_func OF((voidpf opaque, voidpf stream));
static long    ZCALLBACK fseek64_file_func OF((voidpf opaque, voidpf stream, ZPOS64_T offset, int origin));
static int     ZCALLBACK fclose_file_func OF((voidpf opaque, voidpf stream));
# 定义一个静态函数，用于处理文件操作时的错误
static int     ZCALLBACK ferror_file_func OF((voidpf opaque, voidpf stream));

# 定义一个函数，用于以指定模式打开文件
static voidpf ZCALLBACK fopen_file_func (voidpf opaque, const char* filename, int mode)
{
    FILE* file = NULL;
    const char* mode_fopen = NULL;
    (void)opaque;
    # 根据传入的模式参数确定文件打开模式
    if ((mode & ZLIB_FILEFUNC_MODE_READWRITEFILTER)==ZLIB_FILEFUNC_MODE_READ)
        mode_fopen = "rb";
    else
    if (mode & ZLIB_FILEFUNC_MODE_EXISTING)
        mode_fopen = "r+b";
    else
    if (mode & ZLIB_FILEFUNC_MODE_CREATE)
        mode_fopen = "wb";

    # 如果文件名和打开模式都不为空，则打开文件
    if ((filename!=NULL) && (mode_fopen != NULL))
        file = fopen(filename, mode_fopen);
    return file;
}

# 定义一个函数，用于以指定模式打开文件（64位版本）
static voidpf ZCALLBACK fopen64_file_func (voidpf opaque, const void* filename, int mode)
{
    FILE* file = NULL;
    const char* mode_fopen = NULL;
    (void)opaque;
    # 根据传入的模式参数确定文件打开模式
    if ((mode & ZLIB_FILEFUNC_MODE_READWRITEFILTER)==ZLIB_FILEFUNC_MODE_READ)
        mode_fopen = "rb";
    else
    if (mode & ZLIB_FILEFUNC_MODE_EXISTING)
        mode_fopen = "r+b";
    else
    if (mode & ZLIB_FILEFUNC_MODE_CREATE)
        mode_fopen = "wb";

    # 如果文件名和打开模式都不为空，则打开文件
    if ((filename!=NULL) && (mode_fopen != NULL))
        file = FOPEN_FUNC((const char*)filename, mode_fopen);
    return file;
}

# 定义一个函数，用于从文件中读取数据
static uLong ZCALLBACK fread_file_func (voidpf opaque, voidpf stream, void* buf, uLong size)
{
    uLong ret;
    (void)opaque;
    # 从文件流中读取数据到缓冲区
    ret = (uLong)fread(buf, 1, (size_t)size, (FILE *)stream);
    return ret;
}

# 定义一个函数，用于向文件中写入数据
static uLong ZCALLBACK fwrite_file_func (voidpf opaque, voidpf stream, const void* buf, uLong size)
{
    uLong ret;
    (void)opaque;
    # 将数据从缓冲区写入文件流
    ret = (uLong)fwrite(buf, 1, (size_t)size, (FILE *)stream);
    return ret;
}

# 定义一个函数，用于获取文件当前的读写位置
static long ZCALLBACK ftell_file_func (voidpf opaque, voidpf stream)
{
    long ret;
    (void)opaque;
    # 获取文件流的当前读写位置
    ret = ftell((FILE *)stream);
    return ret;
}

# 定义一个函数，用于获取文件当前的读写位置（64位版本）
static ZPOS64_T ZCALLBACK ftell64_file_func (voidpf opaque, voidpf stream)
{
    ZPOS64_T ret;
    (void)opaque;
    # 获取文件流的当前读写位置（64位版本）
    ret = (ZPOS64_T)FTELLO_FUNC((FILE *)stream);
    return ret;
}
static long ZCALLBACK fseek_file_func (voidpf  opaque, voidpf stream, uLong offset, int origin)
{
    int fseek_origin=0;  // 初始化文件指针偏移量
    long ret;  // 返回值
    (void)opaque;  // 忽略参数
    switch (origin)  // 根据 origin 参数选择不同的偏移方式
    {
    case ZLIB_FILEFUNC_SEEK_CUR :  // 当前位置偏移
        fseek_origin = SEEK_CUR;
        break;
    case ZLIB_FILEFUNC_SEEK_END :  // 末尾位置偏移
        fseek_origin = SEEK_END;
        break;
    case ZLIB_FILEFUNC_SEEK_SET :  // 起始位置偏移
        fseek_origin = SEEK_SET;
        break;
    default: return -1;  // 默认返回错误
    }
    ret = 0;  // 初始化返回值
    if (fseek((FILE *)stream, (long)offset, fseek_origin) != 0)  // 根据偏移方式和偏移量设置文件指针位置
        ret = -1;  // 设置返回值为错误
    return ret;  // 返回结果
}

static long ZCALLBACK fseek64_file_func (voidpf  opaque, voidpf stream, ZPOS64_T offset, int origin)
{
    int fseek_origin=0;  // 初始化文件指针偏移量
    long ret;  // 返回值
    (void)opaque;  // 忽略参数
    switch (origin)  // 根据 origin 参数选择不同的偏移方式
    {
    case ZLIB_FILEFUNC_SEEK_CUR :  // 当前位置偏移
        fseek_origin = SEEK_CUR;
        break;
    case ZLIB_FILEFUNC_SEEK_END :  // 末尾位置偏移
        fseek_origin = SEEK_END;
        break;
    case ZLIB_FILEFUNC_SEEK_SET :  // 起始位置偏移
        fseek_origin = SEEK_SET;
        break;
    default: return -1;  // 默认返回错误
    }
    ret = 0;  // 初始化返回值
    if(FSEEKO_FUNC((FILE *)stream, (z_off_t)offset, fseek_origin) != 0)  // 根据偏移方式和偏移量设置文件指针位置
        ret = -1;  // 设置返回值为错误
    return ret;  // 返回结果
}

static int ZCALLBACK fclose_file_func (voidpf opaque, voidpf stream)
{
    int ret;  // 返回值
    (void)opaque;  // 忽略参数
    ret = fclose((FILE *)stream);  // 关闭文件
    return ret;  // 返回结果
}

static int ZCALLBACK ferror_file_func (voidpf opaque, voidpf stream)
{
    int ret;  // 返回值
    (void)opaque;  // 忽略参数
    ret = ferror((FILE *)stream);  // 获取文件错误状态
    return ret;  // 返回结果
}

void fill_fopen_filefunc (pzlib_filefunc_def)
  zlib_filefunc_def* pzlib_filefunc_def;
{
    pzlib_filefunc_def->zopen_file = fopen_file_func;  // 设置打开文件函数
    pzlib_filefunc_def->zread_file = fread_file_func;  // 设置读取文件函数
    pzlib_filefunc_def->zwrite_file = fwrite_file_func;  // 设置写入文件函数
    pzlib_filefunc_def->ztell_file = ftell_file_func;  // 设置获取文件位置函数
    pzlib_filefunc_def->zseek_file = fseek_file_func;  // 设置设置文件位置函数
    pzlib_filefunc_def->zclose_file = fclose_file_func;  // 设置关闭文件函数
    pzlib_filefunc_def->zerror_file = ferror_file_func;  // 设置获取文件错误状态函数
    pzlib_filefunc_def->opaque = NULL;  // 设置参数为空
}
# 填充 zlib_filefunc64_def 结构体的函数指针，用于操作文件
void fill_fopen64_filefunc (zlib_filefunc64_def*  pzlib_filefunc_def)
{
    # 设置打开文件的函数指针
    pzlib_filefunc_def->zopen64_file = fopen64_file_func;
    # 设置读取文件的函数指针
    pzlib_filefunc_def->zread_file = fread_file_func;
    # 设置写入文件的函数指针
    pzlib_filefunc_def->zwrite_file = fwrite_file_func;
    # 设置获取文件位置的函数指针
    pzlib_filefunc_def->ztell64_file = ftell64_file_func;
    # 设置设置文件位置的函数指针
    pzlib_filefunc_def->zseek64_file = fseek64_file_func;
    # 设置关闭文件的函数指针
    pzlib_filefunc_def->zclose_file = fclose_file_func;
    # 设置获取文件错误的函数指针
    pzlib_filefunc_def->zerror_file = ferror_file_func;
    # 设置透明指针为空
    pzlib_filefunc_def->opaque = NULL;
}
```