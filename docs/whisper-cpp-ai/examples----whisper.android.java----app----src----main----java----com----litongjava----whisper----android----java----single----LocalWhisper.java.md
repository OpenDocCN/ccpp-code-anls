# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\single\LocalWhisper.java`

```cpp
package com.litongjava.whisper.android.java.single;

import android.app.Application;
import android.os.Build;
import android.os.Handler;

import androidx.annotation.RequiresApi;

import com.blankj.utilcode.util.ToastUtils;
import com.blankj.utilcode.util.Utils;
import com.litongjava.jfinal.aop.Aop;
import com.litongjava.whisper.android.java.bean.WhisperSegment;
import com.litongjava.whisper.android.java.utils.AssetUtils;
import com.whispercpp.java.whisper.WhisperContext;

import java.io.File;
import java.util.List;
import java.util.concurrent.ExecutionException;

// 要求 API 版本为 Android O 及以上
@RequiresApi(api = Build.VERSION_CODES.O)
public enum LocalWhisper {
  INSTANCE;

  // 模型文件路径
  public static final String modelFilePath = "models/ggml-tiny.bin";
  // Whisper 上下文对象
  private WhisperContext whisperContext;

  // 构造函数，初始化 Whisper 上下文对象
  @RequiresApi(api = Build.VERSION_CODES.O)
  LocalWhisper() {
    // 获取应用程序上下文
    Application context = Utils.getApp();
    // 获取应用程序文件目录
    File filesDir = context.getFilesDir();
    // 复制模型文件到应用程序文件目录下
    File modelFile = AssetUtils.copyFileIfNotExists(context, filesDir, modelFilePath);
    // 获取模型文件的绝对路径
    String realModelFilePath = modelFile.getAbsolutePath();
    // 创建 Whisper 上下文对象
    whisperContext = WhisperContext.createContextFromFile(realModelFilePath);
  }

  // 同步转录数据
  public synchronized String transcribeData(float[] data) throws ExecutionException, InterruptedException {
    // 如果 Whisper 上下文对象为空，显示模型加载提示并返回空
    if(whisperContext==null){
        toastModelLoading();
        return null;
    }else{
      // 调用 Whisper 上下文对象的转录数据方法
      return whisperContext.transcribeData(data);
    }
  }

  // 显示模型加载提示
  private static void toastModelLoading() {
    // 使用 Aop 获取 Handler 对象，并在主线程中显示 Toast 提示
    Aop.get(Handler.class).post(()->{
      ToastUtils.showShort("please wait for model loading");
    });
  }

  // 带时间信息的转录数据
  public List<WhisperSegment> transcribeDataWithTime(float[] audioData) throws ExecutionException, InterruptedException {
    // 如果 Whisper 上下文对象为空，显示模型加载提示并返回空
    if(whisperContext==null){
        toastModelLoading();
      return null;
    }else{
      // 调用 Whisper 上下文对象的带时间信息的转录数据方法
      return whisperContext.transcribeDataWithTime(audioData);
    }
  }

  // 初始化方法
  public void init() {
    // 无需执行任何操作，只是初始化
  }
}
```