# `nmap\libpcap\pcap-dbus.c`

```cpp
/*
 * 版权声明
 * 作者：Jakub Zawadzki
 * 版权所有
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 *
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名称来认可或推广基于此软件的产品。
 *
 * 此软件由版权所有者和贡献者提供，"按原样"提供，任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保，都是不被允许的。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者都不承担任何直接、间接、附带、特殊、惩罚性或后果性的损害责任，包括但不限于替代商品或服务的采购、使用损失、数据或利润损失，或业务中断，即使已被告知可能发生此类损害的可能性。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <string.h>

#include <time.h>
#include <sys/time.h>

#include <dbus/dbus.h>

#include "pcap-int.h"
#include "pcap-dbus.h"

/*
 * D-Bus捕获的私有数据
 */
struct pcap_dbus {
    DBusConnection *conn;
    u_int    packets_read;    /* 读取的数据包数量 */
};

static int
dbus_read(pcap_t *handle, int max_packets _U_, pcap_handler callback, u_char *user)
{
    // 获取D-Bus捕获的私有数据
    struct pcap_dbus *handlep = handle->priv;
    # 定义一个结构体变量，用于存储捕获数据包的头部信息
    struct pcap_pkthdr pkth;
    # 定义一个 DBusMessage 指针变量
    DBusMessage *message;

    # 定义一个指向字符的指针变量和一个整型变量，用于存储原始消息和其长度
    char *raw_msg;
    int raw_msg_len;

    # 定义一个整型变量，用于计数
    int count = 0;

    # 从连接中获取消息
    message = dbus_connection_pop_message(handlep->conn);

    # 当消息为空时循环
    while (!message) {
        # 设置连接的读写超时时间
        if (!dbus_connection_read_write(handlep->conn, 100)) {
            # 如果连接关闭，则设置错误信息并返回-1
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Connection closed");
            return -1;
        }

        # 如果需要中断循环，则设置标志并返回-2
        if (handle->break_loop) {
            handle->break_loop = 0;
            return -2;
        }

        # 从连接中获取消息
        message = dbus_connection_pop_message(handlep->conn);
    }

    # 如果消息是本地接口的断开信号，则设置错误信息并返回-1
    if (dbus_message_is_signal(message, DBUS_INTERFACE_LOCAL, "Disconnected")) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Disconnected");
        return -1;
    }

    # 将消息序列化为原始消息和其长度
    if (dbus_message_marshal(message, &raw_msg, &raw_msg_len)) {
        # 设置捕获数据包头部的长度和捕获长度
        pkth.caplen = pkth.len = raw_msg_len;
        # 获取当前时间并设置到捕获数据包头部的时间戳中
        gettimeofday(&pkth.ts, NULL);
        # 如果没有设置过滤器或者通过过滤器，则增加已读取数据包的计数，调用回调函数，并增加计数
        if (handle->fcode.bf_insns == NULL ||
            pcap_filter(handle->fcode.bf_insns, (u_char *)raw_msg, pkth.len, pkth.caplen)) {
            handlep->packets_read++;
            callback(user, &pkth, (u_char *)raw_msg);
            count++;
        }
        # 释放原始消息的内存
        dbus_free(raw_msg);
    }
    # 返回计数
    return count;
}
static int
dbus_write(pcap_t *handle, const void *buf, int size)
{
    /* XXX, not tested */
    // 获取 pcap_t 结构体中的私有数据 pcap_dbus
    struct pcap_dbus *handlep = handle->priv;

    // 初始化 DBusError 结构体
    DBusError error = DBUS_ERROR_INIT;
    // 创建一个 DBusMessage 结构体
    DBusMessage *msg;

    // 如果消息解组失败
    if (!(msg = dbus_message_demarshal(buf, size, &error))) {
        // 将错误信息写入错误缓冲区
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "dbus_message_demarshal() failed: %s", error.message);
        // 释放错误结构体
        dbus_error_free(&error);
        return -1;
    }

    // 发送消息到 D-Bus 连接
    dbus_connection_send(handlep->conn, msg, NULL);
    // 刷新 D-Bus 连接
    dbus_connection_flush(handlep->conn);

    // 释放消息
    dbus_message_unref(msg);
    return 0;
}

static int
dbus_stats(pcap_t *handle, struct pcap_stat *stats)
{
    // 获取 pcap_t 结构体中的私有数据 pcap_dbus
    struct pcap_dbus *handlep = handle->priv;

    // 设置接收数据包数和丢弃数据包数
    stats->ps_recv = handlep->packets_read;
    stats->ps_drop = 0;
    stats->ps_ifdrop = 0;
    return 0;
}

static void
dbus_cleanup(pcap_t *handle)
{
    // 获取 pcap_t 结构体中的私有数据 pcap_dbus
    struct pcap_dbus *handlep = handle->priv;

    // 释放 D-Bus 连接
    dbus_connection_unref(handlep->conn);

    // 清理 pcap_t 结构体的公共数据
    pcap_cleanup_live_common(handle);
}

/*
 * We don't support non-blocking mode.  I'm not sure what we'd
 * do to support it and, given that we don't support select()/
 * poll()/epoll_wait()/kevent() etc., it probably doesn't
 * matter.
 */
static int
dbus_getnonblock(pcap_t *p)
{
    // 设置错误信息，D-Bus 不支持非阻塞模式
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Non-blocking mode isn't supported for capturing on D-Bus");
    return (-1);
}

static int
dbus_setnonblock(pcap_t *p, int nonblock _U_)
{
    // 设置错误信息，D-Bus 不支持非阻塞模式
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Non-blocking mode isn't supported for capturing on D-Bus");
    return (-1);
}

static int
dbus_activate(pcap_t *handle)
{
    // 定义用于监听的规则
#define EAVESDROPPING_RULE "eavesdrop=true,"

    static const char *rules[] = {
        EAVESDROPPING_RULE "type='signal'",
        EAVESDROPPING_RULE "type='method_call'",
        EAVESDROPPING_RULE "type='method_return'",
        EAVESDROPPING_RULE "type='error'",
    };

    // 规则数量
    #define N_RULES sizeof(rules)/sizeof(rules[0])

    // 获取 pcap_t 结构体中的私有数据 pcap_dbus
    struct pcap_dbus *handlep = handle->priv;
    // 获取设备名称
    const char *dev = handle->opt.device;
    # 初始化一个 DBusError 对象
    DBusError error = DBUS_ERROR_INIT;
    # 定义一个无符号整数变量 i
    u_int i;

    # 如果设备名为 "dbus-system"
    if (strcmp(dev, "dbus-system") == 0) {
        # 获取系统总线连接，如果失败则设置错误信息并返回错误
        if (!(handlep->conn = dbus_bus_get(DBUS_BUS_SYSTEM, &error))) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Failed to get system bus: %s", error.message);
            dbus_error_free(&error);
            return PCAP_ERROR;
        }

    # 如果设备名为 "dbus-session"
    } else if (strcmp(dev, "dbus-session") == 0) {
        # 获取会话总线连接，如果失败则设置错误信息并返回错误
        if (!(handlep->conn = dbus_bus_get(DBUS_BUS_SESSION, &error))) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Failed to get session bus: %s", error.message);
            dbus_error_free(&error);
            return PCAP_ERROR;
        }

    # 如果设备名以 "dbus://" 开头
    } else if (strncmp(dev, "dbus://", 7) == 0) {
        # 获取地址信息
        const char *addr = dev + 7;
        # 打开到指定地址的连接，如果失败则设置错误信息并返回错误
        if (!(handlep->conn = dbus_connection_open(addr, &error))) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Failed to open connection to: %s: %s", addr, error.message);
            dbus_error_free(&error);
            return PCAP_ERROR;
        }
        # 注册总线，如果失败则设置错误信息并返回错误
        if (!dbus_bus_register(handlep->conn, &error)) {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Failed to register bus %s: %s\n", addr, error.message);
            dbus_error_free(&error);
            return PCAP_ERROR;
        }

    # 如果设备名不符合以上条件
    } else {
        # 设置错误信息并返回错误
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Can't get bus address from %s", handle->opt.device);
        return PCAP_ERROR;
    }

    # 初始化 pcap 结构的一些组件
    handle->bufsize = 0;
    handle->offset = 0;
    handle->linktype = DLT_DBUS;
    handle->read_op = dbus_read;
    handle->inject_op = dbus_write;
    handle->setfilter_op = install_bpf_program; /* XXX, later add support for dbus_bus_add_match() */
    handle->setdirection_op = NULL;
    handle->set_datalink_op = NULL;      /* can't change data link type */
    handle->getnonblock_op = dbus_getnonblock;
    handle->setnonblock_op = dbus_setnonblock;
    handle->stats_op = dbus_stats;
    # 设置 handle 对象的 cleanup_op 属性为 dbus_cleanup 函数
    handle->cleanup_op = dbus_cleanup;
#ifndef _WIN32
    /*
     * 在非 Windows 平台上，尝试在 D-Bus 连接上执行 select()/poll()/epoll_wait()/kevent()/等操作并不简单，
     * 不能简单地“给我一个要等待的 FD”。
     *
     * 显然，你需要使用 dbus_connection_set_watch_functions() 注册“添加监视”、“移除监视”和“切换监视”函数，
     * 保持一组 FD， 在“添加监视”函数中向该集合添加 FD，在“移除监视”函数中从中减去 FD，
     * 并在“切换监视”函数中向该集合添加或减去 FD，并在集合中等待 *所有* FD。
     * （是的，你需要“切换监视”函数，以便主循环本身不需要检查给定监视是否启用或禁用 - 大多数 libpcap 程序对 D-Bus 一无所知，
     * 也不应该对 D-Bus 了解任何东西，除了如何解码 D-Bus 消息之外。）
     *
     * 实现这将需要在 libpcap 导出“可选择的 FD”给其客户端的方式上做出相当大的改变。
     * 在这样做之前，我们只能说“你不能这样做”。
     */
    handle->selectable_fd = handle->fd = -1;
#endif

    if (handle->opt.rfmon) {
        /*
         * 监控模式不适用于 dbus 连接。
         */
        dbus_cleanup(handle);
        return PCAP_ERROR_RFMON_NOTSUP;
    }

    /*
     * 将负快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为 D-Bus 的最大消息长度（128MB）。
     */
    if (handle->snapshot <= 0 || handle->snapshot > 134217728)
        handle->snapshot = 134217728;

    /* dbus_connection_set_max_message_size(handlep->conn, handle->snapshot); */
    if (handle->opt.buffer_size != 0)
        dbus_connection_set_max_received_size(handlep->conn, handle->opt.buffer_size);
    # 遍历规则列表，逐个向 D-Bus 连接添加匹配规则
    for (i = 0; i < N_RULES; i++) {
        # 向 D-Bus 连接添加匹配规则
        dbus_bus_add_match(handlep->conn, rules[i], &error);
        # 如果出现错误
        if (dbus_error_is_set(&error)) {
            # 释放错误对象
            dbus_error_free(&error);

            /* try without eavesdrop */
            # 尝试在不使用窃听的情况下添加匹配规则
            dbus_bus_add_match(handlep->conn, rules[i] + strlen(EAVESDROPPING_RULE), &error);
            # 如果再次出现错误
            if (dbus_error_is_set(&error)) {
                # 格式化错误信息并存储到错误缓冲区
                snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "Failed to add bus match: %s\n", error.message);
                # 释放错误对象
                dbus_error_free(&error);
                # 清理 D-Bus 相关资源
                dbus_cleanup(handle);
                # 返回错误代码
                return PCAP_ERROR;
            }
        }
    }

    # 返回成功代码
    return 0;
}

pcap_t *
dbus_create(const char *device, char *ebuf, int *is_ours)
{
    pcap_t *p;

    // 检查设备名是否为 "dbus-system"、"dbus-session" 或以 "dbus://" 开头，如果不是则设置 is_ours 为 0 并返回空指针
    if (strcmp(device, "dbus-system") &&
        strcmp(device, "dbus-session") &&
        strncmp(device, "dbus://", 7))
    {
        *is_ours = 0;
        return NULL;
    }

    // 设置 is_ours 为 1
    *is_ours = 1;
    // 创建一个 pcap_t 结构体，类型为 struct pcap_dbus
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dbus);
    // 如果创建失败则返回空指针
    if (p == NULL)
        return (NULL);

    // 设置 activate_op 为 dbus_activate 函数
    p->activate_op = dbus_activate;
    /*
     * Set these up front, so that, even if our client tries to set non-blocking mode before we're activated, or query the state of non-blocking mode, they get an error, rather than having the non-blocking mode option set for use later.
     */
    // 设置 getnonblock_op 为 dbus_getnonblock 函数
    p->getnonblock_op = dbus_getnonblock;
    // 设置 setnonblock_op 为 dbus_setnonblock 函数
    p->setnonblock_op = dbus_setnonblock;
    // 返回创建的 pcap_t 结构体
    return (p);
}

int
dbus_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    /*
     * The notion of "connected" vs. "disconnected" doesn't apply.
     * XXX - what about the notions of "up" and "running"?
     */
    // 向设备列表中添加 "dbus-system" 和 "dbus-session" 两个设备
    if (add_dev(devlistp, "dbus-system",
        PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE, "D-Bus system bus",
        err_str) == NULL)
        return -1;
    if (add_dev(devlistp, "dbus-session",
        PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE, "D-Bus session bus",
        err_str) == NULL)
        return -1;
    // 返回成功
    return 0;
}
```