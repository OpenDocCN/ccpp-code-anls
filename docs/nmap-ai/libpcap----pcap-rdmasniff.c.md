# `nmap\libpcap\pcap-rdmasniff.c`

```cpp
/*
 * 版权声明
 * 本软件的再发布和使用需要满足以下条件：
 * 1. 源代码发布需要保留版权声明、条件列表和以下免责声明
 * 2. 二进制发布需要在文档或其他提供的材料中重现版权声明、条件列表和以下免责声明
 * 3. 未经特定书面许可，不得使用作者的名字来认可或推广基于本软件的产品
 * 
 * 本软件由版权所有者和贡献者提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。在任何情况下，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他），版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcap-int.h"
#include "pcap-rdmasniff.h"

#include <infiniband/verbs.h>
#include <stdlib.h>
#include <string.h>
#include <limits.h> /* for INT_MAX */
#include <sys/time.h>

#if !defined(IBV_FLOW_ATTR_SNIFFER)
#define IBV_FLOW_ATTR_SNIFFER    3
#endif

static const int RDMASNIFF_NUM_RECEIVES = 128;  // 定义常量 RDMASNIFF_NUM_RECEIVES 为 128
static const int RDMASNIFF_RECEIVE_SIZE = 10000;  // 定义常量 RDMASNIFF_RECEIVE_SIZE 为 10000

struct pcap_rdmasniff {
    struct ibv_device *        rdma_device;  // 定义结构体成员 rdma_device 为 InfiniBand 设备指针
    # 定义 InfiniBand Verbs 上下文结构体指针
    struct ibv_context *        context;
    # 定义完成通道结构体指针
    struct ibv_comp_channel *    channel;
    # 定义保护域结构体指针
    struct ibv_pd *            pd;
    # 定义完成队列结构体指针
    struct ibv_cq *            cq;
    # 定义队列对结构体指针
    struct ibv_qp *            qp;
    # 定义流控制结构体指针
    struct ibv_flow *               flow;
    # 定义内存区域结构体指针
    struct ibv_mr *            mr;
    # 定义一次性缓冲区指针
    u_char *            oneshot_buffer;
    # 定义端口号
    unsigned long            port_num;
    # 定义完成队列事件
    int                             cq_event;
    # 定义接收数据包数量
    u_int                           packets_recv;
};

# 读取 RDMA 抓包统计信息
static int
rdmasniff_stats(pcap_t *handle, struct pcap_stat *stat)
{
    # 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;

    # 设置接收的数据包数量
    stat->ps_recv = priv->packets_recv;
    # 设置丢弃的数据包数量为 0
    stat->ps_drop = 0;
    # 设置接口丢弃的数据包数量为 0
    stat->ps_ifdrop = 0;

    return 0;
}

# 清理 RDMA 抓包资源
static void
rdmasniff_cleanup(pcap_t *handle)
{
    # 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;

    # 注销内存区域
    ibv_dereg_mr(priv->mr);
    # 销毁流规则
    ibv_destroy_flow(priv->flow);
    # 销毁队列对
    ibv_destroy_qp(priv->qp);
    # 销毁完成队列
    ibv_destroy_cq(priv->cq);
    # 释放保护域
    ibv_dealloc_pd(priv->pd);
    # 销毁完成通道
    ibv_destroy_comp_channel(priv->channel);
    # 关闭设备
    ibv_close_device(priv->context);
    # 释放内存
    free(priv->oneshot_buffer);

    # 清理 pcap 公共资源
    pcap_cleanup_live_common(handle);
}

# 发送接收请求
static void
rdmasniff_post_recv(pcap_t *handle, uint64_t wr_id)
{
    # 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;
    struct ibv_sge sg_entry;
    struct ibv_recv_wr wr, *bad_wr;

    # 设置 sg_entry 结构体的长度
    sg_entry.length = RDMASNIFF_RECEIVE_SIZE;
    # 设置 sg_entry 结构体的地址
    sg_entry.addr = (uintptr_t) handle->buffer + RDMASNIFF_RECEIVE_SIZE * wr_id;
    # 设置 sg_entry 结构体的 lkey
    sg_entry.lkey = priv->mr->lkey;

    # 设置 wr 结构体的 wr_id
    wr.wr_id = wr_id;
    # 设置 wr 结构体的 num_sge
    wr.num_sge = 1;
    # 设置 wr 结构体的 sg_list
    wr.sg_list = &sg_entry;
    # 设置 wr 结构体的 next
    wr.next = NULL;

    # 发送接收请求
    ibv_post_recv(priv->qp, &wr, &bad_wr);
}

# 读取数据包
static int
rdmasniff_read(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user)
{
    # 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;
    struct ibv_cq *ev_cq;
    void *ev_ctx;
    struct ibv_wc wc;
    struct pcap_pkthdr pkth;
    u_char *pktd;
    int count = 0;

    # 如果没有完成队列事件
    if (!priv->cq_event) {
        # 循环等待完成队列事件
        while (ibv_get_cq_event(priv->channel, &ev_cq, &ev_ctx) < 0) {
            # 如果错误不是中断错误，则返回错误
            if (errno != EINTR) {
                return PCAP_ERROR;
            }
            # 如果需要中断循环，则返回中断错误
            if (handle->break_loop) {
                handle->break_loop = 0;
                return PCAP_ERROR_BREAK;
            }
        }
        # 确认完成队列事件
        ibv_ack_cq_events(priv->cq, 1);
        # 请求通知完成队列事件
        ibv_req_notify_cq(priv->cq, 0);
        # 设置完成队列事件标志为 1
        priv->cq_event = 1;
    }
    /*
     * 如果最大数据包数是无限的，可能会处理超过 INT_MAX 个数据包，
     * 这将导致数据包计数溢出，要么看起来像是负数，从而导致我们返回一个看起来像是错误的值，
     * 要么溢出回到正数领域，从而导致我们返回一个太低的计数。
     *
     * 因此，如果数据包计数是无限的，我们将其剪切到 INT_MAX；这个例程不应该无限处理数据包，所以这不是一个问题。
     */
    if (PACKET_COUNT_IS_UNLIMITED(max_packets))
        max_packets = INT_MAX;

    while (count < max_packets) {
        if (ibv_poll_cq(priv->cq, 1, &wc) != 1) {
            priv->cq_event = 0;
            break;
        }

        if (wc.status != IBV_WC_SUCCESS) {
            fprintf(stderr, "failed WC wr_id %" PRIu64 " status %d/%s\n",
                wc.wr_id,
                wc.status, ibv_wc_status_str(wc.status));
            continue;
        }

        pkth.len = wc.byte_len;
        pkth.caplen = min(pkth.len, (u_int)handle->snapshot);
        gettimeofday(&pkth.ts, NULL);

        pktd = (u_char *) handle->buffer + wc.wr_id * RDMASNIFF_RECEIVE_SIZE;

        if (handle->fcode.bf_insns == NULL ||
            pcap_filter(handle->fcode.bf_insns, pktd, pkth.len, pkth.caplen)) {
            callback(user, &pkth, pktd);
            ++priv->packets_recv;
            ++count;
        }

        rdmasniff_post_recv(handle, wc.wr_id);

        if (handle->break_loop) {
            handle->break_loop = 0;
            return PCAP_ERROR_BREAK;
        }
    }

    return count;
}

static void
rdmasniff_oneshot(u_char *user, const struct pcap_pkthdr *h, const u_char *bytes)
{
    // 从用户数据中获取指向 oneshot_userdata 结构体的指针
    struct oneshot_userdata *sp = (struct oneshot_userdata *) user;
    // 获取 pcap_t 结构体指针
    pcap_t *handle = sp->pd;
    // 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;

    // 将数据包头信息拷贝到 sp->hdr 中
    *sp->hdr = *h;
    // 将数据包内容拷贝到 priv->oneshot_buffer 中
    memcpy(priv->oneshot_buffer, bytes, h->caplen);
    // 将 priv->oneshot_buffer 的指针赋值给 sp->pkt
    *sp->pkt = priv->oneshot_buffer;
}

static int
rdmasniff_activate(pcap_t *handle)
{
    // 获取 pcap_rdmasniff 结构体指针
    struct pcap_rdmasniff *priv = handle->priv;
    // 定义用于初始化的结构体和变量
    struct ibv_qp_init_attr qp_init_attr;
    struct ibv_qp_attr qp_attr;
    struct ibv_flow_attr flow_attr;
    struct ibv_port_attr port_attr;
    int i;

    // 打开 RDMA 设备，获取上下文
    priv->context = ibv_open_device(priv->rdma_device);
    if (!priv->context) {
        // 如果打开设备失败，设置错误信息并跳转到错误处理部分
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to open device %s", handle->opt.device);
        goto error;
    }

    // 为设备分配内存池
    priv->pd = ibv_alloc_pd(priv->context);
    if (!priv->pd) {
        // 如果分配内存池失败，设置错误信息并跳转到错误处理部分
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to alloc PD for device %s", handle->opt.device);
        goto error;
    }

    // 创建完成通道
    priv->channel = ibv_create_comp_channel(priv->context);
    if (!priv->channel) {
        // 如果创建完成通道失败，设置错误信息并跳转到错误处理部分
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to create comp channel for device %s", handle->opt.device);
        goto error;
    }

    // 创建完成队列
    priv->cq = ibv_create_cq(priv->context, RDMASNIFF_NUM_RECEIVES,
                 NULL, priv->channel, 0);
    if (!priv->cq) {
        // 如果创建完成队列失败，设置错误信息并跳转到错误处理部分
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to create CQ for device %s", handle->opt.device);
        goto error;
    }

    // 请求通知完成队列
    ibv_req_notify_cq(priv->cq, 0);

    // 初始化 QP 属性
    memset(&qp_init_attr, 0, sizeof qp_init_attr);
    qp_init_attr.send_cq = qp_init_attr.recv_cq = priv->cq;
    qp_init_attr.cap.max_recv_wr = RDMASNIFF_NUM_RECEIVES;
    qp_init_attr.cap.max_recv_sge = 1;
    qp_init_attr.qp_type = IBV_QPT_RAW_PACKET;
    // 创建 QP
    priv->qp = ibv_create_qp(priv->pd, &qp_init_attr);
    // 如果私有数据结构中的 QP 为空，则表示创建 QP 失败，设置错误信息并跳转到错误处理
    if (!priv->qp) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to create QP for device %s", handle->opt.device);
        goto error;
    }

    // 初始化 qp_attr 结构体，并设置 QP 的状态为 INIT
    memset(&qp_attr, 0, sizeof qp_attr);
    qp_attr.qp_state = IBV_QPS_INIT;
    qp_attr.port_num = priv->port_num;
    // 修改 QP 的状态为 INIT，如果失败则设置错误信息并跳转到错误处理
    if (ibv_modify_qp(priv->qp, &qp_attr, IBV_QP_STATE | IBV_QP_PORT)) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to modify QP to INIT for device %s", handle->opt.device);
        goto error;
    }

    // 初始化 qp_attr 结构体，并设置 QP 的状态为 RTR
    memset(&qp_attr, 0, sizeof qp_attr);
    qp_attr.qp_state = IBV_QPS_RTR;
    // 修改 QP 的状态为 RTR，如果失败则设置错误信息并跳转到错误处理
    if (ibv_modify_qp(priv->qp, &qp_attr, IBV_QP_STATE)) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to modify QP to RTR for device %s", handle->opt.device);
        goto error;
    }

    // 初始化 flow_attr 结构体，并设置流的类型为 IBV_FLOW_ATTR_SNIFFER
    memset(&flow_attr, 0, sizeof flow_attr);
    flow_attr.type = IBV_FLOW_ATTR_SNIFFER;
    flow_attr.size = sizeof flow_attr;
    flow_attr.port = priv->port_num;
    // 创建流，如果失败则设置错误信息并跳转到错误处理
    priv->flow = ibv_create_flow(priv->qp, &flow_attr);
    if (!priv->flow) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to create flow for device %s", handle->opt.device);
        goto error;
    }

    // 设置接收缓冲区的大小
    handle->bufsize = RDMASNIFF_NUM_RECEIVES * RDMASNIFF_RECEIVE_SIZE;
    // 分配接收缓冲区的内存，如果失败则设置错误信息并跳转到错误处理
    handle->buffer = malloc(handle->bufsize);
    if (!handle->buffer) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to allocate receive buffer for device %s", handle->opt.device);
        goto error;
    }

    // 分配一次性缓冲区的内存，如果失败则设置错误信息并跳转到错误处理
    priv->oneshot_buffer = malloc(RDMASNIFF_RECEIVE_SIZE);
    if (!priv->oneshot_buffer) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to allocate oneshot buffer for device %s", handle->opt.device);
        goto error;
    }

    // 注册接收缓冲区的内存，以便进行本地写操作
    priv->mr = ibv_reg_mr(priv->pd, handle->buffer, handle->bufsize, IBV_ACCESS_LOCAL_WRITE);
    # 如果没有注册 MR（Memory Region），则设置错误信息并跳转到错误处理部分
    if (!priv->mr) {
        snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
                  "Failed to register MR for device %s", handle->opt.device);
        goto error;
    }

    # 循环创建并提交接收请求
    for (i = 0; i < RDMASNIFF_NUM_RECEIVES; ++i) {
        rdmasniff_post_recv(handle, i);
    }

    # 查询端口属性，根据端口属性设置链路类型
    if (!ibv_query_port(priv->context, priv->port_num, &port_attr) &&
        port_attr.link_layer == IBV_LINK_LAYER_INFINIBAND) {
        handle->linktype = DLT_INFINIBAND;
    } else {
        handle->linktype = DLT_EN10MB;
    }

    # 如果快照长度小于等于0或者大于接收数据的大小，则将快照长度设置为接收数据的大小
    if (handle->snapshot <= 0 || handle->snapshot > RDMASNIFF_RECEIVE_SIZE)
        handle->snapshot = RDMASNIFF_RECEIVE_SIZE;

    # 初始化偏移量和操作函数
    handle->offset = 0;
    handle->read_op = rdmasniff_read;
    handle->stats_op = rdmasniff_stats;
    handle->cleanup_op = rdmasniff_cleanup;
    handle->setfilter_op = install_bpf_program;
    handle->setdirection_op = NULL;
    handle->set_datalink_op = NULL;
    handle->getnonblock_op = pcap_getnonblock_fd;
    handle->setnonblock_op = pcap_setnonblock_fd;
    handle->oneshot_callback = rdmasniff_oneshot;
    handle->selectable_fd = priv->channel->fd;

    # 返回成功
    return 0;
    # 如果私有数据结构中存在内存区域对象，则注销该内存区域对象
    if (priv->mr) {
        ibv_dereg_mr(priv->mr);
    }

    # 如果私有数据结构中存在流对象，则销毁该流对象
    if (priv->flow) {
        ibv_destroy_flow(priv->flow);
    }

    # 如果私有数据结构中存在队列对对象，则销毁该队列对对象
    if (priv->qp) {
        ibv_destroy_qp(priv->qp);
    }

    # 如果私有数据结构中存在完成队列对象，则销毁该完成队列对象
    if (priv->cq) {
        ibv_destroy_cq(priv->cq);
    }

    # 如果私有数据结构中存在完成通道对象，则销毁该完成通道对象
    if (priv->channel) {
        ibv_destroy_comp_channel(priv->channel);
    }

    # 如果私有数据结构中存在保护域对象，则释放该保护域对象
    if (priv->pd) {
        ibv_dealloc_pd(priv->pd);
    }

    # 如果私有数据结构中存在设备上下文对象，则关闭该设备上下文对象
    if (priv->context) {
        ibv_close_device(priv->context);
    }

    # 如果私有数据结构中存在一次性缓冲区对象，则释放该一次性缓冲区对象
    if (priv->oneshot_buffer) {
        free(priv->oneshot_buffer);
    }

    # 返回错误代码
    return PCAP_ERROR;
}

# 创建一个 RDMA 抓包会话
pcap_t *
rdmasniff_create(const char *device, char *ebuf, int *is_ours)
{
    struct pcap_rdmasniff *priv;
    struct ibv_device **dev_list;
    int numdev;
    size_t namelen;
    const char *port;
    unsigned long port_num;
    int i;
    pcap_t *p = NULL;

    # 初始化 is_ours 标志
    *is_ours = 0;

    # 获取设备列表
    dev_list = ibv_get_device_list(&numdev);
    if (!dev_list) {
        return NULL;
    }
    if (!numdev) {
        ibv_free_device_list(dev_list);
        return NULL;
    }

    # 计算设备名的长度
    namelen = strlen(device);

    # 查找设备名中的端口号
    port = strchr(device, ':');
    if (port) {
        port_num = strtoul(port + 1, NULL, 10);
        if (port_num > 0) {
            namelen = port - device;
        } else {
            port_num = 1;
        }
    } else {
        port_num = 1;
    }

    # 遍历设备列表，查找匹配的设备
    for (i = 0; i < numdev; ++i) {
        if (strlen(dev_list[i]->name) == namelen &&
            !strncmp(device, dev_list[i]->name, namelen)) {
            *is_ours = 1;

            # 创建一个 RDMA 抓包会话
            p = PCAP_CREATE_COMMON(ebuf, struct pcap_rdmasniff);
            if (p) {
                p->activate_op = rdmasniff_activate;
                priv = p->priv;
                priv->rdma_device = dev_list[i];
                priv->port_num = port_num;
            }
            break;
        }
    }

    # 释放设备列表
    ibv_free_device_list(dev_list);
    return p;
}

# 查找所有的 RDMA 设备
int
rdmasniff_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
    struct ibv_device **dev_list;
    int numdev;
    int i;
    # 定义一个整型变量 ret，初始化为 0
    int ret = 0;

    # 获取 InfiniBand 设备列表，将列表存储在 dev_list 中，numdev 为列表长度
    dev_list = ibv_get_device_list(&numdev);
    # 如果获取的设备列表为空，则返回 0
    if (!dev_list) {
        return 0;
    }

    # 遍历设备列表
    for (i = 0; i < numdev; ++i) {
        '''
        XXX - 这里是否适用于 "up", "running", 或 "connected" 的概念？
        '''
        # 将设备信息添加到 devlistp 中，如果添加失败，则将 ret 设为 -1 并跳出循环
        if (!add_dev(devlistp, dev_list[i]->name, 0, "RDMA sniffer", err_str)) {
            ret = -1;
            break;
        }
    }

    # 释放设备列表
    ibv_free_device_list(dev_list);
    # 返回 ret
    return ret;
# 闭合前面的函数定义
```