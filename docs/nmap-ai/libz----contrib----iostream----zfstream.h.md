# `nmap\libz\contrib\iostream\zfstream.h`

```
#ifndef zfstream_h
#define zfstream_h

#include <fstream.h>
#include "zlib.h"

class gzfilebuf : public streambuf {

public:

  gzfilebuf( ); // 默认构造函数
  virtual ~gzfilebuf(); // 虚析构函数

  gzfilebuf *open( const char *name, int io_mode ); // 打开指定文件
  gzfilebuf *attach( int file_descriptor, int io_mode ); // 附加到指定文件描述符
  gzfilebuf *close(); // 关闭文件

  int setcompressionlevel( int comp_level ); // 设置压缩级别
  int setcompressionstrategy( int comp_strategy ); // 设置压缩策略

  inline int is_open() const { return (file !=NULL); } // 判断文件是否打开

  virtual streampos seekoff( streamoff, ios::seek_dir, int ); // 定位文件指针

  virtual int sync(); // 同步缓冲区

protected:

  virtual int underflow(); // 读取缓冲区数据
  virtual int overflow( int = EOF ); // 写入缓冲区数据

private:

  gzFile file; // 文件指针
  short mode; // 文件模式
  short own_file_descriptor; // 文件描述符

  int flushbuf(); // 刷新缓冲区
  int fillbuf(); // 填充缓冲区

};

class gzfilestream_common : virtual public ios {

  friend class gzifstream;
  friend class gzofstream;
  friend gzofstream &setcompressionlevel( gzofstream &, int );
  friend gzofstream &setcompressionstrategy( gzofstream &, int );

public:
  virtual ~gzfilestream_common(); // 虚析构函数

  void attach( int fd, int io_mode ); // 附加到指定文件描述符
  void open( const char *name, int io_mode ); // 打开指定文件
  void close(); // 关闭文件

protected:
  gzfilestream_common(); // 默认构造函数

private:
  gzfilebuf *rdbuf(); // 返回文件缓冲区

  gzfilebuf buffer; // 文件缓冲区

};

class gzifstream : public gzfilestream_common, public istream {

public:

  gzifstream(); // 默认构造函数
  gzifstream( const char *name, int io_mode = ios::in ); // 根据文件名和模式构造
  gzifstream( int fd, int io_mode = ios::in ); // 根据文件描述符和模式构造

  virtual ~gzifstream(); // 虚析构函数

};

class gzofstream : public gzfilestream_common, public ostream {

public:

  gzofstream(); // 默认构造函数
  gzofstream( const char *name, int io_mode = ios::out ); // 根据文件名和模式构造
  gzofstream( int fd, int io_mode = ios::out ); // 根据文件描述符和模式构造

  virtual ~gzofstream(); // 虚析构函数

};

template<class T> class gzomanip {
  friend gzofstream &operator<<(gzofstream &, const gzomanip<T> &);
public:
  gzomanip(gzofstream &(*f)(gzofstream &, T), T v) : func(f), val(v) { } // 构造函数
private:
  gzofstream &(*func)(gzofstream &, T); // 函数指针
  T val; // 值
};

template<class T> gzofstream &operator<<(gzofstream &s, const gzomanip<T> &m) // 重载运算符<<
{
  // 调用函数指针指向的函数，并返回结果
  return (*m.func)(s, m.val);
}

// 设置输出流的压缩级别
inline gzofstream &setcompressionlevel( gzofstream &s, int l )
{
  // 调用输出流的底层缓冲区对象的方法，设置压缩级别
  (s.rdbuf())->setcompressionlevel(l);
  // 返回输出流对象
  return s;
}

// 设置输出流的压缩策略
inline gzofstream &setcompressionstrategy( gzofstream &s, int l )
{
  // 调用输出流的底层缓冲区对象的方法，设置压缩策略
  (s.rdbuf())->setcompressionstrategy(l);
  // 返回输出流对象
  return s;
}

// 创建一个用于设置压缩级别的自定义流操作符
inline gzomanip<int> setcompressionlevel(int l)
{
  // 返回一个自定义流操作符对象，指向设置压缩级别的函数
  return gzomanip<int>(&setcompressionlevel,l);
}

// 创建一个用于设置压缩策略的自定义流操作符
inline gzomanip<int> setcompressionstrategy(int l)
{
  // 返回一个自定义流操作符对象，指向设置压缩策略的函数
  return gzomanip<int>(&setcompressionstrategy,l);
}

// 结束条件编译指令
#endif
```