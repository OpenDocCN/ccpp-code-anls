# `nmap\libz\contrib\iostream2\zstream.h`

```cpp
/*
 * 版权声明
 * 1997年
 * Christian Michelsen Research AS
 * Advanced Computing
 * Fantoftvegen 38, 5036 BERGEN, Norway
 * http://www.cmr.no
 *
 * 允许在任何目的下使用、复制、修改、分发和出售此软件及其文档，而不收取费用
 * 前提是上述版权声明出现在所有副本中，并且支持文档中同时出现版权声明和此许可声明
 * Christian Michelsen Research AS对此软件的适用性不作任何陈述。它是"按原样"提供，没有明示或暗示的保证。
 */

#ifndef ZSTREAM__H
#define ZSTREAM__H

/*
 * zstream.h - C++接口到'zlib'通用压缩库
 * $Id: zstream.h 1.1 1997-06-25 12:00:56+02 tyge Exp tyge $
 */

#include <strstream.h>
#include <string.h>
#include <stdio.h>
#include "zlib.h"

#if defined(_WIN32)
#   include <fcntl.h>
#   include <io.h>
#   define SET_BINARY_MODE(file) setmode(fileno(file), O_BINARY)
#else
#   define SET_BINARY_MODE(file)
#endif

class zstringlen {
public:
    zstringlen(class izstream&);
    zstringlen(class ozstream&, const char*);
    size_t value() const { return val.word; }
private:
    struct Val { unsigned char byte; size_t word; } val;
};

//  ----------------------------- izstream -----------------------------

class izstream
*/
    // 默认构造函数，初始化文件指针为0
    izstream() : m_fp(0) {}
    // 以文件指针作为参数的构造函数，初始化文件指针为0，并打开文件
    izstream(FILE* fp) : m_fp(0) { open(fp); }
    // 以文件名作为参数的构造函数，初始化文件指针为0，并打开文件
    izstream(const char* name) : m_fp(0) { open(name); }
    // 析构函数，关闭文件
    ~izstream() { close(); }

    /* 打开一个 gzip (.gz) 文件进行读取。
     * open() 可以用于读取非 gzip 格式的文件；
     * 在这种情况下，read() 将直接从文件中读取而不进行解压缩。
     * 可以检查 errno 来区分两种错误情况（如果 errno 为零，则 zlib 错误为 Z_MEM_ERROR）。
     */
    void open(const char* name) {
        // 如果文件指针存在，则关闭文件
        if (m_fp) close();
        // 以只读方式打开文件
        m_fp = ::gzopen(name, "rb");
    }

    // 以文件指针作为参数打开文件
    void open(FILE* fp) {
        // 设置文件为二进制模式
        SET_BINARY_MODE(fp);
        // 如果文件指针存在，则关闭文件
        if (m_fp) close();
        // 以只读二进制模式打开文件
        m_fp = ::gzdopen(fileno(fp), "rb");
    }

    /* 如果需要，刷新所有挂起的输入，关闭压缩文件
     * 并释放所有（解）压缩状态。返回值是 zlib 错误号（参见下面的 error() 函数）。
     */
    int close() {
        // 关闭文件，并返回 zlib 错误号
        int r = ::gzclose(m_fp);
        // 将文件指针置为0
        m_fp = 0; return r;
    }

    /* 从压缩文件中二进制读取给定数量的字节。
     */
    int read(void* buf, size_t len) {
        // 从压缩文件中读取数据
        return ::gzread(m_fp, buf, len);
    }

    /* 返回给定压缩文件上发生的最后一个错误的错误消息。
     * errnum 被设置为 zlib 错误号。如果在文件系统中发生错误而不是在压缩库中，
     * errnum 被设置为 Z_ERRNO，应用程序可以查看 errno 来获取确切的错误代码。
     */
    const char* error(int* errnum) {
        // 返回给定压缩文件上发生的最后一个错误的错误消息
        return ::gzerror(m_fp, errnum);
    }

    // 返回文件指针
    gzFile fp() { return m_fp; }

private:
    // 文件指针
    gzFile m_fp;
};

/*
 * 从压缩文件中二进制读取给定对象（数组）。
 * 如果输入文件不是gzip格式，read()将对象的字节数复制到缓冲区。
 * 返回实际读取的未压缩字节数（文件结束返回0，错误返回-1）。
 */
template <class T, class Items>
inline int read(izstream& zs, T* x, Items items) {
    return ::gzread(zs.fp(), x, items*sizeof(T));
}

/*
 * 使用'>'运算符进行二进制输入。
 */
template <class T>
inline izstream& operator>(izstream& zs, T& x) {
    ::gzread(zs.fp(), &x, sizeof(T));
    return zs;
}


inline zstringlen::zstringlen(izstream& zs) {
    zs > val.byte;
    if (val.byte == 255) zs > val.word;
    else val.word = val.byte;
}

/*
 * 使用'>'运算符读取字符串长度和字符串。
 */
inline izstream& operator>(izstream& zs, char* x) {
    zstringlen len(zs);
    ::gzread(zs.fp(), x, len.value());
    x[len.value()] = '\0';
    return zs;
}

inline char* read_string(izstream& zs) {
    zstringlen len(zs);
    char* x = new char[len.value()+1];
    ::gzread(zs.fp(), x, len.value());
    x[len.value()] = '\0';
    return x;
}

// ----------------------------- ozstream -----------------------------

class ozstream
{
    private:
        gzFile m_fp;
        ostrstream* m_os;
};

/*
 * 将给定对象（数组）以二进制形式写入压缩文件。
 * 返回实际写入的未压缩字节数（出错时返回0）。
 */
template <class T, class Items>
inline int write(ozstream& zs, const T* x, Items items) {
    return ::gzwrite(zs.fp(), (voidp) x, items*sizeof(T));
}

/*
 * 使用'<'运算符进行二进制输出。
 */
template <class T>
inline ozstream& operator<(ozstream& zs, const T& x) {
    ::gzwrite(zs.fp(), (voidp) &x, sizeof(T));
    return zs;
}

inline zstringlen::zstringlen(ozstream& zs, const char* x) {
    val.byte = 255;  val.word = ::strlen(x);
    if (val.word < 255) zs < (val.byte = val.word);
    else zs < val;
}
/*
 * 使用 '<' 运算符写入字符串的长度和字符串内容。
 */
inline ozstream& operator<(ozstream& zs, const char* x) {
    // 创建 zstringlen 对象，用于获取字符串长度
    zstringlen len(zs, x);
    // 使用 ::gzwrite 写入字符串内容到文件流
    ::gzwrite(zs.fp(), (voidp) x, len.value());
    // 返回文件流对象
    return zs;
}

#ifdef _MSC_VER
// 如果是 Microsoft Visual C++ 编译器，则重载 '<' 运算符
inline ozstream& operator<(ozstream& zs, char* const& x) {
    // 调用前一个重载的 '<' 运算符
    return zs < (const char*) x;
}
#endif

/*
 * 使用 << 运算符进行 ASCII 写入
 */
template <class T>
inline ostream& operator<<(ozstream& zs, const T& x) {
    // 刷新文件流
    zs.os_flush();
    // 使用 << 运算符写入内容到文件流
    return zs.os() << x;
}

#endif
```