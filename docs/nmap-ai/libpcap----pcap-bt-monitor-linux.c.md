# `nmap\libpcap\pcap-bt-monitor-linux.c`

```cpp
/*
 * 版权声明，版权所有
 *
 * 在源代码和二进制形式下的重新分发和使用，无论是否经过修改，都是允许的，前提是满足以下条件：
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名字来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 *
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <errno.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

#include <bluetooth/bluetooth.h>
#include <bluetooth/hci.h>

#include "pcap/bluetooth.h"
#include "pcap-int.h"

#include "pcap-bt-monitor-linux.h"

#define BT_CONTROL_SIZE 32
#define INTERFACE_NAME "bluetooth-monitor"

/*
 * 私有数据
 * 目前不包含任何内容
 */
struct pcap_bt_monitor {
    int    dummy;
};
/*
 * Fields and alignment must match the declaration in the Linux kernel 3.4+.
 * See struct hci_mon_hdr in include/net/bluetooth/hci_mon.h.
 */
// 定义一个结构体，用于表示蓝牙监视器的头部信息
struct hci_mon_hdr {
    uint16_t opcode;  // 操作码
    uint16_t index;   // 索引
    uint16_t len;     // 长度
} __attribute__((packed));  // 确保结构体紧凑排列

// 查找所有蓝牙设备的函数
int
bt_monitor_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    int         ret = 0;  // 返回值初始化为0

    /*
     * Bluetooth is a wireless technology.
     *
     * This is a device to monitor all Bluetooth interfaces, so
     * there's no notion of "connected" or "disconnected", any
     * more than there's a notion of "connected" or "disconnected"
     * for the "any" device.
     */
    // 添加蓝牙监视器设备到设备列表
    if (add_dev(devlistp, INTERFACE_NAME,
                PCAP_IF_WIRELESS|PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
                "Bluetooth Linux Monitor", err_str) == NULL)
    {
        ret = -1;  // 如果添加失败，返回-1
    }

    return ret;  // 返回结果
}

// 读取蓝牙监视器数据的函数
static int
bt_monitor_read(pcap_t *handle, int max_packets _U_, pcap_handler callback, u_char *user)
{
    struct cmsghdr *cmsg;  // 控制消息头
    struct msghdr msg;     // 消息头
    struct iovec  iv[2];   // I/O向量
    ssize_t ret;           // 返回值
    struct pcap_pkthdr pkth;  // 数据包头
    pcap_bluetooth_linux_monitor_header *bthdr;  // 蓝牙监视器数据包头
    u_char *pktd;          // 数据包内容
    struct hci_mon_hdr hdr;  // 蓝牙监视器头部信息

    pktd = (u_char *)handle->buffer + BT_CONTROL_SIZE;  // 设置数据包内容的起始位置
    bthdr = (pcap_bluetooth_linux_monitor_header*)(void *)pktd;  // 蓝牙监视器数据包头的位置

    iv[0].iov_base = &hdr;  // 第一个I/O向量的基地址
    iv[0].iov_len = sizeof(hdr);  // 第一个I/O向量的长度
    iv[1].iov_base = pktd + sizeof(pcap_bluetooth_linux_monitor_header);  // 第二个I/O向量的基地址
    iv[1].iov_len = handle->snapshot;  // 第二个I/O向量的长度

    memset(&pkth.ts, 0, sizeof(pkth.ts));  // 清空数据包头的时间戳
    memset(&msg, 0, sizeof(msg));  // 清空消息头
    msg.msg_iov = iv;  // 设置消息的I/O向量
    msg.msg_iovlen = 2;  // 设置消息的I/O向量长度
    msg.msg_control = handle->buffer;  // 设置消息的控制信息
    msg.msg_controllen = BT_CONTROL_SIZE;  // 设置消息的控制信息长度

    // 接收消息
    do {
        ret = recvmsg(handle->fd, &msg, 0);  // 接收消息
        if (handle->break_loop)  // 如果需要中断循环
        {
            handle->break_loop = 0;  // 重置中断标志
            return -2;  // 返回-2
        }
    } while ((ret == -1) && (errno == EINTR));  // 循环直到接收成功或者出错
    # 如果接收数据返回值小于 0
    if (ret < 0) {
        # 如果错误码是 EAGAIN 或者 EWOULDBLOCK
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            # 非阻塞模式，没有数据
            return 0;
        }
        # 格式化错误消息，返回 -1
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't receive packet");
        return -1;
    }

    # 设置捕获数据包的长度
    pkth.caplen = (bpf_u_int32)(ret - sizeof(hdr) + sizeof(pcap_bluetooth_linux_monitor_header));
    pkth.len = pkth.caplen;

    # 遍历消息控制头
    for (cmsg = CMSG_FIRSTHDR(&msg); cmsg != NULL; cmsg = CMSG_NXTHDR(&msg, cmsg)) {
        # 如果消息控制头的级别不是 SOL_SOCKET，则继续下一个循环
        if (cmsg->cmsg_level != SOL_SOCKET) continue;

        # 如果消息控制头的类型是 SCM_TIMESTAMP
        if (cmsg->cmsg_type == SCM_TIMESTAMP) {
            # 复制时间戳数据到 pkth.ts
            memcpy(&pkth.ts, CMSG_DATA(cmsg), sizeof(pkth.ts));
        }
    }

    # 设置蓝牙头部的适配器 ID 和操作码
    bthdr->adapter_id = htons(hdr.index);
    bthdr->opcode = htons(hdr.opcode);

    # 如果过滤器为空或者通过过滤器
    if (handle->fcode.bf_insns == NULL ||
        pcap_filter(handle->fcode.bf_insns, pktd, pkth.len, pkth.caplen)) {
        # 调用回调函数，返回 1
        callback(user, &pkth, pktd);
        return 1;
    }
    # 没有通过过滤器，返回 0
    return 0;   /* didn't pass filter */
}
# 定义一个名为 bt_monitor_inject 的静态函数，用于在蓝牙监视设备上注入数据包
static int
bt_monitor_inject(pcap_t *handle, const void *buf _U_, int size _U_)
{
    # 将错误信息写入 handle->errbuf，说明蓝牙监视设备不支持数据包注入
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "Packet injection is not supported yet on Bluetooth monitor devices");
    # 返回 -1，表示数据包注入不支持
    return -1;
}

# 定义一个名为 bt_monitor_stats 的静态函数，用于获取蓝牙监视设备的统计信息
static int
bt_monitor_stats(pcap_t *handle _U_, struct pcap_stat *stats)
{
    # 将接收的数据包数量、丢弃的数据包数量和接口丢弃的数据包数量都设置为 0
    stats->ps_recv = 0;
    stats->ps_drop = 0;
    stats->ps_ifdrop = 0;

    # 返回 0，表示成功获取统计信息
    return 0;
}

# 定义一个名为 bt_monitor_activate 的静态函数，用于激活蓝牙监视设备
static int
bt_monitor_activate(pcap_t* handle)
{
    # 定义蓝牙地址结构体和错误码
    struct sockaddr_hci addr;
    int err = PCAP_ERROR;
    int opt;

    # 如果设置了 rfmon 选项，表示不支持监视模式
    if (handle->opt.rfmon) {
        /* monitor mode doesn't apply here */
        return PCAP_ERROR_RFMON_NOTSUP;
    }

    # 如果快照长度为负数、为 0 或者大于最大快照长度，将快照长度设置为最大快照长度
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;

    # 设置缓冲区大小和数据链路类型
    handle->bufsize = BT_CONTROL_SIZE + sizeof(pcap_bluetooth_linux_monitor_header) + handle->snapshot;
    handle->linktype = DLT_BLUETOOTH_LINUX_MONITOR;

    # 设置读取、注入、过滤、方向、数据链路类型等操作
    handle->read_op = bt_monitor_read;
    handle->inject_op = bt_monitor_inject;
    handle->setfilter_op = install_bpf_program; /* no kernel filtering */
    handle->setdirection_op = NULL; /* Not implemented */
    handle->set_datalink_op = NULL; /* can't change data link type */
    handle->getnonblock_op = pcap_getnonblock_fd;
    handle->setnonblock_op = pcap_setnonblock_fd;
    handle->stats_op = bt_monitor_stats;

    # 创建原始蓝牙套接字
    handle->fd = socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_HCI);
    if (handle->fd < 0) {
        # 如果创建套接字失败，将错误信息写入 handle->errbuf
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't create raw socket");
        # 返回错误码
        return PCAP_ERROR;
    }

    # 分配缓冲区内存
    handle->buffer = malloc(handle->bufsize);
    // 如果 handle->buffer 为空，则分配错误消息给 handle->errbuf，然后跳转到 close_fail 标签处
    if (!handle->buffer) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't allocate dump buffer");
        goto close_fail;
    }

    /* 将套接字绑定到 HCI 设备 */
    addr.hci_family = AF_BLUETOOTH;
    addr.hci_dev = HCI_DEV_NONE;
    addr.hci_channel = HCI_CHANNEL_MONITOR;

    // 如果绑定失败，则分配错误消息给 handle->errbuf，然后跳转到 close_fail 标签处
    if (bind(handle->fd, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't attach to interface");
        goto close_fail;
    }

    // 设置套接字选项为 SO_TIMESTAMP，如果设置失败，则分配错误消息给 handle->errbuf，然后跳转到 close_fail 标签处
    opt = 1;
    if (setsockopt(handle->fd, SOL_SOCKET, SO_TIMESTAMP, &opt, sizeof(opt)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't enable time stamp");
        goto close_fail;
    }

    // 将 handle->fd 赋值给 handle->selectable_fd
    handle->selectable_fd = handle->fd;

    // 返回 0 表示成功
    return 0;
// 关闭失败时的处理
close_fail:
    // 清理捕获设备的公共资源
    pcap_cleanup_live_common(handle);
    // 返回错误
    return err;
}

// 创建蓝牙监视器
pcap_t *
bt_monitor_create(const char *device, char *ebuf, int *is_ours)
{
    pcap_t      *p;
    const char  *cp;

    // 获取设备名中最后一个斜杠后的字符串
    cp = strrchr(device, '/');
    // 如果没有找到斜杠，则使用整个设备名
    if (cp == NULL)
        cp = device;

    // 如果设备名不是指定的接口名，则设置 is_ours 为 0，返回空指针
    if (strcmp(cp, INTERFACE_NAME) != 0) {
        *is_ours = 0;
        return NULL;
    }

    // 设置 is_ours 为 1
    *is_ours = 1;
    // 创建蓝牙监视器的公共资源
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_bt_monitor);
    // 如果创建失败，则返回空指针
    if (p == NULL)
        return NULL;

    // 设置激活操作为蓝牙监视器的激活函数
    p->activate_op = bt_monitor_activate;

    // 返回创建的蓝牙监视器
    return p;
}
```