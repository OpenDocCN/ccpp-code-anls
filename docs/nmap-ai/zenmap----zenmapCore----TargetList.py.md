# `nmap\zenmap\zenmapCore\TargetList.py`

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
# 从os模块中导入access、R_OK、W_OK
# access函数用于检验权限模式，R_OK表示可读，W_OK表示可写
from os import access, R_OK, W_OK
# 从os.path模块中导入dirname函数
# dirname函数用于去掉文件名，返回目录
from os.path import dirname
# 从zenmapCore.Paths模块中导入Path类
# Path类用于处理路径相关操作
from zenmapCore.Paths import Path

# 定义TargetList类
class TargetList(object):
    # 初始化方法，初始化临时列表为空
    def __init__(self):
        self.temp_list = []

        # 尝试获取目标列表文件路径，如果失败则将目标列表文件置为 False
        try:
            self.target_list_file = Path.target_list
        except Exception:
            self.target_list_file = False

        # 检查目标列表文件是否存在并且可读写
        if (self.target_list_file and
                (access(self.target_list_file, R_OK and W_OK) or
                    access(dirname(self.target_list_file), R_OK and W_OK))):
            self.using_file = True

            # 从保存的目标列表文件中恢复目标
            target_file = open(self.target_list_file, "r")
            self.temp_list = [
                    t for t in target_file.read().split(";")
                    if t != "" and t != "\n"]
            target_file.close()
        else:
            self.using_file = False

    # 保存目标列表到文件
    def save(self):
        if self.using_file:
            target_file = open(self.target_list_file, "w")
            target_file.write(";".join(self.temp_list))
            target_file.close()

    # 添加目标到临时列表，并保存到文件
    def add_target(self, target):
        if target in self.temp_list:
            return

        self.temp_list.append(target)
        self.save()

    # 清空临时列表并保存到文件
    def clean_list(self):
        del self.temp_list
        self.temp_list = []
        self.save()

    # 获取目标列表的副本，并倒序排列
    def get_target_list(self):
        t = self.temp_list[:]
        t.reverse()
        return t
# 创建一个名为target_list的TargetList对象
target_list = TargetList()

# 如果当前脚本被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个名为t的TargetList对象
    t = TargetList()
    # 打印获取空列表的消息，并调用TargetList对象的get_target_list方法
    print(">>> Getting empty list:", t.get_target_list())
    # 打印添加目标127.0.0.1的消息，并调用TargetList对象的add_target方法
    print(">>> Adding target 127.0.0.1:", t.add_target("127.0.0.3"))
    # 打印获取目标列表的消息，并调用TargetList对象的get_target_list方法
    print(">>> Getting target list:", t.get_target_list())
    # 删除t对象
    del t
```