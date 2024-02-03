# `nmap\zenmap\zenmapGUI\Icons.py`

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
# 导入 gi 模块
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GLib, GdkPixbuf

# 导入 re 和 os.path 模块
import re
import os.path

# 从 zenmapCore.Paths 模块导入 Path
from zenmapCore.Paths import Path
# 从 zenmapCore.UmitLogging 模块导入 log
from zenmapCore.UmitLogging import log

# 定义图标名称的元组
icon_names = (
    # 操作系统
    'default',
    'freebsd',
    'irix',
    'linux',
    'macosx',
    'openbsd',
    'redhat',
    'solaris',
    'ubuntu',
    'unknown',
    'win',
    # 漏洞级别
    'vl_1',
    'vl_2',
    'vl_3',
    'vl_4',
    'vl_5')

# 获取图片路径
pixmap_path = Path.pixmaps_dir
if pixmap_path:
    # 这是一个生成器，按顺序返回应尝试的图片文件名
    def get_pixmap_file_names(icon_name, size):
        yield '%s_%s.png' % (icon_name, size)

    # 创建图标工厂对象
    iconfactory = Gtk.IconFactory()
    # 遍历图标名称列表
    for icon_name in icon_names:
        # 遍历图标类型和尺寸的组合
        for type, size in (('icon', '32'), ('logo', '75')):
            # 生成图标键名
            key = '%s_%s' % (icon_name, type)
            # 寻找可用的图像文件
            for file_name in get_pixmap_file_names(icon_name, size):
                # 拼接文件路径
                file_path = os.path.join(pixmap_path, file_name)
                try:
                    # 从文件路径创建 GdkPixbuf.Pixbuf 对象
                    pixbuf = GdkPixbuf.Pixbuf.new_from_file(file_path)
                    # 找到可用图像文件则跳出循环
                    break
                except GLib.GError:
                    # 如果出现错误则继续尝试
                    pass
            else:
                # 如果找不到图标文件，则记录警告信息并继续下一个图标
                log.warn('Could not find the icon for %s at '
                        'any of (%s) in %s' % (
                            icon_name,
                            ', '.join(get_pixmap_file_names(icon_name, size)),
                            pixmap_path))
                continue
            # 创建 Gtk.IconSet 对象
            iconset = Gtk.IconSet(pixbuf=pixbuf)
            # 将图标键名和图标集添加到图标工厂
            iconfactory.add(key, iconset)
            # 记录调试信息
            log.debug('Register %s icon name for file %s' % (key, file_path))
    # 添加默认图标
    iconfactory.add_default()
# 获取操作系统图标
def get_os_icon(host):
    # 获取最佳的操作系统匹配
    osmatch = host.get_best_osmatch()
    # 如果有最佳匹配并且有操作系统类别
    if osmatch and osmatch['osclasses']:
        # 获取第一个操作系统类别
        osclass = osmatch['osclasses'][0]
    else:
        osclass = None

    # 如果有操作系统类别和最佳匹配
    if osclass and osmatch:
        # 返回操作系统图标
        return get_os(osclass['osfamily'], osmatch['name'], 'icon')
    else:
        # 返回空图标
        return get_os(None, None, 'icon')


# 获取操作系统 logo
def get_os_logo(host):
    # 获取最佳的操作系统匹配
    osmatch = host.get_best_osmatch()
    # 如果有最佳匹配并且有操作系统类别
    if osmatch and osmatch['osclasses']:
        # 获取第一个操作系统类别
        osclass = osmatch['osclasses'][0]
    else:
        osclass = None

    # 如果有操作系统类别和最佳匹配
    if osclass and osmatch:
        # 返回操作系统 logo
        return get_os(osclass['osfamily'], osmatch['name'], 'logo')
    else:
        # 返回空 logo
        return get_os(None, None, 'logo')


# 获取操作系统
def get_os(osfamily, osmatch, type):
    # 如果存在操作系统家族
    if osfamily:
        # 如果操作系统家族是 Linux
        if osfamily == 'Linux':
            # 如果操作系统匹配包含 ubuntu 字符串
            if re.findall("ubuntu", osmatch.lower()):
                # 返回 Ubuntu 图标
                return 'ubuntu_%s' % type
            # 如果操作系统匹配包含 red hat 字符串
            elif re.findall("red hat", osmatch.lower()):
                # 返回 RedHat 图标
                return 'redhat_%s' % type
            else:
                # 返回通用 Linux 图标
                return 'linux_%s' % type
        # 如果操作系统家族是 Windows
        elif osfamily == 'Windows':
            # 返回 Windows 图标
            return 'win_%s' % type
        # 如果操作系统家族是 OpenBSD
        elif osfamily == 'OpenBSD':
            # 返回 OpenBSD 图标
            return 'openbsd_%s' % type
        # 如果操作系统家族是 FreeBSD
        elif osfamily == 'FreeBSD':
            # 返回 FreeBSD 图标
            return 'freebsd_%s' % type
        # 如果操作系统家族是 NetBSD
        elif osfamily == 'NetBSD':
            # 返回 NetBSD 图标
            return 'default_%s' % type
        # 如果操作系统家族是 Solaris
        elif osfamily == 'Solaris':
            # 返回 Solaris 图标
            return 'solaris_%s' % type
        # 如果操作系统家族是 OpenSolaris
        elif osfamily == 'OpenSolaris':
            # 返回 OpenSolaris 图标
            return 'solaris_%s' % type
        # 如果操作系统家族是 IRIX
        elif osfamily == 'IRIX':
            # 返回 Irix 图标
            return 'irix_%s' % type
        # 如果操作系统家族是 Mac OS X
        elif osfamily == 'Mac OS X':
            # 返回 Mac OS X 图标
            return 'macosx_%s' % type
        # 如果操作系统家族是 Mac OS
        elif osfamily == 'Mac OS':
            # 返回 Mac OS 图标
            return 'macosx_%s' % type
        else:
            # 返回默认操作系统图标
            return 'default_%s' % type
    else:
        # 返回未知操作系统图标
        return 'unknown_%s' % type
# 根据开放端口数量获取对应的漏洞图标
def get_vulnerability_logo(open_ports):
    # 将开放端口数量转换为整数
    open_ports = int(open_ports)
    # 如果开放端口数量小于3，返回对应的漏洞图标
    if open_ports < 3:
        return 'vl_1_logo'
    # 如果开放端口数量小于5，返回对应的漏洞图标
    elif open_ports < 5:
        return 'vl_2_logo'
    # 如果开放端口数量小于7，返回对应的漏洞图标
    elif open_ports < 7:
        return 'vl_3_logo'
    # 如果开放端口数量小于9，返回对应的漏洞图标
    elif open_ports < 9:
        return 'vl_4_logo'
    # 如果开放端口数量大于等于9，返回对应的漏洞图标
    else:
        return 'vl_5_logo'
```