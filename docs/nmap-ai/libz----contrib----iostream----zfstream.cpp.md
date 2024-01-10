# `nmap\libz\contrib\iostream\zfstream.cpp`

```
// 默认构造函数，初始化成员变量
gzfilebuf::gzfilebuf() :
  file(NULL),
  mode(0),
  own_file_descriptor(0)
{ }

// 析构函数
gzfilebuf::~gzfilebuf() {
  // 同步缓冲区
  sync();
  // 如果拥有文件描述符，则关闭文件
  if ( own_file_descriptor )
    close();
}

// 打开文件
gzfilebuf *gzfilebuf::open( const char *name,
                            int io_mode ) {
  // 如果文件已经打开，则返回空指针
  if ( is_open() )
    return NULL;
  char char_mode[10];
  char *p = char_mode;
  // 根据输入输出模式设置文件打开模式
  if ( io_mode & ios::in ) {
    mode = ios::in;
    *p++ = 'r';
  } else if ( io_mode & ios::app ) {
    mode = ios::app;
    *p++ = 'a';
  } else {
    mode = ios::out;
    *p++ = 'w';
  }
  // 如果是二进制模式，则设置文件打开模式
  if ( io_mode & ios::binary ) {
    mode |= ios::binary;
    *p++ = 'b';
  }
  // 如果是输出或追加模式，则设置压缩级别为9
  if ( io_mode & (ios::out|ios::app )) {
    *p++ = '9';
  }
  // 设置字符串结束符
  *p = '\0';
  // 打开文件
  if ( (file = gzopen(name, char_mode)) == NULL )
    return NULL;
  // 设置拥有文件描述符标志
  own_file_descriptor = 1;
  return this;
}

// 关联文件描述符
gzfilebuf *gzfilebuf::attach( int file_descriptor,
                              int io_mode ) {
  // 如果文件已经打开，则返回空指针
  if ( is_open() )
    return NULL;
  char char_mode[10];
  char *p = char_mode;
  // 根据输入输出模式设置文件打开模式
  if ( io_mode & ios::in ) {
    mode = ios::in;
    *p++ = 'r';
  } else if ( io_mode & ios::app ) {
    mode = ios::app;
    *p++ = 'a';
  } else {
    mode = ios::out;
    *p++ = 'w';
  }
  // 如果是二进制模式，则设置文件打开模式
  if ( io_mode & ios::binary ) {
    mode |= ios::binary;
    *p++ = 'b';
  }
  // 如果是输出或追加模式，则设置压缩级别为9
  if ( io_mode & (ios::out|ios::app )) {
    *p++ = '9';
  }
  // 设置字符串结束符
  *p = '\0';
  // 关联文件描述符
  if ( (file = gzdopen(file_descriptor, char_mode)) == NULL )
    return NULL;
  // 清除拥有文件描述符标志
  own_file_descriptor = 0;
  return this;
}

// 关闭文件
gzfilebuf *gzfilebuf::close() {
  // 如果文件已经打开
  if ( is_open() ) {
    // 同步缓冲区
    sync();
    // 关闭文件
    gzclose( file );
    file = NULL;
  }
  return this;
}

// 设置压缩级别
int gzfilebuf::setcompressionlevel( int comp_level ) {
  return gzsetparams(file, comp_level, -2);
}

// 设置压缩策略
int gzfilebuf::setcompressionstrategy( int comp_strategy ) {
  return gzsetparams(file, -2, comp_strategy);
}
streampos gzfilebuf::seekoff( streamoff off, ios::seek_dir dir, int which ) {
  // 返回文件流位置的特殊值 EOF
  return streampos(EOF);
}

int gzfilebuf::underflow() {
  // 如果文件没有被打开用于读取，返回错误
  if ( !is_open() || !(mode & ios::in) )
    return EOF;

  // 如果缓冲区不存在，分配一个
  if ( !base() ) {
    if ( (allocate()) == EOF )
      return EOF;
    setp(0,0);
  } else {
    // 如果有可用的输入字符，返回下一个字符
    if ( in_avail() )
      return (unsigned char) *gptr();
    // 如果有等待输出的字符，刷新缓冲区
    if ( out_waiting() ) {
      if ( flushbuf() == EOF )
        return EOF;
    }
  }

  // 尝试填充缓冲区
  int result = fillbuf();
  if ( result == EOF ) {
    // 禁用获取区域
    setg(0,0,0);
    return EOF;
  }

  return (unsigned char) *gptr();
}

int gzfilebuf::overflow( int c ) {
  // 如果文件没有被打开用于写入，返回错误
  if ( !is_open() || !(mode & ios::out) )
    return EOF;

  // 如果缓冲区不存在，分配一个
  if ( !base() ) {
    if ( allocate() == EOF )
      return EOF;
    setg(0,0,0);
  } else {
    // 如果有可用的输入字符，返回错误
    if (in_avail()) {
        return EOF;
    }
    // 如果有等待输出的字符，刷新缓冲区
    if (out_waiting()) {
      if (flushbuf() == EOF)
        return EOF;
    }
  }

  int bl = blen();
  setp( base(), base() + bl);

  if ( c != EOF ) {
    // 将字符写入缓冲区
    *pptr() = c;
    pbump(1);
  }

  return 0;
}

int gzfilebuf::sync() {
  // 如果文件没有被打开，返回错误
  if ( !is_open() )
    return EOF;

  // 如果有等待输出的字符，刷新缓冲区
  if ( out_waiting() )
    return flushbuf();

  return 0;
}

int gzfilebuf::flushbuf() {
  int n;
  char *q;

  q = pbase();
  n = pptr() - q;

  // 将缓冲区的内容写入文件
  if ( gzwrite( file, q, n) < n )
    return EOF;

  setp(0,0);

  return 0;
}

int gzfilebuf::fillbuf() {
  int required;
  char *p;

  p = base();

  required = blen();

  int t = gzread( file, p, required );

  // 如果读取失败，返回错误
  if ( t <= 0) return EOF;

  setg( base(), base(), base()+t);

  return t;
}

gzfilestream_common::gzfilestream_common() :
  ios( gzfilestream_common::rdbuf() )
{ }

gzfilestream_common::~gzfilestream_common()
{ }

void gzfilestream_common::attach( int fd, int io_mode ) {
  // 如果无法将缓冲区附加到文件描述符，设置错误标志
  if ( !buffer.attach( fd, io_mode) )
    clear( ios::failbit | ios::badbit );
  else
    clear();
}
// 打开文件流，根据给定的文件名和 I/O 模式
void gzfilestream_common::open( const char *name, int io_mode ) {

  // 如果无法打开文件流，则设置错误位并清除流状态
  if ( !buffer.open( name, io_mode ) )
    clear( ios::failbit | ios::badbit );
  else
    clear();

}

// 关闭文件流
void gzfilestream_common::close() {

  // 如果无法关闭文件流，则设置错误位并清除流状态
  if ( !buffer.close() )
    clear( ios::failbit | ios::badbit );

}

// 返回文件流缓冲区指针
gzfilebuf *gzfilestream_common::rdbuf()
{
  return &buffer;
}

// 默认构造函数，初始化文件流状态并设置错误位
gzifstream::gzifstream() :
  ios( gzfilestream_common::rdbuf() )
{
  clear( ios::badbit );
}

// 根据文件名和 I/O 模式构造函数，初始化文件流状态并打开文件流
gzifstream::gzifstream( const char *name, int io_mode ) :
  ios( gzfilestream_common::rdbuf() )
{
  gzfilestream_common::open( name, io_mode );
}

// 根据文件描述符和 I/O 模式构造函数，初始化文件流状态并附加到文件描述符
gzifstream::gzifstream( int fd, int io_mode ) :
  ios( gzfilestream_common::rdbuf() )
{
  gzfilestream_common::attach( fd, io_mode );
}

// 析构函数
gzifstream::~gzifstream() { }

// 默认构造函数，初始化文件流状态并设置错误位
gzofstream::gzofstream() :
  ios( gzfilestream_common::rdbuf() )
{
  clear( ios::badbit );
}

// 根据文件名和 I/O 模式构造函数，初始化文件流状态并打开文件流
gzofstream::gzofstream( const char *name, int io_mode ) :
  ios( gzfilestream_common::rdbuf() )
{
  gzfilestream_common::open( name, io_mode );
}

// 根据文件描述符和 I/O 模式构造函数，初始化文件流状态并附加到文件描述符
gzofstream::gzofstream( int fd, int io_mode ) :
  ios( gzfilestream_common::rdbuf() )
{
  gzfilestream_common::attach( fd, io_mode );
}

// 析构函数
gzofstream::~gzofstream() { }
```