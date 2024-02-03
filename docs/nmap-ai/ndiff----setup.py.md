# `nmap\ndiff\setup.py`

```cpp
#!/usr/bin/env python
# 指定脚本的解释器为 Python

import errno
import sys
import os
import os.path
import re

from stat import ST_MODE

import distutils.command
import distutils.command.install
import distutils.core
import distutils.cmd
import distutils.errors
from distutils import log
from distutils.command.install import install
# 导入所需的模块和库

APP_NAME = "ndiff"
# 定义应用程序的名称
# 用于记录已安装文件列表的文件名，以便卸载命令可以删除它们。
INSTALLED_FILES_NAME = "INSTALLED_FILES"

# path_startswith 和 path_strip_prefix 用于处理安装根目录（--root 选项，也称为 DESTDIR）。
def path_startswith(path, prefix):
    """Returns True if path starts with prefix. It's a little more intelligent
    than str.startswith because it normalizes the paths to remove multiple
    directory separators and down-up traversals."""
    # 如果路径以指定前缀开头，则返回 True。它比 str.startswith 更智能，因为它会规范化路径以删除多个目录分隔符和向上向下遍历。
    path = os.path.normpath(path)
    prefix = os.path.normpath(prefix)
    return path.startswith(prefix)

def path_strip_prefix(path, prefix):
    """Return path stripped of its directory prefix if it starts with prefix,
    otherwise return path unmodified. This only works correctly with Unix
    paths; for example it will not replace the drive letter on a Windows path.
    Examples:
    >>> path_strip_prefix('/tmp/destdir/usr/bin', '/tmp/destdir')
    '/usr/bin'
    >>> path_strip_prefix('/tmp/../tmp/destdir/usr/bin', '/tmp///destdir')
    '/usr/bin'
    >>> path_strip_prefix('/etc', '/tmp/destdir')
    '/etc'
    >>> path_strip_prefix('/etc', '/')
    '/etc'
    >>> path_strip_prefix('/etc', '')
    '/etc'
    """
    # 如果路径以指定前缀开头，则返回去除前缀的路径，否则返回未修改的路径。这仅在 Unix 路径上正确工作；例如，它不会替换 Windows 路径上的驱动器号。
    absolute = os.path.isabs(path)
    path = os.path.normpath(path)
    prefix = os.path.normpath(prefix)
    if path.startswith(prefix) and prefix != os.sep:
        path = path[len(prefix):]
    # 绝对路径必须保持绝对，相对路径必须保持相对。
    assert os.path.isabs(path) == absolute
    return path
###############################################################################
# Distutils subclasses

# 创建一个空的 distutils 命令，用于替换 install_egg_info 命令，避免安装 .egg-info 文件
class null_command(distutils.cmd.Command):
    """This is a dummy distutils command that does nothing. We use it to
    replace the install_egg_info and avoid installing a .egg-info file, because
    there's no option to disable that."""
    # 初始化选项
    def initialize_options(self):
        pass

    # 完成选项设置
    def finalize_options(self):
        pass

    # 获取输出结果
    def get_outputs(self):
        return ()

    # 运行命令
    def run(self):
        pass


# 创建一个包装了 install 命令的 checked_install 命令，用于检查由于未安装 python-dev 而导致的错误
class checked_install(distutils.command.install.install):
    """This is a wrapper around the install command that checks for an error
    caused by not having the python-dev package installed. By default,
    distutils gives a misleading error message: "invalid Python installation."
    """

    # 完成选项设置
    def finalize_options(self):
        # Ubuntu 的 python2.6-2.6.4-0ubuntu3 包在 sys.prefix 为 "/usr/local"（我们的默认值）时会改变 sys.prefix
        # 因此，为了后续需要，这里保存未改变的值
        self.saved_prefix = sys.prefix
        try:
            # 调用父类的 finalize_options 方法
            distutils.command.install.install.finalize_options(self)
        except distutils.errors.DistutilsPlatformError as e:
            # 捕获并重新抛出错误，添加额外的解决问题的建议
            raise distutils.errors.DistutilsPlatformError(str(e) + """
Installing your distribution's python-dev package may solve this problem.""")
    # 设置模块路径的方法
    def set_modules_path(self):
        # 拼接应用程序文件名
        app_file_name = os.path.join(self.install_scripts, APP_NAME)
        # 查找模块安装的位置，distutils 会将它们放在 self.install_lib 中，但该路径可能包含根目录（DESTDIR），所以必要时必须将其剥离
        modules_dir = self.install_lib
        if self.root is not None:
            modules_dir = path_strip_prefix(modules_dir, self.root)

        # 打开应用程序文件
        app_file = open(app_file_name, "r")
        # 读取文件的所有行
        lines = app_file.readlines()
        # 关闭文件
        app_file.close()

        # 遍历文件的每一行
        for i in range(len(lines)):
            # 如果匹配到 INSTALL_LIB 开头的行
            if re.match(r'^INSTALL_LIB =', lines[i]):
                # 替换该行为模块目录的路径
                lines[i] = "INSTALL_LIB = %s\n" % repr(modules_dir)
                # 结束循环
                break
        else:
            # 如果未找到替换的行，则抛出数值错误
            raise ValueError(
                    "INSTALL_LIB replacement not found in %s" % app_file_name)

        # 以写模式打开应用程序文件
        app_file = open(app_file_name, "w")
        # 将修改后的行写入文件
        app_file.writelines(lines)
        # 关闭文件
        app_file.close()

    # 运行方法
    def run(self):
        # 调用父类的 run 方法
        install.run(self)
# 从 Zenmap 中获取以下内容。目前只使用 set_modules_path，但我们可能会考虑其他内容是否有用（或者，如果没有用，是否应该从 Zenmap 中删除它们）。
# 设置权限
# 设置模块路径
# 修复路径
# 创建卸载程序
# 写入已安装的文件

def get_installed_files(self):
    """返回已安装的文件和目录列表，如果给定安装根目录，则每个都带有前缀。已安装目录的列表不是来自 distutils，因此可能不完整。"""
    # 获取已安装的文件列表
    installed_files = self.get_outputs()
    # 遍历分发的 Python 模块列表，将每个模块的目录添加到已安装文件列表中
    for package in self.distribution.py_modules:
        dir = package.replace(".", "/")
        installed_files.append(os.path.join(self.install_lib, dir))
    # 添加卸载程序文件路径到已安装文件列表中
    installed_files.append(
            os.path.join(self.install_scripts, "uninstall_" + APP_NAME))
    return installed_files

def create_uninstaller(self):
    # 设置卸载程序文件名
    uninstaller_filename = os.path.join(
            self.install_scripts, "uninstall_" + APP_NAME)

    # 创建卸载程序内容
    uninstaller = """\
#!/usr/bin/env python
import errno, os, os.path, sys

print('Uninstall %(name)s')

answer = raw_input('Are you sure that you want to uninstall '
    '%(name)s (yes/no) ')

if answer != 'yes' and answer != 'y':
    print('Not uninstalling.')
    sys.exit(0)
# 使用字符串格式化将 APP_NAME 替换到字符串中
""" % {'name': APP_NAME}

# 初始化已安装文件列表
installed_files = []
# 遍历已安装文件列表
for output in self.get_installed_files():
    if self.root is not None:
        # 如果有 root（DESTDIR），需要去掉路径前面的 root，以便卸载程序在目标主机上运行
        if not path_startswith(output, self.root):
            # 如果发生这种情况（所有内容都安装在 root 内），则安全起见，不删除任何内容
            uninstaller += ("print('%s was not installed inside "
                "the root %s; skipping.')\n" % (output, self.root))
            continue
        output = path_strip_prefix(output, self.root)
        assert os.path.isabs(output)
    installed_files.append(output)

# 将已安装文件列表添加到卸载程序中
uninstaller += """\
INSTALLED_FILES = (
"""
for file in installed_files:
    uninstaller += "    %s,\n" % repr(file)
uninstaller += """\
)

# 将已安装文件列表分成文件和目录两个列表
files = []
dirs = []
for path in INSTALLED_FILES:
    if os.path.isfile(path) or os.path.islink(path):
        files.append(path)
    elif os.path.isdir(path):
        dirs.append(path)
# 删除文件
for file in files:
    print("Removing '%s'." % file)
    try:
        os.remove(file)
    except OSError, e:
        print('  Error: %s.' % str(e), file=sys.stderr)
# 删除目录。首先按长度逆向排序标准化路径，以便子目录在父目录之前被删除
dirs = [os.path.normpath(dir) for dir in dirs]
dirs.sort(key = len, reverse = True)
for dir in dirs:
    try:
        print("Removing the directory '%s'." % dir)
        os.rmdir(dir)
    # 捕获 OSError 异常，并将异常对象赋值给变量 e
    except OSError, e:
        # 如果异常的错误码是 ENOTEMPTY
        if e.errno == errno.ENOTEMPTY:
            # 打印目录不为空的提示信息
            print("Directory '%s' not empty; not removing." % dir)
        # 如果异常的错误码不是 ENOTEMPTY
        else:
            # 打印异常信息到标准错误输出
            print(str(e), file=sys.stderr)
# 打开卸载程序文件，以写入方式
uninstaller_file = open(uninstaller_filename, 'w')
# 写入卸载程序内容
uninstaller_file.write(uninstaller)
# 关闭卸载程序文件
uninstaller_file.close()

# 设置卸载程序的执行权限
mode = ((os.stat(uninstaller_filename)[ST_MODE]) | 0o555) & 0o7777
os.chmod(uninstaller_filename, mode)

# 定义写入已安装文件列表的方法
def write_installed_files(self):
    """Write a list of installed files for use by the uninstall command.
    This is similar to what happens with the --record option except that it
    doesn't strip off the installation root, if any. File names containing
    newline characters are not handled."""
    # 如果已安装文件名与记录文件名相同，则发出警告
    if INSTALLED_FILES_NAME == self.record:
        distutils.log.warn("warning: installation record is overwriting "
            "--record file '%s'." % self.record)
    # 以写入方式打开已安装文件列表文件
    with open(INSTALLED_FILES_NAME, "w") as f:
        # 遍历已安装文件列表并写入文件
        for output in self.get_installed_files():
            # 确保输出中不包含换行符
            assert "\n" not in output
            print(output, file=f)

# 定义卸载命令类
class my_uninstall(distutils.cmd.Command):
    """A distutils command that performs uninstallation. It reads the list of
    installed files written by the install command."""

    # 命令名称
    command_name = "uninstall"
    # 描述
    description = "uninstall installed files recorded in '%s'" % (
            INSTALLED_FILES_NAME)
    # 用户选项
    user_options = []

    def initialize_options(self):
        pass

    def finalize_options(self):
        pass
    # 定义一个方法，用于运行卸载操作
    def run(self):
        # 尝试打开已安装文件列表
        try:
            f = open(INSTALLED_FILES_NAME, "r")
        except IOError as e:
            # 如果文件不存在，则输出错误信息并返回
            if e.errno == errno.ENOENT:
                log.error("Couldn't open the installation record '%s'. "
                        "Have you installed yet?" % INSTALLED_FILES_NAME)
                return
        # 读取已安装文件列表，并去除换行符
        installed_files = [file.rstrip("\n") for file in f.readlines()]
        f.close()
        # 将安装记录文件名添加到已安装文件列表中
        installed_files.append(INSTALLED_FILES_NAME)
        # 将文件列表分割成文件和目录两个列表
        files = []
        dirs = []
        for path in installed_files:
            # 如果是文件或符号链接，则添加到文件列表中
            if os.path.isfile(path) or os.path.islink(path):
                files.append(path)
            # 如果是目录，则添加到目录列表中
            elif os.path.isdir(path):
                dirs.append(path)
        # 删除文件
        for file in files:
            log.info("Removing '%s'." % file)
            try:
                # 如果不是模拟运行，则删除文件
                if not self.dry_run:
                    os.remove(file)
            except OSError as e:
                log.error(str(e))
        # 删除目录。首先对标准化后的路径进行反向排序，以便子目录在父目录之前被删除
        dirs = [os.path.normpath(dir) for dir in dirs]
        dirs.sort(key=len, reverse=True)
        for dir in dirs:
            try:
                log.info("Removing the directory '%s'." % dir)
                if not self.dry_run:
                    os.rmdir(dir)
            except OSError as e:
                # 如果目录不为空，则输出信息并不删除
                if e.errno == errno.ENOTEMPTY:
                    log.info("Directory '%s' not empty; not removing." % dir)
                else:
                    log.error(str(e))
# 使用 distutils.core.setup 方法设置项目名称和相关配置
distutils.core.setup(name="ndiff", scripts=["scripts/ndiff"],
    py_modules=["ndiff"],
    data_files=[("share/man/man1", ["docs/ndiff.1"])],
    # 设置自定义命令的类
    cmdclass={
        "install_egg_info": null_command,
        "install": checked_install,
        "uninstall": my_uninstall
        })
```