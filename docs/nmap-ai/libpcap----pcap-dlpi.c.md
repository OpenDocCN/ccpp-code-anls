# `nmap\libpcap\pcap-dlpi.c`

```
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中，必须保留以上版权声明和本段文字，包括在其全部文档或其他材料中
 * 在包含二进制代码的发布中，必须在文档或其他材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料必须显示以下声明：
 * “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 此代码由Atanu Ghosh（atanu@cs.ucl.ac.uk）贡献，后来由Guy Harris（guy@alum.mit.edu）、Mark Pizzolato
 * <List-tcpdump-workers@subscriptions.pizzolato.net>、Mark C. Brown（mbrown@hp.com）和Sagun Shakya <Sagun.Shakya@Sun.COM>修改
 */
/*
 * Packet capture routine for DLPI under SunOS 5, HP-UX 9/10/11, and AIX.
 *
 * Notes:
 *
 *    - The DLIOCRAW ioctl() is specific to SunOS.
 *      DLIOCRAW ioctl()用于SunOS特定的操作
 *
 *    - There is a bug in bufmod(7) such that setting the snapshot
 *      length results in data being left of the front of the packet.
 *      在bufmod(7)中存在一个bug，设置快照长度会导致数据留在数据包的前面
 *
 *    - It might be desirable to use pfmod(7) to filter packets in the
 *      kernel when possible.
 *      在可能的情况下，使用pfmod(7)在内核中过滤数据包可能是可取的
 *
 *    - An older version of the HP-UX DLPI Programmer's Guide, which
 *      I think was advertised as the 10.20 version, used to be available
 *      at
 *
 *            http://docs.hp.com/hpux/onlinedocs/B2355-90093/B2355-90093.html
 *
 *      but is no longer available; it can still be found at
 *
 *            http://h21007.www2.hp.com/dspp/files/unprotected/Drivers/Docs/Refs/B2355-90093.pdf
 *
 *      in PDF form.
 *      HP-UX DLPI程序员指南的旧版本，我认为是10.20版本，曾经可以在上述网址找到，但现在不再可用；它仍然可以以PDF形式在另一个网址找到
 *
 *    - The HP-UX 10.x, 11.0, and 11i v1.6 version of the HP-UX DLPI
 *      Programmer's Guide, which I think was once advertised as the
 *      11.00 version is available at
 *
 *            http://docs.hp.com/en/B2355-90139/index.html
 *
 *    - The HP-UX 11i v2 version of the HP-UX DLPI Programmer's Guide
 *      is available at
 *
 *            http://docs.hp.com/en/B2355-90871/index.html
 *
 *    - All of the HP documents describe raw-mode services, which are
 *      what we use if DL_HP_RAWDLS is defined.  XXX - we use __hpux
 *      in some places to test for HP-UX, but use DL_HP_RAWDLS in
 *      other places; do we support any versions of HP-UX without
 *      DL_HP_RAWDLS?
 *      所有的HP文档描述了原始模式服务，如果定义了DL_HP_RAWDLS，我们将使用它们。在某些地方我们使用__hpux来测试HP-UX，但在其他地方使用DL_HP_RAWDLS；我们是否支持没有DL_HP_RAWDLS的任何HP-UX版本？
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#ifdef HAVE_SYS_BUFMOD_H
#include <sys/bufmod.h>
#endif
#include <sys/dlpi.h>
#ifdef HAVE_SYS_DLPI_EXT_H
#include <sys/dlpi_ext.h>
#endif
#ifdef HAVE_HPUX9
#include <sys/socket.h>
#endif
#ifdef DL_HP_PPA_REQ
#include <sys/stat.h>
#endif
#include <sys/stream.h>
#if defined(HAVE_SOLARIS) && defined(HAVE_SYS_BUFMOD_H)
#include <sys/systeminfo.h>
#endif

#ifdef HAVE_HPUX9
#include <net/if.h>
#endif

#ifdef HAVE_HPUX9
#include <nlist.h>
#endif
#include <errno.h>
#include <fcntl.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stropts.h>
#include <unistd.h>
#include <limits.h>

#include "pcap-int.h"
#include "dlpisubs.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#if defined(__hpux)
  /*
   * HP-UX has a /dev/dlpi device; you open it and set the PPA of the actual
   * network device you want.
   */
  #define HAVE_DEV_DLPI
#elif defined(_AIX)
  /*
   * AIX has a /dev/dlpi directory, with devices named after the interfaces
   * underneath it.
   */
  #define PCAP_DEV_PREFIX "/dev/dlpi"
#elif defined(HAVE_SOLARIS)
  /*
   * Solaris has devices named after the interfaces underneath /dev.
   */
  #define PCAP_DEV_PREFIX "/dev"
#endif

#define    MAXDLBUF    8192

/* Forwards */
static char *split_dname(char *, u_int *, char *);
static int dl_doattach(int, int, char *);
#ifdef DL_HP_RAWDLS
static int dl_dohpuxbind(int, char *);
#endif
static int dlpromiscon(pcap_t *, bpf_u_int32);
static int dlbindreq(int, bpf_u_int32, char *);
static int dlbindack(int, char *, char *, int *);
static int dlokack(int, const char *, char *, char *, int *);
static int dlinforeq(int, char *);
static int dlinfoack(int, char *, char *);

#ifdef HAVE_DL_PASSIVE_REQ_T
static void dlpassive(int, char *);
#endif

#ifdef DL_HP_RAWDLS
static int dlrawdatareq(int, const u_char *, int);
#endif
static int recv_ack(int, int, const char *, char *, char *, int *);
static char *dlstrerror(char *, size_t, bpf_u_int32);
static char *dlprim(char *, size_t, bpf_u_int32);
#if defined(HAVE_SOLARIS) && defined(HAVE_SYS_BUFMOD_H)
#define GET_RELEASE_BUFSIZE    32
static void get_release(char *, size_t, bpf_u_int32 *, bpf_u_int32 *,
    bpf_u_int32 *);
#endif
static int send_request(int, char *, int, char *, char *);
#ifdef HAVE_HPUX9
static int dlpi_kread(int, off_t, void *, u_int, char *);
#endif
#ifdef HAVE_DEV_DLPI
# 定义一个静态函数，获取 DLPI 的 PPA 值
static int get_dlpi_ppa(int, const char *, u_int, u_int *, char *);

/*
 * 将一个缓冲区强制转换为 "union DL_primitives"，避免编译器警告
 */
#define MAKE_DL_PRIMITIVES(ptr)    ((union DL_primitives *)(void *)(ptr))

static int
pcap_read_dlpi(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    int cc;  # 用于存储读取的字节数
    u_char *bp;  # 指向数据缓冲区的指针
    int flags;  # 用于存储标志位
    bpf_u_int32 ctlbuf[MAXDLBUF];  # 用于存储控制信息的缓冲区
    struct strbuf ctl = {  # 定义控制信息的结构体
        MAXDLBUF,  # 最大长度
        0,  # 实际长度
        (char *)ctlbuf  # 缓冲区指针
    };
    struct strbuf data;  # 定义数据信息的结构体

    flags = 0;  # 初始化标志位
    cc = p->cc;  # 从 pcap_t 结构体中获取当前数据包的长度
    if (cc == 0) {  # 如果当前数据包长度为0
        data.buf = (char *)p->buffer + p->offset;  # 设置数据信息的缓冲区指针
        data.maxlen = p->bufsize;  # 设置数据信息的最大长度
        data.len = 0;  # 设置数据信息的实际长度为0
        do {
            """
             * "pcap_breakloop()" 被调用了吗？
             """
            if (p->break_loop) {  # 如果 pcap_breakloop() 被调用
                """
                 * 是的 - 清除指示调用了该函数的标志，并返回 -2 表示被告知跳出循环
                 """
                p->break_loop = 0;  # 清除调用标志
                return (-2);  # 返回 -2 表示跳出循环
            }
            """
             * XXX - 检查 DLPI 原语，如果我们处于原始模式下，HP-UX 上会是 DL_HP_RAWDATA_IND 吗？
             """
            ctl.buf = (char *)ctlbuf;  # 设置控制信息的缓冲区指针
            ctl.maxlen = MAXDLBUF;  # 设置控制信息的最大长度
            ctl.len = 0;  # 设置控制信息的实际长度为0
            if (getmsg(p->fd, &ctl, &data, &flags) < 0) {  # 调用 getmsg() 函数获取消息
                """
                 * 当我们被 ptraced 时不要中断
                 """
                switch (errno) {  # 根据错误码进行处理

                case EINTR:  # 如果是中断错误
                    cc = 0;  # 将数据包长度设置为0
                    continue;  # 继续循环
                case EAGAIN:  # 如果是再试一次错误
                    return (0);  # 返回0
                }
                pcap_fmt_errmsg_for_errno(p->errbuf,  # 格式化错误消息
                    sizeof(p->errbuf), errno, "getmsg");
                return (-1);  # 返回-1表示出错
            }
            cc = data.len;  # 更新数据包长度
        } while (cc == 0);  # 当数据包长度为0时继续循环
        bp = (u_char *)p->buffer + p->offset;  # 设置数据缓冲区指针
    } else
        bp = p->bp;  # 如果数据包长度不为0，设置数据缓冲区指针为 p->bp
    # 调用函数 pcap_process_pkts，处理数据包并返回结果
    return (pcap_process_pkts(p, callback, user, cnt, bp, cc));
}
// 定义一个名为 pcap_inject_dlpi 的静态整型函数，接受 pcap_t 结构体指针、数据缓冲区和大小作为参数
static int
pcap_inject_dlpi(pcap_t *p, const void *buf, int size)
{
#ifdef DL_HP_RAWDLS
    // 如果定义了 DL_HP_RAWDLS 宏，则创建 pcap_dlpi 结构体指针 pd，并指向 pcap_t 结构体的私有数据
    struct pcap_dlpi *pd = p->priv;
#endif
    // 定义一个整型变量 ret
    int ret;

    // 如果定义了 DLIOCRAW 宏
#if defined(DLIOCRAW)
    // 使用 write 函数将数据写入文件描述符 p->fd，返回写入的字节数
    ret = write(p->fd, buf, size);
    // 如果返回值为 -1，表示写入失败，将错误信息格式化到错误缓冲区中，并返回 -1
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
// 如果定义了 DL_HP_RAWDLS 宏
#elif defined(DL_HP_RAWDLS)
    // 如果 pd->send_fd 小于 0，表示输出文件描述符无法打开，将错误信息格式化到错误缓冲区中，并返回 -1
    if (pd->send_fd < 0) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
            "send: Output FD couldn't be opened");
        return (-1);
    }
    // 使用 dlrawdatareq 函数将数据发送到 pd->send_fd，返回发送的字节数
    ret = dlrawdatareq(pd->send_fd, buf, size);
    // 如果返回值为 -1，表示发送失败，将错误信息格式化到错误缓冲区中，并返回 -1
    if (ret == -1) {
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "send");
        return (-1);
    }
    /*
     * putmsg() 返回值为 0 或 -1；它不表示写入了多少字节（可能全部写入或者没有写入）。
     * OpenBSD 的 pcap_inject() 返回写入的字节数，为了 API 兼容性，我们返回被告知要写入的字节数。
     */
    ret = size;
// 如果没有定义原始模式
#else /* no raw mode */
    /*
     * 这是一个痛点，因为你可能需要从数据包中提取地址，并在 DL_UNITDATA_REQ 请求中使用它。这将取决于链路层类型。
     *
     * 我也不知道你需要将描述符绑定到哪个SAP，或者你是否需要单独的“接收”和“发送”FD，也不知道你是否需要为D/I/X以太网和802.3，或者{FDDI，Token Ring}加上802.2和{FDDI，Token Ring}加上802.2加上SNAP需要不同的绑定。
     *
     * 因此，目前我们只返回“无法发送”指示，并留给那些缺少DLIOCRAW和DL_HP_RAWDLS的基于DLPI的系统的人提供实现该系统上数据包传输的代码。如果他们这样做，他们应该将其发送给我们 - 但不应该向我们发送假设以太网的代码；如果代码在非以太网接口上不起作用，它应该检查“p->linktype”，如果不是DLT_EN10MB，则拒绝发送请求。
     */
    pcap_strlcpy(p->errbuf, "send: Not supported on this version of this OS",
        PCAP_ERRBUF_SIZE);
    ret = -1;
#endif /* raw mode */
    // 返回 ret 变量的值
    return (ret);
}

#ifndef DL_IPATM
#define DL_IPATM    0x12    /* ATM Classical IP interface */
#endif

#ifdef HAVE_SOLARIS
/*
 * For SunATM.
 */
#ifndef A_GET_UNITS
#define A_GET_UNITS    (('A'<<8)|118)
#endif /* A_GET_UNITS */
#ifndef A_PROMISCON_REQ
#define A_PROMISCON_REQ    (('A'<<8)|121)
#endif /* A_PROMISCON_REQ */
#endif /* HAVE_SOLARIS */

static void
pcap_cleanup_dlpi(pcap_t *p)
{
#ifdef DL_HP_RAWDLS
    // 获取 pcap_t 结构体中的私有数据
    struct pcap_dlpi *pd = p->priv;

    // 如果 send_fd 大于等于 0，则关闭 send_fd，并将其值设为 -1
    if (pd->send_fd >= 0) {
        close(pd->send_fd);
        pd->send_fd = -1;
    }
#endif
    // 调用 pcap_cleanup_live_common 函数
    pcap_cleanup_live_common(p);
}

static int
open_dlpi_device(const char *name, u_int *ppa, char *errbuf)
{
    int status;
    char dname[100];
    char *cp;
    int fd;
#ifdef HAVE_DEV_DLPI
    u_int unit;
#else
    char dname2[100];
#endif

#ifdef HAVE_DEV_DLPI
    /*
    ** Remove any "/dev/" on the front of the device.
    */
    // 找到设备名中最后一个 '/' 的位置
    cp = strrchr(name, '/');
    // 如果找不到 '/'，则将整个设备名拷贝到 dname 中
    if (cp == NULL)
        pcap_strlcpy(dname, name, sizeof(dname));
    // 否则，将 '/' 后面的部分拷贝到 dname 中
    else
        pcap_strlcpy(dname, cp + 1, sizeof(dname));

    /*
     * Split the device name into a device type name and a unit number;
     * chop off the unit number, so "dname" is just a device type name.
     */
    // 将设备名拆分成设备类型名称和单元号，将单元号从设备名中去掉，使 dname 只包含设备类型名称
    cp = split_dname(dname, &unit, errbuf);
    // 如果 split_dname 返回空指针，表示出错，返回错误码
    if (cp == NULL) {
        /*
         * split_dname() has filled in the error message.
         */
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }
    *cp = '\0';

    /*
     * Use "/dev/dlpi" as the device.
     *
     * XXX - HP's DLPI Programmer's Guide for HP-UX 11.00 says that
     * the "dl_mjr_num" field is for the "major number of interface
     * driver"; that's the major of "/dev/dlpi" on the system on
     * which I tried this, but there may be DLPI devices that
     * use a different driver, in which case we may need to
     * search "/dev" for the appropriate device with that major
     * device number, rather than hardwiring "/dev/dlpi".
     */
    // 将 "/dev/dlpi" 作为设备
    cp = "/dev/dlpi";
    # 如果以读写方式打开文件失败
    if ((fd = open(cp, O_RDWR)) < 0) {
        # 如果错误码是 EPERM 或者 EACCES
        if (errno == EPERM || errno == EACCES) {
            # 设置权限被拒绝的错误状态
            status = PCAP_ERROR_PERM_DENIED;
            # 格式化错误信息，提示可能需要 root 权限
            snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to open %s failed with %s - root privilege may be required",
                cp, (errno == EPERM) ? "EPERM" : "EACCES");
        } else {
            # 设置一般错误状态
            status = PCAP_ERROR;
            # 格式化错误信息，提示打开文件失败
            pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
                errno, "Attempt to open %s failed", cp);
        }
        # 返回状态
        return (status);
    }

    /*
     * 获取该设备的所有 PPAs 表，并在表中搜索指定的设备类型名称和单元编号。
     */
    # 获取设备的 PPA
    status = get_dlpi_ppa(fd, dname, unit, ppa, errbuf);
    # 如果获取失败
    if (status < 0) {
        # 关闭文件
        close(fd);
        # 返回状态
        return (status);
    }
#else
    /*
     * 如果设备名称以“/”开头，则假定它以包含要打开设备的目录的路径名开头；
     * 否则，将设备目录名称和设备名称连接起来。
     */
    if (*name == '/')
        pcap_strlcpy(dname, name, sizeof(dname));
    else
        snprintf(dname, sizeof(dname), "%s/%s", PCAP_DEV_PREFIX,
            name);

    /*
     * 获取单元号，并获取设备类型名称的末尾指针。
     */
    cp = split_dname(dname, ppa, errbuf);
    if (cp == NULL) {
        /*
         * split_dname() 已经填写了错误消息。
         */
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }

    /*
     * 复制设备路径名，然后从设备路径名中删除单元号。
     */
    pcap_strlcpy(dname2, dname, sizeof(dname));
    *cp = '\0';

    /* 尝试不带单元号的设备 */
    }
#endif
    return (fd);
}

static int
pcap_activate_dlpi(pcap_t *p)
{
#ifdef DL_HP_RAWDLS
    struct pcap_dlpi *pd = p->priv;
#endif
    int status = 0;
    int retv;
    u_int ppa;
#ifdef HAVE_SOLARIS
    int isatm = 0;
#endif
    register dl_info_ack_t *infop;
#ifdef HAVE_SYS_BUFMOD_H
    bpf_u_int32 ss;
#ifdef HAVE_SOLARIS
    char release[GET_RELEASE_BUFSIZE];
    bpf_u_int32 osmajor, osminor, osmicro;
#endif
#endif
    bpf_u_int32 buf[MAXDLBUF];

    p->fd = open_dlpi_device(p->opt.device, &ppa, p->errbuf);
    if (p->fd < 0) {
        status = p->fd;
        goto bad;
    }

#ifdef DL_HP_RAWDLS
    /*
     * HP-UX 10.20和11.xx似乎不支持在同一个描述符上发送和接收数据包 - 需要分别使用发送和接收的描述符，绑定到不同的SAP。
     *
     * 如果打开失败，我们只需在"pd->send_fd"中留下-1，并拒绝发送数据包的尝试，就像在pcap-bpf.c中，如果我们无法打开BPF设备进行读写，我们只尝试打开它进行只读操作，如果成功了，就让发送尝试失败。
     */
    pd->send_fd = open("/dev/dlpi", O_RDWR);
#endif

    /*
    ** 如果是 "style 2" 提供者，则附加
    */
    if (dlinforeq(p->fd, p->errbuf) < 0 ||  // 如果获取信息请求失败
        dlinfoack(p->fd, (char *)buf, p->errbuf) < 0) {  // 或者获取信息响应失败
        status = PCAP_ERROR;  // 设置状态为错误
        goto bad;  // 跳转到错误处理
    }
    infop = &(MAKE_DL_PRIMITIVES(buf))->info_ack;  // 获取信息响应
#ifdef HAVE_SOLARIS
    if (infop->dl_mac_type == DL_IPATM)  // 如果是 IPATM 类型
        isatm = 1;  // 设置为 ATM
#endif
    if (infop->dl_provider_style == DL_STYLE2) {  // 如果是 "style 2" 提供者
        retv = dl_doattach(p->fd, ppa, p->errbuf);  // 执行附加操作
        if (retv < 0) {  // 如果附加操作失败
            status = retv;  // 设置状态为返回值
            goto bad;  // 跳转到错误处理
        }
#ifdef DL_HP_RAWDLS
        if (pd->send_fd >= 0) {  // 如果发送文件描述符大于等于 0
            retv = dl_doattach(pd->send_fd, ppa, p->errbuf);  // 执行附加操作
            if (retv < 0) {  // 如果附加操作失败
                status = retv;  // 设置状态为返回值
                goto bad;  // 跳转到错误处理
            }
        }
#endif
    }

    if (p->opt.rfmon) {  // 如果是监控模式
        /*
         * 此设备存在，但我们不支持监控模式
         * 任何支持 DLPI 的平台。
         */
        status = PCAP_ERROR_RFMON_NOTSUP;  // 设置状态为不支持监控模式
        goto bad;  // 跳转到错误处理
    }

#ifdef HAVE_DL_PASSIVE_REQ_T
    /*
     * 启用被动模式以便在聚合链路上进行捕获。
     * 并非所有 Solaris 版本都支持。
     */
    dlpassive(p->fd, p->errbuf);  // 启用被动模式
#endif
    /*
    ** 绑定（如果使用 HP-UX 9 或 HP-UX 10.20 或更高版本，则延迟，如果使用 SINIX，则完全跳过）
    */
#if !defined(HAVE_HPUX9) && !defined(HAVE_HPUX10_20_OR_LATER) && !defined(sinix)
#ifdef _AIX
    /*
    ** AIX。
    ** 根据 IBM 的 AIX 支持线路，dl_sap 值
    ** 对于标准以太网不应小于 0x600（1536）。
    ** 但是，如果我们在 "tr0" 设备上使用 1537，我们似乎会得到 DL_BADADDR - "DLSAP addr in improper
    ** format or invalid" - 错误，这可能意味着 Token Ring 设备。
    ** （也许我们需要在 "/dev/dlpi/en" 上使用 1537，因为该设备用于 D/I/X 以太网，"SAP" 实际上是以太网类型，它拒绝无效的以太网类型。）
    */
    /*
    如果 1537 失败，我们尝试 2，因为 IBM 韩国的 Hyung Sik Yoon 表示在 Token Ring 上这样可以工作
    他说 0 不行；也许那被认为是一个无效的 LLC SAP 值 - 我假设 DLPI 绑定中的 SAP 值是 802.2 LLC 使用的网络类型的 LLC SAP
    */
    if ((dlbindreq(p->fd, 1537, p->errbuf) < 0 &&
         dlbindreq(p->fd, 2, p->errbuf) < 0) ||
         dlbindack(p->fd, (char *)buf, p->errbuf, NULL) < 0) {
        status = PCAP_ERROR;
        goto bad;
    }
#elif defined(DL_HP_RAWDLS)
    /*
    ** HP-UX 10.0x and 10.1x.
    */
    // 如果定义了 DL_HP_RAWDLS，则执行以下代码块，用于处理 HP-UX 10.0x 和 10.1x
    if (dl_dohpuxbind(p->fd, p->errbuf) < 0) {
        // 如果 dl_dohpuxbind 函数返回值小于 0，则设置状态为 PCAP_ERROR，并跳转到 bad 标签处
        status = PCAP_ERROR;
        goto bad;
    }
    if (pd->send_fd >= 0) {
        /*
        ** XXX - if this fails, just close send_fd and
        ** set it to -1, so that you can't send but can
        ** still receive?
        */
        // 如果 pd->send_fd 大于等于 0，则执行以下代码块
        if (dl_dohpuxbind(pd->send_fd, p->errbuf) < 0) {
            // 如果 dl_dohpuxbind 函数返回值小于 0，则设置状态为 PCAP_ERROR，并跳转到 bad 标签处
            status = PCAP_ERROR;
            goto bad;
        }
    }
#else /* neither AIX nor HP-UX */
    /*
    ** Not Sinix, and neither AIX nor HP-UX - Solaris, and any other
    ** OS using DLPI.
    **/
    // 如果既不是 AIX 也不是 HP-UX，则执行以下代码块，用于处理 Solaris 和其他使用 DLPI 的操作系统
    if (dlbindreq(p->fd, 0, p->errbuf) < 0 ||
        dlbindack(p->fd, (char *)buf, p->errbuf, NULL) < 0) {
        // 如果 dlbindreq 函数返回值小于 0，或者 dlbindack 函数返回值小于 0，则设置状态为 PCAP_ERROR，并跳转到 bad 标签处
        status = PCAP_ERROR;
        goto bad;
    }
#endif /* AIX vs. HP-UX vs. other */
#endif /* !HP-UX 9 and !HP-UX 10.20 or later and !SINIX */

    /*
     * Turn a negative snapshot value (invalid), a snapshot value of
     * 0 (unspecified), or a value bigger than the normal maximum
     * value, into the maximum allowed value.
     *
     * If some application really *needs* a bigger snapshot
     * length, we should just increase MAXIMUM_SNAPLEN.
     */
    // 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为允许的最大值
    if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
        p->snapshot = MAXIMUM_SNAPLEN;

#ifdef HAVE_SOLARIS
    if (isatm) {
        /*
        ** Have to turn on some special ATM promiscuous mode
        ** for SunATM.
        ** Do *NOT* turn regular promiscuous mode on; it doesn't
        ** help, and may break things.
        */
        // 如果 isatm 为真，则执行以下代码块，用于为 SunATM 打开特殊的 ATM 混杂模式
        if (strioctl(p->fd, A_PROMISCON_REQ, 0, NULL) < 0) {
            // 如果 strioctl 函数返回值小于 0，则设置状态为 PCAP_ERROR，并根据错误号设置错误消息，然后跳转到 bad 标签处
            status = PCAP_ERROR;
            pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
                errno, "A_PROMISCON_REQ");
            goto bad;
        }
    } else
#endif
    # 如果选项中包含混杂模式，则启用混杂模式（在发送文件描述符上不需要）
    if (p->opt.promisc) {
        # 调用函数启用混杂模式，设置为物理层混杂模式
        retv = dlpromiscon(p, DL_PROMISC_PHYS);
        # 如果返回值小于 0
        if (retv < 0) {
            # 如果返回值为权限被拒绝
            if (retv == PCAP_ERROR_PERM_DENIED)
                # 设置状态为权限被拒绝的混杂模式错误
                status = PCAP_ERROR_PROMISC_PERM_DENIED;
            else
                # 否则设置状态为返回值
                status = retv;
            # 跳转到错误处理标签
            goto bad;
        }

        # 尝试启用多播（你本以为混杂模式就足够了）。（如果使用 HP-UX 或 SINIX，则跳过）（在发送文件描述符上不需要）
#if !defined(__hpux) && !defined(sinix)
    # 如果不是在 HP-UX 或 SINIX 系统下
    retv = dlpromiscon(p, DL_PROMISC_MULTI);
    # 尝试启用多播混杂模式
    if (retv < 0)
        # 如果失败，设置状态为警告
        status = PCAP_WARNING;
#endif
    }
    /*
    ** 尝试启用 SAP 混杂模式（当不在混杂模式下使用 HP-UX 时，当在 Solaris 上不使用 SunATM 时，以及在 SINIX 下从不使用）（在发送 FD 上不需要）
    */
#ifndef sinix
#if defined(__hpux)
    /* HP-UX - 只有在不在混杂模式下时才这样做 */
    if (!p->opt.promisc) {
#elif defined(HAVE_SOLARIS)
    /* Solaris - 在 SunATM 设备上不这样做 */
    if (!isatm) {
#else
    /* 其他所有情况（除了 SINIX）- 总是这样做 */
    {
#endif
        retv = dlpromiscon(p, DL_PROMISC_SAP);
        # 尝试启用 SAP 混杂模式
        if (retv < 0) {
            if (p->opt.promisc) {
                /*
                 * 不是致命的，因为 DL_PROMISC_PHYS 模式有效。
                 *
                 * 但是报告为警告。
                 */
                status = PCAP_WARNING;
            } else {
                /*
                 * 致命的。
                 */
                status = retv;
                goto bad;
            }
        }
    }
#endif /* sinix */

    /*
    ** HP-UX 9 和 HP-UX 10.20 或更高版本，在设置混杂选项后必须绑定。
    */
#if defined(HAVE_HPUX9) || defined(HAVE_HPUX10_20_OR_LATER)
    if (dl_dohpuxbind(p->fd, p->errbuf) < 0) {
        status = PCAP_ERROR;
        goto bad;
    }
    /*
    ** 我们不在发送 FD 上设置混杂模式，但是我们仍然推迟绑定，只是为了将 HP-UX 9/10.20 或更高版本的代码放在一起。
    */
    if (pd->send_fd >= 0) {
        /*
        ** XXX - 如果这失败了，只需关闭 send_fd 并将其设置为 -1，这样您就无法发送但仍然可以接收？
        */
        if (dl_dohpuxbind(pd->send_fd, p->errbuf) < 0) {
            status = PCAP_ERROR;
            goto bad;
        }
    }
#endif

    /*
    ** 确定链路类型
    # 获取SAP长度和地址长度，用于发送数据包时的使用
    if (dlinforeq(p->fd, p->errbuf) < 0 ||  # 如果获取设备信息请求失败
        dlinfoack(p->fd, (char *)buf, p->errbuf) < 0):  # 或者获取设备信息响应失败
        status = PCAP_ERROR  # 设置状态为错误
        goto bad  # 跳转到错误处理部分

    infop = &(MAKE_DL_PRIMITIVES(buf))->info_ack  # 获取设备信息响应的指针
    if (pcap_process_mactype(p, infop->dl_mac_type) != 0):  # 如果处理 MAC 类型失败
        status = PCAP_ERROR  # 设置状态为错误
        goto bad  # 跳转到错误处理部分
#ifdef    DLIOCRAW
    /*
    ** 这是一个非标准的 SunOS hack，用于获取完整的原始链路层头部。
    */
    if (strioctl(p->fd, DLIOCRAW, 0, NULL) < 0) {
        status = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "DLIOCRAW");
        goto bad;
    }
#endif

#ifdef HAVE_SYS_BUFMOD_H
    ss = p->snapshot;

    /*
    ** 在 bufmod(7) 中存在一个 bug。当处理小于 snaplen 大小的消息时，它会从开头而不是结尾剥离数据。
    **
    ** 这个 bug 在 5.3.2 中已修复。此外，还有一个可用的补丁。请询问 bugid 1149065。
    */
#ifdef HAVE_SOLARIS
    get_release(release, sizeof (release), &osmajor, &osminor, &osmicro);
    if (osmajor == 5 && (osminor <= 2 || (osminor == 3 && osmicro < 2)) &&
        getenv("BUFMOD_FIXED") == NULL) {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "警告：SunOS %s 中的 bufmod 有问题；忽略 snaplen。", release);
        ss = 0;
        status = PCAP_WARNING;
    }
#endif

    /* 推送并配置 bufmod。*/
    if (pcap_conf_bufmod(p, ss) != 0) {
        status = PCAP_ERROR;
        goto bad;
    }
#endif

    /*
    ** 作为最后一个操作刷新读取端。
    */
    if (ioctl(p->fd, I_FLUSH, FLUSHR) != 0) {
        status = PCAP_ERROR;
        pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
            errno, "FLUSHR");
        goto bad;
    }

    /* 分配数据缓冲区。*/
    if (pcap_alloc_databuf(p) != 0) {
        status = PCAP_ERROR;
        goto bad;
    }

    /*
     * 成功。
     *
     * "p->fd" 是一个 STREAMS 设备的 FD，因此 "select()" 和 "poll()" 应该在其上工作。
     */
    p->selectable_fd = p->fd;

    p->read_op = pcap_read_dlpi;
    p->inject_op = pcap_inject_dlpi;
    p->setfilter_op = install_bpf_program;    /* 没有内核过滤 */
    p->setdirection_op = NULL;    /* 未实现。*/
    p->set_datalink_op = NULL;    /* 设置数据链路类型操作为NULL，无法更改数据链路类型 */
    p->getnonblock_op = pcap_getnonblock_fd;  /* 获取非阻塞操作为pcap_getnonblock_fd */
    p->setnonblock_op = pcap_setnonblock_fd;  /* 设置非阻塞操作为pcap_setnonblock_fd */
    p->stats_op = pcap_stats_dlpi;  /* 统计操作为pcap_stats_dlpi */
    p->cleanup_op = pcap_cleanup_dlpi;  /* 清理操作为pcap_cleanup_dlpi */

    return (status);  /* 返回状态 */
# 清理并释放 DLPI 对象
pcap_cleanup_dlpi(p);
# 返回状态值
return (status);
}

/*
 * 将设备名称分割成设备类型名称和单元号；
 * 返回指向单元号开头的指针，即设备类型名称的末尾，并将 "*unitp" 设置为单元号。
 *
 * 在错误时返回 NULL，并使用错误消息填充 "ebuf"。
 */
static char *
split_dname(char *device, u_int *unitp, char *ebuf)
{
    char *cp;
    char *eos;
    long unit;

    /*
     * 在设备名称字符串的末尾查找一个数字。
     */
    cp = device + strlen(device) - 1;
    if (*cp < '0' || *cp > '9') {
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "%s missing unit number",
            device);
        return (NULL);
    }

    /* 字符串末尾的数字是单元号 */
    while (cp-1 >= device && *(cp-1) >= '0' && *(cp-1) <= '9')
        cp--;

    errno = 0;
    unit = strtol(cp, &eos, 10);
    if (*eos != '\0') {
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "%s bad unit number", device);
        return (NULL);
    }
    if (errno == ERANGE || unit > INT_MAX) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "%s unit number too large",
            device);
        return (NULL);
    }
    if (unit < 0) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "%s unit number is negative",
            device);
        return (NULL);
    }
    *unitp = (u_int)unit;
    return (cp);
}

static int
dl_doattach(int fd, int ppa, char *ebuf)
{
    dl_attach_req_t    req;
    bpf_u_int32 buf[MAXDLBUF];
    int err;

    req.dl_primitive = DL_ATTACH_REQ;
    req.dl_ppa = ppa;
    if (send_request(fd, (char *)&req, sizeof(req), "attach", ebuf) < 0)
        return (PCAP_ERROR);

    err = dlokack(fd, "attach", (char *)buf, ebuf, NULL);
    if (err < 0)
        return (err);
    return (0);
}

#ifdef DL_HP_RAWDLS
static int
dl_dohpuxbind(int fd, char *ebuf)
{
    int hpsap;
    int uerror;
    bpf_u_int32 buf[MAXDLBUF];
    """
    XXX - 我们从22开始是因为以前只使用22，但那只是因为在HP的一些示例代码中使用了这个值。
    我们应该用什么值开始呢？考虑到我们在输入FD上启用了SAP的宽容性，这重要吗？
    """
    # 初始化 hpsap 变量为 22
    hpsap = 22
    # 无限循环
    for (;;):
        # 如果 dlbindreq 函数返回值小于 0，则返回 -1
        if (dlbindreq(fd, hpsap, ebuf) < 0)
            return (-1)
        # 如果 dlbindack 函数返回值大于等于 0，则跳出循环
        if (dlbindack(fd, (char *)buf, ebuf, &uerror) >= 0)
            break
        """
        对于除了 UNIX EBUSY 之外的任何错误，放弃。
        dlbindack() 已经填充了 ebuf 来表示这个错误。
        """
        if (uerror != EBUSY):
            return (-1)
        """
        对于 EBUSY，尝试下一个 SAP 值；这意味着其他人正在使用该 SAP。
        清除 ebuf，以便应用程序不将“设备忙”错误报告为警告。
        """
        *ebuf = '\0'
        hpsap++
        # 如果 hpsap 大于 100，则返回错误
        if (hpsap > 100):
            pcap_strlcpy(ebuf,
                "All SAPs from 22 through 100 are in use",
                PCAP_ERRBUF_SIZE)
            return (-1)
    # 返回 0
    return (0)
# 结束条件判断
}
#endif

# 定义宏，将参数转换为字符串
#define STRINGIFY(n)    #n

# 设置混杂模式
static int
dlpromiscon(pcap_t *p, bpf_u_int32 level)
{
    # DLPI 接口的混杂模式请求
    dl_promiscon_req_t req;
    # 缓冲区
    bpf_u_int32 buf[MAXDLBUF];
    # 错误码
    int err;
    int uerror;

    # 设置混杂模式请求
    req.dl_primitive = DL_PROMISCON_REQ;
    req.dl_level = level;
    # 发送请求
    if (send_request(p->fd, (char *)&req, sizeof(req), "promiscon",
        p->errbuf) < 0)
        return (PCAP_ERROR);
    # 接收响应
    err = dlokack(p->fd, "promiscon" STRINGIFY(level), (char *)buf,
        p->errbuf, &uerror);
    if (err < 0) {
        # 处理错误
        if (err == PCAP_ERROR_PERM_DENIED) {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
                "Attempt to set promiscuous mode failed with %s - root privilege may be required",
                (uerror == EPERM) ? "EPERM" : "EACCES");
            err = PCAP_ERROR_PROMISC_PERM_DENIED;
        }
        return (err);
    }
    return (0);
}

# 检查是否为 DLPI 接口
static int
is_dlpi_interface(const char *name)
{
    # 文件描述符
    int fd;
    # PPA
    u_int ppa;
    # 错误缓冲区
    char errbuf[PCAP_ERRBUF_SIZE];

    # 打开 DLPI 设备
    fd = open_dlpi_device(name, &ppa, errbuf);
    # 如果文件描述符小于 0
    if (fd < 0) {
        /*
         * 错误 - 是 PCAP_ERROR_NO_SUCH_DEVICE 吗？
         */
        if (fd == PCAP_ERROR_NO_SUCH_DEVICE) {
            /*
             * 是的，所以我们无法打开它，因为它不是 DLPI 接口。
             */
            return (0);
        }
        /*
         * 不是，所以，在所有这种类型接口的单个 DLPI 设备的情况下（“style 2”提供程序？），
         * 我们不知道它是否是 DLPI 接口，因为我们没有尝试附加。
         * 假设它是 DLPI 设备，这样用户至少可以尝试打开它并报告错误
         * （这个错误可能是“您没有权限打开该 DLPI 设备”；报告这些接口意味着用户会问“为什么我尝试捕获时会收到权限错误”，而不是“为什么我看不到任何接口”，这样可以更清楚地了解潜在问题）。
         */
        return (1);
    }

    /*
     * 成功。
     */
    close(fd);
    return (1);
}
// 获取网络接口的标志位
static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
    /*
     * 除了将环回设备标记为“连接/断开状态不适用”外，我们无法做任何其他事情。
     *
     * XXX - 在 Solaris 上，我们能否像 dladm 命令那样，从 kstat 获取连接/断开指示？
     * （请注意，您还可以从 kstat 获取链接速度，可能还有其他信息。）
     */
    if (*flags & PCAP_IF_LOOPBACK) {
        /*
         * 环回设备不是无线设备，“连接”/“断开”对它们不适用。
         */
        *flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
        return (0);
    }
    return (0);
}

int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
#ifdef HAVE_SOLARIS
    int fd;
    union {
        u_int nunits;
        char pad[516];    /* XXX - must be at least 513; is 516
                   in "atmgetunits" */
    } buf;
    char baname[2+1+1];
    u_int i;
#endif

    /*
     * 首先获取常规接口的列表。
     */
    if (pcap_findalldevs_interfaces(devlistp, errbuf, is_dlpi_interface,
        get_if_flags) == -1)
        return (-1);    /* 失败 */

#ifdef HAVE_SOLARIS
    /*
     * 我们可能需要特殊的魔术来获取 ATM 设备。
     */
    if ((fd = open("/dev/ba", O_RDWR)) < 0) {
        /*
         * 我们无法打开“ba”设备。
         * 目前，就放弃吧；也许如果问题既不是“该设备不存在”错误（ENOENT、ENXIO 等），
         * 也不是“您无权这样做”错误（EPERM、EACCES），我们应该返回错误。
         */
        return (0);
    }

    if (strioctl(fd, A_GET_UNITS, sizeof(buf), (char *)&buf) < 0) {
        pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
            errno, "A_GET_UNITS");
        return (-1);
    }
    # 遍历 buf 结构体中的单元数量
    for (i = 0; i < buf.nunits; i++) {
        # 格式化生成设备名称，例如 ba0、ba1 等
        snprintf(baname, sizeof baname, "ba%u", i);
        '''
         * XXX - 是否有"up"和"running"的概念？
         * 是否有一种方法可以确定接口是否插入到网络中？
         '''
        # 如果添加设备失败，则返回 -1
        if (add_dev(devlistp, baname, 0, NULL, errbuf) == NULL)
            return (-1);
    }
#endif
# 返回 0，表示成功
    return (0);
}

static int
send_request(int fd, char *ptr, int len, char *what, char *ebuf)
{
    struct    strbuf    ctl;  # 定义 strbuf 结构体变量 ctl
    int    flags;  # 定义整型变量 flags

    ctl.maxlen = 0;  # 设置 ctl 结构体的最大长度为 0
    ctl.len = len;  # 设置 ctl 结构体的长度为 len
    ctl.buf = ptr;  # 设置 ctl 结构体的缓冲区为 ptr

    flags = 0;  # 设置 flags 为 0
    if (putmsg(fd, &ctl, (struct strbuf *) NULL, flags) < 0) {  # 调用 putmsg 函数发送消息
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,  # 格式化错误消息
            errno, "send_request: putmsg \"%s\"", what);
        return (-1);  # 返回 -1，表示发送请求失败
    }
    return (0);  # 返回 0，表示发送请求成功
}

static int
recv_ack(int fd, int size, const char *what, char *bufp, char *ebuf, int *uerror)
{
    union    DL_primitives    *dlp;  # 定义 DL_primitives 联合体指针变量 dlp
    struct    strbuf    ctl;  # 定义 strbuf 结构体变量 ctl
    int    flags;  # 定义整型变量 flags
    char    errmsgbuf[PCAP_ERRBUF_SIZE];  # 定义字符数组 errmsgbuf
    char    dlprimbuf[64];  # 定义字符数组 dlprimbuf

    """
     * Clear out "*uerror", so it's only set for DL_ERROR_ACK/DL_SYSERR,
     * making that the only place where EBUSY is treated specially.
     """
    if (uerror != NULL)  # 如果 uerror 不为空
        *uerror = 0;  # 将 uerror 设置为 0

    ctl.maxlen = MAXDLBUF;  # 设置 ctl 结构体的最大长度为 MAXDLBUF
    ctl.len = 0;  # 设置 ctl 结构体的长度为 0
    ctl.buf = bufp;  # 设置 ctl 结构体的缓冲区为 bufp

    flags = 0;  # 设置 flags 为 0
    if (getmsg(fd, &ctl, (struct strbuf*)NULL, &flags) < 0) {  # 调用 getmsg 函数接收消息
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,  # 格式化错误消息
            errno, "recv_ack: %s getmsg", what);
        return (PCAP_ERROR);  # 返回 PCAP_ERROR，表示接收确认失败
    }

    dlp = MAKE_DL_PRIMITIVES(ctl.buf);  # 使用 ctl.buf 创建 DL_primitives 联合体指针
    switch (dlp->dl_primitive) {  # 根据 dl_primitive 的值进行判断

    case DL_INFO_ACK:  # 如果 dl_primitive 为 DL_INFO_ACK
    case DL_BIND_ACK:  # 如果 dl_primitive 为 DL_BIND_ACK
    case DL_OK_ACK:  # 如果 dl_primitive 为 DL_OK_ACK
#ifdef DL_HP_PPA_ACK
    case DL_HP_PPA_ACK:  # 如果 dl_primitive 为 DL_HP_PPA_ACK
#endif
        /* These are OK */
        break;  # 跳出 switch 语句
    # 如果收到的数据链路层控制信息是错误应答
    case DL_ERROR_ACK:
        # 根据错误码进行处理
        switch (dlp->error_ack.dl_errno) {

        # 如果是系统错误
        case DL_SYSERR:
            # 如果用户错误指针不为空，将 UNIX 错误码存入其中
            if (uerror != NULL)
                *uerror = dlp->error_ack.dl_unix_errno;
            # 格式化错误消息，将 UNIX 错误码转换为错误消息
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                dlp->error_ack.dl_unix_errno,
                "recv_ack: %s: UNIX error", what);
            # 如果 UNIX 错误码是 EPERM 或 EACCES，则返回权限被拒绝错误
            if (dlp->error_ack.dl_unix_errno == EPERM ||
                dlp->error_ack.dl_unix_errno == EACCES)
                return (PCAP_ERROR_PERM_DENIED);
            break;

        # 如果是其他类型的错误
        default:
            # 格式化错误消息，将错误码转换为错误消息
            snprintf(ebuf, PCAP_ERRBUF_SIZE,
                "recv_ack: %s: %s", what,
                dlstrerror(errmsgbuf, sizeof (errmsgbuf), dlp->error_ack.dl_errno));
            # 如果错误码是 DL_BADPPA，则返回没有这样的设备错误
            if (dlp->error_ack.dl_errno == DL_BADPPA)
                return (PCAP_ERROR_NO_SUCH_DEVICE);
            # 如果错误码是 DL_ACCESS，则返回权限被拒绝错误
            else if (dlp->error_ack.dl_errno == DL_ACCESS)
                return (PCAP_ERROR_PERM_DENIED);
            break;
        }
        # 返回捕获错误
        return (PCAP_ERROR);

    # 如果收到的数据链路层控制信息不是错误应答
    default:
        # 格式化错误消息，表示收到了意外的原始应答
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "recv_ack: %s: Unexpected primitive ack %s",
            what, dlprim(dlprimbuf, sizeof (dlprimbuf), dlp->dl_primitive));
        # 返回捕获错误
        return (PCAP_ERROR);
    }

    # 如果控制信息的长度小于指定的大小
    if (ctl.len < size) {
        # 格式化错误消息，表示应答太小
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "recv_ack: %s: Ack too small (%d < %d)",
            what, ctl.len, size);
        # 返回捕获错误
        return (PCAP_ERROR);
    }
    # 返回控制信息的长度
    return (ctl.len);
    # 根据 DL 错误码返回对应的错误信息字符串
    static char * dlstrerror(char *errbuf, size_t errbufsize, bpf_u_int32 dl_errno) {
        # 根据 DL 错误码进行不同的处理
        switch (dl_errno) {
            # 如果错误码为 DL_ACCESS
            case DL_ACCESS:
                # 返回对应的错误信息字符串
                return ("Improper permissions for request");
            # 如果错误码为 DL_BADADDR
            case DL_BADADDR:
                # 返回对应的错误信息字符串
                return ("DLSAP addr in improper format or invalid");
            # 如果错误码为 DL_BADCORR
            case DL_BADCORR:
                # 返回对应的错误信息字符串
                return ("Seq number not from outstand DL_CONN_IND");
            # 如果错误码为 DL_BADDATA
            case DL_BADDATA:
                # 返回对应的错误信息字符串
                return ("User data exceeded provider limit");
            # 如果错误码为 DL_BADPPA
            case DL_BADPPA:
                # 如果定义了 HAVE_DEV_DLPI
                # 返回对应的错误信息字符串
                # 否则返回对应的错误信息字符串
                # 根据不同的情况返回不同的错误信息字符串
                # 根据不同的情况返回不同的错误信息字符串
                return ("Specified PPA was invalid");
            # 如果错误码为 DL_BADPRIM
            case DL_BADPRIM:
                # 返回对应的错误信息字符串
                return ("Primitive received not known by provider");
            # 如果错误码为 DL_BADQOSPARAM
            case DL_BADQOSPARAM:
                # 返回对应的错误信息字符串
                return ("QOS parameters contained invalid values");
            # 如果错误码为 DL_BADQOSTYPE
            case DL_BADQOSTYPE:
                # 返回对应的错误信息字符串
                return ("QOS structure type is unknown/unsupported");
            # 如果错误码为 DL_BADSAP
            case DL_BADSAP:
                # 返回对应的错误信息字符串
                return ("Bad LSAP selector");
            # 如果错误码为 DL_BADTOKEN
            case DL_BADTOKEN:
                # 返回对应的错误信息字符串
                return ("Token used not an active stream");
            # 如果错误码为 DL_BOUND
            case DL_BOUND:
                # 返回对应的错误信息字符串
                return ("Attempted second bind with dl_max_conind");
            # 如果错误码为 DL_INITFAILED
            case DL_INITFAILED:
                # 返回对应的错误信息字符串
                return ("Physical link initialization failed");
            # 如果错误码为 DL_NOADDR
            case DL_NOADDR:
                # 返回对应的错误信息字符串
                return ("Provider couldn't allocate alternate address");
            # 如果错误码为 DL_NOTINIT
            case DL_NOTINIT:
                # 返回对应的错误信息字符串
                return ("Physical link not initialized");
            # 如果错误码为 DL_OUTSTATE
            case DL_OUTSTATE:
                # 返回对应的错误信息字符串
                return ("Primitive issued in improper state");
            # 如果错误码为 DL_SYSERR
            case DL_SYSERR:
                # 返回对应的错误信息字符串
                return ("UNIX system error occurred");
            # 如果错误码为 DL_UNSUPPORTED
            case DL_UNSUPPORTED:
                # 返回对应的错误信息字符串
                return ("Requested service not supplied by provider");
            # 如果错误码为 DL_UNDELIVERABLE
            case DL_UNDELIVERABLE:
                # 返回对应的错误信息字符串
                return ("Previous data unit could not be delivered");
            # 如果错误码为 DL_NOTSUPPORTED
            case DL_NOTSUPPORTED:
                # 返回对应的错误信息字符串
                return ("Primitive is known but not supported");
        }
    }
    # 如果错误代码为 DL_TOOMANY，则返回"Limit exceeded"
    case DL_TOOMANY:
        return ("Limit exceeded");

    # 如果错误代码为 DL_NOTENAB，则返回"Promiscuous mode not enabled"
    case DL_NOTENAB:
        return ("Promiscuous mode not enabled");

    # 如果错误代码为 DL_BUSY，则返回"Other streams for PPA in post-attached"
    case DL_BUSY:
        return ("Other streams for PPA in post-attached");

    # 如果错误代码为 DL_NOAUTO，则返回"Automatic handling XID&TEST not supported"
    case DL_NOAUTO:
        return ("Automatic handling XID&TEST not supported");

    # 如果错误代码为 DL_NOXIDAUTO，则返回"Automatic handling of XID not supported"
    case DL_NOXIDAUTO:
        return ("Automatic handling of XID not supported");

    # 如果错误代码为 DL_NOTESTAUTO，则返回"Automatic handling of TEST not supported"
    case DL_NOTESTAUTO:
        return ("Automatic handling of TEST not supported");

    # 如果错误代码为 DL_XIDAUTO，则返回"Automatic handling of XID response"
    case DL_XIDAUTO:
        return ("Automatic handling of XID response");

    # 如果错误代码为 DL_TESTAUTO，则返回"Automatic handling of TEST response"
    case DL_TESTAUTO:
        return ("Automatic handling of TEST response");

    # 如果错误代码为 DL_PENDING，则返回"Pending outstanding connect indications"
    case DL_PENDING:
        return ("Pending outstanding connect indications");

    # 如果错误代码为其他值，则返回错误代码的十六进制表示
    default:
        snprintf(errbuf, errbufsize, "Error %02x", dl_errno);
        return (errbuf);
    }
# 定义一个静态函数，用于将原始数据链路层的原语转换为字符串
static char *
dlprim(char *primbuf, size_t primbufsize, bpf_u_int32 prim)
{
    # 根据原语类型进行不同的处理
    switch (prim) {

    case DL_INFO_REQ:
        return ("DL_INFO_REQ");

    case DL_INFO_ACK:
        return ("DL_INFO_ACK");

    case DL_ATTACH_REQ:
        return ("DL_ATTACH_REQ");

    case DL_DETACH_REQ:
        return ("DL_DETACH_REQ");

    case DL_BIND_REQ:
        return ("DL_BIND_REQ");

    case DL_BIND_ACK:
        return ("DL_BIND_ACK");

    case DL_UNBIND_REQ:
        return ("DL_UNBIND_REQ");

    case DL_OK_ACK:
        return ("DL_OK_ACK");

    case DL_ERROR_ACK:
        return ("DL_ERROR_ACK");

    case DL_SUBS_BIND_REQ:
        return ("DL_SUBS_BIND_REQ");

    case DL_SUBS_BIND_ACK:
        return ("DL_SUBS_BIND_ACK");

    case DL_UNITDATA_REQ:
        return ("DL_UNITDATA_REQ");

    case DL_UNITDATA_IND:
        return ("DL_UNITDATA_IND");

    case DL_UDERROR_IND:
        return ("DL_UDERROR_IND");

    case DL_UDQOS_REQ:
        return ("DL_UDQOS_REQ");

    case DL_CONNECT_REQ:
        return ("DL_CONNECT_REQ");

    case DL_CONNECT_IND:
        return ("DL_CONNECT_IND");

    case DL_CONNECT_RES:
        return ("DL_CONNECT_RES");

    case DL_CONNECT_CON:
        return ("DL_CONNECT_CON");

    case DL_TOKEN_REQ:
        return ("DL_TOKEN_REQ");

    case DL_TOKEN_ACK:
        return ("DL_TOKEN_ACK");

    case DL_DISCONNECT_REQ:
        return ("DL_DISCONNECT_REQ");

    case DL_DISCONNECT_IND:
        return ("DL_DISCONNECT_IND");

    case DL_RESET_REQ:
        return ("DL_RESET_REQ");

    case DL_RESET_IND:
        return ("DL_RESET_IND");

    case DL_RESET_RES:
        return ("DL_RESET_RES");

    case DL_RESET_CON:
        return ("DL_RESET_CON");

    default:
        # 如果是未知的原语类型，将其转换为十六进制字符串
        snprintf(primbuf, primbufsize, "unknown primitive 0x%x",
            prim);
        return (primbuf);
    }
}

static int
dlbindreq(int fd, bpf_u_int32 sap, char *ebuf)
{
    # 定义一个数据链路层绑定请求结构
    dl_bind_req_t    req;
    # 将请求结构清零
    memset((char *)&req, 0, sizeof(req));
    # 设置请求结构的原语类型为绑定请求
    req.dl_primitive = DL_BIND_REQ;
    # 如果这两个都没有被定义，会发生什么？
#if defined(DL_HP_RAWDLS)
    # 如果定义了 DL_HP_RAWDLS，则设置最大连接指数为1，服务模式为 DL_HP_RAWDLS
    req.dl_max_conind = 1;            /* XXX magic number */
    req.dl_service_mode = DL_HP_RAWDLS;
#elif defined(DL_CLDLS)
    # 如果定义了 DL_CLDLS，则设置服务模式为 DL_CLDLS
    req.dl_service_mode = DL_CLDLS;
#endif
    # 设置请求的 SAP
    req.dl_sap = sap;

    # 发送请求并返回结果
    return (send_request(fd, (char *)&req, sizeof(req), "bind", ebuf));
}

static int
dlbindack(int fd, char *bufp, char *ebuf, int *uerror)
{
    # 接收绑定确认并返回结果
    return (recv_ack(fd, DL_BIND_ACK_SIZE, "bind", bufp, ebuf, uerror));
}

static int
dlokack(int fd, const char *what, char *bufp, char *ebuf, int *uerror)
{
    # 接收 OK 确认并返回结果
    return (recv_ack(fd, DL_OK_ACK_SIZE, what, bufp, ebuf, uerror));
}

static int
dlinforeq(int fd, char *ebuf)
{
    # 设置信息请求的 DLPI 结构体
    dl_info_req_t req;
    req.dl_primitive = DL_INFO_REQ;

    # 发送信息请求并返回结果
    return (send_request(fd, (char *)&req, sizeof(req), "info", ebuf));
}

static int
dlinfoack(int fd, char *bufp, char *ebuf)
{
    # 接收信息确认并返回结果
    return (recv_ack(fd, DL_INFO_ACK_SIZE, "info", bufp, ebuf, NULL));
}

#ifdef HAVE_DL_PASSIVE_REQ_T
/*
 * Enable DLPI passive mode. We do not care if this request fails, as this
 * indicates the underlying DLPI device does not support link aggregation.
 */
static void
dlpassive(int fd, char *ebuf)
{
    # 设置被动模式请求的 DLPI 结构体
    dl_passive_req_t req;
    bpf_u_int32 buf[MAXDLBUF];

    req.dl_primitive = DL_PASSIVE_REQ;

    # 如果发送请求成功，则接收 OK 确认
    if (send_request(fd, (char *)&req, sizeof(req), "dlpassive", ebuf) == 0)
        (void) dlokack(fd, "dlpassive", (char *)buf, ebuf, NULL);
}
#endif

#ifdef DL_HP_RAWDLS
/*
 * There's an ack *if* there's an error.
 */
static int
dlrawdatareq(int fd, const u_char *datap, int datalen)
{
    # 设置控制和数据缓冲区
    struct strbuf ctl, data;
    long buf[MAXDLBUF];    /* XXX - char? */
    union DL_primitives *dlp;
    int dlen;

    dlp = MAKE_DL_PRIMITIVES(buf);

    # 设置原始数据请求的 DLPI 原语和长度
    dlp->dl_primitive = DL_HP_RAWDATA_REQ;
    dlen = DL_HP_RAWDATA_REQ_SIZE;
    /*
     * HP的文档似乎没有显示我们提供消息控制部分指向的任何地址。
     * 我认为这就是原始模式的意思 - 你只发送原始数据包，不需要指定发送到哪里，因为目的地地址是隐含的。
     */
    // 设置控制消息的最大长度为0
    ctl.maxlen = 0;
    // 设置控制消息的长度为dlen
    ctl.len = dlen;
    // 将buf指向的数据赋值给控制消息的缓冲区
    ctl.buf = (void *)buf;

    // 设置数据消息的最大长度为0
    data.maxlen = 0;
    // 设置数据消息的长度为datalen
    data.len = datalen;
    // 将datap指向的数据赋值给数据消息的缓冲区
    data.buf = (void *)datap;

    // 调用putmsg函数发送消息
    return (putmsg(fd, &ctl, &data, 0));
#ifdef DL_HP_PPA_REQ
/*
 * Under HP-UX 10 and HP-UX 11, we can ask for the ppa
 */
// 在 HP-UX 10 和 HP-UX 11 下，我们可以请求 ppa

/*
 * Determine ppa number that specifies ifname.
 *
 * If the "dl_hp_ppa_info_t" doesn't have a "dl_module_id_1" member,
 * the code that's used here is the old code for HP-UX 10.x.
 *
 * However, HP-UX 10.20, at least, appears to have such a member
 * in its "dl_hp_ppa_info_t" structure, so the new code is used.
 * The new code didn't work on an old 10.20 system on which Rick
 * Jones of HP tried it, but with later patches installed, it
 * worked - it appears that the older system had those members but
 * didn't put anything in them, so, if the search by name fails, we
 * do the old search.
 *
 * Rick suggests that making sure your system is "up on the latest
 * lancommon/DLPI/driver patches" is probably a good idea; it'd fix
 * that problem, as well as allowing libpcap to see packets sent
 * from the system on which the libpcap application is being run.
 * (On 10.20, in addition to getting the latest patches, you need
 * to turn the kernel "lanc_outbound_promisc_flag" flag on with ADB;
 * a posting to "comp.sys.hp.hpux" at
 *
 *    http://www.deja.com/[ST_rn=ps]/getdoc.xp?AN=558092266
 *
 * says that, to see the machine's outgoing traffic, you'd need to
 * apply the right patches to your system, and also set that variable
 * with:
 */
// 确定指定 ifname 的 ppa 号码。
//
// 如果 "dl_hp_ppa_info_t" 没有 "dl_module_id_1" 成员，
// 这里使用的代码是 HP-UX 10.x 的旧代码。
//
// 但是，至少 HP-UX 10.20 似乎在其 "dl_hp_ppa_info_t" 结构中有这样的成员，
// 因此使用新代码。新代码在 Rick Jones of HP 尝试的旧 10.20 系统上无法工作，
// 但安装了后续补丁后，它可以工作 - 看起来旧系统有这些成员，但没有在其中放任何内容，
// 因此，如果按名称搜索失败，我们会执行旧搜索。
//
// Rick 建议确保您的系统 "更新到最新的 lancommon/DLPI/driver 补丁" 可能是个好主意；
// 它会解决这个问题，同时还允许 libpcap 看到从运行 libpcap 应用程序的系统发送的数据包。
// （在 10.20 上，除了获取最新的补丁，您还需要使用 ADB 打开内核 "lanc_outbound_promisc_flag" 标志；
// 在 "comp.sys.hp.hpux" 上的一篇帖子
//
//    http://www.deja.com/[ST_rn=ps]/getdoc.xp?AN=558092266
//
// 说，要看到机器的出站流量，您需要将正确的补丁应用到您的系统，并使用以下命令设置该变量：
# 使用 adb 命令读取指定路径下的内核内存
echo 'lanc_outbound_promisc_flag/W1' | /usr/bin/adb -w /stand/vmunix /dev/kmem

# 定义一个静态函数，用于获取 DLPI 的 PPA 信息
# 参数包括文件描述符、设备名称、单元号、PPA、错误缓冲区
static int
get_dlpi_ppa(register int fd, register const char *device, register u_int unit,
    u_int *ppa, register char *ebuf)
{
    register dl_hp_ppa_ack_t *ap;  # 定义 DLPI PPA 应答指针
    register dl_hp_ppa_info_t *ipstart, *ip;  # 定义 DLPI PPA 信息指针
    register u_int i;  # 定义无符号整型变量 i
    char dname[100];  # 定义长度为 100 的字符数组
    register u_long majdev;  # 定义主设备号
    struct stat statbuf;  # 定义 stat 结构体
    dl_hp_ppa_req_t    req;  # 定义 DLPI PPA 请求结构体
    char buf[MAXDLBUF];  # 定义最大 DL 缓冲区
    char *ppa_data_buf;  # 定义 PPA 数据缓冲区
    dl_hp_ppa_ack_t    *dlp;  # 定义 DLPI PPA 应答指针
    struct strbuf ctl;  # 定义 strbuf 结构体
    int flags;  # 定义整型变量 flags

    # 将 req 结构体清零
    memset((char *)&req, 0, sizeof(req));
    req.dl_primitive = DL_HP_PPA_REQ;  # 设置 DLPI PPA 请求类型

    # 将 buf 缓冲区清零
    memset((char *)buf, 0, sizeof(buf));
    # 发送请求并获取返回值
    if (send_request(fd, (char *)&req, sizeof(req), "hpppa", ebuf) < 0)
        return (PCAP_ERROR);  # 如果发送请求失败，则返回错误码

    ctl.maxlen = DL_HP_PPA_ACK_SIZE;  # 设置控制缓冲区最大长度
    ctl.len = 0;  # 设置控制缓冲区长度为 0
    ctl.buf = (char *)buf;  # 设置控制缓冲区内容为 buf

    flags = 0;  # 设置 flags 为 0
    """
    DLPI 可能会返回一个大块数据作为 DL_HP_PPA_REQ 的应答。
    正常的 recv_ack 会因为将 maxlen 设置为 MAXDLBUF (8192) 而失败，
    这对于 DL_HP_PPA_REQ 来说是不够大的。
    这会导致安装了 HP-APA 的系统上 libpcap 应用程序失败。
    为了找出返回数据的实际大小，我们首先调用 getmsg 来获取小头部，
    并在头部查看实际数据长度，然后发出另一个 getmsg 来获取实际的 PPA 数据。
    """
    # 首先获取头部
    if (getmsg(fd, &ctl, (struct strbuf *)NULL, &flags) < 0) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "get_dlpi_ppa: hpppa getmsg");
        return (PCAP_ERROR);  # 如果获取头部失败，则返回错误码
    }
    if (ctl.len == -1) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "get_dlpi_ppa: hpppa getmsg: control buffer has no data");
        return (PCAP_ERROR);  # 如果控制缓冲区没有数据，则返回错误码
    }

    dlp = (dl_hp_ppa_ack_t *)ctl.buf;  # 将控制缓冲区内容转换为 DLPI PPA 应答指针
}
    // 如果收到的数据链路层原语不是 DL_HP_PPA_ACK，则返回错误信息
    if (dlp->dl_primitive != DL_HP_PPA_ACK) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "get_dlpi_ppa: hpppa unexpected primitive ack 0x%x",
            (bpf_u_int32)dlp->dl_primitive);
        return (PCAP_ERROR);
    }

    // 如果控制消息的长度小于 DL_HP_PPA_ACK_SIZE，则返回错误信息
    if ((size_t)ctl.len < DL_HP_PPA_ACK_SIZE) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "get_dlpi_ppa: hpppa ack too small (%d < %lu)",
             ctl.len, (unsigned long)DL_HP_PPA_ACK_SIZE);
        return (PCAP_ERROR);
    }

    /* 分配缓冲区 */
    if ((ppa_data_buf = (char *)malloc(dlp->dl_length)) == NULL) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "get_dlpi_ppa: hpppa malloc");
        return (PCAP_ERROR);
    }
    ctl.maxlen = dlp->dl_length;
    ctl.len = 0;
    ctl.buf = (char *)ppa_data_buf;
    /* 获取数据 */
    if (getmsg(fd, &ctl, (struct strbuf *)NULL, &flags) < 0) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "get_dlpi_ppa: hpppa getmsg");
        free(ppa_data_buf);
        return (PCAP_ERROR);
    }
    if (ctl.len == -1) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "get_dlpi_ppa: hpppa getmsg: control buffer has no data");
        return (PCAP_ERROR);
    }
    if ((u_int)ctl.len < dlp->dl_length) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "get_dlpi_ppa: hpppa ack too small (%d < %lu)",
            ctl.len, (unsigned long)dlp->dl_length);
        free(ppa_data_buf);
        return (PCAP_ERROR);
    }

    ap = (dl_hp_ppa_ack_t *)buf;
    ipstart = (dl_hp_ppa_info_t *)ppa_data_buf;
    ip = ipstart;
#ifdef HAVE_DL_HP_PPA_INFO_T_DL_MODULE_ID_1
    /*
     * 如果定义了 HAVE_DL_HP_PPA_INFO_T_DL_MODULE_ID_1，则执行以下代码块
     * 结构体 "dl_hp_ppa_info_t" 有一个 "dl_module_id_1" 成员，理论上应该包含设备名称中单位编号之前的部分
     * 也应该有一个 "dl_module_id_2" 成员，可能包含备用名称（例如，我认为以太网设备既有 "lan"，表示 "lanN"，也有 "snap"，表示 "snapN"，前者用于以太网数据包，后者用于 802.3/802.2 数据包）
     *
     * 搜索具有指定名称和实例编号的设备
     */
    for (i = 0; i < ap->dl_count; i++) {
        if ((strcmp((const char *)ip->dl_module_id_1, device) == 0 ||
             strcmp((const char *)ip->dl_module_id_2, device) == 0) &&
            ip->dl_instance_num == unit)
            break;

        ip = (dl_hp_ppa_info_t *)((u_char *)ipstart + ip->dl_next_offset);
    }
#else
    /*
     * 如果没有该成员，则搜索是不可能的；使其看起来好像搜索失败了。
     */
    i = ap->dl_count;
#endif
    if (i == ap->dl_count) {
        /*
         * 如果 i 等于 ap 指针所指向结构体的 dl_count 字段的值，说明没有找到设备或者无法通过名称找到设备。
         *
         * 在 HP-UX 10.20 中，"dl_hp_ppa_info_t" 结构体中有 "dl_module_id_1" 和 "dl_module_id_2" 字段，
         * 但是除非系统处于相当新的补丁级别，否则似乎不会填充这些字段。
         *
         * 较旧的 HP-UX 10.x 系统可能根本没有这些字段。
         *
         * 因此，我们将搜索具有名称 "/dev/<dev><unit>" 的设备的主设备号的条目，如果存在这样的设备，就像旧代码一样。
         */
        snprintf(dname, sizeof(dname), "/dev/%s%u", device, unit);
        if (stat(dname, &statbuf) < 0) {
            pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
                errno, "stat: %s", dname);
            return (PCAP_ERROR);
        }
        majdev = major(statbuf.st_rdev);

        ip = ipstart;

        for (i = 0; i < ap->dl_count; i++) {
            if (ip->dl_mjr_num == majdev &&
                ip->dl_instance_num == unit)
                break;

            ip = (dl_hp_ppa_info_t *)((u_char *)ipstart + ip->dl_next_offset);
        }
    }
    if (i == ap->dl_count) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "can't find /dev/dlpi PPA for %s%u", device, unit);
        return (PCAP_ERROR_NO_SUCH_DEVICE);
    }
    if (ip->dl_hdw_state == HDW_DEAD) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "%s%d: hardware state: DOWN\n", device, unit);
        free(ppa_data_buf);
        return (PCAP_ERROR);
    }
    *ppa = ip->dl_ppa;
    free(ppa_data_buf);
    return (0);
}
#endif

#ifdef HAVE_HPUX9
'''
 * 在 HP-UX 9 下，没有很好的方法来确定 ppa。
 * 因此，从 /dev/kmem 中读取它。
'''
static struct nlist nl[] = {
#define NL_IFNET 0
    { "ifnet" },
    { "" }
};

static char path_vmunix[] = "/hp-ux";

/* 确定指定 ifname 的 ppa 号码 */
static int
get_dlpi_ppa(register int fd, register const char *ifname, register u_int unit,
    u_int *ppa, register char *ebuf)
{
    register const char *cp;
    register int kd;
    void *addr;
    struct ifnet ifnet;
    char if_name[sizeof(ifnet.if_name) + 1];

    cp = strrchr(ifname, '/');
    if (cp != NULL)
        ifname = cp + 1;
    if (nlist(path_vmunix, &nl) < 0) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "nlist %s failed",
            path_vmunix);
        return (PCAP_ERROR);
    }
    if (nl[NL_IFNET].n_value == 0) {
        snprintf(ebuf, PCAP_ERRBUF_SIZE,
            "couldn't find %s kernel symbol",
            nl[NL_IFNET].n_name);
        return (PCAP_ERROR);
    }
    kd = open("/dev/kmem", O_RDONLY);
    if (kd < 0) {
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "kmem open");
        return (PCAP_ERROR);
    }
    if (dlpi_kread(kd, nl[NL_IFNET].n_value,
        &addr, sizeof(addr), ebuf) < 0) {
        close(kd);
        return (PCAP_ERROR);
    }
    for (; addr != NULL; addr = ifnet.if_next) {
        if (dlpi_kread(kd, (off_t)addr,
            &ifnet, sizeof(ifnet), ebuf) < 0 ||
            dlpi_kread(kd, (off_t)ifnet.if_name,
            if_name, sizeof(ifnet.if_name), ebuf) < 0) {
            (void)close(kd);
            return (PCAP_ERROR);
        }
        if_name[sizeof(ifnet.if_name)] = '\0';
        if (strcmp(if_name, ifname) == 0 && ifnet.if_unit == unit) {
            *ppa = ifnet.if_index;
            return (0);
        }
    }

    snprintf(ebuf, PCAP_ERRBUF_SIZE, "Can't find %s", ifname);
    return (PCAP_ERROR_NO_SUCH_DEVICE);
}

static int
dlpi_kread(register int fd, register off_t addr,
    # 声明一个指向 void 类型的指针 buf，一个无符号整数类型 len，一个指向字符类型的指针 ebuf
    register void *buf, register u_int len, register char *ebuf)
{
    // 声明一个整型变量 cc
    register int cc;

    // 如果文件描述符 fd 定位到 addr 处失败
    if (lseek(fd, addr, SEEK_SET) < 0) {
        // 格式化错误消息，存储到 ebuf 中
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "lseek");
        // 返回 -1 表示失败
        return (-1);
    }
    // 从文件描述符 fd 中读取 len 长度的数据到 buf 中
    cc = read(fd, buf, len);
    // 如果读取失败
    if (cc < 0) {
        // 格式化错误消息，存储到 ebuf 中
        pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
            errno, "read");
        // 返回 -1 表示失败
        return (-1);
    } else if (cc != len) {
        // 格式化错误消息，存储到 ebuf 中
        snprintf(ebuf, PCAP_ERRBUF_SIZE, "short read (%d != %d)", cc,
            len);
        // 返回 -1 表示失败
        return (-1);
    }
    // 返回读取的字节数
    return (cc);
}
#endif

// 创建一个 pcap_t 结构体，用于表示一个网络接口
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    pcap_t *p;
#ifdef DL_HP_RAWDLS
    struct pcap_dlpi *pd;
#endif

    // 调用 PCAP_CREATE_COMMON 函数创建一个 pcap_t 结构体
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dlpi);
    // 如果创建失败，返回 NULL
    if (p == NULL)
        return (NULL);

#ifdef DL_HP_RAWDLS
    // 如果定义了 DL_HP_RAWDLS 宏
    pd = p->priv;
    pd->send_fd = -1;    /* it hasn't been opened yet */
#endif

    // 设置 pcap_t 结构体的 activate_op 指向 pcap_activate_dlpi 函数
    p->activate_op = pcap_activate_dlpi;
    // 返回创建的 pcap_t 结构体
    return (p);
}

/*
 * 返回 libpcap 版本字符串
 */
const char *
pcap_lib_version(void)
{
    // 返回预定义的 PCAP_VERSION_STRING 字符串
    return (PCAP_VERSION_STRING);
}
```