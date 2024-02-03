# `nmap\zenmap\zenmapCore\RecentScans.py`

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
# 从os模块中导入access、R_OK、W_OK
# access函数用于检验权限模式，R_OK表示可读，W_OK表示可写
from os import access, R_OK, W_OK
# 从os.path模块中导入dirname函数
from os.path import dirname
# 从zenmapCore.Paths模块中导入Path类
from zenmapCore.Paths import Path

# 创建RecentScans类
class RecentScans(object):
    # 初始化方法，创建一个空的临时列表
    def __init__(self):
        self.temp_list = []
    
        # 尝试获取最近扫描文件的路径，如果失败则将最近扫描文件设置为 False
        try:
            self.recent_scans_file = Path.recent_scans
        except Exception:
            self.recent_scans_file = False
    
        # 如果最近扫描文件存在并且具有读写权限，则将 using_file 设置为 True
        if (self.recent_scans_file and
                (access(self.recent_scans_file, R_OK and W_OK) or
                    access(dirname(self.recent_scans_file), R_OK and W_OK))):
            self.using_file = True
    
            # 从保存的目标中恢复
            recent_file = open(self.recent_scans_file, "r")
            # 从文件中读取内容并根据分号分割，去除空字符串和换行符，存入临时列表
            self.temp_list = [
                    t for t in recent_file.read().split(";")
                    if t != "" and t != "\n"]
            recent_file.close()
        else:
            self.using_file = False
    
    # 保存方法，如果使用文件则将临时列表内容写入最近扫描文件
    def save(self):
        if self.using_file:
            recent_file = open(self.recent_scans_file, "w")
            recent_file.write(";".join(self.temp_list))
            recent_file.close()
    
    # 添加最近扫描方法，如果最近扫描不在临时列表中则添加，并保存
    def add_recent_scan(self, recent_scan):
        if recent_scan in self.temp_list:
            return
    
        self.temp_list.append(recent_scan)
        self.save()
    
    # 清空列表方法，删除临时列表并重新创建一个空的列表，然后保存
    def clean_list(self):
        del self.temp_list
        self.temp_list = []
        self.save()
    
    # 获取最近扫描列表方法，复制临时列表并进行倒序，返回倒序后的列表
    def get_recent_scans_list(self):
        t = self.temp_list[:]
        t.reverse()
        return t
# 创建一个名为recent_scans的RecentScans对象实例
recent_scans = RecentScans()

# 如果当前脚本作为主程序执行
if __name__ == "__main__":
    # 创建一个名为r的RecentScans对象实例
    r = RecentScans()
    # 打印获取空列表的消息，并调用r对象的get_recent_scans_list方法
    print(">>> Getting empty list:", r.get_recent_scans_list())
    # 打印添加最近扫描的消息，并调用r对象的add_recent_scan方法，传入参数"bla"
    print(">>> Adding recent scan bla:", r.add_recent_scan("bla"))
    # 打印获取最近扫描列表的消息，并调用r对象的get_recent_scans_list方法
    print(">>> Getting recent scan list:", r.get_recent_scans_list())
    # 删除r对象
    del r
```