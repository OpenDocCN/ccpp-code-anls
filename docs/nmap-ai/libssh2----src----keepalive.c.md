# `nmap\libssh2\src\keepalive.c`

```
/*
 * 版权声明和许可声明
 * 作者：Simon Josefsson
 *
 * 在源代码和二进制形式下，允许进行修改或不进行修改的情况下进行再发布和使用
 * 必须满足以下条件：
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责
 * 即使已被告知可能发生此类损害，也不得在任何责任理论下对本软件的使用负责
 */

#include "libssh2_priv.h"
#include "transport.h" /* _libssh2_transport_write */

/* 保持连接的相关设置 */

LIBSSH2_API void
libssh2_keepalive_config (LIBSSH2_SESSION *session,
                          int want_reply,
                          unsigned interval)
{
    如果间隔为1，则将会话的保持连接间隔设置为2
    if(interval == 1)
        session->keepalive_interval = 2;
    否则将会话的保持连接间隔设置为指定的间隔
    else
        session->keepalive_interval = interval;
    # 设置会话的keepalive_want_reply属性，根据want_reply的值来决定是1还是0
    session->keepalive_want_reply = want_reply ? 1 : 0;
# 定义 libssh2_keepalive_send 函数，用于发送 SSH 保持活动消息
LIBSSH2_API int
libssh2_keepalive_send (LIBSSH2_SESSION *session,
                        int *seconds_to_next)
{
    time_t now;  # 声明变量 now 用于存储当前时间

    if(!session->keepalive_interval) {  # 如果会话的保持活动间隔为 0
        if(seconds_to_next)  # 如果 seconds_to_next 不为空
            *seconds_to_next = 0  # 将 seconds_to_next 设置为 0
        return 0  # 返回 0
    }

    now = time(NULL)  # 获取当前时间

    if(session->keepalive_last_sent + session->keepalive_interval <= now) {  # 如果上次发送保持活动消息的时间加上保持活动间隔小于等于当前时间
        /* Format is
           "SSH_MSG_GLOBAL_REQUEST || 4-byte len || str || want-reply". */
        unsigned char keepalive_data[]  # 声明并初始化 keepalive_data 数组
            = "\x50\x00\x00\x00\x15keepalive@libssh2.orgW";  # 设置 keepalive_data 的值
        size_t len = sizeof(keepalive_data) - 1;  # 计算 keepalive_data 的长度
        int rc;  # 声明变量 rc 用于存储返回值

        keepalive_data[len - 1] =  # 修改 keepalive_data 的倒数第二个字符
            (unsigned char)session->keepalive_want_reply;  # 根据会话的保持活动是否需要回复来设置 keepalive_data 的倒数第二个字符

        rc = _libssh2_transport_send(session, keepalive_data, len, NULL, 0);  # 调用 _libssh2_transport_send 函数发送 keepalive_data
        /* Silently ignore PACKET_EAGAIN here: if the write buffer is
           already full, sending another keepalive is not useful. */
        if(rc && rc != LIBSSH2_ERROR_EAGAIN) {  # 如果返回值不为 0 且不是 EAGAIN 错误
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send keepalive message");  # 输出错误信息
            return rc  # 返回错误码
        }

        session->keepalive_last_sent = now  # 更新上次发送保持活动消息的时间为当前时间
        if(seconds_to_next)  # 如果 seconds_to_next 不为空
            *seconds_to_next = session->keepalive_interval  # 将 seconds_to_next 设置为保持活动间隔
    }
    else if(seconds_to_next) {  # 如果 seconds_to_next 不为空
        *seconds_to_next = (int) (session->keepalive_last_sent - now)  # 计算下次发送保持活动消息的时间
            + session->keepalive_interval;  # 加上保持活动间隔
    }

    return 0  # 返回 0
}
```