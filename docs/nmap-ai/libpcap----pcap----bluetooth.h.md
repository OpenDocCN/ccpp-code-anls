# `nmap\libpcap\pcap\bluetooth.h`

```cpp
/*
 * 版权声明，声明作者保留所有权利
 * 可以在源代码和二进制形式下重新分发和使用，无论是否经过修改
 * 需要满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 未经特定事先书面许可，不得使用作者的名字来认可或推广基于此软件的产品
 *
 * 此软件由版权所有者和贡献者提供"按原样"，任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保，都是不被允许的。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、偶然、特殊、惩罚性或后果性损害，包括但不限于替代商品或服务的采购、使用损失、数据或利润损失，或业务中断，即使已被告知此类损害的可能性。
 *
 * 蓝牙数据结构
 * 作者：Paolo Abeni <paolo.abeni@email.it>
 */

#ifndef lib_pcap_bluetooth_h
#define lib_pcap_bluetooth_h

#include <pcap/pcap-inttypes.h>

/*
 * libpcap添加到每个蓝牙h4帧的头部，
 * 字段以网络字节顺序存储
 */
typedef struct _pcap_bluetooth_h4_header {
    uint32_t direction; /* 如果第一位设置了，方向是传入的 */
} pcap_bluetooth_h4_header;
/*
 * 在每个蓝牙 Linux 监视器帧之前添加的 libpcap 头部，
 * 字段以网络字节顺序存储
 */
typedef struct _pcap_bluetooth_linux_monitor_header {
    uint16_t adapter_id; // 适配器 ID
    uint16_t opcode; // 操作码
} pcap_bluetooth_linux_monitor_header; // 定义蓝牙 Linux 监视器帧头部结构体
#endif // 结束条件编译指令
```