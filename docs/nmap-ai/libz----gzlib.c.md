# `nmap\libz\gzlib.c`

```cpp
/* gzlib.c -- zlib functions common to reading and writing gzip files
 * zlib函数，用于读写gzip文件的公共函数
 * 版权所有 (C) 2004-2019 Mark Adler
 * 分发和使用条件，请参阅zlib.h中的版权声明
 */

#include "gzguts.h"

#if defined(_WIN32) && !defined(__BORLANDC__)
#  define LSEEK _lseeki64
#else
#if defined(_LARGEFILE64_SOURCE) && _LFS64_LARGEFILE-0
#  define LSEEK lseek64
#else
#  define LSEEK lseek
#endif
#endif

/* 本地函数 */
/* 重置gzip文件状态 */
local void gz_reset OF((gz_statep));
local gzFile gz_open OF((const void *, int, const char *));

#if defined UNDER_CE

/* 将ERROR中的Windows错误号映射到与区域设置相关的错误消息字符串，并返回指向它的指针。
   通常，ERROR的值来自GetLastError。

   指向的字符串不得被应用程序修改，但可能会被后续对gz_strwinerror的调用覆盖

   gz_strwinerror函数不会更改GetLastError的当前设置。 */
char ZLIB_INTERNAL *gz_strwinerror(error)
     DWORD error;
{
    static char buf[1024];

    wchar_t *msgbuf;
    DWORD lasterr = GetLastError();
    DWORD chars = FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM
        | FORMAT_MESSAGE_ALLOCATE_BUFFER,
        NULL,
        error,
        0, /* 默认语言 */
        (LPVOID)&msgbuf,
        0,
        NULL);
    if (chars != 0) {
        /* 如果附加了\r\n，则将其删除。 */
        if (chars >= 2
            && msgbuf[chars - 2] == '\r' && msgbuf[chars - 1] == '\n') {
            chars -= 2;
            msgbuf[chars] = 0;
        }

        if (chars > sizeof (buf) - 1) {
            chars = sizeof (buf) - 1;
            msgbuf[chars] = 0;
        }

        wcstombs(buf, msgbuf, chars + 1);
        LocalFree(msgbuf);
    }
    else {
        sprintf(buf, "unknown win32 error (%ld)", error);
    }

    SetLastError(lasterr);
    return buf;
}

#endif /* UNDER_CE */

/* 重置gzip文件状态 */
local void gz_reset(state)
    gz_statep state;
{
    state->x.have = 0;              /* 设置输出数据为0，表示没有可用的输出数据 */
    if (state->mode == GZ_READ) {   /* 如果是读取模式... */
        state->eof = 0;             /* 文件尚未结束 */
        state->past = 0;            /* 尚未读取超过文件末尾的数据 */
        state->how = LOOK;          /* 寻找gzip头部 */
    }
    else                            /* 如果是写入模式... */
        state->reset = 0;           /* 没有待处理的deflateReset */
    state->seek = 0;                /* 没有寻找请求 */
    gz_error(state, Z_OK, NULL);    /* 清除错误 */
    state->x.pos = 0;               /* 尚未有未压缩的数据 */
    state->strm.avail_in = 0;       /* 尚未有输入数据 */
    # 定义一个函数，用于打开一个 gzip 文件，可以通过文件名或文件描述符来打开
    def gz_open(path, fd, mode):
        # 定义一个结构体指针 state，用于保存 gzip 文件的状态信息
        gz_statep state;
        # 定义变量 len 用于保存长度，oflag 用于保存打开文件的标志
        z_size_t len;
        int oflag;
        #ifdef O_CLOEXEC
        int cloexec = 0;
        #endif
        #ifdef O_EXCL
        int exclusive = 0;
        #endif

        # 检查输入参数是否为空
        if (path == NULL)
            return NULL;

        # 为 gzFile 结构分配内存空间
        state = (gz_statep)malloc(sizeof(gz_state));
        if (state == NULL)
            return NULL;
        state->size = 0;            /* 尚未分配缓冲区 */
        state->want = GZBUFSIZE;    /* 请求的缓冲区大小 */
        state->msg = NULL;          /* 尚未出现错误信息 */

        # 解释打开模式
        state->mode = GZ_NONE;
        state->level = Z_DEFAULT_COMPRESSION;
        state->strategy = Z_DEFAULT_STRATEGY;
        state->direct = 0;
        while (*mode):
            if (*mode >= '0' and *mode <= '9'):
                state->level = *mode - '0';
            else:
                switch (*mode):
                    case 'r':
                        state->mode = GZ_READ;
                        break;
                    #ifndef NO_GZCOMPRESS
                    case 'w':
                        state->mode = GZ_WRITE;
                        break;
                    case 'a':
                        state->mode = GZ_APPEND;
                        break;
                    #endif
                    case '+':       /* 不能同时读写 */
                        free(state);
                        return NULL;
                    case 'b':       /* 忽略 -- 无论如何都会请求二进制模式 */
                        break;
                    #ifdef O_CLOEXEC
                    case 'e':
                        cloexec = 1;
                        break;
                    #endif
                    #ifdef O_EXCL
                    case 'x':
                        exclusive = 1;
                        break;
#endif
            // 如果遇到 'f'，设置压缩策略为 Z_FILTERED
            case 'f':
                state->strategy = Z_FILTERED;
                break;
            // 如果遇到 'h'，设置压缩策略为 Z_HUFFMAN_ONLY
            case 'h':
                state->strategy = Z_HUFFMAN_ONLY;
                break;
            // 如果遇到 'R'，设置压缩策略为 Z_RLE
            case 'R':
                state->strategy = Z_RLE;
                break;
            // 如果遇到 'F'，设置压缩策略为 Z_FIXED
            case 'F':
                state->strategy = Z_FIXED;
                break;
            // 如果遇到 'T'，设置 direct 标志为 1
            case 'T':
                state->direct = 1;
                break;
            // 默认情况下，忽略并继续执行
            default:        /* could consider as an error, but just ignore */
                ;
            }
        mode++;
    }

    // 必须提供 "r", "w", 或 "a" 中的一个模式
    if (state->mode == GZ_NONE) {
        free(state);
        return NULL;
    }

    // 不能强制透明读取
    if (state->mode == GZ_READ) {
        if (state->direct) {
            free(state);
            return NULL;
        }
        state->direct = 1;      /* for empty file */
    }

    // 保存路径名以便错误消息使用
#ifdef WIDECHAR
    // 如果文件描述符为 -2，计算宽字符转换为多字节字符后的长度
    if (fd == -2) {
        len = wcstombs(NULL, path, 0);
        if (len == (z_size_t)-1)
            len = 0;
    }
    else
#endif
        // 否则，计算路径名的长度
        len = strlen((const char *)path);
    // 分配内存保存路径名
    state->path = (char *)malloc(len + 1);
    if (state->path == NULL) {
        free(state);
        return NULL;
    }
#ifdef WIDECHAR
    // 如果文件描述符为 -2，将宽字符转换为多字节字符
    if (fd == -2)
        if (len)
            wcstombs(state->path, path, len + 1);
        else
            *(state->path) = 0;
    else
#endif
#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
        // 使用 snprintf 将路径名复制到 state->path 中
        (void)snprintf(state->path, len + 1, "%s", (const char *)path);
#else
        // 否则，使用 strcpy 将路径名复制到 state->path 中
        strcpy(state->path, path);
#endif

    // 计算 open() 函数的标志位
    oflag =
#ifdef O_LARGEFILE
        O_LARGEFILE |
#endif
#ifdef O_BINARY
        O_BINARY |
#endif
#ifdef O_CLOEXEC
        (cloexec ? O_CLOEXEC : 0) |
#endif
        (state->mode == GZ_READ ?
         O_RDONLY :
         (O_WRONLY | O_CREAT |
#ifdef O_EXCL
          (exclusive ? O_EXCL : 0) |
#endif
          (state->mode == GZ_WRITE ?  # 如果模式是写入，则使用 O_TRUNC，否则使用 O_APPEND
           O_TRUNC :
           O_APPEND)));

    /* open the file with the appropriate flags (or just use fd) */
    state->fd = fd > -1 ? fd : (  # 如果 fd 大于 -1，则使用 fd，否则根据条件打开文件
#ifdef WIDECHAR
        fd == -2 ? _wopen(path, oflag, 0666) :  # 如果 fd 等于 -2，则使用 _wopen 打开文件，否则使用 open 打开文件
#endif
        open((const char *)path, oflag, 0666));
    if (state->fd == -1) {  # 如果文件打开失败，则释放内存并返回空指针
        free(state->path);
        free(state);
        return NULL;
    }
    if (state->mode == GZ_APPEND) {  # 如果模式是追加，则将文件指针移到文件末尾，并将模式设置为写入
        LSEEK(state->fd, 0, SEEK_END);  /* so gzoffset() is correct */
        state->mode = GZ_WRITE;         /* simplify later checks */
    }

    /* save the current position for rewinding (only if reading) */
    if (state->mode == GZ_READ) {  # 如果模式是读取，则保存当前位置作为重置时的起始位置
        state->start = LSEEK(state->fd, 0, SEEK_CUR);
        if (state->start == -1) state->start = 0;
    }

    /* initialize stream */
    gz_reset(state);  # 初始化流

    /* return stream */
    return (gzFile)state;  # 返回流对象
}

/* -- see zlib.h -- */
gzFile ZEXPORT gzopen(path, mode)  # 打开文件
    const char *path;
    const char *mode;
{
    return gz_open(path, -1, mode);  # 调用 gz_open 函数打开文件
}

/* -- see zlib.h -- */
gzFile ZEXPORT gzopen64(path, mode)  # 打开文件
    const char *path;
    const char *mode;
{
    return gz_open(path, -1, mode);  # 调用 gz_open 函数打开文件
}

/* -- see zlib.h -- */
gzFile ZEXPORT gzdopen(fd, mode)  # 打开文件
    int fd;
    const char *mode;
{
    char *path;         /* identifier for error messages */
    gzFile gz;

    if (fd == -1 || (path = (char *)malloc(7 + 3 * sizeof(int))) == NULL)  # 如果文件描述符为-1或分配内存失败，则返回空指针
        return NULL;
#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
    (void)snprintf(path, 7 + 3 * sizeof(int), "<fd:%d>", fd);  # 格式化文件描述符信息
#else
    sprintf(path, "<fd:%d>", fd);   /* for debugging */  # 格式化文件描述符信息（用于调试）
#endif
    gz = gz_open(path, fd, mode);  # 调用 gz_open 函数打开文件
    free(path);  # 释放内存
    return gz;  # 返回流对象
}

/* -- see zlib.h -- */
#ifdef WIDECHAR
gzFile ZEXPORT gzopen_w(path, mode)  # 打开文件
    const wchar_t *path;
    const char *mode;
{
    return gz_open(path, -2, mode);  # 调用 gz_open 函数打开文件
}
#endif

/* -- see zlib.h -- */
int ZEXPORT gzbuffer(file, size)  # 设置文件缓冲区大小
    gzFile file;
    unsigned size;
{
    gz_statep state;
    # 获取内部结构并检查完整性
    if (file == NULL)
        return -1;
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return -1;

    # 确保我们还没有分配内存
    if (state->size != 0)
        return -1;

    # 检查并设置请求的大小
    if ((size << 1) < size)
        return -1;              # 需要能够将其加倍
    if (size < 2)
        size = 2;               # 需要两个字节来检查魔术头部
    state->want = size;
    return 0;
}
/* -- see zlib.h -- */
// 重新定位到文件开头
int ZEXPORT gzrewind(file)
    // 定义一个名为 file 的 gzFile 类型参数
    gzFile file;
{
    // 定义一个指向 gz_state 结构的指针
    gz_statep state;

    /* get internal structure */
    // 如果文件为空，返回-1
    if (file == NULL)
        return -1;
    // 将 file 转换为 gz_statep 类型
    state = (gz_statep)file;

    /* check that we're reading and that there's no error */
    // 检查是否正在读取并且没有错误
    if (state->mode != GZ_READ ||
            (state->err != Z_OK && state->err != Z_BUF_ERROR))
        return -1;

    /* back up and start over */
    // 回退并重新开始
    if (LSEEK(state->fd, state->start, SEEK_SET) == -1)
        return -1;
    // 重置状态
    gz_reset(state);
    return 0;
}

/* -- see zlib.h -- */
// 64位偏移量的重新定位
z_off64_t ZEXPORT gzseek64(file, offset, whence)
    // 定义一个名为 file 的 gzFile 类型参数
    gzFile file;
    // 定义一个名为 offset 的 z_off64_t 类型参数
    z_off64_t offset;
    // 定义一个名为 whence 的整数类型参数
    int whence;
{
    // 定义无符号整数 n 和偏移量 ret
    unsigned n;
    z_off64_t ret;
    // 定义一个指向 gz_state 结构的指针
    gz_statep state;

    /* get internal structure and check integrity */
    // 获取内部结构并检查完整性
    if (file == NULL)
        return -1;
    // 将 file 转换为 gz_statep 类型
    state = (gz_statep)file;
    // 检查是否正在读取或写入
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return -1;

    /* check that there's no error */
    // 检查是否没有错误
    if (state->err != Z_OK && state->err != Z_BUF_ERROR)
        return -1;

    /* can only seek from start or relative to current position */
    // 只能从开头或相对于当前位置进行定位
    if (whence != SEEK_SET && whence != SEEK_CUR)
        return -1;

    /* normalize offset to a SEEK_CUR specification */
    // 将偏移量规范化为 SEEK_CUR 规范
    if (whence == SEEK_SET)
        offset -= state->x.pos;
    else if (state->seek)
        offset += state->skip;
    state->seek = 0;

    /* if within raw area while reading, just go there */
    // 如果在读取时在原始区域内，直接定位到那里
    if (state->mode == GZ_READ && state->how == COPY &&
            state->x.pos + offset >= 0) {
        ret = LSEEK(state->fd, offset - (z_off64_t)state->x.have, SEEK_CUR);
        if (ret == -1)
            return -1;
        state->x.have = 0;
        state->eof = 0;
        state->past = 0;
        state->seek = 0;
        gz_error(state, Z_OK, NULL);
        state->strm.avail_in = 0;
        state->x.pos += offset;
        return state->x.pos;
    }

    /* calculate skip amount, rewinding if needed for back seek when reading */
    // 计算跳过的数量，如果需要在读取时进行后退定位，则进行重绕
    # 如果偏移量小于0
    if (offset < 0) {
        # 如果是读取模式，不能向后移动
        if (state->mode != GZ_READ)         
            return -1;
        # 将偏移量加上当前位置，如果小于0，表示在文件开始之前
        offset += state->x.pos;
        if (offset < 0)                     
            return -1;
        # 回到文件开始，然后跳到指定偏移位置
        if (gzrewind(file) == -1)           
            return -1;
    }

    # 如果是读取模式，跳过输出缓冲区中的内容（少一个gzgetc()检查）
    if (state->mode == GZ_READ) {
        n = GT_OFF(state->x.have) || (z_off64_t)state->x.have > offset ?
            (unsigned)offset : state->x.have;
        state->x.have -= n;
        state->x.next += n;
        state->x.pos += n;
        offset -= n;
    }

    # 请求跳过（如果偏移量不为零）
    if (offset) {
        state->seek = 1;
        state->skip = offset;
    }
    # 返回当前位置加上偏移量
    return state->x.pos + offset;
/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gzseek 的函数，用于在 gzip 文件中定位到指定位置
z_off_t ZEXPORT gzseek(file, offset, whence)
    # 定义函数参数：gzip 文件对象，偏移量，定位方式
    gzFile file;
    z_off_t offset;
    int whence;
{
    # 定义返回值为 64 位偏移量
    z_off64_t ret;

    # 调用 gzseek64 函数，将返回值赋给 ret
    ret = gzseek64(file, (z_off64_t)offset, whence);
    # 如果返回值等于 ret 的类型转换值，则返回 ret，否则返回 -1
    return ret == (z_off_t)ret ? (z_off_t)ret : -1;
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gztell64 的函数，用于获取 gzip 文件的当前位置
z_off64_t ZEXPORT gztell64(file)
    # 定义函数参数：gzip 文件对象
    gzFile file;
{
    # 定义 gzip 文件的内部状态结构体指针
    gz_statep state;

    /* get internal structure and check integrity */
    # 如果文件对象为空，则返回 -1
    if (file == NULL)
        return -1;
    # 将文件对象转换为 gz_statep 类型，并检查完整性
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return -1;

    /* return position */
    # 返回文件的位置
    return state->x.pos + (state->seek ? state->skip : 0);
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gztell 的函数，用于获取 gzip 文件的当前位置
z_off_t ZEXPORT gztell(file)
    # 定义函数参数：gzip 文件对象
    gzFile file;
{
    # 定义返回值为 64 位偏移量
    z_off64_t ret;

    # 调用 gztell64 函数，将返回值赋给 ret
    ret = gztell64(file);
    # 如果返回值等于 ret 的类型转换值，则返回 ret，否则返回 -1
    return ret == (z_off_t)ret ? (z_off_t)ret : -1;
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gzoffset64 的函数，用于获取 gzip 文件的当前偏移量
z_off64_t ZEXPORT gzoffset64(file)
    # 定义函数参数：gzip 文件对象
    gzFile file;
{
    # 定义偏移量和 gzip 文件的内部状态结构体指针
    z_off64_t offset;
    gz_statep state;

    /* get internal structure and check integrity */
    # 如果文件对象为空，则返回 -1
    if (file == NULL)
        return -1;
    # 将文件对象转换为 gz_statep 类型，并检查完整性
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return -1;

    /* compute and return effective offset in file */
    # 计算并返回文件中的有效偏移量
    offset = LSEEK(state->fd, 0, SEEK_CUR);
    if (offset == -1)
        return -1;
    # 如果是读取模式，则减去缓冲区中的输入数据
    if (state->mode == GZ_READ)             /* reading */
        offset -= state->strm.avail_in;     /* don't count buffered input */
    return offset;
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gzoffset 的函数，用于获取 gzip 文件的当前偏移量
z_off_t ZEXPORT gzoffset(file)
    # 定义函数参数：gzip 文件对象
    gzFile file;
{
    # 定义返回值为 64 位偏移量
    z_off64_t ret;

    # 调用 gzoffset64 函数，将返回值赋给 ret
    ret = gzoffset64(file);
    # 如果返回值等于 ret 的类型转换值，则返回 ret，否则返回 -1
    return ret == (z_off_t)ret ? (z_off_t)ret : -1;
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个名为 gzeof 的函数，用于判断 gzip 文件是否已到达文件末尾
int ZEXPORT gzeof(file)
    # 定义函数参数：gzip 文件对象
    gzFile file;
{
    # 定义 gzip 文件的内部状态结构体指针
    gz_statep state;

    /* get internal structure and check integrity */
    # 如果文件对象为空，则返回 0
    if (file == NULL)
        return 0;
    # 将文件对象转换为 gz_statep 类型，并检查完整性
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return 0;

    /* return end-of-file state */
    # 返回文件是否已到达末尾的状态
    return state->mode == GZ_READ ? state->past : 0;
}
# 根据 zlib.h 查看，定义了一个返回错误信息的函数，接受一个 gzFile 对象和一个错误号指针作为参数
const char * ZEXPORT gzerror(file, errnum)
    gzFile file;
    int *errnum;
{
    # 声明一个指向 gz_state 结构体的指针
    gz_statep state;

    # 获取内部结构并检查完整性
    if (file == NULL)
        return NULL;
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return NULL;

    # 返回错误信息
    if (errnum != NULL)
        *errnum = state->err;
    return state->err == Z_MEM_ERROR ? "out of memory" :
                                       (state->msg == NULL ? "" : state->msg);
}

# 根据 zlib.h 查看，定义了一个清除错误标志的函数，接受一个 gzFile 对象作为参数
void ZEXPORT gzclearerr(file)
    gzFile file;
{
    # 声明一个指向 gz_state 结构体的指针
    gz_statep state;

    # 获取内部结构并检查完整性
    if (file == NULL)
        return;
    state = (gz_statep)file;
    if (state->mode != GZ_READ && state->mode != GZ_WRITE)
        return;

    # 清除错误和文件末尾标志
    if (state->mode == GZ_READ) {
        state->eof = 0;
        state->past = 0;
    }
    gz_error(state, Z_OK, NULL);
}

# 创建一个在分配内存中的错误消息，并相应地设置 state->err 和 state->msg。释放任何先前存在的错误消息。如果错误是 Z_MEM_ERROR（内存不足），则不要尝试释放或分配空间。只需将错误消息保存为静态字符串。如果构造错误消息时出现分配失败，则将错误转换为内存不足。
void ZLIB_INTERNAL gz_error(state, err, msg)
    gz_statep state;
    int err;
    const char *msg;
{
    # 释放先前分配的消息并清除
    if (state->msg != NULL) {
        if (state->err != Z_MEM_ERROR)
            free(state->msg);
        state->msg = NULL;
    }

    # 如果是致命错误，将 state->x.have 设置为 0，以便 gzgetc() 宏失败
    if (err != Z_OK && err != Z_BUF_ERROR)
        state->x.have = 0;

    # 设置错误代码，如果没有消息，则完成
    state->err = err;
    if (msg == NULL)
        return;
    # 如果发生内存错误，返回文字字符串
    if (err == Z_MEM_ERROR)
        return;

    # 构建带有路径的错误消息
    if ((state->msg = (char *)malloc(strlen(state->path) + strlen(msg) + 3)) ==
            NULL) {
        state->err = Z_MEM_ERROR;
        return;
    }
#if !defined(NO_snprintf) && !defined(NO_vsnprintf)
    # 如果未定义 NO_snprintf 和 NO_vsnprintf，则使用 snprintf 函数格式化输出错误信息
    (void)snprintf(state->msg, strlen(state->path) + strlen(msg) + 3,
                   "%s%s%s", state->path, ": ", msg);
#else
    # 否则，使用 strcpy 和 strcat 函数手动拼接错误信息
    strcpy(state->msg, state->path);
    strcat(state->msg, ": ");
    strcat(state->msg, msg);
#endif
}

#ifndef INT_MAX
/* 如果未定义 INT_MAX，则定义一个函数返回 int 类型的最大值
   这是为了覆盖一些情况下 limits.h 不可用的情况，因为 C 标准允许 1's 补码和符号位表示
   否则我们可以直接使用 ((unsigned)-1) >> 1 */
unsigned ZLIB_INTERNAL gz_intmax()
{
    unsigned p, q;

    p = 1;
    do {
        q = p;
        p <<= 1;
        p++;
    } while (p > q);
    return q >> 1;
}
#endif
```