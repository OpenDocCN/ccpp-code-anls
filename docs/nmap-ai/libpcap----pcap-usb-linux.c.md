# `nmap\libpcap\pcap-usb-linux.c`

```
/*
 * 版权声明和许可声明
 * 作者：Paolo Abeni (Italy)
 * 版权所有
 *
 * 在源代码和二进制形式下，无论是否经过修改，都允许重新分发和使用，前提是满足以下条件：
 *
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定事先书面许可，不得使用作者的名称来认可或推广基于此软件的产品。
 *
 * 此软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）负责。
 *
 * Linux 平台的 USB 抓包 API 实现
 * 作者：Paolo Abeni <paolo.abeni@email.it>
 * 修改：Kris Katterjohn <katterjohn@gmail.com>
 *
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "pcap-int.h"
#include "pcap-usb-linux.h"
#include "pcap-usb-linux-common.h"
#include "pcap/usb.h"

#include "extract.h"

#ifdef NEED_STRERROR_H
#include "strerror.h"
#endif

#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <limits.h>
#include <string.h>
#include <dirent.h>
#include <byteswap.h>
#include <netinet/in.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/utsname.h>
#ifdef HAVE_LINUX_USBDEVICE_FS_H
/*
 * 如果需要 <linux/usbdevice_fs.h>，可能需要 <linux/compiler.h> 来定义 __user。
 */
#ifdef HAVE_LINUX_COMPILER_H
#include <linux/compiler.h>
#endif /* HAVE_LINUX_COMPILER_H */
#include <linux/usbdevice_fs.h>
#endif /* HAVE_LINUX_USBDEVICE_FS_H */

#include "diag-control.h"

#define USB_IFACE "usbmon"

#define USBMON_DEV_PREFIX "usbmon"
#define USBMON_DEV_PREFIX_LEN    (sizeof USBMON_DEV_PREFIX - 1)
#define USB_LINE_LEN 4096

#if __BYTE_ORDER == __LITTLE_ENDIAN
#define htols(s) s
#define htoll(l) l
#define htol64(ll) ll
#else
#define htols(s) bswap_16(s)
#define htoll(l) bswap_32(l)
#define htol64(ll) bswap_64(ll)
#endif

struct mon_bin_stats {
    uint32_t queued;
    uint32_t dropped;
};

struct mon_bin_get {
    pcap_usb_header *hdr;
    void *data;
    size_t data_len;   /* 数据长度（可以为零） */
};

struct mon_bin_mfetch {
    int32_t *offvec;   /* 获取的事件向量 */
    int32_t nfetch;    /* 要获取的事件数量（输出：已获取） */
    int32_t nflush;    /* 要刷新的事件数量 */
};

#define MON_IOC_MAGIC 0x92

#define MON_IOCQ_URB_LEN _IO(MON_IOC_MAGIC, 1)
#define MON_IOCX_URB  _IOWR(MON_IOC_MAGIC, 2, struct mon_bin_hdr)
#define MON_IOCG_STATS _IOR(MON_IOC_MAGIC, 3, struct mon_bin_stats)
#define MON_IOCT_RING_SIZE _IO(MON_IOC_MAGIC, 4)
#define MON_IOCQ_RING_SIZE _IO(MON_IOC_MAGIC, 5)
#define MON_IOCX_GET   _IOW(MON_IOC_MAGIC, 6, struct mon_bin_get)
#define MON_IOCX_MFETCH _IOWR(MON_IOC_MAGIC, 7, struct mon_bin_mfetch)
#define MON_IOCH_MFLUSH _IO(MON_IOC_MAGIC, 8)

#define MON_BIN_SETUP    0x1 /* 设置头部存在 */
#define MON_BIN_SETUP_ZERO    0x2 /* 设置缓冲区不可用 */
#define MON_BIN_DATA_ZERO    0x4 /* 数据缓冲区不可用 */
#define MON_BIN_ERROR    0x8
/*
 * Private data for capturing on Linux USB.
 */
struct pcap_usb_linux {
    u_char *mmapbuf;    /* memory-mapped region pointer */
    size_t mmapbuflen;    /* size of region */
    int bus_index;
    u_int packets_read;
};

/* forward declaration */
static int usb_activate(pcap_t *);
static int usb_stats_linux_bin(pcap_t *, struct pcap_stat *);
static int usb_read_linux_bin(pcap_t *, int , pcap_handler , u_char *);
static int usb_read_linux_mmap(pcap_t *, int , pcap_handler , u_char *);
static int usb_inject_linux(pcap_t *, const void *, int);
static int usb_setdirection_linux(pcap_t *, pcap_direction_t);
static void usb_cleanup_linux_mmap(pcap_t *);

/* facility to add an USB device to the device list*/
static int
usb_dev_add(pcap_if_list_t *devlistp, int n, char *err_str)
{
    char dev_name[10];
    char dev_descr[30];
    snprintf(dev_name, 10, USB_IFACE"%d", n);
    /*
     * XXX - is there any notion of "up" and "running"?
     */
    if (n == 0) {
        /*
         * As this refers to all buses, there's no notion of
         * "connected" vs. "disconnected", as that's a property
         * that would apply to a particular USB interface.
         */
        if (add_dev(devlistp, dev_name,
            PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
            "Raw USB traffic, all USB buses", err_str) == NULL)
            return -1;
    } else {
        /*
         * XXX - is there a way to determine whether anything's
         * plugged into this bus interface or not, and set
         * PCAP_IF_CONNECTION_STATUS_CONNECTED or
         * PCAP_IF_CONNECTION_STATUS_DISCONNECTED?
         */
        snprintf(dev_descr, 30, "Raw USB traffic, bus number %d", n);
        if (add_dev(devlistp, dev_name, 0, dev_descr, err_str) == NULL)
            return -1;
    }

    return 0;
}

int
usb_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    struct dirent* data;
    int ret = 0;
    DIR* dir;
    int n;
    char* name;
    # 我们需要 2.6.27 或更高版本的内核，因此我们有二进制模式支持。
    # 设备的形式为 /dev/usbmon{N}。
    # 打开 /dev 目录并扫描它。
    dir = opendir("/dev");
    if (dir != NULL) {
        while ((ret == 0) && ((data = readdir(dir)) != 0)) {
            name = data->d_name;
    
            # 这是一个 usbmon 设备吗？
            if (strncmp(name, USBMON_DEV_PREFIX,
                USBMON_DEV_PREFIX_LEN) != 0)
                continue;    # 不是
    
            # 这个设备的编号是多少？
            if (sscanf(&name[USBMON_DEV_PREFIX_LEN], "%d", &n) == 0)
                continue;    # 失败
    
            ret = usb_dev_add(devlistp, n, err_str);
        }
    
        closedir(dir);
    }
    return 0;
}
/*
 * 在 Linux 内核中的 mon_bin.c 中定义的内容匹配。
 */
#define MIN_RING_SIZE    (8*1024)  // 最小环大小为 8KB
#define MAX_RING_SIZE    (1200*1024)  // 最大环大小为 1200KB

static int
usb_set_ring_size(pcap_t* handle, int header_size)
{
    /*
     * 从二进制 usbmon 中抓取的数据包包括：
     *  1) 固定长度的头部，大小为 header_size；
     *  2) 描述符，用于等时传输；
     *  3) 负载。
     *
     * 内核缓冲区的大小，默认为 300KB，最小为 8KB，最大为 1200KB。
     * 大小通过 MON_IOCT_RING_SIZE ioctl 设置；传入的大小会向上舍入到页面大小。
     *
     * 最多保存 {缓冲区大小}/5 字节的负载。
     * 因此，如果我们从快照长度中减去固定长度大小，就得到了我们想要的最大负载（我们不担心描述符 - 如果有描述符，我们只需丢弃负载的最后一部分以使其适合）。
     * 我们将该结果乘以 5 并将缓冲区大小设置为该值。
     */
    int ring_size;

    if (handle->snapshot < header_size)  // 如果快照长度小于头部大小
        handle->snapshot = header_size;  // 将快照长度设置为头部大小
    /* 最大快照大小足够小，不会溢出 */
    ring_size = (handle->snapshot - header_size) * 5;  // 计算环大小

    /*
     * 这会产生错误吗？
     * （无法查询最小或最大值，因此我们只需从内核源代码中复制该值。我们不会将其舍入为页面大小的倍数。）
     */
    if (ring_size > MAX_RING_SIZE) {  // 如果环大小大于最大环大小
        /*
         * 是的。将环大小降低到最大值，并将快照长度设置为使我们获得最大大小环的值。
         */
        ring_size = MAX_RING_SIZE;  // 将环大小设置为最大环大小
        handle->snapshot = header_size + (MAX_RING_SIZE/5);  // 将快照长度设置为使我们获得最大大小环的值
    } else if (ring_size < MIN_RING_SIZE) {
        /*
         * 如果环形缓冲区大小小于最小值，将环形缓冲区大小提升至最小值，
         * 但保持快照长度不变，这样我们向回调函数展示的数据量不会超过快照长度所指定的范围。
         */
        ring_size = MIN_RING_SIZE;
    }

    // 使用 ioctl 函数设置文件描述符 handle->fd 对应的环形缓冲区大小
    if (ioctl(handle->fd, MON_IOCT_RING_SIZE, ring_size) == -1) {
        // 如果设置失败，将错误信息格式化到 handle->errbuf 中，并返回 -1
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't set ring size from fd %d", handle->fd);
        return -1;
    }
    // 返回环形缓冲区大小
    return ring_size;
}
# 定义一个静态函数，用于在 USB 设备上进行内存映射
static
int usb_mmap(pcap_t* handle)
{
    # 获取 pcap_t 结构体中的私有数据，即 pcap_usb_linux 结构体
    struct pcap_usb_linux *handlep = handle->priv;
    int len;

    """
    尝试根据快照长度设置环形缓冲区的大小，如果环形缓冲区比内核支持的还要大，则减小快照长度
    """
    len = usb_set_ring_size(handle, (int)sizeof(pcap_usb_header_mmapped));
    if (len == -1) {
        # 设置失败，回退到非内存映射访问
        return 0;
    }

    # 设置 pcap_usb_linux 结构体中的内存映射缓冲区大小
    handlep->mmapbuflen = len;
    # 使用 mmap() 函数创建内存映射
    handlep->mmapbuf = mmap(0, handlep->mmapbuflen, PROT_READ,
        MAP_SHARED, handle->fd, 0);
    if (handlep->mmapbuf == MAP_FAILED) {
        """
        失败。我们不将其视为致命错误，只是尝试回退到非内存映射访问
        """
        return 0;
    }
    return 1;
}

#ifdef HAVE_LINUX_USBDEVICE_FS_H

# 定义控制超时时间
#define CTRL_TIMEOUT    (5*1000)        /* milliseconds */

# 定义 USB 方向和类型
#define USB_DIR_IN        0x80
#define USB_TYPE_STANDARD    0x00
#define USB_RECIP_DEVICE    0x00

# 定义 USB 请求类型和描述符类型
#define USB_REQ_GET_DESCRIPTOR    6
#define USB_DT_DEVICE        1
#define USB_DT_CONFIG        2

# 定义 USB 设备描述符和配置描述符的大小
#define USB_DEVICE_DESCRIPTOR_SIZE    18
#define USB_CONFIG_DESCRIPTOR_SIZE    9

"""
探测连接到总线上的设备的描述符
这些描述符将出现在捕获的数据包流中，并由诸如 Wireshark 等外部应用程序进行解码
没有这些识别探测，数据包数据无法完全解码
"""
static void
probe_devices(int bus)
{
    struct usbdevfs_ctrltransfer ctrl;
    struct dirent* data;
    int ret = 0;
    char busdevpath[sizeof("/dev/bus/usb/000/") + NAME_MAX];
    DIR* dir;
    uint8_t descriptor[USB_DEVICE_DESCRIPTOR_SIZE];
    uint8_t configdesc[USB_CONFIG_DESCRIPTOR_SIZE];

    # 扫描 USB 总线目录以查找设备节点
    snprintf(busdevpath, sizeof(busdevpath), "/dev/bus/usb/%03d", bus);
    dir = opendir(busdevpath);
    if (!dir)
        return;
}
    # 当返回值大于等于 0 且读取到的数据不为空时执行循环
    while ((ret >= 0) && ((data = readdir(dir)) != 0)) {
        # 定义文件描述符和文件名变量
        int fd;
        char* name = data->d_name;

        # 如果文件名以 '.' 开头，则跳过本次循环
        if (name[0] == '.')
            continue;

        # 根据设备总线号和设备名构建设备路径
        snprintf(busdevpath, sizeof(busdevpath), "/dev/bus/usb/%03d/%s", bus, data->d_name);

        # 以读写方式打开设备路径对应的文件
        fd = open(busdevpath, O_RDWR);
        # 如果打开失败，则跳过本次循环
        if (fd == -1)
            continue;

        # 注释说明：不同的内核版本对于该结构体的成员名称可能不同
        /*
         * Sigh.  Different kernels have different member names
         * for this structure.
         */
#ifdef HAVE_STRUCT_USBDEVFS_CTRLTRANSFER_BREQUESTTYPE
        // 设置控制传输的请求类型为设备到主机的标准请求
        ctrl.bRequestType = USB_DIR_IN | USB_TYPE_STANDARD | USB_RECIP_DEVICE;
        // 设置控制传输的请求为获取描述符
        ctrl.bRequest = USB_REQ_GET_DESCRIPTOR;
        // 设置描述符类型为设备描述符
        ctrl.wValue = USB_DT_DEVICE << 8;
        // 设置索引为0
        ctrl.wIndex = 0;
        // 设置数据长度为描述符的大小
        ctrl.wLength = sizeof(descriptor);
#else
        // 设置控制传输的请求类型为设备到主机的标准请求
        ctrl.requesttype = USB_DIR_IN | USB_TYPE_STANDARD | USB_RECIP_DEVICE;
        // 设置控制传输的请求为获取描述符
        ctrl.request = USB_REQ_GET_DESCRIPTOR;
        // 设置描述符类型为设备描述符
        ctrl.value = USB_DT_DEVICE << 8;
        // 设置索引为0
        ctrl.index = 0;
        // 设置数据长度为描述符的大小
        ctrl.length = sizeof(descriptor);
#endif
        // 设置控制传输的数据为描述符
        ctrl.data = descriptor;
        // 设置控制传输的超时时间
        ctrl.timeout = CTRL_TIMEOUT;

        // 发送控制传输请求
        ret = ioctl(fd, USBDEVFS_CONTROL, &ctrl);

        /* Request CONFIGURATION descriptor alone to know wTotalLength */
#ifdef HAVE_STRUCT_USBDEVFS_CTRLTRANSFER_BREQUESTTYPE
        // 设置控制传输的请求为获取配置描述符
        ctrl.wValue = USB_DT_CONFIG << 8;
        // 设置数据长度为配置描述符的大小
        ctrl.wLength = sizeof(configdesc);
#else
        // 设置控制传输的请求为获取配置描述符
        ctrl.value = USB_DT_CONFIG << 8;
        // 设置数据长度为配置描述符的大小
        ctrl.length = sizeof(configdesc);
#endif
        // 设置控制传输的数据为配置描述符
        ctrl.data = configdesc;
        // 发送控制传输请求
        ret = ioctl(fd, USBDEVFS_CONTROL, &ctrl);
        if (ret >= 0) {
            uint16_t wtotallength;
            // 从配置描述符中提取wTotalLength字段
            wtotallength = EXTRACT_LE_U_2(&configdesc[2]);
#ifdef HAVE_STRUCT_USBDEVFS_CTRLTRANSFER_BREQUESTTYPE
            // 设置数据长度为wTotalLength
            ctrl.wLength = wtotallength;
#else
            // 设置数据长度为wTotalLength
            ctrl.length = wtotallength;
#endif
            // 分配内存用于存储wTotalLength大小的数据
            ctrl.data = malloc(wtotallength);
            if (ctrl.data) {
                // 发送控制传输请求
                ret = ioctl(fd, USBDEVFS_CONTROL, &ctrl);
                free(ctrl.data);
            }
        }
        // 关闭文件描述符
        close(fd);
    }
    // 关闭目录流
    closedir(dir);
}
#endif /* HAVE_LINUX_USBDEVICE_FS_H */

// 创建USB抓包设备
pcap_t *
usb_create(const char *device, char *ebuf, int *is_ours)
{
    const char *cp;
    char *cpend;
    long devnum;
    pcap_t *p;

    /* Does this look like a USB monitoring device? */
    // 检查设备名是否类似于USB监控设备
    cp = strrchr(device, '/');
    if (cp == NULL)
        cp = device;
    /* Does it begin with USB_IFACE? */
    // 检查设备名是否以USB_IFACE开头
    # 如果字符串 cp 不是以 USB_IFACE 开头，则执行以下操作
    if (strncmp(cp, USB_IFACE, sizeof USB_IFACE - 1) != 0) {
        # 不是以 USB_IFACE 开头，将 is_ours 设置为 0，并返回空指针
        *is_ours = 0;
        return NULL;
    }
    # 如果字符串以 USB_IFACE 开头，判断后面是否跟着一个数字
    cp += sizeof USB_IFACE - 1;
    devnum = strtol(cp, &cpend, 10);
    # 如果后面不是数字，将 is_ours 设置为 0，并返回空指针
    if (cpend == cp || *cpend != '\0') {
        *is_ours = 0;
        return NULL;
    }
    # 如果数字为负数，将 is_ours 设置为 0，并返回空指针
    if (devnum < 0) {
        *is_ours = 0;
        return NULL;
    }

    # 如果以上条件都不满足，将 is_ours 设置为 1
    *is_ours = 1;

    # 创建一个 pcap_usb_linux 结构体，并将其地址赋给 p
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_usb_linux);
    # 如果创建失败，返回空指针
    if (p == NULL)
        return (NULL);

    # 设置 p 的 activate_op 指向 usb_activate 函数
    p->activate_op = usb_activate;
    # 返回 p
    return (p);
    }
    
    static int
    usb_activate(pcap_t* handle)
    {
        // 获取 pcap_usb_linux 结构体指针
        struct pcap_usb_linux *handlep = handle->priv;
        // 定义存储完整路径的数组
        char        full_path[USB_LINE_LEN];

        /*
         * 将负的快照值（无效）、快照值为0（未指定）或大于正常最大值的值，转换为最大允许的值。
         *
         * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
         */
        if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
            handle->snapshot = MAXIMUM_SNAPLEN;

        /* 初始化 pcap 结构的一些组件。 */
        handle->bufsize = handle->snapshot;
        handle->offset = 0;
        handle->linktype = DLT_USB_LINUX;

        handle->inject_op = usb_inject_linux;
        handle->setfilter_op = install_bpf_program; /* 没有内核过滤 */
        handle->setdirection_op = usb_setdirection_linux;
        handle->set_datalink_op = NULL;    /* 无法更改数据链路类型 */
        handle->getnonblock_op = pcap_getnonblock_fd;
        handle->setnonblock_op = pcap_setnonblock_fd;

        /* 从设备名称中获取 USB 总线索引 */
        if (sscanf(handle->opt.device, USB_IFACE"%d", &handlep->bus_index) != 1)
        {
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Can't get USB bus index from %s", handle->opt.device);
            return PCAP_ERROR;
        }

        /*
         * 我们需要 2.6.27 或更高版本的内核，因此我们具有二进制模式支持。
         * 尝试打开二进制接口。
         */
        snprintf(full_path, USB_LINE_LEN, "/dev/"USBMON_DEV_PREFIX"%d",
            handlep->bus_index);
        handle->fd = open(full_path, O_RDONLY, 0);
        if (handle->fd < 0)
        {
            /*
             * 尝试失败了；为什么？
             */
            switch (errno) {

            case ENOENT:
                /*
                 * 设备不存在。
                 * 这可能意味着没有支持监视 USB 总线
                 * （这可能意味着“usbmon 模块未加载”），
                 * 或者是支持了但是*特定*设备不存在
                 * （如果总线索引为 0，则没有“扫描所有总线”设备，
                 * 如果总线索引不为 0，则没有这样的总线）。
                 *
                 * 目前，不提供错误消息；
                 * 如果我们能确定特定问题是什么，我们应该报告它。
                 */
                handle->errbuf[0] = '\0';
                return PCAP_ERROR_NO_SUCH_DEVICE;

            case EACCES:
                /*
                 * 我们没有权限打开它。
                 */
    // 如果出现格式截断错误，将错误信息写入错误缓冲区并返回权限被拒绝的错误代码
    DIAG_OFF_FORMAT_TRUNCATION
            snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open %s failed with EACCES - root privileges may be required",
                full_path);
    // 恢复格式截断错误检测
    DIAG_ON_FORMAT_TRUNCATION
            return PCAP_ERROR_PERM_DENIED;

        default:
            /*
             * 出现了其他错误。
             */
            // 根据错误号生成错误信息并写入错误缓冲区
            pcap_fmt_errmsg_for_errno(handle->errbuf,
                PCAP_ERRBUF_SIZE, errno,
                "Can't open USB bus file %s", full_path);
            // 返回通用错误代码
            return PCAP_ERROR;
        }
    }

    // 如果设置了混杂模式，关闭文件描述符并返回不支持混杂模式的错误代码
    if (handle->opt.rfmon)
    {
        /*
         * USB 设备不支持混杂模式。
         */
        close(handle->fd);
        return PCAP_ERROR_RFMON_NOTSUP;
    }

    /* 尝试使用快速内存映射访问 */
    if (usb_mmap(handle))
    {
        /* 成功。*/
        // 设置链路类型为 DLT_USB_LINUX_MMAPPED
        handle->linktype = DLT_USB_LINUX_MMAPPED;
        // 设置统计操作为 usb_stats_linux_bin
        handle->stats_op = usb_stats_linux_bin;
        // 设置读取操作为 usb_read_linux_mmap
        handle->read_op = usb_read_linux_mmap;
        // 设置清理操作为 usb_cleanup_linux_mmap
        handle->cleanup_op = usb_cleanup_linux_mmap;
        // 如果定义了 HAVE_LINUX_USBDEVICE_FS_H，调用 probe_devices() 函数
#ifdef HAVE_LINUX_USBDEVICE_FS_H
        probe_devices(handlep->bus_index);
#endif

        /*
         * "handle->fd" 是一个真实的文件，所以
         * "select()" 和 "poll()" 可以在其上工作。
         */
        // 设置可选择的文件描述符为 handle->fd，并返回成功代码
        handle->selectable_fd = handle->fd;
        return 0;
    }

    /*
     * 失败；尝试使用普通的二进制接口访问。
     *
     * 尝试根据快照长度设置环形缓冲区的大小，如果这样会使环形缓冲区大于内核支持的大小，则减小快照长度。
     */
    // 如果设置环形缓冲区大小失败，关闭文件描述符并返回通用错误代码
    if (usb_set_ring_size(handle, (int)sizeof(pcap_usb_header)) == -1) {
        /* 失败。*/
        close(handle->fd);
        return PCAP_ERROR;
    }
    // 设置统计操作为 usb_stats_linux_bin
    handle->stats_op = usb_stats_linux_bin;
    // 设置读取操作为 usb_read_linux_bin
    handle->read_op = usb_read_linux_bin;
    // 如果定义了 HAVE_LINUX_USBDEVICE_FS_H，调用 probe_devices() 函数
#ifdef HAVE_LINUX_USBDEVICE_FS_H
    probe_devices(handlep->bus_index);
#endif

    /*
     * "handle->fd" 是一个真实的文件，所以 "select()" 和 "poll()"
     * 可以在其上工作。
     */
    # 将句柄的可选择文件描述符设置为句柄的文件描述符
    handle->selectable_fd = handle->fd;

    /* 为了进行纯二进制访问和文本访问，我们需要分配读取缓冲区 */
    # 分配读取缓冲区的内存空间
    handle->buffer = malloc(handle->bufsize);
    # 如果内存分配失败
    if (!handle->buffer) {
        # 格式化错误消息，将错误信息存储到错误缓冲区中
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        # 关闭文件描述符
        close(handle->fd);
        # 返回错误代码
        return PCAP_ERROR;
    }
    # 返回成功代码
    return 0;
}
// 在 Linux 系统上无法支持 USB 设备的数据包注入，因此设置错误信息并返回 -1
static int
usb_inject_linux(pcap_t *handle, const void *buf _U_, int size _U_)
{
    snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
        "Packet injection is not supported on USB devices");
    return (-1);
}

// 设置 USB 设备的数据包方向
static int
usb_setdirection_linux(pcap_t *p, pcap_direction_t d)
{
    /*
     * 在这一点上，可以保证 d 是一个有效的方向值。
     */
    p->direction = d;
    return 0;
}

// 从 USB 设备获取统计信息
static int
usb_stats_linux_bin(pcap_t *handle, struct pcap_stat *stats)
{
    struct pcap_usb_linux *handlep = handle->priv;
    int ret;
    struct mon_bin_stats st;
    ret = ioctl(handle->fd, MON_IOCG_STATS, &st);
    if (ret < 0)
    {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't read stats from fd %d", handle->fd);
        return -1;
    }

    stats->ps_recv = handlep->packets_read + st.queued;
    stats->ps_drop = st.dropped;
    stats->ps_ifdrop = 0;
    return 0;
}

/*
 * 参考 <linux-kernel-source>/Documentation/usb/usbmon.txt 和
 * <linux-kernel-source>/drivers/usb/mon/mon_bin.c 二进制 ABI
 */
// 从 USB 设备读取数据包
static int
usb_read_linux_bin(pcap_t *handle, int max_packets _U_, pcap_handler callback, u_char *user)
{
    struct pcap_usb_linux *handlep = handle->priv;
    struct mon_bin_get info;
    int ret;
    struct pcap_pkthdr pkth;
    u_int clen = handle->snapshot - sizeof(pcap_usb_header);

    /* USB 头部将成为 'packet' 数据的一部分 */
    info.hdr = (pcap_usb_header*) handle->buffer;
    info.data = (u_char *)handle->buffer + sizeof(pcap_usb_header);
    info.data_len = clen;

    /* 忽略中断系统调用错误 */
    do {
        ret = ioctl(handle->fd, MON_IOCX_GET, &info);
        if (handle->break_loop)
        {
            handle->break_loop = 0;
            return -2;
        }
    } while ((ret == -1) && (errno == EINTR));
    if (ret < 0)
    {
        // 如果错误号是 EAGAIN，则表示没有数据可读
        if (errno == EAGAIN)
            return 0;    /* no data there */

        // 根据文件描述符和错误号生成错误消息
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't read from fd %d", handle->fd);
        // 返回读取失败
        return -1;
    }

    /*
     * info.hdr->data_len 是等时描述符（如果有）和提供的数据字节数之和。
     * 这里没有等时描述符，因为我们使用旧的 48 字节头。
     *
     * 如果 info.hdr->data_flag 非零，则没有 URB 数据；
     * info.hdr->urb_len 是要放置数据的缓冲区的大小；它不代表传输的数据量。
     * 如果 info.hdr->data_flag 为零，则存在 URB 数据，info.hdr->urb_len 是传输或接收的字节数；它不包括等时描述符。
     *
     * 内核可能会给我们比 snaplen 更多的数据；如果是这样，就减少数据长度，以便我们告诉客户端的总字节数不超过 snaplen。
     */
    if (info.hdr->data_len < clen)
        clen = info.hdr->data_len;
    info.hdr->data_len = clen;
    pkth.caplen = sizeof(pcap_usb_header) + clen;
    if (info.hdr->data_flag) {
        /*
         * 没有数据；只是基于 info.hdr->data_len 的长度（以便它 >= 捕获长度）。
         */
        pkth.len = sizeof(pcap_usb_header) + info.hdr->data_len;
    } else {
        /*
         * 我们得到了数据；基于 info.hdr->urb_len 的长度，以便它包括由于 USB 监视设备的缓冲区太小而丢弃的数据。
         */
        pkth.len = sizeof(pcap_usb_header) + info.hdr->urb_len;
    }
    pkth.ts.tv_sec = (time_t)info.hdr->ts_sec;
    pkth.ts.tv_usec = info.hdr->ts_usec;
    # 如果过滤器指令为空，或者通过过滤器检查，执行以下操作
    if (handle->fcode.bf_insns == NULL ||
        pcap_filter(handle->fcode.bf_insns, handle->buffer,
          pkth.len, pkth.caplen)) {
        # 增加已读取数据包的计数
        handlep->packets_read++;
        # 调用回调函数处理数据包
        callback(user, &pkth, handle->buffer);
        # 返回1表示通过过滤器
        return 1;
    }

    # 如果未通过过滤器，返回0
    return 0;    /* didn't pass filter */
# 定义一个常量 VEC_SIZE，表示向量的大小为32
#define VEC_SIZE 32
# 定义一个函数 usb_read_linux_mmap，用于从 Linux 内核中读取 USB 数据
static int
usb_read_linux_mmap(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user)
{
    # 获取 pcap_usb_linux 结构体指针
    struct pcap_usb_linux *handlep = handle->priv;
    # 定义一个 mon_bin_mfetch 结构体变量 fetch
    struct mon_bin_mfetch fetch;
    # 定义一个长度为32的整型数组 vec
    int32_t vec[VEC_SIZE];
    # 定义一个 pcap_pkthdr 结构体变量 pkth
    struct pcap_pkthdr pkth;
    # 定义一个指向无符号字符的指针 bp
    u_char *bp;
    # 定义一个 pcap_usb_header_mmapped 结构体指针 hdr
    pcap_usb_header_mmapped* hdr;
    # 定义一个整型变量 nflush，用于记录刷新的次数
    int nflush = 0;
    # 定义一个整型变量 packets，用于记录读取的数据包数量
    int packets = 0;
    # 定义一个无符号整型变量 clen 和最大长度 max_clen
    u_int clen, max_clen;

    # 计算最大长度
    max_clen = handle->snapshot - sizeof(pcap_usb_header_mmapped);

    }

    /* 刷新挂起的事件 */
    if (ioctl(handle->fd, MON_IOCH_MFLUSH, nflush) == -1) {
        # 如果刷新失败，设置错误信息并返回-1
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "Can't mflush fd %d", handle->fd);
        return -1;
    }
    # 返回读取的数据包数量
    return packets;
}

# 定义一个函数 usb_cleanup_linux_mmap，用于清理 Linux 内核中的 USB 数据
static void
usb_cleanup_linux_mmap(pcap_t* handle)
{
    # 获取 pcap_usb_linux 结构体指针
    struct pcap_usb_linux *handlep = handle->priv;

    /* 如果存在内存映射缓冲区，则取消映射 */
    if (handlep->mmapbuf != NULL) {
        munmap(handlep->mmapbuf, handlep->mmapbuflen);
        handlep->mmapbuf = NULL;
    }
    # 调用通用的清理函数
    pcap_cleanup_live_common(handle);
}
```