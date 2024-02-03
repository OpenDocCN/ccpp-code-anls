# `nmap\libpcap\pcap-enet.c`

```cpp
/*
 * Stanford Enetfilter子程序用于tcpdump
 *
 * 基于MERIT NNstat etherifrt.c和Ultrix pcap-pf.c子程序。
 *
 * Rayan Zachariassen，CA*Net
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>

#include <net/if.h>
#include <pcap/bpf.h>
#include <net/enet.h>

#include <netinet/in.h>
#include <netinet/if_ether.h>

#include <stdio.h>
#include <errno.h>

#include "interface.h"

struct packet_header {
#ifdef    IBMRTPC
    struct LengthWords    length;
    struct tap_header    tap;
#endif    /* IBMRTPC */
    u_char            packet[8]
};

extern int errno;

#define BUFSPACE (4*1024)

/* Forwards */
static void efReadError(int, char *);

void
readloop(int cnt, int if_fd, struct bpf_program *fp, printfunc printit)
{
#ifdef    IBMRTPC
    register struct packet_header *ph;
    register u_char *bp;
    register int inc;
#else    /* !IBMRTPC */
    static struct timeval tv = { 0 };
#endif    /* IBMRTPC */
    register int cc, caplen;
    register struct bpf_insn *fcode = fp->bf_insns;
    union {
        struct packet_header hdr;
        u_char    p[BUFSPACE];
        u_short    s;
    } buf;

    while (1) {
        if ((cc = read(if_fd, (char *)buf.p, sizeof(buf))) < 0)
            efReadError(if_fd, "reader");

#ifdef    IBMRTPC
        /*
         * 遍历每个数据包。
         */
        bp = buf.p;
        while (cc > 0) {
            ph = (struct packet_header *)bp;
            caplen = ph->tap.th_wirelen > snaplen ? snaplen : ph->tap
/* 定义一个函数，用于处理捕获的数据包 */
.th_wirelen ;
            if (pcap_filter(fcode, (char *)ph->packet,
                        ph->tap.th_wirelen, caplen)) {
                if (cnt >= 0 && --cnt < 0)
                    goto out;
                (*printit)((char *)ph->packet,
                    (struct timeval *)ph->tap.th_timestamp,
                    ph->tap.th_wirelen, caplen);
            }
            inc = ph->length.PacketOffset;
            cc -= inc;
            bp += inc;
        }
#else    /* !IBMRTPC */
        caplen = cc > snaplen ? snaplen : cc ;
        if (pcap_filter(fcode, buf.hdr.packet, cc, caplen)) {
            if (cnt >= 0 && --cnt < 0)
                goto out;
            (*printit)(buf.hdr.packet, &tv, cc, caplen);
        }
#endif    /* IBMRTPC */
    }
 out:
    wrapup(if_fd);
}

/* 如果read()在数据包过滤器上返回错误，则调用此函数 */
static void
efReadError(int fid, char *msg)
{
    if (errno == EINVAL) {    /* 已经读取了MAXINT字节！ */
        if (lseek(fid, 0, 0) < 0) {
            perror("tcpdump: efReadError/lseek");
            exit(-1);
        }
        else
            return;
    }
    else {
        (void) fprintf(stderr, "tcpdump: ");
        perror(msg);
        exit(-1);
    }
}

/* 结束函数，关闭文件描述符 */
void
wrapup(int fd)
{
#ifdef    IBMRTPC
    struct enstats es;

    if (ioctl(fd, EIOSTATS, &es) == -1) {
        perror("tcpdump: enet ioctl EIOSTATS error");
        exit(-1);
    }

    fprintf(stderr, "%d packets queued", es.enStat_Rcnt);
    if (es.enStat_Rdrops > 0)
        fprintf(stderr, ", %d dropped", es.enStat_Rdrops);
    if (es.enStat_Reads > 0)
        fprintf(stderr, ", %d tcpdump %s", es.enStat_Reads,
                es.enStat_Reads > 1 ? "reads" : "read");
    if (es.enStat_MaxRead > 1)
        fprintf(stderr, ", %d packets in largest read",
            es.enStat_MaxRead);
    putc('\n', stderr);
#endif    /* IBMRTPC */
    close(fd);
}

/* 初始化设备函数，用于设置过滤器和控制块 */
int
initdevice(char *device, int pflag, int *linktype)
{
    struct eniocb ctl;
    struct enfilter filter;
    # 定义一个无符号整数变量 maxwaiting，用于存储最大等待时间
    u_int maxwaiting;
    # 定义一个整数变量 if_fd，用于存储文件描述符
    int if_fd;
#ifdef    IBMRTPC
    GETENETDEVICE(0, O_RDONLY, &if_fd);
#else    /* !IBMRTPC */
    // 如果不是 IBMRTPC 系统，则打开 /dev/enet 文件
    if_fd = open("/dev/enet", O_RDONLY, 0);
#endif    /* IBMRTPC */

    // 如果打开文件失败，则输出错误信息并退出程序
    if (if_fd == -1) {
        perror("tcpdump: enet open error");
        error(
"your system may not be properly configured; see \"man enet(4)\"");
        exit(-1);
    }

    /*  获取操作参数。*/

    // 通过 ioctl 获取网络接口的操作参数
    if (ioctl(if_fd, EIOCGETP, (char *)&ctl) == -1) {
        perror("tcpdump: enet ioctl EIOCGETP error");
        exit(-1);
    }

    /*  设置操作参数。*/

#ifdef    IBMRTPC
    // 如果是 IBMRTPC 系统，则设置特定的操作参数
    ctl.en_rtout = 1 * ctl.en_hz;
    ctl.en_tr_etherhead = 1;
    ctl.en_tap_network = 1;
    ctl.en_multi_packet = 1;
    ctl.en_maxlen = BUFSPACE;
#else    /* !IBMRTPC */
    // 如果不是 IBMRTPC 系统，则设置默认的操作参数
    ctl.en_rtout = 64;    /* randomly picked value for HZ */
#endif    /* IBMRTPC */
    // 通过 ioctl 设置网络接口的操作参数
    if (ioctl(if_fd, EIOCSETP, &ctl) == -1) {
        perror("tcpdump: enet ioctl EIOCSETP error");
        exit(-1);
    }

    /*  刷新接收队列，因为我们已经改变了操作参数，否则可能会接收没有头部的数据。*/

    // 通过 ioctl 刷新接收队列
    if (ioctl(if_fd, EIOCFLUSH) == -1) {
        perror("tcpdump: enet ioctl EIOCFLUSH error");
        exit(-1);
    }

    /*  将接收队列深度设置为最大值。*/

    // 获取最大等待数
    maxwaiting = ctl.en_maxwaiting;
    // 通过 ioctl 设置接收队列深度
    if (ioctl(if_fd, EIOCSETW, &maxwaiting) == -1) {
        perror("tcpdump: enet ioctl EIOCSETW error");
        exit(-1);
    }

#ifdef    IBMRTPC
    /*  清除统计信息。*/

    // 如果是 IBMRTPC 系统，则清除统计信息
    if (ioctl(if_fd, EIOCLRSTAT, 0) == -1) {
        perror("tcpdump: enet ioctl EIOCLRSTAT error");
        exit(-1);
    }
#endif    /* IBMRTPC */

    /*  设置过滤器（接受所有数据包）。*/

    // 设置过滤器，接受所有数据包
    filter.enf_Priority = 3;
    filter.enf_FilterLen = 0;
    if (ioctl(if_fd, EIOCSETF, &filter) == -1) {
        perror("tcpdump: enet ioctl EIOCSETF error");
        exit(-1);
    }
    /*
     * "enetfilter" 仅支持以太网。
     */
    *linktype = DLT_EN10MB;

    return(if_fd);
}
```