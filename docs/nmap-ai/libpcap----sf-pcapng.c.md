# `nmap\libpcap\sf-pcapng.c`

```
/*
 * 版权声明，版权所有
 * 1993年至1997年，加利福尼亚大学理事会保留所有权利
 *
 * 允许在源代码和二进制形式下进行重新分发和使用，无论是否进行修改，只要满足以下条件：
 * (1) 源代码分发中保留以上版权声明和本段文字的完整性
 * (2) 包含二进制代码的分发中，在文档或其他提供的材料中完整包含以上版权声明和本段文字
 * (3) 所有提及此软件特性或使用的广告材料显示以下声明：
 * “本产品包括由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。
 *
 * sf-pcapng.c - 从savefile.c中提取的pcapng文件格式特定代码
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap/pcap-inttypes.h>

#include <errno.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "pcap-int.h"
#include "pcap-util.h"

#include "pcap-common.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#include "sf-pcapng.h"

/*
 * 块类型
 */

/*
 * 所有块开头的公共部分
 */
struct block_header {
    bpf_u_int32    block_type;
    bpf_u_int32    total_length;
};

/*
 * 所有块结尾的公共部分
 */
struct block_trailer {
    bpf_u_int32    total_length;
};

/*
 * 公共选项
 */
#define OPT_ENDOFOPT    0    /* 选项结束 */
#define OPT_COMMENT    1    /* 注释字符串 */
/*
 * Option header.
 */
struct option_header {
    u_short        option_code;      // 选项代码
    u_short        option_length;    // 选项长度
};

/*
 * Structures for the part of each block type following the common
 * part.
 */

/*
 * Section Header Block.
 */
#define BT_SHB            0x0A0D0D0A    // 区段头块标识
#define BT_SHB_INSANE_MAX       1024U*1024U*1U  /* 1MB should be enough */    // 区段头块最大长度

struct section_header_block {
    bpf_u_int32    byte_order_magic;    // 字节顺序魔术值
    u_short        major_version;    // 主版本号
    u_short        minor_version;    // 次版本号
    uint64_t    section_length;    // 区段长度
    /* followed by options and trailer */
};

/*
 * Byte-order magic value.
 */
#define BYTE_ORDER_MAGIC    0x1A2B3C4D    // 字节顺序魔术值

/*
 * Current version number.  If major_version isn't PCAP_NG_VERSION_MAJOR,
 * or if minor_version isn't PCAP_NG_VERSION_MINOR or 2, that means that
 * this code can't read the file.
 */
#define PCAP_NG_VERSION_MAJOR    1    // 当前主版本号
#define PCAP_NG_VERSION_MINOR    0    // 当前次版本号

/*
 * Interface Description Block.
 */
#define BT_IDB            0x00000001    // 接口描述块标识

struct interface_description_block {
    u_short        linktype;    // 链路类型
    u_short        reserved;    // 保留字段
    bpf_u_int32    snaplen;    // 抓包长度
    /* followed by options and trailer */
};

/*
 * Options in the IDB.
 */
#define IF_NAME        2    // 接口名称字符串
#define IF_DESCRIPTION    3    // 接口描述字符串
#define IF_IPV4ADDR    4    // 接口的IPv4地址和子网掩码
#define IF_IPV6ADDR    5    // 接口的IPv6地址和前缀长度
#define IF_MACADDR    6    // 接口的MAC地址
#define IF_EUIADDR    7    // 接口的EUI地址
#define IF_SPEED    8    // 接口的速度，以比特/秒为单位
#define IF_TSRESOL    9    // 接口的时间戳分辨率
#define IF_TZONE    10    // 接口的时区
#define IF_FILTER    11    // 在接口上捕获时使用的过滤器
#define IF_OS        12    // 在此接口上进行捕获的操作系统字符串
#define IF_FCSLEN    13    // 此接口的FCS长度
#define IF_TSOFFSET    14    /* time stamp offset for this interface */
// 定义接口的时间戳偏移量为14

/*
 * Enhanced Packet Block.
 */
#define BT_EPB            0x00000006
// 定义增强数据包块的标识为0x00000006

struct enhanced_packet_block {
    bpf_u_int32    interface_id;
    bpf_u_int32    timestamp_high;
    bpf_u_int32    timestamp_low;
    bpf_u_int32    caplen;
    bpf_u_int32    len;
    /* followed by packet data, options, and trailer */
};
// 定义增强数据包块的结构，包含接口ID、时间戳高位、时间戳低位、捕获长度、总长度等字段

/*
 * Simple Packet Block.
 */
#define BT_SPB            0x00000003
// 定义简单数据包块的标识为0x00000003

struct simple_packet_block {
    bpf_u_int32    len;
    /* followed by packet data and trailer */
};
// 定义简单数据包块的结构，包含长度字段，后面跟着数据包和尾部信息

/*
 * Packet Block.
 */
#define BT_PB            0x00000002
// 定义数据包块的标识为0x00000002

struct packet_block {
    u_short        interface_id;
    u_short        drops_count;
    bpf_u_int32    timestamp_high;
    bpf_u_int32    timestamp_low;
    bpf_u_int32    caplen;
    bpf_u_int32    len;
    /* followed by packet data, options, and trailer */
};
// 定义数据包块的结构，包含接口ID、丢包计数、时间戳高位、时间戳低位、捕获长度、总长度等字段

/*
 * Block cursor - used when processing the contents of a block.
 * Contains a pointer into the data being processed and a count
 * of bytes remaining in the block.
 */
struct block_cursor {
    u_char        *data;
    size_t        data_remaining;
    bpf_u_int32    block_type;
};
// 定义块游标的结构，用于处理块内容，包含指向正在处理的数据的指针和块中剩余字节数的计数

typedef enum {
    PASS_THROUGH,
    SCALE_UP_DEC,
    SCALE_DOWN_DEC,
    SCALE_UP_BIN,
    SCALE_DOWN_BIN
} tstamp_scale_type_t;
// 定义时间戳缩放类型的枚举

/*
 * Per-interface information.
 */
struct pcap_ng_if {
    uint32_t snaplen;        /* snapshot length */
    uint64_t tsresol;        /* time stamp resolution */
    tstamp_scale_type_t scale_type;    /* how to scale */
    uint64_t scale_factor;        /* time stamp scale factor for power-of-10 tsresol */
    uint64_t tsoffset;        /* time stamp offset */
};
// 定义每个接口的信息结构，包含快照长度、时间戳分辨率、时间戳缩放类型、时间戳缩放因子、时间戳偏移量等字段
/*
 * Per-pcap_t private data.
 *
 * max_blocksize is the maximum size of a block that we'll accept.  We
 * reject blocks bigger than this, so we don't consume too much memory
 * with a truly huge block.  It can change as we see IDBs with different
 * link-layer header types.  (Currently, we don't support IDBs with
 * different link-layer header types, but we will support it in the
 * future, when we offer file-reading APIs that support it.)
 *
 * XXX - that's an issue on ILP32 platforms, where the maximum block
 * size of 2^31-1 would eat all but one byte of the entire address space.
 * It's less of an issue on ILP64/LLP64 platforms, but the actual size
 * of the address space may be limited by 1) the number of *significant*
 * address bits (currently, x86-64 only supports 48 bits of address), 2)
 * any limitations imposed by the operating system; 3) any limitations
 * imposed by the amount of available backing store for anonymous pages,
 * so we impose a limit regardless of the size of a pointer.
 */
struct pcap_ng_sf {
    uint64_t user_tsresol;        /* time stamp resolution requested by the user */
    u_int max_blocksize;        /* don't grow buffer size past this */
    bpf_u_int32 ifcount;        /* number of interfaces seen in this capture */
    bpf_u_int32 ifaces_size;    /* size of array below */
    struct pcap_ng_if *ifaces;    /* array of interface information */
};

/*
 * The maximum block size we start with; we use an arbitrary value of
 * 16 MiB.
 */
#define INITIAL_MAX_BLOCKSIZE    (16*1024*1024)

/*
 * Maximum block size for a given maximum snapshot length; we define it
 * as the size of an EPB with a max_snaplen-sized packet and 128KB of
 * options.
 */
#define MAX_BLOCKSIZE_FOR_SNAPLEN(max_snaplen) \
    (sizeof (struct block_header) + \
     sizeof (struct enhanced_packet_block) + \
     (max_snaplen) + 131072 + \
     sizeof (struct block_trailer))

static void pcap_ng_cleanup(pcap_t *p);
# 从 pcap_t 结构体指向的文件中读取下一个数据包，存储在结构体 pcap_pkthdr 中，数据存储在指针 data 中
static int pcap_ng_next_packet(pcap_t *p, struct pcap_pkthdr *hdr,
    u_char **data);

# 从文件中读取指定字节数到缓冲区中，可选择在文件结束时失败或者继续执行，错误信息存储在 errbuf 中
static int
read_bytes(FILE *fp, void *buf, size_t bytes_to_read, int fail_on_eof,
    char *errbuf)
{
    size_t amt_read;

    # 从文件中读取指定字节数到缓冲区中
    amt_read = fread(buf, 1, bytes_to_read, fp);
    # 判断读取的字节数是否和指定的字节数相等
    if (amt_read != bytes_to_read) {
        # 判断是否发生了文件读取错误
        if (ferror(fp)) {
            # 格式化错误信息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
        } else {
            # 判断是否读取到了文件末尾
            if (amt_read == 0 && !fail_on_eof)
                return (0);    /* EOF */
            # 格式化错误信息
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "truncated pcapng dump file; tried to read %zu bytes, only got %zu",
                bytes_to_read, amt_read);
        }
        return (-1);
    }
    return (1);
}

# 从文件中读取一个块（block），并存储在结构体中，错误信息存储在 errbuf 中
static int
read_block(FILE *fp, pcap_t *p, struct block_cursor *cursor, char *errbuf)
{
    struct pcap_ng_sf *ps;
    int status;
    struct block_header bhdr;
    struct block_trailer *btrlr;
    u_char *bdata;
    size_t data_remaining;

    # 获取 pcap_t 结构体中的私有数据
    ps = p->priv;

    # 读取块头部信息
    status = read_bytes(fp, &bhdr, sizeof(bhdr), 0, errbuf);
    if (status <= 0)
        return (status);    /* error or EOF */

    # 如果需要进行字节序转换，则进行转换
    if (p->swapped) {
        bhdr.block_type = SWAPLONG(bhdr.block_type);
        bhdr.total_length = SWAPLONG(bhdr.total_length);
    }

    # 判断块是否过小
    if (bhdr.total_length < sizeof(struct block_header) +
        sizeof(struct block_trailer)) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "block in pcapng dump file has a length of %u < %zu",
            bhdr.total_length,
            sizeof(struct block_header) + sizeof(struct block_trailer));
        return (-1);
    }

    # 判断块的总长度是否是4的倍数
    # 检查数据块的总长度是否是4的倍数，如果不是则报告错误
    if ((bhdr.total_length % 4) != 0) {
        /*
         * No.  Report that as an error.
         */
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "block in pcapng dump file has a length of %u that is not a multiple of 4",
            bhdr.total_length);
        return (-1);
    }

    /*
     * 检查缓冲区是否足够大
     */
    if (p->bufsize < bhdr.total_length) {
        /*
         * No - make it big enough, unless it's too big, in which case we fail.
         */
        void *bigger_buffer;

        if (bhdr.total_length > ps->max_blocksize) {
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "pcapng block size %u > maximum %u", bhdr.total_length,
                ps->max_blocksize);
            return (-1);
        }
        // 重新分配足够大的缓冲区
        bigger_buffer = realloc(p->buffer, bhdr.total_length);
        if (bigger_buffer == NULL) {
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "out of memory");
            return (-1);
        }
        p->buffer = bigger_buffer;
    }

    /*
     * 将读取的内容复制到缓冲区，并读取块的剩余部分
     */
    memcpy(p->buffer, &bhdr, sizeof(bhdr));
    bdata = (u_char *)p->buffer + sizeof(bhdr);
    data_remaining = bhdr.total_length - sizeof(bhdr);
    if (read_bytes(fp, bdata, data_remaining, 1, errbuf) == -1)
        return (-1);

    /*
     * 从尾部获取块大小
     */
    btrlr = (struct block_trailer *)(bdata + data_remaining - sizeof (struct block_trailer));
    if (p->swapped)
        btrlr->total_length = SWAPLONG(btrlr->total_length);

    /*
     * 检查头部和尾部的总长度是否一致
     */
    if (bhdr.total_length != btrlr->total_length) {
        /*
         * No.
         */
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "block total length in header and trailer don't match");
        return (-1);
    }

    /*
     * 初始化游标
     */
    cursor->data = bdata;
    # 更新游标中剩余数据的长度，减去块尾的大小
    cursor->data_remaining = data_remaining - sizeof(struct block_trailer);
    # 将块头中的块类型赋值给游标的块类型
    cursor->block_type = bhdr.block_type;
    # 返回值为1，表示成功
    return (1);
    /*
     * 从块数据中获取指定大小的数据块
     */
    static void *
    get_from_block_data(struct block_cursor *cursor, size_t chunk_size,
        char *errbuf)
    {
        void *data;

        /*
         * 确保块数据中剩余的数据量足够
         */
        if (cursor->data_remaining < chunk_size) {
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "block of type %u in pcapng dump file is too short",
                cursor->block_type);
            return (NULL);
        }

        /*
         * 返回当前指针，并跳过数据块
         */
        data = cursor->data;
        cursor->data += chunk_size;
        cursor->data_remaining -= chunk_size;
        return (data);
    }

    /*
     * 从块数据中获取选项头部
     */
    static struct option_header *
    get_opthdr_from_block_data(pcap_t *p, struct block_cursor *cursor, char *errbuf)
    {
        struct option_header *opthdr;

        opthdr = get_from_block_data(cursor, sizeof(*opthdr), errbuf);
        if (opthdr == NULL) {
            /*
             * 选项头部被截断
             */
            return (NULL);
        }

        /*
         * 如果需要，进行字节交换
         */
        if (p->swapped) {
            opthdr->option_code = SWAPSHORT(opthdr->option_code);
            opthdr->option_length = SWAPSHORT(opthdr->option_length);
        }

        return (opthdr);
    }

    /*
     * 从块数据中获取选项值
     */
    static void *
    get_optvalue_from_block_data(struct block_cursor *cursor,
        struct option_header *opthdr, char *errbuf)
    {
        size_t padded_option_len;
        void *optvalue;

        /* 将选项长度填充到4字节边界 */
        padded_option_len = opthdr->option_length;
        padded_option_len = ((padded_option_len + 3)/4)*4;

        optvalue = get_from_block_data(cursor, padded_option_len, errbuf);
        if (optvalue == NULL) {
            /*
             * 选项值被截断
             */
            return (NULL);
        }

        return (optvalue);
    }

    /*
     * 处理接口描述块的选项
     */
    static int
    process_idb_options(pcap_t *p, struct block_cursor *cursor, uint64_t *tsresol,
        uint64_t *tsoffset, int *is_binary, char *errbuf)
    {
        struct option_header *opthdr;
        void *optvalue;
        int saw_tsresol, saw_tsoffset;
    # 定义一个名为 tsresol_opt 的 8 位无符号整数变量
    uint8_t tsresol_opt;
    # 定义一个名为 i 的无符号整数变量
    u_int i;

    # 将 saw_tsresol 变量的值设置为 0
    saw_tsresol = 0;
    # 将 saw_tsoffset 变量的值设置为 0
    saw_tsoffset = 0;
    # 关闭函数定义
    }
# 返回 0，表示接口添加成功
done:
    return (0);
}

static int
add_interface(pcap_t *p, struct interface_description_block *idbp,
    struct block_cursor *cursor, char *errbuf)
{
    struct pcap_ng_sf *ps;
    uint64_t tsresol;
    uint64_t tsoffset;
    int is_binary;

    ps = p->priv;

    '''
    * 计算接口数量
    '''
    ps->ifcount++;

    '''
    * 根据需要扩展每个接口信息的数组
    '''
    }

    ps->ifaces[ps->ifcount - 1].snaplen = idbp->snaplen;

    '''
    * 设置默认时间戳分辨率和偏移量
    '''
    tsresol = 1000000;    # 微秒分辨率
    is_binary = 0;        # 这是10的幂
    tsoffset = 0;        # 绝对时间戳

    '''
    * 现在查找各种时间戳选项，以便知道如何解释此接口的时间戳
    '''
    if (process_idb_options(p, cursor, &tsresol, &tsoffset, &is_binary,
        errbuf) == -1)
        return (0);

    ps->ifaces[ps->ifcount - 1].tsresol = tsresol;
    ps->ifaces[ps->ifcount - 1].tsoffset = tsoffset;

    '''
    * 确定我们是为此接口放大、缩小还是不做任何处理
    '''
    if (tsresol == ps->user_tsresol) {
        '''
        * 分辨率是用户想要的分辨率，所以我们不必进行缩放
        '''
        ps->ifaces[ps->ifcount - 1].scale_type = PASS_THROUGH;
    } else if (tsresol > ps->user_tsresol) {
        '''
        * 分辨率大于用户想要的，所以我们必须将时间戳缩小
        '''
        if (is_binary)
            ps->ifaces[ps->ifcount - 1].scale_type = SCALE_DOWN_BIN;
        else {
            '''
            * 计算缩放因子
            '''
            ps->ifaces[ps->ifcount - 1].scale_factor = tsresol/ps->user_tsresol;
            ps->ifaces[ps->ifcount - 1].scale_type = SCALE_DOWN_DEC;
        }
    } else {
        /*
         * 如果分辨率低于用户要求的分辨率，
         * 那么我们需要将时间戳放大。
         */
        if (is_binary)
            ps->ifaces[ps->ifcount - 1].scale_type = SCALE_UP_BIN;
        else {
            /*
             * 计算放大倍数。
             */
            ps->ifaces[ps->ifcount - 1].scale_factor = ps->user_tsresol/tsresol;
            ps->ifaces[ps->ifcount - 1].scale_type = SCALE_UP_DEC;
        }
    }
    return (1);
}
/*
 * 检查这是否是一个 pcapng 保存文件，如果是，则从头部提取相关信息。
 */
pcap_t *
pcap_ng_check_header(const uint8_t *magic, FILE *fp, u_int precision,
    char *errbuf, int *err)
{
    bpf_u_int32 magic_int;  // 用于存储 magic 数组的前4个字节
    size_t amt_read;  // 用于存储读取的字节数
    bpf_u_int32 total_length;  // 用于存储总长度
    bpf_u_int32 byte_order_magic;  // 用于存储字节顺序的魔术值
    struct block_header *bhdrp;  // 用于存储块头部信息
    struct section_header_block *shbp;  // 用于存储节头部块信息
    pcap_t *p;  // pcap 结构体指针
    int swapped = 0;  // 用于标记是否进行了字节交换
    struct pcap_ng_sf *ps;  // pcap_ng_sf 结构体指针
    int status;  // 用于存储状态信息
    struct block_cursor cursor;  // 用于存储块游标信息
    struct interface_description_block *idbp;  // 用于存储接口描述块信息

    /*
     * 假设没有读取错误。
     */
    *err = 0;

    /*
     * 检查文件的前4个字节是否是 pcapng 保存文件的块类型。
     */
    memcpy(&magic_int, magic, sizeof(magic_int));
    if (magic_int != BT_SHB) {
        /*
         * XXX - 检查这是否看起来像是经过 UN*X 和 DOS/Windows 文本文件格式映射后的块类型，
         * 如果是，查找适当位置的字节顺序魔术值，如果找到，报告这可能是在文本文件格式下在 UN*X 和 Windows 之间传输的 pcapng 文件？
         */
        return (NULL);    /* 不是 */
    }

    /*
     * 好的，是的。然而，那只是 \n\r\r\n，所以它可能是一个普通的文本文件。
     *
     * 但是，它不可能是任何其他类型的捕获文件，所以我们可以读取假设的节头部块的其余部分；
     * 将块类型放入公共头部，读取公共头部的其余部分和 SHB 的固定长度部分，并查找字节顺序魔术值。
     */
    amt_read = fread(&total_length, 1, sizeof(total_length), fp);
    # 如果读取的字节数小于总长度，说明读取出错
    if (amt_read < sizeof(total_length)) {
        # 如果文件流出错，将错误信息写入errbuf，设置err为1，返回NULL
        if (ferror(fp)) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
            *err = 1;
            return (NULL);    /* fail */
        }

        '''
         * 可能是一个奇怪的短文本文件，所以只需说
         * "不是pcapng"。
         '''
        return (NULL);
    }
    # 读取字节顺序魔术数
    amt_read = fread(&byte_order_magic, 1, sizeof(byte_order_magic), fp);
    # 如果读取的字节数小于字节顺序魔术数的长度，说明读取出错
    if (amt_read < sizeof(byte_order_magic)) {
        # 如果文件流出错，将错误信息写入errbuf，设置err为1，返回NULL
        if (ferror(fp)) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "error reading dump file");
            *err = 1;
            return (NULL);    /* fail */
        }

        '''
         * 可能是一个奇怪的短文本文件，所以只需说
         * "不是pcapng"。
         '''
        return (NULL);
    }
    # 如果字节顺序魔术数不等于预定义的魔术数
    if (byte_order_magic != BYTE_ORDER_MAGIC) {
        # 将字节顺序魔术数进行字节交换
        byte_order_magic = SWAPLONG(byte_order_magic);
        # 如果交换后的字节顺序魔术数仍然不等于预定义的魔术数
        if (byte_order_magic != BYTE_ORDER_MAGIC) {
            '''
             * 不是一个pcapng文件。
             '''
            return (NULL);
        }
        # 设置交换标志为1
        swapped = 1;
        # 将总长度进行字节交换
        total_length = SWAPLONG(total_length);
    }

    '''
     * 检查总长度的合理性。
     '''
    # 如果总长度小于数据块头部、section头部和块尾部的长度之和，或者大于BT_SHB_INSANE_MAX
    if (total_length < sizeof(*bhdrp) + sizeof(*shbp) + sizeof(struct block_trailer) ||
            (total_length > BT_SHB_INSANE_MAX)) {
        # 将错误信息写入errbuf，设置err为1，返回NULL
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "Section Header Block in pcapng dump file has invalid length %zu < _%u_ < %u (BT_SHB_INSANE_MAX)",
            sizeof(*bhdrp) + sizeof(*shbp) + sizeof(struct block_trailer),
            total_length,
            BT_SHB_INSANE_MAX);

        *err = 1;
        return (NULL);
    }

    '''
     * 好的pcapng文件。
     * 为其分配一个pcap_t。
     '''
    # 为pcapng文件分配一个pcap_t
    p = PCAP_OPEN_OFFLINE_COMMON(errbuf, struct pcap_ng_sf);
    # 如果分配失败，设置err为1，返回NULL
    if (p == NULL) {
        /* 分配失败。 */
        *err = 1;
        return (NULL);
    }
    # 设置结构体 p 中的 swapped 属性为传入的 swapped 值
    p->swapped = swapped;
    # 设置结构体 ps 为结构体 p 的 priv 属性
    ps = p->priv;

    """
     * 用户需要什么样的时间精度？
     """
    # 根据 precision 的值进行不同的处理
    switch (precision) {

    case PCAP_TSTAMP_PRECISION_MICRO:
        # 如果 precision 为 PCAP_TSTAMP_PRECISION_MICRO，则设置 ps 中的 user_tsresol 为 1000000
        ps->user_tsresol = 1000000;
        break;

    case PCAP_TSTAMP_PRECISION_NANO:
        # 如果 precision 为 PCAP_TSTAMP_PRECISION_NANO，则设置 ps 中的 user_tsresol 为 1000000000
        ps->user_tsresol = 1000000000;
        break;

    default:
        # 如果 precision 为其他值，则设置错误信息到 errbuf 中，释放 p，设置 err 为 1，返回 NULL
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "unknown time stamp resolution %u", precision);
        free(p);
        *err = 1;
        return (NULL);
    }

    # 设置结构体 p 中的 opt.tstamp_precision 为 precision 的值
    p->opt.tstamp_precision = precision;

    """
     * 分配一个缓冲区来读取块。我们默认为最大值：
     *
     *    我们读取头部的 SHB 的总长度；
     *
     *    2K，这应该足够大，可以容纳一个包含完整大小以太网帧的增强数据包块，并留有一些选项空间。
     *
     * 如果我们找到一个更大的块，我们重新分配缓冲区，直到最大大小。我们从初始最大块大小开始；如果我们看到任何链路层头部类型的最大快照导致更大的最大块长度，我们会增加最大值。
     """
    # 设置结构体 p 中的 bufsize 为 2048
    p->bufsize = 2048;
    # 如果 bufsize 小于 total_length，则将 bufsize 设置为 total_length
    if (p->bufsize < total_length)
        p->bufsize = total_length;
    # 分配大小为 bufsize 的内存给 p 的 buffer 属性
    p->buffer = malloc(p->bufsize);
    # 如果分配内存失败，则设置错误信息到 errbuf 中，释放 p，设置 err 为 1，返回 NULL
    if (p->buffer == NULL) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "out of memory");
        free(p);
        *err = 1;
        return (NULL);
    }
    # 设置结构体 ps 中的 max_blocksize 为 INITIAL_MAX_BLOCKSIZE
    ps->max_blocksize = INITIAL_MAX_BLOCKSIZE;

    """
     * 将我们读取的内容复制到缓冲区中，并读取剩下的 SHB。
     """
    # 设置 bhdrp 为 p->buffer 的地址
    bhdrp = (struct block_header *)p->buffer;
    # 设置 shbp 为 p->buffer 加上结构体 block_header 的大小的地址
    shbp = (struct section_header_block *)((u_char *)p->buffer + sizeof(struct block_header));
    # 设置 bhdrp 中的 block_type 为 magic_int
    bhdrp->block_type = magic_int;
    # 设置 bhdrp 中的 total_length 为 total_length
    bhdrp->total_length = total_length;
    # 设置 shbp 中的 byte_order_magic 为 byte_order_magic
    shbp->byte_order_magic = byte_order_magic;
    # 从文件指针中读取指定长度的字节到缓冲区中，如果失败则跳转到失败标签
    if (read_bytes(fp,
        (u_char *)p->buffer + (sizeof(magic_int) + sizeof(total_length) + sizeof(byte_order_magic)),
        total_length - (sizeof(magic_int) + sizeof(total_length) + sizeof(byte_order_magic)),
        1, errbuf) == -1)
        goto fail;

    # 如果数据需要进行字节交换
    if (p->swapped) {
        '''
         * Byte-swap the fields we've read.
         * 对读取的字段进行字节交换
         '''
        shbp->major_version = SWAPSHORT(shbp->major_version);
        shbp->minor_version = SWAPSHORT(shbp->minor_version);

        '''
         * XXX - we don't care about the section length.
         * 我们不关心部分长度。
         '''
    }
    '''
     * 目前只支持 SHB 版本 1.0 和 1.2；
     * 版本 1.2 被视为与版本 1.0 相同。
     * 请参阅当前版本的 pcapng 规范。
     *
     * 版本 1.2 是由一些编写额外块类型的程序编写的
     * （可以由任何处理它们的代码读取，无论次要版本是否为 0 或 2，因此这不是更改次要版本号的原因）。
     *
     * XXX - pcapng 规范表示读取器应该忽略不支持的版本号的部分；
     * 可能他们也可以报告错误，如果他们跳到文件的末尾而找不到任何他们支持的版本。
     * 目前只支持 SHB 版本 1.0 和 1.2；
     * 版本 1.2 被视为与版本 1.0 相同。
     * 请参阅当前版本的 pcapng 规范。
     *
     * 版本 1.2 是由一些编写额外块类型的程序编写的
     * （可以由任何处理它们的代码读取，无论次要版本是否为 0 或 2，因此这不是更改次要版本号的原因）。
     *
     * XXX - pcapng 规范表示读取器应该忽略不支持的版本号的部分；
     * 可能他们也可以报告错误，如果他们跳到文件的末尾而找不到任何他们支持的版本。
     '''
    # 如果不支持读取的 pcapng 文件版本，则设置错误信息并跳转到失败标签
    if (! (shbp->major_version == PCAP_NG_VERSION_MAJOR &&
           (shbp->minor_version == PCAP_NG_VERSION_MINOR ||
            shbp->minor_version == 2))) {
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "unsupported pcapng savefile version %u.%u",
            shbp->major_version, shbp->minor_version);
        goto fail;
    }
    # 设置主版本号和次版本号
    p->version_major = shbp->major_version;
    p->version_minor = shbp->minor_version;

    '''
     * 保存用户请求的时间戳分辨率。
     * Save the time stamp resolution the user requested.
     '''
    p->opt.tstamp_precision = precision;

    '''
     * 现在开始寻找接口描述块。
     * Now start looking for an Interface Description Block.
     '''
    for (;;) {
        /*
         * 读取下一个数据块。
         */
        status = read_block(fp, p, &cursor, errbuf);
        if (status == 0) {
            /* 文件结束 - 文件中没有 IDB */
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "the capture file has no Interface Description Blocks");
            goto fail;
        }
        if (status == -1)
            goto fail;    /* 出错 */
        switch (cursor.block_type) {

        case BT_IDB:
            /*
             * 获取指向 IDB 固定长度部分的指针。
             */
            idbp = get_from_block_data(&cursor, sizeof(*idbp),
                errbuf);
            if (idbp == NULL)
                goto fail;    /* 出错 */

            /*
             * 如果需要，进行字节交换。
             */
            if (p->swapped) {
                idbp->linktype = SWAPSHORT(idbp->linktype);
                idbp->snaplen = SWAPLONG(idbp->snaplen);
            }

            /*
             * 尝试添加这个接口。
             */
            if (!add_interface(p, idbp, &cursor, errbuf))
                goto fail;

            goto done;

        case BT_EPB:
        case BT_SPB:
        case BT_PB:
            /*
             * 在看到任何 IDB 之前就看到了数据包。这是无效的，因为我们不知道数据包的链路层封装类型。
             */
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "the capture file has a packet block before any Interface Description Blocks");
            goto fail;

        default:
            /*
             * 忽略它。
             */
            break;
        }
    }
done:
    # 设置数据链路类型为给定 ID 对应的数据链路类型
    p->linktype = linktype_to_dlt(idbp->linktype);
    # 调整快照长度为给定数据链路类型对应的快照长度
    p->snapshot = pcap_adjust_snapshot(p->linktype, idbp->snaplen);
    # 设置扩展数据链路类型为 0
    p->linktype_ext = 0;

    """
     如果具有给定数据链路类型的最大快照长度的数据包的最大块大小大于当前最大块大小，则增加最大块大小。
    """
    if (MAX_BLOCKSIZE_FOR_SNAPLEN(max_snaplen_for_dlt(p->linktype)) > ps->max_blocksize)
        ps->max_blocksize = MAX_BLOCKSIZE_FOR_SNAPLEN(max_snaplen_for_dlt(p->linktype));

    # 设置下一个数据包操作为 pcap_ng_next_packet
    p->next_packet_op = pcap_ng_next_packet;
    # 设置清理操作为 pcap_ng_cleanup
    p->cleanup_op = pcap_ng_cleanup;

    # 返回结果
    return (p);

fail:
    # 释放接口列表内存
    free(ps->ifaces);
    # 释放缓冲区内存
    free(p->buffer);
    # 释放 pcap_t 结构内存
    free(p);
    # 设置错误标志为 1
    *err = 1;
    # 返回空指针
    return (NULL);
}

static void
pcap_ng_cleanup(pcap_t *p)
{
    # 获取 pcap_ng_sf 结构
    struct pcap_ng_sf *ps = p->priv;
    # 释放接口列表内存
    free(ps->ifaces);
    # 调用 sf_cleanup 函数
    sf_cleanup(p);
}

"""
  读取并返回保存文件中的下一个数据包。在 hdr 中返回数据包头，在 data 中返回数据内容。
  成功时返回 1，如果没有更多数据包则返回 0，出现错误时返回 -1。
"""
static int
pcap_ng_next_packet(pcap_t *p, struct pcap_pkthdr *hdr, u_char **data)
{
    # 获取 pcap_ng_sf 结构
    struct pcap_ng_sf *ps = p->priv;
    # 定义块游标
    struct block_cursor cursor;
    # 定义状态
    int status;
    # 定义增强数据包块
    struct enhanced_packet_block *epbp;
    # 定义简单数据包块
    struct simple_packet_block *spbp;
    # 定义数据包块
    struct packet_block *pbp;
    # 定义接口 ID 为 0xFFFFFFFF
    bpf_u_int32 interface_id = 0xFFFFFFFF;
    # 定义接口描述块
    struct interface_description_block *idbp;
    # 定义段头块
    struct section_header_block *shbp;
    # 获取文件指针
    FILE *fp = p->rfile;
    # 定义时间戳、秒数、分数
    uint64_t t, sec, frac;

    """
    寻找增强数据包块、简单数据包块或数据包块。
    """

found:
    """
    接口 ID 是否为我们已知的接口？
    """
    if (interface_id >= ps->ifcount) {
        """
        是。失败。
        """
        # 设置错误信息
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "a packet arrived on interface %u, but there's no Interface Description Block for that interface",
            interface_id);
        # 返回 -1
        return (-1);
    }
    # 如果捕获的数据包长度大于快照长度，则返回错误
    if (hdr->caplen > (bpf_u_int32)p->snapshot) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "invalid packet capture length %u, bigger than "
            "snaplen of %d", hdr->caplen, p->snapshot);
        return (-1);
    }

    /*
     * 将时间戳转换为秒和秒的小数部分，
     * 小数部分的单位是文件提供的分辨率单位。
     */
    sec = t / ps->ifaces[interface_id].tsresol + ps->ifaces[interface_id].tsoffset;
    frac = t % ps->ifaces[interface_id].tsresol;

    /*
     * 将小数部分从文件提供的分辨率单位转换为用户请求的分辨率单位。
     */
    switch (ps->ifaces[interface_id].scale_type) {

    case PASS_THROUGH:
        /*
         * 接口的分辨率就是用户想要的，
         * 所以我们完成了。
         */
        break;

    case SCALE_UP_DEC:
        /*
         * 接口的分辨率小于用户想要的；
         * 将小数部分缩放到用户请求的分辨率单位，
         * 方法是乘以用户请求的分辨率和文件提供的分辨率的商。
         *
         * 这两个分辨率都是10的幂，用户请求的分辨率大于文件提供的分辨率，
         * 所以所需的商是一个整数。
         * 我们已经计算了这个商，所以我们只需要乘以它。
         */
        frac *= ps->ifaces[interface_id].scale_factor;
        break;
    # 如果需要放大比例，将小数部分放大到用户请求的分辨率单位
    # 通过将小数部分乘以用户请求的分辨率和文件提供的分辨率的商来实现
    # 文件提供的分辨率是2的幂，所以商不是整数，为了完全使用整数运算，我们将小数部分乘以用户请求的分辨率并除以文件提供的分辨率
    # XXX - 在这里是否有什么聪明的方法可以做到，考虑到我们知道文件提供的分辨率是2的幂？进行乘法后跟除法会有溢出的风险，并且涉及两个非简单的算术运算。
    case SCALE_UP_BIN:
        frac *= ps->user_tsresol;
        frac /= ps->ifaces[interface_id].tsresol;
        break;

    # 如果需要缩小比例，将小数部分缩小到用户请求的分辨率单位
    # 通过将小数部分除以接口提供的比例因子来实现
    # 这些分辨率都是10的幂，用户请求的分辨率小于文件提供的分辨率，所以所讨论的商不是整数，但它的倒数是整数，我们可以直接除以该商的倒数。我们已经计算了该商的倒数，所以我们必须除以它。
    case SCALE_DOWN_DEC:
        frac /= ps->ifaces[interface_id].scale_factor;
        break;
    # 如果需要缩小比例，则将接口分辨率转换为用户请求的分辨率单位
    # 通过将分数部分乘以用户请求的分辨率和文件提供的分辨率的商来实现
    # 我们通过将用户请求的分辨率乘以文件提供的分辨率并除以文件提供的分辨率来实现这一点，因为商可能不适合整数
    # 文件提供的分辨率是2的幂，所以商不是整数，它的倒数也不是整数，因此为了完全使用整数算术，我们将用户请求的分辨率乘以文件提供的分辨率并除以文件提供的分辨率
    # XXX - 在这里我们是否可以做一些聪明的事情，因为我们知道文件提供的分辨率是2的幂？进行乘法后跟除法会有溢出的风险，并且涉及两个非简单的算术操作。
    frac *= ps->user_tsresol;
    frac /= ps->ifaces[interface_id].tsresol;
    break;
#ifdef _WIN32
    /*
     * 如果是在 Windows 平台，struct timeval 的 tv_sec 和 tv_usec 都是 long 类型。
     */
    hdr->ts.tv_sec = (long)sec;
    hdr->ts.tv_usec = (long)frac;
#else
    /*
     * 如果是在 UN*X 平台，struct timeval 的 tv_sec 是 time_t 类型；tv_usec 是 suseconds_t 类型，
     * 在符合当前单一 UNIX 标准的 UN*X 系统中是这样，但并不是所有旧的 UN*X 系统一定支持这种类型，所以只需转换为 int 类型。
     */
    hdr->ts.tv_sec = (time_t)sec;
    hdr->ts.tv_usec = (int)frac;
#endif

    /*
     * 获取数据包数据的指针。
     */
    *data = get_from_block_data(&cursor, hdr->caplen, p->errbuf);
    if (*data == NULL)
        return (-1);

    // 对捕获的数据进行后处理
    pcap_post_process(p->linktype, p->swapped, hdr, *data);

    return (1);
}
```