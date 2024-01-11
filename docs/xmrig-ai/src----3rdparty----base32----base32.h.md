# `xmrig\src\3rdparty\base32\base32.h`

```
// 定义了一个名为 base32_encode 的函数，用于将数据进行 base32 编码
// 参数 data: 要编码的数据
// 参数 length: 数据的长度
// 参数 result: 编码后的结果
// 参数 bufSize: 结果缓冲区的大小
int base32_encode(const uint8_t *data, int length, uint8_t *result, int bufSize) {
  // 检查数据长度是否合法
  if (length < 0 || length > (1 << 28)) {
    return -1;
  }
  // 初始化计数器
  int count = 0;
  // 如果数据长度大于0
  if (length > 0) {
    // 初始化缓冲区、下一个要处理的数据、剩余的比特数
    int buffer = data[0];
    int next = 1;
    int bitsLeft = 8;
    // 循环直到结果缓冲区满或者数据处理完毕
    while (count < bufSize && (bitsLeft > 0 || next < length)) {
      // 如果剩余比特数小于5
      if (bitsLeft < 5) {
        // 如果还有未处理的数据
        if (next < length) {
          // 将下一个字节加入缓冲区
          buffer <<= 8;
          buffer |= data[next++] & 0xFF;
          bitsLeft += 8;
        } else {
          // 如果没有未处理的数据，填充0
          int pad = 5 - bitsLeft;
          buffer <<= pad;
          bitsLeft += pad;
        }
      }
      // 计算索引并将对应的 base32 字符写入结果缓冲区
      int index = 0x1F & (buffer >> (bitsLeft - 5));
      bitsLeft -= 5;
      result[count++] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567"[index];
    }
  }
  // 如果结果缓冲区未满，添加字符串结束符
  if (count < bufSize) {
    result[count] = '\000';
  }
  // 返回编码后的结果长度
  return count;
}
```