# `nmap\libpcap\pcap\can_socketcan.h`

```cpp
#ifndef lib_pcap_can_socketcan_h
#define lib_pcap_can_socketcan_h

#include <pcap/pcap-inttypes.h>

/*
 * SocketCAN header, as per Documentation/networking/can.txt in the
 * Linux source.
 */
// 定义一个结构体，用于表示 SocketCAN 的数据帧格式
typedef struct {
    uint32_t can_id; // CAN 标识符
    uint8_t payload_length; // 数据负载长度
    uint8_t fd_flags; // CAN FD 数据帧标志
    uint8_t reserved1; // 保留字段1
    uint8_t reserved2; // 保留字段2
} pcap_can_socketcan_hdr;

/* Bits in the fd_flags field */
// CAN FD 数据帧标志字段中的位定义
#define CANFD_BRS   0x01 /* bit rate switch (second bitrate for payload data) */
#define CANFD_ESI   0x02 /* error state indicator of the transmitting node */
#define CANFD_FDF   0x04 /* mark CAN FD for dual use of CAN format */

#endif
```