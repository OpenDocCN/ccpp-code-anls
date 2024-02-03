# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\whispercpp\java\whisper\WhisperUtils.java`

```cpp
package com.whispercpp.java.whisper;

import android.os.Build;
import android.util.Log;

import androidx.annotation.RequiresApi;

import java.io.File;
import java.nio.file.Path;

public class WhisperUtils {
  private static final String LOG_TAG = "LibWhisper";


  // 检查设备是否为 armeabi-v7a 架构
  public static boolean isArmEabiV7a() {
    return Build.SUPPORTED_ABIS[0].equals("armeabi-v7a");
  }

  // 检查设备是否为 arm64-v8a 架构
  public static boolean isArmEabiV8a() {
    return Build.SUPPORTED_ABIS[0].equals("arm64-v8a");
  }

  // 获取设备的 CPU 信息
  @RequiresApi(api = Build.VERSION_CODES.O)
  public static String cpuInfo() {
    try {
      // 获取 /proc/cpuinfo 文件的路径
      Path path = new File("/proc/cpuinfo").toPath();
      // 读取 /proc/cpuinfo 文件的内容并转换为字符串返回
      return new String(java.nio.file.Files.readAllBytes(path));
    } catch (Exception e) {
      // 如果读取失败，记录错误日志并返回空
      Log.w(LOG_TAG, "Couldn't read /proc/cpuinfo", e);
      return null;
    }

  }
}
```