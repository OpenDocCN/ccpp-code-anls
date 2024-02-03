# `nmap\ncat\sys_wrap.c`

```cpp
/* $Id$ */

#include <limits.h>

#include "sys_wrap.h"
#include "util.h"

// 分配内存并清零，检查是否溢出
void *Calloc(size_t nmemb, size_t size)
{
    void *ret;

    /* older libcs don't check for int overflow */
    smul(nmemb, size);

    // 调用系统函数 calloc 分配内存
    ret = calloc(nmemb, size);
    // 检查分配是否成功
    if (ret == NULL)
        die("calloc");

    return ret;
}

// 关闭文件描述符
int Close(int fd)
{
    // 调用系统函数 close 关闭文件描述符
    if (close(fd) < 0)
        die("close");

    return 0;
}

// 连接到服务器
int Connect(int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen)
{
    // 调用系统函数 connect 连接到服务器
    if (connect(sockfd, serv_addr, addrlen) < 0)
        die("connect");

    return 0;
}

// 复制文件描述符
int Dup2(int oldfd, int newfd)
{
    int ret;

    // 调用系统函数 dup2 复制文件描述符
    ret = dup2(oldfd, newfd);
    // 检查复制是否成功
    if (ret < 0)
        die("dup2");

    return ret;
}

// 监听连接
int Listen(int s, int backlog)
{
    // 调用系统函数 listen 监听连接
    if (listen(s, backlog) < 0)
        die("listen");

    return 0;
}

// 打开文件
int Open(const char *pathname, int flags, mode_t mode)
{
    int ret;

    // 调用系统函数 open 打开文件
    ret = open(pathname, flags, mode);
    // 检查打开是否成功
    if (ret < 0)
        die("open");

    return ret;
}

// 读取数据
ssize_t Read(int fd, void *buf, size_t count)
{
    ssize_t ret;

    // 调用系统函数 read 读取数据
    ret = read(fd, buf, count);
    // 检查读取是否成功
    if (ret < 0)
        die("read");

    return ret;
}

// 设置套接字选项
int Setsockopt(int s, int level, int optname, const void *optval,
                    socklen_t optlen)
{
    int ret;

    // 调用系统函数 setsockopt 设置套接字选项
    ret = setsockopt(s, level, optname, (const char *) optval, optlen);
    // 检查设置是否成功
    if (ret < 0)
        die("setsockopt");

    return ret;
}

// 设置信号处理函数
sighandler_t Signal(int signum, sighandler_t handler)
{
    sighandler_t ret;

    // 调用系统函数 signal 设置信号处理函数
    ret = signal(signum, handler);
    // 检查设置是否成功
    if (ret == SIG_ERR)
        die("signal");

    return ret;
}

// 创建套接字
int Socket(int domain, int type, int protocol)
{
    int ret;

    // 调用系统函数 socket 创建套接字
    ret = socket(domain, type, protocol);
    // 检查创建是否成功
    if (ret < 0)
        die("socket");

    return ret;
}

// 复制字符串
char *Strdup(const char *s)
{
    char *ret;

    // 调用系统函数 strdup 复制字符串
    ret = strdup(s);
    // 检查复制是否成功
    if (ret == NULL)
        die("strdup");

    return ret;
}

// 写入数据
ssize_t Write(int fd, const void *buf, size_t count)
{
    // 调用系统函数 write 写入数据
    ssize_t ret = write(fd, buf, count);
    # 如果写入的字节数小于 0，则不会中止程序
    if (ret < 0)         
        # 输出错误信息 "write"
        die("write");
    # 返回写入的字节数
    return ret;
# 闭合前面的函数定义
```