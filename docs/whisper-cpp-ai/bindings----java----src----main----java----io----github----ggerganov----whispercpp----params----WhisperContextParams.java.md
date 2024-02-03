# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\WhisperContextParams.java`

```cpp
/**
 * 参数类，用于 whisper_init_from_file_with_params() 函数
 * 如果更改参数顺序或添加新参数，请确保在 whisper.cpp 中更新默认值：whisper_context_default_params()
 */
public class WhisperContextParams extends Structure {

    public WhisperContextParams(Pointer p) {
        super(p);
    }

    /** 使用 GPU 进行推断（默认为 true） */
    public CBool use_gpu;

    /** 设置是否使用 GPU 进行推断（默认为 true） */
    public void useGpu(boolean enable) {
        use_gpu = enable ? CBool.TRUE : CBool.FALSE;
    }

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("use_gpu");
    }
}
```