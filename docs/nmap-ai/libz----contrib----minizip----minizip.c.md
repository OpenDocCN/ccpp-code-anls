# `nmap\libz\contrib\minizip\minizip.c`

```
/*
   minizip.c
   Version 1.1, February 14h, 2010
   sample part of the MiniZip project - ( http://www.winimage.com/zLibDll/minizip.html )

         Copyright (C) 1998-2010 Gilles Vollant (minizip) ( http://www.winimage.com/zLibDll/minizip.html )

         Modifications of Unzip for Zip64
         Copyright (C) 2007-2008 Even Rouault

         Modifications for Zip64 support on both zip and unzip
         Copyright (C) 2009-2010 Mathias Svensson ( http://result42.com )
*/

#if (!defined(_WIN32)) && (!defined(WIN32)) && (!defined(__APPLE__))
        #ifndef __USE_FILE_OFFSET64
                #define __USE_FILE_OFFSET64
        #endif
        #ifndef __USE_LARGEFILE64
                #define __USE_LARGEFILE64
        #endif
        #ifndef _LARGEFILE64_SOURCE
                #define _LARGEFILE64_SOURCE
        #endif
        #ifndef _FILE_OFFSET_BIT
                #define _FILE_OFFSET_BIT 64
        #endif
#endif

#ifdef __APPLE__
// In darwin and perhaps other BSD variants off_t is a 64 bit value, hence no need for specific 64 bit functions
#define FOPEN_FUNC(filename, mode) fopen(filename, mode)
#define FTELLO_FUNC(stream) ftello(stream)
#define FSEEKO_FUNC(stream, offset, origin) fseeko(stream, offset, origin)
#else
#define FOPEN_FUNC(filename, mode) fopen64(filename, mode)
#define FTELLO_FUNC(stream) ftello64(stream)
#define FSEEKO_FUNC(stream, offset, origin) fseeko64(stream, offset, origin)
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>
#include <fcntl.h>

#ifdef _WIN32
# include <direct.h>
# include <io.h>
#else
# include <unistd.h>
# include <utime.h>
# include <sys/types.h>
# include <sys/stat.h>
#endif

#include "zip.h"

#ifdef _WIN32
        #define USEWIN32IOAPI
        #include "iowin32.h"
#endif

#define WRITEBUFFERSIZE (16384)  // 定义写缓冲区大小
#define MAXFILENAME (256)  // 定义最大文件名长度

#ifdef _WIN32
static int filetime(f, tmzip, dt)  // 定义文件时间函数
    const char *f;          /* name of file to get info on */
    // 定义一个指向 tm_zip 结构体的指针，用于返回访问、修改和创建时间
    tm_zip *tmzip;
    // 定义一个指向 uLong 类型的指针，用于存储 dostime
    uLong *dt;
{
  // 定义返回值变量 ret
  int ret = 0;
  {
      // 定义本地文件时间变量 ftLocal
      FILETIME ftLocal;
      // 定义文件查找句柄变量 hFind
      HANDLE hFind;
      // 定义 WIN32_FIND_DATAA 结构体变量 ff32
      WIN32_FIND_DATAA ff32;

      // 使用文件名 f 进行文件查找，获取文件信息
      hFind = FindFirstFileA(f,&ff32);
      // 如果文件查找句柄有效
      if (hFind != INVALID_HANDLE_VALUE)
      {
        // 将文件最后修改时间转换为本地文件时间
        FileTimeToLocalFileTime(&(ff32.ftLastWriteTime),&ftLocal);
        // 将本地文件时间转换为 DOS 时间
        FileTimeToDosDateTime(&ftLocal,((LPWORD)dt)+1,((LPWORD)dt)+0);
        // 关闭文件查找句柄
        FindClose(hFind);
        // 设置返回值为 1
        ret = 1;
      }
  }
  // 返回结果
  return ret;
}
#else
#if defined(unix) || defined(__APPLE__)
// 定义获取文件时间的函数
static int filetime(f, tmzip, dt)
    const char *f;         /* name of file to get info on */
    tm_zip *tmzip;         /* return value: access, modific. and creation times */
    uLong *dt;             /* dostime */
{
  (void)dt;
  // 初始化返回值为 0
  int ret=0;
  // 定义文件信息结构体
  struct stat s;        /* results of stat() */
  // 定义文件时间结构体指针
  struct tm* filedate;
  // 初始化时间变量为 0
  time_t tm_t=0;

  // 如果文件名不是 "-"，即不是标准输入
  if (strcmp(f,"-")!=0)
  {
    // 定义文件名变量
    char name[MAXFILENAME+1];
    // 获取文件名长度
    size_t len = strlen(f);
    // 如果文件名长度超过最大长度，则截断
    if (len > MAXFILENAME)
      len = MAXFILENAME;

    // 将文件名复制到 name 变量中
    strncpy(name, f,MAXFILENAME-1);
    /* strncpy doesnt append the trailing NULL, of the string is too long. */
    // 添加字符串结尾的 NULL 字符
    name[ MAXFILENAME ] = '\0';

    // 如果文件名最后一个字符是 "/"，则去掉
    if (name[len - 1] == '/')
      name[len - 1] = '\0';
    // 不是所有系统都允许在文件名末尾加 "/"
    // 获取文件信息
    if (stat(name,&s)==0)
    {
      // 获取文件最后修改时间
      tm_t = s.st_mtime;
      // 设置返回值为 1
      ret = 1;
    }
  }
  // 将时间转换为本地时间
  filedate = localtime(&tm_t);

  // 设置 tmzip 结构体中的时间信息
  tmzip->tm_sec  = filedate->tm_sec;
  tmzip->tm_min  = filedate->tm_min;
  tmzip->tm_hour = filedate->tm_hour;
  tmzip->tm_mday = filedate->tm_mday;
  tmzip->tm_mon  = filedate->tm_mon ;
  tmzip->tm_year = filedate->tm_year;

  // 返回结果
  return ret;
}
#else
// 定义获取文件时间的函数
uLong filetime(f, tmzip, dt)
    const char *f;          /* name of file to get info on */
    tm_zip *tmzip;             /* return value: access, modific. and creation times */
    uLong *dt;             /* dostime */
{
    // 默认返回 0
    return 0;
}
#endif
#endif

// 检查文件是否存在
static int check_exist_file(filename)
    const char* filename;
{
    // 定义文件指针变量
    FILE* ftestexist;
    // 初始化返回值为 1
    int ret = 1;
    // 以只读方式打开文件
    ftestexist = FOPEN_FUNC(filename,"rb");
    // 如果文件指针为空
    if (ftestexist==NULL)
        // 设置返回值为 0
        ret = 0;
    # 如果条件不成立，则关闭ftestexist文件
    else
        fclose(ftestexist);
    # 返回ret的值
    return ret;
}

static void do_banner()
{
    // 打印 MiniZip 的版本信息和作者信息
    printf("MiniZip 1.1, demo of zLib + MiniZip64 package, written by Gilles Vollant\n");
    printf("more info on MiniZip at http://www.winimage.com/zLibDll/minizip.html\n\n");
}

static void do_help()
{
    // 打印 MiniZip 的使用帮助信息
    printf("Usage : minizip [-o] [-a] [-0 to -9] [-p password] [-j] file.zip [files_to_add]\n\n" \
           "  -o  Overwrite existing file.zip\n" \
           "  -a  Append to existing file.zip\n" \
           "  -0  Store only\n" \
           "  -1  Compress faster\n" \
           "  -9  Compress better\n\n" \
           "  -j  exclude path. store only the file name.\n\n");
}

/* 计算文件的 CRC32 值，
   因为要加密文件，需要知道文件的 CRC32 值 */
static int getFileCrc(const char* filenameinzip,void*buf,unsigned long size_buf,unsigned long* result_crc)
{
   unsigned long calculate_crc=0;
   int err=ZIP_OK;
   FILE * fin = FOPEN_FUNC(filenameinzip,"rb");

   unsigned long size_read = 0;
   /* unsigned long total_read = 0; */
   if (fin==NULL)
   {
       err = ZIP_ERRNO;
   }

    if (err == ZIP_OK)
        do
        {
            err = ZIP_OK;
            size_read = fread(buf,1,size_buf,fin);
            if (size_read < size_buf)
                if (feof(fin)==0)
            {
                printf("error in reading %s\n",filenameinzip);
                err = ZIP_ERRNO;
            }

            if (size_read>0)
                calculate_crc = crc32_z(calculate_crc,buf,size_read);
            /* total_read += size_read; */

        } while ((err == ZIP_OK) && (size_read>0));

    if (fin)
        fclose(fin);

    *result_crc=calculate_crc;
    printf("file %s crc %lx\n", filenameinzip, calculate_crc);
    return err;
}

static int isLargeFile(const char* filename)
{
  int largeFile = 0;
  ZPOS64_T pos = 0;
  FILE* pFile = FOPEN_FUNC(filename, "rb");

  if(pFile != NULL)
  {
    FSEEKO_FUNC(pFile, 0, SEEK_END);
    # 获取文件当前位置的偏移量，使用 ZPOS64_T 类型进行存储
    pos = (ZPOS64_T)FTELLO_FUNC(pFile);
    
    # 打印文件名和文件大小
    printf("File : %s is %lld bytes\n", filename, pos);
    
    # 如果文件大小超过 0xffffffff，则设置 largeFile 为 1
    if(pos >= 0xffffffff)
        largeFile = 1;
    
    # 关闭文件指针
    fclose(pFile);
    
    # 返回 largeFile 的值
    return largeFile;
}
# 结束 main 函数

int main(argc,argv)
    int argc;
    char *argv[];
{
    int i;
    int opt_overwrite=0;
    int opt_compress_level=Z_DEFAULT_COMPRESSION;
    int opt_exclude_path=0;
    int zipfilenamearg = 0;
    char filename_try[MAXFILENAME+16];
    int zipok;
    int err=0;
    size_t size_buf=0;
    void* buf=NULL;
    const char* password=NULL;

    # 打印程序的横幅信息
    do_banner();
    # 如果命令行参数数量为1，则打印帮助信息并返回
    if (argc==1)
    {
        do_help();
        return 0;
    }
    else
    {
        # 遍历命令行参数
        for (i=1;i<argc;i++)
        {
            # 如果参数以 '-' 开头
            if ((*argv[i])=='-')
            {
                const char *p=argv[i]+1;

                # 遍历参数中的每个字符
                while ((*p)!='\0')
                {
                    char c=*(p++);
                    # 如果参数包含 'o' 或 'O'，则设置 opt_overwrite 为 1
                    if ((c=='o') || (c=='O'))
                        opt_overwrite = 1;
                    # 如果参数包含 'a' 或 'A'，则设置 opt_overwrite 为 2
                    if ((c=='a') || (c=='A'))
                        opt_overwrite = 2;
                    # 如果参数是数字，则设置 opt_compress_level 为对应的数字
                    if ((c>='0') && (c<='9'))
                        opt_compress_level = c-'0';
                    # 如果参数包含 'j' 或 'J'，则设置 opt_exclude_path 为 1
                    if ((c=='j') || (c=='J'))
                        opt_exclude_path = 1;

                    # 如果参数包含 'p' 或 'P'，并且后面还有参数，则将下一个参数作为密码
                    if (((c=='p') || (c=='P')) && (i+1<argc))
                    {
                        password=argv[i+1];
                        i++;
                    }
                }
            }
            else
            {
                # 如果不是以 '-' 开头的参数，则将其作为压缩文件名
                if (zipfilenamearg == 0)
                {
                    zipfilenamearg = i ;
                }
            }
        }
    }

    # 分配写缓冲区的内存
    size_buf = WRITEBUFFERSIZE;
    buf = (void*)malloc(size_buf);
    # 如果内存分配失败，则打印错误信息并返回 ZIP_INTERNALERROR
    if (buf==NULL)
    {
        printf("Error allocating memory\n");
        return ZIP_INTERNALERROR;
    }

    # 如果压缩文件名参数为0，则将 zipok 设置为0
    if (zipfilenamearg==0)
    {
        zipok=0;
    }
    else
    # 定义变量 i 和 len，用于循环和存储字符串长度
    int i,len;
    # 定义变量 dot_found，并初始化为 0
    int dot_found=0;

    # 初始化变量 zipok 为 1
    zipok = 1 ;
    # 将命令行参数中的 ZIP 文件名拷贝到 filename_try 中，最大长度为 MAXFILENAME-1
    strncpy(filename_try, argv[zipfilenamearg],MAXFILENAME-1);
    /* strncpy 不会在字符串过长时追加结尾的 NULL 字符。*/
    # 手动在 filename_try 的最大长度处添加结尾的 NULL 字符
    filename_try[ MAXFILENAME ] = '\0';

    # 计算 filename_try 的长度
    len=(int)strlen(filename_try);
    # 遍历 filename_try 中的字符，查找是否有点号
    for (i=0;i<len;i++)
        if (filename_try[i]=='.')
            dot_found=1;

    # 如果未找到点号，则在 filename_try 后追加 ".zip"
    if (dot_found==0)
        strcat(filename_try,".zip");

    # 如果 opt_overwrite 等于 2
    if (opt_overwrite==2)
    {
        /* 如果文件不存在，我们不追加文件 */
        # 如果文件不存在，则将 opt_overwrite 设置为 1
        if (check_exist_file(filename_try)==0)
            opt_overwrite=1;
    }
    # 否则，如果 opt_overwrite 等于 0
    else
    if (opt_overwrite==0)
        # 如果文件存在
        if (check_exist_file(filename_try)!=0)
        {
            # 定义变量 rep 并初始化为 0
            char rep=0;
            # 循环直到用户输入合法的选项
            do
            {
                # 定义变量 answer 用于存储用户输入的答案
                char answer[128];
                int ret;
                # 提示用户输入是否覆盖文件，追加文件还是取消操作
                printf("The file %s exists. Overwrite ? [y]es, [n]o, [a]ppend : ",filename_try);
                # 读取用户输入的答案
                ret = scanf("%1s",answer);
                # 如果读取失败，则退出程序
                if (ret != 1)
                {
                   exit(EXIT_FAILURE);
                }
                # 将用户输入的答案转换为大写字母
                rep = answer[0] ;
                if ((rep>='a') && (rep<='z'))
                    rep -= 0x20;
            }
            # 当用户输入不是 Y、N 或 A 时，继续循环
            while ((rep!='Y') && (rep!='N') && (rep!='A'));
            # 如果用户选择不覆盖文件，则将 zipok 设置为 0
            if (rep=='N')
                zipok = 0;
            # 如果用户选择追加文件，则将 opt_overwrite 设置为 2
            if (rep=='A')
                opt_overwrite = 2;
        }
    }

    # 如果 zipok 为 1
    if (zipok==1)
    {
        # 定义变量 zf 为 zipFile 类型
        zipFile zf;
        # 定义变量 errclose
        int errclose;
# 如果定义了 USEWIN32IOAPI，则使用 win32 文件操作函数来填充文件操作结构体
        zlib_filefunc64_def ffunc;
        fill_win32_filefunc64A(&ffunc);
        # 使用给定的文件名打开或创建一个 ZIP 文件
        zf = zipOpen2_64(filename_try,(opt_overwrite==2) ? 2 : 0,NULL,&ffunc);
#        else
        # 否则，使用给定的文件名打开或创建一个 ZIP 文件
        zf = zipOpen64(filename_try,(opt_overwrite==2) ? 2 : 0);
#        endif

        # 如果打开或创建 ZIP 文件失败
        if (zf == NULL)
        {
            # 打印错误信息
            printf("error opening %s\n",filename_try);
            # 设置错误码为 ZIP_ERRNO
            err= ZIP_ERRNO;
        }
        # 否则
        else
            # 打印创建 ZIP 文件的信息
            printf("creating %s\n",filename_try);

        # 遍历命令行参数中的文件名
        for (i=zipfilenamearg+1;(i<argc) && (err==ZIP_OK);i++)
        {
            # 如果命令行参数不是选项
            if (!((((*(argv[i]))=='-') || ((*(argv[i]))=='/')) &&
                  ((argv[i][1]=='o') || (argv[i][1]=='O') ||
                   (argv[i][1]=='a') || (argv[i][1]=='A') ||
                   (argv[i][1]=='p') || (argv[i][1]=='P') ||
                   ((argv[i][1]>='0') || (argv[i][1]<='9'))) &&
                  (strlen(argv[i]) == 2)))
            {
                # 打开要添加到 ZIP 文件中的文件
                FILE * fin;
                size_t size_read;
                const char* filenameinzip = argv[i];
                const char *savefilenameinzip;
                zip_fileinfo zi;
                unsigned long crcFile=0;
                int zip64 = 0;

                # 初始化 ZIP 文件信息结构体
                zi.tmz_date.tm_sec = zi.tmz_date.tm_min = zi.tmz_date.tm_hour =
                zi.tmz_date.tm_mday = zi.tmz_date.tm_mon = zi.tmz_date.tm_year = 0;
                zi.dosDate = 0;
                zi.internal_fa = 0;
                zi.external_fa = 0;
                # 获取文件的时间信息
                filetime(filenameinzip,&zi.tmz_date,&zi.dosDate);

                # 打开一个新文件并将其添加到 ZIP 文件中
                err = zipOpenNewFileInZip(zf,filenameinzip,&zi,
                                 NULL,0,NULL,0,NULL / * comment * /,
                                 (opt_compress_level != 0) ? Z_DEFLATED : 0,
                                 opt_compress_level);
    }
    else
    {
       # 执行帮助函数
       do_help();
    }

    # 释放缓冲区内存
    free(buf);
    # 返回 0
    return 0;
}
```