# `nmap\ncat\sys_wrap.h`

```cpp
/* $Id$ */

#ifndef SYS_WRAP_H
#define SYS_WRAP_H

#include "nbase.h"

#include "util.h"

#ifndef WIN32
#include <unistd.h>  // 包含 POSIX 系统调用的头文件
#include <sys/wait.h>  // 包含等待进程状态改变的头文件
#include <sys/types.h>  // 包含基本系统数据类型的头文件
#include <netinet/in.h>  // 包含互联网地址族的头文件
#include <sys/socket.h>  // 包含套接字接口的头文件
#include <sys/select.h>  // 包含多路复用 I/O 的头文件
#include <netdb.h>  // 包含网络数据库操作的头文件
#include <arpa/inet.h>  // 包含互联网操作的头文件
#else
#define pid_t int  // 定义进程 ID 类型
#define mode_t int  // 定义文件权限类型
#define uid_t int  // 定义用户 ID 类型
#define socklen_t int  // 定义套接字长度类型
#define uint16_t int  // 定义 16 位无符号整数类型
#define ssize_t int  // 定义有符号大小类型
#include <WinDef.h>  // 包含 Windows 数据类型定义的头文件
#endif

#include <sys/stat.h>  // 包含文件状态的头文件
#include <stdarg.h>  // 包含可变参数列表的头文件
#include <fcntl.h>  // 包含文件控制的头文件
#include <stdio.h>  // 包含标准输入输出的头文件
#include <signal.h>  // 包含信号处理的头文件
#include <string.h>  // 包含字符串操作的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <errno.h>  // 包含错误码的头文件

/* need an autoconf to check for this */
typedef void (*sighandler_t)(int);  // 定义信号处理函数类型

void *Calloc(size_t nmemb, size_t size);  // 分配内存并初始化为 0
int Close(int fd);  // 关闭文件描述符
int Connect(int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen);  // 连接到指定地址的套接字
int Dup2(int oldfd, int newfd);  // 复制文件描述符
int Listen(int s, int backlog);  // 监听套接字
int Open(const char *pathname, int flags, mode_t mode);  // 打开文件
ssize_t Read(int fd, void *buf, size_t count);  // 读取文件内容
int Setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen);  // 设置套接字选项
sighandler_t Signal(int signum, sighandler_t handler);  // 设置信号处理函数
int Socket(int domain, int type, int protocol);  // 创建套接字
char *Strdup(const char *s);  // 复制字符串
ssize_t Write(int fd, const void *buf, size_t count);  // 写入文件内容

#endif
```