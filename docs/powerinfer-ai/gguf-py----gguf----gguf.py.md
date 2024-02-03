# `PowerInfer\gguf-py\gguf\gguf.py`

```cpp
# 这个文件保留以确保兼容性。如果你想要从 Python 使用 GGUF API，
# 则不要直接导入 gguf/gguf.py。如果你正在寻找示例，请查看 gguf-py 目录。

# 导入模块
import importlib
import sys
from pathlib import Path

# 将上级目录添加到系统路径中
sys.path.insert(0, str(Path(__file__).parent.parent))

# 为尝试直接导入 gguf/gguf.py 而不是作为一个包的人提供兼容性
importlib.invalidate_caches()
import gguf  # noqa: E402

# 重新加载 gguf 模块
importlib.reload(gguf)
```