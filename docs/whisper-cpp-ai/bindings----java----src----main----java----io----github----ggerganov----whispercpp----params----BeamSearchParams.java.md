# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\BeamSearchParams.java`

```cpp
package io.github.ggerganov.whispercpp.params;

import com.sun.jna.Structure;

import java.util.Arrays;
import java.util.List;

public class BeamSearchParams extends Structure {
    /** 定义 BeamSearchParams 结构体中的成员变量 beam_size，表示束搜索的大小 */
    public int beam_size;

    /** 定义 BeamSearchParams 结构体中的成员变量 patience，表示耐心值 */
    public float patience;

    /** 重写 getFieldOrder 方法，指定结构体中成员变量的顺序 */
    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("beam_size", "patience");
    }
}
```