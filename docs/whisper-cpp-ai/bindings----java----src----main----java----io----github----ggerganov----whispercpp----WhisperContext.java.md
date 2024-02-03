# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\WhisperContext.java`

```cpp
// 导入所需的类
package io.github.ggerganov.whispercpp;

import com.sun.jna.Structure;
import com.sun.jna.ptr.PointerByReference;
import io.github.ggerganov.whispercpp.ggml.GgmlType;
import io.github.ggerganov.whispercpp.WhisperModel;
import io.github.ggerganov.whispercpp.params.WhisperContextParams;

import java.util.List;

// 定义 WhisperContext 类，继承自 Structure 类
public class WhisperContext extends Structure {
    // 加载时间和开始时间的微秒数
    int t_load_us = 0;
    int t_start_us = 0;

    /** weight type (FP32 / FP16 / QX) */
    // 权重类型，可以是 FP32 / FP16 / QX，默认为 FP16
    GgmlType wtype = GgmlType.GGML_TYPE_F16;
    /** intermediate type (FP32 or FP16) */
    // 中间类型，可以是 FP32 或 FP16，默认为 FP16
    GgmlType itype = GgmlType.GGML_TYPE_F16;

    // 指向 WhisperModel 对象的指针
    public PointerByReference model;
    // 指向 whisper_vocab 对象的指针
    public PointerByReference vocab;
    // 指向 whisper_state 对象的指针，默认为 nullptr
    public PointerByReference state;

    /** populated by whisper_init_from_file_with_params() */
    // 由 whisper_init_from_file_with_params() 方法填充的模型路径
    String path_model;
    // WhisperContextParams 对象
    WhisperContextParams params;

    // 定义 ByReference 类，继承自 WhisperContext 类，用于引用传递
//    public static class ByReference extends WhisperContext implements Structure.ByReference {
//    }
//
//    // 定义 ByValue 类，继承自 WhisperContext 类，用于值传递
//    public static class ByValue extends WhisperContext implements Structure.ByValue {
//    }
//
//    // 指定字段的顺序
//    @Override
//    protected List<String> getFieldOrder() {
//        return List.of("t_load_us", "t_start_us", "wtype", "itype", "model", "vocab", "state", "path_model");
//    }
}
```