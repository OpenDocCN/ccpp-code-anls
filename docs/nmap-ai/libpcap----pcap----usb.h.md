# `nmap\libpcap\pcap\usb.h`

```cpp
/*
 * 版权声明
 * 版权所有（c）2006年 Paolo Abeni（意大利）
 *
 * 源代码和二进制形式的再发布和使用，无论是否经过修改，
 * 都是允许的，前提是满足以下条件：
 *
 * 1. 源代码的再发布必须保留上述版权声明、此条件列表和以下免责声明。
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，
 * 包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，
 * 版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）承担任何责任。
 * 即使已被告知可能发生此类损害的可能性，也不承担任何责任。
 *
 * 基本的 USB 数据结构
 * 作者：Paolo Abeni <paolo.abeni@email.it>
 */

#ifndef lib_pcap_usb_h
#define lib_pcap_usb_h

#include <pcap/pcap-inttypes.h>

/*
 * 可能的传输模式
 */
#define URB_TRANSFER_IN   0x80
#define URB_ISOCHRONOUS   0x0
#define URB_INTERRUPT     0x1
#define URB_CONTROL       0x2
#define URB_BULK          0x3

/*
 * 可能的事件类型
 */
#define URB_SUBMIT        'S'
#define URB_COMPLETE      'C'
#define URB_ERROR         'E'
/*
 * USB setup header as defined in USB specification.
 * Appears at the front of each Control S-type packet in DLT_USB captures.
 */
typedef struct _usb_setup {
    uint8_t bmRequestType;  // USB请求类型
    uint8_t bRequest;  // USB请求
    uint16_t wValue;  // 值
    uint16_t wIndex;  // 索引
    uint16_t wLength;  // 长度
} pcap_usb_setup;

/*
 * Information from the URB for Isochronous transfers.
 */
typedef struct _iso_rec {
    int32_t    error_count;  // 错误计数
    int32_t    numdesc;  // 描述数量
} iso_rec;

/*
 * Header prepended by linux kernel to each event.
 * Appears at the front of each packet in DLT_USB_LINUX captures.
 */
typedef struct _usb_header {
    uint64_t id;  // ID
    uint8_t event_type;  // 事件类型
    uint8_t transfer_type;  // 传输类型
    uint8_t endpoint_number;  // 端点号
    uint8_t device_address;  // 设备地址
    uint16_t bus_id;  // 总线ID
    char setup_flag;  // 设置标志
    char data_flag;  // 数据标志
    int64_t ts_sec;  // 时间戳秒
    int32_t ts_usec;  // 时间戳微秒
    int32_t status;  // 状态
    uint32_t urb_len;  // URB长度
    uint32_t data_len;  // 数据长度
    pcap_usb_setup setup;  // USB设置
} pcap_usb_header;

/*
 * Header prepended by linux kernel to each event for the 2.6.31
 * and later kernels; for the 2.6.21 through 2.6.30 kernels, the
 * "iso_rec" information, and the fields starting with "interval"
 * are zeroed-out padding fields.
 *
 * Appears at the front of each packet in DLT_USB_LINUX_MMAPPED captures.
 */
typedef struct _usb_header_mmapped {
    uint64_t id;  // ID
    uint8_t event_type;  // 事件类型
    uint8_t transfer_type;  // 传输类型
    uint8_t endpoint_number;  // 端点号
    uint8_t device_address;  // 设备地址
    uint16_t bus_id;  // 总线ID
    char setup_flag;  // 设置标志
    char data_flag;  // 数据标志
    int64_t ts_sec;  // 时间戳秒
    int32_t ts_usec;  // 时间戳微秒
    int32_t status;  // 状态
    uint32_t urb_len;  // URB长度
    uint32_t data_len;  // 数据长度
    union {
        pcap_usb_setup setup;  // USB设置
        iso_rec iso;  // Isochronous传输的信息
    } s;
    int32_t    interval;    // 间隔时间，用于中断和等时事件
    # 定义一个32位整数变量start_frame，用于存储等时事件的起始帧
    int32_t start_frame;    /* for Isochronous events */
    # 定义一个32位无符号整数变量xfer_flags，用于存储URB的传输标志的副本
    uint32_t xfer_flags;    /* copy of URB's transfer flags */
    # 定义一个32位无符号整数变量ndesc，用于存储等时描述符的数量
    uint32_t ndesc;    /* number of isochronous descriptors */
// 定义了一个结构体 usb_isodesc，用于描述等时传输的描述符信息
typedef struct _usb_isodesc {
    int32_t        status;  // 描述符状态
    uint32_t    offset;  // 数据偏移量
    uint32_t    len;  // 数据长度
    uint8_t    pad[4];  // 填充字节
} usb_isodesc;  // usb_isodesc 结构体定义结束
#endif  // 结束条件编译指令
```