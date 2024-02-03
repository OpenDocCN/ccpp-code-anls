# `nmap\libz\gzread.c`

```cpp
# gzread.c -- 用于读取 gzip 文件的 zlib 函数
# 版权所有 2004-2017 Mark Adler
# 有关分发和使用条件，请参阅 zlib.h 中的版权声明

# 包含 gzguts.h 文件
#include "gzguts.h"

# 本地函数
# 使用 read() 函数加载缓冲区 -- 在错误时返回 -1，否则返回 0。从 state->fd 读取，并根据需要更新 state->eof、state->err 和 state->msg。
# 由于 read() 不能保证读取请求的字节数，因此此函数需要在 read() 上循环。
local int gz_load OF((gz_statep, unsigned char *, unsigned, unsigned *));
{
    int ret;
    unsigned get, max = ((unsigned)-1 >> 2) + 1;

    *have = 0;
    do {
        get = len - *have;
        if (get > max)
            get = max;
        ret = read(state->fd, buf + *have, get);
        if (ret <= 0)
            break;
        *have += (unsigned)ret;
    } while (*have < len);
    if (ret < 0) {
        gz_error(state, Z_ERRNO, zstrerror());
        return -1;
    }
    if (ret == 0)
        state->eof = 1;
    return 0;
}

# 加载输入缓冲区，并在加载最后的数据时设置 eof 标志 -- 在错误时返回 -1，否则返回 0。
# 注意，即使在缓冲区中可能有未使用的数据，当到达输入文件的末尾时，eof 标志也会被设置。
# 一旦使用了该数据，将不会再尝试读取文件。
# 如果 strm->avail_in != 0，则当前数据将移动到输入缓冲区的开头，然后用输入文件中的可用数据加载缓冲区的其余部分。
local int gz_avail(state)
    gz_statep state;
{
    # 声明一个无符号整数变量 got
    unsigned got;
    # 声明一个指向 z_stream 结构体的指针 strm，并初始化为 state->strm 的地址
    z_streamp strm = &(state->strm);

    # 如果 state->err 不等于 Z_OK 并且不等于 Z_BUF_ERROR，则返回 -1
    if (state->err != Z_OK && state->err != Z_BUF_ERROR)
        return -1;
    # 如果 state->eof 等于 0
    if (state->eof == 0) {
        # 如果 strm->avail_in 不为 0，则将其内容复制到 state->in 的起始位置
        if (strm->avail_in) {       /* copy what's there to the start */
            unsigned char *p = state->in;
            unsigned const char *q = strm->next_in;
            unsigned n = strm->avail_in;
            do {
                *p++ = *q++;
            } while (--n);
        }
        # 调用 gz_load 函数，将 state->in + strm->avail_in 的内容加载到 state->in 中，最多加载 state->size - strm->avail_in 的内容，结果存储在 got 中
        if (gz_load(state, state->in + strm->avail_in,
                    state->size - strm->avail_in, &got) == -1)
            return -1;
        # 更新 strm->avail_in，增加 got 的值
        strm->avail_in += got;
        # 更新 strm->next_in，指向 state->in
        strm->next_in = state->in;
    }
    # 返回 0
    return 0;
# 查找 gzip 头部，为解压缩或复制做准备。state->x.have 必须为 0。
# 如果这是第一次进入该函数，分配所需的内存。
# 如果没有更多的输入数据可用，state->how 将保持不变，如果没有 gzip 头部并且将执行直接复制，则将设置为 COPY，或者如果要解压缩，则设置为 GZIP。
# 如果进行直接复制，则将从输入缓冲区中剩余的输入数据复制到输出缓冲区。在这种情况下，所有后续文件读取将直接到输出缓冲区或用户缓冲区。
# 如果进行解压缩，则将初始化 inflate 状态。
# gz_look() 成功返回 0，失败返回 -1。
local int gz_look(state)
    gz_statep state;
{
    z_streamp strm = &(state->strm);

    # 分配读取缓冲区和解压内存
    if (state->size == 0) {
        # 分配缓冲区
        state->in = (unsigned char *)malloc(state->want);
        state->out = (unsigned char *)malloc(state->want << 1);
        if (state->in == NULL || state->out == NULL) {
            free(state->out);
            free(state->in);
            gz_error(state, Z_MEM_ERROR, "out of memory");
            return -1;
        }
        state->size = state->want;

        # 分配解压内存
        state->strm.zalloc = Z_NULL;
        state->strm.zfree = Z_NULL;
        state->strm.opaque = Z_NULL;
        state->strm.avail_in = 0;
        state->strm.next_in = Z_NULL;
        if (inflateInit2(&(state->strm), 15 + 16) != Z_OK) {    # gunzip
            free(state->out);
            free(state->in);
            state->size = 0;
            gz_error(state, Z_MEM_ERROR, "out of memory");
            return -1;
        }
    }

    # 获取至少输入缓冲区中的魔术字节
    if (strm->avail_in < 2) {
        if (gz_avail(state) == -1)
            return -1;
        if (strm->avail_in == 0)
            return 0;
    }
}
    /* 检查是否存在 gzip 魔术字节 -- 如果存在，则进行 gzip 解码（注意：在考虑部分写入的 gzip 文件时存在逻辑困境，即如果只写入一个字节，我们无法确定这是一个单字节文件，还是一个部分写入的 gzip 文件 -- 在这里我们假设如果正在写入一个 gzip 文件，那么头部将在单个操作中被写入，因此读取单个字节足以表明它不是一个 gzip 文件） */
    if (strm->avail_in > 1 &&
            strm->next_in[0] == 31 && strm->next_in[1] == 139) {
        // 重置解压缩状态
        inflateReset(strm);
        state->how = GZIP;
        state->direct = 0;
        return 0;
    }

    /* 没有 gzip 头部 -- 如果之前正在解码 gzip，则这是尾部垃圾。忽略尾部垃圾并完成。 */
    if (state->direct == 0) {
        strm->avail_in = 0;
        state->eof = 1;
        state->x.have = 0;
        return 0;
    }

    /* 进行原始 I/O，将任何剩余的输入复制到输出 -- 这假设输出缓冲区大于输入缓冲区，这也确保了 gzungetc() 的空间 */
    state->x.next = state->out;
    memcpy(state->x.next, strm->next_in, strm->avail_in);
    state->x.have = strm->avail_in;
    strm->avail_in = 0;
    state->how = COPY;
    state->direct = 1;
    return 0;
# 从输入解压缩到状态中提供的 next_out 和 avail_out
# 在返回时，state->x.have 和 state->x.next 指向刚刚解压缩的数据
# 如果 gzip 流完成，state->how 被重置为 LOOK，以查找下一个 gzip 流或原始数据，一旦 state->x.have 被耗尽
# 成功返回 0，失败返回 -1
local int gz_decomp(state)
    gz_statep state;
{
    int ret = Z_OK;
    unsigned had;
    z_streamp strm = &(state->strm);

    # 填充输出缓冲区直到 deflate 流的末尾
    had = strm->avail_out;
    do {
        # 为 inflate() 获取更多输入
        if (strm->avail_in == 0 && gz_avail(state) == -1)
            return -1;
        if (strm->avail_in == 0) {
            gz_error(state, Z_BUF_ERROR, "unexpected end of file");
            break;
        }

        # 解压缩并处理错误
        ret = inflate(strm, Z_NO_FLUSH);
        if (ret == Z_STREAM_ERROR or ret == Z_NEED_DICT) {
            gz_error(state, Z_STREAM_ERROR, "internal error: inflate stream corrupt");
            return -1;
        }
        if (ret == Z_MEM_ERROR) {
            gz_error(state, Z_MEM_ERROR, "out of memory");
            return -1;
        }
        if (ret == Z_DATA_ERROR) {              # deflate 流无效
            gz_error(state, Z_DATA_ERROR, strm->msg == NULL ? "compressed data error" : strm->msg);
            return -1;
        }
    } while (strm->avail_out and ret != Z_STREAM_END);

    # 更新可用输出
    state->x.have = had - strm->avail_out;
    state->x.next = strm->next_out - state->x.have;

    # 如果 gzip 流成功完成，寻找另一个
    if (ret == Z_STREAM_END)
        state->how = LOOK;

    # 良好的解压缩
    return 0;
}
/* 从输入缓冲区获取数据并放入输出缓冲区。假设 state->x.have 为 0。
   数据要么从输入文件复制，要么从输入文件解压，取决于 state->how。
   如果 state->how 为 LOOK，则查找 gzip 头以确定是复制还是解压缩。
   出错时返回 -1，否则返回 0。
   gz_fetch() 会在处理完所有数据并到达输入文件末尾之前，将 state->how 保持为 COPY 或 GZIP。 */
local int gz_fetch(state)
    gz_statep state;
{
    z_streamp strm = &(state->strm);

    do {
        switch(state->how) {
        case LOOK:      /* -> LOOK, COPY (only if never GZIP), or GZIP */
            if (gz_look(state) == -1)
                return -1;
            if (state->how == LOOK)
                return 0;
            break;
        case COPY:      /* -> COPY */
            if (gz_load(state, state->out, state->size << 1, &(state->x.have))
                    == -1)
                return -1;
            state->x.next = state->out;
            return 0;
        case GZIP:      /* -> GZIP or LOOK (if end of gzip stream) */
            strm->avail_out = state->size << 1;
            strm->next_out = state->out;
            if (gz_decomp(state) == -1)
                return -1;
        }
    } while (state->x.have == 0 && (!state->eof || strm->avail_in));
    return 0;
}

/* 跳过 len 个未压缩字节的输出。出错时返回 -1，成功时返回 0。 */
local int gz_skip(state, len)
    gz_statep state;
    z_off64_t len;
{
    unsigned n;

    /* 跳过 len 个字节或到达文件末尾，以先到者为准 */
    # 当输出缓冲区中还有数据时，跳过输出缓冲区中的内容
    if (state->x.have) {
        # 计算需要跳过的数据长度
        n = GT_OFF(state->x.have) || (z_off64_t)state->x.have > len ?
            (unsigned)len : state->x.have;
        state->x.have -= n;
        state->x.next += n;
        state->x.pos += n;
        len -= n;
    }
    
    # 输出缓冲区为空时，如果已经到达输入的末尾，则结束循环
    else if (state->eof && state->strm.avail_in == 0)
        break;
    
    # 需要更多数据来跳过时，加载输出缓冲区
    else {
        # 获取更多的输出数据，如果需要的话查找头部
        if (gz_fetch(state) == -1)
            return -1;
    }
    # 返回 0 表示成功
    return 0;
# 从文件中读取长度为 len 的字节到缓冲区 buf 中，如果文件末尾不足 len 字节，则读取末尾剩余的字节。返回实际读取的字节数。
# 如果返回值为零，表示要么已经到达文件末尾，要么出现了错误。在这种情况下，必须查看 state->err 来确定具体是哪种情况。
local z_size_t gz_read(state, buf, len)
    # gz_statep 类型的 state 指针
    gz_statep state;
    # 用于存储读取的数据的缓冲区指针
    voidp buf;
    # 要读取的字节数
    z_size_t len;
{
    # 用于存储实际读取的字节数
    z_size_t got;
    # 用于临时存储变量
    unsigned n;

    # 如果 len 为零，避免不必要的操作，直接返回零
    if (len == 0)
        return 0;

    # 处理跳过请求
    if (state->seek) {
        state->seek = 0;
        # 如果调用 gz_skip 函数返回 -1，表示出现错误，直接返回零
        if (gz_skip(state, state->skip) == -1)
            return 0;
    }

    # 将 len 个字节读取到 buf 中，如果已经到达文件末尾，则读取末尾剩余的字节
    got = 0;
    /* 设置 n 为一个无符号整数能表示的最大长度 */
    n = (unsigned)-1;
    if (n > len)
        n = (unsigned)len;

    /* 首先尝试从输出缓冲区复制数据 */
    if (state->x.have) {
        if (state->x.have < n)
            n = state->x.have;
        memcpy(buf, state->x.next, n);
        state->x.next += n;
        state->x.have -= n;
    }

    /* 输出缓冲区为空 -- 如果已经到达输入的末尾，则返回 */
    else if (state->eof && state->strm.avail_in == 0) {
        state->past = 1;        /* 尝试读取超出末尾 */
        break;
    }

    /* 需要输出数据 -- 对于小的长度或新的流，加载输出缓冲区 */
    else if (state->how == LOOK || n < (state->size << 1)) {
        /* 获取更多输出，如果需要的话查找头部 */
        if (gz_fetch(state) == -1)
            return 0;
        continue;       /* 没有进展 -- 回到上面的复制 */
        /* 上面的复制确保我们将在输出缓冲区留有空间，至少允许一个 gzungetc() 成功 */
    }

    /* 大长度 -- 直接读入用户缓冲区 */
    else if (state->how == COPY) {      /* 直接读取 */
        if (gz_load(state, (unsigned char *)buf, n, &n) == -1)
            return 0;
    }

    /* 大长度 -- 直接解压缩到用户缓冲区 */
    else {  /* state->how == GZIP */
        state->strm.avail_out = n;
        state->strm.next_out = (unsigned char *)buf;
        if (gz_decomp(state) == -1)
            return 0;
        n = state->x.have;
        state->x.have = 0;
    }

    /* 更新进度 */
    len -= n;
    buf = (char *)buf + n;
    got += n;
    state->x.pos += n;
} while (len);

/* 返回读入用户缓冲区的字节数 */
return got;
# 从 zlib.h 文件中查看 gzread 函数的定义
int ZEXPORT gzread(file, buf, len)
    gzFile file;
    voidp buf;
    unsigned len;
{
    gz_statep state;  # 定义 gz_statep 结构体指针变量

    # 获取内部结构
    if (file == NULL)  # 如果文件为空，返回-1
        return -1;
    state = (gz_statep)file;  # 将文件转换为 gz_statep 结构体指针

    # 检查是否在读取模式，并且没有（严重的）错误
    if (state->mode != GZ_READ || (state->err != Z_OK && state->err != Z_BUF_ERROR))  # 如果不是读取模式，或者出现了严重错误，返回-1
        return -1;

    # 由于返回的是一个整数，确保 len 可以放入一个整数中，否则返回错误（避免接口中的缺陷）
    if ((int)len < 0)  # 如果 len 小于0，返回错误
        gz_error(state, Z_STREAM_ERROR, "request does not fit in an int");  # 报告错误
        return -1;

    # 读取 len 或更少的字节到 buf
    len = (unsigned)gz_read(state, buf, len);  # 读取 len 或更少的字节到 buf

    # 检查是否有错误
    if (len == 0 && state->err != Z_OK && state->err != Z_BUF_ERROR)  # 如果读取了0个字节，并且出现了严重错误，返回-1
        return -1;

    # 返回读取的字节数（确保可以放入一个整数中）
    return (int)len;  # 返回读取的字节数
}

# 从 zlib.h 文件中查看 gzfread 函数的定义
z_size_t ZEXPORT gzfread(buf, size, nitems, file)
    voidp buf;
    z_size_t size;
    z_size_t nitems;
    gzFile file;
{
    z_size_t len;  # 定义 z_size_t 类型的变量
    gz_statep state;  # 定义 gz_statep 结构体指针变量

    # 获取内部结构
    if (file == NULL)  # 如果文件为空，返回0
        return 0;
    state = (gz_statep)file;  # 将文件转换为 gz_statep 结构体指针

    # 检查是否在读取模式，并且没有（严重的）错误
    if (state->mode != GZ_READ || (state->err != Z_OK && state->err != Z_BUF_ERROR))  # 如果不是读取模式，或者出现了严重错误，返回0
        return 0;

    # 计算要读取的字节数，如果溢出则报错
    len = nitems * size;  # 计算要读取的字节数
    if (size && len / size != nitems)  # 如果溢出，报错
        gz_error(state, Z_STREAM_ERROR, "request does not fit in a size_t");  # 报告错误
        return 0;

    # 读取 len 或更少的字节到 buf，返回完整项目的数量
    return len ? gz_read(state, buf, len) / size : 0;  # 如果 len 不为0，则返回读取的字节数除以 size，否则返回0
}

# 从 zlib.h 文件中查看 gzgetc 函数的定义
#ifdef Z_PREFIX_SET
#  undef z_gzgetc
#else
#  undef gzgetc
#endif
int ZEXPORT gzgetc(file)
    gzFile file;
{
    unsigned char buf[1];  # 定义一个长度为1的无符号字符数组
    # 定义一个指向gz_statep结构体的指针变量state
    gz_statep state;

    # 获取文件的内部结构
    if (file == NULL)
        return -1;
    state = (gz_statep)file;

    # 检查我们是否在读取状态，并且没有（严重的）错误发生
    if (state->mode != GZ_READ ||
        (state->err != Z_OK && state->err != Z_BUF_ERROR))
        return -1;

    # 尝试输出缓冲区（无需检查是否有跳过请求）
    if (state->x.have) {
        state->x.have--;
        state->x.pos++;
        return *(state->x.next)++;
    }

    # 没有数据 -- 尝试gz_read()
    return gz_read(state, buf, 1) < 1 ? -1 : buf[0];
}
# 定义一个返回类型为 int，参数为 gzFile 类型的函数 gzgetc_
int ZEXPORT gzgetc_(file)
gzFile file;
{
    # 调用 gzgetc 函数并返回结果
    return gzgetc(file);
}

/* -- see zlib.h -- */
# 定义一个返回类型为 int，参数为 int 和 gzFile 类型的函数 gzungetc
int ZEXPORT gzungetc(c, file)
    int c;
    gzFile file;
{
    # 声明一个指向 gz_state 结构体的指针 state
    gz_statep state;

    /* get internal structure */
    # 如果 file 为空指针，则返回 -1
    if (file == NULL)
        return -1;
    # 将 file 强制类型转换为 gz_statep 类型，并赋值给 state
    state = (gz_statep)file;

    /* check that we're reading and that there's no (serious) error */
    # 检查当前模式是否为读取模式，以及是否没有（严重的）错误
    if (state->mode != GZ_READ ||
        (state->err != Z_OK && state->err != Z_BUF_ERROR))
        return -1;

    /* process a skip request */
    # 如果 state->seek 为真，则执行跳过操作
    if (state->seek) {
        state->seek = 0;
        # 如果执行跳过操作失败，则返回 -1
        if (gz_skip(state, state->skip) == -1)
            return -1;
    }

    /* can't push EOF */
    # 如果 c 小于 0，则返回 -1
    if (c < 0)
        return -1;

    /* if output buffer empty, put byte at end (allows more pushing) */
    # 如果输出缓冲区为空，则将字节放在末尾（允许更多推送）
    if (state->x.have == 0) {
        state->x.have = 1;
        state->x.next = state->out + (state->size << 1) - 1;
        state->x.next[0] = (unsigned char)c;
        state->x.pos--;
        state->past = 0;
        return c;
    }

    /* if no room, give up (must have already done a gzungetc()) */
    # 如果没有空间，则放弃（必须已经执行了 gzungetc()）
    if (state->x.have == (state->size << 1)) {
        gz_error(state, Z_DATA_ERROR, "out of room to push characters");
        return -1;
    }

    /* slide output data if needed and insert byte before existing data */
    # 如果需要，滑动输出数据并在现有数据之前插入字节
    if (state->x.next == state->out) {
        unsigned char *src = state->out + state->x.have;
        unsigned char *dest = state->out + (state->size << 1);
        while (src > state->out)
            *--dest = *--src;
        state->x.next = dest;
    }
    state->x.have++;
    state->x.next--;
    state->x.next[0] = (unsigned char)c;
    state->x.pos--;
    state->past = 0;
    return c;
}

/* -- see zlib.h -- */
# 定义一个返回类型为 char*，参数为 gzFile、char* 和 int 类型的函数 gzgets
char * ZEXPORT gzgets(file, buf, len)
    gzFile file;
    char *buf;
    int len;
{
    unsigned left, n;
    char *str;
    unsigned char *eol;
    gz_statep state;

    /* check parameters and get internal structure */
    # 检查参数并获取内部结构
    if (file == NULL || buf == NULL || len < 1)
        return NULL;
}
    # 将file强制类型转换为gz_statep类型，并赋值给state变量
    state = (gz_statep)file;

    # 检查是否处于读取模式，并且没有严重错误发生
    if (state->mode != GZ_READ ||
        (state->err != Z_OK && state->err != Z_BUF_ERROR))
        return NULL;

    # 处理跳过请求
    if (state->seek) {
        state->seek = 0;
        if (gz_skip(state, state->skip) == -1)
            return NULL;
    }

    # 复制输出字节直到新行或len-1，以先到者为准 --
    # 在字符串末尾添加一个终止零（我们不检查内容中是否有零，让用户自行处理）
    str = buf;
    left = (unsigned)len - 1;
    if (left) do {
        # 确保输出缓冲区中有内容
        if (state->x.have == 0 && gz_fetch(state) == -1)
            return NULL;                # 错误
        if (state->x.have == 0) {       # 文件结束
            state->past = 1;            # 读取超出末尾
            break;                      # 返回已有内容
        }

        # 在当前输出缓冲区中查找换行符
        n = state->x.have > left ? left : state->x.have;
        eol = (unsigned char *)memchr(state->x.next, '\n', n);
        if (eol != NULL)
            n = (unsigned)(eol - state->x.next) + 1;

        # 复制直到换行符，或者如果未找到，则复制剩余部分
        memcpy(buf, state->x.next, n);
        state->x.have -= n;
        state->x.next += n;
        state->x.pos += n;
        left -= n;
        buf += n;
    } while (left && eol == NULL);

    # 返回以零结尾的字符串，如果没有内容，则返回文件结束
    if (buf == str)
        return NULL;
    buf[0] = 0;
    return str;
}
/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个返回值为 int，参数为 gzFile 类型的函数 gzdirect
int ZEXPORT gzdirect(file)
    gzFile file;
{
    gz_statep state;

    /* get internal structure */
    # 获取内部结构
    if (file == NULL)
        return 0;
    state = (gz_statep)file;

    /* if the state is not known, but we can find out, then do so (this is
       mainly for right after a gzopen() or gzdopen()) */
    # 如果状态未知，但我们可以找出来，那么就这样做（这主要是在 gzopen() 或 gzdopen() 之后）
    if (state->mode == GZ_READ && state->how == LOOK && state->x.have == 0)
        (void)gz_look(state);

    /* return 1 if transparent, 0 if processing a gzip stream */
    # 如果是透明的则返回 1，如果正在处理 gzip 流则返回 0
    return state->direct;
}

/* -- see zlib.h -- */
# 根据 zlib.h 文件中的说明，定义了一个返回值为 int，参数为 gzFile 类型的函数 gzclose_r
int ZEXPORT gzclose_r(file)
    gzFile file;
{
    int ret, err;
    gz_statep state;

    /* get internal structure */
    # 获取内部结构
    if (file == NULL)
        return Z_STREAM_ERROR;
    state = (gz_statep)file;

    /* check that we're reading */
    # 检查我们是否在读取
    if (state->mode != GZ_READ)
        return Z_STREAM_ERROR;

    /* free memory and close file */
    # 释放内存并关闭文件
    if (state->size) {
        inflateEnd(&(state->strm));
        free(state->out);
        free(state->in);
    }
    err = state->err == Z_BUF_ERROR ? Z_BUF_ERROR : Z_OK;
    gz_error(state, Z_OK, NULL);
    free(state->path);
    ret = close(state->fd);
    free(state);
    return ret ? Z_ERRNO : err;
}
```