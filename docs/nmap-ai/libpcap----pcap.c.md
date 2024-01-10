# `nmap\libpcap\pcap.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 */
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *    This product includes software developed by the Computer Systems
 *    Engineering Group at Lawrence Berkeley Laboratory.
 * 4. Neither the name of the University nor of the Laboratory may be used
 *    to endorse or promote products derived from this software without
 *    specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>
#ifndef _WIN32
#include <sys/param.h>
#ifndef MSDOS
#include <sys/file.h>  // 包含系统文件操作相关的头文件
#endif
#include <sys/ioctl.h>  // 包含系统 IO 控制相关的头文件
#include <sys/socket.h>  // 包含系统套接字相关的头文件
#ifdef HAVE_SYS_SOCKIO_H
#include <sys/sockio.h>  // 如果定义了 HAVE_SYS_SOCKIO_H，则包含系统套接字 IO 相关的头文件
#endif

struct mbuf;        /* Squelch compiler warnings on some platforms for */
struct rtentry;        /* declarations in <net/if.h> */
#include <net/if.h>  // 包含网络接口相关的头文件
#include <netinet/in.h>  // 包含网络地址相关的头文件
#endif /* _WIN32 */

#include <stdio.h>  // 包含标准输入输出相关的头文件
#include <stdlib.h>  // 包含标准库相关的头文件
#include <string.h>  // 包含字符串处理相关的头文件
#if !defined(_MSC_VER) && !defined(__BORLANDC__) && !defined(__MINGW32__)
#include <unistd.h>  // 如果不是在 Windows 平台上编译，则包含系统调用相关的头文件
#endif
#include <fcntl.h>  // 包含文件控制相关的头文件
#include <errno.h>  // 包含错误码相关的头文件
#include <limits.h>  // 包含整数类型的限制相关的头文件

#include "diag-control.h"  // 包含诊断控制相关的头文件

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"  // 如果定义了 HAVE_OS_PROTO_H，则包含操作系统协议相关的头文件
#endif

#ifdef MSDOS
#include "pcap-dos.h"  // 如果是在 MSDOS 平台上编译，则包含 MSDOS 相关的头文件
#endif

#include "pcap-int.h"  // 包含 pcap 库内部实现相关的头文件

#include "optimize.h"  // 包含优化相关的头文件

#ifdef HAVE_DAG_API
#include "pcap-dag.h"  // 如果定义了 HAVE_DAG_API，则包含 DAG 相关的头文件
#endif /* HAVE_DAG_API */

#ifdef HAVE_SEPTEL_API
#include "pcap-septel.h"  // 如果定义了 HAVE_SEPTEL_API，则包含 SEPTEL 相关的头文件
#endif /* HAVE_SEPTEL_API */

#ifdef HAVE_SNF_API
#include "pcap-snf.h"  // 如果定义了 HAVE_SNF_API，则包含 SNF 相关的头文件
#endif /* HAVE_SNF_API */

#ifdef HAVE_TC_API
#include "pcap-tc.h"  // 如果定义了 HAVE_TC_API，则包含 TC 相关的头文件
#endif /* HAVE_TC_API */

#ifdef PCAP_SUPPORT_LINUX_USBMON
#include "pcap-usb-linux.h"  // 如果定义了 PCAP_SUPPORT_LINUX_USBMON，则包含 Linux USB 相关的头文件
#endif

#ifdef PCAP_SUPPORT_BT
#include "pcap-bt-linux.h"  // 如果定义了 PCAP_SUPPORT_BT，则包含蓝牙 Linux 相关的头文件
#endif

#ifdef PCAP_SUPPORT_BT_MONITOR
#include "pcap-bt-monitor-linux.h"  // 如果定义了 PCAP_SUPPORT_BT_MONITOR，则包含蓝牙监视器 Linux 相关的头文件
#endif

#ifdef PCAP_SUPPORT_NETFILTER
#include "pcap-netfilter-linux.h"  // 如果定义了 PCAP_SUPPORT_NETFILTER，则包含网络过滤器 Linux 相关的头文件
#endif

#ifdef PCAP_SUPPORT_NETMAP
#include "pcap-netmap.h"  // 如果定义了 PCAP_SUPPORT_NETMAP，则包含网络映射相关的头文件
#endif

#ifdef PCAP_SUPPORT_DBUS
#include "pcap-dbus.h"  // 如果定义了 PCAP_SUPPORT_DBUS，则包含 DBus 相关的头文件
#endif

#ifdef PCAP_SUPPORT_RDMASNIFF
#include "pcap-rdmasniff.h"  // 如果定义了 PCAP_SUPPORT_RDMASNIFF，则包含 RDMA 抓包相关的头文件
#endif

#ifdef PCAP_SUPPORT_DPDK
#include "pcap-dpdk.h"  // 如果定义了 PCAP_SUPPORT_DPDK，则包含 DPDK 相关的头文件
#endif

#ifdef HAVE_AIRPCAP_API
#include "pcap-airpcap.h"  // 如果定义了 HAVE_AIRPCAP_API，则包含 AirPcap 相关的头文件
#endif

#ifdef _WIN32
/*
 * To quote the WSAStartup() documentation:
 *
 *   The WSAStartup function typically leads to protocol-specific helper
 *   DLLs being loaded. As a result, the WSAStartup function should not
 *   be called from the DllMain function in a application DLL. This can
 *   potentially cause deadlocks.
 *
 * and the WSACleanup() documentation:
 *
 *   The WSACleanup function typically leads to protocol-specific helper
 *   DLLs being unloaded. As a result, the WSACleanup function should not
 *   be called from the DllMain function in a application DLL. This can
 *   potentially cause deadlocks.
 *
 * So we don't initialize Winsock in a DllMain() routine.
 *
 * pcap_init() should be called to initialize pcap on both UN*X and
 * Windows; it will initialize Winsock on Windows.  (It will also be
 * initialized as needed if pcap_init() hasn't been called.)
 */

/*
 * Start Winsock.
 * Internal routine.
 */
static int
internal_wsockinit(char *errbuf)
{
    WORD wVersionRequested;  // 用于指定所需的 Winsock 版本
    WSADATA wsaData;  // 用于接收 Winsock 的详细信息
    static int err = -1;  // 错误码，默认为 -1
    static int done = 0;  // 标记是否已经初始化
    int status;  // 用于保存函数调用的状态

    if (done)  // 如果已经初始化过，则直接返回错误码
        return (err);

    /*
     * Versions of Windows that don't support Winsock 2.2 are
     * too old for us.
     */
    wVersionRequested = MAKEWORD(2, 2);  // 指定所需的 Winsock 版本为 2.2
    status = WSAStartup(wVersionRequested, &wsaData);  // 初始化 Winsock
    done = 1;  // 标记已经初始化
    if (status != 0) {  // 如果初始化失败
        if (errbuf != NULL) {  // 如果错误缓冲区不为空
            pcap_fmt_errmsg_for_win32_err(errbuf, PCAP_ERRBUF_SIZE,
                status, "WSAStartup() failed");  // 格式化错误消息
        }
        return (err);  // 返回错误码
    }
    atexit ((void(*)(void))WSACleanup);  // 在程序退出时调用 WSACleanup
    err = 0;  // 设置错误码为 0，表示初始化成功
    return (err);  // 返回错误码
}

/*
 * Exported in case some applications using WinPcap/Npcap called it,
 * even though it wasn't exported.
 */
int
wsockinit(void)
{
    return (internal_wsockinit(NULL));  // 调用内部函数进行初始化
}

/*
 * This is the exported function; new programs should call this.
 * *Newer* programs should call pcap_init().
 */
int
pcap_wsockinit(void)
{
    return (internal_wsockinit(NULL));  // 调用内部函数进行初始化
}
#endif /* _WIN32 */
/*
 * 初始化 libpcap 所需的任何初始化工作。
 *
 * 参数指定我们在字符串上使用本地代码页还是 UTF-8；
 * 在 UN*X 上，我们假设在编码会有影响的地方使用 UTF-8，
 * 而在 Windows 上，我们对于 PCAP_CHAR_ENC_LOCAL 使用本地代码页，
 * 对于 PCAP_CHAR_ENC_UTF_8 使用 UTF-8。
 *
 * 在 Windows 上，我们还禁用了 pcap_create() 中处理被传递 UTF-16 字符串的方法，
 * 因为如果用户调用这个函数，他们明确声明他们将要传递本地代码页字符串或 UTF-8 字符串，
 * 所以我们不需要允许传递 UTF-16LE 字符串。为了保险起见，在 Windows 和 UN*X 上，
 * 我们禁用了 pcap_lookupdev()，以防止任何人甚至尝试将 pcap_lookupdev() 的结果
 * （在 Windows 上可能是 UTF-16LE，出于丑陋的兼容性原因）传递给 pcap_create()、
 * pcap_open_live() 或 pcap_open()。
 *
 * 成功返回 0，错误返回 -1。
 */
int pcap_new_api;        /* pcap_lookupdev() 总是失败 */
int pcap_utf_8_mode;        /* 字符串应该是 UTF-8 */

int
pcap_init(unsigned int opts, char *errbuf)
{
    static int initialized;

    /*
     * 不允许多次调用设置不同模式；这可能意味着一个库以一种模式初始化 pcap，
     * 而使用该库的程序或该程序使用的另一个库以另一种模式初始化 pcap。
     */
    switch (opts) {

    case PCAP_CHAR_ENC_LOCAL:
        /* 不启用 "UTF-8 模式"。 */
        if (initialized) {
            if (pcap_utf_8_mode) {
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "多次调用 pcap_init 时使用不同的字符编码");
                return (PCAP_ERROR);
            }
        }
        break;
    case PCAP_CHAR_ENC_UTF_8:
        /* 如果字符编码为 UTF-8，则打开“UTF-8模式” */
        if (initialized) {
            /* 如果已经初始化，则检查是否已经打开了 UTF-8 模式 */
            if (!pcap_utf_8_mode) {
                /* 如果没有打开 UTF-8 模式，则返回错误信息 */
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "Multiple pcap_init calls with different character encodings");
                return (PCAP_ERROR);
            }
        }
        /* 打开 UTF-8 模式 */
        pcap_utf_8_mode = 1;
        break;

    default:
        /* 如果字符编码不是 UTF-8，则返回错误信息 */
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "Unknown options specified");
        return (PCAP_ERROR);
    }

    /*
     * 打开适当的模式以用于错误消息；这些例程也用于 rpcapd，它无法访问 pcap 的内部 UTF-8 模式标志，
     * 因此我们必须调用一个例程来设置其 UTF-8 模式标志。
     */
    pcap_fmt_set_encoding(opts);

    if (initialized) {
        /*
         * 没有更多的事情要做；例如，在 Windows 上，我们已经初始化了 Winsock。
         */
        return (0);
    }
#ifdef _WIN32
    /*
     * 现在设置 Winsock。
     */
    if (internal_wsockinit(errbuf) == -1) {
        /* 失败。 */
        return (PCAP_ERROR);
    }
#endif

    /*
     * 完成。
     */
    initialized = 1;
    pcap_new_api = 1;
    return (0);
}

/*
 * 包含库版本的字符串。
 * 不通过头文件显式导出 - 使用的正确 API 是 pcap_lib_version() - 但有些程序包含它，所以我们提供它。
 *
 * 我们在这里声明它，就在定义它之前，以消除编译器可能发出的关于缺少声明的警告。
 */
PCAP_API char pcap_version[];
PCAP_API_DEF char pcap_version[] = PACKAGE_VERSION;

static void
pcap_set_not_initialized_message(pcap_t *pcap)
{
    if (pcap->activated) {
        /* 模块可能忘记设置函数指针 */
        (void)snprintf(pcap->errbuf, sizeof(pcap->errbuf),
            "该设备未正确处理此操作");
        return;
    }
    /* 以防调用者不检查 PCAP_ERROR_NOT_ACTIVATED */
    (void)snprintf(pcap->errbuf, sizeof(pcap->errbuf),
        "该句柄尚未激活");
}

static int
pcap_read_not_initialized(pcap_t *pcap, int cnt _U_, pcap_handler callback _U_,
    u_char *user _U_)
{
    pcap_set_not_initialized_message(pcap);
    /* 这意味着 '未初始化' */
    return (PCAP_ERROR_NOT_ACTIVATED);
}

static int
pcap_inject_not_initialized(pcap_t *pcap, const void * buf _U_, int size _U_)
{
    pcap_set_not_initialized_message(pcap);
    /* 这意味着 '未初始化' */
    return (PCAP_ERROR_NOT_ACTIVATED);
}

static int
pcap_setfilter_not_initialized(pcap_t *pcap, struct bpf_program *fp _U_)
{
    pcap_set_not_initialized_message(pcap);
    /* 这意味着 '未初始化' */
    return (PCAP_ERROR_NOT_ACTIVATED);
}

static int
pcap_setdirection_not_initialized(pcap_t *pcap, pcap_direction_t d _U_)
{
    pcap_set_not_initialized_message(pcap);
    # 这表示“未初始化”
    return (PCAP_ERROR_NOT_ACTIVATED);
# 设置数据链路类型未初始化时的处理函数
static int
pcap_set_datalink_not_initialized(pcap_t *pcap, int dlt _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 获取非阻塞模式未初始化时的处理函数
static int
pcap_getnonblock_not_initialized(pcap_t *pcap)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 获取统计信息未初始化时的处理函数
static int
pcap_stats_not_initialized(pcap_t *pcap, struct pcap_stat *ps _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# Windows平台下获取扩展统计信息未初始化时的处理函数
#ifdef _WIN32
static struct pcap_stat *
pcap_stats_ex_not_initialized(pcap_t *pcap, int *pcap_stat_size _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回空指针
    return (NULL);
}

# 设置缓冲区大小未初始化时的处理函数
static int
pcap_setbuff_not_initialized(pcap_t *pcap, int dim _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 设置模式未初始化时的处理函数
static int
pcap_setmode_not_initialized(pcap_t *pcap, int mode _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 设置最小拷贝大小未初始化时的处理函数
static int
pcap_setmintocopy_not_initialized(pcap_t *pcap, int size _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 获取事件句柄未初始化时的处理函数
static HANDLE
pcap_getevent_not_initialized(pcap_t *pcap)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回无效句柄值
    return (INVALID_HANDLE_VALUE);
}

# 获取OID请求未初始化时的处理函数
static int
pcap_oid_get_request_not_initialized(pcap_t *pcap, bpf_u_int32 oid _U_,
    void *data _U_, size_t *lenp _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 设置OID请求未初始化时的处理函数
static int
pcap_oid_set_request_not_initialized(pcap_t *pcap, bpf_u_int32 oid _U_,
    const void *data _U_, size_t *lenp _U_)
{
    # 设置未初始化消息
    pcap_set_not_initialized_message(pcap);
    # 返回未激活错误
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 返回未初始化错误
static u_int
# 当发送队列未初始化时，设置 pcap_t 对象的未初始化消息，并返回 0
pcap_sendqueue_transmit_not_initialized(pcap_t *pcap, pcap_send_queue* queue _U_,
    int sync _U_)
{
    pcap_set_not_initialized_message(pcap);
    return (0);
}

# 当用户缓冲区未初始化时，设置 pcap_t 对象的未初始化消息，并返回 PCAP_ERROR_NOT_ACTIVATED
static int
pcap_setuserbuffer_not_initialized(pcap_t *pcap, int size _U_)
{
    pcap_set_not_initialized_message(pcap);
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 当实时转储未初始化时，设置 pcap_t 对象的未初始化消息，并返回 PCAP_ERROR_NOT_ACTIVATED
static int
pcap_live_dump_not_initialized(pcap_t *pcap, char *filename _U_, int maxsize _U_,
    int maxpacks _U_)
{
    pcap_set_not_initialized_message(pcap);
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 当实时转储结束未初始化时，设置 pcap_t 对象的未初始化消息，并返回 PCAP_ERROR_NOT_ACTIVATED
static int
pcap_live_dump_ended_not_initialized(pcap_t *pcap, int sync _U_)
{
    pcap_set_not_initialized_message(pcap);
    return (PCAP_ERROR_NOT_ACTIVATED);
}

# 当获取 AirPcap 句柄未初始化时，设置 pcap_t 对象的未初始化消息，并返回 NULL
static PAirpcapHandle
pcap_get_airpcap_handle_not_initialized(pcap_t *pcap)
{
    pcap_set_not_initialized_message(pcap);
    return (NULL);
}
#endif

# 返回 1 如果可以在 pcap_t 上设置 rfmon 模式，返回 0 如果不能，在错误时返回 PCAP_ERROR 值
int
pcap_can_set_rfmon(pcap_t *p)
{
    return (p->can_set_rfmon_op(p));
}

# 对于永远不支持 rfmon 模式的系统，返回 0
static int
pcap_cant_set_rfmon(pcap_t *p _U_)
{
    return (0);
}

# 设置 *tstamp_typesp 指向一个或多个支持的时间戳类型的数组；返回值是支持的时间戳类型的数量。
# 当你完成后，应该通过调用 pcap_free_tstamp_types() 来释放列表。
# 返回值为 0 表示“你不能选择时间戳类型”，此时 *tstamp_typesp 被设置为 null。
# 在错误时返回 PCAP_ERROR。
int
pcap_list_tstamp_types(pcap_t *p, int **tstamp_typesp)
{
    # 如果时间戳类型计数为0
    if (p->tstamp_type_count == 0) {
        # 不支持多种时间戳类型
        # 这意味着我们只支持 PCAP_TSTAMP_HOST 类型；
        # 设置一个只包含该类型的列表。
        *tstamp_typesp = (int*)malloc(sizeof(**tstamp_typesp));
        # 分配内存失败
        if (*tstamp_typesp == NULL) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "malloc");
            # 返回错误
            return (PCAP_ERROR);
        }
        **tstamp_typesp = PCAP_TSTAMP_HOST;
        # 返回成功
        return (1);
    } else {
        # 分配内存并清零
        *tstamp_typesp = (int*)calloc(sizeof(**tstamp_typesp),
            p->tstamp_type_count);
        # 分配内存失败
        if (*tstamp_typesp == NULL) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "malloc");
            # 返回错误
            return (PCAP_ERROR);
        }
        # 复制时间戳类型列表到分配的内存中
        (void)memcpy(*tstamp_typesp, p->tstamp_type_list,
            sizeof(**tstamp_typesp) * p->tstamp_type_count);
        # 返回时间戳类型计数
        return (p->tstamp_type_count);
    }
}
/*
 * 在 Windows 中，你可能会有一个使用一个版本的 C 运行时库构建的库，和一个使用另一个版本的 C 运行时库构建的应用程序，这意味着库可能使用一个版本的 malloc() 和 free()，而应用程序可能使用另一个版本的 malloc() 和 free()。如果是这样，那么意味着库分配的内容不能被应用程序释放，因此我们需要一个 pcap_free_tstamp_types() 程序来释放由 pcap_list_tstamp_types() 分配的列表，即使它只是一个 free() 的包装。
 */
void
pcap_free_tstamp_types(int *tstamp_type_list)
{
    free(tstamp_type_list);
}

/*
 * 默认的一次性回调；对于捕获类型，其中包的数据在回调返回后不能保证可用，因此必须进行复制，这个回调将被覆盖。
 */
void
pcap_oneshot(u_char *user, const struct pcap_pkthdr *h, const u_char *pkt)
{
    struct oneshot_userdata *sp = (struct oneshot_userdata *)user;

    *sp->hdr = *h;
    *sp->pkt = pkt;
}

const u_char *
pcap_next(pcap_t *p, struct pcap_pkthdr *h)
{
    struct oneshot_userdata s;
    const u_char *pkt;

    s.hdr = h;
    s.pkt = &pkt;
    s.pd = p;
    if (pcap_dispatch(p, 1, p->oneshot_callback, (u_char *)&s) <= 0)
        return (0);
    return (pkt);
}

int
pcap_next_ex(pcap_t *p, struct pcap_pkthdr **pkt_header,
    const u_char **pkt_data)
{
    struct oneshot_userdata s;

    s.hdr = &p->pcap_header;
    s.pkt = pkt_data;
    s.pd = p;

    /* 保存数据包头的指针 */
    *pkt_header= &p->pcap_header;
    # 如果捕获器的 rfile 属性不为空
    if (p->rfile != NULL) {
        # 定义一个整型变量 status
        int status;

        # 在离线捕获模式下进行捕获
        status = pcap_offline_read(p, 1, p->oneshot_callback,
            (u_char *)&s);

        '''
         * pcap_offline_read() 的返回码:
         *   -  0: 文件结束
         *   - -1: 错误
         *   - >0: OK - 返回捕获的数据包数量，因此在这种情况下将是 1，因为我们传递了最大数据包数量为 1
         * 第一个返回码 ('0') 与 pcap_read() 的返回码 0 冲突，表示“超时前未收到数据包”，因此我们将其映射为 -2，以便区分保存文件的 EOF 和“超时前未收到数据包，重试”的实时捕获
         '''
        if (status == 0)
            return (-2);
        else
            return (status);
    }

    '''
     * pcap_read() 的返回码:
     *   -  0: 超时
     *   - -1: 错误
     *   - -2: 使用 pcap_breakloop() 中断循环
     *   - >0: OK，返回捕获的数据包数量，因此在这种情况下将是 1，因为我们传递了最大数据包数量为 1
     * 第一个返回码 ('0') 与 pcap_offline_read() 的返回码 0 冲突，表示“文件结束”
     '''
    return (p->read_op(p, 1, p->oneshot_callback, (u_char *)&s));
}

/*
 * Implementation of a pcap_if_list_t.
 */
// 定义了一个 pcap_if_list_t 结构体，用于表示网络接口列表
struct pcap_if_list {
    pcap_if_t *beginning;
};

// 定义了一个 capture_source_type 结构体数组，包含了不同类型的捕获源
static struct capture_source_type {
    int (*findalldevs_op)(pcap_if_list_t *, char *);
    pcap_t *(*create_op)(const char *, char *, int *);
} capture_source_types[] = {
#ifdef HAVE_DAG_API
    { dag_findalldevs, dag_create },
#endif
#ifdef HAVE_SEPTEL_API
    { septel_findalldevs, septel_create },
#endif
#ifdef HAVE_SNF_API
    { snf_findalldevs, snf_create },
#endif
#ifdef HAVE_TC_API
    { TcFindAllDevs, TcCreate },
#endif
#ifdef PCAP_SUPPORT_BT
    { bt_findalldevs, bt_create },
#endif
#ifdef PCAP_SUPPORT_BT_MONITOR
    { bt_monitor_findalldevs, bt_monitor_create },
#endif
#ifdef PCAP_SUPPORT_LINUX_USBMON
    { usb_findalldevs, usb_create },
#endif
#ifdef PCAP_SUPPORT_NETFILTER
    { netfilter_findalldevs, netfilter_create },
#endif
#ifdef PCAP_SUPPORT_NETMAP
    { pcap_netmap_findalldevs, pcap_netmap_create },
#endif
#ifdef PCAP_SUPPORT_DBUS
    { dbus_findalldevs, dbus_create },
#endif
#ifdef PCAP_SUPPORT_RDMASNIFF
    { rdmasniff_findalldevs, rdmasniff_create },
#endif
#ifdef PCAP_SUPPORT_DPDK
    { pcap_dpdk_findalldevs, pcap_dpdk_create },
#endif
#ifdef HAVE_AIRPCAP_API
    { airpcap_findalldevs, airpcap_create },
#endif
    { NULL, NULL }
};

/*
 * Get a list of all capture sources that are up and that we can open.
 * Returns -1 on error, 0 otherwise.
 * The list, as returned through "alldevsp", may be null if no interfaces
 * were up and could be opened.
 */
// 获取所有可用的捕获源列表
int
pcap_findalldevs(pcap_if_t **alldevsp, char *errbuf)
{
    size_t i;
    pcap_if_list_t devlist;

    /*
     * Find all the local network interfaces on which we
     * can capture.
     */
    // 初始化网络接口列表
    devlist.beginning = NULL;
    # 如果查找设备失败，则释放之前获取的所有设备条目并返回错误
    if (pcap_platform_finddevs(&devlist, errbuf) == -1) {
        /*
         * 失败 - 在失败之前释放我们获取的所有条目
         */
        if (devlist.beginning != NULL)
            pcap_freealldevs(devlist.beginning);
        *alldevsp = NULL;
        return (-1);
    }

    /*
     * 询问每个非本地网络接口捕获源类型它们有哪些接口
     */
    for (i = 0; capture_source_types[i].findalldevs_op != NULL; i++) {
        if (capture_source_types[i].findalldevs_op(&devlist, errbuf) == -1) {
            /*
             * 出现错误；释放我们正在构建的列表
             */
            if (devlist.beginning != NULL)
                pcap_freealldevs(devlist.beginning);
            *alldevsp = NULL;
            return (-1);
        }
    }

    /*
     * 返回所有设备列表的第一个条目
     */
    *alldevsp = devlist.beginning;
    return (0);
# 复制给定长度的套接字地址结构
static struct sockaddr *
dup_sockaddr(struct sockaddr *sa, size_t sa_length)
{
    struct sockaddr *newsa;

    # 分配给定长度的内存空间
    if ((newsa = malloc(sa_length)) == NULL)
        return (NULL);
    # 将原始套接字地址结构复制到新分配的内存空间中
    return (memcpy(newsa, sa, sa_length));
}

/*
 * 构造接口的“优势指标”，用于在排序接口列表时使用，其中处于启用状态的接口优于未启用的接口，
 * 处于启用和运行状态的接口优于处于启用但未运行的接口，非环回接口优于环回接口，
 * 并且具有相同标志的接口的优势指标较低，实例号越低，优势指标越高。
 *
 * 目标是尝试将最有可能用于捕获的接口放在列表的开头。
 *
 * 优势指标越低表示接口越“好”，如果接口未运行，则设置最高位，如果接口未启用，则设置下面的位，
 * 如果接口是环回接口，则设置下面的位，如果是“任何”接口，则设置下面的位。
 *
 * 注意：我们不按单元号排序，因为 1) 并非所有接口都有单元号（例如，systemd 可能根据接口的 MAC 地址
 * 或适配器连接器的物理位置分配接口名称），2) 如果名称以简单的单元号结尾，它不是接口的全局属性，
 * 它只对具有相同前缀的设备名称有用，因此 xyz0 不一定应该在 abc2 之前排序。这意味着具有相同优势指标的
 * 接口将按照我们获取接口的机制提供它们的顺序进行排序。
 */
static u_int
get_figure_of_merit(pcap_if_t *dev)
{
    u_int n;

    n = 0;
    # 如果接口未运行，则设置最高位
    if (!(dev->flags & PCAP_IF_RUNNING))
        n |= 0x80000000;
    # 如果接口未启用，则设置下面的位
    if (!(dev->flags & PCAP_IF_UP))
        n |= 0x40000000;
    /*
     * 如果非无线接口未断开连接，则给予其比已断开连接的接口更好的优先级，
     * 因为“已断开连接”应该表示接口未连接到网络，因此不会产生任何流量。
     *
     * 对于无线接口，它表示“已关联到网络”，我们假设这并不一定会阻止捕获，
     * 因为您可能会以某种监视模式运行适配器。
     */
    if (!(dev->flags & PCAP_IF_WIRELESS) &&
        (dev->flags & PCAP_IF_CONNECTION_STATUS) == PCAP_IF_CONNECTION_STATUS_DISCONNECTED)
        n |= 0x20000000;

    /*
     * 将环回设备排序在非环回设备之后，*除了*已断开连接的设备。
     */
    if (dev->flags & PCAP_IF_LOOPBACK)
        n |= 0x10000000;

    /*
     * 将“any”设备排序在环回和已断开连接的设备之前，但在所有其他设备之后。
     */
    if (strcmp(dev->name, "any") == 0)
        n |= 0x08000000;

    return (n);
}



#ifndef _WIN32

- 如果不是在 Windows 平台上编译，则编译以下代码


static char *
#ifdef SIOCGIFDESCR

- 定义一个静态的字符指针，如果 SIOCGIFDESCR 被定义了的话


get_if_description(const char *name)

- 定义一个名为 get_if_description 的函数，接受一个字符指针参数 name


{
    char *description = NULL;
    int s;
    struct ifreq ifrdesc;
#ifndef IFDESCRSIZE
    size_t descrlen = 64;
#else
    size_t descrlen = IFDESCRSIZE;
#endif /* IFDESCRSIZE */

- 初始化一个字符指针 description 为 NULL，一个整型变量 s，一个 ifreq 结构体 ifrdesc，一个 size_t 类型的变量 descrlen，如果 IFDESCRSIZE 没有定义，则 descrlen 为 64，否则为 IFDESCRSIZE 的值


    /*
     * Get the description for the interface.
     */
    memset(&ifrdesc, 0, sizeof ifrdesc);
    pcap_strlcpy(ifrdesc.ifr_name, name, sizeof ifrdesc.ifr_name);
    s = socket(AF_INET, SOCK_DGRAM, 0);

- 清空 ifrdesc 结构体的内存，将 name 复制到 ifrdesc.ifr_name 中，创建一个 AF_INET 和 SOCK_DGRAM 类型的套接字并将其赋值给 s


    if (s >= 0) {
#ifdef __FreeBSD__
        /*
         * On FreeBSD, if the buffer isn't big enough for the
         * description, the ioctl succeeds, but the description
         * isn't copied, ifr_buffer.length is set to the description
         * length, and ifr_buffer.buffer is set to NULL.
         */
        for (;;) {
            free(description);
            if ((description = malloc(descrlen)) != NULL) {
                ifrdesc.ifr_buffer.buffer = description;
                ifrdesc.ifr_buffer.length = descrlen;
                if (ioctl(s, SIOCGIFDESCR, &ifrdesc) == 0) {
                    if (ifrdesc.ifr_buffer.buffer ==
                        description)
                        break;
                    else
                        descrlen = ifrdesc.ifr_buffer.length;
                } else {
                    /*
                     * Failed to get interface description.
                     */
                    free(description);
                    description = NULL;
                    break;
                }
            } else
                break;
        }

- 如果 s 大于等于 0，则执行以下代码块，如果是在 FreeBSD 上，则执行以下注释中的代码块，否则跳过
#else /* __FreeBSD__ */
        /*
         * The only other OS that currently supports
         * SIOCGIFDESCR is OpenBSD, and it has no way
         * to get the description length - it's clamped
         * to a maximum of IFDESCRSIZE.
         */
        // 如果不是 FreeBSD 操作系统，则执行以下代码块
        if ((description = malloc(descrlen)) != NULL) {
            // 分配描述长度的内存空间
            ifrdesc.ifr_data = (caddr_t)description;
            // 设置接口描述数据
            if (ioctl(s, SIOCGIFDESCR, &ifrdesc) != 0) {
                /*
                 * Failed to get interface description.
                 */
                // 获取接口描述失败，释放内存空间
                free(description);
                description = NULL;
            }
        }
#endif /* __FreeBSD__ */
        close(s);
        // 关闭套接字
        if (description != NULL && description[0] == '\0') {
            /*
             * Description is empty, so discard it.
             */
            // 如果描述为空，则丢弃它
            free(description);
            description = NULL;
        }
    }

#ifdef __FreeBSD__
    /*
     * For FreeBSD, if we didn't get a description, and this is
     * a device with a name of the form usbusN, label it as a USB
     * bus.
     */
    // 如果是 FreeBSD 操作系统，则执行以下代码块
    if (description == NULL) {
        if (strncmp(name, "usbus", 5) == 0) {
            /*
             * OK, it begins with "usbus".
             */
            // 如果名称以 "usbus" 开头
            long busnum;
            char *p;

            errno = 0;
            busnum = strtol(name + 5, &p, 10);
            if (errno == 0 && p != name + 5 && *p == '\0' &&
                busnum >= 0 && busnum <= INT_MAX) {
                /*
                 * OK, it's a valid number that's not
                 * bigger than INT_MAX.  Construct
                 * a description from it.
                 * (If that fails, we don't worry about
                 * it, we just return NULL.)
                 */
                // 如果是有效的数字，构建描述
                if (pcap_asprintf(&description,
                    "USB bus number %ld", busnum) == -1) {
                    /* Failed. */
                    // 构建描述失败
                    description = NULL;
                }
            }
        }
    }
#endif
    // 返回描述
    return (description);
#else /* SIOCGIFDESCR */
get_if_description(const char *name _U_)
{
    return (NULL);
#endif /* SIOCGIFDESCR */
}

这段代码是一个函数定义，用于获取指定设备的描述信息。如果编译时没有定义 SIOCGIFDESCR，则返回 NULL。


/*
 * Look for a given device in the specified list of devices.
 *
 * If we find it, return a pointer to its entry.
 *
 * If we don't find it, attempt to add an entry for it, with the specified
 * IFF_ flags and description, and, if that succeeds, return a pointer to
 * the new entry, otherwise return NULL and set errbuf to an error message.
 */
pcap_if_t *
find_or_add_if(pcap_if_list_t *devlistp, const char *name,
    bpf_u_int32 if_flags, get_if_flags_func get_flags_func, char *errbuf)
{
    bpf_u_int32 pcap_flags;

    /*
     * Convert IFF_ flags to pcap flags.
     */
    pcap_flags = 0;
#ifdef IFF_LOOPBACK
    if (if_flags & IFF_LOOPBACK)
        pcap_flags |= PCAP_IF_LOOPBACK;
#else
    /*
     * We don't have IFF_LOOPBACK, so look at the device name to
     * see if it looks like a loopback device.
     */
    if (name[0] == 'l' && name[1] == 'o' &&
        (PCAP_ISDIGIT(name[2]) || name[2] == '\0'))
        pcap_flags |= PCAP_IF_LOOPBACK;
#endif
#ifdef IFF_UP
    if (if_flags & IFF_UP)
        pcap_flags |= PCAP_IF_UP;
#endif
#ifdef IFF_RUNNING
    if (if_flags & IFF_RUNNING)
        pcap_flags |= PCAP_IF_RUNNING;
#endif

    /*
     * Attempt to find an entry for this device; if we don't find one,
     * attempt to add one.
     */
    return (find_or_add_dev(devlistp, name, pcap_flags,
        get_flags_func, get_if_description(name), errbuf));
}

这段代码是一个函数定义，用于在指定的设备列表中查找指定的设备。如果找到了指定的设备，则返回指向其条目的指针。如果没有找到，则尝试添加一个条目，如果成功，则返回指向新条目的指针，否则返回 NULL 并设置 errbuf 为错误消息。
/*
 * 在指定的设备列表中查找给定设备。
 *
 * 如果找到设备，则如果指定的地址不为空，则将其添加到设备的地址列表中并返回0。
 *
 * 如果找不到设备，则尝试为其添加一个条目，使用指定的IFF_标志和描述，如果成功，则将指定的地址添加到其地址列表中（如果地址不为空），并返回0；否则返回-1，并将errbuf设置为错误消息。
 *
 * （我们可能会收到一个空地址，因为底层操作系统可能会返回接口名称/地址组合的列表，有时地址会缺失，而不是每个接口都有地址列表，因此这个调用可能是唯一的调用，我们希望添加接口，即使它们没有地址。）
 */
int
add_addr_to_if(pcap_if_list_t *devlistp, const char *name,
    bpf_u_int32 if_flags, get_if_flags_func get_flags_func,
    struct sockaddr *addr, size_t addr_size,
    struct sockaddr *netmask, size_t netmask_size,
    struct sockaddr *broadaddr, size_t broadaddr_size,
    struct sockaddr *dstaddr, size_t dstaddr_size,
    char *errbuf)
{
    pcap_if_t *curdev;

    /*
     * 检查设备是否存在，如果不存在，则添加它。
     */
    curdev = find_or_add_if(devlistp, name, if_flags, get_flags_func,
        errbuf);
    if (curdev == NULL) {
        /*
         * 错误 - 放弃。
         */
        return (-1);
    }

    if (addr == NULL) {
        /*
         * 没有要添加的地址；这个条目只是表示“这是一个新接口”。
         */
        return (0);
    }

    /*
     * “curdev”是这个接口的条目，我们有一个地址；将该地址的条目添加到接口的地址列表中。
     */
    # 调用函数 add_addr_to_dev，并返回其结果
    return (add_addr_to_dev(curdev, addr, addr_size, netmask,
        netmask_size, broadaddr, broadaddr_size, dstaddr,
        dstaddr_size, errbuf));
    # 结束符号，表示条件编译结束
    }
#endif /* _WIN32 */

/*
 * 为一个接口的地址列表添加一个条目。
 * "curdev" 是该接口的条目。
 */
int
add_addr_to_dev(pcap_if_t *curdev,
    struct sockaddr *addr, size_t addr_size,
    struct sockaddr *netmask, size_t netmask_size,
    struct sockaddr *broadaddr, size_t broadaddr_size,
    struct sockaddr *dstaddr, size_t dstaddr_size,
    char *errbuf)
{
    pcap_addr_t *curaddr, *prevaddr, *nextaddr;

    /*
     * 分配新条目并填充它。
     */
    curaddr = (pcap_addr_t *)malloc(sizeof(pcap_addr_t));
    if (curaddr == NULL) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return (-1);
    }

    curaddr->next = NULL;
    # 如果地址不为空且大小不为0，则复制地址并赋值给当前地址的addr字段
    if (addr != NULL && addr_size != 0) {
        curaddr->addr = (struct sockaddr *)dup_sockaddr(addr, addr_size);
        if (curaddr->addr == NULL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            free(curaddr);
            return (-1);
        }
    } else
        curaddr->addr = NULL;

    # 如果子网掩码不为空且大小不为0，则复制子网掩码并赋值给当前地址的netmask字段
    if (netmask != NULL && netmask_size != 0) {
        curaddr->netmask = (struct sockaddr *)dup_sockaddr(netmask, netmask_size);
        if (curaddr->netmask == NULL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            if (curaddr->addr != NULL)
                free(curaddr->addr);
            free(curaddr);
            return (-1);
        }
    } else
        curaddr->netmask = NULL;
    # 如果广播地址不为空且大小不为0
    if (broadaddr != NULL && broadaddr_size != 0) {
        # 为当前地址的广播地址分配内存并复制传入的广播地址
        curaddr->broadaddr = (struct sockaddr *)dup_sockaddr(broadaddr, broadaddr_size);
        # 如果分配内存失败
        if (curaddr->broadaddr == NULL) {
            # 格式化错误消息并返回错误码-1
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            # 释放当前地址的网络掩码
            if (curaddr->netmask != NULL)
                free(curaddr->netmask);
            # 释放当前地址的地址
            if (curaddr->addr != NULL)
                free(curaddr->addr);
            # 释放当前地址
            free(curaddr);
            return (-1);
        }
    } else
        # 如果广播地址为空，则当前地址的广播地址为NULL
        curaddr->broadaddr = NULL;

    # 如果目的地址不为空且大小不为0
    if (dstaddr != NULL && dstaddr_size != 0) {
        # 为当前地址的目的地址分配内存并复制传入的目的地址
        curaddr->dstaddr = (struct sockaddr *)dup_sockaddr(dstaddr, dstaddr_size);
        # 如果分配内存失败
        if (curaddr->dstaddr == NULL) {
            # 格式化错误消息并返回错误码-1
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            # 释放当前地址的广播地址
            if (curaddr->broadaddr != NULL)
                free(curaddr->broadaddr);
            # 释放当前地址的网络掩码
            if (curaddr->netmask != NULL)
                free(curaddr->netmask);
            # 释放当前地址的地址
            if (curaddr->addr != NULL)
                free(curaddr->addr);
            # 释放当前地址
            free(curaddr);
            return (-1);
        }
    } else
        # 如果目的地址为空，则当前地址的目的地址为NULL
        curaddr->dstaddr = NULL;

    '''
     * 查找地址列表的末尾。
     '''
    # 从当前设备的地址开始遍历，直到找到最后一个地址
    for (prevaddr = curdev->addresses; prevaddr != NULL; prevaddr = nextaddr) {
        nextaddr = prevaddr->next;
        # 如果下一个地址为空，则说明当前地址是列表的最后一个地址
        if (nextaddr == NULL) {
            '''
             * 这是列表的末尾。
             '''
            # 跳出循环
            break;
        }
    }

    # 如果prevaddr为空
    if (prevaddr == NULL) {
        '''
         * 列表为空；这是第一个成员。
         '''
        # 将当前地址设置为设备的地址
        curdev->addresses = curaddr;
    } else {
        '''
         * “prevaddr”是列表的最后一个成员；将此成员附加到它上面。
         '''
        # 将当前地址附加到列表的最后一个地址后面
        prevaddr->next = curaddr;
    }

    # 返回0表示成功
    return (0);
/*
 * 在指定的设备列表中查找给定设备。
 *
 * 如果找到了，返回 0 并将 *curdev_ret 设置为指向该设备的指针。
 *
 * 如果没有找到，尝试添加一个具有指定标志和描述的条目，如果成功则返回 0，否则返回 -1 并将 errbuf 设置为错误消息。
 */
pcap_if_t *
find_or_add_dev(pcap_if_list_t *devlistp, const char *name, bpf_u_int32 flags,
    get_if_flags_func get_flags_func, const char *description, char *errbuf)
{
    pcap_if_t *curdev;

    /*
     * 列表中已经有这个设备的条目了吗？
     */
    curdev = find_dev(devlistp, name);
    if (curdev != NULL) {
        /*
         * 是的，返回它。
         */
        return (curdev);
    }

    /*
     * 没有找到。
     */

    /*
     * 尝试获取设备的附加标志。
     */
    if ((*get_flags_func)(name, &flags, errbuf) == -1) {
        /*
         * 失败。
         */
        return (NULL);
    }

    /*
     * 现在，尝试将其添加到设备列表中。
     */
    return (add_dev(devlistp, name, flags, description, errbuf));
}

/*
 * 在指定的设备列表中查找给定设备，并返回找到的条目，如果找不到则返回 NULL。
 */
pcap_if_t *
find_dev(pcap_if_list_t *devlistp, const char *name)
{
    pcap_if_t *curdev;

    /*
     * 列表中是否有这个设备的条目？
     */
    for (curdev = devlistp->beginning; curdev != NULL;
        curdev = curdev->next) {
        if (strcmp(name, curdev->name) == 0) {
            /*
             * 找到了，是的，有。不需要添加。将找到的条目提供给调用者。
             */
            return (curdev);
        }
    }

    /*
     * 没有。
     */
    return (NULL);
}
# 尝试为设备添加一个条目，使用指定的标志和描述，如果成功则返回0并返回新条目的指针，否则返回NULL并将errbuf设置为错误消息。

# 如果没有给定描述，尝试获取一个描述。
pcap_if_t *
add_dev(pcap_if_list_t *devlistp, const char *name, bpf_u_int32 flags,
    const char *description, char *errbuf)
{
    pcap_if_t *curdev, *prevdev, *nextdev;
    u_int this_figure_of_merit, nextdev_figure_of_merit;

    # 为当前设备分配内存
    curdev = malloc(sizeof(pcap_if_t));
    if (curdev == NULL) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return (NULL);
    }

    # 填充条目
    curdev->next = NULL;
    curdev->name = strdup(name);
    if (curdev->name == NULL) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        free(curdev);
        return (NULL);
    }
    if (description == NULL) {
        # 如果没有给定描述，将描述设置为NULL
        curdev->description = NULL;
    } else {
        # 如果给定了描述，复制描述
        curdev->description = strdup(description);
        if (curdev->description == NULL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            free(curdev->name);
            free(curdev);
            return (NULL);
        }
    }
    # 地址列表初始为空
    curdev->addresses = NULL;
    curdev->flags = flags;

    # 将设备添加到列表的适当位置
    # 首先，获取此接口的“merit值”
    this_figure_of_merit = get_figure_of_merit(curdev);
    /*
     * 现在查找最后一个具有小于或等于新接口的优点的接口。
     *
     * 我们从 "prevdev" 为 NULL 开始，表示我们在列表中的第一个元素之前。
     */
    prevdev = NULL;
    for (;;) {
        /*
         * 获取此接口之后的接口。
         */
        if (prevdev == NULL) {
            /*
             * 下一个元素是第一个元素。
             */
            nextdev = devlistp->beginning;
        } else
            nextdev = prevdev->next;

        /*
         * 我们是否在列表末尾？
         */
        if (nextdev == NULL) {
            /*
             * 是的 - 我们必须将新条目放在 "prevdev" 之后。
             */
            break;
        }

        /*
         * 新接口的优点是否小于下一个接口的优点，
         * 这意味着新接口比下一个接口更好？
         */
        nextdev_figure_of_merit = get_figure_of_merit(nextdev);
        if (this_figure_of_merit < nextdev_figure_of_merit) {
            /*
             * 是的 - 我们应该将新条目放在 "nextdev" 之前，即在 "prevdev" 之后。
             */
            break;
        }

        prevdev = nextdev;
    }

    /*
     * 在 "nextdev" 之前插入。
     */
    curdev->next = nextdev;

    /*
     * 在 "prevdev" 之后插入 - 除非 "prevdev" 为 null，
     * 在这种情况下，这是第一个接口。
     */
    if (prevdev == NULL) {
        /*
         * 这是第一个接口。使其成为设备列表中的第一个元素。
         */
        devlistp->beginning = curdev;
    } else
        prevdev->next = curdev;
    return (curdev);
/*
 * 释放所有接口的内存
 */
void
pcap_freealldevs(pcap_if_t *alldevs)
{
    pcap_if_t *curdev, *nextdev;
    pcap_addr_t *curaddr, *nextaddr;

    // 遍历所有接口
    for (curdev = alldevs; curdev != NULL; curdev = nextdev) {
        nextdev = curdev->next;

        /*
         * 释放所有地址
         */
        for (curaddr = curdev->addresses; curaddr != NULL; curaddr = nextaddr) {
            nextaddr = curaddr->next;
            if (curaddr->addr)
                free(curaddr->addr);
            if (curaddr->netmask)
                free(curaddr->netmask);
            if (curaddr->broadaddr)
                free(curaddr->broadaddr);
            if (curaddr->dstaddr)
                free(curaddr->dstaddr);
            free(curaddr);
        }

        /*
         * 释放名称字符串
         */
        free(curdev->name);

        /*
         * 释放描述字符串（如果有的话）
         */
        if (curdev->description != NULL)
            free(curdev->description);

        /*
         * 释放接口
         */
        free(curdev);
    }
}

/*
 * pcap-npf.c 有自己的 pcap_lookupdev()，出于兼容性原因，它实际上返回所有接口的名称，并在它们之间使用 NUL 分隔符；一些调用者可能依赖于这一点。
 *
 * MS-DOS 有自己的 pcap_lookupdev()，但可能只作为一种优化有用。
 *
 * 在所有其他情况下，我们只需使用 pcap_findalldevs() 来获取设备列表，并从该列表中选择。
 */
#if !defined(HAVE_PACKET32) && !defined(MSDOS)
/*
 * 返回连接到系统的网络接口的名称，如果找不到则返回 NULL。接口必须配置为启用；首选最低单元编号；忽略回环。
 */
char *
pcap_lookupdev(char *errbuf)
{
    pcap_if_t *alldevs;
#ifdef _WIN32
  /*
   * Windows - use the same size as the old WinPcap 3.1 code.
   * XXX - this is probably bigger than it needs to be.
   */
  #define IF_NAMESIZE 8192
#else
  /*
   * UN*X - use the system's interface name size.
   * XXX - that might not be large enough for capture devices
   * that aren't regular network interfaces.
   */
  /* for old BSD systems, including bsdi3 */
  #ifndef IF_NAMESIZE
  #define IF_NAMESIZE IFNAMSIZ
  #endif
#endif
    static char device[IF_NAMESIZE + 1];
    char *ret;

    /*
     * We disable this in "new API" mode, because 1) in WinPcap/Npcap,
     * it may return UTF-16 strings, for backwards-compatibility
     * reasons, and we're also disabling the hack to make that work,
     * for not-going-past-the-end-of-a-string reasons, and 2) we
     * want its behavior to be consistent.
     *
     * In addition, it's not thread-safe, so we've marked it as
     * deprecated.
     */
    if (pcap_new_api) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "pcap_lookupdev() is deprecated and is not supported in programs calling pcap_init()");
        return (NULL);
    }

    if (pcap_findalldevs(&alldevs, errbuf) == -1)
        return (NULL);

    if (alldevs == NULL || (alldevs->flags & PCAP_IF_LOOPBACK)) {
        /*
         * There are no devices on the list, or the first device
         * on the list is a loopback device, which means there
         * are no non-loopback devices on the list.  This means
         * we can't return any device.
         *
         * XXX - why not return a loopback device?  If we can't
         * capture on it, it won't be on the list, and if it's
         * on the list, there aren't any non-loopback devices,
         * so why not just supply it as the default device?
         */
        (void)pcap_strlcpy(errbuf, "no suitable device found",
            PCAP_ERRBUF_SIZE);
        ret = NULL;
    } else {
        /*
         * Return the name of the first device on the list.
         */
        (void)pcap_strlcpy(device, alldevs->name, sizeof(device));
        ret = device;
    }

    pcap_freealldevs(alldevs);
    return (ret);
}
#endif /* !defined(HAVE_PACKET32) && !defined(MSDOS) */

#if !defined(_WIN32) && !defined(MSDOS)
/*
 * 如果没有定义 HAVE_PACKET32 和 MSDOS
 */
int
pcap_lookupnet(const char *device, bpf_u_int32 *netp, bpf_u_int32 *maskp,
    char *errbuf)
{
    register int fd;
    register struct sockaddr_in *sin4;
    struct ifreq ifr;

    /*
     * 我们不仅仅获取设备的整个列表，搜索特定设备，并使用其第一个 IPv4 地址，因为这样做太费力了，只为了获取一个设备的网络掩码。
     *
     * 如果我们有一个获取给定设备属性的 API，我们可以使用它。
     */
    /*
     * 如果我们有一个获取给定设备属性的 API，我们可以使用它。
     */
    /*
     * 伪设备 "any" 监听所有接口，因此具有网络地址和掩码 "0.0.0.0"，因此捕获所有流量。使用 NULL 作为接口与 "any" 相同。
     */
    if (!device || strcmp(device, "any") == 0
#ifdef HAVE_DAG_API
        || strstr(device, "dag") != NULL
#endif
#ifdef HAVE_SEPTEL_API
        || strstr(device, "septel") != NULL
#endif
#ifdef PCAP_SUPPORT_BT
        || strstr(device, "bluetooth") != NULL
#endif
#ifdef PCAP_SUPPORT_LINUX_USBMON
        || strstr(device, "usbmon") != NULL
#endif
#ifdef HAVE_SNF_API
        || strstr(device, "snf") != NULL
#endif
#ifdef PCAP_SUPPORT_NETMAP
        || strncmp(device, "netmap:", 7) == 0
        || strncmp(device, "vale", 4) == 0
#endif
#ifdef PCAP_SUPPORT_DPDK
        || strncmp(device, "dpdk:", 5) == 0
#endif
        ) {
        *netp = *maskp = 0;
        return 0;
    }

    fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "socket");
        return (-1);
    }
    memset(&ifr, 0, sizeof(ifr));
#ifdef linux
    /* XXX Work around Linux kernel bug */
    ifr.ifr_addr.sa_family = AF_INET;
#endif
    (void)pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));
    # 如果ioctl函数返回值小于0，表示出现错误
    if (ioctl(fd, SIOCGIFADDR, (char *)&ifr) < 0) {
        # 如果错误码是EADDRNOTAVAIL，表示地址不可用
        if (errno == EADDRNOTAVAIL) {
            # 格式化错误信息，表示没有分配IPv4地址
            (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "%s: no IPv4 address assigned", device);
        } else {
            # 格式化错误信息，表示SIOCGIFADDR出错
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "SIOCGIFADDR: %s", device);
        }
        # 关闭文件描述符
        (void)close(fd);
        # 返回-1表示出错
        return (-1);
    }
    # 将ifr.ifr_addr强制转换为struct sockaddr_in类型
    sin4 = (struct sockaddr_in *)&ifr.ifr_addr;
    # 将sin4->sin_addr.s_addr的值赋给netp
    *netp = sin4->sin_addr.s_addr;
    # 将ifr的内容清零
    memset(&ifr, 0, sizeof(ifr));
#ifdef linux
    /* 如果是在 Linux 系统下，需要解决 Linux 内核的一个 bug */
    ifr.ifr_addr.sa_family = AF_INET;
#endif
    // 将设备名称复制到 ifr.ifr_name 中
    (void)pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));
    // 使用 ioctl 获取设备的网络掩码
    if (ioctl(fd, SIOCGIFNETMASK, (char *)&ifr) < 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGIFNETMASK: %s", device);
        // 关闭文件描述符
        (void)close(fd);
        // 返回错误
        return (-1);
    }
    // 关闭文件描述符
    (void)close(fd);
    // 将网络掩码赋值给 maskp
    *maskp = sin4->sin_addr.s_addr;
    // 如果网络掩码为 0，则根据网络地址的类别设置默认的网络掩码
    if (*maskp == 0) {
        if (IN_CLASSA(*netp))
            *maskp = IN_CLASSA_NET;
        else if (IN_CLASSB(*netp))
            *maskp = IN_CLASSB_NET;
        else if (IN_CLASSC(*netp))
            *maskp = IN_CLASSC_NET;
        else {
            // 格式化错误消息
            (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "inet class for 0x%x unknown", *netp);
            // 返回错误
            return (-1);
        }
    }
    // 对网络地址进行按位与操作，得到网络地址的子网地址
    *netp &= *maskp;
    // 返回成功
    return (0);
}
#endif /* !defined(_WIN32) && !defined(MSDOS) */

#ifdef ENABLE_REMOTE
#include "pcap-rpcap.h"

/*
 * 从字符串中提取子字符串
 */
static char *
get_substring(const char *p, size_t len, char *ebuf)
{
    char *token;

    // 分配内存空间
    token = malloc(len + 1);
    if (token == NULL) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        // 返回空指针
        return (NULL);
    }
    // 复制子字符串
    memcpy(token, p, len);
    token[len] = '\0';
    // 返回子字符串
    return (token);
}
# 解析可能是 URL 的捕获源
# 如果源不是 URL，则将 *schemep、*userinfop、*hostp 和 *portp 设置为 NULL，将 *pathp 设置为指向源的指针，并返回 0
# 如果源是 URL，并且 URL 指向本地设备（rpcap 的特殊情况），则将 *schemep、*userinfop、*hostp 和 *portp 设置为 NULL，将 *pathp 设置为指向设备名称的指针，并返回 0
# 如果源是 URL，并且不是指向本地设备的特殊情况，并且解析成功：
#    *schemep 被设置为指向包含方案的分配字符串的指针
#    如果 URL 中存在用户信息，则将 *userinfop 设置为指向包含用户信息的分配字符串的指针，否则设置为 NULL
#    如果 URL 中存在主机信息，则将 *hostp 设置为指向包含主机信息的分配字符串的指针，否则设置为 NULL
#    如果 URL 中存在端口号，则将 *portp 设置为指向包含端口号的分配字符串的指针，否则设置为 NULL
#    *pathp 被设置为指向包含路径的分配字符串的指针
#    并返回 0
# 如果解析失败，则将 ebuf 设置为错误字符串，并返回 -1
static int
pcap_parse_source(const char *source, char **schemep, char **userinfop,
    char **hostp, char **portp, char **pathp, char *ebuf)
{
    char *colonp;
    size_t scheme_len;
    char *scheme;
    const char *endp;
    size_t authority_len;
    char *authority;
    char *parsep, *atsignp, *bracketp;
    char *userinfo, *host, *port, *path;

    # 开始时什么都不返回
    *schemep = NULL;
    *userinfop = NULL;
    *hostp = NULL;
    *portp = NULL;
    *pathp = NULL;
    /*
     * RFC 3986 规定：
     *
     *   URI         = scheme ":" hier-part [ "?" query ] [ "#" fragment ]
     *
     *   hier-part   = "//" authority path-abempty
     *               / path-absolute
     *               / path-rootless
     *               / path-empty
     *
     *   authority   = [ userinfo "@" ] host [ ":" port ]
     *
     *   userinfo    = *( unreserved / pct-encoded / sub-delims / ":" )
     *
     * 步骤 1：查找 scheme 结尾的冒号。
     * 源字符串中的冒号 *不足以* 表示这是一个 URL，因为一些平台上的接口名称可能包含冒号（例如，我认为一些 Solaris 接口可能会包含冒号）。
     */
    colonp = strchr(source, ':');
    if (colonp == NULL) {
        /*
         * 源字符串是要打开的设备。
         * 为 scheme、用户信息、主机和端口返回一个空指针，并将设备作为路径返回。
         */
        *pathp = strdup(source);
        if (*pathp == NULL) {
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            return (-1);
        }
        return (0);
    }

    /*
     * 所有 scheme 必须在它们后面有 "//"，即我们只支持 hier-part   = "//" authority path-abempty，而不支持
     * hier-part   = path-absolute
     * hier-part   = path-rootless
     * hier-part   = path-empty
     *
     * 我们需要这样做以区分包含冒号的本地设备名称和 URI。
     */
    # 如果冒号后面的两个字符不是“//”，则执行以下操作
    if (strncmp(colonp + 1, "//", 2) != 0) {
        /*
         * 源是要打开的设备。
         * 为方案、用户信息、主机和端口返回一个空指针，并将设备作为路径返回。
         */
        *pathp = strdup(source);
        # 如果分配内存失败，则返回错误信息
        if (*pathp == NULL) {
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            return (-1);
        }
        return (0);
    }

    /*
     * XXX - 检查所谓的方案是否可能是一个方案？
     */

    /*
     * 好的，这看起来像一个 URL。
     * 获取方案。
     */
    scheme_len = colonp - source;
    scheme = malloc(scheme_len + 1);
    # 如果分配内存失败，则返回错误信息
    if (scheme == NULL) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return (-1);
    }
    memcpy(scheme, source, scheme_len);
    scheme[scheme_len] = '\0';

    /*
     * 特殊处理 file: - 将 file:// 后面的所有内容作为路径名。
     */
    if (pcap_strcasecmp(scheme, "file") == 0) {
        *pathp = strdup(colonp + 3);
        # 如果分配内存失败，则返回错误信息
        if (*pathp == NULL) {
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            free(scheme);
            return (-1);
        }
        *schemep = scheme;
        return (0);
    }

    /*
     * WinPcap 文档表示可以使用 "rpcap://{device}" 指定本地接口；我们在这里特殊处理。
     * 如果方案是 "rpcap"，并且“//”后面没有斜杠，我们只返回设备。
     *
     * XXX - %-escaping?
     */
    # 如果 scheme 是 "rpcap" 或 "rpcaps"，并且冒号后的第三个字符之后没有斜杠
    if ((pcap_strcasecmp(scheme, "rpcap") == 0 ||
        pcap_strcasecmp(scheme, "rpcaps") == 0) &&
        strchr(colonp + 3, '/') == NULL) {
        '''
        * 本地设备。
        *
        * 为 scheme、用户信息、主机和端口返回一个空指针，并将设备作为路径返回。
        '''
        # 释放 scheme 的内存
        free(scheme)
        # 复制冒号后的第三个字符之后的内容作为路径
        *pathp = strdup(colonp + 3)
        # 如果路径为空，返回错误
        if (*pathp == NULL) {
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "malloc")
            return (-1)
        }
        return (0)
    }

    '''
     * 现在开始解析权限部分。
     * 获取以 / 结束的令牌，或者在字符串末尾结束的令牌。
     '''
    # 计算权限部分的长度
    authority_len = strcspn(colonp + 3, "/")
    # 获取权限部分的子字符串
    authority = get_substring(colonp + 3, authority_len, ebuf)
    # 如果权限部分为空，返回错误
    if (authority == NULL) {
        free(scheme)
        return (-1)
    }
    # 将 endp 指向权限部分的末尾
    endp = colonp + 3 + authority_len

    '''
     * 现在将权限字段划分为其组成部分。
     '''
    # 解析指针指向权限字段
    parsep = authority

    '''
     * 是否有用户信息字段？
     '''
    # 查找 @ 符号
    atsignp = strchr(parsep, '@')
    # 如果存在用户信息字段
    if (atsignp != NULL) {
        '''
         * 有。
         '''
        # 计算用户信息字段的长度
        size_t userinfo_len
        userinfo_len = atsignp - parsep
        # 获取用户信息字段的子字符串
        userinfo = get_substring(parsep, userinfo_len, ebuf)
        # 如果用户信息字段为空，返回错误
        if (userinfo == NULL) {
            free(authority)
            free(scheme)
            return (-1)
        }
        # 将解析指针指向 @ 符号后的位置
        parsep = atsignp + 1
    } else {
        '''
         * 没有。
         '''
        userinfo = NULL
    }

    '''
     * 是否有主机字段？
     '''
    # 如果解析指针指向字符串末尾
    if (*parsep == '\0') {
        '''
         * 没有；没有主机字段或端口字段。
         '''
        host = NULL
        port = NULL
    }
    # 释放权限字段的内存

    '''
     * 其余部分是路径。去掉前导的 /。
     */
    # 如果指针指向的字符是字符串结束符，则将 path 指向一个空字符串的副本
    if (*endp == '\0')
        path = strdup("");
    # 否则，将 path 指向 endp+1 处的字符串的副本
    else
        path = strdup(endp + 1);
    # 如果 path 为 NULL，则表示内存分配失败，需要进行错误处理
    if (path == NULL) {
        # 格式化错误消息，将错误信息存储到 ebuf 中
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        # 释放内存
        free(port);
        free(host);
        free(userinfo);
        free(scheme);
        # 返回错误标志
        return (-1);
    }
    # 将指针指向的值赋给相应的指针变量
    *schemep = scheme;
    *userinfop = userinfo;
    *hostp = host;
    *portp = port;
    *pathp = path;
    # 返回成功标志
    return (0);
}
// 定义一个返回整数的函数，用于创建捕获源字符串
pcap_createsrcstr_ex(char *source, int type, const char *host, const char *port,
    const char *name, unsigned char uses_ssl, char *errbuf)
{
    // 根据类型进行不同的处理
    switch (type) {

    case PCAP_SRC_FILE:
        // 将 PCAP_SRC_FILE_STRING 复制到 source 中
        pcap_strlcpy(source, PCAP_SRC_FILE_STRING, PCAP_BUF_SIZE);
        // 如果 name 不为空，则将其追加到 source 中
        if (name != NULL && *name != '\0') {
            pcap_strlcat(source, name, PCAP_BUF_SIZE);
            return (0);
        } else {
            // 如果 name 为空，则返回错误信息
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "The file name cannot be NULL.");
            return (-1);
        }

    case PCAP_SRC_IFREMOTE:
        // 根据 uses_ssl 的值选择不同的前缀，并将其复制到 source 中
        pcap_strlcpy(source,
            (uses_ssl ? "rpcaps://" : PCAP_SRC_IF_STRING),
            PCAP_BUF_SIZE);
        // 如果 host 不为空，则根据情况进行处理
        if (host != NULL && *host != '\0') {
            // 如果 host 包含冒号，则将其加入方括号中
            if (strchr(host, ':') != NULL) {
                /*
                 * The host name contains a colon, so it's
                 * probably an IPv6 address, and needs to
                 * be included in square brackets.
                 */
                pcap_strlcat(source, "[", PCAP_BUF_SIZE);
                pcap_strlcat(source, host, PCAP_BUF_SIZE);
                pcap_strlcat(source, "]", PCAP_BUF_SIZE);
            } else
                pcap_strlcat(source, host, PCAP_BUF_SIZE);

            // 如果 port 不为空，则将其追加到 source 中
            if (port != NULL && *port != '\0') {
                pcap_strlcat(source, ":", PCAP_BUF_SIZE);
                pcap_strlcat(source, port, PCAP_BUF_SIZE);
            }

            // 在 source 末尾追加斜杠
            pcap_strlcat(source, "/", PCAP_BUF_SIZE);
        } else {
            // 如果 host 为空，则返回错误信息
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "The host name cannot be NULL.");
            return (-1);
        }

        // 如果 name 不为空，则将其追加到 source 中
        if (name != NULL && *name != '\0')
            pcap_strlcat(source, name, PCAP_BUF_SIZE);

        return (0);

    case PCAP_SRC_IFLOCAL:
        // 将 PCAP_SRC_IF_STRING 复制到 source 中
        pcap_strlcpy(source, PCAP_SRC_IF_STRING, PCAP_BUF_SIZE);

        // 如果 name 不为空，则将其追加到 source 中
        if (name != NULL && *name != '\0')
            pcap_strlcat(source, name, PCAP_BUF_SIZE);

        return (0);
    # 如果没有匹配到任何情况，则执行默认操作
    default:
        # 使用 snprintf 函数将错误信息格式化并存储到 errbuf 中
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "The interface type is not valid.");
        # 返回 -1，表示出现错误
        return (-1);
    }
}

int
pcap_createsrcstr(char *source, int type, const char *host, const char *port,
    const char *name, char *errbuf)
{
    return (pcap_createsrcstr_ex(source, type, host, port, name, 0, errbuf));
}

int
pcap_parsesrcstr_ex(const char *source, int *type, char *host, char *port,
    char *name, unsigned char *uses_ssl, char *errbuf)
{
    char *scheme, *tmpuserinfo, *tmphost, *tmpport, *tmppath;

    /* 初始化一些变量 */
    if (host)
        *host = '\0';
    if (port)
        *port = '\0';
    if (name)
        *name = '\0';
    if (uses_ssl)
        *uses_ssl = 0;

    /* 解析源字符串 */
    if (pcap_parse_source(source, &scheme, &tmpuserinfo, &tmphost,
        &tmpport, &tmppath, errbuf) == -1) {
        /*
         * 失败。
         */
        return (-1);
    }

    if (scheme == NULL) {
        /*
         * 本地设备。
         */
        if (name && tmppath)
            pcap_strlcpy(name, tmppath, PCAP_BUF_SIZE);
        if (type)
            *type = PCAP_SRC_IFLOCAL;
        free(tmppath);
        free(tmpport);
        free(tmphost);
        free(tmpuserinfo);
        return (0);
    }

    int is_rpcap = 0;
    if (strcmp(scheme, "rpcaps") == 0) {
        is_rpcap = 1;
        if (uses_ssl) *uses_ssl = 1;
    } else if (strcmp(scheme, "rpcap") == 0) {
        is_rpcap = 1;
    }
    if (is_rpcap) {
        /*
         * 如果是 rpcap 协议
         * rpcap[s]://
         *
         * pcap_parse_source() 已经处理了 rpcap[s]://device 的情况
         */
        if (host && tmphost) {
            if (tmpuserinfo)
                snprintf(host, PCAP_BUF_SIZE, "%s@%s",
                    tmpuserinfo, tmphost);
            else
                pcap_strlcpy(host, tmphost, PCAP_BUF_SIZE);
        }
        if (port && tmpport)
            pcap_strlcpy(port, tmpport, PCAP_BUF_SIZE);
        if (name && tmppath)
            pcap_strlcpy(name, tmppath, PCAP_BUF_SIZE);
        if (type)
            *type = PCAP_SRC_IFREMOTE;
        free(tmppath);
        free(tmpport);
        free(tmphost);
        free(tmpuserinfo);
        free(scheme);
        return (0);
    }

    if (strcmp(scheme, "file") == 0) {
        /*
         * 如果是 file 协议
         * file://
         */
        if (name && tmppath)
            pcap_strlcpy(name, tmppath, PCAP_BUF_SIZE);
        if (type)
            *type = PCAP_SRC_FILE;
        free(tmppath);
        free(tmpport);
        free(tmphost);
        free(tmpuserinfo);
        free(scheme);
        return (0);
    }

    /*
     * 如果既不是 rpcap 也不是 file 协议；将整个字符串视为本地设备
     */
    if (name)
        pcap_strlcpy(name, source, PCAP_BUF_SIZE);
    if (type)
        *type = PCAP_SRC_IFLOCAL;
    free(tmppath);
    free(tmpport);
    free(tmphost);
    free(tmpuserinfo);
    free(scheme);
    return (0);
}

int
pcap_parsesrcstr(const char *source, int *type, char *host, char *port,
    char *name, char *errbuf)
{
    return (pcap_parsesrcstr_ex(source, type, host, port, name, NULL, errbuf));
}
#endif

pcap_t *
pcap_create(const char *device, char *errbuf)
{
    size_t i;
    int is_theirs;
    pcap_t *p;
    char *device_str;

    /*
     * A null device name is equivalent to the "any" device -
     * which might not be supported on this platform, but
     * this means that you'll get a "not supported" error
     * rather than, say, a crash when we try to dereference
     * the null pointer.
     */
    if (device == NULL)
        device_str = strdup("any");
    else {
#endif
            device_str = strdup(device);
    }
    if (device_str == NULL) {
        // 如果分配内存失败，则格式化错误消息并返回空指针
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return (NULL);
    }

    /*
     * 尝试每种非本地网络接口捕获源类型，直到找到适用于此设备的类型或用尽类型。
     */
    for (i = 0; capture_source_types[i].create_op != NULL; i++) {
        is_theirs = 0;
        p = capture_source_types[i].create_op(device_str, errbuf,
            &is_theirs);
        if (is_theirs) {
            /*
             * 设备名称指的是该类型的设备；要么成功，此时 p 指向稍后激活的设备的 pcap_t，要么失败，此时 p 为 null，我们应该返回 null 以报告创建失败。
             */
            if (p == NULL) {
                /*
                 * 我们假设调用者填写了 errbuf。
                 */
                free(device_str);
                return (NULL);
            }
            p->opt.device = device_str;
            return (p);
        }
    }

    /*
     * 好吧，将其视为常规网络接口。
     */
    # 创建一个 pcap 对象，使用指定的设备字符串和错误缓冲区
    p = pcap_create_interface(device_str, errbuf);
    # 如果创建失败，假设调用者已经填写了错误缓冲区，释放设备字符串并返回空指针
    if (p == NULL) {
        free(device_str);
        return (NULL);
    }
    # 设置 pcap 对象的设备属性为设备字符串
    p->opt.device = device_str;
    # 返回创建的 pcap 对象
    return (p);
}
/*
 * 在未激活的 pcap_t 上设置非阻塞模式；这会设置一个标志，
 * pcap_activate() 会检查这个标志，并在调用激活例程后设置模式。
 */
static int
pcap_setnonblock_unactivated(pcap_t *p, int nonblock)
{
    // 设置非阻塞模式
    p->opt.nonblock = nonblock;
    // 返回 0 表示成功
    return (0);
}

static void
initialize_ops(pcap_t *p)
{
    /*
     * 设置仅在已激活的 pcap_t 上工作的操作指针，指向一个返回
     * “未初始化”错误的例程。
     */
    p->read_op = pcap_read_not_initialized;
    p->inject_op = pcap_inject_not_initialized;
    p->setfilter_op = pcap_setfilter_not_initialized;
    p->setdirection_op = pcap_setdirection_not_initialized;
    p->set_datalink_op = pcap_set_datalink_not_initialized;
    p->getnonblock_op = pcap_getnonblock_not_initialized;
    p->stats_op = pcap_stats_not_initialized;
#ifdef _WIN32
    p->stats_ex_op = pcap_stats_ex_not_initialized;
    p->setbuff_op = pcap_setbuff_not_initialized;
    p->setmode_op = pcap_setmode_not_initialized;
    p->setmintocopy_op = pcap_setmintocopy_not_initialized;
    p->getevent_op = pcap_getevent_not_initialized;
    p->oid_get_request_op = pcap_oid_get_request_not_initialized;
    p->oid_set_request_op = pcap_oid_set_request_not_initialized;
    p->sendqueue_transmit_op = pcap_sendqueue_transmit_not_initialized;
    p->setuserbuffer_op = pcap_setuserbuffer_not_initialized;
    p->live_dump_op = pcap_live_dump_not_initialized;
    p->live_dump_ended_op = pcap_live_dump_ended_not_initialized;
    p->get_airpcap_handle_op = pcap_get_airpcap_handle_not_initialized;
#endif

    /*
     * 默认的清理操作 - 实现可以覆盖这个，但应在进行自己的额外清理后调用
     * pcap_cleanup_live_common()。
     */
    p->cleanup_op = pcap_cleanup_live_common;

    /*
     * 在大多数情况下，标准的一次性回调可以用于 pcap_next()/pcap_next_ex()。
     */
}
    # 设置回调函数为 pcap_oneshot
    p->oneshot_callback = pcap_oneshot;

    '''
     * 默认的中断循环操作 - 实现可以覆盖这个操作，
     * 但应该在执行自己的逻辑之前调用 pcap_breakloop_common()。
     '''
    # 设置中断循环操作为 pcap_breakloop_common
    p->breakloop_op = pcap_breakloop_common;
}

static pcap_t *
pcap_alloc_pcap_t(char *ebuf, size_t total_size, size_t private_offset)
{
    char *chunk;
    pcap_t *p;

    /*
     * total_size is the size of a structure containing a pcap_t
     * followed by a private structure.
     */
    // 分配总大小为 total_size 字节的内存块
    chunk = calloc(total_size, 1);
    if (chunk == NULL) {
        // 如果内存分配失败，设置错误消息并返回空指针
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        return (NULL);
    }

    /*
     * Get a pointer to the pcap_t at the beginning.
     */
    // 获取指向 pcap_t 结构的指针
    p = (pcap_t *)chunk;

#ifdef _WIN32
    p->handle = INVALID_HANDLE_VALUE;    /* not opened yet */
#else /* _WIN32 */
    p->fd = -1;    /* not opened yet */
#ifndef MSDOS
    p->selectable_fd = -1;
    p->required_select_timeout = NULL;
#endif /* MSDOS */
#endif /* _WIN32 */

    /*
     * private_offset is the offset, in bytes, of the private data from the beginning of the structure.
     *
     * Set the pointer to the private data; that's private_offset bytes past the pcap_t.
     */
    // 设置指向私有数据的指针，即 pcap_t 开始后 private_offset 字节处
    p->priv = (void *)(chunk + private_offset);

    return (p);
}

pcap_t *
pcap_create_common(char *ebuf, size_t total_size, size_t private_offset)
{
    pcap_t *p;

    // 调用 pcap_alloc_pcap_t() 分配内存
    p = pcap_alloc_pcap_t(ebuf, total_size, private_offset);
    if (p == NULL)
        return (NULL);

    /*
     * Default to "can't set rfmon mode"; if it's supported by
     * a platform, the create routine that called us can set
     * the op to its routine to check whether a particular
     * device supports it.
     */
    // 默认设置为“无法设置 rfmon 模式”
    p->can_set_rfmon_op = pcap_cant_set_rfmon;

    /*
     * If pcap_setnonblock() is called on a not-yet-activated
     * pcap_t, default to setting a flag and turning
     * on non-blocking mode when activated.
     */
    // 如果在尚未激活的 pcap_t 上调用 pcap_setnonblock()，默认设置标志并在激活时打开非阻塞模式
    p->setnonblock_op = pcap_setnonblock_unactivated;

    // 初始化操作
    initialize_ops(p);

    /* put in some defaults*/
    p->snapshot = 0;        /* max packet size unspecified */
    p->opt.timeout = 0;        /* no timeout specified */
}
    # 设置缓冲区大小为0，使用平台的默认值
    p->opt.buffer_size = 0;
    # 关闭混杂模式
    p->opt.promisc = 0;
    # 关闭RFMON（无线网卡的监控模式）
    p->opt.rfmon = 0;
    # 关闭立即模式
    p->opt.immediate = 0;
    # 设置时间戳类型为默认值
    p->opt.tstamp_type = -1;
    # 设置时间戳精度为微秒级
    p->opt.tstamp_precision = PCAP_TSTAMP_PRECISION_MICRO;
    """
     * 平台相关选项
     */
#ifdef __linux__
    // 如果是在 Linux 系统下编译，设置协议为 0
    p->opt.protocol = 0;
#endif
#ifdef _WIN32
    // 如果是在 Windows 系统下编译，设置本地不捕获为 0
    p->opt.nocapture_local = 0;
#endif

    /*
     * Start out with no BPF code generation flags set.
     */
    // 开始时不设置任何 BPF 代码生成标志
    p->bpf_codegen_flags = 0;

    // 返回 pcap_t 结构体指针
    return (p);
}

int
pcap_check_activated(pcap_t *p)
{
    // 如果已经激活，返回错误
    if (p->activated) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "can't perform "
            " operation on activated capture");
        return (-1);
    }
    return (0);
}

int
pcap_set_snaplen(pcap_t *p, int snaplen)
{
    // 检查是否已激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 设置快照长度
    p->snapshot = snaplen;
    return (0);
}

int
pcap_set_promisc(pcap_t *p, int promisc)
{
    // 检查是否已激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 设置混杂模式
    p->opt.promisc = promisc;
    return (0);
}

int
pcap_set_rfmon(pcap_t *p, int rfmon)
{
    // 检查是否已激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 设置 RF 监控模式
    p->opt.rfmon = rfmon;
    return (0);
}

int
pcap_set_timeout(pcap_t *p, int timeout_ms)
{
    // 检查是否已激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 设置超时时间
    p->opt.timeout = timeout_ms;
    return (0);
}

int
pcap_set_tstamp_type(pcap_t *p, int tstamp_type)
{
    int i;

    // 检查是否已激活
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);

    /*
     * The argument should have been u_int, but that's too late
     * to change now - it's an API.
     */
    // 参数应该是 u_int，但现在改变已经太晚了 - 这是一个 API
    if (tstamp_type < 0)
        return (PCAP_WARNING_TSTAMP_TYPE_NOTSUP);

    /*
     * If p->tstamp_type_count is 0, we only support PCAP_TSTAMP_HOST;
     * the default time stamp type is PCAP_TSTAMP_HOST.
     */
    // 如果 p->tstamp_type_count 为 0，我们只支持 PCAP_TSTAMP_HOST；默认时间戳类型是 PCAP_TSTAMP_HOST
    if (p->tstamp_type_count == 0) {
        if (tstamp_type == PCAP_TSTAMP_HOST) {
            p->opt.tstamp_type = tstamp_type;
            return (0);
        }
    } else {
        /*
         * 检查我们是否声称支持这种类型的时间戳。
         */
        for (i = 0; i < p->tstamp_type_count; i++) {
            if (p->tstamp_type_list[i] == (u_int)tstamp_type) {
                /*
                 * 是的。
                 */
                p->opt.tstamp_type = tstamp_type;
                return (0);
            }
        }
    }

    /*
     * 我们不支持这种类型的时间戳。
     */
    return (PCAP_WARNING_TSTAMP_TYPE_NOTSUP);
}

int
pcap_set_immediate_mode(pcap_t *p, int immediate)
{
    // 检查 pcap_t 是否已经被激活，如果是则返回错误码
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 设置 pcap_t 的立即模式
    p->opt.immediate = immediate;
    return (0);
}

int
pcap_set_buffer_size(pcap_t *p, int buffer_size)
{
    // 检查 pcap_t 是否已经被激活，如果是则返回错误码
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    // 如果缓冲区大小小于等于0，则静默忽略无效值
    if (buffer_size <= 0) {
        /*
         * Silently ignore invalid values.
         */
        return (0);
    }
    // 设置 pcap_t 的缓冲区大小
    p->opt.buffer_size = buffer_size;
    return (0);
}

int
pcap_set_tstamp_precision(pcap_t *p, int tstamp_precision)
{
    int i;

    // 检查 pcap_t 是否已经被激活，如果是则返回错误码
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);

    /*
     * The argument should have been u_int, but that's too late
     * to change now - it's an API.
     */
    // 如果时间戳精度小于0，则返回不支持的时间戳精度错误码
    if (tstamp_precision < 0)
        return (PCAP_ERROR_TSTAMP_PRECISION_NOTSUP);

    /*
     * If p->tstamp_precision_count is 0, we only support setting
     * the time stamp precision to microsecond precision; every
     * pcap module *MUST* support microsecond precision, even if
     * it does so by converting the native precision to
     * microseconds.
     */
    // 如果时间戳精度计数为0，则只支持将时间戳精度设置为微秒精度
    if (p->tstamp_precision_count == 0) {
        if (tstamp_precision == PCAP_TSTAMP_PRECISION_MICRO) {
            p->opt.tstamp_precision = tstamp_precision;
            return (0);
        }
    } else {
        /*
         * Check whether we claim to support this precision of
         * time stamp.
         */
        // 检查是否支持此时间戳精度
        for (i = 0; i < p->tstamp_precision_count; i++) {
            if (p->tstamp_precision_list[i] == (u_int)tstamp_precision) {
                /*
                 * Yes.
                 */
                p->opt.tstamp_precision = tstamp_precision;
                return (0);
            }
        }
    }

    /*
     * We don't support this time stamp precision.
     */
    // 不支持此时间戳精度
    return (PCAP_ERROR_TSTAMP_PRECISION_NOTSUP);
}

int
pcap_get_tstamp_precision(pcap_t *p)
{
        return (p->opt.tstamp_precision);
}

int
pcap_activate(pcap_t *p)
    # 定义整型变量 status
    int status;

    # 检查是否尝试重新激活已经激活的 pcap_t；例如，捕获调用了 pcap_open_live() 后又调用了 pcap_activate() 的代码
    if (pcap_check_activated(p))
        return (PCAP_ERROR_ACTIVATED);
    # 调用激活操作函数，将结果保存到 status 变量
    status = p->activate_op(p);
    if (status >= 0) {
        # 如果在调用 pcap_activate() 前请求了非阻塞模式，则现在开启非阻塞模式
        if (p->opt.nonblock) {
            status = p->setnonblock_op(p, 1);
            if (status < 0) {
                # 失败，撤销激活操作所做的一切
                p->cleanup_op(p);
                initialize_ops(p);
                return (status);
            }
        }
        # 设置 pcap_t 已激活标志
        p->activated = 1;
    } else {
        if (p->errbuf[0] == '\0') {
            # 激活例程未提供错误消息；为了不专门处理 PCAP_ERROR 以外的错误的程序，返回与状态对应的错误消息
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "%s",
                pcap_statustostr(status));
        }

        # 撤销激活操作所设置的任何操作指针等
        initialize_ops(p);
    }
    return (status);
}
// 打开一个实时捕获会话
pcap_t *
pcap_open_live(const char *device, int snaplen, int promisc, int to_ms, char *errbuf)
{
    pcap_t *p;
    int status;
#ifdef ENABLE_REMOTE
    char host[PCAP_BUF_SIZE + 1];
    char port[PCAP_BUF_SIZE + 1];
    char name[PCAP_BUF_SIZE + 1];
    int srctype;

    /*
     * 如果设备名为空，则等同于使用"any"设备 - 这可能在该平台上不受支持，但这意味着当我们尝试解引用空指针时，你会得到一个"不支持"的错误，而不是崩溃。
     */
    if (device == NULL)
        device = "any";

    /*
     * Retrofit - 我们必须使旧应用程序与远程捕获兼容。
     * 因此，我们从这里调用pcap_open_remote()；这是一个非常肮脏的hack。
     * 显然，我们无法利用所有新功能；例如，我们无法发送身份验证，无法使用UDP数据连接等。
     */
    if (pcap_parsesrcstr(device, &srctype, host, port, name, errbuf))
        return (NULL);

    if (srctype == PCAP_SRC_IFREMOTE) {
        /*
         * 尽管我们已经有主机、端口和接口，但我们更喜欢只传递'device'给pcap_open_rpcap()，这样它就必须再次调用pcap_parsesrcstr()。
         * 这样做不太优化，但更清晰。
         */
        return (pcap_open_rpcap(device, snaplen,
            promisc ? PCAP_OPENFLAG_PROMISCUOUS : 0, to_ms,
            NULL, errbuf));
    }
    if (srctype == PCAP_SRC_FILE) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "unknown URL scheme \"file\"");
        return (NULL);
    }
    # 如果数据源类型为本地接口
    if (srctype == PCAP_SRC_IFLOCAL) {
        /*
         * 如果数据源以rpcap://开头，表示引用本地设备
         * URL中没有主机部分。移除rpcap://，然后
         * 继续执行常规的打开路径。
         */
        if (strncmp(device, PCAP_SRC_IF_STRING, strlen(PCAP_SRC_IF_STRING)) == 0) {
            # 计算去除rpcap://后的设备名长度
            size_t len = strlen(device) - strlen(PCAP_SRC_IF_STRING) + 1;

            # 如果长度大于0，则移除rpcap://
            if (len > 0)
                device += strlen(PCAP_SRC_IF_STRING);
        }
    }
#endif    /* ENABLE_REMOTE */

# 创建一个 pcap 对象，用于捕获数据包
p = pcap_create(device, errbuf);
if (p == NULL)
    return (NULL);
# 设置捕获数据包的快照长度
status = pcap_set_snaplen(p, snaplen);
if (status < 0)
    goto fail;
# 设置是否混杂模式
status = pcap_set_promisc(p, promisc);
if (status < 0)
    goto fail;
# 设置超时时间
status = pcap_set_timeout(p, to_ms);
if (status < 0)
    goto fail;
# 标记使用 pcap_open_live() 打开，以便显示完整的 DLT_ 值列表
p->oldstyle = 1;
# 激活 pcap 对象
status = pcap_activate(p);
if (status < 0)
    goto fail;
# 返回 pcap 对象
return (p);
fail:
if (status == PCAP_ERROR) {
    # 复制错误信息到新的缓冲区
    char trimbuf[PCAP_ERRBUF_SIZE - 5]; /* 2 bytes shorter */
    pcap_strlcpy(trimbuf, p->errbuf, sizeof(trimbuf));
    snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %.*s", device,
        PCAP_ERRBUF_SIZE - 3, trimbuf);
    } else if (status == PCAP_ERROR_NO_SUCH_DEVICE ||
        status == PCAP_ERROR_PERM_DENIED ||
        status == PCAP_ERROR_PROMISC_PERM_DENIED) {
        /*
         * 只有在附加消息不为空时才显示
         */
        if (p->errbuf[0] != '\0') {
            /*
             * 同上
             */
            char trimbuf[PCAP_ERRBUF_SIZE - 8]; /* 比原来短 2 个字节 */

            // 将 p->errbuf 复制到 trimbuf 中
            pcap_strlcpy(trimbuf, p->errbuf, sizeof(trimbuf));
            // 格式化错误消息，包括设备名、状态和部分错误消息
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s (%.*s)",
                device, pcap_statustostr(status),
                PCAP_ERRBUF_SIZE - 6, trimbuf);
        } else {
            // 格式化错误消息，只包括设备名和状态
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s",
                device, pcap_statustostr(status));
        }
    } else {
        // 格式化错误消息，只包括设备名和状态
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s", device,
            pcap_statustostr(status));
    }
    // 关闭 pcap_t 对象
    pcap_close(p);
    // 返回空指针
    return (NULL);
}

pcap_t *
pcap_open_offline_common(char *ebuf, size_t total_size, size_t private_offset)
{
    pcap_t *p;

    // 分配一个 pcap_t 结构体
    p = pcap_alloc_pcap_t(ebuf, total_size, private_offset);
    if (p == NULL)
        return (NULL);

    // 设置时间戳精度为微秒
    p->opt.tstamp_precision = PCAP_TSTAMP_PRECISION_MICRO;

    return (p);
}

int
pcap_dispatch(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 调用 pcap_t 结构体中的 read_op 函数
    return (p->read_op(p, cnt, callback, user));
}

int
pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    register int n;

    for (;;) {
        if (p->rfile != NULL) {
            /*
             * 0 means EOF, so don't loop if we get 0.
             */
            // 如果 rfile 不为空，则调用 pcap_offline_read 函数
            n = pcap_offline_read(p, cnt, callback, user);
        } else {
            /*
             * XXX keep reading until we get something
             * (or an error occurs)
             */
            // 否则，循环调用 read_op 函数，直到返回值不为 0
            do {
                n = p->read_op(p, cnt, callback, user);
            } while (n == 0);
        }
        if (n <= 0)
            return (n);
        if (!PACKET_COUNT_IS_UNLIMITED(cnt)) {
            cnt -= n;
            if (cnt <= 0)
                return (0);
        }
    }
}

/*
 * Force the loop in "pcap_read()" or "pcap_read_offline()" to terminate.
 */
void
pcap_breakloop(pcap_t *p)
{
    // 调用 breakloop_op 函数，强制终止循环
    p->breakloop_op(p);
}

int
pcap_datalink(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    // 返回链路类型
    return (p->linktype);
}

int
pcap_datalink_ext(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    // 返回扩展链路类型
    return (p->linktype_ext);
}

int
pcap_list_datalinks(pcap_t *p, int **dlt_buffer)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    // 返回支持的链路类型列表
    # 如果设备支持的数据链路类型数量为0
    if (p->dlt_count == 0) {
        /*
         * 无法获取数据链路类型列表，这意味着该平台不支持更改接口的数据链路类型。
         * 返回一个只包含设备支持的数据链路类型的列表。
         */
        *dlt_buffer = (int*)malloc(sizeof(**dlt_buffer));
        # 如果内存分配失败
        if (*dlt_buffer == NULL) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "malloc");
            # 返回错误
            return (PCAP_ERROR);
        }
        **dlt_buffer = p->linktype;
        # 返回1，表示只有一个数据链路类型
        return (1);
    } else {
        # 分配内存以存储数据链路类型列表
        *dlt_buffer = (int*)calloc(sizeof(**dlt_buffer), p->dlt_count);
        # 如果内存分配失败
        if (*dlt_buffer == NULL) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
                errno, "malloc");
            # 返回错误
            return (PCAP_ERROR);
        }
        # 将设备支持的数据链路类型列表复制到分配的内存中
        (void)memcpy(*dlt_buffer, p->dlt_list,
            sizeof(**dlt_buffer) * p->dlt_count);
        # 返回设备支持的数据链路类型数量
        return (p->dlt_count);
    }
/*
 * 在Windows中，你可能会有一个使用C运行时库的一个版本构建的库，和一个使用另一个版本构建的应用程序，这意味着库可能使用一个版本的malloc()和free()，而应用程序可能使用另一个版本的malloc()和free()。如果是这样，那么意味着库分配的内容不能被应用程序释放，因此我们需要一个pcap_free_datalinks()例程来释放pcap_list_datalinks()分配的列表，即使它只是一个free()的包装。
 */
void
pcap_free_datalinks(int *dlt_list)
{
    // 释放dlt_list指向的内存
    free(dlt_list);
}

int
pcap_set_datalink(pcap_t *p, int dlt)
{
    int i;
    const char *dlt_name;

    if (dlt < 0)
        // 跳转到不支持的标签
        goto unsupported;

    if (p->dlt_count == 0 || p->set_datalink_op == NULL) {
        /*
         * 我们无法获取DLT列表，或者我们没有"set datalink"操作，这意味着这个平台不支持更改接口的DLT。检查新的DLT是否是这个接口支持的DLT。
         */
        if (p->linktype != dlt)
            // 跳转到不支持的标签
            goto unsupported;

        /*
         * 是的，所以这里我们不需要做任何事情。
         */
        return (0);
    }
    for (i = 0; i < p->dlt_count; i++)
        if (p->dlt_list[i] == (u_int)dlt)
            break;
    if (i >= p->dlt_count)
        // 跳转到不支持的标签
        goto unsupported;
    # 如果数据链路类型数量为2且第一个类型为DLT_EN10MB，第二个类型为DLT_DOCSIS
    if (p->dlt_count == 2 && p->dlt_list[0] == DLT_EN10MB &&
        dlt == DLT_DOCSIS) {
        '''
        这很可能是一个以太网设备，因为它提供的第一个链路层类型是DLT_EN10MB，而唯一的其他类型是DLT_DOCSIS。
        这意味着我们无法告诉驱动程序提供DOCSIS链路层头 - 我们只是假装我们得到的是这个，因为很可能我们正在捕获连接到思科电缆调制解调器终端系统的专用链路上的原始DOCSIS帧。
        '''
        # 设置数据链路类型为dlt
        p->linktype = dlt;
        # 返回0表示成功
        return (0);
    }
    # 如果设置数据链路类型操作失败
    if (p->set_datalink_op(p, dlt) == -1)
        # 返回-1表示失败
        return (-1);
    # 设置数据链路类型为dlt
    p->linktype = dlt;
    # 返回0表示成功
    return (0);
# 获取数据链路类型对应的名称
dlt_name = pcap_datalink_val_to_name(dlt);
# 如果数据链路类型名称不为空
if (dlt_name != NULL) {
    # 格式化错误信息，指明数据链路类型不被设备支持
    (void) snprintf(p->errbuf, sizeof(p->errbuf),
        "%s is not one of the DLTs supported by this device",
        dlt_name);
} else {
    # 格式化错误信息，指明数据链路类型不被设备支持
    (void) snprintf(p->errbuf, sizeof(p->errbuf),
        "DLT %d is not one of the DLTs supported by this device",
        dlt);
}
# 返回错误代码
return (-1);
}

'''
 * 该数组用于将大写和小写字母进行映射，以进行大小写无关的比较。映射基于ASCII字符序列。
'''
static const u_char charmap[] = {
    (u_char)'\000', (u_char)'\001', (u_char)'\002', (u_char)'\003',
    (u_char)'\004', (u_char)'\005', (u_char)'\006', (u_char)'\007',
    (u_char)'\010', (u_char)'\011', (u_char)'\012', (u_char)'\013',
    (u_char)'\014', (u_char)'\015', (u_char)'\016', (u_char)'\017',
    (u_char)'\020', (u_char)'\021', (u_char)'\022', (u_char)'\023',
    (u_char)'\024', (u_char)'\025', (u_char)'\026', (u_char)'\027',
    (u_char)'\030', (u_char)'\031', (u_char)'\032', (u_char)'\033',
    (u_char)'\034', (u_char)'\035', (u_char)'\036', (u_char)'\037',
    (u_char)'\040', (u_char)'\041', (u_char)'\042', (u_char)'\043',
    (u_char)'\044', (u_char)'\045', (u_char)'\046', (u_char)'\047',
    (u_char)'\050', (u_char)'\051', (u_char)'\052', (u_char)'\053',
    (u_char)'\054', (u_char)'\055', (u_char)'\056', (u_char)'\057',
    (u_char)'\060', (u_char)'\061', (u_char)'\062', (u_char)'\063',
    (u_char)'\064', (u_char)'\065', (u_char)'\066', (u_char)'\067',
    (u_char)'\070', (u_char)'\071', (u_char)'\072', (u_char)'\073',
    (u_char)'\074', (u_char)'\075', (u_char)'\076', (u_char)'\077',
    (u_char)'\100', (u_char)'\141', (u_char)'\142', (u_char)'\143',
    (u_char)'\144', (u_char)'\145', (u_char)'\146', (u_char)'\147',
    (u_char)'\150', (u_char)'\151', (u_char)'\152', (u_char)'\153',
    (u_char)'\154', (u_char)'\155', (u_char)'\156', (u_char)'\157',
    # 定义一系列的字符，每个字符都是一个无符号字符
    (u_char)'\160', (u_char)'\161', (u_char)'\162', (u_char)'\163',
    (u_char)'\164', (u_char)'\165', (u_char)'\166', (u_char)'\167',
    (u_char)'\170', (u_char)'\171', (u_char)'\172', (u_char)'\133',
    (u_char)'\134', (u_char)'\135', (u_char)'\136', (u_char)'\137',
    (u_char)'\140', (u_char)'\141', (u_char)'\142', (u_char)'\143',
    (u_char)'\144', (u_char)'\145', (u_char)'\146', (u_char)'\147',
    (u_char)'\150', (u_char)'\151', (u_char)'\152', (u_char)'\153',
    (u_char)'\154', (u_char)'\155', (u_char)'\156', (u_char)'\157',
    (u_char)'\160', (u_char)'\161', (u_char)'\162', (u_char)'\163',
    (u_char)'\164', (u_char)'\165', (u_char)'\166', (u_char)'\167',
    (u_char)'\170', (u_char)'\171', (u_char)'\172', (u_char)'\173',
    (u_char)'\174', (u_char)'\175', (u_char)'\176', (u_char)'\177',
    (u_char)'\200', (u_char)'\201', (u_char)'\202', (u_char)'\203',
    (u_char)'\204', (u_char)'\205', (u_char)'\206', (u_char)'\207',
    (u_char)'\210', (u_char)'\211', (u_char)'\212', (u_char)'\213',
    (u_char)'\214', (u_char)'\215', (u_char)'\216', (u_char)'\217',
    (u_char)'\220', (u_char)'\221', (u_char)'\222', (u_char)'\223',
    (u_char)'\224', (u_char)'\225', (u_char)'\226', (u_char)'\227',
    (u_char)'\230', (u_char)'\231', (u_char)'\232', (u_char)'\233',
    (u_char)'\234', (u_char)'\235', (u_char)'\236', (u_char)'\237',
    (u_char)'\240', (u_char)'\241', (u_char)'\242', (u_char)'\243',
    (u_char)'\244', (u_char)'\245', (u_char)'\246', (u_char)'\247',
    (u_char)'\250', (u_char)'\251', (u_char)'\252', (u_char)'\253',
    (u_char)'\254', (u_char)'\255', (u_char)'\256', (u_char)'\257',
    (u_char)'\260', (u_char)'\261', (u_char)'\262', (u_char)'\263',
    (u_char)'\264', (u_char)'\265', (u_char)'\266', (u_char)'\267',
    (u_char)'\270', (u_char)'\271', (u_char)'\272', (u_char)'\273',
    (u_char)'\274', (u_char)'\275', (u_char)'\276', (u_char)'\277',
    (u_char)'\300', (u_char)'\341', (u_char)'\342', (u_char)'\343',
    # 定义一系列的无符号字符，每个字符都是以八进制表示的 Unicode 编码
    (u_char)'\344', (u_char)'\345', (u_char)'\346', (u_char)'\347',
    (u_char)'\350', (u_char)'\351', (u_char)'\352', (u_char)'\353',
    (u_char)'\354', (u_char)'\355', (u_char)'\356', (u_char)'\357',
    (u_char)'\360', (u_char)'\361', (u_char)'\362', (u_char)'\363',
    (u_char)'\364', (u_char)'\365', (u_char)'\366', (u_char)'\367',
    (u_char)'\370', (u_char)'\371', (u_char)'\372', (u_char)'\333',
    (u_char)'\334', (u_char)'\335', (u_char)'\336', (u_char)'\337',
    (u_char)'\340', (u_char)'\341', (u_char)'\342', (u_char)'\343',
    (u_char)'\344', (u_char)'\345', (u_char)'\346', (u_char)'\347',
    (u_char)'\350', (u_char)'\351', (u_char)'\352', (u_char)'\353',
    (u_char)'\354', (u_char)'\355', (u_char)'\356', (u_char)'\357',
    (u_char)'\360', (u_char)'\361', (u_char)'\362', (u_char)'\363',
    (u_char)'\364', (u_char)'\365', (u_char)'\366', (u_char)'\367',
    (u_char)'\370', (u_char)'\371', (u_char)'\372', (u_char)'\373',
    (u_char)'\374', (u_char)'\375', (u_char)'\376', (u_char)'\377',
};

int
pcap_strcasecmp(const char *s1, const char *s2)
{
    // 定义指向字符映射表的指针
    register const u_char    *cm = charmap,
                // 定义指向 s1 的 u_char 指针
                *us1 = (const u_char *)s1,
                // 定义指向 s2 的 u_char 指针
                *us2 = (const u_char *)s2;

    // 循环比较两个字符串的每个字符
    while (cm[*us1] == cm[*us2++])
        // 如果字符相等，继续比较下一个字符
        if (*us1++ == '\0')
            // 如果 s1 已经比较完毕，返回 0
            return(0);
    // 返回两个字符串第一个不相等字符的差值
    return (cm[*us1] - cm[*--us2]);
}

// 定义结构体 dlt_choice
struct dlt_choice {
    const char *name;
    const char *description;
    int    dlt;
};

// 定义宏 DLT_CHOICE，用于初始化 dlt_choice 结构体
#define DLT_CHOICE(code, description) { #code, description, DLT_ ## code }
// 定义结构体数组 dlt_choices，用于存储各种数据链路类型的选择
static struct dlt_choice dlt_choices[] = {
    // 初始化各种数据链路类型的选择
    DLT_CHOICE(NULL, "BSD loopback"),
    DLT_CHOICE(EN10MB, "Ethernet"),
    DLT_CHOICE(IEEE802, "Token ring"),
    DLT_CHOICE(ARCNET, "BSD ARCNET"),
    DLT_CHOICE(SLIP, "SLIP"),
    DLT_CHOICE(PPP, "PPP"),
    DLT_CHOICE(FDDI, "FDDI"),
    DLT_CHOICE(ATM_RFC1483, "RFC 1483 LLC-encapsulated ATM"),
    DLT_CHOICE(RAW, "Raw IP"),
    DLT_CHOICE(SLIP_BSDOS, "BSD/OS SLIP"),
    DLT_CHOICE(PPP_BSDOS, "BSD/OS PPP"),
    DLT_CHOICE(ATM_CLIP, "Linux Classical IP over ATM"),
    DLT_CHOICE(PPP_SERIAL, "PPP over serial"),
    DLT_CHOICE(PPP_ETHER, "PPPoE"),
    DLT_CHOICE(SYMANTEC_FIREWALL, "Symantec Firewall"),
    DLT_CHOICE(C_HDLC, "Cisco HDLC"),
    DLT_CHOICE(IEEE802_11, "802.11"),
    DLT_CHOICE(FRELAY, "Frame Relay"),
    DLT_CHOICE(LOOP, "OpenBSD loopback"),
    DLT_CHOICE(ENC, "OpenBSD encapsulated IP"),
    DLT_CHOICE(LINUX_SLL, "Linux cooked v1"),
    DLT_CHOICE(LTALK, "Localtalk"),
    DLT_CHOICE(PFLOG, "OpenBSD pflog file"),
    DLT_CHOICE(PFSYNC, "Packet filter state syncing"),
    DLT_CHOICE(PRISM_HEADER, "802.11 plus Prism header"),
    DLT_CHOICE(IP_OVER_FC, "RFC 2625 IP-over-Fibre Channel"),
    DLT_CHOICE(SUNATM, "Sun raw ATM"),
    DLT_CHOICE(IEEE802_11_RADIO, "802.11 plus radiotap header"),
    DLT_CHOICE(ARCNET_LINUX, "Linux ARCNET"),
    DLT_CHOICE(JUNIPER_MLPPP, "Juniper Multi-Link PPP"),
    DLT_CHOICE(JUNIPER_MLFR, "Juniper Multi-Link Frame Relay"),
    DLT_CHOICE(JUNIPER_ES, "Juniper Encryption Services PIC"),  # 定义一个名为 JUNIPER_ES 的常量，表示 Juniper 加密服务 PIC
    DLT_CHOICE(JUNIPER_GGSN, "Juniper GGSN PIC"),  # 定义一个名为 JUNIPER_GGSN 的常量，表示 Juniper GGSN PIC
    DLT_CHOICE(JUNIPER_MFR, "Juniper FRF.16 Frame Relay"),  # 定义一个名为 JUNIPER_MFR 的常量，表示 Juniper FRF.16 帧中继
    DLT_CHOICE(JUNIPER_ATM2, "Juniper ATM2 PIC"),  # 定义一个名为 JUNIPER_ATM2 的常量，表示 Juniper ATM2 PIC
    DLT_CHOICE(JUNIPER_SERVICES, "Juniper Advanced Services PIC"),  # 定义一个名为 JUNIPER_SERVICES 的常量，表示 Juniper 高级服务 PIC
    DLT_CHOICE(JUNIPER_ATM1, "Juniper ATM1 PIC"),  # 定义一个名为 JUNIPER_ATM1 的常量，表示 Juniper ATM1 PIC
    DLT_CHOICE(APPLE_IP_OVER_IEEE1394, "Apple IP-over-IEEE 1394"),  # 定义一个名为 APPLE_IP_OVER_IEEE1394 的常量，表示 Apple IP-over-IEEE 1394
    DLT_CHOICE(MTP2_WITH_PHDR, "SS7 MTP2 with Pseudo-header"),  # 定义一个名为 MTP2_WITH_PHDR 的常量，表示 SS7 MTP2 带伪头
    DLT_CHOICE(MTP2, "SS7 MTP2"),  # 定义一个名为 MTP2 的常量，表示 SS7 MTP2
    DLT_CHOICE(MTP3, "SS7 MTP3"),  # 定义一个名为 MTP3 的常量，表示 SS7 MTP3
    DLT_CHOICE(SCCP, "SS7 SCCP"),  # 定义一个名为 SCCP 的常量，表示 SS7 SCCP
    DLT_CHOICE(DOCSIS, "DOCSIS"),  # 定义一个名为 DOCSIS 的常量，表示 DOCSIS
    DLT_CHOICE(LINUX_IRDA, "Linux IrDA"),  # 定义一个名为 LINUX_IRDA 的常量，表示 Linux IrDA
    DLT_CHOICE(IEEE802_11_RADIO_AVS, "802.11 plus AVS radio information header"),  # 定义一个名为 IEEE802_11_RADIO_AVS 的常量，表示 802.11 加 AVS 无线电信息头
    DLT_CHOICE(JUNIPER_MONITOR, "Juniper Passive Monitor PIC"),  # 定义一个名为 JUNIPER_MONITOR 的常量，表示 Juniper 被动监视 PIC
    DLT_CHOICE(BACNET_MS_TP, "BACnet MS/TP"),  # 定义一个名为 BACNET_MS_TP 的常量，表示 BACnet MS/TP
    DLT_CHOICE(PPP_PPPD, "PPP for pppd, with direction flag"),  # 定义一个名为 PPP_PPPD 的常量，表示 pppd 的 PPP，带有方向标志
    DLT_CHOICE(JUNIPER_PPPOE, "Juniper PPPoE"),  # 定义一个名为 JUNIPER_PPPOE 的常量，表示 Juniper PPPoE
    DLT_CHOICE(JUNIPER_PPPOE_ATM, "Juniper PPPoE/ATM"),  # 定义一个名为 JUNIPER_PPPOE_ATM 的常量，表示 Juniper PPPoE/ATM
    DLT_CHOICE(GPRS_LLC, "GPRS LLC"),  # 定义一个名为 GPRS_LLC 的常量，表示 GPRS LLC
    DLT_CHOICE(GPF_T, "GPF-T"),  # 定义一个名为 GPF_T 的常量，表示 GPF-T
    DLT_CHOICE(GPF_F, "GPF-F"),  # 定义一个名为 GPF_F 的常量，表示 GPF-F
    DLT_CHOICE(JUNIPER_PIC_PEER, "Juniper PIC Peer"),  # 定义一个名为 JUNIPER_PIC_PEER 的常量，表示 Juniper PIC 对等
    DLT_CHOICE(ERF_ETH, "Ethernet with Endace ERF header"),  # 定义一个名为 ERF_ETH 的常量，表示 带有 Endace ERF 头的以太网
    DLT_CHOICE(ERF_POS, "Packet-over-SONET with Endace ERF header"),  # 定义一个名为 ERF_POS 的常量，表示 带有 Endace ERF 头的 SONET 包
    DLT_CHOICE(LINUX_LAPD, "Linux vISDN LAPD"),  # 定义一个名为 LINUX_LAPD 的常量，表示 Linux vISDN LAPD
    DLT_CHOICE(JUNIPER_ETHER, "Juniper Ethernet"),  # 定义一个名为 JUNIPER_ETHER 的常量，表示 Juniper 以太网
    DLT_CHOICE(JUNIPER_PPP, "Juniper PPP"),  # 定义一个名为 JUNIPER_PPP 的常量，表示 Juniper PPP
    DLT_CHOICE(JUNIPER_FRELAY, "Juniper Frame Relay"),  # 定义一个名为 JUNIPER_FRELAY 的常量，表示 Juniper 帧中继
    DLT_CHOICE(JUNIPER_CHDLC, "Juniper C-HDLC"),  # 定义一个名为 JUNIPER_CHDLC 的常量，表示 Juniper C-HDLC
    DLT_CHOICE(MFR, "FRF.16 Frame Relay"),  # 定义一个名为 MFR 的常量，表示 FRF.16 帧中继
    DLT_CHOICE(JUNIPER_VP, "Juniper Voice PIC"),  # 定义一个名为 JUNIPER_VP 的常量，表示 Juniper 语音 PIC
    DLT_CHOICE(A429, "Arinc 429"),  # 定义一个名为 A429 的常量，表示 Arinc 429
    DLT_CHOICE(A653_ICM, "Arinc 653 Interpartition Communication"),  # 定义一个名为 A653_ICM 的常量，表示 Arinc 653 分区间通信
    DLT_CHOICE(USB_FREEBSD, "USB with FreeBSD header"),  # 定义一个名为 USB_FREEBSD 的常量，表示 带有 FreeBSD 头的 USB
    DLT_CHOICE(BLUETOOTH_HCI_H4, "Bluetooth HCI UART transport layer"),  # 定义一个名为 BLUETOOTH_HCI_H4 的常量，表示 Bluetooth HCI UART 传输层
    DLT_CHOICE(IEEE802_16_MAC_CPS, "IEEE 802.16 MAC Common Part Sublayer"),  # 定义一个名为 IEEE802_16_MAC_CPS 的常量，表示 IEEE 802.16 MAC 公共部分子层
    DLT_CHOICE(USB_LINUX, "USB with Linux header"),  # 定义一个名为 USB_LINUX 的常量，表示 带有 Linux 头的 USB
    DLT_CHOICE(CAN20B, "Controller Area Network (CAN) v. 2.0B"),  # 定义一个常量 CAN20B，表示 Controller Area Network (CAN) v. 2.0B
    DLT_CHOICE(IEEE802_15_4_LINUX, "IEEE 802.15.4 with Linux padding"),  # 定义一个常量 IEEE802_15_4_LINUX，表示 IEEE 802.15.4 with Linux padding
    DLT_CHOICE(PPI, "Per-Packet Information"),  # 定义一个常量 PPI，表示 Per-Packet Information
    DLT_CHOICE(IEEE802_16_MAC_CPS_RADIO, "IEEE 802.16 MAC Common Part Sublayer plus radiotap header"),  # 定义一个常量 IEEE802_16_MAC_CPS_RADIO，表示 IEEE 802.16 MAC Common Part Sublayer plus radiotap header
    DLT_CHOICE(JUNIPER_ISM, "Juniper Integrated Service Module"),  # 定义一个常量 JUNIPER_ISM，表示 Juniper Integrated Service Module
    DLT_CHOICE(IEEE802_15_4, "IEEE 802.15.4 with FCS"),  # 定义一个常量 IEEE802_15_4，表示 IEEE 802.15.4 with FCS
    DLT_CHOICE(SITA, "SITA pseudo-header"),  # 定义一个常量 SITA，表示 SITA pseudo-header
    DLT_CHOICE(ERF, "Endace ERF header"),  # 定义一个常量 ERF，表示 Endace ERF header
    DLT_CHOICE(RAIF1, "Ethernet with u10 Networks pseudo-header"),  # 定义一个常量 RAIF1，表示 Ethernet with u10 Networks pseudo-header
    DLT_CHOICE(IPMB_KONTRON, "IPMB with Kontron pseudo-header"),  # 定义一个常量 IPMB_KONTRON，表示 IPMB with Kontron pseudo-header
    DLT_CHOICE(JUNIPER_ST, "Juniper Secure Tunnel"),  # 定义一个常量 JUNIPER_ST，表示 Juniper Secure Tunnel
    DLT_CHOICE(BLUETOOTH_HCI_H4_WITH_PHDR, "Bluetooth HCI UART transport layer plus pseudo-header"),  # 定义一个常量 BLUETOOTH_HCI_H4_WITH_PHDR，表示 Bluetooth HCI UART transport layer plus pseudo-header
    DLT_CHOICE(AX25_KISS, "AX.25 with KISS header"),  # 定义一个常量 AX25_KISS，表示 AX.25 with KISS header
    DLT_CHOICE(IPMB_LINUX, "IPMB with Linux/Pigeon Point pseudo-header"),  # 定义一个常量 IPMB_LINUX，表示 IPMB with Linux/Pigeon Point pseudo-header
    DLT_CHOICE(IEEE802_15_4_NONASK_PHY, "IEEE 802.15.4 with non-ASK PHY data"),  # 定义一个常量 IEEE802_15_4_NONASK_PHY，表示 IEEE 802.15.4 with non-ASK PHY data
    DLT_CHOICE(MPLS, "MPLS with label as link-layer header"),  # 定义一个常量 MPLS，表示 MPLS with label as link-layer header
    DLT_CHOICE(LINUX_EVDEV, "Linux evdev events"),  # 定义一个常量 LINUX_EVDEV，表示 Linux evdev events
    DLT_CHOICE(USB_LINUX_MMAPPED, "USB with padded Linux header"),  # 定义一个常量 USB_LINUX_MMAPPED，表示 USB with padded Linux header
    DLT_CHOICE(DECT, "DECT"),  # 定义一个常量 DECT，表示 DECT
    DLT_CHOICE(AOS, "AOS Space Data Link protocol"),  # 定义一个常量 AOS，表示 AOS Space Data Link protocol
    DLT_CHOICE(WIHART, "Wireless HART"),  # 定义一个常量 WIHART，表示 Wireless HART
    DLT_CHOICE(FC_2, "Fibre Channel FC-2"),  # 定义一个常量 FC_2，表示 Fibre Channel FC-2
    DLT_CHOICE(FC_2_WITH_FRAME_DELIMS, "Fibre Channel FC-2 with frame delimiters"),  # 定义一个常量 FC_2_WITH_FRAME_DELIMS，表示 Fibre Channel FC-2 with frame delimiters
    DLT_CHOICE(IPNET, "Solaris ipnet"),  # 定义一个常量 IPNET，表示 Solaris ipnet
    DLT_CHOICE(CAN_SOCKETCAN, "CAN-bus with SocketCAN headers"),  # 定义一个常量 CAN_SOCKETCAN，表示 CAN-bus with SocketCAN headers
    DLT_CHOICE(IPV4, "Raw IPv4"),  # 定义一个常量 IPV4，表示 Raw IPv4
    DLT_CHOICE(IPV6, "Raw IPv6"),  # 定义一个常量 IPV6，表示 Raw IPv6
    DLT_CHOICE(IEEE802_15_4_NOFCS, "IEEE 802.15.4 without FCS"),  # 定义一个常量 IEEE802_15_4_NOFCS，表示 IEEE 802.15.4 without FCS
    DLT_CHOICE(DBUS, "D-Bus"),  # 定义一个常量 DBUS，表示 D-Bus
    DLT_CHOICE(JUNIPER_VS, "Juniper Virtual Server"),  # 定义一个常量 JUNIPER_VS，表示 Juniper Virtual Server
    DLT_CHOICE(JUNIPER_SRX_E2E, "Juniper SRX E2E"),  # 定义一个常量 JUNIPER_SRX_E2E，表示 Juniper SRX E2E
    DLT_CHOICE(JUNIPER_FIBRECHANNEL, "Juniper Fibre Channel"),  # 定义一个常量 JUNIPER_FIBRECHANNEL，表示 Juniper Fibre Channel
    DLT_CHOICE(DVB_CI, "DVB-CI"),  # 定义一个常量 DVB_CI，表示 DVB-CI
    DLT_CHOICE(MUX27010, "MUX27010"),  # 定义一个常量 MUX27010，表示 MUX27010
    DLT_CHOICE(STANAG_5066_D_PDU, "STANAG 5066 D_PDUs"),  # 定义一个常量 STANAG_5066_D_PDU，表示 STANAG 5066 D_PDUs
    DLT_CHOICE(JUNIPER_ATM_CEMIC, "Juniper ATM CEMIC"),  # 定义一个数据链路类型，表示Juniper ATM CEMIC
    DLT_CHOICE(NFLOG, "Linux netfilter log messages"),  # 定义一个数据链路类型，表示Linux netfilter日志消息
    DLT_CHOICE(NETANALYZER, "Ethernet with Hilscher netANALYZER pseudo-header"),  # 定义一个数据链路类型，表示带有Hilscher netANALYZER伪头的以太网
    DLT_CHOICE(NETANALYZER_TRANSPARENT, "Ethernet with Hilscher netANALYZER pseudo-header and with preamble and SFD"),  # 定义一个数据链路类型，表示带有Hilscher netANALYZER伪头、前导码和SFD的以太网
    DLT_CHOICE(IPOIB, "RFC 4391 IP-over-Infiniband"),  # 定义一个数据链路类型，表示RFC 4391中的IP-over-Infiniband
    DLT_CHOICE(MPEG_2_TS, "MPEG-2 transport stream"),  # 定义一个数据链路类型，表示MPEG-2传输流
    DLT_CHOICE(NG40, "ng40 protocol tester Iub/Iur"),  # 定义一个数据链路类型，表示ng40协议测试仪Iub/Iur
    DLT_CHOICE(NFC_LLCP, "NFC LLCP PDUs with pseudo-header"),  # 定义一个数据链路类型，表示带有伪头的NFC LLCP PDU
    DLT_CHOICE(INFINIBAND, "InfiniBand"),  # 定义一个数据链路类型，表示InfiniBand
    DLT_CHOICE(SCTP, "SCTP"),  # 定义一个数据链路类型，表示SCTP
    DLT_CHOICE(USBPCAP, "USB with USBPcap header"),  # 定义一个数据链路类型，表示带有USBPcap头的USB
    DLT_CHOICE(RTAC_SERIAL, "Schweitzer Engineering Laboratories RTAC packets"),  # 定义一个数据链路类型，表示Schweitzer Engineering Laboratories RTAC数据包
    DLT_CHOICE(BLUETOOTH_LE_LL, "Bluetooth Low Energy air interface"),  # 定义一个数据链路类型，表示蓝牙低功耗空中接口
    DLT_CHOICE(NETLINK, "Linux netlink"),  # 定义一个数据链路类型，表示Linux netlink
    DLT_CHOICE(BLUETOOTH_LINUX_MONITOR, "Bluetooth Linux Monitor"),  # 定义一个数据链路类型，表示蓝牙Linux监视器
    DLT_CHOICE(BLUETOOTH_BREDR_BB, "Bluetooth Basic Rate/Enhanced Data Rate baseband packets"),  # 定义一个数据链路类型，表示蓝牙基本速率/增强数据速率基带数据包
    DLT_CHOICE(BLUETOOTH_LE_LL_WITH_PHDR, "Bluetooth Low Energy air interface with pseudo-header"),  # 定义一个数据链路类型，表示带有伪头的蓝牙低功耗空中接口
    DLT_CHOICE(PROFIBUS_DL, "PROFIBUS data link layer"),  # 定义一个数据链路类型，表示PROFIBUS数据链路层
    DLT_CHOICE(PKTAP, "Apple DLT_PKTAP"),  # 定义一个数据链路类型，表示Apple DLT_PKTAP
    DLT_CHOICE(EPON, "Ethernet with 802.3 Clause 65 EPON preamble"),  # 定义一个数据链路类型，表示带有802.3 Clause 65 EPON前导码的以太网
    DLT_CHOICE(IPMI_HPM_2, "IPMI trace packets"),  # 定义一个数据链路类型，表示IPMI跟踪数据包
    DLT_CHOICE(ZWAVE_R1_R2, "Z-Wave RF profile R1 and R2 packets"),  # 定义一个数据链路类型，表示Z-Wave RF配置文件R1和R2数据包
    DLT_CHOICE(ZWAVE_R3, "Z-Wave RF profile R3 packets"),  # 定义一个数据链路类型，表示Z-Wave RF配置文件R3数据包
    DLT_CHOICE(WATTSTOPPER_DLM, "WattStopper Digital Lighting Management (DLM) and Legrand Nitoo Open protocol"),  # 定义一个数据链路类型，表示WattStopper数字照明管理（DLM）和Legrand Nitoo开放协议
    DLT_CHOICE(ISO_14443, "ISO 14443 messages"),  # 定义一个数据链路类型，表示ISO 14443消息
    DLT_CHOICE(RDS, "IEC 62106 Radio Data System groups"),  # 定义一个数据链路类型，表示IEC 62106无线电数据系统组
    DLT_CHOICE(USB_DARWIN, "USB with Darwin header"),  # 定义一个数据链路类型，表示带有Darwin头的USB
    DLT_CHOICE(OPENFLOW, "OpenBSD DLT_OPENFLOW"),  # 定义一个数据链路类型，表示OpenBSD DLT_OPENFLOW
    DLT_CHOICE(SDLC, "IBM SDLC frames"),  # 定义一个数据链路类型，表示IBM SDLC帧
    DLT_CHOICE(TI_LLN_SNIFFER, "TI LLN sniffer frames"),  # 定义一个数据链路类型，表示TI LLN嗅探器帧
    DLT_CHOICE(VSOCK, "Linux vsock"),  # 定义一个数据链路类型，表示Linux vsock
    DLT_CHOICE(NORDIC_BLE, "Nordic Semiconductor Bluetooth LE sniffer frames"),  # 选择Nordic蓝牙低功耗嗅探器帧
    DLT_CHOICE(DOCSIS31_XRA31, "Excentis XRA-31 DOCSIS 3.1 RF sniffer frames"),  # 选择Excentis XRA-31 DOCSIS 3.1射频嗅探器帧
    DLT_CHOICE(ETHERNET_MPACKET, "802.3br mPackets"),  # 选择802.3br mPackets
    DLT_CHOICE(DISPLAYPORT_AUX, "DisplayPort AUX channel monitoring data"),  # 选择DisplayPort AUX通道监控数据
    DLT_CHOICE(LINUX_SLL2, "Linux cooked v2"),  # 选择Linux cooked v2
    DLT_CHOICE(OPENVIZSLA, "OpenVizsla USB"),  # 选择OpenVizsla USB
    DLT_CHOICE(EBHSCR, "Elektrobit High Speed Capture and Replay (EBHSCR)"),  # 选择Elektrobit高速捕获和重放（EBHSCR）
    DLT_CHOICE(VPP_DISPATCH, "VPP graph dispatch tracer"),  # 选择VPP图分发跟踪器
    DLT_CHOICE(DSA_TAG_BRCM, "Broadcom tag"),  # 选择Broadcom标签
    DLT_CHOICE(DSA_TAG_BRCM_PREPEND, "Broadcom tag (prepended)"),  # 选择Broadcom标签（前置）
    DLT_CHOICE(IEEE802_15_4_TAP, "IEEE 802.15.4 with pseudo-header"),  # 选择带有伪标头的IEEE 802.15.4
    DLT_CHOICE(DSA_TAG_DSA, "Marvell DSA"),  # 选择Marvell DSA
    DLT_CHOICE(DSA_TAG_EDSA, "Marvell EDSA"),  # 选择Marvell EDSA
    DLT_CHOICE(ELEE, "ELEE lawful intercept packets"),  # 选择ELEE合法拦截数据包
    DLT_CHOICE(Z_WAVE_SERIAL, "Z-Wave serial frames between host and chip"),  # 选择主机和芯片之间的Z-Wave串行帧
    DLT_CHOICE(USB_2_0, "USB 2.0/1.1/1.0 as transmitted over the cable"),  # 选择通过电缆传输的USB 2.0/1.1/1.0
    DLT_CHOICE(ATSC_ALP, "ATSC Link-Layer Protocol packets"),  # 选择ATSC链路层协议数据包
    DLT_CHOICE_SENTINEL  # 选择哨兵
};

// 根据数据链路类型名称获取对应的数值
int
pcap_datalink_name_to_val(const char *name)
{
    int i;

    // 遍历数据链路类型选择列表
    for (i = 0; dlt_choices[i].name != NULL; i++) {
        // 如果找到匹配的数据链路类型名称，则返回对应的数值
        if (pcap_strcasecmp(dlt_choices[i].name, name) == 0)
            return (dlt_choices[i].dlt);
    }
    // 如果未找到匹配的数据链路类型名称，则返回-1
    return (-1);
}

// 根据数据链路类型数值获取对应的名称
const char *
pcap_datalink_val_to_name(int dlt)
{
    int i;

    // 遍历数据链路类型选择列表
    for (i = 0; dlt_choices[i].name != NULL; i++) {
        // 如果找到匹配的数据链路类型数值，则返回对应的名称
        if (dlt_choices[i].dlt == dlt)
            return (dlt_choices[i].name);
    }
    // 如果未找到匹配的数据链路类型数值，则返回NULL
    return (NULL);
}

// 根据数据链路类型数值获取对应的描述
const char *
pcap_datalink_val_to_description(int dlt)
{
    int i;

    // 遍历数据链路类型选择列表
    for (i = 0; dlt_choices[i].name != NULL; i++) {
        // 如果找到匹配的数据链路类型数值，则返回对应的描述
        if (dlt_choices[i].dlt == dlt)
            return (dlt_choices[i].description);
    }
    // 如果未找到匹配的数据链路类型数值，则返回NULL
    return (NULL);
}

// 根据数据链路类型数值获取对应的描述或数值
const char *
pcap_datalink_val_to_description_or_dlt(int dlt)
{
        static char unkbuf[40];
        const char *description;

        // 获取数据链路类型数值对应的描述
        description = pcap_datalink_val_to_description(dlt);
        // 如果找到描述，则返回描述
        if (description != NULL) {
                return description;
        } else {
                // 否则返回未知的数据链路类型数值
                (void)snprintf(unkbuf, sizeof(unkbuf), "DLT %d", dlt);
                return unkbuf;
        }
}

// 时间戳类型选择列表
struct tstamp_type_choice {
    const char *name;
    const char *description;
    int    type;
};

// 时间戳类型选择列表的具体内容
static struct tstamp_type_choice tstamp_type_choices[] = {
    { "host", "Host", PCAP_TSTAMP_HOST },
    { "host_lowprec", "Host, low precision", PCAP_TSTAMP_HOST_LOWPREC },
    { "host_hiprec", "Host, high precision", PCAP_TSTAMP_HOST_HIPREC },
    { "adapter", "Adapter", PCAP_TSTAMP_ADAPTER },
    { "adapter_unsynced", "Adapter, not synced with system time", PCAP_TSTAMP_ADAPTER_UNSYNCED },
    { "host_hiprec_unsynced", "Host, high precision, not synced with system time", PCAP_TSTAMP_HOST_HIPREC_UNSYNCED },
    { NULL, NULL, 0 }
};

// 根据时间戳类型名称获取对应的数值
int
pcap_tstamp_type_name_to_val(const char *name)
{
    int i;

    // 遍历时间戳类型选择列表
    for (i = 0; tstamp_type_choices[i].name != NULL; i++) {
        // 如果找到匹配的时间戳类型名称，则返回对应的数值
        if (pcap_strcasecmp(tstamp_type_choices[i].name, name) == 0)
            return (tstamp_type_choices[i].type);
    }
    # 返回一个错误代码，表示出现了 PCAP 错误
    return (PCAP_ERROR);
}

这是一个函数结束的右括号，表示函数定义结束。


const char *
pcap_tstamp_type_val_to_name(int tstamp_type)
{
    int i;

    for (i = 0; tstamp_type_choices[i].name != NULL; i++) {
        if (tstamp_type_choices[i].type == tstamp_type)
            return (tstamp_type_choices[i].name);
    }
    return (NULL);
}

定义了一个函数，根据时间戳类型的值返回时间戳类型的名称。


const char *
pcap_tstamp_type_val_to_description(int tstamp_type)
{
    int i;

    for (i = 0; tstamp_type_choices[i].name != NULL; i++) {
        if (tstamp_type_choices[i].type == tstamp_type)
            return (tstamp_type_choices[i].description);
    }
    return (NULL);
}

定义了一个函数，根据时间戳类型的值返回时间戳类型的描述。


int
pcap_snapshot(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    return (p->snapshot);
}

定义了一个函数，返回抓包快照长度。


int
pcap_is_swapped(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    return (p->swapped);
}

定义了一个函数，返回抓包数据是否进行了交换。


int
pcap_major_version(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    return (p->version_major);
}

定义了一个函数，返回抓包文件的主要版本号。


int
pcap_minor_version(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    return (p->version_minor);
}

定义了一个函数，返回抓包文件的次要版本号。


int
pcap_bufsize(pcap_t *p)
{
    if (!p->activated)
        return (PCAP_ERROR_NOT_ACTIVATED);
    return (p->bufsize);
}

定义了一个函数，返回抓包缓冲区大小。


FILE *
pcap_file(pcap_t *p)
{
    return (p->rfile);
}

定义了一个函数，返回抓包文件。


#ifdef _WIN32
int
pcap_fileno(pcap_t *p)
{
    if (p->handle != INVALID_HANDLE_VALUE) {
        /*
         * This is a bogus and now-deprecated API; we
         * squelch the narrowing warning for the cast
         * from HANDLE to intptr_t.  If Windows programmmers
         * need to get at the HANDLE for a pcap_t, *if*
         * there is one, they should request such a
         * routine (and be prepared for it to return
         * INVALID_HANDLE_VALUE).
         */
DIAG_OFF_NARROWING
        return ((int)(intptr_t)p->handle);
DIAG_ON_NARROWING
    } else
        return (PCAP_ERROR);
}
#else /* _WIN32 */
int
pcap_fileno(pcap_t *p)
{
    return (p->fd);
}
#endif /* _WIN32 */

定义了一个函数，根据操作系统返回抓包文件描述符。 Windows 系统使用 `HANDLE`，其他系统使用文件描述符。
// 获取 pcap_t 结构体中的 selectable_fd 成员变量
pcap_get_selectable_fd(pcap_t *p)
{
    return (p->selectable_fd);
}

// 获取 pcap_t 结构体中的 required_select_timeout 成员变量
const struct timeval *
pcap_get_required_select_timeout(pcap_t *p)
{
    return (p->required_select_timeout);
}

// 打印错误信息到标准错误输出
void
pcap_perror(pcap_t *p, const char *prefix)
{
    fprintf(stderr, "%s: %s\n", prefix, p->errbuf);
}

// 获取 pcap_t 结构体中的 errbuf 成员变量
char *
pcap_geterr(pcap_t *p)
{
    return (p->errbuf);
}

// 获取 pcap_t 结构体中的 getnonblock_op 成员变量
int
pcap_getnonblock(pcap_t *p, char *errbuf)
{
    int ret;

    ret = p->getnonblock_op(p);
    if (ret == -1) {
        /*
         * The get nonblock operation sets p->errbuf; this
         * function *shouldn't* have had a separate errbuf
         * argument, as it didn't need one, but I goofed
         * when adding it.
         *
         * We copy the error message to errbuf, so callers
         * can find it in either place.
         */
        pcap_strlcpy(errbuf, p->errbuf, PCAP_ERRBUF_SIZE);
    }
    return (ret);
}

// 获取 pcap_t 结构体中的 fd 成员变量的非阻塞模式
int
pcap_getnonblock_fd(pcap_t *p)
{
    int fdflags;

    fdflags = fcntl(p->fd, F_GETFL, 0);
    if (fdflags == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "F_GETFL");
        return (-1);
    }
    if (fdflags & O_NONBLOCK)
        return (1);
    else
        return (0);
}

// 设置 pcap_t 结构体中的 setnonblock_op 成员变量
int
pcap_setnonblock(pcap_t *p, int nonblock, char *errbuf)
{
    int ret;

    ret = p->setnonblock_op(p, nonblock);
    if (ret == -1) {
        /*
         * The set nonblock operation sets p->errbuf; this
         * function *shouldn't* have had a separate errbuf
         * argument, as it didn't need one, but I goofed
         * when adding it.
         *
         * We copy the error message to errbuf, so callers
         * can find it in either place.
         */
        pcap_strlcpy(errbuf, p->errbuf, PCAP_ERRBUF_SIZE);
    }
    return (ret);
}
# 设置非阻塞模式，假设这只是标准的 POSIX 非阻塞标志。 （如果每个平台的非阻塞模式例程还需要做一些额外的工作，这可以由每个平台的非阻塞模式例程调用。）
int
pcap_setnonblock_fd(pcap_t *p, int nonblock)
{
    int fdflags;

    # 获取文件描述符的标志
    fdflags = fcntl(p->fd, F_GETFL, 0);
    if (fdflags == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "F_GETFL");
        return (-1);
    }
    # 根据非阻塞标志设置文件描述符的标志
    if (nonblock)
        fdflags |= O_NONBLOCK;
    else
        fdflags &= ~O_NONBLOCK;
    # 设置文件描述符的标志
    if (fcntl(p->fd, F_SETFL, fdflags) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "F_SETFL");
        return (-1);
    }
    return (0);
}
#endif

# 为 PCAP_ERROR_ 和 PCAP_WARNING_ 值生成错误字符串
const char *
pcap_statustostr(int errnum)
{
    static char ebuf[15+10+1];

    switch (errnum) {

    case PCAP_WARNING:
        return("Generic warning");

    case PCAP_WARNING_TSTAMP_TYPE_NOTSUP:
        return ("That type of time stamp is not supported by that device");

    case PCAP_WARNING_PROMISC_NOTSUP:
        return ("That device doesn't support promiscuous mode");

    case PCAP_ERROR:
        return("Generic error");

    case PCAP_ERROR_BREAK:
        return("Loop terminated by pcap_breakloop");

    case PCAP_ERROR_NOT_ACTIVATED:
        return("The pcap_t has not been activated");

    case PCAP_ERROR_ACTIVATED:
        return ("The setting can't be changed after the pcap_t is activated");

    case PCAP_ERROR_NO_SUCH_DEVICE:
        return ("No such device exists");

    case PCAP_ERROR_RFMON_NOTSUP:
        return ("That device doesn't support monitor mode");

    case PCAP_ERROR_NOT_RFMON:
        return ("That operation is supported only in monitor mode");

    case PCAP_ERROR_PERM_DENIED:
        return ("You don't have permission to perform this capture on that device");
    # 如果捕获设备未启用，则返回错误消息
    case PCAP_ERROR_IFACE_NOT_UP:
        return ("That device is not up");

    # 如果设备不支持设置时间戳类型，则返回错误消息
    case PCAP_ERROR_CANTSET_TSTAMP_TYPE:
        return ("That device doesn't support setting the time stamp type");

    # 如果没有权限在设备上以混杂模式进行捕获，则返回错误消息
    case PCAP_ERROR_PROMISC_PERM_DENIED:
        return ("You don't have permission to capture in promiscuous mode on that device");

    # 如果设备不支持该时间戳精度，则返回错误消息
    case PCAP_ERROR_TSTAMP_PRECISION_NOTSUP:
        return ("That device doesn't support that time stamp precision");
    }

    # 如果出现未知错误，则返回错误消息
    (void)snprintf(ebuf, sizeof ebuf, "Unknown error: %d", errnum);
    return(ebuf);
}
/*
 * Not all systems have strerror().
 * 并非所有系统都有strerror()函数。
 */
const char *
pcap_strerror(int errnum)
{
#ifdef HAVE_STRERROR
#ifdef _WIN32
    static char errbuf[PCAP_ERRBUF_SIZE];
    errno_t err = strerror_s(errbuf, PCAP_ERRBUF_SIZE, errnum);

    if (err != 0) /* err = 0 if successful */
        pcap_strlcpy(errbuf, "strerror_s() error", PCAP_ERRBUF_SIZE);
    return (errbuf);
#else
    return (strerror(errnum));
#endif /* _WIN32 */
#else
    extern int sys_nerr;
    extern const char *const sys_errlist[];
    static char errbuf[PCAP_ERRBUF_SIZE];

    if ((unsigned int)errnum < sys_nerr)
        return ((char *)sys_errlist[errnum]);
    (void)snprintf(errbuf, sizeof errbuf, "Unknown error: %d", errnum);
    return (errbuf);
#endif
}
/*
 * Set direction flag, which controls whether we accept only incoming
 * packets, only outgoing packets, or both.
 * Note that, depending on the platform, some or all direction arguments
 * might not be supported.
 * 设置方向标志，控制我们是只接收传入数据包，只接收传出数据包，还是两者都接收。
 * 请注意，根据平台的不同，某些或所有方向参数可能不受支持。
 */
int
pcap_setdirection(pcap_t *p, pcap_direction_t d)
{
    if (p->setdirection_op == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "Setting direction is not supported on this device");
        return (-1);
    } else {
        switch (d) {

        case PCAP_D_IN:
        case PCAP_D_OUT:
        case PCAP_D_INOUT:
            /*
             * Valid direction.
             * 有效的方向。
             */
            return (p->setdirection_op(p, d));

        default:
            /*
             * Invalid direction.
             * 无效的方向。
             */
            snprintf(p->errbuf, sizeof(p->errbuf),
                "Invalid direction");
            return (-1);
        }
    }
}
/*
 * Get capture statistics.
 * 获取捕获统计信息。
 */
int
pcap_stats(pcap_t *p, struct pcap_stat *ps)
{
    return (p->stats_op(p, ps));
}
#ifdef _WIN32
struct pcap_stat *
pcap_stats_ex(pcap_t *p, int *pcap_stat_size)
{
    return (p->stats_ex_op(p, pcap_stat_size));
}
/*
 * Set the buffer size for a not-yet-activated capture handle.
 * 为尚未激活的捕获句柄设置缓冲区大小。
 */
int
pcap_setbuff(pcap_t *p, int dim)
{
    # 调用 setbuff_op 函数，将其返回值作为参数传递给 p，然后返回结果
    return (p->setbuff_op(p, dim));
}

int
pcap_setmode(pcap_t *p, int mode)
{
    return (p->setmode_op(p, mode));  // 设置抓包模式
}

int
pcap_setmintocopy(pcap_t *p, int size)
{
    return (p->setmintocopy_op(p, size));  // 设置最小拷贝大小
}

HANDLE
pcap_getevent(pcap_t *p)
{
    return (p->getevent_op(p));  // 获取事件
}

int
pcap_oid_get_request(pcap_t *p, bpf_u_int32 oid, void *data, size_t *lenp)
{
    return (p->oid_get_request_op(p, oid, data, lenp));  // 获取 OID 请求
}

int
pcap_oid_set_request(pcap_t *p, bpf_u_int32 oid, const void *data, size_t *lenp)
{
    return (p->oid_set_request_op(p, oid, data, lenp));  // 设置 OID 请求
}

pcap_send_queue *
pcap_sendqueue_alloc(u_int memsize)
{
    pcap_send_queue *tqueue;

    /* Allocate the queue */
    tqueue = (pcap_send_queue *)malloc(sizeof(pcap_send_queue));  // 分配发送队列内存
    if (tqueue == NULL){
        return (NULL);
    }

    /* Allocate the buffer */
    tqueue->buffer = (char *)malloc(memsize);  // 分配缓冲区内存
    if (tqueue->buffer == NULL) {
        free(tqueue);
        return (NULL);
    }

    tqueue->maxlen = memsize;  // 设置最大长度
    tqueue->len = 0;  // 设置长度为0

    return (tqueue);
}

void
pcap_sendqueue_destroy(pcap_send_queue *queue)
{
    free(queue->buffer);  // 释放缓冲区内存
    free(queue);  // 释放发送队列内存
}

int
pcap_sendqueue_queue(pcap_send_queue *queue, const struct pcap_pkthdr *pkt_header, const u_char *pkt_data)
{
    if (queue->len + sizeof(struct pcap_pkthdr) + pkt_header->caplen > queue->maxlen){
        return (-1);  // 如果队列剩余空间不足以存放数据包，则返回-1
    }

    /* Copy the pcap_pkthdr header*/
    memcpy(queue->buffer + queue->len, pkt_header, sizeof(struct pcap_pkthdr));  // 复制数据包头部
    queue->len += sizeof(struct pcap_pkthdr);  // 更新队列长度

    /* copy the packet */
    memcpy(queue->buffer + queue->len, pkt_data, pkt_header->caplen);  // 复制数据包
    queue->len += pkt_header->caplen;  // 更新队列长度

    return (0);
}

u_int
pcap_sendqueue_transmit(pcap_t *p, pcap_send_queue *queue, int sync)
{
    return (p->sendqueue_transmit_op(p, queue, sync));  // 发送队列中的数据包
}

int
pcap_setuserbuffer(pcap_t *p, int size)
{
    return (p->setuserbuffer_op(p, size));  // 设置用户缓冲区大小
}

int
pcap_live_dump(pcap_t *p, char *filename, int maxsize, int maxpacks)
{
    # 调用 live_dump_op 方法，传入参数 p, filename, maxsize, maxpacks，并返回结果
    return (p->live_dump_op(p, filename, maxsize, maxpacks));
/*
 * 检查捕获会话是否已经结束
 * 参数：
 *   p - pcap_t 结构体指针
 *   sync - 同步标志
 * 返回值：
 *   0 - 捕获会话未结束
 *   非0 - 捕获会话已结束
 */
int
pcap_live_dump_ended(pcap_t *p, int sync)
{
    return (p->live_dump_ended_op(p, sync));
}

/*
 * 获取 AirPcap 设备的句柄
 * 参数：
 *   p - pcap_t 结构体指针
 * 返回值：
 *   PAirpcapHandle - AirPcap 设备句柄
 */
PAirpcapHandle
pcap_get_airpcap_handle(pcap_t *p)
{
    PAirpcapHandle handle;

    // 调用操作函数获取 AirPcap 设备句柄
    handle = p->get_airpcap_handle_op(p);
    // 如果句柄为空，设置错误信息
    if (handle == NULL) {
        (void)snprintf(p->errbuf, sizeof(p->errbuf),
            "This isn't an AirPcap device");
    }
    return (handle);
}

/*
 * 在某些平台上，当关闭设备时需要清理混杂模式或监听模式
 * 即使应用程序在不显式关闭设备的情况下退出，我们也希望发生这种情况。
 * 在这些平台上，我们需要注册一个“关闭所有pcap”的例程，以便在退出时调用，并且需要维护一个需要关闭以清理模式的pcap列表。
 * XXX - 不是线程安全的。
 */

/*
 * 需要进行清理的 pcap 列表
 * 如果有任何这样的 pcap，我们安排在退出时调用“pcap_close_all()”，并且让它关闭所有这些 pcap。
 */
static struct pcap *pcaps_to_close;

/*
 * 如果我们已经调用“atexit()”来导致在退出时调用“pcap_close_all()”，则为 TRUE。
 */
static int did_atexit;

/*
 * 关闭所有 pcap
 */
static void
pcap_close_all(void)
{
    struct pcap *handle;
    # 当待关闭的 pcap_t 对象列表中还有对象时，执行循环
    while ((handle = pcaps_to_close) != NULL) {
        # 关闭当前的 pcap_t 对象
        pcap_close(handle);

        '''
         * 如果一个 pcap 模块通过调用 pcap_add_to_pcaps_to_close() 将一个 pcap_t 添加到“关闭所有”列表中，
         * 那么它必须有一个清理例程，通过调用 pcap_remove_from_pcaps_to_close() 从列表中移除它，
         * 并且必须将该清理例程作为 pcap_t 的清理操作。
         *
         * 这意味着，在 pcap_close() 调用清理操作后，pcap_t 必须已经从列表中移除，因此 pcaps_to_close
         * 不应该等于 handle。
         *
         * 我们检查这一点，如果 handle 仍然在列表的头部，则终止程序以防止无限循环。
         '''
        # 如果 handle 仍然在列表的头部，则终止程序以防止无限循环
        if (pcaps_to_close == handle)
            abort();
    }
}

int
pcap_do_addexit(pcap_t *p)
{
    /*
     * 如果我们还没有这样做，就安排在退出时调用"pcap_close_all()"
     */
    if (!did_atexit) {
        if (atexit(pcap_close_all) != 0) {
            /*
             * "atexit()" 失败；让调用者知道。
             */
            pcap_strlcpy(p->errbuf, "atexit failed", PCAP_ERRBUF_SIZE);
            return (0);
        }
        did_atexit = 1;
    }
    return (1);
}

void
pcap_add_to_pcaps_to_close(pcap_t *p)
{
    p->next = pcaps_to_close;
    pcaps_to_close = p;
}

void
pcap_remove_from_pcaps_to_close(pcap_t *p)
{
    pcap_t *pc, *prevpc;

    for (pc = pcaps_to_close, prevpc = NULL; pc != NULL;
        prevpc = pc, pc = pc->next) {
        if (pc == p) {
            /*
             * 找到了。从列表中移除它。
             */
            if (prevpc == NULL) {
                /*
                 * 它在列表头部。
                 */
                pcaps_to_close = pc->next;
            } else {
                /*
                 * 它在列表中间。
                 */
                prevpc->next = pc->next;
            }
            break;
        }
    }
}

void
pcap_breakloop_common(pcap_t *p)
{
    p->break_loop = 1;
}


void
pcap_cleanup_live_common(pcap_t *p)
{
    if (p->opt.device != NULL) {
        free(p->opt.device);
        p->opt.device = NULL;
    }
    if (p->buffer != NULL) {
        free(p->buffer);
        p->buffer = NULL;
    }
    if (p->dlt_list != NULL) {
        free(p->dlt_list);
        p->dlt_list = NULL;
        p->dlt_count = 0;
    }
    if (p->tstamp_type_list != NULL) {
        free(p->tstamp_type_list);
        p->tstamp_type_list = NULL;
        p->tstamp_type_count = 0;
    }
    if (p->tstamp_precision_list != NULL) {
        free(p->tstamp_precision_list);
        p->tstamp_precision_list = NULL;
        p->tstamp_precision_count = 0;
    }
    pcap_freecode(&p->fcode);
#if !defined(_WIN32) && !defined(MSDOS)
    // 如果不是在 Windows 或者 MSDOS 环境下
    if (p->fd >= 0) {
        // 如果文件描述符大于等于0
        close(p->fd);
        // 关闭文件描述符
        p->fd = -1;
        // 将文件描述符设置为-1
    }
    p->selectable_fd = -1;
    // 将可选择的文件描述符设置为-1
#endif
}

/*
 * API compatible with WinPcap's "send a packet" routine - returns -1
 * on error, 0 otherwise.
 *
 * XXX - what if we get a short write?
 */
int
pcap_sendpacket(pcap_t *p, const u_char *buf, int size)
{
    // 如果要发送的字节数小于等于0
    if (size <= 0) {
        // 格式化错误消息，说明发送的字节数必须是正数
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "The number of bytes to be sent must be positive");
        return (PCAP_ERROR);
        // 返回发送错误
    }

    // 调用注入操作函数，如果返回-1则返回错误，否则返回0
    if (p->inject_op(p, buf, size) == -1)
        return (-1);
    return (0);
}

/*
 * API compatible with OpenBSD's "send a packet" routine - returns -1 on
 * error, number of bytes written otherwise.
 */
int
pcap_inject(pcap_t *p, const void *buf, size_t size)
{
    /*
     * We return the number of bytes written, so the number of
     * bytes to write must fit in an int.
     */
    // 返回写入的字节数，因此要写入的字节数必须适合int类型
    if (size > INT_MAX) {
        // 格式化错误消息，说明不能注入超过INT_MAX字节的数据
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "More than %d bytes cannot be injected", INT_MAX);
        return (PCAP_ERROR);
        // 返回注入错误
    }

    // 如果要注入的字节数为0
    if (size == 0) {
        // 格式化错误消息，说明不能注入0字节的数据
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "The number of bytes to be injected must not be zero");
        return (PCAP_ERROR);
        // 返回注入错误
    }

    return (p->inject_op(p, buf, (int)size));
}

void
pcap_close(pcap_t *p)
{
    // 调用清理操作函数
    p->cleanup_op(p);
    // 释放内存
    free(p);
}

/*
 * Helpers for safely loading code at run time.
 * Currently Windows-only.
 */
#ifdef _WIN32
//
// This wrapper around loadlibrary appends the system folder (usually
// C:\Windows\System32) to the relative path of the DLL, so that the DLL
// is always loaded from an absolute path (it's no longer possible to
// load modules from the application folder).
// This solves the DLL Hijacking issue discovered in August 2010:
//
// https://blog.rapid7.com/2010/08/23/exploiting-dll-hijacking-flaws/
// 定义一个函数，用于加载指定名称的代码库
pcap_code_handle_t
pcap_load_code(const char *name)
{
    /*
     * XXX - should this work in UTF-16LE rather than in the local
     * ANSI code page?
     */
    // 定义存储路径和完整文件名的变量
    CHAR path[MAX_PATH];
    CHAR fullFileName[MAX_PATH];
    UINT res;
    HMODULE hModule = NULL;

    // 循环执行以下操作
    do
    {
        // 获取系统目录并存储到path中
        res = GetSystemDirectoryA(path, MAX_PATH);

        // 如果获取失败，则跳出循环
        if (res == 0) {
            //
            // some bad failure occurred;
            //
            break;
        }

        // 如果获取的路径长度超过了MAX_PATH，则跳出循环
        if (res > MAX_PATH) {
            //
            // the buffer was not big enough
            //
            SetLastError(ERROR_INSUFFICIENT_BUFFER);
            break;
        }

        // 如果路径长度加上文件名长度小于MAX_PATH，则拼接完整文件名并加载代码库
        if (res + 1 + strlen(name) + 1 < MAX_PATH) {
            memcpy(fullFileName, path, res * sizeof(TCHAR));
            fullFileName[res] = '\\';
            memcpy(&fullFileName[res + 1], name, (strlen(name) + 1) * sizeof(TCHAR));

            hModule = LoadLibraryA(fullFileName);
        } else
            SetLastError(ERROR_INSUFFICIENT_BUFFER);

    } while(FALSE);

    // 返回加载的代码库句柄
    return hModule;
}

// 定义一个函数，用于查找代码库中的指定函数
pcap_funcptr_t
pcap_find_function(pcap_code_handle_t code, const char *func)
{
    return (GetProcAddress(code, func));
}
#endif

/*
 * Given a BPF program, a pcap_pkthdr structure for a packet, and the raw
 * data for the packet, check whether the packet passes the filter.
 * Returns the return value of the filter program, which will be zero if
 * the packet doesn't pass and non-zero if the packet does pass.
 */
// 定义一个函数，用于检查数据包是否通过过滤器
int
# 从给定的 BPF 程序过滤数据包，返回过滤结果
pcap_offline_filter(const struct bpf_program *fp, const struct pcap_pkthdr *h,
    const u_char *pkt)
{
    # 从 BPF 程序中获取指令集
    const struct bpf_insn *fcode = fp->bf_insns;

    # 如果指令集不为空，则调用 pcap_filter 进行数据包过滤
    if (fcode != NULL)
        return (pcap_filter(fcode, pkt, h->len, h->caplen));
    # 如果指令集为空，则返回 0
    else
        return (0);
}

# 检查是否可以设置 RFMON 模式，对于 pcap_open_dead 创建的 pcap_t 不支持 RFMON 模式
static int
pcap_can_set_rfmon_dead(pcap_t *p)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Rfmon mode doesn't apply on a pcap_open_dead pcap_t");
    # 返回错误
    return (PCAP_ERROR);
}

# 从 pcap_open_dead 创建的 pcap_t 中读取数据包，但不支持此操作
static int
pcap_read_dead(pcap_t *p, int cnt _U_, pcap_handler callback _U_,
    u_char *user _U_)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Packets aren't available from a pcap_open_dead pcap_t");
    # 返回 -1 表示错误
    return (-1);
}

# 中断从 pcap_open_dead 创建的 pcap_t 中的数据包捕获
static void
pcap_breakloop_dead(pcap_t *p _U_)
{
    # 对于 "dead" pcap_t，不支持捕获或读取数据包，因此不需要执行任何操作
}

# 向 pcap_open_dead 创建的 pcap_t 中注入数据包，但不支持此操作
static int
pcap_inject_dead(pcap_t *p, const void *buf _U_, int size _U_)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Packets can't be sent on a pcap_open_dead pcap_t");
    # 返回 -1 表示错误
    return (-1);
}

# 在 pcap_open_dead 创建的 pcap_t 上设置过滤器，但不支持此操作
static int
pcap_setfilter_dead(pcap_t *p, struct bpf_program *fp _U_)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "A filter cannot be set on a pcap_open_dead pcap_t");
    # 返回 -1 表示错误
    return (-1);
}

# 设置 pcap_open_dead 创建的 pcap_t 的数据包方向，但不支持此操作
static int
pcap_setdirection_dead(pcap_t *p, pcap_direction_t d _U_)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The packet direction cannot be set on a pcap_open_dead pcap_t");
    # 返回 -1 表示错误
    return (-1);
}

# 设置 pcap_open_dead 创建的 pcap_t 的数据链路类型，但不支持此操作
static int
pcap_set_datalink_dead(pcap_t *p, int dlt _U_)
{
    # 设置错误信息
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The link-layer header type cannot be set on a pcap_open_dead pcap_t");
    # 返回 -1 表示错误
    return (-1);
}

# 获取 pcap_open_dead 创建的 pcap_t 的非阻塞状态，但不支持此操作
static int
pcap_getnonblock_dead(pcap_t *p)
    # 使用snprintf函数将错误信息格式化并存储到p->errbuf中，PCAP_ERRBUF_SIZE为errbuf的大小
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "A pcap_open_dead pcap_t does not have a non-blocking mode setting");
    # 返回-1，表示出错
    return (-1);
# 设置 pcap_t 结构体的非阻塞模式为死模式
static int
pcap_setnonblock_dead(pcap_t *p, int nonblock _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "A pcap_open_dead pcap_t does not have a non-blocking mode setting");
    # 返回 -1 表示失败
    return (-1);
}

# 获取死模式下的 pcap_t 结构体的统计信息
static int
pcap_stats_dead(pcap_t *p, struct pcap_stat *ps _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Statistics aren't available from a pcap_open_dead pcap_t");
    # 返回 -1 表示失败
    return (-1);
}

# Windows 下获取死模式下的 pcap_t 结构体的统计信息
#ifdef _WIN32
static struct pcap_stat *
pcap_stats_ex_dead(pcap_t *p, int *pcap_stat_size _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Statistics aren't available from a pcap_open_dead pcap_t");
    # 返回 NULL 表示失败
    return (NULL);
}

# 设置死模式下的 pcap_t 结构体的缓冲区大小
static int
pcap_setbuff_dead(pcap_t *p, int dim _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The kernel buffer size cannot be set on a pcap_open_dead pcap_t");
    # 返回 -1 表示失败
    return (-1);
}

# 设置死模式下的 pcap_t 结构体的模式
static int
pcap_setmode_dead(pcap_t *p, int mode _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "impossible to set mode on a pcap_open_dead pcap_t");
    # 返回 -1 表示失败
    return (-1);
}

# 设置死模式下的 pcap_t 结构体的最小拷贝大小
static int
pcap_setmintocopy_dead(pcap_t *p, int size _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The mintocopy parameter cannot be set on a pcap_open_dead pcap_t");
    # 返回 -1 表示失败
    return (-1);
}

# 获取死模式下的 pcap_t 结构体的事件句柄
static HANDLE
pcap_getevent_dead(pcap_t *p)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "A pcap_open_dead pcap_t has no event handle");
    # 返回无效句柄表示失败
    return (INVALID_HANDLE_VALUE);
}

# 获取死模式下的 pcap_t 结构体的 OID get 请求
static int
pcap_oid_get_request_dead(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_,
    size_t *lenp _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID get request cannot be performed on a pcap_open_dead pcap_t");
    # 返回 PCAP_ERROR 表示失败
    return (PCAP_ERROR);
}

# 设置死模式下的 pcap_t 结构体的 OID set 请求
static int
pcap_oid_set_request_dead(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
    # 将错误信息写入 errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID set request cannot be performed on a pcap_open_dead pcap_t");
    # 返回 PCAP_ERROR 表示失败
    return (PCAP_ERROR);
}

# 传输死模式下的 pcap_t 结构体的发送队列
static u_int
pcap_sendqueue_transmit_dead(pcap_t *p, pcap_send_queue *queue _U_,
    # 声明一个名为 sync 的整型变量，但是语法错误，应该为 int sync_U_
{
    // 设置错误缓冲区的错误消息，指示不能在 pcap_open_dead pcap_t 上传输数据包
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Packets cannot be transmitted on a pcap_open_dead pcap_t");
    // 返回 0，表示失败
    return (0);
}

static int
pcap_setuserbuffer_dead(pcap_t *p, int size _U_)
{
    // 设置错误缓冲区的错误消息，指示不能在 pcap_open_dead pcap_t 上设置用户缓冲区
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The user buffer cannot be set on a pcap_open_dead pcap_t");
    // 返回 -1，表示失败
    return (-1);
}

static int
pcap_live_dump_dead(pcap_t *p, char *filename _U_, int maxsize _U_,
    int maxpacks _U_)
{
    // 设置错误缓冲区的错误消息，指示不能在 pcap_open_dead pcap_t 上执行实时数据包转储
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed on a pcap_open_dead pcap_t");
    // 返回 -1，表示失败
    return (-1);
}

static int
pcap_live_dump_ended_dead(pcap_t *p, int sync _U_)
{
    // 设置错误缓冲区的错误消息，指示不能在 pcap_open_dead pcap_t 上执行实时数据包转储
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed on a pcap_open_dead pcap_t");
    // 返回 -1，表示失败
    return (-1);
}

static PAirpcapHandle
pcap_get_airpcap_handle_dead(pcap_t *p _U_)
{
    // 返回 NULL，表示失败
    return (NULL);
}
#endif /* _WIN32 */

static void
pcap_cleanup_dead(pcap_t *p _U_)
{
    // 无需执行任何操作
    /* Nothing to do. */
}

pcap_t *
pcap_open_dead_with_tstamp_precision(int linktype, int snaplen, u_int precision)
{
    pcap_t *p;

    switch (precision) {

    case PCAP_TSTAMP_PRECISION_MICRO:
    case PCAP_TSTAMP_PRECISION_NANO:
        break;

    default:
        /*
         * 这并不重要，但我们没有任何方法来报告特定的错误，所以我们唯一可能的失败是内存分配失败。只需选择微秒精度。
         */
        precision = PCAP_TSTAMP_PRECISION_MICRO;
        break;
    }
    // 分配内存给 pcap_t 结构体
    p = malloc(sizeof(*p));
    if (p == NULL)
        return NULL;
    // 将分配的内存清零
    memset (p, 0, sizeof(*p));
    // 设置快照长度、链路类型和时间戳精度
    p->snapshot = snaplen;
    p->linktype = linktype;
    p->opt.tstamp_precision = precision;
    p->can_set_rfmon_op = pcap_can_set_rfmon_dead;
    p->read_op = pcap_read_dead;
    p->inject_op = pcap_inject_dead;
    p->setfilter_op = pcap_setfilter_dead;
    p->setdirection_op = pcap_setdirection_dead;
    p->set_datalink_op = pcap_set_datalink_dead;
    # 设置获取非阻塞操作为 pcap_getnonblock_dead 函数
    p->getnonblock_op = pcap_getnonblock_dead;
    # 设置设置非阻塞操作为 pcap_setnonblock_dead 函数
    p->setnonblock_op = pcap_setnonblock_dead;
    # 设置统计操作为 pcap_stats_dead 函数
    p->stats_op = pcap_stats_dead;
#ifdef _WIN32
    // 如果是在 Windows 平台下，设置一些操作为“死”操作
    p->stats_ex_op = pcap_stats_ex_dead;
    p->setbuff_op = pcap_setbuff_dead;
    p->setmode_op = pcap_setmode_dead;
    p->setmintocopy_op = pcap_setmintocopy_dead;
    p->getevent_op = pcap_getevent_dead;
    p->oid_get_request_op = pcap_oid_get_request_dead;
    p->oid_set_request_op = pcap_oid_set_request_dead;
    p->sendqueue_transmit_op = pcap_sendqueue_transmit_dead;
    p->setuserbuffer_op = pcap_setuserbuffer_dead;
    p->live_dump_op = pcap_live_dump_dead;
    p->live_dump_ended_op = pcap_live_dump_ended_dead;
    p->get_airpcap_handle_op = pcap_get_airpcap_handle_dead;
#endif
    // 设置“中断循环”操作为“死”操作
    p->breakloop_op = pcap_breakloop_dead;
    // 设置“清理”操作为“死”操作
    p->cleanup_op = pcap_cleanup_dead;

    /*
     * A "dead" pcap_t never requires special BPF code generation.
     */
    // 对于“死” pcap_t，不需要特殊的 BPF 代码生成
    p->bpf_codegen_flags = 0;

    // 激活 pcap_t
    p->activated = 1;
    // 返回激活后的 pcap_t
    return (p);
}

// 使用给定的链路类型和快照长度打开一个“死” pcap_t
pcap_t *
pcap_open_dead(int linktype, int snaplen)
{
    return (pcap_open_dead_with_tstamp_precision(linktype, snaplen,
        PCAP_TSTAMP_PRECISION_MICRO));
}

#ifdef YYDEBUG
/*
 * Set the internal "debug printout" flag for the filter expression parser.
 * The code to print that stuff is present only if YYDEBUG is defined, so
 * the flag, and the routine to set it, are defined only if YYDEBUG is defined.
 *
 * This is intended for libpcap developers, not for general use.
 * If you want to set these in a program, you'll have to declare this
 * routine yourself, with the appropriate DLL import attribute on Windows;
 * it's not declared in any header file, and won't be declared in any
 * header file provided by libpcap.
 */
// 设置过滤表达式解析器的内部“调试打印”标志
PCAP_API void pcap_set_parser_debug(int value);

PCAP_API_DEF void
pcap_set_parser_debug(int value)
{
    // 设置全局变量 pcap_debug 的值
    pcap_debug = value;
}
#endif
```