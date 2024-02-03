# `nmap\libz\contrib\iostream\test.cpp`

```cpp
#include "zfstream.h"

int main() {

  // 使用文件缓冲构造一个流对象。发送到这个流的任何内容都将被发送到标准输出。
  gzofstream os( 1, ios::out );

  // 这段文本被压缩并发送到标准输出。为了证明这一点，运行'test | zcat'。
  os << "Hello, Mommy" << endl;

  // 设置压缩级别为无压缩
  os << setcompressionlevel( Z_NO_COMPRESSION );
  os << "hello, hello, hi, ho!" << endl;

  // 设置压缩级别为默认压缩
  setcompressionlevel( os, Z_DEFAULT_COMPRESSION )
    << "I'm compressing again" << endl;

  // 关闭流
  os.close();

  return 0;

}
```