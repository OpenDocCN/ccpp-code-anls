# `nmap\libpcap\charconv.c`

```
/*
 * 设置 C 语言的编辑模式和缩进格式
 */
/* -*- Mode: c; tab-width: 8; indent-tabs-mode: 1; c-basic-offset: 8; -*- */
/*
 * 版权声明
 */
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 */
/*
 * 在源代码和二进制形式中重新分发和使用，需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 */
/*
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保
 * 理事会或贡献者不对任何直接、间接、附带、特殊、惩罚性或后果性损害负责，包括但不限于替代商品或服务的采购、使用损失、数据或利润损失或业务中断
 * 无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害，也不承担任何责任
 */
#ifdef _WIN32
#include <stdio.h>
#include <errno.h>
#include <pcap/pcap.h>    /* Needed for PCAP_ERRBUF_SIZE */

#include "charconv.h"

wchar_t *
cp_to_utf_16le(UINT codepage, const char *cp_string, DWORD flags)
{
    int utf16le_len;
    wchar_t *utf16le_string;

    /*
     * Map from the specified code page to UTF-16LE.
     * First, find out how big a buffer we'll need.
     */
    utf16le_len = MultiByteToWideChar(codepage, flags, cp_string, -1,
        NULL, 0);
    if (utf16le_len == 0) {
        /*
         * Error.  Fail with EINVAL.
         */
        errno = EINVAL;
        return (NULL);
    }

    /*
     * Now attempt to allocate a buffer for that.
     */
    utf16le_string = malloc(utf16le_len * sizeof (wchar_t));
    if (utf16le_string == NULL) {
        /*
         * Not enough memory; assume errno has been
         * set, and fail.
         */
        return (NULL);
    }

    /*
     * Now convert.
     */
    utf16le_len = MultiByteToWideChar(codepage, flags, cp_string, -1,
        utf16le_string, utf16le_len);
    if (utf16le_len == 0) {
        /*
         * Error.  Fail with EINVAL.
         * XXX - should this ever happen, given that
         * we already ran the string through
         * MultiByteToWideChar() to find out how big
         * a buffer we needed?
         */
        free(utf16le_string);
        errno = EINVAL;
        return (NULL);
    }
    return (utf16le_string);
}

char *
utf_16le_to_cp(UINT codepage, const wchar_t *utf16le_string)
{
    int cp_len;
    char *cp_string;

    /*
     * Map from UTF-16LE to the specified code page.
     * First, find out how big a buffer we'll need.
     * We convert composite characters to precomposed characters,
     * as that's what Windows expects.
     */
    cp_len = WideCharToMultiByte(codepage, WC_COMPOSITECHECK,
        utf16le_string, -1, NULL, 0, NULL, NULL);
    if (cp_len == 0) {
        /*
         * Error.  Fail with EINVAL.
         */
        errno = EINVAL;
        return (NULL);
    }
    /*
     * 现在尝试为其分配一个缓冲区。
     */
    // 为 cp_string 分配内存，大小为 cp_len 乘以 char 类型的大小
    cp_string = malloc(cp_len * sizeof (char));
    // 如果分配失败，返回 NULL
    if (cp_string == NULL) {
        /*
         * 内存不足；假设 errno 已经被设置，然后失败。
         */
        return (NULL);
    }

    /*
     * 现在进行转换。
     */
    // 将 utf16le_string 转换为多字节字符，存储到 cp_string 中
    cp_len = WideCharToMultiByte(codepage, WC_COMPOSITECHECK,
        utf16le_string, -1, cp_string, cp_len, NULL, NULL);
    // 如果转换失败，释放内存，设置 errno 为 EINVAL，然后返回 NULL
    if (cp_len == 0) {
        /*
         * 错误。以 EINVAL 失败。
         * XXX - 在我们已经通过 WideCharToMultiByte() 运行字符串以找出需要多大的缓冲区之后，这种情况会发生吗？
         */
        free(cp_string);
        errno = EINVAL;
        return (NULL);
    }
    // 返回转换后的 cp_string
    return (cp_string);
/*
 * 将 UTF-8 编码的错误消息字符串转换为本地代码页，尽可能地进行转换。
 *
 * 假设缓冲区长度为 PCAP_ERRBUF_SIZE 字节；如果不适合，则进行截断。
 */
void
utf_8_to_acp_truncated(char *errbuf)
{
    wchar_t *utf_16_errbuf;
    int retval;
    DWORD err;

    /*
     * 通过先转换为 UTF-16LE，再转换为本地代码页来实现。
     * 这意味着我们可以使用 Microsoft 的转换例程，而不必自己理解所有的代码页，
     * 并且这个例程可以原地转换。
     */

    /*
     * 从 UTF-8 转换为 UTF-16LE。
     * 首先，找出我们需要多大的缓冲区。
     * 将任何无效字符转换为 REPLACEMENT CHARACTER。
     */
    utf_16_errbuf = cp_to_utf_16le(CP_UTF8, errbuf, 0);
    if (utf_16_errbuf == NULL) {
        /*
         * 出错。放弃。
         */
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "Can't convert error string to the local code page");
        return;
    }

    /*
     * 现在，将其转换为本地代码页。
     * 使用当前线程的代码页。对于无法转换的字符，让它选择“最佳匹配”字符。
     *
     * XXX - 如果缓冲区不够大，我们希望有一种方法来做到 utf_16le_to_utf_8_truncated() 的效果，
     * 但我们不想自己处理所有的本地代码页；这样做需要了解所有这些代码页，
     * 包括了解这些代码页中字符是如何形成的，以便我们可以避免将多字节字符切割成片段。
     *
     * 使用 Windows API 转换为非截断字符串，然后复制到缓冲区，仍然需要了解目标代码页中字符是如何形成的。
     */
    retval = WideCharToMultiByte(CP_THREAD_ACP, 0, utf_16_errbuf, -1,
        errbuf, PCAP_ERRBUF_SIZE, NULL, NULL);
}
    # 如果返回值为0，则表示成功
    if (retval == 0) {
        # 获取错误码
        err = GetLastError();
        # 释放内存
        free(utf_16_errbuf);
        # 如果错误码为ERROR_INSUFFICIENT_BUFFER，则说明错误字符串在本地代码页中不适合缓冲区
        if (err == ERROR_INSUFFICIENT_BUFFER)
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "The error string, in the local code page, didn't fit in the buffer");
        # 否则说明无法将错误字符串转换为本地代码页
        else
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "Can't convert error string to the local code page");
        # 返回
        return;
    }
    # 释放内存
    free(utf_16_errbuf);
}
# 结束条件编译指令的代码块
#endif
# 结束条件编译指令
```