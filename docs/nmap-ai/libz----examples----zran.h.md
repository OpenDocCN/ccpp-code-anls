# `nmap\libz\examples\zran.h`

```
/* zran.h -- example of zlib/gzip stream indexing and random access
 * zlib/gzip流索引和随机访问的示例
 * 版权所有 (C) 2005, 2012, 2018 Mark Adler
 * 分发和使用条件，请参阅zlib.h中的版权声明
 * 版本 1.2  2018年10月14日  Mark Adler */

#include <stdio.h>
#include "zlib.h"

/* 访问点列表。*/
struct deflate_index {
    int have;           /* 列表条目数 */
    int gzip;           /* 如果索引是gzip文件，则为1，如果是zlib流，则为0 */
    off_t length;       /* 未压缩数据的总长度 */
    void *list;         /* 分配的条目列表 */
};

/* 通过zlib或gzip压缩流进行一次完整遍历，并构建一个索引，其中包含每个未压缩输出的span字节的访问点。
   具有多个成员的gzip文件将被完全索引。span应该被选择为在随机访问速度和列表的内存需求之间取得平衡，大约每个访问点32K字节。
   返回值是成功时的访问点数（>= 1），内存不足时为Z_MEM_ERROR，输入文件错误时为Z_DATA_ERROR，文件读取错误时为Z_ERRNO。
   成功时，*built指向生成的索引。 */
int deflate_index_build(FILE *in, off_t span, struct deflate_index **built);

/* 释放由deflate_index_build()构建的索引 */
void deflate_index_free(struct deflate_index *index);

/* 使用索引从偏移量读取len字节到buf中。返回读取的字节数或错误（Z_DATA_ERROR或Z_MEM_ERROR）。
   如果请求的数据超出未压缩数据的末尾，则deflate_index_extract()将返回小于len的值，指示实际读取到buf中的量。
   除非在生成索引时文件已被修改，否则此函数不应返回数据错误，因为deflate_index_build()验证了所有输入。
   如果读取或寻找输入文件时出现错误，则deflate_index_extract()将返回Z_ERRNO。 */
# 从输入文件中提取压缩索引数据，并将结果存储在结构体中
int deflate_index_extract(FILE *in, struct deflate_index *index, off_t offset,
                          unsigned char *buf, int len);
```