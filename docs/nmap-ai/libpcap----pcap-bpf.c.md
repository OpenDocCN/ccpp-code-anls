# `nmap\libpcap\pcap-bpf.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，需要保留以上版权声明和本段文字
 * 在包含二进制代码的发布中，需要在文档或其他提供的材料中保留以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都需要显示以下声明：
 * “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>            /* optionally get BSD define */
#include <sys/socket.h>
#include <time.h>
/*
 * <net/bpf.h> defines ioctls, but doesn't include <sys/ioccom.h>.
 *
 * We include <sys/ioctl.h> as it might be necessary to declare ioctl();
 * at least on *BSD and macOS, it also defines various SIOC ioctls -
 * we could include <sys/sockio.h>, but if we're already including
 * <sys/ioctl.h>, which includes <sys/sockio.h> on those platforms,
 * there's not much point in doing so.
 *
 * If we have <sys/ioccom.h>, we include it as well, to handle systems
 * such as Solaris which don't arrange to include <sys/ioccom.h> if you
 * include <sys/ioctl.h>
 */
#include <sys/ioctl.h>
#ifdef HAVE_SYS_IOCCOM_H
#include <sys/ioccom.h>
#endif
#include <sys/utsname.h>

#if defined(__FreeBSD__) && defined(SIOCIFCREATE2)
/*
 * 在 FreeBSD 上添加对 usbusN 接口的捕获支持
 */
static const char usbus_prefix[] = "usbus";
#define USBUS_PREFIX_LEN    (sizeof(usbus_prefix) - 1)
#include <dirent.h>
#endif

#include <net/if.h>

#ifdef _AIX

/*
 * 防止 "pcap.h" 包含 "pcap/bpf.h"；我们将包含本地操作系统版本，因为我们需要其中的 "struct bpf_config"。
 */
#define PCAP_DONT_INCLUDE_PCAP_BPF_H

#include <sys/types.h>

/*
 * 防止 bpf.h 重新定义 DLT_ 值为它们的 IFT_ 值，因为我们将返回标准 libpcap 值，而不是 IBM 的非标准 IFT_ 值。
 */
#undef _AIX
#include <net/bpf.h>
#define _AIX

/*
 * 如果同时定义了 BIOCROTZBUF 和 BPF_BUFMODE_ZBUF，则表示有零拷贝 BPF。
 */
#if defined(BIOCROTZBUF) && defined(BPF_BUFMODE_ZBUF)
  #define HAVE_ZEROCOPY_BPF
  #include <sys/mman.h>
  #include <machine/atomic.h>
#endif

#include <net/if_types.h>        /* 用于 IFT_ 值 */
#include <sys/sysconfig.h>
#include <sys/device.h>
#include <sys/cfgodm.h>
#include <cf.h>

#ifdef __64BIT__
#define domakedev makedev64
#define getmajor major64
#define bpf_hdr bpf_hdr32
#else /* __64BIT__ */
#define domakedev makedev
#define getmajor major
#endif /* __64BIT__ */

#define BPF_NAME "bpf"
#define BPF_MINORS 4
#define DRIVER_PATH "/usr/lib/drivers"
#define BPF_NODE "/dev/bpf"
static int bpfloadedflag = 0;
static int odmlockid = 0;

static int bpf_load(char *errbuf);

#else /* _AIX */

#include <net/bpf.h>

#endif /* _AIX */

#include <fcntl.h>
#include <errno.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#ifdef SIOCGIFMEDIA
# include <net/if_media.h>
#endif

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * NetBSD 的较新版本在 FDDI 帧前面插入填充以将 IP 头对齐到4字节边界。
 */
#if defined(__NetBSD__) && __NetBSD_Version__ > 106000000
#define       PCAP_FDDIPAD 3
#endif



/*
 * Private data for capturing on BPF devices.
 */
struct pcap_bpf {
#ifdef HAVE_ZEROCOPY_BPF
    /*
     * Zero-copy read buffer -- for zero-copy BPF.  'buffer' above will
     * alternative between these two actual mmap'd buffers as required.
     * As there is a header on the front size of the mmap'd buffer, only
     * some of the buffer is exposed to libpcap as a whole via bufsize;
     * zbufsize is the true size.  zbuffer tracks the current zbuf
     * associated with buffer so that it can be used to decide which the
     * next buffer to read will be.
     */
    u_char *zbuf1, *zbuf2, *zbuffer;
    u_int zbufsize;
    u_int zerocopy;
    u_int interrupted;
    struct timespec firstsel;
    /*
     * If there's currently a buffer being actively processed, then it is
     * referenced here; 'buffer' is also pointed at it, but offset by the
     * size of the header.
     */
    struct bpf_zbuf_header *bzh;
    int nonblock;        /* true if in nonblocking mode */
#endif /* HAVE_ZEROCOPY_BPF */

    char *device;        /* device name */
    int filtering_in_kernel; /* using kernel filter */
    int must_do_on_close;    /* stuff we must do when we close */
};

/*
 * Stuff to do when we close.
 */
#define MUST_CLEAR_RFMON    0x00000001    /* clear rfmon (monitor) mode */
#define MUST_DESTROY_USBUS    0x00000002    /* destroy usbusN interface */

#ifdef BIOCGDLTLIST
# if (defined(HAVE_NET_IF_MEDIA_H) && defined(IFM_IEEE80211)) && !defined(__APPLE__)
#define HAVE_BSD_IEEE80211

/*
 * The ifm_ulist member of a struct ifmediareq is an int * on most systems,
 * but it's a uint64_t on newer versions of OpenBSD.
 *
 * We check this by checking whether IFM_GMASK is defined and > 2^32-1.
 */
#  if defined(IFM_GMASK) && IFM_GMASK > 0xFFFFFFFF
#    define IFM_ULIST_TYPE    uint64_t
#  else
#    define IFM_ULIST_TYPE    int
#  endif
# endif
# if defined(__APPLE__) || defined(HAVE_BSD_IEEE80211)
static int find_802_11(struct bpf_dltlist *);
#  ifdef HAVE_BSD_IEEE80211
static int monitor_mode(pcap_t *, int);
#  endif
#  if defined(__APPLE__)
static void remove_non_802_11(pcap_t *);
static void remove_802_11(pcap_t *);
#  endif
# endif /* defined(__APPLE__) || defined(HAVE_BSD_IEEE80211) */
#endif /* BIOCGDLTLIST */

#if defined(sun) && defined(LIFNAMSIZ) && defined(lifr_zoneid)
#include <zone.h>
#endif

/*
 * We include the OS's <net/bpf.h>, not our "pcap/bpf.h", so we probably
 * don't get DLT_DOCSIS defined.
 */
#ifndef DLT_DOCSIS
#define DLT_DOCSIS    143
#endif

/*
 * In some versions of macOS, we might not even get any of the
 * 802.11-plus-radio-header DLT_'s defined, even though some
 * of them are used by various Airport drivers in those versions.
 */
#ifndef DLT_PRISM_HEADER
#define DLT_PRISM_HEADER    119
#endif
#ifndef DLT_AIRONET_HEADER
#define DLT_AIRONET_HEADER    120
#endif
#ifndef DLT_IEEE802_11_RADIO
#define DLT_IEEE802_11_RADIO    127
#endif
#ifndef DLT_IEEE802_11_RADIO_AVS
#define DLT_IEEE802_11_RADIO_AVS 163
#endif

static int pcap_can_set_rfmon_bpf(pcap_t *p);
static int pcap_activate_bpf(pcap_t *p);
static int pcap_setfilter_bpf(pcap_t *p, struct bpf_program *fp);
static int pcap_setdirection_bpf(pcap_t *, pcap_direction_t);
static int pcap_set_datalink_bpf(pcap_t *p, int dlt);

/*
 * For zerocopy bpf, the setnonblock/getnonblock routines need to modify
 * pb->nonblock so we don't call select(2) if the pcap handle is in non-
 * blocking mode.
 */
static int
pcap_getnonblock_bpf(pcap_t *p)
{
#ifdef HAVE_ZEROCOPY_BPF
    struct pcap_bpf *pb = p->priv;

    if (pb->zerocopy)
        return (pb->nonblock);
#endif
    return (pcap_getnonblock_fd(p));
}

static int
pcap_setnonblock_bpf(pcap_t *p, int nonblock)
{
#ifdef HAVE_ZEROCOPY_BPF
    struct pcap_bpf *pb = p->priv;

    if (pb->zerocopy) {
        pb->nonblock = nonblock;
        return (0);
    }
#endif
    # 调用 pcap_setnonblock_fd 函数设置文件描述符为非阻塞模式，并返回结果
    return (pcap_setnonblock_fd(p, nonblock));
#ifdef HAVE_ZEROCOPY_BPF
/*
 * Zero-copy BPF buffer routines to check for and acknowledge BPF data in
 * shared memory buffers.
 *
 * pcap_next_zbuf_shm(): Check for a newly available shared memory buffer,
 * and set up p->buffer and cc to reflect one if available.  Notice that if
 * there was no prior buffer, we select zbuf1 as this will be the first
 * buffer filled for a fresh BPF session.
 */
static int
pcap_next_zbuf_shm(pcap_t *p, int *cc)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_bpf *pb = p->priv;
    // 定义指向共享内存缓冲区头部的指针
    struct bpf_zbuf_header *bzh;

    // 如果当前共享内存缓冲区是 zbuf2 或者为空
    if (pb->zbuffer == pb->zbuf2 || pb->zbuffer == NULL) {
        // 将 bzh 指向 zbuf1 的头部
        bzh = (struct bpf_zbuf_header *)pb->zbuf1;
        // 如果用户生成的序号不等于内核生成的序号
        if (bzh->bzh_user_gen !=
            atomic_load_acq_int(&bzh->bzh_kernel_gen)) {
            // 设置 pcap_bpf 结构体的 bzh 指针指向 bzh
            pb->bzh = bzh;
            // 设置共享内存缓冲区指针指向 zbuf1
            pb->zbuffer = (u_char *)pb->zbuf1;
            // 设置 p->buffer 指向共享内存缓冲区中的数据
            p->buffer = pb->zbuffer + sizeof(*bzh);
            // 设置 cc 为内核数据长度
            *cc = bzh->bzh_kernel_len;
            return (1);
        }
    } else if (pb->zbuffer == pb->zbuf1) {
        // 将 bzh 指向 zbuf2 的头部
        bzh = (struct bpf_zbuf_header *)pb->zbuf2;
        // 如果用户生成的序号不等于内核生成的序号
        if (bzh->bzh_user_gen !=
            atomic_load_acq_int(&bzh->bzh_kernel_gen)) {
            // 设置 pcap_bpf 结构体的 bzh 指针指向 bzh
            pb->bzh = bzh;
            // 设置共享内存缓冲区指针指向 zbuf2
            pb->zbuffer = (u_char *)pb->zbuf2;
            // 设置 p->buffer 指向共享内存缓冲区中的数据
            p->buffer = pb->zbuffer + sizeof(*bzh);
            // 设置 cc 为内核数据长度
            *cc = bzh->bzh_kernel_len;
            return (1);
        }
    }
    // 设置 cc 为 0
    *cc = 0;
    return (0);
}

/*
 * pcap_next_zbuf() -- Similar to pcap_next_zbuf_shm(), except wait using
 * select() for data or a timeout, and possibly force rotation of the buffer
 * in the event we time out or are in immediate mode.  Invoke the shared
 * memory check before doing system calls in order to avoid doing avoidable
 * work.
 */
static int
pcap_next_zbuf(pcap_t *p, int *cc)
{
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_bpf *pb = p->priv;
    // 定义 bpf_zbuf 结构体
    struct bpf_zbuf bz;
    // 定义时间结构体
    struct timeval tv;
    // 定义当前时间结构体
    struct timespec cur;
    // 定义文件描述符集合
    fd_set r_set;
    // 定义变量
    int data, r;
    // 定义变量
    int expire, tmout;

#define TSTOMILLI(ts) (((ts)->tv_sec * 1000) + ((ts)->tv_nsec / 1000000))
    /*
     * 通过检查下一个共享内存缓冲区的数据来查看是否有等待的数据。
     */
    data = pcap_next_zbuf_shm(p, cc);
    if (data)
        return (data);
    /*
     * 如果先前的睡眠由于信号传递而被中断，确保相应地调整超时时间。
     * 这需要我们分析超时应该何时到期，并从当前时间中减去它。
     * 如果在这个操作之后，我们的超时小于或等于零，则处理它就像一个常规超时一样。
     */
    tmout = p->opt.timeout;
    if (tmout)
        (void) clock_gettime(CLOCK_MONOTONIC, &cur);
    if (pb->interrupted && p->opt.timeout) {
        expire = TSTOMILLI(&pb->firstsel) + p->opt.timeout;
        tmout = expire - TSTOMILLI(&cur);
#undef TSTOMILLI
        // 如果超时时间小于等于0，重置中断标志，读取共享内存中的数据包
        if (tmout <= 0) {
            pb->interrupted = 0;
            data = pcap_next_zbuf_shm(p, cc);
            if (data)
                return (data);
            // 如果共享内存中没有数据包，尝试旋转缓冲区
            if (ioctl(p->fd, BIOCROTZBUF, &bz) < 0) {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    PCAP_ERRBUF_SIZE, errno, "BIOCROTZBUF");
                return (PCAP_ERROR);
            }
            return (pcap_next_zbuf_shm(p, cc));
        }
    }
    /*
     * 缓冲区中没有数据，因此必须使用select()等待数据或下一个超时。请注意，只有在句柄处于阻塞模式时才调用select。
     */
    if (!pb->nonblock) {
        // 清空并设置文件描述符集合
        FD_ZERO(&r_set);
        FD_SET(p->fd, &r_set);
        // 如果超时时间不为0，设置超时时间
        if (tmout != 0) {
            tv.tv_sec = tmout / 1000;
            tv.tv_usec = (tmout * 1000) % 1000000;
        }
        // 调用select()等待数据或超时
        r = select(p->fd + 1, &r_set, NULL, NULL,
            p->opt.timeout != 0 ? &tv : NULL);
        // 如果select()被中断，且超时时间不为0，则设置中断标志并返回0
        if (r < 0 && errno == EINTR) {
            if (!pb->interrupted && p->opt.timeout) {
                pb->interrupted = 1;
                pb->firstsel = cur;
            }
            return (0);
        } else if (r < 0) {
            // 如果select()出错，返回错误信息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "select");
            return (PCAP_ERROR);
        }
    }
    pb->interrupted = 0;
    /*
     * 再次检查数据，现在可能存在数据，因为我们已经被唤醒，作为数据或超时的结果。首先尝试“有数据”的情况，因为它不需要系统调用。
     */
    data = pcap_next_zbuf_shm(p, cc);
    if (data)
        return (data);
    /*
     * 尝试强制旋转缓冲区以清除超时或立即数据。
     */
    if (ioctl(p->fd, BIOCROTZBUF, &bz) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCROTZBUF");
        return (PCAP_ERROR);
    }
    return (pcap_next_zbuf_shm(p, cc));
}
/*
 * Notify kernel that we are done with the buffer.  We don't reset zbuffer so
 * that we know which buffer to use next time around.
 */
static int
pcap_ack_zbuf(pcap_t *p)
{
    // 获取 pcap_t 结构体中的私有数据结构 pcap_bpf
    struct pcap_bpf *pb = p->priv;
    // 将用户生成的序号存储到内核生成的序号中
    atomic_store_rel_int(&pb->bzh->bzh_user_gen,
        pb->bzh->bzh_kernel_gen);
    // 将私有数据结构中的 bzh 和 buffer 置空
    pb->bzh = NULL;
    p->buffer = NULL;
    return (0);
}
#endif /* HAVE_ZEROCOPY_BPF */

// 创建一个 pcap_t 结构体，用于表示一个网络接口
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    pcap_t *p;

    // 调用 PCAP_CREATE_COMMON 宏创建一个 pcap_t 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_bpf);
    if (p == NULL)
        return (NULL);

    // 设置 pcap_t 结构体的激活操作为 pcap_activate_bpf
    p->activate_op = pcap_activate_bpf;
    // 设置 pcap_t 结构体的 rfmon 操作为 pcap_can_set_rfmon_bpf
    p->can_set_rfmon_op = pcap_can_set_rfmon_bpf;
#ifdef BIOCSTSTAMP
    /*
     * We claim that we support microsecond and nanosecond time
     * stamps.
     */
    // 分配内存用于存储时间戳精度列表
    p->tstamp_precision_list = malloc(2 * sizeof(u_int));
    if (p->tstamp_precision_list == NULL) {
        // 如果分配内存失败，则设置错误消息并释放内存
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno,
            "malloc");
        free(p);
        return (NULL);
    }
    // 设置时间戳精度列表的值
    p->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
    p->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
    p->tstamp_precision_count = 2;
#endif /* BIOCSTSTAMP */
    return (p);
}

/*
 * On success, returns a file descriptor for a BPF device.
 * On failure, returns a PCAP_ERROR_ value, and sets p->errbuf.
 */
static int
bpf_open(char *errbuf)
{
    int fd = -1;
    static const char cloning_device[] = "/dev/bpf";
    u_int n = 0;
    char device[sizeof "/dev/bpf0000000000"];
    static int no_cloning_bpf = 0;

#ifdef _AIX
    /*
     * Load the bpf driver, if it isn't already loaded,
     * and create the BPF device entries, if they don't
     * already exist.
     */
    if (bpf_load(errbuf) == PCAP_ERROR)
        return (PCAP_ERROR);
#endif

    /*
     * First, unless we've already tried opening /dev/bpf and
     * gotten ENOENT, try opening /dev/bpf.
     * If it fails with ENOENT, remember that, so we don't try
     * again, and try /dev/bpfN.
     */
    // 如果不禁用克隆设备，并且尝试以读写方式打开克隆设备失败，并且错误码不是 EACCES 和 ENOENT，
    // 或者以只读方式打开克隆设备失败，则执行以下代码块
    if (!no_cloning_bpf &&
        (fd = open(cloning_device, O_RDWR)) == -1 &&
        ((errno != EACCES && errno != ENOENT) ||
         (fd = open(cloning_device, O_RDONLY)) == -1)) {
        // 如果错误码不是 ENOENT
        if (errno != ENOENT) {
            // 如果错误码是 EACCES
            if (errno == EACCES) {
                // 设置 fd 为 PCAP_ERROR_PERM_DENIED
                fd = PCAP_ERROR_PERM_DENIED;
                // 格式化错误消息，提示可能需要 root 权限
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "Attempt to open %s failed - root privileges may be required",
                    cloning_device);
            } else {
                // 设置 fd 为 PCAP_ERROR
                fd = PCAP_ERROR;
                // 格式化错误消息，提示无法打开设备
                pcap_fmt_errmsg_for_errno(errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "(cannot open device) %s", cloning_device);
            }
            // 返回错误码
            return (fd);
        }
        // 设置 no_cloning_bpf 为 1
        no_cloning_bpf = 1;
    }

    // 如果禁用了克隆设备
    if (no_cloning_bpf) {
        /*
         * We don't have /dev/bpf.
         * Go through all the /dev/bpfN minors and find one
         * that isn't in use.
         */
        // 循环查找未被使用的 /dev/bpfN 设备
        do {
            (void)snprintf(device, sizeof(device), "/dev/bpf%u", n++);
            /*
             * Initially try a read/write open (to allow the inject
             * method to work).  If that fails due to permission
             * issues, fall back to read-only.  This allows a
             * non-root user to be granted specific access to pcap
             * capabilities via file permissions.
             *
             * XXX - we should have an API that has a flag that
             * controls whether to open read-only or read-write,
             * so that denial of permission to send (or inability
             * to send, if sending packets isn't supported on
             * the device in question) can be indicated at open
             * time.
             */
            // 尝试以读写方式打开设备，如果由于权限问题失败，则尝试以只读方式打开
            fd = open(device, O_RDWR);
            if (fd == -1 && errno == EACCES)
                fd = open(device, O_RDONLY);
        } while (fd < 0 && errno == EBUSY);
    }

    /*
     * XXX better message for all minors used
     */
    # 如果文件描述符小于0
    if (fd < 0) {
        # 根据错误码进行不同的处理
        switch (errno) {

        # 如果文件不存在
        case ENOENT:
            # 将错误码设置为PCAP_ERROR
            fd = PCAP_ERROR;
            # 如果n为1
            if (n == 1) {
                # 设置错误信息为"(there are no BPF devices)"
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "(there are no BPF devices)");
            } else {
                # 设置错误信息为"(all BPF devices are busy)"
                snprintf(errbuf, PCAP_ERRBUF_SIZE,
                    "(all BPF devices are busy)");
            }
            break;

        # 如果没有权限
        case EACCES:
            # 将错误码设置为PCAP_ERROR_PERM_DENIED
            fd = PCAP_ERROR_PERM_DENIED;
            # 设置错误信息为"Attempt to open %s failed - root privileges may be required"
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open %s failed - root privileges may be required",
                device);
            break;

        # 其他问题
        default:
            # 将错误码设置为PCAP_ERROR
            fd = PCAP_ERROR;
            # 格式化错误信息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "(cannot open BPF device) %s", device);
            break;
        }
    }

    # 返回文件描述符
    return (fd);
/*
 * 将网络适配器绑定到 BPF 设备，给定 BPF 设备的描述符和网络适配器的名称。
 *
 * 如果可用，使用 BIOCSETLIF（即“在 Solaris 上”），因为它支持更长的设备名称。
 *
 * 如果名称超过了适配器名称的长度，返回 PCAP_ERROR_NO_SUCH_DEVICE，表示不存在这样的设备。
 *
 * 如果尝试成功，则返回 BPF_BIND_SUCCEEDED。
 *
 * 如果尝试失败：
 *
 *    如果以 ENXIO 失败，则返回 PCAP_ERROR_NO_SUCH_DEVICE，因为设备不存在；
 *
 *    如果以 ENETDOWN 失败，则返回 PCAP_ERROR_IFACE_NOT_UP，因为接口存在但未启动，并且操作系统不允许绑定未启动的接口；
 *
 *    如果以 ENOBUFS 失败，则返回 BPF_BIND_BUFFER_TOO_BIG，并填写错误消息，因为请求的缓冲区太大；
 *
 *    否则，返回 PCAP_ERROR，并填写错误消息。
 */
#define BPF_BIND_SUCCEEDED    0
#define BPF_BIND_BUFFER_TOO_BIG    1

static int
bpf_bind(int fd, const char *name, char *errbuf)
{
    int status;
#ifdef LIFNAMSIZ
    struct lifreq ifr;

    if (strlen(name) >= sizeof(ifr.lifr_name)) {
        /* 名称太长，因此不可能存在这样的设备。 */
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }
    (void)pcap_strlcpy(ifr.lifr_name, name, sizeof(ifr.lifr_name));
    status = ioctl(fd, BIOCSETLIF, (caddr_t)&ifr);
#else
    struct ifreq ifr;

    if (strlen(name) >= sizeof(ifr.ifr_name)) {
        /* 名称太长，因此不可能存在这样的设备。 */
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }
    (void)pcap_strlcpy(ifr.ifr_name, name, sizeof(ifr.ifr_name));
    status = ioctl(fd, BIOCSETIF, (caddr_t)&ifr);
#endif
    # 如果状态小于 0，则进入 switch 分支
    if (status < 0) {
        # 根据错误码进行不同的处理
        switch (errno) {

        case ENXIO:
            # 如果错误码为 ENXIO，则表示没有这样的设备
            # 清空错误消息，返回没有这样的设备的错误码
            errbuf[0] = '\0';
            return (PCAP_ERROR_NO_SUCH_DEVICE);

        case ENETDOWN:
            # 如果错误码为 ENETDOWN，则表示网络连接已断开
            # 返回接口未启动的错误码
            return (PCAP_ERROR_IFACE_NOT_UP);

        case ENOBUFS:
            # 如果错误码为 ENOBUFS，则表示缓冲区大小过大
            # 返回缓冲区过大的错误码，并添加错误消息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "The requested buffer size for %s is too large",
                name);
            return (BPF_BIND_BUFFER_TOO_BIG);

        default:
            # 其他情况下，返回绑定接口到 BPF 设备失败的错误码，并添加错误消息
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "Binding interface %s to BPF device failed",
                name);
            return (PCAP_ERROR);
        }
    }
    # 返回绑定成功的状态
    return (BPF_BIND_SUCCEEDED);
}
/*
 * 打开并绑定到设备；如果我们实际上不打算使用设备，而只是测试它是否可以打开，或者打开它以获取有关它的信息，则使用此函数。
 *
 * 在失败时返回错误代码（始终为负数），在成功时返回绑定到的 BPF 设备的文件描述符（始终为非负数）。
 */
static int
bpf_open_and_bind(const char *name, char *errbuf)
{
    int fd;
    int status;

    /*
     * 首先，打开一个 BPF 设备。
     */
    fd = bpf_open(errbuf);
    if (fd < 0)
        return (fd);    /* fd 是适当的错误代码 */

    /*
     * 现在绑定到设备。
     */
    status = bpf_bind(fd, name, errbuf);
    if (status != BPF_BIND_SUCCEEDED) {
        close(fd);
        if (status == BPF_BIND_BUFFER_TOO_BIG) {
            /*
             * 我们没有指定缓冲区大小，所以
             * 这个 *真的* 不应该因为
             * 没有缓冲区空间而失败。失败。
             */
            return (PCAP_ERROR);
        }
        return (status);
    }

    /*
     * 成功。
     */
    return (fd);
}

#ifdef __APPLE__
static int
device_exists(int fd, const char *name, char *errbuf)
{
    int status;
    struct ifreq ifr;

    if (strlen(name) >= sizeof(ifr.ifr_name)) {
        /* 名称太长，因此不可能存在。 */
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }
    (void)pcap_strlcpy(ifr.ifr_name, name, sizeof(ifr.ifr_name));
    status = ioctl(fd, SIOCGIFFLAGS, (caddr_t)&ifr);
    # 如果状态小于 0
    if (status < 0) {
        # 如果错误码是 ENXIO 或者 EINVAL
        if (errno == ENXIO || errno == EINVAL) {
            '''
             * macOS 和 *BSD 返回这两个错误之一
             * 如果设备不存在，则不填写错误，因为这是一个"预期"的情况。
             '''
            return (PCAP_ERROR_NO_SUCH_DEVICE);
        }

        '''
         * 其他错误 - 为其提供消息，因为这是"意外"的情况。
         '''
        # 为错误码提供消息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno,
            "Can't get interface flags on %s", name);
        return (PCAP_ERROR);
    }

    '''
     * 设备存在。
     '''
    # 返回 0
    return (0);
    }
#endif

#ifdef BIOCGDLTLIST
// 定义一个函数，获取指定文件描述符的数据链路类型列表
static int
get_dlt_list(int fd, int v, struct bpf_dltlist *bdlp, char *ebuf)
{
    // 使用零填充指定大小的内存块，初始化数据链路类型列表结构体
    memset(bdlp, 0, sizeof(*bdlp));
        # 如果调用 ioctl 函数成功
        if (ioctl(fd, BIOCGDLTLIST, (caddr_t)bdlp) == 0) {
            # 定义变量 i 和 is_ethernet
            u_int i;
            int is_ethernet;

            # 为 bdlp->bfl_list 分配内存空间
            bdlp->bfl_list = (u_int *) malloc(sizeof(u_int) * (bdlp->bfl_len + 1));
            # 如果分配内存失败
            if (bdlp->bfl_list == NULL) {
                # 格式化错误消息并返回错误码
                pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "malloc");
                return (PCAP_ERROR);
            }

            # 再次调用 ioctl 函数
            if (ioctl(fd, BIOCGDLTLIST, (caddr_t)bdlp) < 0) {
                # 格式化错误消息并释放内存空间，然后返回错误码
                pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno, "BIOCGDLTLIST");
                free(bdlp->bfl_list);
                return (PCAP_ERROR);
            }

            # 对于真正的以太网设备，将 DLT_DOCSIS 添加到列表中
            # 以便应用程序可以选择它，以捕获由 Cisco Cable Modem Termination System 放在以太网上的 DOCSIS 流量
            # 在 Solaris 上，以太网设备还提供 DLT_IPNET，因此如果 DLT_IPNET 被定义，我们不将其视为设备不是以太网
            if (v == DLT_EN10MB) {
                is_ethernet = 1;
                for (i = 0; i < bdlp->bfl_len; i++) {
                    if (bdlp->bfl_list[i] != DLT_EN10MB

注意：由于代码片段过长，无法一次性解释完毕，建议分段进行解释。
#ifdef DLT_IPNET
                    && bdlp->bfl_list[i] != DLT_IPNET
#endif
                    ) {
                    // 如果定义了 DLT_IPNET 并且当前数据链路类型不是 DLT_IPNET
                    is_ethernet = 0;
                    // 跳出循环
                    break;
                }
            }
            if (is_ethernet) {
                /*
                 * 在列表末尾预留一个额外的插槽。
                 */
                bdlp->bfl_list[bdlp->bfl_len] = DLT_DOCSIS;
                bdlp->bfl_len++;
            }
        }
    } else {
        /*
         * EINVAL 只是意味着“我们不支持该设备上的此 ioctl”；不要将其视为错误。
         */
        if (errno != EINVAL) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "BIOCGDLTLIST");
            // 返回错误码
            return (PCAP_ERROR);
        }
    }
    // 返回成功
    return (0);
}
#endif

#if defined(__APPLE__)
static int
pcap_can_set_rfmon_bpf(pcap_t *p)
{
    struct utsname osinfo;
    int fd;
#ifdef BIOCGDLTLIST
    struct bpf_dltlist bdl;
    int err;
#endif

    /*
     * 在 Mac OS X/OS X/macOS 上监视模式的乐趣。
     *
     * 在 10.4 之前，根本不支持。
     *
     * 在 10.4 中，如果适配器 enN 支持监视模式，则有一个对应的 wltN 适配器；您打开它，而不是 enN，以获取监视模式。您会得到它提供的任何链路层标头。
     *
     * 在 10.5 和之后的版本中，如果适配器 enN 支持监视模式，则在其可选择的 DLT_ 值中，提供了让您获取 802.11 标头的值；选择其中一个值会将适配器置于监视模式（即，您只能在监视模式下获取 802.11 标头，而在监视模式下无法获取以太网标头）。
     */
    if (uname(&osinfo) == -1) {
        /*
         * 无法获取操作系统版本；只返回“否”。
         */
        return (0);
    }
    /*
     * 假设 osinfo.sysname 是 "Darwin"，因为 __APPLE__ 被定义了。我们只检查版本。
     */
    if (osinfo.release[0] < '8' && osinfo.release[1] == '.') {
        /*
         * 10.3 (Darwin 7.x) 或更早版本。
         * 不支持监视模式。
         */
        return (0);
    }
    if (osinfo.release[0] == '8' && osinfo.release[1] == '.') {
        char *wlt_name;
        int status;

        /*
         * 10.4 (Darwin 8.x)。将 s/en/wlt/，并检查设备是否存在。
         */
        if (strncmp(p->opt.device, "en", 2) != 0) {
            /*
             * 不是 enN 设备；不支持监视模式。
             */
            return (0);
        }
        fd = socket(AF_INET, SOCK_DGRAM, 0);
        if (fd == -1) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "socket");
            return (PCAP_ERROR);
        }
        if (pcap_asprintf(&wlt_name, "wlt%s", p->opt.device + 2) == -1) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            close(fd);
            return (PCAP_ERROR);
        }
        status = device_exists(fd, wlt_name, p->errbuf);
        free(wlt_name);
        close(fd);
        if (status != 0) {
            if (status == PCAP_ERROR_NO_SUCH_DEVICE)
                return (0);

            /*
             * 出错。
             */
            return (status);
        }
        return (1);
    }
#ifdef BIOCGDLTLIST
    /*
     * 如果支持 BIOCGDLTLIST，说明系统版本是 10.5 或更高；
     * 我们只需打开 enN 设备，并检查是否有任何 802.11 设备。
     *
     * 首先，打开一个 BPF 设备。
     */
    fd = bpf_open(p->errbuf);
    if (fd < 0)
        return (fd);    /* fd 是适当的错误代码 */

    /*
     * 现在绑定到设备。
     */
    err = bpf_bind(fd, p->opt.device, p->errbuf);
    if (err != BPF_BIND_SUCCEEDED) {
        close(fd);
        if (err == BPF_BIND_BUFFER_TOO_BIG) {
            /*
             * 我们没有指定缓冲区大小，所以这个失败是不应该发生的，
             * 因为没有缓冲区空间。失败。
             */
            return (PCAP_ERROR);
        }
        return (err);
    }

    /*
     * 我们知道默认的链路类型 -- 现在确定该接口支持的所有 DLTs。
     * 如果这个失败了并返回 EINVAL，这不是致命的；我们只是不能在后面使用这个功能。
     * (我们不关心 DLT_DOCSIS，所以我们将 DLT_NULL 作为该适配器的默认 DLT。)
     */
    if (get_dlt_list(fd, DLT_NULL, &bdl, p->errbuf) == PCAP_ERROR) {
        close(fd);
        return (PCAP_ERROR);
    }
    if (find_802_11(&bdl) != -1) {
        /*
         * 我们有一个 802.11 DLT，所以我们可以设置监控模式。
         */
        free(bdl.bfl_list);
        close(fd);
        return (1);
    }
    free(bdl.bfl_list);
    close(fd);
#endif /* BIOCGDLTLIST */
    return (0);
}
#elif defined(HAVE_BSD_IEEE80211)
static int
pcap_can_set_rfmon_bpf(pcap_t *p)
{
    int ret;

    ret = monitor_mode(p, 0);
    if (ret == PCAP_ERROR_RFMON_NOTSUP)
        return (0);    /* 不是错误，只是“不能执行” */
    if (ret == 0)
        return (1);    /* 成功 */
    return (ret);
}
#else
static int
pcap_can_set_rfmon_bpf(pcap_t *p _U_)
{
    return (0);
}
#endif

static int
pcap_stats_bpf(pcap_t *p, struct pcap_stat *ps)
{
    struct bpf_stat s;
    /*
     * "ps_recv" counts packets handed to the filter, not packets
     * that passed the filter.  This includes packets later dropped
     * because we ran out of buffer space.
     *
     * "ps_drop" counts packets dropped inside the BPF device
     * because we ran out of buffer space.  It doesn't count
     * packets dropped by the interface driver.  It counts
     * only packets that passed the filter.
     *
     * Both statistics include packets not yet read from the kernel
     * by libpcap, and thus not yet seen by the application.
     */
    // 如果获取统计信息失败，则设置错误消息并返回错误代码
    if (ioctl(p->fd, BIOCGSTATS, (caddr_t)&s) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCGSTATS");
        return (PCAP_ERROR);
    }

    // 设置接收的数据包数为获取的统计信息中的接收数据包数
    ps->ps_recv = s.bs_recv;
    // 设置丢弃的数据包数为获取的统计信息中的丢弃数据包数
    ps->ps_drop = s.bs_drop;
    // 设置接口丢弃的数据包数为0
    ps->ps_ifdrop = 0;
    // 返回成功代码
    return (0);
static int
pcap_read_bpf(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    // 获取 pcap_t 结构体中的 pcap_bpf 结构体
    struct pcap_bpf *pb = p->priv;
    int cc;
    int n = 0;
    register u_char *bp, *ep;
    u_char *datap;
#ifdef PCAP_FDDIPAD
    register u_int pad;
#endif
#ifdef HAVE_ZEROCOPY_BPF
    int i;
#endif

 again:
    /*
     * Has "pcap_breakloop()" been called?
     */
    // 检查是否调用了 pcap_breakloop() 函数
    if (p->break_loop) {
        /*
         * Yes - clear the flag that indicates that it
         * has, and return PCAP_ERROR_BREAK to indicate
         * that we were told to break out of the loop.
         */
        // 清除标志位，返回 PCAP_ERROR_BREAK 表示被告知跳出循环
        p->break_loop = 0;
        return (PCAP_ERROR_BREAK);
    }
    cc = p->cc;
    if (p->cc == 0) {
        /*
         * When reading without zero-copy from a file descriptor, we
         * use a single buffer and return a length of data in the
         * buffer.  With zero-copy, we update the p->buffer pointer
         * to point at whatever underlying buffer contains the next
         * data and update cc to reflect the data found in the
         * buffer.
         */
#ifdef HAVE_ZEROCOPY_BPF
        // 如果支持零拷贝
        if (pb->zerocopy) {
            // 如果 p->buffer 不为空，确认已经处理完数据
            if (p->buffer != NULL)
                pcap_ack_zbuf(p);
            // 从下层获取下一个数据包
            i = pcap_next_zbuf(p, &cc);
            // 如果没有数据包，重新尝试
            if (i == 0)
                goto again;
            // 如果出错，返回 PCAP_ERROR
            if (i < 0)
                return (PCAP_ERROR);
        } else
#endif
        {
            // 从文件描述符中读取数据到缓冲区
            cc = (int)read(p->fd, p->buffer, p->bufsize);
        }
        // 如果读取出错
        if (cc < 0) {
            /* Don't choke when we get ptraced */
            // 当被 ptraced 时不中断
            switch (errno) {

            case EINTR:
                goto again;
#ifdef _AIX
            case EFAULT:
                /*
                 * AIX特有的问题。
                 *
                 * 由于某种未知的原因，bpf内核扩展中的uiomove()操作
                 * 有时会返回EFAULT。我不知道为什么会这样，因为
                 * 内核调试器显示用户缓冲区是正确的。这个问题似乎
                 * 大部分情况下可以通过在首次使用前对缓冲区进行memset来缓解。
                 * 非常奇怪.... Shaun Clowes
                 *
                 * 无论如何，这意味着我们不应该将EFAULT视为致命错误；
                 * 因为我们没有一个API来返回“自上次看到数据包以来有一些数据包丢失”的指示，
                 * 我们只是忽略EFAULT并继续读取。
                 */
                goto again;
#endif

            case EWOULDBLOCK:
                return (0);

            case ENXIO:    /* FreeBSD, DragonFly BSD, and Darwin */
            case EIO:    /* OpenBSD */
                    /* NetBSD似乎在这种情况下不返回错误 */
                /*
                 * 我们捕获数据的设备消失了。
                 *
                 * XXX - 我们真的应该返回一个适当的错误，
                 * 但是pcap_dispatch()等并没有文档化的错误返回
                 * 除了PCAP_ERROR或PCAP_ERROR_BREAK。
                 */
                snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                    "The interface disappeared");
                return (PCAP_ERROR);
#if defined(sun) && !defined(BSD) && !defined(__svr4__) && !defined(__SVR4)
            /*
             * 由于 SunOS 存在一个 bug，当文件偏移量超过 2^31 字节时，内核文件偏移量会溢出，并且 read 函数会因为 EINVAL 而失败。
             * 通过 lseek() 函数将偏移量设置为 0 可以解决这个问题。
             */
            case EINVAL:
                if (lseek(p->fd, 0L, SEEK_CUR) +
                    p->bufsize < 0) {
                    (void)lseek(p->fd, 0L, SEEK_SET);
                    goto again;
                }
                /* 继续执行下面的代码 */
#endif
            }
            // 根据错误号生成错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "read");
            return (PCAP_ERROR);
        }
        // 将缓冲区的起始地址赋给 bp
        bp = (u_char *)p->buffer;
    } else
        // 如果缓冲区为空，则将当前位置赋给 bp
        bp = p->bp;

    /*
     * 遍历每个数据包。
     *
     * 这里假设单个数据包缓冲区中的数据包数量不会超过 INT_MAX，因此数据包计数不会溢出。
     */
#ifdef BIOCSTSTAMP
#define bhp ((struct bpf_xhdr *)bp)
#else
#define bhp ((struct bpf_hdr *)bp)
#endif
    // 计算数据包结束位置
    ep = bp + cc;
#ifdef PCAP_FDDIPAD
    // 如果定义了 PCAP_FDDIPAD，则获取 FDDI 填充字节数
    pad = p->fddipad;
#endif
    while (bp < ep) {
        register u_int caplen, hdrlen;

        /*
         * 检查是否调用了 "pcap_breakloop()" 函数
         * 如果是，立即返回 - 如果我们还没有读取任何数据包，清除标志并返回 PCAP_ERROR_BREAK
         * 表示我们被告知要跳出循环，否则保持标志设置，这样下一次调用将在没有读取任何数据包的情况下跳出循环，并返回到目前为止已处理的数据包数量。
         */
        if (p->break_loop) {
            p->bp = bp;
            p->cc = (int)(ep - bp);
            /*
             * ep 是根据 read() 的返回值设置的，
             * 但是从 BPF 设备读取的 read() 不一定返回 BPF_WORDALIGN() 的倍数。
             * 但是，每当我们增加 bp 时，我们都会将增量值向上舍入 BPF_WORDALIGN() 的值，因此在处理缓冲区中的最后一个数据包后，我们可能会将 bp 增加到 ep 之后。
             *
             * 我们将 ep < bp 视为这种情况的指示，并将 p->cc 设置为 0。
             */
            if (p->cc < 0)
                p->cc = 0;
            if (n == 0) {
                p->break_loop = 0;
                return (PCAP_ERROR_BREAK);
            } else
                return (n);
        }

        caplen = bhp->bh_caplen;
        hdrlen = bhp->bh_hdrlen;
        datap = bp + hdrlen;
        /*
         * 短路求值：如果在内核中使用 BPF 过滤器，则现在不需要执行过滤器 - 我们已经知道数据包通过了过滤器。
         */
#ifdef PCAP_FDDIPAD
         * 如果定义了 PCAP_FDDIPAD，则说明需要考虑数据包的填充情况
         * 注意：过滤代码是根据假设 p->fddipad 是头部之前的填充量生成的
         * 因为这是内核所需的，所以我们在跳过填充之前运行过滤器
#endif
         */
        // 如果在内核中进行过滤，或者过滤器通过
        if (pb->filtering_in_kernel ||
            pcap_filter(p->fcode.bf_insns, datap, bhp->bh_datalen, caplen)) {
            // 创建一个 pcap 数据包头
            struct pcap_pkthdr pkthdr;
#ifdef BIOCSTSTAMP
            // 如果定义了 BIOCSTSTAMP，则需要处理时间戳
            struct bintime bt;

            // 将时间戳转换为 bintime 结构
            bt.sec = bhp->bh_tstamp.bt_sec;
            bt.frac = bhp->bh_tstamp.bt_frac;
            // 根据时间戳精度设置时间戳
            if (p->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO) {
                struct timespec ts;

                // 将 bintime 转换为 timespec
                bintime2timespec(&bt, &ts);
                pkthdr.ts.tv_sec = ts.tv_sec;
                pkthdr.ts.tv_usec = ts.tv_nsec;
            } else {
                struct timeval tv;

                // 将 bintime 转换为 timeval
                bintime2timeval(&bt, &tv);
                pkthdr.ts.tv_sec = tv.tv_sec;
                pkthdr.ts.tv_usec = tv.tv_usec;
            }
#else
            // 如果没有定义 BIOCSTSTAMP，则直接使用时间戳
            pkthdr.ts.tv_sec = bhp->bh_tstamp.tv_sec;
#ifdef _AIX
            /*
             * AIX 的 BPF 返回秒/纳秒时间戳，而不是秒/微秒时间戳。
             */
            pkthdr.ts.tv_usec = bhp->bh_tstamp.tv_usec/1000;
#else
            pkthdr.ts.tv_usec = bhp->bh_tstamp.tv_usec;
#endif
#endif /* BIOCSTSTAMP */
#ifdef PCAP_FDDIPAD
            // 如果定义了 PCAP_FDDIPAD，则需要考虑填充
            if (caplen > pad)
                pkthdr.caplen = caplen - pad;
            else
                pkthdr.caplen = 0;
            if (bhp->bh_datalen > pad)
                pkthdr.len = bhp->bh_datalen - pad;
            else
                pkthdr.len = 0;
            datap += pad;
#else
            // 如果没有定义 PCAP_FDDIPAD，则直接使用 caplen 和数据包长度
            pkthdr.caplen = caplen;
            pkthdr.len = bhp->bh_datalen;
#endif
            // 调用回调函数处理数据包
            (*callback)(user, &pkthdr, datap);
            // 移动指针到下一个数据包的位置
            bp += BPF_WORDALIGN(caplen + hdrlen);
            // 如果达到指定的数据包数量，返回
            if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
                // 更新数据包指针和长度
                p->bp = bp;
                p->cc = (int)(ep - bp);
                /*
                 * 如果数据包长度小于0，设置为0
                 */
                if (p->cc < 0)
                    p->cc = 0;
                return (n);
            }
        } else {
            /*
             * 跳过这个数据包
             */
            bp += BPF_WORDALIGN(caplen + hdrlen);
        }
    }
#undef bhp
    // 重置数据包长度
    p->cc = 0;
    return (n);
}

static int
pcap_inject_bpf(pcap_t *p, const void *buf, int size)
{
    int ret;

    // 使用write函数将数据包写入文件描述符
    ret = (int)write(p->fd, buf, size);
#ifdef __APPLE__
    # 如果返回值为-1并且错误码为EAFNOSUPPORT
    if (ret == -1 && errno == EAFNOSUPPORT) {
        '''
        在某些 macOS 版本中存在一个 bug，设置 BIOCSHDRCMPLT 标志会导致写操作失败；参见：
        
        http://cerberus.sourcefire.com/~jeff/archives/patches/macosx/BIOCSHDRCMPLT-10.3.3.patch
        
        因此，如果在 macOS 上写操作返回 EAFNOSUPPORT，我们假设是由于该 bug 导致，然后关闭该标志并重试。
        如果重试成功，要么意味着有人应用了来自该 URL 的修复程序，或者应用了其他来自
        
        http://cerberus.sourcefire.com/~jeff/archives/patches/macosx/
        
        的修补程序，并且正在运行具有这些修复程序的 Darwin 内核，或者苹果在某个 macOS 版本中修复了该问题。
        '''
        # 设置 spoof_eth_src 变量为 0
        u_int spoof_eth_src = 0;

        # 如果关闭 BIOCSHDRCMPLT 标志失败
        if (ioctl(p->fd, BIOCSHDRCMPLT, &spoof_eth_src) == -1) {
            # 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "send: can't turn off BIOCSHDRCMPLT");
            # 返回错误
            return (PCAP_ERROR);
        }

        '''
        现在再次尝试写操作。
        '''
        # 重新尝试写操作
        ret = (int)write(p->fd, buf, size);
    }
#endif /* __APPLE__ */
// 如果不是苹果系统，则执行以下代码

    if (ret == -1) {
        // 如果返回值为-1，表示发送失败
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        // 格式化错误消息并返回错误码
        return (PCAP_ERROR);
    }
    // 返回发送结果
    return (ret);
}

#ifdef _AIX
// 如果是AIX系统，则执行以下代码
static int
bpf_odminit(char *errbuf)
{
    char *errstr;

    if (odm_initialize() == -1) {
        // 如果初始化失败，则返回错误信息
        if (odm_err_msg(odmerrno, &errstr) == -1)
            errstr = "Unknown error";
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "bpf_load: odm_initialize failed: %s",
            errstr);
        return (PCAP_ERROR);
    }

    if ((odmlockid = odm_lock("/etc/objrepos/config_lock", ODM_WAIT)) == -1) {
        // 如果锁定失败，则返回错误信息
        if (odm_err_msg(odmerrno, &errstr) == -1)
            errstr = "Unknown error";
        snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "bpf_load: odm_lock of /etc/objrepos/config_lock failed: %s",
            errstr);
        (void)odm_terminate();
        return (PCAP_ERROR);
    }

    return (0);
}

static int
bpf_odmcleanup(char *errbuf)
{
    char *errstr;

    if (odm_unlock(odmlockid) == -1) {
        // 如果解锁失败，则返回错误信息
        if (errbuf != NULL) {
            if (odm_err_msg(odmerrno, &errstr) == -1)
                errstr = "Unknown error";
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "bpf_load: odm_unlock failed: %s",
                errstr);
        }
        return (PCAP_ERROR);
    }

    if (odm_terminate() == -1) {
        // 如果终止失败，则返回错误信息
        if (errbuf != NULL) {
            if (odm_err_msg(odmerrno, &errstr) == -1)
                errstr = "Unknown error";
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "bpf_load: odm_terminate failed: %s",
                errstr);
        }
        return (PCAP_ERROR);
    }

    return (0);
}

static int
bpf_load(char *errbuf)
{
    long major;
    int *minors;
    int numminors, i, rc;
    char buf[1024];
    struct stat sbuf;
    struct bpf_config cfg_bpf;
    struct cfg_load cfg_ld;
    struct cfg_kmod cfg_km;
    // 其他代码，未提供足够信息，无法添加注释
}
    /*
     * 这非常接近实际实现的情况，但我修复了一些（不太可能发生的）错误情况。
     */
    // 如果 bpfloadedflag 为真，则返回 0
    if (bpfloadedflag)
        return (0);

    // 初始化 BPF 设备，如果失败则返回错误
    if (bpf_odminit(errbuf) == PCAP_ERROR)
        return (PCAP_ERROR);

    // 生成 BPF 设备的主设备号，如果失败则返回错误
    major = genmajor(BPF_NAME);
    if (major == -1) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "bpf_load: genmajor failed");
        (void)bpf_odmcleanup(NULL);
        return (PCAP_ERROR);
    }

    // 获取 BPF 设备的次设备号，如果失败则生成新的次设备号
    minors = getminor(major, &numminors, BPF_NAME);
    if (!minors) {
        minors = genminor("bpf", major, 0, BPF_MINORS, 1, 1);
        if (!minors) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "bpf_load: genminor failed");
            (void)bpf_odmcleanup(NULL);
            return (PCAP_ERROR);
        }
    }

    // 清理 BPF 设备，如果失败则返回错误
    if (bpf_odmcleanup(errbuf) == PCAP_ERROR)
        return (PCAP_ERROR);

    // 检查 BPF 设备节点是否存在，如果不存在则创建
    rc = stat(BPF_NODE "0", &sbuf);
    if (rc == -1 && errno != ENOENT) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "bpf_load: can't stat %s", BPF_NODE "0");
        return (PCAP_ERROR);
    }

    // 如果节点不存在或者主设备号不匹配，则重新创建节点
    if (rc == -1 || getmajor(sbuf.st_rdev) != major) {
        for (i = 0; i < BPF_MINORS; i++) {
            snprintf(buf, sizeof(buf), "%s%d", BPF_NODE, i);
            unlink(buf);
            if (mknod(buf, S_IRUSR | S_IFCHR, domakedev(major, i)) == -1) {
                pcap_fmt_errmsg_for_errno(errbuf,
                    PCAP_ERRBUF_SIZE, errno,
                    "bpf_load: can't mknod %s", buf);
                return (PCAP_ERROR);
            }
        }
    }

    // 检查驱动程序是否加载
    memset(&cfg_ld, 0x0, sizeof(cfg_ld));
    snprintf(buf, sizeof(buf), "%s/%s", DRIVER_PATH, BPF_NAME);
    cfg_ld.path = buf;
    # 检查驱动程序是否已加载，如果未加载则加载
    if ((sysconfig(SYS_QUERYLOAD, (void *)&cfg_ld, sizeof(cfg_ld)) == -1) ||
        (cfg_ld.kmid == 0)) {
        # 驱动程序未加载，现在加载它
        if (sysconfig(SYS_SINGLELOAD, (void *)&cfg_ld, sizeof(cfg_ld)) == -1) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "bpf_load: could not load driver");
            return (PCAP_ERROR);
        }
    }

    # 配置驱动程序
    cfg_km.cmd = CFG_INIT;
    cfg_km.kmid = cfg_ld.kmid;
    cfg_km.mdilen = sizeof(cfg_bpf);
    cfg_km.mdiptr = (void *)&cfg_bpf;
    for (i = 0; i < BPF_MINORS; i++) {
        cfg_bpf.devno = domakedev(major, i);
        if (sysconfig(SYS_CFGKMOD, (void *)&cfg_km, sizeof(cfg_km)) == -1) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "bpf_load: could not configure driver");
            return (PCAP_ERROR);
        }
    }

    # 设置驱动程序加载标志为已加载
    bpfloadedflag = 1;

    # 返回成功加载的标志
    return (0);
}
#endif

/*
 * 当需要时，撤消打开设备时执行的任何操作。
 */
static void
pcap_cleanup_bpf(pcap_t *p)
{
    struct pcap_bpf *pb = p->priv;
#ifdef HAVE_BSD_IEEE80211
    int sock;
    struct ifmediareq req;
    struct ifreq ifr;
#endif

    if (pb->must_do_on_close != 0) {
        /*
         * 当关闭此 pcap_t 时，有一些操作我们必须执行。
         */
#endif /* HAVE_BSD_IEEE80211 */

#if defined(__FreeBSD__) && defined(SIOCIFCREATE2)
        /*
         * 尝试销毁我们创建的 usbusN 接口。
         */
        if (pb->must_do_on_close & MUST_DESTROY_USBUS) {
            if (if_nametoindex(pb->device) > 0) {
                int s;

                s = socket(AF_LOCAL, SOCK_DGRAM, 0);
                if (s >= 0) {
                    pcap_strlcpy(ifr.ifr_name, pb->device,
                        sizeof(ifr.ifr_name));
                    ioctl(s, SIOCIFDESTROY, &ifr);
                    close(s);
                }
            }
        }
#endif /* defined(__FreeBSD__) && defined(SIOCIFCREATE2) */
        /*
         * 从需要将接口从某种模式中移出的 pcap 列表中移除此 pcap。
         */
        pcap_remove_from_pcaps_to_close(p);
        pb->must_do_on_close = 0;
    }

#ifdef HAVE_ZEROCOPY_BPF
    if (pb->zerocopy) {
        /*
         * 删除映射。请注意，在这种情况下，p->buffer 被初始化为这些 mmapped 区域中的一个，因此不要尝试直接释放它；将其置空，以便 pcap_cleanup_live_common() 不会尝试释放它。
         */
        if (pb->zbuf1 != MAP_FAILED && pb->zbuf1 != NULL)
            (void) munmap(pb->zbuf1, pb->zbufsize);
        if (pb->zbuf2 != MAP_FAILED && pb->zbuf2 != NULL)
            (void) munmap(pb->zbuf2, pb->zbufsize);
        p->buffer = NULL;
    }
#endif
    if (pb->device != NULL) {
        free(pb->device);
        pb->device = NULL;
    }
    # 调用pcap_cleanup_live_common函数，传入参数p
    pcap_cleanup_live_common(p);
#ifdef __APPLE__
static int
check_setif_failure(pcap_t *p, int error)
{
    int fd;
    int err;

    // 在苹果系统上检查设置接口失败的情况
    // 未实现具体的操作，直接返回错误状态
    return (error);
}
#else
static int
check_setif_failure(pcap_t *p _U_, int error)
{
    // 在非苹果系统上检查设置接口失败的情况
    // 未实现具体的操作，直接返回错误状态
    return (error);
}
#endif

/*
 * 默认捕获缓冲区大小。
 * 对于现代机器和快速网络来说，32K 太小了；我们选择 .5M，因为至少一些系统的 BPF 上限是这个值。
 *
 * 但是，在 AIX 3.5 上，更大的缓冲区大小在压力下导致不可恢复的读取失败，所以我们将其保留为 32K；这是 AIX 的 BPF 又一个有问题的地方。
 */
#ifdef _AIX
#define DEFAULT_BUFSIZE    32768
#else
#define DEFAULT_BUFSIZE    524288
#endif

static int
pcap_activate_bpf(pcap_t *p)
{
    struct pcap_bpf *pb = p->priv;
    int status = 0;
#ifdef HAVE_BSD_IEEE80211
    int retv;
#endif
    int fd;
#if defined(LIFNAMSIZ) && defined(ZONENAME_MAX) && defined(lifr_zoneid)
    struct lifreq ifr;
    char *zonesep;
#endif
    struct bpf_version bv;
#ifdef __APPLE__
    int sockfd;
    char *wltdev = NULL;
#endif
#ifdef BIOCGDLTLIST
    struct bpf_dltlist bdl;
#if defined(__APPLE__) || defined(HAVE_BSD_IEEE80211)
    int new_dlt;
#endif
#endif /* BIOCGDLTLIST */
#if defined(BIOCGHDRCMPLT) && defined(BIOCSHDRCMPLT)
    u_int spoof_eth_src = 1;
#endif
    u_int v;
    struct bpf_insn total_insn;
    struct bpf_program total_prog;
    struct utsname osinfo;
    int have_osinfo = 0;
#ifdef HAVE_ZEROCOPY_BPF
    struct bpf_zbuf bz;
    u_int bufmode, zbufmax;
#endif

    // 打开 BPF 设备
    fd = bpf_open(p->errbuf);
    if (fd < 0) {
        status = fd;
        // 如果打开失败，跳转到错误处理
        goto bad;
    }

    // 将打开的文件描述符赋值给 pcap_t 结构体
    p->fd = fd;
    # 如果调用 ioctl 函数失败
    if (ioctl(fd, BIOCVERSION, (caddr_t)&bv) < 0) {
        # 格式化错误消息，将错误信息存储到 p->errbuf 中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE, errno, "BIOCVERSION");
        # 设置状态为错误
        status = PCAP_ERROR;
        # 跳转到标签 bad 处
        goto bad;
    }
    # 如果内核 bpf 过滤器版本不兼容
    if (bv.bv_major != BPF_MAJOR_VERSION || bv.bv_minor < BPF_MINOR_VERSION) {
        # 将错误消息存储到 p->errbuf 中
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "kernel bpf filter out of date");
        # 设置状态为错误
        status = PCAP_ERROR;
        # 跳转到标签 bad 处
        goto bad;
    }

    '''
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为最大允许的值。
     *
     * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     '''
    # 如果快照长度小于等于 0 或大于最大快照长度
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        # 将快照长度设置为最大快照长度
        p->snapshot = MAXIMUM_SNAPLEN;
#if defined(LIFNAMSIZ) && defined(ZONENAME_MAX) && defined(lifr_zoneid)
    /*
     * 如果定义了LIFNAMSIZ、ZONENAME_MAX和lifr_zoneid，则执行以下代码块
     */
    /*
     * 获取当前执行的区域的zoneid。
     */
    if ((ifr.lifr_zoneid = getzoneid()) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "getzoneid()");
        status = PCAP_ERROR;
        goto bad;
    }
    /*
     * 检查给定的源数据链路名称是否有'/'分隔的zonename前缀字符串。
     * 在Solaris全局区域中，带有zonename前缀的源数据链路可以被pcap消费者使用，
     * 用于捕获非全局区域中数据链路的流量。
     * 非全局区域无法访问其它命名空间中的数据链路。
     */
    if ((zonesep = strchr(p->opt.device, '/')) != NULL) {
        char path_zname[ZONENAME_MAX];
        int  znamelen;
        char *lnamep;

        if (ifr.lifr_zoneid != GLOBAL_ZONEID) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "zonename/linkname only valid in global zone.");
            status = PCAP_ERROR;
            goto bad;
        }
        znamelen = zonesep - p->opt.device;
        (void) pcap_strlcpy(path_zname, p->opt.device, znamelen + 1);
        ifr.lifr_zoneid = getzoneidbyname(path_zname);
        if (ifr.lifr_zoneid == -1) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "getzoneidbyname(%s)", path_zname);
            status = PCAP_ERROR;
            goto bad;
        }
        lnamep = strdup(zonesep + 1);
        if (lnamep == NULL) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "strdup");
            status = PCAP_ERROR;
            goto bad;
        }
        free(p->opt.device);
        p->opt.device = lnamep;
    }
#endif

    pb->device = strdup(p->opt.device);
    // 复制p->opt.device的值给pb->device
    # 如果设备为空，则使用错误号生成错误消息，并设置状态为错误，然后跳转到标签bad
    if (pb->device == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "strdup");
        status = PCAP_ERROR;
        goto bad;
    }

    # 尝试获取当前操作系统的版本信息
    /*
     * Attempt to find out the version of the OS on which we're running.
     */
    if (uname(&osinfo) == 0)
        have_osinfo = 1;
#ifdef __APPLE__
    /*
     * 如果是苹果系统，查看 pcap_can_set_rfmon_bpf() 中的注释，解释为什么要检查版本号。
     */
    }
#endif /* __APPLE__ */

    /*
     * 如果这是 FreeBSD，并且设备名称以 "usbus" 开头，尝试创建接口（如果尚未可用）。
     */
#if defined(__FreeBSD__) && defined(SIOCIFCREATE2)
    }
#endif /* defined(__FreeBSD__) && defined(SIOCIFCREATE2) */

#ifdef HAVE_ZEROCOPY_BPF
    /*
     * 如果 BPF 扩展设置缓冲区模式存在，则尝试将模式设置为零拷贝。
     * 如果失败，则使用常规缓冲区。如果成功但其他设置失败，则向用户返回错误。
     */
    bufmode = BPF_BUFMODE_ZBUF;
    if (ioctl(fd, BIOCSETBUFMODE, (caddr_t)&bufmode) == 0) {
        /*
         * 我们有零拷贝 BPF；使用它。
         */
        pb->zerocopy = 1;

        /*
         * 如何选择缓冲区大小：首先，查询零拷贝支持的最大缓冲区大小。
         * 这也让我们快速确定内核是否通常支持零拷贝。
         * 然后，如果指定了缓冲区大小，则使用该大小，否则查询默认缓冲区大小，这反映了内核对所需默认值的策略。四舍五入到最近的页面大小。
         */
        if (ioctl(fd, BIOCGETZMAX, (caddr_t)&zbufmax) < 0) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "BIOCGETZMAX");
            status = PCAP_ERROR;
            goto bad;
        }

        if (p->opt.buffer_size != 0) {
            /*
             * 明确指定了缓冲区大小；使用它。
             */
            v = p->opt.buffer_size;
        } else {
            if ((ioctl(fd, BIOCGBLEN, (caddr_t)&v) < 0) ||
                v < DEFAULT_BUFSIZE)
                v = DEFAULT_BUFSIZE;
        }
#ifndef roundup
#define roundup(x, y)   ((((x)+((y)-1))/(y))*(y))  /* to any y */
#endif
        // 将缓冲区大小设置为页面大小的倍数
        pb->zbufsize = roundup(v, getpagesize());
        // 如果缓冲区大小超过最大限制，则将其设置为最大限制
        if (pb->zbufsize > zbufmax)
            pb->zbufsize = zbufmax;
        // 使用 mmap() 创建匿名映射，映射到 pb->zbuf1
        pb->zbuf1 = mmap(NULL, pb->zbufsize, PROT_READ | PROT_WRITE,
            MAP_ANON, -1, 0);
        // 使用 mmap() 创建匿名映射，映射到 pb->zbuf2
        pb->zbuf2 = mmap(NULL, pb->zbufsize, PROT_READ | PROT_WRITE,
            MAP_ANON, -1, 0);
        // 如果映射失败，则设置错误信息并返回错误状态
        if (pb->zbuf1 == MAP_FAILED || pb->zbuf2 == MAP_FAILED) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "mmap");
            status = PCAP_ERROR;
            goto bad;
        }
        // 使用 memset() 将 bz 清零
        memset(&bz, 0, sizeof(bz)); /* bzero() deprecated, replaced with memset() */
        // 设置 bz 的缓冲区指针和长度
        bz.bz_bufa = pb->zbuf1;
        bz.bz_bufb = pb->zbuf2;
        bz.bz_buflen = pb->zbufsize;
        // 使用 ioctl() 设置 BPF 缓冲区
        if (ioctl(fd, BIOCSETZBUF, (caddr_t)&bz) < 0) {
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "BIOCSETZBUF");
            status = PCAP_ERROR;
            goto bad;
        }
        // 绑定文件描述符到指定设备
        status = bpf_bind(fd, p->opt.device, ifnamsiz, p->errbuf);
        // 如果绑定失败，则根据失败原因进行处理
        if (status != BPF_BIND_SUCCEEDED) {
            if (status == BPF_BIND_BUFFER_TOO_BIG) {
                /*
                 * The requested buffer size
                 * is too big.  Fail.
                 *
                 * XXX - should we do the "keep cutting
                 * the buffer size in half" loop here if
                 * we're using the default buffer size?
                 */
                status = PCAP_ERROR;
            }
            goto bad;
        }
        // 计算剩余空间大小
        v = pb->zbufsize - sizeof(struct bpf_zbuf_header);
    } else
#endif
    }

    /* 获取数据链路层类型 */
    if (ioctl(fd, BIOCGDLT, (caddr_t)&v) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCGDLT");
        status = PCAP_ERROR;
        goto bad;
    }

#ifdef _AIX
    /*
     * AIX's BPF returns IFF_ types, not DLT_ types, in BIOCGDLT.
     */
    switch (v) {

    case IFT_ETHER:
    # 如果接口类型为ISO88023，则设置数据链路类型为EN10MB
    case IFT_ISO88023:
        v = DLT_EN10MB;
        break;

    # 如果接口类型为FDDI，则设置数据链路类型为FDDI
    case IFT_FDDI:
        v = DLT_FDDI;
        break;

    # 如果接口类型为ISO88025，则设置数据链路类型为IEEE802
    case IFT_ISO88025:
        v = DLT_IEEE802;
        break;

    # 如果接口类型为LOOP，则设置数据链路类型为NULL
    case IFT_LOOP:
        v = DLT_NULL;
        break;

    # 如果接口类型为其他未知类型，则设置错误信息并返回错误状态
    default:
        '''
         * We don't know what to map this to yet.
         '''
        # 格式化错误信息到错误缓冲区
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "unknown interface type %u",
            v);
        # 设置错误状态为PCAP_ERROR
        status = PCAP_ERROR;
        # 跳转到错误处理标签
        goto bad;
    }
#endif
#if _BSDI_VERSION - 0 >= 199510
    /* 如果 BSD/OS 版本大于等于 199510，则 SLIP 和 PPP 链路层头发生了变化 */
    switch (v) {

    case DLT_SLIP:
        v = DLT_SLIP_BSDOS;
        break;

    case DLT_PPP:
        v = DLT_PPP_BSDOS;
        break;

    case 11:    /*DLT_FR*/
        v = DLT_FRELAY;
        break;

    case 12:    /*DLT_C_HDLC*/
        v = DLT_CHDLC;
        break;
    }
#endif

#ifdef BIOCGDLTLIST
    /*
     * 我们知道默认的链路类型，现在确定该接口支持的所有 DLTs。
     * 如果这个操作失败并返回 EINVAL，这不是致命的；我们只是不能在后面使用这个特性。
     */
    if (get_dlt_list(fd, v, &bdl, p->errbuf) == -1) {
        status = PCAP_ERROR;
        goto bad;
    }
    p->dlt_count = bdl.bfl_len;
    p->dlt_list = bdl.bfl_list;

#ifdef __APPLE__
    /*
     * 监视模式的有趣之处，继续。
     *
     * 对于 10.5 和之后的版本，如上所述，支持监视模式的 802.1 适配器提供 DLT_EN10MB、DLT_IEEE802_11，可能还有一些 802.11 加无线电信息的 DLT_ 值。
     * 选择其中一个 802.11 DLT_ 值将打开监视模式。
     *
     * 因此，如果用户要求监视模式，我们过滤掉 DLT_EN10MB 值，因为在监视模式下无法获取该值；如果用户没有要求监视模式，我们过滤掉 802.11 DLT_ 值，因为选择这些值将打开监视模式。
     * 然后，在监视模式下，如果提供了 802.11 加无线电 DLT_ 值，我们尝试选择该值，否则我们尝试选择 DLT_IEEE802_11。
     */
    }
#elif defined(HAVE_BSD_IEEE80211)
    /*
     * 具有新的 802.11 ioctls 的 *BSD。
     * 我们是否需要监视模式？
     */
    if (p->opt.rfmon) {
        /*
         * 如果设置了 rfmon 选项，则尝试将接口设置为监控模式。
         */
        retv = monitor_mode(p, 1);
        if (retv != 0) {
            /*
             * 如果设置监控模式失败，则记录错误状态并跳转到 bad 标签处。
             */
            status = retv;
            goto bad;
        }

        /*
         * 接口已经处于监控模式。
         * 尝试找到最佳的 802.11 DLT_ 值，如果成功，则尝试切换到该模式（如果尚未处于该模式）。
         */
        new_dlt = find_802_11(&bdl);
        if (new_dlt != -1) {
            /*
             * 至少有一个 802.11 DLT_ 值。
             * new_dlt 是列表中最佳的 802.11 DLT_ 值。
             *
             * 如果我们想要的新模式不是默认模式，则尝试选择新模式。
             */
            if ((u_int)new_dlt != v) {
                if (ioctl(p->fd, BIOCSDLT, &new_dlt) != -1) {
                    /*
                     * 成功选择新模式；将其作为新的 DLT_ 值。
                     */
                    v = new_dlt;
                }
            }
        }
    }
#endif /* various platforms */
#endif /* BIOCGDLTLIST */

    /*
     * 如果这是一个以太网设备，并且我们没有 DLT_ 列表，
     * 给它一个包含 DLT_EN10MB 和 DLT_DOCSIS 的列表。
     * （这将给 802.11 接口 DLT_DOCSIS，这不是正确的做法，
     * 但是没有其他方法确定它是以太网还是 802.11 设备。）
     */
    if (v == DLT_EN10MB && p->dlt_count == 0) {
        // 分配包含两个 u_int 元素的内存空间
        p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
        /*
         * 如果分配失败，就让列表保持为空。
         */
        if (p->dlt_list != NULL) {
            // 设置列表的第一个元素为 DLT_EN10MB，第二个元素为 DLT_DOCSIS
            p->dlt_list[0] = DLT_EN10MB;
            p->dlt_list[1] = DLT_DOCSIS;
            // 设置列表元素个数为 2
            p->dlt_count = 2;
        }
    }
#ifdef PCAP_FDDIPAD
    if (v == DLT_FDDI)
        // 如果是 FDDI 设备，设置 fddipad 为 PCAP_FDDIPAD
        p->fddipad = PCAP_FDDIPAD;
    else
#endif
        // 否则，将 fddipad 设置为 0
        p->fddipad = 0;
    // 设置链路类型为 v
    p->linktype = v;

#if defined(BIOCGHDRCMPLT) && defined(BIOCSHDRCMPLT)
    /*
     * 如果定义了 BIOCSHDRCMPLT，执行 BIOCSHDRCMPLT 操作，
     * 以打开该标志，这样链路层源地址就不会被强制覆盖。
     * （我们应该忽略错误吗？我们应该只在写入模式下执行吗？）
     *
     * XXX - 我记得在一些 BSD 中有一些发送数据包的 bug - 检查 "bpf.c" 的 CVS 日志？
     */
    if (ioctl(fd, BIOCSHDRCMPLT, &spoof_eth_src) == -1) {
        // 如果执行失败，设置错误消息并返回错误状态
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCSHDRCMPLT");
        status = PCAP_ERROR;
        goto bad;
    }
#endif
    /* 设置超时 */
#ifdef HAVE_ZEROCOPY_BPF
    /*
     * 在零拷贝模式下，我们只使用 select() 中的超时。
     * XXX - 如果我们处于非阻塞模式，而 *应用程序* 使用 select()、poll()、kqueue 等，会怎么样？
     */
    if (p->opt.timeout && !pb->zerocopy) {
#else
    if (p->opt.timeout) {
#endif
        /*
         * XXX - is this seconds/nanoseconds in AIX?
         * (Treating it as such doesn't fix the timeout
         * problem described below.)
         *
         * XXX - Mac OS X 10.6 mishandles BIOCSRTIMEOUT in
         * 64-bit userland - it takes, as an argument, a
         * "struct BPF_TIMEVAL", which has 32-bit tv_sec
         * and tv_usec, rather than a "struct timeval".
         *
         * If this platform defines "struct BPF_TIMEVAL",
         * we check whether the structure size in BIOCSRTIMEOUT
         * is that of a "struct timeval" and, if not, we use
         * a "struct BPF_TIMEVAL" rather than a "struct timeval".
         * (That way, if the bug is fixed in a future release,
         * we will still do the right thing.)
         */
        struct timeval to;
#ifdef HAVE_STRUCT_BPF_TIMEVAL
        struct BPF_TIMEVAL bpf_to;

        // 如果定义了 "struct BPF_TIMEVAL"，则检查 BIOCSRTIMEOUT 中的结构大小是否为 "struct timeval"，如果不是，则使用 "struct BPF_TIMEVAL" 而不是 "struct timeval"
        if (IOCPARM_LEN(BIOCSRTIMEOUT) != sizeof(struct timeval)) {
            bpf_to.tv_sec = p->opt.timeout / 1000;
            bpf_to.tv_usec = (p->opt.timeout * 1000) % 1000000;
            // 如果 ioctl 调用失败，则设置错误消息并跳转到 bad 标签
            if (ioctl(p->fd, BIOCSRTIMEOUT, (caddr_t)&bpf_to) < 0) {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    errno, PCAP_ERRBUF_SIZE, "BIOCSRTIMEOUT");
                status = PCAP_ERROR;
                goto bad;
            }
        } else {
#endif
            to.tv_sec = p->opt.timeout / 1000;
            to.tv_usec = (p->opt.timeout * 1000) % 1000000;
            // 如果 ioctl 调用失败，则设置错误消息并跳转到 bad 标签
            if (ioctl(p->fd, BIOCSRTIMEOUT, (caddr_t)&to) < 0) {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    errno, PCAP_ERRBUF_SIZE, "BIOCSRTIMEOUT");
                status = PCAP_ERROR;
                goto bad;
            }
#ifdef HAVE_STRUCT_BPF_TIMEVAL
        }
#endif
    }

#ifdef    BIOCIMMEDIATE
    /*
     * Darren Reed notes that
     *
     *    On AIX (4.2 at least), if BIOCIMMEDIATE is not set, the
     *    timeout appears to be ignored and it waits until the buffer
     *    is filled before returning.  The result of not having it
     *    set is almost worse than useless if your BPF filter
     *    is reducing things to only a few packets (i.e. one every
     *    second or so).
     *
     * so we always turn BIOCIMMEDIATE mode on if this is AIX.
     *
     * For other platforms, we don't turn immediate mode on by default,
     * as that would mean we get woken up for every packet, which
     * probably isn't what you want for a packet sniffer.
     *
     * We set immediate mode if the caller requested it by calling
     * pcap_set_immediate() before calling pcap_activate().
     */
#ifndef _AIX
    // 如果不是在 AIX 系统上
    if (p->opt.immediate) {
#endif /* _AIX */
        // 设置 v 为 1
        v = 1;
        // 如果设置立即模式失败
        if (ioctl(p->fd, BIOCIMMEDIATE, &v) < 0) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "BIOCIMMEDIATE");
            // 设置状态为错误
            status = PCAP_ERROR;
            // 跳转到 bad 标签
            goto bad;
        }
#ifndef _AIX
    }
#endif /* _AIX */
#else /* BIOCIMMEDIATE */
    // 如果不支持立即模式
    if (p->opt.immediate) {
        /*
         * 不支持立即模式，失败。
         */
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Immediate mode not supported");
        // 设置状态为错误
        status = PCAP_ERROR;
        // 跳转到 bad 标签
        goto bad;
    }
#endif /* BIOCIMMEDIATE */

    // 如果设置了混杂模式
    if (p->opt.promisc) {
        /* 设置混杂模式，如果失败则警告 */
        if (ioctl(p->fd, BIOCPROMISC, NULL) < 0) {
            // 格式化错误消息
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "BIOCPROMISC");
            // 设置状态为混杂模式不支持的警告
            status = PCAP_WARNING_PROMISC_NOTSUP;
        }
    }

#ifdef BIOCSTSTAMP
    // 设置时间戳类型为 BPF_T_BINTIME
    v = BPF_T_BINTIME;
    // 如果设置时间戳类型失败
    if (ioctl(p->fd, BIOCSTSTAMP, &v) < 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCSTSTAMP");
        // 设置状态为错误
        status = PCAP_ERROR;
        // 跳转到 bad 标签
        goto bad;
    }
#endif /* BIOCSTSTAMP */

    // 获取缓冲区长度
    if (ioctl(fd, BIOCGBLEN, (caddr_t)&v) < 0) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCGBLEN");
        // 设置状态为错误
        status = PCAP_ERROR;
        // 跳转到 bad 标签
        goto bad;
    }
    // 设置 pcap_t 结构体的缓冲区大小
    p->bufsize = v;
#ifdef HAVE_ZEROCOPY_BPF
    // 如果不支持零拷贝
    if (!pb->zerocopy) {
#endif
    // 分配缓冲区内存
    p->buffer = malloc(p->bufsize);
    // 如果分配内存失败
    if (p->buffer == NULL) {
        // 格式化错误消息
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        // 设置状态为错误
        status = PCAP_ERROR;
        // 跳转到 bad 标签
        goto bad;
    }
#ifdef _AIX
    /* 由于某种奇怪的原因，这似乎可以防止我们在 AIX BPF 中遇到的 EFAULT 问题。 */
    memset(p->buffer, 0x0, p->bufsize);
#endif
#ifdef HAVE_ZEROCOPY_BPF
    }
#endif
    /*
     * 如果没有安装过滤程序，内核就无法知道快照长度应该是多少，因此不会进行快照。
     *
     * 因此，当我们打开设备时，我们会安装一个“接受所有”过滤器，并指定快照长度。
     */
    // 设置指令代码为接受所有数据
    total_insn.code = (u_short)(BPF_RET | BPF_K);
    // 跳转条件为0
    total_insn.jt = 0;
    // 跳转条件为0
    total_insn.jf = 0;
    // 设置快照长度为指定的长度
    total_insn.k = p->snapshot;

    // 设置过滤器程序的指令数组长度为1
    total_prog.bf_len = 1;
    // 设置过滤器程序的指令数组为total_insn
    total_prog.bf_insns = &total_insn;
    // 使用ioctl函数将过滤器程序设置到设备上
    if (ioctl(p->fd, BIOCSETF, (caddr_t)&total_prog) < 0) {
        // 如果设置失败，将错误信息格式化到错误缓冲区中
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCSETF");
        // 设置状态为错误
        status = PCAP_ERROR;
        // 跳转到错误处理部分
        goto bad;
    }
    /*
     * 在大多数 BPF 平台上，可以在 BPF 文件描述符上执行 "select()" 或 "poll()"，并且它可以正常工作，
     * 或者可以执行它，如果保持缓冲区已满，则返回 "可读"，但如果超时到期并且非阻塞读取将会返回 "可读"，
     * 如果保持缓冲区为空但存储缓冲区不为空，则旋转缓冲区并返回可用的数据包。
     *
     * 在后一种情况下，非阻塞读取将给您可用的数据包，这意味着您可以通过使用超时作为 "select()" 或 "poll()" 的超时，
     * 将 BPF 描述符设置为非阻塞模式，并从中读取，无论 "select()" 报告它是否可读或不可读。
     *
     * 但是，在 FreeBSD 4.3 和 4.4 中，如果计时器到期，"select()" 和 "poll()" 不会唤醒并返回 "可读"，
     * 并且如果保持缓冲区为空，非阻塞读取将返回 EWOULDBLOCK，即使存储缓冲区非空。
     *
     * 这意味着所讨论的解决方法将无法工作。
     *
     * 因此，在 FreeBSD 4.3 和 4.4 上，我们将 "p->selectable_fd" 设置为 -1，这意味着 "抱歉，在这里不能使用 'select()' 或 'poll()'"。
     * 在所有其他 BPF 平台上，我们将其设置为 BPF 设备的文件描述符；在 NetBSD、OpenBSD 和 Darwin 中，如果保持缓冲区为空并且存储缓冲区不为空，
     * 非阻塞读取将旋转缓冲区并返回可用的数据包（在足够新的 OpenBSD 版本中，"select()" 和 "poll()" 应该能够正常工作）。
     *
     * XXX - AIX 呢？
     */
    p->selectable_fd = p->fd;    /* 假设 select() 正常工作，直到我们知道情况有所不同 */
    if (have_osinfo) {
        /*
         * 如果已经获取了操作系统信息，可以进行操作系统的检查
         */
        if (strcmp(osinfo.sysname, "FreeBSD") == 0) {
            // 如果操作系统是 FreeBSD，并且版本号以 "4.3-" 或 "4.4-" 开头，将可选择的文件描述符设置为 -1
            if (strncmp(osinfo.release, "4.3-", 4) == 0 ||
                 strncmp(osinfo.release, "4.4-", 4) == 0)
                p->selectable_fd = -1;
        }
    }

    // 设置读取操作
    p->read_op = pcap_read_bpf;
    // 设置注入操作
    p->inject_op = pcap_inject_bpf;
    // 设置过滤器操作
    p->setfilter_op = pcap_setfilter_bpf;
    // 设置数据流方向操作
    p->setdirection_op = pcap_setdirection_bpf;
    // 设置数据链路类型操作
    p->set_datalink_op = pcap_set_datalink_bpf;
    // 获取非阻塞状态操作
    p->getnonblock_op = pcap_getnonblock_bpf;
    // 设置非阻塞状态操作
    p->setnonblock_op = pcap_setnonblock_bpf;
    // 获取统计信息操作
    p->stats_op = pcap_stats_bpf;
    // 清理操作
    p->cleanup_op = pcap_cleanup_bpf;

    // 返回状态
    return (status);
 bad:
    // 执行清理操作
    pcap_cleanup_bpf(p);
    // 返回状态
    return (status);
/*
 * 检查指定的接口是否可以被 BPF 绑定；
 * 如果我们在尝试绑定时失败并返回 PCAP_ERROR_NO_SUCH_DEVICE（这意味着我们在尝试绑定时得到了 ENXIO，这意味着这个接口不在附加到 BPF 的接口列表中），则返回 0，否则返回 1。
 */
static int
check_bpf_bindable(const char *name)
{
    int fd;
    char errbuf[PCAP_ERRBUF_SIZE];

    /*
     * 在 macOS 上，如果设备名称以 "wlt" 开头，我们不进行此检查；
     * 至少某些版本的 macOS（实际上，当时称为 "Mac OS X"…）通过为每个无线适配器提供单独的 "监视模式" 设备来提供监视模式捕获，而不是通过实现 {Free,Net,Open,DragonFly}BSD 提供的 ioctls。打开该设备会将适配器置于监视模式，这至少对于某些适配器会导致它们与其关联的网络断开连接。
     *
     * 相反，我们尝试打开相应的 "en" 设备（这样我们就不会得到一个只包括 wlt 设备的适配器列表，对于没有足够权限打开捕获设备的用户来说）。
     */
#ifdef __APPLE__
    if (strncmp(name, "wlt", 3) == 0) {
        char *en_name;
        size_t en_name_len;

        /*
         * 尝试为 "en" 设备的名称分配缓冲区。
         */
        en_name_len = strlen(name) - 1;
        en_name = malloc(en_name_len + 1);
        if (en_name == NULL) {
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "malloc");
            return (-1);
        }
        strcpy(en_name, "en");
        strcat(en_name, name + 3);
        fd = bpf_open_and_bind(en_name, errbuf);
        free(en_name);
    } else
#endif /* __APPLE */
    fd = bpf_open_and_bind(name, errbuf);
    # 如果文件描述符小于0，表示出现错误
    if (fd < 0) {
        /*
         * 错误 - 是 PCAP_ERROR_NO_SUCH_DEVICE 吗？
         */
        if (fd == PCAP_ERROR_NO_SUCH_DEVICE) {
            /*
             * 是的，所以我们无法绑定到这个设备，因为它不是 BPF 支持的内容。
             */
            return (0);
        }
        /*
         * 不是，所以我们不知道它是否被支持；
         * 假设它被支持，这样用户至少可以尝试打开它并报告错误
         * （这可能是“您没有权限打开 BPF 设备”；
         * 报告这些接口意味着用户会问“为什么我尝试捕获时出现权限错误”而不是“为什么我看不到任何接口”，
         * 使潜在问题更清晰）。
         */
        return (1);
    }

    /*
     * 成功。
     */
    close(fd);
    return (1);
}
// 如果定义了 __FreeBSD__ 和 SIOCIFCREATE2，则执行以下函数
static int
get_usb_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
    /*
     * XXX - 如果有一种方法可以确定是否有设备插入到给定的 USB 总线中，
     * 使用该方法来确定该设备是否“连接”。
     */
    return (0);
}

static int
finddevs_usb(pcap_if_list_t *devlistp, char *errbuf)
{
    DIR *usbdir;
    struct dirent *usbitem;
    size_t name_max;
    char *name;

    /*
     * 我们可能有 USB 抓包支持，所以尝试查找 USB 接口。
     *
     * 我们想要为每个 USB 总线报告一个 usbusN 设备，但是
     * 对于它们可能存在或不存在 - 如果不存在，我们会创建一个。
     *
     * 因此，我们在 /dev/usb 中查找所有总线，并为每个总线创建一个“usbusN”设备。
     */
    usbdir = opendir("/dev/usb");
    if (usbdir == NULL) {
        /*
         * 只是放弃。
         */
        return (0);
    }

    /*
     * 留出足够的空间来存放 32 位（10 位数字）的总线号。
     * 是的，这有点过分了，但我们不会使用
     * 缓冲区很长时间。
     */
    name_max = USBUS_PREFIX_LEN + 10 + 1;
    name = malloc(name_max);
    if (name == NULL) {
        closedir(usbdir);
        return (0);
    }
    # 遍历 USB 设备目录中的每个项目
    while ((usbitem = readdir(usbdir)) != NULL) {
        # 定义指针变量和长度变量
        char *p;
        size_t busnumlen;

        # 如果项目名为"."或"..", 则忽略
        if (strcmp(usbitem->d_name, ".") == 0 ||
            strcmp(usbitem->d_name, "..") == 0) {
            '''
            * 忽略这些项目。
            '''
            continue;
        }
        # 在项目名中查找第一个"."的位置
        p = strchr(usbitem->d_name, '.');
        if (p == NULL)
            continue;
        # 计算设备号的长度
        busnumlen = p - usbitem->d_name;
        # 将设备名拼接成完整的名称
        memcpy(name, usbus_prefix, USBUS_PREFIX_LEN);
        memcpy(name + USBUS_PREFIX_LEN, usbitem->d_name, busnumlen);
        *(name + USBUS_PREFIX_LEN + busnumlen) = '\0';
        '''
        * 这个目录中的每个 USB 设备都有一个条目，
        * 而不是每个总线都有一个条目；如果总线上有多个设备，
        * 那么该总线将有多个条目，因此我们需要避免为每个总线添加多个捕获设备。
        '''
        # 如果设备列表中没有找到或添加设备，则释放内存并返回错误
        if (find_or_add_dev(devlistp, name, PCAP_IF_UP,
            get_usb_if_flags, NULL, errbuf) == NULL) {
            free(name);
            closedir(usbdir);
            return (PCAP_ERROR);
        }
    }
    # 释放设备名称的内存
    free(name);
    # 关闭 USB 设备目录
    closedir(usbdir);
    # 返回成功
    return (0);
}
#endif

/*
 * 使用 SIOCGIFMEDIA 获取设备的附加标志。
 */
#ifdef SIOCGIFMEDIA
static int
get_if_flags(const char *name, bpf_u_int32 *flags, char *errbuf)
{
    int sock;
    struct ifmediareq req;

    // 创建一个 AF_INET 套接字，用于获取媒体信息
    sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock == -1) {
        // 如果创建套接字失败，返回错误信息
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno,
            "Can't create socket to get media information for %s",
            name);
        return (-1);
    }
    // 清空 req 结构体
    memset(&req, 0, sizeof(req));
    // 将设备名拷贝到 req 结构体中
    pcap_strlcpy(req.ifm_name, name, sizeof(req.ifm_name));
    // 使用 SIOCGIFMEDIA 命令获取设备信息
    if (ioctl(sock, SIOCGIFMEDIA, &req) < 0) {
        // 如果获取失败，检查错误码并返回相应的错误信息
        if (errno == EOPNOTSUPP || errno == EINVAL || errno == ENOTTY ||
            errno == ENODEV || errno == EPERM
#ifdef EPWROFF
            || errno == EPWROFF
#endif
            ) {
            /*
             * 不支持，因此我们无法提供任何额外信息。假设这意味着“连接”与“断开连接”不适用。
             *
             * Apple的pktap设备的ioctl例程很烦人地在检查“你是root吗？”之前检查ioctl是否有效，因此对于无效的SIOCGIFMEDIA，除非你是root，否则它会返回EPERM，而不是ENOTSUP。因此，就像我们在Linux上对一些ethtool ioctl做的一样，它犯了同样的错误，我们也将EPERM视为“不支持”的意思。
             *
             * 看起来Apple的llw0设备，似乎是Skywalk子系统的一部分：
             *
             *    http://newosxbook.com/bonus/vol1ch16.html
             *
             * 有时对该ioctl返回EPWROFF（“设备电源已关闭”），因此我们将*那*视为我们无法获取连接状态的另一个指示。（如果它*不是*“已关闭”，它被报告为无线设备，包括活动/非活动状态。）
             */
            *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
            close(sock);
            return (0);
        }
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno,
            "SIOCGIFMEDIA on %s failed", name);
        close(sock);
        return (-1);
    }
    close(sock);

    /*
     * 好的，这是什么类型的网络？
     */
    switch (IFM_TYPE(req.ifm_active)) {

    case IFM_IEEE80211:
        /*
         * 无线网络。
         */
        *flags |= PCAP_IF_WIRELESS;
        break;
    }

    /*
     * 我们知道它是否连接了吗？
     */
    # 检查接口是否有效
    if (req.ifm_status & IFM_AVALID) {
        /*
         * 是有效的。
         */
        # 检查接口是否活跃
        if (req.ifm_status & IFM_ACTIVE) {
            /*
             * 它是连接的。
             */
            # 设置连接状态为已连接
            *flags |= PCAP_IF_CONNECTION_STATUS_CONNECTED;
        } else {
            /*
             * 它是断开的。
             */
            # 设置连接状态为已断开
            *flags |= PCAP_IF_CONNECTION_STATUS_DISCONNECTED;
        }
    }
    # 返回 0 表示成功
    return (0);
}
#else
static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags, char *errbuf _U_)
{
    /*
     * Nothing we can do other than mark loopback devices as "the connected/disconnected status doesn't apply".
     *
     * XXX - on Solaris, can we do what the dladm command does,
     * i.e. get a connected/disconnected indication from a kstat?
     * (Note that you can also get the link speed, and possibly
     * other information, from a kstat as well.)
     */
    if (*flags & PCAP_IF_LOOPBACK) {
        /*
         * Loopback devices aren't wireless, and "connected"/
         * "disconnected" doesn't apply to them.
         */
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        return (0);
    }
    return (0);
}
#endif

int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
    /*
     * Get the list of regular interfaces first.
     */
    if (pcap_findalldevs_interfaces(devlistp, errbuf, check_bpf_bindable,
        get_if_flags) == -1)
        return (-1);    /* failure */

#if defined(__FreeBSD__) && defined(SIOCIFCREATE2)
    if (finddevs_usb(devlistp, errbuf) == -1)
        return (-1);
#endif

    return (0);
}

#ifdef HAVE_BSD_IEEE80211
static int
monitor_mode(pcap_t *p, int set)
{
    struct pcap_bpf *pb = p->priv;
    int sock;
    struct ifmediareq req;
    IFM_ULIST_TYPE *media_list;
    int i;
    int can_do;
    struct ifreq ifr;

    sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "can't open socket");
        return (PCAP_ERROR);
    }

    memset(&req, 0, sizeof req);
    pcap_strlcpy(req.ifm_name, p->opt.device, sizeof req.ifm_name);

    /*
     * Find out how many media types we have.
     */
    # 如果无法获取媒体类型
    if (ioctl(sock, SIOCGIFMEDIA, &req) < 0) {
        /*
         * 无法获取媒体类型。
         */
        switch (errno) {

        case ENXIO:
            /*
             * 没有这样的设备。
             *
             * 没有更多可说的，所以清除错误消息。
             */
            p->errbuf[0] = '\0';
            close(sock);
            return (PCAP_ERROR_NO_SUCH_DEVICE);

        case EINVAL:
            /*
             * 接口不支持 SIOC{G,S}IFMEDIA。
             */
            close(sock);
            return (PCAP_ERROR_RFMON_NOTSUP);

        default:
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "SIOCGIFMEDIA");
            close(sock);
            return (PCAP_ERROR);
        }
    }
    # 如果媒体类型计数为0
    if (req.ifm_count == 0) {
        /*
         * 没有媒体类型。
         */
        close(sock);
        return (PCAP_ERROR_RFMON_NOTSUP);
    }

    /*
     * 分配一个缓冲区来保存所有媒体类型，并获取媒体类型。
     */
    media_list = malloc(req.ifm_count * sizeof(*media_list));
    if (media_list == NULL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        close(sock);
        return (PCAP_ERROR);
    }
    req.ifm_ulist = media_list;
    if (ioctl(sock, SIOCGIFMEDIA, &req) < 0) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "SIOCGIFMEDIA");
        free(media_list);
        close(sock);
        return (PCAP_ERROR);
    }

    /*
     * 寻找一个802.11“自动”媒体类型。
     * 我们假设所有802.11适配器都有该媒体类型，
     * 并且它将携带支持监控模式的标志。
     */
    can_do = 0;
    # 遍历接口媒体列表中的每个接口
    for (i = 0; i < req.ifm_count; i++) {
        # 检查媒体类型是否为 IEEE80211，并且子类型是否为 IFM_AUTO
        if (IFM_TYPE(media_list[i]) == IFM_IEEE80211
            && IFM_SUBTYPE(media_list[i]) == IFM_AUTO) {
            # 如果是 IEEE80211 并且支持自动模式，则继续判断是否支持监控模式
            /* OK, does it do monitor mode? */
            if (media_list[i] & IFM_IEEE80211_MONITOR) {
                # 如果支持监控模式，则设置 can_do 为 1 并跳出循环
                can_do = 1;
                break;
            }
        }
    }
    # 释放媒体列表的内存
    free(media_list);
    # 如果不支持监控模式，则关闭套接字并返回错误码 PCAP_ERROR_RFMON_NOTSUP
    if (!can_do) {
        /*
         * This adapter doesn't support monitor mode.
         */
        close(sock);
        return (PCAP_ERROR_RFMON_NOTSUP);
    }
    if (set) {
        /*
         * 不仅仅检查是否可以启用监视模式，
         * 如果尚未启用，则启用监视模式。
         */
        if ((req.ifm_current & IFM_IEEE80211_MONITOR) == 0) {
            /*
             * 监视模式当前未开启，因此打开它，
             * 并记住在关闭 pcap_t 时应该关闭它。
             */

            /*
             * 如果尚未这样做，安排在退出时调用 "pcap_close_all()"。
             */
            if (!pcap_do_addexit(p)) {
                /*
                 * "atexit()" 失败；不要将接口置于监视模式，直接放弃。
                 */
                close(sock);
                return (PCAP_ERROR);
            }
            memset(&ifr, 0, sizeof(ifr));
            (void)pcap_strlcpy(ifr.ifr_name, p->opt.device,
                sizeof(ifr.ifr_name));
            ifr.ifr_media = req.ifm_current | IFM_IEEE80211_MONITOR;
            if (ioctl(sock, SIOCSIFMEDIA, &ifr) == -1) {
                pcap_fmt_errmsg_for_errno(p->errbuf,
                    PCAP_ERRBUF_SIZE, errno, "SIOCSIFMEDIA");
                close(sock);
                return (PCAP_ERROR);
            }

            pb->must_do_on_close |= MUST_CLEAR_RFMON;

            /*
             * 将其添加到在退出时关闭的 pcap 列表中。
             */
            pcap_add_to_pcaps_to_close(p);
        }
    }
    return (0);
#endif /* HAVE_BSD_IEEE80211 */

#if defined(BIOCGDLTLIST) && (defined(__APPLE__) || defined(HAVE_BSD_IEEE80211))
/*
 * 检查是否有任何 802.11 链路层类型；如果找到，则返回最佳的 802.11 链路层类型，否则返回 -1。
 *
 * DLT_IEEE802_11_RADIO，带有 radiotap 头的，被认为是最佳的 802.11 链路层类型；任何其他 802.11 加上无线电头的类型是次佳的；没有无线电信息的 802.11 是最差的。
 */
static int
find_802_11(struct bpf_dltlist *bdlp)
{
    int new_dlt;
    u_int i;

    /*
     * 扫描 DLT_ 值列表，查找 802.11 值，并选择其中最佳的。
     */
    new_dlt = -1;
    for (i = 0; i < bdlp->bfl_len; i++) {
        switch (bdlp->bfl_list[i]) {

        case DLT_IEEE802_11:
            /*
             * 802.11，但没有无线电。
             *
             * 提供此选项，并将其选择为新模式，除非我们已经找到带有无线电信息的 802.11 头。
             */
            if (new_dlt == -1)
                new_dlt = bdlp->bfl_list[i];
            break;

#ifdef DLT_PRISM_HEADER
        case DLT_PRISM_HEADER:
#endif
#ifdef DLT_AIRONET_HEADER
        case DLT_AIRONET_HEADER:
#endif
        case DLT_IEEE802_11_RADIO_AVS:
            /*
             * 802.11 with radio, but not radiotap.
             *
             * Offer this, and select it as the new mode
             * unless we've already found the radiotap DLT_.
             */
            // 如果新的数据链路类型不是 DLT_IEEE802_11_RADIO，则将其设为新的数据链路类型
            if (new_dlt != DLT_IEEE802_11_RADIO)
                new_dlt = bdlp->bfl_list[i];
            break;

        case DLT_IEEE802_11_RADIO:
            /*
             * 802.11 with radiotap.
             *
             * Offer this, and select it as the new mode.
             */
            // 将数据链路类型设为 DLT_IEEE802_11_RADIO
            new_dlt = bdlp->bfl_list[i];
            break;

        default:
            /*
             * Not 802.11.
             */
            // 不是 802.11 数据链路类型，不做任何操作
            break;
        }
    }

    return (new_dlt);
}
#endif /* defined(BIOCGDLTLIST) && (defined(__APPLE__) || defined(HAVE_BSD_IEEE80211)) */

#if defined(__APPLE__) && defined(BIOCGDLTLIST)
/*
 * Remove non-802.11 header types from the list of DLT_ values, as we're in
 * monitor mode, and those header types aren't supported in monitor mode.
 */
static void
remove_non_802_11(pcap_t *p)
{
    int i, j;

    /*
     * Scan the list of DLT_ values and discard non-802.11 ones.
     */
    // 遍历 DLT_ 值的列表并丢弃非 802.11 的值
    j = 0;
    for (i = 0; i < p->dlt_count; i++) {
        switch (p->dlt_list[i]) {

        case DLT_EN10MB:
        case DLT_RAW:
            /*
             * Not 802.11.  Don't offer this one.
             */
            // 不是 802.11 数据链路类型，不提供这个类型
            continue;

        default:
            /*
             * Just copy this mode over.
             */
            // 只是复制这个模式
            break;
        }

        /*
         * Copy this DLT_ value to its new position.
         */
        // 将这个 DLT_ 值复制到新的位置
        p->dlt_list[j] = p->dlt_list[i];
        j++;
    }

    /*
     * Set the DLT_ count to the number of entries we copied.
     */
    // 将 DLT_ 计数设置为我们复制的条目数
    p->dlt_count = j;
}

/*
 * Remove 802.11 link-layer types from the list of DLT_ values, as
 * we're not in monitor mode, and those DLT_ values will switch us
 * to monitor mode.
 */
static void
remove_802_11(pcap_t *p)
{
    int i, j;
    /*
     * 扫描 DLT_ 值的列表，并丢弃 802.11 值。
     */
    j = 0;  // 初始化变量 j 为 0
    for (i = 0; i < p->dlt_count; i++) {  // 遍历 dlt_list 列表
        switch (p->dlt_list[i]) {  // 切换到当前 dlt_list[i] 的值

        case DLT_IEEE802_11:  // 如果当前值为 DLT_IEEE802_11
#ifdef DLT_PRISM_HEADER
        // 如果定义了 DLT_PRISM_HEADER，则执行下面的语句
        case DLT_PRISM_HEADER:
#endif
#ifdef DLT_AIRONET_HEADER
        // 如果定义了 DLT_AIRONET_HEADER，则执行下面的语句
        case DLT_AIRONET_HEADER:
#endif
        // 以下两个 case 语句无条件执行
        case DLT_IEEE802_11_RADIO:
        case DLT_IEEE802_11_RADIO_AVS:
#ifdef DLT_PPI
        // 如果定义了 DLT_PPI，则执行下面的语句
        case DLT_PPI:
#endif
            /*
             * 802.11.  Don't offer this one.
             */
            // 如果是以上几种情况，则跳过当前循环，继续下一次循环
            continue;

        default:
            /*
             * Just copy this mode over.
             */
            // 默认情况下，执行以下语句
            break;
        }

        /*
         * Copy this DLT_ value to its new position.
         */
        // 将当前 DLT_ 值复制到新的位置
        p->dlt_list[j] = p->dlt_list[i];
        j++;
    }

    /*
     * Set the DLT_ count to the number of entries we copied.
     */
    // 将 DLT_ 计数设置为我们复制的条目数
    p->dlt_count = j;
}
#endif /* defined(__APPLE__) && defined(BIOCGDLTLIST) */

static int
pcap_setfilter_bpf(pcap_t *p, struct bpf_program *fp)
{
    struct pcap_bpf *pb = p->priv;

    /*
     * Free any user-mode filter we might happen to have installed.
     */
    // 释放任何可能已安装的用户模式过滤器
    pcap_freecode(&p->fcode);

    /*
     * Try to install the kernel filter.
     */
    // 尝试安装内核过滤器
    if (ioctl(p->fd, BIOCSETF, (caddr_t)fp) == 0) {
        /*
         * It worked.
         */
        // 它起作用了
        pb->filtering_in_kernel = 1;    /* filtering in the kernel */

        /*
         * Discard any previously-received packets, as they might
         * have passed whatever filter was formerly in effect, but
         * might not pass this filter (BIOCSETF discards packets
         * buffered in the kernel, so you can lose packets in any
         * case).
         */
        // 丢弃任何先前接收的数据包，因为它们可能通过以前生效的任何过滤器，但可能无法通过此过滤器（BIOCSETF 会丢弃内核中缓冲的数据包，因此在任何情况下都可能丢失数据包）
        p->cc = 0;
        return (0);
    }
    /*
     * 如果失败了，可能是因为程序无效或太大导致的 EINVAL 错误。
     * 我们自己验证程序；如果我们喜欢它（目前允许后向分支，以支持 protochain），
     * 就在用户空间运行它。（对于用户空间来说，没有“太大”的概念。）
     *
     * 否则，就放弃。
     * XXX - 如果将程序复制到内核失败，我们将得到 EINVAL 错误，而不是像某些内核上的 EFAULT 错误。
     */
    if (errno != EINVAL) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "BIOCSETF");
        return (-1);
    }

    /*
     * install_bpf_program() 验证程序。
     *
     * XXX - 如果内核中已经有过滤器怎么办？
     */
    if (install_bpf_program(p, fp) < 0)
        return (-1);
    pb->filtering_in_kernel = 0;    /* 在用户空间进行过滤 */
    return (0);
# 设置数据包的方向标志：在一个转发单一设备上，我们接受哪些数据包？进入、离开还是两者都接受？
#if defined(BIOCSDIRECTION)
static int
pcap_setdirection_bpf(pcap_t *p, pcap_direction_t d)
{
    u_int direction;
    const char *direction_name;

    /*
     * FreeBSD and NetBSD.
     */
    switch (d) {

    case PCAP_D_IN:
        /*
         * 只接受进入的数据包，不接受离开的数据包。
         */
        direction = BPF_D_IN;
        direction_name = "\"incoming only\"";
        break;

    case PCAP_D_OUT:
        /*
         * 只接受离开的数据包，不接受进入的数据包。
         */
        direction = BPF_D_OUT;
        direction_name = "\"outgoing only\"";
        break;

    default:
        /*
         * 进入和离开的数据包都接受。
         * 在这一点上，可以保证 d 是一个有效的方向值，所以如果不是 PCAP_D_IN 或 PCAP_D_OUT，我们知道这是 PCAP_D_INOUT。
         */
        direction = BPF_D_INOUT;
        direction_name = "\"incoming and outgoing\"";
        break;
    }

    if (ioctl(p->fd, BIOCSDIRECTION, &direction) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "Cannot set direction to %s", direction_name);
        return (-1);
    }
    return (0);
}
#elif defined(BIOCSDIRFILT)
static int
pcap_setdirection_bpf(pcap_t *p, pcap_direction_t d)
{
    u_int dirfilt;
    const char *direction_name;

    /*
     * OpenBSD; same functionality, different names, different
     * semantics (the flags mean "*don't* capture packets in
     * that direction", not "*capture only* packets in that
     * direction").
     */
    switch (d) {
    case PCAP_D_IN:
        /*
         * 设置过滤条件为只接收传入的数据包，不包括传出的数据包。
         */
        dirfilt = BPF_DIRECTION_OUT;
        direction_name = "\"incoming only\"";
        break;

    case PCAP_D_OUT:
        /*
         * 设置过滤条件为只接收传出的数据包，不包括传入的数据包。
         */
        dirfilt = BPF_DIRECTION_IN;
        direction_name = "\"outgoing only\"";
        break;

    default:
        /*
         * 设置过滤条件为同时接收传入和传出的数据包。
         * 在这一点上，可以保证 d 是一个有效的方向值，所以如果不是 PCAP_D_IN 或 PCAP_D_OUT，就知道这是 PCAP_D_INOUT。
         */
        dirfilt = 0;
        direction_name = "\"incoming and outgoing\"";
        break;
    }
    if (ioctl(p->fd, BIOCSDIRFILT, &dirfilt) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "Cannot set direction to %s", direction_name);
        return (-1);
    }
    return (0);
}
#elif defined(BIOCSSEESENT)
static int
pcap_setdirection_bpf(pcap_t *p, pcap_direction_t d)
{
    u_int seesent;
    const char *direction_name;

    /*
     * OS with just BIOCSSEESENT.
     */
    switch (d) {

    case PCAP_D_IN:
        /*
         * Incoming, but not outgoing, so we don't want to
         * see transmitted packets.
         */
        seesent = 0;
        direction_name = "\"incoming only\"";
        break;

    case PCAP_D_OUT:
        /*
         * Outgoing, but not incoming; we can't specify that.
         */
        snprintf(p->errbuf, sizeof(p->errbuf),
            "Setting direction to \"outgoing only\" is not supported on this device");
        return (-1);

    default:
        /*
         * Incoming and outgoing, so we want to see transmitted
         * packets.
         *
         * It's guaranteed, at this point, that d is a valid
         * direction value, so we know that this is PCAP_D_INOUT
         * if it's not PCAP_D_IN or PCAP_D_OUT.
         */
        seesent = 1;
        direction_name = "\"incoming and outgoing\"";
        break;
    }

    if (ioctl(p->fd, BIOCSSEESENT, &seesent) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "Cannot set direction to %s", direction_name);
        return (-1);
    }
    return (0);
}
#else
static int
pcap_setdirection_bpf(pcap_t *p, pcap_direction_t d _U_)
{
    (void) snprintf(p->errbuf, sizeof(p->errbuf),
        "Setting direction is not supported on this device");
    return (-1);
}
#endif

#ifdef BIOCSDLT
static int
pcap_set_datalink_bpf(pcap_t *p, int dlt)
{
    if (ioctl(p->fd, BIOCSDLT, &dlt) == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
            errno, "Cannot set DLT %d", dlt);
        return (-1);
    }
    return (0);
}
#else
static int
pcap_set_datalink_bpf(pcap_t *p _U_, int dlt _U_)
{
    return (0);
}
#endif

/*
 * Platform-specific information.
 */
const char *
pcap_lib_version(void)
{
#ifdef HAVE_ZEROCOPY_BPF
    // 如果支持零拷贝 BPF，则返回带有 zerocopy 支持的版本字符串
    return (PCAP_VERSION_STRING " (with zerocopy support)");
#else
    // 如果不支持零拷贝 BPF，则返回普通版本字符串
    return (PCAP_VERSION_STRING);
#endif
}
```