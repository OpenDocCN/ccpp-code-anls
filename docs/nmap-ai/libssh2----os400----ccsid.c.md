# `nmap\libssh2\os400\ccsid.c`

```
/*
 * 版权声明
 * 版权所有（C）2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 *
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）责任，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害。
 */

/* 字符编码包装器 */

#include "libssh2_priv.h"  // 导入 libssh2_priv.h 头文件
#include "libssh2_ccsid.h"  // 导入 libssh2_ccsid.h 头文件

#include <qtqiconv.h>  // 导入 qtqiconv.h 头文件
#include <iconv.h>  // 导入 iconv.h 头文件
#include <errno.h>  // 导入 errno.h 头文件
#include <string.h>  // 导入 string.h 头文件
#include <stdio.h>  // 导入 stdio.h 头文件

#define CCSID_UTF8      1208  // 定义 CCSID_UTF8 常量为 1208
#define CCSID_UTF16BE   13488  // 定义 CCSID_UTF16BE 常量为 13488
#define STRING_GRANULE  256  // 定义 STRING_GRANULE 常量为 256
#define MAX_CHAR_SIZE   4  // 定义 MAX_CHAR_SIZE 常量为 4
# 定义一个宏，用于计算结构体中某个字段的偏移量
#define OFFSET_OF(t, f) ((size_t) ((char *) &((t *) 0)->f - (char *) 0))

# 定义 libssh2_string_cache 结构体
struct _libssh2_string_cache {
    libssh2_string_cache *  next;  # 指向下一个 libssh2_string_cache 结构体的指针
    char                    string[1];  # 存储字符串的数组，长度为1
};

# 定义一个常量 utf8code，类型为 QtqCode_T
static const QtqCode_T  utf8code = { CCSID_UTF8 };

# 定义一个函数 terminator_size，用于返回给定 CCSID 的空终止符大小
static ssize_t
terminator_size(unsigned short ccsid) {
    QtqCode_T outcode;  # 定义一个 QtqCode_T 类型的变量 outcode
    iconv_t cd;  # 定义一个 iconv_t 类型的变量 cd
    char *inp;  # 定义一个指向字符的指针变量 inp
    char *outp;  # 定义一个指向字符的指针变量 outp
    size_t ilen;  # 定义一个 size_t 类型的变量 ilen
    size_t olen;  # 定义一个 size_t 类型的变量 olen
    char buf[MAX_CHAR_SIZE];  # 定义一个大小为 MAX_CHAR_SIZE 的字符数组 buf

    # 返回给定 CCSID 的空终止符大小
    /* Return the null-terminator size for the given CCSID. */

    # 快速检查常见的 CCSID
    /* Fast check usual CCSIDs. */
    switch (ccsid) {
    case CCSID_UTF8:
    case 0:  # 作业 CCSID 是 SBCS EBCDIC
        return 1;
    case CCSID_UTF16BE:
        return 2;
    }

    # 将一个 UTF-8 的 NUL 转换为目标 CCSID：使用转换后的大小作为结果
    /* Convert an UTF-8 NUL to the target CCSID: use the converted size as result. */
    memset((void *) &outcode, 0, sizeof outcode);  # 将 outcode 变量清零
    outcode.CCSID = ccsid;  # 设置 outcode 变量的 CCSID 属性为给定的 CCSID
    cd = QtqIconvOpen(&outcode, (QtqCode_T *) &utf8code);  # 打开一个 iconv 转换句柄
    if (cd.return_value == -1)  # 如果转换句柄返回值为 -1
        return -1;
    inp = "";  # 设置输入指针指向空字符串
    ilen = 1;  # 设置输入长度为1
    outp = buf;  # 设置输出指针指向 buf 数组
    olen = sizeof buf;  # 设置输出长度为 buf 数组的大小
    iconv(cd, &inp, &ilen, &outp, &olen);  # 进行转换
    iconv_close(cd);  # 关闭转换句柄
    olen = sizeof buf - olen;  # 计算转换后的大小
    return olen? olen: -1;  # 如果转换后的大小大于0，则返回转换后的大小，否则返回-1
}

# 定义一个函数 convert_ccsid，用于将输入字符串从输入 CCSID 转换为输出 CCSID
static char *
convert_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
              unsigned short outccsid, unsigned short inccsid,
              const char *instring, ssize_t inlen, size_t *outlen) {
    char *inp;  # 定义一个指向字符的指针变量 inp
    char *outp;  # 定义一个指向字符的指针变量 outp
    size_t olen;  # 定义一个 size_t 类型的变量 olen
    size_t ilen;  # 定义一个 size_t 类型的变量 ilen
    size_t buflen;  # 定义一个 size_t 类型的变量 buflen
    size_t curlen;  # 定义一个 size_t 类型的变量 curlen
    ssize_t termsize;  # 定义一个 ssize_t 类型的变量 termsize
    int i;  # 定义一个整型变量 i
    char *dst;  # 定义一个指向字符的指针变量 dst
    libssh2_string_cache *outstring;  # 定义一个指向 libssh2_string_cache 结构体的指针变量 outstring
    QtqCode_T incode;  # 定义一个 QtqCode_T 类型的变量 incode
    QtqCode_T outcode;  # 定义一个 QtqCode_T 类型的变量 outcode
    iconv_t cd;  # 定义一个 iconv_t 类型的变量 cd

    if (!instring) {  # 如果输入字符串为空
        if (outlen)  # 如果输出长度不为空
            *outlen = 0;  # 将输出长度设置为0
        return NULL;  # 返回空指针
    }
    if (outlen)  # 如果输出长度不为空
        *outlen = -1;  # 将输出长度设置为-1
    if (!session || !cache)  # 如果会话或缓存为空
        return NULL;  # 返回空指针

    # 获取终止符大小
    /* Get terminator size. */
    termsize = terminator_size(outccsid);  # 调用 terminator_size 函数获取输出 CCSID 的终止符大小
    if (termsize < 0)  # 如果终止符大小小于0
        return NULL;  # 返回空指针

    # 准备转换参数
    /* Prepare conversion parameters. */
    # 将 incode 的内存清零，大小为 incode 的大小
    memset((void *) &incode, 0, sizeof incode);
    # 将 outcode 的内存清零，大小为 outcode 的大小
    memset((void *) &outcode, 0, sizeof outcode);
    # 设置 incode 的字符集编码为 inccsid
    incode.CCSID = inccsid;
    # 设置 outcode 的字符集编码为 outccsid
    outcode.CCSID = outccsid;
    # 设置 curlen 为 libssh2_string_cache 结构体中 string 字段的偏移量
    curlen = OFFSET_OF(libssh2_string_cache, string);
    # 将 inp 指向 instring，ilen 设置为 inlen
    inp = (char *) instring;
    ilen = inlen;
    # 设置 buflen 为 inlen 加上 curlen
    buflen = inlen + curlen;
    # 如果 inlen 小于 0
    if (inlen < 0) {
        # 设置 incode 的长度选项为 1
        incode.length_option = 1;
        # 设置 buflen 为 STRING_GRANULE
        buflen = STRING_GRANULE;
        # 设置 ilen 为 0
        ilen = 0;
    }

    # 分配输出字符串缓冲区并打开转换描述符
    dst = LIBSSH2_ALLOC(session, buflen + termsize);
    # 如果分配失败，返回空指针
    if (!dst)
        return NULL;
    # 打开转换描述符，将结果保存在 cd 中
    cd = QtqIconvOpen(&outcode, &incode);
    # 如果打开转换描述符失败
    if (cd.return_value == -1) {
        # 释放 dst 的内存
        LIBSSH2_FREE(session, (char *) dst);
        return NULL;
    }

    # 转换字符串
    for (;;) {
        # 设置 outp 指向 dst 加上 curlen，设置 olen 为 buflen 减去 curlen
        outp = dst + curlen;
        olen = buflen - curlen;
        # 调用 iconv 函数进行转换
        i = iconv(cd, &inp, &ilen, &outp, &olen);
        # 如果 inlen 小于 0 并且 olen 等于 buflen 减去 curlen
        if (inlen < 0 && olen == buflen - curlen) {
            # 特殊情况：转换为 0 长度的子字符串不存储终止符
            if (termsize) {
                # 将 outp 的内存清零，大小为 termsize
                memset(outp, 0, termsize);
                # 减去 termsize
                olen -= termsize;
            }
        }
        # 设置 curlen 为 buflen 减去 olen
        curlen = buflen - olen;
        # 如果 i 大于等于 0 或者 errno 不等于 E2BIG
        if (i >= 0 || errno != E2BIG)
            break;
        # 必须扩展缓冲区
        buflen += STRING_GRANULE;
        # 重新分配内存
        outp = LIBSSH2_REALLOC(session, dst, buflen + termsize);
        # 如果分配失败，跳出循环
        if (!outp)
            break;
        dst = outp;
    }

    # 关闭转换描述符
    iconv_close(cd);

    # 检查错误
    if (i < 0 || !outp) {
        # 释放 dst 的内存
        LIBSSH2_FREE(session, dst);
        return NULL;
    }

    # 处理终止符
    if (inlen < 0)
        curlen -= termsize;
    else if (termsize)
        # 将 dst 加上 curlen 的内存清零，大小为 termsize
        memset(dst + curlen, 0, termsize);

    # 如果 curlen 小于 buflen
    if (curlen < buflen)
        # 重新分配内存
        dst = LIBSSH2_REALLOC(session, dst, curlen + termsize);

    # 链接到缓存
    outstring = (libssh2_string_cache *) dst;
    outstring->next = *cache;
    *cache = outstring;
    /* 如果需要返回长度，则返回长度 */
    if (outlen)
        *outlen = curlen - OFFSET_OF(libssh2_string_cache, string);

    /* 返回字符串数据 */
    return outstring->string;
# 从指定的字符集编码转换为 UTF-8 编码
LIBSSH2_API char *
libssh2_from_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                   unsigned short ccsid, const char *string, ssize_t inlen,
                   size_t *outlen)
{
    return convert_ccsid(session, cache,
                         CCSID_UTF8, ccsid, string, inlen, outlen);
}

# 从 UTF-8 编码转换为指定的字符集编码
LIBSSH2_API char *
libssh2_to_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                 unsigned short ccsid, const char *string, ssize_t inlen,
                 size_t *outlen)
{
    return convert_ccsid(session, cache,
                         ccsid, CCSID_UTF8, string, inlen, outlen);
}

# 释放字符串缓存
LIBSSH2_API void
libssh2_release_string_cache(LIBSSH2_SESSION *session,
                             libssh2_string_cache **cache)
{
    libssh2_string_cache *p;

    # 如果会话和缓存都存在，则循环释放缓存中的字符串
    if (session && cache)
        while ((p = *cache)) {
            *cache = p->next;
            LIBSSH2_FREE(session, (char *) p);
        }
}

# 设置 vim 编辑器的选项
/* vim: set expandtab ts=4 sw=4: */
```