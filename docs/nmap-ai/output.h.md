# `nmap\output.h`

```
/* $Id$ */

#ifndef OUTPUT_H
#define OUTPUT_H

#include <nbase.h> // __attribute__

// 定义日志文件的数量
#define LOG_NUM_FILES 4 /* # of values that actual files (they must come first */
// 定义日志文件类型的掩码
#define LOG_FILE_MASK 15 /* The mask for log types in the file array */
// 定义不同类型的日志文件
#define LOG_NORMAL 1
#define LOG_MACHINE 2
#define LOG_SKID 4
#define LOG_XML 8
#define LOG_STDOUT 1024
#define LOG_STDERR 2048
#define LOG_SKID_NOXLT 4096
#define LOG_MAX LOG_SKID_NOXLT /* The maximum log type value */

// 定义日志文件类型的组合
#define LOG_PLAIN LOG_NORMAL|LOG_SKID|LOG_STDOUT

// 定义日志文件类型的名称
#define LOG_NAMES {"normal", "machine", "$Cr!pT |<!dd!3", "XML"}

// 定义错误消息常量
#define PCAP_OPEN_ERRMSG "Call to pcap_open_live() failed three times. "\
"There are several possible reasons for this, depending on your operating "\
"system:\nLINUX: If you are getting Socket type not supported, try "\
"modprobe af_packet or recompile your kernel with PACKET enabled.\n "\
 "*BSD:  If you are getting device not configured, you need to recompile "\
 "your kernel with Berkeley Packet Filter support.  If you are getting "\
 "No such file or directory, try creating the device (eg cd /dev; "\
 "MAKEDEV <device>; or use mknod).\n*WINDOWS:  Nmap only supports "\
 "ethernet interfaces on Windows for most operations because Microsoft "\
 "disabled raw sockets as of Windows XP SP2.  Depending on the reason for "\
 "this error, it is possible that the --unprivileged command-line argument "\
 "will help.\nSOLARIS:  If you are trying to scan localhost or the "\
 "address of an interface and are getting '/dev/lo0: No such file or "\
 "directory' or 'lo0: No DLPI device found', complain to Sun.  I don't "\
 "think Solaris can support advanced localhost scans.  You can probably "\
 "use \"-Pn -sT localhost\" though.\n\n"

#include "scan_lists.h"
#ifndef NOLUA
#include "nse_main.h"
#endif
class PortList;
class Target;

#include <stdarg.h>
#include <string>

#if TIME_WITH_SYS_TIME
# include <sys/time.h>
# include <time.h>
#else
# if HAVE_SYS_TIME_H
#  include <sys/time.h>
# else
#  include <time.h>
# endif
#endif

#ifdef WIN32
/* 如果在 Windows 上使用了 --send-ip (PACKET_SEND_IP_STRONG) 选项，则不执行任何操作；否则显示一个致命错误，说明接口不是以太网接口，在 Windows 上无法工作。 */
void win32_fatal_raw_sockets(const char *devname);
#endif

/* 打印熟悉的 Nmap 表格输出，显示在机器上找到的“有趣”的端口。它还处理了机器/可筛选输出和 XML 输出。它相当丑陋 -- 特别是我应该编写辅助函数来处理表格创建 */
void printportoutput(const Target *currenths, const PortList *plist);

/* 如果找到了目标的 MAC 地址（通常意味着目标直接连接在以太网网络上），则打印 MAC 地址。这只打印到人类输出 -- XML 由单独的调用处理（print_MAC_XML_Info），因为它需要在特定位置打印以符合 DTD。 */
void printmacinfo(const Target *currenths);

char *logfilename(const char *str, struct tm *tm);

/* 将一些信息（printf 样式的参数）写入给定的日志流。记得注意格式字符串漏洞。 */
void log_write(int logt, const char *fmt, ...)
     __attribute__ ((format (printf, 2, 3)));

/* 这是日志函数的工作核心。通常通过 log_write() 调用，但如果处理 vfprintf 风格的 va_list，则可以直接调用它。与 log_write 不同，你只能使用一个日志类型调用它（而不是一个完整的位掩码）。此外，你必须在每次调用此函数之间使用 va_start() 和 va_end() 调用。 */
void log_vwrite(int logt, const char *fmt, va_list ap);

/* 关闭给定的日志流 */
void log_close(int logt);

/* 刷新给定的日志流。换句话说，所有缓冲输出立即写入日志 */
void log_flush(int logt);

/* 刷新每个日志流 -- 所有缓冲输出立即写入相应的日志 */
void log_flush_all();
/* Open a log descriptor of the type given to the filename given.  If
   append is nonzero, the file will be appended instead of clobbered if
   it already exists.  If the file does not exist, it will be created */
int log_open(int logt, bool append, const char *filename);

/* Output the list of ports scanned to the top of machine parseable
   logs (in a comment, unfortunately).  The items in ports should be
   in sequential order for space savings and easier to read output */
void output_ports_to_machine_parseable_output(const struct scan_lists *ports);

/* Return a std::string containing all n strings separated by whitespace, and
   individually quoted if needed. */
std::string join_quoted(const char * const strings[], unsigned int n);

/* Similar to output_ports_to_machine_parseable_output, this function
   outputs the XML version, which is scaninfo records of each scan
   requested and the ports which it will scan for */
void output_xml_scaninfo_records(const struct scan_lists *ports);

/* Writes a heading for a full scan report ("Nmap scan report for..."),
   including host status and DNS records. */
void write_host_header(const Target *currenths);

/* Writes host status info to the log streams (including STDOUT).  An
   example is "Host: 10.11.12.13 (foo.bar.example.com)\tStatus: Up\n" to
   machine log. */
void write_host_status(const Target *currenths);

/* Writes host status info to the XML stream wrapped in a <hosthint> tag */
void write_xml_hosthint(const Target *currenths);

/* Prints the formatted OS Scan output to stdout, logfiles, etc (but only
   if an OS Scan was performed */
void printosscanoutput(const Target *currenths);

/* Prints the alternate hostname/OS/device information we got from the
   service scan (if it was performed) */
void printserviceinfooutput(const Target *currenths);

#ifndef NOLUA
std::string protect_xml(const std::string s);

/* Use this function to report NSE_PRE_SCAN and NSE_POST_SCAN results */
/* 声明一个函数，用于打印脚本的结果 */
void printscriptresults(const ScriptResults *scriptResults, stype scantype);

/* 声明一个函数，用于打印主机脚本的结果 */
void printhostscriptresults(const Target *currenths);

/* 打印具有跟踪路由跳数的表格 */
void printtraceroute(const Target *currenths);

/* 打印带有延迟的“主机的时间”输出 */
void printtimes(const Target *currenths);

/* 打印Nmap接口和路由的详细列表到正常/非正常/标准输出 */
int print_iflist(void);

/* 在程序运行时打印状态消息 */
void printStatusMessage();

/* 打印XML结束标签的开头部分 */
void print_xml_finished_open(time_t timep, const struct timeval *tv);

/* 打印XML主机信息 */
void print_xml_hosts();

/* 打印Nmap运行结束时的统计信息和其他信息 */
void printfinaloutput();

/* 打印加载的数据文件的名称和路径 */
void printdatafilepaths();

/* nsock日志记录接口 */
void nmap_adjust_loglevel(bool trace);
void nmap_set_nsock_logger();
```