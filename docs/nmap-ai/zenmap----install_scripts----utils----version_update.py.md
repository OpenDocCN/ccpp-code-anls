# `nmap\zenmap\install_scripts\utils\version_update.py`

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
# 这个程序更新了需要更新的所有版本号。它接受一个命令行参数，即新的版本号。例如：
# python install_scripts/utils/version_update.py X.YY

# 导入所需的模块
import os
import sys
import re
from datetime import datetime

# 定义版本文件的路径
VERSION = os.path.join("share", "zenmap", "config", "zenmap_version")
VERSION_PY = os.path.join("zenmapCore", "Version.py")
NAME_PY = os.path.join("zenmapCore", "Name.py")

# 更新日期的函数
def update_date(base_dir):
    # 定义文件路径
    name_file = os.path.join(base_dir, NAME_PY)
    # 打印正在更新的文件
    print(">>> Updating %s" % name_file)
    # 打开文件，读取内容
    nf = open(name_file, "r")
    ncontent = nf.read()
    nf.close()
    # 使用正则表达式替换文件内容中的版权信息，将年份部分替换为当前年份
    ncontent = re.sub(r'APP_COPYRIGHT *= *"Copyright 2005-....',
            'APP_COPYRIGHT = "Copyright 2005-%d' % (datetime.today().year),
            ncontent)
    # 打开文件以写入修改后的内容
    nf = open(name_file, "w")
    # 将修改后的内容写入文件
    nf.write(ncontent)
    # 关闭文件
    nf.close()
# 更新版本号函数，接受基础目录和版本号作为参数
def update_version(base_dir, version):
    # 打印正在更新的文件路径
    print(">>> Updating %s" % os.path.join(base_dir, VERSION))
    # 打开版本文件，以写入模式
    vf = open(os.path.join(base_dir, VERSION), "w")
    # 将版本号写入文件
    print(version, file=vf)
    # 关闭文件
    vf.close()
    # 打印正在更新的文件路径
    print(">>> Updating %s" % os.path.join(base_dir, VERSION_PY))
    # 打开版本文件，以写入模式
    vf = open(os.path.join(base_dir, VERSION_PY), "w")
    # 将版本号写入文件
    print("VERSION = \"%s\"" % version, file=vf)
    # 关闭文件
    vf.close()

# 如果作为脚本直接运行
if __name__ == "__main__":
    # 如果命令行参数不等于2，打印使用说明并退出
    if len(sys.argv) != 2:
        print("Usage: %s <version>" % sys.argv[0], file=sys.stderr)
        sys.exit(1)

    # 获取版本号参数
    version = sys.argv[1]
    # 打印正在更新版本号到指定版本
    print(">>> Updating version number to \"%s\"" % version)
    # 调用更新版本号函数
    update_version(".", version)
    # 调用更新日期函数
    update_date(".")
```