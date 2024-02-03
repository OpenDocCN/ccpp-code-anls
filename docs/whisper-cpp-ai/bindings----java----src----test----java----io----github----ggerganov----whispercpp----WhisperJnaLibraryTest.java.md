# `whisper.cpp\bindings\java\src\test\java\io\github\ggerganov\whispercpp\WhisperJnaLibraryTest.java`

```cpp
// 导入所需的类
package io.github.ggerganov.whispercpp;

// 导入断言类
import static org.junit.jupiter.api.Assertions.*;

// 导入测试类
import org.junit.jupiter.api.Test;

// 定义测试类 WhisperJnaLibraryTest
class WhisperJnaLibraryTest {

    // 定义测试方法 testWhisperPrint_system_info
    @Test
    void testWhisperPrint_system_info() {
        // 调用 WhisperCppJnaLibrary 实例的 whisper_print_system_info 方法，获取系统信息字符串
        String systemInfo = WhisperCppJnaLibrary.instance.whisper_print_system_info();
        // 打印系统信息字符串
        System.out.println("System info: " + systemInfo);
        // 断言系统信息字符串长度大于 10
        assertTrue(systemInfo.length() > 10);
    }
}
```