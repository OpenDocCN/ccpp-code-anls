# `nmap\zenmap\zenmapCore\Diff.py`

```cpp
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
# * Project"). Nmap is also a registered trademark of the Nmap Project.
# *
# * This program is distributed under the terms of the Nmap Public Source
# * License (NPSL). The exact license text applying to a particular Nmap
# * release or source code control revision is contained in the LICENSE
# * file distributed with that version of Nmap or source code control
# * revision. More Nmap copyright/legal information is available from
# * https://nmap.org/book/man-legal.html, and further information on the
# * NPSL license itself can be found at https://nmap.org/npsl/ . This
# * header summarizes some key points from the Nmap license, but is no
# * substitute for the actual license text.
# *
# * Nmap is generally free for end users to download and use themselves,
# * including commercial use. It is available from https://nmap.org.
# *
# * The Nmap license generally prohibits companies from using and
# * redistributing Nmap in commercial products, but we sell a special Nmap
# * OEM Edition with a more permissive license and special features for
# * this purpose. See https://nmap.org/oem/
# *
# * If you have received a written Nmap license agreement or contract
# * stating terms other than these (such as an Nmap OEM license), you may
# * choose to use and redistribute Nmap under those terms instead.
# *
# * The official Nmap Windows builds include the Npcap software
# * (https://npcap.com) for packet capture and transmission. It is under
# * separate license terms which forbid redistribution without special
# * permission. So the official Nmap Windows builds may not be redistributed
# * without special permission (such as an Nmap OEM license).
# *
# * Source is provided to this software because we believe users have a
# * right to know exactly what a program is going to do before they run it.
# 导入必要的模块
import os  # 导入操作系统模块
import subprocess  # 导入子进程管理模块
import sys  # 导入系统相关的模块
import tempfile  # 导入临时文件模块
# 防止加载 PyXML
import xml  # 导入 XML 模块
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]  # 从 XML 路径中排除包含 "_xmlplus" 的路径

import xml.sax  # 导入 XML 解析模块

from zenmapCore.Name import APP_NAME  # 从 zenmapCore.Name 模块导入 APP_NAME
from zenmapCore.NmapParser import NmapParserSAX  # 从 zenmapCore.NmapParser 模块导入 NmapParserSAX
from zenmapCore.UmitConf import PathsConfig  # 从 zenmapCore.UmitConf 模块导入 PathsConfig
from zenmapCore.UmitLogging import log  # 从 zenmapCore.UmitLogging 模块导入 log
import zenmapCore.Paths  # 导入 zenmapCore.Paths 模块

# 从 zenmap.conf 中获取 [paths] 配置，用于获取 ndiff_command_path
paths_config = PathsConfig()

class NdiffParseException(Exception):
    pass

def get_path():
    """Return a value for the PATH environment variable that is appropriate
    for the current platform. It will be the PATH from the environment plus
    possibly some platform-specific directories."""
    path_env = os.getenv("PATH")  # 获取当前环境的 PATH 变量
    # 如果环境变量中没有指定路径，则将搜索路径列表设置为空列表
    if path_env is None:
        search_paths = []
    # 如果环境变量中指定了路径，则将搜索路径列表设置为环境变量中指定的路径列表
    else:
        search_paths = path_env.split(os.pathsep)
    # 遍历额外的可执行文件搜索路径列表
    for path in zenmapCore.Paths.get_extra_executable_search_paths():
        # 如果路径不在搜索路径列表中，则将其添加到搜索路径列表中
        if path not in search_paths:
            search_paths.append(path)
    # 将搜索路径列表中的路径用 os.pathsep 连接起来，返回连接后的字符串
    return os.pathsep.join(search_paths)
class NdiffCommand(subprocess.Popen):
    # 定义 NdiffCommand 类，继承自 subprocess.Popen
    def __init__(self, filename_a, filename_b, temporary_filenames=[]):
        # 初始化方法，接受两个文件名参数和一个临时文件名列表参数
        self.temporary_filenames = temporary_filenames
        # 将临时文件名列表赋给实例变量 temporary_filenames

        search_paths = get_path()
        # 调用 get_path() 函数获取搜索路径
        env = dict(os.environ)
        # 复制一份当前环境变量
        env["PATH"] = search_paths
        # 将搜索路径添加到环境变量中
        if "Zenmap.app" in sys.executable:
            # 如果在 Zenmap.app 中执行
            # 移除可能干扰 Ndiff 运行的环境变量
            if "PYTHONPATH" in env:
                del env["PYTHONPATH"]
            if "PYTHONHOME" in env:
                del env["PYTHONHOME"]

        command_list = [
                paths_config.ndiff_command_path,
                "--verbose",
                "--",
                filename_a,
                filename_b
                ]
        # 构建命令列表，包括 ndiff 命令路径、参数和文件名
        self.stdout_file = tempfile.TemporaryFile(
                mode="r",
                prefix=APP_NAME + "-ndiff-",
                suffix=".xml"
                )
        # 创建一个临时文件对象，用于存储命令输出

        log.debug("Running command: %s" % repr(command_list))
        # 输出调试信息，显示即将执行的命令
        subprocess.Popen.__init__(
                self,
                command_list,
                universal_newlines=True,
                stdout=self.stdout_file,
                stderr=self.stdout_file,
                env=env,
                shell=(sys.platform == "win32")
                )
        # 调用父类的初始化方法，执行命令并将输出重定向到临时文件

    def get_scan_diff(self):
        # 定义获取扫描差异的方法
        self.wait()
        # 等待命令执行完成
        self.stdout_file.seek(0)
        # 将临时文件指针移动到文件开头

        return self.stdout_file.read()
        # 返回临时文件的内容

    def close(self):
        """Clean up temporary files."""
        # 定义清理临时文件的方法
        self.stdout_file.close()
        # 关闭临时文件
        for filename in self.temporary_filenames:
            # 遍历临时文件名列表
            log.debug("Remove temporary diff file %s." % filename)
            # 输出调试信息，显示即将删除的临时文件名
            os.remove(filename)
            # 删除临时文件
        self.temporary_filenames = []
        # 清空临时文件名列表
    # 定义一个方法，用于关闭当前对象
    def kill(self):
        # 调用对象的 close 方法，关闭当前对象
        self.close()
def ndiff(scan_a, scan_b):
    """Run Ndiff on two scan results, which may be filenames or NmapParserSAX
    objects, and return a running NdiffCommand object."""
    # 用于存储临时文件名的列表
    temporary_filenames = []

    # 如果 scan_a 是 NmapParserSAX 对象
    if isinstance(scan_a, NmapParserSAX):
        # 创建临时文件，将文件名添加到列表中
        fd, filename_a = tempfile.mkstemp(
                prefix=APP_NAME + "-diff-",
                suffix=".xml"
                )
        temporary_filenames.append(filename_a)
        f = os.fdopen(fd, "w")
        # 将 scan_a 写入到临时文件中
        scan_a.write_xml(f)
        f.close()
    else:
        # 如果 scan_a 不是 NmapParserSAX 对象，则直接使用其文件名
        filename_a = scan_a

    # 如果 scan_b 是 NmapParserSAX 对象
    if isinstance(scan_b, NmapParserSAX):
        # 创建临时文件，将文件名添加到列表中
        fd, filename_b = tempfile.mkstemp(
                prefix=APP_NAME + "-diff-",
                suffix=".xml"
                )
        temporary_filenames.append(filename_b)
        f = os.fdopen(fd, "w")
        # 将 scan_b 写入到临时文件中
        scan_b.write_xml(f)
        f.close()
    else:
        # 如果 scan_b 不是 NmapParserSAX 对象，则直接使用其文件名
        filename_b = scan_b

    # 返回 NdiffCommand 对象，传入两个文件名和临时文件名列表
    return NdiffCommand(filename_a, filename_b, temporary_filenames)
```