# `nmap\nsock\include\nsock_winconfig.h`

```
#ifndef NSOCK_WINCONFIG_H
#define NSOCK_WINCONFIG_H
// 如果 NSOCK_PCAP 没有被禁用，则定义 HAVE_PCAP 为 1
#ifndef DISABLE_NSOCK_PCAP
#define HAVE_PCAP 1
#endif

/* Need this for _WIN32_WINNT below */
#include <nbase.h>
 /* WSAPoll() isn't available before Vista */
#if defined(_WIN32_WINNT) && (_WIN32_WINNT >= 0x0600)
// 如果 _WIN32_WINNT 宏定义大于等于 0x0600，则定义 HAVE_POLL 和 HAVE_IOCP 为 1
#define HAVE_POLL 1
#define HAVE_IOCP 1
#endif

#endif /* NSOCK_WINCONFIG_H */
```