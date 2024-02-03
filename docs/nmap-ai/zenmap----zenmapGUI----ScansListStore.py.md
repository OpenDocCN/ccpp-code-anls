# `nmap\zenmap\zenmapGUI\ScansListStore.py`

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
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 创建一个名为 ScansListStoreEntry 的类，用于表示运行和完成的扫描
class ScansListStoreEntry(object):
    """This class is an abstraction for running and completed scans, which are
    otherwise represented by very different classes."""

    # 扫描可能的状态
    UNINITIALIZED, RUNNING, FINISHED, FAILED, CANCELED = list(range(5))

    # 初始化方法，设置初始状态和属性
    def __init__(self):
        self.state = self.UNINITIALIZED
        self.command = None
        self.parsed = None

    # 设置扫描状态为运行中
    def set_running(self, command=None):
        self.state = self.RUNNING
        self.command = command

    # 设置扫描状态为完成
    def set_finished(self, parsed=None):
        self.state = self.FINISHED
        self.parsed = parsed

    # 设置扫描状态为失败
    def set_failed(self):
        self.state = self.FAILED
    # 设置任务状态为取消
    def set_canceled(self):
        self.state = self.CANCELED
    
    # 获取命令字符串
    def get_command_string(self):
        # 如果已解析，则返回解析后的 nmap 命令字符串
        if self.parsed is not None:
            return self.parsed.get_nmap_command()
        # 如果未解析但存在命令，则返回命令字符串
        elif self.command is not None:
            return self.command.command
        # 否则返回空
        else:
            return None
    
    # 判断任务是否正在运行
    running = property(lambda self: self.state == self.RUNNING)
    # 判断任务是否已完成
    finished = property(lambda self: self.state == self.FINISHED)
    # 判断任务是否失败
    failed = property(lambda self: self.state == self.FAILED)
    # 判断任务是否被取消
    canceled = property(lambda self: self.state == self.CANCELED)
class ScansListStore(Gtk.ListStore):
    """This is a specialization of a Gtk.ListStore that holds running,
    completed, and failed scans."""
    # 初始化方法，继承自 Gtk.ListStore，用于创建一个包含对象的列表存储
    def __init__(self):
        Gtk.ListStore.__init__(self, object)

    # 添加一个正在运行的 NmapCommand 对象到扫描列表中
    def add_running_scan(self, command):
        entry = ScansListStoreEntry()
        entry.set_running(command)
        return self.append([entry])

    # 完成一个正在运行的扫描，用给定的解析表示替换它
    def finish_running_scan(self, command, parsed):
        i = self._find_running_scan(command)
        if i is not None:
            entry = self.get_value(i, 0)
            entry.set_finished(parsed)
            path = self.get_path(i)
            self.row_changed(path, i)
            return i

    # 将一个正在运行的扫描标记为失败
    def fail_running_scan(self, command):
        i = self._find_running_scan(command)
        if i is not None:
            entry = self.get_value(i, 0)
            entry.set_failed()
            path = self.get_path(i)
            self.row_changed(path, i)
            return i

    # 将一个正在运行的扫描标记为取消
    def cancel_running_scan(self, command):
        i = self._find_running_scan(command)
        if i is not None:
            entry = self.get_value(i, 0)
            entry.set_canceled()
            path = self.get_path(i)
            self.row_changed(path, i)
            return i

    # 添加一个解析后的 NmapParser 对象到扫描列表中
    def add_scan(self, parsed):
        entry = ScansListStoreEntry()
        entry.set_finished(parsed)
        return self.append([entry])
    # 查找运行中的扫描任务，其命令为指定的命令
    def _find_running_scan(self, command):
        # 获取第一个迭代器
        i = self.get_iter_first()
        # 循环遍历迭代器
        while i is not None:
            # 获取当前迭代器指向的扫描任务
            entry = self.get_value(i, 0)
            # 如果扫描任务的命令与指定的命令相同，则返回当前迭代器
            if entry.command is command:
                return i
            # 获取下一个迭代器
            i = self.iter_next(i)
        # 如果没有找到匹配的扫描任务，则返回 None
        return None
```