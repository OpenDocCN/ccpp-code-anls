# `PowerInfer\gguf-py\gguf\gguf.py`

```
# 这个文件保留以确保兼容性。如果你想从 Python 使用 GGUF API，
# 则不要直接导入 gguf/gguf.py。如果你正在寻找示例，请查看 gguf-py 目录下的示例。

# 导入必要的模块
import importlib
import sys
from pathlib import Path

# 将父目录添加到系统路径中
sys.path.insert(0, str(Path(__file__).parent.parent))

# 为尝试直接导入 gguf/gguf.py 的人提供兼容性
importlib.invalidate_caches()
import gguf  # noqa: E402

# 重新加载 gguf 模块
importlib.reload(gguf)
```