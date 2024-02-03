# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\whispercpp\java\whisper\WhisperLib.java`

```cpp
package com.whispercpp.java.whisper;

import android.content.res.AssetManager; // 导入 Android 资产管理器类
import android.os.Build; // 导入 Android 构建类
import android.util.Log; // 导入 Android 日志类

import androidx.annotation.RequiresApi; // 导入 Android 版本要求注解

import java.io.InputStream; // 导入输入流类

@RequiresApi(api = Build.VERSION_CODES.O) // 标记需要 API 版本为 Android O

public class WhisperLib { // 定义 WhisperLib 类
  private static final String LOG_TAG = "LibWhisper"; // 定义日志标签

  static { // 静态代码块开始

    Log.d(LOG_TAG, "Primary ABI: " + Build.SUPPORTED_ABIS[0]); // 打印主要 ABI 信息
    boolean loadVfpv4 = false; // 初始化是否加载 vfpv4 库的标志
    boolean loadV8fp16 = false; // 初始化是否加载 v8fp16 库的标志
    if (WhisperUtils.isArmEabiV7a()) { // 如果是 ARM EABI V7a 架构
      String cpuInfo = WhisperUtils.cpuInfo(); // 获取 CPU 信息
      if (cpuInfo != null) { // 如果 CPU 信息不为空
        Log.d(LOG_TAG, "CPU info: " + cpuInfo); // 打印 CPU 信息
        if (cpuInfo.contains("vfpv4")) { // 如果 CPU 支持 vfpv4
          Log.d(LOG_TAG, "CPU supports vfpv4"); // 打印 CPU 支持 vfpv4
          loadVfpv4 = true; // 设置加载 vfpv4 库的标志为 true
        }
      }
    } else if (WhisperUtils.isArmEabiV8a()) { // 如果是 ARM EABI V8a 架构
      String cpuInfo = WhisperUtils.cpuInfo(); // 获取 CPU 信息
      if (cpuInfo != null) { // 如果 CPU 信息不为空
        Log.d(LOG_TAG, "CPU info: " + cpuInfo); // 打印 CPU 信息
        if (cpuInfo.contains("fphp")) { // 如果 CPU 支持 fp16 算术
          Log.d(LOG_TAG, "CPU supports fp16 arithmetic"); // 打印 CPU 支持 fp16 算术
          loadV8fp16 = true; // 设置加载 v8fp16 库的标志为 true
        }
      }
    }

    if (loadVfpv4) { // 如果需要加载 vfpv4 库
      Log.d(LOG_TAG, "Loading libwhisper_vfpv4.so"); // 打印加载 vfpv4 库信息
      System.loadLibrary("whisper_vfpv4"); // 加载 vfpv4 库
    } else if (loadV8fp16) { // 如果需要加载 v8fp16 库
      Log.d(LOG_TAG, "Loading libwhisper_v8fp16_va.so"); // 打印加载 v8fp16 库信息
      System.loadLibrary("whisper_v8fp16_va"); // 加载 v8fp16 库
    } else { // 如果需要加载默认库
      Log.d(LOG_TAG, "Loading libwhisper.so"); // 打印加载默认库信息
      System.loadLibrary("whisper"); // 加载默认库
  // 从输入流初始化上下文
  public static native long initContextFromInputStream(InputStream inputStream);

  // 从资产管理器和资产路径初始化上下文
  public static native long initContextFromAsset(AssetManager assetManager, String assetPath);

  // 从模型路径初始化上下文
  public static native long initContext(String modelPath);

  // 释放上下文
  public static native void freeContext(long contextPtr);

  // 完全转录
  public static native void fullTranscribe(long contextPtr, int numThreads, float[] audioData);

  // 获取文本段数
  public static native int getTextSegmentCount(long contextPtr);

  // 获取指定索引的文本段
  public static native String getTextSegment(long contextPtr, int index);

  // 获取指定索引的文本段的起始时间
  public static native long getTextSegmentT0(long contextPtr, int index);

  // 获取指定索引的文本段的结束时间
  public static native long getTextSegmentT1(long contextPtr, int index);

  // 获取系统信息
  public static native String getSystemInfo();

  // 进行内存拷贝基准测试
  public static native String benchMemcpy(int nthread);

  // 进行 GGML 矩阵乘法基准测试
  public static native String benchGgmlMulMat(int nthread);
# 闭合大括号，表示代码块的结束
```