# `nmap\libssh2\src\sftp.h`

```cpp
#ifndef __LIBSSH2_SFTP_H
#define __LIBSSH2_SFTP_H
/*
 * 版权声明
 * 作者信息
 * 允许源代码和二进制形式的再发布和使用的条件
 * 版权声明和免责声明的保留
 * 未经特定事先书面许可，不得使用版权持有者的名称或其他贡献者的名称来认可或推广基于此软件的产品
 * 免责声明
 */

/*
 * MAX_SFTP_OUTGOING_SIZE 不得大于32500。这是每个FXP_WRITE数据包发送的数据量
 */
#define MAX_SFTP_OUTGOING_SIZE 30000

/* MAX_SFTP_READ_SIZE 是每个FXP_READ数据包中最多请求的数据量 */
#define MAX_SFTP_READ_SIZE 30000
// 定义 SFTP 传输管道的数据块结构体
struct sftp_pipeline_chunk {
    struct list_node node; // 链表节点
    libssh2_uint64_t offset; // 读取时：开始读取的偏移量；写入时：未使用
    size_t len; // 写入时：要写入的数据大小；读取时：请求的数据大小
    size_t sent; // 已发送的数据大小
    ssize_t lefttosend; // 如果为 0，则整个数据包已发送
    uint32_t request_id; // 请求 ID
    unsigned char packet[1]; // 数据
};

// 定义 SFTP 僵尸请求结构体
struct sftp_zombie_requests {
    struct list_node node; // 链表节点
    uint32_t request_id; // 请求 ID
};

// 定义最小值宏
#ifndef MIN
#define MIN(x,y) ((x)<(y)?(x):(y))
#endif

// 定义 SFTP 数据包结构体
struct _LIBSSH2_SFTP_PACKET
{
    struct list_node node; // 链表节点
    uint32_t request_id; // 请求 ID
    unsigned char *data; // 数据
    size_t data_len; // 数据大小
};

// 定义 SFTP 句柄最大长度
#define SFTP_HANDLE_MAXLEN 256 // 根据规范定义的最大长度

// 定义 SFTP 句柄结构体
struct _LIBSSH2_SFTP_HANDLE
{
    struct list_node node; // 链表节点

    LIBSSH2_SFTP *sftp; // SFTP 对象指针

    char handle[SFTP_HANDLE_MAXLEN]; // 句柄数据
    size_t handle_len; // 句柄数据长度

    enum {
        LIBSSH2_SFTP_HANDLE_FILE, // 文件句柄
        LIBSSH2_SFTP_HANDLE_DIR // 目录句柄
    } handle_type; // 句柄类型

    union _libssh2_sftp_handle_data
    {
        // 定义结构体 _libssh2_sftp_handle_file_data，用于存储文件相关的数据
        struct _libssh2_sftp_handle_file_data
        {
            libssh2_uint64_t offset; // 文件偏移量
            libssh2_uint64_t offset_sent; // 已发送的偏移量
            size_t acked; /* 用于存储已确认但尚未返回给调用者的数据，用于 sftp_write */

            /* 'data' 用于 sftp_read()，存储已从服务器接收但尚未返回给调用者的分配数据。
               其大小为 'data_len'，'data_left' 是从缓冲区末尾开始计算的尚未返回的字节数。 */
            unsigned char *data; // 数据
            size_t data_len; // 数据长度
            size_t data_left; // 剩余数据长度

            char eof; // 是否已读取到文件末尾
        } file; // 文件相关数据

        // 定义结构体 _libssh2_sftp_handle_dir_data，用于存储目录相关的数据
        struct _libssh2_sftp_handle_dir_data
        {
            uint32_t names_left; // 剩余文件名数量
            void *names_packet; // 文件名数据包
            char *next_name; // 下一个文件名
            size_t names_packet_len; // 文件名数据包长度
        } dir; // 目录相关数据
    } u; // 结构体实例

    /* 用于 libssh2_sftp_close_handle() 中的状态变量 */
    libssh2_nonblocking_states close_state; // 关闭状态
    uint32_t close_request_id; // 关闭请求 ID
    unsigned char *close_packet; // 关闭数据包

    /* 发送到服务器的未完成数据包列表 */
    struct list_head packet_list; // 数据包列表
};  // 结构体定义结束

struct _LIBSSH2_SFTP
{
    LIBSSH2_CHANNEL *channel;  // SFTP 通道

    uint32_t request_id, version;  // 请求 ID 和 SFTP 版本号

    struct list_head packets;  // 数据包列表

    /* List of FXP_READ responses to ignore because EOF already received. */
    struct list_head zombie_requests;  // 需要忽略的 FXP_READ 响应列表，因为已经收到了 EOF

    /* a list of _LIBSSH2_SFTP_HANDLE structs */
    struct list_head sftp_handles;  // _LIBSSH2_SFTP_HANDLE 结构体的列表

    uint32_t last_errno;  // 最后的错误码

    /* Holder for partial packet, use in libssh2_sftp_packet_read() */
    unsigned char partial_size[4];      /* buffer for size field   */  // 部分数据包的大小字段缓冲区
    size_t partial_size_len;            /* size field length       */  // 大小字段的长度
    unsigned char *partial_packet;      /* The data                */  // 部分数据包的数据
    uint32_t partial_len;               /* Desired number of bytes */  // 期望的字节数
    size_t partial_received;            /* Bytes received so far   */  // 到目前为止接收的字节数

    /* Time that libssh2_sftp_packet_requirev() started reading */
    time_t requirev_start;  // libssh2_sftp_packet_requirev() 开始读取的时间

    /* State variables used in libssh2_sftp_open_ex() */
    libssh2_nonblocking_states open_state;  // 在 libssh2_sftp_open_ex() 中使用的状态变量
    unsigned char *open_packet;  // 打开数据包
    uint32_t open_packet_len; /* 32 bit on the wire */  // 数据包长度
    size_t open_packet_sent;  // 已发送的数据包长度
    uint32_t open_request_id;  // 打开请求的 ID

    /* State variable used in sftp_read() */
    libssh2_nonblocking_states read_state;  // 在 sftp_read() 中使用的状态变量

    /* State variable used in sftp_packet_read() */
    libssh2_nonblocking_states packet_state;  // 在 sftp_packet_read() 中使用的状态变量

    /* State variable used in sftp_write() */
    libssh2_nonblocking_states write_state;  // 在 sftp_write() 中使用的状态变量

    /* State variables used in sftp_fsync() */
    libssh2_nonblocking_states fsync_state;  // 在 sftp_fsync() 中使用的状态变量
    unsigned char *fsync_packet;  // fsync 数据包
    uint32_t fsync_request_id;  // fsync 请求的 ID

    /* State variables used in libssh2_sftp_readdir() */
    libssh2_nonblocking_states readdir_state;  // 在 libssh2_sftp_readdir() 中使用的状态变量
    unsigned char *readdir_packet;  // readdir 数据包
    uint32_t readdir_request_id;  // readdir 请求的 ID

    /* State variables used in libssh2_sftp_fstat_ex() */
    libssh2_nonblocking_states fstat_state;  // 在 libssh2_sftp_fstat_ex() 中使用的状态变量
    unsigned char *fstat_packet;  // fstat 数据包
    uint32_t fstat_request_id;  // fstat 请求的 ID

    /* State variables used in libssh2_sftp_unlink_ex() */
    libssh2_nonblocking_states unlink_state;  // 在 libssh2_sftp_unlink_ex() 中使用的状态变量
    # 指向要解除链接的数据包的指针
    unsigned char *unlink_packet;
    # 解除链接请求的ID
    uint32_t unlink_request_id;

    # 在libssh2_sftp_rename_ex()中使用的状态变量
    libssh2_nonblocking_states rename_state;
    # 重命名数据包的指针
    unsigned char *rename_packet;
    # 重命名源路径
    unsigned char *rename_s;
    # 重命名请求的ID
    uint32_t rename_request_id;

    # 在libssh2_sftp_fstatvfs()中使用的状态变量
    libssh2_nonblocking_states fstatvfs_state;
    # fstatvfs数据包的指针
    unsigned char *fstatvfs_packet;
    # fstatvfs请求的ID
    uint32_t fstatvfs_request_id;

    # 在libssh2_sftp_statvfs()中使用的状态变量
    libssh2_nonblocking_states statvfs_state;
    # statvfs数据包的指针
    unsigned char *statvfs_packet;
    # statvfs请求的ID
    uint32_t statvfs_request_id;

    # 在libssh2_sftp_mkdir()中使用的状态变量
    libssh2_nonblocking_states mkdir_state;
    # mkdir数据包的指针
    unsigned char *mkdir_packet;
    # mkdir请求的ID
    uint32_t mkdir_request_id;

    # 在libssh2_sftp_rmdir()中使用的状态变量
    libssh2_nonblocking_states rmdir_state;
    # rmdir数据包的指针
    unsigned char *rmdir_packet;
    # rmdir请求的ID
    uint32_t rmdir_request_id;

    # 在libssh2_sftp_stat()中使用的状态变量
    libssh2_nonblocking_states stat_state;
    # stat数据包的指针
    unsigned char *stat_packet;
    # stat请求的ID
    uint32_t stat_request_id;

    # 在libssh2_sftp_symlink()中使用的状态变量
    libssh2_nonblocking_states symlink_state;
    # symlink数据包的指针
    unsigned char *symlink_packet;
    # symlink请求的ID
    uint32_t symlink_request_id;
};
#endif /* __LIBSSH2_SFTP_H */

这段代码是C/C++中的预处理指令，用于结束一个条件编译块。`#endif`表示结束条件编译块，`__LIBSSH2_SFTP_H`是条件编译的标识符，表示如果该标识符已经定义过，则结束条件编译块。
```