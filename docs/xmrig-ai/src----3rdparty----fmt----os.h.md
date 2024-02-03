# `xmrig\src\3rdparty\fmt\os.h`

```cpp
// C++格式化库的操作系统特定功能
//
// 版权所有，Victor Zverovich，2012年至今
// 保留所有权利
//
// 有关许可信息，请参阅format.h。

#ifndef FMT_OS_H_
#define FMT_OS_H_

#if defined(__MINGW32__) || defined(__CYGWIN__)
// 解决MinGW bug https://sourceforge.net/p/mingw/bugs/2024/。
#  undef __STRICT_ANSI__
#endif

#include <cerrno>
#include <clocale>  // 用于locale_t
#include <cstddef>
#include <cstdio>
#include <cstdlib>  // 用于strtod_l

#if defined __APPLE__ || defined(__FreeBSD__)
#  include <xlocale.h>  // 用于OS X上的LC_NUMERIC_MASK
#endif

#include "format.h"

// UWP不提供_pipe。
#if FMT_HAS_INCLUDE("winapifamily.h")
#  include <winapifamily.h>
#endif
#if (FMT_HAS_INCLUDE(<fcntl.h>) || defined(__APPLE__) || \
     defined(__linux__)) &&                              \
    (!defined(WINAPI_FAMILY) || (WINAPI_FAMILY == WINAPI_FAMILY_DESKTOP_APP))
#  include <fcntl.h>  // 用于O_RDONLY
#  define FMT_USE_FCNTL 1
#else
#  define FMT_USE_FCNTL 0
#endif

#ifndef FMT_POSIX
#  if defined(_WIN32) && !defined(__MINGW32__)
// 修复关于废弃符号的警告。
#    define FMT_POSIX(call) _##call
#  else
#    define FMT_POSIX(call) call
#  endif
#endif

// 对系统函数的调用在FMT_SYSTEM中进行包装，以便进行测试。
#ifdef FMT_SYSTEM
#  define FMT_POSIX_CALL(call) FMT_SYSTEM(call)
#else
#  define FMT_SYSTEM(call) ::call
#  ifdef _WIN32
// 修复关于废弃符号的警告。
#    define FMT_POSIX_CALL(call) ::_##call
#  else
#    define FMT_POSIX_CALL(call) ::call
#  endif
#endif

// 当表达式求值为error_result并且errno等于EINTR时，重试表达式。
#ifndef _WIN32
#  define FMT_RETRY_VAL(result, expression, error_result) \
    do {                                                  \
      (result) = (expression);                            \
    } while ((result) == (error_result) && errno == EINTR)
#else
// 定义宏，用于将表达式的结果赋给result，如果表达式失败，则返回error_result
#define FMT_RETRY_VAL(result, expression, error_result) result = (expression)
#endif

// 定义宏，用于将表达式的结果赋给result，如果表达式失败，则返回-1
#define FMT_RETRY(result, expression) FMT_RETRY_VAL(result, expression, -1)

FMT_BEGIN_NAMESPACE

/**
  \rst
  对空结尾字符串的引用。可以从C字符串或``std::string``构造。

  您可以使用以下常见字符类型的类型别名之一：

  +---------------+-----------------------------+
  | 类型          | 定义                        |
  +===============+=============================+
  | cstring_view  | basic_cstring_view<char>    |
  +---------------+-----------------------------+
  | wcstring_view | basic_cstring_view<wchar_t> |
  +---------------+-----------------------------+

  此类最有用作参数类型，以允许将不同类型的字符串传递给函数，例如::

    template <typename... Args>
    std::string format(cstring_view format_str, const Args & ... args);

    format("{}", 42);
    format(std::string("{}"), 42);
  \endrst
 */
template <typename Char> class basic_cstring_view {
 private:
  const Char* data_;

 public:
  /** 从C字符串构造一个字符串引用对象。 */
  basic_cstring_view(const Char* s) : data_(s) {}

  /**
    \rst
    从``std::string``对象构造一个字符串引用。
    \endrst
   */
  basic_cstring_view(const std::basic_string<Char>& s) : data_(s.c_str()) {}

  /** 返回指向C字符串的指针。 */
  const Char* c_str() const { return data_; }
};

using cstring_view = basic_cstring_view<char>;
using wcstring_view = basic_cstring_view<wchar_t>;

// 错误代码。
class error_code {
 private:
  int value_;

 public:
  explicit error_code(int value = 0) FMT_NOEXCEPT : value_(value) {}

  int get() const FMT_NOEXCEPT { return value_; }
};

#ifdef _WIN32
namespace detail {
// 从UTF-16转换为UTF-8的转换器。
// 仅为Windows提供，因为其他系统原生支持UTF-8。
// 定义一个名为 utf16_to_utf8 的类
class utf16_to_utf8 {
 private:
  // 声明一个内存缓冲区对象
  memory_buffer buffer_;

 public:
  // 默认构造函数
  utf16_to_utf8() {}
  // 显式构造函数，接受一个 wstring_view 对象作为参数
  FMT_API explicit utf16_to_utf8(wstring_view s);
  // 类型转换函数，将对象转换为 string_view 类型
  operator string_view() const { return string_view(&buffer_[0], size()); }
  // 返回缓冲区大小
  size_t size() const { return buffer_.size() - 1; }
  // 返回指向缓冲区的指针
  const char* c_str() const { return &buffer_[0]; }
  // 返回缓冲区内容的字符串表示
  std::string str() const { return std::string(&buffer_[0], size()); }

  // 执行转换，返回系统错误代码而不是在转换错误时抛出异常
  // 在内存分配错误的情况下，此方法仍可能抛出异常
  FMT_API int convert(wstring_view s);
};

// 格式化 Windows 错误信息并将结果存储在输出缓冲区中
FMT_API void format_windows_error(buffer<char>& out, int error_code,
                                  string_view message) FMT_NOEXCEPT;
}  // namespace detail

/** 一个 Windows 错误。*/
class windows_error : public system_error {
 private:
  FMT_API void init(int error_code, string_view format_str, format_args args);

 public:
  /**
   \rst
   Constructs a :class:`fmt::windows_error` object with the description
   of the form

   .. parsed-literal::
     *<message>*: *<system-message>*

   where *<message>* is the formatted message and *<system-message>* is the
   system message corresponding to the error code.
   *error_code* is a Windows error code as given by ``GetLastError``.
   If *error_code* is not a valid error code such as -1, the system message
   will look like "error -1".

   **Example**::

     // This throws a windows_error with the description
     //   cannot open file 'madeup': The system cannot find the file specified.
     // or similar (system message may vary).
     const char *filename = "madeup";
     LPOFSTRUCT of = LPOFSTRUCT();
     HFILE file = OpenFile(filename, &of, OF_READ);
     if (file == HFILE_ERROR) {
       throw fmt::windows_error(GetLastError(),
                                "cannot open file '{}'", filename);
     }
   \endrst
  */
  template <typename... Args>
  // 使用给定的错误码、消息和参数构造 windows_error 对象
  windows_error(int error_code, string_view message, const Args&... args) {
    // 调用 init 方法进行初始化
    init(error_code, message, make_format_args(args...));
  }
};

// 报告 Windows 错误，但不抛出异常
// 可用于在析构函数中报告错误
FMT_API void report_windows_error(int error_code,
                                  string_view message) FMT_NOEXCEPT;
#endif  // _WIN32

// 一个带缓冲的文件
// 定义一个名为buffered_file的类
class buffered_file {
 private:
  FILE* file_;  // 文件指针

  friend class file;  // file类是buffered_file的友元类

  explicit buffered_file(FILE* f) : file_(f) {}  // 显式构造函数，接受一个文件指针作为参数，用于初始化file_

 public:
  buffered_file(const buffered_file&) = delete;  // 禁用拷贝构造函数
  void operator=(const buffered_file&) = delete;  // 禁用赋值运算符重载

  // 构造一个不代表任何文件的buffered_file对象
  buffered_file() FMT_NOEXCEPT : file_(nullptr) {}

  // 销毁对象，关闭它所代表的文件（如果有的话）
  FMT_API ~buffered_file() FMT_NOEXCEPT;

 public:
  // 移动构造函数，接受另一个buffered_file对象的引用，并将其资源移动到当前对象
  buffered_file(buffered_file&& other) FMT_NOEXCEPT : file_(other.file_) {
    other.file_ = nullptr;
  }

  // 移动赋值运算符，接受另一个buffered_file对象的引用，并将其资源移动到当前对象
  buffered_file& operator=(buffered_file&& other) {
    close();  // 关闭当前对象代表的文件
    file_ = other.file_;  // 将另一个对象的文件指针赋值给当前对象
    other.file_ = nullptr;  // 将另一个对象的文件指针置空
    return *this;  // 返回当前对象
  }

  // 打开一个文件
  FMT_API buffered_file(cstring_view filename, cstring_view mode);

  // 关闭文件
  FMT_API void close();

  // 返回表示该文件的FILE对象的指针
  FILE* get() const FMT_NOEXCEPT { return file_; }

  // 在fileno周围放置括号，以解决某些MinGW版本中fileno被定义为宏的问题
  FMT_API int(fileno)() const;

  // 使用给定的格式字符串和参数在文件中打印内容
  void vprint(string_view format_str, format_args args) {
    fmt::vprint(file_, format_str, args);
  }

  // 使用给定的格式字符串和参数在文件中打印内容
  template <typename... Args>
  inline void print(string_view format_str, const Args&... args) {
    vprint(format_str, make_format_args(args...));
  }
};

#if FMT_USE_FCNTL
// 一个文件。关闭的文件由具有描述符-1的文件对象表示。
// 未声明为FMT_NOEXCEPT的方法可能在失败时抛出fmt::system_error异常。
// 请注意，某些错误（例如多次关闭文件）在Windows上会导致崩溃而不是异常。
// 您可以通过使用_set_invalid_parameter_handler来覆盖无效参数处理程序来获得标准行为。
class file {
 private:
  int fd_;  // 文件描述符

  // 使用给定描述符构造文件对象
  explicit file(int fd) : fd_(fd) {}

 public:
  // 构造函数的oflag参数的可能取值
  enum {
    RDONLY = FMT_POSIX(O_RDONLY),  // 只读模式打开文件
    WRONLY = FMT_POSIX(O_WRONLY),  // 只写模式打开文件
    RDWR = FMT_POSIX(O_RDWR),      // 读写模式打开文件
    CREATE = FMT_POSIX(O_CREAT),   // 如果文件不存在则创建
    APPEND = FMT_POSIX(O_APPEND)   // 以追加模式打开文件
  };

  // 构造一个不表示任何文件的文件对象
  file() FMT_NOEXCEPT : fd_(-1) {}

  // 打开文件并构造一个表示该文件的文件对象
  FMT_API file(cstring_view path, int oflag);

 public:
  file(const file&) = delete;
  void operator=(const file&) = delete;

  file(file&& other) FMT_NOEXCEPT : fd_(other.fd_) { other.fd_ = -1; }

  file& operator=(file&& other) FMT_NOEXCEPT {
    close();
    fd_ = other.fd_;
    other.fd_ = -1;
  }
    // 返回当前对象的引用
    return *this;
  }

  // 销毁对象，关闭它所代表的文件（如果有的话）
  FMT_API ~file() FMT_NOEXCEPT;

  // 返回文件描述符
  int descriptor() const FMT_NOEXCEPT { return fd_; }

  // 关闭文件
  FMT_API void close();

  // 返回文件大小。为了与 stat::st_size 保持一致，大小具有有符号类型
  FMT_API long long size() const;

  // 尝试从文件中读取 count 个字节到指定的缓冲区
  FMT_API size_t read(void* buffer, size_t count);

  // 尝试将指定缓冲区中的 count 个字节写入文件
  FMT_API size_t write(const void* buffer, size_t count);

  // 使用 dup 函数复制文件描述符，并将复制后的文件描述符作为文件对象返回
  FMT_API static file dup(int fd);

  // 使 fd 成为该文件描述符的副本，如果需要的话，先关闭 fd
  FMT_API void dup2(int fd);

  // 使 fd 成为该文件描述符的副本，如果需要的话，先关闭 fd，并在出现错误时将错误信息存储在 error_code 中
  FMT_API void dup2(int fd, error_code& ec) FMT_NOEXCEPT;

  // 创建一个管道，为读取和写入分别设置 read_end 和 write_end 文件对象
  FMT_API static void pipe(file& read_end, file& write_end);

  // 创建一个与该文件关联的 buffered_file 对象，并将该文件对象从文件中分离
  FMT_API buffered_file fdopen(const char* mode);
// 结构体 buffer_size 用于存储缓冲区大小
namespace detail {

// 结构体 buffer_size 用于存储缓冲区大小
struct buffer_size {
  size_t value = 0;
  // 重载赋值运算符，将传入的值赋给 buffer_size 的 value 成员
  buffer_size operator=(size_t val) const {
    auto bs = buffer_size();
    bs.value = val;
    return bs;
  }
};

// 结构体 ostream_params 用于存储输出流的参数
struct ostream_params {
  int oflag = file::WRONLY | file::CREATE; // 默认的输出标志
  size_t buffer_size = BUFSIZ > 32768 ? BUFSIZ : 32768; // 默认的缓冲区大小

  // 默认构造函数
  ostream_params() {}

  // 模板构造函数，支持传入输出标志
  template <typename... T>
  ostream_params(T... params, int oflag) : ostream_params(params...) {
    this->oflag = oflag;
  }

  // 模板构造函数，支持传入缓冲区大小
  template <typename... T>
  ostream_params(T... params, detail::buffer_size bs)
      : ostream_params(params...) {
    this->buffer_size = bs.value;
  }
};
}  // namespace detail

// 定义常量 buffer_size，用于设置缓冲区大小
static constexpr detail::buffer_size buffer_size;

// 输出流类，继承自 detail::buffer<char>
class ostream : private detail::buffer<char> {
 private:
  file file_;

  // 刷新缓冲区
  void flush() {
    if (size() == 0) return;
    file_.write(data(), size());
    clear();
  }

  // 扩展缓冲区大小
  void grow(size_t) final;

  // 构造函数，打开文件并设置输出流参数
  ostream(cstring_view path, const detail::ostream_params& params)
      : file_(path, params.oflag) {
    set(new char[params.buffer_size], params.buffer_size);
  }

 public:
  // 移动构造函数
  ostream(ostream&& other)
      : detail::buffer<char>(other.data(), other.size(), other.capacity()),
        file_(std::move(other.file_)) {
    other.set(nullptr, 0);
  }
  // 析构函数，刷新缓冲区并释放内存
  ~ostream() {
    flush();
    delete[] data();
  }

  // 友元函数，用于创建输出文件流
  template <typename... T>
  friend ostream output_file(cstring_view path, T... params);

  // 关闭输出流
  void close() {
    flush();
    file_.close();
  }

  // 打印函数，将格式化字符串和参数写入输出流
  template <typename S, typename... Args>
  void print(const S& format_str, const Args&... args) {
    format_to(detail::buffer_appender<char>(*this), format_str, args...);
  }
};

/**
  打开文件进行写操作，支持传入的参数:
  * ``<integer>``: 输出标志（默认为 ``file::WRONLY | file::CREATE``）
  * ``buffer_size=<integer>``: 输出缓冲区大小
 */
template <typename... T>
#ifdef FMT_USE_FCNTL
// 如果定义了 FMT_USE_FCNTL 宏，则定义一个输出文件的函数，接受文件路径和参数
inline ostream output_file(cstring_view path, T... params) {
  return {path, detail::ostream_params(params...)};
}
#endif  // FMT_USE_FCNTL

#ifdef FMT_LOCALE
// 如果定义了 FMT_LOCALE 宏，则定义一个“C”数字区域设置类
class locale {
 private:
#  ifdef _WIN32
  using locale_t = _locale_t;

  // 释放区域设置
  static void freelocale(locale_t loc) { _free_locale(loc); }

  // 将字符串转换为浮点数
  static double strtod_l(const char* nptr, char** endptr, _locale_t loc) {
    return _strtod_l(nptr, endptr, loc);
  }
#  endif

  locale_t locale_;

 public:
  using type = locale_t;
  // 禁用拷贝构造函数和赋值运算符
  locale(const locale&) = delete;
  void operator=(const locale&) = delete;

  // 构造函数，创建一个“C”数字区域设置
  locale() {
#  ifndef _WIN32
    locale_ = FMT_SYSTEM(newlocale(LC_NUMERIC_MASK, "C", nullptr));
#  else
    locale_ = _create_locale(LC_NUMERIC, "C");
#  endif
    // 如果创建失败，抛出系统错误
    if (!locale_) FMT_THROW(system_error(errno, "cannot create locale"));
  }
  // 析构函数，释放区域设置
  ~locale() { freelocale(locale_); }

  // 获取区域设置
  type get() const { return locale_; }

  // 将字符串转换为浮点数，并更新字符串指针
  double strtod(const char*& str) const {
    char* end = nullptr;
    double result = strtod_l(str, &end, locale_);
    str = end;
    return result;
  }
};
using Locale FMT_DEPRECATED_ALIAS = locale;
#endif  // FMT_LOCALE
FMT_END_NAMESPACE

#endif  // FMT_OS_H_
```