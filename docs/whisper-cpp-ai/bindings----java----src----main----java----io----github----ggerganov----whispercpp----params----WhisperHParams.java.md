# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\WhisperHParams.java`

```cpp
public class WhisperHParams {
    // 定义词汇表大小
    int n_vocab       = 51864;
    // 定义音频上下文大小
    int n_audio_ctx   = 1500;
    // 定义音频状态大小
    int n_audio_state = 384;
    // 定义音频头部数量
    int n_audio_head  = 6;
    // 定义音频层数
    int n_audio_layer = 4;
    // 定义文本上下文大小
    int n_text_ctx    = 448;
    // 定义文本状态大小
    int n_text_state  = 384;
    // 定义文本头部数量
    int n_text_head   = 6;
    // 定义文本层数
    int n_text_layer  = 4;
    // 定义梅尔频谱数量
    int n_mels        = 80;
    // 定义特征类型
    int ftype         = 1;
}
```