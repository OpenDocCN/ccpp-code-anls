# `nmap\libpcap\charconv.h`

```cpp
/*
 * 版权声明，版权所有
 * 1. 源代码的再发布和使用需要保留版权声明、条件列表和下面的免责声明
 * 2. 二进制形式的再发布需要在文档和/或其他提供的材料中重现版权声明、条件列表和下面的免责声明
 * 3. 所有提到本软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 4. 未经特定书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 * 
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他原因）引起的任何损害，版权所有者和贡献者均不承担责任。
 */
#ifndef charconv_h
#define charconv_h

#ifdef _WIN32
# 声明一个外部函数，将指定代码页的多字节字符串转换为 UTF-16 小端编码的宽字符字符串
extern wchar_t *cp_to_utf_16le(UINT codepage, const char *cp_string, DWORD flags);
# 声明一个外部函数，将 UTF-16 小端编码的宽字符字符串转换为指定代码页的多字节字符串
extern char *utf_16le_to_cp(UINT codepage, const wchar_t *utf16le_string);
# 声明一个函数，将 UTF-8 编码的字符串转换为系统默认代码页的多字节字符串，如果转换后的字符串超出系统默认代码页的范围，则截断多余部分
extern void utf_8_to_acp_truncated(char *);
# 结束 charconv.h 文件的定义
#endif

#endif /* charconv_h */
```