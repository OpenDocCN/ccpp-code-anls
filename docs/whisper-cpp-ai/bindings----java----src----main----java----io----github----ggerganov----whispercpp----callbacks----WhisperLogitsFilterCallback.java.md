# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\callbacks\WhisperLogitsFilterCallback.java`

```cpp
/**
 * 定义一个回调接口 WhisperLogitsFilterCallback，用于过滤 logits
 * 可以在采样之前修改 logits
 * 如果不为 null，在应用温度到 logits 后调用
 */
package io.github.ggerganov.whispercpp.callbacks;

import com.sun.jna.Callback;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.model.WhisperTokenData;

public interface WhisperLogitsFilterCallback extends Callback {

    /**
     * 回调方法，用于过滤 logits
     *
     * @param ctx        Whisper 上下文
     * @param state      Whisper 状态
     * @param tokens     whisper_token_data 数组
     * @param n_tokens   tokens 的数量
     * @param logits     logits 数组
     * @param user_data  用户数据
     */
    void callback(Pointer ctx, Pointer state, WhisperTokenData[] tokens, int n_tokens, float[] logits, Pointer user_data);
}
```