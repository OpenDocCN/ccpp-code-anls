# `nmap\libz\contrib\minizip\zip.c`

```cpp
/* zip.c -- IO on .zip files using zlib
   Version 1.1, February 14h, 2010
   part of the MiniZip project - ( http://www.winimage.com/zLibDll/minizip.html )

         Copyright (C) 1998-2010 Gilles Vollant (minizip) ( http://www.winimage.com/zLibDll/minizip.html )

         Modifications for Zip64 support
         Copyright (C) 2009-2010 Mathias Svensson ( http://result42.com )

         For more info read MiniZip_info.txt

         Changes
   Oct-2009 - Mathias Svensson - Remove old C style function prototypes
   Oct-2009 - Mathias Svensson - Added Zip64 Support when creating new file archives
   Oct-2009 - Mathias Svensson - Did some code cleanup and refactoring to get better overview of some functions.
   Oct-2009 - Mathias Svensson - Added zipRemoveExtraInfoBlock to strip extra field data from its ZIP64 data
                                 It is used when recreting zip archive with RAW when deleting items from a zip.
                                 ZIP64 data is automatically added to items that needs it, and existing ZIP64 data need to be removed.
   Oct-2009 - Mathias Svensson - Added support for BZIP2 as compression mode (bzip2 lib is required)
   Jan-2010 - back to unzip and minizip 1.0 name scheme, with compatibility layer
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "zlib.h"
#include "zip.h"

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

#ifndef local
#  define local static
#endif
/* compile with -Dlocal if your debugger can't find static symbols */

#ifndef VERSIONMADEBY
# define VERSIONMADEBY   (0x0) /* platform depedent */
#endif

#ifndef Z_BUFSIZE
#define Z_BUFSIZE (64*1024) //(16384)
#endif

#ifndef Z_MAXFILENAMEINZIP
#define Z_MAXFILENAMEINZIP (256)
#endif

#ifndef ALLOC
# define ALLOC(size) (malloc(size))
#endif
#ifndef TRYFREE
# define TRYFREE(p) {if (p) free(p);}
#endif
// 定义中央目录项的大小
#define SIZECENTRALDIRITEM (0x2e)
// 定义 ZIP 本地头的大小
#define SIZEZIPLOCALHEADER (0x1e)

/* 我发现一个旧的 Unix 系统（SunOS 4.1.3_U1）没有定义所有的 SEEK_*.... */

// 不确定这个在所有平台上都能工作
#define MAKEULONG64(a, b) ((ZPOS64_T)(((unsigned long)(a)) | ((ZPOS64_T)((unsigned long)(b))) << 32))

// 如果未定义 SEEK_CUR，则定义为 1
#ifndef SEEK_CUR
#define SEEK_CUR    1
#endif

// 如果未定义 SEEK_END，则定义为 2
#ifndef SEEK_END
#define SEEK_END    2
#endif

// 如果未定义 SEEK_SET，则定义为 0
#ifndef SEEK_SET
#define SEEK_SET    0
#endif

// 如果未定义 DEF_MEM_LEVEL，则根据 MAX_MEM_LEVEL 的值进行定义
#ifndef DEF_MEM_LEVEL
#if MAX_MEM_LEVEL >= 8
#  define DEF_MEM_LEVEL 8
#else
#  define DEF_MEM_LEVEL  MAX_MEM_LEVEL
#endif
#endif

// 定义 ZIP 版权信息
const char zip_copyright[] =" zip 1.01 Copyright 1998-2004 Gilles Vollant - http://www.winimage.com/zLibDll";

// 定义数据块的大小
#define SIZEDATA_INDATABLOCK (4096-(4*4))

// 定义本地头的魔数
#define LOCALHEADERMAGIC    (0x04034b50)
// 定义中央目录头的魔数
#define CENTRALHEADERMAGIC  (0x02014b50)
// 定义结束头的魔数
#define ENDHEADERMAGIC      (0x06054b50)
// 定义 ZIP64 结束头的魔数
#define ZIP64ENDHEADERMAGIC      (0x6064b50)
// 定义 ZIP64 结束位置头的魔数
#define ZIP64ENDLOCHEADERMAGIC   (0x7064b50)

// 定义本地头标志的偏移量
#define FLAG_LOCALHEADER_OFFSET (0x06)
// 定义本地头 CRC 的偏移量
#define CRC_LOCALHEADER_OFFSET  (0x0e)

// 定义中央目录头的大小
#define SIZECENTRALHEADER (0x2e) /* 46 */

// 定义链表数据块内部结构
typedef struct linkedlist_datablock_internal_s
{
  struct linkedlist_datablock_internal_s* next_datablock;
  uLong  avail_in_this_block;
  uLong  filled_in_this_block;
  uLong  unused; /* 用于将来使用和对齐 */
  unsigned char data[SIZEDATA_INDATABLOCK];
} linkedlist_datablock_internal;

// 定义链表数据结构
typedef struct linkedlist_data_s
{
    linkedlist_datablock_internal* first_block;
    linkedlist_datablock_internal* last_block;
} linkedlist_data;

// 定义解压缩流结构
typedef struct
{
    z_stream stream;            /* 用于解压缩的 zLib 流结构 */
#ifdef HAVE_BZIP2
    bz_stream bstream;          /* 用于 bzip2 解压缩的 bzLib 流结构 */
#endif

    int  stream_initialised;    /* 如果流已初始化，则为 1 */
    uInt pos_in_buffered_data;  /* 缓冲数据中最后写入的字节位置 */

    ZPOS64_T pos_local_header;     /* 当前正在写入的文件的本地头的偏移量 */
    /* 用于存储当前文件的中央目录数据 */
    char* central_header;
    /* 当前文件中央目录额外数据的大小 */
    uLong size_centralExtra;
    /* 当前文件中央目录的大小 */
    uLong size_centralheader;
    /* 分配给中央目录的额外字节，但未被使用 */
    uLong size_centralExtraFree;
    /* 当前正在写入的文件的标志 */
    uLong flag;

    /* 当前文件的压缩方法 */
    int  method;
    /* 直接写入原始数据的标志，1表示直接写入原始数据 */
    int  raw;
    /* 包含待写入的压缩数据的缓冲区 */
    Byte buffered_data[Z_BUFSIZE];
    /* DOS 日期 */
    uLong dosDate;
    /* CRC32 校验值 */
    uLong crc32;
    /* 加密标志 */
    int  encrypt;
    /* ZIP64 扩展信息的标志，在额外字段中添加 ZIP64 扩展信息 */
    int  zip64;
    /* ZIP64 扩展信息的位置 */
    ZPOS64_T pos_zip64extrainfo;
    /* 总压缩数据大小 */
    ZPOS64_T totalCompressedData;
    /* 总未压缩数据大小 */
    ZPOS64_T totalUncompressedData;
#ifndef NOCRYPT
    unsigned long keys[3];     /* 定义伪随机序列的密钥 */
    const z_crc_t* pcrc_32_tab;    /* 指向 CRC32 表的指针 */
    unsigned crypt_header_size;    /* 加密头部的大小 */
#endif
} curfile64_info;    /* 定义了 curfile64_info 结构体 */

typedef struct
{
    zlib_filefunc64_32_def z_filefunc;    /* zlib 文件操作函数 */
    voidpf filestream;        /* 指向 zipfile 的 IO 结构 */
    linkedlist_data central_dir;    /* 构造中央目录的数据块 */
    int  in_opened_file_inzip;  /* 如果 ZIP 文件中当前正在写入文件，则为 1 */
    curfile64_info ci;            /* 当前正在写入的文件的信息 */

    ZPOS64_T begin_pos;            /* ZIP 文件的起始位置 */
    ZPOS64_T add_position_when_writing_offset;    /* 写入偏移时的附加位置 */
    ZPOS64_T number_entry;        /* 条目数量 */

#ifndef NO_ADDFILEINEXISTINGZIP
    char *globalcomment;    /* 全局注释 */
#endif

} zip64_internal;    /* 定义了 zip64_internal 结构体 */


#ifndef NOCRYPT
#define INCLUDECRYPTINGCODE_IFCRYPTALLOWED
#include "crypt.h"    /* 如果允许加密，则包含加密代码 */
#endif

local linkedlist_datablock_internal* allocate_new_datablock()
{
    linkedlist_datablock_internal* ldi;
    ldi = (linkedlist_datablock_internal*)
                 ALLOC(sizeof(linkedlist_datablock_internal));    /* 分配新的数据块 */
    if (ldi!=NULL)
    {
        ldi->next_datablock = NULL ;    /* 下一个数据块为空 */
        ldi->filled_in_this_block = 0 ;    /* 当前数据块中填充的字节数为 0 */
        ldi->avail_in_this_block = SIZEDATA_INDATABLOCK ;    /* 当前数据块中可用的字节数 */
    }
    return ldi;    /* 返回新分配的数据块 */
}

local void free_datablock(linkedlist_datablock_internal* ldi)
{
    while (ldi!=NULL)
    {
        linkedlist_datablock_internal* ldinext = ldi->next_datablock;    /* 下一个数据块 */
        TRYFREE(ldi);    /* 释放当前数据块 */
        ldi = ldinext;    /* 指向下一个数据块 */
    }
}

local void init_linkedlist(linkedlist_data* ll)
{
    ll->first_block = ll->last_block = NULL;    /* 初始化链表，首尾均为空 */
}

local void free_linkedlist(linkedlist_data* ll)
{
    free_datablock(ll->first_block);    /* 释放链表的数据块 */
    ll->first_block = ll->last_block = NULL;    /* 链表首尾均为空 */
}


local int add_data_in_datablock(linkedlist_data* ll, const void* buf, uLong len)
{
    linkedlist_datablock_internal* ldi;
    const unsigned char* from_copy;

    if (ll==NULL)
        return ZIP_INTERNALERROR;    /* 如果链表为空，返回内部错误 */

    if (ll->last_block == NULL)
    {
        // 如果链表的第一个数据块为空，则分配一个新的数据块，并将其设置为链表的第一个和最后一个数据块
        ll->first_block = ll->last_block = allocate_new_datablock();
        // 如果无法分配新的数据块，则返回 ZIP_INTERNALERROR
        if (ll->first_block == NULL)
            return ZIP_INTERNALERROR;
    }
    
    // 将链表的最后一个数据块赋给 ldi
    ldi = ll->last_block;
    // 将 buf 强制转换为无符号字符指针赋给 from_copy
    from_copy = (unsigned char*)buf;
    
    // 当还有数据需要复制时执行循环
    while (len>0)
    {
        uInt copy_this; // 需要复制的数据长度
        uInt i; // 循环变量
        unsigned char* to_copy; // 指向需要复制的位置
    
        // 如果当前数据块中没有可用的空间
        if (ldi->avail_in_this_block==0)
        {
            // 分配一个新的数据块，并将其设置为当前数据块的下一个数据块
            ldi->next_datablock = allocate_new_datablock();
            // 如果无法分配新的数据块，则返回 ZIP_INTERNALERROR
            if (ldi->next_datablock == NULL)
                return ZIP_INTERNALERROR;
            // 将新分配的数据块设置为当前数据块，并更新链表的最后一个数据块
            ldi = ldi->next_datablock ;
            ll->last_block = ldi;
        }
    
        // 如果当前数据块中可用空间小于需要复制的数据长度，则将可用空间长度赋给 copy_this
        if (ldi->avail_in_this_block < len)
            copy_this = (uInt)ldi->avail_in_this_block;
        else
            copy_this = (uInt)len;
    
        // 将当前数据块中需要复制的位置赋给 to_copy
        to_copy = &(ldi->data[ldi->filled_in_this_block]);
    
        // 循环复制数据
        for (i=0;i<copy_this;i++)
            *(to_copy+i)=*(from_copy+i);
    
        // 更新当前数据块中已填充的长度和可用空间长度
        ldi->filled_in_this_block += copy_this;
        ldi->avail_in_this_block -= copy_this;
        // 更新 from_copy 和 len，准备复制下一段数据
        from_copy += copy_this ;
        len -= copy_this;
    }
    // 返回 ZIP_OK
    return ZIP_OK;
/****************************************************************************/

#ifndef NO_ADDFILEINEXISTINGZIP
/* ===========================================================================
   Inputs a long in LSB order to the given file
   nbByte == 1, 2 ,4 or 8 (byte, short or long, ZPOS64_T)
*/
// 将一个长整型按照 LSB（最低有效字节在前）的顺序写入给定的文件
// nbByte == 1, 2 ,4 or 8 (分别表示字节、短整型、长整型、ZPOS64_T)
local int zip64local_putValue OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, ZPOS64_T x, int nbByte));
local int zip64local_putValue (const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, ZPOS64_T x, int nbByte)
{
    unsigned char buf[8];
    int n;
    for (n = 0; n < nbByte; n++)
    {
        buf[n] = (unsigned char)(x & 0xff);
        x >>= 8;
    }
    if (x != 0)
      {     /* data overflow - hack for ZIP64 (X Roche) */
      for (n = 0; n < nbByte; n++)
        {
          buf[n] = 0xff;
        }
      }

    if (ZWRITE64(*pzlib_filefunc_def,filestream,buf,(uLong)nbByte)!=(uLong)nbByte)
        return ZIP_ERRNO;
    else
        return ZIP_OK;
}

local void zip64local_putValue_inmemory OF((void* dest, ZPOS64_T x, int nbByte));
local void zip64local_putValue_inmemory (void* dest, ZPOS64_T x, int nbByte)
{
    unsigned char* buf=(unsigned char*)dest;
    int n;
    for (n = 0; n < nbByte; n++) {
        buf[n] = (unsigned char)(x & 0xff);
        x >>= 8;
    }

    if (x != 0)
    {     /* data overflow - hack for ZIP64 */
       for (n = 0; n < nbByte; n++)
       {
          buf[n] = 0xff;
       }
    }
}

/****************************************************************************/

// 将 tm_zip 结构体表示的日期转换为 DOS 格式的日期
local uLong zip64local_TmzDateToDosDate(const tm_zip* ptm)
{
    uLong year = (uLong)ptm->tm_year;
    if (year>=1980)
        year-=1980;
    else if (year>=80)
        year-=80;
    return
      (uLong) (((uLong)(ptm->tm_mday) + (32 * (uLong)(ptm->tm_mon+1)) + (512 * year)) << 16) |
        (((uLong)ptm->tm_sec/2) + (32 * (uLong)ptm->tm_min) + (2048 * (uLong)ptm->tm_hour));
}
# 定义一个本地函数，用于从文件流中获取一个字节的数据
local int zip64local_getByte OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, int *pi));

# 实现获取一个字节的数据的函数
local int zip64local_getByte(const zlib_filefunc64_32_def* pzlib_filefunc_def,voidpf filestream,int* pi)
{
    # 定义一个无符号字符变量
    unsigned char c;
    # 从文件流中读取一个字节的数据
    int err = (int)ZREAD64(*pzlib_filefunc_def,filestream,&c,1);
    # 如果成功读取一个字节的数据
    if (err==1)
    {
        # 将读取的字节数据赋值给指针指向的变量
        *pi = (int)c;
        # 返回成功状态
        return ZIP_OK;
    }
    else
    {
        # 如果读取失败
        if (ZERROR64(*pzlib_filefunc_def,filestream))
            # 返回错误状态
            return ZIP_ERRNO;
        else
            # 返回文件结束状态
            return ZIP_EOF;
    }
}

# 定义一个本地函数，用于从文件流中获取两个字节的数据
local int zip64local_getShort OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, uLong *pX));

# 实现获取两个字节的数据的函数
local int zip64local_getShort (const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, uLong* pX)
{
    # 定义一个无符号长整型变量
    uLong x ;
    # 定义一个整型变量
    int i = 0;
    # 定义一个错误状态变量
    int err;

    # 调用获取一个字节的数据的函数
    err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将获取的字节数据赋值给无符号长整型变量
    x = (uLong)i;

    # 如果获取成功
    if (err==ZIP_OK)
        # 继续获取下一个字节的数据
        err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将获取的字节数据左移8位，并加到无符号长整型变量上
    x += ((uLong)i)<<8;

    # 如果获取成功
    if (err==ZIP_OK)
        # 将结果赋值给指针指向的变量
        *pX = x;
    else
        # 将结果置为0
        *pX = 0;
    # 返回获取状态
    return err;
}

# 定义一个本地函数，用于从文件流中获取四个字节的数据
local int zip64local_getLong OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, uLong *pX));

# 实现获取四个字节的数据的函数
local int zip64local_getLong (const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, uLong* pX)
{
    # 定义一个无符号长整型变量
    uLong x ;
    # 定义一个整型变量
    int i = 0;
    # 定义一个错误状态变量
    int err;

    # 调用获取一个字节的数据的函数
    err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将获取的字节数据赋值给无符号长整型变量
    x = (uLong)i;

    # 如果获取成功
    if (err==ZIP_OK)
        # 继续获取下一个字节的数据
        err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将获取的字节数据左移8位，并加到无符号长整型变量上
    x += ((uLong)i)<<8;

    # 如果获取成功
    if (err==ZIP_OK)
        # 继续获取下一个字节的数据
        err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将获取的字节数据左移16位，并加到无符号长整型变量上
    x += ((uLong)i)<<16;

    # 如果获取成功
    if (err==ZIP_OK)
        # 继续获取下一个字节的数据
        err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    # 将 i 左移 24 位，然后加到 x 上
    x += ((uLong)i)<<24;

    # 如果 err 等于 ZIP_OK，则将 x 的值赋给指针 pX 所指向的变量
    if (err==ZIP_OK)
        *pX = x;
    # 否则将 0 赋给指针 pX 所指向的变量
    else
        *pX = 0;
    # 返回 err 变量的值
    return err;
  }
  # 定义一个函数，用于从文件流中获取64位整数
  local int zip64local_getLong64 OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream));

  # 实现获取64位整数的函数
  local int zip64local_getLong64 (const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream, ZPOS64_T *pX)
  {
    ZPOS64_T x;
    int i = 0;
    int err;

    # 从文件流中获取一个字节，存入 i 中
    err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x = (ZPOS64_T)i;

    # 如果获取成功，继续获取下一个字节，并将其左移8位后加到 x 中
    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<8;

    # 重复上述步骤，每次左移16位后加到 x 中
    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<16;

    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<24;

    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<32;

    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<40;

    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<48;

    if (err==ZIP_OK)
      err = zip64local_getByte(pzlib_filefunc_def,filestream,&i);
    x += ((ZPOS64_T)i)<<56;

    # 如果获取成功，将 x 赋值给 pX
    if (err==ZIP_OK)
      *pX = x;
    else
      *pX = 0;

    return err;
  }

  # 如果未定义 BUFREADCOMMENT，则定义为 0x400
  # 定位 ZIP 文件的中央目录（在全局注释之前）
  # 实现定位中央目录的函数
  local ZPOS64_T zip64local_SearchCentralDir OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream));

  local ZPOS64_T zip64local_SearchCentralDir(const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream)
  {
    unsigned char* buf;
    ZPOS64_T uSizeFile;
    ZPOS64_T uBackRead;
    ZPOS64_T uMaxBack=0xffff; # 全局注释的最大大小
    ZPOS64_T uPosFound=0;

    # 将文件流定位到末尾，获取文件大小
    if (ZSEEK64(*pzlib_filefunc_def,filestream,0,ZLIB_FILEFUNC_SEEK_END) != 0)
      return 0;

    uSizeFile = ZTELL64(*pzlib_filefunc_def,filestream);

    # 如果全局注释大小大于文件大小
    # 设置最大回溯值为文件大小
    uMaxBack = uSizeFile;

    # 分配内存用于存储读取的注释内容
    buf = (unsigned char*)ALLOC(BUFREADCOMMENT+4);
    if (buf==NULL)
        return 0;

    # 初始化回读字节数为4
    uBackRead = 4;
    while (uBackRead<uMaxBack)
    {
        uLong uReadSize;
        ZPOS64_T uReadPos ;
        int i;
        # 如果回读字节数加上读取注释的长度大于最大回溯值，则将回读字节数设置为最大回溯值，否则增加BUFREADCOMMENT
        if (uBackRead+BUFREADCOMMENT>uMaxBack)
            uBackRead = uMaxBack;
        else
            uBackRead+=BUFREADCOMMENT;
        # 计算读取位置和读取大小
        uReadPos = uSizeFile-uBackRead ;

        uReadSize = ((BUFREADCOMMENT+4) < (uSizeFile-uReadPos)) ?
            (BUFREADCOMMENT+4) : (uLong)(uSizeFile-uReadPos);
        # 移动文件指针到读取位置
        if (ZSEEK64(*pzlib_filefunc_def,filestream,uReadPos,ZLIB_FILEFUNC_SEEK_SET)!=0)
            break;

        # 读取指定大小的数据到buf中
        if (ZREAD64(*pzlib_filefunc_def,filestream,buf,uReadSize)!=uReadSize)
            break;

        # 在读取的数据中查找注释的结束标志
        for (i=(int)uReadSize-3; (i--)>0;)
            if (((*(buf+i))==0x50) && ((*(buf+i+1))==0x4b) &&
                ((*(buf+i+2))==0x05) && ((*(buf+i+3))==0x06))
            {
                uPosFound = uReadPos+(unsigned)i;
                break;
            }

        # 如果找到注释的结束标志，则跳出循环
        if (uPosFound!=0)
            break;
    }
    # 释放内存
    TRYFREE(buf);
    # 返回找到的注释位置
    return uPosFound;
}
/*
定位 Zip64 中央目录定位器的结束位置，并从那里找到一个 ZIP 文件的中央目录（就在全局注释之前）
*/
local ZPOS64_T zip64local_SearchCentralDir64 OF((const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream));

local ZPOS64_T zip64local_SearchCentralDir64(const zlib_filefunc64_32_def* pzlib_filefunc_def, voidpf filestream)
{
  unsigned char* buf; // 用于存储读取的数据的缓冲区
  ZPOS64_T uSizeFile; // 文件的大小
  ZPOS64_T uBackRead; // 用于向后读取的偏移量
  ZPOS64_T uMaxBack=0xffff; /* 全局注释的最大大小 */
  ZPOS64_T uPosFound=0; // 找到的位置
  uLong uL; // 无符号长整型
  ZPOS64_T relativeOffset; // 相对偏移量

  if (ZSEEK64(*pzlib_filefunc_def,filestream,0,ZLIB_FILEFUNC_SEEK_END) != 0) // 将文件指针移动到文件末尾
    return 0;

  uSizeFile = ZTELL64(*pzlib_filefunc_def,filestream); // 获取文件大小

  if (uMaxBack>uSizeFile) // 如果全局注释的最大大小大于文件大小
    uMaxBack = uSizeFile; // 则将全局注释的最大大小设置为文件大小

  buf = (unsigned char*)ALLOC(BUFREADCOMMENT+4); // 分配缓冲区的内存空间
  if (buf==NULL) // 如果分配失败
    return 0;

  uBackRead = 4; // 初始向后读取的偏移量为4
  while (uBackRead<uMaxBack) // 当向后读取的偏移量小于全局注释的最大大小时
  {
    uLong uReadSize; // 读取的大小
    ZPOS64_T uReadPos; // 读取的位置
    int i;
    if (uBackRead+BUFREADCOMMENT>uMaxBack) // 如果向后读取的偏移量加上读取注释的大小大于全局注释的最大大小
      uBackRead = uMaxBack; // 则将向后读取的偏移量设置为全局注释的最大大小
    else
      uBackRead+=BUFREADCOMMENT; // 否则向后读取的偏移量增加读取注释的大小
    uReadPos = uSizeFile-uBackRead ; // 计算读取的位置

    uReadSize = ((BUFREADCOMMENT+4) < (uSizeFile-uReadPos)) ?
      (BUFREADCOMMENT+4) : (uLong)(uSizeFile-uReadPos); // 计算实际读取的大小
    if (ZSEEK64(*pzlib_filefunc_def,filestream,uReadPos,ZLIB_FILEFUNC_SEEK_SET)!=0) // 将文件指针移动到读取的位置
      break;

    if (ZREAD64(*pzlib_filefunc_def,filestream,buf,uReadSize)!=uReadSize) // 读取数据到缓冲区
      break;

    for (i=(int)uReadSize-3; (i--)>0;) // 遍历读取的数据
    {
      // 签名 "0x07064b50" Zip64 中央目录定位器的结束
      if (((*(buf+i))==0x50) && ((*(buf+i+1))==0x4b) && ((*(buf+i+2))==0x06) && ((*(buf+i+3))==0x07)) // 如果找到了 Zip64 中央目录定位器的结束
      {
        uPosFound = uReadPos+(unsigned)i; // 计算找到的位置
        break;
      }
    }

      if (uPosFound!=0) // 如果找到了位置
        break;
  }

  TRYFREE(buf); // 释放缓冲区的内存空间
  if (uPosFound == 0) // 如果未找到位置
    return 0;

  /* Zip64 中央目录定位器 */
  if (ZSEEK64(*pzlib_filefunc_def,filestream, uPosFound,ZLIB_FILEFUNC_SEEK_SET)!=0) // 将文件指针移动到找到的位置
    # 返回值为0
    return 0;

  # 检查过的签名
  if (zip64local_getLong(pzlib_filefunc_def,filestream,&uL)!=ZIP_OK)
    return 0;

  # 带有zip64结束中央目录的磁盘号
  if (zip64local_getLong(pzlib_filefunc_def,filestream,&uL)!=ZIP_OK)
    return 0;
  if (uL != 0)
    return 0;

  # zip64结束中央目录记录的相对偏移量
  if (zip64local_getLong64(pzlib_filefunc_def,filestream,&relativeOffset)!=ZIP_OK)
    return 0;

  # 总磁盘数
  if (zip64local_getLong(pzlib_filefunc_def,filestream,&uL)!=ZIP_OK)
    return 0;
  if (uL != 1)
    return 0;

  # 转到Zip64结束中央目录记录
  if (ZSEEK64(*pzlib_filefunc_def,filestream, relativeOffset,ZLIB_FILEFUNC_SEEK_SET)!=0)
    return 0;

  # 签名
  if (zip64local_getLong(pzlib_filefunc_def,filestream,&uL)!=ZIP_OK)
    return 0;

  if (uL != 0x06064b50) // 'Zip64结束中央目录'的签名
    return 0;

  return relativeOffset;
}

local int LoadCentralDirectoryRecord(zip64_internal* pziinit)
{
  int err=ZIP_OK;
  ZPOS64_T byte_before_the_zipfile;/* byte before the zipfile, (>0 for sfx)*/

  ZPOS64_T size_central_dir;     /* size of the central directory  */
  ZPOS64_T offset_central_dir;   /* offset of start of central directory */
  ZPOS64_T central_pos;
  uLong uL;

  uLong number_disk;          /* number of the current dist, used for
                              spaning ZIP, unsupported, always 0*/
  uLong number_disk_with_CD;  /* number the the disk with central dir, used
                              for spaning ZIP, unsupported, always 0*/
  ZPOS64_T number_entry;
  ZPOS64_T number_entry_CD;      /* total number of entries in
                                the central dir
                                (same than number_entry on nospan) */
  uLong VersionMadeBy;
  uLong VersionNeeded;
  uLong size_comment;

  int hasZIP64Record = 0;

  // check first if we find a ZIP64 record
  central_pos = zip64local_SearchCentralDir64(&pziinit->z_filefunc,pziinit->filestream);
  if(central_pos > 0)
  {
    hasZIP64Record = 1;
  }
  else if(central_pos == 0)
  {
    central_pos = zip64local_SearchCentralDir(&pziinit->z_filefunc,pziinit->filestream);
  }

/* disable to allow appending to empty ZIP archive
        if (central_pos==0)
            err=ZIP_ERRNO;
*/

  if(hasZIP64Record)
  {
    ZPOS64_T sizeEndOfCentralDirectory;
    if (ZSEEK64(pziinit->z_filefunc, pziinit->filestream, central_pos, ZLIB_FILEFUNC_SEEK_SET) != 0)
      err=ZIP_ERRNO;

    /* the signature, already checked */
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream,&uL)!=ZIP_OK)
      err=ZIP_ERRNO;

    /* size of zip64 end of central directory record */
    if (zip64local_getLong64(&pziinit->z_filefunc, pziinit->filestream, &sizeEndOfCentralDirectory)!=ZIP_OK)
      err=ZIP_ERRNO;

    /* version made by */
    # 读取 ZIP 文件的版本信息，如果失败则设置错误码
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream, &VersionMadeBy)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取 ZIP 文件需要的版本信息，如果失败则设置错误码
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream, &VersionNeeded)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取 ZIP 文件所在磁盘的编号，如果失败则设置错误码
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream,&number_disk)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取包含中央目录起始位置的磁盘编号，如果失败则设置错误码
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream,&number_disk_with_CD)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取中央目录中的条目总数，如果失败则设置错误码
    if (zip64local_getLong64(&pziinit->z_filefunc, pziinit->filestream, &number_entry)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取中央目录中的条目总数，如果失败则设置错误码
    if (zip64local_getLong64(&pziinit->z_filefunc, pziinit->filestream,&number_entry_CD)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 检查中央目录中的条目总数、包含中央目录起始位置的磁盘编号、ZIP 文件所在磁盘的编号是否符合要求，如果不符合则设置错误码
    if ((number_entry_CD!=number_entry) || (number_disk_with_CD!=0) || (number_disk!=0))
      err=ZIP_BADZIPFILE;

    # 读取中央目录的大小，如果失败则设置错误码
    if (zip64local_getLong64(&pziinit->z_filefunc, pziinit->filestream,&size_central_dir)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取中央目录的起始偏移位置，如果失败则设置错误码
    if (zip64local_getLong64(&pziinit->z_filefunc, pziinit->filestream,&offset_central_dir)!=ZIP_OK)
      err=ZIP_ERRNO;

    // TODO..
    // 读取标准中央目录头部的注释内容
    size_comment = 0;
  }
  else
  {
    # 读取中央目录结束信息，如果失败则设置错误码
    if (ZSEEK64(pziinit->z_filefunc, pziinit->filestream, central_pos,ZLIB_FILEFUNC_SEEK_SET)!=0)
      err=ZIP_ERRNO;

    # 读取签名信息，已经在之前检查过，如果失败则设置错误码
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream,&uL)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 读取 ZIP 文件所在磁盘的编号
    # 通过调用zip64local_getShort函数获取文件中的磁盘号，如果返回值不是ZIP_OK，则将err设置为ZIP_ERRNO
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream,&number_disk)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 获取中央目录开始的磁盘号
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream,&number_disk_with_CD)!=ZIP_OK)
      err=ZIP_ERRNO;

    # 获取当前磁盘上中央目录中的条目总数
    number_entry = 0;
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream, &uL)!=ZIP_OK)
      err=ZIP_ERRNO;
    else
      number_entry = uL;

    # 获取中央目录中的总条目数
    number_entry_CD = 0;
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream, &uL)!=ZIP_OK)
      err=ZIP_ERRNO;
    else
      number_entry_CD = uL;

    # 检查中央目录的条目数、磁盘号是否正确
    if ((number_entry_CD!=number_entry) || (number_disk_with_CD!=0) || (number_disk!=0))
      err=ZIP_BADZIPFILE;

    # 获取中央目录的大小
    size_central_dir = 0;
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream, &uL)!=ZIP_OK)
      err=ZIP_ERRNO;
    else
      size_central_dir = uL;

    # 获取中央目录相对于起始磁盘号的偏移量
    offset_central_dir = 0;
    if (zip64local_getLong(&pziinit->z_filefunc, pziinit->filestream, &uL)!=ZIP_OK)
      err=ZIP_ERRNO;
    else
      offset_central_dir = uL;

    # 获取全局注释的长度
    if (zip64local_getShort(&pziinit->z_filefunc, pziinit->filestream, &size_comment)!=ZIP_OK)
      err=ZIP_ERRNO;
  }

  # 检查中央目录的位置是否正确，以及err是否为ZIP_OK
  if ((central_pos<offset_central_dir+size_central_dir) &&
    (err==ZIP_OK))
    err=ZIP_BADZIPFILE;

  # 如果err不等于ZIP_OK，则关闭文件流并返回ZIP_ERRNO
  if (err!=ZIP_OK)
  {
    ZCLOSE64(pziinit->z_filefunc, pziinit->filestream);
    return ZIP_ERRNO;
  }

  # 如果全局注释的长度大于0，则分配内存并读取全局注释
  if (size_comment>0)
  {
    pziinit->globalcomment = (char*)ALLOC(size_comment+1);
    if (pziinit->globalcomment)
    {
      size_comment = ZREAD64(pziinit->z_filefunc, pziinit->filestream, pziinit->globalcomment,size_comment);
      pziinit->globalcomment[size_comment]=0;
  }
}

// 计算 ZIP 文件之前的字节数
byte_before_the_zipfile = central_pos - (offset_central_dir+size_central_dir);
// 设置写入偏移量
pziinit->add_position_when_writing_offset = byte_before_the_zipfile;

{
  // 读取中央目录的大小
  ZPOS64_T size_central_dir_to_read = size_central_dir;
  // 设置缓冲区大小
  size_t buf_size = SIZEDATA_INDATABLOCK;
  // 分配缓冲区内存
  void* buf_read = (void*)ALLOC(buf_size);
  // 移动文件指针到中央目录的偏移位置
  if (ZSEEK64(pziinit->z_filefunc, pziinit->filestream, offset_central_dir + byte_before_the_zipfile, ZLIB_FILEFUNC_SEEK_SET) != 0)
    err=ZIP_ERRNO;

  // 循环读取中央目录数据
  while ((size_central_dir_to_read>0) && (err==ZIP_OK))
  {
    ZPOS64_T read_this = SIZEDATA_INDATABLOCK;
    if (read_this > size_central_dir_to_read)
      read_this = size_central_dir_to_read;

    // 读取数据到缓冲区
    if (ZREAD64(pziinit->z_filefunc, pziinit->filestream,buf_read,(uLong)read_this) != read_this)
      err=ZIP_ERRNO;

    // 将数据添加到中央目录的数据块中
    if (err==ZIP_OK)
      err = add_data_in_datablock(&pziinit->central_dir,buf_read, (uLong)read_this);

    size_central_dir_to_read-=read_this;
  }
  // 释放缓冲区内存
  TRYFREE(buf_read);
}
// 设置起始位置和中央目录的条目数
pziinit->begin_pos = byte_before_the_zipfile;
pziinit->number_entry = number_entry_CD;

// 移动文件指针到中央目录的偏移位置
if (ZSEEK64(pziinit->z_filefunc, pziinit->filestream, offset_central_dir+byte_before_the_zipfile,ZLIB_FILEFUNC_SEEK_SET) != 0)
  err=ZIP_ERRNO;

// 返回错误码
return err;
# 结束宏定义部分
#endif /* !NO_ADDFILEINEXISTINGZIP*/

# 外部函数，用于打开 ZIP 文件
extern zipFile ZEXPORT zipOpen3 (const void *pathname, int append, zipcharpc* globalcomment, zlib_filefunc64_32_def* pzlib_filefunc64_32_def)
{
    # 初始化 ZIP 内部结构
    zip64_internal ziinit;
    zip64_internal* zi;
    int err=ZIP_OK;

    # 设置文件操作函数指针为空
    ziinit.z_filefunc.zseek32_file = NULL;
    ziinit.z_filefunc.ztell32_file = NULL;
    # 如果文件操作函数指针为空，则使用默认的文件操作函数
    if (pzlib_filefunc64_32_def==NULL)
        fill_fopen64_filefunc(&ziinit.z_filefunc.zfile_func64);
    else
        ziinit.z_filefunc = *pzlib_filefunc64_32_def;

    # 打开 ZIP 文件流
    ziinit.filestream = ZOPEN64(ziinit.z_filefunc,
                  pathname,
                  (append == APPEND_STATUS_CREATE) ?
                  (ZLIB_FILEFUNC_MODE_READ | ZLIB_FILEFUNC_MODE_WRITE | ZLIB_FILEFUNC_MODE_CREATE) :
                    (ZLIB_FILEFUNC_MODE_READ | ZLIB_FILEFUNC_MODE_WRITE | ZLIB_FILEFUNC_MODE_EXISTING));

    # 如果文件流为空，则返回空指针
    if (ziinit.filestream == NULL)
        return NULL;

    # 如果是在 ZIP 文件末尾追加，则将文件指针移到末尾
    if (append == APPEND_STATUS_CREATEAFTER)
        ZSEEK64(ziinit.z_filefunc,ziinit.filestream,0,SEEK_END);

    # 初始化 ZIP 内部结构的一些属性
    ziinit.begin_pos = ZTELL64(ziinit.z_filefunc,ziinit.filestream);
    ziinit.in_opened_file_inzip = 0;
    ziinit.ci.stream_initialised = 0;
    ziinit.number_entry = 0;
    ziinit.add_position_when_writing_offset = 0;
    init_linkedlist(&(ziinit.central_dir));

    # 分配内存给 ZIP 内部结构
    zi = (zip64_internal*)ALLOC(sizeof(zip64_internal));
    # 如果分配内存失败，则关闭文件流并返回空指针
    if (zi==NULL)
    {
        ZCLOSE64(ziinit.z_filefunc,ziinit.filestream);
        return NULL;
    }

    /* 现在我们在 ZIP 文件中添加文件 */
#    ifndef NO_ADDFILEINEXISTINGZIP
    ziinit.globalcomment = NULL;
    if (append == APPEND_STATUS_ADDINZIP)
    {
      // 读取并缓存中央目录记录
      err = LoadCentralDirectoryRecord(&ziinit);
    }

    if (globalcomment)
    {
      *globalcomment = ziinit.globalcomment;
    }
#    endif /* !NO_ADDFILEINEXISTINGZIP*/

    # 如果出现错误，则关闭文件流并返回空指针
    if (err != ZIP_OK)
    {
# 如果没有定义 NO_ADDFILEINEXISTINGZIP，则释放全局注释内存
    TRYFREE(ziinit.globalcomment);
# 结束条件判断，释放 ZIP 对象内存
    TRYFREE(zi);
    return NULL;
}
else
{
    # 将初始化好的 ZIP 对象赋值给传入的指针
    *zi = ziinit;
    return (zipFile)zi;
}
}

# 根据给定路径和参数打开 ZIP 文件
extern zipFile ZEXPORT zipOpen2 (const char *pathname, int append, zipcharpc* globalcomment, zlib_filefunc_def* pzlib_filefunc32_def)
{
    # 如果传入的文件操作函数指针不为空
    if (pzlib_filefunc32_def != NULL)
    {
        # 填充 64 位文件操作函数结构体
        zlib_filefunc64_32_def zlib_filefunc64_32_def_fill;
        fill_zlib_filefunc64_32_def_from_filefunc32(&zlib_filefunc64_32_def_fill,pzlib_filefunc32_def);
        # 调用 zipOpen3 函数，传入路径、参数、全局注释和文件操作函数结构体
        return zipOpen3(pathname, append, globalcomment, &zlib_filefunc64_32_def_fill);
    }
    else
        # 调用 zipOpen3 函数，传入路径、参数、全局注释和空指针
        return zipOpen3(pathname, append, globalcomment, NULL);
}

# 根据给定路径和参数打开 64 位 ZIP 文件
extern zipFile ZEXPORT zipOpen2_64 (const void *pathname, int append, zipcharpc* globalcomment, zlib_filefunc64_def* pzlib_filefunc_def)
{
    # 如果传入的 64 位文件操作函数指针不为空
    if (pzlib_filefunc_def != NULL)
    {
        # 填充 64 位文件操作函数结构体
        zlib_filefunc64_32_def zlib_filefunc64_32_def_fill;
        zlib_filefunc64_32_def_fill.zfile_func64 = *pzlib_filefunc_def;
        zlib_filefunc64_32_def_fill.ztell32_file = NULL;
        zlib_filefunc64_32_def_fill.zseek32_file = NULL;
        # 调用 zipOpen3 函数，传入路径、参数、全局注释和文件操作函数结构体
        return zipOpen3(pathname, append, globalcomment, &zlib_filefunc64_32_def_fill);
    }
    else
        # 调用 zipOpen3 函数，传入路径、参数、全局注释和空指针
        return zipOpen3(pathname, append, globalcomment, NULL);
}

# 根据给定路径和参数打开 ZIP 文件
extern zipFile ZEXPORT zipOpen (const char* pathname, int append)
{
    # 调用 zipOpen3 函数，传入路径、参数、空指针和空指针
    return zipOpen3((const void*)pathname,append,NULL,NULL);
}

# 根据给定路径和参数打开 64 位 ZIP 文件
extern zipFile ZEXPORT zipOpen64 (const void* pathname, int append)
{
    # 调用 zipOpen3 函数，传入路径、参数、空指针和空指针
    return zipOpen3(pathname,append,NULL,NULL);
}

# 写入本地文件头部信息
local int Write_LocalFileHeader(zip64_internal* zi, const char* filename, uInt size_extrafield_local, const void* extrafield_local)
{
  # 写入本地头部信息
  int err;
  uInt size_filename = (uInt)strlen(filename);
  uInt size_extrafield = size_extrafield_local;
  # 将本地头部信息写入文件流
  err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)LOCALHEADERMAGIC, 4);

  if (err==ZIP_OK)
  {
  // 如果 ZIP 文件使用了 ZIP64 格式
  if(zi->ci.zip64)
    // 将版本号写入文件流，需要解压的版本号为45
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)45,2);/* version needed to extract */
  else
    // 将版本号写入文件流，需要解压的版本号为20
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)20,2);/* version needed to extract */
  }

  // 如果没有错误，将标志位写入文件流
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)zi->ci.flag,2);

  // 如果没有错误，将压缩方法写入文件流
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)zi->ci.method,2);

  // 如果没有错误，将 DOS 日期写入文件流
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)zi->ci.dosDate,4);

  // CRC / 压缩大小 / 未压缩大小将在后面填充并重新写入
  // 如果没有错误，将 CRC 写入文件流，初始值为0
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4); /* crc 32, unknown */
  // 如果没有错误
  if (err==ZIP_OK)
  {
    // 如果使用了 ZIP64 格式，将压缩大小写入文件流，初始值为0xFFFFFFFF
    if(zi->ci.zip64)
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0xFFFFFFFF,4); /* compressed size, unknown */
    else
      // 如果没有使用 ZIP64 格式，将压缩大小写入文件流，初始值为0
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4); /* compressed size, unknown */
  }
  // 如果没有错误
  if (err==ZIP_OK)
  {
    // 如果使用了 ZIP64 格式，将未压缩大小写入文件流，初始值为0xFFFFFFFF
    if(zi->ci.zip64)
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0xFFFFFFFF,4); /* uncompressed size, unknown */
    else
      // 如果没有使用 ZIP64 格式，将未压缩大小写入文件流，初始值为0
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4); /* uncompressed size, unknown */
  }

  // 如果没有错误，将文件名长度写入文件流
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)size_filename,2);

  // 如果使用了 ZIP64 格式，增加额外字段长度
  if(zi->ci.zip64)
  {
    size_extrafield += 20;
  }

  // 如果没有错误，将额外字段长度写入文件流
  if (err==ZIP_OK)
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)size_extrafield,2);

  // 如果没有错误，并且文件名长度大于0
  if ((err==ZIP_OK) && (size_filename > 0))
  {
    // 如果写入文件名长度不等于文件名长度，返回错误
    if (ZWRITE64(zi->z_filefunc,zi->filestream,filename,size_filename)!=size_filename)
      err = ZIP_ERRNO;
  }

  // 如果没有错误，并且额外字段长度大于0
  if ((err==ZIP_OK) && (size_extrafield_local > 0))
  {
    # 如果写入 ZIP64 扩展信息时出错，则将 err 设置为 ZIP_ERRNO
    if (ZWRITE64(zi->z_filefunc, zi->filestream, extrafield_local, size_extrafield_local) != size_extrafield_local)
      err = ZIP_ERRNO;
  }


  # 如果没有错误并且启用了 ZIP64
  if ((err==ZIP_OK) && (zi->ci.zip64))
  {
      # 写入 Zip64 扩展信息
      short HeaderID = 1;
      short DataSize = 16;
      ZPOS64_T CompressedSize = 0;
      ZPOS64_T UncompressedSize = 0;

      # 记住本地文件头的 Zip64 扩展信息位置（在文件处理完后更新大小时需要）
      zi->ci.pos_zip64extrainfo = ZTELL64(zi->z_filefunc,zi->filestream);

      err = zip64local_putValue(&zi->z_filefunc, zi->filestream, (ZPOS64_T)HeaderID,2);
      err = zip64local_putValue(&zi->z_filefunc, zi->filestream, (ZPOS64_T)DataSize,2);

      err = zip64local_putValue(&zi->z_filefunc, zi->filestream, (ZPOS64_T)UncompressedSize,8);
      err = zip64local_putValue(&zi->z_filefunc, zi->filestream, (ZPOS64_T)CompressedSize,8);
  }

  # 返回错误码
  return err;
}
/*
   NOTE.
   当以原始方式写入 ZIP64 扩展信息时，需要在调用此函数之前剥离 extrafield_local 和 extrafield_global 中的 ZIP64 扩展信息块
   这可以通过 zipRemoveExtraInfoBlock 来完成

   这里没有这样做是因为我们需要重新分配一个新的缓冲区，因为参数是 'const'，我想尽量减少不必要的分配。
*/
extern int ZEXPORT zipOpenNewFileInZip4_64 (zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                         const void* extrafield_local, uInt size_extrafield_local,
                                         const void* extrafield_global, uInt size_extrafield_global,
                                         const char* comment, int method, int level, int raw,
                                         int windowBits,int memLevel, int strategy,
                                         const char* password, uLong crcForCrypting,
                                         uLong versionMadeBy, uLong flagBase, int zip64)
{
    zip64_internal* zi;
    uInt size_filename;
    uInt size_comment;
    uInt i;
    int err = ZIP_OK;

#    ifdef NOCRYPT
    (crcForCrypting);
    if (password != NULL)
        return ZIP_PARAMERROR;
#    endif

    if (file == NULL)
        return ZIP_PARAMERROR;

#ifdef HAVE_BZIP2
    if ((method!=0) && (method!=Z_DEFLATED) && (method!=Z_BZIP2ED))
      return ZIP_PARAMERROR;
#else
    if ((method!=0) && (method!=Z_DEFLATED))
      return ZIP_PARAMERROR;
#endif

    zi = (zip64_internal*)file;

    if (zi->in_opened_file_inzip == 1)
    {
        err = zipCloseFileInZip (file);
        if (err != ZIP_OK)
            return err;
    }

    if (filename==NULL)
        filename="-";

    if (comment==NULL)
        size_comment = 0;
    else
        size_comment = (uInt)strlen(comment);

    size_filename = (uInt)strlen(filename);

    if (zipfi == NULL)
        zi->ci.dosDate = 0;
    else
    {
        // 如果 ZIP 文件的 dosDate 不为 0，则将 zi->ci.dosDate 设置为 zipfi->dosDate，否则将其转换为 dosDate
        if (zipfi->dosDate != 0)
            zi->ci.dosDate = zipfi->dosDate;
        else
          zi->ci.dosDate = zip64local_TmzDateToDosDate(&zipfi->tmz_date);
    }

    // 设置标志位
    zi->ci.flag = flagBase;
    // 如果压缩级别为 8 或 9，则设置标志位的第 2 位
    if ((level==8) || (level==9))
      zi->ci.flag |= 2;
    // 如果压缩级别为 2，则设置标志位的第 3 位
    if (level==2)
      zi->ci.flag |= 4;
    // 如果压缩级别为 1，则设置标志位的第 2 和第 3 位
    if (level==1)
      zi->ci.flag |= 6;
    // 如果有密码，则设置标志位的第 1 位
    if (password != NULL)
      zi->ci.flag |= 1;

    // 初始化一些字段
    zi->ci.crc32 = 0;
    zi->ci.method = method;
    zi->ci.encrypt = 0;
    zi->ci.stream_initialised = 0;
    zi->ci.pos_in_buffered_data = 0;
    zi->ci.raw = raw;
    zi->ci.pos_local_header = ZTELL64(zi->z_filefunc,zi->filestream);

    // 计算中央目录头的大小
    zi->ci.size_centralheader = SIZECENTRALHEADER + size_filename + size_extrafield_global + size_comment;
    // 预留的 ZIP64 额外信息数据的空间大小
    zi->ci.size_centralExtraFree = 32;

    // 分配中央目录头的内存空间
    zi->ci.central_header = (char*)ALLOC((uInt)zi->ci.size_centralheader + zi->ci.size_centralExtraFree);

    // 设置中央目录头的一些信息
    zi->ci.size_centralExtra = size_extrafield_global;
    zip64local_putValue_inmemory(zi->ci.central_header,(uLong)CENTRALHEADERMAGIC,4);
    zip64local_putValue_inmemory(zi->ci.central_header+4,(uLong)versionMadeBy,2);
    zip64local_putValue_inmemory(zi->ci.central_header+6,(uLong)20,2);
    zip64local_putValue_inmemory(zi->ci.central_header+8,(uLong)zi->ci.flag,2);
    zip64local_putValue_inmemory(zi->ci.central_header+10,(uLong)zi->ci.method,2);
    zip64local_putValue_inmemory(zi->ci.central_header+12,(uLong)zi->ci.dosDate,4);
    zip64local_putValue_inmemory(zi->ci.central_header+16,(uLong)0,4); /*crc*/
    zip64local_putValue_inmemory(zi->ci.central_header+20,(uLong)0,4); /*compr size*/
    zip64local_putValue_inmemory(zi->ci.central_header+24,(uLong)0,4); /*uncompr size*/
    zip64local_putValue_inmemory(zi->ci.central_header+28,(uLong)size_filename,2);
    zip64local_putValue_inmemory(zi->ci.central_header+30,(uLong)size_extrafield_global,2);
    # 在中央目录头部的偏移量32处写入注释的大小，使用zip64local_putValue_inmemory函数
    zip64local_putValue_inmemory(zi->ci.central_header+32,(uLong)size_comment,2);
    # 在中央目录头部的偏移量34处写入磁盘号的起始位置，使用zip64local_putValue_inmemory函数
    zip64local_putValue_inmemory(zi->ci.central_header+34,(uLong)0,2); /*disk nm start*/

    # 如果zipfi为空，则在中央目录头部的偏移量36处写入0，否则写入zipfi的内部文件属性
    if (zipfi==NULL)
        zip64local_putValue_inmemory(zi->ci.central_header+36,(uLong)0,2);
    else
        zip64local_putValue_inmemory(zi->ci.central_header+36,(uLong)zipfi->internal_fa,2);

    # 如果zipfi为空，则在中央目录头部的偏移量38处写入0，否则写入zipfi的外部文件属性
    if (zipfi==NULL)
        zip64local_putValue_inmemory(zi->ci.central_header+38,(uLong)0,4);
    else
        zip64local_putValue_inmemory(zi->ci.central_header+38,(uLong)zipfi->external_fa,4);

    # 如果中央目录头部的偏移量42大于等于0xffffffff，则写入0xffffffff，否则写入zi->ci.pos_local_header - zi->add_position_when_writing_offset
    if(zi->ci.pos_local_header >= 0xffffffff)
      zip64local_putValue_inmemory(zi->ci.central_header+42,(uLong)0xffffffff,4);
    else
      zip64local_putValue_inmemory(zi->ci.central_header+42,(uLong)zi->ci.pos_local_header - zi->add_position_when_writing_offset,4);

    # 将文件名写入中央目录头部
    for (i=0;i<size_filename;i++)
        *(zi->ci.central_header+SIZECENTRALHEADER+i) = *(filename+i);

    # 将全局额外字段写入中央目录头部
    for (i=0;i<size_extrafield_global;i++)
        *(zi->ci.central_header+SIZECENTRALHEADER+size_filename+i) =
              *(((const char*)extrafield_global)+i);

    # 将注释写入中央目录头部
    for (i=0;i<size_comment;i++)
        *(zi->ci.central_header+SIZECENTRALHEADER+size_filename+
              size_extrafield_global+i) = *(comment+i);
    # 如果中央目录头部为空，则返回ZIP_INTERNALERROR
    if (zi->ci.central_header == NULL)
        return ZIP_INTERNALERROR;

    # 设置zip64标志为zip64
    zi->ci.zip64 = zip64;
    # 设置压缩数据总量为0
    zi->ci.totalCompressedData = 0;
    # 设置未压缩数据总量为0
    zi->ci.totalUncompressedData = 0;
    # 设置zip64扩展信息的位置为0
    zi->ci.pos_zip64extrainfo = 0;

    # 调用Write_LocalFileHeader函数，写入本地文件头部
    err = Write_LocalFileHeader(zi, filename, size_extrafield_local, extrafield_local);
#ifdef HAVE_BZIP2
    # 如果支持 BZIP2，则设置压缩流的输入可用字节数为0
    zi->ci.bstream.avail_in = (uInt)0;
    # 设置压缩流的输出可用字节数为 Z_BUFSIZE
    zi->ci.bstream.avail_out = (uInt)Z_BUFSIZE;
    # 设置压缩流的输出缓冲区
    zi->ci.bstream.next_out = (char*)zi->ci.buffered_data;
    # 初始化压缩流的输入字节数高32位和低32位
    zi->ci.bstream.total_in_hi32 = 0;
    zi->ci.bstream.total_in_lo32 = 0;
    # 初始化压缩流的输出字节数高32位和低32位
    zi->ci.bstream.total_out_hi32 = 0;
    zi->ci.bstream.total_out_lo32 = 0;
#endif

# 设置压缩流的输入可用字节数为0
zi->ci.stream.avail_in = (uInt)0;
# 设置压缩流的输出可用字节数为 Z_BUFSIZE
zi->ci.stream.avail_out = (uInt)Z_BUFSIZE;
# 设置压缩流的输出缓冲区
zi->ci.stream.next_out = zi->ci.buffered_data;
# 初始化压缩流的输入字节数
zi->ci.stream.total_in = 0;
# 初始化压缩流的输出字节数
zi->ci.stream.total_out = 0;
# 设置压缩流的数据类型为二进制
zi->ci.stream.data_type = Z_BINARY;

#ifdef HAVE_BZIP2
    # 如果支持 BZIP2 并且压缩方法为 Z_DEFLATED 或 Z_BZIP2ED 并且不是原始数据
    if ((err==ZIP_OK) && (zi->ci.method == Z_DEFLATED || zi->ci.method == Z_BZIP2ED) && (!zi->ci.raw))
#else
    # 如果不支持 BZIP2 并且压缩方法为 Z_DEFLATED 并且不是原始数据
    if ((err==ZIP_OK) && (zi->ci.method == Z_DEFLATED) && (!zi->ci.raw))
#endif
    {
        # 如果压缩方法为 Z_DEFLATED
        if(zi->ci.method == Z_DEFLATED)
        {
          # 设置压缩流的内存分配函数和释放函数为0
          zi->ci.stream.zalloc = (alloc_func)0;
          zi->ci.stream.zfree = (free_func)0;
          zi->ci.stream.opaque = (voidpf)0;

          # 如果窗口位数大于0，则将其设置为负值
          if (windowBits>0)
              windowBits = -windowBits;

          # 初始化压缩流为 Z_DEFLATED 类型
          err = deflateInit2(&zi->ci.stream, level, Z_DEFLATED, windowBits, memLevel, strategy);

          # 如果初始化成功，则设置压缩流初始化状态为 Z_DEFLATED
          if (err==Z_OK)
              zi->ci.stream_initialised = Z_DEFLATED;
        }
        # 如果压缩方法为 Z_BZIP2ED
        else if(zi->ci.method == Z_BZIP2ED)
        {
#ifdef HAVE_BZIP2
            // 在这里初始化 BZip 相关内容
          # 设置 BZip 压缩流的内存分配函数和释放函数为0
          zi->ci.bstream.bzalloc = 0;
          zi->ci.bstream.bzfree = 0;
          zi->ci.bstream.opaque = (voidpf)0;

          # 初始化 BZip 压缩流
          err = BZ2_bzCompressInit(&zi->ci.bstream, level, 0,35);
          # 如果初始化成功，则设置压缩流初始化状态为 Z_BZIP2ED
          if(err == BZ_OK)
            zi->ci.stream_initialised = Z_BZIP2ED;
#endif
        }

    }

#    ifndef NOCRYPT
    # 设置加密头部大小为0
    zi->ci.crypt_header_size = 0;
    # 如果初始化成功并且密码不为空
    if ((err==Z_OK) && (password != NULL))
    {
        // 定义一个长度为RAND_HEAD_LEN的无符号字符数组bufHead
        unsigned char bufHead[RAND_HEAD_LEN];
        // 定义一个无符号整数sizeHead
        unsigned int sizeHead;
        // 设置压缩文件的加密标志为1
        zi->ci.encrypt = 1;
        // 获取CRC表并赋值给zi->ci.pcrc_32_tab
        zi->ci.pcrc_32_tab = get_crc_table();
        // 调用init_keys函数，初始化加密密钥
        /*init_keys(password,zi->ci.keys,zi->ci.pcrc_32_tab);*/
        
        // 调用crypthead函数，对密码和bufHead进行加密，返回加密后的数据长度
        sizeHead=crypthead(password,bufHead,RAND_HEAD_LEN,zi->ci.keys,zi->ci.pcrc_32_tab,crcForCrypting);
        // 将加密后的头部数据长度赋值给zi->ci.crypt_header_size
        zi->ci.crypt_header_size = sizeHead;
        
        // 如果向文件流写入加密后的头部数据长度不等于sizeHead，则将错误码赋值给err
        if (ZWRITE64(zi->z_filefunc,zi->filestream,bufHead,sizeHead) != sizeHead)
            err = ZIP_ERRNO;
    }
# 如果条件成立，表示没有错误，设置标志位表明已经打开了 ZIP 文件中的文件
    if (err==Z_OK)
        zi->in_opened_file_inzip = 1;
    # 返回错误码
    return err;
}

# 打开一个新文件，使用 zipOpenNewFileInZip4_64 函数，传入参数并设置额外的标志位为 0
extern int ZEXPORT zipOpenNewFileInZip4 (zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                         const void* extrafield_local, uInt size_extrafield_local,
                                         const void* extrafield_global, uInt size_extrafield_global,
                                         const char* comment, int method, int level, int raw,
                                         int windowBits,int memLevel, int strategy,
                                         const char* password, uLong crcForCrypting,
                                         uLong versionMadeBy, uLong flagBase)
{
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, raw,
                                 windowBits, memLevel, strategy,
                                 password, crcForCrypting, versionMadeBy, flagBase, 0);
}

# 打开一个新文件，传入参数并设置额外的标志位为 0
extern int ZEXPORT zipOpenNewFileInZip3 (zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                         const void* extrafield_local, uInt size_extrafield_local,
                                         const void* extrafield_global, uInt size_extrafield_global,
                                         const char* comment, int method, int level, int raw,
                                         int windowBits,int memLevel, int strategy,
                                         const char* password, uLong crcForCrypting)
{
    # 调用函数 zipOpenNewFileInZip4_64，创建一个新的文件并将其添加到 ZIP 压缩包中
    # 参数包括文件对象、文件名、压缩包信息、本地额外字段、本地额外字段大小、全局额外字段、全局额外字段大小、注释、压缩方法、压缩级别、原始数据、窗口大小、内存级别、压缩策略、密码、用于加密的 CRC、创建文件的版本信息、以及一些额外的参数
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, raw,
                                 windowBits, memLevel, strategy,
                                 password, crcForCrypting, VERSIONMADEBY, 0, 0);
# 打开一个新文件，准备将数据写入到 ZIP 文件中
extern int ZEXPORT zipOpenNewFileInZip3_64(zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                         const void* extrafield_local, uInt size_extrafield_local,
                                         const void* extrafield_global, uInt size_extrafield_global,
                                         const char* comment, int method, int level, int raw,
                                         int windowBits,int memLevel, int strategy,
                                         const char* password, uLong crcForCrypting, int zip64)
{
    # 调用 zipOpenNewFileInZip4_64 函数，准备将数据写入到 ZIP 文件中
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, raw,
                                 windowBits, memLevel, strategy,
                                 password, crcForCrypting, VERSIONMADEBY, 0, zip64);
}

# 打开一个新文件，准备将数据写入到 ZIP 文件中
extern int ZEXPORT zipOpenNewFileInZip2(zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                        const void* extrafield_local, uInt size_extrafield_local,
                                        const void* extrafield_global, uInt size_extrafield_global,
                                        const char* comment, int method, int level, int raw)
{
    # 调用 zipOpenNewFileInZip4_64 函数，准备将数据写入到 ZIP 文件中
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, raw,
                                 -MAX_WBITS, DEF_MEM_LEVEL, Z_DEFAULT_STRATEGY,
                                 NULL, 0, VERSIONMADEBY, 0, 0);
}
# 在 ZIP 文件中打开一个新文件，支持 64 位版本
extern int ZEXPORT zipOpenNewFileInZip2_64(zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                        const void* extrafield_local, uInt size_extrafield_local,
                                        const void* extrafield_global, uInt size_extrafield_global,
                                        const char* comment, int method, int level, int raw, int zip64)
{
    # 调用 zipOpenNewFileInZip4_64 函数，传入相应参数，并返回结果
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, raw,
                                 -MAX_WBITS, DEF_MEM_LEVEL, Z_DEFAULT_STRATEGY,
                                 NULL, 0, VERSIONMADEBY, 0, zip64);
}

# 在 ZIP 文件中打开一个新文件，支持 64 位版本
extern int ZEXPORT zipOpenNewFileInZip64 (zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                        const void* extrafield_local, uInt size_extrafield_local,
                                        const void*extrafield_global, uInt size_extrafield_global,
                                        const char* comment, int method, int level, int zip64)
{
    # 调用 zipOpenNewFileInZip4_64 函数，传入相应参数，并返回结果
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, 0,
                                 -MAX_WBITS, DEF_MEM_LEVEL, Z_DEFAULT_STRATEGY,
                                 NULL, 0, VERSIONMADEBY, 0, zip64);
}
# 在 ZIP 文件中打开一个新文件，准备写入数据
extern int ZEXPORT zipOpenNewFileInZip (zipFile file, const char* filename, const zip_fileinfo* zipfi,
                                        const void* extrafield_local, uInt size_extrafield_local,
                                        const void*extrafield_global, uInt size_extrafield_global,
                                        const char* comment, int method, int level)
{
    # 调用 zipOpenNewFileInZip4_64 函数，传入参数并返回结果
    return zipOpenNewFileInZip4_64 (file, filename, zipfi,
                                 extrafield_local, size_extrafield_local,
                                 extrafield_global, size_extrafield_global,
                                 comment, method, level, 0,
                                 -MAX_WBITS, DEF_MEM_LEVEL, Z_DEFAULT_STRATEGY,
                                 NULL, 0, VERSIONMADEBY, 0, 0);
}

# 刷新写入缓冲区，将数据写入 ZIP 文件
local int zip64FlushWriteBuffer(zip64_internal* zi)
{
    # 初始化错误码
    int err=ZIP_OK;

    # 如果文件加密标志不为0
    if (zi->ci.encrypt != 0)
    {
        # 如果未定义 NOCRYPT
        # 对缓冲区中的数据进行加密处理
        uInt i;
        int t;
        for (i=0;i<zi->ci.pos_in_buffered_data;i++)
            zi->ci.buffered_data[i] = zencode(zi->ci.keys, zi->ci.pcrc_32_tab, zi->ci.buffered_data[i],t);
    }

    # 将缓冲区中的数据写入文件流
    if (ZWRITE64(zi->z_filefunc,zi->filestream,zi->ci.buffered_data,zi->ci.pos_in_buffered_data) != zi->ci.pos_in_buffered_data)
      err = ZIP_ERRNO;

    # 更新压缩后数据的总大小
    zi->ci.totalCompressedData += zi->ci.pos_in_buffered_data;

    # 如果使用了 BZIP2 压缩方法
    # 更新未压缩数据的总大小
    # 重置 BZIP2 压缩流的输入大小
    # 重置 BZIP2 压缩流的输入大小
    # 否则
    # 更新未压缩数据的总大小
    # 重置压缩流的输入大小
    # 重置压缩流的输入大小
    # 重置缓冲区中的数据位置
    zi->ci.pos_in_buffered_data = 0;

    # 返回错误码
    return err;
}

# 在 ZIP 文件中写入数据
extern int ZEXPORT zipWriteInFileInZip (zipFile file,const void* buf,unsigned int len)
{
    # 定义 zip64_internal 指针
    zip64_internal* zi;
    # 初始化错误码
    int err=ZIP_OK;

    # 如果文件指针为空
    # 返回参数错误
    if (file == NULL)
        return ZIP_PARAMERROR;
    # 将文件指针转换为 zip64_internal 指针
    zi = (zip64_internal*)file;
    # 如果当前 ZIP 文件中没有打开的文件，则返回参数错误
    if (zi->in_opened_file_inzip == 0)
        return ZIP_PARAMERROR;

    # 更新当前文件的 CRC32 校验值
    zi->ci.crc32 = crc32(zi->ci.crc32,buf,(uInt)len);
#ifdef HAVE_BZIP2
    // 如果支持 BZIP2 压缩，并且压缩方法不是原始数据
    if(zi->ci.method == Z_BZIP2ED && (!zi->ci.raw))
    {
      // 设置 BZIP2 流的输入缓冲区和长度
      zi->ci.bstream.next_in = (void*)buf;
      zi->ci.bstream.avail_in = len;
      // 初始化 BZIP2 压缩状态
      err = BZ_RUN_OK;

      // 循环处理输入数据，直到压缩完成或出现错误
      while ((err==BZ_RUN_OK) && (zi->ci.bstream.avail_in>0))
      {
        // 如果输出缓冲区已满，将缓冲区中的数据写入到文件
        if (zi->ci.bstream.avail_out == 0)
        {
          if (zip64FlushWriteBuffer(zi) == ZIP_ERRNO)
            err = ZIP_ERRNO;
          // 重置输出缓冲区和指针
          zi->ci.bstream.avail_out = (uInt)Z_BUFSIZE;
          zi->ci.bstream.next_out = (char*)zi->ci.buffered_data;
        }

        // 如果出现错误，跳出循环
        if(err != BZ_RUN_OK)
          break;

        // 如果压缩方法是 BZIP2，并且不是原始数据
        if ((zi->ci.method == Z_BZIP2ED) && (!zi->ci.raw))
        {
          // 记录压缩前的输出总字节数
          uLong uTotalOutBefore_lo = zi->ci.bstream.total_out_lo32;
          // 压缩数据
          err=BZ2_bzCompress(&zi->ci.bstream,  BZ_RUN);
          // 更新已压缩的数据长度
          zi->ci.pos_in_buffered_data += (uInt)(zi->ci.bstream.total_out_lo32 - uTotalOutBefore_lo) ;
        }
      }

      // 如果压缩完成，设置压缩状态为 ZIP_OK
      if(err == BZ_RUN_OK)
        err = ZIP_OK;
    }
    else
#endif
    {
      # 设置输入缓冲区为 buf，长度为 len
      zi->ci.stream.next_in = (Bytef*)buf;
      zi->ci.stream.avail_in = len;

      # 当输入缓冲区还有数据时，执行循环
      while ((err==ZIP_OK) && (zi->ci.stream.avail_in>0))
      {
          # 如果输出缓冲区已满，执行刷新写入缓冲区操作
          if (zi->ci.stream.avail_out == 0)
          {
              if (zip64FlushWriteBuffer(zi) == ZIP_ERRNO)
                  err = ZIP_ERRNO;
              zi->ci.stream.avail_out = (uInt)Z_BUFSIZE;
              zi->ci.stream.next_out = zi->ci.buffered_data;
          }

          # 如果 err 不等于 ZIP_OK，跳出循环
          if(err != ZIP_OK)
              break;

          # 如果压缩方法为 Z_DEFLATED 并且不是原始数据
          if ((zi->ci.method == Z_DEFLATED) && (!zi->ci.raw))
          {
              # 记录压缩前的输出总字节数
              uLong uTotalOutBefore = zi->ci.stream.total_out;
              # 执行压缩操作
              err=deflate(&zi->ci.stream,  Z_NO_FLUSH);
              # 更新缓冲区中的位置
              zi->ci.pos_in_buffered_data += (uInt)(zi->ci.stream.total_out - uTotalOutBefore) ;
          }
          else
          {
              # 复制数据到输出缓冲区
              uInt copy_this,i;
              if (zi->ci.stream.avail_in < zi->ci.stream.avail_out)
                  copy_this = zi->ci.stream.avail_in;
              else
                  copy_this = zi->ci.stream.avail_out;

              for (i = 0; i < copy_this; i++)
                  *(((char*)zi->ci.stream.next_out)+i) =
                      *(((const char*)zi->ci.stream.next_in)+i);
              {
                  # 更新输入输出缓冲区的位置和总字节数
                  zi->ci.stream.avail_in -= copy_this;
                  zi->ci.stream.avail_out-= copy_this;
                  zi->ci.stream.next_in+= copy_this;
                  zi->ci.stream.next_out+= copy_this;
                  zi->ci.stream.total_in+= copy_this;
                  zi->ci.stream.total_out+= copy_this;
                  zi->ci.pos_in_buffered_data += copy_this;
              }
          }
      }// while(...)
    }

    # 返回操作结果
    return err;
# 关闭当前正在写入的文件并完成压缩
extern int ZEXPORT zipCloseFileInZipRaw (zipFile file, uLong uncompressed_size, uLong crc32)
{
    # 调用 zipCloseFileInZipRaw64 函数关闭当前正在写入的文件并完成压缩
    return zipCloseFileInZipRaw64 (file, uncompressed_size, crc32);
}

# 关闭当前正在写入的文件并完成压缩（64位版本）
extern int ZEXPORT zipCloseFileInZipRaw64 (zipFile file, ZPOS64_T uncompressed_size, uLong crc32)
{
    # 定义变量
    zip64_internal* zi;
    ZPOS64_T compressed_size;
    uLong invalidValue = 0xffffffff;
    unsigned datasize = 0;
    int err=ZIP_OK;

    # 如果文件为空，则返回参数错误
    if (file == NULL)
        return ZIP_PARAMERROR;
    # 将文件转换为 zip64_internal 类型
    zi = (zip64_internal*)file;

    # 如果当前没有打开的文件在 zip 中，则返回参数错误
    if (zi->in_opened_file_inzip == 0)
        return ZIP_PARAMERROR;
    # 将压缩流中的输入数据大小设置为 0
    zi->ci.stream.avail_in = 0;

    # 如果压缩方法为 Z_DEFLATED 并且不是原始数据
    if ((zi->ci.method == Z_DEFLATED) && (!zi->ci.raw))
                {
                        # 循环执行压缩操作
                        while (err==ZIP_OK)
                        {
                                uLong uTotalOutBefore;
                                # 如果输出缓冲区已满
                                if (zi->ci.stream.avail_out == 0)
                                {
                                        # 刷新写入缓冲区
                                        if (zip64FlushWriteBuffer(zi) == ZIP_ERRNO)
                                                err = ZIP_ERRNO;
                                        # 重置输出缓冲区大小和位置
                                        zi->ci.stream.avail_out = (uInt)Z_BUFSIZE;
                                        zi->ci.stream.next_out = zi->ci.buffered_data;
                                }
                                # 记录压缩前的输出总量
                                uTotalOutBefore = zi->ci.stream.total_out;
                                # 执行压缩操作
                                err=deflate(&zi->ci.stream,  Z_FINISH);
                                # 更新缓冲区中的位置
                                zi->ci.pos_in_buffered_data += (uInt)(zi->ci.stream.total_out - uTotalOutBefore) ;
                        }
                }
    # 如果压缩方法为 Z_BZIP2ED 并且不是原始数据
    else if ((zi->ci.method == Z_BZIP2ED) && (!zi->ci.raw))
    {
#ifdef HAVE_BZIP2
      // 如果支持 BZIP2 压缩
      err = BZ_FINISH_OK;
      // 循环执行 BZIP2 压缩，直到压缩结束
      while (err==BZ_FINISH_OK)
      {
        uLong uTotalOutBefore;
        // 如果输出缓冲区为空
        if (zi->ci.bstream.avail_out == 0)
        {
          // 刷新写缓冲区
          if (zip64FlushWriteBuffer(zi) == ZIP_ERRNO)
            err = ZIP_ERRNO;
          // 重置输出缓冲区
          zi->ci.bstream.avail_out = (uInt)Z_BUFSIZE;
          zi->ci.bstream.next_out = (char*)zi->ci.buffered_data;
        }
        // 记录压缩前的输出总量
        uTotalOutBefore = zi->ci.bstream.total_out_lo32;
        // 执行 BZIP2 压缩
        err=BZ2_bzCompress(&zi->ci.bstream,  BZ_FINISH);
        // 如果压缩结束，将错误码转换为 Z_STREAM_END
        if(err == BZ_STREAM_END)
          err = Z_STREAM_END;

        // 更新已缓冲数据的位置
        zi->ci.pos_in_buffered_data += (uInt)(zi->ci.bstream.total_out_lo32 - uTotalOutBefore);
      }

      // 如果压缩正常结束，将错误码转换为 ZIP_OK
      if(err == BZ_FINISH_OK)
        err = ZIP_OK;
#endif
    }

    // 如果流结束，将错误码转换为 ZIP_OK
    if (err==Z_STREAM_END)
        err=ZIP_OK; /* this is normal */

    // 如果存在未写入的缓冲数据，并且没有错误
    if ((zi->ci.pos_in_buffered_data>0) && (err==ZIP_OK))
                {
        // 如果刷新写缓冲区出错，将错误码设置为 ZIP_ERRNO
        if (zip64FlushWriteBuffer(zi)==ZIP_ERRNO)
            err = ZIP_ERRNO;
                }

    // 如果使用 DEFLATE 压缩，并且不是原始数据
    if ((zi->ci.method == Z_DEFLATED) && (!zi->ci.raw))
    {
        // 结束 DEFLATE 压缩流
        int tmp_err = deflateEnd(&zi->ci.stream);
        // 如果没有错误，将错误码设置为 tmp_err
        if (err == ZIP_OK)
            err = tmp_err;
        // 标记压缩流未初始化
        zi->ci.stream_initialised = 0;
    }
#ifdef HAVE_BZIP2
    // 如果使用 BZIP2 压缩，并且不是原始数据
    else if((zi->ci.method == Z_BZIP2ED) && (!zi->ci.raw))
    {
      // 结束 BZIP2 压缩流
      int tmperr = BZ2_bzCompressEnd(&zi->ci.bstream);
                        // 如果没有错误，将错误码设置为 tmperr
                        if (err==ZIP_OK)
                                err = tmperr;
                        // 标记压缩流未初始化
                        zi->ci.stream_initialised = 0;
    }
#endif

    // 如果不是原始数据
    if (!zi->ci.raw)
    {
        // 记录 CRC32 和未压缩大小
        crc32 = (uLong)zi->ci.crc32;
        uncompressed_size = zi->ci.totalUncompressedData;
    }
    // 记录压缩大小
    compressed_size = zi->ci.totalCompressedData;

#    ifndef NOCRYPT
    // 如果存在加密头，增加压缩大小
    compressed_size += zi->ci.crypt_header_size;
#    endif

    // 更新当前项目的 CRC 和大小
    if(compressed_size >= 0xffffffff || uncompressed_size >= 0xffffffff || zi->ci.pos_local_header >= 0xffffffff)
    {
      /*version Made by*/
      // 在中央目录头部的第4个字节处写入版本信息
      zip64local_putValue_inmemory(zi->ci.central_header+4,(uLong)45,2);
      /*version needed*/
      // 在中央目录头部的第6个字节处写入所需版本信息
      zip64local_putValue_inmemory(zi->ci.central_header+6,(uLong)45,2);

    }

    // 在中央目录头部的第16个字节处写入 CRC（循环冗余校验）值
    zip64local_putValue_inmemory(zi->ci.central_header+16,crc32,4); /*crc*/

    // 如果压缩后的文件大小大于等于0xffffffff，则在中央目录头部的第20个字节处写入无效值，否则写入压缩后的文件大小
    if(compressed_size >= 0xffffffff)
      zip64local_putValue_inmemory(zi->ci.central_header+20, invalidValue,4); /*compr size*/
    else
      zip64local_putValue_inmemory(zi->ci.central_header+20, compressed_size,4); /*compr size*/

    /// 设置内部文件属性字段
    if (zi->ci.stream.data_type == Z_ASCII)
        zip64local_putValue_inmemory(zi->ci.central_header+36,(uLong)Z_ASCII,2);

    // 如果未压缩的文件大小大于等于0xffffffff，则在中央目录头部的第24个字节处写入无效值，否则写入未压缩的文件大小
    if(uncompressed_size >= 0xffffffff)
      zip64local_putValue_inmemory(zi->ci.central_header+24, invalidValue,4); /*uncompr size*/
    else
      zip64local_putValue_inmemory(zi->ci.central_header+24, uncompressed_size,4); /*uncompr size*/

    // 如果未压缩的文件大小大于等于0xffffffff，则增加8个字节的 ZIP64 额外信息字段
    if(uncompressed_size >= 0xffffffff)
      datasize += 8;

    // 如果压缩后的文件大小大于等于0xffffffff，则增加8个字节的 ZIP64 额外信息字段
    if(compressed_size >= 0xffffffff)
      datasize += 8;

    // 如果当前文件的本地文件头的相对偏移量大于等于0xffffffff，则增加8个字节的 ZIP64 额外信息字段
    if(zi->ci.pos_local_header >= 0xffffffff)
      datasize += 8;

    // 如果数据大小大于0，则执行以下操作
    if(datasize > 0)
    {
      // 声明一个指向字符的指针，并初始化为 NULL
      char* p = NULL;

      // 检查是否可以将更多数据写入中央目录缓冲区
      if((uLong)(datasize + 4) > zi->ci.size_centralExtraFree)
      {
        // 如果没有足够的空间，返回 ZIP_BADZIPFILE 错误
        return ZIP_BADZIPFILE;
      }

      // 将 p 指向中央目录的末尾
      p = zi->ci.central_header + zi->ci.size_centralheader;

      // 为 'ZIP64 信息' 添加额外信息头
      zip64local_putValue_inmemory(p, 0x0001, 2); // HeaderID
      p += 2;
      zip64local_putValue_inmemory(p, datasize, 2); // DataSize
      p += 2;

      // 如果未压缩大小大于等于 0xffffffff
      if(uncompressed_size >= 0xffffffff)
      {
        zip64local_putValue_inmemory(p, uncompressed_size, 8);
        p += 8;
      }

      // 如果压缩大小大于等于 0xffffffff
      if(compressed_size >= 0xffffffff)
      {
        zip64local_putValue_inmemory(p, compressed_size, 8);
        p += 8;
      }

      // 如果本地头位置大于等于 0xffffffff
      if(zi->ci.pos_local_header >= 0xffffffff)
      {
        zip64local_putValue_inmemory(p, zi->ci.pos_local_header, 8);
        p += 8;
      }

      // 更新内存缓冲区中的额外空闲空间，并增加中央目录大小，以便包含新的 ZIP64 字段
      // （下面的 4 是 HeaderID 和 DataSize 字段的大小）
      zi->ci.size_centralExtraFree -= datasize + 4;
      zi->ci.size_centralheader += datasize + 4;

      // 更新额外信息大小字段
      zi->ci.size_centralExtra += datasize + 4;
      zip64local_putValue_inmemory(zi->ci.central_header+30,(uLong)zi->ci.size_centralExtra,2);
    }

    // 如果 err 为 ZIP_OK
    if (err==ZIP_OK)
        // 将中央目录的数据添加到数据块中
        err = add_data_in_datablock(&zi->central_dir, zi->ci.central_header, (uLong)zi->ci.size_centralheader);

    // 释放中央目录的内存
    free(zi->ci.central_header);

    // 如果 err 为 ZIP_OK
    if (err==ZIP_OK)
    {
        // 更新 LocalFileHeader 的新值。

        // 获取当前文件流的位置
        ZPOS64_T cur_pos_inzip = ZTELL64(zi->z_filefunc,zi->filestream);

        // 移动文件流到本地文件头的位置
        if (ZSEEK64(zi->z_filefunc,zi->filestream, zi->ci.pos_local_header + 14,ZLIB_FILEFUNC_SEEK_SET)!=0)
            err = ZIP_ERRNO;

        // 如果没有错误，将 crc32 值写入文件流
        if (err==ZIP_OK)
            err = zip64local_putValue(&zi->z_filefunc,zi->filestream,crc32,4); /* crc 32, unknown */

        // 如果未压缩大小或压缩大小大于等于 0xffffffff
        if(uncompressed_size >= 0xffffffff || compressed_size >= 0xffffffff )
        {
          // 如果存在 ZIP64 扩展信息
          if(zi->ci.pos_zip64extrainfo > 0)
          {
            // 更新 ZIP64 扩展字段中的大小
            if (ZSEEK64(zi->z_filefunc,zi->filestream, zi->ci.pos_zip64extrainfo + 4,ZLIB_FILEFUNC_SEEK_SET)!=0)
              err = ZIP_ERRNO;

            // 如果没有错误，将未压缩大小写入文件流
            if (err==ZIP_OK) /* compressed size, unknown */
              err = zip64local_putValue(&zi->z_filefunc, zi->filestream, uncompressed_size, 8);

            // 如果没有错误，将压缩大小写入文件流
            if (err==ZIP_OK) /* uncompressed size, unknown */
              err = zip64local_putValue(&zi->z_filefunc, zi->filestream, compressed_size, 8);
          }
          else
              err = ZIP_BADZIPFILE; // 调用者传递的 zip64 = 0，因此没有空间存储 zip64 信息 -> 致命错误
        }
        else
        {
          // 如果没有错误，将压缩大小写入文件流
          if (err==ZIP_OK) /* compressed size, unknown */
              err = zip64local_putValue(&zi->z_filefunc,zi->filestream,compressed_size,4);

          // 如果没有错误，将未压缩大小写入文件流
          if (err==ZIP_OK) /* uncompressed size, unknown */
              err = zip64local_putValue(&zi->z_filefunc,zi->filestream,uncompressed_size,4);
        }

        // 将文件流移动到之前的位置
        if (ZSEEK64(zi->z_filefunc,zi->filestream, cur_pos_inzip,ZLIB_FILEFUNC_SEEK_SET)!=0)
            err = ZIP_ERRNO;
    }

    // 增加 ZIP 文件中的条目数
    zi->number_entry ++;
    // 重置在 ZIP 文件中打开的文件标志
    zi->in_opened_file_inzip = 0;

    // 返回错误码
    return err;
}

# 关闭正在写入的文件
extern int ZEXPORT zipCloseFileInZip (zipFile file)
{
    # 调用 zipCloseFileInZipRaw 函数关闭正在写入的文件
    return zipCloseFileInZipRaw (file,0,0);
}

# 写入 Zip64 中央目录定位器
local int Write_Zip64EndOfCentralDirectoryLocator(zip64_internal* zi, ZPOS64_T zip64eocd_pos_inzip)
{
  int err = ZIP_OK;
  ZPOS64_T pos = zip64eocd_pos_inzip - zi->add_position_when_writing_offset;

  # 写入 ZIP64ENDLOCHEADERMAGIC 到文件流中
  err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)ZIP64ENDLOCHEADERMAGIC,4);

  # 写入磁盘号，这里始终为0
    if (err==ZIP_OK) 
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4);

  # 写入相对偏移量
    if (err==ZIP_OK) 
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream, pos,8);

  # 写入总磁盘数，这里始终为1
    if (err==ZIP_OK) 
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)1,4);

    return err;
}

# 写入 Zip64 中央目录记录
local int Write_Zip64EndOfCentralDirectoryRecord(zip64_internal* zi, uLong size_centraldir, ZPOS64_T centraldir_pos_inzip)
{
  int err = ZIP_OK;

  uLong Zip64DataSize = 44;

  # 写入 ZIP64ENDHEADERMAGIC 到文件流中
  err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)ZIP64ENDHEADERMAGIC,4);

  # 写入 ZIP64 数据大小
  if (err==ZIP_OK) 
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(ZPOS64_T)Zip64DataSize,8); // why ZPOS64_T of this ?

  # 写入创建版本号
  if (err==ZIP_OK) 
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)45,2);

  # 写入所需版本号
  if (err==ZIP_OK) 
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)45,2);

  # 写入磁盘号
  if (err==ZIP_OK) 
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4);

  # 写入中央目录开始的磁盘号
  if (err==ZIP_OK) 
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4);
    # 将值放入 ZIP64 本地文件头
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,4);

  # 如果没有错误，将磁盘上中央目录中的条目总数放入 ZIP64 本地文件头
  if (err==ZIP_OK) /* total number of entries in the central dir on this disk */
    err = zip64local_putValue(&zi->z_filefunc, zi->filestream, zi->number_entry, 8);

  # 如果没有错误，将中央目录中的条目总数放入 ZIP64 本地文件头
  if (err==ZIP_OK) /* total number of entries in the central dir */
    err = zip64local_putValue(&zi->z_filefunc, zi->filestream, zi->number_entry, 8);

  # 如果没有错误，将中央目录的大小放入 ZIP64 本地文件头
  if (err==ZIP_OK) /* size of the central directory */
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(ZPOS64_T)size_centraldir,8);

  # 如果没有错误，将中央目录的起始偏移量放入 ZIP64 本地文件头
  if (err==ZIP_OK) /* offset of start of central directory with respect to the starting disk number */
  {
    # 计算中央目录在 ZIP 文件中的偏移量
    ZPOS64_T pos = centraldir_pos_inzip - zi->add_position_when_writing_offset;
    # 将偏移量放入 ZIP64 本地文件头
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream, (ZPOS64_T)pos,8);
  }
  # 返回错误码
  return err;
}
# 写入中央目录结束记录
local int Write_EndOfCentralDirectoryRecord(zip64_internal* zi, uLong size_centraldir, ZPOS64_T centraldir_pos_inzip)
{
  int err = ZIP_OK;

  /*signature*/
  # 写入结束记录的标识
  err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)ENDHEADERMAGIC,4);

  if (err==ZIP_OK) /* number of this disk */
    # 写入磁盘编号
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,2);

  if (err==ZIP_OK) /* number of the disk with the start of the central directory */
    # 写入包含中央目录起始位置的磁盘编号
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0,2);

  if (err==ZIP_OK) /* total number of entries in the central dir on this disk */
  {
    {
      if(zi->number_entry >= 0xFFFF)
        # 如果文件数超过了 0xFFFF，则使用 ZIP64 记录中的值
        err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0xffff,2); // use value in ZIP64 record
      else
        # 否则写入文件数
        err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)zi->number_entry,2);
    }
  }

  if (err==ZIP_OK) /* total number of entries in the central dir */
  {
    if(zi->number_entry >= 0xFFFF)
      # 如果文件数超过了 0xFFFF，则使用 ZIP64 记录中的值
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)0xffff,2); // use value in ZIP64 record
    else
      # 否则写入文件数
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)zi->number_entry,2);
  }

  if (err==ZIP_OK) /* size of the central directory */
    # 写入中央目录的大小
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)size_centraldir,4);

  if (err==ZIP_OK) /* offset of start of central directory with respect to the starting disk number */
  {
    ZPOS64_T pos = centraldir_pos_inzip - zi->add_position_when_writing_offset;
    if(pos >= 0xffffffff)
    {
      # 如果偏移量超过了 0xffffffff，则使用 ZIP64 记录中的值
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream, (uLong)0xffffffff,4);
    }
    else
      # 否则写入偏移量
      err = zip64local_putValue(&zi->z_filefunc,zi->filestream, (uLong)(centraldir_pos_inzip - zi->add_position_when_writing_offset),4);
  }

   return err;
}

# 写入全局注释
local int Write_GlobalComment(zip64_internal* zi, const char* global_comment)
{
  int err = ZIP_OK;
  uInt size_global_comment = 0;

  if(global_comment != NULL)
    # 计算全局注释的长度
    size_global_comment = (uInt)strlen(global_comment);

    # 将全局注释的长度写入文件流，使用 ZIP64 格式
    err = zip64local_putValue(&zi->z_filefunc,zi->filestream,(uLong)size_global_comment,2);

    # 如果写入成功且全局注释长度大于0
    if (err == ZIP_OK && size_global_comment > 0)
    {
        # 将全局注释内容写入文件流
        if (ZWRITE64(zi->z_filefunc,zi->filestream, global_comment, size_global_comment) != size_global_comment)
            err = ZIP_ERRNO;
    }
    # 返回操作结果
    return err;
# 关闭 ZIP 文件
extern int ZEXPORT zipClose (zipFile file, const char* global_comment)
{
    zip64_internal* zi;  # 定义 ZIP 内部结构体指针
    int err = 0;  # 初始化错误码为 0
    uLong size_centraldir = 0;  # 初始化中央目录大小为 0
    ZPOS64_T centraldir_pos_inzip;  # 定义中央目录在 ZIP 文件中的位置
    ZPOS64_T pos;  # 定义位置变量

    if (file == NULL)  # 如果文件为空
        return ZIP_PARAMERROR;  # 返回参数错误

    zi = (zip64_internal*)file;  # 将文件转换为 ZIP 内部结构体指针

    if (zi->in_opened_file_inzip == 1)  # 如果文件在 ZIP 中已经打开
    {
        err = zipCloseFileInZip (file);  # 关闭 ZIP 中的文件
    }

#ifndef NO_ADDFILEINEXISTINGZIP
    if (global_comment==NULL)  # 如果全局注释为空
        global_comment = zi->globalcomment;  # 使用 ZIP 内部结构体中的全局注释
#endif

    centraldir_pos_inzip = ZTELL64(zi->z_filefunc,zi->filestream);  # 获取中央目录在 ZIP 文件中的位置

    if (err==ZIP_OK)  # 如果没有错误
    {
        linkedlist_datablock_internal* ldi = zi->central_dir.first_block;  # 获取中央目录的第一个数据块
        while (ldi!=NULL)  # 遍历数据块
        {
            if ((err==ZIP_OK) && (ldi->filled_in_this_block>0))  # 如果没有错误且数据块中有数据
            {
                if (ZWRITE64(zi->z_filefunc,zi->filestream, ldi->data, ldi->filled_in_this_block) != ldi->filled_in_this_block)  # 将数据块中的数据写入 ZIP 文件
                    err = ZIP_ERRNO;  # 如果写入失败，设置错误码为系统错误
            }

            size_centraldir += ldi->filled_in_this_block;  # 更新中央目录大小
            ldi = ldi->next_datablock;  # 获取下一个数据块
        }
    }
    free_linkedlist(&(zi->central_dir));  # 释放中央目录的链表内存

    pos = centraldir_pos_inzip - zi->add_position_when_writing_offset;  # 计算位置偏移
    if(pos >= 0xffffffff || zi->number_entry > 0xFFFF)  # 如果位置偏移大于等于 0xffffffff 或者 ZIP 文件中的条目数大于 0xFFFF
    {
      ZPOS64_T Zip64EOCDpos = ZTELL64(zi->z_filefunc,zi->filestream);  # 获取 ZIP64 结束中央目录记录的位置
      Write_Zip64EndOfCentralDirectoryRecord(zi, size_centraldir, centraldir_pos_inzip);  # 写入 ZIP64 结束中央目录记录

      Write_Zip64EndOfCentralDirectoryLocator(zi, Zip64EOCDpos);  # 写入 ZIP64 结束中央目录定位器
    }

    if (err==ZIP_OK)  # 如果没有错误
      err = Write_EndOfCentralDirectoryRecord(zi, size_centraldir, centraldir_pos_inzip);  # 写入中央目录结束记录

    if(err == ZIP_OK)  # 如果没有错误
      err = Write_GlobalComment(zi, global_comment);  # 写入全局注释

    if (ZCLOSE64(zi->z_filefunc,zi->filestream) != 0)  # 如果关闭 ZIP 文件失败
        if (err == ZIP_OK)  # 如果之前没有错误
            err = ZIP_ERRNO;  # 设置错误码为系统错误

#ifndef NO_ADDFILEINEXISTINGZIP
    TRYFREE(zi->globalcomment);  # 释放全局注释内存
#endif
    TRYFREE(zi);  # 释放 ZIP 内部结构体内存

    return err;  # 返回错误码
}

extern int ZEXPORT zipRemoveExtraInfoBlock (char* pData, int* dataLen, short sHeader)  # 定义移除额外信息块的函数
{
  // 定义指向数据的字符指针
  char* p = pData;
  // 初始化变量 size 为 0
  int size = 0;
  // 定义指向新头部的字符指针
  char* pNewHeader;
  // 定义临时指针
  char* pTmp;
  // 定义 short 类型的 header 和 dataSize
  short header;
  short dataSize;

  // 初始化返回值为 ZIP_OK
  int retVal = ZIP_OK;

  // 检查参数是否合法
  if(pData == NULL || dataLen == NULL || *dataLen < 4)
    return ZIP_PARAMERROR;

  // 分配新头部的内存空间
  pNewHeader = (char*)ALLOC((unsigned)*dataLen);
  // 将临时指针指向新头部
  pTmp = pNewHeader;

  // 循环处理数据
  while(p < (pData + *dataLen))
  {
    // 读取头部和数据大小
    header = *(short*)p;
    dataSize = *(((short*)p)+1);

    // 如果找到指定的头部
    if( header == sHeader ) // Header found.
    {
      // 跳过该部分数据，不复制到临时缓冲区
      p += dataSize + 4; // skip it. do not copy to temp buffer
    }
    else
    {
      // 复制额外信息块到临时缓冲区
      memcpy(pTmp, p, dataSize + 4);
      // 移动指针
      p += dataSize + 4;
      // 更新 size
      size += dataSize + 4;
    }

  }

  // 如果处理后的数据大小小于原始数据大小
  if(size < *dataLen)
  {
    // 清空原始额外信息块
    memset(pData,0, *dataLen);

    // 如果 size 大于 0，则将新的额外信息块复制到原始数据中
    if(size > 0)
      memcpy(pData, pNewHeader, size);

    // 更新额外信息块的大小
    *dataLen = size;

    // 设置返回值为 ZIP_OK
    retVal = ZIP_OK;
  }
  else
    // 设置返回值为 ZIP_ERRNO
    retVal = ZIP_ERRNO;

  // 释放新头部的内存空间
  TRYFREE(pNewHeader);

  // 返回结果
  return retVal;
}
```