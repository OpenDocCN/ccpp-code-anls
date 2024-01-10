# `nmap\libpcap\pcap-haiku.cpp`

```
/*
 * 版权声明
 * 作者
 */


#include "config.h"  // 包含配置文件
#include "pcap-int.h"  // 包含 pcap 内部头文件

#include <OS.h>  // 包含 Haiku 操作系统头文件

#include <sys/socket.h>  // 包含套接字相关头文件
#include <sys/sockio.h>  // 包含套接字 I/O 相关头文件

#include <net/if.h>  // 包含网络接口相关头文件
#include <net/if_dl.h>  // 包含数据链路层相关头文件
#include <net/if_types.h>  // 包含网络接口类型相关头文件

#include <errno.h>  // 包含错误处理相关头文件
#include <stdio.h>  // 包含标准输入输出相关头文件
#include <stdlib.h>  // 包含标准库相关头文件
#include <string.h>  // 包含字符串处理相关头文件


/*
 * Haiku 套接字捕获的私有数据
 */
struct pcap_haiku {
    struct pcap_stat    stat;  // 统计信息
    char    *device;    /* 设备名称 */
};


bool
prepare_request(struct ifreq& request, const char* name)
{
    if (strlen(name) >= IF_NAMESIZE)  // 如果设备名称长度超过限制
        return false;  // 返回 false

    strcpy(request.ifr_name, name);  // 复制设备名称到请求结构体
    return true;  // 返回 true
}


static int
pcap_read_haiku(pcap_t* handle, int maxPackets _U_, pcap_handler callback,
    u_char* userdata)
{
    // 接收单个数据包

    u_char* buffer = (u_char*)handle->buffer + handle->offset;  // 设置缓冲区
    struct sockaddr_dl from;  // 定义数据链路层地址结构
    ssize_t bytesReceived;  // 接收到的字节数
    do {
        if (handle->break_loop) {
            // 清除中断循环标志，并返回 -2 表示中断原因
            handle->break_loop = 0;
            return -2;
        }

        socklen_t fromLength = sizeof(from);  // 设置地址结构长度
        bytesReceived = recvfrom(handle->fd, buffer, handle->bufsize, MSG_TRUNC,
            (struct sockaddr*)&from, &fromLength);  // 接收数据包
    } while (bytesReceived < 0 && errno == B_INTERRUPTED);  // 如果接收到的字节数小于 0 并且错误码为中断

    if (bytesReceived < 0) {
        if (errno == B_WOULD_BLOCK) {
            // 没有数据包
            return 0;
        }

        snprintf(handle->errbuf, sizeof(handle->errbuf),
            "recvfrom: %s", strerror(errno));  // 格式化错误信息
        return -1;  // 返回 -1
    }

    int32 captureLength = bytesReceived;  // 设置捕获长度为接收到的字节数
    if (captureLength > handle->snapshot)  // 如果捕获长度大于快照长度
        captureLength = handle->snapshot;  // 设置捕获长度为快照长度

    // 运行数据包过滤器
    // 如果过滤器存在，则使用 pcap_filter 进行过滤
    if (handle->fcode.bf_insns) {
        if (pcap_filter(handle->fcode.bf_insns, buffer, bytesReceived,
                captureLength) == 0) {
            // 如果数据包被拒绝，则返回 0
            return 0;
        }
    }

    // 填充 pcap 包头信息
    pcap_pkthdr header;
    header.caplen = captureLength;
    header.len = bytesReceived;
    header.ts.tv_usec = system_time() % 1000000;
    header.ts.tv_sec = system_time() / 1000000;
    // TODO: 从数据包中获取时间信息!!!

    /* 调用用户提供的回调函数 */
    callback(userdata, &header, buffer);
    // 返回 1
    return 1;
# Haiku系统下的 pcap_inject 函数，用于向网络发送数据包
static int
pcap_inject_haiku(pcap_t *handle, const void *buffer, int size)
{
    # 目前不支持发送数据包
    # TODO: 使用 AF_LINK 协议（需要另一个套接字）来发送数据包
    strlcpy(handle->errbuf, "Sending packets isn't supported yet",
        PCAP_ERRBUF_SIZE);
    return -1;
}


# Haiku系统下的 pcap_stats 函数，用于获取网络接口的统计信息
static int
pcap_stats_haiku(pcap_t *handle, struct pcap_stat *stats)
{
    # 获取 pcap_haiku 结构体指针
    struct pcap_haiku* handlep = (struct pcap_haiku*)handle->priv;
    # 创建一个套接字
    int socket = ::socket(AF_INET, SOCK_DGRAM, 0);
    if (socket < 0) {
        return -1;
    }
    # 准备请求信息
    prepare_request(request, handlep->device);
    # 获取网络接口的统计信息
    if (ioctl(socket, SIOCGIFSTATS, &request, sizeof(struct ifreq)) < 0) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "pcap_stats: %s",
            strerror(errno));
        close(socket);
        return -1;
    }

    close(socket);
    # 更新统计信息
    handlep->stat.ps_recv += request.ifr_stats.receive.packets;
    handlep->stat.ps_drop += request.ifr_stats.receive.dropped;
    *stats = handlep->stat;
    return 0;
}


# Haiku系统下的 pcap_activate 函数，用于激活网络接口
static int
pcap_activate_haiku(pcap_t *handle)
{
    # 获取 pcap_haiku 结构体指针
    struct pcap_haiku* handlep = (struct pcap_haiku*)handle->priv;

    # 获取设备名称
    const char* device = handle->opt.device;

    # 设置读取操作函数
    handle->read_op = pcap_read_haiku;
    # 设置过滤操作函数
    handle->setfilter_op = install_bpf_program; /* no kernel filtering */
    # 设置发送数据包操作函数
    handle->inject_op = pcap_inject_haiku;
    # 设置获取统计信息操作函数
    handle->stats_op = pcap_stats_haiku;

    # 在可能的情况下使用默认钩子
    handle->getnonblock_op = pcap_getnonblock_fd;
    handle->setnonblock_op = pcap_setnonblock_fd;

    '''
     * 将负的快照值（无效）、快照值为0（未指定）或大于正常最大值的值，
     * 转换为允许的最大值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     '''
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;
    // 为 handlep 结构体中的 device 成员分配内存，并将 device 的值复制给它
    handlep->device    = strdup(device);
    // 如果分配内存失败，则设置错误信息并返回错误码
    if (handlep->device == NULL) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "strdup");
        return PCAP_ERROR;
    }

    // 设置 handle 结构体中的 bufsize 成员为 65536
    // TODO: 应该根据接口的最大传输单元（MTU）来确定
    handle->bufsize = 65536;

    // 为监视设备分配缓冲区
    handle->buffer = (u_char*)malloc(handle->bufsize);
    // 如果分配内存失败，则设置错误信息并返回错误码
    if (handle->buffer == NULL) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "buffer malloc");
        return PCAP_ERROR;
    }

    // 设置 handle 结构体中的 offset 成员为 0
    handle->offset = 0;
    // 设置 handle 结构体中的 linktype 成员为 DLT_EN10MB
    // TODO: 检查接口类型！
    handle->linktype = DLT_EN10MB;

    // 返回成功
    return 0;
// 结束函数定义
}

// 声明外部的 pcap_create_interface 函数，参数为设备名和错误缓冲区
extern "C" pcap_t *
pcap_create_interface(const char *device, char *errorBuffer)
{
    // TODO: 处理混杂模式

    // 需要一个套接字来与网络堆栈通信
    int socket = ::socket(AF_INET, SOCK_DGRAM, 0);
    // 如果套接字创建失败
    if (socket < 0) {
        // 将错误信息写入错误缓冲区
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE,
            "网络堆栈似乎不可用。\n");
        return NULL;
    }

    struct ifreq request;
    // 准备请求结构体
    if (!prepare_request(request, device)) {
        // 如果准备请求失败，将错误信息写入错误缓冲区
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE,
            "接口名 \"%s\" 太长。", device);
        // 关闭套接字并返回空指针
        close(socket);
        return NULL;
    }

    // 检查接口是否存在
    if (ioctl(socket, SIOCGIFINDEX, &request, sizeof(request)) < 0) {
        // 如果接口不存在，将错误信息写入错误缓冲区
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE,
            "接口 \"%s\" 不存在。\n", device);
        // 关闭套接字并返回空指针
        close(socket);
        return NULL;
    }

    // 关闭套接字，此后不再需要

    // 获取该接口的链路级接口
    socket = ::socket(AF_LINK, SOCK_DGRAM, 0);
    // 如果获取链路级接口失败
    if (socket < 0) {
        // 将错误信息写入错误缓冲区
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE, "没有链路级接口: %s\n",
            strerror(errno));
        return NULL;
    }

    // 开始监视
    if (ioctl(socket, SIOCSPACKETCAP, &request, sizeof(struct ifreq)) < 0) {
        // 如果无法开始监视，将错误信息写入错误缓冲区
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE, "无法开始监视: %s\n",
            strerror(errno));
        // 关闭套接字并返回空指针
        close(socket);
        return NULL;
    }

    // 创建 pcap_t 结构体
    struct wrapper_struct { pcap_t __common; struct pcap_haiku __private; };
    pcap_t* handle = pcap_create_common(errorBuffer,
        sizeof (struct wrapper_struct),
        offsetof (struct wrapper_struct, __private));

    // 如果创建失败，将错误信息写入错误缓冲区
    if (handle == NULL) {
        snprintf(errorBuffer, PCAP_ERRBUF_SIZE, "malloc: %s", strerror(errno));
        // 关闭套接字并返回空指针
        close(socket);
        return NULL;
    }

    // 设置可选择的文件描述符和文件描述符
    handle->selectable_fd = socket;
    handle->fd = socket;

    // 设置激活操作为 pcap_activate_haiku
    handle->activate_op = pcap_activate_haiku;
    # 返回变量 handle 的值
    return handle;
}

static int
can_be_bound(const char *name _U_)
{
    return 1;
}

static int
get_if_flags(const char *name, bpf_u_int32 *flags, char *errbuf)
{
    /* TODO */
    // 如果标志中包含 PCAP_IF_LOOPBACK
    if (*flags & PCAP_IF_LOOPBACK) {
        /*
         * Loopback devices aren't wireless, and "connected"/
         * "disconnected" doesn't apply to them.
         */
        // 将标志设置为 PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        return (0);
    }
    // 返回 0
    return (0);
}

// 导出函数，用于查找设备
extern "C" int
pcap_platform_finddevs(pcap_if_list_t* _allDevices, char* errorBuffer)
{
    // 调用 pcap_findalldevs_interfaces 函数
    return pcap_findalldevs_interfaces(_allDevices, errorBuffer, can_be_bound,
        get_if_flags);
}

/*
 * Libpcap version string.
 */
// 导出函数，返回 Libpcap 版本字符串
extern "C" const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING);
}
```