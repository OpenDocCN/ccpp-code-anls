# `nmap\zenmap\install_scripts\windows\boot_script.py`

```
# 自定义 py2exe 启动脚本

# 这个脚本在 py2exe 的 boot_common.py 之后运行。该文件覆盖了 sys.stderr 和 sys.stdout，将它们分别写入日志文件和黑洞。然而，stderr 日志文件的位置是 sys.executable + '.log'，通常是不可写的。我们将在这里更改它，以便写入其他可写路径。

# 导入必要的模块
import sys
import os
import os.path

# 只有在 py2exe 安装了其 Stderr 对象时才执行以下操作
if sys.stderr.__class__.__name__ == "Stderr":
    # 获取可写路径
    logdir = os.environ.get("LOCALAPPDATA",
            os.environ.get("APPDATA",
                os.environ.get("TEMP", "")))

    # 如果 stderr 文件不为空，则关闭它
    if sys.stderr._file is not None:
        sys.stderr._file.close()
    # 将 stderr 文件重定向到指定路径下的日志文件，并以追加模式打开
    sys.stderr._file = open(os.path.join(logdir, "zenmap.exe.log"), 'a')

# 清理不再需要的模块引用
del os
del sys
```