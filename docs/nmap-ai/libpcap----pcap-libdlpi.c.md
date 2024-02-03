# `nmap\libpcap\pcap-libdlpi.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，需要保留以上版权声明和本段文字，包括在文档或其他材料中
 * 在包含二进制代码的发布中，需要在文档或其他材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料需要显示以下声明：
 * “本产品包含加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定书面许可，不得使用加利福尼亚大学或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 * 本代码由 Sagun Shakya (sagun.shakya@sun.com) 贡献
 */
/*
 * 使用 libdlpi 在 SunOS 5.11 下进行 DLPI 的数据包捕获
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/bufmod.h>
#include <sys/stream.h>
#include <libdlpi.h>
#include <errno.h>
#include <memory.h>
#include <stropts.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "pcap-int.h"
#include "dlpisubs.h"

/* 前向声明 */
static int dlpromiscon(pcap_t *, bpf_u_int32);
static int pcap_read_libdlpi(pcap_t *, int, pcap_handler, u_char *);
static int pcap_inject_libdlpi(pcap_t *, const void *, int);
static void pcap_libdlpi_err(const char *, const char *, int, char *);
static void pcap_cleanup_libdlpi(pcap_t *);
/*
 * list_interfaces() will list all the network links that are
 * available on a system.
 */
static boolean_t list_interfaces(const char *, void *);

typedef struct linknamelist {
    char    linkname[DLPI_LINKNAME_MAX];  // 定义存储网络链接名称的结构体
    struct linknamelist *lnl_next;  // 定义指向下一个链接名称的指针
} linknamelist_t;

typedef struct linkwalk {
    linknamelist_t    *lw_list;  // 定义存储链接名称列表的结构体指针
    int        lw_err;  // 定义存储错误信息的整型变量
} linkwalk_t;

/*
 * The caller of this function should free the memory allocated
 * for each linknamelist_t "entry" allocated.
 */
static boolean_t
list_interfaces(const char *linkname, void *arg)
{
    linkwalk_t    *lwp = arg;  // 将参数转换为linkwalk_t类型指针
    linknamelist_t    *entry;  // 定义存储链接名称的结构体指针

    if ((entry = calloc(1, sizeof(linknamelist_t))) == NULL) {  // 分配存储链接名称的结构体内存空间
        lwp->lw_err = ENOMEM;  // 如果分配内存失败，设置错误信息为ENOMEM
        return (B_TRUE);  // 返回真值
    }
    (void) pcap_strlcpy(entry->linkname, linkname, DLPI_LINKNAME_MAX);  // 将链接名称拷贝到结构体中

    if (lwp->lw_list == NULL) {  // 如果链接名称列表为空
        lwp->lw_list = entry;  // 将当前链接名称添加到列表中
    } else {
        entry->lnl_next = lwp->lw_list;  // 将当前链接名称指向原有列表的下一个链接名称
        lwp->lw_list = entry;  // 更新列表头指针
    }

    return (B_FALSE);  // 返回假值
}

static int
pcap_activate_libdlpi(pcap_t *p)
{
    struct pcap_dlpi *pd = p->priv;  // 获取pcap_t结构体中的私有数据
    int status = 0;  // 定义状态变量并初始化为0
    int retv;  // 定义返回值变量
    dlpi_handle_t dh;  // 定义DLPI句柄
    dlpi_info_t dlinfo;  // 定义DLPI信息结构体

    /*
     * Enable Solaris raw and passive DLPI extensions;
     * dlpi_open() will not fail if the underlying link does not support
     * passive mode. See dlpi(7P) for details.
     */
    retv = dlpi_open(p->opt.device, &dh, DLPI_RAW|DLPI_PASSIVE);  // 打开DLPI设备，启用原始和被动模式扩展
    # 如果返回值不是 DLPI_SUCCESS
    if (retv != DLPI_SUCCESS) {
        # 如果返回值是 DLPI_ELINKNAMEINVAL 或者 DLPI_ENOLINK
        if (retv == DLPI_ELINKNAMEINVAL || retv == DLPI_ENOLINK) {
            '''
            没有更多可说的，所以清除错误消息。
            '''
            status = PCAP_ERROR_NO_SUCH_DEVICE;
            p->errbuf[0] = '\0';
        } else if (retv == DL_SYSERR &&
            (errno == EPERM || errno == EACCES)) {
            status = PCAP_ERROR_PERM_DENIED;
            # 格式化错误消息
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open DLPI device failed with %s - root privilege may be required",
                (errno == EPERM) ? "EPERM" : "EACCES");
        } else {
            status = PCAP_ERROR;
            # 输出错误消息
            pcap_libdlpi_err(p->opt.device, "dlpi_open", retv,
                p->errbuf);
        }
        return (status);
    }
    # 设置 pd->dlpi_hd 为 dh
    pd->dlpi_hd = dh;

    # 如果选项中包含 rfmon
    if (p->opt.rfmon) {
        '''
        此设备存在，但我们不支持在支持 DLPI 的任何平台上的监控模式。
        '''
        status = PCAP_ERROR_RFMON_NOTSUP;
        # 跳转到 bad 标签
        goto bad;
    }

    # 使用 DLPI_ANY_SAP 进行绑定
    if ((retv = dlpi_bind(pd->dlpi_hd, DLPI_ANY_SAP, 0)) != DLPI_SUCCESS) {
        status = PCAP_ERROR;
        # 输出错误消息
        pcap_libdlpi_err(p->opt.device, "dlpi_bind", retv, p->errbuf);
        # 跳转到 bad 标签
        goto bad;
    }

    '''
     * 将负快照值（无效）、快照值为 0（未指定）或大于正常最大值的值转换为允许的最大值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该只增加 MAXIMUM_SNAPLEN。
     '''
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

    # 启用混杂模式
    # 如果设置了混杂模式
    if (p->opt.promisc) {
        # 尝试在物理层启用混杂模式
        retv = dlpromiscon(p, DL_PROMISC_PHYS);
        # 如果返回值小于0
        if (retv < 0) {
            # 如果是权限被拒绝
            if (retv == PCAP_ERROR_PERM_DENIED)
                # 设置状态为混杂模式权限被拒绝
                status = PCAP_ERROR_PROMISC_PERM_DENIED;
            else
                # 否则设置状态为返回值
                status = retv;
            # 跳转到错误处理
            goto bad;
        }
    } else {
        # 尝试启用多播模式
        retv = dlpromiscon(p, DL_PROMISC_MULTI);
        # 如果返回值小于0
        if (retv < 0) {
            # 设置状态为返回值
            status = retv;
            # 跳转到错误处理
            goto bad;
        }
    }

    # 尝试启用 SAP 混杂模式
    retv = dlpromiscon(p, DL_PROMISC_SAP);
    # 如果返回值小于0
    if (retv < 0) {
        # 如果之前的 DL_PROMISC_PHYS 模式有效
        if (p->opt.promisc)
            # 设置状态为警告
            status = PCAP_WARNING;
        else {
            # 否则设置状态为返回值
            status = retv;
            # 跳转到错误处理
            goto bad;
        }
    }

    # 确定链路类型
    if ((retv = dlpi_info(pd->dlpi_hd, &dlinfo, 0)) != DLPI_SUCCESS) {
        # 设置状态为错误
        status = PCAP_ERROR;
        # 记录错误信息
        pcap_libdlpi_err(p->opt.device, "dlpi_info", retv, p->errbuf);
        # 跳转到错误处理
        goto bad;
    }

    # 处理链路类型
    if (pcap_process_mactype(p, dlinfo.di_mactype) != 0) {
        # 设置状态为错误
        status = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }

    # 设置文件描述符为 DLPI 文件描述符
    p->fd = dlpi_fd(pd->dlpi_hd);

    # 推送和配置 bufmod
    if (pcap_conf_bufmod(p, p->snapshot) != 0) {
        # 设置状态为错误
        status = PCAP_ERROR;
        # 跳转到错误处理
        goto bad;
    }

    # 刷新读取端
    # 如果调用 ioctl 函数失败，则设置状态为错误，并格式化错误消息
    if (ioctl(p->fd, I_FLUSH, FLUSHR) != 0) {
        status = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "FLUSHR");
        goto bad;
    }

    # 分配数据缓冲区
    if (pcap_alloc_databuf(p) != 0) {
        status = PCAP_ERROR;
        goto bad;
    }

    """
     "p->fd" 是一个 STREAMS 设备的文件描述符，因此应该可以在其上使用 "select()" 和 "poll()"
    """
    p->selectable_fd = p->fd;

    # 设置读取操作为 pcap_read_libdlpi
    p->read_op = pcap_read_libdlpi;
    # 设置注入操作为 pcap_inject_libdlpi
    p->inject_op = pcap_inject_libdlpi;
    # 设置过滤操作为 install_bpf_program（没有内核过滤）
    p->setfilter_op = install_bpf_program;
    # 设置方向操作为 NULL（未实现）
    p->setdirection_op = NULL;
    # 设置数据链路类型操作为 NULL（无法更改数据链路类型）
    p->set_datalink_op = NULL;
    # 获取非阻塞操作为 pcap_getnonblock_fd
    p->getnonblock_op = pcap_getnonblock_fd;
    # 设置非阻塞操作为 pcap_setnonblock_fd
    p->setnonblock_op = pcap_setnonblock_fd;
    # 统计操作为 pcap_stats_dlpi
    p->stats_op = pcap_stats_dlpi;
    # 清理操作为 pcap_cleanup_libdlpi
    p->cleanup_op = pcap_cleanup_libdlpi;

    # 返回状态
    return (status);
static int
dlpromiscon(pcap_t *p, bpf_u_int32 level)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_dlpi *pd = p->priv;
    int retv;
    int err;

    // 调用 dlpi_promiscon 函数设置混杂模式
    retv = dlpi_promiscon(pd->dlpi_hd, level);
    // 如果设置失败
    if (retv != DLPI_SUCCESS) {
        // 如果是系统错误并且是权限错误
        if (retv == DL_SYSERR &&
            (errno == EPERM || errno == EACCES)) {
            // 如果是物理混杂模式
            if (level == DL_PROMISC_PHYS) {
                // 设置错误码和错误信息
                err = PCAP_ERROR_PROMISC_PERM_DENIED;
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "Attempt to set promiscuous mode failed with %s - root privilege may be required",
                    (errno == EPERM) ? "EPERM" : "EACCES");
            } else {
                // 设置错误码和错误信息
                err = PCAP_ERROR_PERM_DENIED;
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "Attempt to set %s mode failed with %s - root privilege may be required",
                    (level == DL_PROMISC_MULTI) ? "multicast" : "SAP promiscuous",
                    (errno == EPERM) ? "EPERM" : "EACCES");
            }
        } else {
            // 设置错误码和错误信息
            err = PCAP_ERROR;
            pcap_libdlpi_err(p->opt.device,
                "dlpi_promiscon" STRINGIFY(level),
                retv, p->errbuf);
        }
        // 返回错误码
        return (err);
    }
    // 设置成功，返回 0
    return (0);
}

/*
 * Presumably everything returned by dlpi_walk() is a DLPI device,
 * so there's no work to be done here to check whether name refers
 * to a DLPI device.
 */
static int
is_dlpi_interface(const char *name _U_)
{
    // 返回 1，表示是 DLPI 设备
    return (1);
}

static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
    /*
     * Nothing we can do other than mark loopback devices as "the
     * connected/disconnected status doesn't apply".
     *
     * XXX - on Solaris, can we do what the dladm command does,
     * i.e. get a connected/disconnected indication from a kstat?
     * (Note that you can also get the link speed, and possibly
     * other information, from a kstat as well.)
     */
    // 没有其他操作，返回 0
}
    # 检查是否是回环设备
    if (*flags & PCAP_IF_LOOPBACK) {
        '''
        回环设备不是无线设备，因此“连接”/“断开”不适用于它们。
        '''
        # 将连接状态标志设置为不适用
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        # 返回0表示成功
        return (0);
    }
    # 返回0表示成功
    return (0);
}
/*
 * 在Solaris中，“标准”机制即SIOCGLIFCONF只会找到已连接并且处于启用状态的网络链接。dlpi_walk(3DLPI)会找到系统中存在的额外网络链接。
 */
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    int retv = 0;

    linknamelist_t    *entry, *next;
    linkwalk_t    lw = {NULL, 0};
    int        save_errno;

    /*
     * 首先获取常规接口的列表。
     */
    if (pcap_findalldevs_interfaces(devlistp, errbuf,
        is_dlpi_interface, get_if_flags) == -1)
        return (-1);    /* 失败 */

    /* dlpi_walk() 用于回环将在这里添加。 */

    /*
     * 查找当前区域中的所有DLPI设备。
     *
     * XXX - pcap_findalldevs_interfaces()是否会在当前区域之外找到任何设备？如果不会，调用它的唯一原因是获取接口地址。
     */
    dlpi_walk(list_interfaces, &lw, 0);

    if (lw.lw_err != 0) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            lw.lw_err, "dlpi_walk");
        retv = -1;
        goto done;
    }

    /* 如果列表中不存在链接名，则添加链接名。 */
    for (entry = lw.lw_list; entry != NULL; entry = entry->lnl_next) {
        /*
         * 如果它尚未在设备列表中，则尝试添加它。
         */
        if (find_or_add_dev(devlistp, entry->linkname, 0, get_if_flags,
            NULL, errbuf) == NULL)
            retv = -1;
    }
done:
    save_errno = errno;
    for (entry = lw.lw_list; entry != NULL; entry = next) {
        next = entry->lnl_next;
        free(entry);
    }
    errno = save_errno;

    return (retv);
}

/*
 * 读取在DLPI句柄上接收的数据。如果被告知终止，则返回-2，否则返回读取的数据包数。
 */
static int
pcap_read_libdlpi(pcap_t *p, int count, pcap_handler callback, u_char *user)
{
    struct pcap_dlpi *pd = p->priv;
    int len;
    u_char *bufp;
    size_t msglen;
    int retv;
    # 获取数据包的数量
    len = p->cc;
    # 如果数据包数量不为0
    if (len != 0) {
        # 将数据包指针指向缓冲区指针
        bufp = p->bp;
        # 跳转到处理数据包的过程
        goto process_pkts;
    }
    # 循环处理数据包
    do {
        # 检查是否调用了"pcap_breakloop()"
        if (p->break_loop) {
            # 是 - 清除标志并返回-2表示被告知跳出循环
            p->break_loop = 0;
            return (-2);
        }
        # 获取消息长度和缓冲区指针
        msglen = p->bufsize;
        bufp = (u_char *)p->buffer + p->offset;
        # 接收数据链路层的数据包
        retv = dlpi_recv(pd->dlpi_hd, NULL, NULL, bufp, &msglen, -1, NULL);
        # 如果接收失败
        if (retv != DLPI_SUCCESS) {
            # 如果是调用终止循环的情况，不返回错误消息，而是检查上面是否调用了"pcap_breakloop()"
            if (retv == DL_SYSERR && errno == EINTR) {
                len = 0;
                continue;
            }
            # 返回错误消息
            pcap_libdlpi_err(dlpi_linkname(pd->dlpi_hd), "dlpi_recv", retv, p->errbuf);
            return (-1);
        }
        # 更新数据包长度
        len = msglen;
    } while (len == 0);
# 处理数据包
process_pkts:
    # 调用 pcap_process_pkts 函数处理数据包
    return (pcap_process_pkts(p, callback, user, count, bufp, len));
}

# 使用 DLPI 接口发送数据包
static int
pcap_inject_libdlpi(pcap_t *p, const void *buf, int size)
{
    # 获取 pcap_dlpi 结构体指针
    struct pcap_dlpi *pd = p->priv;
    int retv;

    # 使用 dlpi_send 函数发送数据包
    retv = dlpi_send(pd->dlpi_hd, NULL, 0, buf, size, NULL);
    # 如果发送失败，则记录错误信息并返回 -1
    if (retv != DLPI_SUCCESS) {
        pcap_libdlpi_err(dlpi_linkname(pd->dlpi_hd), "dlpi_send", retv,
            p->errbuf);
        return (-1);
    }
    # 返回发送的数据包大小
    /*
     * dlpi_send(3DLPI) does not provide a way to return the number of
     * bytes sent on the wire. Based on the fact that DLPI_SUCCESS was
     * returned we are assuming 'size' bytes were sent.
     */
    return (size);
}

# 关闭 DLPI 句柄
static void
pcap_cleanup_libdlpi(pcap_t *p)
{
    # 获取 pcap_dlpi 结构体指针
    struct pcap_dlpi *pd = p->priv;

    # 如果句柄不为空，则关闭句柄并重置相关参数
    if (pd->dlpi_hd != NULL) {
        dlpi_close(pd->dlpi_hd);
        pd->dlpi_hd = NULL;
        p->fd = -1;
    }
    # 调用公共的清理函数
    pcap_cleanup_live_common(p);
}

# 将错误消息写入缓冲区
static void
pcap_libdlpi_err(const char *linkname, const char *func, int err, char *errbuf)
{
    # 将错误消息格式化写入缓冲区
    snprintf(errbuf, PCAP_ERRBUF_SIZE, "libpcap: %s failed on %s: %s",
        func, linkname, dlpi_strerror(err));
}

# 创建接口
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    # 创建 pcap_t 结构体
    pcap_t *p;

    # 调用公共的创建函数，创建 pcap_t 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dlpi);
    # 如果创建失败，则返回空指针
    if (p == NULL)
        return (NULL);

    # 设置激活操作为 pcap_activate_libdlpi
    p->activate_op = pcap_activate_libdlpi;
    return (p);
}

# 获取 libpcap 版本字符串
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING);
}
```