# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\GreedyParams.java`

```cpp
package io.github.ggerganov.whispercpp.params;

import com.sun.jna.Structure;

import java.util.Collections;
import java.util.List;

public class GreedyParams extends Structure {
    /** 
     * 定义 GreedyParams 类，继承自 Structure 类
     * <a href="https://github.com/openai/whisper/blob/f82bc59f5ea234d4b97fb2860842ed38519f7e65/whisper/transcribe.py#L264">...</a> 
     * best_of 属性表示最佳结果
     */
    public int best_of;

    @Override
    protected List<String> getFieldOrder() {
        // 返回只包含 best_of 字段的字段顺序列表
        return Collections.singletonList("best_of");
    }
}
```