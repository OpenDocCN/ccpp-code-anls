# `nmap\libz\examples\gzlog.c`

```
/*
 * gzlog.c
 * 版权所有 2004, 2008, 2012, 2016, 2019 Mark Adler
 * 有关分发和使用条件，请参阅 gzlog.h 中的版权声明
 * 版本 2.3, 2019年5月25日
 */

/*
   gzlog 提供了一种频繁向 gzip 文件追加短字符串的机制，既能在执行时间上高效，又能保持较高的压缩比。策略是将短字符串以未压缩形式写入 gzip 文件的末尾，只有在未压缩数据量达到给定阈值时才进行压缩。

   gzlog 还提供了对系统崩溃中断的保护机制。操作的状态记录在 gzip 文件的一个额外字段中，只有在将 gzip 文件带到有效状态后才会更新。最后追加或压缩的数据保存在一个辅助文件中，因此如果操作中断，可以在下次尝试追加操作时完成。

   gzlog 还维护另一个辅助文件，其中包含压缩部分的最后32K数据，这些数据会预加载以用于后续数据的压缩。这最小化了追加对压缩比的影响。
 */

#include <sys/types.h>
#include <stdio.h>      /* rename, fopen, fprintf, fclose */
#include <stdlib.h>     /* malloc, free */
#include <string.h>     /* strlen, strrchr, strcpy, strncpy, strcmp */
#include <fcntl.h>      /* open */
#include <unistd.h>     /* lseek, read, write, close, unlink, sleep, */
                        /* ftruncate, fsync */
#include <errno.h>      /* errno */
#include <time.h>       /* time, ctime */
#include <sys/stat.h>   /* stat */
#include <sys/time.h>   /* utimes */
#include "zlib.h"       /* crc32 */

#include "gzlog.h"      /* 用于外部访问的头文件 */

#define local static
typedef unsigned int uint;
typedef unsigned long ulong;

/* 用于调试的宏，以确定性地强制执行恢复操作 */
#ifdef GZLOG_DEBUG
    #include <setjmp.h>         /* 引入 setjmp.h 头文件，用于支持 longjmp 函数 */
    jmp_buf gzlog_jump;         /* 定义一个跳转位置的缓冲区变量 */
    int gzlog_bail = 0;         /* 定义一个用于指示在哪个点跳出的变量，取值范围为 1 到 8 */
    int gzlog_count = -1;       /* 定义一个用于记录循环次数的变量，初始值为 -1 */
#ifdef gzlog_bail
// 定义一个宏，用于在特定条件下跳转到指定位置
#define BAIL(n) do { if (n == gzlog_bail && gzlog_count-- == 0) \
                            longjmp(gzlog_jump, gzlog_bail); } while (0)
#else
// 如果未定义gzlog_bail，则定义BAIL为空
#define BAIL(n)
#endif

// 定义锁文件在多少秒之前被认为是过期的
#define PATIENCE 300

// 存储块的最大大小（以KB为单位），必须在1到63之间
#define MAX_STORE 16

// 触发压缩的存储KB数（必须大于等于32以允许字典构建，并且小于等于204 * MAX_STORE，以便>> 10丢弃每个存储块头部的贡献）
#define TRIGGER 1024

// deflate字典的大小（不能更改）
#define DICT 32768U

// 操作的值（2位）
#define NO_OP 0
#define APPEND_OP 1
#define COMPRESS_OP 2
#define REPLACE_OP 3

// 从无符号字节缓冲区中提取小端整数的宏
#define PULL2(p) ((p)[0]+((uint)((p)[1])<<8))
#define PULL4(p) (PULL2(p)+((ulong)PULL2(p+2)<<16))
#define PULL8(p) (PULL4(p)+((off_t)PULL4(p+4)<<32))

// 将整数按小端顺序存储到字节缓冲区的宏
#define PUT2(p,a) do {(p)[0]=a;(p)[1]=(a)>>8;} while(0)
#define PUT4(p,a) do {PUT2(p,a);PUT2(p+2,a>>16);} while(0)
#define PUT8(p,a) do {PUT4(p,a);PUT4(p+4,a>>32);} while(0)

// 日志信息的内部结构
#define LOGID "\106\035\172"    /* 应该是三个非零字符 */
struct log {
    char id[4];     /* 包含LOGID以检测意外覆盖 */
    int fd;         /* 用于.gz文件的文件描述符，以读/写方式打开 */
    char *path;     /* 分配的路径，例如"/var/log/foo"或"foo" */
    char *end;      /* 路径的末尾，用于添加后缀，如".gz" */
    off_t first;    /* 第一个存储块的偏移量，第一个长度字节 */
    int back;       /* 第一个块ID的位置，从第一个开始向后的位数 */
    uint stored;    /* 当前最后一个存储块中的字节数 */
    off_t last;     /* 最后一个存储块的偏移量，第一个长度字节 */
    ulong ccrc;     /* 压缩数据的 CRC 校验值 */
    ulong clen;     /* 压缩数据的长度（模 2^32） */
    ulong tcrc;     /* 总数据的 CRC 校验值 */
    ulong tlen;     /* 总数据的长度（模 2^32） */
    time_t lock;    /* 我们的锁定文件的最后修改时间 */
/* gzip header for gzlog */
local unsigned char log_gzhead[] = {
    0x1f, 0x8b,                 /* magic gzip id */
    8,                          /* compression method is deflate */
    4,                          /* there is an extra field (no file name) */
    0, 0, 0, 0,                 /* no modification time provided */
    0, 0xff,                    /* no extra flags, no OS specified */
    39, 0, 'a', 'p', 35, 0      /* extra field with "ap" subfield */
                                /* 35 is EXTRA, 39 is EXTRA + 4 */
};

#define HEAD sizeof(log_gzhead)     /* should be 16 */

/* initial gzip extra field content (52 == HEAD + EXTRA + 1) */
local unsigned char log_gzext[] = {
    52, 0, 0, 0, 0, 0, 0, 0,    /* offset of first stored block length */
    52, 0, 0, 0, 0, 0, 0, 0,    /* offset of last stored block length */
    0, 0, 0, 0, 0, 0, 0, 0,     /* compressed data crc and length */
    0, 0, 0, 0, 0, 0, 0, 0,     /* total data crc and length */
    0, 0,                       /* final stored block data length */
    5                           /* op is NO_OP, last bit 8 bits back */
};

#define EXTRA sizeof(log_gzext)     /* should be 35 */

/* initial gzip data and trailer */
local unsigned char log_gzbody[] = {
    1, 0, 0, 0xff, 0xff,        /* empty stored block (last) */
    0, 0, 0, 0,                 /* crc */
    0, 0, 0, 0                  /* uncompressed length */
};

#define BODY sizeof(log_gzbody)
/* 为了独占地创建 foo.lock 文件，以便独占访问 foo.* 文件。如果现有锁文件的修改时间比 PATIENCE 秒之前，则认为锁文件已被放弃，删除它，并再次尝试独占创建。保存锁文件的修改时间以验证所有权。成功返回 0，失败返回 -1，通常是由于访问限制或无效路径。请注意，如果 stat() 或 unlink() 失败，可能是由于另一个进程稍早地注意到了放弃的锁文件并删除了它，因此这些不被标记为错误。 */
local int log_lock(struct log *log)
{
    int fd;
    struct stat st;

    strcpy(log->end, ".lock");
    while ((fd = open(log->path, O_CREAT | O_EXCL, 0644)) < 0) {
        if (errno != EEXIST)
            return -1;
        if (stat(log->path, &st) == 0 && time(NULL) - st.st_mtime > PATIENCE) {
            unlink(log->path);
            continue;
        }
        sleep(2);       /* 在等待期间释放 CPU 两秒钟 */
    }
    close(fd);
    if (stat(log->path, &st) == 0)
        log->lock = st.st_mtime;
    return 0;
}

/* 更新锁文件的修改时间为当前时间，以防止另一个任务认为锁已经过时。保存锁文件的修改时间以验证所有权。 */
local void log_touch(struct log *log)
{
    struct stat st;

    strcpy(log->end, ".lock");
    utimes(log->path, NULL);
    if (stat(log->path, &st) == 0)
        log->lock = st.st_mtime;
}

/* 检查日志文件的修改时间是否符合预期。如果这不是我们的锁，则返回 true。如果是我们的锁，则触摸它以保持它。 */
local int log_check(struct log *log)
{
    struct stat st;

    strcpy(log->end, ".lock");
    if (stat(log->path, &st) || st.st_mtime != log->lock)
        return 1;
    log_touch(log);
    return 0;
}

/* 解锁先前获取的锁，但仅当它是我们的锁时。 */
# 解锁日志文件，如果日志文件已经被检查过则直接返回
local void log_unlock(struct log *log)
{
    if (log_check(log))
        return;
    # 在日志文件名末尾添加".lock"后缀
    strcpy(log->end, ".lock");
    # 删除原始日志文件
    unlink(log->path);
    # 将日志文件的锁状态设置为0
    log->lock = 0;
}

/* 检查 gzip 头并读取额外字段，填充日志结构中的值。
   成功时返回操作符 op，如果 gzip 头不符合预期则返回-1。
   op 是最后写入额外字段的当前操作。
   这假设 gzip 文件已经被打开，文件描述符为 log->fd。 */
local int log_head(struct log *log)
{
    int op;
    unsigned char buf[HEAD + EXTRA];

    # 移动文件指针到文件开头，读取头部和额外字段
    if (lseek(log->fd, 0, SEEK_SET) < 0 ||
        read(log->fd, buf, HEAD + EXTRA) != HEAD + EXTRA ||
        memcmp(buf, log_gzhead, HEAD)) {
        return -1;
    }
    # 填充日志结构中的值
    log->first = PULL8(buf + HEAD);
    log->last = PULL8(buf + HEAD + 8);
    log->ccrc = PULL4(buf + HEAD + 16);
    log->clen = PULL4(buf + HEAD + 20);
    log->tcrc = PULL4(buf + HEAD + 24);
    log->tlen = PULL4(buf + HEAD + 28);
    log->stored = PULL2(buf + HEAD + 32);
    log->back = 3 + (buf[HEAD + 34] & 7);
    op = (buf[HEAD + 34] >> 3) & 3;
    return op;
}

/* 覆盖额外字段内容，标记操作为 op。
   使用 fsync 确保设备被写入，并按请求的顺序。
   这个操作，只有这个操作，被假设为原子操作，以确保在任何过程中的中断事件中日志是可恢复的。
   如果写入到 foo.gz 失败则返回-1。 */
local int log_mark(struct log *log, int op)
{
    int ret;
    unsigned char ext[EXTRA];

    # 填充额外字段内容
    PUT8(ext, log->first);
    PUT8(ext + 8, log->last);
    PUT4(ext + 16, log->ccrc);
    PUT4(ext + 20, log->clen);
    PUT4(ext + 24, log->tcrc);
    PUT4(ext + 28, log->tlen);
    PUT2(ext + 32, log->stored);
    ext[34] = log->back - 3 + (op << 3);
    # 同步文件到磁盘
    fsync(log->fd);
    # 移动文件指针到头部，写入额外字段内容
    ret = lseek(log->fd, HEAD, SEEK_SET) < 0 ||
          write(log->fd, ext, EXTRA) != EXTRA ? -1 : 0;
    # 将日志文件的数据同步到磁盘
    fsync(log->fd);
    # 返回函数的结果
    return ret;
/* 重写最后一个块头部的位和后续的零位，以达到字节边界，如果last为真，则设置最后一个块的位，然后写入存储块头部的剩余部分（长度和补码）。将文件指针留在最后一个存储块数据的末尾。如果在foo.gz文件上发生读取或写入失败，则返回-1 */
local int log_last(struct log *log, int last)
{
    int back, len, mask;
    unsigned char buf[6];

    /* 确定要修改的字节和位的位置 */
    back = log->last == log->first ? log->back : 8;
    len = back > 8 ? 2 : 1;                 /* 从log->last开始的字节数 */
    mask = 0x80 >> ((back - 1) & 7);        /* 块最后一位的掩码 */

    /* 将要修改的字节（一个或两个字节）读入buf[0] -- 如果最后一位是八位前，就不需要读取字节，因为在这种情况下整个字节都将被修改 */
    buf[0] = 0;
    if (back != 8 && (lseek(log->fd, log->last - len, SEEK_SET) < 0 ||
                      read(log->fd, buf, 1) != 1))
        return -1;

    /* 根据请求更改最后一个存储块的最后一位 -- 注意，最后一位以上的所有位都被设置为零，根据存储块的类型位为00和将流带到字节边界的约定，这些位也是零 */
    buf[1] = 0;
    buf[2 - len] = (*buf & (mask - 1)) + (last ? mask : 0);

    /* 写入修改后的存储块头部和长度，将文件指针移动到最后一个存储块数据之后 */
    PUT2(buf + 2, log->stored);
    PUT2(buf + 4, log->stored ^ 0xffff);
    return lseek(log->fd, log->last - len, SEEK_SET) < 0 ||
           write(log->fd, buf + 2 - len, len + 4) != len + 4 ||
           lseek(log->fd, log->stored, SEEK_CUR) < 0 ? -1 : 0;
}
/* 将长度为len的数据从data追加到锁定并打开的日志文件中。如果正在恢复并且找不到.add文件，则len可以为零。在这种情况下，将恢复foo.gz文件的先前状态。数据以未压缩形式追加到deflate存储块中。如果读取或写入foo.gz文件时出现错误，则返回-1。 */
local int log_append(struct log *log, unsigned char *data, size_t len)
{
    uint put;
    off_t end;
    unsigned char buf[8];

    /* 设置最后一个块的最后位和长度，以防止中断追加，然后将文件指针定位到块的末尾以进行追加 */
    if (log_last(log, 1))
        return -1;

    /* 追加，根据需要添加存储块并更新最后存储块的偏移量，并更新总crc和长度 */
    while (len) {
        /* 尽可能多地追加到最后一个块 */
        put = (MAX_STORE << 10) - log->stored;
        if (put > len)
            put = (uint)len;
        if (put) {
            if (write(log->fd, data, put) != put)
                return -1;
            BAIL(1);
            log->tcrc = crc32(log->tcrc, data, put);
            log->tlen += put;
            log->stored += put;
            data += put;
            len -= put;
        }

        /* 如果需要，添加一个新的空存储块 */
        if (len) {
            /* 将当前块标记为非最后一个 */
            if (log_last(log, 0))
                return -1;

            /* 指向新的空存储块 */
            log->last += 4 + log->stored + 1;
            log->stored = 0;
        }

        /* 将最后一个块标记为最后一个，并更新其长度 */
        if (log_last(log, 1))
            return -1;
        BAIL(2);
    }

    /* 写入新的crc和长度尾部，并进行截断以防万一（可能正在从缺少foo.add文件的部分追加中恢复） */
    PUT4(buf, log->tcrc);
    PUT4(buf + 4, log->tlen);
    # 如果写入日志文件的内容不是8个字节，或者无法获取当前文件位置，或者无法截断文件，则返回-1
    if (write(log->fd, buf, 8) != 8 ||
        (end = lseek(log->fd, 0, SEEK_CUR)) < 0 || ftruncate(log->fd, end))
        return -1;

    # 写入额外的字段，标记日志文件已完成，删除 .add 文件
    if (log_mark(log, NO_OP))
        return -1;
    # 将文件名末尾改为 ".add"
    strcpy(log->end, ".add");
    # 删除日志文件，忽略错误，因为文件可能不存在
    unlink(log->path);
    # 返回0
    return 0;
}

/* 用 foo.temp 文件替换 foo.dict 文件。同时删除 foo.add 文件，因为压缩操作可能在完成之前被中断。如果无法分配内存则返回1，如果读取或写入 foo.gz 失败，或者由于某种原因重命名失败，则返回-1，除非是 foo.temp 不存在。foo.temp 不存在是允许的错误，因为替换操作可能在重命名完成后，但在 foo.gz 标记为完成之前被中断。 */
local int log_replace(struct log *log)
{
    int ret;
    char *dest;

    /* 删除 foo.add 文件 */
    strcpy(log->end, ".add");
    unlink(log->path);         /* 忽略错误，因为可能不存在 */
    BAIL(3);

    /* 重命名 foo.name 为 foo.dict，如果 foo.dict 存在则替换 */
    strcpy(log->end, ".dict");
    dest = malloc(strlen(log->path) + 1);
    if (dest == NULL)
        return -2;
    strcpy(dest, log->path);
    strcpy(log->end, ".temp");
    ret = rename(log->path, dest);
    free(dest);
    if (ret && errno != ENOENT)
        return -1;
    BAIL(4);

    /* 标记 foo.gz 文件为完成状态 */
    return log_mark(log, NO_OP);
}

/* 压缩 data 中的 len 字节并将压缩数据追加到 foo.gz 的 deflate 数据中，紧接在先前压缩数据之后。这将覆盖先前存储在 foo.add 中的未压缩数据，该数据在 data[0..len-1] 中提供。如果此操作被中断，则会在此处重新开始，重新读取 foo.add 文件。如果没有要压缩的数据（len == 0），则我们简单地在先前压缩数据之后终止 foo.gz 文件，追加最终的空存储块和 gzip 尾部。如果读取或写入 log.gz 文件失败，则返回-1，如果存在内存分配失败则返回-2。 */
local int log_compress(struct log *log, unsigned char *data, size_t len)
{
    int fd;
    uint got, max;
    ssize_t dict;
    off_t end;
    # 定义一个压缩流对象
    z_stream strm;
    # 定义一个字节数组，用于存储数据
    unsigned char buf[DICT];

    # 如果有数据需要压缩，则执行以下操作
    /* 压缩并追加压缩数据 */
    }
    # 如果没有数据需要压缩，则修复现有的 gzip 流
    else {
        # 将日志的 tcrc 值赋给 log->tcrc
        log->tcrc = log->ccrc;
        # 将日志的 clen 值赋给 log->tlen
        log->tlen = log->clen;
    }

    # 完成并截断 gzip 流
    # 将日志的 first 值赋给 log->last
    log->last = log->first;
    # 将日志的 stored 值设为 0
    log->stored = 0;
    # 将 log->tcrc 和 log->tlen 的值写入 buf 数组
    PUT4(buf, log->tcrc);
    PUT4(buf + 4, log->tlen);
    # 如果 log_last 函数返回 true，或者写入文件的字节数不等于 8，或者 lseek 或 ftruncate 操作失败，则返回 -1
    if (log_last(log, 1) || write(log->fd, buf, 8) != 8 ||
        (end = lseek(log->fd, 0, SEEK_CUR)) < 0 || ftruncate(log->fd, end))
        return -1;
    # 跳转到标签 6 处
    BAIL(6);

    # 标记为正在进行替换操作
    # 如果标记失败，则返回 -1
    if (log_mark(log, REPLACE_OP))
        return -1;

    # 执行替换操作并标记文件为已完成
    return log_replace(log);
/* log a repair record to the .repairs file */
local void log_log(struct log *log, int op, char *record)
{
    // 获取当前时间
    time_t now;
    now = time(NULL);
    // 将文件名后缀改为 .repairs
    strcpy(log->end, ".repairs");
    // 以追加模式打开文件
    FILE *rec;
    rec = fopen(log->path, "a");
    // 如果文件打开失败则返回
    if (rec == NULL)
        return;
    // 将修复记录写入文件
    fprintf(rec, "%.24s %s recovery: %s\n", ctime(&now), op == APPEND_OP ?
            "append" : (op == COMPRESS_OP ? "compress" : "replace"), record);
    // 关闭文件
    fclose(rec);
    return;
}

/* Recover the interrupted operation op.  First read foo.add for recovering an
   append or compress operation.  Return -1 if there was an error reading or
   writing foo.gz or reading an existing foo.add, or -2 if there was a memory
   allocation failure. */
local int log_recover(struct log *log, int op)
{
    int fd, ret = 0;
    unsigned char *data = NULL;
    size_t len = 0;
    struct stat st;

    /* log recovery */
    // 记录恢复操作开始
    log_log(log, op, "start");

    /* load foo.add file if expected and present */
    // 如果是追加或压缩操作，加载 .add 文件
    if (op == APPEND_OP || op == COMPRESS_OP) {
        strcpy(log->end, ".add");
        // 获取文件信息
        if (stat(log->path, &st) == 0 && st.st_size) {
            len = (size_t)(st.st_size);
            // 分配内存
            if ((off_t)len != st.st_size ||
                    (data = malloc(st.st_size)) == NULL) {
                log_log(log, op, "allocation failure");
                return -2;
            }
            // 打开文件
            if ((fd = open(log->path, O_RDONLY, 0)) < 0) {
                free(data);
                log_log(log, op, ".add file read failure");
                return -1;
            }
            // 读取文件内容
            ret = (size_t)read(fd, data, len) != len;
            close(fd);
            // 如果读取失败则释放内存并返回
            if (ret) {
                free(data);
                log_log(log, op, ".add file read failure");
                return -1;
            }
            // 记录加载 .add 文件成功
            log_log(log, op, "loaded .add file");
        }
        else
            // 记录缺少 .add 文件
            log_log(log, op, "missing .add file!");
    }

    /* recover the interrupted operation */
    // 恢复中断的操作
    switch (op) {
    # 如果操作是追加操作，调用log_append函数，将返回值赋给ret
    case APPEND_OP:
        ret = log_append(log, data, len);
        break;
    # 如果操作是压缩操作，调用log_compress函数，将返回值赋给ret
    case COMPRESS_OP:
        ret = log_compress(log, data, len);
        break;
    # 如果操作是替换操作，调用log_replace函数，将返回值赋给ret
    case REPLACE_OP:
        ret = log_replace(log);
    }

    # 记录日志状态，如果ret为真，记录为"failure"，否则记录为"complete"
    log_log(log, op, ret ? "failure" : "complete");

    # 清理内存，如果data不为空，释放其内存
    if (data != NULL)
        free(data);
    # 返回ret
    return ret;
/* 关闭 foo.gz 文件（如果已打开），释放锁 */
local void log_close(struct log *log)
{
    // 如果文件描述符大于等于0，关闭文件
    if (log->fd >= 0)
        close(log->fd);
    // 重置文件描述符为-1
    log->fd = -1;
    // 释放锁
    log_unlock(log);
}

/* 打开 foo.gz，验证头部，并加载额外字段内容，在创建 foo.* 文件的 foo.lock 文件以获得独占访问权之后。
   如果 foo.gz 不存在或为空，则写入一个空的 foo.gz 日志文件的初始头部、额外字段和正文内容。
   如果由于访问限制而无法创建锁文件，或者读取或写入 foo.gz 文件时出现错误，
   或者 foo.gz 文件不是该对象的正确日志文件（例如不是 gzip 文件或不包含预期的额外字段），
   则返回 true。如果出现错误，则释放锁。否则，保留锁。 */
local int log_open(struct log *log)
{
    int op;

    /* 释放打开的文件资源（如果有）-- 如果在 gzlog_open() 和 gzlog_write() 之间丢失锁，可能会发生 */
    if (log->fd >= 0)
        close(log->fd);
    log->fd = -1;

    /* 协商独占访问权 */
    if (log_lock(log) < 0)
        return -1;

    /* 打开日志文件，foo.gz */
    strcpy(log->end, ".gz");
    log->fd = open(log->path, O_RDWR | O_CREAT, 0644);
    if (log->fd < 0) {
        log_close(log);
        return -1;
    }

    /* 如果是新文件，用空日志初始化 foo.gz，删除旧字典 */
    if (lseek(log->fd, 0, SEEK_END) == 0) {
        if (write(log->fd, log_gzhead, HEAD) != HEAD ||
            write(log->fd, log_gzext, EXTRA) != EXTRA ||
            write(log->fd, log_gzbody, BODY) != BODY) {
            log_close(log);
            return -1;
        }
        strcpy(log->end, ".dict");
        unlink(log->path);
    }

    /* 验证日志文件并加载额外字段信息 */
    if ((op = log_head(log)) < 0) {
        log_close(log);
        return -1;
    }

    /* 检查是否有中断的进程，如果有，则恢复 */
}
    # 如果操作不是无操作，并且日志恢复成功
    if (op != NO_OP && log_recover(log, op)) {
        # 关闭日志文件
        log_close(log);
        # 返回-1表示失败
        return -1;
    }

    # 触碰锁文件以防止另一个进程获取它
    log_touch(log);
    # 返回0表示成功
    return 0;
}
/* See gzlog.h for the description of the external methods below */

/* 打开一个日志文件，返回一个指向 gzlog 结构的指针 */
gzlog *gzlog_open(char *path)
{
    size_t n;
    struct log *log;

    /* 检查参数 */
    if (path == NULL || *path == 0)
        return NULL;

    /* 分配并初始化日志结构 */
    log = malloc(sizeof(struct log));
    if (log == NULL)
        return NULL;
    strcpy(log->id, LOGID);
    log->fd = -1;

    /* 保存路径和路径末尾以便构造文件名 */
    n = strlen(path);
    log->path = malloc(n + 9);              /* 允许 ".repairs" */
    if (log->path == NULL) {
        free(log);
        return NULL;
    }
    strcpy(log->path, path);
    log->end = log->path + n;

    /* 获得独占访问权并验证日志文件 -- 如果需要可能执行恢复操作 */
    if (log_open(log)) {
        free(log->path);
        free(log);
        return NULL;
    }

    /* 返回指向日志结构的指针 */
    return log;
}

/* gzlog_compress() 返回值:
    0: 成功
   -1: 文件 I/O 错误 (通常是访问问题)
   -2: 内存分配失败
   -3: 无效的日志指针参数 */
int gzlog_compress(gzlog *logd)
{
    int fd, ret;
    uint block;
    size_t len, next;
    unsigned char *data, buf[5];
    struct log *log = logd;

    /* 检查参数 */
    if (log == NULL || strcmp(log->id, LOGID))
        return -3;

    /* 检查是否丢失了锁 -- 如果是，重新获取锁并重新加载额外字段信息 (它可能已经改变)，如果需要则恢复上次操作 */
    if (log_check(log) && log_open(log))
        return -1;

    /* 为未压缩数据创建空间 */
    len = ((size_t)(log->last - log->first) & ~(((size_t)1 << 10) - 1)) +
          log->stored;
    if ((data = malloc(len)) == NULL)
        return -2;

    /* 这里的 do 语句只是一个处理错误的小技巧 */
    do {
        /* 读取未压缩的数据 */
        // 如果文件指针移动到 log->first - 1 处失败，则跳出循环
        if (lseek(log->fd, log->first - 1, SEEK_SET) < 0)
            break;
        next = 0;
        while (next < len) {
            // 读取 5 个字节到 buf 中，如果失败则跳出循环
            if (read(log->fd, buf, 5) != 5)
                break;
            // 从 buf 中获取 block 大小
            block = PULL2(buf + 1);
            // 如果 next + block 大于 len，或者读取 block 大小的数据失败，则跳出循环
            if (next + block > len ||
                read(log->fd, (char *)data + next, block) != block)
                break;
            next += block;
        }
        // 如果当前文件指针位置不等于 log->last + 4 + log->stored，则跳出循环
        if (lseek(log->fd, 0, SEEK_CUR) != log->last + 4 + log->stored)
            break;
        // 更新 log 的时间戳
        log_touch(log);

        /* 将未压缩的数据写入 .add 文件 */
        // 将 ".add" 追加到 log->end 后面
        strcpy(log->end, ".add");
        // 以写入模式打开文件，如果失败则跳出循环
        fd = open(log->path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd < 0)
            break;
        // 将数据写入文件，如果写入失败或者关闭文件失败，则跳出循环
        ret = (size_t)write(fd, data, len) != len;
        if (ret | close(fd))
            break;
        // 更新 log 的时间戳
        log_touch(log);

        /* 将下一次压缩所需的字典写入 .temp 文件 */
        // 将 ".temp" 追加到 log->end 后面
        strcpy(log->end, ".temp");
        // 以写入模式打开文件，如果失败则跳出循环
        fd = open(log->path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd < 0)
            break;
        // 计算需要写入的字典大小
        next = DICT > len ? len : DICT;
        // 将字典数据写入文件，如果写入失败或者关闭文件失败，则跳出循环
        ret = (size_t)write(fd, (char *)data + len - next, next) != next;
        if (ret | close(fd))
            break;
        // 更新 log 的时间戳
        log_touch(log);

        /* 回滚到压缩数据，标记压缩正在进行中 */
        // 更新 log 的 last 和 stored 属性，如果失败则跳出循环
        log->last = log->first;
        log->stored = 0;
        if (log_mark(log, COMPRESS_OP))
            break;
        BAIL(7);

        /* 压缩并追加数据（清除标记） */
        // 压缩并追加数据，返回压缩结果，如果失败则跳出循环
        ret = log_compress(log, data, len);
        // 释放 data 内存
        free(data);
        return ret;
    } while (0);

    /* 在上面的 do 循环中因为 I/O 错误而跳出 */
    // 释放 data 内存
    free(data);
    return -1;
/* gzlog_write() return values:
    0: all good
   -1: file i/o error (usually access issue)
   -2: memory allocation failure
   -3: invalid log pointer argument */
int gzlog_write(gzlog *logd, void *data, size_t len)
{
    int fd, ret;
    struct log *log = logd;

    /* check arguments */
    // 检查日志指针是否为空，以及日志标识是否匹配
    if (log == NULL || strcmp(log->id, LOGID))
        return -3;
    // 检查数据指针是否为空，以及数据长度是否大于0
    if (data == NULL || len <= 0)
        return 0;

    /* see if we lost the lock -- if so get it again and reload the extra
       field information (it probably changed), recover last operation if
       necessary */
    // 检查是否丢失了锁 -- 如果是，重新获取锁并重新加载额外字段信息（它可能已经改变），如果必要则恢复上次操作
    if (log_check(log) && log_open(log))
        return -1;

    /* create and write .add file */
    // 创建并写入 .add 文件
    strcpy(log->end, ".add");
    fd = open(log->path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0)
        return -1;
    ret = (size_t)write(fd, data, len) != len;
    if (ret | close(fd))
        return -1;
    log_touch(log);

    /* mark log file with append in progress */
    // 用正在进行的追加标记日志文件
    if (log_mark(log, APPEND_OP))
        return -1;
    BAIL(8);

    /* append data (clears mark) */
    // 追加数据（清除标记）
    if (log_append(log, data, len))
        return -1;

    /* check to see if it's time to compress -- if not, then done */
    // 检查是否是压缩时间 -- 如果不是，那么完成
    if (((log->last - log->first) >> 10) + (log->stored >> 10) < TRIGGER)
        return 0;

    /* time to compress */
    // 压缩时间到了
    return gzlog_compress(log);
}

/* gzlog_close() return values:
    0: ok
   -3: invalid log pointer argument */
int gzlog_close(gzlog *logd)
{
    struct log *log = logd;

    /* check arguments */
    // 检查日志指针是否为空，以及日志标识是否匹配
    if (log == NULL || strcmp(log->id, LOGID))
        return -3;

    /* close the log file and release the lock */
    // 关闭日志文件并释放锁
    log_close(log);

    /* free structure and return */
    // 释放结构并返回
    if (log->path != NULL)
        free(log->path);
    strcpy(log->id, "bad");
    free(log);
    return 0;
}
```