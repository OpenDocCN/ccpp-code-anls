# `nmap\libz\compress.c`

```cpp
/* compress.c -- compress a memory buffer
 * Copyright (C) 1995-2005, 2014, 2016 Jean-loup Gailly, Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

/* @(#) $Id$ */

#define ZLIB_INTERNAL
#include "zlib.h"

/* ===========================================================================
     Compresses the source buffer into the destination buffer. The level
   parameter has the same meaning as in deflateInit.  sourceLen is the byte
   length of the source buffer. Upon entry, destLen is the total size of the
   destination buffer, which must be at least 0.1% larger than sourceLen plus
   12 bytes. Upon exit, destLen is the actual size of the compressed buffer.

     compress2 returns Z_OK if success, Z_MEM_ERROR if there was not enough
   memory, Z_BUF_ERROR if there was not enough room in the output buffer,
   Z_STREAM_ERROR if the level parameter is invalid.
*/
int ZEXPORT compress2(dest, destLen, source, sourceLen, level)
    Bytef *dest;
    uLongf *destLen;
    const Bytef *source;
    uLong sourceLen;
    int level;
{
    z_stream stream;  /* 初始化z_stream结构体 */
    int err;  /* 定义错误码 */
    const uInt max = (uInt)-1;  /* 定义最大值 */
    uLong left;  /* 定义剩余空间大小 */

    left = *destLen;  /* 将destLen的值赋给left */
    *destLen = 0;  /* 将destLen的值设为0 */

    stream.zalloc = (alloc_func)0;  /* 分配函数设为0 */
    stream.zfree = (free_func)0;  /* 释放函数设为0 */
    stream.opaque = (voidpf)0;  /* 透明指针设为0 */

    err = deflateInit(&stream, level);  /* 初始化压缩流 */
    if (err != Z_OK) return err;  /* 如果初始化失败，返回错误码 */

    stream.next_out = dest;  /* 设置输出缓冲区 */
    stream.avail_out = 0;  /* 设置输出缓冲区可用大小为0 */
    stream.next_in = (z_const Bytef *)source;  /* 设置输入缓冲区 */
    stream.avail_in = 0;  /* 设置输入缓冲区可用大小为0 */

    do {
        if (stream.avail_out == 0) {
            stream.avail_out = left > (uLong)max ? max : (uInt)left;  /* 计算输出缓冲区可用大小 */
            left -= stream.avail_out;  /* 更新剩余空间大小 */
        }
        if (stream.avail_in == 0) {
            stream.avail_in = sourceLen > (uLong)max ? max : (uInt)sourceLen;  /* 计算输入缓冲区可用大小 */
            sourceLen -= stream.avail_in;  /* 更新剩余输入数据大小 */
        }
        err = deflate(&stream, sourceLen ? Z_NO_FLUSH : Z_FINISH);  /* 压缩数据 */
    } while (err == Z_OK);  /* 循环直到压缩完成 */

    *destLen = stream.total_out;  /* 将实际压缩后的数据大小赋给destLen */
    deflateEnd(&stream);  /* 结束压缩流 */
}
    # 如果 err 等于 Z_STREAM_END，则返回 Z_OK，否则返回 err
    return err == Z_STREAM_END ? Z_OK : err;
# 压缩函数，将源数据压缩到目标数据中
int ZEXPORT compress(dest, destLen, source, sourceLen)
    Bytef *dest;  # 目标数据的指针
    uLongf *destLen;  # 目标数据的长度
    const Bytef *source;  # 源数据的指针
    uLong sourceLen;  # 源数据的长度
{
    # 调用 compress2 函数，使用默认压缩级别进行压缩
    return compress2(dest, destLen, source, sourceLen, Z_DEFAULT_COMPRESSION);
}

# 计算使用默认压缩级别压缩后的数据长度的上限
uLong ZEXPORT compressBound(sourceLen)
    uLong sourceLen;  # 源数据的长度
{
    # 根据源数据长度计算压缩后的数据长度的上限
    return sourceLen + (sourceLen >> 12) + (sourceLen >> 14) +
           (sourceLen >> 25) + 13;
}
```