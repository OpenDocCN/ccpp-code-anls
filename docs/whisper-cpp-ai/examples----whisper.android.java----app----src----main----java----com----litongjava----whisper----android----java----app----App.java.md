# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\app\App.java`

```cpp
# 导入 Android 应用程序类
import android.app.Application;
# 导入工具类 Utils
import com.blankj.utilcode.util.Utils;

# 创建自定义的应用程序类 App，继承自 Application 类
public class App extends Application {
  # 重写 onCreate 方法
  @Override
  public void onCreate() {
    # 调用父类的 onCreate 方法
    super.onCreate();
    # 初始化 Utils 工具类，传入当前应用程序的上下文
    Utils.init(this);
  }
}
```