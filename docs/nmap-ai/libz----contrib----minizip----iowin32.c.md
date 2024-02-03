# `nmap\libz\contrib\minizip\iowin32.c`

```cpp
/*
    该文件是用于压缩/解压缩.zip文件的IO基本函数的头文件
    版本1.1，2010年2月14日
    MiniZip项目的一部分 - (http://www.winimage.com/zLibDll/minizip.html)

    版权所有（C）1998-2010 Gilles Vollant（minizip）（http://www.winimage.com/zLibDll/minizip.html）

    用于Zip64支持的修改
    版权所有（C）2009-2010 Mathias Svensson（http://result42.com）

    有关更多信息，请阅读MiniZip_info.txt
*/

#include <stdlib.h>  // 包含标准库头文件

#include "zlib.h"  // 包含zlib压缩库头文件
#include "ioapi.h"  // 包含ioapi头文件
#include "iowin32.h"  // 包含iowin32头文件

#ifndef INVALID_HANDLE_VALUE
#define INVALID_HANDLE_VALUE (0xFFFFFFFF)  // 如果未定义INVALID_HANDLE_VALUE，则定义为0xFFFFFFFF
#endif

#ifndef INVALID_SET_FILE_POINTER
#define INVALID_SET_FILE_POINTER ((DWORD)-1)  // 如果未定义INVALID_SET_FILE_POINTER，则定义为((DWORD)-1)
#endif

// 查看Windows Kit中的Include/shared/winapifamily.h
#if defined(WINAPI_FAMILY_PARTITION) && (!(defined(IOWIN32_USING_WINRT_API)))

#if !defined(WINAPI_FAMILY_ONE_PARTITION)
#define WINAPI_FAMILY_ONE_PARTITION(PartitionSet, Partition) ((WINAPI_FAMILY & PartitionSet) == Partition)
#endif

#if WINAPI_FAMILY_ONE_PARTITION(WINAPI_FAMILY, WINAPI_PARTITION_APP)
#define IOWIN32_USING_WINRT_API 1  // 如果WINAPI_FAMILY和WINAPI_PARTITION_APP在同一分区，则定义IOWIN32_USING_WINRT_API为1
#endif
#endif

// 定义win32_open_file_func函数
voidpf  ZCALLBACK win32_open_file_func  OF((voidpf opaque, const char* filename, int mode));
// 定义win32_read_file_func函数
uLong   ZCALLBACK win32_read_file_func  OF((voidpf opaque, voidpf stream, void* buf, uLong size));
// 定义win32_write_file_func函数
uLong   ZCALLBACK win32_write_file_func OF((voidpf opaque, voidpf stream, const void* buf, uLong size));
// 定义win32_tell64_file_func函数
ZPOS64_T ZCALLBACK win32_tell64_file_func  OF((voidpf opaque, voidpf stream));
// 定义win32_seek64_file_func函数
long    ZCALLBACK win32_seek64_file_func  OF((voidpf opaque, voidpf stream, ZPOS64_T offset, int origin));
// 定义win32_close_file_func函数
int     ZCALLBACK win32_close_file_func OF((voidpf opaque, voidpf stream));
// 定义win32_error_file_func函数
int     ZCALLBACK win32_error_file_func OF((voidpf opaque, voidpf stream));

// 定义WIN32FILE_IOWIN结构体
typedef struct
{
    HANDLE hf;  // 文件句柄
    int error;  // 错误码
} WIN32FILE_IOWIN;
# 根据传入的打开模式转换为 Windows 平台下的文件操作参数
static void win32_translate_open_mode(int mode,
                                      DWORD* lpdwDesiredAccess,
                                      DWORD* lpdwCreationDisposition,
                                      DWORD* lpdwShareMode,
                                      DWORD* lpdwFlagsAndAttributes)
{
    # 将所有参数初始化为 0
    *lpdwDesiredAccess = *lpdwShareMode = *lpdwFlagsAndAttributes = *lpdwCreationDisposition = 0;

    # 如果是读取模式
    if ((mode & ZLIB_FILEFUNC_MODE_READWRITEFILTER)==ZLIB_FILEFUNC_MODE_READ)
    {
        *lpdwDesiredAccess = GENERIC_READ;  # 设置为读取权限
        *lpdwCreationDisposition = OPEN_EXISTING;  # 打开已存在的文件
        *lpdwShareMode = FILE_SHARE_READ;  # 设置为可共享读取
    }
    # 如果是已存在模式
    else if (mode & ZLIB_FILEFUNC_MODE_EXISTING)
    {
        *lpdwDesiredAccess = GENERIC_WRITE | GENERIC_READ;  # 设置为读写权限
        *lpdwCreationDisposition = OPEN_EXISTING;  # 打开已存在的文件
    }
    # 如果是创建模式
    else if (mode & ZLIB_FILEFUNC_MODE_CREATE)
    {
        *lpdwDesiredAccess = GENERIC_WRITE | GENERIC_READ;  # 设置为读写权限
        *lpdwCreationDisposition = CREATE_ALWAYS;  # 创建新文件或覆盖已存在的文件
    }
}

# 构建 Windows 平台下的文件操作结构
static voidpf win32_build_iowin(HANDLE hFile)
{
    voidpf ret=NULL;

    # 如果文件句柄有效
    if ((hFile != NULL) && (hFile != INVALID_HANDLE_VALUE))
    {
        WIN32FILE_IOWIN w32fiow;
        w32fiow.hf = hFile;
        w32fiow.error = 0;
        ret = malloc(sizeof(WIN32FILE_IOWIN));

        # 如果内存分配失败，则关闭文件句柄
        if (ret==NULL)
            CloseHandle(hFile);
        else
            *((WIN32FILE_IOWIN*)ret) = w32fiow;
    }
    return ret;
}

# 用于在 Windows 平台下打开文件的回调函数
voidpf ZCALLBACK win32_open64_file_func (voidpf opaque,const void* filename,int mode)
{
    const char* mode_fopen = NULL;
    DWORD dwDesiredAccess,dwCreationDisposition,dwShareMode,dwFlagsAndAttributes ;
    HANDLE hFile = NULL;

    # 根据传入的打开模式转换为 Windows 平台下的文件操作参数
    win32_translate_open_mode(mode,&dwDesiredAccess,&dwCreationDisposition,&dwShareMode,&dwFlagsAndAttributes);

    # 如果使用 WinRT API 并且文件名和权限有效，则调用 CreateFile2 打开文件
    if ((filename!=NULL) && (dwDesiredAccess != 0))
        hFile = CreateFile2((LPCTSTR)filename, dwDesiredAccess, dwShareMode, dwCreationDisposition, NULL);
    # 定义一个宽字符数组，用于存储文件名
    WCHAR filenameW[FILENAME_MAX + 0x200 + 1];
    # 将多字节字符转换为宽字符，存储到filenameW数组中
    MultiByteToWideChar(CP_ACP,0,(const char*)filename,-1,filenameW,FILENAME_MAX + 0x200);
    # 使用宽字符文件名创建或打开文件，并返回文件句柄
    hFile = CreateFile2(filenameW, dwDesiredAccess, dwShareMode, dwCreationDisposition, NULL);
#endif
#else
    // 如果文件名不为空且期望访问权限不为0，则创建文件句柄
    if ((filename!=NULL) && (dwDesiredAccess != 0))
        hFile = CreateFile((LPCTSTR)filename, dwDesiredAccess, dwShareMode, NULL, dwCreationDisposition, dwFlagsAndAttributes, NULL);
#endif

    // 返回创建的文件句柄
    return win32_build_iowin(hFile);
}


voidpf ZCALLBACK win32_open64_file_funcA (voidpf opaque,const void* filename,int mode)
{
    const char* mode_fopen = NULL;
    DWORD dwDesiredAccess,dwCreationDisposition,dwShareMode,dwFlagsAndAttributes ;
    HANDLE hFile = NULL;

    // 根据打开模式翻译出期望的访问权限、创建方式、共享模式和属性标志
    win32_translate_open_mode(mode,&dwDesiredAccess,&dwCreationDisposition,&dwShareMode,&dwFlagsAndAttributes);

#ifdef IOWIN32_USING_WINRT_API
    // 如果文件名不为空且期望访问权限不为0，则创建文件句柄
    if ((filename!=NULL) && (dwDesiredAccess != 0))
    {
        // 将文件名从多字节转换为宽字符
        WCHAR filenameW[FILENAME_MAX + 0x200 + 1];
        MultiByteToWideChar(CP_ACP,0,(const char*)filename,-1,filenameW,FILENAME_MAX + 0x200);
        hFile = CreateFile2(filenameW, dwDesiredAccess, dwShareMode, dwCreationDisposition, NULL);
    }
#else
    // 如果文件名不为空且期望访问权限不为0，则创建文件句柄
    if ((filename!=NULL) && (dwDesiredAccess != 0))
        hFile = CreateFileA((LPCSTR)filename, dwDesiredAccess, dwShareMode, NULL, dwCreationDisposition, dwFlagsAndAttributes, NULL);
#endif

    // 返回创建的文件句柄
    return win32_build_iowin(hFile);
}


voidpf ZCALLBACK win32_open64_file_funcW (voidpf opaque,const void* filename,int mode)
{
    const char* mode_fopen = NULL;
    DWORD dwDesiredAccess,dwCreationDisposition,dwShareMode,dwFlagsAndAttributes ;
    HANDLE hFile = NULL;

    // 根据打开模式翻译出期望的访问权限、创建方式、共享模式和属性标志
    win32_translate_open_mode(mode,&dwDesiredAccess,&dwCreationDisposition,&dwShareMode,&dwFlagsAndAttributes);

#ifdef IOWIN32_USING_WINRT_API
    // 如果文件名不为空且期望访问权限不为0，则创建文件句柄
    if ((filename!=NULL) && (dwDesiredAccess != 0))
        hFile = CreateFile2((LPCWSTR)filename, dwDesiredAccess, dwShareMode, dwCreationDisposition,NULL);
#else
    // 如果文件名不为空且期望访问权限不为0，则创建文件句柄
    if ((filename!=NULL) && (dwDesiredAccess != 0))
        hFile = CreateFileW((LPCWSTR)filename, dwDesiredAccess, dwShareMode, NULL, dwCreationDisposition, dwFlagsAndAttributes, NULL);
#endif

    // 返回创建的文件句柄
    return win32_build_iowin(hFile);
}
voidpf ZCALLBACK win32_open_file_func (voidpf opaque,const char* filename,int mode)
{
    const char* mode_fopen = NULL;  // 声明一个指向字符常量的指针，用于存储文件打开模式
    DWORD dwDesiredAccess,dwCreationDisposition,dwShareMode,dwFlagsAndAttributes ;  // 声明四个 DWORD 类型的变量，用于存储文件操作的参数
    HANDLE hFile = NULL;  // 声明一个 HANDLE 类型的变量，用于存储文件句柄

    win32_translate_open_mode(mode,&dwDesiredAccess,&dwCreationDisposition,&dwShareMode,&dwFlagsAndAttributes);  // 调用函数，将文件打开模式转换为相应的参数值

#ifdef IOWIN32_USING_WINRT_API
#ifdef UNICODE
    if ((filename!=NULL) && (dwDesiredAccess != 0))  // 如果文件名和操作参数都不为空
        hFile = CreateFile2((LPCTSTR)filename, dwDesiredAccess, dwShareMode, dwCreationDisposition, NULL);  // 调用 CreateFile2 函数打开文件
#else
    if ((filename!=NULL) && (dwDesiredAccess != 0))  // 如果文件名和操作参数都不为空
    {
        WCHAR filenameW[FILENAME_MAX + 0x200 + 1];  // 声明一个宽字符数组，用于存储转换后的文件名
        MultiByteToWideChar(CP_ACP,0,(const char*)filename,-1,filenameW,FILENAME_MAX + 0x200);  // 将文件名从多字节转换为宽字符
        hFile = CreateFile2(filenameW, dwDesiredAccess, dwShareMode, dwCreationDisposition, NULL);  // 调用 CreateFile2 函数打开文件
    }
#endif
#else
    if ((filename!=NULL) && (dwDesiredAccess != 0))  // 如果文件名和操作参数都不为空
        hFile = CreateFile((LPCTSTR)filename, dwDesiredAccess, dwShareMode, NULL, dwCreationDisposition, dwFlagsAndAttributes, NULL);  // 调用 CreateFile 函数打开文件
#endif

    return win32_build_iowin(hFile);  // 返回创建的文件操作对象
}


uLong ZCALLBACK win32_read_file_func (voidpf opaque, voidpf stream, void* buf,uLong size)
{
    uLong ret=0;  // 声明一个 uLong 类型的变量，用于存储读取的字节数
    HANDLE hFile = NULL;  // 声明一个 HANDLE 类型的变量，用于存储文件句柄
    if (stream!=NULL)  // 如果文件操作对象不为空
        hFile = ((WIN32FILE_IOWIN*)stream) -> hf;  // 获取文件句柄

    if (hFile != NULL)  // 如果文件句柄不为空
    {
        if (!ReadFile(hFile, buf, size, &ret, NULL))  // 如果读取文件失败
        {
            DWORD dwErr = GetLastError();  // 获取错误码
            if (dwErr == ERROR_HANDLE_EOF)  // 如果错误码表示已到达文件末尾
                dwErr = 0;  // 将错误码置为 0
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;  // 将错误码存储到文件操作对象中
        }
    }

    return ret;  // 返回读取的字节数
}


uLong ZCALLBACK win32_write_file_func (voidpf opaque,voidpf stream,const void* buf,uLong size)
{
    uLong ret=0;  // 声明一个 uLong 类型的变量，用于存储写入的字节数
    HANDLE hFile = NULL;  // 声明一个 HANDLE 类型的变量，用于存储文件句柄
    if (stream!=NULL)  // 如果文件操作对象不为空
        hFile = ((WIN32FILE_IOWIN*)stream) -> hf;  // 获取文件句柄
    {
        // 如果写入文件失败
        if (!WriteFile(hFile, buf, size, &ret, NULL))
        {
            // 获取错误码
            DWORD dwErr = GetLastError();
            // 如果错误码为文件结束标识，将错误码置为0
            if (dwErr == ERROR_HANDLE_EOF)
                dwErr = 0;
            // 将错误码转换为整数，存储在stream的error属性中
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;
        }
    }

    // 返回写入的字节数
    return ret;
}
# 定义静态函数 MySetFilePointerEx，用于在 Windows 平台下设置文件指针位置
static BOOL MySetFilePointerEx(HANDLE hFile, LARGE_INTEGER pos, LARGE_INTEGER *newPos,  DWORD dwMoveMethod)
{
    # 如果使用 WinRT API，则调用 SetFilePointerEx 函数
    # 否则，使用 SetFilePointer 函数设置文件指针位置
#ifdef IOWIN32_USING_WINRT_API
    return SetFilePointerEx(hFile, pos, newPos, dwMoveMethod);
#else
    LONG lHigh = pos.HighPart;
    DWORD dwNewPos = SetFilePointer(hFile, pos.LowPart, &lHigh, dwMoveMethod);
    BOOL fOk = TRUE;
    # 如果设置文件指针位置失败，则返回 FALSE
    if (dwNewPos == 0xFFFFFFFF)
        if (GetLastError() != NO_ERROR)
            fOk = FALSE;
    # 如果新位置不为空且设置成功，则更新新位置信息
    if ((newPos != NULL) && (fOk))
    {
        newPos->LowPart = dwNewPos;
        newPos->HighPart = lHigh;
    }
    return fOk;
#endif
}

# 定义函数 win32_tell_file_func，用于获取文件当前位置
long ZCALLBACK win32_tell_file_func (voidpf opaque,voidpf stream)
{
    long ret=-1;
    HANDLE hFile = NULL;
    # 如果流不为空，则获取文件句柄
    if (stream!=NULL)
        hFile = ((WIN32FILE_IOWIN*)stream) -> hf;
    # 如果文件句柄不为空，则获取文件当前位置
    if (hFile != NULL)
    {
        LARGE_INTEGER pos;
        pos.QuadPart = 0;
        # 调用 MySetFilePointerEx 函数设置文件指针位置
        if (!MySetFilePointerEx(hFile, pos, &pos, FILE_CURRENT))
        {
            DWORD dwErr = GetLastError();
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;
            ret = -1;
        }
        else
            ret=(long)pos.LowPart;
    }
    return ret;
}

# 定义函数 win32_tell64_file_func，用于获取文件当前位置（64 位）
ZPOS64_T ZCALLBACK win32_tell64_file_func (voidpf opaque, voidpf stream)
{
    ZPOS64_T ret= (ZPOS64_T)-1;
    HANDLE hFile = NULL;
    # 如果流不为空，则获取文件句柄
    if (stream!=NULL)
        hFile = ((WIN32FILE_IOWIN*)stream)->hf;
    # 如果文件句柄不为空，则获取文件当前位置（64 位）
    if (hFile)
    {
        LARGE_INTEGER pos;
        pos.QuadPart = 0;
        # 调用 MySetFilePointerEx 函数设置文件指针位置
        if (!MySetFilePointerEx(hFile, pos, &pos, FILE_CURRENT))
        {
            DWORD dwErr = GetLastError();
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;
            ret = (ZPOS64_T)-1;
        }
        else
            ret=pos.QuadPart;
    }
    return ret;
}

# 定义函数 win32_seek_file_func，用于设置文件指针位置
long ZCALLBACK win32_seek_file_func (voidpf opaque,voidpf stream,uLong offset,int origin)
{
    DWORD dwMoveMethod=0xFFFFFFFF;
    HANDLE hFile = NULL;

    long ret=-1;
    # 如果流不为空，则获取文件句柄
    if (stream!=NULL)
        hFile = ((WIN32FILE_IOWIN*)stream) -> hf;
    # 根据 origin 参数确定移动方式
    switch (origin)
    {
    # 根据传入的偏移参数设置文件指针的移动方式
    case ZLIB_FILEFUNC_SEEK_CUR :
        dwMoveMethod = FILE_CURRENT;
        break;
    # 根据传入的偏移参数设置文件指针的移动方式
    case ZLIB_FILEFUNC_SEEK_END :
        dwMoveMethod = FILE_END;
        break;
    # 根据传入的偏移参数设置文件指针的移动方式
    case ZLIB_FILEFUNC_SEEK_SET :
        dwMoveMethod = FILE_BEGIN;
        break;
    # 如果传入的偏移参数不在预期范围内，返回错误
    default: return -1;
    }

    # 如果文件句柄不为空
    if (hFile != NULL)
    {
        # 创建一个大整数对象，用于设置文件指针的位置
        LARGE_INTEGER pos;
        pos.QuadPart = offset;
        # 使用 MySetFilePointerEx 函数设置文件指针的位置
        if (!MySetFilePointerEx(hFile, pos, NULL, dwMoveMethod))
        {
            # 如果设置文件指针位置失败，获取错误码并将错误码存储到流对象中
            DWORD dwErr = GetLastError();
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;
            ret = -1;
        }
        else
            # 如果设置文件指针位置成功，返回 0
            ret=0;
    }
    # 返回设置文件指针位置的结果
    return ret;
# 定义一个回调函数，用于在 Win32 平台下实现文件定位
long ZCALLBACK win32_seek64_file_func (voidpf opaque, voidpf stream,ZPOS64_T offset,int origin)
{
    # 初始化变量
    DWORD dwMoveMethod=0xFFFFFFFF;
    HANDLE hFile = NULL;
    long ret=-1;

    # 如果流不为空，则获取文件句柄
    if (stream!=NULL)
        hFile = ((WIN32FILE_IOWIN*)stream)->hf;

    # 根据 origin 参数确定文件指针的移动方式
    switch (origin)
    {
        case ZLIB_FILEFUNC_SEEK_CUR :
            dwMoveMethod = FILE_CURRENT;
            break;
        case ZLIB_FILEFUNC_SEEK_END :
            dwMoveMethod = FILE_END;
            break;
        case ZLIB_FILEFUNC_SEEK_SET :
            dwMoveMethod = FILE_BEGIN;
            break;
        default: return -1;
    }

    # 如果文件句柄存在，则根据指定的偏移量和移动方式移动文件指针
    if (hFile)
    {
        LARGE_INTEGER pos;
        pos.QuadPart = offset;
        if (!MySetFilePointerEx(hFile, pos, NULL, dwMoveMethod))
        {
            # 获取错误码并设置到流的 error 属性中
            DWORD dwErr = GetLastError();
            ((WIN32FILE_IOWIN*)stream) -> error=(int)dwErr;
            ret = -1;
        }
        else
            ret=0;
    }
    return ret;
}

# 定义一个回调函数，用于在 Win32 平台下实现文件关闭
int ZCALLBACK win32_close_file_func (voidpf opaque, voidpf stream)
{
    int ret=-1;

    # 如果流不为空，则关闭文件句柄并释放内存
    if (stream!=NULL)
    {
        HANDLE hFile;
        hFile = ((WIN32FILE_IOWIN*)stream) -> hf;
        if (hFile != NULL)
        {
            CloseHandle(hFile);
            ret=0;
        }
        free(stream);
    }
    return ret;
}

# 定义一个回调函数，用于在 Win32 平台下获取文件操作的错误码
int ZCALLBACK win32_error_file_func (voidpf opaque,voidpf stream)
{
    int ret=-1;
    # 如果流不为空，则获取流的 error 属性作为返回值
    if (stream!=NULL)
    {
        ret = ((WIN32FILE_IOWIN*)stream) -> error;
    }
    return ret;
}

# 定义一个函数，用于填充 Win32 平台下的文件操作函数指针结构体
void fill_win32_filefunc (zlib_filefunc_def* pzlib_filefunc_def)
{
    # 分别将文件操作函数指针赋值给对应的成员变量
    pzlib_filefunc_def->zopen_file = win32_open_file_func;
    pzlib_filefunc_def->zread_file = win32_read_file_func;
    pzlib_filefunc_def->zwrite_file = win32_write_file_func;
    pzlib_filefunc_def->ztell_file = win32_tell_file_func;
    pzlib_filefunc_def->zseek_file = win32_seek_file_func;
    pzlib_filefunc_def->zclose_file = win32_close_file_func;
    pzlib_filefunc_def->zerror_file = win32_error_file_func;
    pzlib_filefunc_def->opaque = NULL;
}
# 填充 win32_filefunc64_def 结构体的函数，设置文件操作函数指针
void fill_win32_filefunc64(zlib_filefunc64_def* pzlib_filefunc_def)
{
    # 设置打开文件的函数指针为 win32_open64_file_func
    pzlib_filefunc_def->zopen64_file = win32_open64_file_func;
    # 设置读取文件的函数指针为 win32_read_file_func
    pzlib_filefunc_def->zread_file = win32_read_file_func;
    # 设置写入文件的函数指针为 win32_write_file_func
    pzlib_filefunc_def->zwrite_file = win32_write_file_func;
    # 设置获取文件位置的函数指针为 win32_tell64_file_func
    pzlib_filefunc_def->ztell64_file = win32_tell64_file_func;
    # 设置设置文件位置的函数指针为 win32_seek64_file_func
    pzlib_filefunc_def->zseek64_file = win32_seek64_file_func;
    # 设置关闭文件的函数指针为 win32_close_file_func
    pzlib_filefunc_def->zclose_file = win32_close_file_func;
    # 设置文件错误处理的函数指针为 win32_error_file_func
    pzlib_filefunc_def->zerror_file = win32_error_file_func;
    # 设置 opaque 指针为空
    pzlib_filefunc_def->opaque = NULL;
}

# 填充 win32_filefunc64_def 结构体的函数，设置文件操作函数指针
void fill_win32_filefunc64A(zlib_filefunc64_def* pzlib_filefunc_def)
{
    # 设置打开文件的函数指针为 win32_open64_file_funcA
    pzlib_filefunc_def->zopen64_file = win32_open64_file_funcA;
    # 设置读取文件的函数指针为 win32_read_file_func
    pzlib_filefunc_def->zread_file = win32_read_file_func;
    # 设置写入文件的函数指针为 win32_write_file_func
    pzlib_filefunc_def->zwrite_file = win32_write_file_func;
    # 设置获取文件位置的函数指针为 win32_tell64_file_func
    pzlib_filefunc_def->ztell64_file = win32_tell64_file_func;
    # 设置设置文件位置的函数指针为 win32_seek64_file_func
    pzlib_filefunc_def->zseek64_file = win32_seek64_file_func;
    # 设置关闭文件的函数指针为 win32_close_file_func
    pzlib_filefunc_def->zclose_file = win32_close_file_func;
    # 设置文件错误处理的函数指针为 win32_error_file_func
    pzlib_filefunc_def->zerror_file = win32_error_file_func;
    # 设置 opaque 指针为空
    pzlib_filefunc_def->opaque = NULL;
}

# 填充 win32_filefunc64_def 结构体的函数，设置文件操作函数指针
void fill_win32_filefunc64W(zlib_filefunc64_def* pzlib_filefunc_def)
{
    # 设置打开文件的函数指针为 win32_open64_file_funcW
    pzlib_filefunc_def->zopen64_file = win32_open64_file_funcW;
    # 设置读取文件的函数指针为 win32_read_file_func
    pzlib_filefunc_def->zread_file = win32_read_file_func;
    # 设置写入文件的函数指针为 win32_write_file_func
    pzlib_filefunc_def->zwrite_file = win32_write_file_func;
    # 设置获取文件位置的函数指针为 win32_tell64_file_func
    pzlib_filefunc_def->ztell64_file = win32_tell64_file_func;
    # 设置设置文件位置的函数指针为 win32_seek64_file_func
    pzlib_filefunc_def->zseek64_file = win32_seek64_file_func;
    # 设置关闭文件的函数指针为 win32_close_file_func
    pzlib_filefunc_def->zclose_file = win32_close_file_func;
    # 设置文件错误处理的函数指针为 win32_error_file_func
    pzlib_filefunc_def->zerror_file = win32_error_file_func;
    # 设置 opaque 指针为空
    pzlib_filefunc_def->opaque = NULL;
}
```