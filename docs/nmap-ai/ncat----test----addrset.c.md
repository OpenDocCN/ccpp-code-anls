# `nmap\ncat\test\addrset.c`

```
/*
    Usage: ./addrset [<specification> ...]

    This program tests the addrset functions in nbase/nbase_addrset.c,
    the ones that maintain the lists of addresses for --allow and
    --deny. It takes as arguments specifications that are added to an
    addrset. It then reads whitespace-separated host names or IP
    addresses from standard input and echoes only those that are in the
    addrset.

    David Fifield

    Example:
    $ echo "1.2.3.4 1.0.0.5 1.2.3.8" | ./addrset "1.2.3.10/24"
    1.2.3.4
    1.2.3.8
*/

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "ncat_core.h"

#ifdef WIN32
#include "../nsock/src/error.h"
#endif


#ifdef WIN32
static void win_init(void)
{
  WSADATA data;
  int rc;

  rc = WSAStartup(MAKEWORD(2,2), &data);
  if (rc)
    fatal("failed to start winsock: %s\n", socket_strerror(rc));
}
#endif

static int resolve_name(const char *name, struct addrinfo **result)
{
    // 初始化地址信息结构体
    struct addrinfo hints = { 0 };

    // 设置协议为TCP
    hints.ai_protocol = IPPROTO_TCP;
    // 初始化结果为NULL
    *result = NULL;

    // 解析主机名或IP地址
    return getaddrinfo(name, NULL, &hints, result);
}

int main(int argc, char *argv[])
{
    // 创建地址集合
    struct addrset *set;
    // 读取输入的行
    char line[1024];
    int i;

#ifdef WIN32
    // 在Windows下初始化Winsock
    win_init();
#endif

    // 创建地址集合
    set = addrset_new();

    // 初始化选项
    options_init();

    // 遍历命令行参数
    for (i = 1; i < argc; i++) {
        // 将规范添加到地址集合中
        if (!addrset_add_spec(set, argv[i], o.af, !o.nodns)) {
            // 如果添加失败，输出错误信息并退出
            fprintf(stderr, "Error adding spec \"%s\".\n", argv[i]);
            exit(1);
        }
    }
    # 从标准输入中逐行读取数据，直到读取到 NULL 为止
    while (fgets(line, sizeof(line), stdin) != NULL) {
        char *s, *hostname;
        struct addrinfo *addrs;

        # 将 line 赋值给 s
        s = line;
        # 使用空格、制表符和换行符作为分隔符，将 s 分割成 token
        while ((hostname = strtok(s, " \t\n")) != NULL) {
            int rc;

            # 将 s 置为 NULL，表示下一次调用 strtok 会继续从 line 开始分割
            s = NULL;

            # 解析主机名，获取地址信息
            rc = resolve_name(hostname, &addrs);
            # 如果解析出错，输出错误信息并继续下一次循环
            if (rc != 0) {
                fprintf(stderr, "Error resolving \"%s\": %s.\n", hostname, gai_strerror(rc));
                continue;
            }
            # 如果没有找到地址信息，输出提示信息并继续下一次循环
            if (addrs == NULL) {
                fprintf(stderr, "No addresses found for \"%s\".\n", hostname);
                continue;
            }

            # 检查返回的地址信息是否在集合中，如果在则输出主机名
            if (addrset_contains(set, addrs->ai_addr))
                    printf("%s\n", hostname);

            # 释放地址信息
            freeaddrinfo(addrs);
        }
    }

    # 释放地址集合
    addrset_free(set);

    # 返回 0 表示程序正常结束
    return 0;
# 闭合前面的函数定义
```