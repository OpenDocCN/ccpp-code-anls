# `nmap\libz\contrib\minizip\mztools.h`

```
/*
  为Minizip提供额外的工具
  作者：Xavier Roche '2004
  许可证：与ZLIB相同 (www.gzip.org)
*/

#ifndef _zip_tools_H
#define _zip_tools_H

#ifdef __cplusplus
extern "C" {
#endif

#ifndef _ZLIB_H
#include "zlib.h"
#endif

#include "unzip.h"

/* 修复一个ZIP文件（缺少中央目录）
   file: 要恢复的文件
   fileOut: 恢复后的输出文件
   fileOutTmp: 用于恢复的临时文件名
*/
extern int ZEXPORT unzRepair(const char* file,
                             const char* fileOut,
                             const char* fileOutTmp,
                             uLong* nRecovered,
                             uLong* bytesRecovered);


#ifdef __cplusplus
}
#endif


#endif
```