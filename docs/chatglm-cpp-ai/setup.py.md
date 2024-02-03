# `chatglm.cpp\setup.py`

```cpp
# 引入必要的库
import os
import re
import subprocess
import sys
from pathlib import Path

# 引入 setuptools 库中的 Extension、find_packages、setup 和 build_ext 函数
from setuptools import Extension, find_packages, setup
from setuptools.command.build_ext import build_ext

# 将 distutils Windows 平台规范转换为 CMake 的 -A 参数
PLAT_TO_CMAKE = {
    "win32": "Win32",
    "win-amd64": "x64",
    "win-arm32": "ARM",
    "win-arm64": "ARM64",
}

# 定义一个 CMakeExtension 类，需要一个 sourcedir 而不是文件列表
# 名称必须是 CMake 构建的单个输出扩展
# 如果需要多个扩展，请参考 scikit-build
class CMakeExtension(Extension):
    def __init__(self, name: str, sourcedir: str = "") -> None:
        super().__init__(name, sources=[])
        self.sourcedir = os.fspath(Path(sourcedir).resolve())

# 定义一个 CMakeBuild 类，继承自 build_ext
class CMakeBuild(build_ext):
    # 获取当前文件的路径
    HERE = Path(__file__).resolve().parent
    # 从指定文件中读取版本号
    version = re.search(r'__version__ = "(.*?)"', (HERE / "chatglm_cpp/__init__.py").read_text(encoding="utf-8")).group(1)

# 调用 setup 函数，配置项目信息
setup(
    version=version,  # 设置版本号
    packages=find_packages(),  # 查找所有包
    ext_modules=[CMakeExtension("chatglm_cpp._C")],  # 设置扩展模块
    cmdclass={"build_ext": CMakeBuild},  # 设置自定义构建命令
)
```