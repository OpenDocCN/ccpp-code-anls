# `nmap\nbase\test\test-escape_windows_command_arg.c`

```
/*
Usage: test-escape_windows_command_arg.exe

This is a test program for the escape_windows_command_arg function from
nbase_str.c. Its code is strictly Windows-specific. Basically, it performs
escape_windows_command_arg on arrays of strings merging its results with spaces
and tests if an attempt to decode them with CommandLineToArgvW results in the
same strings.
*/

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "nbase.h"

#include <shellapi.h>

const char *TESTS[][5] = {
    { NULL },  // 空字符串数组
    {"", NULL},  // 单个空字符串数组
    {"", "", NULL},  // 两个空字符串数组
    {"1", "2", "3", "4", NULL},  // 包含数字的字符串数组
    {"a", "b", "c", NULL},  // 包含字母的字符串数组
    {"a b", "c", NULL},  // 包含空格的字符串数组
    {"a b c", NULL},  // 包含多个空格的字符串数组
    {"  a  b  c  ", NULL},  // 包含多个空格的字符串数组
    {"\"quote\"", NULL},  // 包含引号的字符串数组
    {"back\\slash", NULL},  // 包含反斜杠的字符串数组
    {"backslash at end\\", NULL},  // 包含以反斜杠结尾的字符串数组
    {"double\"\"quote", NULL},  // 包含双引号的字符串数组
    {" a\nb\tc\rd\ne", NULL},  // 包含换行符、制表符和回车符的字符串数组
    {"..\\test\\toupper.lua", NULL},  // 包含文件路径的字符串数组
    {"backslash at end\\", "som\\ething\"af\\te\\r", NULL},  // 包含多个字符串的字符串数组
    {"three\\\\\\backslashes", "som\\ething\"af\\te\\r", NULL},  // 包含多个字符串的字符串数组
    {"three\"\"\"quotes", "som\\ething\"af\\te\\r", NULL},  // 包含多个字符串的字符串数组
};

static LPWSTR utf8_to_wchar(const char *s)
{
    LPWSTR result;
    int size, ret;

    /* Get needed buffer size. */
    size = MultiByteToWideChar(CP_UTF8, 0, s, -1, NULL, 0);  // 获取转换为宽字符所需的缓冲区大小
    if (size == 0) {
        fprintf(stderr, "MultiByteToWideChar 1 failed: %d\n", GetLastError());  // 输出错误信息
        exit(1);  // 退出程序
    }
    result = (LPWSTR) malloc(sizeof(*result) * size);  // 分配内存空间
    ret = MultiByteToWideChar(CP_UTF8, 0, s, -1, result, size);  // 将 UTF-8 编码的字符串转换为宽字符
    if (ret == 0) {
        fprintf(stderr, "MultiByteToWideChar 2 failed: %d\n", GetLastError());  // 输出错误信息
        exit(1);  // 退出程序
    }

    return result;  // 返回转换后的宽字符
}

static char *wchar_to_utf8(const LPWSTR s)
{
    char *result;
    int size, ret;

    /* Get needed buffer size. */
    size = WideCharToMultiByte(CP_UTF8, 0, s, -1, NULL, 0, NULL, NULL);  // 获取转换为 UTF-8 编码所需的缓冲区大小
    if (size == 0) {
        fprintf(stderr, "WideCharToMultiByte 1 failed: %d\n", GetLastError());  // 输出错误信息
        exit(1);  // 退出程序
    }
    result = (char *) malloc(size);  // 分配内存空间
    # 使用 WideCharToMultiByte 函数将宽字符转换为多字节字符，使用 UTF-8 编码
    ret = WideCharToMultiByte(CP_UTF8, 0, s, -1, result, size, NULL, NULL);
    # 如果转换失败，输出错误信息并退出程序
    if (ret == 0) {
        fprintf(stderr, "WideCharToMultiByte 2 failed: %d\n", GetLastError());
        exit(1);
    }
    # 返回转换后的多字节字符结果
    return result;
static char **wchar_to_utf8_array(const LPWSTR a[], unsigned int len)
{
    char **result;  // 声明一个指向指针的指针result
    unsigned int i;  // 声明一个无符号整型变量i

    result = (char **) malloc(sizeof(*result) * len);  // 分配存储空间给result指针数组
    if (result == NULL)  // 如果分配失败，返回NULL
        return NULL;
    for (i = 0; i < len; i++)  // 遍历LPWSTR数组a，将每个元素转换成UTF-8编码存储到result数组中
        result[i] = wchar_to_utf8(a[i]);

    return result;  // 返回转换后的UTF-8编码数组
}

static unsigned int nullarray_length(const char *a[])
{
    unsigned int i;  // 声明一个无符号整型变量i

    for (i = 0; a[i] != NULL; i++)  // 遍历char指针数组a，直到遇到NULL为止
        ;

    return i;  // 返回数组的长度
}

static char *append(char *p, const char *s)
{
    size_t plen, slen;  // 声明两个size_t类型的变量plen和slen

    plen = strlen(p);  // 获取字符串p的长度
    slen = strlen(s);  // 获取字符串s的长度
    p = (char *) realloc(p, plen + slen + 1);  // 重新分配p的存储空间，用于存储拼接后的字符串
    if (p == NULL)  // 如果分配失败，返回NULL
        return NULL;

    return strncat(p, s, plen+slen);  // 将字符串s拼接到字符串p的末尾
}

/* Turns an array of strings into an escaped flat command line. */
static LPWSTR build_commandline(const char *args[], unsigned int len)
{
    unsigned int i;  // 声明一个无符号整型变量i
    char *result;  // 声明一个指向字符的指针result

    result = strdup("progname");  // 复制字符串"progname"给result
    for (i = 0; i < len; i++) {  // 遍历args数组
        result = append(result, " ");  // 在result末尾添加空格
        result = append(result, escape_windows_command_arg(args[i]));  // 在result末尾添加经过转义的args[i]字符串
    }

    return utf8_to_wchar(result);  // 将result转换成宽字符编码并返回
}

static int arrays_equal(const char **a, unsigned int alen, const char **b, unsigned int blen)
{
    unsigned int i;  // 声明一个无符号整型变量i

    if (alen != blen)  // 如果两个数组的长度不相等，返回0
        return 0;
    for (i = 0; i < alen; i++) {  // 遍历两个数组
        if (strcmp(a[i], b[i]) != 0)  // 如果对应位置的元素不相等，返回0
            return 0;
    }

    return 1;  // 如果两个数组完全相等，返回1
}

static char *format_array(const char **args, unsigned int len)
{
    char *result;  // 声明一个指向字符的指针result
    unsigned int i;  // 声明一个无符号整型变量i

    result = strdup("");  // 复制空字符串给result
    result = append(result, "{");  // 在result末尾添加左大括号
    for (i = 0; i < len; i++) {  // 遍历args数组
        if (i > 0)  // 如果不是第一个元素
            result = append(result, ", ");  // 在result末尾添加逗号和空格
        result = append(result, "[");  // 在result末尾添加左中括号
        result = append(result, args[i]);  // 在result末尾添加args[i]字符串
        result = append(result, "]");  // 在result末尾添加右中括号
    }
    result = append(result, "}");  // 在result末尾添加右大括号

    return result;  // 返回格式化后的数组字符串
}

static int run_test(const char *args[])
{
    LPWSTR *argvw;  // 声明一个LPWSTR指针数组argvw
    char **result;  // 声明一个指向指针的指针result
    int args_len, argvw_len, result_len;  // 声明三个整型变量args_len, argvw_len, result_len

    args_len = nullarray_length(args);  // 获取args数组的长度
    # 使用 build_commandline 函数构建命令行参数，并将结果存储在 argvw 中
    argvw = CommandLineToArgvW(build_commandline(args, args_len), &argvw_len);
    # 考虑到在 argvw 中添加了 argv[0]，需要调整结果数组的长度
    /* Account for added argv[0] in argvw. */
    result = wchar_to_utf8_array(argvw+1, argvw_len-1);
    result_len = argvw_len - 1;

    # 比较转换后的参数数组和原始参数数组是否相等
    if (arrays_equal((const char **) result, result_len, args, args_len)) {
        # 如果相等，打印 PASS 和原始参数数组
        printf("PASS %s\n", format_array(args, args_len));
        return 1;
    } else {
        # 如果不相等，打印 FAIL 和转换后的参数数组，以及期望的参数数组
        printf("FAIL got %s\n", format_array((const char **) result, result_len));
        printf("expected %s\n", format_array(args, args_len));
        return 0;
    }
// 主函数，接受命令行参数，返回测试结果
int main(int argc, char *argv[])
{
    // 声明变量，用于记录测试总数和通过的测试数
    unsigned int num_tests, num_passed;
    // 声明变量，用于循环计数
    unsigned int i;

    // 初始化测试总数和通过的测试数
    num_tests = 0;
    num_passed = 0;
    // 遍历测试数组，统计测试总数并运行测试
    for (i = 0; i < sizeof(TESTS) / sizeof(*TESTS); i++) {
        num_tests++;
        if (run_test(TESTS[i]))
            num_passed++;
    }

    // 输出测试结果
    printf("%ld / %ld tests passed.\n", num_passed, num_tests);

    // 根据测试结果返回相应的值
    return num_passed == num_tests ? 0 : 1;
}
```