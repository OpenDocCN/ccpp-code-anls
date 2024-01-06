# `PowerInfer\examples\server\httplib.h`

```
// 包含头文件 httplib.h
// 版权声明
// 使用 MIT 许可证
#ifndef CPPHTTPLIB_HTTPLIB_H
#define CPPHTTPLIB_HTTPLIB_H

// 定义 CPPHTTPLIB_VERSION 为 "0.12.2"
#define CPPHTTPLIB_VERSION "0.12.2"

/*
 * 配置
 */

// 如果未定义 CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND，则定义为 5
#ifndef CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND
#define CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND 5
#endif
// 如果未定义 CPPHTTPLIB_KEEPALIVE_MAX_COUNT，则设置为默认值 5
#ifndef CPPHTTPLIB_KEEPALIVE_MAX_COUNT
#define CPPHTTPLIB_KEEPALIVE_MAX_COUNT 5
#endif

// 如果未定义 CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND，则设置为默认值 300
#ifndef CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND
#define CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND 300
#endif

// 如果未定义 CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND，则设置为默认值 0
#ifndef CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND
#define CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND 0
#endif

// 如果未定义 CPPHTTPLIB_READ_TIMEOUT_SECOND，则设置为默认值 5
#ifndef CPPHTTPLIB_READ_TIMEOUT_SECOND
#define CPPHTTPLIB_READ_TIMEOUT_SECOND 5
#endif

// 如果未定义 CPPHTTPLIB_READ_TIMEOUT_USECOND，则设置为默认值 0
#ifndef CPPHTTPLIB_READ_TIMEOUT_USECOND
#define CPPHTTPLIB_READ_TIMEOUT_USECOND 0
#endif
// 如果未定义 CPPHTTPLIB_WRITE_TIMEOUT_SECOND，则设置为 5
#ifndef CPPHTTPLIB_WRITE_TIMEOUT_SECOND
#define CPPHTTPLIB_WRITE_TIMEOUT_SECOND 5
#endif

// 如果未定义 CPPHTTPLIB_WRITE_TIMEOUT_USECOND，则设置为 0
#ifndef CPPHTTPLIB_WRITE_TIMEOUT_USECOND
#define CPPHTTPLIB_WRITE_TIMEOUT_USECOND 0
#endif

// 如果未定义 CPPHTTPLIB_IDLE_INTERVAL_SECOND，则设置为 0
#ifndef CPPHTTPLIB_IDLE_INTERVAL_SECOND
#define CPPHTTPLIB_IDLE_INTERVAL_SECOND 0
#endif

// 如果未定义 CPPHTTPLIB_IDLE_INTERVAL_USECOND
#ifdef _WIN32
// 如果是 Windows 系统，则设置为 10000
#define CPPHTTPLIB_IDLE_INTERVAL_USECOND 10000
#else
// 如果不是 Windows 系统，则设置为 0
#define CPPHTTPLIB_IDLE_INTERVAL_USECOND 0
#endif
#endif
// 如果未定义 CPPHTTPLIB_REQUEST_URI_MAX_LENGTH，则设置为 8192
#ifndef CPPHTTPLIB_REQUEST_URI_MAX_LENGTH
#define CPPHTTPLIB_REQUEST_URI_MAX_LENGTH 8192
#endif

// 如果未定义 CPPHTTPLIB_HEADER_MAX_LENGTH，则设置为 8192
#ifndef CPPHTTPLIB_HEADER_MAX_LENGTH
#define CPPHTTPLIB_HEADER_MAX_LENGTH 8192
#endif

// 如果未定义 CPPHTTPLIB_REDIRECT_MAX_COUNT，则设置为 20
#ifndef CPPHTTPLIB_REDIRECT_MAX_COUNT
#define CPPHTTPLIB_REDIRECT_MAX_COUNT 20
#endif

// 如果未定义 CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT，则设置为 1024
#ifndef CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT
#define CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT 1024
#endif

// 如果未定义 CPPHTTPLIB_PAYLOAD_MAX_LENGTH，则设置为 size_t 类型的最大值
#ifndef CPPHTTPLIB_PAYLOAD_MAX_LENGTH
#define CPPHTTPLIB_PAYLOAD_MAX_LENGTH ((std::numeric_limits<size_t>::max)())
#endif
// 如果未定义 CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH，则设置默认值为 8192
#ifndef CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH
#define CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH 8192
#endif

// 如果未定义 CPPHTTPLIB_TCP_NODELAY，则设置默认值为 false
#ifndef CPPHTTPLIB_TCP_NODELAY
#define CPPHTTPLIB_TCP_NODELAY false
#endif

// 如果未定义 CPPHTTPLIB_RECV_BUFSIZ，则设置默认值为 4096
#ifndef CPPHTTPLIB_RECV_BUFSIZ
#define CPPHTTPLIB_RECV_BUFSIZ size_t(4096u)
#endif

// 如果未定义 CPPHTTPLIB_COMPRESSION_BUFSIZ，则设置默认值为 16384
#ifndef CPPHTTPLIB_COMPRESSION_BUFSIZ
#define CPPHTTPLIB_COMPRESSION_BUFSIZ size_t(16384u)
#endif

// 如果未定义 CPPHTTPLIB_THREAD_POOL_COUNT，则设置默认值为 8 或者硬件线程数减 1
#ifndef CPPHTTPLIB_THREAD_POOL_COUNT
#define CPPHTTPLIB_THREAD_POOL_COUNT                                           \
  ((std::max)(8u, std::thread::hardware_concurrency() > 0                      \
                      ? std::thread::hardware_concurrency() - 1
// 定义接收数据的默认标志位为0
#ifndef CPPHTTPLIB_RECV_FLAGS
#define CPPHTTPLIB_RECV_FLAGS 0
#endif

// 定义发送数据的默认标志位为0
#ifndef CPPHTTPLIB_SEND_FLAGS
#define CPPHTTPLIB_SEND_FLAGS 0
#endif

// 定义监听连接的默认最大排队数为5
#ifndef CPPHTTPLIB_LISTEN_BACKLOG
#define CPPHTTPLIB_LISTEN_BACKLOG 5
#endif

/*
 * Headers
 */

// 如果是在 Windows 系统下编译，则执行以下代码
#ifdef _WIN32
// 如果未定义 _CRT_SECURE_NO_WARNINGS，则定义它
#ifndef _CRT_SECURE_NO_WARNINGS
#define _CRT_SECURE_NO_WARNINGS
#endif //_CRT_SECURE_NO_WARNINGS

// 如果未定义 _CRT_NONSTDC_NO_DEPRECATE，则定义它
#ifndef _CRT_NONSTDC_NO_DEPRECATE
#define _CRT_NONSTDC_NO_DEPRECATE
#endif //_CRT_NONSTDC_NO_DEPRECATE

// 如果编译器为 Visual Studio，并且版本低于 1900（2015），则报错
#if defined(_MSC_VER)
#if _MSC_VER < 1900
#error Sorry, Visual Studio versions prior to 2015 are not supported
#endif

// 添加链接到 ws2_32.lib 库
#pragma comment(lib, "ws2_32.lib")

// 如果是 64 位 Windows 系统，则使用 __int64 类型作为 ssize_t
#ifdef _WIN64
using ssize_t = __int64;
// 否则使用 long 类型作为 ssize_t
#else
using ssize_t = long;
#endif
// 如果不是 MSC 编译器，则定义 _MSC_VER
#endif // _MSC_VER

// 如果没有定义 S_ISREG 宏，则定义 S_ISREG 宏
#ifndef S_ISREG
#define S_ISREG(m) (((m)&S_IFREG) == S_IFREG)
#endif // S_ISREG

// 如果没有定义 S_ISDIR 宏，则定义 S_ISDIR 宏
#ifndef S_ISDIR
#define S_ISDIR(m) (((m)&S_IFDIR) == S_IFDIR)
#endif // S_ISDIR

// 如果没有定义 NOMINMAX 宏，则定义 NOMINMAX 宏
#ifndef NOMINMAX
#define NOMINMAX
#endif // NOMINMAX

// 包含头文件 io.h, winsock2.h, ws2tcpip.h
#include <io.h>
#include <winsock2.h>
#include <ws2tcpip.h>

// 如果没有定义 WSA_FLAG_NO_HANDLE_INHERIT 宏，则定义 WSA_FLAG_NO_HANDLE_INHERIT 宏
#ifndef WSA_FLAG_NO_HANDLE_INHERIT
#define WSA_FLAG_NO_HANDLE_INHERIT 0x80
#endif
// 如果没有定义 strcasecmp，则定义为 _stricmp
#ifndef strcasecmp
#define strcasecmp _stricmp
#endif // strcasecmp

// 使用 socket_t 别名表示 SOCKET 类型
using socket_t = SOCKET;
#ifdef CPPHTTPLIB_USE_POLL
// 如果定义了 CPPHTTPLIB_USE_POLL，则使用 WSAPoll
#define poll(fds, nfds, timeout) WSAPoll(fds, nfds, timeout)
#endif

#else // not _WIN32

#include <arpa/inet.h>
#ifndef _AIX
#include <ifaddrs.h>
#endif
#include <net/if.h>
#include <netdb.h>
#include <netinet/in.h>
```
#ifdef __linux__
#include <resolv.h>  // 包含 Linux 系统的 DNS 解析库
#endif
#include <netinet/tcp.h>  // 包含 TCP 协议相关的头文件
#ifdef CPPHTTPLIB_USE_POLL
#include <poll.h>  // 如果使用了 poll，包含 poll 相关的头文件
#endif
#include <csignal>  // 包含信号处理相关的头文件
#include <pthread.h>  // 包含多线程相关的头文件
#include <sys/select.h>  // 包含 select 系统调用相关的头文件
#include <sys/socket.h>  // 包含 socket 相关的头文件
#include <sys/un.h>  // 包含 UNIX 域套接字相关的头文件
#include <unistd.h>  // 包含 POSIX 系统调用相关的头文件

using socket_t = int;  // 定义 socket_t 类型为整型
#ifndef INVALID_SOCKET
#define INVALID_SOCKET (-1)  // 如果未定义 INVALID_SOCKET，则定义为 -1
#endif
#endif //_WIN32  // 结束 _WIN32 的定义
#include <algorithm>               // 包含算法库，提供各种算法函数
#include <array>                   // 包含数组库，提供固定大小的数组容器
#include <atomic>                  // 包含原子操作库，提供原子操作类型和函数
#include <cassert>                 // 包含断言库，提供断言宏
#include <cctype>                  // 包含字符处理库，提供字符处理函数
#include <climits>                 // 包含限制库，提供各种常量
#include <condition_variable>      // 包含条件变量库，提供条件变量类
#include <cstring>                 // 包含字符串库，提供字符串处理函数
#include <errno.h>                 // 包含错误库，提供错误码宏
#include <fcntl.h>                 // 包含文件控制库，提供文件控制函数
#include <fstream>                 // 包含文件流库，提供文件流类
#include <functional>              // 包含函数对象库，提供函数对象类
#include <iomanip>                 // 包含格式控制库，提供格式控制函数
#include <iostream>                // 包含输入输出流库，提供输入输出流类
#include <list>                    // 包含列表库，提供双向链表容器
#include <map>                     // 包含映射库，提供映射容器
#include <memory>                  // 包含内存库，提供智能指针类
#include <mutex>                   // 包含互斥量库，提供互斥量类
#include <random>                  // 包含随机数库，提供随机数生成器
#include <regex>                   // 包含正则表达式库，提供正则表达式类
#include <set> // 包含 set 头文件
#include <sstream> // 包含 stringstream 头文件
#include <string> // 包含 string 头文件
#include <sys/stat.h> // 包含系统 stat 头文件
#include <thread> // 包含线程头文件

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT // 如果定义了 CPPHTTPLIB_OPENSSL_SUPPORT

#ifdef _WIN32 // 如果是在 Windows 环境下
#include <wincrypt.h> // 包含 Windows 加密 API 头文件

// 下面这些在 wincrypt.h 中已经定义了，如果使用 BoringSSL 会导致编译错误
#undef X509_NAME
#undef X509_CERT_PAIR
#undef X509_EXTENSIONS
#undef PKCS7_SIGNER_INFO

#ifdef _MSC_VER // 如果是在使用 Microsoft Visual C++ 编译器下
#pragma comment(lib, "crypt32.lib") // 添加 crypt32 库的链接
#pragma comment(lib, "cryptui.lib") // 添加 cryptui 库的链接
// 如果定义了 _WIN32 并且定义了 OPENSSL_USE_APPLINK，则包含 openssl/applink.c 文件
#if defined(_WIN32) && defined(OPENSSL_USE_APPLINK)
#include <openssl/applink.c>
#endif

// 包含 OpenSSL 库的头文件
#include <openssl/err.h>
#include <openssl/evp.h>
#include <openssl/ssl.h>
#include <openssl/x509v3.h>

// 如果定义了 CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN 并且在苹果系统上，则包含相关头文件
#if defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#include <TargetConditionals.h>
#if TARGET_OS_OSX
#include <CoreFoundation/CoreFoundation.h>
#include <Security/Security.h>
#endif // TARGET_OS_OSX
#endif // _WIN32

// 包含输入输出流的头文件
#include <iostream>
#include <sstream>
#if OPENSSL_VERSION_NUMBER < 0x1010100fL
#error Sorry, OpenSSL versions prior to 1.1.1 are not supported
#elif OPENSSL_VERSION_NUMBER < 0x30000000L
#define SSL_get1_peer_certificate SSL_get_peer_certificate
#endif
```
- 如果 OpenSSL 版本号小于 1.1.1，则抛出错误，表示不支持该版本
- 否则，如果 OpenSSL 版本号小于 3.0.0，则定义 SSL_get1_peer_certificate 为 SSL_get_peer_certificate

```
#endif
```
- 结束条件编译指令

```
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
#include <zlib.h>
#endif
```
- 如果定义了 CPPHTTPLIB_ZLIB_SUPPORT，则包含 zlib.h 头文件

```
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
#include <brotli/decode.h>
#include <brotli/encode.h>
#endif
```
- 如果定义了 CPPHTTPLIB_BROTLI_SUPPORT，则包含 brotli/decode.h 和 brotli/encode.h 头文件

```
/*
 * Declaration
```
- 声明部分的注释，表示接下来是声明部分的代码
/*
 * 命名空间 httplib 中的细节命名空间
 */
namespace httplib {

namespace detail {

/*
 * 从 C++14 中回溯 std::make_unique。
 *
 * 注意：这段代码是从以下 stackoverflow 帖子中得到的：
 * https://stackoverflow.com/questions/10149840/c-arrays-and-make-unique
 */
 
/*
 * 模板函数，用于创建一个唯一指针，返回类型为 T 类型的对象
 * 如果 T 不是数组类型，则返回一个指向 T 类型对象的唯一指针
 */
template <class T, class... Args>
typename std::enable_if<!std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(Args &&...args) {
  return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

/*
 * 模板函数，用于创建一个唯一指针，返回类型为 T 类型的对象
 * 如果 T 是数组类型，则返回一个指向 T 类型对象的唯一指针
 */
template <class T>
// 如果 T 是数组类型，则返回一个包含 T 类型的唯一指针
typename std::enable_if<std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(std::size_t n) {
  // 定义类型 RT 为 T 的基础类型
  typedef typename std::remove_extent<T>::type RT;
  // 返回一个包含 n 个 RT 类型对象的唯一指针
  return std::unique_ptr<T>(new RT[n]);
}

// 定义一个函数对象 ci，用于比较两个字符串，不区分大小写
struct ci {
  bool operator()(const std::string &s1, const std::string &s2) const {
    // 使用 lexicographical_compare 函数比较两个字符串，忽略大小写
    return std::lexicographical_compare(s1.begin(), s1.end(), s2.begin(),
                                        s2.end(),
                                        [](unsigned char c1, unsigned char c2) {
                                          // 将字符转换为小写后比较
                                          return ::tolower(c1) < ::tolower(c2);
                                        });
  }
};

// 定义一个结构体 scope_exit
// 基于 "http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4189"。
struct scope_exit {
// 定义一个带有参数为函数对象的构造函数，用于初始化 exit_function 和 execute_on_destruction
explicit scope_exit(std::function<void(void)> &&f)
    : exit_function(std::move(f)), execute_on_destruction{true} {}

// 定义一个移动构造函数，用于将另一个 scope_exit 对象的资源移动到当前对象
scope_exit(scope_exit &&rhs)
    : exit_function(std::move(rhs.exit_function)),
      execute_on_destruction{rhs.execute_on_destruction} {
  // 释放原对象的资源
  rhs.release();
}

// 定义析构函数，用于在对象销毁时执行 exit_function
~scope_exit() {
  if (execute_on_destruction) { this->exit_function(); }
}

// 定义一个 release 函数，用于释放对象的资源
void release() { this->execute_on_destruction = false; }

// 私有成员，禁止拷贝构造函数和赋值操作符
private:
  scope_exit(const scope_exit &) = delete;
  void operator=(const scope_exit &) = delete;
  scope_exit &operator=(scope_exit &&) = delete;
// 定义一个名为exit_function的函数对象，该函数没有参数和返回值
std::function<void(void)> exit_function;

// 定义一个名为execute_on_destruction的布尔变量
bool execute_on_destruction;
};

} // namespace detail

// 使用detail命名空间中的ci结构的多重映射作为Headers类型
using Headers = std::multimap<std::string, std::string, detail::ci>;

// 使用标准库中的多重映射作为Params类型
using Params = std::multimap<std::string, std::string;

// 使用标准库中的smatch作为Match类型
using Match = std::smatch;

// 定义一个名为Progress的函数对象，该函数接受两个uint64_t类型的参数并返回一个布尔值
using Progress = std::function<bool(uint64_t current, uint64_t total)>;

// 定义一个名为ResponseHandler的函数对象，该函数接受一个Response类型的引用并返回一个布尔值
using ResponseHandler = std::function<bool(const Response &response)>;

// 定义一个名为MultipartFormData的结构体，包含三个字符串类型的成员变量
struct MultipartFormData {
  std::string name;
  std::string content;
  std::string filename;
// 定义一个字符串变量 content_type
std::string content_type;
};

// 定义一个类型为 MultipartFormData 的向量
using MultipartFormDataItems = std::vector<MultipartFormData>;
// 定义一个键为字符串，值为 MultipartFormData 的多重映射
using MultipartFormDataMap = std::multimap<std::string, MultipartFormData>;

// 定义一个数据接收器类
class DataSink {
public:
  // 构造函数，初始化输出流和字符串流
  DataSink() : os(&sb_), sb_(*this) {}

  // 禁用拷贝构造函数和赋值运算符重载
  DataSink(const DataSink &) = delete;
  DataSink &operator=(const DataSink &) = delete;
  // 禁用移动构造函数和移动赋值运算符重载
  DataSink(DataSink &&) = delete;
  DataSink &operator=(DataSink &&) = delete;

  // 写入数据的函数对象
  std::function<bool(const char *data, size_t data_len)> write;
  // 完成时的函数对象
  std::function<void()> done;
  // 带尾部的完成函数对象
  std::function<void(const Headers &trailer)> done_with_trailer;
  // 输出流对象
  std::ostream os;

private:
# 定义一个名为data_sink_streambuf的类，继承自std::streambuf
class data_sink_streambuf : public std::streambuf {
public:
  # 构造函数，接受一个DataSink类型的参数，并将其赋值给成员变量sink_
  explicit data_sink_streambuf(DataSink &sink) : sink_(sink) {}

protected:
  # 重写xsputn方法，将输入的字符数组s写入到DataSink中，并返回写入的字符数
  std::streamsize xsputn(const char *s, std::streamsize n) {
    sink_.write(s, static_cast<size_t>(n));
    return n;
  }

private:
  # 成员变量，用于保存DataSink对象的引用
  DataSink &sink_;
};

# 声明一个名为sb_的data_sink_streambuf对象

using ContentProvider = 
    # 定义一个名为ContentProvider的别名，表示一个接受offset、length和DataSink参数的函数，并返回布尔值
    std::function<bool(size_t offset, size_t length, DataSink &sink)>;
// 定义一个类型别名 ContentProviderWithoutLength，表示一个没有长度信息的内容提供器
using ContentProviderWithoutLength = std::function<bool(size_t offset, DataSink &sink)>;

// 定义一个类型别名 ContentProviderResourceReleaser，表示一个资源释放器
using ContentProviderResourceReleaser = std::function<void(bool success)>;

// 定义一个结构体 MultipartFormDataProvider，包含表单数据的名称、内容提供器、文件名和内容类型
struct MultipartFormDataProvider {
  std::string name; // 表单数据的名称
  ContentProviderWithoutLength provider; // 内容提供器
  std::string filename; // 文件名
  std::string content_type; // 内容类型
};
// 定义一个类型别名 MultipartFormDataProviderItems，表示多个表单数据提供器的集合
using MultipartFormDataProviderItems = std::vector<MultipartFormDataProvider>;

// 定义一个类型别名 ContentReceiverWithProgress，表示一个带有进度信息的内容接收器
using ContentReceiverWithProgress =
    std::function<bool(const char *data, size_t data_length, uint64_t offset,
                       uint64_t total_length)>;

// 定义一个类型别名 ContentReceiver，表示一个内容接收器
using ContentReceiver =
    std::function<bool(const char *data, size_t data_length)>;
// 使用 MultipartFormData 类型的函数作为参数，返回布尔值
using MultipartContentHeader = std::function<bool(const MultipartFormData &file)>;

// ContentReader 类
class ContentReader {
public:
  // 定义 Reader 类型为接受 ContentReceiver 参数并返回布尔值的函数
  using Reader = std::function<bool(ContentReceiver receiver)>;
  // 定义 MultipartReader 类型为接受 MultipartContentHeader 和 ContentReceiver 参数并返回布尔值的函数
  using MultipartReader = std::function<bool(MultipartContentHeader header, ContentReceiver receiver)>;

  // ContentReader 构造函数，接受 Reader 和 MultipartReader 作为参数
  ContentReader(Reader reader, MultipartReader multipart_reader)
      : reader_(std::move(reader)),
        multipart_reader_(std::move(multipart_reader)) {}

  // 重载运算符，接受 MultipartContentHeader 和 ContentReceiver 参数，调用 multipart_reader_ 函数并返回其结果
  bool operator()(MultipartContentHeader header, ContentReceiver receiver) const {
    return multipart_reader_(std::move(header), std::move(receiver));
  }

  // 重载运算符，接受 ContentReceiver 参数，调用 reader_ 函数并返回其结果
  bool operator()(ContentReceiver receiver) const {
    return reader_(std::move(receiver));
  }
  }

  // 定义一个名为 reader_ 的 Reader 对象
  Reader reader_;
  // 定义一个名为 multipart_reader_ 的 MultipartReader 对象
  MultipartReader multipart_reader_;
};

// 使用 Range 类型定义一个名为 Range 的别名
using Range = std::pair<ssize_t, ssize_t>;
// 使用 Ranges 类型定义一个名为 Ranges 的别名
using Ranges = std::vector<Range>;

// 定义一个名为 Request 的结构体
struct Request {
  // 请求方法
  std::string method;
  // 请求路径
  std::string path;
  // 请求头部
  Headers headers;
  // 请求体
  std::string body;

  // 远程地址
  std::string remote_addr;
  // 远程端口，默认值为 -1
  int remote_port = -1;
  // 本地地址
  std::string local_addr;
  // 本地端口，默认值为 -1
  int local_port = -1;
  // 用于服务器端的变量声明
  std::string version;  // 版本号
  std::string target;   // 目标
  Params params;        // 参数
  MultipartFormDataMap files;  // 文件数据
  Ranges ranges;        // 范围
  Match matches;        // 匹配

  // 用于客户端的变量声明
  ResponseHandler response_handler;  // 响应处理器
  ContentReceiverWithProgress content_receiver;  // 带进度的内容接收器
  Progress progress;    // 进度
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  const SSL *ssl = nullptr;  // SSL 支持
#endif

  // 检查是否存在指定的头部信息
  bool has_header(const std::string &key) const;
  // 获取指定头部信息的值
  std::string get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定头部信息的值，并转换为指定类型
  template <typename T>
  T get_header_value(const std::string &key, size_t id = 0) const;
// 获取指定键对应的头部值的数量
size_t get_header_value_count(const std::string &key) const;

// 设置指定键对应的头部值
void set_header(const std::string &key, const std::string &val);

// 检查是否存在指定键的参数
bool has_param(const std::string &key) const;

// 获取指定键对应的参数值，可以指定参数的索引
std::string get_param_value(const std::string &key, size_t id = 0) const;

// 获取指定键对应的参数值的数量
size_t get_param_value_count(const std::string &key) const;

// 检查是否为多部分表单数据
bool is_multipart_form_data() const;

// 检查是否存在指定键的文件
bool has_file(const std::string &key) const;

// 获取指定键对应的文件值
MultipartFormData get_file_value(const std::string &key) const;

// 获取指定键对应的文件值的向量
std::vector<MultipartFormData> get_file_values(const std::string &key) const;

// 私有成员...
size_t redirect_count_ = CPPHTTPLIB_REDIRECT_MAX_COUNT; // 重定向次数，默认为 CPPHTTPLIB_REDIRECT_MAX_COUNT
size_t content_length_ = 0; // 内容长度，默认为 0
ContentProvider content_provider_; // 内容提供者
bool is_chunked_content_provider_ = false; // 是否为分块内容提供者，默认为 false
size_t authorization_count_ = 0; // 授权次数，默认为 0
// 定义一个名为 Response 的结构体，包含版本、状态、原因、头部、正文和重定向地址等成员变量
struct Response {
  std::string version; // HTTP 版本
  int status = -1; // 状态码，默认为 -1
  std::string reason; // 状态原因
  Headers headers; // 头部信息
  std::string body; // 正文
  std::string location; // 重定向地址

  // 检查是否存在指定的头部信息
  bool has_header(const std::string &key) const;
  // 获取指定头部信息的值
  std::string get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定头部信息的值，并转换为指定类型
  template <typename T>
  T get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定头部信息值的数量
  size_t get_header_value_count(const std::string &key) const;
  // 设置指定头部信息的值
  void set_header(const std::string &key, const std::string &val);

  // 设置重定向地址和状态码
  void set_redirect(const std::string &url, int status = 302);
  // 设置正文内容和内容类型
  void set_content(const char *s, size_t n, const std::string &content_type);
  // 设置正文内容和内容类型
  void set_content(const std::string &s, const std::string &content_type);
  // 设置内容提供者，包括长度、内容类型、提供者和资源释放器
  void set_content_provider(
      size_t length, const std::string &content_type, ContentProvider provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 设置内容提供者，包括内容类型、提供者和资源释放器
  void set_content_provider(
      const std::string &content_type, ContentProviderWithoutLength provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 设置分块内容提供者，包括内容类型、提供者和资源释放器
  void set_chunked_content_provider(
      const std::string &content_type, ContentProviderWithoutLength provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 默认构造函数
  Response() = default;
  // 拷贝构造函数
  Response(const Response &) = default;
  // 拷贝赋值运算符
  Response &operator=(const Response &) = default;
  // 移动构造函数
  Response(Response &&) = default;
  // 移动赋值运算符
  Response &operator=(Response &&) = default;
  // 析构函数
  ~Response() {
    // 如果存在资源释放器，则释放资源
    if (content_provider_resource_releaser_) {
      content_provider_resource_releaser_(content_provider_success_);
    }
  }
// 类的结束标记
    }
  }

  // 私有成员...
  size_t content_length_ = 0; // 内容长度，默认为0
  ContentProvider content_provider_; // 内容提供者
  ContentProviderResourceReleaser content_provider_resource_releaser_; // 内容提供者资源释放器
  bool is_chunked_content_provider_ = false; // 是否为分块内容提供者，默认为false
  bool content_provider_success_ = false; // 内容提供者是否成功

};

class Stream {
public:
  virtual ~Stream() = default; // 虚析构函数

  virtual bool is_readable() const = 0; // 虚函数，判断是否可读
  virtual bool is_writable() const = 0; // 虚函数，判断是否可写

  virtual ssize_t read(char *ptr, size_t size) = 0; // 虚函数，读取数据
  virtual ssize_t write(const char *ptr, size_t size) = 0; // 虚函数，写入数据
// 获取远程IP地址和端口号的抽象方法
virtual void get_remote_ip_and_port(std::string &ip, int &port) const = 0;
// 获取本地IP地址和端口号的抽象方法
virtual void get_local_ip_and_port(std::string &ip, int &port) const = 0;
// 返回套接字的抽象方法
virtual socket_t socket() const = 0;

// 格式化写入数据的模板方法
template <typename... Args>
ssize_t write_format(const char *fmt, const Args &...args);
// 写入字符数组的方法
ssize_t write(const char *ptr);
// 写入字符串的方法
ssize_t write(const std::string &s);
};

// 任务队列类
class TaskQueue {
public:
  // 默认构造函数
  TaskQueue() = default;
  // 虚析构函数
  virtual ~TaskQueue() = default;

  // 将任务函数加入队列的抽象方法
  virtual void enqueue(std::function<void()> fn) = 0;
  // 关闭任务队列的抽象方法
  virtual void shutdown() = 0;

  // 空闲时执行的方法
  virtual void on_idle() {}
};
# 定义一个线程池类，继承自任务队列类
class ThreadPool : public TaskQueue {
public:
  # 构造函数，初始化线程池并设置关闭标志为false
  explicit ThreadPool(size_t n) : shutdown_(false) {
    # 创建n个工作线程，并加入线程池
    while (n) {
      threads_.emplace_back(worker(*this));
      n--;
    }
  }

  # 禁用拷贝构造函数
  ThreadPool(const ThreadPool &) = delete;
  # 虚析构函数，用于释放资源
  ~ThreadPool() override = default;

  # 将任务加入任务队列
  void enqueue(std::function<void()> fn) override {
    {
      # 加锁，保护任务队列
      std::unique_lock<std::mutex> lock(mutex_);
      # 将任务加入任务队列
      jobs_.push_back(std::move(fn));
    }

    # 通知一个等待中的线程去执行任务
    cond_.notify_one();
```

  }

  // 重写父类的关闭方法
  void shutdown() override {
    // 停止所有工作线程...
    {
      // 获取互斥锁
      std::unique_lock<std::mutex> lock(mutex_);
      // 设置关闭标志为true
      shutdown_ = true;
    }

    // 通知所有等待的线程
    cond_.notify_all();

    // 等待所有工作线程结束
    for (auto &t : threads_) {
      t.join();
    }
  }

private:
  // 定义内部结构体worker
  struct worker {
    // 显式构造函数，传入线程池的引用
    explicit worker(ThreadPool &pool) : pool_(pool) {}
# 重载操作符()，用于线程池中的线程执行任务
void operator()() {
  # 无限循环，等待并执行任务
  for (;;) {
    # 定义一个函数对象
    std::function<void()> fn;
    {
      # 创建互斥锁，保护线程池的共享资源
      std::unique_lock<std::mutex> lock(pool_.mutex_);
      # 等待条件变量，直到有任务或线程池关闭
      pool_.cond_.wait(
          lock, [&] { return !pool_.jobs_.empty() || pool_.shutdown_; });

      # 如果线程池关闭且没有任务，则退出循环
      if (pool_.shutdown_ && pool_.jobs_.empty()) { break; }

      # 获取并移除队列中的第一个任务
      fn = std::move(pool_.jobs_.front());
      pool_.jobs_.pop_front();
    }

    # 断言任务是否有效
    assert(true == static_cast<bool>(fn));
    # 执行任务
    fn();
  }
}
// 声明一个 ThreadPool 类，包含一个线程池和相关的方法
class ThreadPool {
public:
  // 声明一个 worker 结构体，用于执行线程池中的任务
  struct worker {
    ThreadPool *pool_;
    void operator()();
  };
  friend struct worker;

  // 声明一个线程向量，用于存储线程池中的线程
  std::vector<std::thread> threads_;
  // 声明一个任务列表，用于存储线程池中的任务
  std::list<std::function<void()>> jobs_;

  // 声明一个布尔值，表示线程池是否关闭
  bool shutdown_;

  // 声明一个条件变量，用于线程同步
  std::condition_variable cond_;
  // 声明一个互斥锁，用于线程同步
  std::mutex mutex_;
};

// 声明一个 Logger 类型，用于记录请求和响应的日志
using Logger = std::function<void(const Request &, const Response &)>;

// 声明一个 SocketOptions 类型，用于设置套接字选项
using SocketOptions = std::function<void(socket_t sock)>;

// 声明一个默认的套接字选项设置方法
void default_socket_options(socket_t sock);
# 定义一个名为 Server 的类
class Server {
public:
  # 定义一个名为 Handler 的函数类型，用于处理请求和响应
  using Handler = std::function<void(const Request &, Response &)>;

  # 定义一个名为 ExceptionHandler 的函数类型，用于处理异常情况
  using ExceptionHandler =
      std::function<void(const Request &, Response &, std::exception_ptr ep)>;

  # 定义一个名为 HandlerResponse 的枚举类型，表示处理器的响应状态
  enum class HandlerResponse {
    Handled,  # 已处理
    Unhandled,  # 未处理
  };
  # 定义一个名为 HandlerWithResponse 的函数类型，用于处理请求和响应，并返回处理器的响应状态
  using HandlerWithResponse =
      std::function<HandlerResponse(const Request &, Response &)>;

  # 定义一个名为 HandlerWithContentReader 的函数类型，用于处理请求和响应，并接受内容读取器作为参数
  using HandlerWithContentReader = std::function<void(
      const Request &, Response &, const ContentReader &content_reader)>;

  # 定义一个名为 Expect100ContinueHandler 的函数类型，用于处理 100 Continue 请求
  using Expect100ContinueHandler =
      std::function<int(const Request &, Response &)>;
  // 默认构造函数，用于创建 Server 对象
  Server();

  // 虚析构函数，用于释放 Server 对象
  virtual ~Server();

  // 判断 Server 对象是否有效
  virtual bool is_valid() const;

  // 添加 GET 请求处理器
  Server &Get(const std::string &pattern, Handler handler);

  // 添加 POST 请求处理器
  Server &Post(const std::string &pattern, Handler handler);

  // 添加带内容读取的 POST 请求处理器
  Server &Post(const std::string &pattern, HandlerWithContentReader handler);

  // 添加 PUT 请求处理器
  Server &Put(const std::string &pattern, Handler handler);

  // 添加带内容读取的 PUT 请求处理器
  Server &Put(const std::string &pattern, HandlerWithContentReader handler);

  // 添加 PATCH 请求处理器
  Server &Patch(const std::string &pattern, Handler handler);

  // 添加带内容读取的 PATCH 请求处理器
  Server &Patch(const std::string &pattern, HandlerWithContentReader handler);

  // 添加 DELETE 请求处理器
  Server &Delete(const std::string &pattern, Handler handler);

  // 添加带内容读取的 DELETE 请求处理器
  Server &Delete(const std::string &pattern, HandlerWithContentReader handler);

  // 添加 OPTIONS 请求处理器
  Server &Options(const std::string &pattern, Handler handler);

  // 设置服务器的基本目录和挂载点
  bool set_base_dir(const std::string &dir, const std::string &mount_point = std::string());

  // 设置挂载点和目录
  bool set_mount_point(const std::string &mount_point, const std::string &dir);
  // 设置服务器的默认头部信息
  Server &set_default_headers(Headers headers);

  // 移除指定挂载点
  bool remove_mount_point(const std::string &mount_point);

  // 设置文件扩展名和 MIME 类型的映射关系
  Server &set_file_extension_and_mimetype_mapping(const std::string &ext,
                                                  const std::string &mime);

  // 设置文件请求处理程序
  Server &set_file_request_handler(Handler handler);

  // 设置错误处理程序（带响应）
  Server &set_error_handler(HandlerWithResponse handler);

  // 设置错误处理程序
  Server &set_error_handler(Handler handler);

  // 设置异常处理程序
  Server &set_exception_handler(ExceptionHandler handler);

  // 设置预路由处理程序（带响应）
  Server &set_pre_routing_handler(HandlerWithResponse handler);

  // 设置后路由处理程序
  Server &set_post_routing_handler(Handler handler);

  // 设置期望 100 继续处理程序
  Server &set_expect_100_continue_handler(Expect100ContinueHandler handler);

  // 设置日志记录器
  Server &set_logger(Logger logger);

  // 设置地址族
  Server &set_address_family(int family);

  // 设置 TCP 禁用延迟
  Server &set_tcp_nodelay(bool on);

  // 设置套接字选项
  Server &set_socket_options(SocketOptions socket_options);
// 设置服务器保持活动连接的最大数量
Server &set_keep_alive_max_count(size_t count);

// 设置服务器保持活动连接的超时时间
Server &set_keep_alive_timeout(time_t sec);

// 设置服务器读取数据的超时时间
Server &set_read_timeout(time_t sec, time_t usec = 0);
// 设置服务器读取数据的超时时间，使用 std::chrono::duration 表示时间
template <class Rep, class Period>
Server &set_read_timeout(const std::chrono::duration<Rep, Period> &duration);

// 设置服务器写入数据的超时时间
Server &set_write_timeout(time_t sec, time_t usec = 0);
// 设置服务器写入数据的超时时间，使用 std::chrono::duration 表示时间
template <class Rep, class Period>
Server &set_write_timeout(const std::chrono::duration<Rep, Period> &duration);

// 设置服务器空闲间隔的超时时间
Server &set_idle_interval(time_t sec, time_t usec = 0);
// 设置服务器空闲间隔的超时时间，使用 std::chrono::duration 表示时间
template <class Rep, class Period>
Server &set_idle_interval(const std::chrono::duration<Rep, Period> &duration);

// 设置服务器接收数据的最大长度
Server &set_payload_max_length(size_t length);

// 将服务器绑定到指定主机和端口
bool bind_to_port(const std::string &host, int port, int socket_flags = 0);
// 将服务器绑定到任意可用端口
int bind_to_any_port(const std::string &host, int socket_flags = 0);
  // 在绑定后监听连接
  bool listen_after_bind();

  // 监听指定主机和端口的连接，可以设置套接字标志
  bool listen(const std::string &host, int port, int socket_flags = 0);

  // 检查服务器是否正在运行
  bool is_running() const;

  // 等待服务器准备就绪
  void wait_until_ready() const;

  // 停止服务器
  void stop();

  // 处理请求的方法，接受一个流对象、是否关闭连接、连接是否关闭、设置请求的函数
  bool process_request(Stream &strm, bool close_connection,
                       bool &connection_closed,
                       const std::function<void(Request &)> &setup_request);

  // 服务器套接字
  std::atomic<socket_t> svr_sock_{INVALID_SOCKET};

  // 最大保持活动连接数
  size_t keep_alive_max_count_ = CPPHTTPLIB_KEEPALIVE_MAX_COUNT;

  // 保持活动连接的超时时间（秒）
  time_t keep_alive_timeout_sec_ = CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND;

  // 读取超时时间（秒）
  time_t read_timeout_sec_ = CPPHTTPLIB_READ_TIMEOUT_SECOND;

  // 读取超时时间（微秒）
  time_t read_timeout_usec_ = CPPHTTPLIB_READ_TIMEOUT_USECOND;
  // 设置写超时时间（秒）
  time_t write_timeout_sec_ = CPPHTTPLIB_WRITE_TIMEOUT_SECOND;
  // 设置写超时时间（微秒）
  time_t write_timeout_usec_ = CPPHTTPLIB_WRITE_TIMEOUT_USECOND;
  // 设置空闲间隔时间（秒）
  time_t idle_interval_sec_ = CPPHTTPLIB_IDLE_INTERVAL_SECOND;
  // 设置空闲间隔时间（微秒）
  time_t idle_interval_usec_ = CPPHTTPLIB_IDLE_INTERVAL_USECOND;
  // 设置有效载荷最大长度
  size_t payload_max_length_ = CPPHTTPLIB_PAYLOAD_MAX_LENGTH;

private:
  // 定义处理程序的正则表达式和处理程序的对应关系
  using Handlers = std::vector<std::pair<std::regex, Handler>>;
  // 定义带内容读取器的处理程序的正则表达式和处理程序的对应关系
  using HandlersForContentReader =
      std::vector<std::pair<std::regex, HandlerWithContentReader>>;

  // 创建服务器套接字
  socket_t create_server_socket(const std::string &host, int port,
                                int socket_flags,
                                SocketOptions socket_options) const;
  // 绑定内部套接字
  int bind_internal(const std::string &host, int port, int socket_flags);
  // 监听内部套接字
  bool listen_internal();

  // 路由请求
  bool routing(Request &req, Response &res, Stream &strm);
  // 处理文件请求
  bool handle_file_request(const Request &req, Response &res,
                           bool head = false);
# 调度请求，根据请求和处理程序执行相应的操作
bool dispatch_request(Request &req, Response &res, const Handlers &handlers);

# 为内容阅读器调度请求，根据请求和内容阅读器执行相应的操作
bool dispatch_request_for_content_reader(Request &req, Response &res,
                                      ContentReader content_reader,
                                      const HandlersForContentReader &handlers);

# 解析请求行，将请求行字符串解析为请求对象
bool parse_request_line(const char *s, Request &req);

# 应用范围，根据请求和响应对象，内容类型和边界应用范围
void apply_ranges(const Request &req, Response &res,
                    std::string &content_type, std::string &boundary);

# 写入响应，将响应对象写入流中
bool write_response(Stream &strm, bool close_connection, const Request &req,
                      Response &res);

# 带内容写入响应，将响应对象及其内容写入流中
bool write_response_with_content(Stream &strm, bool close_connection,
                                   const Request &req, Response &res);

# 写入响应核心，根据需要应用范围，将响应对象写入流中
bool write_response_core(Stream &strm, bool close_connection,
                           const Request &req, Response &res,
                           bool need_apply_ranges);

# 通过提供者写入内容，根据请求和响应对象，边界和内容类型将内容写入流中
bool write_content_with_provider(Stream &strm, const Request &req,
                                   Response &res, const std::string &boundary,
                                   const std::string &content_type);

# 读取内容，从流中读取内容并填充请求和响应对象
bool read_content(Stream &strm, Request &req, Response &res);
# 读取内容并使用内容接收器处理，返回布尔值表示是否成功
bool read_content_with_content_receiver(Stream &strm, Request &req, Response &res,
                                     ContentReceiver receiver,
                                     MultipartContentHeader multipart_header,
                                     ContentReceiver multipart_receiver);

# 读取内容的核心功能，返回布尔值表示是否成功
bool read_content_core(Stream &strm, Request &req, Response &res,
                         ContentReceiver receiver,
                         MultipartContentHeader multipart_header,
                         ContentReceiver multipart_receiver);

# 处理并关闭套接字，返回布尔值表示是否成功
virtual bool process_and_close_socket(socket_t sock);

# 挂载点条目结构，包含挂载点、基本目录和头部信息
struct MountPointEntry {
    std::string mount_point;
    std::string base_dir;
    Headers headers;
};
# 基本目录的向量
std::vector<MountPointEntry> base_dirs_;

# 表示服务器是否正在运行的原子布尔值
std::atomic<bool> is_running_{false};
// 声明一个原子布尔类型的变量，用于表示任务是否完成
std::atomic<bool> done_{false};

// 声明一个映射，用于存储文件扩展名和对应的 MIME 类型
std::map<std::string, std::string> file_extension_and_mimetype_map_;

// 声明一个处理文件请求的处理程序
Handler file_request_handler_;

// 声明用于处理 GET 请求的处理程序集合
Handlers get_handlers_;

// 声明用于处理 POST 请求的处理程序集合
Handlers post_handlers_;

// 声明用于处理 POST 请求的内容读取器的处理程序集合
HandlersForContentReader post_handlers_for_content_reader_;

// 声明用于处理 PUT 请求的处理程序集合
Handlers put_handlers_;

// 声明用于处理 PUT 请求的内容读取器的处理程序集合
HandlersForContentReader put_handlers_for_content_reader_;

// 声明用于处理 PATCH 请求的处理程序集合
Handlers patch_handlers_;

// 声明用于处理 PATCH 请求的内容读取器的处理程序集合
HandlersForContentReader patch_handlers_for_content_reader_;

// 声明用于处理 DELETE 请求的处理程序集合
Handlers delete_handlers_;

// 声明用于处理 DELETE 请求的内容读取器的处理程序集合
HandlersForContentReader delete_handlers_for_content_reader_;

// 声明用于处理 OPTIONS 请求的处理程序集合
Handlers options_handlers_;

// 声明一个带有响应的错误处理程序
HandlerWithResponse error_handler_;

// 声明一个异常处理程序
ExceptionHandler exception_handler_;

// 声明一个带有响应的预路由处理程序
HandlerWithResponse pre_routing_handler_;

// 声明一个后路由处理程序
Handler post_routing_handler_;

// 声明一个日志记录器
Logger logger_;

// 声明一个处理 100 Continue 期望的处理程序
Expect100ContinueHandler expect_100_continue_handler_;
  // 设置地址族为未指定
  int address_family_ = AF_UNSPEC;
  // 设置是否启用 TCP 禁用算法
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  // 设置套接字选项为默认选项
  SocketOptions socket_options_ = default_socket_options;

  // 默认的请求头
  Headers default_headers_;
};

// 错误类型枚举
enum class Error {
  Success = 0,  // 成功
  Unknown,  // 未知错误
  Connection,  // 连接错误
  BindIPAddress,  // 绑定 IP 地址错误
  Read,  // 读取错误
  Write,  // 写入错误
  ExceedRedirectCount,  // 超过重定向次数
  Canceled,  // 取消
  SSLConnection,  // SSL 连接错误
  SSLLoadingCerts,  // SSL 加载证书错误
  SSLServerVerification,  // SSL 服务器验证错误
  UnsupportedMultipartBoundaryChars,  // 不支持的多部分边界字符
  Compression,  // 压缩
  ConnectionTimeout,  // 连接超时

  // 仅供内部使用
  SSLPeerCouldBeClosed_,  // SSL 对等方可能已关闭

};

std::string to_string(const Error error);  // 将错误转换为字符串

std::ostream &operator<<(std::ostream &os, const Error &obj);  // 重载输出流操作符，用于输出错误对象

class Result {
public:
  Result(std::unique_ptr<Response> &&res, Error err,
         Headers &&request_headers = Headers{})
      : res_(std::move(res)), err_(err),
        request_headers_(std::move(request_headers)) {}  // 构造函数，初始化响应、错误和请求头

  // 响应
  operator bool() const { return res_ != nullptr; }  // 转换为布尔值，判断是否有响应
  bool operator==(std::nullptr_t) const { return res_ == nullptr; }  // 重载等于操作符，判断是否为空指针
  // 重载不等于操作符，用于判断指针是否为空
  bool operator!=(std::nullptr_t) const { return res_ != nullptr; }
  // 返回常量引用，获取响应对象的值
  const Response &value() const { return *res_; }
  // 返回引用，获取响应对象的值
  Response &value() { return *res_; }
  // 重载解引用操作符，返回常量引用，获取响应对象的值
  const Response &operator*() const { return *res_; }
  // 重载解引用操作符，返回引用，获取响应对象的值
  Response &operator*() { return *res_; }
  // 重载箭头操作符，返回常量指针，获取响应对象的指针
  const Response *operator->() const { return res_.get(); }
  // 重载箭头操作符，返回指针，获取响应对象的指针
  Response *operator->() { return res_.get(); }

  // 返回错误对象
  Error error() const { return err_; }

  // 检查是否存在指定的请求头
  bool has_request_header(const std::string &key) const;
  // 获取指定请求头的值
  std::string get_request_header_value(const std::string &key,
                                       size_t id = 0) const;
  // 获取指定请求头的值，并转换为指定类型
  template <typename T>
  T get_request_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定请求头的值的数量
  size_t get_request_header_value_count(const std::string &key) const;

private:
  // 声明一个指向 Response 对象的独占指针
  std::unique_ptr<Response> res_;
  // 声明一个 Error 对象
  Error err_;
  // 声明一个 Headers 对象用于存储请求头信息
  Headers request_headers_;
};

// 定义 ClientImpl 类
class ClientImpl {
public:
  // 构造函数，接受主机名参数
  explicit ClientImpl(const std::string &host);

  // 构造函数，接受主机名和端口参数
  explicit ClientImpl(const std::string &host, int port);

  // 构造函数，接受主机名、端口、客户端证书路径和客户端密钥路径参数
  explicit ClientImpl(const std::string &host, int port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path);

  // 虚析构函数
  virtual ~ClientImpl();

  // 虚函数，用于检查客户端是否有效
  virtual bool is_valid() const;

  // 发起 GET 请求的方法，接受路径参数
  Result Get(const std::string &path);
# 定义了多个重载的 Get 方法，用于发送 HTTP GET 请求并获取响应结果
# 第一个重载方法，接收路径和请求头，返回响应结果
Result Get(const std::string &path, const Headers &headers);

# 第二个重载方法，接收路径和进度回调函数，返回响应结果
Result Get(const std::string &path, Progress progress);

# 第三个重载方法，接收路径、请求头和进度回调函数，返回响应结果
Result Get(const std::string &path, const Headers &headers, Progress progress);

# 第四个重载方法，接收路径和内容接收器，返回响应结果
Result Get(const std::string &path, ContentReceiver content_receiver);

# 第五个重载方法，接收路径、请求头和内容接收器，返回响应结果
Result Get(const std::string &path, const Headers &headers, ContentReceiver content_receiver);

# 第六个重载方法，接收路径、内容接收器和进度回调函数，返回响应结果
Result Get(const std::string &path, ContentReceiver content_receiver, Progress progress);

# 第七个重载方法，接收路径、请求头、内容接收器和进度回调函数，返回响应结果
Result Get(const std::string &path, const Headers &headers, ContentReceiver content_receiver, Progress progress);

# 第八个重载方法，接收路径、响应处理器和内容接收器，返回响应结果
Result Get(const std::string &path, ResponseHandler response_handler, ContentReceiver content_receiver);

# 第九个重载方法，接收路径、请求头、响应处理器和内容接收器，返回响应结果
Result Get(const std::string &path, const Headers &headers, ResponseHandler response_handler, ContentReceiver content_receiver);

# 第十个重载方法，接收路径、响应处理器、内容接收器和进度回调函数，返回响应结果
Result Get(const std::string &path, ResponseHandler response_handler, ContentReceiver content_receiver, Progress progress);

# 第十一个重载方法，接收路径、请求头、响应处理器、内容接收器和进度回调函数，返回响应结果
Result Get(const std::string &path, const Headers &headers, ResponseHandler response_handler, ContentReceiver content_receiver, Progress progress);
// 通过 GET 方法获取指定路径的资源，可以传入参数、请求头和进度回调函数
Result Get(const std::string &path, const Params &params,
           const Headers &headers, Progress progress = nullptr);

// 通过 GET 方法获取指定路径的资源，可以传入参数、请求头、内容接收器和进度回调函数
Result Get(const std::string &path, const Params &params,
           const Headers &headers, ContentReceiver content_receiver,
           Progress progress = nullptr);

// 通过 GET 方法获取指定路径的资源，可以传入参数、请求头、响应处理器、内容接收器和进度回调函数
Result Get(const std::string &path, const Params &params,
           const Headers &headers, ResponseHandler response_handler,
           ContentReceiver content_receiver, Progress progress = nullptr);

// 通过 HEAD 方法获取指定路径的资源的头部信息
Result Head(const std::string &path);

// 通过 HEAD 方法获取指定路径的资源的头部信息，可以传入请求头
Result Head(const std::string &path, const Headers &headers);

// 通过 POST 方法向指定路径发送请求
Result Post(const std::string &path);

// 通过 POST 方法向指定路径发送请求，可以传入请求头
Result Post(const std::string &path, const Headers &headers);

// 通过 POST 方法向指定路径发送请求，可以传入请求体、内容长度和内容类型
Result Post(const std::string &path, const char *body, size_t content_length,
          const std::string &content_type);

// 通过 POST 方法向指定路径发送请求，可以传入请求头、请求体、内容长度和内容类型
Result Post(const std::string &path, const Headers &headers, const char *body,
          size_t content_length, const std::string &content_type);
// 发送一个 POST 请求，包含路径和请求体，返回请求结果
Result Post(const std::string &path, const std::string &body,
            const std::string &content_type);

// 发送一个 POST 请求，包含路径、请求头、请求体和内容类型，返回请求结果
Result Post(const std::string &path, const Headers &headers,
            const std::string &body, const std::string &content_type);

// 发送一个 POST 请求，包含路径、内容长度、内容提供者和内容类型，返回请求结果
Result Post(const std::string &path, size_t content_length,
            ContentProvider content_provider,
            const std::string &content_type);

// 发送一个 POST 请求，包含路径、无长度的内容提供者和内容类型，返回请求结果
Result Post(const std::string &path,
            ContentProviderWithoutLength content_provider,
            const std::string &content_type);

// 发送一个 POST 请求，包含路径、请求头、内容长度、内容提供者和内容类型，返回请求结果
Result Post(const std::string &path, const Headers &headers,
            size_t content_length, ContentProvider content_provider,
            const std::string &content_type);

// 发送一个 POST 请求，包含路径、请求头、无长度的内容提供者和内容类型，返回请求结果
Result Post(const std::string &path, const Headers &headers,
            ContentProviderWithoutLength content_provider,
            const std::string &content_type);

// 发送一个 POST 请求，包含路径和参数，返回请求结果
Result Post(const std::string &path, const Params &params);

// 发送一个 POST 请求，包含路径、请求头和参数，返回请求结果
Result Post(const std::string &path, const Headers &headers,
            const Params &params);

// 发送一个 POST 请求，包含路径和多部分表单数据项，返回请求结果
Result Post(const std::string &path, const MultipartFormDataItems &items);
  // 发送带有多部分表单数据的 POST 请求，不带自定义边界
  Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items);
  // 发送带有多部分表单数据的 POST 请求，带有自定义边界
  Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items, const std::string &boundary);
  // 发送带有多部分表单数据和提供者数据的 POST 请求
  Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items,
              const MultipartFormDataProviderItems &provider_items);

  // 发送不带数据的 PUT 请求
  Result Put(const std::string &path);
  // 发送带有字符数组数据的 PUT 请求
  Result Put(const std::string &path, const char *body, size_t content_length,
             const std::string &content_type);
  // 发送带有字符数组数据和自定义头部的 PUT 请求
  Result Put(const std::string &path, const Headers &headers, const char *body,
             size_t content_length, const std::string &content_type);
  // 发送带有字符串数据的 PUT 请求
  Result Put(const std::string &path, const std::string &body,
             const std::string &content_type);
  // 发送带有字符串数据和自定义头部的 PUT 请求
  Result Put(const std::string &path, const Headers &headers,
             const std::string &body, const std::string &content_type);
  // 发送带有内容提供者和自定义头部的 PUT 请求
  Result Put(const std::string &path, size_t content_length,
             ContentProvider content_provider, const std::string &content_type);
  // 发送带有路径参数的 PUT 请求
  Result Put(const std::string &path,
# 使用给定的内容提供程序和内容类型将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             ContentProviderWithoutLength content_provider,
             const std::string &content_type);

# 使用给定的内容提供程序和内容类型将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             size_t content_length, ContentProvider content_provider,
             const std::string &content_type);

# 使用给定的参数将数据放入指定路径，返回结果
Result Put(const std::string &path, const Params &params);

# 使用给定的参数和头部信息将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             const Params &params);

# 使用给定的多部分表单数据项将数据放入指定路径，返回结果
Result Put(const std::string &path, const MultipartFormDataItems &items);

# 使用给定的多部分表单数据项和头部信息将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items);

# 使用给定的多部分表单数据项、头部信息和边界将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items, const std::string &boundary);

# 使用给定的多部分表单数据项、头部信息、提供者项将数据放入指定路径，返回结果
Result Put(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items,
             const MultipartFormDataProviderItems &provider_items);
  // 用于对指定路径进行补丁操作，不包含请求体和请求头
  Result Patch(const std::string &path);

  // 用于对指定路径进行补丁操作，包含请求体和请求头
  Result Patch(const std::string &path, const char *body, size_t content_length,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求体、请求头和自定义的请求头
  Result Patch(const std::string &path, const Headers &headers,
               const char *body, size_t content_length,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求体和内容类型
  Result Patch(const std::string &path, const std::string &body,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求体、请求头和内容类型
  Result Patch(const std::string &path, const Headers &headers,
               const std::string &body, const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求体长度、内容提供者和内容类型
  Result Patch(const std::string &path, size_t content_length,
               ContentProvider content_provider,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含内容提供者和内容类型，但不包含请求体长度
  Result Patch(const std::string &path,
               ContentProviderWithoutLength content_provider,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求头、请求体长度、内容提供者和内容类型
  Result Patch(const std::string &path, const Headers &headers,
               size_t content_length, ContentProvider content_provider,
               const std::string &content_type);

  // 用于对指定路径进行补丁操作，包含请求头、请求体长度和内容提供者，但不包含内容类型
  Result Patch(const std::string &path, const Headers &headers,
// 使用给定的内容提供者和内容类型发送请求
Result Send(const std::string &path, ContentProviderWithoutLength content_provider, const std::string &content_type);

// 删除指定路径的资源
Result Delete(const std::string &path);

// 删除指定路径的资源，并附带请求头
Result Delete(const std::string &path, const Headers &headers);

// 删除指定路径的资源，并附带请求体和内容长度
Result Delete(const std::string &path, const char *body, size_t content_length, const std::string &content_type);

// 删除指定路径的资源，并附带请求头、请求体和内容长度
Result Delete(const std::string &path, const Headers &headers, const char *body, size_t content_length, const std::string &content_type);

// 删除指定路径的资源，并附带请求体和内容类型
Result Delete(const std::string &path, const std::string &body, const std::string &content_type);

// 删除指定路径的资源，并附带请求头、请求体和内容类型
Result Delete(const std::string &path, const Headers &headers, const std::string &body, const std::string &content_type);

// 发送 OPTIONS 请求到指定路径
Result Options(const std::string &path);

// 发送 OPTIONS 请求到指定路径，并附带请求头
Result Options(const std::string &path, const Headers &headers);

// 发送请求并获取响应
bool send(Request &req, Response &res, Error &error);

// 发送请求并获取响应结果
Result send(const Request &req);
// 检查套接字是否打开，返回值为 size_t 类型
size_t is_socket_open() const;

// 返回套接字对象
socket_t socket() const;

// 停止套接字的操作
void stop();

// 设置主机名到地址的映射关系
void set_hostname_addr_map(std::map<std::string, std::string> addr_map);

// 设置默认的请求头
void set_default_headers(Headers headers);

// 设置地址族
void set_address_family(int family);

// 设置 TCP 的无延迟选项
void set_tcp_nodelay(bool on);

// 设置套接字选项
void set_socket_options(SocketOptions socket_options);

// 设置连接超时时间，单位为秒和微秒
void set_connection_timeout(time_t sec, time_t usec = 0);

// 设置连接超时时间，使用 std::chrono::duration 类型
template <class Rep, class Period>
void set_connection_timeout(const std::chrono::duration<Rep, Period> &duration);
  // 设置读取超时时间，单位为秒和微秒
  void set_read_timeout(time_t sec, time_t usec = 0);
  // 设置读取超时时间，使用 std::chrono::duration 表示时间间隔
  template <class Rep, class Period>
  void set_read_timeout(const std::chrono::duration<Rep, Period> &duration);

  // 设置写入超时时间，单位为秒和微秒
  void set_write_timeout(time_t sec, time_t usec = 0);
  // 设置写入超时时间，使用 std::chrono::duration 表示时间间隔
  template <class Rep, class Period>
  void set_write_timeout(const std::chrono::duration<Rep, Period> &duration);

  // 设置基本认证的用户名和密码
  void set_basic_auth(const std::string &username, const std::string &password);
  // 设置 Bearer Token 认证的 token
  void set_bearer_token_auth(const std::string &token);
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要认证的用户名和密码，仅在支持 OpenSSL 时可用
  void set_digest_auth(const std::string &username,
                       const std::string &password);
#endif

  // 设置是否保持连接
  void set_keep_alive(bool on);
  // 设置是否跟随重定向
  void set_follow_location(bool on);

  // 设置是否进行 URL 编码
  void set_url_encode(bool on);
// 设置是否开启压缩功能
void set_compress(bool on);

// 设置是否开启解压缩功能
void set_decompress(bool on);

// 设置网络请求的接口
void set_interface(const std::string &intf);

// 设置代理服务器的主机和端口
void set_proxy(const std::string &host, int port);

// 设置代理服务器的基本认证用户名和密码
void set_proxy_basic_auth(const std::string &username, const std::string &password);

// 设置代理服务器的 Bearer Token 认证
void set_proxy_bearer_token_auth(const std::string &token);

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理服务器的摘要认证用户名和密码
void set_proxy_digest_auth(const std::string &username, const std::string &password);
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书文件路径和目录路径
void set_ca_cert_path(const std::string &ca_cert_file_path, const std::string &ca_cert_dir_path = std::string());

// 设置 CA 证书存储
void set_ca_cert_store(X509_STORE *ca_cert_store);
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则启用服务器证书验证
  void enable_server_certificate_verification(bool enabled);
#endif

  // 设置日志记录器
  void set_logger(Logger logger);

protected:
  // 定义 Socket 结构体
  struct Socket {
    // 套接字
    socket_t sock = INVALID_SOCKET;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    // 如果支持 OpenSSL，则定义 SSL 对象
    SSL *ssl = nullptr;
#endif

    // 判断套接字是否打开
    bool is_open() const { return sock != INVALID_SOCKET; }
  };

  // 创建并连接套接字
  virtual bool create_and_connect_socket(Socket &socket, Error &error);

  // 所有的：
  // 关闭 SSL 连接
  // 关闭套接字
  // 关闭套接字时应该只在套接字互斥锁被锁定时调用。
  // 另外，关闭 SSL 和关闭套接字也不应该与另一个线程同时使用该套接字发送请求。
  // 通过套接字关闭 SSL 连接
  virtual void shutdown_ssl(Socket &socket, bool shutdown_gracefully);
  // 关闭套接字
  void shutdown_socket(Socket &socket);
  // 关闭套接字
  void close_socket(Socket &socket);

  // 处理请求
  bool process_request(Stream &strm, Request &req, Response &res,
                       bool close_connection, Error &error);

  // 使用提供程序写入内容
  bool write_content_with_provider(Stream &strm, const Request &req,
                                   Error &error);

  // 复制设置
  void copy_settings(const ClientImpl &rhs);

  // 套接字端点信息
  const std::string host_;
  // 声明一个常量端口号
  const int port_;
  // 声明一个常量主机和端口号的字符串

  const std::string host_and_port_;

  // 当前打开的套接字
  Socket socket_;
  // 可变的互斥锁，用于保护 socket_
  mutable std::mutex socket_mutex_;
  // 递归互斥锁，用于保护请求
  std::recursive_mutex request_mutex_;

  // 这些都受 socket_mutex 保护
  // 在传输中的套接字请求数量
  size_t socket_requests_in_flight_ = 0;
  // 发送请求的线程 ID
  std::thread::id socket_requests_are_from_thread_ = std::thread::id();
  // 当请求完成时，套接字是否应该关闭
  bool socket_should_be_closed_when_request_is_done_ = false;

  // 主机名-IP 地址映射
  std::map<std::string, std::string> addr_map_;

  // 默认头部
  Headers default_headers_;

  // 设置
// 客户端证书路径
std::string client_cert_path_;
// 客户端密钥路径
std::string client_key_path_;

// 连接超时时间（秒）
time_t connection_timeout_sec_ = CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND;
// 连接超时时间（微秒）
time_t connection_timeout_usec_ = CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND;
// 读取超时时间（秒）
time_t read_timeout_sec_ = CPPHTTPLIB_READ_TIMEOUT_SECOND;
// 读取超时时间（微秒）
time_t read_timeout_usec_ = CPPHTTPLIB_READ_TIMEOUT_USECOND;
// 写入超时时间（秒）
time_t write_timeout_sec_ = CPPHTTPLIB_WRITE_TIMEOUT_SECOND;
// 写入超时时间（微秒）
time_t write_timeout_usec_ = CPPHTTPLIB_WRITE_TIMEOUT_USECOND;

// 基本认证用户名
std::string basic_auth_username_;
// 基本认证密码
std::string basic_auth_password_;
// Bearer Token 认证令牌
std::string bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 摘要认证用户名（仅在 OpenSSL 支持时有效）
std::string digest_auth_username_;
// 摘要认证密码（仅在 OpenSSL 支持时有效）
std::string digest_auth_password_;
#endif

// 是否保持长连接
bool keep_alive_ = false;
// 是否跟随重定向
bool follow_location_ = false;
  # 设置是否对 URL 进行编码
  bool url_encode_ = true;

  # 设置地址族，默认为 AF_UNSPEC
  int address_family_ = AF_UNSPEC;
  # 设置是否启用 TCP_NODELAY，默认为 CPPHTTPLIB_TCP_NODELAY
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  # 设置套接字选项，默认为 nullptr
  SocketOptions socket_options_ = nullptr;

  # 设置是否启用压缩，默认为 false
  bool compress_ = false;
  # 设置是否启用解压缩，默认为 true
  bool decompress_ = true;

  # 设置网络接口
  std::string interface_;

  # 设置代理主机
  std::string proxy_host_;
  # 设置代理端口，默认为 -1
  int proxy_port_ = -1;

  # 设置代理基本认证用户名
  std::string proxy_basic_auth_username_;
  # 设置代理基本认证密码
  std::string proxy_basic_auth_password_;
  # 设置代理 Bearer Token 认证令牌
  std::string proxy_bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  # 设置代理摘要认证用户名
  std::string proxy_digest_auth_username_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 存储代理服务器的摘要认证密码
  std::string proxy_digest_auth_password_;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 存储 CA 证书文件路径
  std::string ca_cert_file_path_;
  // 存储 CA 证书目录路径
  std::string ca_cert_dir_path_;

  // 用于存储 CA 证书的 X509 存储对象
  X509_STORE *ca_cert_store_ = nullptr;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 用于指示是否进行服务器证书验证
  bool server_certificate_verification_ = true;
#endif

  // 日志记录器对象
  Logger logger_;

private:
  // 发送 HTTP 请求并接收响应的私有方法
  bool send_(Request &req, Response &res, Error &error);
  // 发送 HTTP 请求并接收响应的私有方法，使用移动语义
  Result send_(Request &&req);
  # 创建客户端套接字，返回错误信息
  socket_t create_client_socket(Error &error) const;
  # 读取响应行，填充请求和响应对象，返回布尔值
  bool read_response_line(Stream &strm, const Request &req, Response &res);
  # 写入请求，填充请求对象，返回布尔值
  bool write_request(Stream &strm, Request &req, bool close_connection,
                     Error &error);
  # 重定向请求，填充请求和响应对象，返回布尔值
  bool redirect(Request &req, Response &res, Error &error);
  # 处理请求，填充请求和响应对象，返回布尔值
  bool handle_request(Stream &strm, Request &req, Response &res,
                      bool close_connection, Error &error);
  # 发送请求并获取响应，返回响应对象的唯一指针
  std::unique_ptr<Response> send_with_content_provider(
      Request &req, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type, Error &error);
  # 发送请求并获取响应，返回结果对象
  Result send_with_content_provider(
      const std::string &method, const std::string &path,
      const Headers &headers, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type);
  # 获取多部分内容提供程序，返回内容提供程序对象
  ContentProviderWithoutLength get_multipart_content_provider(
      const std::string &boundary, const MultipartFormDataItems &items,
// 定义一个类，用于提供多部分表单数据的项目
class MultipartFormDataProviderItems &provider_items);

// 调整主机字符串，返回调整后的主机字符串
std::string adjust_host_string(const std::string &host) const;

// 处理套接字，通过回调函数处理套接字流
virtual bool process_socket(const Socket &socket,
                            std::function<bool(Stream &strm)> callback);

// 判断是否为 SSL 连接
virtual bool is_ssl() const;
};

// 客户端类
class Client {
public:
  // 通用接口，根据给定的 scheme_host_port 创建客户端
  explicit Client(const std::string &scheme_host_port);

  // 通用接口，根据给定的 scheme_host_port、客户端证书路径和客户端密钥路径创建客户端
  explicit Client(const std::string &scheme_host_port,
                  const std::string &client_cert_path,
                  const std::string &client_key_path);

  // 仅支持 HTTP 的接口，根据给定的主机和端口创建客户端
  explicit Client(const std::string &host, int port);
  // 客户端构造函数，接受主机名、端口号、客户端证书路径和客户端密钥路径作为参数
  explicit Client(const std::string &host, int port,
                  const std::string &client_cert_path,
                  const std::string &client_key_path);

  // 移动构造函数，使用默认实现
  Client(Client &&) = default;

  // 客户端析构函数
  ~Client();

  // 检查客户端是否有效
  bool is_valid() const;

  // 发起 GET 请求，返回结果
  Result Get(const std::string &path);
  // 发起带自定义头部的 GET 请求，返回结果
  Result Get(const std::string &path, const Headers &headers);
  // 发起带进度回调的 GET 请求，返回结果
  Result Get(const std::string &path, Progress progress);
  // 发起带自定义头部和进度回调的 GET 请求，返回结果
  Result Get(const std::string &path, const Headers &headers,
             Progress progress);
  // 发起带内容接收器的 GET 请求，返回结果
  Result Get(const std::string &path, ContentReceiver content_receiver);
  // 发起带自定义头部和内容接收器的 GET 请求，返回结果
  Result Get(const std::string &path, const Headers &headers,
             ContentReceiver content_receiver);
  // 发起带内容接收器和自定义头部的 GET 请求，返回结果
  Result Get(const std::string &path, ContentReceiver content_receiver,
             const Headers &headers);
  # 使用 GET 方法发送请求，获取指定路径的资源，不带自定义头部，不带进度回调
  Result Get(const std::string &path, Progress progress);
  # 使用 GET 方法发送请求，获取指定路径的资源，带自定义头部，不带进度回调
  Result Get(const std::string &path, const Headers &headers, ContentReceiver content_receiver, Progress progress);
  # 使用 GET 方法发送请求，获取指定路径的资源，不带自定义头部，带响应处理器和数据接收器
  Result Get(const std::string &path, ResponseHandler response_handler, ContentReceiver content_receiver);
  # 使用 GET 方法发送请求，获取指定路径的资源，带自定义头部，响应处理器和数据接收器
  Result Get(const std::string &path, const Headers &headers, ResponseHandler response_handler, ContentReceiver content_receiver);
  # 使用 GET 方法发送请求，获取指定路径的资源，带自定义头部，响应处理器，数据接收器和进度回调
  Result Get(const std::string &path, const Headers &headers, ResponseHandler response_handler, ContentReceiver content_receiver, Progress progress);
  # 使用 GET 方法发送请求，获取指定路径的资源，带响应处理器，数据接收器和进度回调
  Result Get(const std::string &path, ResponseHandler response_handler, ContentReceiver content_receiver, Progress progress);
  # 使用 GET 方法发送请求，获取指定路径的资源，带参数，自定义头部，可选的进度回调
  Result Get(const std::string &path, const Params &params, const Headers &headers, Progress progress = nullptr);
  # 使用 GET 方法发送请求，获取指定路径的资源，带参数，自定义头部，数据接收器，可选的进度回调
  Result Get(const std::string &path, const Params &params, const Headers &headers, ContentReceiver content_receiver, Progress progress = nullptr);
  # 使用 GET 方法发送请求，获取指定路径的资源，带参数，自定义头部，响应处理器，数据接收器和进度回调
// 发送 HTTP HEAD 请求，传入路径和响应处理函数
Result Head(const std::string &path);
// 发送 HTTP HEAD 请求，传入路径和自定义头部
Result Head(const std::string &path, const Headers &headers);

// 发送 HTTP POST 请求，传入路径
Result Post(const std::string &path);
// 发送 HTTP POST 请求，传入路径和自定义头部
Result Post(const std::string &path, const Headers &headers);
// 发送 HTTP POST 请求，传入路径、请求体、内容长度和内容类型
Result Post(const std::string &path, const char *body, size_t content_length, const std::string &content_type);
// 发送 HTTP POST 请求，传入路径、自定义头部、请求体、内容长度和内容类型
Result Post(const std::string &path, const Headers &headers, const char *body, size_t content_length, const std::string &content_type);
// 发送 HTTP POST 请求，传入路径、请求体、内容类型
Result Post(const std::string &path, const std::string &body, const std::string &content_type);
// 发送 HTTP POST 请求，传入路径、自定义头部、请求体、内容类型
Result Post(const std::string &path, const Headers &headers, const std::string &body, const std::string &content_type);
// 发送 HTTP POST 请求，传入路径、内容长度、内容提供者和内容类型
Result Post(const std::string &path, size_t content_length, ContentProvider content_provider, const std::string &content_type);
// 发送 HTTP POST 请求，传入路径
# 使用给定的内容提供者和内容类型进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              ContentProviderWithoutLength content_provider,
              const std::string &content_type);

# 使用给定的路径、头部、内容长度、内容提供者和内容类型进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              size_t content_length, ContentProvider content_provider,
              const std::string &content_type);

# 使用给定的路径、头部、内容提供者和内容类型进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              ContentProviderWithoutLength content_provider,
              const std::string &content_type);

# 使用给定的路径和参数进行 POST 请求
Result Post(const std::string &path, const Params &params);

# 使用给定的路径、头部和参数进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              const Params &params);

# 使用给定的路径和多部分表单数据项进行 POST 请求
Result Post(const std::string &path, const MultipartFormDataItems &items);

# 使用给定的路径、头部和多部分表单数据项进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items);

# 使用给定的路径、头部、多部分表单数据项和边界进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items, const std::string &boundary);

# 使用给定的路径、头部、多部分表单数据项和多部分表单数据提供者项进行 POST 请求
Result Post(const std::string &path, const Headers &headers,
              const MultipartFormDataItems &items,
              const MultipartFormDataProviderItems &provider_items);
// 将指定路径的内容放入结果中
Result Put(const std::string &path);

// 将指定路径的内容以指定的内容类型和长度放入结果中
Result Put(const std::string &path, const char *body, size_t content_length, const std::string &content_type);

// 将指定路径的内容以指定的头部信息、内容类型和长度放入结果中
Result Put(const std::string &path, const Headers &headers, const char *body, size_t content_length, const std::string &content_type);

// 将指定路径的内容以指定的内容类型和字符串形式的数据放入结果中
Result Put(const std::string &path, const std::string &body, const std::string &content_type);

// 将指定路径的内容以指定的头部信息、内容类型和字符串形式的数据放入结果中
Result Put(const std::string &path, const Headers &headers, const std::string &body, const std::string &content_type);

// 将指定路径的内容以指定的内容类型和长度以及内容提供者放入结果中
Result Put(const std::string &path, size_t content_length, ContentProvider content_provider, const std::string &content_type);

// 将指定路径的内容以指定的内容类型和没有长度的内容提供者放入结果中
Result Put(const std::string &path, ContentProviderWithoutLength content_provider, const std::string &content_type);

// 将指定路径的内容以指定的头部信息、内容类型和长度以及内容提供者放入结果中
Result Put(const std::string &path, const Headers &headers, size_t content_length, ContentProvider content_provider, const std::string &content_type);

// 将指定路径的内容以指定的头部信息、没有长度的内容提供者和内容类型放入结果中
Result Put(const std::string &path, const Headers &headers, ContentProviderWithoutLength content_provider, const std::string &content_type);
# 使用给定的路径和参数执行 PUT 请求，并返回结果
Result Put(const std::string &path, const Params &params);

# 使用给定的路径、头部和参数执行 PUT 请求，并返回结果
Result Put(const std::string &path, const Headers &headers, const Params &params);

# 使用给定的路径和多部分表单数据项执行 PUT 请求，并返回结果
Result Put(const std::string &path, const MultipartFormDataItems &items);

# 使用给定的路径、头部和多部分表单数据项执行 PUT 请求，并返回结果
Result Put(const std::string &path, const Headers &headers, const MultipartFormDataItems &items);

# 使用给定的路径、头部、多部分表单数据项和边界执行 PUT 请求，并返回结果
Result Put(const std::string &path, const Headers &headers, const MultipartFormDataItems &items, const std::string &boundary);

# 使用给定的路径、头部、多部分表单数据项和提供者项执行 PUT 请求，并返回结果
Result Put(const std::string &path, const Headers &headers, const MultipartFormDataItems &items, const MultipartFormDataProviderItems &provider_items);

# 使用给定的路径执行 PATCH 请求，并返回结果
Result Patch(const std::string &path);

# 使用给定的路径、内容、内容长度和内容类型执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const char *body, size_t content_length, const std::string &content_type);

# 使用给定的路径、头部、内容、内容长度和内容类型执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const Headers &headers, const char *body, size_t content_length, const std::string &content_type);

# 使用给定的路径、内容和内容类型执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const std::string &body, const std::string &content_type);
// 使用给定的路径、头部和请求体来执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const Headers &headers,
             const std::string &body, const std::string &content_type);

// 使用给定的路径、内容长度、内容提供者和内容类型来执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, size_t content_length,
             ContentProvider content_provider,
             const std::string &content_type);

// 使用给定的路径、无长度的内容提供者和内容类型来执行 PATCH 请求，并返回结果
Result Patch(const std::string &path,
             ContentProviderWithoutLength content_provider,
             const std::string &content_type);

// 使用给定的路径、头部、内容长度、内容提供者和内容类型来执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const Headers &headers,
             size_t content_length, ContentProvider content_provider,
             const std::string &content_type);

// 使用给定的路径、头部、无长度的内容提供者和内容类型来执行 PATCH 请求，并返回结果
Result Patch(const std::string &path, const Headers &headers,
             ContentProviderWithoutLength content_provider,
             const std::string &content_type);

// 使用给定的路径来执行 DELETE 请求，并返回结果
Result Delete(const std::string &path);

// 使用给定的路径和头部来执行 DELETE 请求，并返回结果
Result Delete(const std::string &path, const Headers &headers);

// 使用给定的路径、请求体、内容长度和内容类型来执行 DELETE 请求，并返回结果
Result Delete(const std::string &path, const char *body,
              size_t content_length, const std::string &content_type);

// 使用给定的路径、头部和内容类型来执行 DELETE 请求，并返回结果
Result Delete(const std::string &path, const Headers &headers,
              const std::string &content_type);
// 声明一个函数，接受指向字符的指针、内容长度和内容类型作为参数
Result Post(const std::string &path, const char *body, size_t content_length,
            const std::string &content_type);

// 声明一个函数，接受路径和内容作为参数，用于删除资源
Result Delete(const std::string &path, const std::string &body,
              const std::string &content_type);

// 声明一个函数，接受路径、头部、内容和内容类型作为参数，用于删除资源
Result Delete(const std::string &path, const Headers &headers,
              const std::string &body, const std::string &content_type);

// 声明一个函数，接受路径作为参数，用于发送 OPTIONS 请求
Result Options(const std::string &path);

// 声明一个函数，接受路径和头部作为参数，用于发送 OPTIONS 请求
Result Options(const std::string &path, const Headers &headers);

// 声明一个函数，接受请求和响应作为参数，用于发送请求并获取响应
bool send(Request &req, Response &res, Error &error);

// 声明一个函数，接受请求作为参数，用于发送请求并获取响应
Result send(const Request &req);

// 返回套接字是否打开的大小
size_t is_socket_open() const;

// 返回套接字
socket_t socket() const;

// 停止操作
void stop();

// 设置主机名和地址的映射关系
void set_hostname_addr_map(std::map<std::string, std::string> addr_map);
// 设置默认的请求头
void set_default_headers(Headers headers);

// 设置地址族
void set_address_family(int family);

// 设置是否启用 TCP 的无延迟模式
void set_tcp_nodelay(bool on);

// 设置套接字选项
void set_socket_options(SocketOptions socket_options);

// 设置连接超时时间，单位为秒和微秒
void set_connection_timeout(time_t sec, time_t usec = 0);
// 设置连接超时时间，使用 std::chrono::duration 表示
template <class Rep, class Period>
void set_connection_timeout(const std::chrono::duration<Rep, Period> &duration);

// 设置读取超时时间，单位为秒和微秒
void set_read_timeout(time_t sec, time_t usec = 0);
// 设置读取超时时间，使用 std::chrono::duration 表示
template <class Rep, class Period>
void set_read_timeout(const std::chrono::duration<Rep, Period> &duration);

// 设置写入超时时间，单位为秒和微秒
void set_write_timeout(time_t sec, time_t usec = 0);
// 设置写入超时时间，使用 std::chrono::duration 表示
template <class Rep, class Period>
void set_write_timeout(const std::chrono::duration<Rep, Period> &duration);
// 设置基本身份验证的用户名和密码
void set_basic_auth(const std::string &username, const std::string &password);

// 设置Bearer令牌身份验证的令牌
void set_bearer_token_auth(const std::string &token);

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要身份验证的用户名和密码
  void set_digest_auth(const std::string &username,
                       const std::string &password);
#endif

// 设置是否保持连接
void set_keep_alive(bool on);

// 设置是否跟随重定向
void set_follow_location(bool on);

// 设置是否进行URL编码
void set_url_encode(bool on);

// 设置是否压缩请求数据
void set_compress(bool on);

// 设置是否解压响应数据
void set_decompress(bool on);

// 设置网络接口
void set_interface(const std::string &intf);

// 设置代理服务器的主机和端口
void set_proxy(const std::string &host, int port);

// 设置代理服务器的基本身份验证的用户名和密码
void set_proxy_basic_auth(const std::string &username,
// 设置代理服务器的基本认证信息
void set_proxy_basic_auth(const std::string &username, const std::string &password);

// 设置代理服务器的 Bearer Token 认证信息
void set_proxy_bearer_token_auth(const std::string &token);

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置代理服务器的摘要认证信息
  void set_proxy_digest_auth(const std::string &username, const std::string &password);
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 启用/禁用服务器证书验证
  void enable_server_certificate_verification(bool enabled);
#endif

// 设置日志记录器
void set_logger(Logger logger);

// SSL
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置 CA 证书文件路径和目录路径
  void set_ca_cert_path(const std::string &ca_cert_file_path, const std::string &ca_cert_dir_path = std::string());

  // 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);
// 获取 OpenSSL 验证结果
long get_openssl_verify_result() const;

// 获取 SSL 上下文
SSL_CTX *ssl_context() const;
#endif

private:
// 客户端实现的唯一指针
std::unique_ptr<ClientImpl> cli_;

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 是否为 SSL 连接
bool is_ssl_ = false;
#endif
};

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// SSL 服务器类，继承自服务器类
class SSLServer : public Server {
public:
  // SSL 服务器构造函数，接受证书路径、私钥路径、客户端 CA 证书文件路径、客户端 CA 证书目录路径、私钥密码
  SSLServer(const char *cert_path, const char *private_key_path,
            const char *client_ca_cert_file_path = nullptr,
            const char *client_ca_cert_dir_path = nullptr,
            const char *private_key_password = nullptr);
// SSLServer 类的构造函数，接受 X509 证书、私钥和客户端 CA 证书存储作为参数
SSLServer(X509 *cert, EVP_PKEY *private_key, X509_STORE *client_ca_cert_store = nullptr);

// SSLServer 类的构造函数，接受一个设置 SSL 上下文回调函数作为参数
SSLServer(const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback);

// SSLServer 类的析构函数
~SSLServer() override;

// 检查 SSLServer 对象是否有效
bool is_valid() const override;

// 获取 SSL 上下文
SSL_CTX *ssl_context() const;

private:
// 处理并关闭套接字的私有方法
bool process_and_close_socket(socket_t sock) override;

// SSL 上下文
SSL_CTX *ctx_;

// SSL 上下文互斥锁
std::mutex ctx_mutex_;
# 定义一个名为 SSLClient 的类，继承自 ClientImpl 类
class SSLClient : public ClientImpl {
public:
  # 构造函数，接受一个字符串类型的参数作为主机名
  explicit SSLClient(const std::string &host);

  # 构造函数，接受两个参数，分别为主机名和端口号
  explicit SSLClient(const std::string &host, int port);

  # 构造函数，接受四个参数，分别为主机名、端口号、客户端证书路径和客户端密钥路径
  explicit SSLClient(const std::string &host, int port,
                     const std::string &client_cert_path,
                     const std::string &client_key_path);

  # 构造函数，接受三个参数，分别为主机名、端口号、客户端证书和客户端密钥
  explicit SSLClient(const std::string &host, int port, X509 *client_cert,
                     EVP_PKEY *client_key);

  # 析构函数，用于释放资源
  ~SSLClient() override;

  # 重写父类的方法，用于检查 SSLClient 对象是否有效
  bool is_valid() const override;

  # 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);

  # 获取 OpenSSL 验证结果
  long get_openssl_verify_result() const;
// 返回 SSL 上下文
SSL_CTX *ssl_context() const;

private:
  // 创建并连接套接字
  bool create_and_connect_socket(Socket &socket, Error &error) override;
  // 关闭 SSL 连接
  void shutdown_ssl(Socket &socket, bool shutdown_gracefully) override;
  // 实际执行 SSL 关闭操作
  void shutdown_ssl_impl(Socket &socket, bool shutdown_socket);

  // 处理套接字，传入回调函数
  bool process_socket(const Socket &socket,
                      std::function<bool(Stream &strm)> callback) override;
  // 判断是否为 SSL 连接
  bool is_ssl() const override;

  // 使用代理连接套接字
  bool connect_with_proxy(Socket &sock, Response &res, bool &success,
                          Error &error);
  // 初始化 SSL 连接
  bool initialize_ssl(Socket &socket, Error &error);

  // 加载证书
  bool load_certs();

  // 验证主机名
  bool verify_host(X509 *server_cert) const;
  // 使用主题备用名称验证主机名
  bool verify_host_with_subject_alt_name(X509 *server_cert) const;
  // 验证服务器证书的通用名称与主机名是否匹配
  bool verify_host_with_common_name(X509 *server_cert) const;
  // 检查主机名是否匹配给定的模式和长度
  bool check_host_name(const char *pattern, size_t pattern_len) const;

  // SSL 上下文对象
  SSL_CTX *ctx_;
  // 用于保护 SSL 上下文对象的互斥锁
  std::mutex ctx_mutex_;
  // 用于确保证书初始化只执行一次的标志
  std::once_flag initialize_cert_;

  // 存储主机名的组件的向量
  std::vector<std::string> host_components_;

  // 用于存储验证结果的变量，默认为0
  long verify_result_ = 0;

  // 声明友元类 ClientImpl
  friend class ClientImpl;
};
#endif

/*
 * Implementation of template methods.
 */

// 实现模板方法的命名空间
namespace detail {
// 将持续时间转换为秒和微秒，并通过回调函数返回结果
template <typename T, typename U>
inline void duration_to_sec_and_usec(const T &duration, U callback) {
  // 将持续时间转换为秒，并获取秒数
  auto sec = std::chrono::duration_cast<std::chrono::seconds>(duration).count();
  // 将持续时间减去秒数后转换为微秒，并获取微秒数
  auto usec = std::chrono::duration_cast<std::chrono::microseconds>(
                  duration - std::chrono::seconds(sec))
                  .count();
  // 调用回调函数，传递秒数和微秒数
  callback(static_cast<time_t>(sec), static_cast<time_t>(usec));
}

// 获取头部值的模板函数，返回类型为T
template <typename T>
inline T get_header_value(const Headers & /*headers*/,
                          const std::string & /*key*/, size_t /*id*/ = 0,
                          uint64_t /*def*/ = 0) {}

// 获取头部值的模板函数，返回类型为uint64_t
template <>
inline uint64_t get_header_value<uint64_t>(const Headers &headers,
                                           const std::string &key, size_t id,
                                           uint64_t def) {
  // 查找头部中指定键的所有值的范围
  auto rng = headers.equal_range(key);
}
// 使用迭代器it指向范围rng的第一个元素
auto it = rng.first;
// 将迭代器it向前移动id个位置
std::advance(it, static_cast<ssize_t>(id));
// 如果迭代器it不等于范围rng的末尾迭代器
if (it != rng.second) {
  // 将it指向的值转换为无符号长长整型并返回
  return std::strtoull(it->second.data(), nullptr, 10);
}
// 如果迭代器it等于范围rng的末尾迭代器，则返回默认值def
return def;
}

} // namespace detail

// 获取请求对象中指定键的header值
template <typename T>
inline T Request::get_header_value(const std::string &key, size_t id) const {
  // 调用detail命名空间中的get_header_value函数
  return detail::get_header_value<T>(headers, key, id, 0);
}

// 获取响应对象中指定键的header值
template <typename T>
inline T Response::get_header_value(const std::string &key, size_t id) const {
  // 调用detail命名空间中的get_header_value函数
  return detail::get_header_value<T>(headers, key, id, 0);
}
// 写入格式化数据到流中，返回写入的字节数
template <typename... Args>
inline ssize_t Stream::write_format(const char *fmt, const Args &...args) {
  // 定义缓冲区大小
  const auto bufsiz = 2048;
  // 创建固定大小的字符数组作为缓冲区
  std::array<char, bufsiz> buf{};

  // 使用给定的格式和参数将数据格式化写入缓冲区
  auto sn = snprintf(buf.data(), buf.size() - 1, fmt, args...);
  // 如果写入失败，返回错误码
  if (sn <= 0) { return sn; }

  // 转换写入的字节数为无符号整数
  auto n = static_cast<size_t>(sn);

  // 如果写入的字节数超过缓冲区大小减一
  if (n >= buf.size() - 1) {
    // 创建可增长的字符数组作为缓冲区
    std::vector<char> glowable_buf(buf.size());

    // 当写入的字节数超过可增长缓冲区大小减一时，增大缓冲区大小并重新写入
    while (n >= glowable_buf.size() - 1) {
      glowable_buf.resize(glowable_buf.size() * 2);
      n = static_cast<size_t>(
          snprintf(&glowable_buf[0], glowable_buf.size() - 1, fmt, args...));
    }
    // 将数据写入流中并返回写入的字节数
    return write(&glowable_buf[0], n);
  } else {
// 返回写入的数据长度
return write(buf.data(), n);
}

// 设置默认的套接字选项
inline void default_socket_options(socket_t sock) {
  int yes = 1;
#ifdef _WIN32
  // 设置套接字选项为允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, reinterpret_cast<char *>(&yes),
             sizeof(yes));
  // 设置套接字选项为独占地址使用
  setsockopt(sock, SOL_SOCKET, SO_EXCLUSIVEADDRUSE,
             reinterpret_cast<char *>(&yes), sizeof(yes));
#else
#ifdef SO_REUSEPORT
  // 如果支持 SO_REUSEPORT，则设置套接字选项为允许端口重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEPORT, reinterpret_cast<void *>(&yes),
             sizeof(yes));
#else
  // 否则设置套接字选项为允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, reinterpret_cast<void *>(&yes),
             sizeof(yes));
#endif
#endif
```
// 设置读取超时时间，接受一个持续时间参数，并将其转换为秒和微秒，然后调用set_read_timeout函数设置读取超时时间
template <class Rep, class Period>
inline Server &
Server::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
  return *this;
}

// 设置写入超时时间，接受一个持续时间参数，并将其转换为秒和微秒，然后调用set_write_timeout函数设置写入超时时间
template <class Rep, class Period>
inline Server &
Server::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_write_timeout(sec, usec); });
  return *this;
}

// 其他函数的定义
template <class Rep, class Period>
inline Server &
// 设置服务器空闲时间间隔
Server::set_idle_interval(const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用 set_idle_interval 函数
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_idle_interval(sec, usec); });
  // 返回当前对象的引用
  return *this;
}

// 将错误枚举转换为字符串
inline std::string to_string(const Error error) {
  // 根据错误枚举返回相应的错误信息
  switch (error) {
  case Error::Success: return "Success (no error)";
  case Error::Connection: return "Could not establish connection";
  case Error::BindIPAddress: return "Failed to bind IP address";
  case Error::Read: return "Failed to read connection";
  case Error::Write: return "Failed to write connection";
  case Error::ExceedRedirectCount: return "Maximum redirect count exceeded";
  case Error::Canceled: return "Connection handling canceled";
  case Error::SSLConnection: return "SSL connection failed";
  case Error::SSLLoadingCerts: return "SSL certificate loading failed";
  case Error::SSLServerVerification: return "SSL server verification failed";
  case Error::UnsupportedMultipartBoundaryChars:
    return "Unsupported HTTP multipart boundary characters";
  }
}
// 根据错误类型返回对应的错误信息
case Error::Compression: return "Compression failed";
case Error::ConnectionTimeout: return "Connection timed out";
case Error::Unknown: return "Unknown";
default: break;
}

// 重载输出流操作符，将错误类型转换为字符串输出
inline std::ostream &operator<<(std::ostream &os, const Error &obj) {
  os << to_string(obj); // 将错误类型转换为字符串输出
  os << " (" << static_cast<std::underlying_type<Error>::type>(obj) << ')'; // 输出错误类型的数值表示
  return os;
}

// 获取请求头部的值
template <typename T>
inline T Result::get_request_header_value(const std::string &key,
                                          size_t id) const {
  return detail::get_header_value<T>(request_headers_, key, id, 0); // 获取请求头部指定键的值
}
// 设置连接超时时间
template <class Rep, class Period>
inline void ClientImpl::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用 set_connection_timeout 方法设置连接超时时间
  detail::duration_to_sec_and_usec(duration, [&](time_t sec, time_t usec) {
    set_connection_timeout(sec, usec);
  });
}

// 设置读取超时时间
template <class Rep, class Period>
inline void ClientImpl::set_read_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用 set_read_timeout 方法设置读取超时时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
}

// 设置写入超时时间
template <class Rep, class Period>
inline void ClientImpl::set_write_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用 set_write_timeout 方法设置写入超时时间
  detail::duration_to_sec_and_usec(
// 设置连接超时时间
template <class Rep, class Period>
inline void Client::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 调用底层客户端对象的设置连接超时时间的方法
  cli_->set_connection_timeout(duration);
}

// 设置读取超时时间
template <class Rep, class Period>
inline void
Client::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 调用底层客户端对象的设置读取超时时间的方法
  cli_->set_read_timeout(duration);
}

// 设置写入超时时间
template <class Rep, class Period>
inline void
Client::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 调用底层客户端对象的设置写入超时时间的方法
  cli_->set_write_timeout(duration);
}
/*
 * 前向声明和类型，如果拆分成 .h + .cc 文件，这些将成为 .h 文件的一部分。
 */

// 返回指定主机名的主机地址
std::string hosted_at(const std::string &hostname);

// 返回指定主机名的主机地址列表
void hosted_at(const std::string &hostname, std::vector<std::string> &addrs);

// 在路径后附加查询参数
std::string append_query_params(const std::string &path, const Params &params);

// 生成范围请求的头部信息
std::pair<std::string, std::string> make_range_header(Ranges ranges);

// 生成基本身份验证头部信息
std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password,
                                 bool is_proxy = false);

// 命名空间 detail
namespace detail {
*/
// 对给定的值进行查询参数编码
std::string encode_query_param(const std::string &value);

// 对给定的字符串进行 URL 解码，可以选择是否将加号转换为空格
std::string decode_url(const std::string &s, bool convert_plus_to_space);

// 读取指定路径的文件内容，并将结果存储在给定的字符串中
void read_file(const std::string &path, std::string &out);

// 复制给定字符串并去除两端的空格，返回处理后的字符串副本
std::string trim_copy(const std::string &s);

// 将给定范围内的字符按照指定分隔符进行分割，并对每个分割部分执行给定的函数
void split(const char *b, const char *e, char d,
           std::function<void(const char *, const char *)> fn);

// 处理客户端套接字的函数，包括读取超时时间、写入超时时间和回调函数
bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                           time_t read_timeout_usec, time_t write_timeout_sec,
                           time_t write_timeout_usec,
                           std::function<bool(Stream &)> callback);

// 创建客户端套接字，包括主机名、IP 地址、端口号、地址族、TCP 无延迟、套接字选项等参数
socket_t create_client_socket(
    const std::string &host, const std::string &ip, int port,
    int address_family, bool tcp_nodelay, SocketOptions socket_options,
// 设置连接超时时间（秒和微秒）、读取超时时间（秒和微秒）、写入超时时间（秒和微秒）、接口名称和错误信息
void set_timeout(time_t connection_timeout_sec, time_t connection_timeout_usec,
                 time_t read_timeout_sec, time_t read_timeout_usec, 
                 time_t write_timeout_sec, time_t write_timeout_usec, 
                 const std::string &intf, Error &error);

// 获取指定键对应的头部值，可以指定索引和默认值
const char *get_header_value(const Headers &headers, const std::string &key,
                             size_t id = 0, const char *def = nullptr);

// 将参数转换为查询字符串
std::string params_to_query_str(const Params &params);

// 解析查询文本并存储到参数中
void parse_query_text(const std::string &s, Params &params);

// 解析多部分内容类型的边界
bool parse_multipart_boundary(const std::string &content_type,
                              std::string &boundary);

// 解析范围头部，存储到范围对象中
bool parse_range_header(const std::string &s, Ranges &ranges);

// 关闭套接字
int close_socket(socket_t sock);

// 发送数据到套接字
ssize_t send_socket(socket_t sock, const void *ptr, size_t size, int flags);
# 从套接字中读取数据到指定的缓冲区中，返回读取的字节数
ssize_t read_socket(socket_t sock, void *ptr, size_t size, int flags);

# 返回请求和响应的编码类型，可能是无编码、Gzip 或 Brotli
enum class EncodingType { None = 0, Gzip, Brotli };

# 缓冲流类，继承自流类
class BufferStream : public Stream {
public:
  # 默认构造函数
  BufferStream() = default;
  # 虚析构函数
  ~BufferStream() override = default;

  # 判断流是否可读
  bool is_readable() const override;
  # 判断流是否可写
  bool is_writable() const override;
  # 从流中读取数据到指定的缓冲区中，返回读取的字节数
  ssize_t read(char *ptr, size_t size) override;
  # 将指定的数据写入流中，返回写入的字节数
  ssize_t write(const char *ptr, size_t size) override;
  # 获取远程 IP 地址和端口号
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  # 获取本地 IP 地址和端口号
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  # 获取套接字
  socket_t socket() const override;

  # 获取缓冲区内容
  const std::string &get_buffer() const;
# 定义私有成员变量，用于存储数据和当前位置
private:
  std::string buffer;  # 存储数据的字符串
  size_t position = 0;  # 当前位置的偏移量

# 定义压缩器类
class compressor {
public:
  virtual ~compressor() = default;  # 声明虚析构函数

  # 定义回调函数类型，用于在压缩过程中传递数据
  typedef std::function<bool(const char *data, size_t data_len)> Callback;
  # 声明纯虚函数，用于压缩数据
  virtual bool compress(const char *data, size_t data_length, bool last,
                        Callback callback) = 0;
};

# 定义解压器类
class decompressor {
public:
  virtual ~decompressor() = default;  # 声明虚析构函数

  # 声明纯虚函数，用于检查解压器是否有效
  virtual bool is_valid() const = 0;
```

// 使用 std::function 定义一个回调函数类型，该函数接受一个指向字符数据和数据长度的指针，并返回布尔值
typedef std::function<bool(const char *data, size_t data_len)> Callback;

// 定义一个虚函数 decompress，用于解压缩数据，接受原始数据指针、数据长度和回调函数作为参数
virtual bool decompress(const char *data, size_t data_length, Callback callback) = 0;
};

// 定义一个 nocompressor 类，继承自 compressor 类
class nocompressor : public compressor {
public:
  // 默认析构函数
  virtual ~nocompressor() = default;

  // 重写 compress 函数，用于压缩数据，接受原始数据指针、数据长度、是否为最后一块数据和回调函数作为参数
  bool compress(const char *data, size_t data_length, bool /*last*/, Callback callback) override;
};

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 定义一个 gzip_compressor 类，继承自 compressor 类
class gzip_compressor : public compressor {
public:
  // 构造函数
  gzip_compressor();
  // 析构函数
  ~gzip_compressor();
// 压缩函数，接受数据和数据长度，是否为最后一块数据，以及回调函数作为参数
bool compress(const char *data, size_t data_length, bool last, Callback callback) override;

private:
// 用于标识压缩器是否有效的布尔变量
bool is_valid_ = false;
// 用于压缩数据的 z_stream 对象
z_stream strm_;
};

// gzip 解压缩器类，继承自解压缩器类
class gzip_decompressor : public decompressor {
public:
// 默认构造函数
gzip_decompressor();
// 析构函数
~gzip_decompressor();

// 判断解压缩器是否有效的函数
bool is_valid() const override;

// 解压缩函数，接受数据和数据长度，以及回调函数作为参数
bool decompress(const char *data, size_t data_length, Callback callback) override;

private:
// 用于标识解压缩器是否有效的布尔变量
bool is_valid_ = false;
// 如果支持Brotli压缩，定义Brotli压缩器类
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
class brotli_compressor : public compressor {
public:
  // 构造函数
  brotli_compressor();
  // 析构函数
  ~brotli_compressor();

  // 压缩数据
  bool compress(const char *data, size_t data_length, bool last,
                Callback callback) override;

private:
  // Brotli压缩状态
  BrotliEncoderState *state_ = nullptr;
};

// 如果支持Brotli解压缩，定义Brotli解压缩器类
class brotli_decompressor : public decompressor {
public:
  // 构造函数
  brotli_decompressor();
// 调用brotli解压缩器的析构函数
~brotli_decompressor();

// 检查解压缩器是否有效
bool is_valid() const override;

// 解压缩数据，并通过回调函数返回结果
bool decompress(const char *data, size_t data_length,
                Callback callback) override;

private:
  // 存储解压缩结果的状态
  BrotliDecoderResult decoder_r;
  // 解压缩器的状态
  BrotliDecoderState *decoder_s = nullptr;
};
#endif

// 注意：直到读取的大小达到`fixed_buffer_size`，使用`fixed_buffer`来存储数据。调用可以在堆栈上设置内存以提高性能。
class stream_line_reader {
public:
  // 构造函数，初始化流和固定缓冲区
  stream_line_reader(Stream &strm, char *fixed_buffer,
                     size_t fixed_buffer_size);
  // 返回指向数据的指针
  const char *ptr() const;
// 返回流的大小
size_t size() const;

// 检查流是否以回车换行符结尾
bool end_with_crlf() const;

// 从流中读取一行数据
bool getline();

// 在内部缓冲区中追加字符
void append(char c);

// 流对象的引用
Stream &strm_;

// 固定大小的缓冲区
char *fixed_buffer_;

// 固定缓冲区的大小
const size_t fixed_buffer_size_;

// 固定缓冲区已使用的大小
size_t fixed_buffer_used_size_ = 0;

// 可增长的缓冲区
std::string glowable_buffer_;
// 定义一个内部命名空间，用于存放一些内部实现细节的函数或变量
namespace detail {

// 判断字符是否为十六进制，并将其转换为对应的整数值
inline bool is_hex(char c, int &v) {
  if (0x20 <= c && isdigit(c)) {  // 判断字符是否为数字
    v = c - '0';  // 将字符转换为对应的整数值
    return true;
  } else if ('A' <= c && c <= 'F') {  // 判断字符是否为大写字母A-F
    v = c - 'A' + 10;  // 将字符转换为对应的整数值
    return true;
  } else if ('a' <= c && c <= 'f') {  // 判断字符是否为小写字母a-f
    v = c - 'a' + 10;  // 将字符转换为对应的整数值
    return true;
  }
  return false;  // 如果不是十六进制字符，则返回false
}

// 将十六进制字符串转换为整数值
inline bool from_hex_to_i(const std::string &s, size_t i, size_t cnt,
                          int &val) {
  // 如果 i 大于等于字符串 s 的大小，则返回 false
  if (i >= s.size()) { return false; }

  // 初始化 val 为 0
  val = 0;
  // 循环，每次循环减少 cnt，增加 i
  for (; cnt; i++, cnt--) {
    // 如果 s[i] 为 0，则返回 false
    if (!s[i]) { return false; }
    // 初始化 v 为 0
    int v = 0;
    // 如果 s[i] 是十六进制字符，则将其转换为整数并赋值给 v，然后更新 val
    if (is_hex(s[i], v)) {
      val = val * 16 + v;
    } else {
      // 如果 s[i] 不是十六进制字符，则返回 false
      return false;
    }
  }
  // 循环结束后返回 true
  return true;
}

// 将整数转换为十六进制字符串
inline std::string from_i_to_hex(size_t n) {
  // 十六进制字符集
  const char *charset = "0123456789abcdef";
  // 初始化结果字符串
  std::string ret;
  // 循环将整数转换为十六进制字符串
  do {
    ret = charset[n & 15] + ret;
// 将整数右移4位，相当于除以16，用于UTF-8编码
    n >>= 4;
  } while (n > 0);
  return ret;
}

// 将Unicode编码转换为UTF-8编码
inline size_t to_utf8(int code, char *buff) {
  // 如果编码小于0x0080，使用一个字节表示
    buff[0] = (code & 0x7F);
    return 1;
  // 如果编码小于0x0800，使用两个字节表示
  } else if (code < 0x0800) {
    buff[0] = static_cast<char>(0xC0 | ((code >> 6) & 0x1F));
    buff[1] = static_cast<char>(0x80 | (code & 0x3F));
    return 2;
  // 如果编码小于0xD800，使用三个字节表示
  } else if (code < 0xD800) {
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    return 3;
  // 如果编码在0xD800到0xDFFF之间，表示为无效编码
  } else if (code < 0xE000) { // D800 - DFFF is invalid...
    return 0;
  } else if (code < 0x10000) {
    // 如果 Unicode 编码小于 0x10000，则使用 3 个字节表示
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    // 返回 3，表示使用了 3 个字节
    return 3;
  } else if (code < 0x110000) {
    // 如果 Unicode 编码小于 0x110000，则使用 4 个字节表示
    buff[0] = static_cast<char>(0xF0 | ((code >> 18) & 0x7));
    buff[1] = static_cast<char>(0x80 | ((code >> 12) & 0x3F));
    buff[2] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[3] = static_cast<char>(0x80 | (code & 0x3F));
    // 返回 4，表示使用了 4 个字节
    return 4;
  }

  // NOTREACHED
  // 如果未到达此处，表示出现了意外情况
  return 0;
}

// NOTE: This code came up with the following stackoverflow post:
// https://stackoverflow.com/questions/180947/base64-decode-snippet-in-c
// 注意：此代码来源于以下 stackoverflow 帖子
// https://stackoverflow.com/questions/180947/base64-decode-snippet-in-c
inline std::string base64_encode(const std::string &in) {
// 在此处实现 base64 编码函数
// 定义一个常量lookup，用于Base64编码中的字符映射
static const auto lookup =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

// 初始化一个空字符串out，预留in字符串大小的空间
std::string out;
out.reserve(in.size());

// 初始化val和valb变量，用于存储编码过程中的临时值
int val = 0;
int valb = -6;

// 遍历输入字符串in的每个字符
for (auto c : in) {
  // 将字符转换为8位整数，并存入val中
  val = (val << 8) + static_cast<uint8_t>(c);
  // 更新valb的值
  valb += 8;
  // 当valb大于等于0时，进行Base64编码并添加到out字符串中
  while (valb >= 0) {
    out.push_back(lookup[(val >> valb) & 0x3F]);
    valb -= 6;
  }
}

// 如果valb大于-6，进行Base64编码并添加到out字符串中
if (valb > -6) { out.push_back(lookup[((val << 8) >> (valb + 8)) & 0x3F]); }
// 当输出大小不是4的倍数时，向输出末尾添加'='，使其大小为4的倍数
while (out.size() % 4) {
  out.push_back('=');
}

// 检查给定路径是否为文件
inline bool is_file(const std::string &path) {
#ifdef _WIN32
  // 在 Windows 平台上使用_access_s函数检查文件是否存在
  return _access_s(path.c_str(), 0) == 0;
#else
  // 在其他平台上使用stat函数检查文件是否存在，并且判断其是否为普通文件
  struct stat st;
  return stat(path.c_str(), &st) >= 0 && S_ISREG(st.st_mode);
#endif
}

// 检查给定路径是否为目录
inline bool is_dir(const std::string &path) {
  // 使用stat函数检查路径是否存在，并且判断其是否为目录
  struct stat st;
  return stat(path.c_str(), &st) >= 0 && S_ISDIR(st.st_mode);
}
// 检查给定路径是否有效
inline bool is_valid_path(const std::string &path) {
  size_t level = 0; // 初始化路径级别为0
  size_t i = 0; // 初始化索引为0

  // 跳过斜杠
  while (i < path.size() && path[i] == '/') {
    i++;
  }

  while (i < path.size()) {
    // 读取路径组件
    auto beg = i; // 记录组件起始位置
    while (i < path.size() && path[i] != '/') {
      i++;
    }

    auto len = i - beg; // 计算组件长度
    assert(len > 0); // 断言组件长度大于0
    // 如果路径是当前目录"."，则不做任何操作
    if (!path.compare(beg, len, ".")) {
      ;
    } 
    // 如果路径是上一级目录".."，且当前层级为0，则返回false
    else if (!path.compare(beg, len, "..")) {
      if (level == 0) { return false; }
      level--;
    } 
    // 其他情况，层级加一
    else {
      level++;
    }

    // 跳过斜杠
    while (i < path.size() && path[i] == '/') {
      i++;
    }
  }

  return true;
}

// 对查询参数进行编码
inline std::string encode_query_param(const std::string &value) {
  // 创建一个字符串输出流对象
  std::ostringstream escaped;
  // 将字符串填充为 '0'
  escaped.fill('0');
  // 设置输出流为十六进制
  escaped << std::hex;

  // 遍历输入的字符串
  for (auto c : value) {
    // 如果字符是字母或数字，或者是以下特殊字符之一：- _ . ! ~ * ' ( )
    if (std::isalnum(static_cast<uint8_t>(c)) || c == '-' || c == '_' ||
        c == '.' || c == '!' || c == '~' || c == '*' || c == '\'' || c == '(' ||
        c == ')') {
      // 直接输出字符
      escaped << c;
    } else {
      // 将字符转换为大写
      escaped << std::uppercase;
      // 输出 % 后面跟着两位十六进制表示的字符
      escaped << '%' << std::setw(2)
              << static_cast<int>(static_cast<unsigned char>(c));
      // 将字符转换为小写
      escaped << std::nouppercase;
    }
  }

  // 返回转义后的字符串
  return escaped.str();
}

// 对输入的字符串进行 URL 编码
inline std::string encode_url(const std::string &s) {
  // 创建一个字符串变量result，预留与输入字符串s相同大小的空间
  std::string result;
  result.reserve(s.size());

  // 遍历输入字符串s的每个字符
  for (size_t i = 0; s[i]; i++) {
    // 根据字符的不同情况进行处理
    switch (s[i]) {
    case ' ': result += "%20"; break;  // 空格替换为%20
    case '+': result += "%2B"; break;   // +替换为%2B
    case '\r': result += "%0D"; break;  // 回车符替换为%0D
    case '\n': result += "%0A"; break;  // 换行符替换为%0A
    case '\'': result += "%27"; break;  // 单引号替换为%27
    case ',': result += "%2C"; break;   // 逗号替换为%2C
    // case ':': result += "%3A"; break; // ok? probably...  // 冒号替换为%3A，可能需要确认
    case ';': result += "%3B"; break;   // 分号替换为%3B
    default:
      auto c = static_cast<uint8_t>(s[i]);
      // 如果字符的ASCII码大于等于0x80，将其转换为十六进制表示
      if (c >= 0x80) {
        result += '%';
        char hex[4];
        auto len = snprintf(hex, sizeof(hex) - 1, "%02X", c);
        assert(len == 2);
        // 如果遇到 '%' 符号并且后面还有字符
        if (s[i] == '%' && i + 1 < s.size()) {
            // 如果下一个字符是 'u'
            if (s[i + 1] == 'u') {
                // 从十六进制转换为整数
                int val = 0;
                if (from_hex_to_i(s, i + 2, 4, val)) {
// 4 digits Unicode codes
// 创建一个长度为4的字符数组，用于存储Unicode编码
char buff[4];
// 将val转换为UTF-8编码，返回转换后的长度
size_t len = to_utf8(val, buff);
// 如果转换长度大于0，将转换后的内容追加到结果字符串中
if (len > 0) { result.append(buff, len); }
// 移动索引i到下一个Unicode编码的位置
i += 5; // 'u0000'
} else {
// 如果不是Unicode编码，直接将当前字符追加到结果字符串中
result += s[i];
}
} else {
// 如果不是%开头，表示是普通字符
int val = 0;
// 将%后面的两位十六进制数转换为整数
if (from_hex_to_i(s, i + 1, 2, val)) {
// 将转换后的整数转换为对应的字符，并追加到结果字符串中
result += static_cast<char>(val);
// 移动索引i到下一个十六进制编码的位置
i += 2; // '00'
} else {
// 如果无法转换为整数，直接将当前字符追加到结果字符串中
result += s[i];
}
}
} else if (convert_plus_to_space && s[i] == '+') {
// 如果需要将加号转换为空格，并且当前字符是加号，将空格追加到结果字符串中
result += ' ';
  } else {
    // 如果不是特殊字符，则将字符添加到结果字符串中
    result += s[i];
  }
}

// 从指定路径读取文件内容到字符串中
inline void read_file(const std::string &path, std::string &out) {
  // 以二进制方式打开文件流
  std::ifstream fs(path, std::ios_base::binary);
  // 定位到文件末尾，获取文件大小
  fs.seekg(0, std::ios_base::end);
  auto size = fs.tellg();
  // 重新定位到文件开头
  fs.seekg(0);
  // 调整输出字符串大小以容纳文件内容
  out.resize(static_cast<size_t>(size));
  // 从文件流中读取内容到输出字符串中
  fs.read(&out[0], static_cast<std::streamsize>(size));
}

// 获取文件路径的扩展名
inline std::string file_extension(const std::string &path) {
  std::smatch m;
  // 使用正则表达式匹配文件扩展名
  static auto re = std::regex("\\.([a-zA-Z0-9]+)$");
// 如果路径中匹配正则表达式，则返回匹配的子串
if (std::regex_search(path, m, re)) { return m[1].str(); }
// 如果没有匹配，则返回空字符串
return std::string();
}

// 判断字符是否为空格或制表符
inline bool is_space_or_tab(char c) { return c == ' ' || c == '\t'; }

// 去除字符串两端的空格或制表符，并返回去除后的起始和结束位置
inline std::pair<size_t, size_t> trim(const char *b, const char *e, size_t left,
                                      size_t right) {
  // 去除左端的空格或制表符
  while (b + left < e && is_space_or_tab(b[left])) {
    left++;
  }
  // 去除右端的空格或制表符
  while (right > 0 && is_space_or_tab(b[right - 1])) {
    right--;
  }
  // 返回去除空格后的起始和结束位置
  return std::make_pair(left, right);
}

// 复制去除两端空格后的字符串
inline std::string trim_copy(const std::string &s) {
  // 调用trim函数去除两端空格，并返回去除后的子串
  auto r = trim(s.data(), s.data() + s.size(), 0, s.size());
  return s.substr(r.first, r.second - r.first);
// 定义一个函数，用于将字符串根据指定的分隔符进行分割，并对每个分割后的子串执行指定的操作
inline void split(const char *b, const char *e, char d,
                  std::function<void(const char *, const char *)> fn) {
  size_t i = 0;  // 初始化索引 i 为 0
  size_t beg = 0;  // 初始化起始位置 beg 为 0

  // 遍历字符串，直到遇到结束符 e 或者字符串结束符 '\0'
  while (e ? (b + i < e) : (b[i] != '\0')) {
    // 如果当前字符是分隔符 d
    if (b[i] == d) {
      // 对当前子串进行修剪操作，得到去除空格后的子串范围
      auto r = trim(b, e, beg, i);
      // 如果修剪后的子串范围有效，则执行指定的操作
      if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
      // 更新起始位置为当前位置的下一个位置
      beg = i + 1;
    }
    // 移动到下一个字符
    i++;
  }

  // 如果字符串非空
  if (i) {
    // 对最后一个子串进行修剪操作，得到去除空格后的子串范围
    auto r = trim(b, e, beg, i);
    // 如果修剪后的子串范围有效，则执行指定的操作
    if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
  }
}
// stream_line_reader 类的构造函数，接受一个流对象和固定大小的缓冲区，用于读取流中的数据
inline stream_line_reader::stream_line_reader(Stream &strm, char *fixed_buffer,
                                              size_t fixed_buffer_size)
    : strm_(strm), fixed_buffer_(fixed_buffer),
      fixed_buffer_size_(fixed_buffer_size) {}

// 返回指向数据的指针，如果可增长缓冲区为空，则返回固定缓冲区的指针，否则返回可增长缓冲区的指针
inline const char *stream_line_reader::ptr() const {
  if (glowable_buffer_.empty()) {
    return fixed_buffer_;
  } else {
    return glowable_buffer_.data();
  }
}

// 返回数据的大小，如果可增长缓冲区为空，则返回固定缓冲区已使用的大小，否则返回可增长缓冲区的大小
inline size_t stream_line_reader::size() const {
  if (glowable_buffer_.empty()) {
    return fixed_buffer_used_size_;
  } else {
    return glowable_buffer_.size();
  }
}

// 检查当前缓冲区是否以回车换行结尾
inline bool stream_line_reader::end_with_crlf() const {
  auto end = ptr() + size();
  return size() >= 2 && end[-2] == '\r' && end[-1] == '\n';
}

// 从流中读取一行数据
inline bool stream_line_reader::getline() {
  fixed_buffer_used_size_ = 0;
  glowable_buffer_.clear();

  for (size_t i = 0;; i++) {
    char byte;
    auto n = strm_.read(&byte, 1);

    if (n < 0) {
      return false;
    } else if (n == 0) {
      if (i == 0) {
        return false; // 如果流已经结束，返回false
      } else {
        break; // 否则跳出循环
      }
    }

    append(byte); // 调用append函数，将读取的字节添加到缓冲区中

    if (byte == '\n') { break; } // 如果读取到换行符，跳出循环
  }

  return true; // 返回true，表示成功读取一行数据
}

inline void stream_line_reader::append(char c) {
  if (fixed_buffer_used_size_ < fixed_buffer_size_ - 1) { // 如果固定缓冲区未满
    fixed_buffer_[fixed_buffer_used_size_++] = c; // 将字符添加到固定缓冲区中
    fixed_buffer_[fixed_buffer_used_size_] = '\0'; // 在末尾添加字符串结束符
  } else { // 如果固定缓冲区已满
    if (glowable_buffer_.empty()) { // 如果可扩展缓冲区为空
# 断言固定缓冲区中已使用的大小处的字符为'\0'
assert(fixed_buffer_[fixed_buffer_used_size_] == '\0');
# 将固定缓冲区中已使用的大小处的字符之前的内容赋值给可增长缓冲区
glowable_buffer_.assign(fixed_buffer_, fixed_buffer_used_size_);
# 将字符c添加到可增长缓冲区的末尾
glowable_buffer_ += c;
# 关闭套接字
}

# 关闭套接字
inline int close_socket(socket_t sock) {
#ifdef _WIN32
  return closesocket(sock);
#else
  return close(sock);
#endif
}

# 处理由于系统调用被信号中断而产生的错误
template <typename T> inline ssize_t handle_EINTR(T fn) {
  ssize_t res = false;
  # 循环直到系统调用成功执行
  while (true) {
    # 调用传入的函数
    res = fn();
    # 如果返回值小于0且错误码为EINTR，则继续循环
    if (res < 0 && errno == EINTR) { continue; }
    // 如果条件满足，跳出循环
    break;
  }
  // 返回结果
  return res;
}

// 从套接字中读取数据
inline ssize_t read_socket(socket_t sock, void *ptr, size_t size, int flags) {
  // 处理中断，然后执行接收数据操作
  return handle_EINTR([&]() {
    return recv(sock,
#ifdef _WIN32
                static_cast<char *>(ptr), static_cast<int>(size),
#else
                ptr, size,
#endif
                flags);
  });
}

// 向套接字发送数据
inline ssize_t send_socket(socket_t sock, const void *ptr, size_t size,
                           int flags) {
  // 处理中断，然后执行发送数据操作
  return handle_EINTR([&]() {
    // 根据操作系统选择合适的 send 函数进行发送数据
    return send(sock,
#ifdef _WIN32
                static_cast<const char *>(ptr), static_cast<int>(size),
#else
                ptr, size,
#endif
                flags);
  });
}

// 在指定的时间内等待套接字可读，使用 poll 函数实现
inline ssize_t select_read(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建一个 pollfd 结构体，设置套接字和事件类型
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLIN;

  // 计算超时时间
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 使用 poll 函数等待套接字可读
  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 如果不是在 Windows 平台上运行，检查套接字是否超出范围
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 创建文件描述符集合并清空
  fd_set fds;
  FD_ZERO(&fds);
  // 将套接字加入到文件描述符集合中
  FD_SET(sock, &fds);

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 调用 select 函数，监视套接字是否可写
  return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), &fds, nullptr, nullptr, &tv);
  });
#endif
}

// 在 Windows 平台上使用 select 函数监视套接字是否可写
inline ssize_t select_write(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建一个用于读取操作的文件描述符结构体
  struct pollfd pfd_read;
  // 设置文件描述符为指定的套接字
  pfd_read.fd = sock;
  // 设置事件类型为可写
  pfd_read.events = POLLOUT;

  // 将秒和微秒转换为毫秒，并赋值给超时变量
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 使用 poll 函数进行阻塞式的 I/O 多路复用
  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 如果套接字超出了文件描述符的范围，返回 1
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 创建一个文件描述符集合
  fd_set fds;
  // 清空文件描述符集合
  FD_ZERO(&fds);
  // 将套接字加入文件描述符集合
  FD_SET(sock, &fds);

  // 设置超时时间
  timeval tv;
  // 设置秒
  tv.tv_sec = static_cast<long>(sec);
  // 设置微秒
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);
// 返回一个 lambda 函数，该函数在处理 EINTR 信号后调用 select 函数
return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), nullptr, &fds, nullptr, &tv);
  });
#endif
}

// 等待直到套接字准备就绪
inline Error wait_until_socket_is_ready(socket_t sock, time_t sec,
                                        time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建一个用于 poll 函数的结构体
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLIN | POLLOUT;

  // 计算超时时间
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 调用 poll 函数，处理 EINTR 信号
  auto poll_res = handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });

  // 如果超时，返回连接超时错误
  if (poll_res == 0) { return Error::ConnectionTimeout; }

  // 如果 poll 成功且套接字准备就绪，返回无错误
  if (poll_res > 0 && pfd_read.revents & (POLLIN | POLLOUT)) {
    // 定义一个整型变量error，并初始化为0
    int error = 0;
    // 定义一个socklen_t类型的变量len，并初始化为error的大小
    socklen_t len = sizeof(error);
    // 调用getsockopt函数获取socket选项的值，并将结果存储在res中
    auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                          reinterpret_cast<char *>(&error), &len);
    // 判断是否获取socket选项成功，并且error为0，如果是则返回Error::Success，否则返回Error::Connection
    auto successful = res >= 0 && !error;
    return successful ? Error::Success : Error::Connection;
  }

  // 如果不是在Windows平台下
  return Error::Connection;
#else
#ifndef _WIN32
  // 如果sock大于等于FD_SETSIZE，则返回Error::Connection
  if (sock >= FD_SETSIZE) { return Error::Connection; }
#endif

  // 定义一个文件描述符集合fdsr，并将其清空
  fd_set fdsr;
  FD_ZERO(&fdsr);
  // 将sock加入到fdsr中
  FD_SET(sock, &fdsr);

  // 复制fdsr到fdsw和fdse
  auto fdsw = fdsr;
  auto fdse = fdsr;
// 创建一个 timeval 结构体变量，用于设置超时时间
timeval tv;
// 设置秒数
tv.tv_sec = static_cast<long>(sec);
// 设置微秒数
tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

// 调用 handle_EINTR 函数处理可能被中断的系统调用 select
auto ret = handle_EINTR([&]() {
  return select(static_cast<int>(sock + 1), &fdsr, &fdsw, &fdse, &tv);
});

// 如果 select 返回 0，表示超时
if (ret == 0) { return Error::ConnectionTimeout; }

// 如果 select 返回大于 0，并且套接字可读或可写
if (ret > 0 && (FD_ISSET(sock, &fdsr) || FD_ISSET(sock, &fdsw))) {
  // 获取套接字错误信息
  int error = 0;
  socklen_t len = sizeof(error);
  auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                        reinterpret_cast<char *>(&error), &len);
  // 判断是否获取成功并且没有错误
  auto successful = res >= 0 && !error;
  return successful ? Error::Success : Error::Connection;
}
// 其他情况返回连接错误
return Error::Connection;
// 结束条件编译指令
#endif
}

// 检查套接字是否处于活动状态
inline bool is_socket_alive(socket_t sock) {
  // 使用 select 函数检查套接字是否可读
  const auto val = detail::select_read(sock, 0, 0);
  // 如果可读，返回 true
  if (val == 0) {
    return true;
  } 
  // 如果 select 出错并且错误码为 EBADF，说明套接字已关闭
  else if (val < 0 && errno == EBADF) {
    return false;
  }
  // 否则，尝试从套接字中读取一个字节，如果成功则返回 true
  char buf[1];
  return detail::read_socket(sock, &buf[0], sizeof(buf), MSG_PEEK) > 0;
}

// 套接字流类，继承自流类
class SocketStream : public Stream {
public:
  // 构造函数，设置读写超时时间
  SocketStream(socket_t sock, time_t read_timeout_sec, time_t read_timeout_usec,
               time_t write_timeout_sec, time_t write_timeout_usec);
  // 析构函数
  ~SocketStream() override;
// 检查是否可读
bool is_readable() const override;

// 检查是否可写
bool is_writable() const override;

// 从套接字中读取数据到指定的缓冲区
ssize_t read(char *ptr, size_t size) override;

// 将指定的数据写入套接字
ssize_t write(const char *ptr, size_t size) override;

// 获取远程 IP 地址和端口
void get_remote_ip_and_port(std::string &ip, int &port) const override;

// 获取本地 IP 地址和端口
void get_local_ip_and_port(std::string &ip, int &port) const override;

// 获取套接字
socket_t socket() const override;

// 套接字对象
private:
  socket_t sock_;

  // 读取超时时间（秒）
  time_t read_timeout_sec_;

  // 读取超时时间（微秒）
  time_t read_timeout_usec_;

  // 写入超时时间（秒）
  time_t write_timeout_sec_;

  // 写入超时时间（微秒）
  time_t write_timeout_usec_;

  // 读取缓冲区
  std::vector<char> read_buff_;

  // 读取缓冲区偏移量
  size_t read_buff_off_ = 0;

  // 读取缓冲区内容大小
  size_t read_buff_content_size_ = 0;

  // 读取缓冲区大小
  static const size_t read_buff_size_ = 1024 * 4;
// 如果支持 OpenSSL，则定义 SSLSocketStream 类，继承自 Stream 类
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
class SSLSocketStream : public Stream {
public:
  // 构造函数，初始化 SSL 套接字流
  SSLSocketStream(socket_t sock, SSL *ssl, time_t read_timeout_sec,
                  time_t read_timeout_usec, time_t write_timeout_sec,
                  time_t write_timeout_usec);
  // 析构函数，释放资源
  ~SSLSocketStream() override;

  // 判断套接字流是否可读
  bool is_readable() const override;
  // 判断套接字流是否可写
  bool is_writable() const override;
  // 从套接字流中读取数据
  ssize_t read(char *ptr, size_t size) override;
  // 向套接字流中写入数据
  ssize_t write(const char *ptr, size_t size) override;
  // 获取远程 IP 地址和端口
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  // 获取本地 IP 地址和端口
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  // 获取套接字
  socket_t socket() const override;

private:
  // 套接字
  socket_t sock_;
  // SSL 对象指针
  SSL *ssl_;
  // 读取超时时间（秒）
  time_t read_timeout_sec_;
  // 读取超时时间（微秒）
  time_t read_timeout_usec_;
  // 写入超时时间（秒）
  time_t write_timeout_sec_;
  // 写入超时时间（微秒）
  time_t write_timeout_usec_;
};
#endif

// 检查套接字是否保持连接
inline bool keep_alive(socket_t sock, time_t keep_alive_timeout_sec) {
  // 使用 std::chrono 命名空间
  using namespace std::chrono;
  // 获取当前时间点
  auto start = steady_clock::now();
  // 循环检查套接字是否保持连接
  while (true) {
    // 使用 select_read 函数检查套接字是否可读，超时时间为 10000 微秒
    auto val = select_read(sock, 0, 10000);
    // 如果 select_read 返回值小于 0，表示出错，返回 false
    if (val < 0) {
      return false;
    } 
    // 如果 select_read 返回值为 0，表示超时
    else if (val == 0) {
      // 获取当前时间点
      auto current = steady_clock::now();
      // 计算时间间隔
      auto duration = duration_cast<milliseconds>(current - start);
      // 计算超时时间
      auto timeout = keep_alive_timeout_sec * 1000;
      // 如果时间间隔超过超时时间，返回 false
      if (duration.count() > timeout) { return false; }
      // 使当前线程休眠1毫秒
      std::this_thread::sleep_for(std::chrono::milliseconds(1));
    } else {
      // 如果条件不满足，返回true
      return true;
    }
  }
}

template <typename T>
inline bool
process_server_socket_core(const std::atomic<socket_t> &svr_sock, socket_t sock,
                           size_t keep_alive_max_count,
                           time_t keep_alive_timeout_sec, T callback) {
  // 断言保持连接的最大次数大于0
  assert(keep_alive_max_count > 0);
  auto ret = false;
  auto count = keep_alive_max_count;
  // 当服务器套接字不是无效套接字且计数大于0且保持连接时
  while (svr_sock != INVALID_SOCKET && count > 0 &&
         keep_alive(sock, keep_alive_timeout_sec)) {
    // 判断是否是最后一次保持连接
    auto close_connection = count == 1;
    auto connection_closed = false;
    // 调用回调函数处理连接
    ret = callback(close_connection, connection_closed);
// 如果返回值为假或者连接已关闭，则跳出循环
if (!ret || connection_closed) { break; }
// 计数器减一
count--;
// 返回结果
}
return ret;
}

// 处理服务器套接字的模板函数，包括套接字、保持连接的最大次数、保持连接的超时时间、读取超时时间、写入超时时间和回调函数
template <typename T>
inline bool
process_server_socket(const std::atomic<socket_t> &svr_sock, socket_t sock,
                      size_t keep_alive_max_count,
                      time_t keep_alive_timeout_sec, time_t read_timeout_sec,
                      time_t read_timeout_usec, time_t write_timeout_sec,
                      time_t write_timeout_usec, T callback) {
  // 调用处理服务器套接字核心函数
  return process_server_socket_core(
      svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
      // 匿名函数，处理是否关闭连接和连接是否已关闭
      [&](bool close_connection, bool &connection_closed) {
        // 创建套接字流对象
        SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                          write_timeout_sec, write_timeout_usec);
        // 调用回调函数
        return callback(strm, close_connection, connection_closed);
      });
}
// 关闭客户端套接字的处理函数，设置读写超时时间，并执行回调函数
inline bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                                  time_t read_timeout_usec,
                                  time_t write_timeout_sec,
                                  time_t write_timeout_usec,
                                  std::function<bool(Stream &)> callback) {
  // 使用套接字创建 SocketStream 对象，设置读写超时时间
  SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                    write_timeout_sec, write_timeout_usec);
  // 执行回调函数，并返回结果
  return callback(strm);
}

// 关闭套接字的函数，根据操作系统不同采用不同的关闭方式
inline int shutdown_socket(socket_t sock) {
#ifdef _WIN32
  // Windows 系统下使用 SD_BOTH 关闭套接字
  return shutdown(sock, SD_BOTH);
#else
  // 非 Windows 系统下使用 SHUT_RDWR 关闭套接字
  return shutdown(sock, SHUT_RDWR);
#endif
}
// 创建一个套接字，根据参数不同可以用于绑定或连接
template <typename BindOrConnect>
socket_t create_socket(const std::string &host, const std::string &ip, int port,
                       int address_family, int socket_flags, bool tcp_nodelay,
                       SocketOptions socket_options,
                       BindOrConnect bind_or_connect) {
  // 获取地址信息
  const char *node = nullptr;
  struct addrinfo hints;
  struct addrinfo *result;

  // 清空hints结构体
  memset(&hints, 0, sizeof(struct addrinfo));
  // 设置套接字类型为流式套接字
  hints.ai_socktype = SOCK_STREAM;
  // 设置协议为默认
  hints.ai_protocol = 0;

  // 如果IP地址不为空
  if (!ip.empty()) {
    // 将IP地址转换为c字符串
    node = ip.c_str();
    // 请求getaddrinfo将IP地址转换为地址
    hints.ai_family = AF_UNSPEC;
    hints.ai_flags = AI_NUMERICHOST;
  } else {
    // 如果主机名不为空，则将其转换为 C 风格字符串
    if (!host.empty()) { node = host.c_str(); }
    // 设置地址族
    hints.ai_family = address_family;
    // 设置套接字标志
    hints.ai_flags = socket_flags;
  }

#ifndef _WIN32
  // 如果地址族为 AF_UNIX
  if (hints.ai_family == AF_UNIX) {
    // 获取主机名的长度
    const auto addrlen = host.length();
    // 如果长度超过 sockaddr_un 结构体中 sun_path 的长度，则返回无效套接字
    if (addrlen > sizeof(sockaddr_un::sun_path)) return INVALID_SOCKET;

    // 创建 AF_UNIX 类型的套接字
    auto sock = socket(hints.ai_family, hints.ai_socktype, hints.ai_protocol);
    // 如果套接字创建成功
    if (sock != INVALID_SOCKET) {
      // 初始化 sockaddr_un 结构体
      sockaddr_un addr{};
      addr.sun_family = AF_UNIX;
      // 将主机名拷贝到 sun_path 中
      std::copy(host.begin(), host.end(), addr.sun_path);

      // 设置地址信息
      hints.ai_addr = reinterpret_cast<sockaddr *>(&addr);
      // 设置地址信息的长度
      hints.ai_addrlen = static_cast<socklen_t>(
          sizeof(addr) - sizeof(addr.sun_path) + addrlen);
# 设置套接字的文件描述符标志，确保在执行 exec 系统调用时关闭套接字
fcntl(sock, F_SETFD, FD_CLOEXEC);
# 如果存在套接字选项，则调用 socket_options 函数设置套接字选项
if (socket_options) { socket_options(sock); }

# 如果套接字绑定或连接失败，则关闭套接字并将其置为无效套接字
if (!bind_or_connect(sock, hints)) {
  close_socket(sock);
  sock = INVALID_SOCKET;
}
# 返回套接字
return sock;
# 如果是在 Linux 平台且不是在 Android 平台上
# 则重新初始化解析器
auto service = std::to_string(port);
# 将端口号转换为字符串形式
if (getaddrinfo(node, service.c_str(), &hints, &result)) {
  return INVALID_SOCKET;
}
  for (auto rp = result; rp; rp = rp->ai_next) {
    // 遍历地址信息链表，创建一个 socket
#ifdef _WIN32
    // 在 Windows 平台下，使用 WSASocketW 函数创建 socket
    auto sock =
        WSASocketW(rp->ai_family, rp->ai_socktype, rp->ai_protocol, nullptr, 0,
                   WSA_FLAG_NO_HANDLE_INHERIT | WSA_FLAG_OVERLAPPED);
    /**
     * 由于 WSA_FLAG_NO_HANDLE_INHERIT 仅在 Windows 7 SP1 及以上版本支持，
     * 在旧版本的 Windows 系统上，使用该标志创建 socket 会失败。
     *
     * 在这种情况下，让我们尝试以旧的方式创建 socket。
     *
     * 参考链接：
     * https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasocketa
     *
     * WSA_FLAG_NO_HANDLE_INHERIT:
     * 该标志在 Windows 7 with SP1、Windows Server 2008 R2 with SP1 及更高版本上支持
     *
    // 如果套接字无效，则创建一个新的套接字
    if (sock == INVALID_SOCKET) {
      sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    }
    // 如果不是 Windows 系统，则创建一个新的套接字
    else
    auto sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    // 如果套接字无效，则继续下一次循环
    if (sock == INVALID_SOCKET) { continue; }
    // 如果不是 Windows 系统，设置套接字的文件描述符标志为FD_CLOEXEC
    #ifndef _WIN32
    if (fcntl(sock, F_SETFD, FD_CLOEXEC) == -1) {
      close_socket(sock);
      continue;
    }
    #endif
    // 如果启用了 TCP_NODELAY 选项，则设置套接字的 TCP_NODELAY 选项
    if (tcp_nodelay) {
      int yes = 1;
      setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, reinterpret_cast<char *>(&yes),
                 sizeof(yes));
    }
    }

    // 如果存在 socket_options，则调用 socket_options 函数设置 socket 选项
    if (socket_options) { socket_options(sock); }

    // 如果地址族为 AF_INET6，则设置 IPV6_V6ONLY 选项
    if (rp->ai_family == AF_INET6) {
      int no = 0;
      setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY, reinterpret_cast<char *>(&no),
                 sizeof(no));
    }

    // 绑定或连接 socket
    // 如果成功，则释放地址信息并返回 socket
    if (bind_or_connect(sock, *rp)) {
      freeaddrinfo(result);
      return sock;
    }

    // 关闭 socket
    close_socket(sock);
  }

  // 释放地址信息
  freeaddrinfo(result);
// 返回无效的套接字
return INVALID_SOCKET;
}

// 设置套接字为非阻塞模式
inline void set_nonblocking(socket_t sock, bool nonblocking) {
#ifdef _WIN32
  // 如果是 Windows 系统，设置套接字的非阻塞标志
  auto flags = nonblocking ? 1UL : 0UL;
  ioctlsocket(sock, FIONBIO, &flags);
#else
  // 如果是其他系统，获取当前套接字的标志
  auto flags = fcntl(sock, F_GETFL, 0);
  // 设置套接字的非阻塞标志
  fcntl(sock, F_SETFL,
        nonblocking ? (flags | O_NONBLOCK) : (flags & (~O_NONBLOCK)));
#endif
}

// 检查是否是连接错误
inline bool is_connection_error() {
#ifdef _WIN32
  // 如果是 Windows 系统，检查最近的套接字操作是否是阻塞错误
  return WSAGetLastError() != WSAEWOULDBLOCK;
#else
  // 如果是其他系统，检查最近的套接字操作是否是连接正在进行中的错误
  return errno != EINPROGRESS;
#endif
}
// 绑定 IP 地址到套接字
inline bool bind_ip_address(socket_t sock, const std::string &host) {
  // 初始化地址信息结构体和结果指针
  struct addrinfo hints;
  struct addrinfo *result;

  // 清空地址信息结构体并设置参数
  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_family = AF_UNSPEC; // 地址族为未指定
  hints.ai_socktype = SOCK_STREAM; // 套接字类型为流式套接字
  hints.ai_protocol = 0; // 协议为未指定

  // 获取主机地址信息
  if (getaddrinfo(host.c_str(), "0", &hints, &result)) { return false; }

  auto ret = false;
  // 遍历地址信息结果
  for (auto rp = result; rp; rp = rp->ai_next) {
    const auto &ai = *rp;
    // 尝试绑定套接字和地址信息
    if (!::bind(sock, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen))) {
      ret = true;
      break;
    }
  }

  // 释放地址信息
  freeaddrinfo(result);
  // 返回结果
  return ret;
}

// 如果不是在 Windows、Android 和 AIX 平台上，则定义 USE_IF2IP
#if !defined _WIN32 && !defined ANDROID && !defined _AIX
#define USE_IF2IP
#endif

// 如果定义了 USE_IF2IP，则定义 if2ip 函数
#ifdef USE_IF2IP
// 获取接口地址
inline std::string if2ip(int address_family, const std::string &ifn) {
  // 获取接口地址信息
  struct ifaddrs *ifap;
  getifaddrs(&ifap);
  // 候选地址字符串
  std::string addr_candidate;
  // 遍历接口地址信息
  for (auto ifa = ifap; ifa; ifa = ifa->ifa_next) {
    // 如果接口地址存在，并且接口名匹配，并且地址族匹配
    if (ifa->ifa_addr && ifn == ifa->ifa_name &&
        (AF_UNSPEC == address_family ||
         ifa->ifa_addr->sa_family == address_family)) {
      // 如果是 IPv4 地址
      if (ifa->ifa_addr->sa_family == AF_INET) {
        // 将 ifa->ifa_addr 强制转换为 struct sockaddr_in 类型的指针，并赋值给 sa
        auto sa = reinterpret_cast<struct sockaddr_in *>(ifa->ifa_addr);
        // 创建一个长度为 INET_ADDRSTRLEN 的字符数组 buf，用于存储转换后的 IP 地址
        char buf[INET_ADDRSTRLEN];
        // 将 IPv4 地址转换为可读的字符串形式，并存储在 buf 中
        if (inet_ntop(AF_INET, &sa->sin_addr, buf, INET_ADDRSTRLEN)) {
          // 释放 ifaddrs 结构体链表
          freeifaddrs(ifap);
          // 返回转换后的 IPv4 地址字符串
          return std::string(buf, INET_ADDRSTRLEN);
        }
      } else if (ifa->ifa_addr->sa_family == AF_INET6) {
        // 将 ifa->ifa_addr 强制转换为 struct sockaddr_in6 类型的指针，并赋值给 sa
        auto sa = reinterpret_cast<struct sockaddr_in6 *>(ifa->ifa_addr);
        // 检查是否为链路本地地址
        if (!IN6_IS_ADDR_LINKLOCAL(&sa->sin6_addr)) {
          // 创建一个长度为 INET6_ADDRSTRLEN 的字符数组 buf，用于存储转换后的 IPv6 地址
          char buf[INET6_ADDRSTRLEN] = {};
          // 将 IPv6 地址转换为可读的字符串形式，并存储在 buf 中
          if (inet_ntop(AF_INET6, &sa->sin6_addr, buf, INET6_ADDRSTRLEN)) {
            // 判断是否为唯一本地地址
            auto s6_addr_head = sa->sin6_addr.s6_addr[0];
            if (s6_addr_head == 0xfc || s6_addr_head == 0xfd) {
              // 将转换后的 IPv6 地址字符串存储在 addr_candidate 中
              addr_candidate = std::string(buf, INET6_ADDRSTRLEN);
            } else {
              // 释放 ifaddrs 结构体链表
              freeifaddrs(ifap);
              // 返回转换后的 IPv6 地址字符串
              return std::string(buf, INET6_ADDRSTRLEN);
            }
          }
// 结束 ifap 块
        }
      }
    }
  }
  // 释放 ifaddrs 结构
  freeifaddrs(ifap);
  // 返回地址候选
  return addr_candidate;
}
#endif

// 创建客户端套接字
inline socket_t create_client_socket(
    const std::string &host, const std::string &ip, int port,
    int address_family, bool tcp_nodelay, SocketOptions socket_options,
    time_t connection_timeout_sec, time_t connection_timeout_usec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    time_t write_timeout_usec, const std::string &intf, Error &error) {
  // 创建套接字
  auto sock = create_socket(
      host, ip, port, address_family, 0, tcp_nodelay, std::move(socket_options),
      // 匿名函数，用于设置套接字选项
      [&](socket_t sock2, struct addrinfo &ai) -> bool {
        // 如果接口不为空
        if (!intf.empty()) {
#ifdef USE_IF2IP
          // 从网络接口获取指定地址族的 IP 地址
          auto ip_from_if = if2ip(address_family, intf);
          // 如果获取的 IP 地址为空，则使用网络接口的名称作为 IP 地址
          if (ip_from_if.empty()) { ip_from_if = intf; }
          // 绑定 IP 地址到套接字
          if (!bind_ip_address(sock2, ip_from_if.c_str())) {
            // 如果绑定失败，设置错误类型并返回 false
            error = Error::BindIPAddress;
            return false;
          }
#endif
        }

        // 设置套接字为非阻塞模式
        set_nonblocking(sock2, true);

        // 尝试连接套接字到指定地址
        auto ret =
            ::connect(sock2, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen));

        // 如果连接失败
        if (ret < 0) {
          // 如果是连接错误，设置错误类型并返回 false
          if (is_connection_error()) {
            error = Error::Connection;
            return false;
          }
          // 等待套接字就绪，设置错误类型并返回 false
          error = wait_until_socket_is_ready(sock2, connection_timeout_sec,
// 设置连接超时时间
setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO, (char *)&connection_timeout_usec, sizeof(connection_timeout_usec));
// 如果设置失败，返回false
if (error != Error::Success) { return false; }

// 设置套接字为阻塞模式
set_nonblocking(sock2, false);

// 根据操作系统设置读取超时时间
#ifdef _WIN32
  // 如果是 Windows 系统，将毫秒级的超时时间转换为 uint32_t 类型，并设置 SO_RCVTIMEO 选项
  auto timeout = static_cast<uint32_t>(read_timeout_sec * 1000 + read_timeout_usec / 1000);
  setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO, (char *)&timeout, sizeof(timeout));
#else
  // 如果是其他系统，设置 timeval 结构体的秒和微秒，并设置 SO_RCVTIMEO 选项
  timeval tv;
  tv.tv_sec = static_cast<long>(read_timeout_sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec);
  setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO, (char *)&tv, sizeof(tv));
#endif
#ifdef _WIN32
          // 如果是在 Windows 平台下，将写超时时间转换为毫秒，并设置 SO_SNDTIMEO 选项
          auto timeout = static_cast<uint32_t>(write_timeout_sec * 1000 +
                                               write_timeout_usec / 1000);
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO, (char *)&timeout,
                     sizeof(timeout));
#else
          // 如果不是在 Windows 平台下，设置 timeval 结构体的秒和微秒，并设置 SO_SNDTIMEO 选项
          timeval tv;
          tv.tv_sec = static_cast<long>(write_timeout_sec);
          tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec);
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO, (char *)&tv, sizeof(tv));
#endif
        }

        // 设置错误状态为成功
        error = Error::Success;
        // 返回 true
        return true;
      });

  // 如果套接字不是无效的，设置错误状态为成功
  if (sock != INVALID_SOCKET) {
    error = Error::Success;
  } else {
    // 如果错误类型为成功，将错误类型设置为连接错误
    if (error == Error::Success) { error = Error::Connection; }
  }

  // 返回套接字
  return sock;
}

// 获取 IP 地址和端口号
inline bool get_ip_and_port(const struct sockaddr_storage &addr,
                            socklen_t addr_len, std::string &ip, int &port) {
  // 如果地址族为 IPv4
  if (addr.ss_family == AF_INET) {
    // 获取端口号
    port = ntohs(reinterpret_cast<const struct sockaddr_in *>(&addr)->sin_port);
  } else if (addr.ss_family == AF_INET6) {
    // 如果地址族为 IPv6，获取端口号
    port =
        ntohs(reinterpret_cast<const struct sockaddr_in6 *>(&addr)->sin6_port);
  } else {
    // 如果地址族既不是 IPv4 也不是 IPv6，返回 false
    return false;
  }

  // 用于存储 IP 地址的数组
  std::array<char, NI_MAXHOST> ipstr{};
  // 获取 IP 地址
  if (getnameinfo(reinterpret_cast<const struct sockaddr *>(&addr), addr_len,
  // 从给定的 socket_t 对象中获取本地 IP 地址和端口号
inline void get_local_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 创建一个 sockaddr_storage 结构体用于存储地址信息
  struct sockaddr_storage addr;
  // 获取地址信息的长度
  socklen_t addr_len = sizeof(addr);
  // 如果成功获取到本地地址信息
  if (!getsockname(sock, reinterpret_cast<struct sockaddr *>(&addr),
                   &addr_len)) {
    // 调用 get_ip_and_port 函数获取 IP 地址和端口号
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 从给定的 socket_t 对象中获取远程 IP 地址和端口号
inline void get_remote_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 创建一个 sockaddr_storage 结构体用于存储地址信息
  struct sockaddr_storage addr;
  // 获取地址结构体的长度
  socklen_t addr_len = sizeof(addr);

  // 获取对等端的地址信息，并判断是否成功
  if (!getpeername(sock, reinterpret_cast<struct sockaddr *>(&addr), &addr_len)) {
    // 如果是 UNIX 域套接字
#ifndef _WIN32
    if (addr.ss_family == AF_UNIX) {
      // 如果是 Linux 系统
#if defined(__linux__)
      // 获取对等端进程的用户凭证
      struct ucred ucred;
      socklen_t len = sizeof(ucred);
      if (getsockopt(sock, SOL_SOCKET, SO_PEERCRED, &ucred, &len) == 0) {
        // 获取对等端进程的 PID
        port = ucred.pid;
      }
      // 如果是苹果系统
#elif defined(SOL_LOCAL) && defined(SO_PEERPID) // __APPLE__
      // 获取对等端进程的 PID
      pid_t pid;
      socklen_t len = sizeof(pid);
      if (getsockopt(sock, SOL_LOCAL, SO_PEERPID, &pid, &len) == 0) {
        port = pid;
      }
#endif
      // 返回
      return;
    }
#endif
    // 从地址中获取 IP 和端口信息
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 将字符串转换为标签的核心函数
inline constexpr unsigned int str2tag_core(const char *s, size_t l,
                                           unsigned int h) {
  return (l == 0)
             ? h
             : str2tag_core(
                   s + 1, l - 1,
                   // 重置 h 的高 6 位，以避免溢出
                   (((std::numeric_limits<unsigned int>::max)() >> 6) &
                    h * 33) ^
                       static_cast<unsigned char>(*s));
}

// 将字符串转换为标签的函数
inline unsigned int str2tag(const std::string &s) {
  return str2tag_core(s.data(), s.size(), 0);
} // 结束命名空间 udl

namespace udl {

// 自定义字面量操作符，将字符串转换为无符号整数
inline constexpr unsigned int operator"" _t(const char *s, size_t l) {
  return str2tag_core(s, l, 0);
}

} // namespace udl

// 查找文件内容类型
inline const char *
find_content_type(const std::string &path,
                  const std::map<std::string, std::string> &user_data) {
  // 获取文件扩展名
  auto ext = file_extension(path);

  // 在用户数据中查找文件扩展名对应的内容类型
  auto it = user_data.find(ext);
  if (it != user_data.end()) { return it->second.c_str(); }

  // 使用自定义字面量操作符
  using udl::operator""_t;
# 根据文件扩展名转换为对应的 MIME 类型
switch (str2tag(ext)) {
  # 如果扩展名不在列表中，则返回空指针
  default: return nullptr;
  # 如果扩展名是 "css"，则返回 "text/css"
  case "css"_t: return "text/css";
  # 如果扩展名是 "csv"，则返回 "text/csv"
  case "csv"_t: return "text/csv";
  # 如果扩展名是 "htm" 或 "html"，则返回 "text/html"
  case "htm"_t:
  case "html"_t: return "text/html";
  # 如果扩展名是 "js" 或 "mjs"，则返回 "text/javascript"
  case "js"_t:
  case "mjs"_t: return "text/javascript";
  # 如果扩展名是 "txt"，则返回 "text/plain"
  case "txt"_t: return "text/plain";
  # 如果扩展名是 "vtt"，则返回 "text/vtt"
  case "vtt"_t: return "text/vtt";

  # 如果扩展名是 "apng"，则返回 "image/apng"
  case "apng"_t: return "image/apng";
  # 如果扩展名是 "avif"，则返回 "image/avif"
  case "avif"_t: return "image/avif";
  # 如果扩展名是 "bmp"，则返回 "image/bmp"
  case "bmp"_t: return "image/bmp";
  # 如果扩展名是 "gif"，则返回 "image/gif"
  case "gif"_t: return "image/gif";
  # 如果扩展名是 "png"，则返回 "image/png"
  case "png"_t: return "image/png";
  # 如果扩展名是 "svg"，则返回 "image/svg+xml"
  case "svg"_t: return "image/svg+xml";
  # 如果扩展名是 "webp"，则返回 "image/webp"
  case "webp"_t: return "image/webp";
  # 如果扩展名是 "ico"，则返回 "image/x-icon"
  case "ico"_t: return "image/x-icon";
  # 如果扩展名是 "tif"，则返回 "image/tiff"
  case "tif"_t: return "image/tiff";
}
  case "tiff"_t: return "image/tiff"; // 如果文件类型是 tiff，则返回 MIME 类型 image/tiff
  case "jpg"_t: // 如果文件类型是 jpg
  case "jpeg"_t: return "image/jpeg"; // 或者是 jpeg，则返回 MIME 类型 image/jpeg

  case "mp4"_t: return "video/mp4"; // 如果文件类型是 mp4，则返回 MIME 类型 video/mp4
  case "mpeg"_t: return "video/mpeg"; // 如果文件类型是 mpeg，则返回 MIME 类型 video/mpeg
  case "webm"_t: return "video/webm"; // 如果文件类型是 webm，则返回 MIME 类型 video/webm

  case "mp3"_t: return "audio/mp3"; // 如果文件类型是 mp3，则返回 MIME 类型 audio/mp3
  case "mpga"_t: return "audio/mpeg"; // 如果文件类型是 mpga，则返回 MIME 类型 audio/mpeg
  case "weba"_t: return "audio/webm"; // 如果文件类型是 weba，则返回 MIME 类型 audio/webm
  case "wav"_t: return "audio/wave"; // 如果文件类型是 wav，则返回 MIME 类型 audio/wave

  case "otf"_t: return "font/otf"; // 如果文件类型是 otf，则返回 MIME 类型 font/otf
  case "ttf"_t: return "font/ttf"; // 如果文件类型是 ttf，则返回 MIME 类型 font/ttf
  case "woff"_t: return "font/woff"; // 如果文件类型是 woff，则返回 MIME 类型 font/woff
  case "woff2"_t: return "font/woff2"; // 如果文件类型是 woff2，则返回 MIME 类型 font/woff2

  case "7z"_t: return "application/x-7z-compressed"; // 如果文件类型是 7z，则返回 MIME 类型 application/x-7z-compressed
  case "atom"_t: return "application/atom+xml"; // 如果文件类型是 atom，则返回 MIME 类型 application/atom+xml
// 根据文件类型返回对应的 MIME 类型
switch (file_type) {
  case "pdf"_t: return "application/pdf";
  case "json"_t: return "application/json";
  case "rss"_t: return "application/rss+xml";
  case "tar"_t: return "application/x-tar";
  case "xht"_t:
  case "xhtml"_t: return "application/xhtml+xml";
  case "xslt"_t: return "application/xslt+xml";
  case "xml"_t: return "application/xml";
  case "gz"_t: return "application/gzip";
  case "zip"_t: return "application/zip";
  case "wasm"_t: return "application/wasm";
}

// 根据状态码返回对应的状态消息
inline const char *status_message(int status) {
  switch (status) {
  case 100: return "Continue";
  case 101: return "Switching Protocol";
  case 102: return "Processing";
  case 103: return "Early Hints";
  case 200: return "OK"; // 返回状态码200对应的描述信息
  case 201: return "Created"; // 返回状态码201对应的描述信息
  case 202: return "Accepted"; // 返回状态码202对应的描述信息
  case 203: return "Non-Authoritative Information"; // 返回状态码203对应的描述信息
  case 204: return "No Content"; // 返回状态码204对应的描述信息
  case 205: return "Reset Content"; // 返回状态码205对应的描述信息
  case 206: return "Partial Content"; // 返回状态码206对应的描述信息
  case 207: return "Multi-Status"; // 返回状态码207对应的描述信息
  case 208: return "Already Reported"; // 返回状态码208对应的描述信息
  case 226: return "IM Used"; // 返回状态码226对应的描述信息
  case 300: return "Multiple Choice"; // 返回状态码300对应的描述信息
  case 301: return "Moved Permanently"; // 返回状态码301对应的描述信息
  case 302: return "Found"; // 返回状态码302对应的描述信息
  case 303: return "See Other"; // 返回状态码303对应的描述信息
  case 304: return "Not Modified"; // 返回状态码304对应的描述信息
  case 305: return "Use Proxy"; // 返回状态码305对应的描述信息
  case 306: return "unused"; // 返回状态码306对应的描述信息
  case 307: return "Temporary Redirect"; // 返回状态码307对应的描述信息
  case 308: return "Permanent Redirect"; // 返回状态码308对应的描述信息
  case 400: return "Bad Request"; // 返回状态码400对应的描述信息
  case 401: return "Unauthorized";  // 返回状态码401表示未经授权
  case 402: return "Payment Required";  // 返回状态码402表示需要付款
  case 403: return "Forbidden";  // 返回状态码403表示禁止访问
  case 404: return "Not Found";  // 返回状态码404表示未找到资源
  case 405: return "Method Not Allowed";  // 返回状态码405表示不允许使用该方法
  case 406: return "Not Acceptable";  // 返回状态码406表示不可接受的内容
  case 407: return "Proxy Authentication Required";  // 返回状态码407表示需要代理身份验证
  case 408: return "Request Timeout";  // 返回状态码408表示请求超时
  case 409: return "Conflict";  // 返回状态码409表示冲突
  case 410: return "Gone";  // 返回状态码410表示资源已经不再可用
  case 411: return "Length Required";  // 返回状态码411表示需要Content-Length头字段
  case 412: return "Precondition Failed";  // 返回状态码412表示先决条件失败
  case 413: return "Payload Too Large";  // 返回状态码413表示请求实体过大
  case 414: return "URI Too Long";  // 返回状态码414表示请求URI过长
  case 415: return "Unsupported Media Type";  // 返回状态码415表示不支持的媒体类型
  case 416: return "Range Not Satisfiable";  // 返回状态码416表示请求范围不符合要求
  case 417: return "Expectation Failed";  // 返回状态码417表示期望失败
  case 418: return "I'm a teapot";  // 返回状态码418表示我是一个茶壶
  case 421: return "Misdirected Request";  // 返回状态码421表示请求被误导
  case 422: return "Unprocessable Entity";  // 返回状态码422表示无法处理的实体
  case 423: return "Locked";  // 返回状态码 423 对应的描述信息
  case 424: return "Failed Dependency";  // 返回状态码 424 对应的描述信息
  case 425: return "Too Early";  // 返回状态码 425 对应的描述信息
  case 426: return "Upgrade Required";  // 返回状态码 426 对应的描述信息
  case 428: return "Precondition Required";  // 返回状态码 428 对应的描述信息
  case 429: return "Too Many Requests";  // 返回状态码 429 对应的描述信息
  case 431: return "Request Header Fields Too Large";  // 返回状态码 431 对应的描述信息
  case 451: return "Unavailable For Legal Reasons";  // 返回状态码 451 对应的描述信息
  case 501: return "Not Implemented";  // 返回状态码 501 对应的描述信息
  case 502: return "Bad Gateway";  // 返回状态码 502 对应的描述信息
  case 503: return "Service Unavailable";  // 返回状态码 503 对应的描述信息
  case 504: return "Gateway Timeout";  // 返回状态码 504 对应的描述信息
  case 505: return "HTTP Version Not Supported";  // 返回状态码 505 对应的描述信息
  case 506: return "Variant Also Negotiates";  // 返回状态码 506 对应的描述信息
  case 507: return "Insufficient Storage";  // 返回状态码 507 对应的描述信息
  case 508: return "Loop Detected";  // 返回状态码 508 对应的描述信息
  case 510: return "Not Extended";  // 返回状态码 510 对应的描述信息
  case 511: return "Network Authentication Required";  // 返回状态码 511 对应的描述信息

  default:  // 默认情况下
// 根据状态码返回对应的错误信息
case 500: return "Internal Server Error";
}

// 判断是否可以压缩指定的内容类型
inline bool can_compress_content_type(const std::string &content_type) {
  using udl::operator""_t; // 使用用户定义字面量

  auto tag = str2tag(content_type); // 将内容类型转换为标签

  switch (tag) { // 根据标签进行判断
  case "image/svg+xml"_t: // 如果是 SVG 图像类型
  case "application/javascript"_t: // 如果是 JavaScript 类型
  case "application/json"_t: // 如果是 JSON 类型
  case "application/xml"_t: // 如果是 XML 类型
  case "application/protobuf"_t: // 如果是 Protocol Buffers 类型
  case "application/xhtml+xml"_t: return true; // 如果是 XHTML 类型，则返回 true

  default:
    return !content_type.rfind("text/", 0) && tag != "text/event-stream"_t; // 如果不是以上类型，则判断是否以"text/"开头且不是"text/event-stream"类型
  }
}
// 返回请求和响应的编码类型
inline EncodingType encoding_type(const Request &req, const Response &res) {
  // 检查响应的内容类型是否可以压缩
  auto ret = detail::can_compress_content_type(res.get_header_value("Content-Type"));
  if (!ret) { return EncodingType::None; }

  // 获取请求中的 Accept-Encoding 头部信息
  const auto &s = req.get_header_value("Accept-Encoding");
  (void)(s);

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
  // 如果支持 Brotli 压缩，检查 Accept-Encoding 中是否包含 br
  // 如果包含，则返回 Brotli 编码类型
  ret = s.find("br") != std::string::npos;
  if (ret) { return EncodingType::Brotli; }
#endif

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持 Gzip 压缩，检查 Accept-Encoding 中是否包含 gzip
  // 如果包含，则返回 Gzip 编码类型
  ret = s.find("gzip") != std::string::npos;
  if (ret) { return EncodingType::Gzip; }
```

在这个示例中，代码是一个用于确定请求和响应的编码类型的函数。注释解释了每个部分的作用，包括检查响应内容类型是否可以压缩，获取请求中的 Accept-Encoding 头部信息，以及检查是否支持 Brotli 和 Gzip 压缩，并根据情况返回相应的编码类型。
#endif
// 结束条件编译指令

  return EncodingType::None;
}
// 返回编码类型为None

inline bool nocompressor::compress(const char *data, size_t data_length,
                                   bool /*last*/, Callback callback) {
// 压缩函数，接受数据和数据长度作为参数，以及一个回调函数
  if (!data_length) { return true; }
  // 如果数据长度为0，直接返回true
  return callback(data, data_length);
  // 调用回调函数对数据进行处理
}

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 如果支持zlib，则进行以下操作
inline gzip_compressor::gzip_compressor() {
// gzip压缩构造函数
  std::memset(&strm_, 0, sizeof(strm_));
  // 将strm_的内存初始化为0
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  is_valid_ = deflateInit2(&strm_, Z_DEFAULT_COMPRESSION, Z_DEFLATED, 31, 8,
                           Z_DEFAULT_STRATEGY) == Z_OK;
  // 使用deflateInit2函数初始化strm_，并将结果存储在is_valid_中
// 在析构函数中结束压缩流
inline gzip_compressor::~gzip_compressor() { deflateEnd(&strm_); }

// 压缩数据的函数，参数为数据指针、数据长度、是否为最后一块数据、回调函数
inline bool gzip_compressor::compress(const char *data, size_t data_length,
                                      bool last, Callback callback) {
  // 确保压缩流有效
  assert(is_valid_);

  do {
    // 设置可用输入数据的最大长度
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    // 将数据长度和最大可用输入数据长度中较小的值赋给可用输入数据长度
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    // 设置下一个输入数据的指针
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    // 减去已经处理的数据长度，更新数据指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    // 根据是否为最后一块数据和数据长度是否为0来确定压缩模式
    auto flush = (last && data_length == 0) ? Z_FINISH : Z_NO_FLUSH;
    # 定义一个整型变量 ret，用于存储压缩操作的返回值
    int ret = Z_OK;

    # 创建一个大小为 CPPHTTPLIB_COMPRESSION_BUFSIZ 的字符数组 buff，用于存储压缩后的数据
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    do {
      # 设置输出缓冲区的大小和指针
      strm_.avail_out = static_cast<uInt>(buff.size());
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      # 执行压缩操作
      ret = deflate(&strm_, flush);
      # 如果返回值为 Z_STREAM_ERROR，则表示压缩操作出错，返回 false
      if (ret == Z_STREAM_ERROR) { return false; }

      # 调用回调函数，处理压缩后的数据
      if (!callback(buff.data(), buff.size() - strm_.avail_out)) {
        return false;
      }
    } while (strm_.avail_out == 0);

    # 断言压缩操作是否成功完成
    assert((flush == Z_FINISH && ret == Z_STREAM_END) ||
           (flush == Z_NO_FLUSH && ret == Z_OK));
    # 断言输入缓冲区是否已经全部处理完
    assert(strm_.avail_in == 0);
  } while (data_length > 0);
// 返回 true，表示成功执行
  return true;
}

// gzip 解压缩器的构造函数
inline gzip_decompressor::gzip_decompressor() {
  // 将 strm_ 的内存清零
  std::memset(&strm_, 0, sizeof(strm_));
  // 设置内存分配函数和释放函数
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  // wbits 的值为 15，表示应该使用最大可能的值，以确保可以解码任何 gzip 流
  // 偏移量 32 指定应自动检测流类型，无论是 gzip 还是 deflate
  is_valid_ = inflateInit2(&strm_, 32 + 15) == Z_OK;
}

// gzip 解压缩器的析构函数
inline gzip_decompressor::~gzip_decompressor() { 
  // 结束解压缩器的操作
  inflateEnd(&strm_); 
}

// 返回解压缩器是否有效
inline bool gzip_decompressor::is_valid() const { return is_valid_; }
// 压缩解码器的解压函数，接受数据和数据长度作为输入，并调用回调函数处理解压后的数据
inline bool gzip_decompressor::decompress(const char *data, size_t data_length,
                                          Callback callback) {
  // 确保解压器有效
  assert(is_valid_);

  int ret = Z_OK;

  do {
    // 设置输入数据的最大长度
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    // 将输入数据长度限制在最大长度内，并设置解压器的输入数据和长度
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    // 调整输入数据长度和指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    // 创建缓冲区，用于存储解压后的数据
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    // 当输入数据还有剩余时，进行解压
    while (strm_.avail_in > 0) {
      // 设置输出缓冲区的大小
      strm_.avail_out = static_cast<uInt>(buff.size());
      // 将输出缓冲区指针设置为buff.data()的重新解释类型
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      // 保存当前输入缓冲区的可用字节数
      auto prev_avail_in = strm_.avail_in;

      // 调用inflate函数进行解压缩
      ret = inflate(&strm_, Z_NO_FLUSH);

      // 检查是否有新的输入数据被处理
      if (prev_avail_in - strm_.avail_in == 0) { return false; }

      // 断言，确保返回值不是Z_STREAM_ERROR
      assert(ret != Z_STREAM_ERROR);
      switch (ret) {
      // 处理不同的解压缩返回状态
      case Z_NEED_DICT:
      case Z_DATA_ERROR:
      case Z_MEM_ERROR: inflateEnd(&strm_); return false;
      }

      // 调用回调函数处理解压缩后的数据
      if (!callback(buff.data(), buff.size() - strm_.avail_out)) {
        return false;
      }
    }
    // 如果返回值不是 Z_OK 且不是 Z_STREAM_END，则返回 false
    if (ret != Z_OK && ret != Z_STREAM_END) return false;

  } while (data_length > 0);

  // 返回 true
  return true;
}
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
// 创建 Brotli 压缩器实例
inline brotli_compressor::brotli_compressor() {
  state_ = BrotliEncoderCreateInstance(nullptr, nullptr, nullptr);
}

// 销毁 Brotli 压缩器实例
inline brotli_compressor::~brotli_compressor() {
  BrotliEncoderDestroyInstance(state_);
}

// 压缩数据
inline bool brotli_compressor::compress(const char *data, size_t data_length,
                                        bool last, Callback callback) {
  // 创建缓冲区
  std::array<uint8_t, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
  // 设置操作类型，如果是最后一次操作则为结束，否则为处理
  auto operation = last ? BROTLI_OPERATION_FINISH : BROTLI_OPERATION_PROCESS;
  // 设置输入数据的长度
  auto available_in = data_length;
  // 设置输入数据的指针
  auto next_in = reinterpret_cast<const uint8_t *>(data);

  // 循环压缩数据流
  for (;;) {
    // 如果是最后一次操作
    if (last) {
      // 检查压缩器是否已经完成
      if (BrotliEncoderIsFinished(state_)) { break; }
    } else {
      // 如果输入数据已经处理完毕，则跳出循环
      if (!available_in) { break; }
    }

    // 设置输出数据的长度
    auto available_out = buff.size();
    // 设置输出数据的指针
    auto next_out = buff.data();

    // 压缩数据流
    if (!BrotliEncoderCompressStream(state_, operation, &available_in, &next_in,
                                     &available_out, &next_out, nullptr)) {
      return false;
    }
  }
// 计算输出字节数，如果大于0，则调用回调函数传递输出数据
auto output_bytes = buff.size() - available_out;
if (output_bytes) {
  callback(reinterpret_cast<const char *>(buff.data()), output_bytes);
}

// 返回解压缩是否成功的标志
return true;
}

// 构造函数，创建 Brotli 解压缩器实例
inline brotli_decompressor::brotli_decompressor() {
  decoder_s = BrotliDecoderCreateInstance(0, 0, 0);
  decoder_r = decoder_s ? BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT
                        : BROTLI_DECODER_RESULT_ERROR;
}

// 析构函数，销毁 Brotli 解压缩器实例
inline brotli_decompressor::~brotli_decompressor() {
  if (decoder_s) { BrotliDecoderDestroyInstance(decoder_s); }
}

// 检查解压缩器实例是否有效
inline bool brotli_decompressor::is_valid() const { return decoder_s; }
// 压缩函数，用于解压缩数据
inline bool brotli_decompressor::decompress(const char *data,
                                            size_t data_length,
                                            Callback callback) {
  // 如果解码结果为成功或错误，则直接返回
  if (decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
      decoder_r == BROTLI_DECODER_RESULT_ERROR) {
    return 0;
  }

  // 设置输入数据的指针和长度
  const uint8_t *next_in = (const uint8_t *)data;
  size_t avail_in = data_length;
  size_t total_out;

  // 设置解码结果为需要更多输出
  decoder_r = BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT;

  // 创建缓冲区用于存储解压缩后的数据
  std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
  // 循环直到解码结果不再需要更多输出
  while (decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT) {
    // 设置输出数据的指针和长度
    char *next_out = buff.data();
    size_t avail_out = buff.size();
// 使用 Brotli 解码器解压缩流数据，将解压缩后的数据存储在指定的缓冲区中
decoder_r = BrotliDecoderDecompressStream(
    decoder_s, &avail_in, &next_in, &avail_out,
    reinterpret_cast<uint8_t **>(&next_out), &total_out);

// 如果解码器返回错误结果，则返回 false
if (decoder_r == BROTLI_DECODER_RESULT_ERROR) { return false; }

// 如果回调函数返回 false，则返回 false
if (!callback(buff.data(), buff.size() - avail_out)) { return false; }

// 如果解码器结果为成功或需要更多输入，则返回 true，否则返回 false
return decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
       decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT;
}

// 检查给定的 headers 中是否包含指定的 key
inline bool has_header(const Headers &headers, const std::string &key) {
  return headers.find(key) != headers.end();
}

// 获取给定 headers 中指定 key 的值，并返回其指针
inline const char *get_header_value(const Headers &headers,
                                    const std::string &key, size_t id,
// 根据给定的键值在头部中查找对应的值，并返回指定索引的值，如果不存在则返回默认值
const char *get_header_value(const std::multimap<std::string, std::string> &headers, const std::string &key, size_t id, const char *def) {
  // 在头部中查找指定键的所有值的范围
  auto rng = headers.equal_range(key);
  // 获取指定索引的值
  auto it = rng.first;
  std::advance(it, static_cast<ssize_t>(id));
  // 如果找到对应值，则返回其C风格字符串形式
  if (it != rng.second) { return it->second.c_str(); }
  // 如果未找到对应值，则返回默认值
  return def;
}

// 忽略大小写比较两个字符串是否相等
inline bool compare_case_ignore(const std::string &a, const std::string &b) {
  // 如果两个字符串长度不相等，则直接返回false
  if (a.size() != b.size()) { return false; }
  // 逐个字符比较，忽略大小写
  for (size_t i = 0; i < b.size(); i++) {
    if (::tolower(a[i]) != ::tolower(b[i])) { return false; }
  }
  // 如果所有字符都相等，则返回true
  return true;
}

// 解析头部内容，并调用指定的函数进行处理
template <typename T>
inline bool parse_header(const char *beg, const char *end, T fn) {
  // 跳过末尾的空格和制表符
  while (beg < end && is_space_or_tab(end[-1])) {
  // 将 end 指针减一
  end--;
  // 定义指针 p 指向 beg
  auto p = beg;
  // 当 p 小于 end 并且 p 指向的字符不是冒号时，p 向后移动
  while (p < end && *p != ':') {
    p++;
  }
  // 如果 p 等于 end，则返回 false
  if (p == end) { return false; }
  // 定义 key_end 指向 p
  auto key_end = p;
  // 如果 *p 不是冒号，则返回 false
  if (*p++ != ':') { return false; }
  // 当 p 小于 end 并且 p 指向的字符是空格或制表符时，p 向后移动
  while (p < end && is_space_or_tab(*p)) {
    p++;
  }
  // 如果 p 小于 end，则定义 key 为从 beg 到 key_end 的字符串
  if (p < end) {
    auto key = std::string(beg, key_end);
    // 比较 key 是否与 "Location" 相同，如果是则直接取 p 到 end 之间的字符串，否则对 p 到 end 之间的字符串进行 URL 解码
    auto val = compare_case_ignore(key, "Location")
                   ? std::string(p, end)
                   : decode_url(std::string(p, end), false);
    // 调用 fn 函数，传入移动后的 key 和 val
    fn(std::move(key), std::move(val));
    // 返回 true
    return true;
  }

  // 如果循环结束仍未找到符合条件的行，则返回 false
  return false;
}

// 从流中读取头部信息
inline bool read_headers(Stream &strm, Headers &headers) {
  // 定义缓冲区大小
  const auto bufsiz = 2048;
  // 创建缓冲区
  char buf[bufsiz];
  // 创建流行读取器
  stream_line_reader line_reader(strm, buf, bufsiz);

  // 循环读取头部信息
  for (;;) {
    // 如果无法读取行，则返回 false
    if (!line_reader.getline()) { return false; }

    // 检查行是否以 CRLF 结尾
    auto line_terminator_len = 2;
    // 如果行以CRLF结尾，则表示头部结束
    if (line_reader.end_with_crlf()) {
      // 如果行大小为2，则表示空行，即头部结束
      if (line_reader.size() == 2) { break; }
#ifdef CPPHTTPLIB_ALLOW_LF_AS_LINE_TERMINATOR
    } else {
      // 如果行大小为1，则表示空行，即头部结束
      if (line_reader.size() == 1) { break; }
      line_terminator_len = 1;
    }
#else
    } else {
      continue; // 跳过无效的行
    }
#endif

    // 如果行大小超过最大头部长度，则返回false
    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }

    // 排除行终止符
    auto end = line_reader.ptr() + line_reader.size() - line_terminator_len;
// 解析请求头部，将解析出的键值对添加到headers中
parse_header(line_reader.ptr(), end,
             [&](std::string &&key, std::string &&val) {
               headers.emplace(std::move(key), std::move(val));
             });
}

// 读取指定长度的内容，并将读取的进度传递给progress函数，将内容传递给out函数
inline bool read_content_with_length(Stream &strm, uint64_t len,
                                     Progress progress,
                                     ContentReceiverWithProgress out) {
  // 创建缓冲区
  char buf[CPPHTTPLIB_RECV_BUFSIZ];

  uint64_t r = 0;
  // 循环读取内容，直到读取长度达到指定长度
  while (r < len) {
    auto read_len = static_cast<size_t>(len - r);
    // 从流中读取数据到缓冲区
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ));
    // 如果读取失败，返回false
    if (n <= 0) { return false; }
    // 如果输出缓冲区中的数据没有全部写入到流中，则返回false
    if (!out(buf, static_cast<size_t>(n), r, len)) { return false; }
    // 更新已经写入的数据长度
    r += static_cast<uint64_t>(n);

    // 如果有进度回调函数，则调用进度回调函数
    if (progress) {
      // 如果进度回调函数返回false，则返回false
      if (!progress(r, len)) { return false; }
    }
  }

  // 返回true表示成功
  return true;
}

// 跳过指定长度的内容
inline void skip_content_with_length(Stream &strm, uint64_t len) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ];
  uint64_t r = 0;
  // 当已读取的数据长度小于指定长度时，继续读取数据
  while (r < len) {
    // 计算剩余需要读取的数据长度
    auto read_len = static_cast<size_t>(len - r);
    // 从流中读取数据到缓冲区中
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ));
    // 如果读取的数据长度小于等于0，则返回
    if (n <= 0) { return; }
    // 更新已读取的数据长度
    r += static_cast<uint64_t>(n);
  }
// 从流中读取内容，但不知道内容的长度
inline bool read_content_without_length(Stream &strm,
                                        ContentReceiverWithProgress out) {
  // 创建一个缓冲区
  char buf[CPPHTTPLIB_RECV_BUFSIZ];
  // 初始化已读取的字节数
  uint64_t r = 0;
  // 循环读取流中的内容
  for (;;) {
    // 从流中读取数据到缓冲区
    auto n = strm.read(buf, CPPHTTPLIB_RECV_BUFSIZ);
    // 如果读取出错，返回false
    if (n < 0) {
      return false;
    } 
    // 如果已经读取完所有内容，返回true
    else if (n == 0) {
      return true;
    }

    // 将读取的数据传递给输出函数，并更新已读取的字节数
    if (!out(buf, static_cast<size_t>(n), r, 0)) { return false; }
    r += static_cast<uint64_t>(n);
  }

  // 返回true
  return true;
}
// 以分块方式读取内容，将结果传递给带有进度的内容接收器
template <typename T>
inline bool read_content_chunked(Stream &strm, T &x,
                                 ContentReceiverWithProgress out) {
  // 定义缓冲区大小
  const auto bufsiz = 16;
  // 创建缓冲区
  char buf[bufsiz];

  // 创建流行读取器，用于逐行读取数据
  stream_line_reader line_reader(strm, buf, bufsiz);

  // 如果无法读取行，则返回false
  if (!line_reader.getline()) { return false; }

  // 定义变量用于存储分块长度
  unsigned long chunk_len;
  // 循环读取分块数据
  while (true) {
    // 定义指针用于存储分块长度的结束位置
    char *end_ptr;

    // 将分块长度转换为无符号长整型
    chunk_len = std::strtoul(line_reader.ptr(), &end_ptr, 16);

    // 如果无法转换或者分块长度为最大值，则返回false
    if (end_ptr == line_reader.ptr()) { return false; }
    if (chunk_len == ULONG_MAX) { return false; }
    // 如果块长度为0，则跳出循环
    if (chunk_len == 0) { break; }

    // 根据指定长度读取内容，如果失败则返回false
    if (!read_content_with_length(strm, chunk_len, nullptr, out)) {
      return false;
    }

    // 如果无法读取一行，则返回false
    if (!line_reader.getline()) { return false; }

    // 如果读取的行与"\r\n"不相等，则返回false
    if (strcmp(line_reader.ptr(), "\r\n")) { return false; }

    // 如果无法读取一行，则返回false
    if (!line_reader.getline()) { return false; }
  }

  // 断言块长度为0
  assert(chunk_len == 0);

  // Trailer
  // 如果无法读取一行，则返回false
  if (!line_reader.getline()) { return false; }

  // 当读取的行与"\r\n"不相等时，执行循环
  while (strcmp(line_reader.ptr(), "\r\n")) {
    // 如果行的长度超过最大长度限制，则返回false
    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }
// 排除行终止符
constexpr auto line_terminator_len = 2;
auto end = line_reader.ptr() + line_reader.size() - line_terminator_len;

// 解析头部信息，将键值对添加到 x.headers 中
parse_header(line_reader.ptr(), end,
             [&](std::string &&key, std::string &&val) {
               x.headers.emplace(std::move(key), std::move(val));
             });

// 如果无法读取行，则返回 false
if (!line_reader.getline()) { return false; }
}

// 返回 true
return true;
}

// 检查是否使用分块传输编码
inline bool is_chunked_transfer_encoding(const Headers &headers) {
  return !strcasecmp(get_header_value(headers, "Transfer-Encoding", 0, ""),
                     "chunked");
}
// 准备内容接收器，根据需要解压缩内容
template <typename T, typename U>
bool prepare_content_receiver(T &x, int &status,
                              ContentReceiverWithProgress receiver,
                              bool decompress, U callback) {
  // 如果需要解压缩
  if (decompress) {
    // 获取内容编码
    std::string encoding = x.get_header_value("Content-Encoding");
    // 创建解压缩器对象
    std::unique_ptr<decompressor> decompressor;

    // 如果内容编码为 gzip 或 deflate
    if (encoding == "gzip" || encoding == "deflate") {
      // 如果支持 zlib，则创建 gzip 解压缩器
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
      decompressor = detail::make_unique<gzip_decompressor>();
      // 如果不支持 zlib，则返回不支持的状态码
#else
      status = 415;
      return false;
#endif
    } 
    // 如果内容编码包含 br
    else if (encoding.find("br") != std::string::npos) {
      // 如果支持 brotli，则创建 brotli 解压缩器
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
      decompressor = detail::make_unique<brotli_decompressor>();
      // 如果不支持 brotli，则返回不支持的状态码
#else
      status = 415;
      // 设置状态码为415
      return false;
      // 返回false
#endif
    }

    if (decompressor) {
      // 如果存在解压器
      if (decompressor->is_valid()) {
        // 如果解压器有效
        ContentReceiverWithProgress out = [&](const char *buf, size_t n,
                                              uint64_t off, uint64_t len) {
          // 创建一个接收内容和进度的回调函数
          return decompressor->decompress(buf, n,
                                          [&](const char *buf2, size_t n2) {
                                            return receiver(buf2, n2, off, len);
                                          });
        };
        // 调用回调函数，并返回结果
        return callback(std::move(out));
      } else {
        // 如果解压器无效，设置状态码为500
        status = 500;
        // 返回false
        return false;
      }
    }
  }

  // 定义一个带有进度的内容接收器，用于接收数据并调用回调函数
  ContentReceiverWithProgress out = [&](const char *buf, size_t n, uint64_t off,
                                        uint64_t len) {
    return receiver(buf, n, off, len);
  };
  // 调用回调函数，并返回结果
  return callback(std::move(out));
}

// 读取内容的模板函数，用于从流中读取数据并进行处理
template <typename T>
bool read_content(Stream &strm, T &x, size_t payload_max_length, int &status,
                  Progress progress, ContentReceiverWithProgress receiver,
                  bool decompress) {
  // 准备内容接收器，用于接收数据并进行处理
  return prepare_content_receiver(
      x, status, std::move(receiver), decompress,
      [&](const ContentReceiverWithProgress &out) {
        auto ret = true;
        auto exceed_payload_max_length = false;

        // 检查是否使用分块传输编码
        if (is_chunked_transfer_encoding(x.headers)) {
// 根据流和输出对象读取分块内容
ret = read_content_chunked(strm, x, out);
// 如果没有 Content-Length 头部信息，则读取没有长度的内容
} else if (!has_header(x.headers, "Content-Length")) {
  ret = read_content_without_length(strm, out);
} else {
  // 获取 Content-Length 头部信息的值
  auto len = get_header_value<uint64_t>(x.headers, "Content-Length");
  // 如果长度超过最大长度限制，则跳过内容并返回 false
  if (len > payload_max_length) {
    exceed_payload_max_length = true;
    skip_content_with_length(strm, len);
    ret = false;
  } else if (len > 0) {
    // 如果长度合法，则读取指定长度的内容
    ret = read_content_with_length(strm, len, std::move(progress), out);
  }
}

// 如果读取内容失败，则根据是否超过最大长度限制设置状态码
if (!ret) { status = exceed_payload_max_length ? 413 : 400; }
// 返回读取结果
return ret;
}); // namespace detail

// 写入头部信息到流中
inline ssize_t write_headers(Stream &strm, const Headers &headers) {
// 初始化写入长度为0
ssize_t write_len = 0;
// 遍历headers，将每个键值对格式化为字符串并写入流中
for (const auto &x : headers) {
  auto len = strm.write_format("%s: %s\r\n", x.first.c_str(), x.second.c_str());
  // 如果写入长度小于0，返回错误
  if (len < 0) { return len; }
  // 累加写入长度
  write_len += len;
}
// 写入空行
auto len = strm.write("\r\n");
// 如果写入长度小于0，返回错误
if (len < 0) { return len; }
// 累加写入长度
write_len += len;
// 返回总的写入长度
return write_len;
}

// 写入数据到流中
inline bool write_data(Stream &strm, const char *d, size_t l) {
  // 初始化偏移量为0
  size_t offset = 0;
  // 循环写入数据，直到所有数据都被写入
  while (offset < l) {
    // 写入数据到流中，并获取写入长度
    auto length = strm.write(d + offset, l - offset);
    // 如果写入长度小于0，返回错误
    if (length < 0) { return false; }
    // 更新偏移量
    offset += static_cast<size_t>(length);
  }
// 返回 true
  return true;
}

// 写入内容到流中
template <typename T>
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length, T is_shutting_down,
                          Error &error) {
  // 计算结束偏移量
  size_t end_offset = offset + length;
  // 初始化标志位
  auto ok = true;
  // 创建数据接收器
  DataSink data_sink;

  // 写入数据的回调函数
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果标志位为真
    if (ok) {
      // 如果流可写，并且成功写入数据
      if (strm.is_writable() && write_data(strm, d, l)) {
        // 更新偏移量
        offset += l;
      } else {
        // 标志位置为假
        ok = false;
      }
    }
    // 返回标志位
    return ok;
  };

  // 当偏移量小于结束偏移量并且系统没有关闭时执行循环
  while (offset < end_offset && !is_shutting_down()) {
    // 如果流不可写，则返回错误
    if (!strm.is_writable()) {
      error = Error::Write;
      return false;
    } 
    // 如果内容提供者返回false，则返回取消错误
    else if (!content_provider(offset, end_offset - offset, data_sink)) {
      error = Error::Canceled;
      return false;
    } 
    // 如果出现其他错误，则返回写入错误
    else if (!ok) {
      error = Error::Write;
      return false;
    }
  }

  // 循环结束后，设置错误为成功
  error = Error::Success;
  // 返回true
  return true;
}

// 模板函数定义
template <typename T>
// 写入内容到流中，使用给定的内容提供者和偏移量、长度，同时检查是否正在关闭
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length,
                          const T &is_shutting_down) {
  auto error = Error::Success;
  // 调用重载的write_content函数，传入流、内容提供者、偏移量、长度、是否正在关闭的标志和错误信息
  return write_content(strm, content_provider, offset, length, is_shutting_down,
                       error);
}

// 写入内容到流中，使用给定的内容提供者和是否正在关闭的标志，不指定长度
template <typename T>
inline bool
write_content_without_length(Stream &strm,
                             const ContentProvider &content_provider,
                             const T &is_shutting_down) {
  // 初始化偏移量为0，数据是否可用标志为true，操作是否成功标志为true
  size_t offset = 0;
  auto data_available = true;
  auto ok = true;
  DataSink data_sink;

  // 数据写入函数，接受数据和长度，返回是否成功的标志
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果操作成功标志为true
    if (ok) {
      offset += l; // 将偏移量增加l，l为数据长度
      if (!strm.is_writable() || !write_data(strm, d, l)) { ok = false; } // 如果流不可写或者写入数据失败，则将ok标记为false
    }
    return ok; // 返回ok标记，表示数据写入是否成功
  };

  data_sink.done = [&](void) { data_available = false; }; // 当数据传输完成时，将data_available标记为false

  while (data_available && !is_shutting_down()) { // 当数据可用且未关闭时
    if (!strm.is_writable()) { // 如果流不可写
      return false; // 返回false
    } else if (!content_provider(offset, 0, data_sink)) { // 如果内容提供者无法提供数据
      return false; // 返回false
    } else if (!ok) { // 如果数据写入失败
      return false; // 返回false
    }
  }
  return true; // 返回true，表示数据传输成功
}
// 写入内容的分块函数，接受流、内容提供者、关闭标志、压缩器和错误对象作为参数
template <typename T, typename U>
inline bool
write_content_chunked(Stream &strm, const ContentProvider &content_provider,
                      const T &is_shutting_down, U &compressor, Error &error) {
  // 偏移量初始化为0
  size_t offset = 0;
  // 数据是否可用的标志初始化为true
  auto data_available = true;
  // 操作是否成功的标志初始化为true
  auto ok = true;
  // 数据接收器对象
  DataSink data_sink;

  // 数据接收器的写入函数
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果操作成功
    if (ok) {
      // 检查数据是否可用
      data_available = l > 0;
      // 更新偏移量
      offset += l;

      // 初始化负载字符串
      std::string payload;
      // 如果压缩器成功压缩数据
      if (compressor.compress(d, l, false,
                              [&](const char *data, size_t data_len) {
                                // 将压缩后的数据追加到负载字符串
                                payload.append(data, data_len);
                                return true;
                              })) {
        // 如果 payload 不为空
        if (!payload.empty()) {
          // 为每个数据块发出分块响应头和尾部
          auto chunk =
              from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
          // 如果流不可写或者写入数据失败，则将 ok 置为 false
          if (!strm.is_writable() ||
              !write_data(strm, chunk.data(), chunk.size())) {
            ok = false;
          }
        }
        // 如果 payload 为空
        else {
          ok = false;
        }
      }
    }
    // 返回 ok
    return ok;
  };

  // trailer 处理完成后的回调函数
  auto done_with_trailer = [&](const Headers *trailer) {
    // 如果 ok 为 false，则直接返回
    if (!ok) { return; }
    // 将 data_available 置为 false
    data_available = false;
// 声明一个字符串变量 payload
std::string payload;
// 使用压缩器对空指针进行压缩，将压缩后的数据追加到 payload 中
if (!compressor.compress(nullptr, 0, true,
                         [&](const char *data, size_t data_len) {
                           payload.append(data, data_len);
                           return true;
                         })) {
  // 如果压缩失败，将 ok 置为 false 并返回
  ok = false;
  return;
}

// 如果 payload 不为空
if (!payload.empty()) {
  // 生成分块传输编码的响应头和尾部，将每个分块发送给 strm
  auto chunk = from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
  // 如果 strm 不可写或者写入数据失败，将 ok 置为 false 并返回
  if (!strm.is_writable() ||
      !write_data(strm, chunk.data(), chunk.size())) {
    ok = false;
    return;
  }
}
// 创建一个静态的字符串常量作为标记，用于标识操作完成
static const std::string done_marker("0\r\n");
// 将标记写入流中，如果写入失败则将 ok 置为 false
if (!write_data(strm, done_marker.data(), done_marker.size())) {
  ok = false;
}

// Trailer
// 如果存在 trailer，则遍历 trailer 中的键值对
if (trailer) {
  for (const auto &kv : *trailer) {
    // 拼接键值对成为字段行
    std::string field_line = kv.first + ": " + kv.second + "\r\n";
    // 将字段行写入流中，如果写入失败则将 ok 置为 false
    if (!write_data(strm, field_line.data(), field_line.size())) {
      ok = false;
    }
  }
}

// 创建一个静态的字符串常量作为回车换行符
static const std::string crlf("\r\n");
// 将回车换行符写入流中，如果写入失败则将 ok 置为 false
if (!write_data(strm, crlf.data(), crlf.size())) { ok = false; }
  // 设置一个回调函数，当数据传输完成时调用 done_with_trailer 函数
  data_sink.done = [&](void) { done_with_trailer(nullptr); };

  // 设置一个回调函数，当数据传输完成并有 trailer 时调用 done_with_trailer 函数
  data_sink.done_with_trailer = [&](const Headers &trailer) {
    done_with_trailer(&trailer);
  };

  // 当数据可用且未关闭时，进行数据传输
  while (data_available && !is_shutting_down()) {
    // 如果流不可写，设置错误并返回
    if (!strm.is_writable()) {
      error = Error::Write;
      return false;
    } 
    // 如果内容提供者无法提供数据，设置错误并返回
    else if (!content_provider(offset, 0, data_sink)) {
      error = Error::Canceled;
      return false;
    } 
    // 如果出现其他错误，设置错误并返回
    else if (!ok) {
      error = Error::Write;
      return false;
    }
  }

  // 数据传输成功，设置错误为成功
  error = Error::Success;
// 返回 true
}

// 使用分块传输编写内容
template <typename T, typename U>
inline bool write_content_chunked(Stream &strm,
                                  const ContentProvider &content_provider,
                                  const T &is_shutting_down, U &compressor) {
  auto error = Error::Success;
  // 调用重载的 write_content_chunked 函数
  return write_content_chunked(strm, content_provider, is_shutting_down,
                               compressor, error);
}

// 重定向请求
template <typename T>
inline bool redirect(T &cli, Request &req, Response &res,
                     const std::string &path, const std::string &location,
                     Error &error) {
  // 创建新的请求对象
  Request new_req = req;
  // 设置新请求的路径
  new_req.path = path;
  // 减少重定向次数
  new_req.redirect_count_ -= 1;
  # 如果响应状态码为303，并且请求方法不是GET或HEAD，则修改请求方法为GET，清空请求体和请求头部
  if (res.status == 303 && (req.method != "GET" && req.method != "HEAD")) {
    new_req.method = "GET";
    new_req.body.clear();
    new_req.headers.clear();
  }

  # 创建新的响应对象
  Response new_res;

  # 发送新的请求，并将结果保存在新的响应对象中
  auto ret = cli.send(new_req, new_res, error);
  if (ret) {
    # 如果发送成功，则更新请求和响应对象，并设置重定向地址
    req = new_req;
    res = new_res;
    res.location = location;
  }
  # 返回发送结果
  return ret;
}

# 将参数转换为查询字符串
inline std::string params_to_query_str(const Params &params) {
  std::string query;
  // 遍历参数列表，构建查询字符串
  for (auto it = params.begin(); it != params.end(); ++it) {
    // 如果不是第一个参数，则在参数之间添加 &
    if (it != params.begin()) { query += "&"; }
    // 添加参数名
    query += it->first;
    query += "=";
    // 添加经过编码的参数值
    query += encode_query_param(it->second);
  }
  // 返回构建好的查询字符串
  return query;
}

// 解析查询字符串，将其转换为参数列表
inline void parse_query_text(const std::string &s, Params &params) {
  // 创建一个缓存集合，用于存储已经解析过的键值对
  std::set<std::string> cache;
  // 使用分隔符 & 对查询字符串进行分割，并对每个键值对进行处理
  split(s.data(), s.data() + s.size(), '&', [&](const char *b, const char *e) {
    // 将分割出的键值对转换为字符串
    std::string kv(b, e);
    // 如果缓存中已经存在该键值对，则跳过
    if (cache.find(kv) != cache.end()) { return; }
    // 将键值对添加到缓存中
    cache.insert(kv);

    // 分割键值对，获取键和值
    std::string key;
    std::string val;
    split(b, e, '=', [&](const char *b2, const char *e2) {
      // 如果键为空，则将当前分割出的字符串作为键
      if (key.empty()) {
// 如果 key 不为空，则将 key 和 value 添加到 params 中
if (!key.empty()) {
  params.emplace(decode_url(key, true), decode_url(val, true));
}

// 解析 multipart/form-data 的 Content-Type 中的 boundary 参数
inline bool parse_multipart_boundary(const std::string &content_type, std::string &boundary) {
  // 查找 Content-Type 中的 boundary 参数
  auto boundary_keyword = "boundary=";
  auto pos = content_type.find(boundary_keyword);
  // 如果找不到 boundary 参数，则返回 false
  if (pos == std::string::npos) { return false; }
  // 查找 boundary 参数值的结束位置
  auto end = content_type.find(';', pos);
  // 获取 boundary 参数值的起始位置
  auto beg = pos + strlen(boundary_keyword);
  // 截取 boundary 参数值
  boundary = content_type.substr(beg, end - beg);
}
  // 如果边界字符串长度大于等于2，并且第一个字符和最后一个字符都是双引号，则去掉双引号
  if (boundary.length() >= 2 && boundary.front() == '"' &&
      boundary.back() == '"') {
    boundary = boundary.substr(1, boundary.size() - 2);
  }
  // 返回边界字符串是否为空
  return !boundary.empty();
}

#ifdef CPPHTTPLIB_NO_EXCEPTIONS
// 如果不支持异常处理，则使用此函数解析范围头部
inline bool parse_range_header(const std::string &s, Ranges &ranges) {
#else
// 如果支持异常处理，则使用此函数解析范围头部
inline bool parse_range_header(const std::string &s, Ranges &ranges) try {
#endif
  // 定义正则表达式来匹配范围头部的格式
  static auto re_first_range = std::regex(R"(bytes=(\d*-\d*(?:,\s*\d*-\d*)*))");
  std::smatch m;
  // 如果字符串匹配范围头部的格式
  if (std::regex_match(s, m, re_first_range)) {
    // 获取匹配的位置和长度
    auto pos = static_cast<size_t>(m.position(1));
    auto len = static_cast<size_t>(m.length(1));
    bool all_valid_ranges = true;
    // 分割范围字符串，对每个范围进行处理
    split(&s[pos], &s[pos + len], ',', [&](const char *b, const char *e) {
      // 如果不是所有范围都有效，则直接返回
      if (!all_valid_ranges) return;
// 创建一个静态的正则表达式对象，用于匹配形如 "数字-数字" 的字符串
static auto re_another_range = std::regex(R"(\s*(\d*)-(\d*))");
// 创建一个匹配结果对象
std::cmatch cm;
// 如果字符串匹配成功，则执行以下操作
if (std::regex_match(b, e, cm, re_another_range)) {
  // 初始化第一个数字的值为-1
  ssize_t first = -1;
  // 如果第一个数字不为空，则将字符串转换为整数赋值给first
  if (!cm.str(1).empty()) {
    first = static_cast<ssize_t>(std::stoll(cm.str(1)));
  }
  // 初始化第二个数字的值为-1
  ssize_t last = -1;
  // 如果第二个数字不为空，则将字符串转换为整数赋值给last
  if (!cm.str(2).empty()) {
    last = static_cast<ssize_t>(std::stoll(cm.str(2)));
  }
  // 如果第一个数字和第二个数字都不为-1，并且第一个数字大于第二个数字，则设置all_valid_ranges为false并返回
  if (first != -1 && last != -1 && first > last) {
    all_valid_ranges = false;
    return;
  }
  // 将第一个数字和第二个数字组成的pair加入到ranges中
  ranges.emplace_back(std::make_pair(first, last));
}
    return all_valid_ranges;
  }
  // 如果不满足条件，返回 false
  return false;
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
}
// 如果 CPPHTTPLIB_NO_EXCEPTIONS 宏被定义，执行下面的代码
#else
} catch (...) { return false; }
#endif

// 定义一个名为 MultipartFormDataParser 的类
class MultipartFormDataParser {
public:
  // 默认构造函数
  MultipartFormDataParser() = default;

  // 设置边界
  void set_boundary(std::string &&boundary) {
    boundary_ = boundary;
    dash_boundary_crlf_ = dash_ + boundary_ + crlf_;
    crlf_dash_boundary_ = crlf_ + dash_ + boundary_;
  }

  // 检查是否有效
  bool is_valid() const { return is_valid_; }
// 解析传入的数据，将内容和头部信息分别传递给 content_callback 和 header_callback
bool parse(const char *buf, size_t n, const ContentReceiver &content_callback,
           const MultipartContentHeader &header_callback) {

  // TODO: support 'filename*'
  // 定义正则表达式，用于匹配 Content-Disposition 头部信息
  static const std::regex re_content_disposition(
      R"~(^Content-Disposition:\s*form-data;\s*name="(.*?)"(?:;\s*filename="(.*?)")?(?:;\s*filename\*=\S+)?\s*$)~",
      std::regex_constants::icase);

  // 将传入的数据追加到缓冲区
  buf_append(buf, n);

  // 循环处理缓冲区中的数据
  while (buf_size() > 0) {
    switch (state_) {
    case 0: { // Initial boundary
      // 删除缓冲区中的初始边界
      buf_erase(buf_find(dash_boundary_crlf_));
      // 如果初始边界的长度大于缓冲区的长度，返回 true
      if (dash_boundary_crlf_.size() > buf_size()) { return true; }
      // 如果缓冲区不以初始边界开头，返回 false
      if (!buf_start_with(dash_boundary_crlf_)) { return false; }
      // 删除缓冲区中的初始边界
      buf_erase(dash_boundary_crlf_.size());
      // 设置状态为 1
      state_ = 1;
      break;
      }
      case 1: { // New entry
        // 清除文件信息
        clear_file_info();
        // 设置状态为2，表示正在处理头部信息
        state_ = 2;
        break;
      }
      case 2: { // Headers
        // 查找换行符的位置
        auto pos = buf_find(crlf_);
        // 如果位置超过最大头部长度，返回false
        if (pos > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }
        // 当位置小于缓冲区大小时
        while (pos < buf_size()) {
          // 空行
          if (pos == 0) {
            // 如果头部回调函数返回false，设置is_valid_为false，返回false
            if (!header_callback(file_)) {
              is_valid_ = false;
              return false;
            }
            // 删除换行符的大小，设置状态为3，表示正在处理正文
            buf_erase(crlf_.size());
            state_ = 3;
            break;
          }
// 定义常量字符串，表示要匹配的头部名称
static const std::string header_name = "content-type:";
// 获取当前位置开始的头部内容
const auto header = buf_head(pos);
// 如果头部内容以不区分大小写的方式以指定的头部名称开头，则提取内容类型并去除空格
if (start_with_case_ignore(header, header_name)) {
  file_.content_type = trim_copy(header.substr(header_name.size()));
} else {
  // 如果不是指定的头部名称，则尝试匹配内容描述头部的正则表达式
  std::smatch m;
  if (std::regex_match(header, m, re_content_disposition)) {
    // 如果匹配成功，提取文件名和文件名
    file_.name = m[1];
    file_.filename = m[2];
  } else {
    // 如果都不匹配，则标记为无效并返回false
    is_valid_ = false;
    return false;
  }
}
// 删除已处理的头部内容，并更新位置
buf_erase(pos + crlf_.size());
pos = buf_find(crlf_);
// 如果状态不等于3，则返回true
if (state_ != 3) { return true; }
// 否则跳出循环
break;
      }
      case 3: { // Body
        // 如果缓冲区大小大于分隔符的大小，则返回true
        if (crlf_dash_boundary_.size() > buf_size()) { return true; }
        // 在缓冲区中查找分隔符的位置
        auto pos = buf_find(crlf_dash_boundary_);
        // 如果找到分隔符
        if (pos < buf_size()) {
          // 调用内容回调函数处理缓冲区中的数据
          if (!content_callback(buf_data(), pos)) {
            is_valid_ = false;
            return false;
          }
          // 删除已处理的数据，并将状态设置为4
          buf_erase(pos + crlf_dash_boundary_.size());
          state_ = 4;
        } else {
          // 如果未找到分隔符
          auto len = buf_size() - crlf_dash_boundary_.size();
          // 如果缓冲区中还有数据
          if (len > 0) {
            // 调用内容回调函数处理缓冲区中的数据
            if (!content_callback(buf_data(), len)) {
              is_valid_ = false;
              return false;
            }
            // 删除已处理的数据
            buf_erase(len);
          }
        return true; // 返回 true，表示解析成功
        }
        break; // 结束当前 case 分支
      }
      case 4: { // Boundary
        if (crlf_.size() > buf_size()) { return true; } // 如果换行符的长度大于缓冲区大小，返回 true
        if (buf_start_with(crlf_)) { // 如果缓冲区以换行符开头
          buf_erase(crlf_.size()); // 删除缓冲区中的换行符
          state_ = 1; // 设置状态为 1
        } else {
          if (dash_crlf_.size() > buf_size()) { return true; } // 如果破折号加换行符的长度大于缓冲区大小，返回 true
          if (buf_start_with(dash_crlf_)) { // 如果缓冲区以破折号加换行符开头
            buf_erase(dash_crlf_.size()); // 删除缓冲区中的破折号加换行符
            is_valid_ = true; // 设置为有效
            buf_erase(buf_size()); // 删除缓冲区中的所有内容，移除尾声
          } else {
            return true; // 返回 true，表示解析失败
          }
        }
        break; // 结束当前 case 分支
  }
  }
}

return true;
}

// 清空文件信息
private:
void clear_file_info() {
  file_.name.clear(); // 清空文件名
  file_.filename.clear(); // 清空文件名
  file_.content_type.clear(); // 清空内容类型
}

// 检查字符串是否以指定字符串开头（不区分大小写）
bool start_with_case_ignore(const std::string &a,
                            const std::string &b) const {
  if (a.size() < b.size()) { return false; } // 如果a的长度小于b的长度，返回false
  for (size_t i = 0; i < b.size(); i++) { // 遍历字符串b
    if (::tolower(a[i]) != ::tolower(b[i])) { return false; } // 如果a和b在相同位置的字符（忽略大小写）不相等，返回false
  }
    // 返回 true
    return true;
  }

  // 定义常量字符串
  const std::string dash_ = "--";
  const std::string crlf_ = "\r\n";
  const std::string dash_crlf_ = "--\r\n";
  // 定义字符串变量
  std::string boundary_;
  std::string dash_boundary_crlf_;
  std::string crlf_dash_boundary_;

  // 定义状态和有效性变量
  size_t state_ = 0;
  bool is_valid_ = false;
  MultipartFormData file_;

  // 缓冲区
  // 检查字符串 a 是否以字符串 b 开头
  bool start_with(const std::string &a, size_t spos, size_t epos,
                  const std::string &b) const {
    // 如果结束位置减去开始位置小于字符串 b 的长度，则返回 false
    if (epos - spos < b.size()) { return false; }
    // 遍历字符串 b，逐个比较字符
    for (size_t i = 0; i < b.size(); i++) {
      if (a[i + spos] != b[i]) { return false; }
    }
  }
  // 返回 true
  return true;
}

// 返回缓冲区大小
size_t buf_size() const { return buf_epos_ - buf_spos_; }

// 返回缓冲区数据的指针
const char *buf_data() const { return &buf_[buf_spos_]; }

// 返回缓冲区前 l 个字符组成的字符串
std::string buf_head(size_t l) const { return buf_.substr(buf_spos_, l); }

// 检查缓冲区是否以字符串 s 开头
bool buf_start_with(const std::string &s) const {
  return start_with(buf_, buf_spos_, buf_epos_, s);
}

// 在缓冲区中查找字符串 s 的位置
size_t buf_find(const std::string &s) const {
  auto c = s.front();

  size_t off = buf_spos_;
  while (off < buf_epos_) {
    auto pos = off;
      // 进入循环，直到条件不满足为止
      while (true) {
        // 如果当前位置等于缓冲区结束位置，返回缓冲区大小
        if (pos == buf_epos_) { return buf_size(); }
        // 如果当前位置的字符等于给定字符，跳出循环
        if (buf_[pos] == c) { break; }
        // 移动到下一个位置
        pos++;
      }

      // 计算剩余大小
      auto remaining_size = buf_epos_ - pos;
      // 如果给定字符串大小大于剩余大小，返回缓冲区大小
      if (s.size() > remaining_size) { return buf_size(); }

      // 如果缓冲区从当前位置开始的子串与给定字符串匹配，返回匹配的起始位置
      if (start_with(buf_, pos, buf_epos_, s)) { return pos - buf_spos_; }

      // 更新偏移量
      off = pos + 1;
    }

    // 返回缓冲区大小
    return buf_size();
  }

  // 向缓冲区追加数据
  void buf_append(const char *data, size_t n) {
    // 计算剩余大小
    auto remaining_size = buf_size();
    // 如果剩余大小大于0且缓冲区起始位置大于0
    if (remaining_size > 0 && buf_spos_ > 0) {
  // 从 buf_ 的 buf_spos_ 位置开始，将剩余的数据向前移动，覆盖之前的数据
  for (size_t i = 0; i < remaining_size; i++) {
    buf_[i] = buf_[buf_spos_ + i];
  }
}
// 重置 buf_spos_ 为 0，表示 buf_ 的起始位置
buf_spos_ = 0;
// 设置 buf_epos_ 为 remaining_size，表示 buf_ 的结束位置
buf_epos_ = remaining_size;

// 如果剩余的数据大小加上新数据大小超过了 buf_ 的大小，就扩展 buf_ 的大小
if (remaining_size + n > buf_.size()) { buf_.resize(remaining_size + n); }

// 将新数据 data 复制到 buf_ 的结束位置 buf_epos_ 开始的位置
for (size_t i = 0; i < n; i++) {
  buf_[buf_epos_ + i] = data[i];
}
// 更新 buf_epos_ 的位置
buf_epos_ += n;
}

// 从 buf_ 中删除指定大小的数据
void buf_erase(size_t size) { buf_spos_ += size; }

// 存储数据的缓冲区
std::string buf_;
// 缓冲区中数据的起始位置
size_t buf_spos_ = 0;
// 缓冲区中数据的结束位置
size_t buf_epos_ = 0;
// 将输入的字符范围转换为小写字符串
inline std::string to_lower(const char *beg, const char *end) {
  std::string out;
  auto it = beg;
  while (it != end) {
    out += static_cast<char>(::tolower(*it)); // 将字符转换为小写并添加到输出字符串
    it++;
  }
  return out; // 返回转换后的小写字符串
}

// 生成一个用于multipart数据的边界字符串
inline std::string make_multipart_data_boundary() {
  static const char data[] =
      "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

  // 使用随机设备生成种子
  std::random_device seed_gen;
  // 请求128位的熵进行初始化
  std::seed_seq seed_sequence{seed_gen(), seed_gen(), seed_gen(), seed_gen()};
  // 使用熵序列初始化伪随机数生成器
  std::mt19937 engine(seed_sequence);

  // 初始化结果字符串
  std::string result = "--cpp-httplib-multipart-data-";

  // 生成16个随机字符，拼接到结果字符串中
  for (auto i = 0; i < 16; i++) {
    result += data[engine() % (sizeof(data) - 1)];
  }

  // 返回结果字符串
  return result;
}

// 检查多部分边界字符串是否有效
inline bool is_multipart_boundary_chars_valid(const std::string &boundary) {
  // 初始化有效标志为真
  auto valid = true;
  // 遍历边界字符串的每个字符
  for (size_t i = 0; i < boundary.size(); i++) {
    // 获取当前字符
    auto c = boundary[i];
    // 如果当前字符不是字母、数字、破折号或下划线，则将有效标志设为假
    if (!std::isalnum(c) && c != '-' && c != '_') {
      valid = false;
// 结束当前循环，跳出循环体
      break;
    }
  }
  // 返回 valid 变量的值
  return valid;
}

// 开始序列化多部分表单数据项
template <typename T>
inline std::string
serialize_multipart_formdata_item_begin(const T &item,
                                        const std::string &boundary) {
  // 创建一个字符串，以 boundary 作为分隔符
  std::string body = "--" + boundary + "\r\n";
  // 添加表单数据项的 Content-Disposition
  body += "Content-Disposition: form-data; name=\"" + item.name + "\"";
  // 如果文件名不为空，添加文件名
  if (!item.filename.empty()) {
    body += "; filename=\"" + item.filename + "\"";
  }
  body += "\r\n";
  // 如果内容类型不为空，添加内容类型
  if (!item.content_type.empty()) {
    body += "Content-Type: " + item.content_type + "\r\n";
  }
  // 添加空行
  body += "\r\n";
// 返回一个字符串变量 body
  return body;
}

// 返回一个字符串变量，表示多部分表单数据项的结束
inline std::string serialize_multipart_formdata_item_end() { return "\r\n"; }

// 返回一个字符串变量，表示多部分表单数据的结束
inline std::string
serialize_multipart_formdata_finish(const std::string &boundary) {
  return "--" + boundary + "--\r\n";
}

// 返回一个字符串变量，表示多部分表单数据的内容类型
inline std::string
serialize_multipart_formdata_get_content_type(const std::string &boundary) {
  return "multipart/form-data; boundary=" + boundary;
}

// 返回一个字符串变量，表示序列化后的多部分表单数据
inline std::string
serialize_multipart_formdata(const MultipartFormDataItems &items,
                             const std::string &boundary, bool finish = true) {
  // 创建一个空字符串变量 body
  std::string body;
// 遍历 items 容器中的每个元素，将每个元素的序列化形式添加到 body 中
for (const auto &item : items) {
    body += serialize_multipart_formdata_item_begin(item, boundary); // 将每个元素的起始序列化形式添加到 body 中
    body += item.content + serialize_multipart_formdata_item_end(); // 将每个元素的内容和结束序列化形式添加到 body 中
}

// 如果 finish 为真，则将多部分表单数据的结束序列化形式添加到 body 中
if (finish) body += serialize_multipart_formdata_finish(boundary);

// 返回序列化后的多部分表单数据
return body;
}

// 获取请求中指定范围的偏移量和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, size_t content_length,
                            size_t index) {
  auto r = req.ranges[index]; // 获取请求中指定索引的范围

  // 如果范围的起始和结束都为-1，则返回整个内容的偏移量和长度
  if (r.first == -1 && r.second == -1) {
    return std::make_pair(0, content_length);
  }
// 将 content_length 转换为 ssize_t 类型
auto slen = static_cast<ssize_t>(content_length);

// 如果 r.first 为 -1，则设置其值为 0 或者 slen - r.second 中的较大值，设置 r.second 为 slen - 1
if (r.first == -1) {
  r.first = (std::max)(static_cast<ssize_t>(0), slen - r.second);
  r.second = slen - 1;
}

// 如果 r.second 为 -1，则设置其值为 slen - 1
if (r.second == -1) { r.second = slen - 1; }

// 返回一个 pair 对象，包含 r.first 和 r.second - r.first + 1 的值
return std::make_pair(r.first, static_cast<size_t>(r.second - r.first) + 1);
}

// 创建 Content-Range 头字段的字符串表示，包括 offset, length 和 content_length
inline std::string make_content_range_header_field(size_t offset, size_t length,
                                                   size_t content_length) {
  std::string field = "bytes ";
  field += std::to_string(offset);
  field += "-";
  field += std::to_string(offset + length - 1);
  field += "/";
  field += std::to_string(content_length);
  return field;
}
// 处理多部分范围数据的函数
template <typename SToken, typename CToken, typename Content>
bool process_multipart_ranges_data(const Request &req, Response &res,
                                   const std::string &boundary,
                                   const std::string &content_type,
                                   SToken stoken, CToken ctoken,
                                   Content content) {
  // 遍历请求中的范围
  for (size_t i = 0; i < req.ranges.size(); i++) {
    // 添加分隔符
    ctoken("--");
    stoken(boundary);
    ctoken("\r\n");
    // 如果内容类型不为空，添加内容类型
    if (!content_type.empty()) {
      ctoken("Content-Type: ");
      stoken(content_type);
      ctoken("\r\n");
    }
    // 获取范围的偏移和长度
    auto offsets = get_range_offset_and_length(req, res.body.size(), i);
    auto offset = offsets.first;
    // 获取偏移量
    auto length = offsets.second;

    // 添加内容范围头字段
    ctoken("Content-Range: ");
    stoken(make_content_range_header_field(offset, length, res.body.size()));
    ctoken("\r\n");
    ctoken("\r\n");

    // 如果内容不符合要求，则返回false
    if (!content(offset, length)) { return false; }
    ctoken("\r\n");
  }

  // 添加多部分范围数据结束标记
  ctoken("--");
  stoken(boundary);
  ctoken("--\r\n");

  // 返回true
  return true;
}

// 生成多部分范围数据
inline bool make_multipart_ranges_data(const Request &req, Response &res,
                                       const std::string &boundary,
                                       const std::string &content_type,
// 处理多部分范围数据，将数据添加到传入的字符串引用中
inline size_t process_multipart_ranges_data(
    const Request &req, Response &res, const std::string &boundary,
    const std::string &content_type, const std::function<void(const std::string &)> &on_data,
    const std::function<void(const std::string &)> &on_file_data,
    const std::function<bool(size_t, size_t)> &on_range_data) {
  // 在请求和响应中处理多部分范围数据
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { data += token; },  // 处理数据部分
      [&](const std::string &token) { data += token; },  // 处理文件数据部分
      [&](size_t offset, size_t length) {  // 处理范围数据部分
        if (offset < res.body.size()) {
          data += res.body.substr(offset, length);
          return true;
        }
        return false;
      });
}

// 获取多部分范围数据的长度
inline size_t get_multipart_ranges_data_length(const Request &req, Response &res,
                                               const std::string &boundary,
                                               const std::string &content_type) {
  size_t data_length = 0;
// 处理多部分范围数据，根据不同的token类型更新数据长度
process_multipart_ranges_data(
    req, res, boundary, content_type,
    [&](const std::string &token) { data_length += token.size(); }, // 匿名函数，处理token为string类型时更新数据长度
    [&](const std::string &token) { data_length += token.size(); }, // 匿名函数，处理token为string类型时更新数据长度
    [&](size_t /*offset*/, size_t length) { // 匿名函数，处理偏移量和长度，更新数据长度
      data_length += length;
      return true;
    });

// 返回数据长度
return data_length;
}

// 写入多部分范围数据
template <typename T>
inline bool write_multipart_ranges_data(Stream &strm, const Request &req,
                                        Response &res,
                                        const std::string &boundary,
                                        const std::string &content_type,
                                        const T &is_shutting_down) {
  // 调用处理多部分范围数据的函数
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
// 使用lambda表达式将token写入流
      [&](const std::string &token) { strm.write(token); },
// 使用lambda表达式将token写入流
      [&](const std::string &token) { strm.write(token); },
// 使用lambda表达式写入内容到流
      [&](size_t offset, size_t length) {
        return write_content(strm, res.content_provider_, offset, length,
                             is_shutting_down);
      });
}

// 获取请求范围的偏移量和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, const Response &res,
                            size_t index) {
  auto r = req.ranges[index];

  // 如果范围的第二个值为-1，则设置为内容长度减1
  if (r.second == -1) {
    r.second = static_cast<ssize_t>(res.content_length_) - 1;
  }

  // 返回范围的偏移量和长度
  return std::make_pair(r.first, r.second - r.first + 1);
}
// 检查请求方法是否为 POST、PUT、PATCH、PRI 或 DELETE，如果是则返回 true，否则返回 false
inline bool expect_content(const Request &req) {
  if (req.method == "POST" || req.method == "PUT" || req.method == "PATCH" ||
      req.method == "PRI" || req.method == "DELETE") {
    return true;
  }
  // TODO: check if Content-Length is set
  return false;
}

// 检查字符串中是否包含回车换行符，如果包含则返回 true，否则返回 false
inline bool has_crlf(const std::string &s) {
  auto p = s.c_str();
  while (*p) {
    if (*p == '\r' || *p == '\n') { return true; }
    p++;
  }
  return false;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 计算字符串的消息摘要，使用指定的算法
inline std::string message_digest(const std::string &s, const EVP_MD *algo) {
  // 创建一个自动释放的 EVP_MD_CTX 上下文对象，使用自定义的释放函数
  auto context = std::unique_ptr<EVP_MD_CTX, decltype(&EVP_MD_CTX_free)>(
      EVP_MD_CTX_new(), EVP_MD_CTX_free);

  // 哈希值的长度
  unsigned int hash_length = 0;
  // 哈希值的字节数组
  unsigned char hash[EVP_MAX_MD_SIZE];

  // 初始化哈希算法上下文
  EVP_DigestInit_ex(context.get(), algo, nullptr);
  // 更新哈希算法上下文的数据
  EVP_DigestUpdate(context.get(), s.c_str(), s.size());
  // 完成哈希计算，获取哈希值
  EVP_DigestFinal_ex(context.get(), hash, &hash_length);

  // 将哈希值转换为十六进制字符串
  std::stringstream ss;
  for (auto i = 0u; i < hash_length; ++i) {
    ss << std::hex << std::setw(2) << std::setfill('0')
       << (unsigned int)hash[i];
  }

  // 返回哈希值的十六进制字符串表示
  return ss.str();
}

// 计算输入字符串的 MD5 哈希值
inline std::string MD5(const std::string &s) {
// 返回输入字符串的 MD5 消息摘要
inline std::string MD5(const std::string &s) {
  return message_digest(s, EVP_md5());
}

// 返回输入字符串的 SHA-256 消息摘要
inline std::string SHA_256(const std::string &s) {
  return message_digest(s, EVP_sha256());
}

// 返回输入字符串的 SHA-512 消息摘要
inline std::string SHA_512(const std::string &s) {
  return message_digest(s, EVP_sha512());
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef _WIN32
// 注意：此代码来源于以下 stackoverflow 帖子：
// https://stackoverflow.com/questions/9507184/can-openssl-on-windows-use-the-system-certificate-store
// 在 Windows 上加载系统证书
inline bool load_system_certs_on_windows(X509_STORE *store) {
  // 打开 ROOT 系统存储区域
  auto hStore = CertOpenSystemStoreW((HCRYPTPROV_LEGACY)NULL, L"ROOT");
  // 如果打开失败，返回 false
  if (!hStore) { return false; }
// 初始化结果为假
auto result = false;
// 初始化证书上下文为空
PCCERT_CONTEXT pContext = NULL;
// 遍历证书存储中的证书，直到遍历完所有证书
while ((pContext = CertEnumCertificatesInStore(hStore, pContext)) !=
       nullptr) {
  // 获取证书的编码数据
  auto encoded_cert =
      static_cast<const unsigned char *>(pContext->pbCertEncoded);
  // 将编码数据解析成 X509 证书对象
  auto x509 = d2i_X509(NULL, &encoded_cert, pContext->cbCertEncoded);
  // 如果成功解析成 X509 证书对象
  if (x509) {
    // 将证书添加到证书存储中
    X509_STORE_add_cert(store, x509);
    // 释放 X509 证书对象
    X509_free(x509);
    // 设置结果为真
    result = true;
  }
}
// 释放证书上下文
CertFreeCertificateContext(pContext);
// 关闭证书存储
CertCloseStore(hStore, 0);
// 返回结果
return result;
// 如果定义了CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN并且是苹果系统，则使用苹果系统的证书
#if defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
// 如果目标操作系统是OSX，则使用CFObjectPtr模板
#if TARGET_OS_OSX
// 定义一个CFObjectPtr模板，用于管理Core Foundation对象的内存
template <typename T>
using CFObjectPtr =
    std::unique_ptr<typename std::remove_pointer<T>::type, void (*)(CFTypeRef)>;

// 定义一个CF对象指针的删除器，用于释放Core Foundation对象的内存
inline void cf_object_ptr_deleter(CFTypeRef obj) {
  if (obj) { CFRelease(obj); }
}

// 从钥匙串中检索证书
inline bool retrieve_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 定义查询的键和值
  CFStringRef keys[] = {kSecClass, kSecMatchLimit, kSecReturnRef};
  CFTypeRef values[] = {kSecClassCertificate, kSecMatchLimitAll,
                        kCFBooleanTrue};

  // 创建一个CF字典对象，用于查询
  CFObjectPtr<CFDictionaryRef> query(
      CFDictionaryCreate(nullptr, reinterpret_cast<const void **>(keys), values,
                         sizeof(keys) / sizeof(keys[0]),
                         &kCFTypeDictionaryKeyCallBacks,
                         &kCFTypeDictionaryValueCallBacks),
// 使用自定义的删除器来释放 cf_object_ptr_deleter 指向的对象
cf_object_ptr_deleter);

// 如果 query 为空，则返回 false
if (!query) { return false; }

// 定义 security_items 为 CFTypeRef 类型的空指针
CFTypeRef security_items = nullptr;
// 如果 SecItemCopyMatching 函数调用不成功，或者 security_items 不是 CFArray 类型，则返回 false
if (SecItemCopyMatching(query.get(), &security_items) != errSecSuccess ||
    CFArrayGetTypeID() != CFGetTypeID(security_items)) {
  return false;
}

// 将 security_items 转换为 CFArrayRef 类型，并赋值给 certs
certs.reset(reinterpret_cast<CFArrayRef>(security_items));
// 返回 true
return true;
}

// 从钥匙串中检索根证书
inline bool retrieve_root_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 定义 root_security_items 为 CFArrayRef 类型的空指针
  CFArrayRef root_security_items = nullptr;
  // 如果 SecTrustCopyAnchorCertificates 函数调用不成功，则返回 false
  if (SecTrustCopyAnchorCertificates(&root_security_items) != errSecSuccess) {
    return false;
  }
// 重置根安全项的证书
certs.reset(root_security_items);
// 返回 true
return true;
}

// 将证书添加到 X509 存储中
inline bool add_certs_to_x509_store(CFArrayRef certs, X509_STORE *store) {
  auto result = false;
  // 遍历证书数组
  for (int i = 0; i < CFArrayGetCount(certs); ++i) {
    // 获取证书
    const auto cert = reinterpret_cast<const __SecCertificate *>(
        CFArrayGetValueAtIndex(certs, i));

    // 如果证书类型不是 SecCertificate 类型，则继续下一次循环
    if (SecCertificateGetTypeID() != CFGetTypeID(cert)) { continue; }

    // 导出证书数据
    CFDataRef cert_data = nullptr;
    if (SecItemExport(cert, kSecFormatX509Cert, 0, nullptr, &cert_data) !=
        errSecSuccess) {
      continue;
    }

    // 创建 CFObjectPtr 对象，用于管理证书数据的内存
    CFObjectPtr<CFDataRef> cert_data_ptr(cert_data, cf_object_ptr_deleter);
    // 将证书数据转换为无符号字符指针
    auto encoded_cert = static_cast<const unsigned char *>(CFDataGetBytePtr(cert_data_ptr.get()));

    // 解析证书数据并创建 X509 对象
    auto x509 = d2i_X509(NULL, &encoded_cert, CFDataGetLength(cert_data_ptr.get()));

    // 如果成功创建了 X509 对象
    if (x509) {
      // 将 X509 对象添加到证书存储中
      X509_STORE_add_cert(store, x509);
      // 释放 X509 对象的内存
      X509_free(x509);
      // 设置结果为 true
      result = true;
    }
  }

  // 返回结果
  return result;
}

// 在 macOS 上加载系统证书
inline bool load_system_certs_on_macos(X509_STORE *store) {
  // 初始化结果为 false
  auto result = false;
  // 创建 CFArrayRef 类型的指针 certs
  CFObjectPtr<CFArrayRef> certs(nullptr, cf_object_ptr_deleter);
  // 如果成功从钥匙串中检索到证书
  if (retrieve_certs_from_keychain(certs) && certs) {
  // 将证书添加到 X509 存储中
  result = add_certs_to_x509_store(certs.get(), store);
}

// 如果从钥匙串中检索到根证书并且证书存在
if (retrieve_root_certs_from_keychain(certs) && certs) {
  // 将证书添加到 X509 存储中，并将结果与之前的结果进行逻辑或操作
  result = add_certs_to_x509_store(certs.get(), store) || result;
}

// 返回结果
return result;
}
#endif // TARGET_OS_OSX
#endif // _WIN32
#endif // CPPHTTPLIB_OPENSSL_SUPPORT

#ifdef _WIN32
// 定义 WSInit 类
class WSInit {
public:
  // 构造函数
  WSInit() {
    // 初始化 Windows 套接字库
    WSADATA wsaData;
    if (WSAStartup(0x0002, &wsaData) == 0) is_valid_ = true;
  }
// 在析构函数中，如果对象有效，则调用WSACleanup()函数
~WSInit() {
    if (is_valid_) WSACleanup();
}

// 默认构造函数中，初始化is_valid_为false
bool is_valid_ = false;
};

// 静态WSInit对象wsinit_，用于初始化Winsock库
static WSInit wsinit_;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 生成摘要认证头部的函数
inline std::pair<std::string, std::string> make_digest_authentication_header(
    const Request &req, const std::map<std::string, std::string> &auth,
    size_t cnonce_count, const std::string &cnonce, const std::string &username,
    const std::string &password, bool is_proxy = false) {
  // 初始化nc变量
  std::string nc;
  {
    // 使用stringstream将cnonce_count转换为8位16进制数，并存入nc
    std::stringstream ss;
    ss << std::setfill('0') << std::setw(8) << std::hex << cnonce_count;
  # 将ss转换为字符串并赋值给nc
  nc = ss.str();
  }

  # 定义并初始化qop变量
  std::string qop;
  # 如果auth中包含"qop"字段
  if (auth.find("qop") != auth.end()) {
    # 将"qop"字段的值赋给qop
    qop = auth.at("qop");
    # 如果qop中包含"auth-int"
    if (qop.find("auth-int") != std::string::npos) {
      # 将qop设置为"auth-int"
      qop = "auth-int";
    } 
    # 如果qop中包含"auth"
    else if (qop.find("auth") != std::string::npos) {
      # 将qop设置为"auth"
      qop = "auth";
    } 
    # 如果qop中不包含"auth-int"或"auth"
    else {
      # 清空qop
      qop.clear();
    }
  }

  # 定义并初始化algo变量为"MD5"
  std::string algo = "MD5";
  # 如果auth中包含"algorithm"字段
  if (auth.find("algorithm") != auth.end()) { 
    # 将"algorithm"字段的值赋给algo
    algo = auth.at("algorithm"); 
  }

  # 定义并初始化response变量
  std::string response;
  {
    // 根据算法类型选择对应的哈希函数
    auto H = algo == "SHA-256"   ? detail::SHA_256
             : algo == "SHA-512" ? detail::SHA_512
                                 : detail::MD5;

    // 计算 A1
    auto A1 = username + ":" + auth.at("realm") + ":" + password;

    // 计算 A2，如果 qop 为 "auth-int"，则加上请求体的哈希值
    auto A2 = req.method + ":" + req.path;
    if (qop == "auth-int") { A2 += ":" + H(req.body); }

    // 根据 qop 是否为空选择不同的计算方式
    if (qop.empty()) {
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + H(A2));
    } else {
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + nc + ":" + cnonce +
                   ":" + qop + ":" + H(A2));
    }

    // 获取 opaque 字段，如果不存在则为空字符串
    auto opaque = (auth.find("opaque") != auth.end()) ? auth.at("opaque") : "";

    // 构建 Digest 字段
    auto field = "Digest username=\"" + username + "\", realm=\"" +
// 构建包含认证信息的字符串，包括 realm、nonce、uri、algorithm、qop、nc、cnonce、response、opaque 等字段
field = "Digest username=\"" + auth.at("username") + "\", realm=\"" + 
               auth.at("realm") + "\", nonce=\"" + auth.at("nonce") +
               "\", uri=\"" + req.path + "\", algorithm=" + algo +
               (qop.empty() ? ", response=\""
                            : ", qop=" + qop + ", nc=" + nc + ", cnonce=\"" +
                                  cnonce + "\", response=\"") +
               response + "\"" +
               (opaque.empty() ? "" : ", opaque=\"" + opaque + "\"");

// 根据是否代理请求，选择设置 Authorization 或 Proxy-Authorization 头部
auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
return std::make_pair(key, field);
}

// 解析响应中的认证信息，并存储到 auth 字典中
inline bool parse_www_authenticate(const Response &res,
                                   std::map<std::string, std::string> &auth,
                                   bool is_proxy) {
  // 根据是否代理请求，选择解析 Proxy-Authenticate 或 WWW-Authenticate 头部
  auto auth_key = is_proxy ? "Proxy-Authenticate" : "WWW-Authenticate";
  if (res.has_header(auth_key)) {
    // 使用正则表达式解析认证信息字段
    static auto re = std::regex(R"~((?:(?:,\s*)?(.+?)=(?:"(.*?)"|([^,]*))))~");
    auto s = res.get_header_value(auth_key);
    // 在字符串中查找空格的位置
    auto pos = s.find(' ');
    // 如果找到了空格
    if (pos != std::string::npos) {
      // 提取空格前的部分作为类型
      auto type = s.substr(0, pos);
      // 如果类型是"Basic"，返回false
      if (type == "Basic") {
        return false;
      } 
      // 如果类型是"Digest"
      else if (type == "Digest") {
        // 从空格后的部分开始处理
        s = s.substr(pos + 1);
        // 使用正则表达式匹配字符串
        auto beg = std::sregex_iterator(s.begin(), s.end(), re);
        // 遍历匹配结果
        for (auto i = beg; i != std::sregex_iterator(); ++i) {
          auto m = *i;
          // 提取匹配到的键和值
          auto key = s.substr(static_cast<size_t>(m.position(1)),
                              static_cast<size_t>(m.length(1)));
          auto val = m.length(2) > 0
                         ? s.substr(static_cast<size_t>(m.position(2)),
                                    static_cast<size_t>(m.length(2)))
                         : s.substr(static_cast<size_t>(m.position(3)),
                                    static_cast<size_t>(m.length(3)));
          // 将键值对存入auth字典
          auth[key] = val;
        }
        // 返回true
        return true;
// 返回 false，表示未找到匹配项
return false;
}

// 生成指定长度的随机字符串
// 参考自 https://stackoverflow.com/questions/440133/how-do-i-create-a-random-alpha-numeric-string-in-c/440240#answer-440240
inline std::string random_string(size_t length) {
  // 定义一个 lambda 函数，用于生成随机字符
  auto randchar = []() -> char {
    // 定义字符集合
    const char charset[] = "0123456789"
                           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
                           "abcdefghijklmnopqrstuvwxyz";
    // 计算字符集合的最大索引
    const size_t max_index = (sizeof(charset) - 1);
    // 生成随机字符
    return charset[static_cast<size_t>(std::rand()) % max_index];
  };
  // 创建一个长度为 length 的字符串
  std::string str(length, 0);
  // 使用 randchar 函数生成随机字符填充字符串
  std::generate_n(str.begin(), length, randchar);
  // 返回生成的随机字符串
  return str;
}
// 定义一个名为ContentProviderAdapter的类
class ContentProviderAdapter {
public:
  // 显式构造函数，接受ContentProviderWithoutLength类型的参数，并将其移动构造到content_provider_成员变量中
  explicit ContentProviderAdapter(
      ContentProviderWithoutLength &&content_provider)
      : content_provider_(content_provider) {}

  // 重载()运算符，接受偏移量和数据接收器参数，调用content_provider_对象的操作符()来处理数据
  bool operator()(size_t offset, size_t, DataSink &sink) {
    return content_provider_(offset, sink);
  }

private:
  // ContentProviderWithoutLength类型的成员变量content_provider_
  ContentProviderWithoutLength content_provider_;
};

} // namespace detail

// 根据主机名获取主机地址的函数
inline std::string hosted_at(const std::string &hostname) {
  // 定义一个存储主机地址的字符串向量
  std::vector<std::string> addrs;
  // 调用hosted_at函数获取主机地址
  hosted_at(hostname, addrs);
  // 如果主机地址为空，则返回空字符串
  if (addrs.empty()) { return std::string(); }
// 返回地址列表中的第一个地址
  return addrs[0];
}

// 根据主机名获取地址信息并存储在地址列表中
inline void hosted_at(const std::string &hostname,
                      std::vector<std::string> &addrs) {
  // 定义地址信息结构体和结果指针
  struct addrinfo hints;
  struct addrinfo *result;

  // 将hints结构体清零
  memset(&hints, 0, sizeof(struct addrinfo));
  // 设置地址族为未指定
  hints.ai_family = AF_UNSPEC;
  // 设置套接字类型为流式套接字
  hints.ai_socktype = SOCK_STREAM;
  // 设置协议为默认
  hints.ai_protocol = 0;

  // 根据主机名获取地址信息
  if (getaddrinfo(hostname.c_str(), nullptr, &hints, &result)) {
    // 如果获取失败，根据操作系统类型进行特定处理
#if defined __linux__ && !defined __ANDROID__
    res_init();
#endif
    // 返回
    return;
  }
// 遍历结果链表，获取每个地址的信息
for (auto rp = result; rp; rp = rp->ai_next) {
  // 将地址信息转换为 sockaddr_storage 结构体
  const auto &addr = *reinterpret_cast<struct sockaddr_storage *>(rp->ai_addr);
  // 定义 IP 地址和端口号
  std::string ip;
  int dummy = -1;
  // 调用 get_ip_and_port 函数获取 IP 地址和端口号，并将 IP 地址添加到 addrs 容器中
  if (detail::get_ip_and_port(addr, sizeof(struct sockaddr_storage), ip, dummy)) {
    addrs.push_back(ip);
  }
}

// 释放地址信息链表
freeaddrinfo(result);
}

// 拼接查询参数到路径后面
inline std::string append_query_params(const std::string &path, const Params &params) {
  // 初始化路径和查询参数
  std::string path_with_query = path;
  // 定义正则表达式，用于匹配路径是否已经包含查询参数
  const static std::regex re("[^?]+\\?.*");
  // 根据是否已经包含查询参数，选择连接符号
  auto delm = std::regex_match(path, re) ? '&' : '?';
  // 将查询参数拼接到路径后面
  path_with_query += delm + detail::params_to_query_str(params);
// 返回带有查询的路径
return path_with_query;
}

// 头部工具
// 创建范围头部，用于指定请求的数据范围
inline std::pair<std::string, std::string> make_range_header(Ranges ranges) {
  std::string field = "bytes=";
  auto i = 0;
  for (auto r : ranges) {
    if (i != 0) { field += ", "; } // 如果不是第一个范围，添加逗号和空格
    if (r.first != -1) { field += std::to_string(r.first); } // 如果范围有起始值，添加起始值
    field += '-';
    if (r.second != -1) { field += std::to_string(r.second); } // 如果范围有结束值，添加结束值
    i++;
  }
  return std::make_pair("Range", std::move(field)); // 返回范围头部
}

// 创建基本身份验证头部，用于在请求中包含用户名和密码
inline std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password, bool is_proxy) {
// 创建基本身份验证头部信息，使用用户名和密码进行 base64 编码
auto field = "Basic " + detail::base64_encode(username + ":" + password);
// 根据是否使用代理，选择不同的头部字段名
auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
// 返回头部字段名和字段值的 pair 对象
return std::make_pair(key, std::move(field));
}

// 创建 Bearer token 身份验证头部信息
inline std::pair<std::string, std::string>
make_bearer_token_authentication_header(const std::string &token,
                                        bool is_proxy = false) {
  // 创建 Bearer token 头部字段值
  auto field = "Bearer " + token;
  // 根据是否使用代理，选择不同的头部字段名
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  // 返回头部字段名和字段值的 pair 对象
  return std::make_pair(key, std::move(field));
}

// 检查请求是否包含特定头部信息
inline bool Request::has_header(const std::string &key) const {
  // 调用 detail 命名空间中的函数，检查请求头部中是否包含特定的键
  return detail::has_header(headers, key);
}

// 获取特定头部信息的值
inline std::string Request::get_header_value(const std::string &key,
                                             size_t id) const {
// 返回指定键的值
return detail::get_header_value(headers, key, id, "");

// 返回指定键对应的值的数量
inline size_t Request::get_header_value_count(const std::string &key) const {
  // 查找指定键的所有值的范围
  auto r = headers.equal_range(key);
  // 返回值的数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置请求头的键值对
inline void Request::set_header(const std::string &key,
                                const std::string &val) {
  // 如果键和值都不包含换行符，则添加到请求头中
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    headers.emplace(key, val);
  }
}

// 检查是否存在指定的参数
inline bool Request::has_param(const std::string &key) const {
  // 查找参数是否存在
  return params.find(key) != params.end();
}

// 获取指定参数的值
inline std::string Request::get_param_value(const std::string &key,
// 在参数列表中查找指定键的值范围
auto rng = params.equal_range(key);
// 获取指定键的值范围的第一个迭代器
auto it = rng.first;
// 将迭代器向前移动指定的距离
std::advance(it, static_cast<ssize_t>(id));
// 如果迭代器不等于值范围的末尾迭代器，则返回对应的值，否则返回空字符串
if (it != rng.second) { return it->second; }
return std::string();

// 获取指定键的值范围的元素个数
inline size_t Request::get_param_value_count(const std::string &key) const {
  auto r = params.equal_range(key);
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 检查请求是否为多部分表单数据
inline bool Request::is_multipart_form_data() const {
  // 获取请求头中的 Content-Type 值
  const auto &content_type = get_header_value("Content-Type");
  // 如果 Content-Type 以 "multipart/form-data" 开头，则返回 true，否则返回 false
  return !content_type.rfind("multipart/form-data", 0);
}

// 检查是否存在指定键的文件
inline bool Request::has_file(const std::string &key) const {
  // 在文件列表中查找指定键，如果找到则返回 true，否则返回 false
  return files.find(key) != files.end();
}
// 获取指定键对应的文件数据，如果找到则返回对应的文件数据，否则返回空的 MultipartFormData 对象
inline MultipartFormData Request::get_file_value(const std::string &key) const {
  // 查找指定键对应的文件数据
  auto it = files.find(key);
  // 如果找到则返回对应的文件数据
  if (it != files.end()) { return it->second; }
  // 否则返回空的 MultipartFormData 对象
  return MultipartFormData();
}

// 获取指定键对应的所有文件数据，返回一个包含所有文件数据的向量
inline std::vector<MultipartFormData>
Request::get_file_values(const std::string &key) const {
  // 创建一个存储文件数据的向量
  std::vector<MultipartFormData> values;
  // 查找指定键对应的文件数据范围
  auto rng = files.equal_range(key);
  // 遍历文件数据范围，将文件数据添加到向量中
  for (auto it = rng.first; it != rng.second; it++) {
    values.push_back(it->second);
  }
  // 返回包含所有文件数据的向量
  return values;
}

// Response 实现
// 检查响应是否包含指定键的头部信息
inline bool Response::has_header(const std::string &key) const {
// 检查给定的键是否存在于响应头中
return headers.find(key) != headers.end();

// 获取指定键的响应头值
inline std::string Response::get_header_value(const std::string &key,
                                              size_t id) const {
  return detail::get_header_value(headers, key, id, "");
}

// 获取指定键的响应头值的数量
inline size_t Response::get_header_value_count(const std::string &key) const {
  auto r = headers.equal_range(key);
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置响应头的键值对
inline void Response::set_header(const std::string &key,
                                 const std::string &val) {
  // 检查键和值中是否包含换行符，如果不包含则添加到响应头中
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    headers.emplace(key, val);
  }
}
// 设置重定向的 URL 和状态码
inline void Response::set_redirect(const std::string &url, int stat) {
  // 检查 URL 是否包含换行符和回车符
  if (!detail::has_crlf(url)) {
    // 设置响应头中的 Location 字段为给定的 URL
    set_header("Location", url);
    // 如果状态码在 300 到 399 之间，则设置响应的状态码为给定的状态码，否则设置为 302
    if (300 <= stat && stat < 400) {
      this->status = stat;
    } else {
      this->status = 302;
    }
  }
}

// 设置响应的内容、内容长度和内容类型
inline void Response::set_content(const char *s, size_t n,
                                  const std::string &content_type) {
  // 将给定的字符数组和长度赋值给响应的内容
  body.assign(s, n);

  // 查找并删除响应头中已存在的 Content-Type 字段
  auto rng = headers.equal_range("Content-Type");
  headers.erase(rng.first, rng.second);
  // 设置响应头中的 Content-Type 字段为给定的内容类型
  set_header("Content-Type", content_type);
}
// 设置响应的内容和内容类型
inline void Response::set_content(const std::string &s,
                                  const std::string &content_type) {
  // 调用另一个重载的set_content函数，传入字符串的数据指针和大小
  set_content(s.data(), s.size(), content_type);
}

// 设置内容提供者和内容类型
inline void Response::set_content_provider(
    size_t in_length, const std::string &content_type, ContentProvider provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置响应头的Content-Type字段
  set_header("Content-Type", content_type);
  // 设置内容长度
  content_length_ = in_length;
  // 如果内容长度大于0，则移动内容提供者
  if (in_length > 0) { content_provider_ = std::move(provider); }
  // 设置内容提供者资源释放器
  content_provider_resource_releaser_ = resource_releaser;
  // 设置为非分块内容提供者
  is_chunked_content_provider_ = false;
}

// 设置没有长度的内容提供者和内容类型
inline void Response::set_content_provider(
    const std::string &content_type, ContentProviderWithoutLength provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置响应头的Content-Type字段
  set_header("Content-Type", content_type);
  // 设置内容长度为0
  content_length_ = 0;
}
// 使用提供的内容提供者适配器初始化内容提供者
content_provider_ = detail::ContentProviderAdapter(std::move(provider));
// 设置资源释放器
content_provider_resource_releaser_ = resource_releaser;
// 将内容提供者设置为非分块内容提供者
is_chunked_content_provider_ = false;
}

// 设置为分块内容提供者
inline void Response::set_chunked_content_provider(
    const std::string &content_type, ContentProviderWithoutLength provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置响应头的内容类型
  set_header("Content-Type", content_type);
  // 设置内容长度为0
  content_length_ = 0;
  // 使用提供的内容提供者适配器初始化内容提供者
  content_provider_ = detail::ContentProviderAdapter(std::move(provider));
  // 设置资源释放器
  content_provider_resource_releaser_ = resource_releaser;
  // 将内容提供者设置为分块内容提供者
  is_chunked_content_provider_ = true;
}

// 检查请求头中是否包含指定的键
inline bool Result::has_request_header(const std::string &key) const {
  return request_headers_.find(key) != request_headers_.end();
}
// 获取请求头中指定键的值
inline std::string Result::get_request_header_value(const std::string &key,
                                                    size_t id) const {
  return detail::get_header_value(request_headers_, key, id, "");
}

// 获取请求头中指定键的值的数量
inline size_t
Result::get_request_header_value_count(const std::string &key) const {
  auto r = request_headers_.equal_range(key);
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 写入字符数组到流中
inline ssize_t Stream::write(const char *ptr) {
  return write(ptr, strlen(ptr));
}

// 写入字符串到流中
inline ssize_t Stream::write(const std::string &s) {
  return write(s.data(), s.size());
}
namespace detail {

// Socket stream implementation
// 构造函数，初始化 SocketStream 对象
inline SocketStream::SocketStream(socket_t sock, time_t read_timeout_sec,
                                  time_t read_timeout_usec,
                                  time_t write_timeout_sec,
                                  time_t write_timeout_usec)
    : sock_(sock), read_timeout_sec_(read_timeout_sec),
      read_timeout_usec_(read_timeout_usec),
      write_timeout_sec_(write_timeout_sec),
      write_timeout_usec_(write_timeout_usec), read_buff_(read_buff_size_, 0) {}

// 析构函数，释放 SocketStream 对象
inline SocketStream::~SocketStream() {}

// 检查套接字是否可读
inline bool SocketStream::is_readable() const {
  return select_read(sock_, read_timeout_sec_, read_timeout_usec_) > 0;
}

// 检查套接字是否可写
inline bool SocketStream::is_writable() const {
  return select_write(sock_, write_timeout_sec_, write_timeout_usec_) > 0 &&
// 检查套接字是否处于活动状态
is_socket_alive(sock_);

// 读取指定大小的数据到指定的缓冲区中
#ifdef _WIN32
  // 如果是 Windows 系统，限制读取大小为 int 类型的最大值
  size = (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#else
  // 如果不是 Windows 系统，限制读取大小为 ssize_t 类型的最大值
  size = (std::min)(size, static_cast<size_t>((std::numeric_limits<ssize_t>::max)()));
#endif

// 如果读取缓冲区中还有剩余数据
if (read_buff_off_ < read_buff_content_size_) {
  // 计算剩余数据的大小
  auto remaining_size = read_buff_content_size_ - read_buff_off_;
  // 如果需要读取的大小小于等于剩余数据的大小
  if (size <= remaining_size) {
    // 将剩余数据拷贝到指定的缓冲区中
    memcpy(ptr, read_buff_.data() + read_buff_off_, size);
    // 更新读取缓冲区的偏移量
    read_buff_off_ += size;
    // 返回实际读取的数据大小
    return static_cast<ssize_t>(size);
  } else {
    // 如果需要读取的大小大于剩余数据的大小，则将剩余数据拷贝到指定的缓冲区中
    memcpy(ptr, read_buff_.data() + read_buff_off_, remaining_size);
      read_buff_off_ += remaining_size;
      // 将读取偏移量增加剩余大小，表示已经读取了这么多数据
      return static_cast<ssize_t>(remaining_size);
      // 返回剩余大小的类型转换结果
    }
  }

  if (!is_readable()) { return -1; }
  // 如果不可读，返回-1

  read_buff_off_ = 0;
  // 重置读取偏移量为0
  read_buff_content_size_ = 0;
  // 重置读取缓冲区内容大小为0

  if (size < read_buff_size_) {
    // 如果请求的大小小于读取缓冲区的大小
    auto n = read_socket(sock_, read_buff_.data(), read_buff_size_,
                         CPPHTTPLIB_RECV_FLAGS);
    // 从套接字中读取数据到读取缓冲区
    if (n <= 0) {
      return n;
      // 如果读取失败，返回错误码
    } else if (n <= static_cast<ssize_t>(size)) {
      memcpy(ptr, read_buff_.data(), static_cast<size_t>(n));
      // 如果读取的数据小于等于请求的大小，将数据拷贝到指定位置
      return n;
      // 返回实际读取的大小
    } else {
      memcpy(ptr, read_buff_.data(), size);
      // 如果读取的数据大于请求的大小，只拷贝请求大小的数据
// 设置读取缓冲区的偏移量为指定大小
read_buff_off_ = size;
// 设置读取缓冲区的内容大小为给定大小
read_buff_content_size_ = static_cast<size_t>(n);
// 返回读取的数据大小
return static_cast<ssize_t>(size);
// 如果不是可写状态，返回-1
if (!is_writable()) { return -1; }
// 如果是 Windows 平台且不是 64 位系统，限制 size 的大小为 int 类型的最大值
#if defined(_WIN32) && !defined(_WIN64)
  size =
      (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#endif
// 发送数据到套接字
return send_socket(sock_, ptr, size, CPPHTTPLIB_SEND_FLAGS);
// 获取远程主机的 IP 地址和端口号
inline void SocketStream::get_remote_ip_and_port(std::string &ip, int &port) const {
  return detail::get_remote_ip_and_port(sock_, ip, port);
}

// 获取本地主机的 IP 地址和端口号
inline void SocketStream::get_local_ip_and_port(std::string &ip, int &port) const {
  return detail::get_local_ip_and_port(sock_, ip, port);
}

// 返回套接字对象
inline socket_t SocketStream::socket() const { return sock_; }

// 缓冲流实现
// 判断流是否可读
inline bool BufferStream::is_readable() const { return true; }

// 判断流是否可写
inline bool BufferStream::is_writable() const { return true; }

// 从缓冲流中读取数据
inline ssize_t BufferStream::read(char *ptr, size_t size) {
#if defined(_MSC_VER) && _MSC_VER < 1910
  auto len_read = buffer._Copy_s(ptr, size, size, position);
// 如果不是上述情况，则从缓冲区中复制数据到指定位置，并返回复制的长度
#else
  auto len_read = buffer.copy(ptr, size, position);
#endif
  // 更新位置
  position += static_cast<size_t>(len_read);
  // 返回已经复制的长度
  return static_cast<ssize_t>(len_read);
}

// 向缓冲区中写入数据
inline ssize_t BufferStream::write(const char *ptr, size_t size) {
  // 将数据追加到缓冲区末尾
  buffer.append(ptr, size);
  // 返回写入的数据长度
  return static_cast<ssize_t>(size);
}

// 获取远程 IP 地址和端口号
inline void BufferStream::get_remote_ip_and_port(std::string & /*ip*/,
                                                 int & /*port*/) const {}

// 获取本地 IP 地址和端口号
inline void BufferStream::get_local_ip_and_port(std::string & /*ip*/,
                                                int & /*port*/) const {}

// 获取套接字
inline socket_t BufferStream::socket() const { return 0; }
// 返回缓冲区内容的引用
inline const std::string &BufferStream::get_buffer() const { return buffer; }

} // namespace detail

// HTTP 服务器实现
// 构造函数，初始化新任务队列和线程池
inline Server::Server()
    : new_task_queue(
          [] { return new ThreadPool(CPPHTTPLIB_THREAD_POOL_COUNT); }) {
#ifndef _WIN32
  // 忽略 SIGPIPE 信号
  signal(SIGPIPE, SIG_IGN);
#endif
}

// 析构函数
inline Server::~Server() {}

// 添加 GET 请求处理器
inline Server &Server::Get(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理器添加到 GET 处理器列表中
  get_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}
// 在服务器对象中注册一个处理 POST 请求的处理程序
inline Server &Server::Post(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理程序作为一对添加到 POST 请求处理程序列表中
  post_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  // 返回服务器对象的引用
  return *this;
}

// 在服务器对象中注册一个处理 POST 请求的处理程序，该处理程序包含内容读取器
inline Server &Server::Post(const std::string &pattern,
                            HandlerWithContentReader handler) {
  // 将匹配模式和包含内容读取器的处理程序作为一对添加到 POST 请求处理程序列表中
  post_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  // 返回服务器对象的引用
  return *this;
}

// 在服务器对象中注册一个处理 PUT 请求的处理程序
inline Server &Server::Put(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理程序作为一对添加到 PUT 请求处理程序列表中
  put_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  // 返回服务器对象的引用
  return *this;
}
// 将处理器和内容读取器的处理函数添加到PUT请求的处理器列表中
inline Server &Server::Put(const std::string &pattern,
                           HandlerWithContentReader handler) {
  put_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 将处理器添加到PATCH请求的处理器列表中
inline Server &Server::Patch(const std::string &pattern, Handler handler) {
  patch_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 将处理器和内容读取器的处理函数添加到PATCH请求的处理器列表中
inline Server &Server::Patch(const std::string &pattern,
                             HandlerWithContentReader handler) {
  patch_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}
// 在服务器对象中注册一个删除请求的处理程序，将匹配模式和处理程序存储到删除处理程序列表中
inline Server &Server::Delete(const std::string &pattern, Handler handler) {
  delete_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 在服务器对象中注册一个带内容读取器的删除请求的处理程序，将匹配模式和处理程序存储到带内容读取器的删除处理程序列表中
inline Server &Server::Delete(const std::string &pattern,
                              HandlerWithContentReader handler) {
  delete_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 在服务器对象中注册一个选项请求的处理程序，将匹配模式和处理程序存储到选项处理程序列表中
inline Server &Server::Options(const std::string &pattern, Handler handler) {
  options_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 设置服务器对象的基本目录
inline bool Server::set_base_dir(const std::string &dir,
// 设置挂载点，将挂载点和目录关联起来
inline bool Server::set_mount_point(const std::string &mount_point,
                                    const std::string &dir, Headers headers) {
  // 如果目录存在
  if (detail::is_dir(dir)) {
    // 如果挂载点不为空，则使用挂载点，否则使用根目录
    std::string mnt = !mount_point.empty() ? mount_point : "/";
    // 如果挂载点不为空且以斜杠开头
    if (!mnt.empty() && mnt[0] == '/') {
      // 将挂载点、目录和头信息添加到基本目录列表中
      base_dirs_.push_back({mnt, dir, std::move(headers)});
      return true;
    }
  }
  return false;
}

// 移除挂载点
inline bool Server::remove_mount_point(const std::string &mount_point) {
  // 遍历基本目录列表
  for (auto it = base_dirs_.begin(); it != base_dirs_.end(); ++it) {
    // 如果找到对应的挂载点
    if (it->mount_point == mount_point) {
      // 移除该挂载点
      base_dirs_.erase(it);
// 返回 true
      return true;
    }
  }
  // 返回 false
  return false;
}

// 设置文件扩展名和 MIME 类型的映射关系
inline Server &
Server::set_file_extension_and_mimetype_mapping(const std::string &ext,
                                                const std::string &mime) {
  // 将文件扩展名和 MIME 类型添加到映射关系中
  file_extension_and_mimetype_map_[ext] = mime;
  // 返回当前 Server 对象的引用
  return *this;
}

// 设置文件请求处理程序
inline Server &Server::set_file_request_handler(Handler handler) {
  // 移动赋值文件请求处理程序
  file_request_handler_ = std::move(handler);
  // 返回当前 Server 对象的引用
  return *this;
}

// 设置错误处理程序
inline Server &Server::set_error_handler(HandlerWithResponse handler) {
  // 移动赋值错误处理程序
  error_handler_ = std::move(handler);
// 返回当前对象的引用
  return *this;
}

// 设置错误处理程序，接受一个处理程序作为参数
inline Server &Server::set_error_handler(Handler handler) {
  // 使用lambda表达式将传入的处理程序包装成错误处理程序，并赋值给error_handler_
  error_handler_ = [handler](const Request &req, Response &res) {
    handler(req, res);
    return HandlerResponse::Handled;
  };
  // 返回当前对象的引用
  return *this;
}

// 设置异常处理程序，接受一个异常处理程序作为参数
inline Server &Server::set_exception_handler(ExceptionHandler handler) {
  // 将传入的异常处理程序移动到exception_handler_中，并返回当前对象的引用
  exception_handler_ = std::move(handler);
  return *this;
}

// 设置预路由处理程序，接受一个带有响应的处理程序作为参数
inline Server &Server::set_pre_routing_handler(HandlerWithResponse handler) {
  // 将传入的带有响应的处理程序移动到pre_routing_handler_中，并返回当前对象的引用
  pre_routing_handler_ = std::move(handler);
  return *this;
}
// 设置 post_routing_handler_ 的值为传入的 handler，并返回当前 Server 对象的引用
inline Server &Server::set_post_routing_handler(Handler handler) {
  post_routing_handler_ = std::move(handler);
  return *this;
}

// 设置 logger_ 的值为传入的 logger，并返回当前 Server 对象的引用
inline Server &Server::set_logger(Logger logger) {
  logger_ = std::move(logger);
  return *this;
}

// 设置 expect_100_continue_handler_ 的值为传入的 handler，并返回当前 Server 对象的引用
inline Server &Server::set_expect_100_continue_handler(Expect100ContinueHandler handler) {
  expect_100_continue_handler_ = std::move(handler);
  return *this;
}

// 设置 address_family_ 的值为传入的 family
inline Server &Server::set_address_family(int family) {
  address_family_ = family;
// 返回当前对象的引用
return *this;
// 设置是否启用TCP无延迟
inline Server &Server::set_tcp_nodelay(bool on) {
  tcp_nodelay_ = on;
  return *this;
}
// 设置套接字选项
inline Server &Server::set_socket_options(SocketOptions socket_options) {
  socket_options_ = std::move(socket_options);
  return *this;
}
// 设置默认的HTTP头部
inline Server &Server::set_default_headers(Headers headers) {
  default_headers_ = std::move(headers);
  return *this;
}
// 设置保持活动的最大计数
inline Server &Server::set_keep_alive_max_count(size_t count) {
  keep_alive_max_count_ = count;
// 返回当前对象的引用
return *this;
// 设置服务器的保持连接超时时间，单位为秒
inline Server &Server::set_keep_alive_timeout(time_t sec) {
  keep_alive_timeout_sec_ = sec;
  return *this;
}
// 设置服务器的读取超时时间，单位为秒和微秒
inline Server &Server::set_read_timeout(time_t sec, time_t usec) {
  read_timeout_sec_ = sec;
  read_timeout_usec_ = usec;
  return *this;
}
// 设置服务器的写入超时时间，单位为秒和微秒
inline Server &Server::set_write_timeout(time_t sec, time_t usec) {
  write_timeout_sec_ = sec;
  write_timeout_usec_ = usec;
  return *this;
}
// 设置服务器的空闲间隔时间，以秒和微秒为单位
inline Server &Server::set_idle_interval(time_t sec, time_t usec) {
  idle_interval_sec_ = sec;
  idle_interval_usec_ = usec;
  return *this;
}

// 设置服务器的最大负载长度
inline Server &Server::set_payload_max_length(size_t length) {
  payload_max_length_ = length;
  return *this;
}

// 将服务器绑定到指定主机和端口，使用给定的套接字标志
inline bool Server::bind_to_port(const std::string &host, int port,
                                 int socket_flags) {
  // 如果绑定失败，则返回false
  if (bind_internal(host, port, socket_flags) < 0) return false;
  // 绑定成功，返回true
  return true;
}

// 将服务器绑定到任意端口，使用给定的套接字标志
inline int Server::bind_to_any_port(const std::string &host, int socket_flags) {
  // 返回绑定的结果
  return bind_internal(host, 0, socket_flags);
}
// 在绑定后监听服务器
inline bool Server::listen_after_bind() {
  // 创建一个作用域退出对象，用于在函数返回时设置 done_ 为 true
  auto se = detail::scope_exit([&]() { done_ = true; });
  // 调用内部的监听函数
  return listen_internal();
}

// 监听指定主机和端口
inline bool Server::listen(const std::string &host, int port,
                           int socket_flags) {
  // 创建一个作用域退出对象，用于在函数返回时设置 done_ 为 true
  auto se = detail::scope_exit([&]() { done_ = true; });
  // 绑定到指定的主机和端口，并调用内部的监听函数
  return bind_to_port(host, port, socket_flags) && listen_internal();
}

// 检查服务器是否正在运行
inline bool Server::is_running() const { return is_running_; }

// 等待服务器准备就绪
inline void Server::wait_until_ready() const {
  // 当服务器未运行且未完成时，循环等待
  while (!is_running() && !done_) {
    std::this_thread::sleep_for(std::chrono::milliseconds{1});
  }
}

// 停止服务器
inline void Server::stop() {
// 如果服务器正在运行
if (is_running_) {
  // 确保服务器套接字不是无效套接字
  assert(svr_sock_ != INVALID_SOCKET);
  // 使用原子操作将服务器套接字设置为无效套接字，并返回之前的值
  std::atomic<socket_t> sock(svr_sock_.exchange(INVALID_SOCKET));
  // 关闭套接字
  detail::shutdown_socket(sock);
  // 关闭套接字
  detail::close_socket(sock);
}

// 解析请求行
inline bool Server::parse_request_line(const char *s, Request &req) {
  // 获取字符串长度
  auto len = strlen(s);
  // 如果长度小于2或者倒数第二个字符不是'\r'或者最后一个字符不是'\n'，返回false
  if (len < 2 || s[len - 2] != '\r' || s[len - 1] != '\n') { return false; }
  len -= 2;

  {
    // 定义计数器
    size_t count = 0;

    // 使用split函数将字符串s分割，并对每个部分进行处理
    detail::split(s, s + len, ' ', [&](const char *b, const char *e) {
      // 根据计数器的值，将分割的部分赋值给请求对象的对应属性
      switch (count) {
      case 0: req.method = std::string(b, e); break;
      case 1: req.target = std::string(b, e); break;
      case 2: req.version = std::string(b, e); break;
      // 如果匹配到第二个字段，将其赋值给请求对象的版本属性
      default: break;
      }
      count++;
    });

    if (count != 3) { return false; }
    // 如果字段数量不等于3，返回false

  }

  static const std::set<std::string> methods{
      "GET",     "HEAD",    "POST",  "PUT",   "DELETE",
      "CONNECT", "OPTIONS", "TRACE", "PATCH", "PRI"};
  // 定义HTTP请求方法的集合

  if (methods.find(req.method) == methods.end()) { return false; }
  // 如果请求方法不在定义的集合中，返回false

  if (req.version != "HTTP/1.1" && req.version != "HTTP/1.0") { return false; }
  // 如果请求的版本不是HTTP/1.1或HTTP/1.0，返回false

  {
    // 跳过URL片段
    for (size_t i = 0; i < req.target.size(); i++) {
    // 遍历请求目标的大小
    // 如果目标字符串中包含 '#'，则删除 '#' 及其后面的内容
    if (req.target[i] == '#') {
        req.target.erase(i);
        break;
    }

    // 初始化计数器
    size_t count = 0;

    // 使用 '?' 分割目标字符串，并对每个部分进行处理
    detail::split(req.target.data(), req.target.data() + req.target.size(), '?',
                  [&](const char *b, const char *e) {
                    // 根据计数器的值，对不同部分进行不同的处理
                    switch (count) {
                    case 0:
                      // 将第一部分解码后赋值给 req.path
                      req.path = detail::decode_url(std::string(b, e), false);
                      break;
                    case 1: {
                      // 如果第二部分长度大于 0，则解析查询参数并存入 req.params
                      if (e - b > 0) {
                        detail::parse_query_text(std::string(b, e), req.params);
                      }
                      break;
                    }
                    default: break; // 如果没有匹配的 case，则跳出 switch 语句
                    }
                    count++; // 每次循环增加计数器
                  });

    if (count > 2) { return false; } // 如果计数器大于2，则返回 false
  }

  return true; // 如果没有超过2次匹配，则返回 true
}

inline bool Server::write_response(Stream &strm, bool close_connection,
                                   const Request &req, Response &res) {
  return write_response_core(strm, close_connection, req, res, false); // 调用 write_response_core 函数
}

inline bool Server::write_response_with_content(Stream &strm,
                                                bool close_connection,
                                                const Request &req,
                                                Response &res) {
// 返回响应核心函数，将流、关闭连接标志、请求、响应和是否需要应用范围作为参数
bool Server::write_response_core(Stream &strm, bool close_connection,
                                        const Request &req, Response &res,
                                        bool need_apply_ranges) {
  // 确保响应状态码已经被设置
  assert(res.status != -1);

  // 如果响应状态码大于等于400，并且存在错误处理函数，并且错误处理函数处理了请求和响应，则需要应用范围
  if (400 <= res.status && error_handler_ &&
      error_handler_(req, res) == HandlerResponse::Handled) {
    need_apply_ranges = true;
  }

  // 定义内容类型和边界字符串
  std::string content_type;
  std::string boundary;
  // 如果需要应用范围，则调用应用范围函数
  if (need_apply_ranges) { apply_ranges(req, res, content_type, boundary); }

  // 准备额外的头部信息
  // 如果需要关闭连接或者请求头部中的连接值为"close"，则设置响应头部中的连接值为"close"
  if (close_connection || req.get_header_value("Connection") == "close") {
    res.set_header("Connection", "close");
  } else {
    // 如果不满足上述条件，设置 Keep-Alive 头部信息
    std::stringstream ss;
    ss << "timeout=" << keep_alive_timeout_sec_
       << ", max=" << keep_alive_max_count_;
    res.set_header("Keep-Alive", ss.str());
  }

  // 如果响应中没有设置 Content-Type 头部，并且响应体不为空，或者内容长度大于 0，或者有内容提供者，则设置 Content-Type 为 text/plain
  if (!res.has_header("Content-Type") &&
      (!res.body.empty() || res.content_length_ > 0 || res.content_provider_)) {
    res.set_header("Content-Type", "text/plain");
  }

  // 如果响应中没有设置 Content-Length 头部，并且响应体为空，并且内容长度为 0，并且没有内容提供者，则设置 Content-Length 为 0
  if (!res.has_header("Content-Length") && res.body.empty() &&
      !res.content_length_ && !res.content_provider_) {
    res.set_header("Content-Length", "0");
  }

  // 如果响应中没有设置 Accept-Ranges 头部，并且请求方法为 HEAD，则设置 Accept-Ranges 为 bytes
  if (!res.has_header("Accept-Ranges") && req.method == "HEAD") {
    res.set_header("Accept-Ranges", "bytes");
  }
  // 如果存在后置路由处理器，则调用后置路由处理器处理请求和响应
  if (post_routing_handler_) { post_routing_handler_(req, res); }

  // 响应行和头部
  {
    // 创建一个缓冲流对象
    detail::BufferStream bstrm;

    // 将响应状态码和状态消息写入缓冲流
    if (!bstrm.write_format("HTTP/1.1 %d %s\r\n", res.status,
                            detail::status_message(res.status))) {
      return false;
    }

    // 将响应头部写入缓冲流
    if (!detail::write_headers(bstrm, res.headers)) { return false; }

    // 刷新缓冲区
    auto &data = bstrm.get_buffer();
    detail::write_data(strm, data.data(), data.size());
  }

  // 响应体
  // 初始化返回值为 true
  auto ret = true;
  // 如果请求方法不是 HEAD
  if (req.method != "HEAD") {
    // 如果响应体不为空
    if (!res.body.empty()) {
      // 将响应体数据写入流中
      if (!detail::write_data(strm, res.body.data(), res.body.size())) {
        // 如果写入失败，将返回值设为 false
        ret = false;
      }
    } 
    // 如果响应体为空且存在内容提供者
    else if (res.content_provider_) {
      // 使用内容提供者写入内容到流中
      if (write_content_with_provider(strm, req, res, boundary, content_type)) {
        // 内容提供者写入成功
        res.content_provider_success_ = true;
      } else {
        // 内容提供者写入失败，将返回值设为 false
        res.content_provider_success_ = false;
        ret = false;
      }
    }
  }

  // 记录日志
  if (logger_) { logger_(req, res); }

  // 返回结果
  return ret;
}

// 内联函数，用于在服务器关闭时检查是否正在关闭
inline bool
Server::write_content_with_provider(Stream &strm, const Request &req,
                                    Response &res, const std::string &boundary,
                                    const std::string &content_type) {
  // 检查服务器是否正在关闭
  auto is_shutting_down = [this]() {
    return this->svr_sock_ == INVALID_SOCKET;
  };

  // 如果响应内容长度大于0
  if (res.content_length_ > 0) {
    // 如果请求中不包含范围
    if (req.ranges.empty()) {
      // 写入内容到流中
      return detail::write_content(strm, res.content_provider_, 0,
                                   res.content_length_, is_shutting_down);
    } 
    // 如果请求中包含一个范围
    else if (req.ranges.size() == 1) {
      // 获取范围的偏移和长度
      auto offsets =
          detail::get_range_offset_and_length(req, res.content_length_, 0);
      auto offset = offsets.first;
      auto length = offsets.second;
      // 写入内容到流中
      return detail::write_content(strm, res.content_provider_, offset, length,
      // 如果响应是分块内容提供程序，则根据请求和响应的编码类型进行压缩
      auto type = detail::encoding_type(req, res);

      // 创建一个压缩器对象
      std::unique_ptr<detail::compressor> compressor;

      // 如果编码类型是 Gzip，则使用 Gzip 压缩器
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        compressor = detail::make_unique<detail::gzip_compressor>();
#endif
      } 
      // 如果编码类型是 Brotli，则使用 Brotli 压缩器
      else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        compressor = detail::make_unique<detail::brotli_compressor>();
#endif
      } 
      // 如果编码类型不是 Gzip 或 Brotli，则使用无压缩器
      else {
        compressor = detail::make_unique<detail::nocompressor>();
      }
      // 确保压缩器不为空
      assert(compressor != nullptr);

      // 如果存在压缩器，则使用分块写入内容
      return detail::write_content_chunked(strm, res.content_provider_,
                                           is_shutting_down, *compressor);
    } else {
      // 如果不存在压缩器，则直接写入内容
      return detail::write_content_without_length(strm, res.content_provider_,
                                                  is_shutting_down);
    }
  }
}

// 读取请求内容
inline bool Server::read_content(Stream &strm, Request &req, Response &res) {
  // 初始化文件计数器
  MultipartFormDataMap::iterator cur;
  auto file_count = 0;
  // 读取请求内容的核心函数
  if (read_content_core(
          strm, req, res,
          // Regular
          // 如果请求体大小超过最大限制，则返回false
          [&](const char *buf, size_t n) {
            if (req.body.size() + n > req.body.max_size()) { return false; }
            // 将数据追加到请求体中
            req.body.append(buf, n);
            // 返回 true 表示成功
            return true;
          },
          // 处理多部分表单数据
          [&](const MultipartFormData &file) {
            // 如果文件数量达到最大限制，则返回 false
            if (file_count++ == CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT) {
              return false;
            }
            // 将文件信息添加到请求对象的文件列表中
            cur = req.files.emplace(file.name, file);
            // 返回 true 表示成功
            return true;
          },
          // 处理数据流
          [&](const char *buf, size_t n) {
            // 获取当前文件的内容
            auto &content = cur->second.content;
            // 如果数据大小超过最大限制，则返回 false
            if (content.size() + n > content.max_size()) { return false; }
            // 将数据追加到文件内容中
            content.append(buf, n);
            // 返回 true 表示成功
            return true;
          })) {
    // 获取请求头中的 Content-Type
    const auto &content_type = req.get_header_value("Content-Type");
    // 如果 Content-Type 不是 application/x-www-form-urlencoded
    if (!content_type.find("application/x-www-form-urlencoded")) {
      // 如果请求体大小超过最大限制
      if (req.body.size() > CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH) {
        res.status = 413; // 设置响应状态码为413，表示请求实体过大
        return false; // 返回false，表示读取内容失败
      }
      detail::parse_query_text(req.body, req.params); // 解析请求体中的文本数据，将解析结果存储到请求参数中
    }
    return true; // 返回true，表示读取内容成功
  }
  return false; // 返回false，表示读取内容失败
}

inline bool Server::read_content_with_content_receiver(
    Stream &strm, Request &req, Response &res, ContentReceiver receiver,
    MultipartContentHeader multipart_header,
    ContentReceiver multipart_receiver) {
  return read_content_core(strm, req, res, std::move(receiver),
                           std::move(multipart_header),
                           std::move(multipart_receiver)); // 调用read_content_core函数，传入参数并返回结果
}

inline bool Server::read_content_core(Stream &strm, Request &req, Response &res,
// 定义一个接收内容的接收器和一个多部分内容头的接收器
void handle_multipart_form_data(const Request &req, Response &res,
                                      ContentReceiver receiver,
                                      MultipartContentHeader multipart_header,
                                      ContentReceiver multipart_receiver) {
  // 创建一个多部分表单数据解析器和一个带进度的内容接收器
  detail::MultipartFormDataParser multipart_form_data_parser;
  ContentReceiverWithProgress out;

  // 如果请求是多部分表单数据
  if (req.is_multipart_form_data()) {
    // 获取请求的内容类型
    const auto &content_type = req.get_header_value("Content-Type");
    std::string boundary;
    // 解析多部分边界
    if (!detail::parse_multipart_boundary(content_type, boundary)) {
      // 如果解析失败，设置响应状态为400，并返回false
      res.status = 400;
      return false;
    }

    // 设置多部分表单数据解析器的边界
    multipart_form_data_parser.set_boundary(std::move(boundary));
    // 设置内容接收器为一个lambda函数
    out = [&](const char *buf, size_t n, uint64_t /*off*/, uint64_t /*len*/) {
      /* For debug
      size_t pos = 0;
      while (pos < n) {
        auto read_size = (std::min)<size_t>(1, n - pos);
      */
      // 使用 multipart_form_data_parser 解析数据，并将解析结果存储在 ret 中
      auto ret = multipart_form_data_parser.parse(
          buf + pos, read_size, multipart_receiver, multipart_header);
      // 如果解析失败，则返回 false
      if (!ret) { return false; }
      // 更新 pos 的值
      pos += read_size;
    }
    // 返回 true
    return true;
    */
    // 调用 multipart_form_data_parser 解析数据，并返回结果
    return multipart_form_data_parser.parse(buf, n, multipart_receiver,
                                            multipart_header);
  };
} else {
  // 如果不是 multipart 类型的数据，则将数据传递给 receiver
  out = [receiver](const char *buf, size_t n, uint64_t /*off*/,
                   uint64_t /*len*/) { return receiver(buf, n); };
}

// 如果请求方法为 "DELETE" 并且没有设置 "Content-Length" 头部，则返回 true
if (req.method == "DELETE" && !req.has_header("Content-Length")) {
  return true;
}

// 读取请求内容并处理
if (!detail::read_content(strm, req, payload_max_length_, res.status, nullptr,
// 检查是否输出和真实路径匹配，如果不匹配则返回false
if (!path_match(req.path, entry.mount_point, out, true)) {
    return false;
}

// 如果请求是多部分表单数据，则检查是否有效，如果无效则返回400错误
if (req.is_multipart_form_data()) {
    if (!multipart_form_data_parser.is_valid()) {
        res.status = 400;
        return false;
    }
}

// 返回true表示处理文件请求成功
return true;
}

// 处理文件请求的函数
inline bool Server::handle_file_request(const Request &req, Response &res,
                                        bool head) {
  // 遍历基本目录列表
  for (const auto &entry : base_dirs_) {
    // 前缀匹配
    if (!req.path.compare(0, entry.mount_point.size(), entry.mount_point)) {
      // 获取子路径
      std::string sub_path = "/" + req.path.substr(entry.mount_point.size());
      // 检查子路径是否有效
      if (detail::is_valid_path(sub_path)) {
        // 拼接完整的文件路径
        auto path = entry.base_dir + sub_path;
        // 如果路径以斜杠结尾，则添加默认的文件名index.html
        if (path.back() == '/') { path += "index.html"; }

        // 如果路径对应的是文件
        if (detail::is_file(path)) {
          // 读取文件内容到响应体
          detail::read_file(path, res.body);
          // 查找文件类型并设置Content-Type响应头
          auto type =
              detail::find_content_type(path, file_extension_and_mimetype_map_);
          if (type) { res.set_header("Content-Type", type); }
          // 设置其他自定义的响应头
          for (const auto &kv : entry.headers) {
            res.set_header(kv.first.c_str(), kv.second);
          }
          // 根据请求是否包含Range头部设置响应状态码
          res.status = req.has_header("Range") ? 206 : 200;
          // 如果不是HEAD请求且存在文件请求处理函数，则调用处理函数
          if (!head && file_request_handler_) {
            file_request_handler_(req, res);
          }
          // 返回处理成功
          return true;
        }
      }
    }
// 创建服务器套接字，用于监听指定主机和端口的连接请求
inline socket_t
Server::create_server_socket(const std::string &host, int port,
                             int socket_flags,
                             SocketOptions socket_options) const {
  // 调用内部函数创建套接字
  return detail::create_socket(
      host, std::string(), port, address_family_, socket_flags, tcp_nodelay_,
      std::move(socket_options),
      // 匿名函数，用于绑定套接字并监听连接请求
      [](socket_t sock, struct addrinfo &ai) -> bool {
        // 绑定套接字到指定地址
        if (::bind(sock, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen))) {
          return false;
        }
        // 监听套接字连接请求
        if (::listen(sock, CPPHTTPLIB_LISTEN_BACKLOG)) { return false; }
        return true;
      });
}
// 在服务器对象内部绑定主机和端口，并设置套接字标志
inline int Server::bind_internal(const std::string &host, int port,
                                 int socket_flags) {
  // 如果服务器套接字无效，则返回-1
  if (!is_valid()) { return -1; }

  // 创建服务器套接字
  svr_sock_ = create_server_socket(host, port, socket_flags, socket_options_);
  // 如果创建失败，则返回-1
  if (svr_sock_ == INVALID_SOCKET) { return -1; }

  // 如果端口为0，则获取套接字的本地地址信息
  if (port == 0) {
    struct sockaddr_storage addr;
    socklen_t addr_len = sizeof(addr);
    // 获取套接字的本地地址信息失败，则返回-1
    if (getsockname(svr_sock_, reinterpret_cast<struct sockaddr *>(&addr),
                    &addr_len) == -1) {
      return -1;
    }
    // 如果地址族为IPv4，则返回端口号
    if (addr.ss_family == AF_INET) {
      return ntohs(reinterpret_cast<struct sockaddr_in *>(&addr)->sin_port);
    } 
    // 如果地址族为IPv6，则返回端口号
    else if (addr.ss_family == AF_INET6) {
      return ntohs(reinterpret_cast<struct sockaddr_in6 *>(&addr)->sin6_port);
    } 
    // 其他情况返回-1
    else {
      return -1;
    }
  }
    }
  } else {
    return port;
  }
}

// 内联函数，用于服务器监听
inline bool Server::listen_internal() {
  auto ret = true; // 初始化返回值为 true
  is_running_ = true; // 设置服务器运行状态为 true
  auto se = detail::scope_exit([&]() { is_running_ = false; }); // 创建一个作用域退出对象，用于在函数退出时设置服务器运行状态为 false

  {
    std::unique_ptr<TaskQueue> task_queue(new_task_queue()); // 创建一个任务队列的智能指针

    while (svr_sock_ != INVALID_SOCKET) { // 当服务器套接字不为无效套接字时循环
#ifndef _WIN32
      if (idle_interval_sec_ > 0 || idle_interval_usec_ > 0) { // 如果空闲间隔秒数大于 0 或者空闲间隔微秒数大于 0
#endif
        auto val = detail::select_read(svr_sock_, idle_interval_sec_, // 调用 select_read 函数，等待服务器套接字可读
                                       idle_interval_usec_);
        if (val == 0) { // 如果val等于0，表示超时
          task_queue->on_idle(); // 调用任务队列的on_idle方法
          continue; // 继续下一次循环
        }
#ifndef _WIN32
      }
#endif
      // 接受客户端连接，创建新的socket
      socket_t sock = accept(svr_sock_, nullptr, nullptr);

      // 如果接受连接失败
      if (sock == INVALID_SOCKET) {
        // 如果错误码是EMFILE，表示进程打开文件描述符的限制已经达到
        if (errno == EMFILE) {
          // 尝试在短暂休眠后再次接受新的连接
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          continue; // 继续下一次循环
        } else if (errno == EINTR || errno == EAGAIN) {
          continue; // 继续下一次循环
        }
        // 如果服务器socket仍然有效，则关闭服务器socket
        if (svr_sock_ != INVALID_SOCKET) {
          detail::close_socket(svr_sock_);
      // 如果连接超时，则返回 false
      ret = false;
    } else {
      // 服务器套接字被用户关闭
      ; // The server socket was closed by user.
    }
    // 跳出循环
    break;
  }

  {
    // 根据操作系统设置读取超时时间
#ifdef _WIN32
    // 如果是 Windows 系统，将毫秒级的超时时间转换为 uint32_t 类型，并设置套接字选项
    auto timeout = static_cast<uint32_t>(read_timeout_sec_ * 1000 +
                                         read_timeout_usec_ / 1000);
    setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (char *)&timeout,
               sizeof(timeout));
#else
    // 如果是其他系统，设置 timeval 结构体的秒和微秒，并设置套接字选项
    timeval tv;
    tv.tv_sec = static_cast<long>(read_timeout_sec_);
    tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec_);
    setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (char *)&tv, sizeof(tv));
#endif
  }
      {
#ifdef _WIN32
        // 如果是 Windows 系统，将写超时时间转换为毫秒，并设置 SO_SNDTIMEO 选项
        auto timeout = static_cast<uint32_t>(write_timeout_sec_ * 1000 +
                                             write_timeout_usec_ / 1000);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (char *)&timeout,
                   sizeof(timeout));
#else
        // 如果不是 Windows 系统，设置 timeval 结构体的秒和微秒，并设置 SO_SNDTIMEO 选项
        timeval tv;
        tv.tv_sec = static_cast<long>(write_timeout_sec_);
        tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec_);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (char *)&tv, sizeof(tv));
#endif
      }

      // 将处理和关闭套接字的操作加入任务队列
      task_queue->enqueue([this, sock]() { process_and_close_socket(sock); });
    }

    // 关闭任务队列
    task_queue->shutdown();
  }
```

  // 返回处理结果
  return ret;
}

// 路由处理函数，根据请求和响应进行处理
inline bool Server::routing(Request &req, Response &res, Stream &strm) {
  // 如果存在预处理函数，并且预处理函数处理结果为Handled，则返回true
  if (pre_routing_handler_ &&
      pre_routing_handler_(req, res) == HandlerResponse::Handled) {
    return true;
  }

  // 文件处理
  // 判断是否为HEAD请求
  bool is_head_request = req.method == "HEAD";
  // 如果是GET请求或者HEAD请求，并且处理文件请求成功，则返回true
  if ((req.method == "GET" || is_head_request) &&
      handle_file_request(req, res, is_head_request)) {
    return true;
  }

  // 如果请求中包含内容
  if (detail::expect_content(req)) {
    // 内容读取处理函数
    {
      // 创建一个ContentReader对象，其中包含两个lambda表达式，用于处理不同类型的内容
      ContentReader reader(
          [&](ContentReceiver receiver) {
            return read_content_with_content_receiver(
                strm, req, res, std::move(receiver), nullptr, nullptr);
          },
          [&](MultipartContentHeader header, ContentReceiver receiver) {
            return read_content_with_content_receiver(strm, req, res, nullptr,
                                                      std::move(header),
                                                      std::move(receiver));
          });

      // 如果请求方法为POST，则调用dispatch_request_for_content_reader处理请求
      if (req.method == "POST") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                post_handlers_for_content_reader_)) {
          return true;
        }
      } 
      // 如果请求方法为PUT，则调用dispatch_request_for_content_reader处理请求
      else if (req.method == "PUT") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
// 检查请求方法是否为 "GET"，如果是则调度对应的处理程序并返回 true
if (req.method == "GET") {
  if (dispatch_request_for_content_reader(
          req, res, std::move(reader),
          get_handlers_for_content_reader_)) {
    return true;
  }
} 
// 如果请求方法为 "PATCH"，则调度对应的处理程序并返回 true
else if (req.method == "PATCH") {
  if (dispatch_request_for_content_reader(
          req, res, std::move(reader),
          patch_handlers_for_content_reader_)) {
    return true;
  }
} 
// 如果请求方法为 "DELETE"，则调度对应的处理程序并返回 true
else if (req.method == "DELETE") {
  if (dispatch_request_for_content_reader(
          req, res, std::move(reader),
          delete_handlers_for_content_reader_)) {
    return true;
  }
}

// 将内容读取到 `req.body` 中，如果读取失败则返回 false
if (!read_content(strm, req, res)) { return false; }
  }

  // Regular handler
  // 如果请求方法是 GET 或 HEAD，则调用 dispatch_request 函数处理请求
  if (req.method == "GET" || req.method == "HEAD") {
    return dispatch_request(req, res, get_handlers_);
  } else if (req.method == "POST") {
    // 如果请求方法是 POST，则调用 dispatch_request 函数处理请求
    return dispatch_request(req, res, post_handlers_);
  } else if (req.method == "PUT") {
    // 如果请求方法是 PUT，则调用 dispatch_request 函数处理请求
    return dispatch_request(req, res, put_handlers_);
  } else if (req.method == "DELETE") {
    // 如果请求方法是 DELETE，则调用 dispatch_request 函数处理请求
    return dispatch_request(req, res, delete_handlers_);
  } else if (req.method == "OPTIONS") {
    // 如果请求方法是 OPTIONS，则调用 dispatch_request 函数处理请求
    return dispatch_request(req, res, options_handlers_);
  } else if (req.method == "PATCH") {
    // 如果请求方法是 PATCH，则调用 dispatch_request 函数处理请求
    return dispatch_request(req, res, patch_handlers_);
  }

  // 如果请求方法不在上述条件中，则返回状态码 400
  res.status = 400;
  return false;
}
// 在给定的处理程序集合中分发请求，如果找到匹配的处理程序，则调用该处理程序并返回 true，否则返回 false
inline bool Server::dispatch_request(Request &req, Response &res, const Handlers &handlers) {
  // 遍历处理程序集合
  for (const auto &x : handlers) {
    // 获取处理程序的模式和处理函数
    const auto &pattern = x.first;
    const auto &handler = x.second;

    // 如果请求路径与模式匹配
    if (std::regex_match(req.path, req.matches, pattern)) {
      // 调用处理函数处理请求，并返回 true
      handler(req, res);
      return true;
    }
  }
  // 没有找到匹配的处理程序，返回 false
  return false;
}

// 根据请求的范围应用内容范围，如果范围大于 1，则生成多部分数据边界
inline void Server::apply_ranges(const Request &req, Response &res, std::string &content_type, std::string &boundary) {
  // 如果请求的范围大于 1
  if (req.ranges.size() > 1) {
    // 生成多部分数据边界
    boundary = detail::make_multipart_data_boundary();
    // 在响应头中查找"Content-Type"字段
    auto it = res.headers.find("Content-Type");
    // 如果找到了"Content-Type"字段
    if (it != res.headers.end()) {
      // 将字段值赋给content_type变量
      content_type = it->second;
      // 删除该字段
      res.headers.erase(it);
    }

    // 添加"Content-Type"字段到响应头中，值为"multipart/byteranges; boundary=" + boundary
    res.headers.emplace("Content-Type",
                        "multipart/byteranges; boundary=" + boundary);
  }

  // 获取请求和响应的编码类型
  auto type = detail::encoding_type(req, res);

  // 如果响应体为空
  if (res.body.empty()) {
    // 如果响应体长度大于0
    if (res.content_length_ > 0) {
      size_t length = 0;
      // 如果请求中没有范围
      if (req.ranges.empty()) {
        // 将响应体长度赋给length
        length = res.content_length_;
      } 
      // 如果请求中只有一个范围
      else if (req.ranges.size() == 1) {
        // 获取偏移量
        auto offsets =
        // 根据请求和响应的内容长度获取范围的偏移和长度
        detail::get_range_offset_and_length(req, res.content_length_, 0);
        // 获取偏移量
        auto offset = offsets.first;
        // 获取长度
        length = offsets.second;
        // 创建 Content-Range 头字段
        auto content_range = detail::make_content_range_header_field(
            offset, length, res.content_length_);
        // 设置 Content-Range 头字段
        res.set_header("Content-Range", content_range);
      } else {
        // 如果不是范围请求，获取多部分范围数据的长度
        length = detail::get_multipart_ranges_data_length(req, res, boundary,
                                                          content_type);
      }
      // 设置 Content-Length 头字段
      res.set_header("Content-Length", std::to_string(length));
    } else {
      // 如果响应有内容提供者
      if (res.content_provider_) {
        // 如果是分块内容提供者，设置 Transfer-Encoding 头字段为 chunked
        if (res.is_chunked_content_provider_) {
          res.set_header("Transfer-Encoding", "chunked");
          // 如果编码类型是 Gzip，设置 Content-Encoding 头字段为 gzip
          if (type == detail::EncodingType::Gzip) {
            res.set_header("Content-Encoding", "gzip");
          } 
          // 如果编码类型是 Brotli，设置 Content-Encoding 头字段为 br
          else if (type == detail::EncodingType::Brotli) {
            res.set_header("Content-Encoding", "br");
          }
    // 如果请求中的范围为空，则不做处理
    if (req.ranges.empty()) {
      ;
    } 
    // 如果请求中的范围只有一个
    else if (req.ranges.size() == 1) {
      // 获取范围的偏移量和长度
      auto offsets =
          detail::get_range_offset_and_length(req, res.body.size(), 0);
      auto offset = offsets.first;
      auto length = offsets.second;
      // 生成Content-Range头字段
      auto content_range = detail::make_content_range_header_field(
          offset, length, res.body.size());
      // 设置响应头中的Content-Range字段
      res.set_header("Content-Range", content_range);
      // 如果偏移量小于响应体的大小，则截取响应体
      if (offset < res.body.size()) {
        res.body = res.body.substr(offset, length);
      } 
      // 如果偏移量大于等于响应体的大小，则清空响应体并设置状态码为416
      else {
        res.body.clear();
        res.status = 416;
      }
    } else {
      // 如果不是multipart类型的请求，处理其他类型的请求
      std::string data; // 声明一个字符串变量data
      // 调用make_multipart_ranges_data函数生成multipart范围数据，如果成功则交换res.body和data，否则清空res.body并设置状态码为416
      if (detail::make_multipart_ranges_data(req, res, boundary, content_type,
                                             data)) {
        res.body.swap(data);
      } else {
        res.body.clear();
        res.status = 416;
      }
    }

    // 如果请求的类型不是None，则进行压缩处理
    if (type != detail::EncodingType::None) {
      std::unique_ptr<detail::compressor> compressor; // 声明一个压缩器指针
      std::string content_encoding; // 声明一个内容编码字符串

      // 如果请求的类型是Gzip，并且支持zlib，则创建一个gzip压缩器，并设置内容编码为gzip
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        compressor = detail::make_unique<detail::gzip_compressor>();
        content_encoding = "gzip";
#endif
      } else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        // 如果编码类型是Brotli，并且支持Brotli压缩，则创建Brotli压缩器，并设置内容编码为"br"
        compressor = detail::make_unique<detail::brotli_compressor>();
        content_encoding = "br";
#endif
      }

      // 如果存在压缩器
      if (compressor) {
        // 创建一个空的字符串用于存储压缩后的数据
        std::string compressed;
        // 使用压缩器对响应体进行压缩，并将压缩后的数据追加到compressed中
        if (compressor->compress(res.body.data(), res.body.size(), true,
                                 [&](const char *data, size_t data_len) {
                                   compressed.append(data, data_len);
                                   return true;
                                 })) {
          // 将压缩后的数据替换原来的响应体
          res.body.swap(compressed);
          // 设置响应头中的Content-Encoding字段为压缩算法的类型
          res.set_header("Content-Encoding", content_encoding);
        }
      }
    }
    // 将响应体的大小转换为字符串
    auto length = std::to_string(res.body.size());
    // 设置响应头中的Content-Length字段为响应体的大小
    res.set_header("Content-Length", length);
  }
}

// 为内容读取器分发请求
inline bool Server::dispatch_request_for_content_reader(
    Request &req, Response &res, ContentReader content_reader,
    const HandlersForContentReader &handlers) {
  // 遍历内容读取器的处理函数
  for (const auto &x : handlers) {
    const auto &pattern = x.first;  // 获取URL匹配模式
    const auto &handler = x.second;  // 获取处理函数

    // 如果请求路径与URL匹配模式匹配
    if (std::regex_match(req.path, req.matches, pattern)) {
      // 调用对应的处理函数处理请求
      handler(req, res, content_reader);
      return true;  // 处理成功，返回true
    }
  }
  return false;  // 未找到匹配的处理函数，返回false
}
// 在服务器处理请求时，处理请求流，关闭连接标志，连接是否关闭标志，设置请求的函数
inline bool
Server::process_request(Stream &strm, bool close_connection,
                        bool &connection_closed,
                        const std::function<void(Request &)> &setup_request) {
  // 创建一个大小为2048的字符数组作为缓冲区
  std::array<char, 2048> buf{};

  // 使用流行读取器从流中读取一行数据
  detail::stream_line_reader line_reader(strm, buf.data(), buf.size());

  // 如果无法读取到数据，表示客户端已关闭连接
  if (!line_reader.getline()) { return false; }

  // 创建请求和响应对象
  Request req;
  Response res;

  // 设置响应的版本为HTTP/1.1
  res.version = "HTTP/1.1";

  // 遍历默认头部，如果响应头部中没有该头部，则插入该头部
  for (const auto &header : default_headers_) {
    if (res.headers.find(header.first) == res.headers.end()) {
      res.headers.insert(header);
    }
  }

#ifdef _WIN32
  // 如果是在 Windows 平台，需要增加 FD_SETSIZE 的大小（libzmq），或者动态增加（MySQL）。
#else
#ifndef CPPHTTPLIB_USE_POLL
  // 如果没有使用 CPPHTTPLIB_USE_POLL 宏定义，且套接字文件描述符超过了 FD_SETSIZE...
  if (strm.socket() >= FD_SETSIZE) {
    // 创建一个空的 Headers 对象
    Headers dummy;
    // 读取请求头部信息
    detail::read_headers(strm, dummy);
    // 设置响应状态码为 500
    res.status = 500;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }
#endif
#endif

  // 检查请求 URI 是否超过了限制长度
  if (line_reader.size() > CPPHTTPLIB_REQUEST_URI_MAX_LENGTH) {
    // 创建一个空的 Headers 对象
    Headers dummy;
    // 读取请求头部信息
    detail::read_headers(strm, dummy);
    // 设置响应状态码为414
    res.status = 414;
    // 调用write_response函数，返回响应并关闭连接
    return write_response(strm, close_connection, req, res);
  }

  // 解析请求行和头部
  if (!parse_request_line(line_reader.ptr(), req) ||
      !detail::read_headers(strm, req.headers)) {
    // 如果解析失败，设置响应状态码为400
    res.status = 400;
    // 调用write_response函数，返回响应并关闭连接
    return write_response(strm, close_connection, req, res);
  }

  // 如果请求头中的Connection字段值为close，表示连接将被关闭
  if (req.get_header_value("Connection") == "close") {
    connection_closed = true;
  }

  // 如果请求版本为HTTP/1.0且Connection字段值不为Keep-Alive，表示连接将被关闭
  if (req.version == "HTTP/1.0" &&
      req.get_header_value("Connection") != "Keep-Alive") {
    connection_closed = true;
  }
  # 获取远程客户端的 IP 地址和端口号，并设置到请求对象中
  strm.get_remote_ip_and_port(req.remote_addr, req.remote_port);
  req.set_header("REMOTE_ADDR", req.remote_addr);
  req.set_header("REMOTE_PORT", std::to_string(req.remote_port));

  # 获取本地服务器的 IP 地址和端口号，并设置到请求对象中
  strm.get_local_ip_and_port(req.local_addr, req.local_port);
  req.set_header("LOCAL_ADDR", req.local_addr);
  req.set_header("LOCAL_PORT", std::to_string(req.local_port));

  # 如果请求头中包含 Range 字段，则解析该字段的值，并设置到请求对象中
  if (req.has_header("Range")) {
    const auto &range_header_value = req.get_header_value("Range");
    if (!detail::parse_range_header(range_header_value, req.ranges)) {
      # 如果解析失败，则返回 416 状态码
      res.status = 416;
      return write_response(strm, close_connection, req, res);
    }
  }

  # 如果有设置请求的预处理函数，则执行该函数
  if (setup_request) { setup_request(req); }

  # 如果请求头中包含 Expect 字段，并且值为 "100-continue"，则返回 100 状态码
  if (req.get_header_value("Expect") == "100-continue") {
    auto status = 100;
    // 如果存在 expect_100_continue_handler_，则调用该处理函数处理请求和响应
    if (expect_100_continue_handler_) {
      status = expect_100_continue_handler_(req, res);
    }
    // 根据处理结果进行不同的操作
    switch (status) {
    // 如果状态码为100或417，则直接返回对应的响应
    case 100:
    case 417:
      strm.write_format("HTTP/1.1 %d %s\r\n\r\n", status,
                        detail::status_message(status));
      break;
    // 其他情况下调用 write_response 函数处理响应
    default: return write_response(strm, close_connection, req, res);
    }
  }

  // 路由处理
  bool routed = false;
  // 根据宏定义选择是否使用异常处理
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
  routed = routing(req, res, strm);
#else
  // 使用异常处理来进行路由处理
  try {
    routed = routing(req, res, strm);
  } catch (std::exception &e) {
    # 如果捕获到标准异常，执行以下代码
    if (exception_handler_) {
      # 获取当前异常的指针
      auto ep = std::current_exception();
      # 调用异常处理函数，传入请求、响应和异常指针
      exception_handler_(req, res, ep);
      # 设置路由标志为已处理
      routed = true;
    } else {
      # 如果没有异常处理函数，设置响应状态为500
      res.status = 500;
      # 定义字符串变量val
      std::string val;
      # 获取异常信息
      auto s = e.what();
      # 遍历异常信息的每个字符
      for (size_t i = 0; s[i]; i++) {
        # 根据字符类型进行处理
        switch (s[i]) {
        case '\r': val += "\\r"; break;
        case '\n': val += "\\n"; break;
        default: val += s[i]; break;
        }
      }
      # 设置响应头部，将异常信息存储在"EXCEPTION_WHAT"字段中
      res.set_header("EXCEPTION_WHAT", val);
    }
  } catch (...) {
    # 如果捕获到其他类型的异常，执行以下代码
    if (exception_handler_) {
// 获取当前的异常对象
auto ep = std::current_exception();
// 调用异常处理函数处理请求和响应对象以及异常对象
exception_handler_(req, res, ep);
// 设置路由标志为true
routed = true;
// 如果没有发生异常
} else {
  // 设置响应状态码为500
  res.status = 500;
  // 设置响应头部信息
  res.set_header("EXCEPTION_WHAT", "UNKNOWN");
}

// 如果已经路由
if (routed) {
  // 如果响应状态码为-1，则根据请求范围设置响应状态码为200或206
  if (res.status == -1) { res.status = req.ranges.empty() ? 200 : 206; }
  // 调用写响应内容的函数
  return write_response_with_content(strm, close_connection, req, res);
} else {
  // 如果响应状态码为-1，则设置响应状态码为404
  if (res.status == -1) { res.status = 404; }
  // 调用写响应的函数
  return write_response(strm, close_connection, req, res);
}

// 返回服务器是否有效的标志
inline bool Server::is_valid() const { return true; }
// 在服务器类中处理并关闭套接字
inline bool Server::process_and_close_socket(socket_t sock) {
  // 调用细节函数处理服务器套接字
  auto ret = detail::process_server_socket(
      svr_sock_, sock, keep_alive_max_count_, keep_alive_timeout_sec_,
      read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
      write_timeout_usec_,
      // 使用 lambda 表达式处理请求
      [this](Stream &strm, bool close_connection, bool &connection_closed) {
        return process_request(strm, close_connection, connection_closed,
                               nullptr);
      });

  // 关闭套接字的读写功能
  detail::shutdown_socket(sock);
  // 关闭套接字
  detail::close_socket(sock);
  // 返回处理结果
  return ret;
}

// HTTP 客户端实现
inline ClientImpl::ClientImpl(const std::string &host)
    // 使用默认端口号 80 初始化客户端
    : ClientImpl(host, 80, std::string(), std::string()) {}
// 使用给定的主机和端口初始化客户端实现，调用另一个构造函数
inline ClientImpl::ClientImpl(const std::string &host, int port)
    : ClientImpl(host, port, std::string(), std::string()) {}

// 使用给定的主机、端口、客户端证书路径和客户端密钥路径初始化客户端实现
inline ClientImpl::ClientImpl(const std::string &host, int port,
                              const std::string &client_cert_path,
                              const std::string &client_key_path)
    : host_(host), port_(port),
      host_and_port_(adjust_host_string(host) + ":" + std::to_string(port)),
      client_cert_path_(client_cert_path), client_key_path_(client_key_path) {}

// 客户端实现的析构函数，关闭套接字并释放资源
inline ClientImpl::~ClientImpl() {
  std::lock_guard<std::mutex> guard(socket_mutex_);
  shutdown_socket(socket_);
  close_socket(socket_);
}

// 检查客户端实现是否有效
inline bool ClientImpl::is_valid() const { return true; }

// 复制另一个客户端实现的设置
inline void ClientImpl::copy_settings(const ClientImpl &rhs) {
  client_cert_path_ = rhs.client_cert_path_;
  # 将 rhs 对象的 client_key_path_ 赋值给当前对象的 client_key_path_
  client_key_path_ = rhs.client_key_path_;
  # 将 rhs 对象的 connection_timeout_sec_ 赋值给当前对象的 connection_timeout_sec_
  connection_timeout_sec_ = rhs.connection_timeout_sec_;
  # 将 rhs 对象的 read_timeout_sec_ 赋值给当前对象的 read_timeout_sec_
  read_timeout_sec_ = rhs.read_timeout_sec_;
  # 将 rhs 对象的 read_timeout_usec_ 赋值给当前对象的 read_timeout_usec_
  read_timeout_usec_ = rhs.read_timeout_usec_;
  # 将 rhs 对象的 write_timeout_sec_ 赋值给当前对象的 write_timeout_sec_
  write_timeout_sec_ = rhs.write_timeout_sec_;
  # 将 rhs 对象的 write_timeout_usec_ 赋值给当前对象的 write_timeout_usec_
  write_timeout_usec_ = rhs.write_timeout_usec_;
  # 将 rhs 对象的 basic_auth_username_ 赋值给当前对象的 basic_auth_username_
  basic_auth_username_ = rhs.basic_auth_username_;
  # 将 rhs 对象的 basic_auth_password_ 赋值给当前对象的 basic_auth_password_
  basic_auth_password_ = rhs.basic_auth_password_;
  # 将 rhs 对象的 bearer_token_auth_token_ 赋值给当前对象的 bearer_token_auth_token_
  bearer_token_auth_token_ = rhs.bearer_token_auth_token_;
  # 如果支持 OpenSSL，则将 rhs 对象的 digest_auth_username_ 赋值给当前对象的 digest_auth_username_
  # 如果支持 OpenSSL，则将 rhs 对象的 digest_auth_password_ 赋值给当前对象的 digest_auth_password_
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  digest_auth_username_ = rhs.digest_auth_username_;
  digest_auth_password_ = rhs.digest_auth_password_;
#endif
  # 将 rhs 对象的 keep_alive_ 赋值给当前对象的 keep_alive_
  keep_alive_ = rhs.keep_alive_;
  # 将 rhs 对象的 follow_location_ 赋值给当前对象的 follow_location_
  follow_location_ = rhs.follow_location_;
  # 将 rhs 对象的 url_encode_ 赋值给当前对象的 url_encode_
  url_encode_ = rhs.url_encode_;
  # 将 rhs 对象的 address_family_ 赋值给当前对象的 address_family_
  address_family_ = rhs.address_family_;
  # 将 rhs 对象的 tcp_nodelay_ 赋值给当前对象的 tcp_nodelay_
  tcp_nodelay_ = rhs.tcp_nodelay_;
  # 将 rhs 对象的 socket_options_ 赋值给当前对象的 socket_options_
  socket_options_ = rhs.socket_options_;
  # 将 rhs 对象的 compress_ 赋值给当前对象的 compress_
  compress_ = rhs.compress_;
  # 将 rhs 对象的 decompress_ 赋值给当前对象的 decompress_
  decompress_ = rhs.decompress_;
  # 将 rhs 对象的 interface_ 赋值给当前对象的 interface_
  interface_ = rhs.interface_;
  # 将 rhs 对象的 proxy_host_ 赋值给当前对象的 proxy_host_
  proxy_host_ = rhs.proxy_host_;
  # 将 rhs 对象的 proxy_port_ 赋值给当前对象的 proxy_port_
  proxy_port_ = rhs.proxy_port_;
  # 将 rhs 对象的 proxy_basic_auth_username_ 赋值给当前对象的 proxy_basic_auth_username_
  proxy_basic_auth_username_ = rhs.proxy_basic_auth_username_;
  # 将 rhs 对象的 proxy_basic_auth_password_ 赋值给当前对象的 proxy_basic_auth_password_
  proxy_basic_auth_password_ = rhs.proxy_basic_auth_password_;
  # 将 rhs 对象的 proxy_bearer_token_auth_token_ 赋值给当前对象的 proxy_bearer_token_auth_token_
  proxy_bearer_token_auth_token_ = rhs.proxy_bearer_token_auth_token_;
  # 如果支持 OpenSSL，则将 rhs 对象的 proxy_digest_auth_username_ 赋值给当前对象的 proxy_digest_auth_username_
  proxy_digest_auth_username_ = rhs.proxy_digest_auth_username_;
  # 如果支持 OpenSSL，则将 rhs 对象的 proxy_digest_auth_password_ 赋值给当前对象的 proxy_digest_auth_password_
  proxy_digest_auth_password_ = rhs.proxy_digest_auth_password_;
  # 如果支持 OpenSSL，则将 rhs 对象的 ca_cert_file_path_ 赋值给当前对象的 ca_cert_file_path_
  ca_cert_file_path_ = rhs.ca_cert_file_path_;
  # 如果支持 OpenSSL，则将 rhs 对象的 ca_cert_dir_path_ 赋值给当前对象的 ca_cert_dir_path_
  ca_cert_dir_path_ = rhs.ca_cert_dir_path_;
  # 如果支持 OpenSSL，则将 rhs 对象的 ca_cert_store_ 赋值给当前对象的 ca_cert_store_
  ca_cert_store_ = rhs.ca_cert_store_;
  # 如果支持 OpenSSL，则将 rhs 对象的 server_certificate_verification_ 赋值给当前对象的 server_certificate_verification_
  server_certificate_verification_ = rhs.server_certificate_verification_;
  # 将 rhs 对象的 logger_ 赋值给当前对象的 logger_
  logger_ = rhs.logger_;
}

// 创建客户端套接字
inline socket_t ClientImpl::create_client_socket(Error &error) const {
  // 如果代理主机和代理端口不为空，则创建带有代理的客户端套接字
  if (!proxy_host_.empty() && proxy_port_ != -1) {
    return detail::create_client_socket(
        proxy_host_, std::string(), proxy_port_, address_family_, tcp_nodelay_,
        socket_options_, connection_timeout_sec_, connection_timeout_usec_,
        read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
        write_timeout_usec_, interface_, error);
  }

  // 检查是否为主机指定了自定义 IP
  std::string ip;
  auto it = addr_map_.find(host_);
  if (it != addr_map_.end()) ip = it->second;

  // 创建不带代理的客户端套接字
  return detail::create_client_socket(
      host_, ip, port_, address_family_, tcp_nodelay_, socket_options_,
      connection_timeout_sec_, connection_timeout_usec_, read_timeout_sec_,
      read_timeout_usec_, write_timeout_sec_, write_timeout_usec_, interface_,
// 创建并连接套接字，如果创建失败则返回false
inline bool ClientImpl::create_and_connect_socket(Socket &socket,
                                                  Error &error) {
  // 创建客户端套接字
  auto sock = create_client_socket(error);
  // 如果创建失败，返回false
  if (sock == INVALID_SOCKET) { return false; }
  // 将创建的套接字赋值给socket
  socket.sock = sock;
  return true;
}

// 关闭SSL连接
inline void ClientImpl::shutdown_ssl(Socket & /*socket*/,
                                     bool /*shutdown_gracefully*/) {
  // 如果有其他线程发出的请求正在处理中，那么这是一个线程不安全的竞争条件，因为单独的ssl*对象不是线程安全的
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());
}

// 关闭套接字
inline void ClientImpl::shutdown_socket(Socket &socket) {
// 如果套接字无效，则直接返回，不执行后续操作
if (socket.sock == INVALID_SOCKET) { return; }

// 关闭套接字，确保没有请求在使用该套接字
detail::shutdown_socket(socket.sock);
}

// 关闭套接字
inline void ClientImpl::close_socket(Socket &socket) {
  // 如果有其他线程中有请求在处理中，关闭套接字可能会导致错误
  // 但通常情况下，它们会收到套接字关闭的错误，但仍然是一个 bug
  // 因为极少数情况下，操作系统可能会重新分配套接字 ID 用于新的套接字
  // 这样一来，它们将在一个不同于预期的活动套接字上进行操作
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());

  // 如果在 SSL 仍然活动的情况下关闭套接字也是一个 bug
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  assert(socket.ssl == nullptr);
#endif

  // 如果套接字无效，则直接返回，不执行后续操作
  if (socket.sock == INVALID_SOCKET) { return; }

  // 关闭套接字
  detail::close_socket(socket.sock);
// 将 socket.sock 设置为无效的套接字
socket.sock = INVALID_SOCKET;
}

// 从流中读取响应行，解析请求和响应
inline bool ClientImpl::read_response_line(Stream &strm, const Request &req, Response &res) {
  // 创建一个大小为2048的字符数组作为缓冲区
  std::array<char, 2048> buf{};

  // 创建一个流行读取器对象，用于从流中读取一行数据
  detail::stream_line_reader line_reader(strm, buf.data(), buf.size());

  // 如果无法读取到数据，则返回false
  if (!line_reader.getline()) { return false; }

  // 根据不同的编译选项，创建不同的正则表达式对象来匹配响应行
#ifdef CPPHTTPLIB_ALLOW_LF_AS_LINE_TERMINATOR
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r\n");
#else
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r?\n");
#endif

  // 使用正则表达式匹配响应行
  std::cmatch m;
  if (!std::regex_match(line_reader.ptr(), m, re)) {
    // 如果无法匹配响应行，则返回请求方法是否为"CONNECT"
    return req.method == "CONNECT";
  // 设置响应的版本号为正则表达式匹配的第一个分组
  res.version = std::string(m[1]);
  // 设置响应的状态码为正则表达式匹配的第二个分组转换为整数
  res.status = std::stoi(std::string(m[2]));
  // 设置响应的原因为正则表达式匹配的第三个分组
  res.reason = std::string(m[3]);

  // 忽略 '100 Continue' 的响应
  while (res.status == 100) {
    // 如果无法读取下一行，则返回 false
    if (!line_reader.getline()) { return false; } // CRLF
    // 如果无法读取下一行，则返回 false
    if (!line_reader.getline()) { return false; } // next response line

    // 如果当前行无法匹配正则表达式，则返回 false
    if (!std::regex_match(line_reader.ptr(), m, re)) { return false; }
    // 设置响应的版本号为正则表达式匹配的第一个分组
    res.version = std::string(m[1]);
    // 设置响应的状态码为正则表达式匹配的第二个分组转换为整数
    res.status = std::stoi(std::string(m[2]));
    // 设置响应的原因为正则表达式匹配的第三个分组
    res.reason = std::string(m[3]);
  }

  // 返回 true 表示发送成功
  return true;
}

// 发送请求并获取响应
inline bool ClientImpl::send(Request &req, Response &res, Error &error) {
// 使用递归互斥锁来保护请求操作，确保在同一时间只有一个线程可以访问请求
std::lock_guard<std::recursive_mutex> request_mutex_guard(request_mutex_);

// 调用send_函数发送请求，并将返回值存储在ret中
auto ret = send_(req, res, error);

// 如果发生SSLPeerCouldBeClosed_错误，断言ret为false，并再次调用send_函数发送请求
if (error == Error::SSLPeerCouldBeClosed_) {
  assert(!ret);
  ret = send_(req, res, error);
}

// 返回send_函数的返回值
return ret;
}

// 定义send_函数，用于发送请求
inline bool ClientImpl::send_(Request &req, Response &res, Error &error) {
  {
    // 使用互斥锁来保护socket的访问
    std::lock_guard<std::mutex> guard(socket_mutex_);

    // 立即将socket_should_be_closed_when_request_is_done_设置为false，如果在请求结束时它被设置为true，我们知道另一个线程指示我们关闭socket
    socket_should_be_closed_when_request_is_done_ = false;

    // 初始化is_alive为false
    auto is_alive = false;

    // 如果socket是打开的，调用detail::is_socket_alive函数检查socket是否存活
    if (socket_.is_open()) {
      is_alive = detail::is_socket_alive(socket_.sock);
      // 如果连接已经关闭，则尝试避免 SIGPIPE 信号，通过非优雅地关闭连接
      // 同时，由于已经锁定了 request_mutex_，所以不会有其他线程中的请求在进行中，可以立即关闭所有内容
      const bool shutdown_gracefully = false;
      shutdown_ssl(socket_, shutdown_gracefully);
      shutdown_socket(socket_);
      close_socket(socket_);
    }

    if (!is_alive) {
      // 如果连接已经关闭，则尝试创建并连接新的 socket
      if (!create_and_connect_socket(socket_, error)) { return false; }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 如果使用 SSL，则进行 SSL 客户端的设置
      if (is_ssl()) {
        auto &scli = static_cast<SSLClient &>(*this);
        // 如果使用代理，则设置代理主机和端口
        if (!proxy_host_.empty() && proxy_port_ != -1) {
// 定义一个布尔变量 success，并初始化为 false
auto success = false;
// 如果连接失败，则返回 success
if (!scli.connect_with_proxy(socket_, res, success, error)) {
  return success;
}

// 如果 SSL 初始化失败，则返回 false
if (!scli.initialize_ssl(socket_, error)) { 
  return false; 
}

// 如果当前环境支持多线程
#ifdef
  // 如果当前有其他请求正在使用该 socket，则断言当前请求来自同一个线程
  if (socket_requests_in_flight_ > 1) {
    assert(socket_requests_are_from_thread_ == std::this_thread::get_id());
  }
  // 增加正在使用该 socket 的请求数量
  socket_requests_in_flight_ += 1;
  // 记录当前请求来自哪个线程
  socket_requests_are_from_thread_ = std::this_thread::get_id();
}
  // 遍历默认请求头
  for (const auto &header : default_headers_) {
    // 如果请求中不包含默认请求头中的某个头部信息，则插入该头部信息
    if (req.headers.find(header.first) == req.headers.end()) {
      req.headers.insert(header);
    }
  }

  // 初始化返回值为false
  auto ret = false;
  // 如果不需要保持长连接，则关闭连接
  auto close_connection = !keep_alive_;

  // 创建一个作用域退出对象，用于在退出作用域时执行特定的操作
  auto se = detail::scope_exit([&]() {
    // 简要锁定互斥锁，以标记请求不再进行中
    std::lock_guard<std::mutex> guard(socket_mutex_);
    // 减少正在进行的请求数量
    socket_requests_in_flight_ -= 1;
    // 如果没有正在进行的请求，则重置相关状态
    if (socket_requests_in_flight_ <= 0) {
      assert(socket_requests_in_flight_ == 0);
      socket_requests_are_from_thread_ = std::thread::id();
    }

    // 如果应该在请求完成时关闭连接，或者需要关闭连接，则执行相应操作
    if (socket_should_be_closed_when_request_is_done_ || close_connection ||
// 如果返回值为假，则关闭 SSL 连接，关闭套接字，关闭连接
if (!ret) {
  shutdown_ssl(socket_, true);
  shutdown_socket(socket_);
  close_socket(socket_);
}

// 处理套接字，处理请求，返回结果
ret = process_socket(socket_, [&](Stream &strm) {
  return handle_request(strm, req, res, close_connection, error);
});

// 如果返回值为假且错误为成功，则将错误设置为未知错误
if (!ret) {
  if (error == Error::Success) { error = Error::Unknown; }
}

// 返回结果
return ret;
}

// 发送请求
inline Result ClientImpl::send(const Request &req) {
  // 复制请求
  auto req2 = req;
// 返回移动后的请求对象
  return send_(std::move(req2));
}

// 发送请求的内部实现
inline Result ClientImpl::send_(Request &&req) {
  // 创建响应对象
  auto res = detail::make_unique<Response>();
  // 初始化错误状态为成功
  auto error = Error::Success;
  // 发送请求并获取返回值
  auto ret = send(req, *res, error);
  // 返回结果对象，包括响应对象、错误状态和请求头
  return Result{ret ? std::move(res) : nullptr, error, std::move(req.headers)};
}

// 处理请求的内部实现
inline bool ClientImpl::handle_request(Stream &strm, Request &req,
                                       Response &res, bool close_connection,
                                       Error &error) {
  // 如果请求路径为空，设置错误状态为连接错误并返回false
  if (req.path.empty()) {
    error = Error::Connection;
    return false;
  }
  // 保存请求对象的副本
  auto req_save = req;
  # 声明布尔变量 ret
  bool ret;

  # 如果不是 SSL 连接，并且代理主机和端口都不为空
  if (!is_ssl() && !proxy_host_.empty() && proxy_port_ != -1) {
    # 复制请求对象
    auto req2 = req;
    # 修改请求路径为使用代理的路径
    req2.path = "http://" + host_and_port_ + req.path;
    # 处理修改后的请求
    ret = process_request(strm, req2, res, close_connection, error);
    # 将原始请求对象更新为修改后的请求对象
    req = req2;
    req.path = req_save.path;
  } else {
    # 否则，直接处理原始请求
    ret = process_request(strm, req, res, close_connection, error);
  }

  # 如果处理请求失败，则返回 false
  if (!ret) { return false; }

  # 如果响应状态码在 300 到 400 之间，并且允许重定向
  if (300 < res.status && res.status < 400 && follow_location_) {
    # 恢复原始请求对象
    req = req_save;
    # 发起重定向请求
    ret = redirect(req, res, error);
  }

  # 如果支持 OpenSSL，则执行以下代码
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 检查响应状态是否为 401 或 407，并且请求的授权次数小于 5
  if ((res.status == 401 || res.status == 407) &&
      req.authorization_count_ < 5) {
    // 判断是否为代理认证
    auto is_proxy = res.status == 407;
    // 获取对应的用户名和密码
    const auto &username =
        is_proxy ? proxy_digest_auth_username_ : digest_auth_username_;
    const auto &password =
        is_proxy ? proxy_digest_auth_password_ : digest_auth_password_;

    // 如果用户名和密码不为空
    if (!username.empty() && !password.empty()) {
      // 解析认证信息
      std::map<std::string, std::string> auth;
      if (detail::parse_www_authenticate(res, auth, is_proxy)) {
        // 创建新的请求对象，并增加授权次数
        Request new_req = req;
        new_req.authorization_count_ += 1;
        // 移除旧的认证头部，并插入新的摘要认证头部
        new_req.headers.erase(is_proxy ? "Proxy-Authorization"
                                       : "Authorization");
        new_req.headers.insert(detail::make_digest_authentication_header(
            req, auth, new_req.authorization_count_, detail::random_string(10),
            username, password, is_proxy));

        // 创建新的响应对象
        Response new_res;
        // 调用 send 函数发送新的请求，并将返回值赋给 ret
        ret = send(new_req, new_res, error);
        // 如果 ret 不为 0，则将 new_res 赋给 res
        if (ret) { res = new_res; }
      }
    }
  }
#endif

  // 返回 ret
  return ret;
}

// 重定向函数，处理重定向请求
inline bool ClientImpl::redirect(Request &req, Response &res, Error &error) {
  // 如果重定向次数已经达到最大值，返回错误并设置错误类型为 ExceedRedirectCount
  if (req.redirect_count_ == 0) {
    error = Error::ExceedRedirectCount;
    return false;
  }

  // 获取响应头中的 location 字段
  auto location = res.get_header_value("location");
  // 如果 location 为空，返回 false
  if (location.empty()) { return false; }
  // 定义一个静态的正则表达式，用于匹配 URL
  const static std::regex re(
      R"((?:(https?):)?(?://(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)?([^?#]*)(\?[^#]*)?(?:#.*)?)");

  // 创建一个 std::smatch 对象，用于存储正则匹配结果
  std::smatch m;
  // 如果给定的 location 字符串不匹配正则表达式，则返回 false
  if (!std::regex_match(location, m, re)) { return false; }

  // 根据当前是否为 SSL 连接，选择相应的协议
  auto scheme = is_ssl() ? "https" : "http";

  // 从正则匹配结果中获取下一个 URL 的协议、主机、端口、路径和查询参数
  auto next_scheme = m[1].str();
  auto next_host = m[2].str();
  if (next_host.empty()) { next_host = m[3].str(); }
  auto port_str = m[4].str();
  auto next_path = m[5].str();
  auto next_query = m[6].str();

  // 初始化下一个 URL 的端口号为当前端口号
  auto next_port = port_;
  // 如果端口号不为空，则将其转换为整数
  if (!port_str.empty()) {
    next_port = std::stoi(port_str);
  } else if (!next_scheme.empty()) {
    // 如果协议不为空，则根据协议选择默认端口号
    next_port = next_scheme == "https" ? 443 : 80;
  }
  }

  // 如果下一个 scheme 为空，则将其设为当前 scheme
  if (next_scheme.empty()) { next_scheme = scheme; }
  // 如果下一个 host 为空，则将其设为当前 host
  if (next_host.empty()) { next_host = host_; }
  // 如果下一个 path 为空，则将其设为根路径
  if (next_path.empty()) { next_path = "/"; }

  // 解码 URL 中的路径，并添加查询参数
  auto path = detail::decode_url(next_path, true) + next_query;

  // 如果下一个 scheme、host 和 port 与当前相同，则重定向
  if (next_scheme == scheme && next_host == host_ && next_port == port_) {
    return detail::redirect(*this, req, res, path, location, error);
  } else {
    // 如果下一个 scheme 是 https
    if (next_scheme == "https") {
      #ifdef CPPHTTPLIB_OPENSSL_SUPPORT
        // 创建 SSL 客户端对象
        SSLClient cli(next_host.c_str(), next_port);
        // 复制当前对象的设置
        cli.copy_settings(*this);
        // 如果有 CA 证书存储，则设置到 SSL 客户端对象中
        if (ca_cert_store_) { cli.set_ca_cert_store(ca_cert_store_); }
        // 重定向到 SSL 客户端
        return detail::redirect(cli, req, res, path, location, error);
      #else
        // 如果不支持 OpenSSL，则返回 false
        return false;
      #endif
    } else {
      // 创建一个新的客户端实现对象，连接到下一个主机和端口
      ClientImpl cli(next_host.c_str(), next_port);
      // 复制当前客户端的设置到新的客户端对象
      cli.copy_settings(*this);
      // 重定向请求到新的客户端对象，并返回重定向结果
      return detail::redirect(cli, req, res, path, location, error);
    }
  }
}

// 使用数据提供者向流中写入内容
inline bool ClientImpl::write_content_with_provider(Stream &strm,
                                                    const Request &req,
                                                    Error &error) {
  // 检查服务器是否正在关闭
  auto is_shutting_down = []() { return false; };

  // 如果请求使用分块内容提供者
  if (req.is_chunked_content_provider_) {
    // TODO: Brotli support
    // 创建一个压缩器对象
    std::unique_ptr<detail::compressor> compressor;
    // 如果支持压缩
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
    if (compress_) {
      // 使用gzip压缩算法
      compressor = detail::make_unique<detail::gzip_compressor>();
    } else
// 如果定义了宏 #endif，则创建一个 nocompressor 对象
{
  compressor = detail::make_unique<detail::nocompressor>();
}

// 根据请求的内容提供者和压缩器，以分块的方式写入内容到流中
return detail::write_content_chunked(strm, req.content_provider_,
                                     is_shutting_down, *compressor, error);
} else {
// 否则，以普通方式写入内容到流中
return detail::write_content(strm, req.content_provider_, 0,
                             req.content_length_, is_shutting_down, error);
}

// 准备额外的请求头
if (close_connection) {
  // 如果请求中没有 Connection 头部，则添加 Connection: close 头部
  if (!req.has_header("Connection")) {
    req.headers.emplace("Connection", "close");
  }
  }

  // 如果请求头中没有包含 "Host"，则根据是否为 SSL 连接和端口号的不同情况添加 "Host" 头部
  if (!req.has_header("Host")) {
    if (is_ssl()) {
      if (port_ == 443) {
        req.headers.emplace("Host", host_);
      } else {
        req.headers.emplace("Host", host_and_port_);
      }
    } else {
      if (port_ == 80) {
        req.headers.emplace("Host", host_);
      } else {
        req.headers.emplace("Host", host_and_port_);
      }
    }
  }

  // 如果请求头中没有包含 "Accept"，则添加 "Accept" 头部并设置为 "*/*"
  if (!req.has_header("Accept")) { req.headers.emplace("Accept", "*/*"); }
#ifndef CPPHTTPLIB_NO_DEFAULT_USER_AGENT
  // 如果请求中没有设置 User-Agent 头部，则添加默认的 User-Agent 头部
  if (!req.has_header("User-Agent")) {
    auto agent = std::string("cpp-httplib/") + CPPHTTPLIB_VERSION;
    req.headers.emplace("User-Agent", agent);
  }
#endif

  // 如果请求体为空
  if (req.body.empty()) {
    // 如果请求中有内容提供者
    if (req.content_provider_) {
      // 如果不是分块内容提供者
      if (!req.is_chunked_content_provider_) {
        // 如果请求中没有设置 Content-Length 头部，则添加 Content-Length 头部
        if (!req.has_header("Content-Length")) {
          auto length = std::to_string(req.content_length_);
          req.headers.emplace("Content-Length", length);
        }
      }
    } else {
      // 如果请求中没有内容提供者，并且请求方法是 POST、PUT 或 PATCH，则设置 Content-Length 头部为 0
      if (req.method == "POST" || req.method == "PUT" ||
          req.method == "PATCH") {
        req.headers.emplace("Content-Length", "0");
      }
    }
  }
  }
  // 如果请求中没有指定 Content-Type，则添加默认的 text/plain 类型
  } else {
    if (!req.has_header("Content-Type")) {
      req.headers.emplace("Content-Type", "text/plain");
    }
    // 如果请求中没有指定 Content-Length，则根据请求体的大小添加 Content-Length 头部
    if (!req.has_header("Content-Length")) {
      auto length = std::to_string(req.body.size());
      req.headers.emplace("Content-Length", length);
    }
  }

  // 如果设置了基本身份验证的用户名和密码，则在请求头中添加 Authorization 头部
  if (!basic_auth_password_.empty() || !basic_auth_username_.empty()) {
    if (!req.has_header("Authorization")) {
      req.headers.insert(make_basic_authentication_header(
          basic_auth_username_, basic_auth_password_, false));
    }
  }

  // 如果设置了代理服务器的基本身份验证的用户名和密码
  if (!proxy_basic_auth_username_.empty() &&
# 检查是否设置了代理基本认证密码
if (!proxy_basic_auth_password_.empty()) {
    # 如果请求头中没有包含"Proxy-Authorization"，则插入基本认证头部
    if (!req.has_header("Proxy-Authorization")) {
        req.headers.insert(make_basic_authentication_header(
            proxy_basic_auth_username_, proxy_basic_auth_password_, true));
    }
}

# 检查是否设置了 Bearer Token 认证令牌
if (!bearer_token_auth_token_.empty()) {
    # 如果请求头中没有包含"Authorization"，则插入 Bearer Token 认证头部
    if (!req.has_header("Authorization")) {
        req.headers.insert(make_bearer_token_authentication_header(
            bearer_token_auth_token_, false));
    }
}

# 检查是否设置了代理 Bearer Token 认证令牌
if (!proxy_bearer_token_auth_token_.empty()) {
    # 如果请求头中没有包含"Proxy-Authorization"，则插入代理 Bearer Token 认证头部
    if (!req.has_header("Proxy-Authorization")) {
        req.headers.insert(make_bearer_token_authentication_header(
            proxy_bearer_token_auth_token_, true));
    }
}
  // 创建一个缓冲流对象
  {
    detail::BufferStream bstrm;

    // 如果需要对 URL 进行编码，则对请求路径进行编码
    const auto &path = url_encode_ ? detail::encode_url(req.path) : req.path;
    // 将请求方法和路径写入缓冲流
    bstrm.write_format("%s %s HTTP/1.1\r\n", req.method.c_str(), path.c_str());

    // 将请求头信息写入缓冲流
    detail::write_headers(bstrm, req.headers);

    // 刷新缓冲区
    auto &data = bstrm.get_buffer();
    // 如果无法将数据写入流，则返回错误
    if (!detail::write_data(strm, data.data(), data.size())) {
      error = Error::Write;
      return false;
    }
  }

  // 请求体
  if (req.body.empty()) {
  // 使用提供的内容提供者和请求发送内容，并返回响应
  return write_content_with_provider(strm, req, error);
}

// 如果请求的内容不是通过内容提供者提供的，直接写入请求体
if (!detail::write_data(strm, req.body.data(), req.body.size())) {
  error = Error::Write;
  return false;
}

// 写入请求体成功，返回 true
return true;
}

// 使用内容提供者发送请求，并返回响应
inline std::unique_ptr<Response> ClientImpl::send_with_content_provider(
    Request &req, const char *body, size_t content_length,
    ContentProvider content_provider,
    ContentProviderWithoutLength content_provider_without_length,
    const std::string &content_type, Error &error) {
  // 如果内容类型不为空，将内容类型添加到请求头中
  if (!content_type.empty()) {
    req.headers.emplace("Content-Type", content_type);
  }
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持压缩，则在请求头中添加 Content-Encoding: gzip
  if (compress_) { req.headers.emplace("Content-Encoding", "gzip"); }
#endif

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持压缩且内容提供者没有指定长度，则进行压缩
  if (compress_ && !content_provider_without_length) {
    // TODO: Brotli support
    // 创建 gzip 压缩器
    detail::gzip_compressor compressor;

    if (content_provider) {
      auto ok = true;
      size_t offset = 0;
      DataSink data_sink;

      // 写入数据的回调函数
      data_sink.write = [&](const char *data, size_t data_len) -> bool {
        if (ok) {
          auto last = offset + data_len == content_length;

          // 压缩数据
          auto ret = compressor.compress(
              data, data_len, last,
// 使用lambda表达式将压缩数据附加到请求体中
([&](const char *compressed_data, size_t compressed_data_len) {
    req.body.append(compressed_data, compressed_data_len);
    return true;
});

// 如果成功将压缩数据附加到请求体中，则增加偏移量
if (ret) {
    offset += data_len;
} else {
    ok = false;
}

// 循环直到偏移量小于内容长度，并且将内容提供者提供的数据传输给数据接收器
while (ok && offset < content_length) {
    if (!content_provider(offset, content_length - offset, data_sink)) {
        // 如果内容提供者返回false，则设置错误为Canceled并返回空指针
        error = Error::Canceled;
        return nullptr;
    }
}
    } else {
      // 如果不需要压缩，则执行以下操作
      if (!compressor.compress(body, content_length, true,
                               // 使用lambda表达式处理压缩后的数据
                               [&](const char *data, size_t data_len) {
                                 req.body.append(data, data_len);
                                 return true;
                               })) {
        // 如果压缩失败，设置错误类型并返回空指针
        error = Error::Compression;
        return nullptr;
      }
    }
  } else
#endif
  {
    // 如果不需要压缩，则执行以下操作
    if (content_provider) {
      // 如果有内容提供者，则设置请求的内容长度和内容提供者
      req.content_length_ = content_length;
      req.content_provider_ = std::move(content_provider);
      req.is_chunked_content_provider_ = false;
    } else if (content_provider_without_length) {
      // 如果没有内容提供者长度，则设置请求的内容长度为0，并设置内容提供者
      req.content_length_ = 0;
      req.content_provider_ = detail::ContentProviderAdapter(
// 使用内容提供者发送请求，如果内容长度未知，则使用分块传输编码
inline Result ClientImpl::send_with_content_provider(
    const std::string &method, const std::string &path, const Headers &headers,
    const char *body, size_t content_length, ContentProvider content_provider,
    ContentProviderWithoutLength content_provider_without_length,
    const std::string &content_type) {
  // 创建请求对象
  Request req;
  // 设置请求方法
  req.method = method;
  // 如果内容提供者没有指定内容长度
  if (content_length == 0) {
    // 使用不带长度的内容提供者发送请求
    req.content_provider_without_length_ = std::move(content_provider_without_length);
    // 标记请求使用分块传输编码
    req.is_chunked_content_provider_ = true;
    // 添加传输编码头部
    req.headers.emplace("Transfer-Encoding", "chunked");
  } else {
    // 如果内容提供者指定了内容长度，则将内容赋值给请求体
    req.body.assign(body, content_length);
    // 空语句，多余的分号
    ;
  }
  // 创建响应对象
  auto res = detail::make_unique<Response>();
  // 发送请求并返回结果
  return send(req, *res, error) ? std::move(res) : nullptr;
}
  # 将请求的头部信息设置为给定的头部信息
  req.headers = headers;
  # 将请求的路径设置为给定的路径
  req.path = path;

  # 初始化错误状态为成功
  auto error = Error::Success;

  # 发送请求并获取响应
  auto res = send_with_content_provider(
      req, body, content_length, std::move(content_provider),
      std::move(content_provider_without_length), content_type, error);

  # 返回包含响应、错误状态和请求头部信息的结果
  return Result{std::move(res), error, std::move(req.headers)};
}

# 调整主机字符串格式
inline std::string
ClientImpl::adjust_host_string(const std::string &host) const {
  # 如果主机字符串中包含冒号，则在主机字符串两端添加方括号
  if (host.find(':') != std::string::npos) { return "[" + host + "]"; }
  # 否则直接返回主机字符串
  return host;
}

# 处理请求
inline bool ClientImpl::process_request(Stream &strm, Request &req,
                                        Response &res, bool close_connection,
  // 发送请求
  if (!write_request(strm, req, close_connection, error)) { return false; }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL
  if (is_ssl()) {
    // 检查是否启用了代理
    auto is_proxy_enabled = !proxy_host_.empty() && proxy_port_ != -1;
    // 如果未启用代理
    if (!is_proxy_enabled) {
      // 检查 SSL 连接是否已关闭
      char buf[1];
      if (SSL_peek(socket_.ssl, buf, 1) == 0 &&
          SSL_get_error(socket_.ssl, 0) == SSL_ERROR_ZERO_RETURN) {
        // 如果 SSL 连接已关闭，则返回错误
        error = Error::SSLPeerCouldBeClosed_;
        return false;
      }
    }
  }
#endif

  // 接收响应和头部信息
  if (!read_response_line(strm, req, res) ||
  // 检查是否成功读取响应头部信息，如果失败则返回错误
  if (!detail::read_headers(strm, res.headers)) {
    error = Error::Read;
    return false;
  }

  // 处理响应体
  if ((res.status != 204) && req.method != "HEAD" && req.method != "CONNECT") {
    // 检查是否需要重定向
    auto redirect = 300 < res.status && res.status < 400 && follow_location_;

    // 如果有响应处理函数且不需要重定向，则调用响应处理函数
    if (req.response_handler && !redirect) {
      if (!req.response_handler(res)) {
        error = Error::Canceled;
        return false;
      }
    }

    // 如果有内容接收器，则调用内容接收器处理响应体数据
    auto out =
        req.content_receiver
            ? static_cast<ContentReceiverWithProgress>(
                  [&](const char *buf, size_t n, uint64_t off, uint64_t len) {
// 如果发生重定向，则返回 true
if (redirect) { return true; }
// 调用 req.content_receiver 方法，将数据写入 buf 中
auto ret = req.content_receiver(buf, n, off, len);
// 如果返回值为 false，则将错误标记为 Error::Canceled
if (!ret) { error = Error::Canceled; }
// 返回 ret
return ret;
// 如果没有指定进度回调函数，则使用匿名函数作为进度回调函数
: static_cast<ContentReceiverWithProgress>(
      [&](const char *buf, size_t n, uint64_t /*off*/,
          uint64_t /*len*/) {
        // 如果将要添加的数据超过了最大限制，则返回 false
        if (res.body.size() + n > res.body.max_size()) {
          return false;
        }
        // 将数据添加到 res.body 中
        res.body.append(buf, n);
        // 返回 true
        return true;
      });

// 定义进度回调函数
auto progress = [&](uint64_t current, uint64_t total) {
  // 如果没有指定进度回调函数或发生重定向，则返回 true
  if (!req.progress || redirect) { return true; }
  // 调用进度回调函数，将当前进度和总进度传入
  auto ret = req.progress(current, total);
  // 如果返回值为 false，则将错误标记为 Error::Canceled
  if (!ret) { error = Error::Canceled; }
  // 返回 ret
  return ret;
    };

    // 定义一个整型变量dummy_status
    int dummy_status;
    // 调用detail命名空间下的read_content函数，读取内容并解压缩
    if (!detail::read_content(strm, res, (std::numeric_limits<size_t>::max)(),
                              dummy_status, std::move(progress), std::move(out),
                              decompress_)) {
      // 如果读取内容失败并且错误不是Canceled，则将错误设置为Read
      if (error != Error::Canceled) { error = Error::Read; }
      return false;
    }
  }

  // 如果响应头中的Connection字段为close，或者响应版本为HTTP/1.0且原因不是Connection established
  if (res.get_header_value("Connection") == "close" ||
      (res.version == "HTTP/1.0" && res.reason != "Connection established")) {
    // TODO 这需要一系列调用的正确性才能安全地调用，也许重构代码可以使这更明显
    // 这是安全的调用，因为process_request只能被handle_request调用，而handle_request只能被send调用，send会锁定请求
    // 可能需要重构代码以使这更明显
    // 这是安全的调用，因为process_request只能被handle_request调用，而handle_request只能被send调用，send会锁定请求
    // 在整个过程中使用互斥锁。如果从不同的线程调用它，会导致线程安全问题，因为在另一个线程使用套接字时，对套接字进行这些操作会导致 bug。
    std::lock_guard<std::mutex> guard(socket_mutex_);
    // 关闭 SSL 连接
    shutdown_ssl(socket_, true);
    // 关闭套接字
    shutdown_socket(socket_);
    // 关闭套接字
    close_socket(socket_);
  }

  // 如果存在日志记录器，则记录请求和响应
  if (logger_) { logger_(req, res); }

  // 返回 true
  return true;
}

// 获取多部分内容提供程序，用于处理多部分表单数据
inline ContentProviderWithoutLength ClientImpl::get_multipart_content_provider(
    const std::string &boundary, const MultipartFormDataItems &items,
    const MultipartFormDataProviderItems &provider_items) {
  // 初始化当前项和当前起始位置
  size_t cur_item = 0, cur_start = 0;
  // cur_item 和 cur_start 被复制到 std::function 中，并保持不变
  // 用于在连续调用之间保持状态
  return [&, cur_item, cur_start](size_t offset,
                                  DataSink &sink) mutable -> bool {
    // 如果偏移量为0且items不为空，则序列化多部分表单数据并写入到sink中
    if (!offset && items.size()) {
      sink.os << detail::serialize_multipart_formdata(items, boundary, false);
      return true;
    } else if (cur_item < provider_items.size()) {
      // 如果当前起始位置为0
      if (!cur_start) {
        // 序列化多部分表单数据项的起始部分，并将其写入到sink中
        const auto &begin = detail::serialize_multipart_formdata_item_begin(
            provider_items[cur_item], boundary);
        offset += begin.size();
        cur_start = offset;
        sink.os << begin;
      }

      // 创建一个新的数据接收器
      DataSink cur_sink;
      bool has_data = true;
      // 将sink的写入函数赋值给cur_sink
      cur_sink.write = sink.write;
      // 定义一个函数，当数据接收完成时将has_data设置为false
      cur_sink.done = [&]() { has_data = false; };
      // 如果当前提供者的项目没有数据，则将当前项目的结束标记写入到输出流中，并更新当前项目和起始偏移量
      if (!provider_items[cur_item].provider(offset - cur_start, cur_sink))
        return false;

      // 如果没有数据，则将多部分表单数据项的结束标记写入到输出流中，并更新当前项目和起始偏移量
      if (!has_data) {
        sink.os << detail::serialize_multipart_formdata_item_end();
        cur_item++;
        cur_start = 0;
      }
      // 返回 true 表示处理成功
      return true;
    } else {
      // 如果有数据，则将多部分表单数据的结束标记写入到输出流中，并标记处理完成
      sink.os << detail::serialize_multipart_formdata_finish(boundary);
      sink.done();
      // 返回 true 表示处理成功
      return true;
    }
  };
}

// 处理套接字数据，并调用回调函数处理数据流
inline bool
ClientImpl::process_socket(const Socket &socket,
                           std::function<bool(Stream &strm)> callback) {
// 调用 process_client_socket 函数处理客户端套接字，传入读写超时时间和回调函数
return detail::process_client_socket(
    socket.sock, read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
    write_timeout_usec_, std::move(callback));
}

// 返回是否使用 SSL
inline bool ClientImpl::is_ssl() const { return false; }

// 调用 Get 函数，传入路径、空的头部信息和进度回调函数
inline Result ClientImpl::Get(const std::string &path) {
  return Get(path, Headers(), Progress());
}

// 调用 Get 函数，传入路径、空的头部信息和自定义的进度回调函数
inline Result ClientImpl::Get(const std::string &path, Progress progress) {
  return Get(path, Headers(), std::move(progress));
}

// 调用 Get 函数，传入路径、自定义的头部信息和空的进度回调函数
inline Result ClientImpl::Get(const std::string &path, const Headers &headers) {
  return Get(path, headers, Progress());
}

// 调用 Get 函数，传入路径和自定义的头部信息和进度回调函数
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
// 使用给定的路径和进度对象创建一个GET请求
Request req;
req.method = "GET";
req.path = path;
req.headers = headers;
req.progress = std::move(progress);

// 发送创建的请求并返回结果
return send_(std::move(req));
```

```
// 使用给定的路径和内容接收器创建一个GET请求并返回结果
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver) {
  return Get(path, Headers(), nullptr, std::move(content_receiver), nullptr);
}
```

```
// 使用给定的路径、内容接收器和进度对象创建一个GET请求并返回结果
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver,
                              Progress progress) {
  return Get(path, Headers(), nullptr, std::move(content_receiver),
             std::move(progress));
```
// 定义 ClientImpl 类的 Get 方法，用于发送 GET 请求并接收响应
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver) {
  // 调用重载的 Get 方法，传入路径、请求头、空指针、内容接收器，并返回结果
  return Get(path, headers, nullptr, std::move(content_receiver), nullptr);
}

// 定义 ClientImpl 类的 Get 方法的重载，用于发送 GET 请求并接收响应，并带有进度参数
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver,
                              Progress progress) {
  // 调用重载的 Get 方法，传入路径、请求头、空指针、内容接收器、进度参数，并返回结果
  return Get(path, headers, nullptr, std::move(content_receiver),
             std::move(progress));
}

// 定义 ClientImpl 类的 Get 方法的重载，用于发送 GET 请求并接收响应，并带有响应处理器和内容接收器
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  // 调用重载的 Get 方法，传入路径、空请求头、响应处理器、内容接收器、空指针，并返回结果
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), nullptr);
}
# 定义一个内联函数，用于发送 GET 请求并处理响应
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  # 调用重载的 Get 函数，传入路径、请求头、响应处理器和内容接收器
  return Get(path, headers, std::move(response_handler),
             std::move(content_receiver), nullptr);
}

# 定义一个内联函数，用于发送 GET 请求并处理响应
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 调用重载的 Get 函数，传入路径、空的请求头、响应处理器、内容接收器和进度
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), std::move(progress));
}

# 定义一个内联函数，用于发送 GET 请求并处理响应
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  // 创建一个请求对象
  Request req;
  // 设置请求方法为GET
  req.method = "GET";
  // 设置请求路径为给定的路径
  req.path = path;
  // 设置请求头部为给定的头部信息
  req.headers = headers;
  // 将响应处理器移动到请求对象中
  req.response_handler = std::move(response_handler);
  // 设置内容接收器，接收数据并传递给内容接收器
  req.content_receiver =
      [content_receiver](const char *data, size_t data_length,
                         uint64_t /*offset*/, uint64_t /*total_length*/) {
        return content_receiver(data, data_length);
      };
  // 将进度处理器移动到请求对象中
  req.progress = std::move(progress);

  // 发送请求并返回结果
  return send_(std::move(req));
}

// 发送GET请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers, Progress progress) {
  // 如果参数为空，则直接发送GET请求
  if (params.empty()) { return Get(path, headers); }
  // 否则，将参数拼接到路径中
  std::string path_with_query = append_query_params(path, params);
// 返回使用给定查询路径和标头的 GET 请求的结果
Result ClientImpl::Get(const std::string &path, const Params &params,
                       const Headers &headers,
                       ContentReceiver content_receiver,
                       Progress progress) {
  // 调用重载的 Get 函数，传递空的响应处理程序
  return Get(path, params, headers, nullptr, content_receiver, progress);
}

// 返回使用给定路径、参数和标头的 GET 请求的结果
Result ClientImpl::Get(const std::string &path, const Params &params,
                       const Headers &headers,
                       ResponseHandler response_handler,
                       ContentReceiver content_receiver,
                       Progress progress) {
  // 如果参数为空，则调用重载的 Get 函数，传递空的参数
  if (params.empty()) {
    return Get(path, headers, response_handler, content_receiver, progress);
  }
  // 将参数附加到路径上
  std::string path_with_query = append_query_params(path, params);
// 使用给定的路径和查询字符串发送 GET 请求，包括自定义头部、响应处理器、内容接收器和进度
return Get(path_with_query.c_str(), headers, response_handler, content_receiver, progress);
}

// 发送 HEAD 请求，不包括自定义头部
inline Result ClientImpl::Head(const std::string &path) {
  return Head(path, Headers());
}

// 发送 HEAD 请求，包括自定义头部
inline Result ClientImpl::Head(const std::string &path, const Headers &headers) {
  // 创建请求对象
  Request req;
  req.method = "HEAD";
  req.headers = headers;
  req.path = path;

  // 发送请求
  return send_(std::move(req));
}

// 发送 POST 请求，不包括自定义头部和内容
inline Result ClientImpl::Post(const std::string &path) {
  return Post(path, std::string(), std::string());
// 客户端实现类的成员函数，用于发送 POST 请求，不带自定义内容类型和内容提供者
inline Result ClientImpl::Post(const std::string &path,
                               const Headers &headers) {
  // 调用另一个重载的 Post 函数，传入默认的空 body 和 content_type
  return Post(path, headers, nullptr, 0, std::string());
}

// 客户端实现类的成员函数，用于发送 POST 请求，带自定义内容类型和内容长度
inline Result ClientImpl::Post(const std::string &path, const char *body,
                               size_t content_length,
                               const std::string &content_type) {
  // 调用另一个重载的 Post 函数，传入默认的空 headers
  return Post(path, Headers(), body, content_length, content_type);
}

// 客户端实现类的成员函数，用于发送 POST 请求，带自定义内容类型和内容长度
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const char *body, size_t content_length,
                               const std::string &content_type) {
  // 调用发送函数，传入请求方法、路径、自定义 headers、body、内容长度和内容类型
  return send_with_content_provider("POST", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}
// 使用默认的头部信息和内容类型发送 POST 请求
inline Result ClientImpl::Post(const std::string &path, const std::string &body,
                               const std::string &content_type) {
  return Post(path, Headers(), body, content_type);
}

// 发送带有自定义头部信息的 POST 请求
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const std::string &body,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

// 发送带有参数的 POST 请求
inline Result ClientImpl::Post(const std::string &path, const Params &params) {
  return Post(path, Headers(), params);
}

// 发送带有内容长度、内容提供者和内容类型的 POST 请求
inline Result ClientImpl::Post(const std::string &path, size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
// 使用给定的路径、头部、内容长度、内容提供者和内容类型创建一个 POST 请求，并返回结果
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  // 调用 send_with_content_provider 函数发送带有内容提供者的 POST 请求，并返回结果
  return send_with_content_provider("POST", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

// 使用给定的路径、头部、内容提供者和内容类型创建一个 POST 请求，并返回结果
inline Result ClientImpl::Post(const std::string &path,
                               ContentProviderWithoutLength content_provider,
                               const std::string &content_type) {
  // 调用上面定义的 Post 函数，传入空的头部和内容长度，并返回结果
  return Post(path, Headers(), std::move(content_provider), content_type);
}

// 使用给定的路径、头部、内容长度、内容提供者和内容类型创建一个 POST 请求，并返回结果
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  // 调用 send_with_content_provider 函数发送带有内容提供者的 POST 请求，并返回结果
  return send_with_content_provider("POST", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

// 使用给定的路径、头部、内容长度、内容提供者和内容类型创建一个 POST 请求，并返回结果
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  // 调用 send_with_content_provider 函数发送带有内容提供者的 POST 请求，并返回结果
  return send_with_content_provider("POST", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}
// 使用内容提供者发送 POST 请求，包括路径、头部、内容提供者、内容类型
Result ClientImpl::send_with_content_provider(
    "POST", path, headers, nullptr, 0, nullptr,
    std::move(content_provider), content_type);

// 发送 POST 请求，包括路径、头部、参数，将参数转换为查询字符串
auto query = detail::params_to_query_str(params);
return Post(path, headers, query, "application/x-www-form-urlencoded");

// 发送 POST 请求，包括路径和多部分表单数据项
return Post(path, Headers(), items);

// 发送 POST 请求，包括路径、头部、多部分表单数据项
const auto &boundary = detail::make_multipart_data_boundary();
// 获取多部分表单数据的内容类型
const auto &content_type =
    detail::serialize_multipart_formdata_get_content_type(boundary);
// 序列化多部分表单数据
const auto &body = detail::serialize_multipart_formdata(items, boundary);
// 发送 POST 请求
return Post(path, headers, body, content_type.c_str());
}

// 发送带有多部分表单数据的 POST 请求
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const MultipartFormDataItems &items,
                               const std::string &boundary) {
  // 检查多部分边界字符是否有效
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    return Result{nullptr, Error::UnsupportedMultipartBoundaryChars};
  }

  // 获取多部分表单数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分表单数据
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  // 发送 POST 请求
  return Post(path, headers, body, content_type.c_str());
}

// 其他函数或代码
// 定义一个名为Post的函数，接受路径、头部信息、多部分表单数据项和多部分表单数据提供者项作为参数
ClientImpl::Post(const std::string &path, const Headers &headers,
                 const MultipartFormDataItems &items,
                 const MultipartFormDataProviderItems &provider_items) {
  // 生成多部分数据边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 序列化多部分表单数据并获取内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 使用内容提供者发送请求
  return send_with_content_provider(
      "POST", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}

// 定义一个名为Put的内联函数，接受路径作为参数
inline Result ClientImpl::Put(const std::string &path) {
  // 调用另一个重载的Put函数，传入路径和空字符串作为参数
  return Put(path, std::string(), std::string());
}

// 定义一个名为Put的内联函数，接受路径、数据、数据长度和内容类型作为参数
inline Result ClientImpl::Put(const std::string &path, const char *body,
                              size_t content_length,
                              const std::string &content_type) {
  // 调用另一个重载的Put函数，传入路径、头部信息、数据、数据长度和内容类型作为参数
  return Put(path, Headers(), body, content_length, content_type);
}
// 客户端实现的 Put 方法，用于发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const char *body, size_t content_length,
                              const std::string &content_type) {
  // 调用 send_with_content_provider 方法发送带有内容提供者的 PUT 请求
  return send_with_content_provider("PUT", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}

// 客户端实现的 Put 方法的重载，用于发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const std::string &body,
                              const std::string &content_type) {
  // 调用上面定义的 Put 方法，传入默认的 Headers
  return Put(path, Headers(), body, content_type);
}

// 客户端实现的 Put 方法的重载，用于发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const std::string &body,
                              const std::string &content_type) {
  // 调用 send_with_content_provider 方法发送带有内容提供者的 PUT 请求
  return send_with_content_provider("PUT", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}
// 客户端实现的Put方法，用于向服务器发送数据
inline Result ClientImpl::Put(const std::string &path, size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  // 调用重载的Put方法，传入路径、内容长度、内容提供者和内容类型
  return Put(path, Headers(), content_length, std::move(content_provider),
             content_type);
}

// 客户端实现的Put方法的重载，用于向服务器发送数据
inline Result ClientImpl::Put(const std::string &path,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  // 调用重载的Put方法，传入路径、空的头部、内容提供者和内容类型
  return Put(path, Headers(), std::move(content_provider), content_type);
}

// 客户端实现的Put方法的重载，用于向服务器发送数据
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  // 调用发送带内容提供者的方法，传入请求类型、路径、头部、空的回调函数、内容长度、内容提供者和内容类型
  return send_with_content_provider("PUT", path, headers, nullptr,
                                   content_length, std::move(content_provider),
                                   content_type);
}
// 使用给定的内容提供者发送 PUT 请求，返回结果
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 发送不带内容长度的 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Params &params) {
  return Put(path, Headers(), params);
}

// 发送带有头部信息和参数的 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const Params &params) {
  // 将参数转换为查询字符串
  auto query = detail::params_to_query_str(params);
  // 发送带有头部信息、查询字符串和默认内容类型的 PUT 请求
  return Put(path, headers, query, "application/x-www-form-urlencoded");
}
// 将文件上传到指定路径，使用默认的请求头和表单数据
inline Result ClientImpl::Put(const std::string &path,
                              const MultipartFormDataItems &items) {
  return Put(path, Headers(), items);
}

// 将文件上传到指定路径，使用指定的请求头和表单数据
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items) {
  // 生成多部分数据的边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 生成多部分数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分数据
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  // 发起 PUT 请求
  return Put(path, headers, body, content_type);
}

// 将文件上传到指定路径，使用指定的请求头、表单数据和边界
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items,
                              const std::string &boundary) {
  // 检查多部分数据的边界是否合法
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    return Result{nullptr, Error::UnsupportedMultipartBoundaryChars};
// 定义一个函数，用于发送带有多部分表单数据的 PUT 请求
inline Result
ClientImpl::Put(const std::string &path, const Headers &headers,
                const MultipartFormDataItems &items,
                const MultipartFormDataProviderItems &provider_items) {
  // 生成多部分表单数据的边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 获取多部分表单数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 使用多部分表单数据和内容类型发送 PUT 请求
  return send_with_content_provider(
      "PUT", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}
// 使用默认的空字符串和长度调用重载的 Patch 函数
inline Result ClientImpl::Patch(const std::string &path) {
  return Patch(path, std::string(), std::string());
}

// 使用给定的路径、请求体、内容长度和内容类型调用重载的 Patch 函数
inline Result ClientImpl::Patch(const std::string &path, const char *body,
                                size_t content_length,
                                const std::string &content_type) {
  return Patch(path, Headers(), body, content_length, content_type);
}

// 使用给定的路径、请求头、请求体、内容长度和内容类型调用 send_with_content_provider 函数
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const char *body, size_t content_length,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, body,
                                    content_length, nullptr, nullptr,
                                    content_type);
}

// 使用给定的路径和请求体调用重载的 Patch 函数
inline Result ClientImpl::Patch(const std::string &path,
                                const std::string &body,
// 使用给定的路径、头部、请求体和内容类型发送 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const std::string &body,
                                const std::string &content_type) {
  // 调用另一个重载的 Patch 函数，传递请求体的数据和内容类型
  return send_with_content_provider("PATCH", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

// 使用给定的路径、头部、内容长度、内容提供者和内容类型发送 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, size_t content_length,
                                ContentProvider content_provider,
                                const std::string &content_type) {
  // 调用另一个重载的 Patch 函数，传递内容长度、内容提供者和内容类型
  return Patch(path, Headers(), content_length, std::move(content_provider),
               content_type);
}

// 使用给定的路径、内容类型发送 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path,
// 使用给定的内容提供者和内容类型发送 PATCH 请求
Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                        ContentProviderWithoutLength content_provider,
                        const std::string &content_type) {
  // 调用另一个重载的 Patch 函数，传入空的 Headers 和 content_length
  return send_with_content_provider("PATCH", path, headers, nullptr, 0, nullptr,
                                  std::move(content_provider), content_type);
}

// 使用给定的内容长度、内容提供者和内容类型发送 PATCH 请求
Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                        size_t content_length,
                        ContentProvider content_provider,
                        const std::string &content_type) {
  // 调用另一个重载的 Patch 函数，传入内容长度和内容提供者
  return send_with_content_provider("PATCH", path, headers, nullptr,
                                  content_length, std::move(content_provider),
                                  nullptr, content_type);
}

// 使用给定的内容提供者和内容类型发送 PATCH 请求
Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                        ContentProviderWithoutLength content_provider,
                        const std::string &content_type) {
  // 调用另一个重载的 Patch 函数，传入空的 Headers 和 content_length
  return send_with_content_provider("PATCH", path, headers, nullptr, 0, nullptr,
                                  std::move(content_provider), content_type);
}
// 使用默认的请求头和空的请求体发送 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path) {
  return Delete(path, Headers(), std::string(), std::string());
}

// 使用指定的请求头发送 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers) {
  return Delete(path, headers, std::string(), std::string());
}

// 使用指定的请求头和请求体发送 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  return Delete(path, Headers(), body, content_length, content_type);
}

// 使用指定的请求头和请求体发送 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  // 创建一个请求对象
  Request req;
  // 设置请求方法为DELETE
  req.method = "DELETE";
  // 设置请求头部
  req.headers = headers;
  // 设置请求路径
  req.path = path;

  // 如果内容类型不为空，添加Content-Type到请求头部
  if (!content_type.empty()) {
    req.headers.emplace("Content-Type", content_type);
  }
  // 将请求体内容和长度赋给请求对象
  req.body.assign(body, content_length);

  // 调用send_函数发送请求并返回结果
  return send_(std::move(req));
}

// 重载的Delete函数，调用时不传入请求头部
inline Result ClientImpl::Delete(const std::string &path,
                                 const std::string &body,
                                 const std::string &content_type) {
  return Delete(path, Headers(), body.data(), body.size(), content_type);
}

// 重载的Delete函数，调用时传入请求头部
inline Result ClientImpl::Delete(const std::string &path,
// 使用给定的路径、头部、请求体和内容类型发送一个 DELETE 请求
Result ClientImpl::Delete(const std::string &path,
                           const Headers &headers,
                           const char *body,
                           size_t body_size,
                           const std::string &content_type) {
  // 创建一个请求对象
  Request req;
  // 设置请求方法为 DELETE
  req.method = "DELETE";
  // 设置请求头部
  req.headers = headers;
  // 设置请求路径
  req.path = path;
  // 设置请求体和内容类型
  req.body = body;
  req.body_size = body_size;
  req.content_type = content_type;

  // 发送请求并返回结果
  return send_(std::move(req));
}

// 使用给定的路径发送一个 OPTIONS 请求
inline Result ClientImpl::Options(const std::string &path) {
  // 调用重载的 Options 方法，传入路径和空的头部
  return Options(path, Headers());
}

// 使用给定的路径和头部发送一个 OPTIONS 请求
inline Result ClientImpl::Options(const std::string &path,
                                  const Headers &headers) {
  // 创建一个请求对象
  Request req;
  // 设置请求方法为 OPTIONS
  req.method = "OPTIONS";
  // 设置请求头部
  req.headers = headers;
  // 设置请求路径
  req.path = path;

  // 发送请求并返回结果
  return send_(std::move(req));
}
// 检查套接字是否打开，使用互斥锁保护
inline size_t ClientImpl::is_socket_open() const {
  std::lock_guard<std::mutex> guard(socket_mutex_);
  return socket_.is_open();
}

// 返回套接字对象的套接字
inline socket_t ClientImpl::socket() const { return socket_.sock; }

// 停止套接字操作
inline void ClientImpl::stop() {
  std::lock_guard<std::mutex> guard(socket_mutex_);

  // 如果当前有正在进行的操作，唯一线程安全的操作是关闭套接字，以便使用该套接字的线程发现无法再读写并报错
  if (socket_requests_in_flight_ > 0) {
    shutdown_socket(socket_);

    // 除此之外，我们设置一个标志，以便在完成后关闭套接字
// 设置当请求完成时是否关闭套接字
socket_should_be_closed_when_request_is_done_ = true;
// 返回，结束函数执行
return;
}

// 否则，在仍持有互斥锁的情况下，我们可以自己关闭所有连接
shutdown_ssl(socket_, true);
shutdown_socket(socket_);
close_socket(socket_);
}

// 设置连接超时时间
inline void ClientImpl::set_connection_timeout(time_t sec, time_t usec) {
connection_timeout_sec_ = sec;
connection_timeout_usec_ = usec;
}

// 设置读取超时时间
inline void ClientImpl::set_read_timeout(time_t sec, time_t usec) {
read_timeout_sec_ = sec;
read_timeout_usec_ = usec;
}
// 设置写超时时间，单位为秒和微秒
inline void ClientImpl::set_write_timeout(time_t sec, time_t usec) {
  write_timeout_sec_ = sec; // 设置写超时时间的秒部分
  write_timeout_usec_ = usec; // 设置写超时时间的微秒部分
}

// 设置基本身份验证的用户名和密码
inline void ClientImpl::set_basic_auth(const std::string &username,
                                       const std::string &password) {
  basic_auth_username_ = username; // 设置基本身份验证的用户名
  basic_auth_password_ = password; // 设置基本身份验证的密码
}

// 设置Bearer令牌身份验证的令牌
inline void ClientImpl::set_bearer_token_auth(const std::string &token) {
  bearer_token_auth_token_ = token; // 设置Bearer令牌身份验证的令牌
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置摘要身份验证的用户名和密码
inline void ClientImpl::set_digest_auth(const std::string &username,
                                        const std::string &password) {
  digest_auth_username_ = username; // 设置摘要身份验证的用户名
  digest_auth_password_ = password; // 设置摘要身份验证的密码
// 结束条件判断，结束宏定义
}
#endif

// 设置是否保持连接
inline void ClientImpl::set_keep_alive(bool on) { keep_alive_ = on; }

// 设置是否跟随重定向
inline void ClientImpl::set_follow_location(bool on) { follow_location_ = on; }

// 设置是否进行 URL 编码
inline void ClientImpl::set_url_encode(bool on) { url_encode_ = on; }

// 设置主机名到地址的映射
inline void
ClientImpl::set_hostname_addr_map(std::map<std::string, std::string> addr_map) {
  addr_map_ = std::move(addr_map);
}

// 设置默认的请求头
inline void ClientImpl::set_default_headers(Headers headers) {
  default_headers_ = std::move(headers);
}

// 设置地址族
inline void ClientImpl::set_address_family(int family) {
  address_family_ = family;
// 设置是否开启 TCP 无延迟
inline void ClientImpl::set_tcp_nodelay(bool on) { tcp_nodelay_ = on; }

// 设置套接字选项
inline void ClientImpl::set_socket_options(SocketOptions socket_options) {
  socket_options_ = std::move(socket_options);
}

// 设置是否开启压缩
inline void ClientImpl::set_compress(bool on) { compress_ = on; }

// 设置是否开启解压缩
inline void ClientImpl::set_decompress(bool on) { decompress_ = on; }

// 设置接口
inline void ClientImpl::set_interface(const std::string &intf) {
  interface_ = intf;
}

// 设置代理
inline void ClientImpl::set_proxy(const std::string &host, int port) {
  proxy_host_ = host;
  proxy_port_ = port;
}
// 设置代理服务器的基本身份验证信息，包括用户名和密码
inline void ClientImpl::set_proxy_basic_auth(const std::string &username,
                                             const std::string &password) {
  proxy_basic_auth_username_ = username;
  proxy_basic_auth_password_ = password;
}

// 设置代理服务器的令牌身份验证信息
inline void ClientImpl::set_proxy_bearer_token_auth(const std::string &token) {
  proxy_bearer_token_auth_token_ = token;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理服务器的摘要身份验证信息，包括用户名和密码
inline void ClientImpl::set_proxy_digest_auth(const std::string &username,
                                              const std::string &password) {
  proxy_digest_auth_username_ = username;
  proxy_digest_auth_password_ = password;
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书文件路径和目录路径
inline void ClientImpl::set_ca_cert_path(const std::string &ca_cert_file_path,
                                         const std::string &ca_cert_dir_path) {
  ca_cert_file_path_ = ca_cert_file_path;
  ca_cert_dir_path_ = ca_cert_dir_path;
}

// 设置 CA 证书存储
inline void ClientImpl::set_ca_cert_store(X509_STORE *ca_cert_store) {
  // 如果传入的证书存储不为空且与当前存储不同，则更新证书存储
  if (ca_cert_store && ca_cert_store != ca_cert_store_) {
    ca_cert_store_ = ca_cert_store;
  }
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 启用/禁用服务器证书验证
inline void ClientImpl::enable_server_certificate_verification(bool enabled) {
  server_certificate_verification_ = enabled;
}
#endif

// 设置日志记录器
inline void ClientImpl::set_logger(Logger logger) {
// 移动构造函数，将logger_的资源所有权转移给logger
logger_ = std::move(logger);
}

/*
 * SSL Implementation
 */
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
namespace detail {

// 创建一个新的SSL对象
template <typename U, typename V>
inline SSL *ssl_new(socket_t sock, SSL_CTX *ctx, std::mutex &ctx_mutex,
                    U SSL_connect_or_accept, V setup) {
  SSL *ssl = nullptr;
  {
    // 使用互斥锁保护SSL_CTX对象，防止多线程访问
    std::lock_guard<std::mutex> guard(ctx_mutex);
    // 创建一个新的SSL对象
    ssl = SSL_new(ctx);
  }

  // 如果成功创建SSL对象
  if (ssl) {
    // 设置套接字为非阻塞模式
    set_nonblocking(sock, true);
    // 创建一个新的基于套接字的 BIO对象，不关闭套接字
    auto bio = BIO_new_socket(static_cast<int>(sock), BIO_NOCLOSE);
    // 设置BIO对象为非阻塞模式
    BIO_set_nbio(bio, 1);
    // 将BIO对象与SSL对象关联
    SSL_set_bio(ssl, bio, bio);

    // 如果SSL设置或连接失败
    if (!setup(ssl) || SSL_connect_or_accept(ssl) != 1) {
      // 关闭SSL连接
      SSL_shutdown(ssl);
      {
        // 上锁，释放SSL对象
        std::lock_guard<std::mutex> guard(ctx_mutex);
        SSL_free(ssl);
      }
      // 设置套接字为非阻塞模式
      set_nonblocking(sock, false);
      // 返回空指针
      return nullptr;
    }
    // 设置BIO对象为阻塞模式
    BIO_set_nbio(bio, 0);
    // 设置套接字为非阻塞模式
    set_nonblocking(sock, false);
  }

  // 返回SSL对象
  return ssl;
}
// 在删除 SSL 连接时，根据需要优雅地关闭连接，避免 SIGPIPE 信号
// 注意：并不总是能够避免 SIGPIPE，这只是尽力而为
if (shutdown_gracefully) { SSL_shutdown(ssl); }

// 使用互斥锁保护 SSL 对象的释放操作
std::lock_guard<std::mutex> guard(ctx_mutex);
SSL_free(ssl);
}

// 非阻塞方式进行 SSL 连接或接受连接
template <typename U>
bool ssl_connect_or_accept_nonblocking(socket_t sock, SSL *ssl,
                                       U ssl_connect_or_accept,
                                       time_t timeout_sec,
                                       time_t timeout_usec) {
  int res = 0;
  // 循环尝试进行 SSL 连接或接受连接，直到成功
  while ((res = ssl_connect_or_accept(ssl)) != 1) {
    // 获取 SSL 连接的错误码
    auto err = SSL_get_error(ssl, res);
    switch (err) {  // 根据错误类型进行不同的处理
    case SSL_ERROR_WANT_READ:  // 如果是 SSL 需要读取数据的错误
      if (select_read(sock, timeout_sec, timeout_usec) > 0) { continue; }  // 如果可以进行读取操作，则继续循环
      break;  // 否则跳出循环
    case SSL_ERROR_WANT_WRITE:  // 如果是 SSL 需要写入数据的错误
      if (select_write(sock, timeout_sec, timeout_usec) > 0) { continue; }  // 如果可以进行写入操作，则继续循环
      break;  // 否则跳出循环
    default: break;  // 其他类型的错误不做处理
    }
    return false;  // 返回 false
  }
  return true;  // 返回 true
}

template <typename T>
inline bool process_server_socket_ssl(
    const std::atomic<socket_t> &svr_sock, SSL *ssl, socket_t sock,
    size_t keep_alive_max_count, time_t keep_alive_timeout_sec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    time_t write_timeout_usec, T callback) {  // 处理服务器端 SSL 连接的函数模板
# 返回处理服务器套接字核心的结果
# 参数包括服务器套接字、套接字、保持连接的最大次数、保持连接的超时时间（秒）
# 以及一个lambda表达式作为回调函数
return process_server_socket_core(
    svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
    [&](bool close_connection, bool &connection_closed) {
        # 使用SSL套接字流创建SSL套接字流对象
        SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                             write_timeout_sec, write_timeout_usec);
        # 调用回调函数并返回结果
        return callback(strm, close_connection, connection_closed);
    });

# 模板函数，处理SSL客户端套接字
# 参数包括SSL对象、套接字、读取超时时间（秒）、读取超时时间（微秒）、写入超时时间（秒）、写入超时时间（微秒）
# 以及一个回调函数
template <typename T>
inline bool
process_client_socket_ssl(SSL *ssl, socket_t sock, time_t read_timeout_sec,
                          time_t read_timeout_usec, time_t write_timeout_sec,
                          time_t write_timeout_usec, T callback) {
    # 使用SSL套接字流创建SSL套接字流对象
    SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                       write_timeout_sec, write_timeout_usec);
    # 调用回调函数并返回结果
    return callback(strm);
}

# SSL初始化类
class SSLInit {
// SSLInit 类的构造函数，用于初始化 SSL 库
public:
  SSLInit() {
    // 初始化 SSL 库，加载 SSL 和加密字符串
    OPENSSL_init_ssl(
        OPENSSL_INIT_LOAD_SSL_STRINGS | OPENSSL_INIT_LOAD_CRYPTO_STRINGS, NULL);
  }
};

// SSLSocketStream 类的构造函数，用于初始化 SSL 套接字流
inline SSLSocketStream::SSLSocketStream(socket_t sock, SSL *ssl,
                                        time_t read_timeout_sec,
                                        time_t read_timeout_usec,
                                        time_t write_timeout_sec,
                                        time_t write_timeout_usec)
    : sock_(sock), ssl_(ssl), read_timeout_sec_(read_timeout_sec),
      read_timeout_usec_(read_timeout_usec),
      write_timeout_sec_(write_timeout_sec),
      write_timeout_usec_(write_timeout_usec) {
  // 清除 SSL 的自动重试模式
  SSL_clear_mode(ssl, SSL_MODE_AUTO_RETRY);
}
// SSLSocketStream 类的析构函数
inline SSLSocketStream::~SSLSocketStream() {}

// 检查套接字是否可读
inline bool SSLSocketStream::is_readable() const {
  return detail::select_read(sock_, read_timeout_sec_, read_timeout_usec_) > 0;
}

// 检查套接字是否可写
inline bool SSLSocketStream::is_writable() const {
  return select_write(sock_, write_timeout_sec_, write_timeout_usec_) > 0 &&
         is_socket_alive(sock_);
}

// 从套接字中读取数据
inline ssize_t SSLSocketStream::read(char *ptr, size_t size) {
  // 如果 SSL 缓冲区中有数据，直接读取
  if (SSL_pending(ssl_) > 0) {
    return SSL_read(ssl_, ptr, static_cast<int>(size));
  } 
  // 如果套接字可读，读取数据
  else if (is_readable()) {
    auto ret = SSL_read(ssl_, ptr, static_cast<int>(size));
    // 如果读取失败，获取错误信息
    if (ret < 0) {
      auto err = SSL_get_error(ssl_, ret);
      int n = 1000;
#ifdef _WIN32
      // 当错误为 SSL_ERROR_WANT_READ 或者 (错误为 SSL_ERROR_SYSCALL 并且 WSAGetLastError() 为 WSAETIMEDOUT) 时，循环执行
      while (--n >= 0 && (err == SSL_ERROR_WANT_READ ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 当错误为 SSL_ERROR_WANT_READ 时，循环执行
      while (--n >= 0 && err == SSL_ERROR_WANT_READ) {
#endif
        // 如果 SSL 连接中有未读取的数据
        if (SSL_pending(ssl_) > 0) {
          // 直接读取数据并返回
          return SSL_read(ssl_, ptr, static_cast<int>(size));
        } else if (is_readable()) {
          // 如果连接可读，等待1毫秒
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          // 读取数据
          ret = SSL_read(ssl_, ptr, static_cast<int>(size));
          // 如果读取成功，返回读取的字节数
          if (ret >= 0) { return ret; }
          // 获取错误码
          err = SSL_get_error(ssl_, ret);
        } else {
          // 如果连接不可读，返回-1
          return -1;
        }
      }
    }
    // 返回读取的字节数
    return ret;
  }
```

// 返回-1，表示写入失败
  return -1;
}

// 写入函数，将指定大小的数据写入到 SSL 套接字流中
inline ssize_t SSLSocketStream::write(const char *ptr, size_t size) {
  // 如果可写
  if (is_writable()) {
    // 计算实际可处理的数据大小
    auto handle_size = static_cast<int>(
        std::min<size_t>(size, (std::numeric_limits<int>::max)()));

    // 调用 SSL_write 函数写入数据
    auto ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));
    // 如果写入失败
    if (ret < 0) {
      // 获取 SSL 错误码
      auto err = SSL_get_error(ssl_, ret);
      int n = 1000;
#ifdef _WIN32
      // 在 Windows 平台下，处理 SSL_ERROR_WANT_WRITE 和 SSL_ERROR_SYSCALL 错误
      while (--n >= 0 && (err == SSL_ERROR_WANT_WRITE ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 在其他平台下，处理 SSL_ERROR_WANT_WRITE 错误
      while (--n >= 0 && err == SSL_ERROR_WANT_WRITE) {
#endif
        // 如果仍然可写
        if (is_writable()) {
// 使当前线程休眠1毫秒
std::this_thread::sleep_for(std::chrono::milliseconds(1));
// 使用SSL写入数据到SSL连接，返回写入的字节数
ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));
// 如果写入成功，则返回写入的字节数
if (ret >= 0) { return ret; }
// 如果写入失败，则获取SSL错误码
err = SSL_get_error(ssl_, ret);
// 如果SSL连接已关闭，则返回-1
} else {
  return -1;
}
// 获取远程IP地址和端口号
inline void SSLSocketStream::get_remote_ip_and_port(std::string &ip, int &port) const {
  detail::get_remote_ip_and_port(sock_, ip, port);
}
// 获取本地IP地址和端口号
inline void SSLSocketStream::get_local_ip_and_port(std::string &ip,
// 获取本地 IP 地址和端口号
void get_local_ip_and_port(const socket_t &sock, std::string &ip, int &port) const {
  detail::get_local_ip_and_port(sock_, ip, port);
}

// 返回套接字对象
inline socket_t SSLSocketStream::socket() const { return sock_; }

// SSL 初始化对象
static SSLInit sslinit_;

} // namespace detail

// SSL HTTP 服务器实现
// 构造函数，初始化 SSL 上下文
inline SSLServer::SSLServer(const char *cert_path, const char *private_key_path,
                            const char *client_ca_cert_file_path,
                            const char *client_ca_cert_dir_path,
                            const char *private_key_password) {
  ctx_ = SSL_CTX_new(TLS_server_method());

  // 如果 SSL 上下文对象存在
  if (ctx_) {
    // 设置 SSL 上下文选项
    SSL_CTX_set_options(ctx_,
                        SSL_OP_NO_COMPRESSION |
    // 设置 SSL 上下文的选项，禁用会话重用
    SSL_CTX_set_options(ctx_, SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

    // 设置 SSL 上下文的最小协议版本为 TLS1.1
    SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

    // 在打开加密私钥之前添加默认密码回调
    if (private_key_password != nullptr && (private_key_password[0] != '\0')) {
      SSL_CTX_set_default_passwd_cb_userdata(ctx_,
                                             (char *)private_key_password);
    }

    // 使用证书链文件和私钥文件来配置 SSL 上下文
    if (SSL_CTX_use_certificate_chain_file(ctx_, cert_path) != 1 ||
        SSL_CTX_use_PrivateKey_file(ctx_, private_key_path, SSL_FILETYPE_PEM) !=
            1) {
      // 如果配置失败，则释放 SSL 上下文并将其置为空
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    } else if (client_ca_cert_file_path || client_ca_cert_dir_path) {
      // 如果存在客户端 CA 证书文件路径或目录路径，则加载验证位置
      SSL_CTX_load_verify_locations(ctx_, client_ca_cert_file_path,
                                    client_ca_cert_dir_path);

      // 设置验证模式
      SSL_CTX_set_verify(
// 创建 SSLServer 类的构造函数，接受证书、私钥和客户端 CA 证书存储作为参数
inline SSLServer::SSLServer(X509 *cert, EVP_PKEY *private_key, X509_STORE *client_ca_cert_store) {
  // 创建 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_server_method());

  // 如果 SSL 上下文对象创建成功
  if (ctx_) {
    // 设置 SSL 选项，禁用压缩和禁用会话重用
    SSL_CTX_set_options(ctx_, SSL_OP_NO_COMPRESSION | SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

    // 设置 SSL 最小协议版本为 TLS 1.1
    SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

    // 如果证书和私钥设置不成功
    if (SSL_CTX_use_certificate(ctx_, cert) != 1 || SSL_CTX_use_PrivateKey(ctx_, private_key) != 1) {
      // 释放 SSL 上下文对象并将其置为空
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}
// 如果客户端 CA 证书存储不为空，则设置客户端 CA 证书存储
    } else if (client_ca_cert_store) {
      SSL_CTX_set_cert_store(ctx_, client_ca_cert_store);

      // 设置 SSL 上下文的验证模式为要求对等方提供证书，并且如果没有对等方证书则验证失败
      SSL_CTX_set_verify(
          ctx_, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, nullptr);
    }
  }
}

// SSLServer 类的构造函数，接受一个设置 SSL 上下文的回调函数
inline SSLServer::SSLServer(
    const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback) {
  // 创建一个 TLS 方法的 SSL 上下文
  ctx_ = SSL_CTX_new(TLS_method());
  if (ctx_) {
    // 如果设置 SSL 上下文的回调函数返回 false，则释放 SSL 上下文并置为空
    if (!setup_ssl_ctx_callback(*ctx_)) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}
// SSLServer 类的析构函数，用于释放 SSL 上下文对象
inline SSLServer::~SSLServer() {
  if (ctx_) { SSL_CTX_free(ctx_); }
}

// 检查 SSL 上下文对象是否有效
inline bool SSLServer::is_valid() const { return ctx_; }

// 获取 SSL 上下文对象的指针
inline SSL_CTX *SSLServer::ssl_context() const { return ctx_; }

// 处理并关闭套接字的 SSL 连接
inline bool SSLServer::process_and_close_socket(socket_t sock) {
  // 创建 SSL 对象并进行 SSL 握手
  auto ssl = detail::ssl_new(
      sock, ctx_, ctx_mutex_,
      [&](SSL *ssl2) {
        return detail::ssl_connect_or_accept_nonblocking(
            sock, ssl2, SSL_accept, read_timeout_sec_, read_timeout_usec_);
      },
      [](SSL * /*ssl2*/) { return true; });

  // 初始化返回值为 false
  auto ret = false;
  // 如果 SSL 对象存在
  if (ssl) {
    // 处理服务器端套接字的 SSL 连接
    ret = detail::process_server_socket_ssl(
// 定义变量和参数，包括服务器套接字、SSL、套接字、保持活动的最大计数、保持活动的超时秒数、读取超时秒数、读取超时微秒数、写入超时秒数、写入超时微秒数
// 调用匿名函数，处理请求，设置请求的 SSL 属性
// 根据处理请求的结果，优雅地关闭连接或非优雅地关闭连接
// 调用 ssl_delete 函数，删除 SSL 对象
// 优雅地关闭套接字
// 关闭套接字
// 返回处理请求的结果
// SSL HTTP客户端实现
inline SSLClient::SSLClient(const std::string &host)
    : SSLClient(host, 443, std::string(), std::string()) {}  // 使用默认端口443调用另一个构造函数

inline SSLClient::SSLClient(const std::string &host, int port)
    : SSLClient(host, port, std::string(), std::string()) {}  // 调用另一个构造函数

inline SSLClient::SSLClient(const std::string &host, int port,
                            const std::string &client_cert_path,
                            const std::string &client_key_path)
    : ClientImpl(host, port, client_cert_path, client_key_path) {  // 调用基类构造函数
  ctx_ = SSL_CTX_new(TLS_client_method());  // 创建SSL上下文

  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));
                });  // 将主机名分割成组件并存储在host_components_中

  if (!client_cert_path.empty() && !client_key_path.empty()) {
    if (SSL_CTX_use_certificate_file(ctx_, client_cert_path.c_str(),
```  // 如果客户端证书路径和客户端密钥路径不为空，则使用它们来设置SSL上下文中的证书和密钥
// 如果客户端证书和客户端私钥都存在
if (client_cert != nullptr && client_key != nullptr) {
  // 创建一个新的 SSL 上下文对象，使用 TLS 客户端方法
  ctx_ = SSL_CTX_new(TLS_client_method());

  // 将主机名按照 '.' 分割，存储到主机组件列表中
  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));
                });

  // 如果 SSL 上下文对象创建成功
  if (ctx_ != nullptr) {
    // 使用客户端证书和客户端私钥文件填充 SSL 上下文对象
    if (SSL_CTX_use_certificate_file(ctx_, client_cert_path.c_str(),
                                     SSL_FILETYPE_PEM) != 1 ||
        SSL_CTX_use_PrivateKey_file(ctx_, client_key_path.c_str(),
                                    SSL_FILETYPE_PEM) != 1) {
      // 如果填充失败，则释放 SSL 上下文对象并将其置为空指针
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}
    // 如果证书或私钥的使用不成功，则释放上下文并将其设置为nullptr
    if (SSL_CTX_use_certificate(ctx_, client_cert) != 1 ||
        SSL_CTX_use_PrivateKey(ctx_, client_key) != 1) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}

// SSLClient类的析构函数
inline SSLClient::~SSLClient() {
  // 如果上下文存在，则释放上下文
  if (ctx_) { SSL_CTX_free(ctx_); }
  // 确保关闭SSL连接，因为在基类析构函数中，shutdown_ssl将解析为基类函数而不是派生函数，不会释放SSL（导致内存泄漏）
  shutdown_ssl_impl(socket_, true);
}

// 检查SSLClient对象是否有效
inline bool SSLClient::is_valid() const { return ctx_; }

// 设置CA证书存储
inline void SSLClient::set_ca_cert_store(X509_STORE *ca_cert_store) {
  // 如果CA证书存储存在
  if (ca_cert_store) {
// 检查 SSL 上下文是否存在
if (ctx_) {
  // 如果 SSL 上下文存在，检查其证书存储是否与 ca_cert_store 不同
  if (SSL_CTX_get_cert_store(ctx_) != ca_cert_store) {
    // 如果不同，释放为旧证书分配的内存，并使用新的证书存储 ca_cert_store
    SSL_CTX_set_cert_store(ctx_, ca_cert_store);
  }
} else {
  // 如果 SSL 上下文不存在，释放 ca_cert_store 分配的内存
  X509_STORE_free(ca_cert_store);
}

// 返回 SSL 验证结果
inline long SSLClient::get_openssl_verify_result() const {
  return verify_result_;
}

// 返回 SSL 上下文
inline SSL_CTX *SSLClient::ssl_context() const { return ctx_; }

// 创建并连接套接字
inline bool SSLClient::create_and_connect_socket(Socket &socket, Error &error) {
  // 检查 SSL 上下文是否有效，并调用父类的创建并连接套接字方法
  return is_valid() && ClientImpl::create_and_connect_socket(socket, error);
}
// 假设 socket_mutex_ 已锁定，并且没有请求在传输中
inline bool SSLClient::connect_with_proxy(Socket &socket, Response &res,
                                          bool &success, Error &error) {
  success = true;
  Response res2;
  // 如果没有处理客户端套接字的函数返回 false
  if (!detail::process_client_socket(
          socket.sock, read_timeout_sec_, read_timeout_usec_,
          write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
            Request req2;
            req2.method = "CONNECT";
            req2.path = host_and_port_;
            return process_request(strm, req2, res2, false, error);
          })) {
    // 线程安全地关闭所有内容，因为我们假设没有请求在传输中
    shutdown_ssl(socket, true);
    shutdown_socket(socket);
    close_socket(socket);
    success = false;
```

在这个示例中，我们对代码中的每个语句进行了注释，解释了它们的作用。这有助于其他程序员理解代码的功能和实现细节。
    # 返回 false
    return false;
  }

  # 如果状态码为 407
  if (res2.status == 407) {
    # 如果代理摘要认证用户名和密码都不为空
    if (!proxy_digest_auth_username_.empty() &&
        !proxy_digest_auth_password_.empty()) {
      # 创建一个空的字符串到字符串的映射
      std::map<std::string, std::string> auth;
      # 解析响应头中的摘要认证信息，并存储到 auth 中
      if (detail::parse_www_authenticate(res2, auth, true)) {
        # 创建一个新的响应对象
        Response res3;
        # 处理客户端套接字，发送 CONNECT 请求，进行代理认证
        if (!detail::process_client_socket(
                socket.sock, read_timeout_sec_, read_timeout_usec_,
                write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
                  # 创建一个新的请求对象
                  Request req3;
                  req3.method = "CONNECT";
                  req3.path = host_and_port_;
                  # 插入摘要认证头到请求头中
                  req3.headers.insert(detail::make_digest_authentication_header(
                      req3, auth, 1, detail::random_string(10),
                      proxy_digest_auth_username_, proxy_digest_auth_password_,
                      true));
                  # 处理请求
                  return process_request(strm, req3, res3, false, error);
// 如果 SSL 客户端对象的证书和密钥加载成功
if (certs_loaded) {
  // 返回 true
  return true;
} else {
  // 如果证书和密钥加载失败
  if (!load_cert_file(cert_file, key_file)) {
    // 如果加载失败，关闭 SSL 连接，关闭套接字，返回 false
    shutdown_ssl(socket, true);
    shutdown_socket(socket);
    close_socket(socket);
    success = false;
    return false;
  } else {
    // 如果加载成功，返回 true
    return true;
  }
}
// 声明并初始化 ret 变量为 true
bool ret = true;

// 使用 std::call_once 保证初始化函数只被调用一次
std::call_once(initialize_cert_, [&]() {
  // 使用互斥锁保护临界区
  std::lock_guard<std::mutex> guard(ctx_mutex_);
  // 如果 CA 证书文件路径不为空，则加载证书文件
  if (!ca_cert_file_path_.empty()) {
    if (!SSL_CTX_load_verify_locations(ctx_, ca_cert_file_path_.c_str(),
                                       nullptr)) {
      ret = false;
    }
  } 
  // 如果 CA 证书目录路径不为空，则加载证书目录
  else if (!ca_cert_dir_path_.empty()) {
    if (!SSL_CTX_load_verify_locations(ctx_, nullptr,
                                       ca_cert_dir_path_.c_str())) {
      ret = false;
    }
  } 
  // 如果 CA 证书文件路径和目录路径都为空，则根据操作系统加载系统证书
  else {
    auto loaded = false;
    #ifdef _WIN32
    // 在 Windows 平台上加载系统证书
    loaded = detail::load_system_certs_on_windows(SSL_CTX_get_cert_store(ctx_));
    #elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#if TARGET_OS_OSX
      // 如果目标操作系统是OSX，则加载系统证书
      loaded = detail::load_system_certs_on_macos(SSL_CTX_get_cert_store(ctx_));
#endif // TARGET_OS_OSX
#endif // _WIN32
      // 如果系统证书未加载成功，则设置默认的验证路径
      if (!loaded) { SSL_CTX_set_default_verify_paths(ctx_); }
    }
  });

  return ret;
}

// 初始化SSL客户端
inline bool SSLClient::initialize_ssl(Socket &socket, Error &error) {
  // 创建SSL对象
  auto ssl = detail::ssl_new(
      socket.sock, ctx_, ctx_mutex_,
      [&](SSL *ssl2) {
        // 如果需要进行服务器证书验证
        if (server_certificate_verification_) {
          // 加载证书，如果加载失败则返回错误
          if (!load_certs()) {
            error = Error::SSLLoadingCerts;
            return false;
          }
        // 设置 SSL 连接的验证方式为 SSL_VERIFY_NONE，即不进行验证
        SSL_set_verify(ssl2, SSL_VERIFY_NONE, nullptr);
        }

        // 如果 SSL 连接或接受非阻塞连接失败，则返回 SSLConnection 错误
        if (!detail::ssl_connect_or_accept_nonblocking(
                socket.sock, ssl2, SSL_connect, connection_timeout_sec_,
                connection_timeout_usec_)) {
          error = Error::SSLConnection;
          return false;
        }

        // 如果需要进行服务器证书验证
        if (server_certificate_verification_) {
          // 获取 SSL 连接的验证结果
          verify_result_ = SSL_get_verify_result(ssl2);

          // 如果验证结果不是 X509_V_OK，则返回 SSLServerVerification 错误
          if (verify_result_ != X509_V_OK) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 获取服务器证书
          auto server_cert = SSL_get1_peer_certificate(ssl2);
          // 如果服务器证书为空指针，则设置错误类型为 SSLServerVerification，并返回 false
          if (server_cert == nullptr) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 如果服务器证书主机验证失败，则释放服务器证书内存，设置错误类型为 SSLServerVerification，并返回 false
          if (!verify_host(server_cert)) {
            X509_free(server_cert);
            error = Error::SSLServerVerification;
            return false;
          }
          // 释放服务器证书内存
          X509_free(server_cert);
        }

        // 返回 true
        return true;
      },
      // 使用 lambda 表达式设置 TLS 扩展主机名，并返回 true
      [&](SSL *ssl2) {
        SSL_set_tlsext_host_name(ssl2, host_.c_str());
        return true;
      });
# 如果存在 SSL 连接
if (ssl) {
    # 将 SSL 对象赋值给 socket 的 ssl 属性
    socket.ssl = ssl;
    # 返回 true，表示 SSL 连接成功
    return true;
}

# 关闭 socket 的 SSL 连接
shutdown_socket(socket);
# 关闭 socket
close_socket(socket);
# 返回 false，表示 SSL 连接失败
return false;
}

# 关闭 SSL 连接
inline void SSLClient::shutdown_ssl(Socket &socket, bool shutdown_gracefully) {
  # 调用 shutdown_ssl_impl 方法关闭 SSL 连接
  shutdown_ssl_impl(socket, shutdown_gracefully);
}

# 实际关闭 SSL 连接的方法
inline void SSLClient::shutdown_ssl_impl(Socket &socket, bool shutdown_gracefully) {
  # 如果 socket 的 sock 属性为无效值，且 ssl 属性为 nullptr
  if (socket.sock == INVALID_SOCKET) {
    assert(socket.ssl == nullptr);
    # 直接返回，不执行关闭操作
    return;
  }
// 如果 socket 中有 SSL 连接，则关闭 SSL 连接
if (socket.ssl) {
  // 调用 detail 命名空间中的 ssl_delete 函数来删除 SSL 连接
  detail::ssl_delete(ctx_mutex_, socket.ssl, shutdown_gracefully);
  // 将 socket 中的 SSL 指针置为空
  socket.ssl = nullptr;
}
// 断言 socket 中的 SSL 指针为空
assert(socket.ssl == nullptr);
}

// 处理 SSL 客户端的 socket 连接，并调用回调函数
inline bool
SSLClient::process_socket(const Socket &socket,
                          std::function<bool(Stream &strm)> callback) {
  // 断言 socket 中有 SSL 连接
  assert(socket.ssl);
  // 调用 detail 命名空间中的 process_client_socket_ssl 函数来处理 SSL 客户端的 socket 连接
  return detail::process_client_socket_ssl(
      socket.ssl, socket.sock, read_timeout_sec_, read_timeout_usec_,
      write_timeout_sec_, write_timeout_usec_, std::move(callback));
}

// 返回 SSL 客户端是否使用 SSL 连接
inline bool SSLClient::is_ssl() const { return true; }

// 验证服务器证书的主机名
inline bool SSLClient::verify_host(X509 *server_cert) const {
  /* Quote from RFC2818 section 3.1 "Server Identity"
# 如果证书中存在类型为dNSName的subjectAltName扩展，则必须使用该扩展作为身份标识。
# 否则，必须使用证书主体字段中（最具体的）通用名称字段。尽管使用通用名称是现有的做法，但已被弃用，鼓励证书颁发机构使用dNSName。
# 匹配使用[RFC2459]中指定的匹配规则。如果证书中存在同一类型的多个身份标识（例如，多个dNSName名称），则在该集合中的任何一个匹配都被视为可接受的。
# 名称可能包含通配符*，被视为匹配任何单个域名组件或组件片段。例如，*.a.com匹配foo.a.com但不匹配bar.foo.a.com。f*.com匹配foo.com但不匹配bar.com。
# 在某些情况下，URI被指定为IP地址而不是主机名。在这种情况下，证书中必须存在iPAddress subjectAltName，并且必须与URI中的IP完全匹配。
// 返回使用主题备用名称验证主机或使用通用名称验证主机的结果
bool SSLClient::verify_host_with_subject_alt_name(X509 *server_cert) const {
  // 初始化返回值为 false
  auto ret = false;

  // 初始化类型为通用名称
  auto type = GEN_DNS;

  // 初始化 IPv6 和 IPv4 地址结构
  struct in6_addr addr6;
  struct in_addr addr;
  // 初始化地址长度为 0
  size_t addr_len = 0;

  // 如果不是 Windows 平台
  #ifndef __MINGW32__
  // 如果主机名可以转换为 IPv6 地址
  if (inet_pton(AF_INET6, host_.c_str(), &addr6)) {
    // 设置类型为 IP 地址
    type = GEN_IPADD;
    // 设置地址长度为 IPv6 地址结构的大小
    addr_len = sizeof(struct in6_addr);
  } 
  // 如果主机名可以转换为 IPv4 地址
  else if (inet_pton(AF_INET, host_.c_str(), &addr)) {
    // 设置类型为 IP 地址
    type = GEN_IPADD;
  }
  ```
  此处代码未完整，无法完全理解上下文，因此无法为其余部分添加注释。
    # 计算结构体 in_addr 的大小
    addr_len = sizeof(struct in_addr);
  }
#endif

  # 获取服务器证书中的主体备用名称扩展
  auto alt_names = static_cast<const struct stack_st_GENERAL_NAME *>(
      X509_get_ext_d2i(server_cert, NID_subject_alt_name, nullptr, nullptr));

  # 如果存在主体备用名称扩展
  if (alt_names) {
    # 初始化匹配标志
    auto dsn_matched = false;
    auto ip_matched = false;

    # 获取主体备用名称扩展中的条目数量
    auto count = sk_GENERAL_NAME_num(alt_names);

    # 遍历主体备用名称扩展中的条目
    for (decltype(count) i = 0; i < count && !dsn_matched; i++) {
      # 获取当前条目的值
      auto val = sk_GENERAL_NAME_value(alt_names, i);
      # 如果值的类型与指定类型相符
      if (val->type == type) {
        # 获取条目的名称和长度
        auto name = (const char *)ASN1_STRING_get0_data(val->d.ia5);
        auto name_len = (size_t)ASN1_STRING_length(val->d.ia5);

        # 根据类型进行不同的处理
        switch (type) {
// 根据类型进行不同的处理
case GEN_DNS: 
    // 检查主机名是否匹配
    dsn_matched = check_host_name(name, name_len); 
    break;

case GEN_IPADD:
    // 如果 IPv6 或 IPv4 地址匹配，则设置 ip_matched 为 true
    if (!memcmp(&addr6, name, addr_len) ||
        !memcmp(&addr, name, addr_len)) {
        ip_matched = true;
    }
    break;
}

// 如果主机名或 IP 地址匹配，则设置 ret 为 true
if (dsn_matched || ip_matched) { 
    ret = true; 
}

// 释放 GENERAL_NAMES 结构体
GENERAL_NAMES_free((STACK_OF(GENERAL_NAME) *)alt_names); 
// 返回结果
return ret;
}

// 在 SSLClient 类中验证服务器证书的通用名称
inline bool SSLClient::verify_host_with_common_name(X509 *server_cert) const {
// 获取服务器证书的主题名称
const auto subject_name = X509_get_subject_name(server_cert);

// 如果主题名称不为空
if (subject_name != nullptr) {
  // 创建一个字符数组来存储主题名称
  char name[BUFSIZ];
  // 获取主题名称中的通用名称
  auto name_len = X509_NAME_get_text_by_NID(subject_name, NID_commonName,
                                            name, sizeof(name));

  // 如果成功获取到通用名称
  if (name_len != -1) {
    // 检查主机名是否匹配通用名称
    return check_host_name(name, static_cast<size_t>(name_len));
  }
}

// 如果没有匹配到通用名称，则返回false
return false;
}

// 检查主机名是否与模式匹配
inline bool SSLClient::check_host_name(const char *pattern,
                                       size_t pattern_len) const {
  // 如果主机名的长度与模式长度相等，并且内容相同，则返回true
  if (host_.size() == pattern_len && host_ == pattern) { return true; }

  // 如果主机名与模式不完全匹配，则进行通配符匹配
// 创建一个存储字符串的向量，用于存储分割后的模式组件
std::vector<std::string> pattern_components;
// 使用lambda表达式将模式按照'.'分割，并将分割后的组件存储到pattern_components中
detail::split(&pattern[0], &pattern[pattern_len], '.',
              [&](const char *b, const char *e) {
                pattern_components.emplace_back(std::string(b, e));
              });

// 如果主机组件的数量不等于模式组件的数量，则返回false
if (host_components_.size() != pattern_components.size()) { return false; }

// 创建一个迭代器，用于遍历模式组件
auto itr = pattern_components.begin();
// 遍历主机组件，与对应的模式组件进行比较
for (const auto &h : host_components_) {
  auto &p = *itr;
  // 如果模式组件不等于主机组件，并且不等于"*"，则返回false
  if (p != h && p != "*") {
    // 检查是否存在部分匹配
    auto partial_match = (p.size() > 0 && p[p.size() - 1] == '*' &&
                          !p.compare(0, p.size() - 1, h));
    if (!partial_match) { return false; }
  }
  // 移动模式组件的迭代器到下一个位置
  ++itr;
}
// 返回 true
  return true;
}
#endif

// 通用客户端实现
// 使用给定的 scheme_host_port 构造 Client 对象
inline Client::Client(const std::string &scheme_host_port)
    : Client(scheme_host_port, std::string(), std::string()) {}

// 使用给定的 scheme_host_port, client_cert_path, client_key_path 构造 Client 对象
inline Client::Client(const std::string &scheme_host_port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path) {
  // 定义正则表达式，用于解析 scheme_host_port
  const static std::regex re(
      R"((?:([a-z]+):\/\/)?(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)");

  // 使用正则表达式匹配 scheme_host_port
  std::smatch m;
  if (std::regex_match(scheme_host_port, m, re)) {
    // 获取 scheme
    auto scheme = m[1].str();

    // 如果支持 OpenSSL 并且 scheme 不为空且不是 "http" 或 "https"，则执行以下操作
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    if (!scheme.empty() && (scheme != "http" && scheme != "https")) {
// 如果不是以http开头的scheme，则抛出异常
#else
    if (!scheme.empty() && scheme != "http") {
#endif
#ifndef CPPHTTPLIB_NO_EXCEPTIONS
      // 如果不支持该scheme，则抛出无效参数异常
      std::string msg = "'" + scheme + "' scheme is not supported.";
      throw std::invalid_argument(msg);
#endif
      // 返回空
      return;
    }

    // 判断是否是https
    auto is_ssl = scheme == "https";

    // 获取主机名
    auto host = m[2].str();
    // 如果主机名为空，则使用第三个匹配的字符串作为主机名
    if (host.empty()) { host = m[3].str(); }

    // 获取端口号字符串
    auto port_str = m[4].str();
    // 如果端口号字符串不为空，则转换为整数，否则根据是否是https来确定默认端口号
    auto port = !port_str.empty() ? std::stoi(port_str) : (is_ssl ? 443 : 80);

    // 如果是https，则使用OpenSSL支持
    if (is_ssl) {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 如果是 SSL 连接
if (is_ssl) {
  // 如果有客户端证书和密钥路径，则创建 SSLClient 对象
  cli_ = detail::make_unique<SSLClient>(host, port, client_cert_path, client_key_path);
  // 设置 is_ssl_ 为 true
  is_ssl_ = is_ssl;
} else {
  // 如果不是 SSL 连接，创建普通的 ClientImpl 对象
  cli_ = detail::make_unique<ClientImpl>(host, port, client_cert_path, client_key_path);
}
// 如果没有指定端口，则默认使用 80 端口创建 ClientImpl 对象
} else {
  cli_ = detail::make_unique<ClientImpl>(scheme_host_port, 80, client_cert_path, client_key_path);
}

// 使用指定的主机和端口创建 ClientImpl 对象
inline Client::Client(const std::string &host, int port)
    : cli_(detail::make_unique<ClientImpl>(host, port)) {}

// 使用指定的主机、端口、客户端证书路径和客户端密钥路径创建 ClientImpl 对象
inline Client::Client(const std::string &host, int port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path)
// 使用给定的主机、端口、客户端证书路径和客户端密钥路径创建一个新的客户端对象
cli_(detail::make_unique<ClientImpl>(host, port, client_cert_path, client_key_path)) {}

// 客户端对象的析构函数
inline Client::~Client() {}

// 检查客户端对象是否有效
inline bool Client::is_valid() const {
  return cli_ != nullptr && cli_->is_valid();
}

// 发起 GET 请求，获取指定路径的资源
inline Result Client::Get(const std::string &path) { return cli_->Get(path); }

// 发起带有自定义头部的 GET 请求，获取指定路径的资源
inline Result Client::Get(const std::string &path, const Headers &headers) {
  return cli_->Get(path, headers);
}

// 发起带有进度回调的 GET 请求，获取指定路径的资源
inline Result Client::Get(const std::string &path, Progress progress) {
  return cli_->Get(path, std::move(progress));
}

// 发起带有自定义头部和进度回调的 GET 请求，获取指定路径的资源
inline Result Client::Get(const std::string &path, const Headers &headers, Progress progress) {
  return cli_->Get(path, headers, std::move(progress));
}
// 使用给定的路径和内容接收器从服务器获取数据，并返回结果
inline Result Client::Get(const std::string &path, ContentReceiver content_receiver) {
  return cli_->Get(path, std::move(content_receiver));
}

// 使用给定的路径、头部和内容接收器从服务器获取数据，并返回结果
inline Result Client::Get(const std::string &path, const Headers &headers, ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(content_receiver));
}

// 使用给定的路径、内容接收器和进度从服务器获取数据，并返回结果
inline Result Client::Get(const std::string &path, ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(content_receiver), std::move(progress));
}

// 使用给定的路径、头部、内容接收器和进度从服务器获取数据，并返回结果
inline Result Client::Get(const std::string &path, const Headers &headers, ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(content_receiver), std::move(progress));
}

// 使用给定的路径、响应处理器和内容接收器从服务器获取数据，并返回结果
inline Result Client::Get(const std::string &path, ResponseHandler response_handler, ContentReceiver content_receiver) {
// 使用给定的路径和响应处理器、内容接收器，调用 cli_ 的 Get 方法，并返回结果
return cli_->Get(path, std::move(response_handler), std::move(content_receiver));
}

// 使用给定的路径、请求头、响应处理器、内容接收器，调用 cli_ 的 Get 方法，并返回结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(response_handler), std::move(content_receiver));
}

// 使用给定的路径、响应处理器、内容接收器、进度，调用 cli_ 的 Get 方法，并返回结果
inline Result Client::Get(const std::string &path,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(response_handler), std::move(content_receiver), std::move(progress));
}

// 使用给定的路径、请求头、响应处理器、内容接收器、进度，调用 cli_ 的 Get 方法，并返回结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(response_handler), std::move(content_receiver), std::move(progress));
}
// 客户端发送 GET 请求，获取指定路径的资源，并传递参数、请求头和进度回调函数
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers, Progress progress) {
  return cli_->Get(path, params, headers, progress);
}

// 客户端发送 GET 请求，获取指定路径的资源，并传递参数、请求头、内容接收器和进度回调函数
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, content_receiver, progress);
}

// 客户端发送 GET 请求，获取指定路径的资源，并传递参数、请求头、响应处理器、内容接收器和进度回调函数
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, response_handler, content_receiver,
                   progress);
}

// 客户端发送 HEAD 请求，获取指定路径的资源头信息
inline Result Client::Head(const std::string &path) { return cli_->Head(path); }

// 客户端发送 HEAD 请求，获取指定路径的资源头信息，并传递请求头
inline Result Client::Head(const std::string &path, const Headers &headers) {
// 使用 HTTP HEAD 方法发送请求，返回结果
inline Result Client::Head(const std::string &path, const Headers &headers) {
  return cli_->Head(path, headers);
}

// 使用 HTTP POST 方法发送请求，不带自定义头部，返回结果
inline Result Client::Post(const std::string &path) { return cli_->Post(path); }

// 使用 HTTP POST 方法发送请求，带自定义头部，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers) {
  return cli_->Post(path, headers);
}

// 使用 HTTP POST 方法发送请求，带请求体、内容长度和内容类型，返回结果
inline Result Client::Post(const std::string &path, const char *body,
                           size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_length, content_type);
}

// 使用 HTTP POST 方法发送请求，带自定义头部、请求体、内容长度和内容类型，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const char *body, size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, headers, body, content_length, content_type);
}

// 使用 HTTP POST 方法发送请求，带请求体和内容类型，返回结果
inline Result Client::Post(const std::string &path, const std::string &body,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_type);
}
// 客户端发送 POST 请求，传入路径、请求头、请求体、内容类型，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const std::string &body,
                           const std::string &content_type) {
  return cli_->Post(path, headers, body, content_type);
}

// 客户端发送 POST 请求，传入路径、内容长度、内容提供者、内容类型，返回结果
inline Result Client::Post(const std::string &path, size_t content_length,
                           ContentProvider content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, content_length, std::move(content_provider),
                    content_type);
}

// 客户端发送 POST 请求，传入路径、无长度的内容提供者、内容类型，返回结果
inline Result Client::Post(const std::string &path,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, std::move(content_provider), content_type);
}

// 客户端发送 POST 请求，传入路径、请求头、内容长度、内容提供者、内容类型，返回结果
// 使用给定的路径、头部、内容长度、内容提供者和内容类型进行 POST 请求
inline Result Client::Post(const std::string &path, const Headers &headers,
                           size_t content_length, ContentProvider content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, headers, content_length, std::move(content_provider),
                    content_type);
}

// 使用给定的路径、头部、内容提供者和内容类型进行 POST 请求
inline Result Client::Post(const std::string &path, const Headers &headers,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, headers, std::move(content_provider), content_type);
}

// 使用给定的路径和参数进行 POST 请求
inline Result Client::Post(const std::string &path, const Params &params) {
  return cli_->Post(path, params);
}

// 使用给定的路径、头部和参数进行 POST 请求
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const Params &params) {
  return cli_->Post(path, headers, params);
}

// 使用给定的路径和多部分表单数据项进行 POST 请求
inline Result Client::Post(const std::string &path,
                           const MultipartFormDataItems &items) {
  return cli_->Post(path, items);
}
// 使用给定的路径、头部和多部分表单数据项发送 POST 请求
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items) {
  return cli_->Post(path, headers, items);
}

// 使用给定的路径、头部、多部分表单数据项和边界发送 POST 请求
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items,
                           const std::string &boundary) {
  return cli_->Post(path, headers, items, boundary);
}

// 使用给定的路径、头部、多部分表单数据项和多部分表单数据提供者项发送 POST 请求
inline Result
Client::Post(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items,
             const MultipartFormDataProviderItems &provider_items) {
  return cli_->Post(path, headers, items, provider_items);
}

// 使用给定的路径发送 PUT 请求
inline Result Client::Put(const std::string &path) { return cli_->Put(path); }

// 使用给定的路径、请求体、内容长度和内容类型发送 PUT 请求
inline Result Client::Put(const std::string &path, const char *body,
                          size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_length, content_type);
}
// 客户端发送 PUT 请求，传入路径、请求头、请求体、请求体长度、内容类型，返回请求结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const char *body, size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_length, content_type);
}

// 客户端发送 PUT 请求，传入路径、请求体、内容类型，返回请求结果
inline Result Client::Put(const std::string &path, const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_type);
}

// 客户端发送 PUT 请求，传入路径、请求头、请求体、内容类型，返回请求结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_type);
}

// 客户端发送 PUT 请求，传入路径、请求体长度、内容提供者、内容类型，返回请求结果
inline Result Client::Put(const std::string &path, size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, content_length, std::move(content_provider),
                   content_type);
}
// 客户端发送 PUT 请求，使用内容提供器和内容类型
inline Result Client::Put(const std::string &path,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, std::move(content_provider), content_type);
}

// 客户端发送 PUT 请求，使用头部信息、内容长度、内容提供器和内容类型
inline Result Client::Put(const std::string &path, const Headers &headers,
                          size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, content_length, std::move(content_provider),
                   content_type);
}

// 客户端发送 PUT 请求，使用头部信息、内容提供器和内容类型
inline Result Client::Put(const std::string &path, const Headers &headers,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, std::move(content_provider), content_type);
}

// 客户端发送 PUT 请求，使用路径和参数
inline Result Client::Put(const std::string &path, const Params &params) {
  return cli_->Put(path, params);
}
// 客户端发送 PUT 请求，传递路径、请求头和参数，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const Params &params) {
  return cli_->Put(path, headers, params);
}

// 客户端发送 PUT 请求，传递路径和多部分表单数据项，返回结果
inline Result Client::Put(const std::string &path,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, items);
}

// 客户端发送 PUT 请求，传递路径、请求头和多部分表单数据项，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, headers, items);
}

// 客户端发送 PUT 请求，传递路径、请求头、多部分表单数据项和边界，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items,
                          const std::string &boundary) {
  return cli_->Put(path, headers, items, boundary);
}

// 客户端发送 PUT 请求，传递路径、请求头和多部分表单数据项，返回结果
inline Result
Client::Put(const std::string &path, const Headers &headers,
// 使用给定的多部分表单数据项和提供者项发送 PUT 请求
inline Result Client::Put(const std::string &path, const Headers &headers,
            const MultipartFormDataItems &items,
            const MultipartFormDataProviderItems &provider_items) {
  return cli_->Put(path, headers, items, provider_items);
}

// 发送 PATCH 请求，不带请求体
inline Result Client::Patch(const std::string &path) {
  return cli_->Patch(path);
}

// 发送 PATCH 请求，带指定的请求体内容和长度
inline Result Client::Patch(const std::string &path, const char *body,
                            size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_length, content_type);
}

// 发送 PATCH 请求，带指定的请求头、请求体内容和长度
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const char *body, size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_length, content_type);
}

// 发送 PATCH 请求，带指定的请求体内容和类型
inline Result Client::Patch(const std::string &path, const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_type);
}
// 客户端发送 PATCH 请求，传入路径、请求头、请求体、内容类型，返回请求结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_type);
}

// 客户端发送 PATCH 请求，传入路径、内容长度、内容提供者、内容类型，返回请求结果
inline Result Client::Patch(const std::string &path, size_t content_length,
                            ContentProvider content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, content_length, std::move(content_provider),
                     content_type);
}

// 客户端发送 PATCH 请求，传入路径、无长度的内容提供者、内容类型，返回请求结果
inline Result Client::Patch(const std::string &path,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, std::move(content_provider), content_type);
}

// 客户端发送 PATCH 请求，传入路径、请求头、内容长度、内容提供者、内容类型，返回请求结果
// 使用给定的内容类型发送 PATCH 请求
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            size_t content_length, ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, content_length, std::move(content_provider),
                     content_type);
}

// 发送 PATCH 请求
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, std::move(content_provider), content_type);
}

// 发送 DELETE 请求
inline Result Client::Delete(const std::string &path) {
  return cli_->Delete(path);
}

// 发送带有自定义头部的 DELETE 请求
inline Result Client::Delete(const std::string &path, const Headers &headers) {
  return cli_->Delete(path, headers);
}

// 发送带有请求体和内容类型的 DELETE 请求
inline Result Client::Delete(const std::string &path, const char *body,
                             size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_length, content_type);
}
// 使用给定的路径、请求头、请求体、内容长度和内容类型发送 DELETE 请求
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const char *body, size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_length, content_type);
}

// 使用给定的路径、请求体和内容类型发送 DELETE 请求
inline Result Client::Delete(const std::string &path, const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_type);
}

// 使用给定的路径、请求头、请求体和内容类型发送 DELETE 请求
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_type);
}

// 发送 OPTIONS 请求以获取给定路径的可用选项
inline Result Client::Options(const std::string &path) {
  return cli_->Options(path);
}

// 使用给定的路径和请求头发送 OPTIONS 请求以获取可用选项
inline Result Client::Options(const std::string &path, const Headers &headers) {
  return cli_->Options(path, headers);
}
// 发送请求并接收响应，返回操作是否成功
inline bool Client::send(Request &req, Response &res, Error &error) {
  return cli_->send(req, res, error);
}

// 发送请求并返回结果
inline Result Client::send(const Request &req) { return cli_->send(req); }

// 检查套接字是否打开
inline size_t Client::is_socket_open() const { return cli_->is_socket_open(); }

// 返回套接字
inline socket_t Client::socket() const { return cli_->socket(); }

// 停止客户端
inline void Client::stop() { cli_->stop(); }

// 设置主机名和地址的映射关系
inline void
Client::set_hostname_addr_map(std::map<std::string, std::string> addr_map) {
  cli_->set_hostname_addr_map(std::move(addr_map));
}

// 设置默认的请求头
inline void Client::set_default_headers(Headers headers) {
  cli_->set_default_headers(std::move(headers));
}
// 设置客户端的地址族
inline void Client::set_address_family(int family) {
  cli_->set_address_family(family);
}

// 设置客户端的 TCP NoDelay 选项
inline void Client::set_tcp_nodelay(bool on) { cli_->set_tcp_nodelay(on); }

// 设置客户端的套接字选项
inline void Client::set_socket_options(SocketOptions socket_options) {
  cli_->set_socket_options(std::move(socket_options));
}

// 设置客户端的连接超时时间
inline void Client::set_connection_timeout(time_t sec, time_t usec) {
  cli_->set_connection_timeout(sec, usec);
}

// 设置客户端的读取超时时间
inline void Client::set_read_timeout(time_t sec, time_t usec) {
  cli_->set_read_timeout(sec, usec);
}
// 设置写超时时间
inline void Client::set_write_timeout(time_t sec, time_t usec) {
  cli_->set_write_timeout(sec, usec);
}

// 设置基本身份验证
inline void Client::set_basic_auth(const std::string &username,
                                   const std::string &password) {
  cli_->set_basic_auth(username, password);
}

// 设置令牌身份验证
inline void Client::set_bearer_token_auth(const std::string &token) {
  cli_->set_bearer_token_auth(token);
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置摘要身份验证
inline void Client::set_digest_auth(const std::string &username,
                                    const std::string &password) {
  cli_->set_digest_auth(username, password);
}
#endif

// 设置保持连接
inline void Client::set_keep_alive(bool on) { cli_->set_keep_alive(on); }

// 设置是否跟随重定向
inline void Client::set_follow_location(bool on) {
// 设置是否跟随重定向
inline void Client::set_follow_location(bool on) { cli_->set_follow_location(on); }

// 设置是否进行 URL 编码
inline void Client::set_url_encode(bool on) { cli_->set_url_encode(on); }

// 设置是否启用压缩
inline void Client::set_compress(bool on) { cli_->set_compress(on); }

// 设置是否启用解压缩
inline void Client::set_decompress(bool on) { cli_->set_decompress(on); }

// 设置网络接口
inline void Client::set_interface(const std::string &intf) {
  cli_->set_interface(intf);
}

// 设置代理服务器
inline void Client::set_proxy(const std::string &host, int port) {
  cli_->set_proxy(host, port);
}

// 设置代理服务器的基本认证信息
inline void Client::set_proxy_basic_auth(const std::string &username,
                                         const std::string &password) {
  cli_->set_proxy_basic_auth(username, password);
}
// 设置代理服务器的身份验证方式为 Bearer Token
inline void Client::set_proxy_bearer_token_auth(const std::string &token) {
  cli_->set_proxy_bearer_token_auth(token);
}
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理服务器的身份验证方式为 Digest Auth（仅在 OpenSSL 支持的情况下）
inline void Client::set_proxy_digest_auth(const std::string &username,
                                          const std::string &password) {
  cli_->set_proxy_digest_auth(username, password);
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 启用/禁用服务器证书验证（仅在 OpenSSL 支持的情况下）
inline void Client::enable_server_certificate_verification(bool enabled) {
  cli_->enable_server_certificate_verification(enabled);
}
#endif

// 设置日志记录器
inline void Client::set_logger(Logger logger) { cli_->set_logger(logger); }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书文件路径（仅在 OpenSSL 支持的情况下）
inline void Client::set_ca_cert_path(const std::string &ca_cert_file_path,
```
// 设置客户端的 CA 证书路径
void Client::set_ca_cert_path(const std::string &ca_cert_file_path, const std::string &ca_cert_dir_path) {
  cli_->set_ca_cert_path(ca_cert_file_path, ca_cert_dir_path);
}

// 设置客户端的 CA 证书存储
void Client::set_ca_cert_store(X509_STORE *ca_cert_store) {
  // 如果是 SSL 连接，调用 SSLClient 类的 set_ca_cert_store 方法
  if (is_ssl_) {
    static_cast<SSLClient &>(*cli_).set_ca_cert_store(ca_cert_store);
  } else {
    // 否则调用普通客户端的 set_ca_cert_store 方法
    cli_->set_ca_cert_store(ca_cert_store);
  }
}

// 获取 OpenSSL 验证结果
long Client::get_openssl_verify_result() const {
  // 如果是 SSL 连接，调用 SSLClient 类的 get_openssl_verify_result 方法
  if (is_ssl_) {
    return static_cast<SSLClient &>(*cli_).get_openssl_verify_result();
  }
  // 否则返回 -1，表示没有匹配到任何 X509_V_ERR_???
  return -1; // NOTE: -1 doesn't match any of X509_V_ERR_???
}

// 获取 SSL 上下文
SSL_CTX *Client::ssl_context() const {
// 如果使用了 SSL，则返回 SSLClient 对象的 SSL 上下文，否则返回空指针
if (is_ssl_) { return static_cast<SSLClient &>(*cli_).ssl_context(); }
// 如果未使用 SSL，则返回空指针
return nullptr;
}
#endif

// ----------------------------------------------------------------------------

} // namespace httplib

// 如果在 Windows 平台且定义了 CPPHTTPLIB_USE_POLL，则取消对 poll 函数的定义
#if defined(_WIN32) && defined(CPPHTTPLIB_USE_POLL)
#undef poll
#endif

#endif // CPPHTTPLIB_HTTPLIB_H
```