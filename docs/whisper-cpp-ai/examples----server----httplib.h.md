# `whisper.cpp\examples\server\httplib.h`

```cpp
//
//  httplib.h
//
//  版权所有 2023 年 Yuji Hirose。保留所有权利。
//  MIT 许可证
//

#ifndef CPPHTTPLIB_HTTPLIB_H
#define CPPHTTPLIB_HTTPLIB_H

#define CPPHTTPLIB_VERSION "0.14.1"

/*
 * 配置
 */

#ifndef CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND
#define CPPHTTPLIB_KEEPALIVE_TIMEOUT_SECOND 5
#endif

#ifndef CPPHTTPLIB_KEEPALIVE_MAX_COUNT
#define CPPHTTPLIB_KEEPALIVE_MAX_COUNT 5
#endif

#ifndef CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND
#define CPPHTTPLIB_CONNECTION_TIMEOUT_SECOND 300
#endif

#ifndef CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND
#define CPPHTTPLIB_CONNECTION_TIMEOUT_USECOND 0
#endif

#ifndef CPPHTTPLIB_READ_TIMEOUT_SECOND
#define CPPHTTPLIB_READ_TIMEOUT_SECOND 5
#endif

#ifndef CPPHTTPLIB_READ_TIMEOUT_USECOND
#define CPPHTTPLIB_READ_TIMEOUT_USECOND 0
#endif

#ifndef CPPHTTPLIB_WRITE_TIMEOUT_SECOND
#define CPPHTTPLIB_WRITE_TIMEOUT_SECOND 5
#endif

#ifndef CPPHTTPLIB_WRITE_TIMEOUT_USECOND
#define CPPHTTPLIB_WRITE_TIMEOUT_USECOND 0
#endif

#ifndef CPPHTTPLIB_IDLE_INTERVAL_SECOND
#define CPPHTTPLIB_IDLE_INTERVAL_SECOND 0
#endif

#ifndef CPPHTTPLIB_IDLE_INTERVAL_USECOND
#ifdef _WIN32
#define CPPHTTPLIB_IDLE_INTERVAL_USECOND 10000
#else
#define CPPHTTPLIB_IDLE_INTERVAL_USECOND 0
#endif
#endif

#ifndef CPPHTTPLIB_REQUEST_URI_MAX_LENGTH
#define CPPHTTPLIB_REQUEST_URI_MAX_LENGTH 8192
#endif

#ifndef CPPHTTPLIB_HEADER_MAX_LENGTH
#define CPPHTTPLIB_HEADER_MAX_LENGTH 8192
#endif

#ifndef CPPHTTPLIB_REDIRECT_MAX_COUNT
#define CPPHTTPLIB_REDIRECT_MAX_COUNT 20
#endif

#ifndef CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT
#define CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT 1024
#endif

#ifndef CPPHTTPLIB_PAYLOAD_MAX_LENGTH
#define CPPHTTPLIB_PAYLOAD_MAX_LENGTH ((std::numeric_limits<size_t>::max)())
#endif

#ifndef CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH
#define CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH 8192
#endif

#ifndef CPPHTTPLIB_TCP_NODELAY
#define CPPHTTPLIB_TCP_NODELAY false
#endif

#ifndef CPPHTTPLIB_RECV_BUFSIZ
#ifndef CPPHTTPLIB_RECV_BUFSIZ
#define CPPHTTPLIB_RECV_BUFSIZ size_t(4096u)
#endif

#ifndef CPPHTTPLIB_COMPRESSION_BUFSIZ
#define CPPHTTPLIB_COMPRESSION_BUFSIZ size_t(16384u)
#endif

#ifndef CPPHTTPLIB_THREAD_POOL_COUNT
#define CPPHTTPLIB_THREAD_POOL_COUNT                                           \
  ((std::max)(8u, std::thread::hardware_concurrency() > 0                      \
                      ? std::thread::hardware_concurrency() - 1                \
                      : 0))
#endif

#ifndef CPPHTTPLIB_RECV_FLAGS
#define CPPHTTPLIB_RECV_FLAGS 0
#endif

#ifndef CPPHTTPLIB_SEND_FLAGS
#define CPPHTTPLIB_SEND_FLAGS 0
#endif

#ifndef CPPHTTPLIB_LISTEN_BACKLOG
#define CPPHTTPLIB_LISTEN_BACKLOG 5
#endif

/*
 * Headers
 */

#ifdef _WIN32
#ifndef _CRT_SECURE_NO_WARNINGS
#define _CRT_SECURE_NO_WARNINGS
#endif //_CRT_SECURE_NO_WARNINGS

#ifndef _CRT_NONSTDC_NO_DEPRECATE
#define _CRT_NONSTDC_NO_DEPRECATE
#endif //_CRT_NONSTDC_NO_DEPRECATE

#if defined(_MSC_VER)
#if _MSC_VER < 1900
#error Sorry, Visual Studio versions prior to 2015 are not supported
#endif

#pragma comment(lib, "ws2_32.lib")

#ifdef _WIN64
using ssize_t = __int64;
#else
using ssize_t = long;
#endif
#endif // _MSC_VER

#ifndef S_ISREG
#define S_ISREG(m) (((m)&S_IFREG) == S_IFREG)
#endif // S_ISREG

#ifndef S_ISDIR
#define S_ISDIR(m) (((m)&S_IFDIR) == S_IFDIR)
#endif // S_ISDIR

#ifndef NOMINMAX
#define NOMINMAX
#endif // NOMINMAX

#include <io.h>
#include <winsock2.h>
#include <ws2tcpip.h>

#ifndef WSA_FLAG_NO_HANDLE_INHERIT
#define WSA_FLAG_NO_HANDLE_INHERIT 0x80
#endif

#ifndef strcasecmp
#define strcasecmp _stricmp
#endif // strcasecmp

using socket_t = SOCKET;
#ifdef CPPHTTPLIB_USE_POLL
#define poll(fds, nfds, timeout) WSAPoll(fds, nfds, timeout)
#endif

#else // not _WIN32

#include <arpa/inet.h>
#if !defined(_AIX) && !defined(__MVS__)
#include <ifaddrs.h>
#endif
#ifdef __MVS__
#include <strings.h>
#ifndef NI_MAXHOST
#define NI_MAXHOST 1025
#endif
#endif
#include <net/if.h>
#include <netdb.h>
#include <netinet/in.h>
#ifdef __linux__
#include <resolv.h>
#endif
#include <netinet/tcp.h>
#ifdef CPPHTTPLIB_USE_POLL
#include <poll.h>
#endif
#include <csignal>
#include <pthread.h>
#include <sys/mman.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

using socket_t = int;
#ifndef INVALID_SOCKET
#define INVALID_SOCKET (-1)
#endif
#endif //_WIN32



#ifdef __linux__
// 包含 Linux 系统的网络解析头文件
#include <resolv.h>
#endif
// 包含 TCP 协议相关的头文件
#include <netinet/tcp.h>
#ifdef CPPHTTPLIB_USE_POLL
// 如果使用 poll，则包含 poll 头文件
#include <poll.h>
#endif
// 包含 C 信号处理相关的头文件
#include <csignal>
// 包含 POSIX 线程相关的头文件
#include <pthread.h>
// 包含内存映射相关的头文件
#include <sys/mman.h>
// 包含 select 函数相关的头文件
#include <sys/select.h>
// 包含 socket 相关的头文件
#include <sys/socket.h>
// 包含 UNIX 域套接字相关的头文件
#include <sys/un.h>
// 包含 POSIX 标准函数相关的头文件
#include <unistd.h>

// 定义 socket_t 类型为 int
using socket_t = int;
// 如果未定义 INVALID_SOCKET，则定义为 -1
#ifndef INVALID_SOCKET
#define INVALID_SOCKET (-1)
#endif
#endif //_WIN32



#include <algorithm>
#include <array>
#include <atomic>
#include <cassert>
#include <cctype>
#include <climits>
#include <condition_variable>
#include <cstring>
#include <errno.h>
#include <fcntl.h>
#include <fstream>
#include <functional>
#include <iomanip>
#include <iostream>
#include <list>
#include <map>
#include <memory>
#include <mutex>
#include <random>
#include <regex>
#include <set>
#include <sstream>
#include <string>
#include <sys/stat.h>
#include <thread>
#include <unordered_map>
#include <unordered_set>
#include <utility>



// 包含 C++ 标准库相关的头文件
#include <algorithm>
#include <array>
#include <atomic>
#include <cassert>
#include <cctype>
#include <climits>
#include <condition_variable>
#include <cstring>
#include <errno.h>
#include <fcntl.h>
#include <fstream>
#include <functional>
#include <iomanip>
#include <iostream>
#include <list>
#include <map>
#include <memory>
#include <mutex>
#include <random>
#include <regex>
#include <set>
#include <sstream>
#include <string>
#include <sys/stat.h>
#include <thread>
#include <unordered_map>
#include <unordered_set>
#include <utility>
``` 


#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef _WIN32
#include <wincrypt.h>

// these are defined in wincrypt.h and it breaks compilation if BoringSSL is
// used
#undef X509_NAME
#undef X509_CERT_PAIR
#undef X509_EXTENSIONS
#undef PKCS7_SIGNER_INFO

#ifdef _MSC_VER
#pragma comment(lib, "crypt32.lib")
#pragma comment(lib, "cryptui.lib")
#endif
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#include <TargetConditionals.h>
#if TARGET_OS_OSX
#include <CoreFoundation/CoreFoundation.h>
#include <Security/Security.h>
#endif // TARGET_OS_OSX
#endif // _WIN32

#include <openssl/err.h>
#include <openssl/evp.h>
#include <openssl/ssl.h>
#include <openssl/x509v3.h>

#if defined(_WIN32) && defined(OPENSSL_USE_APPLINK)
#include <openssl/applink.c>
#endif

#include <iostream>
#include <sstream>

#if OPENSSL_VERSION_NUMBER < 0x1010100fL
#error Sorry, OpenSSL versions prior to 1.1.1 are not supported
#elif OPENSSL_VERSION_NUMBER < 0x30000000L
#define SSL_get1_peer_certificate SSL_get_peer_certificate
#endif

#endif



#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef _WIN32
// 如果是 Windows 系统，包含 Windows 加密相关的头文件
#include <wincrypt.h>

// 这些在 wincrypt.h 中定义，如果使用 BoringSSL 会导致编译错误
#undef X509_NAME
#undef X509_CERT_PAIR
#undef X509_EXTENSIONS
#undef PKCS7_SIGNER_INFO

#ifdef _MSC_VER
// 如果是 Microsoft 编译器，添加链接库
#pragma comment(lib, "crypt32.lib")
#pragma comment(lib, "cryptui.lib")
#endif
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#include <TargetConditionals.h>
#if TARGET_OS_OSX
// 如果是 macOS 系统，包含 macOS 相关的头文件
#include <CoreFoundation/CoreFoundation.h>
#include <Security/Security.h>
#endif // TARGET_OS_OSX
#endif // _WIN32

// 包含 OpenSSL 错误处理相关的头文件
#include <openssl/err.h>
// 包含 OpenSSL 加密相关的头文件
#include <openssl/evp.h>
// 包含 OpenSSL SSL 相关的头文件
#include <openssl/ssl.h>
// 包含 OpenSSL X.509 v3 相关的头文件
#include <openssl/x509v3>

#if defined(_WIN32) && defined(OPENSSL_USE_APPLINK)
#include <openssl/applink.c>
#endif

#include <iostream>
#include <sstream>

#if OPENSSL_VERSION_NUMBER < 0x1010100fL
// 如果 OpenSSL 版本低于 1.1.1，则报错
#error Sorry, OpenSSL versions prior to 1.1.1 are not supported
#elif OPENSSL_VERSION_NUMBER < 0x30000000L
// 如果 OpenSSL 版本低于 3.0.0，则定义 SSL_get1_peer_certificate 为 SSL_get_peer_certificate
#define SSL_get1_peer_certificate SSL_get_peer_certificate
#endif

#endif



#ifdef CPPHTTPLIB_ZLIB_SUPPORT
#include <zlib.h>



#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 如果支持 ZLIB，则包含 ZLIB 头文件
#include <zlib.h>
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
#include <brotli/decode.h>
#include <brotli/encode.h>
#endif

/*
 * Declaration
 */
// 声明 httplib 命名空间
namespace httplib {

// 声明 detail 命名空间
namespace detail {

/*
 * Backport std::make_unique from C++14.
 *
 * NOTE: This code came up with the following stackoverflow post:
 * https://stackoverflow.com/questions/10149840/c-arrays-and-make-unique
 *
 */

// 实现 make_unique 函数模板，用于创建非数组类型的 unique_ptr
template <class T, class... Args>
typename std::enable_if<!std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(Args &&...args) {
  return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 实现 make_unique 函数模板，用于创建数组类型的 unique_ptr
template <class T>
typename std::enable_if<std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(std::size_t n) {
  typedef typename std::remove_extent<T>::type RT;
  return std::unique_ptr<T>(new RT[n]);
}

// 定义一个函数对象 ci，用于比较两个字符串的大小写不敏感排序
struct ci {
  bool operator()(const std::string &s1, const std::string &s2) const {
    return std::lexicographical_compare(s1.begin(), s1.end(), s2.begin(),
                                        s2.end(),
                                        [](unsigned char c1, unsigned char c2) {
                                          return ::tolower(c1) < ::tolower(c2);
                                        });
  }
};

// 基于 "http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4189" 实现的结构体 scope_exit
struct scope_exit {
  explicit scope_exit(std::function<void(void)> &&f)
      : exit_function(std::move(f)), execute_on_destruction{true} {}

  scope_exit(scope_exit &&rhs)
      : exit_function(std::move(rhs.exit_function)),
        execute_on_destruction{rhs.execute_on_destruction} {
    rhs.release();
  }

  ~scope_exit() {
    if (execute_on_destruction) { this->exit_function(); }
  }

  void release() { this->execute_on_destruction = false; }

private:
  scope_exit(const scope_exit &) = delete;
  void operator=(const scope_exit &) = delete;
  scope_exit &operator=(scope_exit &&) = delete;

  std::function<void(void)> exit_function;
  bool execute_on_destruction;
};

} // namespace detail
// 使用 std::multimap 定义 Headers 类型，键和值都是字符串，使用自定义的不区分大小写比较器
using Headers = std::multimap<std::string, std::string, detail::ci>;

// 使用 std::multimap 定义 Params 类型，键和值都是字符串
using Params = std::multimap<std::string, std::string>;
// 使用 std::smatch 定义 Match 类型
using Match = std::smatch;

// 定义 Progress 类型为接受两个 uint64_t 参数并返回布尔值的函数对象
using Progress = std::function<bool(uint64_t current, uint64_t total)>;

// 定义 ResponseHandler 类型为接受 Response 对象并返回布尔值的函数对象
struct Response;
using ResponseHandler = std::function<bool(const Response &response)>;

// 定义 MultipartFormData 结构体，包含四个字符串成员变量
struct MultipartFormData {
  std::string name;
  std::string content;
  std::string filename;
  std::string content_type;
};
// 使用 std::vector 定义 MultipartFormDataItems 类型
using MultipartFormDataItems = std::vector<MultipartFormData>;
// 使用 std::multimap 定义 MultipartFormDataMap 类型，键为字符串，值为 MultipartFormData 结构体
using MultipartFormDataMap = std::multimap<std::string, MultipartFormData>;

// 定义 DataSink 类
class DataSink {
public:
  // 默认构造函数初始化 os 和 sb_
  DataSink() : os(&sb_), sb_(*this) {}

  // 禁用拷贝构造函数和赋值运算符
  DataSink(const DataSink &) = delete;
  DataSink &operator=(const DataSink &) = delete;
  DataSink(DataSink &&) = delete;
  DataSink &operator=(DataSink &&) = delete;

  // 函数对象成员变量，用于写入数据、完成操作和完成带尾部的操作
  std::function<bool(const char *data, size_t data_len)> write;
  std::function<void()> done;
  std::function<void(const Headers &trailer)> done_with_trailer;
  // 输出流对象 os
  std::ostream os;

private:
  // 内部类 data_sink_streambuf 继承自 std::streambuf
  class data_sink_streambuf : public std::streambuf {
  public:
    // 构造函数初始化 sink_
    explicit data_sink_streambuf(DataSink &sink) : sink_(sink) {}

  protected:
    // 重写 xsputn 方法，将数据写入 sink_
    std::streamsize xsputn(const char *s, std::streamsize n) {
      sink_.write(s, static_cast<size_t>(n));
      return n;
    }

  private:
    // DataSink 引用
    DataSink &sink_;
  };

  // data_sink_streambuf 对象
  data_sink_streambuf sb_;
};

// 定义 ContentProvider 类型为接受 offset、length、DataSink 对象并返回布尔值的函数对象
using ContentProvider =
    std::function<bool(size_t offset, size_t length, DataSink &sink)>;

// 定义 ContentProviderWithoutLength 类型为接受 offset、DataSink 对象并返回布尔值的函数对象
using ContentProviderWithoutLength =
    std::function<bool(size_t offset, DataSink &sink)>;

// 定义 ContentProviderResourceReleaser 类型为接受布尔值参数的函数对象
using ContentProviderResourceReleaser = std::function<void(bool success)>;

// 定义 MultipartFormDataProvider 结构体，包含四个成员变量
struct MultipartFormDataProvider {
  std::string name;
  ContentProviderWithoutLength provider;
  std::string filename;
  std::string content_type;
};
// 使用 std::vector 定义 MultipartFormDataProviderItems 类型
using MultipartFormDataProviderItems = std::vector<MultipartFormDataProvider>;

// 定义 ContentReceiverWithProgress 类型
using ContentReceiverWithProgress =
    // 定义一个名为std::function的函数对象，该函数对象接受四个参数：const char *data, size_t data_length, uint64_t offset, uint64_t total_length，并返回一个bool类型的值
// 定义一个类型别名 ContentReceiver，表示接收数据的回调函数
using ContentReceiver =
    std::function<bool(const char *data, size_t data_length)>;

// 定义一个类型别名 MultipartContentHeader，表示处理多部分内容头部的回调函数
using MultipartContentHeader =
    std::function<bool(const MultipartFormData &file)>;

// 定义一个类 ContentReader
class ContentReader {
public:
  // 定义类型别名 Reader，表示读取内容的回调函数
  using Reader = std::function<bool(ContentReceiver receiver)>;
  // 定义类型别名 MultipartReader，表示读取多部分内容的回调函数
  using MultipartReader = std::function<bool(MultipartContentHeader header,
                                             ContentReceiver receiver)>;

  // ContentReader 类的构造函数，接受 Reader 和 MultipartReader 作为参数
  ContentReader(Reader reader, MultipartReader multipart_reader)
      : reader_(std::move(reader)),
        multipart_reader_(std::move(multipart_reader)) {}

  // 重载 () 运算符，用于处理多部分内容
  bool operator()(MultipartContentHeader header,
                  ContentReceiver receiver) const {
    return multipart_reader_(std::move(header), std::move(receiver));
  }

  // 重载 () 运算符，用于处理内容
  bool operator()(ContentReceiver receiver) const {
    return reader_(std::move(receiver));
  }

  // 成员变量，保存 Reader 回调函数
  Reader reader_;
  // 成员变量，保存 MultipartReader 回调函数
  MultipartReader multipart_reader_;
};

// 定义类型别名 Range，表示范围的起始和结束位置
using Range = std::pair<ssize_t, ssize_t>;
// 定义类型别名 Ranges，表示多个范围
using Ranges = std::vector<Range>;

// 定义结构体 Request，表示请求信息
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
  // 远程端口
  int remote_port = -1;
  // 本地地址
  std::string local_addr;
  // 本地端口
  int local_port = -1;

  // 服务器端使用的字段
  // HTTP 版本
  std::string version;
  // 请求目标
  std::string target;
  // 请求参数
  Params params;
  // 多部分表单数据
  MultipartFormDataMap files;
  // 请求范围
  Ranges ranges;
  // 匹配结果
  Match matches;
  // 路径参数
  std::unordered_map<std::string, std::string> path_params;

  // 客户端端使用的字段
  // 响应处理器
  ResponseHandler response_handler;
  // 带进度的内容接收器
  ContentReceiverWithProgress content_receiver;
  // 进度
  Progress progress;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // SSL 连接
  const SSL *ssl = nullptr;
// 检查是否存在指定键的请求头
bool has_header(const std::string &key) const;

// 获取指定键的请求头值
std::string get_header_value(const std::string &key, size_t id = 0) const;

// 获取指定键的请求头值并转换为 uint64_t 类型
uint64_t get_header_value_u64(const std::string &key, size_t id = 0) const;

// 获取指定键的请求头值的数量
size_t get_header_value_count(const std::string &key) const;

// 设置请求头的键值对
void set_header(const std::string &key, const std::string &val);

// 检查是否存在指定键的参数
bool has_param(const std::string &key) const;

// 获取指定键的参数值
std::string get_param_value(const std::string &key, size_t id = 0) const;

// 获取指定键的参数值的数量
size_t get_param_value_count(const std::string &key) const;

// 检查是否为多部分表单数据
bool is_multipart_form_data() const;

// 检查是否存在指定键的文件
bool has_file(const std::string &key) const;

// 获取指定键的文件值
MultipartFormData get_file_value(const std::string &key) const;

// 获取指定键的文件值的向量
std::vector<MultipartFormData> get_file_values(const std::string &key) const;

// 私有成员...
size_t redirect_count_ = CPPHTTPLIB_REDIRECT_MAX_COUNT;
size_t content_length_ = 0;
ContentProvider content_provider_;
bool is_chunked_content_provider_ = false;
size_t authorization_count_ = 0;
// 定义 Response 结构体，包含版本、状态、原因、头部、正文、重定向地址等信息
struct Response {
  std::string version; // HTTP 版本号
  int status = -1; // HTTP 状态码，默认为 -1
  std::string reason; // HTTP 状态原因
  Headers headers; // HTTP 头部信息
  std::string body; // HTTP 响应正文
  std::string location; // 重定向地址

  // 检查是否存在指定键的头部信息
  bool has_header(const std::string &key) const;
  // 获取指定键的头部值
  std::string get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定键的头部值并转换为 uint64_t 类型
  uint64_t get_header_value_u64(const std::string &key, size_t id = 0) const;
  // 获取指定键的头部值的数量
  size_t get_header_value_count(const std::string &key) const;
  // 设置指定键的头部信息
  void set_header(const std::string &key, const std::string &val);

  // 设置重定向地址和状态码
  void set_redirect(const std::string &url, int status = 302);
  // 设置响应正文内容和类型
  void set_content(const char *s, size_t n, const std::string &content_type);
  void set_content(const std::string &s, const std::string &content_type);

  // 设置响应正文内容提供者
  void set_content_provider(
      size_t length, const std::string &content_type, ContentProvider provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  void set_content_provider(
      const std::string &content_type, ContentProviderWithoutLength provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  void set_chunked_content_provider(
      const std::string &content_type, ContentProviderWithoutLength provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 默认构造函数和拷贝构造函数
  Response() = default;
  Response(const Response &) = default;
  Response &operator=(const Response &) = default;
  Response(Response &&) = default;
  Response &operator=(Response &&) = default;
  // 析构函数，释放资源
  ~Response() {
    if (content_provider_resource_releaser_) {
      content_provider_resource_releaser_(content_provider_success_);
    }
  }

  // 私有成员变量...
  size_t content_length_ = 0; // 响应正文长度
  ContentProvider content_provider_; // 响应正文内容提供者
  ContentProviderResourceReleaser content_provider_resource_releaser_; // 响应正文内容提供者资源释放器
  bool is_chunked_content_provider_ = false; // 是否为分块响应内容提供者
  bool content_provider_success_ = false; // 响应正文内容提供是否成功
};

// 定义 Stream 类
class Stream {
// 定义一个公共接口类 Stream，包含一些纯虚函数和模板函数
public:
  virtual ~Stream() = default;

  // 判断流是否可读
  virtual bool is_readable() const = 0;
  // 判断流是否可写
  virtual bool is_writable() const = 0;

  // 读取数据到指定缓冲区
  virtual ssize_t read(char *ptr, size_t size) = 0;
  // 将数据写入流
  virtual ssize_t write(const char *ptr, size_t size) = 0;
  // 获取远程 IP 地址和端口
  virtual void get_remote_ip_and_port(std::string &ip, int &port) const = 0;
  // 获取本地 IP 地址和端口
  virtual void get_local_ip_and_port(std::string &ip, int &port) const = 0;
  // 获取底层 socket 描述符
  virtual socket_t socket() const = 0;

  // 格式化写入数据
  template <typename... Args>
  ssize_t write_format(const char *fmt, const Args &...args);
  // 写入字符数组
  ssize_t write(const char *ptr);
  // 写入字符串
  ssize_t write(const std::string &s);
};

// 定义一个任务队列类 TaskQueue，包含一些纯虚函数
class TaskQueue {
public:
  TaskQueue() = default;
  virtual ~TaskQueue() = default;

  // 将任务函数加入队列
  virtual void enqueue(std::function<void()> fn) = 0;
  // 关闭任务队列
  virtual void shutdown() = 0;

  // 空闲时执行的函数
  virtual void on_idle() {}
};

// 定义一个线程池类 ThreadPool，继承自 TaskQueue
class ThreadPool : public TaskQueue {
public:
  // 构造函数，初始化线程池
  explicit ThreadPool(size_t n) : shutdown_(false) {
    // 创建 n 个工作线程
    while (n) {
      threads_.emplace_back(worker(*this));
      n--;
    }
  }

  // 禁用拷贝构造函数
  ThreadPool(const ThreadPool &) = delete;
  // 虚析构函数
  ~ThreadPool() override = default;

  // 将任务函数加入队列
  void enqueue(std::function<void()> fn) override {
    {
      std::unique_lock<std::mutex> lock(mutex_);
      jobs_.push_back(std::move(fn));
    }

    // 通知一个等待的线程
    cond_.notify_one();
  }

  // 关闭线程池
  void shutdown() override {
    // 停止所有工作线程
    {
      std::unique_lock<std::mutex> lock(mutex_);
      shutdown_ = true;
    }

    // 通知所有等待的线程
    cond_.notify_all();

    // 等待所有线程结束
    for (auto &t : threads_) {
      t.join();
    }
  }

private:
  // 定义一个内部结构体 worker，用于执行任务
  struct worker {
    explicit worker(ThreadPool &pool) : pool_(pool) {}
    // 定义 operator() 函数，用于线程池中的工作线程执行任务
    void operator()() {
      // 无限循环，直到线程池关闭
      for (;;) {
        // 定义函数对象 fn
        std::function<void()> fn;
        {
          // 创建互斥锁 lock，保护线程池的访问
          std::unique_lock<std::mutex> lock(pool_.mutex_);

          // 等待条件变量 cond_，直到有任务或线程池关闭
          pool_.cond_.wait(
              lock, [&] { return !pool_.jobs_.empty() || pool_.shutdown_; });

          // 如果线程池关闭且没有任务，则跳出循环
          if (pool_.shutdown_ && pool_.jobs_.empty()) { break; }

          // 获取队列中的第一个任务，并移除
          fn = std::move(pool_.jobs_.front());
          pool_.jobs_.pop_front();
        }

        // 断言任务 fn 不为空
        assert(true == static_cast<bool>(fn));
        // 执行任务 fn
        fn();
      }
    }

    // 声明 worker 结构体为友元，以便访问私有成员
    friend struct worker;

    // 线程池中的工作线程
    std::vector<std::thread> threads_;
    // 存储待执行的任务
    std::list<std::function<void()>> jobs_;

    // 标记线程池是否关闭
    bool shutdown_;

    // 条件变量，用于线程同步
    std::condition_variable cond_;
    // 互斥锁，保护线程池的访问
    std::mutex mutex_;
};

// 使用 Logger 函数类型定义日志记录器
using Logger = std::function<void(const Request &, const Response &)>;

// 使用 SocketOptions 函数类型定义套接字选项
using SocketOptions = std::function<void(socket_t sock)>;

// 默认的套接字选项函数
void default_socket_options(socket_t sock);

// 根据状态码返回状态消息
const char *status_message(int status);

// detail 命名空间
namespace detail {

// MatcherBase 类，用于匹配请求路径并填充匹配结果
class MatcherBase {
public:
  virtual ~MatcherBase() = default;

  // 匹配请求路径并填充匹配结果
  virtual bool match(Request &request) const = 0;
};

/**
 * PathParamsMatcher 类，用于捕获请求路径中的参数并存储在 Request::path_params 中
 *
 * 捕获名称是从 : 到 / 的模式的子字符串。
 * 模式的其余部分直接与请求路径匹配
 * 参数是从上一个匹配的静态模式片段的结束字符的下一个字符开始捕获，直到下一个 /
 *
 * 示例模式:
 * "/path/fragments/:capture/more/fragments/:second_capture"
 * 静态片段:
 * "/path/fragments/", "more/fragments/"
 *
 * 给定以下请求路径:
 * "/path/fragments/:1/more/fragments/:2"
 * 结果捕获将是
 * {{"capture", "1"}, {"second_capture", "2"}}
 */
class PathParamsMatcher : public MatcherBase {
public:
  // 构造函数，接受模式字符串
  PathParamsMatcher(const std::string &pattern);

  // 匹配请求路径并填充匹配结果
  bool match(Request &request) const override;

private:
  // 标记字符为 :
  static constexpr char marker = ':';
  // 将段分隔符视为路径参数捕获的结束
  // 不需要处理查询参数，因为它们在路径匹配之前被解析
  static constexpr char separator = '/';

  // 包含要与之匹配的静态路径片段，不包括路径参数后的 /
  // 片段由路径参数分隔
  std::vector<std::string> static_fragments_;
  // 存储要用作 Request::path_params 映射中键的路径参数名称
  std::vector<std::string> param_names_;
};
/**
 * 执行 std::regex_match 对请求路径进行匹配，并将结果存储在 Request::matches 中
 *
 * 注意，正则表达式匹配直接在整个请求上执行。
 * 这意味着通配符模式可能匹配多个路径段，包括 /：
 * "/begin/(.*)/end" 将匹配 "/begin/middle/end" 和 "/begin/1/2/end"。
 */
class RegexMatcher : public MatcherBase {
public:
  RegexMatcher(const std::string &pattern) : regex_(pattern) {}

  bool match(Request &request) const override;

private:
  std::regex regex_;
};

// 写入响应头部信息到流
ssize_t write_headers(Stream &strm, const Headers &headers);

} // namespace detail

class Server {
public:
  using Handler = std::function<void(const Request &, Response &)>;

  using ExceptionHandler =
      std::function<void(const Request &, Response &, std::exception_ptr ep)>;

  enum class HandlerResponse {
    Handled,
protected:
  // 处理请求，设置请求是否关闭连接
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
  // 写入超时时间（秒）
  time_t write_timeout_sec_ = CPPHTTPLIB_WRITE_TIMEOUT_SECOND;
  // 写入超时时间（微秒）
  time_t write_timeout_usec_ = CPPHTTPLIB_WRITE_TIMEOUT_USECOND;
  // 空闲间隔时间（秒）
  time_t idle_interval_sec_ = CPPHTTPLIB_IDLE_INTERVAL_SECOND;
  // 空闲间隔时间（微秒）
  time_t idle_interval_usec_ = CPPHTTPLIB_IDLE_INTERVAL_USECOND;
  // 负载最大长度
  size_t payload_max_length_ = CPPHTTPLIB_PAYLOAD_MAX_LENGTH;

    // 挂载点
    std::string mount_point;
    // 基础目录
    std::string base_dir;
    // 定义一个 Headers 对象
    Headers headers;
  };
  // 存储基本目录的向量
  std::vector<MountPointEntry> base_dirs_;
  // 存储文件扩展名和 MIME 类型的映射关系
  std::map<std::string, std::string> file_extension_and_mimetype_map_;
  // 默认文件的 MIME 类型为 "application/octet-stream"
  std::string default_file_mimetype_ = "application/octet-stream";
  // 处理文件请求的处理程序
  Handler file_request_handler_;

  // GET 请求处理程序的集合
  Handlers get_handlers_;
  // POST 请求处理程序的集合
  Handlers post_handlers_;
  // POST 请求内容读取处理程序的集合
  HandlersForContentReader post_handlers_for_content_reader_;
  // PUT 请求处理程序的集合
  Handlers put_handlers_;
  // PUT 请求内容读取处理程序的集合
  HandlersForContentReader put_handlers_for_content_reader_;
  // PATCH 请求处理程序的集合
  Handlers patch_handlers_;
  // PATCH 请求内容读取处理程序的集合
  HandlersForContentReader patch_handlers_for_content_reader_;
  // DELETE 请求处理程序的集合
  Handlers delete_handlers_;
  // DELETE 请求内容读取处理程序的集合
  HandlersForContentReader delete_handlers_for_content_reader_;
  // OPTIONS 请求处理程序的集合
  Handlers options_handlers_;

  // 错误处理程序
  HandlerWithResponse error_handler_;
  // 异常处理程序
  ExceptionHandler exception_handler_;
  // 路由前处理程序
  HandlerWithResponse pre_routing_handler_;
  // 路由后处理程序
  Handler post_routing_handler_;
  // 100 Continue 期望处理程序
  Expect100ContinueHandler expect_100_continue_handler_;

  // 日志记录器
  Logger logger_;

  // 地址族，默认为 AF_UNSPEC
  int address_family_ = AF_UNSPEC;
  // 是否启用 TCP_NODELAY，默认为 CPPHTTPLIB_TCP_NODELAY
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  // 套接字选项，默认为 default_socket_options
  SocketOptions socket_options_ = default_socket_options;

  // 默认的 Headers 对象
  Headers default_headers_;
  // 头部写入函数，默认为 detail::write_headers
  std::function<ssize_t(Stream &, Headers &)> header_writer_ =
      detail::write_headers;
// 枚举类型定义不同的错误类型
enum class Error {
  Success = 0,  // 成功
  Unknown,  // 未知错误
  Connection,  // 连接错误
  BindIPAddress,  // 绑定 IP 地址错误
  Read,  // 读取错误
  Write,  // 写入错误
  ExceedRedirectCount,  // 超过重定向次数
  Canceled,  // 取消操作
  SSLConnection,  // SSL 连接错误
  SSLLoadingCerts,  // SSL 加载证书错误
  SSLServerVerification,  // SSL 服务器验证错误
  UnsupportedMultipartBoundaryChars,  // 不支持的多部分边界字符
  Compression,  // 压缩错误
  ConnectionTimeout,  // 连接超时
  ProxyConnection,  // 代理连接错误

  // 仅供内部使用
  SSLPeerCouldBeClosed_,  // SSL 对等方可能已关闭
};

// 将 Error 枚举类型转换为字符串
std::string to_string(const Error error);

// 重载输出流操作符，用于输出 Error 类型对象
std::ostream &operator<<(std::ostream &os, const Error &obj);

// 定义 Result 类
class Result {
public:
  Result() = default;
  // 构造函数，接受 Response 指针、Error 类型和请求头参数
  Result(std::unique_ptr<Response> &&res, Error err,
         Headers &&request_headers = Headers{})
      : res_(std::move(res)), err_(err),
        request_headers_(std::move(request_headers)) {}
  // Response
  // 转换为 bool 类型，判断是否存在 Response 对象
  operator bool() const { return res_ != nullptr; }
  // 判断是否等于 nullptr
  bool operator==(std::nullptr_t) const { return res_ == nullptr; }
  // 判断是否不等于 nullptr
  bool operator!=(std::nullptr_t) const { return res_ != nullptr; }
  // 返回 Response 对象的引用
  const Response &value() const { return *res_; }
  Response &value() { return *res_; }
  const Response &operator*() const { return *res_; }
  Response &operator*() { return *res_; }
  const Response *operator->() const { return res_.get(); }
  Response *operator->() { return res_.get(); }

  // Error
  // 返回 Error 类型对象
  Error error() const { return err_; }

  // Request Headers
  // 检查是否存在指定键的请求头
  bool has_request_header(const std::string &key) const;
  // 获取指定键的请求头值
  std::string get_request_header_value(const std::string &key,
                                       size_t id = 0) const;
  // 获取指定键的请求头值并转换为 uint64_t 类型
  uint64_t get_request_header_value_u64(const std::string &key,
                                        size_t id = 0) const;
  // 获取指定键的请求头值数量
  size_t get_request_header_value_count(const std::string &key) const;

private:
  std::unique_ptr<Response> res_;
  Error err_ = Error::Unknown;
  Headers request_headers_;
};

// 定义 ClientImpl 类
class ClientImpl {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要认证的用户名和密码
  void set_digest_auth(const std::string &username,
                       const std::string &password);
#endif
#endif 指令，用于结束一个条件编译块

  void set_keep_alive(bool on);
  声明一个函数，用于设置是否保持连接活跃

  void set_follow_location(bool on);
  声明一个函数，用于设置是否跟随重定向

  void set_url_encode(bool on);
  声明一个函数，用于设置是否进行 URL 编码

  void set_compress(bool on);
  声明一个函数，用于设置是否启用压缩

  void set_decompress(bool on);
  声明一个函数，用于设置是否启用解压缩

  void set_interface(const std::string &intf);
  声明一个函数，用于设置网络接口

  void set_proxy(const std::string &host, int port);
  声明一个函数，用于设置代理主机和端口

  void set_proxy_basic_auth(const std::string &username,
                            const std::string &password);
  声明一个函数，用于设置代理的基本认证信息

  void set_proxy_bearer_token_auth(const std::string &token);
  声明一个函数，用于设置代理的 Bearer Token 认证信息

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void set_proxy_digest_auth(const std::string &username,
                             const std::string &password);
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  声明一个函数，用于设置代理的摘要认证信息
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void set_ca_cert_path(const std::string &ca_cert_file_path,
                        const std::string &ca_cert_dir_path = std::string());
  声明一个函数，用于设置 CA 证书的文件路径和目录路径
  void set_ca_cert_store(X509_STORE *ca_cert_store);
  声明一个函数，用于设置 CA 证书存储
  X509_STORE *create_ca_cert_store(const char *ca_cert, std::size_t size);
  声明一个函数，用于创建 CA 证书存储
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void enable_server_certificate_verification(bool enabled);
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  声明一个函数，用于启用/禁用服务器证书验证
#endif

  void set_logger(Logger logger);
  声明一个函数，用于设置日志记录器

protected:
  struct Socket {
    socket_t sock = INVALID_SOCKET;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    SSL *ssl = nullptr;
#endif
  std::string digest_auth_username_;
  std::string digest_auth_password_;
  bool keep_alive_ = false;
  bool follow_location_ = false;
  bool url_encode_ = true;
  int address_family_ = AF_UNSPEC;
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  SocketOptions socket_options_ = nullptr;
  bool compress_ = false;
  bool decompress_ = true;
  std::string interface_;
  std::string proxy_host_;
  int proxy_port_ = -1;
  std::string proxy_basic_auth_username_;
  std::string proxy_basic_auth_password_;
  std::string proxy_bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  std::string proxy_digest_auth_username_;
  std::string proxy_digest_auth_password_;
#endif
  在 Socket 结构体中定义了一系列成员变量，用于保存与网络通信相关的信息
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则定义 CA 证书文件路径和目录路径
  std::string ca_cert_file_path_;
  std::string ca_cert_dir_path_;

  // 如果支持 OpenSSL，则定义 CA 证书存储对象
  X509_STORE *ca_cert_store_ = nullptr;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则定义是否进行服务器证书验证
  bool server_certificate_verification_ = true;
#endif

  // 定义日志记录器对象
  Logger logger_;

private:
  // 发送请求的私有方法，返回是否发送成功
  bool send_(Request &req, Response &res, Error &error);
  // 发送请求的私有方法，返回发送结果
  Result send_(Request &&req);

  // 创建客户端套接字
  socket_t create_client_socket(Error &error) const;
  // 读取响应行
  bool read_response_line(Stream &strm, const Request &req, Response &res);
  // 写入请求
  bool write_request(Stream &strm, Request &req, bool close_connection,
                     Error &error);
  // 重定向请求
  bool redirect(Request &req, Response &res, Error &error);
  // 处理请求
  bool handle_request(Stream &strm, Request &req, Response &res,
                      bool close_connection, Error &error);
  // 使用内容提供者发送请求
  std::unique_ptr<Response> send_with_content_provider(
      Request &req, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type, Error &error);
  // 使用内容提供者发送请求，返回发送结果
  Result send_with_content_provider(
      const std::string &method, const std::string &path,
      const Headers &headers, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type);
  // 获取多部分内容提供者
  ContentProviderWithoutLength get_multipart_content_provider(
      const std::string &boundary, const MultipartFormDataItems &items,
      const MultipartFormDataProviderItems &provider_items);

  // 调整主机字符串
  std::string adjust_host_string(const std::string &host) const;

  // 处理套接字的虚拟方法
  virtual bool process_socket(const Socket &socket,
                              std::function<bool(Stream &strm)> callback);
  // 判断是否为 SSL 连接
  virtual bool is_ssl() const;
};

class Client {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要认证
  void set_digest_auth(const std::string &username,
                       const std::string &password);
#endif
// 结束预处理指令

  void set_keep_alive(bool on);
  // 设置是否保持连接

  void set_follow_location(bool on);
  // 设置是否跟随重定向

  void set_url_encode(bool on);
  // 设置是否进行 URL 编码

  void set_compress(bool on);
  // 设置是否压缩请求数据

  void set_decompress(bool on);
  // 设置是否解压缩响应数据

  void set_interface(const std::string &intf);
  // 设置网络接口

  void set_proxy(const std::string &host, int port);
  // 设置代理服务器地址和端口

  void set_proxy_basic_auth(const std::string &username,
                            const std::string &password);
  // 设置代理服务器基本认证信息

  void set_proxy_bearer_token_auth(const std::string &token);
  // 设置代理服务器 Bearer Token 认证信息
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void set_proxy_digest_auth(const std::string &username,
                             const std::string &password);
#endif
// 设置代理服务器摘要认证信息

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void enable_server_certificate_verification(bool enabled);
#endif
// 启用/禁用服务器证书验证

  void set_logger(Logger logger);
  // 设置日志记录器

  // SSL
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  void set_ca_cert_path(const std::string &ca_cert_file_path,
                        const std::string &ca_cert_dir_path = std::string());
  // 设置 CA 证书路径

  void set_ca_cert_store(X509_STORE *ca_cert_store);
  // 设置 CA 证书存储

  void load_ca_cert_store(const char *ca_cert, std::size_t size);
  // 加载 CA 证书存储

  long get_openssl_verify_result() const;
  // 获取 OpenSSL 验证结果

  SSL_CTX *ssl_context() const;
#endif
// 获取 SSL 上下文

private:
  std::unique_ptr<ClientImpl> cli_;
// 客户端实现的唯一指针

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  bool is_ssl_ = false;
#endif
// 是否启用 SSL

};

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
class SSLServer : public Server {
public:
  SSLServer(const char *cert_path, const char *private_key_path,
            const char *client_ca_cert_file_path = nullptr,
            const char *client_ca_cert_dir_path = nullptr,
            const char *private_key_password = nullptr);
  // SSL 服务器构造函数

  SSLServer(X509 *cert, EVP_PKEY *private_key,
            X509_STORE *client_ca_cert_store = nullptr);
  // SSL 服务器构造函数

  SSLServer(
      const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback);
  // SSL 服务器构造函数

  ~SSLServer() override;
  // SSL 服务器析构函数

  bool is_valid() const override;
  // 检查 SSL 服务器是否有效

  SSL_CTX *ssl_context() const;
  // 获取 SSL 上下文

private:
  bool process_and_close_socket(socket_t sock) override;
  // 处理并关闭套接字

  SSL_CTX *ctx_;
  // SSL 上下文
  std::mutex ctx_mutex_;
  // 上下文互斥锁
};
// SSLClient 类继承自 ClientImpl 类，用于处理 SSL 客户端连接
class SSLClient : public ClientImpl {
public:
  // 构造函数，根据主机名创建 SSL 客户端对象
  explicit SSLClient(const std::string &host);

  // 构造函数，根据主机名和端口号创建 SSL 客户端对象
  explicit SSLClient(const std::string &host, int port);

  // 构造函数，根据主机名、端口号、客户端证书路径和客户端密钥路径创建 SSL 客户端对象
  explicit SSLClient(const std::string &host, int port,
                     const std::string &client_cert_path,
                     const std::string &client_key_path);

  // 构造函数，根据主机名、端口号、客户端证书和客户端密钥创建 SSL 客户端对象
  explicit SSLClient(const std::string &host, int port, X509 *client_cert,
                     EVP_PKEY *client_key);

  // 析构函数，释放 SSL 客户端对象
  ~SSLClient() override;

  // 检查 SSL 客户端对象是否有效
  bool is_valid() const override;

  // 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);
  // 加载 CA 证书存储
  void load_ca_cert_store(const char *ca_cert, std::size_t size);

  // 获取 OpenSSL 验证结果
  long get_openssl_verify_result() const;

  // 获取 SSL 上下文
  SSL_CTX *ssl_context() const;

private:
  // 创建并连接套接字
  bool create_and_connect_socket(Socket &socket, Error &error) override;
  // 关闭 SSL 连接
  void shutdown_ssl(Socket &socket, bool shutdown_gracefully) override;
  // 实现 SSL 关闭
  void shutdown_ssl_impl(Socket &socket, bool shutdown_socket);

  // 处理套接字
  bool process_socket(const Socket &socket,
                      std::function<bool(Stream &strm)> callback) override;
  // 检查是否为 SSL 连接
  bool is_ssl() const override;

  // 使用代理连接
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
  // 使用通用名称验证主机名
  bool verify_host_with_common_name(X509 *server_cert) const;
  // 检查主机名
  bool check_host_name(const char *pattern, size_t pattern_len) const;

  // SSL 上下文
  SSL_CTX *ctx_;
  // 上下文互斥锁
  std::mutex ctx_mutex_;
  // 证书初始化标志
  std::once_flag initialize_cert_;

  // 主机名组件
  std::vector<std::string> host_components_;

  // 验证结果
  long verify_result_ = 0;

  // 声明 ClientImpl 类为友元类
  friend class ClientImpl;
};
#endif

/*
 * Implementation of template methods.
 */

// 实现模板方法的命名空间
namespace detail {

// 模板方法的实现
template <typename T, typename U>
// 将持续时间转换为秒和微秒，并通过回调函数返回结果
inline void duration_to_sec_and_usec(const T &duration, U callback) {
  // 将持续时间转换为秒
  auto sec = std::chrono::duration_cast<std::chrono::seconds>(duration).count();
  // 计算剩余的微秒
  auto usec = std::chrono::duration_cast<std::chrono::microseconds>(
                  duration - std::chrono::seconds(sec))
                  .count();
  // 调用回调函数返回秒和微秒
  callback(static_cast<time_t>(sec), static_cast<time_t>(usec));
}

// 获取头部中指定键的第 id 个值作为 uint64_t 类型
inline uint64_t get_header_value_u64(const Headers &headers,
                                     const std::string &key, size_t id,
                                     uint64_t def) {
  // 查找指定键的值范围
  auto rng = headers.equal_range(key);
  auto it = rng.first;
  // 移动到指定 id 的位置
  std::advance(it, static_cast<ssize_t>(id));
  // 如果找到对应值，则将其转换为 uint64_t 类型返回
  if (it != rng.second) {
    return std::strtoull(it->second.data(), nullptr, 10);
  }
  // 如果未找到对应值，则返回默认值
  return def;
}

} // namespace detail

// 获取请求头部中指定键的第 id 个值作为 uint64_t 类型
inline uint64_t Request::get_header_value_u64(const std::string &key,
                                              size_t id) const {
  return detail::get_header_value_u64(headers, key, id, 0);
}

// 获取响应头部中指定键的第 id 个值作为 uint64_t 类型
inline uint64_t Response::get_header_value_u64(const std::string &key,
                                               size_t id) const {
  return detail::get_header_value_u64(headers, key, id, 0);
}

// 格式化输出数据到流
template <typename... Args>
inline ssize_t Stream::write_format(const char *fmt, const Args &...args) {
  const auto bufsiz = 2048;
  std::array<char, bufsiz> buf{};

  // 格式化数据到缓冲区
  auto sn = snprintf(buf.data(), buf.size() - 1, fmt, args...);
  // 如果格式化失败，则返回错误码
  if (sn <= 0) { return sn; }

  auto n = static_cast<size_t>(sn);

  // 如果数据超过缓冲区大小，则动态扩展缓冲区并继续写入
  if (n >= buf.size() - 1) {
    std::vector<char> glowable_buf(buf.size());

    while (n >= glowable_buf.size() - 1) {
      glowable_buf.resize(glowable_buf.size() * 2);
      n = static_cast<size_t>(
          snprintf(&glowable_buf[0], glowable_buf.size() - 1, fmt, args...));
    }
    return write(&glowable_buf[0], n);
  } else {
    return write(buf.data(), n);
  }
}

// 设置默认的套接字选项
inline void default_socket_options(socket_t sock) {
  int yes = 1;
#ifdef _WIN32
  // 如果是在 Windows 平台下，设置套接字选项为 SO_REUSEADDR，允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR,
             reinterpret_cast<const char *>(&yes), sizeof(yes));
  // 如果是在 Windows 平台下，设置套接字选项为 SO_EXCLUSIVEADDRUSE，独占地址使用
  setsockopt(sock, SOL_SOCKET, SO_EXCLUSIVEADDRUSE,
             reinterpret_cast<const char *>(&yes), sizeof(yes));
#else
#ifdef SO_REUSEPORT
  // 如果支持 SO_REUSEPORT 选项，设置套接字选项为 SO_REUSEPORT，允许端口重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEPORT,
             reinterpret_cast<const void *>(&yes), sizeof(yes));
#else
  // 如果不支持 SO_REUSEPORT 选项，设置套接字选项为 SO_REUSEADDR，允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR,
             reinterpret_cast<const void *>(&yes), sizeof(yes));
#endif
#endif
}

}

template <class Rep, class Period>
inline Server &
Server::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 将传入的时间段转换为秒和微秒，并调用 set_read_timeout 方法设置读取超时时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
  return *this;
}

template <class Rep, class Period>
inline Server &
Server::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 将传入的时间段转换为秒和微秒，并调用 set_write_timeout 方法设置写入超时时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_write_timeout(sec, usec); });
  return *this;
}

template <class Rep, class Period>
inline Server &
Server::set_idle_interval(const std::chrono::duration<Rep, Period> &duration) {
  // 将传入的时间段转换为秒和微秒，并调用 set_idle_interval 方法设置空闲间隔时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_idle_interval(sec, usec); });
  return *this;
}
// 将错误枚举类型转换为字符串表示
inline std::string to_string(const Error error) {
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
  case Error::Compression: return "Compression failed";
  case Error::ConnectionTimeout: return "Connection timed out";
  case Error::ProxyConnection: return "Proxy connection failed";
  case Error::Unknown: return "Unknown";
  default: break;
  }

  return "Invalid";
}

// 重载输出流操作符，输出错误枚举类型的字符串表示和对应的整数值
inline std::ostream &operator<<(std::ostream &os, const Error &obj) {
  os << to_string(obj);
  os << " (" << static_cast<std::underlying_type<Error>::type>(obj) << ')';
  return os;
}

// 获取请求头中指定键的值，并将其转换为无符号64位整数
inline uint64_t Result::get_request_header_value_u64(const std::string &key,
                                                     size_t id) const {
  return detail::get_header_value_u64(request_headers_, key, id, 0);
}

// 设置连接超时时间，接受以秒和微秒为单位的持续时间
template <class Rep, class Period>
inline void ClientImpl::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用相应的设置连接超时函数
  detail::duration_to_sec_and_usec(duration, [&](time_t sec, time_t usec) {
    set_connection_timeout(sec, usec);
  });
}

// 设置读取超时时间
template <class Rep, class Period>
inline void ClientImpl::set_read_timeout(
    // 设置读取超时时间，将持续时间转换为秒和微秒，并调用 set_read_timeout 方法
    const std::chrono::duration<Rep, Period> &duration) {
  // 调用 detail 命名空间中的 duration_to_sec_and_usec 函数，将持续时间转换为秒和微秒
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
// 设置写入超时时间，将持续时间转换为秒和微秒，并调用相应的函数设置超时时间
template <class Rep, class Period>
inline void ClientImpl::set_write_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_write_timeout(sec, usec); });
}

// 设置连接超时时间
template <class Rep, class Period>
inline void Client::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  cli_->set_connection_timeout(duration);
}

// 设置读取超时时间
template <class Rep, class Period>
inline void
Client::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  cli_->set_read_timeout(duration);
}

// 设置写入超时时间
template <class Rep, class Period>
inline void
Client::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  cli_->set_write_timeout(duration);
}

/*
 * Forward declarations and types that will be part of the .h file if split into
 * .h + .cc.
 */

// 返回主机名所在的位置
std::string hosted_at(const std::string &hostname);

// 返回主机名所在的位置，并将结果存储在向量中
void hosted_at(const std::string &hostname, std::vector<std::string> &addrs);

// 在路径后附加查询参数
std::string append_query_params(const std::string &path, const Params &params);

// 生成范围头部
std::pair<std::string, std::string> make_range_header(Ranges ranges);

// 生成基本身份验证头部
std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password,
                                 bool is_proxy = false);

namespace detail {

// 编码查询参数
std::string encode_query_param(const std::string &value);

// 解码 URL
std::string decode_url(const std::string &s, bool convert_plus_to_space);

// 读取文件内容
void read_file(const std::string &path, std::string &out);

// 复制并修剪字符串
std::string trim_copy(const std::string &s);

// 分割字符串
void split(const char *b, const char *e, char d,
           std::function<void(const char *, const char *)> fn);
# 处理客户端套接字，设置读写超时时间，并调用回调函数
bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                           time_t read_timeout_usec, time_t write_timeout_sec,
                           time_t write_timeout_usec,
                           std::function<bool(Stream &)> callback);

# 创建客户端套接字，连接到指定主机和端口，设置各种参数和超时时间
socket_t create_client_socket(
    const std::string &host, const std::string &ip, int port,
    int address_family, bool tcp_nodelay, SocketOptions socket_options,
    time_t connection_timeout_sec, time_t connection_timeout_usec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    time_t write_timeout_usec, const std::string &intf, Error &error);

# 获取指定键的头部值
const char *get_header_value(const Headers &headers, const std::string &key,
                             size_t id = 0, const char *def = nullptr);

# 将参数转换为查询字符串
std::string params_to_query_str(const Params &params);

# 解析查询文本，将结果存储在参数中
void parse_query_text(const std::string &s, Params &params);

# 解析多部分边界
bool parse_multipart_boundary(const std::string &content_type,
                              std::string &boundary);

# 解析范围头部
bool parse_range_header(const std::string &s, Ranges &ranges);

# 关闭套接字
int close_socket(socket_t sock);

# 发送数据到套接字
ssize_t send_socket(socket_t sock, const void *ptr, size_t size, int flags);

# 从套接字读取数据
ssize_t read_socket(socket_t sock, void *ptr, size_t size, int flags);

# 编码类型枚举
enum class EncodingType { None = 0, Gzip, Brotli };

# 获取请求和响应的编码类型
EncodingType encoding_type(const Request &req, const Response &res);

# 缓冲流类，继承自流类
class BufferStream : public Stream {
public:
  BufferStream() = default;
  ~BufferStream() override = default;

  # 判断是否可读
  bool is_readable() const override;
  # 判断是否可写
  bool is_writable() const override;
  # 读取数据
  ssize_t read(char *ptr, size_t size) override;
  # 写入数据
  ssize_t write(const char *ptr, size_t size) override;
  # 获取远程 IP 和端口
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  # 获取本地 IP 和端口
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  # 获取套接字
  socket_t socket() const override;

  # 获取缓冲区内容
  const std::string &get_buffer() const;

private:
  std::string buffer;
  size_t position = 0;
};
// 定义一个压缩器基类，包含压缩函数的纯虚函数接口
class compressor {
public:
  virtual ~compressor() = default;

  // 定义回调函数类型，用于在压缩过程中传递数据
  typedef std::function<bool(const char *data, size_t data_len)> Callback;
  // 压缩函数，接受数据、数据长度、是否为最后一块数据和回调函数
  virtual bool compress(const char *data, size_t data_length, bool last,
                        Callback callback) = 0;
};

// 定义一个解压缩器基类，包含解压函数的纯虚函数接口
class decompressor {
public:
  virtual ~decompressor() = default;

  // 判断解压器是否有效的函数
  virtual bool is_valid() const = 0;

  // 定义回调函数类型，用于在解压过程中传递数据
  typedef std::function<bool(const char *data, size_t data_len)> Callback;
  // 解压函数，接受数据、数据长度和回调函数
  virtual bool decompress(const char *data, size_t data_length,
                          Callback callback) = 0;
};

// 定义一个不压缩的压缩器类，继承自压缩器基类
class nocompressor : public compressor {
public:
  virtual ~nocompressor() = default;

  // 不进行压缩，直接传递数据的压缩函数
  bool compress(const char *data, size_t data_length, bool /*last*/,
                Callback callback) override;
};

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 如果支持 ZLIB，则定义 gzip 压缩器类，继承自压缩器基类
class gzip_compressor : public compressor {
public:
  gzip_compressor();
  ~gzip_compressor();

  // 对数据进行 gzip 压缩的函数
  bool compress(const char *data, size_t data_length, bool last,
                Callback callback) override;

private:
  bool is_valid_ = false;
  z_stream strm_;
};

// 如果支持 ZLIB，则定义 gzip 解压缩器类，继承自解压缩器基类
class gzip_decompressor : public decompressor {
public:
  gzip_decompressor();
  ~gzip_decompressor();

  // 判断解压器是否有效的函数
  bool is_valid() const override;

  // 对数据进行 gzip 解压缩的函数
  bool decompress(const char *data, size_t data_length,
                  Callback callback) override;

private:
  bool is_valid_ = false;
  z_stream strm_;
};
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
// 如果支持 Brotli，则定义 Brotli 压缩器类，继承自压缩器基类
class brotli_compressor : public compressor {
public:
  brotli_compressor();
  ~brotli_compressor();

  // 对数据进行 Brotli 压缩的函数
  bool compress(const char *data, size_t data_length, bool last,
                Callback callback) override;

private:
  BrotliEncoderState *state_ = nullptr;
};

// 如果支持 Brotli，则定义 Brotli 解压缩器类，继承自解压缩器基类
class brotli_decompressor : public decompressor {
public:
  brotli_decompressor();
  ~brotli_decompressor();

  // 判断解压器是否有效的函数
  bool is_valid() const override;

  // 对数据进行 Brotli 解压缩的函数
  bool decompress(const char *data, size_t data_length,
                  Callback callback) override;
// 定义私有成员变量，用于存储 Brotli 解码器的结果和状态
private:
  BrotliDecoderResult decoder_r;
  BrotliDecoderState *decoder_s = nullptr;
};
#endif

// 注意：在读取大小达到 `fixed_buffer_size` 之前，使用 `fixed_buffer` 存储数据。调用可以在堆栈上设置内存以提高性能。
// 定义 stream_line_reader 类，用于从流中读取一行数据
class stream_line_reader {
public:
  // 构造函数，初始化流、固定缓冲区和固定缓冲区大小
  stream_line_reader(Stream &strm, char *fixed_buffer,
                     size_t fixed_buffer_size);
  // 返回当前指针位置
  const char *ptr() const;
  // 返回当前数据大小
  size_t size() const;
  // 检查是否以 CRLF 结尾
  bool end_with_crlf() const;
  // 读取一行数据
  bool getline();

private:
  // 追加字符到缓冲区
  void append(char c);

  // 成员变量，存储流、固定缓冲区、固定缓冲区大小和已使用的固定缓冲区大小
  Stream &strm_;
  char *fixed_buffer_;
  const size_t fixed_buffer_size_;
  size_t fixed_buffer_used_size_ = 0;
  std::string glowable_buffer_;
};

// 定义 mmap 类，用于内存映射文件
class mmap {
public:
  // 构造函数，打开指定路径的文件
  mmap(const char *path);
  // 析构函数，关闭文件
  ~mmap();

  // 打开指定路径的文件
  bool open(const char *path);
  // 关闭文件
  void close();

  // 检查文件是否打开
  bool is_open() const;
  // 返回文件大小
  size_t size() const;
  // 返回文件数据指针
  const char *data() const;

private:
  // 成员变量，根据操作系统不同存储文件句柄和映射句柄
#if defined(_WIN32)
  HANDLE hFile_;
  HANDLE hMapping_;
#else
  int fd_;
#endif
  size_t size_;
  void *addr_;
};

} // namespace detail

// ----------------------------------------------------------------------------

/*
 * 如果拆分为 .h + .cc 文件，以下实现将成为 .cc 文件的一部分。
 */

namespace detail {

// 判断字符是否为十六进制字符，并将其转换为整数
inline bool is_hex(char c, int &v) {
  if (0x20 <= c && isdigit(c)) {
    v = c - '0';
    return true;
  } else if ('A' <= c && c <= 'F') {
    v = c - 'A' + 10;
    return true;
  } else if ('a' <= c && c <= 'f') {
    v = c - 'a' + 10;
    return true;
  }
  return false;
}

// 将十六进制字符串转换为整数
inline bool from_hex_to_i(const std::string &s, size_t i, size_t cnt,
                          int &val) {
  if (i >= s.size()) { return false; }

  val = 0;
  for (; cnt; i++, cnt--) {
    if (!s[i]) { return false; }
    auto v = 0;
    if (is_hex(s[i], v)) {
      val = val * 16 + v;
    } else {
      return false;
    }
  }
  return true;
}

// 将整数转换为十六进制字符串
inline std::string from_i_to_hex(size_t n) {
  static const auto charset = "0123456789abcdef";
  std::string ret;
  do {
    ret = charset[n & 15] + ret;
    # 将 n 右移 4 位，相当于 n 除以 16
    n >>= 4;
  # 循环直到 n 变为 0
  } while (n > 0);
  # 返回结果 ret
  return ret;
// 将 Unicode 编码转换为 UTF-8 编码，返回转换后的字节数
inline size_t to_utf8(int code, char *buff) {
  // 如果 Unicode 编码小于 0x0080，直接转换为一个字节的 UTF-8 编码
  if (code < 0x0080) {
    buff[0] = (code & 0x7F);
    return 1;
  } else if (code < 0x0800) {
    // 如果 Unicode 编码小于 0x0800，转换为两个字节的 UTF-8 编码
    buff[0] = static_cast<char>(0xC0 | ((code >> 6) & 0x1F));
    buff[1] = static_cast<char>(0x80 | (code & 0x3F));
    return 2;
  } else if (code < 0xD800) {
    // 如果 Unicode 编码小于 0xD800，转换为三个字节的 UTF-8 编码
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    return 3;
  } else if (code < 0xE000) { // D800 - DFFF is invalid...
    // D800 - DFFF 是无效的 Unicode 编码范围，返回 0
    return 0;
  } else if (code < 0x10000) {
    // 如果 Unicode 编码小于 0x10000，转换为三个字节的 UTF-8 编码
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    return 3;
  } else if (code < 0x110000) {
    // 如果 Unicode 编码小于 0x110000，转换为四个字节的 UTF-8 编码
    buff[0] = static_cast<char>(0xF0 | ((code >> 18) & 0x7));
    buff[1] = static_cast<char>(0x80 | ((code >> 12) & 0x3F));
    buff[2] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[3] = static_cast<char>(0x80 | (code & 0x3F));
    return 4;
  }

  // NOTREACHED
  return 0;
}

// 注意：此代码是根据以下 stackoverflow 帖子编写的：
// https://stackoverflow.com/questions/180947/base64-decode-snippet-in-c
// 将输入字符串进行 Base64 编码
inline std::string base64_encode(const std::string &in) {
  // Base64 编码表
  static const auto lookup =
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

  // 初始化输出字符串
  std::string out;
  out.reserve(in.size());

  auto val = 0;
  auto valb = -6;

  // 遍历输入字符串
  for (auto c : in) {
    val = (val << 8) + static_cast<uint8_t>(c);
    valb += 8;
    // 当累积的位数达到 6 位时，进行 Base64 编码
    while (valb >= 0) {
      out.push_back(lookup[(val >> valb) & 0x3F]);
      valb -= 6;
    }
  }

  // 处理最后不足 4 字节的情况，补充 '='
  if (valb > -6) { out.push_back(lookup[((val << 8) >> (valb + 8)) & 0x3F]); }

  // 补充 '=' 直到字符串长度为 4 的倍数
  while (out.size() % 4) {
    out.push_back('=');
  }

  return out;
}

// 检查给定路径是否为文件
inline bool is_file(const std::string &path) {
#ifdef _WIN32
  // 在 Windows 平台上检查文件是否存在
  return _access_s(path.c_str(), 0) == 0;
// 检查路径是否为文件
inline bool is_file(const std::string &path) {
  // 创建 stat 结构体
  struct stat st;
  // 检查路径是否存在并且是一个常规文件
  return stat(path.c_str(), &st) >= 0 && S_ISREG(st.st_mode);
}

// 检查路径是否为目录
inline bool is_dir(const std::string &path) {
  // 创建 stat 结构体
  struct stat st;
  // 检查路径是否存在并且是一个目录
  return stat(path.c_str(), &st) >= 0 && S_ISDIR(st.st_mode);
}

// 检查路径是否为有效路径
inline bool is_valid_path(const std::string &path) {
  size_t level = 0;
  size_t i = 0;

  // 跳过斜杠
  while (i < path.size() && path[i] == '/') {
    i++;
  }

  while (i < path.size()) {
    // 读取路径组件
    auto beg = i;
    while (i < path.size() && path[i] != '/') {
      i++;
    }

    auto len = i - beg;
    assert(len > 0);

    if (!path.compare(beg, len, ".")) {
      ;
    } else if (!path.compare(beg, len, "..")) {
      if (level == 0) { return false; }
      level--;
    } else {
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
  std::ostringstream escaped;
  escaped.fill('0');
  escaped << std::hex;

  for (auto c : value) {
    if (std::isalnum(static_cast<uint8_t>(c)) || c == '-' || c == '_' ||
        c == '.' || c == '!' || c == '~' || c == '*' || c == '\'' || c == '(' ||
        c == ')') {
      escaped << c;
    } else {
      escaped << std::uppercase;
      escaped << '%' << std::setw(2)
              << static_cast<int>(static_cast<unsigned char>(c));
      escaped << std::nouppercase;
    }
  }

  return escaped.str();
}

// 对 URL 进行编码
inline std::string encode_url(const std::string &s) {
  std::string result;
  result.reserve(s.size());

  for (size_t i = 0; s[i]; i++) {
    switch (s[i]) {
    case ' ': result += "%20"; break;
    case '+': result += "%2B"; break;
    case '\r': result += "%0D"; break;
    case '\n': result += "%0A"; break;
    case '\'': result += "%27"; break;
    case ',': result += "%2C"; break;
    // case ':': result += "%3A"; break; // ok? probably...
    case ';': result += "%3B"; break;
    // 对于默认情况，将字符转换为无符号8位整数
    default:
      auto c = static_cast<uint8_t>(s[i]);
      // 如果字符大于等于0x80，表示为非ASCII字符，需要进行URL编码
      if (c >= 0x80) {
        // 将结果字符串添加'%'
        result += '%';
        // 创建一个存储十六进制值的字符数组
        char hex[4];
        // 将字符转换为十六进制表示，并存储在hex中
        auto len = snprintf(hex, sizeof(hex) - 1, "%02X", c);
        // 确保转换成功
        assert(len == 2);
        // 将十六进制值添加到结果字符串中
        result.append(hex, static_cast<size_t>(len));
      } else {
        // 如果是ASCII字符，直接添加到结果字符串中
        result += s[i];
      }
      // 结束当前case
      break;
    }
  }

  // 返回URL编码后的结果字符串
  return result;
// 解码 URL 字符串，将 %xx 转换为对应字符，将 + 转换为空格
inline std::string decode_url(const std::string &s,
                              bool convert_plus_to_space) {
  // 存储解码后的结果
  std::string result;

  // 遍历输入字符串
  for (size_t i = 0; i < s.size(); i++) {
    // 如果当前字符是 '%'，并且后面还有字符
    if (s[i] == '%' && i + 1 < s.size()) {
      // 如果下一个字符是 'u'
      if (s[i + 1] == 'u') {
        auto val = 0;
        // 将 %u 后的 4 位十六进制数转换为整数
        if (from_hex_to_i(s, i + 2, 4, val)) {
          // 4 位 Unicode 编码
          char buff[4];
          // 将 Unicode 编码转换为 UTF-8 字符
          size_t len = to_utf8(val, buff);
          if (len > 0) { result.append(buff, len); }
          i += 5; // 'u0000'
        } else {
          result += s[i];
        }
      } else {
        auto val = 0;
        // 将 % 后的 2 位十六进制数转换为整数
        if (from_hex_to_i(s, i + 1, 2, val)) {
          // 2 位十六进制编码
          result += static_cast<char>(val);
          i += 2; // '00'
        } else {
          result += s[i];
        }
      }
    } else if (convert_plus_to_space && s[i] == '+') {
      // 如果需要将 + 转换为空格
      result += ' ';
    } else {
      result += s[i];
    }
  }

  return result;
}

// 读取文件内容到字符串
inline void read_file(const std::string &path, std::string &out) {
  // 打开文件流
  std::ifstream fs(path, std::ios_base::binary);
  // 定位到文件末尾，获取文件大小
  fs.seekg(0, std::ios_base::end);
  auto size = fs.tellg();
  // 定位到文件开头
  fs.seekg(0);
  // 调整输出字符串大小
  out.resize(static_cast<size_t>(size));
  // 读取文件内容到字符串
  fs.read(&out[0], static_cast<std::streamsize>(size));
}

// 获取文件扩展名
inline std::string file_extension(const std::string &path) {
  std::smatch m;
  // 正则表达式匹配文件扩展名
  static auto re = std::regex("\\.([a-zA-Z0-9]+)$");
  if (std::regex_search(path, m, re)) { return m[1].str(); }
  return std::string();
}

// 判断字符是否为空格或制表符
inline bool is_space_or_tab(char c) { return c == ' ' || c == '\t'; }

// 去除字符串两端的空格和制表符
inline std::pair<size_t, size_t> trim(const char *b, const char *e, size_t left,
                                      size_t right) {
  // 去除左侧空格和制表符
  while (b + left < e && is_space_or_tab(b[left])) {
    left++;
  }
  // 去除右侧空格和制表符
  while (right > 0 && is_space_or_tab(b[right - 1])) {
    right--;
  }
  return std::make_pair(left, right);
}
// 复制去除字符串两端空格的函数
inline std::string trim_copy(const std::string &s) {
  // 调用 trim 函数去除字符串两端空格，并返回去除空格后的子串
  auto r = trim(s.data(), s.data() + s.size(), 0, s.size());
  return s.substr(r.first, r.second - r.first);
}

// 复制去除字符串两端双引号的函数
inline std::string trim_double_quotes_copy(const std::string &s) {
  // 如果字符串长度大于等于2且首尾字符为双引号，则去除首尾双引号并返回
  if (s.length() >= 2 && s.front() == '"' && s.back() == '"') {
    return s.substr(1, s.size() - 2);
  }
  // 否则返回原字符串
  return s;
}

// 分割字符串函数
inline void split(const char *b, const char *e, char d,
                  std::function<void(const char *, const char *)> fn) {
  size_t i = 0;
  size_t beg = 0;

  // 遍历字符串，根据分隔符 d 进行分割
  while (e ? (b + i < e) : (b[i] != '\0')) {
    if (b[i] == d) {
      // 调用 trim 函数去除子串两端空格，并调用回调函数 fn 处理分割后的子串
      auto r = trim(b, e, beg, i);
      if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
      beg = i + 1;
    }
    i++;
  }

  // 处理最后一个子串
  if (i) {
    auto r = trim(b, e, beg, i);
    if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
  }
}

// 流式读取器构造函数
inline stream_line_reader::stream_line_reader(Stream &strm, char *fixed_buffer,
                                              size_t fixed_buffer_size)
    : strm_(strm), fixed_buffer_(fixed_buffer),
      fixed_buffer_size_(fixed_buffer_size) {}

// 获取当前指针位置
inline const char *stream_line_reader::ptr() const {
  // 如果动态缓冲区为空，则返回固定缓冲区指针，否则返回动态缓冲区指针
  if (glowable_buffer_.empty()) {
    return fixed_buffer_;
  } else {
    return glowable_buffer_.data();
  }
}

// 获取当前大小
inline size_t stream_line_reader::size() const {
  // 如果动态缓冲区为空，则返回固定缓冲区已使用大小，否则返回动态缓冲区大小
  if (glowable_buffer_.empty()) {
    return fixed_buffer_used_size_;
  } else {
    return glowable_buffer_.size();
  }
}

// 判断是否以 CRLF 结尾
inline bool stream_line_reader::end_with_crlf() const {
  auto end = ptr() + size();
  // 判断大小是否大于等于2且末尾两个字符为 CRLF
  return size() >= 2 && end[-2] == '\r' && end[-1] == '\n';
}

// 读取一行数据
inline bool stream_line_reader::getline() {
  fixed_buffer_used_size_ = 0;
  glowable_buffer_.clear();

  for (size_t i = 0;; i++) {
    char byte;
    auto n = strm_.read(&byte, 1);

    // 读取失败返回 false
    if (n < 0) {
      return false;
    } else if (n == 0) {
      // 如果未读取到数据，且 i 为 0，则返回 false；否则跳出循环
      if (i == 0) {
        return false;
      } else {
        break;
      }
    }

    // 将读取的字节追加到缓冲区
    append(byte);

    // 如果读取到换行符，则跳出循环
    if (byte == '\n') { break; }
  }

  return true;
}
// 在流行的行阅读器中追加字符
inline void stream_line_reader::append(char c) {
  // 如果固定缓冲区使用的大小小于固定缓冲区大小减1
  if (fixed_buffer_used_size_ < fixed_buffer_size_ - 1) {
    // 将字符添加到固定缓冲区中，并在末尾添加空字符
    fixed_buffer_[fixed_buffer_used_size_++] = c;
    fixed_buffer_[fixed_buffer_used_size_] = '\0';
  } else {
    // 如果可增长缓冲区为空
    if (glowable_buffer_.empty()) {
      // 确保固定缓冲区的末尾是空字符
      assert(fixed_buffer_[fixed_buffer_used_size_] == '\0');
      // 将固定缓冲区的内容复制到可增长缓冲区
      glowable_buffer_.assign(fixed_buffer_, fixed_buffer_used_size_);
    }
    // 在可增长缓冲区中追加字符
    glowable_buffer_ += c;
  }
}

// 构造函数，根据不同操作系统初始化成员变量
inline mmap::mmap(const char *path)
#if defined(_WIN32)
    : hFile_(NULL), hMapping_(NULL)
#else
    : fd_(-1)
#endif
      ,
      size_(0), addr_(nullptr) {
  // 如果无法打开文件，则抛出运行时错误
  if (!open(path)) { std::runtime_error(""); }
}

// 析构函数，关闭映射
inline mmap::~mmap() { close(); }

// 打开文件并进行内存映射
inline bool mmap::open(const char *path) {
  // 关闭之前的映射
  close();

#if defined(_WIN32)
  // 在 Windows 平台上打开文件
  hFile_ = ::CreateFileA(path, GENERIC_READ, FILE_SHARE_READ, NULL,
                         OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);

  // 如果无法打开文件，则返回 false
  if (hFile_ == INVALID_HANDLE_VALUE) { return false; }

  // 获取文件大小
  size_ = ::GetFileSize(hFile_, NULL);

  // 创建文件映射
  hMapping_ = ::CreateFileMapping(hFile_, NULL, PAGE_READONLY, 0, 0, NULL);

  // 如果无法创建文件映射，则关闭并返回 false
  if (hMapping_ == NULL) {
    close();
    return false;
  }

  // 映射文件到内存
  addr_ = ::MapViewOfFile(hMapping_, FILE_MAP_READ, 0, 0, 0);
#else
  // 在非 Windows 平台上打开文件
  fd_ = ::open(path, O_RDONLY);
  // 如果无法打开文件，则返回 false
  if (fd_ == -1) { return false; }

  // 获取文件信息
  struct stat sb;
  if (fstat(fd_, &sb) == -1) {
    close();
    return false;
  }
  size_ = static_cast<size_t>(sb.st_size);

  // 映射文件到内存
  addr_ = ::mmap(NULL, size_, PROT_READ, MAP_PRIVATE, fd_, 0);
#endif

  // 如果映射地址为空，则关闭并返回 false
  if (addr_ == nullptr) {
    close();
    return false;
  }

  return true;
}

// 检查映射是否打开
inline bool mmap::is_open() const { return addr_ != nullptr; }

// 返回映射文件的大小
inline size_t mmap::size() const { return size_; }

// 返回映射文件的数据
inline const char *mmap::data() const { return (const char *)addr_; }

// 关闭映射
inline void mmap::close() {
#if defined(_WIN32)
  // 如果映射地址不为空，则取消映射
  if (addr_) {
    ::UnmapViewOfFile(addr_);
    addr_ = nullptr;
  }

  // 如果文件映射句柄不为空，则关闭
  if (hMapping_) {
    ::CloseHandle(hMapping_);
    hMapping_ = NULL;
  }

  // 如果文件句柄不是无效句柄，则关闭
  if (hFile_ != INVALID_HANDLE_VALUE) {
    ::CloseHandle(hFile_);
    # 将 hFile_ 变量设置为无效的句柄值
    hFile_ = INVALID_HANDLE_VALUE;
  }
#else
  // 如果不是 Windows 平台
  if (addr_ != nullptr) {
    // 如果地址指针不为空，释放内存映射
    munmap(addr_, size_);
    addr_ = nullptr;
  }

  if (fd_ != -1) {
    // 如果文件描述符不为 -1，关闭文件描述符
    ::close(fd_);
    fd_ = -1;
  }
#endif
  // 重置 size_ 为 0
  size_ = 0;
}
// 关闭套接字
inline int close_socket(socket_t sock) {
#ifdef _WIN32
  // 如果是 Windows 平台，调用 closesocket 函数
  return closesocket(sock);
#else
  // 如果不是 Windows 平台，调用 close 函数
  return close(sock);
#endif
}

// 处理 EINTR 信号
template <typename T> inline ssize_t handle_EINTR(T fn) {
  ssize_t res = 0;
  while (true) {
    // 调用传入的函数
    res = fn();
    // 如果返回值小于 0 且错误码为 EINTR，则继续循环
    if (res < 0 && errno == EINTR) { continue; }
    break;
  }
  return res;
}

// 读取套接字数据
inline ssize_t read_socket(socket_t sock, void *ptr, size_t size, int flags) {
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

// 发送套接字数据
inline ssize_t send_socket(socket_t sock, const void *ptr, size_t size,
                           int flags) {
  return handle_EINTR([&]() {
    return send(sock,
#ifdef _WIN32
                static_cast<const char *>(ptr), static_cast<int>(size),
#else
                ptr, size,
#endif
                flags);
  });
}

// 读取套接字数据并进行超时处理
inline ssize_t select_read(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 使用 poll 函数进行读取
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLIN;

  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 如果不是 Windows 平台且套接字大于 FD_SETSIZE，返回 1
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 使用 select 函数进行读取
  fd_set fds;
  FD_ZERO(&fds);
  FD_SET(sock, &fds);

  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), &fds, nullptr, nullptr, &tv);
  });
#endif
}

// 写入套接字数据并进行超时处理
inline ssize_t select_write(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 如果定义了 CPPHTTPLIB_USE_POLL，则使用 poll 函数进行套接字的等待
  struct pollfd pfd_read;
  // 设置 pollfd 结构体的文件描述符为 sock
  pfd_read.fd = sock;
  // 设置 pollfd 结构体的事件为 POLLOUT，表示等待写事件
  pfd_read.events = POLLOUT;

  // 计算超时时间，将秒和微秒转换为毫秒
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 调用 poll 函数进行套接字的等待，并处理 EINTR 信号
  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 如果未定义 CPPHTTPLIB_USE_POLL 且不在 Windows 环境下
  // 检查套接字是否超出 FD_SETSIZE 的限制
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 创建 fd_set 结构体并清空
  fd_set fds;
  FD_ZERO(&fds);
  // 将套接字添加到 fd_set 结构体中
  FD_SET(sock, &fds);

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 调用 select 函数进行套接字的等待，并处理 EINTR 信号
  return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), nullptr, &fds, nullptr, &tv);
  });
#endif
}

// 等待套接字准备就绪，包括读和写
inline Error wait_until_socket_is_ready(socket_t sock, time_t sec,
                                        time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 如果定义了 CPPHTTPLIB_USE_POLL，则使用 poll 函数进行套接字的等待
  struct pollfd pfd_read;
  // 设置 pollfd 结构体的文件描述符为 sock
  pfd_read.fd = sock;
  // 设置 pollfd 结构体的事件为 POLLIN | POLLOUT，表示等待读和写事件
  pfd_read.events = POLLIN | POLLOUT;

  // 计算超时时间，将秒和微秒转换为毫秒
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 调用 poll 函数进行套接字的等待，并处理 EINTR 信号
  auto poll_res = handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });

  // 如果等待超时，则返回连接超时错误
  if (poll_res == 0) { return Error::ConnectionTimeout; }

  // 如果等待成功且套接字准备就绪
  if (poll_res > 0 && pfd_read.revents & (POLLIN | POLLOUT)) {
    auto error = 0;
    socklen_t len = sizeof(error);
    // 获取套接字错误信息
    auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                          reinterpret_cast<char *>(&error), &len);
    // 判断是否连接成功
    auto successful = res >= 0 && !error;
    return successful ? Error::Success : Error::Connection;
  }

  // 返回连接错误
  return Error::Connection;
#else
#ifndef _WIN32
  // 如果未定义 CPPHTTPLIB_USE_POLL 且不在 Windows 环境下
  // 检查套接字是否超出 FD_SETSIZE 的限制
  if (sock >= FD_SETSIZE) { return Error::Connection; }
#endif

  // 创建读、写、异常 fd_set 结构体并清空
  fd_set fdsr;
  FD_ZERO(&fdsr);
  FD_SET(sock, &fdsr);

  auto fdsw = fdsr;
  auto fdse = fdsr;

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 调用 select 函数进行套接字的等待，并处理 EINTR 信号
  auto ret = handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), &fdsr, &fdsw, &fdse, &tv);
  });

  // 如果等待超时，则返回连接超时错误
  if (ret == 0) { return Error::ConnectionTimeout; }

  // 如果等待成功且套接字准备就绪
  if (ret > 0 && (FD_ISSET(sock, &fdsr) || FD_ISSET(sock, &fdsw))) {
    auto error = 0;
    socklen_t len = sizeof(error);
    // 调用 getsockopt 函数获取套接字选项信息，存储在 res 中
    auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                          reinterpret_cast<char *>(&error), &len);
    // 判断是否获取成功并且错误码为0，表示连接成功
    auto successful = res >= 0 && !error;
    // 如果连接成功，返回成功状态，否则返回连接错误状态
    return successful ? Error::Success : Error::Connection;
  }
  // 如果连接失败，返回连接错误状态
  return Error::Connection;
#endif
}

// 检查套接字是否仍然存活
inline bool is_socket_alive(socket_t sock) {
  // 使用 select_read 函数检查套接字是否可读
  const auto val = detail::select_read(sock, 0, 0);
  // 如果返回值为 0，表示套接字存活
  if (val == 0) {
    return true;
  } 
  // 如果返回值小于 0 且错误码为 EBADF，表示套接字已关闭
  else if (val < 0 && errno == EBADF) {
    return false;
  }
  // 否则，尝试从套接字中读取一个字节，如果成功则套接字存活
  char buf[1];
  return detail::read_socket(sock, &buf[0], sizeof(buf), MSG_PEEK) > 0;
}

// SocketStream 类，继承自 Stream 类
class SocketStream : public Stream {
public:
  // 构造函数
  SocketStream(socket_t sock, time_t read_timeout_sec, time_t read_timeout_usec,
               time_t write_timeout_sec, time_t write_timeout_usec);
  // 析构函数
  ~SocketStream() override;

  // 判断是否可读
  bool is_readable() const override;
  // 判断是否可写
  bool is_writable() const override;
  // 读取数据
  ssize_t read(char *ptr, size_t size) override;
  // 写入数据
  ssize_t write(const char *ptr, size_t size) override;
  // 获取远程 IP 和端口
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  // 获取本地 IP 和端口
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  // 获取套接字
  socket_t socket() const override;

private:
  socket_t sock_;
  time_t read_timeout_sec_;
  time_t read_timeout_usec_;
  time_t write_timeout_sec_;
  time_t write_timeout_usec_;

  std::vector<char> read_buff_;
  size_t read_buff_off_ = 0;
  size_t read_buff_content_size_ = 0;

  static const size_t read_buff_size_ = 1024 * 4;
};

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// SSLSocketStream 类，继承自 Stream 类
class SSLSocketStream : public Stream {
public:
  // 构造函数
  SSLSocketStream(socket_t sock, SSL *ssl, time_t read_timeout_sec,
                  time_t read_timeout_usec, time_t write_timeout_sec,
                  time_t write_timeout_usec);
  // 析构函数
  ~SSLSocketStream() override;

  // 判断是否可读
  bool is_readable() const override;
  // 判断是否可写
  bool is_writable() const override;
  // 读取数据
  ssize_t read(char *ptr, size_t size) override;
  // 写入数据
  ssize_t write(const char *ptr, size_t size) override;
  // 获取远程 IP 和端口
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  // 获取本地 IP 和端口
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  // 获取套接字
  socket_t socket() const override;

private:
  socket_t sock_;
  SSL *ssl_;
  time_t read_timeout_sec_;
  time_t read_timeout_usec_;
  time_t write_timeout_sec_;
  time_t write_timeout_usec_;
// 结束 C++ 头文件的条件编译
};
#endif

// 检查是否保持连接
inline bool keep_alive(socket_t sock, time_t keep_alive_timeout_sec) {
  // 使用命名空间 std::chrono
  using namespace std::chrono;
  // 获取当前时间点
  auto start = steady_clock::now();
  // 循环检查保持连接状态
  while (true) {
    // 调用 select_read 函数检查是否有可读数据
    auto val = select_read(sock, 0, 10000);
    // 如果 select_read 返回值小于 0，表示出错，返回 false
    if (val < 0) {
      return false;
    } else if (val == 0) {
      // 获取当前时间点
      auto current = steady_clock::now();
      // 计算时间间隔
      auto duration = duration_cast<milliseconds>(current - start);
      // 计算超时时间
      auto timeout = keep_alive_timeout_sec * 1000;
      // 如果超时，返回 false
      if (duration.count() > timeout) { return false; }
      // 线程休眠 1 毫秒
      std::this_thread::sleep_for(std::chrono::milliseconds(1));
    } else {
      // 如果有可读数据，返回 true
      return true;
    }
  }
}

// 处理服务器套接字的核心函数模板
template <typename T>
inline bool
process_server_socket_core(const std::atomic<socket_t> &svr_sock, socket_t sock,
                           size_t keep_alive_max_count,
                           time_t keep_alive_timeout_sec, T callback) {
  // 断言保持连接的最大次数大于 0
  assert(keep_alive_max_count > 0);
  // 初始化返回值为 false，保持连接次数为最大次数
  auto ret = false;
  auto count = keep_alive_max_count;
  // 循环处理保持连接
  while (svr_sock != INVALID_SOCKET && count > 0 &&
         keep_alive(sock, keep_alive_timeout_sec)) {
    // 判断是否关闭连接
    auto close_connection = count == 1;
    auto connection_closed = false;
    // 调用回调函数处理连接
    ret = callback(close_connection, connection_closed);
    // 如果回调函数返回 false 或连接已关闭，跳出循环
    if (!ret || connection_closed) { break; }
    count--;
  }
  return ret;
}

// 处理服务器套接字的核心函数模板
template <typename T>
inline bool
// 处理服务器套接字，接收参数包括服务器套接字、客户端套接字、保持连接的最大次数、保持连接的超时时间、读取超时时间、写入超时时间和回调函数
process_server_socket(const std::atomic<socket_t> &svr_sock, socket_t sock,
                      size_t keep_alive_max_count,
                      time_t keep_alive_timeout_sec, time_t read_timeout_sec,
                      time_t read_timeout_usec, time_t write_timeout_sec,
                      time_t write_timeout_usec, T callback) {
  // 调用核心服务器套接字处理函数，传入参数包括服务器套接字、客户端套接字、保持连接的最大次数、保持连接的超时时间
  return process_server_socket_core(
      svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
      // 匿名函数，处理是否关闭连接和连接是否关闭的逻辑
      [&](bool close_connection, bool &connection_closed) {
        // 创建套接字流对象，设置读取和写入超时时间
        SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                          write_timeout_sec, write_timeout_usec);
        // 调用回调函数，传入套接字流对象和是否关闭连接的标志，返回回调函数的结果
        return callback(strm, close_connection, connection_closed);
      });
}

// 处理客户端套接字，接收参数包括客户端套接字、读取超时时间、写入超时时间和回调函数
inline bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                                  time_t read_timeout_usec,
                                  time_t write_timeout_sec,
                                  time_t write_timeout_usec,
                                  std::function<bool(Stream &)> callback) {
  // 创建套接字流对象，设置读取和写入超时时间
  SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                    write_timeout_sec, write_timeout_usec);
  // 调用回调函数，传入套接字流对象，返回回调函数的结果
  return callback(strm);
}

// 关闭套接字
inline int shutdown_socket(socket_t sock) {
#ifdef _WIN32
  // Windows系统下关闭套接字
  return shutdown(sock, SD_BOTH);
#else
  // 非Windows系统下关闭套接字
  return shutdown(sock, SHUT_RDWR);
#endif
}

// 创建套接字，接收参数包括主机名、IP地址、端口号、地址族、套接字标志、TCP无延迟标志、套接字选项和绑定或连接函数
template <typename BindOrConnect>
socket_t create_socket(const std::string &host, const std::string &ip, int port,
                       int address_family, int socket_flags, bool tcp_nodelay,
                       SocketOptions socket_options,
                       BindOrConnect bind_or_connect) {
  // 获取地址信息
  const char *node = nullptr;
  struct addrinfo hints;
  struct addrinfo *result;

  // 初始化地址信息结构体
  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_protocol = 0;

  if (!ip.empty()) {
    node = ip.c_str();
    // 请求getaddrinfo将IP地址转换为地址
    # 设置地址族为未指定
    hints.ai_family = AF_UNSPEC;
    # 设置标志为数字主机
    hints.ai_flags = AI_NUMERICHOST;
  } else {
    # 如果主机名不为空，则将其转换为 C 字符串并赋给 node
    if (!host.empty()) { node = host.c_str(); }
    # 设置地址族为指定的地址族
    hints.ai_family = address_family;
    # 设置标志为指定的套接字标志
    hints.ai_flags = socket_flags;
  }
#ifndef _WIN32
  // 如果不是在 Windows 平台下
  if (hints.ai_family == AF_UNIX) {
    // 如果地址族是 AF_UNIX
    const auto addrlen = host.length();
    // 获取主机地址的长度
    if (addrlen > sizeof(sockaddr_un::sun_path) return INVALID_SOCKET;
    // 如果地址长度超过最大长度，则返回无效套接字

    auto sock = socket(hints.ai_family, hints.ai_socktype, hints.ai_protocol);
    // 创建一个套接字
    if (sock != INVALID_SOCKET) {
      // 如果套接字创建成功
      sockaddr_un addr{};
      // 创建 AF_UNIX 地址结构
      addr.sun_family = AF_UNIX;
      // 设置地址族为 AF_UNIX
      std::copy(host.begin(), host.end(), addr.sun_path);
      // 将主机地址复制到地址结构中

      hints.ai_addr = reinterpret_cast<sockaddr *>(&addr);
      // 设置地址信息为地址结构的指针
      hints.ai_addrlen = static_cast<socklen_t>(
          sizeof(addr) - sizeof(addr.sun_path) + addrlen);
      // 设置地址信息的长度

      fcntl(sock, F_SETFD, FD_CLOEXEC);
      // 设置套接字属性
      if (socket_options) { socket_options(sock); }
      // 如果有套接字选项，则设置套接字选项

      if (!bind_or_connect(sock, hints)) {
        // 如果绑定或连接失败
        close_socket(sock);
        // 关闭套接字
        sock = INVALID_SOCKET;
        // 将套接字设置为无效套接字
      }
    }
    return sock;
    // 返回套接字
  }
#endif

  auto service = std::to_string(port);
  // 将端口转换为字符串

  if (getaddrinfo(node, service.c_str(), &hints, &result)) {
    // 获取地址信息
#if defined __linux__ && !defined __ANDROID__
    res_init();
#endif
    return INVALID_SOCKET;
    // 如果获取地址信息失败，则返回无效套接字
  }

  for (auto rp = result; rp; rp = rp->ai_next) {
    // 遍历地址信息列表
    // 创建一个套接字
#ifdef _WIN32
    auto sock =
        WSASocketW(rp->ai_family, rp->ai_socktype, rp->ai_protocol, nullptr, 0,
                   WSA_FLAG_NO_HANDLE_INHERIT | WSA_FLAG_OVERLAPPED);
    /**
     * Since the WSA_FLAG_NO_HANDLE_INHERIT is only supported on Windows 7 SP1
     * and above the socket creation fails on older Windows Systems.
     *
     * Let's try to create a socket the old way in this case.
     *
     * Reference:
     * https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasocketa
     *
     * WSA_FLAG_NO_HANDLE_INHERIT:
     * This flag is supported on Windows 7 with SP1, Windows Server 2008 R2 with
     * SP1, and later
     *
     */
    // 如果使用 WSA_FLAG_NO_HANDLE_INHERIT 创建套接字失败，则尝试使用旧的方式创建套接字
    if (sock == INVALID_SOCKET) {
      sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
      // 创建套接字
    }
#else
    auto sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    // 创建套接字
#endif
    if (sock == INVALID_SOCKET) { continue; }
    // 如果创建套接字失败，则继续下一个
#ifndef _WIN32
    // 如果不是 Windows 系统，设置套接字为关闭时自动关闭
    if (fcntl(sock, F_SETFD, FD_CLOEXEC) == -1) {
      // 设置失败时关闭套接字
      close_socket(sock);
      // 继续下一次循环
      continue;
    }
#endif

    // 如果启用 TCP 无延迟
    if (tcp_nodelay) {
      auto yes = 1;
#ifdef _WIN32
      // 在 Windows 系统下设置 TCP_NODELAY 选项
      setsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                 reinterpret_cast<const char *>(&yes), sizeof(yes));
#else
      // 在非 Windows 系统下设置 TCP_NODELAY 选项
      setsockopt(sock, IPPROTO_TCP, TCP_NODELAY,
                 reinterpret_cast<const void *>(&yes), sizeof(yes));
#endif
    }

    // 如果存在套接字选项函数，调用该函数
    if (socket_options) { socket_options(sock); }

    // 如果地址族为 AF_INET6
    if (rp->ai_family == AF_INET6) {
      auto no = 0;
#ifdef _WIN32
      // 在 Windows 系统下设置 IPV6_V6ONLY 选项
      setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY,
                 reinterpret_cast<const char *>(&no), sizeof(no));
#else
      // 在非 Windows 系统下设置 IPV6_V6ONLY 选项
      setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY,
                 reinterpret_cast<const void *>(&no), sizeof(no));
#endif
    }

    // 绑定或连接套接字
    if (bind_or_connect(sock, *rp)) {
      // 释放地址信息并返回套接字
      freeaddrinfo(result);
      return sock;
    }

    // 关闭套接字
    close_socket(sock);
  }

  // 释放地址信息并返回无效套接字
  freeaddrinfo(result);
  return INVALID_SOCKET;
}

// 设置套接字为非阻塞模式
inline void set_nonblocking(socket_t sock, bool nonblocking) {
#ifdef _WIN32
  auto flags = nonblocking ? 1UL : 0UL;
  // 在 Windows 系统下设置非阻塞模式
  ioctlsocket(sock, FIONBIO, &flags);
#else
  auto flags = fcntl(sock, F_GETFL, 0);
  // 在非 Windows 系统下设置非阻塞模式
  fcntl(sock, F_SETFL,
        nonblocking ? (flags | O_NONBLOCK) : (flags & (~O_NONBLOCK)));
#endif
}

// 检查是否为连接错误
inline bool is_connection_error() {
#ifdef _WIN32
  // 在 Windows 系统下检查是否为连接错误
  return WSAGetLastError() != WSAEWOULDBLOCK;
#else
  // 在非 Windows 系统下检查是否为连接错误
  return errno != EINPROGRESS;
#endif
}

// 绑定 IP 地址
inline bool bind_ip_address(socket_t sock, const std::string &host) {
  struct addrinfo hints;
  struct addrinfo *result;

  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_family = AF_UNSPEC;
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_protocol = 0;

  // 获取地址信息
  if (getaddrinfo(host.c_str(), "0", &hints, &result)) { return false; }

  auto ret = false;
  for (auto rp = result; rp; rp = rp->ai_next) {
    const auto &ai = *rp;
    // 绑定套接字和地址信息
    if (!::bind(sock, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen))) {
      ret = true;
      break;
    }
  }
  
  // 释放地址信息内存
  freeaddrinfo(result);
  // 返回结果
  return ret;
// 如果不是在 Windows、Android、AIX 或者 __MVS__ 平台上，则定义 USE_IF2IP
#if !defined _WIN32 && !defined ANDROID && !defined _AIX && !defined __MVS__
#define USE_IF2IP
#endif

#ifdef USE_IF2IP
// 根据地址族和接口名获取 IP 地址
inline std::string if2ip(int address_family, const std::string &ifn) {
  // 获取接口地址信息
  struct ifaddrs *ifap;
  getifaddrs(&ifap);
  std::string addr_candidate;
  // 遍历接口地址信息
  for (auto ifa = ifap; ifa; ifa = ifa->ifa_next) {
    // 如果接口地址存在且接口名匹配，并且地址族为指定的地址族
    if (ifa->ifa_addr && ifn == ifa->ifa_name &&
        (AF_UNSPEC == address_family ||
         ifa->ifa_addr->sa_family == address_family)) {
      // 如果地址族为 AF_INET
      if (ifa->ifa_addr->sa_family == AF_INET) {
        auto sa = reinterpret_cast<struct sockaddr_in *>(ifa->ifa_addr);
        char buf[INET_ADDRSTRLEN];
        // 将 IPv4 地址转换为字符串形式
        if (inet_ntop(AF_INET, &sa->sin_addr, buf, INET_ADDRSTRLEN)) {
          freeifaddrs(ifap);
          return std::string(buf, INET_ADDRSTRLEN);
        }
      } else if (ifa->ifa_addr->sa_family == AF_INET6) {
        auto sa = reinterpret_cast<struct sockaddr_in6 *>(ifa->ifa_addr);
        // 如果不是链路本地地址
        if (!IN6_IS_ADDR_LINKLOCAL(&sa->sin6_addr)) {
          char buf[INET6_ADDRSTRLEN] = {};
          // 将 IPv6 地址转换为字符串形式
          if (inet_ntop(AF_INET6, &sa->sin6_addr, buf, INET6_ADDRSTRLEN)) {
            // 判断是否为独立局域网地址
            auto s6_addr_head = sa->sin6_addr.s6_addr[0];
            if (s6_addr_head == 0xfc || s6_addr_head == 0xfd) {
              addr_candidate = std::string(buf, INET6_ADDRSTRLEN);
            } else {
              freeifaddrs(ifap);
              return std::string(buf, INET6_ADDRSTRLEN);
            }
          }
        }
      }
    }
  }
  freeifaddrs(ifap);
  return addr_candidate;
}
#endif

// 创建客户端套接字
inline socket_t create_client_socket(
    const std::string &host, const std::string &ip, int port,
    int address_family, bool tcp_nodelay, SocketOptions socket_options,
    time_t connection_timeout_sec, time_t connection_timeout_usec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    // 定义写超时时间（微秒）、接口名称、错误信息
    time_t write_timeout_usec, const std::string &intf, Error &error) {
  // 创建套接字，设置参数并返回套接字对象
  auto sock = create_socket(
      host, ip, port, address_family, 0, tcp_nodelay, std::move(socket_options),
      // 匿名函数，接受套接字和地址信息作为参数，返回布尔值
      [&](socket_t sock2, struct addrinfo &ai) -> bool {
        // 如果接口名称不为空
        if (!intf.empty()) {
#ifdef USE_IF2IP
          // 如果定义了 USE_IF2IP 宏
          auto ip_from_if = if2ip(address_family, intf);
          // 获取接口对应的 IP 地址，如果为空则使用接口名称
          if (ip_from_if.empty()) { ip_from_if = intf; }
          // 如果获取的 IP 地址不为空，绑定到套接字上
          if (!bind_ip_address(sock2, ip_from_if.c_str())) {
            // 如果绑定失败，设置错误类型为绑定 IP 地址失败，返回 false
            error = Error::BindIPAddress;
            return false;
          }
#endif
        }

        // 设置套接字为非阻塞模式
        set_nonblocking(sock2, true);

        // 尝试连接到目标地址
        auto ret =
            ::connect(sock2, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen));

        // 如果连接失败
        if (ret < 0) {
          // 如果是连接错误，设置错误类型为连接错误，返回 false
          if (is_connection_error()) {
            error = Error::Connection;
            return false;
          }
          // 等待套接字就绪，直到超时
          error = wait_until_socket_is_ready(sock2, connection_timeout_sec,
                                             connection_timeout_usec);
          // 如果等待过程中出现错误，返回 false
          if (error != Error::Success) { return false; }
        }

        // 设置套接字为阻塞模式
        set_nonblocking(sock2, false);

        {
#ifdef _WIN32
          // 如果是 Windows 系统
          auto timeout = static_cast<uint32_t>(read_timeout_sec * 1000 +
                                               read_timeout_usec / 1000);
          // 设置接收超时时间
          setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO,
                     reinterpret_cast<const char *>(&timeout), sizeof(timeout));
#else
          // 如果是其他系统
          timeval tv;
          tv.tv_sec = static_cast<long>(read_timeout_sec);
          tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec);
          // 设置接收超时时间
          setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO,
                     reinterpret_cast<const void *>(&tv), sizeof(tv));
#endif
        }
        {

#ifdef _WIN32
          // 如果是 Windows 系统
          auto timeout = static_cast<uint32_t>(write_timeout_sec * 1000 +
                                               write_timeout_usec / 1000);
          // 设置发送超时时间
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO,
                     reinterpret_cast<const char *>(&timeout), sizeof(timeout));
// 如果不是 Windows 平台
#else
          // 定义 timeval 结构体
          timeval tv;
          // 设置秒数
          tv.tv_sec = static_cast<long>(write_timeout_sec);
          // 设置微秒数
          tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec);
          // 设置套接字选项 SO_SNDTIMEO，用于发送超时
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO,
                     reinterpret_cast<const void *>(&tv), sizeof(tv));
#endif
        }

        // 设置错误为成功
        error = Error::Success;
        // 返回 true
        return true;
      });

  // 如果套接字不是无效套接字
  if (sock != INVALID_SOCKET) {
    // 设置错误为成功
    error = Error::Success;
  } else {
    // 如果错误为成功，则设置错误为连接错误
    if (error == Error::Success) { error = Error::Connection; }
  }

  // 返回套接字
  return sock;
}

// 获取 IP 和端口
inline bool get_ip_and_port(const struct sockaddr_storage &addr,
                            socklen_t addr_len, std::string &ip, int &port) {
  // 如果地址族是 AF_INET
  if (addr.ss_family == AF_INET) {
    // 获取端口号
    port = ntohs(reinterpret_cast<const struct sockaddr_in *>(&addr)->sin_port);
  } else if (addr.ss_family == AF_INET6) {
    // 如果地址族是 AF_INET6，获取端口号
    port =
        ntohs(reinterpret_cast<const struct sockaddr_in6 *>(&addr)->sin6_port);
  } else {
    // 如果不是以上两种地址族，返回 false
    return false;
  }

  // 定义存储 IP 地址的数组
  std::array<char, NI_MAXHOST> ipstr{};
  // 获取 IP 地址
  if (getnameinfo(reinterpret_cast<const struct sockaddr *>(&addr), addr_len,
                  ipstr.data(), static_cast<socklen_t>(ipstr.size()), nullptr,
                  0, NI_NUMERICHOST)) {
    // 如果获取失败，返回 false
    return false;
  }

  // 将 IP 地址存储到 ip 变量中
  ip = ipstr.data();
  // 返回 true
  return true;
}

// 获取本地 IP 和端口
inline void get_local_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 定义存储地址信息的结构体
  struct sockaddr_storage addr;
  // 获取地址信息的长度
  socklen_t addr_len = sizeof(addr);
  // 如果获取本地地址信息成功，获取 IP 和端口
  if (!getsockname(sock, reinterpret_cast<struct sockaddr *>(&addr),
                   &addr_len)) {
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 获取远程 IP 和端口
inline void get_remote_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 定义存储地址信息的结构体
  struct sockaddr_storage addr;
  // 获取地址信息的长度
  socklen_t addr_len = sizeof(addr);

  // 如果获取远程地址信息成功
  if (!getpeername(sock, reinterpret_cast<struct sockaddr *>(&addr),
                   &addr_len)) {
    // 如果地址族是 AF_UNIX
#ifndef _WIN32
    if (addr.ss_family == AF_UNIX) {
#if defined(__linux__)
      // 如果是 Linux 系统
      struct ucred ucred;
      // 创建 ucred 结构体
      socklen_t len = sizeof(ucred);
      // 设置结构体长度
      if (getsockopt(sock, SOL_SOCKET, SO_PEERCRED, &ucred, &len) == 0) {
        // 获取对等端的 PID
        port = ucred.pid;
      }
#elif defined(SOL_LOCAL) && defined(SO_PEERPID) // __APPLE__
      // 如果是苹果系统
      pid_t pid;
      // 创建 PID 变量
      socklen_t len = sizeof(pid);
      // 设置 PID 变量长度
      if (getsockopt(sock, SOL_LOCAL, SO_PEERPID, &pid, &len) == 0) {
        // 获取对等端的 PID
        port = pid;
      }
#endif
      // 返回
      return;
    }
#endif
    // 获取 IP 和端口
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 将字符串转换为标签
inline constexpr unsigned int str2tag_core(const char *s, size_t l,
                                           unsigned int h) {
  return (l == 0)
             ? h
             : str2tag_core(
                   s + 1, l - 1,
                   // 重置 h 的高 6 位，因此不会发生溢出
                   (((std::numeric_limits<unsigned int>::max)() >> 6) &
                    h * 33) ^
                       static_cast<unsigned char>(*s));
}

// 将字符串转换为标签
inline unsigned int str2tag(const std::string &s) {
  return str2tag_core(s.data(), s.size(), 0);
}

// 用户定义的字面值操作符命名空间
namespace udl {

// 用户定义的字面值操作符
inline constexpr unsigned int operator"" _t(const char *s, size_t l) {
  return str2tag_core(s, l, 0);
}

} // namespace udl

// 检查是否可以压缩内容类型
inline bool can_compress_content_type(const std::string &content_type) {
  using udl::operator""_t;

  auto tag = str2tag(content_type);

  switch (tag) {
  case "image/svg+xml"_t:
  case "application/javascript"_t:
  case "application/json"_t:
  case "application/xml"_t:
  case "application/protobuf"_t:
  case "application/xhtml+xml"_t: return true;

  default:
    return !content_type.rfind("text/", 0) && tag != "text/event-stream"_t;
  }
}

// 获取编码类型
inline EncodingType encoding_type(const Request &req, const Response &res) {
  auto ret =
      detail::can_compress_content_type(res.get_header_value("Content-Type"));
  if (!ret) { return EncodingType::None; }

  const auto &s = req.get_header_value("Accept-Encoding");
  (void)(s);
}
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
  // 检查是否支持 Brotli 压缩，'Accept-Encoding' 中包含 br
  ret = s.find("br") != std::string::npos;
  // 如果支持 Brotli 压缩，则返回 Brotli 编码类型
  if (ret) { return EncodingType::Brotli; }
#endif

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 检查是否支持 Gzip 压缩，'Accept-Encoding' 中包含 gzip
  ret = s.find("gzip") != std::string::npos;
  // 如果支持 Gzip 压缩，则返回 Gzip 编码类型
  if (ret) { return EncodingType::Gzip; }
#endif

  // 如果不支持任何压缩类型，则返回无压缩类型
  return EncodingType::None;
}

// 压缩数据，使用回调函数进行处理
inline bool nocompressor::compress(const char *data, size_t data_length,
                                   bool /*last*/, Callback callback) {
  // 如果数据长度为0，则直接返回true
  if (!data_length) { return true; }
  // 调用回调函数处理数据，并返回结果
  return callback(data, data_length);
}

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// Gzip 压缩器的构造函数
inline gzip_compressor::gzip_compressor() {
  // 初始化压缩器结构体
  std::memset(&strm_, 0, sizeof(strm_));
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  // 初始化 Gzip 压缩器，设置压缩级别等参数
  is_valid_ = deflateInit2(&strm_, Z_DEFAULT_COMPRESSION, Z_DEFLATED, 31, 8,
                           Z_DEFAULT_STRATEGY) == Z_OK;
}

// Gzip 压缩器的析构函数
inline gzip_compressor::~gzip_compressor() { 
  // 结束 Gzip 压缩器
  deflateEnd(&strm_); 
}

// 压缩数据使用 Gzip 压缩算法
inline bool gzip_compressor::compress(const char *data, size_t data_length,
                                      bool last, Callback callback) {
  // 断言 Gzip 压缩器有效
  assert(is_valid_);

  do {
    // 设置最大可用输入数据长度
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    // 将数据长度转换为可用输入数据长度
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    // 设置输入数据指针
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    // 更新数据长度和数据指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    // 根据是否为最后一块数据和数据长度是否为0，设置压缩模式
    auto flush = (last && data_length == 0) ? Z_FINISH : Z_NO_FLUSH;
    auto ret = Z_OK;

    // 定义缓冲区，用于存储压缩后的数据
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    do {
      // 设置输出缓冲区大小
      strm_.avail_out = static_cast<uInt>(buff.size());
      // 设置输出缓冲区指针
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      // 执行压缩操作
      ret = deflate(&strm_, flush);
      // 如果压缩出错，返回false
      if (ret == Z_STREAM_ERROR) { return false; }

      // 调用回调函数处理压缩后的数据
      if (!callback(buff.data(), buff.size() - strm_.avail_out)) {
        return false;
      }
    } while (strm_.avail_out == 0);
    # 当输出缓冲区没有剩余空间时，继续循环执行压缩操作

    assert((flush == Z_FINISH && ret == Z_STREAM_END) ||
           (flush == Z_NO_FLUSH && ret == Z_OK));
    # 断言压缩操作的状态，要么是结束状态且返回值为 Z_STREAM_END，要么是非结束状态且返回值为 Z_OK

    assert(strm_.avail_in == 0);
    # 断言输入缓冲区已经没有剩余数据

  } while (data_length > 0);
  # 当数据长度大于 0 时，继续循环执行压缩操作

  return true;
  # 返回 true，表示压缩操作成功完成
// 默认构造函数，初始化 gzip 解压缩器
inline gzip_decompressor::gzip_decompressor() {
  // 将解压缩器结构体清零
  std::memset(&strm_, 0, sizeof(strm_));
  // 设置内存分配器和释放器为 NULL
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  // wbits 的值为 15，表示应该设置为最大可能值，以确保可以解码任何 gzip 流
  // 偏移量 32 指定流类型应该自动检测 gzip 或 deflate
  is_valid_ = inflateInit2(&strm_, 32 + 15) == Z_OK;
}

// 析构函数，释放 gzip 解压缩器
inline gzip_decompressor::~gzip_decompressor() { inflateEnd(&strm_); }

// 检查解压缩器是否有效
inline bool gzip_decompressor::is_valid() const { return is_valid_; }

// 解压缩数据
inline bool gzip_decompressor::decompress(const char *data, size_t data_length,
                                          Callback callback) {
  // 断言解压缩器有效
  assert(is_valid_);

  auto ret = Z_OK;

  do {
    // 设置可用输入数据的最大值
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    // 将输入数据长度限制在最大可用输入数据范围内
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    // 设置输入数据指针
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    // 更新数据长度和数据指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    // 创建缓冲区
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    // 当有可用输入数据且解压缩成功时循环
    while (strm_.avail_in > 0 && ret == Z_OK) {
      // 设置输出缓冲区大小
      strm_.avail_out = static_cast<uInt>(buff.size());
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      // 解压缩数据
      ret = inflate(&strm_, Z_NO_FLUSH);

      // 断言解压缩操作不会返回 Z_STREAM_ERROR
      assert(ret != Z_STREAM_ERROR);
      switch (ret) {
      case Z_NEED_DICT:
      case Z_DATA_ERROR:
      case Z_MEM_ERROR: inflateEnd(&strm_); return false;
      }

      // 如果回调函数返回 false，则返回 false
      if (!callback(buff.data(), buff.size() - strm_.avail_out)) {
        return false;
      }
    }

    // 如果解压缩操作不是 Z_OK 或 Z_STREAM_END，则返回 false
    if (ret != Z_OK && ret != Z_STREAM_END) return false;

  } while (data_length > 0);

  return true;
}
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
// 默认构造函数，创建 brotli 压缩器实例
inline brotli_compressor::brotli_compressor() {
  state_ = BrotliEncoderCreateInstance(nullptr, nullptr, nullptr);
}
// 在析构函数中销毁 Brotli 压缩器实例
inline brotli_compressor::~brotli_compressor() {
  BrotliEncoderDestroyInstance(state_);
}

// 压缩数据的函数，接受数据、数据长度、是否为最后一块数据和回调函数
inline bool brotli_compressor::compress(const char *data, size_t data_length,
                                        bool last, Callback callback) {
  // 创建一个大小为 CPPHTTPLIB_COMPRESSION_BUFSIZ 的缓冲区
  std::array<uint8_t, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};

  // 设置操作类型为结束或处理
  auto operation = last ? BROTLI_OPERATION_FINISH : BROTLI_OPERATION_PROCESS;
  auto available_in = data_length;
  auto next_in = reinterpret_cast<const uint8_t *>(data);

  // 循环处理数据
  for (;;) {
    // 如果是最后一块数据，检查是否压缩完成
    if (last) {
      if (BrotliEncoderIsFinished(state_)) { break; }
    } else {
      // 如果不是最后一块数据，检查是否还有数据可用
      if (!available_in) { break; }
    }

    auto available_out = buff.size();
    auto next_out = buff.data();

    // 压缩数据流
    if (!BrotliEncoderCompressStream(state_, operation, &available_in, &next_in,
                                     &available_out, &next_out, nullptr)) {
      return false;
    }

    // 计算输出的字节数并调用回调函数
    auto output_bytes = buff.size() - available_out;
    if (output_bytes) {
      callback(reinterpret_cast<const char *>(buff.data()), output_bytes);
    }
  }

  return true;
}

// 创建 Brotli 解压缩器实例
inline brotli_decompressor::brotli_decompressor() {
  decoder_s = BrotliDecoderCreateInstance(0, 0, 0);
  decoder_r = decoder_s ? BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT
                        : BROTLI_DECODER_RESULT_ERROR;
}

// 在析构函数中销毁 Brotli 解压缩器实例
inline brotli_decompressor::~brotli_decompressor() {
  if (decoder_s) { BrotliDecoderDestroyInstance(decoder_s); }
}

// 检查解压缩器实例是否有效
inline bool brotli_decompressor::is_valid() const { return decoder_s; }

// 解压数据的函数，接受数据、数据长度和回调函数
inline bool brotli_decompressor::decompress(const char *data,
                                            size_t data_length,
                                            Callback callback) {
  // 如果解压缩成功或出现错误，则返回
  if (decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
      decoder_r == BROTLI_DECODER_RESULT_ERROR) {
    // 返回整数值 0
    return 0;
  }

  // 将 data 强制转换为 uint8_t 类型的指针，并赋值给 next_in
  auto next_in = reinterpret_cast<const uint8_t *>(data);
  // 将 data_length 赋值给 avail_in
  size_t avail_in = data_length;
  // 定义 total_out 变量

  // 初始化 decoder_r 为 BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT
  decoder_r = BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT;

  // 创建大小为 CPPHTTPLIB_COMPRESSION_BUFSIZ 的字符数组 buff
  std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
  // 当 decoder_r 等于 BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT 时循环执行以下代码块
  while (decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT) {
    // 将 buff.data() 赋值给 next_out
    char *next_out = buff.data();
    // 将 buff.size() 赋值给 avail_out
    size_t avail_out = buff.size();

    // 调用 BrotliDecoderDecompressStream 函数进行解压缩操作
    decoder_r = BrotliDecoderDecompressStream(
        decoder_s, &avail_in, &next_in, &avail_out,
        reinterpret_cast<uint8_t **>(&next_out), &total_out);

    // 如果解压缩出错，则返回 false
    if (decoder_r == BROTLI_DECODER_RESULT_ERROR) { return false; }

    // 如果回调函数返回 false，则返回 false
    if (!callback(buff.data(), buff.size() - avail_out)) { return false; }
  }

  // 返回解压缩结果是否成功或需要更多输入
  return decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
         decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT;
// 结束 C++ 头文件的条件编译
}
#endif

// 检查 Headers 中是否存在指定 key
inline bool has_header(const Headers &headers, const std::string &key) {
  return headers.find(key) != headers.end();
}

// 获取指定 key 的 header 值
inline const char *get_header_value(const Headers &headers,
                                    const std::string &key, size_t id,
                                    const char *def) {
  auto rng = headers.equal_range(key);
  auto it = rng.first;
  std::advance(it, static_cast<ssize_t>(id));
  if (it != rng.second) { return it->second.c_str(); }
  return def;
}

// 不区分大小写比较两个字符串
inline bool compare_case_ignore(const std::string &a, const std::string &b) {
  if (a.size() != b.size()) { return false; }
  for (size_t i = 0; i < b.size(); i++) {
    if (::tolower(a[i]) != ::tolower(b[i])) { return false; }
  }
  return true;
}

// 解析 HTTP 头部
template <typename T>
inline bool parse_header(const char *beg, const char *end, T fn) {
  // 跳过末尾的空格和制表符
  while (beg < end && is_space_or_tab(end[-1])) {
    end--;
  }

  auto p = beg;
  while (p < end && *p != ':') {
    p++;
  }

  if (p == end) { return false; }

  auto key_end = p;

  if (*p++ != ':') { return false; }

  while (p < end && is_space_or_tab(*p)) {
    p++;
  }

  if (p < end) {
    auto key = std::string(beg, key_end);
    auto val = compare_case_ignore(key, "Location")
                   ? std::string(p, end)
                   : decode_url(std::string(p, end), false);
    fn(std::move(key), std::move(val));
    return true;
  }

  return false;
}

// 读取 HTTP 头部
inline bool read_headers(Stream &strm, Headers &headers) {
  const auto bufsiz = 2048;
  char buf[bufsiz];
  stream_line_reader line_reader(strm, buf, bufsiz);

  for (;;) {
    if (!line_reader.getline()) { return false; }

    // 检查行是否以 CRLF 结尾
    auto line_terminator_len = 2;
    if (line_reader.end_with_crlf()) {
      // 空行表示头部结束
      if (line_reader.size() == 2) { break; }
#ifdef CPPHTTPLIB_ALLOW_LF_AS_LINE_TERMINATOR
    } else {
      // 如果遇到空行，则表示头部结束
      if (line_reader.size() == 1) { break; }
      // 设置行终止符长度为1
      line_terminator_len = 1;
    }
#else
    } else {
      continue; // 跳过无效行。
    }
#endif

    // 检查读取的行是否超过最大长度限制
    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }

    // 排除行终止符
    auto end = line_reader.ptr() + line_reader.size() - line_terminator_len;

    // 解析头部信息，将键值对添加到 headers 中
    parse_header(line_reader.ptr(), end,
                 [&](std::string &&key, std::string &&val) {
                   headers.emplace(std::move(key), std::move(val));
                 });
  }

  return true;
}

// 读取指定长度的内容，并通过进度回调函数进行处理
inline bool read_content_with_length(Stream &strm, uint64_t len,
                                     Progress progress,
                                     ContentReceiverWithProgress out) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ];

  uint64_t r = 0;
  while (r < len) {
    auto read_len = static_cast<size_t>(len - r);
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ));
    if (n <= 0) { return false; }

    if (!out(buf, static_cast<size_t>(n), r, len)) { return false; }
    r += static_cast<uint64_t>(n);

    if (progress) {
      if (!progress(r, len)) { return false; }
    }
  }

  return true;
}

// 跳过指定长度的内容
inline void skip_content_with_length(Stream &strm, uint64_t len) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ];
  uint64_t r = 0;
  while (r < len) {
    auto read_len = static_cast<size_t>(len - r);
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ));
    if (n <= 0) { return; }
    r += static_cast<uint64_t>(n);
  }
}

// 读取不定长度的内容，并通过回调函数进行处理
inline bool read_content_without_length(Stream &strm,
                                        ContentReceiverWithProgress out) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ];
  uint64_t r = 0;
  for (;;) {
    auto n = strm.read(buf, CPPHTTPLIB_RECV_BUFSIZ);
    if (n < 0) {
      return false;
    } else if (n == 0) {
      return true;
    }

    if (!out(buf, static_cast<size_t>(n), r, 0)) { return false; }
    r += static_cast<uint64_t>(n);
  }

  return true;
}

// 模板函数定义
template <typename T>
// 从流中读取分块内容，并将内容传递给指定的接收器，同时显示进度
inline bool read_content_chunked(Stream &strm, T &x,
                                 ContentReceiverWithProgress out) {
  // 定义缓冲区大小
  const auto bufsiz = 16;
  // 创建缓冲区
  char buf[bufsiz];

  // 创建流行读取器对象
  stream_line_reader line_reader(strm, buf, bufsiz);

  // 读取第一行
  if (!line_reader.getline()) { return false; }

  unsigned long chunk_len;
  // 循环读取分块数据
  while (true) {
    char *end_ptr;

    // 解析分块长度
    chunk_len = std::strtoul(line_reader.ptr(), &end_ptr, 16);

    // 检查是否解析成功
    if (end_ptr == line_reader.ptr()) { return false; }
    if (chunk_len == ULONG_MAX) { return false; }

    // 处理分块数据
    if (chunk_len == 0) { break; }

    // 读取指定长度的内容
    if (!read_content_with_length(strm, chunk_len, nullptr, out)) {
      return false;
    }

    // 读取下一行
    if (!line_reader.getline()) { return false; }

    // 检查分块结束符
    if (strcmp(line_reader.ptr(), "\r\n")) { return false; }

    // 读取下一行
    if (!line_reader.getline()) { return false; }
  }

  // 确保分块长度为0
  assert(chunk_len == 0);

  // Trailer
  // 读取 trailer 部分
  if (!line_reader.getline()) { return false; }

  // 处理 trailer 部分的头信息
  while (strcmp(line_reader.ptr(), "\r\n")) {
    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }

    // 排除行终止符
    constexpr auto line_terminator_len = 2;
    auto end = line_reader.ptr() + line_reader.size() - line_terminator_len;

    // 解析头信息
    parse_header(line_reader.ptr(), end,
                 [&](std::string &&key, std::string &&val) {
                   x.headers.emplace(std::move(key), std::move(val));
                 });

    // 读取下一行
    if (!line_reader.getline()) { return false; }
  }

  return true;
}

// 检查是否使用分块传输编码
inline bool is_chunked_transfer_encoding(const Headers &headers) {
  return !strcasecmp(get_header_value(headers, "Transfer-Encoding", 0, ""),
                     "chunked");
}

// 准备内容接收器
template <typename T, typename U>
bool prepare_content_receiver(T &x, int &status,
                              ContentReceiverWithProgress receiver,
                              bool decompress, U callback) {
  if (decompress) {
    // 获取内容编码
    std::string encoding = x.get_header_value("Content-Encoding");
    std::unique_ptr<decompressor> decompressor;
    # 如果编码是 gzip 或 deflate
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
      // 如果支持 ZLIB 压缩，则创建一个 gzip 解压缩器
      decompressor = detail::make_unique<gzip_decompressor>();
#else
      // 如果不支持 ZLIB 压缩，则设置状态码为 415 并返回 false
      status = 415;
      return false;
#endif
    } else if (encoding.find("br") != std::string::npos) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
      // 如果支持 BROTLI 压缩，则创建一个 brotli 解压缩器
      decompressor = detail::make_unique<brotli_decompressor>();
#else
      // 如果不支持 BROTLI 压缩，则设置状态码为 415 并返回 false
      status = 415;
      return false;
#endif
    }

    if (decompressor) {
      // 如果存在解压缩器
      if (decompressor->is_valid()) {
        // 创建一个带进度的内容接收器，用于解压缩数据并传递给接收器
        ContentReceiverWithProgress out = [&](const char *buf, size_t n,
                                              uint64_t off, uint64_t len) {
          return decompressor->decompress(buf, n,
                                          [&](const char *buf2, size_t n2) {
                                            return receiver(buf2, n2, off, len);
                                          });
        };
        // 调用回调函数，并传递内容接收器
        return callback(std::move(out));
      } else {
        // 如果解压缩器无效，则设置状态码为 500 并返回 false
        status = 500;
        return false;
      }
    }
  }

  // 创建一个内容接收器，用于接收数据并传递给接收器
  ContentReceiverWithProgress out = [&](const char *buf, size_t n, uint64_t off,
                                        uint64_t len) {
    return receiver(buf, n, off, len);
  };
  // 调用回调函数，并传递内容接收器
  return callback(std::move(out));
}

template <typename T>
// 从流中读取内容并根据参数进行处理，返回处理结果的布尔值
bool read_content(Stream &strm, T &x, size_t payload_max_length, int &status,
                  Progress progress, ContentReceiverWithProgress receiver,
                  bool decompress) {
  // 准备内容接收器，根据条件调用不同的处理函数
  return prepare_content_receiver(
      x, status, std::move(receiver), decompress,
      [&](const ContentReceiverWithProgress &out) {
        auto ret = true;
        auto exceed_payload_max_length = false;

        // 如果使用分块传输编码，则调用相应的处理函数
        if (is_chunked_transfer_encoding(x.headers)) {
          ret = read_content_chunked(strm, x, out);
        } 
        // 如果没有 Content-Length 头部，则调用相应的处理函数
        else if (!has_header(x.headers, "Content-Length")) {
          ret = read_content_without_length(strm, out);
        } 
        // 否则根据 Content-Length 头部的值进行处理
        else {
          auto len = get_header_value_u64(x.headers, "Content-Length", 0, 0);
          // 如果内容长度超过最大长度限制，则跳过内容并返回 false
          if (len > payload_max_length) {
            exceed_payload_max_length = true;
            skip_content_with_length(strm, len);
            ret = false;
          } 
          // 否则根据内容长度调用相应的处理函数
          else if (len > 0) {
            ret = read_content_with_length(strm, len, std::move(progress), out);
          }
        }

        // 如果处理结果为 false，则根据是否超过最大长度限制设置状态码
        if (!ret) { status = exceed_payload_max_length ? 413 : 400; }
        return ret;
      });
} // namespace detail

// 写入头部信息到流中，并返回写入的字节数
inline ssize_t write_headers(Stream &strm, const Headers &headers) {
  ssize_t write_len = 0;
  // 遍历头部信息，格式化写入到流中
  for (const auto &x : headers) {
    auto len =
        strm.write_format("%s: %s\r\n", x.first.c_str(), x.second.c_str());
    if (len < 0) { return len; }
    write_len += len;
  }
  // 写入空行
  auto len = strm.write("\r\n");
  if (len < 0) { return len; }
  write_len += len;
  return write_len;
}

// 写入数据到流中，返回写入是否成功的布尔值
inline bool write_data(Stream &strm, const char *d, size_t l) {
  size_t offset = 0;
  // 循环写入数据直到全部写入完成
  while (offset < l) {
    auto length = strm.write(d + offset, l - offset);
    if (length < 0) { return false; }
    offset += static_cast<size_t>(length);
  }
  return true;
}

template <typename T>
// 写入内容到流中，根据提供的内容提供者、偏移量、长度、关闭标志和错误信息
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length, T is_shutting_down,
                          Error &error) {
  // 计算结束偏移量
  size_t end_offset = offset + length;
  // 初始化变量
  auto ok = true;
  DataSink data_sink;

  // 写入数据的 Lambda 函数
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果操作成功
    if (ok) {
      // 如果流可写，并且成功写入数据
      if (strm.is_writable() && write_data(strm, d, l)) {
        offset += l;
      } else {
        ok = false;
      }
    }
    return ok;
  };

  // 循环写入数据，直到达到结束偏移量或关闭标志为真
  while (offset < end_offset && !is_shutting_down()) {
    // 如果流不可写
    if (!strm.is_writable()) {
      error = Error::Write;
      return false;
    } else if (!content_provider(offset, end_offset - offset, data_sink)) {
      error = Error::Canceled;
      return false;
    } else if (!ok) {
      error = Error::Write;
      return false;
    }
  }

  // 操作成功，返回 true
  error = Error::Success;
  return true;
}

// 写入内容到流中，根据提供的内容提供者、偏移量、长度和关闭标志
template <typename T>
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length,
                          const T &is_shutting_down) {
  auto error = Error::Success;
  // 调用上面的函数
  return write_content(strm, content_provider, offset, length, is_shutting_down,
                       error);
}

// 写入内容到流中，根据提供的内容提供者和关闭标志，不指定长度
template <typename T>
inline bool
write_content_without_length(Stream &strm,
                             const ContentProvider &content_provider,
                             const T &is_shutting_down) {
  // 初始化变量
  size_t offset = 0;
  auto data_available = true;
  auto ok = true;
  DataSink data_sink;

  // 写入数据的 Lambda 函数
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果操作成功
    if (ok) {
      offset += l;
      // 如果流不可写或写入数据失败
      if (!strm.is_writable() || !write_data(strm, d, l)) { ok = false; }
    }
    return ok;
  };

  // 完成写入数据的 Lambda 函数
  data_sink.done = [&](void) { data_available = false; };

  // 循环写入数据，直到数据不可用或关闭标志为真
  while (data_available && !is_shutting_down()) {
    // 如果流不可写
    if (!strm.is_writable()) {
      return false;
    } else if (!content_provider(offset, 0, data_sink)) {
      return false;
    }
  }
}
    } else if (!ok) {
      // 如果不满足条件，返回 false
      return false;
    }
  }
  // 循环结束后返回 true
  return true;
// 写入内容的函数，支持分块传输，接受流、内容提供者、关闭标志、压缩器和错误对象作为参数
template <typename T, typename U>
inline bool
write_content_chunked(Stream &strm, const ContentProvider &content_provider,
                      const T &is_shutting_down, U &compressor, Error &error) {
  // 初始化偏移量、数据是否可用、操作是否成功的标志和数据接收器
  size_t offset = 0;
  auto data_available = true;
  auto ok = true;
  DataSink data_sink;

  // 数据接收器的写入操作，处理数据压缩和分块传输
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果操作成功
    if (ok) {
      // 检查数据是否可用
      data_available = l > 0;
      offset += l;

      // 压缩数据并生成分块传输的数据块
      std::string payload;
      if (compressor.compress(d, l, false,
                              [&](const char *data, size_t data_len) {
                                payload.append(data, data_len);
                                return true;
                              })) {
        // 如果生成的数据块不为空
        if (!payload.empty()) {
          // 发送分块传输的响应头和尾部
          auto chunk =
              from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
          // 如果流不可写或写入数据失败，则操作失败
          if (!strm.is_writable() ||
              !write_data(strm, chunk.data(), chunk.size())) {
            ok = false;
          }
        }
      } else {
        ok = false;
      }
    }
    return ok;
  };

  // 完成传输时的操作，处理尾部数据
  auto done_with_trailer = [&](const Headers *trailer) {
    // 如果操作失败，则直接返回
    if (!ok) { return; }

    // 数据不可用
    data_available = false;

    // 压缩尾部数据
    std::string payload;
    if (!compressor.compress(nullptr, 0, true,
                             [&](const char *data, size_t data_len) {
                               payload.append(data, data_len);
                               return true;
                             })) {
      ok = false;
      return;
    }

    // 如果生成的尾部数据不为空
    if (!payload.empty()) {
      // 发送分块传输的响应头和尾部
      auto chunk = from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
      // 如果流不可写或写入数据失败，则操作失败
      if (!strm.is_writable() ||
          !write_data(strm, chunk.data(), chunk.size())) {
        ok = false;
        return;
      }
    }

    // 完成传输的标志
    static const std::string done_marker("0\r\n");
    // 如果写入数据失败，则将 ok 置为 false
    if (!write_data(strm, done_marker.data(), done_marker.size())) {
      ok = false;
    }

    // Trailer
    // 如果 trailer 存在，则遍历 trailer，将每个键值对组成 field_line，并写入数据流
    if (trailer) {
      for (const auto &kv : *trailer) {
        std::string field_line = kv.first + ": " + kv.second + "\r\n";
        if (!write_data(strm, field_line.data(), field_line.size())) {
          ok = false;
        }
      }
    }

    // 写入回车换行符
    static const std::string crlf("\r\n");
    if (!write_data(strm, crlf.data(), crlf.size())) { ok = false; }
  };

  // 当数据传输完成时调用 done_with_trailer 函数
  data_sink.done = [&](void) { done_with_trailer(nullptr); };

  // 当数据传输完成且 trailer 存在时调用 done_with_trailer 函数
  data_sink.done_with_trailer = [&](const Headers &trailer) {
    done_with_trailer(&trailer);
  };

  // 循环处理数据，直到数据不可用或正在关闭连接
  while (data_available && !is_shutting_down()) {
    // 如果数据流不可写，则返回写入错误
    if (!strm.is_writable()) {
      error = Error::Write;
      return false;
    } else if (!content_provider(offset, 0, data_sink)) {
      // 如果内容提供者返回 false，则返回取消错误
      error = Error::Canceled;
      return false;
    } else if (!ok) {
      // 如果写入数据失败，则返回写入错误
      error = Error::Write;
      return false;
    }
  }

  // 数据传输成功，返回成功状态
  error = Error::Success;
  return true;
// 写入内容分块到流中，使用内容提供者、关闭标志、压缩器
template <typename T, typename U>
inline bool write_content_chunked(Stream &strm,
                                  const ContentProvider &content_provider,
                                  const T &is_shutting_down, U &compressor) {
  auto error = Error::Success;
  // 调用重载函数，写入内容分块到流中
  return write_content_chunked(strm, content_provider, is_shutting_down,
                               compressor, error);
}

// 重定向请求到指定路径，更新请求和响应对象
template <typename T>
inline bool redirect(T &cli, Request &req, Response &res,
                     const std::string &path, const std::string &location,
                     Error &error) {
  // 复制原始请求对象
  Request new_req = req;
  new_req.path = path;
  new_req.redirect_count_ -= 1;

  // 处理 303 状态码的情况
  if (res.status == 303 && (req.method != "GET" && req.method != "HEAD")) {
    new_req.method = "GET";
    new_req.body.clear();
    new_req.headers.clear();
  }

  // 创建新的响应对象
  Response new_res;

  // 发送新请求，获取新响应
  auto ret = cli.send(new_req, new_res, error);
  if (ret) {
    req = new_req;
    res = new_res;

    // 更新响应的重定向地址
    if (res.location.empty()) res.location = location;
  }
  return ret;
}

// 将参数转换为查询字符串
inline std::string params_to_query_str(const Params &params) {
  std::string query;

  // 遍历参数，构建查询字符串
  for (auto it = params.begin(); it != params.end(); ++it) {
    if (it != params.begin()) { query += "&"; }
    query += it->first;
    query += "=";
    query += encode_query_param(it->second);
  }
  return query;
}

// 解析查询字符串为参数
inline void parse_query_text(const std::string &s, Params &params) {
  std::set<std::string> cache;
  // 分割查询字符串，解析键值对
  split(s.data(), s.data() + s.size(), '&', [&](const char *b, const char *e) {
    std::string kv(b, e);
    if (cache.find(kv) != cache.end()) { return; }
    cache.insert(kv);

    std::string key;
    std::string val;
    split(b, e, '=', [&](const char *b2, const char *e2) {
      if (key.empty()) {
        key.assign(b2, e2);
      } else {
        val.assign(b2, e2);
      }
    });

    // 解码键值对，并添加到参数中
    if (!key.empty()) {
      params.emplace(decode_url(key, true), decode_url(val, true));
    }
  });
}
// 解析 multipart 请求中的 boundary 参数，并将结果保存在 boundary 变量中
inline bool parse_multipart_boundary(const std::string &content_type,
                                     std::string &boundary) {
  // 查找 boundary= 关键字在 content_type 中的位置
  auto boundary_keyword = "boundary=";
  auto pos = content_type.find(boundary_keyword);
  // 如果未找到 boundary= 关键字，则返回 false
  if (pos == std::string::npos) { return false; }
  // 查找 boundary= 后的第一个分号的位置
  auto end = content_type.find(';', pos);
  auto beg = pos + strlen(boundary_keyword);
  // 提取 boundary= 后的值，并去除双引号
  boundary = trim_double_quotes_copy(content_type.substr(beg, end - beg));
  // 如果 boundary 不为空，则返回 true
  return !boundary.empty();
}

// 解析 disposition 参数，并将结果保存在 params 中
inline void parse_disposition_params(const std::string &s, Params &params) {
  // 创建一个缓存集合
  std::set<std::string> cache;
  // 使用分号分割字符串 s，并遍历每个分割后的子串
  split(s.data(), s.data() + s.size(), ';', [&](const char *b, const char *e) {
    std::string kv(b, e);
    // 如果 kv 已经在缓存中存在，则直接返回
    if (cache.find(kv) != cache.end()) { return; }
    cache.insert(kv);

    std::string key;
    std::string val;
    // 使用等号分割子串，并提取 key 和 value
    split(b, e, '=', [&](const char *b2, const char *e2) {
      if (key.empty()) {
        key.assign(b2, e2);
      } else {
        val.assign(b2, e2);
      }
    });

    // 如果 key 不为空，则将 key 和 value 添加到 params 中
    if (!key.empty()) {
      params.emplace(trim_double_quotes_copy((key)),
                     trim_double_quotes_copy((val)));
    }
  });
}

#ifdef CPPHTTPLIB_NO_EXCEPTIONS
// 解析 Range 头部，并将结果保存在 ranges 中，不使用异常处理
inline bool parse_range_header(const std::string &s, Ranges &ranges) {
#else
// 解析 Range 头部，并将结果保存在 ranges 中，使用异常处理
inline bool parse_range_header(const std::string &s, Ranges &ranges) try {
#endif
  // 定义正则表达式来匹配 Range 头部
  static auto re_first_range = std::regex(R"(bytes=(\d*-\d*(?:,\s*\d*-\d*)*))");
  std::smatch m;
  // 使用正则表达式匹配 Range 头部
  if (std::regex_match(s, m, re_first_range)) {
    auto pos = static_cast<size_t>(m.position(1));
    auto len = static_cast<size_t>(m.length(1));
    auto all_valid_ranges = true;
    // 使用 split 函数按逗号分割字符串，并对每个分割后的子串执行 lambda 函数
    split(&s[pos], &s[pos + len], ',', [&](const char *b, const char *e) {
      // 如果不是所有范围都有效，则直接返回
      if (!all_valid_ranges) return;
      // 定义正则表达式 re_another_range 匹配数字范围
      static auto re_another_range = std::regex(R"(\s*(\d*)-(\d*))");
      // 定义匹配结果对象 cm
      std::cmatch cm;
      // 如果子串匹配到数字范围的正则表达式
      if (std::regex_match(b, e, cm, re_another_range)) {
        // 初始化第一个数字范围为 -1
        ssize_t first = -1;
        // 如果第一个数字范围不为空，则转换为整数
        if (!cm.str(1).empty()) {
          first = static_cast<ssize_t>(std::stoll(cm.str(1)));
        }

        // 初始化最后一个数字范围为 -1
        ssize_t last = -1;
        // 如果最后一个数字范围不为空，则转换为整数
        if (!cm.str(2).empty()) {
          last = static_cast<ssize_t>(std::stoll(cm.str(2)));
        }

        // 如果第一个和最后一个数字范围都有效且第一个大于最后一个，则标记所有范围无效并返回
        if (first != -1 && last != -1 && first > last) {
          all_valid_ranges = false;
          return;
        }
        // 将有效的数字范围加入到 ranges 中
        ranges.emplace_back(std::make_pair(first, last));
      }
    });
    // 返回所有范围是否有效的结果
    return all_valid_ranges;
  }
  // 如果没有匹配到有效的数字范围，则返回 false
  return false;
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
} // end of #ifdef CPPHTTPLIB_NO_EXCEPTIONS block
#else
} catch (...) { return false; } // catch any exception and return false
#endif

// 定义一个类 MultipartFormDataParser
class MultipartFormDataParser {
public:
  // 默认构造函数
  MultipartFormDataParser() = default;

  // 设置分隔符
  void set_boundary(std::string &&boundary) {
    boundary_ = boundary;
    dash_boundary_crlf_ = dash_ + boundary_ + crlf_;
    crlf_dash_boundary_ = crlf_ + dash_ + boundary_;
  }

  // 检查是否有效
  bool is_valid() const { return is_valid_; }

  // 解析数据
  bool parse(const char *buf, size_t n, const ContentReceiver &content_callback,
             const MultipartContentHeader &header_callback) {

    buf_append(buf, n);

    // 返回解析结果
    return true;
  }

private:
  // 清除文件信息
  void clear_file_info() {
    file_.name.clear();
    file_.filename.clear();
    file_.content_type.clear();
  }

  // 不区分大小写比较字符串
  bool start_with_case_ignore(const std::string &a,
                              const std::string &b) const {
    if (a.size() < b.size()) { return false; }
    for (size_t i = 0; i < b.size(); i++) {
      if (::tolower(a[i]) != ::tolower(b[i])) { return false; }
    }
    return true;
  }

  const std::string dash_ = "--";
  const std::string crlf_ = "\r\n";
  std::string boundary_;
  std::string dash_boundary_crlf_;
  std::string crlf_dash_boundary_;

  size_t state_ = 0;
  bool is_valid_ = false;
  MultipartFormData file_;

  // 缓冲区
  bool start_with(const std::string &a, size_t spos, size_t epos,
                  const std::string &b) const {
    if (epos - spos < b.size()) { return false; }
    for (size_t i = 0; i < b.size(); i++) {
      if (a[i + spos] != b[i]) { return false; }
    }
    return true;
  }

  size_t buf_size() const { return buf_epos_ - buf_spos_; }

  const char *buf_data() const { return &buf_[buf_spos_]; }

  std::string buf_head(size_t l) const { return buf_.substr(buf_spos_, l); }

  bool buf_start_with(const std::string &s) const {
    return start_with(buf_, buf_spos_, buf_epos_, s);
  }

  size_t buf_find(const std::string &s) const {
    auto c = s.front();

    size_t off = buf_spos_;
    // 当 off 小于 buf_epos_ 时，进入循环
    while (off < buf_epos_) {
      // 记录当前位置为 pos
      auto pos = off;
      // 在当前位置开始查找字符 c
      while (true) {
        // 如果 pos 等于 buf_epos_，返回缓冲区大小
        if (pos == buf_epos_) { return buf_size(); }
        // 如果找到字符 c，跳出循环
        if (buf_[pos] == c) { break; }
        // 继续向后查找
        pos++;
      }

      // 计算剩余大小
      auto remaining_size = buf_epos_ - pos;
      // 如果字符串 s 的长度大于剩余大小，返回缓冲区大小
      if (s.size() > remaining_size) { return buf_size(); }

      // 如果 buf_ 从 pos 开始的子串与 s 相同，返回匹配位置
      if (start_with(buf_, pos, buf_epos_, s)) { return pos - buf_spos_; }

      // 更新 off 位置
      off = pos + 1;
    }

    // 返回缓冲区大小
    return buf_size();
  }

  // 向缓冲区追加数据
  void buf_append(const char *data, size_t n) {
    // 计算剩余大小
    auto remaining_size = buf_size();
    // 如果剩余大小大于 0 且 buf_spos_ 大于 0
    if (remaining_size > 0 && buf_spos_ > 0) {
      // 将 buf_ 中的数据向前移动
      for (size_t i = 0; i < remaining_size; i++) {
        buf_[i] = buf_[buf_spos_ + i];
      }
    }
    // 重置 buf_spos_ 和 buf_epos_
    buf_spos_ = 0;
    buf_epos_ = remaining_size;

    // 如果剩余大小加上 n 大于 buf_ 的大小，重新调整 buf_ 的大小
    if (remaining_size + n > buf_.size()) { buf_.resize(remaining_size + n); }

    // 将 data 中的数据追加到 buf_ 中
    for (size_t i = 0; i < n; i++) {
      buf_[buf_epos_ + i] = data[i];
    }
    // 更新 buf_epos_
    buf_epos_ += n;
  }

  // 从缓冲区中删除指定大小的数据
  void buf_erase(size_t size) { buf_spos_ += size; }

  // 缓冲区数据
  std::string buf_;
  // 缓冲区起始位置
  size_t buf_spos_ = 0;
  // 缓冲区结束位置
  size_t buf_epos_ = 0;
// 结束 C++ 代码块
};

// 将字符数组转换为小写字符串
inline std::string to_lower(const char *beg, const char *end) {
  // 初始化空字符串
  std::string out;
  // 从起始位置开始遍历到结束位置
  auto it = beg;
  while (it != end) {
    // 将字符转换为小写并添加到字符串中
    out += static_cast<char>(::tolower(*it));
    it++;
  }
  // 返回转换后的字符串
  return out;
}

// 生成用于分隔 multipart 数据的随机字符串
inline std::string make_multipart_data_boundary() {
  // 静态字符数组作为随机字符串的字符集
  static const char data[] =
      "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

  // 使用随机设备生成种子
  std::random_device seed_gen;

  // 请求 128 位的熵用于初始化
  std::seed_seq seed_sequence{seed_gen(), seed_gen(), seed_gen(), seed_gen()};
  std::mt19937 engine(seed_sequence);

  // 初始化结果字符串
  std::string result = "--cpp-httplib-multipart-data-";

  // 生成随机字符串
  for (auto i = 0; i < 16; i++) {
    result += data[engine() % (sizeof(data) - 1)];
  }

  // 返回生成的随机字符串
  return result;
}

// 检查 multipart 边界字符串是否有效
inline bool is_multipart_boundary_chars_valid(const std::string &boundary) {
  // 初始化有效标志
  auto valid = true;
  // 遍历边界字符串的每个字符
  for (size_t i = 0; i < boundary.size(); i++) {
    auto c = boundary[i];
    // 检查字符是否为字母、数字、'-' 或 '_'
    if (!std::isalnum(c) && c != '-' && c != '_') {
      valid = false;
      break;
    }
  }
  // 返回边界字符串是否有效的结果
  return valid;
}

// 序列化 multipart 表单数据项的起始部分
template <typename T>
inline std::string
serialize_multipart_formdata_item_begin(const T &item,
                                        const std::string &boundary) {
  // 初始化字符串
  std::string body = "--" + boundary + "\r\n";
  body += "Content-Disposition: form-data; name=\"" + item.name + "\"";
  // 如果存在文件名，则添加到 Content-Disposition 中
  if (!item.filename.empty()) {
    body += "; filename=\"" + item.filename + "\"";
  }
  body += "\r\n";
  // 如果存在内容类型，则添加到字符串中
  if (!item.content_type.empty()) {
    body += "Content-Type: " + item.content_type + "\r\n";
  }
  body += "\r\n";

  // 返回序列化后的字符串
  return body;
}

// 序列化 multipart 表单数据项的结束部分
inline std::string serialize_multipart_formdata_item_end() { return "\r\n"; }

// 完成 multipart 表单数据的序列化
inline std::string
serialize_multipart_formdata_finish(const std::string &boundary) {
  // 返回 multipart 数据的结束标志
  return "--" + boundary + "--\r\n";
}

// 开始下一个 C++ 代码块
// 返回包含指定边界的 multipart/form-data 内容类型字符串
serialize_multipart_formdata_get_content_type(const std::string &boundary) {
  return "multipart/form-data; boundary=" + boundary;
}

// 序列化 multipart/form-data 内容
inline std::string
serialize_multipart_formdata(const MultipartFormDataItems &items,
                             const std::string &boundary, bool finish = true) {
  std::string body;

  // 遍历 multipart/form-data 项，序列化每一项的开始部分和内容，拼接到 body 中
  for (const auto &item : items) {
    body += serialize_multipart_formdata_item_begin(item, boundary);
    body += item.content + serialize_multipart_formdata_item_end();
  }

  // 如果需要结束标记，拼接结束标记到 body 中
  if (finish) body += serialize_multipart_formdata_finish(boundary);

  return body;
}

// 获取请求范围的偏移量和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, size_t content_length,
                            size_t index) {
  auto r = req.ranges[index];

  // 如果范围未指定，则返回整个内容的偏移量和长度
  if (r.first == -1 && r.second == -1) {
    return std::make_pair(0, content_length);
  }

  auto slen = static_cast<ssize_t>(content_length);

  // 如果起始位置未指定，则计算起始位置和结束位置
  if (r.first == -1) {
    r.first = (std::max)(static_cast<ssize_t>(0), slen - r.second);
    r.second = slen - 1;
  }

  // 如果结束位置未指定，则设置结束位置为内容长度减一
  if (r.second == -1) { r.second = slen - 1; }
  return std::make_pair(r.first, static_cast<size_t>(r.second - r.first) + 1);
}

// 生成 Content-Range 头字段
inline std::string
make_content_range_header_field(const std::pair<ssize_t, ssize_t> &range,
                                size_t content_length) {
  std::string field = "bytes ";
  if (range.first != -1) { field += std::to_string(range.first); }
  field += "-";
  if (range.second != -1) { field += std::to_string(range.second); }
  field += "/";
  field += std::to_string(content_length);
  return field;
}

template <typename SToken, typename CToken, typename Content>
// 处理多部分范围数据，根据请求和响应生成多部分范围数据，返回是否成功
bool process_multipart_ranges_data(const Request &req, Response &res,
                                   const std::string &boundary,
                                   const std::string &content_type,
                                   SToken stoken, CToken ctoken,
                                   Content content) {
  // 遍历请求中的范围
  for (size_t i = 0; i < req.ranges.size(); i++) {
    // 添加边界标记
    ctoken("--");
    stoken(boundary);
    ctoken("\r\n");
    // 如果内容类型不为空，添加内容类型
    if (!content_type.empty()) {
      ctoken("Content-Type: ");
      stoken(content_type);
      ctoken("\r\n");
    }

    // 添加内容范围
    ctoken("Content-Range: ");
    const auto &range = req.ranges[i];
    stoken(make_content_range_header_field(range, res.content_length_));
    ctoken("\r\n");
    ctoken("\r\n");

    // 获取范围的偏移和长度
    auto offsets = get_range_offset_and_length(req, res.content_length_, i);
    auto offset = offsets.first;
    auto length = offsets.second;
    // 如果内容函数返回 false，返回失败
    if (!content(offset, length)) { return false; }
    ctoken("\r\n");
  }

  // 添加结束标记
  ctoken("--");
  stoken(boundary);
  ctoken("--");

  return true;
}

// 生成多部分范围数据，根据请求和响应生成多部分范围数据，存储在 data 中，返回是否成功
inline bool make_multipart_ranges_data(const Request &req, Response &res,
                                       const std::string &boundary,
                                       const std::string &content_type,
                                       std::string &data) {
  // 调用处理多部分范围数据的函数，将结果存储在 data 中
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
      // 匿名函数，将 token 添加到 data 中
      [&](const std::string &token) { data += token; },
      // 匿名函数，将 token 添加到 data 中
      [&](const std::string &token) { data += token; },
      // 匿名函数，根据偏移和长度将响应体的部分添加到 data 中
      [&](size_t offset, size_t length) {
        if (offset < res.body.size()) {
          data += res.body.substr(offset, length);
          return true;
        }
        return false;
      });
}

inline size_t
// 计算多部分范围数据的总长度
get_multipart_ranges_data_length(const Request &req, Response &res,
                                 const std::string &boundary,
                                 const std::string &content_type) {
  // 初始化数据长度为0
  size_t data_length = 0;

  // 处理多部分范围数据，根据不同情况更新数据长度
  process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { data_length += token.size(); },
      [&](const std::string &token) { data_length += token.size(); },
      [&](size_t /*offset*/, size_t length) {
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
  // 处理多部分范围数据，根据不同情况写入数据
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { strm.write(token); },
      [&](const std::string &token) { strm.write(token); },
      [&](size_t offset, size_t length) {
        return write_content(strm, res.content_provider_, offset, length,
                             is_shutting_down);
      });
}

// 获取范围偏移和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, const Response &res,
                            size_t index) {
  // 获取请求中指定索引的范围
  auto r = req.ranges[index];

  // 如果范围的结束位置为-1，则设置为内容长度减1
  if (r.second == -1) {
    r.second = static_cast<ssize_t>(res.content_length_) - 1;
  }

  // 返回范围的偏移和长度
  return std::make_pair(r.first, r.second - r.first + 1);
}

// 检查请求是否包含内容
inline bool expect_content(const Request &req) {
  // 如果请求方法为POST、PUT、PATCH、PRI或DELETE，则返回true
  if (req.method == "POST" || req.method == "PUT" || req.method == "PATCH" ||
      req.method == "PRI" || req.method == "DELETE") {
    return true;
  }
  // TODO: 检查Content-Length是否设置
  return false;
}

// 检查字符串是否包含回车换行符
inline bool has_crlf(const std::string &s) {
  // 获取字符串的C风格字符数组
  auto p = s.c_str();
  // 遍历字符数组，查找回车换行符
  while (*p) {
    // 如果指针指向的字符是回车或换行符，则返回true
    if (*p == '\r' || *p == '\n') { return true; }
    // 指针向后移动一个位置
    p++;
  }
  // 如果没有找到回车或换行符，则返回false
  return false;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 如果支持 OpenSSL，则定义一个计算消息摘要的函数
inline std::string message_digest(const std::string &s, const EVP_MD *algo) {
  // 创建一个 EVP_MD_CTX 对象，并使用自定义的删除器进行管理
  auto context = std::unique_ptr<EVP_MD_CTX, decltype(&EVP_MD_CTX_free)>(
      EVP_MD_CTX_new(), EVP_MD_CTX_free);

  // 初始化哈希长度和哈希值数组
  unsigned int hash_length = 0;
  unsigned char hash[EVP_MAX_MD_SIZE];

  // 使用指定算法初始化哈希上下文
  EVP_DigestInit_ex(context.get(), algo, nullptr);
  // 更新哈希上下文的数据
  EVP_DigestUpdate(context.get(), s.c_str(), s.size());
  // 完成哈希计算，获取哈希值和长度
  EVP_DigestFinal_ex(context.get(), hash, &hash_length);

  // 将哈希值转换为十六进制字符串
  std::stringstream ss;
  for (auto i = 0u; i < hash_length; ++i) {
    ss << std::hex << std::setw(2) << std::setfill('0')
       << static_cast<unsigned int>(hash[i]);
  }

  // 返回计算得到的消息摘要字符串
  return ss.str();
}

// 定义计算 MD5 摘要的函数
inline std::string MD5(const std::string &s) {
  return message_digest(s, EVP_md5());
}

// 定义计算 SHA-256 摘要的函数
inline std::string SHA_256(const std::string &s) {
  return message_digest(s, EVP_sha256());
}

// 定义计算 SHA-512 摘要的函数
inline std::string SHA_512(const std::string &s) {
  return message_digest(s, EVP_sha512());
}
#endif
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef _WIN32
// 如果在 Windows 平台上，并且支持 OpenSSL，则加载系统证书存储中的证书
// 代码来源于以下 stackoverflow 帖子：https://stackoverflow.com/questions/9507184/can-openssl-on-windows-use-the-system-certificate-store
inline bool load_system_certs_on_windows(X509_STORE *store) {
  // 打开 ROOT 系统证书存储
  auto hStore = CertOpenSystemStoreW((HCRYPTPROV_LEGACY)NULL, L"ROOT");
  if (!hStore) { return false; }

  auto result = false;
  PCCERT_CONTEXT pContext = NULL;
  // 枚举存储中的证书，并将其添加到 X509_STORE 中
  while ((pContext = CertEnumCertificatesInStore(hStore, pContext)) !=
         nullptr) {
    auto encoded_cert =
        static_cast<const unsigned char *>(pContext->pbCertEncoded);

    auto x509 = d2i_X509(NULL, &encoded_cert, pContext->cbCertEncoded);
    if (x509) {
      X509_STORE_add_cert(store, x509);
      X509_free(x509);
      result = true;
    }
  }

  // 释放证书上下文和关闭证书存储
  CertFreeCertificateContext(pContext);
  CertCloseStore(hStore, 0);

  // 返回加载结果
  return result;
}
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#if TARGET_OS_OSX
// 如果在 macOS 平台上，并且定义了使用 macOS 密钥链中的证书，则使用 CFObjectPtr 模板
template <typename T>
using CFObjectPtr =
    // 使用 std::unique_ptr 智能指针，指向 T 类型的对象，同时指定了自定义的删除器函数
    std::unique_ptr<typename std::remove_pointer<T>::type, void (*)(CFTypeRef)>;
// 定义一个内联函数，用于释放 Core Foundation 对象
inline void cf_object_ptr_deleter(CFTypeRef obj) {
  // 如果对象存在，则释放对象
  if (obj) { CFRelease(obj); }
}

// 从钥匙串中检索证书
inline bool retrieve_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 定义查询的键和值
  CFStringRef keys[] = {kSecClass, kSecMatchLimit, kSecReturnRef};
  CFTypeRef values[] = {kSecClassCertificate, kSecMatchLimitAll, kCFBooleanTrue};

  // 创建查询字典
  CFObjectPtr<CFDictionaryRef> query(
      CFDictionaryCreate(nullptr, reinterpret_cast<const void **>(keys), values,
                         sizeof(keys) / sizeof(keys[0]),
                         &kCFTypeDictionaryKeyCallBacks,
                         &kCFTypeDictionaryValueCallBacks),
      cf_object_ptr_deleter);

  // 如果查询字典为空，则返回 false
  if (!query) { return false; }

  // 定义安全项变量
  CFTypeRef security_items = nullptr;
  // 从查询中获取匹配的安全项
  if (SecItemCopyMatching(query.get(), &security_items) != errSecSuccess ||
      CFArrayGetTypeID() != CFGetTypeID(security_items)) {
    return false;
  }

  // 重置证书数组
  certs.reset(reinterpret_cast<CFArrayRef>(security_items));
  return true;
}

// 从钥匙串中检索根证书
inline bool retrieve_root_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 定义根证书安全项变量
  CFArrayRef root_security_items = nullptr;
  // 从信任中心复制锚点证书
  if (SecTrustCopyAnchorCertificates(&root_security_items) != errSecSuccess) {
    return false;
  }

  // 重置证书数组
  certs.reset(root_security_items);
  return true;
}

// 将证书添加到 X509 存储中
inline bool add_certs_to_x509_store(CFArrayRef certs, X509_STORE *store) {
  // 初始化结果为 false
  auto result = false;
  // 遍历证书数组
  for (auto i = 0; i < CFArrayGetCount(certs); ++i) {
    // 获取证书对象
    const auto cert = reinterpret_cast<const __SecCertificate *>(
        CFArrayGetValueAtIndex(certs, i));

    // 如果证书对象类型不匹配，则继续下一次循环
    if (SecCertificateGetTypeID() != CFGetTypeID(cert)) { continue; }

    // 定义证书数据变量
    CFDataRef cert_data = nullptr;
    // 导出证书数据
    if (SecItemExport(cert, kSecFormatX509Cert, 0, nullptr, &cert_data) !=
        errSecSuccess) {
      continue;
    }

    // 创建证书数据指针
    CFObjectPtr<CFDataRef> cert_data_ptr(cert_data, cf_object_ptr_deleter);

    // 获取编码后的证书数据
    auto encoded_cert = static_cast<const unsigned char *>(
        CFDataGetBytePtr(cert_data_ptr.get()));
    // 使用 d2i_X509 函数将编码后的证书数据解析为 X509 结构体
    auto x509 =
        d2i_X509(NULL, &encoded_cert, CFDataGetLength(cert_data_ptr.get()));

    // 如果成功解析出 X509 结构体
    if (x509) {
      // 将 X509 结构体添加到 X509_STORE 中
      X509_STORE_add_cert(store, x509);
      // 释放 X509 结构体的内存
      X509_free(x509);
      // 设置结果为 true
      result = true;
    }
  }

  // 返回结果
  return result;
}

// 在 macOS 上加载系统证书到 X509_STORE 中
inline bool load_system_certs_on_macos(X509_STORE *store) {
  auto result = false;
  // 从钥匙串中检索证书
  CFObjectPtr<CFArrayRef> certs(nullptr, cf_object_ptr_deleter);
  if (retrieve_certs_from_keychain(certs) && certs) {
    // 将证书添加到 X509_STORE 中
    result = add_certs_to_x509_store(certs.get(), store);
  }

  if (retrieve_root_certs_from_keychain(certs) && certs) {
    // 将根证书添加到 X509_STORE 中
    result = add_certs_to_x509_store(certs.get(), store) || result;
  }

  return result;
}
#endif // TARGET_OS_OSX
#endif // _WIN32
#endif // CPPHTTPLIB_OPENSSL_SUPPORT

#ifdef _WIN32
// Windows Socket 初始化类
class WSInit {
public:
  WSInit() {
    WSADATA wsaData;
    // 初始化 Windows Socket
    if (WSAStartup(0x0002, &wsaData) == 0) is_valid_ = true;
  }

  ~WSInit() {
    // 清理 Windows Socket
    if (is_valid_) WSACleanup();
  }

  bool is_valid_ = false;
};

// 静态的 Windows Socket 初始化对象
static WSInit wsinit_;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 生成摘要认证头部
inline std::pair<std::string, std::string> make_digest_authentication_header(
    const Request &req, const std::map<std::string, std::string> &auth,
    size_t cnonce_count, const std::string &cnonce, const std::string &username,
    const std::string &password, bool is_proxy = false) {
  std::string nc;
  {
    // 生成 cnonce_count
    std::stringstream ss;
    ss << std::setfill('0') << std::setw(8) << std::hex << cnonce_count;
    nc = ss.str();
  }

  std::string qop;
  if (auth.find("qop") != auth.end()) {
    qop = auth.at("qop");
    // 确定 qop 类型
    if (qop.find("auth-int") != std::string::npos) {
      qop = "auth-int";
    } else if (qop.find("auth") != std::string::npos) {
      qop = "auth";
    } else {
      qop.clear();
    }
  }

  std::string algo = "MD5";
  if (auth.find("algorithm") != auth.end()) { algo = auth.at("algorithm"); }

  std::string response;
  {
    // 根据算法选择哈希函数
    auto H = algo == "SHA-256"   ? detail::SHA_256
             : algo == "SHA-512" ? detail::SHA_512
                                 : detail::MD5;

    auto A1 = username + ":" + auth.at("realm") + ":" + password;

    auto A2 = req.method + ":" + req.path;
    if (qop == "auth-int") { A2 += ":" + H(req.body); }
    // 如果 qop 为空，则计算 response 值为 H(A1) + ":" + auth["nonce"] + ":" + H(A2)
    if (qop.empty()) {
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + H(A2));
    } else {
      // 如果 qop 不为空，则计算 response 值为 H(A1) + ":" + auth["nonce"] + ":" + nc + ":" + cnonce + ":" + qop + ":" + H(A2)
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + nc + ":" + cnonce +
                   ":" + qop + ":" + H(A2));
    }
  }

  // 检查是否存在 opaque 字段，如果存在则赋值给 opaque 变量，否则为空字符串
  auto opaque = (auth.find("opaque") != auth.end()) ? auth.at("opaque") : "";

  // 拼接 Authorization 字段的值
  auto field = "Digest username=\"" + username + "\", realm=\"" +
               auth.at("realm") + "\", nonce=\"" + auth.at("nonce") +
               "\", uri=\"" + req.path + "\", algorithm=" + algo +
               (qop.empty() ? ", response=\""
                            : ", qop=" + qop + ", nc=" + nc + ", cnonce=\"" +
                                  cnonce + "\", response=\"") +
               response + "\"" +
               (opaque.empty() ? "" : ", opaque=\"" + opaque + "\"");

  // 根据是否为代理服务器，选择设置 key 为 "Proxy-Authorization" 或 "Authorization"
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  // 返回 key 和 field 组成的键值对
  return std::make_pair(key, field);
// 结束条件编译指令，用于条件编译
#endif

// 解析 HTTP 响应中的认证信息，将认证信息存储在 auth 字典中，is_proxy 用于指示是否为代理认证
inline bool parse_www_authenticate(const Response &res,
                                   std::map<std::string, std::string> &auth,
                                   bool is_proxy) {
  // 根据是否为代理认证选择对应的认证头字段
  auto auth_key = is_proxy ? "Proxy-Authenticate" : "WWW-Authenticate";
  // 检查响应中是否包含指定的认证头字段
  if (res.has_header(auth_key)) {
    // 定义正则表达式用于解析认证信息
    static auto re = std::regex(R"~((?:(?:,\s*)?(.+?)=(?:"(.*?)"|([^,]*))))~");
    // 获取指定认证头字段的值
    auto s = res.get_header_value(auth_key);
    // 查找空格位置，分离认证类型和认证信息
    auto pos = s.find(' ');
    if (pos != std::string::npos) {
      auto type = s.substr(0, pos);
      // 根据认证类型进行处理
      if (type == "Basic") {
        return false;
      } else if (type == "Digest") {
        // 提取认证信息部分
        s = s.substr(pos + 1);
        // 使用正则表达式迭代解析认证信息
        auto beg = std::sregex_iterator(s.begin(), s.end(), re);
        for (auto i = beg; i != std::sregex_iterator(); ++i) {
          const auto &m = *i;
          // 提取键和值，并存储在 auth 字典中
          auto key = s.substr(static_cast<size_t>(m.position(1)),
                              static_cast<size_t>(m.length(1)));
          auto val = m.length(2) > 0
                         ? s.substr(static_cast<size_t>(m.position(2)),
                                    static_cast<size_t>(m.length(2)))
                         : s.substr(static_cast<size_t>(m.position(3)),
                                    static_cast<size_t>(m.length(3)));
          auth[key] = val;
        }
        return true;
      }
    }
  }
  return false;
}

// 生成指定长度的随机字符串
// 参考：https://stackoverflow.com/questions/440133/how-do-i-create-a-random-alpha-numeric-string-in-c/440240#answer-440240
inline std::string random_string(size_t length) {
  // 定义 lambda 函数用于生成随机字符
  auto randchar = []() -> char {
    const char charset[] = "0123456789"
                           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
                           "abcdefghijklmnopqrstuvwxyz";
    const size_t max_index = (sizeof(charset) - 1);
    return charset[static_cast<size_t>(std::rand()) % max_index];
  };
  // 生成指定长度的随机字符串
  std::string str(length, 0);
  std::generate_n(str.begin(), length, randchar);
  return str;
}

// 定义 ContentProviderAdapter 类
class ContentProviderAdapter {
// 定义一个公共的类 ContentProviderAdapter，接受一个 ContentProviderWithoutLength 类型的参数，并初始化成员变量 content_provider_
public:
  explicit ContentProviderAdapter(
      ContentProviderWithoutLength &&content_provider)
      : content_provider_(content_provider) {}

  // 重载 () 运算符，接受 offset、size_t 类型的参数和一个 DataSink 类型的引用参数，调用 content_provider_ 的 () 运算符
  bool operator()(size_t offset, size_t, DataSink &sink) {
    return content_provider_(offset, sink);
  }

private:
  // 声明一个 ContentProviderWithoutLength 类型的成员变量 content_provider_
  ContentProviderWithoutLength content_provider_;
};

} // namespace detail

// 定义一个函数 hosted_at，接受一个 const std::string & 类型的参数 hostname
inline std::string hosted_at(const std::string &hostname) {
  // 声明一个空的字符串向量 addrs
  std::vector<std::string> addrs;
  // 调用另一个 hosted_at 函数，传入 hostname 和 addrs 作为参数
  hosted_at(hostname, addrs);
  // 如果 addrs 为空，返回一个空字符串
  if (addrs.empty()) { return std::string(); }
  // 返回 addrs 中的第一个元素
  return addrs[0];
}

// 定义一个函数 hosted_at，接受一个 const std::string & 类型的参数 hostname 和一个 std::vector<std::string> & 类型的参数 addrs
inline void hosted_at(const std::string &hostname,
                      std::vector<std::string> &addrs) {
  // 声明 addrinfo 结构体 hints 和指针 result
  struct addrinfo hints;
  struct addrinfo *result;

  // 将 hints 的内存清零
  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_family = AF_UNSPEC;
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_protocol = 0;

  // 调用 getaddrinfo 函数，获取 hostname 对应的地址信息，存储在 result 中
  if (getaddrinfo(hostname.c_str(), nullptr, &hints, &result)) {
#if defined __linux__ && !defined __ANDROID__
    res_init();
#endif
    return;
  }

  // 遍历 result 中的地址信息，获取 IP 地址并存储在 addrs 中
  for (auto rp = result; rp; rp = rp->ai_next) {
    const auto &addr =
        *reinterpret_cast<struct sockaddr_storage *>(rp->ai_addr);
    std::string ip;
    auto dummy = -1;
    if (detail::get_ip_and_port(addr, sizeof(struct sockaddr_storage), ip,
                                dummy)) {
      addrs.push_back(ip);
    }
  }

  // 释放 result 的内存
  freeaddrinfo(result);
}

// 定义一个函数 append_query_params，接受一个 const std::string & 类型的参数 path 和一个 Params 类型的参数 params
inline std::string append_query_params(const std::string &path,
                                       const Params &params) {
  // 将 path 赋值给 path_with_query
  std::string path_with_query = path;
  // 定义一个静态的正则表达式 re，用于匹配 path 是否已经包含查询参数
  const static std::regex re("[^?]+\\?.*");
  // 根据 path 是否已经包含查询参数，选择连接符 delm
  auto delm = std::regex_match(path, re) ? '&' : '?';
  // 将 params 转换成查询字符串，拼接到 path_with_query 后面
  path_with_query += delm + detail::params_to_query_str(params);
  // 返回拼接后的字符串
  return path_with_query;
}

// 定义一个函数 make_range_header，接受一个 Ranges 类型的参数 ranges
inline std::pair<std::string, std::string> make_range_header(Ranges ranges) {
  // 初始化一个字符串 field 为 "bytes="
  std::string field = "bytes=";
  // 初始化一个计数器 i 为 0，遍历 ranges
  auto i = 0;
  for (auto r : ranges) {
    // 如果不是第一个范围，添加逗号和空格
    if (i != 0) { field += ", "; }
    // 如果范围的起始位置不为 -1，添加起始位置到 field
    if (r.first != -1) { field += std::to_string(r.first); }
    field += '-';
    // 如果第二个值不为-1，则将其转换为字符串并添加到字段中
    if (r.second != -1) { field += std::to_string(r.second); }
    // 增加索引值
    i++;
  }
  // 返回一个包含"Range"和字段的键值对
  return std::make_pair("Range", std::move(field));
// 创建基本身份验证头部，包含用户名和密码，返回键值对
inline std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password, bool is_proxy) {
  // 拼接用户名和密码，进行 base64 编码，生成 Basic 认证字段
  auto field = "Basic " + detail::base64_encode(username + ":" + password);
  // 根据是否代理设置键值
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  return std::make_pair(key, std::move(field));
}

// 创建 Bearer 令牌身份验证头部，包含令牌信息，返回键值对
inline std::pair<std::string, std::string>
make_bearer_token_authentication_header(const std::string &token,
                                        bool is_proxy = false) {
  // 拼接令牌信息，生成 Bearer 认证字段
  auto field = "Bearer " + token;
  // 根据是否代理设置键值
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  return std::make_pair(key, std::move(field));
}

// 请求实现
// 检查是否存在指定键的头部
inline bool Request::has_header(const std::string &key) const {
  return detail::has_header(headers, key);
}

// 获取指定键的头部值，可以指定多个值中的一个
inline std::string Request::get_header_value(const std::string &key,
                                             size_t id) const {
  return detail::get_header_value(headers, key, id, "");
}

// 获取指定键的头部值的数量
inline size_t Request::get_header_value_count(const std::string &key) const {
  auto r = headers.equal_range(key);
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置指定键的头部值
inline void Request::set_header(const std::string &key,
                                const std::string &val) {
  // 检查键和值是否包含换行符，如果不包含则添加到头部
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    headers.emplace(key, val);
  }
}

// 检查是否存在指定键的参数
inline bool Request::has_param(const std::string &key) const {
  return params.find(key) != params.end();
}

// 获取指定键的参数值，可以指定多个值中的一个
inline std::string Request::get_param_value(const std::string &key,
                                            size_t id) const {
  auto rng = params.equal_range(key);
  auto it = rng.first;
  std::advance(it, static_cast<ssize_t>(id));
  if (it != rng.second) { return it->second; }
  return std::string();
}
// 返回给定键对应的参数值的数量
inline size_t Request::get_param_value_count(const std::string &key) const {
  // 查找参数中键对应的范围
  auto r = params.equal_range(key);
  // 返回范围内元素的数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 检查请求是否为多部分表单数据
inline bool Request::is_multipart_form_data() const {
  // 获取请求头中的 Content-Type
  const auto &content_type = get_header_value("Content-Type");
  // 检查 Content-Type 是否以 "multipart/form-data" 开头
  return !content_type.rfind("multipart/form-data", 0);
}

// 检查是否存在给定键对应的文件
inline bool Request::has_file(const std::string &key) const {
  // 在文件映射中查找给定键
  return files.find(key) != files.end();
}

// 获取给定键对应的文件值
inline MultipartFormData Request::get_file_value(const std::string &key) const {
  // 在文件映射中查找给定键
  auto it = files.find(key);
  // 如果找到对应的文件值，则返回该值
  if (it != files.end()) { return it->second; }
  // 否则返回空的 MultipartFormData 对象
  return MultipartFormData();
}

// 获取给定键对应的所有文件值
inline std::vector<MultipartFormData>
Request::get_file_values(const std::string &key) const {
  // 存储所有文件值的向量
  std::vector<MultipartFormData> values;
  // 获取文件映射中给定键对应的范围
  auto rng = files.equal_range(key);
  // 遍历范围内的所有文件值，并添加到向量中
  for (auto it = rng.first; it != rng.second; it++) {
    values.push_back(it->second);
  }
  // 返回所有文件值的向量
  return values;
}

// 检查是否存在给定键对应的响应头
inline bool Response::has_header(const std::string &key) const {
  // 在响应头映射中查找给定键
  return headers.find(key) != headers.end();
}

// 获取给定键对应的响应头值
inline std::string Response::get_header_value(const std::string &key,
                                              size_t id) const {
  // 调用辅助函数获取给定键对应的值
  return detail::get_header_value(headers, key, id, "");
}

// 返回给定键对应的响应头值的数量
inline size_t Response::get_header_value_count(const std::string &key) const {
  // 获取响应头映射中给定键对应的范围
  auto r = headers.equal_range(key);
  // 返回范围内元素的数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置响应头的键值对
inline void Response::set_header(const std::string &key,
                                 const std::string &val) {
  // 检查键和值是否不包含换行符
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    // 添加键值对到响应头映射中
    headers.emplace(key, val);
  }
}

// 设置重定向响应
inline void Response::set_redirect(const std::string &url, int stat) {
  // 检查 URL 是否不包含换行符
  if (!detail::has_crlf(url)) {
    // 设置 Location 响应头为给定 URL
    set_header("Location", url);
    // 根据状态码设置响应状态
    if (300 <= stat && stat < 400) {
      this->status = stat;
    } else {
      this->status = 302;
    }
  }
}
// 设置响应内容，接受字符指针和长度，以及内容类型
inline void Response::set_content(const char *s, size_t n,
                                  const std::string &content_type) {
  // 将字符指针和长度赋值给响应体
  body.assign(s, n);

  // 查找并删除已存在的"Content-Type"头部
  auto rng = headers.equal_range("Content-Type");
  headers.erase(rng.first, rng.second);
  // 设置新的"Content-Type"头部
  set_header("Content-Type", content_type);
}

// 设置响应内容，接受字符串和内容类型
inline void Response::set_content(const std::string &s,
                                  const std::string &content_type) {
  // 调用上一个函数，传入字符串的数据和长度
  set_content(s.data(), s.size(), content_type);
}

// 设置内容提供者，接受内容长度、内容类型、内容提供者和资源释放器
inline void Response::set_content_provider(
    size_t in_length, const std::string &content_type, ContentProvider provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置"Content-Type"头部
  set_header("Content-Type", content_type);
  // 设置内容长度
  content_length_ = in_length;
  // 如果内容长度大于0，则移动内容提供者
  if (in_length > 0) { content_provider_ = std::move(provider); }
  // 设置资源释放器
  content_provider_resource_releaser_ = resource_releaser;
  // 设置是否为分块内容提供者
  is_chunked_content_provider_ = false;
}

// 设置内容提供者，接受内容类型、内容提供者和资源释放器
inline void Response::set_content_provider(
    const std::string &content_type, ContentProviderWithoutLength provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置"Content-Type"头部
  set_header("Content-Type", content_type);
  // 设置内容长度为0
  content_length_ = 0;
  // 移动内容提供者
  content_provider_ = detail::ContentProviderAdapter(std::move(provider));
  // 设置资源释放器
  content_provider_resource_releaser_ = resource_releaser;
  // 设置是否为分块内容提供者
  is_chunked_content_provider_ = false;
}

// 设置分块内容提供者，接受内容类型、内容提供者和资源释放器
inline void Response::set_chunked_content_provider(
    const std::string &content_type, ContentProviderWithoutLength provider,
    ContentProviderResourceReleaser resource_releaser) {
  // 设置"Content-Type"头部
  set_header("Content-Type", content_type);
  // 设置内容长度为0
  content_length_ = 0;
  // 移动内容提供者
  content_provider_ = detail::ContentProviderAdapter(std::move(provider));
  // 设置资源释放器
  content_provider_resource_releaser_ = resource_releaser;
  // 设置为分块内容提供者
  is_chunked_content_provider_ = true;
}

// 检查请求头中是否存在指定键的请求头
// 返回布尔值
inline bool Result::has_request_header(const std::string &key) const {
  // 查找请求头中是否存在指定键，返回结果
  return request_headers_.find(key) != request_headers_.end();
}
// 获取请求头中指定键的值
inline std::string Result::get_request_header_value(const std::string &key,
                                                    size_t id) const {
  // 调用 detail 命名空间中的函数获取请求头中指定键的值
  return detail::get_header_value(request_headers_, key, id, "");
}

// 获取请求头中指定键的值的数量
inline size_t
Result::get_request_header_value_count(const std::string &key) const {
  // 获取请求头中指定键的值的范围
  auto r = request_headers_.equal_range(key);
  // 返回请求头中指定键的值的数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 写入字符数组到流中
inline ssize_t Stream::write(const char *ptr) {
  // 调用重载函数，写入字符数组到流中
  return write(ptr, strlen(ptr));
}

// 写入字符串到流中
inline ssize_t Stream::write(const std::string &s) {
  // 调用重载函数，写入字符串到流中
  return write(s.data(), s.size());
}

namespace detail {

// SocketStream 类的构造函数
inline SocketStream::SocketStream(socket_t sock, time_t read_timeout_sec,
                                  time_t read_timeout_usec,
                                  time_t write_timeout_sec,
                                  time_t write_timeout_usec)
    : sock_(sock), read_timeout_sec_(read_timeout_sec),
      read_timeout_usec_(read_timeout_usec),
      write_timeout_sec_(write_timeout_sec),
      write_timeout_usec_(write_timeout_usec), read_buff_(read_buff_size_, 0) {}

// SocketStream 类的析构函数
inline SocketStream::~SocketStream() {}

// 检查流是否可读
inline bool SocketStream::is_readable() const {
  // 使用 select 函数检查流是否可读
  return select_read(sock_, read_timeout_sec_, read_timeout_usec_) > 0;
}

// 检查流是否可写
inline bool SocketStream::is_writable() const {
  // 使用 select 函数检查流是否可写，并且检查套接字是否存活
  return select_write(sock_, write_timeout_sec_, write_timeout_usec_) > 0 &&
         is_socket_alive(sock_);
}

// 从流中读取数据到字符数组中
inline ssize_t SocketStream::read(char *ptr, size_t size) {
#ifdef _WIN32
  // 在 Windows 平台下限制读取的大小
  size =
      (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#else
  // 在非 Windows 平台下限制读取的大小
  size = (std::min)(size,
                    static_cast<size_t>((std::numeric_limits<ssize_t>::max)()));
#endif

  // 如果读取缓冲区中还有数据
  if (read_buff_off_ < read_buff_content_size_) {
    // 计算剩余可读取的数据大小
    auto remaining_size = read_buff_content_size_ - read_buff_off_;
    // 如果请求的数据大小小于等于剩余数据大小
    if (size <= remaining_size) {
      // 将 read_buff_ 中的数据拷贝到 ptr 中，并更新 read_buff_off_
      memcpy(ptr, read_buff_.data() + read_buff_off_, size);
      read_buff_off_ += size;
      // 返回实际拷贝的数据大小
      return static_cast<ssize_t>(size);
    } else {
      // 如果请求的数据大小大于剩余数据大小
      // 将 read_buff_ 中的部分数据拷贝到 ptr 中，并更新 read_buff_off_
      memcpy(ptr, read_buff_.data() + read_buff_off_, remaining_size);
      read_buff_off_ += remaining_size;
      // 返回实际拷贝的数据大小
      return static_cast<ssize_t>(remaining_size);
    }
  }

  // 如果不可读，则返回 -1
  if (!is_readable()) { return -1; }

  // 重置 read_buff_off_ 和 read_buff_content_size_
  read_buff_off_ = 0;
  read_buff_content_size_ = 0;

  // 如果请求的数据大小小于 read_buff_size_
  if (size < read_buff_size_) {
    // 从 socket 中读取数据到 read_buff_ 中
    auto n = read_socket(sock_, read_buff_.data(), read_buff_size_,
                         CPPHTTPLIB_RECV_FLAGS);
    // 如果读取失败，则返回错误码
    if (n <= 0) {
      return n;
    } else if (n <= static_cast<ssize_t>(size)) {
      // 如果读取的数据大小小于等于请求的数据大小，则将数据拷贝到 ptr 中
      memcpy(ptr, read_buff_.data(), static_cast<size_t>(n));
      return n;
    } else {
      // 如果读取的数据大小大于请求的数据大小
      // 将数据拷贝到 ptr 中，并更新 read_buff_off_ 和 read_buff_content_size_
      memcpy(ptr, read_buff_.data(), size);
      read_buff_off_ = size;
      read_buff_content_size_ = static_cast<size_t>(n);
      return static_cast<ssize_t>(size);
    }
  } else {
    // 如果请求的数据大小大于等于 read_buff_size_
    // 直接从 socket 中读取数据到 ptr 中
    return read_socket(sock_, ptr, size, CPPHTTPLIB_RECV_FLAGS);
  }
// 写入数据到套接字流，如果不可写则返回-1
inline ssize_t SocketStream::write(const char *ptr, size_t size) {
  if (!is_writable()) { return -1; }

#if defined(_WIN32) && !defined(_WIN64)
  // 如果是 Windows 32 位系统，限制写入大小为 int 最大值
  size = (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#endif

  // 发送数据到套接字
  return send_socket(sock_, ptr, size, CPPHTTPLIB_SEND_FLAGS);
}

// 获取远程 IP 地址和端口
inline void SocketStream::get_remote_ip_and_port(std::string &ip, int &port) const {
  // 调用 detail 命名空间的函数获取远程 IP 地址和端口
  return detail::get_remote_ip_and_port(sock_, ip, port);
}

// 获取本地 IP 地址和端口
inline void SocketStream::get_local_ip_and_port(std::string &ip, int &port) const {
  // 调用 detail 命名空间的函数获取本地 IP 地址和端口
  return detail::get_local_ip_and_port(sock_, ip, port);
}

// 返回套接字
inline socket_t SocketStream::socket() const { return sock_; }

// 缓冲流实现
// 判断是否可读
inline bool BufferStream::is_readable() const { return true; }

// 判断是否可写
inline bool BufferStream::is_writable() const { return true; }

// 从缓冲流中读取数据
inline ssize_t BufferStream::read(char *ptr, size_t size) {
#if defined(_MSC_VER) && _MSC_VER < 1910
  // 如果是旧版本的 Visual Studio，使用 _Copy_s 函数
  auto len_read = buffer._Copy_s(ptr, size, size, position);
#else
  // 否则使用 copy 函数
  auto len_read = buffer.copy(ptr, size, position);
#endif
  position += static_cast<size_t>(len_read);
  return static_cast<ssize_t>(len_read);
}

// 写入数据到缓冲流
inline ssize_t BufferStream::write(const char *ptr, size_t size) {
  // 将数据追加到缓冲区
  buffer.append(ptr, size);
  return static_cast<ssize_t>(size);
}

// 获取远程 IP 地址和端口（缓冲流不需要实现）
inline void BufferStream::get_remote_ip_and_port(std::string & /*ip*/, int & /*port*/) const {}

// 获取本地 IP 地址和端口（缓冲流不需要实现）
inline void BufferStream::get_local_ip_and_port(std::string & /*ip*/, int & /*port*/) const {}

// 返回套接字（缓冲流不需要实现）
inline socket_t BufferStream::socket() const { return 0; }

// 返回缓冲区内容
inline const std::string &BufferStream::get_buffer() const { return buffer; }

// 路径参数匹配器构造函数
inline PathParamsMatcher::PathParamsMatcher(const std::string &pattern) {
  // 路径参数子字符串的最后一个结束位置
  std::size_t last_param_end = 0;
#ifndef CPPHTTPLIB_NO_EXCEPTIONS
  // 如果异常未禁用，则需要确保在匹配器构造期间参数名是唯一的
  // 如果异常被禁用，只有最后一个重复的路径参数会被设置
  std::unordered_set<std::string> param_name_set;
#endif

  // 循环直到条件为真
  while (true) {
    // 查找下一个标记的位置
    const auto marker_pos = pattern.find(marker, last_param_end);
    // 如果找不到标记，则跳出循环
    if (marker_pos == std::string::npos) { break; }

    // 将静态片段添加到静态片段列表中
    static_fragments_.push_back(
        pattern.substr(last_param_end, marker_pos - last_param_end));

    // 获取参数名的起始位置
    const auto param_name_start = marker_pos + 1;

    // 查找参数名的分隔符位置
    auto sep_pos = pattern.find(separator, param_name_start);
    // 如果找不到分隔符，则设置为字符串的长度
    if (sep_pos == std::string::npos) { sep_pos = pattern.length(); }

    // 提取参数名
    auto param_name =
        pattern.substr(param_name_start, sep_pos - param_name_start);

#ifndef CPPHTTPLIB_NO_EXCEPTIONS
    // 如果参数名已经存在于集合中，则抛出异常
    if (param_name_set.find(param_name) != param_name_set.cend()) {
      std::string msg = "Encountered path parameter '" + param_name +
                        "' multiple times in route pattern '" + pattern + "'.";
      throw std::invalid_argument(msg);
    }
#endif

    // 将参数名添加到参数名列表中
    param_names_.push_back(std::move(param_name));

    // 更新上一个参数的结束位置
    last_param_end = sep_pos + 1;
  }

  // 如果最后一个参数的结束位置小于模式的长度，则将其添加到静态片段列表中
  if (last_param_end < pattern.length()) {
    static_fragments_.push_back(pattern.substr(last_param_end));
  }
}

// 匹配路径参数
inline bool PathParamsMatcher::match(Request &request) const {
  // 初始化匹配结果
  request.matches = std::smatch();
  // 清空路径参数
  request.path_params.clear();
  // 预留路径参数的空间
  request.path_params.reserve(param_names_.size());

  // 上次路径匹配模式的位置
  std::size_t starting_pos = 0;
  // 遍历静态片段列表
  for (size_t i = 0; i < static_fragments_.size(); ++i) {
    const auto &fragment = static_fragments_[i];

    // 如果路径长度不足以匹配当前片段，则返回false
    if (starting_pos + fragment.length() > request.path.length()) {
      return false;
    }

    // 使用strncmp而不是substr+比较，避免不必要的分配
    // 检查请求路径中从指定位置开始的片段是否与给定片段匹配，如果不匹配则返回 false
    if (std::strncmp(request.path.c_str() + starting_pos, fragment.c_str(),
                     fragment.length()) != 0) {
      return false;
    }

    // 更新起始位置，跳过已匹配的片段长度
    starting_pos += fragment.length();

    // 如果当前索引超出参数名称列表的大小，则跳过当前循环
    // 例如：'/users/:id/subscriptions'
    // 这里的 'subscriptions' 片段没有对应的参数
    if (i >= param_names_.size()) { continue; }

    // 查找路径中下一个分隔符的位置
    auto sep_pos = request.path.find(separator, starting_pos);
    // 如果找不到分隔符，则将其设置为路径的末尾
    if (sep_pos == std::string::npos) { sep_pos = request.path.length(); }

    // 获取当前参数的名称
    const auto &param_name = param_names_[i];

    // 将参数名和对应的值添加到请求的路径参数中
    request.path_params.emplace(
        param_name, request.path.substr(starting_pos, sep_pos - starting_pos));

    // 将起始位置更新为下一个片段的起始位置
    starting_pos = sep_pos + 1;
  }
  // 如果路径长度超过模式长度，则返回 false
  return starting_pos >= request.path.length();
} // 结束命名空间 detail

// HTTP 服务器实现
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

// 创建匹配器
inline std::unique_ptr<detail::MatcherBase>
Server::make_matcher(const std::string &pattern) {
  // 如果模式中包含 "/:"，则创建 PathParamsMatcher
  if (pattern.find("/:") != std::string::npos) {
    return detail::make_unique<detail::PathParamsMatcher>(pattern);
  } else {
    // 否则创建 RegexMatcher
    return detail::make_unique<detail::RegexMatcher>(pattern);
  }
}

// 处理 GET 请求
inline Server &Server::Get(const std::string &pattern, Handler handler) {
  // 添加 GET 请求处理器
  get_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 处理 POST 请求
inline Server &Server::Post(const std::string &pattern, Handler handler) {
  // 添加 POST 请求处理器
  post_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 处理 POST 请求（带内容读取器）
inline Server &Server::Post(const std::string &pattern,
                            HandlerWithContentReader handler) {
  // 添加带内容读取器的 POST 请求处理器
  post_handlers_for_content_reader_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 处理 PUT 请求
inline Server &Server::Put(const std::string &pattern, Handler handler) {
  // 添加 PUT 请求处理器
  put_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 处理 PUT 请求（带内容读取器）
inline Server &Server::Put(const std::string &pattern,
                           HandlerWithContentReader handler) {
  // 添加带内容读取器的 PUT 请求处理器
  put_handlers_for_content_reader_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 处理 PATCH 请求
inline Server &Server::Patch(const std::string &pattern, Handler handler) {
  // 添加 PATCH 请求处理器
  patch_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}
// 在服务器对象上注册一个 PATCH 请求处理程序，将匹配模式和处理程序添加到处理程序列表中
inline Server &Server::Patch(const std::string &pattern,
                             HandlerWithContentReader handler) {
  patch_handlers_for_content_reader_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 在服务器对象上注册一个 DELETE 请求处理程序，将匹配模式和处理程序添加到处理程序列表中
inline Server &Server::Delete(const std::string &pattern, Handler handler) {
  delete_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 在服务器对象上注册一个 DELETE 请求处理程序，将匹配模式和处理程序添加到处理程序列表中
inline Server &Server::Delete(const std::string &pattern,
                              HandlerWithContentReader handler) {
  delete_handlers_for_content_reader_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 在服务器对象上注册一个 OPTIONS 请求处理程序，将匹配模式和处理程序添加到处理程序列表中
inline Server &Server::Options(const std::string &pattern, Handler handler) {
  options_handlers_.push_back(
      std::make_pair(make_matcher(pattern), std::move(handler)));
  return *this;
}

// 设置服务器对象的基本目录，将其作为挂载点
inline bool Server::set_base_dir(const std::string &dir,
                                 const std::string &mount_point) {
  return set_mount_point(mount_point, dir);
}

// 设置服务器对象的挂载点，将挂载点、目录和头部信息添加到基本目录列表中
inline bool Server::set_mount_point(const std::string &mount_point,
                                    const std::string &dir, Headers headers) {
  if (detail::is_dir(dir)) {
    std::string mnt = !mount_point.empty() ? mount_point : "/";
    if (!mnt.empty() && mnt[0] == '/') {
      base_dirs_.push_back({mnt, dir, std::move(headers)});
      return true;
    }
  }
  return false;
}

// 移除服务器对象的挂载点，从基本目录列表中查找并删除指定挂载点
inline bool Server::remove_mount_point(const std::string &mount_point) {
  for (auto it = base_dirs_.begin(); it != base_dirs_.end(); ++it) {
    if (it->mount_point == mount_point) {
      base_dirs_.erase(it);
      return true;
    }
  }
  return false;
}

// 设置文件扩展名和 MIME 类型的映射关系，将扩展名和 MIME 类型添加到映射表中
inline Server &
Server::set_file_extension_and_mimetype_mapping(const std::string &ext,
                                                const std::string &mime) {
  file_extension_and_mimetype_map_[ext] = mime;
  return *this;
}
// 设置默认文件的 MIME 类型
inline Server &Server::set_default_file_mimetype(const std::string &mime) {
  default_file_mimetype_ = mime;
  return *this;
}

// 设置文件请求处理程序
inline Server &Server::set_file_request_handler(Handler handler) {
  file_request_handler_ = std::move(handler);
  return *this;
}

// 设置错误处理程序
inline Server &Server::set_error_handler(HandlerWithResponse handler) {
  error_handler_ = std::move(handler);
  return *this;
}

// 设置错误处理程序
inline Server &Server::set_error_handler(Handler handler) {
  error_handler_ = [handler](const Request &req, Response &res) {
    handler(req, res);
    return HandlerResponse::Handled;
  };
  return *this;
}

// 设置异常处理程序
inline Server &Server::set_exception_handler(ExceptionHandler handler) {
  exception_handler_ = std::move(handler);
  return *this;
}

// 设置预路由处理程序
inline Server &Server::set_pre_routing_handler(HandlerWithResponse handler) {
  pre_routing_handler_ = std::move(handler);
  return *this;
}

// 设置后路由处理程序
inline Server &Server::set_post_routing_handler(Handler handler) {
  post_routing_handler_ = std::move(handler);
  return *this;
}

// 设置日志记录器
inline Server &Server::set_logger(Logger logger) {
  logger_ = std::move(logger);
  return *this;
}

// 设置预期 100 继续处理程序
inline Server &
Server::set_expect_100_continue_handler(Expect100ContinueHandler handler) {
  expect_100_continue_handler_ = std::move(handler);
  return *this;
}

// 设置地址族
inline Server &Server::set_address_family(int family) {
  address_family_ = family;
  return *this;
}

// 设置 TCP 禁用延迟
inline Server &Server::set_tcp_nodelay(bool on) {
  tcp_nodelay_ = on;
  return *this;
}

// 设置套接字选项
inline Server &Server::set_socket_options(SocketOptions socket_options) {
  socket_options_ = std::move(socket_options);
  return *this;
}

// 设置默认头部
inline Server &Server::set_default_headers(Headers headers) {
  default_headers_ = std::move(headers);
  return *this;
}

// 设置头部写入器
inline Server &Server::set_header_writer(
    std::function<ssize_t(Stream &, Headers &)> const &writer) {
  header_writer_ = writer;
  return *this;
}

// 设置保持活动的最大计数
inline Server &Server::set_keep_alive_max_count(size_t count) {
  keep_alive_max_count_ = count;
  return *this;
}
// 设置服务器的 keep-alive 超时时间，返回服务器对象的引用
inline Server &Server::set_keep_alive_timeout(time_t sec) {
  keep_alive_timeout_sec_ = sec;
  return *this;
}

// 设置服务器的读取超时时间，返回服务器对象的引用
inline Server &Server::set_read_timeout(time_t sec, time_t usec) {
  read_timeout_sec_ = sec;
  read_timeout_usec_ = usec;
  return *this;
}

// 设置服务器的写入超时时间，返回服务器对象的引用
inline Server &Server::set_write_timeout(time_t sec, time_t usec) {
  write_timeout_sec_ = sec;
  write_timeout_usec_ = usec;
  return *this;
}

// 设置服务器的空闲间隔时间，返回服务器对象的引用
inline Server &Server::set_idle_interval(time_t sec, time_t usec) {
  idle_interval_sec_ = sec;
  idle_interval_usec_ = usec;
  return *this;
}

// 设置服务器的最大负载长度，返回服务器对象的引用
inline Server &Server::set_payload_max_length(size_t length) {
  payload_max_length_ = length;
  return *this;
}

// 绑定服务器到指定端口，返回绑定是否成功
inline bool Server::bind_to_port(const std::string &host, int port,
                                 int socket_flags) {
  if (bind_internal(host, port, socket_flags) < 0) return false;
  return true;
}

// 绑定服务器到任意端口，返回绑定结果
inline int Server::bind_to_any_port(const std::string &host, int socket_flags) {
  return bind_internal(host, 0, socket_flags);
}

// 在绑定后开始监听连接，返回监听结果
inline bool Server::listen_after_bind() {
  auto se = detail::scope_exit([&]() { done_ = true; });
  return listen_internal();
}

// 绑定到指定主机和端口后开始监听连接，返回监听结果
inline bool Server::listen(const std::string &host, int port,
                           int socket_flags) {
  auto se = detail::scope_exit([&]() { done_ = true; });
  return bind_to_port(host, port, socket_flags) && listen_internal();
}

// 检查服务器是否正在运行
inline bool Server::is_running() const { return is_running_; }

// 等待服务器准备就绪
inline void Server::wait_until_ready() const {
  while (!is_running() && !done_) {
    std::this_thread::sleep_for(std::chrono::milliseconds{1});
  }
}

// 停止服务器运行
inline void Server::stop() {
  if (is_running_) {
    assert(svr_sock_ != INVALID_SOCKET);
    std::atomic<socket_t> sock(svr_sock_.exchange(INVALID_SOCKET));
    detail::shutdown_socket(sock);
    detail::close_socket(sock);
  }
}

// 解析请求行，填充请求对象，返回解析结果
inline bool Server::parse_request_line(const char *s, Request &req) {
  auto len = strlen(s);
  if (len < 2 || s[len - 2] != '\r' || s[len - 1] != '\n') { return false; }
  len -= 2;

  {
    // 初始化计数器为0
    size_t count = 0;

    // 使用自定义的split函数将字符串s按空格分割，并对每个部分进行处理
    detail::split(s, s + len, ' ', [&](const char *b, const char *e) {
      // 根据计数器的值，将分割后的部分赋值给请求对象的不同属性
      switch (count) {
      case 0: req.method = std::string(b, e); break;
      case 1: req.target = std::string(b, e); break;
      case 2: req.version = std::string(b, e); break;
      default: break;
      }
      // 计数器自增
      count++;
    });

    // 如果计数器不等于3，返回false
    if (count != 3) { return false; }
  }

  // 定义HTTP请求方法的集合
  static const std::set<std::string> methods{
      "GET",     "HEAD",    "POST",  "PUT",   "DELETE",
      "CONNECT", "OPTIONS", "TRACE", "PATCH", "PRI"};

  // 如果请求方法不在定义的集合中，返回false
  if (methods.find(req.method) == methods.end()) { return false; }

  // 如果请求版本不是HTTP/1.1或HTTP/1.0，返回false
  if (req.version != "HTTP/1.1" && req.version != "HTTP/1.0") { return false; }

  {
    // 跳过URL中的片段部分
    for (size_t i = 0; i < req.target.size(); i++) {
      if (req.target[i] == '#') {
        req.target.erase(i);
        break;
      }
    }

    // 重新初始化计数器为0
    size_t count = 0;

    // 使用自定义的split函数将请求目标按'?'分割，并对每个部分进行处理
    detail::split(req.target.data(), req.target.data() + req.target.size(), '?',
                  [&](const char *b, const char *e) {
                    // 根据计数器的值，将分割后的部分赋值给请求对象的不同属性
                    switch (count) {
                    case 0:
                      req.path = detail::decode_url(std::string(b, e), false);
                      break;
                    case 1: {
                      if (e - b > 0) {
                        detail::parse_query_text(std::string(b, e), req.params);
                      }
                      break;
                    }
                    default: break;
                    }
                    // 计数器自增
                    count++;
                  });

    // 如果计数器大于2，返回false
    if (count > 2) { return false; }
  }

  // 返回true表示解析成功
  return true;
// 写入响应内容到流中，根据需要关闭连接，处理请求和响应，返回是否成功
inline bool Server::write_response(Stream &strm, bool close_connection,
                                   const Request &req, Response &res) {
  return write_response_core(strm, close_connection, req, res, false);
}

// 写入带有内容的响应到流中，根据需要关闭连接，处理请求和响应，返回是否成功
inline bool Server::write_response_with_content(Stream &strm,
                                                bool close_connection,
                                                const Request &req,
                                                Response &res) {
  return write_response_core(strm, close_connection, req, res, true);
}

// 写入响应核心功能，根据需要关闭连接，处理请求和响应，返回是否成功
inline bool Server::write_response_core(Stream &strm, bool close_connection,
                                        const Request &req, Response &res,
                                        bool need_apply_ranges) {
  // 确保响应状态码不为-1
  assert(res.status != -1);

  // 如果响应状态码大于等于400且存在错误处理函数且错误处理函数返回Handled，则需要应用范围
  if (400 <= res.status && error_handler_ &&
      error_handler_(req, res) == HandlerResponse::Handled) {
    need_apply_ranges = true;
  }

  // 准备内容类型和边界
  std::string content_type;
  std::string boundary;
  if (need_apply_ranges) { apply_ranges(req, res, content_type, boundary); }

  // 准备额外的头部信息
  if (close_connection || req.get_header_value("Connection") == "close") {
    res.set_header("Connection", "close");
  } else {
    std::stringstream ss;
    ss << "timeout=" << keep_alive_timeout_sec_
       << ", max=" << keep_alive_max_count_;
    res.set_header("Keep-Alive", ss.str());
  }

  // 如果响应没有Content-Type头部且响应体不为空或内容长度大于0或存在内容提供者，则设置为"text/plain"
  if (!res.has_header("Content-Type") &&
      (!res.body.empty() || res.content_length_ > 0 || res.content_provider_)) {
    res.set_header("Content-Type", "text/plain");
  }

  // 如果响应没有Content-Length头部且响应体为空且内容长度为0且不存在内容提供者，则设置为"0"
  if (!res.has_header("Content-Length") && res.body.empty() &&
      !res.content_length_ && !res.content_provider_) {
    res.set_header("Content-Length", "0");
  }

  // 如果响应没有Accept-Ranges头部且请求方法为"HEAD"，则设置为"bytes"
  if (!res.has_header("Accept-Ranges") && req.method == "HEAD") {
    res.set_header("Accept-Ranges", "bytes");
  }

  // 如果存在后置路由处理函数，则调用
  if (post_routing_handler_) { post_routing_handler_(req, res); }

  // 响应行和头部信息
  {
    // 创建一个 BufferStream 对象
    detail::BufferStream bstrm;

    // 写入 HTTP 响应的状态行，如果写入失败则返回 false
    if (!bstrm.write_format("HTTP/1.1 %d %s\r\n", res.status,
                            status_message(res.status))) {
      return false;
    }

    // 写入响应头部信息，如果写入失败则返回 false
    if (!header_writer_(bstrm, res.headers)) { return false; }

    // 刷新缓冲区
    auto &data = bstrm.get_buffer();
    detail::write_data(strm, data.data(), data.size());
  }

  // 处理响应体
  auto ret = true;
  if (req.method != "HEAD") {
    // 如果响应体不为空
    if (!res.body.empty()) {
      // 将响应体数据写入流，如果写入失败则将 ret 置为 false
      if (!detail::write_data(strm, res.body.data(), res.body.size())) {
        ret = false;
      }
    } else if (res.content_provider_) {
      // 如果存在内容提供者，则调用相应函数处理响应体
      if (write_content_with_provider(strm, req, res, boundary, content_type)) {
        res.content_provider_success_ = true;
      } else {
        res.content_provider_success_ = false;
        ret = false;
      }
    }
  }

  // 记录日志
  if (logger_) { logger_(req, res); }

  // 返回处理结果
  return ret;
// 内联函数，用于在服务器关闭时检查是否正在关闭
inline bool
Server::write_content_with_provider(Stream &strm, const Request &req,
                                    Response &res, const std::string &boundary,
                                    const std::string &content_type) {
  // Lambda函数，检查服务器是否正在关闭
  auto is_shutting_down = [this]() {
    return this->svr_sock_ == INVALID_SOCKET;
  };

  // 如果响应内容长度大于0
  if (res.content_length_ > 0) {
    // 如果请求中没有范围
    if (req.ranges.empty()) {
      // 写入内容到流中，从0开始，到内容长度结束
      return detail::write_content(strm, res.content_provider_, 0,
                                   res.content_length_, is_shutting_down);
    } else if (req.ranges.size() == 1) {
      // 获取范围的偏移和长度
      auto offsets =
          detail::get_range_offset_and_length(req, res.content_length_, 0);
      auto offset = offsets.first;
      auto length = offsets.second;
      // 写入内容到流中，从偏移开始，指定长度
      return detail::write_content(strm, res.content_provider_, offset, length,
                                   is_shutting_down);
    } else {
      // 写入多部分范围数据
      return detail::write_multipart_ranges_data(
          strm, req, res, boundary, content_type, is_shutting_down);
    }
  } else {
    // 如果是分块内容提供程序
    if (res.is_chunked_content_provider_) {
      // 获取编码类型
      auto type = detail::encoding_type(req, res);

      std::unique_ptr<detail::compressor> compressor;
      // 如果是Gzip编码
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        // 创建Gzip压缩器
        compressor = detail::make_unique<detail::gzip_compressor>();
#endif
      } else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        // 创建Brotli压缩器
        compressor = detail::make_unique<detail::brotli_compressor>();
#endif
      } else {
        // 创建无压缩器
        compressor = detail::make_unique<detail::nocompressor>();
      }
      // 断言压缩器不为空
      assert(compressor != nullptr);

      // 写入分块内容到流中
      return detail::write_content_chunked(strm, res.content_provider_,
                                           is_shutting_down, *compressor);
    } else {
      // 写入无长度的内容到流中
      return detail::write_content_without_length(strm, res.content_provider_,
                                                  is_shutting_down);
    }
  }
}
// 读取请求内容并解析，将解析结果存储在请求对象中，返回是否成功的布尔值
inline bool Server::read_content(Stream &strm, Request &req, Response &res) {
  // 定义迭代器和文件计数器
  MultipartFormDataMap::iterator cur;
  auto file_count = 0;
  // 调用 read_content_core 函数，处理请求内容
  if (read_content_core(
          strm, req, res,
          // 处理普通请求内容
          [&](const char *buf, size_t n) {
            // 检查请求体大小是否超过最大限制
            if (req.body.size() + n > req.body.max_size()) { return false; }
            // 将数据追加到请求体中
            req.body.append(buf, n);
            return true;
          },
          // 处理多部分请求内容
          [&](const MultipartFormData &file) {
            // 检查文件计数是否达到最大限制
            if (file_count++ == CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT) {
              return false;
            }
            // 将文件信息存储到请求对象的文件映射中
            cur = req.files.emplace(file.name, file);
            return true;
          },
          // 处理多部分请求内容数据
          [&](const char *buf, size_t n) {
            // 获取当前文件内容
            auto &content = cur->second.content;
            // 检查内容大小是否超过最大限制
            if (content.size() + n > content.max_size()) { return false; }
            // 将数据追加到文件内容中
            content.append(buf, n);
            return true;
          })) {
    // 获取请求头中的 Content-Type
    const auto &content_type = req.get_header_value("Content-Type");
    // 检查是否为 application/x-www-form-urlencoded 类型
    if (!content_type.find("application/x-www-form-urlencoded")) {
      // 检查请求体大小是否超过最大限制
      if (req.body.size() > CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH) {
        // 设置响应状态码为 413
        res.status = 413; // NOTE: should be 414?
        return false;
      }
      // 解析查询字符串并存储到请求参数中
      detail::parse_query_text(req.body, req.params);
    }
    return true;
  }
  return false;
}

// 读取请求内容并解析，使用自定义的内容接收器，返回是否成功的布尔值
inline bool Server::read_content_with_content_receiver(
    Stream &strm, Request &req, Response &res, ContentReceiver receiver,
    MultipartContentHeader multipart_header,
    ContentReceiver multipart_receiver) {
  // 调用 read_content_core 函数，处理请求内容
  return read_content_core(strm, req, res, std::move(receiver),
                           std::move(multipart_header),
                           std::move(multipart_receiver));
}
// 读取请求内容的核心函数，根据请求类型选择不同的处理方式
inline bool Server::read_content_core(Stream &strm, Request &req, Response &res,
                                      ContentReceiver receiver,
                                      MultipartContentHeader multipart_header,
                                      ContentReceiver multipart_receiver) {
  // 创建一个用于解析多部分表单数据的对象
  detail::MultipartFormDataParser multipart_form_data_parser;
  // 定义一个带进度的内容接收器
  ContentReceiverWithProgress out;

  // 如果请求是多部分表单数据
  if (req.is_multipart_form_data()) {
    // 获取请求头中的 Content-Type
    const auto &content_type = req.get_header_value("Content-Type");
    std::string boundary;
    // 解析出多部分数据的边界
    if (!detail::parse_multipart_boundary(content_type, boundary)) {
      // 如果解析失败，设置响应状态码为 400，并返回 false
      res.status = 400;
      return false;
    }

    // 设置多部分表单数据解析器的边界
    multipart_form_data_parser.set_boundary(std::move(boundary));
    // 定义一个内容接收器，用于处理多部分数据
    out = [&](const char *buf, size_t n, uint64_t /*off*/, uint64_t /*len*/) {
      // 调用多部分表单数据解析器的 parse 方法，解析数据并处理
      return multipart_form_data_parser.parse(buf, n, multipart_receiver, multipart_header);
    };
  } else {
    // 如果请求不是多部分表单数据，直接使用普通的内容接收器
    out = [receiver](const char *buf, size_t n, uint64_t /*off*/,
                     uint64_t /*len*/) { return receiver(buf, n); };
  }

  // 如果请求方法是 DELETE 且没有 Content-Length 头部，则直接返回 true
  if (req.method == "DELETE" && !req.has_header("Content-Length")) {
    return true;
  }

  // 读取请求内容，并根据情况调用不同的内容接收器
  if (!detail::read_content(strm, req, payload_max_length_, res.status, nullptr,
                            out, true)) {
    return false;
  }

  // 如果请求是多部分表单数据，检查解析器是否有效
  if (req.is_multipart_form_data()) {
    if (!multipart_form_data_parser.is_valid()) {
      // 如果解析器无效，设置响应状态码为 400，并返回 false
      res.status = 400;
      return false;
    }
  }

  // 处理完毕，返回 true
  return true;
}
// 处理文件请求的函数，根据请求路径查找对应文件并返回内容
inline bool Server::handle_file_request(const Request &req, Response &res,
                                        bool head) {
  // 遍历基本目录列表
  for (const auto &entry : base_dirs_) {
    // 前缀匹配
    if (!req.path.compare(0, entry.mount_point.size(), entry.mount_point)) {
      // 构造子路径
      std::string sub_path = "/" + req.path.substr(entry.mount_point.size());
      // 检查子路径是否有效
      if (detail::is_valid_path(sub_path)) {
        // 拼接完整文件路径
        auto path = entry.base_dir + sub_path;
        // 如果路径以 '/' 结尾，则追加 "index.html"
        if (path.back() == '/') { path += "index.html"; }

        // 检查文件是否存在
        if (detail::is_file(path)) {
          // 设置响应头
          for (const auto &kv : entry.headers) {
            res.set_header(kv.first.c_str(), kv.second);
          }

          // 创建共享内存映射对象
          auto mm = std::make_shared<detail::mmap>(path.c_str());
          // 如果打开失败，则返回 false
          if (!mm->is_open()) { return false; }

          // 设置内容提供者，提供文件内容
          res.set_content_provider(
              mm->size(),
              detail::find_content_type(path, file_extension_and_mimetype_map_,
                                        default_file_mimetype_),
              [mm](size_t offset, size_t length, DataSink &sink) -> bool {
                sink.write(mm->data() + offset, length);
                return true;
              });

          // 如果不是 head 请求且存在文件请求处理函数，则调用处理函数
          if (!head && file_request_handler_) {
            file_request_handler_(req, res);
          }

          return true;
        }
      }
    }
  }
  return false;
}

// 创建服务器套接字
inline socket_t
Server::create_server_socket(const std::string &host, int port,
                             int socket_flags,
                             SocketOptions socket_options) const {
  // 调用底层函数创建套接字
  return detail::create_socket(
      host, std::string(), port, address_family_, socket_flags, tcp_nodelay_,
      std::move(socket_options),
      // 绑定套接字地址并监听连接
      [](socket_t sock, struct addrinfo &ai) -> bool {
        if (::bind(sock, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen))) {
          return false;
        }
        if (::listen(sock, CPPHTTPLIB_LISTEN_BACKLOG)) { return false; }
        return true;
      });
}
// 在服务器对象内部绑定主机和端口，返回绑定的端口号
inline int Server::bind_internal(const std::string &host, int port,
                                 int socket_flags) {
  // 如果服务器对象无效，则返回错误
  if (!is_valid()) { return -1; }

  // 创建服务器套接字并绑定主机、端口以及套接字标志
  svr_sock_ = create_server_socket(host, port, socket_flags, socket_options_);
  // 如果创建套接字失败，则返回错误
  if (svr_sock_ == INVALID_SOCKET) { return -1; }

  // 如果端口号为0，则获取套接字的本地地址信息
  if (port == 0) {
    struct sockaddr_storage addr;
    socklen_t addr_len = sizeof(addr);
    // 获取套接字的本地地址信息
    if (getsockname(svr_sock_, reinterpret_cast<struct sockaddr *>(&addr),
                    &addr_len) == -1) {
      return -1;
    }
    // 根据地址家族返回对应的端口号
    if (addr.ss_family == AF_INET) {
      return ntohs(reinterpret_cast<struct sockaddr_in *>(&addr)->sin_port);
    } else if (addr.ss_family == AF_INET6) {
      return ntohs(reinterpret_cast<struct sockaddr_in6 *>(&addr)->sin6_port);
    } else {
      return -1;
    }
  } else {
    // 如果端口号不为0，则直接返回端口号
    return port;
  }
}

// 内部函数，用于监听服务器套接字
inline bool Server::listen_internal() {
  auto ret = true;
  // 设置服务器正在运行标志为true，并在作用域结束时将其设置为false
  is_running_ = true;
  auto se = detail::scope_exit([&]() { is_running_ = false; });

  {
    // 创建一个新的任务队列
    std::unique_ptr<TaskQueue> task_queue(new_task_queue());

    // 循环监听服务器套接字的活动
    while (svr_sock_ != INVALID_SOCKET) {
#ifndef _WIN32
      // 如果不是Windows系统，且设置了空闲间隔时间
      if (idle_interval_sec_ > 0 || idle_interval_usec_ > 0) {
#endif
        // 使用select函数检查套接字是否可读
        auto val = detail::select_read(svr_sock_, idle_interval_sec_,
                                       idle_interval_usec_);
        // 如果返回值为0，表示超时
        if (val == 0) { // Timeout
          // 在任务队列上执行空闲操作
          task_queue->on_idle();
          continue;
        }
#ifndef _WIN32
      }
    }
  }
}
#endif
      // 接受客户端连接请求，返回新的套接字
      socket_t sock = accept(svr_sock_, nullptr, nullptr);

      // 如果接受失败
      if (sock == INVALID_SOCKET) {
        // 如果错误为 EMFILE，表示进程打开文件描述符的限制已达到
        // 等待一段时间后再尝试接受新连接
        if (errno == EMFILE) {
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          continue;
        } else if (errno == EINTR || errno == EAGAIN) {
          continue;
        }
        // 如果服务器套接字不是无效套接字
        if (svr_sock_ != INVALID_SOCKET) {
          // 关闭服务器套接字
          detail::close_socket(svr_sock_);
          ret = false;
        } else {
          ; // 服务器套接字已被用户关闭
        }
        break;
      }

      {
#ifdef _WIN32
        // 设置接收超时时间
        auto timeout = static_cast<uint32_t>(read_timeout_sec_ * 1000 +
                                             read_timeout_usec_ / 1000);
        setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO,
                   reinterpret_cast<const char *>(&timeout), sizeof(timeout));
#else
        // 设置接收超时时间
        timeval tv;
        tv.tv_sec = static_cast<long>(read_timeout_sec_);
        tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec_);
        setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO,
                   reinterpret_cast<const void *>(&tv), sizeof(tv));
#endif
      }
      {
#ifdef _WIN32
        // 设置发送超时时间
        auto timeout = static_cast<uint32_t>(write_timeout_sec_ * 1000 +
                                             write_timeout_usec_ / 1000);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO,
                   reinterpret_cast<const char *>(&timeout), sizeof(timeout));
#else
        // 设置发送超时时间
        timeval tv;
        tv.tv_sec = static_cast<long>(write_timeout_sec_);
        tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec_);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO,
                   reinterpret_cast<const void *>(&tv), sizeof(tv));
#endif
      }

      // 将处理套接字的任务加入任务队列
      task_queue->enqueue([this, sock]() { process_and_close_socket(sock); });
    }

    // 关闭任务队列
    task_queue->shutdown();
  }

  // 返回操作结果
  return ret;
}
  // 检查是否有预处理路由处理器，如果有并且处理成功，则返回true
  if (pre_routing_handler_ &&
      pre_routing_handler_(req, res) == HandlerResponse::Handled) {
    return true;
  }

  // 文件处理器
  auto is_head_request = req.method == "HEAD";
  // 如果是GET请求或者HEAD请求，并且处理文件请求成功，则返回true
  if ((req.method == "GET" || is_head_request) &&
      handle_file_request(req, res, is_head_request)) {
    return true;
  }

  // 如果请求中包含内容
  if (detail::expect_content(req)) {
    // 内容读取处理器
    {
      // 创建内容读取器，根据请求方法不同选择不同的处理方式
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

      // 根据请求方法选择不同的处理器进行处理
      if (req.method == "POST") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                post_handlers_for_content_reader_)) {
          return true;
        }
      } else if (req.method == "PUT") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                put_handlers_for_content_reader_)) {
          return true;
        }
      } else if (req.method == "PATCH") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                patch_handlers_for_content_reader_)) {
          return true;
        }
      } else if (req.method == "DELETE") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                delete_handlers_for_content_reader_)) {
          return true;
        }
      }
    }

    // 将内容读取到`req.body`中
    // 如果读取请求内容失败，则返回 false
    if (!read_content(strm, req, res)) { return false; }
  }

  // 处理 GET 和 HEAD 请求
  if (req.method == "GET" || req.method == "HEAD") {
    return dispatch_request(req, res, get_handlers_);
  } 
  // 处理 POST 请求
  else if (req.method == "POST") {
    return dispatch_request(req, res, post_handlers_);
  } 
  // 处理 PUT 请求
  else if (req.method == "PUT") {
    return dispatch_request(req, res, put_handlers_);
  } 
  // 处理 DELETE 请求
  else if (req.method == "DELETE") {
    return dispatch_request(req, res, delete_handlers_);
  } 
  // 处理 OPTIONS 请求
  else if (req.method == "OPTIONS") {
    return dispatch_request(req, res, options_handlers_);
  } 
  // 处理 PATCH 请求
  else if (req.method == "PATCH") {
    return dispatch_request(req, res, patch_handlers_);
  }

  // 如果请求方法不在上述处理范围内，则返回 400 错误
  res.status = 400;
  return false;
// 在服务器上分发请求，根据请求和处理程序的匹配情况执行相应的处理程序
inline bool Server::dispatch_request(Request &req, Response &res,
                                     const Handlers &handlers) {
  // 遍历处理程序集合
  for (const auto &x : handlers) {
    // 获取匹配器和处理程序
    const auto &matcher = x.first;
    const auto &handler = x.second;

    // 如果请求与匹配器匹配
    if (matcher->match(req)) {
      // 执行处理程序
      handler(req, res);
      return true;
    }
  }
  return false;
}

// 应用范围请求，处理响应中的内容类型和边界
inline void Server::apply_ranges(const Request &req, Response &res,
                                 std::string &content_type,
                                 std::string &boundary) {
  // 如果请求中包含多个范围
  if (req.ranges.size() > 1) {
    // 生成多部分数据的边界
    boundary = detail::make_multipart_data_boundary();

    // 查找响应头中的内容类型
    auto it = res.headers.find("Content-Type");
    if (it != res.headers.end()) {
      // 保存内容类型并从响应头中删除
      content_type = it->second;
      res.headers.erase(it);
    }

    // 设置响应头中的内容类型为多部分字节范围，并包含边界
    res.set_header("Content-Type",
                   "multipart/byteranges; boundary=" + boundary);
  }

  // 获取编码类型
  auto type = detail::encoding_type(req, res);

  // 如果响应体为空
  if (res.body.empty()) {
    // 如果内容长度大于0
    if (res.content_length_ > 0) {
      size_t length = 0;
      // 如果请求中没有范围
      if (req.ranges.empty()) {
        length = res.content_length_;
      } else if (req.ranges.size() == 1) {
        // 获取范围的偏移和长度
        auto offsets =
            detail::get_range_offset_and_length(req, res.content_length_, 0);
        length = offsets.second;

        // 生成内容范围头字段并设置到响应头中
        auto content_range = detail::make_content_range_header_field(
            req.ranges[0], res.content_length_);
        res.set_header("Content-Range", content_range);
      } else {
        // 获取多部分范围数据的长度
        length = detail::get_multipart_ranges_data_length(req, res, boundary,
                                                          content_type);
      }
      // 设置响应头中的内容长度
      res.set_header("Content-Length", std::to_string(length));
    } else {
      // 如果响应内容提供者存在
      if (res.content_provider_) {
        // 如果响应内容以分块方式提供
        if (res.is_chunked_content_provider_) {
          // 设置响应头部，表明内容以分块传输
          res.set_header("Transfer-Encoding", "chunked");
          // 根据编码类型设置内容编码
          if (type == detail::EncodingType::Gzip) {
            res.set_header("Content-Encoding", "gzip");
          } else if (type == detail::EncodingType::Brotli) {
            res.set_header("Content-Encoding", "br");
          }
        }
      }
    }
  } else {
    // 如果请求中没有范围信息
    if (req.ranges.empty()) {
      // 空语句，不执行任何操作
      ;
    } else if (req.ranges.size() == 1) {
      // 生成内容范围头字段
      auto content_range = detail::make_content_range_header_field(
          req.ranges[0], res.body.size());
      res.set_header("Content-Range", content_range);

      // 获取范围偏移和长度
      auto offsets =
          detail::get_range_offset_and_length(req, res.body.size(), 0);
      auto offset = offsets.first;
      auto length = offsets.second;

      // 如果偏移小于响应体大小
      if (offset < res.body.size()) {
        // 截取响应体的部分内容
        res.body = res.body.substr(offset, length);
      } else {
        // 清空响应体并设置状态码为416
        res.body.clear();
        res.status = 416;
      }
    } else {
      // 处理多个范围请求
      std::string data;
      // 生成多部分范围数据
      if (detail::make_multipart_ranges_data(req, res, boundary, content_type,
                                             data)) {
        // 交换响应体内容
        res.body.swap(data);
      } else {
        // 清空响应体并设置状态码为416
        res.body.clear();
        res.status = 416;
      }
    }

    // 如果存在内容编码
    if (type != detail::EncodingType::None) {
      std::unique_ptr<detail::compressor> compressor;
      std::string content_encoding;

      // 根据编码类型选择压缩器
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        // 如果支持 ZLIB 压缩，则创建 gzip 压缩器，并设置内容编码为 gzip
        compressor = detail::make_unique<detail::gzip_compressor>();
        content_encoding = "gzip";
#endif
      } else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        // 如果支持 Brotli 压缩，则创建 brotli 压缩器，并设置内容编码为 br
        compressor = detail::make_unique<detail::brotli_compressor>();
        content_encoding = "br";
#endif
      }

      // 如果存在压缩器
      if (compressor) {
        // 压缩请求体数据
        std::string compressed;
        if (compressor->compress(res.body.data(), res.body.size(), true,
                                 [&](const char *data, size_t data_len) {
                                   compressed.append(data, data_len);
                                   return true;
                                 })) {
          // 替换原始请求体数据为压缩后的数据
          res.body.swap(compressed);
          // 设置响应头中的内容编码
          res.set_header("Content-Encoding", content_encoding);
        }
      }
    }

    // 计算响应体数据的长度，并设置 Content-Length 响应头
    auto length = std::to_string(res.body.size());
    res.set_header("Content-Length", length);
  }
}

// 处理请求的内容读取器
inline bool Server::dispatch_request_for_content_reader(
    Request &req, Response &res, ContentReader content_reader,
    const HandlersForContentReader &handlers) {
  // 遍历内容读取器的处理器
  for (const auto &x : handlers) {
    const auto &matcher = x.first;
    const auto &handler = x.second;

    // 如果请求匹配到内容读取器的处理器，则调用处理器处理请求
    if (matcher->match(req)) {
      handler(req, res, content_reader);
      return true;
    }
  }
  return false;
}

// 处理请求
inline bool
Server::process_request(Stream &strm, bool close_connection,
                        bool &connection_closed,
                        const std::function<void(Request &)> &setup_request) {
  // 读取请求数据
  std::array<char, 2048> buf{};

  detail::stream_line_reader line_reader(strm, buf.data(), buf.size());

  // 如果客户端关闭连接，则返回 false
  if (!line_reader.getline()) { return false; }

  // 创建请求对象
  Request req;

  // 创建响应对象，并设置默认的 HTTP 版本和响应头
  Response res;
  res.version = "HTTP/1.1";
  res.headers = default_headers_;

#ifdef _WIN32
  // TODO: Increase FD_SETSIZE statically (libzmq), dynamically (MySQL).
#else
#ifndef CPPHTTPLIB_USE_POLL
  // 如果套接字文件描述符超过了 FD_SETSIZE...
  if (strm.socket() >= FD_SETSIZE) {
    // 创建一个空的 Headers 对象
    Headers dummy;
    // 读取请求头信息
    detail::read_headers(strm, dummy);
    // 设置响应状态码为 500
    res.status = 500;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }
#endif
#endif

  // 检查请求 URI 是否超过了限制
  if (line_reader.size() > CPPHTTPLIB_REQUEST_URI_MAX_LENGTH) {
    // 创建一个空的 Headers 对象
    Headers dummy;
    // 读取请求头信息
    detail::read_headers(strm, dummy);
    // 设置响应状态码为 414
    res.status = 414;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }

  // 解析请求行和请求头
  if (!parse_request_line(line_reader.ptr(), req) ||
      !detail::read_headers(strm, req.headers)) {
    // 设置响应状态码为 400
    res.status = 400;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }

  // 检查是否需要关闭连接
  if (req.get_header_value("Connection") == "close") {
    connection_closed = true;
  }

  // 检查是否为 HTTP/1.0 版本且不是 Keep-Alive 连接
  if (req.version == "HTTP/1.0" &&
      req.get_header_value("Connection") != "Keep-Alive") {
    connection_closed = true;
  }

  // 获取远程 IP 和端口信息
  strm.get_remote_ip_and_port(req.remote_addr, req.remote_port);
  req.set_header("REMOTE_ADDR", req.remote_addr);
  req.set_header("REMOTE_PORT", std::to_string(req.remote_port));

  // 获取本地 IP 和端口信息
  strm.get_local_ip_and_port(req.local_addr, req.local_port);
  req.set_header("LOCAL_ADDR", req.local_addr);
  req.set_header("LOCAL_PORT", std::to_string(req.local_port));

  // 如果请求头中包含 Range 字段
  if (req.has_header("Range")) {
    // 获取 Range 头字段的值
    const auto &range_header_value = req.get_header_value("Range");
    // 解析 Range 头字段
    if (!detail::parse_range_header(range_header_value, req.ranges)) {
      // 设置响应状态码为 416
      res.status = 416;
      // 返回响应
      return write_response(strm, close_connection, req, res);
    }
  }

  // 如果设置了请求处理函数，则调用
  if (setup_request) { setup_request(req); }

  // 如果请求头中包含 Expect 字段为 "100-continue"
  if (req.get_header_value("Expect") == "100-continue") {
    auto status = 100;
    // 如果存在 100 Continue 处理函数，则调用
    if (expect_100_continue_handler_) {
      status = expect_100_continue_handler_(req, res);
    }
    switch (status) {
    case 100:
    # 根据状态码和状态消息格式化输出 HTTP 响应头部信息
    case 417:
      strm.write_format("HTTP/1.1 %d %s\r\n\r\n", status,
                        status_message(status));
      # 结束当前 case 分支
      break;
    # 默认情况下，调用 write_response 函数处理响应
    default: return write_response(strm, close_connection, req, res);
    }
  }

  // 路由标记，用于标识是否已经进行了路由处理
  auto routed = false;
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
  // 如果 CPPHTTPLIB_NO_EXCEPTIONS 宏被定义，则直接调用 routing 函数处理请求
  routed = routing(req, res, strm);
#else
  // 否则尝试调用 routing 函数处理请求，捕获可能抛出的异常
  try {
    routed = routing(req, res, strm);
  } catch (std::exception &e) {
    // 如果捕获到异常且存在异常处理函数，则调用异常处理函数
    if (exception_handler_) {
      auto ep = std::current_exception();
      exception_handler_(req, res, ep);
      routed = true;
    } else {
      // 否则设置响应状态码为 500，并将异常信息转义后存入响应头
      res.status = 500;
      std::string val;
      auto s = e.what();
      for (size_t i = 0; s[i]; i++) {
        switch (s[i]) {
        case '\r': val += "\\r"; break;
        case '\n': val += "\\n"; break;
        default: val += s[i]; break;
        }
      }
      res.set_header("EXCEPTION_WHAT", val);
    }
  } catch (...) {
    // 捕获其他类型的异常，同样调用异常处理函数或设置默认异常信息
    if (exception_handler_) {
      auto ep = std::current_exception();
      exception_handler_(req, res, ep);
      routed = true;
    } else {
      res.status = 500;
      res.set_header("EXCEPTION_WHAT", "UNKNOWN");
    }
  }
#endif

  // 如果请求已被处理，则根据响应状态码返回响应内容
  if (routed) {
    if (res.status == -1) { res.status = req.ranges.empty() ? 200 : 206; }
    return write_response_with_content(strm, close_connection, req, res);
  } else {
    // 如果请求未被处理，则设置响应状态码为 404，并返回响应内容
    if (res.status == -1) { res.status = 404; }
    return write_response(strm, close_connection, req, res);
  }
}

// 检查服务器是否有效
inline bool Server::is_valid() const { return true; }

// 处理并关闭套接字
inline bool Server::process_and_close_socket(socket_t sock) {
  // 调用 detail::process_server_socket 处理服务器套接字
  auto ret = detail::process_server_socket(
      svr_sock_, sock, keep_alive_max_count_, keep_alive_timeout_sec_,
      read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
      write_timeout_usec_,
      [this](Stream &strm, bool close_connection, bool &connection_closed) {
        return process_request(strm, close_connection, connection_closed,
                               nullptr);
      });

  // 关闭套接字的读写功能
  detail::shutdown_socket(sock);
  // 关闭套接字
  detail::close_socket(sock);
  return ret;
}

// HTTP 客户端实现
inline ClientImpl::ClientImpl(const std::string &host)
    : ClientImpl(host, 80, std::string(), std::string()) {}

// HTTP 客户端实现
inline ClientImpl::ClientImpl(const std::string &host, int port)
    # 调用 ClientImpl 类的构造函数，传入参数 host, port, 空字符串和空字符串
    ClientImpl(host, port, std::string(), std::string()) {}
// 客户端实现的构造函数，初始化成员变量
inline ClientImpl::ClientImpl(const std::string &host, int port,
                              const std::string &client_cert_path,
                              const std::string &client_key_path)
    : host_(host), port_(port),
      host_and_port_(adjust_host_string(host) + ":" + std::to_string(port)),
      client_cert_path_(client_cert_path), client_key_path_(client_key_path) {}

// 客户端实现的析构函数，释放资源
inline ClientImpl::~ClientImpl() {
  // 使用互斥锁保护 socket 操作
  std::lock_guard<std::mutex> guard(socket_mutex_);
  // 关闭 socket 连接
  shutdown_socket(socket_);
  // 关闭 socket
  close_socket(socket_);
}

// 检查客户端是否有效
inline bool ClientImpl::is_valid() const { return true; }

// 复制客户端设置
inline void ClientImpl::copy_settings(const ClientImpl &rhs) {
  // 复制客户端证书路径和密钥路径
  client_cert_path_ = rhs.client_cert_path_;
  client_key_path_ = rhs.client_key_path_;
  // 复制连接超时、读取超时、写入超时等设置
  connection_timeout_sec_ = rhs.connection_timeout_sec_;
  read_timeout_sec_ = rhs.read_timeout_sec_;
  read_timeout_usec_ = rhs.read_timeout_usec_;
  write_timeout_sec_ = rhs.write_timeout_sec_;
  write_timeout_usec_ = rhs.write_timeout_usec_;
  // 复制基本认证用户名和密码
  basic_auth_username_ = rhs.basic_auth_username_;
  basic_auth_password_ = rhs.basic_auth_password_;
  // 复制令牌认证令牌
  bearer_token_auth_token_ = rhs.bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 复制摘要认证用户名和密码（如果支持 OpenSSL）
  digest_auth_username_ = rhs.digest_auth_username_;
  digest_auth_password_ = rhs.digest_auth_password_;
#endif
  // 复制保持连接、跟随重定向、URL 编码等设置
  keep_alive_ = rhs.keep_alive_;
  follow_location_ = rhs.follow_location_;
  url_encode_ = rhs.url_encode_;
  // 复制地址族、TCP NoDelay 等设置
  address_family_ = rhs.address_family_;
  tcp_nodelay_ = rhs.tcp_nodelay_;
  // 复制 socket 选项
  socket_options_ = rhs.socket_options_;
  // 复制压缩和解压缩设置
  compress_ = rhs.compress_;
  decompress_ = rhs.decompress_;
  // 复制接口、代理主机和端口等设置
  interface_ = rhs.interface_;
  proxy_host_ = rhs.proxy_host_;
  proxy_port_ = rhs.proxy_port_;
  proxy_basic_auth_username_ = rhs.proxy_basic_auth_username_;
  proxy_basic_auth_password_ = rhs.proxy_basic_auth_password_;
  proxy_bearer_token_auth_token_ = rhs.proxy_bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则复制代理服务器的摘要认证用户名和密码
  proxy_digest_auth_username_ = rhs.proxy_digest_auth_username_;
  proxy_digest_auth_password_ = rhs.proxy_digest_auth_password_;
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则复制 CA 证书文件路径、CA 证书目录路径和 CA 证书存储
  ca_cert_file_path_ = rhs.ca_cert_file_path_;
  ca_cert_dir_path_ = rhs.ca_cert_dir_path_;
  ca_cert_store_ = rhs.ca_cert_store_;
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则复制服务器证书验证选项
  server_certificate_verification_ = rhs.server_certificate_verification_;
#endif
  // 复制日志记录器
  logger_ = rhs.logger_;
}

// 创建客户端套接字
inline socket_t ClientImpl::create_client_socket(Error &error) const {
  // 如果设置了代理主机和代理端口，则创建代理客户端套接字
  if (!proxy_host_.empty() && proxy_port_ != -1) {
    return detail::create_client_socket(
        proxy_host_, std::string(), proxy_port_, address_family_, tcp_nodelay_,
        socket_options_, connection_timeout_sec_, connection_timeout_usec_,
        read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
        write_timeout_usec_, interface_, error);
  }

  // 检查是否为主机指定了自定义 IP 地址
  std::string ip;
  auto it = addr_map_.find(host_);
  if (it != addr_map_.end()) ip = it->second;

  // 创建客户端套接字
  return detail::create_client_socket(
      host_, ip, port_, address_family_, tcp_nodelay_, socket_options_,
      connection_timeout_sec_, connection_timeout_usec_, read_timeout_sec_,
      read_timeout_usec_, write_timeout_sec_, write_timeout_usec_, interface_,
      error);
}

// 创建并连接套接字
inline bool ClientImpl::create_and_connect_socket(Socket &socket,
                                                  Error &error) {
  // 创建客户端套接字
  auto sock = create_client_socket(error);
  // 如果套接字无效，则返回 false
  if (sock == INVALID_SOCKET) { return false; }
  // 将套接字赋值给 socket
  socket.sock = sock;
  return true;
}
// 关闭 SSL 连接，如果有其他线程中有请求正在进行，则存在线程不安全的竞争条件，因为单独的 ssl* 对象不是线程安全的
inline void ClientImpl::shutdown_ssl(Socket & /*socket*/,
                                     bool /*shutdown_gracefully*/) {
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());
}

// 关闭套接字连接
inline void ClientImpl::shutdown_socket(Socket &socket) {
  if (socket.sock == INVALID_SOCKET) { return; }
  detail::shutdown_socket(socket.sock);
}

// 关闭套接字连接，如果有其他线程中有请求正在进行，则存在潜在的 bug，因为操作系统可能会重新分配套接字 ID 用于新的套接字
inline void ClientImpl::close_socket(Socket &socket) {
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());

  // 如果 SSL 仍然处于活动状态，则存在 bug
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  assert(socket.ssl == nullptr);
#endif
  if (socket.sock == INVALID_SOCKET) { return; }
  detail::close_socket(socket.sock);
  socket.sock = INVALID_SOCKET;
}

// 读取响应行
inline bool ClientImpl::read_response_line(Stream &strm, const Request &req,
                                           Response &res) {
  std::array<char, 2048> buf{};

  // 从流中读取一行数据
  detail::stream_line_reader line_reader(strm, buf.data(), buf.size());

  // 如果无法读取到数据，则返回 false
  if (!line_reader.getline()) { return false; }

#ifdef CPPHTTPLIB_ALLOW_LF_AS_LINE_TERMINATOR
  // 定义正则表达式来匹配 HTTP 响应行
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r?\n");
#else
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r\n");
#endif

  std::cmatch m;
  // 使用正则表达式匹配响应行
  if (!std::regex_match(line_reader.ptr(), m, re)) {
    // 检查请求方法是否为 CONNECT
    return req.method == "CONNECT";
  }
  // 解析响应行中的 HTTP 版本、状态码和原因短语
  res.version = std::string(m[1]);
  res.status = std::stoi(std::string(m[2]));
  res.reason = std::string(m[3]);

  // 忽略 '100 Continue' 响应
  while (res.status == 100) {
    // 读取空行（CRLF）
    if (!line_reader.getline()) { return false; } // CRLF
    // 读取下一行响应
    if (!line_reader.getline()) { return false; } // next response line

    // 使用正则表达式匹配响应行
    if (!std::regex_match(line_reader.ptr(), m, re)) { return false; }
    // 更新响应的 HTTP 版本、状态码和原因短语
    res.version = std::string(m[1]);
    res.status = std::stoi(std::string(m[2]));
    res.reason = std::string(m[3]);
  }

  // 返回解析结果
  return true;
// 定义 ClientImpl 类的 send 方法，用于发送请求和接收响应
inline bool ClientImpl::send(Request &req, Response &res, Error &error) {
  // 使用递归互斥锁保护请求
  std::lock_guard<std::recursive_mutex> request_mutex_guard(request_mutex_);
  // 调用 send_ 方法发送请求，并接收响应
  auto ret = send_(req, res, error);
  // 如果出现 SSLPeerCouldBeClosed_ 错误，重新发送请求
  if (error == Error::SSLPeerCouldBeClosed_) {
    assert(!ret);
    ret = send_(req, res, error);
  }
  // 返回发送请求的结果
  return ret;
}

// 定义 ClientImpl 类的 send_ 方法，用于发送请求和接收响应
inline bool ClientImpl::send_(Request &req, Response &res, Error &error) {
  {
    // 使用互斥锁保护 socket
    std::lock_guard<std::mutex> guard(socket_mutex_);

    // 立即将 socket_should_be_closed_when_request_is_done_ 设置为 false
    // 如果在请求结束时它被设置为 true，说明另一个线程指示我们关闭 socket
    socket_should_be_closed_when_request_is_done_ = false;

    auto is_alive = false;
    // 检查 socket 是否打开
    if (socket_.is_open()) {
      // 检查 socket 是否存活
      is_alive = detail::is_socket_alive(socket_.sock);
      if (!is_alive) {
        // 如果另一端已经关闭连接，尝试避免 sigpipe，关闭 socket
        const bool shutdown_gracefully = false;
        shutdown_ssl(socket_, shutdown_gracefully);
        shutdown_socket(socket_);
        close_socket(socket_);
      }
    }

    // 如果 socket 不存活
    if (!is_alive) {
      // 创建并连接 socket
      if (!create_and_connect_socket(socket_, error)) { return false; }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 如果支持 OpenSSL
      if (is_ssl()) {
        auto &scli = static_cast<SSLClient &>(*this);
        // 如果使用代理，连接代理服务器
        if (!proxy_host_.empty() && proxy_port_ != -1) {
          auto success = false;
          if (!scli.connect_with_proxy(socket_, res, success, error)) {
            return success;
          }
        }

        // 初始化 SSL
        if (!scli.initialize_ssl(socket_, error)) { return false; }
      }
#endif
    }

    // 标记当前 socket 正在使用，防止在请求进行时被其他线程关闭
    // 如果当前有多个 socket 请求在处理，则断言这些请求都来自同一个线程
    if (socket_requests_in_flight_ > 1) {
      assert(socket_requests_are_from_thread_ == std::this_thread::get_id());
    }
    // 增加正在处理的 socket 请求数量
    socket_requests_in_flight_ += 1;
    // 记录当前处理请求的线程 ID
    socket_requests_are_from_thread_ = std::this_thread::get_id();
  }

  // 遍历默认请求头，如果请求中没有该头部，则插入
  for (const auto &header : default_headers_) {
    if (req.headers.find(header.first) == req.headers.end()) {
      req.headers.insert(header);
    }
  }

  // 初始化返回值和是否关闭连接的标志
  auto ret = false;
  auto close_connection = !keep_alive_;

  // 创建一个 scope_exit 对象，用于在作用域结束时执行指定的操作
  auto se = detail::scope_exit([&]() {
    // 加锁以标记请求不再进行中
    std::lock_guard<std::mutex> guard(socket_mutex_);
    // 减少正在处理的 socket 请求数量
    socket_requests_in_flight_ -= 1;
    // 如果没有正在处理的请求，则重置线程 ID
    if (socket_requests_in_flight_ <= 0) {
      assert(socket_requests_in_flight_ == 0);
      socket_requests_are_from_thread_ = std::thread::id();
    }

    // 如果需要关闭连接或者请求失败，则关闭 SSL、socket 并关闭连接
    if (socket_should_be_closed_when_request_is_done_ || close_connection ||
        !ret) {
      shutdown_ssl(socket_, true);
      shutdown_socket(socket_);
      close_socket(socket_);
    }
  });

  // 处理 socket，调用 handle_request 处理请求
  ret = process_socket(socket_, [&](Stream &strm) {
    return handle_request(strm, req, res, close_connection, error);
  });

  // 如果处理失败且错误码为 Success，则将错误码设置为 Unknown
  if (!ret) {
    if (error == Error::Success) { error = Error::Unknown; }
  }

  // 返回处理结果
  return ret;
// 结束 ClientImpl 类的定义
}

// 发送请求的实现，复制请求对象并调用 send_ 函数
inline Result ClientImpl::send(const Request &req) {
  auto req2 = req;
  return send_(std::move(req2));
}

// 发送请求的实际实现，创建响应对象、错误对象，并调用 send 函数发送请求
inline Result ClientImpl::send_(Request &&req) {
  auto res = detail::make_unique<Response>();
  auto error = Error::Success;
  auto ret = send(req, *res, error);
  return Result{ret ? std::move(res) : nullptr, error, std::move(req.headers)};
}

// 处理请求的函数，根据请求路径是否为空进行处理
inline bool ClientImpl::handle_request(Stream &strm, Request &req,
                                       Response &res, bool close_connection,
                                       Error &error) {
  if (req.path.empty()) {
    error = Error::Connection;
    return false;
  }

  auto req_save = req;

  bool ret;

  // 如果不是 SSL 并且有代理主机和端口，则处理代理请求
  if (!is_ssl() && !proxy_host_.empty() && proxy_port_ != -1) {
    auto req2 = req;
    req2.path = "http://" + host_and_port_ + req.path;
    ret = process_request(strm, req2, res, close_connection, error);
    req = req2;
    req.path = req_save.path;
  } else {
    ret = process_request(strm, req, res, close_connection, error);
  }

  // 如果处理请求失败，则返回 false
  if (!ret) { return false; }

  // 如果响应头中包含 Connection: close 或者版本为 HTTP/1.0 且不是 Connection established，则关闭连接
  std::lock_guard<std::mutex> guard(socket_mutex_);
  shutdown_ssl(socket_, true);
  shutdown_socket(socket_);
  close_socket(socket_);

  // 如果状态码为 3xx 且开启了重定向，则重定向请求
  if (300 < res.status && res.status < 400 && follow_location_) {
    req = req_save;
    ret = redirect(req, res, error);
  }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果状态码为 401 或 407 且授权次数小于 5，则处理授权
  if ((res.status == 401 || res.status == 407) &&
      req.authorization_count_ < 5) {
    // 检查响应状态码是否为407，判断是否为代理
    auto is_proxy = res.status == 407;
    // 根据是否为代理选择使用代理认证用户名或普通认证用户名
    const auto &username =
        is_proxy ? proxy_digest_auth_username_ : digest_auth_username_;
    // 根据是否为代理选择使用代理认证密码或普通认证密码
    const auto &password =
        is_proxy ? proxy_digest_auth_password_ : digest_auth_password_;

    // 如果用户名和密码均不为空
    if (!username.empty() && !password.empty()) {
      // 创建一个空的认证信息映射
      std::map<std::string, std::string> auth;
      // 解析响应中的认证信息，填充到auth映射中
      if (detail::parse_www_authenticate(res, auth, is_proxy)) {
        // 复制原始请求
        Request new_req = req;
        // 增加认证计数
        new_req.authorization_count_ += 1;
        // 移除原始请求中的认证头部信息
        new_req.headers.erase(is_proxy ? "Proxy-Authorization"
                                       : "Authorization");
        // 插入新的摘要认证头部信息到请求中
        new_req.headers.insert(detail::make_digest_authentication_header(
            req, auth, new_req.authorization_count_, detail::random_string(10),
            username, password, is_proxy));

        // 创建一个新的响应对象
        Response new_res;

        // 发送新的请求，接收新的响应
        ret = send(new_req, new_res, error);
        // 如果发送成功，更新原始响应为新的响应
        if (ret) { res = new_res; }
      }
    }
// 结束预处理指令
#endif

// 返回结果
return ret;
}

// 重定向函数
inline bool ClientImpl::redirect(Request &req, Response &res, Error &error) {
  // 如果重定向次数为0，返回错误
  if (req.redirect_count_ == 0) {
    error = Error::ExceedRedirectCount;
    return false;
  }

  // 获取重定向地址
  auto location = res.get_header_value("location");
  if (location.empty()) { return false; }

  // 定义正则表达式匹配重定向地址
  const static std::regex re(
      R"((?:(https?):)?(?://(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)?([^?#]*)(\?[^#]*)?(?:#.*)?)");

  std::smatch m;
  // 匹配重定向地址
  if (!std::regex_match(location, m, re)) { return false; }

  // 获取当前协议
  auto scheme = is_ssl() ? "https" : "http";

  // 解析重定向地址的各部分信息
  auto next_scheme = m[1].str();
  auto next_host = m[2].str();
  if (next_host.empty()) { next_host = m[3].str(); }
  auto port_str = m[4].str();
  auto next_path = m[5].str();
  auto next_query = m[6].str();

  // 获取下一个端口号
  auto next_port = port_;
  if (!port_str.empty()) {
    next_port = std::stoi(port_str);
  } else if (!next_scheme.empty()) {
    next_port = next_scheme == "https" ? 443 : 80;
  }

  // 设置默认值
  if (next_scheme.empty()) { next_scheme = scheme; }
  if (next_host.empty()) { next_host = host_; }
  if (next_path.empty()) { next_path = "/"; }

  // 解码 URL，并拼接查询参数
  auto path = detail::decode_url(next_path, true) + next_query;

  // 如果重定向地址与当前地址相同，则直接重定向
  if (next_scheme == scheme && next_host == host_ && next_port == port_) {
    return detail::redirect(*this, req, res, path, location, error);
  } else {
    // 如果是 HTTPS 请求
    if (next_scheme == "https") {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 创建 SSL 客户端对象
      SSLClient cli(next_host.c_str(), next_port);
      cli.copy_settings(*this);
      if (ca_cert_store_) { cli.set_ca_cert_store(ca_cert_store_); }
      return detail::redirect(cli, req, res, path, location, error);
#else
      return false;
#endif
    } else {
      // 创建普通客户端对象
      ClientImpl cli(next_host.c_str(), next_port);
      cli.copy_settings(*this);
      return detail::redirect(cli, req, res, path, location, error);
    }
  }
}
inline bool ClientImpl::write_content_with_provider(Stream &strm,
                                                    const Request &req,
                                                    Error &error) {
  // 定义一个 lambda 函数，用于检查是否正在关闭连接
  auto is_shutting_down = []() { return false; };

  // 如果请求使用分块内容提供程序
  if (req.is_chunked_content_provider_) {
    // TODO: Brotli 支持
    // 创建一个压缩器对象
    std::unique_ptr<detail::compressor> compressor;
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
    // 如果启用了压缩功能
    if (compress_) {
      // 使用 gzip 压缩算法
      compressor = detail::make_unique<detail::gzip_compressor>();
    } else
#endif
    {
      // 否则使用无压缩算法
      compressor = detail::make_unique<detail::nocompressor>();
    }

    // 写入分块内容
    return detail::write_content_chunked(strm, req.content_provider_,
                                         is_shutting_down, *compressor, error);
  } else {
    // 写入非分块内容
    return detail::write_content(strm, req.content_provider_, 0,
                                 req.content_length_, is_shutting_down, error);
  }
}

inline bool ClientImpl::write_request(Stream &strm, Request &req,
                                      bool close_connection, Error &error) {
  // 准备额外的请求头部信息
  // 如果需要关闭连接
  if (close_connection) {
    // 如果请求中没有 Connection 头部
    if (!req.has_header("Connection")) {
      // 设置 Connection 头部为 close
      req.set_header("Connection", "close");
    }
  }

  // 如果请求中没有 Host 头部
  if (!req.has_header("Host")) {
    // 如果是 SSL 连接
    if (is_ssl()) {
      // 如果端口是 443
      if (port_ == 443) {
        // 设置 Host 头部为 host_
        req.set_header("Host", host_);
      } else {
        // 设置 Host 头部为 host_and_port_
        req.set_header("Host", host_and_port_);
      }
    } else {
      // 如果端口是 80
      if (port_ == 80) {
        // 设置 Host 头部为 host_
        req.set_header("Host", host_);
      } else {
        // 设置 Host 头部为 host_and_port_
        req.set_header("Host", host_and_port_);
      }
    }
  }

  // 如果请求中没有 Accept 头部
  if (!req.has_header("Accept")) { req.set_header("Accept", "*/*"); }

  // 如果未定义 CPPHTTPLIB_NO_DEFAULT_USER_AGENT
#ifndef CPPHTTPLIB_NO_DEFAULT_USER_AGENT
  // 如果请求中没有 User-Agent 头部
  if (!req.has_header("User-Agent")) {
    // 设置 User-Agent 头部为 cpp-httplib 版本号
    auto agent = std::string("cpp-httplib/") + CPPHTTPLIB_VERSION;
    req.set_header("User-Agent", agent);
  }
#endif

  // 如果请求体为空
  if (req.body.empty()) {
    // 检查请求是否有内容提供者
    if (req.content_provider_) {
      // 如果请求不是分块内容提供者
      if (!req.is_chunked_content_provider_) {
        // 如果请求没有设置 "Content-Length" 头
        if (!req.has_header("Content-Length")) {
          // 将请求的内容长度转换为字符串，并设置为 "Content-Length" 头
          auto length = std::to_string(req.content_length_);
          req.set_header("Content-Length", length);
        }
      }
    } else {
      // 如果请求没有内容提供者
      if (req.method == "POST" || req.method == "PUT" ||
          req.method == "PATCH") {
        // 对于 POST、PUT、PATCH 方法，设置 "Content-Length" 头为 "0"
        req.set_header("Content-Length", "0");
      }
    }
  } else {
    // 如果请求没有设置 "Content-Type" 头，设置为 "text/plain"
    if (!req.has_header("Content-Type")) {
      req.set_header("Content-Type", "text/plain");
    }

    // 如果请求没有设置 "Content-Length" 头
    if (!req.has_header("Content-Length")) {
      // 将请求体的大小转换为字符串，并设置为 "Content-Length" 头
      auto length = std::to_string(req.body.size());
      req.set_header("Content-Length", length);
    }
  }

  // 如果有基本身份验证的用户名和密码
  if (!basic_auth_password_.empty() || !basic_auth_username_.empty()) {
    // 如果请求没有设置 "Authorization" 头，插入基本身份验证头
    if (!req.has_header("Authorization")) {
      req.headers.insert(make_basic_authentication_header(
          basic_auth_username_, basic_auth_password_, false));
    }
  }

  // 如果有代理基本身份验证的用户名和密码
  if (!proxy_basic_auth_username_.empty() &&
      !proxy_basic_auth_password_.empty()) {
    // 如果请求没有设置 "Proxy-Authorization" 头，插入代理基本身份验证头
    if (!req.has_header("Proxy-Authorization")) {
      req.headers.insert(make_basic_authentication_header(
          proxy_basic_auth_username_, proxy_basic_auth_password_, true));
    }
  }

  // 如果有 Bearer Token 身份验证的令牌
  if (!bearer_token_auth_token_.empty()) {
    // 如果请求没有设置 "Authorization" 头，插入 Bearer Token 身份验证头
    if (!req.has_header("Authorization")) {
      req.headers.insert(make_bearer_token_authentication_header(
          bearer_token_auth_token_, false));
    }
  }

  // 如果有代理 Bearer Token 身份验证的令牌
  if (!proxy_bearer_token_auth_token_.empty()) {
    // 如果请求没有设置 "Proxy-Authorization" 头，插入代理 Bearer Token 身份验证头
    if (!req.has_header("Proxy-Authorization")) {
      req.headers.insert(make_bearer_token_authentication_header(
          proxy_bearer_token_auth_token_, true));
    }
  }

  // 请求行和头部
  {
    // 创建缓冲流对象
    detail::BufferStream bstrm;

    // 对路径进行 URL 编码（如果需要），并写入请求方法和路径到缓冲流
    const auto &path = url_encode_ ? detail::encode_url(req.path) : req.path;
    bstrm.write_format("%s %s HTTP/1.1\r\n", req.method.c_str(), path.c_str());

    // 写入头部到缓冲流
    header_writer_(bstrm, req.headers);

    // 刷新缓冲区
    auto &data = bstrm.get_buffer();
    // 如果写入数据失败，则设置错误类型为写入错误，并返回 false
    if (!detail::write_data(strm, data.data(), data.size())) {
      error = Error::Write;
      return false;
    }
  }

  // 如果请求体为空，则使用提供者写入内容
  if (req.body.empty()) {
    return write_content_with_provider(strm, req, error);
  }

  // 如果写入请求体数据失败，则设置错误类型为写入错误，并返回 false
  if (!detail::write_data(strm, req.body.data(), req.body.size())) {
    error = Error::Write;
    return false;
  }

  // 返回 true 表示写入成功
  return true;
// 结束 ClientImpl 类的定义
}

// 使用内容提供者发送请求，支持有和无长度的内容提供者
inline std::unique_ptr<Response> ClientImpl::send_with_content_provider(
    Request &req, const char *body, size_t content_length,
    ContentProvider content_provider,
    ContentProviderWithoutLength content_provider_without_length,
    const std::string &content_type, Error &error) {
  // 如果内容类型不为空，设置请求头的 Content-Type
  if (!content_type.empty()) { req.set_header("Content-Type", content_type); }

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持压缩，设置请求头的 Content-Encoding 为 gzip
  if (compress_) { req.set_header("Content-Encoding", "gzip"); }
#endif

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持压缩并且内容提供者没有长度
  if (compress_ && !content_provider_without_length) {
    // TODO: Brotli 支持
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

          // 压缩数据并将压缩后的数据追加到请求体中
          auto ret = compressor.compress(
              data, data_len, last,
              [&](const char *compressed_data, size_t compressed_data_len) {
                req.body.append(compressed_data, compressed_data_len);
                return true;
              });

          if (ret) {
            offset += data_len;
          } else {
            ok = false;
          }
        }
        return ok;
      };

      // 循环调用内容提供者，直到所有数据都被处理
      while (ok && offset < content_length) {
        if (!content_provider(offset, content_length - offset, data_sink)) {
          error = Error::Canceled;
          return nullptr;
        }
      }
    } else {
      // 如果没有内容提供者，直接压缩请求体中的数据
      if (!compressor.compress(body, content_length, true,
                               [&](const char *data, size_t data_len) {
                                 req.body.append(data, data_len);
                                 return true;
                               })) {
        error = Error::Compression;
        return nullptr;
      }
    }
  } else
#endif
  {
    // 如果有 content_provider，则设置请求的内容长度和内容提供者，并标记为非分块内容提供者
    if (content_provider) {
      req.content_length_ = content_length;
      req.content_provider_ = std::move(content_provider);
      req.is_chunked_content_provider_ = false;
    // 如果没有 content_provider 但有 content_provider_without_length，则设置请求的内容长度为 0，设置内容提供者和标记为分块内容提供者，并设置 Transfer-Encoding 为 chunked
    } else if (content_provider_without_length) {
      req.content_length_ = 0;
      req.content_provider_ = detail::ContentProviderAdapter(
          std::move(content_provider_without_length));
      req.is_chunked_content_provider_ = true;
      req.set_header("Transfer-Encoding", "chunked");
    // 如果既没有 content_provider 也没有 content_provider_without_length，则将 body 的内容和长度赋给请求的 body
    } else {
      req.body.assign(body, content_length);
    }
  }

  // 创建一个新的响应对象
  auto res = detail::make_unique<Response>();
  // 发送请求并返回响应，如果发送成功则返回响应对象，否则返回空指针
  return send(req, *res, error) ? std::move(res) : nullptr;
// 发送带有内容提供程序的请求，返回结果对象
inline Result ClientImpl::send_with_content_provider(
    const std::string &method, const std::string &path, const Headers &headers,
    const char *body, size_t content_length, ContentProvider content_provider,
    ContentProviderWithoutLength content_provider_without_length,
    const std::string &content_type) {
  // 创建请求对象
  Request req;
  req.method = method;
  req.headers = headers;
  req.path = path;

  // 初始化错误状态
  auto error = Error::Success;

  // 发送带有内容提供程序的请求
  auto res = send_with_content_provider(
      req, body, content_length, std::move(content_provider),
      std::move(content_provider_without_length), content_type, error);

  // 返回结果对象
  return Result{std::move(res), error, std::move(req.headers)};
}

// 调整主机字符串，如果包含端口号则加上方括号
inline std::string
ClientImpl::adjust_host_string(const std::string &host) const {
  if (host.find(':') != std::string::npos) { return "[" + host + "]"; }
  return host;
}

// 处理请求，发送请求并接收响应
inline bool ClientImpl::process_request(Stream &strm, Request &req,
                                        Response &res, bool close_connection,
                                        Error &error) {
  // 发送请求
  if (!write_request(strm, req, close_connection, error)) { return false; }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果使用 SSL
  if (is_ssl()) {
    // 检查是否启用代理
    auto is_proxy_enabled = !proxy_host_.empty() && proxy_port_ != -1;
    // 如果未启用代理
    if (!is_proxy_enabled) {
      // 检查 SSL 连接是否已关闭
      char buf[1];
      if (SSL_peek(socket_.ssl, buf, 1) == 0 &&
          SSL_get_error(socket_.ssl, 0) == SSL_ERROR_ZERO_RETURN) {
        error = Error::SSLPeerCouldBeClosed_;
        return false;
      }
    }
  }
#endif

  // 接收响应行和头部
  if (!read_response_line(strm, req, res) ||
      !detail::read_headers(strm, res.headers)) {
    error = Error::Read;
    return false;
  }

  // 处理响应体
  if ((res.status != 204) && req.method != "HEAD" && req.method != "CONNECT") {
    // 处理重定向
    auto redirect = 300 < res.status && res.status < 400 && follow_location_;
    // 如果请求有响应处理函数且不是重定向请求
    if (req.response_handler && !redirect) {
      // 调用响应处理函数，如果返回 false，则设置错误为取消并返回 false
      if (!req.response_handler(res)) {
        error = Error::Canceled;
        return false;
      }
    }

    // 创建输出函数对象，根据是否有内容接收器选择不同的处理方式
    auto out =
        req.content_receiver
            ? static_cast<ContentReceiverWithProgress>(
                  [&](const char *buf, size_t n, uint64_t off, uint64_t len) {
                    // 如果是重定向请求，直接返回 true
                    if (redirect) { return true; }
                    // 调用内容接收器函数，如果返回 false，则设置错误为取消
                    auto ret = req.content_receiver(buf, n, off, len);
                    if (!ret) { error = Error::Canceled; }
                    return ret;
                  })
            : static_cast<ContentReceiverWithProgress>(
                  [&](const char *buf, size_t n, uint64_t /*off*/,
                      uint64_t /*len*/) {
                    // 如果响应体大小超过最大限制，返回 false
                    if (res.body.size() + n > res.body.max_size()) {
                      return false;
                    }
                    // 将数据追加到响应体中
                    res.body.append(buf, n);
                    return true;
                  });

    // 创建进度函数对象
    auto progress = [&](uint64_t current, uint64_t total) {
      // 如果没有进度函数或是重定向请求，直接返回 true
      if (!req.progress || redirect) { return true; }
      // 调用进度函数，如果返回 false，则设置错误为取消
      auto ret = req.progress(current, total);
      if (!ret) { error = Error::Canceled; }
      return ret;
    };

    // 读取内容并处理
    int dummy_status;
    if (!detail::read_content(strm, res, (std::numeric_limits<size_t>::max)(),
                              dummy_status, std::move(progress), std::move(out),
                              decompress_)) {
      // 如果读取内容失败且错误不是取消，则设置错误为读取错误并返回 false
      if (error != Error::Canceled) { error = Error::Read; }
      return false;
    }
  }

  // 记录日志
  if (logger_) { logger_(req, res); }

  // 返回 true
  return true;
// 返回一个函数对象，该函数对象用于生成多部分内容，维护当前项和当前起始位置的状态
inline ContentProviderWithoutLength ClientImpl::get_multipart_content_provider(
    const std::string &boundary, const MultipartFormDataItems &items,
    const MultipartFormDataProviderItems &provider_items) {
  size_t cur_item = 0, cur_start = 0;
  // cur_item 和 cur_start 被复制到 std::function 中，并在连续调用之间保持状态
  return [&, cur_item, cur_start](size_t offset,
                                  DataSink &sink) mutable -> bool {
    // 如果偏移量为0且items不为空，则将多部分表单数据序列化到输出流中
    if (!offset && items.size()) {
      sink.os << detail::serialize_multipart_formdata(items, boundary, false);
      return true;
    } else if (cur_item < provider_items.size()) {
      if (!cur_start) {
        // 序列化多部分表单数据项的开始部分到输出流中
        const auto &begin = detail::serialize_multipart_formdata_item_begin(
            provider_items[cur_item], boundary);
        offset += begin.size();
        cur_start = offset;
        sink.os << begin;
      }

      DataSink cur_sink;
      auto has_data = true;
      cur_sink.write = sink.write;
      cur_sink.done = [&]() { has_data = false; };

      // 如果提供者项的提供者函数返回false，则返回false
      if (!provider_items[cur_item].provider(offset - cur_start, cur_sink))
        return false;

      if (!has_data) {
        // 将多部分表单数据项的结束部分序列化到输出流中
        sink.os << detail::serialize_multipart_formdata_item_end();
        cur_item++;
        cur_start = 0;
      }
      return true;
    } else {
      // 将多部分表单数据的结束部分序列化到输出流中，并调用sink.done()
      sink.os << detail::serialize_multipart_formdata_finish(boundary);
      sink.done();
      return true;
    }
  };
}

// 处理套接字，调用回调函数
inline bool
ClientImpl::process_socket(const Socket &socket,
                           std::function<bool(Stream &strm)> callback) {
  return detail::process_client_socket(
      socket.sock, read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
      write_timeout_usec_, std::move(callback));
}

// 返回是否为SSL连接
inline bool ClientImpl::is_ssl() const { return false; }

// 发送GET请求
inline Result ClientImpl::Get(const std::string &path) {
  return Get(path, Headers(), Progress());
}
// 使用给定路径和进度函数调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path, Progress progress) {
  return Get(path, Headers(), std::move(progress));
}

// 使用给定路径和头部信息调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path, const Headers &headers) {
  return Get(path, headers, Progress());
}

// 使用给定路径、头部信息和进度函数调用 send_ 函数，返回结果
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              Progress progress) {
  // 创建请求对象
  Request req;
  req.method = "GET";
  req.path = path;
  req.headers = headers;
  req.progress = std::move(progress);

  // 调用 send_ 函数并返回结果
  return send_(std::move(req));
}

// 使用给定路径和内容接收器调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver) {
  return Get(path, Headers(), nullptr, std::move(content_receiver), nullptr);
}

// 使用给定路径、内容接收器和进度函数调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver,
                              Progress progress) {
  return Get(path, Headers(), nullptr, std::move(content_receiver),
             std::move(progress));
}

// 使用给定路径、头部信息和内容接收器调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver) {
  return Get(path, headers, nullptr, std::move(content_receiver), nullptr);
}

// 使用给定路径、头部信息、内容接收器和进度函数调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver,
                              Progress progress) {
  return Get(path, headers, nullptr, std::move(content_receiver),
             std::move(progress));
}

// 使用给定路径、响应处理器、内容接收器调用重载的 Get 函数，返回结果
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), nullptr);
}
# 定义 ClientImpl 类的 Get 方法，用于发送 GET 请求并接收响应
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  # 调用重载的 Get 方法，传入参数并返回结果
  return Get(path, headers, std::move(response_handler),
             std::move(content_receiver), nullptr);
}

# 定义 ClientImpl 类的 Get 方法的重载，用于发送 GET 请求并接收响应
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 调用重载的 Get 方法，传入参数并返回结果
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), std::move(progress));
}

# 定义 ClientImpl 类的 Get 方法的重载，用于发送 GET 请求并接收响应
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 创建请求对象 req
  Request req;
  # 设置请求方法为 GET
  req.method = "GET";
  # 设置请求路径
  req.path = path;
  # 设置请求头部
  req.headers = headers;
  # 移动响应处理器到请求对象
  req.response_handler = std::move(response_handler);
  # 设置内容接收器，将数据传递给 content_receiver
  req.content_receiver =
      [content_receiver](const char *data, size_t data_length,
                         uint64_t /*offset*/, uint64_t /*total_length*/) {
        return content_receiver(data, data_length);
      };
  # 移动进度对象到请求对象
  req.progress = std::move(progress);

  # 调用 send_ 方法发送请求并返回结果
  return send_(std::move(req));
}

# 定义 ClientImpl 类的 Get 方法的重载，用于发送 GET 请求并接收响应
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers, Progress progress) {
  # 如果参数为空，则调用 Get 方法并返回结果
  if (params.empty()) { return Get(path, headers); }
  
  # 将参数拼接到路径中
  std::string path_with_query = append_query_params(path, params);
  # 调用 Get 方法并返回结果
  return Get(path_with_query.c_str(), headers, progress);
}
# 定义 ClientImpl 类的 Get 方法，用于发送 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 调用另一个 Get 方法，传入参数并返回结果
  return Get(path, params, headers, nullptr, content_receiver, progress);
}

# 定义 ClientImpl 类的 Get 方法，用于发送 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 如果参数为空，则调用另一个 Get 方法，传入参数并返回结果
  if (params.empty()) {
    return Get(path, headers, response_handler, content_receiver, progress);
  }

  # 将参数拼接到路径中，然后调用另一个 Get 方法，传入参数并返回结果
  std::string path_with_query = append_query_params(path, params);
  return Get(path_with_query.c_str(), headers, response_handler,
             content_receiver, progress);
}

# 定义 ClientImpl 类的 Head 方法，用于发送 HEAD 请求
inline Result ClientImpl::Head(const std::string &path) {
  # 调用另一个 Head 方法，传入参数并返回结果
  return Head(path, Headers());
}

# 定义 ClientImpl 类的 Head 方法，用于发送 HEAD 请求
inline Result ClientImpl::Head(const std::string &path,
                               const Headers &headers) {
  # 创建请求对象，设置方法为 HEAD，设置头部信息和路径
  Request req;
  req.method = "HEAD";
  req.headers = headers;
  req.path = path;

  # 调用 send_ 方法，传入请求对象并返回结果
  return send_(std::move(req));
}

# 定义 ClientImpl 类的 Post 方法，用于发送 POST 请求
inline Result ClientImpl::Post(const std::string &path) {
  # 调用另一个 Post 方法，传入参数并返回结果
  return Post(path, std::string(), std::string());
}

# 定义 ClientImpl 类的 Post 方法，用于发送 POST 请求
inline Result ClientImpl::Post(const std::string &path,
                               const Headers &headers) {
  # 调用另一个 Post 方法，传入参数并返回结果
  return Post(path, headers, nullptr, 0, std::string());
}

# 定义 ClientImpl 类的 Post 方法，用于发送 POST 请求
inline Result ClientImpl::Post(const std::string &path, const char *body,
                               size_t content_length,
                               const std::string &content_type) {
  # 调用另一个 Post 方法，传入参数并返回结果
  return Post(path, Headers(), body, content_length, content_type);
}
// 发送 POST 请求，包含路径、请求头、请求体、请求体长度和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const char *body, size_t content_length,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}

// 发送 POST 请求，包含路径、请求体和内容类型
inline Result ClientImpl::Post(const std::string &path, const std::string &body,
                               const std::string &content_type) {
  return Post(path, Headers(), body, content_type);
}

// 发送 POST 请求，包含路径、请求头、请求体和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const std::string &body,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

// 发送 POST 请求，包含路径和参数
inline Result ClientImpl::Post(const std::string &path, const Params &params) {
  return Post(path, Headers(), params);
}

// 发送 POST 请求，包含路径、请求体长度、内容提供者和内容类型
inline Result ClientImpl::Post(const std::string &path, size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  return Post(path, Headers(), content_length, std::move(content_provider),
              content_type);
}

// 发送 POST 请求，包含路径、无长度的内容提供者和内容类型
inline Result ClientImpl::Post(const std::string &path,
                               ContentProviderWithoutLength content_provider,
                               const std::string &content_type) {
  return Post(path, Headers(), std::move(content_provider), content_type);
}
// 发送 POST 请求，包含内容长度，内容提供者，内容类型等信息
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

// 发送 POST 请求，不包含内容长度，但包含内容提供者和内容类型等信息
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               ContentProviderWithoutLength content_provider,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 发送 POST 请求，包含参数并将参数转换为查询字符串
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const Params &params) {
  auto query = detail::params_to_query_str(params);
  return Post(path, headers, query, "application/x-www-form-urlencoded");
}

// 发送 POST 请求，包含多部分表单数据项
inline Result ClientImpl::Post(const std::string &path,
                               const MultipartFormDataItems &items) {
  return Post(path, Headers(), items);
}

// 发送 POST 请求，包含多部分表单数据项，边界，内容类型和内容体
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const MultipartFormDataItems &items) {
  const auto &boundary = detail::make_multipart_data_boundary();
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Post(path, headers, body, content_type.c_str());
}
# 发送 POST 请求，包含路径、请求头、多部分表单数据项、边界
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const MultipartFormDataItems &items,
                               const std::string &boundary) {
  # 检查多部分边界字符是否有效，如果无效则返回错误
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    return Result{nullptr, Error::UnsupportedMultipartBoundaryChars};
  }

  # 序列化多部分表单数据获取内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  # 序列化多部分表单数据，获取请求体
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  # 发送 POST 请求，包含路径、请求头、请求体、内容类型
  return Post(path, headers, body, content_type.c_str());
}

# 发送 POST 请求，包含路径、请求头、多部分表单数据项、多部分表单数据提供者项
inline Result
ClientImpl::Post(const std::string &path, const Headers &headers,
                 const MultipartFormDataItems &items,
                 const MultipartFormDataProviderItems &provider_items) {
  # 生成多部分数据边界
  const auto &boundary = detail::make_multipart_data_boundary();
  # 序列化多部分表单数据获取内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  # 使用内容提供者发送请求
  return send_with_content_provider(
      "POST", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}

# 发送 PUT 请求，只包含路径
inline Result ClientImpl::Put(const std::string &path) {
  return Put(path, std::string(), std::string());
}

# 发送 PUT 请求，包含路径、请求体、内容长度、内容类型
inline Result ClientImpl::Put(const std::string &path, const char *body,
                              size_t content_length,
                              const std::string &content_type) {
  return Put(path, Headers(), body, content_length, content_type);
}

# 发送 PUT 请求，包含路径、请求头、请求体、内容长度、内容类型
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const char *body, size_t content_length,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}
// 使用默认的头部信息和请求体内容发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const std::string &body,
                              const std::string &content_type) {
  return Put(path, Headers(), body, content_type);
}

// 使用指定的头部信息、请求体内容发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const std::string &body,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

// 使用默认的头部信息和内容提供者发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  return Put(path, Headers(), content_length, std::move(content_provider),
             content_type);
}

// 使用默认的头部信息和无长度内容提供者发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  return Put(path, Headers(), std::move(content_provider), content_type);
}

// 使用指定的头部信息、内容长度、内容提供者发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}
// 使用 PUT 方法将内容提供者的数据上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 使用 PUT 方法将参数上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path, const Params &params) {
  return Put(path, Headers(), params);
}

// 使用 PUT 方法将参数和头部信息上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const Params &params) {
  auto query = detail::params_to_query_str(params);
  return Put(path, headers, query, "application/x-www-form-urlencoded");
}

// 使用 PUT 方法将多部分表单数据上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path,
                              const MultipartFormDataItems &items) {
  return Put(path, Headers(), items);
}

// 使用 PUT 方法将多部分表单数据和头部信息上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items) {
  // 生成多部分数据边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 获取多部分数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分数据
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Put(path, headers, body, content_type);
}

// 使用 PUT 方法将多部分表单数据、头部信息和指定边界上传到指定路径，返回结果
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items,
                              const std::string &boundary) {
  // 检查多部分数据边界字符是否有效
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    return Result{nullptr, Error::UnsupportedMultipartBoundaryChars};
  }

  // 获取多部分数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分数据
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Put(path, headers, body, content_type);
}

// 继续下一个函数的定义
inline Result
// 在客户端实现中执行 PUT 请求，发送多部分表单数据
ClientImpl::Put(const std::string &path, const Headers &headers,
                const MultipartFormDataItems &items,
                const MultipartFormDataProviderItems &provider_items) {
  // 生成多部分数据的边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 序列化多部分表单数据并获取内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 使用内容提供程序发送请求
  return send_with_content_provider(
      "PUT", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}

// 发送 PATCH 请求的重载函数，不带任何数据
inline Result ClientImpl::Patch(const std::string &path) {
  return Patch(path, std::string(), std::string());
}

// 发送 PATCH 请求的重载函数，带有字符数组数据
inline Result ClientImpl::Patch(const std::string &path, const char *body,
                                size_t content_length,
                                const std::string &content_type) {
  return Patch(path, Headers(), body, content_length, content_type);
}

// 发送 PATCH 请求的重载函数，带有字符串数据
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const char *body, size_t content_length,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, body,
                                    content_length, nullptr, nullptr,
                                    content_type);
}

// 发送 PATCH 请求的重载函数，带有字符串数据
inline Result ClientImpl::Patch(const std::string &path,
                                const std::string &body,
                                const std::string &content_type) {
  return Patch(path, Headers(), body, content_type);
}

// 发送 PATCH 请求的重载函数，带有字符串数据和头部信息
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const std::string &body,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}
// 使用给定的路径、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, size_t content_length,
                                ContentProvider content_provider,
                                const std::string &content_type) {
  return Patch(path, Headers(), content_length, std::move(content_provider),
               content_type);
}

// 使用给定的路径、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path,
                                ContentProviderWithoutLength content_provider,
                                const std::string &content_type) {
  return Patch(path, Headers(), std::move(content_provider), content_type);
}

// 使用给定的路径、头部、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                size_t content_length,
                                ContentProvider content_provider,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

// 使用给定的路径、头部、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                ContentProviderWithoutLength content_provider,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 使用给定的路径执行 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path) {
  return Delete(path, Headers(), std::string(), std::string());
}

// 使用给定的路径和头部执行 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers) {
  return Delete(path, headers, std::string(), std::string());
}
// 使用默认的请求头和请求体内容发送 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  return Delete(path, Headers(), body, content_length, content_type);
}

// 发送 DELETE 请求，可以指定请求头和请求体内容
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  // 创建请求对象
  Request req;
  req.method = "DELETE";
  req.headers = headers;
  req.path = path;

  // 如果请求体内容类型不为空，设置请求头的 Content-Type
  if (!content_type.empty()) { req.set_header("Content-Type", content_type); }
  // 将请求体内容添加到请求对象中
  req.body.assign(body, content_length);

  // 发送请求并返回结果
  return send_(std::move(req));
}

// 发送 DELETE 请求，请求体内容为字符串
inline Result ClientImpl::Delete(const std::string &path,
                                 const std::string &body,
                                 const std::string &content_type) {
  return Delete(path, Headers(), body.data(), body.size(), content_type);
}

// 发送 DELETE 请求，可以指定请求头和请求体内容为字符串
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers,
                                 const std::string &body,
                                 const std::string &content_type) {
  return Delete(path, headers, body.data(), body.size(), content_type);
}

// 使用默认的请求头发送 OPTIONS 请求
inline Result ClientImpl::Options(const std::string &path) {
  return Options(path, Headers());
}

// 发送 OPTIONS 请求，可以指定请求头
inline Result ClientImpl::Options(const std::string &path,
                                  const Headers &headers) {
  // 创建请求对象
  Request req;
  req.method = "OPTIONS";
  req.headers = headers;
  req.path = path;

  // 发送请求并返回结果
  return send_(std::move(req));
}
// 停止客户端操作，包括关闭 socket
inline void ClientImpl::stop() {
  // 使用互斥锁保护 socket 操作
  std::lock_guard<std::mutex> guard(socket_mutex_);

  // 如果有正在进行的请求，只能安全地关闭 socket，其他操作不安全
  if (socket_requests_in_flight_ > 0) {
    // 关闭 socket
    shutdown_socket(socket_);

    // 设置标志，请求完成后关闭 socket
    socket_should_be_closed_when_request_is_done_ = true;
    return;
  }

  // 否则，仍然持有互斥锁，可以自行关闭所有内容
  shutdown_ssl(socket_, true);
  shutdown_socket(socket_);
  close_socket(socket_);
}

// 返回主机地址
inline std::string ClientImpl::host() const { return host_; }

// 返回端口号
inline int ClientImpl::port() const { return port_; }

// 返回 socket 是否打开
inline size_t ClientImpl::is_socket_open() const {
  std::lock_guard<std::mutex> guard(socket_mutex_);
  return socket_.is_open();
}

// 返回 socket
inline socket_t ClientImpl::socket() const { return socket_.sock; }

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

// 设置写入超时时间
inline void ClientImpl::set_write_timeout(time_t sec, time_t usec) {
  write_timeout_sec_ = sec;
  write_timeout_usec_ = usec;
}

// 设置基本认证信息
inline void ClientImpl::set_basic_auth(const std::string &username,
                                       const std::string &password) {
  basic_auth_username_ = username;
  basic_auth_password_ = password;
}

// 设置 Bearer Token 认证信息
inline void ClientImpl::set_bearer_token_auth(const std::string &token) {
  bearer_token_auth_token_ = token;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置摘要认证的用户名和密码
inline void ClientImpl::set_digest_auth(const std::string &username,
                                        const std::string &password) {
  digest_auth_username_ = username;
  digest_auth_password_ = password;
}

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

// 设置默认请求头
inline void ClientImpl::set_default_headers(Headers headers) {
  default_headers_ = std::move(headers);
}

// 设置自定义请求头写入函数
inline void ClientImpl::set_header_writer(
    std::function<ssize_t(Stream &, Headers &)> const &writer) {
  header_writer_ = writer;
}

// 设置地址族
inline void ClientImpl::set_address_family(int family) {
  address_family_ = family;
}

// 设置是否启用 TCP_NODELAY
inline void ClientImpl::set_tcp_nodelay(bool on) { tcp_nodelay_ = on; }

// 设置套接字选项
inline void ClientImpl::set_socket_options(SocketOptions socket_options) {
  socket_options_ = std::move(socket_options);
}

// 设置是否启用压缩
inline void ClientImpl::set_compress(bool on) { compress_ = on; }

// 设置是否启用解压缩
inline void ClientImpl::set_decompress(bool on) { decompress_ = on; }

// 设置网络接口
inline void ClientImpl::set_interface(const std::string &intf) {
  interface_ = intf;
}

// 设置代理主机和端口
inline void ClientImpl::set_proxy(const std::string &host, int port) {
  proxy_host_ = host;
  proxy_port_ = port;
}

// 设置代理基本认证的用户名和密码
inline void ClientImpl::set_proxy_basic_auth(const std::string &username,
                                             const std::string &password) {
  proxy_basic_auth_username_ = username;
  proxy_basic_auth_password_ = password;
}

// 设置代理 Bearer Token 认证的 Token
inline void ClientImpl::set_proxy_bearer_token_auth(const std::string &token) {
  proxy_bearer_token_auth_token_ = token;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理服务器的摘要认证用户名和密码
inline void ClientImpl::set_proxy_digest_auth(const std::string &username,
                                              const std::string &password) {
  proxy_digest_auth_username_ = username;
  proxy_digest_auth_password_ = password;
}

// 设置 CA 证书路径
inline void ClientImpl::set_ca_cert_path(const std::string &ca_cert_file_path,
                                         const std::string &ca_cert_dir_path) {
  ca_cert_file_path_ = ca_cert_file_path;
  ca_cert_dir_path_ = ca_cert_dir_path;
}

// 设置 CA 证书存储
inline void ClientImpl::set_ca_cert_store(X509_STORE *ca_cert_store) {
  if (ca_cert_store && ca_cert_store != ca_cert_store_) {
    ca_cert_store_ = ca_cert_store;
  }
}

// 创建 CA 证书存储
inline X509_STORE *ClientImpl::create_ca_cert_store(const char *ca_cert,
                                                    std::size_t size) {
  auto mem = BIO_new_mem_buf(ca_cert, static_cast<int>(size));
  if (!mem) return nullptr;

  auto inf = PEM_X509_INFO_read_bio(mem, nullptr, nullptr, nullptr);
  if (!inf) {
    BIO_free_all(mem);
    return nullptr;
  }

  auto cts = X509_STORE_new();
  if (cts) {
    for (auto i = 0; i < static_cast<int>(sk_X509_INFO_num(inf)); i++) {
      auto itmp = sk_X509_INFO_value(inf, i);
      if (!itmp) { continue; }

      if (itmp->x509) { X509_STORE_add_cert(cts, itmp->x509); }
      if (itmp->crl) { X509_STORE_add_crl(cts, itmp->crl); }
    }
  }

  sk_X509_INFO_pop_free(inf, X509_INFO_free);
  BIO_free_all(mem);
  return cts;
}

// 启用/禁用服务器证书验证
inline void ClientImpl::enable_server_certificate_verification(bool enabled) {
  server_certificate_verification_ = enabled;
}

// 设置日志记录器
inline void ClientImpl::set_logger(Logger logger) {
  logger_ = std::move(logger);
}

/*
 * SSL Implementation
 */
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
namespace detail {

// SSL 新建连接或接受连接
template <typename U, typename V>
inline SSL *ssl_new(socket_t sock, SSL_CTX *ctx, std::mutex &ctx_mutex,
                    U SSL_connect_or_accept, V setup) {
  SSL *ssl = nullptr;
  {
    std::lock_guard<std::mutex> guard(ctx_mutex);
    // 使用 SSL 上下文创建一个 SSL 对象
    ssl = SSL_new(ctx);
  }

  // 如果 SSL 对象存在
  if (ssl) {
    // 将套接字设置为非阻塞模式
    set_nonblocking(sock, true);
    // 创建一个与套接字相关联的 BIO 对象
    auto bio = BIO_new_socket(static_cast<int>(sock), BIO_NOCLOSE);
    // 将 BIO 对象设置为非阻塞模式
    BIO_set_nbio(bio, 1);
    // 将 SSL 对象与 BIO 对象关联
    SSL_set_bio(ssl, bio, bio);

    // 如果 SSL 设置或连接失败
    if (!setup(ssl) || SSL_connect_or_accept(ssl) != 1) {
      // 关闭 SSL 连接
      SSL_shutdown(ssl);
      {
        // 使用互斥锁保护 SSL 对象的释放
        std::lock_guard<std::mutex> guard(ctx_mutex);
        // 释放 SSL 对象
        SSL_free(ssl);
      }
      // 将套接字设置为阻塞模式
      set_nonblocking(sock, false);
      // 返回空指针
      return nullptr;
    }
    // 将 BIO 对象设置为阻塞模式
    BIO_set_nbio(bio, 0);
    // 将套接字设置为阻塞模式
    set_nonblocking(sock, false);
  }

  // 返回 SSL 对象
  return ssl;
}

// 删除 SSL 连接，可选择是否优雅关闭连接
inline void ssl_delete(std::mutex &ctx_mutex, SSL *ssl,
                       bool shutdown_gracefully) {
  // 有时我们可能希望跳过这一步，尝试避免 SIGPIPE，如果我们知道远程已关闭网络连接
  // 请注意，无法始终避免 SIGPIPE，这仅仅是尽力而为
  if (shutdown_gracefully) { SSL_shutdown(ssl); }

  // 加锁，保护 SSL 对象
  std::lock_guard<std::mutex> guard(ctx_mutex);
  // 释放 SSL 对象
  SSL_free(ssl);
}

// SSL 连接或接受非阻塞函数模板
template <typename U>
bool ssl_connect_or_accept_nonblocking(socket_t sock, SSL *ssl,
                                       U ssl_connect_or_accept,
                                       time_t timeout_sec,
                                       time_t timeout_usec) {
  auto res = 0;
  // 循环直到 SSL 连接或接受成功
  while ((res = ssl_connect_or_accept(ssl)) != 1) {
    auto err = SSL_get_error(ssl, res);
    switch (err) {
    case SSL_ERROR_WANT_READ:
      // 如果需要读取，则等待可读事件
      if (select_read(sock, timeout_sec, timeout_usec) > 0) { continue; }
      break;
    case SSL_ERROR_WANT_WRITE:
      // 如果需要写入，则等待可写事件
      if (select_write(sock, timeout_sec, timeout_usec) > 0) { continue; }
      break;
    default: break;
    }
    return false;
  }
  return true;
}

// 处理服务器端 SSL 套接字函数模板
template <typename T>
inline bool process_server_socket_ssl(
    const std::atomic<socket_t> &svr_sock, SSL *ssl, socket_t sock,
    size_t keep_alive_max_count, time_t keep_alive_timeout_sec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    time_t write_timeout_usec, T callback) {
  return process_server_socket_core(
      svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
      [&](bool close_connection, bool &connection_closed) {
        // 创建 SSL 套接字流对象
        SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                             write_timeout_sec, write_timeout_usec);
        // 调用回调函数处理连接
        return callback(strm, close_connection, connection_closed);
      });
}

// 其他函数模板
template <typename T>
inline bool
// 使用 SSL 连接处理客户端套接字，设置读写超时时间，并调用回调函数处理数据
process_client_socket_ssl(SSL *ssl, socket_t sock, time_t read_timeout_sec,
                          time_t read_timeout_usec, time_t write_timeout_sec,
                          time_t write_timeout_usec, T callback) {
  // 创建 SSL 套接字流对象
  SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                       write_timeout_sec, write_timeout_usec);
  // 调用回调函数处理数据并返回结果
  return callback(strm);
}

// SSL 初始化类
class SSLInit {
public:
  // SSL 初始化构造函数
  SSLInit() {
    // 初始化 OpenSSL 库
    OPENSSL_init_ssl(
        OPENSSL_INIT_LOAD_SSL_STRINGS | OPENSSL_INIT_LOAD_CRYPTO_STRINGS, NULL);
  }
};

// SSL 套接字流实现
inline SSLSocketStream::SSLSocketStream(socket_t sock, SSL *ssl,
                                        time_t read_timeout_sec,
                                        time_t read_timeout_usec,
                                        time_t write_timeout_sec,
                                        time_t write_timeout_usec)
    : sock_(sock), ssl_(ssl), read_timeout_sec_(read_timeout_sec),
      read_timeout_usec_(read_timeout_usec),
      write_timeout_sec_(write_timeout_sec),
      write_timeout_usec_(write_timeout_usec) {
  // 设置 SSL 连接模式
  SSL_clear_mode(ssl, SSL_MODE_AUTO_RETRY);
}

// SSL 套接字流析构函数
inline SSLSocketStream::~SSLSocketStream() {}

// 判断套接字是否可读
inline bool SSLSocketStream::is_readable() const {
  // 使用 select 函数判断套接字是否可读
  return detail::select_read(sock_, read_timeout_sec_, read_timeout_usec_) > 0;
}

// 判断套接字是否可写
inline bool SSLSocketStream::is_writable() const {
  // 使用 select 函数判断套接字是否可写，并且套接字仍然存活
  return select_write(sock_, write_timeout_sec_, write_timeout_usec_) > 0 &&
         is_socket_alive(sock_);
}

// 从套接字中读取数据
inline ssize_t SSLSocketStream::read(char *ptr, size_t size) {
  // 如果 SSL 缓冲区中有数据，直接读取
  if (SSL_pending(ssl_) > 0) {
    return SSL_read(ssl_, ptr, static_cast<int>(size));
  } else if (is_readable()) {
    // 如果套接字可读，读取数据
    auto ret = SSL_read(ssl_, ptr, static_cast<int>(size));
    if (ret < 0) {
      // 处理 SSL 读取错误
      auto err = SSL_get_error(ssl_, ret);
      auto n = 1000;
#ifdef _WIN32
      // 如果是在 Windows 平台下
      while (--n >= 0 && (err == SSL_ERROR_WANT_READ ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 如果不是在 Windows 平台下
      while (--n >= 0 && err == SSL_ERROR_WANT_READ) {
#endif
        // 如果 SSL 连接有数据可读
        if (SSL_pending(ssl_) > 0) {
          // 直接读取数据
          return SSL_read(ssl_, ptr, static_cast<int>(size));
        } else if (is_readable()) {
          // 如果可读性检查通过，等待 1 毫秒
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          // 读取数据
          ret = SSL_read(ssl_, ptr, static_cast<int>(size));
          // 如果读取成功，返回读取的字节数
          if (ret >= 0) { return ret; }
          // 获取 SSL 错误码
          err = SSL_get_error(ssl_, ret);
        } else {
          // 如果不可读，返回 -1
          return -1;
        }
      }
    }
    // 返回读取结果
    return ret;
  }
  // 如果不满足读取条件，返回 -1
  return -1;
}

// 写入数据到 SSL 连接
inline ssize_t SSLSocketStream::write(const char *ptr, size_t size) {
  // 如果可写性检查通过
  if (is_writable()) {
    // 计算实际写入数据的大小
    auto handle_size = static_cast<int>(
        std::min<size_t>(size, (std::numeric_limits<int>::max)()));

    // 写入数据到 SSL 连接
    auto ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));
    // 如果写入失败
    if (ret < 0) {
      // 获取 SSL 错误码
      auto err = SSL_get_error(ssl_, ret);
      auto n = 1000;
#ifdef _WIN32
      // 如果是在 Windows 平台下
      while (--n >= 0 && (err == SSL_ERROR_WANT_WRITE ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 如果不是在 Windows 平台下
      while (--n >= 0 && err == SSL_ERROR_WANT_WRITE) {
#endif
        // 如果可写性检查通过
        if (is_writable()) {
          // 等待 1 毫秒
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          // 写入数据到 SSL 连接
          ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));
          // 如果写入成功，返回写入的字节数
          if (ret >= 0) { return ret; }
          // 获取 SSL 错误码
          err = SSL_get_error(ssl_, ret);
        } else {
          // 如果不可写，返回 -1
          return -1;
        }
      }
    }
    // 返回写入结果
    return ret;
  }
  // 如果不满足写入条件，返回 -1
  return -1;
}

// 获取远程 IP 地址和端口号
inline void SSLSocketStream::get_remote_ip_and_port(std::string &ip,
                                                    int &port) const {
  // 调用 detail 命名空间下的函数获取远程 IP 地址和端口号
  detail::get_remote_ip_and_port(sock_, ip, port);
}
namespace detail
{
    // 获取本地 IP 地址和端口号
    inline void SSLSocketStream::get_local_ip_and_port(std::string &ip, int &port) const {
        detail::get_local_ip_and_port(sock_, ip, port);
    }

    // 返回套接字
    inline socket_t SSLSocketStream::socket() const { return sock_; }

    // SSL 初始化
    static SSLInit sslinit_;
} // namespace detail

// SSL HTTP 服务器实现
inline SSLServer::SSLServer(const char *cert_path, const char *private_key_path,
                            const char *client_ca_cert_file_path,
                            const char *client_ca_cert_dir_path,
                            const char *private_key_password) {
    // 创建 SSL 上下文对象
    ctx_ = SSL_CTX_new(TLS_server_method());

    if (ctx_) {
        // 设置 SSL 上下文选项
        SSL_CTX_set_options(ctx_,
                            SSL_OP_NO_COMPRESSION |
                                SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

        // 设置 SSL 最小协议版本
        SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

        // 在打开加密私钥之前添加默认密码回调
        if (private_key_password != nullptr && (private_key_password[0] != '\0')) {
            SSL_CTX_set_default_passwd_cb_userdata(
                ctx_,
                reinterpret_cast<void *>(const_cast<char *>(private_key_password)));
        }

        // 使用证书链文件和私钥文件
        if (SSL_CTX_use_certificate_chain_file(ctx_, cert_path) != 1 ||
            SSL_CTX_use_PrivateKey_file(ctx_, private_key_path, SSL_FILETYPE_PEM) != 1) {
            SSL_CTX_free(ctx_);
            ctx_ = nullptr;
        } else if (client_ca_cert_file_path || client_ca_cert_dir_path) {
            // 加载客户端 CA 证书位置
            SSL_CTX_load_verify_locations(ctx_, client_ca_cert_file_path,
                                        client_ca_cert_dir_path);

            // 设置验证选项
            SSL_CTX_set_verify(
                ctx_, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, nullptr);
        }
    }
}

// SSL HTTP 服务器实现
inline SSLServer::SSLServer(X509 *cert, EVP_PKEY *private_key,
                            X509_STORE *client_ca_cert_store) {
    // 创建 SSL 上下文对象
    ctx_ = SSL_CTX_new(TLS_server_method());

    if (ctx_) {
    # 设置 SSL 上下文的选项，禁用压缩和禁用会话重用
    SSL_CTX_set_options(ctx_,
                        SSL_OP_NO_COMPRESSION |
                            SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

    # 设置 SSL 上下文的最低协议版本为 TLS 1.1
    SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

    # 如果成功加载证书和私钥，则继续执行，否则释放 SSL 上下文并置空
    if (SSL_CTX_use_certificate(ctx_, cert) != 1 ||
        SSL_CTX_use_PrivateKey(ctx_, private_key) != 1) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    } else if (client_ca_cert_store) {
      # 设置 SSL 上下文的客户端 CA 证书存储
      SSL_CTX_set_cert_store(ctx_, client_ca_cert_store);

      # 设置 SSL 上下文的验证方式为要求对等方证书验证和如果没有对等方证书则验证失败
      SSL_CTX_set_verify(
          ctx_, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, nullptr);
    }
  }
// SSLServer 类的构造函数，接受一个设置 SSL 上下文的回调函数作为参数
inline SSLServer::SSLServer(
    const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback) {
  // 创建一个新的 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_method());
  // 如果成功创建了 SSL 上下文对象
  if (ctx_) {
    // 调用回调函数设置 SSL 上下文，如果设置失败则释放 SSL 上下文对象并置空
    if (!setup_ssl_ctx_callback(*ctx_)) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}

// SSLServer 类的析构函数
inline SSLServer::~SSLServer() {
  // 如果 SSL 上下文对象存在，则释放它
  if (ctx_) { SSL_CTX_free(ctx_); }
}

// 检查 SSLServer 对象是否有效
inline bool SSLServer::is_valid() const { return ctx_; }

// 获取 SSL 上下文对象
inline SSL_CTX *SSLServer::ssl_context() const { return ctx_; }

// 处理并关闭套接字
inline bool SSLServer::process_and_close_socket(socket_t sock) {
  // 创建 SSL 对象
  auto ssl = detail::ssl_new(
      sock, ctx_, ctx_mutex_,
      [&](SSL *ssl2) {
        return detail::ssl_connect_or_accept_nonblocking(
            sock, ssl2, SSL_accept, read_timeout_sec_, read_timeout_usec_);
      },
      [](SSL * /*ssl2*/) { return true; });

  // 初始化返回值为 false
  auto ret = false;
  // 如果成功创建了 SSL 对象
  if (ssl) {
    // 处理服务器套接字的 SSL 连接
    ret = detail::process_server_socket_ssl(
        svr_sock_, ssl, sock, keep_alive_max_count_, keep_alive_timeout_sec_,
        read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
        write_timeout_usec_,
        [this, ssl](Stream &strm, bool close_connection,
                    bool &connection_closed) {
          return process_request(strm, close_connection, connection_closed,
                                 [&](Request &req) { req.ssl = ssl; });
        });

    // 根据返回值判断是否优雅关闭 SSL 连接
    const bool shutdown_gracefully = ret;
    detail::ssl_delete(ctx_mutex_, ssl, shutdown_gracefully);
  }

  // 关闭套接字
  detail::shutdown_socket(sock);
  // 关闭套接字
  detail::close_socket(sock);
  // 返回处理结果
  return ret;
}

// SSL HTTP 客户端实现
// 构造函数，指定主机名，默认端口为 443
inline SSLClient::SSLClient(const std::string &host)
    : SSLClient(host, 443, std::string(), std::string()) {}

// 构造函数，指定主机名和端口号
inline SSLClient::SSLClient(const std::string &host, int port)
    : SSLClient(host, port, std::string(), std::string()) {}
// 构造函数，初始化 SSLClient 对象，传入主机名、端口号、客户端证书路径和客户端密钥路径
inline SSLClient::SSLClient(const std::string &host, int port,
                            const std::string &client_cert_path,
                            const std::string &client_key_path)
    : ClientImpl(host, port, client_cert_path, client_key_path) {
  // 创建 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_client_method());

  // 将主机名按 '.' 分割成组件，存储到 host_components_ 中
  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));
                });

  // 如果客户端证书路径和客户端密钥路径不为空，则加载证书和密钥
  if (!client_cert_path.empty() && !client_key_path.empty()) {
    if (SSL_CTX_use_certificate_file(ctx_, client_cert_path.c_str(),
                                     SSL_FILETYPE_PEM) != 1 ||
        SSL_CTX_use_PrivateKey_file(ctx_, client_key_path.c_str(),
                                    SSL_FILETYPE_PEM) != 1) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}

// 构造函数，初始化 SSLClient 对象，传入主机名、端口号、客户端证书和客户端密钥
inline SSLClient::SSLClient(const std::string &host, int port,
                            X509 *client_cert, EVP_PKEY *client_key)
    : ClientImpl(host, port) {
  // 创建 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_client_method());

  // 将主机名按 '.' 分割成组件，存储到 host_components_ 中
  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));
                });

  // 如果客户端证书和客户端密钥不为空，则加载证书和密钥
  if (client_cert != nullptr && client_key != nullptr) {
    if (SSL_CTX_use_certificate(ctx_, client_cert) != 1 ||
        SSL_CTX_use_PrivateKey(ctx_, client_key) != 1) {
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    }
  }
}

// 析构函数，释放 SSLClient 对象占用的资源
inline SSLClient::~SSLClient() {
  // 如果 SSL 上下文对象存在，则释放资源
  if (ctx_) { SSL_CTX_free(ctx_); }
  // 确保关闭 SSL 连接，因为在基类析构函数中 shutdown_ssl 将解析为基类函数而不是派生类函数，不会释放 SSL（导致泄漏）
  shutdown_ssl_impl(socket_, true);
}

// 检查 SSLClient 对象是否有效
inline bool SSLClient::is_valid() const { return ctx_; }

// 设置 CA 证书存储
inline void SSLClient::set_ca_cert_store(X509_STORE *ca_cert_store) {
  // 如果 CA 证书存储存在
    // 检查是否存在 SSL 上下文
    if (ctx_) {
      // 检查 SSL 上下文中的证书存储是否与 ca_cert_store 不同
      if (SSL_CTX_get_cert_store(ctx_) != ca_cert_store) {
        // 释放为旧证书分配的内存，并使用新的存储 ca_cert_store
        SSL_CTX_set_cert_store(ctx_, ca_cert_store);
      }
    } else {
      // 如果 SSL 上下文不存在，则释放 ca_cert_store 分配的内存
      X509_STORE_free(ca_cert_store);
    }
  }
// 加载 CA 证书存储，使用给定的 CA 证书和大小
inline void SSLClient::load_ca_cert_store(const char *ca_cert,
                                          std::size_t size) {
  // 设置 CA 证书存储
  set_ca_cert_store(ClientImpl::create_ca_cert_store(ca_cert, size));
}

// 获取 OpenSSL 验证结果
inline long SSLClient::get_openssl_verify_result() const {
  return verify_result_;
}

// 返回 SSL 上下文
inline SSL_CTX *SSLClient::ssl_context() const { return ctx_; }

// 创建并连接套接字
inline bool SSLClient::create_and_connect_socket(Socket &socket, Error &error) {
  // 检查 SSLClient 是否有效，并创建并连接套接字
  return is_valid() && ClientImpl::create_and_connect_socket(socket, error);
}

// 假设 socket_mutex_ 已锁定且没有请求在传输中
inline bool SSLClient::connect_with_proxy(Socket &socket, Response &res,
                                          bool &success, Error &error) {
  success = true;
  Response proxy_res;
  // 处理客户端套接字
  if (!detail::process_client_socket(
          socket.sock, read_timeout_sec_, read_timeout_usec_,
          write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
            Request req2;
            req2.method = "CONNECT";
            req2.path = host_and_port_;
            return process_request(strm, req2, proxy_res, false, error);
          })) {
    // 线程安全关闭所有内容，因为假设没有请求在传输中
    shutdown_ssl(socket, true);
    shutdown_socket(socket);
    close_socket(socket);
    success = false;
    return false;
  }

  // 如果代理响应状态码为 407
  if (proxy_res.status == 407) {
    // 如果代理用户名和密码都不为空
    if (!proxy_digest_auth_username_.empty() &&
        !proxy_digest_auth_password_.empty()) {
      // 创建一个空的字符串到字符串的映射
      std::map<std::string, std::string> auth;
      // 解析代理响应中的身份验证信息
      if (detail::parse_www_authenticate(proxy_res, auth, true)) {
        // 重置代理响应
        proxy_res = Response();
        // 处理客户端套接字
        if (!detail::process_client_socket(
                socket.sock, read_timeout_sec_, read_timeout_usec_,
                write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
                  // 创建一个请求对象
                  Request req3;
                  req3.method = "CONNECT";
                  req3.path = host_and_port_;
                  // 插入摘要认证头部到请求头部
                  req3.headers.insert(detail::make_digest_authentication_header(
                      req3, auth, 1, detail::random_string(10),
                      proxy_digest_auth_username_, proxy_digest_auth_password_,
                      true));
                  // 处理请求
                  return process_request(strm, req3, proxy_res, false, error);
                })) {
          // 线程安全地关闭所有连接，因为我们假设没有请求在进行中
          shutdown_ssl(socket, true);
          shutdown_socket(socket);
          close_socket(socket);
          success = false;
          return false;
        }
      }
    }

  // 如果状态码不是 200，代理请求失败
  // 将错误设置为 ProxyConnection，并将代理响应作为请求的响应返回
  if (proxy_res.status != 200) {
    error = Error::ProxyConnection;
    res = std::move(proxy_res);
    // 线程安全地关闭所有连接，因为我们假设没有请求在进行中
    shutdown_ssl(socket, true);
    shutdown_socket(socket);
    close_socket(socket);
    return false;
  }

  return true;
// 加载 SSL 证书，返回加载结果
inline bool SSLClient::load_certs() {
  auto ret = true;

  // 使用 std::call_once 保证只初始化一次证书
  std::call_once(initialize_cert_, [&]() {
    // 使用互斥锁保护证书上下文
    std::lock_guard<std::mutex> guard(ctx_mutex_);
    // 如果 CA 证书文件路径不为空
    if (!ca_cert_file_path_.empty()) {
      // 加载 CA 证书文件
      if (!SSL_CTX_load_verify_locations(ctx_, ca_cert_file_path_.c_str(),
                                         nullptr)) {
        ret = false;
      }
    } 
    // 如果 CA 证书目录路径不为空
    else if (!ca_cert_dir_path_.empty()) {
      // 加载 CA 证书目录
      if (!SSL_CTX_load_verify_locations(ctx_, nullptr,
                                         ca_cert_dir_path_.c_str())) {
        ret = false;
      }
    } 
    // 如果以上两种情况都不满足
    else {
      auto loaded = false;
      // 如果是 Windows 系统
#ifdef _WIN32
      loaded =
          detail::load_system_certs_on_windows(SSL_CTX_get_cert_store(ctx_));
      // 如果是 macOS 系统，并且定义了使用 macOS Keychain 证书
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
      // 如果是 macOS 系统
#if TARGET_OS_OSX
      loaded = detail::load_system_certs_on_macos(SSL_CTX_get_cert_store(ctx_));
#endif // TARGET_OS_OSX
#endif // _WIN32
      // 如果证书加载失败，设置默认的证书路径
      if (!loaded) { SSL_CTX_set_default_verify_paths(ctx_); }
    }
  });

  // 返回加载结果
  return ret;
}
// 初始化 SSL 客户端，包括创建 SSL 对象、加载证书、建立连接等操作
inline bool SSLClient::initialize_ssl(Socket &socket, Error &error) {
  // 使用 lambda 表达式创建 SSL 对象，并设置相关参数
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
          // 设置 SSL 验证方式为 SSL_VERIFY_NONE
          SSL_set_verify(ssl2, SSL_VERIFY_NONE, nullptr);
        }

        // 进行 SSL 连接，如果连接失败则返回错误
        if (!detail::ssl_connect_or_accept_nonblocking(
                socket.sock, ssl2, SSL_connect, connection_timeout_sec_,
                connection_timeout_usec_)) {
          error = Error::SSLConnection;
          return false;
        }

        // 如果需要进行服务器证书验证
        if (server_certificate_verification_) {
          // 获取服务器证书验证结果
          verify_result_ = SSL_get_verify_result(ssl2);

          // 如果验证结果不是 X509_V_OK，则返回错误
          if (verify_result_ != X509_V_OK) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 获取服务器证书
          auto server_cert = SSL_get1_peer_certificate(ssl2);

          // 如果服务器证书为空，则返回错误
          if (server_cert == nullptr) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 验证服务器主机名，如果验证失败则返回错误
          if (!verify_host(server_cert)) {
            X509_free(server_cert);
            error = Error::SSLServerVerification;
            return false;
          }
          X509_free(server_cert);
        }

        return true;
      },
      [&](SSL *ssl2) {
        // 设置 TLS 扩展主机名
        SSL_set_tlsext_host_name(ssl2, host_.c_str());
        return true;
      });

  // 如果 SSL 对象创建成功，则将其赋值给 socket.ssl，并返回成功
  if (ssl) {
    socket.ssl = ssl;
    return true;
  }

  // 如果 SSL 对象创建失败，则关闭 socket 并返回失败
  shutdown_socket(socket);
  close_socket(socket);
  return false;
}

// 关闭 SSL 连接
inline void SSLClient::shutdown_ssl(Socket &socket, bool shutdown_gracefully) {
  // 调用 shutdown_ssl_impl 函数关闭 SSL 连接
  shutdown_ssl_impl(socket, shutdown_gracefully);
}
// 在 SSLClient 类中实现关闭 SSL 连接的函数，根据参数决定是否优雅关闭
inline void SSLClient::shutdown_ssl_impl(Socket &socket,
                                         bool shutdown_gracefully) {
  // 如果套接字无效，则直接返回
  if (socket.sock == INVALID_SOCKET) {
    assert(socket.ssl == nullptr);
    return;
  }
  // 如果存在 SSL 连接，则删除 SSL 对象
  if (socket.ssl) {
    detail::ssl_delete(ctx_mutex_, socket.ssl, shutdown_gracefully);
    socket.ssl = nullptr;
  }
  // 确保 SSL 对象已经被删除
  assert(socket.ssl == nullptr);
}

// 处理套接字的函数，调用回调函数处理 SSL 连接
inline bool
SSLClient::process_socket(const Socket &socket,
                          std::function<bool(Stream &strm)> callback) {
  // 确保存在 SSL 连接
  assert(socket.ssl);
  // 调用 detail 命名空间中的函数处理客户端 SSL 套接字
  return detail::process_client_socket_ssl(
      socket.ssl, socket.sock, read_timeout_sec_, read_timeout_usec_,
      write_timeout_sec_, write_timeout_usec_, std::move(callback));
}

// 返回是否为 SSL 连接
inline bool SSLClient::is_ssl() const { return true; }
// 验证服务器证书的主机名是否匹配
inline bool SSLClient::verify_host(X509 *server_cert) const {
  /* 引用自 RFC2818 第 3.1 节 "服务器标识"

     如果存在类型为 dNSName 的 subjectAltName 扩展，则必须将其用作标识。
     否则，必须使用证书主体字段中的（最具体的）Common Name字段。
     尽管使用 Common Name 是现有的做法，但已被弃用，并鼓励证书颁发机构使用 dNSName。

     匹配使用[RFC2459]指定的匹配规则。如果证书中存在给定类型的多个标识（例如，多个 dNSName 名称，
     则在集合中的任何一个匹配被视为可接受。）名称可能包含通配符字符*，被视为匹配任何单个域名组件或组件片段。
     例如，*.a.com 匹配 foo.a.com 但不匹配 bar.foo.a.com。f*.com 匹配 foo.com 但不匹配 bar.com。

     在某些情况下，URI被指定为 IP 地址而不是主机名。在这种情况下，证书中必须存在 iPAddress subjectAltName，
     并且必须与 URI 中的 IP 完全匹配。
  */
  return verify_host_with_subject_alt_name(server_cert) ||
         verify_host_with_common_name(server_cert);
}

// 使用 subjectAltName 验证主机名
inline bool
SSLClient::verify_host_with_subject_alt_name(X509 *server_cert) const {
  auto ret = false;

  auto type = GEN_DNS;

  struct in6_addr addr6;
  struct in_addr addr;
  size_t addr_len = 0;

#ifndef __MINGW32__
  // 检查主机名是否为 IPv6 或 IPv4 地址
  if (inet_pton(AF_INET6, host_.c_str(), &addr6)) {
    type = GEN_IPADD;
    addr_len = sizeof(struct in6_addr);
  } else if (inet_pton(AF_INET, host_.c_str(), &addr)) {
    type = GEN_IPADD;
    addr_len = sizeof(struct in_addr);
  }
#endif

  // 获取证书中的 subjectAltName 扩展
  auto alt_names = static_cast<const struct stack_st_GENERAL_NAME *>(
      X509_get_ext_d2i(server_cert, NID_subject_alt_name, nullptr, nullptr));

  if (alt_names) {
    auto dsn_matched = false;
    // 初始化 IP 匹配标志为 false
    auto ip_matched = false;

    // 获取备用名称列表中的名称数量
    auto count = sk_GENERAL_NAME_num(alt_names);

    // 遍历备用名称列表，检查是否有匹配的名称
    for (decltype(count) i = 0; i < count && !dsn_matched; i++) {
      // 获取备用名称列表中的值
      auto val = sk_GENERAL_NAME_value(alt_names, i);
      // 检查值的类型是否与指定类型相匹配
      if (val->type == type) {
        // 将值转换为字符串
        auto name =
            reinterpret_cast<const char *>(ASN1_STRING_get0_data(val->d.ia5));
        // 获取字符串的长度
        auto name_len = static_cast<size_t>(ASN1_STRING_length(val->d.ia5));

        // 根据类型进行不同的处理
        switch (type) {
        case GEN_DNS: dsn_matched = check_host_name(name, name_len); break;

        case GEN_IPADD:
          // 检查 IP 地址是否匹配
          if (!memcmp(&addr6, name, addr_len) ||
              !memcmp(&addr, name, addr_len)) {
            ip_matched = true;
          }
          break;
        }
      }
    }

    // 如果主机名匹配或者 IP 地址匹配，则返回 true
    if (dsn_matched || ip_matched) { ret = true; }
  }

  // 释放备用名称列表
  GENERAL_NAMES_free(const_cast<STACK_OF(GENERAL_NAME) *>(
      reinterpret_cast<const STACK_OF(GENERAL_NAME) *>(alt_names)));
  // 返回结果
  return ret;
// 验证服务器证书的通用名称与主机名是否匹配
inline bool SSLClient::verify_host_with_common_name(X509 *server_cert) const {
  // 获取服务器证书的主题名称
  const auto subject_name = X509_get_subject_name(server_cert);

  // 如果主题名称不为空
  if (subject_name != nullptr) {
    // 创建一个缓冲区来存储通用名称
    char name[BUFSIZ];
    // 获取通用名称的文本表示
    auto name_len = X509_NAME_get_text_by_NID(subject_name, NID_commonName,
                                              name, sizeof(name));

    // 如果成功获取通用名称
    if (name_len != -1) {
      // 检查主机名是否与通用名称匹配
      return check_host_name(name, static_cast<size_t>(name_len));
    }
  }

  // 如果未能获取通用名称或者主机名与通用名称不匹配，则返回false
  return false;
}

// 检查主机名是否与模式匹配
inline bool SSLClient::check_host_name(const char *pattern,
                                       size_t pattern_len) const {
  // 如果主机名长度与模式长度相等且内容相同，则返回true
  if (host_.size() == pattern_len && host_ == pattern) { return true; }

  // 如果主机名与模式不完全匹配，则进行通配符匹配
  std::vector<std::string> pattern_components;
  // 将模式按点分割成组件
  detail::split(&pattern[0], &pattern[pattern_len], '.',
                [&](const char *b, const char *e) {
                  pattern_components.emplace_back(std::string(b, e));
                });

  // 如果主机名的组件数量与模式的组件数量不相等，则返回false
  if (host_components_.size() != pattern_components.size()) { return false; }

  // 逐个比较主机名的组件与模式的组件
  auto itr = pattern_components.begin();
  for (const auto &h : host_components_) {
    auto &p = *itr;
    // 如果组件不相等且不是通配符，则返回false
    if (p != h && p != "*") {
      // 进行部分匹配，支持通配符
      auto partial_match = (p.size() > 0 && p[p.size() - 1] == '*' &&
                            !p.compare(0, p.size() - 1, h));
      if (!partial_match) { return false; }
    }
    ++itr;
  }

  // 如果所有组件匹配，则返回true
  return true;
}
#endif

// 通用客户端实现
// 使用给定的 scheme_host_port 构造客户端对象
inline Client::Client(const std::string &scheme_host_port)
    : Client(scheme_host_port, std::string(), std::string()) {}

// 使用给定的 scheme_host_port、client_cert_path 和 client_key_path 构造客户端对象
inline Client::Client(const std::string &scheme_host_port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path) {
  // 定义正则表达式来解析 scheme_host_port
  const static std::regex re(
      R"((?:([a-z]+):\/\/)?(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)");

  std::smatch m;
  // 使用正则表达式匹配 scheme_host_port
  if (std::regex_match(scheme_host_port, m, re)) {
    // 提取 scheme
    auto scheme = m[1].str();
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    // 如果支持 OpenSSL 并且 scheme 不为空且不是 "http" 或 "https"，则抛出异常
    if (!scheme.empty() && (scheme != "http" && scheme != "https")) {
#else
    // 如果不支持 OpenSSL 并且 scheme 不为空且不是 "http"，则抛出异常
    if (!scheme.empty() && scheme != "http") {
#endif
#ifndef CPPHTTPLIB_NO_EXCEPTIONS
      // 如果不支持异常处理，则构造错误信息并抛出无效参数异常
      std::string msg = "'" + scheme + "' scheme is not supported.";
      throw std::invalid_argument(msg);
#endif
      // 返回
      return;
    }

    // 判断是否为 SSL
    auto is_ssl = scheme == "https";

    // 获取主机名
    auto host = m[2].str();
    if (host.empty()) { host = m[3].str(); }

    // 获取端口号
    auto port_str = m[4].str();
    auto port = !port_str.empty() ? std::stoi(port_str) : (is_ssl ? 443 : 80);

    // 如果是 SSL
    if (is_ssl) {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 使用 SSLClient 创建客户端对象
      cli_ = detail::make_unique<SSLClient>(host, port, client_cert_path,
                                            client_key_path);
      is_ssl_ = is_ssl;
#endif
    } else {
      // 使用 ClientImpl 创建客户端对象
      cli_ = detail::make_unique<ClientImpl>(host, port, client_cert_path,
                                             client_key_path);
    }
  } else {
    // 使用 ClientImpl 创建客户端对象
    cli_ = detail::make_unique<ClientImpl>(scheme_host_port, 80,
                                           client_cert_path, client_key_path);
  }
}

// 构造函数，根据主机名和端口号创建客户端对象
inline Client::Client(const std::string &host, int port)
    : cli_(detail::make_unique<ClientImpl>(host, port)) {}

// 构造函数，根据主机名、端口号、客户端证书路径和客户端密钥路径创建客户端对象
inline Client::Client(const std::string &host, int port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path)
    : cli_(detail::make_unique<ClientImpl>(host, port, client_cert_path,
                                           client_key_path)) {}

// 析构函数
inline Client::~Client() {}

// 检查客户端对象是否有效
inline bool Client::is_valid() const {
  return cli_ != nullptr && cli_->is_valid();
}

// 发起 GET 请求
inline Result Client::Get(const std::string &path) { return cli_->Get(path); }
// 发起带自定义头部的 GET 请求
inline Result Client::Get(const std::string &path, const Headers &headers) {
  return cli_->Get(path, headers);
}
// 发起带进度回调的 GET 请求
inline Result Client::Get(const std::string &path, Progress progress) {
  return cli_->Get(path, std::move(progress));
}
# 使用给定路径、头部和进度函数执行 GET 请求
inline Result Client::Get(const std::string &path, const Headers &headers,
                          Progress progress) {
  return cli_->Get(path, headers, std::move(progress));
}

# 使用给定路径和内容接收器执行 GET 请求
inline Result Client::Get(const std::string &path,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, std::move(content_receiver));
}

# 使用给定路径、头部和内容接收器执行 GET 请求
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(content_receiver));
}

# 使用给定路径和内容接收器执行 GET 请求，同时传入进度函数
inline Result Client::Get(const std::string &path,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(content_receiver), std::move(progress));
}

# 使用给定路径、头部、内容接收器和进度函数执行 GET 请求
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(content_receiver),
                   std::move(progress));
}

# 使用给定路径、响应处理器和内容接收器执行 GET 请求
inline Result Client::Get(const std::string &path,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, std::move(response_handler),
                   std::move(content_receiver));
}

# 使用给定路径、头部、响应处理器和内容接收器执行 GET 请求
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(response_handler),
                   std::move(content_receiver));
}

# 使用给定路径、响应处理器、内容接收器和进度函数执行 GET 请求
inline Result Client::Get(const std::string &path,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(response_handler),
                   std::move(content_receiver), std::move(progress));
}
# 发起 GET 请求，传入路径、请求头、响应处理器、内容接收器和进度回调函数，返回请求结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(response_handler),
                   std::move(content_receiver), std::move(progress));
}

# 发起 GET 请求，传入路径、参数、请求头和进度回调函数，返回请求结果
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers, Progress progress) {
  return cli_->Get(path, params, headers, progress);
}

# 发起 GET 请求，传入路径、参数、请求头和内容接收器，返回请求结果
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, content_receiver, progress);
}

# 发起 GET 请求，传入路径、参数、请求头、响应处理器、内容接收器和进度回调函数，返回请求结果
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, response_handler, content_receiver,
                   progress);
}

# 发起 HEAD 请求，传入路径，返回请求结果
inline Result Client::Head(const std::string &path) { return cli_->Head(path); }

# 发起 HEAD 请求，传入路径和请求头，返回请求结果
inline Result Client::Head(const std::string &path, const Headers &headers) {
  return cli_->Head(path, headers);
}

# 发起 POST 请求，传入路径，返回请求结果
inline Result Client::Post(const std::string &path) { return cli_->Post(path); }

# 发起 POST 请求，传入路径和请求头，返回请求结果
inline Result Client::Post(const std::string &path, const Headers &headers) {
  return cli_->Post(path, headers);
}

# 发起 POST 请求，传入路径、请求体、内容长度和内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const char *body,
                           size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_length, content_type);
}
# 发送 POST 请求，传入路径、请求头、请求体、请求体长度、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const char *body, size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, headers, body, content_length, content_type);
}

# 发送 POST 请求，传入路径、请求体、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const std::string &body,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_type);
}

# 发送 POST 请求，传入路径、请求头、请求体、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const std::string &body,
                           const std::string &content_type) {
  return cli_->Post(path, headers, body, content_type);
}

# 发送 POST 请求，传入路径、请求体长度、内容提供者、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, size_t content_length,
                           ContentProvider content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, content_length, std::move(content_provider),
                    content_type);
}

# 发送 POST 请求，传入路径、无长度内容提供者、内容类型，返回请求结果
inline Result Client::Post(const std::string &path,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, std::move(content_provider), content_type);
}

# 发送 POST 请求，传入路径、请求头、请求体长度、内容提供者、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           size_t content_length,
                           ContentProvider content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, headers, content_length, std::move(content_provider),
                    content_type);
}

# 发送 POST 请求，传入路径、请求头、无长度内容提供者、内容类型，返回请求结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  return cli_->Post(path, headers, std::move(content_provider), content_type);
}
// 发送 POST 请求，传递路径和参数，返回结果
inline Result Client::Post(const std::string &path, const Params &params) {
  return cli_->Post(path, params);
}

// 发送 POST 请求，传递路径、头部和参数，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const Params &params) {
  return cli_->Post(path, headers, params);
}

// 发送 POST 请求，传递路径和多部分表单数据项，返回结果
inline Result Client::Post(const std::string &path,
                           const MultipartFormDataItems &items) {
  return cli_->Post(path, items);
}

// 发送 POST 请求，传递路径、头部和多部分表单数据项，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items) {
  return cli_->Post(path, headers, items);
}

// 发送 POST 请求，传递路径、头部、多部分表单数据项和边界，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items,
                           const std::string &boundary) {
  return cli_->Post(path, headers, items, boundary);
}

// 发送 POST 请求，传递路径、头部、多部分表单数据项和多部分表单数据提供者项，返回结果
inline Result
Client::Post(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items,
             const MultipartFormDataProviderItems &provider_items) {
  return cli_->Post(path, headers, items, provider_items);
}

// 发送 PUT 请求，传递路径，返回结果
inline Result Client::Put(const std::string &path) { return cli_->Put(path); }

// 发送 PUT 请求，传递路径、内容、内容长度和内容类型，返回结果
inline Result Client::Put(const std::string &path, const char *body,
                          size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_length, content_type);
}

// 发送 PUT 请求，传递路径、头部、内容、内容长度和内容类型，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const char *body, size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_length, content_type);
}

// 发送 PUT 请求，传递路径、内容和内容类型，返回结果
inline Result Client::Put(const std::string &path, const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_type);
}
# 使用给定路径、头部、请求体和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_type);
}

# 使用给定路径、内容长度、内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, content_length, std::move(content_provider),
                   content_type);
}

# 使用给定路径、无长度内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, std::move(content_provider), content_type);
}

# 使用给定路径、头部、内容长度、内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, content_length, std::move(content_provider),
                   content_type);
}

# 使用给定路径、头部、无长度内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, std::move(content_provider), content_type);
}

# 使用给定路径和参数执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Params &params) {
  return cli_->Put(path, params);
}

# 使用给定路径、头部和参数执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const Params &params) {
  return cli_->Put(path, headers, params);
}

# 使用给定路径和多部分表单数据项执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, items);
}
// 将数据通过 HTTP PUT 方法发送到指定路径，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, headers, items);
}

// 将数据通过 HTTP PUT 方法发送到指定路径，带有自定义边界，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items,
                          const std::string &boundary) {
  return cli_->Put(path, headers, items, boundary);
}

// 将数据通过 HTTP PUT 方法发送到指定路径，带有提供者数据项，返回结果
inline Result
Client::Put(const std::string &path, const Headers &headers,
            const MultipartFormDataItems &items,
            const MultipartFormDataProviderItems &provider_items) {
  return cli_->Put(path, headers, items, provider_items);
}

// 使用 HTTP PATCH 方法部分更新指定路径的资源，返回结果
inline Result Client::Patch(const std::string &path) {
  return cli_->Patch(path);
}

// 使用 HTTP PATCH 方法部分更新指定路径的资源，带有指定内容和类型，返回结果
inline Result Client::Patch(const std::string &path, const char *body,
                            size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_length, content_type);
}

// 使用 HTTP PATCH 方法部分更新指定路径的资源，带有指定头部、内容和类型，返回结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const char *body, size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_length, content_type);
}

// 使用 HTTP PATCH 方法部分更新指定路径的资源，带有指定内容和类型，返回结果
inline Result Client::Patch(const std::string &path, const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_type);
}

// 使用 HTTP PATCH 方法部分更新指定路径的资源，带有指定头部、内容和类型，返回结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_type);
}
// 使用给定的路径、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result Client::Patch(const std::string &path, size_t content_length,
                            ContentProvider content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, content_length, std::move(content_provider),
                     content_type);
}

// 使用给定的路径、内容提供者（无长度）和内容类型执行 PATCH 请求
inline Result Client::Patch(const std::string &path,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, std::move(content_provider), content_type);
}

// 使用给定的路径、头部、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            size_t content_length,
                            ContentProvider content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, content_length, std::move(content_provider),
                     content_type);
}

// 使用给定的路径、头部、内容提供者（无长度）和内容类型执行 PATCH 请求
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, std::move(content_provider), content_type);
}

// 使用给定的路径执行 DELETE 请求
inline Result Client::Delete(const std::string &path) {
  return cli_->Delete(path);
}

// 使用给定的路径和头部执行 DELETE 请求
inline Result Client::Delete(const std::string &path, const Headers &headers) {
  return cli_->Delete(path, headers);
}

// 使用给定的路径、内容、内容长度和内容类型执行 DELETE 请求
inline Result Client::Delete(const std::string &path, const char *body,
                             size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_length, content_type);
}

// 使用给定的路径、头部、内容、内容长度和内容类型执行 DELETE 请求
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const char *body, size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_length, content_type);
}
// 调用底层 Client 对象的 Delete 方法，传入路径、请求体和内容类型，返回结果
inline Result Client::Delete(const std::string &path, const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_type);
}

// 调用底层 Client 对象的 Delete 方法，传入路径、请求头、请求体和内容类型，返回结果
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_type);
}

// 调用底层 Client 对象的 Options 方法，传入路径，返回结果
inline Result Client::Options(const std::string &path) {
  return cli_->Options(path);
}

// 调用底层 Client 对象的 Options 方法，传入路径和请求头，返回结果
inline Result Client::Options(const std::string &path, const Headers &headers) {
  return cli_->Options(path, headers);
}

// 调用底层 Client 对象的 send 方法，传入请求、响应和错误对象，返回布尔值
inline bool Client::send(Request &req, Response &res, Error &error) {
  return cli_->send(req, res, error);
}

// 调用底层 Client 对象的 send 方法，传入请求对象，返回结果
inline Result Client::send(const Request &req) { return cli_->send(req); }

// 调用底层 Client 对象的 stop 方法
inline void Client::stop() { cli_->stop(); }

// 调用底层 Client 对象的 host 方法，返回主机名
inline std::string Client::host() const { return cli_->host(); }

// 调用底层 Client 对象的 port 方法，返回端口号
inline int Client::port() const { return cli_->port(); }

//
// 设置客户端读取超时时间
inline void Client::set_read_timeout(time_t sec, time_t usec) {
  cli_->set_read_timeout(sec, usec);
}

// 设置客户端写入超时时间
inline void Client::set_write_timeout(time_t sec, time_t usec) {
  cli_->set_write_timeout(sec, usec);
}

// 设置基本身份验证
inline void Client::set_basic_auth(const std::string &username,
                                   const std::string &password) {
  cli_->set_basic_auth(username, password);
}

// 设置 Bearer Token 身份验证
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

// 设置是否保持连接
inline void Client::set_keep_alive(bool on) { cli_->set_keep_alive(on); }

// 设置是否跟随重定向
inline void Client::set_follow_location(bool on) {
  cli_->set_follow_location(on);
}

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

// 设置代理服务器基本身份验证
inline void Client::set_proxy_basic_auth(const std::string &username,
                                         const std::string &password) {
  cli_->set_proxy_basic_auth(username, password);
}

// 设置代理服务器 Bearer Token 身份验证
inline void Client::set_proxy_bearer_token_auth(const std::string &token) {
  cli_->set_proxy_bearer_token_auth(token);
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理服务器摘要身份验证
inline void Client::set_proxy_digest_auth(const std::string &username,
                                          const std::string &password) {
  cli_->set_proxy_digest_auth(username, password);
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 启用或禁用服务器证书验证
inline void Client::enable_server_certificate_verification(bool enabled) {
  // 调用底层客户端对象的方法来设置服务器证书验证
  cli_->enable_server_certificate_verification(enabled);
}
#endif

// 设置日志记录器
inline void Client::set_logger(Logger logger) {
  // 调用底层客户端对象的方法来设置日志记录器
  cli_->set_logger(std::move(logger));
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书路径
inline void Client::set_ca_cert_path(const std::string &ca_cert_file_path,
                                     const std::string &ca_cert_dir_path) {
  // 调用底层客户端对象的方法来设置 CA 证书路径
  cli_->set_ca_cert_path(ca_cert_file_path, ca_cert_dir_path);
}

// 设置 CA 证书存储
inline void Client::set_ca_cert_store(X509_STORE *ca_cert_store) {
  // 如果使用 SSL，则调用 SSLClient 对象的方法设置 CA 证书存储，否则调用普通客户端对象的方法
  if (is_ssl_) {
    static_cast<SSLClient &>(*cli_).set_ca_cert_store(ca_cert_store);
  } else {
    cli_->set_ca_cert_store(ca_cert_store);
  }
}

// 加载 CA 证书存储
inline void Client::load_ca_cert_store(const char *ca_cert, std::size_t size) {
  // 调用底层客户端对象的方法来创建并设置 CA 证书存储
  set_ca_cert_store(cli_->create_ca_cert_store(ca_cert, size));
}

// 获取 OpenSSL 验证结果
inline long Client::get_openssl_verify_result() const {
  // 如果使用 SSL，则调用 SSLClient 对象的方法获取 OpenSSL 验证结果，否则返回 -1
  if (is_ssl_) {
    return static_cast<SSLClient &>(*cli_).get_openssl_verify_result();
  }
  return -1; // 注意：-1 不匹配任何 X509_V_ERR_???
}

// 获取 SSL 上下文
inline SSL_CTX *Client::ssl_context() const {
  // 如果使用 SSL，则返回 SSLClient 对象的 SSL 上下文，否则返回空指针
  if (is_ssl_) { return static_cast<SSLClient &>(*cli_).ssl_context(); }
  return nullptr;
}
#endif

// ----------------------------------------------------------------------------

} // namespace httplib

// 如果在 Windows 下并且使用了 CPPHTTPLIB_USE_POLL 宏，则取消定义 poll 函数
#if defined(_WIN32) && defined(CPPHTTPLIB_USE_POLL)
#undef poll
#endif

#endif // CPPHTTPLIB_HTTPLIB_H
```