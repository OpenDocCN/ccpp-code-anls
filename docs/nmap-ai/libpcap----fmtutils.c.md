# `nmap\libpcap\fmtutils.c`

```
/*
 * 版权声明，版权所有，保留所有权利
 *
 * 在源代码和二进制形式下的重新分发和修改是允许的，前提是满足以下条件：
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括在劳伦斯伯克利实验室的计算机系统工程组开发的软件。
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由权利人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的任何责任理论下，权利人或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，即使已被告知可能发生此类损害。
 */

/*
 * 用于 libpcap 和 rpcapd 都使用的消息格式化工具
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
#include "ftmacros.h"  // 包含自定义宏的头文件

#include <stddef.h>  // 包含标准库的头文件
#include <stdarg.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#include "pcap-int.h"  // 包含自定义 pcap 库的头文件

#include "portability.h"  // 包含自定义的可移植性头文件

#include "fmtutils.h"  // 包含自定义的格式化工具头文件

#ifdef _WIN32  // 如果是 Windows 系统
#include "charconv.h"  // 包含字符转换的头文件
#endif

/*
 * 设置编码。
 */
#ifdef _WIN32  // 如果是 Windows 系统
/*
 * 如果应该使用 UTF-8，则为真。
 */
static int use_utf_8;

void
pcap_fmt_set_encoding(unsigned int opts)  // 设置编码的函数
{
    if (opts == PCAP_CHAR_ENC_UTF_8)  // 如果选项是 UTF-8 编码
        use_utf_8 = 1;  // 设置使用 UTF-8 编码
}
#else  // 如果不是 Windows 系统
void
pcap_fmt_set_encoding(unsigned int opts _U_)  // 设置编码的函数
{
    /*
     * 这里不需要做任何事情。
     */
}
#endif

#ifdef _WIN32  // 如果是 Windows 系统
/*
 * 将以 null 结尾的 UTF-16LE 字符串转换为 UTF-8，并将其放入从指定位置开始的缓冲区中，如果超出指定大小则停止。这将只输出完整的 UTF-8 序列。
 *
 * 我们自己做这个是因为 Microsoft 没有提供“在空间不足时转换并在 UTF-8 字符边界停止”的例程。
 */
#define IS_LEADING_SURROGATE(c) \  // 定义宏，判断是否为 UTF-16LE 的 leading surrogate
    ((c) >= 0xd800 && (c) < 0xdc00)
#define IS_TRAILING_SURROGATE(c) \  // 定义宏，判断是否为 UTF-16LE 的 trailing surrogate
    ((c) >= 0xdc00 && (c) < 0xe000)
#define SURROGATE_VALUE(leading, trailing) \  // 定义宏，计算代理对的值
    (((((leading) - 0xd800) << 10) | ((trailing) - 0xdc00)) + 0x10000)
#define REPLACEMENT_CHARACTER    0x0FFFD  // 定义替换字符的值

static char *
utf_16le_to_utf_8_truncated(const wchar_t *utf_16, char *utf_8,  // 将 UTF-16LE 转换为 UTF-8，如果空间不足则截断
    size_t utf_8_len)
{
    wchar_t c, c2;
    uint32_t uc;

    if (utf_8_len == 0) {  // 如果没有足够的空间
        /*
         * 连一个结尾的 '\0' 都没有足够的空间。
         * 不要将任何东西放入缓冲区。
         */
        return (utf_8);  // 返回空指针
    }

    }
    /*
     * 确保至少有一个额外的空间用于存放字符串结尾的 '\0' 字符
     * （我们一开始就有足够的空间，这要归功于一开始对零长度缓冲区的测试，
     * 如果没有足够的空间来存放我们想要放入缓冲区的任何字符以及结尾的 '\0'，
     * 那么在将其放入缓冲区之前我们就会退出，因此会留下足够的空间来存放结尾的 '\0'。）
     *
     * 将 '\0' 放入缓冲区
     */
    *utf_8 = '\0';

    /*
     * 返回指向结尾的 '\0' 的指针，以防我们想在其后放入一些内容
     */
    return (utf_8);
}
#endif /* _WIN32 */

/*
 * 根据格式、参数和 errno 生成错误消息，格式化输出后再加上 errno 的消息。
 */
void
pcap_fmt_errmsg_for_errno(char *errbuf, size_t errbuflen, int errnum,
    const char *fmt, ...)
{
    va_list ap;

    // 初始化参数列表
    va_start(ap, fmt);
    // 调用带参数列表的函数 pcap_vfmt_errmsg_for_errno
    pcap_vfmt_errmsg_for_errno(errbuf, errbuflen, errnum, fmt, ap);
    // 结束参数列表
    va_end(ap);
}

/*
 * 根据格式、参数列表和 errno 生成错误消息。
 */
void
pcap_vfmt_errmsg_for_errno(char *errbuf, size_t errbuflen, int errnum,
    const char *fmt, va_list ap)
{
    size_t msglen;
    char *p;
    size_t errbuflen_remaining;

    // 格式化输出到 errbuf
    (void)vsnprintf(errbuf, errbuflen, fmt, ap);
    // 计算输出的字符串长度
    msglen = strlen(errbuf);

    /*
     * 是否有足够的空间来追加 ": "？
     * 包括终止的 '\0'，需要 3 个字节。
     */
    if (msglen + 3 > errbuflen) {
        /* 没有足够的空间 - 返回已生成的内容 */
        return;
    }
    p = errbuf + msglen;
    errbuflen_remaining = errbuflen - msglen;
    *p++ = ':';
    *p++ = ' ';
    *p = '\0';
    errbuflen_remaining -= 2;

    /*
     * 现在追加错误代码的字符串。
     */
#if defined(HAVE__WCSERROR_S)
    /*
     * 我们有一个 Windows 风格的 _wcserror_s()。
     * 生成一个 UTF-16LE 错误消息。
     */
    wchar_t utf_16_errbuf[PCAP_ERRBUF_SIZE];
    errno_t err = _wcserror_s(utf_16_errbuf, PCAP_ERRBUF_SIZE, errnum);
    if (err != 0) {
        /*
         * 似乎没有明显的地方记录 _wcserror_s() 的错误返回值。
         */
        snprintf(p, errbuflen_remaining, "Error %d", errnum);
        return;
    }

    /*
     * 现在将其从 UTF-16LE 转换为 UTF-8，放入缓冲区中的剩余空间，并在 UTF-8 字符边界上截断它 - 如果不适合的话。
     */
    utf_16le_to_utf_8_truncated(utf_16_errbuf, p, errbuflen_remaining);

    /*
     * 现在，如果我们不是在 UTF-8 模式下，将 errbuf 转换为本地代码页。
     */
    # 如果不使用 UTF-8 编码
    if (!use_utf_8)
        # 将 UTF-8 编码转换为 ACP 编码，并截断错误信息
        utf_8_to_acp_truncated(errbuf);
#elif defined(HAVE_GNU_STRERROR_R)
    /*
     * 如果定义了 HAVE_GNU_STRERROR_R，则表示我们有 GNU 风格的 strerror_r()，
     * 它不保证对传递给它的缓冲区做任何操作，并且返回一个指向错误字符串的指针，
     * 这个字符串可能在缓冲区中，也可能不在。
     *
     * 但是，它保证会成功。
     */
    char strerror_buf[PCAP_ERRBUF_SIZE];  // 创建一个缓冲区
    char *errstring = strerror_r(errnum, strerror_buf, PCAP_ERRBUF_SIZE);  // 调用 strerror_r() 函数获取错误字符串
    snprintf(p, errbuflen_remaining, "%s", errstring);  // 将错误字符串格式化并存储到指定的缓冲区中
#elif defined(HAVE_POSIX_STRERROR_R)
    /*
     * 如果定义了 HAVE_POSIX_STRERROR_R，则表示我们有 POSIX 风格的 strerror_r()，
     * 它保证会填充缓冲区，但不保证会成功。
     */
    int err = strerror_r(errnum, p, errbuflen_remaining);  // 调用 strerror_r() 函数获取错误字符串
    if (err == EINVAL) {
        /*
         * UNIX 03 规范表示这不保证会产生一个回退的错误消息。
         */
        snprintf(p, errbuflen_remaining, "Unknown error: %d", errnum);  // 格式化并存储未知错误消息到指定的缓冲区中
    } else if (err == ERANGE) {
        /*
         * UNIX 03 规范表示这不保证会产生一个回退的错误消息。
         */
        snprintf(p, errbuflen_remaining, "Message for error %d is too long", errnum);  // 格式化并存储错误消息过长的提示到指定的缓冲区中
    }
#else
    /*
     * 如果既没有 _wcserror_s() 也没有 strerror_r()，那么我们只能使用 pcap_strerror()。
     */
    snprintf(p, errbuflen_remaining, "%s", pcap_strerror(errnum));  // 使用 pcap_strerror() 获取错误消息并存储到指定的缓冲区中
#endif
}

#ifdef _WIN32
/*
 * 基于格式、参数和 Win32 错误生成错误消息，格式化输出后跟着 Win32 错误的消息。
 */
void
pcap_fmt_errmsg_for_win32_err(char *errbuf, size_t errbuflen, DWORD errnum,
    const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    pcap_vfmt_errmsg_for_win32_err(errbuf, errbuflen, errnum, fmt, ap);  // 调用 pcap_vfmt_errmsg_for_win32_err() 函数
    va_end(ap);
}

void
pcap_vfmt_errmsg_for_win32_err(char *errbuf, size_t errbuflen, DWORD errnum,
    const char *fmt, va_list ap)
{
    size_t msglen;
    char *p;
    size_t errbuflen_remaining;
    // 定义一个 DWORD 类型的变量 retval
    DWORD retval;
    // 定义一个大小为 PCAP_ERRBUF_SIZE 的 wchar_t 类型的数组 utf_16_errbuf
    wchar_t utf_16_errbuf[PCAP_ERRBUF_SIZE];
    // 定义一个 size_t 类型的变量 utf_8_len

    // 使用可变参数列表和格式化字符串将错误信息格式化到 errbuf 中
    vsnprintf(errbuf, errbuflen, fmt, ap);
    // 获取格式化后的错误信息长度
    msglen = strlen(errbuf);

    /*
     * Do we have enough space to append ": "?
     * Including the terminating '\0', that's 3 bytes.
     */
    // 检查是否有足够的空间来追加 ": "，包括终止符 '\0' 在内，需要 3 个字节
    if (msglen + 3 > errbuflen) {
        /* No - just give them what we've produced. */
        // 如果没有足够的空间，直接返回
        return;
    }
    // 指针 p 指向 errbuf 中错误信息的末尾
    p = errbuf + msglen;
    // 计算剩余的错误信息长度
    errbuflen_remaining = errbuflen - msglen;
    // 在错误信息末尾追加 ": "
    *p++ = ':';
    *p++ = ' ';
    *p = '\0';
    // 更新错误信息长度和剩余长度
    msglen += 2;
    errbuflen_remaining -= 2;

    /*
     * Now append the string for the error code.
     *
     * XXX - what language ID to use?
     *
     * For UN*Xes, pcap_strerror() may or may not return localized
     * strings.
     *
     * We currently don't have localized messages for libpcap, but we
     * might want to do so.  On the other hand, if most of these
     * messages are going to be read by libpcap developers and
     * perhaps by developers of libpcap-based applications, English
     * might be a better choice, so the developer doesn't have to
     * get the message translated if it's in a language they don't
     * happen to understand.
     */
    // 使用 FormatMessageW 函数获取错误码对应的字符串
    retval = FormatMessageW(FORMAT_MESSAGE_FROM_SYSTEM|FORMAT_MESSAGE_IGNORE_INSERTS|FORMAT_MESSAGE_MAX_WIDTH_MASK,
        NULL, errnum, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
        utf_16_errbuf, PCAP_ERRBUF_SIZE, NULL);
    // 如果获取失败，将错误信息格式化到 p 指向的位置
    if (retval == 0) {
        /*
         * Failed.
         */
        snprintf(p, errbuflen_remaining,
            "Couldn't get error message for error (%lu)", errnum);
        return;
    }

    /*
     * Now convert it from UTF-16LE to UTF-8.
     */
    // 将 UTF-16LE 格式的错误信息转换为 UTF-8
    p = utf_16le_to_utf_8_truncated(utf_16_errbuf, p, errbuflen_remaining);

    /*
     * Now append the error number, if it fits.
     */
    // 计算 UTF-8 格式的错误信息长度，并更新剩余长度
    utf_8_len = p - errbuf;
    errbuflen_remaining -= utf_8_len;
    # 如果 UTF-8 长度为 0，则消息为空
    if (utf_8_len == 0) {
        # 将错误号格式化到错误缓冲区中
        snprintf(p, errbuflen_remaining, "(%lu)", errnum);
    } else
        # 否则，将错误号格式化到错误缓冲区中
        snprintf(p, errbuflen_remaining, " (%lu)", errnum);

    /*
     * 现在，如果我们不是在 UTF-8 模式下，将 errbuf 转换为本地代码页。
     */
    # 如果不是使用 UTF-8 模式
    if (!use_utf_8)
        # 将 UTF-8 编码的错误缓冲区转换为本地代码页
        utf_8_to_acp_truncated(errbuf);
# 结束一个条件编译的代码块
#endif
```