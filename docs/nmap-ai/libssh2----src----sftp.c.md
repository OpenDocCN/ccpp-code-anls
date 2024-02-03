# `nmap\libssh2\src\sftp.c`

```cpp
/*
 * 作者声明和版权信息
 * 作者：Sara Golemon <sarag@libssh2.org>，Eli Fant <elifantu@mail.ru>，Daniel Stenberg
 * 版权声明：保留所有权利
 *
 * 在源代码和二进制形式下的重新分发和修改都是允许的，前提是满足以下条件：
 *   - 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明
 *   - 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和免责声明
 *   - 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任
 * 即使已被告知可能发生此类损害，也不承担任何责任
 */
#include <assert.h>

#include "libssh2_priv.h"
#include "libssh2_sftp.h"
#include "channel.h"
#include "session.h"
#include "sftp.h"
/* Note: Version 6 was documented at the time of writing
 * However it was marked as "DO NOT IMPLEMENT" due to pending changes
 *
 * This release of libssh2 implements Version 5 with automatic downgrade
 * based on server's declaration
 */

/* SFTP packet types */
# 定义 SFTP 数据包类型
#define SSH_FXP_INIT                            1
#define SSH_FXP_VERSION                         2
#define SSH_FXP_OPEN                            3
#define SSH_FXP_CLOSE                           4
#define SSH_FXP_READ                            5
#define SSH_FXP_WRITE                           6
#define SSH_FXP_LSTAT                           7
#define SSH_FXP_FSTAT                           8
#define SSH_FXP_SETSTAT                         9
#define SSH_FXP_FSETSTAT                        10
#define SSH_FXP_OPENDIR                         11
#define SSH_FXP_READDIR                         12
#define SSH_FXP_REMOVE                          13
#define SSH_FXP_MKDIR                           14
#define SSH_FXP_RMDIR                           15
#define SSH_FXP_REALPATH                        16
#define SSH_FXP_STAT                            17
#define SSH_FXP_RENAME                          18
#define SSH_FXP_READLINK                        19
#define SSH_FXP_SYMLINK                         20
#define SSH_FXP_STATUS                          101
#define SSH_FXP_HANDLE                          102
#define SSH_FXP_DATA                            103
#define SSH_FXP_NAME                            104
#define SSH_FXP_ATTRS                           105
#define SSH_FXP_EXTENDED                        200
#define SSH_FXP_EXTENDED_REPLY                  201

/* S_IFREG */
# 定义文件类型 S_IFREG
#define LIBSSH2_SFTP_ATTR_PFILETYPE_FILE        0100000
/* S_IFDIR */
# 定义目录类型 S_IFDIR
#define LIBSSH2_SFTP_ATTR_PFILETYPE_DIR         0040000

# 定义 SSH_FXE_STATVFS_ST_RDONLY
#define SSH_FXE_STATVFS_ST_RDONLY               0x00000001
# 定义 SSH_FXE_STATVFS_ST_NOSUID
#define SSH_FXE_STATVFS_ST_NOSUID               0x00000002
/* This is the maximum packet length to accept, as larger than this indicate
   some kind of server problem. */
#define LIBSSH2_SFTP_PACKET_MAXLEN  (256 * 1024)

static int sftp_packet_ask(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                           uint32_t request_id, unsigned char **data,
                           size_t *data_len);
static void sftp_packet_flush(LIBSSH2_SFTP *sftp);

/* sftp_attrsize
 * Size that attr with this flagset will occupy when turned into a bin struct
 */
static int sftp_attrsize(unsigned long flags)
{
    return (4 +                                 /* flags(4) */
            ((flags & LIBSSH2_SFTP_ATTR_SIZE) ? 8 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_UIDGID) ? 8 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) ? 4 : 0) +
            ((flags & LIBSSH2_SFTP_ATTR_ACMODTIME) ? 8 : 0));
    /* atime + mtime as u32 */
}

/* _libssh2_store_u64
 */
static void _libssh2_store_u64(unsigned char **ptr, libssh2_uint64_t value)
{
    uint32_t msl = (uint32_t)(value >> 32);
    unsigned char *buf = *ptr;

    buf[0] = (unsigned char)((msl >> 24) & 0xFF);
    buf[1] = (unsigned char)((msl >> 16) & 0xFF);
    buf[2] = (unsigned char)((msl >> 8)  & 0xFF);
    buf[3] = (unsigned char)( msl        & 0xFF);

    buf[4] = (unsigned char)((value >> 24) & 0xFF);
    buf[5] = (unsigned char)((value >> 16) & 0xFF);
    buf[6] = (unsigned char)((value >> 8)  & 0xFF);
    buf[7] = (unsigned char)( value        & 0xFF);

    *ptr += 8;
}

/*
 * Search list of zombied FXP_READ request IDs.
 *
 * Returns NULL if ID not in list.
 */
static struct sftp_zombie_requests *
find_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    struct sftp_zombie_requests *zombie =
        _libssh2_list_first(&sftp->zombie_requests);

    while(zombie) {
        if(zombie->request_id == request_id)
            break;
        else
            zombie = _libssh2_list_next(&zombie->node);
    }

    return zombie;
}

static void
# 从 zombie 请求列表中移除指定请求
remove_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    # 获取 SFTP 会话对应的 SSH 会话
    LIBSSH2_SESSION *session = sftp->channel->session;

    # 在 zombie 请求列表中查找指定请求
    struct sftp_zombie_requests *zombie = find_zombie_request(sftp, request_id);
    # 如果找到了指定请求
    if(zombie) {
        # 输出调试信息，表示正在从 zombie 请求列表中移除指定请求
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Removing request ID %ld from the list of "
                       "zombie requests",
                       request_id);
        # 从列表中移除指定请求
        _libssh2_list_remove(&zombie->node);
        # 释放请求占用的内存
        LIBSSH2_FREE(session, zombie);
    }
}

# 将请求标记为 zombie 请求
static int
add_zombie_request(LIBSSH2_SFTP *sftp, uint32_t request_id)
{
    # 获取 SFTP 会话对应的 SSH 会话
    LIBSSH2_SESSION *session = sftp->channel->session;

    # 定义 zombie 请求结构体
    struct sftp_zombie_requests *zombie;

    # 输出调试信息，表示正在将请求标记为 zombie 请求
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Marking request ID %ld as a zombie request", request_id);

    # 分配内存给 zombie 请求
    zombie = LIBSSH2_ALLOC(sftp->channel->session,
                           sizeof(struct sftp_zombie_requests));
    # 如果内存分配失败
    if(!zombie)
        # 返回内存分配失败的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "malloc fail for zombie request  ID");
    else {
        # 设置请求 ID
        zombie->request_id = request_id;
        # 将请求添加到 zombie 请求列表中
        _libssh2_list_add(&sftp->zombie_requests, &zombie->node);
        # 返回无错误
        return LIBSSH2_ERROR_NONE;
    }
}

# 添加一个数据包到 SFTP 数据包队列中
static int
sftp_packet_add(LIBSSH2_SFTP *sftp, unsigned char *data,
                size_t data_len)
{
    # 获取 SFTP 会话对应的 SSH 会话
    LIBSSH2_SESSION *session = sftp->channel->session;
    # 定义 SFTP 数据包结构体
    LIBSSH2_SFTP_PACKET *packet;
    # 请求 ID
    uint32_t request_id;

    # 如果数据长度小于 5
    if(data_len < 5) {
        # 返回越界错误
        return LIBSSH2_ERROR_OUT_OF_BOUNDARY;
    }

    # 输出调试信息，表示接收到数据包
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Received packet type %d (len %d)",
                   (int) data[0], data_len);
    /*
     * 经验表明，如果我们在某个地方搞乱了 EAGAIN 处理，或者与通道不同步，
     * 这里是我们第一次得到错误字节的地方，如果是这样，我们需要立即退出以更好地跟踪问题。
     */

    // 根据 data[0] 的值进行不同的处理
    switch(data[0]) {
    // 如果是以下情况之一，不做任何处理
    case SSH_FXP_INIT:
    case SSH_FXP_VERSION:
    case SSH_FXP_OPEN:
    case SSH_FXP_CLOSE:
    case SSH_FXP_READ:
    case SSH_FXP_WRITE:
    case SSH_FXP_LSTAT:
    case SSH_FXP_FSTAT:
    case SSH_FXP_SETSTAT:
    case SSH_FXP_FSETSTAT:
    case SSH_FXP_OPENDIR:
    case SSH_FXP_READDIR:
    case SSH_FXP_REMOVE:
    case SSH_FXP_MKDIR:
    case SSH_FXP_RMDIR:
    case SSH_FXP_REALPATH:
    case SSH_FXP_STAT:
    case SSH_FXP_RENAME:
    case SSH_FXP_READLINK:
    case SSH_FXP_SYMLINK:
    case SSH_FXP_STATUS:
    case SSH_FXP_HANDLE:
    case SSH_FXP_DATA:
    case SSH_FXP_NAME:
    case SSH_FXP_ATTRS:
    case SSH_FXP_EXTENDED:
    case SSH_FXP_EXTENDED_REPLY:
        break;
    // 如果不是以上情况之一，返回错误信息
    default:
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Out of sync with the world");
    }

    // 获取请求的 ID
    request_id = _libssh2_ntohu32(&data[1]);

    // 打印调试信息
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Received packet id %d",
                   request_id);

    /* 如果数据包是回应我们放弃的请求，则不添加数据包 */
    if((data[0] == SSH_FXP_STATUS || data[0] == SSH_FXP_DATA)
       && find_zombie_request(sftp, request_id)) {

        /* 如果我们到达这里，文件在响应到达之前结束了。我们不再对请求感兴趣，所以我们丢弃它 */

        // 释放内存
        LIBSSH2_FREE(session, data);

        // 移除放弃的请求
        remove_zombie_request(sftp, request_id);
        return LIBSSH2_ERROR_NONE;
    }

    // 分配内存给数据包
    packet = LIBSSH2_ALLOC(session, sizeof(LIBSSH2_SFTP_PACKET));
    if(!packet) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate datablock for SFTP packet");
    }

    // 将数据包的数据指针指向 data
    packet->data = data;
    # 设置数据长度字段为给定的数据长度
    packet->data_len = data_len;
    # 设置请求 ID 字段为给定的请求 ID
    packet->request_id = request_id;

    # 将当前数据包添加到 SFTP 对象的数据包列表中
    _libssh2_list_add(&sftp->packets, &packet->node);

    # 返回无错误状态
    return LIBSSH2_ERROR_NONE;
}

/*
 * sftp_packet_read
 *
 * 从通道中获取一个 SFTP 数据包
 */
static int
sftp_packet_read(LIBSSH2_SFTP *sftp)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;  // 获取 SFTP 对应的通道
    LIBSSH2_SESSION *session = channel->session;  // 获取通道对应的会话
    unsigned char *packet = NULL;  // 初始化数据包为空
    ssize_t rc;  // 读取结果
    unsigned long recv_window;  // 接收窗口大小
    int packet_type;  // 数据包类型

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "recv packet");  // 调试信息

    switch(sftp->packet_state) {  // 根据数据包状态进行处理
    case libssh2_NB_state_sent: /* EAGAIN from window adjusting */  // 窗口调整导致的 EAGAIN
        sftp->packet_state = libssh2_NB_state_idle;  // 设置数据包状态为 idle

        packet = sftp->partial_packet;  // 获取部分数据包
        goto window_adjust;  // 跳转到窗口调整处理

    case libssh2_NB_state_sent1: /* EAGAIN from channel read */  // 通道读取导致的 EAGAIN
        sftp->packet_state = libssh2_NB_state_idle;  // 设置数据包状态为 idle

        packet = sftp->partial_packet;  // 获取部分数据包

        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "partial read cont, len: %lu", sftp->partial_len);  // 部分读取继续，长度信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "partial read cont, already recvd: %lu",
                       sftp->partial_received);  // 部分读取继续，已接收信息
        /* fall-through */
    }
    /* WON'T REACH */
}
/*
 * sftp_packetlist_flush
 *
 * 清空数据包列表中的所有待处理数据包，并在 SFTP 数据包队列中对应地清空
 */
static void sftp_packetlist_flush(LIBSSH2_SFTP_HANDLE *handle)
{
    struct sftp_pipeline_chunk *chunk;  // 数据包块
    LIBSSH2_SFTP *sftp = handle->sftp;  // 获取 SFTP 对象
    LIBSSH2_SESSION *session = sftp->channel->session;  // 获取会话对象

    /* remove pending packets, if any */
    chunk = _libssh2_list_first(&handle->packet_list);  // 获取待处理数据包列表中的第一个数据包
    // 当 chunk 存在时执行循环
    while(chunk) {
        // 定义指向数据的指针和数据长度
        unsigned char *data;
        size_t data_len;
        int rc;
        // 获取下一个 chunk
        struct sftp_pipeline_chunk *next = _libssh2_list_next(&chunk->node);

        // 向服务器发送请求，获取响应数据
        rc = sftp_packet_ask(sftp, SSH_FXP_STATUS,
                             chunk->request_id, &data, &data_len);
        // 如果没有响应数据，则再次发送请求获取数据
        if(rc)
            rc = sftp_packet_ask(sftp, SSH_FXP_DATA,
                                 chunk->request_id, &data, &data_len);

        // 如果存在响应数据，则释放它
        if(!rc)
            /* we found a packet, free it */
            LIBSSH2_FREE(session, data);
        // 如果 chunk 已发送请求但没有收到响应，则将其标记为 zombie
        else if(chunk->sent)
            /* there was no incoming packet for this request, mark this
               request as a zombie if it ever sent the request */
            add_zombie_request(sftp, chunk->request_id);

        // 从链表中移除当前 chunk
        _libssh2_list_remove(&chunk->node);
        // 释放当前 chunk
        LIBSSH2_FREE(session, chunk);
        // 将 chunk 指向下一个 chunk
        chunk = next;
    }
/*
 * sftp_packet_ask()
 *
 * 检查是否有匹配的 SFTP 数据包可用。
 */
static int
sftp_packet_ask(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                uint32_t request_id, unsigned char **data,
                size_t *data_len)
{
    LIBSSH2_SESSION *session = sftp->channel->session;  // 获取 SFTP 会话的会话对象
    LIBSSH2_SFTP_PACKET *packet = _libssh2_list_first(&sftp->packets);  // 获取 SFTP 数据包链表的第一个数据包

    if(!packet)  // 如果数据包为空
        return -1;  // 返回错误码 -1

    /* 获取 VERSION 数据包时的特殊考虑 */

    while(packet) {  // 循环遍历数据包链表
        if((packet->data[0] == packet_type) &&  // 如果数据包类型匹配
           ((packet_type == SSH_FXP_VERSION) ||  // 或者数据包类型为 SSH_FXP_VERSION
            (packet->request_id == request_id))) {  // 或者数据包的请求 ID 匹配

            /* 匹配！获取数据 */
            *data = packet->data;  // 将数据包的数据赋值给传入的数据指针
            *data_len = packet->data_len;  // 将数据包的数据长度赋值给传入的数据长度指针

            /* 取消链接并释放这个结构 */
            _libssh2_list_remove(&packet->node);  // 从链表中移除数据包
            LIBSSH2_FREE(session, packet);  // 释放数据包内存

            return 0;  // 返回成功码 0
        }
        /* 检查链表中的下一个结构 */
        packet = _libssh2_list_next(&packet->node);  // 获取链表中下一个数据包
    }
    return -1;  // 返回错误码 -1
}

/* sftp_packet_require
 * 类似于 libssh2_packet_require
 */
static int
sftp_packet_require(LIBSSH2_SFTP *sftp, unsigned char packet_type,
                    uint32_t request_id, unsigned char **data,
                    size_t *data_len, size_t required_size)
{
    LIBSSH2_SESSION *session = sftp->channel->session;  // 获取 SFTP 会话的会话对象
    int rc;

    if(data == NULL || data_len == NULL || required_size == 0) {  // 如果传入的数据指针为空，或者数据长度指针为空，或者要求的数据大小为 0
        return LIBSSH2_ERROR_BAD_USE;  // 返回错误码 LIBSSH2_ERROR_BAD_USE
    }

    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Requiring packet %d id %ld",
                   (int) packet_type, request_id);  // 打印调试信息，要求的数据包类型和请求 ID
}
    # 如果 sftp_packet_ask 函数返回 0，表示在数据包队列中找到了正确的数据包
    if(sftp_packet_ask(sftp, packet_type, request_id, data, data_len) == 0) {
        /* The right packet was available in the packet brigade */
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Got %d",
                       (int) packet_type);

        # 如果数据长度小于所需长度，返回缓冲区过小的错误
        if (*data_len < required_size) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }

        # 返回无错误
        return LIBSSH2_ERROR_NONE;
    }

    # 当会话的套接字状态为 LIBSSH2_SOCKET_CONNECTED 时，循环执行以下代码块
    while(session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        # 读取 SFTP 数据包
        rc = sftp_packet_read(sftp);
        if(rc < 0)
            return rc;

        /* data was read, check the queue again */
        # 如果读取到数据，再次检查数据包队列
        if(!sftp_packet_ask(sftp, packet_type, request_id, data, data_len)) {
            /* The right packet was available in the packet brigade */
            _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Got %d",
                           (int) packet_type);

            # 如果数据长度小于所需长度，返回缓冲区过小的错误
            if (*data_len < required_size) {
                return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
            }

            # 返回无错误
            return LIBSSH2_ERROR_NONE;
        }
    }

    # 只有在套接字断开时才会执行到这里
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
# 函数：sftp_packet_requirev
# 功能：要求 N 个可能的响应中的一个
# 参数：sftp - LIBSSH2_SFTP 对象
#       num_valid_responses - 有效响应的数量
#       valid_responses - 有效响应的数组
#       request_id - 请求 ID
#       data - 数据
#       data_len - 数据长度
#       required_size - 所需数据的大小
static int
sftp_packet_requirev(LIBSSH2_SFTP *sftp, int num_valid_responses,
                     const unsigned char *valid_responses,
                     uint32_t request_id, unsigned char **data,
                     size_t *data_len, size_t required_size)
{
    int i;
    int rc;

    if(data == NULL || data_len == NULL || required_size == 0) {
        return LIBSSH2_ERROR_BAD_USE;
    }

    # 如果没有超时活动，则启动一个新的超时
    if(sftp->requirev_start == 0)
        sftp->requirev_start = time(NULL);

    while(sftp->channel->session->socket_state == LIBSSH2_SOCKET_CONNECTED) {
        for(i = 0; i < num_valid_responses; i++) {
            if(sftp_packet_ask(sftp, valid_responses[i], request_id,
                                data, data_len) == 0) {
                '''
                 * 在所有返回之前设置为零，表示超时未激活
                 '''
                sftp->requirev_start = 0;

                if (*data_len < required_size) {
                    return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
                }

                return LIBSSH2_ERROR_NONE;
            }
        }

        rc = sftp_packet_read(sftp);
        if((rc < 0) && (rc != LIBSSH2_ERROR_EAGAIN)) {
            sftp->requirev_start = 0;
            return rc;
        }
        else if(rc <= 0) {
            # 防止忙等
            long left =
                LIBSSH2_READ_TIMEOUT -
                (long)(time(NULL) - sftp->requirev_start);

            if(left <= 0) {
                sftp->requirev_start = 0;
                return LIBSSH2_ERROR_TIMEOUT;
            }
            else if(rc == LIBSSH2_ERROR_EAGAIN) {
                return rc;
            }
        }
    }

    sftp->requirev_start = 0;

    # 只有在套接字断开时才会到达此处
    return LIBSSH2_ERROR_SOCKET_DISCONNECT;
}
/* sftp_attr2bin
 * Populate attributes into an SFTP block
 */
static ssize_t
sftp_attr2bin(unsigned char *p, const LIBSSH2_SFTP_ATTRIBUTES * attrs)
{
    // 指向字节流的指针
    unsigned char *s = p;
    // 属性标志位掩码
    uint32_t flag_mask =
        LIBSSH2_SFTP_ATTR_SIZE | LIBSSH2_SFTP_ATTR_UIDGID |
        LIBSSH2_SFTP_ATTR_PERMISSIONS | LIBSSH2_SFTP_ATTR_ACMODTIME;

    /* TODO: When we add SFTP4+ functionality flag_mask can get additional
       bits */

    // 如果属性为空
    if(!attrs) {
        // 将 0 存入字节流
        _libssh2_htonu32(s, 0);
        return 4;
    }

    // 存储属性标志位
    _libssh2_store_u32(&s, attrs->flags & flag_mask);

    // 如果包含文件大小属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_SIZE) {
        // 存储文件大小
        _libssh2_store_u64(&s, attrs->filesize);
    }

    // 如果包含用户 ID 和组 ID 属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_UIDGID) {
        // 存储用户 ID 和组 ID
        _libssh2_store_u32(&s, attrs->uid);
        _libssh2_store_u32(&s, attrs->gid);
    }

    // 如果包含权限属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) {
        // 存储权限
        _libssh2_store_u32(&s, attrs->permissions);
    }

    // 如果包含访问时间和修改时间属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_ACMODTIME) {
        // 存储访问时间和修改时间
        _libssh2_store_u32(&s, attrs->atime);
        _libssh2_store_u32(&s, attrs->mtime);
    }

    // 返回字节流中存储的字节数
    return (s - p);
}

/* sftp_bin2attr
 */
static int
sftp_bin2attr(LIBSSH2_SFTP_ATTRIBUTES *attrs, const unsigned char *p,
              size_t data_len)
{
    // 字符串缓冲区
    struct string_buf buf;
    // 属性标志位
    uint32_t flags = 0;
    buf.data = (unsigned char *)p;
    buf.dataptr = buf.data;
    buf.len = data_len;

    // 从缓冲区中获取属性标志位
    if(_libssh2_get_u32(&buf, &flags) != 0) {
        return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
    }
    attrs->flags = flags;

    // 如果包含文件大小属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_SIZE) {
        // 从缓冲区中获取文件大小
        if(_libssh2_get_u64(&buf, &(attrs->filesize)) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
    }

    // 如果包含用户 ID 和组 ID 属性
    if(attrs->flags & LIBSSH2_SFTP_ATTR_UIDGID) {
        // 获取用户 ID 和组 ID
        uint32_t uid = 0;
        uint32_t gid = 0;
        if(_libssh2_get_u32(&buf, &uid) != 0 ||
           _libssh2_get_u32(&buf, &gid) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        attrs->uid = uid;
        attrs->gid = gid;
    }
    # 检查属性中是否包含权限标志
    if(attrs->flags & LIBSSH2_SFTP_ATTR_PERMISSIONS) {
        # 定义变量存储权限值
        uint32_t permissions;
        # 从缓冲区中获取权限值，如果失败则返回错误
        if(_libssh2_get_u32(&buf, &permissions) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        # 将获取的权限值赋给属性对象
        attrs->permissions = permissions;
    }

    # 检查属性中是否包含访问时间和修改时间标志
    if(attrs->flags & LIBSSH2_SFTP_ATTR_ACMODTIME) {
        # 定义变量存储访问时间和修改时间
        uint32_t atime;
        uint32_t mtime;
        # 从缓冲区中获取访问时间和修改时间，如果失败则返回错误
        if(_libssh2_get_u32(&buf, &atime) != 0 ||
           _libssh2_get_u32(&buf, &mtime) != 0) {
            return LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        }
        # 将获取的访问时间和修改时间赋给属性对象
        attrs->atime = atime;
        attrs->mtime = mtime;
    }

    # 返回缓冲区中已处理的数据长度
    return (buf.dataptr - buf.data);
}
/* ************
 * SFTP API *
 ************ */

// 定义一个函数指针，用于在通道关闭时关闭 SFTP 流
LIBSSH2_CHANNEL_CLOSE_FUNC(libssh2_sftp_dtor);

/* libssh2_sftp_dtor
 * 当通道关闭时关闭 SFTP 流
 */
LIBSSH2_CHANNEL_CLOSE_FUNC(libssh2_sftp_dtor)
{
    // 将通道抽象类型转换为 SFTP 类型
    LIBSSH2_SFTP *sftp = (LIBSSH2_SFTP *) (*channel_abstract);

    // 忽略会话抽象和通道
    (void) session_abstract;
    (void) channel;

    /* 释放 sftp_packet_read 的部分数据包存储空间 */
    if(sftp->partial_packet) {
        LIBSSH2_FREE(session, sftp->partial_packet);
    }

    /* 释放 _libssh2_sftp_packet_readdir 的数据包存储空间 */
    if(sftp->readdir_packet) {
        LIBSSH2_FREE(session, sftp->readdir_packet);
    }

    // 释放 SFTP 结构体内存空间
    LIBSSH2_FREE(session, sftp);
}

/*
 * sftp_init
 *
 * 启动一个 SFTP 会话
 */
static LIBSSH2_SFTP *sftp_init(LIBSSH2_SESSION *session)
{
    unsigned char *data;
    size_t data_len;
    ssize_t rc;
    LIBSSH2_SFTP *sftp_handle;
    struct string_buf buf;
    unsigned char *endp;

    if(session->sftpInit_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Initializing SFTP subsystem");

        /*
         * session 结构体内的 'sftpInit_sftp' 和 'sftpInit_channel' 字段
         * 仅在设置阶段使用。一旦创建了 SFTP 会话，它们就会被清除，
         * 可以再次使用以允许每个会话有任意数量的 SFTP 句柄。
         *
         * 请注意，您绝对不应该尝试再次调用 libssh2_sftp_init() 来获取
         * 另一个句柄，直到前一个调用已经完成并且成功创建了一个句柄，
         * 或者失败并返回错误（不包括 *EAGAIN）。
         */

        assert(session->sftpInit_sftp == NULL);
        session->sftpInit_sftp = NULL;
        session->sftpInit_state = libssh2_NB_state_created;
    }

    sftp_handle = session->sftpInit_sftp;
    # 如果 SFTP 初始化状态为已创建，则执行以下操作
    if(session->sftpInit_state == libssh2_NB_state_created) {
        # 打开一个通道，用于 SFTP 初始化
        session->sftpInit_channel =
            _libssh2_channel_open(session, "session", sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL, 0);
        # 如果通道打开失败，则根据错误类型进行相应处理
        if(!session->sftpInit_channel) {
            # 如果是因为阻塞，则记录错误信息并返回空值
            if(libssh2_session_last_errno(session) == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            # 如果是其他错误，则记录错误信息，将 SFTP 初始化状态设为闲置，并返回空值
            else {
                _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                               "Unable to startup channel");
                session->sftpInit_state = libssh2_NB_state_idle;
            }
            return NULL;
        }

        # 将 SFTP 初始化状态设为已发送
        session->sftpInit_state = libssh2_NB_state_sent;
    }

    # 如果 SFTP 初始化状态为已发送，则执行以下操作
    if(session->sftpInit_state == libssh2_NB_state_sent) {
        # 处理 SFTP 通道的启动过程，请求 SFTP 子系统
        int ret = _libssh2_channel_process_startup(session->sftpInit_channel,
                                                   "subsystem",
                                                   sizeof("subsystem") - 1,
                                                   "sftp",
                                                   strlen("sftp"));
        # 如果是因为阻塞，则记录错误信息并返回空值
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block to request SFTP subsystem");
            return NULL;
        }
        # 如果是其他错误，则记录错误信息，并跳转到错误处理标签
        else if(ret) {
            _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                           "Unable to request SFTP subsystem");
            goto sftp_init_error;
        }

        # 将 SFTP 初始化状态设为已发送1
        session->sftpInit_state = libssh2_NB_state_sent1;
    }
    # 如果 SFTP 初始化状态为已发送1
    if(session->sftpInit_state == libssh2_NB_state_sent1) {
        # 请求通道扩展数据，忽略扩展数据
        rc = _libssh2_channel_extended_data(session->sftpInit_channel,
                                         LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE);
        # 如果返回值为需要再次尝试
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 设置错误信息并返回空
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting handle extended data");
            return NULL;
        }

        # 分配 SFTP 结构的内存空间
        sftp_handle =
            session->sftpInit_sftp =
            LIBSSH2_CALLOC(session, sizeof(LIBSSH2_SFTP));
        # 如果分配失败
        if(!sftp_handle) {
            # 设置错误信息并跳转到 sftp_init_error 标签
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a new SFTP structure");
            goto sftp_init_error;
        }
        # 设置 SFTP 结构的通道和请求 ID
        sftp_handle->channel = session->sftpInit_channel;
        sftp_handle->request_id = 0;

        # 将 SFTP 初始化缓冲区的前4个字节设置为5
        _libssh2_htonu32(session->sftpInit_buffer, 5);
        # 将 SFTP 初始化缓冲区的第5个字节设置为SSH_FXP_INIT
        session->sftpInit_buffer[4] = SSH_FXP_INIT;
        # 将 SFTP 初始化缓冲区的后4个字节设置为SFTP版本号
        _libssh2_htonu32(session->sftpInit_buffer + 5, LIBSSH2_SFTP_VERSION);
        # 设置已发送的字节数为0
        session->sftpInit_sent = 0; /* nothing's sent yet */

        # 输出 SFTP 调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Sending FXP_INIT packet advertising "
                       "version %d support",
                       (int) LIBSSH2_SFTP_VERSION);

        # 设置 SFTP 初始化状态为已发送2
        session->sftpInit_state = libssh2_NB_state_sent2;
    }
    # 如果 SFTP 初始化状态为已发送2，则执行以下操作
    if(session->sftpInit_state == libssh2_NB_state_sent2) {
        # 将剩余的初始化缓冲区内容发送出去
        rc = _libssh2_channel_write(session->sftpInit_channel, 0,
                                    session->sftpInit_buffer +
                                    session->sftpInit_sent,
                                    9 - session->sftpInit_sent);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次发送，返回空指针
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending SSH_FXP_INIT");
            return NULL;
        }
        # 如果返回值小于0，则表示发送失败，记录错误信息并跳转到错误处理标签
        else if(rc < 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send SSH_FXP_INIT");
            goto sftp_init_error;
        }
        # 如果发送成功，则累加已发送的字节数
        else {
            # 累加已发送的字节数
            session->sftpInit_sent += rc;

            # 如果已发送的字节数等于9，则将状态更新为已发送3
            if(session->sftpInit_sent == 9)
                # 继续下一步操作
                session->sftpInit_state = libssh2_NB_state_sent3;

            # 如果小于9，则保持当前状态以便稍后继续发送
        }
    }

    # 要求接收 SFTP 数据包，类型为 SSH_FXP_VERSION，超时时间为5
    rc = sftp_packet_require(sftp_handle, SSH_FXP_VERSION,
                             0, &data, &data_len, 5);
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次接收，返回空指针
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                       "Would block receiving SSH_FXP_VERSION");
        return NULL;
    }
    # 如果返回值为 LIBSSH2_ERROR_BUFFER_TOO_SMALL，则表示缓冲区太小，释放数据并记录错误信息，跳转到错误处理标签
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                       "Invalid SSH_FXP_VERSION response");
        goto sftp_init_error;
    }
    # 如果返回值不为0，则表示等待 SFTP 子系统的响应超时，记录错误信息并跳转到错误处理标签
    else if(rc) {
        _libssh2_error(session, rc,
                       "Timeout waiting for response from SFTP subsystem");
        goto sftp_init_error;
    }

    # 设置数据缓冲区的指针和长度
    buf.data = data;
    buf.dataptr = buf.data + 1;
    buf.len = data_len;
    endp = &buf.data[data_len];
    # 从缓冲区中获取一个无符号32位整数，并将其存储到sftp_handle->version中，如果失败则释放内存并返回错误码
    if(_libssh2_get_u32(&buf, &(sftp_handle->version)) != 0) {
        LIBSSH2_FREE(session, data);
        rc = LIBSSH2_ERROR_BUFFER_TOO_SMALL;
        goto sftp_init_error;
    }

    # 如果远程SFTP版本大于本地支持的版本，则将远程版本号截断为本地版本号
    if(sftp_handle->version > LIBSSH2_SFTP_VERSION) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Truncating remote SFTP version from %lu",
                       sftp_handle->version);
        sftp_handle->version = LIBSSH2_SFTP_VERSION;
    }
    # 启用SFTP版本兼容性，并打印调试信息
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                   "Enabling SFTP version %lu compatibility",
                   sftp_handle->version);
    # 循环处理扩展名和扩展数据
    while(buf.dataptr < endp) {
        unsigned char *extname, *extdata;

        # 从缓冲区中获取字符串类型的扩展名，如果失败则释放内存并返回错误码
        if(_libssh2_get_string(&buf, &extname, NULL)) {
            LIBSSH2_FREE(session, data);
            _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Data too short when extracting extname");
            goto sftp_init_error;
        }

        # 从缓冲区中获取字符串类型的扩展数据，如果失败则释放内存并返回错误码
        if(_libssh2_get_string(&buf, &extdata, NULL)) {
            LIBSSH2_FREE(session, data);
            _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Data too short when extracting extdata");
            goto sftp_init_error;
        }
    }
    # 释放内存
    LIBSSH2_FREE(session, data);

    # 设置当通道关闭时，SFTP服务也关闭
    sftp_handle->channel->abstract = sftp_handle;
    sftp_handle->channel->close_cb = libssh2_sftp_dtor;

    # 设置SFTP初始化状态为闲置
    session->sftpInit_state = libssh2_NB_state_idle;

    # 清空会话结构中的sftp和channel指针
    session->sftpInit_sftp = NULL;
    session->sftpInit_channel = NULL;

    # 初始化sftp_handle->sftp_handles链表
    _libssh2_list_init(&sftp_handle->sftp_handles);

    # 返回sftp_handle
    return sftp_handle;

  # 处理SFTP初始化错误
  sftp_init_error:
    # 释放sftpInit_channel，并等待直到成功释放为止
    while(_libssh2_channel_free(session->sftpInit_channel) ==
           LIBSSH2_ERROR_EAGAIN);
    session->sftpInit_channel = NULL;
    # 如果会话中的 sftpInit_sftp 不为空
    if(session->sftpInit_sftp) {
        # 释放 sftpInit_sftp 占用的内存
        LIBSSH2_FREE(session, session->sftpInit_sftp);
        # 将 sftpInit_sftp 置为 NULL
        session->sftpInit_sftp = NULL;
    }
    # 将 sftpInit_state 状态设置为 libssh2_NB_state_idle
    session->sftpInit_state = libssh2_NB_state_idle;
    # 返回空指针
    return NULL;
}

/*
 * libssh2_sftp_init
 *
 * Startup an SFTP session
 */
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session)
{
    LIBSSH2_SFTP *ptr;

    if(!session)
        return NULL;

    if(!(session->state & LIBSSH2_STATE_AUTHENTICATED)) {
        _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                       "session not authenticated yet");
        return NULL;
    }

    BLOCK_ADJUST_ERRNO(ptr, session, sftp_init(session));
    return ptr;
}

/*
 * sftp_shutdown
 *
 * Shuts down the SFTP subsystem
 */
static int
sftp_shutdown(LIBSSH2_SFTP *sftp)
{
    int rc;
    LIBSSH2_SESSION *session = sftp->channel->session;
    /*
     * Make sure all memory used in the state variables are free
     */
    if(sftp->partial_packet) {
        LIBSSH2_FREE(session, sftp->partial_packet);
        sftp->partial_packet = NULL;
    }
    if(sftp->open_packet) {
        LIBSSH2_FREE(session, sftp->open_packet);
        sftp->open_packet = NULL;
    }
    if(sftp->readdir_packet) {
        LIBSSH2_FREE(session, sftp->readdir_packet);
        sftp->readdir_packet = NULL;
    }
    if(sftp->fstat_packet) {
        LIBSSH2_FREE(session, sftp->fstat_packet);
        sftp->fstat_packet = NULL;
    }
    if(sftp->unlink_packet) {
        LIBSSH2_FREE(session, sftp->unlink_packet);
        sftp->unlink_packet = NULL;
    }
    if(sftp->rename_packet) {
        LIBSSH2_FREE(session, sftp->rename_packet);
        sftp->rename_packet = NULL;
    }
    if(sftp->fstatvfs_packet) {
        LIBSSH2_FREE(session, sftp->fstatvfs_packet);
        sftp->fstatvfs_packet = NULL;
    }
    if(sftp->statvfs_packet) {
        LIBSSH2_FREE(session, sftp->statvfs_packet);
        sftp->statvfs_packet = NULL;
    }
    if(sftp->mkdir_packet) {
        LIBSSH2_FREE(session, sftp->mkdir_packet);
        sftp->mkdir_packet = NULL;
    }
    if(sftp->rmdir_packet) {
        LIBSSH2_FREE(session, sftp->rmdir_packet);
        sftp->rmdir_packet = NULL;
    }
    # 如果存在 stat_packet，则释放其内存并将其置为 NULL
    if(sftp->stat_packet) {
        LIBSSH2_FREE(session, sftp->stat_packet);
        sftp->stat_packet = NULL;
    }
    # 如果存在 symlink_packet，则释放其内存并将其置为 NULL
    if(sftp->symlink_packet) {
        LIBSSH2_FREE(session, sftp->symlink_packet);
        sftp->symlink_packet = NULL;
    }
    # 如果存在 fsync_packet，则释放其内存并将其置为 NULL
    if(sftp->fsync_packet) {
        LIBSSH2_FREE(session, sftp->fsync_packet);
        sftp->fsync_packet = NULL;
    }

    # 刷新 SFTP 数据包
    sftp_packet_flush(sftp);

    # TODO: 我们应该考虑遍历 sftp_handles 列表并关闭任何剩余的 sftp 句柄...

    # 释放 SFTP 通道
    rc = _libssh2_channel_free(sftp->channel);

    # 返回释放结果
    return rc;
/* libssh2_sftp_shutdown
 * 关闭 SFTP 子系统
 */
LIBSSH2_API int
libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session, sftp_shutdown(sftp));
    return rc;
}

/* *******************************
 * SFTP 文件和目录操作 *
 ******************************* */

/* sftp_open
 * 打开 SFTP 文件
 */
static LIBSSH2_SFTP_HANDLE *
sftp_open(LIBSSH2_SFTP *sftp, const char *filename,
          size_t filename_len, uint32_t flags, long mode,
          int open_type)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_SFTP_HANDLE *fp;
    LIBSSH2_SFTP_ATTRIBUTES attrs = {
        LIBSSH2_SFTP_ATTR_PERMISSIONS, 0, 0, 0, 0, 0, 0
    };
    unsigned char *s;
    ssize_t rc;
    int open_file = (open_type == LIBSSH2_SFTP_OPENFILE)?1:0;
    # 如果 SFTP 的打开状态为闲置
    if(sftp->open_state == libssh2_NB_state_idle) {
        # 计算打开数据包的长度，包括文件名长度和其他固定长度
        sftp->open_packet_len = filename_len + 13 +
            (open_file? (4 +
                         sftp_attrsize(LIBSSH2_SFTP_ATTR_PERMISSIONS)) : 0);

        # 初始化已发送的打开数据包长度为0
        sftp->open_packet_sent = 0;
        # 分配内存用于存储打开数据包
        s = sftp->open_packet = LIBSSH2_ALLOC(session, sftp->open_packet_len);
        # 如果内存分配失败，则返回空指针
        if(!sftp->open_packet) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for FXP_OPEN or "
                           "FXP_OPENDIR packet");
            return NULL;
        }
        # 设置文件类型属性
        attrs.permissions = mode |
            (open_file ? LIBSSH2_SFTP_ATTR_PFILETYPE_FILE :
             LIBSSH2_SFTP_ATTR_PFILETYPE_DIR);

        # 存储数据包长度和请求ID
        _libssh2_store_u32(&s, sftp->open_packet_len - 4);
        *(s++) = open_file? SSH_FXP_OPEN : SSH_FXP_OPENDIR;
        sftp->open_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->open_request_id);
        _libssh2_store_str(&s, filename, filename_len);

        # 如果是打开文件，则存储标志和属性
        if(open_file) {
            _libssh2_store_u32(&s, flags);
            s += sftp_attr2bin(s, &attrs);
        }

        # 调试信息，发送打开请求
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Sending %s open request",
                       open_file? "file" : "directory");

        # 设置打开状态为已创建
        sftp->open_state = libssh2_NB_state_created;
    }
    # 如果 SFTP 的打开状态为已创建
    if(sftp->open_state == libssh2_NB_state_created) {
        # 向通道写入数据，发送 SFTP 的打开数据包
        rc = _libssh2_channel_write(channel, 0, sftp->open_packet+
                                    sftp->open_packet_sent,
                                    sftp->open_packet_len -
                                    sftp->open_packet_sent);
        # 如果返回值为需要再次尝试，则返回错误信息并返回空值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending FXP_OPEN or "
                           "FXP_OPENDIR command");
            return NULL;
        }
        # 如果返回值小于 0，则返回错误信息并释放资源后返回空值
        else if(rc < 0) {
            _libssh2_error(session, rc, "Unable to send FXP_OPEN*");
            LIBSSH2_FREE(session, sftp->open_packet);
            sftp->open_packet = NULL;
            sftp->open_state = libssh2_NB_state_idle;
            return NULL;
        }

        # 增加已发送数据的计数器，并保持在当前状态直到所有数据发送完成
        sftp->open_packet_sent += rc;

        # 如果已发送数据长度等于数据包长度，则释放资源并更新状态为已发送
        if(sftp->open_packet_len == sftp->open_packet_sent) {
            LIBSSH2_FREE(session, sftp->open_packet);
            sftp->open_packet = NULL;
            sftp->open_state = libssh2_NB_state_sent;
        }
    }

    # 返回空值
    return NULL;
/* libssh2_sftp_open_ex
 */
// 定义 libssh2_sftp_open_ex 函数，用于在 SFTP 中打开文件
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_open_ex(LIBSSH2_SFTP *sftp, const char *filename,
                     unsigned int filename_len, unsigned long flags, long mode,
                     int open_type)
{
    LIBSSH2_SFTP_HANDLE *hnd; // 声明一个 SFTP 文件句柄指针

    if(!sftp) // 如果 SFTP 对象为空，则返回空指针
        return NULL;

    // 调整错误码，调用 sftp_open 函数打开文件
    BLOCK_ADJUST_ERRNO(hnd, sftp->channel->session,
                       sftp_open(sftp, filename, filename_len, flags, mode,
                                 open_type));
    return hnd; // 返回文件句柄指针
}

/*
 * sftp_read
 *
 * 从 SFTP 文件句柄中读取数据
 *
 */
// 定义 sftp_read 函数，用于从 SFTP 文件句柄中读取数据
static ssize_t sftp_read(LIBSSH2_SFTP_HANDLE * handle, char *buffer,
                         size_t buffer_size)
{
    LIBSSH2_SFTP *sftp = handle->sftp; // 获取 SFTP 对象指针
    LIBSSH2_CHANNEL *channel = sftp->channel; // 获取 SFTP 对应的通道指针
    LIBSSH2_SESSION *session = channel->session; // 获取通道对应的会话指针
    size_t count = 0; // 初始化计数器
    struct sftp_pipeline_chunk *chunk; // 声明 SFTP 管道块指针
    struct sftp_pipeline_chunk *next; // 声明下一个 SFTP 管道块指针
    ssize_t rc; // 声明返回值
    struct _libssh2_sftp_handle_file_data *filep =
        &handle->u.file; // 获取文件句柄对应的文件数据指针
    size_t bytes_in_buffer = 0; // 初始化缓冲区中的字节数
    char *sliding_bufferp = buffer; // 初始化滑动缓冲区指针
    /* 这个函数可能在三个不同的地方被中断，需要等待来自网络的数据。它返回 EAGAIN，以允许非阻塞客户端进行其他工作，但是这些客户端预计会再次调用这个函数（可能多次）来完成操作。

       麻烦的地方在于，如果之前由于 EAGAIN 中断了 sftp_read，我们必须在相同的位置继续以完成先前中断的操作。这是通过使用状态机记录我们所处的执行阶段来完成的。状态存储在 sftp->read_state 中。

       libssh2_NB_state_idle: 第一个阶段是我们准备多个 FXP_READ 数据包以进行乐观的预读。我们在第二阶段尽可能多地发送它们，而不必等待每个响应；这是快速读取的关键。但是我们可能需要调整通道窗口大小来做到这一点，这可能会在等待时中断这个函数。状态机将阶段保存为 libssh2_NB_state_idle，因此在下一次调用时会返回到这里。

       libssh2_NB_state_sent: 第二阶段是我们发送 FXP_READ 数据包的地方。将它们写入通道可能会被 EAGAIN 中断，但状态机确保我们在下一次调用时跳过第一阶段并继续发送。

       libssh2_NB_state_sent2: 在第三阶段（由...表示）中，我们从到目前为止已经到达的响应中读取数据。读取可能会被 EAGAIN 中断，但状态机确保我们在下一次调用时跳过第一和第二阶段并继续发送。
    */

    switch(sftp->read_state) {
    # 根据状态进行相应操作
    case libssh2_NB_state_sent:
        # 将 SFTP 读取状态设置为闲置
        sftp->read_state = libssh2_NB_state_idle;

        # 移动到尚未发送的 READ 数据包并尽可能发送
        chunk = _libssh2_list_first(&handle->packet_list);

        while(chunk) {
            if(chunk->lefttosend) {
                # 使用非阻塞方式向通道写入数据
                rc = _libssh2_channel_write(channel, 0, &chunk->packet[chunk->sent], chunk->lefttosend);
                if(rc < 0) {
                    # 如果写入失败，将 SFTP 读取状态设置为已发送，并返回错误码
                    sftp->read_state = libssh2_NB_state_sent;
                    return rc;
                }

                # 记录下次继续发送的位置
                chunk->lefttosend -= rc;
                chunk->sent += rc;

                if(chunk->lefttosend) {
                    # 如果仍有数据需要发送，检查是否有完全发送的数据包，如果有则跳出循环开始读取
                    if(chunk != _libssh2_list_first(&handle->packet_list)) {
                        break;
                    }
                    else {
                        continue;
                    }
                }
            }

            # 移动到下一个有数据需要发送的数据包
            chunk = _libssh2_list_next(&chunk->node);
        }
        # 继续执行下面的代码

    default:
        # 断言，如果程序执行到这里，说明状态机出错，状态未被识别
        assert(!"State machine error; unrecognised read state");
    }

    # 程序不应该执行到这里，如果执行到这里，返回 SFTP 读取内部错误
    return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL, "sftp_read() internal error");
/* libssh2_sftp_read
 * 从 SFTP 文件句柄中读取数据
 */
LIBSSH2_API ssize_t
libssh2_sftp_read(LIBSSH2_SFTP_HANDLE *hnd, char *buffer,
                  size_t buffer_maxlen)
{
    ssize_t rc;
    // 如果句柄为空，则返回错误码
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    // 调整阻塞状态，读取 SFTP 文件句柄中的数据
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_read(hnd, buffer, buffer_maxlen));
    return rc;
}

/* sftp_readdir
 * 从 SFTP 目录句柄中读取数据
 */
static ssize_t sftp_readdir(LIBSSH2_SFTP_HANDLE *handle, char *buffer,
                            size_t buffer_maxlen, char *longentry,
                            size_t longentry_maxlen,
                            LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    uint32_t num_names;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len = handle->handle_len + 13;
    unsigned char *s, *data;
    static const unsigned char read_responses[2] = {
        SSH_FXP_NAME, SSH_FXP_STATUS };
    ssize_t retcode;

    // 这里需要添加对 sftp_readdir 函数的具体注释
}
    # 如果读取目录状态为已创建
    if(sftp->readdir_state == libssh2_NB_state_created) {
        # 输出调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Reading entries from directory handle");
        # 将读取到的目录内容写入通道
        retcode = _libssh2_channel_write(channel, 0, sftp->readdir_packet, packet_len);
        # 如果返回值为需要再次尝试，则返回该值
        if(retcode == LIBSSH2_ERROR_EAGAIN) {
            return retcode;
        }
        # 如果返回的字节数不等于期望的字节数
        else if((ssize_t)packet_len != retcode) {
            # 释放内存
            LIBSSH2_FREE(session, sftp->readdir_packet);
            sftp->readdir_packet = NULL;
            sftp->readdir_state = libssh2_NB_state_idle;
            # 返回错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND, "_libssh2_channel_write() failed");
        }

        # 释放内存
        LIBSSH2_FREE(session, sftp->readdir_packet);
        sftp->readdir_packet = NULL;

        # 设置读取目录状态为已发送
        sftp->readdir_state = libssh2_NB_state_sent;
    }

    # 要求SFTP数据包，读取响应
    retcode = sftp_packet_requirev(sftp, 2, read_responses, sftp->readdir_request_id, &data, &data_len, 9);
    # 如果返回值为需要再次尝试，则返回该值
    if(retcode == LIBSSH2_ERROR_EAGAIN)
        return retcode;
    # 如果返回值为缓冲区太小
    else if(retcode == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于0，则释放内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回SFTP协议错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL, "Status message too short");
    }
    # 如果返回值为其他错误
    else if(retcode) {
        # 设置读取目录状态为空闲
        sftp->readdir_state = libssh2_NB_state_idle;
        # 返回超时等待状态消息的错误信息
        return _libssh2_error(session, retcode, "Timeout waiting for status message");
    }
    # 如果收到的数据的第一个字节是 SSH_FXP_STATUS
    if(data[0] == SSH_FXP_STATUS) {
        # 从数据中解析出返回码
        retcode = _libssh2_ntohu32(data + 5);
        # 释放内存
        LIBSSH2_FREE(session, data);
        # 如果返回码是 LIBSSH2_FX_EOF
        if(retcode == LIBSSH2_FX_EOF) {
            # 设置状态为闲置，表示读取目录结束
            sftp->readdir_state = libssh2_NB_state_idle;
            return 0;
        }
        # 如果返回码不是 LIBSSH2_FX_EOF
        else {
            # 记录返回码到 sftp 结构体
            sftp->last_errno = retcode;
            # 设置状态为闲置
            sftp->readdir_state = libssh2_NB_state_idle;
            # 返回 SFTP 协议错误
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    # 设置状态为闲置
    sftp->readdir_state = libssh2_NB_state_idle;

    # 解析出返回的文件名数量
    num_names = _libssh2_ntohu32(data + 5);
    # 调试信息，记录返回的文件名数量
    _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%lu entries returned",
                   num_names);
    # 如果没有返回文件名
    if(!num_names) {
        # 释放内存
        LIBSSH2_FREE(session, data);
        return 0;
    }

    # 设置目录句柄的剩余文件名数量
    handle->u.dir.names_left = num_names;
    # 记录返回的数据包
    handle->u.dir.names_packet = data;
    # 设置下一个文件名的指针
    handle->u.dir.next_name = (char *) data + 9;
    # 记录返回的数据包长度
    handle->u.dir.names_packet_len = data_len - 9;

    # 使用从函数开始处的名称弹出机制
    return sftp_readdir(handle, buffer, buffer_maxlen, longentry,
                        longentry_maxlen, attrs);
/* libssh2_sftp_readdir_ex
 * 从 SFTP 目录句柄中读取数据
 */
LIBSSH2_API int
libssh2_sftp_readdir_ex(LIBSSH2_SFTP_HANDLE *hnd, char *buffer,
                        size_t buffer_maxlen, char *longentry,
                        size_t longentry_maxlen,
                        LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    int rc;
    // 如果句柄为空，则返回错误码
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    // 调整块大小并调用 sftp_readdir 函数
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_readdir(hnd, buffer, buffer_maxlen, longentry,
                              longentry_maxlen, attrs));
    return rc;
}

/*
 * sftp_write
 *
 * 向 SFTP 句柄写入数据。返回已写入的字节数，或者负数的错误码。
 *
 * 我们建议将非常大的数据缓冲区发送到这个函数！
 *
 * 概念：
 *
 * - 通过检查传出块的链表，检测之前调用中已发送的给定缓冲区的数据量。确保跳过已处理的数据。
 *
 * - 将所有（新的）传出数据分割成不超过 N 的块。
 *
 * - 每个 N 字节的块被创建为一个单独的 SFTP 数据包。
 *
 * - 将所有创建的传出数据包添加到链表中。
 *
 * - 遍历列表并发送尚未发送的块，尽可能多地发送直到出现 EAGAIN。一些块可能已经在之前的调用中放入列表中。
 *
 * - 对于链表中已完全发送的所有块，检查 ACK。如果一个块已经被 ACK，它将从链表中移除，并且“acked”计数器将增加相应的数据量。
 *
 * - 返回到目前为止已确认的总字节数。
 *
 * 注意事项：
 * - 要小心：我们不能返回比给定的数字更高的数字！
 *
 * 待办事项：
 *   引入一个选项，禁用这种“推测性”预先写入，因为这可能会对某些应用程序造成伤害。
 */
static ssize_t sftp_write(LIBSSH2_SFTP_HANDLE *handle, const char *buffer,
                          size_t count)
{
    // 获取 SFTP 对象
    LIBSSH2_SFTP *sftp = handle->sftp;
    // 获取 SFTP 对象所属的通道
    LIBSSH2_CHANNEL *channel = sftp->channel;
    // 获取 SFTP 对象所属的会话
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    uint32_t retcode;
    uint32_t packet_len;
    unsigned char *s, *data;
    ssize_t rc;
    struct sftp_pipeline_chunk *chunk;
    struct sftp_pipeline_chunk *next;
    size_t acked = 0;
    size_t org_count = count;
    size_t already;

    switch(sftp->write_state) {
    default:
    }

    /* if there were acked data in a previous call that wasn't returned then,
       add that up and try to return it all now. This can happen if the app
       first sends a huge buffer of data, and then in a second call it sends a
       smaller one. */
    // 如果在之前的调用中有已确认的数据没有返回，则将其累加并尝试现在返回所有数据。这可能发生在应用程序首先发送了大量数据缓冲区，然后在第二次调用中发送了较小的数据。
    acked += handle->u.file.acked;

    if(acked) {
        ssize_t ret = MIN(acked, org_count);
        /* we got data acked so return that amount, but no more than what
           was asked to get sent! */

        /* store the remainder. 'ret' is always equal to or less than 'acked'
           here */
        // 我们已经确认了数据，所以返回相应数量的数据，但不超过要发送的数量！
        handle->u.file.acked = acked - ret;

        return ret;
    }

    else
        return 0; /* nothing was acked, and no EAGAIN was received! */
    // 没有确认的数据，并且没有收到 EAGAIN！
}

/* libssh2_sftp_write
 * Write data to a file handle
 */
// 将数据写入文件句柄
LIBSSH2_API ssize_t
libssh2_sftp_write(LIBSSH2_SFTP_HANDLE *hnd, const char *buffer,
                   size_t count)
{
    ssize_t rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_write(hnd, buffer, count));
    return rc;

}

static int sftp_fsync(LIBSSH2_SFTP_HANDLE *handle)
{
    // 获取 SFTP 对象
    LIBSSH2_SFTP *sftp = handle->sftp;
    // 获取 SFTP 对象所属的通道
    LIBSSH2_CHANNEL *channel = sftp->channel;
    // 获取 SFTP 对象所属的会话
    LIBSSH2_SESSION *session = channel->session;
    /* 34 = packet_len(4) + packet_type(1) + request_id(4) +
       string_len(4) + strlen("fsync@openssh.com")(17) + handle_len(4) */
    // 计算数据包长度
    // 计算数据包长度，包括句柄长度和固定长度 34
    uint32_t packet_len = handle->handle_len + 34;
    size_t data_len;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    uint32_t retcode;

    // 如果当前状态为 idle，则发出 fsync 命令
    if(sftp->fsync_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Issuing fsync command");
        // 分配内存给数据包
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        // 如果分配失败，则返回错误
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_EXTENDED "
                                  "packet");
        }

        // 将数据包长度写入数据包
        _libssh2_store_u32(&s, packet_len - 4);
        // 写入 SSH_FXP_EXTENDED 类型
        *(s++) = SSH_FXP_EXTENDED;
        // 为 fsync 命令分配请求 ID
        sftp->fsync_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->fsync_request_id);
        // 写入命令字符串 "fsync@openssh.com"
        _libssh2_store_str(&s, "fsync@openssh.com", 17);
        // 写入句柄数据
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        // 设置状态为 created
        sftp->fsync_state = libssh2_NB_state_created;
    }
    else {
        // 如果状态不为 idle，则使用之前创建的数据包
        packet = sftp->fsync_packet;
    }

    // 如果状态为 created，则将数据包写入通道
    if(sftp->fsync_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        // 如果返回值为 EAGAIN 或者小于数据包长度，则将数据包保存，返回 EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN ||
            (0 <= rc && rc < (ssize_t)packet_len)) {
            sftp->fsync_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }

        // 释放数据包内存
        LIBSSH2_FREE(session, packet);
        sftp->fsync_packet = NULL;

        // 如果写入失败，则设置状态为 idle，返回错误
        if(rc < 0) {
            sftp->fsync_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        // 设置状态为 sent
        sftp->fsync_state = libssh2_NB_state_sent;
    }

    // 等待接收服务器返回的状态数据包
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->fsync_request_id, &data, &data_len, 9);
    // 如果返回值为 EAGAIN，则继续等待
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果返回的错误码是缓冲区太小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于0
        if(data_len > 0) {
            # 释放之前分配的数据内存
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 协议错误，说明 fsync 包太短
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fsync packet too short");
    }
    # 如果返回的错误码不为0
    else if(rc) {
        # 将 fsync 状态设置为闲置
        sftp->fsync_state = libssh2_NB_state_idle;
        # 返回等待 FXP EXTENDED REPLY 时出错的错误信息
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    # 将 fsync 状态设置为闲置
    sftp->fsync_state = libssh2_NB_state_idle;

    # 获取数据中的返回码
    retcode = _libssh2_ntohu32(data + 5);
    # 释放之前分配的数据内存
    LIBSSH2_FREE(session, data);

    # 如果返回码不是 LIBSSH2_FX_OK
    if(retcode != LIBSSH2_FX_OK) {
        # 将 sftp 对象的最后一个错误码设置为返回码
        sftp->last_errno = retcode;
        # 返回 fsync 失败的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "fsync failed");
    }

    # 返回0，表示没有错误
    return 0;
/* libssh2_sftp_fsync
 * Commit data on the handle to disk.
 */
LIBSSH2_API int
libssh2_sftp_fsync(LIBSSH2_SFTP_HANDLE *hnd)
{
    int rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_fsync(hnd));
    return rc;
}
/* 
 * Commit data on the handle to disk.
 * 
 * This function is used to commit data on the handle to disk.
 * It takes a handle as input and returns an integer as output.
 * If the handle is null, it returns an error code indicating bad use.
 * It then adjusts the block and returns the result.
 */

/*
 * sftp_fstat
 *
 * Get or Set stat on a file
 */
static int sftp_fstat(LIBSSH2_SFTP_HANDLE *handle,
                      LIBSSH2_SFTP_ATTRIBUTES *attrs, int setstat)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len =
        handle->handle_len + 13 + (setstat ? sftp_attrsize(attrs->flags) : 0);
    unsigned char *s, *data;
    static const unsigned char fstat_responses[2] =
        { SSH_FXP_ATTRS, SSH_FXP_STATUS };
    ssize_t rc;

    if(sftp->fstat_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Issuing %s command",
                       setstat ? "set-stat" : "stat");
        s = sftp->fstat_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->fstat_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "FSTAT/FSETSTAT packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = setstat ? SSH_FXP_FSETSTAT : SSH_FXP_FSTAT;
        sftp->fstat_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->fstat_request_id);
        _libssh2_store_str(&s, handle->handle, handle->handle_len);

        if(setstat) {
            s += sftp_attr2bin(s, attrs);
        }

        sftp->fstat_state = libssh2_NB_state_created;
    }
}
/*
 * Get or Set stat on a file
 * 
 * This function is used to get or set stat on a file.
 * It takes a handle, attributes, and setstat as input and returns an integer as output.
 * It initializes various variables and then checks the state of fstat.
 * If the state is idle, it issues the command and allocates memory for the FSTAT/FSETSTAT packet.
 * It then stores the necessary information in the packet and updates the state.
 */
    # 如果 SFTP 的 fstat_state 状态为创建状态
    if(sftp->fstat_state == libssh2_NB_state_created) {
        # 向通道写入 fstat_packet 数据
        rc = _libssh2_channel_write(channel, 0, sftp->fstat_packet,
                                    packet_len);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果写入的数据长度不等于返回的长度
        else if((ssize_t)packet_len != rc) {
            # 释放 fstat_packet 的内存
            LIBSSH2_FREE(session, sftp->fstat_packet);
            sftp->fstat_packet = NULL;
            # 将 fstat_state 状态设置为闲置状态
            sftp->fstat_state = libssh2_NB_state_idle;
            # 返回发送 FXP_FSETSTAT 或 FXP_FSTAT 命令失败的错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  (setstat ? "Unable to send FXP_FSETSTAT"
                                   : "Unable to send FXP_FSTAT command"));
        }
        # 释放 fstat_packet 的内存
        LIBSSH2_FREE(session, sftp->fstat_packet);
        sftp->fstat_packet = NULL;
        # 将 fstat_state 状态设置为已发送状态
        sftp->fstat_state = libssh2_NB_state_sent;
    }

    # 要求接收 SFTP 的响应数据
    rc = sftp_packet_requirev(sftp, 2, fstat_responses,
                              sftp->fstat_request_id, &data,
                              &data_len, 9);
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;
    # 如果返回值为 LIBSSH2_ERROR_BUFFER_TOO_SMALL，则表示接收到的数据长度过小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于 0，则释放数据内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP fstat 数据包过短的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fstat packet too short");
    }
    # 如果返回值不为 0，则表示出现错误
    else if(rc) {
        # 将 fstat_state 状态设置为闲置状态
        sftp->fstat_state = libssh2_NB_state_idle;
        # 返回等待状态消息超时的错误信息
        return _libssh2_error(session, rc,
                              "Timeout waiting for status message");
    }

    # 将 fstat_state 状态设置为闲置状态
    sftp->fstat_state = libssh2_NB_state_idle;

    # 如果接收到的数据第一个字节为 SSH_FXP_STATUS
    if(data[0] == SSH_FXP_STATUS) {
        uint32_t retcode;
        # 从数据中获取返回码
        retcode = _libssh2_ntohu32(data + 5);
        # 释放数据内存
        LIBSSH2_FREE(session, data);
        # 如果返回码为 LIBSSH2_FX_OK，则表示操作成功
        if(retcode == LIBSSH2_FX_OK) {
            return 0;
        }
        # 否则，将返回码保存到 sftp->last_errno 中，并返回 SFTP 协议错误信息
        else {
            sftp->last_errno = retcode;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }
    # 如果将二进制数据转换为属性时出错，返回错误
    if(sftp_bin2attr(attrs, data + 5, data_len - 5) < 0) {
        # 释放内存
        LIBSSH2_FREE(session, data);
        # 返回 SFTP 协议中属性过短的错误
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Attributes too short in SFTP fstat");
    }

    # 释放内存
    LIBSSH2_FREE(session, data);

    # 返回成功
    return 0;
}
/* libssh2_sftp_fstat_ex
 * 获取或设置文件的状态
 */
LIBSSH2_API int
libssh2_sftp_fstat_ex(LIBSSH2_SFTP_HANDLE *hnd,
                      LIBSSH2_SFTP_ATTRIBUTES *attrs, int setstat)
{
    int rc;
    if(!hnd || !attrs)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_fstat(hnd, attrs, setstat));
    return rc;
}

/* libssh2_sftp_seek64
 * 将读/写指针设置到文件中的任意位置
 */
LIBSSH2_API void
libssh2_sftp_seek64(LIBSSH2_SFTP_HANDLE *handle, libssh2_uint64_t offset)
{
    if(!handle)
        return;
    if(handle->u.file.offset == offset && handle->u.file.offset_sent == offset)
        return;

    handle->u.file.offset = handle->u.file.offset_sent = offset;
    /* 丢弃所有挂起的请求和当前读取的数据 */
    sftp_packetlist_flush(handle);

    /* 释放剩余的接收缓冲数据 */
    if(handle->u.file.data_left) {
        LIBSSH2_FREE(handle->sftp->channel->session, handle->u.file.data);
        handle->u.file.data_left = handle->u.file.data_len = 0;
        handle->u.file.data = NULL;
    }

    /* 重置 EOF 为 False */
    handle->u.file.eof = FALSE;
}

/* libssh2_sftp_seek
 * 将读/写指针设置到文件中的任意位置
 */
LIBSSH2_API void
libssh2_sftp_seek(LIBSSH2_SFTP_HANDLE *handle, size_t offset)
{
    libssh2_sftp_seek64(handle, (libssh2_uint64_t)offset);
}

/* libssh2_sftp_tell
 * 返回当前读/写指针的偏移量
 */
LIBSSH2_API size_t
libssh2_sftp_tell(LIBSSH2_SFTP_HANDLE *handle)
{
    if(!handle)
        return 0; /* 没有句柄，没有大小 */

    /* 注意：如果大小大于 size_t 可以容纳的大小，这可能会截断大小，所以你真的应该使用 libssh2_sftp_tell64() 函数 */
    return (size_t)(handle->u.file.offset);
}

/* libssh2_sftp_tell64
 * 返回当前读/写指针的偏移量
 */
LIBSSH2_API libssh2_uint64_t
# 返回当前 SFTP 文件句柄的偏移量
libssh2_sftp_tell64(LIBSSH2_SFTP_HANDLE *handle)
{
    # 如果句柄为空，则返回 0
    if(!handle)
        return 0; /* no handle, no size */

    # 返回句柄中文件偏移量
    return handle->u.file.offset;
}

/*
 * 刷新所有剩余的传入 SFTP 数据包和僵尸请求
 */
static void sftp_packet_flush(LIBSSH2_SFTP *sftp)
{
    # 获取 SFTP 对象所关联的通道和会话
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    # 获取 SFTP 对象中的第一个数据包和僵尸请求
    LIBSSH2_SFTP_PACKET *packet = _libssh2_list_first(&sftp->packets);
    struct sftp_zombie_requests *zombie =
        _libssh2_list_first(&sftp->zombie_requests);

    # 循环处理所有数据包
    while(packet) {
        LIBSSH2_SFTP_PACKET *next;

        /* 检查列表中的下一个结构 */
        next =  _libssh2_list_next(&packet->node);
        _libssh2_list_remove(&packet->node);
        LIBSSH2_FREE(session, packet->data);
        LIBSSH2_FREE(session, packet);

        packet = next;
    }

    # 循环处理所有僵尸请求
    while(zombie) {
        /* 找到下一个节点 */
        struct sftp_zombie_requests *next = _libssh2_list_next(&zombie->node);
        /* 取消链接当前节点 */
        _libssh2_list_remove(&zombie->node);
        /* 释放内存 */
        LIBSSH2_FREE(session, zombie);
        zombie = next;
    }

}

/* sftp_close_handle
 *
 * 关闭文件或目录句柄
 * 同时释放句柄资源并从 SFTP 结构中取消链接
 * 在此函数返回后，句柄将不再可用，除非返回值为 LIBSSH2_ERROR_EAGAIN，在这种情况下应再次调用此函数
 */
static int
sftp_close_handle(LIBSSH2_SFTP_HANDLE *handle)
{
    # 获取句柄所属的 SFTP 对象、通道和会话
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + handle_len(4) */
    uint32_t packet_len = handle->handle_len + 13;
    unsigned char *s, *data = NULL;
    int rc = 0;
    # 如果句柄的关闭状态为闲置
    if(handle->close_state == libssh2_NB_state_idle) {
        # 输出调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Closing handle");
        # 分配内存给关闭数据包
        s = handle->close_packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果分配内存失败
        if(!handle->close_packet) {
            # 将关闭状态设置为闲置
            handle->close_state = libssh2_NB_state_idle;
            # 返回内存分配错误
            rc = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for FXP_CLOSE "
                                "packet");
        }
        else {
            # 存储数据包长度
            _libssh2_store_u32(&s, packet_len - 4);
            # 存储 SSH_FXP_CLOSE 类型
            *(s++) = SSH_FXP_CLOSE;
            # 存储关闭请求的 ID
            handle->close_request_id = sftp->request_id++;
            _libssh2_store_u32(&s, handle->close_request_id);
            # 存储句柄的数据
            _libssh2_store_str(&s, handle->handle, handle->handle_len);
            # 将关闭状态设置为已创建
            handle->close_state = libssh2_NB_state_created;
        }
    }

    # 如果句柄的关闭状态为已创建
    if(handle->close_state == libssh2_NB_state_created) {
        # 写入关闭数据包到通道
        rc = _libssh2_channel_write(channel, 0, handle->close_packet,
                                    packet_len);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回错误码
            return rc;
        }
        # 如果写入长度不等于数据包长度
        else if((ssize_t)packet_len != rc) {
            # 将关闭状态设置为闲置
            handle->close_state = libssh2_NB_state_idle;
            # 返回发送关闭命令失败的错误
            rc = _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                "Unable to send FXP_CLOSE command");
        }
        else
            # 将关闭状态设置为已发送
            handle->close_state = libssh2_NB_state_sent;

        # 释放关闭数据包的内存
        LIBSSH2_FREE(session, handle->close_packet);
        handle->close_packet = NULL;
    }
    # 如果关闭状态为已发送
    if(handle->close_state == libssh2_NB_state_sent) {
        # 要求接收 SSH_FXP_STATUS 类型的数据包，等待关闭请求的响应
        rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                                 handle->close_request_id, &data,
                                 &data_len, 9);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值为 LIBSSH2_ERROR_BUFFER_TOO_SMALL，表示数据包过小
        else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
            # 如果数据长度大于 0，释放数据内存
            if(data_len > 0) {
                LIBSSH2_FREE(session, data);
            }
            # 将 data 置为 NULL
            data = NULL;
            # 设置 SFTP 协议错误信息
            _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                           "Packet too short in FXP_CLOSE command");
        }
        # 如果返回值不为 0，设置错误信息
        else if(rc) {
            _libssh2_error(session, rc,
                           "Error waiting for status message");
        }

        # 设置关闭状态为已发送1
        handle->close_state = libssh2_NB_state_sent1;
    }

    # 如果 data 为空
    if(!data) {
        # 如果在此处 data 未设置，说明发生了意外情况，应该设置错误代码
        assert(rc);

    }
    else {
        # 获取返回码
        int retcode = _libssh2_ntohu32(data + 5);
        # 释放 data 内存
        LIBSSH2_FREE(session, data);

        # 如果返回码不为 LIBSSH2_FX_OK，设置 SFTP 错误码，并设置关闭状态为闲置
        if(retcode != LIBSSH2_FX_OK) {
            sftp->last_errno = retcode;
            handle->close_state = libssh2_NB_state_idle;
            rc = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                "SFTP Protocol Error");
        }
    }

    # 从父节点的列表中移除该句柄
    _libssh2_list_remove(&handle->node);

    # 如果句柄类型为 LIBSSH2_SFTP_HANDLE_DIR
    if(handle->handle_type == LIBSSH2_SFTP_HANDLE_DIR) {
        # 如果目录中还有剩余文件名，释放文件名数据包内存
        if(handle->u.dir.names_left)
            LIBSSH2_FREE(session, handle->u.dir.names_packet);
    }
    # 如果句柄类型为 LIBSSH2_SFTP_HANDLE_FILE
    else if(handle->handle_type == LIBSSH2_SFTP_HANDLE_FILE) {
        # 如果文件数据存在，释放文件数据内存
        if(handle->u.file.data)
            LIBSSH2_FREE(session, handle->u.file.data);
    }

    # 清空句柄的数据包列表
    sftp_packetlist_flush(handle);
    # 设置 SFTP 读取状态为闲置
    sftp->read_state = libssh2_NB_state_idle;

    # 设置关闭状态为闲置
    handle->close_state = libssh2_NB_state_idle;

    # 释放句柄内存
    LIBSSH2_FREE(session, handle);

    # 返回结果代码
    return rc;
/* libssh2_sftp_close_handle
 *
 * Close a file or directory handle
 * Also frees handle resource and unlinks it from the SFTP structure
 */
LIBSSH2_API int
libssh2_sftp_close_handle(LIBSSH2_SFTP_HANDLE *hnd)
{
    int rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session, sftp_close_handle(hnd));
    return rc;
}
/* sftp_unlink
 * Delete a file from the remote server
 */
static int sftp_unlink(LIBSSH2_SFTP *sftp, const char *filename,
                       size_t filename_len)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + filename_len(4) */
    uint32_t packet_len = filename_len + 13;
    unsigned char *s, *data;
    int rc;

    if(sftp->unlink_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Unlinking %s", filename);
        s = sftp->unlink_packet = LIBSSH2_ALLOC(session, packet_len);
        if(!sftp->unlink_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_REMOVE "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_REMOVE;
        sftp->unlink_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->unlink_request_id);
        _libssh2_store_str(&s, filename, filename_len);
        sftp->unlink_state = libssh2_NB_state_created;
    }
}
    # 如果 SFTP 对象的删除状态为已创建
    if(sftp->unlink_state == libssh2_NB_state_created) {
        # 向通道写入数据，发送删除文件的命令
        rc = _libssh2_channel_write(channel, 0, sftp->unlink_packet,
                                    packet_len);
        # 如果返回值为需要再次调用，则直接返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果发送的数据长度与返回的长度不一致
        else if((ssize_t)packet_len != rc) {
            # 释放内存，重置删除命令数据和状态
            LIBSSH2_FREE(session, sftp->unlink_packet);
            sftp->unlink_packet = NULL;
            sftp->unlink_state = libssh2_NB_state_idle;
            # 返回发送删除命令失败的错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_REMOVE command");
        }
        # 释放内存，重置删除命令数据
        LIBSSH2_FREE(session, sftp->unlink_packet);
        sftp->unlink_packet = NULL;

        # 设置删除状态为已发送
        sftp->unlink_state = libssh2_NB_state_sent;
    }

    # 等待接收删除文件的状态响应
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->unlink_request_id, &data,
                             &data_len, 9);
    # 如果返回值为需要再次调用，则直接返回该值
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果返回值为缓冲区太小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于0，则释放数据内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 删除命令数据过短的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP unlink packet too short");
    }
    # 如果返回值为其他错误
    else if(rc) {
        # 重置删除状态，返回等待删除状态响应的错误信息
        sftp->unlink_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    # 重置删除状态
    sftp->unlink_state = libssh2_NB_state_idle;

    # 获取状态响应中的返回码
    retcode = _libssh2_ntohu32(data + 5);
    # 释放数据内存
    LIBSSH2_FREE(session, data);

    # 如果返回码为成功
    if(retcode == LIBSSH2_FX_OK) {
        return 0;
    }
    # 如果返回码为其他错误
    else {
        # 设置 SFTP 对象的最后错误码，返回 SFTP 协议错误信息
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
/* libssh2_sftp_unlink_ex
 * 从远程服务器删除文件
 */
LIBSSH2_API int
libssh2_sftp_unlink_ex(LIBSSH2_SFTP *sftp, const char *filename,
                       unsigned int filename_len)
{
    int rc;
    // 如果 sftp 为空，则返回错误码
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    // 调用 sftp_unlink 函数删除文件，并调整返回结果
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_unlink(sftp, filename, filename_len));
    return rc;
}

/*
 * sftp_rename
 * 在远程服务器上重命名文件
 */
static int sftp_rename(LIBSSH2_SFTP *sftp, const char *source_filename,
                       unsigned int source_filename_len,
                       const char *dest_filename,
                       unsigned int dest_filename_len, long flags)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    uint32_t packet_len =
        source_filename_len + dest_filename_len + 17 + (sftp->version >=
                                                        5 ? 4 : 0);
    /* packet_len(4) + packet_type(1) + request_id(4) +
       source_filename_len(4) + dest_filename_len(4) + flags(4){SFTP5+) */
    unsigned char *data;
    ssize_t rc;
    // 如果 SFTP 版本小于 2，则返回 SFTP 协议错误
    if(sftp->version < 2) {
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Server does not support RENAME");
    }
}
    # 如果重命名状态为闲置，则执行以下操作
    if(sftp->rename_state == libssh2_NB_state_idle) {
        # 在调试模式下记录重命名的源文件名和目标文件名
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Renaming %s to %s",
                       source_filename, dest_filename);
        # 分配内存以存储重命名数据包
        sftp->rename_s = sftp->rename_packet =
            LIBSSH2_ALLOC(session, packet_len);
        # 如果内存分配失败，则返回内存分配错误
        if(!sftp->rename_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_RENAME "
                                  "packet");
        }

        # 存储数据包长度
        _libssh2_store_u32(&sftp->rename_s, packet_len - 4);
        # 存储 SSH_FXP_RENAME 命令
        *(sftp->rename_s++) = SSH_FXP_RENAME;
        # 存储请求 ID
        sftp->rename_request_id = sftp->request_id++;
        _libssh2_store_u32(&sftp->rename_s, sftp->rename_request_id);
        # 存储源文件名和目标文件名
        _libssh2_store_str(&sftp->rename_s, source_filename,
                           source_filename_len);
        _libssh2_store_str(&sftp->rename_s, dest_filename, dest_filename_len);

        # 如果 SFTP 版本大于等于 5，则存储标志
        if(sftp->version >= 5)
            _libssh2_store_u32(&sftp->rename_s, flags);

        # 设置重命名状态为已创建
        sftp->rename_state = libssh2_NB_state_created;
    }

    # 如果重命名状态为已创建，则执行以下操作
    if(sftp->rename_state == libssh2_NB_state_created) {
        # 将重命名数据包写入通道
        rc = _libssh2_channel_write(channel, 0, sftp->rename_packet,
                                    sftp->rename_s - sftp->rename_packet);
        # 如果返回值为 EAGAIN，则表示需要继续等待
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不等于数据包长度，则释放内存并返回错误
        else if((ssize_t)packet_len != rc) {
            LIBSSH2_FREE(session, sftp->rename_packet);
            sftp->rename_packet = NULL;
            sftp->rename_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_RENAME command");
        }
        # 释放内存并设置重命名状态为闲置
        LIBSSH2_FREE(session, sftp->rename_packet);
        sftp->rename_packet = NULL;
        sftp->rename_state = libssh2_NB_state_idle;

        # 设置重命名状态为已发送
        sftp->rename_state = libssh2_NB_state_sent;
    }
    # 调用 sftp_packet_require 函数，等待 SSH_FXP_STATUS 类型的数据包
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->rename_request_id, &data,
                             &data_len, 9)
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc
    }
    # 如果返回值为 LIBSSH2_ERROR_BUFFER_TOO_SMALL，则表示数据包长度不够
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据包长度大于0，则释放数据内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data)
        }
        # 返回 SFTP 重命名数据包过短的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rename packet too short")
    }
    # 如果返回值不为0，则表示出现了错误
    else if(rc) {
        # 将重命名状态设置为闲置
        sftp->rename_state = libssh2_NB_state_idle
        # 返回等待 FXP STATUS 时出现的错误信息
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS")
    }

    # 将重命名状态设置为闲置
    sftp->rename_state = libssh2_NB_state_idle

    # 获取数据中的返回码
    retcode = _libssh2_ntohu32(data + 5)
    # 释放数据内存
    LIBSSH2_FREE(session, data)

    # 将返回码设置为 SFTP 对象的最后错误码
    sftp->last_errno = retcode

    # 将 SFTP 错误码转换为 libssh2 的返回码或错误信息
    switch(retcode) {
    case LIBSSH2_FX_OK:
        retcode = LIBSSH2_ERROR_NONE
        break

    case LIBSSH2_FX_FILE_ALREADY_EXISTS:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "File already exists and "
                                 "SSH_FXP_RENAME_OVERWRITE not specified")
        break

    case LIBSSH2_FX_OP_UNSUPPORTED:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "Operation Not Supported")
        break

    default:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "SFTP Protocol Error")
        break
    }

    # 返回转换后的返回码
    return retcode
# 重命名远程服务器上的文件
LIBSSH2_API int
libssh2_sftp_rename_ex(LIBSSH2_SFTP *sftp, const char *source_filename,
                       unsigned int source_filename_len,
                       const char *dest_filename,
                       unsigned int dest_filename_len, long flags)
{
    int rc;
    # 如果 sftp 为空，则返回错误码 LIBSSH2_ERROR_BAD_USE
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    # 调整 rc 的值，调用 sftp_rename 函数进行文件重命名
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_rename(sftp, source_filename, source_filename_len,
                             dest_filename, dest_filename_len, flags));
    return rc;
}

# 获取文件系统统计信息
static int sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_STATVFS *st)
{
    LIBSSH2_SFTP *sftp = handle->sftp;
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    # 计算数据包长度
    # 17 = packet_len(4) + packet_type(1) + request_id(4) + ext_len(4) + handle_len (4)
    # 20 = strlen ("fstatvfs@openssh.com")
    uint32_t packet_len = handle->handle_len + 20 + 17;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    unsigned int flag;
    static const unsigned char responses[2] =
        { SSH_FXP_EXTENDED_REPLY, SSH_FXP_STATUS };
}
    # 如果 sftp->fstatvfs_state 状态为 libssh2_NB_state_idle
    if(sftp->fstatvfs_state == libssh2_NB_state_idle) {
        # 打印获取文件系统统计信息的调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Getting file system statistics");
        # 分配内存用于存储数据包
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果分配内存失败，则返回内存分配错误
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC, "Unable to allocate memory for FXP_EXTENDED packet");
        }
        # 存储数据包长度
        _libssh2_store_u32(&s, packet_len - 4);
        # 存储 SSH_FXP_EXTENDED 类型
        *(s++) = SSH_FXP_EXTENDED;
        # 为 fstatvfs 请求分配请求 ID
        sftp->fstatvfs_request_id = sftp->request_id++;
        # 存储请求 ID
        _libssh2_store_u32(&s, sftp->fstatvfs_request_id);
        # 存储字符串 "fstatvfs@openssh.com"，长度为 20
        _libssh2_store_str(&s, "fstatvfs@openssh.com", 20);
        # 存储文件句柄
        _libssh2_store_str(&s, handle->handle, handle->handle_len);
        # 设置 fstatvfs_state 状态为 libssh2_NB_state_created
        sftp->fstatvfs_state = libssh2_NB_state_created;
    }
    else {
        # 如果状态不为 libssh2_NB_state_idle，则使用之前存储的数据包
        packet = sftp->fstatvfs_packet;
    }

    # 如果状态为 libssh2_NB_state_created
    if(sftp->fstatvfs_state == libssh2_NB_state_created) {
        # 将数据包写入通道
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN 或者大于等于 0 且小于数据包长度
        if(rc == LIBSSH2_ERROR_EAGAIN || (0 <= rc && rc < (ssize_t)packet_len)) {
            # 存储数据包，并返回 LIBSSH2_ERROR_EAGAIN
            sftp->fstatvfs_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }
        # 释放数据包内存
        LIBSSH2_FREE(session, packet);
        sftp->fstatvfs_packet = NULL;
        # 如果返回值小于 0
        if(rc < 0) {
            # 设置状态为 libssh2_NB_state_idle，并返回套接字发送错误
            sftp->fstatvfs_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND, "_libssh2_channel_write() failed");
        }
        # 设置状态为 libssh2_NB_state_sent
        sftp->fstatvfs_state = libssh2_NB_state_sent;
    }

    # 要求 SFTP 数据包的响应
    rc = sftp_packet_requirev(sftp, 2, responses, sftp->fstatvfs_request_id, &data, &data_len, 9);
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则返回该值
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果返回的错误码是缓冲区太小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于0
        if(data_len > 0) {
            # 释放之前分配的内存
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 协议错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rename packet too short");
    }
    # 如果返回的错误码不是缓冲区太小
    else if(rc) {
        # 将状态设置为闲置
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        # 返回等待 FXP EXTENDED REPLY 的错误信息
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    # 如果数据的第一个字节是 SSH_FXP_STATUS
    if(data[0] == SSH_FXP_STATUS) {
        # 获取返回码
        int retcode = _libssh2_ntohu32(data + 5);
        # 将状态设置为闲置
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        # 释放内存
        LIBSSH2_FREE(session, data);
        # 将返回码保存到 sftp 对象的 last_errno 属性
        sftp->last_errno = retcode;
        # 返回 SFTP 协议错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }

    # 如果数据长度小于93
    if(data_len < 93) {
        # 释放内存
        LIBSSH2_FREE(session, data);
        # 将状态设置为闲置
        sftp->fstatvfs_state = libssh2_NB_state_idle;
        # 返回 SFTP 协议错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error: short response");
    }

    # 将状态设置为闲置
    sftp->fstatvfs_state = libssh2_NB_state_idle;

    # 从数据中获取文件系统块大小等信息，并保存到 st 结构体中
    st->f_bsize = _libssh2_ntohu64(data + 5);
    st->f_frsize = _libssh2_ntohu64(data + 13);
    st->f_blocks = _libssh2_ntohu64(data + 21);
    st->f_bfree = _libssh2_ntohu64(data + 29);
    st->f_bavail = _libssh2_ntohu64(data + 37);
    st->f_files = _libssh2_ntohu64(data + 45);
    st->f_ffree = _libssh2_ntohu64(data + 53);
    st->f_favail = _libssh2_ntohu64(data + 61);
    st->f_fsid = _libssh2_ntohu64(data + 69);
    flag = (unsigned int)_libssh2_ntohu64(data + 77);
    st->f_namemax = _libssh2_ntohu64(data + 85);

    # 根据 flag 设置文件系统的 flag 属性
    st->f_flag = (flag & SSH_FXE_STATVFS_ST_RDONLY)
        ? LIBSSH2_SFTP_ST_RDONLY : 0;
    st->f_flag |= (flag & SSH_FXE_STATVFS_ST_NOSUID)
        ? LIBSSH2_SFTP_ST_NOSUID : 0;

    # 释放内存
    LIBSSH2_FREE(session, data);
    # 返回成功
    return 0;
}
/* libssh2_sftp_fstatvfs
 * 获取文件系统空间和inode利用率（需要服务器上的fstatvfs@openssh.com支持）
 */
LIBSSH2_API int
libssh2_sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_STATVFS *st)
{
    int rc;
    if(!handle || !st)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, handle->sftp->channel->session,
                 sftp_fstatvfs(handle, st));
    return rc;
}

/*
 * sftp_statvfs
 *
 * 获取文件系统统计信息
 */
static int sftp_statvfs(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, LIBSSH2_SFTP_STATVFS *st)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 17 = packet_len(4) + packet_type(1) + request_id(4) + ext_len(4)
       + path_len (4) */
    /* 19 = strlen ("statvfs@openssh.com") */
    uint32_t packet_len = path_len + 19 + 17;
    unsigned char *packet, *s, *data;
    ssize_t rc;
    unsigned int flag;
    static const unsigned char responses[2] =
        { SSH_FXP_EXTENDED_REPLY, SSH_FXP_STATUS };

    if(sftp->statvfs_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Getting file system statistics of %s", path);
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_EXTENDED "
                                  "packet");
        }

        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_EXTENDED;
        sftp->statvfs_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->statvfs_request_id);
        _libssh2_store_str(&s, "statvfs@openssh.com", 19);
        _libssh2_store_str(&s, path, path_len);

        sftp->statvfs_state = libssh2_NB_state_created;
    }
    else {
        packet = sftp->statvfs_packet;
    }
    # 如果sftp->statvfs_state为libssh2_NB_state_created，则执行以下代码块
    if(sftp->statvfs_state == libssh2_NB_state_created) {
        # 调用_libssh2_channel_write函数向通道写入数据包
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN或者大于等于0且小于数据包长度，则执行以下代码块
        if(rc == LIBSSH2_ERROR_EAGAIN ||
            (0 <= rc && rc < (ssize_t)packet_len)) {
            # 将数据包保存到sftp->statvfs_packet中，并返回LIBSSH2_ERROR_EAGAIN
            sftp->statvfs_packet = packet;
            return LIBSSH2_ERROR_EAGAIN;
        }

        # 释放packet指向的内存
        LIBSSH2_FREE(session, packet);
        # 将sftp->statvfs_packet置为NULL
        sftp->statvfs_packet = NULL;

        # 如果rc小于0，则执行以下代码块
        if(rc < 0) {
            # 将sftp->statvfs_state置为libssh2_NB_state_idle，并返回_LIBSSH2_ERROR函数的结果
            sftp->statvfs_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        # 将sftp->statvfs_state置为libssh2_NB_state_sent
        sftp->statvfs_state = libssh2_NB_state_sent;
    }

    # 调用sftp_packet_requirev函数等待SFTP响应
    rc = sftp_packet_requirev(sftp, 2, responses, sftp->statvfs_request_id,
                              &data, &data_len, 9);
    # 如果返回值为LIBSSH2_ERROR_EAGAIN，则返回rc
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果返回值为LIBSSH2_ERROR_BUFFER_TOO_SMALL，则执行以下代码块
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果data_len大于0，则释放data指向的内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回_LIBSSH2_ERROR函数的结果
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP fstat packet too short");
    }
    # 如果返回值不为0，则执行以下代码块
    else if(rc) {
        # 将sftp->statvfs_state置为libssh2_NB_state_idle，并返回_LIBSSH2_ERROR函数的结果
        sftp->statvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, rc,
                              "Error waiting for FXP EXTENDED REPLY");
    }

    # 如果data的第一个字节为SSH_FXP_STATUS，则执行以下代码块
    if(data[0] == SSH_FXP_STATUS) {
        # 从data中获取返回码，保存到retcode中
        int retcode = _libssh2_ntohu32(data + 5);
        # 将sftp->statvfs_state置为libssh2_NB_state_idle
        sftp->statvfs_state = libssh2_NB_state_idle;
        # 释放data指向的内存
        LIBSSH2_FREE(session, data);
        # 将retcode保存到sftp->last_errno中，并返回_LIBSSH2_ERROR函数的结果
        sftp->last_errno = retcode;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }

    # 如果data_len小于93，则执行以下代码块
    if(data_len < 93) {
        # 释放data指向的内存
        LIBSSH2_FREE(session, data);
        # 将sftp->statvfs_state置为libssh2_NB_state_idle，并返回_LIBSSH2_ERROR函数的结果
        sftp->statvfs_state = libssh2_NB_state_idle;
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error: short response");
    }

    # 将sftp->statvfs_state置为libssh2_NB_state_idle
    sftp->statvfs_state = libssh2_NB_state_idle;
    # 设置文件系统块大小
    st->f_bsize = _libssh2_ntohu64(data + 5);
    # 设置文件系统的基本块大小
    st->f_frsize = _libssh2_ntohu64(data + 13);
    # 设置文件系统的总块数
    st->f_blocks = _libssh2_ntohu64(data + 21);
    # 设置文件系统的可用块数
    st->f_bfree = _libssh2_ntohu64(data + 29);
    # 设置文件系统的可用块数（非超级用户）
    st->f_bavail = _libssh2_ntohu64(data + 37);
    # 设置文件系统的总文件节点数
    st->f_files = _libssh2_ntohu64(data + 45);
    # 设置文件系统的可用文件节点数
    st->f_ffree = _libssh2_ntohu64(data + 53);
    # 设置文件系统的可用文件节点数（非超级用户）
    st->f_favail = _libssh2_ntohu64(data + 61);
    # 设置文件系统 ID
    st->f_fsid = _libssh2_ntohu64(data + 69);
    # 设置文件系统标志
    flag = (unsigned int)_libssh2_ntohu64(data + 77);
    # 设置文件系统的最大文件名长度
    st->f_namemax = _libssh2_ntohu64(data + 85);

    # 设置文件系统的只读标志
    st->f_flag = (flag & SSH_FXE_STATVFS_ST_RDONLY)
        ? LIBSSH2_SFTP_ST_RDONLY : 0;
    # 设置文件系统的无 suid 标志
    st->f_flag |= (flag & SSH_FXE_STATVFS_ST_NOSUID)
        ? LIBSSH2_SFTP_ST_NOSUID : 0;

    # 释放数据内存
    LIBSSH2_FREE(session, data);
    # 返回成功
    return 0;
}

/* libssh2_sftp_statvfs_ex
 * 获取文件系统空间和inode利用率（需要服务器上的statvfs@openssh.com支持）
 */
LIBSSH2_API int
libssh2_sftp_statvfs(LIBSSH2_SFTP *sftp, const char *path,
                     size_t path_len, LIBSSH2_SFTP_STATVFS *st)
{
    int rc;
    if(!sftp || !st)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session, sftp_statvfs(sftp, path, path_len,
                                                          st));
    return rc;
}


/*
 * sftp_mkdir
 *
 * 创建一个SFTP目录
 */
static int sftp_mkdir(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len, long mode)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    LIBSSH2_SFTP_ATTRIBUTES attrs = {
        0, 0, 0, 0, 0, 0, 0
    };
    size_t data_len;
    int retcode;
    ssize_t packet_len;
    unsigned char *packet, *s, *data;
    int rc;

    if(mode != LIBSSH2_SFTP_DEFAULT_MODE) {
        /* SFTP 3及更早版本中的文件类型 */
        attrs.flags = LIBSSH2_SFTP_ATTR_PERMISSIONS;
        attrs.permissions = mode | LIBSSH2_SFTP_ATTR_PFILETYPE_DIR;
    }

    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    packet_len = path_len + 13 + sftp_attrsize(attrs.flags);
    # 如果创建目录的状态为闲置
    if(sftp->mkdir_state == libssh2_NB_state_idle) {
        # 打印调试信息，包括要创建的目录路径和权限模式
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP,
                       "Creating directory %s with mode 0%lo", path, mode);
        # 分配内存用于存储创建目录的数据包
        s = packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果内存分配失败，则返回内存分配错误
        if(!packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_MKDIR "
                                  "packet");
        }

        # 存储数据包长度
        _libssh2_store_u32(&s, packet_len - 4);
        # 存储 SSH_FXP_MKDIR 操作码
        *(s++) = SSH_FXP_MKDIR;
        # 设置创建目录请求的 ID
        sftp->mkdir_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->mkdir_request_id);
        # 存储目录路径
        _libssh2_store_str(&s, path, path_len);

        # 存储目录属性
        s += sftp_attr2bin(s, &attrs);

        # 设置创建目录的状态为已创建
        sftp->mkdir_state = libssh2_NB_state_created;
    }
    # 如果创建目录的状态为已创建
    else {
        # 使用之前存储的数据包
        packet = sftp->mkdir_packet;
    }

    # 如果创建目录的状态为已创建
    if(sftp->mkdir_state == libssh2_NB_state_created) {
        # 将数据包写入通道
        rc = _libssh2_channel_write(channel, 0, packet, packet_len);
        # 如果写入操作被暂停，则保存数据包并返回暂停状态
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            sftp->mkdir_packet = packet;
            return rc;
        }
        # 如果写入的数据包长度与实际长度不符，则释放内存并返回错误
        if(packet_len != rc) {
            LIBSSH2_FREE(session, packet);
            sftp->mkdir_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "_libssh2_channel_write() failed");
        }
        # 释放内存并设置创建目录的状态为已发送
        LIBSSH2_FREE(session, packet);
        sftp->mkdir_state = libssh2_NB_state_sent;
        sftp->mkdir_packet = NULL;
    }

    # 等待 SFTP 响应，获取创建目录的状态
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS, sftp->mkdir_request_id,
                             &data, &data_len, 9);
    # 如果需要继续等待，则返回暂停状态
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果缓冲区太小，则释放数据并返回 SFTP 协议错误
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP mkdir packet too short");
    }
    # 如果 rc 不为 0，则执行以下代码
    else if(rc) {
        # 将 sftp->mkdir_state 设置为 libssh2_NB_state_idle
        sftp->mkdir_state = libssh2_NB_state_idle;
        # 返回一个带有错误信息的 libssh2_error 对象
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    # 将 sftp->mkdir_state 设置为 libssh2_NB_state_idle
    sftp->mkdir_state = libssh2_NB_state_idle;

    # 从数据中解析出 retcode
    retcode = _libssh2_ntohu32(data + 5);
    # 释放 data 的内存
    LIBSSH2_FREE(session, data);

    # 如果 retcode 等于 LIBSSH2_FX_OK，则执行以下代码
    if(retcode == LIBSSH2_FX_OK) {
        # 在调试日志中记录 "OK!"
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "OK!");
        # 返回 0
        return 0;
    }
    # 否则执行以下代码
    else {
        # 将 sftp->last_errno 设置为 retcode
        sftp->last_errno = retcode;
        # 返回一个带有 SFTP 协议错误信息的 libssh2_error 对象
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
}

/*
 * libssh2_sftp_mkdir_ex
 *
 * Create an SFTP directory
 */
LIBSSH2_API int
libssh2_sftp_mkdir_ex(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len, long mode)
{
    int rc;
    // 如果 sftp 为空，则返回错误码 LIBSSH2_ERROR_BAD_USE
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    // 调用 sftp_mkdir 函数创建 SFTP 目录，并调整返回值
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_mkdir(sftp, path, path_len, mode));
    return rc;
}

/* sftp_rmdir
 * Remove a directory
 */
static int sftp_rmdir(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    int retcode;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    ssize_t packet_len = path_len + 13;
    unsigned char *s, *data;
    int rc;

    // 如果 sftp->rmdir_state 等于 libssh2_NB_state_idle
    if(sftp->rmdir_state == libssh2_NB_state_idle) {
        // 打印调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "Removing directory: %s",
                       path);
        // 分配内存给 sftp->rmdir_packet
        s = sftp->rmdir_packet = LIBSSH2_ALLOC(session, packet_len);
        // 如果分配内存失败，则返回错误信息
        if(!sftp->rmdir_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_RMDIR "
                                  "packet");
        }

        // 存储数据到 sftp->rmdir_packet 中
        _libssh2_store_u32(&s, packet_len - 4);
        *(s++) = SSH_FXP_RMDIR;
        sftp->rmdir_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->rmdir_request_id);
        _libssh2_store_str(&s, path, path_len);

        // 设置 sftp->rmdir_state 为 libssh2_NB_state_created
        sftp->rmdir_state = libssh2_NB_state_created;
    }
}
    # 如果删除目录的状态为已创建
    if(sftp->rmdir_state == libssh2_NB_state_created) {
        # 向通道写入删除目录的数据包
        rc = _libssh2_channel_write(channel, 0, sftp->rmdir_packet,
                                    packet_len);
        # 如果返回值为需要再次调用，则返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果写入的数据包长度不等于返回的长度
        else if(packet_len != rc) {
            # 释放内存，将删除目录的数据包指针置为空
            LIBSSH2_FREE(session, sftp->rmdir_packet);
            sftp->rmdir_packet = NULL;
            # 将删除目录的状态置为闲置
            sftp->rmdir_state = libssh2_NB_state_idle;
            # 返回发送 FXR_RMDIR 命令失败的错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send FXP_RMDIR command");
        }
        # 释放内存，将删除目录的数据包指针置为空
        LIBSSH2_FREE(session, sftp->rmdir_packet);
        sftp->rmdir_packet = NULL;
        # 将删除目录的状态置为已发送
        sftp->rmdir_state = libssh2_NB_state_sent;
    }

    # 等待接收删除目录的状态数据包
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->rmdir_request_id, &data, &data_len, 9);
    # 如果返回值为需要再次调用，则返回该值
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 如果返回值为缓冲区太小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据包长度大于0，则释放内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 删除目录数据包太短的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rmdir packet too short");
    }
    # 如果返回值不为0
    else if(rc) {
        # 将删除目录的状态置为闲置
        sftp->rmdir_state = libssh2_NB_state_idle;
        # 返回等待 FXR STATUS 命令失败的错误信息
        return _libssh2_error(session, rc,
                              "Error waiting for FXP STATUS");
    }

    # 将删除目录的状态置为闲置
    sftp->rmdir_state = libssh2_NB_state_idle;

    # 获取数据包中的返回码
    retcode = _libssh2_ntohu32(data + 5);
    # 释放内存
    LIBSSH2_FREE(session, data);

    # 如果返回码为成功
    if(retcode == LIBSSH2_FX_OK) {
        return 0;
    }
    # 否则
    else {
        # 将最后的错误码置为返回码
        sftp->last_errno = retcode;
        # 返回 SFTP 协议错误的错误信息
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP Protocol Error");
    }
/* libssh2_sftp_rmdir_ex
 * 删除一个目录
 */
LIBSSH2_API int
libssh2_sftp_rmdir_ex(LIBSSH2_SFTP *sftp, const char *path,
                      unsigned int path_len)
{
    int rc;
    // 如果 sftp 为空，则返回错误码 LIBSSH2_ERROR_BAD_USE
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    // 调整 rc 的值，调用 sftp_rmdir 函数删除目录
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_rmdir(sftp, path, path_len));
    return rc;
}

/* sftp_stat
 * 获取文件或符号链接的状态
 */
static int sftp_stat(LIBSSH2_SFTP *sftp, const char *path,
                     unsigned int path_len, int stat_type,
                     LIBSSH2_SFTP_ATTRIBUTES * attrs)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len;
    /* 13 = packet_len(4) + packet_type(1) + request_id(4) + path_len(4) */
    ssize_t packet_len =
        path_len + 13 +
        ((stat_type ==
          LIBSSH2_SFTP_SETSTAT) ? sftp_attrsize(attrs->flags) : 0);
    unsigned char *s, *data;
    static const unsigned char stat_responses[2] =
        { SSH_FXP_ATTRS, SSH_FXP_STATUS };
    int rc;
    # 如果 SFTP 状态为闲置
    if(sftp->stat_state == libssh2_NB_state_idle) {
        # 打印 SFTP 操作类型和路径
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%s %s",
                       (stat_type == LIBSSH2_SFTP_SETSTAT) ? "Set-statting" :
                       (stat_type ==
                        LIBSSH2_SFTP_LSTAT ? "LStatting" : "Statting"), path);
        # 分配内存用于存储 FXP_*STAT 数据包
        s = sftp->stat_packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果内存分配失败，则返回错误
        if(!sftp->stat_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for FXP_*STAT "
                                  "packet");
        }

        # 存储数据包长度
        _libssh2_store_u32(&s, packet_len - 4);

        # 根据操作类型存储相应的 SFTP 操作码
        switch(stat_type) {
        case LIBSSH2_SFTP_SETSTAT:
            *(s++) = SSH_FXP_SETSTAT;
            break;

        case LIBSSH2_SFTP_LSTAT:
            *(s++) = SSH_FXP_LSTAT;
            break;

        case LIBSSH2_SFTP_STAT:
        default:
            *(s++) = SSH_FXP_STAT;
        }
        # 存储请求 ID
        sftp->stat_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->stat_request_id);
        # 存储路径
        _libssh2_store_str(&s, path, path_len);

        # 如果操作类型为 SETSTAT，则存储属性数据
        if(stat_type == LIBSSH2_SFTP_SETSTAT)
            s += sftp_attr2bin(s, attrs);

        # 设置 SFTP 状态为已创建
        sftp->stat_state = libssh2_NB_state_created;
    }

    # 如果 SFTP 状态为已创建
    if(sftp->stat_state == libssh2_NB_state_created) {
        # 写入数据包到通道
        rc = _libssh2_channel_write(channel, 0, sftp->stat_packet, packet_len);
        # 如果写入操作被暂停，则返回暂停状态
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果写入长度与数据包长度不一致，则释放内存并返回错误
        else if(packet_len != rc) {
            LIBSSH2_FREE(session, sftp->stat_packet);
            sftp->stat_packet = NULL;
            sftp->stat_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send STAT/LSTAT/SETSTAT command");
        }
        # 释放内存并设置 SFTP 状态为闲置
        LIBSSH2_FREE(session, sftp->stat_packet);
        sftp->stat_packet = NULL;
        sftp->stat_state = libssh2_NB_state_idle;
    }
    # 发送 SFTP 请求，等待响应，要求响应的类型为 stat_responses 中的一种
    rc = sftp_packet_requirev(sftp, 2, stat_responses,
                              sftp->stat_request_id, &data, &data_len, 9);
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;
    # 如果返回值为 LIBSSH2_ERROR_BUFFER_TOO_SMALL，表示数据包太小
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于 0，释放数据内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 协议错误，提示数据包太短
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP stat packet too short");
    }
    # 如果返回值不为 0，表示出现错误
    else if(rc) {
        # 将状态设置为闲置
        sftp->stat_state = libssh2_NB_state_idle;
        # 返回超时等待状态消息的错误
        return _libssh2_error(session, rc,
                              "Timeout waiting for status message");
    }

    # 将状态设置为闲置
    sftp->stat_state = libssh2_NB_state_idle;

    # 如果数据的第一个字节为 SSH_FXP_STATUS
    if(data[0] == SSH_FXP_STATUS) {
        int retcode;

        # 从数据中获取返回码
        retcode = _libssh2_ntohu32(data + 5);
        # 释放数据内存
        LIBSSH2_FREE(session, data);
        # 如果返回码为 LIBSSH2_FX_OK，表示成功，清空属性并返回 0
        if(retcode == LIBSSH2_FX_OK) {
            memset(attrs, 0, sizeof(LIBSSH2_SFTP_ATTRIBUTES));
            return 0;
        }
        # 否则，设置最后的错误码，并返回 SFTP 协议错误
        else {
            sftp->last_errno = retcode;
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }

    # 清空属性
    memset(attrs, 0, sizeof(LIBSSH2_SFTP_ATTRIBUTES));
    # 将二进制数据转换为属性，如果失败，释放数据内存并返回 SFTP 协议错误
    if(sftp_bin2attr(attrs, data + 5, data_len - 5) < 0) {
        LIBSSH2_FREE(session, data);
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Attributes too short in SFTP fstat");
    }

    # 释放数据内存
    LIBSSH2_FREE(session, data);

    # 返回 0，表示成功
    return 0;
# 检查传入的 sftp 对象是否为空，如果为空则返回错误码
LIBSSH2_API int
libssh2_sftp_stat_ex(LIBSSH2_SFTP *sftp, const char *path,
                     unsigned int path_len, int stat_type,
                     LIBSSH2_SFTP_ATTRIBUTES *attrs)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    # 调用 sftp_stat 函数获取文件或符号链接的属性，并将结果存储在 attrs 中
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_stat(sftp, path, path_len, stat_type, attrs));
    return rc;
}

# 读取或设置符号链接
static int sftp_symlink(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, char *target,
                        unsigned int target_len, int link_type)
{
    LIBSSH2_CHANNEL *channel = sftp->channel;
    LIBSSH2_SESSION *session = channel->session;
    size_t data_len, link_len;
    # 计算数据包的长度
    ssize_t packet_len =
        path_len + 13 +
        ((link_type == LIBSSH2_SFTP_SYMLINK) ? (4 + target_len) : 0);
    unsigned char *s, *data;
    static const unsigned char link_responses[2] =
        { SSH_FXP_NAME, SSH_FXP_STATUS };
    int retcode;

    # 如果服务器不支持 SYMLINK 或 READLINK，并且 sftp 版本小于 3，则返回错误码
    if((sftp->version < 3) && (link_type != LIBSSH2_SFTP_REALPATH)) {
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Server does not support SYMLINK or READLINK");
    }
}
    # 如果 SFTP 的符号链接状态为闲置
    if(sftp->symlink_state == libssh2_NB_state_idle) {
        # 分配内存以存储符号链接/读取链接/真实路径数据包
        s = sftp->symlink_packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果内存分配失败，则返回内存分配错误
        if(!sftp->symlink_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "SYMLINK/READLINK/REALPATH packet");
        }

        # 打印 SFTP 操作的调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_SFTP, "%s %s on %s",
                       (link_type ==
                        LIBSSH2_SFTP_SYMLINK) ? "Creating" : "Reading",
                       (link_type ==
                        LIBSSH2_SFTP_REALPATH) ? "realpath" : "symlink", path);

        # 存储数据包长度
        _libssh2_store_u32(&s, packet_len - 4);

        # 根据链接类型存储相应的 SFTP 操作类型
        switch(link_type) {
        case LIBSSH2_SFTP_REALPATH:
            *(s++) = SSH_FXP_REALPATH;
            break;

        case LIBSSH2_SFTP_SYMLINK:
            *(s++) = SSH_FXP_SYMLINK;
            break;

        case LIBSSH2_SFTP_READLINK:
        default:
            *(s++) = SSH_FXP_READLINK;
        }
        # 存储请求 ID
        sftp->symlink_request_id = sftp->request_id++;
        _libssh2_store_u32(&s, sftp->symlink_request_id);
        # 存储路径字符串
        _libssh2_store_str(&s, path, path_len);

        # 如果是创建符号链接操作，还需要存储目标路径字符串
        if(link_type == LIBSSH2_SFTP_SYMLINK)
            _libssh2_store_str(&s, target, target_len);

        # 将符号链接状态设置为已创建
        sftp->symlink_state = libssh2_NB_state_created;
    }
    # 如果符号链接状态为已创建
    if(sftp->symlink_state == libssh2_NB_state_created) {
        # 向通道写入符号链接数据包
        ssize_t rc = _libssh2_channel_write(channel, 0, sftp->symlink_packet,
                                            packet_len);
        # 如果返回值为再试一次，则返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        # 如果数据包长度不等于写入长度
        else if(packet_len != rc) {
            # 释放内存
            LIBSSH2_FREE(session, sftp->symlink_packet);
            sftp->symlink_packet = NULL;
            sftp->symlink_state = libssh2_NB_state_idle;
            # 返回无法发送符号链接/读取链接命令的错误
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send SYMLINK/READLINK command");
        }
        # 释放内存
        LIBSSH2_FREE(session, sftp->symlink_packet);
        sftp->symlink_packet = NULL;
        # 设置符号链接状态为已发送
        sftp->symlink_state = libssh2_NB_state_sent;
    }

    # 要求SFTP数据包
    retcode = sftp_packet_requirev(sftp, 2, link_responses,
                                   sftp->symlink_request_id, &data,
                                   &data_len, 9);
    # 如果返回值为再试一次，则返回该值
    if(retcode == LIBSSH2_ERROR_EAGAIN)
        return retcode;
    # 如果返回值为缓冲区太小
    else if(retcode == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        # 如果数据长度大于0，则释放内存
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        # 返回SFTP符号链接数据包太短的错误
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP symlink packet too short");
    }
    # 如果返回值不为0
    else if(retcode) {
        sftp->symlink_state = libssh2_NB_state_idle;
        # 返回等待状态消息时出错的错误
        return _libssh2_error(session, retcode,
                              "Error waiting for status message");
    }

    # 设置符号链接状态为闲置
    sftp->symlink_state = libssh2_NB_state_idle;

    # 如果数据的第一个字节为SSH_FXP_STATUS
    if(data[0] == SSH_FXP_STATUS) {
        # 获取返回码
        retcode = _libssh2_ntohu32(data + 5);
        # 释放内存
        LIBSSH2_FREE(session, data);
        # 如果返回码为OK，则返回无错误
        if(retcode == LIBSSH2_FX_OK)
            return LIBSSH2_ERROR_NONE;
        # 否则
        else {
            # 设置SFTP的最后错误码
            sftp->last_errno = retcode;
            # 返回SFTP协议错误
            return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                  "SFTP Protocol Error");
        }
    }
    # 如果数据中的第5个字节表示的无符号32位整数小于1
    if(_libssh2_ntohu32(data + 5) < 1) {
        # 释放数据内存
        LIBSSH2_FREE(session, data);
        # 返回 SFTP 协议错误，表示无效的 READLINK/REALPATH 响应，没有名称条目
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Invalid READLINK/REALPATH response, "
                              "no name entries");
    }

    # 如果数据长度小于13
    if(data_len < 13) {
        # 如果数据长度大于0
        if(data_len > 0) {
            # 释放数据内存
            LIBSSH2_FREE(session, data);
        }
        # 返回 SFTP 协议错误，表示 SFTP stat 数据包太短
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP stat packet too short");
    }

    # 读取一个无符号32位整数并将其存储为有符号32位值
    link_len = _libssh2_ntohu32(data + 9);
    # 如果链接长度小于目标长度
    if(link_len < target_len) {
        # 将数据中的链接内容复制到目标数组中
        memcpy(target, data + 13, link_len);
        # 在目标数组的链接内容后面添加字符串结束符
        target[link_len] = 0;
        # 将链接长度作为返回值
        retcode = (int)link_len;
    }
    # 否则返回缓冲区太小的错误代码
    else
        retcode = LIBSSH2_ERROR_BUFFER_TOO_SMALL;
    # 释放数据内存
    LIBSSH2_FREE(session, data);

    # 返回结果代码
    return retcode;
/* libssh2_sftp_symlink_ex
 * 读取或设置一个符号链接
 */
LIBSSH2_API int
libssh2_sftp_symlink_ex(LIBSSH2_SFTP *sftp, const char *path,
                        unsigned int path_len, char *target,
                        unsigned int target_len, int link_type)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_symlink(sftp, path, path_len, target, target_len,
                              link_type));
    return rc;
}

/* libssh2_sftp_last_error
 * 返回 SFTP 报告的最后一个错误代码
 */
LIBSSH2_API unsigned long
libssh2_sftp_last_error(LIBSSH2_SFTP *sftp)
{
    if(!sftp)
        return 0;

    return sftp->last_errno;
}

/* libssh2_sftp_get_channel
 * 返回 sftp 的通道，然后调用者可以控制通道的行为。
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp)
{
    if(!sftp)
        return NULL;

    return sftp->channel;
}
```