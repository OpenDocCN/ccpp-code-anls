# `nmap\libz\contrib\untgz\untgz.c`

```cpp
/*
 * untgz.c -- Display contents and extract files from a gzip'd TAR file
 * 从 gzip'd TAR 文件中显示内容并提取文件
 *
 * written by Pedro A. Aranda Gutierrez <paag@tid.es>
 * adaptation to Unix by Jean-loup Gailly <jloup@gzip.org>
 * various fixes by Cosmin Truta <cosmint@cs.ubbcluj.ro>
 * 由 Pedro A. Aranda Gutierrez 编写，适配到 Unix 由 Jean-loup Gailly 完成，由 Cosmin Truta 进行各种修复
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>

#include "zlib.h"

#ifdef unix
#  include <unistd.h>
#else
#  include <direct.h>
#  include <io.h>
#endif

#ifdef WIN32
#include <windows.h>
#  ifndef F_OK
#    define F_OK  0
#  endif
#  define mkdir(dirname,mode)   _mkdir(dirname)
#  ifdef _MSC_VER
#    define access(path,mode)   _access(path,mode)
#    define chmod(path,mode)    _chmod(path,mode)
#    define strdup(str)         _strdup(str)
#  endif
#else
#  include <utime.h>
#endif

/* values used in typeflag field */
/* typeflag 字段中使用的值 */

#define REGTYPE  '0'            /* regular file */
#define AREGTYPE '\0'           /* regular file */
#define LNKTYPE  '1'            /* link */
#define SYMTYPE  '2'            /* reserved */
#define CHRTYPE  '3'            /* character special */
#define BLKTYPE  '4'            /* block special */
#define DIRTYPE  '5'            /* directory */
#define FIFOTYPE '6'            /* FIFO special */
#define CONTTYPE '7'            /* reserved */

/* GNU tar extensions */
/* GNU tar 扩展 */

#define GNUTYPE_DUMPDIR  'D'    /* file names from dumped directory */
#define GNUTYPE_LONGLINK 'K'    /* long link name */
#define GNUTYPE_LONGNAME 'L'    /* long file name */
#define GNUTYPE_MULTIVOL 'M'    /* continuation of file from another volume */
#define GNUTYPE_NAMES    'N'    /* file name that does not fit into main hdr */
#define GNUTYPE_SPARSE   'S'    /* sparse file */
#define GNUTYPE_VOLHDR   'V'    /* tape/volume header */

/* tar header */
/* tar 头部 */

#define BLOCKSIZE     512
#define SHORTNAMESIZE 100

struct tar_header
{                               /* byte offset */
  char name[100];               /*   0 */
  char mode[8];                 /* 100 */
  char uid[8];                  /* 108 */
  char gid[8];                  /* 116 */
  char size[12];                /* 124 */
  char mtime[12];               /* 136 */
  char chksum[8];               /* 148 */
  char typeflag;                /* 156 */
  char linkname[100];           /* 157 */
  char magic[6];                /* 257 */
  char version[2];              /* 263 */
  char uname[32];               /* 265 */
  char gname[32];               /* 297 */
  char devmajor[8];             /* 329 */
  char devminor[8];             /* 337 */
  char prefix[155];             /* 345 */
                                /* 500 */
};

union tar_buffer
{
  char               buffer[BLOCKSIZE];
  struct tar_header  header;
};

struct attr_item
{
  struct attr_item  *next;      /* pointer to the next attribute item */
  char              *fname;      /* file name */
  int                mode;       /* file mode */
  time_t             time;       /* file modification time */
};

enum { TGZ_EXTRACT, TGZ_LIST, TGZ_INVALID };  /* enumeration for different operations on TGZ files */

char *TGZfname          OF((const char *));    /* function to return the file name of the TGZ archive */
void TGZnotfound        OF((const char *));    /* function to handle case when TGZ archive is not found */

int getoct              OF((char *, int));     /* function to convert octal string to integer */
char *strtime           OF((time_t *));        /* function to convert time_t to string */
int setfiletime         OF((char *, time_t));  /* function to set file modification time */
void push_attr          OF((struct attr_item **, char *, int, time_t));  /* function to push attribute item to the list */
void restore_attr       OF((struct attr_item **));  /* function to restore attributes from the list */

int ExprMatch           OF((char *, char *));   /* function to match regular expression */

int makedir             OF((char *));           /* function to create directory */
int matchname           OF((int, int, char **, char *));  /* function to match file name */

void error              OF((const char *));     /* function to handle errors */
int tar                 OF((gzFile, int, int, int, char **));  /* function to handle tar operations */

void help               OF((int));               /* function to display help */
int main                OF((int, char **));     /* main function */

char *prog;                                     /* pointer to program name */

const char *TGZsuffix[] = { "\0", ".tar", ".tar.gz", ".taz", ".tgz", NULL };  /* array of TGZ suffixes */

/* return the file name of the TGZ archive */
/* or NULL if it does not exist */

char *TGZfname (const char *arcname)           /* function to return the file name of the TGZ archive */
{
  // 定义一个静态字符数组，用于存储文件名
  static char buffer[1024];
  // 定义变量 origlen 和 i
  int origlen,i;

  // 将 arcname 复制到 buffer 中
  strcpy(buffer,arcname);
  // 获取 buffer 的长度
  origlen = strlen(buffer);

  // 遍历 TGZsuffix 数组
  for (i=0; TGZsuffix[i]; i++)
    {
       // 将 TGZsuffix[i] 连接到 buffer 的末尾
       strcpy(buffer+origlen,TGZsuffix[i]);
       // 如果文件存在，则返回文件名
       if (access(buffer,F_OK) == 0)
         return buffer;
    }
  // 如果文件不存在，则返回 NULL
  return NULL;
}


/* error message for the filename */

// 定义一个函数，用于输出文件名的错误信息
void TGZnotfound (const char *arcname)
{
  int i;

  // 输出错误信息到 stderr
  fprintf(stderr,"%s: Couldn't find ",prog);
  // 遍历 TGZsuffix 数组，输出错误信息
  for (i=0;TGZsuffix[i];i++)
    fprintf(stderr,(TGZsuffix[i+1]) ? "%s%s, " : "or %s%s\n",
            arcname,
            TGZsuffix[i]);
  // 退出程序
  exit(1);
}


/* convert octal digits to int */
/* on error return -1 */

// 定义一个函数，将八进制数字转换为整数
int getoct (char *p,int width)
{
  int result = 0;
  char c;

  // 遍历字符串，将八进制数字转换为整数
  while (width--)
    {
      c = *p++;
      if (c == 0)
        break;
      if (c == ' ')
        continue;
      if (c < '0' || c > '7')
        return -1;
      result = result * 8 + (c - '0');
    }
  return result;
}


/* convert time_t to string */
/* use the "YYYY/MM/DD hh:mm:ss" format */

// 定义一个函数，将 time_t 转换为字符串
char *strtime (time_t *t)
{
  struct tm   *local;
  static char result[32];

  // 获取本地时间
  local = localtime(t);
  // 格式化时间为 "YYYY/MM/DD hh:mm:ss" 格式
  sprintf(result,"%4d/%02d/%02d %02d:%02d:%02d",
          local->tm_year+1900, local->tm_mon+1, local->tm_mday,
          local->tm_hour, local->tm_min, local->tm_sec);
  return result;
}


/* set file time */

// 定义一个函数，用于设置文件的时间
int setfiletime (char *fname,time_t ftime)
{
#ifdef WIN32
  static int isWinNT = -1;
  SYSTEMTIME st;
  FILETIME locft, modft;
  struct tm *loctm;
  HANDLE hFile;
  int result;

  // 获取本地时间
  loctm = localtime(&ftime);
  if (loctm == NULL)
    return -1;

  // 设置 SYSTEMTIME 结构体的值
  st.wYear         = (WORD)loctm->tm_year + 1900;
  st.wMonth        = (WORD)loctm->tm_mon + 1;
  st.wDayOfWeek    = (WORD)loctm->tm_wday;
  st.wDay          = (WORD)loctm->tm_mday;
  st.wHour         = (WORD)loctm->tm_hour;
  st.wMinute       = (WORD)loctm->tm_min;
  st.wSecond       = (WORD)loctm->tm_sec;
  st.wMilliseconds = 0;
  // 将 SYSTEMTIME 转换为 FILETIME
  if (!SystemTimeToFileTime(&st, &locft) ||
      !LocalFileTimeToFileTime(&locft, &modft))
    return -1;

  if (isWinNT < 0)
    # 判断操作系统是否为 Windows NT
    isWinNT = (GetVersion() < 0x80000000) ? 1 : 0;
    # 根据文件名创建或打开文件，设置文件的写入权限
    hFile = CreateFile(fname, GENERIC_WRITE, 0, NULL, OPEN_EXISTING,
                       (isWinNT ? FILE_FLAG_BACKUP_SEMANTICS : 0),
                       NULL);
    # 如果文件句柄无效，则返回错误代码
    if (hFile == INVALID_HANDLE_VALUE)
      return -1;
    # 设置文件的修改时间为指定时间
    result = SetFileTime(hFile, NULL, NULL, &modft) ? 0 : -1;
    # 关闭文件句柄
    CloseHandle(hFile);
    # 返回操作结果
    return result;
#else
  // 定义结构体变量 settime
  struct utimbuf settime;

  // 设置文件的访问时间和修改时间为 ftime
  settime.actime = settime.modtime = ftime;
  // 调用 utime 函数设置文件的访问时间和修改时间
  return utime(fname,&settime);
#endif
}


/* push file attributes */

// 将文件属性推入属性列表
void push_attr(struct attr_item **list,char *fname,int mode,time_t time)
{
  // 分配内存给属性项
  struct attr_item *item;
  item = (struct attr_item *)malloc(sizeof(struct attr_item));
  if (item == NULL)
    error("Out of memory");
  // 复制文件名
  item->fname = strdup(fname);
  item->mode  = mode;
  item->time  = time;
  item->next  = *list;
  *list       = item;
}


/* restore file attributes */

// 恢复文件属性
void restore_attr(struct attr_item **list)
{
  struct attr_item *item, *prev;

  // 遍历属性列表
  for (item = *list; item != NULL; )
    {
      // 设置文件时间
      setfiletime(item->fname,item->time);
      // 修改文件权限
      chmod(item->fname,item->mode);
      prev = item;
      item = item->next;
      // 释放内存
      free(prev);
    }
  *list = NULL;
}


/* match regular expression */

// 匹配正则表达式
#define ISSPECIAL(c) (((c) == '*') || ((c) == '/'))

int ExprMatch (char *string,char *expr)
{
  while (1)
    {
      if (ISSPECIAL(*expr))
        {
          if (*expr == '/')
            {
              if (*string != '\\' && *string != '/')
                return 0;
              string ++; expr++;
            }
          else if (*expr == '*')
            {
              if (*expr ++ == 0)
                return 1;
              while (*++string != *expr)
                if (*string == 0)
                  return 0;
            }
        }
      else
        {
          if (*string != *expr)
            return 0;
          if (*expr++ == 0)
            return 1;
          string++;
        }
    }
}


/* recursive mkdir */
/* abort on ENOENT; ignore other errors like "directory already exists" */
/* return 1 if OK */
/*        0 on error */

// 递归创建目录
int makedir (char *newdir)
{
  // 复制目录路径
  char *buffer = strdup(newdir);
  char *p;
  int  len = strlen(buffer);

  // 如果目录路径长度小于等于0，释放内存并返回0
  if (len <= 0) {
    free(buffer);
    return 0;
  }
  // 如果目录路径最后一个字符是斜杠，将其替换为结束符
  if (buffer[len-1] == '/') {
    buffer[len-1] = '\0';
  }
  // 创建目录并设置权限
  if (mkdir(buffer, 0755) == 0)
    {
      free(buffer);
      return 1;
    }
    # 将指针 p 指向 buffer 的下一个位置
    p = buffer+1;
    # 进入无限循环
    while (1)
    {
      # 定义变量 hold，用于存储当前指针位置的字符
      char hold;
      # 当指针指向的字符不为空且不是反斜杠或斜杠时，指针向后移动
      while(*p && *p != '\\' && *p != '/')
        p++;
      # 将当前指针位置的字符存储到 hold 中，并将当前指针位置的字符设为 0
      hold = *p;
      *p = 0;
      # 如果创建目录失败且错误码为 ENOENT，则输出错误信息并返回 0
      if ((mkdir(buffer, 0755) == -1) && (errno == ENOENT))
        {
          fprintf(stderr,"%s: Couldn't create directory %s\n",prog,buffer);
          free(buffer);
          return 0;
        }
      # 如果 hold 为 0，则跳出循环
      if (hold == 0)
        break;
      # 恢复当前指针位置的字符，并将指针向后移动一位
      *p++ = hold;
    }
    # 释放 buffer 的内存
    free(buffer);
    # 返回 1
    return 1;
}

int matchname (int arg,int argc,char **argv,char *fname)
{
  // 如果没有给定参数（untgz tgzarchive），返回1
  if (arg == argc)      
    return 1;

  // 遍历参数列表，匹配文件名
  while (arg < argc)
    if (ExprMatch(fname,argv[arg++])
      return 1;

  // 暂时忽略这个返回值
  return 0; 
}


/* tar file list or extract */

int tar (gzFile in,int action,int arg,int argc,char **argv)
{
  // 定义变量
  union  tar_buffer buffer;
  int    len;
  int    err;
  int    getheader = 1;
  int    remaining = 0;
  FILE   *outfile = NULL;
  char   fname[BLOCKSIZE];
  int    tarmode;
  time_t tartime;
  struct attr_item *attributes = NULL;

  // 如果是列出文件列表操作，打印表头
  if (action == TGZ_LIST)
    printf("    date      time     size                       file\n"
           " ---------- -------- --------- -------------------------------------\n");
  while (1)
    }

  /*
   * 恢复文件模式和时间戳
   */
  restore_attr(&attributes);

  // 关闭输入流
  if (gzclose(in) != Z_OK)
    error("failed gzclose");

  return 0;
}


/* ============================================================ */

void help(int exitval)
{
  // 打印版本信息和使用说明
  printf("untgz version 0.2.1\n"
         "  using zlib version %s\n\n",
         zlibVersion());
  printf("Usage: untgz file.tgz            extract all files\n"
         "       untgz file.tgz fname ...  extract selected files\n"
         "       untgz -l file.tgz         list archive contents\n"
         "       untgz -h                  display this help\n");
  exit(exitval);
}

void error(const char *msg)
{
  // 打印错误信息并退出程序
  fprintf(stderr, "%s: %s\n", prog, msg);
  exit(1);
}


/* ============================================================ */

#if defined(WIN32) && defined(__GNUC__)
int _CRT_glob = 0;      /* disable argument globbing in MinGW */
#endif

int main(int argc,char **argv)
{
    // 定义变量
    int         action = TGZ_EXTRACT;
    int         arg = 1;
    char        *TGZfile;
    gzFile      *f;

    // 获取程序名
    prog = strrchr(argv[0],'\\');
    # 如果程序名为空
    if (prog == NULL)
      {
        # 从参数中获取程序名，找到最后一个 '/' 的位置
        prog = strrchr(argv[0],'/');
        # 如果找不到 '/'，则找最后一个 ':' 的位置
        if (prog == NULL)
          {
            prog = strrchr(argv[0],':');
            # 如果找不到 ':'，则程序名为参数中的第一个字符串
            if (prog == NULL)
              prog = argv[0];
            else
              # 否则程序名为 ':' 后面的字符串
              prog++;
          }
        else
          # 否则程序名为 '/' 后面的字符串
          prog++;
      }
    else
      # 否则程序名为下一个字符串
      prog++;

    # 如果参数个数为 1，调用帮助函数
    if (argc == 1)
      help(0);

    # 如果参数为 "-l"，设置操作为列出文件
    if (strcmp(argv[arg],"-l") == 0)
      {
        action = TGZ_LIST;
        # 如果参数个数为下一个字符串，调用帮助函数
        if (argc == ++arg)
          help(0);
      }
    # 如果参数为 "-h"，调用帮助函数
    else if (strcmp(argv[arg],"-h") == 0)
      {
        help(0);
      }

    # 获取 TGZ 文件名
    if ((TGZfile = TGZfname(argv[arg])) == NULL)
      TGZnotfound(argv[arg]);

    # 下一个参数
    ++arg;
    # 如果操作为列出文件且参数不等于参数个数，调用帮助函数
    if ((action == TGZ_LIST) && (arg != argc))
      help(1);
/*
 *  Process the TGZ file
 */
    // 根据不同的操作类型执行不同的操作
    switch(action)
      {
      // 如果是列出操作或者解压操作
      case TGZ_LIST:
      case TGZ_EXTRACT:
        // 打开 TGZ 文件
        f = gzopen(TGZfile,"rb");
        // 如果打开失败，输出错误信息并返回 1
        if (f == NULL)
          {
            fprintf(stderr,"%s: Couldn't gzopen %s\n",prog,TGZfile);
            return 1;
          }
        // 调用 tar 函数执行操作，并返回结果
        exit(tar(f, action, arg, argc, argv));
      break;

      // 如果是其他操作类型，输出错误信息并返回 1
      default:
        error("Unknown option");
        exit(1);
      }

    // 返回 0
    return 0;
}
```