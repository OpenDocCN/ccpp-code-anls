# `nmap\libssh2\src\session.h`

```cpp
#ifndef __LIBSSH2_SESSION_H
#define __LIBSSH2_SESSION_H
/* 定义宏，用于条件编译，避免重复包含同一头文件 */
/* 版权声明，版权归属及授权条件 */
/* 在源代码中重新分发时，必须保留版权声明、条件列表和以下免责声明 */
/* 在二进制形式中重新分发时，必须在文档和/或其他提供的材料中复制版权声明、条件列表和以下免责声明 */
/* 不得使用版权所有者的名称或任何其他贡献者的名称，未经特定事先书面许可，不得用于认可或推广从本软件衍生的产品 */
/* 免责声明，声明软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
/* 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，版权所有者或贡献者均不承担任何直接、间接、附带、特殊、惩罚性或后果性的损害责任，即使已被告知可能发生此类损害。
#endif
/* 定义一个宏，用于调整返回值，确保在非阻塞模式下无论返回值如何都会返回，但在阻塞模式下，如果返回的原因是 EAGAIN，则会阻塞直到有数据可读 */
#define BLOCK_ADJUST(rc, sess, x) \
    do { \
       time_t entry_time = time(NULL); \  // 获取当前时间
       do { \
          rc = x; \  // 调用函数 x，并将返回值赋给 rc
          /* 下面的检查顺序很重要，以正确处理 'sess' 被释放的情况 */ \
          if((rc != LIBSSH2_ERROR_EAGAIN) || !sess->api_block_mode) \  // 如果返回值不是 EAGAIN 或者 sess 不是阻塞模式，则跳出循环
              break; \
          rc = _libssh2_wait_socket(sess, entry_time);  \  // 调用内部函数等待套接字动作
       } while(!rc);   \  // 如果返回值为 0，则继续循环
    } while(0)  // 结束宏定义

/*
 * 对于返回指针的函数，我们需要检查 API 是否是非阻塞的并立即返回。如果指针不是 NULL，则立即返回。如果 API 是阻塞的并且我们得到一个 NULL，则检查 errno，只有当它是 EAGAIN 时我们才循环并等待套接字动作。
 */
#define BLOCK_ADJUST_ERRNO(ptr, sess, x) \
    do { \
       time_t entry_time = time(NULL); \  // 获取当前时间
       int rc; \  // 定义返回值变量
       do { \
           ptr = x; \  // 调用函数 x，并将返回值赋给 ptr
           if(!sess->api_block_mode || \
              (ptr != NULL) || \
              (libssh2_session_last_errno(sess) != LIBSSH2_ERROR_EAGAIN) ) \  // 如果不是阻塞模式，或者 ptr 不是 NULL，或者最后的错误码不是 EAGAIN，则跳出循环
               break; \
           rc = _libssh2_wait_socket(sess, entry_time); \  // 调用内部函数等待套接字动作
        } while(!rc); \  // 如果返回值为 0，则继续循环
    } while(0)  // 结束宏定义

// 定义一个内部函数，用于等待套接字动作
int _libssh2_wait_socket(LIBSSH2_SESSION *session, time_t entry_time);

// 这是内部的设置阻塞函数
int _libssh2_session_set_blocking(LIBSSH2_SESSION * session, int blocking);

#endif /* __LIBSSH2_SESSION_H */
```