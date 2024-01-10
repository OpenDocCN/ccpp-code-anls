# `nmap\libssh2\os400\macros.h`

```
/*
 * 版权声明
 * 版权所有（C）2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 * 保留所有权利。
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，前提是满足以下条件：
 *
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 *
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）责任的理论，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。
 */

#ifndef LIBSSH2_MACROS_H_
#define LIBSSH2_MACROS_H_

#include "libssh2.h"  // 包含 libssh2 库的头文件
#include "libssh2_publickey.h"  // 包含 libssh2 公钥的头文件
#include "libssh2_sftp.h"  // 包含 libssh2 SFTP 的头文件

/*
 * 生成 C 宏的包装程序的虚拟原型。
 * 这是为了没有聪明预处理器（ILE/RPG）的语言而设计的辅助工具。
 */
LIBSSH2_API LIBSSH2_SESSION * libssh2_session_init(void);  // 生成 libssh2_session_init 的虚拟原型
# 断开 SSH 会话
LIBSSH2_API int libssh2_session_disconnect(LIBSSH2_SESSION *session,
                                           const char *description);
# 使用用户名和密码进行用户认证
LIBSSH2_API int libssh2_userauth_password(LIBSSH2_SESSION *session,
                                          const char *username,
                                          const char *password);
# 使用公钥和私钥文件进行用户认证
LIBSSH2_API int
libssh2_userauth_publickey_fromfile(LIBSSH2_SESSION *session,
                                    const char *username,
                                    const char *publickey,
                                    const char *privatekey,
                                    const char *passphrase);
# 使用基于主机的公钥和私钥文件进行用户认证
LIBSSH2_API int
libssh2_userauth_hostbased_fromfile(LIBSSH2_SESSION *session,
                                    const char *username,
                                    const char *publickey,
                                    const char *privatekey,
                                    const char *passphrase,
                                    const char *hostname);
# 使用键盘交互式方式进行用户认证
LIBSSH2_API int
libssh2_userauth_keyboard_interactive(LIBSSH2_SESSION* session,
                                      const char *username,
                                      LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC(
                                                       (*response_callback)));
# 打开一个会话通道
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_open_session(LIBSSH2_SESSION *session);
# 直接打开一个 TCP/IP 通道
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_direct_tcpip(LIBSSH2_SESSION *session, const char *host,
                             int port);
# 监听并转发指定端口的通道
LIBSSH2_API LIBSSH2_LISTENER *
libssh2_channel_forward_listen(LIBSSH2_SESSION *session, int port);
# 设置通道环境变量
LIBSSH2_API int
libssh2_channel_setenv(LIBSSH2_CHANNEL *channel,
                       const char *varname, const char *value);
# 请求分配一个伪终端
LIBSSH2_API int
libssh2_channel_request_pty(LIBSSH2_CHANNEL *channel, const char *term);
# 请求设置伪终端的大小
LIBSSH2_API int
libssh2_channel_request_pty_size(LIBSSH2_CHANNEL *channel,
                                 int width, int height);
# 请求在 SSH 通道上启用 X11 转发
LIBSSH2_API int
libssh2_channel_x11_req(LIBSSH2_CHANNEL *channel, int screen_number);

# 在 SSH 通道上启动一个交互式 shell
LIBSSH2_API int
libssh2_channel_shell(LIBSSH2_CHANNEL *channel);

# 在 SSH 通道上执行指定的命令
LIBSSH2_API int
libssh2_channel_exec(LIBSSH2_CHANNEL *channel, const char *command);

# 在 SSH 通道上启动指定的子系统
LIBSSH2_API int
libssh2_channel_subsystem(LIBSSH2_CHANNEL *channel, const char *subsystem);

# 从 SSH 通道中读取数据
LIBSSH2_API ssize_t
libssh2_channel_read(LIBSSH2_CHANNEL *channel, char *buf, size_t buflen);

# 从 SSH 通道的标准错误流中读取数据
LIBSSH2_API ssize_t
libssh2_channel_read_stderr(LIBSSH2_CHANNEL *channel, char *buf, size_t buflen);

# 获取 SSH 通道的读取窗口大小
LIBSSH2_API unsigned long
libssh2_channel_window_read(LIBSSH2_CHANNEL *channel);

# 向 SSH 通道中写入数据
LIBSSH2_API ssize_t
libssh2_channel_write(LIBSSH2_CHANNEL *channel, const char *buf, size_t buflen);

# 向 SSH 通道的标准错误流中写入数据
LIBSSH2_API ssize_t
libssh2_channel_write_stderr(LIBSSH2_CHANNEL *channel,
                             const char *buf, size_t buflen);

# 获取 SSH 通道的写入窗口大小
LIBSSH2_API unsigned long
libssh2_channel_window_write(LIBSSH2_CHANNEL *channel);

# 刷新 SSH 通道的写入缓冲区
LIBSSH2_API int libssh2_channel_flush(LIBSSH2_CHANNEL *channel);

# 刷新 SSH 通道的标准错误流的写入缓冲区
LIBSSH2_API int libssh2_channel_flush_stderr(LIBSSH2_CHANNEL *channel);

# 使用 SCP 协议发送文件到远程主机
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send(LIBSSH2_SESSION *session,
                 const char *path, int mode, libssh2_int64_t size);

# 向公钥列表中添加公钥
LIBSSH2_API int
libssh2_publickey_add(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
              const unsigned char *blob, unsigned long blob_len,
                      char overwrite, unsigned long num_attrs,
              const libssh2_publickey_attribute attrs[]);

# 从公钥列表中移除公钥
LIBSSH2_API int
libssh2_publickey_remove(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
                         const unsigned char *blob, unsigned long blob_len);

# 判断 SFTP 权限是否表示一个符号链接
LIBSSH2_API int LIBSSH2_SFTP_S_ISLNK(unsigned long permissions);

# 判断 SFTP 权限是否表示一个普通文件
LIBSSH2_API int LIBSSH2_SFTP_S_ISREG(unsigned long permissions);

# 判断 SFTP 权限是否表示一个目录
LIBSSH2_API int LIBSSH2_SFTP_S_ISDIR(unsigned long permissions);

# 判断 SFTP 权限是否表示一个字符设备
LIBSSH2_API int LIBSSH2_SFTP_S_ISCHR(unsigned long permissions);

# 判断 SFTP 权限是否表示一个块设备
LIBSSH2_API int LIBSSH2_SFTP_S_ISBLK(unsigned long permissions);
# 定义一个函数，用于判断给定权限是否表示一个 FIFO 文件
LIBSSH2_API int LIBSSH2_SFTP_S_ISFIFO(unsigned long permissions);

# 定义一个函数，用于判断给定权限是否表示一个套接字文件
LIBSSH2_API int LIBSSH2_SFTP_S_ISSOCK(unsigned long permissions);

# 打开一个文件，并返回一个文件句柄
LIBSSH2_API LIBSSH2_SFTP_HANDLE *libssh2_sftp_open(LIBSSH2_SFTP *sftp, const char *filename,
                                                  unsigned long flags, long mode);

# 打开一个目录，并返回一个目录句柄
LIBSSH2_API LIBSSH2_SFTP_HANDLE *libssh2_sftp_opendir(LIBSSH2_SFTP *sftp, const char *path);

# 读取目录中的条目，并返回文件名和属性
LIBSSH2_API int libssh2_sftp_readdir(LIBSSH2_SFTP_HANDLE *handle,
                                     char *buffer, size_t buffer_maxlen,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);

# 关闭一个文件
LIBSSH2_API int libssh2_sftp_close(LIBSSH2_SFTP_HANDLE *handle);

# 关闭一个目录
LIBSSH2_API int libssh2_sftp_closedir(LIBSSH2_SFTP_HANDLE *handle);

# 将目录句柄的位置重置到开头
LIBSSH2_API void libssh2_sftp_rewind(LIBSSH2_SFTP_HANDLE *handle);

# 获取文件的属性
LIBSSH2_API int libssh2_sftp_fstat(LIBSSH2_SFTP_HANDLE *handle,
                                   LIBSSH2_SFTP_ATTRIBUTES *attrs);

# 设置文件的属性
LIBSSH2_API int libssh2_sftp_fsetstat(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_ATTRIBUTES *attrs);

# 重命名文件或目录
LIBSSH2_API int libssh2_sftp_rename(LIBSSH2_SFTP *sftp,
                                    const char *source_filename,
                                    const char *dest_filename);

# 删除文件
LIBSSH2_API int libssh2_sftp_unlink(LIBSSH2_SFTP *sftp, const char *filename);

# 创建目录
LIBSSH2_API int libssh2_sftp_mkdir(LIBSSH2_SFTP *sftp,
                                   const char *path, long mode);

# 删除目录
LIBSSH2_API int libssh2_sftp_rmdir(LIBSSH2_SFTP *sftp, const char *path);

# 获取文件的属性
LIBSSH2_API int libssh2_sftp_stat(LIBSSH2_SFTP *sftp, const char *path,
                                  LIBSSH2_SFTP_ATTRIBUTES *attrs);

# 获取文件的属性（不会跟随符号链接）
LIBSSH2_API int libssh2_sftp_lstat(LIBSSH2_SFTP *sftp, const char *path,
                                   LIBSSH2_SFTP_ATTRIBUTES *attrs);

# 设置文件的属性
LIBSSH2_API int libssh2_sftp_setstat(LIBSSH2_SFTP *sftp, const char *path,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);
# 定义 libssh2_sftp_symlink 函数，用于在 SFTP 服务器上创建符号链接
LIBSSH2_API int libssh2_sftp_symlink(LIBSSH2_SFTP *sftp, const char *orig,
                                     char *linkpath);

# 定义 libssh2_sftp_readlink 函数，用于在 SFTP 服务器上读取符号链接的目标路径
LIBSSH2_API int libssh2_sftp_readlink(LIBSSH2_SFTP *sftp, const char *path,
                                      char *target, unsigned int maxlen);

# 定义 libssh2_sftp_realpath 函数，用于在 SFTP 服务器上获取指定路径的真实路径
LIBSSH2_API int libssh2_sftp_realpath(LIBSSH2_SFTP *sftp, const char *path,
                                      char *target, unsigned int maxlen);

# 结束条件编译指令
#endif
```