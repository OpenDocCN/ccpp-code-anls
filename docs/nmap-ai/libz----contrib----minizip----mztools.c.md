# `nmap\libz\contrib\minizip\mztools.c`

```
/*
  Additional tools for Minizip
  Code: Xavier Roche '2004
  License: Same as ZLIB (www.gzip.org)
*/

/* Code */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "zlib.h"
#include "unzip.h"

#define READ_8(adr)  ((unsigned char)*(adr))  // 读取地址adr处的8位数据
#define READ_16(adr) ( READ_8(adr) | (READ_8(adr+1) << 8) )  // 读取地址adr处的16位数据
#define READ_32(adr) ( READ_16(adr) | (READ_16((adr)+2) << 16) )  // 读取地址adr处的32位数据

#define WRITE_8(buff, n) do { \
  *((unsigned char*)(buff)) = (unsigned char) ((n) & 0xff); \
} while(0)  // 将n写入buff地址处的8位数据
#define WRITE_16(buff, n) do { \
  WRITE_8((unsigned char*)(buff), n); \
  WRITE_8(((unsigned char*)(buff)) + 1, (n) >> 8); \
} while(0)  // 将n写入buff地址处的16位数据
#define WRITE_32(buff, n) do { \
  WRITE_16((unsigned char*)(buff), (n) & 0xffff); \
  WRITE_16((unsigned char*)(buff) + 2, (n) >> 16); \
} while(0)  // 将n写入buff地址处的32位数据

extern int ZEXPORT unzRepair(file, fileOut, fileOutTmp, nRecovered, bytesRecovered)
const char* file;
const char* fileOut;
const char* fileOutTmp;
uLong* nRecovered;
uLong* bytesRecovered;
{
  int err = Z_OK;  // 初始化错误码为Z_OK
  FILE* fpZip = fopen(file, "rb");  // 以只读方式打开file文件
  FILE* fpOut = fopen(fileOut, "wb");  // 以写入方式打开fileOut文件
  FILE* fpOutCD = fopen(fileOutTmp, "wb");  // 以写入方式打开fileOutTmp文件
  if (fpZip != NULL &&  fpOut != NULL) {  // 如果文件打开成功
    int entries = 0;  // 初始化文件条目数为0
    uLong totalBytes = 0;  // 初始化总字节数为0
    char header[30];  // 定义长度为30的字符数组header
    char filename[1024];  // 定义长度为1024的字符数组filename
    char extra[1024];  // 定义长度为1024的字符数组extra
    int offset = 0;  // 初始化偏移量为0
    int offsetCD = 0;  // 初始化中央目录偏移量为0
    }

    /* Final central directory  */
    {
      int entriesZip = entries;  // 将参数 entries 的值赋给 entriesZip
      char header[22];  // 创建一个长度为 22 的字符数组 header
      char* comment = ""; // "ZIP File recovered by zlib/minizip/mztools";  // 创建一个指向空字符串的指针 comment
      int comsize = (int) strlen(comment);  // 计算 comment 字符串的长度，并赋给 comsize
      if (entriesZip > 0xffff) {  // 如果 entriesZip 大于 0xffff
        entriesZip = 0xffff;  // 将 entriesZip 的值设为 0xffff
      }
      WRITE_32(header, 0x06054b50);  // 将 0x06054b50 写入 header 的前 4 个字节
      WRITE_16(header + 4, 0);    /* disk # */  // 将 0 写入 header 的第 5、6 个字节
      WRITE_16(header + 6, 0);    /* disk # */  // 将 0 写入 header 的第 7、8 个字节
      WRITE_16(header + 8, entriesZip);   /* hack */  // 将 entriesZip 写入 header 的第 9、10 个字节
      WRITE_16(header + 10, entriesZip);  /* hack */  // 将 entriesZip 写入 header 的第 11、12 个字节
      WRITE_32(header + 12, offsetCD);    /* size of CD */  // 将 offsetCD 写入 header 的第 13-16 个字节
      WRITE_32(header + 16, offset);      /* offset to CD */  // 将 offset 写入 header 的第 17-20 个字节
      WRITE_16(header + 20, comsize);     /* comment */  // 将 comsize 写入 header 的第 21、22 个字节

      /* Header */
      if (fwrite(header, 1, 22, fpOutCD) == 22) {  // 如果成功将 header 写入 fpOutCD
        /* Comment field */
        if (comsize > 0) {  // 如果 comsize 大于 0
          if ((int)fwrite(comment, 1, comsize, fpOutCD) != comsize) {  // 如果写入 comment 失败
            err = Z_ERRNO;  // 将 Z_ERRNO 赋给 err
          }
        }

      } else {
        err = Z_ERRNO;  // 将 Z_ERRNO 赋给 err
      }
    }

    /* Final merge (file + central directory) */
    fclose(fpOutCD);  // 关闭 fpOutCD
    if (err == Z_OK) {  // 如果 err 等于 Z_OK
      fpOutCD = fopen(fileOutTmp, "rb");  // 以只读方式打开 fileOutTmp 文件，并赋给 fpOutCD
      if (fpOutCD != NULL) {  // 如果 fpOutCD 不为空
        int nRead;  // 声明整型变量 nRead
        char buffer[8192];  // 创建一个长度为 8192 的字符数组 buffer
        while ( (nRead = (int)fread(buffer, 1, sizeof(buffer), fpOutCD)) > 0) {  // 当成功读取到数据时
          if ((int)fwrite(buffer, 1, nRead, fpOut) != nRead) {  // 如果写入数据失败
            err = Z_ERRNO;  // 将 Z_ERRNO 赋给 err
            break;  // 跳出循环
          }
        }
        fclose(fpOutCD);  // 关闭 fpOutCD
      }
    }

    /* Close */
    fclose(fpZip);  // 关闭 fpZip
    fclose(fpOut);  // 关闭 fpOut

    /* Wipe temporary file */
    (void)remove(fileOutTmp);  // 删除 fileOutTmp 文件

    /* Number of recovered entries */
    if (err == Z_OK) {  // 如果 err 等于 Z_OK
      if (nRecovered != NULL) {  // 如果 nRecovered 不为空
        *nRecovered = entries;  // 将 entries 的值赋给 nRecovered 指向的变量
      }
      if (bytesRecovered != NULL) {  // 如果 bytesRecovered 不为空
        *bytesRecovered = totalBytes;  // 将 totalBytes 的值赋给 bytesRecovered 指向的变量
      }
    }
  } else {
    err = Z_STREAM_ERROR;  // 将 Z_STREAM_ERROR 赋给 err
  }
  return err;  // 返回 err 的值
# 闭合前面的函数定义
```