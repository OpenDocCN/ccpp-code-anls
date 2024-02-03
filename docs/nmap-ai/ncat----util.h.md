# `nmap\ncat\util.h`

```cpp
/* $Id$ */

#ifndef UTIL_H_
#define UTIL_H_

#include "ncat_config.h"

#include "nbase.h"
#ifndef WIN32
#include <sys/types.h>
#include <netinet/in.h>
#endif

#include "sockaddr_u.h"

#if HAVE_SYS_UN_H
#include <sys/un.h>
#include <string.h>

#define NCAT_INIT_SUN(_Sock, _Source) do { \
  // 将 _Sock 内存清零
  memset(_Sock, 0, sizeof(union sockaddr_u)); \
  // 设置 _Sock 的地址族为 AF_UNIX
  (_Sock)->un.sun_family = AF_UNIX; \
  // 如果 _Source 的长度超过了地址路径的最大长度减1，则退出程序并打印错误信息
  if (strlen(_Source) > sizeof((_Sock)->un.sun_path) - 1) \
    bye("Socket path length is too long. Max: %lu", sizeof((_Sock)->un.sun_path) - 1); \
  // 将 _Source 复制到 _Sock 的地址路径中
  strncpy((_Sock)->un.sun_path, _Source, sizeof((_Sock)->un.sun_path) - 1); \
} while (0);

#endif

#ifdef HAVE_OPENSSL
#include <openssl/ssl.h>
#endif

/* add/multiply unsigned values safely */
// 安全地对无符号值进行加法操作
size_t sadd(size_t, size_t);
// 安全地对无符号值进行乘法操作
size_t smul(size_t, size_t);

#ifdef WIN32
// Windows 环境初始化
void windows_init();
#endif

// 记录用户日志
void loguser(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));
// 记录用户日志（无前缀）
void loguser_noprefix(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));
// 记录调试日志
void logdebug(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));
// 记录测试日志
void logtest(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));

/* handle errors */

// 断言宏，如果表达式不成立，则退出程序并打印错误信息
#define ncat_assert(expr) \
do { \
        if (!(expr)) \
                bye("assertion failed: %s", #expr); \
} while (0)

// 退出程序并打印错误信息
void die(char *);

// 退出程序并打印错误信息，不返回
NORETURN void bye(const char *, ...)
     __attribute__ ((format (printf, 1, 2)));

/* zero out some memory, bzero() is deprecated */
// 将一些内存清零，bzero() 已经被弃用
void zmem(void *, size_t);

// 向缓冲区追加字符串
int strbuf_append(char **buf, size_t *size, size_t *offset, const char *s, size_t n);

// 向缓冲区追加字符串
int strbuf_append_str(char **buf, size_t *size, size_t *offset, const char *s);

// 向缓冲区按格式追加字符串
int strbuf_sprintf(char **buf, size_t *size, size_t *offset, const char *fmt, ...)
     __attribute__ ((format (printf, 4, 5)));

// 检查地址是否为本地地址
int addr_is_local(const union sockaddr_u *su);

// 获取地址字符串表示形式
const char *socktop(const union sockaddr_u *su, socklen_t ss_len);
// 获取 Internet 地址字符串表示形式
const char *inet_socktop(const union sockaddr_u *su);
// 声明一个函数，接受一个 sockaddr_u 类型的参数，并返回一个无符号短整型
unsigned short inet_port(const union sockaddr_u *su);

// 声明一个函数，接受两个整型参数和一个 sockaddr_u 类型的参数，并返回一个整型
int do_listen(int, int, const union sockaddr_u *su);

// 声明一个函数，接受一个整型参数，并返回一个整型
int do_connect(int);

// 声明一个函数，接受一个结构体 in_addr 类型的参数，一个结构体 in_addr 数组类型的参数，一个整型参数，一个指针参数，一个 size_t 类型的指针参数，并返回一个无符号字符型指针
unsigned char *buildsrcrte(struct in_addr dstaddr, struct in_addr routes[],
                  int numroutes, int ptr, size_t *len);

// 声明一个函数，接受一个 sockaddr_u 类型的参数，并返回一个整型
int allow_access(const union sockaddr_u *su);

// 声明一个函数，接受一个结构体 timeval 类型的指针参数和一个长整型参数，无返回值
void ms_to_timeval(struct timeval *tv, long ms)
    __attribute__ ((nonnull));

// 声明一个结构体 fdinfo，包含整型 fd、lasterr，sockaddr_u 类型的 remoteaddr，socklen_t 类型的 ss_len，以及可能的 SSL 指针
struct fdinfo {
    int fd;
    int lasterr;
    union sockaddr_u remoteaddr;
    socklen_t ss_len;
#ifdef HAVE_OPENSSL
    SSL *ssl;
#endif
};

// 声明一个结构体 fd_list，包含 fdinfo 指针、整型 nfds、maxfds、fdmax，以及一个整型 state
typedef struct fd_list {
    struct fdinfo *fds;
    int nfds, maxfds, fdmax;
    int state; /* incremented each time the list is modified */
} fd_list_t;

// 声明一个函数，接受一个 fd_list_t 指针和一个 fdinfo 指针参数，并返回一个整型
int add_fdinfo(fd_list_t *, struct fdinfo *);

// 声明一个函数，接受一个 fd_list_t 指针和一个整型参数，并返回一个整型
int add_fd(fd_list_t *fdl, int fd);

// 声明一个函数，接受一个 fd_list_t 指针和一个整型参数，并返回一个整型
int rm_fd(fd_list_t *, int);

// 声明一个函数，接受一个 fd_list_t 指针参数，无返回值
void free_fdlist(fd_list_t *);

// 声明一个函数，接受一个 fd_list_t 指针和一个整型参数，无返回值
void init_fdlist(fd_list_t *, int);

// 声明一个函数，接受一个 fd_list_t 指针参数，并返回一个整型
int get_maxfd(fd_list_t *);

// 声明一个函数，接受一个 fd_list_t 指针和一个整型参数，返回一个 fdinfo 指针
struct fdinfo *get_fdinfo(const fd_list_t *, int);

// 声明一个函数，接受一个字符型指针、一个整型指针、一个字符型指针指针、一个整型指针，无返回值
int fix_line_endings(char *src, int *len, char **dst, int *state);

// 声明一个函数，接受一个 size_t 类型的指针参数和一个字符型指针参数，返回一个无符号字符型指针
unsigned char *next_protos_parse(size_t *outlen, const char *in);

// 结束头文件的声明
#endif
```