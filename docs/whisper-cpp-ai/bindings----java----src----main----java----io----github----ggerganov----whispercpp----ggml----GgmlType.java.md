# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\ggml\GgmlType.java`

```cpp
// 定义枚举类型 GgmlType，表示不同的数据类型
package io.github.ggerganov.whispercpp.ggml;

public enum GgmlType {
    GGML_TYPE_F32, // 单精度浮点数类型
    GGML_TYPE_F16, // 半精度浮点数类型
    GGML_TYPE_Q4_0, // 四位定点数类型，小数点位置为0
    GGML_TYPE_Q4_1, // 四位定点数类型，小数点位置为1
    REMOVED_GGML_TYPE_Q4_2,  // 已移除的四位定点数类型，不再支持
    REMOVED_GGML_TYPE_Q4_3, // 已移除的四位定点数类型，不再支持
    GGML_TYPE_Q5_0, // 五位定点数类型，小数点位置为0
    GGML_TYPE_Q5_1, // 五位定点数类型，小数点位置为1
    GGML_TYPE_Q8_0, // 八位定点数类型，小数点位置为0
    GGML_TYPE_Q8_1, // 八位定点数类型，小数点位置为1
    GGML_TYPE_I8, // 有符号八位整数类型
    GGML_TYPE_I16, // 有符号十六位整数类型
    GGML_TYPE_I32, // 有符号三十二位整数类型
    GGML_TYPE_COUNT, // 数据类型数量
}
```