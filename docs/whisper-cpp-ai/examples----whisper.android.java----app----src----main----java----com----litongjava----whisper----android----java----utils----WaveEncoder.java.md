# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\utils\WaveEncoder.java`

```cpp
package com.litongjava.whisper.android.java.utils;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.ShortBuffer;

public class WaveEncoder {

  // 解码 WAV 文件，返回浮点数数组
  public static float[] decodeWaveFile(File file) throws IOException {
    // 创建一个字节数组输出流
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    // 使用文件输入流读取文件内容
    try (FileInputStream fis = new FileInputStream(file)) {
      byte[] buffer = new byte[1024];
      int bytesRead;
      // 循环读取文件内容到字节数组输出流
      while ((bytesRead = fis.read(buffer)) != -1) {
        baos.write(buffer, 0, bytesRead);
      }
    }
    // 将字节数组输出流的内容包装成 ByteBuffer
    ByteBuffer byteBuffer = ByteBuffer.wrap(baos.toByteArray());
    byteBuffer.order(ByteOrder.LITTLE_ENDIAN);

    // 从 ByteBuffer 中获取声道数
    int channel = byteBuffer.getShort(22);
    byteBuffer.position(44);

    // 将 ByteBuffer 转换为 ShortBuffer
    ShortBuffer shortBuffer = byteBuffer.asShortBuffer();
    short[] shortArray = new short[shortBuffer.limit()];
    shortBuffer.get(shortArray);

    // 创建浮点数数组用于存储解码后的数据
    float[] output = new float[shortArray.length / channel];

    // 解码数据并存储到浮点数数组中
    for (int index = 0; index < output.length; index++) {
      if (channel == 1) {
        output[index] = Math.max(-1f, Math.min(1f, shortArray[index] / 32767.0f));
      } else {
        output[index] = Math.max(-1f, Math.min(1f, (shortArray[2 * index] + shortArray[2 * index + 1]) / 32767.0f / 2.0f));
      }
    }
    return output;
  }

  // 编码 WAV 文件，将 short 数组写入文件
  public static void encodeWaveFile(File file, short[] data) throws IOException {
    // 使用文件输出流写入 WAV 文件头部信息
    try (FileOutputStream fos = new FileOutputStream(file)) {
      fos.write(headerBytes(data.length * 2));

      // 创建 ByteBuffer 用于存储 short 数组数据
      ByteBuffer buffer = ByteBuffer.allocate(data.length * 2);
      buffer.order(ByteOrder.LITTLE_ENDIAN);
      buffer.asShortBuffer().put(data);

      // 将 ByteBuffer 转换为字节数组并写入文件
      byte[] bytes = new byte[buffer.limit()];
      buffer.get(bytes);

      fos.write(bytes);
    }
  }

  // 生成 WAV 文件头部信息的字节数组
  private static byte[] headerBytes(int totalLength) {
    # 如果总长度小于44个字节，则抛出非法参数异常
    if (totalLength < 44)
      throw new IllegalArgumentException("Total length must be at least 44 bytes");

    # 创建一个容量为44字节的字节缓冲区
    ByteBuffer buffer = ByteBuffer.allocate(44);
    # 设置字节顺序为小端序
    buffer.order(ByteOrder.LITTLE_ENDIAN);

    # 向缓冲区中放入'R','I','F','F'四个字节
    buffer.put((byte) 'R');
    buffer.put((byte) 'I');
    buffer.put((byte) 'F');
    buffer.put((byte) 'F');

    # 放入总长度减去8的整数值
    buffer.putInt(totalLength - 8);

    # 放入'W','A','V','E'四个字节
    buffer.put((byte) 'W');
    buffer.put((byte) 'A');
    buffer.put((byte) 'V');
    buffer.put((byte) 'E');

    # 放入'f','m','t',' '四个字节
    buffer.put((byte) 'f');
    buffer.put((byte) 'm');
    buffer.put((byte) 't');
    buffer.put((byte) ' ');

    # 放入16、1、1、16000、32000、2、16等整数或短整数值
    buffer.putInt(16);
    buffer.putShort((short) 1);
    buffer.putShort((short) 1);
    buffer.putInt(16000);
    buffer.putInt(32000);
    buffer.putShort((short) 2);
    buffer.putShort((short) 16);

    # 放入'd','a','t','a'四个字节
    buffer.put((byte) 'd');
    buffer.put((byte) 'a');
    buffer.put((byte) 't');
    buffer.put((byte) 'a');

    # 放入总长度减去44的整数值
    buffer.putInt(totalLength - 44);
    # 设置缓冲区位置为0
    buffer.position(0);

    # 创建一个与缓冲区大小相同的字节数组
    byte[] bytes = new byte[buffer.limit()];
    # 将缓冲区中的数据读取到字节数组中
    buffer.get(bytes);

    # 返回字节数组
    return bytes;
  }
# 闭合大括号，表示代码块的结束
```