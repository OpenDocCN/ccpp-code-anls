# `nmap\libz\contrib\iostream3\zfstream.h`

```cpp
/*
 * A C++ I/O streams interface to the zlib gz* functions
 *
 * by Ludwig Schwardt <schwardt@sun.ac.za>
 * original version by Kevin Ruland <kevin@rodin.wustl.edu>
 *
 * This version is standard-compliant and compatible with gcc 3.x.
 */

#ifndef ZFSTREAM_H
#define ZFSTREAM_H

#include <istream>  // not iostream, since we don't need cin/cout
#include <ostream>
#include "zlib.h"

/*****************************************************************************/

/**
 *  @brief  Gzipped file stream buffer class.
 *
 *  This class implements basic_filebuf for gzipped files. It doesn't yet support
 *  seeking (allowed by zlib but slow/limited), putback and read/write access
 *  (tricky). Otherwise, it attempts to be a drop-in replacement for the standard
 *  file streambuf.
*/
class gzfilebuf : public std::streambuf
{
public:
  // 默认构造函数
  gzfilebuf();

  // 析构函数
  virtual
  ~gzfilebuf();

  /**
   *  @brief  在运行时设置压缩级别和策略
   *  @param  comp_level  压缩级别（参见 zlib.h 中允许的值）
   *  @param  comp_strategy  压缩策略（参见 zlib.h 中允许的值）
   *  @return  成功返回 Z_OK，否则返回 Z_STREAM_ERROR
   *
   *  不幸的是，这些参数不能分别修改，就像之前的 zfstream 版本假设的那样。由于策略很少改变，它可以默认设置，然后 setcompression(level) 就变成了旧的 setcompressionlevel(level)。
  */
  int
  setcompression(int comp_level,
                 int comp_strategy = Z_DEFAULT_STRATEGY);

  /**
   *  @brief  检查文件是否已打开
   *  @return  如果文件已打开则返回 true
  */
  bool
  is_open() const { return (file != NULL); }

  /**
   *  @brief  打开 gzip 压缩文件
   *  @param  name  文件名
   *  @param  mode  打开模式标志
   *  @return  成功返回 this，失败返回 NULL
  */
  gzfilebuf*
  open(const char* name,
       std::ios_base::openmode mode);

  /**
   *  @brief  连接到已经打开的 gzip 压缩文件
   *  @param  fd  文件描述符
   *  @param  mode  打开模式标志
   *  @return  成功返回 this，失败返回 NULL
  */
  gzfilebuf*
  attach(int fd,
         std::ios_base::openmode mode);

  /**
   *  @brief  关闭 gzip 压缩文件
   *  @return  成功返回 this，失败返回 NULL
  */
  gzfilebuf*
  close();
  /**
   *  @brief  Convert ios open mode int to mode string used by zlib.
   *  @return  True if valid mode flag combination.
  */
  bool
  open_mode(std::ios_base::openmode mode,
            char* c_mode) const;

  /**
   *  @brief  Number of characters available in stream buffer.
   *  @return  Number of characters.
   *
   *  This indicates number of characters in get area of stream buffer.
   *  These characters can be read without accessing the gzipped file.
  */
  virtual std::streamsize
  showmanyc();

  /**
   *  @brief  Fill get area from gzipped file.
   *  @return  First character in get area on success, EOF on error.
   *
   *  This actually reads characters from gzipped file to stream
   *  buffer. Always buffered.
  */
  virtual int_type
  underflow();

  /**
   *  @brief  Write put area to gzipped file.
   *  @param  c  Extra character to add to buffer contents.
   *  @return  Non-EOF on success, EOF on error.
   *
   *  This actually writes characters in stream buffer to
   *  gzipped file. With unbuffered output this is done one
   *  character at a time.
  */
  virtual int_type
  overflow(int_type c = traits_type::eof());

  /**
   *  @brief  Installs external stream buffer.
   *  @param  p  Pointer to char buffer.
   *  @param  n  Size of external buffer.
   *  @return  @c this on success, NULL on failure.
   *
   *  Call setbuf(0,0) to enable unbuffered output.
  */
  virtual std::streambuf*
  setbuf(char_type* p,
         std::streamsize n);

  /**
   *  @brief  Flush stream buffer to file.
   *  @return  0 on success, -1 on error.
   *
   *  This calls underflow(EOF) to do the job.
  */
  virtual int
  sync();

//
// Some future enhancements
//
//  virtual int_type uflow();
//  virtual int_type pbackfail(int_type c = traits_type::eof());
//  virtual pos_type
//  seekoff(off_type off,
//          std::ios_base::seekdir way,
//          std::ios_base::openmode mode = std::ios_base::in|std::ios_base::out);
//  virtual pos_type
//  seekpos(pos_type sp,
//          std::ios_base::openmode mode = std::ios_base::in|std::ios_base::out);
// 定义了一个名为seekpos的函数，用于设置文件指针的位置和打开模式

private:
  /**
   *  @brief  Allocate internal buffer.
   *
   *  This function is safe to call multiple times. It will ensure
   *  that a proper internal buffer exists if it is required. If the
   *  buffer already exists or is external, the buffer pointers will be
   *  reset to their original state.
  */
  void
  enable_buffer();
  // 分配内部缓冲区的函数，可以安全地多次调用。如果需要，它将确保存在适当的内部缓冲区。如果缓冲区已经存在或是外部的，缓冲区指针将被重置为它们的原始状态。

  /**
   *  @brief  Destroy internal buffer.
   *
   *  This function is safe to call multiple times. It will ensure
   *  that the internal buffer is deallocated if it exists. In any
   *  case, it will also reset the buffer pointers.
  */
  void
  disable_buffer();
  // 销毁内部缓冲区的函数，可以安全地多次调用。如果存在内部缓冲区，它将确保内部缓冲区被释放。在任何情况下，它还将重置缓冲区指针。

  /**
   *  Underlying file pointer.
  */
  gzFile file;
  // 底层文件指针

  /**
   *  Mode in which file was opened.
  */
  std::ios_base::openmode io_mode;
  // 文件打开的模式

  /**
   *  @brief  True if this object owns file descriptor.
   *
   *  This makes the class responsible for closing the file
   *  upon destruction.
  */
  bool own_fd;
  // 如果这个对象拥有文件描述符，则为true。这使得类在销毁时负责关闭文件。

  /**
   *  @brief  Stream buffer.
   *
   *  For simplicity this remains allocated on the free store for the
   *  entire life span of the gzfilebuf object, unless replaced by setbuf.
  */
  char_type* buffer;
  // 流缓冲区

  /**
   *  @brief  Stream buffer size.
   *
   *  Defaults to system default buffer size (typically 8192 bytes).
   *  Modified by setbuf.
  */
  std::streamsize buffer_size;
  // 流缓冲区的大小，默认为系统默认的缓冲区大小（通常为8192字节）。可以通过setbuf进行修改。

  /**
   *  @brief  True if this object owns stream buffer.
   *
   *  This makes the class responsible for deleting the buffer
   *  upon destruction.
  */
  bool own_buffer;
  // 如果这个对象拥有流缓冲区，则为true。这使得类在销毁时负责删除缓冲区。
};

/*****************************************************************************/

/**
 *  @brief  Gzipped file input stream class.
 *
 *  This class implements ifstream for gzipped files. Seeking and putback
 *  is not supported yet.
*/
class gzifstream : public std::istream
// Gzipped文件输入流类，实现了对gzipped文件的ifstream。目前不支持寻找和putback。
// 默认构造函数
gzifstream();

/**
 *  @brief  构造一个打开的压缩文件流
 *  @param  name  文件名
 *  @param  mode  打开模式标志（强制包含 ios::in）
*/
explicit
gzifstream(const char* name,
           std::ios_base::openmode mode = std::ios_base::in);

/**
 *  @brief  构造一个已经打开的压缩文件流
 *  @param  fd    文件描述符
 *  @param  mode  打开模式标志（强制包含 ios::in）
*/
explicit
gzifstream(int fd,
           std::ios_base::openmode mode = std::ios_base::in);

/**
 *  获取底层流缓冲区
*/
gzfilebuf*
rdbuf() const
{ return const_cast<gzfilebuf*>(&sb); }

/**
 *  @brief  检查文件是否打开
 *  @return  如果文件打开则返回 true
*/
bool
is_open() { return sb.is_open(); }

/**
 *  @brief  打开压缩文件
 *  @param  name  文件名
 *  @param  mode  打开模式标志（强制包含 ios::in）
 *
 *  如果文件成功打开，流将处于 good() 状态；否则处于 fail() 状态。
 * 这与 ifstream 的行为不同，后者永远不会将状态设置为 good()，因此
 * 除非手动清除状态，否则不允许将流重用于第二个文件。这是一种方便的选择。
*/
void
open(const char* name,
     std::ios_base::openmode mode = std::ios_base::in);

/**
 *  @brief  附加到已经打开的压缩文件
 *  @param  fd  文件描述符
 *  @param  mode  打开模式标志（强制包含 ios::in）
 *
 *  如果附加成功，流将处于 good() 状态；否则处于 fail() 状态。
*/
void
attach(int fd,
       std::ios_base::openmode mode = std::ios_base::in);

/**
 *  @brief  关闭压缩文件
 *
 *  如果关闭失败，流将处于 fail() 状态。
*/
void
close();

/**
 *  底层流缓冲区
*/
gzfilebuf sb;
/*****************************************************************************/

/**
 *  @brief  Gzipped file output stream class.
 *
 *  This class implements ofstream for gzipped files. Seeking and putback
 *  is not supported yet.
*/
class gzofstream : public std::ostream
{
// 默认构造函数
gzofstream();

/**
 * @brief  构造一个打开的压缩文件流
 * @param  name  文件名
 * @param  mode  打开模式标志（强制包含 ios::out）
 */
explicit
gzofstream(const char* name,
           std::ios_base::openmode mode = std::ios_base::out);

/**
 * @brief  构造一个已经打开的压缩文件流
 * @param  fd    文件描述符
 * @param  mode  打开模式标志（强制包含 ios::out）
 */
explicit
gzofstream(int fd,
           std::ios_base::openmode mode = std::ios_base::out);

/**
 *  获取底层流缓冲区
 */
gzfilebuf*
rdbuf() const
{ return const_cast<gzfilebuf*>(&sb); }

/**
 * @brief  检查文件是否打开
 * @return  如果文件打开则返回 true
 */
bool
is_open() { return sb.is_open(); }

/**
 * @brief  打开压缩文件
 * @param  name  文件名
 * @param  mode  打开模式标志（强制包含 ios::out）
 *
 * 如果文件成功打开，流将处于 good() 状态；否则处于 fail() 状态。
 * 这与 ofstream 的行为不同，ofstream 从不将状态设置为 good()，因此
 * 除非手动清除状态，否则不允许您重用流以打开第二个文件。选择是一种方便性问题。
 */
void
open(const char* name,
     std::ios_base::openmode mode = std::ios_base::out);

/**
 * @brief  附加到已经打开的压缩文件
 * @param  fd  文件描述符
 * @param  mode  打开模式标志（强制包含 ios::out）
 *
 * 如果附加成功，流将处于 good() 状态；否则处于 fail() 状态。
 */
void
attach(int fd,
       std::ios_base::openmode mode = std::ios_base::out);

/**
 * @brief  关闭压缩文件
 *
 * 如果关闭失败，流将处于 fail() 状态。
 */
void
close();

/**
 *  底层流缓冲区
 */
gzfilebuf sb;
/*****************************************************************************/
/**
 *  @brief  Gzipped file output stream manipulator class.
 *
 *  This class defines a two-argument manipulator for gzofstream. It is used
 *  as base for the setcompression(int,int) manipulator.
*/
template<typename T1, typename T2>
  class gzomanip2
  {
  public:
    // Allows insertor to peek at internals
    template <typename Ta, typename Tb>
      friend gzofstream&
      operator<<(gzofstream&,
                 const gzomanip2<Ta,Tb>&);

    // Constructor
    gzomanip2(gzofstream& (*f)(gzofstream&, T1, T2),
              T1 v1,
              T2 v2);
  private:
    // Underlying manipulator function
    gzofstream&
    (*func)(gzofstream&, T1, T2);

    // Arguments for manipulator function
    T1 val1;
    T2 val2;
  };
/*****************************************************************************/
// Manipulator function thunks through to stream buffer
inline gzofstream&
setcompression(gzofstream &gzs, int l, int s = Z_DEFAULT_STRATEGY)
{
  (gzs.rdbuf())->setcompression(l, s);
  return gzs;
}
// Manipulator constructor stores arguments
template<typename T1, typename T2>
  inline
  gzomanip2<T1,T2>::gzomanip2(gzofstream &(*f)(gzofstream &, T1, T2),
                              T1 v1,
                              T2 v2)
  : func(f), val1(v1), val2(v2)
  { }
// Insertor applies underlying manipulator function to stream
template<typename T1, typename T2>
  inline gzofstream&
  operator<<(gzofstream& s, const gzomanip2<T1,T2>& m)
  { return (*m.func)(s, m.val1, m.val2); }
// Insert this onto stream to simplify setting of compression level
inline gzomanip2<int,int>
setcompression(int l, int s = Z_DEFAULT_STRATEGY)
{ return gzomanip2<int,int>(&setcompression, l, s); }
#endif // ZFSTREAM_H
```