# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\task\TranscriptionTask.java`

```cpp
package com.litongjava.whisper.android.java.task;

import android.content.Context;
import android.os.Build;
import android.widget.TextView;

import com.blankj.utilcode.util.ThreadUtils;
import com.litongjava.jfinal.aop.Aop;
import com.litongjava.whisper.android.java.services.WhisperService;

import java.io.File;

// 定义一个名为 TranscriptionTask 的类，继承自 ThreadUtils.Task<Object>
public class TranscriptionTask extends ThreadUtils.Task<Object> {
  // 声明私有的 TextView 类型变量 tv 和 File 类型变量 sampleFile
  private final TextView tv;
  private final File sampleFile;

  // 构造方法，接收一个 TextView 对象和一个 File 对象作为参数
  public TranscriptionTask(TextView tv, File sampleFile) {
    this.tv = tv;
    this.sampleFile = sampleFile;

  }

  // 重写 doInBackground 方法
  @Override
  public Object doInBackground() {
    // 检查当前 Android 版本是否大于等于 Oreo
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      // 如果是，则调用 WhisperService 类的 transcribeSample 方法进行转录
      Aop.get(WhisperService.class).transcribeSample(tv, sampleFile);
    }else{
      // 如果不是，则在 TextView 中追加文本提示不支持当前 Android 设备
      tv.append("not supported android devices");
    }
    // 返回 null
    return null;
  }

  // 重写 onSuccess 方法
  @Override
  public void onSuccess(Object result) {
  }

  // 重写 onCancel 方法
  @Override
  public void onCancel() {
  }

  // 重写 onFail 方法
  @Override
  public void onFail(Throwable t) {
  }
}
```