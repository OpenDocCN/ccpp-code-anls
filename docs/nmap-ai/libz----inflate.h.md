# `nmap\libz\inflate.h`

```cpp
/* inflate.h -- internal inflate state definition
 * 压缩文件头文件，定义内部解压缩状态
 * 版权所有 (C) 1995-2019 Mark Adler
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

/* 警告：应用程序不应该使用这个文件。它是压缩库的一部分，可能会发生变化。应用程序应该只使用 zlib.h。 */

/* 在编译时定义 NO_GZIP，如果要禁用 inflate() 对 gzip 头和尾的解码。当不需要 crc 代码时，可以使用 NO_GZIP 来避免链接。对于共享库，gzip 解码应该保持启用。 */
#ifndef NO_GZIP
#  define GUNZIP
#endif

/* 在 inflate() 调用之间可能的解压缩模式 */
typedef enum {
    HEAD = 16180,   /* i: 等待魔术头部 */
    FLAGS,      /* i: 等待方法和标志 (gzip) */
    TIME,       /* i: 等待修改时间 (gzip) */
    OS,         /* i: 等待额外标志和操作系统 (gzip) */
    EXLEN,      /* i: 等待额外长度 (gzip) */
    EXTRA,      /* i: 等待额外字节 (gzip) */
    NAME,       /* i: 等待文件名结束 (gzip) */
    COMMENT,    /* i: 等待注释结束 (gzip) */
    HCRC,       /* i: 等待头部 crc (gzip) */
    DICTID,     /* i: 等待字典检查值 */
    DICT,       /* 等待调用inflateSetDictionary() */
        TYPE,       /* i: 等待类型位，包括最后一个标志位 */
        TYPEDO,     /* i: 相同，但跳过检查以在新块上退出inflate */
        STORED,     /* i: 等待存储大小（长度和补码） */
        COPY_,      /* i/o: 与下面的COPY相同，但只在第一次进入时 */
        COPY,       /* i/o: 等待输入或输出以复制存储的块 */
        TABLE,      /* i: 等待动态块表长度 */
        LENLENS,    /* i: 等待代码长度代码长度 */
        CODELENS,   /* i: 等待长度/字面值和距离代码长度 */
            LEN_,       /* i: 与下面的LEN相同，但只在第一次进入时 */
            LEN,        /* i: 等待长度/字面值/eob代码 */
            LENEXT,     /* i: 等待长度额外位 */
            DIST,       /* i: 等待距离代码 */
            DISTEXT,    /* i: 等待距离额外位 */
            MATCH,      /* o: 等待输出空间以复制字符串 */
            LIT,        /* o: 等待输出空间以写入字面值 */
    CHECK,      /* i: 等待32位校验值 */
    LENGTH,     /* i: 等待32位长度（gzip） */
    DONE,       /* 完成检查，完成 - 保持在此处直到重置 */
    BAD,        /* 发生数据错误 - 保持在此处直到重置 */
    MEM,        /* 发生inflate()内存错误 - 保持在此处直到重置 */
    SYNC        /* 寻找同步字节以重新启动inflate() */
} inflate_mode;

/*
    State transitions between above modes -

    (most modes can go to BAD or MEM on error -- not shown for clarity)

    Process header:
        HEAD -> (gzip) or (zlib) or (raw)
        (gzip) -> FLAGS -> TIME -> OS -> EXLEN -> EXTRA -> NAME -> COMMENT ->
                  HCRC -> TYPE
        (zlib) -> DICTID or TYPE
        DICTID -> DICT -> TYPE
        (raw) -> TYPEDO
    Read deflate blocks:
            TYPE -> TYPEDO -> STORED or TABLE or LEN_ or CHECK
            STORED -> COPY_ -> COPY -> TYPE
            TABLE -> LENLENS -> CODELENS -> LEN_
            LEN_ -> LEN
    Read deflate codes in fixed or dynamic block:
                LEN -> LENEXT or LIT or TYPE
                LENEXT -> DIST -> DISTEXT -> MATCH -> LEN
                LIT -> LEN
    Process trailer:
        CHECK -> LENGTH -> DONE
 */

/* State maintained between inflate() calls -- approximately 7K bytes, not
   including the allocated sliding window, which is up to 32K bytes. */
struct inflate_state {
    z_streamp strm;             /* pointer back to this zlib stream */
    inflate_mode mode;          /* current inflate mode */
    int last;                   /* true if processing last block */
    int wrap;                   /* bit 0 true for zlib, bit 1 true for gzip,
                                   bit 2 true to validate check value */
    int havedict;               /* true if dictionary provided */
    int flags;                  /* gzip header method and flags, 0 if zlib, or
                                   -1 if raw or no header yet */
    unsigned dmax;              /* zlib header max distance (INFLATE_STRICT) */
    unsigned long check;        /* protected copy of check value */
    unsigned long total;        /* protected copy of output count */
    gz_headerp head;            /* where to save gzip header information */
        /* sliding window */
    unsigned wbits;             /* log base 2 of requested window size */
    # 窗口大小，如果不使用窗口则为零
    unsigned wsize;
    # 窗口中有效的字节数
    unsigned whave;
    # 窗口写入索引
    unsigned wnext;
    # 分配的滑动窗口，如果需要的话
    unsigned char FAR *window;
    # 输入位累加器
    unsigned long hold;
    # "in"中的位数
    unsigned bits;
    # 用于字符串和存储块复制
    # 数据的文字或长度
    unsigned length;
    # 从哪里复制字符串的距离
    unsigned offset;
    # 需要额外位
    unsigned extra;
    # 长度/文字代码的起始表
    code const FAR *lencode;
    # 距离代码的起始表
    code const FAR *distcode;
    # lencode的索引位
    unsigned lenbits;
    # distcode的索引位
    unsigned distbits;
    # 动态表构建
    # 代码长度代码长度的数量
    unsigned ncode;
    # 长度代码长度的数量
    unsigned nlen;
    # 距离代码长度的数量
    unsigned ndist;
    # lens[]中的代码长度数量
    unsigned have;
    # codes[]中下一个可用空间
    code FAR *next;
    # 用于代码长度的临时存储
    unsigned short lens[320];
    # 用于代码表构建的工作区
    unsigned short work[288];
    # 代码表的空间
    code codes[ENOUGH];
    # 如果为false，则允许无效距离太远
    int sane;
    # 最后一个未处理的长度/文字的位数
    int back;
    # 匹配的初始长度
    unsigned was;
# 代码结尾的分号，可能是用于结束某个语句或代码块的标记
```