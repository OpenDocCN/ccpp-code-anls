# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\MainActivity.java`

```cpp
package com.litongjava.whisper.android.java;

import androidx.annotation.RequiresApi;
import androidx.appcompat.app.AppCompatActivity;

import android.content.Context;
import android.os.Build;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.view.View;
import android.widget.TextView;

import com.blankj.utilcode.util.ThreadUtils;
import com.litongjava.android.view.inject.annotation.FindViewById;
import com.litongjava.android.view.inject.annotation.FindViewByIdLayout;
import com.litongjava.android.view.inject.annotation.OnClick;
import com.litongjava.android.view.inject.utils.ViewInjectUtils;
import com.litongjava.jfinal.aop.Aop;
import com.litongjava.jfinal.aop.AopManager;
import com.litongjava.whisper.android.java.services.WhisperService;
import com.litongjava.whisper.android.java.task.LoadModelTask;
import com.litongjava.whisper.android.java.task.TranscriptionTask;
import com.litongjava.whisper.android.java.utils.AssetUtils;
import com.whispercpp.java.whisper.WhisperLib;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;

// 使用注解指定布局文件
@FindViewByIdLayout(R.layout.activity_main)
public class MainActivity extends AppCompatActivity {

  // 使用注解找到对应的 View
  @FindViewById(R.id.sample_text)
  private TextView tv;

  // 初始化日志记录器
  Logger log = LoggerFactory.getLogger(this.getClass());
  // 获取 WhisperService 实例
  private WhisperService whisperService = Aop.get(WhisperService.class);

  // 在指定 API 版本下执行的方法
  @RequiresApi(api = Build.VERSION_CODES.O)
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //setContentView(R.layout.activity_main);
    // 注入当前 Activity 的 View
    ViewInjectUtils.injectActivity(this, this);
    // 初始化 AopBean
    initAopBean();
    // 显示系统信息
    showSystemInfo();
  }

  // 初始化 AopBean
  private void initAopBean() {
    // 创建主线程 Handler
    Handler mainHandler = new Handler(Looper.getMainLooper());
    // 将主线程 Handler 添加为单例对象
    AopManager.me().addSingletonObject(mainHandler);
  }

  // 在指定 API 版本下执行的点击事件处理方法
  @RequiresApi(api = Build.VERSION_CODES.O)
  @OnClick(R.id.loadModelBtn)
  public void loadModelBtn_OnClick(View v) {
  // 获取应用程序的上下文
  Context context = getBaseContext();
  // 在IO线程中执行加载模型的任务
  ThreadUtils.executeByIo(new LoadModelTask(tv));
}

// 点击transcriptSampleBtn按钮时的事件处理方法
@OnClick(R.id.transcriptSampleBtn)
public void transcriptSampleBtn_OnClick(View v) {
  // 获取应用程序的上下文
  Context context = getBaseContext();

  // 记录开始时间
  long start = System.currentTimeMillis();
  // 设置示例文件路径
  String sampleFilePath = "samples/jfk.wav";
  // 获取应用程序文件目录
  File filesDir = context.getFilesDir();
  // 如果示例文件不存在，则复制文件到应用程序文件目录
  File sampleFile = AssetUtils.copyFileIfNotExists(context, filesDir, sampleFilePath);
  // 记录结束时间
  long end = System.currentTimeMillis();
  // 输出复制文件所花费的时间
  String msg = "copy file:" + (end - start) + "ms";
  outputMsg(tv, msg);
  // 在IO线程中执行转录任务
  ThreadUtils.executeByIo(new TranscriptionTask(tv, sampleFile));
}

// 输出消息到TextView和日志
private void outputMsg(TextView tv, String msg) {
  tv.append(msg + "\n");
  log.info(msg);
}

// 点击systemInfoBtn按钮时的事件处理方法
@RequiresApi(api = Build.VERSION_CODES.O)
@OnClick(R.id.systemInfoBtn)
public void systemInfoBtn_OnClick(View v) {
  // 显示系统信息
  showSystemInfo();
}

// 显示系统信息
@RequiresApi(api = Build.VERSION_CODES.O)
public void showSystemInfo() {
  // 获取系统信息并显示在TextView中
  String systemInfo = WhisperLib.getSystemInfo();
  tv.append(systemInfo + "\n");
}

// 点击clearBtn按钮时的事件处理方法
@OnClick(R.id.clearBtn)
public void clearBtn_OnClick(View v) {
  // 清空TextView中的内容
  tv.setText("");
}

// 在Activity销毁时释放资源
@RequiresApi(api = Build.VERSION_CODES.O)
@Override
protected void onDestroy() {
  super.onDestroy();
  // 释放WhisperService资源
  whisperService.release();
}
# 闭合大括号，表示代码块的结束
```