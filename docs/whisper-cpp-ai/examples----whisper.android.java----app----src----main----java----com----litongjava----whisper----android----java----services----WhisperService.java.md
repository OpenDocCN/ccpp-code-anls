# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\services\WhisperService.java`

```cpp
package com.litongjava.whisper.android.java.services;

import android.content.Context;
import android.os.Build;
import android.os.Handler;
import android.widget.TextView;
import android.widget.Toast;

import androidx.annotation.RequiresApi;

import com.blankj.utilcode.util.ToastUtils;
import com.blankj.utilcode.util.Utils;
import com.litongjava.android.utils.dialog.AlertDialogUtils;
import com.litongjava.jfinal.aop.Aop;
import com.litongjava.whisper.android.java.bean.WhisperSegment;
import com.litongjava.whisper.android.java.single.LocalWhisper;
import com.litongjava.whisper.android.java.utils.WaveEncoder;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;
import java.io.IOException;
import java.util.List;
import java.util.concurrent.ExecutionException;

public class WhisperService {
  private Logger log = LoggerFactory.getLogger(this.getClass());

  private final Object lock = new Object();

  @RequiresApi(api = Build.VERSION_CODES.O)
  // 从指定 TextView 中加载模型
  public void loadModel(TextView tv) {
    // 获取模型文件路径
    String modelFilePath = LocalWhisper.modelFilePath;
    String msg = "load model from :" + modelFilePath + "\n";
    // 在 TextView 中输出消息
    outputMsg(tv, msg);

    // 记录开始时间
    long start = System.currentTimeMillis();
    // 初始化本地 Whisper 实例
    LocalWhisper.INSTANCE.init();
    // 记录结束时间
    long end = System.currentTimeMillis();
    msg = "model load successful:" + (end - start) + "ms";
    // 在 TextView 中输出消息
    outputMsg(tv, msg);
    // 显示 Toast 消息
    ToastUtils.showLong(msg);

  }

  @RequiresApi(api = Build.VERSION_CODES.O)
  // 从指定文件中转录音频样本
  public void transcribeSample(TextView tv, File sampleFile) {
    String msg = "";
    msg = "transcribe file from :" + sampleFile.getAbsolutePath();
    // 在 TextView 中输出消息
    outputMsg(tv, msg);

    Long start = System.currentTimeMillis();
    float[] audioData = new float[0];  // 读取音频样本
    try {
      // 解码音频文件为浮点数数组
      audioData = WaveEncoder.decodeWaveFile(sampleFile);
    } catch (IOException e) {
      e.printStackTrace();
      return;
    }
    long end = System.currentTimeMillis();
    msg = "decode wave file:" + (end - start) + "ms";
    // 在 TextView 中输出消息
    outputMsg(tv, msg);
    // 获取当前系统时间的毫秒数
    start = System.currentTimeMillis();
    // 初始化一个用于存储转录结果的列表
    List<WhisperSegment> transcription = null;
    try {
      // 调用 LocalWhisper 实例的 transcribeDataWithTime 方法对音频数据进行转录
      transcription = LocalWhisper.INSTANCE.transcribeDataWithTime(audioData);
    } catch (ExecutionException e) {
      e.printStackTrace();
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    // 获取转录结束时的系统时间的毫秒数
    end = System.currentTimeMillis();
    // 如果转录结果不为空
    if(transcription!=null){
      // 在界面上显示转录结果
      ToastUtils.showLong(transcription.toString());
      // 计算转录所花费的时间，并生成相应的消息
      msg = "Transcript successful:" + (end - start) + "ms";
      // 在界面上输出消息
      outputMsg(tv, msg);

      // 在界面上输出转录结果
      outputMsg(tv, transcription.toString());

    }else{
      // 如果转录结果为空，则生成相应的消息
      msg = "Transcript failed:" + (end - start) + "ms";
      // 在界面上输出消息
      outputMsg(tv, msg);
    }

  }

  // 在 TextView 中输出消息的方法
  private void outputMsg(TextView tv, String msg) {
    // 使用 log 输出消息
    log.info(msg);
    // 如果 TextView 不为空
    if(tv!=null){
      // 使用 Aop 获取 Handler 实例，并在主线程中更新 TextView 的内容
      Aop.get(Handler.class).post(()->{ tv.append(msg + "\n");});
    }
  }

  // 释放资源的方法，仅用于 API 版本大于等于 O
  @RequiresApi(api = Build.VERSION_CODES.O)
  public void release() {
    // 无需执行任何操作
  }
# 闭合之前的代码块
```