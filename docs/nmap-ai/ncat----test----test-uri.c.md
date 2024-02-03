# `nmap\ncat\test\test-uri.c`

```cpp
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "ncat_core.h"
#include "http.h"

static long test_count = 0;  // 初始化测试计数
static long success_count = 0;  // 初始化成功测试计数

/* 检查字符串或空指针是否相等 */
int nullstreq(const char *s, const char *t)
{
    if (s == NULL) {
        if (t == NULL)
            return 1;
        else
            return 0;
    } else {
        if (t == NULL)
            return 0;
        else
            return strcmp(s, t) == 0;
    }
}

int test_uri(const char *uri_s, const char *scheme, const char *host, int port, const char *path)
{
    struct uri uri;
    int scheme_match, host_match, port_match, path_match;

    test_count++;  // 增加测试计数

    if (uri_parse(&uri, uri_s) == NULL) {  // 解析 URI，如果失败则打印错误信息并返回
        printf("FAIL %s: couldn't parse.\n", uri_s);
        return 0;
    }

    scheme_match = nullstreq(uri.scheme, scheme);  // 检查 scheme 是否匹配
    host_match = nullstreq(uri.host, host);  // 检查 host 是否匹配
    port_match = uri.port == port;  // 检查 port 是否匹配
    path_match = nullstreq(uri.path, path);  // 检查 path 是否匹配

    if (scheme_match && host_match && port_match && path_match) {  // 如果所有条件都匹配，则打印 PASS 并增加成功测试计数
        printf("PASS %s\n", uri_s);
        uri_free(&uri);
        success_count++;
        return 1;
    } else {  // 如果有任何条件不匹配，则打印 FAIL 并列出不匹配的条件
        printf("FAIL %s:", uri_s);
        if (!scheme_match)
            printf(" \"%s\" != \"%s\".", uri.scheme, scheme);
        if (!host_match)
            printf(" \"%s\" != \"%s\".", uri.host, host);
        if (!port_match)
            printf(" %d != %d.", uri.port, port);
        if (!path_match)
            printf(" \"%s\" != \"%s\".", uri.path, path);
        printf("\n");
        uri_free(&uri);
        return 0;
    }
}

int test_fail(const char *uri_s)
{
    struct uri uri;

    test_count++;  // 增加测试计数

    if (uri_parse(&uri, uri_s) != NULL) {  // 如果解析 URI 成功，则打印错误信息并返回
        uri_free(&uri);
        printf("FAIL %s: not expected to parse.\n", uri_s);
        return 0;
    } else {  // 如果解析 URI 失败，则打印 PASS 并增加成功测试计数
        printf("PASS %s\n", uri_s);
        success_count++;
        return 0;
    }
}

int main(int argc, char *argv[])
{
    # 测试给定 URI 是否符合预期，依次为 URI、协议、主机、端口、路径
    test_uri("http://www.example.com", "http", "www.example.com", 80, "");

    test_uri("HTTP://www.example.com", "http", "www.example.com", 80, "");
    test_uri("http://WWW.EXAMPLE.COM", "http", "WWW.EXAMPLE.COM", 80, "");

    test_uri("http://www.example.com:100", "http", "www.example.com", 100, "");
    test_uri("http://www.example.com:1", "http", "www.example.com", 1, "");
    test_uri("http://www.example.com:65535", "http", "www.example.com", 65535, "");
    test_uri("http://www.example.com:", "http", "www.example.com", 80, "");
    test_uri("http://www.example.com:/", "http", "www.example.com", 80, "/");

    test_uri("http://www.example.com/", "http", "www.example.com", 80, "/");
    test_uri("http://www.example.com:100/", "http", "www.example.com", 100, "/");

    test_uri("http://1.2.3.4", "http", "1.2.3.4", 80, "");
    test_uri("http://1.2.3.4:100", "http", "1.2.3.4", 100, "");
    test_uri("http://[::ffff]", "http", "::ffff", 80, "");
    test_uri("http://[::ffff]:100", "http", "::ffff", 100, "");

    test_uri("http://www.example.com/path?query#frag", "http", "www.example.com", 80, "/path?query#frag");

    test_uri("http://www.exampl%65.com", "http", "www.example.com", 80, "");
    test_uri("http://www.exampl%6a.com", "http", "www.examplj.com", 80, "");
    test_uri("http://www.exampl%6A.com", "http", "www.examplj.com", 80, "");
    test_uri("http://www.exampl%2523.com", "http", "www.exampl%23.com", 80, "");
    test_fail("http://www.example.com:%380");
    test_uri("http://www.example.com/a%23b", "http", "www.example.com", 80, "/a%23b");

    test_uri("unknown://www.example.com", "unknown", "www.example.com", -1, "");

    test_uri("unknown:", "unknown", NULL, -1, "");

    test_fail("");
    test_fail("/dir/file");
    test_fail("http://www.example.com:-1");
    test_fail("http://www.example.com:0");
    test_fail("http://www.example.com:65536");

    # 我们明确不支持在主机名中包含用户信息。
    # 测试失败的情况：包含用户名但不包含密码的 URL
    test_fail("http://user@www.example.com");
    # 测试失败的情况：包含用户名和密码的 URL
    test_fail("http://user:pass@www.example.com");

    # 输出测试通过的数量和总测试数量
    printf("%ld / %ld tests passed.\n", success_count, test_count);

    # 返回测试是否全部通过，是则返回 0，否则返回 1
    return success_count == test_count ? 0 : 1;
# 闭合前面的函数定义
```