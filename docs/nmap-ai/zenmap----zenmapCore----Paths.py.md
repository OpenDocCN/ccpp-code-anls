# `nmap\zenmap\zenmapCore\Paths.py`

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
# 导入所需的模块
from os.path import join, dirname
import errno
import os
import os.path
import sys
import shutil
from zenmapCore.BasePaths import base_paths
from zenmapCore.Name import APP_NAME

# 查找存储数据文件（接口定义 XML、像素图等）的前缀
# 这取决于我们是否在可执行包中运行以及包的类型，我们使用 sys.frozen 属性进行检查
# 参考：http://mail.python.org/pipermail/pythonmac-sig/2004-November/012121.html
def get_prefix():
    # 导入 getsitepackages 函数
    from site import getsitepackages
    # 获取 sys.frozen 属性
    frozen = getattr(sys, "frozen", None)
    # 如果是 "macosx_app" 或者 sys.executable 中包含 "Zenmap.app"，则返回 .app 包的资源路径
    if frozen == "macosx_app" or "Zenmap.app" in sys.executable:
        return os.path.join(dirname(sys.executable), "..", "Resources")
    # 如果程序被冻结，则假设为 py2exe 可执行文件，返回执行文件的目录名
    elif frozen is not None:
        return dirname(sys.executable)
    # 如果文件路径以 getsitepackages() 返回的任何路径开头，则假设已安装在 site-packages 中，使用配置的前缀
    elif any(__file__.startswith(pdir) for pdir in getsitepackages()):
        return sys.prefix
    # 否则，正常脚本执行，查找当前目录以允许从分发中运行
    else:
        return os.path.abspath(os.path.dirname(sys.argv[0]))
# 获取安装路径的前缀
prefix = get_prefix()

# 这些行将被安装程序覆盖，以硬编码安装位置
CONFIG_DIR = join(prefix, "share", APP_NAME, "config")
LOCALE_DIR = join(prefix, "share", APP_NAME, "locale")
MISC_DIR = join(prefix, "share", APP_NAME, "misc")
PIXMAPS_DIR = join(prefix, "share", "zenmap", "pixmaps")
DOCS_DIR = join(prefix, "share", APP_NAME, "docs")
NMAPDATADIR = join(prefix, "..")

# 返回额外的可执行文件搜索路径列表，方便在默认路径不足的平台上使用
def get_extra_executable_search_paths():
    if sys.platform == 'darwin':
        return ["/usr/local/bin"]
    elif sys.platform == 'win32':
        return [dirname(sys.executable)]
    return []

# 路径类
class Paths(object):
    """Paths
    """
    # 硬编码的路径列表
    hardcoded = ["config_dir",
                 "locale_dir",
                 "pixmaps_dir",
                 "misc_dir",
                 "docs_dir"]

    # 配置文件列表
    config_files_list = ["config_file",
                         "scan_profile",
                         "version"]

    # 空配置文件列表
    empty_config_files_list = ["target_list",
                               "recent_scans",
                               "db"]

    # 杂项文件列表
    misc_files_list = ["options",
                       "profile_editor"]

    def __init__(self):
        # 初始化路径
        self.config_dir = CONFIG_DIR
        self.locale_dir = LOCALE_DIR
        self.pixmaps_dir = PIXMAPS_DIR
        self.misc_dir = MISC_DIR
        self.docs_dir = DOCS_DIR
        self.nmap_dir = NMAPDATADIR
        self._delayed_incomplete = True

    # 延迟初始化这些路径，以便 zenmapCore.I18N.install_gettext 可以在需要它的模块被导入之前安装 _()
    # zenmapCore.I18N.install_gettext 可以在需要它的模块被导入之前安装 _()
    # 延迟初始化函数，用于延迟初始化对象属性
    def _delayed_init(self):
        # 如果延迟初始化未完成
        if self._delayed_incomplete:
            # 从zenmapCore.UmitOptionParser模块导入option_parser对象
            from zenmapCore.UmitOptionParser import option_parser
            # 获取用户配置目录
            self.user_config_dir = option_parser.get_confdir()
            # 拼接用户配置文件路径
            self.user_config_file = os.path.join(
                    self.user_config_dir, base_paths['user_config_file'])
            # 标记延迟初始化已完成
            self._delayed_incomplete = False

    # 获取属性的特殊方法
    def __getattr__(self, name):
        # 如果属性名在硬编码列表中
        if name in self.hardcoded:
            # 返回属性值
            return self.__dict__[name]

        # 执行延迟初始化
        self._delayed_init()
        # 如果属性名在配置文件列表中
        if name in self.config_files_list:
            # 返回存在时的路径
            return return_if_exists(
                    join(self.user_config_dir, base_paths[name]))

        # 如果属性名在空配置文件列表中
        if name in self.empty_config_files_list:
            # 返回存在时的路径，且创建空文件
            return return_if_exists(
                    join(self.user_config_dir, base_paths[name]), True)

        # 如果属性名在杂项文件列表中
        if name in self.misc_files_list:
            # 返回存在时的路径
            return return_if_exists(join(self.misc_dir, base_paths[name]))

        try:
            # 尝试返回属性值
            return self.__dict__[name]
        except Exception:
            # 抛出属性名错误
            raise NameError(name)

    # 设置属性的特殊方法
    def __setattr__(self, name, value):
        # 设置属性值
        self.__dict__[name] = value
# 使用 os.makedirs 创建一个目录，如果目录已经存在则不会引发错误
def create_dir(path):
    try:
        os.makedirs(path)
    except OSError as e:
        if e.errno != errno.EEXIST:
            raise

# 创建用户配置目录，首先创建目录（如果需要），然后从给定的模板目录中复制所有文件，跳过已经存在的文件
def create_user_config_dir(user_dir, template_dir):
    from zenmapCore.UmitLogging import log
    log.debug(">>> Create user dir at %s" % user_dir)
    create_dir(user_dir)

    for filename in os.listdir(template_dir):
        template_filename = os.path.join(template_dir, filename)
        user_filename = os.path.join(user_dir, filename)
        # 只复制普通文件
        if not os.path.isfile(template_filename):
            continue
        # 不覆盖已存在的文件
        if os.path.exists(user_filename):
            log.debug(">>> %s already exists." % user_filename)
            continue
        shutil.copyfile(template_filename, user_filename)
        log.debug(">>> Copy %s to %s." % (template_filename, user_filename))

# 如果路径存在则返回路径，如果 create 参数为 True，则创建文件并返回路径
def return_if_exists(path, create=False):
    path = os.path.abspath(path)
    if os.path.exists(path):
        return path
    elif create:
        f = open(path, "w")
        f.close()
        return path
    raise Exception("File '%s' does not exist or could not be found!" % path)

# 单例模式
Path = Paths()

if __name__ == '__main__':
    print(">>> SAVED DIRECTORIES:")
    print(">>> LOCALE DIR:", Path.locale_dir)
    print(">>> PIXMAPS DIR:", Path.pixmaps_dir)
    print(">>> CONFIG DIR:", Path.config_dir)
    print()
    print(">>> FILES:")
    print(">>> USER CONFIG FILE:", Path.user_config_file)
    print(">>> CONFIG FILE:", Path.user_config_file)
    print(">>> TARGET_LIST:", Path.target_list)
    print(">>> PROFILE_EDITOR:", Path.profile_editor)
    # 打印出扫描配置的路径
    print(">>> SCAN_PROFILE:", Path.scan_profile)
    # 打印出最近扫描的路径
    print(">>> RECENT_SCANS:", Path.recent_scans)
    # 打印出选项的路径
    print(">>> OPTIONS:", Path.options)
    # 打印空行
    print()
    # 打印出数据库的路径
    print(">>> DB:", Path.db)
    # 打印出版本的路径
    print(">>> VERSION:", Path.version)
```