# `nmap\zenmap\setup.py`

```
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）拥有版权，1996-2023 年间开发
# Nmap 也是 Nmap 项目的注册商标
# 本程序遵循 Nmap 公共源代码许可证（NPSL）的条款。具体的许可证文本包含在相应 Nmap 版本或源代码控制修订版的 LICENSE 文件中
# 更多 Nmap 版权/法律信息可从 https://nmap.org/book/man-legal.html 获取，NPSL 许可证本身的更多信息可在 https://nmap.org/npsl/ 找到
# 本头部摘要了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本

# Nmap 通常允许最终用户自由下载和使用，包括商业用途。可从 https://nmap.org 获取

# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售特殊的 Nmap OEM 版本，具有更宽松的许可证和特殊功能。请参阅 https://nmap.org/oem/

# 如果您收到书面的 Nmap 许可协议或合同，其中规定了与这些条款不同的条款（如 Nmap OEM 许可证），您可以选择根据那些条款使用和重新分发 Nmap

# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到单独的许可证条款的约束，禁止未经特殊许可的重新分发。因此，官方的 Nmap Windows 构建可能不得未经特殊许可（如 Nmap OEM 许可证）重新分发

# 提供此软件的源代码是因为我们认为用户在运行程序之前有权知道程序将要做什么
# 导入 sys 模块
import sys

# 如果 Python 版本不是 3，就退出程序并打印错误信息
if sys.version_info[0] != 3:
    sys.exit("Sorry, Zenmap requires Python 3")

# 导入 errno、os、os.path、re 模块
import errno
import os
import os.path
import re

# 导入 distutils.sysconfig、distutils.log、distutils.core、distutils.command.install 模块
import distutils.sysconfig
from distutils import log
from distutils.core import setup, Command
from distutils.command.install import install

# 导入 glob、stat 模块
from glob import glob
from stat import S_IRGRP, S_IROTH, S_IRUSR, S_IRWXU, S_IWUSR, S_IXGRP, S_IXOTH, ST_MODE

# 导入 zenmapCore.Version、zenmapCore.Name 模块中的 VERSION、APP_NAME、APP_DISPLAY_NAME、APP_WEB_SITE、APP_DOWNLOAD_SITE、NMAP_DISPLAY_NAME
from zenmapCore.Version import VERSION
from zenmapCore.Name import APP_NAME, APP_DISPLAY_NAME, APP_WEB_SITE, APP_DOWNLOAD_SITE, NMAP_DISPLAY_NAME

# 用于记录已安装文件列表的文件名
INSTALLED_FILES_NAME = "INSTALLED_FILES"

# POSIX 操作系统的目录
# （这里缺少具体的代码，无法添加注释）
# 这些是在"install"或"py2exe"命令之后创建的
# 这些目录是相对于安装或分发目录的
data_dir = os.path.join('share', APP_NAME)  # 创建数据目录路径
pixmaps_dir = os.path.join(data_dir, 'pixmaps')  # 创建图标目录路径
locale_dir = os.path.join(data_dir, 'locale')  # 创建本地化目录路径
config_dir = os.path.join(data_dir, 'config')  # 创建配置文件目录路径
docs_dir = os.path.join(data_dir, 'docs')  # 创建文档目录路径
misc_dir = os.path.join(data_dir, 'misc')  # 创建杂项目录路径

# 安装.desktop文件的位置
desktop_dir = os.path.join('share', 'applications')  # 创建.desktop文件目录路径

# mo_find函数用于查找.mo文件
def mo_find(result, dirname, fnames):
    files = []
    for f in fnames:
        p = os.path.join(dirname, f)
        if os.path.isfile(p) and f.endswith(".mo"):  # 如果是文件且以.mo结尾
            files.append(p)

    if files:
        result.append((dirname, files))  # 将.mo文件路径添加到结果列表中

###############################################################################
# 安装变量

data_files = [
        (pixmaps_dir, glob(os.path.join(pixmaps_dir, '*.gif')) +
            glob(os.path.join(pixmaps_dir, '*.png'))),  # 图标文件路径及格式

        (os.path.join(pixmaps_dir, "radialnet"),
            glob(os.path.join(pixmaps_dir, "radialnet", '*.png'))),  # radialnet图标文件路径及格式

        (config_dir, [os.path.join(config_dir, APP_NAME + '.conf'),
            os.path.join(config_dir, 'scan_profile.usp'),
            os.path.join(config_dir, APP_NAME + '_version')]),  # 配置文件路径及格式

        (misc_dir, glob(os.path.join(misc_dir, '*.xml'))),  # 杂项文件路径及格式

        (docs_dir, [os.path.join(docs_dir, 'help.html')])  # 文档文件路径及格式
        ]

# 将i18n文件添加到data_files列表中
os.walk(locale_dir, mo_find, data_files)

# path_startswith和path_strip_prefix用于处理安装根目录(--root选项，也称为DESTDIR)。
def path_startswith(path, prefix):
    """如果路径以前缀开头，则返回True。它比str.startswith更智能，因为它规范化路径以删除多个目录分隔符和向上向下遍历。"""
    path = os.path.normpath(path)
    prefix = os.path.normpath(prefix)
    return path.startswith(prefix)  # 返回路径是否以前缀开头的布尔值
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
    # 检查路径是否为绝对路径
    absolute = os.path.isabs(path)
    # 规范化路径格式
    path = os.path.normpath(path)
    # 规范化前缀路径格式
    prefix = os.path.normpath(prefix)
    # 如果路径以前缀开头且前缀不是根目录，则去除前缀
    if path.startswith(prefix) and prefix != os.sep:
        path = path[len(prefix):]
    # 绝对路径必须保持绝对，相对路径必须保持相对
    assert os.path.isabs(path) == absolute
    return path

###############################################################################
# Distutils subclasses


class my_install(install):
    def finalize_options(self):
        # Ubuntu的python2.6-2.6.4-0ubuntu3软件包在sys.prefix为"/usr/local"（我们的默认值）时，
        # 会在install.finalize_options中更改sys.prefix。因为我们稍后需要未更改的值，所以在这里记住它。
        self.saved_prefix = self.prefix
        install.finalize_options(self)

    def run(self):
        install.run(self)

        # 设置权限
        self.set_perms()
        # 设置模块路径
        self.set_modules_path()
        # 修复路径
        self.fix_paths()
        # 创建卸载程序
        self.create_uninstaller()
        # 写入已安装文件
        self.write_installed_files()
    # 返回一个列表，包含已安装的文件和目录，如果有安装根目录，则以其为前缀。
    # 已安装目录的列表不是来自于 distutils，因此可能不完整。
    def get_installed_files(self):
        installed_files = self.get_outputs()  # 获取已安装的文件列表
        for package in self.distribution.packages:  # 遍历安装的包
            dir = package.replace(".", "/")  # 将包名中的点替换为斜杠
            installed_files.append(os.path.join(self.install_lib, dir))  # 将包的安装路径添加到已安装文件列表中
        installed_files.append(os.path.join(self.install_data, data_dir))  # 将数据目录的安装路径添加到已安装文件列表中
        for dirpath, dirs, files in os.walk(os.path.join(self.install_data, data_dir)):  # 递归包含数据目录中的所有目录
            for dir in dirs:
                installed_files.append(os.path.join(dirpath, dir))  # 将数据目录中的目录添加到已安装文件列表中
        installed_files.append(os.path.join(self.install_scripts, "uninstall_" + APP_NAME))  # 将卸载程序的安装路径添加到已安装文件列表中
        return installed_files  # 返回已安装文件列表
    
    # 创建卸载程序
    def create_uninstaller(self):
        uninstaller_filename = os.path.join(self.install_scripts, "uninstall_" + APP_NAME)  # 设置卸载程序的文件名
    
        uninstaller = """\  # 创建卸载程序的字符串
#!/usr/bin/env python3
# 设置脚本的解释器为 Python3

import errno, os, os.path, sys
# 导入需要使用的模块

print('Uninstall %(name)s %(version)s')
# 打印卸载的软件名称和版本号

answer = raw_input('Are you sure that you want to uninstall '
    '%(name)s %(version)s? (yes/no) ')
# 提示用户确认是否卸载软件

if answer != 'yes' and answer != 'y':
    print('Not uninstalling.')
    sys.exit(0)
# 如果用户输入不是 yes 或者 y，则不卸载软件

""" % {'name': APP_DISPLAY_NAME, 'version': VERSION}
# 格式化字符串，替换卸载软件的名称和版本号

        installed_files = []
        for output in self.get_installed_files():
            if self.root is not None:
                # 如果有 root（DESTDIR），则需要从路径中去掉它，以便卸载程序在目标主机上运行
                # 路径操作比较棘手，但因为卸载程序只需要在 Unix 上运行，所以相对容易
                if not path_startswith(output, self.root):
                    # 这种情况不应该发生（所有内容都安装在 root 内），但如果发生了，为了安全起见，不要删除任何东西
                    uninstaller += ("print('%s was not installed inside "
                        "the root %s; skipping.')\n" % (output, self.root))
                    continue
                output = path_strip_prefix(output, self.root)
                assert os.path.isabs(output)
            installed_files.append(output)
# 遍历已安装的文件，处理路径和添加到已安装文件列表中

        uninstaller += """\
INSTALLED_FILES = (
"""
        for file in installed_files:
            uninstaller += "    %s,\n" % repr(file)
        uninstaller += """\
)
# 将已安装的文件列表添加到卸载程序中

# Split the list into lists of files and directories.
files = []
dirs = []
for path in INSTALLED_FILES:
    if os.path.isfile(path) or os.path.islink(path):
        files.append(path)
    elif os.path.isdir(path):
        dirs.append(path)
# 将已安装的文件列表分成文件和目录两个列表

# Delete the files.
for file in files:
    print("Removing '%s'." % file)
    try:
        os.remove(file)
    except OSError as e:
        print('  Error: %s.' % str(e), file=sys.stderr)
# 删除文件

# Delete the directories. First reverse-sort the normalized paths by
# 删除目录。首先按照规范化后的路径进行反向排序
# 根据目录列表中的路径规范化，确保子目录在父目录之前被删除
dirs = [os.path.normpath(dir) for dir in dirs]
# 根据路径长度对目录列表进行排序，以便先删除子目录
dirs.sort(key = len, reverse = True)
# 遍历目录列表中的每个目录
for dir in dirs:
    try:
        # 尝试删除目录，并打印提示信息
        print("Removing the directory '%s'." % dir)
        os.rmdir(dir)
    except OSError as e:
        # 如果目录不为空，则打印提示信息
        if e.errno == errno.ENOTEMPTY:
            print("Directory '%s' not empty; not removing." % dir)
        # 如果出现其他错误，则打印错误信息
        else:
            print(str(e), file=sys.stderr)

# 打开卸载程序文件，写入卸载程序内容，然后关闭文件
uninstaller_file = open(uninstaller_filename, 'w')
uninstaller_file.write(uninstaller)
uninstaller_file.close()

# 设置卸载程序的执行权限
mode = ((os.stat(uninstaller_filename)[ST_MODE]) | 0o555) & 0o7777
os.chmod(uninstaller_filename, mode)

# 设置模块路径
def set_modules_path(self):
    # 获取应用程序文件的路径
    app_file_name = os.path.join(self.install_scripts, APP_NAME)
    # 查找模块安装的路径，如果存在根目录，则需要去除根目录的路径
    modules_dir = self.install_lib
    if self.root is not None:
        modules_dir = path_strip_prefix(modules_dir, self.root)

    # 打开应用程序文件，读取内容
    app_file = open(app_file_name, "r")
    lines = app_file.readlines()
    app_file.close()

    # 遍历文件内容，查找并替换模块安装路径
    for i in range(len(lines)):
        if re.match(r'^INSTALL_LIB =', lines[i]):
            lines[i] = "INSTALL_LIB = %s\n" % repr(modules_dir)
            break
    else:
        # 如果未找到替换位置，则抛出数值错误
        raise ValueError(
                "INSTALL_LIB replacement not found in %s" % app_file_name)

    # 重新打开应用程序文件，写入修改后的内容，然后关闭文件
    app_file = open(app_file_name, "w")
    app_file.writelines(lines)
    app_file.close()
    # 设置文件权限
    def set_perms(self):
        # 编译正则表达式，匹配文件名中包含"bin"或".sh"的文件
        re_bin = re.compile("(bin|\.sh)")
        # 遍历已安装的文件列表
        for output in self.get_installed_files():
            # 如果文件名中包含"bin"或".sh"，则跳过
            if re_bin.findall(output):
                continue

            # 如果是目录，设置读写执行权限给所有者，读权限给组和其他用户
            if os.path.isdir(output):
                os.chmod(output, S_IRWXU |
                                 S_IRGRP |
                                 S_IXGRP |
                                 S_IROTH |
                                 S_IXOTH)
            # 如果是文件，设置读写权限给所有者，读权限给组和其他用户
            else:
                os.chmod(output, S_IRUSR |
                                 S_IWUSR |
                                 S_IRGRP |
                                 S_IROTH)

    # 写入已安装文件列表
    def write_installed_files(self):
        """Write a list of installed files for use by the uninstall command.
        This is similar to what happens with the --record option except that it
        doesn't strip off the installation root, if any. File names containing
        newline characters are not handled."""
        # 如果已安装文件名与记录文件名相同，则发出警告
        if INSTALLED_FILES_NAME == self.record:
            distutils.log.warn("warning: installation record is overwriting "
                "--record file '%s'." % self.record)
        # 打开已安装文件列表文件，写入已安装文件列表
        with open(INSTALLED_FILES_NAME, "w") as f:
            for output in self.get_installed_files():
                # 断言文件名中不包含换行符
                assert "\n" not in output
                # 将文件名写入文件
                print(output, file=f)
# 定义一个名为 my_uninstall 的类，继承自 Command 类
class my_uninstall(Command):
    """A distutils command that performs uninstallation. It reads the list of
    installed files written by the install command."""
    # 定义类属性 command_name，表示命令名称为 "uninstall"
    command_name = "uninstall"
    # 定义类属性 description，描述为 "uninstall installed files recorded in '%s'"，其中 %s 为 INSTALLED_FILES_NAME 的值
    description = "uninstall installed files recorded in '%s'" % (
            INSTALLED_FILES_NAME)
    # 定义类属性 user_options，表示用户选项为空列表
    user_options = []

    # 定义初始化选项的方法
    def initialize_options(self):
        pass

    # 定义最终选项的方法
    def finalize_options(self):
        pass
    # 定义一个方法，用于运行删除操作
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
        # 将文件和目录分开
        files = []
        dirs = []
        # 遍历已安装文件列表，将文件和目录分别添加到对应的列表中
        for path in installed_files:
            if os.path.isfile(path) or os.path.islink(path):
                files.append(path)
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
                # 如果目录不为空，则输出信息
                if e.errno == errno.ENOTEMPTY:
                    log.info("Directory '%s' not empty; not removing." % dir)
                else:
                    log.error(str(e))
# 根据不同的操作方式调用设置，这些参数在所有操作中都是通用的
COMMON_SETUP_ARGS = {
    'name': APP_NAME,  # 应用程序名称
    'license': 'Nmap License (https://nmap.org/book/man-legal.html)',  # 许可证信息
    'url': APP_WEB_SITE,  # 应用程序网站
    'download_url': APP_DOWNLOAD_SITE,  # 应用程序下载地址
    'author': 'Nmap Project',  # 作者
    'maintainer': 'Nmap Project',  # 维护者
    'description': "%s frontend and results viewer" % NMAP_DISPLAY_NAME,  # 描述
    'long_description': "%s is an %s frontend that is really useful"
        "for advanced users and easy to be used by newbies." % (
            APP_DISPLAY_NAME, NMAP_DISPLAY_NAME),  # 长描述
    'version': VERSION,  # 版本号
    'scripts': [APP_NAME],  # 脚本文件
    'packages': ['zenmapCore', 'zenmapGUI', 'zenmapGUI.higwidgets',
                 'radialnet', 'radialnet.bestwidgets', 'radialnet.core',
                 'radialnet.gui', 'radialnet.util'],  # 包含的包
    'data_files': data_files,  # 数据文件
}

# 所有设置的参数都被收集在 setup_args 中
setup_args = {}
setup_args.update(COMMON_SETUP_ARGS)

if 'py2exe' in sys.argv:
    # Windows 和 py2exe 特定的参数
    import py2exe
    # 定义 Windows 平台的设置参数字典
    WINDOWS_SETUP_ARGS = {
        'zipfile': 'py2exe/library.zip',  # 指定 ZIP 文件路径
        'name': APP_NAME,  # 设置应用程序名称
        'windows': [{  # 配置 Windows 窗口程序
            "script": APP_NAME,  # 指定脚本文件名
            "icon_resources": [(1, "install_scripts/windows/nmap-eye.ico")]  # 设置图标资源
            }],
        # 在 Windows 上，在 Zenmap 的 setup.py 中构建 Ndiff，以便两个 Python 程序共享一个公共运行时
        'py_modules': ["ndiff"],  # 指定要打包的 Python 模块
        # 重写包搜索路径，以便找到 Ndiff
        'package_dir': {
            'zenmapCore': 'zenmapCore',
            'zenmapGUI': 'zenmapGUI',
            'radialnet': 'radialnet',
            '': '../ndiff'  # 指定 Ndiff 的路径
            },
        'console': [{  # 配置控制台程序
            "script": "../ndiff/scripts/ndiff",  # 指定脚本文件路径
            "description": "Nmap scan comparison tool"  # 设置程序描述
            }],
        'options': {"py2exe": {  # 配置 py2exe 选项
            "compressed": 1,  # 启用压缩
            "optimize": 2,  # 优化级别
            "packages": ["encodings"],  # 指定要包含的包
            "includes": ["pango", "atk", "gobject", "gio", "pickle", "bz2",
                "encodings", "encodings.*", "cairo", "pangocairo"],  # 指定要包含的模块
            "dll_excludes": ["USP10.dll", "NSI.dll", "MSIMG32.dll",
                "DNSAPI.dll"],  # 排除的 DLL 文件
            "custom_boot_script": "install_scripts/windows/boot_script.py",  # 指定自定义启动脚本
            }
        }
    }

    # 更新设置参数字典
    setup_args.update(WINDOWS_SETUP_ARGS)
# 如果在命令行参数中包含'py2app'，则执行以下代码块
elif 'py2app' in sys.argv:
    # 为 Mac OS X 和 py2app 准备参数
    import py2app
    import shutil

    # py2app 需要一个 ".py" 后缀的文件
    extended_app_name = APP_NAME + ".py"
    # 复制文件，将原文件名改为带有 ".py" 后缀的文件名
    shutil.copyfile(APP_NAME, extended_app_name)

    # 设置 Mac OS X 的配置参数
    MACOSX_SETUP_ARGS = {
        'app': [extended_app_name],
        'options': {"py2app": {
            "packages": ["gio", "gobject", "gtk", "cairo"],
            "includes": ["atk", "pango", "pangocairo"],
            "argv_emulation": True,
            "compressed": True,
            "plist": "install_scripts/macosx/Info.plist",
            "iconfile": "install_scripts/macosx/zenmap.icns"
        }}
    }

    # 更新设置参数
    setup_args.update(MACOSX_SETUP_ARGS)
# 如果在命令行参数中包含'vanilla'，则执行以下代码块
elif 'vanilla' in sys.argv:
    # 不创建卸载程序，不修复路径。用于在 OS X 上打包
    sys.argv.remove('vanilla')
# 如果以上条件都不满足，则执行以下代码块
else:
    # 默认参数
    DEFAULT_SETUP_ARGS = {
        'cmdclass': {'install': my_install, 'uninstall': my_uninstall},
    }
    # 更新设置参数
    setup_args.update(DEFAULT_SETUP_ARGS)

    # 数据文件
    data_files = [
        (desktop_dir, glob('install_scripts/unix/*.desktop')),
        (data_dir, ['install_scripts/unix/su-to-zenmap.sh'])
    ]
    # 将数据文件添加到设置参数中
    setup_args["data_files"].extend(data_files)

# 执行设置函数，传入设置参数
setup(**setup_args)
```