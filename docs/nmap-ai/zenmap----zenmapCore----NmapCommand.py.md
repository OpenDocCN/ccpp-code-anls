# `nmap\zenmap\zenmapCore\NmapCommand.py`

```
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
# 本文件包含 NmapCommand 类的定义，该类表示并运行一个 Nmap 命令行。
# 导入所需的模块和库
import codecs
import errno
import locale
import sys
import os
import tempfile
import unittest
# 导入 zenmapCore.I18N 模块，但未使用
import zenmapCore.I18N

# 导入 subprocess 模块
import subprocess

# 导入 zenmapCore.Paths 模块
import zenmapCore.Paths
# 导入 NmapOptions 类
from zenmapCore.NmapOptions import NmapOptions
# 导入日志模块
from zenmapCore.UmitLogging import log
# 导入 zenmapCore.UmitConf 模块中的 PathsConfig 类
from zenmapCore.UmitConf import PathsConfig
# 导入应用名称
from zenmapCore.Name import APP_NAME

# 从 zenmap.conf 中获取 [paths] 配置，用于获取 nmap_command_path
paths_config = PathsConfig()

# 打印平台信息
log.debug(">>> Platform: %s" % sys.platform)

# 定义函数，用于转义 Nmap 文件名中的 '%' 字符，避免被解释为 strftime 格式
def escape_nmap_filename(filename):
    """Escape '%' characters so they are not interpreted as strftime format
    specifiers, which are not supported by Zenmap."""
    # 将字符串中的百分号替换为两个百分号，用于格式化输出
    return filename.replace("%", "%%")
# 定义一个 NmapCommand 类，用于表示一个 Nmap 命令行。该类负责启动、停止和返回命令行扫描的结果。命令行表示为一个字符串，但会被拆分为执行的参数列表。
# 正常的输出（stdout 和 stderr）会被写入到 self.stdout_file 文件对象中。
class NmapCommand(object):
    """This class represents an Nmap command line. It is responsible for
    starting, stopping, and returning the results from a command-line scan. A
    command line is represented as a string but it is split into a list of
    arguments for execution.

    The normal output (stdout and stderr) are written to the file object
    self.stdout_file."""
    # 初始化一个 Nmap 命令对象，创建临时文件用于重定向各种类型的输出，并设置命令行字符串
    def __init__(self, command):
        self.command = command
        self.command_process = None

        self.stdout_file = None

        # 创建 NmapOptions 对象并解析命令字符串
        self.ops = NmapOptions()
        self.ops.parse_string(command)
        # 用 nmap_command_path 的值替换可执行文件名
        self.ops.executable = paths_config.nmap_command_path

        # 通常我们会生成一个随机的临时文件名来保存 XML 输出。如果找到 -oX 或 -oA，用户选择了自己的输出文件。
        # 将 self.xml_is_temp 设置为 False，并在完成后不删除文件。
        self.xml_is_temp = True
        self.xml_output_filename = None
        if self.ops["-oX"]:
            self.xml_is_temp = False
            self.xml_output_filename = self.ops["-oX"]
        if self.ops["-oA"]:
            self.xml_is_temp = False
            self.xml_output_filename = self.ops["-oA"] + ".xml"

        # 转义 '%' 以避免 strftime 扩展
        for op in ("-oA", "-oX", "-oG", "-oN", "-oS"):
            if self.ops[op]:
                self.ops[op] = escape_nmap_filename(self.ops[op])

        # 如果是临时文件，则创建临时文件名并设置为 XML 输出文件名
        if self.xml_is_temp:
            fh, self.xml_output_filename = tempfile.mkstemp(
                    prefix=APP_NAME + "-", suffix=".xml")
            os.close(fh)
            self.ops["-oX"] = escape_nmap_filename(self.xml_output_filename)

        log.debug(">>> 临时文件:")
        log.debug(">>> XML 输出: %s" % self.xml_output_filename)

    # 关闭并删除命令使用的临时输出文件
    def close(self):
        self.stdout_file.close()
        if self.xml_is_temp:
            try:
                os.remove(self.xml_output_filename)
            except OSError as e:
                if e.errno != errno.ENOENT:
                    raise
    # 终止 nmap 子进程
    def kill(self):
        """Kill the nmap subprocess."""
        # 导入 sleep 函数
        from time import sleep

        # 打印日志，显示要终止的扫描进程的 PID
        log.debug(">>> Killing scan process %s" % self.command_process.pid)

        # 如果不是 Windows 平台
        if sys.platform != "win32":
            try:
                # 导入信号常量
                from signal import SIGTERM, SIGKILL
                # 发送 SIGTERM 信号终止进程
                os.kill(self.command_process.pid, SIGTERM)
                # 等待一段时间，检查进程是否已经终止
                for i in range(10):
                    sleep(0.5)
                    if self.command_process.poll() is not None:
                        # 进程已经被终止
                        break
                else:
                    # 打印日志，SIGTERM 在等待 5 秒后仍未生效，使用 SIGKILL
                    log.debug(">>> SIGTERM has not worked even after waiting for 5 seconds. Using SIGKILL.")  # noqa
                    # 发送 SIGKILL 信号终止进程
                    os.kill(self.command_process.pid, SIGKILL)
                    # 等待进程结束
                    self.command_process.wait()
            except Exception:
                pass
        # 如果是 Windows 平台
        else:
            try:
                # 导入 ctypes 库
                import ctypes
                # 使用 TerminateProcess 函数终止进程
                ctypes.windll.kernel32.TerminateProcess(
                        int(self.command_process._handle), -1)
            except Exception:
                pass

    # 获取适合当前平台的 PATH 环境变量值
    def get_path(self):
        """Return a value for the PATH environment variable that is appropriate
        for the current platform. It will be the PATH from the environment plus
        possibly some platform-specific directories."""
        # 获取当前的 PATH 环境变量值
        path_env = os.getenv("PATH")
        # 如果 PATH 环境变量值为空，则初始化搜索路径列表
        if path_env is None:
            search_paths = []
        else:
            # 否则，将 PATH 环境变量值拆分成搜索路径列表
            search_paths = path_env.split(os.pathsep)
        # 获取额外的可执行文件搜索路径
        for path in zenmapCore.Paths.get_extra_executable_search_paths():
            # 如果路径不在搜索路径列表中，则添加进去
            if path not in search_paths:
                search_paths.append(path)
        # 返回拼接后的搜索路径列表
        return os.pathsep.join(search_paths)
    def run_scan(self, stderr=None):
        """Run the command represented by this class."""

        # 创建一个临时文件对象，用于存储标准输出，文件名以应用程序名开头，文件关闭后会被删除
        f = tempfile.TemporaryFile(mode="r", prefix=APP_NAME + "-stdout-")
        self.stdout_file = f
        # 如果未提供标准错误输出文件，则使用标准输出文件
        if stderr is None:
            stderr = f

        # 获取系统路径，并将其添加到环境变量中
        search_paths = self.get_path()
        env = dict(os.environ)
        env["PATH"] = search_paths
        log.debug("PATH=%s" % env["PATH"])

        # 获取表示要执行的命令的列表
        command_list = self.ops.render()
        log.debug("Running command: %s" % repr(command_list))

        startupinfo = None
        # 如果是 Windows 平台，设置启动信息以防止打开终端窗口
        if sys.platform == "win32":
            startupinfo = subprocess.STARTUPINFO()
            try:
                startupinfo.dwFlags |= \
                        subprocess._subprocess.STARTF_USESHOWWINDOW
            except AttributeError:
                # 在 Python 2.6.5 之前的版本中使用此名称
                startupinfo.dwFlags |= subprocess.STARTF_USESHOWWINDOW

        # 启动子进程来执行命令
        self.command_process = subprocess.Popen(command_list, bufsize=1,
                                     universal_newlines=True,
                                     stdin=subprocess.PIPE,
                                     stdout=f,
                                     stderr=stderr,
                                     startupinfo=startupinfo,
                                     env=env)
    # 返回运行扫描的当前状态。返回True表示扫描正在运行，返回False表示子进程成功完成。如果子进程以错误终止，则会引发异常。在调用此方法之前，必须使用run_scan启动扫描。
    def scan_state(self):
        # 如果命令进程为空，则抛出异常
        if self.command_process is None:
            raise Exception("Scan is not running yet!")

        # 获取命令进程的状态
        state = self.command_process.poll()

        # 如果状态为None，表示进程仍在运行
        if state is None:
            return True  # True表示进程仍在运行
        # 如果状态为0，表示进程成功退出
        elif state == 0:
            return False  # False表示进程成功退出
        # 如果状态不为0，表示扫描执行过程中发生错误
        else:
            log.warning("An error occurred during the scan execution!")
            log.warning("Command that raised the exception: '%s'" %
                    self.ops.render_string())
            log.warning("Scan output:\n%s" % self.get_output())

            # 抛出异常，显示扫描执行过程中发生的错误
            raise Exception(
                    "An error occurred during the scan execution!\n\n'%s'" %
                    self.get_output())

    # 返回self.stdout_file的完整内容。这会修改文件指针。
    def get_output(self):
        self.stdout_file.seek(0)
        return self.stdout_file.read()

    # 返回XML（-oX）输出文件的名称
    def get_xml_output_filename(self):
        return self.xml_output_filename
# 如果当前模块被直接执行，而不是被导入到另一个模块中
if __name__ == '__main__':
    # 运行单元测试，输出结果
    unittest.TextTestRunner().run(
            # 从给定的测试用例类中加载测试用例
            unittest.TestLoader().loadTestsFromTestCase(SplitQuotedTest))
```