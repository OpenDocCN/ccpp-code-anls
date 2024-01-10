# `PowerInfer\examples\server\httplib.h`

```
#ifndef CPPHTTPLIB_HTTPLIB_H
#define CPPHTTPLIB_HTTPLIB_H

#define CPPHTTPLIB_VERSION "0.12.2"

/*
 * Configuration
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
// 定义接收缓冲区大小为4096字节
#define CPPHTTPLIB_RECV_BUFSIZ size_t(4096u)
#endif

// 定义压缩缓冲区大小为16384字节
#ifndef CPPHTTPLIB_COMPRESSION_BUFSIZ
#define CPPHTTPLIB_COMPRESSION_BUFSIZ size_t(16384u)
#endif

// 定义线程池数量为硬件并发数减1，最小为8
#ifndef CPPHTTPLIB_THREAD_POOL_COUNT
#define CPPHTTPLIB_THREAD_POOL_COUNT                                           \
  ((std::max)(8u, std::thread::hardware_concurrency() > 0                      \
                      ? std::thread::hardware_concurrency() - 1                \
                      : 0))
#endif

// 定义接收标志为0
#ifndef CPPHTTPLIB_RECV_FLAGS
#define CPPHTTPLIB_RECV_FLAGS 0
#endif

// 定义发送标志为0
#ifndef CPPHTTPLIB_SEND_FLAGS
#define CPPHTTPLIB_SEND_FLAGS 0
#endif

// 定义监听队列长度为5
#ifndef CPPHTTPLIB_LISTEN_BACKLOG
#define CPPHTTPLIB_LISTEN_BACKLOG 5
#endif

/*
 * Headers
 */

#ifdef _WIN32
// 如果未定义_CRT_SECURE_NO_WARNINGS，则定义_CRT_SECURE_NO_WARNINGS
#ifndef _CRT_SECURE_NO_WARNINGS
#define _CRT_SECURE_NO_WARNINGS
#endif //_CRT_SECURE_NO_WARNINGS

// 如果未定义_CRT_NONSTDC_NO_DEPRECATE，则定义_CRT_NONSTDC_NO_DEPRECATE
#ifndef _CRT_NONSTDC_NO_DEPRECATE
#define _CRT_NONSTDC_NO_DEPRECATE
#endif //_CRT_NONSTDC_NO_DEPRECATE

// 如果是MSC版本小于1900，则报错不支持
#if defined(_MSC_VER)
#if _MSC_VER < 1900
#error Sorry, Visual Studio versions prior to 2015 are not supported
#endif

// 添加ws2_32.lib库
#pragma comment(lib, "ws2_32.lib")

// 根据_WIN64宏定义不同的ssize_t类型
#ifdef _WIN64
using ssize_t = __int64;
#else
using ssize_t = long;
#endif
#endif // _MSC_VER

// 如果未定义S_ISREG宏，则定义S_ISREG宏
#ifndef S_ISREG
#define S_ISREG(m) (((m)&S_IFREG) == S_IFREG)
#endif // S_ISREG

// 如果未定义S_ISDIR宏，则定义S_ISDIR宏
#ifndef S_ISDIR
#define S_ISDIR(m) (((m)&S_IFDIR) == S_IFDIR)
#endif // S_ISDIR

// 如果未定义NOMINMAX宏，则定义NOMINMAX宏
#ifndef NOMINMAX
#define NOMINMAX
#endif // NOMINMAX

// 包含Windows特有的头文件
#include <io.h>
#include <winsock2.h>
#include <ws2tcpip.h>

// 如果未定义WSA_FLAG_NO_HANDLE_INHERIT宏，则定义WSA_FLAG_NO_HANDLE_INHERIT宏
#ifndef WSA_FLAG_NO_HANDLE_INHERIT
#define WSA_FLAG_NO_HANDLE_INHERIT 0x80
#endif

// 如果未定义strcasecmp宏，则定义strcasecmp宏
#ifndef strcasecmp
#define strcasecmp _stricmp
#endif // strcasecmp

// 定义socket_t类型为SOCKET类型
using socket_t = SOCKET;
// 如果定义了CPPHTTPLIB_USE_POLL宏，则定义poll宏为WSAPoll
#ifdef CPPHTTPLIB_USE_POLL
#define poll(fds, nfds, timeout) WSAPoll(fds, nfds, timeout)
#endif

#else // not _WIN32

// 包含Unix特有的头文件
#include <arpa/inet.h>
#ifndef _AIX
#include <ifaddrs.h>
#endif
#include <net/if.h>
#include <netdb.h>
#include <netinet/in.h>
#ifdef __linux__
#include <resolv.h>
#endif
#include <netinet/tcp.h>
// 如果定义了CPPHTTPLIB_USE_POLL宏，则包含poll头文件
#ifdef CPPHTTPLIB_USE_POLL
#include <poll.h>
#endif
// 包含必要的头文件
#include <csignal>
#include <pthread.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

// 定义 socket_t 类型为 int
using socket_t = int;
// 如果未定义 INVALID_SOCKET，则定义为 -1
#ifndef INVALID_SOCKET
#define INVALID_SOCKET (-1)
#endif
#endif //_WIN32

// 包含其他必要的头文件
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

// 如果定义了 CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 如果在 Windows 平台
#ifdef _WIN32
#include <wincrypt.h>

// 下面这些在 wincrypt.h 中已经定义，如果使用 BoringSSL 则会导致编译错误
#undef X509_NAME
#undef X509_CERT_PAIR
#undef X509_EXTENSIONS
#undef PKCS7_SIGNER_INFO

// 如果是 MSC 编译器，则链接 crypt32.lib 和 cryptui.lib 库
#ifdef _MSC_VER
#pragma comment(lib, "crypt32.lib")
#pragma comment(lib, "cryptui.lib")
#endif
// 如果是苹果系统，并且定义了 CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#include <TargetConditionals.h>
// 如果是 macOS 系统
#if TARGET_OS_OSX
#include <CoreFoundation/CoreFoundation.h>
#include <Security/Security.h>
#endif // TARGET_OS_OSX
#endif // _WIN32

// 包含 OpenSSL 相关的头文件
#include <openssl/err.h>
#include <openssl/evp.h>
#include <openssl/ssl.h>
#include <openssl/x509v3.h>

// 如果是 Windows 平台，并且定义了 OPENSSL_USE_APPLINK
#if defined(_WIN32) && defined(OPENSSL_USE_APPLINK)
#include <openssl/applink.c>
#endif

#include <iostream>
#include <sstream>

// 如果 OpenSSL 版本低于 1.1.1，则报错
#if OPENSSL_VERSION_NUMBER < 0x1010100fL
#error Sorry, OpenSSL versions prior to 1.1.1 are not supported
// 如果 OpenSSL 版本低于 3.0.0，则定义 SSL_get1_peer_certificate 为 SSL_get_peer_certificate
#elif OPENSSL_VERSION_NUMBER < 0x30000000L
#define SSL_get1_peer_certificate SSL_get_peer_certificate
#endif

#endif

// 如果定义了 CPPHTTPLIB_ZLIB_SUPPORT，则包含 zlib 头文件
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
#include <zlib.h>
#endif

// 如果定义了 CPPHTTPLIB_BROTLI_SUPPORT，则包含 brotli 头文件
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
#include <brotli/decode.h>
#include <brotli/encode.h>
#endif

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

// 定义模板函数 make_unique，用于创建非数组类型的 unique_ptr
template <class T, class... Args>
typename std::enable_if<!std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(Args &&...args) {
  return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 定义模板函数 make_unique，用于创建数组类型的 unique_ptr
template <class T>
typename std::enable_if<std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(std::size_t n) {
  typedef typename std::remove_extent<T>::type RT;
  return std::unique_ptr<T>(new RT[n]);
}

// 定义结构体 ci，用于在不区分大小写的情况下比较两个字符串
struct ci {
  bool operator()(const std::string &s1, const std::string &s2) const {
    return std::lexicographical_compare(s1.begin(), s1.end(), s2.begin(),
                                        s2.end(),
                                        [](unsigned char c1, unsigned char c2) {
                                          return ::tolower(c1) < ::tolower(c2);
                                        });
  }
};

// 定义结构体 scope_exit，用于在作用域结束时执行指定的函数
// 基于 "http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4189"
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

// 使用 detail 命名空间中的 ci 结构体定义 Headers 类型
using Headers = std::multimap<std::string, std::string, detail::ci>;

// 定义 Params 类型为 std::multimap<std::string, std::string>
using Params = std::multimap<std::string, std::string;

// 定义 Match 类型为 std::smatch
using Match = std::smatch;
// 定义一个名为 Progress 的别名，表示一个返回布尔值的函数，参数为当前进度和总进度
using Progress = std::function<bool(uint64_t current, uint64_t total)>;

// 定义一个名为 Response 的结构体
struct Response;

// 定义一个名为 ResponseHandler 的别名，表示一个返回布尔值的函数，参数为一个 Response 对象的引用
using ResponseHandler = std::function<bool(const Response &response)>;

// 定义一个名为 MultipartFormData 的结构体，包含四个字符串成员变量
struct MultipartFormData {
  std::string name;
  std::string content;
  std::string filename;
  std::string content_type;
};

// 定义一个名为 MultipartFormDataItems 的别名，表示一个 MultipartFormData 结构体的向量
using MultipartFormDataItems = std::vector<MultipartFormData>;

// 定义一个名为 MultipartFormDataMap 的别名，表示一个键为字符串，值为 MultipartFormData 结构体的多重映射
using MultipartFormDataMap = std::multimap<std::string, MultipartFormData>;

// 定义一个名为 DataSink 的类
class DataSink {
public:
  // 默认构造函数，初始化 os 和 sb_，并将 sb_ 传入 os
  DataSink() : os(&sb_), sb_(*this) {}

  // 禁用拷贝构造函数和赋值运算符重载
  DataSink(const DataSink &) = delete;
  DataSink &operator=(const DataSink &) = delete;
  DataSink(DataSink &&) = delete;
  DataSink &operator=(DataSink &&) = delete;

  // 写入数据的函数对象
  std::function<bool(const char *data, size_t data_len)> write;
  // 完成时的函数对象
  std::function<void()> done;
  // 带尾部的完成时的函数对象
  std::function<void(const Headers &trailer)> done_with_trailer;
  // 输出流对象
  std::ostream os;

private:
  // 内部类，继承自 std::streambuf
  class data_sink_streambuf : public std::streambuf {
  public:
    // 显式构造函数，接受一个 DataSink 的引用
    explicit data_sink_streambuf(DataSink &sink) : sink_(sink) {}

  protected:
    // 重写 xsputn 函数，将数据写入到 DataSink 的 write 函数对象中
    std::streamsize xsputn(const char *s, std::streamsize n) {
      sink_.write(s, static_cast<size_t>(n));
      return n;
    }

  private:
    // DataSink 的引用
    DataSink &sink_;
  };

  // DataSink 的输出流缓冲区对象
  data_sink_streambuf sb_;
};

// 定义一个名为 ContentProvider 的别名，表示一个返回布尔值的函数，参数为偏移量、长度和 DataSink 的引用
using ContentProvider =
    std::function<bool(size_t offset, size_t length, DataSink &sink)>;

// 定义一个名为 ContentProviderWithoutLength 的别名，表示一个返回布尔值的函数，参数为偏移量和 DataSink 的引用
using ContentProviderWithoutLength =
    std::function<bool(size_t offset, DataSink &sink)>;

// 定义一个名为 ContentProviderResourceReleaser 的别名，表示一个没有返回值的函数，参数为布尔值
using ContentProviderResourceReleaser = std::function<void(bool success)>;

// 定义一个名为 MultipartFormDataProvider 的结构体，包含四个成员变量
struct MultipartFormDataProvider {
  std::string name;
  ContentProviderWithoutLength provider;
  std::string filename;
  std::string content_type;
};

// 定义一个名为 MultipartFormDataProviderItems 的别名，表示一个 MultipartFormDataProvider 结构体的向量
using MultipartFormDataProviderItems = std::vector<MultipartFormDataProvider>;

// 定义一个名为 ContentReceiverWithProgress 的别名，表示一个返回布尔值的函数，参数为数据、数据长度、偏移量和总长度
using ContentReceiverWithProgress =
    std::function<bool(const char *data, size_t data_length, uint64_t offset,
                       uint64_t total_length)>;

// 定义一个名为 ContentReceiver 的别名，表示一个返回布尔值的函数，参数为数据和数据长度
using ContentReceiver =
    std::function<bool(const char *data, size_t data_length)>;

// 定义一个名为 MultipartContentHeader 的
    # 定义一个名为std的函数类型，该函数接受一个MultipartFormData类型的参数，并返回一个布尔值
    std::function<bool(const MultipartFormData &file)>;
# 定义 ContentReader 类
class ContentReader {
public:
  # 定义 Reader 类型别名，接受 ContentReceiver 参数并返回布尔值
  using Reader = std::function<bool(ContentReceiver receiver)>;
  # 定义 MultipartReader 类型别名，接受 MultipartContentHeader 和 ContentReceiver 参数并返回布尔值
  using MultipartReader = std::function<bool(MultipartContentHeader header, ContentReceiver receiver)>;

  # ContentReader 类的构造函数，接受 Reader 和 MultipartReader 参数
  ContentReader(Reader reader, MultipartReader multipart_reader)
      : reader_(std::move(reader)),
        multipart_reader_(std::move(multipart_reader)) {}

  # 重载 () 运算符，接受 MultipartContentHeader 和 ContentReceiver 参数并返回布尔值
  bool operator()(MultipartContentHeader header, ContentReceiver receiver) const {
    return multipart_reader_(std::move(header), std::move(receiver));
  }

  # 重载 () 运算符，接受 ContentReceiver 参数并返回布尔值
  bool operator()(ContentReceiver receiver) const {
    return reader_(std::move(receiver));
  }

  # 定义 Reader 成员变量
  Reader reader_;
  # 定义 MultipartReader 成员变量
  MultipartReader multipart_reader_;
};

# 定义 Range 类型别名，表示 ssize_t 类型的一对值
using Range = std::pair<ssize_t, ssize_t>;
# 定义 Ranges 类型别名，表示 Range 类型的向量
using Ranges = std::vector<Range>;

# 定义 Request 结构体
struct Request {
  # 请求方法
  std::string method;
  # 请求路径
  std::string path;
  # 请求头部
  Headers headers;
  # 请求体
  std::string body;

  # 远程地址
  std::string remote_addr;
  # 远程端口
  int remote_port = -1;
  # 本地地址
  std::string local_addr;
  # 本地端口
  int local_port = -1;

  # 服务器端特有字段
  # 版本
  std::string version;
  # 目标
  std::string target;
  # 参数
  Params params;
  # 多部分表单数据映射
  MultipartFormDataMap files;
  # 范围
  Ranges ranges;
  # 匹配
  Match matches;

  # 客户端端特有字段
  # 响应处理器
  ResponseHandler response_handler;
  # 带进度的内容接收器
  ContentReceiverWithProgress content_receiver;
  # 进度
  Progress progress;
  # 如果支持 OpenSSL，则包含 SSL 对象
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  const SSL *ssl = nullptr;
// 检查是否存在指定的请求头
bool has_header(const std::string &key) const;

// 获取指定请求头的数值，可以指定多个值中的第几个
std::string get_header_value(const std::string &key, size_t id = 0) const;

// 获取指定请求头的数值，可以指定多个值中的第几个，支持模板类型
template <typename T>
T get_header_value(const std::string &key, size_t id = 0) const;

// 获取指定请求头的数值个数
size_t get_header_value_count(const std::string &key) const;

// 设置请求头的值
void set_header(const std::string &key, const std::string &val);

// 检查是否存在指定的参数
bool has_param(const std::string &key) const;

// 获取指定参数的值，可以指定多个值中的第几个
std::string get_param_value(const std::string &key, size_t id = 0) const;

// 获取指定参数的值个数
size_t get_param_value_count(const std::string &key) const;

// 检查是否为多部分表单数据
bool is_multipart_form_data() const;

// 检查是否存在指定的文件
bool has_file(const std::string &key) const;

// 获取指定文件的值
MultipartFormData get_file_value(const std::string &key) const;

// 获取指定文件的值，返回一个文件值的向量
std::vector<MultipartFormData> get_file_values(const std::string &key) const;

// 私有成员...
size_t redirect_count_ = CPPHTTPLIB_REDIRECT_MAX_COUNT;
size_t content_length_ = 0;
ContentProvider content_provider_;
bool is_chunked_content_provider_ = false;
size_t authorization_count_ = 0;
// 定义一个名为 Response 的结构体
struct Response {
  // 版本号
  std::string version;
  // 状态码，默认值为 -1
  int status = -1;
  // 状态原因
  std::string reason;
  // 头部信息
  Headers headers;
  // 响应体
  std::string body;
  // 重定向地址
  std::string location; // Redirect location

  // 检查是否存在指定的头部信息
  bool has_header(const std::string &key) const;
  // 获取指定头部信息的值
  std::string get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定头部信息的值，并转换为指定类型
  template <typename T>
  T get_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定头部信息的值的数量
  size_t get_header_value_count(const std::string &key) const;
  // 设置指定头部信息的值
  void set_header(const std::string &key, const std::string &val);

  // 设置重定向地址和状态码
  void set_redirect(const std::string &url, int status = 302);
  // 设置响应体内容和内容类型
  void set_content(const char *s, size_t n, const std::string &content_type);
  // 设置响应体内容和内容类型
  void set_content(const std::string &s, const std::string &content_type);

  // 设置内容提供者和相关资源释放器
  void set_content_provider(
      size_t length, const std::string &content_type, ContentProvider provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 设置内容提供者和相关资源释放器
  void set_content_provider(
      const std::string &content_type, ContentProviderWithoutLength provider,
      ContentProviderResourceReleaser resource_releaser = nullptr);

  // 设置分块内容提供者和相关资源释放器
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
    // 如果存在资源释放器，则释放相关资源
    if (content_provider_resource_releaser_) {
      content_provider_resource_releaser_(content_provider_success_);
    }
  }

  // 私有成员...
  // 内容长度
  size_t content_length_ = 0;
  // 内容提供者
  ContentProvider content_provider_;
  // 内容提供者资源释放器
  ContentProviderResourceReleaser content_provider_resource_releaser_;
  // 是否为分块内容提供者
  bool is_chunked_content_provider_ = false;
  // 内容提供者是否成功
  bool content_provider_success_ = false;
};

// 定义一个名为 Stream 的类
class Stream {
# 定义一个公共接口类 Stream
public:
  # 虚析构函数，使用默认实现
  virtual ~Stream() = default;

  # 纯虚函数，用于判断流是否可读
  virtual bool is_readable() const = 0;
  # 纯虚函数，用于判断流是否可写
  virtual bool is_writable() const = 0;

  # 纯虚函数，用于从流中读取数据
  virtual ssize_t read(char *ptr, size_t size) = 0;
  # 纯虚函数，用于向流中写入数据
  virtual ssize_t write(const char *ptr, size_t size) = 0;
  # 纯虚函数，获取远程 IP 地址和端口
  virtual void get_remote_ip_and_port(std::string &ip, int &port) const = 0;
  # 纯虚函数，获取本地 IP 地址和端口
  virtual void get_local_ip_and_port(std::string &ip, int &port) const = 0;
  # 纯虚函数，获取套接字
  virtual socket_t socket() const = 0;

  # 模板函数，用于按格式写入数据
  template <typename... Args>
  ssize_t write_format(const char *fmt, const Args &...args);
  # 重载函数，用于向流中写入数据
  ssize_t write(const char *ptr);
  # 重载函数，用于向流中写入数据
  ssize_t write(const std::string &s);
};

# 定义一个任务队列类 TaskQueue
class TaskQueue {
public:
  # 默认构造函数
  TaskQueue() = default;
  # 虚析构函数，使用默认实现
  virtual ~TaskQueue() = default;

  # 纯虚函数，将任务加入队列
  virtual void enqueue(std::function<void()> fn) = 0;
  # 纯虚函数，关闭任务队列
  virtual void shutdown() = 0;

  # 虚函数，空实现，用于空闲时的操作
  virtual void on_idle() {}
};

# 定义一个线程池类 ThreadPool，继承自 TaskQueue
class ThreadPool : public TaskQueue {
public:
  # 显式构造函数，初始化线程池并设置关闭标志为 false
  explicit ThreadPool(size_t n) : shutdown_(false) {
    # 循环创建 n 个工作线程
    while (n) {
      threads_.emplace_back(worker(*this));
      n--;
    }
  }

  # 禁用拷贝构造函数
  ThreadPool(const ThreadPool &) = delete;
  # 虚析构函数，使用默认实现
  ~ThreadPool() override = default;

  # 重写父类的纯虚函数，将任务加入队列
  void enqueue(std::function<void()> fn) override {
    {
      std::unique_lock<std::mutex> lock(mutex_);
      jobs_.push_back(std::move(fn));
    }

    cond_.notify_one();
  }

  # 重写父类的纯虚函数，关闭任务队列
  void shutdown() override {
    # 停止所有工作线程
    {
      std::unique_lock<std::mutex> lock(mutex_);
      shutdown_ = true;
    }

    cond_.notify_all();

    # 等待所有线程结束
    for (auto &t : threads_) {
      t.join();
    }
  }

private:
  # 定义一个内部结构体 worker
  struct worker {
    # 显式构造函数，初始化工作线程并传入线程池的引用
    explicit worker(ThreadPool &pool) : pool_(pool) {}
    // 重载函数调用操作符，用于线程池中的工作线程执行任务
    void operator()() {
      // 无限循环，等待并执行任务
      for (;;) {
        // 定义一个函数对象
        std::function<void()> fn;
        {
          // 创建互斥锁，保护共享资源
          std::unique_lock<std::mutex> lock(pool_.mutex_);

          // 等待条件变量满足，即任务队列不为空或线程池已关闭
          pool_.cond_.wait(
              lock, [&] { return !pool_.jobs_.empty() || pool_.shutdown_; });

          // 如果线程池已关闭且任务队列为空，则跳出循环
          if (pool_.shutdown_ && pool_.jobs_.empty()) { break; }

          // 从任务队列中取出一个任务，并移除
          fn = std::move(pool_.jobs_.front());
          pool_.jobs_.pop_front();
        }

        // 断言任务对象有效
        assert(true == static_cast<bool>(fn));
        // 执行任务
        fn();
      }
    }

    // 友元结构体，用于访问线程池对象
    ThreadPool &pool_;
  };
  friend struct worker;

  // 存储工作线程的容器
  std::vector<std::thread> threads_;
  // 存储任务的队列
  std::list<std::function<void()>> jobs_;

  // 标识线程池是否已关闭
  bool shutdown_;

  // 条件变量，用于线程同步
  std::condition_variable cond_;
  // 互斥锁，保护共享资源
  std::mutex mutex_;
};

// 使用 Logger 函数类型定义日志记录器
using Logger = std::function<void(const Request &, const Response &)>;

// 使用 SocketOptions 函数类型定义套接字选项设置器
using SocketOptions = std::function<void(socket_t sock)>;

// 默认的套接字选项设置函数
void default_socket_options(socket_t sock);

// 服务器类
class Server {
public:
  // 使用 Handler 函数类型定义请求处理器
  using Handler = std::function<void(const Request &, Response &)>;

  // 使用 ExceptionHandler 函数类型定义异常处理器
  using ExceptionHandler =
      std::function<void(const Request &, Response &, std::exception_ptr ep)>;

  // 请求处理器的响应类型枚举
  enum class HandlerResponse {
    Handled,
protected:
  // 处理请求的方法，处理请求流，关闭连接标志，连接是否已关闭标志，设置请求的函数
  bool process_request(Stream &strm, bool close_connection,
                       bool &connection_closed,
                       const std::function<void(Request &)> &setup_request);

  // 服务器套接字的原子类型
  std::atomic<socket_t> svr_sock_{INVALID_SOCKET};
  // 最大保持活动连接数
  size_t keep_alive_max_count_ = CPPHTTPLIB_KEEPALIVE_MAX_COUNT;
  // 保持活动连接的超时时间
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
  // 定义一个Headers对象
  Headers headers;
  // 存储基本目录的向量
  std::vector<MountPointEntry> base_dirs_;

  // 标识服务器是否正在运行
  std::atomic<bool> is_running_{false};
  // 标识操作是否完成
  std::atomic<bool> done_{false};
  // 存储文件扩展名和MIME类型的映射
  std::map<std::string, std::string> file_extension_and_mimetype_map_;
  // 处理文件请求的处理程序
  Handler file_request_handler_;
  // GET请求处理程序
  Handlers get_handlers_;
  // POST请求处理程序
  Handlers post_handlers_;
  // 用于读取内容的POST请求处理程序
  HandlersForContentReader post_handlers_for_content_reader_;
  // PUT请求处理程序
  Handlers put_handlers_;
  // 用于读取内容的PUT请求处理程序
  HandlersForContentReader put_handlers_for_content_reader_;
  // PATCH请求处理程序
  Handlers patch_handlers_;
  // 用于读取内容的PATCH请求处理程序
  HandlersForContentReader patch_handlers_for_content_reader_;
  // DELETE请求处理程序
  Handlers delete_handlers_;
  // 用于读取内容的DELETE请求处理程序
  HandlersForContentReader delete_handlers_for_content_reader_;
  // OPTIONS请求处理程序
  Handlers options_handlers_;
  // 错误处理程序
  HandlerWithResponse error_handler_;
  // 异常处理程序
  ExceptionHandler exception_handler_;
  // 预路由处理程序
  HandlerWithResponse pre_routing_handler_;
  // 后路由处理程序
  Handler post_routing_handler_;
  // 日志记录器
  Logger logger_;
  // 100继续处理程序
  Expect100ContinueHandler expect_100_continue_handler_;

  // 地址族，默认为未指定
  int address_family_ = AF_UNSPEC;
  // 是否启用TCP_NODELAY
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  // 套接字选项，默认选项
  SocketOptions socket_options_ = default_socket_options;

  // 默认头部
  Headers default_headers_;
// 枚举类型，定义了不同的错误类型
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

  // 仅供内部使用
  SSLPeerCouldBeClosed_,  // SSL 对等端可能已关闭
};

// 将 Error 枚举类型转换为字符串
std::string to_string(const Error error);

// 重载输出流操作符，用于输出 Error 对象
std::ostream &operator<<(std::ostream &os, const Error &obj);

// 定义 Result 类
class Result {
public:
  // 构造函数，接受一个 Response 对象指针、一个 Error 对象和一个 Headers 对象
  Result(std::unique_ptr<Response> &&res, Error err,
         Headers &&request_headers = Headers{})
      : res_(std::move(res)), err_(err),
        request_headers_(std::move(request_headers)) {}
  // Response
  // 转换为布尔值，判断是否存在 Response 对象
  operator bool() const { return res_ != nullptr; }
  // 判断是否等于 nullptr
  bool operator==(std::nullptr_t) const { return res_ == nullptr; }
  // 判断是否不等于 nullptr
  bool operator!=(std::nullptr_t) const { return res_ != nullptr; }
  // 获取 Response 对象的引用
  const Response &value() const { return *res_; }
  Response &value() { return *res_; }
  const Response &operator*() const { return *res_; }
  Response &operator*() { return *res_; }
  const Response *operator->() const { return res_.get(); }
  Response *operator->() { return res_.get(); }

  // Error
  // 获取 Error 对象
  Error error() const { return err_; }

  // Request Headers
  // 检查是否存在指定键的请求头
  bool has_request_header(const std::string &key) const;
  // 获取指定键的请求头值
  std::string get_request_header_value(const std::string &key,
                                       size_t id = 0) const;
  // 获取指定键的请求头值，并转换为指定类型
  template <typename T>
  T get_request_header_value(const std::string &key, size_t id = 0) const;
  // 获取指定键的请求头值的数量
  size_t get_request_header_value_count(const std::string &key) const;

private:
  std::unique_ptr<Response> res_;  // Response 对象指针
  Error err_;  // Error 对象
  Headers request_headers_;  // 请求头对象
};

// 定义 ClientImpl 类
class ClientImpl {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要认证的用户名和密码
  void set_digest_auth(const std::string &username,
                       const std::string &password);
  #endif

  // 设置是否保持连接
  void set_keep_alive(bool on);
  // 设置是否跟随重定向
  void set_follow_location(bool on);

  // 设置是否进行 URL 编码
  void set_url_encode(bool on);

  // 设置是否压缩请求数据
  void set_compress(bool on);

  // 设置是否解压响应数据
  void set_decompress(bool on);

  // 设置网络接口
  void set_interface(const std::string &intf);

  // 设置代理服务器
  void set_proxy(const std::string &host, int port);
  // 设置代理服务器的基本认证信息
  void set_proxy_basic_auth(const std::string &username,
                            const std::string &password);
  // 设置代理服务器的 Bearer Token 认证信息
  void set_proxy_bearer_token_auth(const std::string &token);
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置代理服务器的摘要认证信息
  void set_proxy_digest_auth(const std::string &username,
                             const std::string &password);
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置 CA 证书路径
  void set_ca_cert_path(const std::string &ca_cert_file_path,
                        const std::string &ca_cert_dir_path = std::string());
  // 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 启用/禁用服务器证书验证
  void enable_server_certificate_verification(bool enabled);
#endif

  // 设置日志记录器
  void set_logger(Logger logger);

protected:
  struct Socket {
    socket_t sock = INVALID_SOCKET;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    SSL *ssl = nullptr;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  std::string digest_auth_username_;
  std::string digest_auth_password_;
#endif

  // 是否保持连接
  bool keep_alive_ = false;
  // 是否跟随重定向
  bool follow_location_ = false;

  // 是否进行 URL 编码
  bool url_encode_ = true;

  // 地址族
  int address_family_ = AF_UNSPEC;
  // 是否启用 TCP_NODELAY
  bool tcp_nodelay_ = CPPHTTPLIB_TCP_NODELAY;
  // 套接字选项
  SocketOptions socket_options_ = nullptr;

  // 是否压缩请求数据
  bool compress_ = false;
  // 是否解压响应数据
  bool decompress_ = true;

  // 网络接口
  std::string interface_;

  // 代理服务器主机和端口
  std::string proxy_host_;
  int proxy_port_ = -1;

  // 代理服务器的基本认证信息
  std::string proxy_basic_auth_username_;
  std::string proxy_basic_auth_password_;
  // 代理服务器的 Bearer Token 认证信息
  std::string proxy_bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 代理服务器的摘要认证信息
  std::string proxy_digest_auth_username_;
  std::string proxy_digest_auth_password_;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // CA 证书文件路径
  std::string ca_cert_file_path_;
  // CA 证书目录路径
  std::string ca_cert_dir_path_;

  // CA 证书存储
  X509_STORE *ca_cert_store_ = nullptr;
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则启用服务器证书验证
  bool server_certificate_verification_ = true;
#endif

  // 日志记录器
  Logger logger_;

private:
  // 发送请求的私有方法
  bool send_(Request &req, Response &res, Error &error);
  // 发送请求的私有方法，使用移动语义
  Result send_(Request &&req);

  // 创建客户端套接字
  socket_t create_client_socket(Error &error) const;
  // 读取响应行
  bool read_response_line(Stream &strm, const Request &req, Response &res);
  // 写入请求
  bool write_request(Stream &strm, Request &req, bool close_connection,
                     Error &error);
  // 重定向
  bool redirect(Request &req, Response &res, Error &error);
  // 处理请求
  bool handle_request(Stream &strm, Request &req, Response &res,
                      bool close_connection, Error &error);
  // 使用内容提供程序发送请求
  std::unique_ptr<Response> send_with_content_provider(
      Request &req, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type, Error &error);
  // 使用内容提供程序发送请求，返回结果
  Result send_with_content_provider(
      const std::string &method, const std::string &path,
      const Headers &headers, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type);
  // 获取多部分内容提供程序
  ContentProviderWithoutLength get_multipart_content_provider(
      const std::string &boundary, const MultipartFormDataItems &items,
      const MultipartFormDataProviderItems &provider_items);

  // 调整主机字符串
  std::string adjust_host_string(const std::string &host) const;

  // 处理套接字的虚拟方法
  virtual bool process_socket(const Socket &socket,
                              std::function<bool(Stream &strm)> callback);
  // 是否使用 SSL
  virtual bool is_ssl() const;
};

class Client {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置摘要认证
  void set_digest_auth(const std::string &username,
                       const std::string &password);
#endif

  // 设置是否保持连接
  void set_keep_alive(bool on);
  // 设置是否跟随重定向
  void set_follow_location(bool on);

  // 设置是否进行 URL 编码
  void set_url_encode(bool on);

  // 设置是否压缩请求
  void set_compress(bool on);

  // 设置是否解压缩响应
  void set_decompress(bool on);

  // 设置网络接口
  void set_interface(const std::string &intf);

  // 设置代理服务器
  void set_proxy(const std::string &host, int port);
  // 设置代理服务器的基本认证
  void set_proxy_basic_auth(const std::string &username,
                            const std::string &password);
  // 设置代理服务器的 Bearer Token 认证
  void set_proxy_bearer_token_auth(const std::string &token);
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置代理服务器的摘要认证
  void set_proxy_digest_auth(const std::string &username,
                             const std::string &password);
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 启用/禁用服务器证书验证
  void enable_server_certificate_verification(bool enabled);
#endif

  // 设置日志记录器
  void set_logger(Logger logger);

  // SSL
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 设置 CA 证书路径
  void set_ca_cert_path(const std::string &ca_cert_file_path,
                        const std::string &ca_cert_dir_path = std::string());
  // 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);
  // 获取 OpenSSL 验证结果
  long get_openssl_verify_result() const;
  // 获取 SSL 上下文
  SSL_CTX *ssl_context() const;
#endif

private:
  // 客户端实现对象
  std::unique_ptr<ClientImpl> cli_;

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 是否使用 SSL
  bool is_ssl_ = false;
#endif
};

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// SSL 服务器类
class SSLServer : public Server {
public:
  // 构造函数，使用证书路径和私钥路径
  SSLServer(const char *cert_path, const char *private_key_path,
            const char *client_ca_cert_file_path = nullptr,
            const char *client_ca_cert_dir_path = nullptr,
            const char *private_key_password = nullptr);
  // 构造函数，使用证书和私钥
  SSLServer(X509 *cert, EVP_PKEY *private_key,
            X509_STORE *client_ca_cert_store = nullptr);
  // 构造函数，使用 SSL 上下文设置回调函数
  SSLServer(
      const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback);
  // 析构函数
  ~SSLServer() override;
  // 是否有效
  bool is_valid() const override;
  // 获取 SSL 上下文
  SSL_CTX *ssl_context() const;

private:
  // 处理并关闭套接字
  bool process_and_close_socket(socket_t sock) override;
  // SSL 上下文
  SSL_CTX *ctx_;
  // 上下文互斥锁
  std::mutex ctx_mutex_;
};

// SSL 客户端类
class SSLClient : public ClientImpl {
// SSLClient 类的公共接口
public:
  // 构造函数，根据主机名创建 SSLClient 对象
  explicit SSLClient(const std::string &host);

  // 构造函数，根据主机名和端口号创建 SSLClient 对象
  explicit SSLClient(const std::string &host, int port);

  // 构造函数，根据主机名、端口号、客户端证书路径和客户端密钥路径创建 SSLClient 对象
  explicit SSLClient(const std::string &host, int port,
                     const std::string &client_cert_path,
                     const std::string &client_key_path);

  // 构造函数，根据主机名、端口号、客户端证书和客户端密钥创建 SSLClient 对象
  explicit SSLClient(const std::string &host, int port, X509 *client_cert,
                     EVP_PKEY *client_key);

  // 析构函数
  ~SSLClient() override;

  // 检查 SSLClient 对象是否有效
  bool is_valid() const override;

  // 设置 CA 证书存储
  void set_ca_cert_store(X509_STORE *ca_cert_store);

  // 获取 OpenSSL 验证结果
  long get_openssl_verify_result() const;

  // 获取 SSL 上下文
  SSL_CTX *ssl_context() const;

private:
  // 创建并连接套接字
  bool create_and_connect_socket(Socket &socket, Error &error) override;
  // 关闭 SSL 连接
  void shutdown_ssl(Socket &socket, bool shutdown_gracefully) override;
  // 实际关闭 SSL 连接
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
  // 仅初始化一次的标志
  std::once_flag initialize_cert_;

  // 主机名组件
  std::vector<std::string> host_components_;

  // 验证结果
  long verify_result_ = 0;

  // 友元类
  friend class ClientImpl;
};
#endif

/*
 * 模板方法的实现
 */

namespace detail {

template <typename T, typename U>
// 将持续时间转换为秒和微秒，并通过回调函数返回结果
inline void duration_to_sec_and_usec(const T &duration, U callback) {
  // 将持续时间转换为秒，并获取秒数
  auto sec = std::chrono::duration_cast<std::chrono::seconds>(duration).count();
  // 将持续时间减去秒数后转换为微秒，并获取微秒数
  auto usec = std::chrono::duration_cast<std::chrono::microseconds>(
                  duration - std::chrono::seconds(sec))
                  .count();
  // 调用回调函数，传入秒数和微秒数
  callback(static_cast<time_t>(sec), static_cast<time_t>(usec));
}

// 获取指定键的头部值，如果未找到则返回默认值
template <typename T>
inline T get_header_value(const Headers & /*headers*/,
                          const std::string & /*key*/, size_t /*id*/ = 0,
                          uint64_t /*def*/ = 0) {}

// 获取指定键的头部值，返回 uint64_t 类型的值
template <>
inline uint64_t get_header_value<uint64_t>(const Headers &headers,
                                           const std::string &key, size_t id,
                                           uint64_t def) {
  // 获取指定键的头部值的范围
  auto rng = headers.equal_range(key);
  auto it = rng.first;
  // 将迭代器移动到指定位置
  std::advance(it, static_cast<ssize_t>(id));
  // 如果找到对应值，则将其转换为 uint64_t 类型并返回
  if (it != rng.second) {
    return std::strtoull(it->second.data(), nullptr, 10);
  }
  // 如果未找到对应值，则返回默认值
  return def;
}

// 获取请求对象中指定键的头部值
template <typename T>
inline T Request::get_header_value(const std::string &key, size_t id) const {
  return detail::get_header_value<T>(headers, key, id, 0);
}

// 获取响应对象中指定键的头部值
template <typename T>
inline T Response::get_header_value(const std::string &key, size_t id) const {
  return detail::get_header_value<T>(headers, key, id, 0);
}

// 格式化写入数据到流中
template <typename... Args>
inline ssize_t Stream::write_format(const char *fmt, const Args &...args) {
  const auto bufsiz = 2048;
  std::array<char, bufsiz> buf{};

  // 使用给定格式和参数将数据格式化写入缓冲区
  auto sn = snprintf(buf.data(), buf.size() - 1, fmt, args...);
  // 如果格式化失败，则返回错误码
  if (sn <= 0) { return sn; }

  auto n = static_cast<size_t>(sn);

  // 如果格式化后的数据超出缓冲区大小，则使用动态缓冲区进行写入
  if (n >= buf.size() - 1) {
    std::vector<char> glowable_buf(buf.size());

    while (n >= glowable_buf.size() - 1) {
      glowable_buf.resize(glowable_buf.size() * 2);
      n = static_cast<size_t>(
          snprintf(&glowable_buf[0], glowable_buf.size() - 1, fmt, args...));
    }
    return write(&glowable_buf[0], n);
  } else {
    // 如果格式化后的数据未超出缓冲区大小，则直接写入缓冲区
    # 返回调用 write 函数，传入 buf.data() 和 n 作为参数，并返回其结果
    return write(buf.data(), n);
  }
// 设置套接字的默认选项
inline void default_socket_options(socket_t sock) {
  int yes = 1;
#ifdef _WIN32
  // 设置套接字选项，允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, reinterpret_cast<char *>(&yes),
             sizeof(yes));
  // 设置套接字选项，独占地址使用
  setsockopt(sock, SOL_SOCKET, SO_EXCLUSIVEADDRUSE,
             reinterpret_cast<char *>(&yes), sizeof(yes));
#else
#ifdef SO_REUSEPORT
  // 设置套接字选项，允许端口重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEPORT, reinterpret_cast<void *>(&yes),
             sizeof(yes));
#else
  // 设置套接字选项，允许地址重用
  setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, reinterpret_cast<void *>(&yes),
             sizeof(yes));
#endif
#endif
}

// 设置读取超时时间
template <class Rep, class Period>
inline Server &
Server::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 将时间转换为秒和微秒，并设置读取超时时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
  return *this;
}

// 设置写入超时时间
template <class Rep, class Period>
inline Server &
Server::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 将时间转换为秒和微秒，并设置写入超时时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_write_timeout(sec, usec); });
  return *this;
}

// 设置空闲间隔时间
template <class Rep, class Period>
inline Server &
Server::set_idle_interval(const std::chrono::duration<Rep, Period> &duration) {
  // 将时间转换为秒和微秒，并设置空闲间隔时间
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_idle_interval(sec, usec); });
  return *this;
}
// 将错误枚举值转换为对应的字符串描述
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
  case Error::Unknown: return "Unknown";
  default: break;
  }
  // 默认返回无效错误描述
  return "Invalid";
}

// 重载输出流操作符，将错误枚举值转换为对应的字符串描述并输出
inline std::ostream &operator<<(std::ostream &os, const Error &obj) {
  os << to_string(obj);
  os << " (" << static_cast<std::underlying_type<Error>::type>(obj) << ')';
  return os;
}

// 获取请求头中指定键的值
template <typename T>
inline T Result::get_request_header_value(const std::string &key,
                                          size_t id) const {
  return detail::get_header_value<T>(request_headers_, key, id, 0);
}

// 设置连接超时时间
template <class Rep, class Period>
inline void ClientImpl::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  detail::duration_to_sec_and_usec(duration, [&](time_t sec, time_t usec) {
    set_connection_timeout(sec, usec);
  });
}

// 设置读取超时时间
template <class Rep, class Period>
inline void ClientImpl::set_read_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_read_timeout(sec, usec); });
}

// 其他模板函数定义
// 设置写入超时时间
inline void ClientImpl::set_write_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 将持续时间转换为秒和微秒，并调用 set_write_timeout 函数
  detail::duration_to_sec_and_usec(
      duration, [&](time_t sec, time_t usec) { set_write_timeout(sec, usec); });
}

// 设置连接超时时间
template <class Rep, class Period>
inline void Client::set_connection_timeout(
    const std::chrono::duration<Rep, Period> &duration) {
  // 调用 cli_ 对象的 set_connection_timeout 函数
  cli_->set_connection_timeout(duration);
}

// 设置读取超时时间
template <class Rep, class Period>
inline void
Client::set_read_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 调用 cli_ 对象的 set_read_timeout 函数
  cli_->set_read_timeout(duration);
}

// 设置写入超时时间
template <class Rep, class Period>
inline void
Client::set_write_timeout(const std::chrono::duration<Rep, Period> &duration) {
  // 调用 cli_ 对象的 set_write_timeout 函数
  cli_->set_write_timeout(duration);
}

/*
 * 如果拆分为 .h + .cc 文件，以下是将包含在 .h 文件中的前置声明和类型。
 */

// 获取主机名所在的地址
std::string hosted_at(const std::string &hostname);

// 获取主机名所在的地址
void hosted_at(const std::string &hostname, std::vector<std::string> &addrs);

// 在路径后附加查询参数
std::string append_query_params(const std::string &path, const Params &params);

// 生成范围头部
std::pair<std::string, std::string> make_range_header(Ranges ranges);

// 生成基本认证头部
std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password,
                                 bool is_proxy = false);

// 命名空间 detail 中的函数声明
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

// 处理客户端套接字
bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                           time_t read_timeout_usec, time_t write_timeout_sec,
                           time_t write_timeout_usec,
                           std::function<bool(Stream &)> callback);
}
# 创建客户端套接字，连接指定的主机和端口
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

# 解析查询文本并存储到参数中
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

# 缓冲流类，继承自流
class BufferStream : public Stream {
public:
  BufferStream() = default;
  ~BufferStream() override = default;

  bool is_readable() const override;
  bool is_writable() const override;
  ssize_t read(char *ptr, size_t size) override;
  ssize_t write(const char *ptr, size_t size) override;
  void get_remote_ip_and_port(std::string &ip, int &port) const override;
  void get_local_ip_and_port(std::string &ip, int &port) const override;
  socket_t socket() const override;

  const std::string &get_buffer() const;

private:
  std::string buffer;
  size_t position = 0;
};

# 压缩器类
class compressor {
public:
  virtual ~compressor() = default;

  typedef std::function<bool(const char *data, size_t data_len)> Callback;
  virtual bool compress(const char *data, size_t data_length, bool last,
                        Callback callback) = 0;
};
// 定义一个名为 decompressor 的类
class decompressor {
public:
  // 虚析构函数
  virtual ~decompressor() = default;

  // 声明一个纯虚函数，用于检查解压器是否有效
  virtual bool is_valid() const = 0;

  // 定义一个类型为 Callback 的函数指针，用于回调
  typedef std::function<bool(const char *data, size_t data_len)> Callback;
  // 声明一个纯虚函数，用于解压数据
  virtual bool decompress(const char *data, size_t data_length,
                          Callback callback) = 0;
};

// 定义一个名为 nocompressor 的类，继承自 compressor 类
class nocompressor : public compressor {
public:
  // 虚析构函数
  virtual ~nocompressor() = default;

  // 重写父类的 compress 函数，用于压缩数据
  bool compress(const char *data, size_t data_length, bool /*last*/,
                Callback callback) override;
};

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 定义一个名为 gzip_compressor 的类，继承自 compressor 类
class gzip_compressor : public compressor {
public:
  // 构造函数
  gzip_compressor();
  // 析构函数
  ~gzip_compressor();

  // 重写父类的 compress 函数，用于压缩数据
  bool compress(const char *data, size_t data_length, bool last,
                Callback callback) override;

private:
  // 声明一个私有变量，用于标识是否有效
  bool is_valid_ = false;
  // 声明一个私有变量，用于存储压缩流
  z_stream strm_;
};

// 定义一个名为 gzip_decompressor 的类，继承自 decompressor 类
class gzip_decompressor : public decompressor {
public:
  // 构造函数
  gzip_decompressor();
  // 析构函数
  ~gzip_decompressor();

  // 实现 is_valid 函数，用于检查解压器是否有效
  bool is_valid() const override;

  // 实现 decompress 函数，用于解压数据
  bool decompress(const char *data, size_t data_length,
                  Callback callback) override;

private:
  // 声明一个私有变量，用于标识是否有效
  bool is_valid_ = false;
  // 声明一个私有变量，用于存储解压缩流
  z_stream strm_;
};
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
// 定义一个名为 brotli_compressor 的类，继承自 compressor 类
class brotli_compressor : public compressor {
public:
  // 构造函数
  brotli_compressor();
  // 析构函数
  ~brotli_compressor();

  // 重写父类的 compress 函数，用于压缩数据
  bool compress(const char *data, size_t data_length, bool last,
                Callback callback) override;

private:
  // 声明一个私有变量，用于存储压缩状态
  BrotliEncoderState *state_ = nullptr;
};

// 定义一个名为 brotli_decompressor 的类，继承自 decompressor 类
class brotli_decompressor : public decompressor {
public:
  // 构造函数
  brotli_decompressor();
  // 析构函数
  ~brotli_decompressor();

  // 实现 is_valid 函数，用于检查解压器是否有效
  bool is_valid() const override;

  // 实现 decompress 函数，用于解压数据
  bool decompress(const char *data, size_t data_length,
                  Callback callback) override;

private:
  // 声明一个私有变量，用于存储解压结果
  BrotliDecoderResult decoder_r;
  // 声明一个私有变量，用于存储解压状态
  BrotliDecoderState *decoder_s = nullptr;
};
#endif

// 注意：直到读取的大小达到 `fixed_buffer_size`，使用 `fixed_buffer` 存储数据。调用可以在堆栈上设置内存以提高性能。
// 定义一个名为 stream_line_reader 的类
class stream_line_reader {
// 定义一个公共类，包含一些成员函数和私有成员变量
public:
  // 构造函数，接受一个流和一个固定大小的缓冲区
  stream_line_reader(Stream &strm, char *fixed_buffer,
                     size_t fixed_buffer_size);
  // 返回指向缓冲区的指针
  const char *ptr() const;
  // 返回缓冲区的大小
  size_t size() const;
  // 检查缓冲区是否以CRLF结尾
  bool end_with_crlf() const;
  // 从流中读取一行数据到缓冲区
  bool getline();

private:
  // 在缓冲区末尾追加一个字符
  void append(char c);

  // 引用流对象
  Stream &strm_;
  // 指向固定大小缓冲区的指针
  char *fixed_buffer_;
  // 固定大小缓冲区的大小
  const size_t fixed_buffer_size_;
  // 固定大小缓冲区已使用的大小
  size_t fixed_buffer_used_size_ = 0;
  // 可增长缓冲区
  std::string glowable_buffer_;
};

} // namespace detail

// ----------------------------------------------------------------------------

/*
 * 如果拆分为.h和.cc文件，这部分实现将成为.cc文件的一部分。
 */

namespace detail {

// 判断字符是否为十六进制，并将其转换为整数
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
    int v = 0;
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
  const char *charset = "0123456789abcdef";
  std::string ret;
  do {
    ret = charset[n & 15] + ret;
    n >>= 4;
  } while (n > 0);
  return ret;
}

// 将Unicode编码转换为UTF-8编码
inline size_t to_utf8(int code, char *buff) {
  if (code < 0x0080) {
    buff[0] = (code & 0x7F);
    return 1;
  } else if (code < 0x0800) {
    buff[0] = static_cast<char>(0xC0 | ((code >> 6) & 0x1F));
    buff[1] = static_cast<char>(0x80 | (code & 0x3F));
    return 2;
  } else if (code < 0xD800) {
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    return 3;
  } else if (code < 0xE000) { // D800 - DFFF is invalid...
    // 如果 Unicode 编码小于 0x80，则直接返回 0
    return 0;
  } else if (code < 0x10000) {
    // 如果 Unicode 编码在 0x80 到 0xFFFF 之间，则使用 3 个字节表示
    buff[0] = static_cast<char>(0xE0 | ((code >> 12) & 0xF));
    buff[1] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[2] = static_cast<char>(0x80 | (code & 0x3F));
    return 3;
  } else if (code < 0x110000) {
    // 如果 Unicode 编码在 0x10000 到 0x10FFFF 之间，则使用 4 个字节表示
    buff[0] = static_cast<char>(0xF0 | ((code >> 18) & 0x7));
    buff[1] = static_cast<char>(0x80 | ((code >> 12) & 0x3F));
    buff[2] = static_cast<char>(0x80 | ((code >> 6) & 0x3F));
    buff[3] = static_cast<char>(0x80 | (code & 0x3F));
    return 4;
  }

  // 不会执行到这里
  // 如果 Unicode 编码大于 0x10FFFF，则返回 0
  return 0;
// 结束上一个函数或代码块
}

// NOTE: This code came up with the following stackoverflow post:
// https://stackoverflow.com/questions/180947/base64-decode-snippet-in-c
// 定义一个内联函数，用于将输入字符串进行 base64 编码
inline std::string base64_encode(const std::string &in) {
  // 定义 base64 编码表
  static const auto lookup =
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

  // 初始化输出字符串
  std::string out;
  out.reserve(in.size());

  // 初始化变量
  int val = 0;
  int valb = -6;

  // 遍历输入字符串
  for (auto c : in) {
    // 将字符转换为对应的整数值
    val = (val << 8) + static_cast<uint8_t>(c);
    valb += 8;
    // 当 valb 大于等于 0 时，进行 base64 编码
    while (valb >= 0) {
      out.push_back(lookup[(val >> valb) & 0x3F]);
      valb -= 6;
    }
  }

  // 处理剩余的位数
  if (valb > -6) { out.push_back(lookup[((val << 8) >> (valb + 8)) & 0x3F]); }

  // 补全 base64 编码后的字符串
  while (out.size() % 4) {
    out.push_back('=');
  }

  // 返回 base64 编码后的字符串
  return out;
}

// 定义一个内联函数，用于判断给定路径是否为文件
inline bool is_file(const std::string &path) {
#ifdef _WIN32
  return _access_s(path.c_str(), 0) == 0;
#else
  struct stat st;
  return stat(path.c_str(), &st) >= 0 && S_ISREG(st.st_mode);
#endif
}

// 定义一个内联函数，用于判断给定路径是否为目录
inline bool is_dir(const std::string &path) {
  struct stat st;
  return stat(path.c_str(), &st) >= 0 && S_ISDIR(st.st_mode);
}

// 定义一个内联函数，用于判断给定路径是否为有效路径
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

    // 判断路径组件
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

// 定义一个内联函数，用于对查询参数进行编码
inline std::string encode_query_param(const std::string &value) {
  std::ostringstream escaped;
  escaped.fill('0');
  escaped << std::hex;

  for (auto c : value) {
    # 检查字符是否是字母或数字，或者是特殊字符 - _ . ! ~ * ' ( )
    if (std::isalnum(static_cast<uint8_t>(c)) || c == '-' || c == '_' ||
        c == '.' || c == '!' || c == '~' || c == '*' || c == '\'' || c == '(' ||
        c == ')') {
      # 如果是字母或数字，或者是特殊字符，则直接添加到转义后的字符串中
      escaped << c;
    } else {
      # 如果是其他字符，则进行 URL 编码
      escaped << std::uppercase;
      escaped << '%' << std::setw(2)
              << static_cast<int>(static_cast<unsigned char>(c));
      escaped << std::nouppercase;
    }
  }
  # 返回 URL 编码后的字符串
  return escaped.str();
// 将输入的字符串进行 URL 编码处理，返回编码后的字符串
inline std::string encode_url(const std::string &s) {
  // 预先分配足够的空间以容纳编码后的字符串
  std::string result;
  result.reserve(s.size());

  // 遍历输入字符串的每个字符
  for (size_t i = 0; s[i]; i++) {
    // 根据字符的不同情况进行不同的编码处理
    switch (s[i]) {
    case ' ': result += "%20"; break;
    case '+': result += "%2B"; break;
    case '\r': result += "%0D"; break;
    case '\n': result += "%0A"; break;
    case '\'': result += "%27"; break;
    case ',': result += "%2C"; break;
    // case ':': result += "%3A"; break; // ok? probably...
    case ';': result += "%3B"; break;
    default:
      // 如果字符是非 ASCII 字符，则进行百分号编码处理
      auto c = static_cast<uint8_t>(s[i]);
      if (c >= 0x80) {
        result += '%';
        char hex[4];
        auto len = snprintf(hex, sizeof(hex) - 1, "%02X", c);
        assert(len == 2);
        result.append(hex, static_cast<size_t>(len));
      } else {
        result += s[i];
      }
      break;
    }
  }

  return result;
}

// 将输入的字符串进行 URL 解码处理，返回解码后的字符串
inline std::string decode_url(const std::string &s,
                              bool convert_plus_to_space) {
  std::string result;

  // 遍历输入字符串的每个字符
  for (size_t i = 0; i < s.size(); i++) {
    // 如果遇到百分号编码的字符，则进行解码处理
    if (s[i] == '%' && i + 1 < s.size()) {
      if (s[i + 1] == 'u') {
        int val = 0;
        if (from_hex_to_i(s, i + 2, 4, val)) {
          // 4 位 Unicode 编码
          char buff[4];
          size_t len = to_utf8(val, buff);
          if (len > 0) { result.append(buff, len); }
          i += 5; // 'u0000'
        } else {
          result += s[i];
        }
      } else {
        int val = 0;
        if (from_hex_to_i(s, i + 1, 2, val)) {
          // 2 位十六进制编码
          result += static_cast<char>(val);
          i += 2; // '00'
        } else {
          result += s[i];
        }
      }
    } else if (convert_plus_to_space && s[i] == '+') {
      result += ' ';
    } else {
      result += s[i];
    }
  }

  return result;
}
// 从指定路径读取文件内容到字符串中
inline void read_file(const std::string &path, std::string &out) {
  // 以二进制模式打开文件流
  std::ifstream fs(path, std::ios_base::binary);
  // 定位到文件末尾，获取文件大小
  fs.seekg(0, std::ios_base::end);
  auto size = fs.tellg();
  // 重新定位到文件开头
  fs.seekg(0);
  // 调整输出字符串大小以容纳文件内容
  out.resize(static_cast<size_t>(size));
  // 读取文件内容到输出字符串中
  fs.read(&out[0], static_cast<std::streamsize>(size));
}

// 获取文件路径的扩展名
inline std::string file_extension(const std::string &path) {
  std::smatch m;
  static auto re = std::regex("\\.([a-zA-Z0-9]+)$");
  // 使用正则表达式匹配文件路径中的扩展名
  if (std::regex_search(path, m, re)) { return m[1].str(); }
  return std::string();
}

// 判断字符是否为空格或制表符
inline bool is_space_or_tab(char c) { return c == ' ' || c == '\t'; }

// 去除字符串两端的空格和制表符
inline std::pair<size_t, size_t> trim(const char *b, const char *e, size_t left,
                                      size_t right) {
  // 去除左端空格和制表符
  while (b + left < e && is_space_or_tab(b[left])) {
    left++;
  }
  // 去除右端空格和制表符
  while (right > 0 && is_space_or_tab(b[right - 1])) {
    right--;
  }
  return std::make_pair(left, right);
}

// 返回去除两端空格和制表符后的字符串副本
inline std::string trim_copy(const std::string &s) {
  auto r = trim(s.data(), s.data() + s.size(), 0, s.size());
  return s.substr(r.first, r.second - r.first);
}

// 根据指定分隔符拆分字符串，并对每个部分执行指定函数
inline void split(const char *b, const char *e, char d,
                  std::function<void(const char *, const char *)> fn) {
  size_t i = 0;
  size_t beg = 0;

  while (e ? (b + i < e) : (b[i] != '\0')) {
    if (b[i] == d) {
      auto r = trim(b, e, beg, i);
      if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
      beg = i + 1;
    }
    i++;
  }

  if (i) {
    auto r = trim(b, e, beg, i);
    if (r.first < r.second) { fn(&b[r.first], &b[r.second]); }
  }
}

// 从流中读取一行数据
inline stream_line_reader::stream_line_reader(Stream &strm, char *fixed_buffer,
                                              size_t fixed_buffer_size)
    : strm_(strm), fixed_buffer_(fixed_buffer),
      fixed_buffer_size_(fixed_buffer_size) {}

// 返回当前读取位置的指针
inline const char *stream_line_reader::ptr() const {
  // 如果动态缓冲区为空，则返回固定缓冲区的指针，否则返回动态缓冲区的指针
  if (glowable_buffer_.empty()) {
    return fixed_buffer_;
  } else {
    return glowable_buffer_.data();
  }
}
// 返回当前流行读取器的大小
inline size_t stream_line_reader::size() const {
  // 如果可增长缓冲区为空，则返回固定缓冲区使用的大小
  if (glowable_buffer_.empty()) {
    return fixed_buffer_used_size_;
  } else {
    return glowable_buffer_.size();
  }
}

// 检查流行读取器是否以CRLF结尾
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
        return false;
      } else {
        break;
      }
    }

    append(byte);

    if (byte == '\n') { break; }
  }

  return true;
}

// 向缓冲区追加字符
inline void stream_line_reader::append(char c) {
  if (fixed_buffer_used_size_ < fixed_buffer_size_ - 1) {
    fixed_buffer_[fixed_buffer_used_size_++] = c;
    fixed_buffer_[fixed_buffer_used_size_] = '\0';
  } else {
    if (glowable_buffer_.empty()) {
      assert(fixed_buffer_[fixed_buffer_used_size_] == '\0');
      glowable_buffer_.assign(fixed_buffer_, fixed_buffer_used_size_);
    }
    glowable_buffer_ += c;
  }
}

// 关闭套接字
inline int close_socket(socket_t sock) {
#ifdef _WIN32
  return closesocket(sock);
#else
  return close(sock);
#endif
}

// 处理EINTR错误
template <typename T> inline ssize_t handle_EINTR(T fn) {
  ssize_t res = false;
  while (true) {
    res = fn();
    if (res < 0 && errno == EINTR) { continue; }
    break;
  }
  return res;
}

// 从套接字中读取数据
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

// 向套接字发送数据
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
// 根据操作系统选择合适的参数类型进行读取操作

inline ssize_t select_read(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建用于读取的 pollfd 结构体
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLIN;

  // 计算超时时间
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 使用 poll 函数进行读取操作
  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 检查套接字是否超出范围
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 创建文件描述符集合
  fd_set fds;
  FD_ZERO(&fds);
  FD_SET(sock, &fds);

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 使用 select 函数进行读取操作
  return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), &fds, nullptr, nullptr, &tv);
  });
#endif
}

inline ssize_t select_write(socket_t sock, time_t sec, time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建用于写入的 pollfd 结构体
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLOUT;

  // 计算超时时间
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 使用 poll 函数进行写入操作
  return handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });
#else
#ifndef _WIN32
  // 检查套接字是否超出范围
  if (sock >= FD_SETSIZE) { return 1; }
#endif

  // 创建文件描述符集合
  fd_set fds;
  FD_ZERO(&fds);
  FD_SET(sock, &fds);

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 使用 select 函数进行写入操作
  return handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), nullptr, &fds, nullptr, &tv);
  });
#endif
}

inline Error wait_until_socket_is_ready(socket_t sock, time_t sec,
                                        time_t usec) {
#ifdef CPPHTTPLIB_USE_POLL
  // 创建用于读写的 pollfd 结构体
  struct pollfd pfd_read;
  pfd_read.fd = sock;
  pfd_read.events = POLLIN | POLLOUT;

  // 计算超时时间
  auto timeout = static_cast<int>(sec * 1000 + usec / 1000);

  // 使用 poll 函数等待套接字就绪
  auto poll_res = handle_EINTR([&]() { return poll(&pfd_read, 1, timeout); });

  // 如果超时，则返回连接超时错误
  if (poll_res == 0) { return Error::ConnectionTimeout; }

  // 如果成功并且套接字就绪，则返回无错误
  if (poll_res > 0 && pfd_read.revents & (POLLIN | POLLOUT)) {
    int error = 0;
    // 获取错误信息的长度
    socklen_t len = sizeof(error);
    // 获取套接字选项信息，存储在error中
    auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                          reinterpret_cast<char *>(&error), &len);
    // 判断是否获取成功并且没有错误
    auto successful = res >= 0 && !error;
    // 如果成功则返回Success，否则返回Connection
    return successful ? Error::Success : Error::Connection;
  }

  // 如果获取套接字选项信息失败，则返回Connection
  return Error::Connection;
#else
#ifndef _WIN32
  // 如果不是 Windows 平台，且套接字超出范围，则返回连接错误
  if (sock >= FD_SETSIZE) { return Error::Connection; }
#endif

  // 创建文件描述符集合并清空
  fd_set fdsr;
  FD_ZERO(&fdsr);
  // 将套接字加入到读取集合中
  FD_SET(sock, &fdsr);

  // 复制读取集合到写入和异常集合
  auto fdsw = fdsr;
  auto fdse = fdsr;

  // 设置超时时间
  timeval tv;
  tv.tv_sec = static_cast<long>(sec);
  tv.tv_usec = static_cast<decltype(tv.tv_usec)>(usec);

  // 调用 select 函数，处理中断
  auto ret = handle_EINTR([&]() {
    return select(static_cast<int>(sock + 1), &fdsr, &fdsw, &fdse, &tv);
  });

  // 如果超时，则返回连接超时错误
  if (ret == 0) { return Error::ConnectionTimeout; }

  // 如果有可读或可写事件发生
  if (ret > 0 && (FD_ISSET(sock, &fdsr) || FD_ISSET(sock, &fdsw))) {
    // 获取套接字错误信息
    int error = 0;
    socklen_t len = sizeof(error);
    auto res = getsockopt(sock, SOL_SOCKET, SO_ERROR,
                          reinterpret_cast<char *>(&error), &len);
    // 判断是否成功连接
    auto successful = res >= 0 && !error;
    return successful ? Error::Success : Error::Connection;
  }
  // 返回连接错误
  return Error::Connection;
#endif
}

// 检查套接字是否存活
inline bool is_socket_alive(socket_t sock) {
  const auto val = detail::select_read(sock, 0, 0);
  if (val == 0) {
    return true;
  } else if (val < 0 && errno == EBADF) {
    return false;
  }
  char buf[1];
  // 通过 peek 操作检查套接字是否存活
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
// 私有成员变量，用于存储套接字、读写超时时间、读取缓冲区等信息
private:
  socket_t sock_;
  time_t read_timeout_sec_;
  time_t read_timeout_usec_;
  time_t write_timeout_sec_;
  time_t write_timeout_usec_;

  std::vector<char> read_buff_; // 用于存储读取的数据
  size_t read_buff_off_ = 0; // 读取缓冲区的偏移量
  size_t read_buff_content_size_ = 0; // 读取缓冲区中的数据大小

  static const size_t read_buff_size_ = 1024 * 4; // 读取缓冲区的大小
};

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 继承自 Stream 类，用于处理 SSL 套接字的读写操作
class SSLSocketStream : public Stream {
public:
  SSLSocketStream(socket_t sock, SSL *ssl, time_t read_timeout_sec,
                  time_t read_timeout_usec, time_t write_timeout_sec,
                  time_t write_timeout_usec);
  ~SSLSocketStream() override;

  bool is_readable() const override; // 判断 SSL 套接字是否可读
  bool is_writable() const override; // 判断 SSL 套接字是否可写
  ssize_t read(char *ptr, size_t size) override; // 从 SSL 套接字读取数据
  ssize_t write(const char *ptr, size_t size) override; // 向 SSL 套接字写入数据
  void get_remote_ip_and_port(std::string &ip, int &port) const override; // 获取远程 IP 和端口
  void get_local_ip_and_port(std::string &ip, int &port) const override; // 获取本地 IP 和端口
  socket_t socket() const override; // 获取套接字

private:
  socket_t sock_; // SSL 套接字
  SSL *ssl_; // SSL 对象
  time_t read_timeout_sec_; // 读取超时时间（秒）
  time_t read_timeout_usec_; // 读取超时时间（微秒）
  time_t write_timeout_sec_; // 写入超时时间（秒）
  time_t write_timeout_usec_; // 写入超时时间（微秒）
};
#endif

// 判断套接字是否保持连接
inline bool keep_alive(socket_t sock, time_t keep_alive_timeout_sec) {
  using namespace std::chrono;
  auto start = steady_clock::now(); // 记录开始时间
  while (true) {
    auto val = select_read(sock, 0, 10000); // 使用 select 函数检查套接字是否可读
    if (val < 0) {
      return false; // 如果出错，则返回 false
    } else if (val == 0) {
      auto current = steady_clock::now(); // 获取当前时间
      auto duration = duration_cast<milliseconds>(current - start); // 计算经过的时间
      auto timeout = keep_alive_timeout_sec * 1000; // 将超时时间转换为毫秒
      if (duration.count() > timeout) { return false; } // 如果超时，则返回 false
      std::this_thread::sleep_for(std::chrono::milliseconds(1)); // 等待 1 毫秒
    } else {
      return true; // 如果套接字可读，则返回 true
    }
  }
}

// 模板函数，用于判断套接字是否保持连接
template <typename T>
inline bool
// 处理服务器套接字的核心功能，接收参数为服务器套接字、套接字、最大保持连接次数、保持连接超时时间、回调函数
void process_server_socket_core(const std::atomic<socket_t> &svr_sock, socket_t sock,
                           size_t keep_alive_max_count,
                           time_t keep_alive_timeout_sec, T callback) {
  // 断言保持连接最大次数大于0
  assert(keep_alive_max_count > 0);
  // 初始化返回值为false
  auto ret = false;
  // 初始化保持连接次数
  auto count = keep_alive_max_count;
  // 当服务器套接字不为无效、保持连接次数大于0、保持连接超时时执行循环
  while (svr_sock != INVALID_SOCKET && count > 0 &&
         keep_alive(sock, keep_alive_timeout_sec)) {
    // 判断是否是最后一次保持连接
    auto close_connection = count == 1;
    // 初始化连接是否关闭的标志
    auto connection_closed = false;
    // 调用回调函数，获取返回值
    ret = callback(close_connection, connection_closed);
    // 如果返回值为false或者连接已关闭，则跳出循环
    if (!ret || connection_closed) { break; }
    // 保持连接次数减一
    count--;
  }
  // 返回结果
  return ret;
}

// 处理服务器套接字的功能，接收参数为服务器套接字、套接字、最大保持连接次数、保持连接超时时间、读取超时时间、写入超时时间、回调函数
template <typename T>
inline bool
process_server_socket(const std::atomic<socket_t> &svr_sock, socket_t sock,
                      size_t keep_alive_max_count,
                      time_t keep_alive_timeout_sec, time_t read_timeout_sec,
                      time_t read_timeout_usec, time_t write_timeout_sec,
                      time_t write_timeout_usec, T callback) {
  // 调用处理服务器套接字的核心功能，传入参数和lambda表达式
  return process_server_socket_core(
      svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
      [&](bool close_connection, bool &connection_closed) {
        // 创建套接字流对象
        SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                          write_timeout_sec, write_timeout_usec);
        // 调用回调函数，传入套接字流对象和其他参数
        return callback(strm, close_connection, connection_closed);
      });
}

// 处理客户端套接字的功能，接收参数为套接字、读取超时时间、写入超时时间、回调函数
inline bool process_client_socket(socket_t sock, time_t read_timeout_sec,
                                  time_t read_timeout_usec,
                                  time_t write_timeout_sec,
                                  time_t write_timeout_usec,
                                  std::function<bool(Stream &)> callback) {
  // 创建套接字流对象
  SocketStream strm(sock, read_timeout_sec, read_timeout_usec,
                    write_timeout_sec, write_timeout_usec);
  // 调用回调函数，传入套接字流对象
  return callback(strm);
}

// 关闭套接字的功能，接收参数为套接字
inline int shutdown_socket(socket_t sock) {
  // 根据操作系统类型选择关闭套接字的方式
#ifdef _WIN32
  return shutdown(sock, SD_BOTH);
#else
  return shutdown(sock, SHUT_RDWR);
#endif
}
  // 创建一个套接字，根据传入的参数进行绑定或连接操作
  template <typename BindOrConnect>
  socket_t create_socket(const std::string &host, const std::string &ip, int port,
                         int address_family, int socket_flags, bool tcp_nodelay,
                         SocketOptions socket_options,
                         BindOrConnect bind_or_connect) {
    // 获取地址信息
    const char *node = nullptr;
    struct addrinfo hints;
    struct addrinfo *result;

    // 清空 hints 结构体，并设置套接字类型为 SOCK_STREAM
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = 0;

    // 如果传入的 IP 不为空
    if (!ip.empty()) {
      node = ip.c_str();
      // 请求 getaddrinfo 将 IP 转换为地址
      hints.ai_family = AF_UNSPEC;
      hints.ai_flags = AI_NUMERICHOST;
    } else {
      // 如果传入的 host 不为空
      if (!host.empty()) { node = host.c_str(); }
      hints.ai_family = address_family;
      hints.ai_flags = socket_flags;
    }

    // 如果不是在 Windows 平台下
    #ifndef _WIN32
    if (hints.ai_family == AF_UNIX) {
      // 如果地址长度超过了 sockaddr_un::sun_path 的长度，则返回无效套接字
      const auto addrlen = host.length();
      if (addrlen > sizeof(sockaddr_un::sun_path)) return INVALID_SOCKET;

      // 创建一个 AF_UNIX 类型的套接字
      auto sock = socket(hints.ai_family, hints.ai_socktype, hints.ai_protocol);
      if (sock != INVALID_SOCKET) {
        sockaddr_un addr{};
        addr.sun_family = AF_UNIX;
        std::copy(host.begin(), host.end(), addr.sun_path);

        hints.ai_addr = reinterpret_cast<sockaddr *>(&addr);
        hints.ai_addrlen = static_cast<socklen_t>(
            sizeof(addr) - sizeof(addr.sun_path) + addrlen);

        // 设置套接字描述符的 close-on-exec 标志
        fcntl(sock, F_SETFD, FD_CLOEXEC);
        if (socket_options) { socket_options(sock); }

        // 如果绑定或连接失败，则关闭套接字并返回无效套接字
        if (!bind_or_connect(sock, hints)) {
          close_socket(sock);
          sock = INVALID_SOCKET;
        }
      }
      return sock;
    }
    #endif

    // 将端口号转换为字符串
    auto service = std::to_string(port);

    // 获取地址信息
    if (getaddrinfo(node, service.c_str(), &hints, &result)) {
      #if defined __linux__ && !defined __ANDROID__
      res_init();
      #endif
      return INVALID_SOCKET;
    }

    // 遍历地址信息链表
    for (auto rp = result; rp; rp = rp->ai_next) {
      // 创建一个套接字
      #ifdef _WIN32
      // 在 Windows 平台下创建套接字
      // ...
      #else
      // 在非 Windows 平台下创建套接字
      // ...
      #endif
    }
  }
  ```
    // 使用WSASocketW函数创建一个套接字，指定地址族、套接字类型和协议，同时设置一些标志位
    auto sock =
        WSASocketW(rp->ai_family, rp->ai_socktype, rp->ai_protocol, nullptr, 0,
                   WSA_FLAG_NO_HANDLE_INHERIT | WSA_FLAG_OVERLAPPED);
    /**
     * 由于WSA_FLAG_NO_HANDLE_INHERIT仅在Windows 7 SP1及以上版本支持，因此在旧版本的Windows系统上套接字创建会失败。
     * 在这种情况下，让我们尝试以旧的方式创建套接字。
     *
     * 参考链接：
     * https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasocketa
     *
     * WSA_FLAG_NO_HANDLE_INHERIT:
     * 该标志位在Windows 7 SP1、Windows Server 2008 R2 SP1及更高版本支持
     *
     */
    // 如果套接字创建失败，尝试以旧的方式创建套接字
    if (sock == INVALID_SOCKET) {
      sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    }
#else
    // 如果不是 Windows 平台，执行以下代码
    auto sock = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
#endif
    // 如果创建套接字失败，继续循环
    if (sock == INVALID_SOCKET) { continue; }

#ifndef _WIN32
    // 如果不是 Windows 平台，设置套接字为关闭时自动关闭
    if (fcntl(sock, F_SETFD, FD_CLOEXEC) == -1) {
      close_socket(sock);
      continue;
    }
#endif

    // 如果需要启用 TCP 禁用算法，设置套接字选项
    if (tcp_nodelay) {
      int yes = 1;
      setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, reinterpret_cast<char *>(&yes),
                 sizeof(yes));
    }

    // 如果有套接字选项，执行套接字选项函数
    if (socket_options) { socket_options(sock); }

    // 如果地址族是 IPv6，设置套接字选项
    if (rp->ai_family == AF_INET6) {
      int no = 0;
      setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY, reinterpret_cast<char *>(&no),
                 sizeof(no));
    }

    // 绑定或连接套接字
    if (bind_or_connect(sock, *rp)) {
      freeaddrinfo(result);
      return sock;
    }

    // 关闭套接字
    close_socket(sock);
  }

  freeaddrinfo(result);
  return INVALID_SOCKET;
}

// 设置套接字为非阻塞模式
inline void set_nonblocking(socket_t sock, bool nonblocking) {
#ifdef _WIN32
  auto flags = nonblocking ? 1UL : 0UL;
  ioctlsocket(sock, FIONBIO, &flags);
#else
  auto flags = fcntl(sock, F_GETFL, 0);
  fcntl(sock, F_SETFL,
        nonblocking ? (flags | O_NONBLOCK) : (flags & (~O_NONBLOCK)));
#endif
}

// 判断是否是连接错误
inline bool is_connection_error() {
#ifdef _WIN32
  return WSAGetLastError() != WSAEWOULDBLOCK;
#else
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

  freeaddrinfo(result);
  return ret;
}

// 如果不是 Windows、Android、AIX 平台，定义 USE_IF2IP
#if !defined _WIN32 && !defined ANDROID && !defined _AIX
#define USE_IF2IP
#endif

#ifdef USE_IF2IP
// 根据地址族和接口名获取对应的 IP 地址
inline std::string if2ip(int address_family, const std::string &ifn) {
  // 获取系统接口地址信息
  struct ifaddrs *ifap;
  getifaddrs(&ifap);
  // 候选地址字符串
  std::string addr_candidate;
  // 遍历接口地址信息
  for (auto ifa = ifap; ifa; ifa = ifa->ifa_next) {
    // 判断接口地址是否存在，并且接口名匹配，并且地址族匹配
    if (ifa->ifa_addr && ifn == ifa->ifa_name &&
        (AF_UNSPEC == address_family ||
         ifa->ifa_addr->sa_family == address_family)) {
      // 如果是 IPv4 地址
      if (ifa->ifa_addr->sa_family == AF_INET) {
        // 转换地址结构体指针类型
        auto sa = reinterpret_cast<struct sockaddr_in *>(ifa->ifa_addr);
        char buf[INET_ADDRSTRLEN];
        // 将 IPv4 地址转换为字符串形式
        if (inet_ntop(AF_INET, &sa->sin_addr, buf, INET_ADDRSTRLEN)) {
          // 释放接口地址信息内存
          freeifaddrs(ifap);
          return std::string(buf, INET_ADDRSTRLEN);
        }
      } else if (ifa->ifa_addr->sa_family == AF_INET6) {
        // 转换地址结构体指针类型
        auto sa = reinterpret_cast<struct sockaddr_in6 *>(ifa->ifa_addr);
        // 检查是否为链路本地地址
        if (!IN6_IS_ADDR_LINKLOCAL(&sa->sin6_addr)) {
          char buf[INET6_ADDRSTRLEN] = {};
          // 将 IPv6 地址转换为字符串形式
          if (inet_ntop(AF_INET6, &sa->sin6_addr, buf, INET6_ADDRSTRLEN)) {
            // 判断是否为唯一本地地址
            auto s6_addr_head = sa->sin6_addr.s6_addr[0];
            if (s6_addr_head == 0xfc || s6_addr_head == 0xfd) {
              addr_candidate = std::string(buf, INET6_ADDRSTRLEN);
            } else {
              // 释放接口地址信息内存
              freeifaddrs(ifap);
              return std::string(buf, INET6_ADDRSTRLEN);
            }
          }
        }
      }
    }
  }
  // 释放接口地址信息内存
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
    // 设置写超时时间（微秒）
    time_t write_timeout_usec, 
    // 网络接口名称
    const std::string &intf, 
    // 错误信息
    Error &error) {
  // 创建套接字
  auto sock = create_socket(
      // 主机名
      host, 
      // IP 地址
      ip, 
      // 端口号
      port, 
      // 地址族
      address_family, 
      // 0
      0, 
      // 是否启用 TCP 禁用算法
      tcp_nodelay, 
      // 移动套接字选项
      std::move(socket_options),
      // 回调函数
      [&](socket_t sock2, struct addrinfo &ai) -> bool {
        // 如果接口名称不为空
        if (!intf.empty()) {
        #ifdef USE_IF2IP
          // 如果定义了 USE_IF2IP，则获取接口对应的 IP 地址
          auto ip_from_if = if2ip(address_family, intf);
          // 如果获取的 IP 地址为空，则使用接口名称作为 IP 地址
          if (ip_from_if.empty()) { ip_from_if = intf; }
          // 绑定获取到的 IP 地址到套接字
          if (!bind_ip_address(sock2, ip_from_if.c_str())) {
            // 如果绑定失败，则设置错误类型为 BindIPAddress，返回失败
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
          // 如果是连接错误，则设置错误类型为 Connection，返回失败
          if (is_connection_error()) {
            error = Error::Connection;
            return false;
          }
          // 否则等待套接字就绪，直到超时
          error = wait_until_socket_is_ready(sock2, connection_timeout_sec,
                                             connection_timeout_usec);
          // 如果等待失败，则返回失败
          if (error != Error::Success) { return false; }
        }

        // 设置套接字为阻塞模式
        set_nonblocking(sock2, false);

        {
        #ifdef _WIN32
          // 如果是 Windows 系统，则设置接收超时时间
          auto timeout = static_cast<uint32_t>(read_timeout_sec * 1000 +
                                               read_timeout_usec / 1000);
          setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO, (char *)&timeout,
                     sizeof(timeout));
        #else
          // 如果是其他系统，则设置 timeval 结构体来设置接收超时时间
          timeval tv;
          tv.tv_sec = static_cast<long>(read_timeout_sec);
          tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec);
          setsockopt(sock2, SOL_SOCKET, SO_RCVTIMEO, (char *)&tv, sizeof(tv));
        #endif
        }
        {
        #ifdef _WIN32
          // 如果是 Windows 系统，则设置发送超时时间
          auto timeout = static_cast<uint32_t>(write_timeout_sec * 1000 +
                                               write_timeout_usec / 1000);
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO, (char *)&timeout,
                     sizeof(timeout));
        #else
          // 如果是其他系统，则设置 timeval 结构体来设置发送超时时间
          timeval tv;
          tv.tv_sec = static_cast<long>(write_timeout_sec);
          tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec);
          setsockopt(sock2, SOL_SOCKET, SO_SNDTIMEO, (char *)&tv, sizeof(tv));
        #endif
        }
#endif
        }

        // 设置错误码为成功
        error = Error::Success;
        // 返回 true
        return true;
      });

  // 如果套接字不是无效的
  if (sock != INVALID_SOCKET) {
    // 设置错误码为成功
    error = Error::Success;
  } else {
    // 如果错误码为成功，则设置错误码为连接错误
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
    // 如果地址族不是 AF_INET 或 AF_INET6，则返回 false
    return false;
  }

  // 定义存储 IP 地址的数组
  std::array<char, NI_MAXHOST> ipstr{};
  // 获取 IP 地址
  if (getnameinfo(reinterpret_cast<const struct sockaddr *>(&addr), addr_len,
                  ipstr.data(), static_cast<socklen_t>(ipstr.size()), nullptr,
                  0, NI_NUMERICHOST)) {
    // 如果获取失败，则返回 false
    return false;
  }

  // 将 IP 地址存储到 ip 变量中
  ip = ipstr.data();
  // 返回 true
  return true;
}

// 获取本地 IP 和端口
inline void get_local_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 定义存储地址的结构体
  struct sockaddr_storage addr;
  // 获取地址的长度
  socklen_t addr_len = sizeof(addr);
  // 如果获取套接字的本地地址成功
  if (!getsockname(sock, reinterpret_cast<struct sockaddr *>(&addr),
                   &addr_len)) {
    // 获取 IP 和端口
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 获取远程 IP 和端口
inline void get_remote_ip_and_port(socket_t sock, std::string &ip, int &port) {
  // 定义存储地址的结构体
  struct sockaddr_storage addr;
  // 获取地址的长度
  socklen_t addr_len = sizeof(addr);

  // 如果获取套接字的远程地址成功
  if (!getpeername(sock, reinterpret_cast<struct sockaddr *>(&addr),
                   &addr_len)) {
    // 如果地址族是 AF_UNIX
#ifndef _WIN32
    if (addr.ss_family == AF_UNIX) {
      // 如果是 Linux 系统
#if defined(__linux__)
      struct ucred ucred;
      socklen_t len = sizeof(ucred);
      // 获取对等进程的 PID
      if (getsockopt(sock, SOL_SOCKET, SO_PEERCRED, &ucred, &len) == 0) {
        port = ucred.pid;
      }
      // 如果是苹果系统
#elif defined(SOL_LOCAL) && defined(SO_PEERPID) // __APPLE__
      pid_t pid;
      socklen_t len = sizeof(pid);
      // 获取对等进程的 PID
      if (getsockopt(sock, SOL_LOCAL, SO_PEERPID, &pid, &len) == 0) {
        port = pid;
      }
#endif
      // 返回
      return;
    # 代码块结束
#endif
    // 结束预处理指令，用于条件编译
    get_ip_and_port(addr, addr_len, ip, port);
  }
}

// 递归计算字符串哈希值的核心函数
inline constexpr unsigned int str2tag_core(const char *s, size_t l,
                                           unsigned int h) {
  // 如果字符串长度为0，返回当前哈希值
  return (l == 0)
             ? h
             // 递归计算哈希值
             : str2tag_core(
                   s + 1, l - 1,
                   // 通过位运算和异或操作计算哈希值
                   (((std::numeric_limits<unsigned int>::max)() >> 6) &
                    h * 33) ^
                       static_cast<unsigned char>(*s));
}

// 对外接口，将字符串转换为哈希值
inline unsigned int str2tag(const std::string &s) {
  // 调用核心函数计算哈希值
  return str2tag_core(s.data(), s.size(), 0);
}

// 自定义字面量运算符命名空间
namespace udl {

// 自定义字面量运算符，将字符串转换为哈希值
inline constexpr unsigned int operator"" _t(const char *s, size_t l) {
  // 调用核心函数计算哈希值
  return str2tag_core(s, l, 0);
}

} // namespace udl

// 返回指向常量字符的指针
inline const char *
// 根据文件路径和用户数据的映射，查找并返回文件的内容类型
find_content_type(const std::string &path,
                  const std::map<std::string, std::string> &user_data) {
  // 获取文件扩展名
  auto ext = file_extension(path);

  // 在用户数据中查找扩展名对应的内容类型
  auto it = user_data.find(ext);
  if (it != user_data.end()) { return it->second.c_str(); }

  // 使用用户定义字面量转换运算符
  using udl::operator""_t;

  // 根据扩展名进行不同的处理
  switch (str2tag(ext)) {
  default: return nullptr;
  case "css"_t: return "text/css";
  case "csv"_t: return "text/csv";
  case "htm"_t:
  case "html"_t: return "text/html";
  case "js"_t:
  case "mjs"_t: return "text/javascript";
  case "txt"_t: return "text/plain";
  case "vtt"_t: return "text/vtt";

  // 图像类型
  case "apng"_t: return "image/apng";
  case "avif"_t: return "image/avif";
  case "bmp"_t: return "image/bmp";
  case "gif"_t: return "image/gif";
  case "png"_t: return "image/png";
  case "svg"_t: return "image/svg+xml";
  case "webp"_t: return "image/webp";
  case "ico"_t: return "image/x-icon";
  case "tif"_t: return "image/tiff";
  case "tiff"_t: return "image/tiff";
  case "jpg"_t:
  case "jpeg"_t: return "image/jpeg";

  // 视频类型
  case "mp4"_t: return "video/mp4";
  case "mpeg"_t: return "video/mpeg";
  case "webm"_t: return "video/webm";

  // 音频类型
  case "mp3"_t: return "audio/mp3";
  case "mpga"_t: return "audio/mpeg";
  case "weba"_t: return "audio/webm";
  case "wav"_t: return "audio/wave";

  // 字体类型
  case "otf"_t: return "font/otf";
  case "ttf"_t: return "font/ttf";
  case "woff"_t: return "font/woff";
  case "woff2"_t: return "font/woff2";

  // 压缩文件类型
  case "7z"_t: return "application/x-7z-compressed";
  case "atom"_t: return "application/atom+xml";
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
}

}
// 检查给定的内容类型是否可以进行压缩
inline bool can_compress_content_type(const std::string &content_type) {
  using udl::operator""_t;

  // 将内容类型转换为标签
  auto tag = str2tag(content_type);

  // 根据标签进行判断
  switch (tag) {
  case "image/svg+xml"_t:
  case "application/javascript"_t:
  case "application/json"_t:
  case "application/xml"_t:
  case "application/protobuf"_t:
  case "application/xhtml+xml"_t: return true;

  // 默认情况下，检查内容类型是否以"text/"开头，并且不是"text/event-stream"
  default:
    return !content_type.rfind("text/", 0) && tag != "text/event-stream"_t;
  }
}

// 获取请求和响应的编码类型
inline EncodingType encoding_type(const Request &req, const Response &res) {
  // 检查响应的内容类型是否可以进行压缩
  auto ret =
      detail::can_compress_content_type(res.get_header_value("Content-Type"));
  if (!ret) { return EncodingType::None; }

  // 获取请求头中的"Accept-Encoding"字段
  const auto &s = req.get_header_value("Accept-Encoding");
  (void)(s);

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
  // 如果支持Brotli压缩，检查请求头中是否包含"br"
  ret = s.find("br") != std::string::npos;
  if (ret) { return EncodingType::Brotli; }
#endif

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
  // 如果支持Gzip压缩，检查请求头中是否包含"gzip"
  ret = s.find("gzip") != std::string::npos;
  if (ret) { return EncodingType::Gzip; }
#endif

  // 默认情况下不进行压缩
  return EncodingType::None;
}

// 对数据进行压缩
inline bool nocompressor::compress(const char *data, size_t data_length,
                                   bool /*last*/, Callback callback) {
  // 如果数据长度为0，则直接返回true
  if (!data_length) { return true; }
  // 调用回调函数处理数据
  return callback(data, data_length);
}

#ifdef CPPHTTPLIB_ZLIB_SUPPORT
// 构造函数，初始化gzip压缩器
inline gzip_compressor::gzip_compressor() {
  std::memset(&strm_, 0, sizeof(strm_));
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  // 初始化deflate压缩器
  is_valid_ = deflateInit2(&strm_, Z_DEFAULT_COMPRESSION, Z_DEFLATED, 31, 8,
                           Z_DEFAULT_STRATEGY) == Z_OK;
}

// 析构函数，释放gzip压缩器
inline gzip_compressor::~gzip_compressor() { deflateEnd(&strm_); }

// 对数据进行压缩
inline bool gzip_compressor::compress(const char *data, size_t data_length,
                                      bool last, Callback callback) {
  assert(is_valid_);

  // 压缩数据
  do {
    // ...
    # 定义最大可用输入大小，使用 std::numeric_limits 获取 avail_in 类型的最大值
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    # 将数据长度与最大可用输入大小进行比较，取较小值作为可用输入大小
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    # 设置输入数据指针
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    # 更新数据长度和数据指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    # 根据是否为最后一次压缩和数据长度是否为0来确定压缩方式
    auto flush = (last && data_length == 0) ? Z_FINISH : Z_NO_FLUSH;
    int ret = Z_OK;

    # 定义缓冲区，用于存储压缩后的数据
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    do {
      # 设置输出缓冲区大小和指针
      strm_.avail_out = static_cast<uInt>(buff.size());
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      # 执行压缩操作
      ret = deflate(&strm_, flush);
      # 如果返回 Z_STREAM_ERROR 则表示压缩出错，返回 false
      if (ret == Z_STREAM_ERROR) { return false; }

      # 调用回调函数处理压缩后的数据，如果返回 false 则表示处理出错，返回 false
      if (!callback(buff.data(), buff.size() - strm_.avail_out)) {
        return false;
      }
    } while (strm_.avail_out == 0);

    # 断言判断压缩是否成功
    assert((flush == Z_FINISH && ret == Z_STREAM_END) ||
           (flush == Z_NO_FLUSH && ret == Z_OK));
    # 断言判断输入数据是否全部被使用
    assert(strm_.avail_in == 0);
  } while (data_length > 0);

  # 压缩完成，返回 true
  return true;
// 定义 gzip_decompressor 类的构造函数
inline gzip_decompressor::gzip_decompressor() {
  // 将 strm_ 的内存清零
  std::memset(&strm_, 0, sizeof(strm_));
  // 设置内存分配函数、释放内存函数和不透明数据的指针为 NULL
  strm_.zalloc = Z_NULL;
  strm_.zfree = Z_NULL;
  strm_.opaque = Z_NULL;

  // wbits 的值为 15，这是确保可以解码任何 gzip 流的最大可能值
  // 偏移量 32 指定应自动检测流类型，可以是 gzip 或 deflate
  is_valid_ = inflateInit2(&strm_, 32 + 15) == Z_OK;
}

// 定义 gzip_decompressor 类的析构函数
inline gzip_decompressor::~gzip_decompressor() { inflateEnd(&strm_); }

// 返回 is_valid_ 的值，表示解压器是否有效
inline bool gzip_decompressor::is_valid() const { return is_valid_; }

// 解压数据，并通过回调函数返回解压后的数据
inline bool gzip_decompressor::decompress(const char *data, size_t data_length,
                                          Callback callback) {
  // 断言解压器是否有效
  assert(is_valid_);

  int ret = Z_OK;

  do {
    // 定义最大可用输入大小
    constexpr size_t max_avail_in =
        (std::numeric_limits<decltype(strm_.avail_in)>::max)();

    // 将输入大小设置为最大可用输入大小和数据长度的较小值
    strm_.avail_in = static_cast<decltype(strm_.avail_in)>(
        (std::min)(data_length, max_avail_in));
    // 设置输入数据的指针
    strm_.next_in = const_cast<Bytef *>(reinterpret_cast<const Bytef *>(data));

    // 减去已经处理的输入数据长度，并更新数据指针
    data_length -= strm_.avail_in;
    data += strm_.avail_in;

    // 定义缓冲区，用于存储解压后的数据
    std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
    while (strm_.avail_in > 0) {
      // 设置输出缓冲区的大小
      strm_.avail_out = static_cast<uInt>(buff.size());
      // 设置输出缓冲区的指针
      strm_.next_out = reinterpret_cast<Bytef *>(buff.data());

      // 保存上一次处理前的可用输入大小
      auto prev_avail_in = strm_.avail_in;

      // 调用 inflate 函数进行解压
      ret = inflate(&strm_, Z_NO_FLUSH);

      // 如果没有处理输入数据，则返回 false
      if (prev_avail_in - strm_.avail_in == 0) { return false; }

      // 断言不会出现 Z_STREAM_ERROR
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

    // 如果返回值不是 Z_OK 或 Z_STREAM_END，则返回 false
    if (ret != Z_OK && ret != Z_STREAM_END) return false;

  } while (data_length > 0);

  return true;
}
#endif

#ifdef CPPHTTPLIB_BROTLI_SUPPORT
# 创建 brotli 压缩器的构造函数
inline brotli_compressor::brotli_compressor() {
  # 使用默认参数创建 Brotli 压缩器实例
  state_ = BrotliEncoderCreateInstance(nullptr, nullptr, nullptr);
}

# 创建 brotli 压缩器的析构函数
inline brotli_compressor::~brotli_compressor() {
  # 销毁 Brotli 压缩器实例
  BrotliEncoderDestroyInstance(state_);
}

# 对数据进行压缩的函数
inline bool brotli_compressor::compress(const char *data, size_t data_length,
                                        bool last, Callback callback) {
  # 创建用于存储压缩后数据的缓冲区
  std::array<uint8_t, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};

  # 设置压缩操作类型
  auto operation = last ? BROTLI_OPERATION_FINISH : BROTLI_OPERATION_PROCESS;
  auto available_in = data_length;
  auto next_in = reinterpret_cast<const uint8_t *>(data);

  # 循环执行压缩操作
  for (;;) {
    if (last) {
      # 如果是最后一次压缩，则检查压缩是否完成
      if (BrotliEncoderIsFinished(state_)) { break; }
    } else {
      # 如果不是最后一次压缩，则检查输入数据是否已经处理完
      if (!available_in) { break; }
    }

    auto available_out = buff.size();
    auto next_out = buff.data();

    # 执行压缩操作
    if (!BrotliEncoderCompressStream(state_, operation, &available_in, &next_in,
                                     &available_out, &next_out, nullptr)) {
      return false;
    }

    # 计算压缩后的数据大小，并调用回调函数传递压缩后的数据
    auto output_bytes = buff.size() - available_out;
    if (output_bytes) {
      callback(reinterpret_cast<const char *>(buff.data()), output_bytes);
    }
  }

  return true;
}

# 创建 brotli 解压缩器的构造函数
inline brotli_decompressor::brotli_decompressor() {
  # 创建 Brotli 解压缩器实例
  decoder_s = BrotliDecoderCreateInstance(0, 0, 0);
  # 初始化解压缩器的状态
  decoder_r = decoder_s ? BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT
                        : BROTLI_DECODER_RESULT_ERROR;
}

# 创建 brotli 解压缩器的析构函数
inline brotli_decompressor::~brotli_decompressor() {
  # 销毁 Brotli 解压缩器实例
  if (decoder_s) { BrotliDecoderDestroyInstance(decoder_s); }
}

# 检查解压缩器是否有效的函数
inline bool brotli_decompressor::is_valid() const { return decoder_s; }

# 对数据进行解压缩的函数
inline bool brotli_decompressor::decompress(const char *data,
                                            size_t data_length,
                                            Callback callback) {
  # 检查解压缩器的状态是否为成功或错误
  if (decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
      decoder_r == BROTLI_DECODER_RESULT_ERROR) {
  // 返回整数 0
  return 0;
}

// 将输入数据转换为无符号8位整数指针
const uint8_t *next_in = (const uint8_t *)data;
// 输入数据的长度
size_t avail_in = data_length;
// 输出数据的总长度
size_t total_out;

// 解码器结果初始化为需要更多输出
decoder_r = BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT;

// 创建一个大小为 CPPHTTPLIB_COMPRESSION_BUFSIZ 的字符数组
std::array<char, CPPHTTPLIB_COMPRESSION_BUFSIZ> buff{};
// 当解码器结果为需要更多输出时循环
while (decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT) {
  // 下一个输出位置为 buff 的数据指针
  char *next_out = buff.data();
  // 可用的输出空间大小为 buff 的大小
  size_t avail_out = buff.size();

  // 解压缩流，将输入数据解压缩到输出缓冲区
  decoder_r = BrotliDecoderDecompressStream(
      decoder_s, &avail_in, &next_in, &avail_out,
      reinterpret_cast<uint8_t **>(&next_out), &total_out);

  // 如果解码器结果为错误，则返回 false
  if (decoder_r == BROTLI_DECODER_RESULT_ERROR) { return false; }

  // 如果回调函数返回 false，则返回 false
  if (!callback(buff.data(), buff.size() - avail_out)) { return false; }
}

// 返回解码器结果是否为成功或需要更多输入
return decoder_r == BROTLI_DECODER_RESULT_SUCCESS ||
       decoder_r == BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT;
// 结束 C++ 的条件编译指令
}
#endif

// 判断给定的 headers 中是否包含指定的 key
inline bool has_header(const Headers &headers, const std::string &key) {
  return headers.find(key) != headers.end();
}

// 获取给定 key 对应的 header 值
inline const char *get_header_value(const Headers &headers,
                                    const std::string &key, size_t id,
                                    const char *def) {
  auto rng = headers.equal_range(key);
  auto it = rng.first;
  std::advance(it, static_cast<ssize_t>(id));
  if (it != rng.second) { return it->second.c_str(); }
  return def;
}

// 比较两个字符串（不区分大小写）
inline bool compare_case_ignore(const std::string &a, const std::string &b) {
  if (a.size() != b.size()) { return false; }
  for (size_t i = 0; i < b.size(); i++) {
    if (::tolower(a[i]) != ::tolower(b[i])) { return false; }
  }
  return true;
}

// 解析 header 内容
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

// 读取 headers 内容
inline bool read_headers(Stream &strm, Headers &headers) {
  const auto bufsiz = 2048;
  char buf[bufsiz];
  stream_line_reader line_reader(strm, buf, bufsiz);

  for (;;) {
    if (!line_reader.getline()) { return false; }

    // 检查行是否以 CRLF 结尾
    auto line_terminator_len = 2;
    if (line_reader.end_with_crlf()) {
      // 空行表示 headers 结束
      if (line_reader.size() == 2) { break; }
#ifdef CPPHTTPLIB_ALLOW_LF_AS_LINE_TERMINATOR
    } else {
      // 如果遇到空行，则表示头部结束
      // 检查当前行的长度是否为1，如果是则跳出循环
      if (line_reader.size() == 1) { break; }
      // 设置行终止符的长度为1
      line_terminator_len = 1;
    }
#else
    } else {
      continue; // Skip invalid line. // 如果不满足条件，则跳过当前循环，继续下一个循环
    }
#endif

    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; } // 如果读取的行长度超过最大长度限制，则返回false

    // Exclude line terminator
    auto end = line_reader.ptr() + line_reader.size() - line_terminator_len; // 排除行终止符

    parse_header(line_reader.ptr(), end, // 解析头部信息
                 [&](std::string &&key, std::string &&val) {
                   headers.emplace(std::move(key), std::move(val)); // 将解析出的键值对添加到头部信息中
                 });
  }

  return true; // 返回true表示成功读取内容

}

inline bool read_content_with_length(Stream &strm, uint64_t len,
                                     Progress progress,
                                     ContentReceiverWithProgress out) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ]; // 定义缓冲区

  uint64_t r = 0; // 初始化已读取的长度为0
  while (r < len) { // 当已读取的长度小于指定的长度时循环
    auto read_len = static_cast<size_t>(len - r); // 计算剩余需要读取的长度
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ)); // 从流中读取数据到缓冲区
    if (n <= 0) { return false; } // 如果读取失败，则返回false

    if (!out(buf, static_cast<size_t>(n), r, len)) { return false; } // 将读取的数据传递给回调函数，并检查是否成功
    r += static_cast<uint64_t>(n); // 更新已读取的长度

    if (progress) { // 如果存在进度回调函数
      if (!progress(r, len)) { return false; } // 调用进度回调函数，并检查是否成功
    }
  }

  return true; // 返回true表示成功读取内容
}

inline void skip_content_with_length(Stream &strm, uint64_t len) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ]; // 定义缓冲区
  uint64_t r = 0; // 初始化已读取的长度为0
  while (r < len) { // 当已读取的长度小于指定的长度时循环
    auto read_len = static_cast<size_t>(len - r); // 计算剩余需要读取的长度
    auto n = strm.read(buf, (std::min)(read_len, CPPHTTPLIB_RECV_BUFSIZ)); // 从流中读取数据到缓冲区
    if (n <= 0) { return; } // 如果读取失败，则直接返回

    r += static_cast<uint64_t>(n); // 更新已读取的长度
  }
}

inline bool read_content_without_length(Stream &strm,
                                        ContentReceiverWithProgress out) {
  char buf[CPPHTTPLIB_RECV_BUFSIZ]; // 定义缓冲区
  uint64_t r = 0; // 初始化已读取的长度为0
  for (;;) { // 无限循环
    auto n = strm.read(buf, CPPHTTPLIB_RECV_BUFSIZ); // 从流中读取数据到缓冲区
    if (n < 0) { // 如果读取失败
      return false; // 返回false
    } else if (n == 0) { // 如果读取到末尾
      return true; // 返回true
    }

    if (!out(buf, static_cast<size_t>(n), r, 0)) { return false; } // 将读取的数据传递给回调函数，并检查是否成功
    r += static_cast<uint64_t>(n); // 更新已读取的长度
  }

  return true; // 返回true表示成功读取内容
}

template <typename T>
// 以分块传输方式读取内容，并将内容传递给指定的接收器
inline bool read_content_chunked(Stream &strm, T &x,
                                 ContentReceiverWithProgress out) {
  // 定义缓冲区大小
  const auto bufsiz = 16;
  // 创建缓冲区
  char buf[bufsiz];

  // 创建流行读取器
  stream_line_reader line_reader(strm, buf, bufsiz);

  // 读取第一行数据
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

    // 如果分块长度为0，则结束循环
    if (chunk_len == 0) { break; }

    // 读取指定长度的内容并传递给接收器
    if (!read_content_with_length(strm, chunk_len, nullptr, out)) {
      return false;
    }

    // 读取下一行数据
    if (!line_reader.getline()) { return false; }

    // 检查分块结束符
    if (strcmp(line_reader.ptr(), "\r\n")) { return false; }

    // 读取下一行数据
    if (!line_reader.getline()) { return false; }
  }

  // 断言分块长度为0
  assert(chunk_len == 0);

  // Trailer
  // 读取 trailer 部分
  if (!line_reader.getline()) { return false; }

  // 解析 trailer 部分的头信息
  while (strcmp(line_reader.ptr(), "\r\n")) {
    if (line_reader.size() > CPPHTTPLIB_HEADER_MAX_LENGTH) { return false; }

    // 排除行终止符
    constexpr auto line_terminator_len = 2;
    auto end = line_reader.ptr() + line_reader.size() - line_terminator_len;

    // 解析头信息并添加到响应对象的头部
    parse_header(line_reader.ptr(), end,
                 [&](std::string &&key, std::string &&val) {
                   x.headers.emplace(std::move(key), std::move(val));
                 });

    // 读取下一行数据
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
    // 创建解压缩器
    std::unique_ptr<decompressor> decompressor;
    # 如果编码是 gzip 或者 deflate
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
      // 如果支持 ZLIB 压缩，则创建 gzip 解压缩器
      decompressor = detail::make_unique<gzip_decompressor>();
#else
      // 如果不支持，则返回 415 状态码并结束函数
      status = 415;
      return false;
#endif
    } else if (encoding.find("br") != std::string::npos) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
      // 如果支持 BROTLI 压缩，则创建 brotli 解压缩器
      decompressor = detail::make_unique<brotli_decompressor>();
#else
      // 如果不支持，则返回 415 状态码并结束函数
      status = 415;
      return false;
#endif
    }

    if (decompressor) {
      if (decompressor->is_valid()) {
        // 如果解压缩器有效，则创建 ContentReceiverWithProgress 对象，并调用回调函数
        ContentReceiverWithProgress out = [&](const char *buf, size_t n,
                                              uint64_t off, uint64_t len) {
          return decompressor->decompress(buf, n,
                                          [&](const char *buf2, size_t n2) {
                                            return receiver(buf2, n2, off, len);
                                          });
        };
        return callback(std::move(out));
      } else {
        // 如果解压缩器无效，则返回 500 状态码并结束函数
        status = 500;
        return false;
      }
    }
  }

  // 创建 ContentReceiverWithProgress 对象，并调用回调函数
  ContentReceiverWithProgress out = [&](const char *buf, size_t n, uint64_t off,
                                        uint64_t len) {
    return receiver(buf, n, off, len);
  };
  return callback(std::move(out));
}

template <typename T>
// 从流中读取内容，并根据参数进行处理，返回处理结果的布尔值
bool read_content(Stream &strm, T &x, size_t payload_max_length, int &status,
                  Progress progress, ContentReceiverWithProgress receiver,
                  bool decompress) {
  // 准备内容接收器，根据参数进行处理
  return prepare_content_receiver(
      x, status, std::move(receiver), decompress,
      // 匿名函数，根据不同情况进行内容读取和处理
      [&](const ContentReceiverWithProgress &out) {
        auto ret = true;
        auto exceed_payload_max_length = false;

        // 如果使用分块传输编码，则调用相应的函数进行内容读取
        if (is_chunked_transfer_encoding(x.headers)) {
          ret = read_content_chunked(strm, x, out);
        } 
        // 如果没有指定内容长度，则调用相应的函数进行内容读取
        else if (!has_header(x.headers, "Content-Length")) {
          ret = read_content_without_length(strm, out);
        } 
        // 否则根据内容长度进行处理
        else {
          auto len = get_header_value<uint64_t>(x.headers, "Content-Length");
          if (len > payload_max_length) {
            exceed_payload_max_length = true;
            skip_content_with_length(strm, len);
            ret = false;
          } else if (len > 0) {
            ret = read_content_with_length(strm, len, std::move(progress), out);
          }
        }

        // 根据处理结果设置状态码
        if (!ret) { status = exceed_payload_max_length ? 413 : 400; }
        return ret;
      });
} // namespace detail

// 写入头部信息到流中，并返回写入的字节数
inline ssize_t write_headers(Stream &strm, const Headers &headers) {
  ssize_t write_len = 0;
  // 遍历头部信息，将每一项格式化后写入流中
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

// 将数据写入流中，返回写入是否成功的布尔值
inline bool write_data(Stream &strm, const char *d, size_t l) {
  size_t offset = 0;
  // 循环写入数据，直到全部写入完成
  while (offset < l) {
    auto length = strm.write(d + offset, l - offset);
    if (length < 0) { return false; }
    offset += static_cast<size_t>(length);
  }
  return true;
}

template <typename T>
// 写入内容到流中，使用提供的内容提供者和偏移量、长度等参数
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length, T is_shutting_down,
                          Error &error) {
  // 计算结束偏移量
  size_t end_offset = offset + length;
  // 初始化标志位
  auto ok = true;
  // 创建数据接收器
  DataSink data_sink;

  // 数据接收器的写入操作
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果标志位为真
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

  // 循环直到写入完成或者关闭
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

  // 写入成功，返回成功
  error = Error::Success;
  return true;
}

// 重载函数，简化调用
template <typename T>
inline bool write_content(Stream &strm, const ContentProvider &content_provider,
                          size_t offset, size_t length,
                          const T &is_shutting_down) {
  auto error = Error::Success;
  return write_content(strm, content_provider, offset, length, is_shutting_down,
                       error);
}

// 重载函数，处理不带长度的写入
template <typename T>
inline bool
write_content_without_length(Stream &strm,
                             const ContentProvider &content_provider,
                             const T &is_shutting_down) {
  // 初始化偏移量和标志位
  size_t offset = 0;
  auto data_available = true;
  auto ok = true;
  // 创建数据接收器
  DataSink data_sink;

  // 数据接收器的写入操作
  data_sink.write = [&](const char *d, size_t l) -> bool {
    // 如果标志位为真
    if (ok) {
      offset += l;
      // 如果流不可写或者写入失败
      if (!strm.is_writable() || !write_data(strm, d, l)) { ok = false; }
    }
    return ok;
  };

  // 数据接收器的完成操作
  data_sink.done = [&](void) { data_available = false; };

  // 循环直到数据不可用或者关闭
  while (data_available && !is_shutting_down()) {
    // 如果流不可写
    if (!strm.is_writable()) {
      return false;
    } else if (!content_provider(offset, 0, data_sink)) {
      return false;
    }
  }
}
    } else if (!ok) {  # 如果条件不满足，则执行下面的语句
      return false;  # 返回 false
    }  # 结束 if-else 语句
  }  # 结束循环
  return true;  # 如果循环正常结束，则返回 true
  // 写入内容的块，使用数据提供者提供的数据，以分块传输编码的方式写入到流中
  template <typename T, typename U>
  inline bool
  write_content_chunked(Stream &strm, const ContentProvider &content_provider,
                        const T &is_shutting_down, U &compressor, Error &error) {
    // 偏移量
    size_t offset = 0;
    // 数据是否可用
    auto data_available = true;
    // 操作是否成功
    auto ok = true;
    // 数据接收器
    DataSink data_sink;

    // 写入数据的 Lambda 函数
    data_sink.write = [&](const char *d, size_t l) -> bool {
      if (ok) {
        // 检查数据是否可用
        data_available = l > 0;
        // 更新偏移量
        offset += l;

        // 压缩数据
        std::string payload;
        if (compressor.compress(d, l, false,
                                [&](const char *data, size_t data_len) {
                                  payload.append(data, data_len);
                                  return true;
                                })) {
          if (!payload.empty()) {
            // 发送每个数据块的分块响应头和尾
            auto chunk =
                from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
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

    // 完成尾部的 Lambda 函数
    auto done_with_trailer = [&](const Headers *trailer) {
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

      if (!payload.empty()) {
        // 发送每个数据块的分块响应头和尾
        auto chunk = from_i_to_hex(payload.size()) + "\r\n" + payload + "\r\n";
        if (!strm.is_writable() ||
            !write_data(strm, chunk.data(), chunk.size())) {
          ok = false;
          return;
        }
      }

      static const std::string done_marker("0\r\n");
    };
  }
    // 如果写入数据失败，则将 ok 置为 false
    if (!write_data(strm, done_marker.data(), done_marker.size())) {
      ok = false;
    }

    // Trailer
    // 如果存在 trailer，则将 trailer 中的每个键值对拼接成字段行，写入数据流
    if (trailer) {
      for (const auto &kv : *trailer) {
        std::string field_line = kv.first + ": " + kv.second + "\r\n";
        if (!write_data(strm, field_line.data(), field_line.size())) {
          ok = false;
        }
      }
    }

    // 写入 CRLF（回车换行符）到数据流
    static const std::string crlf("\r\n");
    if (!write_data(strm, crlf.data(), crlf.size())) { ok = false; }
  };

  // 数据流处理完成时的回调函数，调用 done_with_trailer 函数
  data_sink.done = [&](void) { done_with_trailer(nullptr); };

  // 数据流处理完成时的回调函数，调用 done_with_trailer 函数，并传入 trailer
  data_sink.done_with_trailer = [&](const Headers &trailer) {
    done_with_trailer(&trailer);
  };

  // 当数据可用且未关闭时，循环执行以下操作
  while (data_available && !is_shutting_down()) {
    // 如果数据流不可写，则将 error 置为 Error::Write，并返回 false
    if (!strm.is_writable()) {
      error = Error::Write;
      return false;
    } 
    // 如果内容提供者返回 false，则将 error 置为 Error::Canceled，并返回 false
    else if (!content_provider(offset, 0, data_sink)) {
      error = Error::Canceled;
      return false;
    } 
    // 如果 ok 为 false，则将 error 置为 Error::Write，并返回 false
    else if (!ok) {
      error = Error::Write;
      return false;
    }
  }

  // 循环结束后，将 error 置为 Error::Success，并返回 true
  error = Error::Success;
  return true;
// 写入分块内容到流中，使用压缩器压缩数据
template <typename T, typename U>
inline bool write_content_chunked(Stream &strm,
                                  const ContentProvider &content_provider,
                                  const T &is_shutting_down, U &compressor) {
  auto error = Error::Success;
  return write_content_chunked(strm, content_provider, is_shutting_down,
                               compressor, error);
}

// 重定向请求到指定路径和位置
template <typename T>
inline bool redirect(T &cli, Request &req, Response &res,
                     const std::string &path, const std::string &location,
                     Error &error) {
  // 创建新的请求对象
  Request new_req = req;
  new_req.path = path;
  new_req.redirect_count_ -= 1;

  // 如果响应状态为303且请求方法不是GET或HEAD，则修改请求方法为GET，并清空请求体和请求头
  if (res.status == 303 && (req.method != "GET" && req.method != "HEAD")) {
    new_req.method = "GET";
    new_req.body.clear();
    new_req.headers.clear();
  }

  // 创建新的响应对象
  Response new_res;

  // 发送新的请求，并获取返回值
  auto ret = cli.send(new_req, new_res, error);
  if (ret) {
    req = new_req;
    res = new_res;
    res.location = location;
  }
  return ret;
}

// 将参数转换为查询字符串
inline std::string params_to_query_str(const Params &params) {
  std::string query;

  // 遍历参数，拼接成查询字符串
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
  // 使用split函数分割查询字符串，解析出键值对并插入参数中
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

    if (!key.empty()) {
      params.emplace(decode_url(key, true), decode_url(val, true));
    }
  });
}
// 解析 multipart/form-data 内容类型中的 boundary 字段
inline bool parse_multipart_boundary(const std::string &content_type,
                                     std::string &boundary) {
  // 查找 boundary= 关键字在 content_type 中的位置
  auto boundary_keyword = "boundary=";
  auto pos = content_type.find(boundary_keyword);
  // 如果找不到 boundary= 关键字，则返回 false
  if (pos == std::string::npos) { return false; }
  // 查找 boundary= 后的分号位置
  auto end = content_type.find(';', pos);
  auto beg = pos + strlen(boundary_keyword);
  // 提取出 boundary 的值
  boundary = content_type.substr(beg, end - beg);
  // 如果 boundary 的长度大于等于2且以双引号开头和结尾，则去掉双引号
  if (boundary.length() >= 2 && boundary.front() == '"' &&
      boundary.back() == '"') {
    boundary = boundary.substr(1, boundary.size() - 2);
  }
  // 返回 boundary 是否为空
  return !boundary.empty();
}

#ifdef CPPHTTPLIB_NO_EXCEPTIONS
// 解析 Range 头部的内容
inline bool parse_range_header(const std::string &s, Ranges &ranges) {
#else
// 解析 Range 头部的内容，使用异常处理
inline bool parse_range_header(const std::string &s, Ranges &ranges) try {
#endif
  // 定义正则表达式来匹配 bytes= 开头的 Range 头部
  static auto re_first_range = std::regex(R"(bytes=(\d*-\d*(?:,\s*\d*-\d*)*))");
  std::smatch m;
  // 如果匹配成功
  if (std::regex_match(s, m, re_first_range)) {
    auto pos = static_cast<size_t>(m.position(1));
    auto len = static_cast<size_t>(m.length(1));
    bool all_valid_ranges = true;
    // 分割多个 Range
    split(&s[pos], &s[pos + len], ',', [&](const char *b, const char *e) {
      if (!all_valid_ranges) return;
      // 定义正则表达式来匹配每个 Range
      static auto re_another_range = std::regex(R"(\s*(\d*)-(\d*))");
      std::cmatch cm;
      // 如果匹配成功
      if (std::regex_match(b, e, cm, re_another_range)) {
        ssize_t first = -1;
        if (!cm.str(1).empty()) {
          first = static_cast<ssize_t>(std::stoll(cm.str(1)));
        }

        ssize_t last = -1;
        if (!cm.str(2).empty()) {
          last = static_cast<ssize_t>(std::stoll(cm.str(2)));
        }

        // 如果 first 和 last 都存在且 first 大于 last，则标记为无效 Range
        if (first != -1 && last != -1 && first > last) {
          all_valid_ranges = false;
          return;
        }
        // 将有效的 Range 添加到 ranges 中
        ranges.emplace_back(std::make_pair(first, last));
      }
    });
    // 返回所有 Range 是否都有效
    return all_valid_ranges;
  }
  // 如果匹配失败，则返回 false
  return false;
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
}
#else
} catch (...) { return false; }
#endif

class MultipartFormDataParser {
// 默认构造函数
public:
  MultipartFormDataParser() = default;

  // 设置边界
  void set_boundary(std::string &&boundary) {
    boundary_ = boundary;
    dash_boundary_crlf_ = dash_ + boundary_ + crlf_;
    crlf_dash_boundary_ = crlf_ + dash_ + boundary_;
  }

  // 检查是否有效
  bool is_valid() const { return is_valid_; }

  // 解析多部分表单数据
  bool parse(const char *buf, size_t n, const ContentReceiver &content_callback,
             const MultipartContentHeader &header_callback) {

    // TODO: support 'filename*'
    // 正则表达式匹配 Content-Disposition
    static const std::regex re_content_disposition(
        R"~(^Content-Disposition:\s*form-data;\s*name="(.*?)"(?:;\s*filename="(.*?)")?(?:;\s*filename\*=\S+)?\s*$)~",
        std::regex_constants::icase);

    // 追加缓冲区内容
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

  // 不区分大小写比较字符串前缀
  bool start_with_case_ignore(const std::string &a,
                              const std::string &b) const {
    if (a.size() < b.size()) { return false; }
    for (size_t i = 0; i < b.size(); i++) {
      if (::tolower(a[i]) != ::tolower(b[i])) { return false; }
    }
    return true;
  }

  // 常量字符串
  const std::string dash_ = "--";
  const std::string crlf_ = "\r\n";
  const std::string dash_crlf_ = "--\r\n";
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

  // 缓冲区大小
  size_t buf_size() const { return buf_epos_ - buf_spos_; }

  // 缓冲区数据
  const char *buf_data() const { return &buf_[buf_spos_]; }

  // 缓冲区头部
  std::string buf_head(size_t l) const { return buf_.substr(buf_spos_, l); }

  // 缓冲区是否以指定字符串开头
  bool buf_start_with(const std::string &s) const {
  // 在缓冲区中查找以给定字符串开头的位置，并返回其索引
  return start_with(buf_, buf_spos_, buf_epos_, s);
}

// 在缓冲区中查找给定字符串的位置
size_t buf_find(const std::string &s) const {
  // 获取给定字符串的第一个字符
  auto c = s.front();

  // 从缓冲区的起始位置开始遍历
  size_t off = buf_spos_;
  while (off < buf_epos_) {
    // 记录当前位置
    auto pos = off;
    // 在缓冲区中查找给定字符的位置
    while (true) {
      // 如果已经到达缓冲区末尾，则返回缓冲区的大小
      if (pos == buf_epos_) { return buf_size(); }
      // 如果找到给定字符，则跳出循环
      if (buf_[pos] == c) { break; }
      pos++;
    }

    // 计算剩余的字符数
    auto remaining_size = buf_epos_ - pos;
    // 如果给定字符串的长度大于剩余字符数，则返回缓冲区的大小
    if (s.size() > remaining_size) { return buf_size(); }

    // 如果在当前位置开始的子串与给定字符串相同，则返回当前位置相对于缓冲区起始位置的偏移量
    if (start_with(buf_, pos, buf_epos_, s)) { return pos - buf_spos_; }

    // 更新遍历的起始位置
    off = pos + 1;
  }

  // 如果未找到给定字符串，则返回缓冲区的大小
  return buf_size();
}

// 向缓冲区追加数据
void buf_append(const char *data, size_t n) {
  // 计算剩余的字符数
  auto remaining_size = buf_size();
  // 如果缓冲区中有剩余字符且起始位置不为0，则将剩余字符向前移动
  if (remaining_size > 0 && buf_spos_ > 0) {
    for (size_t i = 0; i < remaining_size; i++) {
      buf_[i] = buf_[buf_spos_ + i];
    }
  }
  // 更新缓冲区的起始位置和结束位置
  buf_spos_ = 0;
  buf_epos_ = remaining_size;

  // 如果追加数据后的大小超过了缓冲区的容量，则扩展缓冲区的大小
  if (remaining_size + n > buf_.size()) { buf_.resize(remaining_size + n); }

  // 将数据追加到缓冲区末尾
  for (size_t i = 0; i < n; i++) {
    buf_[buf_epos_ + i] = data[i];
  }
  // 更新缓冲区的结束位置
  buf_epos_ += n;
}

// 从缓冲区中擦除指定大小的数据
void buf_erase(size_t size) { buf_spos_ += size; }

// 缓冲区
std::string buf_;
// 缓冲区的起始位置
size_t buf_spos_ = 0;
// 缓冲区的结束位置
size_t buf_epos_ = 0;
};

// 将字符数组转换为小写字符串
inline std::string to_lower(const char *beg, const char *end) {
  std::string out;
  auto it = beg;
  while (it != end) {
    out += static_cast<char>(::tolower(*it));
    it++;
  }
  return out;
}

// 生成一个用于multipart数据的边界字符串
inline std::string make_multipart_data_boundary() {
  static const char data[] =
      "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

  // 使用随机设备生成种子
  std::random_device seed_gen;

  // 请求128位的熵进行初始化
  std::seed_seq seed_sequence{seed_gen(), seed_gen(), seed_gen(), seed_gen()};
  std::mt19937 engine(seed_sequence);

  std::string result = "--cpp-httplib-multipart-data-";

  for (auto i = 0; i < 16; i++) {
    result += data[engine() % (sizeof(data) - 1)];
  }

  return result;
}

// 检查multipart边界字符串的有效性
inline bool is_multipart_boundary_chars_valid(const std::string &boundary) {
  auto valid = true;
  for (size_t i = 0; i < boundary.size(); i++) {
    auto c = boundary[i];
    if (!std::isalnum(c) && c != '-' && c != '_') {
      valid = false;
      break;
    }
  }
  return valid;
}

// 序列化multipart表单数据项的开始部分
template <typename T>
inline std::string
serialize_multipart_formdata_item_begin(const T &item,
                                        const std::string &boundary) {
  std::string body = "--" + boundary + "\r\n";
  body += "Content-Disposition: form-data; name=\"" + item.name + "\"";
  if (!item.filename.empty()) {
    body += "; filename=\"" + item.filename + "\"";
  }
  body += "\r\n";
  if (!item.content_type.empty()) {
    body += "Content-Type: " + item.content_type + "\r\n";
  }
  body += "\r\n";

  return body;
}

// 序列化multipart表单数据项的结束部分
inline std::string serialize_multipart_formdata_item_end() { return "\r\n"; }

// 完成multipart表单数据的序列化
inline std::string
serialize_multipart_formdata_finish(const std::string &boundary) {
  return "--" + boundary + "--\r\n";
}

inline std::string
// 返回带有指定边界的多部分表单数据的内容类型
serialize_multipart_formdata_get_content_type(const std::string &boundary) {
  return "multipart/form-data; boundary=" + boundary;
}

// 序列化多部分表单数据
inline std::string
serialize_multipart_formdata(const MultipartFormDataItems &items,
                             const std::string &boundary, bool finish = true) {
  std::string body;

  // 遍历多部分表单数据项，序列化每个项的开始部分和内容，拼接到body中
  for (const auto &item : items) {
    body += serialize_multipart_formdata_item_begin(item, boundary);
    body += item.content + serialize_multipart_formdata_item_end();
  }

  // 如果需要结束部分，拼接结束部分到body中
  if (finish) body += serialize_multipart_formdata_finish(boundary);

  return body;
}

// 获取范围偏移和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, size_t content_length,
                            size_t index) {
  auto r = req.ranges[index];

  // 如果范围未指定，则返回整个内容的偏移和长度
  if (r.first == -1 && r.second == -1) {
    return std::make_pair(0, content_length);
  }

  auto slen = static_cast<ssize_t>(content_length);

  // 如果起始偏移未指定，则计算起始偏移
  if (r.first == -1) {
    r.first = (std::max)(static_cast<ssize_t>(0), slen - r.second);
    r.second = slen - 1;
  }

  // 如果结束偏移未指定，则计算结束偏移
  if (r.second == -1) { r.second = slen - 1; }
  return std::make_pair(r.first, static_cast<size_t>(r.second - r.first) + 1);
}

// 生成内容范围头字段
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

// 处理多部分范围数据
template <typename SToken, typename CToken, typename Content>
bool process_multipart_ranges_data(const Request &req, Response &res,
                                   const std::string &boundary,
                                   const std::string &content_type,
                                   SToken stoken, CToken ctoken,
                                   Content content) {
  for (size_t i = 0; i < req.ranges.size(); i++) {
    ctoken("--");
    stoken(boundary);
    # 在输出中添加回车换行符
    ctoken("\r\n");
    # 如果内容类型不为空，则添加内容类型到输出中
    if (!content_type.empty()) {
      ctoken("Content-Type: ");
      stoken(content_type);
      ctoken("\r\n");
    }

    # 获取范围偏移量和长度
    auto offsets = get_range_offset_and_length(req, res.body.size(), i);
    auto offset = offsets.first;
    auto length = offsets.second;

    # 添加内容范围到输出中
    ctoken("Content-Range: ");
    stoken(make_content_range_header_field(offset, length, res.body.size()));
    ctoken("\r\n");
    ctoken("\r\n");
    # 如果内容偏移和长度不正确，则返回false
    if (!content(offset, length)) { return false; }
    # 在输出中添加回车换行符
    ctoken("\r\n");
  }

  # 在输出中添加分隔符
  ctoken("--");
  stoken(boundary);
  ctoken("--\r\n");

  # 返回true
  return true;
// 生成多部分范围数据，将其存储在data中，并返回处理结果
inline bool make_multipart_ranges_data(const Request &req, Response &res,
                                       const std::string &boundary,
                                       const std::string &content_type,
                                       std::string &data) {
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { data += token; },  // 处理token并添加到data中
      [&](const std::string &token) { data += token; },  // 处理token并添加到data中
      [&](size_t offset, size_t length) {
        if (offset < res.body.size()) {  // 如果偏移量小于响应体大小
          data += res.body.substr(offset, length);  // 将响应体中指定范围的数据添加到data中
          return true;
        }
        return false;
      });
}

// 获取多部分范围数据的长度
inline size_t
get_multipart_ranges_data_length(const Request &req, Response &res,
                                 const std::string &boundary,
                                 const std::string &content_type) {
  size_t data_length = 0;

  process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { data_length += token.size(); },  // 计算token的长度并添加到data_length中
      [&](const std::string &token) { data_length += token.size(); },  // 计算token的长度并添加到data_length中
      [&](size_t /*offset*/, size_t length) {
        data_length += length;  // 添加指定范围的长度到data_length中
        return true;
      });

  return data_length;  // 返回数据的长度
}

// 将多部分范围数据写入流中
template <typename T>
inline bool write_multipart_ranges_data(Stream &strm, const Request &req,
                                        Response &res,
                                        const std::string &boundary,
                                        const std::string &content_type,
                                        const T &is_shutting_down) {
  return process_multipart_ranges_data(
      req, res, boundary, content_type,
      [&](const std::string &token) { strm.write(token); },  // 将token写入流中
      [&](const std::string &token) { strm.write(token); },  // 将token写入流中
      [&](size_t offset, size_t length) {
        return write_content(strm, res.content_provider_, offset, length,  // 写入内容到流中
                             is_shutting_down);
      });
}
// 返回请求中指定范围的偏移量和长度
inline std::pair<size_t, size_t>
get_range_offset_and_length(const Request &req, const Response &res,
                            size_t index) {
  auto r = req.ranges[index];

  // 如果范围的结束位置为-1，将其设置为响应内容长度减1
  if (r.second == -1) {
    r.second = static_cast<ssize_t>(res.content_length_) - 1;
  }

  // 返回范围的起始位置和长度
  return std::make_pair(r.first, r.second - r.first + 1);
}

// 检查请求是否包含内容
inline bool expect_content(const Request &req) {
  // 如果请求方法为POST、PUT、PATCH、PRI或DELETE，则返回true
  if (req.method == "POST" || req.method == "PUT" || req.method == "PATCH" ||
      req.method == "PRI" || req.method == "DELETE") {
    return true;
  }
  // TODO: 检查Content-Length是否已设置
  return false;
}

// 检查字符串中是否包含回车换行符
inline bool has_crlf(const std::string &s) {
  auto p = s.c_str();
  while (*p) {
    // 如果字符串中包含回车或换行符，则返回true
    if (*p == '\r' || *p == '\n') { return true; }
    p++;
  }
  // 如果字符串中不包含回车或换行符，则返回false
  return false;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 计算字符串的消息摘要，使用指定的算法
inline std::string message_digest(const std::string &s, const EVP_MD *algo) {
  // 创建EVP_MD_CTX上下文对象
  auto context = std::unique_ptr<EVP_MD_CTX, decltype(&EVP_MD_CTX_free)>(
      EVP_MD_CTX_new(), EVP_MD_CTX_free);

  unsigned int hash_length = 0;
  unsigned char hash[EVP_MAX_MD_SIZE];

  // 初始化消息摘要上下文
  EVP_DigestInit_ex(context.get(), algo, nullptr);
  // 更新消息摘要上下文的数据
  EVP_DigestUpdate(context.get(), s.c_str(), s.size());
  // 完成消息摘要计算
  EVP_DigestFinal_ex(context.get(), hash, &hash_length);

  // 将消息摘要转换为十六进制字符串
  std::stringstream ss;
  for (auto i = 0u; i < hash_length; ++i) {
    ss << std::hex << std::setw(2) << std::setfill('0')
       << (unsigned int)hash[i];
  }

  // 返回消息摘要的十六进制字符串表示
  return ss.str();
}

// 计算字符串的MD5消息摘要
inline std::string MD5(const std::string &s) {
  return message_digest(s, EVP_md5());
}

// 计算字符串的SHA-256消息摘要
inline std::string SHA_256(const std::string &s) {
  return message_digest(s, EVP_sha256());
}

// 计算字符串的SHA-512消息摘要
inline std::string SHA_512(const std::string &s) {
  return message_digest(s, EVP_sha512());
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
#ifdef _WIN32
// 注意：此代码来源于以下stackoverflow帖子：
// https://stackoverflow.com/questions/9507184/can-openssl-on-windows-use-the-system-certificate-store
// 在 Windows 平台加载系统证书到指定的 X509_STORE 对象中
inline bool load_system_certs_on_windows(X509_STORE *store) {
  // 打开系统存储区域中的 ROOT 证书存储区
  auto hStore = CertOpenSystemStoreW((HCRYPTPROV_LEGACY)NULL, L"ROOT");
  // 如果打开失败，返回 false
  if (!hStore) { return false; }

  auto result = false;
  PCCERT_CONTEXT pContext = NULL;
  // 遍历证书存储区中的证书
  while ((pContext = CertEnumCertificatesInStore(hStore, pContext)) !=
         nullptr) {
    // 获取证书的编码数据
    auto encoded_cert =
        static_cast<const unsigned char *>(pContext->pbCertEncoded);
    // 将编码数据解析为 X509 结构
    auto x509 = d2i_X509(NULL, &encoded_cert, pContext->cbCertEncoded);
    // 如果解析成功，将证书添加到 X509_STORE 中，并释放 X509 结构
    if (x509) {
      X509_STORE_add_cert(store, x509);
      X509_free(x509);
      result = true;
    }
  }
  // 释放证书上下文
  CertFreeCertificateContext(pContext);
  // 关闭证书存储区
  CertCloseStore(hStore, 0);
  // 返回操作结果
  return result;
}
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#if TARGET_OS_OSX
// 定义 CFObjectPtr 类型别名
template <typename T>
using CFObjectPtr =
    std::unique_ptr<typename std::remove_pointer<T>::type, void (*)(CFTypeRef)>;

// CFObjectPtr 的删除器函数
inline void cf_object_ptr_deleter(CFTypeRef obj) {
  if (obj) { CFRelease(obj); }
}

// 从 macOS 密钥链中检索证书
inline bool retrieve_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 定义查询的键和值
  CFStringRef keys[] = {kSecClass, kSecMatchLimit, kSecReturnRef};
  CFTypeRef values[] = {kSecClassCertificate, kSecMatchLimitAll,
                        kCFBooleanTrue};
  // 创建查询字典
  CFObjectPtr<CFDictionaryRef> query(
      CFDictionaryCreate(nullptr, reinterpret_cast<const void **>(keys), values,
                         sizeof(keys) / sizeof(keys[0]),
                         &kCFTypeDictionaryKeyCallBacks,
                         &kCFTypeDictionaryValueCallBacks),
      cf_object_ptr_deleter);
  // 如果创建失败，返回 false
  if (!query) { return false; }
  // 从密钥链中匹配查询，获取证书数组
  CFTypeRef security_items = nullptr;
  if (SecItemCopyMatching(query.get(), &security_items) != errSecSuccess ||
      CFArrayGetTypeID() != CFGetTypeID(security_items)) {
    return false;
  }
  // 将获取的证书数组赋值给 certs，并返回 true
  certs.reset(reinterpret_cast<CFArrayRef>(security_items));
  return true;
}
// 从钥匙串中检索根证书，并将结果存储在 certs 中
inline bool retrieve_root_certs_from_keychain(CFObjectPtr<CFArrayRef> &certs) {
  // 初始化 root_security_items 为 nullptr
  CFArrayRef root_security_items = nullptr;
  // 如果 SecTrustCopyAnchorCertificates 函数调用失败，则返回 false
  if (SecTrustCopyAnchorCertificates(&root_security_items) != errSecSuccess) {
    return false;
  }

  // 重置 certs 指针，指向 root_security_items
  certs.reset(root_security_items);
  return true;
}

// 将证书添加到 X509 存储中
inline bool add_certs_to_x509_store(CFArrayRef certs, X509_STORE *store) {
  auto result = false;
  // 遍历 certs 中的证书
  for (int i = 0; i < CFArrayGetCount(certs); ++i) {
    // 将 CFArray 中的证书转换为 __SecCertificate 类型
    const auto cert = reinterpret_cast<const __SecCertificate *>(
        CFArrayGetValueAtIndex(certs, i));

    // 如果证书类型不是 SecCertificate，则继续下一次循环
    if (SecCertificateGetTypeID() != CFGetTypeID(cert)) { continue; }

    // 初始化 cert_data 为 nullptr
    CFDataRef cert_data = nullptr;
    // 如果 SecItemExport 函数调用失败，则继续下一次循环
    if (SecItemExport(cert, kSecFormatX509Cert, 0, nullptr, &cert_data) !=
        errSecSuccess) {
      continue;
    }

    // 使用 cert_data_ptr 指向 cert_data，并在作用域结束时自动释放 cert_data
    CFObjectPtr<CFDataRef> cert_data_ptr(cert_data, cf_object_ptr_deleter);

    // 获取证书的编码数据
    auto encoded_cert = static_cast<const unsigned char *>(
        CFDataGetBytePtr(cert_data_ptr.get()));

    // 将编码数据解析为 X509 结构
    auto x509 =
        d2i_X509(NULL, &encoded_cert, CFDataGetLength(cert_data_ptr.get()));

    // 如果成功解析为 X509 结构，则将其添加到存储中，并释放 x509
    if (x509) {
      X509_STORE_add_cert(store, x509);
      X509_free(x509);
      result = true;
    }
  }

  return result;
}

// 在 macOS 上加载系统证书到 X509 存储中
inline bool load_system_certs_on_macos(X509_STORE *store) {
  auto result = false;
  // 初始化 certs 为 nullptr
  CFObjectPtr<CFArrayRef> certs(nullptr, cf_object_ptr_deleter);
  // 如果成功从钥匙串中检索证书，并且 certs 不为空
  if (retrieve_certs_from_keychain(certs) && certs) {
    // 将 certs 中的证书添加到 X509 存储中
    result = add_certs_to_x509_store(certs.get(), store);
  }

  // 如果成功从钥匙串中检索根证书，并且 certs 不为空
  if (retrieve_root_certs_from_keychain(certs) && certs) {
    // 将 certs 中的证书添加到 X509 存储中，并将结果与之前的结果进行逻辑或操作
    result = add_certs_to_x509_store(certs.get(), store) || result;
  }

  return result;
}
#endif // TARGET_OS_OSX
#endif // _WIN32
#endif // CPPHTTPLIB_OPENSSL_SUPPORT

#ifdef _WIN32
// 在 Windows 上初始化 Winsock
class WSInit {
public:
  WSInit() {
    // 初始化 Winsock
    WSADATA wsaData;
    if (WSAStartup(0x0002, &wsaData) == 0) is_valid_ = true;
  }

  // 在作用域结束时自动清理 Winsock
  ~WSInit() {
    if (is_valid_) WSACleanup();
  }

  // 标记 Winsock 是否有效
  bool is_valid_ = false;
};

// 静态变量，用于在程序启动时初始化 Winsock
static WSInit wsinit_;
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
inline std::pair<std::string, std::string> make_digest_authentication_header(
    const Request &req, const std::map<std::string, std::string> &auth,
    size_t cnonce_count, const std::string &cnonce, const std::string &username,
    const std::string &password, bool is_proxy = false) {
  // 初始化 nc 变量
  std::string nc;
  {
    // 使用 cnonce_count 构造 nc 变量
    std::stringstream ss;
    ss << std::setfill('0') << std::setw(8) << std::hex << cnonce_count;
    nc = ss.str();
  }

  // 初始化 qop 变量
  std::string qop;
  // 检查是否存在 qop 字段
  if (auth.find("qop") != auth.end()) {
    qop = auth.at("qop");
    // 如果 qop 包含 "auth-int"，则设置 qop 为 "auth-int"
    if (qop.find("auth-int") != std::string::npos) {
      qop = "auth-int";
    } 
    // 如果 qop 包含 "auth"，则设置 qop 为 "auth"
    else if (qop.find("auth") != std::string::npos) {
      qop = "auth";
    } 
    // 否则清空 qop
    else {
      qop.clear();
    }
  }

  // 初始化 algo 变量
  std::string algo = "MD5";
  // 检查是否存在 algorithm 字段
  if (auth.find("algorithm") != auth.end()) { algo = auth.at("algorithm"); }

  // 初始化 response 变量
  std::string response;
  {
    // 根据算法选择哈希函数
    auto H = algo == "SHA-256"   ? detail::SHA_256
             : algo == "SHA-512" ? detail::SHA_512
                                 : detail::MD5;

    // 计算 A1
    auto A1 = username + ":" + auth.at("realm") + ":" + password;

    // 计算 A2
    auto A2 = req.method + ":" + req.path;
    // 如果 qop 为 "auth-int"，则在 A2 后面加上请求体的哈希值
    if (qop == "auth-int") { A2 += ":" + H(req.body); }

    // 如果 qop 为空
    if (qop.empty()) {
      // 计算 response
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + H(A2));
    } else {
      // 计算 response
      response = H(H(A1) + ":" + auth.at("nonce") + ":" + nc + ":" + cnonce +
                   ":" + qop + ":" + H(A2));
    }
  }
    }
  }

  // 检查是否存在 opaque 字段，如果存在则取出其值，否则为空字符串
  auto opaque = (auth.find("opaque") != auth.end()) ? auth.at("opaque") : "";

  // 拼接认证字段，包括用户名、领域、随机数、URI、算法等信息
  auto field = "Digest username=\"" + username + "\", realm=\"" +
               auth.at("realm") + "\", nonce=\"" + auth.at("nonce") +
               "\", uri=\"" + req.path + "\", algorithm=" + algo +
               (qop.empty() ? ", response=\""
                            : ", qop=" + qop + ", nc=" + nc + ", cnonce=\"" +
                                  cnonce + "\", response=\"") +
               response + "\"" +
               (opaque.empty() ? "" : ", opaque=\"" + opaque + "\"");

  // 根据是否为代理，选择不同的认证字段名称
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  // 返回认证字段名称和字段内容的键值对
  return std::make_pair(key, field);
// 结束条件编译指令
}
#endif

// 解析响应中的认证信息，并存储到auth映射中，is_proxy表示是否为代理
inline bool parse_www_authenticate(const Response &res,
                                   std::map<std::string, std::string> &auth,
                                   bool is_proxy) {
  // 根据is_proxy选择认证信息的键值
  auto auth_key = is_proxy ? "Proxy-Authenticate" : "WWW-Authenticate";
  // 如果响应头中包含认证信息
  if (res.has_header(auth_key)) {
    // 定义正则表达式，用于解析认证信息
    static auto re = std::regex(R"~((?:(?:,\s*)?(.+?)=(?:"(.*?)"|([^,]*))))~");
    // 获取认证信息的值
    auto s = res.get_header_value(auth_key);
    // 查找空格的位置
    auto pos = s.find(' ');
    // 如果找到空格
    if (pos != std::string::npos) {
      // 获取认证类型
      auto type = s.substr(0, pos);
      // 如果是基本认证
      if (type == "Basic") {
        return false;
      } 
      // 如果是摘要认证
      else if (type == "Digest") {
        // 截取认证信息的值
        s = s.substr(pos + 1);
        // 使用正则表达式解析认证信息
        auto beg = std::sregex_iterator(s.begin(), s.end(), re);
        for (auto i = beg; i != std::sregex_iterator(); ++i) {
          auto m = *i;
          // 获取键和值
          auto key = s.substr(static_cast<size_t>(m.position(1)),
                              static_cast<size_t>(m.length(1)));
          auto val = m.length(2) > 0
                         ? s.substr(static_cast<size_t>(m.position(2)),
                                    static_cast<size_t>(m.length(2)))
                         : s.substr(static_cast<size_t>(m.position(3)),
                                    static_cast<size_t>(m.length(3)));
          // 存储到auth映射中
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
  // 生成随机字符的lambda函数
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

// ContentProviderAdapter类
class ContentProviderAdapter {
// ContentProviderAdapter 类的构造函数，接受一个 ContentProviderWithoutLength 类型的参数，并将其保存在成员变量 content_provider_ 中
public:
  explicit ContentProviderAdapter(
      ContentProviderWithoutLength &&content_provider)
      : content_provider_(content_provider) {}

  // 重载函数调用操作符，接受偏移量和数据接收器参数，调用 content_provider_ 对象的操作符函数
  bool operator()(size_t offset, size_t, DataSink &sink) {
    return content_provider_(offset, sink);
  }

private:
  ContentProviderWithoutLength content_provider_;
};

} // namespace detail

// 根据主机名获取对应的 IP 地址，返回第一个 IP 地址
inline std::string hosted_at(const std::string &hostname) {
  std::vector<std::string> addrs;
  hosted_at(hostname, addrs);
  if (addrs.empty()) { return std::string(); }
  return addrs[0];
}

// 根据主机名获取对应的 IP 地址，将结果保存在 addrs 参数中
inline void hosted_at(const std::string &hostname,
                      std::vector<std::string> &addrs) {
  // 初始化 addrinfo 结构体 hints
  struct addrinfo hints;
  struct addrinfo *result;

  memset(&hints, 0, sizeof(struct addrinfo));
  hints.ai_family = AF_UNSPEC;
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_protocol = 0;

  // 获取主机名对应的地址信息
  if (getaddrinfo(hostname.c_str(), nullptr, &hints, &result)) {
#if defined __linux__ && !defined __ANDROID__
    res_init();
#endif
    return;
  }

  // 遍历获取的地址信息，将 IP 地址添加到 addrs 中
  for (auto rp = result; rp; rp = rp->ai_next) {
    const auto &addr =
        *reinterpret_cast<struct sockaddr_storage *>(rp->ai_addr);
    std::string ip;
    int dummy = -1;
    if (detail::get_ip_and_port(addr, sizeof(struct sockaddr_storage), ip,
                                dummy)) {
      addrs.push_back(ip);
    }
  }

  freeaddrinfo(result);
}

// 在路径后追加查询参数，返回新的路径
inline std::string append_query_params(const std::string &path,
                                       const Params &params) {
  std::string path_with_query = path;
  const static std::regex re("[^?]+\\?.*");
  auto delm = std::regex_match(path, re) ? '&' : '?';
  path_with_query += delm + detail::params_to_query_str(params);
  return path_with_query;
}

// 生成 Range 头部的字符串，返回 Range 头部字段和值的 pair
inline std::pair<std::string, std::string> make_range_header(Ranges ranges) {
  std::string field = "bytes=";
  auto i = 0;
  for (auto r : ranges) {
    if (i != 0) { field += ", "; }
    if (r.first != -1) { field += std::to_string(r.first); }
    field += '-';
    # 如果 r.second 不等于 -1，则将其转换为字符串并添加到 field 中
    if (r.second != -1) { field += std::to_string(r.second); }
    # 增加计数器 i 的值
    i++;
    # 返回一个包含 "Range" 和 field 的 pair 对象
    return std::make_pair("Range", std::move(field));
// 创建基本身份验证头部信息，包括用户名和密码，返回键值对
inline std::pair<std::string, std::string>
make_basic_authentication_header(const std::string &username,
                                 const std::string &password, bool is_proxy) {
  // 拼接用户名和密码，并进行 base64 编码，生成认证字段
  auto field = "Basic " + detail::base64_encode(username + ":" + password);
  // 根据是否代理，选择不同的键名
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  return std::make_pair(key, std::move(field));
}

// 创建 Bearer token 身份验证头部信息，包括 token，返回键值对
inline std::pair<std::string, std::string>
make_bearer_token_authentication_header(const std::string &token,
                                        bool is_proxy = false) {
  // 拼接 token，生成认证字段
  auto field = "Bearer " + token;
  // 根据是否代理，选择不同的键名
  auto key = is_proxy ? "Proxy-Authorization" : "Authorization";
  return std::make_pair(key, std::move(field));
}

// 检查请求头中是否包含指定键名的头部信息
inline bool Request::has_header(const std::string &key) const {
  return detail::has_header(headers, key);
}

// 获取请求头中指定键名的头部信息值
inline std::string Request::get_header_value(const std::string &key,
                                             size_t id) const {
  return detail::get_header_value(headers, key, id, "");
}

// 获取请求头中指定键名的头部信息值的数量
inline size_t Request::get_header_value_count(const std::string &key) const {
  // 获取指定键名的头部信息值的数量
  auto r = headers.equal_range(key);
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置请求头中指定键名的头部信息值
inline void Request::set_header(const std::string &key,
                                const std::string &val) {
  // 如果键名和值都不包含换行符，设置请求头信息
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    headers.emplace(key, val);
  }
}

// 检查请求参数中是否包含指定键名的参数
inline bool Request::has_param(const std::string &key) const {
  return params.find(key) != params.end();
}

// 获取请求参数中指定键名的参数值
inline std::string Request::get_param_value(const std::string &key,
                                            size_t id) const {
  // 获取指定键名的参数值
  auto rng = params.equal_range(key);
  auto it = rng.first;
  std::advance(it, static_cast<ssize_t>(id));
  if (it != rng.second) { return it->second; }
  return std::string();
}
// 返回指定键对应的参数值的数量
inline size_t Request::get_param_value_count(const std::string &key) const {
  // 查找指定键的参数范围
  auto r = params.equal_range(key);
  // 返回参数数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 检查请求是否为多部分表单数据
inline bool Request::is_multipart_form_data() const {
  // 获取请求头中的 Content-Type
  const auto &content_type = get_header_value("Content-Type");
  // 检查 Content-Type 是否为 multipart/form-data
  return !content_type.rfind("multipart/form-data", 0);
}

// 检查是否存在指定键的文件
inline bool Request::has_file(const std::string &key) const {
  // 查找文件中是否存在指定键
  return files.find(key) != files.end();
}

// 获取指定键对应的文件值
inline MultipartFormData Request::get_file_value(const std::string &key) const {
  // 查找指定键对应的文件值
  auto it = files.find(key);
  // 如果存在则返回文件值，否则返回空的 MultipartFormData 对象
  if (it != files.end()) { return it->second; }
  return MultipartFormData();
}

// 获取指定键对应的文件值列表
inline std::vector<MultipartFormData>
Request::get_file_values(const std::string &key) const {
  // 创建文件值列表
  std::vector<MultipartFormData> values;
  // 获取指定键对应的文件值范围
  auto rng = files.equal_range(key);
  // 遍历文件值范围，将文件值添加到列表中
  for (auto it = rng.first; it != rng.second; it++) {
    values.push_back(it->second);
  }
  // 返回文件值列表
  return values;
}

// 检查是否存在指定键的响应头
inline bool Response::has_header(const std::string &key) const {
  // 查找响应头中是否存在指定键
  return headers.find(key) != headers.end();
}

// 获取指定键对应的响应头值
inline std::string Response::get_header_value(const std::string &key,
                                              size_t id) const {
  // 调用 detail 命名空间中的函数获取指定键对应的响应头值
  return detail::get_header_value(headers, key, id, "");
}

// 返回指定键对应的响应头值的数量
inline size_t Response::get_header_value_count(const std::string &key) const {
  // 查找指定键的响应头值范围
  auto r = headers.equal_range(key);
  // 返回响应头值的数量
  return static_cast<size_t>(std::distance(r.first, r.second));
}

// 设置指定键对应的响应头
inline void Response::set_header(const std::string &key,
                                 const std::string &val) {
  // 检查键和值中是否包含换行符，如果不包含则添加到响应头中
  if (!detail::has_crlf(key) && !detail::has_crlf(val)) {
    headers.emplace(key, val);
  }
}

// 设置重定向响应
inline void Response::set_redirect(const std::string &url, int stat) {
  // 检查 URL 中是否包含换行符，如果不包含则设置 Location 响应头
  if (!detail::has_crlf(url)) {
    set_header("Location", url);
    // 设置状态码为指定值或默认值
    if (300 <= stat && stat < 400) {
      this->status = stat;
    } else {
      this->status = 302;
    }
  }
}
// 设置响应内容，接受字符指针和大小，以及内容类型
inline void Response::set_content(const char *s, size_t n,
                                  const std::string &content_type) {
  // 使用字符指针和大小设置响应体
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
  // 调用前一个函数，传入字符串的指针和大小
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
  // 设置是否为分块内容提供者
  is_chunked_content_provider_ = true;
}

// 检查请求头部中是否存在指定的键
inline bool Result::has_request_header(const std::string &key) const {
  // 返回请求头部中是否存在指定的键
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

// 写入字符指针的数据到流中
inline ssize_t Stream::write(const char *ptr) {
  return write(ptr, strlen(ptr));
}

// 写入字符串数据到流中
inline ssize_t Stream::write(const std::string &s) {
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

// 检查套接字是否可读
inline bool SocketStream::is_readable() const {
  return select_read(sock_, read_timeout_sec_, read_timeout_usec_) > 0;
}

// 检查套接字是否可写
inline bool SocketStream::is_writable() const {
  return select_write(sock_, write_timeout_sec_, write_timeout_usec_) > 0 &&
         is_socket_alive(sock_);
}

// 从套接字中读取数据到指定的字符指针中
inline ssize_t SocketStream::read(char *ptr, size_t size) {
#ifdef _WIN32
  size =
      (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#else
  size = (std::min)(size,
                    static_cast<size_t>((std::numeric_limits<ssize_t>::max)()));
#endif

  if (read_buff_off_ < read_buff_content_size_) {
    auto remaining_size = read_buff_content_size_ - read_buff_off_;
    # 如果请求的数据大小小于等于剩余缓冲区大小
    if (size <= remaining_size) {
      # 将缓冲区中的数据拷贝到指定的内存地址中
      memcpy(ptr, read_buff_.data() + read_buff_off_, size);
      # 更新缓冲区偏移量
      read_buff_off_ += size;
      # 返回实际读取的数据大小
      return static_cast<ssize_t>(size);
    } else {
      # 如果请求的数据大小大于剩余缓冲区大小，则只拷贝剩余缓冲区大小的数据
      memcpy(ptr, read_buff_.data() + read_buff_off_, remaining_size);
      # 更新缓冲区偏移量
      read_buff_off_ += remaining_size;
      # 返回实际读取的数据大小
      return static_cast<ssize_t>(remaining_size);
    }
  }

  # 如果不可读，则返回错误
  if (!is_readable()) { return -1; }

  # 重置缓冲区偏移量和缓冲区内容大小
  read_buff_off_ = 0;
  read_buff_content_size_ = 0;

  # 如果请求的数据大小小于缓冲区大小
  if (size < read_buff_size_) {
    # 从套接字中读取数据到缓冲区中
    auto n = read_socket(sock_, read_buff_.data(), read_buff_size_,
                         CPPHTTPLIB_RECV_FLAGS);
    # 如果读取失败，则返回错误码
    if (n <= 0) {
      return n;
    } else if (n <= static_cast<ssize_t>(size)) {
      # 如果实际读取的数据大小小于等于请求的数据大小，则将数据拷贝到指定的内存地址中
      memcpy(ptr, read_buff_.data(), static_cast<size_t>(n));
      # 返回实际读取的数据大小
      return n;
    } else {
      # 如果实际读取的数据大小大于请求的数据大小，则只拷贝请求的数据大小的数据
      memcpy(ptr, read_buff_.data(), size);
      # 更新缓冲区偏移量和缓冲区内容大小
      read_buff_off_ = size;
      read_buff_content_size_ = static_cast<size_t>(n);
      # 返回实际读取的数据大小
      return static_cast<ssize_t>(size);
    }
  } else {
    # 如果请求的数据大小大于等于缓冲区大小，则直接从套接字中读取数据到指定的内存地址中
    return read_socket(sock_, ptr, size, CPPHTTPLIB_RECV_FLAGS);
  }
} // 结束 SocketStream 类的定义

// 写入数据到套接字流
inline ssize_t SocketStream::write(const char *ptr, size_t size) {
  // 如果不可写，返回 -1
  if (!is_writable()) { return -1; }

  // 如果在 Windows 32 位平台，并且不是 Windows 64 位平台
  // 限制 size 的大小为 int 类型的最大值
#if defined(_WIN32) && !defined(_WIN64)
  size =
      (std::min)(size, static_cast<size_t>((std::numeric_limits<int>::max)()));
#endif

  // 发送数据到套接字
  return send_socket(sock_, ptr, size, CPPHTTPLIB_SEND_FLAGS);
}

// 获取远程 IP 地址和端口
inline void SocketStream::get_remote_ip_and_port(std::string &ip,
                                                 int &port) const {
  // 调用 detail 命名空间的函数获取远程 IP 地址和端口
  return detail::get_remote_ip_and_port(sock_, ip, port);
}

// 获取本地 IP 地址和端口
inline void SocketStream::get_local_ip_and_port(std::string &ip,
                                                int &port) const {
  // 调用 detail 命名空间的函数获取本地 IP 地址和端口
  return detail::get_local_ip_and_port(sock_, ip, port);
}

// 返回套接字
inline socket_t SocketStream::socket() const { return sock_; }

// BufferStream 类的实现

// 判断是否可读
inline bool BufferStream::is_readable() const { return true; }

// 判断是否可写
inline bool BufferStream::is_writable() const { return true; }

// 从缓冲区读取数据
inline ssize_t BufferStream::read(char *ptr, size_t size) {
  // 如果是 Visual Studio 2015 及更早版本
  // 使用 _Copy_s 函数
  // 否则使用 copy 函数
#if defined(_MSC_VER) && _MSC_VER < 1910
  auto len_read = buffer._Copy_s(ptr, size, size, position);
#else
  auto len_read = buffer.copy(ptr, size, position);
#endif
  position += static_cast<size_t>(len_read);
  return static_cast<ssize_t>(len_read);
}

// 写入数据到缓冲区
inline ssize_t BufferStream::write(const char *ptr, size_t size) {
  buffer.append(ptr, size);
  return static_cast<ssize_t>(size);
}

// 获取远程 IP 地址和端口
inline void BufferStream::get_remote_ip_and_port(std::string & /*ip*/,
                                                 int & /*port*/) const {}

// 获取本地 IP 地址和端口
inline void BufferStream::get_local_ip_and_port(std::string & /*ip*/,
                                                int & /*port*/) const {}

// 返回套接字
inline socket_t BufferStream::socket() const { return 0; }

// 返回缓冲区内容
inline const std::string &BufferStream::get_buffer() const { return buffer; }

} // 结束 detail 命名空间

// HTTP 服务器的实现
// 构造函数
inline Server::Server()
    : new_task_queue(
          [] { return new ThreadPool(CPPHTTPLIB_THREAD_POOL_COUNT); }) {
#ifndef _WIN32
  // 如果不是在 Windows 平台下，忽略 SIGPIPE 信号
  signal(SIGPIPE, SIG_IGN);
#endif
}

// 空的 Server 类析构函数
inline Server::~Server() {}

// 添加 GET 请求处理函数
inline Server &Server::Get(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理函数添加到 GET 请求处理函数列表中
  get_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加 POST 请求处理函数
inline Server &Server::Post(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理函数添加到 POST 请求处理函数列表中
  post_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加带内容读取器的 POST 请求处理函数
inline Server &Server::Post(const std::string &pattern,
                            HandlerWithContentReader handler) {
  // 将匹配模式和带内容读取器的处理函数添加到 POST 请求处理函数列表中
  post_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加 PUT 请求处理函数
inline Server &Server::Put(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理函数添加到 PUT 请求处理函数列表中
  put_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加带内容读取器的 PUT 请求处理函数
inline Server &Server::Put(const std::string &pattern,
                           HandlerWithContentReader handler) {
  // 将匹配模式和带内容读取器的处理函数添加到 PUT 请求处理函数列表中
  put_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加 PATCH 请求处理函数
inline Server &Server::Patch(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理函数添加到 PATCH 请求处理函数列表中
  patch_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加带内容读取器的 PATCH 请求处理函数
inline Server &Server::Patch(const std::string &pattern,
                             HandlerWithContentReader handler) {
  // 将匹配模式和带内容读取器的处理函数添加到 PATCH 请求处理函数列表中
  patch_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}

// 添加 DELETE 请求处理函数
inline Server &Server::Delete(const std::string &pattern, Handler handler) {
  // 将匹配模式和处理函数添加到 DELETE 请求处理函数列表中
  delete_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  return *this;
}
# 在服务器对象中注册一个用于处理删除请求的处理程序，根据给定的模式和内容读取器
inline Server &Server::Delete(const std::string &pattern,
                              HandlerWithContentReader handler) {
  # 将给定的模式和处理程序添加到删除处理程序列表中
  delete_handlers_for_content_reader_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  # 返回服务器对象的引用
  return *this;
}

# 在服务器对象中注册一个用于处理选项请求的处理程序，根据给定的模式
inline Server &Server::Options(const std::string &pattern, Handler handler) {
  # 将给定的模式和处理程序添加到选项处理程序列表中
  options_handlers_.push_back(
      std::make_pair(std::regex(pattern), std::move(handler)));
  # 返回服务器对象的引用
  return *this;
}

# 设置服务器对象的基本目录
inline bool Server::set_base_dir(const std::string &dir,
                                 const std::string &mount_point) {
  # 调用set_mount_point函数，将挂载点和目录关联起来
  return set_mount_point(mount_point, dir);
}

# 设置服务器对象的挂载点和目录，以及可选的头部信息
inline bool Server::set_mount_point(const std::string &mount_point,
                                    const std::string &dir, Headers headers) {
  # 如果目录存在
  if (detail::is_dir(dir)) {
    # 如果挂载点不为空并且以斜杠开头
    std::string mnt = !mount_point.empty() ? mount_point : "/";
    if (!mnt.empty() && mnt[0] == '/') {
      # 将挂载点、目录和头部信息添加到基本目录列表中
      base_dirs_.push_back({mnt, dir, std::move(headers)});
      return true;
    }
  }
  return false;
}

# 移除服务器对象的指定挂载点
inline bool Server::remove_mount_point(const std::string &mount_point) {
  # 遍历基本目录列表
  for (auto it = base_dirs_.begin(); it != base_dirs_.end(); ++it) {
    # 如果找到指定的挂载点
    if (it->mount_point == mount_point) {
      # 移除该挂载点
      base_dirs_.erase(it);
      return true;
    }
  }
  return false;
}

# 设置文件扩展名和 MIME 类型的映射关系
inline Server &
Server::set_file_extension_and_mimetype_mapping(const std::string &ext,
                                                const std::string &mime) {
  # 将文件扩展名和 MIME 类型添加到映射关系中
  file_extension_and_mimetype_map_[ext] = mime;
  # 返回服务器对象的引用
  return *this;
}

# 设置服务器对象的文件请求处理程序
inline Server &Server::set_file_request_handler(Handler handler) {
  # 设置文件请求处理程序
  file_request_handler_ = std::move(handler);
  # 返回服务器对象的引用
  return *this;
}

# 设置服务器对象的错误处理程序，根据请求和响应
inline Server &Server::set_error_handler(HandlerWithResponse handler) {
  # 设置错误处理程序
  error_handler_ = std::move(handler);
  # 返回服务器对象的引用
  return *this;
}

# 设置服务器对象的错误处理程序，只接受请求处理程序
inline Server &Server::set_error_handler(Handler handler) {
  # 设置错误处理程序，将请求处理程序包装成带响应的处理程序
  error_handler_ = [handler](const Request &req, Response &res) {
    handler(req, res);
    return HandlerResponse::Handled;
  };
  # 返回服务器对象的引用
  return *this;
}
# 设置异常处理程序，使用 std::move 将 handler 移动到 exception_handler_，并返回 Server 对象的引用
inline Server &Server::set_exception_handler(ExceptionHandler handler) {
  exception_handler_ = std::move(handler);
  return *this;
}

# 设置预路由处理程序，使用 std::move 将 handler 移动到 pre_routing_handler_，并返回 Server 对象的引用
inline Server &Server::set_pre_routing_handler(HandlerWithResponse handler) {
  pre_routing_handler_ = std::move(handler);
  return *this;
}

# 设置后路由处理程序，使用 std::move 将 handler 移动到 post_routing_handler_，并返回 Server 对象的引用
inline Server &Server::set_post_routing_handler(Handler handler) {
  post_routing_handler_ = std::move(handler);
  return *this;
}

# 设置日志记录器，使用 std::move 将 logger 移动到 logger_，并返回 Server 对象的引用
inline Server &Server::set_logger(Logger logger) {
  logger_ = std::move(logger);
  return *this;
}

# 设置 100 Continue 期望处理程序，使用 std::move 将 handler 移动到 expect_100_continue_handler_，并返回 Server 对象的引用
inline Server &Server::set_expect_100_continue_handler(Expect100ContinueHandler handler) {
  expect_100_continue_handler_ = std::move(handler);
  return *this;
}

# 设置地址族，将 family 赋值给 address_family_，并返回 Server 对象的引用
inline Server &Server::set_address_family(int family) {
  address_family_ = family;
  return *this;
}

# 设置 TCP NoDelay 选项，将 on 赋值给 tcp_nodelay_，并返回 Server 对象的引用
inline Server &Server::set_tcp_nodelay(bool on) {
  tcp_nodelay_ = on;
  return *this;
}

# 设置套接字选项，使用 std::move 将 socket_options 移动到 socket_options_，并返回 Server 对象的引用
inline Server &Server::set_socket_options(SocketOptions socket_options) {
  socket_options_ = std::move(socket_options);
  return *this;
}

# 设置默认头部，使用 std::move 将 headers 移动到 default_headers_，并返回 Server 对象的引用
inline Server &Server::set_default_headers(Headers headers) {
  default_headers_ = std::move(headers);
  return *this;
}

# 设置保持活动的最大计数，将 count 赋值给 keep_alive_max_count_，并返回 Server 对象的引用
inline Server &Server::set_keep_alive_max_count(size_t count) {
  keep_alive_max_count_ = count;
  return *this;
}

# 设置保持活动的超时时间，将 sec 赋值给 keep_alive_timeout_sec_，并返回 Server 对象的引用
inline Server &Server::set_keep_alive_timeout(time_t sec) {
  keep_alive_timeout_sec_ = sec;
  return *this;
}

# 设置读取超时时间，将 sec 赋值给 read_timeout_sec_，usec 赋值给 read_timeout_usec_，并返回 Server 对象的引用
inline Server &Server::set_read_timeout(time_t sec, time_t usec) {
  read_timeout_sec_ = sec;
  read_timeout_usec_ = usec;
  return *this;
}

# 设置写入超时时间，将 sec 赋值给 write_timeout_sec_，usec 赋值给 write_timeout_usec_，并返回 Server 对象的引用
inline Server &Server::set_write_timeout(time_t sec, time_t usec) {
  write_timeout_sec_ = sec;
  write_timeout_usec_ = usec;
  return *this;
}

# 设置空闲间隔时间，将 sec 赋值给 idle_interval_sec_，usec 赋值给 idle_interval_usec_，并返回 Server 对象的引用
inline Server &Server::set_idle_interval(time_t sec, time_t usec) {
  idle_interval_sec_ = sec;
  idle_interval_usec_ = usec;
  return *this;
}

# 设置有效载荷最大长度，将 length 赋值给 payload_max_length_，并返回 Server 对象的引用
inline Server &Server::set_payload_max_length(size_t length) {
  payload_max_length_ = length;
  return *this;
}
// 绑定到指定主机和端口，使用给定的套接字标志
inline bool Server::bind_to_port(const std::string &host, int port,
                                 int socket_flags) {
  // 调用内部绑定函数，如果返回值小于 0 则返回 false
  if (bind_internal(host, port, socket_flags) < 0) return false;
  // 返回 true
  return true;
}

// 绑定到任意端口，使用给定的套接字标志
inline int Server::bind_to_any_port(const std::string &host, int socket_flags) {
  // 调用内部绑定函数，返回结果
  return bind_internal(host, 0, socket_flags);
}

// 绑定后开始监听
inline bool Server::listen_after_bind() {
  // 创建一个 scope_exit 对象，设置 done_ 为 true
  auto se = detail::scope_exit([&]() { done_ = true; });
  // 调用内部监听函数
  return listen_internal();
}

// 绑定到指定主机和端口，使用给定的套接字标志，并开始监听
inline bool Server::listen(const std::string &host, int port,
                           int socket_flags) {
  // 创建一个 scope_exit 对象，设置 done_ 为 true
  auto se = detail::scope_exit([&]() { done_ = true; });
  // 调用绑定到端口函数和内部监听函数，返回结果
  return bind_to_port(host, port, socket_flags) && listen_internal();
}

// 返回服务器是否正在运行
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
    // 断言服务器套接字不为无效套接字
    assert(svr_sock_ != INVALID_SOCKET);
    // 创建一个原子的套接字对象，将服务器套接字设置为无效套接字
    std::atomic<socket_t> sock(svr_sock_.exchange(INVALID_SOCKET));
    // 关闭套接字
    detail::shutdown_socket(sock);
    detail::close_socket(sock);
  }
}

// 解析请求行
inline bool Server::parse_request_line(const char *s, Request &req) {
  // 获取字符串长度
  auto len = strlen(s);
  // 如果长度小于 2 或倒数第二个字符不是 '\r' 或最后一个字符不是 '\n'，返回 false
  if (len < 2 || s[len - 2] != '\r' || s[len - 1] != '\n') { return false; }
  len -= 2;

  {
    size_t count = 0;

    // 使用空格分割字符串，并根据 count 的值填充请求对象的 method、target、version
    detail::split(s, s + len, ' ', [&](const char *b, const char *e) {
      switch (count) {
      case 0: req.method = std::string(b, e); break;
      case 1: req.target = std::string(b, e); break;
      case 2: req.version = std::string(b, e); break;
      default: break;
      }
      count++;
    });
  }
}
    // 如果请求参数不等于3，则返回false
    if (count != 3) { return false; }
  }

  // 定义HTTP请求方法的集合
  static const std::set<std::string> methods{
      "GET",     "HEAD",    "POST",  "PUT",   "DELETE",
      "CONNECT", "OPTIONS", "TRACE", "PATCH", "PRI"};

  // 如果请求方法不在定义的方法集合中，则返回false
  if (methods.find(req.method) == methods.end()) { return false; }

  // 如果请求版本不是"HTTP/1.1"且不是"HTTP/1.0"，则返回false
  if (req.version != "HTTP/1.1" && req.version != "HTTP/1.0") { return false; }

  {
    // 跳过URL片段
    for (size_t i = 0; i < req.target.size(); i++) {
      if (req.target[i] == '#') {
        req.target.erase(i);
        break;
      }
    }

    size_t count = 0;

    // 分割URL参数
    detail::split(req.target.data(), req.target.data() + req.target.size(), '?',
                  [&](const char *b, const char *e) {
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
                    count++;
                  });

    // 如果参数数量大于2，则返回false
    if (count > 2) { return false; }
  }

  // 返回true
  return true;
  // 写入响应内容到流中，根据需要关闭连接
  inline bool Server::write_response(Stream &strm, bool close_connection,
                                     const Request &req, Response &res) {
    return write_response_core(strm, close_connection, req, res, false);
  }

  // 写入带有内容的响应到流中，根据需要关闭连接
  inline bool Server::write_response_with_content(Stream &strm,
                                                  bool close_connection,
                                                  const Request &req,
                                                  Response &res) {
    return write_response_core(strm, close_connection, req, res, true);
  }

  // 写入响应核心功能，根据需要关闭连接和是否需要应用范围
  inline bool Server::write_response_core(Stream &strm, bool close_connection,
                                          const Request &req, Response &res,
                                          bool need_apply_ranges) {
    assert(res.status != -1);

    // 如果状态码大于等于400且存在错误处理程序，则处理错误
    if (400 <= res.status && error_handler_ &&
        error_handler_(req, res) == HandlerResponse::Handled) {
      need_apply_ranges = true;
    }

    std::string content_type;
    std::string boundary;
    // 如果需要应用范围，则调用apply_ranges函数
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

    // 如果响应中没有Content-Type头部信息且存在响应体或内容长度大于0或存在内容提供程序，则设置Content-Type为text/plain
    if (!res.has_header("Content-Type") &&
        (!res.body.empty() || res.content_length_ > 0 || res.content_provider_)) {
      res.set_header("Content-Type", "text/plain");
    }

    // 如果响应中没有Content-Length头部信息且响应体为空且内容长度为0且不存在内容提供程序，则设置Content-Length为0
    if (!res.has_header("Content-Length") && res.body.empty() &&
        !res.content_length_ && !res.content_provider_) {
      res.set_header("Content-Length", "0");
    }

    // 如果响应中没有Accept-Ranges头部信息且请求方法为HEAD，则设置Accept-Ranges为bytes
    if (!res.has_header("Accept-Ranges") && req.method == "HEAD") {
      res.set_header("Accept-Ranges", "bytes");
    }

    // 如果存在后置路由处理程序，则调用该处理程序
    if (post_routing_handler_) { post_routing_handler_(req, res); }

    // 响应行和头部信息
    {
    // 创建一个BufferStream对象
    detail::BufferStream bstrm;

    // 写入HTTP响应的状态行，如果失败则返回false
    if (!bstrm.write_format("HTTP/1.1 %d %s\r\n", res.status,
                            detail::status_message(res.status))) {
      return false;
    }

    // 写入响应头部，如果失败则返回false
    if (!detail::write_headers(bstrm, res.headers)) { return false; }

    // 刷新缓冲区
    auto &data = bstrm.get_buffer();
    detail::write_data(strm, data.data(), data.size());
  }

  // 响应体
  auto ret = true;
  if (req.method != "HEAD") {
    // 如果响应体不为空，则写入响应体数据
    if (!res.body.empty()) {
      if (!detail::write_data(strm, res.body.data(), res.body.size())) {
        ret = false;
      }
    } else if (res.content_provider_) {
      // 如果存在内容提供者，则使用内容提供者写入响应体数据
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

  // 返回操作结果
  return ret;
// 写入带有提供程序的内容到流中
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
    // 如果请求范围为空
    if (req.ranges.empty()) {
      return detail::write_content(strm, res.content_provider_, 0,
                                   res.content_length_, is_shutting_down);
    } 
    // 如果请求范围只有一个
    else if (req.ranges.size() == 1) {
      auto offsets =
          detail::get_range_offset_and_length(req, res.content_length_, 0);
      auto offset = offsets.first;
      auto length = offsets.second;
      return detail::write_content(strm, res.content_provider_, offset, length,
                                   is_shutting_down);
    } 
    // 如果请求范围不止一个
    else {
      return detail::write_multipart_ranges_data(
          strm, req, res, boundary, content_type, is_shutting_down);
    }
  } 
  // 如果响应内容长度为0
  else {
    // 如果是分块内容提供程序
    if (res.is_chunked_content_provider_) {
      auto type = detail::encoding_type(req, res);

      std::unique_ptr<detail::compressor> compressor;
      // 如果是Gzip编码类型
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        compressor = detail::make_unique<detail::gzip_compressor>();
#endif
      } 
      // 如果是Brotli编码类型
      else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        compressor = detail::make_unique<detail::brotli_compressor>();
#endif
      } 
      // 如果是其他编码类型
      else {
        compressor = detail::make_unique<detail::nocompressor>();
      }
      assert(compressor != nullptr);

      return detail::write_content_chunked(strm, res.content_provider_,
                                           is_shutting_down, *compressor);
    } 
    // 如果不是分块内容提供程序
    else {
      return detail::write_content_without_length(strm, res.content_provider_,
                                                  is_shutting_down);
    }
  }
}
inline bool Server::read_content(Stream &strm, Request &req, Response &res) {
  MultipartFormDataMap::iterator cur;  // 定义一个迭代器cur，用于遍历multipart form data map
  auto file_count = 0;  // 初始化文件计数器为0
  if (read_content_core(
          strm, req, res,
          // Regular
          [&](const char *buf, size_t n) {  // 传入lambda表达式，处理普通请求体数据
            if (req.body.size() + n > req.body.max_size()) { return false; }  // 如果请求体数据大小超过最大限制，则返回false
            req.body.append(buf, n);  // 将数据追加到请求体中
            return true;  // 返回true表示处理成功
          },
          // Multipart
          [&](const MultipartFormData &file) {  // 传入lambda表达式，处理multipart请求体数据
            if (file_count++ == CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT) {  // 如果文件计数器达到最大限制，则返回false
              return false;
            }
            cur = req.files.emplace(file.name, file);  // 将multipart form data插入到请求对象的文件map中
            return true;  // 返回true表示处理成功
          },
          [&](const char *buf, size_t n) {  // 传入lambda表达式，处理multipart请求体数据的内容
            auto &content = cur->second.content;  // 获取当前文件的内容
            if (content.size() + n > content.max_size()) { return false; }  // 如果内容大小超过最大限制，则返回false
            content.append(buf, n);  // 将数据追加到内容中
            return true;  // 返回true表示处理成功
          })) {
    const auto &content_type = req.get_header_value("Content-Type");  // 获取请求头中的Content-Type
    if (!content_type.find("application/x-www-form-urlencoded")) {  // 如果Content-Type不是application/x-www-form-urlencoded
      if (req.body.size() > CPPHTTPLIB_FORM_URL_ENCODED_PAYLOAD_MAX_LENGTH) {  // 如果请求体数据大小超过最大限制
        res.status = 413; // NOTE: should be 414?  // 设置响应状态码为413
        return false;  // 返回false表示处理失败
      }
      detail::parse_query_text(req.body, req.params);  // 解析请求体中的查询参数
    }
    return true;  // 返回true表示处理成功
  }
  return false;  // 返回false表示处理失败
}

inline bool Server::read_content_with_content_receiver(
    Stream &strm, Request &req, Response &res, ContentReceiver receiver,
    MultipartContentHeader multipart_header,
    ContentReceiver multipart_receiver) {
  return read_content_core(strm, req, res, std::move(receiver),  // 调用read_content_core函数处理请求体数据
                           std::move(multipart_header),  // 传入multipart头部处理器
                           std::move(multipart_receiver));  // 传入multipart内容处理器
}
inline bool Server::read_content_core(Stream &strm, Request &req, Response &res,
                                      ContentReceiver receiver,
                                      MultipartContentHeader multipart_header,
                                      ContentReceiver multipart_receiver) {
  // 创建一个multipart_form_data_parser对象
  detail::MultipartFormDataParser multipart_form_data_parser;
  // 创建一个ContentReceiverWithProgress对象
  ContentReceiverWithProgress out;

  // 如果请求是multipart/form-data类型
  if (req.is_multipart_form_data()) {
    // 获取请求头中的Content-Type
    const auto &content_type = req.get_header_value("Content-Type");
    std::string boundary;
    // 解析Content-Type中的boundary
    if (!detail::parse_multipart_boundary(content_type, boundary)) {
      // 如果解析失败，设置响应状态码为400，并返回false
      res.status = 400;
      return false;
    }

    // 设置multipart_form_data_parser的boundary
    multipart_form_data_parser.set_boundary(std::move(boundary));
    // 设置out为一个lambda函数，用于解析multipart数据
    out = [&](const char *buf, size_t n, uint64_t /*off*/, uint64_t /*len*/) {
      return multipart_form_data_parser.parse(buf, n, multipart_receiver,
                                              multipart_header);
    };
  } else {
    // 如果不是multipart/form-data类型，设置out为receiver
    out = [receiver](const char *buf, size_t n, uint64_t /*off*/,
                     uint64_t /*len*/) { return receiver(buf, n); };
  }

  // 如果请求方法是DELETE并且没有Content-Length头部
  if (req.method == "DELETE" && !req.has_header("Content-Length")) {
    // 返回true
    return true;
  }

  // 读取请求内容，并根据情况调用out函数处理
  if (!detail::read_content(strm, req, payload_max_length_, res.status, nullptr,
                            out, true)) {
    return false;
  }

  // 如果请求是multipart/form-data类型，检查multipart_form_data_parser是否有效
  if (req.is_multipart_form_data()) {
    if (!multipart_form_data_parser.is_valid()) {
      // 设置响应状态码为400，并返回false
      res.status = 400;
      return false;
    }
  }

  // 返回true
  return true;
}
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
        // 构造完整路径
        auto path = entry.base_dir + sub_path;
        // 如果路径以斜杠结尾，则追加 index.html
        if (path.back() == '/') { path += "index.html"; }

        // 检查路径是否为文件
        if (detail::is_file(path)) {
          // 读取文件内容到响应体
          detail::read_file(path, res.body);
          // 查找文件类型并设置 Content-Type 头部
          auto type =
              detail::find_content_type(path, file_extension_and_mimetype_map_);
          if (type) { res.set_header("Content-Type", type); }
          // 设置额外的头部
          for (const auto &kv : entry.headers) {
            res.set_header(kv.first.c_str(), kv.second);
          }
          // 根据请求是否包含 Range 头部设置响应状态码
          res.status = req.has_header("Range") ? 206 : 200;
          // 如果不是 head 请求且存在文件请求处理器，则调用处理器
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

inline socket_t
Server::create_server_socket(const std::string &host, int port,
                             int socket_flags,
                             SocketOptions socket_options) const {
  // 创建服务器套接字
  return detail::create_socket(
      host, std::string(), port, address_family_, socket_flags, tcp_nodelay_,
      std::move(socket_options),
      // 绑定套接字并监听连接
      [](socket_t sock, struct addrinfo &ai) -> bool {
        if (::bind(sock, ai.ai_addr, static_cast<socklen_t>(ai.ai_addrlen))) {
          return false;
        }
        if (::listen(sock, CPPHTTPLIB_LISTEN_BACKLOG)) { return false; }
        return true;
      });
}
// 在给定主机和端口上绑定服务器套接字，使用指定的套接字标志
inline int Server::bind_internal(const std::string &host, int port,
                                 int socket_flags) {
  // 如果服务器套接字无效，则返回-1
  if (!is_valid()) { return -1; }

  // 创建服务器套接字并赋值给成员变量svr_sock_
  svr_sock_ = create_server_socket(host, port, socket_flags, socket_options_);
  // 如果创建的服务器套接字无效，则返回-1
  if (svr_sock_ == INVALID_SOCKET) { return -1; }

  // 如果端口为0，则获取套接字地址信息并返回端口号
  if (port == 0) {
    struct sockaddr_storage addr;
    socklen_t addr_len = sizeof(addr);
    if (getsockname(svr_sock_, reinterpret_cast<struct sockaddr *>(&addr),
                    &addr_len) == -1) {
      return -1;
    }
    if (addr.ss_family == AF_INET) {
      return ntohs(reinterpret_cast<struct sockaddr_in *>(&addr)->sin_port);
    } else if (addr.ss_family == AF_INET6) {
      return ntohs(reinterpret_cast<struct sockaddr_in6 *>(&addr)->sin6_port);
    } else {
      return -1;
    }
  } else {
    return port;
  }
}

// 内部函数，用于监听服务器套接字
inline bool Server::listen_internal() {
  auto ret = true;
  // 设置服务器正在运行的标志为true
  is_running_ = true;
  // 在作用域结束时，设置服务器正在运行的标志为false
  auto se = detail::scope_exit([&]() { is_running_ = false; });

  {
    // 创建一个新的任务队列
    std::unique_ptr<TaskQueue> task_queue(new_task_queue());

    // 当服务器套接字不为无效套接字时，执行循环
    while (svr_sock_ != INVALID_SOCKET) {
#ifndef _WIN32
      // 如果不是Windows系统，且空闲间隔大于0，则执行以下代码
      if (idle_interval_sec_ > 0 || idle_interval_usec_ > 0) {
#endif
        // 使用select函数等待套接字可读，并返回结果
        auto val = detail::select_read(svr_sock_, idle_interval_sec_,
                                       idle_interval_usec_);
        // 如果返回值为0，表示超时，执行任务队列的空闲处理并继续循环
        if (val == 0) { // Timeout
          task_queue->on_idle();
          continue;
        }
#ifndef _WIN32
      }
    }
  }
}
#endif
      // 接受客户端连接，返回新的套接字
      socket_t sock = accept(svr_sock_, nullptr, nullptr);

      // 如果接受失败
      if (sock == INVALID_SOCKET) {
        // 如果错误是 EMFILE，表示打开文件描述符的进程限制已达到
        if (errno == EMFILE) {
          // 尝试在短暂休眠后重新接受新连接
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
          ; // 服务器套接字被用户关闭
        }
        break;
      }

      {
#ifdef _WIN32
        // 设置接收超时时间
        auto timeout = static_cast<uint32_t>(read_timeout_sec_ * 1000 +
                                             read_timeout_usec_ / 1000);
        setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (char *)&timeout,
                   sizeof(timeout));
#else
        // 设置接收超时时间
        timeval tv;
        tv.tv_sec = static_cast<long>(read_timeout_sec_);
        tv.tv_usec = static_cast<decltype(tv.tv_usec)>(read_timeout_usec_);
        setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (char *)&tv, sizeof(tv));
#endif
      }
      {
#ifdef _WIN32
        // 设置发送超时时间
        auto timeout = static_cast<uint32_t>(write_timeout_sec_ * 1000 +
                                             write_timeout_usec_ / 1000);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (char *)&timeout,
                   sizeof(timeout));
#else
        // 设置发送超时时间
        timeval tv;
        tv.tv_sec = static_cast<long>(write_timeout_sec_);
        tv.tv_usec = static_cast<decltype(tv.tv_usec)>(write_timeout_usec_);
        setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (char *)&tv, sizeof(tv));
#endif
      }

      // 将处理套接字的任务加入任务队列
      task_queue->enqueue([this, sock]() { process_and_close_socket(sock); });
    }

    // 关闭任务队列
    task_queue->shutdown();
  }

  return ret;
}
  // 如果存在预处理路由处理器，并且预处理路由处理器处理请求后返回已处理状态，则返回 true
  if (pre_routing_handler_ &&
      pre_routing_handler_(req, res) == HandlerResponse::Handled) {
    return true;
  }

  // 文件处理器
  // 判断是否为 HEAD 请求
  bool is_head_request = req.method == "HEAD";
  // 如果是 GET 请求或者 HEAD 请求，并且处理文件请求成功，则返回 true
  if ((req.method == "GET" || is_head_request) &&
      handle_file_request(req, res, is_head_request)) {
    return true;
  }

  // 如果请求中包含内容
  if (detail::expect_content(req)) {
    // 内容读取处理器
    {
      // 创建内容读取器
      ContentReader reader(
          // 读取内容并使用内容接收器处理
          [&](ContentReceiver receiver) {
            return read_content_with_content_receiver(
                strm, req, res, std::move(receiver), nullptr, nullptr);
          },
          // 读取多部分内容并使用内容接收器处理
          [&](MultipartContentHeader header, ContentReceiver receiver) {
            return read_content_with_content_receiver(strm, req, res, nullptr,
                                                      std::move(header),
                                                      std::move(receiver));
          });

      // 如果是 POST 请求，使用 POST 请求内容读取器处理请求
      if (req.method == "POST") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                post_handlers_for_content_reader_)) {
          return true;
        }
      } 
      // 如果是 PUT 请求，使用 PUT 请求内容读取器处理请求
      else if (req.method == "PUT") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                put_handlers_for_content_reader_)) {
          return true;
        }
      } 
      // 如果是 PATCH 请求，使用 PATCH 请求内容读取器处理请求
      else if (req.method == "PATCH") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                patch_handlers_for_content_reader_)) {
          return true;
        }
      } 
      // 如果是 DELETE 请求，使用 DELETE 请求内容读取器处理请求
      else if (req.method == "DELETE") {
        if (dispatch_request_for_content_reader(
                req, res, std::move(reader),
                delete_handlers_for_content_reader_)) {
          return true;
        }
      }
    }

    // 将内容读取到 `req.body` 中
    // 如果读取请求内容失败，则返回 false
    if (!read_content(strm, req, res)) { return false; }
  }

  // 处理常规请求
  if (req.method == "GET" || req.method == "HEAD") {
    // 如果是 GET 或 HEAD 请求，则调用相应的处理函数
    return dispatch_request(req, res, get_handlers_);
  } else if (req.method == "POST") {
    // 如果是 POST 请求，则调用相应的处理函数
    return dispatch_request(req, res, post_handlers_);
  } else if (req.method == "PUT") {
    // 如果是 PUT 请求，则调用相应的处理函数
    return dispatch_request(req, res, put_handlers_);
  } else if (req.method == "DELETE") {
    // 如果是 DELETE 请求，则调用相应的处理函数
    return dispatch_request(req, res, delete_handlers_);
  } else if (req.method == "OPTIONS") {
    // 如果是 OPTIONS 请求，则调用相应的处理函数
    return dispatch_request(req, res, options_handlers_);
  } else if (req.method == "PATCH") {
    // 如果是 PATCH 请求，则调用相应的处理函数
    return dispatch_request(req, res, patch_handlers_);
  }

  // 如果请求方法不在上述列表中，则返回 400 错误
  res.status = 400;
  return false;
}

// 分发请求给对应的处理函数
inline bool Server::dispatch_request(Request &req, Response &res,
                                     const Handlers &handlers) {
  // 遍历处理函数集合
  for (const auto &x : handlers) {
    // 获取请求路径的正则表达式模式和对应的处理函数
    const auto &pattern = x.first;
    const auto &handler = x.second;

    // 如果请求路径匹配正则表达式模式
    if (std::regex_match(req.path, req.matches, pattern)) {
      // 调用对应的处理函数处理请求，并返回 true
      handler(req, res);
      return true;
    }
  }
  // 如果没有匹配的处理函数，则返回 false
  return false;
}

// 应用范围请求
inline void Server::apply_ranges(const Request &req, Response &res,
                                 std::string &content_type,
                                 std::string &boundary) {
  // 如果请求中包含多个范围
  if (req.ranges.size() > 1) {
    // 生成多部分数据的边界
    boundary = detail::make_multipart_data_boundary();

    // 查找响应头中的 Content-Type
    auto it = res.headers.find("Content-Type");
    // 如果找到了 Content-Type
    if (it != res.headers.end()) {
      // 保存原始的 Content-Type
      content_type = it->second;
      // 删除原始的 Content-Type
      res.headers.erase(it);
    }

    // 添加新的 Content-Type，指定为多部分范围请求，包含边界信息
    res.headers.emplace("Content-Type",
                        "multipart/byteranges; boundary=" + boundary);
  }

  // 获取编码类型
  auto type = detail::encoding_type(req, res);

  // 如果响应体为空
  if (res.body.empty()) {
    // 如果响应体长度大于 0
    if (res.content_length_ > 0) {
      size_t length = 0;
      // 如果请求中不包含范围
      if (req.ranges.empty()) {
        length = res.content_length_;
      } 
      // 如果请求中只包含一个范围
      else if (req.ranges.size() == 1) {
        // 获取范围的偏移量和长度
        auto offsets =
            detail::get_range_offset_and_length(req, res.content_length_, 0);
        auto offset = offsets.first;
        length = offsets.second;
        // 生成 Content-Range 头字段
        auto content_range = detail::make_content_range_header_field(
            offset, length, res.content_length_);
        res.set_header("Content-Range", content_range);
      } 
      // 如果请求中包含多个范围
      else {
        // 获取多部分范围数据的长度
        length = detail::get_multipart_ranges_data_length(req, res, boundary,
                                                          content_type);
      }
      // 设置 Content-Length 头字段
      res.set_header("Content-Length", std::to_string(length));
    } else {
      // 如果响应有内容提供者
      if (res.content_provider_) {
        // 如果响应的内容提供者是分块的
        if (res.is_chunked_content_provider_) {
          // 设置响应头部，表明内容是分块传输
          res.set_header("Transfer-Encoding", "chunked");
          // 如果编码类型是 Gzip，则设置内容编码为 gzip
          if (type == detail::EncodingType::Gzip) {
            res.set_header("Content-Encoding", "gzip");
          } 
          // 如果编码类型是 Brotli，则设置内容编码为 br
          else if (type == detail::EncodingType::Brotli) {
            res.set_header("Content-Encoding", "br");
          }
        }
      }
    }
  } else {
    // 如果请求的范围为空
    if (req.ranges.empty()) {
      // 空语句，不执行任何操作
      ;
    } 
    // 如果请求的范围大小为1
    else if (req.ranges.size() == 1) {
      // 获取范围的偏移和长度
      auto offsets =
          detail::get_range_offset_and_length(req, res.body.size(), 0);
      auto offset = offsets.first;
      auto length = offsets.second;
      // 生成内容范围头部字段
      auto content_range = detail::make_content_range_header_field(
          offset, length, res.body.size());
      res.set_header("Content-Range", content_range);
      // 如果偏移小于响应体的大小，则截取响应体
      if (offset < res.body.size()) {
        res.body = res.body.substr(offset, length);
      } else {
        // 否则清空响应体，设置状态为416
        res.body.clear();
        res.status = 416;
      }
    } 
    // 如果请求的范围大小大于1
    else {
      std::string data;
      // 生成多部分范围数据
      if (detail::make_multipart_ranges_data(req, res, boundary, content_type,
                                             data)) {
        res.body.swap(data);
      } else {
        // 否则清空响应体，设置状态为416
        res.body.clear();
        res.status = 416;
      }
    }

    // 如果编码类型不是 None
    if (type != detail::EncodingType::None) {
      std::unique_ptr<detail::compressor> compressor;
      std::string content_encoding;

      // 如果编码类型是 Gzip
      if (type == detail::EncodingType::Gzip) {
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
        // 如果支持 ZLIB 压缩，则创建 gzip 压缩器，并设置内容编码为 gzip
        compressor = detail::make_unique<detail::gzip_compressor>();
        content_encoding = "gzip";
#endif
      } else if (type == detail::EncodingType::Brotli) {
#ifdef CPPHTTPLIB_BROTLI_SUPPORT
        // 如果支持 Brotli 压缩，则创建 Brotli 压缩器，并设置内容编码为 br
        compressor = detail::make_unique<detail::brotli_compressor>();
        content_encoding = "br";
#endif
      }

      if (compressor) {
        // 如果存在压缩器，则进行压缩操作
        std::string compressed;
        if (compressor->compress(res.body.data(), res.body.size(), true,
                                 [&](const char *data, size_t data_len) {
                                   compressed.append(data, data_len);
                                   return true;
                                 })) {
          // 将压缩后的数据替换原始数据，并设置内容编码头部
          res.body.swap(compressed);
          res.set_header("Content-Encoding", content_encoding);
        }
      }
    }

    // 计算压缩后的内容长度，并设置内容长度头部
    auto length = std::to_string(res.body.size());
    res.set_header("Content-Length", length);
  }
}

// 根据内容读取器分发请求
inline bool Server::dispatch_request_for_content_reader(
    Request &req, Response &res, ContentReader content_reader,
    const HandlersForContentReader &handlers) {
  for (const auto &x : handlers) {
    const auto &pattern = x.first;
    const auto &handler = x.second;

    // 使用正则表达式匹配请求路径，找到对应的处理器并处理请求
    if (std::regex_match(req.path, req.matches, pattern)) {
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
  std::array<char, 2048> buf{};

  detail::stream_line_reader line_reader(strm, buf.data(), buf.size());

  // 如果客户端关闭连接，则返回 false
  if (!line_reader.getline()) { return false; }

  Request req;
  Response res;

  res.version = "HTTP/1.1";

  // 添加默认头部到响应头部中
  for (const auto &header : default_headers_) {
    if (res.headers.find(header.first) == res.headers.end()) {
      res.headers.insert(header);
    }
  }
#ifdef _WIN32
  // 如果是在 Windows 平台，需要增加 FD_SETSIZE 的大小（libzmq），或者动态增加（MySQL）。
#else
#ifndef CPPHTTPLIB_USE_POLL
  // 如果不是在 Windows 平台，并且没有启用 CPPHTTPLIB_USE_POLL 宏
  // 检查套接字文件描述符是否超过了 FD_SETSIZE 的限制
  if (strm.socket() >= FD_SETSIZE) {
    // 读取并丢弃请求头部
    Headers dummy;
    detail::read_headers(strm, dummy);
    // 设置响应状态码为 500
    res.status = 500;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }
#endif
#endif

  // 检查请求 URI 是否超过了最大长度限制
  if (line_reader.size() > CPPHTTPLIB_REQUEST_URI_MAX_LENGTH) {
    // 读取并丢弃请求头部
    Headers dummy;
    detail::read_headers(strm, dummy);
    // 设置响应状态码为 414
    res.status = 414;
    // 返回响应
    return write_response(strm, close_connection, req, res);
  }

  // 解析请求行和请求头部
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

  // 检查是否是 HTTP/1.0 协议，并且不是 Keep-Alive 连接
  if (req.version == "HTTP/1.0" &&
      req.get_header_value("Connection") != "Keep-Alive") {
    connection_closed = true;
  }

  // 获取远程 IP 和端口，并设置请求头部
  strm.get_remote_ip_and_port(req.remote_addr, req.remote_port);
  req.set_header("REMOTE_ADDR", req.remote_addr);
  req.set_header("REMOTE_PORT", std::to_string(req.remote_port));

  // 获取本地 IP 和端口，并设置请求头部
  strm.get_local_ip_and_port(req.local_addr, req.local_port);
  req.set_header("LOCAL_ADDR", req.local_addr);
  req.set_header("LOCAL_PORT", std::to_string(req.local_port));

  // 如果请求头部包含 Range 字段
  if (req.has_header("Range")) {
    // 解析 Range 字段的值
    const auto &range_header_value = req.get_header_value("Range");
    if (!detail::parse_range_header(range_header_value, req.ranges)) {
      // 设置响应状态码为 416
      res.status = 416;
      // 返回响应
      return write_response(strm, close_connection, req, res);
    }
  }

  // 如果设置了自定义的请求处理函数，则调用该函数
  if (setup_request) { setup_request(req); }

  // 如果请求头部包含 Expect 字段，并且值为 "100-continue"
  if (req.get_header_value("Expect") == "100-continue") {
    // 默认设置响应状态码为 100
    auto status = 100;
    // 如果设置了自定义的 100 Continue 处理函数，则调用该函数
    if (expect_100_continue_handler_) {
      status = expect_100_continue_handler_(req, res);
    }
    // 根据处理函数的返回状态码进行处理
    switch (status) {
    case 100:
    # 根据状态码和状态消息格式化输出HTTP响应头部
    case 417:
      strm.write_format("HTTP/1.1 %d %s\r\n\r\n", status,
                        detail::status_message(status));
      # 结束当前的case分支
      break;
    # 默认情况下调用write_response函数处理响应
    default: return write_response(strm, close_connection, req, res);
    }
  }

  # 路由标记，用于标识是否已经进行了路由处理
  bool routed = false;
#ifdef CPPHTTPLIB_NO_EXCEPTIONS
  // 如果 CPPHTTPLIB_NO_EXCEPTIONS 宏被定义，则直接调用 routing 函数处理请求
  routed = routing(req, res, strm);
#else
  // 否则，尝试调用 routing 函数处理请求，捕获可能抛出的异常
  try {
    routed = routing(req, res, strm);
  } catch (std::exception &e) {
    // 如果捕获到异常，并且存在异常处理函数，则调用异常处理函数处理异常
    if (exception_handler_) {
      auto ep = std::current_exception();
      exception_handler_(req, res, ep);
      routed = true;
    } else {
      // 否则，设置响应状态码为 500，并将异常信息转义后存储在响应头中
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
    // 捕获其他类型的异常，同样调用异常处理函数处理异常，或者设置响应状态码为 500，并在响应头中标记异常为 UNKNOWN
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

  // 如果请求已经被处理
  if (routed) {
    // 如果响应状态码为 -1，则根据请求是否包含范围来设置响应状态码为 200 或 206，并返回带有内容的响应
    if (res.status == -1) { res.status = req.ranges.empty() ? 200 : 206; }
    return write_response_with_content(strm, close_connection, req, res);
  } else {
    // 如果请求未被处理，设置响应状态码为 404，并返回响应
    if (res.status == -1) { res.status = 404; }
    return write_response(strm, close_connection, req, res);
  }
}

// 检查服务器是否有效
inline bool Server::is_valid() const { return true; }

// 处理并关闭套接字
inline bool Server::process_and_close_socket(socket_t sock) {
  // 调用 detail::process_server_socket 处理服务器套接字，并关闭套接字
  auto ret = detail::process_server_socket(
      svr_sock_, sock, keep_alive_max_count_, keep_alive_timeout_sec_,
      read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
      write_timeout_usec_,
      [this](Stream &strm, bool close_connection, bool &connection_closed) {
        return process_request(strm, close_connection, connection_closed,
                               nullptr);
      });

  detail::shutdown_socket(sock);
  detail::close_socket(sock);
  return ret;
}

// HTTP 客户端实现
// 使用默认端口号 80 创建客户端实例
inline ClientImpl::ClientImpl(const std::string &host)
    : ClientImpl(host, 80, std::string(), std::string()) {}

// 使用指定端口号创建客户端实例
inline ClientImpl::ClientImpl(const std::string &host, int port)
    # 调用 ClientImpl 构造函数，传入参数 host, port, 空字符串, 空字符串
// 客户端实现的构造函数，初始化成员变量
inline ClientImpl::ClientImpl(const std::string &host, int port,
                              const std::string &client_cert_path,
                              const std::string &client_key_path)
    : host_(host), port_(port),
      host_and_port_(adjust_host_string(host) + ":" + std::to_string(port)),
      client_cert_path_(client_cert_path), client_key_path_(client_key_path) {}

// 客户端实现的析构函数，关闭套接字并释放资源
inline ClientImpl::~ClientImpl() {
  // 使用互斥锁保护套接字，防止并发访问
  std::lock_guard<std::mutex> guard(socket_mutex_);
  // 关闭套接字的写入功能
  shutdown_socket(socket_);
  // 关闭套接字
  close_socket(socket_);
}

// 检查客户端实例是否有效
inline bool ClientImpl::is_valid() const { return true; }

// 复制另一个客户端实例的设置
inline void ClientImpl::copy_settings(const ClientImpl &rhs) {
  // 复制证书路径
  client_cert_path_ = rhs.client_cert_path_;
  // 复制密钥路径
  client_key_path_ = rhs.client_key_path_;
  // 复制连接超时时间
  connection_timeout_sec_ = rhs.connection_timeout_sec_;
  // 复制读取超时时间
  read_timeout_sec_ = rhs.read_timeout_sec_;
  // 复制读取超时微秒
  read_timeout_usec_ = rhs.read_timeout_usec_;
  // 复制写入超时时间
  write_timeout_sec_ = rhs.write_timeout_sec_;
  // 复制写入超时微秒
  write_timeout_usec_ = rhs.write_timeout_usec_;
  // 复制基本认证用户名
  basic_auth_username_ = rhs.basic_auth_username_;
  // 复制基本认证密码
  basic_auth_password_ = rhs.basic_auth_password_;
  // 复制令牌认证令牌
  bearer_token_auth_token_ = rhs.bearer_token_auth_token_;
  // 如果支持 OpenSSL，复制摘要认证用户名和密码
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  digest_auth_username_ = rhs.digest_auth_username_;
  digest_auth_password_ = rhs.digest_auth_password_;
#endif
  // 复制保持连接状态
  keep_alive_ = rhs.keep_alive_;
  // 复制跟随重定向
  follow_location_ = rhs.follow_location_;
  // 复制 URL 编码
  url_encode_ = rhs.url_encode_;
  // 复制地址族
  address_family_ = rhs.address_family_;
  // 复制 TCP 禁用 Nagle 算法
  tcp_nodelay_ = rhs.tcp_nodelay_;
  // 复制套接字选项
  socket_options_ = rhs.socket_options_;
  // 复制压缩选项
  compress_ = rhs.compress_;
  // 复制解压选项
  decompress_ = rhs.decompress_;
  // 复制接口
  interface_ = rhs.interface_;
  // 复制代理主机
  proxy_host_ = rhs.proxy_host_;
  // 复制代理端口
  proxy_port_ = rhs.proxy_port_;
  // 复制代理基本认证用户名
  proxy_basic_auth_username_ = rhs.proxy_basic_auth_username_;
  // 复制代理基本认证密码
  proxy_basic_auth_password_ = rhs.proxy_basic_auth_password_;
  // 复制代理令牌认证令牌
  proxy_bearer_token_auth_token_ = rhs.proxy_bearer_token_auth_token_;
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则将 rhs 的代理服务器摘要认证用户名赋值给当前对象
  proxy_digest_auth_username_ = rhs.proxy_digest_auth_username_;
  // 如果支持 OpenSSL，则将 rhs 的代理服务器摘要认证密码赋值给当前对象
  proxy_digest_auth_password_ = rhs.proxy_digest_auth_password_;
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则将 rhs 的 CA 证书文件路径赋值给当前对象
  ca_cert_file_path_ = rhs.ca_cert_file_path_;
  // 如果支持 OpenSSL，则将 rhs 的 CA 证书目录路径赋值给当前对象
  ca_cert_dir_path_ = rhs.ca_cert_dir_path_;
  // 如果支持 OpenSSL，则将 rhs 的 CA 证书存储对象赋值给当前对象
  ca_cert_store_ = rhs.ca_cert_store_;
#endif
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  // 如果支持 OpenSSL，则将 rhs 的服务器证书验证选项赋值给当前对象
  server_certificate_verification_ = rhs.server_certificate_verification_;
#endif
  // 将 rhs 的日志对象赋值给当前对象
  logger_ = rhs.logger_;
}

// 创建客户端套接字
inline socket_t ClientImpl::create_client_socket(Error &error) const {
  // 如果代理主机和代理端口不为空，则创建代理客户端套接字
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
  // 将套接字赋值给 socket 对象
  socket.sock = sock;
  // 返回 true
  return true;
}
// 关闭 SSL 连接
inline void ClientImpl::shutdown_ssl(Socket & /*socket*/,
                                     bool /*shutdown_gracefully*/) {
  // 如果有来自其他线程的请求正在进行，则存在线程不安全的竞争条件，因为单独的 ssl* 对象不是线程安全的
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());
}

// 关闭套接字
inline void ClientImpl::shutdown_socket(Socket &socket) {
  if (socket.sock == INVALID_SOCKET) { return; }
  detail::shutdown_socket(socket.sock);
}

// 关闭套接字
inline void ClientImpl::close_socket(Socket &socket) {
  // 如果有来自其他线程的请求正在进行，则关闭套接字通常是可以的，它们只会在使用关闭的套接字时收到错误，但这仍然是一个 bug，因为很少情况下操作系统可能会重新分配套接字 ID 用于新的套接字，然后它们将突然操作一个与他们预期的不同的活动套接字！
  assert(socket_requests_in_flight_ == 0 ||
         socket_requests_are_from_thread_ == std::this_thread::get_id());

  // 如果在 SSL 仍然活动时发生这种情况也是一个 bug
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
  // 定义正则表达式，匹配 HTTP 响应行
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r\n");
#else
  const static std::regex re("(HTTP/1\\.[01]) (\\d{3})(?: (.*?))?\r?\n");
#endif

  std::cmatch m;
  // 使用正则表达式匹配响应行
  if (!std::regex_match(line_reader.ptr(), m, re)) {
    // 检查请求方法是否为 CONNECT
    return req.method == "CONNECT";
  }
  // 设置响应的 HTTP 版本
  res.version = std::string(m[1]);
  // 设置响应的状态码
  res.status = std::stoi(std::string(m[2]));
  // 设置响应的状态原因
  res.reason = std::string(m[3]);

  // 忽略 '100 Continue' 响应
  while (res.status == 100) {
    // 读取空行
    if (!line_reader.getline()) { return false; } // CRLF
    // 读取下一行响应
    if (!line_reader.getline()) { return false; } // next response line

    // 使用正则表达式匹配响应行
    if (!std::regex_match(line_reader.ptr(), m, re)) { return false; }
    // 更新响应的 HTTP 版本
    res.version = std::string(m[1]);
    // 更新响应的状态码
    res.status = std::stoi(std::string(m[2]));
    // 更新响应的状态原因
    res.reason = std::string(m[3]);
  }

  // 返回 true 表示解析成功
  return true;
}
// 客户端发送请求的内部函数，接收请求、响应和错误对象的引用
inline bool ClientImpl::send(Request &req, Response &res, Error &error) {
  // 使用递归互斥锁保护请求
  std::lock_guard<std::recursive_mutex> request_mutex_guard(request_mutex_);
  // 调用内部发送函数
  auto ret = send_(req, res, error);
  // 如果出现 SSLPeerCouldBeClosed_ 错误，重新发送请求
  if (error == Error::SSLPeerCouldBeClosed_) {
    assert(!ret);
    ret = send_(req, res, error);
  }
  return ret;
}

// 实际发送请求的内部函数
inline bool ClientImpl::send_(Request &req, Response &res, Error &error) {
  {
    // 使用互斥锁保护套接字
    std::lock_guard<std::mutex> guard(socket_mutex_);

    // 立即将 socket_should_be_closed_when_request_is_done_ 设置为 false
    // 如果在请求结束时它被设置为 true，我们知道另一个线程指示我们关闭套接字
    socket_should_be_closed_when_request_is_done_ = false;

    auto is_alive = false;
    if (socket_.is_open()) {
      // 检查套接字是否存活
      is_alive = detail::is_socket_alive(socket_.sock);
      if (!is_alive) {
        // 尝试避免 sigpipe，如果似乎对方已经关闭连接，则非优雅地关闭
        // 由于我们锁定了 request_mutex_，所以不可能有其他线程中的请求在进行中，因此可以立即关闭一切
        const bool shutdown_gracefully = false;
        shutdown_ssl(socket_, shutdown_gracefully);
        shutdown_socket(socket_);
        close_socket(socket_);
      }
    }

    if (!is_alive) {
      // 如果套接字不存活，创建并连接套接字
      if (!create_and_connect_socket(socket_, error)) { return false; }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 如果支持 SSL
      if (is_ssl()) {
        auto &scli = static_cast<SSLClient &>(*this);
        // 如果使用代理，通过代理连接套接字
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

    // 标记当前套接字正在使用，以防在请求进行中时被其他线程关闭
    // 如果当前有多个 socket 请求在处理，则断言这些请求都来自同一个线程
    if (socket_requests_in_flight_ > 1) {
      assert(socket_requests_are_from_thread_ == std::this_thread::get_id());
    }
    // 增加正在处理的 socket 请求数量
    socket_requests_in_flight_ += 1;
    // 记录当前处理请求的线程 ID
    socket_requests_are_from_thread_ = std::this_thread::get_id();
  }

  // 遍历默认请求头，如果请求中没有该头部信息，则插入默认的头部信息
  for (const auto &header : default_headers_) {
    if (req.headers.find(header.first) == req.headers.end()) {
      req.headers.insert(header);
    }
  }

  // 初始化返回值为 false，初始化是否关闭连接为 !keep_alive_
  auto ret = false;
  auto close_connection = !keep_alive_;

  // 创建一个 scope_exit 对象，用于在作用域结束时执行特定的操作
  auto se = detail::scope_exit([&]() {
    // 使用互斥锁锁定，标记请求不再进行中
    std::lock_guard<std::mutex> guard(socket_mutex_);
    // 减少正在处理的 socket 请求数量
    socket_requests_in_flight_ -= 1;
    // 如果没有正在处理的请求，则重置请求线程 ID
    if (socket_requests_in_flight_ <= 0) {
      assert(socket_requests_in_flight_ == 0);
      socket_requests_are_from_thread_ = std::thread::id();
    }
    // 如果应该在请求完成时关闭连接，或者需要关闭连接，或者处理失败，则关闭 SSL、关闭 socket、关闭连接
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

  // 如果处理失败且错误类型为 Success，则将错误类型设置为 Unknown
  if (!ret) {
    if (error == Error::Success) { error = Error::Unknown; }
  }

  // 返回处理结果
  return ret;
}

// 发送请求的内部实现，接收一个请求对象并发送
inline Result ClientImpl::send(const Request &req) {
  // 复制请求对象
  auto req2 = req;
  // 调用 send_ 函数发送请求
  return send_(std::move(req2));
}

// 发送请求的实际实现，接收一个移动语义的请求对象
inline Result ClientImpl::send_(Request &&req) {
  // 创建一个响应对象的智能指针
  auto res = detail::make_unique<Response>();
  // 初始化错误状态为成功
  auto error = Error::Success;
  // 发送请求并接收返回值
  auto ret = send(req, *res, error);
  // 根据返回值创建结果对象
  return Result{ret ? std::move(res) : nullptr, error, std::move(req.headers)};
}

// 处理请求的函数，接收一个流对象、请求对象、响应对象、连接关闭标志和错误状态
inline bool ClientImpl::handle_request(Stream &strm, Request &req,
                                       Response &res, bool close_connection,
                                       Error &error) {
  // 如果请求路径为空，设置错误状态为连接错误并返回 false
  if (req.path.empty()) {
    error = Error::Connection;
    return false;
  }

  // 保存原始请求对象
  auto req_save = req;

  bool ret;

  // 如果不是 SSL 并且代理主机和端口不为空，则处理代理请求
  if (!is_ssl() && !proxy_host_.empty() && proxy_port_ != -1) {
    // 复制请求对象
    auto req2 = req;
    // 修改请求路径为代理路径
    req2.path = "http://" + host_and_port_ + req.path;
    // 处理代理请求
    ret = process_request(strm, req2, res, close_connection, error);
    // 更新请求对象
    req = req2;
    req.path = req_save.path;
  } else {
    // 处理普通请求
    ret = process_request(strm, req, res, close_connection, error);
  }

  // 如果处理请求失败，直接返回 false
  if (!ret) { return false; }

  // 如果响应状态码为 3xx 且启用重定向，则处理重定向
  if (300 < res.status && res.status < 400 && follow_location_) {
    req = req_save;
    ret = redirect(req, res, error);
  }

  // 如果支持 OpenSSL 并且状态码为 401 或 407 且授权次数小于 5，则处理摘要认证
  #ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  if ((res.status == 401 || res.status == 407) &&
      req.authorization_count_ < 5) {
    auto is_proxy = res.status == 407;
    const auto &username =
        is_proxy ? proxy_digest_auth_username_ : digest_auth_username_;
    const auto &password =
        is_proxy ? proxy_digest_auth_password_ : digest_auth_password_;
  }
  // ...
  // 其他代码
  // ...
}
    // 如果用户名和密码都不为空
    if (!username.empty() && !password.empty()) {
      // 创建一个空的字符串到字符串的映射
      std::map<std::string, std::string> auth;
      // 解析响应中的身份验证信息，并将结果存储在auth中
      if (detail::parse_www_authenticate(res, auth, is_proxy)) {
        // 复制原始请求
        Request new_req = req;
        // 增加授权计数
        new_req.authorization_count_ += 1;
        // 删除原始请求中的代理授权或授权头
        new_req.headers.erase(is_proxy ? "Proxy-Authorization"
                                       : "Authorization");
        // 插入新的摘要认证头
        new_req.headers.insert(detail::make_digest_authentication_header(
            req, auth, new_req.authorization_count_, detail::random_string(10),
            username, password, is_proxy));

        // 创建新的响应对象
        Response new_res;

        // 发送新的请求，并将结果存储在new_res中
        ret = send(new_req, new_res, error);
        // 如果发送成功，将new_res赋值给res
        if (ret) { res = new_res; }
      }
    }
  }
#endif

  return ret;
}

// 重定向函数，处理重定向逻辑
inline bool ClientImpl::redirect(Request &req, Response &res, Error &error) {
  // 如果重定向次数为0，返回错误
  if (req.redirect_count_ == 0) {
    error = Error::ExceedRedirectCount;
    return false;
  }

  // 获取重定向地址
  auto location = res.get_header_value("location");
  if (location.empty()) { return false; }

  // 定义 URL 正则表达式
  const static std::regex re(
      R"((?:(https?):)?(?://(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)?([^?#]*)(\?[^#]*)?(?:#.*)?)");

  std::smatch m;
  // 使用正则表达式匹配重定向地址
  if (!std::regex_match(location, m, re)) { return false; }

  // 获取当前协议
  auto scheme = is_ssl() ? "https" : "http";

  // 解析重定向地址的各个部分
  auto next_scheme = m[1].str();
  auto next_host = m[2].str();
  if (next_host.empty()) { next_host = m[3].str(); }
  auto port_str = m[4].str();
  auto next_path = m[5].str();
  auto next_query = m[6].str();

  // 解析重定向地址的端口
  auto next_port = port_;
  if (!port_str.empty()) {
    next_port = std::stoi(port_str);
  } else if (!next_scheme.empty()) {
    next_port = next_scheme == "https" ? 443 : 80;
  }

  // 处理重定向地址的协议、主机和路径
  if (next_scheme.empty()) { next_scheme = scheme; }
  if (next_host.empty()) { next_host = host_; }
  if (next_path.empty()) { next_path = "/"; }

  // 解码 URL，并拼接查询参数
  auto path = detail::decode_url(next_path, true) + next_query;

  // 如果重定向地址和当前地址相同，则执行重定向
  if (next_scheme == scheme && next_host == host_ && next_port == port_) {
    return detail::redirect(*this, req, res, path, location, error);
  } else {
    // 如果是 https 协议
    if (next_scheme == "https") {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 创建 SSL 客户端对象，执行重定向
      SSLClient cli(next_host.c_str(), next_port);
      cli.copy_settings(*this);
      if (ca_cert_store_) { cli.set_ca_cert_store(ca_cert_store_); }
      return detail::redirect(cli, req, res, path, location, error);
#else
      return false;
#endif
    } else {
      // 创建普通客户端对象，执行重定向
      ClientImpl cli(next_host.c_str(), next_port);
      cli.copy_settings(*this);
      return detail::redirect(cli, req, res, path, location, error);
    }
  }
}
inline bool ClientImpl::write_content_with_provider(Stream &strm,
                                                    const Request &req,
                                                    Error &error) {
  // 定义一个闭包函数，用于检查是否正在关闭连接
  auto is_shutting_down = []() { return false; };

  // 如果请求使用分块内容提供程序
  if (req.is_chunked_content_provider_) {
    // TODO: Brotli support
    // 创建一个压缩器对象
    std::unique_ptr<detail::compressor> compressor;
#ifdef CPPHTTPLIB_ZLIB_SUPPORT
    // 如果支持压缩，使用 gzip 压缩器
    if (compress_) {
      compressor = detail::make_unique<detail::gzip_compressor>();
    } else
#endif
    {
      // 否则使用无压缩器
      compressor = detail::make_unique<detail::nocompressor>();
    }

    // 写入分块内容
    return detail::write_content_chunked(strm, req.content_provider_,
                                         is_shutting_down, *compressor, error);
  } else {
    // 否则写入普通内容
    return detail::write_content(strm, req.content_provider_, 0,
                                 req.content_length_, is_shutting_down, error);
  }
}

inline bool ClientImpl::write_request(Stream &strm, Request &req,
                                      bool close_connection, Error &error) {
  // 准备额外的请求头部
  if (close_connection) {
    // 如果需要关闭连接，并且请求中没有 Connection 头部，则添加 Connection: close 头部
    if (!req.has_header("Connection")) {
      req.headers.emplace("Connection", "close");
    }
  }

  if (!req.has_header("Host")) {
    // 如果请求中没有 Host 头部
    if (is_ssl()) {
      // 如果是 SSL 连接
      if (port_ == 443) {
        // 如果端口是 443，则 Host 头部为 host_
        req.headers.emplace("Host", host_);
      } else {
        // 否则为 host_and_port_
        req.headers.emplace("Host", host_and_port_);
      }
    } else {
      // 如果不是 SSL 连接
      if (port_ == 80) {
        // 如果端口是 80，则 Host 头部为 host_
        req.headers.emplace("Host", host_);
      } else {
        // 否则为 host_and_port_
        req.headers.emplace("Host", host_and_port_);
      }
    }
  }

  if (!req.has_header("Accept")) { 
    // 如果请求中没有 Accept 头部，则添加 Accept: */* 头部
    req.headers.emplace("Accept", "*/*"); 
  }

#ifndef CPPHTTPLIB_NO_DEFAULT_USER_AGENT
  if (!req.has_header("User-Agent")) {
    // 如果请求中没有 User-Agent 头部，则添加 cpp-httplib 版本信息
    auto agent = std::string("cpp-httplib/") + CPPHTTPLIB_VERSION;
    req.headers.emplace("User-Agent", agent);
  }
#endif

  if (req.body.empty()) {
    // 如果请求体为空，则设置 Content-Length 头部为 0
    // 如果请求有内容提供者
    if (req.content_provider_) {
      // 如果不是分块内容提供者
      if (!req.is_chunked_content_provider_) {
        // 如果请求头中没有包含"Content-Length"
        if (!req.has_header("Content-Length")) {
          // 将请求内容的长度转换为字符串，并添加到请求头中
          auto length = std::to_string(req.content_length_);
          req.headers.emplace("Content-Length", length);
        }
      }
    } else {
      // 如果请求没有内容提供者
      if (req.method == "POST" || req.method == "PUT" ||
          req.method == "PATCH") {
        // 对于POST、PUT、PATCH方法，添加"Content-Length"为"0"到请求头中
        req.headers.emplace("Content-Length", "0");
      }
    }
  } else {
    // 如果请求头中没有包含"Content-Type"
    if (!req.has_header("Content-Type")) {
      // 添加"Content-Type"为"text/plain"到请求头中
      req.headers.emplace("Content-Type", "text/plain");
    }

    // 如果请求头中没有包含"Content-Length"
    if (!req.has_header("Content-Length")) {
      // 将请求体的长度转换为字符串，并添加到请求头中
      auto length = std::to_string(req.body.size());
      req.headers.emplace("Content-Length", length);
    }
  }

  // 如果有基本身份验证的用户名和密码
  if (!basic_auth_password_.empty() || !basic_auth_username_.empty()) {
    // 如果请求头中没有包含"Authorization"
    if (!req.has_header("Authorization")) {
      // 插入基本身份验证的请求头
      req.headers.insert(make_basic_authentication_header(
          basic_auth_username_, basic_auth_password_, false));
    }
  }

  // 如果有代理基本身份验证的用户名和密码
  if (!proxy_basic_auth_username_.empty() &&
      !proxy_basic_auth_password_.empty()) {
    // 如果请求头中没有包含"Proxy-Authorization"
    if (!req.has_header("Proxy-Authorization")) {
      // 插入代理基本身份验证的请求头
      req.headers.insert(make_basic_authentication_header(
          proxy_basic_auth_username_, proxy_basic_auth_password_, true));
    }
  }

  // 如果有Bearer令牌身份验证的令牌
  if (!bearer_token_auth_token_.empty()) {
    // 如果请求头中没有包含"Authorization"
    if (!req.has_header("Authorization")) {
      // 插入Bearer令牌身份验证的请求头
      req.headers.insert(make_bearer_token_authentication_header(
          bearer_token_auth_token_, false));
    }
  }

  // 如果有代理Bearer令牌身份验证的令牌
  if (!proxy_bearer_token_auth_token_.empty()) {
    // 如果请求头中没有包含"Proxy-Authorization"
    if (!req.has_header("Proxy-Authorization")) {
      // 插入代理Bearer令牌身份验证的请求头
      req.headers.insert(make_bearer_token_authentication_header(
          proxy_bearer_token_auth_token_, true));
    }
  }

  // 请求行和请求头
  {
    // 创建缓冲流对象
    detail::BufferStream bstrm;

    // 对路径进行URL编码（如果需要），并将请求方法和路径写入缓冲流
    const auto &path = url_encode_ ? detail::encode_url(req.path) : req.path;
    bstrm.write_format("%s %s HTTP/1.1\r\n", req.method.c_str(), path.c_str());

    // 将请求头写入缓冲流
    detail::write_headers(bstrm, req.headers);

    // 刷新缓冲流
    // 获取缓冲区的引用
    auto &data = bstrm.get_buffer();
    // 如果无法将数据写入流，则设置错误类型并返回false
    if (!detail::write_data(strm, data.data(), data.size())) {
      error = Error::Write;
      return false;
    }
  }

  // Body
  // 如果请求体为空，则使用提供程序写入内容并返回结果
  if (req.body.empty()) {
    return write_content_with_provider(strm, req, error);
  }

  // 如果无法将请求体数据写入流，则设置错误类型并返回false
  if (!detail::write_data(strm, req.body.data(), req.body.size())) {
    error = Error::Write;
    return false;
  }

  // 返回true
  return true;
  // 使用内容提供者发送请求，并返回响应的唯一指针
  inline std::unique_ptr<Response> ClientImpl::send_with_content_provider(
      // 请求对象，请求体，请求体长度，内容提供者，无长度的内容提供者，内容类型，错误对象
      Request &req, const char *body, size_t content_length,
      ContentProvider content_provider,
      ContentProviderWithoutLength content_provider_without_length,
      const std::string &content_type, Error &error) {
    // 如果内容类型不为空，则将内容类型添加到请求头中
    if (!content_type.empty()) {
      req.headers.emplace("Content-Type", content_type);
    }

    // 如果启用了压缩，则将内容编码设置为 gzip
    #ifdef CPPHTTPLIB_ZLIB_SUPPORT
    if (compress_) { req.headers.emplace("Content-Encoding", "gzip"); }
    #endif

    // 如果启用了压缩并且内容提供者没有长度
    #ifdef CPPHTTPLIB_ZLIB_SUPPORT
    if (compress_ && !content_provider_without_length) {
      // TODO: Brotli support
      // 创建 gzip 压缩器
      detail::gzip_compressor compressor;

      // 如果存在内容提供者
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

        // 循环调用内容提供者，直到所有数据都被写入
        while (ok && offset < content_length) {
          if (!content_provider(offset, content_length - offset, data_sink)) {
            error = Error::Canceled;
            return nullptr;
          }
        }
      } else {
        // 如果没有内容提供者，则直接压缩请求体并追加到请求体中
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
    # 如果存在内容提供者，则设置请求的内容长度和内容提供者，并标记为非分块内容提供者
    if (content_provider) {
      req.content_length_ = content_length;
      req.content_provider_ = std::move(content_provider);
      req.is_chunked_content_provider_ = false;
    } 
    # 如果存在没有长度的内容提供者，则设置请求的内容长度为0，设置内容提供者和标记为分块内容提供者，并添加传输编码为分块的头部信息
    else if (content_provider_without_length) {
      req.content_length_ = 0;
      req.content_provider_ = detail::ContentProviderAdapter(
          std::move(content_provider_without_length));
      req.is_chunked_content_provider_ = true;
      req.headers.emplace("Transfer-Encoding", "chunked");
    } 
    # 如果不存在内容提供者，则将请求的body设置为给定的内容和长度
    else {
      req.body.assign(body, content_length);
      ;
    }
  }
  # 创建一个新的响应对象
  auto res = detail::make_unique<Response>();
  # 发送请求，并根据返回结果决定是否返回响应对象或空指针
  return send(req, *res, error) ? std::move(res) : nullptr;
// 定义 ClientImpl 类的 send_with_content_provider 方法，用于发送带有内容提供者的请求
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

  // 发送带有内容提供者的请求
  auto res = send_with_content_provider(
      req, body, content_length, std::move(content_provider),
      std::move(content_provider_without_length), content_type, error);

  // 返回结果对象
  return Result{std::move(res), error, std::move(req.headers)};
}

// 定义 ClientImpl 类的 adjust_host_string 方法，用于调整主机字符串
inline std::string
ClientImpl::adjust_host_string(const std::string &host) const {
  // 如果主机字符串中包含冒号，则在主机字符串两端添加方括号
  if (host.find(':') != std::string::npos) { return "[" + host + "]"; }
  // 返回调整后的主机字符串
  return host;
}

// 定义 ClientImpl 类的 process_request 方法，用于处理请求
inline bool ClientImpl::process_request(Stream &strm, Request &req,
                                        Response &res, bool close_connection,
                                        Error &error) {
  // 发送请求
  if (!write_request(strm, req, close_connection, error)) { return false; }

  // 如果支持 SSL
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
  if (is_ssl()) {
    // 检查是否启用了代理
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

  // 接收响应和头部信息
  if (!read_response_line(strm, req, res) ||
      !detail::read_headers(strm, res.headers)) {
    error = Error::Read;
    return false;
  }

  // 处理响应体
  if ((res.status != 204) && req.method != "HEAD" && req.method != "CONNECT") {
    // 处理重定向
    auto redirect = 300 < res.status && res.status < 400 && follow_location_;
    // 如果存在响应处理函数并且没有重定向
    if (req.response_handler && !redirect) {
        // 如果响应处理函数返回 false，则设置错误为 Error::Canceled，返回 false
        if (!req.response_handler(res)) {
            error = Error::Canceled;
            return false;
        }
    }

    // 根据是否存在内容接收器，创建不同的 out 函数对象
    auto out =
        req.content_receiver
            ? static_cast<ContentReceiverWithProgress>(
                  // 如果有重定向，则直接返回 true；否则调用内容接收器函数，并根据返回值设置错误
                  [&](const char *buf, size_t n, uint64_t off, uint64_t len) {
                    if (redirect) { return true; }
                    auto ret = req.content_receiver(buf, n, off, len);
                    if (!ret) { error = Error::Canceled; }
                    return ret;
                  })
            : static_cast<ContentReceiverWithProgress>(
                  // 如果响应体大小超过最大限制，则返回 false；否则将数据追加到响应体中
                  [&](const char *buf, size_t n, uint64_t /*off*/,
                      uint64_t /*len*/) {
                    if (res.body.size() + n > res.body.max_size()) {
                      return false;
                    }
                    res.body.append(buf, n);
                    return true;
                  });

    // 进度回调函数，根据是否存在进度回调函数和重定向，进行相应的处理
    auto progress = [&](uint64_t current, uint64_t total) {
      if (!req.progress || redirect) { return true; }
      auto ret = req.progress(current, total);
      if (!ret) { error = Error::Canceled; }
      return ret;
    };

    // 读取内容，并根据返回值设置错误
    int dummy_status;
    if (!detail::read_content(strm, res, (std::numeric_limits<size_t>::max)(),
                              dummy_status, std::move(progress), std::move(out),
                              decompress_)) {
      if (error != Error::Canceled) { error = Error::Read; }
      return false;
    }
  }

  // 如果响应头中的 Connection 字段为 "close"，或者版本为 "HTTP/1.0" 且原因不是 "Connection established"
  if (res.get_header_value("Connection") == "close" ||
      (res.version == "HTTP/1.0" && res.reason != "Connection established")) {
    // TODO 需要一系列调用来正确执行，可能需要重构代码以使其更明显
    // 这里的调用是安全的，因为 process_request 只会被 send 函数调用，且互斥锁的递归性已经被移除
    // 可能需要重构代码以使其更明显
  }
    // 处理请求，只能被 send 调用，send 在整个处理过程中会锁定请求互斥锁。
    // 如果从不同的线程调用它，会导致线程安全问题，因为在另一个线程使用套接字时
    // 对套接字进行这些操作是有问题的。
    std::lock_guard<std::mutex> guard(socket_mutex_);  // 使用互斥锁保护临界区
    shutdown_ssl(socket_, true);  // 关闭 SSL 连接
    shutdown_socket(socket_);  // 关闭套接字
    close_socket(socket_);  // 关闭套接字

  }

  // 如果有日志记录器，则记录日志
  if (logger_) { logger_(req, res); }

  return true;  // 返回 true 表示处理成功
// 返回一个内联的内容提供程序，用于处理多部分表单数据
inline ContentProviderWithoutLength ClientImpl::get_multipart_content_provider(
    const std::string &boundary, const MultipartFormDataItems &items,
    const MultipartFormDataProviderItems &provider_items) {
  // 初始化当前项和当前起始位置
  size_t cur_item = 0, cur_start = 0;
  // cur_item 和 cur_start 被复制到 std::function 中，并在连续调用之间保持状态
  return [&, cur_item, cur_start](size_t offset,
                                  DataSink &sink) mutable -> bool {
    // 如果偏移量为0且items不为空，则序列化多部分表单数据并写入sink
    if (!offset && items.size()) {
      sink.os << detail::serialize_multipart_formdata(items, boundary, false);
      return true;
    } else if (cur_item < provider_items.size()) {
      // 如果当前项小于提供程序项的大小
      if (!cur_start) {
        // 序列化多部分表单数据项的开始部分并写入sink
        const auto &begin = detail::serialize_multipart_formdata_item_begin(
            provider_items[cur_item], boundary);
        offset += begin.size();
        cur_start = offset;
        sink.os << begin;
      }

      DataSink cur_sink;
      bool has_data = true;
      cur_sink.write = sink.write;
      cur_sink.done = [&]() { has_data = false; };

      // 如果提供程序项的提供程序返回false，则返回false
      if (!provider_items[cur_item].provider(offset - cur_start, cur_sink))
        return false;

      // 如果没有数据，则序列化多部分表单数据项的结束部分并更新当前项和当前起始位置
      if (!has_data) {
        sink.os << detail::serialize_multipart_formdata_item_end();
        cur_item++;
        cur_start = 0;
      }
      return true;
    } else {
      // 序列化多部分表单数据的结束部分并调用sink的done方法
      sink.os << detail::serialize_multipart_formdata_finish(boundary);
      sink.done();
      return true;
    }
  };
}

// 处理套接字的函数，调用detail::process_client_socket处理客户端套接字
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
# 使用给定路径和进度参数执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, Progress progress) {
  return Get(path, Headers(), std::move(progress));  # 调用下面的 Get 函数，传入路径、空的头部信息和进度参数
}

# 使用给定路径和头部信息执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers) {
  return Get(path, headers, Progress());  # 调用下面的 Get 函数，传入路径、头部信息和空的进度参数
}

# 使用给定路径、头部信息和进度参数执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              Progress progress) {
  Request req;  # 创建请求对象
  req.method = "GET";  # 设置请求方法为 GET
  req.path = path;  # 设置请求路径
  req.headers = headers;  # 设置请求头部信息
  req.progress = std::move(progress);  # 设置请求进度参数

  return send_(std::move(req));  # 调用 send_ 函数发送请求
}

# 使用给定路径和内容接收器执行 GET 请求
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver) {
  return Get(path, Headers(), nullptr, std::move(content_receiver), nullptr);  # 调用下面的 Get 函数，传入路径、空的头部信息、空的响应处理器、内容接收器和空的进度参数
}

# 使用给定路径、内容接收器和进度参数执行 GET 请求
inline Result ClientImpl::Get(const std::string &path,
                              ContentReceiver content_receiver,
                              Progress progress) {
  return Get(path, Headers(), nullptr, std::move(content_receiver),
             std::move(progress));  # 调用下面的 Get 函数，传入路径、空的头部信息、空的响应处理器、内容接收器和进度参数
}

# 使用给定路径、头部信息和内容接收器执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver) {
  return Get(path, headers, nullptr, std::move(content_receiver), nullptr);  # 调用下面的 Get 函数，传入路径、头部信息、空的响应处理器、内容接收器和空的进度参数
}

# 使用给定路径、头部信息、内容接收器和进度参数执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ContentReceiver content_receiver,
                              Progress progress) {
  return Get(path, headers, nullptr, std::move(content_receiver),
             std::move(progress));  # 调用下面的 Get 函数，传入路径、头部信息、空的响应处理器、内容接收器和进度参数
}

# 使用给定路径、响应处理器和内容接收器执行 GET 请求
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), nullptr);  # 调用下面的 Get 函数，传入路径、空的头部信息、响应处理器、内容接收器和空的进度参数
}
// 使用给定的路径、请求头、响应处理器和内容接收器发送 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver) {
  // 调用重载的 Get 函数，传入参数并移动响应处理器和内容接收器
  return Get(path, headers, std::move(response_handler),
             std::move(content_receiver), nullptr);
}

// 使用给定的路径、响应处理器、内容接收器和进度发送 GET 请求
inline Result ClientImpl::Get(const std::string &path,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  // 调用重载的 Get 函数，传入参数并移动响应处理器、内容接收器和进度
  return Get(path, Headers(), std::move(response_handler),
             std::move(content_receiver), std::move(progress));
}

// 使用给定的路径、请求头、响应处理器、内容接收器和进度发送 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  // 创建一个请求对象
  Request req;
  // 设置请求方法为 GET
  req.method = "GET";
  // 设置请求路径
  req.path = path;
  // 设置请求头
  req.headers = headers;
  // 移动响应处理器
  req.response_handler = std::move(response_handler);
  // 设置内容接收器，将接收到的数据传递给内容接收器
  req.content_receiver =
      [content_receiver](const char *data, size_t data_length,
                         uint64_t /*offset*/, uint64_t /*total_length*/) {
        return content_receiver(data, data_length);
      };
  // 移动进度
  req.progress = std::move(progress);

  // 调用 send_ 函数发送请求并返回结果
  return send_(std::move(req));
}

// 使用给定的路径、参数、请求头和进度发送 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers, Progress progress) {
  // 如果参数为空，则调用重载的 Get 函数，只传入路径和请求头
  if (params.empty()) { return Get(path, headers); }
  // 将参数拼接到路径后，然后调用重载的 Get 函数
  std::string path_with_query = append_query_params(path, params);
  return Get(path_with_query.c_str(), headers, progress);
}
# 使用给定的路径、参数、头部、内容接收器和进度来执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 调用另一个重载的 Get 函数，传入参数并返回结果
  return Get(path, params, headers, nullptr, content_receiver, progress);
}

# 使用给定的路径、参数、头部、响应处理器、内容接收器和进度来执行 GET 请求
inline Result ClientImpl::Get(const std::string &path, const Params &params,
                              const Headers &headers,
                              ResponseHandler response_handler,
                              ContentReceiver content_receiver,
                              Progress progress) {
  # 如果参数为空，则调用另一个重载的 Get 函数，传入参数并返回结果
  if (params.empty()) {
    return Get(path, headers, response_handler, content_receiver, progress);
  }
  # 将路径和参数拼接成带查询参数的路径
  std::string path_with_query = append_query_params(path, params);
  # 调用另一个重载的 Get 函数，传入参数并返回结果
  return Get(path_with_query.c_str(), headers, response_handler,
             content_receiver, progress);
}

# 执行 HEAD 请求，不带头部
inline Result ClientImpl::Head(const std::string &path) {
  return Head(path, Headers());
}

# 执行 HEAD 请求，带头部
inline Result ClientImpl::Head(const std::string &path,
                               const Headers &headers) {
  # 创建一个请求对象，设置方法为 HEAD，头部为给定的头部，路径为给定的路径
  Request req;
  req.method = "HEAD";
  req.headers = headers;
  req.path = path;
  # 调用 send_ 函数，传入请求对象并返回结果
  return send_(std::move(req));
}

# 执行 POST 请求，不带头部和内容
inline Result ClientImpl::Post(const std::string &path) {
  return Post(path, std::string(), std::string());
}

# 执行 POST 请求，带头部，不带内容
inline Result ClientImpl::Post(const std::string &path,
                               const Headers &headers) {
  return Post(path, headers, nullptr, 0, std::string());
}

# 执行 POST 请求，带头部、内容长度、内容类型和内容
inline Result ClientImpl::Post(const std::string &path, const char *body,
                               size_t content_length,
                               const std::string &content_type) {
  return Post(path, Headers(), body, content_length, content_type);
}
# 使用 POST 方法发送请求，包含路径、请求头、请求体、请求体长度和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const char *body, size_t content_length,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}

# 使用 POST 方法发送请求，包含路径、请求体和内容类型
inline Result ClientImpl::Post(const std::string &path, const std::string &body,
                               const std::string &content_type) {
  return Post(path, Headers(), body, content_type);
}

# 使用 POST 方法发送请求，包含路径、请求头、请求体和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const std::string &body,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

# 使用 POST 方法发送请求，包含路径、参数
inline Result ClientImpl::Post(const std::string &path, const Params &params) {
  return Post(path, Headers(), params);
}

# 使用 POST 方法发送请求，包含路径、请求体长度、内容提供者和内容类型
inline Result ClientImpl::Post(const std::string &path, size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  return Post(path, Headers(), content_length, std::move(content_provider),
              content_type);
}

# 使用 POST 方法发送请求，包含路径、无长度的内容提供者和内容类型
inline Result ClientImpl::Post(const std::string &path,
                               ContentProviderWithoutLength content_provider,
                               const std::string &content_type) {
  return Post(path, Headers(), std::move(content_provider), content_type);
}
// 使用 POST 方法发送请求，包含路径、请求头、内容长度、内容提供者和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               size_t content_length,
                               ContentProvider content_provider,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

// 使用 POST 方法发送请求，包含路径、请求头、内容提供者（无长度）和内容类型
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               ContentProviderWithoutLength content_provider,
                               const std::string &content_type) {
  return send_with_content_provider("POST", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 使用 POST 方法发送请求，包含路径、请求头和参数
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const Params &params) {
  // 将参数转换为查询字符串，然后调用上一个 Post 方法发送请求
  auto query = detail::params_to_query_str(params);
  return Post(path, headers, query, "application/x-www-form-urlencoded");
}

// 使用 POST 方法发送请求，包含路径和多部分表单数据项
inline Result ClientImpl::Post(const std::string &path,
                               const MultipartFormDataItems &items) {
  // 调用上一个 Post 方法发送请求，传入空的请求头
  return Post(path, Headers(), items);
}

// 使用 POST 方法发送请求，包含路径、请求头和多部分表单数据项
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const MultipartFormDataItems &items) {
  // 生成多部分数据的边界、内容类型和请求体，然后调用上一个 Post 方法发送请求
  const auto &boundary = detail::make_multipart_data_boundary();
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Post(path, headers, body, content_type.c_str());
}
// 使用 POST 方法发送请求，包含路径、请求头、多部分表单数据项、分隔符
inline Result ClientImpl::Post(const std::string &path, const Headers &headers,
                               const MultipartFormDataItems &items,
                               const std::string &boundary) {
  // 检查多部分表单数据的分隔符是否有效
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    // 如果无效，返回错误结果
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

// 使用 POST 方法发送请求，包含路径、请求头、多部分表单数据项、多部分表单数据提供者项
inline Result
ClientImpl::Post(const std::string &path, const Headers &headers,
                 const MultipartFormDataItems &items,
                 const MultipartFormDataProviderItems &provider_items) {
  // 生成多部分表单数据的分隔符
  const auto &boundary = detail::make_multipart_data_boundary();
  // 获取多部分表单数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 使用内容提供者发送请求
  return send_with_content_provider(
      "POST", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}

// 使用 PUT 方法发送请求，只包含路径
inline Result ClientImpl::Put(const std::string &path) {
  return Put(path, std::string(), std::string());
}

// 使用 PUT 方法发送请求，包含路径、请求体、内容长度、内容类型
inline Result ClientImpl::Put(const std::string &path, const char *body,
                              size_t content_length,
                              const std::string &content_type) {
  return Put(path, Headers(), body, content_length, content_type);
}

// 使用 PUT 方法发送请求，包含路径、请求头、请求体、内容长度、内容类型
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const char *body, size_t content_length,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, body, content_length,
                                    nullptr, nullptr, content_type);
}
# 使用给定的路径和内容发送 PUT 请求，不包含自定义头部
inline Result ClientImpl::Put(const std::string &path, const std::string &body,
                              const std::string &content_type) {
  return Put(path, Headers(), body, content_type);
}

# 使用给定的路径、自定义头部、内容和内容类型发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const std::string &body,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}

# 使用给定的路径、内容长度、内容提供者和内容类型发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  return Put(path, Headers(), content_length, std::move(content_provider),
             content_type);
}

# 使用给定的路径、无长度内容提供者和内容类型发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  return Put(path, Headers(), std::move(content_provider), content_type);
}

# 使用给定的路径、自定义头部、内容长度、内容提供者和内容类型发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              size_t content_length,
                              ContentProvider content_provider,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}
// 使用给定的路径、头部、内容提供者和内容类型发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              ContentProviderWithoutLength content_provider,
                              const std::string &content_type) {
  return send_with_content_provider("PUT", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

// 使用给定的路径和参数发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Params &params) {
  return Put(path, Headers(), params);
}

// 使用给定的路径、头部和参数发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const Params &params) {
  auto query = detail::params_to_query_str(params);
  return Put(path, headers, query, "application/x-www-form-urlencoded");
}

// 使用给定的路径和多部分表单数据项发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path,
                              const MultipartFormDataItems &items) {
  return Put(path, Headers(), items);
}

// 使用给定的路径、头部和多部分表单数据项发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items) {
  // 生成多部分数据的边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 生成多部分数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分数据项
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Put(path, headers, body, content_type);
}

// 使用给定的路径、头部、多部分表单数据项和边界发送 PUT 请求
inline Result ClientImpl::Put(const std::string &path, const Headers &headers,
                              const MultipartFormDataItems &items,
                              const std::string &boundary) {
  // 检查多部分数据的边界字符是否有效
  if (!detail::is_multipart_boundary_chars_valid(boundary)) {
    return Result{nullptr, Error::UnsupportedMultipartBoundaryChars};
  }
  // 生成多部分数据的内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 序列化多部分数据项
  const auto &body = detail::serialize_multipart_formdata(items, boundary);
  return Put(path, headers, body, content_type);
}

// 其他函数未提供，可能在后续代码中
// 使用给定的路径、头部、表单数据项和表单数据提供者项发送 PUT 请求
ClientImpl::Put(const std::string &path, const Headers &headers,
                const MultipartFormDataItems &items,
                const MultipartFormDataProviderItems &provider_items) {
  // 生成多部分数据的边界
  const auto &boundary = detail::make_multipart_data_boundary();
  // 序列化多部分表单数据并获取内容类型
  const auto &content_type =
      detail::serialize_multipart_formdata_get_content_type(boundary);
  // 使用内容提供者发送请求
  return send_with_content_provider(
      "PUT", path, headers, nullptr, 0, nullptr,
      get_multipart_content_provider(boundary, items, provider_items),
      content_type);
}

// 发送 PATCH 请求的重载函数，不带任何数据
inline Result ClientImpl::Patch(const std::string &path) {
  return Patch(path, std::string(), std::string());
}

// 发送 PATCH 请求的重载函数，带有字符数组类型的请求体和内容长度
inline Result ClientImpl::Patch(const std::string &path, const char *body,
                                size_t content_length,
                                const std::string &content_type) {
  return Patch(path, Headers(), body, content_length, content_type);
}

// 发送 PATCH 请求的重载函数，带有头部、字符数组类型的请求体和内容长度
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const char *body, size_t content_length,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, body,
                                    content_length, nullptr, nullptr,
                                    content_type);
}

// 发送 PATCH 请求的重载函数，带有字符串类型的请求体和内容类型
inline Result ClientImpl::Patch(const std::string &path,
                                const std::string &body,
                                const std::string &content_type) {
  return Patch(path, Headers(), body, content_type);
}

// 发送 PATCH 请求的重载函数，带有头部、字符串类型的请求体和内容类型
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                const std::string &body,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, body.data(),
                                    body.size(), nullptr, nullptr,
                                    content_type);
}
# 使用给定的路径、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, size_t content_length,
                                ContentProvider content_provider,
                                const std::string &content_type) {
  return Patch(path, Headers(), content_length, std::move(content_provider),
               content_type);
}

# 使用给定的路径、内容提供者和内容类型执行 PATCH 请求（无需内容长度）
inline Result ClientImpl::Patch(const std::string &path,
                                ContentProviderWithoutLength content_provider,
                                const std::string &content_type) {
  return Patch(path, Headers(), std::move(content_provider), content_type);
}

# 使用给定的路径、标头、内容长度、内容提供者和内容类型执行 PATCH 请求
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                size_t content_length,
                                ContentProvider content_provider,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, nullptr,
                                    content_length, std::move(content_provider),
                                    nullptr, content_type);
}

# 使用给定的路径、标头、内容提供者和内容类型执行 PATCH 请求（无需内容长度）
inline Result ClientImpl::Patch(const std::string &path, const Headers &headers,
                                ContentProviderWithoutLength content_provider,
                                const std::string &content_type) {
  return send_with_content_provider("PATCH", path, headers, nullptr, 0, nullptr,
                                    std::move(content_provider), content_type);
}

# 执行不带标头的 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path) {
  return Delete(path, Headers(), std::string(), std::string());
}

# 使用给定的路径和标头执行 DELETE 请求
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers) {
  return Delete(path, headers, std::string(), std::string());
}
# 使用给定路径和内容发送 DELETE 请求，返回结果
inline Result ClientImpl::Delete(const std::string &path, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  # 调用另一个重载的 Delete 函数，传入默认的 Headers
  return Delete(path, Headers(), body, content_length, content_type);
}

# 使用给定路径、自定义头部和内容发送 DELETE 请求，返回结果
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers, const char *body,
                                 size_t content_length,
                                 const std::string &content_type) {
  # 创建一个请求对象
  Request req;
  # 设置请求方法为 DELETE
  req.method = "DELETE";
  # 设置请求头部为传入的头部
  req.headers = headers;
  # 设置请求路径为传入的路径
  req.path = path;

  # 如果内容类型不为空，将内容类型添加到请求头部
  if (!content_type.empty()) {
    req.headers.emplace("Content-Type", content_type);
  }
  # 将传入的内容和长度添加到请求体中
  req.body.assign(body, content_length);

  # 调用私有函数 send_ 发送请求并返回结果
  return send_(std::move(req));
}

# 使用给定路径、内容和内容类型发送 DELETE 请求，返回结果
inline Result ClientImpl::Delete(const std::string &path,
                                 const std::string &body,
                                 const std::string &content_type) {
  # 调用另一个重载的 Delete 函数，传入默认的 Headers
  return Delete(path, Headers(), body.data(), body.size(), content_type);
}

# 使用给定路径、自定义头部、内容和内容类型发送 DELETE 请求，返回结果
inline Result ClientImpl::Delete(const std::string &path,
                                 const Headers &headers,
                                 const std::string &body,
                                 const std::string &content_type) {
  # 调用另一个重载的 Delete 函数，传入自定义的头部、内容和内容类型
  return Delete(path, headers, body.data(), body.size(), content_type);
}

# 发送 OPTIONS 请求，使用给定路径和默认头部，返回结果
inline Result ClientImpl::Options(const std::string &path) {
  # 调用另一个重载的 Options 函数，传入默认的 Headers
  return Options(path, Headers());
}

# 发送 OPTIONS 请求，使用给定路径和自定义头部，返回结果
inline Result ClientImpl::Options(const std::string &path,
                                  const Headers &headers) {
  # 创建一个请求对象
  Request req;
  # 设置请求方法为 OPTIONS
  req.method = "OPTIONS";
  # 设置请求头部为传入的头部
  req.headers = headers;
  # 设置请求路径为传入的路径
  req.path = path;

  # 调用私有函数 send_ 发送请求并返回结果
  return send_(std::move(req));
}

# 返回套接字是否打开的状态
inline size_t ClientImpl::is_socket_open() const {
  # 使用互斥锁保护，返回套接字是否打开的状态
  std::lock_guard<std::mutex> guard(socket_mutex_);
  return socket_.is_open();
}

# 返回套接字
inline socket_t ClientImpl::socket() const { return socket_.sock; }
// 停止客户端的操作
inline void ClientImpl::stop() {
  // 使用互斥锁保护
  std::lock_guard<std::mutex> guard(socket_mutex_);

  // 如果有正在进行的请求，唯一线程安全的操作是关闭套接字，使得使用该套接字的线程发现无法再读写并报错
  if (socket_requests_in_flight_ > 0) {
    shutdown_socket(socket_);

    // 设置标志，表示请求完成后应关闭套接字
    socket_should_be_closed_when_request_is_done_ = true;
    return;
  }

  // 否则，在持有互斥锁的情况下，可以自行关闭所有连接
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

// 设置Bearer Token认证信息
inline void ClientImpl::set_bearer_token_auth(const std::string &token) {
  bearer_token_auth_token_ = token;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置摘要认证信息（仅在支持OpenSSL时有效）
inline void ClientImpl::set_digest_auth(const std::string &username,
                                        const std::string &password) {
  digest_auth_username_ = username;
  digest_auth_password_ = password;
}
#endif

// 设置是否保持长连接
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

// 设置代理
inline void ClientImpl::set_proxy(const std::string &host, int port) {
  proxy_host_ = host;
  proxy_port_ = port;
}

// 设置代理的基本认证信息
inline void ClientImpl::set_proxy_basic_auth(const std::string &username,
                                             const std::string &password) {
  proxy_basic_auth_username_ = username;
  proxy_basic_auth_password_ = password;
}

// 设置代理的 Bearer Token 认证信息
inline void ClientImpl::set_proxy_bearer_token_auth(const std::string &token) {
  proxy_bearer_token_auth_token_ = token;
}

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理的摘要认证信息
inline void ClientImpl::set_proxy_digest_auth(const std::string &username,
                                              const std::string &password) {
  proxy_digest_auth_username_ = username;
  proxy_digest_auth_password_ = password;
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书路径
inline void ClientImpl::set_ca_cert_path(const std::string &ca_cert_file_path,
                                         const std::string &ca_cert_dir_path) {
  ca_cert_file_path_ = ca_cert_file_path;
  ca_cert_dir_path_ = ca_cert_dir_path;
}

// 设置 CA 证书存储
inline void ClientImpl::set_ca_cert_store(X509_STORE *ca_cert_store) {
  if (ca_cert_store && ca_cert_store != ca_cert_store_) {
    # 将参数 ca_cert_store 的值赋给类成员变量 ca_cert_store_
    ca_cert_store_ = ca_cert_store;
  }
// 结束条件编译指令
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 启用或禁用服务器证书验证
inline void ClientImpl::enable_server_certificate_verification(bool enabled) {
  server_certificate_verification_ = enabled;
}
#endif

// 设置日志记录器
inline void ClientImpl::set_logger(Logger logger) {
  logger_ = std::move(logger);
}

/*
 * SSL Implementation
 */
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
namespace detail {

// SSL 新建连接函数
template <typename U, typename V>
inline SSL *ssl_new(socket_t sock, SSL_CTX *ctx, std::mutex &ctx_mutex,
                    U SSL_connect_or_accept, V setup) {
  SSL *ssl = nullptr;
  {
    std::lock_guard<std::mutex> guard(ctx_mutex);
    ssl = SSL_new(ctx);
  }

  if (ssl) {
    set_nonblocking(sock, true);
    auto bio = BIO_new_socket(static_cast<int>(sock), BIO_NOCLOSE);
    BIO_set_nbio(bio, 1);
    SSL_set_bio(ssl, bio, bio);

    if (!setup(ssl) || SSL_connect_or_accept(ssl) != 1) {
      SSL_shutdown(ssl);
      {
        std::lock_guard<std::mutex> guard(ctx_mutex);
        SSL_free(ssl);
      }
      set_nonblocking(sock, false);
      return nullptr;
    }
    BIO_set_nbio(bio, 0);
    set_nonblocking(sock, false);
  }

  return ssl;
}

// SSL 删除连接函数
inline void ssl_delete(std::mutex &ctx_mutex, SSL *ssl,
                       bool shutdown_gracefully) {
  // 有时我们可能希望跳过这一步，尝试避免 SIGPIPE，如果我们知道远程已关闭网络连接
  // 请注意，无法始终避免 SIGPIPE，这只是尽力而为
  if (shutdown_gracefully) { SSL_shutdown(ssl); }

  std::lock_guard<std::mutex> guard(ctx_mutex);
  SSL_free(ssl);
}

// SSL 非阻塞连接或接受函数
template <typename U>
bool ssl_connect_or_accept_nonblocking(socket_t sock, SSL *ssl,
                                       U ssl_connect_or_accept,
                                       time_t timeout_sec,
                                       time_t timeout_usec) {
  int res = 0;
  while ((res = ssl_connect_or_accept(ssl)) != 1) {
    auto err = SSL_get_error(ssl, res);
    switch (err) {
    # 如果 SSL 连接出现 SSL_ERROR_WANT_READ 错误
    case SSL_ERROR_WANT_READ:
      # 如果套接字可读，继续循环
      if (select_read(sock, timeout_sec, timeout_usec) > 0) { continue; }
      # 否则跳出循环
      break;
    # 如果 SSL 连接出现 SSL_ERROR_WANT_WRITE 错误
    case SSL_ERROR_WANT_WRITE:
      # 如果套接字可写，继续循环
      if (select_write(sock, timeout_sec, timeout_usec) > 0) { continue; }
      # 否则跳出循环
      break;
    # 其他情况
    default: break;
    }
    # 返回 false
    return false;
  }
  # 返回 true
  return true;
// 使用模板处理服务器端 SSL 套接字
template <typename T>
inline bool process_server_socket_ssl(
    const std::atomic<socket_t> &svr_sock, SSL *ssl, socket_t sock,
    size_t keep_alive_max_count, time_t keep_alive_timeout_sec,
    time_t read_timeout_sec, time_t read_timeout_usec, time_t write_timeout_sec,
    time_t write_timeout_usec, T callback) {
  return process_server_socket_core(
      svr_sock, sock, keep_alive_max_count, keep_alive_timeout_sec,
      [&](bool close_connection, bool &connection_closed) {
        // 使用 SSL 套接字流处理函数回调
        SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                             write_timeout_sec, write_timeout_usec);
        return callback(strm, close_connection, connection_closed);
      });
}

// 使用模板处理客户端 SSL 套接字
template <typename T>
inline bool
process_client_socket_ssl(SSL *ssl, socket_t sock, time_t read_timeout_sec,
                          time_t read_timeout_usec, time_t write_timeout_sec,
                          time_t write_timeout_usec, T callback) {
  // 使用 SSL 套接字流处理函数回调
  SSLSocketStream strm(sock, ssl, read_timeout_sec, read_timeout_usec,
                       write_timeout_sec, write_timeout_usec);
  return callback(strm);
}

// SSL 初始化类
class SSLInit {
public:
  SSLInit() {
    // 初始化 SSL 库
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
  // 清除 SSL 模式中的自动重试标志
  SSL_clear_mode(ssl, SSL_MODE_AUTO_RETRY);
}

// SSL 套接字流析构函数
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
  // 如果 SSL 有未读取的数据，则直接读取
  if (SSL_pending(ssl_) > 0) {
    return SSL_read(ssl_, ptr, static_cast<int>(size));
  } else if (is_readable()) {  // 如果套接字可读
    auto ret = SSL_read(ssl_, ptr, static_cast<int>(size));  // 读取数据
    if (ret < 0) {  // 如果读取失败
      auto err = SSL_get_error(ssl_, ret);  // 获取错误码
      int n = 1000;
#ifdef _WIN32
      // 在 Windows 平台上，处理 SSL_ERROR_WANT_READ 和 SSL_ERROR_SYSCALL 错误
      while (--n >= 0 && (err == SSL_ERROR_WANT_READ ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 在非 Windows 平台上，处理 SSL_ERROR_WANT_READ 错误
      while (--n >= 0 && err == SSL_ERROR_WANT_READ) {
#endif
        if (SSL_pending(ssl_) > 0) {  // 如果 SSL 有未读取的数据
          return SSL_read(ssl_, ptr, static_cast<int>(size));  // 直接读取数据
        } else if (is_readable()) {  // 如果套接字可读
          std::this_thread::sleep_for(std::chrono::milliseconds(1));  // 等待一段时间
          ret = SSL_read(ssl_, ptr, static_cast<int>(size));  // 再次尝试读取数据
          if (ret >= 0) { return ret; }  // 如果成功读取数据，则返回
          err = SSL_get_error(ssl_, ret);  // 获取错误码
        } else {
          return -1;  // 如果套接字不可读，则返回错误
        }
      }
    }
    return ret;  // 返回读取的数据长度
  }
  return -1;  // 如果套接字不可读，则返回错误
}

// 向套接字中写入数据
inline ssize_t SSLSocketStream::write(const char *ptr, size_t size) {
  if (is_writable()) {  // 如果套接字可写
    auto handle_size = static_cast<int>(
        std::min<size_t>(size, (std::numeric_limits<int>::max)()));  // 计算可写入的数据长度

    auto ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));  // 写入数据
    if (ret < 0) {  // 如果写入失败
      auto err = SSL_get_error(ssl_, ret);  // 获取错误码
      int n = 1000;
#ifdef _WIN32
      // 在 Windows 平台上，处理 SSL_ERROR_WANT_WRITE 和 SSL_ERROR_SYSCALL 错误
      while (--n >= 0 && (err == SSL_ERROR_WANT_WRITE ||
                          (err == SSL_ERROR_SYSCALL &&
                           WSAGetLastError() == WSAETIMEDOUT))) {
#else
      // 在非 Windows 平台上，处理 SSL_ERROR_WANT_WRITE 错误
      while (--n >= 0 && err == SSL_ERROR_WANT_WRITE) {
#endif
        // 如果 SSL 套接字可写
        if (is_writable()) {
          // 线程休眠 1 毫秒
          std::this_thread::sleep_for(std::chrono::milliseconds(1));
          // 将数据写入 SSL 套接字
          ret = SSL_write(ssl_, ptr, static_cast<int>(handle_size));
          // 如果写入成功，返回写入的字节数
          if (ret >= 0) { return ret; }
          // 获取 SSL 错误码
          err = SSL_get_error(ssl_, ret);
        } else {
          // 如果 SSL 套接字不可写，返回 -1
          return -1;
        }
      }
    }
    // 返回写入的字节数或错误码
    return ret;
  }
  // 如果没有写入任何数据，返回 -1
  return -1;
}

// 获取远程 IP 地址和端口号
inline void SSLSocketStream::get_remote_ip_and_port(std::string &ip,
                                                    int &port) const {
  detail::get_remote_ip_and_port(sock_, ip, port);
}

// 获取本地 IP 地址和端口号
inline void SSLSocketStream::get_local_ip_and_port(std::string &ip,
                                                   int &port) const {
  detail::get_local_ip_and_port(sock_, ip, port);
}

// 获取套接字
inline socket_t SSLSocketStream::socket() const { return sock_; }

// SSL 初始化
static SSLInit sslinit_;

} // namespace detail

// SSL HTTP 服务器实现
// 构造函数，初始化 SSL 上下文
inline SSLServer::SSLServer(const char *cert_path, const char *private_key_path,
                            const char *client_ca_cert_file_path,
                            const char *client_ca_cert_dir_path,
                            const char *private_key_password) {
  // 创建 SSL 上下文
  ctx_ = SSL_CTX_new(TLS_server_method());

  if (ctx_) {
    // 设置 SSL 上下文选项
    SSL_CTX_set_options(ctx_,
                        SSL_OP_NO_COMPRESSION |
                            SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

    // 设置 SSL 上下文最小协议版本
    SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

    // 在打开加密私钥之前添加默认密码回调
    if (private_key_password != nullptr && (private_key_password[0] != '\0')) {
      SSL_CTX_set_default_passwd_cb_userdata(ctx_,
                                             (char *)private_key_password);
    }

    // 使用证书链文件和私钥文件配置 SSL 上下文
    if (SSL_CTX_use_certificate_chain_file(ctx_, cert_path) != 1 ||
        SSL_CTX_use_PrivateKey_file(ctx_, private_key_path, SSL_FILETYPE_PEM) !=
            1) {
      // 如果配置失败，释放 SSL 上下文并置空
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    } else if (client_ca_cert_file_path || client_ca_cert_dir_path) {
      // 如果客户端 CA 证书文件路径或客户端 CA 证书目录路径存在，则加载客户端 CA 证书位置
      SSL_CTX_load_verify_locations(ctx_, client_ca_cert_file_path,
                                    client_ca_cert_dir_path);

      // 设置 SSL 上下文的验证方式为要求对等方提供证书，并且如果没有对等方证书则验证失败
      SSL_CTX_set_verify(
          ctx_, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, nullptr);
    }
  }
// SSLServer 类的构造函数，接受证书、私钥和客户端 CA 证书存储作为参数
inline SSLServer::SSLServer(X509 *cert, EVP_PKEY *private_key,
                            X509_STORE *client_ca_cert_store) {
  // 创建 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_server_method());

  if (ctx_) {
    // 设置 SSL 上下文对象的选项
    SSL_CTX_set_options(ctx_,
                        SSL_OP_NO_COMPRESSION |
                            SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION);

    // 设置 SSL 上下文对象的最小协议版本
    SSL_CTX_set_min_proto_version(ctx_, TLS1_1_VERSION);

    // 使用证书和私钥配置 SSL 上下文对象
    if (SSL_CTX_use_certificate(ctx_, cert) != 1 ||
        SSL_CTX_use_PrivateKey(ctx_, private_key) != 1) {
      // 如果配置失败，则释放 SSL 上下文对象并置空
      SSL_CTX_free(ctx_);
      ctx_ = nullptr;
    } else if (client_ca_cert_store) {
      // 如果客户端 CA 证书存储存在，则设置到 SSL 上下文对象中
      SSL_CTX_set_cert_store(ctx_, client_ca_cert_store);

      // 设置 SSL 上下文对象的验证方式
      SSL_CTX_set_verify(
          ctx_, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, nullptr);
    }
  }
}

// SSLServer 类的构造函数，接受一个设置 SSL 上下文对象的回调函数作为参数
inline SSLServer::SSLServer(
    const std::function<bool(SSL_CTX &ssl_ctx)> &setup_ssl_ctx_callback) {
  // 创建 SSL 上下文对象
  ctx_ = SSL_CTX_new(TLS_method());
  if (ctx_) {
    // 调用回调函数设置 SSL 上下文对象，如果失败则释放 SSL 上下文对象并置空
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

// 处理并关闭套接字的 SSL 连接
inline bool SSLServer::process_and_close_socket(socket_t sock) {
  // 创建 SSL 对象
  auto ssl = detail::ssl_new(
      sock, ctx_, ctx_mutex_,
      [&](SSL *ssl2) {
        return detail::ssl_connect_or_accept_nonblocking(
            sock, ssl2, SSL_accept, read_timeout_sec_, read_timeout_usec_);
      },
      [](SSL * /*ssl2*/) { return true; });

  auto ret = false;
  if (ssl) {
    // 如果 SSL 对象存在，则执行相应操作
    // 调用 detail 命名空间中的 process_server_socket_ssl 函数，处理服务器套接字的 SSL 连接
    ret = detail::process_server_socket_ssl(
        svr_sock_, ssl, sock, keep_alive_max_count_, keep_alive_timeout_sec_,
        read_timeout_sec_, read_timeout_usec_, write_timeout_sec_,
        write_timeout_usec_,
        // 使用 lambda 表达式处理请求
        [this, ssl](Stream &strm, bool close_connection,
                    bool &connection_closed) {
          return process_request(strm, close_connection, connection_closed,
                                 [&](Request &req) { req.ssl = ssl; });
        });

    // 如果结果看起来成功，则优雅地关闭连接，如果连接似乎已关闭，则非优雅地关闭
    const bool shutdown_gracefully = ret;
    // 调用 detail 命名空间中的 ssl_delete 函数，删除 SSL 连接
    detail::ssl_delete(ctx_mutex_, ssl, shutdown_gracefully);
  }

  // 优雅地关闭套接字
  detail::shutdown_socket(sock);
  // 关闭套接字
  detail::close_socket(sock);
  // 返回处理结果
  return ret;
// SSL HTTP 客户端实现
inline SSLClient::SSLClient(const std::string &host)
    : SSLClient(host, 443, std::string(), std::string()) {}  // 调用带默认端口号和空客户端证书路径、密钥路径的构造函数

inline SSLClient::SSLClient(const std::string &host, int port)
    : SSLClient(host, port, std::string(), std::string()) {}  // 调用带空客户端证书路径、密钥路径的构造函数

inline SSLClient::SSLClient(const std::string &host, int port,
                            const std::string &client_cert_path,
                            const std::string &client_key_path)
    : ClientImpl(host, port, client_cert_path, client_key_path) {  // 调用基类构造函数，设置主机、端口、客户端证书路径、密钥路径
  ctx_ = SSL_CTX_new(TLS_client_method());  // 创建 SSL 上下文对象

  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));  // 将主机名按点分割成组件并存储
                });

  if (!client_cert_path.empty() && !client_key_path.empty()) {  // 如果客户端证书路径和密钥路径不为空
    if (SSL_CTX_use_certificate_file(ctx_, client_cert_path.c_str(),
                                     SSL_FILETYPE_PEM) != 1 ||  // 使用客户端证书文件
        SSL_CTX_use_PrivateKey_file(ctx_, client_key_path.c_str(),
                                    SSL_FILETYPE_PEM) != 1) {  // 使用客户端私钥文件
      SSL_CTX_free(ctx_);  // 释放 SSL 上下文对象
      ctx_ = nullptr;
    }
  }
}

inline SSLClient::SSLClient(const std::string &host, int port,
                            X509 *client_cert, EVP_PKEY *client_key)
    : ClientImpl(host, port) {  // 调用基类构造函数，设置主机、端口
  ctx_ = SSL_CTX_new(TLS_client_method());  // 创建 SSL 上下文对象

  detail::split(&host_[0], &host_[host_.size()], '.',
                [&](const char *b, const char *e) {
                  host_components_.emplace_back(std::string(b, e));  // 将主机名按点分割成组件并存储
                });

  if (client_cert != nullptr && client_key != nullptr) {  // 如果客户端证书和密钥不为空
    if (SSL_CTX_use_certificate(ctx_, client_cert) != 1 ||  // 使用客户端证书
        SSL_CTX_use_PrivateKey(ctx_, client_key) != 1) {  // 使用客户端私钥
      SSL_CTX_free(ctx_);  // 释放 SSL 上下文对象
      ctx_ = nullptr;
    }
  }
}
// SSLClient 析构函数，用于释放 SSL 上下文
inline SSLClient::~SSLClient() {
  // 如果上下文存在，则释放上下文
  if (ctx_) { SSL_CTX_free(ctx_); }
  // 确保关闭 SSL，因为在基类析构函数中，shutdown_ssl 将解析为基类函数而不是派生函数，不会释放 SSL（导致内存泄漏）
  shutdown_ssl_impl(socket_, true);
}

// 检查 SSLClient 是否有效
inline bool SSLClient::is_valid() const { return ctx_; }

// 设置 CA 证书存储
inline void SSLClient::set_ca_cert_store(X509_STORE *ca_cert_store) {
  if (ca_cert_store) {
    if (ctx_) {
      // 如果上下文存在且证书存储不同，则释放旧的证书存储并使用新的证书存储
      if (SSL_CTX_get_cert_store(ctx_) != ca_cert_store) {
        SSL_CTX_set_cert_store(ctx_, ca_cert_store);
      }
    } else {
      // 如果上下文不存在，则释放证书存储
      X509_STORE_free(ca_cert_store);
    }
  }
}

// 获取 OpenSSL 验证结果
inline long SSLClient::get_openssl_verify_result() const {
  return verify_result_;
}

// 获取 SSL 上下文
inline SSL_CTX *SSLClient::ssl_context() const { return ctx_; }

// 创建并连接套接字
inline bool SSLClient::create_and_connect_socket(Socket &socket, Error &error) {
  // 检查 SSLClient 是否有效，并调用基类函数创建并连接套接字
  return is_valid() && ClientImpl::create_and_connect_socket(socket, error);
}

// 假设 socket_mutex_ 已锁定且没有请求在传输中，使用代理连接
inline bool SSLClient::connect_with_proxy(Socket &socket, Response &res,
                                          bool &success, Error &error) {
  success = true;
  Response res2;
  // 如果没有处理客户端套接字，则关闭 SSL、套接字并返回失败
  if (!detail::process_client_socket(
          socket.sock, read_timeout_sec_, read_timeout_usec_,
          write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
            Request req2;
            req2.method = "CONNECT";
            req2.path = host_and_port_;
            return process_request(strm, req2, res2, false, error);
          })) {
    shutdown_ssl(socket, true);
    shutdown_socket(socket);
    close_socket(socket);
    success = false;
    return false;
  }

  // 如果响应状态码为 407，则需要代理认证
  if (res2.status == 407) {
    // 如果代理身份验证用户名和密码都不为空
    if (!proxy_digest_auth_username_.empty() &&
        !proxy_digest_auth_password_.empty()) {
      // 创建一个空的字符串到字符串的映射
      std::map<std::string, std::string> auth;
      // 解析响应头中的身份验证信息，存储到auth中
      if (detail::parse_www_authenticate(res2, auth, true)) {
        // 创建一个空的响应对象
        Response res3;
        // 处理客户端套接字，发送CONNECT请求
        if (!detail::process_client_socket(
                socket.sock, read_timeout_sec_, read_timeout_usec_,
                write_timeout_sec_, write_timeout_usec_, [&](Stream &strm) {
                  // 创建一个空的请求对象
                  Request req3;
                  // 设置请求方法为"CONNECT"
                  req3.method = "CONNECT";
                  // 设置请求路径为host_and_port_
                  req3.path = host_and_port_;
                  // 插入摘要认证头到请求头中
                  req3.headers.insert(detail::make_digest_authentication_header(
                      req3, auth, 1, detail::random_string(10),
                      proxy_digest_auth_username_, proxy_digest_auth_password_,
                      true));
                  // 处理请求并返回结果
                  return process_request(strm, req3, res3, false, error);
                })) {
          // 线程安全地关闭所有连接，因为我们假设没有请求在传输中
          shutdown_ssl(socket, true);
          shutdown_socket(socket);
          close_socket(socket);
          success = false;
          return false;
        }
      }
    } else {
      // 将res2赋值给res
      res = res2;
      // 返回false
      return false;
    }
  }
  // 返回true
  return true;
}

// 加载证书
inline bool SSLClient::load_certs() {
  bool ret = true;

  // 使用 std::call_once 保证初始化只执行一次
  std::call_once(initialize_cert_, [&]() {
    // 使用互斥锁保证线程安全
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
    // 如果都为空
    else {
      auto loaded = false;
      // 如果是 Windows 系统
#ifdef _WIN32
      loaded =
          detail::load_system_certs_on_windows(SSL_CTX_get_cert_store(ctx_));
      // 如果是 macOS 系统
#elif defined(CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN) && defined(__APPLE__)
#if TARGET_OS_OSX
      loaded = detail::load_system_certs_on_macos(SSL_CTX_get_cert_store(ctx_));
#endif // TARGET_OS_OSX
#endif // _WIN32
      // 如果加载失败，使用默认的证书路径
      if (!loaded) { SSL_CTX_set_default_verify_paths(ctx_); }
    }
  });

  return ret;
}
// 初始化 SSL 客户端，使用给定的套接字和 SSL 上下文
inline bool SSLClient::initialize_ssl(Socket &socket, Error &error) {
  // 创建 SSL 对象
  auto ssl = detail::ssl_new(
      socket.sock, ctx_, ctx_mutex_,
      // SSL 连接回调函数
      [&](SSL *ssl2) {
        // 如果需要服务器证书验证
        if (server_certificate_verification_) {
          // 加载证书
          if (!load_certs()) {
            error = Error::SSLLoadingCerts;
            return false;
          }
          // 设置 SSL 验证方式
          SSL_set_verify(ssl2, SSL_VERIFY_NONE, nullptr);
        }

        // 非阻塞 SSL 连接或接受
        if (!detail::ssl_connect_or_accept_nonblocking(
                socket.sock, ssl2, SSL_connect, connection_timeout_sec_,
                connection_timeout_usec_)) {
          error = Error::SSLConnection;
          return false;
        }

        // 如果需要服务器证书验证
        if (server_certificate_verification_) {
          // 获取验证结果
          verify_result_ = SSL_get_verify_result(ssl2);

          // 如果验证结果不通过
          if (verify_result_ != X509_V_OK) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 获取服务器证书
          auto server_cert = SSL_get1_peer_certificate(ssl2);

          // 如果服务器证书为空
          if (server_cert == nullptr) {
            error = Error::SSLServerVerification;
            return false;
          }

          // 验证服务器主机名
          if (!verify_host(server_cert)) {
            X509_free(server_cert);
            error = Error::SSLServerVerification;
            return false;
          }
          X509_free(server_cert);
        }

        return true;
      },
      // SSL 设置主机名回调函数
      [&](SSL *ssl2) {
        SSL_set_tlsext_host_name(ssl2, host_.c_str());
        return true;
      });

  // 如果 SSL 对象创建成功
  if (ssl) {
    // 将 SSL 对象赋值给套接字
    socket.ssl = ssl;
    return true;
  }

  // 关闭套接字
  shutdown_socket(socket);
  // 关闭 SSL 连接
  close_socket(socket);
  return false;
}

// 关闭 SSL 客户端
inline void SSLClient::shutdown_ssl(Socket &socket, bool shutdown_gracefully) {
  // 调用关闭 SSL 实现函数
  shutdown_ssl_impl(socket, shutdown_gracefully);
}

// 关闭 SSL 实现函数
inline void SSLClient::shutdown_ssl_impl(Socket &socket,
                                         bool shutdown_gracefully) {
  // 如果套接字无效
  if (socket.sock == INVALID_SOCKET) {
    assert(socket.ssl == nullptr);
    return;
  }
  // 如果 SSL 对象存在
  if (socket.ssl) {
    // 删除 SSL 对象
    detail::ssl_delete(ctx_mutex_, socket.ssl, shutdown_gracefully);
    # 将 socket 对象的 ssl 属性设置为 nullptr，表示没有 SSL 连接
    socket.ssl = nullptr;
  }
  # 断言 socket 对象的 ssl 属性为 nullptr，即确保 SSL 连接已经关闭
  assert(socket.ssl == nullptr);
// 检查是否为 SSL 客户端，处理套接字的 SSL 连接，并调用回调函数
inline bool
SSLClient::process_socket(const Socket &socket,
                          std::function<bool(Stream &strm)> callback) {
  // 断言套接字为 SSL 类型
  assert(socket.ssl);
  // 调用 detail 命名空间中的函数处理 SSL 客户端套接字
  return detail::process_client_socket_ssl(
      socket.ssl, socket.sock, read_timeout_sec_, read_timeout_usec_,
      write_timeout_sec_, write_timeout_usec_, std::move(callback));
}

// 检查是否为 SSL 客户端
inline bool SSLClient::is_ssl() const { return true; }

// 验证服务器证书的主机名
inline bool SSLClient::verify_host(X509 *server_cert) const {
  /* 引用自 RFC2818 第 3.1 节 "服务器标识"

     如果存在类型为 dNSName 的 subjectAltName 扩展，则必须使用该扩展作为标识。
     否则，必须使用证书主体字段中的（最具体的）通用名称字段。尽管使用通用名称是现有的做法，
     但已被弃用，并鼓励认证机构使用 dNSName。

     匹配使用[RFC2459]指定的匹配规则执行。如果证书中存在给定类型的多个标识（例如，多个 dNSName 名称），
     则在集合中的任何一个匹配都被视为可接受。名称可能包含通配符字符*，被视为匹配任何单个域名组件或组件片段。
     例如，*.a.com 匹配 foo.a.com 但不匹配 bar.foo.a.com。f*.com 匹配 foo.com 但不匹配 bar.com。

     在某些情况下，URI 被指定为 IP 地址而不是主机名。在这种情况下，证书中必须存在 iPAddress subjectAltName，
     并且必须与 URI 中的 IP 完全匹配。
  */
  return verify_host_with_subject_alt_name(server_cert) ||
         verify_host_with_common_name(server_cert);
}

// 使用 subjectAltName 验证服务器证书的主机名
inline bool
SSLClient::verify_host_with_subject_alt_name(X509 *server_cert) const {
  auto ret = false;

  auto type = GEN_DNS;

  struct in6_addr addr6;
  struct in_addr addr;
  size_t addr_len = 0;

#ifndef __MINGW32__
  // 如果主机名为 IPv6 地址，则使用 inet_pton 函数将其转换为网络地址结构
  if (inet_pton(AF_INET6, host_.c_str(), &addr6)) {
    # 设置类型为IP地址
    type = GEN_IPADD;
    # 如果是IPv6地址，则设置地址长度为IPv6地址结构体的大小
    addr_len = sizeof(struct in6_addr);
  # 如果不是IPv6地址，但是能够转换为IPv4地址，则设置类型为IP地址
  } else if (inet_pton(AF_INET, host_.c_str(), &addr)) {
    # 设置类型为IP地址
    type = GEN_IPADD;
    # 设置地址长度为IPv4地址结构体的大小
    addr_len = sizeof(struct in_addr);
  }
#endif
// 结束预处理指令

  auto alt_names = static_cast<const struct stack_st_GENERAL_NAME *>(
      X509_get_ext_d2i(server_cert, NID_subject_alt_name, nullptr, nullptr));
  // 获取证书中的主体备用名称扩展

  if (alt_names) {
    auto dsn_matched = false;
    auto ip_matched = false;

    auto count = sk_GENERAL_NAME_num(alt_names);
    // 获取备用名称的数量

    for (decltype(count) i = 0; i < count && !dsn_matched; i++) {
      auto val = sk_GENERAL_NAME_value(alt_names, i);
      // 获取备用名称的值
      if (val->type == type) {
        auto name = (const char *)ASN1_STRING_get0_data(val->d.ia5);
        auto name_len = (size_t)ASN1_STRING_length(val->d.ia5);
        // 获取备用名称的类型和长度

        switch (type) {
        case GEN_DNS: dsn_matched = check_host_name(name, name_len); break;
        // 如果是 DNS 类型的备用名称，则检查主机名是否匹配
        case GEN_IPADD:
          if (!memcmp(&addr6, name, addr_len) ||
              !memcmp(&addr, name, addr_len)) {
            ip_matched = true;
          }
          // 如果是 IP 地址类型的备用名称，则检查 IP 地址是否匹配
          break;
        }
      }
    }

    if (dsn_matched || ip_matched) { ret = true; }
    // 如果主机名或 IP 地址匹配，则返回 true
  }

  GENERAL_NAMES_free((STACK_OF(GENERAL_NAME) *)alt_names);
  // 释放备用名称
  return ret;
  // 返回结果

}

inline bool SSLClient::verify_host_with_common_name(X509 *server_cert) const {
  const auto subject_name = X509_get_subject_name(server_cert);
  // 获取证书的主体名称

  if (subject_name != nullptr) {
    char name[BUFSIZ];
    auto name_len = X509_NAME_get_text_by_NID(subject_name, NID_commonName,
                                              name, sizeof(name));
    // 获取证书的通用名称

    if (name_len != -1) {
      return check_host_name(name, static_cast<size_t>(name_len));
      // 检查主机名是否匹配
    }
  }

  return false;
  // 如果没有通用名称，则返回 false
}
// 检查主机名是否与给定模式匹配
inline bool SSLClient::check_host_name(const char *pattern,
                                       size_t pattern_len) const {
  // 如果主机名长度与模式长度相等且内容相同，则返回true
  if (host_.size() == pattern_len && host_ == pattern) { return true; }

  // 通配符匹配
  // https://bugs.launchpad.net/ubuntu/+source/firefox-3.0/+bug/376484
  // 将模式按'.'分割成组件，存储到pattern_components中
  std::vector<std::string> pattern_components;
  detail::split(&pattern[0], &pattern[pattern_len], '.',
                [&](const char *b, const char *e) {
                  pattern_components.emplace_back(std::string(b, e));
                });

  // 如果主机名组件数量与模式组件数量不相等，则返回false
  if (host_components_.size() != pattern_components.size()) { return false; }

  // 遍历主机名组件和模式组件，进行匹配
  auto itr = pattern_components.begin();
  for (const auto &h : host_components_) {
    auto &p = *itr;
    if (p != h && p != "*") {
      // 如果模式组件不等于主机名组件且不为通配符'*'，则进行部分匹配
      auto partial_match = (p.size() > 0 && p[p.size() - 1] == '*' &&
                            !p.compare(0, p.size() - 1, h));
      if (!partial_match) { return false; }
    }
    ++itr;
  }

  return true;
}
#endif

// 通用客户端实现
// 如果scheme_host_port为空，则调用第一个构造函数
inline Client::Client(const std::string &scheme_host_port)
    : Client(scheme_host_port, std::string(), std::string()) {}

// 根据给定的scheme_host_port、client_cert_path和client_key_path构造客户端
inline Client::Client(const std::string &scheme_host_port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path) {
  // 定义正则表达式模式
  const static std::regex re(
      R"((?:([a-z]+):\/\/)?(?:\[([\d:]+)\]|([^:/?#]+))(?::(\d+))?)");

  std::smatch m;
  // 使用正则表达式匹配scheme_host_port
  if (std::regex_match(scheme_host_port, m, re)) {
    auto scheme = m[1].str();

    // 如果scheme不为空且不为http或https，则抛出异常
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
    if (!scheme.empty() && (scheme != "http" && scheme != "https")) {
#else
    if (!scheme.empty() && scheme != "http") {
#endif
#ifndef CPPHTTPLIB_NO_EXCEPTIONS
      std::string msg = "'" + scheme + "' scheme is not supported.";
      throw std::invalid_argument(msg);
#endif
      return;
    }

    // 判断是否为https
    auto is_ssl = scheme == "https";

    // 获取主机名
    auto host = m[2].str();
    if (host.empty()) { host = m[3].str(); }

    // 获取端口号
    auto port_str = m[4].str();
    # 如果端口字符串不为空，则将其转换为整数，否则根据是否使用 SSL 来选择默认端口号
    auto port = !port_str.empty() ? std::stoi(port_str) : (is_ssl ? 443 : 80);
    
    # 如果使用 SSL，则执行以下代码块
    if (is_ssl) {
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
      // 如果支持 OpenSSL，则创建 SSLClient 对象
      cli_ = detail::make_unique<SSLClient>(host, port, client_cert_path,
                                            client_key_path);
      // 设置为 SSL 连接
      is_ssl_ = is_ssl;
#endif
    } else {
      // 否则创建普通的 ClientImpl 对象
      cli_ = detail::make_unique<ClientImpl>(host, port, client_cert_path,
                                             client_key_path);
    }
  } else {
    // 如果没有指定端口，则默认使用 80 端口创建 ClientImpl 对象
    cli_ = detail::make_unique<ClientImpl>(scheme_host_port, 80,
                                           client_cert_path, client_key_path);
  }
}

// 根据主机名和端口号创建 Client 对象
inline Client::Client(const std::string &host, int port)
    : cli_(detail::make_unique<ClientImpl>(host, port)) {}

// 根据主机名、端口号、客户端证书路径和客户端密钥路径创建 Client 对象
inline Client::Client(const std::string &host, int port,
                      const std::string &client_cert_path,
                      const std::string &client_key_path)
    : cli_(detail::make_unique<ClientImpl>(host, port, client_cert_path,
                                           client_key_path)) {}

// Client 对象的析构函数
inline Client::~Client() {}

// 检查 Client 对象是否有效
inline bool Client::is_valid() const {
  return cli_ != nullptr && cli_->is_valid();
}

// 发起 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path) { return cli_->Get(path); }
// 发起带自定义头部的 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path, const Headers &headers) {
  return cli_->Get(path, headers);
}
// 发起带进度回调的 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path, Progress progress) {
  return cli_->Get(path, std::move(progress));
}
// 发起带自定义头部和进度回调的 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          Progress progress) {
  return cli_->Get(path, headers, std::move(progress));
}
// 发起带内容接收器的 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, std::move(content_receiver));
}
// 发起带自定义头部和内容接收器的 GET 请求，获取指定路径的结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(content_receiver));
}
# 使用给定路径进行 GET 请求，接收内容并返回结果
inline Result Client::Get(const std::string &path,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(content_receiver), std::move(progress));
}

# 使用给定路径和头部信息进行 GET 请求，接收内容并返回结果
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(content_receiver),
                   std::move(progress));
}

# 使用给定路径进行 GET 请求，处理响应并接收内容
inline Result Client::Get(const std::string &path,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, std::move(response_handler),
                   std::move(content_receiver));
}

# 使用给定路径和头部信息进行 GET 请求，处理响应并接收内容
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver) {
  return cli_->Get(path, headers, std::move(response_handler),
                   std::move(content_receiver));
}

# 使用给定路径进行 GET 请求，处理响应并接收内容，同时显示进度
inline Result Client::Get(const std::string &path,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, std::move(response_handler),
                   std::move(content_receiver), std::move(progress));
}

# 使用给定路径和头部信息进行 GET 请求，处理响应并接收内容，同时显示进度
inline Result Client::Get(const std::string &path, const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, headers, std::move(response_handler),
                   std::move(content_receiver), std::move(progress));
}

# 使用给定路径、参数和头部信息进行 GET 请求，同时显示进度
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers, Progress progress) {
  return cli_->Get(path, params, headers, progress);
}
// 发起 GET 请求，不带自定义响应处理函数
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, content_receiver, progress);
}
// 发起 GET 请求，带自定义响应处理函数
inline Result Client::Get(const std::string &path, const Params &params,
                          const Headers &headers,
                          ResponseHandler response_handler,
                          ContentReceiver content_receiver, Progress progress) {
  return cli_->Get(path, params, headers, response_handler, content_receiver,
                   progress);
}

// 发起 HEAD 请求，不带自定义头部
inline Result Client::Head(const std::string &path) { return cli_->Head(path); }
// 发起 HEAD 请求，带自定义头部
inline Result Client::Head(const std::string &path, const Headers &headers) {
  return cli_->Head(path, headers);
}

// 发起 POST 请求，不带自定义头部和请求体
inline Result Client::Post(const std::string &path) { return cli_->Post(path); }
// 发起 POST 请求，带自定义头部
inline Result Client::Post(const std::string &path, const Headers &headers) {
  return cli_->Post(path, headers);
}
// 发起 POST 请求，带自定义请求体长度和类型
inline Result Client::Post(const std::string &path, const char *body,
                           size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_length, content_type);
}
// 发起 POST 请求，带自定义头部、请求体长度和类型
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const char *body, size_t content_length,
                           const std::string &content_type) {
  return cli_->Post(path, headers, body, content_length, content_type);
}
// 发起 POST 请求，带自定义请求体和类型
inline Result Client::Post(const std::string &path, const std::string &body,
                           const std::string &content_type) {
  return cli_->Post(path, body, content_type);
}
# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const std::string &body,
                           const std::string &content_type) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, headers, body, content_type);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, size_t content_length,
                           ContentProvider content_provider,
                           const std::string &content_type) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, content_length, std::move(content_provider),
                    content_type);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, std::move(content_provider), content_type);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           size_t content_length,
                           ContentProvider content_provider,
                           const std::string &content_type) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, headers, content_length, std::move(content_provider),
                    content_type);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           ContentProviderWithoutLength content_provider,
                           const std::string &content_type) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, headers, std::move(content_provider), content_type);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, const Params &params) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, params);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const Params &params) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, headers, params);
}

# 使用 POST 方法向服务器发送请求，并返回结果
inline Result Client::Post(const std::string &path,
                           const MultipartFormDataItems &items) {
  # 调用底层 HTTP 客户端的 Post 方法，并返回结果
  return cli_->Post(path, items);
}
# 使用 HTTP POST 方法发送请求，包含路径、请求头和多部分表单数据项，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items) {
  return cli_->Post(path, headers, items);
}

# 使用 HTTP POST 方法发送请求，包含路径、请求头、多部分表单数据项和分隔符，返回结果
inline Result Client::Post(const std::string &path, const Headers &headers,
                           const MultipartFormDataItems &items,
                           const std::string &boundary) {
  return cli_->Post(path, headers, items, boundary);
}

# 使用 HTTP POST 方法发送请求，包含路径、请求头、多部分表单数据项和多部分表单数据提供者项，返回结果
inline Result
Client::Post(const std::string &path, const Headers &headers,
             const MultipartFormDataItems &items,
             const MultipartFormDataProviderItems &provider_items) {
  return cli_->Post(path, headers, items, provider_items);
}

# 使用 HTTP PUT 方法发送请求，只包含路径，返回结果
inline Result Client::Put(const std::string &path) { return cli_->Put(path); }

# 使用 HTTP PUT 方法发送请求，包含路径、请求体、内容长度和内容类型，返回结果
inline Result Client::Put(const std::string &path, const char *body,
                          size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_length, content_type);
}

# 使用 HTTP PUT 方法发送请求，包含路径、请求头、请求体、内容长度和内容类型，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const char *body, size_t content_length,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_length, content_type);
}

# 使用 HTTP PUT 方法发送请求，包含路径、请求体和内容类型，返回结果
inline Result Client::Put(const std::string &path, const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, body, content_type);
}

# 使用 HTTP PUT 方法发送请求，包含路径、请求头、请求体和内容类型，返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const std::string &body,
                          const std::string &content_type) {
  return cli_->Put(path, headers, body, content_type);
}
# 使用给定的路径、内容长度、内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, content_length, std::move(content_provider),
                   content_type);
}

# 使用给定的路径、不带长度的内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, std::move(content_provider), content_type);
}

# 使用给定的路径、头部、内容长度、内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          size_t content_length,
                          ContentProvider content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, content_length, std::move(content_provider),
                   content_type);
}

# 使用给定的路径、头部、不带长度的内容提供者和内容类型执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          ContentProviderWithoutLength content_provider,
                          const std::string &content_type) {
  return cli_->Put(path, headers, std::move(content_provider), content_type);
}

# 使用给定的路径和参数执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Params &params) {
  return cli_->Put(path, params);
}

# 使用给定的路径、头部和参数执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const Params &params) {
  return cli_->Put(path, headers, params);
}

# 使用给定的路径和多部分表单数据项执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, items);
}

# 使用给定的路径、头部和多部分表单数据项执行 PUT 请求，并返回结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items) {
  return cli_->Put(path, headers, items);
}
# 将数据上传到指定路径，使用给定的头部信息和多部分表单数据项，返回上传结果
inline Result Client::Put(const std::string &path, const Headers &headers,
                          const MultipartFormDataItems &items,
                          const std::string &boundary) {
  return cli_->Put(path, headers, items, boundary);
}

# 将数据上传到指定路径，使用给定的头部信息和多部分表单数据项提供者，返回上传结果
inline Result
Client::Put(const std::string &path, const Headers &headers,
            const MultipartFormDataItems &items,
            const MultipartFormDataProviderItems &provider_items) {
  return cli_->Put(path, headers, items, provider_items);
}

# 对指定路径执行部分更新操作，返回更新结果
inline Result Client::Patch(const std::string &path) {
  return cli_->Patch(path);
}

# 对指定路径执行部分更新操作，使用给定的内容和长度，返回更新结果
inline Result Client::Patch(const std::string &path, const char *body,
                            size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_length, content_type);
}

# 对指定路径执行部分更新操作，使用给定的头部信息、内容和长度，返回更新结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const char *body, size_t content_length,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_length, content_type);
}

# 对指定路径执行部分更新操作，使用给定的内容和内容类型，返回更新结果
inline Result Client::Patch(const std::string &path, const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, body, content_type);
}

# 对指定路径执行部分更新操作，使用给定的头部信息、内容和内容类型，返回更新结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            const std::string &body,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, body, content_type);
}

# 对指定路径执行部分更新操作，使用给定的内容长度、内容提供者和内容类型，返回更新结果
inline Result Client::Patch(const std::string &path, size_t content_length,
                            ContentProvider content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, content_length, std::move(content_provider),
                     content_type);
}
# 使用给定的路径、内容提供器和内容类型执行 PATCH 请求，并返回结果
inline Result Client::Patch(const std::string &path,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, std::move(content_provider), content_type);
}

# 使用给定的路径、头部、内容长度、内容提供器和内容类型执行 PATCH 请求，并返回结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            size_t content_length,
                            ContentProvider content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, content_length, std::move(content_provider),
                     content_type);
}

# 使用给定的路径、头部、无长度内容提供器和内容类型执行 PATCH 请求，并返回结果
inline Result Client::Patch(const std::string &path, const Headers &headers,
                            ContentProviderWithoutLength content_provider,
                            const std::string &content_type) {
  return cli_->Patch(path, headers, std::move(content_provider), content_type);
}

# 使用给定的路径执行 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path) {
  return cli_->Delete(path);
}

# 使用给定的路径和头部执行 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path, const Headers &headers) {
  return cli_->Delete(path, headers);
}

# 使用给定的路径、内容、内容长度和内容类型执行 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path, const char *body,
                             size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_length, content_type);
}

# 使用给定的路径、头部、内容、内容长度和内容类型执行 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const char *body, size_t content_length,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_length, content_type);
}

# 使用给定的路径、内容和内容类型执行 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path, const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, body, content_type);
}
# 使用给定路径、头部、请求体和内容类型发送 DELETE 请求，并返回结果
inline Result Client::Delete(const std::string &path, const Headers &headers,
                             const std::string &body,
                             const std::string &content_type) {
  return cli_->Delete(path, headers, body, content_type);
}

# 发送 OPTIONS 请求，并返回结果
inline Result Client::Options(const std::string &path) {
  return cli_->Options(path);
}

# 使用给定路径和头部发送 OPTIONS 请求，并返回结果
inline Result Client::Options(const std::string &path, const Headers &headers) {
  return cli_->Options(path, headers);
}

# 发送请求并接收响应
inline bool Client::send(Request &req, Response &res, Error &error) {
  return cli_->send(req, res, error);
}

# 发送请求并返回结果
inline Result Client::send(const Request &req) { return cli_->send(req); }

# 返回套接字是否打开
inline size_t Client::is_socket_open() const { return cli_->is_socket_open(); }

# 返回套接字
inline socket_t Client::socket() const { return cli_->socket(); }

# 停止客户端
inline void Client::stop() { cli_->stop(); }

# 设置主机名地址映射
inline void
Client::set_hostname_addr_map(std::map<std::string, std::string> addr_map) {
  cli_->set_hostname_addr_map(std::move(addr_map));
}

# 设置默认头部
inline void Client::set_default_headers(Headers headers) {
  cli_->set_default_headers(std::move(headers));
}

# 设置地址族
inline void Client::set_address_family(int family) {
  cli_->set_address_family(family);
}

# 设置 TCP 禁用延迟
inline void Client::set_tcp_nodelay(bool on) { cli_->set_tcp_nodelay(on); }

# 设置套接字选项
inline void Client::set_socket_options(SocketOptions socket_options) {
  cli_->set_socket_options(std::move(socket_options));
}

# 设置连接超时
inline void Client::set_connection_timeout(time_t sec, time_t usec) {
  cli_->set_connection_timeout(sec, usec);
}

# 设置读取超时
inline void Client::set_read_timeout(time_t sec, time_t usec) {
  cli_->set_read_timeout(sec, usec);
}

# 设置写入超时
inline void Client::set_write_timeout(time_t sec, time_t usec) {
  cli_->set_write_timeout(sec, usec);
}

# 设置基本认证
inline void Client::set_basic_auth(const std::string &username,
                                   const std::string &password) {
  cli_->set_basic_auth(username, password);
}
// 设置使用 Bearer Token 进行身份验证
inline void Client::set_bearer_token_auth(const std::string &token) {
  cli_->set_bearer_token_auth(token);
}
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置使用 Digest 身份验证
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

// 设置代理
inline void Client::set_proxy(const std::string &host, int port) {
  cli_->set_proxy(host, port);
}
// 设置代理的基本身份验证
inline void Client::set_proxy_basic_auth(const std::string &username,
                                         const std::string &password) {
  cli_->set_proxy_basic_auth(username, password);
}
// 设置代理的 Bearer Token 身份验证
inline void Client::set_proxy_bearer_token_auth(const std::string &token) {
  cli_->set_proxy_bearer_token_auth(token);
}
#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置代理的 Digest 身份验证
inline void Client::set_proxy_digest_auth(const std::string &username,
                                          const std::string &password) {
  cli_->set_proxy_digest_auth(username, password);
}
#endif

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 启用服务器证书验证
inline void Client::enable_server_certificate_verification(bool enabled) {
  cli_->enable_server_certificate_verification(enabled);
}
#endif

// 设置日志记录器
inline void Client::set_logger(Logger logger) { cli_->set_logger(logger); }

#ifdef CPPHTTPLIB_OPENSSL_SUPPORT
// 设置 CA 证书路径
inline void Client::set_ca_cert_path(const std::string &ca_cert_file_path,
                                     const std::string &ca_cert_dir_path) {
  cli_->set_ca_cert_path(ca_cert_file_path, ca_cert_dir_path);
}
// 设置 CA 证书存储，如果是 SSL 连接，则调用 SSLClient 对象的 set_ca_cert_store 方法，否则调用 cli_ 对象的 set_ca_cert_store 方法
inline void Client::set_ca_cert_store(X509_STORE *ca_cert_store) {
  if (is_ssl_) {
    static_cast<SSLClient &>(*cli_).set_ca_cert_store(ca_cert_store);
  } else {
    cli_->set_ca_cert_store(ca_cert_store);
  }
}

// 获取 OpenSSL 验证结果，如果是 SSL 连接，则调用 SSLClient 对象的 get_openssl_verify_result 方法，否则返回 -1
inline long Client::get_openssl_verify_result() const {
  if (is_ssl_) {
    return static_cast<SSLClient &>(*cli_).get_openssl_verify_result();
  }
  return -1; // 注意：-1 不匹配任何 X509_V_ERR_???
}

// 获取 SSL 上下文，如果是 SSL 连接，则返回 SSLClient 对象的 ssl_context，否则返回 nullptr
inline SSL_CTX *Client::ssl_context() const {
  if (is_ssl_) { return static_cast<SSLClient &>(*cli_).ssl_context(); }
  return nullptr;
}
#endif

// ----------------------------------------------------------------------------

} // namespace httplib

// 如果在 Windows 下并且定义了 CPPHTTPLIB_USE_POLL，则取消定义 poll
#if defined(_WIN32) && defined(CPPHTTPLIB_USE_POLL)
#undef poll
#endif

#endif // CPPHTTPLIB_HTTPLIB_H
```