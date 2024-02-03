# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\callbacks\WhisperProgressCallback.java`

```cpp
// 导入必要的包
package io.github.ggerganov.whispercpp.callbacks;

// 导入 Callback 接口和相关类
import com.sun.jna.Callback;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.WhisperContext;
import io.github.ggerganov.whispercpp.model.WhisperState;

/**
 * 用于进度更新的回调接口。
 */
public interface WhisperProgressCallback extends Callback {

    /**
     * 进度更新的回调方法。
     *
     * @param ctx        Whisper 上下文。
     * @param state      Whisper 状态。
     * @param progress   进度值。
     * @param user_data  用户数据。
     */
    void callback(Pointer ctx, Pointer state, int progress, Pointer user_data);
}
```