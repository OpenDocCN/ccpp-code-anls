# `nmap\libssh2\src\comp.c`

```cpp
/*
 * 该部分是版权声明，规定了源代码和二进制形式的再分发和使用条件
 * 版权声明、条件列表和免责声明必须在源代码的再分发中保留
 * 二进制形式的再分发必须在文档和/或其他提供的材料中重现版权声明、条件列表和免责声明
 * 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，无论是出于任何责任理论的原因，即使已被告知可能发生此类损害
 */

#include "libssh2_priv.h"
#ifdef LIBSSH2_HAVE_ZLIB
#include <zlib.h>
#undef compress // 避免与 ZLIB 宏发生命名冲突
#endif

#include "comp.h"

/* ********
 * none *
 ******** */

/*
 * comp_method_none_comp
 *
 * Minimalist compression: Absolutely none
 */
static int
# 定义了一个名为comp_method_none_comp的函数，接受session, dest, dest_len, src, src_len, abstract参数
comp_method_none_comp(LIBSSH2_SESSION *session,
                      unsigned char *dest,
                      size_t *dest_len,
                      const unsigned char *src,
                      size_t src_len,
                      void **abstract)
{
    # 忽略未使用的参数
    (void) session;
    (void) abstract;
    (void) dest;
    (void) dest_len;
    (void) src;
    (void) src_len;

    # 返回0
    return 0;
}

/*
 * comp_method_none_decomp
 *
 * Minimalist decompression: Absolutely none
 */
# 定义了一个名为comp_method_none_decomp的静态函数，接受session, dest, dest_len, payload_limit, src, src_len, abstract参数
static int
comp_method_none_decomp(LIBSSH2_SESSION * session,
                        unsigned char **dest,
                        size_t *dest_len,
                        size_t payload_limit,
                        const unsigned char *src,
                        size_t src_len, void **abstract)
{
    # 忽略未使用的参数
    (void) session;
    (void) payload_limit;
    (void) abstract;
    # 将dest指向src，dest_len指向src_len
    *dest = (unsigned char *) src;
    *dest_len = src_len;
    # 返回0
    return 0;
}

# 定义了一个名为comp_method_none的静态常量，类型为LIBSSH2_COMP_METHOD
static const LIBSSH2_COMP_METHOD comp_method_none = {
    "none",  # 压缩方法名称
    0,       # 不真正压缩
    0,       # 在用户认证中不使用
    NULL,    # 未使用的指针
    comp_method_none_comp,  # 压缩函数
    comp_method_none_decomp,  # 解压函数
    NULL     # 未使用的指针
};

#ifdef LIBSSH2_HAVE_ZLIB
/* ********
 * zlib *
 ******** */

# 定义了一个名为comp_method_zlib_alloc的函数，接受opaque, items, size参数
static voidpf
comp_method_zlib_alloc(voidpf opaque, uInt items, uInt size)
{
    LIBSSH2_SESSION *session = (LIBSSH2_SESSION *) opaque;

    # 分配内存并返回
    return (voidpf) LIBSSH2_ALLOC(session, items * size);
}

# 定义了一个名为comp_method_zlib_free的函数，接受opaque, address参数
static void
comp_method_zlib_free(voidpf opaque, voidpf address)
{
    LIBSSH2_SESSION *session = (LIBSSH2_SESSION *) opaque;

    # 释放内存
    LIBSSH2_FREE(session, address);
}

# 定义了一个名为comp_method_zlib_init的函数，接受session, compr, abstract参数
static int
comp_method_zlib_init(LIBSSH2_SESSION * session, int compr,
                      void **abstract)
{
    z_stream *strm;
    int status;

    # 分配内存并返回
    strm = LIBSSH2_CALLOC(session, sizeof(z_stream));
    # 如果压缩/解压缩流为空，则返回内存分配错误
    if(!strm) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for "
                              "zlib compression/decompression");
    }

    # 将会话作为不透明数据传递给压缩/解压缩流
    strm->opaque = (voidpf) session;
    # 设置压缩/解压缩流的内存分配函数
    strm->zalloc = (alloc_func) comp_method_zlib_alloc;
    # 设置压缩/解压缩流的内存释放函数
    strm->zfree = (free_func) comp_method_zlib_free;
    # 如果需要压缩
    if(compr) {
        # 初始化压缩流
        status = deflateInit(strm, Z_DEFAULT_COMPRESSION);
    }
    # 如果需要解压缩
    else {
        # 初始化解压缩流
        status = inflateInit(strm);
    }

    # 如果初始化失败
    if(status != Z_OK) {
        # 释放压缩/解压缩流
        LIBSSH2_FREE(session, strm);
        # 输出未处理的 zlib 错误信息
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "unhandled zlib error %d", status);
        # 返回压缩错误
        return LIBSSH2_ERROR_COMPRESS;
    }
    # 将压缩/解压缩流传递给抽象指针
    *abstract = strm;

    # 返回无错误
    return LIBSSH2_ERROR_NONE;
}

/*
 * libssh2_comp_method_zlib_comp
 *
 * 压缩源数据到目标数据，无需分配内存。
 */
static int
comp_method_zlib_comp(LIBSSH2_SESSION *session,
                      unsigned char *dest,

                      /* dest_len 是一个指针，允许这个函数更新它的最终实际使用大小 */
                      size_t *dest_len,
                      const unsigned char *src,
                      size_t src_len,
                      void **abstract)
{
    z_stream *strm = *abstract;
    int out_maxlen = *dest_len;
    int status;

    strm->next_in = (unsigned char *) src;
    strm->avail_in = src_len;
    strm->next_out = dest;
    strm->avail_out = out_maxlen;

    status = deflate(strm, Z_PARTIAL_FLUSH);

    if((status == Z_OK) && (strm->avail_out > 0)) {
        *dest_len = out_maxlen - strm->avail_out;
        return 0;
    }

    _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                   "unhandled zlib compression error %d, avail_out",
                   status, strm->avail_out);
    return _libssh2_error(session, LIBSSH2_ERROR_ZLIB, "compression failure");
}

/*
 * libssh2_comp_method_zlib_decomp
 *
 * 解压源数据到目标数据，分配输出内存。
 */
static int
comp_method_zlib_decomp(LIBSSH2_SESSION * session,
                        unsigned char **dest,
                        size_t *dest_len,
                        size_t payload_limit,
                        const unsigned char *src,
                        size_t src_len, void **abstract)
{
    z_stream *strm = *abstract;
    /* 临时分配一个完整的数据块比一系列重新分配更好 */
    char *out;
    size_t out_maxlen = src_len;

    if(src_len <= SIZE_MAX / 4)
        out_maxlen = src_len * 4;
    else
        out_maxlen = payload_limit;

    /* 如果 strm 为空，则表示我们还没有初始化。 */
    # 如果压缩流为空，则返回未初始化的解压缩错误
    if(strm == NULL)
        return _libssh2_error(session, LIBSSH2_ERROR_COMPRESS,
                              "decompression uninitialized");;

    # 实际上它们永远不会小于这个值
    if(out_maxlen < 25)
        out_maxlen = 25;

    # 如果输出最大长度大于有效载荷限制，则将输出最大长度设置为有效载荷限制
    if(out_maxlen > payload_limit)
        out_maxlen = payload_limit;

    # 设置压缩流的输入为源数据的无符号字符指针，可用输入为源数据长度
    strm->next_in = (unsigned char *) src;
    strm->avail_in = src_len;
    # 设置压缩流的输出为分配的输出最大长度的内存，out指向压缩流的下一个输出位置，可用输出为输出最大长度
    strm->next_out = (unsigned char *) LIBSSH2_ALLOC(session, out_maxlen);
    out = (char *) strm->next_out;
    strm->avail_out = out_maxlen;
    # 如果没有分配到输出缓冲区，则返回分配解压缩缓冲区失败的错误
    if(!strm->next_out)
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate decompression buffer");

    # 循环直到全部解压或出现错误
    // 无限循环，用于解压缩数据
    for(;;) {
        int status;
        size_t out_ofs;
        char *newout;

        // 使用 Z_PARTIAL_FLUSH 模式解压缩数据
        status = inflate(strm, Z_PARTIAL_FLUSH);

        // 如果解压缩状态为 Z_OK
        if(status == Z_OK) {
            // 如果输出缓冲区还有剩余空间，则表示解压缩完成，跳出循环
            if(strm->avail_out > 0)
                /* status is OK and the output buffer has not been exhausted
                   so we're done */
                break;
        }
        // 如果解压缩状态为 Z_BUF_ERROR
        else if(status == Z_BUF_ERROR) {
            /* the input data has been exhausted so we are done */
            // 输入数据已经用尽，解压缩完成，跳出循环
            break;
        }
        // 如果解压缩状态为其它值，表示出现错误
        else {
            /* error state */
            // 释放内存，打印错误信息，返回解压缩失败的错误状态
            LIBSSH2_FREE(session, out);
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                           "unhandled zlib error %d", status);
            return _libssh2_error(session, LIBSSH2_ERROR_ZLIB,
                                  "decompression failure");
        }

        // 如果解压缩后的数据长度超出限制或者超出系统能够处理的最大长度
        if(out_maxlen > payload_limit || out_maxlen > SIZE_MAX / 2) {
            // 释放内存，返回解压缩阶段出现过度增长的错误状态
            LIBSSH2_FREE(session, out);
            return _libssh2_error(session, LIBSSH2_ERROR_ZLIB,
                                  "Excessive growth in decompression phase");
        }

        // 如果程序执行到这里，表示需要扩展输出缓冲区并重试解压缩
        out_ofs = out_maxlen - strm->avail_out;
        out_maxlen *= 2;
        newout = LIBSSH2_REALLOC(session, out, out_maxlen);
        if(!newout) {
            // 如果无法分配足够的内存，释放内存，返回内存分配失败的错误状态
            LIBSSH2_FREE(session, out);
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to expand decompression buffer");
        }
        out = newout;
        strm->next_out = (unsigned char *) out + out_ofs;
        strm->avail_out = out_maxlen - out_ofs;
    }

    // 将解压缩后的数据和长度返回
    *dest = (unsigned char *) out;
    *dest_len = out_maxlen - strm->avail_out;

    // 返回解压缩成功的状态
    return 0;
/* libssh2_comp_method_zlib_dtor
 * All done, no more compression for you
 */
static int
comp_method_zlib_dtor(LIBSSH2_SESSION *session, int compr, void **abstract)
{
    z_stream *strm = *abstract;

    if(strm) {
        if(compr)
            deflateEnd(strm); // 如果是压缩，结束压缩
        else
            inflateEnd(strm); // 如果是解压，结束解压
        LIBSSH2_FREE(session, strm); // 释放内存
    }

    *abstract = NULL; // 将指针置为空
    return 0; // 返回0
}

static const LIBSSH2_COMP_METHOD comp_method_zlib = {
    "zlib", // 压缩方法名称
    1, /* yes, this compresses */ // 表示支持压缩
    1, /* do compression during userauth */ // 在用户认证期间进行压缩
    comp_method_zlib_init, // 初始化压缩方法
    comp_method_zlib_comp, // 压缩方法
    comp_method_zlib_decomp, // 解压方法
    comp_method_zlib_dtor, // 结束压缩/解压方法
};

static const LIBSSH2_COMP_METHOD comp_method_zlib_openssh = {
    "zlib@openssh.com", // 压缩方法名称
    1, /* yes, this compresses */ // 表示支持压缩
    0, /* don't use compression during userauth */ // 在用户认证期间不进行压缩
    comp_method_zlib_init, // 初始化压缩方法
    comp_method_zlib_comp, // 压缩方法
    comp_method_zlib_decomp, // 解压方法
    comp_method_zlib_dtor, // 结束压缩/解压方法
};
#endif /* LIBSSH2_HAVE_ZLIB */

/* If compression is enabled by the API, then this array is used which then
   may allow compression if zlib is available at build time */
static const LIBSSH2_COMP_METHOD *comp_methods[] = {
#ifdef LIBSSH2_HAVE_ZLIB
    &comp_method_zlib, // 使用zlib压缩方法
    &comp_method_zlib_openssh, // 使用openssh的zlib压缩方法
#endif /* LIBSSH2_HAVE_ZLIB */
    &comp_method_none, // 使用无压缩方法
    NULL
};

/* If compression is disabled by the API, then this array is used */
static const LIBSSH2_COMP_METHOD *no_comp_methods[] = {
    &comp_method_none, // 使用无压缩方法
    NULL
};

const LIBSSH2_COMP_METHOD **
_libssh2_comp_methods(LIBSSH2_SESSION *session)
{
    if(session->flag.compress)
        return comp_methods; // 如果启用了压缩，返回支持压缩的方法数组
    else
        return no_comp_methods; // 如果未启用压缩，返回不支持压缩的方法数组
}
```