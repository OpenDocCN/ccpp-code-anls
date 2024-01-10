# `nmap\nsock\src\nsock_write.c`

```
/* $Id$ */

#include "nsock.h"  // 包含自定义的网络套接字头文件
#include "nsock_internal.h"  // 包含自定义的网络套接字内部头文件
#include "nsock_log.h"  // 包含自定义的网络套接字日志头文件
#include "netutils.h"  // 包含自定义的网络工具头文件

#include <nbase.h>  // 包含自定义的基础网络头文件
#include <stdarg.h>  // 包含标准参数头文件
#include <errno.h>  // 包含错误处理头文件

nsock_event_id nsock_sendto(nsock_pool ms_pool, nsock_iod ms_iod, nsock_ev_handler handler, int timeout_msecs,
                            void *userdata, struct sockaddr *saddr, size_t sslen, unsigned short port, const char *data, int datalen) {
  struct npool *nsp = (struct npool *)ms_pool;  // 定义并初始化网络池指针
  struct niod *nsi = (struct niod *)ms_iod;  // 定义并初始化网络 IOD 指针
  struct nevent *nse;  // 定义网络事件指针
  char displaystr[256];  // 定义用于显示的字符串数组
  struct sockaddr_in *sin = (struct sockaddr_in *)saddr;  // 定义并初始化 IPv4 地址结构体指针
#if HAVE_IPV6
  struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *)saddr;  // 定义并初始化 IPv6 地址结构体指针
#endif

  nse = event_new(nsp, NSE_TYPE_WRITE, nsi, timeout_msecs, handler, userdata);  // 创建新的网络事件
  assert(nse);  // 断言网络事件非空

  if (saddr->sa_family == AF_INET) {  // 如果地址族为 IPv4
    sin->sin_port = htons(port);  // 将端口号转换为网络字节顺序
#if HAVE_SYS_UN_H
  } else if (saddr->sa_family == AF_INET6) {  // 如果地址族为 IPv6
#else
  } else {  // 如果没有 IPv6 支持
#endif
    assert(saddr->sa_family == AF_INET6);  // 断言地址族为 IPv6
#if HAVE_IPV6
    sin6->sin6_port = htons(port);  // 将端口号转换为网络字节顺序
#else
    fatal("IPv6 address passed to %s call, but nsock was not compiled w/IPv6 support", __func__);  // 输出错误信息
#endif
  }

  assert(sslen <= sizeof(nse->writeinfo.dest));  // 断言目标地址长度不超过指定长度
  memcpy(&nse->writeinfo.dest, saddr, sslen);  // 复制目标地址信息
  nse->writeinfo.destlen = sslen;  // 设置目标地址长度

  assert(sslen <= sizeof(nse->iod->peer));  // 断言对等地址长度不超过指定长度
  memcpy(&nse->iod->peer, saddr, sslen);  // 复制对等地址信息
  nse->iod->peerlen = sslen;  // 设置对等地址长度

  if (datalen < 0)  // 如果数据长度小于 0
    datalen = (int) strlen(data);  // 计算数据长度

  if (NsockLogLevel == NSOCK_LOG_DBG_ALL && datalen < 80) {  // 如果日志级别为调试且数据长度小于 80
    memcpy(displaystr, ": ", 2);  // 复制字符串
    memcpy(displaystr + 2, data, datalen);  // 复制数据
    displaystr[2 + datalen] = '\0';  // 设置字符串结束符
    replacenonprintable(displaystr + 2, datalen, '.');  // 替换不可打印字符
  } else {
    displaystr[0] = '\0';  // 设置空字符串
  }
  nsock_log_info("Sendto request for %d bytes to IOD #%li EID %li [%s]%s",
                 datalen, nsi->id, nse->id, get_peeraddr_string(nse->iod),
                 displaystr);  // 记录发送请求信息

  fs_cat(&nse->iobuf, data, datalen);  // 将数据追加到事件的输入缓冲区

  nsock_pool_add_event(nsp, nse);  // 将事件添加到网络池中

  return nse->id;  // 返回事件 ID
}
/* Write some data to the socket.  If the write is not COMPLETED within
 * timeout_msecs , NSE_STATUS_TIMEOUT will be returned.  If you are supplying
 * NUL-terminated data, you can optionally pass -1 for datalen and nsock_write
 * will figure out the length itself */
# 将一些数据写入套接字。如果在 timeout_msecs 内未完成写入，将返回 NSE_STATUS_TIMEOUT。如果您提供的是以 NUL 结尾的数据，可以选择将 datalen 设置为 -1，nsock_write 将自行计算长度

nsock_event_id nsock_write(nsock_pool ms_pool, nsock_iod ms_iod,
          nsock_ev_handler handler, int timeout_msecs, void *userdata, const char *data, int datalen) {
  struct npool *nsp = (struct npool *)ms_pool;
  struct niod *nsi = (struct niod *)ms_iod;
  struct nevent *nse;
  char displaystr[256];

  nse = event_new(nsp, NSE_TYPE_WRITE, nsi, timeout_msecs, handler, userdata);
  assert(nse);

  nse->writeinfo.dest.ss_family = AF_UNSPEC;

  if (datalen < 0)
    datalen = (int)strlen(data);

  if (NsockLogLevel == NSOCK_LOG_DBG_ALL && datalen < 80) {
    memcpy(displaystr, ": ", 2);
    memcpy(displaystr + 2, data, datalen);
    displaystr[2 + datalen] = '\0';
    replacenonprintable(displaystr + 2, datalen, '.');
  } else {
    displaystr[0] = '\0';
  }

  nsock_log_info("Write request for %d bytes to IOD #%li EID %li [%s]%s",
      datalen, nsi->id, nse->id, get_peeraddr_string(nsi),
      displaystr);

  fs_cat(&nse->iobuf, data, datalen);

  nsock_pool_add_event(nsp, nse);

  return nse->id;
}

/* Same as nsock_write except you can use a printf-style format and you can only use this for ASCII strings */
# 与 nsock_write 相同，除了您可以使用 printf 样式的格式，并且只能用于 ASCII 字符串
# 定义一个函数，用于向套接字池中的套接字发送格式化的数据，并返回事件ID
nsock_event_id nsock_printf(nsock_pool ms_pool, nsock_iod ms_iod,
          nsock_ev_handler handler, int timeout_msecs, void *userdata, char *format, ...) {
  # 将套接字池和套接字转换为相应的结构体指针
  struct npool *nsp = (struct npool *)ms_pool;
  struct niod *nsi = (struct niod *)ms_iod;
  struct nevent *nse;
  char buf[4096];  # 定义一个缓冲区
  char *buf2 = NULL;  # 定义另一个缓冲区指针，并初始化为空
  size_t buf2size;  # 定义缓冲区大小
  int res, res2;  # 定义整型变量
  int strlength = 0;  # 初始化字符串长度为0
  char displaystr[256];  # 定义显示字符串的缓冲区

  va_list ap;  # 定义一个可变参数列表

  nse = event_new(nsp, NSE_TYPE_WRITE, nsi, timeout_msecs, handler, userdata);  # 创建一个新的事件
  assert(nse);  # 断言事件存在

  va_start(ap,format);  # 初始化可变参数列表
  res = Vsnprintf(buf, sizeof(buf), format, ap);  # 格式化字符串
  va_end(ap);  # 结束可变参数列表

  if (res >= 0) {  # 如果格式化成功
    if (res >= sizeof(buf)) {  # 如果格式化后的字符串长度大于等于缓冲区大小
      buf2size = res + 16;  # 计算新的缓冲区大小
      buf2 = (char * )safe_malloc(buf2size);  # 分配新的缓冲区内存
      va_start(ap,format);  # 重新初始化可变参数列表
      res2 = Vsnprintf(buf2, buf2size, format, ap);  # 格式化字符串到新的缓冲区
      va_end(ap);  # 结束可变参数列表
      if (res2 < 0 || (size_t) res2 >= buf2size) {  # 如果格式化失败或者格式化后的字符串长度大于等于缓冲区大小
        free(buf2);  # 释放新的缓冲区内存
        buf2 = NULL;  # 将新的缓冲区指针置为空
      } else
        strlength = res2;  # 更新字符串长度
    } else {
      buf2 = buf;  # 将新的缓冲区指针指向原始缓冲区
      strlength = res;  # 更新字符串长度
    }
  }

  if (!buf2) {  # 如果新的缓冲区为空
    nse->event_done = 1;  # 设置事件完成标志
    nse->status = NSE_STATUS_ERROR;  # 设置事件状态为错误
    nse->errnum = EMSGSIZE;  # 设置错误号
  } else {
    if (strlength == 0) {  # 如果字符串长度为0
      nse->event_done = 1;  # 设置事件完成标志
      nse->status = NSE_STATUS_SUCCESS;  # 设置事件状态为成功
    } else {
      fs_cat(&nse->iobuf, buf2, strlength);  # 将格式化后的字符串拼接到事件的输入输出缓冲区中
    }
  }

  if (NsockLogLevel == NSOCK_LOG_DBG_ALL &&  # 如果日志级别为调试模式并且事件状态不是错误并且字符串长度小于80
      nse->status != NSE_STATUS_ERROR &&
      strlength < 80) {
    memcpy(displaystr, ": ", 2);  # 复制字符串到显示缓冲区
    memcpy(displaystr + 2, buf2, strlength);  # 复制字符串到显示缓冲区
    displaystr[2 + strlength] = '\0';  # 添加字符串结束符
    replacenonprintable(displaystr + 2, strlength, '.');  # 替换非打印字符为'.'
  } else {
    displaystr[0] = '\0';  # 否则将显示缓冲区置为空
  }

  nsock_log_info("Write request for %d bytes to IOD #%li EID %li [%s]%s",
                 strlength, nsi->id, nse->id, get_peeraddr_string(nsi),
                 displaystr);  # 记录日志

  if (buf2 != buf)  # 如果新的缓冲区不等于原始缓冲区
    free(buf2);  # 释放新的缓冲区内存

  nsock_pool_add_event(nsp, nse);  # 将事件添加到套接字池中

  return nse->id;  # 返回事件ID
}
```