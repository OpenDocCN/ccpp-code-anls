# `nmap\libdnet-stripped\include\dnet_winconfig.h`

```cpp
/* include/dnet_winconfig.h -- Windows configuration #defines.  It is modified
   from the config.h generated by configure on other platforms. */

/* Define if arpreq struct has arp_dev. */
#define HAVE_ARPREQ_ARP_DEV 1

/* Define if you have the Berkeley Packet Filter. */
/* #undef HAVE_BSD_BPF */

/* Define if you have the <dlfcn.h> header file. */
#define HAVE_DLFCN_H 1

/* Define if you have the `err' function. */
#define HAVE_ERR 1

/* Define if you have the <fcntl.h> header file. */
#define HAVE_FCNTL_H 1

/* Define if you have the <hpsecurity.h> header file. */
/* #undef HAVE_HPSECURITY_H */

/* Define if you have the <inttypes.h> header file. */
#define HAVE_INTTYPES_H 1

/* Define if you have arp(7) ioctls. */
#define HAVE_IOCTL_ARP 1

/* Define if you have the <Iphlpapi.h> header file. */
/* #undef HAVE_IPHLPAPI_H */

/* Define if you have the <ip_compat.h> header file. */
/* #undef HAVE_IP_COMPAT_H */

/* Define if you have the <ip_fil_compat.h> header file. */
/* #undef HAVE_IP_FIL_COMPAT_H */

/* Define if you have the <ip_fil.h> header file. */
/* #undef HAVE_IP_FIL_H */

/* Define if you have the `iphlpapi' library (-liphlpapi). */
/* #undef HAVE_LIBIPHLPAPI */

/* Define if you have the `nm' library (-lnm). */
/* #undef HAVE_LIBNM */

/* Define if you have the `nsl' library (-lnsl). */
/* #undef HAVE_LIBNSL */

/* Define if you have the `resolv' library (-lresolv). */
/* #undef HAVE_LIBRESOLV */

/* Define if you have the `socket' library (-lsocket). */
/* #undef HAVE_LIBSOCKET */

/* Define if you have the `str' library (-lstr). */
/* #undef HAVE_LIBSTR */

/* Define if you have the `ws2_32' library (-lws2_32). */
/* #undef HAVE_LIBWS2_32 */

/* Define if you have the <linux/if_tun.h> header file. */
#define HAVE_LINUX_IF_TUN_H 1

/* Define if you have the <linux/ip_fwchains.h> header file. */
/* #undef HAVE_LINUX_IP_FWCHAINS_H */

/* Define if you have the <linux/ip_fw.h> header file. */
/* #undef HAVE_LINUX_IP_FW_H */
/* 如果你有 <linux/netfilter_ipv4/ipchains_core.h> 头文件，则定义为 1 */
#define HAVE_LINUX_NETFILTER_IPV4_IPCHAINS_CORE_H 1

/* 如果你有 Linux PF_PACKET sockets，则定义为 1 */
#define HAVE_LINUX_PF_PACKET 1

/* 如果你有 Linux /proc 文件系统，则定义为 1 */
#define HAVE_LINUX_PROCFS 1

/* 如果你有 <memory.h> 头文件，则定义为 1 */
#define HAVE_MEMORY_H 1

/* 如果你有 <netinet/in_var.h> 头文件，则定义为 1 */
/* #undef HAVE_NETINET_IN_VAR_H */

/* 如果你有 <netinet/ip_compat.h> 头文件，则定义为 1 */
/* #undef HAVE_NETINET_IP_COMPAT_H */

/* 如果你有 <netinet/ip_fil_compat.h> 头文件，则定义为 1 */
/* #undef HAVE_NETINET_IP_FIL_COMPAT_H */

/* 如果你有 <netinet/ip_fil.h> 头文件，则定义为 1 */
/* #undef HAVE_NETINET_IP_FIL_H */

/* 如果你有 <netinet/ip_fw.h> 头文件，则定义为 1 */
/* #undef HAVE_NETINET_IP_FW_H */

/* 如果你有 <net/bpf.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_BPF_H */

/* 如果你有 <net/if_arp.h> 头文件，则定义为 1 */
#define HAVE_NET_IF_ARP_H 1

/* 如果你有 <net/if_dl.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_IF_DL_H */

/* 如果你有 <net/if.h> 头文件，则定义为 1 */
// #define HAVE_NET_IF_H 1

/* 如果你有 <net/if_tun.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_IF_TUN_H */

/* 如果你有 <net/if_var.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_IF_VAR_H */

/* 如果你有 <net/pfilt.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_PFILT_H */

/* 如果你有 <net/pfvar.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_PFVAR_H */

/* 如果你有 <net/radix.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_RADIX_H */

/* 如果你有 <net/raw.h> 头文件，则定义为 1 */
/* #undef HAVE_NET_RAW_H */

/* 如果你有 <net/route.h> 头文件，则定义为 1 */
#define HAVE_NET_ROUTE_H 1

/* 如果你有 cooked raw IP sockets，则定义为 1 */
/* #undef HAVE_RAWIP_COOKED */

/* 如果 <sys/kinfo.h> 有 getkerninfo，则定义为 1 */
/* #undef HAVE_GETKERNINFO */
/* Define if raw IP sockets require host byte ordering for ip_off, ip_len. */
/* #undef HAVE_RAWIP_HOST_OFFLEN */

/* Define if <net/route.h> has rt_msghdr struct. */
/* #undef HAVE_ROUTE_RT_MSGHDR */

/* Define if <netinet/in.h> has sockaddr_in6 struct. */
#define HAVE_SOCKADDR_IN6 1

/* Define if sockaddr struct has sa_len. */
/* #undef HAVE_SOCKADDR_SA_LEN */

/* Define if you have the <stdint.h> header file. */
#define HAVE_STDINT_H 1

/* Define if you have the <stdlib.h> header file. */
#define HAVE_STDLIB_H 1

/* Define if you have SNMP MIB2 STREAMS. */
/* #undef HAVE_STREAMS_MIB2 */

/* Define if you have route(7) STREAMS. */
/* #undef HAVE_STREAMS_ROUTE */

/* Define if you have the <strings.h> header file. */
#define HAVE_STRINGS_H 1

/* Define if you have the <string.h> header file. */
#define HAVE_STRING_H 1

/* Define if you have the `strlcpy' function. */
/* #undef HAVE_STRLCPY */

/* Define if you have the <stropts.h> header file. */
#define HAVE_STROPTS_H 1

/* Define if you have the `strsep' function. */
#define HAVE_STRSEP 1

/* Define if you have the <sys/bufmod.h> header file. */
/* #undef HAVE_SYS_BUFMOD_H */

/* Define if you have the <sys/dlpihdr.h> header file. */
/* #undef HAVE_SYS_DLPIHDR_H */

/* Define if you have the <sys/dlpi_ext.h> header file. */
/* #undef HAVE_SYS_DLPI_EXT_H */

/* Define if you have the <sys/dlpi.h> header file. */
/* #undef HAVE_SYS_DLPI_H */

/* Define if you have the <sys/ioctl.h> header file. */
#define HAVE_SYS_IOCTL_H 1

/* Define if you have the <sys/mib.h> header file. */
/* #undef HAVE_SYS_MIB_H */

/* Define if you have the <sys/ndd_var.h> header file. */
/* #undef HAVE_SYS_NDD_VAR_H */

/* Define if you have the <sys/socket.h> header file. */
/* #undef HAVE_SYS_SOCKET_H */

/* Define if you have the <sys/sockio.h> header file. */
/* #undef HAVE_SYS_SOCKIO_H */

/* Define if you have the <sys/stat.h> header file. */
#define HAVE_SYS_STAT_H 1

/* Define if you have the <sys/sysctl.h> header file. */
/* 定义是否有 <sys/sysctl.h> 头文件 */
#define HAVE_SYS_SYSCTL_H 1

/* 定义是否有 <sys/time.h> 头文件 */
#define HAVE_SYS_TIME_H 1

/* 定义是否有 <sys/types.h> 头文件 */
#define HAVE_SYS_TYPES_H 1

/* 定义是否有 <unistd.h> 头文件 */
#define HAVE_UNISTD_H 1

/* 定义是否有 <winsock2.h> 头文件 */
#define HAVE_WINSOCK2_H

/* 软件包名称 */
#define PACKAGE "libdnet"

/* 定义是否有 ANSI C 头文件 */
#define STDC_HEADERS 1

/* 软件包版本号 */
#define VERSION "1.12"

/* 用于更快的代码生成 */
#define WIN32_LEAN_AND_MEAN

/* 如果 'const' 不符合 ANSI C 标准，则定义为空 */
/* #undef const */

/* 如果 C 编译器将内联函数称为 `__inline`，则定义为 `__inline`，否则为空 */
/* #undef inline */

/* 如果 <sys/types.h> 未定义，则定义为 `int` */
/* #undef pid_t */

/* 如果 <sys/types.h> 未定义，则定义为 `unsigned` */
/* #undef size_t */

/* 使用 MingW32 的内部 snprintf */
/* #undef snprintf */

#include <sys/types.h>

#ifdef HAVE_WINSOCK2_H
# include <winsock2.h>
# include <ws2tcpip.h>
# include <windows.h>
#endif

#ifdef __svr4__
# define BSD_COMP    1
#endif

#if defined(__osf__) && !defined(_SOCKADDR_LEN)
# define _SOCKADDR_LEN    1
#endif

#ifndef HAVE_STRLCPY
int    strlcpy(char *, const char *, int);
#endif

#ifndef HAVE_STRSEP
char    *strsep(char **, const char *);
#endif

#if defined(_MSC_VER) && _MSC_VER < 1900
#define snprintf _snprintf
#endif

/* 没有这个，Windows 将会在使用像 strcpy() 这样的函数时给出各种错误 */
#define _CRT_SECURE_NO_DEPRECATE 1
```