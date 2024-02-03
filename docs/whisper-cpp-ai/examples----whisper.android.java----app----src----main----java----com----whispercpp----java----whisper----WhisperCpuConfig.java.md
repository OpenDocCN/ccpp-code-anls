# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\whispercpp\java\whisper\WhisperCpuConfig.java`

```cpp
# 导入必要的包
package com.whispercpp.java.whisper;

# 导入 Android 版本相关的类
import android.os.Build;

# 导入需要的注解
import androidx.annotation.RequiresApi;

# 定义 WhisperCpuConfig 类
public class WhisperCpuConfig {
  
  # 使用 RequiresApi 注解指定需要的 API 版本为 N
  @RequiresApi(api = Build.VERSION_CODES.N)
  
  # 定义静态方法 getPreferredThreadCount，返回优选线程数
  public static int getPreferredThreadCount() {
    
    # 返回高性能 CPU 数量和 2 中的较大值
    return Math.max(CpuInfo.getHighPerfCpuCount(), 2);
  }
}
```