# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\CBool.java`

```cpp
package io.github.ggerganov.whispercpp.params;

import com.sun.jna.IntegerType;

import java.util.function.BooleanSupplier;

// 定义一个继承自 IntegerType 类并实现 BooleanSupplier 接口的 CBool 类
public class CBool extends IntegerType implements BooleanSupplier {
    // 定义 CBool 类型的大小为 1
    public static final int SIZE = 1;
    // 定义 CBool 类型的 FALSE 值为 0
    public static final CBool FALSE = new CBool(0);
    // 定义 CBool 类型的 TRUE 值为 1
    public static final CBool TRUE = new CBool(1);

    // 无参构造函数，默认值为 0
    public CBool() {
        this(0);
    }

    // 带参构造函数，根据传入的值初始化 CBool 类型对象
    public CBool(long value) {
        super(SIZE, value, true);
    }

    // 实现 BooleanSupplier 接口的方法，返回 CBool 类型对象的值是否为 1
    @Override
    public boolean getAsBoolean() {
        return intValue() == 1;
    }

    // 重写 toString 方法，根据 CBool 类型对象的值返回对应的字符串 "true" 或 "false"
    @Override
    public String toString() {
        return intValue() == 1 ? "true" : "false";
    }
}
```