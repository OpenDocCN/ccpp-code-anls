# `nmap\libpcap\bpf_filter.c`

```
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap/pcap-inttypes.h>
#include "pcap-types.h"
#include "extract.h"
#include "diag-control.h"

#define EXTRACT_SHORT    EXTRACT_BE_U_2
#define EXTRACT_LONG    EXTRACT_BE_U_4

#ifndef _WIN32
#include <sys/param.h>
#include <sys/types.h>
#include <sys/time.h>
#endif /* _WIN32 */

#include <pcap-int.h>

#include <stdlib.h>

#ifdef __linux__
#include <linux/types.h>
#include <linux/if_packet.h>
#include <linux/filter.h>
#endif

enum {
        BPF_S_ANC_NONE,
        BPF_S_ANC_VLAN_TAG,
        BPF_S_ANC_VLAN_TAG_PRESENT,
};

/*
 * Execute the filter program starting at pc on the packet p
 * wirelen is the length of the original packet
 * buflen is the amount of data present
 * aux_data is auxiliary data, currently used only when interpreting
 * filters intended for the Linux kernel in cases where the kernel
 * rejects the filter; it contains VLAN tag information
 * For the kernel, p is assumed to be a pointer to an mbuf if buflen is 0,
 * in all other cases, p is a pointer to a buffer and buflen is its size.
 *
 * Thanks to Ani Sinha <ani@arista.com> for providing initial implementation
 */
#if defined(SKF_AD_VLAN_TAG_PRESENT)
u_int
pcap_filter_with_aux_data(const struct bpf_insn *pc, const u_char *p,
    u_int wirelen, u_int buflen, const struct pcap_bpf_aux_data *aux_data)
#else
u_int
pcap_filter_with_aux_data(const struct bpf_insn *pc, const u_char *p,
    u_int wirelen, u_int buflen, const struct pcap_bpf_aux_data *aux_data _U_)
#endif
{
    register uint32_t A, X;
    register bpf_u_int32 k;
    uint32_t mem[BPF_MEMWORDS];

    if (pc == 0)
        /*
         * No filter means accept all.
         */
        return (u_int)-1;
    A = 0;
    X = 0;
    --pc;
    # 进入无限循环
    for (;;) {
        # 增加程序计数器的值
        ++pc;
        # 根据程序计数器的指令代码进行不同的操作
        switch (pc->code) {

        # 默认情况下，终止程序
        default:
            abort();
        # 如果指令是返回常数值，则返回该常数值
        case BPF_RET|BPF_K:
            return (u_int)pc->k;

        # 如果指令是返回寄存器A的值，则返回寄存器A的值
        case BPF_RET|BPF_A:
            return (u_int)A;

        # 如果指令是加载绝对地址的32位字，进行相应的操作
        case BPF_LD|BPF_W|BPF_ABS:
            # 获取绝对地址
            k = pc->k;
            # 如果地址超出数据长度或者32位字超出数据长度，则返回0
            if (k > buflen || sizeof(int32_t) > buflen - k) {
                return 0;
            }
            # 从数据中提取32位字，存入寄存器A
            A = EXTRACT_LONG(&p[k]);
            # 继续下一次循环
            continue;

        # 如果指令是加载绝对地址的16位字，进行相应的操作
        case BPF_LD|BPF_H|BPF_ABS:
            # 获取绝对地址
            k = pc->k;
            # 如果地址超出数据长度或者16位字超出数据长度，则返回0
            if (k > buflen || sizeof(int16_t) > buflen - k) {
                return 0;
            }
            # 从数据中提取16位字，存入寄存器A
            A = EXTRACT_SHORT(&p[k]);
            # 继续下一次循环
            continue;

        # 如果指令是加载绝对地址的8位字，进行相应的操作
        case BPF_LD|BPF_B|BPF_ABS:
            # 这个开关语句在构建用于具有移除的 VLAN 标签可用作元数据的 Linux 内核时才会执行
            # 什么都不做
DIAG_OFF_DEFAULT_ONLY_SWITCH
            switch (pc->k) {
#if defined(SKF_AD_VLAN_TAG_PRESENT)
            case SKF_AD_OFF + SKF_AD_VLAN_TAG:
                if (!aux_data)
                    return 0;
                A = aux_data->vlan_tag;
                break;

            case SKF_AD_OFF + SKF_AD_VLAN_TAG_PRESENT:
                if (!aux_data)
                    return 0;
                A = aux_data->vlan_tag_present;
                break;
#endif
            default:
                k = pc->k;
                if (k >= buflen) {
                    return 0;
                }
                A = p[k];
                break;
            }
    }
}

u_int
pcap_filter(const struct bpf_insn *pc, const u_char *p, u_int wirelen,
    u_int buflen)
{
    return pcap_filter_with_aux_data(pc, p, wirelen, buflen, NULL);
}
/*
 * Return true if the 'fcode' is a valid filter program.
 * The constraints are that each jump be forward and to a valid
 * code, that memory accesses are within valid ranges (to the
 * extent that this can be checked statically; loads of packet
 * data have to be, and are, also checked at run time), and that
 * the code terminates with either an accept or reject.
 *
 * The kernel needs to be able to verify an application's filter code.
 * Otherwise, a bogus program could easily crash the system.
 */
int
pcap_validate_filter(const struct bpf_insn *f, int len)
{
    u_int i, from;
    const struct bpf_insn *p;

    if (len < 1)
        return 0;

    }
    return BPF_CLASS(f[len - 1].code) == BPF_RET;
}

/*
 * Exported because older versions of libpcap exported them.
 */
u_int
bpf_filter(const struct bpf_insn *pc, const u_char *p, u_int wirelen,
    u_int buflen)
{
    return pcap_filter(pc, p, wirelen, buflen);
}

int
bpf_validate(const struct bpf_insn *f, int len)
{
    return pcap_validate_filter(f, len);
}
```