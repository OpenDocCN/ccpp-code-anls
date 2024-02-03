# `nmap\libpcap\pcap-bt-linux.c`

```cpp
/*
 * 版权声明和许可声明
 * 作者：Paolo Abeni (Italy)
 * 版权声明：保留所有权利
 *
 * 在源代码和二进制形式下，无论是否经过修改，都允许重新分发和使用，前提是满足以下条件：
 *
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在提供的文档和/或其他材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名字来认可或推广基于此软件的产品。
 *
 * 此软件由版权所有者和贡献者提供"原样"，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他情况）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）承担任何责任。
 *
 * Linux平台的蓝牙嗅探API实现
 * 作者：Paolo Abeni <paolo.abeni@email.it>
 *
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "pcap-int.h"
#include "pcap-bt-linux.h"
#include "pcap/bluetooth.h"

#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#include <bluetooth/bluetooth.h>
#include <bluetooth/hci.h>
#define BT_IFACE "bluetooth"  // 定义蓝牙接口名称为"bluetooth"
#define BT_CTRL_SIZE 128  // 定义蓝牙控制大小为128

/* forward declaration */  // 前向声明

/*
 * Private data for capturing on Linux Bluetooth devices.
 */
struct pcap_bt {
    int dev_id;        /* device ID of device we're bound to */  // 设备ID，我们绑定的设备
};

int
bt_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    struct hci_dev_list_req *dev_list;  // 定义蓝牙设备列表请求结构体指针
    struct hci_dev_req *dev_req;  // 定义蓝牙设备请求结构体指针
    int sock;  // 定义套接字
    unsigned i;  // 无符号整型变量i
    int ret = 0;  // 定义返回值为0

    sock  = socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_HCI);  // 创建原始蓝牙套接字
    if (sock < 0)  // 如果套接字小于0
    {
        /* if bluetooth is not supported this is not fatal*/  // 如果蓝牙不受支持，这不是致命的
        if (errno == EAFNOSUPPORT)  // 如果错误号为EAFNOSUPPORT
            return 0;  // 返回0
        pcap_fmt_errmsg_for_errno(err_str, PCAP_ERRBUF_SIZE,
            errno, "Can't open raw Bluetooth socket");  // 格式化错误消息
        return -1;  // 返回-1
    }

    dev_list = malloc(HCI_MAX_DEV * sizeof(*dev_req) + sizeof(*dev_list));  // 分配蓝牙设备列表请求结构体的内存空间
    if (!dev_list)  // 如果dev_list为空
    {
        snprintf(err_str, PCAP_ERRBUF_SIZE, "Can't allocate %zu bytes for Bluetooth device list",
            HCI_MAX_DEV * sizeof(*dev_req) + sizeof(*dev_list));  // 格式化错误消息
        ret = -1;  // 返回-1
        goto done;  // 跳转到done标签
    }

    /*
     * Zero the complete header, which is larger than dev_num because of tail
     * padding, to silence Valgrind, which overshoots validating that dev_num
     * has been set.
     * https://github.com/the-tcpdump-group/libpcap/issues/1083
     * https://bugs.kde.org/show_bug.cgi?id=448464
     */
    memset(dev_list, 0, sizeof(*dev_list));  // 将dev_list清零
    dev_list->dev_num = HCI_MAX_DEV;  // 设置dev_num为HCI_MAX_DEV

    if (ioctl(sock, HCIGETDEVLIST, (void *) dev_list) < 0)  // 如果ioctl操作失败
    {
        pcap_fmt_errmsg_for_errno(err_str, PCAP_ERRBUF_SIZE,
            errno, "Can't get Bluetooth device list via ioctl");  // 格式化错误消息
        ret = -1;  // 返回-1
        goto free;  // 跳转到free标签
    }

    dev_req = dev_list->dev_req;  // 设置dev_req为dev_list的dev_req成员
    # 遍历设备列表中的每个设备
    for (i = 0; i < dev_list->dev_num; i++, dev_req++) {
        # 定义设备名称和设备描述的字符数组
        char dev_name[20], dev_descr[40];

        # 格式化设备名称，使用BT_IFACE作为前缀，加上设备ID
        snprintf(dev_name, sizeof(dev_name), BT_IFACE"%u", dev_req->dev_id);
        # 格式化设备描述，描述为"Bluetooth adapter number"加上设备序号
        snprintf(dev_descr, sizeof(dev_descr), "Bluetooth adapter number %u", i);

        '''
         * Bluetooth is a wireless technology.
         * XXX - if there's the notion of associating with a
         * network, and we can determine whether the interface
         * is associated with a network, check that and set
         * the status to PCAP_IF_CONNECTION_STATUS_CONNECTED
         * or PCAP_IF_CONNECTION_STATUS_DISCONNECTED.
         '''
        # 如果添加设备失败，则设置返回值为-1，并跳出循环
        if (add_dev(devlistp, dev_name, PCAP_IF_WIRELESS, dev_descr, err_str)  == NULL)
        {
            ret = -1;
            break;
        }
    }
// 释放设备列表的内存
free:
    free(dev_list);

// 关闭套接字
done:
    close(sock);
    // 返回结果
    return ret;
}

// 创建蓝牙抓包器
pcap_t *
bt_create(const char *device, char *ebuf, int *is_ours)
{
    const char *cp;
    char *cpend;
    long devnum;
    pcap_t *p;

    /* 检查设备名是否是蓝牙设备 */
    cp = strrchr(device, '/');
    if (cp == NULL)
        cp = device;
    /* 检查设备名是否以 BT_IFACE 开头 */
    if (strncmp(cp, BT_IFACE, sizeof BT_IFACE - 1) != 0) {
        /* 不是以 BT_IFACE 开头 */
        *is_ours = 0;
        return NULL;
    }
    /* 是以 BT_IFACE 开头，检查后面是否跟着数字 */
    cp += sizeof BT_IFACE - 1;
    devnum = strtol(cp, &cpend, 10);
    if (cpend == cp || *cpend != '\0') {
        /* 后面不是数字 */
        *is_ours = 0;
        return NULL;
    }
    if (devnum < 0) {
        /* 后面跟着无效的数字 */
        *is_ours = 0;
        return NULL;
    }

    /* 符合条件，设备可能是我们的 */
    *is_ours = 1;

    // 创建蓝牙抓包器
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_bt);
    if (p == NULL)
        return (NULL);

    // 设置激活操作为 bt_activate
    p->activate_op = bt_activate;
    return (p);
}

// 激活蓝牙抓包器
static int
bt_activate(pcap_t* handle)
{
    struct pcap_bt *handlep = handle->priv;
    struct sockaddr_hci addr;
    int opt;
    int dev_id;
    struct hci_filter flt;
    int err = PCAP_ERROR;

    /* 获取蓝牙接口的 ID */
    if (sscanf(handle->opt.device, BT_IFACE"%d", &dev_id) != 1)
    {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
            "Can't get Bluetooth device index from %s",
             handle->opt.device);
        return PCAP_ERROR;
    }

    /*
     * 将负的快照值（无效）、快照值为0（未指定）或大于正常最大值的值，转换为最大允许的值。
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     */
    # 如果快照长度小于等于0或者大于最大快照长度，则将快照长度设置为最大快照长度
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;

    # 初始化 pcap 结构的一些组件
    handle->bufsize = BT_CTRL_SIZE+sizeof(pcap_bluetooth_h4_header)+handle->snapshot;
    handle->linktype = DLT_BLUETOOTH_HCI_H4_WITH_PHDR;

    # 设置读取操作为 bt_read_linux
    handle->read_op = bt_read_linux;
    # 设置注入操作为 bt_inject_linux
    handle->inject_op = bt_inject_linux;
    # 设置过滤器操作为 install_bpf_program，表示没有内核过滤
    handle->setfilter_op = install_bpf_program;
    # 设置方向操作为 bt_setdirection_linux
    handle->setdirection_op = bt_setdirection_linux;
    # 设置数据链路操作为 NULL，表示无法更改数据链路类型
    handle->set_datalink_op = NULL;
    # 获取非阻塞操作为 pcap_getnonblock_fd
    handle->getnonblock_op = pcap_getnonblock_fd;
    # 设置非阻塞操作为 pcap_setnonblock_fd
    handle->setnonblock_op = pcap_setnonblock_fd;
    # 设置统计操作为 bt_stats_linux
    handle->stats_op = bt_stats_linux;
    # 设置设备 ID
    handlep->dev_id = dev_id;

    # 创建 HCI 套接字
    handle->fd = socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_HCI);
    # 如果创建套接字失败，则返回错误
    if (handle->fd < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't create raw socket");
        return PCAP_ERROR;
    }

    # 分配缓冲区
    handle->buffer = malloc(handle->bufsize);
    # 如果分配缓冲区失败，则返回错误
    if (!handle->buffer) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't allocate dump buffer");
        goto close_fail;
    }

    # 设置选项为1，启用数据方向信息
    opt = 1;
    if (setsockopt(handle->fd, SOL_HCI, HCI_DATA_DIR, &opt, sizeof(opt)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't enable data direction info");
        goto close_fail;
    }

    # 设置选项为1，启用时间戳
    opt = 1;
    if (setsockopt(handle->fd, SOL_HCI, HCI_TIME_STAMP, &opt, sizeof(opt)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't enable time stamp");
        goto close_fail;
    }

    # 设置过滤器，避免依赖外部库调用 hci 函数
    memset(&flt, 0, sizeof(flt));
    memset((void *) &flt.type_mask, 0xff, sizeof(flt.type_mask));
    # 使用memset函数将flt.event_mask的内容全部设置为0xff，大小为flt.event_mask的大小
    memset((void *) &flt.event_mask, 0xff, sizeof(flt.event_mask));
    # 如果无法设置过滤器，则将错误信息格式化到handle->errbuf中，并跳转到close_fail标签处
    if (setsockopt(handle->fd, SOL_HCI, HCI_FILTER, &flt, sizeof(flt)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't set filter");
        goto close_fail;
    }

    # 将地址族设置为AF_BLUETOOTH，将设备ID设置为handlep->dev_id
    addr.hci_family = AF_BLUETOOTH;
    addr.hci_dev = handlep->dev_id;
#ifdef HAVE_STRUCT_SOCKADDR_HCI_HCI_CHANNEL
    // 如果定义了 HAVE_STRUCT_SOCKADDR_HCI_HCI_CHANNEL，则设置 addr.hci_channel 为 HCI_CHANNEL_RAW
    addr.hci_channel = HCI_CHANNEL_RAW;
#endif
    // 将套接字与地址绑定，如果失败则设置错误信息并跳转到关闭失败的处理
    if (bind(handle->fd, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't attach to device %d", handlep->dev_id);
        goto close_fail;
    }

    if (handle->opt.rfmon) {
        /*
         * 监控模式不适用于蓝牙设备。
         */
        err = PCAP_ERROR_RFMON_NOTSUP;
        goto close_fail;
    }

    if (handle->opt.buffer_size != 0) {
        /*
         * 将套接字缓冲区大小设置为指定值。
         */
        if (setsockopt(handle->fd, SOL_SOCKET, SO_RCVBUF,
            &handle->opt.buffer_size,
            sizeof(handle->opt.buffer_size)) == -1) {
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                errno, PCAP_ERRBUF_SIZE, "SO_RCVBUF");
            goto close_fail;
        }
    }

    // 设置可选择的文件描述符为套接字文件描述符
    handle->selectable_fd = handle->fd;
    // 返回 0 表示成功
    return 0;

close_fail:
    // 清理资源并返回错误码
    pcap_cleanup_live_common(handle);
    return err;
}

static int
bt_read_linux(pcap_t *handle, int max_packets _U_, pcap_handler callback, u_char *user)
{
    struct cmsghdr *cmsg;
    struct msghdr msg;
    struct iovec  iv;
    ssize_t ret;
    struct pcap_pkthdr pkth;
    pcap_bluetooth_h4_header* bthdr;
    u_char *pktd;
    int in = 0;

    // 设置数据包指针和蓝牙头指针
    pktd = (u_char *)handle->buffer + BT_CTRL_SIZE;
    bthdr = (pcap_bluetooth_h4_header*)(void *)pktd;
    iv.iov_base = pktd + sizeof(pcap_bluetooth_h4_header);
    iv.iov_len  = handle->snapshot;

    // 初始化消息结构体
    memset(&msg, 0, sizeof(msg));
    msg.msg_iov = &iv;
    msg.msg_iovlen = 1;
    msg.msg_control = handle->buffer;
    msg.msg_controllen = BT_CTRL_SIZE;

    /* 忽略中断系统调用错误 */
    do {
        // 接收消息，如果中断循环并返回 -2
        ret = recvmsg(handle->fd, &msg, 0);
        if (handle->break_loop)
        {
            handle->break_loop = 0;
            return -2;
        }
    } while ((ret == -1) && (errno == EINTR));
    # 如果接收数据返回值小于 0
    if (ret < 0) {
        # 如果错误码是 EAGAIN 或者 EWOULDBLOCK，表示非阻塞模式下没有数据
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            /* Nonblocking mode, no data */
            return 0;
        }
        # 将错误信息格式化到 handle->errbuf 中
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't receive packet");
        return -1;
    }

    # 设置捕获数据包的长度为 ret
    pkth.caplen = (bpf_u_int32)ret;

    /* 获取数据包的方向和时间戳 */
    cmsg = CMSG_FIRSTHDR(&msg);
    while (cmsg) {
        switch (cmsg->cmsg_type) {
            case HCI_CMSG_DIR:
                # 复制数据包方向到 in
                memcpy(&in, CMSG_DATA(cmsg), sizeof in);
                break;
            case HCI_CMSG_TSTAMP:
                # 复制时间戳到 pkth.ts
                memcpy(&pkth.ts, CMSG_DATA(cmsg),
                    sizeof pkth.ts);
                break;
        }
        # 获取下一个控制消息头
        cmsg = CMSG_NXTHDR(&msg, cmsg);
    }
    # 根据捕获方向进行处理
    switch (handle->direction) {

    case PCAP_D_IN:
        # 如果方向是 PCAP_D_IN 且 in 为 0，则返回 0
        if (!in)
            return 0;
        break;

    case PCAP_D_OUT:
        # 如果方向是 PCAP_D_OUT 且 in 不为 0，则返回 0
        if (in)
            return 0;
        break;

    default:
        break;
    }

    # 设置 bthdr->direction 为网络字节序的 in
    bthdr->direction = htonl(in != 0);
    # 增加捕获数据包的长度
    pkth.caplen+=sizeof(pcap_bluetooth_h4_header);
    pkth.len = pkth.caplen;
    # 如果没有设置过滤器或者数据包通过了过滤器
    if (handle->fcode.bf_insns == NULL ||
        pcap_filter(handle->fcode.bf_insns, pktd, pkth.len, pkth.caplen)) {
        # 调用回调函数处理数据包
        callback(user, &pkth, pktd);
        return 1;
    }
    # 数据包未通过过滤器
    return 0;    /* didn't pass filter */
# 静态函数，用于在 Linux 平台上向蓝牙设备注入数据包，但此函数不支持蓝牙设备的数据包注入
static int
bt_inject_linux(pcap_t *handle, const void *buf _U_, int size _U_)
{
    # 将错误信息写入 handle->errbuf，说明蓝牙设备不支持数据包注入
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "Packet injection is not supported on Bluetooth devices");
    # 返回 -1，表示数据包注入不支持
    return (-1);
}

# 静态函数，用于获取蓝牙设备的统计信息
static int
bt_stats_linux(pcap_t *handle, struct pcap_stat *stats)
{
    # 获取 pcap_bt 结构体指针
    struct pcap_bt *handlep = handle->priv;
    int ret;
    struct hci_dev_info dev_info;
    struct hci_dev_stats * s = &dev_info.stat;
    dev_info.dev_id = handlep->dev_id;

    # 忽略 EINTR 错误
    do {
        # 通过 ioctl 获取蓝牙设备信息
        ret = ioctl(handle->fd, HCIGETDEVINFO, (void *)&dev_info);
    } while ((ret == -1) && (errno == EINTR));

    # 如果获取失败，将错误信息写入 handle->errbuf，并返回 -1
    if (ret < 0) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't get stats via ioctl");
        return (-1);
    }

    # 计算接收和丢弃的数据包数量，并将结果存入 stats 结构体
    stats->ps_recv = s->evt_rx + s->acl_rx + s->sco_rx + s->cmd_tx +
        s->acl_tx +s->sco_tx;
    stats->ps_drop = s->err_rx + s->err_tx;
    stats->ps_ifdrop = 0;
    # 返回 0，表示获取统计信息成功
    return 0;
}

# 静态函数，用于设置蓝牙设备的数据包捕获方向
static int
bt_setdirection_linux(pcap_t *p, pcap_direction_t d)
{
    # 在这一点上，可以保证 d 是一个有效的方向值
    # 设置 p 的数据包捕获方向为 d
    p->direction = d;
    # 返回 0，表示设置成功
    return 0;
}
```