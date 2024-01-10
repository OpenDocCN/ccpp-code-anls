# `nmap\libz\uncompr.c`

```
# 定义了一个内部函数 uncompress2，用于将源缓冲区解压缩到目标缓冲区
# destLen 是目标缓冲区的总大小，必须足够大以容纳整个解压缩的数据
# sourceLen 是源缓冲区的字节长度
int ZEXPORT uncompress2(dest, destLen, source, sourceLen)
    Bytef *dest;  # 目标缓冲区
    uLongf *destLen;  # 目标缓冲区大小
    const Bytef *source;  # 源缓冲区
    uLong *sourceLen;  # 源缓冲区大小
{
    z_stream stream;  # 定义一个压缩流对象
    int err;  # 错误码
    const uInt max = (uInt)-1;  # 最大值
    uLong len, left;  # 长度和剩余空间
    Byte buf[1];  # 用于检测当 *destLen == 0 时的不完整流

    len = *sourceLen;  # 获取源缓冲区的长度
    if (*destLen) {  # 如果目标缓冲区大小不为0
        left = *destLen;  # 剩余空间为目标缓冲区大小
        *destLen = 0;  # 目标缓冲区大小置为0
    }
    else {  # 如果目标缓冲区大小为0
        left = 1;  # 剩余空间为1
        dest = buf;  # 目标缓冲区指向 buf
    }

    stream.next_in = (z_const Bytef *)source;  # 设置压缩流的输入缓冲区
    stream.avail_in = 0;  # 输入缓冲区可用字节数为0
    stream.zalloc = (alloc_func)0;  # 分配函数置为0
    stream.zfree = (free_func)0;  # 释放函数置为0
    stream.opaque = (voidpf)0;  # 不透明数据置为0

    err = inflateInit(&stream);  # 初始化解压缩流
    if (err != Z_OK) return err;  # 如果初始化失败，返回错误码

    stream.next_out = dest;  # 设置解压缩流的输出缓冲区
    stream.avail_out = 0;  # 输出缓冲区可用字节数为0
    # 循环执行以下操作，直到解压缩操作返回值不为 Z_OK
    do {
        # 如果输出缓冲区为空
        if (stream.avail_out == 0) {
            # 将输出缓冲区大小设置为剩余空间和最大值的较小值
            stream.avail_out = left > (uLong)max ? max : (uInt)left;
            # 更新剩余空间
            left -= stream.avail_out;
        }
        # 如果输入缓冲区为空
        if (stream.avail_in == 0) {
            # 将输入缓冲区大小设置为剩余长度和最大值的较小值
            stream.avail_in = len > (uLong)max ? max : (uInt)len;
            # 更新剩余长度
            len -= stream.avail_in;
        }
        # 执行解压缩操作
        err = inflate(&stream, Z_NO_FLUSH);
    } while (err == Z_OK);

    # 更新源数据长度
    *sourceLen -= len + stream.avail_in;
    # 如果目标缓冲区不等于 buf
    if (dest != buf)
        # 更新目标数据长度
        *destLen = stream.total_out;
    # 如果输出数据不为空且返回值为 Z_BUF_ERROR
    else if (stream.total_out && err == Z_BUF_ERROR)
        # 设置 left 为 1
        left = 1;

    # 结束解压缩操作
    inflateEnd(&stream);
    # 返回解压缩操作的结果
    return err == Z_STREAM_END ? Z_OK :
           err == Z_NEED_DICT ? Z_DATA_ERROR  :
           err == Z_BUF_ERROR && left + stream.avail_out ? Z_DATA_ERROR :
           err;
# 结束函数定义
}

# 定义一个名为uncompress的函数，参数为dest, destLen, source, sourceLen
int ZEXPORT uncompress(dest, destLen, source, sourceLen)
    # 定义一个指向Bytef类型的指针dest
    Bytef *dest;
    # 定义一个指向uLongf类型的指针destLen
    uLongf *destLen;
    # 定义一个指向const Bytef类型的指针source
    const Bytef *source;
    # 定义一个uLong类型的sourceLen
    uLong sourceLen;
{
    # 调用uncompress2函数，传入参数dest, destLen, source, &sourceLen，并返回结果
    return uncompress2(dest, destLen, source, &sourceLen);
}
```