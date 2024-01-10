# `nmap\libz\gzwrite.c`

```
/* gzwrite.c -- zlib functions for writing gzip files
 * zlib函数用于写入gzip文件
 * 版权所有 (C) 2004-2019 Mark Adler
 * 分发和使用条件，请参阅zlib.h中的版权声明
 */

#include "gzguts.h"

/* 本地函数 */
本地函数
local int gz_init OF((gz_statep));
local int gz_comp OF((gz_statep, int));
local int gz_zero OF((gz_statep, z_off64_t));
local z_size_t gz_write OF((gz_statep, voidpc, z_size_t));

/* 为写入gzip文件初始化状态。通过将state->size设置为非零来标记初始化。内存分配失败时返回-1，成功时返回0。 */
local int gz_init(state)
    gz_statep state;
{
    int ret;
    z_streamp strm = &(state->strm);

    /* 分配输入缓冲区（为gzprintf分配双倍大小） */
    state->in = (unsigned char *)malloc(state->want << 1);
    if (state->in == NULL) {
        gz_error(state, Z_MEM_ERROR, "out of memory");
        return -1;
    }

    /* 只有在压缩时才需要输出缓冲区和压缩状态 */
    if (!state->direct) {
        /* 分配输出缓冲区 */
        state->out = (unsigned char *)malloc(state->want);
        if (state->out == NULL) {
            free(state->in);
            gz_error(state, Z_MEM_ERROR, "out of memory");
            return -1;
        }

        /* 分配压缩内存，设置为gzip压缩 */
        strm->zalloc = Z_NULL;
        strm->zfree = Z_NULL;
        strm->opaque = Z_NULL;
        ret = deflateInit2(strm, state->level, Z_DEFLATED,
                           MAX_WBITS + 16, DEF_MEM_LEVEL, state->strategy);
        if (ret != Z_OK) {
            free(state->out);
            free(state->in);
            gz_error(state, Z_MEM_ERROR, "out of memory");
            return -1;
        }
        strm->next_in = NULL;
    }

    /* 标记状态已初始化 */
    state->size = state->want;

    /* 如果压缩，则初始化写缓冲区 */
    # 如果状态中的 direct 属性为假
    if (!state->direct) {
        # 将输出缓冲区大小设置为状态中的大小
        strm->avail_out = state->size;
        # 将输出缓冲区的起始地址设置为状态中的输出地址
        strm->next_out = state->out;
        # 将状态中的 x 属性的下一个指针设置为输出缓冲区的起始地址
        state->x.next = strm->next_out;
    }
    # 返回 0
    return 0;
# 压缩 avail_in 和 next_in 中的内容，并写入输出文件
# 如果写入输出文件时出现错误，或者 gz_init() 分配内存失败，则返回 -1；否则返回 0
# flush 被假定为有效的 deflate() 刷新值
# 如果 flush 是 Z_FINISH，则重置 deflate() 状态以开始一个新的 gzip 流
# 如果 gz->direct 为真，则直接写入输出文件而不压缩，并忽略 flush
local int gz_comp(state, flush)
    # gz_statep state; 表示 gz_statep 类型的 state 变量
    gz_statep state;
    # int flush; 表示整型的 flush 变量
    int flush;
{
    # 定义变量 ret, writ, have, put, max，并初始化 max
    int ret, writ;
    unsigned have, put, max = ((unsigned)-1 >> 2) + 1;
    # 定义指针类型的 strm 变量，并指向 state->strm
    z_streamp strm = &(state->strm);

    # 如果这是第一次执行，则分配内存
    if (state->size == 0 && gz_init(state) == -1)
        return -1;

    # 如果 state->direct 为真，则直接写入
    if (state->direct) {
        while (strm->avail_in) {
            put = strm->avail_in > max ? max : strm->avail_in;
            writ = write(state->fd, strm->next_in, put);
            if (writ < 0) {
                gz_error(state, Z_ERRNO, zstrerror());
                return -1;
            }
            strm->avail_in -= (unsigned)writ;
            strm->next_in += writ;
        }
        return 0;
    }

    # 检查是否有挂起的重置
    if (state->reset) {
        # 如果没有要写入的数据，则不开始新的 gzip 成员
        if (strm->avail_in == 0)
            return 0;
        deflateReset(strm);
        state->reset = 0;
    }

    # 在提供的输入上运行 deflate()，直到不再产生输出
    ret = Z_OK;
    do {
        /* 如果输出缓冲区已满，或者在刷新时，或者在 Z_FINISH 时，但是直到 Z_STREAM_END 之前都不写出当前缓冲区的内容 */
        if (strm->avail_out == 0 || (flush != Z_NO_FLUSH &&
            (flush != Z_FINISH || ret == Z_STREAM_END))) {
            /* 当输出缓冲区中有数据时，将数据写入文件 */
            while (strm->next_out > state->x.next) {
                put = strm->next_out - state->x.next > (int)max ? max :
                      (unsigned)(strm->next_out - state->x.next);
                writ = write(state->fd, state->x.next, put);
                /* 如果写入失败，返回错误 */
                if (writ < 0) {
                    gz_error(state, Z_ERRNO, zstrerror());
                    return -1;
                }
                state->x.next += writ;
            }
            /* 如果输出缓冲区已满，重置输出缓冲区和指针 */
            if (strm->avail_out == 0) {
                strm->avail_out = state->size;
                strm->next_out = state->out;
                state->x.next = state->out;
            }
        }

        /* 压缩数据 */
        have = strm->avail_out;
        ret = deflate(strm, flush);
        /* 如果压缩出错，返回错误 */
        if (ret == Z_STREAM_ERROR) {
            gz_error(state, Z_STREAM_ERROR,
                      "internal error: deflate stream corrupt");
            return -1;
        }
        have -= strm->avail_out;
    } while (have);

    /* 如果完成了一个 deflate 流，允许另一个 deflate 流开始 */
    if (flush == Z_FINISH)
        state->reset = 1;

    /* 所有操作完成，没有错误 */
    return 0;
}
/* 压缩 len 个零到输出。在 gz_comp() 写入错误或内存分配失败时返回-1，成功时返回0。 */
local int gz_zero(state, len)
    gz_statep state;
    z_off64_t len;
{
    int first;
    unsigned n;
    z_streamp strm = &(state->strm);

    /* 消耗输入缓冲区中剩余的内容 */
    if (strm->avail_in && gz_comp(state, Z_NO_FLUSH) == -1)
        return -1;

    /* 压缩 len 个零（保证 len > 0） */
    first = 1;
    while (len) {
        n = GT_OFF(state->size) || (z_off64_t)state->size > len ?
            (unsigned)len : state->size;
        if (first) {
            memset(state->in, 0, n);
            first = 0;
        }
        strm->avail_in = n;
        strm->next_in = state->in;
        state->x.pos += n;
        if (gz_comp(state, Z_NO_FLUSH) == -1)
            return -1;
        len -= n;
    }
    return 0;
}

/* 从 buf 中写入 len 个字节到文件。返回写入的字节数。如果返回值小于 len，则表示出现错误。 */
local z_size_t gz_write(state, buf, len)
    gz_statep state;
    voidpc buf;
    z_size_t len;
{
    z_size_t put = len;

    /* 如果 len 为零，避免不必要的操作 */
    if (len == 0)
        return 0;

    /* 如果这是第一次执行，分配内存 */
    if (state->size == 0 && gz_init(state) == -1)
        return 0;

    /* 检查是否有寻址请求 */
    if (state->seek) {
        state->seek = 0;
        if (gz_zero(state, state->skip) == -1)
            return 0;
    }

    /* 对于较小的 len，将其复制到输入缓冲区，否则直接压缩 */
    # 如果输入数据长度小于状态中的缓冲区大小
    if (len < state->size) {
        # 将数据复制到输入缓冲区，当缓冲区满时进行压缩
        do {
            unsigned have, copy;

            # 如果输入流中没有可用数据，则将输入指针指向状态中的输入缓冲区
            if (state->strm.avail_in == 0)
                state->strm.next_in = state->in;
            # 计算已有数据的长度
            have = (unsigned)((state->strm.next_in + state->strm.avail_in) -
                              state->in);
            # 计算需要复制的数据长度
            copy = state->size - have;
            if (copy > len)
                copy = (unsigned)len;
            # 将数据复制到输入缓冲区
            memcpy(state->in + have, buf, copy);
            # 更新输入流中可用数据的长度和位置
            state->strm.avail_in += copy;
            state->x.pos += copy;
            buf = (const char *)buf + copy;
            len -= copy;
            # 如果还有剩余数据且压缩失败，则返回 0
            if (len && gz_comp(state, Z_NO_FLUSH) == -1)
                return 0;
        } while (len);
    }
    else {
        # 如果输入数据长度大于等于状态中的缓冲区大小
        # 将输入缓冲区中剩余的数据进行压缩
        if (state->strm.avail_in && gz_comp(state, Z_NO_FLUSH) == -1)
            return 0;

        # 直接将用户缓冲区的数据压缩到文件中
        state->strm.next_in = (z_const Bytef *)buf;
        do {
            unsigned n = (unsigned)-1;
            if (n > len)
                n = (unsigned)len;
            state->strm.avail_in = n;
            state->x.pos += n;
            if (gz_comp(state, Z_NO_FLUSH) == -1)
                return 0;
            len -= n;
        } while (len);
    }

    # 输入数据已经全部被缓冲或压缩
    return put;
/* -- see zlib.h -- */
// 定义了一个名为 gzwrite 的函数，用于向 gzip 文件中写入数据
int ZEXPORT gzwrite(file, buf, len)
    // 声明一个名为 file 的 gzFile 类型参数，用于表示要写入的 gzip 文件
    gzFile file;
    // 声明一个名为 buf 的 voidpc 类型参数，用于表示要写入的数据缓冲区
    voidpc buf;
    // 声明一个名为 len 的 unsigned 类型参数，用于表示要写入的数据长度
    unsigned len;
{
    // 声明一个名为 state 的 gz_statep 类型变量，用于表示 gzip 文件的内部结构
    gz_statep state;

    /* get internal structure */
    // 如果文件为空，则返回 0
    if (file == NULL)
        return 0;
    // 将 file 转换为 gz_statep 类型，赋值给 state
    state = (gz_statep)file;

    /* check that we're writing and that there's no error */
    // 检查是否正在写入以及是否没有错误，如果不是则返回 0
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return 0;

    /* since an int is returned, make sure len fits in one, otherwise return
       with an error (this avoids a flaw in the interface) */
    // 由于返回一个 int 值，确保 len 可以放入一个 int 中，否则返回错误
    if ((int)len < 0) {
        // 如果 len 无法放入 int 中，则返回错误
        gz_error(state, Z_DATA_ERROR, "requested length does not fit in int");
        return 0;
    }

    /* write len bytes from buf (the return value will fit in an int) */
    // 从 buf 中写入 len 字节的数据（返回值将适合于 int）
    return (int)gz_write(state, buf, len);
}

/* -- see zlib.h -- */
// 定义了一个名为 gzfwrite 的函数，用于向 gzip 文件中写入数据
z_size_t ZEXPORT gzfwrite(buf, size, nitems, file)
    // 声明一个名为 buf 的 voidpc 类型参数，用于表示要写入的数据缓冲区
    voidpc buf;
    // 声明一个名为 size 的 z_size_t 类型参数，用于表示每个数据项的大小
    z_size_t size;
    // 声明一个名为 nitems 的 z_size_t 类型参数，用于表示要写入的数据项的数量
    z_size_t nitems;
    // 声明一个名为 file 的 gzFile 类型参数，用于表示要写入的 gzip 文件
    gzFile file;
{
    // 声明一个名为 len 的 z_size_t 类型变量，用于表示要写入的数据总长度
    z_size_t len;
    // 声明一个名为 state 的 gz_statep 类型变量，用于表示 gzip 文件的内部结构
    gz_statep state;

    /* get internal structure */
    // 如果文件为空，则返回 0
    if (file == NULL)
        return 0;
    // 将 file 转换为 gz_statep 类型，赋值给 state
    state = (gz_statep)file;

    /* check that we're writing and that there's no error */
    // 检查是否正在写入以及是否没有错误，如果不是则返回 0
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return 0;

    /* compute bytes to read -- error on overflow */
    // 计算要写入的字节数，如果溢出则返回错误
    len = nitems * size;
    // 如果 size 不为 0 且 len / size 不等于 nitems，则返回错误
    if (size && len / size != nitems) {
        // 如果计算溢出，则返回错误
        gz_error(state, Z_STREAM_ERROR, "request does not fit in a size_t");
        return 0;
    }

    /* write len bytes to buf, return the number of full items written */
    // 将 len 字节写入 buf，返回写入的完整数据项的数量
    return len ? gz_write(state, buf, len) / size : 0;
}

/* -- see zlib.h -- */
// 定义了一个名为 gzputc 的函数，用于向 gzip 文件中写入一个字符
int ZEXPORT gzputc(file, c)
    // 声明一个名为 file 的 gzFile 类型参数，用于表示要写入的 gzip 文件
    gzFile file;
    // 声明一个名为 c 的 int 类型参数，用于表示要写入的字符
    int c;
{
    // 声明 have 为 unsigned 类型变量
    unsigned have;
    // 声明 buf 为包含一个 unsigned char 类型元素的数组
    unsigned char buf[1];
    // 声明 state 为 gz_statep 类型变量，用于表示 gzip 文件的内部结构
    gz_statep state;
    // 声明 strm 为 z_streamp 类型变量
    z_streamp strm;

    /* get internal structure */
    // 如果文件为空，则返回 -1
    if (file == NULL)
        return -1;
    // 将 file 转换为 gz_statep 类型，赋值给 state
    state = (gz_statep)file;
    // 将 state 的 strm 成员赋值给 strm
    strm = &(state->strm);

    /* check that we're writing and that there's no error */
    // 检查是否正在写入以及是否没有错误，如果不是则返回 -1
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return -1;
    # 检查是否有 seek 请求
    if (state->seek) {
        # 重置 seek 标志位
        state->seek = 0;
        # 如果有 seek 请求，则调用 gz_zero 函数跳过指定字节数
        if (gz_zero(state, state->skip) == -1)
            return -1;
    }

    # 尝试将数据写入输入缓冲区以提高速度（如果缓冲区未初始化，则 state->size == 0）
    if (state->size) {
        # 如果输入缓冲区为空，则将 strm->next_in 指向 state->in
        if (strm->avail_in == 0)
            strm->next_in = state->in;
        # 计算已有数据的长度
        have = (unsigned)((strm->next_in + strm->avail_in) - state->in);
        # 如果已有数据长度小于缓冲区大小，则将数据写入缓冲区
        if (have < state->size) {
            state->in[have] = (unsigned char)c;
            strm->avail_in++;
            state->x.pos++;
            return c & 0xff;
        }
    }

    # 缓冲区没有空间或未初始化，则调用 gz_write() 函数
    buf[0] = (unsigned char)c;
    # 调用 gz_write() 函数将数据写入缓冲区
    if (gz_write(state, buf, 1) != 1)
        return -1;
    return c & 0xff;
}

/* -- see zlib.h -- */
// 将字符串写入到 gzip 文件中
int ZEXPORT gzputs(file, s)
    gzFile file;
    const char *s;
{
    z_size_t len, put;
    gz_statep state;

    /* get internal structure */
    // 获取内部结构
    if (file == NULL)
        return -1;
    state = (gz_statep)file;

    /* check that we're writing and that there's no error */
    // 检查是否正在写入以及是否没有错误
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return -1;

    /* write string */
    // 写入字符串
    len = strlen(s);
    if ((int)len < 0 || (unsigned)len != len) {
        gz_error(state, Z_STREAM_ERROR, "string length does not fit in int");
        return -1;
    }
    put = gz_write(state, s, len);
    return put < len ? -1 : (int)len;
}

#if defined(STDC) || defined(Z_HAVE_STDARG_H)
#include <stdarg.h>

/* -- see zlib.h -- */
// 将格式化字符串写入到 gzip 文件中
int ZEXPORTVA gzvprintf(gzFile file, const char *format, va_list va)
{
    int len;
    unsigned left;
    char *next;
    gz_statep state;
    z_streamp strm;

    /* get internal structure */
    // 获取内部结构
    if (file == NULL)
        return Z_STREAM_ERROR;
    state = (gz_statep)file;
    strm = &(state->strm);

    /* check that we're writing and that there's no error */
    // 检查是否正在写入以及是否没有错误
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return Z_STREAM_ERROR;

    /* make sure we have some buffer space */
    // 确保有一些缓冲区空间
    if (state->size == 0 && gz_init(state) == -1)
        return state->err;

    /* check for seek request */
    // 检查是否有寻址请求
    if (state->seek) {
        state->seek = 0;
        if (gz_zero(state, state->skip) == -1)
            return state->err;
    }

    /* do the printf() into the input buffer, put length in len -- the input
       buffer is double-sized just for this function, so there is guaranteed to
       be state->size bytes available after the current contents */
    // 将 printf() 写入输入缓冲区，将长度放入 len -- 输入缓冲区仅为此函数而加倍，因此当前内容后保证有 state->size 字节可用
    if (strm->avail_in == 0)
        strm->next_in = state->in;
    next = (char *)(state->in + (strm->next_in - state->in) + strm->avail_in);
    next[state->size - 1] = 0;
#ifdef NO_vsnprintf
#  ifdef HAS_vsprintf_void
    (void)vsprintf(next, format, va);
    # 遍历数组，直到找到值为0的元素或者遍历完整个数组
    for (len = 0; len < state->size; len++)
        # 如果数组中当前位置的值为0，则跳出循环
        if (next[len] == 0) break;
# 如果不是标准 C，也没有包含 stdarg.h，那么使用下面的代码块
#else /* !STDC && !Z_HAVE_STDARG_H */

# 定义一个函数 gzprintf，接受不定数量的参数
int ZEXPORTVA gzprintf(file, format, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10,
                       a11, a12, a13, a14, a15, a16, a17, a18, a19, a20)
    gzFile file;  # 文件指针
    const char *format;  # 格式化字符串
    int a1, a2, a3, a4, a5, a6, a7, a8, a9, a10,  # 不定数量的参数
        a11, a12, a13, a14, a15, a16, a17, a18, a19, a20;
{
    unsigned len, left;  # 定义无符号整数 len 和 left
    char *next;  # 定义字符指针 next
    gz_statep state;  # 定义指向 gz_state 结构体的指针 state
    z_streamp strm;  # 定义指向 z_stream 结构体的指针 strm

    /* 获取内部结构 */
    if (file == NULL)  # 如果文件指针为空，返回流错误
        return Z_STREAM_ERROR;
    state = (gz_statep)file;  # 将文件指针转换为 gz_state 结构体指针
    strm = &(state->strm);  # 获取 state 结构体中的 strm 指针

    /* 检查是否真的可以在 int 类型中传递指针 */
    if (sizeof(int) != sizeof(void *))  # 如果 int 类型和指针类型大小不一致，返回流错误
        return Z_STREAM_ERROR;

    /* 检查是否正在写入并且没有错误 */
    if (state->mode != GZ_WRITE || state->err != Z_OK)  # 如果不是写入模式或者存在错误，返回流错误
        return Z_STREAM_ERROR;

    /* 确保有一些缓冲区空间 */
    # 如果状态的大小为0并且初始化失败，则返回错误代码
    if (state->size == 0 && gz_init(state) == -1)
        return state->error;

    # 检查是否有寻址请求
    if (state->seek) {
        state->seek = 0;
        # 如果有寻址请求，则跳过指定的字节数
        if (gz_zero(state, state->skip) == -1)
            return state->error;
    }

    # 将 printf() 的结果写入输入缓冲区，并将长度存入 len -- 输入缓冲区的大小是当前函数的两倍，所以保证当前内容后面有 state->size 字节的空间
    if (strm->avail_in == 0)
        strm->next_in = state->in;
    next = (char *)(strm->next_in + strm->avail_in);
    next[state->size - 1] = 0;
#ifdef NO_snprintf
#  ifdef HAS_sprintf_void
    // 如果没有定义 snprintf，但有定义 sprintf_void，则使用 sprintf 函数
    sprintf(next, format, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, a11, a12,
            a13, a14, a15, a16, a17, a18, a19, a20);
    // 计算字符串长度
    for (len = 0; len < size; len++)
        if (next[len] == 0)
            break;
#  else
    // 使用 sprintf 函数，并将结果赋值给 len
    len = sprintf(next, format, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, a11,
                  a12, a13, a14, a15, a16, a17, a18, a19, a20);
#  endif
#else
#  ifdef HAS_snprintf_void
    // 如果定义了 snprintf_void，则使用 snprintf 函数
    snprintf(next, state->size, format, a1, a2, a3, a4, a5, a6, a7, a8, a9,
             a10, a11, a12, a13, a14, a15, a16, a17, a18, a19, a20);
    // 计算字符串长度
    len = strlen(next);
#  else
    // 使用 snprintf 函数，并将结果赋值给 len
    len = snprintf(next, state->size, format, a1, a2, a3, a4, a5, a6, a7, a8,
                   a9, a10, a11, a12, a13, a14, a15, a16, a17, a18, a19, a20);
#  endif
#endif

    /* check that printf() results fit in buffer */
    // 检查 printf() 的结果是否适合缓冲区
    if (len == 0 || len >= state->size || next[state->size - 1] != 0)
        return 0;

    /* update buffer and position, compress first half if past that */
    // 更新缓冲区和位置，如果超过一半则压缩前半部分
    strm->avail_in += len;
    state->x.pos += len;
    if (strm->avail_in >= state->size) {
        left = strm->avail_in - state->size;
        strm->avail_in = state->size;
        if (gz_comp(state, Z_NO_FLUSH) == -1)
            return state->err;
        memmove(state->in, state->in + state->size, left);
        strm->next_in = state->in;
        strm->avail_in = left;
    }
    return (int)len;
}

#endif

/* -- see zlib.h -- */
// 定义 gzflush 函数
int ZEXPORT gzflush(file, flush)
    gzFile file;
    int flush;
{
    gz_statep state;

    /* get internal structure */
    // 获取内部结构
    if (file == NULL)
        return Z_STREAM_ERROR;
    state = (gz_statep)file;

    /* check that we're writing and that there's no error */
    // 检查是否正在写入且没有错误
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return Z_STREAM_ERROR;

    /* check flush parameter */
    // 检查 flush 参数
    if (flush < 0 || flush > Z_FINISH)
        return Z_STREAM_ERROR;

    /* check for seek request */
    // 检查是否有寻址请求
    # 如果状态中的 seek 标志为真，则执行以下操作
    if (state->seek) {
        # 将状态中的 seek 标志设为 0
        state->seek = 0;
        # 如果使用 gz_zero 函数对状态中的数据进行压缩，返回值为 -1，则返回状态中的错误信息
        if (gz_zero(state, state->skip) == -1)
            return state->err;
    }

    # 使用请求的 flush 参数对剩余的数据进行压缩
    (void)gz_comp(state, flush);
    # 返回状态中的错误信息
    return state->err;
}
/* -- see zlib.h -- */
# 设置压缩参数
int ZEXPORT gzsetparams(file, level, strategy)
    gzFile file;  // 压缩文件
    int level;  // 压缩级别
    int strategy;  // 压缩策略
{
    gz_statep state;  // 压缩文件状态
    z_streamp strm;  // 压缩流

    /* get internal structure */
    // 获取内部结构
    if (file == NULL)
        return Z_STREAM_ERROR;  // 如果文件为空，返回流错误
    state = (gz_statep)file;  // 将文件转换为压缩文件状态
    strm = &(state->strm);  // 获取压缩流

    /* check that we're writing and that there's no error */
    // 检查是否正在写入且没有错误
    if (state->mode != GZ_WRITE || state->err != Z_OK)
        return Z_STREAM_ERROR;  // 如果不是写入模式或有错误，返回流错误

    /* if no change is requested, then do nothing */
    // 如果没有请求更改，则不做任何操作
    if (level == state->level && strategy == state->strategy)
        return Z_OK;  // 如果压缩级别和策略与当前相同，返回OK

    /* check for seek request */
    // 检查是否有寻址请求
    if (state->seek) {
        state->seek = 0;  // 重置寻址标志
        if (gz_zero(state, state->skip) == -1)
            return state->err;  // 如果寻址失败，返回错误
    }

    /* change compression parameters for subsequent input */
    // 更改后续输入的压缩参数
    if (state->size) {
        /* flush previous input with previous parameters before changing */
        // 在更改之前使用先前的参数刷新先前的输入
        if (strm->avail_in && gz_comp(state, Z_BLOCK) == -1)
            return state->err;  // 如果压缩失败，返回错误
        deflateParams(strm, level, strategy);  // 设置新的压缩参数
    }
    state->level = level;  // 更新压缩级别
    state->strategy = strategy;  // 更新压缩策略
    return Z_OK;  // 返回OK
}

/* -- see zlib.h -- */
# 关闭写入模式的压缩文件
int ZEXPORT gzclose_w(file)
    gzFile file;  // 压缩文件
{
    int ret = Z_OK;  // 返回值，默认为OK
    gz_statep state;  // 压缩文件状态

    /* get internal structure */
    // 获取内部结构
    if (file == NULL)
        return Z_STREAM_ERROR;  // 如果文件为空，返回流错误
    state = (gz_statep)file;  // 将文件转换为压缩文件状态

    /* check that we're writing */
    // 检查是否正在写入
    if (state->mode != GZ_WRITE)
        return Z_STREAM_ERROR;  // 如果不是写入模式，返回流错误

    /* check for seek request */
    // 检查是否有寻址请求
    if (state->seek) {
        state->seek = 0;  // 重置寻址标志
        if (gz_zero(state, state->skip) == -1)
            ret = state->err;  // 如果寻址失败，更新返回值为错误
    }

    /* flush, free memory, and close file */
    // 刷新、释放内存和关闭文件
    if (gz_comp(state, Z_FINISH) == -1)
        ret = state->err;  // 如果压缩失败，更新返回值为错误
    if (state->size) {
        if (!state->direct) {
            (void)deflateEnd(&(state->strm));  // 结束压缩
            free(state->out);  // 释放输出内存
        }
        free(state->in);  // 释放输入内存
    }
    gz_error(state, Z_OK, NULL);  // 设置错误状态为OK
    free(state->path);  // 释放路径内存
    # 如果关闭文件描述符失败，返回错误码
    if (close(state->fd) == -1)
        ret = Z_ERRNO;
    # 释放状态结构体内存
    free(state);
    # 返回操作结果
    return ret;
# 闭合前面的函数定义
```