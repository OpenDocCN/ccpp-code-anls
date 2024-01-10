# `nmap\nmap_ftp.h`

```
/* $Id$ */

// 防止头文件重复包含
#ifndef NMAP_FTP_H
#define NMAP_FTP_H

// 包含所需的头文件
#include "scan_lists.h"
#include "nbase.h" /* u16 */
class Target;

// 设置默认的 FTP 登录用户名和密码
#define FTPUSER "anonymous"
#define FTPPASS "-wwwuser@"
// 设置重新登录的次数
#define FTP_RETRIES 2 /* How many times should we relogin if we lose control
                         connection? */

// 定义存储 FTP 信息的结构体
struct ftpinfo {
  char user[64];
  char pass[256]; /* methinks you're paranoid if you need this much space */
  char server_name[FQDN_LEN + 1];
  struct in_addr server;
  u16 port;
  int sd; /* socket descriptor */
};

// 获取默认的 FTP 信息
struct ftpinfo get_default_ftpinfo(void);
// 匿名登录 FTP
int ftp_anon_connect(struct ftpinfo *ftp);

// 解析 URL 形式的 FTP 字符串
int parse_bounce_argument(struct ftpinfo *ftp, char *url);

// FTP 弹跳攻击扫描
void bounce_scan(Target *target, u16 *portarray, int numports,
                 struct ftpinfo *ftp);

// 结束头文件的防止重复包含
#endif /* NMAP_FTP_H */
```