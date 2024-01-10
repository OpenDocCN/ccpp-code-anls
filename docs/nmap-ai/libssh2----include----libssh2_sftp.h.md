# `nmap\libssh2\include\libssh2_sftp.h`

```
# 版权声明，版权所有，保留所有权利
# 在源代码和二进制形式下重新分发和使用，无论是否修改，都需要满足以下条件：
#   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
#   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
#   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
# 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。
# 如果在编写时文档中标记为“不要实现”的版本6，由于待定更改，因此我们从版本3（在OpenSSH中找到的版本）开始，并从那里开始

#ifndef LIBSSH2_SFTP_H
#define LIBSSH2_SFTP_H 1

#include "libssh2.h"

#ifndef WIN32
#include <unistd.h>
#endif

#ifdef __cplusplus
extern "C" {
#endif

# 注意：在编写时，版本6已经记录
# 但由于待定更改，它被标记为“不要实现”
# 让我们从版本3（在OpenSSH中找到的版本）开始，并从那里开始
#define LIBSSH2_SFTP_VERSION        3

typedef struct _LIBSSH2_SFTP                LIBSSH2_SFTP;
typedef struct _LIBSSH2_SFTP_HANDLE         LIBSSH2_SFTP_HANDLE;
typedef struct _LIBSSH2_SFTP_ATTRIBUTES     LIBSSH2_SFTP_ATTRIBUTES;
typedef struct _LIBSSH2_SFTP_STATVFS        LIBSSH2_SFTP_STATVFS;

/* Flags for open_ex() */
#define LIBSSH2_SFTP_OPENFILE           0
#define LIBSSH2_SFTP_OPENDIR            1

/* Flags for rename_ex() */
#define LIBSSH2_SFTP_RENAME_OVERWRITE   0x00000001
#define LIBSSH2_SFTP_RENAME_ATOMIC      0x00000002
#define LIBSSH2_SFTP_RENAME_NATIVE      0x00000004

/* Flags for stat_ex() */
#define LIBSSH2_SFTP_STAT               0
#define LIBSSH2_SFTP_LSTAT              1
#define LIBSSH2_SFTP_SETSTAT            2

/* Flags for symlink_ex() */
#define LIBSSH2_SFTP_SYMLINK            0
#define LIBSSH2_SFTP_READLINK           1
#define LIBSSH2_SFTP_REALPATH           2

/* Flags for sftp_mkdir() */
#define LIBSSH2_SFTP_DEFAULT_MODE      -1

/* SFTP attribute flag bits */
#define LIBSSH2_SFTP_ATTR_SIZE              0x00000001
#define LIBSSH2_SFTP_ATTR_UIDGID            0x00000002
#define LIBSSH2_SFTP_ATTR_PERMISSIONS       0x00000004
#define LIBSSH2_SFTP_ATTR_ACMODTIME         0x00000008
#define LIBSSH2_SFTP_ATTR_EXTENDED          0x80000000

/* SFTP statvfs flag bits */
#define LIBSSH2_SFTP_ST_RDONLY              0x00000001
#define LIBSSH2_SFTP_ST_NOSUID              0x00000002

struct _LIBSSH2_SFTP_ATTRIBUTES {
    /* If flags & ATTR_* bit is set, then the value in this struct will be
     * meaningful Otherwise it should be ignored
     */
    unsigned long flags;

    libssh2_uint64_t filesize;
    unsigned long uid, gid;
    unsigned long permissions;
    unsigned long atime, mtime;
};

struct _LIBSSH2_SFTP_STATVFS {
    libssh2_uint64_t  f_bsize;    /* file system block size */
    libssh2_uint64_t  f_frsize;   /* fragment size */
    libssh2_uint64_t  f_blocks;   /* size of fs in f_frsize units */
    # 可用的空闲块数
    libssh2_uint64_t  f_bfree;    /* # free blocks */
    # 非 root 用户可用的空闲块数
    libssh2_uint64_t  f_bavail;   /* # free blocks for non-root */
    # 文件节点数
    libssh2_uint64_t  f_files;    /* # inodes */
    # 可用的空闲文件节点数
    libssh2_uint64_t  f_ffree;    /* # free inodes */
    # 非 root 用户可用的空闲文件节点数
    libssh2_uint64_t  f_favail;   /* # free inodes for non-root */
    # 文件系统 ID
    libssh2_uint64_t  f_fsid;     /* file system ID */
    # 挂载标志
    libssh2_uint64_t  f_flag;     /* mount flags */
    # 最大文件名长度
    libssh2_uint64_t  f_namemax;  /* maximum filename length */
/* SFTP 文件类型 */
#define LIBSSH2_SFTP_TYPE_REGULAR           1   // 普通文件
#define LIBSSH2_SFTP_TYPE_DIRECTORY         2   // 目录
#define LIBSSH2_SFTP_TYPE_SYMLINK           3   // 符号链接
#define LIBSSH2_SFTP_TYPE_SPECIAL           4   // 特殊文件
#define LIBSSH2_SFTP_TYPE_UNKNOWN           5   // 未知类型
#define LIBSSH2_SFTP_TYPE_SOCKET            6   // 套接字
#define LIBSSH2_SFTP_TYPE_CHAR_DEVICE       7   // 字符设备
#define LIBSSH2_SFTP_TYPE_BLOCK_DEVICE      8   // 块设备
#define LIBSSH2_SFTP_TYPE_FIFO              9   // 先进先出

/*
 * 在这里重现 POSIX 文件模式，用于不符合 POSIX 标准的系统
 *
 * 这些用于 "struct _LIBSSH2_SFTP_ATTRIBUTES" 的 "permissions" 字段
 */
/* 文件类型 */
#define LIBSSH2_SFTP_S_IFMT         0170000     // 文件类型掩码
#define LIBSSH2_SFTP_S_IFIFO        0010000     // 命名管道（FIFO）
#define LIBSSH2_SFTP_S_IFCHR        0020000     // 字符设备
#define LIBSSH2_SFTP_S_IFDIR        0040000     // 目录
#define LIBSSH2_SFTP_S_IFBLK        0060000     // 块设备
#define LIBSSH2_SFTP_S_IFREG        0100000     // 普通文件
#define LIBSSH2_SFTP_S_IFLNK        0120000     // 符号链接
#define LIBSSH2_SFTP_S_IFSOCK       0140000     // 套接字

/* 文件模式 */
/* 所有者的读、写、执行/搜索权限 */
#define LIBSSH2_SFTP_S_IRWXU        0000700     // 所有者的 RWX 掩码
#define LIBSSH2_SFTP_S_IRUSR        0000400     // 所有者的读权限
#define LIBSSH2_SFTP_S_IWUSR        0000200     // 所有者的写权限
#define LIBSSH2_SFTP_S_IXUSR        0000100     // 所有者的执行权限
/* 组的读、写、执行/搜索权限 */
#define LIBSSH2_SFTP_S_IRWXG        0000070     // 组的 RWX 掩码
#define LIBSSH2_SFTP_S_IRGRP        0000040     // 组的读权限
#define LIBSSH2_SFTP_S_IWGRP        0000020     // 组的写权限
#define LIBSSH2_SFTP_S_IXGRP        0000010     // 组的执行权限
/* 其他用户的读、写、执行/搜索权限 */
#define LIBSSH2_SFTP_S_IRWXO        0000007     // 其他用户的 RWX 掩码
#define LIBSSH2_SFTP_S_IROTH        0000004     // 其他用户的读权限
# 定义 SFTP 文件权限掩码，表示其他用户的写权限
#define LIBSSH2_SFTP_S_IWOTH        0000002     /* W for other */
# 定义 SFTP 文件权限掩码，表示其他用户的执行权限
#define LIBSSH2_SFTP_S_IXOTH        0000001     /* X for other */

# 定义宏，用于检查文件类型是否为符号链接
#define LIBSSH2_SFTP_S_ISLNK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFLNK)
# 定义宏，用于检查文件类型是否为普通文件
#define LIBSSH2_SFTP_S_ISREG(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFREG)
# 定义宏，用于检查文件类型是否为目录
#define LIBSSH2_SFTP_S_ISDIR(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFDIR)
# 定义宏，用于检查文件类型是否为字符设备
#define LIBSSH2_SFTP_S_ISCHR(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFCHR)
# 定义宏，用于检查文件类型是否为块设备
#define LIBSSH2_SFTP_S_ISBLK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFBLK)
# 定义宏，用于检查文件类型是否为管道
#define LIBSSH2_SFTP_S_ISFIFO(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFIFO)
# 定义宏，用于检查文件类型是否为套接字
#define LIBSSH2_SFTP_S_ISSOCK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFSOCK)

# 定义 SFTP 文件传输标志，用于 sftp_open() 函数的 flags 参数
#define LIBSSH2_FXF_READ                        0x00000001
#define LIBSSH2_FXF_WRITE                       0x00000002
#define LIBSSH2_FXF_APPEND                      0x00000004
#define LIBSSH2_FXF_CREAT                       0x00000008
#define LIBSSH2_FXF_TRUNC                       0x00000010
#define LIBSSH2_FXF_EXCL                        0x00000020

# 定义 SFTP 状态码，由 libssh2_sftp_last_error() 返回
#define LIBSSH2_FX_OK                       0UL
#define LIBSSH2_FX_EOF                      1UL
#define LIBSSH2_FX_NO_SUCH_FILE             2UL
#define LIBSSH2_FX_PERMISSION_DENIED        3UL
#define LIBSSH2_FX_FAILURE                  4UL
#define LIBSSH2_FX_BAD_MESSAGE              5UL
#define LIBSSH2_FX_NO_CONNECTION            6UL
#define LIBSSH2_FX_CONNECTION_LOST          7UL
#define LIBSSH2_FX_OP_UNSUPPORTED           8UL
#define LIBSSH2_FX_INVALID_HANDLE           9UL
#define LIBSSH2_FX_NO_SUCH_PATH             10UL
#define LIBSSH2_FX_FILE_ALREADY_EXISTS      11UL
/* 定义 SFTP 错误代码 */
#define LIBSSH2_FX_WRITE_PROTECT            12UL
#define LIBSSH2_FX_NO_MEDIA                 13UL
#define LIBSSH2_FX_NO_SPACE_ON_FILESYSTEM   14UL
#define LIBSSH2_FX_QUOTA_EXCEEDED           15UL
#define LIBSSH2_FX_UNKNOWN_PRINCIPLE        16UL /* 初始拼写错误 */
#define LIBSSH2_FX_UNKNOWN_PRINCIPAL        16UL
#define LIBSSH2_FX_LOCK_CONFlICT            17UL /* 初始拼写错误 */
#define LIBSSH2_FX_LOCK_CONFLICT            17UL
#define LIBSSH2_FX_DIR_NOT_EMPTY            18UL
#define LIBSSH2_FX_NOT_A_DIRECTORY          19UL
#define LIBSSH2_FX_INVALID_FILENAME         20UL
#define LIBSSH2_FX_LINK_LOOP                21UL

/* 由任何在读/写操作期间会阻塞的函数返回 */
#define LIBSSH2SFTP_EAGAIN LIBSSH2_ERROR_EAGAIN

/* SFTP API */
/* 初始化 SFTP 会话 */
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session);
/* 关闭 SFTP 会话 */
LIBSSH2_API int libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp);
/* 获取最后一次 SFTP 错误代码 */
LIBSSH2_API unsigned long libssh2_sftp_last_error(LIBSSH2_SFTP *sftp);
/* 获取 SFTP 通道 */
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp);

/* 文件/目录操作 */
/* 打开文件或目录 */
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_open_ex(LIBSSH2_SFTP *sftp,
                     const char *filename,
                     unsigned int filename_len,
                     unsigned long flags,
                     long mode, int open_type);
/* 宏定义，打开文件 */
#define libssh2_sftp_open(sftp, filename, flags, mode)                  \
    libssh2_sftp_open_ex((sftp), (filename), strlen(filename), (flags), \
                         (mode), LIBSSH2_SFTP_OPENFILE)
/* 宏定义，打开目录 */
#define libssh2_sftp_opendir(sftp, path) \
    libssh2_sftp_open_ex((sftp), (path), strlen(path), 0, 0, \
                         LIBSSH2_SFTP_OPENDIR)
/* 读取文件内容 */
LIBSSH2_API ssize_t libssh2_sftp_read(LIBSSH2_SFTP_HANDLE *handle,
                                      char *buffer, size_t buffer_maxlen);
# 读取指定 SFTP 目录句柄的内容，并将结果存储在 buffer 中，同时返回文件属性
LIBSSH2_API int libssh2_sftp_readdir_ex(LIBSSH2_SFTP_HANDLE *handle, \
                                        char *buffer, size_t buffer_maxlen,
                                        char *longentry,
                                        size_t longentry_maxlen,
                                        LIBSSH2_SFTP_ATTRIBUTES *attrs);
# 定义 libssh2_sftp_readdir 的宏，简化调用 libssh2_sftp_readdir_ex 的过程
#define libssh2_sftp_readdir(handle, buffer, buffer_maxlen, attrs)      \
    libssh2_sftp_readdir_ex((handle), (buffer), (buffer_maxlen), NULL, 0, \
                            (attrs))

# 向 SFTP 文件句柄中写入数据
LIBSSH2_API ssize_t libssh2_sftp_write(LIBSSH2_SFTP_HANDLE *handle,
                                       const char *buffer, size_t count);
# 将 SFTP 文件句柄中的数据同步到磁盘
LIBSSH2_API int libssh2_sftp_fsync(LIBSSH2_SFTP_HANDLE *handle);

# 关闭 SFTP 文件句柄
LIBSSH2_API int libssh2_sftp_close_handle(LIBSSH2_SFTP_HANDLE *handle);
# 定义 libssh2_sftp_close 和 libssh2_sftp_closedir 的宏，简化调用 libssh2_sftp_close_handle 的过程
#define libssh2_sftp_close(handle) libssh2_sftp_close_handle(handle)
#define libssh2_sftp_closedir(handle) libssh2_sftp_close_handle(handle)

# 移动 SFTP 文件句柄的读写位置到指定偏移量
LIBSSH2_API void libssh2_sftp_seek(LIBSSH2_SFTP_HANDLE *handle, size_t offset);
# 64 位版本的移动 SFTP 文件句柄的读写位置到指定偏移量
LIBSSH2_API void libssh2_sftp_seek64(LIBSSH2_SFTP_HANDLE *handle,
                                     libssh2_uint64_t offset);
# 定义 libssh2_sftp_rewind 的宏，简化调用 libssh2_sftp_seek64 的过程
#define libssh2_sftp_rewind(handle) libssh2_sftp_seek64((handle), 0)

# 获取 SFTP 文件句柄的当前读写位置
LIBSSH2_API size_t libssh2_sftp_tell(LIBSSH2_SFTP_HANDLE *handle);
# 64 位版本的获取 SFTP 文件句柄的当前读写位置
LIBSSH2_API libssh2_uint64_t libssh2_sftp_tell64(LIBSSH2_SFTP_HANDLE *handle);

# 获取 SFTP 文件句柄的文件属性
LIBSSH2_API int libssh2_sftp_fstat_ex(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_ATTRIBUTES *attrs,
                                      int setstat);
# 定义 libssh2_sftp_fstat 和 libssh2_sftp_fsetstat 的宏，简化调用 libssh2_sftp_fstat_ex 的过程
#define libssh2_sftp_fstat(handle, attrs) \
    libssh2_sftp_fstat_ex((handle), (attrs), 0)
#define libssh2_sftp_fsetstat(handle, attrs) \
    libssh2_sftp_fstat_ex((handle), (attrs), 1)

# 其他操作
# 重命名远程 SFTP 服务器上的文件或目录
LIBSSH2_API int libssh2_sftp_rename_ex(LIBSSH2_SFTP *sftp,
                                       const char *source_filename,
                                       unsigned int srouce_filename_len,
                                       const char *dest_filename,
                                       unsigned int dest_filename_len,
                                       long flags);
# 定义重命名文件或目录的宏，调用 libssh2_sftp_rename_ex 函数
#define libssh2_sftp_rename(sftp, sourcefile, destfile) \
    libssh2_sftp_rename_ex((sftp), (sourcefile), strlen(sourcefile), \
                           (destfile), strlen(destfile),                \
                           LIBSSH2_SFTP_RENAME_OVERWRITE | \
                           LIBSSH2_SFTP_RENAME_ATOMIC | \
                           LIBSSH2_SFTP_RENAME_NATIVE)

# 删除远程 SFTP 服务器上的文件
LIBSSH2_API int libssh2_sftp_unlink_ex(LIBSSH2_SFTP *sftp,
                                       const char *filename,
                                       unsigned int filename_len);
# 定义删除文件的宏，调用 libssh2_sftp_unlink_ex 函数
#define libssh2_sftp_unlink(sftp, filename) \
    libssh2_sftp_unlink_ex((sftp), (filename), strlen(filename))

# 获取远程 SFTP 服务器上文件系统的信息
LIBSSH2_API int libssh2_sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_STATVFS *st);

# 获取远程 SFTP 服务器上指定路径文件系统的信息
LIBSSH2_API int libssh2_sftp_statvfs(LIBSSH2_SFTP *sftp,
                                     const char *path,
                                     size_t path_len,
                                     LIBSSH2_SFTP_STATVFS *st);

# 在远程 SFTP 服务器上创建目录
LIBSSH2_API int libssh2_sftp_mkdir_ex(LIBSSH2_SFTP *sftp,
                                      const char *path,
                                      unsigned int path_len, long mode);
# 定义创建目录的宏，调用 libssh2_sftp_mkdir_ex 函数
#define libssh2_sftp_mkdir(sftp, path, mode) \
    libssh2_sftp_mkdir_ex((sftp), (path), strlen(path), (mode))

# 在远程 SFTP 服务器上删除目录
LIBSSH2_API int libssh2_sftp_rmdir_ex(LIBSSH2_SFTP *sftp,
                                      const char *path,
                                      unsigned int path_len);
# 定义删除目录的宏，调用 libssh2_sftp_rmdir_ex 函数
#define libssh2_sftp_rmdir(sftp, path) \
    # 使用libssh2库的SFTP功能删除指定路径的目录
    libssh2_sftp_rmdir_ex((sftp), (path), strlen(path))
// 定义 libssh2_sftp_stat_ex 函数，用于获取指定路径的文件信息
LIBSSH2_API int libssh2_sftp_stat_ex(LIBSSH2_SFTP *sftp,
                                     const char *path,
                                     unsigned int path_len,
                                     int stat_type,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);
// 定义宏 libssh2_sftp_stat，用于获取指定路径的文件信息
#define libssh2_sftp_stat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_STAT, \
                         (attrs))
// 定义宏 libssh2_sftp_lstat，用于获取指定路径的文件信息（不跟随符号链接）
#define libssh2_sftp_lstat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_LSTAT, \
                         (attrs))
// 定义宏 libssh2_sftp_setstat，用于设置指定路径的文件信息
#define libssh2_sftp_setstat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_SETSTAT, \
                         (attrs))

// 定义 libssh2_sftp_symlink_ex 函数，用于创建符号链接
LIBSSH2_API int libssh2_sftp_symlink_ex(LIBSSH2_SFTP *sftp,
                                        const char *path,
                                        unsigned int path_len,
                                        char *target,
                                        unsigned int target_len,
                                        int link_type);
// 定义宏 libssh2_sftp_symlink，用于创建符号链接
#define libssh2_sftp_symlink(sftp, orig, linkpath) \
    libssh2_sftp_symlink_ex((sftp), (orig), strlen(orig), (linkpath), \
                            strlen(linkpath), LIBSSH2_SFTP_SYMLINK)
// 定义宏 libssh2_sftp_readlink，用于读取符号链接的目标路径
#define libssh2_sftp_readlink(sftp, path, target, maxlen) \
    libssh2_sftp_symlink_ex((sftp), (path), strlen(path), (target), (maxlen), \
    LIBSSH2_SFTP_READLINK)
// 定义宏 libssh2_sftp_realpath，用于获取符号链接的真实路径
#define libssh2_sftp_realpath(sftp, path, target, maxlen) \
    libssh2_sftp_symlink_ex((sftp), (path), strlen(path), (target), (maxlen), \
                            LIBSSH2_SFTP_REALPATH)

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* LIBSSH2_SFTP_H */
```