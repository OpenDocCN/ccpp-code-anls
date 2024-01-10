# `nmap\nsock\src\nsock_read.c`

```
/* $Id$ */

#include "nsock_internal.h"  // 包含内部网络套接字库的头文件
#include "nsock_log.h"  // 包含网络套接字日志的头文件
#include "netutils.h"  // 包含网络工具函数的头文件


/* 读取最多 nlines 行（以 \n 结尾，当然也包括 \r\n），或者直到 EOF，或者直到超时，以先到者为准。
 * 请注意，如果至少读取了 1 个字符，那么在 EOF 或超时的情况下将返回 NSE_STATUS_SUCCESS。
 * 还请注意，您可能会得到比 'nlines' 更多的行 - 我们只是在“至少”读取了 'nlines' 之后停止 */
nsock_event_id nsock_readlines(nsock_pool nsp, nsock_iod ms_iod,
                               nsock_ev_handler handler, int timeout_msecs,
                               void *userdata, int nlines) {
  struct niod *nsi = (struct niod *)ms_iod;  // 将 ms_iod 转换为 niod 结构体指针
  struct npool *ms = (struct npool *)nsp;  // 将 nsp 转换为 npool 结构体指针
  struct nevent *nse;  // 定义 nevent 结构体指针

  nse = event_new(ms, NSE_TYPE_READ, nsi, timeout_msecs, handler, userdata);  // 创建一个新的事件
  assert(nse);  // 断言事件非空

  nsock_log_info("Read request for %d lines from IOD #%li [%s] EID %li",
                 nlines, nsi->id, get_peeraddr_string(nsi), nse->id);  // 记录读取请求的日志信息

  nse->readinfo.read_type = NSOCK_READLINES;  // 设置读取类型为 NSOCK_READLINES
  nse->readinfo.num = nlines;  // 设置读取行数

  nsock_pool_add_event(ms, nse);  // 将事件添加到事件池中

  return nse->id;  // 返回事件的 ID
}

/* 与上面相同，只是它尝试读取至少 'nbytes' 而不是 'nlines'。*/
nsock_event_id nsock_readbytes(nsock_pool nsp, nsock_iod ms_iod,
                               nsock_ev_handler handler, int timeout_msecs,
                               void *userdata, int nbytes) {

  struct niod *nsi = (struct niod *)ms_iod;  // 将 ms_iod 转换为 niod 结构体指针
  struct npool *ms = (struct npool *)nsp;  // 将 nsp 转换为 npool 结构体指针
  struct nevent *nse;  // 定义 nevent 结构体指针

  nse = event_new(ms, NSE_TYPE_READ, nsi, timeout_msecs, handler, userdata);  // 创建一个新的事件
  assert(nse);  // 断言事件非空

  nsock_log_info("Read request for %d bytes from IOD #%li [%s] EID %li",
                 nbytes, nsi->id, get_peeraddr_string(nsi), nse->id);  // 记录读取请求的日志信息

  nse->readinfo.read_type = NSOCK_READBYTES;  // 设置读取类型为 NSOCK_READBYTES
  nse->readinfo.num = nbytes;  // 设置读取字节数

  nsock_pool_add_event(ms, nse);  // 将事件添加到事件池中

  return nse->id;  // 返回事件的 ID
}
/* 简单的读取函数 -- 当读取到任何内容时返回 NSE_STATUS_SUCCESS，否则根据情况返回 timeout、eof 或 error */
nsock_event_id nsock_read(nsock_pool nsp, nsock_iod ms_iod,
                          nsock_ev_handler handler, int timeout_msecs,
                          void *userdata) {
  // 将 ms_iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)ms_iod;
  // 将 nsp 转换为 npool 结构体
  struct npool *ms = (struct npool *)nsp;
  // 创建一个新的事件对象
  struct nevent *nse;
  
  // 使用给定参数创建一个新的事件对象
  nse = event_new(ms, NSE_TYPE_READ, nsi, timeout_msecs, handler, userdata);
  // 断言事件对象创建成功
  assert(nse);

  // 记录读取请求的日志信息
  nsock_log_info("Read request from IOD #%li [%s] (timeout: %dms) EID %li",
                 nsi->id, get_peeraddr_string(nsi), timeout_msecs, nse->id);

  // 设置读取类型为 NSOCK_READ
  nse->readinfo.read_type = NSOCK_READ;

  // 将事件对象添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件对象的 ID
  return nse->id;
}
```