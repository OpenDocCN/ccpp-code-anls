# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\callbacks\WhisperEncoderBeginCallback.java`

```cpp
/**
 * 定义一个回调接口，用于在编码器开始之前执行操作
 * 如果不为null，则在编码器开始之前调用
 * 如果返回false，则计算被中止
 */
package io.github.ggerganov.whispercpp.callbacks;

import com.sun.jna.Callback;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.WhisperContext;
import io.github.ggerganov.whispercpp.model.WhisperState;

public interface WhisperEncoderBeginCallback extends Callback {

    /**
     * 在编码器开始之前调用的回调方法
     *
     * @param ctx        Whisper上下文
     * @param state      Whisper状态
     * @param user_data  用户数据
     * @return 如果计算应继续，则返回true，否则返回false
     */
    boolean callback(Pointer ctx, Pointer state, Pointer user_data);
}
```