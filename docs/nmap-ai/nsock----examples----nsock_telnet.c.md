# `nmap\nsock\examples\nsock_telnet.c`

```
#include "nsock.h"

#include <stdio.h>
#include <stdlib.h>
#include <sys.socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <assert.h>
#include <sys/time.h>

#include <openssl/ssl.h>
#include <openssl/err.h>
/* #include <nbase.h> */

/* from nbase.h */
int socket_errno();


extern char *optarg;

extern int optind;

struct telnet_state {
  nsock_iod tcp_nsi;
  nsock_iod stdin_nsi;
  nsock_event_id latest_readtcpev;
  nsock_event_id latest_readstdinev;
  void *ssl_session;
};

/* Tries to resolve given hostname and stores
   result in ip .  returns 0 if hostname cannot
   be resolved */
int resolve(char *hostname, struct in_addr *ip) {
  struct hostent *h;

  if (!hostname || !*hostname) {
    fprintf(stderr, "NULL or zero-length hostname passed to resolve().  Quitting.\n");
    exit(1);
  }

  if (inet_aton(hostname, ip))
    return 1;                   /* damn, that was easy ;) */
  if ((h = gethostbyname(hostname))) {
    memcpy(ip, h->h_addr_list[0], sizeof(struct in_addr));
    return 1;
  }
  return 0;
}

void telnet_event_handler(nsock_pool nsp, nsock_event nse, void *mydata) {
  nsock_iod nsi = nse_iod(nse);
  enum nse_status status = nse_status(nse);
  enum nse_type type = nse_type(nse);
  struct sockaddr_in peer;
  struct telnet_state *ts;
  int nbytes;
  char *str;
  int read_timeout = -1;
  int write_timeout = 2000;
  ts = (struct telnet_state *)mydata;

  printf("telnet_event_handler: Received callback of type %s with status %s\n", nse_type2str(type), nse_status2str(status));

  if (status == NSE_STATUS_SUCCESS) {
    switch (type) {
    case NSE_TYPE_CONNECT:
    // 处理连接成功的情况
    # 如果连接类型是 NSE_TYPE_CONNECT_SSL
    case NSE_TYPE_CONNECT_SSL:
      # 获取通信信息
      nsock_iod_get_communication_info(nsi, NULL, NULL, NULL, (struct sockaddr *)&peer, sizeof peer);
      # 打印连接成功信息
      printf("Successfully connected %sto %s:%hu -- start typing lines\n", (type == NSE_TYPE_CONNECT_SSL) ? "(SSL!) " : "", inet_ntoa(peer.sin_addr), ntohs(peer.sin_port));
      /* 首先，将标准输入添加到我们关注的文件句柄列表中 */
      if ((ts->stdin_nsi = nsock_iod_new2(nsp, STDIN_FILENO, NULL)) == NULL) {
        fprintf(stderr, "Failed to create stdin msi\n");
        exit(1);
      }

      /* 现在让我们从标准输入和网络中读取，由 nsock 进行行缓冲 */
      ts->latest_readtcpev = nsock_readlines(nsp, ts->tcp_nsi, telnet_event_handler, read_timeout, ts, 1);
      ts->latest_readstdinev = nsock_readlines(nsp, ts->stdin_nsi, telnet_event_handler, read_timeout, ts, 1);
      break;
    # 如果连接类型是 NSE_TYPE_READ
    case NSE_TYPE_READ:
      # 从 nse 中读取数据
      str = nse_readbuf(nse, &nbytes);
      if (nsi == ts->tcp_nsi) {
        # 如果是从 tcp_nsi 中读取的数据，则打印数据
        printf("%s", str);
        /*       printf("Read from tcp socket (%d bytes):\n%s", nbytes, str); */
        ts->latest_readtcpev = nsock_readlines(nsp, ts->tcp_nsi, telnet_event_handler, read_timeout, ts, 1);
      } else {
        # 如果是从 stdin_nsi 中读取的数据，则将数据写入到 tcp_nsi 中
        /*       printf("Read from  stdin (%d bytes):\n%s", nbytes, str); */
        nsock_write(nsp, ts->tcp_nsi, telnet_event_handler, write_timeout, ts, str, nbytes);
        ts->latest_readstdinev = nsock_readlines(nsp, ts->stdin_nsi, telnet_event_handler, read_timeout, ts, 1);
      }
      break;
    # 如果连接类型是 NSE_TYPE_WRITE
    case NSE_TYPE_WRITE:
      # 没有什么需要做的
      break;
    # 如果连接类型是 NSE_TYPE_TIMER
    case NSE_TYPE_TIMER:
      # 没有什么需要做的
      break;
    # 如果连接类型是其他
    default:
      # 打印错误信息并退出
      fprintf(stderr, "telnet_event_handler: Got bogus type -- quitting\n");
      exit(1);
      break;
    }
  # 如果状态是 NSE_STATUS_EOF
  } else if (status == NSE_STATUS_EOF) {
    # 打印从哪里收到 EOF，并取消未完成的读事件
    printf("Got EOF from %s\nCancelling outstanding readevents.\n", (nsi == ts->tcp_nsi) ? "tcp socket" : "stdin");
    /* 其中一个是我当前正在处理的事件！但我想在测试时搞点小动作... */
    # 如果最近的读取 TCP 事件被取消，则执行以下代码
    if (nsock_event_cancel(nsp, ts->latest_readtcpev, 1) != 0) {
      # 打印取消的 TCP 事件信息
      printf("Cancelled tcp event: %li\n", ts->latest_readtcpev);
    }
    # 如果最近的读取标准输入事件被取消，则执行以下代码
    if (nsock_event_cancel(nsp, ts->latest_readstdinev, 1) != 0) {
      # 打印取消的标准输入事件信息
      printf("Cancelled stdin event: %li\n", ts->latest_readstdinev);
    }
  } else if (status == NSE_STATUS_ERROR) {
    # 如果状态为错误状态，则执行以下代码
    if (nsock_iod_check_ssl(nsi)) {
      # 如果是 SSL 错误，则打印 SSL 错误信息
      printf("SSL %s failed: %s\n", nse_type2str(type), ERR_error_string(ERR_get_error(), NULL));
    } else {
      # 否则，获取错误码并打印错误信息
      int err;
      err = nse_errorcode(nse);
      printf("%s failed: (%d) %s\n", nse_type2str(type), err, strerror(err));
    }
  }
  # 返回
  return;
}

void usage() {
  // 打印程序的使用方法
  fprintf(stderr, "\nUsage: nsock_telnet [-s] <hostnameorip> [portnum]\n" "       Where -s enables SSL for the connection\n\n");
  // 退出程序并返回错误码1
  exit(1);
}

int main(int argc, char *argv[]) {
  // 定义变量
  struct in_addr target;
  nsock_pool nsp;
  nsock_event_id ev;
  unsigned short portno;
  enum nsock_loopstatus loopret;
  struct telnet_state ts;
  int c;
  int usessl = 0;
  struct timeval now;
  struct sockaddr_in taddr;

  // 初始化 telnet_state 结构体中的 stdin_nsi 字段
  ts.stdin_nsi = NULL;

  // 解析命令行参数
  while ((c = getopt(argc, argv, "s")) != -1) {
    switch (c) {
    case 's':
      // 设置使用 SSL 标志
      usessl = 1;
      break;
    default:
      // 调用 usage 函数打印使用方法并退出程序
      usage();
      break;
    }
  }

  // 检查命令行参数数量是否合法
  if (argc - optind <= 0 || argc - optind > 2)
    // 调用 usage 函数打印使用方法并退出程序
    usage();

  // 解析目标主机名或 IP 地址
  if (!resolve(argv[optind], &target)) {
    // 打印解析失败的错误信息并退出程序
    fprintf(stderr, "Failed to resolve target host: %s\nQUITTING.\n", argv[optind]);
    exit(1);
  }
  optind++;

  // 解析端口号
  if (optind < argc)
    portno = atoi(argv[optind]);
  else
    portno = 23;

  /* OK, we start with creating a p00l */
  // 创建一个新的 nsock_pool 对象
  if ((nsp = nsock_pool_new(NULL)) == NULL) {
    // 打印创建失败的错误信息并退出程序
    fprintf(stderr, "Failed to create new pool.  QUITTING.\n");
    exit(1);
  }

  // 获取当前时间
  gettimeofday(&now, NULL);

  // 创建一个新的 nsock_iod 对象
  if ((ts.tcp_nsi = nsock_iod_new(nsp, NULL)) == NULL) {
    // 打印创建失败的错误信息并退出程序
    fprintf(stderr, "Failed to create new nsock_iod.  QUITTING.\n");
    exit(1);
  }

  // 设置目标地址和端口
  taddr.sin_family = AF_INET;
  taddr.sin_addr = target;
  taddr.sin_port = portno;

  // 根据是否使用 SSL 进行连接
  if (usessl) {
    ts.ssl_session = NULL;
    // 进行 SSL 连接
    ev = nsock_connect_ssl(nsp, ts.tcp_nsi, telnet_event_handler, 10000, &ts, (struct sockaddr *)&taddr, sizeof taddr, IPPROTO_TCP, portno, ts.ssl_session);
  } else
    // 进行 TCP 连接
    ev = nsock_connect_tcp(nsp, ts.tcp_nsi, telnet_event_handler, 10000, &ts, (struct sockaddr *)&taddr, sizeof taddr, portno);

  // 打印事件 ID
  printf("The event id is %lu -- initiating l00p\n", ev);

  /* Now lets get this party started right! */
  // 进入事件循环
  loopret = nsock_loop(nsp, -1);

  // 打印事件循环的返回值
  printf("nsock_loop returned %d\n", (int)loopret);

  // 返回成功
  return 0;
}
```