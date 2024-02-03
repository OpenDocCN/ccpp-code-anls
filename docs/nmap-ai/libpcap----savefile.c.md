# `nmap\libpcap\savefile.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，需要保留以上版权声明和本段文字
 * 在包含二进制代码的发布中，需要在文档或其他提供的材料中保留以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都需要包含以下声明：
 * “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 *
 * savefile.c - 支持离线使用 tcpdump
 *    由 Jeffrey Mogul, DECWRL 提取/创建
 *    由 Steve McCanne, LBL 修改
 *
 * 用于将经过过滤的接收数据包头保存到文件中，然后以后读取它们
 * 文件中的第一条记录包含了机器相关值的保存值，以便我们可以在任何架构上打印转储文件
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>
#ifdef _WIN32
#include <io.h>
#include <fcntl.h>
#endif /* _WIN32 */

#include <errno.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <limits.h> /* for INT_MAX */

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#include "sf-pcap.h"
#include "sf-pcapng.h"
#include "pcap-common.h"
#ifdef _WIN32
/*
 * 在 Windows 上不导出此函数，因为它只能在 WinPcap/Npcap 和使用它的代码都使用通用 CRT 时才能工作；
 * 否则，如果它们使用不同版本的 C 运行时库，WinPcap/Npcap 中的 FILE 结构和使用它的代码中的 FILE 结构可能不同。
 *
 * 相反，pcap/pcap.h 将其定义为一个宏，用于包装 hopen 版本，包装器调用 _fileno() 和 _get_osfhandle() 自己，
 * 以便将适当版本的 CRT 结构转换为 HANDLE（这是操作系统定义的，而不是 CRT 定义的，并且是 Win32 和 Win64 ABI 的一部分）。
 */
static pcap_t *pcap_fopen_offline_with_tstamp_precision(FILE *, u_int, char *);
#endif

/*
 * 在 DOS/Windows 上设置 O_BINARY 有点棘手
 */
#if defined(_WIN32)
  #define SET_BINMODE(f)  _setmode(_fileno(f), _O_BINARY)
#elif defined(MSDOS)
  #if defined(__HIGHC__)
  #define SET_BINMODE(f)  setmode(f, O_BINARY)
  #else
  #define SET_BINMODE(f)  setmode(fileno(f), O_BINARY)
  #endif
#endif

static int
sf_getnonblock(pcap_t *p _U_)
{
    /*
     * 这是一个保存文件，而不是实时捕获文件，所以永远不要说它处于非阻塞模式。
     */
    return (0);
}

static int
sf_setnonblock(pcap_t *p, int nonblock _U_)
{
    /*
     * 这是一个保存文件，而不是实时捕获文件，所以拒绝将其置于非阻塞模式。
     * （如果它是一个管道，它可以被置于非阻塞模式，但这将显着复杂化读取数据包的代码，
     * 因为它必须处理读取部分数据包并保持读取状态。）
     */
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Savefiles cannot be put into non-blocking mode");
    return (-1);
}

static int
sf_cant_set_rfmon(pcap_t *p _U_)
{
    /*
     * 这是一个保存文件，而不是可以捕获的设备，所以永远不要说它支持被置于监控模式。
     */
    return (0);
}

static int
// 从保存文件中获取统计信息，设置错误消息并返回-1
sf_stats(pcap_t *p, struct pcap_stat *ps _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Statistics aren't available from savefiles");
    return (-1);
}

#ifdef _WIN32
// 从保存文件中获取统计信息，设置错误消息并返回NULL
static struct pcap_stat *
sf_stats_ex(pcap_t *p, int *size _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Statistics aren't available from savefiles");
    return (NULL);
}

// 设置保存文件的缓冲区大小，设置错误消息并返回-1
static int
sf_setbuff(pcap_t *p, int dim _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The kernel buffer size cannot be set while reading from a file");
    return (-1);
}

// 设置保存文件的模式，设置错误消息并返回-1
static int
sf_setmode(pcap_t *p, int mode _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "impossible to set mode while reading from a file");
    return (-1);
}

// 设置保存文件的最小拷贝大小，设置错误消息并返回-1
static int
sf_setmintocopy(pcap_t *p, int size _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The mintocopy parameter cannot be set while reading from a file");
    return (-1);
}

// 获取保存文件的事件，设置错误消息并返回INVALID_HANDLE_VALUE
static HANDLE
sf_getevent(pcap_t *pcap)
{
    (void)snprintf(pcap->errbuf, sizeof(pcap->errbuf),
        "The read event cannot be retrieved while reading from a file");
    return (INVALID_HANDLE_VALUE);
}

// 从保存文件中获取OID请求，设置错误消息并返回PCAP_ERROR
static int
sf_oid_get_request(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_,
    size_t *lenp _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID get request cannot be performed on a file");
    return (PCAP_ERROR);
}

// 设置保存文件的OID请求，设置错误消息并返回PCAP_ERROR
static int
sf_oid_set_request(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID set request cannot be performed on a file");
    return (PCAP_ERROR);
}

// 传输发送队列到保存文件，设置错误消息并返回0
static u_int
sf_sendqueue_transmit(pcap_t *p, pcap_send_queue *queue _U_, int sync _U_)
{
    pcap_strlcpy(p->errbuf, "Sending packets isn't supported on savefiles",
        PCAP_ERRBUF_SIZE);
    return (0);
}

// 设置用户缓冲区大小，设置错误消息并返回-1
static int
sf_setuserbuffer(pcap_t *p, int size _U_)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The user buffer cannot be set when reading from a file");
    return (-1);
}
sf_live_dump(pcap_t *p, char *filename _U_, int maxsize _U_, int maxpacks _U_)
{
    // 设置错误信息，指示无法在从文件中读取时执行实时数据包转储
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed when reading from a file");
    // 返回错误代码
    return (-1);
}

static int
sf_live_dump_ended(pcap_t *p, int sync _U_)
{
    // 设置错误信息，指示无法在 pcap_open_dead pcap_t 上执行实时数据包转储
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed on a pcap_open_dead pcap_t");
    // 返回错误代码
    return (-1);
}

static PAirpcapHandle
sf_get_airpcap_handle(pcap_t *pcap _U_)
{
    // 返回空指针
    return (NULL);
}
#endif

static int
sf_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
    // 设置错误信息，指示不支持在保存文件上发送数据包
    pcap_strlcpy(p->errbuf, "Sending packets isn't supported on savefiles",
        PCAP_ERRBUF_SIZE);
    // 返回错误代码
    return (-1);
}

/*
 * 设置方向标志：在转发单个设备上接受哪些数据包？IN、OUT 还是两者都有？
 */
static int
sf_setdirection(pcap_t *p, pcap_direction_t d _U_)
{
    // 设置错误信息，指示不支持在保存文件上设置方向
    snprintf(p->errbuf, sizeof(p->errbuf),
        "Setting direction is not supported on savefiles");
    // 返回错误代码
    return (-1);
}

void
sf_cleanup(pcap_t *p)
{
    // 如果 rfile 不是标准输入流，则关闭它
    if (p->rfile != stdin)
        (void)fclose(p->rfile);
    // 如果 buffer 不为空，则释放它
    if (p->buffer != NULL)
        free(p->buffer);
    // 释放过滤器
    pcap_freecode(&p->fcode);
}

#ifdef _WIN32
/*
 * fopen() 和 _wfopen() 的包装器。
 *
 * 如果我们处于 UTF-8 模式，则将路径名从 UTF-8 映射到 UTF-16LE，并调用 _wfopen()。
 *
 * 如果不是，则使用 fopen()；它将把它视为处于本地代码页中。
 */
FILE *
charset_fopen(const char *path, const char *mode)
{
    wchar_t *utf16_path;
#define MAX_MODE_LEN    16
    wchar_t utf16_mode[MAX_MODE_LEN+1];
    int i;
    char c;
    FILE *fp;
    int save_errno;
    if (pcap_utf_8_mode) {
        /*
         * 如果 pcap_utf_8_mode 为真，则执行以下操作：
         * 将 UTF-8 编码的路径映射到 UTF-16LE 编码。
         * 如果输入字符串中存在无效字符，则操作失败，而不是将其转换为 REPLACEMENT CHARACTER；
         * 后者适用于要显示给用户的字符串，但对于文件名，你只希望尝试打开文件失败。
         */
        utf16_path = cp_to_utf_16le(CP_UTF8, path,
            MB_ERR_INVALID_CHARS);
        if (utf16_path == NULL) {
            /*
             * 出错。假设 errno 已经被设置。
             *
             * XXX - Windows 错误怎么处理？
             */
            return (NULL);
        }

        /*
         * 现在也将模式转换为 UTF-16LE。
         * 我们假设模式是 ASCII，并且很短，所以很容易处理。
         */
        for (i = 0; (c = *mode) != '\0'; i++, mode++) {
            if (c > 0x7F) {
                /* 不是 ASCII 字符；失败并返回 EINVAL。 */
                free(utf16_path);
                errno = EINVAL;
                return (NULL);
            }
            if (i >= MAX_MODE_LEN) {
                /* 模式字符串超过允许的长度。 */
                free(utf16_path);
                errno = EINVAL;
                return (NULL);
            }
            utf16_mode[i] = c;
        }
        utf16_mode[i] = '\0';

        /*
         * 好的，现在我们有了 UTF-16LE 字符串；将它们传递给 _wfopen()。
         */
        fp = _wfopen(utf16_path, utf16_mode);

        /*
         * 确保释放 UTF-16LE 字符串不会覆盖从 _wfopen() 得到的错误代码。
         */
        save_errno = errno;
        free(utf16_path);
        errno = save_errno;

        return (fp);
    } else {
        /*
         * 这将以本地代码页的字符串作为参数。
         */
        return (fopen(path, mode));
    }
}
#endif

pcap_t *
pcap_open_offline_with_tstamp_precision(const char *fname, u_int precision,
                    char *errbuf)
{
    FILE *fp;
    pcap_t *p;

    // 检查文件名是否为空指针
    if (fname == NULL) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "A null pointer was supplied as the file name");
        return (NULL);
    }
    // 如果文件名为标准输入
    if (fname[0] == '-' && fname[1] == '\0')
    {
        // 使用标准输入作为文件指针
        fp = stdin;
        // 如果标准输入未打开
        if (fp == NULL) {
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "The standard input is not open");
            return (NULL);
        }
#if defined(_WIN32) || defined(MSDOS)
        /*
         * 从标准输入读取，因此将其设置为二进制模式，因为保存文件是二进制文件。
         */
        SET_BINMODE(fp);
#endif
    }
    else {
        /*
         * 使用 charset_fopen(); 在 Windows 上，它会测试我们是否处于“本地代码页”或“UTF-8”模式，并相应地处理路径名，在其他平台上，它只是包装了 fopen()。
         *
         * “b”自C90起就受支持，因此*所有* UN*Xes应该支持它，即使它什么也不做。对于 MS-DOS，我们再次需要它。
         */
        fp = charset_fopen(fname, "rb");
        // 如果文件指针为空
        if (fp == NULL) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "%s", fname);
            return (NULL);
        }
    }
    // 使用指定时间戳精度打开离线捕获文件
    p = pcap_fopen_offline_with_tstamp_precision(fp, precision, errbuf);
    // 如果打开失败
    if (p == NULL) {
        // 如果文件指针不是标准输入，则关闭文件指针
        if (fp != stdin)
            fclose(fp);
    }
    return (p);
}

// 打开离线捕获文件
pcap_t *
pcap_open_offline(const char *fname, char *errbuf)
{
    return (pcap_open_offline_with_tstamp_precision(fname,
        PCAP_TSTAMP_PRECISION_MICRO, errbuf));
}

#ifdef _WIN32
// 使用指定时间戳精度打开离线捕获文件
pcap_t* pcap_hopen_offline_with_tstamp_precision(intptr_t osfd, u_int precision,
    char *errbuf)
{
    int fd;
    FILE *file;

    // 从操作系统文件描述符获取文件描述符
    fd = _open_osfhandle(osfd, _O_RDONLY);
    // 如果文件描述符小于0
    if ( fd < 0 )
    {
        // 使用错误码和指定的错误消息格式化错误缓冲区
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno, "_open_osfhandle");
        // 返回空指针
        return NULL;
    }

    // 使用文件描述符和指定的模式打开文件
    file = _fdopen(fd, "rb");
    // 如果文件指针为空
    if ( file == NULL )
    {
        // 使用错误码和指定的错误消息格式化错误缓冲区
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno, "_fdopen");
        // 关闭文件描述符
        _close(fd);
        // 返回空指针
        return NULL;
    }

    // 使用文件指针和指定的时间戳精度打开离线捕获文件
    return pcap_fopen_offline_with_tstamp_precision(file, precision, errbuf);
}

// 使用给定的文件描述符和错误缓冲区打开离线捕获文件，返回 pcap_t 结构指针
pcap_t* pcap_hopen_offline(intptr_t osfd, char *errbuf)
{
    // 调用 pcap_hopen_offline_with_tstamp_precision 函数，传入时间戳精度为微秒
    return pcap_hopen_offline_with_tstamp_precision(osfd, PCAP_TSTAMP_PRECISION_MICRO, errbuf);
}
#endif

/*
 * 给定链路层头部类型和快照长度，返回读取文件时要使用的快照长度；
 * 必须保证它大于 0 且小于等于 INT_MAX。
 *
 * XXX - 我们将其限制为 <= INT_MAX 的唯一原因是为了它适应 p->snapshot，
 * 而 p->snapshot 是有符号的是因为 pcap_snapshot() 返回的是 int 而不是无符号 int。
 */
bpf_u_int32
pcap_adjust_snapshot(bpf_u_int32 linktype, bpf_u_int32 snaplen)
{
    if (snaplen == 0 || snaplen > INT_MAX) {
        /*
         * 错误的快照长度；使用此链路层类型的最大值作为备用。
         *
         * XXX - 我们不会将快照长度限制在 <= INT_MAX 但 > max_snaplen_for_dlt(linktype) 的范围内，
         * 因此捕获文件可能会导致我们分配一个非常大的缓冲区。
         */
        snaplen = max_snaplen_for_dlt(linktype);
    }
    return snaplen;
}

// 定义一个指向 pcap_check_header 和 pcap_ng_check_header 函数的指针数组
static pcap_t *(*check_headers[])(const uint8_t *, FILE *, u_int, char *, int *) = {
    pcap_check_header,
    pcap_ng_check_header
};

// 定义文件类型数量
#define    N_FILE_TYPES    (sizeof check_headers / sizeof check_headers[0])

#ifdef _WIN32
static
#endif
// 使用给定的文件指针和时间戳精度打开离线捕获文件，返回 pcap_t 结构指针
pcap_t *
pcap_fopen_offline_with_tstamp_precision(FILE *fp, u_int precision, char *errbuf)
{
    register pcap_t *p;
    uint8_t magic[4];
    size_t amt_read;
    u_int i;
    int err;

    /*
     * 如果传入的文件指针为 NULL，则失败。
     *
     * 如果我们使用路径名打开，这不应该发生，但如果有错误的代码使用 FILE * 并且没有确保 FILE * 不为 null，这可能会发生。
     */
    if (fp == NULL) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "Null FILE * pointer provided to savefile open routine");
        return (NULL);
    }
    """
    Read the first 4 bytes of the file; the network analyzer dump
    file formats we support (pcap and pcapng), and several other
    formats we might support in the future (such as snoop, DOS and
    Windows Sniffer, and Microsoft Network Monitor) all have magic
    numbers that are unique in their first 4 bytes.
    """
    # 从文件中读取前4个字节；我们支持的网络分析器转储文件格式（pcap和pcapng），以及将来可能支持的其他格式（如snoop、DOS和Windows Sniffer以及Microsoft Network Monitor）在它们的前4个字节中都有唯一的魔术数字。
    amt_read = fread(&magic, 1, sizeof(magic), fp);
    if (amt_read != sizeof(magic)) {
        if (ferror(fp)) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
        } else {
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "truncated dump file; tried to read %zu file header bytes, only got %zu",
                sizeof(magic), amt_read);
        }
        return (NULL);
    }

    """
    Try all file types.
    """
    # 尝试所有文件类型。
    for (i = 0; i < N_FILE_TYPES; i++) {
        p = (*check_headers[i])(magic, fp, precision, errbuf, &err);
        if (p != NULL) {
            """ Yup, that's it. """
            # 是的，就是它。
            goto found;
        }
        if (err) {
            """
            Error trying to read the header.
            """
            # 尝试读取文件头时出错。
            return (NULL);
        }
    }

    """
    Well, who knows what this mess is....
    """
    # 哎呀，谁知道这是什么鬼东西....
    snprintf(errbuf, PCAP_ERRBUF_SIZE, "unknown file format");
    return (NULL);
# 设置指针 p 的 rfile 属性为 fp
    found:
        p->rfile = fp;

    # 只有在 live capture fcode 情况下才需要填充
    p->fddipad = 0;

    # 如果不是在 Windows 和 MSDOS 平台下
    # 可以在大多数平台上对普通文件进行 "select()" 和 "poll()" 操作
    # 在 Windows 上除了套接字以外不能对其他任何东西进行 "select()" 操作
    # 在 Win32 系统上，我们没有 "selectable_fd"
    p->selectable_fd = fileno(fp);

    # 设置 can_set_rfmon_op 属性为 sf_cant_set_rfmon
    p->can_set_rfmon_op = sf_cant_set_rfmon;
    # 设置 read_op 属性为 pcap_offline_read
    p->read_op = pcap_offline_read;
    # 设置 inject_op 属性为 sf_inject
    p->inject_op = sf_inject;
    # 设置 setfilter_op 属性为 install_bpf_program
    p->setfilter_op = install_bpf_program;
    # 设置 setdirection_op 属性为 sf_setdirection
    p->setdirection_op = sf_setdirection;
    # 设置 set_datalink_op 属性为 NULL，因为我们不支持修改链路层头部
    p->set_datalink_op = NULL;
    # 设置 getnonblock_op 属性为 sf_getnonblock
    p->getnonblock_op = sf_getnonblock;
    # 设置 setnonblock_op 属性为 sf_setnonblock
    p->setnonblock_op = sf_setnonblock;
    # 设置 stats_op 属性为 sf_stats
    p->stats_op = sf_stats;
    # 如果在 Windows 上
    # 设置 stats_ex_op 属性为 sf_stats_ex
    # 设置 setbuff_op 属性为 sf_setbuff
    # 设置 setmode_op 属性为 sf_setmode
    # 设置 setmintocopy_op 属性为 sf_setmintocopy
    # 设置 getevent_op 属性为 sf_getevent
    # 设置 oid_get_request_op 属性为 sf_oid_get_request
    # 设置 oid_set_request_op 属性为 sf_oid_set_request
    # 设置 sendqueue_transmit_op 属性为 sf_sendqueue_transmit
    # 设置 setuserbuffer_op 属性为 sf_setuserbuffer
    # 设置 live_dump_op 属性为 sf_live_dump
    # 设置 live_dump_ended_op 属性为 sf_live_dump_ended
    # 设置 get_airpcap_handle_op 属性为 sf_get_airpcap_handle
    p->stats_ex_op = sf_stats_ex;
    p->setbuff_op = sf_setbuff;
    p->setmode_op = sf_setmode;
    p->setmintocopy_op = sf_setmintocopy;
    p->getevent_op = sf_getevent;
    p->oid_get_request_op = sf_oid_get_request;
    p->oid_set_request_op = sf_oid_set_request;
    p->sendqueue_transmit_op = sf_sendqueue_transmit;
    p->setuserbuffer_op = sf_setuserbuffer;
    p->live_dump_op = sf_live_dump;
    p->live_dump_ended_op = sf_live_dump_ended;
    p->get_airpcap_handle_op = sf_get_airpcap_handle;

    # 对于离线捕获，可以使用标准的一次性回调函数来进行 pcap_next()/pcap_next_ex()
    p->oneshot_callback = pcap_oneshot;

    # 默认的 breakloop 操作
    p->breakloop_op = pcap_breakloop_common;

    # 保存文件不需要特殊的 BPF 代码生成
    p->bpf_codegen_flags = 0;

    # 设置 activated 属性为 1
    p->activated = 1;

    # 返回指针 p
    return (p);
}

# 在 Windows 上不需要这个函数
# 我们将 pcap_fopen_offline() 定义为 pcap_hopen_offline() 的包装函数
# 并且不会在这个文件内部调用它，所以它是未使用的
#ifndef _WIN32
pcap_t *
// 打开离线捕获文件，设置时间戳精度为微秒，返回文件指针
pcap_fopen_offline(FILE *fp, char *errbuf)
{
    return (pcap_fopen_offline_with_tstamp_precision(fp,
        PCAP_TSTAMP_PRECISION_MICRO, errbuf));
}
#endif

/*
 * 从捕获文件中读取数据包，并对每个数据包调用回调函数。
 * 如果 cnt > 0，则在读取 'cnt' 个数据包后返回，否则一直读取直到文件结束。
 */
int
pcap_offline_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    struct bpf_insn *fcode;  // BPF 过滤器指令
    int n = 0;  // 数据包计数
    u_char *data;  // 数据包内容

    /*
     * 这个函数可能处理超过 INT_MAX 个数据包，这会导致数据包计数溢出，
     * 使其看起来像一个负数，从而导致我们返回一个看起来像错误的值，
     * 或者溢出回到正数领域，从而导致我们返回一个太低的计数。
     *
     * 因此，如果数据包计数是无限的，我们将其截断为 INT_MAX；
     * 这个函数不应该无限处理数据包，所以这不是一个问题。
     */
    if (PACKET_COUNT_IS_UNLIMITED(cnt))
        cnt = INT_MAX;
}
    for (;;) {
        struct pcap_pkthdr h;
        int status;

        /*
         * Has "pcap_breakloop()" been called?
         * If so, return immediately - if we haven't read any
         * packets, clear the flag and return -2 to indicate
         * that we were told to break out of the loop, otherwise
         * leave the flag set, so that the *next* call will break
         * out of the loop without having read any packets, and
         * return the number of packets we've processed so far.
         */
        // 检查是否调用了 "pcap_breakloop()"
        // 如果是，则立即返回 - 如果我们还没有读取任何数据包，则清除标志并返回 -2，表示我们被告知要跳出循环，否则保持标志设置，这样*下一个*调用将在没有读取任何数据包的情况下跳出循环，并返回到目前为止我们处理的数据包数量。
        if (p->break_loop) {
            if (n == 0) {
                p->break_loop = 0;
                return (-2);
            } else
                return (n);
        }

        status = p->next_packet_op(p, &h, &data);
        if (status < 0) {
            /*
             * Error.  Pass it back to the caller.
             */
            // 出错。将错误传递给调用者。
            return (status);
        }
        if (status == 0) {
            /*
             * EOF.  Nothing more to process;
             */
            // 文件结束。没有更多要处理的内容。
            break;
        }

        /*
         * OK, we've read a packet; run it through the filter
         * and, if it passes, process it.
         */
        // 好的，我们读取了一个数据包；通过过滤器运行它，如果通过，就处理它。
        if ((fcode = p->fcode.bf_insns) == NULL ||
            pcap_filter(fcode, data, h.len, h.caplen)) {
            (*callback)(user, &h, data);
            n++;    /* count the packet */
            if (n >= cnt)
                break;
        }
    }
    /*XXX this breaks semantics tcpslice expects */
    // 这破坏了 tcpslice 期望的语义
    return (n);
# 闭合前面的函数定义
```