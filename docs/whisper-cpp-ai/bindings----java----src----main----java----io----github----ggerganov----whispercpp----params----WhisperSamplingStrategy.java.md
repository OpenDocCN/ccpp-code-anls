# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\WhisperSamplingStrategy.java`

```cpp
// 定义枚举类型 WhisperSamplingStrategy，表示可用的采样策略
public enum WhisperSamplingStrategy {
    // 采样策略：类似于 OpenAI 的 GreedyDecoder
    WHISPER_SAMPLING_GREEDY,

    // 采样策略：类似于 OpenAI 的 BeamSearchDecoder
    WHISPER_SAMPLING_BEAM_SEARCH
}
```