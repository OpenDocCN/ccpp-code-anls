# `nmap\ncat\ncat_listen.c`

```cpp
/* $Id$ */

#include "ncat.h"

#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <limits.h>
#ifndef WIN32
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#else
#include <fcntl.h>
#endif

#if HAVE_SYS_UN_H
#include <sys/un.h>
#endif

#ifdef HAVE_OPENSSL
#include <openssl/ssl.h>
#include <openssl/err.h>
#endif

#ifdef WIN32
/* Define missing constant for shutdown(2).
 * See:
 * http://msdn.microsoft.com/en-us/library/windows/desktop/ms740481%28v=vs.85%29.aspx
 */
#define SHUT_WR SD_SEND
#endif

/* read_fds is the clients we are accepting data from. broadcast_fds is the
   clients were are sending data to. broadcast_fds doesn't include the listening
   socket and stdin. Network clients are not added to read_fds when --send-only
   is used, because they would be always selected without having data read.
   write_fds is the list of clients that are waiting for some kind of response
   from us, like a pending ssl negotiation. */
static fd_set master_readfds, master_writefds, master_broadcastfds;
#ifdef HAVE_OPENSSL
/* sslpending_fds contains the list of ssl sockets that are waiting to complete
   the ssl handshake */
static fd_set sslpending_fds;
#endif

/* These are bookkeeping data structures that are parallel to read_fds and
   broadcast_fds. */
static fd_list_t client_fdlist, broadcast_fdlist;

static int listen_socket[NUM_LISTEN_ADDRS];
/* Has stdin seen EOF? */
static int stdin_eof = 0;
static int crlf_state = 0;

static void handle_connection(int socket_accept, int type, fd_set *listen_fds);
static int read_stdin(void);
static int read_socket(int recv_fd);
static void post_handle_connection(struct fdinfo *sinfo);
static void close_fd(struct fdinfo *fdn, int eof);
static void read_and_broadcast(int recv_socket);
static void shutdown_sockets(int how);
static int chat_announce_connect(const struct fdinfo *fdi);
/* 声明一个静态函数，用于通知客户端断开连接 */
static int chat_announce_disconnect(int fd);
/* 声明一个静态函数，用于过滤聊天内容 */
static char *chat_filter(char *buf, size_t size, int fd, int *nwritten);

/* 连接的客户端数量是 conn_inc 和 conn_dec 的差值。
   为了信号安全起见，将其分为两个变量。conn_dec 只在信号处理程序中异步修改，
   而 conn_inc 只在主程序中同步修改。get_conn_count 在 conn_dec 被修改时循环。 */
static unsigned int conn_inc = 0;
static volatile unsigned int conn_dec = 0;
static volatile sig_atomic_t conn_dec_changed;

static void decrease_conn_count(void)
{
    conn_dec_changed = 1;
    conn_dec++;
}

static int get_conn_count(void)
{
    unsigned int count;

    /* conn_dec 在信号处理程序中被修改，因此循环直到它停止改变。 */
    do {
        conn_dec_changed = 0;
        count = conn_inc - conn_dec;
    } while (conn_dec_changed);
    ncat_assert(count <= INT_MAX);

    return count;
}

#ifndef WIN32
static void sigchld_handler(int signum)
{
    while (waitpid(-1, NULL, WNOHANG) > 0)
        decrease_conn_count();
}
#endif

/* 创建一个新的监听套接字 */
int new_listen_socket(int type, int proto, const union sockaddr_u *addr, fd_set *listen_fds)
{
  struct fdinfo fdi = {0};
  fdi.fd = do_listen(type, proto, addr);
  if (fdi.fd < 0) {
    return -1;
  }
  fdi.remoteaddr = *addr; /* 实际上是我们的本地地址，但无所谓 */

  /* 将监听套接字设置为非阻塞，因为存在定时问题，可能导致在 select() 表示它可读的情况下在 accept() 上阻塞。
   * 更多信息请参见 UNPv1 2nd ed, p422。 */
  unblock_socket(fdi.fd);

  /* 设置 select 集合和最大 fd */
  checked_fd_set(fdi.fd, &master_readfds);
  add_fdinfo(&client_fdlist, &fdi);

  checked_fd_set(fdi.fd, listen_fds);

  return fdi.fd;
}

int ncat_listen()
{
    int rc, i, j, fds_ready;
    fd_set listen_fds;
    struct timeval tv;
    struct timeval *tvp = NULL;
    unsigned int num_sockets;
    int proto = o.proto;
    # 根据协议类型设置套接字类型，如果是UDP协议则使用数据报套接字，否则使用流套接字
    int type = o.proto == IPPROTO_UDP ? SOCK_DGRAM : SOCK_STREAM;

    # 如果设置了HTTP服务器选项，则调用ncat_http_server()函数
    if (o.httpserver)
        return ncat_http_server();
#if HAVE_SYS_UN_H
    # 如果系统支持 UNIX 域套接字，且地址族为 AF_UNIX，则协议为 0
    if (o.af == AF_UNIX)
        proto = 0;
#endif
#if HAVE_LINUX_VM_SOCKETS_H
    # 如果系统支持 Linux VM 套接字，且地址族为 AF_VSOCK，则协议为 0
    if (o.af == AF_VSOCK)
        proto = 0;
#endif
    /* 清空结构体 */
    FD_ZERO(&master_readfds);
    FD_ZERO(&master_writefds);
    FD_ZERO(&master_broadcastfds);
    FD_ZERO(&listen_fds);
#ifdef HAVE_OPENSSL
    FD_ZERO(&sslpending_fds);
#endif
    // 将 client_fdlist 结构体清零
    zmem(&client_fdlist, sizeof(client_fdlist));
    // 将 broadcast_fdlist 结构体清零
    zmem(&broadcast_fdlist, sizeof(broadcast_fdlist));

#ifdef WIN32
    // 设置伪 SIGCHLD 处理程序为 decrease_conn_count
    set_pseudo_sigchld_handler(decrease_conn_count);
#else
    /* 处理 SIGCHLD 信号 */
    Signal(SIGCHLD, sigchld_handler);
    /* 忽略 SIGPIPE 信号，当客户端突然断开连接并且我们在注意到之前发送数据时会发生 */
    Signal(SIGPIPE, SIG_IGN);
#endif

#ifdef HAVE_OPENSSL
    if (o.ssl)
    {
        if (o.sslalpn)
            bye("ALPN is not supported in listen mode\n");
        // 设置 SSL 监听
        setup_ssl_listen(type == SOCK_STREAM ? SSLv23_server_method() : DTLS_server_method());
    }
#endif

/* 不确定这个问题是否存在于 Windows 上，但 fcntl 和 /dev/null 不存在 */
#ifndef WIN32
    /* 检查 stdin 是否关闭。因为我们特殊对待这个文件描述符，我们不能冒险它被重新打开用于传入连接，所以我们将保持它打开。 */
    if (fcntl(STDIN_FILENO, F_GETFD) == -1 && errno == EBADF) {
      logdebug("stdin is closed, attempting to reserve STDIN_FILENO\n");
      rc = open("/dev/null", O_RDONLY);
      if (rc >= 0 && rc != STDIN_FILENO) {
        /* 哦，我们尝试了 */
        logdebug("Couldn't reserve STDIN_FILENO\n");
        close(rc);
      }
    }
#endif

    /* 我们需要一个 fd 列表来保持当前的 fdmax。第二个参数是添加到提供的连接限制的数字，将补偿默认的监听和 stdin 套接字添加到 maxfds。 */
    init_fdlist(&client_fdlist, sadd(o.conn_limit, num_listenaddrs + 1));

    for (i = 0; i < NUM_LISTEN_ADDRS; i++)
        listen_socket[i] = -1;
    # 初始化变量 num_sockets 为 0
    num_sockets = 0;
    # 遍历监听地址的数量
    for (i = 0; i < num_listenaddrs; i++) {
        # 设置主监听套接字
        listen_socket[num_sockets] = new_listen_socket(type, proto, &listenaddrs[i], &listen_fds);
        # 如果监听套接字创建失败
        if (listen_socket[num_sockets] == -1) {
            # 如果调试模式开启，记录错误日志
            if (o.debug > 0)
                logdebug("do_listen(\"%s\"): %s\n", socktop(&listenaddrs[i], 0), socket_strerror(socket_errno()));
            # 继续下一次循环
            continue;
        }
        # 增加已创建的套接字数量
        num_sockets++;
    }
    # 如果没有创建任何监听套接字
    if (num_sockets == 0) {
        # 如果只有一个监听地址，记录错误日志并退出程序
        if (num_listenaddrs == 1)
            bye("Unable to open listening socket on %s: %s", socktop(&listenaddrs[0], 0), socket_strerror(socket_errno()));
        # 如果有多个监听地址，记录错误日志并退出程序
        else
            bye("Unable to open any listening sockets.");
    }

    # 将标准输入文件描述符添加到客户端文件描述符列表中
    add_fd(&client_fdlist, STDIN_FILENO);

    # 初始化广播文件描述符列表，设置连接限制
    init_fdlist(&broadcast_fdlist, o.conn_limit);

    # 如果空闲超时时间大于 0，将 tvp 指向 tv 变量
    if (o.idletimeout > 0)
        tvp = &tv;
    # 当客户端文件描述符列表中的描述符数量大于1或者当前连接数大于0时，执行循环
    while (client_fdlist.nfds > 1 || get_conn_count() > 0) {
        /* We pass these temporary descriptor sets to fselect, since fselect
           modifies the sets it receives. */
        # 将临时的描述符集传递给fselect，因为fselect会修改它接收到的集合
        fd_set readfds = master_readfds, writefds = master_writefds;

        # 如果调试级别大于1，则记录选择的最大文件描述符数
        if (o.debug > 1)
            logdebug("selecting, fdmax %d\n", client_fdlist.fdmax);

        # 如果调试级别大于1且o.broker为真，则记录经纪人连接数
        if (o.debug > 1 && o.broker)
            logdebug("Broker connection count is %d\n", get_conn_count());

        # 如果空闲超时时间大于0，则将毫秒转换为时间结构
        if (o.idletimeout > 0)
            ms_to_timeval(tvp, o.idletimeout);

        # 当有活动连接时，空闲定时器应该在运行
        if (get_conn_count())
            fds_ready = fselect(client_fdlist.fdmax + 1, &readfds, &writefds, NULL, tvp);
        else
            fds_ready = fselect(client_fdlist.fdmax + 1, &readfds, &writefds, NULL, NULL);

        # 如果调试级别大于1，则记录选择返回的准备好的文件描述符数
        if (o.debug > 1)
            logdebug("select returned %d fds ready\n", fds_ready);

        # 如果没有准备好的文件描述符，则退出程序并记录空闲超时已过期
        if (fds_ready == 0)
            bye("Idle timeout expired (%d ms).", o.idletimeout);

        /* If client_fdlist.state increases, the list has changed and we
         * need to go over it again. */
# 重新开始文件描述符循环
restart_fd_loop:
        # 重置客户端文件描述符列表的状态
        client_fdlist.state = 0;
        # 遍历客户端文件描述符列表，直到找到准备好的文件描述符
        for (i = 0; i < client_fdlist.nfds && fds_ready > 0; i++) {
            # 获取当前文件描述符的信息
            struct fdinfo *fdi = &client_fdlist.fds[i];
            # 获取当前文件描述符的值
            int cfd = fdi->fd;
            # 如果之前发生过错误，关闭该文件描述符并重新开始文件描述符循环
            if (fdi->lasterr != 0) {
                close_fd(fdi, 0);
                goto restart_fd_loop;
            }
            # 如果文件描述符既没有可读事件也没有可写事件，则继续下一个文件描述符
            if (!checked_fd_isset(cfd, &readfds) && !checked_fd_isset(cfd, &writefds))
                continue;

            # 如果调试级别大于1，则记录文件描述符准备就绪的日志信息
            if (o.debug > 1)
                logdebug("fd %d is ready\n", cfd);
#ifdef HAVE_OPENSSL
            /* 检查是否有 OpenSSL 支持，如果是 SSL 套接字且需要握手，则处理握手过程 */
            if (o.ssl && checked_fd_isset(cfd, &sslpending_fds)) {
                checked_fd_clr(cfd, &master_readfds);  // 清除读取文件描述符集合中的 cfd
                checked_fd_clr(cfd, &master_writefds);  // 清除写入文件描述符集合中的 cfd
                switch (ssl_handshake(fdi)) {
                case NCAT_SSL_HANDSHAKE_COMPLETED:
                    /* SSL 建立完成后，从 sslpending_fds 中清除 cfd */
                    checked_fd_clr(cfd, &sslpending_fds);
                    post_handle_connection(fdi);  // 处理连接后续操作
                    break;
                case NCAT_SSL_HANDSHAKE_PENDING_WRITE:
                    checked_fd_set(cfd, &master_writefds);  // 将 cfd 加入写入文件描述符集合
                    break;
                case NCAT_SSL_HANDSHAKE_PENDING_READ:
                    checked_fd_set(cfd, &master_readfds);  // 将 cfd 加入读取文件描述符集合
                    break;
                case NCAT_SSL_HANDSHAKE_FAILED:
                default:
                    SSL_free(fdi->ssl);  // 释放 SSL 对象
                    Close(fdi->fd);  // 关闭文件描述符
                    checked_fd_clr(cfd, &sslpending_fds);  // 从 sslpending_fds 中清除 cfd
                    checked_fd_clr(cfd, &master_readfds);  // 从读取文件描述符集合中清除 cfd
                    rm_fd(&client_fdlist, cfd);  // 从客户端文件描述符列表中移除 cfd
                    /* 如果是单一监听模式（没有 -k 参数），则退出 */
                    if (!o.keepopen && !o.broker)
                        return 1;
                    --conn_inc;  // 连接数减一
                    break;
                }
            } else
#endif
            // 如果文件描述符在监听文件描述符集合中
            if (checked_fd_isset(cfd, &listen_fds)) {
                // 处理新的连接请求
                handle_connection(cfd, type, &listen_fds);
            } else if (cfd == STDIN_FILENO) {
                if (o.broker) {
                    // 从标准输入读取并广播
                    read_and_broadcast(cfd);
                } else {
                    // 从标准输入读取并写入所有客户端
                    rc = read_stdin();
                    if (rc == 0 && type == SOCK_STREAM) {
                        if (o.proto != IPPROTO_TCP || (o.proto == IPPROTO_TCP && o.sendonly)) {
                            // 不会再有更多要发送的内容。如果我们不接收任何内容，可以在这里退出。
                            return 0;
                        }
                        if (!o.noshutdown) shutdown_sockets(SHUT_WR);
                    }
                    if (rc < 0)
                        return 1;
                }
            } else if (!o.sendonly) {
                if (o.broker) {
                    // 从客户端读取并广播
                    read_and_broadcast(cfd);
                } else {
                    // 从客户端读取并写入标准输出
                    rc = read_socket(cfd);
                    if (rc <= 0 && !o.keepopen)
                        return rc == 0 ? 0 : 1;
                }
            }

            // 减少准备好的文件描述符数量
            fds_ready--;
            if (client_fdlist.state > 0)
                // 重新开始文件描述符循环
                goto restart_fd_loop;

            // 检查是否记录了任何发送错误
            for (j = 0; j < broadcast_fdlist.nfds; j++) {
                fdi = &broadcast_fdlist.fds[j];
                if (fdi->lasterr != 0) {
                    close_fd(fdi, 0);
                    // close_fd 会影响 client_fdlist，所以跳回并重新开始循环
                    goto restart_fd_loop;
                }
            }
        }
    }

    return 0;
}
/* 接受一个监听套接字上的连接。允许或拒绝连接。
   如果 o.cmdexec 被设置，就 fork 一个命令。否则，将新套接字添加到监视集合中。 */
static void handle_connection(int socket_accept, int type, fd_set *listen_fds)
{
    // 创建一个 fdinfo 结构体，并初始化为 0
    struct fdinfo s = { 0 };
    int conn_count;

    // 将 s 的内存清零
    zmem(&s, sizeof(s));

    // 设置 s.remoteaddr.storage 的长度
    s.ss_len = sizeof(s.remoteaddr.storage);

    // 设置 errno 为 0
    errno = 0;
    // 如果类型为 SOCK_STREAM
    if (type == SOCK_STREAM) {
      // 接受连接，将新连接的套接字存储在 s.fd 中
      s.fd = accept(socket_accept, &s.remoteaddr.sockaddr, &s.ss_len);
    }
    // 否则
    else {
      // 创建一个长度为 4 的缓冲区
      char buf[4] = {0};
      // 从 socket_accept 接收数据，但不移除数据，存储在 buf 中
      int nbytes = recvfrom(socket_accept, buf, sizeof(buf), MSG_PEEK,
          &s.remoteaddr.sockaddr, &s.ss_len);
      // 如果接收失败
      if (nbytes < 0) {
        // 记录错误信息并返回
        loguser("%s.\n", socket_strerror(socket_errno()));
        return;
      }
      /*
       * 我们正在使用连接的 UDP。这样做的缺点是一次只能处理一个 UDP 客户端
       */
      // 连接到指定的地址
      Connect(socket_accept, &s.remoteaddr.sockaddr, s.ss_len);
      // 将新连接的套接字存储在 s.fd 中
      s.fd = socket_accept;
      /* 如果我们期望新的连接，我们将不得不打开一个新的监听套接字来替换刚刚连接到单个客户端的套接字。 */
      if ((o.keepopen || o.broker)
#if HAVE_SYS_UN_H
          /* 除非它是一个 UNIX 套接字，因为当我们尝试绑定时，我们会收到 EADDRINUSE */
          && s.remoteaddr.storage.ss_family != AF_UNIX
#endif
        ) {
        int i;
        for (i = 0; i < num_listenaddrs; i++) {
          if (listen_socket[i] == socket_accept) {
            // 获取客户端文件描述符的信息
            struct fdinfo *lfdi = get_fdinfo(&client_fdlist, socket_accept);
            // 获取本地地址信息
            union sockaddr_u localaddr = lfdi->remoteaddr;
            // 创建新的监听套接字
            listen_socket[i] = new_listen_socket(type, (o.af == AF_INET || o.af == AF_INET6) ? o.proto : 0, &localaddr, listen_fds);
            // 如果创建失败，则打印错误信息并返回
            if (listen_socket[i] < 0) {
              bye("do_listen(\"%s\"): %s\n", socktop(&listenaddrs[i], 0), socket_strerror(socket_errno()));
              return;
            }
            // 中断循环
            break;
          }
        }
      }
      /* Remove this socket from listening */
      // 从主监听文件描述符集合中清除该套接字
      checked_fd_clr(socket_accept, &master_readfds);
      // 从监听文件描述符集合中清除该套接字
      checked_fd_clr(socket_accept, listen_fds);
      // 从客户端文件描述符列表中移除该套接字
      rm_fd(&client_fdlist, socket_accept);
    }

    // 如果文件描述符小于0，则表示接受连接出现错误
    if (s.fd < 0) {
        // 如果开启了调试模式，则记录错误信息
        if (o.debug)
            logdebug("Error in accept: %s\n", strerror(errno));

        // 关闭文件描述符并返回
        close(s.fd);
        return;
    }

    // 如果不保持连接打开且不是代理模式
    if (!o.keepopen && !o.broker) {
        int i;
        for (i = 0; i < num_listenaddrs; i++) {
            /* If */
            // 如果监听套接字大于等于0且在监听文件描述符集合中
            if (listen_socket[i] >= 0 && checked_fd_isset(listen_socket[i], listen_fds)) {
              // 关闭监听套接字
              Close(listen_socket[i]);
              // 从主监听文件描述符集合中清除该套接字
              checked_fd_clr(listen_socket[i], &master_readfds);
              // 从客户端文件描述符列表中移除该套接字
              rm_fd(&client_fdlist, listen_socket[i]);
              // 将监听套接字置为-1
              listen_socket[i] = -1;
            }
        }
    }

    // 如果开启了详细模式，则记录连接信息
    if (o.verbose) {
        loguser("Connection from %s", socktop(&s.remoteaddr, s.ss_len));
        if (o.chat)
            loguser_noprefix(" on file descriptor %d", s.fd);
        loguser_noprefix(".\n");
    }

    /* Check conditions that might cause us to deny the connection. */
    // 获取当前连接数
    conn_count = get_conn_count();
    // 如果连接数超过限制，则拒绝连接
    if (conn_count >= o.conn_limit) {
        if (o.verbose)
            loguser("New connection denied: connection limit reached (%d)\n", conn_count);
        // 关闭文件描述符并返回
        Close(s.fd);
        return;
    }
    # 如果不允许访问当前远程地址，则执行以下操作
    if (!allow_access(&s.remoteaddr)) {
        # 如果设置了详细日志模式，则记录拒绝连接的信息
        if (o.verbose)
            loguser("New connection denied: not allowed\n");
        # 关闭当前套接字
        Close(s.fd);
        # 返回
        return;
    }

    # 连接计数增加
    conn_inc++;

    # 解除套接字的阻塞状态
    unblock_socket(s.fd);
#ifdef HAVE_OPENSSL
    # 如果使用了 OpenSSL，则将套接字添加到必要的描述符列表中
    checked_fd_set(s.fd, &sslpending_fds)
    checked_fd_set(s.fd, &master_readfds)
    checked_fd_set(s.fd, &master_writefds)
    # 同时也将其添加到维护 maxfd 的套接字列表中
    if (add_fdinfo(&client_fdlist, &s) < 0)
        bye("add_fdinfo() failed.")
    # 如果没有使用 OpenSSL，则执行 post_handle_connection 函数
#else
        post_handle_connection(&s)
#endif

/* 此函数处理连接后特定的操作，需要在套接字初始化后执行（普通套接字或 SSL 套接字） */
static void post_handle_connection(struct fdinfo *sinfo)
{
    /*
     * 是否正在执行命令？如果是，则不将此套接字添加到描述符列表或集合中
     */
    if (o.cmdexec) {
#ifdef HAVE_OPENSSL
      /* 在 handle_connection 中添加了此套接字，但在此时 SSL 连接已经接管了。停止跟踪 */
      if (o.ssl) {
        rm_fd(&client_fdlist, sinfo->fd)
      }
#endif
        # 如果保持连接打开，则运行 netrun 函数，否则运行 netexec 函数
        if (o.keepopen)
            netrun(sinfo, o.cmdexec)
        else
            netexec(sinfo, o.cmdexec)
    } else {
        # 现在客户端已连接，注意 stdin
        if (!stdin_eof)
            checked_fd_set(STDIN_FILENO, &master_readfds)
        if (!o.sendonly) {
            # 将套接字添加到列表中
            checked_fd_set(sinfo->fd, &master_readfds)
            # 将其添加到维护 maxfd 的套接字列表中
#ifdef HAVE_OPENSSL
            # 不要重复添加（参见上面的 handle_connection）
            if (!o.ssl) {
#endif
            if (add_fdinfo(&client_fdlist, sinfo) < 0)
                bye("add_fdinfo() failed.")
#ifdef HAVE_OPENSSL
            }
#endif
        }
        checked_fd_set(sinfo->fd, &master_broadcastfds)
        if (add_fdinfo(&broadcast_fdlist, sinfo) < 0)
            bye("add_fdinfo() failed.")

        # 如果启用了聊天功能，则通知连接
        if (o.chat)
            chat_announce_connect(sinfo)
    }
}
static void close_fd(struct fdinfo *fdn, int eof) {
    /* rm_fd invalidates fdn, so save what we need here. */
    // 从 fdinfo 结构中获取文件描述符
    int fd = fdn->fd;
    // 如果开启了调试模式，记录连接关闭信息
    if (o.debug)
        logdebug("Closing connection.\n");
#ifdef HAVE_OPENSSL
    // 如果开启了 OpenSSL，并且存在 SSL 对象，执行 SSL 关闭和释放操作
    if (o.ssl && fdn->ssl) {
        if (eof)
            SSL_shutdown(fdn->ssl);
        SSL_free(fdn->ssl);
    }
#endif
    // 关闭文件描述符
    Close(fd);
    // 从 master_readfds 集合中清除文件描述符
    checked_fd_clr(fd, &master_readfds);
    // 从 client_fdlist 链表中移除文件描述符
    rm_fd(&client_fdlist, fd);
    // 从 master_broadcastfds 集合中清除文件描述符
    checked_fd_clr(fd, &master_broadcastfds);
    // 从 broadcast_fdlist 链表中移除文件描述符
    rm_fd(&broadcast_fdlist, fd);

    // 连接数减一
    conn_inc--;
    // 如果连接数为 0，从 master_readfds 集合中清除标准输入文件描述符
    if (get_conn_count() == 0)
        checked_fd_clr(STDIN_FILENO, &master_readfds);

    // 如果开启了聊天模式，通知客户端连接断开
    if (o.chat)
        chat_announce_disconnect(fd);
}

/* Read from stdin and broadcast to all client sockets. Return the number of
   bytes read, or -1 on error. */
// 从标准输入读取数据，并广播到所有客户端套接字，返回读取的字节数，出错时返回-1
int read_stdin(void)
{
    int nbytes;
    char buf[DEFAULT_TCP_BUF_LEN];
    char *tempbuf = NULL;

    // 从标准输入读取数据到 buf 中
    nbytes = read(STDIN_FILENO, buf, sizeof(buf));
    // 如果读取的字节数小于等于 0
    if (nbytes <= 0) {
        // 如果读取出错并且开启了详细模式，记录错误信息
        if (nbytes < 0 && o.verbose)
            logdebug("Error reading from stdin: %s\n", strerror(errno));
        // 如果读取到 EOF 并且开启了调试模式，记录信息
        if (nbytes == 0 && o.debug)
            logdebug("EOF on stdin\n");

        /* Don't close the file because that allows a socket to be fd 0. */
        // 不关闭文件，因为这样可以让套接字成为文件描述符 0
        checked_fd_clr(STDIN_FILENO, &master_readfds);
        // 标记已经读取到 EOF，以防止重新添加到 select 列表中
        stdin_eof = 1;

        return nbytes;
    }

    // 如果开启了 CRLF 转换，对读取的数据进行处理
    if (o.crlf)
        fix_line_endings((char *) buf, &nbytes, &tempbuf, &crlf_state);

    // 如果开启了行延迟，执行延迟操作
    if (o.linedelay)
        ncat_delay_timer(o.linedelay);

    /* Write to everything in the broadcast set. */
    // 如果 tempbuf 不为空，广播 tempbuf 中的数据到所有客户端套接字
    if (tempbuf != NULL) {
        ncat_broadcast(&master_broadcastfds, &broadcast_fdlist, tempbuf, nbytes);
        free(tempbuf);
        tempbuf = NULL;
    } else {
        // 否则，广播 buf 中的数据到所有客户端套接字
        ncat_broadcast(&master_broadcastfds, &broadcast_fdlist, buf, nbytes);
    }

    return nbytes;
}
/* 从客户端套接字读取数据并写入标准输出。返回从套接字读取的字节数，或在出错时返回-1。 */
int read_socket(int recv_fd)
{
    char buf[DEFAULT_TCP_BUF_LEN];  // 创建一个缓冲区
    struct fdinfo *fdn;  // 定义一个文件描述符信息结构体指针
    int nbytes, pending;  // 定义整型变量nbytes和pending

    fdn = get_fdinfo(&client_fdlist, recv_fd);  // 获取客户端套接字的文件描述符信息
    ncat_assert(fdn != NULL);  // 断言文件描述符信息不为空

    nbytes = 0;  // 初始化nbytes为0
    do {
        int n;  // 定义整型变量n

        n = ncat_recv(fdn, buf, sizeof(buf), &pending);  // 从套接字接收数据
        if (n <= 0) {
            /* 返回值为0时，并不意味着已经到达文件末尾，有些情况下比如SSL重新协商需要读写套接字操作但没有应用数据。 */
            if(n == 0 && fdn->lasterr == 0) {
                continue;  // 检查是否有待处理的数据
            }
            close_fd(fdn, n == 0);  // 关闭套接字
            return n;  // 返回n
        }
        else {
            Write(STDOUT_FILENO, buf, n);  // 将数据写入标准输出
            nbytes += n;  // 更新nbytes
        }
    } while (pending);  // 当有待处理的数据时循环

    return nbytes;  // 返回nbytes
}

//---------------
/* 从recv_fd读取数据，并将读取的内容广播到read_fds中的所有其他描述符，但不包括stdin、listen_socket和recv_fd本身。
   处理EOL转换和聊天模式。在读取错误或流结束时，关闭套接字并从read_fds列表中移除。 */
static void read_and_broadcast(int recv_fd)
{
    struct fdinfo *fdn;  // 定义一个文件描述符信息结构体指针
    int pending;  // 定义整型变量pending

    fdn = get_fdinfo(&client_fdlist, recv_fd);  // 获取客户端套接字的文件描述符信息
    ncat_assert(fdn != NULL);  // 断言文件描述符信息不为空

    /* 当ncat_recv指示有待处理数据时循环。 */
    } while (pending);
}

static void shutdown_sockets(int how)
{
    struct fdinfo *fdn;  // 定义一个文件描述符信息结构体指针
    int i;  // 定义整型变量i

    for (i = 0; i <= broadcast_fdlist.fdmax; i++) {  // 遍历广播套接字列表
        if (!checked_fd_isset(i, &master_broadcastfds))  // 如果套接字未设置
            continue;  // 继续下一次循环

        fdn = get_fdinfo(&broadcast_fdlist, i);  // 获取广播套接字的文件描述符信息
        ncat_assert(fdn != NULL);  // 断言文件描述符信息不为空
#ifdef HAVE_OPENSSL
        if (o.ssl && fdn->ssl) {  // 如果使用SSL并且套接字有SSL
                SSL_shutdown(fdn->ssl);  // 关闭SSL连接
        }
        else
#endif
            shutdown(fdn->fd, how);  // 关闭套接字
    }
}
/* 宣布新连接以及已连接的用户 */
static int chat_announce_connect(const struct fdinfo *fdi)
{
    char *buf = NULL; // 初始化一个字符指针，用于存储消息内容
    size_t size = 0, offset = 0; // 初始化消息内容的大小和偏移量
    int i, count, ret; // 初始化循环变量和返回值

    // 格式化连接消息，并将其存储到buf中
    strbuf_sprintf(&buf, &size, &offset,
        "<announce> %s is connected as <user%d>.\n", socktop(&fdi->remoteaddr, fdi->ss_len), fdi->fd);

    // 添加已连接用户的消息前缀
    strbuf_sprintf(&buf, &size, &offset, "<announce> already connected: ");
    count = 0;
    for (i = 0; i <= client_fdlist.fdmax; i++) {
        union sockaddr_u tsu;
        socklen_t len = sizeof(tsu.storage);

        if (i == fdi->fd || !checked_fd_isset(i, &master_broadcastfds))
            continue;

        // 获取已连接用户的地址信息，并格式化消息内容
        if (getpeername(i, &tsu.sockaddr, &len) == -1)
            bye("getpeername for sd %d failed: %s.", i, strerror(errno));

        if (count > 0)
            strbuf_sprintf(&buf, &size, &offset, ", ");

        strbuf_sprintf(&buf, &size, &offset, "%s as <user%d>", socktop(&tsu, len), i);

        count++;
    }
    // 如果没有已连接用户，则添加相应消息
    if (count == 0)
        strbuf_sprintf(&buf, &size, &offset, "nobody");
    strbuf_sprintf(&buf, &size, &offset, ".\n");

    // 广播连接消息给所有客户端
    ret = ncat_broadcast(&master_broadcastfds, &broadcast_fdlist, buf, offset);

    // 释放buf的内存
    free(buf);

    return ret; // 返回广播结果
}

// 宣布断开连接
static int chat_announce_disconnect(int fd)
{
    char buf[128]; // 初始化一个长度为128的字符数组
    int n;

    // 格式化断开连接消息
    n = Snprintf(buf, sizeof(buf),
        "<announce> <user%d> is disconnected.\n", fd);
    if (n < 0 || n >= sizeof(buf))
        return -1;

    // 广播断开连接消息给所有客户端
    return ncat_broadcast(&master_broadcastfds, &broadcast_fdlist, buf, n);
}

/*
 * 这很愚蠢。但这只是一点乐趣。
 *
 * 发件人的文件描述符被添加到发送给客户端的消息中，这样你就可以以一定程度的理智区分彼此。这给出了类似IRC会话的效果。但更愚蠢。
 */
static char *chat_filter(char *buf, size_t size, int fd, int *nwritten)
{
    char *result = NULL; // 初始化一个字符指针，用于存储处理后的消息内容
    size_t n = 0; // 初始化一个大小变量
    const char *p;
    int i;

    n = 32; // 初始化n的值为32
    result = (char *) safe_malloc(n); // 分配n大小的内存空间
    # 使用 Snprintf 函数将格式化后的字符串写入 result 中，返回写入的字符数
    i = Snprintf(result, n, "<user%d> ", fd);

    # 对 buf 中的控制字符进行转义处理
    for (p = buf; p - buf < size; p++) {
        char repl[32];
        int repl_len;

        # 判断字符是否可打印，或者是回车、换行、制表符
        if (isprint((int) (unsigned char) *p) || *p == '\r' || *p == '\n' || *p == '\t') {
            repl[0] = *p;
            repl_len = 1;
        } else {
            # 对非可打印字符进行转义处理
            repl_len = Snprintf(repl, sizeof(repl), "\\%03o", (unsigned char) *p);
        }

        # 如果替换后的字符串长度超过了 n，则扩大 n 的大小
        if (i + repl_len > n) {
            n = (i + repl_len) * 2;
            result = (char *) safe_realloc(result, n + 1);
        }
        # 将替换后的字符串拷贝到 result 中
        memcpy(result + i, repl, repl_len);
        i += repl_len;
    }
    # 调整 result 的大小，使其长度为 i，并在末尾添加结束符'\0'
    result = (char *) safe_realloc(result, i + 1);
    result[i] = '\0';

    # 将写入的字符数赋值给 nwritten
    *nwritten = i;

    # 返回处理后的字符串
    return result;
# 闭合前面的函数定义
```