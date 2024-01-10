# `nmap\libz\contrib\minizip\miniunz.c`

```
/*
   miniunz.c
   Version 1.1, February 14h, 2010
   sample part of the MiniZip project - ( http://www.winimage.com/zLibDll/minizip.html )

         Copyright (C) 1998-2010 Gilles Vollant (minizip) ( http://www.winimage.com/zLibDll/minizip.html )

         Modifications of Unzip for Zip64
         Copyright (C) 2007-2008 Even Rouault

         Modifications for Zip64 support on both zip and unzip
         Copyright (C) 2009-2010 Mathias Svensson ( http://result42.com )
*/
// 如果未定义_WIN32、WIN32和__APPLE__，则定义__USE_FILE_OFFSET64、__USE_LARGEFILE64、_LARGEFILE64_SOURCE和_FILE_OFFSET_BIT 64
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
// 在darwin和其他BSD变体中，off_t是一个64位值，因此不需要特定的64位函数
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
#include <sys/stat.h>

#ifdef _WIN32
# include <direct.h>
# include <io.h>
#else
# include <unistd.h>
# include <utime.h>
#endif


#include "unzip.h"

#define CASESENSITIVITY (0)
#define WRITEBUFFERSIZE (8192)
#define MAXFILENAME (256)

#ifdef _WIN32
#define USEWIN32IOAPI
#include "iowin32.h"
#endif
/* mini unzip, demo of unzip package
   演示解压缩包的迷你解压缩程序

   usage :
   用法：
   Usage : miniunz [-exvlo] file.zip [file_to_extract] [-d extractdir]
   用法：miniunz [-exvlo] file.zip [file_to_extract] [-d extractdir]

   list the file in the zipfile, and print the content of FILE_ID.ZIP or README.TXT
   if it exists
   列出压缩包中的文件，并打印FILE_ID.ZIP或README.TXT的内容（如果存在）
*/

/* change_file_date : change the date/time of a file
   更改文件的日期/时间
   filename : the filename of the file where date/time must be modified
   文件名：需要修改日期/时间的文件的文件名
   dosdate : the new date at the MSDos format (4 bytes)
   dosdate：MSDos格式的新日期（4个字节）
   tmu_date : the SAME new date at the tm_unz format
   tmu_date：tm_unz格式的相同新日期
*/
static void change_file_date(filename,dosdate,tmu_date)
    const char *filename;
    uLong dosdate;
    tm_unz tmu_date;
{
#ifdef _WIN32
  HANDLE hFile;
  FILETIME ftm,ftLocal,ftCreate,ftLastAcc,ftLastWrite;

  hFile = CreateFileA(filename,GENERIC_READ | GENERIC_WRITE,
                      0,NULL,OPEN_EXISTING,0,NULL);
  GetFileTime(hFile,&ftCreate,&ftLastAcc,&ftLastWrite);
  DosDateTimeToFileTime((WORD)(dosdate>>16),(WORD)dosdate,&ftLocal);
  LocalFileTimeToFileTime(&ftLocal,&ftm);
  SetFileTime(hFile,&ftm,&ftLastAcc,&ftm);
  CloseHandle(hFile);
#else
#if defined(unix) || defined(__APPLE__)
  (void)dosdate;
  struct utimbuf ut;
  struct tm newdate;
  newdate.tm_sec = tmu_date.tm_sec;
  newdate.tm_min=tmu_date.tm_min;
  newdate.tm_hour=tmu_date.tm_hour;
  newdate.tm_mday=tmu_date.tm_mday;
  newdate.tm_mon=tmu_date.tm_mon;
  if (tmu_date.tm_year > 1900)
      newdate.tm_year=tmu_date.tm_year - 1900;
  else
      newdate.tm_year=tmu_date.tm_year ;
  newdate.tm_isdst=-1;

  ut.actime=ut.modtime=mktime(&newdate);
  utime(filename,&ut);
#endif
#endif
}

/* mymkdir and change_file_date are not 100 % portable
   As I don't know well Unix, I wait feedback for the unix portion */
/* mymkdir和change_file_date不是100%可移植的
   因为我不太了解Unix，所以我等待Unix部分的反馈 */

static int mymkdir(dirname)
    const char* dirname;
{
    int ret=0;
#ifdef _WIN32
    ret = _mkdir(dirname);
#elif unix
    ret = mkdir (dirname,0775);
#elif __APPLE__
    ret = mkdir (dirname,0775);
#endif
    return ret;
}

static int makedir (newdir)
    const char *newdir;
{
  // 定义字符指针变量buffer
  char *buffer ;
  // 定义字符指针变量p
  char *p;
  // 获取newdir的长度
  size_t len = strlen(newdir);

  // 如果长度为0，返回0
  if (len == 0)
    return 0;

  // 分配内存给buffer
  buffer = (char*)malloc(len+1);
  // 如果分配内存失败，打印错误信息并返回UNZ_INTERNALERROR
  if (buffer==NULL)
  {
    printf("Error allocating memory\n");
    return UNZ_INTERNALERROR;
  }
  // 将newdir的内容复制到buffer
  strcpy(buffer,newdir);

  // 如果buffer的最后一个字符是'/'，将其替换为'\0'
  if (buffer[len-1] == '/') {
    buffer[len-1] = '\0';
  }
  // 如果mymkdir(buffer)返回0，释放内存并返回1
  if (mymkdir(buffer) == 0)
  {
    free(buffer);
    return 1;
  }

  // 将p指向buffer的下一个字符
  p = buffer+1;
  // 进入循环
  while (1)
  {
    // 定义字符变量hold
    char hold;

    // 移动p指针直到遇到'\\'或'/'或'\0'
    while(*p && *p != '\\' && *p != '/')
      p++;
    // 将*p的值保存到hold中，并将*p置为'\0'
    hold = *p;
    *p = 0;
    // 如果mymkdir(buffer)返回-1并且errno为ENOENT，打印错误信息，释放内存并返回0
    if ((mymkdir(buffer) == -1) && (errno == ENOENT))
    {
      printf("couldn't create directory %s\n",buffer);
      free(buffer);
      return 0;
    }
    // 如果hold为0，跳出循环
    if (hold == 0)
      break;
    // 将*p的值设置为hold，p指向下一个字符
    *p++ = hold;
  }
  // 释放内存并返回1
  free(buffer);
  return 1;
}

// 打印程序的横幅信息
static void do_banner()
{
    printf("MiniUnz 1.01b, demo of zLib + Unz package written by Gilles Vollant\n");
    printf("more info at http://www.winimage.com/zLibDll/unzip.html\n\n");
}

// 打印程序的帮助信息
static void do_help()
{
    printf("Usage : miniunz [-e] [-x] [-v] [-l] [-o] [-p password] file.zip [file_to_extr.] [-d extractdir]\n\n" \
           "  -e  Extract without pathname (junk paths)\n" \
           "  -x  Extract with pathname\n" \
           "  -v  list files\n" \
           "  -l  list files\n" \
           "  -d  directory to extract into\n" \
           "  -o  overwrite files without prompting\n" \
           "  -p  extract crypted file using password\n\n");
}

// 显示64位大小的信息
static void Display64BitsSize(ZPOS64_T n, int size_char)
{
  /* 为了避免兼容性问题，在这里进行转换 */
  char number[21];  // 创建一个长度为21的字符数组，用于存储数字的字符串形式
  int offset=19;  // 设置偏移量为19
  int pos_string = 19;  // 设置字符串位置为19
  number[20]=0;  // 将数组最后一个元素设置为0，表示字符串结束
  for (;;) {  // 无限循环
      number[offset]=(char)((n%10)+'0');  // 将数字的个位数转换为字符存入数组
      if (number[offset] != '0')  // 如果当前字符不是0
          pos_string=offset;  // 更新字符串位置为当前偏移量
      n/=10;  // 将数字除以10
      if (offset==0)  // 如果偏移量为0
          break;  // 退出循环
      offset--;  // 减小偏移量
  }
  {
      int size_display_string = 19-pos_string;  // 计算显示字符串的长度
      while (size_char > size_display_string)  // 当字符大小大于显示字符串长度时
      {
          size_char--;  // 减小字符大小
          printf(" ");  // 输出空格
      }
  }

  printf("%s",&number[pos_string]);  // 输出字符串
}

static int do_list(uf)  // 定义静态函数 do_list
    unzFile uf;  // 参数为 unzFile 类型
{
    uLong i;  // 定义无符号长整型变量 i
    unz_global_info64 gi;  // 定义 unz_global_info64 结构体变量 gi
    int err;  // 定义整型变量 err

    err = unzGetGlobalInfo64(uf,&gi);  // 调用 unzGetGlobalInfo64 函数获取全局信息
    if (err!=UNZ_OK)  // 如果返回值不等于 UNZ_OK
        printf("error %d with zipfile in unzGetGlobalInfo \n",err);  // 输出错误信息
    printf("  Length  Method     Size Ratio   Date    Time   CRC-32     Name\n");  // 输出表头信息
    printf("  ------  ------     ---- -----   ----    ----   ------     ----\n");  // 输出分隔线
    for (i=0;i<gi.number_entry;i++)  // 遍历文件条目数
    }

    return 0;  // 返回0
}


static int do_extract_currentfile(uf,popt_extract_without_path,popt_overwrite,password)  // 定义静态函数 do_extract_currentfile
    unzFile uf;  // 参数为 unzFile 类型
    const int* popt_extract_without_path;  // 参数为指向常量整型的指针
    int* popt_overwrite;  // 参数为整型指针
    const char* password;  // 参数为指向常量字符的指针
{
    char filename_inzip[256];  // 创建长度为256的字符数组
    char* filename_withoutpath;  // 创建指向字符的指针
    char* p;  // 创建字符指针 p
    int err=UNZ_OK;  // 定义整型变量 err 并初始化为 UNZ_OK
    FILE *fout=NULL;  // 创建文件指针 fout 并初始化为 NULL
    void* buf;  // 创建指向 void 的指针 buf
    uInt size_buf;  // 定义无符号整型变量 size_buf

    unz_file_info64 file_info;  // 定义 unz_file_info64 结构体变量 file_info
    err = unzGetCurrentFileInfo64(uf,&file_info,filename_inzip,sizeof(filename_inzip),NULL,0,NULL,0);  // 调用 unzGetCurrentFileInfo64 函数获取当前文件信息

    if (err!=UNZ_OK)  // 如果返回值不等于 UNZ_OK
    {
        printf("error %d with zipfile in unzGetCurrentFileInfo\n",err);  // 输出错误信息
        return err;  // 返回错误值
    }

    size_buf = WRITEBUFFERSIZE;  // 设置缓冲区大小
    buf = (void*)malloc(size_buf);  // 分配内存给 buf
    if (buf==NULL)  // 如果分配失败
    {
        printf("Error allocating memory\n");  // 输出错误信息
        return UNZ_INTERNALERROR;  // 返回内部错误
    }

    p = filename_withoutpath = filename_inzip;  // 初始化 p 和 filename_withoutpath
    while ((*p) != '\0')  // 当 p 指向的字符不是结束符时
    {
        if (((*p)=='/') || ((*p)=='\\'))  // 如果字符是斜杠或反斜杠
            filename_withoutpath = p+1;  // 更新 filename_withoutpath
        p++;  // 指针后移
    }

    if ((*filename_withoutpath)=='\0')  // 如果 filename_withoutpath 指向的字符是结束符
    {
        // 如果提取时不保留文件路径
        if ((*popt_extract_without_path)==0)
        {
            // 打印创建目录的信息
            printf("creating directory: %s\n",filename_inzip);
            // 创建目录
            mymkdir(filename_inzip);
        }
    }
    else
    }

    // 释放缓冲区内存
    free(buf);
    // 返回错误码
    return err;
# 定义一个静态函数，用于从 ZIP 文件中提取文件
static int do_extract(uf,opt_extract_without_path,opt_overwrite,password)
    unzFile uf;  # 定义一个unzFile类型的变量uf，用于表示ZIP文件
    int opt_extract_without_path;  # 表示是否提取文件时去除路径
    int opt_overwrite;  # 表示是否覆盖已存在的文件
    const char* password;  # 表示ZIP文件的密码
{
    uLong i;  # 定义一个uLong类型的变量i，用于循环计数
    unz_global_info64 gi;  # 定义一个unz_global_info64类型的变量gi，用于存储ZIP文件的全局信息
    int err;  # 定义一个int类型的变量err，用于存储错误码

    # 获取ZIP文件的全局信息
    err = unzGetGlobalInfo64(uf,&gi);
    if (err!=UNZ_OK)
        printf("error %d with zipfile in unzGetGlobalInfo \n",err);  # 如果获取全局信息失败，则打印错误信息

    # 遍历ZIP文件中的每个文件
    for (i=0;i<gi.number_entry;i++)
    {
        # 提取当前文件
        if (do_extract_currentfile(uf,&opt_extract_without_path,
                                      &opt_overwrite,
                                      password) != UNZ_OK)
            break;  # 如果提取文件失败，则跳出循环

        # 如果不是最后一个文件，则继续提取下一个文件
        if ((i+1)<gi.number_entry)
        {
            err = unzGoToNextFile(uf);
            if (err!=UNZ_OK)
            {
                printf("error %d with zipfile in unzGoToNextFile\n",err);  # 如果跳转到下一个文件失败，则打印错误信息
                break;  # 跳出循环
            }
        }
    }

    return 0;  # 返回0表示提取文件操作成功
}

# 定义一个静态函数，用于从ZIP文件中提取单个文件
static int do_extract_onefile(uf,filename,opt_extract_without_path,opt_overwrite,password)
    unzFile uf;  # 定义一个unzFile类型的变量uf，用于表示ZIP文件
    const char* filename;  # 要提取的文件名
    int opt_extract_without_path;  # 表示是否提取文件时去除路径
    int opt_overwrite;  # 表示是否覆盖已存在的文件
    const char* password;  # 表示ZIP文件的密码
{
    # 如果在ZIP文件中找不到指定的文件，则打印错误信息并返回2
    if (unzLocateFile(uf,filename,CASESENSITIVITY)!=UNZ_OK)
    {
        printf("file %s not found in the zipfile\n",filename);
        return 2;
    }

    # 提取当前文件，如果成功则返回0，否则返回1
    if (do_extract_currentfile(uf,&opt_extract_without_path,
                                      &opt_overwrite,
                                      password) == UNZ_OK)
        return 0;
    else
        return 1;
}

# 主函数
int main(argc,argv)
    int argc;  # 表示命令行参数的个数
    char *argv[];  # 表示命令行参数的数组
{
    const char *zipfilename=NULL;  # 定义一个指向ZIP文件名的指针
    const char *filename_to_extract=NULL;  # 定义一个指向要提取文件名的指针
    const char *password=NULL;  # 定义一个指向ZIP文件密码的指针
    char filename_try[MAXFILENAME+16] = "";  # 定义一个用于存储文件名的字符数组
    int i;  # 定义一个循环计数变量
    int ret_value=0;  # 定义一个返回值变量，初始值为0
    int opt_do_list=0;  # 表示是否执行列出文件操作
    int opt_do_extract=1;  # 表示是否执行提取文件操作
    int opt_do_extract_withoutpath=0;  # 表示是否提取文件时去除路径
    int opt_overwrite=0;  # 表示是否覆盖已存在的文件
    int opt_extractdir=0;  # 表示是否提取到指定目录
    const char *dirname=NULL;  # 定义一个指向目录名的指针
    unzFile uf=NULL;  # 定义一个unzFile类型的变量uf，用于表示ZIP文件

    do_banner();  # 调用打印欢迎信息的函数
    if (argc==1)  # 如果命令行参数个数为1
    {
        do_help();  # 调用打印帮助信息的函数
        return 0;  # 返回0
    }
    else
    {
        // 遍历命令行参数，从第二个参数开始
        for (i=1;i<argc;i++)
        {
            // 判断参数是否以 '-' 开头
            if ((*argv[i])=='-')
            {
                // 获取参数值的指针
                const char *p=argv[i]+1;
    
                // 遍历参数值
                while ((*p)!='\0')
                {
                    // 获取当前字符
                    char c=*(p++);
                    // 判断是否包含 'l' 或 'L'，设置相应的选项
                    if ((c=='l') || (c=='L'))
                        opt_do_list = 1;
                    // 判断是否包含 'v' 或 'V'，设置相应的选项
                    if ((c=='v') || (c=='V'))
                        opt_do_list = 1;
                    // 判断是否包含 'x' 或 'X'，设置相应的选项
                    if ((c=='x') || (c=='X'))
                        opt_do_extract = 1;
                    // 判断是否包含 'e' 或 'E'，设置相应的选项
                    if ((c=='e') || (c=='E'))
                        opt_do_extract = opt_do_extract_withoutpath = 1;
                    // 判断是否包含 'o' 或 'O'，设置相应的选项
                    if ((c=='o') || (c=='O'))
                        opt_overwrite=1;
                    // 判断是否包含 'd' 或 'D'，设置相应的选项，并获取下一个参数作为目录名
                    if ((c=='d') || (c=='D'))
                    {
                        opt_extractdir=1;
                        dirname=argv[i+1];
                    }
    
                    // 判断是否包含 'p' 或 'P'，并且后面还有参数，获取下一个参数作为密码
                    if (((c=='p') || (c=='P')) && (i+1<argc))
                    {
                        password=argv[i+1];
                        i++;
                    }
                }
            }
            else
            {
                // 如果 zipfilename 为空，则将当前参数作为 zip 文件名
                if (zipfilename == NULL)
                    zipfilename = argv[i];
                // 如果 filename_to_extract 为空，并且不需要提取到指定目录，则将当前参数作为要提取的文件名
                else if ((filename_to_extract==NULL) && (!opt_extractdir))
                    filename_to_extract = argv[i] ;
            }
        }
    }
    
    // 如果 zipfilename 不为空，则执行以下代码
    if (zipfilename!=NULL)
    {
# ifdef USEWIN32IOAPI
# 定义一个名为ffunc的zlib_filefunc64_def结构体
        zlib_filefunc64_def ffunc;
# endif

# 复制zipfilename到filename_try，最多复制MAXFILENAME-1个字符
        strncpy(filename_try, zipfilename,MAXFILENAME-1)
        /* 如果字符串太长，strncpy不会追加结尾的NULL字符。 */
        filename_try[ MAXFILENAME ] = '\0';

# ifdef USEWIN32IOAPI
# 使用fill_win32_filefunc64A函数填充ffunc结构体
        fill_win32_filefunc64A(&ffunc);
# 根据USEWIN32IOAPI宏的定义，使用不同的方式打开ZIP文件
        uf = unzOpen2_64(zipfilename,&ffunc);
# else
# 使用unzOpen64函数打开ZIP文件
        uf = unzOpen64(zipfilename);
# endif
# 如果打开失败，尝试在文件名后面加上".zip"再次打开
        if (uf==NULL)
        {
            strcat(filename_try,".zip");
# ifdef USEWIN32IOAPI
            uf = unzOpen2_64(filename_try,&ffunc);
# else
            uf = unzOpen64(filename_try);
# endif
        }
    }

# 如果打开失败，输出错误信息并返回1
    if (uf==NULL)
    {
        printf("Cannot open %s or %s.zip\n",zipfilename,zipfilename);
        return 1;
    }
# 打开成功，输出打开的文件名
    printf("%s opened\n",filename_try);

# 如果opt_do_list为1，则执行do_list函数
    if (opt_do_list==1)
        ret_value = do_list(uf);
# 如果opt_do_extract为1
    else if (opt_do_extract==1)
    {
# ifdef _WIN32
# 如果opt_extractdir为真，并且_chdir(dirname)成功
        if (opt_extractdir && _chdir(dirname))
# else
# 如果opt_extractdir为真，并且chdir(dirname)成功
        if (opt_extractdir && chdir(dirname))
# endif
        {
          printf("Error changing into %s, aborting\n", dirname);
          exit(-1);
        }

# 如果filename_to_extract为NULL，则执行do_extract函数，否则执行do_extract_onefile函数
        if (filename_to_extract == NULL)
            ret_value = do_extract(uf, opt_do_extract_withoutpath, opt_overwrite, password);
        else
            ret_value = do_extract_onefile(uf, filename_to_extract, opt_do_extract_withoutpath, opt_overwrite, password);
    }

# 关闭ZIP文件
    unzClose(uf);

# 返回ret_value
    return ret_value;
}
```