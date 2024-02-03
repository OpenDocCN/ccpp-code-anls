# `nmap\ncat\test\test-cmdline-split.c`

```cpp
#include <errno.h>  // 包含错误码定义
#include <stdio.h>  // 包含标准输入输出函数
#include <stdlib.h>  // 包含标准库函数
#include <string.h>  // 包含字符串处理函数

static long test_count = 0;  // 定义测试计数变量
static long success_count = 0;  // 定义成功测试计数变量
char **cmdline_split(const char *cmdexec);  // 声明cmdline_split函数

int test_cmdline(const char *line, const char **target_args)  // 定义test_cmdline函数，传入参数line和target_args
{
    char **cmd_args;  // 定义指向字符指针的指针变量
    int args_match = 1;  // 定义参数匹配标志变量

    test_count++;  // 测试计数加一

    cmd_args = cmdline_split(line);  // 调用cmdline_split函数，将返回值赋给cmd_args

    /*
     * Make sure that all of the target arguments are have been extracted
     * by cmdline_split.
     */
    while (*cmd_args && *target_args) {  // 循环直到cmd_args或target_args为空
        if (strcmp(*cmd_args, *target_args)) {  // 比较cmd_args和target_args指向的字符串是否相等
            args_match = 0;  // 如果不相等，将参数匹配标志设为0
            break;  // 跳出循环
        }
        cmd_args++;  // 指向下一个cmd_args
        target_args++;  // 指向下一个target_args
    }
    if ((*cmd_args != NULL) || (*target_args != NULL)) {  // 如果cmd_args或target_args不为空
        /*
         * One of the argument list had more arguments than the other.
         * Therefore, they do not match
         */
        args_match = 0;  // 参数不匹配
    }

    if (args_match) {  // 如果参数匹配标志为真
        success_count++;  // 成功测试计数加一
        printf("PASS '%s'\n", line);  // 打印通过信息
        return 1;  // 返回1
    } else {
        printf("FAIL '%s'\n", line);  // 打印失败信息
        return 0;  // 返回0
    }
}

int test_cmdline_fail(const char *line)  // 定义test_cmdline_fail函数，传入参数line
{
    char **cmd_args;  // 定义指向字符指针的指针变量

    test_count++;  // 测试计数加一

    cmd_args = cmdline_split(line);  // 调用cmdline_split函数，将返回值赋给cmd_args

    if (*cmd_args == NULL) {  // 如果cmd_args指向的内容为空
        success_count++;  // 成功测试计数加一
        printf("PASS '%s'\n", line);  // 打印通过信息
        return 1;  // 返回1
    } else {
        printf("PASS '%s'\n", line);  // 打印通过信息
        return 0;  // 返回0
    }
}

int main(int argc, char *argv[])  // 定义主函数，传入参数argc和argv
{
    int i;  // 定义循环变量i

    struct {  // 定义结构体
        const char *cmdexec;  // 命令执行字符串
        const char *args[10];  // 参数数组
    // 定义测试用例数组，包含命令和预期参数数组
    } TEST_CASES[] = {
        {"ncat -l -k", {"ncat", "-l", "-k", NULL}},
        {"ncat localhost 793", {"ncat", "localhost", "793", NULL}},
        {"./ncat scanme.nmap.org 80", {"./ncat", "scanme.nmap.org", "80",
                                       NULL}},
        {"t\\ p\\ s hello world how are you?", {"t p s", "hello", "world", "how", "are",
                                              "you?", NULL}},
        {"t\\ p\\ s hello world how\\ are you?", {"t p s", "hello", "world", "how are",
                                               "you?", NULL}},
        {"ncat\\", {"ncat", NULL}},
        {"a\\nb", {"anb", NULL}},
        {" ncat a ", {"ncat", "a", NULL}},
        {"\\ncat \\a", {"ncat", "a", NULL}},
        {"ncat\\\\ a", {"ncat\\", "a", NULL}},
        {"ncat\\", {"ncat", NULL}},
        {"ncat\\ \\", {"ncat ", NULL}},
    };

    // 遍历测试用例数组，依次执行测试
    for (i = 0; i < sizeof(TEST_CASES)/sizeof(TEST_CASES[0]); i++) {
        test_cmdline(TEST_CASES[i].cmdexec,
                     TEST_CASES[i].args);
    }

    // 执行失败测试
    test_cmdline_fail("");
    // 输出测试通过的数量和总测试数量
    printf("%ld / %ld tests passed.\n", success_count, test_count);
    // 返回测试结果，如果全部通过则返回0，否则返回1
    return success_count == test_count ? 0 : 1;
# 闭合前面的函数定义
```