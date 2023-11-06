# Nmap源码解析 4

# `output.h`

This text is a part of the Nmap software license agreement. It explains the terms and conditions of the software, including the prohibition of using the software in commercial products. It also explains the exceptions to this rule, such as the Nmap OEM Edition, which has more permissive terms. The text also provides information about the source code and the rights to use it. It is distributed under the GPL license, but some features and modifications are subject to additional terms.



```cpp

/***************************************************************************
 * output.h -- Handles the Nmap output system.  This currently involves    *
 * console-style human readable output, XML output, Script |<iddi3         *
 * output, and the legacy grepable output (used to be called "machine      *
 * readable").  I expect that future output forms (such as HTML) may be    *
 * created by a different program, library, or script using the XML        *
 * output.                                                                 *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为 "OUTPUT_H" 的头文件，其中包含了一些常量和定义，与 C 语言标准库中的 logging 函数相关。以下是代码的主要部分：

```cpp
#ifndef OUTPUT_H
#define OUTPUT_H
```

这是一个名为 "OUTPUT_H" 的头文件，说明它是一个定义。根据头文件声明模式，这个文件是抽象类（abstract class），意味着它不能被任何具体的类继承。

```cpp
#include <nbase.h> // __attribute__
```

这个头文件引入了名为 "nbase.h" 的头文件，这是一种用于输出信息到基站的功能性头文件。

```cpp
#define LOG_NUM_FILES 4             /* # of values that actual files (they must come first) */
#define LOG_FILE_MASK 15           /* The mask for log types in the file array */
#define LOG_NORMAL 1             /* Normal log level */
#define LOG_MACHINE 2             /* Machine log level */
#define LOG_SKID 4             /* Skid log level */
#define LOG_XML 8             /* XML log level */
#define LOG_STDOUT 1024          /* Standard output log level */
#define LOG_STDERR 2048          /* Standard error log level */
```

这个头文件定义了一些常量，它们定义了输出的文件数量、文件模式、日志级别、日志类型等。

```cpp
#define LOG_NUM_FILES 4             /* # of values that actual files (they must come first) */
```

这个常量定义了要输出的文件数量，值为 4。

```cpp
#define LOG_FILE_MASK 15           /* The mask for log types in the file array */
```

这个常量定义了文件模式，值为 15。

```cpp
#define LOG_NORMAL 1             /* Normal log level */
```

这个常量定义了日志级别，值为 1。

```cpp
#define LOG_MACHINE 2             /* Machine log level */
```

这个常量定义了机器级别，值为 2。

```cpp
#define LOG_SKID 4             /* Skid log level */
```

这个常量定义了滑块级别，值为 4。

```cpp
#define LOG_XML 8             /* XML log level */
```

这个常量定义了 XML 级别，值为 8。

```cpp
#define LOG_STDOUT 1024          /* Standard output log level */
```

这个常量定义了标准输出日志级别，值为 1024。

```cpp
#define LOG_STDERR 2048          /* Standard error log level */
```

这个常量定义了标准错误日志级别，值为 2048。


```cpp
/* $Id$ */

#ifndef OUTPUT_H
#define OUTPUT_H

#include <nbase.h> // __attribute__

#define LOG_NUM_FILES 4 /* # of values that actual files (they must come first */
#define LOG_FILE_MASK 15 /* The mask for log types in the file array */
#define LOG_NORMAL 1
#define LOG_MACHINE 2
#define LOG_SKID 4
#define LOG_XML 8
#define LOG_STDOUT 1024
#define LOG_STDERR 2048
```

这段代码是一个C/C++语言的预处理指令，定义了一些符号常量和宏定义。下面是每个符号常量的解释：

1. LOG_SKID_NOXLT：这是一个整型常量，表示最大日志类型值，值为4096。这个常量在定义LOG_PLAIN时被使用。

2. LOG_PLAIN：这是一个符号常量，表示普通日志类型，由LOG_NORMAL、LOG_SKID和LOG_STDOUT组成。这个符号常量在定义LOG_NAMES时被使用。

3. LOG_NAMES：这是一个符号常量，定义了多个名称，包括"normal"、"machine"、"$Cr!pT |<!dd!3"和"XML"。

4. PCAP_OPEN_ERRMSG：这是一个字符串常量，定义了一个错误消息，用于在PCAP打开失败时打印。这个常量在定义LOG_NORMAL、LOG_SKID和LOG_STDOUT时被使用。

总的来说，这段代码定义了一些符号常量和宏定义，用于定义和输出日志信息。LOG_MAX是LOG_PLAIN的最大值，LOG_NORMAL、LOG_SKID和LOG_STDOUT则定义了不同类型的日志信息。PCAP_OPEN_ERRMSG用于在PCAP Open失败时打印错误消息。


```cpp
#define LOG_SKID_NOXLT 4096
#define LOG_MAX LOG_SKID_NOXLT /* The maximum log type value */

#define LOG_PLAIN LOG_NORMAL|LOG_SKID|LOG_STDOUT

#define LOG_NAMES {"normal", "machine", "$Cr!pT |<!dd!3", "XML"}

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

```

这段代码的作用是包含其他头文件和定义，并声明一个名为PortList的类和一个名为Target的类。它还包含一个名为scan_lists.h的文件，但没有定义该文件的内容。

该代码使用了一个名为TIME_WITH_SYS_TIME的宏，如果当前系统时间与使用系统时间（非用户指定）相同，则包含一个名为<sys/time.h>的 header，否则包含一个名为<time.h>的 header。这意味着该代码将在包含这些头文件的情况下运行，并获得更好的性能。

该代码使用了一个名为PortList的类和一个名为Target的类。但是，这两类并未定义任何成员变量或方法。


```cpp
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
```

这段代码包含了一个头文件 `<sys/time.h>` 和一个 `else` 分支，后面分别跟了一个 `if` 语句和一个 `endif` 结束的定义。我们需要分析这个代码的作用并解释它的逻辑。

首先，我们看到一个 `#include <time.h>`，这个是引入了 `time.h` 这个头文件，它应该是用于在程序中处理时间相关的操作。

接着，我们看到了一个 `#ifdef WIN32`，这个是一个条件编译指令，它用于判断当前操作系统是否为 Windows。如果是，那么后面会输出一个 `fatal_raw_sockets` 函数，这个函数会输出一个致命的错误信息，说明当前操作系统不支持某种接口，但如果是使用了 `--send-ip` 选项，那么这个函数不会被执行。

在 `#ifdef WIN32` 后面的部分，我们看到了 `win32_fatal_raw_sockets` 函数，这个函数的作用是输出一个致命的错误信息，这个信息会说明当前操作系统不支持某种接口，但并不会执行 `fatal_raw_sockets` 函数。这个函数可能会在某些版本的 Windows 上被占用，因此在实际程序中，我们应该使用 `fatal_raw_sockets` 函数。

最后，我们看到了一些注释，它们描述了 `win32_fatal_raw_sockets` 函数的功能，以及说明了一些需要注意的事项。


```cpp
#  include <sys/time.h>
# else
#  include <time.h>
# endif
#endif

#ifdef WIN32
/* Show a fatal error explaining that an interface is not Ethernet and won't
   work on Windows. Do nothing if --send-ip (PACKET_SEND_IP_STRONG) was used. */
void win32_fatal_raw_sockets(const char *devname);
#endif

/* Prints the familiar Nmap tabular output showing the "interesting"
   ports found on the machine.  It also handles the Machine/Grepable
   output and the XML output.  It is pretty ugly -- in particular I
   should write helper functions to handle the table creation */
```



这是一段 C 语言代码，主要作用是打印当前网络接口的 MAC 地址，如果目标设备上存在 MAC 地址，则打印在控制台上，否则打印在 stderr 文件中。

具体来说，代码中定义了两个函数，printportoutput 和 printmacinfo，都使用了 print 函数，但是 printmacinfo 函数多了两个参数，const Target *currenths 和 const PortList *plist，表示要打印的端口列表和目标设备的当前的网络接口。

printmacinfo 函数会先尝试从目标设备的当前网络接口中获取 MAC 地址，如果获取成功，则打印该 MAC 地址。如果当前网络接口中不存在 MAC 地址，则打印一些默认信息，例如目标 ID 为 0。

print 函数用于输出格式化信息，包括参数个数和参数名称。在这里，print 函数被用来输出字符串和打印 MAC 地址。如果参数是 const 类型，则只输出参数的值，不做格式化处理。


```cpp
void printportoutput(const Target *currenths, const PortList *plist);

/* Prints the MAC address if one was found for the target (generally
   this means that the target is directly connected on an ethernet
   network.  This only prints to human output -- XML is handled by a
   separate call ( print_MAC_XML_Info ) because it needs to be printed
   in a certain place to conform to DTD. */
void printmacinfo(const Target *currenths);

char *logfilename(const char *str, struct tm *tm);

/* Write some information (printf style args) to the given log stream(s).
   Remember to watch out for format string bugs. */
void log_write(int logt, const char *fmt, ...)
     __attribute__ ((format (printf, 2, 3)));

```

这段代码定义了三个函数：log_vwrite、log_close和log_flush。这些函数都与日志输出相关。

log_vwrite函数接受三个参数：日历日志类型(int logt)、格式字符串(const char *fmt)和一个可变参数列表(va_list Ap)。日历日志类型和格式字符串决定了要输出的日志信息的内容和格式，而可变参数列表允许在每次调用函数时传递不同的参数。此函数的作用是用于va_list风格的输入，如果使用的是log_write函数，那么这个函数将不起作用。

log_close函数接受一个整数参数logt，表示要关闭的日志流ID。此函数的作用是用于关闭已经打开的日志流。

log_flush函数接受一个整数参数logt，表示要flush的日志流ID。此函数的作用是输出所有缓存的日志信息，将它们立即写入到日志中。如果已经打开了一个日志流，但是还没有调用log_write函数，那么此函数将自动调用log_flush函数将缓存的信息写入到日志中。


```cpp
/* This is the workhorse of the logging functions.  Usually it is
   called through log_write(), but it can be called directly if you
   are dealing with a vfprintf-style va_list.  Unlike log_write, YOU
   CAN ONLY CALL THIS WITH ONE LOG TYPE (not a bitmask full of them).
   In addition, YOU MUST SANDWICH EACH EXECUTION OF THIS CALL BETWEEN
   va_start() AND va_end() calls. */
void log_vwrite(int logt, const char *fmt, va_list ap);

/* Close the given log stream(s) */
void log_close(int logt);

/* Flush the given log stream(s).  In other words, all buffered output
   is written to the log immediately */
void log_flush(int logt);

```

这段代码定义了三个函数，它们的目的是不同的。让我分别解释一下它们的作用。

1. `log_flush_all()` 函数的作用是“刷新所有日志 stream”，即所有缓冲输出数据都会立即写入到相应的日志中。

2. `log_open()` 函数的作用是“打开给定文件名的日志描述符”。如果 `append` 参数为真，则文件将追加到已存在的文件中；如果文件已存在，则不会覆盖现有的内容。

3. `output_ports_to_machine_parseable_output()` 函数的作用是“输出扫描到的所有端口，这些端口将添加到机器可读的日志中”。该函数将端口列表中的所有元素按空间节省的顺序输出，并使用空间节约的行距。

由于没有设置源代码，我无法提供更多有关函数如何工作的背景信息。


```cpp
/* Flush every single log stream -- all buffered output is written to the
   corresponding logs immediately */
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
```

这段代码定义了一个名为`join_quoted`的函数，它接受一个整数数量的`const char *`类型的参数`strings`，并返回一个指向`unsigned char *`类型的指针，指向存储了这些字符串的内存位置。

这个函数的作用是实现了一个类似于`output_ports_to_machine_parseable_output`的函数，它将输入的`const char *`类型的参数`strings`连接成一个XML版本的字符串，并返回这个XML版本的字符串。这个XML版本的字符串包含了每个请求的扫描信息的记录，以及该扫描信息将会扫描哪些端口。

这个函数的实现主要依赖于对外部库的依赖，比如`xml_parser.h`和`xml_tokens.h`。其中`xml_parser.h`用于解析XML版本的字符串，而`xml_tokens.h`则提供了用于处理XML版本的输入和输出。


```cpp
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

```

这段代码定义了三个函数，用于将分别是最新的操作系统扫描结果，作为目标（Target）的属性，通过写入XML流和`<hosthint>`标签的格式将信息输出到文件中。

1. `write_xml_hosthint(const Target *currenths)`：将指定的目标（Target）对象的属性信息写入XML流。如果操作系统扫描（OS Scan）已经执行过，则将结果输出到文件中。

2. `printosscanoutput(const Target *currenths)`：打印格式化后的操作系统扫描输出。只有在操作系统扫描（OS Scan）被执行过时，才会输出文件。

3. `printserviceinfooutput(const Target *currenths)`：打印来自服务扫描（Service Scan）的属性信息。只有在操作系统扫描（OS Scan）被执行过时，才会输出文件。

4. 如果需要，`print_xml_hosthint`函数可以通过将`s`（格式化后的操作系统扫描输出）转义后用`protected_xml`函数进行保护，从而将敏感信息隐藏。


```cpp
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
```

这是一个C语言代码，定义了四个函数，以及一个声明。这些函数和声明一起组成了一个更大的函数，其目的是打印输出一些信息。

函数1名为`printscriptresults`，接受一个指向`ScriptResults`类型的参数。函数2名为`printhostscriptresults`，接受一个指向`Target`类型的参数。

这两个函数的实现都未给出，但它们的名称和参数可以推测出它们的作用。可以想象，这两个函数可能用于在屏幕上打印输出某些信息。

函数3名为`printtraceroute`，接受一个指向`Target`类型的参数。函数4名为`printtimes`，也接受一个指向`Target`类型的参数。

这两个函数的实现未给出，但名字和参数可以推测出它们的作用。可以想象，这两个函数可能用于在屏幕上打印输出某些信息。

函数5名为`print_iflist`，没有参数。

这是一个声明，告诉编译器在当前源文件中包含一个名为`print_iflist`的函数。


```cpp
void printscriptresults(const ScriptResults *scriptResults, stype scantype);

void printhostscriptresults(const Target *currenths);
#endif

/* Print a table with traceroute hops. */
void printtraceroute(const Target *currenths);

/* Print "times for host" output with latency. */
void printtimes(const Target *currenths);

/* Print a detailed list of Nmap interfaces and routes to
   normal/skiddy/stdout output */
int print_iflist(void);

```

这段代码是一个用于打印 Nmap 统计信息的工具函数。下面是每个函数的作用及其实现：

1. `printStatusMessage()`：在程序运行时打印一条状态信息，具体内容由函数内部实现。
2. `print_xml_finished_open()`：打印 XML 文件中 "finished" 标记处的时间。
3. `print_xml_hosts()`：打印 XML 文件中所有主机及其所在的文件路径。
4. `printfinaloutput()`：打印最终输出的统计信息。
5. `printdatafilepaths()`：打印所有加载的数据文件的路径。


```cpp
/* Prints a status message while the program is running */
void printStatusMessage();

void print_xml_finished_open(time_t timep, const struct timeval *tv);

void print_xml_hosts();

/* Prints the statistics and other information that goes at the very end
   of an Nmap run */
void printfinaloutput();

/* Prints the names of data files that were loaded and the paths at which they
   were found. */
void printdatafilepaths();

```

这两行代码定义了一个名为 nmap_adjust_loglevel 的函数和一个名为 nmap_set_nsock_logger 的函数。这两个函数都与 nmap 库的日志输出有关。

nsock 库提供了日志输出接口，可以在调试或错误时记录相关信息。根据 nmap_adjust_loglevel 的函数说明，它可以用来设置或调整 nmap 的日志级别。具体来说，它可以接受一个布尔值 trace，并在这个值为真时记录更多的日志信息，以帮助调试程序。

nmap_set_nsock_logger 函数则接受一个 nsock logger 对象。这个函数可以用来设置或获取 nmap 库使用的日志输出库，例如，输出库可以是输出到屏幕上，也可以是输出到文件中。

这两行代码的主要作用是定义和实现 nmap 库的日志输出功能，以便用户可以更好地调试程序。


```cpp
/* nsock logging interface */
void nmap_adjust_loglevel(bool trace);
void nmap_set_nsock_logger();

#endif /* OUTPUT_H */


```

# `payload.h`

This is a text presentation关于 Nmap, a network scanner used to scan、test applications, websites, and networks. Nmap is free software and open-source, and it can be used for various purposes, including network exploration, identifying vulnerabilities, and performance testing. The official Nmap Windows builds include the Npcap software for packet capture and transmission, but it is under a separate license term that forks the software. Nmap's license generally prohibits companies from using the software in commercial products, but an Nmap OEM edition is available for a more permissive license with special features. Nmap allows users to submit changes to the software through a GitHub PR or by email to [dev@nmap.org](mailto:dev@nmap.org). The text also mentions that the free version of Nmap is distributed with a warranty, but it does not guarantee commercial support or indemnification.



```cpp

/***************************************************************************
 * payload.cc -- Retrieval of UDP payloads.                                *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为 `PAYLOAD_H` 的头文件，其中包括了两个函数：`get_udp_payload` 和 `udp_payload_count`。同时，定义了一个常量 `MAX_PAYLOADS_PER_PORT`，表示每个 UDP 数据包最大可携带的数据负载长度。

接下来是 `udp_payload_count` 函数的实现，这个函数接收一个 UDP 数据端口号 `dport` 和一个指向长度的指针 `length`，返回该数据端口所能携带的最大数据负载长度。这个函数使用了 `u16` 类型来表示数据端口号，因为数据端口号最大可以表示 0xffff。

`get_udp_payload` 函数接收一个数据端口号 `dport` 和一个指向 UDP 数据负载长度的指针 `length`，返回该数据端口所能携带的最大数据负载长度。这个函数使用了 `u16` 类型来表示数据端口号，因为数据端口号最大可以表示 0xffff。同时，这个函数还接收一个指针 `buf`，用于存储数据负载，并使用 ` length` 指向的值来获取数据负载长度。

最后，`payload_service_match` 函数接收一个数据端口号 `dport`、一个 UDP 数据负载 `buf`，以及一个指向服务匹配details的指针。这个函数使用 `u16` 类型来表示数据端口号，并使用 `get_udp_payload` 函数获取该数据端口所能携带的最大数据负载长度。同时，这个函数还使用 `u8` 类型来表示服务匹配 details，包含一个 `ServiceType` 字段和两个 `Details` 字段。`ServiceType` 字段用于表示服务类型，0x01 表示 HTTP，0x02 表示 FTP，以此类推。`Details` 字段包含两个字段，第一个字段表示端口号，第二个字段包含一个字符串，用于表示服务名称。

最后，`init_payloads` 函数是一个静态函数，用于初始化可以携带的数据负载长度。这个函数的实现比较简单，直接将 `MAX_PAYLOADS_PER_PORT` 常量赋值为 0xff，表示允许通过当前网络接口传输的所有数据负载长度。


```cpp
/* $Id$ */

#ifndef PAYLOAD_H
#define PAYLOAD_H

#include "service_scan.h"

// Semi-arbitrary limit, but we use u8 for indexing/retrieval
// and we send all payloads at once and need to not overwhelm.
#define MAX_PAYLOADS_PER_PORT 0xff

const u8 *get_udp_payload(u16 dport, int *length, u8 index);
u8 udp_payload_count(u16 dport);
const struct MatchDetails *payload_service_match(u16 dport, const u8 *buf, int buflen);
void init_payloads(void);
```

这是一个C语言中的函数，名为“free_payloads”。然而，从您提供的代码来看，该函数名似乎不是在定义在stdlib.h或者stdio.h头文件中。因此，我们需要更多的上下文来了解该函数的实际作用。

在实际应用中，void free_payloads（）函数可能是用于释放或者回收在使用完毕或者不需要的内存。free_payloads（）函数接受一个void类型的参数，因此在函数内部，可以对其进行自由地释放或者回收。这个函数可以被用于管理动态内存分配，以避免内存泄漏或者溢出等潜在问题。


```cpp
void free_payloads(void);

#endif /* PAYLOAD_H */

```

# `portlist.h`

This text is a section of the Nmap software's documentation. It explains the Nmap license and its restrictions, as well as providing information about using and redistributing Nmap.

The Nmap license is generally prohibiting companies from using it in commercial products. However, Nmap provides a special Nmap OEM Edition license with a more permissive license and special features for this purpose. The text explains that if a company has received an Nmap license agreement or contract stating other terms, they may choose to use and redistribute Nmap under those terms instead.

The text also mentions that the official Nmap Windows builds include the Npcap software for packet capture and transmission. The Npcap software is under a separate license term which prohibits redistribution without special permission. The text also explains that the source code allows users to port Nmap to new platforms, fix bugs, and add new features. Users are encouraged to submit their changes as a Github PR or by email to the dev@nmap.org mailing list for possible incorporation into the main distribution.

Finally, the text reminds users that the free version of Nmap is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of merchantability or fitness for a particular purpose. Warranties, indemnification, and commercial support are all available through the Npcap OEM program.


```cpp
/***************************************************************************
 * portlist.h -- Functions for manipulating various lists of ports         *
 * maintained internally by Nmap.                                          *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码是一个C代码，定义了一个名为"PORTLIST_H"的 header文件。这个文件包含了以下内容：

1. 定义了一个名为"$Id$"的标识，用于标识这个文件是由谁编写的，以便在需要时进行追踪。

2. 定义了一个名为"PORTLIST_H"的头部文件，格式化字符串为：
```cpp
#ifndef PORTLIST_H
#define PORTLIST_H
```
这个文件是声明的，表示这是一个头部文件，而不是一个具体的文件实现。

3. 引入了两个头文件：
```cpp
#include "nbase.h"
#include "nse_main.h"
```
4. 引入了一个头文件：
```cpp
#include "portreasons.h"
```
5. 定义了一个名为"portstates.h"的头文件，可能是用于定义头文件：
```cpp
#include "portreasons.h"
```
6. 定义了一个名为"portstates"的变量：
```cpp
std::vector<int> portstates;
```
7. 没有定义任何函数，可能是在导入其他库或定义其他变量时需要用到的。


```cpp
/* $Id$ */

#ifndef PORTLIST_H
#define PORTLIST_H

#include "nbase.h"
#ifndef NOLUA
#include "nse_main.h"
#endif

#include "portreasons.h"

#include <vector>

/* port states */
```

这段代码定义了一系列用于定义端口号的宏，包括：PORT_UNKNOWN、PORT_CLOSED、PORT_OPEN、PORT_FILTERED、PORT_TESTING、PORT_FRESH、PORT_UNFILTERED 和 PORT_OPENFILTERED。这些宏分别表示未知、关闭、打开、筛选、测试、 fresh 和 openfilter 端口状态。

另外，还定义了一个名为 PORT_UNFILTERED 的宏，它表示没有过滤的未知端口状态。

接着，定义了一个名为 statenum2str 的函数，用于将端口状态数字转换为字符串，例如：0 表示 unknown，1 表示 closed，2 表示 open，3 表示 filtered，等等。

最后，定义了一些 IP 协议，包括 TCPANDUDPANDSCTP 和 UDPANDSCTP，用于定义网络应用程序的端口号。


```cpp
#define PORT_UNKNOWN 0
#define PORT_CLOSED 1
#define PORT_OPEN 2
#define PORT_FILTERED 3
#define PORT_TESTING 4
#define PORT_FRESH 5
#define PORT_UNFILTERED 6
#define PORT_OPENFILTERED 7 /* Like udp/fin/xmas/null/ipproto scan with no response */
#define PORT_CLOSEDFILTERED 8 /* Idle scan */
#define PORT_HIGHEST_STATE 9 /* ***IMPORTANT -- BUMP THIS UP WHEN STATES ARE
                                ADDED *** */
const char *statenum2str(int state);

#define TCPANDUDPANDSCTP IPPROTO_MAX
#define UDPANDSCTP (IPPROTO_MAX + 1)

```

这段代码定义了一个枚举类型`service_detection_type`，用于表示扫描服务时的状态。枚举类型`serviceprobestate`定义了九种状态，分别为：

- `PROBESTATE_INITIAL`：初始状态，没有开始 probing。
- `PROBESTATE_NULLPROBE`：当前状态，正在处理 NULL Probe。
- `PROBESTATE_MATCHINGPROBES`：正在处理匹配 Probe。
- `PROBESTATE_NONMATCHINGPROBES`：上述状态失败，正在检查非匹配 Probe。
- `PROBESTATE_FINISHED_HARDMATCHED`：找到匹配的服务，且是硬链接。
- `PROBESTATE_FINISHED_SOFTMATCHED`：找到匹配的服务，且是软链接。
- `PROBESTATE_FINISHED_NOMATCH`：没有找到匹配的服务。
- `PROBESTATE_FINISHED_TCPWRAPPED`：该端口被阻塞，通过 tcpwrappers。
- `PROBESTATE_EXCLUDED`：该端口已被排除在扫描之外。
- `PROBESTATE_INCOMPLETE`：扫描失败，由于错误、主机超时等原因未完成。

同时，还定义了一个枚举类型`service_detection_type`，用于表示扫描服务时的状态，它有两个枚举值：`SERVICE_DETECTION_TABLE`和`SERVICE_DETECTION_PROBED`。


```cpp
enum serviceprobestate {
  PROBESTATE_INITIAL=1, // No probes started yet
  PROBESTATE_NULLPROBE, // Is working on the NULL Probe
  PROBESTATE_MATCHINGPROBES, // Is doing matching probe(s)
  PROBESTATE_NONMATCHINGPROBES, // The above failed, is checking nonmatches
  PROBESTATE_FINISHED_HARDMATCHED, // Yay!  Found a match
  PROBESTATE_FINISHED_SOFTMATCHED, // Well, a soft match anyway
  PROBESTATE_FINISHED_NOMATCH, // D'oh!  Failed to find the service.
  PROBESTATE_FINISHED_TCPWRAPPED, // We think the port is blocked via tcpwrappers
  PROBESTATE_EXCLUDED, // The port has been excluded from the scan
  PROBESTATE_INCOMPLETE // failed to complete (error, host timeout, etc.)
};

enum service_detection_type { SERVICE_DETECTION_TABLE, SERVICE_DETECTION_PROBED };

```

这段代码定义了一个枚举类型 `service_tunnel_type`，该枚举类型有两个枚举元素，分别为 `SERVICE_TUNNEL_NONE` 和 `SERVICE_TUNNEL_SSL`。

此外，该文件中还有一段注释，指出 `random_port_cheat` 函数的作用是将一些流行 TCP 端口移动到 `portlist` 数组的开头，以便加速某些扫描，应该在进行任何端口随机化之前已经完成了。

接下来是 `struct serviceDeductions` 结构体，该结构体包含了一些用于确定服务是否成功探测到的信息。

- `serviceDeductions()` 是结构体的默认构造函数，其中所有成员变量都被初始化为默认值。
- `erase()` 函数用于移除结构体中的所有不需要的内存，并设置所有 pointer 成员为 `null`。
- `populateFullVersionString()` 函数用于将结构体中的 `name` 和 `version` 成员填充为正确的字符串，并返回该结构的完整版本字符串。
- `name` 成员是一个字符串，用于表示服务名称。如果无法确定服务名称，则该成员将 `NULL` 初始化。
- `name_confidence` 成员是一个整数，表示对服务名称的信心度量，其值范围为 0 到 10，表示最准确的猜测值。
- `product` 成员是一个字符串，用于标识服务。如果无法确定服务名称，则该成员将 `NULL` 初始化。
- `version` 成员是一个字符串，用于表示服务版本。如果无法确定服务名称和版本，则该成员将 `NULL` 初始化。
- `extrainfo` 成员是一个字符串，用于表示附加信息。
- `hostname` 成员是一个字符串，用于表示主机名。
- `ostype` 成员是一个字符串，表示操作系统类型。
- `devicetype` 成员是一个字符串，表示设备类型。
- `cpe` 成员是一个字符串，表示服务通信协议(CPE)。

最后，该文件中还有一段注释，说明 `random_port_cheat()` 函数的作用是将一些流行 TCP 端口移动到 `portlist` 数组的开头，以便加速某些扫描，应该在进行任何端口随机化之前已经完成了。


```cpp
enum service_tunnel_type { SERVICE_TUNNEL_NONE, SERVICE_TUNNEL_SSL };

// Move some popular TCP ports to the beginning of the portlist, because
// that can speed up certain scans.  You should have already done any port
// randomization, this should prevent the ports from always coming out in the
// same order.
void random_port_cheat(u16 *ports, int portcount);

struct serviceDeductions {
  serviceDeductions();
  // Free any strings that need to be freed and set all pointers to null.
  void erase();
  void populateFullVersionString(char *buf, size_t n) const;

  const char *name; // will be NULL if can't determine
  // Confidence is a number from 0 (least confident) to 10 (most
  // confident) expressing how accurate the service detection is
  // likely to be.
  int name_confidence;
  // Any of these 6 can be NULL if we weren't able to determine it
  char *product;
  char *version;
  char *extrainfo;
  char *hostname;
  char *ostype;
  char *devicetype;
  std::vector<char *> cpe;
  // SERVICE_TUNNEL_NONE or SERVICE_TUNNEL_SSL
  enum service_tunnel_type service_tunnel;
  // if we should give the user a service fingerprint to submit, here it is.  Otherwise NULL.
  char *service_fp;
  enum service_detection_type dtype; // definition above
};

```

这段代码定义了一个名为 "Port" 的类，以及一个名为 "PortList" 的 friend 类。

该 "Port" 类有以下成员变量：

- "portno" 是一个无符号整数，表示 port 号。
- "proto" 是一个无符号整数，表示网络协议类型。
- "state" 是一个无符号整数，表示当前网络状态的类型。
- "reason" 是一个无符号整数，表示状态变化的Reason类型。
- "scriptResults" 是一个名为 "ScriptResults" 的成员变量，类型为 void 类型。

该 "Port" 类有以下方法：

- "freeService(bool del_service)" 是一个 void 类型的函数，用于释放网络服务。它接受一个布尔参数 "del_service"，表示是否删除服务。
- "freeScriptResults(void)" 是一个 void 类型的函数，用于释放 Nmap 服务的结果缓存区。
- "getNmapServiceName(char *namebuf, int buflen)" 是一个 const char * 类型的函数，用于获取 Nmap 服务的名称。它接受一个字符指针 "namebuf" 和一个整数 "buflen"，表示要获取的名称的长度。

此外，该 "Port" 类还有一个 friend 名为 "PortList"，类型为 void 类型的函数 "freeNmapList(Port &port, int num_servlets)"，用于将 "Port" 类的对象加入一个名为 "PortList" 的队列中。


```cpp
class Port {
 friend class PortList;

 public:
  Port();
  void freeService(bool del_service);
  void freeScriptResults(void);
  void getNmapServiceName(char *namebuf, int buflen) const;

  u16 portno;
  u8 proto;
  u8 state;
  state_reason_t reason;

#ifndef NOLUA
  ScriptResults scriptResults;
```

这段代码定义了一个名为Private的类，其中包含一个名为service的指针变量，这个变量是通过调用抽象工厂类来分配的，而这个工厂类会在内存不足时分配内存。

private:
 serviceDeductions *service;

接着，定义了一个名为portlist_proto的枚举类型，这个枚举类型定义了四种不同的网络协议，PORTLIST_PROTO_TCP、PORTLIST_PROTO_UDP、PORTLIST_PROTO_SCTP和PORTLIST_PROTO_IP，以及一个名为PORTLIST_PROTO_MAX的枚举类型，PORTLIST_PROTO_MAX定义了最大的协议类型。

最后，在代码中使用了这些枚举类型来对输入数据进行匹配，以便根据需要采取不同的操作。


```cpp
#endif

 private:
  /* This is allocated only on demand by PortList::setServiceProbeResults
     to save memory for the many closed or filtered ports that don't need it. */
  serviceDeductions *service;
};


/* Needed enums to address some arrays. This values
 * should never be used directly. Use INPROTO2PORTLISTPROTO macro */
enum portlist_proto {	// PortList Protocols
  PORTLIST_PROTO_TCP	= 0,
  PORTLIST_PROTO_UDP	= 1,
  PORTLIST_PROTO_SCTP	= 2,
  PORTLIST_PROTO_IP	= 3,
  PORTLIST_PROTO_MAX	= 4
};

```

 void addTunnel(const struct sockaddr_storage *ip_addr,
                   enum service_tunnel_type tunnel, const char *product,
                   const char *version, const char *hostname,
                   const char *ostype, const char *devicetype,
                   const char *extrainfo, const std::vector<const char *> *cpe,
                   const char *fingerprint);

private:
 struct serviceDeductions *sd;

 int numscriptresults; /* Total number of scripts which produced output */

 /* Get number of ports in this state. This a sum for protocols. */
 int getStateCounts(int state) const;
 /* Get number of ports in this state for requested protocol. */
 int getStateCounts(int protocol, int state) const;

 // The service name, if any, of the service that produced the output.
 const char *sname;

 int protocol;
 const char *product;
 const char *version;
 const char *hostname;
 const char *ostype;
 const char *devicetype;
 const char *extrainfo;
 const std::vector<const char *> *cpe;
 const char *fingerprint;
};
```cpp

```
```cpp


```
class PortList {
 public:
  PortList();
  ~PortList();
  /* Set ports that will be scanned for each protocol. This function
   * must be called before any PortList object will be created. */
  static void initializePortMap(int protocol, u16 *ports, int portcount);
  /* Free memory used by port_map. It should be done somewhere before quitting*/
  static void freePortMap();

  void setDefaultPortState(u8 protocol, int state);
  void setPortState(u16 portno, u8 protocol, int state);
  int getPortState(u16 portno, u8 protocol);
  int forgetPort(u16 portno, u8 protocol);
  bool portIsDefault(u16 portno, u8 protocol);
  /* Saves an identification string for the target containing these
     ports (an IP address might be a good example, but set what you
     want).  Only used when printing new port updates.  Optional.  A
     copy is made. */
  void setIdStr(const char *id);
  /* A function for iterating through the ports.  Give NULL for the
   first "afterthisport".  Then supply the most recent returned port
   for each subsequent call.  When no more matching ports remain, NULL
   will be returned.  To restrict returned ports to just one protocol,
   specify IPPROTO_TCP, IPPROTO_UDP or UPPROTO_SCTP for
   allowed_protocol. A TCPANDUDPANDSCTP for allowed_protocol matches
   either. A 0 for allowed_state matches all possible states. This
   function returns ports in numeric order from lowest to highest,
   except that if you ask for TCP, UDP & SCTP, all TCP ports will be
   returned before we start returning UDP and finally SCTP ports */
  Port *nextPort(const Port *cur, Port *next,
                 int allowed_protocol, int allowed_state) const;

  int setStateReason(u16 portno, u8 proto, reason_t reason, u8 ttl, const struct sockaddr_storage *ip_addr);

  int numscriptresults; /* Total number of scripts which produced output */

  /* Get number of ports in this state. This a sum for protocols. */
  int getStateCounts(int state) const;
  /* Get number of ports in this state for requested protocol. */
  int getStateCounts(int protocol, int state) const;

  // sname should be NULL if sres is not
  // PROBESTATE_FINISHED_MATCHED. product,version, and/or extrainfo
  // will be NULL if unavailable. Note that this function makes its
  // own copy of sname and product/version/extrainfo.  This function
  // also takes care of truncating the version strings to a
  // 'reasonable' length if necessary, and cleaning up any unprintable
  // chars. (these tests are to avoid annoying DOS (or other) attacks
  // by malicious services).  The fingerprint should be NULL unless
  // one is available and the user should submit it.  tunnel must be
  // SERVICE_TUNNEL_NONE (normal) or SERVICE_TUNNEL_SSL (means ssl was
  // detected and we tried to tunnel through it ).
  void setServiceProbeResults(u16 portno, int protocol,
                              enum serviceprobestate sres, const char *sname,
                              enum service_tunnel_type tunnel, const char *product,
                              const char *version, const char *hostname,
                              const char *ostype, const char *devicetype,
                              const char *extrainfo,
                              const std::vector<const char *> *cpe,
                              const char *fingerprint);

  // pass in an allocated struct serviceDeductions (don't worry about initializing, and
  // you don't have to free any internal ptrs.  See the serviceDeductions definition for
  // the fields that are populated.
  void getServiceDeductions(u16 portno, int protocol, struct serviceDeductions *sd) const;

```cpp

The NextIgnoredState function takes an integer parameter representing the current state of the system and returns a boolean indicating whether the state should be ignored or not. It does this by checking if there's any active connection for the given state, and if there's no active connection, it means the state should be ignored.

isIgnoredState function takes two integer parameters, the current state integer and a count of the number of active connections for the given state. It returns true if the state should be ignored, and false otherwise.

numIgnoredStates function returns the total number of states that have been ignored.

numIgnoredPorts function returns the total number of ports that have been ignored.

The hasOpenPorts function checks if there are any open ports, and returns true if there are any open ports, false otherwise.

The mapPort function maps a port number to a counter of the number of active connections for that port.

The lookupPort function looks up a port number and protocol to the corresponding port structure in the portList array.

The createPort function creates a new port structure and returns it.

The setPortEntry function updates the port structure with the given port number, protocol, and the number of active connections for that port.

These functions are used to manage the state of the system, including the counting of active connections, the mapping of ports to active connections, and the filtering of open ports.


```
#ifndef NOLUA
  void addScriptResult(u16 portno, int protocol, ScriptResult *sr);
#endif

  /* Cycles through the 0 or more "ignored" ports which should be
   consolidated for Nmap output.  They are returned sorted by the
   number of ports in the state, starting with the most common.  It
   should first be called with PORT_UNKNOWN to obtain the most popular
   ignored state (if any).  Then call with that state to get the next
   most popular one.  Returns the state if there is one, but returns
   PORT_UNKNOWN if there are no (more) states which qualify for
   consolidation */
  int nextIgnoredState(int prevstate) const;

  /* Returns true if a state should be ignored (consolidated), false otherwise */
  bool isIgnoredState(int state, int *count) const;

  int numIgnoredStates() const;
  int numIgnoredPorts() const;
  int numPorts() const;
  bool hasOpenPorts() const;

  /* Returns true if service scan is done and portno is found to be tcpwrapped, false otherwise */
  bool isTCPwrapped(u16 portno) const;

 private:
  void mapPort(u16 *portno, u8 *protocol) const;
  /* Get Port structure from PortList structure.*/
  const Port *lookupPort(u16 portno, u8 protocol) const;
  Port *createPort(u16 portno, u8 protocol);
  /* Set Port structure to PortList structure.*/
  void  setPortEntry(u16 portno, u8 protocol, Port *port);

  /* A string identifying the system these ports are on.  Just used for
     printing open ports, if it is set with setIdStr() */
  char *idstr;
  /* Number of ports in each state per each protocol. */
  int state_counts_proto[PORTLIST_PROTO_MAX][PORT_HIGHEST_STATE];
  Port **port_list[PORTLIST_PROTO_MAX];
 protected:
  /* Maps port_number to index in port_list array.
   * Only functions: getPortEntry, setPortEntry, initializePortMap and
   * nextPort should access this structure directly. */
  static u16 *port_map[PORTLIST_PROTO_MAX];
  static u16 *port_map_rev[PORTLIST_PROTO_MAX];
  /* Number of allocated elements in port_list per each protocol. */
  static int port_list_count[PORTLIST_PROTO_MAX];
  Port default_port_state[PORTLIST_PROTO_MAX];
};

```cpp

这段代码是一个预处理指令，它用于告诉编译器在编译之前做一些事情。

具体来说，如果这段代码之前没有任何其他预处理指令，那么编译器会默认读取并包含这一行。否则，它将永远不会被包含，因为这是标着预处理指令但没有任何其他代码要读取的行。

因此，这个预处理指令只是在告诉编译器不要读取这一行，而不会对代码产生任何影响，因此代码仍然会被编译。


```
#endif


```cpp

# `portreasons.h`

This text is the official documentation for the Nmap network scanner. It explains the licensing terms for Nmap, which is a free and open-source tool for network discovery and vulnerability scanning.

The text outlines the main differences between the official Nmap version and the Nmap OEM edition. The Nmap OEM edition has a more permissive license and some special features, but it is intended for commercial use and redistribution with special许可.

The text also explains the licensing terms for the source code. It is released under the GPL (General Public License) and there are restrictions on the use and distribution of the code. The text also mentions the Npcap software, which is under its own separate license terms and cannot be redistributed without special permission.

Finally, the text emphasizes the importance of understanding the licensing terms and conditions when using Nmap, and also suggests ways for users to contribute to the project through a Github PR or email to the dev@nmap.org mailing list.


```
/***************************************************************************
 * portreasons.h -- Verbose packet-level information on port states        *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```cpp

这段代码定义了一个名为 "reason_h" 的头文件，其作用是定义了 "reason_h" 标准头文件的内容。

该头文件中包含两个嵌套的声明：

1. 包含自 "nbase.h" 头文件；
2. 包含一个名为 "Target" 的类定义，该类定义了一个名为 "PortList" 的类。

此外，该头文件还包含一个名为 "reason_h" 的定义，该定义在内部声明了一个名为 "REASON_H_" 的标识符，但没有定义任何数据成员量和函数成员量。


```
/*
 * Written by Eddie Bell <ejlbell@gmail.com> 2007
 * Modified by Colin Rice <dah4k0r@gmail.com> 2011
 */

#ifndef REASON_H
#define REASON_H

#include "nbase.h"

#include <sys/types.h>
#include <map>
class Target;
class PortList;

```cpp

这段代码定义了一个名为“reason_t”的无符号short型变量，用于表示各种字符串输出，该输出存储在一个名为“reason_string”的类中。

该代码还定义了一个名为“reason_string”的类，该类包含一个字符串成员变量，用于存储“reason_codes”的到一个字符串的映射。该成员变量包含一个指向一个枚举类型“reason_codes”的指针，以及一个指向一个字符串的指针，用于存储每个“reason_codes”对应的字符串。

该类的构造函数接受两个字符串参数，一个是单数形式的“singular”，另一个是复数形式的“plural”，然后构造一个相应编码的“reason_string”实例，并将其存储在内部map中。

该代码最后还定义了一个名为“reason_codes”的枚举类型，包含多个成员，用于表示各种可能的错误状态。


```
typedef unsigned short reason_t;

/* Holds various string outputs of a reason  *
 * Stored inside a map which maps enum_codes *
 * to reason_strings				     */
class reason_string {
public:
    //Required for map
    reason_string();
    reason_string(const char * singular, const char * plural);
    const char * singular;
    const char * plural;
};

/* stored inside a Port Object and describes
 * why a port is in a specific state */
```cpp

这段代码定义了一个名为 port_reason 的结构体，它表示网络中某种类型的原因。这个结构体包含以下字段：

1. reason_id：表示一个网络原因的唯一标识。
2. ip_addr：表示网络地址的union类型，可以是 IPv4 或 IPv6。
3. ttl：时间戳（ Time to Live ）服务器的寿命。
4. set_ip_addr：设置 ip 地址的函数，使用形参 sockaddr_storage 类型的变量。

此外，还定义了一个名为 state_reason_summary 的结构体，用于计算端口相关Reason的摘要。这个结构体包含以下字段：

1. reason_id：表示一个网络原因的唯一标识。
2. count：该摘要统计的端口数量。
3. next：指向下一个摘要的指针。
4. proto：该摘要所属的协议类型。
5. ports：一个长度为0xffff+1的数组，用于表示过滤的端口号。

该代码可能用于计算网络中的某种状态，例如计算 10 个端口的 no-responses 所导致的原因。


```
typedef struct port_reason {
        reason_t reason_id;
        union {
                struct sockaddr_in in;
                struct sockaddr_in6 in6;
                struct sockaddr sockaddr;
        } ip_addr;
        unsigned short ttl;

        int set_ip_addr(const struct sockaddr_storage *ss);
} state_reason_t;

/* used to calculate state reason summaries.
 * I.E 10 ports filter because of 10 no-responses */
typedef struct port_reason_summary {
        reason_t reason_id;
        unsigned int count;
        struct port_reason_summary *next;
        unsigned short proto;
        unsigned short ports[0xffff+1];
} state_reason_summary_t;


```cpp

这段代码定义了一个枚举类型，名为“reason_codes”，列举了可能出现的 Reasons for Action (RFC) 代码。这些 RFC 代码用于表示网络中的各种错误和问题，如：

- ER_RESETPEER：站点重置
- ER_CONREFUSED：客户端无法重置
- ER_CONACCEPT：客户端已接受
- ER_SYNACK：同步确认
- ER_SYN：同步
- ER_UDPRESPONSE：无状态设备响应
- ER_PROTORESPONSE：协议路由器响应
- ER_ACCES：访问
- ER_NETUNREACH：网络不可达
- ER_HOSTUNREACH：主机不可达
- ER_PROTOUNREACH：协议无法到达
- ER_PORTUNREACH：端口无法到达
- ER_ECHOREPLY：错误消息回复
- ER_DESTUNREACH：数据报丢失
- ER_SOURCEQUENCH：发送者关闭
- ER_NETPROHIBITED：网络禁止
- ER_HOSTPROHIBITED：主机禁止
- ER_ADMINPROHIBITED：管理员禁止
- ER_TIMEEXCEEDED：超时
- ER_TIMESTAMPREPLY：时间戳回复
- ER_ADDRESSMASKREPLY：地址掩码回复
- ER_NOIPIDCHANGE：IP ID 无变化
- ER_IPIDCHANGE：IP ID 变化
- ER_ARPRESPONSE：应用响应
- ER_NDRESPONSE：路由器响应
- ER_TCPRESPONSE：传输控制协议 (TCP) 响应
- ER_NORESPONSE：非路由器响应
- ER_INITACK：初始化标记
- ER_ABORT：终止


```
enum reason_codes {
        ER_RESETPEER, ER_CONREFUSED, ER_CONACCEPT,
        ER_SYNACK, ER_SYN, ER_UDPRESPONSE, ER_PROTORESPONSE, ER_ACCES,

        ER_NETUNREACH, ER_HOSTUNREACH, ER_PROTOUNREACH,
        ER_PORTUNREACH, ER_ECHOREPLY,

        ER_DESTUNREACH, ER_SOURCEQUENCH, ER_NETPROHIBITED,
        ER_HOSTPROHIBITED, ER_ADMINPROHIBITED,
        ER_TIMEEXCEEDED, ER_TIMESTAMPREPLY,

        ER_ADDRESSMASKREPLY, ER_NOIPIDCHANGE, ER_IPIDCHANGE,
        ER_ARPRESPONSE, ER_NDRESPONSE, ER_TCPRESPONSE, ER_NORESPONSE,
        ER_INITACK, ER_ABORT,
        ER_LOCALHOST, ER_SCRIPT, ER_UNKNOWN, ER_USER,
        ER_NOROUTE, ER_BEYONDSCOPE, ER_REJECTROUTE, ER_PARAMPROBLEM,
};

```cpp

这段代码定义了一个名为`reason_map_type`的类，它是一个`std::map<reason_codes,reason_string>`类型的数据结构，用于存储错误代码及其对应的字符串表示。

该类的构造函数在创建一个空`std::map<reason_codes,reason_string>`对象后，将`reason_map`映射到私有成员中。

该类的`find`函数用于在`reason_map`中查找给定`reason_codes`的错误代码，并返回一个指向该错误代码的`std::map<reason_codes,reason_string>::const_iterator`类型的指针。如果所查找的错误代码不存在于`reason_map`中，函数将返回`reason_map.find(ER_UNKNOWN)`返回的指针，其中`ER_UNKNOWN`是`std::map<reason_codes,reason_string>`中的默认错误代码。

该代码提供了一个方便的接口来查找给定错误代码的对应字符串表示。例如，你可以在代码中这样使用该类：
```
reason_map_type my_map; // 定义一个reason_map_type类型的对象
my_map.find(REASON_CODE_ARITH_除法错误) << " should be divided by 2.";
```cpp 
如果编译器没有找到该错误代码，或者你传递的错误代码不正确，函数将返回一个空的`std::map<reason_codes,reason_string>::const_iterator`类型的指针，或者将返回`reason_map.find(ER_UNKNOWN)`返回的指针。


```
/* A map of reason_codes to plural and singular *
 * versions of the error string                 */
class reason_map_type{
private:
    std::map<reason_codes,reason_string > reason_map;
public:
    reason_map_type();
    std::map<reason_codes,reason_string>::const_iterator find(const reason_codes& x) const {
        std::map<reason_codes,reason_string>::const_iterator itr = reason_map.find(x);
        if(itr == reason_map.end())
            return reason_map.find(ER_UNKNOWN);
        return itr;
    };
};

```cpp

这段代码定义了一个名为 `reason_codes` 的函数，它接收两个整数参数 `proto` 和 `icmp_type`，并返回一个整数 `reason`。这个函数的作用是将 ICMP 代码转换为 reason 代码。

接下来定义了一个宏 `#define SINGULAR 1` 和 `#define PLURAL 2`，定义了两个与 `SINGULAR` 和 `PLURAL` 相关的整数。

然后定义了一个名为 `state_reason_init` 的函数，接受一个 `state_reason_t` 类型的指针参数 `reason`。这个函数的作用似乎是初始化 reason 对象。

接着定义了一个名为 `reason_str` 的函数，接收一个 `reason_id` 和一个表示该 reason 对象的 `number` 参数。这个函数的作用是将 reason 代码转换为字符串，其中如果 number 大于 1，则使用 plural 形式的字符串，否则使用 singular 形式的字符串。具体，number 的值可以是 1 或 2。

最后，在程序的底部，定义了一个名为 `main` 的函数，不知道它的具体作用。


```
/* Function to translate ICMP code and typ to reason code */
reason_codes icmp_to_reason(u8 proto, int icmp_type, int icmp_code);

/* Passed to reason_str to determine if string should be in
 * plural of singular form */
#define SINGULAR 1
#define PLURAL 2

void state_reason_init(state_reason_t *reason);

/* converts a reason_id to a string. number represents the
 * amount ports in a given state. If there is more then one
 * port the plural is used, otherwise the singular is used. */
const char *reason_str(reason_t reason_id, unsigned int number);

```cpp

该代码定义了两个函数，get_state_reason_summary 和 state_reason_summary_dinit，以及一个名为 port_reason_str 的函数。

get_state_reason_summary函数接收一个名为 Ports 的链表和一个整数 state，返回一个链表，该链表包含了该状态下所有出现过的端口的理由。

state_reason_summary_dinit函数接收一个名为 r 的链表头节点，该链表头节点存储了一个 state_reason_summary 类型的数据结构。该函数将释放该链表，并将其从 get_state_reason_summary 函数中卸载。

port_reason_str 函数根据传入的 state 和源 IP 地址返回一个字符串，表示该端口处于该 state 下的原因。

target_reason_str 函数根据传入的 target 和源 IP 地址返回一个字符串，表示该目标处于该 state 下的原因。


```
/* Returns a linked list of reasons why ports are in a given state */
state_reason_summary_t *get_state_reason_summary(const PortList *Ports, int state);
/* Frees the linked list from get_state_reason_summary */
void state_reason_summary_dinit(state_reason_summary_t *r);

/* Build an output string based on reason and source ip address.
 * Uses static return value so previous values will be over
 * written by subsequent calls */
const char *port_reason_str(state_reason_t r);
const char *target_reason_str(const Target *t);

#endif


```cpp

# `probespec.h`

This is a text-based AI that is licensed under the Nmap license. The Nmap license is permissive and allows users to use, modify, and redistribute Nmap, but it also has certain restrictions. For example, the license prohibits companies from using and redistributing Nmap in commercial products, but allows the use and redistribution of a special Nmap OEM edition. Additionally, the license for the official Nmap Windows builds includes the Npcap software, which is under a separate license term that prohibits redistribution without special permission.


```
#ifndef PROBESPEC_H
#define PROBESPEC_H
/***************************************************************************
 * probespec.h -- Defines structures which specify probes to network ports *
 * and protocols.                                                          *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/
```cpp

这段代码定义了三种结构体：probespec_tcpdata、probespec_udpdata和probespec_sctpdata。它们都包含一个名为dport的16位无符号整数，表示传输数据的端口号。这些结构体可能还包含其他标志位和长度，但这一部分代码只关注端口号。

这段代码可能是为了在传输层协议（如TCP、UDP或SCTP）中传输数据而设计的。dport字段表示数据将通过哪种协议传输，而flags字段可能包含有关数据分段的一些信息。


```
#include <nbase.h>

struct probespec_tcpdata {
  u16 dport;
  u8 flags;
};

struct probespec_udpdata {
  u16 dport;
};

struct probespec_sctpdata {
  u16 dport;
  u8 chunktype;
};

```cpp

这段代码定义了两个结构体：probespec_icmpdata和probespec_icmpv6data。这两个结构体都包含两个u8类型的成员变量：type和code。

定义了一系列缩写的常量：PS_NONE、PS_TCP、PS_UDP和PS_PROTO，它们定义了协议类型的不同。接着，定义了一些宏，它们定义了可以使用的协议类型。

最后，在probespec_icmpdata结构体中定义了type和code两个成员变量。在probespec_icmpv6data结构体中，对probespec_icmpdata结构体中的type成员变量进行了扩展，定义了PS_NONE到PS_PROTO的四个成员变量。


```
struct probespec_icmpdata {
  u8 type;
  u8 code;
};

struct probespec_icmpv6data {
  u8 type;
  u8 code;
};

#define PS_NONE 0
#define PS_TCP 1
#define PS_UDP 2
#define PS_PROTO 3
#define PS_ICMP 4
```cpp

这段代码定义了一个名为“probespec”的结构体，用于保存IP协议数据。这个结构体有以下几种形式：

1. PS_ARP：ARP协议数据。
2. PS_CONNECTTCP：TCP连接数据。
3. PS_SCTP：SCTP连接数据。
4. PS_ICMPV6：ICMPv6协议数据。
5. PS_ND：没有定义，可能是保留的。

结构体的类型字段被定义为 u8，表示这个结构体的大小为 8 字节。这个结构体中包含一个名为“pd”的成员，它是一个 union类型的数据结构，包含以下字段：

1. PS_ARP：如果类型是 PS_ARP，那么包含一个名为“tcp”的成员变量，这是一个 struct type=u8,protype=u8 的结构体，保存了 TCP 数据。
2. PS_CONNECTTCP：如果类型是 PS_CONNECTTCP，那么包含一个名为“udp”的成员变量，这是一个 struct type=u8,protype=u8 的结构体，保存了 UDP 数据。
3. PS_SCTP：如果类型是 PS_SCTP，那么包含一个名为“sctp”的成员变量，这是一个 struct type=u8,protype=u8 的结构体，保存了 SCTP 数据。
4. PS_ICMPV6：如果类型是 PS_ICMPV6，那么包含一个名为“icmpv6”的成员变量，这是一个 struct type=u8,protype=u8 的结构体，保存了 ICMPv6 数据。
5. PS_ND：如果类型是 PS_ND，那么保存了一个成员变量，但这个成员变量没有被定义。


```
#define PS_ARP 5
#define PS_CONNECTTCP 6
#define PS_SCTP 7
#define PS_ICMPV6 8
#define PS_ND 9

/* The size of this structure is critical, since there can be tens of
   thousands of them stored together ... */
typedef struct probespec {
  /* To save space, I changed this from private enum (took 4 bytes) to
     u8 that uses #defines above */
  u8 type;
  u8 proto; /* If not PS_ARP -- Protocol number ... eg IPPROTO_TCP, etc. */
  union {
    struct probespec_tcpdata tcp; /* If type is PS_TCP or PS_CONNECTTCP. */
    struct probespec_udpdata udp; /* PS_UDP */
    struct probespec_sctpdata sctp; /* PS_SCTP */
    struct probespec_icmpdata icmp; /* PS_ICMP */
    struct probespec_icmpv6data icmpv6; /* PS_ICMPV6 */
    /* Nothing needed for PS_ARP, since src mac and target IP are
       avail from target structure anyway */
  } pd;
} probespec;

```cpp

这是一个C语言中的preprocess指令，通常用于预处理包含一个或多个预处理指令的代码行。

在这里，代码中只包含一个preprocess指令，它是一个标识，告诉编译器在编译之前需要做的一项事情。具体来说，它是一个条件编译，用于检查当前是否支持某种特定形式的代码，如果支持，则不做任何操作，否则会执行preprocess指令中定义的操作。

在这个例子中，preprocess指令后跟了一个注释，告诉编译器不要做任何操作，即不会输出任何预处理指令。


```
#endif

```