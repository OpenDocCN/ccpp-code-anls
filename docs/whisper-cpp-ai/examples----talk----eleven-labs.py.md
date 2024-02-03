# `whisper.cpp\examples\talk\eleven-labs.py`

```cpp
# 导入 sys 模块，用于访问与 Python 解释器交互的变量和函数
import sys
# 导入 importlib.util 模块，用于查找和加载模块
import importlib.util

# 如果未找到 elevenlabs 模块，则输出提示信息并退出程序
if importlib.util.find_spec("elevenlabs") is None:
    print("elevenlabs library is not installed, you can install it to your enviroment using 'pip install elevenlabs'")
    sys.exit()

# 从 elevenlabs 模块中导入 generate, play, save 函数
from elevenlabs import generate, play, save

# 获取一个 Voice 对象，可以通过名称或 UUID 获取
voice = "Arnold" #Possible Voices: Adam Antoni Arnold Bella Domi Elli Josh

# 生成文本转语音
audio = generate(
  text=str(sys.argv[2:]), # 从命令行参数中获取文本内容
  voice=voice
)

# 将生成的语音保存到文件中
save(audio, "audio.mp3") 
```