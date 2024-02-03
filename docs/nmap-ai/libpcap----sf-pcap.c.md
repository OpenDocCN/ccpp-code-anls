# `nmap\libpcap\sf-pcap.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在满足以下条件的情况下允许使用：(1) 源代码发布保留以上版权声明和本段完整内容，(2) 包含二进制代码的发布在文档或其他提供的材料中保留以上版权声明和本段完整内容，(3) 所有提及此软件特性或使用的广告材料展示以下声明："本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件"，未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的保证，包括但不限于对适销性和特定用途的暗示保证
 *
 * sf-pcap.c - 从 savefile.c 中提取/创建的 libpcap 文件格式特定代码
 *    由 Jeffrey Mogul, DECWRL 进行提取/创建
 *    由 Steve McCanne, LBL 进行修改
 *
 * 用于将经过过滤的接收数据包头保存到文件中，然后以后读取它们
 * 文件中的第一条记录包含机器相关值的保存值，以便我们可以在任何架构上打印转储文件
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
#include "pcap-util.h"

#include "pcap-common.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif
/*
 * 包含 sf-pcap.h 头文件
 */
#include "sf-pcap.h"

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

/*
 * 标准 libpcap 格式
 *
 * 在 rpcap 协议中使用相同的值作为服务器字节顺序的指示，让客户端知道是否需要对一些主机字节顺序的元数据进行字节交换。
 */
#define TCPDUMP_MAGIC        0xa1b2c3d4

/*
 * Alexey Kuznetzov 修改的 libpcap 格式
 */
#define KUZNETZOV_TCPDUMP_MAGIC    0xa1b2cd34

/*
 * 保留给 Francisco Mesquita <francisco.mesquita@radiomovel.pt> 的另一种修改格式
 */
#define FMESQUITA_TCPDUMP_MAGIC    0xa1b234cd

/*
 * Navtel Communcations 的格式，带有纳秒级时间戳，根据 Dumas Hwang <dumas.hwang@navtelcom.com> 的请求
 */
#define NAVTEL_TCPDUMP_MAGIC    0xa12b3c4d

/*
 * 正常的 libpcap 格式，除了秒/纳秒时间戳，根据 Ulf Lamping <ulf.lamping@web.de> 的请求
 */
#define NSEC_TCPDUMP_MAGIC    0xa1b23c4d

/*
 * 在捕获文件的链接类型值的高 6 位中存储有关捕获的信息的机制
 *
 * LT_LINKTYPE_EXT(x) 提取附加信息
 *
 * 其余位用于描述链路层值。LT_LINKTYPE(x) 提取该值。
 */
#define LT_LINKTYPE(x)        ((x) & 0x03FFFFFF)
#define LT_LINKTYPE_EXT(x)    ((x) & 0xFC000000)

/*
 * pcap_next_packet 函数的声明
 */
static int pcap_next_packet(pcap_t *p, struct pcap_pkthdr *hdr, u_char **datap);

#ifdef _WIN32
/*
 * This isn't exported on Windows, because it would only work if both
 * libpcap and the code using it were using the same C runtime; otherwise they
 * would be using different definitions of a FILE structure.
 *
 * Instead we define this as a macro in pcap/pcap.h that wraps the hopen
 * version that we do export, passing it a raw OS HANDLE, as defined by the
 * Win32 / Win64 ABI, obtained from the _fileno() and _get_osfhandle()
 * functions of the appropriate CRT.
 */
static pcap_dumper_t *pcap_dump_fopen(pcap_t *p, FILE *f);
#endif /* _WIN32 */

/*
 * Private data for reading pcap savefiles.
 */
typedef enum {
    NOT_SWAPPED,
    SWAPPED,
    MAYBE_SWAPPED
} swapped_type_t;

typedef enum {
    PASS_THROUGH,
    SCALE_UP,
    SCALE_DOWN
} tstamp_scale_type_t;

struct pcap_sf {
    size_t hdrsize;
    swapped_type_t lengths_swapped;
    tstamp_scale_type_t scale_type;
};

/*
 * Check whether this is a pcap savefile and, if it is, extract the
 * relevant information from the header.
 */
pcap_t *
pcap_check_header(const uint8_t *magic, FILE *fp, u_int precision, char *errbuf,
          int *err)
{
    bpf_u_int32 magic_int;
    struct pcap_file_header hdr;
    size_t amt_read;
    pcap_t *p;
    int swapped = 0;
    struct pcap_sf *ps;

    /*
     * Assume no read errors.
     */
    *err = 0;

    /*
     * Check whether the first 4 bytes of the file are the magic
     * number for a pcap savefile, or for a byte-swapped pcap
     * savefile.
     */
    memcpy(&magic_int, magic, sizeof(magic_int));
    if (magic_int != TCPDUMP_MAGIC &&
        magic_int != KUZNETZOV_TCPDUMP_MAGIC &&
        magic_int != NSEC_TCPDUMP_MAGIC) {
        magic_int = SWAPLONG(magic_int);
        if (magic_int != TCPDUMP_MAGIC &&
            magic_int != KUZNETZOV_TCPDUMP_MAGIC &&
            magic_int != NSEC_TCPDUMP_MAGIC)
            return (NULL);    /* nope */
        swapped = 1;
    }
*/
    /*
     * They are.  Put the magic number in the header, and read
     * the rest of the header.
     */
    // 将魔术数字放入头部，并读取头部的其余部分
    hdr.magic = magic_int;
    // 从文件中读取头部的剩余部分
    amt_read = fread(((char *)&hdr) + sizeof hdr.magic, 1,
        sizeof(hdr) - sizeof(hdr.magic), fp);
    // 如果读取的字节数不等于头部剩余部分的字节数
    if (amt_read != sizeof(hdr) - sizeof(hdr.magic)) {
        // 如果发生了文件读取错误
        if (ferror(fp)) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
        } else {
            // 如果文件被截断
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "truncated dump file; tried to read %zu file header bytes, only got %zu",
                sizeof(hdr), amt_read);
        }
        *err = 1;
        return (NULL);
    }

    /*
     * If it's a byte-swapped capture file, byte-swap the header.
     */
    // 如果是字节交换的捕获文件，则对头部进行字节交换
    if (swapped) {
        hdr.version_major = SWAPSHORT(hdr.version_major);
        hdr.version_minor = SWAPSHORT(hdr.version_minor);
        hdr.thiszone = SWAPLONG(hdr.thiszone);
        hdr.sigfigs = SWAPLONG(hdr.sigfigs);
        hdr.snaplen = SWAPLONG(hdr.snaplen);
        hdr.linktype = SWAPLONG(hdr.linktype);
    }

    // 如果主版本号小于PCAP_VERSION_MAJOR
    if (hdr.version_major < PCAP_VERSION_MAJOR) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "archaic pcap savefile format");
        *err = 1;
        return (NULL);
    }

    /*
     * currently only versions 2.[0-4] are supported with
     * the exception of 543.0 for DG/UX tcpdump.
     */
    // 如果版本号不在支持的范围内
    if (! ((hdr.version_major == PCAP_VERSION_MAJOR &&
        hdr.version_minor <= PCAP_VERSION_MINOR) ||
           (hdr.version_major == 543 &&
        hdr.version_minor == 0))) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
             "unsupported pcap savefile version %u.%u",
             hdr.version_major, hdr.version_minor);
        *err = 1;
        return NULL;
    }

    /*
     * OK, this is a good pcap file.
     * Allocate a pcap_t for it.
     */
    // 分配一个pcap_t结构体
    p = PCAP_OPEN_OFFLINE_COMMON(errbuf, struct pcap_sf);
    # 如果分配内存失败，则设置错误标志并返回空指针
    if (p == NULL) {
        *err = 1;
        return (NULL);
    }
    # 设置数据包交换标志
    p->swapped = swapped;
    # 设置主版本号
    p->version_major = hdr.version_major;
    # 设置次版本号
    p->version_minor = hdr.version_minor;
    # 将链路类型转换为数据链路类型
    p->linktype = linktype_to_dlt(LT_LINKTYPE(hdr.linktype));
    # 设置链路类型扩展
    p->linktype_ext = LT_LINKTYPE_EXT(hdr.linktype);
    # 调整快照长度
    p->snapshot = pcap_adjust_snapshot(p->linktype, hdr.snaplen);

    # 设置下一个数据包操作为 pcap_next_packet
    p->next_packet_op = pcap_next_packet;

    # 获取私有数据结构
    ps = p->priv;

    # 设置时间戳精度
    p->opt.tstamp_precision = precision;

    # 根据时间戳精度进行不同的处理
    switch (precision) {

    case PCAP_TSTAMP_PRECISION_MICRO:
        if (magic_int == NSEC_TCPDUMP_MAGIC) {
            # 如果文件的时间戳是纳秒，而用户需要微秒，则进行精度缩减
            ps->scale_type = SCALE_DOWN;
        } else {
            # 如果文件的时间戳是微秒，而用户也需要微秒，则不做处理
            ps->scale_type = PASS_THROUGH;
        }
        break;

    case PCAP_TSTAMP_PRECISION_NANO:
        if (magic_int == NSEC_TCPDUMP_MAGIC) {
            # 如果文件的时间戳是纳秒，而用户也需要纳秒，则不做处理
            ps->scale_type = PASS_THROUGH;
        } else {
            # 如果文件的时间戳是微秒，而用户需要纳秒，则进行精度放大
            ps->scale_type = SCALE_UP;
        }
        break;

    default:
        # 如果时间戳精度未知，则设置错误信息，释放内存，设置错误标志并返回空指针
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "unknown time stamp resolution %u", precision);
        free(p);
        *err = 1;
        return (NULL);
    }
    /*
     * 根据版本号判断数据包头部中的 caplen 和 len 字段是否需要交换
     * 2.3 版本之后的文件头部需要交换字段，但有些文件的头部版本号是 2.3，但字段未交换
     * 另外，DG/UX tcpdump 写出的文件版本号为 543.0，并且 caplen 和 len 字段顺序与 2.3 版本之前的顺序相同
     */
    switch (hdr.version_major) {

    case 2:
        if (hdr.version_minor < 3)
            ps->lengths_swapped = SWAPPED;
        else if (hdr.version_minor == 3)
            ps->lengths_swapped = MAYBE_SWAPPED;
        else
            ps->lengths_swapped = NOT_SWAPPED;
        break;

    case 543:
        ps->lengths_swapped = SWAPPED;
        break;

    default:
        ps->lengths_swapped = NOT_SWAPPED;
        break;
    }

    } else
        ps->hdrsize = sizeof(struct pcap_sf_pkthdr);

    /*
     * 为数据包数据分配缓冲区
     * 选择文件的快照长度和 2K 字节中的最小值；
     * 这应该足够大多数网络数据包 - 如果需要，我们会扩大它
     * 这样，我们不会因为快照长度很大而分配大块内存，因为快照长度可能大于最大数据包的大小
     */
    p->bufsize = p->snapshot;
    if (p->bufsize > 2048)
        p->bufsize = 2048;
    p->buffer = malloc(p->bufsize);
    if (p->buffer == NULL) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "out of memory");
        free(p);
        *err = 1;
        return (NULL);
    }

    p->cleanup_op = sf_cleanup;

    return (p);
/*
 * Grow the packet buffer to the specified size.
 */
static int
grow_buffer(pcap_t *p, u_int bufsize)
{
    // 为数据包缓冲区增大指定大小的空间
    void *bigger_buffer;

    // 重新分配更大的内存空间
    bigger_buffer = realloc(p->buffer, bufsize);
    // 如果内存分配失败
    if (bigger_buffer == NULL) {
        // 将错误信息写入错误缓冲区
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "out of memory");
        return (0);
    }
    // 更新缓冲区指针和大小
    p->buffer = bigger_buffer;
    p->bufsize = bufsize;
    return (1);
}

/*
 * Read and return the next packet from the savefile.  Return the header
 * in hdr and a pointer to the contents in data.  Return 1 on success, 0
 * if there were no more packets, and -1 on an error.
 */
static int
pcap_next_packet(pcap_t *p, struct pcap_pkthdr *hdr, u_char **data)
{
    // 获取私有数据结构
    struct pcap_sf *ps = p->priv;
    // 定义用于读取数据包头的结构
    struct pcap_sf_patched_pkthdr sf_hdr;
    // 获取文件指针
    FILE *fp = p->rfile;
    // 读取的字节数
    size_t amt_read;
    // 用于存储时间戳的变量
    bpf_u_int32 t;

    /*
     * Read the packet header; the structure we use as a buffer
     * is the longer structure for files generated by the patched
     * libpcap, but if the file has the magic number for an
     * unpatched libpcap we only read as many bytes as the regular
     * header has.
     */
    // 读取数据包头部
    amt_read = fread(&sf_hdr, 1, ps->hdrsize, fp);
    // 如果读取的字节数不等于预期的字节数
    if (amt_read != ps->hdrsize) {
        // 如果发生了读取错误
        if (ferror(fp)) {
            // 将错误信息写入错误缓冲区
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
            return (-1);
        } else {
            // 如果读取的字节数不为0
            if (amt_read != 0) {
                // 将错误信息写入错误缓冲区
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "truncated dump file; tried to read %zu header bytes, only got %zu",
                    ps->hdrsize, amt_read);
                return (-1);
            }
            // 文件结束
            return (0);
        }
    }

    // 如果文件的字节序是反的
    if (p->swapped) {
        // 将字节序转换为本地字节序
        hdr->caplen = SWAPLONG(sf_hdr.caplen);
        hdr->len = SWAPLONG(sf_hdr.len);
        hdr->ts.tv_sec = SWAPLONG(sf_hdr.ts.tv_sec);
        hdr->ts.tv_usec = SWAPLONG(sf_hdr.ts.tv_usec);
    } else {
        // 如果条件不成立，将数据包头部的长度和时间戳赋值为抓取到的数据包头部的长度和时间戳
        hdr->caplen = sf_hdr.caplen;
        hdr->len = sf_hdr.len;
        hdr->ts.tv_sec = sf_hdr.ts.tv_sec;
        hdr->ts.tv_usec = sf_hdr.ts.tv_usec;
    }

    switch (ps->scale_type) {

    case PASS_THROUGH:
        /*
         * 只是简单地通过时间戳。
         */
        break;

    case SCALE_UP:
        /*
         * 文件的时间戳是微秒，用户想要纳秒；进行转换。
         */
        hdr->ts.tv_usec = hdr->ts.tv_usec * 1000;
        break;

    case SCALE_DOWN:
        /*
         * 文件的时间戳是纳秒，用户想要微秒；进行转换。
         */
        hdr->ts.tv_usec = hdr->ts.tv_usec / 1000;
        break;
    }

    /* 如果需要，交换 caplen 和 len 字段的值。 */
    switch (ps->lengths_swapped) {

    case NOT_SWAPPED:
        break;

    case MAYBE_SWAPPED:
        if (hdr->caplen <= hdr->len) {
            /*
             * 抓取长度小于等于实际长度，
             * 所以可能它们没有被交换。
             */
            break;
        }
        /* 继续执行下面的代码 */

    case SWAPPED:
        t = hdr->caplen;
        hdr->caplen = hdr->len;
        hdr->len = t;
        break;
    }

    /*
     * 数据包是否比我们认为的合理大小还要大？
     */
    # 如果捕获的数据包长度大于当前链路类型的最大快照长度
    if (hdr->caplen > max_snaplen_for_dlt(p->linktype)) {
        # 是的，这可能是一个损坏或模糊的文件。
        # 它是否大于快照长度？
        # （如果它不大于我们认为合理的最大长度，我们不会将其视为错误；参见下文。）
        
        # 如果捕获的数据包长度大于快照长度
        if (hdr->caplen > (bpf_u_int32)p->snapshot) {
            # 将错误信息写入错误缓冲区，指示捕获的数据包长度大于快照长度
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "invalid packet capture length %u, bigger than "
                "snaplen of %d", hdr->caplen, p->snapshot);
        } else {
            # 将错误信息写入错误缓冲区，指示捕获的数据包长度大于链路类型的最大快照长度
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "invalid packet capture length %u, bigger than "
                "maximum of %u", hdr->caplen,
                max_snaplen_for_dlt(p->linktype));
        }
        # 返回错误代码
        return (-1);
    }
    } else {
        /*
         * 如果数据包在文件的快照长度内。
         */
        if (hdr->caplen > p->bufsize) {
            /*
             * 将缓冲区大小增长到下一个2的幂，或者快照长度，取两者中较小的一个。
             */
            u_int new_bufsize;

            new_bufsize = hdr->caplen;
            /*
             * 使用位操作技巧将 new_bufsize 向上取最近的2的幂
             * 参考链接：https://graphics.stanford.edu/~seander/bithacks.html#RoundUpPowerOf2
             */
            new_bufsize--;
            new_bufsize |= new_bufsize >> 1;
            new_bufsize |= new_bufsize >> 2;
            new_bufsize |= new_bufsize >> 4;
            new_bufsize |= new_bufsize >> 8;
            new_bufsize |= new_bufsize >> 16;
            new_bufsize++;

            if (new_bufsize > (u_int)p->snapshot)
                new_bufsize = p->snapshot;

            if (!grow_buffer(p, new_bufsize))
                return (-1);
        }

        /* 读取数据包本身 */
        amt_read = fread(p->buffer, 1, hdr->caplen, fp);
        if (amt_read != hdr->caplen) {
            if (ferror(fp)) {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "error reading dump file");
            } else {
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "truncated dump file; tried to read %u captured bytes, only got %zu",
                    hdr->caplen, amt_read);
            }
            return (-1);
        }
    }
    *data = p->buffer;

    pcap_post_process(p->linktype, p->swapped, hdr, *data);

    return (1);
# 写入 pcap 文件头部信息
static int
sf_write_header(pcap_t *p, FILE *fp, int linktype, int snaplen)
{
    # 创建 pcap 文件头结构体
    struct pcap_file_header hdr;

    # 根据时间戳精度设置魔数
    hdr.magic = p->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO ? NSEC_TCPDUMP_MAGIC : TCPDUMP_MAGIC;
    # 设置主版本号
    hdr.version_major = PCAP_VERSION_MAJOR;
    # 设置次版本号
    hdr.version_minor = PCAP_VERSION_MINOR;

    # 设置时区偏移为0
    hdr.thiszone = 0;
    # 设置时间戳精度为0
    hdr.sigfigs = 0;
    # 设置数据包捕获长度
    hdr.snaplen = snaplen;
    # 设置链路类型
    hdr.linktype = linktype;

    # 将文件头信息写入文件
    if (fwrite((char *)&hdr, sizeof(hdr), 1, fp) != 1)
        return (-1);

    return (0);
}

'''
'''
# 将数据包写入初始化的转储文件
void
pcap_dump(u_char *user, const struct pcap_pkthdr *h, const u_char *sp)
{
    register FILE *f;
    # 创建 pcap_sf_pkthdr 结构体
    struct pcap_sf_pkthdr sf_hdr;

    # 将用户数据转换为文件指针
    f = (FILE *)user;
    '''
    # 如果输出文件句柄处于错误状态，则不写入任何内容
    if (ferror(f))
        return;
    '''
    # 如果时间戳超过2038-01-19 03:14:07 UTC，则切换到 pcapng 格式
    sf_hdr.ts.tv_sec  = (bpf_int32)h->ts.tv_sec;
    sf_hdr.ts.tv_usec = (bpf_int32)h->ts.tv_usec;
    # 设置 sf_hdr 结构体的 caplen 字段为 h 结构体的 caplen 字段的值
    sf_hdr.caplen     = h->caplen;
    # 设置 sf_hdr 结构体的 len 字段为 h 结构体的 len 字段的值
    sf_hdr.len        = h->len;
    '''
     * 只有在能够正确写入头部的情况下，我们才会写入数据包。
     *
     * 这并不能防止输出数据包损坏，如果由于某种原因我们没有完整地写入数据，我们也没有任何
     * 方法来设置 ferror() 来防止未来的写入尝试，但总比什么都不做要好。
     '''
    # 如果成功写入 sf_hdr 结构体的大小的数据到文件 f 中
    if (fwrite(&sf_hdr, sizeof(sf_hdr), 1, f) == 1) {
        # 写入 h 结构体的 caplen 字段大小的数据到文件 f 中
        (void)fwrite(sp, h->caplen, 1, f);
    }
}

static pcap_dumper_t *
pcap_setup_dump(pcap_t *p, int linktype, FILE *f, const char *fname)
{

#if defined(_WIN32) || defined(MSDOS)
    /*
     * 如果我们要写入标准输出，将其设置为二进制模式，因为保存文件是二进制文件。
     *
     * 否则，关闭缓冲。
     * XXX - 为什么？为什么不在标准输出上关闭缓冲？
     */
    if (f == stdout)
        SET_BINMODE(f);
    else
        setvbuf(f, NULL, _IONBF, 0);
#endif
    // 写入文件头
    if (sf_write_header(p, f, linktype, p->snapshot) == -1) {
        // 如果写入文件头失败，设置错误消息并返回空指针
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't write to %s", fname);
        if (f != stdout)
            (void)fclose(f);
        return (NULL);
    }
    // 返回文件指针类型的对象
    return ((pcap_dumper_t *)f);
}

/*
 * 初始化，使得 sf_write() 输出到名为 'fname' 的文件。
 */
pcap_dumper_t *
pcap_dump_open(pcap_t *p, const char *fname)
{
    FILE *f;
    int linktype;

    /*
     * 如果此 pcap_t 尚未激活，则它没有链路层类型，因此我们无法使用它。
     */
    if (!p->activated) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "%s: not-yet-activated pcap_t passed to pcap_dump_open",
            fname);
        return (NULL);
    }
    linktype = dlt_to_linktype(p->linktype);
    if (linktype == -1) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "%s: link-layer type %d isn't supported in savefiles",
            fname, p->linktype);
        return (NULL);
    }
    linktype |= p->linktype_ext;

    if (fname == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "A null pointer was supplied as the file name");
        return NULL;
    }
    if (fname[0] == '-' && fname[1] == '\0') {
        f = stdout;
        fname = "standard output";
    } else {
        /*
         * 如果是在 C90 标准之后，"b" 模式是被支持的，所以 *所有* 的 UN*X 系统应该支持它，尽管它不起作用。
         * 在 Windows 上是必须的，因为文件是一个二进制文件，必须以二进制模式写入。
         */
        // 以写入二进制模式打开文件
        f = charset_fopen(fname, "wb");
        // 如果文件打开失败
        if (f == NULL) {
            // 格式化错误消息，将错误信息存储到错误缓冲区中
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "%s", fname);
            // 返回空指针
            return (NULL);
        }
    }
    // 设置捕获数据包的转储
    return (pcap_setup_dump(p, linktype, f, fname));
#ifdef _WIN32
/*
 * Initialize so that sf_write() will output to a stream wrapping the given raw
 * OS file HANDLE.
 */
pcap_dumper_t *
pcap_dump_hopen(pcap_t *p, intptr_t osfd)
{
    int fd;
    FILE *file;

    // 将操作系统文件句柄包装成流，以便 sf_write() 输出到该流
    fd = _open_osfhandle(osfd, _O_APPEND);
    if (fd < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "_open_osfhandle");
        return NULL;
    }

    // 用二进制模式打开文件
    file = _fdopen(fd, "wb");
    if (file == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "_fdopen");
        _close(fd);
        return NULL;
    }

    return pcap_dump_fopen(p, file);
}
#endif /* _WIN32 */

/*
 * Initialize so that sf_write() will output to the given stream.
 */
#ifdef _WIN32
static
#endif /* _WIN32 */
pcap_dumper_t *
pcap_dump_fopen(pcap_t *p, FILE *f)
{
    int linktype;

    // 将数据链路类型转换为链接类型
    linktype = dlt_to_linktype(p->linktype);
    if (linktype == -1) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "stream: link-layer type %d isn't supported in savefiles",
            p->linktype);
        return (NULL);
    }
    linktype |= p->linktype_ext;

    return (pcap_setup_dump(p, linktype, f, "stream"));
}

pcap_dumper_t *
pcap_dump_open_append(pcap_t *p, const char *fname)
{
    FILE *f;
    int linktype;
    size_t amt_read;
    struct pcap_file_header ph;

    // 将数据链路类型转换为链接类型
    linktype = dlt_to_linktype(p->linktype);
    if (linktype == -1) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "%s: link-layer type %d isn't supported in savefiles",
            fname, linktype);
        return (NULL);
    }

    // 检查文件名是否为空指针
    if (fname == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "A null pointer was supplied as the file name");
        return NULL;
    }
    // 如果文件名为 "-"，则输出到标准输出
    if (fname[0] == '-' && fname[1] == '\0')
        return (pcap_setup_dump(p, linktype, stdout, "standard output"));
}
    /*
     * 以 "ab+" 模式打开文件，如果文件存在则不截断，不存在则创建。
     * 所有写操作都在文件末尾进行，但允许从文件中的任意位置进行读取。
     * 这是我们需要的，因为我们需要从文件开头读取，查看是否已经有头部和数据包，或者没有。
     *
     * "b" 在 C90 标准中被支持，所以所有的 UN*X 系统应该支持它，即使它什么也不做。
     * 在 Windows 上是必须的，因为文件是二进制文件，必须以二进制模式进行读取。
     */
    f = charset_fopen(fname, "ab+");
    if (f == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "%s", fname);
        return (NULL);
    }

    /*
     * 尝试读取一个 pcap 头部。
     *
     * 我们不假设文件在打开后会立即定位在开头 - 我们将其定位到开头。
     * ISO C 规定在以追加模式打开后，文件位置指示器是在文件开头还是文件末尾是实现定义的，
     * 并且从 Single UNIX Specification 或 Microsoft 文档中并不清楚在符合 SUS 标准的系统或 Windows 上如何工作。
     */
    if (fseek(f, 0, SEEK_SET) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't seek to the beginning of %s", fname);
        (void)fclose(f);
        return (NULL);
    }
    amt_read = fread(&ph, 1, sizeof (ph), f);
    # 如果读取的字节数不等于文件头的大小
    if (amt_read != sizeof (ph)) {
        # 如果发生文件读取错误
        if (ferror(f)) {
            # 格式化错误消息，将错误信息存储到错误缓冲区中
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "%s", fname);
            # 关闭文件
            (void)fclose(f);
            # 返回空指针
            return (NULL);
        # 如果到达文件末尾但是已经读取了部分数据
        } else if (feof(f) && amt_read > 0) {
            # 格式化错误消息，指示文件头被截断
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "%s: truncated pcap file header", fname);
            # 关闭文件
            (void)fclose(f);
            # 返回空指针
            return (NULL);
        }
    }
#if defined(_WIN32) || defined(MSDOS)
    /*
     * 如果定义了_WIN32或者MSDOS，我们关闭缓冲。
     * XXX - 为什么？为什么标准输出没有关闭缓冲？
     */
    setvbuf(f, NULL, _IONBF, 0);
#endif

    /*
     * 如果已经存在一个头部并且：
     *
     *    它不是适合本机器的 pcap 文件的头部
     *    和正确的字节顺序；
     *
     *    链路层头部类型不匹配；
     *
     *    快照长度不匹配；
     *
     * 返回错误。
     */
    } else {
        /*
         * 头部不存在；尝试写入头部。
         */
        if (sf_write_header(p, f, linktype, p->snapshot) == -1) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "Can't write to %s", fname);
            (void)fclose(f);
            return (NULL);
        }
    }

    /*
     * 从文件末尾开始写入。
     *
     * XXX - 这应该是不必要的，因为我们是以追加模式打开文件，
     * 并且 ISO C 规定在该模式下所有写入都在文件末尾进行。
     */
    if (fseek(f, 0, SEEK_END) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't seek to the end of %s", fname);
        (void)fclose(f);
        return (NULL);
    }
    return ((pcap_dumper_t *)f);
}

FILE *
pcap_dump_file(pcap_dumper_t *p)
{
    return ((FILE *)p);
}

long
pcap_dump_ftell(pcap_dumper_t *p)
{
    return (ftell((FILE *)p));
}

#if defined(HAVE_FSEEKO)
/*
 * 我们有 fseeko()，所以我们有 ftello()。
 * 如果我们有大文件支持（文件大于2^31-1字节），
 * ftello()将给我们一个超过32位的当前文件位置。
 */
int64_t
pcap_dump_ftell64(pcap_dumper_t *p)
{
    return (ftello((FILE *)p));
}
#elif defined(_MSC_VER)
/*
 * 我们有 Visual Studio；我们只支持2005年及以后的版本，所以我们有 _ftelli64()。
 */
int64_t
pcap_dump_ftell64(pcap_dumper_t *p)
{
    return (_ftelli64((FILE *)p));
}
#else
/*
 * 如果没有 ftello() 或 _ftelli64()，则回退到使用 ftell()。
 * 如果 long 是 64 位，那么 ftell() 应该足够了，
 * 或者这可能是一个没有大文件支持的旧的 32 位 UN*X 系统，
 * 这意味着尝试写入大于 2^31-1 的文件可能会出错，所以无论如何都不重要。
 *
 * XXX - MinGW 怎么办？
 */
int64_t
pcap_dump_ftell64(pcap_dumper_t *p)
{
    return (ftell((FILE *)p));
}
#endif

int
pcap_dump_flush(pcap_dumper_t *p)
{

    if (fflush((FILE *)p) == EOF)
        return (-1);
    else
        return (0);
}

void
pcap_dump_close(pcap_dumper_t *p)
{

#ifdef notyet
    if (ferror((FILE *)p))
        return-an-error;
    /* XXX 应该也检查 fclose() 的返回值 */
#endif
    (void)fclose((FILE *)p);
}
```