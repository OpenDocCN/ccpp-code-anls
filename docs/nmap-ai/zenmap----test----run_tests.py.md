# `nmap\zenmap\test\run_tests.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

import unittest
# 导入 unittest 模块

if __name__ == "__main__":
    # 如果当前脚本为主程序

    import sys
    # 导入 sys 模块
    import glob
    # 导入 glob 模块
    import os
    # 导入 os 模块

    if not hasattr(unittest.defaultTestLoader, "discover"):
        # 如果 unittest 模块不支持测试用例的自动发现
        print("Python unittest discovery missing. Requires Python 3.0 or newer.")  # noqa
        # 打印提示信息
        sys.exit(0)
        # 退出程序

    os.chdir("..")
    # 切换当前工作目录到上一级目录
    suite = unittest.defaultTestLoader.discover(
        start_dir=glob.glob("build/lib*")[0],
        # 使用 glob 模块找到指定模式的文件夹，并取第一个
        pattern="*.py"
        # 匹配文件名的模式
        )
    # 使用 unittest 模块的默认测试加载器自动发现测试用例
    unittest.TextTestRunner().run(suite)
    # 运行测试用例并输出结果
```