# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\callbacks\WhisperNewSegmentCallback.java`

```cpp
/**
 * 定义一个回调接口，用于处理文本段的回调
 * 每次生成新的文本段时调用
 * 使用 whisper_full_...() 函数获取文本段
 */
package io.github.ggerganov.whispercpp.callbacks;

import com.sun.jna.Callback;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.WhisperContext;
import io.github.ggerganov.whispercpp.model.WhisperState;

public interface WhisperNewSegmentCallback extends Callback {

    /**
     * 处理文本段的回调方法
     *
     * @param ctx        Whisper 上下文
     * @param state      Whisper 状态
     * @param n_new      新生成的文本段数量
     * @param user_data  用户数据
     */
    void callback(Pointer ctx, Pointer state, int n_new, Pointer user_data);
}
```