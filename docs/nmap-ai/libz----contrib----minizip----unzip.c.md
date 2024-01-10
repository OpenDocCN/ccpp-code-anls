# `nmap\libz\contrib\minizip\unzip.c`

```
/*
   以上是一些头文件的引入和宏定义的设置
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#ifndef NOUNCRYPT
        #define NOUNCRYPT
#endif

#include "zlib.h"
#include "unzip.h"

#ifdef STDC
#  include <stddef.h>
#  include <string.h>
#  include <stdlib.h>
#endif
#ifdef NO_ERRNO_H
    extern int errno;
#else
#   include <errno.h>
#endif

/*
   以上是一些条件编译和头文件的引入
*/

#ifndef local
#  define local static
#endif
/* compile with -Dlocal if your debugger can't find static symbols */

/*
   定义了一个静态的宏，用于在调试器中找到静态符号
*/

#ifndef CASESENSITIVITYDEFAULT_NO
#  if !defined(unix) && !defined(CASESENSITIVITYDEFAULT_YES)
#    define CASESENSITIVITYDEFAULT_NO
#  endif
#endif

/*
   设置了文件名大小写敏感性的默认值
*/

#ifndef UNZ_BUFSIZE
#define UNZ_BUFSIZE (16384)
#endif

#ifndef UNZ_MAXFILENAMEINZIP
#define UNZ_MAXFILENAMEINZIP (256)
#endif

#ifndef ALLOC
# define ALLOC(size) (malloc(size))
#endif
#ifndef TRYFREE
# define TRYFREE(p) { free(p);}
#endif

#define SIZECENTRALDIRITEM (0x2e)
#define SIZEZIPLOCALHEADER (0x1e)

/*
   设置了一些常量和默认缓冲区大小
*/

const char unz_copyright[] =
   " unzip 1.01 Copyright 1998-2004 Gilles Vollant - http://www.winimage.com/zLibDll";

/*
   定义了一个版权信息的字符串
*/

/* unz_file_info_interntal contain internal info about a file in zipfile*/
typedef struct unz_file_info64_internal_s
{
    ZPOS64_T offset_curfile;/* relative offset of local header 8 bytes */
} unz_file_info64_internal;

/*
   定义了一个结构体，包含了关于 ZIP 文件中文件的内部信息
*/

/* file_in_zip_read_info_s contain internal information about a file in zipfile,
    when reading and decompress it */
typedef struct
{
    char  *read_buffer;         /* internal buffer for compressed data */
    z_stream stream;            /* zLib stream structure for inflate */

#ifdef HAVE_BZIP2
    bz_stream bstream;          /* bzLib stream structure for bziped */
#endif

    ZPOS64_T pos_in_zipfile;       /* position in byte on the zipfile, for fseek*/
    uLong stream_initialised;   /* flag set if stream structure is initialised*/

    ZPOS64_T offset_local_extrafield;/* offset of the local extra field */
    uInt  size_local_extrafield;/* size of the local extra field */

/*
   定义了一个结构体，包含了在读取和解压缩 ZIP 文件中文件时的内部信息
*/
    # 本地额外字段中的位置
    ZPOS64_T pos_local_extrafield;   /* position in the local extra field in read*/
    # 64位总输出
    ZPOS64_T total_out_64;

    # 未压缩数据的 crc32
    uLong crc32;                /* crc32 of all data uncompressed */
    # 等待解压缩后必须获得的 crc32
    uLong crc32_wait;           /* crc32 we must obtain after decompress all */
    # 剩余待解压缩的压缩数据字节数
    ZPOS64_T rest_read_compressed; /* number of byte to be decompressed */
    # 解压缩后需要获取的未压缩数据字节数
    ZPOS64_T rest_read_uncompressed;/*number of byte to be obtained after decomp*/
    # zlib 文件操作结构体
    zlib_filefunc64_32_def z_filefunc;
    # ZIP 文件流
    voidpf filestream;        /* io structore of the zipfile */
    # 压缩方法（0==store）
    uLong compression_method;   /* compression method (0==store) */
    # ZIP 文件之前的字节数（对于 sfx 文件 >0）
    ZPOS64_T byte_before_the_zipfile;/* byte before the zipfile, (>0 for sfx)*/
    # 原始数据
    int   raw;
} file_in_zip64_read_info_s;  // 定义了一个结构体 file_in_zip64_read_info_s

/* unz64_s contain internal information about the zipfile
*/
typedef struct
{
    zlib_filefunc64_32_def z_filefunc;  // 结构体成员，用于处理文件操作
    int is64bitOpenFunction;  // 用于标识是否为64位打开函数
    voidpf filestream;        /* io structore of the zipfile */  // 文件流指针，指向zip文件的IO结构
    unz_global_info64 gi;       /* public global information */  // 公共全局信息
    ZPOS64_T byte_before_the_zipfile;/* byte before the zipfile, (>0 for sfx)*/  // zipfile之前的字节数
    ZPOS64_T num_file;             /* number of the current file in the zipfile*/  // zipfile中当前文件的数量
    ZPOS64_T pos_in_central_dir;   /* pos of the current file in the central dir*/  // 当前文件在中央目录中的位置
    ZPOS64_T current_file_ok;      /* flag about the usability of the current file*/  // 当前文件可用性的标志
    ZPOS64_T central_pos;          /* position of the beginning of the central dir*/  // 中央目录的起始位置

    ZPOS64_T size_central_dir;     /* size of the central directory  */  // 中央目录的大小
    ZPOS64_T offset_central_dir;   /* offset of start of central directory with
                                   respect to the starting disk number */  // 中央目录相对于起始磁盘号的偏移量

    unz_file_info64 cur_file_info; /* public info about the current file in zip*/  // 当前zip文件的公共信息
    unz_file_info64_internal cur_file_info_internal; /* private info about it*/  // 关于当前文件的私有信息
    file_in_zip64_read_info_s* pfile_in_zip_read; /* structure about the current
                                        file if we are decompressing it */  // 如果正在解压缩当前文件，则关于当前文件的结构

    int encrypted;  // 标识文件是否加密

    int isZip64;  // 标识是否为Zip64格式

#    ifndef NOUNCRYPT
    unsigned long keys[3];     /* keys defining the pseudo-random sequence */  // 定义伪随机序列的密钥
    const z_crc_t* pcrc_32_tab;  // CRC32表
#    endif
} unz64_s;  // 定义了一个结构体 unz64_s

#ifndef NOUNCRYPT
#include "crypt.h"
#endif

/* ===========================================================================
     Read a byte from a gz_stream; update next_in and avail_in. Return EOF
   for end of file.
   IN assertion: the stream s has been successfully opened for reading.
*/

local int unz64local_getByte OF((
    const zlib_filefunc64_32_def* pzlib_filefunc_def,
    voidpf filestream,
    int *pi));

local int unz64local_getByte(const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, int *pi)
{
    # 声明一个无符号字符变量c
    unsigned char c;
    # 调用ZREAD64函数从filestream中读取一个字符到c中，返回错误码
    int err = (int)ZREAD64(*pzlib_filefunc_def,filestream,&c,1);
    # 如果读取成功
    if (err==1)
    {
        # 将c转换为整数赋值给pi
        *pi = (int)c;
        # 返回无错误状态码
        return UNZ_OK;
    }
    # 如果读取失败
    else
    {
        # 如果filestream发生错误
        if (ZERROR64(*pzlib_filefunc_def,filestream))
            # 返回文件错误状态码
            return UNZ_ERRNO;
        # 如果文件结束
        else
            # 返回文件结束状态码
            return UNZ_EOF;
    }
# 从给定的 gz_stream 中以 LSB 顺序读取一个长整型。设置
local int unz64local_getShort OF((
    const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
    voidpf filestream,  # 文件流指针
    uLong *pX));  # 指向无符号长整型的指针

local int unz64local_getShort (const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
                             voidpf filestream,  # 文件流指针
                             uLong *pX)  # 指向无符号长整型的指针
{
    uLong x ;  # 定义一个无符号长整型变量 x
    int i = 0;  # 定义一个整型变量 i，并初始化为 0
    int err;  # 定义一个整型变量 err

    err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x = (uLong)i;  # 将整型变量 i 转换为无符号长整型，并赋值给 x

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x |= ((uLong)i)<<8;  # 将 i 左移 8 位，然后与 x 按位或操作

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        *pX = x;  # 将 x 的值赋给指针所指向的变量
    else  # 否则
        *pX = 0;  # 将 0 赋给指针所指向的变量
    return err;  # 返回 err 变量的值
}

local int unz64local_getLong OF((
    const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
    voidpf filestream,  # 文件流指针
    uLong *pX));  # 指向无符号长整型的指针

local int unz64local_getLong (const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
                            voidpf filestream,  # 文件流指针
                            uLong *pX)  # 指向无符号长整型的指针
{
    uLong x ;  # 定义一个无符号长整型变量 x
    int i = 0;  # 定义一个整型变量 i，并初始化为 0
    int err;  # 定义一个整型变量 err

    err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x = (uLong)i;  # 将整型变量 i 转换为无符号长整型，并赋值给 x

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x |= ((uLong)i)<<8;  # 将 i 左移 8 位，然后与 x 按位或操作

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x |= ((uLong)i)<<16;  # 将 i 左移 16 位，然后与 x 按位或操作

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);  # 调用 unz64local_getByte 函数，将结果赋值给 err
    x += ((uLong)i)<<24;  # 将 i 左移 24 位，然后与 x 相加

    if (err==UNZ_OK)  # 如果 err 等于 UNZ_OK
        *pX = x;  # 将 x 的值赋给指针所指向的变量
    else  # 否则
        *pX = 0;  # 将 0 赋给指针所指向的变量
    return err;  # 返回 err 变量的值
}

local int unz64local_getLong64 OF((
    const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
    voidpf filestream,  # 文件流指针
    ZPOS64_T *pX));  # 指向 ZPOS64_T 类型的指针

local int unz64local_getLong64 (const zlib_filefunc64_32_def* pzlib_filefunc_def,  # 定义一个指向 zlib_filefunc64_32_def 结构体的指针
                            voidpf filestream,  # 文件流指针
                            ZPOS64_T *pX)  # 指向 ZPOS64_T 类型的指针
{
    ZPOS64_T x ;  # 定义一个 ZPOS64_T 类型的变量 x
    int i = 0;  # 定义一个整型变量 i，并初始化为 0
    int err;  # 定义一个整型变量 err
    # 从文件流中读取一个字节，存入变量err
    err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将i转换为ZPOS64_T类型，存入变量x
    x = (ZPOS64_T)i;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移8位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<8;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移16位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<16;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移24位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<24;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移32位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<32;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移40位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<40;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移48位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<48;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 从文件流中读取一个字节，存入变量err
        err = unz64local_getByte(pzlib_filefunc_def,filestream,&i);
        # 将i左移56位，与x进行按位或操作
        x |= ((ZPOS64_T)i)<<56;

    # 如果读取字节操作成功
    if (err==UNZ_OK)
        # 将x的值存入指针pX指向的位置
        *pX = x;
    else
        # 如果读取字节操作失败，将0存入指针pX指向的位置
        *pX = 0;
    # 返回操作结果
    return err;
/* My own strcmpi / strcasecmp */
// 自定义的不区分大小写的字符串比较函数
local int strcmpcasenosensitive_internal (const char* fileName1, const char* fileName2)
{
    for (;;)
    {
        char c1=*(fileName1++);
        char c2=*(fileName2++);
        if ((c1>='a') && (c1<='z'))
            c1 -= 0x20;
        if ((c2>='a') && (c2<='z'))
            c2 -= 0x20;
        if (c1=='\0')
            return ((c2=='\0') ? 0 : -1);
        if (c2=='\0')
            return 1;
        if (c1<c2)
            return -1;
        if (c1>c2)
            return 1;
    }
}

#ifdef  CASESENSITIVITYDEFAULT_NO
#define CASESENSITIVITYDEFAULTVALUE 2
#else
#define CASESENSITIVITYDEFAULTVALUE 1
#endif

#ifndef STRCMPCASENOSENTIVEFUNCTION
#define STRCMPCASENOSENTIVEFUNCTION strcmpcasenosensitive_internal
#endif

/*
   Compare two filename (fileName1,fileName2).
   If iCaseSenisivity = 1, comparision is case sensitivity (like strcmp)
   If iCaseSenisivity = 2, comparision is not case sensitivity (like strcmpi
                                                                or strcasecmp)
   If iCaseSenisivity = 0, case sensitivity is defaut of your operating system
        (like 1 on Unix, 2 on Windows)

*/
// 比较两个文件名（fileName1, fileName2）
// 如果 iCaseSenisivity = 1，比较区分大小写（类似于 strcmp）
// 如果 iCaseSenisivity = 2，比较不区分大小写（类似于 strcmpi 或 strcasecmp）
// 如果 iCaseSenisivity = 0，大小写敏感性是您操作系统的默认值（类似于 Unix 上的 1，Windows 上的 2）
extern int ZEXPORT unzStringFileNameCompare (const char*  fileName1,
                                                 const char*  fileName2,
                                                 int iCaseSensitivity)
{
    if (iCaseSensitivity==0)
        iCaseSensitivity=CASESENSITIVITYDEFAULTVALUE;

    if (iCaseSensitivity==1)
        return strcmp(fileName1,fileName2);

    return STRCMPCASENOSENTIVEFUNCTION(fileName1,fileName2);
}

#ifndef BUFREADCOMMENT
#define BUFREADCOMMENT (0x400)
#endif

/*
  Locate the Central directory of a zipfile (at the end, just before
    the global comment)
*/
// 定位 ZIP 文件的中央目录（在末尾，就在全局注释之前）
local ZPOS64_T unz64local_SearchCentralDir OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream));
local ZPOS64_T unz64local_SearchCentralDir(const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream)
{
    # 声明一个指向无符号字符的指针变量
    unsigned char* buf;
    # 声明文件大小变量
    ZPOS64_T uSizeFile;
    # 声明回读变量
    ZPOS64_T uBackRead;
    # 声明全局注释的最大大小变量
    ZPOS64_T uMaxBack=0xffff; /* maximum size of global comment */
    # 声明找到位置变量
    ZPOS64_T uPosFound=0;

    # 如果文件流的末尾不是0，则返回0
    if (ZSEEK64(*pzlib_filefunc_def,filestream,0,ZLIB_FILEFUNC_SEEK_END) != 0)
        return 0;

    # 获取文件流的当前位置
    uSizeFile = ZTELL64(*pzlib_filefunc_def,filestream);

    # 如果最大回读大小大于文件大小，则将最大回读大小设置为文件大小
    if (uMaxBack>uSizeFile)
        uMaxBack = uSizeFile;

    # 分配内存给buf
    buf = (unsigned char*)ALLOC(BUFREADCOMMENT+4);
    # 如果分配内存失败，则返回0
    if (buf==NULL)
        return 0;

    # 设置回读大小为4
    uBackRead = 4;
    # 当回读大小小于最大回读大小时，执行循环
    while (uBackRead<uMaxBack)
    {
        # 声明读取大小变量
        uLong uReadSize;
        # 声明读取位置变量
        ZPOS64_T uReadPos ;
        # 声明循环变量
        int i;
        # 如果回读大小加上缓冲区读取注释大小大于最大回读大小，则将回读大小设置为最大回读大小，否则增加缓冲区读取注释大小
        if (uBackRead+BUFREADCOMMENT>uMaxBack)
            uBackRead = uMaxBack;
        else
            uBackRead+=BUFREADCOMMENT;
        # 计算读取位置
        uReadPos = uSizeFile-uBackRead ;

        # 计算读取大小
        uReadSize = ((BUFREADCOMMENT+4) < (uSizeFile-uReadPos)) ?
                     (BUFREADCOMMENT+4) : (uLong)(uSizeFile-uReadPos);
        # 如果文件流的位置不是uReadPos，则跳出循环
        if (ZSEEK64(*pzlib_filefunc_def,filestream,uReadPos,ZLIB_FILEFUNC_SEEK_SET)!=0)
            break;

        # 如果从文件流中读取的大小不等于uReadSize，则跳出循环
        if (ZREAD64(*pzlib_filefunc_def,filestream,buf,uReadSize)!=uReadSize)
            break;

        # 从后往前遍历buf，找到全局注释的位置
        for (i=(int)uReadSize-3; (i--)>0;)
            if (((*(buf+i))==0x50) && ((*(buf+i+1))==0x4b) &&
                ((*(buf+i+2))==0x05) && ((*(buf+i+3))==0x06))
            {
                uPosFound = uReadPos+(unsigned)i;
                break;
            }

        # 如果找到位置不等于0，则跳出循环
        if (uPosFound!=0)
            break;
    }
    # 释放buf的内存
    TRYFREE(buf);
    # 返回找到的位置
    return uPosFound;
/*
  定位 ZIP 文件的 Central directory 64（在全局注释之前的末尾）
*/
local ZPOS64_T unz64local_SearchCentralDir64 OF((
    const zlib_filefunc64_32_def* pzlib_filefunc_def,
    voidpf filestream));

local ZPOS64_T unz64local_SearchCentralDir64(const zlib_filefunc64_32_def* pzlib_filefunc_def,
                                      voidpf filestream)
{
    unsigned char* buf; // 用于存储读取的数据的缓冲区
    ZPOS64_T uSizeFile; // 文件的大小
    ZPOS64_T uBackRead; // 回溯读取的字节数
    ZPOS64_T uMaxBack=0xffff; /* 全局注释的最大大小 */
    ZPOS64_T uPosFound=0; // 找到的位置
    uLong uL; // 无符号长整型
    ZPOS64_T relativeOffset; // 相对偏移量

    // 将文件指针移动到文件末尾
    if (ZSEEK64(*pzlib_filefunc_def,filestream,0,ZLIB_FILEFUNC_SEEK_END) != 0)
        return 0;

    // 获取文件指针的当前位置，即文件大小
    uSizeFile = ZTELL64(*pzlib_filefunc_def,filestream);

    // 如果全局注释的最大大小大于文件大小，则将其设置为文件大小
    if (uMaxBack>uSizeFile)
        uMaxBack = uSizeFile;

    // 分配缓冲区内存
    buf = (unsigned char*)ALLOC(BUFREADCOMMENT+4);
    if (buf==NULL)
        return 0;

    uBackRead = 4; // 初始回溯读取的字节数为4
    while (uBackRead<uMaxBack)
    {
        uLong uReadSize; // 读取的大小
        ZPOS64_T uReadPos; // 读取的位置
        int i;
        if (uBackRead+BUFREADCOMMENT>uMaxBack)
            uBackRead = uMaxBack;
        else
            uBackRead+=BUFREADCOMMENT;
        uReadPos = uSizeFile-uBackRead ; // 计算读取的位置

        // 计算实际读取的大小
        uReadSize = ((BUFREADCOMMENT+4) < (uSizeFile-uReadPos)) ?
                     (BUFREADCOMMENT+4) : (uLong)(uSizeFile-uReadPos);
        // 将文件指针移动到指定位置
        if (ZSEEK64(*pzlib_filefunc_def,filestream,uReadPos,ZLIB_FILEFUNC_SEEK_SET)!=0)
            break;

        // 读取数据到缓冲区
        if (ZREAD64(*pzlib_filefunc_def,filestream,buf,uReadSize)!=uReadSize)
            break;

        // 在读取的数据中查找 Zip64 end of central directory locator 的标识
        for (i=(int)uReadSize-3; (i--)>0;)
            if (((*(buf+i))==0x50) && ((*(buf+i+1))==0x4b) &&
                ((*(buf+i+2))==0x06) && ((*(buf+i+3))==0x07))
            {
                uPosFound = uReadPos+(unsigned)i; // 找到标识的位置
                break;
            }

        if (uPosFound!=0)
            break;
    }
    TRYFREE(buf); // 释放缓冲区内存
    if (uPosFound == 0)
        return 0;

    /* Zip64 end of central directory locator */
    # 如果文件流的位置不在指定位置，返回 0
    if (ZSEEK64(*pzlib_filefunc_def,filestream, uPosFound,ZLIB_FILEFUNC_SEEK_SET)!=0)
        return 0;

    # 读取并检查签名
    if (unz64local_getLong(pzlib_filefunc_def,filestream,&uL)!=UNZ_OK)
        return 0;

    # 读取并检查 zip64 结尾中央目录的磁盘号
    if (unz64local_getLong(pzlib_filefunc_def,filestream,&uL)!=UNZ_OK)
        return 0;
    if (uL != 0)
        return 0;

    # 读取 zip64 结尾中央目录记录的相对偏移量
    if (unz64local_getLong64(pzlib_filefunc_def,filestream,&relativeOffset)!=UNZ_OK)
        return 0;

    # 读取并检查总磁盘数
    if (unz64local_getLong(pzlib_filefunc_def,filestream,&uL)!=UNZ_OK)
        return 0;
    if (uL != 1)
        return 0;

    # 跳转到中央目录记录的末尾
    if (ZSEEK64(*pzlib_filefunc_def,filestream, relativeOffset,ZLIB_FILEFUNC_SEEK_SET)!=0)
        return 0;

    # 读取并检查签名
    if (unz64local_getLong(pzlib_filefunc_def,filestream,&uL)!=UNZ_OK)
        return 0;

    # 检查签名是否为 0x06064b50
    if (uL != 0x06064b50)
        return 0;

    # 返回 zip64 结尾中央目录记录的相对偏移量
    return relativeOffset;
# 打开一个 Zip 文件。path 包含完整的路径名（例如，在 Windows NT 计算机上为 "c:\\test\\zlib114.zip"，在 Unix 计算机上为 "zlib/zlib114.zip"）。
# 如果无法打开 zip 文件（文件不存在或无效），返回值为 NULL。
# 否则，返回值是一个 unzFile 句柄，可用于此解压缩包的其他函数。
local unzFile unzOpenInternal (const void *path,
                               zlib_filefunc64_32_def* pzlib_filefunc64_32_def,
                               int is64bitOpenFunction)
{
    unz64_s us;
    unz64_s *s;
    ZPOS64_T central_pos;
    uLong   uL;

    uLong number_disk;          /* 当前 dist 的编号，用于跨 ZIP，不支持，始终为 0 */
    uLong number_disk_with_CD;  /* 带有中央目录的磁盘编号，用于跨 ZIP，不支持，始终为 0 */
    ZPOS64_T number_entry_CD;      /* 中央目录中的条目总数（与 nospan 上的 number_entry 相同） */

    int err=UNZ_OK;

    # 如果版权信息的第一个字符不是空格，则返回 NULL
    if (unz_copyright[0]!=' ')
        return NULL;

    # 初始化 us 的文件操作函数指针
    us.z_filefunc.zseek32_file = NULL;
    us.z_filefunc.ztell32_file = NULL;
    # 如果 pzlib_filefunc64_32_def 为 NULL，则填充 us 的文件操作函数指针
    if (pzlib_filefunc64_32_def==NULL)
        fill_fopen64_filefunc(&us.z_filefunc.zfile_func64);
    else
        us.z_filefunc = *pzlib_filefunc64_32_def;
    # 设置 is64bitOpenFunction
    us.is64bitOpenFunction = is64bitOpenFunction;

    # 使用文件操作函数打开文件流
    us.filestream = ZOPEN64(us.z_filefunc,
                                                 path,
                                                 ZLIB_FILEFUNC_MODE_READ |
                                                 ZLIB_FILEFUNC_MODE_EXISTING);
    # 如果文件流为空，则返回 NULL
    if (us.filestream==NULL)
        return NULL;

    # 搜索中央目录的位置
    central_pos = unz64local_SearchCentralDir64(&us.z_filefunc,us.filestream);
    # 如果 central_pos 存在，则
    if (central_pos)
    }
    else
    {
        // 查找中央目录的位置
        central_pos = unz64local_SearchCentralDir(&us.z_filefunc,us.filestream);
        // 如果位置为0，则表示错误
        if (central_pos==0)
            err=UNZ_ERRNO;

        // 设置为非 Zip64 格式
        us.isZip64 = 0;

        // 定位到中央目录的位置
        if (ZSEEK64(us.z_filefunc, us.filestream,
                                        central_pos,ZLIB_FILEFUNC_SEEK_SET)!=0)
            err=UNZ_ERRNO;

        /* the signature, already checked */
        // 读取签名，已经检查过
        if (unz64local_getLong(&us.z_filefunc, us.filestream,&uL)!=UNZ_OK)
            err=UNZ_ERRNO;

        /* number of this disk */
        // 读取当前磁盘编号
        if (unz64local_getShort(&us.z_filefunc, us.filestream,&number_disk)!=UNZ_OK)
            err=UNZ_ERRNO;

        /* number of the disk with the start of the central directory */
        // 读取包含中央目录起始位置的磁盘编号
        if (unz64local_getShort(&us.z_filefunc, us.filestream,&number_disk_with_CD)!=UNZ_OK)
            err=UNZ_ERRNO;

        /* total number of entries in the central dir on this disk */
        // 读取当前磁盘上中央目录中的条目总数
        if (unz64local_getShort(&us.z_filefunc, us.filestream,&uL)!=UNZ_OK)
            err=UNZ_ERRNO;
        us.gi.number_entry = uL;

        /* total number of entries in the central dir */
        // 读取中央目录中的总条目数
        if (unz64local_getShort(&us.z_filefunc, us.filestream,&uL)!=UNZ_OK)
            err=UNZ_ERRNO;
        number_entry_CD = uL;

        // 检查中央目录中的条目数是否与当前磁盘上的条目数一致，以及磁盘编号是否为0
        if ((number_entry_CD!=us.gi.number_entry) ||
            (number_disk_with_CD!=0) ||
            (number_disk!=0))
            err=UNZ_BADZIPFILE;

        /* size of the central directory */
        // 读取中央目录的大小
        if (unz64local_getLong(&us.z_filefunc, us.filestream,&uL)!=UNZ_OK)
            err=UNZ_ERRNO;
        us.size_central_dir = uL;

        /* offset of start of central directory with respect to the
            starting disk number */
        // 读取中央目录相对于起始磁盘编号的偏移量
        if (unz64local_getLong(&us.z_filefunc, us.filestream,&uL)!=UNZ_OK)
            err=UNZ_ERRNO;
        us.offset_central_dir = uL;

        /* zipfile comment length */
        // 读取 ZIP 文件注释的长度
        if (unz64local_getShort(&us.z_filefunc, us.filestream,&us.gi.size_comment)!=UNZ_OK)
            err=UNZ_ERRNO;
    }
    # 如果中央目录的位置小于中央目录的偏移量加上中央目录的大小，并且没有错误发生，则将错误标记为UNZ_BADZIPFILE
    if ((central_pos<us.offset_central_dir+us.size_central_dir) &&
        (err==UNZ_OK))
        err=UNZ_BADZIPFILE;

    # 如果发生了错误，则关闭文件流并返回空值
    if (err!=UNZ_OK)
    {
        ZCLOSE64(us.z_filefunc, us.filestream);
        return NULL;
    }

    # 计算ZIP文件之前的字节数
    us.byte_before_the_zipfile = central_pos -
                            (us.offset_central_dir+us.size_central_dir);
    # 设置中央目录的位置
    us.central_pos = central_pos;
    # 设置ZIP文件中当前文件的指针为空
    us.pfile_in_zip_read = NULL;
    # 将加密标志位设置为0
    us.encrypted = 0;

    # 分配内存以存储unz64_s结构体
    s=(unz64_s*)ALLOC(sizeof(unz64_s));
    # 如果分配成功
    if( s != NULL)
    {
        # 将us结构体的内容复制到s指向的内存中
        *s=us;
        # 跳转到ZIP文件中的第一个文件
        unzGoToFirstFile((unzFile)s);
    }
    # 返回unzFile类型的指针
    return (unzFile)s;
# 打开一个 ZIP 文件，使用给定的文件操作函数
extern unzFile ZEXPORT unzOpen2 (const char *path,
                                        zlib_filefunc_def* pzlib_filefunc32_def)
{
    # 如果文件操作函数不为空
    if (pzlib_filefunc32_def != NULL)
    {
        # 创建一个 zlib_filefunc64_32_def 结构体，并从文件操作函数32位版本中填充
        zlib_filefunc64_32_def zlib_filefunc64_32_def_fill;
        fill_zlib_filefunc64_32_def_from_filefunc32(&zlib_filefunc64_32_def_fill,pzlib_filefunc32_def);
        # 调用内部函数打开 ZIP 文件
        return unzOpenInternal(path, &zlib_filefunc64_32_def_fill, 0);
    }
    else
        # 否则，直接调用内部函数打开 ZIP 文件
        return unzOpenInternal(path, NULL, 0);
}

# 打开一个 ZIP 文件，使用给定的64位文件操作函数
extern unzFile ZEXPORT unzOpen2_64 (const void *path,
                                     zlib_filefunc64_def* pzlib_filefunc_def)
{
    # 如果文件操作函数不为空
    if (pzlib_filefunc_def != NULL)
    {
        # 创建一个 zlib_filefunc64_32_def 结构体，并填充64位文件操作函数
        zlib_filefunc64_32_def zlib_filefunc64_32_def_fill;
        zlib_filefunc64_32_def_fill.zfile_func64 = *pzlib_filefunc_def;
        zlib_filefunc64_32_def_fill.ztell32_file = NULL;
        zlib_filefunc64_32_def_fill.zseek32_file = NULL;
        # 调用内部函数打开 ZIP 文件
        return unzOpenInternal(path, &zlib_filefunc64_32_def_fill, 1);
    }
    else
        # 否则，直接调用内部函数打开 ZIP 文件
        return unzOpenInternal(path, NULL, 1);
}

# 打开一个 ZIP 文件
extern unzFile ZEXPORT unzOpen (const char *path)
{
    # 调用内部函数打开 ZIP 文件
    return unzOpenInternal(path, NULL, 0);
}

# 打开一个64位 ZIP 文件
extern unzFile ZEXPORT unzOpen64 (const void *path)
{
    # 调用内部函数打开 ZIP 文件
    return unzOpenInternal(path, NULL, 1);
}

# 关闭一个已经打开的 ZIP 文件
extern int ZEXPORT unzClose (unzFile file)
{
    unz64_s* s;
    # 如果文件为空，返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;

    # 如果 ZIP 文件中有文件被打开，先关闭当前文件
    if (s->pfile_in_zip_read!=NULL)
        unzCloseCurrentFile(file);

    # 关闭文件流，释放内存
    ZCLOSE64(s->z_filefunc, s->filestream);
    TRYFREE(s);
    return UNZ_OK;
}

# 将 ZIP 文件的信息写入 *pglobal_info 结构体中
# 不需要准备结构体
# 如果没有问题，返回 UNZ_OK
# 获取 ZIP 文件的全局信息，存储在 unz_global_info64 结构体中
extern int ZEXPORT unzGetGlobalInfo64 (unzFile file, unz_global_info64* pglobal_info)
{
    # 定义指向 unz64_s 结构体的指针 s
    unz64_s* s;
    # 如果文件为空，则返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;
    # 将文件指针转换为 unz64_s 结构体指针
    s=(unz64_s*)file;
    # 将全局信息拷贝到传入的指针中
    *pglobal_info=s->gi;
    # 返回操作成功
    return UNZ_OK;
}

# 获取 ZIP 文件的全局信息，存储在 unz_global_info 结构体中
extern int ZEXPORT unzGetGlobalInfo (unzFile file, unz_global_info* pglobal_info32)
{
    # 定义指向 unz64_s 结构体的指针 s
    unz64_s* s;
    # 如果文件为空，则返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;
    # 将文件指针转换为 unz64_s 结构体指针
    s=(unz64_s*)file;
    # 将文件数量和注释大小拷贝到传入的结构体中
    pglobal_info32->number_entry = (uLong)s->gi.number_entry;
    pglobal_info32->size_comment = s->gi.size_comment;
    # 返回操作成功
    return UNZ_OK;
}

# 将 DOS 格式的日期/时间转换为 tm_unz 结构体
local void unz64local_DosDateToTmuDate (ZPOS64_T ulDosDate, tm_unz* ptm)
{
    # 定义变量 uDate 用于存储转换后的日期
    ZPOS64_T uDate;
    # 取 ulDosDate 的高 32 位作为日期
    uDate = (ZPOS64_T)(ulDosDate>>16);
    # 将日期中的日、月、年、时、分、秒分别转换为 tm_unz 结构体中的成员
    ptm->tm_mday = (int)(uDate&0x1f) ;
    ptm->tm_mon =  (int)((((uDate)&0x1E0)/0x20)-1) ;
    ptm->tm_year = (int)(((uDate&0x0FE00)/0x0200)+1980) ;
    ptm->tm_hour = (int) ((ulDosDate &0xF800)/0x800);
    ptm->tm_min =  (int) ((ulDosDate&0x7E0)/0x20) ;
    ptm->tm_sec =  (int) (2*(ulDosDate&0x1f)) ;
}

# 获取 ZIP 文件中当前文件的内部信息
local int unz64local_GetCurrentFileInfoInternal OF((unzFile file,
                                                  unz_file_info64 *pfile_info,
                                                  unz_file_info64_internal
                                                  *pfile_info_internal,
                                                  char *szFileName,
                                                  uLong fileNameBufferSize,
                                                  void *extraField,
                                                  uLong extraFieldBufferSize,
                                                  char *szComment,
                                                  uLong commentBufferSize));
# 定义函数，获取当前文件信息的内部实现
local int unz64local_GetCurrentFileInfoInternal (unzFile file,
                                                  unz_file_info64 *pfile_info,
                                                  unz_file_info64_internal
                                                  *pfile_info_internal,
                                                  char *szFileName,
                                                  uLong fileNameBufferSize,
                                                  void *extraField,
                                                  uLong extraFieldBufferSize,
                                                  char *szComment,
                                                  uLong commentBufferSize)
{
    # 定义变量
    unz64_s* s;
    unz_file_info64 file_info;
    unz_file_info64_internal file_info_internal;
    int err=UNZ_OK;
    uLong uMagic;
    long lSeek=0;
    uLong uL;

    # 检查文件是否为空
    if (file==NULL)
        return UNZ_PARAMERROR;
    # 将文件转换为unz64_s类型
    s=(unz64_s*)file;
    # 定位到文件流的指定位置
    if (ZSEEK64(s->z_filefunc, s->filestream,
              s->pos_in_central_dir+s->byte_before_the_zipfile,
              ZLIB_FILEFUNC_SEEK_SET)!=0)
        err=UNZ_ERRNO;

    # 检查文件的魔数
    if (err==UNZ_OK)
    {
        if (unz64local_getLong(&s->z_filefunc, s->filestream,&uMagic) != UNZ_OK)
            err=UNZ_ERRNO;
        else if (uMagic!=0x02014b50)
            err=UNZ_BADZIPFILE;
    }

    # 获取文件信息的版本号
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.version) != UNZ_OK)
        err=UNZ_ERRNO;

    # 获取文件信息所需的版本号
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.version_needed) != UNZ_OK)
        err=UNZ_ERRNO;

    # 获取文件信息的标志
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.flag) != UNZ_OK)
        err=UNZ_ERRNO;

    # 获取文件信息的压缩方法
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.compression_method) != UNZ_OK)
        err=UNZ_ERRNO;

    # 获取文件信息的DOS日期
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&file_info.dosDate) != UNZ_OK)
        err=UNZ_ERRNO;
    # 将 DOS 日期转换为 TMU 日期
    unz64local_DosDateToTmuDate(file_info.dosDate,&file_info.tmu_date);

    # 读取文件的 CRC 校验值
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&file_info.crc) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件的压缩大小
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uL) != UNZ_OK)
        err=UNZ_ERRNO;
    file_info.compressed_size = uL;

    # 读取文件的解压缩大小
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uL) != UNZ_OK)
        err=UNZ_ERRNO;
    file_info.uncompressed_size = uL;

    # 读取文件名的长度
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.size_filename) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件额外信息的长度
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.size_file_extra) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件注释的长度
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.size_file_comment) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件所在磁盘号
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.disk_num_start) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件的内部文件属性
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&file_info.internal_fa) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取文件的外部文件属性
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&file_info.external_fa) != UNZ_OK)
        err=UNZ_ERRNO;

    # 读取本地头的相对偏移量
    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uL) != UNZ_OK)
        err=UNZ_ERRNO;
    file_info_internal.offset_curfile = uL;

    # 更新文件名的偏移量
    lSeek+=file_info.size_filename;
    if ((err==UNZ_OK) && (szFileName!=NULL))
    {
        uLong uSizeRead ;
        if (file_info.size_filename<fileNameBufferSize)
        {
            *(szFileName+file_info.size_filename)='\0';
            uSizeRead = file_info.size_filename;
        }
        else
            uSizeRead = fileNameBufferSize;

        if ((file_info.size_filename>0) && (fileNameBufferSize>0))
            # 读取文件名
            if (ZREAD64(s->z_filefunc, s->filestream,szFileName,uSizeRead)!=uSizeRead)
                err=UNZ_ERRNO;
        lSeek -= uSizeRead;
    }

    # 读取额外字段
    if ((err==UNZ_OK) && (extraField!=NULL))
    {
        // 定义变量 uSizeRead，用于存储读取的大小
        ZPOS64_T uSizeRead ;
        // 如果文件额外信息的大小小于额外字段缓冲区的大小，则将 uSizeRead 设置为文件额外信息的大小，否则设置为额外字段缓冲区的大小
        if (file_info.size_file_extra<extraFieldBufferSize)
            uSizeRead = file_info.size_file_extra;
        else
            uSizeRead = extraFieldBufferSize;
    
        // 如果 lSeek 不等于 0
        if (lSeek!=0)
        {
            // 如果通过 ZSEEK64 函数定位成功，则将 lSeek 设置为 0，否则将 err 设置为 UNZ_ERRNO
            if (ZSEEK64(s->z_filefunc, s->filestream,(ZPOS64_T)lSeek,ZLIB_FILEFUNC_SEEK_CUR)==0)
                lSeek=0;
            else
                err=UNZ_ERRNO;
        }
    
        // 如果文件额外信息的大小大于 0 且额外字段缓冲区的大小大于 0
        if ((file_info.size_file_extra>0) && (extraFieldBufferSize>0))
            // 如果通过 ZREAD64 函数读取的大小不等于 uSizeRead，则将 err 设置为 UNZ_ERRNO
            if (ZREAD64(s->z_filefunc, s->filestream,extraField,(uLong)uSizeRead)!=uSizeRead)
                err=UNZ_ERRNO;
    
        // 更新 lSeek 的值
        lSeek += file_info.size_file_extra - (uLong)uSizeRead;
    }
    else
        // 更新 lSeek 的值
        lSeek += file_info.size_file_extra;
    
    // 如果 err 等于 UNZ_OK 且文件注释的大小不等于 0
    if ((err==UNZ_OK) && (file_info.size_file_extra != 0))
    {
        // ...
    }
    
    // 如果 err 等于 UNZ_OK 且 szComment 不为 NULL
    if ((err==UNZ_OK) && (szComment!=NULL))
    {
        // 定义变量 uSizeRead
        uLong uSizeRead ;
        // 如果文件注释的大小小于注释缓冲区的大小
        if (file_info.size_file_comment<commentBufferSize)
        {
            // 在 szComment 中文件注释的大小位置添加结束符'\0'，并将 uSizeRead 设置为文件注释的大小
            *(szComment+file_info.size_file_comment)='\0';
            uSizeRead = file_info.size_file_comment;
        }
        else
            // 将 uSizeRead 设置为注释缓冲区的大小
            uSizeRead = commentBufferSize;
    
        // 如果 lSeek 不等于 0
        if (lSeek!=0)
        {
            // 如果通过 ZSEEK64 函数定位成功，则将 lSeek 设置为 0，否则将 err 设置为 UNZ_ERRNO
            if (ZSEEK64(s->z_filefunc, s->filestream,(ZPOS64_T)lSeek,ZLIB_FILEFUNC_SEEK_CUR)==0)
                lSeek=0;
            else
                err=UNZ_ERRNO;
        }
    
        // 如果文件注释的大小大于 0 且注释缓冲区的大小大于 0
        if ((file_info.size_file_comment>0) && (commentBufferSize>0))
            // 如果通过 ZREAD64 函数读取的大小不等于 uSizeRead，则将 err 设置为 UNZ_ERRNO
            if (ZREAD64(s->z_filefunc, s->filestream,szComment,uSizeRead)!=uSizeRead)
                err=UNZ_ERRNO;
        // 更新 lSeek 的值
        lSeek+=file_info.size_file_comment - uSizeRead;
    }
    else
        // 更新 lSeek 的值
        lSeek+=file_info.size_file_comment;
    
    // 如果 err 等于 UNZ_OK 且 pfile_info 不为 NULL，则将 pfile_info 指向 file_info
    if ((err==UNZ_OK) && (pfile_info!=NULL))
        *pfile_info=file_info;
    
    // 如果 err 等于 UNZ_OK 且 pfile_info_internal 不为 NULL，则将 pfile_info_internal 指向 file_info_internal
    if ((err==UNZ_OK) && (pfile_info_internal!=NULL))
        *pfile_info_internal=file_info_internal;
    
    // 返回 err
    return err;
    }
/*
  将 ZipFile 的信息写入 *pglobal_info 结构中。
  不需要对结构进行任何准备。
  如果没有问题，返回 UNZ_OK。
*/
extern int ZEXPORT unzGetCurrentFileInfo64 (unzFile file,
                                          unz_file_info64 * pfile_info,
                                          char * szFileName, uLong fileNameBufferSize,
                                          void *extraField, uLong extraFieldBufferSize,
                                          char* szComment,  uLong commentBufferSize)
{
    return unz64local_GetCurrentFileInfoInternal(file,pfile_info,NULL,
                                                szFileName,fileNameBufferSize,
                                                extraField,extraFieldBufferSize,
                                                szComment,commentBufferSize);
}

extern int ZEXPORT unzGetCurrentFileInfo (unzFile file,
                                          unz_file_info * pfile_info,
                                          char * szFileName, uLong fileNameBufferSize,
                                          void *extraField, uLong extraFieldBufferSize,
                                          char* szComment,  uLong commentBufferSize)
{
    int err;
    unz_file_info64 file_info64;
    err = unz64local_GetCurrentFileInfoInternal(file,&file_info64,NULL,
                                                szFileName,fileNameBufferSize,
                                                extraField,extraFieldBufferSize,
                                                szComment,commentBufferSize);
    if ((err==UNZ_OK) && (pfile_info != NULL))
*/
    {
        # 将64位文件信息结构体中的版本赋值给pfile_info中的版本
        pfile_info->version = file_info64.version;
        # 将64位文件信息结构体中的所需版本赋值给pfile_info中的所需版本
        pfile_info->version_needed = file_info64.version_needed;
        # 将64位文件信息结构体中的标志赋值给pfile_info中的标志
        pfile_info->flag = file_info64.flag;
        # 将64位文件信息结构体中的压缩方法赋值给pfile_info中的压缩方法
        pfile_info->compression_method = file_info64.compression_method;
        # 将64位文件信息结构体中的DOS日期赋值给pfile_info中的DOS日期
        pfile_info->dosDate = file_info64.dosDate;
        # 将64位文件信息结构体中的CRC赋值给pfile_info中的CRC
        pfile_info->crc = file_info64.crc;

        # 将64位文件信息结构体中的文件名大小赋值给pfile_info中的文件名大小
        pfile_info->size_filename = file_info64.size_filename;
        # 将64位文件信息结构体中的额外文件大小赋值给pfile_info中的额外文件大小
        pfile_info->size_file_extra = file_info64.size_file_extra;
        # 将64位文件信息结构体中的文件注释大小赋值给pfile_info中的文件注释大小
        pfile_info->size_file_comment = file_info64.size_file_comment;

        # 将64位文件信息结构体中的起始磁盘号赋值给pfile_info中的起始磁盘号
        pfile_info->disk_num_start = file_info64.disk_num_start;
        # 将64位文件信息结构体中的内部文件属性赋值给pfile_info中的内部文件属性
        pfile_info->internal_fa = file_info64.internal_fa;
        # 将64位文件信息结构体中的外部文件属性赋值给pfile_info中的外部文件属性
        pfile_info->external_fa = file_info64.external_fa;

        # 将64位文件信息结构体中的TMU日期赋值给pfile_info中的TMU日期
        pfile_info->tmu_date = file_info64.tmu_date,

        # 将64位文件信息结构体中的压缩大小赋值给pfile_info中的压缩大小
        pfile_info->compressed_size = (uLong)file_info64.compressed_size;
        # 将64位文件信息结构体中的未压缩大小赋值给pfile_info中的未压缩大小
        pfile_info->uncompressed_size = (uLong)file_info64.uncompressed_size;

    }
    # 返回错误码
    return err;
# 设置 zipfile 的当前文件为第一个文件
# 如果没有问题，返回 UNZ_OK
extern int ZEXPORT unzGoToFirstFile (unzFile file)
{
    int err=UNZ_OK;  # 初始化错误码为 UNZ_OK
    unz64_s* s;  # 定义 unz64_s 结构体指针 s

    if (file==NULL)  # 如果文件为空，返回参数错误
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;  # 将 file 转换为 unz64_s 结构体指针
    s->pos_in_central_dir=s->offset_central_dir;  # 设置中央目录的位置为偏移中央目录的位置
    s->num_file=0;  # 初始化文件数量为 0
    # 获取当前文件的信息，存储在 s->cur_file_info 和 s->cur_file_info_internal 中
    err=unz64local_GetCurrentFileInfoInternal(file,&s->cur_file_info,
                                             &s->cur_file_info_internal,
                                             NULL,0,NULL,0,NULL,0);
    s->current_file_ok = (err == UNZ_OK);  # 如果 err 等于 UNZ_OK，则当前文件有效
    return err;  # 返回错误码
}

# 设置 zipfile 的当前文件为下一个文件
# 如果没有问题，返回 UNZ_OK
# 如果当前文件是最后一个文件，返回 UNZ_END_OF_LIST_OF_FILE
extern int ZEXPORT unzGoToNextFile (unzFile  file)
{
    unz64_s* s;  # 定义 unz64_s 结构体指针 s
    int err;  # 定义错误码变量 err

    if (file==NULL)  # 如果文件为空，返回参数错误
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;  # 将 file 转换为 unz64_s 结构体指针
    if (!s->current_file_ok)  # 如果当前文件无效，返回文件列表结束
        return UNZ_END_OF_LIST_OF_FILE;
    if (s->gi.number_entry != 0xffff)    /* 2^16 files overflow hack */  # 如果文件数量不等于 2^16，即不是 65535
      if (s->num_file+1==s->gi.number_entry)  # 如果下一个文件是最后一个文件，返回文件列表结束
        return UNZ_END_OF_LIST_OF_FILE;

    # 更新中央目录位置，移动到下一个文件
    s->pos_in_central_dir += SIZECENTRALDIRITEM + s->cur_file_info.size_filename +
            s->cur_file_info.size_file_extra + s->cur_file_info.size_file_comment ;
    s->num_file++;  # 文件数量加一
    # 获取当前文件的信息，存储在 s->cur_file_info 和 s->cur_file_info_internal 中
    err = unz64local_GetCurrentFileInfoInternal(file,&s->cur_file_info,
                                               &s->cur_file_info_internal,
                                               NULL,0,NULL,0,NULL,0);
    s->current_file_ok = (err == UNZ_OK);  # 如果 err 等于 UNZ_OK，则当前文件有效
    return err;  # 返回错误码
}

# 尝试在 zipfile 中定位文件 szFileName
# 根据 iCaseSensitivity 的值确定大小写敏感性
# 返回值：
# 如果找到文件，将其设置为当前文件，返回 UNZ_OK
# 如果未找到文件，返回 UNZ_END_OF_LIST_OF_FILE
extern int ZEXPORT unzLocateFile (unzFile file, const char *szFileName, int iCaseSensitivity)
{
    unz64_s* s;  # 定义 unz64_s 结构体指针 s
    # 定义一个整型变量 err，用于存储函数执行过程中的错误码

    # 保存当前文件信息的变量
    unz_file_info64 cur_file_infoSaved;
    unz_file_info64_internal cur_file_info_internalSaved;
    ZPOS64_T num_fileSaved;
    ZPOS64_T pos_in_central_dirSaved;

    # 如果文件指针为空，返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;

    # 如果文件名长度超过最大长度限制，返回参数错误
    if (strlen(szFileName)>=UNZ_MAXFILENAMEINZIP)
        return UNZ_PARAMERROR;

    # 将文件指针转换为 unz64_s 类型
    s=(unz64_s*)file;

    # 如果当前文件不可用，返回文件列表结束错误
    if (!s->current_file_ok)
        return UNZ_END_OF_LIST_OF_FILE;

    # 保存当前状态
    num_fileSaved = s->num_file;
    pos_in_central_dirSaved = s->pos_in_central_dir;
    cur_file_infoSaved = s->cur_file_info;
    cur_file_info_internalSaved = s->cur_file_info_internal;

    # 跳转到第一个文件
    err = unzGoToFirstFile(file);

    # 循环遍历文件列表
    while (err == UNZ_OK)
    {
        char szCurrentFileName[UNZ_MAXFILENAMEINZIP+1];
        # 获取当前文件信息
        err = unzGetCurrentFileInfo64(file,NULL,
                                    szCurrentFileName,sizeof(szCurrentFileName)-1,
                                    NULL,0,NULL,0);
        if (err == UNZ_OK)
        {
            # 比较当前文件名和给定文件名，如果相同则返回成功
            if (unzStringFileNameCompare(szCurrentFileName,
                                            szFileName,iCaseSensitivity)==0)
                return UNZ_OK;
            # 跳转到下一个文件
            err = unzGoToNextFile(file);
        }
    }

    # 如果遍历失败，恢复当前文件的状态
    s->num_file = num_fileSaved ;
    s->pos_in_central_dir = pos_in_central_dirSaved ;
    s->cur_file_info = cur_file_infoSaved;
    s->cur_file_info_internal = cur_file_info_internalSaved;
    return err;
}

/*
///////////////////////////////////////////
// Contributed by Ryan Haksi (mailto://cryogen@infoserve.net)
// I need random access
//
// Further optimization could be realized by adding an ability
// to cache the directory in memory. The goal being a single
// comprehensive file read to put the file I need in a memory.
*/

/*
typedef struct unz_file_pos_s
{
    ZPOS64_T pos_in_zip_directory;   // offset in file
    ZPOS64_T num_of_file;            // # of file
} unz_file_pos;
*/

// 定义函数 unzGetFilePos64，获取文件位置信息
extern int ZEXPORT unzGetFilePos64(unzFile file, unz64_file_pos*  file_pos)
{
    unz64_s* s;

    // 如果文件或文件位置为空，则返回参数错误
    if (file==NULL || file_pos==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;
    // 如果当前文件不可用，则返回文件列表结束
    if (!s->current_file_ok)
        return UNZ_END_OF_LIST_OF_FILE;

    // 设置文件位置信息
    file_pos->pos_in_zip_directory  = s->pos_in_central_dir;
    file_pos->num_of_file           = s->num_file;

    return UNZ_OK;
}

// 定义函数 unzGetFilePos，获取文件位置信息
extern int ZEXPORT unzGetFilePos(
    unzFile file,
    unz_file_pos* file_pos)
{
    unz64_file_pos file_pos64;
    int err = unzGetFilePos64(file,&file_pos64);
    if (err==UNZ_OK)
    {
        file_pos->pos_in_zip_directory = (uLong)file_pos64.pos_in_zip_directory;
        file_pos->num_of_file = (uLong)file_pos64.num_of_file;
    }
    return err;
}

// 定义函数 unzGoToFilePos64，跳转到指定文件位置
extern int ZEXPORT unzGoToFilePos64(unzFile file, const unz64_file_pos* file_pos)
{
    unz64_s* s;
    int err;

    // 如果文件或文件位置为空，则返回参数错误
    if (file==NULL || file_pos==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;

    /* jump to the right spot */
    // 跳转到指定位置
    s->pos_in_central_dir = file_pos->pos_in_zip_directory;
    s->num_file           = file_pos->num_of_file;

    /* set the current file */
    // 设置当前文件信息
    err = unz64local_GetCurrentFileInfoInternal(file,&s->cur_file_info,
                                               &s->cur_file_info_internal,
                                               NULL,0,NULL,0,NULL,0);
    /* return results */
    // 返回结果
    s->current_file_ok = (err == UNZ_OK);
    return err;
}

// 定义函数 unzGoToFilePos，跳转到指定文件位置
extern int ZEXPORT unzGoToFilePos(
    unzFile file,
    unz_file_pos* file_pos)
    # 定义一个指向unz_file_pos结构的指针变量file_pos
{
    // 定义一个名为file_pos64的unz64_file_pos结构体变量
    unz64_file_pos file_pos64;
    // 如果file_pos为空指针，则返回UNZ_PARAMERROR错误
    if (file_pos == NULL)
        return UNZ_PARAMERROR;

    // 将file_pos中的pos_in_zip_directory和num_of_file赋值给file_pos64
    file_pos64.pos_in_zip_directory = file_pos->pos_in_zip_directory;
    file_pos64.num_of_file = file_pos->num_of_file;
    // 调用unzGoToFilePos64函数，传入file和file_pos64作为参数
    return unzGoToFilePos64(file,&file_pos64);
}

/*
// Unzip Helper Functions - should be here?
///////////////////////////////////////////
*/

/*
  读取当前zip文件的本地头部
  检查本地头部的一致性以及中央目录末尾关于该文件的信息
  将* piSizeVar中的额外信息大小存储在本地头部中（文件名和额外字段数据大小）
*/
local int unz64local_CheckCurrentFileCoherencyHeader (unz64_s* s, uInt* piSizeVar,
                                                    ZPOS64_T * poffset_local_extrafield,
                                                    uInt  * psize_local_extrafield)
{
    uLong uMagic,uData,uFlags;
    uLong size_filename;
    uLong size_extra_field;
    int err=UNZ_OK;

    *piSizeVar = 0;
    *poffset_local_extrafield = 0;
    *psize_local_extrafield = 0;

    // 如果将文件指针移动到当前文件信息的偏移量加上字节偏移量之前的位置失败，则返回UNZ_ERRNO错误
    if (ZSEEK64(s->z_filefunc, s->filestream,s->cur_file_info_internal.offset_curfile +
                                s->byte_before_the_zipfile,ZLIB_FILEFUNC_SEEK_SET)!=0)
        return UNZ_ERRNO;


    if (err==UNZ_OK)
    {
        // 从文件中读取一个长整型数值到uMagic中
        if (unz64local_getLong(&s->z_filefunc, s->filestream,&uMagic) != UNZ_OK)
            err=UNZ_ERRNO;
        // 如果uMagic不等于0x04034b50，则返回UNZ_BADZIPFILE错误
        else if (uMagic!=0x04034b50)
            err=UNZ_BADZIPFILE;
    }

    // 从文件中读取一个短整型数值到uData中
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&uData) != UNZ_OK)
        err=UNZ_ERRNO;
/*
    else if ((err==UNZ_OK) && (uData!=s->cur_file_info.wVersion))
        err=UNZ_BADZIPFILE;
*/
    // 从文件中读取一个短整型数值到uFlags中
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&uFlags) != UNZ_OK)
        err=UNZ_ERRNO;

    // 从文件中读取一个短整型数值到uData中
    if (unz64local_getShort(&s->z_filefunc, s->filestream,&uData) != UNZ_OK)
        err=UNZ_ERRNO;
    // 如果uData不等于s->cur_file_info.compression_method，则返回UNZ_BADZIPFILE错误
    else if ((err==UNZ_OK) && (uData!=s->cur_file_info.compression_method))
        err=UNZ_BADZIPFILE;
    # 如果没有错误，并且当前文件信息的压缩方法不为0
    if ((err==UNZ_OK) && (s->cur_file_info.compression_method!=0) &&
/* #ifdef HAVE_BZIP2 */
                         (s->cur_file_info.compression_method!=Z_BZIP2ED) &&
/* #endif */
                         (s->cur_file_info.compression_method!=Z_DEFLATED))
        err=UNZ_BADZIPFILE;  // 如果压缩方法不是 Z_BZIP2ED 或 Z_DEFLATED，则将错误码设置为 UNZ_BADZIPFILE

    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uData) != UNZ_OK) /* date/time */
        err=UNZ_ERRNO;  // 如果获取日期/时间出错，则将错误码设置为 UNZ_ERRNO

    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uData) != UNZ_OK) /* crc */
        err=UNZ_ERRNO;  // 如果获取 CRC 出错，则将错误码设置为 UNZ_ERRNO
    else if ((err==UNZ_OK) && (uData!=s->cur_file_info.crc) && ((uFlags & 8)==0))
        err=UNZ_BADZIPFILE;  // 如果错误码为 UNZ_OK 且 CRC 不匹配且 uFlags 的第四位为 0，则将错误码设置为 UNZ_BADZIPFILE

    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uData) != UNZ_OK) /* size compr */
        err=UNZ_ERRNO;  // 如果获取压缩大小出错，则将错误码设置为 UNZ_ERRNO
    else if (uData != 0xFFFFFFFF && (err==UNZ_OK) && (uData!=s->cur_file_info.compressed_size) && ((uFlags & 8)==0))
        err=UNZ_BADZIPFILE;  // 如果压缩大小不是 0xFFFFFFFF 且错误码为 UNZ_OK 且压缩大小不匹配且 uFlags 的第四位为 0，则将错误码设置为 UNZ_BADZIPFILE

    if (unz64local_getLong(&s->z_filefunc, s->filestream,&uData) != UNZ_OK) /* size uncompr */
        err=UNZ_ERRNO;  // 如果获取解压缩大小出错，则将错误码设置为 UNZ_ERRNO
    else if (uData != 0xFFFFFFFF && (err==UNZ_OK) && (uData!=s->cur_file_info.uncompressed_size) && ((uFlags & 8)==0))
        err=UNZ_BADZIPFILE;  // 如果解压缩大小不是 0xFFFFFFFF 且错误码为 UNZ_OK 且解压缩大小不匹配且 uFlags 的第四位为 0，则将错误码设置为 UNZ_BADZIPFILE

    if (unz64local_getShort(&s->z_filefunc, s->filestream,&size_filename) != UNZ_OK)
        err=UNZ_ERRNO;  // 如果获取文件名长度出错，则将错误码设置为 UNZ_ERRNO
    else if ((err==UNZ_OK) && (size_filename!=s->cur_file_info.size_filename))
        err=UNZ_BADZIPFILE;  // 如果错误码为 UNZ_OK 且文件名长度不匹配，则将错误码设置为 UNZ_BADZIPFILE

    *piSizeVar += (uInt)size_filename;  // 将文件名长度加到 piSizeVar 上

    if (unz64local_getShort(&s->z_filefunc, s->filestream,&size_extra_field) != UNZ_OK)
        err=UNZ_ERRNO;  // 如果获取额外字段长度出错，则将错误码设置为 UNZ_ERRNO
    *poffset_local_extrafield= s->cur_file_info_internal.offset_curfile +
                                    SIZEZIPLOCALHEADER + size_filename;  // 设置本地额外字段的偏移量
    *psize_local_extrafield = (uInt)size_extra_field;  // 设置本地额外字段的大小

    *piSizeVar += (uInt)size_extra_field;  // 将额外字段长度加到 piSizeVar 上

    return err;  // 返回错误码
}

/*
  Open for reading data the current file in the zipfile.
  If there is no error and the file is opened, the return value is UNZ_OK.
*/
# 打开当前 ZIP 文件中的文件，返回文件的压缩方法、级别、原始数据、密码等信息
extern int ZEXPORT unzOpenCurrentFile3 (unzFile file, int* method,
                                            int* level, int raw, const char* password)
{
    int err=UNZ_OK;  # 初始化错误码为 UNZ_OK
    uInt iSizeVar;  # 定义无符号整型变量 iSizeVar
    unz64_s* s;  # 定义指向 unz64_s 结构体的指针 s
    file_in_zip64_read_info_s* pfile_in_zip_read_info;  # 定义指向 file_in_zip64_read_info_s 结构体的指针 pfile_in_zip_read_info
    ZPOS64_T offset_local_extrafield;  # 定义 64 位整型变量 offset_local_extrafield，表示本地额外字段的偏移量
    uInt  size_local_extrafield;  # 定义无符号整型变量 size_local_extrafield，表示本地额外字段的大小
#    ifndef NOUNCRYPT
    char source[12];  # 定义长度为 12 的字符数组 source
#    else
    if (password != NULL)  # 如果密码不为空
        return UNZ_PARAMERROR;  # 返回参数错误
#    endif

    if (file==NULL)  # 如果文件为空
        return UNZ_PARAMERROR;  # 返回参数错误
    s=(unz64_s*)file;  # 将 file 强制类型转换为 unz64_s 结构体指针，并赋值给 s
    if (!s->current_file_ok)  # 如果当前文件不可用
        return UNZ_PARAMERROR;  # 返回参数错误

    if (s->pfile_in_zip_read != NULL)  # 如果文件读取信息不为空
        unzCloseCurrentFile(file);  # 关闭当前文件

    if (unz64local_CheckCurrentFileCoherencyHeader(s,&iSizeVar, &offset_local_extrafield,&size_local_extrafield)!=UNZ_OK)  # 检查当前文件的一致性头部信息，如果不是 UNZ_OK
        return UNZ_BADZIPFILE;  # 返回坏的 ZIP 文件错误

    pfile_in_zip_read_info = (file_in_zip64_read_info_s*)ALLOC(sizeof(file_in_zip64_read_info_s));  # 分配 file_in_zip64_read_info_s 结构体大小的内存空间，并赋值给 pfile_in_zip_read_info
    if (pfile_in_zip_read_info==NULL)  # 如果分配内存失败
        return UNZ_INTERNALERROR;  # 返回内部错误

    pfile_in_zip_read_info->read_buffer=(char*)ALLOC(UNZ_BUFSIZE);  # 分配 UNZ_BUFSIZE 大小的内存空间，并赋值给 read_buffer
    pfile_in_zip_read_info->offset_local_extrafield = offset_local_extrafield;  # 将本地额外字段的偏移量赋值给 offset_local_extrafield
    pfile_in_zip_read_info->size_local_extrafield = size_local_extrafield;  # 将本地额外字段的大小赋值给 size_local_extrafield
    pfile_in_zip_read_info->pos_local_extrafield=0;  # 将本地额外字段的位置赋值为 0
    pfile_in_zip_read_info->raw=raw;  # 将原始数据标志赋值给 raw

    if (pfile_in_zip_read_info->read_buffer==NULL)  # 如果读取缓冲区为空
    {
        TRYFREE(pfile_in_zip_read_info);  # 释放 pfile_in_zip_read_info 的内存空间
        return UNZ_INTERNALERROR;  # 返回内部错误
    }

    pfile_in_zip_read_info->stream_initialised=0;  # 将流初始化标志赋值为 0

    if (method!=NULL)  # 如果方法不为空
        *method = (int)s->cur_file_info.compression_method;  # 将当前文件信息的压缩方法赋值给 method

    if (level!=NULL)  # 如果级别不为空
    {
        *level = 6;  # 将级别赋值为 6
        switch (s->cur_file_info.flag & 0x06)  # 根据当前文件信息的标志位进行判断
        {
          case 6 : *level = 1; break;  # 如果标志位为 6，将级别赋值为 1
          case 4 : *level = 2; break;  # 如果标志位为 4，将级别赋值为 2
          case 2 : *level = 9; break;  # 如果标志位为 2，将级别赋值为 9
        }
    }

    if ((s->cur_file_info.compression_method!=0) &&  # 如果当前文件信息的压缩方法不为 0
    /* #ifdef HAVE_BZIP2 */
        // 如果定义了 HAVE_BZIP2 并且当前文件的压缩方法不是 Z_BZIP2ED
        (s->cur_file_info.compression_method!=Z_BZIP2ED) &&
    /* #endif */
        // 并且当前文件的压缩方法不是 Z_DEFLATED
        (s->cur_file_info.compression_method!=Z_DEFLATED))

        // 设置错误码为 UNZ_BADZIPFILE
        err=UNZ_BADZIPFILE;

    // 设置文件内读取信息的 CRC32 等于当前文件信息的 CRC32
    pfile_in_zip_read_info->crc32_wait=s->cur_file_info.crc;
    // 设置文件内读取信息的 CRC32 为 0
    pfile_in_zip_read_info->crc32=0;
    // 设置文件内读取信息的 total_out_64 为 0
    pfile_in_zip_read_info->total_out_64=0;
    // 设置文件内读取信息的压缩方法为当前文件信息的压缩方法
    pfile_in_zip_read_info->compression_method = s->cur_file_info.compression_method;
    // 设置文件内读取信息的 filestream 为 s 的 filestream
    pfile_in_zip_read_info->filestream=s->filestream;
    // 设置文件内读取信息的 z_filefunc 为 s 的 z_filefunc
    pfile_in_zip_read_info->z_filefunc=s->z_filefunc;
    // 设置文件内读取信息的 byte_before_the_zipfile 为 s 的 byte_before_the_zipfile
    pfile_in_zip_read_info->byte_before_the_zipfile=s->byte_before_the_zipfile;

    // 设置文件内读取信息的 stream 的 total_out 为 0
    pfile_in_zip_read_info->stream.total_out = 0;

    // 如果当前文件的压缩方法是 Z_BZIP2ED 并且不是原始数据
    if ((s->cur_file_info.compression_method==Z_BZIP2ED) && (!raw))
    {
#ifdef HAVE_BZIP2
      // 设置文件内读取信息的 bstream 的各项属性为 0
      pfile_in_zip_read_info->bstream.bzalloc = (void *(*) (void *, int, int))0;
      pfile_in_zip_read_info->bstream.bzfree = (free_func)0;
      pfile_in_zip_read_info->bstream.opaque = (voidpf)0;
      pfile_in_zip_read_info->bstream.state = (voidpf)0;

      // 设置文件内读取信息的 stream 的各项属性为 0
      pfile_in_zip_read_info->stream.zalloc = (alloc_func)0;
      pfile_in_zip_read_info->stream.zfree = (free_func)0;
      pfile_in_zip_read_info->stream.opaque = (voidpf)0;
      pfile_in_zip_read_info->stream.next_in = (voidpf)0;
      pfile_in_zip_read_info->stream.avail_in = 0;

      // 初始化 BZIP2 解压缩流
      err=BZ2_bzDecompressInit(&pfile_in_zip_read_info->bstream, 0, 0);
      // 如果初始化成功，设置文件内读取信息的 stream_initialised 为 Z_BZIP2ED
      if (err == Z_OK)
        pfile_in_zip_read_info->stream_initialised=Z_BZIP2ED;
      // 否则释放内存并返回错误码
      else
      {
        TRYFREE(pfile_in_zip_read_info->read_buffer);
        TRYFREE(pfile_in_zip_read_info);
        return err;
      }
#else
      // 如果没有定义 HAVE_BZIP2，设置文件内读取信息的 raw 为 1
      pfile_in_zip_read_info->raw=1;
#endif
    }
    // 如果当前文件的压缩方法是 Z_DEFLATED 并且不是原始数据
    else if ((s->cur_file_info.compression_method==Z_DEFLATED) && (!raw))
    {
      // 设置流的内存分配函数为0
      pfile_in_zip_read_info->stream.zalloc = (alloc_func)0;
      // 设置流的内存释放函数为0
      pfile_in_zip_read_info->stream.zfree = (free_func)0;
      // 设置流的opaque字段为0
      pfile_in_zip_read_info->stream.opaque = (voidpf)0;
      // 设置流的下一个输入位置为0
      pfile_in_zip_read_info->stream.next_in = 0;
      // 设置流的可用输入大小为0
      pfile_in_zip_read_info->stream.avail_in = 0;

      // 初始化解压缩流
      err=inflateInit2(&pfile_in_zip_read_info->stream, -MAX_WBITS);
      // 如果初始化成功，设置流的初始化状态为Z_DEFLATED
      if (err == Z_OK)
        pfile_in_zip_read_info->stream_initialised=Z_DEFLATED;
      // 如果初始化失败
      else
      {
        // 释放读取缓冲区
        TRYFREE(pfile_in_zip_read_info->read_buffer);
        // 释放文件读取信息
        TRYFREE(pfile_in_zip_read_info);
        // 返回错误码
        return err;
      }
        /* windowBits is passed < 0 to tell that there is no zlib header.
         * Note that in this case inflate *requires* an extra "dummy" byte
         * after the compressed stream in order to complete decompression and
         * return Z_STREAM_END.
         * In unzip, i don't wait absolutely Z_STREAM_END because I known the
         * size of both compressed and uncompressed data
         */
    }
    // 设置剩余待读取的压缩数据大小
    pfile_in_zip_read_info->rest_read_compressed =
            s->cur_file_info.compressed_size ;
    // 设置剩余待读取的未压缩数据大小
    pfile_in_zip_read_info->rest_read_uncompressed =
            s->cur_file_info.uncompressed_size ;

    // 设置文件在ZIP文件中的位置
    pfile_in_zip_read_info->pos_in_zipfile =
            s->cur_file_info_internal.offset_curfile + SIZEZIPLOCALHEADER +
              iSizeVar;

    // 设置流的可用输入大小为0
    pfile_in_zip_read_info->stream.avail_in = (uInt)0;

    // 设置当前ZIP文件的读取信息
    s->pfile_in_zip_read = pfile_in_zip_read_info;
    // 设置加密标志为0
    s->encrypted = 0;
# 如果未定义 NOUNCRYPT
if (password != NULL)
{
    int i;
    s->pcrc_32_tab = get_crc_table();  # 获取 CRC 表
    init_keys(password,s->keys,s->pcrc_32_tab);  # 使用密码初始化加密密钥
    if (ZSEEK64(s->z_filefunc, s->filestream,
              s->pfile_in_zip_read->pos_in_zipfile +
                 s->pfile_in_zip_read->byte_before_the_zipfile,
              SEEK_SET)!=0)
        return UNZ_INTERNALERROR;  # 如果定位到 ZIP 文件内部位置失败，返回内部错误
    if(ZREAD64(s->z_filefunc, s->filestream,source, 12)<12)
        return UNZ_INTERNALERROR;  # 如果读取文件内容失败，返回内部错误

    for (i = 0; i<12; i++)
        zdecode(s->keys,s->pcrc_32_tab,source[i]);  # 对文件内容进行解密

    s->pfile_in_zip_read->pos_in_zipfile+=12;  # 更新文件位置
    s->encrypted=1;  # 标记文件已加密
}
# 返回操作成功
return UNZ_OK;
}

# 打开当前文件
extern int ZEXPORT unzOpenCurrentFile (unzFile file)
{
    return unzOpenCurrentFile3(file, NULL, NULL, 0, NULL);  # 调用 unzOpenCurrentFile3 函数
}

# 打开当前文件并使用密码
extern int ZEXPORT unzOpenCurrentFilePassword (unzFile file, const char*  password)
{
    return unzOpenCurrentFile3(file, NULL, NULL, 0, password);  # 调用 unzOpenCurrentFile3 函数
}

# 打开当前文件并获取压缩方法、级别和原始标志
extern int ZEXPORT unzOpenCurrentFile2 (unzFile file, int* method, int* level, int raw)
{
    return unzOpenCurrentFile3(file, method, level, raw, NULL);  # 调用 unzOpenCurrentFile3 函数
}

# 获取当前文件的流位置（64 位）
extern ZPOS64_T ZEXPORT unzGetCurrentFileZStreamPos64( unzFile file)
{
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    s=(unz64_s*)file;
    if (file==NULL)
        return 0;  # 如果文件为空，返回 0
    pfile_in_zip_read_info=s->pfile_in_zip_read;
    if (pfile_in_zip_read_info==NULL)
        return 0;  # 如果文件内部信息为空，返回 0
    return pfile_in_zip_read_info->pos_in_zipfile +
                         pfile_in_zip_read_info->byte_before_the_zipfile;  # 返回文件流位置
}

# 从当前文件中读取字节
# buf 包含数据的缓冲区
# len 缓冲区的大小
# 如果成功复制了一些字节，则返回复制的字节数
# 如果到达文件末尾，则返回 0
# 如果出现错误，则返回 <0 的错误代码
    # 返回值为 UNZ_ERRNO 表示 IO 错误，或者返回 zLib 错误表示解压缩错误
extern int ZEXPORT unzReadCurrentFile  (unzFile file, voidp buf, unsigned len)
{
    // 读取当前文件的内容到缓冲区
    int err=UNZ_OK;
    uInt iRead = 0;
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    // 如果文件为空，则返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    // 如果文件读取信息为空，则返回参数错误
    if (pfile_in_zip_read_info==NULL)
        return UNZ_PARAMERROR;

    // 如果读取缓冲区为空，则返回文件列表结束
    if (pfile_in_zip_read_info->read_buffer == NULL)
        return UNZ_END_OF_LIST_OF_FILE;
    // 如果长度为0，则返回0
    if (len==0)
        return 0;

    // 设置输出流的起始位置和可用长度
    pfile_in_zip_read_info->stream.next_out = (Bytef*)buf;
    pfile_in_zip_read_info->stream.avail_out = (uInt)len;

    // 如果长度大于剩余未压缩的数据长度，并且不是原始数据，则设置可用长度为剩余未压缩的数据长度
    if ((len>pfile_in_zip_read_info->rest_read_uncompressed) &&
        (!(pfile_in_zip_read_info->raw)))
        pfile_in_zip_read_info->stream.avail_out =
            (uInt)pfile_in_zip_read_info->rest_read_uncompressed;

    // 如果长度大于剩余未压缩和输入流可用长度之和，并且是原始数据，则设置可用长度为剩余未压缩和输入流可用长度之和
    if ((len>pfile_in_zip_read_info->rest_read_compressed+
           pfile_in_zip_read_info->stream.avail_in) &&
         (pfile_in_zip_read_info->raw))
        pfile_in_zip_read_info->stream.avail_out =
            (uInt)pfile_in_zip_read_info->rest_read_compressed+
            pfile_in_zip_read_info->stream.avail_in;

    // 循环读取数据直到输出流可用长度为0
    while (pfile_in_zip_read_info->stream.avail_out>0)
        // 如果压缩流中没有可用的数据，并且还有剩余的压缩数据未读取
        if ((pfile_in_zip_read_info->stream.avail_in==0) &&
            (pfile_in_zip_read_info->rest_read_compressed>0))
        {
            // 设置每次读取的最大字节数
            uInt uReadThis = UNZ_BUFSIZE;
            // 如果剩余的压缩数据不足以填满缓冲区，则设置实际读取的字节数
            if (pfile_in_zip_read_info->rest_read_compressed<uReadThis)
                uReadThis = (uInt)pfile_in_zip_read_info->rest_read_compressed;
            // 如果实际读取的字节数为0，则返回文件结束标识
            if (uReadThis == 0)
                return UNZ_EOF;
            // 移动文件指针到压缩文件中的指定位置
            if (ZSEEK64(pfile_in_zip_read_info->z_filefunc,
                      pfile_in_zip_read_info->filestream,
                      pfile_in_zip_read_info->pos_in_zipfile +
                         pfile_in_zip_read_info->byte_before_the_zipfile,
                         ZLIB_FILEFUNC_SEEK_SET)!=0)
                return UNZ_ERRNO;
            // 从文件中读取指定字节数的数据到缓冲区
            if (ZREAD64(pfile_in_zip_read_info->z_filefunc,
                      pfile_in_zip_read_info->filestream,
                      pfile_in_zip_read_info->read_buffer,
                      uReadThis)!=uReadThis)
                return UNZ_ERRNO;
# 如果未定义 NOUNCRYPT，并且文件被加密，则执行以下操作
if(s->encrypted)
{
    # 定义变量 i，并初始化为 0
    uInt i;
    # 遍历读取的数据，对每个字节进行解密操作
    for(i=0;i<uReadThis;i++)
        pfile_in_zip_read_info->read_buffer[i] =
            zdecode(s->keys,s->pcrc_32_tab,
                    pfile_in_zip_read_info->read_buffer[i]);
}
# 如果条件满足，则执行以下代码块
            pfile_in_zip_read_info->pos_in_zipfile += uReadThis;  # 更新 ZIP 文件中的当前位置
            pfile_in_zip_read_info->rest_read_compressed-=uReadThis;  # 更新压缩数据的剩余长度
            pfile_in_zip_read_info->stream.next_in = (Bytef*)pfile_in_zip_read_info->read_buffer;  # 设置输入数据的起始位置
            pfile_in_zip_read_info->stream.avail_in = (uInt)uReadThis;  # 设置输入数据的可用长度

        # 如果条件满足，则执行以下代码块
        if ((pfile_in_zip_read_info->compression_method==0) || (pfile_in_zip_read_info->raw)):
            uInt uDoCopy,i ;  # 定义变量 uDoCopy 和 i

            # 如果满足条件，则返回 UNZ_EOF 或者已读取的字节数
            if ((pfile_in_zip_read_info->stream.avail_in == 0) && (pfile_in_zip_read_info->rest_read_compressed == 0)):
                return (iRead==0) ? UNZ_EOF : (int)iRead;

            # 如果输出数据的可用长度小于输入数据的可用长度，则将 uDoCopy 设置为输出数据的可用长度，否则设置为输入数据的可用长度
            if (pfile_in_zip_read_info->stream.avail_out < pfile_in_zip_read_info->stream.avail_in):
                uDoCopy = pfile_in_zip_read_info->stream.avail_out ;
            else:
                uDoCopy = pfile_in_zip_read_info->stream.avail_in ;

            # 将输入数据复制到输出数据
            for (i=0;i<uDoCopy;i++):
                *(pfile_in_zip_read_info->stream.next_out+i) = *(pfile_in_zip_read_info->stream.next_in+i);

            # 更新已输出的总字节数
            pfile_in_zip_read_info->total_out_64 = pfile_in_zip_read_info->total_out_64 + uDoCopy;

            # 计算并更新 CRC32 校验值
            pfile_in_zip_read_info->crc32 = crc32(pfile_in_zip_read_info->crc32, pfile_in_zip_read_info->stream.next_out, uDoCopy);
            pfile_in_zip_read_info->rest_read_uncompressed-=uDoCopy;  # 更新未压缩数据的剩余长度
            pfile_in_zip_read_info->stream.avail_in -= uDoCopy;  # 更新输入数据的可用长度
            pfile_in_zip_read_info->stream.avail_out -= uDoCopy;  # 更新输出数据的可用长度
            pfile_in_zip_read_info->stream.next_out += uDoCopy;  # 更新输出数据的起始位置
            pfile_in_zip_read_info->stream.next_in += uDoCopy;  # 更新输入数据的起始位置
            pfile_in_zip_read_info->stream.total_out += uDoCopy;  # 更新已输出的总字节数
            iRead += uDoCopy;  # 更新已读取的字节数
#endif
        } // end Z_BZIP2ED
        else
        {
            ZPOS64_T uTotalOutBefore,uTotalOutAfter;
            const Bytef *bufBefore;
            ZPOS64_T uOutThis;
            int flush=Z_SYNC_FLUSH;

            uTotalOutBefore = pfile_in_zip_read_info->stream.total_out;
            bufBefore = pfile_in_zip_read_info->stream.next_out;

            /*
            if ((pfile_in_zip_read_info->rest_read_uncompressed ==
                     pfile_in_zip_read_info->stream.avail_out) &&
                (pfile_in_zip_read_info->rest_read_compressed == 0))
                flush = Z_FINISH;
            */
            // 使用inflate函数解压缩数据
            err=inflate(&pfile_in_zip_read_info->stream,flush);

            // 如果解压缩成功但是有错误消息，则将错误码设置为Z_DATA_ERROR
            if ((err>=0) && (pfile_in_zip_read_info->stream.msg!=NULL))
              err = Z_DATA_ERROR;

            uTotalOutAfter = pfile_in_zip_read_info->stream.total_out;
            /* Detect overflow, because z_stream.total_out is uLong (32 bits) */
            // 检测溢出，因为z_stream.total_out是uLong（32位）
            if (uTotalOutAfter<uTotalOutBefore)
                uTotalOutAfter += 1LL << 32; /* Add maximum value of uLong + 1 */
            uOutThis = uTotalOutAfter-uTotalOutBefore;

            // 更新已解压缩的总字节数
            pfile_in_zip_read_info->total_out_64 = pfile_in_zip_read_info->total_out_64 + uOutThis;

            // 计算CRC32校验值
            pfile_in_zip_read_info->crc32 =
                crc32(pfile_in_zip_read_info->crc32,bufBefore,
                        (uInt)(uOutThis));

            // 更新剩余未解压缩的字节数
            pfile_in_zip_read_info->rest_read_uncompressed -=
                uOutThis;

            // 更新已读取的字节数
            iRead += (uInt)(uTotalOutAfter - uTotalOutBefore);

            // 如果解压缩到达流的末尾，则返回UNZ_EOF或已读取的字节数
            if (err==Z_STREAM_END)
                return (iRead==0) ? UNZ_EOF : (int)iRead;
            // 如果解压缩出现错误，则跳出循环
            if (err!=Z_OK)
                break;
        }
    }

    // 如果解压缩成功，则返回已读取的字节数
    if (err==Z_OK)
        return (int)iRead;
    // 否则返回错误码
    return err;
}


/*
  Give the current position in uncompressed data
*/
extern z_off_t ZEXPORT unztell (unzFile file)
{
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    if (file==NULL)
        return UNZ_PARAMERROR;
    # 将文件转换为unz64_s类型的指针
    s=(unz64_s*)file;
    # 从unz64_s结构体中获取pfile_in_zip_read指针
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    # 如果pfile_in_zip_read_info指针为空，则返回参数错误
    if (pfile_in_zip_read_info==NULL)
        return UNZ_PARAMERROR;

    # 返回pfile_in_zip_read_info指针中stream.total_out的值，即已经读取的数据大小
    return (z_off_t)pfile_in_zip_read_info->stream.total_out;
}

extern ZPOS64_T ZEXPORT unztell64 (unzFile file)
{
    // 声明变量
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    // 如果文件为空，返回-1
    if (file==NULL)
        return (ZPOS64_T)-1;
    // 将文件转换为unz64_s类型
    s=(unz64_s*)file;
    // 获取文件信息
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    // 如果文件信息为空，返回-1
    if (pfile_in_zip_read_info==NULL)
        return (ZPOS64_T)-1;

    // 返回文件读取的总字节数
    return pfile_in_zip_read_info->total_out_64;
}


/*
  return 1 if the end of file was reached, 0 elsewhere
*/
extern int ZEXPORT unzeof (unzFile file)
{
    // 声明变量
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    // 如果文件为空，返回UNZ_PARAMERROR
    if (file==NULL)
        return UNZ_PARAMERROR;
    // 将文件转换为unz64_s类型
    s=(unz64_s*)file;
    // 获取文件信息
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    // 如果文件信息为空，返回UNZ_PARAMERROR
    if (pfile_in_zip_read_info==NULL)
        return UNZ_PARAMERROR;

    // 如果剩余未压缩的数据为0，返回1，否则返回0
    if (pfile_in_zip_read_info->rest_read_uncompressed == 0)
        return 1;
    else
        return 0;
}



/*
Read extra field from the current file (opened by unzOpenCurrentFile)
This is the local-header version of the extra field (sometimes, there is
more info in the local-header version than in the central-header)

  if buf==NULL, it return the size of the local extra field that can be read

  if buf!=NULL, len is the size of the buffer, the extra header is copied in
    buf.
  the return value is the number of bytes copied in buf, or (if <0)
    the error code
*/
extern int ZEXPORT unzGetLocalExtrafield (unzFile file, voidp buf, unsigned len)
{
    // 声明变量
    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    uInt read_now;
    ZPOS64_T size_to_read;

    // 如果文件为空，返回UNZ_PARAMERROR
    if (file==NULL)
        return UNZ_PARAMERROR;
    // 将文件转换为unz64_s类型
    s=(unz64_s*)file;
    // 获取文件信息
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    // 如果文件信息为空，返回UNZ_PARAMERROR
    if (pfile_in_zip_read_info==NULL)
        return UNZ_PARAMERROR;

    // 计算需要读取的额外字段大小
    size_to_read = (pfile_in_zip_read_info->size_local_extrafield -
                pfile_in_zip_read_info->pos_local_extrafield);

    // 如果buf为空，返回需要读取的额外字段大小
    if (buf==NULL)
        return (int)size_to_read;

    // 如果len大于需要读取的额外字段大小，设置读取大小为size_to_read，否则设置为len
    if (len>size_to_read)
        read_now = (uInt)size_to_read;
    else
        read_now = (uInt)len ;
    # 如果读取长度为0，则直接返回0
    if (read_now==0)
        return 0;

    # 使用文件流的偏移量和本地额外字段的位置，定位到文件流的指定位置
    if (ZSEEK64(pfile_in_zip_read_info->z_filefunc,
              pfile_in_zip_read_info->filestream,
              pfile_in_zip_read_info->offset_local_extrafield +
              pfile_in_zip_read_info->pos_local_extrafield,
              ZLIB_FILEFUNC_SEEK_SET)!=0)
        return UNZ_ERRNO;

    # 从文件流中读取指定长度的数据到缓冲区，如果读取长度不等于指定长度，则返回错误
    if (ZREAD64(pfile_in_zip_read_info->z_filefunc,
              pfile_in_zip_read_info->filestream,
              buf,read_now)!=read_now)
        return UNZ_ERRNO;

    # 返回成功读取的数据长度
    return (int)read_now;
}
/*
  关闭使用unzOpenCurrentFile打开的zip文件
  如果文件已经读取完毕但CRC校验不通过，则返回UNZ_CRCERROR
*/
extern int ZEXPORT unzCloseCurrentFile (unzFile file)
{
    int err=UNZ_OK;

    unz64_s* s;
    file_in_zip64_read_info_s* pfile_in_zip_read_info;
    if (file==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;
    pfile_in_zip_read_info=s->pfile_in_zip_read;

    if (pfile_in_zip_read_info==NULL)
        return UNZ_PARAMERROR;


    if ((pfile_in_zip_read_info->rest_read_uncompressed == 0) &&
        (!pfile_in_zip_read_info->raw))
    {
        if (pfile_in_zip_read_info->crc32 != pfile_in_zip_read_info->crc32_wait)
            err=UNZ_CRCERROR;
    }


    TRYFREE(pfile_in_zip_read_info->read_buffer);
    pfile_in_zip_read_info->read_buffer = NULL;
    if (pfile_in_zip_read_info->stream_initialised == Z_DEFLATED)
        inflateEnd(&pfile_in_zip_read_info->stream);
#ifdef HAVE_BZIP2
    else if (pfile_in_zip_read_info->stream_initialised == Z_BZIP2ED)
        BZ2_bzDecompressEnd(&pfile_in_zip_read_info->bstream);
#endif


    pfile_in_zip_read_info->stream_initialised = 0;
    TRYFREE(pfile_in_zip_read_info);

    s->pfile_in_zip_read=NULL;

    return err;
}


/*
  获取ZipFile的全局注释字符串，存储在szComment缓冲区中
  uSizeBuf是szComment缓冲区的大小
  返回复制的字节数或小于0的错误代码
*/
extern int ZEXPORT unzGetGlobalComment (unzFile file, char * szComment, uLong uSizeBuf)
{
    unz64_s* s;
    uLong uReadThis ;
    if (file==NULL)
        return (int)UNZ_PARAMERROR;
    s=(unz64_s*)file;

    uReadThis = uSizeBuf;
    if (uReadThis>s->gi.size_comment)
        uReadThis = s->gi.size_comment;

    if (ZSEEK64(s->z_filefunc,s->filestream,s->central_pos+22,ZLIB_FILEFUNC_SEEK_SET)!=0)
        return UNZ_ERRNO;

    if (uReadThis>0)
    {
      *szComment='\0';
      if (ZREAD64(s->z_filefunc,s->filestream,szComment,uReadThis)!=uReadThis)
        return UNZ_ERRNO;
    }
    # 如果注释不为空且缓冲区大小大于压缩文件的注释大小
    if ((szComment != NULL) && (uSizeBuf > s->gi.size_comment))
        # 在注释字符串末尾添加空字符，以确保字符串结尾
        *(szComment+s->gi.size_comment)='\0';
    # 返回已读取的字节数
    return (int)uReadThis;
/* Additions by RX '2004 */
// 定义了一个名为 unzGetOffset64 的函数，用于获取 ZIP 文件中某个文件的偏移量
extern ZPOS64_T ZEXPORT unzGetOffset64(unzFile file)
{
    unz64_s* s;

    // 如果文件为空，则返回偏移量 0
    if (file==NULL)
          return 0; //UNZ_PARAMERROR;
    s=(unz64_s*)file;
    // 如果当前文件不可用，则返回偏移量 0
    if (!s->current_file_ok)
      return 0;
    // 如果文件数量不为 0 且不为 0xffff，则返回偏移量 0
    if (s->gi.number_entry != 0 && s->gi.number_entry != 0xffff)
      if (s->num_file==s->gi.number_entry)
         return 0;
    // 返回当前文件在中央目录中的偏移量
    return s->pos_in_central_dir;
}

// 定义了一个名为 unzGetOffset 的函数，用于获取 ZIP 文件中某个文件的偏移量
extern uLong ZEXPORT unzGetOffset (unzFile file)
{
    ZPOS64_T offset64;

    // 如果文件为空，则返回偏移量 0
    if (file==NULL)
          return 0; //UNZ_PARAMERROR;
    // 调用 unzGetOffset64 函数获取偏移量
    offset64 = unzGetOffset64(file);
    // 将 64 位偏移量转换为 32 位并返回
    return (uLong)offset64;
}

// 定义了一个名为 unzSetOffset64 的函数，用于设置 ZIP 文件中某个文件的偏移量
extern int ZEXPORT unzSetOffset64(unzFile file, ZPOS64_T pos)
{
    unz64_s* s;
    int err;

    // 如果文件为空，则返回参数错误
    if (file==NULL)
        return UNZ_PARAMERROR;
    s=(unz64_s*)file;

    // 设置中央目录中的偏移量
    s->pos_in_central_dir = pos;
    s->num_file = s->gi.number_entry;      /* hack */
    // 获取当前文件信息并判断是否可用
    err = unz64local_GetCurrentFileInfoInternal(file,&s->cur_file_info,
                                              &s->cur_file_info_internal,
                                              NULL,0,NULL,0,NULL,0);
    s->current_file_ok = (err == UNZ_OK);
    return err;
}

// 定义了一个名为 unzSetOffset 的函数，用于设置 ZIP 文件中某个文件的偏移量
extern int ZEXPORT unzSetOffset (unzFile file, uLong pos)
{
    // 调用 unzSetOffset64 函数设置偏移量
    return unzSetOffset64(file,pos);
}
```