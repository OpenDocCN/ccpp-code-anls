# `nmap\libpcap\pcap-util.c`

```cpp
/*
 * 版权声明
 * 本代码版权归加利福尼亚大学所有
 * 可以在源代码和二进制形式下进行再发布和使用，无论是否进行修改，只要满足以下条件：
 * 1. 源代码发布时保留以上版权声明和本段文字
 * 2. 包含二进制代码的发布包括以上版权声明和本段文字在内，并在文档或其他提供的材料中包含
 * 3. 所有提及此软件特性或使用的广告材料都要显示以下声明：
 *    “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 *    不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品，除非有特定的事先书面许可。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 * pcap-common.c - pcap 和 pcapng 文件的通用代码
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include "pcap-int.h"
#include "extract.h"
#include "pcap-usb-linux-common.h"

#include "pcap-util.h"

#include "pflog.h"
#include "pcap/can_socketcan.h"
#include "pcap/sll.h"
#include "pcap/usb.h"
#include "pcap/nflog.h"

/*
 * 大多数 DLT_PFLOG 伪头部的版本都有保存在主机字节顺序的 UID 和 PID 字段。
 * 在读取 DLT_PFLOG 数据包时，需要将这些字段从写入文件的主机字节顺序转换为当前主机的字节顺序。
 */
static void
swap_pflog_header(const struct pcap_pkthdr *hdr, u_char *buf)
{
    u_int caplen = hdr->caplen;
    u_int length = hdr->len;
    u_int pfloghdr_length;
    # 将 buf 强制类型转换为 pfloghdr 结构体指针
    struct pfloghdr *pflhdr = (struct pfloghdr *)buf;

    # 如果捕获的数据长度小于 pfloghdr 结构体中 uid 字段的偏移量加上 uid 字段的长度，或者总长度小于 pfloghdr 结构体中 uid 字段的偏移量加上 uid 字段的长度
    if (caplen < (u_int) (offsetof(struct pfloghdr, uid) + sizeof pflhdr->uid) ||
        length < (u_int) (offsetof(struct pfloghdr, uid) + sizeof pflhdr->uid)) {
        # 没有足够的数据包含 uid 字段
        return;
    }

    # 获取 pfloghdr 结构体中的 length 字段
    pfloghdr_length = pflhdr->length;

    # 如果 pfloghdr 结构体中的 length 字段小于 pfloghdr 结构体中 uid 字段的偏移量加上 uid 字段的长度
    if (pfloghdr_length < (u_int) (offsetof(struct pfloghdr, uid) + sizeof pflhdr->uid)) {
        # 头部不包括 uid 字段
        return;
    }
    # 将 uid 字段的值进行大小端转换
    pflhdr->uid = SWAPLONG(pflhdr->uid);

    # 如果捕获的数据长度小于 pfloghdr 结构体中 pid 字段的偏移量加上 pid 字段的长度，或者总长度小于 pfloghdr 结构体中 pid 字段的偏移量加上 pid 字段的长度
    if (caplen < (u_int) (offsetof(struct pfloghdr, pid) + sizeof pflhdr->pid) ||
        length < (u_int) (offsetof(struct pfloghdr, pid) + sizeof pflhdr->pid)) {
        # 没有足够的数据包含 pid 字段
        return;
    }
    # 如果 pfloghdr 结构体中的 length 字段小于 pfloghdr 结构体中 pid 字段的偏移量加上 pid 字段的长度
    if (pfloghdr_length < (u_int) (offsetof(struct pfloghdr, pid) + sizeof pflhdr->pid)) {
        # 头部不包括 pid 字段
        return;
    }
    # 将 pid 字段的值进行大小端转换
    pflhdr->pid = SWAPLONG(pflhdr->pid);

    # 如果捕获的数据长度小于 pfloghdr 结构体中 rule_uid 字段的偏移量加上 rule_uid 字段的长度，或者总长度小于 pfloghdr 结构体中 rule_uid 字段的偏移量加上 rule_uid 字段的长度
    if (caplen < (u_int) (offsetof(struct pfloghdr, rule_uid) + sizeof pflhdr->rule_uid) ||
        length < (u_int) (offsetof(struct pfloghdr, rule_uid) + sizeof pflhdr->rule_uid)) {
        # 没有足够的数据包含 rule_uid 字段
        return;
    }
    # 如果 pfloghdr 结构体中的 length 字段小于 pfloghdr 结构体中 rule_uid 字段的偏移量加上 rule_uid 字段的长度
    if (pfloghdr_length < (u_int) (offsetof(struct pfloghdr, rule_uid) + sizeof pflhdr->rule_uid)) {
        # 头部不包括 rule_uid 字段
        return;
    }
    # 将 rule_uid 字段的值进行大小端转换
    pflhdr->rule_uid = SWAPLONG(pflhdr->rule_uid);

    # 如果捕获的数据长度小于 pfloghdr 结构体中 rule_pid 字段的偏移量加上 rule_pid 字段的长度，或者总长度小于 pfloghdr 结构体中 rule_pid 字段的偏移量加上 rule_pid 字段的长度
    if (caplen < (u_int) (offsetof(struct pfloghdr, rule_pid) + sizeof pflhdr->rule_pid) ||
        length < (u_int) (offsetof(struct pfloghdr, rule_pid) + sizeof pflhdr->rule_pid)) {
        # 没有足够的数据包含 rule_pid 字段
        return;
    }
    # 如果 pfloghdr 结构体中的 length 字段小于 pfloghdr 结构体中 rule_pid 字段的偏移量加上 rule_pid 字段的长度
    if (pfloghdr_length < (u_int) (offsetof(struct pfloghdr, rule_pid) + sizeof pflhdr->rule_pid)) {
        # 头部不包括 rule_pid 字段
        return;
    }
    # 将 rule_pid 字段的值进行大小端转换
    pflhdr->rule_pid = SWAPLONG(pflhdr->rule_pid);
}
/*
 * DLT_LINUX_SLL packets with a protocol type of LINUX_SLL_P_CAN or
 * LINUX_SLL_P_CANFD have SocketCAN headers in front of the payload,
 * with the CAN ID being in host byte order.
 *
 * When reading a DLT_LINUX_SLL packet, we need to check for those
 * packets and convert the CAN ID from the byte order of the host that
 * wrote the file to this host's byte order.
 */
static void
swap_linux_sll_header(const struct pcap_pkthdr *hdr, u_char *buf)
{
    u_int caplen = hdr->caplen;  // 获取数据包的捕获长度
    u_int length = hdr->len;  // 获取数据包的实际长度
    struct sll_header *shdr = (struct sll_header *)buf;  // 将数据包的头部转换为 sll_header 结构体
    uint16_t protocol;  // 定义一个 16 位的协议字段
    pcap_can_socketcan_hdr *chdr;  // 定义一个指向 pcap_can_socketcan_hdr 结构体的指针

    if (caplen < (u_int) sizeof(struct sll_header) ||  // 如果捕获长度小于 sll_header 结构体的长度
        length < (u_int) sizeof(struct sll_header)) {  // 或者实际长度小于 sll_header 结构体的长度
        /* Not enough data to have the protocol field */
        return;  // 返回，不做处理
    }

    protocol = EXTRACT_BE_U_2(&shdr->sll_protocol);  // 从 sll_header 结构体中提取协议字段，并转换为主机字节序
    if (protocol != LINUX_SLL_P_CAN && protocol != LINUX_SLL_P_CANFD)  // 如果协议字段不是 LINUX_SLL_P_CAN 或 LINUX_SLL_P_CANFD
        return;  // 返回，不做处理

    /*
     * SocketCAN packet; fix up the packet's header.
     */
    chdr = (pcap_can_socketcan_hdr *)(buf + sizeof(struct sll_header));  // 将指针指向数据包头部之后的位置
    if (caplen < (u_int) sizeof(struct sll_header) + sizeof(chdr->can_id) ||  // 如果捕获长度小于 sll_header 结构体长度加上 CAN ID 的长度
        length < (u_int) sizeof(struct sll_header) + sizeof(chdr->can_id)) {  // 或者实际长度小于 sll_header 结构体长度加上 CAN ID 的长度
        /* Not enough data to have the CAN ID */
        return;  // 返回，不做处理
    }
    chdr->can_id = SWAPLONG(chdr->can_id);  // 将 CAN ID 转换为本机字节序
}

/*
 * The same applies for DLT_LINUX_SLL2.
 */
static void
swap_linux_sll2_header(const struct pcap_pkthdr *hdr, u_char *buf)
{
    u_int caplen = hdr->caplen;  // 获取数据包的捕获长度
    u_int length = hdr->len;  // 获取数据包的实际长度
    struct sll2_header *shdr = (struct sll2_header *)buf;  // 将数据包的头部转换为 sll2_header 结构体
    uint16_t protocol;  // 定义一个 16 位的协议字段
    pcap_can_socketcan_hdr *chdr;  // 定义一个指向 pcap_can_socketcan_hdr 结构体的指针

    if (caplen < (u_int) sizeof(struct sll2_header) ||  // 如果捕获长度小于 sll2_header 结构体的长度
        length < (u_int) sizeof(struct sll2_header)) {  // 或者实际长度小于 sll2_header 结构体的长度
        /* Not enough data to have the protocol field */
        return;  // 返回，不做处理
    }

    protocol = EXTRACT_BE_U_2(&shdr->sll2_protocol);  // 从 sll2_header 结构体中提取协议字段，并转换为主机字节序
    # 如果协议不是LINUX_SLL_P_CAN并且不是LINUX_SLL_P_CANFD，则返回
    if (protocol != LINUX_SLL_P_CAN && protocol != LINUX_SLL_P_CANFD)
        return;

    '''
     SocketCAN数据包；修复数据包的头部。
    '''
    # 将buf中的数据转换为pcap_can_socketcan_hdr类型的指针
    chdr = (pcap_can_socketcan_hdr *)(buf + sizeof(struct sll2_header));
    # 如果捕获的数据长度小于结构体sll2_header的大小加上can_id的大小，或者总长度小于结构体sll2_header的大小加上can_id的大小，则返回
    if (caplen < (u_int) sizeof(struct sll2_header) + sizeof(chdr->can_id) ||
        length < (u_int) sizeof(struct sll2_header) + sizeof(chdr->can_id)) {
        # 没有足够的数据来包含CAN ID
        return;
    }
    # 将chdr中的can_id进行大小端转换
    chdr->can_id = SWAPLONG(chdr->can_id);
/*
 * 当捕获时，DLT_USB_LINUX 和 DLT_USB_LINUX_MMAPPED 头部以主机字节顺序存储
 * （直接从内核共享的内存映射缓冲区中提供）。
 *
 * 当读取 DLT_USB_LINUX 或 DLT_USB_LINUX_MMAPPED 数据包时，我们需要将其从写入文件的主机字节顺序转换为当前主机的字节顺序。
 */
static void
swap_linux_usb_header(const struct pcap_pkthdr *hdr, u_char *buf,
    int header_len_64_bytes)
{
    pcap_usb_header_mmapped *uhdr = (pcap_usb_header_mmapped *)buf;
    bpf_u_int32 offset = 0;

    /*
     * “offset”是我们要交换的字段之后的偏移量；
     * 在检查捕获的数据长度是否包含整个字段之前，我们跳过字段。
     */

    /*
     * URB id 是一个完全不透明的值；我们真的需要将其转换为读取主机的字节顺序吗???
     */
    offset += 8;            /* 跳过 id */
    if (hdr->caplen < offset)
        return;
    uhdr->id = SWAPLL(uhdr->id);

    offset += 4;            /* 跳过各种 1 字节字段 */

    offset += 2;            /* 跳过 bus_id */
    if (hdr->caplen < offset)
        return;
    uhdr->bus_id = SWAPSHORT(uhdr->bus_id);

    offset += 2;            /* 跳过各种 1 字节字段 */

    offset += 8;            /* 跳过 ts_sec */
    if (hdr->caplen < offset)
        return;
    uhdr->ts_sec = SWAPLL(uhdr->ts_sec);

    offset += 4;            /* 跳过 ts_usec */
    if (hdr->caplen < offset)
        return;
    uhdr->ts_usec = SWAPLONG(uhdr->ts_usec);

    offset += 4;            /* 跳过 status */
    if (hdr->caplen < offset)
        return;
    uhdr->status = SWAPLONG(uhdr->status);

    offset += 4;            /* 跳过 urb_len */
    if (hdr->caplen < offset)
        return;
    uhdr->urb_len = SWAPLONG(uhdr->urb_len);

    offset += 4;            /* 跳过 data_len */
    if (hdr->caplen < offset)
        return;
    # 将数据长度字段转换为大端字节序
    uhdr->data_len = SWAPLONG(uhdr->data_len);

    # 如果传输类型为等时传输
    if (uhdr->transfer_type == URB_ISOCHRONOUS) {
        # 偏移量增加4，跳过 s.iso.error_count
        offset += 4;            
        # 如果数据包长度小于偏移量，返回
        if (hdr->caplen < offset)
            return;
        # 将 s.iso.error_count 字段转换为大端字节序
        uhdr->s.iso.error_count = SWAPLONG(uhdr->s.iso.error_count);

        # 偏移量增加4，跳过 s.iso.numdesc
        offset += 4;            
        # 如果数据包长度小于偏移量，返回
        if (hdr->caplen < offset)
            return;
        # 将 s.iso.numdesc 字段转换为大端字节序
        uhdr->s.iso.numdesc = SWAPLONG(uhdr->s.iso.numdesc);
    } else
        # 偏移量增加8，跳过 USB 设置头部
        offset += 8;            

    '''
     * 对于旧的头部，头部之后没有等时描述符。
     *
     * 对于新的头部，头部中实际的描述符数量不是 s.iso.numdesc，而是 ndesc - 只有
     * 前 N 个描述符，对于某个 N 的值，被放入头部，并且 ndesc 被设置为实际复制的数量。
     * 另外，如果 s.iso.numdesc 为负数，没有描述符被捕获，并且 ndesc 被设置为 0。
     */
    if (header_len_64_bytes) {
        /*
         * 如果头部长度为64字节，则这可能是“版本1”头部，末尾有16字节的附加字段，
         * 或者是来自内存映射捕获的“版本0”头部，末尾有16字节的零填充。
         * 将它们按照“版本1”头部的格式进行字节交换。
         */
        offset += 4;            /* 跳过间隔 */
        if (hdr->caplen < offset)
            return;
        uhdr->interval = SWAPLONG(uhdr->interval);

        offset += 4;            /* 跳过起始帧 */
        if (hdr->caplen < offset)
            return;
        uhdr->start_frame = SWAPLONG(uhdr->start_frame);

        offset += 4;            /* 跳过传输标志 */
        if (hdr->caplen < offset)
            return;
        uhdr->xfer_flags = SWAPLONG(uhdr->xfer_flags);

        offset += 4;            /* 跳过描述符数量 */
        if (hdr->caplen < offset)
            return;
        uhdr->ndesc = SWAPLONG(uhdr->ndesc);

        if (uhdr->transfer_type == URB_ISOCHRONOUS) {
            /* 交换 struct linux_usb_isodesc 中的值 */
            usb_isodesc *pisodesc;
            uint32_t i;

            pisodesc = (usb_isodesc *)(void *)(buf+offset);
            for (i = 0; i < uhdr->ndesc; i++) {
                offset += 4;        /* 跳过状态 */
                if (hdr->caplen < offset)
                    return;
                pisodesc->status = SWAPLONG(pisodesc->status);

                offset += 4;        /* 跳过偏移量 */
                if (hdr->caplen < offset)
                    return;
                pisodesc->offset = SWAPLONG(pisodesc->offset);

                offset += 4;        /* 跳过长度 */
                if (hdr->caplen < offset)
                    return;
                pisodesc->len = SWAPLONG(pisodesc->len);

                offset += 4;        /* 跳过填充 */

                pisodesc++;
            }
        }
    }
/*
 * The DLT_NFLOG "packets" have a mixture of big-endian and host-byte-order
 * data.  They begin with a fixed-length header with big-endian fields,
 * followed by a set of TLVs, where the type and length are in host byte order but the values are either big-endian or are a raw byte
 * sequence that's the same regardless of the host's byte order.
 *
 * When reading a DLT_NFLOG packet, we need to convert the type and
 * length values from the byte order of the host that wrote the file
 * to the byte order of this host.
 */
static void
swap_nflog_header(const struct pcap_pkthdr *hdr, u_char *buf)
{
    u_char *p = buf;  // 指向缓冲区的指针
    nflog_hdr_t *nfhdr = (nflog_hdr_t *)buf;  // 将缓冲区转换为 nflog_hdr_t 结构体指针
    nflog_tlv_t *tlv;  // 定义 nflog_tlv_t 结构体指针
    u_int caplen = hdr->caplen;  // 获取数据包的捕获长度
    u_int length = hdr->len;  // 获取数据包的实际长度
    uint16_t size;  // 定义一个 16 位整数

    if (caplen < (u_int) sizeof(nflog_hdr_t) ||
        length < (u_int) sizeof(nflog_hdr_t)) {
        /* Not enough data to have any TLVs. */
        return;  // 如果数据包长度不足以包含任何 TLVs，则直接返回
    }

    if (nfhdr->nflog_version != 0) {
        /* Unknown NFLOG version */
        return;  // 如果 NFLOG 版本未知，则直接返回
    }

    length -= sizeof(nflog_hdr_t);  // 减去固定头部的长度
    caplen -= sizeof(nflog_hdr_t);  // 减去固定头部的长度
    p += sizeof(nflog_hdr_t);  // 指针移动到固定头部之后

    while (caplen >= sizeof(nflog_tlv_t)) {  // 当数据包长度大于等于 TLV 的长度时
        tlv = (nflog_tlv_t *) p;  // 将指针 p 转换为 nflog_tlv_t 结构体指针

        /* Swap the type and length. */
        tlv->tlv_type = SWAPSHORT(tlv->tlv_type);  // 交换类型的字节顺序
        tlv->tlv_length = SWAPSHORT(tlv->tlv_length);  // 交换长度的字节顺序

        /* Get the length of the TLV. */
        size = tlv->tlv_length;  // 获取 TLV 的长度
        if (size % 4 != 0)
            size += 4 - size % 4;  // 如果长度不是 4 的倍数，则调整为 4 的倍数

        /* Is the TLV's length less than the minimum? */
        if (size < sizeof(nflog_tlv_t)) {
            /* Yes. Give up now. */
            return;  // 如果 TLV 的长度小于最小长度，则直接返回
        }

        /* Do we have enough data for the full TLV? */
        if (caplen < size || length < size) {
            /* No. */
            return;  // 如果数据包长度不足以包含完整的 TLV，则直接返回
        }

        /* Skip over the TLV. */
        length -= size;  // 减去 TLV 的长度
        caplen -= size;  // 减去 TLV 的长度
        p += size;  // 指针移动到下一个 TLV
    }
}

static void
void swap_pseudo_headers(int linktype, struct pcap_pkthdr *hdr, u_char *data)
{
    /*
     * 将伪头部从文件保存时的主机字节顺序转换为我们的字节顺序，如果需要的话。
     */
    switch (linktype) {

    case DLT_PFLOG:
        swap_pflog_header(hdr, data);
        break;

    case DLT_LINUX_SLL:
        swap_linux_sll_header(hdr, data);
        break;

    case DLT_LINUX_SLL2:
        swap_linux_sll2_header(hdr, data);
        break;

    case DLT_USB_LINUX:
        swap_linux_usb_header(hdr, data, 0);
        break;

    case DLT_USB_LINUX_MMAPPED:
        swap_linux_usb_header(hdr, data, 1);
        break;

    case DLT_NFLOG:
        swap_nflog_header(hdr, data);
        break;
    }
}

void
pcap_post_process(int linktype, int swapped, struct pcap_pkthdr *hdr,
    u_char *data)
{
    if (swapped)
        swap_pseudo_headers(linktype, hdr, data);

    fixup_pcap_pkthdr(linktype, hdr, data);
}

void
fixup_pcap_pkthdr(int linktype, struct pcap_pkthdr *hdr, const u_char *data)
{
    const pcap_usb_header_mmapped *usb_hdr;

    usb_hdr = (const pcap_usb_header_mmapped *) data;
}
    # 如果链路类型为 DLT_USB_LINUX_MMAPPED 并且数据包的捕获长度大于等于 pcap_usb_header_mmapped 结构体的大小
    if (linktype == DLT_USB_LINUX_MMAPPED &&
        hdr->caplen >= sizeof (pcap_usb_header_mmapped)) {
        '''
        在旧版本的 libpcap 中，对于内存映射捕获，传入等时传输的完成事件的“在总线上的长度”被错误计算；
        它需要基于描述符中的偏移和长度进行计算，而不是基于原始 URB 长度，但实际上没有这样做。
        
        如果这个数据包包含传输的数据（是的，如果我们有数据，data_flag 为 0），并且总的在网络上的长度等于从原始 URB 长度计算出的值，
        那么它可能是其中一个这样的传输。
        
        只有当我们有完整的 USB 伪头部时才执行这个操作。
        '''
        # 如果数据包不包含传输的数据，并且数据包的总长度等于 pcap_usb_header_mmapped 结构体的大小加上（usb_hdr->ndesc * sizeof (usb_isodesc)）加上 usb_hdr->urb_len
        if (!usb_hdr->data_flag &&
            hdr->len == sizeof(pcap_usb_header_mmapped) +
              (usb_hdr->ndesc * sizeof (usb_isodesc)) + usb_hdr->urb_len) {
            '''
            它可能需要修复；如果它是传入等时传输的完成事件，则修复它。
            '''
            # 调用修复函数 fix_linux_usb_mmapped_length
            fix_linux_usb_mmapped_length(hdr, data);
        }
    }
# 闭合前面的函数定义
```