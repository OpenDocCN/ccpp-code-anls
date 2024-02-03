# `nmap\nbase\nbase_misc.c`

```cpp
/* $Id$ */

#include "nbase.h"

#ifndef WIN32
#include <errno.h>
#ifndef errno
extern int errno;
#endif
#else
#include <winsock2.h>
#endif

#include <limits.h>
#include <stdio.h>
#include "nbase_ipv6.h"
#include "nbase_crc32ct.h"

#include <assert.h>
#include <fcntl.h>

#ifdef WIN32
#include <conio.h>
#endif

#ifndef INET6_ADDRSTRLEN
#define INET6_ADDRSTRLEN 46
#endif

/* 返回类似于 UNIX/Windows 的 errno 等效值。注意，Windows 调用是特定于套接字/网络的。
   返回的 Windows 错误号类似于 WSAMSGSIZE，但 nbase.h 包含了许多常见 UNIX 错误与它们最接近的 Windows 等效值的 #define。
   因此，您可以使用 EMSGSIZE 或 EINTR。 */
int socket_errno() {
#ifdef WIN32
    return WSAGetLastError();
#else
    return errno;
#endif
}

/* 我们不能只使用 strerror 来获取 Windows 上的套接字错误，因为它有自己的一套错误代码：例如 WSACONNRESET 而不是 ECONNRESET。
   这个函数将在 Windows 上执行正确的操作。调用方式如下
     socket_strerror(socket_errno()) */
char *socket_strerror(int errnum) {
#ifdef WIN32
    static char buffer[256];

    if (!FormatMessageA(FORMAT_MESSAGE_FROM_SYSTEM |
        FORMAT_MESSAGE_IGNORE_INSERTS |
        FORMAT_MESSAGE_MAX_WIDTH_MASK,
        0, errnum, 0, buffer, sizeof(buffer), NULL))
    {
        Snprintf(buffer, 255, "socket error %d; FormatMessage error: %08x", errnum, GetLastError());
    };

    return buffer;
#else
    return strerror(errnum);
#endif
}

/* 比较两个 sockaddr_storage 结构，返回值类似于 strcmp。
   首先比较地址族，然后比较地址（如果地址族相等）。结构必须是真正的完整长度的 sockaddr_storage 结构，而不是像 sockaddr_in 这样较短的结构。 */
int sockaddr_storage_cmp(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b) {
  if (a->ss_family < b->ss_family)
    return -1;
  else if (a->ss_family > b->ss_family)
    # 返回整数1
    return 1;
  # 如果地址族是IPv4
  if (a->ss_family == AF_INET) {
    # 将a和b强制转换为IPv4地址结构体指针
    struct sockaddr_in *sin_a = (struct sockaddr_in *) a;
    struct sockaddr_in *sin_b = (struct sockaddr_in *) b;
    # 比较IPv4地址的大小，返回-1、1或0
    if (sin_a->sin_addr.s_addr < sin_b->sin_addr.s_addr)
      return -1;
    else if (sin_a->sin_addr.s_addr > sin_b->sin_addr.s_addr)
      return 1;
    else
      return 0;
  # 如果地址族是IPv6
  } else if (a->ss_family == AF_INET6) {
    # 将a和b强制转换为IPv6地址结构体指针
    struct sockaddr_in6 *sin6_a = (struct sockaddr_in6 *) a;
    struct sockaddr_in6 *sin6_b = (struct sockaddr_in6 *) b;
    # 比较IPv6地址的内容，返回比较结果
    return memcmp(sin6_a->sin6_addr.s6_addr, sin6_b->sin6_addr.s6_addr,
                  sizeof(sin6_a->sin6_addr.s6_addr));
  # 如果地址族既不是IPv4也不是IPv6
  } else {
    # 断言，如果程序执行到这里，表示出现了意外情况
    assert(0);
  }
  # 返回0，表示未到达此处
  return 0; /* Not reached */
}

// 比较两个 sockaddr_storage 结构体是否相等
int sockaddr_storage_equal(const struct sockaddr_storage *a,
  const struct sockaddr_storage *b) {
  return sockaddr_storage_cmp(a, b) == 0;
}

/* 这个函数是 inet_ntop 的简化版本，因为你不需要传递目标缓冲区。
   相反，它返回一个静态缓冲区，你可以在函数再次被调用之前（由同一线程或进程中的另一个线程）一直使用。
   如果出现奇怪的错误（比如 sslen 太短），则返回 NULL。 */
const char *inet_ntop_ez(const struct sockaddr_storage *ss, size_t sslen) {

  const struct sockaddr_in *sin = (struct sockaddr_in *) ss;
  static char str[INET6_ADDRSTRLEN];
#if HAVE_IPV6
  const struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *) ss;
#endif

  str[0] = '\0';

  if (sin->sin_family == AF_INET) {
    if (sslen < sizeof(struct sockaddr_in))
      return NULL;
    return inet_ntop(AF_INET, &sin->sin_addr, str, sizeof(str));
  }
#if HAVE_IPV6
  else if(sin->sin_family == AF_INET6) {
    if (sslen < sizeof(struct sockaddr_in6))
      return NULL;
    return inet_ntop(AF_INET6, &sin6->sin6_addr, str, sizeof(str));
  }
#endif
  // 有些笔记本电脑将禁用的 wifi 卡的 IP 和地址族报告为 null
  // 所以是的，有时我们会遇到这种情况。
  return NULL;
}

/* 创建一个新的套接字，可以被子进程继承。在非 Windows 系统上，它只是一个普通的套接字。 */
int inheritable_socket(int af, int style, int protocol) {
#ifdef WIN32
  /* WSASocket 与 socket 类似，不同之处在于它创建的套接字可以被子进程继承（例如由 CreateProcess 创建的进程），
     而 socket 创建的套接字则不行。 */
  return WSASocket(af, style, protocol, NULL, 0, WSA_FLAG_OVERLAPPED);
#else
  return socket(af, style, protocol);
#endif
}

/* 在 Windows 上，dup 函数只能用于文件描述符，而不是套接字句柄。这个函数为套接字实现了相同的功能。 */
int dup_socket(int sd) {
#ifdef WIN32
  // 如果是在 Windows 平台下
  HANDLE copy;
  // 复制当前进程的句柄，用于创建新的句柄
  if (DuplicateHandle(GetCurrentProcess(), (HANDLE) sd,
                      GetCurrentProcess(), &copy,
                      0, FALSE, DUPLICATE_SAME_ACCESS) == 0) {
    // 如果复制失败，返回-1
    return -1;
  }
  // 返回新的句柄
  return (int) copy;
#else
  // 如果不是在 Windows 平台下，直接调用 dup 函数返回新的文件描述符
  return dup(sd);
#endif
}

// 将套接字设置为非阻塞模式
int unblock_socket(int sd) {
#ifdef WIN32
  // 如果是在 Windows 平台下
  unsigned long one = 1;
  // 设置套接字为非阻塞模式
  ioctlsocket(sd, FIONBIO, &one);
  // 返回 0 表示成功
  return 0;
#else
  // 如果不是在 Windows 平台下
  int options;
  // 获取当前套接字的状态
  options = fcntl(sd, F_GETFL);
  if (options == -1)
    return -1;
  // 设置套接字为非阻塞模式
  return fcntl(sd, F_SETFL, O_NONBLOCK | options);
#endif /* WIN32 */
}

// 将套接字设置为阻塞模式
int block_socket(int sd) {
#ifdef WIN32
  // 如果是在 Windows 平台下
  unsigned long options = 0;
  // 设置套接字为阻塞模式
  ioctlsocket(sd, FIONBIO, &options);
  // 返回 0 表示成功
  return 0;
#else
  // 如果不是在 Windows 平台下
  int options;
  // 获取当前套接字的状态
  options = fcntl(sd, F_GETFL);
  if (options == -1)
    return -1;
  // 设置套接字为阻塞模式
  return fcntl(sd, F_SETFL, (~O_NONBLOCK) & options);
#endif
}

// 使用 SO_BINDTODEVICE sockopt 来绑定到特定接口（仅限 Linux）
int socket_bindtodevice(int sd, const char *device) {
  char padded[sizeof(int)];
  size_t len;
  // 计算设备名称的长度
  len = strlen(device) + 1;
  // 如果长度小于 int 的大小，进行填充
  if (len < sizeof(padded)) {
    // 使用空字符填充字符串
    strncpy(padded, device, sizeof(padded));
    device = padded;
    len = sizeof(padded);
  }

#ifdef SO_BINDTODEVICE
  // 如果支持 SO_BINDTODEVICE
  // 设置套接字选项，绑定到指定的接口
  if (setsockopt(sd, SOL_SOCKET, SO_BINDTODEVICE, device, len) < 0)
    # 返回整数值0
        return 0;
#endif

  return 1;
}

/* 将时间规范转换为秒数。时间规范是一个非负实数，可能后面跟着一个单位后缀。后缀有"ms"表示毫秒，"s"表示秒，"m"表示分钟，或"h"表示小时。如果无法解析字符串，则返回-1。 */
double tval2secs(const char *tspec) {
  double d;
  char *tail;

  errno = 0;
  // 将字符串转换为双精度浮点数
  d = strtod(tspec, &tail);
  // 如果字符串为空或者转换出错，则返回-1
  if (*tspec == '\0' || errno != 0)
    return -1;
  // 如果单位是毫秒，则返回毫秒数
  if (strcasecmp(tail, "ms") == 0)
    return d / 1000.0;
  // 如果单位为空或者是秒，则返回秒数
  else if (*tail == '\0' || strcasecmp(tail, "s") == 0)
    return d;
  // 如果单位是分钟，则返回分钟数
  else if (strcasecmp(tail, "m") == 0)
    return d * 60.0;
  // 如果单位是小时，则返回小时数
  else if (strcasecmp(tail, "h") == 0)
    return d * 60.0 * 60.0;
  // 其他情况返回-1
  else
    return -1;
}

// 将时间规范转换为毫秒数
long tval2msecs(const char *tspec) {
  double s, ms;

  // 调用tval2secs函数将时间规范转换为秒数
  s = tval2secs(tspec);
  // 如果转换出错，则返回-1
  if (s == -1)
    return -1;
  // 将秒数转换为毫秒数
  ms = s * 1000.0;
  // 如果毫秒数超出long类型的范围，则返回-1
  if (ms > LONG_MAX || ms < LONG_MIN)
    return -1;

  return (long) ms;
}

/* 返回时间规范的单位部分（例如"ms"、"s"、"m"或"h"）。如果解析出错或没有单位，则返回NULL。 */
const char *tval_unit(const char *tspec) {
  double d;
  char *tail;

  errno = 0;
  // 将字符串转换为双精度浮点数
  d = strtod(tspec, &tail);
  // 如果字符串为空或者转换出错，或者单位为空，则返回NULL
  if (*tspec == '\0' || errno != 0 || *tail == '\0')
    return NULL;

  return tail;
}

/* 用于在Windows上替换select函数，允许在stdin（文件描述符0）上进行选择，并在零个文件描述符上进行选择（仅用于超时）。普通的Windows select函数无法在非套接字上工作，比如stdin，并且如果没有给出文件描述符，因为它们为NULL或为空，它会返回错误。这只适用于套接字和stdin；如果在集合中有指向普通打开文件的描述符，Windows将返回WSAENOTSOCK。 */
int fselect(int s, fd_set *rmaster, fd_set *wmaster, fd_set *emaster, struct timeval *tv)
{
#ifdef WIN32
    // 标记 stdin 线程是否已经启动
    static int stdin_thread_started = 0;
    // 准备好的文件描述符数量
    int fds_ready = 0;
    // 迭代器
    int iter = -1;
    // 是否执行 select
    int do_select = 0;
    // 临时存储时间值
    struct timeval stv;
    // 读取文件描述符集合
    fd_set rset, wset, eset;
    // 标记 stdin 是否可读
    int r_stdin = 0;
    // 标记 stdin 是否有异常
    int e_stdin = 0;
    // 标记 stdin 是否准备好
    int stdin_ready = 0;

    /* Figure out whether there are any FDs in the sets, as @$@!$# Windows
       returns WSAINVAL (10022) if you call a select() with no FDs, even though
       the Linux man page says that doing so is a good, reasonably portable way
       to sleep with subsecond precision.  Sigh. */
    // 检查读取文件描述符集合是否为空
    if (rmaster != NULL) {
      /* If stdin is requested, clear it and remember it. */
      // 如果需要读取 stdin，则清除它并记住它
      if (checked_fd_isset(STDIN_FILENO, rmaster)) {
        r_stdin = 1;
        checked_fd_clr(STDIN_FILENO, rmaster);
      }
      // 如果还有其他文件描述符，则执行 select
      do_select = do_select || rmaster->fd_count;
    }

    /* Same thing with exceptions */
    // 检查异常文件描述符集合是否为空
    if (emaster != NULL) {
      // 如果需要读取 stdin 异常，则清除它并记住它
      if (checked_fd_isset(STDIN_FILENO, emaster)) {
        e_stdin = 1;
        checked_fd_clr(STDIN_FILENO, emaster);
      }
      // 如果还有其他异常文件描述符，则执行 select
      do_select = do_select || emaster->fd_count;
    }

    /* stdin can't be written to, so ignore it. */
    // stdin 不能被写入，所以忽略它
    if (wmaster != NULL) {
      assert(!checked_fd_isset(STDIN_FILENO, wmaster));
      // 如果还有其他可写文件描述符，则执行 select
      do_select = do_select || wmaster->fd_count;
    }

    /* Handle the case where stdin is not in scope. */
    // 处理 stdin 不在范围内的情况
    if (!(r_stdin || e_stdin)) {
        if (do_select) {
            /* Do a normal select. */
            // 执行普通的 select
            return select(s, rmaster, wmaster, emaster, tv);
        } else {
            /* No file descriptors given. Just sleep. */
            // 没有给定文件描述符，只是休眠
            if (tv == NULL) {
                /* Sleep forever. */
                // 永久休眠
                while (1)
                    sleep(10000);
            } else {
                // 休眠指定时间
                usleep(tv->tv_sec * 1000000UL + tv->tv_usec);
                return 0;
            }
        }
    }
    /* 这是对 Windows 的一个 hack，因为它不允许在非套接字（如 stdin）上进行 select() 操作。
     * 我们从 fd_set 中移除 stdin，并在其余内容上循环进行 select() 操作，超时时间为 125 毫秒。
     * 然后我们检查 stdin 是否准备好，并增加 fds_ready，并在看起来不错的情况下将 stdin 设置为 rmaster。
     * 我们一直循环直到有结果或超时。
     */

    /* nbase_winunix.c 包含了检查 stdin 是否有输入的所有细节。这涉及到一个后台线程，如果需要的话我们现在就启动。*/
    if (!stdin_thread_started) {
        int ret = win_stdin_start_thread();
        assert(ret != 0);
        stdin_thread_started = 1;
    }

    if (tv) {
        int usecs = (tv->tv_sec * 1000000) + tv->tv_usec;

        iter = usecs / 125000;

        if (usecs % 125000)
            iter++;
    }

    FD_ZERO(&rset);
    FD_ZERO(&wset);
    FD_ZERO(&eset);

    while (!fds_ready && iter) {
        stv.tv_sec = 0;
        stv.tv_usec = 125000;

        if (rmaster)
            rset = *rmaster;
        if (wmaster)
            wset = *wmaster;
        if (emaster)
            eset = *emaster;

        if(r_stdin) {
            stdin_ready = win_stdin_ready();
            if(stdin_ready)
                stv.tv_usec = 0; /* 获取状态但不等待，因为 stdin 已准备好 */
        }

        fds_ready = 0;
        /* 选择除了 stdin 以外的任何内容？ */
        if (do_select)
            fds_ready = select(s, &rset, &wset, &eset, &stv);
        else
            usleep(stv.tv_sec * 1000000UL + stv.tv_usec);

        if (fds_ready > -1 && stdin_ready) {
            checked_fd_set(STDIN_FILENO, &rset);
            fds_ready++;
        }

        if (tv)
            iter--;
    }

    if (rmaster)
        *rmaster = rset;
    if (wmaster)
        *wmaster = wset;
    if (emaster)
        *emaster = eset;

    return fds_ready;
#else
    return select(s, rmaster, wmaster, emaster, tv);
#endif
}

/*
 * CRC32 Cyclic Redundancy Check
 *
 * From: http://www.ietf.org/rfc/rfc1952.txt
 *
 * Copyright (c) 1996 L. Peter Deutsch
 *
 * Permission is granted to copy and distribute this document for any
 * purpose and without charge, including translations into other
 * languages and incorporation into compilations, provided that the
 * copyright notice and this notice are preserved, and that any
 * substantive changes or deletions from the original are clearly
 * marked.
 *
 */

/* Table of CRCs of all 8-bit messages. */
static unsigned long crc_table[256];

/* Flag: has the table been computed? Initially false. */
static int crc_table_computed = 0;

/* Make the table for a fast CRC. */
static void make_crc_table(void)
{
  unsigned long c;
  int n, k;

  for (n = 0; n < 256; n++) {
    c = (unsigned long) n;
    for (k = 0; k < 8; k++) {
      if (c & 1) {
        c = 0xedb88320L ^ (c >> 1);
      } else {
        c = c >> 1;
      }
    }
    crc_table[n] = c;
  }
  crc_table_computed = 1;
}

/*
   Update a running crc with the bytes buf[0..len-1] and return
 the updated crc. The crc should be initialized to zero. Pre- and
 post-conditioning (one's complement) is performed within this
 function so it shouldn't be done by the caller. Usage example:

   unsigned long crc = 0L;

   while (read_buffer(buffer, length) != EOF) {
     crc = update_crc(crc, buffer, length);
   }
   if (crc != original_crc) error();
*/
static unsigned long update_crc(unsigned long crc,
                unsigned char *buf, int len)
{
  unsigned long c = crc ^ 0xffffffffL;
  int n;

  if (!crc_table_computed)
    make_crc_table();
  for (n = 0; n < len; n++) {
    c = crc_table[(c ^ buf[n]) & 0xff] ^ (c >> 8);
  }
  return c ^ 0xffffffffL;
}

/* Return the CRC of the bytes buf[0..len-1]. */
unsigned long nbase_crc32(unsigned char *buf, int len)
{
  return update_crc(0L, buf, len);
}
/*
 * CRC-32C (Castagnoli) Cyclic Redundancy Check.
 * Taken straight from Appendix C of RFC 4960 (SCTP), with the difference that
 * the remainder register (crc32) is initialized to 0xffffffffL rather than ~0L,
 * for correct operation on platforms where unsigned long is longer than 32
 * bits.
 */

/* Return the CRC-32C of the bytes buf[0..len-1] */
unsigned long nbase_crc32c(unsigned char *buf, int len)
{
  int i;
  unsigned long crc32 = 0xffffffffL;  // 初始化 CRC-32C 寄存器为 0xffffffffL
  unsigned long result;
  unsigned char byte0, byte1, byte2, byte3;

  for (i = 0; i < len; i++) {
    CRC32C(crc32, buf[i]);  // 使用 CRC32C 算法更新 CRC-32C 寄存器
  }

  result = ~crc32;  // 对 CRC-32C 寄存器取反

  /*  result now holds the negated polynomial remainder;
   *  since the table and algorithm is "reflected" [williams95].
   *  That is, result has the same value as if we mapped the message
   *  to a polynomial, computed the host-bit-order polynomial
   *  remainder, performed final negation, then did an end-for-end
   *  bit-reversal.
   *  Note that a 32-bit bit-reversal is identical to four inplace
   *  8-bit reversals followed by an end-for-end byteswap.
   *  In other words, the bytes of each bit are in the right order,
   *  but the bytes have been byteswapped.  So we now do an explicit
   *  byteswap.  On a little-endian machine, this byteswap and
   *  the final ntohl cancel out and could be elided.
   */

  byte0 =  result        & 0xff;  // 获取 result 的低 8 位
  byte1 = (result >>  8) & 0xff;  // 获取 result 的第 9-16 位
  byte2 = (result >> 16) & 0xff;  // 获取 result 的第 17-24 位
  byte3 = (result >> 24) & 0xff;  // 获取 result 的高 8 位
  crc32 = ((byte0 << 24) | (byte1 << 16) | (byte2 <<  8) | byte3);  // 重新组合字节顺序
  return crc32;  // 返回 CRC-32C 值
}


/*
 * Adler32 Checksum Calculation.
 * Taken straight from RFC 2960 (SCTP).
 */

#define ADLER32_BASE 65521 /* largest prime smaller than 65536 */

/*
 * Update a running Adler-32 checksum with the bytes buf[0..len-1]
 * and return the updated checksum.  The Adler-32 checksum should
 * be initialized to 1.
 */
static unsigned long update_adler32(unsigned long adler,
                                    unsigned char *buf, int len)
{
  // 初始化 Adler32 校验和算法的两个变量
  unsigned long s1 = adler & 0xffff;
  unsigned long s2 = (adler >> 16) & 0xffff;
  int n;

  // 遍历缓冲区中的字节，更新 Adler32 校验和算法的两个变量
  for (n = 0; n < len; n++) {
    s1 = (s1 + buf[n]) % ADLER32_BASE;
    s2 = (s2 + s1)     % ADLER32_BASE;
  }
  // 返回计算后的 Adler32 校验和
  return (s2 << 16) + s1;
}

/* Return the Adler32 of the bytes buf[0..len-1] */
// 计算 buf 缓冲区中前 len 个字节的 Adler32 校验和
unsigned long nbase_adler32(unsigned char *buf, int len)
{
  return update_adler32(1L, buf, len);
}

#undef ADLER32_BASE


/* This function returns a string containing the hexdump of the supplied
 * buffer. It uses current locale to determine if a character is printable or
 * not. It prints 73char+\n wide lines like these:

0000   e8 60 65 86 d7 86 6d 30  35 97 54 87 ff 67 05 9e  .`e...m05.T..g..
0010   07 5a 98 c0 ea ad 50 d2  62 4f 7b ff e1 34 f8 fc  .Z....P.bO{..4..
0020   c4 84 0a 6a 39 ad 3c 10  63 b2 22 c4 24 40 f4 b1  ...j9.<.c.".$@..

 * The lines look basically like Wireshark's hex dump.
 * WARNING: This function returns a pointer to a DYNAMICALLY allocated buffer
 * that the caller is supposed to free().
 * */
char *hexdump(const u8 *cp, u32 length){
  static char asciify[257];          /* 存储字符表 */
  int asc_init=0;                    /* 生成表的标志，只生成一次 */
  u32 i=0, hex=0, asc=0;             /* 数组索引 */
  u32 line_count=0;                  /* 行起始处的字节计数 */
  char *current_line=NULL;           /* 当前要写入的行 */
  char *buffer=NULL;                 /* 要返回的动态缓冲区 */
  #define LINE_LEN 74                /* 打印行的长度 */
  char line2print[LINE_LEN];         /* 存储当前行 */
  char printbyte[16];                /* 字节转换 */
  int bytes2alloc;                   /* 缓冲区大小 */
  memset(line2print, ' ', LINE_LEN); /* 用空格填充行 */

  /* 在第一次运行时，根据当前区域设置生成可打印字符表 */
  if( asc_init==0){
      asc_init=1;
      for(i=0; i<256; i++){
        if( isalnum(i) || isdigit(i) || ispunct(i) ){ asciify[i]=i; }
        else{ asciify[i]='.'; }
      }
  }
  /* 分配足够的空间来打印十六进制转储 */
  bytes2alloc=(length%16==0)? (1 + LINE_LEN * (length/16)) : (1 + LINE_LEN * (1+(length/16))) ;
  buffer=(char *)safe_zalloc(bytes2alloc);
  current_line=buffer;
#define HEX_START 7
#define ASC_START 57
/* 这是我们的行的样子。
0000   00 01 02 03 04 05 06 07  08 09 0a 0b 0c 0d 0e 0f  .`e...m05.T..g..[\n]
01234567890123456789012345678901234567890123456789012345678901234567890123
0         1         2         3         4         5         6         7
       ^                                                 ^               ^
       |                                                 |               |
    HEX_START                                        ASC_START        换行
*/
  i=0;
  while( i < length ){
    # 用空格填充line2print数组
    memset(line2print, ' ', LINE_LEN); /* Fill line with spaces */
    # 格式化输出行号到line2print数组
    snprintf(line2print, sizeof(line2print), "%04x", (16*line_count++) % 0xFFFF); /* Add line No.*/
    # 用空格替换snprintf()插入的'\0'
    line2print[4]=' '; /* Replace the '\0' inserted by snprintf() with a space */
    hex=HEX_START;  asc=ASC_START;
    # 执行循环，打印16字节的十六进制和ASCII码
    do { /* Print 16 bytes in both hex and ascii */
        # 每8个字节插入一个空格
        if (i%16 == 8) hex++; /* Insert space every 8 bytes */
        # 格式化输出每个字节的十六进制数
        snprintf(printbyte, sizeof(printbyte), "%02x", cp[i]);/* First print the hex number */
        # 将十六进制数添加到line2print数组
        line2print[hex++]=printbyte[0];
        line2print[hex++]=printbyte[1];
        line2print[hex++]=' ';
        # 将ASCII码添加到line2print数组
        line2print[asc++]=asciify[ cp[i] ]; /* Then print its ASCII equivalent */
        i++;
    } while (i < length && i%16 != 0);
    # 将行复制到输出缓冲区
    line2print[LINE_LEN-1]='\n';
    memcpy(current_line, line2print, LINE_LEN);
    current_line += LINE_LEN;
  }
  # 在buffer数组的最后一个字节插入'\0'
  buffer[bytes2alloc-1]='\0';
  # 返回buffer数组
  return buffer;
} /* End of hexdump() */

/* 这个函数类似于strtol或atoi，但它只允许数字。没有空白、符号或基数前缀。 */
long parse_long(const char *s, const char **tail)
{
    if (!isdigit((int) (unsigned char) *s)) {
        *tail = (char *) s;
        return 0;
    }

    return strtol(s, (char **) tail, 10);
}

/* 这个函数接受一个字节计数，并在提供的缓冲区中存储一个短的ASCII等价值。例如：0.122MB，10.322Kb或128B。 */
char *format_bytecount(unsigned long long bytes, char *buf, size_t buflen) {
  assert(buf != NULL);

  if (bytes < 1000)
    Snprintf(buf, buflen, "%uB", (unsigned int) bytes);
  else if (bytes < 1000000)
    Snprintf(buf, buflen, "%.3fKB", bytes / 1000.0);
  else
    Snprintf(buf, buflen, "%.3fMB", bytes / 1000000.0);

  return buf;
}

/* 如果给定的文件路径存在，不是目录，并且可被执行进程读取，则返回1。如果可读并且是目录，则返回2。否则返回0。 */
int file_is_readable(const char *pathname) {
    char *pathname_buf = strdup(pathname);
    int status = 0;
    struct stat st;

#ifdef WIN32
    // 在Windows上，stat仅适用于"dir_name"，而不适用于"dir_name/"或"dir_name\\"
    int pathname_len = strlen(pathname_buf);
    char last_char = pathname_buf[pathname_len - 1];

    if(    last_char == '/'
        || last_char == '\\')
        pathname_buf[pathname_len - 1] = '\0';

#endif

  if (stat(pathname_buf, &st) == -1)
    status = 0;
  else if (access(pathname_buf, R_OK) != -1)
    status = S_ISDIR(st.st_mode) ? 2 : 1;

  free(pathname_buf);
  return status;
}

#if HAVE_PROC_SELF_EXE
static char *executable_path_proc_self_exe(void) {
  char buf[1024];
  char *path;
  int n;

  n = readlink("/proc/self/exe", buf, sizeof(buf));
  if (n < 0 || n >= sizeof(buf))
    return NULL;
  path = (char *) safe_malloc(n + 1);
  /* readlink不会添加空字符。 */
  memcpy(path, buf, n);
  path[n] = '\0';

  return path;
}
#endif
#if HAVE_MACH_O_DYLD_H
#include <mach-o/dyld.h>
/* 如果有 mach-o/dyld.h 头文件，则包含该头文件 */
static char *executable_path_NSGetExecutablePath(void) {
  char buf[1024];
  uint32_t size;

  size = sizeof(buf);
  // 获取可执行文件的路径并存储在 buf 中
  if (_NSGetExecutablePath(buf, &size) == 0)
    return strdup(buf);
  else
    return NULL;
}
#endif

#if WIN32
static char *executable_path_GetModuleFileName(void) {
  char buf[1024];
  int n;

  // 获取模块文件的路径并存储在 buf 中
  n = GetModuleFileName(GetModuleHandle(0), buf, sizeof(buf));
  if (n <= 0 || n >= sizeof(buf))
    return NULL;

  return strdup(buf);
}
#endif

static char *executable_path_argv0(const char *argv0) {
  if (argv0 == NULL)
    return NULL;
  /* 如果 argv[0] 包含目录分隔符，则可以从中获取路径。(否则在 $PATH 中查找) */
  if (strchr(argv0, '/') != NULL)
    return strdup(argv0);
#if WIN32
  if (strchr(argv0, '\\') != NULL)
    return strdup(argv0);
#endif
  return NULL;
}

char *executable_path(const char *argv0) {
  char *path;

  path = NULL;
#if HAVE_PROC_SELF_EXE
  if (path == NULL)
    path = executable_path_proc_self_exe();
#endif
#if HAVE_MACH_O_DYLD_H
  if (path == NULL)
    path = executable_path_NSGetExecutablePath();
#endif
#if WIN32
  if (path == NULL)
    path = executable_path_GetModuleFileName();
#endif
  if (path == NULL)
    path = executable_path_argv0(argv0);

  return path;
}

int sockaddr_storage_inet_pton(const char * ip_str, struct sockaddr_storage * addr)
{
  struct sockaddr_in * addrv4p = (struct sockaddr_in *) addr;
#if HAVE_IPV6
  struct sockaddr_in6 * addrv6p = (struct sockaddr_in6 *) addr;
  if ( 1 == inet_pton(AF_INET6, ip_str, &(addrv6p->sin6_addr)) )
  {
    addr->ss_family = AF_INET6;
    return 1;
  }
#endif // HAVE_IPV6

  if ( 1 == inet_pton(AF_INET, ip_str, &(addrv4p->sin_addr)) )
  {
    addr->ss_family = AF_INET;
    return 1;
  }

  return 0;
}

const char *sockaddr_storage_iptop(const struct sockaddr_storage * addr, char * dst)
{
  switch (addr->ss_family){
  case AF_INET:
  {
    // 将 IPv4 地址转换为字符串格式并存储在 dst 中
    # 定义一个指向 sockaddr_in 结构体的指针，指向传入的地址结构体
    const struct sockaddr_in * ipv4_ptr = (const struct sockaddr_in *) addr;
    # 调用 inet_ntop 函数将地址转换为可读的字符串格式，并存储到 dst 中
    return inet_ntop(addr->ss_family, &(ipv4_ptr->sin_addr), dst, INET_ADDRSTRLEN);
  }
#if HAVE_IPV6
  // 如果支持 IPv6，则执行以下代码
  case AF_INET6:
  {
    // 将地址转换为 IPv6 地址结构
    const struct sockaddr_in6 * addrv6p = (struct sockaddr_in6 *) addr;
    // 将 IPv6 地址转换为可读的字符串格式
    return inet_ntop(addr->ss_family, &(addrv6p->sin6_addr), dst, INET6_ADDRSTRLEN);
  }
#endif
  // 如果不支持 IPv6，则执行以下代码
  default:
  {
    // 返回空指针
    return NULL;
  }}
}
```