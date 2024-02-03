# `nmap\zenmap\zenmapCore\I18N.py`

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
import locale
import os

# 从 zenmapCore.Name 模块中导入 APP_NAME 变量
from zenmapCore.Name import APP_NAME

# 定义函数，获取基于系统配置的区域设置列表
def get_locales():
    locales = []  # 初始化区域设置列表

    # 检查环境变量，以确定要使用的区域设置
    for envar in ("LANGUAGE", "LC_ALL", "LC_MESSAGES", "LANG"):
        val = os.environ.get(envar)  # 获取环境变量的值
        if val:
            locales = val.split(":")  # 将值按冒号分割成列表
            break

    # 尝试获取系统默认的区域设置
    try:
        loc, enc = locale.getdefaultlocale()  # 获取系统默认的区域设置和编码
        if loc is not None:
            locales.append(loc)  # 将系统默认的区域设置添加到列表中
    # 捕获值错误异常
    except ValueError:
        # 当使用 locale.getdefaultlocale 获取默认语言环境时可能会出现值错误异常
        # 在某些语言环境名称下已经观察到这种情况，比如 en_NG
        # 参考链接：http://bugs.python.org/issue6895
        # 什么也不做，直接跳过异常
        pass
    # 返回语言环境列表
    return locales
# 安装 gettext 模块，用于国际化支持
def install_gettext(locale_dir):
    # 尝试设置本地化环境为系统默认
    try:
        locale.setlocale(locale.LC_ALL, '')
    except locale.Error:
        # 如果出现错误，说明 LANG 环境变量设置不正确，比如 LANG=nothing 或 LANG=en_US/utf8 或 LANG=us-ascii
        # 继续不使用国际化
        pass

    # 尝试导入 gettext 模块
    try:
        import gettext
    except ImportError:
        # 如果导入失败，不做任何操作
        pass
    else:
        # 如果导入成功，创建 gettext 对象并安装
        t = gettext.translation(
                APP_NAME, locale_dir, languages=get_locales(), fallback=True)
        t.install()

# 安装一个虚拟的 _ 函数，这样其他模块即使没有安装 gettext 版本也可以安全地使用它
import builtins
builtins.__dict__["_"] = lambda s: s
```