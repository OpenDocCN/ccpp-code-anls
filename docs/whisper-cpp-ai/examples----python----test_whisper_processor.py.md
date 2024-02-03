# `whisper.cpp\examples\python\test_whisper_processor.py`

```cpp
# 导入whisper_processor模块
import whisper_processor

# 尝试处理音频文件"./audio/wake_word_detected16k.wav"，使用语言模型"base.en"
try:
    # 调用whisper_processor模块中的process_audio函数，传入音频文件路径和语言模型
    result = whisper_processor.process_audio("./audio/wake_word_detected16k.wav", "base.en")
    # 打印处理结果
    print(result)
# 捕获可能发生的异常，并打印错误信息
except Exception as e:
    print(f"Error: {e}")
```