# `nmap\zenmap\zenmapCore\UmitLogging.py`

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
# 导入日志记录器、流处理器和格式化器
from logging import Logger, StreamHandler, Formatter
# 导入应用程序显示名称和选项解析器
from zenmapCore.Name import APP_DISPLAY_NAME
from zenmapCore.UmitOptionParser import option_parser
from zenmapCore.DelayedObject import DelayedObject

# 创建日志类，继承自Logger类和object类
class Log(Logger, object):
    # 初始化方法，接受日志名称和日志级别作为参数
    def __init__(self, name, level=0):
        # 如果未指定日志级别，则从选项解析器获取
        if level == 0:
            level = option_parser.get_verbose()
        # 调用Logger类的初始化方法
        Logger.__init__(self, name, level)
        # 设置格式化器为自身的format方法
        self.formatter = self.format

        # 创建流处理器
        handler = StreamHandler()
        # 设置流处理器的格式化器为自身的格式化器
        handler.setFormatter(self.formatter)

        # 添加流处理器到日志记录器
        self.addHandler(handler)

    # 获取格式化器
    def get_formatter(self):
        return self.__formatter

    # 设置格式化器
    def set_formatter(self, fmt):
        self.__formatter = Formatter(fmt)

    # 默认日志格式
    format = "%(levelname)s - %(asctime)s - %(message)s"

    # 格式化器属性
    formatter = property(get_formatter, set_formatter, doc="")
    # 创建一个 Formatter 对象，使用指定的格式
    __formatter = Formatter(format)
# 导入 DelayedObject 类和 Log 类，用于延迟加载日志对象
log = DelayedObject(Log, APP_DISPLAY_NAME)

# 如果当前脚本作为主程序执行
if __name__ == '__main__':
    # 记录调试信息
    log.debug("Debug Message")
    # 记录信息消息
    log.info("Info Message")
    # 记录警告消息
    log.warning("Warning Message")
    # 记录错误消息
    log.error("Error Message")
    # 记录严重错误消息
    log.critical("Critical Message")
```