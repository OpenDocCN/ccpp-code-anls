# `xmrig\src\3rdparty\hwloc\src\base64.c`

```cpp
/*
 * 版权所有 © 2012-2018 Inria。保留所有权利。
 * 请参阅顶层目录中的COPYING文件。
 *
 * 导入后的修改：
 * - 删除所有#if
 * - 更新原型
 * - 更新#include
 */

/* 在任何其他内容之前包含hwloc的配置
 * 以便正确启用扩展和功能
 */
#include "private/private.h"

/*    $OpenBSD: base64.c,v 1.5 2006/10/21 09:55:03 otto Exp $    */

/*
 * 版权所有（c）1996年由Internet Software Consortium。
 *
 * 在此授予使用、复制、修改和分发此软件的许可，无论是否收取费用，
 * 只要上述版权声明和本许可声明出现在所有副本中。
 *
 * 本软件按“原样”提供，Internet Software Consortium放弃
 * 关于本软件的所有担保，包括所有暗示的担保
 * 商品性和适用性。在任何情况下，Internet Software
 * Consortium对于使用本软件的结果不承担任何特殊、直接、间接或后果性的责任
 * 损害或任何损害，无论是因为使用、数据或
 * 利润的丧失，无论是在合同行为、疏忽或其他侵权行为中，
 * 与使用或执行本软件有关或与之有关。
 */
/*
 * 版权声明，授权使用、复制、修改和分发软件，但需包含版权声明和许可声明
 * IBM公司不对软件提供任何担保，也不对使用软件产生的任何损失负责
 */

/* OPENBSD ORIGINAL: lib/libc/net/base64.c */

// Base64编码表
static const char Base64[] =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
// Base64填充字符
static const char Pad64 = '=';

#include <stdlib.h>
#include <string.h>
#include <ctype.h>

// 将输入数据编码为Base64格式
int
hwloc_encode_to_base64(const char *src, size_t srclength, char *target, size_t targsize)
{
    // 输入数据长度
    size_t datalength = 0;
    // 输入数据缓冲区
    unsigned char input[3];
    // 输出数据缓冲区
    unsigned char output[4];
    // 循环计数变量
    unsigned int i;
    # 当源数据长度大于2时，执行循环
    while (2 < srclength) {
        # 从源数据中读取3个字节
        input[0] = *src++;
        input[1] = *src++;
        input[2] = *src++;
        srclength -= 3;

        # 对读取的3个字节进行Base64编码
        output[0] = input[0] >> 2;
        output[1] = ((input[0] & 0x03) << 4) + (input[1] >> 4);
        output[2] = ((input[1] & 0x0f) << 2) + (input[2] >> 6);
        output[3] = input[2] & 0x3f;

        # 如果目标数据长度超过预设大小，则返回错误
        if (datalength + 4 > targsize)
            return (-1);
        # 将编码后的数据存入目标数组
        target[datalength++] = Base64[output[0]];
        target[datalength++] = Base64[output[1]];
        target[datalength++] = Base64[output[2]];
        target[datalength++] = Base64[output[3];
    }

    # 处理剩余不足3个字节的情况
    /* Now we worry about padding. */
    if (0 != srclength) {
        /* Get what's left. */
        # 将剩余不足3个字节的部分填充为'\0'
        input[0] = input[1] = input[2] = '\0';
        for (i = 0; i < srclength; i++)
            input[i] = *src++;

        # 对剩余部分进行Base64编码
        output[0] = input[0] >> 2;
        output[1] = ((input[0] & 0x03) << 4) + (input[1] >> 4);
        output[2] = ((input[1] & 0x0f) << 2) + (input[2] >> 6);

        # 如果目标数据长度超过预设大小，则返回错误
        if (datalength + 4 > targsize)
            return (-1);
        # 将编码后的数据存入目标数组
        target[datalength++] = Base64[output[0]];
        target[datalength++] = Base64[output[1]];
        # 根据剩余字节数决定是否填充'='
        if (srclength == 1)
            target[datalength++] = Pad64;
        else
            target[datalength++] = Base64[output[2]];
        target[datalength++] = Pad64;
    }
    # 如果目标数据长度超过预设大小，则返回错误
    if (datalength >= targsize)
        return (-1);
    # 在目标数组末尾添加'\0'，表示结束
    target[datalength] = '\0';    /* Returned value doesn't count \0. */
    # 返回编码后的数据长度
    return (int)(datalength);
/* 跳过任何位置的空白字符。
   将从 base-64 数字开始的每四个字符转换为目标区域中的三个 8 位字节。
   它返回存储在目标位置的数据字节数，或者在出错时返回 -1。
 */
int
hwloc_decode_from_base64(char const *src, char *target, size_t targsize)
{
    unsigned int tarindex, state;
    int ch;
    char *pos;

    state = 0;
    tarindex = 0;

    while ((ch = *src++) != '\0') {
        if (isspace(ch))    /* 跳过任何位置的空白字符。 */
            continue;

        if (ch == Pad64)
            break;

        pos = strchr(Base64, ch);
        if (pos == 0)         /* 非 base64 字符。 */
            return (-1);

        switch (state) {
        case 0:
            if (target) {
                if (tarindex >= targsize)
                    return (-1);
                target[tarindex] = (char)(pos - Base64) << 2;
            }
            state = 1;
            break;
        case 1:
            if (target) {
                if (tarindex + 1 >= targsize)
                    return (-1);
                target[tarindex]   |=  (pos - Base64) >> 4;
                target[tarindex+1]  = ((pos - Base64) & 0x0f)
                            << 4 ;
            }
            tarindex++;
            state = 2;
            break;
        case 2:
            if (target) {
                if (tarindex + 1 >= targsize)
                    return (-1);
                target[tarindex]   |=  (pos - Base64) >> 2;
                target[tarindex+1]  = ((pos - Base64) & 0x03)
                            << 6;
            }
            tarindex++;
            state = 3;
            break;
        case 3:
            if (target) {
                if (tarindex >= targsize)
                    return (-1);
                target[tarindex] |= (pos - Base64);
            }
            tarindex++;
            state = 0;
            break;
        }
    }
    """
    我们已经完成了Base-64字符的解码。让我们看看是否以字节边界结束，和/或是否有错误的尾随字符。
    """

    if (ch == Pad64):        # 如果我们得到了填充字符
        ch = *src++        # 跳过它，获取下一个字符
        switch (state):
            case 0:        # 无效 = 在第一个位置
            case 1:        # 无效 = 在第二个位置
                return (-1)

            case 2:        # 有效，表示一个字节的信息
                # 跳过任意数量的空格
                for (; ch != '\0'; ch = *src++):
                    if (!isspace(ch)):
                        break
                # 确保有另一个尾随的 = 符号
                if (ch != Pad64):
                    return (-1)
                ch = *src++        # 跳过 =
                # 继续到 "单个尾随 =" 情况
                # 继续执行下面的代码
                # FALLTHROUGH

            case 3:        # 有效，表示两个字节的信息
                """
                我们知道这个字符是一个 =。在它之后有除了空白字符之外的任何内容吗？
                """
                for (; ch != '\0'; ch = *src++):
                    if (!isspace(ch)):
                        return (-1)

                """
                现在确保对于情况2和3，溢出到最后一个完整字节之外的“额外”位都是零。
                如果我们不检查它们，它们就会成为一个潜意识通道。
                """
                if (target && target[tarindex] != 0):
                    return (-1)
    else:
        """
        我们通过看到字符串的结尾来结束。确保我们没有残留的部分字节。
        """
        if (state != 0):
            return (-1)

    return (tarindex)
# 闭合前面的函数定义
```