# `nmap\nbase\nbase_time.c`

```cpp
/* $Id$ */
// 包含必要的头文件
#include "nbase.h"
#if HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <time.h>
#ifdef WIN32
#include <sys/timeb.h>
#include <winsock2.h>
#endif

#ifndef HAVE_USLEEP
// 如果没有定义 usleep 函数，则定义一个
void usleep(unsigned long usec) {
#ifdef HAVE_NANOSLEEP
// 如果有 nanosleep 函数，则使用 nanosleep 实现 usleep
struct timespec ts;
ts.tv_sec = usec / 1000000;
ts.tv_nsec = (usec % 1000000) * 1000;
nanosleep(&ts, NULL);
#else /* Windows style */
// 否则在 Windows 下使用 Sleep 函数实现 usleep
 Sleep( usec / 1000 );
#endif /* HAVE_NANOSLEEP */
}
#endif

/* Thread safe time stuff */
#ifdef WIN32
// 在 Windows 下使用 CRT 函数实现线程安全的时间函数
int n_localtime(const time_t *timer, struct tm *result) {
  return localtime_s(result, timer);
}

int n_gmtime(const time_t *timer, struct tm *result) {
  return gmtime_s(result, timer);
}

int n_ctime(char *buffer, size_t bufsz, const time_t *timer) {
  return ctime_s(buffer, bufsz, timer);
}

#else /* WIN32 */

#include <errno.h>
#ifdef HAVE_LOCALTIME_S
// 如果有 localtime_s 函数，则使用它实现线程安全的时间函数
int n_localtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = localtime_s(timer, result);
  if (!tmp) {
    return errno;
  }
  return 0;
}

int n_gmtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = gmtime_s(timer, result);
  if (!tmp) {
    return errno;
  }
  return 0;
}

int n_ctime(char *buffer, size_t bufsz, const time_t *timer) {
  return ctime_s(buffer, bufsz, timer);
}
#else
#ifdef HAVE_LOCALTIME_R
// 如果有 localtime_r 函数，则使用它实现线程安全的时间函数
int n_localtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = localtime_r(timer, result);
  if (!tmp) {
    return errno;
  }
  return 0;
}

int n_gmtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = gmtime_r(timer, result);
  if (!tmp) {
    # 返回错误代码
    return errno;
  }
  # 返回 0 表示没有错误
  return 0;
#else
/* No thread-safe alternatives. */
// 使用 C99 的单行注释，因为 LGTM.com 不识别 C 风格的块注释。这可能会导致问题，但只有在没有 C99 兼容编译器的非常老的系统上才会出现问题，这些系统没有 localtime_r 或 localtime_s 函数。
int n_localtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = localtime(timer); // lgtm[cpp/potentially-dangerous-function]
  if (tmp)
    *result = *tmp;
  else
    return errno;
  return 0;
}

int n_gmtime(const time_t *timer, struct tm *result) {
  struct tm *tmp = gmtime(timer); // lgtm[cpp/potentially-dangerous-function]
  if (tmp)
    *result = *tmp;
  else
    return errno;
  return 0;
}

int n_ctime(char *buffer, size_t bufsz, const time_t *timer) {
  char *tmp = ctime(timer); // lgtm[cpp/potentially-dangerous-function]
  if (tmp)
    Strncpy(buffer, tmp, bufsz);
  else
    return errno;
  return 0;
}
#endif /* HAVE_LOCALTIME_R */
#endif /* HAVE_LOCALTIME_S */
#endif /* WIN32 */

#ifdef WIN32
int gettimeofday(struct timeval *tv, struct timeval *tz)
{
  struct _timeb timebuffer;

  _ftime( &timebuffer );

  tv->tv_sec = (long) timebuffer.time;
  tv->tv_usec = timebuffer.millitm * 1000;
  return 0;
};

unsigned int sleep(unsigned int seconds)
{
  Sleep(1000*seconds);
  return(0);
};
#endif /* WIN32 */
```