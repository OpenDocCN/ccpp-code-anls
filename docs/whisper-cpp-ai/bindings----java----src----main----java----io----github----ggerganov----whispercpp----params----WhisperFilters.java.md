# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\WhisperFilters.java`

```cpp
// 定义一个名为 WhisperFilters 的类，用于存储滤波器参数
package io.github.ggerganov.whispercpp.params;

// 导入 List 类
import java.util.List;

// 定义 WhisperFilters 类
public class WhisperFilters {
    // 定义整型变量 n_mel，用于存储 mel 滤波器的数量
    int n_mel;
    // 定义整型变量 n_fft，用于存储 FFT 的大小
    int n_fft;

    // 定义一个泛型为 Float 的 List 类型变量 data，用于存储数据
    List<Float> data;
}
```