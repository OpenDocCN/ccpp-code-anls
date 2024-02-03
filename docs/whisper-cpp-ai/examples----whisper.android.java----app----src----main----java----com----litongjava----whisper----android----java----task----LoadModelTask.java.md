# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\task\LoadModelTask.java`

```cpp
package com.litongjava.whisper.android.java.task;

import android.content.Context;
import android.os.Build;
import android.os.Handler;
import android.widget.TextView;

import com.blankj.utilcode.util.ThreadUtils;
import com.litongjava.jfinal.aop.Aop;
import com.litongjava.whisper.android.java.services.WhisperService;

import java.io.File;

// 定义一个加载模型的任务类，继承自ThreadUtils.Task<Object>
public class LoadModelTask extends ThreadUtils.Task<Object> {
  // 声明一个TextView变量
  private final TextView tv;
  // 构造方法，初始化TextView变量
  public LoadModelTask(TextView tv) {
    this.tv = tv;
  }

  // 在后台执行任务
  @Override
  public Object doInBackground() {
    // 判断当前Android版本是否大于等于Oreo
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      // 使用Aop获取WhisperService实例，调用加载模型方法
      Aop.get(WhisperService.class).loadModel(tv);
    }else{
      // 如果Android版本不支持，通过Handler在主线程中更新UI显示提示信息
      Aop.get(Handler.class).post(()->{
        tv.append("not supported android devices");
      });

    }
    return null;
  }

  // 任务执行成功时的回调方法
  @Override
  public void onSuccess(Object result) {
  }

  // 任务被取消时的回调方法
  @Override
  public void onCancel() {
  }

  // 任务执行失败时的回调方法
  @Override
  public void onFail(Throwable t) {
  }
}
```