# `nmap\zenmap\zenmapCore\BasePaths.py`

```cpp
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）拥有版权，1996-2023 年。
# Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。
# 更多关于 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自由下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一种特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 提供此软件的源代码是因为我们相信用户有权在运行程序之前准确了解程序将要执行的操作。
# ***********************IMPORTANT NMAP LICENSE TERMS************************
# 导入操作系统模块
import os
# 导入操作系统路径模块
import os.path
# 导入系统模块
import sys

# 从 zenmapCore.Name 模块中导入 APP_NAME 变量
from zenmapCore.Name import APP_NAME

# 获取用户的主目录路径
HOME = os.path.expanduser("~")

# 在这个文件中，base_paths 字典给各种文件赋予了符号名称。例如，使用 base_paths.target_list 代替 'target_list.txt'。
base_paths = dict(user_config_file=APP_NAME + '.conf',
                  user_config_dir=os.path.join(HOME, '.' + APP_NAME),
                  scan_profile='scan_profile.usp',
                  profile_editor='profile_editor.xml',
                  recent_scans='recent_scans.txt',
                  target_list='target_list.txt',
                  options='options.xml',
                  user_home=HOME,
                  db=APP_NAME + ".db",
                  version=APP_NAME + "_version")
```