# `nmap\libz\contrib\infback9\inflate9.h`

```
/* inflate9.h -- internal inflate state definition
 * 声明内部解压缩状态的定义
 * 版权所有 (C) 1995-2003 Mark Adler
 * 有关发布和使用条件，请参阅 zlib.h 中的版权声明
 */

/* 警告：应用程序不应使用此文件。它是压缩库实现的一部分，并且可能会更改。应用程序应仅使用 zlib.h。
 */

/* 在 inflate() 调用之间可能的解压缩模式 */
typedef enum {
        TYPE,       /* i: 等待类型位，包括最后一个标志位 */
        STORED,     /* i: 等待存储大小（长度和补码） */
        TABLE,      /* i: 等待动态块表长度 */
            LEN,        /* i: 等待长度/字面码 */
    DONE,       /* 完成检查，完成 -- 保持在此状态直到重置 */
    BAD         /* 发生数据错误 -- 保持在此状态直到重置 */
} inflate_mode;

/*
    上述模式之间的状态转换 -

    （大多数模式可以转到 BAD 模式 -- 为了清晰起见未显示）

    读取 deflate 块：
            TYPE -> STORED 或 TABLE 或 LEN 或 DONE
            STORED -> TYPE
            TABLE -> LENLENS -> CODELENS -> LEN
    读取 deflate 代码：
                LEN -> LEN 或 TYPE
 */

/* 在 inflate() 调用之间维护的状态。大约 7K 字节。 */
struct inflate_state {
        /* 滑动窗口 */
    unsigned char FAR *window;  /* 分配的滑动窗口，如果需要的话 */
        /* 动态表构建 */
    unsigned ncode;             /* 代码长度码长度的数量 */
    unsigned nlen;              /* 长度码长度的数量 */
    unsigned ndist;             /* 距离码长度的数量 */
    unsigned have;              /* lens[] 中的代码长度数量 */
    code FAR *next;             /* codes[] 中下一个可用空间 */
    unsigned short lens[320];   /* 代码长度的临时存储 */
    unsigned short work[288];   /* 用于代码表构建的工作区 */
    code codes[ENOUGH];         /* 代码表的空间 */
};
```