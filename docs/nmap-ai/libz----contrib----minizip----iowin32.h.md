# `nmap\libz\contrib\minizip\iowin32.h`

```cpp
/*
    iowin32.h -- 用于压缩/解压缩 .zip 文件的 IO 基本函数头文件
    版本 1.1, 2010年2月14日
    MiniZip 项目的一部分 - (http://www.winimage.com/zLibDll/minizip.html)

    版权所有 (C) 1998-2010 Gilles Vollant (minizip) (http://www.winimage.com/zLibDll/minizip.html)

    Zip64 支持的修改
    版权所有 (C) 2009-2010 Mathias Svensson (http://result42.com)

    有关更多信息，请阅读 MiniZip_info.txt
*/

#include <windows.h>

#ifdef __cplusplus
extern "C" {
#endif

// 填充 zlib_filefunc_def 结构体的 win32 文件函数
void fill_win32_filefunc OF((zlib_filefunc_def* pzlib_filefunc_def));

// 填充 zlib_filefunc64_def 结构体的 win32 文件函数
void fill_win32_filefunc64 OF((zlib_filefunc64_def* pzlib_filefunc_def));

// 填充 zlib_filefunc64_def 结构体的 win32 文件函数（ASCII 版本）
void fill_win32_filefunc64A OF((zlib_filefunc64_def* pzlib_filefunc_def));

// 填充 zlib_filefunc64_def 结构体的 win32 文件函数（宽字符版本）
void fill_win32_filefunc64W OF((zlib_filefunc64_def* pzlib_filefunc_def));

#ifdef __cplusplus
}
#endif
```