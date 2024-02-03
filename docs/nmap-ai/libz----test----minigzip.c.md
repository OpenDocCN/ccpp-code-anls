# `nmap\libz\test\minigzip.c`

```cpp
/* minigzip.c -- 使用 zlib 压缩库模拟 gzip
 * 版权所有 (C) 1995-2006, 2010, 2011, 2016 Jean-loup Gailly
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

/*
 * minigzip 是 gzip 实用程序的最小实现。这只是使用 zlib 的示例，不意味着取代
 * 完整功能的 gzip。不会尝试处理文件系统限制文件名为 14 或 8+3 字符等等... 错误检查非常有限。
 * 因此只能将 minigzip 用于测试；真正的操作请使用 gzip。在 MSDOS 上，只能用于没有扩展名的文件名
 * 或管道模式。
 */

/* @(#) $Id$ */

#include "zlib.h"
#include <stdio.h>

#ifdef STDC
#  include <string.h>
#  include <stdlib.h>
#endif

#ifdef USE_MMAP
#  include <sys/types.h>
#  include <sys/mman.h>
#  include <sys/stat.h>
#endif

#if defined(MSDOS) || defined(OS2) || defined(WIN32) || defined(__CYGWIN__)
#  include <fcntl.h>
#  include <io.h>
#  ifdef UNDER_CE
#    include <stdlib.h>
#  endif
#  define SET_BINARY_MODE(file) setmode(fileno(file), O_BINARY)
#else
#  define SET_BINARY_MODE(file)
#endif

#if defined(_MSC_VER) && _MSC_VER < 1900
#  define snprintf _snprintf
#endif

#ifdef VMS
#  define unlink delete
#  define GZ_SUFFIX "-gz"
#endif
#ifdef RISCOS
#  define unlink remove
#  define GZ_SUFFIX "-gz"
#  define fileno(file) file->__file
#endif
#if defined(__MWERKS__) && __dest_os != __be_os && __dest_os != __win32_os
#  include <unix.h> /* for fileno */
#endif

#if !defined(Z_HAVE_UNISTD_H) && !defined(_LARGEFILE64_SOURCE)
#ifndef WIN32 /* WIN32 中 unlink 已经在 stdio.h 中 */
  extern int unlink OF((const char *));
#endif
#endif

#if defined(UNDER_CE)
#  include <windows.h>
#  define perror(s) pwinerror(s)
/* 将 Windows 错误号映射到与区域设置相关的错误消息字符串，并返回指向它的指针。通常，ERROR 的值来自 GetLastError。

   指向的字符串不得被应用程序修改，但可能会被后续对 strwinerror 的调用覆盖。

   strwinerror 函数不会改变 GetLastError 的当前设置。 */

static char *strwinerror (error)
     DWORD error;
{
    static char buf[1024];  // 静态缓冲区，用于存储错误消息字符串

    wchar_t *msgbuf;  // 宽字符指针，用于存储错误消息
    DWORD lasterr = GetLastError();  // 获取当前的错误号
    DWORD chars = FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM
        | FORMAT_MESSAGE_ALLOCATE_BUFFER,
        NULL,
        error,
        0, /* 默认语言 */
        (LPVOID)&msgbuf,
        0,
        NULL);  // 格式化错误消息
    if (chars != 0) {
        /* 如果有 \r\n 附加在末尾，将其删除。 */
        if (chars >= 2
            && msgbuf[chars - 2] == '\r' && msgbuf[chars - 1] == '\n') {
            chars -= 2;
            msgbuf[chars] = 0;
        }

        if (chars > sizeof (buf) - 1) {
            chars = sizeof (buf) - 1;
            msgbuf[chars] = 0;
        }

        wcstombs(buf, msgbuf, chars + 1);  // 将宽字符转换为多字节字符
        LocalFree(msgbuf);  // 释放消息缓冲区
    }
    else {
        sprintf(buf, "unknown win32 error (%ld)", error);  // 如果格式化消息失败，返回未知错误消息
    }

    SetLastError(lasterr);  // 恢复之前的错误号
    return buf;  // 返回错误消息字符串
}

static void pwinerror (s)
    const char *s;
{
    if (s && *s)
        fprintf(stderr, "%s: %s\n", s, strwinerror(GetLastError ()));
    else
        fprintf(stderr, "%s\n", strwinerror(GetLastError ()));
}

#endif /* UNDER_CE */

#ifndef GZ_SUFFIX
#  define GZ_SUFFIX ".gz"
#endif
#define SUFFIX_LEN (sizeof(GZ_SUFFIX)-1)

#define BUFLEN      16384
#define MAX_NAME_LEN 1024

#ifdef MAXSEG_64K
#  define local static
   /* 对于堆栈大小有限制的系统，需要这个定义。 */
#else
#  define local
#endif

#ifdef Z_SOLO
/* 对于 Z_SOLO，使用 deflate 和 inflate 创建简化的 gz* 函数 */

#if defined(Z_HAVE_UNISTD_H) || defined(Z_LARGE)
#  include <unistd.h>       /* 用于 unlink() */
#endif

void *myalloc OF((void *, unsigned, unsigned));
void myfree OF((void *, void *));

void *myalloc(q, n, m)
    void *q;
    unsigned n, m;
{
    (void)q;
    return calloc(n, m); // 分配内存空间并返回指针
}

void myfree(q, p)
    void *q, *p;
{
    (void)q;
    free(p); // 释放内存空间
}

typedef struct gzFile_s {
    FILE *file; // 文件指针
    int write; // 写入标志
    int err; // 错误标志
    char *msg; // 错误消息
    z_stream strm; // 压缩流
} *gzFile; // gzFile 结构体指针

gzFile gzopen OF((const char *, const char *));
gzFile gzdopen OF((int, const char *));
gzFile gz_open OF((const char *, int, const char *));

gzFile gzopen(path, mode)
const char *path;
const char *mode;
{
    return gz_open(path, -1, mode); // 调用 gz_open 函数
}

gzFile gzdopen(fd, mode)
int fd;
const char *mode;
{
    return gz_open(NULL, fd, mode); // 调用 gz_open 函数
}

gzFile gz_open(path, fd, mode)
    const char *path;
    int fd;
    const char *mode;
{
    gzFile gz; // gzFile 结构体指针
    int ret; // 返回值

    gz = malloc(sizeof(struct gzFile_s)); // 分配内存空间
    if (gz == NULL)
        return NULL; // 分配失败返回空指针
    gz->write = strchr(mode, 'w') != NULL; // 判断是否为写入模式
    gz->strm.zalloc = myalloc; // 设置内存分配函数
    gz->strm.zfree = myfree; // 设置内存释放函数
    gz->strm.opaque = Z_NULL; // 设置透明指针
    if (gz->write)
        ret = deflateInit2(&(gz->strm), -1, 8, 15 + 16, 8, 0); // 初始化压缩流
    else {
        gz->strm.next_in = 0;
        gz->strm.avail_in = Z_NULL;
        ret = inflateInit2(&(gz->strm), 15 + 16); // 初始化解压缩流
    }
    if (ret != Z_OK) {
        free(gz); // 释放内存空间
        return NULL; // 返回空指针
    }
    gz->file = path == NULL ? fdopen(fd, gz->write ? "wb" : "rb") :
                              fopen(path, gz->write ? "wb" : "rb"); // 打开文件
    if (gz->file == NULL) {
        gz->write ? deflateEnd(&(gz->strm)) : inflateEnd(&(gz->strm)); // 结束压缩或解压缩
        free(gz); // 释放内存空间
        return NULL; // 返回空指针
    }
    gz->err = 0; // 错误标志清零
    gz->msg = ""; // 错误消息为空
    return gz; // 返回 gzFile 结构体指针
}

int gzwrite OF((gzFile, const void *, unsigned));

int gzwrite(gz, buf, len)
    gzFile gz;
    const void *buf;
    unsigned len;
{
    z_stream *strm; // 压缩流指针
    unsigned char out[BUFLEN]; // 输出缓冲区

    if (gz == NULL || !gz->write)
        return 0; // 如果 gz 为空或非写入模式，返回 0
    strm = &(gz->strm); // 获取压缩流指针
    strm->next_in = (void *)buf; // 设置输入缓冲区
    strm->avail_in = len; // 设置输入长度
    # 使用 do-while 循环执行以下操作，直到输出缓冲区为空
    do {
        # 设置输出缓冲区的起始位置
        strm->next_out = out;
        # 设置输出缓冲区的可用空间大小
        strm->avail_out = BUFLEN;
        # 使用 deflate 函数进行数据压缩
        (void)deflate(strm, Z_NO_FLUSH);
        # 将压缩后的数据写入文件
        fwrite(out, 1, BUFLEN - strm->avail_out, gz->file);
    } while (strm->avail_out == 0);  # 当输出缓冲区为空时继续循环
    # 返回数据长度
    return len;
# 定义一个函数 gzread，接受三个参数：gzFile 类型的 gz，void* 类型的 buf，unsigned 类型的 len
int gzread OF((gzFile, void *, unsigned));

# 实现函数 gzread，接受三个参数：gzFile 类型的 gz，void* 类型的 buf，unsigned 类型的 len
int gzread(gz, buf, len)
    # 声明变量 ret 为整型
    int ret;
    # 声明变量 got 为无符号整型
    unsigned got;
    # 声明变量 in 为一个长度为 1 的无符号字符数组
    unsigned char in[1];
    # 声明指针 strm 指向 z_stream 结构体
    z_stream *strm;

    # 如果 gz 为 NULL 或者 gz 的 write 属性为真，则返回 0
    if (gz == NULL || gz->write)
        return 0;
    # 如果 gz 的 err 属性为真，则返回 0
    if (gz->err)
        return 0;
    # 将 strm 指向 gz 的 strm 属性
    strm = &(gz->strm);
    # 将 strm 的 next_out 属性指向 buf
    strm->next_out = (void *)buf;
    # 将 strm 的 avail_out 属性设置为 len
    strm->avail_out = len;
    # 循环执行以下操作
    do {
        # 从 gz 的 file 属性中读取一个字节到 in 中，返回值赋给 got
        got = fread(in, 1, 1, gz->file);
        # 如果 got 为 0，则跳出循环
        if (got == 0)
            break;
        # 将 strm 的 next_in 属性指向 in
        strm->next_in = in;
        # 将 strm 的 avail_in 属性设置为 1
        strm->avail_in = 1;
        # 调用 inflate 函数，传入 strm 和 Z_NO_FLUSH，返回值赋给 ret
        ret = inflate(strm, Z_NO_FLUSH);
        # 如果 ret 等于 Z_DATA_ERROR
        if (ret == Z_DATA_ERROR) {
            # 将 gz 的 err 属性设置为 Z_DATA_ERROR
            gz->err = Z_DATA_ERROR;
            # 将 gz 的 msg 属性设置为 strm 的 msg 属性
            gz->msg = strm->msg;
            # 返回 0
            return 0;
        }
        # 如果 ret 等于 Z_STREAM_END
        if (ret == Z_STREAM_END)
            # 调用 inflateReset 函数，传入 strm
            inflateReset(strm);
    } while (strm->avail_out);
    # 返回 len 减去 strm 的 avail_out 属性
    return len - strm->avail_out;
}

# 定义一个函数 gzclose，接受一个参数：gzFile 类型的 gz
int gzclose OF((gzFile));

# 实现函数 gzclose，接受一个参数：gzFile 类型的 gz
int gzclose(gz)
    # 声明指针 strm 指向 z_stream 结构体
    z_stream *strm;
    # 声明一个长度为 BUFLEN 的无符号字符数组 out
    unsigned char out[BUFLEN];

    # 如果 gz 为 NULL，则返回 Z_STREAM_ERROR
    if (gz == NULL)
        return Z_STREAM_ERROR;
    # 将 strm 指向 gz 的 strm 属性
    strm = &(gz->strm);
    # 如果 gz 的 write 属性为真
    if (gz->write) {
        # 将 strm 的 next_in 属性指向 Z_NULL
        strm->next_in = Z_NULL;
        # 将 strm 的 avail_in 属性设置为 0
        strm->avail_in = 0;
        # 循环执行以下操作
        do {
            # 将 strm 的 next_out 属性指向 out
            strm->next_out = out;
            # 将 strm 的 avail_out 属性设置为 BUFLEN
            strm->avail_out = BUFLEN;
            # 调用 deflate 函数，传入 strm 和 Z_FINISH
            (void)deflate(strm, Z_FINISH);
            # 将 out 中的数据写入 gz 的 file 属性
            fwrite(out, 1, BUFLEN - strm->avail_out, gz->file);
        } while (strm->avail_out == 0);
        # 调用 deflateEnd 函数，传入 strm
        deflateEnd(strm);
    }
    # 否则
    else
        # 调用 inflateEnd 函数，传入 strm
        inflateEnd(strm);
    # 关闭 gz 的 file 属性
    fclose(gz->file);
    # 释放 gz 指向的内存
    free(gz);
    # 返回 Z_OK
    return Z_OK;
}

# 定义一个函数 gzerror，接受两个参数：gzFile 类型的 gz 和整型指针 err
const char *gzerror OF((gzFile, int *));

# 实现函数 gzerror，接受两个参数：gzFile 类型的 gz 和整型指针 err
const char *gzerror(gz, err)
    # 将 err 指向的值设置为 gz 的 err 属性
    *err = gz->err;
    # 返回 gz 的 msg 属性
    return gz->msg;
}

# 结束条件编译
#endif

# 声明一个静态字符指针 prog
static char *prog;

# 声明函数 error，接受一个参数：const char* 类型的 msg
void error            OF((const char *msg));
# 声明函数 gz_compress，接受两个参数：FILE* 类型的 in 和 gzFile 类型的 out
void gz_compress      OF((FILE   *in, gzFile out));
# 如果定义了 USE_MMAP，则声明函数 gz_compress_mmap，接受两个参数：FILE* 类型的 in 和 gzFile 类型的 out
#ifdef USE_MMAP
int  gz_compress_mmap OF((FILE   *in, gzFile out));
#endif
# 声明函数 gz_uncompress，接受两个参数：gzFile 类型的 in 和 FILE* 类型的 out
void gz_uncompress    OF((gzFile in, FILE   *out));
# 声明函数 file_compress，接受两个参数：char* 类型的 file 和 char* 类型的 mode
void file_compress    OF((char  *file, char *mode));
# 声明函数 file_uncompress，接受一个参数：char* 类型的 file
void file_uncompress  OF((char  *file));
# 声明函数 main，接受两个参数：int 类型的 argc 和 char* 类型的 argv[]
int  main             OF((int argc, char *argv[]));
/* ===========================================================================
 * 显示错误消息并退出
 */
void error(msg)
    const char *msg;
{
    fprintf(stderr, "%s: %s\n", prog, msg);
    exit(1);
}

/* ===========================================================================
 * 将输入压缩到输出，然后关闭两个文件。
 */
void gz_compress(in, out)
    FILE   *in;
    gzFile out;
{
    local char buf[BUFLEN];
    int len;
    int err;

#ifdef USE_MMAP
    /* 首先尝试使用 mmap 进行压缩。如果 mmap 失败（在管道中使用 minigzip），则使用正常的 fread 循环。
     */
    if (gz_compress_mmap(in, out) == Z_OK) return;
#endif
    for (;;) {
        len = (int)fread(buf, 1, sizeof(buf), in);
        if (ferror(in)) {
            perror("fread");
            exit(1);
        }
        if (len == 0) break;

        if (gzwrite(out, buf, (unsigned)len) != len) error(gzerror(out, &err));
    }
    fclose(in);
    if (gzclose(out) != Z_OK) error("failed gzclose");
}

#ifdef USE_MMAP /* MMAP 版本，Miguel Albrecht <malbrech@eso.org> */

/* 尝试使用 mmap 一次性压缩输入文件。如果成功返回 Z_OK，否则返回 Z_ERRNO。
 */
int gz_compress_mmap(in, out)
    FILE   *in;
    gzFile out;
{
    int len;
    int err;
    int ifd = fileno(in);
    caddr_t buf;    /* mmap 的缓冲区，用于整个输入文件 */
    off_t buf_len;  /* 输入文件的长度 */
    struct stat sb;

    /* 确定文件的大小，需要用于 mmap：
     */
    if (fstat(ifd, &sb) < 0) return Z_ERRNO;
    buf_len = sb.st_size;
    if (buf_len <= 0) return Z_ERRNO;

    /* 现在进行实际的 mmap：
     */
    buf = mmap((caddr_t) 0, buf_len, PROT_READ, MAP_SHARED, ifd, (off_t)0);
    if (buf == (caddr_t)(-1)) return Z_ERRNO;

    /* 一次性压缩整个文件：
     */
    len = gzwrite(out, (char *)buf, (unsigned)buf_len);

    if (len != (int)buf_len) error(gzerror(out, &err));

    munmap(buf, buf_len);
    fclose(in);
    # 如果关闭输出流失败，则输出错误信息
    if (gzclose(out) != Z_OK) error("failed gzclose");
    # 返回操作结果
    return Z_OK;
#endif /* USE_MMAP */

/* ===========================================================================
 * 将输入解压缩到输出，然后关闭两个文件。
 */
void gz_uncompress(in, out)
    gzFile in;
    FILE   *out;
{
    local char buf[BUFLEN];  // 本地字符缓冲区
    int len;  // 读取的长度
    int err;  // 错误码

    for (;;) {
        len = gzread(in, buf, sizeof(buf));  // 从输入文件中读取数据到缓冲区
        if (len < 0) error (gzerror(in, &err));  // 如果读取出错，则报错
        if (len == 0) break;  // 如果读取长度为0，则跳出循环

        if ((int)fwrite(buf, 1, (unsigned)len, out) != len) {  // 将缓冲区的数据写入输出文件
            error("failed fwrite");  // 如果写入失败，则报错
        }
    }
    if (fclose(out)) error("failed fclose");  // 关闭输出文件，如果失败则报错

    if (gzclose(in) != Z_OK) error("failed gzclose");  // 关闭输入文件，如果失败则报错
}


/* ===========================================================================
 * 压缩给定的文件：创建相应的 .gz 文件并删除原始文件。
 */
void file_compress(file, mode)
    char  *file;
    char  *mode;
{
    local char outfile[MAX_NAME_LEN];  // 本地输出文件名
    FILE  *in;  // 输入文件指针
    gzFile out;  // 输出文件指针

    if (strlen(file) + strlen(GZ_SUFFIX) >= sizeof(outfile)) {  // 如果文件名过长，则报错
        fprintf(stderr, "%s: filename too long\n", prog);
        exit(1);
    }

#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
    snprintf(outfile, sizeof(outfile), "%s%s", file, GZ_SUFFIX);  // 使用 snprintf 创建输出文件名
#else
    strcpy(outfile, file);  // 复制文件名
    strcat(outfile, GZ_SUFFIX);  // 添加 .gz 后缀
#endif

    in = fopen(file, "rb");  // 以只读方式打开输入文件
    if (in == NULL) {  // 如果打开失败，则报错
        perror(file);
        exit(1);
    }
    out = gzopen(outfile, mode);  // 以指定模式打开输出文件
    if (out == NULL) {  // 如果打开失败，则报错
        fprintf(stderr, "%s: can't gzopen %s\n", prog, outfile);
        exit(1);
    }
    gz_compress(in, out);  // 调用压缩函数

    unlink(file);  // 删除原始文件
}


/* ===========================================================================
 * 解压缩给定的文件并删除原始文件。
 */
void file_uncompress(file)
    char  *file;
{
    local char buf[MAX_NAME_LEN];  // 本地缓冲区
    char *infile, *outfile;  // 输入文件名和输出文件名
    FILE  *out;  // 输出文件指针
    gzFile in;  // 输入文件指针
    z_size_t len = strlen(file);  // 文件名长度

    if (len + strlen(GZ_SUFFIX) >= sizeof(buf)) {  // 如果文件名过长，则报错
        fprintf(stderr, "%s: filename too long\n", prog);
        exit(1);
    # 代码块结束，表示函数定义结束
#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
    // 如果支持 snprintf 和 vsnprintf，则使用 snprintf 将文件名复制到 buf 中
    snprintf(buf, sizeof(buf), "%s", file);
#else
    // 否则使用 strcpy 将文件名复制到 buf 中
    strcpy(buf, file);
#endif

    // 如果文件名长度大于 SUFFIX_LEN 并且以 GZ_SUFFIX 结尾
    if (len > SUFFIX_LEN && strcmp(file+len-SUFFIX_LEN, GZ_SUFFIX) == 0) {
        // 将文件名赋值给 infile
        infile = file;
        // 将 buf 赋值给 outfile
        outfile = buf;
        // 将 outfile 的倒数第四个字符设为 '\0'，即去掉后缀
        outfile[len-3] = '\0';
    } else {
        // 否则将文件名赋值给 outfile
        outfile = file;
        // 将 buf 赋值给 infile
        infile = buf;
#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
        // 如果支持 snprintf 和 vsnprintf，则使用 snprintf 将 GZ_SUFFIX 追加到 buf 后面
        snprintf(buf + len, sizeof(buf) - len, "%s", GZ_SUFFIX);
#else
        // 否则使用 strcat 将 GZ_SUFFIX 追加到 infile 后面
        strcat(infile, GZ_SUFFIX);
#endif
    }
    // 以只读方式打开压缩文件
    in = gzopen(infile, "rb");
    // 如果打开失败，输出错误信息并退出程序
    if (in == NULL) {
        fprintf(stderr, "%s: can't gzopen %s\n", prog, infile);
        exit(1);
    }
    // 以只写方式打开输出文件
    out = fopen(outfile, "wb");
    // 如果打开失败，输出错误信息并退出程序
    if (out == NULL) {
        perror(file);
        exit(1);
    }

    // 解压缩文件
    gz_uncompress(in, out);

    // 删除输入文件
    unlink(infile);
}


/* ===========================================================================
 * Usage:  minigzip [-c] [-d] [-f] [-h] [-r] [-1 to -9] [files...]
 *   -c : write to standard output
 *   -d : decompress
 *   -f : compress with Z_FILTERED
 *   -h : compress with Z_HUFFMAN_ONLY
 *   -r : compress with Z_RLE
 *   -1 to -9 : compression level
 */

int main(argc, argv)
    int argc;
    char *argv[];
{
    // 是否将输出写入标准输出
    int copyout = 0;
    // 是否解压缩
    int uncompr = 0;
    // 压缩文件
    gzFile file;
    // 程序名，输出模式
    char *prog, outmode[20];

#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
    // 如果支持 snprintf 和 vsnprintf，则使用 snprintf 将 "wb6 " 复制到 outmode 中
    snprintf(outmode, sizeof(outmode), "%s", "wb6 ");
#else
    // 否则使用 strcpy 将 "wb6 " 复制到 outmode 中
    strcpy(outmode, "wb6 ");
#endif

    // 将程序名赋值给 prog
    prog = argv[0];
    // 获取程序名
    bname = strrchr(argv[0], '/');
    if (bname)
      bname++;
    else
      bname = argv[0];
    argc--, argv++;

    // 如果程序名是 "gunzip"，则设置解压缩标志
    if (!strcmp(bname, "gunzip"))
      uncompr = 1;
    // 如果程序名是 "zcat"，则设置输出标志和解压缩标志
    else if (!strcmp(bname, "zcat"))
      copyout = uncompr = 1;
    # 当命令行参数数量大于 0 时，执行循环
    while (argc > 0) {
      # 如果当前参数是 "-c"，则设置复制输出标志为 1
      if (strcmp(*argv, "-c") == 0)
        copyout = 1;
      # 如果当前参数是 "-d"，则设置解压缩标志为 1
      else if (strcmp(*argv, "-d") == 0)
        uncompr = 1;
      # 如果当前参数是 "-f"，则设置输出模式的第四位为 'f'
      else if (strcmp(*argv, "-f") == 0)
        outmode[3] = 'f';
      # 如果当前参数是 "-h"，则设置输出模式的第四位为 'h'
      else if (strcmp(*argv, "-h") == 0)
        outmode[3] = 'h';
      # 如果当前参数是 "-r"，则设置输出模式的第四位为 'R'
      else if (strcmp(*argv, "-r") == 0)
        outmode[3] = 'R';
      # 如果当前参数以 "-" 开头，且第二个字符是数字 1-9，且第三个字符是结束符，则设置输出模式的第三位为第二个字符
      else if ((*argv)[0] == '-' && (*argv)[1] >= '1' && (*argv)[1] <= '9' &&
               (*argv)[2] == 0)
        outmode[2] = (*argv)[1];
      # 如果以上条件都不满足，则跳出循环
      else
        break;
      # 参数数量减一，指针向后移动
      argc--, argv++;
    }
    # 如果输出模式的第四位是空格，则设置为 0
    if (outmode[3] == ' ')
        outmode[3] = 0;
    # 如果参数数量为 0
    if (argc == 0) {
        # 设置标准输入和标准输出为二进制模式
        SET_BINARY_MODE(stdin);
        SET_BINARY_MODE(stdout);
        # 如果解压缩标志为真
        if (uncompr) {
            # 打开标准输入的压缩文件流
            file = gzdopen(fileno(stdin), "rb");
            # 如果打开失败，报错
            if (file == NULL) error("can't gzdopen stdin");
            # 解压缩文件流到标准输出
            gz_uncompress(file, stdout);
        } else {
            # 打开标准输出的压缩文件流，使用指定的输出模式
            file = gzdopen(fileno(stdout), outmode);
            # 如果打开失败，报错
            if (file == NULL) error("can't gzdopen stdout");
            # 压缩标准输入到文件流
            gz_compress(stdin, file);
        }
    } else {
        # 如果不是压缩文件，则执行以下操作
        if (copyout) {
            # 如果需要复制输出，则设置标准输出为二进制模式
            SET_BINARY_MODE(stdout);
        }
        # 循环处理每个文件
        do {
            # 如果是未压缩文件
            if (uncompr) {
                # 如果需要复制输出，则以只读二进制模式打开文件
                file = gzopen(*argv, "rb");
                # 如果文件打开失败，则输出错误信息
                if (file == NULL)
                    fprintf(stderr, "%s: can't gzopen %s\n", prog, *argv);
                # 如果文件打开成功，则解压缩文件并输出到标准输出
                else
                    gz_uncompress(file, stdout);
            } else {
                # 如果需要复制输出
                if (copyout) {
                    # 以只读二进制模式打开文件
                    FILE * in = fopen(*argv, "rb");
                    # 如果文件打开失败，则输出错误信息
                    if (in == NULL) {
                        perror(*argv);
                    } else {
                        # 以指定模式打开标准输出，并返回文件指针
                        file = gzdopen(fileno(stdout), outmode);
                        # 如果文件打开失败，则输出错误信息
                        if (file == NULL) error("can't gzdopen stdout");
                        # 压缩输入文件并输出到标准输出
                        gz_compress(in, file);
                    }

                } else {
                    # 如果不需要复制输出，则压缩输入文件并输出到指定模式的文件
                    file_compress(*argv, outmode);
                }
            }
        } while (argv++, --argc);
    }
    # 返回执行结果
    return 0;
# 闭合前面的函数定义
```