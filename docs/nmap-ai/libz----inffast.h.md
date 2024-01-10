# `nmap\libz\inffast.h`

```
# inffast.h -- 用于包含 inffast.c 的头文件
# 版权所有 1995-2003, 2010 Mark Adler
# 发行和使用条件，请参见 zlib.h 中的版权声明

# 警告：此文件不应该被应用程序使用。它是压缩库实现的一部分，可能会发生变化。应用程序应该只使用 zlib.h。

# 定义了一个名为 inflate_fast 的函数，用于快速解压缩数据
void ZLIB_INTERNAL inflate_fast OF((z_streamp strm, unsigned start));
```