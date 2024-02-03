# `whisper.cpp\examples\whisper.android.java\app\src\androidTest\java\com\litongjava\whisper\android\java\ExampleInstrumentedTest.java`

```cpp
/**
 * 导入必要的类库
 */
package com.litongjava.whisper.android.java;

/**
 * 导入 Android 相关类库
 */
import android.content.Context;

/**
 * 导入测试相关类库
 */
import androidx.test.platform.app.InstrumentationRegistry;
import androidx.test.ext.junit.runners.AndroidJUnit4;

/**
 * 导入测试相关类库
 */
import org.junit.Test;
import org.junit.runner.RunWith;

/**
 * 导入断言相关类库
 */
import static org.junit.Assert.*;

/**
 * 测试类，用于在 Android 设备上执行测试
 */
@RunWith(AndroidJUnit4.class)
public class ExampleInstrumentedTest {
  /**
   * 测试方法，验证应用程序的上下文
   */
  @Test
  public void useAppContext() {
    // 获取被测试应用程序的上下文
    Context appContext = InstrumentationRegistry.getInstrumentation().getTargetContext();
    // 断言被测试应用程序的包名
    assertEquals("com.litongjava.whisper.android.java", appContext.getPackageName());
  }
}
```