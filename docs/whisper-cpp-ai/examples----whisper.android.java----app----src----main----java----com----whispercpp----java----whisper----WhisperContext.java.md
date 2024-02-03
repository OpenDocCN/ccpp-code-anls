# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\whispercpp\java\whisper\WhisperContext.java`

```cpp
package com.whispercpp.java.whisper;

import android.content.res.AssetManager;
import android.os.Build;
import android.util.Log;

import androidx.annotation.RequiresApi;

import com.litongjava.whisper.android.java.bean.WhisperSegment;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class WhisperContext {

  private static final String LOG_TAG = "LibWhisper";
  private long ptr;
  private final ExecutorService executorService;

  private WhisperContext(long ptr) {
    this.ptr = ptr;
    this.executorService = Executors.newSingleThreadExecutor();
  }

  // 将音频数据转录为文本
  public String transcribeData(float[] data) throws ExecutionException, InterruptedException {
    return executorService.submit(new Callable<String>() {
      @RequiresApi(api = Build.VERSION_CODES.O)
      @Override
      public String call() throws Exception {
        // 检查指针是否有效
        if (ptr == 0L) {
          throw new IllegalStateException();
        }
        // 获取首选线程数并输出日志
        int numThreads = WhisperCpuConfig.getPreferredThreadCount();
        Log.d(LOG_TAG, "Selecting " + numThreads + " threads");

        StringBuilder result = new StringBuilder();
        synchronized (this) {
          // 调用底层库进行全文转录
          WhisperLib.fullTranscribe(ptr, numThreads, data);
          // 获取文本段数
          int textCount = WhisperLib.getTextSegmentCount(ptr);
          // 遍历文本段并添加到结果中
          for (int i = 0; i < textCount; i++) {
            String sentence = WhisperLib.getTextSegment(ptr, i);
            result.append(sentence);
          }
        }
        return result.toString();
      }
    }).get();
  }

  // 将音频数据转录为带时间信息的文本段列表
  public List<WhisperSegment> transcribeDataWithTime(float[] data) throws ExecutionException, InterruptedException {
    # 使用 executorService 提交一个 Callable 对象，返回一个 Future 对象
    return executorService.submit(new Callable<List<WhisperSegment>>() {
      # 要求 API 版本为 Build.VERSION_CODES.O
      @RequiresApi(api = Build.VERSION_CODES.O)
      # 重写 call 方法
      @Override
      public List<WhisperSegment> call() throws Exception {
        # 如果 ptr 为 0L，则抛出 IllegalStateException 异常
        if (ptr == 0L) {
          throw new IllegalStateException();
        }
        # 获取 WhisperCpuConfig 中的首选线程数
        int numThreads = WhisperCpuConfig.getPreferredThreadCount();
        # 打印日志，选择 numThreads 个线程
        Log.d(LOG_TAG, "Selecting " + numThreads + " threads");

        # 创建一个空的 WhisperSegment 列表
        List<WhisperSegment> segments = new ArrayList<>();
        # 同步块，确保线程安全
        synchronized (this) {
  // 调用WhisperLib的fullTranscribe方法，进行全文转录
  WhisperLib.fullTranscribe(ptr, numThreads, data);
  // 获取文本段的数量
  int textCount = WhisperLib.getTextSegmentCount(ptr);
  // 遍历每个文本段
  for (int i = 0; i < textCount; i++) {
    // 获取文本段的起始时间
    long start = WhisperLib.getTextSegmentT0(ptr, i);
    // 获取文本段的内容
    String sentence = WhisperLib.getTextSegment(ptr, i);
    // 获取文本段的结束时间
    long end = WhisperLib.getTextSegmentT1(ptr, i);
    // 将文本段信息添加到segments列表中
    segments.add(new WhisperSegment(start, end, sentence));
  }
  // 返回segments列表
  return segments;
}

// 使用executorService提交任务，进行内存基准测试
@RequiresApi(api = Build.VERSION_CODES.O)
public String benchMemory(int nthreads) throws ExecutionException, InterruptedException {
  return executorService.submit(() -> WhisperLib.benchMemcpy(nthreads)).get();
}

// 使用executorService提交任务，进行矩阵乘法基准测试
@RequiresApi(api = Build.VERSION_CODES.O)
public String benchGgmlMulMat(int nthreads) throws ExecutionException, InterruptedException {
  return executorService.submit(() -> WhisperLib.benchGgmlMulMat(nthreads)).get();
}

// 释放资源
@RequiresApi(api = Build.VERSION_CODES.O)
public void release() throws ExecutionException, InterruptedException {
  executorService.submit(() -> {
    // 如果ptr不为0，则释放资源
    if (ptr != 0L) {
      WhisperLib.freeContext(ptr);
      ptr = 0;
    }
  }).get();
}

// 从文件路径创建WhisperContext对象
@RequiresApi(api = Build.VERSION_CODES.O)
public static WhisperContext createContextFromFile(String filePath) {
  // 初始化WhisperContext对象
  long ptr = WhisperLib.initContext(filePath);
  // 如果ptr为0，则抛出异常
  if (ptr == 0L) {
    throw new RuntimeException("Couldn't create context with path " + filePath);
  }
  return new WhisperContext(ptr);
}

// 从输入流创建WhisperContext对象
@RequiresApi(api = Build.VERSION_CODES.O)
public static WhisperContext createContextFromInputStream(InputStream stream) {
  // 初始化WhisperContext对象
  long ptr = WhisperLib.initContextFromInputStream(stream);
  // 如果ptr为0，则抛出异常
  if (ptr == 0L) {
    throw new RuntimeException("Couldn't create context from input stream");
  }
}
  // 根据指针创建一个新的 WhisperContext 对象并返回
  return new WhisperContext(ptr);
}

@RequiresApi(api = Build.VERSION_CODES.O)
public static WhisperContext createContextFromAsset(AssetManager assetManager, String assetPath) {
  // 从 assetManager 和 assetPath 创建一个新的 WhisperContext 对象的指针
  long ptr = WhisperLib.initContextFromAsset(assetManager, assetPath);
  // 如果指针为 0，则抛出异常
  if (ptr == 0L) {
    throw new RuntimeException("Couldn't create context from asset " + assetPath);
  }
  // 根据指针创建一个新的 WhisperContext 对象并返回
  return new WhisperContext(ptr);
}

@RequiresApi(api = Build.VERSION_CODES.O)
public static String getSystemInfo() {
  // 获取系统信息并返回
  return WhisperLib.getSystemInfo();
}
# 闭合之前的代码块
```