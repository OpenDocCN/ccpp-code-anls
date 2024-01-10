# `nmap\nbase\nbase.h`

```
/* $Id$ */

#ifndef NBASE_H
#define NBASE_H

/* NOTE -- libnbase offers the following features that you should probably
 * be aware of:
 *
 * * 'inline' is defined to what is necessary for the C compiler being
 *   used (which may be nothing)
 *
 * * snprintf, inet_pton, memcpy, and bzero are
 *   provided if you don't have them (prototypes for these are
 *   included either way).
 *
 * * WORDS_BIGENDIAN is defined if platform is big endian
 *
 * * Definitions included which give the operating system type.  They
 *   will generally be one of the following: LINUX, FREEBSD, NETBSD,
 *   OPENBSD, SOLARIS, SUNOS, BSDI, IRIX, NETBSD
 *
 * * Insures that getopt_* functions exist (such as getopt_long_only)
 *
 * * Various string functions such as Strncpy() and strcasestr() see protos
 *   for more info.
 *
 * * IPv6 structures like 'sockaddr_storage' are provided if they do
 *   not already exist.
 *
 * * Various Windows -> UNIX compatibility definitions are added (such as defining EMSGSIZE to WSAEMSGSIZE)
 */

#if HAVE_CONFIG_H
#include "nbase_config.h"
#else
#ifdef WIN32
#include "nbase_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifdef WIN32
#include "nbase_winunix.h"
#endif

#if HAVE_SYS_STAT_H
#include <sys/stat.h>
#endif

#if HAVE_UNISTD_H
#include <unistd.h>
#endif

#include <stdlib.h>
#include <ctype.h>
#include <time.h>

#if HAVE_SYS_SELECT_H
#include <sys/select.h>
#endif

#if HAVE_SYS_TYPES_H
#include <sys/types.h>
#endif

#if HAVE_SYS_PARAM_H
#include <sys/param.h>
#endif

#if HAVE_STRING_H
#include <string.h>
#endif

#if HAVE_NETDB_H
#include <netdb.h>
#endif

#if HAVE_INTTYPES_H
#include <inttypes.h>
#endif

#include <stdio.h>

#ifndef MAXHOSTNAMELEN
#define MAXHOSTNAMELEN 64
#endif

#ifndef MAXPATHLEN
#define MAXPATHLEN 2048
#endif

#ifndef HAVE___ATTRIBUTE__
#define __attribute__(args)
#endif

#include <stdarg.h>

/* Keep assert() defined for security reasons */
#undef NDEBUG

/* Integer types */
#include <stdint.h>
typedef uint8_t u8;
// 定义 8 位有符号整数类型
typedef int8_t s8;
// 定义 16 位无符号整数类型
typedef uint16_t u16;
// 定义 16 位有符号整数类型
typedef int16_t s16;
// 定义 32 位无符号整数类型
typedef uint32_t u32;
// 定义 32 位有符号整数类型
typedef int32_t s32;
// 定义 64 位无符号整数类型
typedef uint64_t u64;
// 定义 64 位有符号整数类型
typedef int64_t s64;

// 定义数学中的最大值宏
#ifndef MAX
#define MAX(x,y) (((x)>(y))?(x):(y))
#endif
// 定义数学中的最小值宏
#ifndef MIN
#define MIN(x,y) (((x)<(y))?(x):(y))
#endif
// 定义数学中的绝对值宏
#ifndef ABS
#define ABS(x) (((x) >= 0)?(x):-(x))
#endif

// 计算两个 timeval 结构的时间差，单位为微秒
#define TIMEVAL_SUBTRACT(a,b) (((a).tv_sec - (b).tv_sec) * 1000000 + (a).tv_usec - (b).tv_usec)
// 计算两个 timeval 结构的时间差，单位为毫秒
#define TIMEVAL_MSEC_SUBTRACT(a,b) ((((a).tv_sec - (b).tv_sec) * 1000) + ((a).tv_usec - (b).tv_usec) / 1000)
// 计算两个 timeval 结构的时间差，单位为秒，向零舍入
#define TIMEVAL_SEC_SUBTRACT(a,b) ((a).tv_sec - (b).tv_sec + (((a).tv_usec < (b).tv_usec) ? - 1 : 0))
// 计算两个 timeval 结构的时间差，单位为秒，转换为浮点数
#define TIMEVAL_FSEC_SUBTRACT(a,b) ((a).tv_sec - (b).tv_sec + (((a).tv_usec - (b).tv_usec)/1000000.0))

// 将一个 timeval 结构加上一定的毫秒数
#define TIMEVAL_MSEC_ADD(a, b, msecs) { (a).tv_sec = (b).tv_sec + ((msecs) / 1000); (a).tv_usec = (b).tv_usec + ((msecs) % 1000) * 1000; (a).tv_sec += (a).tv_usec / 1000000; (a).tv_usec %= 1000000; }
// 将一个 timeval 结构加上一定的微秒数
#define TIMEVAL_ADD(a, b, usecs) { (a).tv_sec = (b).tv_sec + ((usecs) / 1000000); (a).tv_usec = (b).tv_usec + ((usecs) % 1000000); (a).tv_sec += (a).tv_usec / 1000000; (a).tv_usec %= 1000000; }

// 判断一个 timeval 结构是否在另一个之前
#define TIMEVAL_BEFORE(a, b) (((a).tv_sec < (b).tv_sec) || ((a).tv_sec == (b).tv_sec && (a).tv_usec < (b).tv_usec))
// 判断一个 timeval 结构是否在另一个之后
#define TIMEVAL_AFTER(a, b) (((a).tv_sec > (b).tv_sec) || ((a).tv_sec == (b).tv_sec && (a).tv_usec > (b).tv_usec))

// 将一个 timeval 结构转换为浮点数秒
#define TIMEVAL_SECS(a) ((double) (a).tv_sec + (double) (a).tv_usec / 1000000)
/* sprintf family */
// 如果没有定义 HAVE_SNPRINTF 且定义了 __cplusplus，则声明 snprintf 函数
extern "C" int snprintf (char *str, size_t sz, const char *format, ...)
     __attribute__ ((format (printf, 3, 4)));

// 如果没有定义 HAVE_VSNPRINTF 且定义了 __cplusplus，则声明 vsnprintf 函数
extern "C" int vsnprintf (char *str, size_t sz, const char *format,
     va_list ap)
     __attribute__((format (printf, 3, 0)));

// 如果没有定义 HAVE_ASPRINTF 且定义了 __cplusplus，则声明 asprintf 函数
extern "C" int asprintf (char **ret, const char *format, ...)
     __attribute__ ((format (printf, 2, 3)));

// 如果没有定义 HAVE_VASPRINTF 且定义了 __cplusplus，则声明 vasprintf 函数
extern "C" int vasprintf (char **ret, const char *format, va_list ap)
     __attribute__((format (printf, 2, 0)));

// 如果没有定义 HAVE_ASNPRINTF 且定义了 __cplusplus，则声明 asnprintf 函数
extern "C" int asnprintf (char **ret, size_t max_sz, const char *format, ...)
     __attribute__ ((format (printf, 3, 4)));

// 如果没有定义 HAVE_VASNPRINTF 且定义了 __cplusplus，则声明 vasnprintf 函数
extern "C" int vasnprintf (char **ret, size_t max_sz, const char *format,
     va_list ap)
     __attribute__((format (printf, 3, 0)));

// 如果定义了 NEED_SNPRINTF_PROTO 且定义了 __cplusplus，则声明 snprintf 函数
extern "C" int snprintf (char *, size_t, const char *, ...);

// 如果定义了 NEED_VSNPRINTF_PROTO 且定义了 __cplusplus，则声明 vsnprintf 函数
extern "C" int vsnprintf (char *, size_t, const char *, va_list);

// 如果定义了 HAVE_GETOPT_H，则包含 <getopt.h> 头文件，否则包含 "getopt.h" 头文件
#ifdef HAVE_GETOPT_H
#include <getopt.h>
#else
#ifndef HAVE_GETOPT_LONG_ONLY
#include "getopt.h"
#endif
#endif /* HAVE_GETOPT_H */

/* More Windows-specific stuff */
#ifdef WIN32

#define WIN32_LEAN_AND_MEAN /* Whatever this means! From winclude.h*/

// 如果 Windows 没有定义 S_ISDIR，则定义 S_ISDIR
#ifndef S_ISDIR
#define S_ISDIR(m)      (((m) & _S_IFMT) == _S_IFDIR)
#endif

// 如果 Windows 没有定义 access() 的相关宏，则定义 F_OK、W_OK、R_OK
#ifndef F_OK
#define F_OK 00
#endif
#ifndef W_OK
#define W_OK 02
#endif
#ifndef R_OK
#define R_OK 04
#endif

// 重新定义 access、stat、execve 函数
#define access _access
#define stat _stat
#define execve _execve
#define getpid _getpid
#define dup _dup
#define dup2 _dup2
#define strdup _strdup
#define write _write
#define open _open
#define stricmp _stricmp
#define putenv _putenv
#define tzset _tzset

#if defined(_MSC_VER) && _MSC_VER < 1900
#define snprintf _snprintf
#endif

#define strcasecmp _stricmp
#define strncasecmp _strnicmp
#define execv _execv

#endif /* WIN32 */

/* Apparently Windows doesn't like /dev/null */
#ifdef WIN32
#define DEVNULL "NUL"  // 定义 Windows 下的空设备路径
#else
#define DEVNULL "/dev/null"  // 定义非 Windows 下的空设备路径
#endif

#if defined(_MSC_VER) && !defined(__cplusplus) && !defined(inline)
#define inline __inline
#endif

#if defined(__GNUC__)
#define NORETURN __attribute__((noreturn))  // 定义函数属性为不返回
#elif defined(_MSC_VER)
#define NORETURN __declspec(noreturn)  // 定义函数属性为不返回
#else
#define NORETURN  // 定义函数属性为空
#endif

#ifndef WIN32
# define CHECK_FD_OP(_Op, _Ctx) \  // 定义检查文件描述符操作的宏
  if (fd >= FD_SETSIZE) { \  // 如果文件描述符大于等于 FD_SETSIZE，则输出错误信息并终止程序
    fprintf(stderr, "Attempt to " #_Op " fd %d, which is not less than " \
                    "FD_SETSIZE (%d). Try using a lower parallelism.", \
                    fd, FD_SETSIZE); \
    abort(); \
  } \
  _Ctx _Op(fd, fds);  // 执行文件描述符操作
#else
# define CHECK_FD_OP(_Op, _Ctx) _Ctx _Op(fd, fds);  // 执行文件描述符操作
#endif

static inline int checked_fd_isset(int fd, fd_set *fds) {  // 定义检查文件描述符是否在集合中的函数
  CHECK_FD_OP(FD_ISSET, return);  // 调用文件描述符操作宏
}

static inline void checked_fd_clr(int fd, fd_set *fds) {  // 定义清除文件描述符在集合中的函数
  CHECK_FD_OP(FD_CLR, /**/);  // 调用文件描述符操作宏
}

static inline void checked_fd_set(int fd, fd_set *fds) {  // 定义设置文件描述符在集合中的函数
#ifdef WIN32
  if (fds->fd_count >= FD_SETSIZE) {  // 如果文件描述符计数大于等于 FD_SETSIZE，则输出错误信息并终止程序
    fprintf(stderr, "Attempt to call FD_SET, but fd_count is %d, which is not less than "
                    "FD_SETSIZE (%d). Try using a lower parallelism.",
                    fds->fd_count, FD_SETSIZE);
    abort();
  }
#endif
  CHECK_FD_OP(FD_SET, /**/);  // 调用文件描述符操作宏
}


#ifdef __cplusplus
extern "C" {
#endif

/* 返回类似 UNIX/Windows 的 errno。注意，Windows 调用是特定于套接字/网络的。此外，WINDOWS 倾向于重置错误，所以下次会返回成功。因此保存结果并重复使用，不要一直调用 socket_errno()。返回的 Windows 错误号类似于 WSAMSGSIZE，但 nbase.h 包括 #define 来将许多常见的 UNIX 错误与它们最接近的 Windows 等效项相关联。因此可以使用 EMSGSIZE 或 EINTR。 */
int socket_errno();

/* 我们不能只使用 strerror 来获取 Windows 上的套接字错误，因为它有自己的一套错误代码：例如 WSACONNRESET 而不是 ECONNRESET。这个函数将在 Windows 上执行正确的操作。调用方式如下：
   socket_strerror(socket_errno()) */
char *socket_strerror(int errnum);

/* usleep() 函数也很重要 */
#ifndef HAVE_USLEEP
#if defined( HAVE_NANOSLEEP) || defined(WIN32)
void usleep(unsigned long usec);
#endif
#endif

/* localtime 不是线程安全的。这将在支持的平台上使用线程安全的替代方法。 */
int n_localtime(const time_t *timer, struct tm *result);
int n_gmtime(const time_t *timer, struct tm *result);
int n_ctime(char *buffer, size_t bufsz, const time_t *timer);

/***************** 字符串函数 -- 请参阅 nbase_str.c ******************/
/* 我修改了这个条件，因为该函数存在但是 Redhat 不容易提供原型 */
#if !defined(HAVE_STRCASESTR) || (defined(LINUX) && !defined(__USE_GNU) && !defined(_GNU_SOURCE))
/* strcasestr 类似于 strstr()，但不区分大小写 */
char *strcasestr(const char *haystack, const char *pneedle);
#endif

#ifndef HAVE_STRCASECMP
int strcasecmp(const char *s1, const char *s2);
#endif

#ifndef HAVE_STRNCASECMP
int strncasecmp(const char *s1, const char *s2, size_t n);
#endif

#ifndef HAVE_GETTIMEOFDAY
int gettimeofday(struct timeval *tv, struct timeval *tz);
#endif

#ifndef HAVE_SLEEP
/* 以秒为单位暂停程序的执行 */
unsigned int sleep(unsigned int seconds);
#endif

/* Strncpy 类似于 strcpy()，但它总是在末尾添加零终止符，即使必须截断 */
int Strncpy(char *dest, const char *src, size_t n);

/* 格式化输出字符串到指定大小的缓冲区，使用可变参数列表 */
int Vsnprintf(char *, size_t, const char *, va_list)
     __attribute__ ((format (printf, 3, 0)));
int Snprintf(char *, size_t, const char *, ...)
     __attribute__ ((format (printf, 3, 4)));

/* 从给定的起始和结束位置创建字符串 */
char *mkstr(const char *start, const char *end);
/* 类似于 strchr，但不会超过结束位置，不会特殊处理空字符 */
const char *strchr_p(const char *str, const char *end, char c);

/* 使用可变参数列表分配并格式化字符串 */
int alloc_vsprintf(char **strp, const char *fmt, va_list va)
     __attribute__ ((format (printf, 2, 0)));

/* 转义 Windows 命令参数 */
char *escape_windows_command_arg(const char *arg);

/* 解析长整型数值，类似于 strtol 或 atoi，但只允许数字 */
long parse_long(const char *s, const char **tail);

/* 将字节计数转换为短 ASCII 表示，存储在提供的缓冲区中 */
char *format_bytecount(unsigned long long bytes, char *buf, size_t buflen);

/* 将字符串中的不可打印字符替换为指定字符 */
void replacenonprintable(char *str, int strlength, char replchar);

/* 检查文件路径是否存在、不是目录且可读取 */
int file_is_readable(const char *pathname);

/* 替代 dirname 和 basename 的可移植但不兼容的实现 */
char *path_get_dirname(const char *path);
char *path_get_basename(const char *path);

/* 一些常见内存分配例程的简单包装，如果分配失败则退出程序 */
void *safe_malloc(size_t size);
void *safe_realloc(void *ptr, size_t size);
/* 带零初始化的 safe_malloc 版本 */
// 分配指定大小的内存，并将其初始化为零
void *safe_zalloc(size_t size);

// 从系统获取随机字节序列
int get_random_bytes(void *buf, int numbytes);
// 获取一个随机整数
int get_random_int();
// 获取一个随机无符号短整型
unsigned short get_random_ushort();
// 获取一个随机无符号整型
unsigned int get_random_uint();
// 获取一个随机无符号64位整型
u64 get_random_u64();
// 获取一个随机无符号32位整型
u32 get_random_u32();
// 获取一个随机无符号16位整型
u16 get_random_u16();
// 获取一个随机无符号8位整型
u8 get_random_u8();
// 获取一个唯一的随机无符号32位整型
u32 get_random_unique_u32();

// 创建一个新的套接字，可以被子进程继承。在非Windows系统上，它只是一个普通的套接字
int inheritable_socket(int af, int style, int protocol);
// 在Windows系统上，dup函数只能用于文件描述符，而不是套接字句柄。这个函数为套接字实现了相同的功能
int dup_socket(int sd);
// 设置套接字为非阻塞模式
int unblock_socket(int sd);
// 设置套接字为阻塞模式
int block_socket(int sd);
// 将套接字绑定到指定的网络设备
int socket_bindtodevice(int sd, const char *device);

// 计算给定数据的CRC32循环冗余校验
unsigned long nbase_crc32(unsigned char *buf, int len);
// 计算给定数据的CRC32C循环冗余校验（Castagnoli算法）
unsigned long nbase_crc32c(unsigned char *buf, int len);
// 计算给定数据的Adler32校验和
unsigned long nbase_adler32(unsigned char *buf, int len);

// 将时间戳转换为秒数
double tval2secs(const char *tspec);
// 将时间戳转换为毫秒数
long tval2msecs(const char *tspec);
// 获取时间戳的单位
const char *tval_unit(const char *tspec);

// 在给定的时间内等待套接字的I/O事件
int fselect(int s, fd_set *rmaster, fd_set *wmaster, fd_set *emaster, struct timeval *tv);

// 将给定数据以十六进制格式进行转储
char *hexdump(const u8 *cp, u32 length);

// 获取可执行文件的路径
char *executable_path(const char *argv0);

// 设置日志输出函数
void nbase_set_log(void (*log_user_func)(const char *, ...),void (*log_debug_func)(const char *, ...));
// 创建一个地址集合
struct addrset *addrset_new();
// 释放地址集合的内存
extern void addrset_free(struct addrset *set);
// 打印地址集合的内容
extern void addrset_print(FILE *fp, const struct addrset *set);
// 向地址集合中添加指定的地址规范
extern int addrset_add_spec(struct addrset *set, const char *spec, int af, int dns);
// 从文件中读取地址，并添加到地址集合中
extern int addrset_add_file(struct addrset *set, FILE *fd, int af, int dns);
# 检查地址集中是否包含指定的地址
extern int addrset_contains(const struct addrset *set, const struct sockaddr *sa);

# 如果未定义 STDIN_FILENO，则定义为 0
#ifndef STDIN_FILENO
#define STDIN_FILENO 0
#endif

# 如果未定义 STDOUT_FILENO，则定义为 1
#ifndef STDOUT_FILENO
#define STDOUT_FILENO 1
#endif

# 如果未定义 STDERR_FILENO，则定义为 2
#ifndef STDERR_FILENO
#define STDERR_FILENO 2
#endif

# 包含 IPv6 相关的头文件
#include "nbase_ipv6.h"

# 如果是 C++ 代码，则结束 extern "C" 块
#ifdef __cplusplus
}
#endif

# 结束 nbase.h 文件的条件编译
#endif /* NBASE_H */
```